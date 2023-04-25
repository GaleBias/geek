<audio title="06 _ 事件触发：各类eBPF程序的触发机制及其应用场景" src="https://static001.geekbang.org/resource/audio/a8/5b/a8961cb3177d96aa05e48219720c015b.mp3" controls="controls"></audio> 
<p>你好，我是倪朋飞。</p><p>上一讲，我带你一起梳理了 eBPF 程序跟内核交互的基本方法。一个完整的 eBPF 程序通常包含用户态和内核态两部分：用户态程序通过 BPF 系统调用，完成 eBPF 程序的加载、事件挂载以及映射创建和更新，而内核态中的 eBPF 程序则需要通过 BPF 辅助函数完成所需的任务。</p><p>在上一讲中我也提到，并不是所有的辅助函数都可以在 eBPF 程序中随意使用，不同类型的 eBPF 程序所支持的辅助函数是不同的。那么，eBPF 程序都有哪些类型，而不同类型的 eBPF 程序又有哪些独特的应用场景呢？今天，我就带你一起来看看。</p><h2>eBPF 程序可以分成几类？</h2><p>eBPF 程序类型决定了一个 eBPF 程序可以挂载的事件类型和事件参数，这也就意味着，内核中不同事件会触发不同类型的 eBPF 程序。</p><p>根据内核头文件 <a href="https://elixir.bootlin.com/linux/v5.13/source/include/uapi/linux/bpf.h#L908">include/uapi/linux/bpf.h</a> 中  <code>bpf_prog_type</code> 的定义，Linux 内核 v5.13 已经支持 30 种不同类型的 eBPF 程序（注意，&nbsp;<code>BPF_PROG_TYPE_UNSPEC</code>表示未定义）：</p><pre><code class="language-c++">enum bpf_prog_type {
	BPF_PROG_TYPE_UNSPEC, /* Reserve 0 as invalid program type */
	BPF_PROG_TYPE_SOCKET_FILTER,
	BPF_PROG_TYPE_KPROBE,
	BPF_PROG_TYPE_SCHED_CLS,
	BPF_PROG_TYPE_SCHED_ACT,
	BPF_PROG_TYPE_TRACEPOINT,
	BPF_PROG_TYPE_XDP,
	BPF_PROG_TYPE_PERF_EVENT,
	BPF_PROG_TYPE_CGROUP_SKB,
	BPF_PROG_TYPE_CGROUP_SOCK,
	BPF_PROG_TYPE_LWT_IN,
	BPF_PROG_TYPE_LWT_OUT,
	BPF_PROG_TYPE_LWT_XMIT,
	BPF_PROG_TYPE_SOCK_OPS,
	BPF_PROG_TYPE_SK_SKB,
	BPF_PROG_TYPE_CGROUP_DEVICE,
	BPF_PROG_TYPE_SK_MSG,
	BPF_PROG_TYPE_RAW_TRACEPOINT,
	BPF_PROG_TYPE_CGROUP_SOCK_ADDR,
	BPF_PROG_TYPE_LWT_SEG6LOCAL,
	BPF_PROG_TYPE_LIRC_MODE2,
	BPF_PROG_TYPE_SK_REUSEPORT,
	BPF_PROG_TYPE_FLOW_DISSECTOR,
	BPF_PROG_TYPE_CGROUP_SYSCTL,
	BPF_PROG_TYPE_RAW_TRACEPOINT_WRITABLE,
	BPF_PROG_TYPE_CGROUP_SOCKOPT,
	BPF_PROG_TYPE_TRACING,
	BPF_PROG_TYPE_STRUCT_OPS,
	BPF_PROG_TYPE_EXT,
	BPF_PROG_TYPE_LSM,
	BPF_PROG_TYPE_SK_LOOKUP,
};
</code></pre><!-- [[[read_end]]] --><p>对于具体的内核来说，因为不同内核的版本和编译配置选项不同，一个内核并不会支持所有的程序类型。你可以在命令行中执行下面的命令，来查询当前系统支持的程序类型：</p><pre><code class="language-plain">bpftool feature probe | grep program_type
</code></pre><p>执行后，你会得到如下的输出：</p><pre><code class="language-plain">eBPF program_type socket_filter is available
eBPF program_type kprobe is available
eBPF program_type sched_cls is available
eBPF program_type sched_act is available
eBPF program_type tracepoint is available
eBPF program_type xdp is available
eBPF program_type perf_event is available
...
eBPF program_type ext is NOT available
eBPF program_type lsm is NOT available
eBPF program_type sk_lookup is available
</code></pre><p>在这些输出中，你可以看到当前内核支持 kprobe、xdp、perf_event 等程序类型，而不支持 ext、lsm 等程序类型。</p><p>根据具体功能和应用场景的不同，这些程序类型大致可以划分为三类：</p><ul>
<li>第一类是跟踪，即从内核和程序的运行状态中提取跟踪信息，来了解当前系统正在发生什么。</li>
<li>第二类是网络，即对网络数据包进行过滤和处理，以便了解和控制网络数据包的收发过程。</li>
<li>第三类是除跟踪和网络之外的其他类型，包括安全控制、BPF 扩展等等。</li>
</ul><p>接下来，我就带你一起分别看看，每一类 eBPF 程序都有哪些具体的类型，以及这些不同类型的程序都是由哪些事件触发执行的。</p><h2>跟踪类 eBPF 程序</h2><p>先看第一类，也就是跟踪类 eBPF 程序。</p><p><strong>跟踪类 eBPF 程序主要用于从系统中提取跟踪信息，进而为监控、排错、性能优化等提供数据支撑。</strong>比如，我们前几讲中的 Hello World 示例就是一个 <code>BPF_PROG_TYPE_KPROBE</code> 类型的跟踪程序，它的目的是跟踪内核函数是否被某个进程调用了。</p><p>为了方便你查询，我把常见的跟踪类 BPF 程序的主要功能以及使用限制整理成了一个表格，你可以在需要时参考。</p><p><img src="https://static001.geekbang.org/resource/image/04/38/042fe319b51yy6bc153ce0f877f54a38.jpg?wh=1920x1098" alt="图片"></p><p>这其中，KPROBE、TRACEPOINT 以及 PERF_EVENT 都是最常用的 eBPF 程序类型，大量应用于监控跟踪、性能优化以及调试排错等场景中。我们前几讲中提到的 <a href="https://github.com/iovisor/bcc">BCC</a>工具集，其中包含的绝大部分工具也都属于这个类型。</p><h2>网络类 eBPF 程序</h2><p>看完跟踪类 eBPF 程序，我们再来看看网络类 eBPF 程序。</p><p><strong>网络类 eBPF 程序主要用于对网络数据包进行过滤和处理，进而实现网络的观测、过滤、流量控制以及性能优化等各种丰富的功能。</strong>根据事件触发位置的不同，网络类 eBPF 程序又可以分为 XDP（eXpress Data Path，高速数据路径）程序、TC（Traffic Control，流量控制）程序、套接字程序以及 cgroup 程序，下面我们来分别看看。</p><h3><strong>XDP 程序</strong></h3><p>XDP 程序的类型定义为 <code>BPF_PROG_TYPE_XDP</code>，它在<strong>网络驱动程序刚刚收到数据包时</strong>触发执行。由于无需通过繁杂的内核网络协议栈，XDP 程序可用来实现高性能的网络处理方案，常用于 DDoS 防御、防火墙、4 层负载均衡等场景。</p><p>你需要注意，XDP 程序并不是绕过了内核协议栈，它只是在内核协议栈之前处理数据包，而处理过的数据包还可以正常通过内核协议栈继续处理。你可以通过下面的图片（图片来自 <a href="https://www.iovisor.org/technology/xdp">iovisor.org</a>）加深对&nbsp;XDP 相对内核协议栈位置的理解：</p><p><img src="https://static001.geekbang.org/resource/image/3b/31/3b77fea948d6264bfb4b4c266526dd31.png?wh=768x420" alt="图片" title="XDP概览"></p><p>根据网卡和网卡驱动是否原生支持 XDP 程序，XDP 运行模式可以分为下面这三种：</p><ul>
<li>通用模式。它不需要网卡和网卡驱动的支持，XDP 程序像常规的网络协议栈一样运行在内核中，性能相对较差，一般用于测试；</li>
<li>原生模式。它需要网卡驱动程序的支持，XDP 程序在网卡驱动程序的早期路径运行；</li>
<li>卸载模式。它需要网卡固件支持 XDP 卸载，XDP 程序直接运行在网卡上，而不再需要消耗主机的 CPU 资源，具有最好的性能。</li>
</ul><p>无论哪种模式，XDP 程序在处理过网络包之后，都需要根据 eBPF 程序执行结果，决定数据包的去处。这些执行结果对应以下 5 种 XDP 程序结果码：</p><p><img src="https://static001.geekbang.org/resource/image/a2/a7/a2cayy9f21129590a91ca07604b070a7.jpg?wh=1920x1237" alt="图片"></p><p>通常来说，XDP 程序通过 <code>ip link</code> 命令加载到具体的网卡上，加载格式为：</p><pre><code class="language-bash"># eth1 为网卡名
# xdpgeneric 设置运行模式为通用模式
# xdp-example.o 为编译后的 XDP 字节码
sudo ip link set dev eth1 xdpgeneric object xdp-example.o
</code></pre><p>而卸载 XDP 程序也是通过 <code>ip link</code> 命令，具体参数如下：</p><pre><code class="language-plain">sudo ip link set veth1 xdpgeneric off
</code></pre><p>除了 <code>ip link</code>之外， BCC 也提供了方便的库函数，让我们可以在同一个程序中管理 XDP 程序的生命周期：</p><pre><code class="language-python">from bcc import BPF

