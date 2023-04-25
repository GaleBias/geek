<audio title="15 分析篇 _ 如何高效地分析TCP重传问题？" src="https://static001.geekbang.org/resource/audio/95/96/95e8a4c8069dec64272259f2b054f796.mp3" controls="controls"></audio> 
<p>你好，我是邵亚方。</p><p>我们在基础篇和案例篇里讲了很多问题，比如说RT抖动问题、丢包问题、无法建连问题等等。这些问题通常都会伴随着TCP重传，所以我们往往也会抓取TCP重传信息来辅助我们分析这些问题。</p><p>而且TCP重传也是一个信号，我们通常会利用这个信号来判断系统是否稳定。比如说，如果一台服务器的TCP重传率很高，那这个服务器肯定是存在问题的，需要我们及时采取措施，否则可能会产生更加严重的故障。</p><p>但是，TCP重传率分析并不是一件很容易的事，比如说现在某台服务器的TCP重传率很高，那究竟是什么业务在进行TCP重传呢？对此，很多人并不懂得如何来分析。所以，在这节课中，我会带你来认识TCP重传是怎么回事，以及如何来高效地分析它。</p><h2>什么是TCP重传 ？</h2><p>我在“<a href="https://time.geekbang.org/column/article/273544">开篇词</a>”中举过一个TCP重传率的例子，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/ab/f6/ab358c52ede21f0983fe7dfb032dc3f6.jpg?wh=896*488" alt=""></p><p>这是互联网企业普遍都有的TCP重传率监控，它是服务器稳定性的一个指标，如果它太高，就像上图中的那些毛刺一样，往往就意味着服务器不稳定了。那TCP重传率究竟表示什么呢？</p><p>其实TCP重传率是通过解析/proc/net/snmp这个文件里的指标计算出来的，这个文件里面和TCP有关的关键指标如下：</p><p><img src="https://static001.geekbang.org/resource/image/d5/e7/d5be65df068c3a2c4d181f492791efe7.jpg?wh=2460*1592" alt=""></p><p>TCP重传率的计算公式如下：</p><!-- [[[read_end]]] --><blockquote>
<p>retrans = (RetransSegs－last RetransSegs) ／ (OutSegs－last OutSegs) * 100</p>
</blockquote><p>也就是说，单位时间内TCP重传包的数量除以TCP总的发包数量，就是TCP重传率。那我们继续看下这个公式中的RetransSegs和OutSegs是怎么回事，我画了两张示例图来演示这两个指标的变化：</p><p><img src="https://static001.geekbang.org/resource/image/ed/54/ed69e93e3c13f0e117021e399500e854.jpg?wh=3600*1889" alt="" title="不存在重传的情况"></p><p><img src="https://static001.geekbang.org/resource/image/0a/b6/0a28a0596bd56174feaec0d82245b5b6.jpg?wh=3478*2009" alt="" title="存在重传的情况"></p><p>通过这两个示例图，你可以发现，发送端在发送一个TCP数据包后，会把该数据包放在发送端的发送队列里，也叫重传队列。此时，OutSegs会相应地加1，队列长度也为1。如果可以收到接收端对这个数据包的ACK，该数据包就会在发送队列中被删掉，然后队列长度变为0；如果收不到这个数据包的ACK，就会触发重传机制，我们在这里演示的就是超时重传这种情况，也就是说发送端在发送数据包的时候，会启动一个超时重传定时器（RTO），如果超过了这个时间，发送端还没有收到ACK，就会重传该数据包，然后OutSegs加1，同时RetransSegs也会加1。</p><p>这就是OutSegs和RetransSegs的含义：每发出去一个TCP包（包括重传包），OutSegs会相应地加1；每发出去一个重传包，RetransSegs会相应地加1。同时，我也在图中展示了重传队列的变化，你可以仔细看下。</p><p>除了上图中展示的超时重传外，还有快速重传机制。关于快速重传，你可以参考“<a href="https://time.geekbang.org/column/article/286494">13讲</a>”，我就不在这里详细描述了。</p><p>明白了TCP重传是如何定义的之后，我们继续来看下哪些情况会导致TCP重传。</p><p>引起TCP重传的情况在整体上可以分为如下两类。</p><ul>
<li><strong>丢包</strong><br>
TCP数据包在网络传输过程中可能会被丢弃；接收端也可能会把该数据包给丢弃；接收端回的ACK也可能在网络传输过程中被丢弃；数据包在传输过程中发生错误而被接收端给丢弃……这些情况都会导致发送端重传该TCP数据包。</li>
<li><strong>拥塞</strong><br>
TCP数据包在网络传输过程中可能会在某个交换机/路由器上排队，比如臭名昭著的Bufferbloat（缓冲膨胀）；TCP数据包在网络传输过程中因为路由变化而产生的乱序；接收端回的ACK在某个交换机/路由器上排队……这些情况都会导致发送端再次重传该TCP数据包。</li>
</ul><p>总之，TCP重传可以很好地作为通信质量的信号，我们需要去重视它。</p><p>那么，当我们发现某个主机上TCP重传率很高时，该如何去分析呢？</p><h2>分析TCP重传的常规手段</h2><p>最常规的分析手段就是tcpdump，我们可以使用它把进出某个网卡的数据包给保存下来：</p><pre><code>$ tcpdump -s 0 -i eth0 -w tcpdumpfile
</code></pre><p>然后在Linux上我们可以使用tshark这个工具（wireshark的Linux版本）来过滤出TCP重传包：</p><pre><code>$ tshark -r tcpdumpfile -R tcp.analysis.retransmission
</code></pre><p>如果有重传包的话，就可以显示出来了，如下是一个TCP重传的示例：</p><pre><code>3481  20.277303 10.17.130.20 -&gt; 124.74.250.144 TCP 70 [TCP Retransmission] 35993 &gt; https [SYN] Seq=0 Win=14600 Len=0 MSS=1460 SACK_PERM=1 TSval=3231504691 TSecr=0

