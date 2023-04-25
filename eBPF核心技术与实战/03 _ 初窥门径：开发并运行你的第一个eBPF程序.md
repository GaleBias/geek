<audio title="03 _ 初窥门径：开发并运行你的第一个eBPF程序" src="https://static001.geekbang.org/resource/audio/e7/61/e7ae9bb7a331279c8f1ef1yyd7257c61.mp3" controls="controls"></audio> 
<p>你好，我是倪朋飞。</p><p>通过前面两讲，我已经带你为正式进入 eBPF 的学习做好了准备，接下来我们进入第二个模块“基础入门篇”的学习。在这个模块里，你会学习到 eBPF 的开发环境搭建方法、运行原理、编程接口，以及各种类型 eBPF 程序的事件触发机制和应用场景。</p><p>上一讲，我带你一起梳理了 eBPF 的学习路径和学习技巧，并特别强调了动手实践在学习 eBPF 过程中的重要性。那么，eBPF 程序到底是什么样子的？如何搭建 eBPF 的开发环境，又如何开发一个 eBPF 应用呢？</p><p>今天，我就带你一起上手开发第一个 eBPF 程序。</p><h2>如何选择 eBPF 开发环境？</h2><p>在前两讲中，我曾提到，虽然 Linux 内核很早就已经支持了 eBPF，但很多新特性都是<a href="https://github.com/iovisor/bcc/blob/master/docs/kernel-versions.md#main-features">在 4.x 版本中逐步增加的</a>。所以，想要稳定运行 eBPF 程序，内核至少需要 <strong>4.9</strong> 或者更新的版本。而在开发和学习 eBPF 时，为了体验和掌握最新的 eBPF 特性，我推荐使用更新的 <strong>5.x</strong> 内核。</p><p>作为 eBPF 最重大的改进之一，一次编译到处执行（简称 CO-RE）解决了内核数据结构在不同版本差异导致的兼容性问题。不过，在使用 CO-RE 之前，内核需要开启  <code>CONFIG_DEBUG_INFO_BTF=y</code> 和 <code>CONFIG_DEBUG_INFO=y</code> 这两个编译选项。为了避免你在首次学习 eBPF 时就去重新编译内核，我推荐使用已经默认开启这些编译选项的发行版，作为你的开发环境，比如：</p><!-- [[[read_end]]] --><ul>
<li>Ubuntu 20.10+</li>
<li>Fedora 31+</li>
<li>RHEL 8.2+</li>
<li>Debian 11+</li>
</ul><p>你可以到公有云平台上创建这些发行版的虚拟机，也可以借助 <a href="https://www.vagrantup.com">Vagrant</a>&nbsp;、<a href="https://multipass.run">Multipass</a> 等工具，创建本地的虚拟机。比如，使用我最喜欢的 <a href="https://www.vagrantup.com/downloads">Vagrant</a>&nbsp;，通过下面几步就可以创建出一个 Ubuntu 21.10 的虚拟机：</p><pre><code class="language-bash">#&nbsp;创建和启动Ubuntu 21.10虚拟机
vagrant init&nbsp;ubuntu/impish64
vagrant up

# 登录到虚拟机
vagrant ssh
</code></pre><h2>如何搭建 eBPF 开发环境？</h2><p>虚拟机创建好之后，接下来就需要安装 eBPF 开发和运行所需要的开发工具，这包括：</p><ul>
<li>将 eBPF 程序编译成字节码的 LLVM；</li>
<li>C 语言程序编译工具 make；</li>
<li>最流行的 eBPF 工具集 BCC 和它依赖的内核头文件；</li>
<li>与内核代码仓库实时同步的 libbpf；</li>
<li>同样是内核代码提供的 eBPF 程序管理工具 bpftool。</li>
</ul><p>你可以执行下面的命令，来安装这些必要的开发工具：</p><pre><code class="language-shell">#&nbsp;For Ubuntu20.10+
sudo apt-get install -y  make clang llvm libelf-dev&nbsp;libbpf-dev bpfcc-tools libbpfcc-dev linux-tools-$(uname -r) linux-headers-$(uname -r)

