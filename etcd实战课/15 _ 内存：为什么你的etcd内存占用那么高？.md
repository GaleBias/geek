<audio title="15 _ 内存：为什么你的etcd内存占用那么高？" src="https://static001.geekbang.org/resource/audio/09/a9/0970df599e44da98ee3cb32b4c7393a9.mp3" controls="controls"></audio> 
<p>你好，我是唐聪。</p><p>在使用etcd的过程中，你是否被异常内存占用等现象困扰过？比如etcd中只保存了1个1MB的key-value，但是经过若干次修改后，最终etcd内存可能达到数G。它是由什么原因导致的？如何分析呢？</p><p>这就是我今天要和你分享的主题：etcd的内存。 希望通过这节课，帮助你掌握etcd内存抖动、异常背后的常见原因和分析方法，当你遇到类似问题时，能独立定位、解决。同时，帮助你在实际业务场景中，为集群节点配置充足的内存资源，遵循最佳实践，尽量减少expensive request，避免etcd内存出现突增，导致OOM。</p><h2>分析整体思路</h2><p>当你遇到etcd内存占用较高的案例时，你脑海中第一反应是什么呢？</p><p>也许你会立刻重启etcd进程，尝试将内存降低到合理水平，避免线上服务出问题。</p><p>也许你会开启etcd debug模式，重启etcd进程等复现，然后采集heap profile分析内存占用。</p><p>以上措施都有其合理性。但作为团队内etcd高手的你，在集群稳定性还不影响业务的前提下，能否先通过内存异常的现场，结合etcd的读写流程、各核心模块中可能会使用较多内存的关键数据结构，推测出内存异常的可能原因？</p><p>全方位的分析内存异常现场，可以帮助我们节省大量复现和定位时间，也是你专业性的体现。</p><!-- [[[read_end]]] --><p>下图是我以etcd写请求流程为例，给你总结的可能导致etcd内存占用较高的核心模块与其数据结构。</p><p><img src="https://static001.geekbang.org/resource/image/c2/49/c2673ebb2db4b555a9fbe229ed1bda49.png?wh=1920*1056" alt=""></p><p>从图中你可以看到，当etcd收到一个写请求后，gRPC Server会和你建立连接。连接数越多，会导致etcd进程的fd、goroutine等资源上涨，因此会使用越来越多的内存。</p><p>其次，基于我们<a href="https://time.geekbang.org/column/article/337604">04</a>介绍的Raft知识背景，它需要将此请求的日志条目保存在raftLog里面。etcd raftLog后端实现是内存存储，核心就是数组。因此raftLog使用的内存与其保存的日志条目成正比，它也是内存分析过程中最容易被忽视的一个数据结构。</p><p>然后当此日志条目被集群多数节点确认后，在应用到状态机的过程中，会在内存treeIndex模块的B-tree中创建、更新key与版本号信息。 在这过程中treeIndex模块的B-tree使用的内存与key、历史版本号数量成正比。</p><p>更新完treeIndex模块的索引信息后，etcd将key-value数据持久化存储到boltdb。boltdb使用了mmap技术，将db文件映射到操作系统内存中。因此在未触发操作系统将db对应的内存page换出的情况下，etcd的db文件越大，使用的内存也就越大。</p><p>同时，在这个过程中还有两个注意事项。</p><p>一方面，其他client可能会创建若干watcher、监听这个写请求涉及的key， etcd也需要使用一定的内存维护watcher、推送key变化监听的事件。</p><p>另一方面，如果这个写请求的key还关联了Lease，Lease模块会在内存中使用数据结构Heap来快速淘汰过期的Lease，因此Heap也是一个占用一定内存的数据结构。</p><p>最后，不仅仅是写请求流程会占用内存，读请求本身也会导致内存上升。尤其是expensive request，当产生大包查询时，MVCC模块需要使用内存保存查询的结果，很容易导致内存突增。</p><p>基于以上读写流程图对核心数据结构使用内存的分析，我们定位问题时就有线索、方法可循了。那如何确定是哪个模块、场景导致的内存异常呢？</p><p>接下来我就通过一个实际案例，和你深入介绍下内存异常的分析方法。</p><h2>一个key使用数G内存的案例</h2><p>我们通过goreman启动一个3节点etcd集群(linux/etcd v3.4.9)，db quota为6G，执行如下的命令并观察etcd内存占用情况：</p><ul>
<li>执行1000次的put同一个key操作，value为1MB；</li>
<li>更新完后并进行compact、defrag操作；</li>
</ul><pre><code># put同一个key，执行1000次
for i in {1..1000}; do dd if=/dev/urandom bs=1024 
count=1024  | ETCDCTL_API=3 etcdctl put key  || break; done

