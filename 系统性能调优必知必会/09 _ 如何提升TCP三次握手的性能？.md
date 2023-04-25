<audio title="09 _ 如何提升TCP三次握手的性能？" src="https://static001.geekbang.org/resource/audio/72/9e/726a6d37e4a50655c75344829823169e.mp3" controls="controls"></audio> 
<p>你好，我是陶辉。</p><p>上一讲我们提到TCP在三次握手建立连接、四次握手关闭连接时是怎样产生事件的，这两个过程中TCP连接经历了复杂的状态变化，既容易导致编程出错，也有很大的优化空间。这一讲我们看看在Linux操作系统下，如何优化TCP的三次握手流程，提升握手速度。</p><p>TCP是一个可以双向传输的全双工协议，所以需要经过三次握手才能建立连接。三次握手在一个HTTP请求中的平均时间占比在10%以上，在网络状况不佳、高并发或者遭遇SYN泛洪攻击等场景中，如果不能正确地调整三次握手中的参数，就会对性能有很大的影响。</p><p>TCP协议是由操作系统实现的，调整TCP必须通过操作系统提供的接口和工具，这就需要理解Linux是怎样把三次握手中的状态暴露给我们，以及通过哪些工具可以找到优化依据，并通过哪些接口修改参数。</p><p>因此，这一讲我们将介绍TCP握手过程中各状态的意义，并以状态变化作为主线，看看如何调整Linux参数才能提升握手的性能。</p><h2>客户端的优化</h2><p>客户端和服务器都可以针对三次握手优化性能。相对而言，主动发起连接的客户端优化相对简单一些，而服务器需要在监听端口上被动等待连接，并保存许多握手的中间状态，优化方法更为复杂一些。我们首先来看如何优化客户端。</p><!-- [[[read_end]]] --><p>三次握手建立连接的首要目的是同步序列号。只有同步了序列号才有可靠的传输，TCP协议的许多特性都是依赖序列号实现的，比如流量控制、消息丢失后的重发等等，这也是三次握手中的报文被称为SYN的原因，因为SYN的全称就叫做Synchronize Sequence Numbers。</p><p><img src="https://static001.geekbang.org/resource/image/c5/aa/c51d9f1604690ab1b69e7c4feb2f31aa.jpg?wh=2052*620" alt=""></p><p>三次握手虽然由操作系统实现，但它通过连接状态把这一过程暴露给了我们，我们来细看下过程中出现的3种状态的意义。客户端发送SYN开启了三次握手，此时在客户端上用netstat命令（后续查看连接状态都使用该命令）可以看到<strong>连接的状态是SYN_SENT</strong>（顾名思义，就是把刚SYN发送出去）。</p><pre><code>tcp    0   1 172.16.20.227:39198     129.28.56.36:81         SYN_SENT
</code></pre><p>客户端在等待服务器回复的ACK报文。正常情况下，服务器会在几毫秒内返回ACK，但如果客户端迟迟没有收到ACK会怎么样呢？客户端会重发SYN，<strong>重试的次数由tcp_syn_retries参数控制</strong>，默认是6次：</p><pre><code>net.ipv4.tcp_syn_retries = 6
</code></pre><p>第1次重试发生在1秒钟后，接着会以翻倍的方式在第2、4、8、16、32秒共做6次重试，最后一次重试会等待64秒，如果仍然没有返回ACK，才会终止三次握手。所以，总耗时是1+2+4+8+16+32+64=127秒，超过2分钟。</p><p>如果这是一台有明确任务的服务器，你可以根据网络的稳定性和目标服务器的繁忙程度修改重试次数，调整客户端的三次握手时间上限。比如内网中通讯时，就可以适当调低重试次数，尽快把错误暴露给应用程序。</p><p><img src="https://static001.geekbang.org/resource/image/a3/8f/a3c5e77a228478da2a6e707054043c8f.png?wh=943*613" alt=""></p><h2>服务器端的优化</h2><p>当服务器收到SYN报文后，服务器会立刻回复SYN+ACK报文，既确认了客户端的序列号，也把自己的序列号发给了对方。此时，服务器端出现了新连接，状态是SYN_RCV（RCV是received的缩写）。这个状态下，服务器必须建立一个SYN半连接队列来维护未完成的握手信息，当这个队列溢出后，服务器将无法再建立新连接。</p><p><img src="https://static001.geekbang.org/resource/image/c3/82/c361e672526ee5bb87d5f6b7ad169982.png?wh=690*304" alt=""></p><p>新连接建立失败的原因有很多，怎样获得由于队列已满而引发的失败次数呢？netstat -s命令给出的统计结果中可以得到。</p><pre><code># netstat -s | grep &quot;SYNs to LISTEN&quot;
    1192450 SYNs to LISTEN sockets dropped
