<audio title="10 _ 网络跟踪：如何使用eBPF排查网络问题？" src="https://static001.geekbang.org/resource/audio/24/d4/24f461c1d2420b39cca5d794e00f1dd4.mp3" controls="controls"></audio> 
<p>你好，我是倪朋飞。</p><p>上一讲，我带你一起梳理了应用进程的跟踪方法。根据编程语言的不同，从应用程序二进制文件的跟踪点中可以获取的信息也不同：编译型语言程序中的符号信息可以直接拿来跟踪应用的执行状态，而对于解释型语言和即时编译型语言程序，我们只能从解释器或即时编译器的跟踪点参数中去获取应用的执行状态。</p><p>除了前面三讲提到的内核和应用程序的跟踪之外，网络不仅是 eBPF 应用最早的领域，也是目前 eBPF 应用最为广泛的一个领域。随着分布式系统、云计算和云原生应用的普及，网络已经成为了大部分应用最核心的依赖，随之而来的网络问题也是最难排查的问题之一。</p><p>那么，eBPF 能不能协助我们更好地排查和定位网络问题呢？我们又该如何利用 eBPF 来排查网络相关的问题呢？今天，我就带你一起具体看看。</p><h2>eBPF 提供了哪些网络功能？</h2><p>既然想要使用 eBPF 排查网络问题，我想进入你头脑的第一个问题就是：eBPF 到底提供了哪些网络相关的功能框架呢？</p><p>要回答这个问题，首先要理解 Linux 网络协议栈的基本原理。下面是一个简化版的内核协议栈示意图：</p><p><img src="https://static001.geekbang.org/resource/image/ce/a5/ce9371b7e3b696feaf4233c1595e20a5.jpg?wh=1920x2960" alt="图片" title="Linux内核协议栈"></p><p>如何理解这个网络栈示意图呢？从上往下看，就是应用程序发送网络包的过程；而反过来，从下往上看，则是网络包接收的过程。比如，从上到下来看这个网络栈，你可以发现：</p><!-- [[[read_end]]] --><ul>
<li>应用程序会通过系统调用来跟套接字接口进行交互；</li>
<li>套接字的下面，就是内核协议栈中的传输层、网络层和网络接口层，这其中也包含了网络过滤（Netfilter）和流量控制（Traffic Control）框架；</li>
<li>最底层，则是网卡驱动程序以及物理网卡设备。</li>
</ul><p>理解了网络协议栈的基本流程，eBPF 提供的网络功能也就容易理解了。如下图所示，eBPF 实际上提供了贯穿整个网络协议栈的过滤、捕获以及重定向等丰富的网络功能：</p><p><img src="https://static001.geekbang.org/resource/image/80/50/80c2a8fe3b36ee2fb033b4332431f750.jpg?wh=2284x2197" alt="" title="内核协议栈 BPF 挂载点"></p><p>一方面，网络协议栈也是内核的一部分，因而<strong>网络相关的内核函数、跟踪点以及用户程序的函数等，也都可以使用前几讲我们提到的 kprobe、uprobe、USDT 等跟踪类 eBPF 程序进行跟踪</strong>（如上图中紫色部分所示）。</p><p>另一方面，回顾一下 <a href="https://time.geekbang.org/column/article/483364">06 讲</a> 的内容，<strong>eBPF 提供了大量专用于网络的 eBPF 程序类型，包括 XDP 程序、TC 程序、套接字程序以及 cgroup 程序等</strong>。这些类型的程序涵盖了从网卡（如卸载到硬件网卡中的 XDP 程序）到网卡队列（如 TC 程序）、封装路由（如轻量级隧道程序）、TCP 拥塞控制、套接字（如 sockops 程序）等内核协议栈，再到同属于一个 cgroup 的一组进程的网络过滤和控制，而这些都是内核协议栈的核心组成部分（如上图中绿色部分所示）。</p><p>从这两个方面，你可以发现 eBPF 已经涵盖了内核协议栈的很多方面，并且内核社区在网络方面的功能还在不断丰富中。</p><p>接下来，我就以最常见的网络丢包为例，带你看看如何使用 eBPF 来排查网络问题。</p><h2>如何跟踪内核网络协议栈？</h2><p>即使理解了内核协议栈的基本原理，以及各种类型 eBPF 程序的基本功能，在想要跟踪网络相关的问题时，你可能还是觉得无从下手，这是为什么呢？</p><p>究其原因，我认为最主要是因为<strong>不清楚内核中都有哪些函数和跟踪点可以拿来跟踪</strong>。而即使通过源码查询到了一系列的内核函数，还是没有一个清晰的思路把这些内核函数与所碰到的网络问题关联起来。</p><p>不过，当你碰到这类困惑的时候不要有畏难心理。要知道，所有非内核开发者都会碰到类似的问题，而解决这类问题也并非难事，下面我们就来看解决方法。</p><p>如何把内核函数跟相关的网络问题关联起来呢？看到本小节的标题，你应该已经想到了：跟踪调用栈，<strong>根据调用栈回溯路径</strong><strong>，</strong><strong>找出导致某个网络事件发生的整个流程，进而就可以再根据这些流程中的内核函数进一步跟踪</strong>。</p><p>既然是调用栈的<strong>回溯</strong>，只有我们知道了最接近整个执行逻辑结尾的函数，才有可能开始这个回溯过程。对 Linux 网络丢包问题来说，内核协议栈执行的结尾，当然就是释放最核心的 SKB（Socket Buffer）数据结构。查询内核 <a href="https://www.kernel.org/doc/htmldocs/networking/ch01s02.html">SKB 文档</a>，你可以发现，内核中释放 SKB 相关的函数有两个：</p><ul>
<li>第一个，<a href="https://www.kernel.org/doc/htmldocs/networking/API-kfree-skb.html">kfree_skb</a> ，它经常在网络异常丢包时调用；</li>
<li>第二个，<a href="https://www.kernel.org/doc/htmldocs/networking/API-consume-skb.html">consume_skb</a> ，它在正常网络连接完成时调用。</li>
</ul><p>这两个函数除了使用场景的不同，其功能和实现流程都是一样的，即都是检查 SKB 的引用计数，当引用计数为 0 时释放其内核内存。所以，要跟踪网络丢包的执行过程，也就可以跟踪 <code>kfree_skb</code> 的内核调用栈。</p><p><strong>接下来，我就以访问极客时间的网站 <strong><code>time.geekbang.org</code></strong> 为例，来带你一起看看</strong><strong>，</strong><strong>如何使用 bpftrace 来进行调用栈的跟踪。</strong></p><p>为了方便调用栈的跟踪，bpftrace 提供了 <a href="https://github.com/iovisor/bpftrace/blob/master/docs/reference_guide.md#7-kstack-stack-traces-kernel">kstack</a> 和 <a href="https://github.com/iovisor/bpftrace/blob/master/docs/reference_guide.md#8-ustack-stack-traces-user">ustack</a> 这两个内置变量，分别用于获取内核和进程的调用栈。打开一个终端，执行下面的命令就可以跟踪 <code>kfree_skb</code> 的内核调用栈了：</p><pre><code class="language-plain">sudo bpftrace -e 'kprobe:kfree_skb /comm=="curl"/ {printf("kstack: %s\n", kstack);}'
</code></pre><p>这个命令中的具体内容含义如下：</p><ul>
<li><code>kprobe:kfree_skb</code> 指定跟踪的内核函数为 <code>kfree_skb</code>；</li>
<li>紧随其后的 <code>/comm=="curl"/</code> ，表示只跟踪 <code>curl</code> 进程，这是为了过滤掉其他不相关的进程操作；</li>
<li>最后的 <code>printf()</code> 函数就是把内核协议栈打印到终端中。</li>
</ul><p>打开一个新终端，并在终端中执行 <code>curl time.geekbang.org</code> 命令，然后回到第一个终端，就可以看到如下的输出：</p><pre><code class="language-c++">kstack:
        kfree_skb+1
        udpv6_destroy_sock+66
        sk_common_release+34
        udp_lib_close+9
        inet_release+75
        inet6_release+49
        __sock_release+66
        sock_close+21
        __fput+159
        ____fput+14
        task_work_run+103
        exit_to_user_mode_loop+411
        exit_to_user_mode_prepare+187
        syscall_exit_to_user_mode+23
        do_syscall_64+110
        entry_SYSCALL_64_after_hwframe+68