# 获取最新revision，并压缩
etcdctl compact `(etcdctl endpoint status --write-out=&quot;json&quot; | egrep -o '&quot;revision&quot;:[0-9]*' | egrep -o '[0-9].*')`

# 对集群所有节点进行碎片整理
etcdctl defrag --cluster
</code></pre><p>在执行操作前，空集群etcd db size 20KB，etcd进程内存36M左右，分别如下图所示。</p><p><img src="https://static001.geekbang.org/resource/image/c1/e6/c1fb89ae1d6218a66cf1db30c41d9be6.png?wh=1164*358" alt=""></p><p><img src="https://static001.geekbang.org/resource/image/6c/6d/6ce074583f39cd9a19bdcb392133426d.png?wh=1318*420" alt=""></p><p>你预测执行1000次同样key更新后，etcd进程占用了多少内存呢? 约37M？ 1G？ 2G？3G？ 还是其他呢？</p><p>执行1000次的put操作后，db大小和etcd内存占用分别如下图所示。</p><p><img src="https://static001.geekbang.org/resource/image/d6/45/d6dc86f76f52dfed73ab1771ebbbf545.png?wh=1302*348" alt=""></p><p><img src="https://static001.geekbang.org/resource/image/9d/70/9d97762851c18a0c4cd89aa5a7bb0270.png?wh=1310*404" alt=""></p><p>当我们执行compact、defrag命令后，如下图所示，db大小只有1M左右，但是你会发现etcd进程实际却仍占用了2G左右内存。<br>
<img src="https://static001.geekbang.org/resource/image/93/bd/937c3fb0bf12595928e8ae4b05b7a5bd.png?wh=1260*354" alt=""></p><p><img src="https://static001.geekbang.org/resource/image/8d/58/8d2d9fb3c0193745d80fe68b0cb4a758.png?wh=1276*406" alt=""></p><p>整个集群只有一个key，为什么etcd占用了这么多的内存呢？是etcd发生了内存泄露吗？</p><h2>raftLog</h2><p>当你发起一个put请求的时候，etcd需通过Raft模块将此请求同步到其他节点，详细流程你可结合下图再次了解下。</p><p><img src="https://static001.geekbang.org/resource/image/df/2c/df9yy18a1e28e18295cfc15a28cd342c.png?wh=1920*1328" alt=""></p><p>从图中你可以看到，Raft模块的输入是一个消息/Msg，输出统一为Ready结构。etcd会把此请求封装成一个消息，提交到Raft模块。</p><p>Raft模块收到此请求后，会把此消息追加到raftLog的unstable存储的entry内存数组中（图中流程2），并且将待持久化的此消息封装到Ready结构内，通过管道通知到etcdserver（图中流程3）。</p><p>etcdserver取出消息，持久化到WAL中，并追加到raftLog的内存存储storage的entry数组中（图中流程5）。</p><p>下面是<a href="https://github.com/etcd-io/etcd/blob/v3.4.9/raft/log.go#L24:L45">raftLog</a>的核心数据结构，它由storage、unstable、committed、applied等组成。storage存储已经持久化到WAL中的日志条目，unstable存储未持久化的条目和快照，一旦持久化会及时删除日志条目，因此不存在过多内存占用的问题。</p><pre><code>type raftLog struct {
   // storage contains all stable entries since the last snapshot.
   storage Storage


   // unstable contains all unstable entries and snapshot.
   // they will be saved into storage.
   unstable unstable


   // committed is the highest log position that is known to be in
   // stable storage on a quorum of nodes.
   committed uint64
   // applied is the highest log position that the application has
   // been instructed to apply to its state machine.
   // Invariant: applied &lt;= committed
   applied uint64
}
</code></pre><p>从上面raftLog结构体中，你可以看到，存储稳定的日志条目的storage类型是Storage，Storage定义了存储Raft日志条目的核心API接口，业务应用层可根据实际场景进行定制化实现。etcd使用的是Raft算法库本身提供的MemoryStorage，其定义如下，核心是使用了一个数组来存储已经持久化后的日志条目。</p><pre><code>// MemoryStorage implements the Storage interface backed
// by an in-memory array.
type MemoryStorage struct {
   // Protects access to all fields. Most methods of MemoryStorage are
   // run on the raft goroutine， but Append() is run on an application
   // goroutine.
   sync.Mutex

   hardState pb.HardState
   snapshot  pb.Snapshot
   // ents[i] has raftLog position i+snapshot.Metadata.Index
   ents []pb.Entry
}
</code></pre><p>那么随着写请求增多，内存中保留的Raft日志条目会越来越多，如何防止etcd出现OOM呢？</p><p>etcd提供了快照和压缩功能来解决这个问题。</p><p>首先你可以通过调整--snapshot-count参数来控制生成快照的频率，其值默认是100000（etcd v3.4.9，早期etcd版本是10000），也就是每10万个写请求触发一次快照生成操作。</p><p>快照生成完之后，etcd会通过压缩来删除旧的日志条目。</p><p>那么是全部删除日志条目还是保留一小部分呢？</p><p>答案是保留一小部分Raft日志条目。数量由DefaultSnapshotCatchUpEntries参数控制，默认5000，目前不支持自定义配置。</p><p>保留一小部分日志条目其实是为了帮助慢的Follower以较低的开销向Leader获取Raft日志条目，以尽快追上Leader进度。若raftLog中不保留任何日志条目，就只能发送快照给慢的Follower，这开销就非常大了。</p><p>通过以上分析可知，如果你的请求key-value比较大，比如上面我们的案例中是1M，1000次修改，那么etcd raftLog至少会消耗1G的内存。这就是为什么内存随着写请求修改次数不断增长的原因。</p><p>除了raftLog占用内存外，MVCC模块的treeIndex/boltdb模块又是如何使用内存的呢？</p><h2>treeIndex</h2><p>一个put写请求的日志条目被集群多数节点确认提交后，这时etcdserver就会从Raft模块获取已提交的日志条目，应用到MVCC模块的treeIndex和boltdb。</p><p>我们知道treeIndex是基于google内存btree库实现的一个索引管理模块，在etcd中每个key都会在treeIndex中保存一个索引项(keyIndex)，记录你的key和版本号等信息，如下面的数据结构所示。</p><pre><code>type keyIndex struct {
   key         []byte
   modified    revision // the main rev of the last modification
   generations []generation
}
</code></pre><p>同时，你每次对key的修改、删除操作都会在key的索引项中追加一条修改记录(revision)。因此，随着修改次数的增加，etcd内存会一直增加。那么如何清理旧版本，防止过多的内存占用呢？</p><p>答案也是压缩。正如我在<a href="https://time.geekbang.org/column/article/342891">11</a>压缩篇和你介绍的，当你执行compact命令时，etcd会遍历treeIndex中的各个keyIndex，清理历史版本号记录与已删除的key，释放内存。</p><p>从上面的keyIndex数据结构我们可知，一个key的索引项内存开销跟你的key大小、保存的历史版本数、compact策略有关。为了避免内存索引项占用过多的内存，key的长度不应过长，同时你需要配置好合理的压缩策略。</p><h2>boltdb</h2><p>在treeIndex模块中创建、更新完keyIndex数据结构后，你的key-value数据、各种版本号、lease等相关信息会保存到如下的一个mvccpb.keyValue结构体中。它是boltdb的value，key则是treeIndex中保存的版本号，然后通过boltdb的写接口保存到db文件中。</p><pre><code>kv := mvccpb.KeyValue{
   Key:            key，
   Value:          value，
   CreateRevision: c，
   ModRevision:    rev，
   Version:        ver，
   Lease:          int64(leaseID)，
}
</code></pre><p>前面我们在介绍boltdb时，提到过etcd在启动时会通过mmap机制，将etcd db文件映射到etcd进程地址空间，并设置mmap的MAP_POPULATE flag，它会告诉Linux内核预读文件，让Linux内核将文件内容拷贝到物理内存中。</p><p>在节点内存足够的情况下，后续读请求可直接从内存中获取。相比read系统调用，mmap少了一次从page cache拷贝到进程内存地址空间的操作，因此具备更好的性能。</p><p>若etcd节点内存不足，可能会导致db文件对应的内存页被换出。当读请求命中的页未在内存中时，就会产生缺页异常，导致读过程中产生磁盘IO。这样虽然避免了etcd进程OOM，但是此过程会产生较大的延时。</p><p>从以上boltdb的key-value和mmap机制介绍中我们可知，我们应控制boltdb文件大小，优化key-value大小，配置合理的压缩策略，回收旧版本，避免过多内存占用。</p><h2>watcher</h2><p>在你写入key的时候，其他client还可通过etcd的Watch监听机制，获取到key的变化事件。</p><p>那创建一个watcher耗费的内存跟哪些因素有关呢?</p><p>在<a href="https://time.geekbang.org/column/article/341060">08</a>Watch机制设计与实现分析中，我和你介绍过创建watcher的整体流程与架构，如下图所示。当你创建一个watcher时，client与server建立连接后，会创建一个gRPC Watch Stream，随后通过这个gRPC Watch Stream发送创建watcher请求。</p><p>每个gRPC Watch Stream中etcd WatchServer会分配两个goroutine处理，一个是sendLoop，它负责Watch事件的推送。一个是recvLoop，负责接收client的创建、取消watcher请求消息。</p><p>同时对每个watcher来说，etcd的WatchableKV模块需将其保存到相应的内存管理数据结构中，实现可靠的Watch事件推送。</p><p><img src="https://static001.geekbang.org/resource/image/42/bf/42575d8d0a034e823b8e48d4ca0a49bf.png?wh=1920*1075" alt=""></p><p>因此watch监听机制耗费的内存跟client连接数、gRPC Stream、watcher数(watching)有关，如下面公式所示：</p><ul>
<li>c1表示每个连接耗费的内存；</li>
<li>c2表示每个gRPC Stream耗费的内存；</li>
<li>c3表示每个watcher耗费的内存。</li>
</ul><pre><code>memory = c1 * number_of_conn + c2 * 
avg_number_of_stream_per_conn + c3 * 
avg_number_of_watch_stream
</code></pre><p>根据etcd社区的<a href="https://etcd.io/docs/v3.4.0/benchmarks/etcd-3-watch-memory-benchmark/">压测报告</a>，大概估算出Watch机制中c1、c2、c3占用的内存分别如下：</p><ul>
<li>每个client连接消耗大约17kb的内存(c1)；</li>
<li>每个gRPC Stream消耗大约18kb的内存(c2)；</li>
<li>每个watcher消耗大约350个字节(c3)；</li>
</ul><p>当你的业务场景大量使用watcher的时候，应提前估算下内存容量大小，选择合适的内存配置节点。</p><p>注意以上估算并不包括watch事件堆积的开销。变更事件较多，服务端、客户端高负载，网络阻塞等情况都可能导致事件堆积。</p><p>在etcd 3.4.9版本中，每个watcher默认buffer是1024。buffer内保存watch响应结果，如watchID、watch事件（watch事件包含key、value）等。</p><p>若大量事件堆积，将产生较高昂的内存的开销。你可以通过etcd_debugging_mvcc_pending_events_total指标监控堆积的事件数，etcd_debugging_slow_watcher_total指标监控慢的watcher数，来及时发现异常。</p><h2>expensive request</h2><p>当你写入比较大的key-value后，如果client频繁查询它，也会产生高昂的内存开销。</p><p>假设我们写入了100个这样1M大小的key， 通过Range接口一次查询100个key， 那么boltdb遍历、反序列化过程将花费至少100MB的内存。如下面代码所示，它会遍历整个key-value，将key-value保存到数组kvs中。</p><pre><code>kvs := make([]mvccpb.KeyValue， limit)
revBytes := newRevBytes()
for i， revpair := range revpairs[:len(kvs)] {
   revToBytes(revpair， revBytes)
   _， vs := tr.tx.UnsafeRange(keyBucketName， revBytes， nil， 0)
   if len(vs) != 1 {
        ......    
   }
   if err := kvs[i].Unmarshal(vs[0]); err != nil {
        .......
   }
</code></pre><p>也就是说，一次查询就耗费了至少100MB的内存、产生了至少100MB的流量，随着你QPS增大后，很容易OOM、网卡出现丢包。</p><p>count-only、limit查询在key百万级以上时，也会产生非常大的内存开销。因为它们在遍历treeIndex的过程中，会将相关key保存在数组里面。当key多时，此开销不容忽视。</p><p>正如我在<a href="https://time.geekbang.org/column/article/343245">13</a>  db大小中讲到的，在master分支，我已提交相关PR解决count-only和limit查询导致内存占用突增的问题。</p><h2>etcd v2/goroutines/bug</h2><p>除了以上介绍的核心模块、expensive request场景可能导致较高的内存开销外，还有以下场景也会导致etcd内存使用较高。</p><p>首先是<strong>etcd中使用了v2的API写入了大量的key-value数据</strong>，这会导致内存飙高。我们知道etcd v2的key-value都是存储在内存树中的，同时v2的watcher不支持多路复用，内存开销相比v3多了一个数量级。</p><p>在etcd 3.4版本之前，etcd默认同时支持etcd v2/v3 API，etcd 3.4版本默认关闭了v2 API。 你可以通过etcd v2 API和etcd v2内存存储模块的metrics前缀etcd_debugging_store，观察集群中是否有v2数据导致的内存占用高。</p><p>其次是<strong>goroutines泄露</strong>导致内存占用高。此问题可能会在容器化场景中遇到。etcd在打印日志的时候，若出现阻塞则可能会导致goroutine阻塞并持续泄露，最终导致内存泄露。你可以通过观察、监控go_goroutines来发现这个问题。</p><p>最后是<strong>etcd bug</strong>导致的内存泄露。当你基本排除以上场景导致的内存占用高后，则很可能是etcd bug导致的内存泄露。</p><p>比如早期etcd clientv3的lease keepalive租约频繁续期bug，它会导致Leader高负载、内存泄露，此bug已在3.2.24/3.3.9版本中修复。</p><p>还有最近我修复的etcd 3.4版本的<a href="https://github.com/etcd-io/etcd/pull/11731">Follower节点内存泄露</a>。具体表现是两个Follower节点内存一直升高，Leader节点正常，已在3.4.6版本中修复。</p><p>若内存泄露并不是已知的etcd bug导致，那你可以开启pprof， 尝试复现，通过分析pprof heap文件来确定消耗大量内存的模块和数据结构。</p><h2>小节</h2><p>今天我通过一个写入1MB  key的实际案例，给你介绍了可能导致etcd内存占用高的核心数据结构、场景，同时我将可能导致内存占用较高的因素总结为了下面这幅图，你可以参考一下。</p><p><img src="https://static001.geekbang.org/resource/image/aa/90/aaf7b4f5f6f568dc70c1a0964fb92790.png?wh=1920*684" alt=""></p><p>首先是raftLog。为了帮助slow Follower同步数据，它至少要保留5000条最近收到的写请求在内存中。若你的key非常大，你更新5000次会产生较大的内存开销。</p><p>其次是treeIndex。 每个key-value会在内存中保留一个索引项。索引项的开销跟key长度、保留的历史版本有关，你可以通过compact命令压缩。</p><p>然后是boltdb。etcd启动的时候，会通过mmap系统调用，将文件映射到虚拟内存中。你可以通过compact命令回收旧版本，defrag命令进行碎片整理。</p><p>接着是watcher。它的内存占用跟连接数、gRPC Watch Stream数、watcher数有关。watch机制一个不可忽视的内存开销其实是事件堆积的占用缓存，你可以通过相关metrics及时发现堆积的事件以及slow watcher。</p><p>最后我介绍了一些典型的场景导致的内存异常，如大包查询等expensive request，etcd中存储了v2 API写入的key， goroutines泄露以及etcd lease bug等。</p><p>希望今天的内容，能够帮助你从容应对etcd内存占用高的问题，合理配置你的集群，优化业务expensive request，让etcd跑得更稳。</p><h2>思考题</h2><p>在一个key使用数G内存的案例中，最后执行compact和defrag后的结果是2G，为什么不是1G左右呢？在macOS下行为是否一样呢？</p><p>欢迎你动手做下这个小实验，分析下原因，分享你的观点。</p><p>感谢你的阅读，如果你认为这节课的内容有收获，也欢迎把它分享给你的朋友，谢谢。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/e0/4e/c266bdb4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>[小狗]</span>
  </div>
  <div class="_2_QraFYR_0">大佬，我想阅读下etcd的源码，有什么建议吗？ 目前只是为了简历好看些，以后会深度学习下</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 建议可以先从早期的v2代码看起，那时逻辑最简单，https:&#47;&#47;github.com&#47;etcd-io&#47;etcd&#47;blob&#47;release-0.4&#47;server&#47;v2&#47;get_handler.go,然后再看etcd v3的代码，在这个过程中，我给你几个小建议：<br>1. 抓住主次，比如核心读写流程是怎样的，忽略一些特殊细节<br>2. 看看测试用例如何使用核心模块的API的，比如etcd v3 mvcc的模块测试文件<br>https:&#47;&#47;github.com&#47;etcd-io&#47;etcd&#47;blob&#47;v3.4.9&#47;mvcc&#47;kv_test.go<br>3. 自己可动手写写源码分析<br>4. 自己多实践下，部署个单机etcd集群，至少要把etcdctl各个命令给操作下<br>5. 日志级别可以改成debug, 更加方便观察</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-24 14:18:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/65/b7/058276dc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>i_chase</span>
  </div>
  <div class="_2_QraFYR_0">这一期的思考题有答案吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 主要原因是etcd 3.4 版本是使用 go 1.12 编译的，go runtime 的默认内存管理策略是 MADV_FREE, 它的性能较好，但是会导致你看到的etcd内存虚高，监控指标异常、用户体验不佳等问题，原因是这种策略，在系统内存有压力的时候，内核才会释放占用的内存。<br>从 go v1.16 起，Go 在 Linux 下的默认内存管理策略变成了 MADV_DONTNEED 策略。MADV_DONTNEED 虽然效率相比 MADV_FREE 策略较低，但是会让 rss 内存下降较快，更加符合直观感受，能避免 MADV_FREE 相关的副作用。<br>更详细的可以参考golang issue和etcd 3.5 blog.<br>https:&#47;&#47;github.com&#47;golang&#47;go&#47;issues&#47;42330<br>https:&#47;&#47;etcd.io&#47;blog&#47;2021&#47;announcing-etcd-3.5 Monitoring部分</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-01 21:45:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83ep075ibtmxMf3eOYlBJ96CE9TEelLUwePaLqp8M75gWHEcM3za0voylA0oe9y3NiaboPB891rypRt7w/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>shuff1e</span>
  </div>
  <div class="_2_QraFYR_0">etcd v2 的 key-value 都是存储在内存树中，具体指的是什么呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是指etcd v2的key-value数据存储在内存中的,如果你etcd v3里面存储了大量的etcd v2 key-value数据，就会导致内存占用较高。<br>你可以参考下etcd v2 store的定义，etcd v2的key-value数据就是直接存储在这个结构哈<br>https:&#47;&#47;github.com&#47;etcd-io&#47;etcd&#47;blob&#47;release-3.3&#47;store&#47;store.go#L73:L83</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-22 16:02:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/a2/94/ae0a60d8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>江山未</span>
  </div>
  <div class="_2_QraFYR_0">老师，想问下。工作中有用到etcd，如果想通过看etcd源码提升了解的话，建议从哪里入手呢。<br>如果可能的话，也希望能像你一样，为社区做一些贡献。<br>直接从启动命令看下去吗？还有就是 treeindex和bbolt有必要看吗。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 在上一个问题，我给了一些要点，建议可以先从早期的v2代码看起，那时逻辑最简单，https:&#47;&#47;github.com&#47;etcd-io&#47;etcd&#47;blob&#47;release-0.4&#47;server&#47;v2&#47;get_handler.go,然后再看etcd v3的代码，在这个过程中，我给你几个小建议：<br>1. 抓住主次，比如核心读写流程是怎样的，忽略一些特殊细节<br>2. 看看测试用例如何使用核心模块的API的，比如etcd v3 mvcc的模块测试文件<br>https:&#47;&#47;github.com&#47;etcd-io&#47;etcd&#47;blob&#47;v3.4.9&#47;mvcc&#47;kv_test.go<br>3. 自己可动手写写源码分析<br>4. 自己多实践下，部署个单机etcd集群，至少要把etcdctl各个命令给操作下<br>5. 日志级别可以改成debug, 更加方便观察<br>如果给社区做贡献，可以多关注下社区正在讨论的issue和PR，寻找切入点，比如文档完善、拼写错误、不稳定的测试用例修复等开始，有时候大家会标注label help wanted, https:&#47;&#47;github.com&#47;etcd-io&#47;etcd&#47;issues?q=is%3Aopen+is%3Aissue+label%3A%22Help+Wanted%22<br><br>treeIndex核心是btree,boltdb核心是b+ tree,早期不建议你看，了解它的功能即可，当你对etcd有一定掌握后，可以详细看看，了解下生产级的btree及b+ tree实现，相信你会收货很多。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-24 09:25:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/46/2a/46c2774c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>孔宣</span>
  </div>
  <div class="_2_QraFYR_0">大佬，能给个联系方式吗？想请教一下kstone方面的问题，目前的问题主要是：<br>1、能否支持3.5.0版本的etcd集群新建？<br>2、是否支持指定不同的k8s集群来新建etcd集群？在dashbord上没看到这能力<br>3、目前支持的k8s版本相对老旧，是否有升级计划，我们也可以共建</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-07 15:58:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/46/2a/46c2774c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>孔宣</span>
  </div>
  <div class="_2_QraFYR_0">请问，kstone目前最新只支持etcd的3.4.13版本，但我们对etcd 3.5版本有强需求，支持3.5版本，需要做改造吗？盼复。。。微信加的kstone助手，没人理</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-06 20:40:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/11/3e/925aa996.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>HelloBug</span>
  </div>
  <div class="_2_QraFYR_0">一个客户端是一个Client，watch一个key是一个watcher，gRPC Watch Stream 数如何确定呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-19 07:28:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/11/3e/925aa996.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>HelloBug</span>
  </div>
  <div class="_2_QraFYR_0">&gt;其次是 goroutines 泄露导致内存占用高。此问题可能会在容器化场景中遇到。etcd 在打印日志的时候，若出现阻塞则可能会导致 goroutine 阻塞并持续泄露，最终导致内存泄露<br>老师，这里不太明白，为什么打印日志阻塞会导致内存泄露呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-19 07:18:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/60/ff/cbe8fb58.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>sxfworks</span>
  </div>
  <div class="_2_QraFYR_0">这几天看了etcd的源码，最开始一直好奇leader和follower怎么只同步一次snapshot，是我看错了，还是真的这么粗暴😂</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-29 16:37:05</div>
  </div>
</div>
</div>
</li>
</ul>