# For RHEL8.2+
sudo yum install libbpf-devel make clang llvm elfutils-libelf-devel bpftool bcc-tools bcc-devel
</code></pre><p>如果你已经熟悉了 Linux 内核的自定义编译和安装方法，并选择在旧的发行版中通过自行编译和升级内核搭建开发环境，上述的开发工具流程也需要做适当的调整。这里特别提醒下，libbpf-dev 这个库很可能需要从源码安装，具体的步骤你可以参考 libbpf 的 <a href="https://github.com/libbpf/libbpf">GitHub 仓库</a>。</p><h2>如何开发第一个 eBPF 程序？</h2><p>当前面这些开发工具和依赖库安装完成后，一个完整的 eBPF 开发环境就准备好了。接下来，你肯定迫不及待地想要体验一下 eBPF 的强大功能了。</p><p>不过，在开发 eBPF 程序之前，我们先来看一下 eBPF 的开发和执行过程。如下图（图片来自 <a href="https://www.brendangregg.com/ebpf.html">brendangregg.com</a>）所示，一般来说，这个过程分为以下 5 步：</p><ul>
<li>第一步，使用 C 语言开发一个 eBPF 程序；</li>
<li>第二步，借助 LLVM 把 eBPF 程序编译成 BPF 字节码；</li>
<li>第三步，通过 bpf 系统调用，把 BPF 字节码提交给内核；</li>
<li>第四步，内核验证并运行 BPF 字节码，并把相应的状态保存到 BPF 映射中；</li>
<li>第五步，用户程序通过 BPF 映射查询 BPF 字节码的运行状态。</li>
</ul><p><img src="https://static001.geekbang.org/resource/image/a7/6a/a7165eea1fd9fc24090a3a1e8987986a.png?wh=1500x550" alt="图片" title="eBPF 程序执行过程"></p><p>这里的每一步，我们当然可以自己动手去完成。但对初学者来说，我推荐你从 BCC（BPF Compiler Collection）开始学起。</p><p>BCC 是一个 BPF 编译器集合，包含了用于构建 BPF 程序的编程框架和库，并提供了大量可以直接使用的工具。使用 BCC 的好处是，<strong>它把上述的 eBPF 执行过程通过内置框架抽象了起来，并提供了 Python、C++ 等编程语言接口</strong>。这样，你就可以直接通过 Python 语言去跟 eBPF 的各种事件和数据进行交互。</p><p>接下来，我就以跟踪 <a href="https://man7.org/linux/man-pages/man2/open.2.html">openat()</a>（即打开文件）这个系统调用为例，带你来看看如何开发并运行第一个 eBPF 程序。</p><p>使用 BCC 开发 eBPF 程序，可以把前面讲到的五步简化为下面的三步。</p><h3><strong>第一步：使用 C 开发一个 eBPF 程序</strong></h3><p>新建一个  <code>hello.c</code>&nbsp;文件，并输入下面的内容：</p><pre><code class="language-c++">int hello_world(void *ctx)
{
&nbsp; &nbsp; bpf_trace_printk("Hello, World!");
&nbsp; &nbsp; return 0;
}
</code></pre><p>就像所有编程语言的“&nbsp;Hello World&nbsp;”示例一样，这段代码的含义就是打印一句 “Hello, World!” 字符串。其中， <code>bpf_trace_printk()</code>&nbsp;是一个最常用的 BPF 辅助函数，它的作用是输出一段字符串。不过，由于 eBPF 运行在内核中，它的输出并不是通常的标准输出（stdout），而是内核调试文件  <code>/sys/kernel/debug/tracing/trace_pipe</code>&nbsp;，你可以直接使用  <code>cat</code>  命令来查看这个文件的内容。</p><h3>第二步：使用 Python 和 BCC 库开发一个用户态程序</h3><p>接下来，创建一个  <code>hello.py</code>  文件，并输入下面的内容：</p><pre><code class="language-python">#!/usr/bin/env python3
# 1) import bcc library
from bcc import BPF

