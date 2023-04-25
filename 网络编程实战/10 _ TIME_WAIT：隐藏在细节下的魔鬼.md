<audio title="10 _ TIME_WAIT：隐藏在细节下的魔鬼" src="https://static001.geekbang.org/resource/audio/0c/49/0cd463a790c13c3414b50d77ebcfa249.mp3" controls="controls"></audio> 
<p>你好，我是盛延敏，这是网络编程实战的第10讲，欢迎回来。</p><p>在前面的基础篇里，我们对网络编程涉及到的基础知识进行了梳理，主要内容包括C/S编程模型、TCP协议、UDP协议和本地套接字等内容。在提高篇里，我将结合我的经验，引导你对TCP和UDP进行更深入的理解。</p><p>学习完提高篇之后，我希望你对如何提高TCP及UDP程序的健壮性有一个全面清晰的认识，从而为深入理解性能篇打下良好的基础。</p><p>在前面的基础篇里，我们了解了TCP四次挥手，在四次挥手的过程中，发起连接断开的一方会有一段时间处于TIME_WAIT的状态，你知道TIME_WAIT是用来做什么的么？在面试和实战中，TIME_WAIT相关的问题始终是绕不过去的一道难题。下面就请跟随我，一起找出隐藏在细节下的魔鬼吧。</p><h2>TIME_WAIT发生的场景</h2><p>让我们先从一例线上故障说起。在一次升级线上应用服务之后，我们发现该服务的可用性变得时好时坏，一段时间可以对外提供服务，一段时间突然又不可以，大家都百思不得其解。运维同学登录到服务所在的主机上，使用netstat命令查看后才发现，主机上有成千上万处于TIME_WAIT状态的连接。</p><p>经过层层剖析后，我们发现罪魁祸首就是TIME_WAIT。为什么呢？我们这个应用服务需要通过发起TCP连接对外提供服务。每个连接会占用一个本地端口，当在高并发的情况下，TIME_WAIT状态的连接过多，多到把本机可用的端口耗尽，应用服务对外表现的症状，就是不能正常工作了。当过了一段时间之后，处于TIME_WAIT的连接被系统回收并关闭后，释放出本地端口可供使用，应用服务对外表现为，可以正常工作。这样周而复始，便会出现了一会儿不可以，过一两分钟又可以正常工作的现象。</p><!-- [[[read_end]]] --><p>那么为什么会产生这么多的TIME_WAIT连接呢？</p><p>这要从TCP的四次挥手说起。</p><p><img src="https://static001.geekbang.org/resource/image/f3/e1/f34823ce42a49e4eadaf642a75d14de1.png?wh=874*556" alt=""><br>
TCP连接终止时，主机1先发送FIN报文，主机2进入CLOSE_WAIT状态，并发送一个ACK应答，同时，主机2通过read调用获得EOF，并将此结果通知应用程序进行主动关闭操作，发送FIN报文。主机1在接收到FIN报文后发送ACK应答，此时主机1进入TIME_WAIT状态。</p><p>主机1在TIME_WAIT停留持续时间是固定的，是最长分节生命期MSL（maximum segment lifetime）的两倍，一般称之为2MSL。和大多数BSD派生的系统一样，Linux系统里有一个硬编码的字段，名称为<code>TCP_TIMEWAIT_LEN</code>，其值为60秒。也就是说，<strong>Linux系统停留在TIME_WAIT的时间为固定的60秒。</strong></p><pre><code>#define TCP_TIMEWAIT_LEN (60*HZ) /* how long to wait to destroy TIME-        WAIT state, about 60 seconds	*/
</code></pre><p>过了这个时间之后，主机1就进入CLOSED状态。为什么是这个时间呢？你可以先想一想，稍后我会给出解答。</p><p>你一定要记住一点，<strong>只有发起连接终止的一方会进入TIME_WAIT状态</strong>。这一点面试的时候经常会被问到。</p><h2>TIME_WAIT的作用</h2><p>你可能会问，为什么不直接进入CLOSED状态，而要停留在TIME_WAIT这个状态？</p><p>这要从两个方面来说。</p><p>首先，这样做是为了确保最后的ACK能让被动关闭方接收，从而帮助其正常关闭。</p><p>TCP在设计的时候，做了充分的容错性设计，比如，TCP假设报文会出错，需要重传。在这里，如果图中主机1的ACK报文没有传输成功，那么主机2就会重新发送FIN报文。</p><p>如果主机1没有维护TIME_WAIT状态，而直接进入CLOSED状态，它就失去了当前状态的上下文，只能回复一个RST操作，从而导致被动关闭方出现错误。</p><p>现在主机1知道自己处于TIME_WAIT的状态，就可以在接收到FIN报文之后，重新发出一个ACK报文，使得主机2可以进入正常的CLOSED状态。</p><p>第二个理由和连接“化身”和报文迷走有关系，为了让旧连接的重复分节在网络中自然消失。</p><p>我们知道，在网络中，经常会发生报文经过一段时间才能到达目的地的情况，产生的原因是多种多样的，如路由器重启，链路突然出现故障等。如果迷走报文到达时，发现TCP连接四元组（源IP，源端口，目的IP，目的端口）所代表的连接不复存在，那么很简单，这个报文自然丢弃。</p><p>我们考虑这样一个场景，在原连接中断后，又重新创建了一个原连接的“化身”，说是化身其实是因为这个连接和原先的连接四元组完全相同，如果迷失报文经过一段时间也到达，那么这个报文会被误认为是连接“化身”的一个TCP分节，这样就会对TCP通信产生影响。</p><p><img src="https://static001.geekbang.org/resource/image/94/5f/945c60ae06d282dcc22ad3b868f1175f.png?wh=856*580" alt=""><br>
所以，TCP就设计出了这么一个机制，经过2MSL这个时间，足以让两个方向上的分组都被丢弃，使得原来连接的分组在网络中都自然消失，再出现的分组一定都是新化身所产生的。</p><p>划重点，2MSL的时间是<strong>从主机1接收到FIN后发送ACK开始计时的</strong>；如果在TIME_WAIT时间内，因为主机1的ACK没有传输到主机2，主机1又接收到了主机2重发的FIN报文，那么2MSL时间将重新计时。道理很简单，因为2MSL的时间，目的是为了让旧连接的所有报文都能自然消亡，现在主机1重新发送了ACK报文，自然需要重新计时，以便防止这个ACK报文对新可能的连接化身造成干扰。</p><h2>TIME_WAIT的危害</h2><p>过多的TIME_WAIT的主要危害有两种。</p><p>第一是内存资源占用，这个目前看来不是太严重，基本可以忽略。</p><p>第二是对端口资源的占用，一个TCP连接至少消耗一个本地端口。要知道，端口资源也是有限的，一般可以开启的端口为32768～61000 ，也可以通过<code>net.ipv4.ip_local_port_range</code>指定，如果TIME_WAIT状态过多，会导致无法创建新连接。这个也是我们在一开始讲到的那个例子。</p><h2>如何优化TIME_WAIT？</h2><p>在高并发的情况下，如果我们想对TIME_WAIT做一些优化，来解决我们一开始提到的例子，该如何办呢？</p><h3>net.ipv4.tcp_max_tw_buckets</h3><p>一个暴力的方法是通过sysctl命令，将系统值调小。这个值默认为18000，当系统中处于TIME_WAIT的连接一旦超过这个值时，系统就会将所有的TIME_WAIT连接状态重置，并且只打印出警告信息。这个方法过于暴力，而且治标不治本，带来的问题远比解决的问题多，不推荐使用。</p><h3>调低TCP_TIMEWAIT_LEN，重新编译系统</h3><p>这个方法是一个不错的方法，缺点是需要“一点”内核方面的知识，能够重新编译内核。我想这个不是大多数人能接受的方式。</p><h3>SO_LINGER的设置</h3><p>英文单词“linger”的意思为停留，我们可以通过设置套接字选项，来设置调用close或者shutdown关闭连接时的行为。</p><pre><code>int setsockopt(int sockfd, int level, int optname, const void *optval,
　　　　　　　　socklen_t optlen);
</code></pre><pre><code>struct linger {
　int　 l_onoff;　　　　/* 0=off, nonzero=on */
　int　 l_linger;　　　　/* linger time, POSIX specifies units as seconds */
}
</code></pre><p>设置linger参数有几种可能：</p><ul>
<li>如果<code>l_onoff</code>为0，那么关闭本选项。<code>l_linger</code>的值被忽略，这对应了默认行为，close或shutdown立即返回。如果在套接字发送缓冲区中有数据残留，系统会将试着把这些数据发送出去。</li>
<li>如果<code>l_onoff</code>为非0， 且<code>l_linger</code>值也为0，那么调用close后，会立该发送一个RST标志给对端，该TCP连接将跳过四次挥手，也就跳过了TIME_WAIT状态，直接关闭。这种关闭的方式称为“强行关闭”。 在这种情况下，排队数据不会被发送，被动关闭方也不知道对端已经彻底断开。只有当被动关闭方正阻塞在<code>recv()</code>调用上时，接受到RST时，会立刻得到一个“connet reset by peer”的异常。</li>
</ul><pre><code>struct linger so_linger;
so_linger.l_onoff = 1;
so_linger.l_linger = 0;
setsockopt(s,SOL_SOCKET,SO_LINGER, &amp;so_linger,sizeof(so_linger));
</code></pre><ul>
<li>如果<code>l_onoff</code>为非0， 且<code>l_linger</code>的值也非0，那么调用close后，调用close的线程就将阻塞，直到数据被发送出去，或者设置的<code>l_linger</code>计时时间到。</li>
</ul><p>第二种可能为跨越TIME_WAIT状态提供了一个可能，不过是一个非常危险的行为，不值得提倡。</p><h3>net.ipv4.tcp_tw_reuse：更安全的设置</h3><p>那么Linux有没有提供更安全的选择呢？</p><p>当然有。这就是<code>net.ipv4.tcp_tw_reuse</code>选项。</p><p>Linux系统对于<code>net.ipv4.tcp_tw_reuse</code>的解释如下:</p><pre><code>Allow to reuse TIME-WAIT sockets for new connections when it is safe from protocol viewpoint. Default value is 0.It should not be changed without advice/request of technical experts.
</code></pre><p>这段话的大意是从协议角度理解如果是安全可控的，可以复用处于TIME_WAIT的套接字为新的连接所用。</p><p>那么什么是协议角度理解的安全可控呢？主要有两点：</p><ol>
<li>只适用于连接发起方（C/S模型中的客户端）；</li>
<li>对应的TIME_WAIT状态的连接创建时间超过1秒才可以被复用。</li>
</ol><p>使用这个选项，还有一个前提，需要打开对TCP时间戳的支持，即<code>net.ipv4.tcp_timestamps=1</code>（默认即为1）。</p><p>要知道，TCP协议也在与时俱进，RFC 1323中实现了TCP拓展规范，以便保证TCP的高可用，并引入了新的TCP选项，两个4字节的时间戳字段，用于记录TCP发送方的当前时间戳和从对端接收到的最新时间戳。由于引入了时间戳，我们在前面提到的2MSL问题就不复存在了，因为重复的数据包会因为时间戳过期被自然丢弃。</p><h2>总结</h2><p>在今天的内容里，我讲了TCP的四次挥手，重点对TIME_WAIT的产生、作用以及优化进行了讲解，你需要记住以下三点：</p><ul>
<li>TIME_WAIT的引入是为了让TCP报文得以自然消失，同时为了让被动关闭方能够正常关闭；</li>
<li>不要试图使用<code>SO_LINGER</code>设置套接字选项，跳过TIME_WAIT；</li>
<li>现代Linux系统引入了更安全可控的方案，可以帮助我们尽可能地复用TIME_WAIT状态的连接。</li>
</ul><h2>思考题</h2><p>最后按照惯例，我留两道思考题，供你消化今天的内容。</p><ol>
<li>最大分组MSL是TCP分组在网络中存活的最长时间，你知道这个最长时间是如何达成的？换句话说，是怎么样的机制，可以保证在MSL达到之后，报文就自然消亡了呢？</li>
<li>RFC 1323引入了TCP时间戳，那么这需要在发送方和接收方之间定义一个统一的时钟吗？</li>
</ol><p>欢迎你在评论区写下你的思考，如果通过这篇文章你理解了TIME_WAIT，欢迎你把这篇文章分享给你的朋友或者同事，一起交流学习一下。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/9b/a8/6a391c66.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Leon📷</span>
  </div>
  <div class="_2_QraFYR_0">net.ipv4.tcp_tw_recycle是客户端和服务器端都可以复用，但是容易造成端口接收数据混乱，4.12内核直接砍掉了，老师是因为内核去掉了所以没提了嘛</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: BINGO。太多了，大家也不好接受，我想我还是不求面面俱到，但求有所启发和引领。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-23 19:18:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/7b/10/723a149c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>何某人</span>
  </div>
  <div class="_2_QraFYR_0">老师，那么通过setsockopt设置SO_REUSEADDR这个方法呢？网上资料基本上都是通过设置这个来解决TIME_WAIT。这个方法有什么优劣吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个是解决端口复用的问题，并不是解决TIME_WAIT，这个是告诉内核，即使TIME_WAIT状态的套接字，我也可以继续使用它做为新的套集字使用。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-23 11:08:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/VZ96qlUtD0jEib5gn8mVthSm6sLJ66o1YRn4OgmCseGWBPw055Cw6sYyib5fRFiabnTzl2Nhuomc3qIhgRibkH6iakw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>雷神的盛宴</span>
  </div>
  <div class="_2_QraFYR_0">net.ipv4.tcp_tw_reuse 要慎用，当客户端与服务端主机时间不同步时，客户端的发送的消息会被直接拒绝掉</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 学到了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-20 11:26:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4d/fd/0aa0e39f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>许童童</span>
  </div>
  <div class="_2_QraFYR_0">1.  MSL的意思是最长报文段寿命。IP头部中有个TTL字段意思是生存时间。TTL每经过一个路由器就减1，到0就会被丢弃，而MSL是由RFC里面规定的2分钟，但实际在工程上2分钟太长，因此TCP允许根据具体的情况配置大小，TTL与MSL是有关系的但不是简单的相等关系，MSL要大于TTL。MSL内部应该就是一个普通的定时器实现的。<br>2.不需要统一时钟，可以在第一次交换双方的时钟，之后用相对时间就可以了。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-23 11:22:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/fb/1a/57480c9c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>AnonymousUser</span>
  </div>
  <div class="_2_QraFYR_0">TIME_WAIT的作用：<br>1） 确保对方能够正确收到最后的ACK，帮助其关闭；<br>2） 防迷走报文对程序带来的影响。<br>TIME_WAIT的危害：<br>1） 占用内存；<br>2） 占用端口。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 总结很到位。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-04 11:43:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/8b/ec/dc03f5ad.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张天屹</span>
  </div>
  <div class="_2_QraFYR_0">文中的问题有个前提，必须是监听的端口。我看有评论提到80,8080这种服务，是否只能同时访问一次？答案是否定的。因为网络中的服务分为监听端口和连接端口，当建立一个连接之后，监听端口是不被占用的，此时会用一个新的端口来建立连接，而且就算是新的端口，一个TCP连接也是（客户端ip,客户端端口 ，服务端ip，服务端端口）共同决定的，不冲突。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 帮我回答了，赞</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-30 14:51:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/88/21/50b2418a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>alan</span>
  </div>
  <div class="_2_QraFYR_0">TIME_WAIT的作用：<br>1. 确保主动断开方的最后一个ACK成功发到对方<br>2. 确保残留的TCP包自然消亡</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-23 10:00:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/98/cd/d85c6361.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>丹枫无迹</span>
  </div>
  <div class="_2_QraFYR_0">由于引入了时间戳，我们在前面提到的 2MSL 问题就不复存在了，因为重复的数据包会因为时间戳过期被自然丢弃。<br>这个没理解，为什么 2MSL 问题就不存在了？老师能解释下吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这是因为时间戳会告诉我们报文发送的时间，这样在迷走报文和正确报文同时到达的情况下，我们就可以很方便的分辨出应该丢弃掉那个报文，并不会对最后收到的报文产生任何不利的影响。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-02 15:07:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/6b/1b/4b397b80.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>云师兄</span>
  </div>
  <div class="_2_QraFYR_0">Reuse只适用于连接发起方（C&#47;S 模型中的客户端），但目前要解决的是服务端连接不足问题，这个方法要如何发挥作用呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这里针对的TIME_WAIT是指主动关闭的一方，不一定是客户端或者服务器端，如果服务器端主动关闭连接，也是属于这样的范畴的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-13 09:29:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/33/03/3f2df287.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>吴光庆</span>
  </div>
  <div class="_2_QraFYR_0">为什么是2MSL，不是3 MSL，4 MSL。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 因为是一来一返。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-26 22:04:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/93/02/fcab58d1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>JasonZhi</span>
  </div>
  <div class="_2_QraFYR_0">SO_REUSEADDR和SO_REUSEPORT可以详细说下作用吗？有点迷糊，文章都没有说明这两个参数，评论区就冒出一大堆关于这两个参数的评论。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 哈哈，提前剧透啊。<br><br>这是为了解决如何快速复用处于TIME_WAIT的连接，如果不设置这个选项，处于TIME_WAIT的连接是不能被快速复用的，必须等待系统回收连接才可以，如果这个时候开启服务器端口，会报地址已被占用的错误。<br><br>这块在第15讲里会有详细阐述。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-10 09:55:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83epKJlW7sqts2ZbPuhMbseTAdvHWnrc4ficAeSZyKibkvn6qyxflPrkKKU3mH6XCNmYvDg11tB6y0pxg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>pc</span>
  </div>
  <div class="_2_QraFYR_0">先说点其他的吧：看完了基础篇，收获了很多，也更加期待后面的内容（本来就是冲着time_wait和epoll来到这个课程）。当然其中也遇到很多问题，其中也在评论区提了两个。本来以为这么长时间了老师也不会再回复了，周末一看，竟然回复了我的问题！其实一边是开心，另一边是得到答案的开心（其实自己也搜索过，但是感觉搜到文章都不是我想问的内容）。<br><br>【提问啦～】<br>1、看到评论区的“通过setsockopt设置SO_REUSEADDR这个方法”，感觉和net.ipv4.tcp_tw_reuse选项的作用也很像，都是端口复用，只是后者是在安全可控的基础上---这样理解对吗？<br>2、老师在文中说的“过了2MLS这个时间之后，主机 1 就进入 CLOSED 状态”，我自己还是没有总结出答案，是评论区所说的“去时ACK的最大存活时间（MSL）+来时FIIN的最大存活时间（MSL） = 2MSL”这个原因吗？<br>3、TIME_WAIT=2MLS和TCP_TIMEWAIT_LEN有啥关系？是：TIME_WAIT实际上是由TCP_TIMEWAIT_LEN控制，然后只不过其值约等于2MLS来控制迷走报文的消亡 这样么？<br>4、文中说TCP_TIMEWAIT_LEN、net.ipv4.tcp_tw_reuse都是linux的选项，但是客户端来说的话，有android、ios、windows各种系统吧？是每个系统都有类似的控制选项么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我一直都在回复的哦，和大家在一起自己也学到不少，哈哈。<br><br>以下试着回答你的问题：<br><br>1、看到评论区的“通过setsockopt设置SO_REUSEADDR这个方法”，感觉和net.ipv4.tcp_tw_reuse选项的作用也很像，都是端口复用，只是后者是在安全可控的基础上---这样理解对吗？<br><br>我认为不对哦，前者是针对服务端连接地址被占用的情况，后者是针对连接发起方；<br><br>2、老师在文中说的“过了2MLS这个时间之后，主机 1 就进入 CLOSED 状态”，我自己还是没有总结出答案，是评论区所说的“去时ACK的最大存活时间（MSL）+来时FIIN的最大存活时间（MSL） = 2MSL”这个原因吗？<br><br>你的意思是为什么要制定2MSL这个时间段才CLOSED是么？如果是这样，评论去的&quot;去时ACK的最大存活时间（MSL）+来时FIIN的最大存活时间（MSL） = 2MSL&quot;算是一个靠谱的理解吧。<br><br><br>3、TIME_WAIT=2MLS和TCP_TIMEWAIT_LEN有啥关系？是：TIME_WAIT实际上是由TCP_TIMEWAIT_LEN控制，然后只不过其值约等于2MLS来控制迷走报文的消亡 这样么？<br>TIME_WAIT是一个抽象的定义，而TCP_TIMEWAIT_LEN是Linux默认的值，是一个常量。你的认识是对的<br><br>4、文中说TCP_TIMEWAIT_LEN、net.ipv4.tcp_tw_reuse都是linux的选项，但是客户端来说的话，有android、ios、windows各种系统吧？是每个系统都有类似的控制选项么？<br>Android是裁剪过的Linux，应该可以复用；其他两个我不是很清楚，想必应该有自己的控制选项。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-25 21:30:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/f2/62/f873cd8f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>tongmin_tsai</span>
  </div>
  <div class="_2_QraFYR_0">老师，我有疑问的是，IP包中TTL每经过一次路由就少1，那么2MSL怎么确保可以一定大于TTL的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 只要每次经过一跳的时间肯定大于1秒以上就可以了，实际处理的时间肯定大于这个值的。<br><br>2MSL设置的为60秒，TTL设置为60，只有每次一跳都大于1秒，那么肯定时间总和大于60秒了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-30 15:14:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/55/45/e4314bc6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>magicnum</span>
  </div>
  <div class="_2_QraFYR_0">1.记录一个值，比如60s，经过一个网关就减去一定短值，值=0的时候网关决定丢弃；<br>2.不需要。timestamp不需要交互，只是发送方使用的</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-01 17:46:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/da/8e/eecebc1a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_9a0180</span>
  </div>
  <div class="_2_QraFYR_0">印象中是当一端关闭socket连接，另一端如果尝试从TCP连接中读取数据，则会报connect reset，如果偿试向连接中写入数据，则会报connect reset by peer，好像和老师说的正相反，还请老师帮忙解答一下，谢谢：）</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 首先，我不明白connect reset和connect reset by peer有啥区别哎，这两个不是一回事么？都是RST信号。<br><br>其次，我确认读的时候会读到FIN报文，连续写则会得到RST结果。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-23 21:54:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/20/08/bc06bc69.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Dovelol</span>
  </div>
  <div class="_2_QraFYR_0">老师好，想问个问题，一般我们服务器上运行一个服务，比如tomcat，zk这种，然后监听8080或2181端口，这时候外部直连（不再经过web服务器转发）的话，虽然有很多连接但服务端应该都是只占用一个端口，也就是说netstat -anp命令看到的本机都是ip+固定端口，那么此时如果服务端主动关闭一些连接的话，也会有大量time_wait问题对吧，但此时好像并没有消耗更多端口，那这个影响对于服务端来说是什么呢？老师讲的出现大量time_wait应该都是在客户端的一方吧，因为客户端发起请求会占用一个新端口，主动关闭到time_wait阶段就相当于这个新的端口一直被占用。我还有个疑问是，这种大量time_wait在连接数多的情况下是肯定会出现的，是不是可以从减少连接的方向去解决问题呢，比如用连接池这种技术可以解决吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个时候你看到的连接都是服务器端被动建立的连接，本地端口都是服务器监听的端口，类似<br>&lt;127.0.0.1, 80, ip1, 51231&gt;<br>&lt;127.0.0.1, 80, ip2, 51331&gt;<br>...<br>所以，不会存在我讲到的那个问题，这些个连接过了一段时间自然会被回收。<br><br>连接池是为了多个线程复用连接，减少TCP连接的数量，是为了更高效的使用TCP，确实也客观减少了连接的数量。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-23 09:39:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/9c/a2/06604a01.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>列夫托尔斯泰克洛伊来文列夫斯德夫</span>
  </div>
  <div class="_2_QraFYR_0">这个可控优化的方法没明白，是复用端口的意思吗？不过复用端口数据不混乱了？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不是端口复用，是复用处于 TIME_WAIT 的套接字为新的连接所用。前提在文中也提到了，是通过TCP时间戳来解决2MSL的问题。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-08 21:49:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/b3/c5/7fc124e2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Liam</span>
  </div>
  <div class="_2_QraFYR_0">老师好，我又2个问题不明白：<br>1 为什么说time_wait会占用过多端口，难道不是占用socket而已吗？比如一个server与多个client建立多个连接，对于server而言只会占用一个端口吧<br><br>2 什么是报文的自然消亡，指的是报文发送到对方或报文正常丢弃吗？然后对连接化身这段看不明白</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. 我说的情况是主动断开连接的一方，比如一个客户端每次对外建立一个连接，这是要消耗一个本地端口的，只不过这个端口在我们建立连接的时候由内核帮我们选好了；<br><br>2. 报文的自然消亡，就是TTL时间为0了，不会在网络中继续传播，到了某个网络设备，报文会被丢弃掉。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-24 11:09:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://wx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTI9zRdkKuXMKh30ibeludlAsztmR4rD9iaiclPicOfIhbC4fWxGPz7iceb3o4hKx7qgX2dKwogYvT6VQ0g/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Initiative Thinker</span>
  </div>
  <div class="_2_QraFYR_0">复用后的套接字，如何恢复旧连接的FIN呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是说&quot;回复&quot;吧？<br>处于 TIME_WAIT 的套接字为新的连接所用，通过时间戳可以知道旧连接的FIN是一个无效的FIN，从而直接回复RST，让旧连接直接出错退出。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-24 21:54:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTIVSkMRsMAXJ0sqO9CPwQmQyZ4l0xf0Bn4kIrD8jd2EOaOfHdibmIEhCexC9g9UTgh6tH9tAvd5Mlw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_9edd4f</span>
  </div>
  <div class="_2_QraFYR_0">“第二是对端口资源的占用，一个 TCP 连接至少消耗一个本地端口。要知道，端口资源也是有限的，一般可以开启的端口为 32768～61000 ，也可以通过net.ipv4.ip_local_port_range指定，如果 TIME_WAIT 状态过多，会导致无法创建新连接。这个也是我们在一开始讲到的那个例子。”<br>这里不是很理解，服务端提供服务应该就只用一个端口号吧？而客户端请求服务应该也是只使用一个端口号吧？普通个人客户就发起一个请求只用一个端口号，为什么会出现端口号不够用的情况？难道指的的为客户服务的代理服务器可能会端口号不够用吗？因为代理服务器要处理来自成千上万的客户请求，需要选择不同的端口号为客户服务，将请求发给服务器吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个例子是客户端发起连接过多，每次发起连接都会占用一个端口。客户端和服务端是相对，比如一个应用程序对于客户的请求是服务端，同时为了服务这个客户请求，又要向另一个服务发起调用请求(典型的例子是向数据库发起连接请求)。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-02 19:00:05</div>
  </div>
</div>
</div>
</li>
</ul>