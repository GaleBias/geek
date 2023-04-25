<audio title="02 _ 基础架构：etcd一个读请求是如何执行的？" src="https://static001.geekbang.org/resource/audio/56/b4/56f71f418yy948b576de9d1a1248a6b4.mp3" controls="controls"></audio> 
<p>你好，我是唐聪。</p><p>在上一讲中，我和你分享了etcd的前世今生，同时也为你重点介绍了etcd  v2的不足之处，以及我们现在广泛使用etcd v3的原因。</p><p>今天，我想跟你介绍一下etcd v3的基础架构，让你从整体上对etcd有一个初步的了解，心中能构筑起一幅etcd模块全景图。这样，在你遇到诸如“Kubernetes在执行kubectl get pod时，etcd如何获取到最新的数据返回给APIServer？”等流程架构问题时，就能知道各个模块由上至下是如何紧密协作的。</p><p>即便是遇到请求报错，你也能通过顶层的模块全景图，推测出请求流程究竟在什么模块出现了问题。</p><h2>基础架构</h2><p>下面是一张etcd的简要基础架构图，我们先从宏观上了解一下etcd都有哪些功能模块。</p><p><img src="https://static001.geekbang.org/resource/image/34/84/34486534722d2748d8cd1172bfe63084.png?wh=1920*1240" alt=""></p><p>你可以看到，按照分层模型，etcd可分为Client层、API网络层、Raft算法层、逻辑层和存储层。这些层的功能如下：</p><ul>
<li>
<p><strong>Client层</strong>：Client层包括client v2和v3两个大版本API客户端库，提供了简洁易用的API，同时支持负载均衡、节点间故障自动转移，可极大降低业务使用etcd复杂度，提升开发效率、服务可用性。</p>
</li>
<li>
<p><strong>API网络层</strong>：API网络层主要包括client访问server和server节点之间的通信协议。一方面，client访问etcd server的API分为v2和v3两个大版本。v2 API使用HTTP/1.x协议，v3 API使用gRPC协议。同时v3通过etcd grpc-gateway组件也支持HTTP/1.x协议，便于各种语言的服务调用。另一方面，server之间通信协议，是指节点间通过Raft算法实现数据复制和Leader选举等功能时使用的HTTP协议。</p>
</li>
<li>
<p><strong>Raft算法层</strong>：Raft算法层实现了Leader选举、日志复制、ReadIndex等核心算法特性，用于保障etcd多个节点间的数据一致性、提升服务可用性等，是etcd的基石和亮点。</p>
</li>
<li>
<p><strong>功能逻辑层</strong>：etcd核心特性实现层，如典型的KVServer模块、MVCC模块、Auth鉴权模块、Lease租约模块、Compactor压缩模块等，其中MVCC模块主要由treeIndex模块和boltdb模块组成。</p>
</li>
<li>
<p><strong>存储层</strong>：存储层包含预写日志(WAL)模块、快照(Snapshot)模块、boltdb模块。其中WAL可保障etcd crash后数据不丢失，boltdb则保存了集群元数据和用户写入的数据。</p>
</li>
</ul><!-- [[[read_end]]] --><p>etcd是典型的读多写少存储，在我们实际业务场景中，读一般占据2/3以上的请求。为了让你对etcd有一个深入的理解，接下来我会分析一个读请求是如何执行的，带你了解etcd的核心模块，进而由点及线、由线到面地帮助你构建etcd的全景知识脉络。</p><p>在下面这张架构图中，我用序号标识了etcd默认读模式（线性读）的执行流程，接下来，我们就按照这个执行流程从头开始说。</p><p><img src="https://static001.geekbang.org/resource/image/45/bb/457db2c506135d5d29a93ef0bd97e4bb.png?wh=1920*1229" alt=""></p><h2>环境准备</h2><p>首先介绍一个好用的进程管理工具<a href="https://github.com/mattn/goreman">goreman</a>，基于它，我们可快速创建、停止本地的多节点etcd集群。</p><p>你可以通过如下<code>go get</code>命令快速安装goreman，然后从<a href="https://github.com/etcd-io/etcd/releases/v3.4.9">etcd release</a>页下载etcd v3.4.9二进制文件，再从<a href="https://github.com/etcd-io/etcd/blob/v3.4.9/Procfile">etcd源码</a>中下载goreman Procfile文件，它描述了etcd进程名、节点数、参数等信息。最后通过<code>goreman -f Procfile start</code>命令就可以快速启动一个3节点的本地集群了。</p><pre><code>go get github.com/mattn/goreman
</code></pre><h2>client</h2><p>启动完etcd集群后，当你用etcd的客户端工具etcdctl执行一个get hello命令（如下）时，对应到图中流程一，etcdctl是如何工作的呢？</p><pre><code>etcdctl get hello --endpoints http://127.0.0.1:2379  
hello  
world  
</code></pre><p>首先，etcdctl会对命令中的参数进行解析。我们来看下这些参数的含义，其中，参数“get”是请求的方法，它是KVServer模块的API；“hello”是我们查询的key名；“endpoints”是我们后端的etcd地址，通常，生产环境下中需要配置多个endpoints，这样在etcd节点出现故障后，client就可以自动重连到其它正常的节点，从而保证请求的正常执行。</p><p>在etcd v3.4.9版本中，etcdctl是通过clientv3库来访问etcd server的，clientv3库基于gRPC client API封装了操作etcd KVServer、Cluster、Auth、Lease、Watch等模块的API，同时还包含了负载均衡、健康探测和故障切换等特性。</p><p>在解析完请求中的参数后，etcdctl会创建一个clientv3库对象，使用KVServer模块的API来访问etcd server。</p><p>接下来，就需要为这个get hello请求选择一个合适的etcd server节点了，这里得用到负载均衡算法。在etcd 3.4中，clientv3库采用的负载均衡算法为Round-robin。针对每一个请求，Round-robin算法通过轮询的方式依次从endpoint列表中选择一个endpoint访问(长连接)，使etcd server负载尽量均衡。</p><p>关于负载均衡算法，你需要特别注意以下两点。</p><ol>
<li>如果你的client 版本&lt;= 3.3，那么当你配置多个endpoint时，负载均衡算法仅会从中选择一个IP并创建一个连接（Pinned endpoint），这样可以节省服务器总连接数。但在这我要给你一个小提醒，在heavy usage场景，这可能会造成server负载不均衡。</li>
<li>在client 3.4之前的版本中，负载均衡算法有一个严重的Bug：如果第一个节点异常了，可能会导致你的client访问etcd server异常，特别是在Kubernetes场景中会导致APIServer不可用。不过，该Bug已在 Kubernetes 1.16版本后被修复。</li>
</ol><p>为请求选择好etcd server节点，client就可调用etcd server的KVServer模块的Range RPC方法，把请求发送给etcd server。</p><p>这里我说明一点，client和server之间的通信，使用的是基于HTTP/2的gRPC协议。相比etcd v2的HTTP/1.x，HTTP/2是基于二进制而不是文本、支持多路复用而不再有序且阻塞、支持数据压缩以减少包大小、支持server push等特性。因此，基于HTTP/2的gRPC协议具有低延迟、高性能的特点，有效解决了我们在上一讲中提到的etcd v2中HTTP/1.x 性能问题。</p><h2>KVServer</h2><p>client发送Range RPC请求到了server后，就开始进入我们架构图中的流程二，也就是KVServer模块了。</p><p>etcd提供了丰富的metrics、日志、请求行为检查等机制，可记录所有请求的执行耗时及错误码、来源IP等，也可控制请求是否允许通过，比如etcd Learner节点只允许指定接口和参数的访问，帮助大家定位问题、提高服务可观测性等，而这些特性是怎么非侵入式的实现呢？</p><p>答案就是拦截器。</p><h3>拦截器</h3><p>etcd server定义了如下的Service KV和Range方法，启动的时候它会将实现KV各方法的对象注册到gRPC Server，并在其上注册对应的拦截器。下面的代码中的Range接口就是负责读取etcd key-value的的RPC接口。</p><pre><code>service KV {  
  // Range gets the keys in the range from the key-value store.  
  rpc Range(RangeRequest) returns (RangeResponse) {  
      option (google.api.http) = {  
        post: &quot;/v3/kv/range&quot;  
        body: &quot;*&quot;  
      };  
  }  
  ....
}  
</code></pre><p>拦截器提供了在执行一个请求前后的hook能力，除了我们上面提到的debug日志、metrics统计、对etcd Learner节点请求接口和参数限制等能力，etcd还基于它实现了以下特性:</p><ul>
<li>要求执行一个操作前集群必须有Leader；</li>
<li>请求延时超过指定阈值的，打印包含来源IP的慢查询日志(3.5版本)。</li>
</ul><p>server收到client的Range RPC请求后，根据ServiceName和RPC Method将请求转发到对应的handler实现，handler首先会将上面描述的一系列拦截器串联成一个执行，在拦截器逻辑中，通过调用KVServer模块的Range接口获取数据。</p><h3>串行读与线性读</h3><p>进入KVServer模块后，我们就进入核心的读流程了，对应架构图中的流程三和四。我们知道etcd为了保证服务高可用，生产环境一般部署多个节点，那各个节点数据在任意时间点读出来都是一致的吗？什么情况下会读到旧数据呢？</p><p>这里为了帮助你更好的理解读流程，我先简单提下写流程。如下图所示，当client发起一个更新hello为world请求后，若Leader收到写请求，它会将此请求持久化到WAL日志，并广播给各个节点，若一半以上节点持久化成功，则该请求对应的日志条目被标识为已提交，etcdserver模块异步从Raft模块获取已提交的日志条目，应用到状态机(boltdb等)。</p><p><img src="https://static001.geekbang.org/resource/image/cf/d5/cffba70a79609f29e1f2ae1f3bd07fd5.png?wh=1920*1074" alt=""></p><p>此时若client发起一个读取hello的请求，假设此请求直接从状态机中读取， 如果连接到的是C节点，若C节点磁盘I/O出现波动，可能导致它应用已提交的日志条目很慢，则会出现更新hello为world的写命令，在client读hello的时候还未被提交到状态机，因此就可能读取到旧数据，如上图查询hello流程所示。</p><p>从以上介绍我们可以看出，在多节点etcd集群中，各个节点的状态机数据一致性存在差异。而我们不同业务场景的读请求对数据是否最新的容忍度是不一样的，有的场景它可以容忍数据落后几秒甚至几分钟，有的场景要求必须读到反映集群共识的最新数据。</p><p>我们首先来看一个<strong>对数据敏感度较低的场景</strong>。</p><p>假如老板让你做一个旁路数据统计服务，希望你每分钟统计下etcd里的服务、配置信息等，这种场景其实对数据时效性要求并不高，读请求可直接从节点的状态机获取数据。即便数据落后一点，也不影响业务，毕竟这是一个定时统计的旁路服务而已。</p><p>这种直接读状态机数据返回、无需通过Raft协议与集群进行交互的模式，在etcd里叫做<strong>串行(<strong><strong>Serializable</strong></strong>)读</strong>，它具有低延时、高吞吐量的特点，适合对数据一致性要求不高的场景。</p><p>我们再看一个<strong>对数据敏感性高的场景</strong>。</p><p>当你发布服务，更新服务的镜像的时候，提交的时候显示更新成功，结果你一刷新页面，发现显示的镜像的还是旧的，再刷新又是新的，这就会导致混乱。再比如说一个转账场景，Alice给Bob转账成功，钱被正常扣出，一刷新页面发现钱又回来了，这也是令人不可接受的。</p><p>以上的业务场景就对数据准确性要求极高了，在etcd里面，提供了一种线性读模式来解决对数据一致性要求高的场景。</p><p><strong>什么是线性读呢?</strong></p><p>你可以理解一旦一个值更新成功，随后任何通过线性读的client都能及时访问到。虽然集群中有多个节点，但client通过线性读就如访问一个节点一样。etcd默认读模式是线性读，因为它需要经过Raft协议模块，反应的是集群共识，因此在延时和吞吐量上相比串行读略差一点，适用于对数据一致性要求高的场景。</p><p>如果你的etcd读请求显示指定了是串行读，就不会经过架构图流程中的流程三、四。默认是线性读，因此接下来我们看看读请求进入线性读模块，它是如何工作的。</p><h3>线性读之ReadIndex</h3><p>前面我们聊到串行读时提到，它之所以能读到旧数据，主要原因是Follower节点收到Leader节点同步的写请求后，应用日志条目到状态机是个异步过程，那么我们能否有一种机制在读取的时候，确保最新的数据已经应用到状态机中？</p><p><img src="https://static001.geekbang.org/resource/image/1c/cc/1c065788051c6eaaee965575a04109cc.png?wh=1920*1095" alt=""></p><p>其实这个机制就是叫ReadIndex，它是在etcd 3.1中引入的，我把简化后的原理图放在了上面。当收到一个线性读请求时，它首先会从Leader获取集群最新的已提交的日志索引(committed index)，如上图中的流程二所示。</p><p>Leader收到ReadIndex请求时，为防止脑裂等异常场景，会向Follower节点发送心跳确认，一半以上节点确认Leader身份后才能将已提交的索引(committed index)返回给节点C(上图中的流程三)。</p><p>C节点则会等待，直到状态机已应用索引(applied index)大于等于Leader的已提交索引时(committed Index)(上图中的流程四)，然后去通知读请求，数据已赶上Leader，你可以去状态机中访问数据了(上图中的流程五)。</p><p>以上就是线性读通过ReadIndex机制保证数据一致性原理， 当然还有其它机制也能实现线性读，如在早期etcd 3.0中读请求通过走一遍Raft协议保证一致性， 这种Raft log read机制依赖磁盘IO， 性能相比ReadIndex较差。</p><p>总体而言，KVServer模块收到线性读请求后，通过架构图中流程三向Raft模块发起ReadIndex请求，Raft模块将Leader最新的已提交日志索引封装在流程四的ReadState结构体，通过channel层层返回给线性读模块，线性读模块等待本节点状态机追赶上Leader进度，追赶完成后，就通知KVServer模块，进行架构图中流程五，与状态机中的MVCC模块进行进行交互了。</p><h2>MVCC</h2><p>流程五中的多版本并发控制(Multiversion concurrency control)模块是为了解决上一讲我们提到etcd v2不支持保存key的历史版本、不支持多key事务等问题而产生的。</p><p>它核心由内存树形索引模块(treeIndex)和嵌入式的KV持久化存储库boltdb组成。</p><p>首先我们需要简单了解下boltdb，它是个基于B+ tree实现的key-value键值库，支持事务，提供Get/Put等简易API给etcd操作。</p><p>那么etcd如何基于boltdb保存一个key的多个历史版本呢?</p><p>比如我们现在有以下方案：方案1是一个key保存多个历史版本的值；方案2每次修改操作，生成一个新的版本号(revision)，以版本号为key， value为用户key-value等信息组成的结构体。</p><p>很显然方案1会导致value较大，存在明显读写放大、并发冲突等问题，而方案2正是etcd所采用的。boltdb的key是全局递增的版本号(revision)，value是用户key、value等字段组合成的结构体，然后通过treeIndex模块来保存用户key和版本号的映射关系。</p><p>treeIndex与boltdb关系如下面的读事务流程图所示，从treeIndex中获取key hello的版本号，再以版本号作为boltdb的key，从boltdb中获取其value信息。</p><p><img src="https://static001.geekbang.org/resource/image/4e/a3/4e2779c265c1da1f7209b5293e3789a3.png?wh=1920*1124" alt=""></p><h3>treeIndex</h3><p>treeIndex模块是基于Google开源的内存版btree库实现的，为什么etcd选择上图中的B-tree数据结构保存用户key与版本号之间的映射关系，而不是哈希表、二叉树呢？在后面的课程中我会再和你介绍。</p><p>treeIndex模块只会保存用户的key和相关版本号信息，用户key的value数据存储在boltdb里面，相比ZooKeeper和etcd v2全内存存储，etcd v3对内存要求更低。</p><p>简单介绍了etcd如何保存key的历史版本后，架构图中流程六也就非常容易理解了， 它需要从treeIndex模块中获取hello这个key对应的版本号信息。treeIndex模块基于B-tree快速查找此key，返回此key对应的索引项keyIndex即可。索引项中包含版本号等信息。</p><h3>buffer</h3><p>在获取到版本号信息后，就可从boltdb模块中获取用户的key-value数据了。不过有一点你要注意，并不是所有请求都一定要从boltdb获取数据。</p><p>etcd出于数据一致性、性能等考虑，在访问boltdb前，首先会从一个内存读事务buffer中，二分查找你要访问key是否在buffer里面，若命中则直接返回。</p><h3>boltdb</h3><p>若buffer未命中，此时就真正需要向boltdb模块查询数据了，进入了流程七。</p><p>我们知道MySQL通过table实现不同数据逻辑隔离，那么在boltdb是如何隔离集群元数据与用户数据的呢？答案是bucket。boltdb里每个bucket类似对应MySQL一个表，用户的key数据存放的bucket名字的是key，etcd MVCC元数据存放的bucket是meta。</p><p>因boltdb使用B+ tree来组织用户的key-value数据，获取bucket key对象后，通过boltdb的游标Cursor可快速在B+ tree找到key hello对应的value数据，返回给client。</p><p>到这里，一个读请求之路执行完成。</p><h2>小结</h2><p>最后我们来小结一下，一个读请求从client通过Round-robin负载均衡算法，选择一个etcd server节点，发出gRPC请求，经过etcd server的KVServer模块、线性读模块、MVCC的treeIndex和boltdb模块紧密协作，完成了一个读请求。</p><p>通过一个读请求，我带你初步了解了etcd的基础架构以及各个模块之间是如何协作的。</p><p>在这过程中，我想和你特别总结下client的节点故障自动转移和线性读。</p><p>一方面， client的通过负载均衡、错误处理等机制实现了etcd节点之间的故障的自动转移，它可助你的业务实现服务高可用，建议使用etcd 3.4分支的client版本。</p><p>另一方面，我详细解释了etcd提供的两种读机制(串行读和线性读)原理和应用场景。通过线性读，对业务而言，访问多个节点的etcd集群就如访问一个节点一样简单，能简洁、快速的获取到集群最新共识数据。</p><p>早期etcd线性读使用的Raft log read，也就是说把读请求像写请求一样走一遍Raft的协议，基于Raft的日志的有序性，实现线性读。但此方案读涉及磁盘IO开销，性能较差，后来实现了ReadIndex读机制来提升读性能，满足了Kubernetes等业务的诉求。</p><h2>思考题</h2><p>etcd在执行读请求过程中涉及磁盘IO吗？如果涉及，是什么模块在什么场景下会触发呢？如果不涉及，又是什么原因呢？</p><p>你可以把你的思考和观点写在留言区里，我会在下一节课里给出我的答案。</p><p>感谢你阅读，也欢迎你把这篇文章分享给更多的朋友一起阅读，我们下节课见。</p>
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
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/4S6xTsiauPNbQrEHiayUVNvNXgl1WR4BFwvuJbPbGicSzpbYeKNGicPJ8RiaibAGZEDLcicJRibGQNUqfjs2t90EBPK9Pg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>hiroshi</span>
  </div>
  <div class="_2_QraFYR_0">老师，readIndex 需要请求 leader，那为啥不直接让 leader 返回读请求的结果，而要等待自己的进度赶上 leader？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 非常好的问题，我个人认为主要还是性能因素，我记得etcd v2早期的时候如果你指定线性读&#47;共识读，它就是直接转发给leader的。后来在etcd v3.0中实现了raft log read但是要走一遍raft log，读涉及到磁盘IO，v3.1中引入了readIndex机制，它是非常轻量级的，开销较小，相比各个follower都转发给leader会导致leader负载较高，特别是expensive request场景，性能会急剧下降，leader的内存、cpu、网络带宽资源都很容易耗尽，readIndex机制的引入，使得每个follower节点都可以处理读请求，极大扩展提升了写性能。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-16 15:53:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/18/4c/e12f3b41.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>姜姜</span>
  </div>
  <div class="_2_QraFYR_0">老师，文中有些地方不太明白:<br>1, KVServer中的拦截器<br>我认为它只是作为一个辅助的功能吧，用于实现一些观测功能。但对于一个普通的读请求，是否必须通过拦截器才能完成读取数据的操作？<br><br>2, 文中“handler 首先会将上面描述的一系列拦截器串联成一个执行”<br>这段话中，拦截器是一系列的，一系列是指会有多个拦截器吗？难道不是一个请求只注册一个拦截器吗，还能注册多个？为什么要注册多个？<br>“串联成一个执行”，如何串联成一个？将多个拦截器串联成一个拦截器？<br><br>3, 串行读与线性读<br>这里我理解串行读是“非强一致性读”，线性读是“强一致性读”，对吗？<br>而且这里的“串行”总让我想到“并行&#47;串行”的概念，不知有关系吗？<br><br>4, ReadIndex，committed index，applied index<br>这几种索引底层实现是一样的吗，它们的数据结构是怎样的？是对同一份数据，分别建立不同的索引？又为什么建立这么多种索引？<br><br>5，版本号<br>您说是一个递增的全局ID， revision{2, 0}，ID指的是2还是0？ 版本号的格式是怎样的，另一个数字代表什么？<br><br>6,  bucket<br>请问一个 bucket 相当于一整个 B+ tree 索引树吗？还是相当于 B+ tree 中一个节点？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢你的提问，我先简单快速回答下，后面不清楚的再写答疑文章深入解答<br>问题1和2是gRPC拦截器相关知识我推荐你看下这篇文章https:&#47;&#47;zhuanlan.zhihu.com&#47;p&#47;80023990<br>问题3你理解串行读是“非强一致性读”，线性读是“强一致性读”没问题，至于串行含义并非你想的那样，你可以参考下维基百科的定义, 09事务篇我也会介绍事务隔离中的串行化<br>https:&#47;&#47;en.wikipedia.org&#47;wiki&#47;Serializability<br>问题4 建议先去阅读下04 raft篇，它本值上就是一个uint64的索引，表示日志条目序号<br>问题5，{2,0}={major,sub} 2是etcd mvcc事务版本号全局递增，0是事务内子版本号随修改操作递增（比如一个txn事务中多个put&#47;delete操作，其会从0递增)，07 mvcc会详细介绍<br>问题6，一个bucket对应一个颗B+tree</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-26 13:37:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/yb5R8iaxicD8sfspaUaqMDpOopzqGjcnqxI83kxJcDlOUcdHTPP8wx6PzEiaNvl5Sf3CuMtU6r1Jzf3M0AQuef96w/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小军</span>
  </div>
  <div class="_2_QraFYR_0">请问老师，当Readindex结束并等待本节点的状态机apply的时候，key又被最新的更新请求给更新了怎么办，这个时候读取到的value是不是又是旧值了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 线性读，读出来的值实际上是你发出读请求时间点的集群最新共识数据，在你读请求发出后，若耗时一定时间还未完成，在这过程中leader又收到了写请求更新了它,   的确你原来读出来的值相比最新的集群共识就是旧的，在实际应用中，我们一般会通过增加版本号检测识别此类问题，后面事务篇会详细和你介绍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-26 09:56:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/53/c4/dea5d7f3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>chapin</span>
  </div>
  <div class="_2_QraFYR_0">没有基础，学习这个，可能会比较吃力。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 没关系的，可以先大概看一篇，了解整个流程，不懂什么地方可以等学完后面后，回过头来再看就非常亲切了，后面每节中都有etcd特性体验案例，建议你跟着我一起实际操作下，比如02你就先准备好环境，能用goreman快速启一个多节点集群，也可以自己直接二进制启动一个单节点集群，然后体验一下get，put命令，随着后面的学习你会越来越了解etcd</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-26 10:10:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/22/28/0f/e0abc71b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>站在树上的松鼠</span>
  </div>
  <div class="_2_QraFYR_0">老师，下面这句话没有理解到，麻烦解答下呢，谢谢！<br>在client 3.4之前的版本中，负载均衡算法有一个严重的Bug：如果第一个节点异常了，可能会导致你的client访问etcd server异常。<br>	（1）这里第一个节点怎么理解呢？ 是指的负载均衡刚好选中的那个etcd server节点异常吗？<br>	（2）如果访问的节点异常了，是client库中会做重试机制，还是业务代码需要做重试呢？<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢超凡帮忙解答第一点，第二点取决于rpc方法，range clientv3库有重试策略，参考一下这个文件clientv3&#47;retry.go</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-22 20:28:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/9d/84/171b2221.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jeffery</span>
  </div>
  <div class="_2_QraFYR_0">干货太多需要慢慢消化！老师能把课程代码放到github上吗……谢谢老师</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，不清楚的地方不要急，后面的每节会帮助你一个个解开疑问</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-22 13:13:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/a5/34/6e3e962f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>yayiyaya</span>
  </div>
  <div class="_2_QraFYR_0">问答： etcd 在执行读请求过程中涉及磁盘 IO 吗？<br>答： 涉及到磁盘， 当读请求从treeIndex获取到用户的 key 和相关版本号信息后，去查询value值时， 没有命中 buffer， 会从boltdb获取数据， 这个时候就涉及到了磁盘。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: etcd启动的时候通过mmap将db文件映射到内存，会告诉内核预读文件，下一讲给了参考答案</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-04 15:06:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/0f/e9/564eaf5b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Want less</span>
  </div>
  <div class="_2_QraFYR_0">当收到一个线性读请求时，它首先会从 Leader 获取集群最新的已提交的日志索引 (committed index)。<br>所有的client请求不是应该都通过leader下发至follower吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不是的哈，follower节点也可以处理读请求的，只是线性读时需要向leader发送readindex消息，然后确保本节点数据是最新的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-27 15:12:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/a5/cd/3aff5d57.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Alery</span>
  </div>
  <div class="_2_QraFYR_0">请教一个问题，在treeIndex中查询key对应的版本号，这里是会返回当前key的所有版本号吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，每个key在treeIndex中有一个对应的数据结构keyIndex,它保存了所有版本号(若未压缩)，07讲mvcc将详细介绍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-26 19:39:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/78/df/424bdc4a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>于途</span>
  </div>
  <div class="_2_QraFYR_0">如果你的 client 版本 &lt;= 3.3，那么当你配置多个 endpoint 时，负载均衡算法仅会从中选择一个 IP 并创建一个连接（Pinned endpoint）<br><br>请问，此句提到的负载均衡算法是否等同：随机选中某个IP？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，可以理解为随机，首先它会尝试连接所有etcd节点，连接建立后选择一个固定的长连接，其他关闭</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-22 17:19:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/5f/e5/54325854.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>范闲</span>
  </div>
  <div class="_2_QraFYR_0">wal log里面会涉及到磁盘读写。lsm树，双memtable，都满了刷到磁盘，继续写memtable.</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 读请求不涉及wal log，读流程中你可以看看不需要它，写请求会介绍，lsm树是leveldb使用的存储模型，etcd使用的是b+tree。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-22 09:27:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_ddfeca</span>
  </div>
  <div class="_2_QraFYR_0">有一点没明白，文中提到：C 节点则会等待，直到状态机已应用索引 (applied index) 大于等于 Leader 的已提交索引时 (committed Index)(上图中的流程四)，然后去通知读请求，数据已赶上 Leader，你可以去状态机中访问数据了 (上图中的流程五)。<br>但12中提到，“etcd 无论 Apply 流程是成功还是失败，都会更新 raftAppliedIndex 值&quot;。那岂不是即使apply失败了，也会更新raftAppliedIndex ，但其实follower并没真正赶上leader，读到的还是旧数据?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，这时候就会产生数据不一致，apply失败的后果是非常严重的，正常情况下不会出现这个问题。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-09 11:40:01</div>
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
  <div class="_2_QraFYR_0">boltdb怎么保证全局的revision呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: boltdb的key是revision，revision本身由etcd mvcc模块维护全局单调递增</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-26 17:49:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/d4/9b/00f6c7f8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>no-one</span>
  </div>
  <div class="_2_QraFYR_0">如果读之前follower节点的索引已经是最新的了，还会先去leader节点读readindex吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，follower节点是无法确认自己是否最新的，数据是leader向follower同步的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-23 16:41:54</div>
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
  <div class="_2_QraFYR_0">另外还想问下唐老师，是否会有章节介绍下etcd集群管理和请求路由的原理，比如节点如何探活及增减，及像在线性读场景里，请求是否一定要通过已提交的quorum内节点处理还是任何节点都可以处理呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 好的，第一点，加餐篇计划增加集群成员管理等内容，第二点，任何follower节点都可以处理读请求，如果是线性读，它们都会向leader节点发出readindex请求，然后等待本节点数据赶上leader，如果一个follower节点io异常，数据落后多就可能超时</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-22 17:36:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/96/47/93838ff7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>青鸟飞鱼</span>
  </div>
  <div class="_2_QraFYR_0">老师，如果ReadIndex读过程中，流程4状态机迟迟不应用索引？或者流程5中，未能通知到读请求？这些情况会换节点读还是幂等重试呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 参考我上面的评论，超时后依赖客户端重试，3.4中round-robin负载均衡算法重试后就会选择另外一个节点</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-22 08:49:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/46/ac/9c324436.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>抖腿冠军</span>
  </div>
  <div class="_2_QraFYR_0">这个goreman  怎么搞不下来？ 只有我一个人搞不下来？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: https:&#47;&#47;github.com&#47;mattn&#47;goreman<br>go install github.com&#47;mattn&#47;goreman@latest<br>具体遇到了什么问题吗</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-13 21:21:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/ib3Rzem884S5MOS96THy0gQXcF26PNsnRBpyr3pM5rVibZdYvAibpVvAGfibF1ddpgrteg9fQUsq4vce9EM95Jj97Q/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_604077</span>
  </div>
  <div class="_2_QraFYR_0">环境准备中踩坑日记<br>	我用的是etcd Version: 3.5.0版本<br>	1、需要将etcd解压文件夹下的bin文件目录设置到环境变量中，<br>	   官网文档修改环境变量用的是export的方式，该方式只会在<br>	   当前会话生效（难怪我打开别的终端窗口就执行不了etcd的<br>	   命令）可以通过sudo vim .&#47;bash_profile的方式添加<br>	   全局才会生效<br>	2、文稿中的下载 goreman Procfile的文件需要做一点修改把<br>	“bin&#47;etcd”改成“etcd”即可因为我们刚刚设置了全局的环境变<br>	量<br>	3、etcdctl get hello --endpoints http:&#47;&#47;127.0.0.1:2379<br>	没有输出任何内容，这个人猜测是因为版本于老师版本不一致的原因<br>	我用官网快速开始文档教程动手试了试有输出正常的hello etcd的<br>	（以上是我在安装etcd踩的坑，因为我比较小白，贴出来印象深刻些，<br>	毕竟老师的课堂作业做不出来，也不想空着）<br><br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-24 00:27:42</div>
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
  <div class="_2_QraFYR_0">从状态机里读数据是什么意思呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你可以理解为从架构图中的mvcc模块(treeindex&#47;boltdb)中读取.</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-02 17:24:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/e1/4f/00476b4c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Remember九离</span>
  </div>
  <div class="_2_QraFYR_0">有一个问题，当我发起一个线性读的时候，此时Leader 发生脑裂，这时候etcd是咋么处理的，响应异常？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 从设计上严格来说，etcd不存在脑裂，除非实现有问题。如果发生了网络分区，假设client与旧leader都处于网络分区节点小于n&#47;2+1的一侧，那么leader收到线性读请求后，在向多数follower节点发送心跳确认leader身份权威性的时候，就会失败，进而会报错给client.</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-29 09:18:06</div>
  </div>
</div>
</div>
</li>
</ul>