# 2) load BPF program
b = BPF(src_file="hello.c")
# 3) attach kprobe
b.attach_kprobe(event="do_sys_openat2", fn_name="hello_world")
# 4) read and print&nbsp;/sys/kernel/debug/tracing/trace_pipe
b.trace_print()
</code></pre><p>让我们来看看每一处的具体含义：</p><ul>
<li>第 1) 处导入了 BCC&nbsp;库的 BPF 模块，以便接下来调用；</li>
<li>第 2) 处调用 BPF() 加载第一步开发的 BPF 源代码；</li>
<li>第 3) 处将 BPF 程序挂载到内核探针（简称 kprobe），其中  <code>do_sys_openat2()</code> 是系统调用  <code>openat()</code>  在内核中的实现；</li>
<li>第 4) 处则是读取内核调试文件  <code>/sys/kernel/debug/tracing/trace_pipe</code>  的内容，并打印到标准输出中。</li>
</ul><p>在运行的时候，BCC 会调用 LLVM，把 BPF 源代码编译为字节码，再加载到内核中运行。</p><h3>第三步：执行 eBPF 程序</h3><p>用户态程序开发完成之后，最后一步就是执行它了。需要注意的是， eBPF 程序需要以 root 用户来运行，非 root 用户需要加上 sudo 来执行：</p><pre><code class="language-python">sudo python3 hello.py
</code></pre><p>稍等一会，你就可以看到如下的输出：</p><pre><code class="language-python">b' cat-10656 [006] d... 2348.114455: bpf_trace_printk: Hello, World!'
</code></pre><p>输出的格式可由  <code>/sys/kernel/debug/tracing/trace_options</code>&nbsp;来修改。比如前面这个默认的输出中，每个字段的含义如下所示：</p><ul>
<li>cat-10656 表示进程的名字和 PID；</li>
<li>[006] 表示 CPU 编号；</li>
<li>d… 表示一系列的选项；</li>
<li>2348.114455 表示时间戳；</li>
<li>bpf_trace_printk 表示函数名；</li>
<li>最后的 “Hello, World!” 就是调用  <code>bpf_trace_printk()</code>  传入的字符串。</li>
</ul><p>到了这里，恭喜你已经成功开发并运行了第一个 eBPF 程序！不过，短暂的兴奋之后，你会发现这个程序还有不少的缺点，比如：</p><ul>
<li>既然跟踪的是打开文件的系统调用，除了调用这个接口进程的名字之外，被打开的文件名也应该在输出中；</li>
<li><code>bpf_trace_printk()</code>&nbsp;的输出格式不够灵活，像是 CPU 编号、bpf_trace_printk 函数名等内容没必要输出；</li>
<li>……</li>
</ul><p>实际上，我并不推荐通过内核调试文件系统输出日志的方式。一方面，它会带来很大的性能问题；另一方面，所有的 eBPF 程序都会把内容输出到同一个位置，很难根据 eBPF 程序去区分日志的来源。</p><p>那么，怎么来解决这些问题呢？接下来，我们就试着一起改进这个程序。</p><h2>如何改进第一个 eBPF 程序？</h2><p>在&nbsp;<a href="https://time.geekbang.org/column/article/479384">01 讲</a>&nbsp;中我曾提到，BPF 程序可以利用 BPF 映射（map）进行数据存储，而用户程序也需要通过 BPF 映射，同运行在内核中的 BPF 程序进行交互。所以，为了解决上面提到的第一个问题，即获取被打开文件名的问题，我们就要引入 BPF 映射。</p><p>为了简化 BPF 映射的交互，BCC 定义了一系列的<a href="https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md#output">库函数和辅助宏定义</a>。比如，你可以使用  <code>BPF_PERF_OUTPUT</code>  来定义一个 Perf 事件类型的 BPF 映射，代码如下：</p><pre><code class="language-c++">// 包含头文件
#include &lt;uapi/linux/openat2.h&gt;
#include &lt;linux/sched.h&gt;

// 定义数据结构
struct data_t {
&nbsp; u32 pid;
&nbsp; u64 ts;
&nbsp; char comm[TASK_COMM_LEN];
&nbsp; char fname[NAME_MAX];
};

