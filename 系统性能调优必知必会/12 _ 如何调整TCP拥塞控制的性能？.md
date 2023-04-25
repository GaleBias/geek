<audio title="12 _ 如何调整TCP拥塞控制的性能？" src="https://static001.geekbang.org/resource/audio/b7/4c/b79a03a9a6ba059cfe329e3f4ccccb4c.mp3" controls="controls"></audio> 
<p>你好，我是陶辉。</p><p>上一讲我们谈到接收主机的处理能力不足时，是通过滑动窗口来减缓对方的发送速度。这一讲我们来看看，当网络处理能力不足时又该如何优化TCP的性能。</p><p>如果你阅读过TCP协议相关的书籍，一定看到过慢启动、拥塞控制等名词。这些概念似乎离应用开发者很远，然而，如果没有拥塞控制，整个网络将会锁死，所有消息都无法传输。</p><p>而且，如果你在开发分布式集群中的高并发服务，理解拥塞控制的工作原理，就可以在内核的TCP层，提升所有进程的网络性能。比如，你可能听过，2013年谷歌把初始拥塞窗口从3个MSS（最大报文长度）左右提升到10个MSS，将Web站点的网络性能提升了10%以上，而有些高速CDN站点，甚至把初始拥塞窗口提升到70个MSS。</p><p>特别是，近年来谷歌提出的BBR拥塞控制算法已经应用在高版本的Linux内核中，从它在YouTube上的应用可以看到，在高性能站点上网络时延有20%以上的降低，传输带宽也有提高。</p><p>Linux允许我们调整拥塞控制算法，但是，正确地设置参数，还需要深入理解拥塞控制对TCP连接的影响。这一讲我们将沿着网络如何影响发送速度这条线，看看如何调整Linux下的拥塞控制参数。</p><h2>慢启动阶段如何调整初始拥塞窗口？</h2><!-- [[[read_end]]] --><p>上一讲谈到，只要接收方的读缓冲区足够大，就可以通过报文中的接收窗口，要求对方更快地发送数据。然而，网络的传输速度是有限的，它会直接丢弃超过其处理能力的报文。而发送方只有在重传定时器超时后，才能发现超发的报文被网络丢弃了，发送速度提不上去。更为糟糕的是，如果网络中的每个连接都按照接收窗口尽可能地发送更多的报文时，就会形成恶性循环，最终超高的网络丢包率会使得每个连接都无法发送数据。</p><p>解决这一问题的方案叫做拥塞控制，它包括4个阶段，我们首先来看TCP连接刚建立时的慢启动阶段。由于TCP连接会穿越许多网络，所以最初并不知道网络的传输能力，为了避免发送超过网络负载的报文，TCP只能先调低发送窗口（关于发送窗口，你可以参考<a href="https://time.geekbang.org/column/article/239176">[第11讲]</a>），减少飞行中的报文来让发送速度变慢，这也是“慢启动”名字的由来。</p><p>让发送速度变慢是通过引入拥塞窗口（全称为congestion window，缩写为CWnd，类似地，接收窗口叫做rwnd，发送窗口叫做swnd）实现的，它用于避免网络出现拥塞。上一讲我们说过，如果不考虑网络拥塞，发送窗口就等于对方的接收窗口，而考虑了网络拥塞后，发送窗口则应当是拥塞窗口与对方接收窗口的最小值：</p><pre><code>swnd = min(cwnd, rwnd)
</code></pre><p>这样，发送速度就综合考虑了接收方和网络的处理能力。</p><p>虽然窗口的计量单位是字节，但为了方便理解，通常我们用MSS作为描述窗口大小的单位，其中MSS是TCP报文的最大长度。</p><p>如果初始拥塞窗口只有1个MSS，当MSS是1KB，而RTT时延是100ms时，发送速度只有10KB/s。所以，当没有发生拥塞时，拥塞窗口必须快速扩大，才能提高互联网的传输速度。因此，慢启动阶段会以指数级扩大拥塞窗口（扩大规则是这样的：发送方每收到一个ACK确认报文，拥塞窗口就增加1个MSS），比如最初的初始拥塞窗口（也称为initcwnd）是1个MSS，经过4个RTT就会变成16个MSS。</p><p>虽然指数级提升发送速度很快，但互联网中的很多资源体积并不大，多数场景下，在传输速度没有达到最大时，资源就已经下载完了。下图是2010年Google对Web对象大小的CDF累积分布统计，大多数对象在10KB左右。</p><p><img src="https://static001.geekbang.org/resource/image/1a/23/1a93f6622d30ef5a138b37ce7f94e323.png?wh=1368*644" alt="" title="图片来源：《An Argument for Increasing TCP's Initial Contestion Window》"></p><p>这样，当MSS是1KB时，多数HTTP请求至少包含10个报文，即使以指数级增加拥塞窗口，也需要至少4个RTT才能传输完，参见下图：</p><p><img src="https://static001.geekbang.org/resource/image/86/5c/865b14ad43c828fdf494b541bb810f5c.png?wh=865*779" alt=""></p><p>因此，2013年TCP的初始拥塞窗口调整到了10个MSS（参见<a href="https://tools.ietf.org/html/rfc6928">RFC6928</a>），这样1个RTT内就可以传输10KB的请求。然而，如果你需要传输的对象体积更大，BDP带宽时延积很大时，完全可以继续提高初始拥塞窗口的大小。下图是2014年、2017年全球主要CDN厂商初始拥塞窗口的变化，可见，随着网速的增加，初始拥塞窗口也变得更大了。</p><p><a href="https://blog.imaginea.com/look-at-tcp-initcwnd-cdns/"><img src="https://static001.geekbang.org/resource/image/20/d2/20bdd4477ed2d837d398a3b43020abd2.png?wh=1027*733" alt="" title="图片来源：https://blog.imaginea.com/look-at-tcp-initcwnd-cdns/"></a></p><p>因此，你可以根据网络状况和传输对象的大小，调整初始拥塞窗口的大小。调整前，先要清楚你的服务器现在的初始拥塞窗口是多大。你可以通过ss命令查看当前拥塞窗口：</p><pre><code># ss -nli|fgrep cwnd
         cubic rto:1000 mss:536 cwnd:10 segs_in:10621866 lastsnd:1716864402 lastrcv:1716864402 lastack:1716864402
