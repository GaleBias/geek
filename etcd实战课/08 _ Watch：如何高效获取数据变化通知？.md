<audio title="08 _ Watch：如何高效获取数据变化通知？" src="https://static001.geekbang.org/resource/audio/84/85/842443cc9ecc3ac47cf13445b39d6985.mp3" controls="controls"></audio> 
<p>你好，我是唐聪。</p><p>在Kubernetes中，各种各样的控制器实现了Deployment、StatefulSet、Job等功能强大的Workload。控制器的核心思想是监听、比较资源实际状态与期望状态是否一致，若不一致则进行协调工作，使其最终一致。</p><p>那么当你修改一个Deployment的镜像时，Deployment控制器是如何高效的感知到期望状态发生了变化呢？</p><p>要回答这个问题，得从etcd的Watch特性说起，它是Kubernetes控制器的工作基础。今天我和你分享的主题就是etcd的核心特性Watch机制设计实现，通过分析Watch机制的四大核心问题，让你了解一个变化数据是如何从0到1推送给client，并给你介绍Watch特性从etcd v2到etcd v3演进、优化过程。</p><p>希望通过这节课，你能在实际业务中应用Watch特性，快速获取数据变更通知，而不是使用可能导致大量expensive request的轮询模式。更进一步，我将帮助你掌握Watch过程中，可能会出现的各种异常错误和原因，并知道在业务中如何优雅处理，让你的服务更稳地运行。</p><h2>Watch特性初体验</h2><p>在详细介绍Watch特性实现原理之前，我先通过几个简单命令，带你初体验下Watch特性。</p><!-- [[[read_end]]] --><p>启动一个空集群，更新两次key hello后，使用Watch特性如何获取key hello的历史修改记录呢？</p><p>如下所示，你可以通过下面的watch命令，带版本号监听key hello，集群版本号可通过endpoint status命令获取，空集群启动后的版本号为1。</p><p>执行后输出如下代码所示，两个事件记录分别对应上面的两次的修改，事件中含有key、value、各类版本号等信息，你还可以通过比较create_revision和mod_revision区分此事件是add还是update事件。</p><p>watch命令执行后，你后续执行的增量put hello修改操作，它同样可持续输出最新的变更事件给你。</p><pre><code>$ etcdctl put hello world1
$ etcdctl put hello world2
$ etcdctl watch hello -w=json --rev=1
{
    &quot;Events&quot;:[
        {
            &quot;kv&quot;:{
                &quot;key&quot;:&quot;aGVsbG8=&quot;,
                &quot;create_revision&quot;:2,
                &quot;mod_revision&quot;:2,
                &quot;version&quot;:1,
                &quot;value&quot;:&quot;d29ybGQx&quot;
            }
        },
        {
            &quot;kv&quot;:{
                &quot;key&quot;:&quot;aGVsbG8=&quot;,
                &quot;create_revision&quot;:2,
                &quot;mod_revision&quot;:3,
                &quot;version&quot;:2,
                &quot;value&quot;:&quot;d29ybGQy&quot;
            }
        }
    ],
    &quot;CompactRevision&quot;:0,
    &quot;Canceled&quot;:false,
    &quot;Created&quot;:false
}
</code></pre><p>从以上初体验中，你可以看到，基于Watch特性，你可以快速获取到你感兴趣的数据变化事件，这也是Kubernetes控制器工作的核心基础。在这过程中，其实有以下四大核心问题：</p><p>第一，client获取事件的机制，etcd是使用轮询模式还是推送模式呢？两者各有什么优缺点？</p><p>第二，事件是如何存储的？ 会保留多久？watch命令中的版本号具有什么作用？</p><p>第三，当client和server端出现短暂网络波动等异常因素后，导致事件堆积时，server端会丢弃事件吗？若你监听的历史版本号server端不存在了，你的代码该如何处理？</p><p>第四，如果你创建了上万个watcher监听key变化，当server端收到一个写请求后，etcd是如何根据变化的key快速找到监听它的watcher呢？</p><p>接下来我就和你分别详细聊聊etcd Watch特性是如何解决这四大问题的。搞懂这四个问题，你就明白etcd甚至各类分布式存储Watch特性的核心实现原理了。</p><h2>轮询 vs 流式推送</h2><p>首先第一个问题是<strong>client获取事件机制</strong>，etcd是使用轮询模式还是推送模式呢？两者各有什么优缺点？</p><p>答案是两种机制etcd都使用过。</p><p>在etcd v2 Watch机制实现中，使用的是HTTP/1.x协议，实现简单、兼容性好，每个watcher对应一个TCP连接。client通过HTTP/1.1协议长连接定时轮询server，获取最新的数据变化事件。</p><p>然而当你的watcher成千上万的时，即使集群空负载，大量轮询也会产生一定的QPS，server端会消耗大量的socket、内存等资源，导致etcd的扩展性、稳定性无法满足Kubernetes等业务场景诉求。</p><p>etcd v3的Watch机制的设计实现并非凭空出现，它正是吸取了etcd v2的经验、教训而重构诞生的。</p><p>在etcd v3中，为了解决etcd v2的以上缺陷，使用的是基于HTTP/2的gRPC协议，双向流的Watch API设计，实现了连接多路复用。</p><p>HTTP/2协议为什么能实现多路复用呢？</p><p><img src="https://static001.geekbang.org/resource/image/be/74/be3a019beaf1310d214e5c9948cc9c74.png?wh=1784*534" alt="" title="引用自Google开发者文档"></p><p>在HTTP/2协议中，HTTP消息被分解独立的帧（Frame），交错发送，帧是最小的数据单位。每个帧会标识属于哪个流（Stream），流由多个数据帧组成，每个流拥有一个唯一的ID，一个数据流对应一个请求或响应包。</p><p>如上图所示，client正在向server发送数据流5的帧，同时server也正在向client发送数据流1和数据流3的一系列帧。一个连接上有并行的三个数据流，HTTP/2可基于帧的流ID将并行、交错发送的帧重新组装成完整的消息。</p><p>通过以上机制，HTTP/2就解决了HTTP/1的请求阻塞、连接无法复用的问题，实现了多路复用、乱序发送。</p><p>etcd基于以上介绍的HTTP/2协议的多路复用等机制，实现了一个client/TCP连接支持多gRPC Stream， 一个gRPC Stream又支持多个watcher，如下图所示。同时事件通知模式也从client轮询优化成server流式推送，极大降低了server端socket、内存等资源。</p><p><img src="https://static001.geekbang.org/resource/image/f0/be/f08d1c50c6bc14f09b5028095ce275be.png?wh=1804*1076" alt=""></p><p>当然在etcd v3 watch性能优化的背后，也带来了Watch API复杂度上升, 不过你不用担心，etcd的clientv3库已经帮助你搞定这些棘手的工作了。</p><p>在clientv3库中，Watch特性被抽象成Watch、Close、RequestProgress三个简单API提供给开发者使用，屏蔽了client与gRPC WatchServer交互的复杂细节，实现了一个client支持多个gRPC Stream，一个gRPC Stream支持多个watcher，显著降低了你的开发复杂度。</p><p>同时当watch连接的节点故障，clientv3库支持自动重连到健康节点，并使用之前已接收的最大版本号创建新的watcher，避免旧事件回放等。</p><h2>滑动窗口 vs MVCC</h2><p>介绍完etcd v2的轮询机制和etcd v3的流式推送机制后，再看第二个问题，事件是如何存储的？ 会保留多久呢？watch命令中的版本号具有什么作用？</p><p>第二个问题的本质是<strong>历史版本存储</strong>，etcd经历了从滑动窗口到MVCC机制的演变，滑动窗口是仅保存有限的最近历史版本到内存中，而MVCC机制则将历史版本保存在磁盘中，避免了历史版本的丢失，极大的提升了Watch机制的可靠性。</p><p>etcd v2滑动窗口是如何实现的？它有什么缺点呢？</p><p>它使用的是如下一个简单的环形数组来存储历史事件版本，当key被修改后，相关事件就会被添加到数组中来。若超过eventQueue的容量，则淘汰最旧的事件。在etcd v2中，eventQueue的容量是固定的1000，因此它最多只会保存1000条事件记录，不会占用大量etcd内存导致etcd OOM。</p><pre><code>type EventHistory struct {
   Queue      eventQueue
   StartIndex uint64
   LastIndex  uint64
   rwl        sync.RWMutex
}
</code></pre><p>但是它的缺陷显而易见的，固定的事件窗口只能保存有限的历史事件版本，是不可靠的。当写请求较多的时候、client与server网络出现波动等异常时，很容易导致事件丢失，client不得不触发大量的expensive查询操作，以获取最新的数据及版本号，才能持续监听数据。</p><p>特别是对于重度依赖Watch机制的Kubernetes来说，显然是无法接受的。因为这会导致控制器等组件频繁的发起expensive List Pod等资源操作，导致APIServer/etcd出现高负载、OOM等，对稳定性造成极大的伤害。</p><p>etcd v3的MVCC机制，正如上一节课所介绍的，就是为解决etcd v2 Watch机制不可靠而诞生。相比etcd v2直接保存事件到内存的环形数组中，etcd v3则是将一个key的历史修改版本保存在boltdb里面。boltdb是一个基于磁盘文件的持久化存储，因此它重启后历史事件不像etcd v2一样会丢失，同时你可通过配置压缩策略，来控制保存的历史版本数，在压缩篇我会和你详细讨论它。</p><p>最后watch命令中的版本号具有什么作用呢?</p><p>在上一节课中我们深入介绍了它的含义，版本号是etcd逻辑时钟，当client因网络等异常出现连接闪断后，通过版本号，它就可从server端的boltdb中获取错过的历史事件，而无需全量同步，它是etcd Watch机制数据增量同步的核心。</p><h2>可靠的事件推送机制</h2><p>再看第三个问题，当client和server端出现短暂网络波动等异常因素后，导致事件堆积时，server端会丢弃事件吗？若你监听的历史版本号server端不存在了，你的代码该如何处理？</p><p>第三个问题的本质是<strong>可靠事件推送机制</strong>，要搞懂它，我们就得弄懂etcd Watch特性的整体架构、核心流程，下图是Watch特性整体架构图。</p><h3>整体架构</h3><p><img src="https://static001.geekbang.org/resource/image/42/bf/42575d8d0a034e823b8e48d4ca0a49bf.png?wh=1920*1075" alt=""></p><p>我先通过上面的架构图，给你简要介绍下一个watch请求流程，让你对全流程有个整体的认识。</p><p>当你通过etcdctl或API发起一个watch key请求的时候，etcd的gRPCWatchServer收到watch请求后，会创建一个serverWatchStream, 它负责接收client的gRPC Stream的create/cancel watcher请求(recvLoop goroutine)，并将从MVCC模块接收的Watch事件转发给client(sendLoop goroutine)。</p><p>当serverWatchStream收到create watcher请求后，serverWatchStream会调用MVCC模块的WatchStream子模块分配一个watcher id，并将watcher注册到MVCC的WatchableKV模块。</p><p>在etcd启动的时候，WatchableKV模块会运行syncWatchersLoop和syncVictimsLoop goroutine，分别负责不同场景下的事件推送，它们也是Watch特性可靠性的核心之一。</p><p>从架构图中你可以看到Watch特性的核心实现是WatchableKV模块，下面我就为你抽丝剥茧，看看"etcdctl watch hello -w=json --rev=1"命令在WatchableKV模块是如何处理的？面对各类异常，它如何实现可靠事件推送？</p><p><strong>etcd核心解决方案是复杂度管理，问题拆分。</strong></p><p>etcd根据不同场景，对问题进行了分解，将watcher按场景分类，实现了轻重分离、低耦合。我首先给你介绍下synced watcher、unsynced watcher它们各自的含义。</p><p><strong>synced watcher</strong>，顾名思义，表示此类watcher监听的数据都已经同步完毕，在等待新的变更。</p><p>如果你创建的watcher未指定版本号(默认0)、或指定的版本号大于etcd sever当前最新的版本号(currentRev)，那么它就会保存到synced watcherGroup中。watcherGroup负责管理多个watcher，能够根据key快速找到监听该key的一个或多个watcher。</p><p><strong>unsynced watcher</strong>，表示此类watcher监听的数据还未同步完成，落后于当前最新数据变更，正在努力追赶。</p><p>如果你创建的watcher指定版本号小于etcd server当前最新版本号，那么它就会保存到unsynced watcherGroup中。比如我们的这个案例中watch带指定版本号1监听时，版本号1和etcd server当前版本之间的数据并未同步给你，因此它就属于此类。</p><p>从以上介绍中，我们可以将可靠的事件推送机制拆分成最新事件推送、异常场景重试、历史事件推送机制三个子问题来进行分析。</p><p>下面是第一个子问题，最新事件推送机制。</p><h3>最新事件推送机制</h3><p>当etcd收到一个写请求，key-value发生变化的时候，处于syncedGroup中的watcher，是如何获取到最新变化事件并推送给client的呢？</p><p><img src="https://static001.geekbang.org/resource/image/5y/48/5yy0cbf2833c438812086287d2ebf948.png?wh=1920*1060" alt=""></p><p>当你创建完成watcher后，此时你执行put hello修改操作时，如上图所示，请求经过KVServer、Raft模块后Apply到状态机时，在MVCC的put事务中，它会将本次修改的后的mvccpb.KeyValue保存到一个changes数组中。</p><p>在put事务结束时，如下面的精简代码所示，它会将KeyValue转换成Event事件，然后回调watchableStore.notify函数（流程5）。notify会匹配出监听过此key并处于synced watcherGroup中的watcher，同时事件中的版本号要大于等于watcher监听的最小版本号，才能将事件发送到此watcher的事件channel中。</p><p>serverWatchStream的sendLoop goroutine监听到channel消息后，读出消息立即推送给client（流程6和7），至此，完成一个最新修改事件推送。</p><pre><code>evs := make([]mvccpb.Event, len(changes))
for i, change := range changes {
   evs[i].Kv = &amp;changes[i]
   if change.CreateRevision == 0 {
      evs[i].Type = mvccpb.DELETE
      evs[i].Kv.ModRevision = rev
   } else {
      evs[i].Type = mvccpb.PUT
   }
}
tw.s.notify(rev, evs)
</code></pre><p>注意接收Watch事件channel的buffer容量默认1024(etcd v3.4.9)。若client与server端因网络波动、高负载等原因导致推送缓慢，buffer满了，事件会丢失吗？</p><p>这就是第二个子问题，异常场景的重试机制。</p><h3>异常场景重试机制</h3><p>若出现channel buffer满了，etcd为了保证Watch事件的高可靠性，并不会丢弃它，而是将此watcher从synced watcherGroup中删除，然后将此watcher和事件列表保存到一个名为受害者victim的watcherBatch结构中，通过<strong>异步机制重试</strong>保证事件的可靠性。</p><p>还有一个点你需要注意的是，notify操作它是在修改事务结束时同步调用的，必须是轻量级、高性能、无阻塞的，否则会严重影响集群写性能。</p><p>那么若因网络波动、CPU高负载等异常导致watcher处于victim集合中后，etcd是如何处理这种slow watcher呢？</p><p>在介绍Watch机制整体架构时，我们知道WatchableKV模块会启动两个异步goroutine，其中一个是syncVictimsLoop，正是它负责slower watcher的堆积的事件推送。</p><p>它的基本工作原理是，遍历victim watcherBatch数据结构，尝试将堆积的事件再次推送到watcher的接收channel中。若推送失败，则再次加入到victim watcherBatch数据结构中等待下次重试。</p><p>若推送成功，watcher监听的最小版本号(minRev)小于等于server当前版本号(currentRev)，说明可能还有历史事件未推送，需加入到unsynced watcherGroup中，由下面介绍的历史事件推送机制，推送minRev到currentRev之间的事件。</p><p>若watcher的最小版本号大于server当前版本号，则加入到synced watcher集合中，进入上面介绍的最新事件通知机制。</p><p>下面我给你画了一幅图总结各类watcher状态转换关系，希望能帮助你快速厘清之间关系。</p><p><img src="https://static001.geekbang.org/resource/image/40/8e/40ec1087113edfc9f7yy0f32394b948e.png?wh=1920*1065" alt=""></p><p>介绍完最新事件推送、异常场景重试机制后，那历史事件推送机制又是怎么工作的呢？</p><h3>历史事件推送机制</h3><p>WatchableKV模块的另一个goroutine，syncWatchersLoop，正是负责unsynced watcherGroup中的watcher历史事件推送。</p><p>在历史事件推送机制中，如果你监听老的版本号已经被etcd压缩了，client该如何处理？</p><p>要了解这个问题，我们就得搞清楚syncWatchersLoop如何工作，它的核心支撑是boltdb中存储了key-value的历史版本。</p><p>syncWatchersLoop，它会遍历处于unsynced watcherGroup中的每个watcher，为了优化性能，它会选择一批unsynced watcher批量同步，找出这一批unsynced watcher中监听的最小版本号。</p><p>因boltdb的key是按版本号存储的，因此可通过指定查询的key范围的最小版本号作为开始区间，当前server最大版本号作为结束区间，遍历boltdb获得所有历史数据。</p><p>然后将KeyValue结构转换成事件，匹配出监听过事件中key的watcher后，将事件发送给对应的watcher事件接收channel即可。发送完成后，watcher从unsynced watcherGroup中移除、添加到synced watcherGroup中，如下面的watcher状态转换图黑色虚线框所示。</p><p><img src="https://static001.geekbang.org/resource/image/a7/b4/a7a04846de2be66f1162af8845b13ab4.png?wh=1920*1098" alt=""></p><p>若watcher监听的版本号已经小于当前etcd server压缩的版本号，历史变更数据就可能已丢失，因此etcd server会返回ErrCompacted错误给client。client收到此错误后，需重新获取数据最新版本号后，再次Watch。你在业务开发过程中，使用Watch API最常见的一个错误之一就是未处理此错误。</p><h2>高效的事件匹配</h2><p>介绍完可靠的事件推送机制后，最后我们再看第四个问题，如果你创建了上万个watcher监听key变化，当server端收到一个写请求后，etcd是如何根据变化的key快速找到监听它的watcher呢？一个个遍历watcher吗？</p><p>显然一个个遍历watcher是最简单的方法，但是它的时间复杂度是O(N)，在watcher数较多的场景下，会导致性能出现瓶颈。更何况etcd是在执行一个写事务结束时，同步触发事件通知流程的，若匹配watcher开销较大，将严重影响etcd性能。</p><p>那使用什么数据结构来快速查找哪些watcher监听了一个事件中的key呢？</p><p>也许你会说使用map记录下哪些watcher监听了什么key不就可以了吗？ etcd的确使用map记录了监听单个key的watcher，但是你要注意的是Watch特性不仅仅可以监听单key，它还可以指定监听key范围、key前缀，因此etcd还使用了如下的区间树。</p><p><img src="https://static001.geekbang.org/resource/image/5a/88/5ae0a99629021e4a05c08yyd0df92f88.png?wh=1920*1040" alt=""></p><p>当收到创建watcher请求的时候，它会把watcher监听的key范围插入到上面的区间树中，区间的值保存了监听同样key范围的watcher集合/watcherSet。</p><p>当产生一个事件时，etcd首先需要从map查找是否有watcher监听了单key，其次它还需要从区间树找出与此key相交的所有区间，然后从区间的值获取监听的watcher集合。</p><p>区间树支持快速查找一个key是否在某个区间内，时间复杂度O(LogN)，因此etcd基于map和区间树实现了watcher与事件快速匹配，具备良好的扩展性。</p><h2>小结</h2><p>最后我们来小结今天的内容，我通过一个Watch特性初体验，提出了Watch特性设计实现的四个核心问题，分别是获取事件机制、事件历史版本存储、如何实现可靠的事件推送机制、如何高效的将事件与watcher进行匹配。</p><p>在获取事件机制、事件历史版本存储两个问题中，我给你介绍了etcd v2在使用HTTP/1.x轮询、滑动窗口时，存在大量的连接数、丢事件等问题，导致扩展性、稳定性较差。</p><p>而etcd v3 Watch特性优化思路是基于HTTP/2的流式传输、多路复用，实现了一个连接支持多个watcher，减少了大量连接数，事件存储也从滑动窗口优化成稳定可靠的MVCC机制，历史版本保存在磁盘中，具备更好的扩展性、稳定性。</p><p>在实现可靠的事件推送机制问题中，我通过一个整体架构图带你了解整个Watch机制的核心链路，数据推送流程。</p><p>Watch特性的核心实现模块是watchableStore，它通过将watcher划分为synced/unsynced/victim三类，将问题进行了分解，并通过多个后台异步循环 goroutine负责不同场景下的事件推送，提供了各类异常等场景下的Watch事件重试机制，尽力确保变更事件不丢失、按逻辑时钟版本号顺序推送给client。</p><p>最后一个事件匹配性能问题，etcd基于map和区间树数实现了watcher与事件快速匹配，保障了大规模场景下的Watch机制性能和读写稳定性。</p><h2>思考题</h2><p>好了，这节课到这里也就结束了。我们一块来做一下思考题吧。</p><p>业务场景是希望agent能通过Watch机制监听server端下发给它的任务信息，简要实现如下，你认为它存在哪些问题呢？ 它一定能监听到server下发给其的所有任务信息吗？欢迎你给出正确的解决方案。</p><pre><code>taskPrefix := &quot;/task/&quot; + &quot;Agent IP&quot;
rsp, err := cli.Get(context.Background(), taskPrefix, clientv3.WithPrefix())
if err != nil {
   log.Fatal(err)
}
// to do something
// ....
// Watch taskPrefix
rch := cli.Watch(context.Background(), taskPrefix, clientv3.WithPrefix())
for wresp := range rch {
   for _, ev := range wresp.Events {
      fmt.Printf(&quot;%s %q : %q\n&quot;, ev.Type, ev.Kv.Key, ev.Kv.Value)
   }
}
</code></pre><p>感谢你的阅读，如果你认为这节课的内容有收获，也欢迎把它分享给你的朋友，谢谢。</p>
<style>
    ul {
      list-style: none;
      display: block;
      list-style-type: disc;
      margin-block-start: 1em;
      margin-block-end: 1em;
      margin-inline-start: 0px;
      margin-inline-end: 0px;
      padding-inline-start: 40px;
    }
    li {
      display: list-item;
      text-align: -webkit-match-parent;
    }
    ._2sjJGcOH_0 {
      list-style-position: inside;
      width: 100%;
      display: -webkit-box;
      display: -ms-flexbox;
      display: flex;
      -webkit-box-orient: horizontal;
      -webkit-box-direction: normal;
      -ms-flex-direction: row;
      flex-direction: row;
      margin-top: 26px;
      border-bottom: 1px solid rgba(233,233,233,0.6);
    }
    ._2sjJGcOH_0 ._3FLYR4bF_0 {
      width: 34px;
      height: 34px;
      -ms-flex-negative: 0;
      flex-shrink: 0;
      border-radius: 50%;
    }
    ._2sjJGcOH_0 ._36ChpWj4_0 {
      margin-left: 0.5rem;
      -webkit-box-flex: 1;
      -ms-flex-positive: 1;
      flex-grow: 1;
      padding-bottom: 20px;
    }
    ._2sjJGcOH_0 ._36ChpWj4_0 ._2zFoi7sd_0 {
      font-size: 16px;
      color: #3d464d;
      font-weight: 500;
      -webkit-font-smoothing: antialiased;
      line-height: 34px;
    }
    ._2sjJGcOH_0 ._36ChpWj4_0 ._2_QraFYR_0 {
      margin-top: 12px;
      color: #505050;
      -webkit-font-smoothing: antialiased;
      font-size: 14px;
      font-weight: 400;
      white-space: normal;
      word-break: break-all;
      line-height: 24px;
    }
    ._2sjJGcOH_0 ._10o3OAxT_0 {
      margin-top: 18px;
      border-radius: 4px;
      background-color: #f6f7fb;
    }
    ._2sjJGcOH_0 ._3klNVc4Z_0 {
      display: -webkit-box;
      display: -ms-flexbox;
      display: flex;
      -webkit-box-orient: horizontal;
      -webkit-box-direction: normal;
      -ms-flex-direction: row;
      flex-direction: row;
      -webkit-box-pack: justify;
      -ms-flex-pack: justify;
      justify-content: space-between;
      -webkit-box-align: center;
      -ms-flex-align: center;
      align-items: center;
      margin-top: 15px;
    }
    ._2sjJGcOH_0 ._10o3OAxT_0 ._3KxQPN3V_0 {
      color: #505050;
      -webkit-font-smoothing: antialiased;
      font-size: 14px;
      font-weight: 400;
      white-space: normal;
      word-break: break-word;
      padding: 20px 20px 20px 24px;
    }
    ._2sjJGcOH_0 ._3klNVc4Z_0 {
      display: -webkit-box;
      display: -ms-flexbox;
      display: flex;
      -webkit-box-orient: horizontal;
      -webkit-box-direction: normal;
      -ms-flex-direction: row;
      flex-direction: row;
      -webkit-box-pack: justify;
      -ms-flex-pack: justify;
      justify-content: space-between;
      -webkit-box-align: center;
      -ms-flex-align: center;
      align-items: center;
      margin-top: 15px;
    }
    ._2sjJGcOH_0 ._3Hkula0k_0 {
      color: #b2b2b2;
      font-size: 14px;
    }