</code></pre><p>这里给出的是队列溢出导致SYN被丢弃的个数。注意这是一个累计值，如果数值在持续增加，则应该调大SYN半连接队列。<strong>修改队列大小的方法，是设置Linux的tcp_max_syn_backlog 参数：</strong></p><pre><code>net.ipv4.tcp_max_syn_backlog = 1024
</code></pre><p>如果SYN半连接队列已满，只能丢弃连接吗？并不是这样，<strong>开启syncookies功能就可以在不使用SYN队列的情况下成功建立连接。</strong>syncookies是这么做的：服务器根据当前状态计算出一个值，放在己方发出的SYN+ACK报文中发出，当客户端返回ACK报文时，取出该值验证，如果合法，就认为连接建立成功，如下图所示。</p><p><img src="https://static001.geekbang.org/resource/image/0d/c0/0d963557347c149a6270d8102d83e0c0.png?wh=690*319" alt=""></p><p>Linux下怎样开启syncookies功能呢？修改tcp_syncookies参数即可，其中值为0时表示关闭该功能，2表示无条件开启功能，而1则表示仅当SYN半连接队列放不下时，再启用它。由于syncookie仅用于应对SYN泛洪攻击（攻击者恶意构造大量的SYN报文发送给服务器，造成SYN半连接队列溢出，导致正常客户端的连接无法建立），这种方式建立的连接，许多TCP特性都无法使用。所以，应当把tcp_syncookies设置为1，仅在队列满时再启用。</p><pre><code>net.ipv4.tcp_syncookies = 1
</code></pre><p>当客户端接收到服务器发来的SYN+ACK报文后，就会回复ACK去通知服务器，同时己方连接状态从SYN_SENT转换为ESTABLISHED，表示连接建立成功。服务器端连接成功建立的时间还要再往后，到它收到ACK后状态才变为ESTABLISHED。</p><p>如果服务器没有收到ACK，就会一直重发SYN+ACK报文。当网络繁忙、不稳定时，报文丢失就会变严重，此时应该调大重发次数。反之则可以调小重发次数。<strong>修改重发次数的方法是，调整tcp_synack_retries参数：</strong></p><pre><code>net.ipv4.tcp_synack_retries = 5
</code></pre><p>tcp_synack_retries 的默认重试次数是5次，与客户端重发SYN类似，它的重试会经历1、2、4、8、16秒，最后一次重试后等待32秒，若仍然没有收到ACK，才会关闭连接，故共需要等待63秒。</p><p>服务器收到ACK后连接建立成功，此时，内核会把连接从SYN半连接队列中移出，再移入accept队列，等待进程调用accept函数时把连接取出来。如果进程不能及时地调用accept函数，就会造成accept队列溢出，最终导致建立好的TCP连接被丢弃。</p><p>实际上，丢弃连接只是Linux的默认行为，我们还可以选择向客户端发送RST复位报文，告诉客户端连接已经建立失败。打开这一功能需要将tcp_abort_on_overflow参数设置为1。</p><pre><code>net.ipv4.tcp_abort_on_overflow = 0
</code></pre><p><strong>通常情况下，应当把tcp_abort_on_overflow设置为0，因为这样更有利于应对突发流量。</strong>举个例子，当accept队列满导致服务器丢掉了ACK，与此同时，客户端的连接状态却是ESTABLISHED，进程就在建立好的连接上发送请求。只要服务器没有为请求回复ACK，请求就会被多次重发。如果服务器上的进程只是短暂的繁忙造成accept队列满，那么当accept队列有空位时，再次接收到的请求报文由于含有ACK，仍然会触发服务器端成功建立连接。所以，<strong>tcp_abort_on_overflow设为0可以提高连接建立的成功率，只有你非常肯定accept队列会长期溢出时，才能设置为1以尽快通知客户端。</strong></p><p>那么，怎样调整accept队列的长度呢？<strong>listen函数的backlog参数就可以设置accept队列的大小。事实上，backlog参数还受限于Linux系统级的队列长度上限，当然这个上限阈值也可以通过somaxconn参数修改。</strong></p><pre><code>net.core.somaxconn = 128
</code></pre><p>当下各监听端口上的accept队列长度可以通过ss -ltn命令查看，但accept队列长度是否需要调整该怎么判断呢？还是通过netstat -s命令给出的统计结果，可以看到究竟有多少个连接因为队列溢出而被丢弃。</p><pre><code># netstat -s | grep &quot;listen queue&quot;
    14 times the listen queue of a socket overflowed
