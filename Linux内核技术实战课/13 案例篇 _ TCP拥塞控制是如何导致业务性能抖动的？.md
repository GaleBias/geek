<audio title="13 案例篇 _ TCP拥塞控制是如何导致业务性能抖动的？" src="https://static001.geekbang.org/resource/audio/a5/28/a5691c482726b20a11ce541ea913af28.mp3" controls="controls"></audio> 
<p>你好，我是邵亚方。这节课我来跟大家分享TCP拥塞控制与业务性能抖动之间的关系。</p><p>TCP拥塞控制是TCP协议的核心，而且是一个非常复杂的过程。如果你不了解TCP拥塞控制的话，那么就相当于不理解TCP协议。这节课的目的是通过一些案例，介绍在TCP拥塞控制中我们要避免踩的一些坑，以及针对TCP性能调优时需要注意的一些点。</p><p>因为在TCP传输过程中引起问题的案例有很多，所以我不会把这些案例拿过来具体去一步步分析，而是希望能够对这些案例做一层抽象，把这些案例和具体的知识点结合起来，这样会更有系统性。并且，在你明白了这些知识点后，案例的分析过程就相对简单了。</p><p>我们在前两节课（<a href="https://time.geekbang.org/column/article/284912">第11讲</a>和<a href="https://time.geekbang.org/column/article/285816">第12讲</a>）中讲述了单机维度可能需要注意的问题点。但是，网络传输是一个更加复杂的过程，这中间涉及的问题会更多，而且更加不好分析。相信很多人都有过这样的经历：</p><ul>
<li>等电梯时和别人聊着微信，进入电梯后微信消息就发不出去了；</li>
<li>和室友共享同一个网络，当玩网络游戏玩得正开心时，游戏忽然卡得很厉害，原来是室友在下载电影；</li>
<li>使用ftp上传一个文件到服务器上，没想到要上传很久；</li>
<li>……</li>
</ul><p>在这些问题中，TCP的拥塞控制就在发挥着作用。</p><h2>TCP拥塞控制是如何对业务网络性能产生影响的 ？</h2><!-- [[[read_end]]] --><p>我们先来看下TCP拥塞控制的大致原理。</p><p><img src="https://static001.geekbang.org/resource/image/5c/3c/5c4504d70ce3abc939yyca54780dd43c.jpg?wh=3055*2029" alt="" title="TCP拥塞控制"></p><p>上图就是TCP拥塞控制的简单图示，它大致分为四个阶段。</p><h4>1. 慢启动</h4><p>TCP连接建立好后，发送方就进入慢速启动阶段，然后逐渐地增大发包数量（TCP Segments）。这个阶段每经过一个RTT（round-trip time），发包数量就会翻倍。如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/05/4d/0534ce8d1e3a09a1def9c27e387eb64d.jpg?wh=2873*1919" alt="" title="TCP Slow Start示意图"></p><p>初始发送数据包的数量是由init_cwnd（初始拥塞窗口）来决定的，该值在Linux内核中被设置为10（TCP_INIT_CWND），这是由Google的研究人员总结出的一个经验值，这个经验值也被写入了<a href="https://tools.ietf.org/html/rfc6928">RFC6928</a>。并且，Linux内核在2.6.38版本中也将它从默认值3修改为了Google建议的10，你感兴趣的话可以看下这个commit：  <a href="https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=442b9635c569fef038d5367a7acd906db4677ae1">tcp: Increase the initial congestion window to 10</a>。</p><p>增大init_cwnd可以显著地提升网络性能，因为这样在初始阶段就可以一次性发送很多TCP Segments，更加细节性的原因你可以参考<a href="https://tools.ietf.org/html/rfc6928">RFC6928</a>的解释。</p><p>如果你的内核版本比较老（低于CentOS-6的内核版本），那不妨考虑增加init_cwnd到10。如果你想要把它增加到一个更大的值，也不是不可以，但是你需要根据你的网络状况多做一些实验，从而得到一个较为理想的值。因为如果初始拥塞窗口设置得过大的话，可能会引起很高的TCP重传率。当然，你也可以通过ip route的方式来更加灵活地调整该值，甚至将它配置为一个sysctl控制项。</p><p>增大init_cwnd的值对于提升短连接的网络性能会很有效，特别是数据量在慢启动阶段就能发送完的短连接，比如针对http这种服务，http的短连接请求数据量一般不大，通常在慢启动阶段就能传输完，这些都可以通过tcpdump来进行观察。</p><p>在慢启动阶段，当拥塞窗口（cwnd）增大到一个阈值（ ssthresh，慢启动阈值）后，TCP拥塞控制就进入了下一个阶段：拥塞避免（Congestion Avoidance）。</p><h4>2.拥塞避免</h4><p>在这个阶段cwnd不再成倍增加，而是一个RTT增加1，即缓慢地增加cwnd，以防止网络出现拥塞。网络出现拥塞是难以避免的，由于网络链路的复杂性，甚至会出现乱序（Out of Order）报文。乱序报文产生原因之一如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/0c/99/0c2ce093d74a1dc76f39b7cbdd386699.jpg?wh=2716*1817" alt="" title="TCP乱序报文"></p><p>在上图中，发送端一次性发送了4个TCP segments，但是第2个segment在传输过程中被丢弃掉了，那么接收方就接收不到该segment了。然而第3个TCP segment和第4个TCP segment能够被接收到，此时3和4就属于乱序报文，它们会被加入到接收端的ofo queue（乱序队列）里。</p><p>丢包这类问题在移动网络环境中比较容易出现，特别是在一个网络状况不好的环境中，比如在电梯里丢包率就会很高，而丢包率高就会导致网络响应特别慢。在数据中心内部的服务上很少会有数据包在网络链路中被丢弃的情况，我说的这类丢包问题主要是针对网关服务这种和外部网络有连接的服务上。</p><p>针对我们的网关服务，我们自己也做过一些TCP单边优化工作，主要是优化Cubic拥塞控制算法，以缓解丢包引起的网络性能下降问题。另外，Google前几年开源的一个新的<a href="https://github.com/google/bbr">拥塞控制算法BBR</a>在理论上也可以很好地缓解TCP丢包问题，但是在我们的实践中，BBR的效果并不好，因此我们最终也没有使用它。</p><p>我们再回到上面这张图，因为接收端没有接收到第2个segment，因此接收端每次收到一个新的segment后都会去ack第2个segment，即ack 17。紧接着，发送端就会接收到三个相同的ack（ack 17）。连续出现了3个响应的ack后，发送端会据此判断数据包出现了丢失，于是就进入了下一个阶段：快速重传。</p><h4>3.快速重传和快速恢复</h4><p>快速重传和快速恢复是一起工作的，它们是为了应对丢包这种行为而做的优化，在这种情况下，由于网络并没有出现拥塞，所以拥塞窗口不必恢复到初始值。判断丢包的依据就是收到3个相同的ack。</p><p>Google的工程师同样对TCP快速重传提出了一个改进策略：<a href="https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=eed530b6c67624db3f2cf477bac7c4d005d8f7ba">tcp early retrans</a>，它允许一些情况下的TCP连接可以绕过重传延时（RTO）来进行快速重传。3.6版本以后的内核都支持了这个特性，因此，如果你还在使用CentOS-6，那么就享受不到它带来的网络性能提升了，你可以将你的操作系统升级为CentOS-7或者最新的CentOS-8。 另外，再多说一句，Google在网络方面的技术实力是其他公司没法比的，Linux内核TCP子系统的maintainer也是Google的工程师（Eric Dumazet）。</p><p>除了快速重传外，还有一种重传机制是超时重传。不过，这是非常糟糕的一种情况。如果发送出去一个数据包，超过一段时间（RTO）都收不到它的ack，那就认为是网络出现了拥塞。这个时候就需要将cwnd恢复为初始值，再次从慢启动开始调整cwnd的大小。</p><p>RTO一般发生在网络链路有拥塞的情况下，如果某一个连接数据量太大，就可能会导致其他连接的数据包排队，从而出现较大的延迟。我们在开头提到的，下载电影影响到别人玩网络游戏的例子就是这个原因。</p><p>关于RTO，它也是一个优化点。如果RTO过大的话，那么业务就可能要阻塞很久，所以在3.1版本的内核里引入了一种改进来将RTO的初始值<a href="https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?h=v5.9-rc2&amp;id=9ad7c049f0f79c418e293b1b68cf10d68f54fcdb">从3s调整为1s</a>，这可以显著节省业务的阻塞时间。不过，RTO=1s  在某些场景下还是有些大了，特别是在数据中心内部这种网络质量相对比较稳定的环境中。</p><p>我们在生产环境中发生过这样的案例：业务人员反馈说业务RT抖动得比较厉害，我们使用strace初步排查后发现，进程阻塞在了send()这类发包函数里。然后我们使用tcpdump来抓包，发现发送方在发送数据后，迟迟不能得到对端的响应，一直到RTO时间再次重传。与此同时，我们还尝试了在对端也使用tcpdump来抓包，发现对端是过了很长时间后才收到数据包。因此，我们判断是网络发生了拥塞，从而导致对端没有及时收到数据包。</p><p>那么，针对这种网络拥塞引起业务阻塞时间太久的情况，有没有什么解决方案呢？一种解决方案是，创建TCP连接，使用SO_SNDTIMEO来设置发送超时时间，以防止应用在发包的时候阻塞在发送端太久，如下所示：</p><blockquote>
<p>ret = setsockopt(sockfd, SOL_SOCKET, SO_SNDTIMEO, &amp;timeout, len);</p>
</blockquote><p>当业务发现该TCP连接超时后，就会主动断开该连接，然后尝试去使用其他的连接。</p><p>这种做法可以针对某个TCP连接来设置RTO时间，那么，有没有什么方法能够设置全局的RTO时间（设置一次，所有的TCP连接都能生效）呢？答案是有的，这就需要修改内核。针对这类需求，我们在生产环境中的实践是：将TCP RTO min、TCP RTO max、TCP RTO init 更改为可以使用sysctl来灵活控制的变量，从而根据实际情况来做调整，比如说针对数据中心内部的服务器，我们可以适当地调小这几个值，从而减少业务阻塞时间。</p><p>上述这4个阶段是TCP拥塞控制的基础，总体来说，拥塞控制就是根据TCP的数据传输状况来灵活地调整拥塞窗口，从而控制发送方发送数据包的行为。换句话说，拥塞窗口的大小可以表示网络传输链路的拥塞情况。TCP连接cwnd的大小可以通过ss这个命令来查看：</p><pre><code>$ ss -nipt
State       Recv-Q Send-Q                        Local Address:Port                                       Peer Address:Port         
ESTAB       0      36                             172.23.245.7:22                                        172.30.16.162:60490      
users:((&quot;sshd&quot;,pid=19256,fd=3))
	 cubic wscale:5,7 rto:272 rtt:71.53/1.068 ato:40 mss:1248 rcvmss:1248 advmss:1448 cwnd:10 bytes_acked:19591 bytes_received:2817 segs_out:64 segs_in:80 data_segs_out:57 data_segs_in:28 send 1.4Mbps lastsnd:6 lastrcv:6 lastack:6 pacing_rate 2.8Mbps delivery_rate 1.5Mbps app_limited busy:2016ms unacked:1 rcv_space:14600 minrtt:69.402