# 编译XDP程序
b = BPF(src_file="xdp-example.c")
fn = b.load_func("xdp-example", BPF.XDP)

# 加载XDP程序到eth0网卡
device = "eth0"
b.attach_xdp(device, fn, 0)

# 其他处理逻辑
...

# 卸载XDP程序
b.remove_xdp(device)
</code></pre><h3><strong>TC 程序</strong></h3><p>TC 程序的类型定义为 <code>BPF_PROG_TYPE_SCHED_CLS</code> 和 <code>BPF_PROG_TYPE_SCHED_ACT</code>，分别作为 <a href="https://tldp.org/HOWTO/Traffic-Control-HOWTO/index.html">Linux 流量控制</a> 的分类器和执行器。Linux 流量控制通过网卡队列、排队规则、分类器、过滤器以及执行器等，实现了对网络流量的整形调度和带宽控制。</p><p>下图（图片来自 <a href="http://linux-ip.net/articles/Traffic-Control-HOWTO/">linux-ip.net</a>）展示了&nbsp;HTB（Hierarchical Token Bucket，层级令牌桶）流量控制的工作原理：</p><p><img src="https://static001.geekbang.org/resource/image/3c/69/3c445830476ed2f32d71e99309b26369.png?wh=1369x1049" alt="图片" title="HTB 流量控制"></p><p>由于 Linux 流量控制并非本课程的重点，这里我就不过多展开了。如果你对它还不熟悉，可以参考&nbsp;<a href="https://tldp.org/HOWTO/Traffic-Control-HOWTO/index.html">官方文档</a>&nbsp;进行学习。</p><p>得益于内核 v4.4 引入的 <a href="https://docs.cilium.io/en/v1.8/bpf/#tc-traffic-control">direct-action</a> 模式，TC 程序可以直接在一个程序内完成分类和执行的动作，而无需再调用其他的 TC 排队规则和分类器，具体如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/31/d5/31ecf04f2477bd4765be9544a62deed5.jpg?wh=1920x888" alt="图片" title="TC eBPF 程序与网络协议栈的关系"></p><p>同 XDP 程序相比，TC 程序可以<strong>直接获取内核解析后的网络报文数据结构</strong><code>sk_buff</code>（XDP 则是 <code>xdp_buff</code>），并且可<strong>在网卡的接收和发送两个方向上执行</strong>（XDP 则只能用于接收）。下面我们来具体看看&nbsp;TC 程序的执行位置：</p><ul>
<li>对于接收的网络包，TC 程序在网卡接收（GRO）之后、协议栈处理（包括 IP 层处理和 iptables 等）之前执行；</li>
<li>对于发送的网络包，TC 程序在协议栈处理（包括 IP 层处理和 iptables 等）之后、数据包发送到网卡队列（GSO）之前执行。</li>
</ul><p>除此之外，由于 TC 运行在内核协议栈中，不需要网卡驱动程序做任何改动，因而可以挂载到任意类型的网卡设备（包括容器等使用的虚拟网卡）上。</p><p>同 XDP 程序一样，TC eBPF 程序也可以通过 Linux 命令行工具来加载到网卡上，不过相应的工具要换成 <code>tc</code>。你可以通过下面的命令，分别加载接收和发送方向的 eBPF 程序：</p><pre><code class="language-bash"># 创建 clsact 类型的排队规则
sudo tc qdisc add dev eth0 clsact