kstack:
        kfree_skb+1
        udpv6_destroy_sock+66
        sk_common_release+34
        udp_lib_close+9
        inet_release+75
        inet6_release+49
        __sock_release+66
        sock_close+21
        __fput+159
        ____fput+14
        task_work_run+103
        exit_to_user_mode_loop+411
        exit_to_user_mode_prepare+187
        syscall_exit_to_user_mode+23
        do_syscall_64+110
        entry_SYSCALL_64_after_hwframe+68

kstack:
        kfree_skb+1
        unix_release+29
        __sock_release+66
        sock_close+21
        __fput+159
        ____fput+14
        task_work_run+103
        exit_to_user_mode_loop+411
        exit_to_user_mode_prepare+187
        syscall_exit_to_user_mode+23
        do_syscall_64+110
        entry_SYSCALL_64_after_hwframe+68

kstack:
        kfree_skb+1
        __sys_connect_file+95
        __sys_connect+162
        __x64_sys_connect+24
        do_syscall_64+97
        entry_SYSCALL_64_after_hwframe+68

kstack:
        kfree_skb+1
        __sys_connect_file+95
        __sys_connect+162
        __x64_sys_connect+24
        do_syscall_64+97
        entry_SYSCALL_64_after_hwframe+68
</code></pre><p>这个输出包含了多个调用栈，每个调用栈从下往上就是 <code>kfree_skb</code> 被调用过程中的各个函数（函数名后的数字表示调用点相对函数地址的偏移），它们都是从系统调用（<code>entry_SYSCALL_64</code>）开始，通过一系列的内核函数之后，最终调用到了跟踪函数。</p><p>输出中包含多个调用栈，是因为同一个内核函数是有可能在多个地方调用的。因此，我们需要对它进一步改进，加上网络信息的过滤，并把源 IP 和目的 IP 等基本信息也打印出来。比如，我们访问一个网址，只需要关心 TCP 协议，而其他协议相关的内核栈就可以忽略掉。</p><p><code>kfree_skb</code> 函数的定义格式如下所示，它包含一个 <code>struct sk_buff</code> 类型的参数，这样我们就可以从中获取协议、源 IP 和目的 IP 等基本信息：</p><pre><code class="language-c++">void kfree_skb(struct sk_buff * skb);
</code></pre><p>由于我们需要添加数据结构读取的过程，为了更好的可读性，你可以把这些过程放入一个脚本文件中，通常后缀为 <code>.bt</code>。下面就是一个改进了的跟踪程序：</p><pre><code class="language-c++">kprobe:kfree_skb /comm=="curl"/
{
  // 1. 第一个参数是 struct sk_buff
  $skb = (struct sk_buff *)arg0;

  // 2. 从网络头中获取源IP和目的IP
  $iph = (struct iphdr *)($skb-&gt;head + $skb-&gt;network_header);
  $sip = ntop(AF_INET, $iph-&gt;saddr);
  $dip = ntop(AF_INET, $iph-&gt;daddr);

  // 3. 只处理TCP协议
  if ($iph-&gt;protocol == IPPROTO_TCP)
  {
    // 4. 打印源IP、目的IP和内核调用栈
    printf("SKB dropped: %s-&gt;%s, kstack: %s\n", $sip, $dip, kstack);
  }
}
</code></pre><p>让我们来看看这个脚本中每一处的具体含义：</p><ul>
<li>第1处是把 bpftrace 的内置参数 <code>arg0</code> 转换成 SKB 数据结构 <code>struct sk_buff *</code>（注意使用指针）。</li>
<li>第2处是从 SKB 数据结构中获取网络头之后，再从中拿到源IP和目的IP，最后再调用内置函数 <code>ntop()</code> ，把整数型的 IP 数据结构转换为可读性更好的字符串格式。</li>
<li>第3处是对网络协议进行了过滤，只保留TCP协议。</li>
<li>第4处是向终端中打印刚才获取的源IP和目的IP，同时也打印内核调用栈。</li>
</ul><p>直接把这个脚本保存到文件中，bpftrace 并不能直接运行。你会看到如下的类型未知错误：</p><pre><code class="language-plain">./dropwatch.bt:9:10-28: ERROR: Unknown struct/union: 'struct sk_buff'
  $skb = (struct sk_buff *)arg0;
         ~~~~~~~~~~~~~~~~~~