</code></pre><p>如果持续不断地有连接因为accept队列溢出被丢弃，就应该调大backlog以及somaxconn参数。</p><h2>TFO技术如何绕过三次握手？</h2><p>以上我们只是在对三次握手的过程进行优化。接下来我们看看如何绕过三次握手发送数据。</p><p>三次握手建立连接造成的后果就是，HTTP请求必须在一次RTT（Round Trip Time，从客户端到服务器一个往返的时间）后才能发送，Google对此做的统计显示，三次握手消耗的时间，在HTTP请求完成的时间占比在10%到30%之间。</p><p><img src="https://static001.geekbang.org/resource/image/1b/a8/1b9d8f49d5a716470481657b07ae77a8.png?wh=1090*779" alt=""></p><p>因此，Google提出了TCP fast open方案（简称<a href="https://tools.ietf.org/html/rfc7413">TFO</a>），客户端可以在首个SYN报文中就携带请求，这节省了1个RTT的时间。</p><p>接下来我们就来看看，TFO具体是怎么实现的。</p><p><strong>为了让客户端在SYN报文中携带请求数据，必须解决服务器的信任问题。</strong>因为此时服务器的SYN报文还没有发给客户端，客户端是否能够正常建立连接还未可知，但此时服务器需要假定连接已经建立成功，并把请求交付给进程去处理，所以服务器必须能够信任这个客户端。</p><p>TFO到底怎样达成这一目的呢？它把通讯分为两个阶段，第一阶段为首次建立连接，这时走正常的三次握手，但在客户端的SYN报文会明确地告诉服务器它想使用TFO功能，这样服务器会把客户端IP地址用只有自己知道的密钥加密（比如AES加密算法），作为Cookie携带在返回的SYN+ACK报文中，客户端收到后会将Cookie缓存在本地。</p><p>之后，如果客户端再次向服务器建立连接，就可以在第一个SYN报文中携带请求数据，同时还要附带缓存的Cookie。很显然，这种通讯方式下不能再采用经典的“先connect再write请求”这种编程方法，而要改用sendto或者sendmsg函数才能实现。</p><p>服务器收到后，会用自己的密钥验证Cookie是否合法，验证通过后连接才算建立成功，再把请求交给进程处理，同时给客户端返回SYN+ACK。虽然客户端收到后还会返回ACK，但服务器不等收到ACK就可以发送HTTP响应了，这就减少了握手带来的1个RTT的时间消耗。</p><p><img src="https://static001.geekbang.org/resource/image/7a/c3/7ac29766ba8515eea5bb331fce6dc2c3.png?wh=961*806" alt=""></p><p>当然，为了防止SYN泛洪攻击，服务器的TFO实现必须能够自动化地定时更新密钥。</p><p>Linux下怎么打开TFO功能呢？这要通过tcp_fastopen参数。由于只有客户端和服务器同时支持时，TFO功能才能使用，<strong>所以tcp_fastopen参数是按比特位控制的。其中，第1个比特位为1时，表示作为客户端时支持TFO；第2个比特位为1时，表示作为服务器时支持TFO</strong>，所以当tcp_fastopen的值为3时（比特为0x11）就表示完全支持TFO功能。</p><pre><code>net.ipv4.tcp_fastopen = 3
</code></pre><h2>小结</h2><p>这一讲，我们沿着三次握手的流程，介绍了Linux系统的优化方法。</p><p>当客户端通过发送SYN发起握手时，可以通过tcp_syn_retries控制重发次数。当服务器的SYN半连接队列溢出后，SYN报文会丢失从而导致连接建立失败。我们可以通过netstat -s给出的统计结果判断队列长度是否合适，进而通过tcp_max_syn_backlog参数调整队列的长度。服务器回复SYN+ACK报文的重试次数由tcp_synack_retries参数控制，网络稳定时可以调小它。为了应对SYN泛洪攻击，应将tcp_syncookies参数设置为1，它仅在SYN队列满后开启syncookie功能，保证连接成功建立。</p><p>服务器收到客户端返回的ACK后，会把连接移入accept队列，等待进程调用accept函数取出连接。如果accept队列溢出，默认系统会丢弃ACK，也可以通过tcp_abort_on_overflow参数用RST通知客户端连接建立失败。如果netstat统计信息显示，大量的ACK被丢弃后，可以通过listen函数的backlog参数和somaxconn系统参数提高队列上限。</p><p>TFO技术绕过三次握手，使得HTTP请求减少了1个RTT的时间。Linux下可以通过tcp_fastopen参数开启该功能。</p><p>从这一讲可以看出，虽然TCP是由操作系统实现的，但Linux通过多种方式提供了修改TCP功能的接口，供我们优化TCP的性能。下一讲我们再来探讨四次握手关闭连接时，Linux怎样帮助我们优化其性能。</p><h2>思考题</h2><p>最后，留给你一个思考题，关于三次握手建立连接，你做过哪些优化？效果如何？欢迎你在留言区与大家一起探讨。</p><p>感谢阅读，如果你觉得这节课对你有一些启发，也欢迎把它分享给你的朋友。</p>
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
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_1386e9</span>
  </div>
  <div class="_2_QraFYR_0">我在生产系统中确实遇到了accept队列溢出导致的请求超时问题，这种问题确实不容易被发现，服务器cpu、内存、连接数监控指标都正常，只是磁盘读写时间偶尔会偏高，当时为了解决这个问题，我搭建了一个测试环境，客户端用wrk发起请求，服务端用dd模拟磁盘大量读写导致io响应慢。并用blktrace+fio分析io，同时两边用tcpdump抓包，通过抓包数据发现服务端忽略了第三次握手客户端发来的ack（tcp_abort_on_overflow=0），又重发了第二次握手的syn+ack的包，响应超过了客户端的超时时间设置，google后才知道了是因为accept队列溢出的问题，结论就是我们使用的公有云虚拟机因为同宿主机上其他客户的虚拟机io大导致我们的虚拟机磁盘读写时间长，应用程序处理慢，没能及时从accept队列里拿连接处理，导致accept队列溢出，调大accept队列只能缓解溢出情况，提升程序处理速度才能根本解决，后面我们申请了独享的虚拟机，并且换成了共享ssd磁盘，并把io调度算法改成了noop，问题完美解决，性能杠杠滴</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢Geek_1386e9同学的实践分享！<br>一般可以通过netstat从丢包上检测出，大流量下在生产环境上抓包不易。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-04 20:54:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/82/3d/356fc3d6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>忆水寒</span>
  </div>
  <div class="_2_QraFYR_0">本文的干货满满的，虽然都熟悉，但是没有这么底层的优化过。<br>当网络丢包严重的时候，可以使用快速重传机制。重传时间间隔是指数级退避，直到达到 120s 为止，总时间将近 15 分钟，重传次数默认是 15次 ，重传次数默认值由 &#47;proc&#47;sys&#47;net&#47;ipv4&#47;tcp_retries2 决定。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢忆水寒的补充！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-20 22:30:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/63/2e/e49116d1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_007</span>
  </div>
  <div class="_2_QraFYR_0">看评论区，很多同学都说是长连接，普通的http keepalive 会不会有坑，三大运营商或者中间网络设备都会将超过一定时间的链接drop掉。如果没有h2这种ping保活的机制，有可能客户端莫名其妙长链接被drop掉，客户端只能依赖超时来感知异常，反倒是影响性能了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，不只网络设备，一些代理服务器为了减轻自己的负担，也会把长连接断掉，比如Nginx默认关闭75秒没有数据交互的keep alive 长连接</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-18 23:08:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/7f/99/6a92ff95.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Jesson</span>
  </div>
  <div class="_2_QraFYR_0">老师，TFO编程的时候，为什么要改为sendto或者sendmsg啊？ 这个地方原理不太懂。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你好Tango281，非TFO场景下，你大可先调用connect函数，在这里传递服务器地址，再调用send函数发送请求，但在TFO场景下，是不能调用connect函数的，否则就回归正常的三次握手了，所以要使用sendto等函数。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-22 08:50:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/51/29/24739c58.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>凉人。</span>
  </div>
  <div class="_2_QraFYR_0">做过优化，长连接， 减少time_wait时长，复用time_wait连接</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢凉人的分享</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-18 14:45:18</div>
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
  <div class="_2_QraFYR_0">看来要做一个能承受高并发的服务端，从前到后各个节点都要做关注和优化：syn半连接队列---&gt;accept队列----&gt;应用程序连接获取速度（epoll异步编程）---&gt;应用程序获取连接后的业务处理速度（非阻塞）。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-28 10:42:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/63/2e/e49116d1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_007</span>
  </div>
  <div class="_2_QraFYR_0">老师，今天的内容很多只适用于内网通信或者服务端单边优化吧。生产场景，客户端经常是手机或者PC.无法修改客户端内核参数。另外TFO适用于公网么？运营商或者移动端会不会对TFO不支持。(难道非得QUIC才能优化客户端的网络连接吗)😀</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-18 18:58:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/8f/cf/890f82d6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>那时刻</span>
  </div>
  <div class="_2_QraFYR_0">使用http的时候，为了减少tcp链接，重复使用已经链接的tcp，设置nginx的keepalive参数。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: http&#47;1.1里的keepalive功能是个很好用的优化点^_^</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-18 11:03:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/f0/a4/9d8c3dc3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>半生</span>
  </div>
  <div class="_2_QraFYR_0">老师，请教一个问题，都是减少tcp握手的代价，那tfo和长链接分别是什么使用场景，或者说，什么场景下，tfo会比长链接性价比更高？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果应用层允许使用长连接（比如应用层的超时可能有上限），或者空闲连接占用的内存资源不在乎（长连接不只在内核，在应用层在每连接也要消耗KB级别以上的内存），就使用长连接。<br>TFO的开启是有前提的，至少两端都要支持。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-01 10:34:46</div>
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
  <div class="_2_QraFYR_0">刚学习了两个知识点：<br>（1）syncookies功能<br>当SYN 半连接队列已满时，并非只能丢弃连接，服务端可以实时计算出一个状态值与SYN+ACK 报文一同发出，客户端返回报文时进行实时解析验证，如果通过也是可以建立连接的，减少了客户端不断重试的时间。<br>（2）TFO<br>首次建立连接之后，客户端与服务端分别存储加密的Cookie，解决后续的信任问题，这样后续就无需再经历同样的三次握手，减少建立连接的时间。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-18 09:14:53</div>
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
  <div class="_2_QraFYR_0">很实用的一篇文章。<br><br>今天又有新的收获，原来TFO是这么个意思，原理是这样的。<br>之前只在某些网络代理软件中看到过tcp_fastopen选项，但不知道啥意思。现在就明白了。<br>话说光软件层面开启了，内核未开启，也是享受不了的。这个就得看看内核是否默认开启了。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-18 07:23:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/5f/83/bb728e53.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Douglas</span>
  </div>
  <div class="_2_QraFYR_0">【accept队列溢出】这里讲的比较模糊，accept队列溢出。是把已经建立好的连接丢弃， 还是因为队列已满，所以，后续进来的连接直接丢弃呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 默认的行为是新进入的报文直接丢弃。这样，如果几秒后accept队列消费掉了，那么客户端超时重发的DATA+ACK到了，连接就会再次建立</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-19 16:07:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c9/75/62ce2d69.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>猿人谷</span>
  </div>
  <div class="_2_QraFYR_0">这篇文章太精彩了，赏心悦目啊</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢^_^</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-22 10:22:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/44/19/a2fa21af.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>爱谁谁</span>
  </div>
  <div class="_2_QraFYR_0">预建连接，连接复用，多IP竞速建立链接和复合连接~ </div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-20 00:14:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/85/0c/252a1149.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>军</span>
  </div>
  <div class="_2_QraFYR_0">感觉小林coding就是抄的这篇，太干货了，😁</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: ^_^</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-09 11:58:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4a/15/106eaaa8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>stackWarn</span>
  </div>
  <div class="_2_QraFYR_0">老师讲的优化在运维工作中都有做过，关于连接队列可以通过systemtap等工具去做下实验，我补充下自己曾经做过测试的结论，半连接队列大小的值和 tcp_max_syn_backlog  以及全连接队列的那两个参数即 somaxconn 和 应用层的backlog 这三个都有关系，感谢老师的讲解，不知道生产环境中tfo有没有什么问题呢？现在有哪些公司是否有开启TFO了，求科普</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-12 22:23:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/af/92/164e40f3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Raymond</span>
  </div>
  <div class="_2_QraFYR_0">老师 今天内容很有价值让我对tcp连接有了更深入的了解，此时我有些问题 1.在内容中只讲到了服务端在建立连接的时候 会维护 半连接队列，和accep队列，我想问在这个过程当中，客户端会维护队列吗？如果有的 是什么样结构和规范?第二个问题  就是确认下 我今天听课的一个 收获， 我的总结是 在tcp传输过程当中 无论是客户端和服务端 再给对方发送请求时 必须要有回应，如果没有回应就会通过重试机制来重发，重试一定次数后仍然没有回应 就确认失败， 而对方在处理请求异常后，比如服务端队列已满，无法建立连接，或者异常之后，默认情况下也不会告诉对方，通过对方的重试来感知失败，RST复位是个例外 </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你好Raymond，<br>1、客户端不需要队列，它并不需要监听某个端口再建立连接。<br>2、对的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-29 18:43:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/67/0e/c77ad9b1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>eason2017</span>
  </div>
  <div class="_2_QraFYR_0">文章精彩，佩付大师！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: ^_^</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-21 08:06:06</div>
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
  <div class="_2_QraFYR_0">访量很大时，time_wait 会非常多，一般都用长连接来规避这个问题。但是长连接也有问题，比如保活，比如当服务端挂了，但是因为网络隔离客户端还没感知道，这时请求就会有大量超时</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-20 23:11:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/88/d0/6e75f766.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>有朋自远方来</span>
  </div>
  <div class="_2_QraFYR_0">这个 cookie  和常说的 cookie session 是一回事么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不是哦，那个是HTTP协议中的概念</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-19 16:57:32</div>
  </div>
</div>
</div>
</li>
</ul>