# 加载接收方向的 eBPF 程序
sudo tc filter add dev eth0 ingress bpf da obj tc-example.o sec ingress

# 加载发送方向的 eBPF 程序
sudo tc filter add dev eth0 egress bpf da obj tc-example.o sec egress
</code></pre><h3><strong>套接字程序</strong></h3><p>套接字程序用于过滤、观测或重定向套接字网络包，具体的种类也比较丰富。根据类型的不同，套接字 eBPF 程序可以挂载到套接字（socket）、控制组（cgroup ）以及网络命名空间（netns）等各个位置。你可以根据具体的应用场景，选择一个或组合多个类型的 eBPF 程序，去控制套接字的网络包收发过程。</p><p>这里，我把常见的套接字程序类型，以及它们的应用场景和挂载方法整理成了一个表格，你可以在需要时参考：</p><p><img src="https://static001.geekbang.org/resource/image/0e/44/0e57bf041262114198fd29e1e5c04044.jpg?wh=1920x1566" alt="图片"></p><h3><strong>cgroup 程序</strong></h3><p>cgroup 程序用于<strong>对 cgroup 内所有进程的网络过滤、套接字选项以及转发等进行动态控制</strong>，它最典型的应用场景是对容器中运行的多个进程进行网络控制。</p><p>cgroup 程序的种类比较丰富，我也帮你整理了一个表格，方便你在需要时查询：</p><p><img src="https://static001.geekbang.org/resource/image/c3/52/c37b8849096311726e734e8549fd9452.jpg?wh=1920x1216" alt="图片"></p><p>这些类型的 BPF 程序都可以通过 BPF 系统调用的 <code>BPF_PROG_ATTACH</code> 命令来进行挂载，并设置<a href="https://elixir.bootlin.com/linux/v5.13/source/include/uapi/linux/bpf.h#L942">挂载类型</a>为匹配的 <code>BPF_CGROUP_xxx</code> 类型。比如，在挂载 <code>BPF_PROG_TYPE_CGROUP_DEVICE</code> 类型的 BPF 程序时，需要设置 <code>bpf_attach_type</code> 为 <code>BPF_CGROUP_DEVICE</code>：</p><pre><code class="language-c++">union bpf_attr attr = {};
attr.target_fd = target_fd;            // cgroup文件描述符
attr.attach_bpf_fd = prog_fd;          // BPF程序文件描述符
attr.attach_type = BPF_CGROUP_DEVICE;  // 挂载类型为BPF_CGROUP_DEVICE