// 定义性能事件映射
BPF_PERF_OUTPUT(events);
</code></pre><p>然后，在 eBPF 程序中，填充这个数据结构，并调用  <code>perf_submit()</code>  把数据提交到刚才定义的 BPF 映射中：</p><pre><code class="language-c++">// 定义kprobe处理函数
int hello_world(struct pt_regs *ctx, int dfd, const char __user * filename, struct open_how *how)
{
&nbsp; struct data_t data = { };

  // 获取PID和时间
&nbsp; data.pid = bpf_get_current_pid_tgid();
&nbsp; data.ts = bpf_ktime_get_ns();

  // 获取进程名
&nbsp; if (bpf_get_current_comm(&amp;data.comm, sizeof(data.comm)) == 0)
&nbsp; {
&nbsp; &nbsp; bpf_probe_read(&amp;data.fname, sizeof(data.fname), (void *)filename);
&nbsp; }

  // 提交性能事件
&nbsp; events.perf_submit(ctx, &amp;data, sizeof(data));
&nbsp; return 0;
}
</code></pre><p>其中，以 bpf 开头的函数都是 eBPF 提供的辅助函数，比如：</p><ul>
<li><code>bpf_get_current_pid_tgid</code>  用于获取进程的 TGID 和 PID。因为这儿定义的 data.pid 数据类型为 u32，所以高 32 位舍弃掉后就是进程的 PID；</li>
<li><code>bpf_ktime_get_ns</code>  用于获取系统自启动以来的时间，单位是纳秒；</li>
<li><code>bpf_get_current_comm</code>  用于获取进程名，并把进程名复制到预定义的缓冲区中；</li>
<li><code>bpf_probe_read</code>  用于从指定指针处读取固定大小的数据，这里则用于读取进程打开的文件名。</li>
</ul><p>有了 BPF 映射之后，前面我们调用的&nbsp;<code>bpf_trace_printk()</code>&nbsp;其实就不再需要了，因为用户态进程可以直接从 BPF 映射中读取内核 eBPF 程序的运行状态。</p><p>这其实也就是上面提到的第二个待解决问题。那么，怎样从用户态读取 BPF 映射内容并输出到标准输出（stdout）呢？</p><p>在 BCC 中，与 eBPF 程序中  <code>BPF_PERF_OUTPUT</code>&nbsp;相对应的用户态辅助函数是  <code>open_perf_buffer()</code> 。它需要传入一个回调函数，用于处理从 Perf 事件类型的 BPF 映射中读取到的数据。具体的使用方法如下所示：</p><pre><code class="language-python">from bcc import BPF

# 1) load BPF program
b = BPF(src_file="trace-open.c")
b.attach_kprobe(event="do_sys_openat2", fn_name="hello_world")

# 2) print header
print("%-18s %-16s %-6s %-16s" % ("TIME(s)", "COMM", "PID", "FILE"))

# 3) define the callback for perf event
start = 0
def print_event(cpu, data, size):
&nbsp; &nbsp; global start
&nbsp; &nbsp; event = b["events"].event(data)
&nbsp; &nbsp; if start == 0:
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; start = event.ts
&nbsp; &nbsp; time_s = (float(event.ts - start)) / 1000000000
&nbsp; &nbsp; print("%-18.9f %-16s %-6d %-16s" % (time_s, event.comm, event.pid, event.fname))

