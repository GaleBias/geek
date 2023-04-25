<audio title="12 基础篇 _ TCP收发包过程会受哪些配置项影响？" src="https://static001.geekbang.org/resource/audio/db/b5/db010a0a728f81ebbf5ebc81e49abab5.mp3" controls="controls"></audio> 
<p>你好，我是邵亚方。我们这节课来讲一下，TCP数据在传输过程中会受到哪些因素干扰。</p><p>TCP收包和发包的过程也是容易引起问题的地方。收包是指数据到达网卡再到被应用程序开始处理的过程。发包则是应用程序调用发包函数到数据包从网卡发出的过程。你应该对TCP收包和发包过程中容易引发的一些问题不会陌生，比如说：</p><ul>
<li>网卡中断太多，占用太多CPU，导致业务频繁被打断；</li>
<li>应用程序调用write()或者send()发包，怎么会发不出去呢；</li>
<li>数据包明明已经被网卡收到了，可是应用程序为什么没收到呢；</li>
<li>我想要调整缓冲区的大小，可是为什么不生效呢；</li>
<li>是不是内核缓冲区满了从而引起丢包，我该怎么观察呢；</li>
<li>…</li>
</ul><p>想要解决这些问题呢，你就需要去了解TCP的收发包过程容易受到哪些因素的影响。这个过程中涉及到很多的配置项，很多问题都是这些配置项跟业务场景不匹配导致的。</p><p>我们先来看下数据包的发送过程，这个过程会受到哪些配置项的影响呢？</p><h2>TCP数据包的发送过程会受什么影响？</h2><p><img src="https://static001.geekbang.org/resource/image/5c/5e/5ce5d202b7a179829f4c9b3863b0b15e.jpg?wh=4500*3274" alt="" title="TCP数据包发送过程"></p><p>上图就是一个简略的TCP数据包的发送过程。应用程序调用write(2)或者send(2)系列系统调用开始往外发包时，这些系统调用会把数据包从用户缓冲区拷贝到TCP发送缓冲区（TCP Send Buffer），这个TCP发送缓冲区的大小是受限制的，这里也是容易引起问题的地方。</p><!-- [[[read_end]]] --><p>TCP发送缓冲区的大小默认是受net.ipv4.tcp_wmem来控制：</p><blockquote>
<p>net.ipv4.tcp_wmem = 8192 65536 16777216</p>
</blockquote><p>tcp_wmem中这三个数字的含义分别为min、default、max。TCP发送缓冲区的大小会在min和max之间动态调整，初始的大小是default，这个动态调整的过程是由内核自动来做的，应用程序无法干预。自动调整的目的，是为了在尽可能少的浪费内存的情况下来满足发包的需要。</p><p>tcp_wmem中的max不能超过net.core.wmem_max这个配置项的值，如果超过了，TCP 发送缓冲区最大就是net.core.wmem_max。通常情况下，我们需要设置net.core.wmem_max的值大于等于net.ipv4.tcp_wmem的max：</p><blockquote>
<p>net.core.wmem_max = 16777216</p>
</blockquote><p>对于TCP 发送缓冲区的大小，我们需要根据服务器的负载能力来灵活调整。通常情况下我们需要调大它们的默认值，我上面列出的 tcp_wmem 的 min、default、max 这几组数值就是调大后的值，也是我们在生产环境中配置的值。</p><p>我之所以将这几个值给调大，是因为我们在生产环境中遇到过TCP发送缓冲区太小，导致业务延迟很大的问题，这类问题也是可以使用systemtap之类的工具在内核里面打点来进行观察的（观察sk_stream_wait_memory这个事件）:</p><pre><code># sndbuf_overflow.stp
# Usage :
# $ stap sndbuf_overflow.stp
probe kernel.function(&quot;sk_stream_wait_memory&quot;)
{
    printf(&quot;%d %s TCP send buffer overflow\n&quot;,
         pid(), execname())
}
</code></pre><p>如果你可以观察到sk_stream_wait_memory这个事件，就意味着TCP发送缓冲区太小了，你需要继续去调大wmem_max和tcp_wmem:max的值了。</p><p>应用程序有的时候会很明确地知道自己发送多大的数据，需要多大的TCP发送缓冲区，这个时候就可以通过setsockopt(2)里的SO_SNDBUF来设置固定的缓冲区大小。一旦进行了这种设置后，tcp_wmem就会失效，而且这个缓冲区大小设置的是固定值，内核也不会对它进行动态调整。</p><p>但是，SO_SNDBUF设置的最大值不能超过net.core.wmem_max，如果超过了该值，内核会把它强制设置为net.core.wmem_max。所以，如果你想要设置SO_SNDBUF，一定要确认好net.core.wmem_max是否满足需求，否则你的设置可能发挥不了作用。通常情况下，我们都不会通过SO_SNDBUF来设置TCP发送缓冲区的大小，而是使用内核设置的tcp_wmem，因为如果SO_SNDBUF设置得太大就会浪费内存，设置得太小又会引起缓冲区不足的问题。</p><p>另外，如果你关注过Linux的最新技术动态，你一定听说过eBPF。你也可以通过eBPF来设置SO_SNDBUF和SO_RCVBUF，进而分别设置TCP发送缓冲区和TCP接收缓冲区的大小。同样地，使用eBPF来设置这两个缓冲区时，也不能超过wmem_max和rmem_max。不过eBPF在一开始增加设置缓冲区大小的特性时并未考虑过最大值的限制，我在使用的过程中发现这里存在问题，就给社区提交了一个PATCH把它给修复了。你感兴趣的话可以看下这个链接：<a href="https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?h=v5.8&amp;id=c9e4576743eeda8d24dedc164d65b78877f9a98c">bpf: sock recvbuff must be limited by rmem_max in bpf_setsockopt()</a>。</p><p>tcp_wmem以及wmem_max的大小设置都是针对单个TCP连接的，这两个值的单位都是Byte（字节）。系统中可能会存在非常多的TCP连接，如果TCP连接太多，就可能导致内存耗尽。因此，所有TCP连接消耗的总内存也有限制：</p><blockquote>
<p>net.ipv4.tcp_mem = 8388608 12582912 16777216</p>
</blockquote><p>我们通常也会把这个配置项给调大。与前两个选项不同的是，该选项中这些值的单位是Page（页数），也就是4K。它也有3个值：min、pressure、max。当所有TCP连接消耗的内存总和达到max后，也会因达到限制而无法再往外发包。</p><p>因tcp_mem达到限制而无法发包或者产生抖动的问题，我们也是可以观测到的。为了方便地观测这类问题，Linux内核里面预置了静态观测点：sock_exceed_buf_limit。不过，这个观测点一开始只是用来观察TCP接收时遇到的缓冲区不足的问题，不能观察TCP发送时遇到的缓冲区不足的问题。后来，我提交了一个patch做了改进，使得它也可以用来观察TCP发送时缓冲区不足的问题：<a href="https://github.com/torvalds/linux/commit/563e0bb0dc74b3ca888e24f8c08f0239fe4016b0">net: expose sk wmem in sock_exceed_buf_limit tracepoint</a>  ，观察时你只需要打开tracepiont（需要4.16+的内核版本）：</p><blockquote>
<p>$ echo 1 &gt; /sys/kernel/debug/tracing/events/sock/sock_exceed_buf_limit/enable</p>
</blockquote><p>然后去看是否有该事件发生：</p><blockquote>
<p>$ cat /sys/kernel/debug/tracing/trace_pipe</p>
</blockquote><p>如果有日志输出（即发生了该事件），就意味着你需要调大tcp_mem了，或者是需要断开一些TCP连接了。</p><p>TCP层处理完数据包后，就继续往下来到了IP层。IP层这里容易触发问题的地方是net.ipv4.ip_local_port_range这个配置选项，它是指和其他服务器建立IP连接时本地端口（local port）的范围。我们在生产环境中就遇到过默认的端口范围太小，以致于无法创建新连接的问题。所以通常情况下，我们都会扩大默认的端口范围：</p><blockquote>
<p>net.ipv4.ip_local_port_range = 1024 65535</p>
</blockquote><p>为了能够对TCP/IP数据流进行流控，Linux内核在IP层实现了qdisc（排队规则）。我们平时用到的TC就是基于qdisc的流控工具。qdisc的队列长度是我们用ifconfig来看到的txqueuelen，我们在生产环境中也遇到过因为txqueuelen太小导致数据包被丢弃的情况，这类问题可以通过下面这个命令来观察：</p><blockquote>
<p>$ ip -s -s link ls dev eth0<br>
…<br>
TX: bytes packets errors dropped carrier collsns<br>
3263284 25060 0 0 0 0</p>
</blockquote><p>如果观察到dropped这一项不为0，那就有可能是txqueuelen太小导致的。当遇到这种情况时，你就需要增大该值了，比如增加eth0这个网络接口的txqueuelen：</p><blockquote>
<p>$ ifconfig eth0 txqueuelen 2000</p>
</blockquote><p>或者使用ip这个工具：</p><blockquote>
<p>$ ip link set eth0 txqueuelen 2000</p>
</blockquote><p>在调整了txqueuelen的值后，你需要持续观察是否可以缓解丢包的问题，这也便于你将它调整到一个合适的值。</p><p>Linux系统默认的qdisc为pfifo_fast（先进先出），通常情况下我们无需调整它。如果你想使用<a href="https://github.com/google/bbr">TCP BBR</a>来改善TCP拥塞控制的话，那就需要将它调整为fq（fair queue, 公平队列）：</p><blockquote>
<p>net.core.default_qdisc = fq</p>
</blockquote><p>经过IP层后，数据包再往下就会进入到网卡了，然后通过网卡发送出去。至此，你需要发送出去的数据就走完了TCP/IP协议栈，然后正常地发送给对端了。</p><p>接下来，我们来看下数据包是怎样收上来的，以及在接收的过程中会受哪些配置项的影响。</p><h2>TCP数据包的接收过程会受什么影响？</h2><p>TCP数据包的接收过程，同样也可以用一张图来简单表示：</p><p><img src="https://static001.geekbang.org/resource/image/9c/56/9ca34a53abf57125334e0278edd10356.jpg?wh=4500*3193" alt="" title="TCP数据包接收过程"></p><p>从上图可以看出，TCP数据包的接收流程在整体上与发送流程类似，只是方向是相反的。数据包到达网卡后，就会触发中断（IRQ）来告诉CPU读取这个数据包。但是在高性能网络场景下，数据包的数量会非常大，如果每来一个数据包都要产生一个中断，那CPU的处理效率就会大打折扣，所以就产生了NAPI（New API）这种机制让CPU一次性地去轮询（poll）多个数据包，以批量处理的方式来提升效率，降低网卡中断带来的性能开销。</p><p>那在poll的过程中，一次可以poll多少个呢？这个poll的个数可以通过sysctl选项来控制：</p><blockquote>
<p>net.core.netdev_budget = 600</p>
</blockquote><p>该控制选项的默认值是300，在网络吞吐量较大的场景中，我们可以适当地增大该值，比如增大到600。增大该值可以一次性地处理更多的数据包。但是这种调整也是有缺陷的，因为这会导致CPU在这里poll的时间增加，如果系统中运行的任务很多的话，其他任务的调度延迟就会增加。</p><p>接下来继续看TCP数据包的接收过程。我们刚才提到，数据包到达网卡后会触发CPU去poll数据包，这些poll的数据包紧接着就会到达IP层去处理，然后再达到TCP层，这时就会面对另外一个很容易引发问题的地方了：TCP Receive Buffer（TCP接收缓冲区）。</p><p>与 TCP发送缓冲区类似，TCP接收缓冲区的大小也是受控制的。通常情况下，默认都是使用tcp_rmem来控制缓冲区的大小。同样地，我们也会适当地增大这几个值的默认值，来获取更好的网络性能，调整为如下数值：</p><blockquote>
<p>net.ipv4.tcp_rmem = 8192 87380 16777216</p>
</blockquote><p>它也有3个字段：min、default、max。TCP接收缓冲区大小也是在min和max之间动态调整 ，不过跟发送缓冲区不同的是，这个动态调整是可以通过控制选项来关闭的，这个选项是tcp_moderate_rcvbuf 。通常我们都是打开它，这也是它的默认值：</p><blockquote>
<p>net.ipv4.tcp_moderate_rcvbuf = 1</p>
</blockquote><p>之所以接收缓冲区有选项可以控制自动调节，而发送缓冲区没有，那是因为TCP接收缓冲区会直接影响TCP拥塞控制，进而影响到对端的发包，所以使用该控制选项可以更加灵活地控制对端的发包行为。</p><p>除了tcp_moderate_rcvbuf 可以控制TCP接收缓冲区的动态调节外，也可以通过setsockopt()中的配置选项SO_RCVBUF来控制，这与TCP发送缓冲区是类似的。如果应用程序设置了SO_RCVBUF这个标记，那么TCP接收缓冲区的动态调整就是关闭，即使tcp_moderate_rcvbuf为1，接收缓冲区的大小始终就为设置的SO_RCVBUF这个值。</p><p>也就是说，只有在tcp_moderate_rcvbuf为1，并且应用程序没有通过SO_RCVBUF来配置缓冲区大小的情况下，TCP接收缓冲区才会动态调节。</p><p>同样地，与TCP发送缓冲区类似，SO_RCVBUF设置的值最大也不能超过net.core.rmem_max。通常情况下，我们也需要设置net.core.rmem_max的值大于等于net.ipv4.tcp_rmem的max：</p><blockquote>
<p>net.core.rmem_max = 16777216</p>
</blockquote><p>我们在生产环境中也遇到过，因达到了TCP接收缓冲区的限制而引发的丢包问题。但是这类问题不是那么好追踪的，没有一种很直观地追踪这种行为的方式，所以我便在我们的内核里添加了针对这种行为的统计。</p><p>为了让使用Linux内核的人都能很好地观察这个行为，我也把我们的实践贡献给了Linux内核社区，具体可以看这个commit：<a href="https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?h=v5.9-rc1&amp;id=ea5d0c32498e1a08ff5f3dbeafa4d74895851b0d">tcp: add new SNMP counter for drops when try to queue in rcv queue</a>。使用这个SNMP计数，我们就可以很方便地通过netstat查看，系统中是否存在因为TCP接收缓冲区不足而引发的丢包。</p><p>不过，该方法还是存在一些局限：如果我们想要查看是哪个TCP连接在丢包，那么这种方式就不行了，这个时候我们就需要去借助其他一些更专业的trace工具，比如eBPF来达到我们的目的。</p><h2>课堂总结</h2><p>好了，这节课就讲到这里，我们简单回顾一下。TCP/IP是一个很复杂的协议栈，它的数据包收发过程也是很复杂的，我们这节课只是重点围绕这个过程中最容易引发问题的地方来讲述的。我们刚才提到的那些配置选项都很容易在生产环境中引发问题，并且也是我们针对高性能网络进行调优时必须要去考虑的。我把这些配置项也总结为了一个表格，方便你来查看：</p><p><img src="https://static001.geekbang.org/resource/image/8d/9b/8d4ba95a95684004f271677f600cda9b.jpg?wh=3946*2779" alt=""></p><p>这些值都需要根据你的业务场景来做灵活的调整，当你不知道针对你的业务该如何调整时，你最好去咨询更加专业的人员，或者一边调整一边观察系统以及业务行为的变化。</p><h2>课后作业</h2><p>我们这节课中有两张图，分别是TCP数据包的发送过程 和 TCP数据包的接收过程，我们可以看到，在TCP发送过程中使用到了qdisc，但是在接收过程中没有使用它，请问是为什么？我们可以在接收过程中也使用qdisc吗？欢迎你在留言区与我讨论。</p><p>感谢你的阅读，如果你认为这节课的内容有收获，也欢迎把它分享给你的朋友，我们下一讲见。</p>
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
  <div class="_2_QraFYR_0">课后作业答案：<br>- 在 TCP 发送过程中使用到了 qdisc，但是在接收过程中没有使用它，请问是为什么？我们可以在接收过程中也使用 qdisc 吗？<br>qdisc的主要目的是流控，流控一般都是在发送端进行控制；对于接收端而言，它已经收到这个包了，再进行流控的话，也就只有选择性的丢包。如果在接收端也使用qdisc之类的流控机制，也需要将它模拟为发送端，也就是增加一个中间设备来做。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-11 13:50:31</div>
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
  <div class="_2_QraFYR_0">终于到TCP篇了.<br><br>看了文中的`TCP数据包发送过程`图,有几个疑问:<br>1. 如果用tcpdump抓包,它是在哪一层抓的包呢?(IP Layer &#47; Link Layer)<br>   最近遇到的问题,就是调用write函数有返回值.但是tcpdump抓包来看,并没有迹象.<br>   只知道在此期间有地方把包给丢了,并不知道具体是哪一层丢的.后来发现是内核把包丢了.<br>2. `TCP Send Buffer`默认是动态调整的.<br>   这个是按需分配的意思么?如果我调整了内核参数,对之前建立的连接产生影响么?<br>3. 如果`TCP Send Buffer`满了,调用`write`时是阻塞还是返回错误码呢?(是不是跟TCP的阻塞&#47;非阻塞模式有关?)<br><br>-------------------<br>最近在CentOS 7.6上遇到了一个TCP内核方面的问题.<br>它的内核版本太低了,还是linux-3.10.0-957.21.3.el7.<br><br>具体的分析和解决过程参考了这篇博文:<br>[TCP SACK 补丁导致的发送队列报文积压](https:&#47;&#47;runsisi.com&#47;2019-12-19&#47;tcp-sack-hang)<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. 这个tcpdump的原理，我们在后面的课程里会讲到，是在link layer来抓包的。<br>2. 对的，会按需调整，调整后会影响之前的连接，因为在检查缓冲区大小时会用到这些全局变量。<br>3. 对的 跟是否设置了阻塞模式有关。<br><br><br>这个blog分享得不错，赞！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-15 15:45:50</div>
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
  <div class="_2_QraFYR_0">又一个问题需要帮忙解答一下，就是网络收包一共会又几次内存拷贝的流程。<br>看到一篇文章中说DMA也算一次的化，会又三次？（https:&#47;&#47;blog.csdn.net&#47;gengzhikui1992&#47;article&#47;details&#47;103142848）<br>对1，2 步中内存拷贝没有理解透。<br>1、DMA操作，网卡寄存器-&gt;内核为网卡分配的缓冲区ring buffer<br>  ring buffer存储的描述符的索引，索引对应存储存储报文的物理地址吧&#47;<br>2、驱动软件从ring buffer中读取，填充内核skbuff结构（第2次拷贝：内核网卡缓冲区ring buffer-&gt;内核专用数据结构skbuff）<br> 它是把填充skbuff头部也当作了一次内存拷贝吗？<br>3、socket系统调用将数据从内核搬移到用户态。(第3次拷贝：内核空间-&gt;用户空间)<br>这个是系统调用，比较好理解。<br><br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-09 14:05:02</div>
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
  <div class="_2_QraFYR_0">从tcp的发送过程和接受过程，讲解过程中可能影响的配置选项。让我既能很好的理解发送和接受过程，又学习了如何结合生产环境业务场景进行性能优化，真是收货满满。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-05 17:58:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/18/fb/c1334976.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>王崧霁</span>
  </div>
  <div class="_2_QraFYR_0">流控应该在上游发送端控制，接收端有个开关net.core.devbudget也是控制发端行为</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-19 16:11:26</div>
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
  <div class="_2_QraFYR_0">捉个虫，课堂总结第一行中关于 tcp_wmem 的说明，应该是「如果通过SO_SNDBUF来设置发送发送缓冲区」而是不「SO_RECVBUF」</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-03 22:52:57</div>
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
  <div class="_2_QraFYR_0">有个疑惑，老师提到qdisc是在ip层里实现的，但在看代码的时候发现，qdisc是在 __dev_queue_xmit （数据链路层）中被使用到，那qdisc是属于哪一层的呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-04 09:05:06</div>
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
  <div class="_2_QraFYR_0">老师这种情况从何处入手：<br>eth0: flags=4163&lt;UP,BROADCAST,RUNNING,MULTICAST&gt;  mtu 9001<br>        inet 10.201.80.130  netmask 255.255.224.0  broadcast 10.201.95.255<br>        ether 02:c6:7a:df:c2:09  txqueuelen 1000  (Ethernet)<br>        RX packets 55403028044  bytes 88466263876451 (80.4 TiB)<br>        RX errors 0  dropped 1432413  overruns 0  frame 0<br>        TX packets 111313645859  bytes 182202219572067 (165.7 TiB)<br>        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-08 20:15:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/4d/fe/882eaf0f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>威</span>
  </div>
  <div class="_2_QraFYR_0">如果用 sysctl -p 来使得tcp缓冲区配置立刻生效，这样做之后，已建立了的tcp链接缓冲区大小会改变吗</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-18 18:10:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/53/b5/d914f2c2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>晴天</span>
  </div>
  <div class="_2_QraFYR_0">老师，还有个内核参数netdev_max_backlog也值得讲讲</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-02 22:05:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/68/12/031a05c3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>A免帅叫哥</span>
  </div>
  <div class="_2_QraFYR_0">19年1月份的commit，合并到5.8的内核，merge也挺慢的。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-06 08:37:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJpJXWFP3dNle88WnTkRTsEQkPJmOhepibiaTfhEtMRrbdg5EAWm4EzurA61oKxvCK2ZjMmy1QvmChw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>唐江</span>
  </div>
  <div class="_2_QraFYR_0">发送和接收端的缓冲区都是针对单个连接的吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 单个tcp连接有自己的缓冲区控制 tcp协议栈也有针对所有连接的统一的缓冲区控制</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-26 08:17:47</div>
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
  <div class="_2_QraFYR_0">用户通过openvpn连接服务器，然后使用的udp报文，然后测速发现连接VPN后的速度比连接VPN前要慢的多，但是服务器的入口带宽其实还剩余很多，这个是由于udp的缓存设置导致的吗？老师能解答下吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-28 14:36:48</div>
  </div>
</div>
</div>
</li>
</ul>