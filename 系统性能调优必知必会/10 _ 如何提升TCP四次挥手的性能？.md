<audio title="10 _ 如何提升TCP四次挥手的性能？" src="https://static001.geekbang.org/resource/audio/0f/0e/0fd9ccef046f64c2131adb2e84135b0e.mp3" controls="controls"></audio> 
<p>你好，我是陶辉。</p><p>上一节课，我们介绍了建立连接时的优化方法，这一节课再来看四次挥手关闭连接时，如何优化性能。</p><p>close和shutdown函数都可以关闭连接，但这两种方式关闭的连接，不只功能上有差异，控制它们的Linux参数也不相同。close函数会让连接变为孤儿连接，shutdown函数则允许在半关闭的连接上长时间传输数据。TCP之所以具备这个功能，是因为它是全双工协议，但这也造成四次挥手非常复杂。</p><p>四次挥手中你可以用netstat命令观察到6种状态。其中，你多半看到过TIME_WAIT状态。网上有许多文章介绍怎样减少TIME_WAIT状态连接的数量，也有文章说TIME_WAIT状态是必不可少、不能优化掉的。这两种看似自相矛盾的观点之所以存在，就在于优化连接关闭时，不能仅基于主机端的视角，还必须站在整个网络的层次上，才能给出正确的解决方案。</p><p>Linux为四次挥手提供了很多控制参数，有些参数的名称与含义并不相符。例如tcp_orphan_retries参数中有orphan孤儿，却同时对非孤儿连接也生效。而且，错误地配置这些参数，不只无法针对高并发场景提升性能，还会降低资源的使用效率，甚至引发数据错误。</p><!-- [[[read_end]]] --><p>这一讲，我们将基于四次挥手的流程，介绍Linux下的优化方法。</p><h2>四次挥手的流程</h2><p>你想没想过，为什么建立连接是三次握手，而关闭连接需要四次挥手呢？</p><p>这是因为TCP不允许连接处于半打开状态时就单向传输数据，所以在三次握手建立连接时，服务器会把ACK和SYN放在一起发给客户端，其中，ACK用来打开客户端的发送通道，SYN用来打开服务器的发送通道。这样，原本的四次握手就降为三次握手了。</p><p><img src="https://static001.geekbang.org/resource/image/74/51/74ac4e70ef719f19270c08201fb53a51.png?wh=943*613" alt=""></p><p>但是当连接处于半关闭状态时，TCP是允许单向传输数据的。为便于下文描述，<strong>接下来我们把先关闭连接的一方叫做主动方，后关闭连接的一方叫做被动方。</strong>当主动方关闭连接时，被动方仍然可以在不调用close函数的状态下，长时间发送数据，此时连接处于半关闭状态。这一特性是TCP的双向通道互相独立所致，却也使得关闭连接必须通过四次挥手才能做到。</p><p><strong>互联网中往往服务器才是主动关闭连接的一方。</strong>这是因为，HTTP消息是单向传输协议，服务器接收完请求才能生成响应，发送完响应后就会立刻关闭TCP连接，这样及时释放了资源，能够为更多的用户服务。</p><p>这就使得服务器的优化策略变得复杂起来。一方面，由于被动方有多种应对策略，从而增加了主动方的处理分支。另一方面，服务器同时为成千上万个用户服务，任何错误都会被庞大的用户数放大。所以对主动方的关闭连接参数调整时，需要格外小心。</p><p>了解了这一点之后，我们再来看四次挥手的流程。</p><p><img src="https://static001.geekbang.org/resource/image/e2/b7/e2ef1347b3b4590da431dc236d9239b7.png?wh=1162*825" alt=""></p><p><strong>其实四次挥手只涉及两种报文：FIN和ACK。</strong>FIN就是Finish结束连接的意思，谁发出FIN报文，就表示它将不再发送任何数据，关闭这一方向的传输通道。ACK是Acknowledge确认的意思，它用来通知对方：你方的发送通道已经关闭。</p><p>当主动方关闭连接时，会发送FIN报文，此时主动方的连接状态由ESTABLISHED变为FIN_WAIT1。当被动方收到FIN报文后，内核自动回复ACK报文，连接状态由ESTABLISHED变为CLOSE_WAIT，顾名思义，它在等待进程调用close函数关闭连接。当主动方接收到这个ACK报文后，连接状态由FIN_WAIT1变为FIN_WAIT2，主动方的发送通道就关闭了。</p><p>再来看被动方的发送通道是如何关闭的。当被动方进入CLOSE_WAIT状态时，进程的read函数会返回0，这样开发人员就会有针对性地调用close函数，进而触发内核发送FIN报文，此时被动方连接的状态变为LAST_ACK。当主动方收到这个FIN报文时，内核会自动回复ACK，同时连接的状态由FIN_WAIT2变为TIME_WAIT，Linux系统下大约1分钟后TIME_WAIT状态的连接才会彻底关闭。而被动方收到ACK报文后，连接就会关闭。</p><h2>主动方的优化</h2><p>关闭连接有多种方式，比如进程异常退出时，针对它打开的连接，内核就会发送RST报文来关闭。RST的全称是Reset复位的意思，它可以不走四次挥手强行关闭连接，但当报文延迟或者重复传输时，这种方式会导致数据错乱，所以这是不得已而为之的关闭连接方案。</p><p>安全关闭连接的方式必须通过四次挥手，它由进程调用close或者shutdown函数发起，这二者都会向对方发送FIN报文（shutdown参数须传入SHUT_WR或者SHUT_RDWR才会发送FIN），区别在于close调用后，哪怕对方在半关闭状态下发送的数据到达主动方，进程也无法接收。</p><p><strong>此时，这个连接叫做孤儿连接，如果你用netstat -p命令，会发现连接对应的进程名为空。而shutdown函数调用后，即使连接进入了FIN_WAIT1或者FIN_WAIT2状态，它也不是孤儿连接，进程仍然可以继续接收数据。</strong>关于孤儿连接的概念，下文调优参数时还会用到。</p><p>主动方发送FIN报文后，连接就处于FIN_WAIT1状态下，该状态通常应在数十毫秒内转为FIN_WAIT2。只有迟迟收不到对方返回的ACK时，才能用netstat命令观察到FIN_WAIT1状态。此时，<strong>内核会定时重发FIN报文，其中重发次数由tcp_orphan_retries参数控制</strong>（注意，orphan虽然是孤儿的意思，该参数却不只对孤儿连接有效，事实上，它对所有FIN_WAIT1状态下的连接都有效），默认值是0，特指8次：</p><pre><code>net.ipv4.tcp_orphan_retries = 0
</code></pre><p>如果FIN_WAIT1状态连接有很多，你就需要考虑降低tcp_orphan_retries的值。当重试次数达到tcp_orphan_retries时，连接就会直接关闭掉。</p><p><strong>对于正常情况来说，调低tcp_orphan_retries已经够用，但如果遇到恶意攻击，FIN报文根本无法发送出去。</strong>这是由TCP的2个特性导致的。</p><ul>
<li>首先，TCP必须保证报文是有序发送的，FIN报文也不例外，当发送缓冲区还有数据没发送时，FIN报文也不能提前发送。</li>
<li>其次，TCP有流控功能，当接收方将接收窗口设为0时，发送方就不能再发送数据。所以，当攻击者下载大文件时，就可以通过将接收窗口设为0，导致FIN报文无法发送，进而导致连接一直处于FIN_WAIT1状态。</li>
</ul><p>解决这种问题的方案是调整tcp_max_orphans参数：</p><pre><code>net.ipv4.tcp_max_orphans = 16384
</code></pre><p>顾名思义，<strong>tcp_max_orphans 定义了孤儿连接的最大数量。</strong>当进程调用close函数关闭连接后，无论该连接是在FIN_WAIT1状态，还是确实关闭了，这个连接都与该进程无关了，它变成了孤儿连接。Linux系统为防止孤儿连接过多，导致系统资源长期被占用，就提供了tcp_max_orphans参数。如果孤儿连接数量大于它，新增的孤儿连接将不再走四次挥手，而是直接发送RST复位报文强制关闭。</p><p>当连接收到ACK进入FIN_WAIT2状态后，就表示主动方的发送通道已经关闭，接下来将等待对方发送FIN报文，关闭对方的发送通道。这时，<strong>如果连接是用shutdown函数关闭的，连接可以一直处于FIN_WAIT2状态。但对于close函数关闭的孤儿连接，这个状态不可以持续太久，而tcp_fin_timeout控制了这个状态下连接的持续时长。</strong></p><pre><code>net.ipv4.tcp_fin_timeout = 60
</code></pre><p>它的默认值是60秒。这意味着对于孤儿连接，如果60秒后还没有收到FIN报文，连接就会直接关闭。这个60秒并不是拍脑袋决定的，它与接下来介绍的TIME_WAIT状态的持续时间是相同的，我们稍后再来回答60秒的由来。</p><p>TIME_WAIT是主动方四次挥手的最后一个状态。当收到被动方发来的FIN报文时，主动方回复ACK，表示确认对方的发送通道已经关闭，连接随之进入TIME_WAIT状态，等待60秒后关闭，为什么呢？我们必须站在整个网络的角度上，才能回答这个问题。</p><p>TIME_WAIT状态的连接，在主动方看来确实已经关闭了。然而，被动方没有收到ACK报文前，连接还处于LAST_ACK状态。如果这个ACK报文没有到达被动方，被动方就会重发FIN报文。重发次数仍然由前面介绍过的tcp_orphan_retries参数控制。</p><p>如果主动方不保留TIME_WAIT状态，会发生什么呢？此时连接的端口恢复了自由身，可以复用于新连接了。然而，被动方的FIN报文可能再次到达，这既可能是网络中的路由器重复发送，也有可能是被动方没收到ACK时基于tcp_orphan_retries参数重发。这样，<strong>正常通讯的新连接就可能被重复发送的FIN报文误关闭。</strong>保留TIME_WAIT状态，就可以应付重发的FIN报文，当然，其他数据报文也有可能重发，所以TIME_WAIT状态还能避免数据错乱。</p><p>我们再回过头来看看，为什么TIME_WAIT状态要保持60秒呢？这与孤儿连接FIN_WAIT2状态默认保留60秒的原理是一样的，<strong>因为这两个状态都需要保持2MSL时长。MSL全称是Maximum Segment Lifetime，它定义了一个报文在网络中的最长生存时间</strong>（报文每经过一次路由器的转发，IP头部的TTL字段就会减1，减到0时报文就被丢弃，这就限制了报文的最长存活时间）。</p><p>为什么是2 MSL的时长呢？这其实是相当于至少允许报文丢失一次。比如，若ACK在一个MSL内丢失，这样被动方重发的FIN会在第2个MSL内到达，TIME_WAIT状态的连接可以应对。为什么不是4或者8 MSL的时长呢？你可以想象一个丢包率达到百分之一的糟糕网络，连续两次丢包的概率只有万分之一，这个概率实在是太小了，忽略它比解决它更具性价比。</p><p><strong>因此，TIME_WAIT和FIN_WAIT2状态的最大时长都是2 MSL，由于在Linux系统中，MSL的值固定为30秒，所以它们都是60秒。</strong></p><p>虽然TIME_WAIT状态的存在是有必要的，但它毕竟在消耗系统资源，比如TIME_WAIT状态的端口就无法供新连接使用。怎样解决这个问题呢？</p><p><strong>Linux提供了tcp_max_tw_buckets 参数，当TIME_WAIT的连接数量超过该参数时，新关闭的连接就不再经历TIME_WAIT而直接关闭。</strong></p><pre><code>net.ipv4.tcp_max_tw_buckets = 5000
</code></pre><p>当服务器的并发连接增多时，相应地，同时处于TIME_WAIT状态的连接数量也会变多，此时就应当调大tcp_max_tw_buckets参数，减少不同连接间数据错乱的概率。</p><p>当然，tcp_max_tw_buckets也不是越大越好，毕竟内存和端口号都是有限的。有没有办法让新连接复用TIME_WAIT状态的端口呢？如果服务器会主动向上游服务器发起连接的话，就可以把tcp_tw_reuse参数设置为1，它允许作为客户端的新连接，在安全条件下使用TIME_WAIT状态下的端口。</p><pre><code>net.ipv4.tcp_tw_reuse = 1
</code></pre><p>当然，要想使tcp_tw_reuse生效，还得把timestamps参数设置为1，它满足安全复用的先决条件（对方也要打开tcp_timestamps ）：</p><pre><code>net.ipv4.tcp_timestamps = 1
</code></pre><p>老版本的Linux还提供了tcp_tw_recycle参数，它并不要求TIME_WAIT状态存在60秒，很容易导致数据错乱，不建议设置为1。</p><pre><code>net.ipv4.tcp_tw_recycle = 0
</code></pre><p>所以在Linux 4.12版本后，直接取消了这一参数。</p><h2>被动方的优化</h2><p>当被动方收到FIN报文时，就开启了被动方的四次挥手流程。内核自动回复ACK报文后，连接就进入CLOSE_WAIT状态，顾名思义，它表示等待进程调用close函数关闭连接。</p><p>内核没有权力替代进程去关闭连接，因为若主动方是通过shutdown关闭连接，那么它就是想在半关闭连接上接收数据。<strong>因此，Linux并没有限制CLOSE_WAIT状态的持续时间。</strong></p><p>当然，大多数应用程序并不使用shutdown函数关闭连接，所以，当你用netstat命令发现大量CLOSE_WAIT状态时，要么是程序出现了Bug，read函数返回0时忘记调用close函数关闭连接，要么就是程序负载太高，close函数所在的回调函数被延迟执行了。此时，我们应当在应用代码层面解决问题。</p><p>由于CLOSE_WAIT状态下，连接已经处于半关闭状态，所以此时进程若要关闭连接，只能调用close函数（再调用shutdown关闭单向通道就没有意义了），内核就会发出FIN报文关闭发送通道，同时连接进入LAST_ACK状态，等待主动方返回ACK来确认连接关闭。</p><p>如果迟迟等不到ACK，内核就会重发FIN报文，重发次数仍然由tcp_orphan_retries参数控制，这与主动方重发FIN报文的优化策略一致。</p><p>至此，由一方主动发起四次挥手的流程就介绍完了。需要你注意的是，<strong>如果被动方迅速调用close函数，那么被动方的ACK和FIN有可能在一个报文中发送，这样看起来，四次挥手会变成三次挥手，这只是一种特殊情况，不用在意。</strong></p><p>我们再来看一种特例，如果连接双方同时关闭连接，会怎么样？</p><p>此时，上面介绍过的优化策略仍然适用。两方发送FIN报文时，都认为自己是主动方，所以都进入了FIN_WAIT1状态，FIN报文的重发次数仍由tcp_orphan_retries参数控制。</p><p><img src="https://static001.geekbang.org/resource/image/04/52/043752a3957d36f4e3c82cd83d472452.png?wh=1165*585" alt=""></p><p>接下来，双方在等待ACK报文的过程中，都等来了FIN报文。这是一种新情况，所以连接会进入一种叫做CLOSING的新状态，它替代了FIN_WAIT2状态。此时，内核回复ACK确认对方发送通道的关闭，仅己方的FIN报文对应的ACK还没有收到。所以，CLOSING状态与LAST_ACK状态下的连接很相似，它会在适时重发FIN报文的情况下最终关闭。</p><h2>小结</h2><p>我们对这一讲的内容做个小结。</p><p>今天我们讲了四次挥手的流程，你需要根据主动方与被动方的连接状态变化来调整系统参数，使它在特定网络条件下更及时地释放资源。</p><p>四次挥手的主动方，为了应对丢包，允许在tcp_orphan_retries次数内重发FIN报文。当收到ACK报文，连接就进入了FIN_WAIT2状态，此时系统的行为依赖这是否为孤儿连接。</p><p>如果这是close函数关闭的孤儿连接，那么在tcp_fin_timeout秒内没有收到对方的FIN报文，连接就直接关闭，反之shutdown函数关闭的连接则不受此限制。毕竟孤儿连接可能在重发次数内存在数分钟之久，为了应对孤儿连接占用太多的资源，tcp_max_orphans定义了最大孤儿连接的数量，超过时连接就会直接释放。</p><p>当接收到FIN报文，并返回ACK后，主动方的连接进入TIME_WAIT状态。这一状态会持续1分钟，为了防止TIME_WAIT状态占用太多的资源，tcp_max_tw_buckets定义了最大数量，超过时连接也会直接释放。当TIME_WAIT状态过多时，还可以通过设置tcp_tw_reuse和tcp_timestamps为1 ，将TIME_WAIT状态的端口复用于作为客户端的新连接。</p><p>被动关闭的连接方应对非常简单，它在回复ACK后就进入了CLOSE_WAIT状态，等待进程调用close函数关闭连接。因此，出现大量CLOSE_WAIT状态的连接时，应当从应用程序中找问题。当被动方发送FIN报文后，连接就进入LAST_ACK状态，在未等来ACK时，会在tcp_orphan_retries参数的控制下重发FIN报文。</p><p>至此，TCP连接建立、关闭时的性能优化就介绍完了。下一讲，我们将专注在TCP上传输数据时，如何优化内存的使用效率。</p><h2>思考题</h2><p>最后，给你留一个思考题。你知道关闭连接时的SO_LINGER选项吗？它希望用四次挥手替代RST关闭连接的方式，防止浏览器没有接收到完整的HTTP响应。请你思考一下，SO_LINGER会怎么影响主动方连接的状态变化？SO_LINGER上的超时时间，是怎样与系统配置参数协作的？欢迎你在留言区与我一起探讨。</p><p>感谢阅读，如果你觉得这节课对你有一些启发，也欢迎把它分享给你的朋友。</p>
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
  <div class="_2_QraFYR_0">本节内容真的很烧脑啊！收获也很多，从来没有从这么多角度看待过这个问题。<br>so_linger是一个结构体，其中有两个参数：l_onoff和l_linger。第一个参数表示是否启用so_linger功能，第二个参数l_linger=0，则 close 函数在阻塞直到 l_linger 时间超时或者数据发送完毕，如果超时则直接情况缓冲区然后RST连接（默认情况下是调用close函数立即返回）。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢忆水寒的分享！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-21 23:38:57</div>
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
  <div class="_2_QraFYR_0">老师的文章都要细品.<br>内核调优的参数算是见识到了,不知道啥时候才能用得上.🤦<br><br>一些边界情况老师也介绍到了:<br>1. tcp_tw_reuse 也是有适用条件的.<br>2. 四次挥手在极端情况下,可能变三次挥手.<br>3. CLOSING 状态确实算是第一次见,之前没考虑过这个问题.<br><br>以前在用c语言的网络编程时,会接触setsockopt函数.<br>最初的使用该函数的原因,只是为了端口复用.<br>因为在调试的过程中,经常会重启服务,如果不设置SO_REUSEADDR参数,就需要等系统释放掉端口,服务才能监听成功.<br><br>见过SO_LINGER选项,但真未细究过.<br>待课代表们来解答.<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Nginx中有SO_LINGER选项的应用，可以参见《Nginx核心知识100讲》第130课</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-20 10:32:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/a9/36/d054c979.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>G.S.K</span>
  </div>
  <div class="_2_QraFYR_0">请教老师，我有一点想不明白。假如A和B之间有个tcp连接，A主动发起关闭，A进入TIME_WAIT状态。如果B端的RTO(重传超时时间)比2msl大，2MSL后，B端还是有可能重传FIN的。感觉TIME_WAIT等待的时间应该是B端RTO+MSL啊。还请老师帮忙解惑。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: RTO通常是毫秒级，2MSL是分钟级，RTO大于2MSL的概率几乎为0</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-01 08:15:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/03/1c/c9fe6738.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Kvicii.Y</span>
  </div>
  <div class="_2_QraFYR_0">TCP的这几篇文章我听了有10几遍</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-07 20:49:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_78d3bb</span>
  </div>
  <div class="_2_QraFYR_0">为什么3次握手4次挥手？ 因为挥手的时候，需要先清空各自缓冲区中的数据，然后才能close，不是这样吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 清空缓冲区，是关闭一端传输能力的子任务。4次挥手的主要原因，是因为TCP由内核实现，但应用层可以半关闭连接且长期存在，所以只能是4次挥手，清空缓冲区只是子目标。个人看法^_^</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-21 21:36:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/47/b0/a9b77a1e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>冬风向左吹</span>
  </div>
  <div class="_2_QraFYR_0">看到一遍文章，tcp_timestamps还有这个坑吗？<br>https:&#47;&#47;blog.51cto.com&#47;fuyuan2016&#47;1795998</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你好东郭，我认为本质上这是打开tcp_tw_recycle造成的，真不关timestamps的事。timestamps的好处很多，不建议关掉。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-21 21:02:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/57/6e/b6795c44.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>夜空中最亮的星</span>
  </div>
  <div class="_2_QraFYR_0">至少要听3遍</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 点个赞！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-20 08:27:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/e4/e5/82132920.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>亦知码蚁</span>
  </div>
  <div class="_2_QraFYR_0"> tcp_tw_reuse 的内核配置选项和 SO_REUSEADDR 套接字选择，这两个有什么区别呢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-06 02:06:43</div>
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
  <div class="_2_QraFYR_0">当 发送 FIN 报文后，长时间没有收到 ACK 报文就会重发 FIN, 这个 长时间是哪个参数进行控制的？ 就是多长时间 没有收到 ACK 才会重发。 这的话就可以在结合 tcp_orphan_retries 这个参数可以算出来一个会消耗多少时间</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-04 09:22:51</div>
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
  <div class="_2_QraFYR_0">TTL每一跳减少1，这些怎么和MSL对应起来呢，每一跳减少的1相当于1秒？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不是，这是一个预估值，所谓每一跳，是指每经过一个路由器网络设备，将IP头部中的TTL字段减少1，并不等于1秒，通常推荐的TTL的初始值是64</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-20 07:49:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/h0KAdRFKjCOSLRjzictvlaIpALTISCOftCIat5ej7QZlndkxTzhKmkvHWqMjvSKcuPZ0bmYkrWiaRXxja3hibwvjHgcLBvA9Uel/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_f1c894</span>
  </div>
  <div class="_2_QraFYR_0">老师，您好！我在想主动方处于FIN_WAIT2状态，被动方处于CLOSE_WAIT，意味可能想在半关闭连接上接收数据，且没有限制CLOSE_WAIT状态的持续时间，若被动方在CLOSE_WAIT状态还要发送的数据量耗时超过了2MSL时间，此时主动方是如何处理的？<br>我猜想可能情况①主动方接收了个含数据的报文后重新记时（收到最新报文的时间开始记时2MSL时间），直到在2MSL内收到FIN包或没接收到数据及FIN包做超时结束连接。②主动方在进入FIN_WAIT2状态的一分钟（2MSL）内能收到多少数据就收到，时间一到直接进入TIME_WAIT状态或者直接关闭连接</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这取决于主动方调用的是close还是shutdown：<br>1、调用的是close，此时连接将成为orphan，一般在 tcp_fin_timeout秒左右连接会被系统关闭；<br>2、调用的是shutdown，此时只有TIME_WAIT才有2MSL限制，FIN_WAIT2可以长时间处于此状态。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-04 01:25:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1c/f4/f7/871ff71d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_David</span>
  </div>
  <div class="_2_QraFYR_0">一篇15分钟的音频，足足花了1.5个小时<br>包括先听音频，再看文字，再理解每个含义，再记笔记<br>我敢说我懂了。<br>前提是对TCP想过比较了解的情况下，太南了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-29 23:37:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/dc/19/c058bcbf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>流浪地球</span>
  </div>
  <div class="_2_QraFYR_0">MSL的数值在所有网络环境中都一样，如果一些跨洋的长距离网络这个时间可以保证吗？如果发现这个数值无法保证，是需要调整网络传输层次结构吗？<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-20 16:55:48</div>
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
  <div class="_2_QraFYR_0">rst报文的发送是比普通数据优先级高吗，也就是socket对端如果先接受到rst，这个socket就不可读不可写了，后面收到的数据自然也没法被进程拿到了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，普通数据要有序接收，但RST不需要按seq序号处理</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-20 08:15:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1c/dc/6d/03130ea9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>JW</span>
  </div>
  <div class="_2_QraFYR_0">不同系统的TTL不一样，30秒是比较少的</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-20 07:23:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/ae/8b/43ce01ca.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ezekiel</span>
  </div>
  <div class="_2_QraFYR_0">如果主动方不保留 TIME_WAIT 状态，会发生什么呢？此时连接的端口恢复了自由身，可以复用于新连接了。然而，被动方的 FIN 报文可能再次到达，这既可能是网络中的路由器重复发送，也有可能是被动方没收到 ACK 时基于 tcp_orphan_retries 参数重发。这样，正常通讯的新连接就可能被重复发送的 FIN 报文误关闭。<br>-----------------------------------------------------------------------------------------------------<br>报文不都是有编号的吗？重新启用的端口，会处理过期的报文吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-24 08:08:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKsUJP9VV8iaZOuQxfRgzUvxqA09EjlCHfdDGSof9ibXWxpPjl6UpjEsWtTTbiay0RrKjrp814wnibkkQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Foxyier</span>
  </div>
  <div class="_2_QraFYR_0">有一个地方没看懂， 文中先说了「互联网中往往服务器才是主动关闭连接的一方」，后续的内容中又把客户端称为主动方，服务端称为被动方。 所以有一些混乱， 四次挥手的过程中到底是谁先关闭的连接。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-15 18:21:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/28/2d/cd/f4a16584.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>神经蛙</span>
  </div>
  <div class="_2_QraFYR_0">老师好，tcp_tw_reuse 配合 tcp_timestamps 可以作为客户端来复用TIME_WAIT的端口。那么作为服务端，应该怎么来减少TIME_WAIT的状态呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-28 11:09:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/22/da/cb/02bf0f14.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>liutebin</span>
  </div>
  <div class="_2_QraFYR_0">这一节干货满满！光看这些知识，却很少在工作中运用。老师有没有好的办法，让我们自己可以模拟这些问题，并调优实践。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-15 18:14:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/rRCSdTPyqWcW6U8DO9xL55ictNPlbQ38VAcaBNgibqaAhcH7mn1W9ddxIJLlMiaA5sngBicMX02w2HP5pAWpBAJsag/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>butterfly</span>
  </div>
  <div class="_2_QraFYR_0">在主动方处于 TIME_WAIT状态时，如果过了2MSL时间，还是有FIN传来，那不是对新连接还是有影响吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-19 18:45:40</div>
  </div>
</div>
</div>
</li>
</ul>