</code></pre><p>再通过ip route change命令修改初始拥塞窗口：</p><pre><code># ip route | while read r; do
           ip route change $r initcwnd 10;
       done
</code></pre><p>当然，更大的初始拥塞窗口以及指数级的提速，连接很快就会遭遇网络拥塞，从而导致慢启动阶段的结束。</p><h2>出现网络拥塞时该怎么办？</h2><p>以下3种场景都会导致慢启动阶段结束：</p><ol>
<li>通过定时器明确探测到了丢包；</li>
<li>拥塞窗口的增长到达了慢启动阈值ssthresh（全称为slow start threshold），也就是之前发现网络拥塞时的窗口大小；</li>
<li>接收到重复的ACK报文，可能存在丢包。</li>
</ol><p>我们先来看第1种场景，在规定时间内没有收到ACK报文，这说明报文丢失了，网络出现了严重的拥塞，必须先降低发送速度，再进入拥塞避免阶段。不同的拥塞控制算法降低速度的幅度并不相同，比如CUBIC算法会把拥塞窗口降为原先的0.8倍（也就是发送速度降到0.8倍）。此时，我们知道了多大的窗口会导致拥塞，因此可以把慢启动阈值设为发生拥塞前的窗口大小。</p><p>再看第2种场景，虽然还没有发生丢包，但发送方已经达到了曾经发生网络拥塞的速度（拥塞窗口达到了慢启动阈值），接下来发生拥塞的概率很高，所以进入<strong>拥塞避免阶段，此时拥塞窗口不能再以指数方式增长，而是要以线性方式增长</strong>。接下来，拥塞窗口会以每个RTT增加1个MSS的方式，代替慢启动阶段每收到1个ACK就增加1个MSS的方式。这里可能有同学会有疑问，在第1种场景发生前，慢启动阈值是多大呢？事实上，<a href="https://tools.ietf.org/html/rfc5681#page-5">RFC5681</a> 建议最初的慢启动阈值尽可能的大，这样才能在第1、3种场景里快速发现网络瓶颈。</p><p>第3种场景最为复杂。我们知道，TCP传输的是字节流，而“流”是天然有序的。因此，当接收方收到不连续的报文时，就可能发生报文丢失或者延迟，等待发送方超时重发太花时间了，为了缩短重发时间，<strong>快速重传算法便应运而生。</strong></p><p>当连续收到3个重复ACK时，发送方便得到了网络发生拥塞的明确信号，通过重复ACK报文的序号，我们知道丢失了哪个报文，这样，不等待定时器的触发，立刻重发丢失的报文，可以让发送速度下降得慢一些，这就是快速重传算法。</p><p>出现拥塞后，发送方会缩小拥塞窗口，再进入前面提到的拥塞避免阶段，用线性速度慢慢增加拥塞窗口。然而，<strong>为了平滑地降低速度，发送方应当先进入快速恢复阶段，在失序报文到达接收方后，再进入拥塞避免阶段。</strong></p><p>那什么是快速恢复呢？我们不妨把网络看成一个容器（上一讲中说过它可以容纳BDP字节的报文），每当接收方从网络中取出一个报文，发送方就可以增加一个报文。当发送方接收到重复ACK时，可以推断有失序报文离开了网络，到达了接收方的缓冲区，因此可以再多发送一个报文。如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/92/d9/92980476c93766887cc260f03c5d50d9.png?wh=992*795" alt=""></p><p>这里你要注意：第6个报文在慢启动阶段丢失，接收方收到失序的第7个报文会触发快速重传算法，它必须立刻返回ACK6。而发送方接收到第1个重复ACK6报文时，就从慢启动进入了快速重传阶段，<strong>此刻的重复ACK不会扩大拥塞窗口。</strong>当连续收到3个ACK6时，发送方会重发报文6，并把慢启动阈值和拥塞窗口都降到之前的一半：3个MSS，再进入快速恢复阶段。按照规则，由于收到3个重复ACK，所以拥塞窗口会增加3个MSS。之后收到的2个ACK，让拥塞窗口增加到了8个MSS，直到收到期待的ACK12，发送方才会进入拥塞避免阶段。</p><p>慢启动、拥塞避免、快速重传、快速恢复，共同构成了拥塞控制算法。Linux上提供了更改拥塞控制算法的配置，你可以通过tcp_available_congestion_control配置查看内核支持的算法列表：</p><pre><code>net.ipv4.tcp_available_congestion_control = cubic reno
</code></pre><p>再通过tcp_congestion_control配置选择一个具体的拥塞控制算法：</p><pre><code>net.ipv4.tcp_congestion_control = cubic
</code></pre><p>但有件事你得清楚，拥塞控制是控制网络流量的算法，主机间会互相影响，在生产环境更改之前必须经过完善的测试。</p><h2>基于测量的拥塞控制算法</h2><p>上文介绍的是传统拥塞控制算法，它是以丢包作为判断拥塞的依据。然而，网络刚出现拥塞时并不会丢包，而真的出现丢包时，拥塞已经非常严重了。如下图所示，像路由器这样的网络设备，都会有缓冲队列应对突发的、超越处理能力的流量：</p><p><img src="https://static001.geekbang.org/resource/image/47/85/4732f8f97aefcb26334f4e7d1d096185.png?wh=957*532" alt=""></p><p>当缓冲队列为空时，传输速度最快。一旦队列开始积压，每个报文的传输时间需要增加排队时间，网速就变慢了。而当队列溢出时，才会出现丢包，基于丢包的拥塞控制算法在这个时间点进入拥塞避免阶段，显然太晚了。因为升高的网络时延降低了用户体验，而且从丢包到重发这段时间，带宽也会出现下降。</p><p>进行拥塞控制的最佳时间点，是缓冲队列刚出现积压的时刻，<strong>此时，网络时延会增高，但带宽维持不变，这两个数值的变化可以给出明确的拥塞信号</strong>，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/2c/ca/2cbda9079294ed5da6617f0f0e83acca.png?wh=521*720" alt="" title="图片来源网络：传输速度_RTT与飞行报文的关系"></p><p>这种以测量带宽、时延来确定拥塞的方法，在丢包率较高的网络中应用效果尤其好。2016年Google推出的BBR算法（全称Bottleneck Bandwidth and Round-trip propagation time），就是测量驱动的拥塞控制算法，它在YouTube站点上应用后使得网络时延下降了20%以上，传输带宽也有5%左右的提升。</p><p>当然，测量驱动的拥塞算法并没有那么简单，因为网络会波动，线路也会变化，算法必须及时地响应网络变化，这里不再展开算法细节，你可以在我的<a href="https://www.taohui.pub/2019/08/07/%e4%b8%80%e6%96%87%e8%a7%a3%e9%87%8a%e6%b8%85%e6%a5%9agoogle-bbr%e6%8b%a5%e5%a1%9e%e6%8e%a7%e5%88%b6%e7%ae%97%e6%b3%95%e5%8e%9f%e7%90%86/">这篇博客</a>中找到BBR算法更详细的介绍。</p><p>Linux 4.9版本之后都支持BBR算法，开启BBR算法仍然使用tcp_congestion_control配置：</p><pre><code>net.ipv4.tcp_congestion_control=bbr
</code></pre><h2>小结</h2><p>我们对这一讲的内容做个小结。</p><p>当TCP连接建立成功后，拥塞控制算法就会发生作用，首先进入慢启动阶段。决定连接此时网速的是初始拥塞窗口，Linux上可以通过route ip change命令修改它。通常，在带宽时延积较大的网络中，应当调高初始拥塞窗口。</p><p>丢包以及重复的ACK都是明确的拥塞信号，此时，发送方就会调低拥塞窗口减速，同时修正慢启动阈值。这样，将来再次到达这个速度时，就会自动进入拥塞避免阶段，用线性速度代替慢启动阶段的指数速度提升窗口大小。</p><p>当然，重复ACK意味着发送方可以提前重发丢失报文，快速重传算法定义了这一行为。同时，为了使得重发报文的过程中，发送速度不至于出现断崖式下降，TCP又定义了快速恢复算法，发送方在报文重新变得有序后，结束快速恢复进入拥塞避免阶段。</p><p>但以丢包作为网络拥塞的信号往往为时已晚，于是以BBR算法为代表的测量型拥塞控制算法应运而生。当飞行中报文数量不变，而网络时延升高时，就说明网络中的缓冲队列出现了积压，这是进行拥塞控制的最好时机。Linux高版本支持BBR算法，你可以通过tcp_congestion_control配置更改拥塞控制算法。</p><h2>思考题</h2><p>最后，请你思考下，快速恢复阶段的拥塞窗口，在报文变得有序后反而会缩小，这是为什么？欢迎你在留言区与大家一起探讨。</p><p>感谢阅读，如果你觉得这节课对你有一些启发，也欢迎把它分享给你的朋友。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/82/3d/356fc3d6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>忆水寒</span>
  </div>
  <div class="_2_QraFYR_0">1、关于课后思考题，快速恢复阶段的拥塞窗口，在报文变得有序后反而会缩小？<br>我想原因可能是 TCP觉得此时网络轻度拥塞，然后拥塞阈值ssthresh降低为cwnd 的一半，并设置cwnd为ssthresh，然后拥塞窗口再线性增长。<br><br>2、本文的一些思考与总结<br>首先由于慢启动拥塞窗口肯定不能无止境的指数级增长下去，否则拥塞控制就会失控。<br>这个时候，ssthresh 就是一道刹车，让拥塞窗口别涨那么快。<br>  当 cwnd&lt;ssthresh 时，拥塞窗口按指数级增长（慢启动）<br>  当 cwnd&gt;ssthresh 时，拥塞窗口按线性增长（拥塞避免）<br><br>  拥塞控制的几种算法：<br>慢启动：拥塞窗口一开始是一个很小的值，然后每 RTT 时间翻倍<br>拥塞避免：当拥塞窗口达到拥塞阈值（ssthresh）时，拥塞窗口从指数增长变为线性增长。<br>快速重传：发送端接收到 3 个重复 ACK 时立即进行重传。<br>快速恢复：当收到三次重复 ACK 时，进入快速恢复阶段，此时拥塞阈值降为之前的一半，然后进入线性增长阶段。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-26 20:08:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/11/78/4f0cd172.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>妥协</span>
  </div>
  <div class="_2_QraFYR_0">为什么报文5之后的ack都是ack6呀</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你好妥协，TCP是有序的字符流，因此接收方收完报文5后，只能接收报文6，但现在却接收到了报文7、8、9、10，此时接收方该怎么办呢？<br>当然，它可以当做不知道，什么也不做，坐等报文6的到来。报文6什么时候会到呢？RTO时间超时后，发送方会重发报文6，因为发送方一直没收到ACK7！<br>但是，RTO是很长的时间，接收方直接反复的传递ACK6，这样发送方就能明白，报文6丢了，他可以提前重发报文6. 这叫做快速重传！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-25 23:01:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/64/9b/d1ab239e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>J.Smile</span>
  </div>
  <div class="_2_QraFYR_0">总结一：<br>1 接收主机的处理能力不足时，是通过滑动窗口来减缓对方的发送速度。<br>2 接收主机的处理能力很强时，也无法通过增加发送方的发送窗口来提升发送速度，因为网络的传输速度是有限的，它会直接丢弃超过其处理能力的报文。<br>发送方只有在重传RTO时间超时后才会发现报文被丢弃然后重传报文。 解决这个问题办法是拥塞控制。<br>3 TCP慢启动<br>  TCP连接传输中会穿越多层网络并不清楚网络传输能力，所以一开始TCP会调低发送窗口大小，避免发送超过网络传输负载大小的报文。这样以来TCP发送报文速度就<br>  会变慢，这就是慢启动。TCP调低发送窗口(swnd)的大小是通过”拥塞窗口“---congestion window(cwnd)实现的。<br>4 如果不考虑网络拥塞，发送窗口就等于对方的接收窗口，而考虑了网络拥塞后，发送窗口则应当是拥塞窗口(cwnd)与对方接收窗口(rwnd)的最小值.<br>swnd = min(cwnd, rwnd),即发送速度就综合考虑了接收方和网络的处理能力。<br>5 通常用 MSS 作为描述窗口大小的单位，其中 MSS 是 TCP 报文的最大长度。虽然窗口的计量单位是字节。<br>6 拥塞窗口增长原理：<br>假如：初始拥塞窗口只有 1 个 MSS，MSS为1KB，RTT时延为100ms。发送速度=1KB&#47;100ms=10KB&#47;s<br>6.1 当没有发生拥塞时，拥塞窗口必须快速扩大，才能提高互联网的传输速度.<br>6.2 启动阶段会以指数级扩大拥塞窗口,发送方每收到一个 ACK 确认报文，拥塞窗口就增加 1 倍 MSS,比如最初的初始拥塞窗口（也称为 initcwnd）是 1 个 MSS，经过 4 个 RTT 就会变成 16 个 MSS。<br>6.3 互联网中的很多资源体积并不大， 2010 年 Google 对 Web 对象大小的 CDF 累积分布统计，大多数对象在 10KB 左右。<br>==》当 MSS 是 1KB 时，则多数 HTTP 请求至少包含 10 个报文才能下载完一个文件。以指数级增加拥塞窗口，经过4个RTT之后，才能收到大于10个MSS。<br>6.4 2013 年 TCP 的初始拥塞窗口调整到了 10 个 MSS，这样下载一个10KB的文件，只需要一个RTT就可以传输10KB的请求。<br>如果你需要传输的对象体积更大，BDP 带宽时延积很大时，完全可以继续提高初始拥塞窗口的大小<br>6.5 根据网络状况和传输对象的大小，调整初始拥塞窗口的大小<br>    ①ss 命令查看当前拥塞窗口：<br>    	ss -nli|fgrep cwnd<br>    ②ip route change 命令修改初始拥塞窗口：<br><br>		# ip route | while read r; do<br>		           ip route change $r initcwnd 10;<br>		       done</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 很好的总结！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-16 11:46:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/39/fa/a7edbc72.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>安排</span>
  </div>
  <div class="_2_QraFYR_0">毕竟还是发生了丢包或者延迟，所以快速恢复后重新进去拥塞避免。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-25 08:19:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/64/9b/d1ab239e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>J.Smile</span>
  </div>
  <div class="_2_QraFYR_0">总结2：<br>7 慢启动结束的三个场景<br>①慢启动阶段结束场景一（通过定时器明确探测到了丢包）：<br>发送数据后发送方在timeout时间内没有收到ack报文，说明报文丢失，网络拥塞。降低发送速度，即调小拥塞窗口大小，cubic可以将拥塞窗口降低为原先的0.8倍。同时记得设置慢启动阈值为发生网络拥塞之前的拥塞窗口的大小。<br>②慢启动阶段结束场景二（拥塞窗口的增长到达了慢启动阈值 ssthresh）：<br>慢启动阶段，拥塞窗口大小达到了慢启动阈值，很可能出现网络拥塞，为了避免拥塞，拥塞窗口增长速度应当更加保守的使用线性增长，而不是开始的指数增长。线性增长方式时每个RTT增加一个MSS，指数增长方式时每个ACK增加一倍MSS。<br>③慢启动阶段结束场景三（接收到重复的 ACK 报文，可能存在丢包）：<br>	如果发送方连续发送多个报文，在慢启动阶段第6个报文丢失。那么接收方就会接下来直接收到发送的第七个报文。接收方收到失序的第7个报文会触发快速重传算法，<br>立刻返回ACK6。以后收到的第8个、第9个报文都会返回ACK6，接收方直接反复的传递ACK6，这样发送方就能明白，报文6丢了，他可以提前重发报文6. 这叫做快速重传！<br>而不是等待发送方在RTO时间内没收到报文6的ACK响应才重传，那样就太慢了！<br>在触发快速重传之后，发送方会把慢启动阈值和拥塞窗口都降到之前的一半，进入”快速恢复阶段“。<br>此时发送方的拥塞窗口会由于接收到重复的ACK6，而增加相应个数的MSS。<br>发送方明白了报文6丢失之后，立即发送报文6给接收方，接收方这次收到了并给出了正常的响应。<br>后续不会再重复ACK6的响应了。然后才进入”拥塞避免阶段“（也就是从指数变线程增长）。<br><br><br>8 拥塞控制算法<br>慢启动、拥塞避免、快速重传、快速恢复，共同构成了拥塞控制算法<br>内核支持的拥塞控制算法列表：<br>net.ipv4.tcp_available_congestion_control = cubic reno<br>内核选择的拥塞控制算法：<br>net.ipv4.tcp_congestion_control = cubic<br>Linux 4.9 版本之后都支持 BBR 算法，是基于测量的拥塞控制算法，开启 BBR 算法仍然使用 tcp_congestion_control 配置：<br>net.ipv4.tcp_congestion_control=bbr</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Jeff.Smile的总结非常棒！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-16 11:46:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/60/71/895ee6cf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>分清云淡</span>
  </div>
  <div class="_2_QraFYR_0">bbr比较适合高rt场景，对机房内网性能反而更差，慎用</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-25 14:01:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>李新龙</span>
  </div>
  <div class="_2_QraFYR_0">收到重复的ACK后，报文是不是暂停发送了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不会啊</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-26 09:10:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/64/9b/d1ab239e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>J.Smile</span>
  </div>
  <div class="_2_QraFYR_0">”慢启动阶段会以指数级扩大拥塞窗口（扩大规则是这样的：发送方每收到一个 ACK 确认报文，拥塞窗口就增加 1 个 MSS）“<br>---------------------------------<br>老师，这一句是不是有问题，发送方收掉一个ack，是增加1倍的吧MSS，这样才跟后面一致。4个RTT就是4个ACK，那也就增加了4个MSS，就不是16了。。。<br>”比如最初的初始拥塞窗口（也称为 initcwnd）是 1 个 MSS，经过 4 个 RTT 就会变成 16 个 MSS“</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不是哦，比如cwnd为1时，每收到1个ACK，cwnd为2，再发出2个MSS，接着收到2个ACK，cwnd为4，依此类推，4个RTT后会变成16个MSS。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-16 11:28:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/47/12/2c47bf36.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_2b3614</span>
  </div>
  <div class="_2_QraFYR_0">扩大规则是这样的：发送方每收到一个 ACK 确认报文，拥塞窗口就增加 1 个 MSS---&gt; 貌似是增加一倍个mss？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-05 08:29:05</div>
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
  <div class="_2_QraFYR_0">我理解：快速重传是窗口虽然是8，但是在网络中传输的数据包数量是&lt;=4的，因为另外几个被“8，9，10，11”占了，所以进入拥塞避免阶段之后，继续保持8实际上是可能造成拥塞，不如恢复的已经被证明能够支撑的cwnd，然后再线性增长。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-25 20:35:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/64/05/6989dce6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我来也</span>
  </div>
  <div class="_2_QraFYR_0">第一次见BBR还是在搭建梯子的过程中.<br>那时ubuntu默认内核还不支持,需要手动开启.</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-25 17:30:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/d4/44/ec084136.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>LoveDlei</span>
  </div>
  <div class="_2_QraFYR_0">如果初始拥塞窗口只有 1 个 MSS，当 MSS 是 1KB，而 RTT 时延是 100ms 时，发送速度只有 10KB&#47;s。所以，当没有发生拥塞时，拥塞窗口必须快速扩大，才能提高互联网的传输速度。因此，慢启动阶段会以指数级扩大拥塞窗口（扩大规则是这样的：发送方每收到一个 ACK 确认报文，拥塞窗口就增加 1 个 MSS），比如最初的初始拥塞窗口（也称为 initcwnd）是 1 个 MSS，经过 4 个 RTT 就会变成 16 个 MSS。<br><br>这段没有整明白？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-10 14:04:06</div>
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
  <div class="_2_QraFYR_0">linux 可以控制 TCP 拥塞窗口 Cwnd 的大小 和 拥塞控制算法 net.ipv4.tcp_congestion_control 这个有两个值： bbr 基于带宽时延的   cubic 基于丢包的</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-17 17:31:16</div>
  </div>
</div>
</div>
</li>
</ul>