</style><ul><li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/57/4f/6fb51ff1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>一步</span>
  </div>
  <div class="_2_QraFYR_0">当某个 key 发生变化的一定要去 key 的 watcher 区间树去查询是否对应的 watcher 吗？ 这样会不会有很大的浪费，当一个系统 监听了一些不经常变的 key, 另一些经常变化的 key 没有对应的 watcher ,这样经常变化的key就会一直去查询key 的 watcher 区间树。  难道 key 不能直接知道是否有对应的 watcher 呢？ （比如 key 的结构体记录一个 watcher Id 的数组）</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，一定会查，这性能开销很低的。正如我文章中所介绍的，有两类watcher, 一类是监听一个key的watcher，这种直接使用map来保存，也就是检查这个key是否有相关watcher，时间复杂度是o(1)，另一类是监听整个区间的watcher，查找这个key落在了哪些watcher监听的范围，一般情况下，时间复杂度o(log n), n是你创建的watcher数。而你建议的方案，使用key结构体记录一个watcher id数组，这在key非常多的情况下，内存开销是非常大，假设一个watcher监听了一个非常大的key范围，那么etcd创建此watcher的时候，还需要遍历这个key范围，给key增加相应的watcher id, 不仅内存，cpu开销也是不可忽略的。<br><br>在哪些场景下watcher事件通知性能会非常慢呢？如果你成千上万个watcher,监听同一个key或者范围时，就会导致性能极差。 </p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-06 17:00:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/e8/ac/7324d5ca.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>七里</span>
  </div>
  <div class="_2_QraFYR_0">如何保证WatchableKV 模块启动的syncVictimsLoop是可靠的呢？机器重启了怎么办？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 好问题，整个watch链路的可靠性，不能光靠server，从业务逻辑正确使用watch api到client watch中断重试机制，再到本讲介绍的server推送机制，层层相连才能尽量保证watch事件可靠性。如果机器重启了，这时就依赖client库的重试机制了，client收到每个watch事件时，会记录收到的版本号，连接到新节点后，再次创建watcher时会带上这个版本号，然后就会进入历史数据同步逻辑。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-07 15:27:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/41/38/4f89095b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>写点啥呢</span>
  </div>
  <div class="_2_QraFYR_0">请问下老师，如果etcd集群某个节点崩溃或者网络问题导致client&#47;server间连接断开，etcd是如何处理watch重新连接？如果是在另外一个节点重连后，etcd如何确定哪些event已经发送来避免event重复发送呢？<br><br>谢谢老师</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: etcd clientv3 watch库有中断重连、恢复机制，它会记录上次已收到的watch事件版本号，重启到新节点后，创建watcher的时候会带上这个版本，这样就可以尽量避免event重复发送哈，你可以使用etcd 3.4.9版本自己测试体验下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-05 16:37:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/9a/fd/30c425f9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>BeanNan</span>
  </div>
  <div class="_2_QraFYR_0">想问下老师，思考题的答案在哪有解惑？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-04 14:36:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/cb/38/4c9cfdf4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>谢小路</span>
  </div>
  <div class="_2_QraFYR_0">学 etcd 还是需要些功底的，基础薄弱应该看不懂这些。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-07 23:21:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTIVvyFCLRcfoWfiaJt99K0wiabvicWtQaJdSseVA6QqWyxcvN5nd2TgZqiaUACc94bBvPHZTibnfnZfdtQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_7d539e</span>
  </div>
  <div class="_2_QraFYR_0">有一个地方没有写明：<br>watch 通过 raft 通讯模块，传播到各个节点，也就是给个节点都有这个 watch(id) ，key 的 put 更新操作，给个节点自然都能监听到该 key 的变化，那么是各个节点都会去通知这个变化吗？还是就只有一个节点会去通知 client ？我本人猜测是，每个节点都会去通知，但是并不是每个节点都有 client watch steam ，没有连接 stream 就自然丢弃通知，有 stream 才会去落实通知。请老师释疑下，多谢。还有顺道多问一个问题，put 请求是单 key 的事务操作吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-27 16:51:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/57/4f/6fb51ff1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>一步</span>
  </div>
  <div class="_2_QraFYR_0">带 --rev 版本号的监听，不是从需要监听某个 key 的某个指定的版本开始吗？ 文章中怎么写的是集群的版本号</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 版本号是集群的逻辑时间，参考07，监听key时，你可以指定任意一个时间点，过去、未来的版本号都可以。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-05 23:26:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/fa/64/457325e6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Sam Fu</span>
  </div>
  <div class="_2_QraFYR_0">老师 unsynced group是怎么推送消息给client端的 unsynced group在状态机中不会转化为victimed suycgroup吗</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-15 12:31:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/fa/64/457325e6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Sam Fu</span>
  </div>
  <div class="_2_QraFYR_0">老师 这个区间树保存key前缀保存到哪一级啊 比如key为abcde, 那么区间数会咋保存呢？a，ab，abc，abcd都会保存吗</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-15 12:26:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/3b/ba/3b30dcde.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>窝窝头</span>
  </div>
  <div class="_2_QraFYR_0">不一定能监听到所有任务，有可能历史变更数据已丢失，返回个ErrCompacted</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-03 13:03:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLDRPejHodutia9Ud8UZLY8g5lTkKXgf3J104c0jM9aFfAGNoUdxkRLnnWRc5Kd3jIeN3EqXxKFT0g/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>蓝莓侠</span>
  </div>
  <div class="_2_QraFYR_0">同时当 watch 连接的节点故障，clientv3 库支持自动重连到健康节点，并使用之前已接收的最大版本号创建新的 watcher，避免旧事件回放等。<br>如果是watch的etcd某个node节点网络和另外两个隔离了，watch会重连到其他健康节点吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-23 16:08:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>woJA1wCgAAduicW8WCYUooKMsgw232aA</span>
  </div>
  <div class="_2_QraFYR_0">老师，我想问一下没有设置集群的情况下。一台服务器能够在本机监听另外一台服务器的etcd服务的key的修改吗。即跨服务器监听key？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-25 18:15:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_af1847</span>
  </div>
  <div class="_2_QraFYR_0">在获取key的最新值到watch期间，监听的key可能会发生变化，这段时间内的事件就会丢失，所以watch的时候需要带上rev。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-07 17:29:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/42/9d/c36b7ef7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>顾骨</span>
  </div>
  <div class="_2_QraFYR_0">老师，你好<br><br>我看区间树的示意图，其中一个节点[&#47;a,&#47;b]，那么：<br>问题1：是否还有两个叶子节点[&#47;a]和[&#47;b]，那么 abc，abcd，abcde 是否存在叶子节点 [&#47;a]，bc，bcde，bcdef 是否存在叶子节点[&#47;b]上。<br>      1.1：如果是，叶子节点（比如 [&#47;a]）上的数据要求有序吗？如果大量的节点落在叶子节点 [&#47;a] 上，查找性能如何。<br>      1.2：如果不是，那么数据又是如何存储在区间树里面呢。<br><br>问题2：如果 key 为汉字，如何存储在区间树上？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-12 11:15:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLDUJyeq54fiaXAgF62tNeocO3lHsKT4mygEcNoZLnibg6ONKicMgCgUHSfgW8hrMUXlwpNSzR8MHZwg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>types</span>
  </div>
  <div class="_2_QraFYR_0">watcher 的最小版本号，会随着事件的推送，而更新吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-25 22:14:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/3a/27/5d218272.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>八台上</span>
  </div>
  <div class="_2_QraFYR_0">想问一下  每个key  有没有时间戳呢 想看真实的创建时间</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 目前没有，你可以通过版本号信息来判断对比key的创建、修改时间等，它是个逻辑时间</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-03 10:19:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/25/83/7e/e8d1771e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>白</span>
  </div>
  <div class="_2_QraFYR_0">如果客户端不需要订阅了，创建的watcher怎么删除呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-11 16:24:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/88/03/805c8e0e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>追忆似水年华</span>
  </div>
  <div class="_2_QraFYR_0">“client 与 server 网络出现波动等异常时，很容易导致事件丢失，client 不得不触发大量的 expensive 查询操作，以获取最新的数据及版本号，才能持续监听数据”这段话不是很明白，事件丢失为什么出发大量的昂贵查询，还要获取最新数据和版本号</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-01 16:17:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/57/4f/6fb51ff1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>一步</span>
  </div>
  <div class="_2_QraFYR_0">watcher key的区间树创建过程是怎么样的？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-07 22:33:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/92/be/8de4e1fe.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>kaizen</span>
  </div>
  <div class="_2_QraFYR_0">老师，什么应用场景需要使用这种key有多个版本value 的机制呢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-06 17:28:23</div>
  </div>
</div>
</div>
</li>
</ul>