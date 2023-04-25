<audio title="49 _ 案例篇：内核线程 CPU 利用率太高，我该怎么办？" src="https://static001.geekbang.org/resource/audio/fc/37/fc09ab6d80b0d233f2adb2f7a88ecd37.mp3" controls="controls"></audio> 
<p>你好，我是倪朋飞。</p><p>上一期，我们一起梳理了，网络时不时丢包的分析定位和优化方法。先简单回顾一下。</p><p>网络丢包，通常会带来严重的性能下降，特别是对 TCP 来说，丢包通常意味着网络拥塞和重传，进而会导致网络延迟增大以及吞吐量降低。</p><p>而分析丢包问题，还是用我们的老套路，从 Linux 网络收发的流程入手，结合 TCP/IP 协议栈的原理来逐层分析。</p><p>其实，在排查网络问题时，我们还经常碰到的一个问题，就是内核线程的 CPU 使用率很高。比如，在高并发的场景中，内核线程 ksoftirqd 的 CPU 使用率通常就会比较高。回顾一下前面学过的 CPU 和网络模块，你应该知道，这是网络收发的软中断导致的。</p><p>而要分析 ksoftirqd 这类 CPU 使用率比较高的内核线程，如果用我前面介绍过的那些分析方法，你一般需要借助于其他性能工具，进行辅助分析。</p><p>比如，还是以 ksoftirqd 为例，如果你怀疑是网络问题，就可以用 sar、tcpdump 等分析网络流量，进一步确认网络问题的根源。</p><p>不过，显然，这种方法在实际操作中需要步骤比较多，可能并不算快捷。你肯定也很想知道，有没有其他更简单的方法，可以直接观察内核线程的行为，更快定位瓶颈呢？</p><p>今天，我就继续以 ksoftirqd 为例，带你一起看看，如何分析内核线程的性能问题。</p><!-- [[[read_end]]] --><h2>内核线程</h2><p>既然要讲内核线程的性能问题，在案例开始之前，我们就先来看看，有哪些常见的内核线程。</p><p>我们知道，在 Linux 中，用户态进程的“祖先”，都是 PID 号为 1 的 init 进程。比如，现在主流的 Linux 发行版中，init 都是 systemd 进程；而其他的用户态进程，会通过 systemd 来进行管理。</p><p>稍微想一下 Linux 中的各种进程，除了用户态进程外，还有大量的内核态线程。按说内核态的线程，应该先于用户态进程启动，可是 systemd 只管理用户态进程。那么，内核态线程又是谁来管理的呢？</p><p>实际上，Linux 在启动过程中，有三个特殊的进程，也就是 PID 号最小的三个进程。</p><ul>
<li>
<p>0 号进程为 idle 进程，这也是系统创建的第一个进程，它在初始化 1 号和 2 号进程后，演变为空闲任务。当 CPU 上没有其他任务执行时，就会运行它。</p>
</li>
<li>
<p>1 号进程为 init 进程，通常是 systemd 进程，在用户态运行，用来管理其他用户态进程。</p>
</li>
<li>
<p>2 号进程为 kthreadd 进程，在内核态运行，用来管理内核线程。</p>
</li>
</ul><p>所以，要查找内核线程，我们只需要从 2 号进程开始，查找它的子孙进程即可。比如，你可以使用 ps 命令，来查找 kthreadd 的子进程：</p><pre><code>$ ps -f --ppid 2 -p 2
UID         PID   PPID  C STIME TTY          TIME CMD
root          2      0  0 12:02 ?        00:00:01 [kthreadd]
root          9      2  0 12:02 ?        00:00:21 [ksoftirqd/0]
root         10      2  0 12:02 ?        00:11:47 [rcu_sched]
root         11      2  0 12:02 ?        00:00:18 [migration/0]
...
root      11094      2  0 14:20 ?        00:00:00 [kworker/1:0-eve]
root      11647      2  0 14:27 ?        00:00:00 [kworker/0:2-cgr]
</code></pre><p>从上面的输出，你能够看到，内核线程的名称（CMD）都在中括号里（这一点，我们前面内容也有提到过）。所以，更简单的方法，就是直接查找名称包含中括号的进程。比如：</p><pre><code>$ ps -ef | grep &quot;\[.*\]&quot;
root         2     0  0 08:14 ?        00:00:00 [kthreadd]
root         3     2  0 08:14 ?        00:00:00 [rcu_gp]
root         4     2  0 08:14 ?        00:00:00 [rcu_par_gp]
...
</code></pre><p>了解内核线程的基本功能，对我们排查问题有非常大的帮助。比如，我们曾经在软中断案例中提到过 ksoftirqd。它是一个用来处理软中断的内核线程，并且每个 CPU 上都有一个。</p><p>如果你知道了这一点，那么，以后遇到 ksoftirqd 的 CPU 使用高的情况，就会首先怀疑是软中断的问题，然后从软中断的角度来进一步分析。</p><p>其实，除了刚才看到的 kthreadd 和 ksoftirqd 外，还有很多常见的内核线程，我们在性能分析中都经常会碰到，比如下面这几个内核线程。</p><ul>
<li>
<p><strong>kswapd0</strong>：用于内存回收。在  <a href="https://time.geekbang.org/column/article/75797">Swap变高</a> 案例中，我曾介绍过它的工作原理。</p>
</li>
<li>
<p><strong>kworker</strong>：用于执行内核工作队列，分为绑定 CPU （名称格式为 kworker/CPU:ID）和未绑定 CPU（名称格式为 kworker/uPOOL:ID）两类。</p>
</li>
<li>
<p><strong>migration</strong>：在负载均衡过程中，把进程迁移到 CPU 上。每个 CPU 都有一个 migration 内核线程。</p>
</li>
<li>
<p><strong>jbd2</strong>/sda1-8：jbd 是 Journaling Block Device 的缩写，用来为文件系统提供日志功能，以保证数据的完整性；名称中的 sda1-8，表示磁盘分区名称和设备号。每个使用了 ext4 文件系统的磁盘分区，都会有一个 jbd2 内核线程。</p>
</li>
<li>
<p><strong>pdflush</strong>：用于将内存中的脏页（被修改过，但还未写入磁盘的文件页）写入磁盘（已经在 3.10 中合并入了 kworker 中）。</p>
</li>
</ul><p>了解这几个容易发生性能问题的内核线程，有助于我们更快地定位性能瓶颈。接下来，我们来看今天的案例。</p><h2>案例准备</h2><p>今天的案例还是基于 Ubuntu 18.04，同样适用于其他的 Linux 系统。我使用的案例环境如下所示：</p><ul>
<li>
<p>机器配置：2 CPU，8GB 内存。</p>
</li>
<li>
<p>预先安装 docker、perf、hping3、curl 等工具，如 apt install docker.io linux-tools-common hping3。</p>
</li>
</ul><p>本次案例用到两台虚拟机，我画了一张图来表示它们的关系。</p><p><img src="https://static001.geekbang.org/resource/image/7d/11/7dd0763a14713940e7c762a62387dd11.png?wh=816*516" alt=""></p><p>你需要打开两个终端，分别登录这两台虚拟机中，并安装上述工具。</p><p>注意，以下所有命令都默认以 root 用户运行，如果你用普通用户身份登陆系统，请运行 sudo su root 命令，切换到 root 用户。</p><blockquote>
<p>如果安装过程有问题，你可以先上网搜索解决，实在解决不了的，记得在留言区向我提问。</p>
</blockquote><p>到这里，准备工作就完成了。接下来，我们正式进入操作环节。</p><h2>案例分析</h2><p>安装完成后，我们先在第一个终端，执行下面的命令运行案例，也就是一个最基本的 Nginx 应用：</p><pre><code># 运行Nginx服务并对外开放80端口
$ docker run -itd --name=nginx -p 80:80 nginx
</code></pre><p>然后，在第二个终端，使用 curl 访问 Nginx 监听的端口，确认 Nginx 正常启动。假设 192.168.0.30 是 Nginx 所在虚拟机的 IP 地址，运行 curl 命令后，你应该会看到下面这个输出界面：</p><pre><code>$ curl http://192.168.0.30/
&lt;!DOCTYPE html&gt;
&lt;html&gt;
&lt;head&gt;
&lt;title&gt;Welcome to nginx!&lt;/title&gt;
...
</code></pre><p>接着，还是在第二个终端中，运行 hping3 命令，模拟 Nginx 的客户端请求：</p><pre><code># -S参数表示设置TCP协议的SYN（同步序列号），-p表示目的端口为80
# -i u10表示每隔10微秒发送一个网络帧
# 注：如果你在实践过程中现象不明显，可以尝试把10调小，比如调成5甚至1
$ hping3 -S -p 80 -i u10 192.168.0.30
</code></pre><p>现在，我们再回到第一个终端，你应该就会发现异常——系统的响应明显变慢了。我们不妨执行 top，观察一下系统和进程的 CPU 使用情况：</p><pre><code>$ top
top - 08:31:43 up 17 min,  1 user,  load average: 0.00, 0.00, 0.02
Tasks: 128 total,   1 running,  69 sleeping,   0 stopped,   0 zombie
%Cpu0  :  0.3 us,  0.3 sy,  0.0 ni, 66.8 id,  0.3 wa,  0.0 hi, 32.4 si,  0.0 st
%Cpu1  :  0.0 us,  0.3 sy,  0.0 ni, 65.2 id,  0.0 wa,  0.0 hi, 34.5 si,  0.0 st
KiB Mem :  8167040 total,  7234236 free,   358976 used,   573828 buff/cache
KiB Swap:        0 total,        0 free,        0 used.  7560460 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
    9 root      20   0       0      0      0 S   7.0  0.0   0:00.48 ksoftirqd/0
   18 root      20   0       0      0      0 S   6.9  0.0   0:00.56 ksoftirqd/1
 2489 root      20   0  876896  38408  21520 S   0.3  0.5   0:01.50 docker-containe
 3008 root      20   0   44536   3936   3304 R   0.3  0.0   0:00.09 top
    1 root      20   0   78116   9000   6432 S   0.0  0.1   0:11.77 systemd
 ...