# 4) loop with callback to print_event
b["events"].open_perf_buffer(print_event)
while 1:
&nbsp; &nbsp; try:
&nbsp; &nbsp; &nbsp; &nbsp; b.perf_buffer_poll()
&nbsp; &nbsp; except KeyboardInterrupt:
&nbsp; &nbsp; &nbsp; &nbsp; exit()
</code></pre><p>让我们来看看每一处的具体含义：</p><ul>
<li>第 1) 处跟前面的 Hello World 一样，加载 eBPF 程序并挂载到内核探针上；</li>
<li>第 2) 处则是输出一行 Header 字符串表示数据的格式；</li>
<li>第 3) 处的  <code>print_event</code>  定义一个数据处理的回调函数，打印进程的名字、PID 以及它调用 openat 时打开的文件；</li>
<li>第 4) 处的  <code>open_perf_buffer</code>  定义了名为 “events” 的 Perf 事件映射，而后通过一个循环调用  <code>perf_buffer_poll</code>  读取映射的内容，并执行回调函数输出进程信息。</li>
</ul><p>将前面的 eBPF 程序保存到  <code>trace-open.c</code> ，然后再把上述的 Python 程序保存到  <code>trace-open.py</code> 之后（你可以在 GitHub <a href="https://github.com/feiskyer/ebpf-apps/blob/main/bcc-apps/python/trace_open.py">ebpf-apps</a> 上找到完整的代码），就能以 root 用户来运行了：</p><pre><code class="language-python">sudo python3 trace-open.py
</code></pre><p>稍等一会，你会看到类似下面的输出：</p><pre><code class="language-python">TIME(s)&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; COMM&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;PID&nbsp; &nbsp; FILE
2.384485400&nbsp; &nbsp; &nbsp; &nbsp; b'irqbalance'&nbsp; &nbsp; 991&nbsp; &nbsp; b'/proc/interrupts'
2.384750400&nbsp; &nbsp; &nbsp; &nbsp; b'irqbalance'&nbsp; &nbsp; 991&nbsp; &nbsp; b'/proc/stat'
2.384838400&nbsp; &nbsp; &nbsp; &nbsp; b'irqbalance'&nbsp; &nbsp; 991&nbsp; &nbsp; b'/proc/irq/0/smp_affinity'
</code></pre><p>恭喜，你已经开发了第一个完整的 eBPF 程序。相对于前面的 Hello World，它的输出不仅格式更为清晰，还把进程打开的文件名输出出来了，这在排查频繁打开文件相关的性能问题时尤其有用。</p><h2>小结</h2><p>今天，我带你一起搭建了 eBPF 的开发环境，安装了 eBPF 开发时常用的工具和依赖库。并且，我从最简单的 Hello World 开始，带你借助 BCC 从零开发了一个跟踪 <a href="https://man7.org/linux/man-pages/man2/open.2.html">openat()</a>&nbsp;系统调用的 eBPF 程序。</p><p>通常，开发一个 eBPF 程序需要经过开发 C 语言 eBPF 程序、编译为 BPF 字节码、加载 BPF 字节码到内核、内核验证并运行 BPF 字节码，以及用户程序读取 BPF 映射五个步骤。使用 BCC 的好处是，它把这几个步骤通过内置框架抽象了起来，并提供了简单易用的 Python 接口，这可以帮你大大简化 eBPF 程序的开发。</p><p>除此之外，BCC 提供的一系列工具不仅可以直接用在生产环境中，还是你学习和开发新的 eBPF 程序的最佳参考示例。在课程后续的内容中，我还会带你深入 BCC 的详细使用方法。</p><h2>思考题</h2><p>最后，我想请你聊一聊这几个问题：</p><ol>
<li>你通常都是如何搭建 Linux 和 eBPF 环境的？</li>
<li>在今天的案例操作中，你遇到了什么问题，又是如何解决的呢？</li>
<li>虽然今天开发的程序非常短小，你觉得它能否在日常的工作中帮助到你呢？</li>
</ol><p>欢迎在留言区和我讨论，也欢迎把这节课分享给你的同事、朋友。让我们一起在实战中演练，在交流中进步。</p>
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
  <div class="_2_QraFYR_0">追踪文件打开事件，采用场景大致有：<br>1、查看某个程序启动时加载了哪些配置文件，便于确认是否加载了正确的配置文件。对于允许自定义配置文件路径的程序尤其有用，例如 MySQL、PostgreSQL。<br>2、查看是否存在频繁或周期性打开某些文件的情况，考虑是否存在优化可能。比如周期性打开某个极少变化的文件，可以一次性读取，且监听文件变动事件，避免多次打开读取。<br>3、分析依赖 &#47;proc、&#47;sys 等虚拟文件系统的 Linux 工具大致工作原理。比如执行 vmstat，，可以通过追踪文件打开事件看到至少打开了 &#47;proc&#47;meminfo、&#47;proc&#47;stat、&#47;proc&#47;vmstat 这几个文件，帮助你更好的理解工具的数据源与实现原理。<br>4、分析 K8s、Docker 等 cgroup 相关操作。比如 docker run xxx 时，可以看到 &#47;sys&#47;fs&#47;cgroup&#47;cpuset&#47;docker&#47;xxx&#47;cpuset.cpus、&#47;sys&#47;fs&#47;cgroup&#47;cpuset&#47;docker&#47;xxx&#47;cpuset.mems 等 cgroup 文件被打开，也可以查看 kube-proxy 在周期性刷新 cgroup 相关文件。<br>5、....</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 太赞了👍 感谢分享追踪文件打开的这些应用场景</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-21 09:50:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/5e/96/a03175bc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>莫名</span>
  </div>
  <div class="_2_QraFYR_0">给倪老师的 Github 仓库里面的这个小程序挑个🐛：<br>1、b = BPF(src_file=&quot;trace-open.c&quot;)，trace-open.c -&gt; trace_open.c<br>2、依赖了 openat2.h 头文件，openat2 系统调用自 5.6 版本才出现，低于这个版本无法运行 trace_open.py，建议用 openat 系统调用，有比较好的版本兼容性。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. 谢谢指出，已经修复了。<br>2. 是的，但我们专栏还是希望使用新版本的内核来学习，这样一方面能体验最新的特性，另一方面不需要等后面需要用到新特性的时候再重新配置开发环境。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-21 10:15:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/50/05/90f1a14e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>JUNLONG</span>
  </div>
  <div class="_2_QraFYR_0">习惯用docker的同学可以试下 这个镜像，github地址：github.com&#47;Jun10ng&#47;ebpf-for-desktop <br>用docker可以用vscode编辑会高效点。如果喜欢的话就来个start吧 谢谢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍谢谢分享！我通常还是使用虚拟机作为开发环境，只在部署的时候才用容器。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-23 14:36:41</div>
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
  <div class="_2_QraFYR_0">请问这种对open系统调用的截获会影响用户文件打开的性能不? 影响到什么程度呢?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对大部分应用来说，性能损失基本可以忽略。但如果应用对性能有比较极致的敏感，比如到了纳秒或者指令集，那就需要考虑eBPF的性能损耗了。<br><br>给你分享一个eBPF性能评估的演讲，希望有所帮助：https:&#47;&#47;ebpf.io&#47;summit-2020-slides&#47;eBPF_Summit_2020-Lightning-Bryce_Kahle-How_and_When_You_Should_Measure_CPU_Overhead_of_eBPF_Programs.pdf。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-08 22:01:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/rQOn22bNV0kHpoPWRLRicjQCOkiaYmcVABiaIJxIDWIibSdqWXYTxjcdjiadibIxFsGVp5UE4DBd6Nx2DxjhAdlMIZeQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ThinkerWalker</span>
  </div>
  <div class="_2_QraFYR_0">有个Python语法的疑问：trace_open.py中，print_event被调用的时候【28行，b[&quot;events&quot;].open_perf_buffer(print_event)】是如何传入三个参数（cpu, data, size）呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这些参数都是由BCC框架默认提供的，不需要使用的时候再额外传入。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-21 16:44:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/pNKoOAa1QXibrykHNXibW4tyaIIhicocPGXtcVnEianCyOQY9bl0P2JQ3wSialUaolcLVEWycCEBz1Oe4Tj4yghH9yw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_5aa343</span>
  </div>
  <div class="_2_QraFYR_0">老师，您好，有几个问题请教下：<br>1. &quot;do_sys_openat2() 是系统调用 openat() 在内核中的实现&quot; 怎么去找到一个系统调用在内核中的实现呢？<br>2. 使用BPF map获取openat的打开文件名这里，&#47;&#47; 定义数据结构struct data_t { u32 pid; u64 ts; char comm[TASK_COMM_LEN]; char fname[NAME_MAX];}; 这个的格式为啥是这样呢，就是有具体每个探针的map说明文档嘛<br>多谢老师</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. 每个系统调用在内核中都有类似 sys_xxx 的实现函数，我们后面课程会讲到怎么查询内核中的跟踪点，根据名字去过滤就可以找到实现函数的名字。当然，如果你要看具体的实现步骤，就得需要去看看内核的源码了。<br>2. 没有固定的格式，这个数据结构其实就是你想在用户态中获取的数据，不同的程序需要的数据是不同的。<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-21 10:48:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/0e/b0/ebf0afc3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小术</span>
  </div>
  <div class="_2_QraFYR_0">eBPF 也可以在 Android 系统上运行，不过在 Android 上搭建环境有点麻烦；自荐一个我写的工具：https:&#47;&#47;github.com&#47;tiann&#47;eadb</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-23 16:48:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://wx.qlogo.cn/mmopen/vi_32/DYAIOgq83eqtXSgThiaEiaEqqic5YIJ7v469nCM3VXiccOJ4SxbYjW91ciczuYYEzcTVtYWaWXaokZqShuLdKsXjnFA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_b85295</span>
  </div>
  <div class="_2_QraFYR_0">老师，我的环境是ubuntu20.04，内核5.4.0-92-generic，把环境依赖都正常安装好后，执行python3 hello.py 出现错误 Failed to attach BPF program b&#39;hello_world&#39; to kprobe b&#39;do_sys_openat2&#39;，网上没找到解决方法，可以帮忙看看嘛</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，碰到这种问题最好的方法是去查man手册。比如，你可以在 https:&#47;&#47;man7.org&#47;linux&#47;man-pages&#47;man2&#47;openat2.2.html#VERSIONS 看到，openat2 是 5.6 内核才支持的。所以，对旧的内核来说，就需要替换成 openat 系统调用。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-08 17:34:19</div>
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
  <div class="_2_QraFYR_0">原来这个 trace_open.py  就是一个「简化」版的 opensnoop-bpfcc  工具呀。<br><br>我为了更清晰更简单的打印出 trace_open.py 的输出把一些服务都关了 ：）<br>systemctl stop multipathd<br>systemctl stop snapd<br>systemctl stop irqbalance </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 看来你已经对BCC比较熟悉了😊</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-25 17:23:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/97/95/aad51e9b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>waterjiao</span>
  </div>
  <div class="_2_QraFYR_0">还有几个问题咨询下老师<br>1. perf缓存区大小如何查看<br>2.如果缓存区满了，后续数据会覆盖之前的数据吗？open_perf_buffer是阻塞的么<br>3. libbpf是怎么做到co-re的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个时候就需要去看文档了。比如前两个问题的答案都在 https:&#47;&#47;github.com&#47;iovisor&#47;bcc&#47;blob&#47;23a21423a31719bd79e9e975b8f6dca8f7a331e5&#47;docs&#47;reference_guide.md#2-open_perf_buffer：<br><br>https:&#47;&#47;github.com&#47;iovisor&#47;bcc&#47;blob&#47;23a21423a31719bd79e9e975b8f6dca8f7a331e5&#47;docs&#47;reference_guide.md#2-open_perf_buffer<br><br>对于CO-RE的原理，可以看一下 https:&#47;&#47;nakryiko.com&#47;posts&#47;bpf-core-reference-guide&#47;</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-17 21:42:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKdVVxCVYJMmSmia51JzwSju1yNgDNNPD6lO9LOY2tygkxpGrVbIbAQZqHb04SDwuaibIEEGzudUD3g/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>HenryHui</span>
  </div>
  <div class="_2_QraFYR_0">老师您好：<br>我在运行例子的时候出现了错误如下错误：<br><br>root@ubuntu-linux-20-04-desktop:~&#47;project&#47;ebpf&#47;study# sudo python3 hello.py<br>cannot attach kprobe, probe entry may not exist<br>Traceback (most recent call last):<br>  File &quot;hello.py&quot;, line 8, in &lt;module&gt;<br>    b.attach_kprobe(event=&quot;do_sys_openat2&quot;, fn_name=&quot;hello_world&quot;)<br>  File &quot;&#47;usr&#47;lib&#47;python3&#47;dist-packages&#47;bcc&#47;__init__.py&quot;, line 658, in attach_kprobe<br>    raise Exception(&quot;Failed to attach BPF program %s to kprobe %s&quot; %<br>Exception: Failed to attach BPF program b&#39;hello_world&#39; to kprobe b&#39;do_sys_openat2<br><br>我的系统内核是：<br>Linux ubuntu-linux-20-04-desktop 5.4.0-80-generic #90-Ubuntu SMP Fri Jul 9 17:43:26 UTC 2021 aarch64 aarch64 aarch64 GNU&#47;Linux<br><br>重点是这句报错：cannot attach kprobe, probe entry may not exist  是说我的kprobe不存在么<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，碰到这种问题最好的方法是去查man手册。比如，你可以在 https:&#47;&#47;man7.org&#47;linux&#47;man-pages&#47;man2&#47;openat2.2.html#VERSIONS 看到，openat2 是 5.6 内核才支持的。所以，对旧的内核来说，就需要替换成 openat 系统调用（我们课程推荐使用新一些的内核，这样可以体验最新的eBPF特性，所以还是继续使用openat2作为示例）。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-11 15:21:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/82/01/8ee18ad0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Y</span>
  </div>
  <div class="_2_QraFYR_0">E: 无法定位软件包 libbpf-dev<br>提示这个报错怎么办？更新了源好像也不行</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: libbpf-dev只包含在比较新的发行版中，其他发行版可以从源码安装，具体步骤可以参考 https:&#47;&#47;github.com&#47;libbpf&#47;libbpf#build。<br><br>另外，我们案例的Github中也有源码编译的详细步骤：https:&#47;&#47;github.com&#47;feiskyer&#47;ebpf-apps&#47;blob&#47;main&#47;bpf-apps&#47;Makefile#L21-L22</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-08 00:12:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/41/38/4f89095b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>写点啥呢</span>
  </div>
  <div class="_2_QraFYR_0">有几个问题想请教老师：<br>1. perf_buffer_poll方法是非阻塞的么<br>2. bpf_probe_read(&amp;data.fname, sizeof(data.fname), (void *)filename); 这里filename指针的内存大小是否也是NAME_MAX，不然读取应该会导致非法访问<br>3. hello_world方法的几个形参是什么含义，感觉ctx是bpf固定的，后面dfd, filename和open_how是openat2的参数，请问是否编写bpf函数都是可以在ctx后面加入对应系统调用接口的入参，然后bpf会在执行时候自动进行参数绑定？<br>4. perf_submit函数传入的c struct在bcc脚本中看上去可以通过event方法自动转化为python对象？<br><br>谢谢老师<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不错的问题，看来是深入学习了，赞一个👍<br>1. 这个函数实际上对所有的perf缓冲区调用它们的回调函数，然后就结束了（所以外层还有一个while循环）。<br>2. 不是的，过长的filename会截断。<br>3和4的理解都是对的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-28 23:25:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/3c/c4/7c2bf312.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>草根</span>
  </div>
  <div class="_2_QraFYR_0">请问老师，hello_world的入参是要和系统调用的入参设置一致吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个示例实际上是kprobe类型的eBPF程序，它的参数格式是表示寄存器上下文的struct pt_regs *ctx和内核函数的实际参数。后面的课程还会详细介绍eBPF支持的各种程序类型。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-21 14:04:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/2a/b9/2bf8cc89.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>无名氏</span>
  </div>
  <div class="_2_QraFYR_0">老师 请问下我想每个程序一个日志输出怎么处理？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-25 14:08:23</div>
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
  <div class="_2_QraFYR_0">BPF_PERF_OUTPUT(events);  请问一下这个events是什么，从哪里传进来的？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-06 09:56:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/05/63/dd59ad18.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>加油加油</span>
  </div>
  <div class="_2_QraFYR_0">老师问下，为什么我的ubuntu:20 里面只有设置 __x64_sys_openat  才有用  按照文章里的就不行呢 ？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 安装bpftrace之后，执行 bpftrace -l | grep openat 查询可用的跟踪点试试？必须是内核支持的跟踪点才可以用 eBPF 来跟踪。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-24 20:10:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_9e8767</span>
  </div>
  <div class="_2_QraFYR_0">希望老师能分享下 bpf 的开发环境搭建、路径配置这些基础。主要困难是如何在mac 上使用 ide 去开发bpf，ide 如何配置可以让头文件可以被索引到。 golang 开发者，接触这个有点陌生。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯 直接在Mac上面确实没有这些头文件。我一般推荐使用vscode的远程开发模式，连接到Linux中的代码库，然后就可以在系统中找到相关的头文件了。 <br><br>详细的步骤可以参考 https:&#47;&#47;code.visualstudio.com&#47;docs&#47;remote&#47;ssh</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-08 17:19:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/bc/e2/e760c8e2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>一位不愿透露姓名的王先生</span>
  </div>
  <div class="_2_QraFYR_0">BPF_PERF_OUTPUT(events);<br>这个 events 是在哪定义的？ </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这一句的含义就是定义 events，并且类型是 perf event。<br><br>详细的文档请参考 https:&#47;&#47;github.com&#47;iovisor&#47;bcc&#47;blob&#47;master&#47;docs&#47;reference_guide.md#2-bpf_perf_output</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-24 15:04:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/5e/67/133d2da6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_5244fa</span>
  </div>
  <div class="_2_QraFYR_0">为什么要先判断 bpf_get_current_comm 成功之后再 bpf_probe_read ？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 虽然是运行在内核中，这些调用也可能会失败。失败后就应该尽早退出，不需要再去消耗额外的指令了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-22 23:32:57</div>
  </div>
</div>
</div>
</li>
</ul>