if (bpf(BPF_PROG_ATTACH, &amp;attr, sizeof(attr)) &lt; 0) {
  return -errno;
}

...
</code></pre><p>注意，这几类网络 eBPF 程序是在不同的事件触发时执行的，因此，在实际应用中我们通常可以把多个类型的 eBPF 程序结合起来，一起使用，来实现复杂的网络控制功能。比如，最流行的 Kubernetes 网络方案 Cilium 就大量使用了 XDP、TC 和套接字 eBPF 程序，如下图（图片来自 Cilium <a href="https://docs.cilium.io/en/v1.11/concepts/ebpf/lifeofapacket/">官方文档</a>，图中黄色部分即为 Cilium eBPF 程序）所示：</p><p><img src="https://static001.geekbang.org/resource/image/45/13/452c809d6e3335fb933a8f991eedf113.png?wh=1454x714" alt="图片" title="Cilium eBPF 数据面"></p><h2>其他类 eBPF 程序</h2><p>除了上面的跟踪和网络 eBPF 程序之外，Linux 内核还支持很多其他的类型。这些类型的 eBPF 程序虽然不太常用，但在需要的时候也可以帮你解决很多特定的问题。</p><p>我将这些无法划分到网络和跟踪的 eBPF 程序都归为其他类，并帮你整理了一个表格：</p><p><img src="https://static001.geekbang.org/resource/image/93/f2/93ae17801e82579e07937e5f1595a0f2.jpg?wh=1920x1332" alt="图片"></p><p>这个表格列出了一些不太常用的 eBPF 程序类型，你可以先大致浏览下，在需要的时候再去深入了解。</p><h2>小结</h2><p>今天，我带你一起梳理了 eBPF 程序的主要类型，以及不同类型 eBPF 程序的应用场景。</p><p>根据具体功能和应用场景的不同，我们可以把 eBPF 程序分为跟踪、网络和其他三类：</p><ul>
<li>跟踪类 eBPF 程序主要用于从系统中提取跟踪信息，进而为监控、排错、性能优化等提供数据支撑；</li>
<li>网络类 eBPF 程序主要用于对网络数据包进行过滤和处理，进而实现网络的观测、过滤、流量控制以及性能优化等；</li>
<li>其他类则包含了跟踪和网络之外的其他&nbsp;eBPF&nbsp;程序类型，如安全控制、BPF 扩展等。</li>
</ul><p>虽然每个 eBPF 程序都有特定的类型和触发事件，但这并不意味着它们都是完全独立的。通过 BPF 映射提供的状态共享机制，各种不同类型的 eBPF 程序完全可以相互配合，不仅可以绕过单个 eBPF 程序指令数量的限制，还可以实现更为复杂的控制逻辑。</p><h2>思考题</h2><p>最后，我想邀请你来聊一聊：</p><ol>
<li>你是怎么理解 eBPF 程序类型的呢？</li>
<li>如果让你来重新设计类似于 Cilium 的网络方案，你会如何选择 eBPF 程序类型呢？</li>
</ol><p>期待你在留言区和我讨论，也欢迎把这节课分享给你的同事、朋友。让我们一起在实战中演练，在交流中进步。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2a/d4/99/594e35c2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>YF</span>
  </div>
  <div class="_2_QraFYR_0">你是怎么理解 eBPF 程序类型的呢？<br><br>eBPF 对应与内核的事件类型，犹如订阅同类消息事件，内核发现对应的事件，则通知订阅者处理。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍非常棒！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-30 19:12:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/87/06/bedf7932.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>阿立</span>
  </div>
  <div class="_2_QraFYR_0">作者大佬，xdp是一个内核网络处理模块，还是网络包进入协议栈之前的事件钩子点呢？ tc呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 都是网络包事件钩子，当然也是网络处理流程中的一部分。XDP和TC其实有些类似，但TC总是在内核中执行，而XDP可以卸载到网卡硬件，从而获得更好的性能。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-10 16:42:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/9c/f3/e01dfe3a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>于競</span>
  </div>
  <div class="_2_QraFYR_0">看了上面介绍的网络类型的eBPF程序，好奇可不可以用来开发流量复制工具呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯嗯 可以的。但实际上只是流量复制的话，并不需要eBPF，无论是网络硬件还是OVS、TC等已有的Linux工具早就支持流量复制了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-23 01:08:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/b7/24/17f6c240.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>janey</span>
  </div>
  <div class="_2_QraFYR_0">我理解ebpf分类，可以分为tracepoint,kprobe,uprobe,USDT等</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果按照内核中定义的bpf_prog_type，它的种类是非常多的（5.13已经支持30种了）。所以文中把这些类型又划分了大类，方便在需要的时候参考。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-24 08:15:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/23/73/de/2d9bc244.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>彬</span>
  </div>
  <div class="_2_QraFYR_0">老师后续会有更全的环境搭建吗？比如升级内核，如果不能直接apt安装bcc等需要采取源码编译等情况，如何在本地环境中编译然后发布到生产环境等情况讲解或者资料</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-13 15:25:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/ed/1a/ce7f7d54.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>asdf100</span>
  </div>
  <div class="_2_QraFYR_0">怎么知道网卡是否支持XDP？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-29 16:43:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/91/21/d38d8391.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>任智杰byte</span>
  </div>
  <div class="_2_QraFYR_0">如何扩展程序类型呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 增加新的类型就需要修改内核源码了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-11 19:38:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/76/b0/14fec62f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>不了峰</span>
  </div>
  <div class="_2_QraFYR_0">打卡</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 加油！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-28 14:54:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/24/b1/9462135f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>重庆杜员外</span>
  </div>
  <div class="_2_QraFYR_0">想要用ebpf实现一个http流量分析和统计的旁路系统，线上内核版本5.4，请问老师有什么好的建议吗</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-24 11:33:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/31/5a/e3/cfc48236.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>白璐</span>
  </div>
  <div class="_2_QraFYR_0">老师，ebpf 可以阻断进程的行为吗，例如某个进程调用了write 系统调用 然后 我应用ebpf阻断调用 不让这个进程调用write系统调用 。 这个可以用ebpf实现吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 应该可以借助LSM BPF程序实现，你可以参考一下https:&#47;&#47;www.kernel.org&#47;doc&#47;html&#47;v5.9&#47;bpf&#47;bpf_lsm.html</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-20 16:47:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/0b/6e/e0409a35.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>XYS</span>
  </div>
  <div class="_2_QraFYR_0">原理：钩子就是系统在各个数据处理路径上放置的一个埋点（例如：map[int]func）,我们需要确定int值（枚举类型）和处理函数，处理路径上会查看map[特定值]下是否有函数，有就执行。<br>应用：熟悉各个枚举值的位置（既作用）就能在实际工作中利用该技术解决实际问题。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢分享👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-11 10:43:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/70/ab/7a7399a7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>tj.•</span>
  </div>
  <div class="_2_QraFYR_0">老师，使用cilium代替iptables之后，pod访问一个service ip，是在哪个hook做DNAT成endpoint pod ip的？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-27 16:36:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2c/7b/7a/70c904e9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Lime</span>
  </div>
  <div class="_2_QraFYR_0">ePBF有没有办法修改内核传进来的参数</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-13 14:21:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2d/b1/60/23c03681.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>一</span>
  </div>
  <div class="_2_QraFYR_0">老师，我想问一下，比如我想在系统弹窗时插装，怎么找到系统弹窗这个hook点呢，比如SEC（“uprobe&#47;这个地方应该填什么呢，或者说在哪里有这个uprobe下面hook点的列表呢”）</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-02 21:52:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/19/1d/d2b6e006.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>火火寻</span>
  </div>
  <div class="_2_QraFYR_0">老师，问下，如果想要做业务层的流量控制，比如dubbo请求部分黑名单不让进来，使用哪种合适？<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个可能直接用dubbo会比ebpf要更简单。ebpf虽然也可以用，但毕竟运行在内核中，实现复杂度会高一些</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-19 23:05:18</div>
  </div>
</div>
</div>
</li>
</ul>