</code></pre><p>从 top 的输出中，你可以看到，两个 CPU 的软中断使用率都超过了 30%；而 CPU 使用率最高的进程，正好是软中断内核线程 ksoftirqd/0 和 ksoftirqd/1。</p><p>虽然，我们已经知道了 ksoftirqd 的基本功能，可以猜测是因为大量网络收发，引起了 CPU 使用率升高；但它到底在执行什么逻辑，我们却并不知道。</p><p>对于普通进程，我们要观察其行为有很多方法，比如 strace、pstack、lsof 等等。但这些工具并不适合内核线程，比如，如果你用 pstack ，或者通过 /proc/pid/stack 查看 ksoftirqd/0（进程号为 9）的调用栈时，分别可以得到以下输出：</p><pre><code>$ pstack 9
Could not attach to target 9: Operation not permitted.
detach: No such process
</code></pre><pre><code>$ cat /proc/9/stack
[&lt;0&gt;] smpboot_thread_fn+0x166/0x170
[&lt;0&gt;] kthread+0x121/0x140
[&lt;0&gt;] ret_from_fork+0x35/0x40
[&lt;0&gt;] 0xffffffffffffffff
</code></pre><p>显然，pstack 报出的是不允许挂载进程的错误；而 /proc/9/stack 方式虽然有输出，但输出中并没有详细的调用栈情况。</p><p>那还有没有其他方法，来观察内核线程 ksoftirqd 的行为呢？</p><p>既然是内核线程，自然应该用到内核中提供的机制。回顾一下我们之前用过的 CPU 性能工具，我想你肯定还记得 perf ，这个内核自带的性能剖析工具。</p><p>perf 可以对指定的进程或者事件进行采样，并且还可以用调用栈的形式，输出整个调用链上的汇总信息。 我们不妨就用 perf ，来试着分析一下进程号为 9 的 ksoftirqd。</p><p>继续在终端一中，执行下面的 perf record 命令；并指定进程号 9 ，以便记录 ksoftirqd 的行为:</p><pre><code># 采样30s后退出
$ perf record -a -g -p 9 -- sleep 30
</code></pre><p>稍等一会儿，在上述命令结束后，继续执行 <code>perf report</code>命令，你就可以得到 perf 的汇总报告。按上下方向键以及回车键，展开比例最高的 ksoftirqd 后，你就可以得到下面这个调用关系链图：</p><p><img src="https://static001.geekbang.org/resource/image/73/01/73f5e9a9e510b9f3bf634c5e94e67801.png?wh=1494*1366" alt=""></p><p>从这个图中，你可以清楚看到 ksoftirqd 执行最多的调用过程。虽然你可能不太熟悉内核源码，但通过这些函数，我们可以大致看出它的调用栈过程。</p><ul>
<li>
<p>net_rx_action 和 netif_receive_skb，表明这是接收网络包（rx 表示 receive）。</p>
</li>
<li>
<p>br_handle_frame ，表明网络包经过了网桥（br 表示 bridge）。</p>
</li>
<li>
<p>br_nf_pre_routing ，表明在网桥上执行了 netfilter 的 PREROUTING（nf 表示 netfilter）。而我们已经知道 PREROUTING 主要用来执行 DNAT，所以可以猜测这里有 DNAT 发生。</p>
</li>
<li>
<p>br_pass_frame_up，表明网桥处理后，再交给桥接的其他桥接网卡进一步处理。比如，在新的网卡上接收网络包、执行 netfilter 过滤规则等等。</p>
</li>
</ul><p>我们的猜测对不对呢？实际上，我们案例最开始用 Docker 启动了容器，而 Docker 会自动为容器创建虚拟网卡、桥接到 docker0 网桥并配置 NAT 规则。这一过程，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/72/70/72f2f1fa7a464e4465108d4eadcc1b70.png?wh=1244*1620" alt=""></p><p>当然了，前面 perf report 界面的调用链还可以继续展开。但很不幸，我的屏幕不够大，如果展开更多的层级，最后几个层级会超出屏幕范围。这样，即使我们能看到大部分的调用过程，却也不能说明后面层级就没问题。</p><p>那么，有没有更好的方法，来查看整个调用栈的信息呢？</p><h2>火焰图</h2><p>针对 perf 汇总数据的展示问题，Brendan Gragg 发明了<a href="http://www.brendangregg.com/flamegraphs.html">火焰图</a>，通过矢量图的形式，更直观展示汇总结果。下图就是一个针对 mysql 的火焰图示例。</p><p><img src="https://static001.geekbang.org/resource/image/68/61/68b80d299b23b0cee518001f78960f61.png?wh=1976*1184" alt=""></p><p>（图片来自 Brendan Gregg <a href="http://www.brendangregg.com/flamegraphs.html">博客</a>）</p><p>这张图看起来像是跳动的火焰，因此也就被称为火焰图。要理解火焰图，我们最重要的是区分清楚横轴和纵轴的含义。</p><ul>
<li>
<p><strong>横轴表示采样数和采样比例</strong>。一个函数占用的横轴越宽，就代表它的执行时间越长。同一层的多个函数，则是按照字母来排序。</p>
</li>
<li>
<p><strong>纵轴表示调用栈</strong>，由下往上根据调用关系逐个展开。换句话说，上下相邻的两个函数中，下面的函数，是上面函数的父函数。这样，调用栈越深，纵轴就越高。</p>
</li>
</ul><p>另外，要注意图中的颜色，并没有特殊含义，只是用来区分不同的函数。</p><p>火焰图是动态的矢量图格式，所以它还支持一些动态特性。比如，鼠标悬停到某个函数上时，就会自动显示这个函数的采样数和采样比例。而当你用鼠标点击函数时，火焰图就会把该层及其上的各层放大，方便你观察这些处于火焰图顶部的调用栈的细节。</p><p>上面 mysql 火焰图的示例，就表示了 CPU 的繁忙情况，这种火焰图也被称为 on-CPU 火焰图。如果我们根据性能分析的目标来划分，火焰图可以分为下面这几种。</p><ul>
<li>
<p><strong>on-CPU 火焰图</strong>：表示 CPU 的繁忙情况，用在 CPU 使用率比较高的场景中。</p>
</li>
<li>
<p><strong>off-CPU 火焰图</strong>：表示 CPU 等待 I/O、锁等各种资源的阻塞情况。</p>
</li>
<li>
<p><strong>内存火焰图</strong>：表示内存的分配和释放情况。</p>
</li>
<li>
<p><strong>热/冷火焰图</strong>：表示将 on-CPU 和 off-CPU 结合在一起综合展示。</p>
</li>
<li>
<p><strong>差分火焰图</strong>：表示两个火焰图的差分情况，红色表示增长，蓝色表示衰减。差分火焰图常用来比较不同场景和不同时期的火焰图，以便分析系统变化前后对性能的影响情况。</p>
</li>
</ul><p>了解了火焰图的含义和查看方法后，接下来，我们再回到案例，运用火焰图来观察刚才 perf record 得到的记录。</p><h2>火焰图分析</h2><p>首先，我们需要生成火焰图。我们先下载几个能从 perf record 记录生成火焰图的工具，这些工具都放在 <a href="https://github.com/brendangregg/FlameGraph">https://github.com/brendangregg/FlameGraph</a> 上面。你可以执行下面的命令来下载：</p><pre><code>$ git clone https://github.com/brendangregg/FlameGraph
$ cd FlameGraph
</code></pre><p>安装好工具后，要生成火焰图，其实主要需要三个步骤：</p><ol>
<li>
<p>执行 perf script ，将 perf record 的记录转换成可读的采样记录；</p>
</li>
<li>
<p>执行 stackcollapse-perf.pl 脚本，合并调用栈信息；</p>
</li>
<li>
<p>执行 flamegraph.pl 脚本，生成火焰图。</p>
</li>
</ol><p>不过，在 Linux 中，我们可以使用管道，来简化这三个步骤的执行过程。假设刚才用 perf record 生成的文件路径为 /root/perf.data，执行下面的命令，你就可以直接生成火焰图：</p><pre><code>$ perf script -i /root/perf.data | ./stackcollapse-perf.pl --all |  ./flamegraph.pl &gt; ksoftirqd.svg
</code></pre><p>执行成功后，使用浏览器打开 ksoftirqd.svg ，你就可以看到生成的火焰图了。如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/6d/cd/6d4f1fece12407906aacedf5078e53cd.png?wh=2370*1360" alt=""></p><p>根据刚刚讲过的火焰图原理，这个图应该从下往上看，沿着调用栈中最宽的函数来分析执行次数最多的函数。这儿看到的结果，其实跟刚才的 perf report 类似，但直观了很多，中间这一团火，很明显就是最需要我们关注的地方。</p><p>我们顺着调用栈由下往上看（顺着图中蓝色箭头），就可以得到跟刚才 perf report 中一样的结果：</p><ul>
<li>
<p>最开始，还是 net_rx_action 到 netif_receive_skb 处理网络收包；</p>
</li>
<li>
<p>然后， br_handle_frame 到 br_nf_pre_routing ，在网桥中接收并执行 netfilter 钩子函数；</p>
</li>
<li>
<p>再向上， br_pass_frame_up 到 netif_receive_skb ，从网桥转到其他网络设备又一次接收。</p>
</li>
</ul><p>不过最后，到了 ip_forward 这里，已经看不清函数名称了。所以我们需要点击 ip_forward，展开最上面这一块调用栈：</p><p><img src="https://static001.geekbang.org/resource/image/41/a3/416291ba2f9c039a0507f913572a21a3.png?wh=2388*1374" alt=""></p><p>这样，就可以进一步看到 ip_forward 后的行为，也就是把网络包发送出去。根据这个调用过程，再结合我们前面学习的网络收发和 TCP/IP 协议栈原理，这个流程中的网络接收、网桥以及 netfilter 调用等，都是导致软中断 CPU 升高的重要因素，也就是影响网络性能的潜在瓶颈。</p><p>不过，回想一下网络收发的流程，你可能会觉得它缺了好多步骤。</p><p>比如，这个堆栈中并没有 TCP 相关的调用，也没有连接跟踪 conntrack 相关的函数。实际上，这些流程都在其他更小的火焰中，你可以点击上图左上角的“Reset Zoom”，回到完整火焰图中，再去查看其他小火焰的堆栈。</p><p>所以，在理解这个调用栈时要注意。从任何一个点出发、纵向来看的整个调用栈，其实只是最顶端那一个函数的调用堆栈，而非完整的内核网络执行流程。</p><p>另外，整个火焰图不包含任何时间的因素，所以并不能看出横向各个函数的执行次序。</p><p>到这里，我们就找出了内核线程 ksoftirqd 执行最频繁的函数调用堆栈，而这个堆栈中的各层级函数，就是潜在的性能瓶颈来源。这样，后面想要进一步分析、优化时，也就有了根据。</p><h2>小结</h2><p>今天这个案例，你可能会觉得比较熟悉。实际上，这个案例，正是我们专栏 CPU 模块中的  <a href="https://time.geekbang.org/column/article/72147">软中断案例</a>。</p><p>当时，我们从软中断 CPU 使用率的角度入手，用网络抓包的方法找出了瓶颈来源，确认是测试机器发送的大量 SYN 包导致的。而通过今天的 perf 和火焰图方法，我们进一步找出了软中断内核线程的热点函数，其实也就找出了潜在的瓶颈和优化方向。</p><p>其实，如果遇到的是内核线程的资源使用异常，很多常用的进程级性能工具并不能帮上忙。这时，你就可以用内核自带的 perf 来观察它们的行为，找出热点函数，进一步定位性能瓶。当然，perf 产生的汇总报告并不够直观，所以我也推荐你用火焰图来协助排查。</p><p>实际上，火焰图方法同样适用于普通进程。比如，在分析 Nginx、MySQL 等各种应用场景的性能问题时，火焰图也能帮你更快定位热点函数，找出潜在性能问题。</p><h2>思考</h2><p>最后，我想邀请你一起来聊聊，你碰到过的内核线程性能问题。你是怎么分析它们的根源？又是怎么解决的？你可以结合我的讲述，总结自己的思路。</p><p>欢迎在留言区和我讨论，也欢迎把这篇文章分享给你的同事、朋友。我们一起在实战中演练，在交流中进步。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/xzHDjCSFicNY3MUMECtNz6sM8yDJhBoyGk5IRoOtUat6ZIkGzxjqEqwqKYWMD3GjehScKvMjicGOGDog5FF18oyg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>李逍遥</span>
  </div>
  <div class="_2_QraFYR_0">cpu火焰图和内存火焰图，在生成数据时有什么不同？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 火焰图的结构是一样的，只是函数堆栈不一样，内存火焰图侧重于内存管理函数的调用栈</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-01 15:02:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/xzHDjCSFicNY3MUMECtNz6sM8yDJhBoyGk5IRoOtUat6ZIkGzxjqEqwqKYWMD3GjehScKvMjicGOGDog5FF18oyg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>李逍遥</span>
  </div>
  <div class="_2_QraFYR_0">老师，能讲讲内存火焰图生成perf.data数据时,perf record加哪些选项吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 要加上内存管理相关的事件（函数），比如perf record -e syscalls:sys_enter_mmap -a -g -- sleep 60</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-03 08:39:27</div>
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
  <div class="_2_QraFYR_0">横轴的长短代表执行时间长短，一个函数被调用多次那横轴很长，一个函数执行一次但是在里面休眠了，这算执行时间很长吗？on-cpu火焰图是不是只记录真正的在cpu上的执行时间而不把睡眠时间算在内？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-27 15:35:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/8c/2b/3ab96998.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>青石</span>
  </div>
  <div class="_2_QraFYR_0">两周时间，终于追上来了。<br><br>请问老师，有哪些书有助于通过内核函数来定位故障，Linux用了9年，看到这还是感觉有些吃力。<br><br>内核线程问题，我的环境和老师的有些区别，没有br_nf_pre_routing函数调用，但是从ip_forward推测与消息转发有关，sar发现有大量小包接收，conntrack -L看到大量本机到docker地址的SYN_SENT状态的连接、hping3服务器到测试服务器的SYN_RECV状态连接。初步定位到具体的docker。<br><br>上面思考的过程，有点因为知道问题点，所以朝这个方向走的感觉。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 内核里面其实就提供了很多工具，可以参考下50和51篇中的动态追踪方法</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-22 11:02:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/80/6e/7f78292e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>无</span>
  </div>
  <div class="_2_QraFYR_0">请问有没有可以表示调用顺序的火焰图? 或者类似的其它图??? 感觉这样更有用阿</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以用 ebpf 跟踪调用栈</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-05 07:44:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>opdozz</span>
  </div>
  <div class="_2_QraFYR_0">老师，最近碰到一个kworker内核进程问题，48C的服务器，docker宿主机， 某个kworker会占满两个CPU，一个sys跑满，另外一个iowait跑满，剩下的CPU都很空闲，但是业务处理很慢，重启之后就好了，但是过段时间又复发， 不规律。<br>perf收集了信息，__rpc_execute 这个调用占了很多，下层nfs4_xdr_enc_open调用也占了很多，和挂的nas存储有关系吗，但是存储那边排查没有任何问题。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-09 10:10:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/3b/36/2d61e080.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>行者</span>
  </div>
  <div class="_2_QraFYR_0">很赞，准备回去用火焰图分析下我们后端服务。^ _ ^</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-19 00:54:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKsz8j0bAayjSne9iakvjzUmvUdxWEbsM9iasQ74spGFayIgbSE232sH2LOWmaKtx1WqAFDiaYgVPwIQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>2xshu</span>
  </div>
  <div class="_2_QraFYR_0">老师，这是最后一节课程吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不是，看原来的目录，还有好些篇</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-18 16:42:35</div>
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
  <div class="_2_QraFYR_0">[D49打卡]<br>之前用火焰图分析过golang程序的内存分配及cpu使用率情况.感觉非常直观.能快速找到瓶颈.</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯嗯 Go内置了pprof 工具，用起来更简单</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-18 12:03:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/fa/03/eba78e43.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>风清扬</span>
  </div>
  <div class="_2_QraFYR_0">$ ps -ef | grep &quot;\[.*\]&quot;<br>与ps -ef | grep &quot;\[*\]&quot; 有啥区别，为啥要加.?<br><br><br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-17 23:52:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/62/dc/8876c73b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>moooofly</span>
  </div>
  <div class="_2_QraFYR_0">“到这里，我们就找出了内核线程 ksoftirqd 执行最频繁的函数调用堆栈，而这个堆栈中的各层级函数，就是潜在的性能瓶颈来源。这样，后面想要进一步分析、优化时，也就有了根据。” -- 然后呢，我还是不知道要如何解决，在高并发的场景中，内核线程 ksoftirqd 的 CPU 使用率高的问题……</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-11 11:19:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_1b7d36</span>
  </div>
  <div class="_2_QraFYR_0">top发现load average : 2270.00  ,2270.26,  2270.26,<br>但是cpu使用率不超过10% ，32C64G的机器，其他的进程的CPU很低，就是这个负载很高，这个怎么排查呢，老师</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-09 14:58:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/99/7a/558666a5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>AceslupK</span>
  </div>
  <div class="_2_QraFYR_0">此处火焰图，吃力了。依旧还是没看明白该怎么找出未知问题，还是觉得这次是有着方向的解决问题</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-15 15:05:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/4Lprf2mIWpJOPibgibbFCicMtp5bpIibyLFOnyOhnBGbusrLZC0frG0FGWqdcdCkcKunKxqiaOHvXbCFE7zKJ8TmvIA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_c2089d</span>
  </div>
  <div class="_2_QraFYR_0">执行  <br>perf script -i &#47;root&#47;perf.data | .&#47;stackcollapse-perf.pl --all |  .&#47;flamegraph.pl &gt; ksoftirqd.svg<br>会报ERROR: No stack counts found错误，但是权限是都777的</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-18 14:27:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJGMphabeneYRlxs1biaO9oKic6Dwgbe312561lE56V93uUHgXXAsGmK1pH18mvpElygoJh8SUtQPUA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>董皋</span>
  </div>
  <div class="_2_QraFYR_0">打卡</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-20 22:32:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Littlesoup</span>
  </div>
  <div class="_2_QraFYR_0">&quot;一个函数占用的横轴越宽，就代表它的执行时间越长。&quot;<br>&quot;另外，整个火焰图不包含任何时间的因素，所以并不能看出横向各个函数的执行次序。&quot;<br>原文这两句话读起来有点困惑，第二句的意思是不是不包含任何时序的因素？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-26 19:14:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/97/e1/0f4d90ff.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>乖，摸摸头</span>
  </div>
  <div class="_2_QraFYR_0">为啥我的一值都是显示的16进制而不是函数名</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-15 11:50:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>如果</span>
  </div>
  <div class="_2_QraFYR_0">DAY49，打卡<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-17 13:45:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://wx.qlogo.cn/mmopen/vi_32/PiajxSqBRaEKQMM4m7NHuicr55aRiblTSEWIYe0QqbpyHweaoAbG7j2v7UUElqqeP3Ihrm3UfDPDRb1Hv8LvPwXqA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ninuxer</span>
  </div>
  <div class="_2_QraFYR_0">打卡day52<br>有碰到一个内核问题，docker宿主机上kworker&#47;u80进程的cpu占用率一直100%，其他的kworker进程都正常，每隔几个月就会碰到一次，为了快速恢复业务，就直接重启了，主要是没办法在线下实验的时候复现问题，所以就没有深入的分析，后面碰到后，可以用老师的方法，把perf record采集一段时间的调用信息，然后拿出去分析下👍</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯嗯</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-18 08:09:38</div>
  </div>
</div>
</div>
</li>
</ul>