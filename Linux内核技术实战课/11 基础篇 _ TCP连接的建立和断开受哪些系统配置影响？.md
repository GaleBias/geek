<audio title="11 基础篇 _ TCP连接的建立和断开受哪些系统配置影响？" src="https://static001.geekbang.org/resource/audio/13/28/1322c2010ab477dc83599f02e1885e28.mp3" controls="controls"></audio> 
<p>你好，我是邵亚方。</p><p>如果你做过Linux上面网络相关的开发，或者分析过Linux网络相关的问题，那你肯定吐槽过Linux系统里面让人眼花缭乱的各种配置项，应该也被下面这些问题困扰过：</p><ul>
<li>Client为什么无法和Server建立连接呢？</li>
<li>三次握手都完成了，为什么会收到Server的reset呢？</li>
<li>建立TCP连接怎么会消耗这么多时间？</li>
<li>系统中为什么会有这么多处于time-wait的连接？该这么处理？</li>
<li>系统中为什么会有这么多close-wait的连接？</li>
<li>针对我的业务场景，这么多的网络配置项，应该要怎么配置呢？</li>
<li>……</li>
</ul><p>因为网络这一块涉及到的场景太多了，Linux内核需要去处理各种各样的网络场景，不同网络场景的处理策略也会有所不同。而Linux内核的默认网络配置可能未必会适用我们的场景，这就可能导致我们的业务出现一些莫名其妙的行为。</p><p>所以，要想让业务行为符合预期，你需要了解Linux的相关网络配置，让这些配置更加适用于你的业务。Linux中的网络配置项是非常多的，为了让你更好地了解它们，我就以最常用的TCP/IP协议为例，从一个网络连接是如何建立起来的以及如何断开的来开始讲起。</p><h2>TCP连接的建立过程会受哪些配置项的影响？</h2><!-- [[[read_end]]] --><p><img src="https://static001.geekbang.org/resource/image/af/44/afc841ee3822fyye3ec186b28ee93744.jpg?wh=3478*2074" alt="" title="TCP建连过程"></p><p>上图就是一个TCP连接的建立过程。TCP连接的建立是一个从Client侧调用connect()，到Server侧accept()成功返回的过程。你可以看到，在整个TCP建立连接的过程中，各个行为都有配置选项来进行控制。</p><p>Client调用connect()后，Linux内核就开始进行三次握手。</p><p>首先Client会给Server发送一个SYN包，但是该SYN包可能会在传输过程中丢失，或者因为其他原因导致Server无法处理，此时Client这一侧就会触发超时重传机制。但是也不能一直重传下去，重传的次数也是有限制的，这就是tcp_syn_retries这个配置项来决定的。</p><p>假设tcp_syn_retries为3，那么SYN包重传的策略大致如下：</p><p><img src="https://static001.geekbang.org/resource/image/01/e4/012b9bf3e59f3abd5c5588a968e354e4.jpg?wh=2907*2066" alt="" title="tcp_syn_retries示意图"></p><p>在Client发出SYN后，如果过了1秒 ，还没有收到Server的响应，那么就会进行第一次重传；如果经过2s的时间还没有收到Server的响应，就会进行第二次重传；一直重传tcp_syn_retries次。</p><p>对于tcp_syn_retries为3而言，总共会重传3次，也就是说从第一次发出SYN包后，会一直等待（1 + 2 + 4 + 8）秒，如果还没有收到Server的响应，connect()就会产生ETIMEOUT的错误。</p><p>tcp_syn_retries的默认值是6，也就是说如果SYN一直发送失败，会在（1 + 2 + 4 + 8 + 16+ 32 + 64）秒，即127秒后产生ETIMEOUT的错误。</p><p>我们在生产环境上就遇到过这种情况，Server因为某些原因被下线，但是Client没有被通知到，所以Client的connect()被阻塞127s才去尝试连接一个新的Server， 这么长的超时等待时间对于应用程序而言是很难接受的。</p><p><strong>所以通常情况下，我们都会将数据中心内部服务器的tcp_syn_retries给调小，这里推荐设置为2，来减少阻塞的时间。</strong>因为对于数据中心而言，它的网络质量是很好的，如果得不到Server的响应，很可能是Server本身出了问题。在这种情况下，Client及早地去尝试连接其他的Server会是一个比较好的选择，所以对于客户端而言，一般都会做如下调整：</p><blockquote>
<p>net.ipv4.tcp_syn_retries = 2</p>
</blockquote><p>有些情况下1s的阻塞时间可能都很久，所以有的时候也会将三次握手的初始超时时间从默认值1s调整为一个较小的值，比如100ms，这样整体的阻塞时间就会小很多。这也是数据中心内部经常进行一些网络优化的原因。</p><p>如果Server没有响应Client的SYN，除了我们刚才提到的Server已经不存在了这种情况外，还有可能是因为Server太忙没有来得及响应，或者是Server已经积压了太多的半连接（incomplete）而无法及时去处理。</p><p>半连接，即收到了SYN后还没有回复SYNACK的连接，Server每收到一个新的SYN包，都会创建一个半连接，然后把该半连接加入到半连接队列（syn queue）中。syn queue的长度就是tcp_max_syn_backlog这个配置项来决定的，当系统中积压的半连接个数超过了该值后，新的SYN包就会被丢弃。对于服务器而言，可能瞬间会有非常多的新建连接，所以我们可以适当地调大该值，以免SYN包被丢弃而导致Client收不到SYNACK：</p><blockquote>
<p>net.ipv4.tcp_max_syn_backlog = 16384</p>
</blockquote><p><strong>Server中积压的半连接较多，也有可能是因为有些恶意的Client在进行SYN Flood攻击</strong>。典型的SYN Flood攻击如下：Client高频地向Server发SYN包，并且这个SYN包的源IP地址不停地变换，那么Server每次接收到一个新的SYN后，都会给它分配一个半连接，Server的SYNACK根据之前的SYN包找到的是错误的Client IP， 所以也就无法收到Client的ACK包，导致无法正确建立TCP连接，这就会让Server的半连接队列耗尽，无法响应正常的SYN包。</p><p>为了防止SYN Flood攻击，Linux内核引入了SYN Cookies机制。SYN Cookie的原理是什么样的呢？</p><p>在Server收到SYN包时，不去分配资源来保存Client的信息，而是根据这个SYN包计算出一个Cookie值，然后将Cookie记录到SYNACK包中发送出去。对于正常的连接，该Cookies值会随着Client的ACK报文被带回来。然后Server再根据这个Cookie检查这个ACK包的合法性，如果合法，才去创建新的TCP连接。通过这种处理，SYN Cookies可以防止部分SYN Flood攻击。所以对于Linux服务器而言，推荐开启SYN Cookies：</p><blockquote>
<p>net.ipv4.tcp_syncookies = 1</p>
</blockquote><p>Server向Client发送的SYNACK包也可能会被丢弃，或者因为某些原因而收不到Client的响应，这个时候Server也会重传SYNACK包。同样地，重传的次数也是由配置选项来控制的，该配置选项是tcp_synack_retries。</p><p>tcp_synack_retries的重传策略跟我们在前面讲的tcp_syn_retries是一致的，所以我们就不再画图来讲解它了。它在系统中默认是5，对于数据中心的服务器而言，通常都不需要这么大的值，推荐设置为2 :</p><blockquote>
<p>net.ipv4.tcp_synack_retries = 2</p>
</blockquote><p>Client在收到Server的SYNACK包后，就会发出ACK，Server收到该ACK后，三次握手就完成了，即产生了一个TCP全连接（complete），它会被添加到全连接队列（accept queue）中。然后Server就会调用accept()来完成TCP连接的建立。</p><p>但是，就像半连接队列（syn queue）的长度有限制一样，全连接队列（accept queue）的长度也有限制，目的就是为了防止Server不能及时调用accept()而浪费太多的系统资源。</p><p>全连接队列（accept queue）的长度是由listen(sockfd, backlog)这个函数里的backlog控制的，而该backlog的最大值则是somaxconn。somaxconn在5.4之前的内核中，默认都是128（5.4开始调整为了默认4096），建议将该值适当调大一些：</p><blockquote>
<p>net.core.somaxconn = 16384</p>
</blockquote><p>当服务器中积压的全连接个数超过该值后，新的全连接就会被丢弃掉。Server在将新连接丢弃时，有的时候需要发送reset来通知Client，这样Client就不会再次重试了。不过，默认行为是直接丢弃不去通知Client。至于是否需要给Client发送reset，是由tcp_abort_on_overflow这个配置项来控制的，该值默认为0，即不发送reset给Client。推荐也是将该值配置为0:</p><blockquote>
<p>net.ipv4.tcp_abort_on_overflow = 0</p>
</blockquote><p>这是因为，Server如果来不及accept()而导致全连接队列满，这往往是由瞬间有大量新建连接请求导致的，正常情况下Server很快就能恢复，然后Client再次重试后就可以建连成功了。也就是说，将 tcp_abort_on_overflow 配置为0，给了Client一个重试的机会。当然，你可以根据你的实际情况来决定是否要使能该选项。</p><p>accept()成功返回后，一个新的TCP连接就建立完成了，TCP连接进入到了ESTABLISHED状态：</p><p><img src="https://static001.geekbang.org/resource/image/e0/3c/e0ea3232fccf6bba8bace54d3f5d8d3c.jpg?wh=3000*2003" alt="" title="TCP状态转换"></p><p>上图就是从Client调用connect()，到Server侧accept()成功返回这一过程中的TCP状态转换。这些状态都可以通过netstat或者ss命令来看。至此，Client和Server两边就可以正常通信了。</p><p>接下来，我们看下TCP连接断开过程中会受哪些系统配置项的影响。</p><h2>TCP连接的断开过程会受哪些配置项的影响？</h2><p><img src="https://static001.geekbang.org/resource/image/1c/cf/1cf68d3eb4f07113ba13d84124f447cf.jpg?wh=3000*1929" alt="" title="TCP的四次挥手"></p><p>如上所示，当应用程序调用close()时，会向对端发送FIN包，然后会接收ACK；对端也会调用close()来发送FIN，然后本端也会向对端回ACK，这就是TCP的四次挥手过程。</p><p>首先调用close()的一侧是active close（主动关闭）；而接收到对端的FIN包后再调用close()来关闭的一侧，称之为passive close（被动关闭）。在四次挥手的过程中，有三个TCP状态需要额外关注，就是上图中深红色的那三个状态：主动关闭方的FIN_WAIT_2和TIME_WAIT，以及被动关闭方的CLOSE_WAIT状态。除了CLOSE_WAIT状态外，其余两个状态都有对应的系统配置项来控制。</p><p>我们首先来看FIN_WAIT_2状态，TCP进入到这个状态后，如果本端迟迟收不到对端的FIN包，那就会一直处于这个状态，于是就会一直消耗系统资源。Linux为了防止这种资源的开销，设置了这个状态的超时时间tcp_fin_timeout，默认为60s，超过这个时间后就会自动销毁该连接。</p><p>至于本端为何迟迟收不到对端的FIN包，通常情况下都是因为对端机器出了问题，或者是因为太繁忙而不能及时close()。所以，通常我们都建议将 tcp_fin_timeout 调小一些，以尽量避免这种状态下的资源开销。对于数据中心内部的机器而言，将它调整为2s足以：</p><blockquote>
<p>net.ipv4.tcp_fin_timeout = 2</p>
</blockquote><p>我们再来看TIME_WAIT状态，TIME_WAIT状态存在的意义是：最后发送的这个ACK包可能会被丢弃掉或者有延迟，这样对端就会再次发送FIN包。如果不维持TIME_WAIT这个状态，那么再次收到对端的FIN包后，本端就会回一个Reset包，这可能会产生一些异常。</p><p>所以维持TIME_WAIT状态一段时间，可以保障TCP连接正常断开。TIME_WAIT的默认存活时间在Linux上是60s（TCP_TIMEWAIT_LEN），这个时间对于数据中心而言可能还是有些长了，所以有的时候也会修改内核做些优化来减小该值，或者将该值设置为可通过sysctl来调节。</p><p>TIME_WAIT状态存在这么长时间，也是对系统资源的一个浪费，所以系统也有配置项来限制该状态的最大个数，该配置选项就是tcp_max_tw_buckets。对于数据中心而言，网络是相对很稳定的，基本不会存在FIN包的异常，所以建议将该值调小一些：</p><blockquote>
<p>net.ipv4.tcp_max_tw_buckets = 10000</p>
</blockquote><p>Client关闭跟Server的连接后，也有可能很快再次跟Server之间建立一个新的连接，而由于TCP端口最多只有65536个，如果不去复用处于TIME_WAIT状态的连接，就可能在快速重启应用程序时，出现端口被占用而无法创建新连接的情况。所以建议你打开复用TIME_WAIT的选项：</p><blockquote>
<p>net.ipv4.tcp_tw_reuse = 1</p>
</blockquote><p>还有另外一个选项tcp_tw_recycle来控制TIME_WAIT状态，但是该选项是很危险的，因为它可能会引起意料不到的问题，比如可能会引起NAT环境下的丢包问题。所以建议将该选项关闭：</p><blockquote>
<p>net.ipv4.tcp_tw_recycle = 0</p>
</blockquote><p>因为打开该选项后引起了太多的问题，所以新版本的内核就索性删掉了这个配置选项：<a href="https://git.kernel.org/pub/scm/linux/kernel/git/next/linux-next.git/commit/?id=4396e46187ca5070219b81773c4e65088dac50cc">tcp: remove tcp_tw_recycle.</a></p><p>对于CLOSE_WAIT状态而言，系统中没有对应的配置项。但是该状态也是一个危险信号，如果这个状态的TCP连接较多，那往往意味着应用程序有Bug，在某些条件下没有调用close()来关闭连接。我们在生产环境上就遇到过很多这类问题。所以，如果你的系统中存在很多CLOSE_WAIT状态的连接，那你最好去排查一下你的应用程序，看看哪里漏掉了close()。</p><p>至此，TCP四次挥手过程中需要注意的事项也讲完了。</p><p>好了，我们这节课就到此为止。</p><h2>课堂总结</h2><p>这节课我们讲了很多的配置项，我把这些配置项汇总到了下面这个表格里，方便你记忆：</p><p><img src="https://static001.geekbang.org/resource/image/3d/de/3d60be2523528f511dec0fbc88ce1ede.jpg?wh=3854*2416" alt=""></p><p>当然了，有些配置项也是可以根据你的服务器负载以及CPU和内存大小来做灵活配置的，比如tcp_max_syn_backlog、somaxconn、tcp_max_tw_buckets这三项，如果你的物理内存足够大、CPU核数足够多，你可以适当地增大这些值，这些往往都是一些经验值。</p><p>另外，我们这堂课的目的不仅仅是为了让你去了解这些配置项，最主要的是想让你了解其背后的机制，这样你在遇到一些问题时，就可以有一个大致的分析方向。</p><h2>课后作业</h2><p>课后请你使用tcpdump这个工具来观察下TCP的三次握手和四次挥手过程，巩固今天的学习内容。欢迎在留言区分享你的看法。</p><p>感谢你的阅读，如果你认为这节课的内容有收获，也欢迎把它分享给你的朋友，我们下一讲见。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/6e/05/d47cee18.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>wong ka seng</span>
  </div>
  <div class="_2_QraFYR_0">老师好，请问一下， 想听说tcp成功连接后很占资源，有没有具体的解说？例如每个连接消耗多少记忆体和cpu？谢谢！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: TCP连接的开销主要是内存开销，也就是TCP相关结构体的开销，每个TCP结构体的大小可以通过&#47;proc&#47;slabinfo来查看，例如：<br>cat &#47;proc&#47;slabinfo | grep -i TCP<br><br>这里面会有全连接和半连接TCP结构体的大小，全连接较大，一个有几K字节。<br><br>除了内存开销外，另外就是软中断的开销，tcp连接会用到很多timer，特别是time-wait连接，这些timer会消耗CPU。<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-05 23:31:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/20/8c/13/62e221c7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Abby</span>
  </div>
  <div class="_2_QraFYR_0">老师，如果不开启abort_on_overflow, 是不是这个时候client认为连接建立成功了，就会发送数据。server端发现这个连接没有建立，直接就再次发送reset回去咯？所以设置成1意义不大</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 并不是这样，在Server侧此时这个连接还是半连接，他会忽略client发的三次握手阶段的最后一个ack，而是继续给client发送synack，synack有次数的限制，Server给client发送的synack超过这个次数后才会断开这个连接。<br>如果是为1的话，Server就不会重传synack，而是直接发送Reset来断开连接。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-17 18:23:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/63/85/1dc41622.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>姑射仙人</span>
  </div>
  <div class="_2_QraFYR_0">“如果不维持 TIME_WAIT 这个状态，那么再次收到对端的 FIN 包后，本端就会回一个 Reset 包，这可能会产生一些异常。”  老师，我有两个问题：<br>1. 如果不维持TCP状态，最后一次ACK结束，Client端认为已经结束，就会关闭连接。这时候Server端在发送FIN包，客户端应该就收不到了吧？<br>2. 假设客户端收到了，回复RESET，会产生什么样的异常？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. 内核协议栈会收到该包。假设该fd已经被关闭，内核协议栈收到对端的fin包后，就查找不到对应的连接，它就会通知对端这是个异常的连接。<br><br>2. 假设客户端重新建立了一个连接，复用了之前的端口，由于网络延迟的原因，这个fin包可能在新连接建立后到达，那么这个新连接就会被误伤。所以一定要设立一个保护时间，来应对网络的不确定性。<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-10 10:45:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/20/81/02/59f5f168.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小白debug</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，文章里提到&quot;有些情况下 1s 的阻塞时间可能都很久，所以有的时候也会将三次握手的初始超时时间从默认值 1s 调整为一个较小的值，比如 100ms&quot; ，这里面怎么把第一次握手syn的初始超时时间从1s改成100ms？查了很多资料只看到介绍改超时次数tcp_syn_retries，而没有看到改默认超时时间。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这需要修改内核代码来实现，内核默认是无法更改这个时间的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-01 09:32:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/20/1d/0c1a184c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>罗辑思维</span>
  </div>
  <div class="_2_QraFYR_0">这个章节收获满满，老师把抽象的网络三次握手和四次挥手的过程，跟Linux网络参数配置对应起来，而且还根据场景给了具体的优化建议。<br><br>请问老师，能再推荐些Linux参数配置的优化建议学习资料么。网络上的资料要么只讲概念，没导入具体场景，或者是没确认基准线下进行优化，适用面很窄。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-19 02:35:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/ee/f0/abb7bfe3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>机制小风风</span>
  </div>
  <div class="_2_QraFYR_0">半连接，即收到了 SYN 后还没有回复 SYNACK 的连接，Server 每收到一个新的 SYN 包，都会创建一个半连接，然后把该半连接加入到半连接队列（syn queue）中。<br><br>开启 SYN Cookies 后，还会创建半连接吗？如果不创建是不是不用设置半连接队列了？<br>如果创建半连接，还会加入队列吗？如果加入，是收到ack,确认好cookies加入，还是什么时候加入？<br><br>不保存客户端信息，是不是指的不创建半连接？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 开启syn cookies后，服务端还是会创建半连接，但是该半连接只是为了回复syn-ack，回复完就会销毁，所以不会有资源浪费。再收到client的最后一个ack包后，服务端验证cookies的有效性后就会创建一个全连接，并把它添加到全连接队列中。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-18 01:23:20</div>
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
  <div class="_2_QraFYR_0">在一些文章中看到TIME_WAIT 和 FIN_WAIT2 状态的最大时长都是 2 MSL，由于在 Linux 系统中，MSL 的值固定为 30 秒，所以它们都是 60 秒。下面时我环境中<br>[13:47:18]root:workspace$ cat &#47;proc&#47;sys&#47;net&#47;ipv4&#47;tcp_fin_timeout<br>60<br>TIME_WAIT 状态存在这么长时间，也是对系统资源的一个浪费，所以系统也有配置项来限制该状态的最大个数，该配置选项就是 tcp_max_tw_buckets=10000，如果当前环境TIME_WAIT数量达到10000时，又是怎么处理的？<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 当time—wait的个数达到tw buckets的限制时 就直接释放掉 而不是再放在这个bucket中。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-09 13:56:26</div>
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
  <div class="_2_QraFYR_0">isn和paws老师可以细讲不，还有nat转换会转换seq吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 最普通的nat是不会的，它只是进行地址转换。不过nat可以工作在tcp层，可以解析tcp协议，所以有些nat也可以进行seq的转换。比如说有些协议地址是在tcp payload中，那这些内容都需要换掉。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-22 23:43:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://wx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTIt0nAFvqib3fpf9AIKUrEJMdbiaPjnKqCryevwjRdqrbzAIxdOn3P5wCz28MNb5Bgb2PwEdCezLEWg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>KennyQ</span>
  </div>
  <div class="_2_QraFYR_0">这块内容学习到了，可以运用到自己在生产中碰到的问题，吹牛可以吹很久！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-13 16:38:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/7e/05/431d380f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>拂尘</span>
  </div>
  <div class="_2_QraFYR_0">老师，有以下疑惑，希望能解答一下。<br>连接建立3次握手过程<br>1.client每次超时重试的超时时间，是采用指数级回退的，那linux有相应的配置参数，可以配置这个指数回退的策略吗？<br>2.根据回退策略和重试次数，可以计算出client总体connect的阻塞时间，linux有没有参数可以精细化或者单独配置这个总体阻塞超时时间呢？<br>连接断开4次挥手过程<br>3.连接断开是分主被动方的。除了client会作为主动方断开连接，server也可以作为主动方断开连接吗？如果是，在这种情况下，client和server双方，他们整体的时间轴状态转换图又是如何的呢？（因为基本上搜索到的资料，都是只描述了client作为主动方发起关闭的流程）<br>4.如果server的ack和fin是打包一起发给client的话，client会有fin_wait_2状态吗？<br>5.您提到选项tcp_tw_recycle用来控制TIME_WAIT状态，而且很危险。能否详细描述一下此选项配置的意义，会影响time_wait的什么行为？以及具体的危害内容？<br>6.您提到，如果在server中CLOSE_WAIT状态的连接很多，基本断定是应用程序里漏掉了close调用。这里的close调用是指client端的吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-07 07:04:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>地下城勇士</span>
  </div>
  <div class="_2_QraFYR_0">看了好多篇，老师的图都很到为，请问一下老师的图是用什么工具画的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: gliffy diagrams，chrome的插件。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-17 11:39:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKT9Tk01eiaQ9aAhszthzGm6lwruRWPXia1YYFozctrdRvKg0Usp8NbwuKBApwD0D6Fty2tib3RdtFJg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>欧阳洲</span>
  </div>
  <div class="_2_QraFYR_0">老师好：<br>FIN_WAIT_2 超时时间是 tcp_fin_timeout 控制，<br>TIME_WAIT 默认也是 60s，但是 &#47;proc&#47;sys&#47;net&#47;ipv4&#47; 下没有wait相关文件名，<br>TIME_WAIT 是与 FIN_WAIT_2 共用了同一个选项吗?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 默认内核里TIME-WAIT时间是不可修改的，也就是没有对应的sysctl选项。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-12 16:31:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/mfZN0Rvvg3fhxtX3vglErbp2rwlwviccS3LXTRXZAdsB88UEwtuCurrWhjlgcHYicQcXoukbbRKWKRAFQNZh9RBQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>打工人</span>
  </div>
  <div class="_2_QraFYR_0">老师好，我学习这门课比较晚了，不过最近测试发现不管是somax还是syn_backlog都设置成多少，模拟syn flood攻击时，netstat里查看syn_recv状态的连接都是256，求解</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-16 11:26:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/63/85/1dc41622.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>姑射仙人</span>
  </div>
  <div class="_2_QraFYR_0">图中的client和server不是确定的吧？还是要看是谁发起的关闭操作。也就是说客户端和服务端都会存在time wait状态的连接，对吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-08 20:28:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/9c/85/d9614715.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Felix</span>
  </div>
  <div class="_2_QraFYR_0">收到SYN还没有回复SYNACK的是半连接，那回复了SYNACK还没有收到ACK的连接算不算是半连接，这也是一个中间状态，是否也在syn queue里面，即syn queue只包含第一次握手还是只包含第二次握手还是第一次第二次都包含</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: client发送syn，sever接收syn然后回复synack，client收到synack回复ack，至此三次握手结束。<br>半连接&#47;全链接针对的是被被动连接方（Server），目的是为了安全以及可靠性，对于主动连接方（client）而言，是无需考虑这些事的，所以不存在全连接&#47;半连接这个设计。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-02 14:54:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/c4/35/2cc10d43.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Wade_阿伟</span>
  </div>
  <div class="_2_QraFYR_0">这节课干活满满，熟悉了tcp的三次握手和四次挥手的过程，以及可以通过哪些配置来调整优化上述过程。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-01 15:02:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>威威</span>
  </div>
  <div class="_2_QraFYR_0">看了第三遍 最近遇到了一个问题 0.0.0.0 dns域名反解ptr，想要快速定位哪些进程发送了此udp服务，通过tcpdump可以看到这些流，得到原端口，但通过lsof查看端口对应进程查不到，猜测是实效性问题，好啥好办法</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-17 12:04:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/0c/1e/74d65100.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jack motor</span>
  </div>
  <div class="_2_QraFYR_0">TIME_WAIT 状态持续时间是60s，修改这个是不是必须要重新编译内核？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的 需要修改内核。有些不需要修改内核的方法，比如使用netfiter，但是并不很通用。通用一些的方法还是要修改内核。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-27 09:49:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/9d/13/a8b614b8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小孩不乖</span>
  </div>
  <div class="_2_QraFYR_0">学到了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-19 19:57:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/fb/7f/746a6f5e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Q</span>
  </div>
  <div class="_2_QraFYR_0">第一次看到把内核网络参数和TCP-IP建连和断连图结合起来的讲！！之前看陶辉老师的Nginx也有讲过，但只是文字版，理解起来有点费劲。这期课图文并茂，更容易理解了！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-14 07:06:13</div>
  </div>
</div>
</div>
</li>
</ul>