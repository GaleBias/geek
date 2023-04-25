<audio title="38 _ 通信开销：限制Redis Cluster规模的关键因素" src="https://static001.geekbang.org/resource/audio/65/dd/659052fe681a6fed9bd80702d75fcddd.mp3" controls="controls"></audio> 
<p>你好，我是蒋德钧。</p><p>Redis Cluster能保存的数据量以及支撑的吞吐量，跟集群的实例规模密切相关。Redis官方给出了Redis Cluster的规模上限，就是一个集群运行1000个实例。</p><p>那么，你可能会问，为什么要限定集群规模呢？其实，这里的一个关键因素就是，<strong>实例间的通信开销会随着实例规模增加而增大</strong>，在集群超过一定规模时（比如800节点），集群吞吐量反而会下降。所以，集群的实际规模会受到限制。</p><p>今天这节课，我们就来聊聊，集群实例间的通信开销是如何影响Redis Cluster规模的，以及如何降低实例间的通信开销。掌握了今天的内容，你就可以通过合理的配置来扩大Redis Cluster的规模，同时保持高吞吐量。</p><h2>实例通信方法和对集群规模的影响</h2><p>Redis Cluster在运行时，每个实例上都会保存Slot和实例的对应关系（也就是Slot映射表），以及自身的状态信息。</p><p>为了让集群中的每个实例都知道其它所有实例的状态信息，实例之间会按照一定的规则进行通信。这个规则就是Gossip协议。</p><p>Gossip协议的工作原理可以概括成两点。</p><p>一是，每个实例之间会按照一定的频率，从集群中随机挑选一些实例，把PING消息发送给挑选出来的实例，用来检测这些实例是否在线，并交换彼此的状态信息。PING消息中封装了发送消息的实例自身的状态信息、部分其它实例的状态信息，以及Slot映射表。</p><!-- [[[read_end]]] --><p>二是，一个实例在接收到PING消息后，会给发送PING消息的实例，发送一个PONG消息。PONG消息包含的内容和PING消息一样。</p><p>下图显示了两个实例间进行PING、PONG消息传递的情况。</p><p><img src="https://static001.geekbang.org/resource/image/5e/86/5eacfc36c4233ae7c99f80b1511yyb86.jpg?wh=2435*890" alt=""></p><p>Gossip协议可以保证在一段时间后，集群中的每一个实例都能获得其它所有实例的状态信息。</p><p>这样一来，即使有新节点加入、节点故障、Slot变更等事件发生，实例间也可以通过PING、PONG消息的传递，完成集群状态在每个实例上的同步。</p><p>经过刚刚的分析，我们可以很直观地看到，实例间使用Gossip协议进行通信时，通信开销受到<strong>通信消息大小</strong>和<strong>通信频率</strong>这两方面的影响，</p><p>消息越大、频率越高，相应的通信开销也就越大。如果想要实现高效的通信，可以从这两方面入手去调优。接下来，我们就来具体分析下这两方面的实际情况。</p><p>首先，我们来看实例通信的消息大小。</p><h3>Gossip消息大小</h3><p>Redis实例发送的PING消息的消息体是由clusterMsgDataGossip结构体组成的，这个结构体的定义如下所示：</p><pre><code>typedef struct {
    char nodename[CLUSTER_NAMELEN];  //40字节
    uint32_t ping_sent; //4字节
    uint32_t pong_received; //4字节
    char ip[NET_IP_STR_LEN]; //46字节
    uint16_t port;  //2字节
    uint16_t cport;  //2字节
    uint16_t flags;  //2字节
    uint32_t notused1; //4字节
} clusterMsgDataGossip;
</code></pre><p>其中，CLUSTER_NAMELEN和NET_IP_STR_LEN的值分别是40和46，分别表示，nodename和ip这两个字节数组的长度是40字节和46字节，我们再把结构体中其它信息的大小加起来，就可以得到一个Gossip消息的大小了，即104字节。</p><p>每个实例在发送一个Gossip消息时，除了会传递自身的状态信息，默认还会传递集群十分之一实例的状态信息。</p><p>所以，对于一个包含了1000个实例的集群来说，每个实例发送一个PING消息时，会包含100个实例的状态信息，总的数据量是 10400字节，再加上发送实例自身的信息，一个Gossip消息大约是10KB。</p><p>此外，为了让Slot映射表能够在不同实例间传播，PING消息中还带有一个长度为 16,384 bit 的 Bitmap，这个Bitmap的每一位对应了一个Slot，如果某一位为1，就表示这个Slot属于当前实例。这个Bitmap大小换算成字节后，是2KB。我们把实例状态信息和Slot分配信息相加，就可以得到一个PING消息的大小了，大约是12KB。</p><p>PONG消息和PING消息的内容一样，所以，它的大小大约是12KB。每个实例发送了PING消息后，还会收到返回的PONG消息，两个消息加起来有24KB。</p><p>虽然从绝对值上来看，24KB并不算很大，但是，如果实例正常处理的单个请求只有几KB的话，那么，实例为了维护集群状态一致传输的PING/PONG消息，就要比单个业务请求大了。而且，每个实例都会给其它实例发送PING/PONG消息。随着集群规模增加，这些心跳消息的数量也会越多，会占据一部分集群的网络通信带宽，进而会降低集群服务正常客户端请求的吞吐量。</p><p>除了心跳消息大小会影响到通信开销，如果实例间通信非常频繁，也会导致集群网络带宽被频繁占用。那么，Redis Cluster中实例的通信频率是什么样的呢？</p><h3>实例间通信频率</h3><p>Redis Cluster的实例启动后，默认会每秒从本地的实例列表中随机选出5个实例，再从这5个实例中找出一个最久没有通信的实例，把PING消息发送给该实例。这是实例周期性发送PING消息的基本做法。</p><p>但是，这里有一个问题：实例选出来的这个最久没有通信的实例，毕竟是从随机选出的5个实例中挑选的，这并不能保证这个实例就一定是整个集群中最久没有通信的实例。</p><p>所以，这有可能会出现，<strong>有些实例一直没有被发送PING消息，导致它们维护的集群状态已经过期了</strong>。</p><p>为了避免这种情况，Redis Cluster的实例会按照每100ms一次的频率，扫描本地的实例列表，如果发现有实例最近一次接收 PONG消息的时间，已经大于配置项 cluster-node-timeout的一半了（cluster-node-timeout/2），就会立刻给该实例发送 PING消息，更新这个实例上的集群状态信息。</p><p>当集群规模扩大之后，因为网络拥塞或是不同服务器间的流量竞争，会导致实例间的网络通信延迟增加。如果有部分实例无法收到其它实例发送的PONG消息，就会引起实例之间频繁地发送PING消息，这又会对集群网络通信带来额外的开销了。</p><p>我们来总结下单实例每秒会发送的PING消息数量，如下所示：</p><blockquote>
<p>PING消息发送数量 = 1 + 10 * 实例数（最近一次接收PONG消息的时间超出cluster-node-timeout/2）</p>
</blockquote><p>其中，1是指单实例常规按照每1秒发送一个PING消息，10是指每1秒内实例会执行10次检查，每次检查后会给PONG消息超时的实例发送消息。</p><p>我来借助一个例子，带你分析一下在这种通信频率下，PING消息占用集群带宽的情况。</p><p>假设单个实例检测发现，每100毫秒有10个实例的PONG消息接收超时，那么，这个实例每秒就会发送101个PING消息，约占1.2MB/s带宽。如果集群中有30个实例按照这种频率发送消息，就会占用36MB/s带宽，这就会挤占集群中用于服务正常请求的带宽。</p><p>所以，我们要想办法降低实例间的通信开销，那该怎么做呢？</p><h2>如何降低实例间的通信开销？</h2><p>为了降低实例间的通信开销，从原理上说，我们可以减小实例传输的消息大小（PING/PONG消息、Slot分配信息），但是，因为集群实例依赖PING、PONG消息和Slot分配信息，来维持集群状态的统一，一旦减小了传递的消息大小，就会导致实例间的通信信息减少，不利于集群维护，所以，我们不能采用这种方式。</p><p>那么，我们能不能降低实例间发送消息的频率呢？我们先来分析一下。</p><p>经过刚才的学习，我们现在知道，实例间发送消息的频率有两个。</p><ul>
<li>每个实例每1秒发送一条PING消息。这个频率不算高，如果再降低该频率的话，集群中各实例的状态可能就没办法及时传播了。</li>
<li>每个实例每100毫秒会做一次检测，给PONG消息接收超过cluster-node-timeout/2的节点发送PING消息。实例按照每100毫秒进行检测的频率，是Redis实例默认的周期性检查任务的统一频率，我们一般不需要修改它。</li>
</ul><p>那么，就只有cluster-node-timeout这个配置项可以修改了。</p><p>配置项cluster-node-timeout定义了集群实例被判断为故障的心跳超时时间，默认是15秒。如果cluster-node-timeout值比较小，那么，在大规模集群中，就会比较频繁地出现PONG消息接收超时的情况，从而导致实例每秒要执行10次“给PONG消息超时的实例发送PING消息”这个操作。</p><p>所以，为了避免过多的心跳消息挤占集群带宽，我们可以调大cluster-node-timeout值，比如说调大到20秒或25秒。这样一来， PONG消息接收超时的情况就会有所缓解，单实例也不用频繁地每秒执行10次心跳发送操作了。</p><p>当然，我们也不要把cluster-node-timeout调得太大，否则，如果实例真的发生了故障，我们就需要等待cluster-node-timeout时长后，才能检测出这个故障，这又会导致实际的故障恢复时间被延长，会影响到集群服务的正常使用。</p><p>为了验证调整cluster-node-timeout值后，是否能减少心跳消息占用的集群网络带宽，我给你提个小建议：<strong>你可以在调整cluster-node-timeout值的前后，使用tcpdump命令抓取实例发送心跳信息网络包的情况</strong>。</p><p>例如，执行下面的命令后，我们可以抓取到192.168.10.3机器上的实例从16379端口发送的心跳网络包，并把网络包的内容保存到r1.cap文件中：</p><pre><code>tcpdump host 192.168.10.3 port 16379 -i 网卡名 -w /tmp/r1.cap
</code></pre><p>通过分析网络包的数量和大小，就可以判断调整cluster-node-timeout值前后，心跳消息占用的带宽情况了。</p><h2>小结</h2><p>这节课，我向你介绍了Redis Cluster实例间以Gossip协议进行通信的机制。Redis Cluster运行时，各实例间需要通过PING、PONG消息进行信息交换，这些心跳消息包含了当前实例和部分其它实例的状态信息，以及Slot分配信息。这种通信机制有助于Redis Cluster中的所有实例都拥有完整的集群状态信息。</p><p>但是，随着集群规模的增加，实例间的通信量也会增加。如果我们盲目地对Redis Cluster进行扩容，就可能会遇到集群性能变慢的情况。这是因为，集群中大规模的实例间心跳消息会挤占集群处理正常请求的带宽。而且，有些实例可能因为网络拥塞导致无法及时收到PONG消息，每个实例在运行时会周期性地（每秒10次）检测是否有这种情况发生，一旦发生，就会立即给这些PONG消息超时的实例发送心跳消息。集群规模越大，网络拥塞的概率就越高，相应的，PONG消息超时的发生概率就越高，这就会导致集群中有大量的心跳消息，影响集群服务正常请求。</p><p>最后，我也给你一个小建议，虽然我们可以通过调整cluster-node-timeout配置项减少心跳消息的占用带宽情况，但是，在实际应用中，如果不是特别需要大容量集群，我建议你把Redis Cluster 的规模控制在400~500个实例。</p><p>假设单个实例每秒能支撑8万请求操作（8万QPS），每个主实例配置1个从实例，那么，400~ 500个实例可支持 1600万~2000万QPS（200/250个主实例*8万QPS=1600/2000万QPS），这个吞吐量性能可以满足不少业务应用的需求。</p><h2>每课一问</h2><p>按照惯例，我给你提个小问题，如果我们采用跟Codis保存Slot分配信息相类似的方法，把集群实例状态信息和Slot分配信息保存在第三方的存储系统上（例如Zookeeper），这种方法会对集群规模产生什么影响吗？</p><p>欢迎在留言区写下你的思考和答案，我们一起交流讨论。如果你觉得今天的内容对你有所帮助，也欢迎你分享给你的朋友或同事。我们下节课见。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/90/8a/288f9f94.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Kaito</span>
  </div>
  <div class="_2_QraFYR_0">如果采用类似 Codis 保存 Slot 信息的方法，把集群实例状态信息和 Slot 分配信息保存在第三方的存储系统上（例如Zookeeper），这种方法会对集群规模产生什么影响？<br><br>由于 Redis Cluster 每个实例需要保存集群完整的路由信息，所以每增加一个实例，都需要多一次与其他实例的通信开销，如果有 N 个实例，集群就要存储 N 份完整的路由信息。而如果像 Codis 那样，把 Slot 信息存储在第三方存储上，那么无论集群实例有多少，这些信息在第三方存储上只会存储一份，也就是说，集群内的通信开销，不会随着实例的增加而增长。当集群需要用到这些信息时，直接从第三方存储上获取即可。<br><br>Redis Cluster 把所有功能都集成在了 Redis 实例上，包括路由表的交换、实例健康检查、故障自动切换等等，这么做的好处是，部署和使用非常简单，只需要部署实例，然后让多个实例组成切片集群即可提供服务。但缺点也很明显，每个实例负责的工作比较重，如果看源码实现，也不太容易理解，而且如果其中一个功能出现 bug，只能升级整个 Redis Server 来解决。<br><br>而 Codis 把这些功能拆分成多个组件，每个组件负责的工作都非常纯粹，codis-proxy 负责转发请求，codis-dashboard 负责路由表的分发、数据迁移控制，codis-server 负责数据存储和数据迁移，哨兵负责故障自动切换，codis-fe 负责提供友好的运维界面，每个组件都可以单独升级，这些组件相互配合，完成整个集群的对外服务。但其缺点是组件比较多，部署和维护比较复杂。<br><br>在实际的业务场景下，我觉得应该尽量避免非常大的分片集群，太大的分片集群一方面存在通信开销大的问题，另一方面也会导致集群变得越来越难以维护。而且当集群出问题时，对业务的影响也比较集中。建议针对不同的业务线、业务模块，单独部署不同的分片集群，这样方便运维和管理的同时，出现问题也只会影响某一个业务模块。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-20 00:06:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/0a/a4/828a431f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张申傲</span>
  </div>
  <div class="_2_QraFYR_0">看到 Gossip 协议，第一时间想到了《人类简史》中说的：八卦是人类进步的动力，但是集群超过一定规模时，八卦的作用就十分有限了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 看到Gossip的八卦本质了 :)</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-22 09:57:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTI2icbib62icXtibTkThtyRksbuJLoTLMts7zook2S30MiaBtbz0f5JskwYicwqXkhpYfvCpuYkcvPTibEaQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>xuanyuan</span>
  </div>
  <div class="_2_QraFYR_0">可以划分管理面和数据面，集群通信走单独的网络</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这是一个方法。分布式系统的通信中，控制平面和数据平面通常会分开来。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-09 23:34:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/7f/91/962eba1a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>唐朝首都</span>
  </div>
  <div class="_2_QraFYR_0">集群的规模应该是可以进一步扩大的。因为集群的信息保存在了第三方存储系统上，意味着redis cluster内部不用再沟通了，这将节省下大量的集群内部的沟通成本。当然就整个集群而言部署、维护也会更加复杂，毕竟引入了一个第三方组件来管理集群。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-25 09:07:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/19/bf/415023b5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>璩雷</span>
  </div>
  <div class="_2_QraFYR_0">约到后面，评论的人越少，看来坚持到最后的人不多啊~~</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-30 10:26:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/28/6c/bc/f751786b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Ming</span>
  </div>
  <div class="_2_QraFYR_0">其实集群不要大，大了通讯是个问题同样后期维护也是个很大的麻烦；不同业务的redis集群区分开来，这样每个集群不至于太大，也不至于一个集群出问题影响到别的业务；</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-28 20:29:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/ec/13/49e98289.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>neohope</span>
  </div>
  <div class="_2_QraFYR_0">用Gossip协议管理配置和Zookeeper统一存储配置信息各有利弊。<br>Gossip协议在节点间传递配置让系统简单，而且发生网络故障时自行恢复能力更强一些，但通讯效率随着网络节点的增加而降低；<br>Zookeeper统一管理配置，通讯效率无论节点多少都比较高，但让系统架构更复杂故障点增多，对抗网络故障时自行恢复能力差一些。<br>但其实无论哪种方式，节点太多了都会更加难以管理维护，出现问题影响面也更难以控制，不推荐。<br>但其实另一个极端，就是单个实例性能特别高，存储特别多数据也不推荐，同样也是更容易出问题，出现问题影响面太大，不推荐。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-24 20:33:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/23/5b/983408b9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>悟空聊架构</span>
  </div>
  <div class="_2_QraFYR_0">Gossip 协议还有很多有意思的东西，可以参照这篇：<br>病毒传播：全靠 Gossip 协议：<br>http:&#47;&#47;www.passjava.cn&#47;#&#47;92.%E5%88%86%E5%B8%83%E5%BC%8F&#47;08.Gossip%E5%8D%8F%E8%AE%AE</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-13 09:53:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/8b/4b/15ab499a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>风轻扬</span>
  </div>
  <div class="_2_QraFYR_0">tcpdump命令，执行之后会报错：syntax error，在host和port之间加一个and就可以了。如下：<br><br>tcpdump host 192.168.10.3 and port 16379 -i 网卡名 -w &#47;tmp&#47;r1.cap</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-06 12:40:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/2a/ff/a9d72102.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>BertGeek</span>
  </div>
  <div class="_2_QraFYR_0">请问老师一个问题：<br>环境：阿里云ack k8s集群，阿里云redis，rds等<br>问题：应用访问突然报错：nested exception is io.lettuce.core.RedisConnectionException<br><br>1. 检查redis 连接ok，健康状态ok<br>2. 应用监控也正常<br>3. 最后java 一个服务pod 删除，自动重建，问题消失<br><br>针对这个不定时的问题，对于生产环境还是需要排查分析，希望老师给点建议，非常感谢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-23 18:13:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/28/a1/d8/42252c48.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>123</span>
  </div>
  <div class="_2_QraFYR_0">差不多看到最后了，还是坚持快看完了，学习还是需要持之以恒</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-30 14:28:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/28/ac/37a2a265.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>弱水穿云天</span>
  </div>
  <div class="_2_QraFYR_0">冒个泡，还在坚持。善始者众善终者寡</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-26 09:03:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/22/bd/9b/366bb87b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>飞龙</span>
  </div>
  <div class="_2_QraFYR_0">没搞明白，如果一定数量的redis实例都部署在同一内网网络环境之内，实例之前通过内网相互PING Pong,怎么会占网络通信带宽呢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-17 14:22:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJN6ZnE6ECdJ2aW1WicDVyGwWjQgBWad8WNqHicajKaE4hkmVBJU8vuVEab2MicC4bdknMndjRspo4Hw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>没想法的岁月</span>
  </div>
  <div class="_2_QraFYR_0">每个实例在发送一个 Gossip 消息时，除了会传递自身的状态信息，默认还会传递集群十分之一实例的状态信息。---------ping的时候为什么要发送其他实例的信息</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-18 19:42:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/47/00/3202bdf0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>piboye</span>
  </div>
  <div class="_2_QraFYR_0">master 故障的时候， 需要整个集群 master 参与选择 一个 slave 重新成为 master,  如果是 100 节点， 50 个master， 需要 49 个 master 参与这个类似raft 的算法， 效率也太低下了吧？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-18 12:00:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/57/fe/beab006d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Jasper</span>
  </div>
  <div class="_2_QraFYR_0">打卡</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-26 17:31:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/d9/ff/b23018a6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Heaven</span>
  </div>
  <div class="_2_QraFYR_0">好处在于,这样减少了每个实例的负担,保证了元信息的一致性<br>坏处是集群系统中需要额外维护第三方系统,增加了系统复杂度</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-06 11:17:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/01/c5/b48d25da.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>cake</span>
  </div>
  <div class="_2_QraFYR_0">我们也不要把 cluster-node-timeout 调得太大，否则，如果实例真的发生了故障，我们就需要等待 cluster-node-timeout 时长后，才能检测出这个故障，这又会导致实际的故障恢复时间被延长，会影响到集群服务的正常使用。 老师请问下这句话 不是因该是 发生故障 cluster-node-timeout &#47;2 就能检测出故障吗? 因为 pong 超过 cluster-node-timeout &#47;2 就会发送ping，为什么是cluster-node-timeout 之后才能检测出故障呢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-21 19:32:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJ8ic8eLTo5rnqIMJicUfpkVBrOUJAW4fANicKIbHdC54O9SOdwSoeK6o8icibaUbh7ZUXAkGF9zwHqo0Q/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ivhong</span>
  </div>
  <div class="_2_QraFYR_0">我一直想问个问题，是每个redis实例配置到一台计算机上？还是每个计算机的物理核上绑定一个redis实例？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-29 16:25:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/92/6d/becd841a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>escray</span>
  </div>
  <div class="_2_QraFYR_0">今天这个话题应该有点类似屠龙之技，菜鸟如我，什么时候才能有机会运维 100 个以上实例的 Redis Cluster。<br><br>Gossip 协议还挺有意思，名字也比较形象。如果翻译成中文，是叫“八卦协议”么？好像容易引起误解。“流言蜚语协议”、“风闻协议”？<br><br>一个 ping 消息大概是 104 字节，1000 个实例的 Redis 集群一个 Gossip 消息大概是 12KB，ping-pong 往返，24KB。再加上实例间的通信，那么集群中用于服务正常请求的带宽就会被占用。在这种情况下，是不是采用类似于 Codis 的集中式管理更合适？<br><br>将 cluster-node-timeout 从 15 秒调整到 20 或 25 秒，大概能减少 1&#47;3 到 2&#47;3 的实例间通信流量（不知道这个计算是否正确）<br><br>PING 消息发送数量 = 1 + 10 * 实例数（最近一次接收 PONG 消息的时间超出 cluster-node-timeout&#47;2）<br><br>估计最后还是要靠 tcpdump 来分析实例间的网络带宽变化情况，然后再找出合适的 cluster-node-timeout。但是业务流量经常会有变化，增加了调优的难度。<br><br>对于课后题，如果是 Codis 模式，将集群实例状态信息和 Slot 分配信息保存在 Zookeeper 上，那么实例太多之后，查询分配信息的时间也会比较长，另外实时保存实例状态信息也比较难。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-04 08:20:04</div>
  </div>
</div>
</div>
</li>
</ul>