./dropwatch.bt:12:10-26: ERROR: Unknown struct/union: 'struct iphdr'
  $iph = (struct iphdr *)($skb-&gt;head + $skb-&gt;network_header);
         ~~~~~~~~~~~~~~~~
./dropwatch.bt:13:10-22: ERROR: Unknown identifier: 'AF_INET'
  $sip = ntop(AF_INET, $iph-&gt;saddr);
         ~~~~~~~~~~~~
</code></pre><p>这是因为，bpftrace 在将上述脚本编译为 BPF 字节码的过程中，找不到这些类型的定义。在内核支持 BTF 之前，这其实是所有 eBPF 程序开发过程中都会遇到的一个问题。要解决这个问题，我们就需要把所需的内核头文件引入进来。</p><p>这里给你一个小提示：v0.9.3 或更新版本的 bpftrace 已经支持 BTF 了，但需要新版本的 libbpf，且还有很多的<a href="https://github.com/iovisor/bpftrace/blob/master/docs/reference_guide.md#btf-support">限制</a>。</p><p>那么，如何找出这些数据结构的头文件呢？我通常使用下面的两种方法：</p><ul>
<li>第一种方法是在内核源码目录中，通过查找的方式，找出定义了这些数据结构的头文件（后缀为 <code>.h</code>）。</li>
<li>另外一种方法是到 <a href="https://elixir.bootlin.com/">https://elixir.bootlin.com/</a> 上选择正确的内核版本后，再搜索数据结构的名字。我在下面还会详细介绍这个网站的使用方法。</li>
</ul><p>我们在脚本文件中加入这些类型定义的头文件：</p><pre><code class="language-c++">#include &lt;linux/skbuff.h&gt;
#include &lt;linux/ip.h&gt;
#include &lt;linux/socket.h&gt;
#include &lt;linux/netdevice.h&gt;
</code></pre><p>然后，保存到文件 <code>dropwatch.bt</code>中（你也可以在 <a href="https://github.com/feiskyer/ebpf-apps/blob/main/bpftrace/dropwatch.bt">GitHub</a> 中找到完整的程序），就可以通过 <code>sudo bpftrace dropwatch.bt</code> 来运行了。</p><h2>如何排查网络丢包问题？</h2><p>有了 eBPF 跟踪脚本之后，它能不能用来排查网络丢包问题呢？我们来验证一下。</p><p>最常见的丢包是由系统防火墙阻止了相应的 IP 或端口导致的，你可以执行下面的 <code>nslookup</code>命令，查询到极客时间的 IP 地址，然后再执行<code>iptables</code> 命令，禁止访问极客时间的 80 端口：</p><pre><code class="language-plain"># 首先查询极客时间的IP
$ nslookup time.geekbang.org
Server:        127.0.0.53
Address:    127.0.0.53#53

Non-authoritative answer:
Name:    time.geekbang.org
Address: 39.106.233.176

