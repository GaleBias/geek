<audio title="01 _ etcd的前世今生：为什么Kubernetes使用etcd？" src="https://static001.geekbang.org/resource/audio/yy/36/yy3d617dbf6yycfeb065064193785436.mp3" controls="controls"></audio> 
<p>你好，我是唐聪。</p><p>今天是专栏课程的第一讲，我们就从etcd的前世今生讲起。让我们一起穿越回2013年，看看etcd最初是在什么业务场景下被设计出来的？</p><p>2013年，有一个叫CoreOS的创业团队，他们构建了一个产品，Container Linux，它是一个开源、轻量级的操作系统，侧重自动化、快速部署应用服务，并要求应用程序都在容器中运行，同时提供集群化的管理方案，用户管理服务就像单机一样方便。</p><p>他们希望在重启任意一节点的时候，用户的服务不会因此而宕机，导致无法提供服务，因此需要运行多个副本。但是多个副本之间如何协调，如何避免变更的时候所有副本不可用呢？</p><p>为了解决这个问题，CoreOS团队需要一个协调服务来存储服务配置信息、提供分布式锁等能力。怎么办呢？当然是分析业务场景、痛点、核心目标，然后是基于目标进行方案选型，评估是选择社区开源方案还是自己造轮子。这其实就是我们遇到棘手问题时的通用解决思路，CoreOS团队同样如此。</p><p>假设你是CoreOS团队成员，你认为在这样的业务场景下，理想中的解决方案应满足哪些目标呢？</p><p>如果你有过一些开发经验，应该能想到一些关键点了，我根据自己的经验来总结一下，一个协调服务，理想状态下大概需要满足以下五个目标：</p><!-- [[[read_end]]] --><ol>
<li><strong>可用性角度：高可用</strong>。协调服务作为集群的控制面存储，它保存了各个服务的部署、运行信息。若它故障，可能会导致集群无法变更、服务副本数无法协调。业务服务若此时出现故障，无法创建新的副本，可能会影响用户数据面。</li>
<li><strong>数据一致性角度：提供读取“最新”数据的机制</strong>。既然协调服务必须具备高可用的目标，就必然不能存在单点故障（single point of failure），而多节点又引入了新的问题，即多个节点之间的数据一致性如何保障？比如一个集群3个节点A、B、C，从节点A、B获取服务镜像版本是新的，但节点C因为磁盘 I/O异常导致数据更新缓慢，若控制端通过C节点获取数据，那么可能会导致读取到过期数据，服务镜像无法及时更新。</li>
<li><strong>容量角度：低容量、仅存储关键元数据配置。</strong>协调服务保存的仅仅是服务、节点的配置信息（属于控制面配置），而不是与用户相关的数据。所以存储上不需要考虑数据分片，无需过度设计。</li>
<li><strong>功能：增删改查，监听数据变化的机制</strong>。协调服务保存了服务的状态信息，若服务有变更或异常，相比控制端定时去轮询检查一个个服务状态，若能快速推送变更事件给控制端，则可提升服务可用性、减少协调服务不必要的性能开销。</li>
<li><strong>运维复杂度：可维护性。</strong>在分布式系统中往往会遇到硬件Bug、软件Bug、人为操作错误导致节点宕机，以及新增、替换节点等运维场景，都需要对协调服务成员进行变更。若能提供API实现平滑地变更成员节点信息，就可以大大降低运维复杂度，减少运维成本，同时可避免因人工变更不规范可能导致的服务异常。</li>
</ol><p>了解完理想中的解决方案目标，我们再来看CoreOS团队当时为什么选择了从0到1开发一个新的协调服务呢？</p><p>如果使用开源软件，当时其实是有ZooKeeper的，但是他们为什么不用ZooKeeper呢？我们来分析一下。</p><p>从高可用性、数据一致性、功能这三个角度来说，ZooKeeper是满足CoreOS诉求的。然而当时的ZooKeeper不支持通过API安全地变更成员，需要人工修改一个个节点的配置，并重启进程。</p><p>若变更姿势不正确，则有可能出现脑裂等严重故障。适配云环境、可平滑调整集群规模、在线变更运行时配置是CoreOS的期望目标，而ZooKeeper在这块的可维护成本相对较高。</p><p>其次ZooKeeper是用 Java 编写的，部署较繁琐，占用较多的内存资源，同时ZooKeeper RPC的序列化机制用的是Jute，自己实现的RPC API。无法使用curl之类的常用工具与之互动，CoreOS期望使用比较简单的HTTP + JSON。</p><p>因此，CoreOS决定自己造轮子，那CoreOS团队是如何根据系统目标进行技术方案选型的呢？</p><h2>etcd v1和v2诞生</h2><p>首先我们来看服务高可用及数据一致性。前面我们提到单副本存在单点故障，而多副本又引入数据一致性问题。</p><p>因此为了解决数据一致性问题，需要引入一个共识算法，确保各节点数据一致性，并可容忍一定节点故障。常见的共识算法有Paxos、ZAB、Raft等。CoreOS团队选择了易理解实现的Raft算法，它将复杂的一致性问题分解成Leader选举、日志同步、安全性三个相对独立的子问题，只要集群一半以上节点存活就可提供服务，具备良好的可用性。</p><p>其次我们再来看数据模型（Data Model）和API。数据模型参考了ZooKeeper，使用的是基于目录的层次模式。API相比ZooKeeper来说，使用了简单、易用的REST API，提供了常用的Get/Set/Delete/Watch等API，实现对key-value数据的查询、更新、删除、监听等操作。</p><p>key-value存储引擎上，ZooKeeper使用的是Concurrent HashMap，而etcd使用的是则是简单内存树，它的节点数据结构精简后如下，含节点路径、值、孩子节点信息。这是一个典型的低容量设计，数据全放在内存，无需考虑数据分片，只能保存key的最新版本，简单易实现。</p><p><img src="https://static001.geekbang.org/resource/image/ff/41/ff4ee032739b9b170af1b2e2ba530e41.png?wh=1024*904" alt=""></p><pre><code>type node struct {
   Path string  //节点路径
   Parent *node //关联父亲节点
   Value      string     //key的value值
   ExpireTime time.Time //过期时间
   Children   map[string]*node //此节点的孩子节点
}
</code></pre><p>最后我们再来看可维护性。Raft算法提供了成员变更算法，可基于此实现成员在线、安全变更，同时此协调服务使用Go语言编写，无依赖，部署简单。</p><p><img src="https://static001.geekbang.org/resource/image/dd/70/dd253e4fc19885fa6f00c278762ba270.png?wh=1920*1402" alt=""></p><p>基于以上技术方案和架构图，CoreOS团队在2013年8月对外发布了第一个测试版本v0.1，API v1版本，命名为etcd。</p><p>那么etcd这个名字是怎么来的呢？其实它源于两个方面，unix的“/etc”文件夹和分布式系统(“D”istribute system)的D，组合在一起表示etcd是用于存储分布式配置的信息存储服务。</p><p>v0.1版本实现了简单的HTTP Get/Set/Delete/Watch API，但读数据一致性无法保证。v0.2版本，支持通过指定consistent模式，从Leader读取数据，并将Test And Set机制修正为CAS(Compare And Swap)，解决原子更新的问题，同时发布了新的API版本v2，这就是大家熟悉的etcd v2版本，第一个非stable版本。</p><p>下面，我用一幅时间轴图，给你总结一下etcd v1/v2关键特性。</p><p><img src="https://static001.geekbang.org/resource/image/d0/0e/d0af3537c0eef89b499a82693da23f0e.png?wh=1920*727" alt=""></p><h2>为什么Kubernetes使用etcd?</h2><p>这张图里，我特别标注出了Kubernetes的发布时间点，这个非常关键。我们必须先来说说这个事儿，也就是Kubernetes和etcd的故事。</p><p>2014年6月，Google的Kubernetes项目诞生了，我们前面所讨论到Go语言编写、etcd高可用、Watch机制、CAS、TTL等特性正是Kubernetes所需要的，它早期的0.4版本，使用的正是etcd v0.2版本。</p><p>Kubernetes是如何使用etcd v2这些特性的呢？举几个简单小例子。</p><p>当你使用Kubernetes声明式API部署服务的时候，Kubernetes的控制器通过etcd Watch机制，会实时监听资源变化事件，对比实际状态与期望状态是否一致，并采取协调动作使其一致。Kubernetes更新数据的时候，通过CAS机制保证并发场景下的原子更新，并通过对key设置TTL来存储Event事件，提升Kubernetes集群的可观测性，基于TTL特性，Event事件key到期后可自动删除。</p><p>Kubernetes项目使用etcd，除了技术因素也与当时的商业竞争有关。CoreOS是Kubernetes容器生态圈的核心成员之一。</p><p>当时Docker容器浪潮正席卷整个开源技术社区，CoreOS也将容器集成到自家产品中。一开始与Docker公司还是合作伙伴，然而Docker公司不断强化Docker的PaaS平台能力，强势控制Docker社区，这与CoreOS核心商业战略出现了冲突，也损害了Google、RedHat等厂商的利益。</p><p>最终CoreOS与Docker分道扬镳，并推出了rkt项目来对抗Docker，然而此时Docker已深入人心，CoreOS被Docker全面压制。</p><p>以Google、RedHat为首的阵营，基于Google多年的大规模容器管理系统Borg经验，结合社区的建议和实践，构建以Kubernetes为核心的容器生态圈。相比Docker的垄断、独裁，Kubernetes社区推行的是民主、开放原则，Kubernetes每一层都可以通过插件化扩展，在Google、RedHat的带领下不断发展壮大，etcd也进入了快速发展期。</p><p>在2015年1月，CoreOS发布了etcd第一个稳定版本2.0，支持了quorum read，提供了严格的线性一致性读能力。7月，基于etcd 2.0的Kubernetes第一个生产环境可用版本v1.0.1发布了，Kubernetes开始了新的里程碑的发展。</p><p>etcd v2在社区获得了广泛关注，GitHub star数在2015年6月就高达6000+，超过500个项目使用，被广泛应用于配置存储、服务发现、主备选举等场景。</p><p>下图我从构建分布式系统的核心要素角度，给你总结了etcd v2核心技术点。无论是NoSQL存储还是SQL存储、文档存储，其实大家要解决的问题都是类似的，基本就是图中总结的数据模型、复制、共识算法、API、事务、一致性、成员故障检测等方面。</p><p>希望通过此图帮助你了解从0到1如何构建、学习一个分布式系统，要解决哪些技术点，在心中有个初步认识，后面的课程中我会再深入介绍。</p><p><img src="https://static001.geekbang.org/resource/image/cd/f0/cde3f155f51bfd3d7fd78fe8e7ac9bf0.png?wh=1920*943" alt=""></p><h2>etcd v3诞生</h2><p>然而随着Kubernetes项目不断发展，v2版本的瓶颈和缺陷逐渐暴露，遇到了若干性能和稳定性问题，Kubernetes社区呼吁支持新的存储、批评etcd不可靠的声音开始不断出现。</p><p>具体有哪些问题呢？我给你总结了如下图：</p><p><img src="https://static001.geekbang.org/resource/image/88/d1/881db1b7d05dc40771e9737f3117f5d1.png?wh=1920*577" alt=""></p><p>下面我分别从功能局限性、Watch事件的可靠性、性能、内存开销来分别给你剖析etcd v2的问题。</p><p>首先是<strong>功能局限性问题。</strong>它主要是指etcd v2不支持范围和分页查询、不支持多key事务。</p><p>第一，etcd v2不支持范围查询和分页。分页对于数据较多的场景是必不可少的。在Kubernetes中，在集群规模增大后，Pod、Event等资源可能会出现数千个以上，但是etcd v2不支持分页，不支持范围查询，大包等expensive request会导致严重的性能乃至雪崩问题。</p><p>第二，etcd v2不支持多key事务。在实际转账等业务场景中，往往我们需要在一个事务中同时更新多个key。</p><p>然后是<strong>Watch机制可靠性问题</strong>。Kubernetes项目严重依赖etcd Watch机制，然而etcd v2是内存型、不支持保存key历史版本的数据库，只在内存中使用滑动窗口保存了最近的1000条变更事件，当etcd server写请求较多、网络波动时等场景，很容易出现事件丢失问题，进而又触发client数据全量拉取，产生大量expensive request，甚至导致etcd雪崩。</p><p>其次是<strong>性能瓶颈问题</strong>。etcd v2早期使用了简单、易调试的HTTP/1.x API，但是随着Kubernetes支撑的集群规模越来越大，HTTP/1.x协议的瓶颈逐渐暴露出来。比如集群规模大时，由于HTTP/1.x协议没有压缩机制，批量拉取较多Pod时容易导致APIServer和etcd出现CPU高负载、OOM、丢包等问题。</p><p>另一方面，etcd v2 client会通过HTTP长连接轮询Watch事件，当watcher较多的时候，因HTTP/1.x不支持多路复用，会创建大量的连接，消耗server端过多的socket和内存资源。</p><p>同时etcd v2支持为每个key设置TTL过期时间，client为了防止key的TTL过期后被删除，需要周期性刷新key的TTL。</p><p>实际业务中很有可能若干key拥有相同的TTL，可是在etcd v2中，即使大量key TTL一样，你也需要分别为每个key发起续期操作，当key较多的时候，这会显著增加集群负载、导致集群性能显著下降。</p><p>最后是<strong>内存开销问题。</strong>etcd v2在内存维护了一颗树来保存所有节点key及value。在数据量场景略大的场景，如配置项较多、存储了大量Kubernetes Events， 它会导致较大的内存开销，同时etcd需要定时把全量内存树持久化到磁盘。这会消耗大量的CPU和磁盘 I/O资源，对系统的稳定性造成一定影响。</p><p>为什么etcd v2有以上若干问题，Consul等其他竞品依然没有被Kubernetes支持呢？</p><p>一方面当时包括Consul在内，没有一个开源项目是十全十美完全满足Kubernetes需求。而CoreOS团队一直在聆听社区的声音并积极改进，解决社区的痛点。用户吐槽etcd不稳定，他们就设计实现自动化的测试方案，模拟、注入各类故障场景，及时发现修复Bug，以提升etcd稳定性。</p><p>另一方面，用户吐槽性能问题，针对etcd v2各种先天性缺陷问题，他们从2015年就开始设计、实现新一代etcd v3方案去解决以上痛点，并积极参与Kubernetes项目，负责etcd v2到v3的存储引擎切换，推动Kubernetes项目的前进。同时，设计开发通用压测工具、输出Consul、ZooKeeper、etcd性能测试报告，证明etcd的优越性。</p><p>etcd v3就是为了解决以上稳定性、扩展性、性能问题而诞生的。</p><p>在内存开销、Watch事件可靠性、功能局限上，它通过引入B-tree、boltdb实现一个MVCC数据库，数据模型从层次型目录结构改成扁平的key-value，提供稳定可靠的事件通知，实现了事务，支持多key原子更新，同时基于boltdb的持久化存储，显著降低了etcd的内存占用、避免了etcd v2定期生成快照时的昂贵的资源开销。</p><p>性能上，首先etcd v3使用了gRPC API，使用protobuf定义消息，消息编解码性能相比JSON超过2倍以上，并通过HTTP/2.0多路复用机制，减少了大量watcher等场景下的连接数。</p><p>其次使用Lease优化TTL机制，每个Lease具有一个TTL，相同的TTL的key关联一个Lease，Lease过期的时候自动删除相关联的所有key，不再需要为每个key单独续期。</p><p>最后是etcd v3支持范围、分页查询，可避免大包等expensive request。</p><p>2016年6月，etcd 3.0诞生，随后Kubernetes 1.6发布，默认启用etcd v3，助力Kubernetes支撑5000节点集群规模。</p><p>下面的时间轴图，我给你总结了etcd3重要特性及版本发布时间。从图中你可以看出，从3.0到未来的3.5，更稳、更快是etcd的追求目标。</p><p><img src="https://static001.geekbang.org/resource/image/5f/6d/5f1bf807db06233ed51d142917798b6d.png?wh=1920*826" alt=""></p><p>从2013年发布第一个版本v0.1到今天的3.5.0-pre，从v2到v3，etcd走过了7年的历程，etcd的稳定性、扩展性、性能不断提升。</p><p>发展到今天，在GitHub上star数超过34K。在Kubernetes的业务场景磨炼下它不断成长，走向稳定和成熟，成为技术圈众所周知的开源产品，而<strong>v3方案的发布，也标志着etcd进入了技术成熟期，成为云原生时代的首选元数据存储产品。</strong></p><h2>小结</h2><p>最后我们来小结下今天的内容，我们从如下几个方面介绍了etcd的前世今生，并在过程中详细解读了为什么Kubernetes使用etcd：</p><ul>
<li>etcd诞生背景， etcd v2源自CoreOS团队遇到的服务协调问题。</li>
<li>etcd目标，我们通过实际业务场景分析，得到理想中的协调服务核心目标：高可用、数据一致性、Watch、良好的可维护性等。而在CoreOS团队看来，高可用、可维护性、适配云、简单的API、良好的性能对他们而言是非常重要的，ZooKeeper无法满足所有诉求，因此决定自己构建一个分布式存储服务。</li>
<li>介绍了v2基于目录的层级数据模型和API，并从分布式系统的角度给你详细总结了etcd v2技术点。etcd的高可用、Watch机制与Kubernetes期望中的元数据存储是匹配的。etcd v2在Kubernetes的带动下，获得了广泛的应用，但也出现若干性能和稳定性、功能不足问题，无法满足Kubernetes项目发展的需求。</li>
<li>CoreOS团队未雨绸缪，从问题萌芽时期就开始构建下一代etcd v3存储模型，分别从性能、稳定性、功能上等成功解决了Kubernetes发展过程中遇到的瓶颈，也捍卫住了作为Kubernetes存储组件的地位。</li>
</ul><p>希望通过今天的介绍， 让你对etcd为什么有v2和v3两个大版本，etcd如何从HTTP/1.x API到gRPC API、单版本数据库到多版本数据库、内存树到boltdb、TTL到Lease、单key原子更新到支持多key事务的演进过程有个清晰了解。希望你能有所收获，在后续的课程中我会和你深入讨论各个模块的细节。</p><h2>思考题</h2><p>最后，我给你留了一个思考题。分享一下在你的项目中，你主要使用的是哪个etcd版本来解决什么问题呢？使用的etcd v2 API还是v3 API呢？在这过程中是否遇到过什么问题？</p><p>感谢你的阅读，欢迎你把思考和观点写在留言区，也欢迎你把这篇文章分享给更多的朋友一起阅读。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/9d/84/171b2221.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jeffery</span>
  </div>
  <div class="_2_QraFYR_0">1.6版本默认自带v3版本，etcd 云原生的数据首选存储产品，老师能说下etcd 和reids 的区别吗？谢谢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Redis是吧，区别挺大的，文章中我用了一副思维导图从数据复制、数据分片、存储引擎、API等方面总结etcd v2技术点，我简单从这些方面给你对比一下，数据复制上Redis是主备异步复制、etcd使用的是Raft，前者可能会丢数据，为了保证读写一致性，etcd读写性能相比Redis差距比较大。数据分片上Redis有各种集群版解决方案，可以承载上T数据，存储的一般是用户数据，而etcd定位是个低容量的关键元数据存储，db大小一般不超过8g。存储引擎和API上Redis内存实现了各种丰富数据结构，而etcd仅是kv API, 使用的是持久化存储boltdb。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-20 23:09:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/e0/4e/c266bdb4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>[小狗]</span>
  </div>
  <div class="_2_QraFYR_0">大佬，可以1周4期吗？ 等着面试用呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢对专栏的支持，正如你所看到的，专栏27讲，每讲内容特别丰富，大部分5000到6000字，整个专栏至少15万字，写作不易，我也是在工作日的凌晨、周末业余时间为大家呈现一个etcd全方位特性解读，更新是固定的1周3篇，春节都在加班加点创作了，麻烦稍等哈，基础篇春节期间会更新完毕，你把基础篇的内容踏踏实实学完，动手做几个小实验，我相信面试哪个大厂，都无压力了哈。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-04 02:18:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/0d/29/18272af9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>hxy</span>
  </div>
  <div class="_2_QraFYR_0">zookeeper的rpc api是thrift，这个感觉说错了吧，zk用的应该是jute</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢你的细心和反馈，我查阅etcd官方资料写得也是的确有误，之前以为是很久之前zk使用的rpc api，看了下commit记录貌似一直都是juite，后面我们修正下，谢谢</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-23 10:11:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/3f/86/4e2d599b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>NICE</span>
  </div>
  <div class="_2_QraFYR_0">目前在使用etcd v3, 高可用，强一致性键值存储。关于etcd集群异地容灾同步目前用的make mirror，有更好的方案吗？ 哈哈😃</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 也有可能在部分场景出现不一致哈，后面实践篇会和大家分析。异地容灾建议使用raft learner特性，添加learner节点，make mirror是基于watch机制的，稳定性、可靠性等相比learner要差点，但是目前社区对learner限制太多，不支持快照等，我们自己做了定制化，大家使用上要等等，估计3.5版本才比较好用</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-21 09:24:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/46/0f/f6cfc659.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>mckee</span>
  </div>
  <div class="_2_QraFYR_0">k8s list pod不带rev情况下，etcdserver key range会造成cpu和mem瞬间飙高，需要读取所有kv并排序</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，list pod太恐怖了，哈哈，线上大部分异常一般都是list pod，configmap，crd</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-26 20:28:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/22/3b/46/3701e908.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Coder</span>
  </div>
  <div class="_2_QraFYR_0">精彩，读起来生动有趣，一文就让我了解了etcd的发展历史与k8s关系，为老师打call！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-21 12:05:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/7f/17/6da0b387.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>SecKiki</span>
  </div>
  <div class="_2_QraFYR_0">学习了，etcd与k8s 绝配</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，etcd与k8s相互影响、促进，etcd社区任何核心特性首先也是考虑是否有益于k8s场景等，新版本发布，也是拿k8s做性能稳定性测试</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-20 22:01:44</div>
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
  <div class="_2_QraFYR_0">头排看戏。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-21 09:42:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/59/c9/f44cb7f3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>爪哇夜未眠</span>
  </div>
  <div class="_2_QraFYR_0">挺好的文章</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢你的留言与支持</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-20 17:53:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/22/b1/54/6d663b95.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>瓜牛</span>
  </div>
  <div class="_2_QraFYR_0">多路复用为啥能减少连接数？多路复用只是提高了socket监听效率，支持更大的连接数，何来的减少连接数，连接数取决于client数啊。不能理解，请老师回复。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-04 10:17:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1c/ac/de/68f35320.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小来子</span>
  </div>
  <div class="_2_QraFYR_0">其次使用 Lease 优化 TTL 机制，每个 Lease 具有一个 TTL，相同的 TTL 的 key 关联一个 Lease，Lease 过期的时候自动删除相关联的所有 key，不再需要为每个 key 单独续期。<br>老师,请问下, Lease过期后, 通过什么手段删除关联的所有Key呢?</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-21 16:55:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/6b/e2/f02e45df.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>恰同学少年。</span>
  </div>
  <div class="_2_QraFYR_0">1. etcd 3.4 + v3API<br>2. 主要遇到以下问题：<br>    - 在使用k8s部署集群时，如何应对多数派、少数派、数据备份与恢复、数据搬迁的运维问题。<br>    - 如何测试watch可靠性<br>    - db size为8GB，项目需要更大运行会存在哪些风险？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-20 12:00:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/ba/ce/fd45714f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>bearlu</span>
  </div>
  <div class="_2_QraFYR_0">学到了学到了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-27 06:44:12</div>
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
  <div class="_2_QraFYR_0">etcd用来做服务发现，治理，分布式锁，还有配置信息存储很好用。<br>缺点是异地多活不好用，延迟有点大。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，我们使用raft learner节点，跨城容灾，进行了一些定制化改造</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-22 09:13:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/46/80/45d989e5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ZHU～JM</span>
  </div>
  <div class="_2_QraFYR_0">老师，我们在使用K8s和etcd的时候经常出现注册和发现超时的情况，这是为什么啊</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这就涉及troubleshooting，通过etcd metric，trace特性，etcd log等各种工具分析才行，后面会有介绍，你可以看看超时时etcd的metric监控和日志</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-21 22:28:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/52/b6/d065c05f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>atom</span>
  </div>
  <div class="_2_QraFYR_0">非常喜欢后面etcd局限性的分析 觉得很不错</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-21 10:51:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/84/c1/dfcad82a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Acter</span>
  </div>
  <div class="_2_QraFYR_0">请问老师，k8s场景etcd v3.3.10版本。由于业务的workflow yaml对象很大，近期已把etcd的max-request-bytes从1.5MiB提高到了5MiB，这对k8s master组件稳定性有何风险吗？该怎么压测呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 后面实践篇会介绍大value风险，容易导致apiserver和etcd oom，超时，雪崩等，建议创建几千个这样的大对象，通过client-go模拟用户list，按标签get，修改等操作</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-21 10:02:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/2d/18/918eaecf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>后端进阶</span>
  </div>
  <div class="_2_QraFYR_0">给唐大佬疯狂打Call，以后学习ETCD就跟着唐大佬的课程走就完了！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 一起持续学习与进步，后续有什么不清楚的，欢迎继续留言交流</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-20 21:13:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/be/35/dd79037e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>a...Z</span>
  </div>
  <div class="_2_QraFYR_0">等真美久  终于有了etcd的讲解,期待</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 一起加油，有任何疑问，欢迎随时留言交流!</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-20 20:27:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/c4/03/f753fda7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>JianXu</span>
  </div>
  <div class="_2_QraFYR_0">我们最近线上一个问题，负责处理的同学给的回复如下：- API server is still up but is operating in a degraded state. How do we detect such issues and re-establish an L4 connection？<br><br>四层重连是可以考虑的，但是这个要改client端，比如可以这样，7层request如果不成功（出现504或者client context timeout），考虑断开4层链接重连。另外要配合flowschema。来减少apiserver由于实际负载过重导致latency增加。目前我们的APF是1.18的 没有引入seats，所以对于一个request，无论是 会返回200k pods的list all pods，还是一个 list只有一个obj的request。都是占用一个concurrency。体现不了实际cost。就不容易配置。设大了起不到限流作用，设小了对于那些小请求吞吐被人为的限住了。https:&#47;&#47;kubernetes.io&#47;docs&#47;concepts&#47;cluster-administration&#47;flow-control&#47;#seats-occupied-by-a-request<br>1.22里面的flowschema加了seats的概念。在做限流的时候一定程度上区分request的cost。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-02 15:40:41</div>
  </div>
</div>
</div>
</li>
</ul>