</code></pre><p>通过该命令，我们可以发现这个TCP连接的cwnd为10。</p><p>如果你想要追踪拥塞窗口的实时变化信息，还有另外一个更好的办法：通过tcp_probe这个tracepoint来追踪：</p><pre><code>/sys/kernel/debug/tracing/events/tcp/tcp_probe
</code></pre><p>但是这个tracepoint只有4.16以后的内核版本才支持，如果你的内核版本比较老，你也可以使用tcp_probe这个内核模块（net/ipv4/tcp_probe.c）来进行追踪。</p><p>除了网络状况外，发送方还需要知道接收方的处理能力。如果接收方的处理能力差，那么发送方就必须要减缓它的发包速度，否则数据包都会挤压在接收方的缓冲区里，甚至被接收方给丢弃掉。接收方的处理能力是通过另外一个窗口——rwnd（接收窗口）来表示的。那么，接收方的rwnd又是如何影响发送方的行为呢？</p><h2>接收方是如何影响发送方发送数据的？</h2><p>同样地，我也画了一张简单的图，来表示接收方的rwnd是如何影响发送方的：</p><p><img src="https://static001.geekbang.org/resource/image/e9/27/e920b93740d9677c5419dee332086827.jpg?wh=2877*1843" alt="" title="rwnd与cwnd"></p><p>如上图所示，接收方在收到数据包后，会给发送方回一个ack，然后把自己的rwnd大小写入到TCP头部的win这个字段，这样发送方就能根据这个字段来知道接收方的rwnd了。接下来，发送方在发送下一个TCP segment的时候，会先对比发送方的cwnd和接收方的rwnd，得出这二者之间的较小值，然后控制发送的TCP segment个数不能超过这个较小值。</p><p>关于接收方的rwnd对发送方发送行为的影响，我们曾经遇到过这样的案例：业务反馈说Server向Client发包很慢，但是Server本身并不忙，而且网络看起来也没有问题，所以不清楚是什么原因导致的。对此，我们使用tcpdump在server上抓包后发现，Client响应的ack里经常出现win为0的情况，也就是Client的接收窗口为0。于是我们就去Client上排查，最终发现是Client代码存在bug，从而导致无法及时读取收到的数据包。</p><p>对于这种行为，我同样给Linux内核写了一个patch来监控它：<a href="https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=fb223502ec0889444965f602f57b1f45f9e9845e">tcp: add SNMP counter for zero-window drops</a> 。这个patch里增加了一个新的SNMP 计数：TCPZeroWindowDrop。如果系统中发生了接收窗口太小而无法收包的情况，就会产生该事件，然后该事件可以通过/proc/net/netstat里的TCPZeroWindowDrop这个字段来查看。</p><p>因为TCP头部大小是有限制的，而其中的win这个字段只有16bit，win能够表示的大小最大只有65535（64K），所以如果想要支持更大的接收窗口以满足高性能网络，我们就需要打开下面这个配置项，系统中也是默认打开了该选项：</p><blockquote>
<p>net.ipv4.tcp_window_scaling = 1</p>
</blockquote><p>关于该选项更加详细的设计，你如果想了解的话，可以去参考<a href="https://tools.ietf.org/html/rfc1323">RFC1323</a>。</p><p>好了，关于TCP拥塞控制对业务网络性能的影响，我们就先讲到这里。</p><h2>课堂总结</h2><p>TCP拥塞控制是一个非常复杂的行为，我们在这节课里讲到的内容只是其中一些基础部分，希望这些基础知识可以让你对TCP拥塞控制有个大致的了解。我来总结一下这节课的重点：</p><ul>
<li>网络拥塞状况会体现在TCP连接的拥塞窗口（cwnd）中，该拥塞窗口会影响发送方的发包行为；</li>
<li>接收方的处理能力同样会反馈给发送方，这个处理是通过rwnd来表示的。rwnd和cwnd会共同作用于发送方，来决定发送方最大能够发送多少TCP包；</li>
<li>TCP拥塞控制的动态变化，可以通过tcp_probe这个tracepoint（对应4.16+的内核版本）或者是tcp_probe这个内核模块（对应4.16之前的内核版本）来进行实时观察，通过tcp_probe你能够很好地观察到TCP连接的数据传输状况。</li>
</ul><h2>课后作业</h2><p>通过ssh登录到服务器上，然后把网络关掉，过几秒后再打开，请问这个ssh连接还正常吗？为什么？欢迎你在留言区与我讨论。</p><p>感谢你的阅读，如果你认为这节课的内容有收获，也欢迎把它分享给你的朋友，我们下一讲见。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/3b/d7/9d942870.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>邵亚方</span>
  </div>
  <div class="_2_QraFYR_0">课后作业答案：<br>- 通过 ssh 登录到服务器上，然后把网络关掉，过几秒后再打开，请问这个 ssh 连接还正常吗？为什么？<br>ssh使用的TCP协议，也就是它是有连接的，这个连接对内核而言就是tcp_sock这个结构体，这个结构体会记录该TCP连接的状态，包括四元组(src_ip:src_port - dst_ip:dst_port)，该连接的路由信息也被保存着。关闭网络后，该TCP连接的这些信息都还在，如果两边没有数据交互的话，这些信息就不会被更新，也就你一直存在着，当你再次打开网络，该连接还可以正常使用；假如关闭网络后，该TCP连接有数据交互，此时就会检查到异常，同样的也会去更新TCP连接状态信息，就有可能会断开这个连接。除此之外，TCP还有keepalive机制，如果该连接长时间没有数据，keepalive机制也会把该连接给关闭掉。<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-11 13:58:03</div>
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
  <div class="_2_QraFYR_0">课后思考题：<br>这个应该是不确定的。<br><br>以前也测试过，只要在网络断开期间不主动发送数据，就会晚一点探测到连接已断开。<br>如果不主动发数据，可能网络恢复时，连接就自动恢复了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对的！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-18 00:06:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/17/96/a10524f5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>计算机漫游</span>
  </div>
  <div class="_2_QraFYR_0">我们金融交易平台的生产环境出现过一个案例，每天早上9点开盘会有用户集中登录的情况，其中登录链路中有两个服务，部署在两台服务器上，事故当天就出现了很多用户无法登陆的情况，开发人员排查日志发现这两个服务之间的通信有非常大的延时，A服务发的消息，B服务过了很久才收到，时间5分钟到20分钟不等，我们运维小伙伴监控服务器的负载非常低，CPU，内存，io都很正常，甚至我们还有专门的程序每秒探测内网机器的存活，ping包每秒一次，延时也都在毫秒级别。所以当时判断可能是两个服务之间的tcp链接出了问题，我比较怀疑是接收方窗口变为0了，但苦于没有抓包无法验证猜想，并且此类事件再也没有出现，但是心里一直存在疑惑，所以想问问老师，在您看来这种情况比较可能原因有哪些？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 看你的描述，这类问题大概率跟TCP缓冲区有关系。A服务所在机器的发送缓冲区太小，或者B服务所在服务器的接受缓冲区太小，都会导致缓冲区排队严重，甚至引起丢包，接受窗口变为0也可能会出现。<br>我建议你可以适当的增加TCP缓冲区大小。<br>你可以通过ss -natp来看是否存在数据积压的情况。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-07 06:57:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/dwehJHP4ycAfDb9MoudXb4QSt7YgmISqwwsa928XZ6aTWqwWh0kx0iatjocSibLa7iajXmbGlJ5svegY3P6LfKJ0w/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>solar</span>
  </div>
  <div class="_2_QraFYR_0">cwnd和rwnd使用的单位是什么呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: TCP segment个数</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-10 18:18:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/a8PMLmCTCBa40j7JIy3d8LsdbW5hne7lkk9KOGQuiaeVk4cn06KWwlP3ic69BsQLpNFtRTjRdUM2ySDBAv1MOFfA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Ilovek8s</span>
  </div>
  <div class="_2_QraFYR_0">keepalive心跳的时间里如果不发送ssh命令操作，断开网络之后再重新打开，由于TCP有重试机制，是可以恢复的，但如果keepalive开始检查了，服务端发现客户端是死的之后就会关闭连接</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-14 10:33:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/48/5e/36d96d40.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>redseed</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，去年公司接入了跨境专线，使用默认对 CUBIC 算法时 TCP 的流量极不稳定，根据网上的优化建议增大了 TCP 的 sendbuf 适应这类高延迟网络，但是 TCP 的传输带宽反而下降了，想请教一下可能的原因出现在哪里？（PS. 后面我们使用了 BBR 算法并增大 sendbuf，这个对长肥管道有奇效...）</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-09 10:47:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/f9/e6/47742988.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>webmin</span>
  </div>
  <div class="_2_QraFYR_0">要分情况：<br>1. 如果关闭网络是发生在Client端或Server端的机器上，那么网络恢复后连接不会正常；<br>2. 如果关闭网络是发生在Client端与Server端之间链路中的某个路由节点上；<br>    2.1 Client端到Server端之间有多条路可用，只要不是和CS直连这个设备有问题，那么设备可以选择其它路走，这时连接还是正常的；<br>    2.2 Client端与Server端之间链路有NAT，且网络关闭发生与NAT端相关的设备上，那么连接就不正常； <br>    2.3 Client端与Server端之间只有一条路，只要不是和CS直连这个设备有问题，那么如果网络在发生tcp_keepalive之前恢复，那么连接还是正常的； <br> 3. 以上讨论的都是在TCP&#47;IP协议情况下，网上查了一下SSH有居于UDP的方案（http:&#47;&#47;publications.lib.chalmers.se&#47;records&#47;fulltext&#47;123799.pdf），如果走UDP的话这个要看SSH应用层的保活或SESSION有效期是否超过网络关闭时间，大小则可以连接，小于则关闭。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 很赞！总结的比较全面，很多因素都考虑到了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-18 12:17:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/5a/56/115c6433.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jssfy</span>
  </div>
  <div class="_2_QraFYR_0">引用：对此，我们使用 tcpdump 在 server 上抓包后发现，Client 响应的 ack 里经常出现 win 为 0 的情况，也就是 Client 的接收窗口为 0。于是我们就去 Client 上排查，最终发现是 Client 代码存在 bug，从而导致无法及时读取收到的数据包。<br><br>请问这里的前一句的接收窗口为0和后一句的代码bug是有什么逻辑关系吗？这里没太看懂</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 哦 是应用被阻塞住 没有及时从缓冲区里读取数据 导致缓冲区满</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-19 03:01:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/00/8e/ebe3c8ea.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>董泽润</span>
  </div>
  <div class="_2_QraFYR_0">连接是否正常，要看是否开启了 tcp_keepalive, 并且探测持续失败，连接才失效</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-17 14:51:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/ed/91/1d332031.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我能走多远</span>
  </div>
  <div class="_2_QraFYR_0">感谢老师分享，学习了拥塞控制的原理，慢启动，拥塞避免，快速重传和快速恢复。cwnd和rwnd使用的单位是什么呢？ TCP segment个数。每个segment的长度就是mss的大小吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: segment的大小最大是mss，最小的话就只是tcp header。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-09 13:08:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/37/a0/032d0828.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>上杉夏香</span>
  </div>
  <div class="_2_QraFYR_0">老师，有个问题。「我们再回到上面这张图，因为接收端没有接收到第 2 个 segment，因此接收端每次收到一个新的 segment 后都会去 ack 第 2 个 segment，即 ack 17。紧接着，发送端就会接收到三个相同的 ack（ack 17）。连续出现了 3 个响应的 ack 后，发送端会据此判断数据包出现了丢失，于是就进入了下一个阶段：快速重传。」我的理解，应该是连续出现 3 个 Dup Ack 才会导致快速重传，进而进入快速恢复阶段吧。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-03 23:02:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/37/a0/032d0828.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>上杉夏香</span>
  </div>
  <div class="_2_QraFYR_0">老师，有个问题。原文为「我们再回到上面这张图，因为接收端没有接收到第 2 个 segment，因此接收端每次收到一个新的 segment 后都会去 ack 第 2 个 segment，即 ack 17。紧接着，发送端就会接收到三个相同的 ack（ack 17）。连续出现了 3 个响应的 ack 后，发送端会据此判断数据包出现了丢失，于是就进入了下一个阶段：快速重传。」</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-03 23:01:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/b5/1d/f2f66e8d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>团团-BB</span>
  </div>
  <div class="_2_QraFYR_0">老师我遇到一个有一系列pod运行的应用，其中有1-2个pod的响应时间抖动比其他pod严重很多，流量相对是均衡的，抖动的pod我们看有出现丢包的情况。<br>出现这种情况的pod所在的宿主机节点网卡的流量比较高，节点网卡eth0有drop包，和应用的延时徒增时间大致吻合，查看网卡rx有drop包的情况。因为是使用的公有云的基础设施，这种情况下，怎么能排除是否是基础设施网络存在的问题呢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-02 09:07:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/4d/49/28e73b9c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>明翼</span>
  </div>
  <div class="_2_QraFYR_0">&quot;如上图所示，接收方在收到数据包后，会给发送方回一个 ack，然后把自己的 rwnd 大小写入到 TCP 头部的 win 这个字段，这样发送方就能根据这个字段来知道接收方的 rwnd 了。接下来，发送方在发送下一个 TCP segment 的时候，会先对比发送方的 cwnd 和接收方的 rwnd，得出这二者之间的较小值，然后控制发送的 TCP segment 个数不能超过这个较小值。&quot;<br><br>老师你好，这段话说发送的tcp报文个数不能超过拥塞控制窗口和接受窗口的最小值，怎么我在其他资料上看到接受窗口单位是字节数，我哪里理解错了，谢谢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-18 09:13:38</div>
  </div>
</div>
</div>
</li>
</ul>