# 然后增加防火墙规则阻止80端口
$ sudo iptables -I OUTPUT -d 39.106.233.176/32 -p tcp -m tcp --dport 80 -j DROP
</code></pre><p>防火墙规则加好之后，在终端一中启动跟踪脚本：</p><pre><code class="language-plain">sudo bpftrace dropwatch.bt
</code></pre><p>然后，新建一个终端，访问极客时间，你应该会看到超时的错误：</p><pre><code class="language-plain">$ curl --connect-timeout 1 39.106.233.176
curl: (28) Connection timed out after 1000 milliseconds
</code></pre><p>返回第一个终端，你就可以看到 eBPF 程序已经成功跟踪到了内核丢包的调用栈信息，如下所示：</p><pre><code class="language-plain">SKB dropped: 192.168.1.129-&gt;39.106.233.176, kstack:
        kfree_skb+1
        __ip_local_out+219
        ip_local_out+29
        __ip_queue_xmit+367
        ip_queue_xmit+21
        __tcp_transmit_skb+2237
        tcp_connect+1009
        tcp_v4_connect+951
        __inet_stream_connect+206
        inet_stream_connect+59
        __sys_connect_file+95
        __sys_connect+162
        __x64_sys_connect+24
        do_syscall_64+97
        entry_SYSCALL_64_after_hwframe+68
</code></pre><p>从这个输出中，我们可以看到，第一行输出中我们成功拿到了源 IP 和目的 IP，而接下来的每一行中都包含了指令地址、函数名以及函数地址偏移。</p><p>从下往上看这个调用栈，最后调用 <code>kfree_skb</code> 函数的是 <code>__ip_local_out</code>，那么 <code>__ip_local_out</code> 这个函数又是干什么的呢？根据函数名，你可以大致猜测出，它是用于向外发送网络包的，但具体的步骤我们就不太确定了。所以，这时候就需要去参考一下内核源代码。</p><p>这里推荐你使用 <a href="https://elixir.bootlin.com/">https://elixir.bootlin.com/</a> 这个网站来查看内核源码，因为它不仅列出了所有内核版本的源代码，还提供了交叉引用的功能。在源码文件中点击任意函数或类型，它就可以自动跳转到其定义和引用的位置。</p><p>比如，对于 <code>__ip_local_out</code> 函数的定义和引用，就可以通过 <a href="https://elixir.bootlin.com/linux/v5.13/A/ident/__ip_local_out">https://elixir.bootlin.com/linux/v5.13/A/ident/__ip_local_out</a> （请注意把 v5.13 替换成你的内核版本）这个网址来查看。点击<a href="https://elixir.bootlin.com/linux/v5.13/A/ident/__ip_local_out">链接</a>，你会看到如下的界面：</p><p><img src="https://static001.geekbang.org/resource/image/98/cc/98891ed6bf8f2d6e273299bba6a3edcc.png?wh=856x614" alt="图片" title="交叉引用搜索结果示意图"></p><p>查询的结果分为三个部分，分别是头文件中的函数声明、源码文件中的函数定义，以及其他文件的引用。点击中间部分（即上图红框中的第一个链接），就可以跳转到源码的定义位置。</p><p>打开<a href="https://elixir.bootlin.com/linux/v5.13/source/net/ipv4/ip_output.c#L99">代码</a>之后，你会发现，其实并不需要因为不懂内核而担心自己看不懂内核源码，内核中的源码还是很简洁的。这里我把原始代码复制了过来，并且加入了详细的注释，如下所示：</p><pre><code class="language-c++">int __ip_local_out(struct net *net, struct sock *sk, struct sk_buff *skb)
{
    struct iphdr *iph = ip_hdr(skb);

  /* 计算总长度 */
    iph-&gt;tot_len = htons(skb-&gt;len);
  /* 计算校验和 */
    ip_send_check(iph);

    /* L3主设备处理 */
    skb = l3mdev_ip_out(sk, skb);
    if (unlikely(!skb))
        return 0;

  /* 设置IP协议 */
    skb-&gt;protocol = htons(ETH_P_IP);

  /* 调用NF_INET_LOCAL_OUT钩子 */
    return nf_hook(NFPROTO_IPV4, NF_INET_LOCAL_OUT,
               net, sk, skb, NULL, skb_dst(skb)-&gt;dev,
               dst_output);
}
</code></pre><p>从这个代码来看，<code>__ip_local_out</code> 函数的主要流程就是计算总长度和校验和，再设置 L3 主设备和协议等属性后，最终调用 <code>nf_hook</code>。</p><p>而 <code>nf</code> 就是 netfilter 的缩写，所以你就可以将其理解为调用 iptables 规则。再根据 <code>NF_INET_LOCAL_OUT</code>参数，你就可以知道接下来调用了 OUTPUT 链（chain）的钩子。</p><p>知道了发生丢包的问题来源，接下来再去定位 iptables 就比较容易了。在终端中执行下面的 iptables 命令，就可以查询 OUTPUT 链的过滤规则：</p><pre><code class="language-plain">sudo iptables -nvL OUTPUT
</code></pre><p>命令执行后，你应该可以看到类似下面的输出。可以看到，正是我们之前加入的 iptables 规则导致了丢包：</p><pre><code class="language-plain">Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
    1   180 DROP       tcp  --  *      *       0.0.0.0/0            39.106.233.176       tcp dpt:80