3659  22.277070 10.17.130.20 -&gt; 124.74.250.144 TCP 70 [TCP Retransmission] 35993 &gt; https [SYN] Seq=0 Win=14600 Len=0 MSS=1460 SACK_PERM=1 TSval=3231506691 TSecr=0

8649  46.539393 58.216.21.165 -&gt; 10.17.130.20 TLSv1 113 [TCP Retransmission] Change Cipher Spec, Encrypted Handshake Messag
</code></pre><p>借助tcpdump，我们就可以看到TCP重传的详细情况。从上面这几个TCP重传信息中，我们可以看到，这是发生在10.17.130.20:35993 - 124.74.250.144: 443这个TCP连接上的重传；通过[SYN]这个TCP连接状态，可以看到这是发生在三次握手阶段的重传。依据这些信息，我们就可以继续去124.74.250.144这个主机上分析https这个服务为什么无法建立新的连接了。</p><p>但是，我们都知道tcpdump很重，如果直接在生产环境上进行采集的话，难免会对业务造成性能影响。那有没有更加轻量级的一些分析方法呢？</p><h2>如何高效地分析TCP重传 ？</h2><p>其实，就像应用程序实现一些功能需要调用对应的函数一样，TCP重传也需要调用特定的内核函数。这个内核函数就是tcp_retransmit_skb()。你可以把这个函数名字里的skb理解为是一个需要发送的网络包。那么，如果我们想要高效地追踪TCP重传情况，那么直接追踪该函数就可以了。</p><p>追踪内核函数最通用的方法是使用Kprobe，Kprobe的大致原理如下：</p><p><img src="https://static001.geekbang.org/resource/image/9f/c8/9f3f412208d8e17dd859a97b017228c8.jpg?wh=3080*1771" alt="" title="Kprobe基本原理"></p><p>你可以实现一个内核模块，该内核模块中使用Kprobe在tcp_retransmit_skb这个函数入口插入一个probe，然后注册一个break_handler，这样在执行到tcp_retransmit_skb时就会异常跳转到注册的break_handler中，然后在break_handler中解析TCP报文（skb）就可以了，从而来判断是什么在重传。</p><p>如果你觉得实现内核模块比较麻烦，可以借助ftrace框架来使用Kprobe。Brendan Gregg实现的<a href="https://github.com/brendangregg/perf-tools/blob/master/net/tcpretrans">tcpretrans</a>采用的就是这种方式，你也可以直接使用它这个工具来追踪TCP重传。不过，该工具也有一些缺陷，因为它是通过读取/proc/net/tcp这个文件来解析是什么在重传，所以它能解析的信息比较有限，而且如果TCP连接持续时间较短（比如短连接），那么该工具就无法解析出来了。另外，你在使用它时需要确保你的内核已经打开了ftrace的tracing功能，也就是/sys/kernel/debug/tracing/tracing_on中的内容需要为1；在CentOS-6上，还需要/sys/kernel/debug/tracing/tracing_enabled也为1。</p><pre><code>$ cat /sys/kernel/debug/tracing/tracing_on 
1
</code></pre><p>如果为0的话，你需要打开它们，例如：</p><pre><code>$ echo 1 &gt; /sys/kernel/debug/tracing/tracing_on 
</code></pre><p>然后在追踪结束后，你需要来关闭他们：</p><pre><code>$ echo 0 &gt; /sys/kernel/debug/tracing/tracing_on 
</code></pre><p>由于Kprobe是通过异常（Exception）这种方式来工作的，所以它还是有一些性能开销的，在TCP发包快速路径上还是要避免使用Kprobe。不过，由于重传路径是慢速路径，所以在重传路径上添加Kprobe也无需担心性能开销。</p><p>Kprobe这种方式使用起来还是略有些不便，为了让Linux用户更方便地观察TCP重传事件，4.16内核版本中专门添加了<a href="https://github.com/torvalds/linux/commit/e086101b150ae8e99e54ab26101ef3835fa9f48d">TCP tracepoint</a>来解析TCP重传事件。如果你使用的操作系统是CentOS-7以及更老的版本，就无法使用该Tracepoint来观察了；如果你的版本是CentOS-8以及后续更新的版本，那你可以直接使用这个Tracepoint来追踪TCP重传，可以使用如下命令：</p><pre><code>$ cd /sys/kernel/debug/tracing/events/
$ echo 1 &gt; tcp/tcp_retransmit_skb/enable
</code></pre><p>然后你就可以追踪TCP重传事件了：</p><pre><code>$ cat trace_pipe
&lt;idle&gt;-0     [007] ..s. 265119.290232: tcp_retransmit_skb: sport=22 dport=62264 saddr=172.23.245.8 daddr=172.30.18.225 saddrv6=::ffff:172.23.245.8 daddrv6=::ffff:172.30.18.225 state=TCP_ESTABLISHED
</code></pre><p>可以看到，当TCP重传发生时，该事件的基本信息就会被打印出来。多说一句，在最开始的版本中是没有“state=TCP_ESTABLISHED”这一项的。如果没有这一项，我们就无法识别该重传事件是不是发生在三次握手阶段了，所以我给内核贡献了一个PATCH来显示TCP连接状态，以便于问题分析，具体见<a href="https://github.com/torvalds/linux/commit/af4325ecc24f45933d5567e72227cff2c1594764">tcp: expose sk_state in tcp_retransmit_skb tracepoint</a>这个commit。</p><p>追踪结束后呢，你需要将这个Tracepoint给关闭：</p><pre><code>$ echo 0 &gt; tcp/tcp_retransmit_skb/enable
</code></pre><p>Tracepoint这种方式不仅使用起来更加方便，而且它的性能开销比Kprobe要小，所以我们在快速路径上也可以使用它。</p><p>因为Tracepoint对TCP重传事件的支持，所以tcpretrans这个工具也跟着进行了一次升级换代。它通过解析该Tracepoint实现了对TCP重传事件的追踪，而不再使用之前的Kprobe方式，具体你可以参考<a href="https://github.com/iovisor/bcc/blob/master/tools/tcpretrans.py">bcc tcpretrans</a>。再多说一句，Brendan Gregg在实现这些基于ebpf的TCP追踪工具之前也曾经跟我讨论过，所以我对他的这个工具才会这么熟悉。</p><p>我们针对TCP重传事件的分析就先讲到这里，希望能给你带来一些启发，去开发一些更加高效的工具来分析你遇到的TCP问题或者其他问题。</p><h2>课堂总结</h2><p>这堂课我们主要讲了TCP重传的一些知识，关于TCP重传你需要重点记住下面这几点：</p><ul>
<li>TCP重传率可以作为TCP通信质量的信号，如果它很高，那说明这个TCP连接很不稳定；</li>
<li>产生TCP重传的问题主要是丢包和网络拥塞这两种情况；</li>
<li>TCP重传时会调用特定的内核函数，我们可以追踪该函数的调用情况来追踪TCP重传事件；</li>
<li>Kprobe是一个很通用的追踪工具，在低版本内核上，你可以使用这个方法来追踪TCP重传事件；</li>
<li>Tracepoint是一个更加轻量级也更加方便的追踪TCP重传的工具，但是需要你的内核版本为4.16+；</li>
<li>如果你想要更简单些，那你可以直接使用tcpretrans这个工具。</li>
</ul><h2>课后作业</h2><p>请问我们提到的tracepoint观察方式，或者tcpretrans这个工具，可以追踪收到的TCP重传包吗？为什么？欢迎你在留言区与我讨论。</p><p>感谢你的阅读，如果你认为这节课的内容有收获，也欢迎把它分享给你的朋友，我们下一讲见。</p>
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
  <div class="_2_QraFYR_0"><br>课后作业答案：<br>- 请问我们提到的 tracepoint 观察方式，或者 tcpretrans 这个工具，可以追踪收到的 TCP 重传包吗？为什么？<br>不可以，因为tracepoint是在发送的地方进行打点来追踪的重传包，所以无法追踪收到的重传包。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-11 14:02:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4c/49/d21c134f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>swordholder</span>
  </div>
  <div class="_2_QraFYR_0">从本文了解到的ftrace简直打开了新天地。<br>学习后写了篇ftrace的介绍文章 https:&#47;&#47;github.com&#47;mz1999&#47;blog&#47;blob&#47;master&#47;docs&#47;ftrace.md</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-30 17:22:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/4a/2c/f8451d77.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>石维康</span>
  </div>
  <div class="_2_QraFYR_0">『如果你觉得实现内核模块比较麻烦，可以借助 ftrace 框架来使用 Kprobe。Brendan Gregg 实现的tcpretrans采用的就是这种方式，你也可以直接使用它这个工具来追踪 TCP 重传。』<br><br>『。它通过解析该 Tracepoint 实现了对 TCP 重传事件的追踪，而不再使用之前的 Kprobe 方式，』</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-22 10:25:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/5c/bd/f5d0a2c8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>逗比</span>
  </div>
  <div class="_2_QraFYR_0">最近遇到一个情况，关掉接收方的网卡GRO就没有重传，乱序，重复ack现象，但是不知道为什么<br><br>场景：A（虚拟机ip） -&gt; B(虚拟机ip:port)， ipvs转发到某一台pod ip，其实就是一个普通的k8s service</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-30 14:38:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/3a/a5/217df1c5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>尼克</span>
  </div>
  <div class="_2_QraFYR_0">请问老师，路由变化为什么会导致发送包乱序呢？这段时间生产有个问题，ospf刷新的时候会有很多的重传，但是我不理解为什么路由变化会导致乱序，我理解的seq 不应该是同一个包是一直不变的吗，谢谢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-28 19:50:59</div>
  </div>
</div>
</div>
</li>
</ul>