</code></pre><p>恭喜，到这里，通过简单的几行 bpftrace 脚本，你就成功使用 eBPF 精确定位了一个常见的网络丢包问题。</p><p>清楚了问题的根源，要解决它当然就很简单了。只要执行下面的命令，把导致丢包的 iptables 规则删除即可：</p><pre><code class="language-plain">sudo iptables -D OUTPUT -d 39.106.233.176/32 -p tcp -m tcp --dport 80 -j DROP
</code></pre><h2>如何根据函数偏移快速定位源码？</h2><p>在内核栈的输出中，我想你一定注意到每一个函数的输出格式都是<code>函数名+偏移量</code>，而这儿的偏移就是调用下一个函数的位置。那么，能不能根据<code>函数名+偏移量</code>直接定位源码的位置呢？</p><p>答案是肯定的。这是因为，不仅是我们这些 eBPF 学习者想要这种工具，内核开发者为了方便问题的排查，也经常需要根据内核栈，快速定位导致问题发生的代码位置。所以，Linux 内核维护了一个 <a href="https://github.com/torvalds/linux/blob/master/scripts/faddr2line">faddr2line</a> 脚本，根据<code>函数名+偏移量</code>输出源码文件名和行号。你可以点击<a href="https://github.com/torvalds/linux/blob/master/scripts/faddr2line">这里</a>，把它下载到本地，然后执行下面的命令加上可执行权限：</p><pre><code class="language-plain">chmod +x faddr2line
</code></pre><p>在使用这个脚本之前，你还需要注意两个前提条件：</p><ul>
<li>第一，带有调试信息的内核文件，一般名字为 vmlinux（注意，/boot 目录下面的 vmlinz 是压缩后的内核，不可以直接拿来使用）。</li>
<li>第二，系统中需要安装 <code>awk</code>、<code>readelf</code>、<code>addr2line</code>、<code>size</code>、<code>nm</code> 等命令。</li>
</ul><p>对于第二个条件，这些命令都包含在 <a href="https://www.gnu.org/software/binutils/">binutils</a> 软件包中，只需要执行 <code>apt</code> 或者 <code>dnf</code> 命令安装即可。</p><p>而对第一个条件中的内核调试信息，各个主要的发行版也都提供了相应的软件仓库，你可以根据文档进行安装。比如，对于 Ubuntu 来说，你可以执行下面的命令安装调试信息：</p><pre><code class="language-plain">codename=$(lsb_release -cs)
sudo tee /etc/apt/sources.list.d/ddebs.list &lt;&lt; EOF
deb http://ddebs.ubuntu.com/ ${codename}      main restricted universe multiverse
deb http://ddebs.ubuntu.com/ ${codename}-updates  main restricted universe multiverse
EOF
sudo apt-get install -y ubuntu-dbgsym-keyring
sudo apt-get update
sudo apt-get install -y linux-image-$(uname -r)-dbgsym
</code></pre><p>而对于 RHEL8 等系统，则可以执行下面的命令：</p><pre><code class="language-plain">sudo debuginfo-install kernel-$(uname -r)
</code></pre><p>调试信息安装好之后，相关的调试文件会放到 <code>/usr/lib/debug</code> 目录下。不同发行版的目录结构是不同的，但你可以使用下面的命令来搜索 <code>vmlinux</code> 开头的文件：</p><pre><code class="language-plain">find /usr/lib/debug/ -name 'vmlinux*'
</code></pre><p>以我使用的 Ubuntu 21.10 为例，查找到的文件路径为 <code>/usr/lib/debug/boot/vmlinux-5.13.0-22-generic</code>。所以，接下来，就可以执行下面的命令，对刚才内核栈中的 <code>__ip_local_out+219</code> 进行定位：</p><pre><code class="language-plain">faddr2line /usr/lib/debug/boot/vmlinux-5.13.0-22-generic __ip_local_out+219
</code></pre><p>命令执行后，可以得到下面的输出：</p><pre><code class="language-plain">__ip_local_out+219/0x150:
nf_hook at include/linux/netfilter.h:256
(inlined by) __ip_local_out at net/ipv4/ip_output.c:115
</code></pre><p>这个输出中的具体内容含义如下：</p><ul>
<li>第二行表示 <code>nf_hook</code>的定义位置在 <code>netfilter.h</code> 的<code>156</code>行。</li>
<li>第三行表示 <code>net/ipv4/ip_output.c</code> 的 <code>115</code>行调用了 <code>kfree_skb</code> 函数。但是，由于 <code>nf_hook</code> 是一个内联函数，所以行号<code>115</code>实际上是内联函数 <code>nf_hook</code> 的调用位置。</li>
</ul><p>对比一下上一个模块查找的<a href="https://elixir.bootlin.com/linux/v5.13/source/net/ipv4/ip_output.c#L115">内核源码</a>，<code>net/ipv4/ip_output.c</code> 的 115 号也刚好是调用 <code>nf_hook</code> 的位置：</p><p><img src="https://static001.geekbang.org/resource/image/31/6a/311b02beafc800c2bba434d24aa7b56a.png?wh=703x352" alt="图片" title="__ip_local_out 源码"></p><p>而再点击 <a href="https://elixir.bootlin.com/linux/v5.13/source/include/linux/netfilter.h#L205">nf_hook</a> 继续去看它的定义，你可以发现，这的确是个内联函数：</p><pre><code class="language-c++">static inline int nf_hook(...)
</code></pre><blockquote>
<p>小提示：内联是一种常用的编程优化技术，它告诉编译器把指定函数展开之后再进行编译，这样就省去了函数调用的开销。对频繁调用的小函数来说，这就可以大大提高程序的运行效率。</p>
</blockquote><p>有了 faddr2line 工具，在以后排查内核协议栈时，你就可以根据栈中<code>函数名+偏移量</code>，直接定位到源代码的位置。这样，你就可以直接去内核源码或 <a href="https://elixir.bootlin.com/">elixir.bootlin.com</a> 网站中查找相关函数的实现逻辑，进而更深层次地理解内核的实现原理。</p><h2>小结</h2><p>今天，我带你一起梳理了 eBPF 的网络功能，并以最常见的网络丢包问题为例，使用 bpftrace 开发了一个跟踪内核网络协议栈的 eBPF 程序。</p><p>eBPF 不仅诞生于网络过滤，它在网络方面的应用也是最为广泛和活跃的。由于内核协议栈也是内核的核心组成部分，前几讲我们讲到的 kprobe 跟踪、uprobe/USDT 跟踪等，也都可以应用到网络问题的跟踪和排查上来。由于内核协议栈相对比较复杂，在排查网络问题时，我们可以从内核调用栈入手。根据调用栈的执行过程，再配合 faddr2line 这样的工具，你就可以快速定位到问题发生所在的内核源码，进而找出问题的根源。</p><p>实际上，我们今天讲到的调用栈跟踪也可以用到其他内核功能和用户应用的跟踪上，并且也特别适用于性能优化领域经常需要的热点函数跟踪。在后续的课程中，我还将为你介绍更多的应用案例。</p><h2>思考题</h2><p>由于加入了进程信息和网络协议的限制，今天我们使用 bpftrace 开发的 eBPF 程序，实际上只能跟踪到 <code>curl</code>命令发出的 TCP 请求丢包问题。而在实际的应用中，很可能是其他的进程发生了丢包问题，并且丢包的也不一定都是 TCP 协议。</p><p>那么，根据这一讲的内容和 bpftrace 的文档，你可以对今天的跟踪程序进行改进，并把进程信息（如 PID 和进程名）加到输出中吗？欢迎在评论区和我分享你的思路和解决方法。</p><p>期待你在留言区和我讨论，也欢迎把这节课分享给你的同事、朋友。让我们一起在实战中演练，在交流中进步。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/5e/96/a03175bc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>莫名</span>
  </div>
  <div class="_2_QraFYR_0">1、比较赞同【最主要是因为不清楚内核中都有哪些函数和跟踪点可以拿来跟踪】这一观点。如果对内核不熟悉，借助 tracepoint 通配符可以很好的协助理解网络协议栈的关键路径。举两个简单例子。<br><br>&gt; 例 1：追踪 net 相关追踪点以及调用堆栈：<br><br># bpftrace -e &#39;tracepoint:net:* { printf(&quot;%s(%d): %s %s\n&quot;, comm, pid, probe, kstack()); }&#39;<br><br>输出示例（sshd 发送数据包关键路径，涉及 tracepoint:net:net_dev_queue、tracepoint:net:net_dev_xmit 等追踪点，可以借助这些追踪点可进一步计算数据包排队时间等）：<br><br>sshd(219966): tracepoint:net:net_dev_queue<br>        __dev_queue_xmit+1524<br>        dev_queue_xmit+16<br>        ip_finish_output2+718<br>        __ip_finish_output+191<br>        ip_finish_output+54<br>        ip_output+112<br>        ip_local_out+53<br>        __ip_queue_xmit+354<br>        ip_queue_xmit+16<br>        __tcp_transmit_skb+1407<br>        tcp_write_xmit+982<br>        __tcp_push_pending_frames+57<br>        tcp_push+219<br>        tcp_sendmsg_locked+2415<br>        tcp_sendmsg+44<br>        inet_sendmsg+59<br>        sock_write_iter+156<br>        new_sync_write+287<br>        __vfs_write+38<br>        vfs_write+171<br>        ksys_write+97<br>        __x64_sys_write+26<br>        do_syscall_64+71<br>        entry_SYSCALL_64_after_hwframe+68<br><br>sshd(219966): tracepoint:net:net_dev_xmit<br>        dev_hard_start_xmit+368<br>        dev_hard_start_xmit+368<br>        sch_direct_xmit+278<br>        __dev_queue_xmit+1713<br>        dev_queue_xmit+16<br>        ip_finish_output2+718<br>        ...<br>        __x64_sys_write+26<br>        do_syscall_64+71<br>        entry_SYSCALL_64_after_hwframe+68<br><br>&gt; 例 2：使用 perf trace 也比较方便：<br><br># perf trace --no-syscalls -e &#39;net:*&#39; curl -s time.geekbang.org &gt; &#47;dev&#47;null<br><br>输出示例（可以看到具体走了哪些网络设备、数据包长度、skb 内存地址等）：<br><br>0.439 curl&#47;2240 net:net_dev_queue:dev=eth1 skbaddr=... len=54<br>0.452 curl&#47;2240 net:net_dev_start_xmit:dev=eth1 skbaddr=... len=54<br>0.455 curl&#47;2240 net:net_dev_xmit:dev=eth1 skbaddr=... len=54<br>0.707 curl&#47;2240 net:netif_receive_skb:dev=eth1 skbaddr=... len=52<br><br>2、dropwatch.bt 头文件有些没使用到，可以做下简化：<br><br>#include &lt;linux&#47;skbuff.h&gt;<br>#include &lt;linux&#47;ip.h&gt;<br>#include &lt;linux&#47;socket.h&gt;<br>#include &lt;linux&#47;netdevice.h&gt;<br><br>=====================&gt;<br><br>#include &lt;linux&#47;skbuff.h&gt;<br>#include &lt;net&#47;ip.h&gt;<br><br>思考题直接用 bpftrace 内置变量 comm、pid 即可。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 非常👍的答案！也谢谢分享跟踪点学习的经验！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-07 12:19:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2d/27/c5/d4d00da2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>为了维护世界和平</span>
  </div>
  <div class="_2_QraFYR_0">github上的程序 执行 段错误<br>#bpftrace dropwatch.bt<br>Segmentation fault (core dumped)<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-19 18:26:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/cd/db/7467ad23.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Bachue Zhou</span>
  </div>
  <div class="_2_QraFYR_0">$ sudo bpftrace -l &#39;kprobe:kfree_skb*&#39;<br>kprobe:kfree_skb_list_reason<br>kprobe:kfree_skb_partial<br>kprobe:kfree_skb_reason<br>kprobe:kfree_skbmem<br><br>$ uname -r<br>5.19.0-38-generic<br><br>尴尬 我这边 5.19 就没有 kfree_skb 这个函数了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-06 16:02:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/83/fb/621adceb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>linker</span>
  </div>
  <div class="_2_QraFYR_0">$ apt list --installed |grep &#39;\&lt;linux-&#39;<br><br>WARNING: apt does not have a stable CLI interface. Use with caution in scripts.<br><br>binutils-x86-64-linux-gnu&#47;jammy-updates,jammy-security,now 2.38-4ubuntu2.1 amd64 [installed,automatic]<br>linux-base&#47;jammy,now 4.5ubuntu9 all [installed,automatic]<br>linux-buildinfo-5.19.0-35-generic&#47;jammy-updates,jammy-security,now 5.19.0-35.36~22.04.1 amd64 [installed]<br>linux-firmware&#47;jammy-security,now 20220329.git681281e4-0ubuntu3.9 all [installed,upgradable to: 20220329.git681281e4-0ubuntu3.10]<br>linux-headers-5.19.0-35-generic&#47;jammy-updates,jammy-security,now 5.19.0-35.36~22.04.1 amd64 [installed]<br>linux-hwe-5.19-headers-5.19.0-35&#47;jammy-updates,jammy-security,now 5.19.0-35.36~22.04.1 all [installed,automatic]<br>linux-hwe-5.19-tools-5.19.0-35&#47;jammy-updates,jammy-security,now 5.19.0-35.36~22.04.1 amd64 [installed,automatic]<br>linux-image-5.19.0-35-generic&#47;jammy-updates,jammy-security,now 5.19.0-35.36~22.04.1 amd64 [installed]<br>linux-libc-dev&#47;jammy-updates,jammy-security,now 5.15.0-67.74 amd64 [installed,automatic]<br>linux-modules-5.19.0-35-generic&#47;jammy-updates,jammy-security,now 5.19.0-35.36~22.04.1 amd64 [installed,automatic]<br>linux-sound-base&#47;jammy,now 1.0.25+dfsg-0ubuntu7 all [installed,automatic]<br>linux-source-5.19.0&#47;jammy-updates,jammy-security,now 5.19.0-35.36~22.04.1 all [installed]<br>linux-tools-5.19.0-35-generic&#47;jammy-updates,jammy-security,now 5.19.0-35.36~22.04.1 amd64 [installed]<br>linux-tools-common&#47;jammy-updates,jammy-security,now 5.15.0-67.74 all [installed,automatic]</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-13 19:22:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/83/fb/621adceb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>linker</span>
  </div>
  <div class="_2_QraFYR_0">$ cat &#47;etc&#47;os-release               <br>PRETTY_NAME=&quot;Ubuntu 22.04.1 LTS&quot;<br>NAME=&quot;Ubuntu&quot;<br>VERSION_ID=&quot;22.04&quot;<br>VERSION=&quot;22.04.1 LTS (Jammy Jellyfish)&quot;<br>VERSION_CODENAME=jammy<br>ID=ubuntu<br>ID_LIKE=debian<br>HOME_URL=&quot;https:&#47;&#47;www.ubuntu.com&#47;&quot;<br>SUPPORT_URL=&quot;https:&#47;&#47;help.ubuntu.com&#47;&quot;<br>BUG_REPORT_URL=&quot;https:&#47;&#47;bugs.launchpad.net&#47;ubuntu&#47;&quot;<br>PRIVACY_POLICY_URL=&quot;https:&#47;&#47;www.ubuntu.com&#47;legal&#47;terms-and-policies&#47;privacy-policy&quot;<br>UBUNTU_CODENAME=jammy<br><br>这个版本升级到 5.19 内核后，源码安装最新版bpftrace , 执行本节程序报错<br><br>definitions.h:2:10: fatal error: &#39;linux&#47;skbuff.h&#39; file not found<br>内核头文件已经安装了，内核开发包也安装了，请问老师这可能是什么问题？<br><br>下面是安装的包<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-13 19:21:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83erFUhEiazDuzntpbECvyYdp6IdLwO1ic01sE02op3ZtGvqJEwxJhoUozjHwqF5vHprluUKeAIvmru8Q/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_9e1ece</span>
  </div>
  <div class="_2_QraFYR_0">老师 ntop这个函数返回的是字符串么？为什么我用$sip或$dip去做if判断时候提示我两端的数据类不正确呢。<br><br>另外我翻了一下内核的源码，好像没有直接找到ntop（）这个名字的函数，请问还有pton（）这样的函数可以将字符串转成inet数据类型么，以便我在if判断中作为过滤条件。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: ntop是bpftrace提供的一个内置函数，把IP地址转换成字符串形式，方便展示。<br><br>具体文档可以参考 https:&#47;&#47;github.com&#47;iovisor&#47;bpftrace&#47;blob&#47;master&#47;docs&#47;reference_guide.md#14-ntop-convert-ip-address-data-to-text</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-03 11:51:31</div>
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
  <div class="_2_QraFYR_0">第二图里怎么没有tracepoint?应该是跟kprobe在同一个层面吧</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 图里的syscall其实就是tracepoint。不过你说的也有道理，tracepoint其实有很多，不只是syscall，也包括很多网络跟踪点。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-25 08:04:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/fe/fc/e56c9c4a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>嘿！我的gakki</span>
  </div>
  <div class="_2_QraFYR_0">老师想请问下，我看 cilium 的 ebpf 相关源码中，有个节叫 “from-container”，就在源码的 “cilium&#47;bpf&#47;bpf_lxc.c” 文件下，写作 “__section(&quot;from-container&quot;)”，但是关于这个 from-container 它长得很像一个 ebpf 的类型，但是我在 kernel 中又搜不到这个，用 &quot;bpftool feature probe&quot; 也没有和它相关的，所以想问下老师知不知道这个 from-container 是怎么变成 ebpf 程序被编译的？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-06 00:12:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/8a/b0/6ab66f1b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>牛金霖</span>
  </div>
  <div class="_2_QraFYR_0">老师，请问如何获取5.16的debuginfo</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-24 18:19:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/ec/0e/d64d4663.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>k8svip</span>
  </div>
  <div class="_2_QraFYR_0">老师 如何跟踪下PHP-fpm线程，去处理MySQL 域名解析的过程呢？偶现连不通的情况，回出现udp receive error 包数量增加，tcpdump又捕获不到</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以考虑去跟踪DNS请求？这可能比跟踪进程更简单一些</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-16 20:28:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>从远方过来</span>
  </div>
  <div class="_2_QraFYR_0">老师，看了你举的查找SKB的例子，我很好奇要怎么才能快速地从内核相关文档中找到相关的函数呢？  例如：找出关于内存分配的函数，  这个有什么好窍门么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 最容易的就是使用eBPF跟踪系统调用相关的内核栈，然后对栈中的相关函数进行跟踪。当然，如果熟悉内核代码，去内核源码查看更好。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-24 01:25:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/28/1c/b7e3941c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>阿bin斯基</span>
  </div>
  <div class="_2_QraFYR_0">ubuntu 18.04 ,snap安装的bpftrace,kernel symbol解析失败<br>➜  ~ bpftrace -V<br>bpftrace v0.13.0<br>➜  ~ echo $BPFTRACE_VMLINUX<br>&#47;usr&#47;lib&#47;debug&#47;boot&#47;vmlinux-5.4.0-96-generic<br>➜  ~ uname -a              <br>Linux max-machine 5.4.0-96-generic #109~18.04.1-Ubuntu SMP Thu Jan 13 15:06:26 UTC 2022 x86_64 x86_64 x86_64 GNU&#47;Linux<br>➜  ~                       <br>sudo bpftrace -e &#39;kprobe:kfree_skb &#47;comm==&quot;curl&quot;&#47; {printf(&quot;kstack: %s\n&quot;, kstack);}&#39;<br>[sudo] password for max: <br>Attaching 1 probe...<br>kstack: <br>        0xffffffff9091b941<br>        0xffffffff90a5f5f7<br>        0xffffffff90914e41<br>        0xffffffff90a5eae9<br>        0xffffffff909f4009<br>        0xffffffff90a351c0<br>        0xffffffff9090b8e2<br>        0xffffffff9090b975<br>        0xffffffff902e0276<br>        0xffffffff902e047e<br>        0xffffffff900c493d<br>        0xffffffff90003ec9<br>        0xffffffff90004320<br>        0xffffffff90c0008c<br><br>kstack: <br>        0xffffffff9091b941<br>        0xffffffff90a5f5f7<br>        0xffffffff90914e41<br>        0xffffffff90a5eae9<br>        0xffffffff909f4009<br>        0xffffffff90a351c0<br>        0xffffffff9090b8e2<br>        0xffffffff9090b975<br>        0xffffffff902e0276<br>        0xffffffff902e047e<br>        0xffffffff900c493d<br>        0xffffffff90003ec9<br>        0xffffffff90004320<br>        0xffffffff90c0008c<br><br>kstack: <br>        0xffffffff9091b941<br>        0xffffffff9090fad3<br>        0xffffffff9090fb6a<br>        0xffffffff90004207<br>        0xffffffff90c0008c<br><br>kstack: <br>        0xffffffff9091b941<br>        0xffffffff9090fad3<br>        0xffffffff9090fb6a<br>        0xffffffff90004207<br>        0xffffffff90c0008c<br><br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 有没有使用snap install --devmode来安装？Github上面也有个类似的issuehttps:&#47;&#47;github.com&#47;iovisor&#47;bpftrace&#47;issues&#47;1488。旧的发行版最好还是从源码安装 BCC 和 bpftrace。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-07 17:29:16</div>
  </div>
</div>
</div>
</li>
</ul>