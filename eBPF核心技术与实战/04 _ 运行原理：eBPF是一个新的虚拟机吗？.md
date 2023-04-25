<audio title="04 _ 运行原理：eBPF是一个新的虚拟机吗？" src="https://static001.geekbang.org/resource/audio/42/61/42bd4663fe40c75d81e5d32b830e0c61.mp3" controls="controls"></audio> 
<p>你好，我是倪朋飞。</p><p>上一讲，我带你一起搭建了 eBPF 的开发环境，并从最简单的 Hello World 开始，带你借助 BCC 库从零开发了一个跟踪 <a href="https://man7.org/linux/man-pages/man2/open.2.html">openat()</a>&nbsp;系统调用的 eBPF 程序。</p><p>不过，虽然第一个 eBPF 程序已经成功运行起来了，你很可能还在想：这个 eBPF 程序到底是如何编译成内核可识别的格式的？又是如何在内核中运行起来的？还有，既然允许普通用户去修改内核的行为，它又是如何确保内核安全的呢？</p><p>今天，我就带你一起深入看看 eBPF 虚拟机的原理，以及&nbsp;eBPF 程序是如何执行的。</p><h2>eBPF 虚拟机是如何工作的？</h2><p>eBPF 是一个运行在内核中的虚拟机，很多人在初次接触它时，会把它跟系统虚拟化（比如kvm）中的虚拟机弄混。其实，虽然都被称为“虚拟机”，系统虚拟化和 eBPF 虚拟机还是有着本质不同的。</p><p>系统虚拟化基于 x86 或 arm64 等通用指令集，这些指令集足以完成完整计算机的所有功能。而为了确保在内核中安全地执行，eBPF 只提供了非常有限的指令集。这些指令集可用于完成一部分内核的功能，但却远不足以模拟完整的计算机。为了更高效地与内核进行交互，eBPF 指令还有意采用了 C 调用约定，其提供的辅助函数可以在 C 语言中直接调用，极大地方便了 eBPF 程序的开发。</p><!-- [[[read_end]]] --><p>如下图（图片来自 <a href="https://www.usenix.org/conference/lisa21/presentation/gregg-bpf">BPF Internals</a>）所示，eBPF 在内核中的运行时主要由&nbsp;5&nbsp;个模块组成：</p><p><img src="https://static001.geekbang.org/resource/image/45/d2/453f8d99cea1b35da8f6c57e552yy3d2.png?wh=915x503" alt="图片" title="eBPF 运行时"></p><ul>
<li>第一个模块是&nbsp;<strong>eBPF 辅助函数</strong>。它提供了一系列用于 eBPF 程序与内核其他模块进行交互的函数。这些函数并不是任意一个 eBPF 程序都可以调用的，具体可用的函数集由 BPF 程序类型决定。关于 BPF 程序类型，我会在 06 讲 中进行讲解。</li>
<li>第二个模块是&nbsp;<strong>eBPF 验证器</strong>。它用于确保 eBPF 程序的安全。验证器会将待执行的指令创建为一个有向无环图（DAG），确保程序中不包含不可达指令；接着再模拟指令的执行过程，确保不会执行无效指令。</li>
<li>第三个模块是由&nbsp;<strong>11 个 64 位寄存器、一个程序计数器和一个 512 字节的栈组成的存储模块</strong>。这个模块用于控制 eBPF 程序的执行。其中，R0 寄存器用于存储函数调用和 eBPF 程序的返回值，这意味着函数调用最多只能有一个返回值；R1-R5 寄存器用于函数调用的参数，因此函数调用的参数最多不能超过 5 个；而 R10 则是一个只读寄存器，用于从栈中读取数据。</li>
<li>第四个模块是<strong>即时编译器</strong>，它将 eBPF 字节码编译成本地机器指令，以便更高效地在内核中执行。</li>
<li>第五个模块是&nbsp;<strong>BPF 映射（map）</strong>，它用于提供大块的存储。这些存储可被用户空间程序用来进行访问，进而控制 eBPF 程序的运行状态。</li>
</ul><p>关于 BPF 辅助函数和 BPF 映射的具体内容，我在后面的课程中还会为你详细介绍。接下来，我们先来看看 BPF 指令的具体格式，以及它是如何加载到内核中，又是何时运行的。</p><h2>BPF 指令是什么样的？</h2><p>只看图中的这些模块，你可能觉得它们并不是太直观。所以接下来，我们还是用上一讲的 Hello World 作为例子，一起看下 BPF 指令到底是什么样子的。</p><p>首先，回顾一下上一讲的  eBPF 程序&nbsp;Hello World&nbsp;的源代码。它的逻辑其实很简单，先调用&nbsp;  <code>bpf_trace_printk</code>  输出一个 “Hello, World!” 字符串，然后就返回成功了：</p><pre><code class="language-c++">int hello_world(void *ctx)
{
&nbsp; bpf_trace_printk("Hello, World!");
&nbsp; return 0;
}
</code></pre><p>然后，我们通过 BCC 的 Python 库，加载并运行了这个 eBPF 程序：</p><pre><code class="language-python">#!/usr/bin/env python3
# This is a Hello World example of BPF.
from bcc import BPF

# load BPF program
b = BPF(src_file="hello.c")
b.attach_kprobe(event="do_sys_openat2", fn_name="hello_world")
b.trace_print()
</code></pre><p>在终端中运行下面的命令，就可以启动这个 eBPF 程序（注意， BCC 帮你完成了编译和加载的过程）：</p><pre><code class="language-python">sudo python3 hello.py
</code></pre><p><strong>接下来，我为你介绍一个新的工具 bpftool，<strong><strong>用它可以</strong></strong>查看 eBPF 程序的运行状态。</strong></p><p>首先，打开一个新的终端，执行下面的命令，查询系统中正在运行的 eBPF 程序：</p><pre><code class="language-bash"># sudo bpftool prog list
89: kprobe  name hello_world  tag 38dd440716c4900f  gpl
      loaded_at 2021-11-27T13:20:45+0000  uid 0
      xlated 104B  jited 70B  memlock 4096B
      btf_id 131
      pids python3(152027)
</code></pre><p>输出中，89 是这个 eBPF 程序的编号，kprobe 是程序的类型，而 hello_world 是程序的名字。</p><p>有了 eBPF 程序编号之后，执行下面的命令就可以导出这个 eBPF 程序的指令（注意把 89 替换成你查询到的编号）：</p><pre><code class="language-bash">sudo bpftool prog dump xlated id 89
</code></pre><p>你会看到如下所示的输出：</p><pre><code class="language-bash">int hello_world(void * ctx):
; int hello_world(void *ctx)
&nbsp; &nbsp;0: (b7) r1 = 33&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; /* ! */
; ({ char _fmt[] = "Hello, World!"; bpf_trace_printk_(_fmt, sizeof(_fmt)); });
&nbsp; &nbsp;1: (6b) *(u16 *)(r10 -4) = r1
&nbsp; &nbsp;2: (b7) r1 = 1684828783&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; /*&nbsp;dlro */
&nbsp; &nbsp;3: (63) *(u32 *)(r10 -8) = r1
&nbsp; &nbsp;4: (18) r1 = 0x57202c6f6c6c6548&nbsp; /*&nbsp;W ,olleH */
&nbsp; &nbsp;6: (7b) *(u64 *)(r10 -16) = r1
&nbsp; &nbsp;7: (bf) r1 = r10
;
&nbsp; &nbsp;8: (07) r1 += -16
; ({ char _fmt[] = "Hello, World!"; bpf_trace_printk_(_fmt, sizeof(_fmt)); });
&nbsp; &nbsp;9: (b7) r2 = 14
&nbsp; 10: (85) call bpf_trace_printk#-61616
; return 0;
&nbsp; 11: (b7) r0 = 0
&nbsp; 12: (95) exit
</code></pre><p>其中，分号开头的部分，正是我们前面写的 C 代码，而其他行则是具体的 BPF 指令。具体每一行的 BPF 指令又分为三部分：</p><ul>
<li>第一部分，冒号前面的数字 0-12 ，代表 BPF 指令行数；</li>
<li>第二部分，括号中的16进制数值，表示 BPF 指令码。它的具体含义你可以参考 <a href="https://github.com/iovisor/bpf-docs/blob/master/eBPF.md">IOVisor BPF 文档</a>，比如第 0 行的 0xb7 表示为 64 位寄存器赋值。</li>
<li>第三部分，括号后面的部分，就是 BPF 指令的伪代码。</li>
</ul><p>结合前面讲述的各个寄存器的作用，不难理解这些 BPF 指令的含义：</p><ul>
<li>第0-8行，借助 R10 寄存器从栈中把字符串 “Hello, World!” 读出来，并放入 R1 寄存器中；</li>
<li>第9行，向 R2 寄存器写入字符串的长度 14（即代码注释里面的  <code>sizeof(_fmt)</code> ）；</li>
<li>第10行，调用 BPF 辅助函数  <code>bpf_trace_printk</code>  输出字符串；</li>
<li>第11行，向 R0 寄存器写入0，表示程序的返回值是0；</li>
<li>最后一行，程序执行成功退出。</li>
</ul><p>总结起来，<strong>这些指令先通过 R1 和 R2 寄存器设置了</strong> <code>bpf_trace_printk</code> <strong>的参数，然后调用</strong> <code>bpf_trace_printk</code> <strong>函数输出字符串，最后再通过 R0 寄存器返回成功。</strong></p><p>实际上，你也可以通过类似的 <a href="https://man7.org/linux/man-pages/man2/bpf.2.html#EXAMPLES">BPF 指令</a>来开发 eBPF 程序（具体指令的定义，请参考 <a href="https://elixir.bootlin.com/linux/v5.4/source/include/uapi/linux/bpf_common.h">include/uapi/linux/bpf_common.h</a> 以及 <a href="https://elixir.bootlin.com/linux/v5.4/source/include/uapi/linux/bpf.h">include/uapi/linux/bpf.h</a>），不过通常并不推荐你这么做。跟一开始的 C 程序相比，你会发现 BPF 指令的可读性和可维护性明显要差得多。所以，我建议你还是使用 C 语言来开发 eBPF 程序，而只把&nbsp;BPF 指令作为排查 eBPF 程序疑难杂症时的参考。</p><p>这里，我来简单讲讲&nbsp;BPF 指令加载后是如何运行的。当这些 BPF 指令加载到内核后， BPF 即时编译器会将其编译成本地机器指令，最后才会执行编译后的机器指令：</p><pre><code class="language-bash"># bpftool prog dump jited id 89
int hello_world(void * ctx):
bpf_prog_38dd440716c4900f_hello_world:
; int hello_world(void *ctx)
&nbsp; &nbsp;0:	nopl&nbsp; &nbsp;0x0(%rax,%rax,1)
&nbsp; &nbsp;5:	xchg&nbsp; &nbsp;%ax,%ax
&nbsp; &nbsp;7:	push&nbsp; &nbsp;%rbp
&nbsp; &nbsp;8:	mov&nbsp; &nbsp; %rsp,%rbp
&nbsp; &nbsp;b:	sub&nbsp; &nbsp; $0x10,%rsp
&nbsp; 12:	mov&nbsp; &nbsp; $0x21,%edi
; ({ char _fmt[] = "Hello, World!"; bpf_trace_printk_(_fmt, sizeof(_fmt)); });
&nbsp; 17:	mov&nbsp; &nbsp; %di,-0x4(%rbp)
&nbsp; 1b:	mov&nbsp; &nbsp; $0x646c726f,%edi
&nbsp; 20:	mov&nbsp; &nbsp; %edi,-0x8(%rbp)
&nbsp; 23:	movabs $0x57202c6f6c6c6548,%rdi
&nbsp; 2d:	mov&nbsp; &nbsp; %rdi,-0x10(%rbp)
&nbsp; 31:	mov&nbsp; &nbsp; %rbp,%rdi
;
&nbsp; 34:	add&nbsp; &nbsp; $0xfffffffffffffff0,%rdi
; ({ char _fmt[] = "Hello, World!"; bpf_trace_printk_(_fmt, sizeof(_fmt)); });
&nbsp; 38:	mov&nbsp; &nbsp; $0xe,%esi
&nbsp; 3d:	call&nbsp; &nbsp;0xffffffffd8c7e834
; return 0;
&nbsp; 42:	xor&nbsp; &nbsp; %eax,%eax
&nbsp; 44:	leave
&nbsp; 45:	ret
</code></pre><p>这些机器指令的含义跟前面的 BPF 指令是类似的，但具体的指令和寄存器都换成了 x86 的格式。你不需要掌握这些机器指令的具体含义，只要知道查询的具体方法就足够了。这是因为，就像你曾接触过的其他高级语言一样，在实际的 eBPF 使用过程中，并不需要直接使用机器指令，而是 eBPF 虚拟机帮你自动完成了转换。</p><h2>eBPF 程序是什么时候执行的？</h2><p>到这里，我想你已经理解了 BPF 指令的具体格式，以及它与  C 源代码之间的对应关系。不过，这个 eBPF 程序到底是什么时候执行的呢？接下来，我们再一起看看 BPF 指令的加载和执行过程。</p><p>在上一讲中我提到，BCC 负责了 eBPF 程序的编译和加载过程。因而，要了解 BPF 指令的加载过程，就可以从 BCC 执行 eBPF 程序的过程入手。</p><p>那么，怎么才能查看到 BCC 的执行过程呢？我想，你一定想到了，那就是跟踪它的系统调用过程。</p><p>首先，我们打开一个终端，执行下面的命令：</p><pre><code class="language-bash"># -ebpf表示只跟踪bpf系统调用
sudo strace -v -f -ebpf ./hello.py
</code></pre><p>稍等一会，你会看到如下的输出：</p><pre><code class="language-bash">bpf(BPF_PROG_LOAD,
    {
        prog_type=BPF_PROG_TYPE_KPROBE,
        insn_cnt=13,
        insns=[
            {code=BPF_ALU64|BPF_K|BPF_MOV, dst_reg=BPF_REG_1, src_reg=BPF_REG_0, off=0, imm=0x21},
            {code=BPF_STX|BPF_H|BPF_MEM, dst_reg=BPF_REG_10, src_reg=BPF_REG_1, off=-4, imm=0},
            {code=BPF_ALU64|BPF_K|BPF_MOV, dst_reg=BPF_REG_1, src_reg=BPF_REG_0, off=0, imm=0x646c726f},
            {code=BPF_STX|BPF_W|BPF_MEM, dst_reg=BPF_REG_10, src_reg=BPF_REG_1, off=-8, imm=0},
            {code=BPF_LD|BPF_DW|BPF_IMM, dst_reg=BPF_REG_1, src_reg=BPF_REG_0, off=0, imm=0x6c6c6548},
            {code=BPF_LD|BPF_W|BPF_IMM, dst_reg=BPF_REG_0, src_reg=BPF_REG_0, off=0, imm=0x57202c6f},
            {code=BPF_STX|BPF_DW|BPF_MEM, dst_reg=BPF_REG_10, src_reg=BPF_REG_1, off=-16, imm=0},
            {code=BPF_ALU64|BPF_X|BPF_MOV, dst_reg=BPF_REG_1, src_reg=BPF_REG_10, off=0, imm=0},
            {code=BPF_ALU64|BPF_K|BPF_ADD, dst_reg=BPF_REG_1, src_reg=BPF_REG_0, off=0, imm=0xfffffff0},
            {code=BPF_ALU64|BPF_K|BPF_MOV, dst_reg=BPF_REG_2, src_reg=BPF_REG_0, off=0, imm=0xe},
            {code=BPF_JMP|BPF_K|BPF_CALL, dst_reg=BPF_REG_0, src_reg=BPF_REG_0, off=0, imm=0x6},
            {code=BPF_ALU64|BPF_K|BPF_MOV, dst_reg=BPF_REG_0, src_reg=BPF_REG_0, off=0, imm=0},
            {code=BPF_JMP|BPF_K|BPF_EXIT, dst_reg=BPF_REG_0, src_reg=BPF_REG_0, off=0, imm=0}
        ],
        prog_name="hello_world",
        ...
    },
    128) = 4
</code></pre><p>这些参数看起来很复杂，但实际上，如果你查询  <code>bpf</code> 系统调用的格式（执行  <code>man bpf</code> 命令），就可以发现，它实际上只需要三个参数：</p><pre><code class="language-bash">int bpf(int cmd, union bpf_attr *attr, unsigned int size);
</code></pre><p>对应前面的 strace 输出结果，这三个参数的具体含义如下。</p><ul>
<li>第一个参数是  <code>BPF_PROG_LOAD</code> ， 表示加载 BPF 程序。</li>
<li>第二个参数是  <code>bpf_attr</code>  类型的结构体，表示 BPF 程序的属性。其中，有几个需要你留意的参数，比如：
<ul>
<li><code>prog_type</code>  表示 BPF 程序的类型，这儿是  <code>BPF_PROG_TYPE_KPROBE</code> ，跟我们Python 代码中的  <code>attach_kprobe</code>  一致；</li>
<li><code>insn_cnt</code>  (instructions count) 表示指令条数；</li>
<li><code>insns</code>  (instructions) 包含了具体的每一条指令，这儿的 13 条指令跟我们前面  <code>bpftool prog dump</code>  的结果是一致的（具体的指令格式，你可以参考内核中 <a href="https://elixir.bootlin.com/linux/v5.4/source/include/uapi/linux/bpf.h#L65">bpf_insn</a> 的定义）；</li>
<li><code>prog_name</code>  则表示 BPF 程序的名字，即  <code>hello_world</code> 。</li>
</ul>
</li>
<li>第三个参数 128 表示属性的大小。</li>
</ul><p>到这里，我们已经了解了 bpf 系统调用的基本格式。对于  <code>bpf</code>  系统调用在内核中的实现原理，你并不需要详细了解。我们只要知道它的具体功能，就可以掌握 eBPF 的核心原理了。当然，如果你对它的实现方法有兴趣的话，可以参考内核源码 kernel/bpf/syscall.c 中 <a href="https://elixir.bootlin.com/linux/v5.4/source/kernel/bpf/syscall.c#L2837">SYSCALL_DEFINE3</a> 的实现。</p><p>BPF 程序加载到内核后，并不会立刻执行，那么它什么时候才会执行呢？这里，回想一下我在 <a href="https://time.geekbang.org/column/article/479384">01 讲</a> 中提到的 eBPF 的基本原理：</p><blockquote>
<p>eBPF 程序并不像常规的线程那样，启动后就一直运行在那里，它需要事件触发后才会执行。这些事件包括系统调用、内核跟踪点、内核函数和用户态函数的调用退出、网络事件，等等。</p>
</blockquote><p>对于我们的 Hello World 来说，由于调用了  <code>attach_kprobe</code>  函数，很明显，这是一个内核跟踪事件：</p><pre><code class="language-bash">b.attach_kprobe(event="do_sys_openat2", fn_name="hello_world")
</code></pre><p>所以，除了把 eBPF 程序加载到内核之外，还需要把加载后的程序跟具体的内核函数调用事件进行绑定。在 eBPF 的实现中，诸如内核跟踪（kprobe）、用户跟踪（uprobe）等的事件绑定，都是通过  <code>perf_event_open()</code>  来完成的。</p><p>为什么这么说呢？我们再用  <code>strace</code>  来确认一下。把前面  <code>strace</code>  命令中的  <code>-ebpf</code>  参数去掉，重新执行：</p><pre><code class="language-bash">sudo strace -v -f ./hello.py
</code></pre><p>忽略无关的输出后，你会发现如下的系统调用：</p><pre><code class="language-c++">...
/* 1) 加载BPF程序 */
bpf(BPF_PROG_LOAD,...) = 4
...

/* 2）查询事件类型 */
openat(AT_FDCWD, "/sys/bus/event_source/devices/kprobe/type", O_RDONLY) = 5
read(5, "6\n", 4096)                    = 2
close(5)                                = 0
...

/* 3）创建性能监控事件 */
perf_event_open(
    {
        type=0x6 /* PERF_TYPE_??? */,
        size=PERF_ATTR_SIZE_VER7,
        ...
        wakeup_events=1,
        config1=0x7f275d195c50,
        ...
    },
    -1,
    0,
    -1,
    PERF_FLAG_FD_CLOEXEC) = 5

/* 4）绑定BPF到kprobe事件 */
ioctl(5, PERF_EVENT_IOC_SET_BPF, 4)     = 0
...
</code></pre><p>从输出中，你可以看出 BPF 与性能事件的绑定过程分为以下几步：</p><ul>
<li>首先，借助 bpf 系统调用，加载 BPF 程序，并记住返回的文件描述符；</li>
<li>然后，查询 kprobe 类型的事件编号。BCC 实际上是通过  <code>/sys/bus/event_source/devices/kprobe/type</code> 来查询的；</li>
<li>接着，调用  <code>perf_event_open</code>  创建性能监控事件。比如，事件类型（type 是上一步查询到的 6）、事件的参数（ <code>config1 包含了内核函数 do_sys_openat2</code> ）等；</li>
<li>最后，再通过  <code>ioctl</code>  的  <code>PERF_EVENT_IOC_SET_BPF</code>  命令，将 BPF 程序绑定到性能监控事件。</li>
</ul><p>对于绑定性能监控（perf event）的内核实现原理，你也不需要详细了解，只需要知道它的具体功能，就足够我们掌握 eBPF 了。如果你对它的实现方法有兴趣的话，可以参考内核源码 <a href="https://elixir.bootlin.com/linux/v5.4/source/kernel/events/core.c#L9039">perf_event_set_bpf_prog</a> 的实现；而最终性能监控调用 BPF 程序的实现，则可以参考内核源码 <a href="https://elixir.bootlin.com/linux/v5.4/source/kernel/trace/trace_kprobe.c#L1351">kprobe_perf_func</a> 的实现。</p><h2>小结</h2><p>今天，我带你一起梳理了 eBPF 在内核中的实现原理，并以上一讲的 Hello World 程序为例，借助 bpftool、strace 等工具，带你观察了 BPF 指令的具体格式。</p><p>然后，我们从 BCC 执行 eBPF 程序的过程入手，一起看了BPF 指令的加载和执行过程。用高级语言开发的 eBPF 程序，需要首先编译为 BPF 字节码（即 BPF 指令），然后借助  <code>bpf</code>  系统调用加载到内核中，最后再通过性能监控等接口，与具体的内核事件进行绑定。这样，内核的性能监控模块才会在内核事件发生时，自动执行我们开发的 eBPF 程序。</p><h2>思考题</h2><p>最后，我想邀请你来聊一聊这两个问题。</p><ol>
<li>你通常是如何快速理解一门新技术的运行原理的？</li>
<li>在今天的内容中，我使用 strace 跟踪 BCC 程序，进而找到了相关的系统调用。那么，有没有可能直接使用 BCC 来跟踪  <code>bpf</code> 系统调用呢？如果你的答案是肯定的，可以试着把它开发出来，并在评论区分享你的实践经验。</li>
</ol><p>欢迎在留言区和我讨论，也欢迎把这节课分享给你的同事、朋友。我们一起在实战中演练，在交流中进步。</p>
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
  <div class="_2_QraFYR_0">追踪 bpf 系统调用，借助 BCC 宏定义 TRACEPOINT_PROBE(category, event) 比较方便，例如：<br><br>-------------- example.c ----------------<br><br>TRACEPOINT_PROBE(syscalls, sys_enter_bpf)<br>{<br>    bpf_trace_printk(&quot;%d\\n&quot;, args-&gt;cmd);<br>    return 0;<br>}<br><br>-------------- example.py -----------------<br><br>#!&#47;usr&#47;bin&#47;env python3<br><br>from bcc import BPF<br><br># load BPF program<br>b = BPF(src_file=&quot;example.c&quot;)<br>b.trace_print()<br><br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢分享实践经验👍，BCC内置的很多宏的确非常方便。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-24 10:19:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/58/eb/cf3608bd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>18646333118</span>
  </div>
  <div class="_2_QraFYR_0">解决方法:<br>{<br>    &quot;features&quot;: {<br>        &quot;libbfd&quot;: false<br>    }<br>}<br><br>uname -r<br>5.13.0-19-generic<br><br>apt-cache search linux-source<br>apt install linux-source-5.13.0<br><br>cd  &#47;usr&#47;src&#47;<br>tar -jxvf linux-source-5.13.0.tar.bz2<br><br>apt install libelf-dev<br>cd linux-source-5.13.0&#47;tools<br>make -C  bpf&#47;bpftool<br>.&#47;bpf&#47;bpftool&#47;bpftool version -p<br>{<br>    &quot;version&quot;: &quot;5.13.19&quot;,<br>    &quot;features&quot;: {<br>        &quot;libbfd&quot;: true,<br>        &quot;skeletons&quot;: true<br>    }<br>}<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍 谢谢分享源码编译bpftool的详细步骤！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-09 16:01:10</div>
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
  <div class="_2_QraFYR_0">你通常是如何快速理解一门新技术的运行原理的？<br>--- 看一下官方文档，了解体系架构，多看几遍。买书看感觉也是一个快速入门的方法。<br>但是对于没有编程经验，对于 字节码、cpu寄存器、jit ，编译器的理解还是很抽象，学到这里还是有点晕。感觉还是要把这课程从头再看一遍。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢分享。我的理解是，要深入一门技术的每个细节需要看很多书籍，但一开始要有个侧重点，不要发散的太广了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-28 09:55:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_b84e15</span>
  </div>
  <div class="_2_QraFYR_0">倪老师您好，我看hello_world的参数列表是(void * ctx)，而有的例子里参数是这样的：int hello_world(struct pt_regs *ctx, int dfd, const char __user * filename, struct open_how *how)，请问怎么确定参数的个数和参数的类型呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-25 09:40:12</div>
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
  <div class="_2_QraFYR_0">1、你通常是如何快速理解一门新技术的运行原理的？<br>Get Essentials， ADEPT五步法：类比，画图，例子，文字说明，定义。<br><br>剩下的就是根据需要侧重地深入到细节。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍 </p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-26 23:12:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/e8/ac/7324d5ca.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>七里</span>
  </div>
  <div class="_2_QraFYR_0">请问，不能执行&#39;bpftool prog dump jited id 78&#39;是怎么回事？bpf相关的包都按照上一讲的提示按照上了<br><br>root@maqi-ubt:~# bpftool prog dump xlated id 78<br>int hello_world(void * ctx):<br>; int hello_world(void *ctx)<br>   0: (b7) r1 = 33<br>; ({ char _fmt[] = &quot;Hello, World!&quot;; bpf_trace_printk_(_fmt, sizeof(_fmt)); });<br>   1: (6b) *(u16 *)(r10 -4) = r1<br>   2: (b7) r1 = 1684828783<br>   3: (63) *(u32 *)(r10 -8) = r1<br>   4: (18) r1 = 0x57202c6f6c6c6548<br>   6: (7b) *(u64 *)(r10 -16) = r1<br>   7: (bf) r1 = r10<br>;<br>   8: (07) r1 += -16<br>; ({ char _fmt[] = &quot;Hello, World!&quot;; bpf_trace_printk_(_fmt, sizeof(_fmt)); });<br>   9: (b7) r2 = 14<br>  10: (85) call bpf_trace_printk#-63952<br>; return 0;<br>  11: (b7) r0 = 0<br>  12: (95) exit<br>root@maqi-ubt:~#<br>root@maqi-ubt:~# bpftool prog dump jited id 78<br>Error: No libbfd support</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以运行下面的命令来查询bpftool支持的特性：<br><br>sudo bpftool version -p<br><br>如果出现下面的输出说明发行版自带的bpftool默认不支持libbfd，需要下载内核源码并安装binutils-dev之后重新编译bpftool：<br>{<br>    &quot;features&quot;: {<br>        &quot;libbfd&quot;: false<br>    }<br>}</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-02 17:22:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJGXndj5N66z9BL1ic9GibZzWWgoVeWaWTL2XUnCYic7iba2kAEvN9WfjmlXELD5lqt8IJ1P023N5ZWicg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_f93234</span>
  </div>
  <div class="_2_QraFYR_0"># strace -v -f -ebpf .&#47;hello.py<br>strace: exec: Exec format error<br>+++ exited with 1 +++<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-23 09:49:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/20/9d/61efb39e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>赵杨健</span>
  </div>
  <div class="_2_QraFYR_0">有学习群吗？遇到一些环境问题</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-14 20:12:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/31/42/a1/716f3c50.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>22</span>
  </div>
  <div class="_2_QraFYR_0">老师，sudo strace -v -f -ebpf .&#47;hello.py<br>strace: exec: 可执行文件格式错误<br>+++ exited with 1 +++<br>想请问一下这是什么原因啊？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 去掉strace，只执行 .&#47;hello.py 可以正常执行吗？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-25 11:12:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/31/42/a1/716f3c50.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>22</span>
  </div>
  <div class="_2_QraFYR_0">追踪系统调用，显示没有权限该怎么解决啊<br>sudo strace -v -f -ebpf .&#47;hello.py<br>strace: exec: 权限不够<br>+++ exited with 1 +++<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 去掉strace，只执行 .&#47;hello.py 可以正常执行吗？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-25 09:24:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_5ada4a</span>
  </div>
  <div class="_2_QraFYR_0">strace .&#47;hello.py<br>execve(&quot;.&#47;hello.py&quot;, [&quot;.&#47;hello.py&quot;], 0x7ffed513d760 &#47;* 30 vars *&#47;) = -1 EACCES (Permission denied)<br>strace: exec: Permission denied<br>+++ exited with 1 +++<br><br>哪位大佬帮忙看看，是 root 用户，ls -l hello.py 的结果是<br>-rw-r--r-- 1 root root 147 Nov  3 14:44 .&#47;hello.py</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-03 15:49:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_5ada4a</span>
  </div>
  <div class="_2_QraFYR_0">openat(AT_FDCWD, &quot;&#47;sys&#47;bus&#47;event_source&#47;devices&#47;kprobe&#47;type&quot;, O_RDONLY) = 5<br>read(5, &quot;6\n&quot;, 4096) = 2<br>close(5) = 0<br><br>为什么下面的 type = 0x6 ，对不上诶<br>另外这个 read 作用是？<br><br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-01 11:06:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/c5/a1/f6a2d80d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>穿靴子的outman</span>
  </div>
  <div class="_2_QraFYR_0">1.你通常是如何快速理解一门新技术的运行原理的？<br><br>a.我通常首先搞清楚这门新技术想解决什么问题等背景知识.谁在什么场景下,会遇到什么问题.人永远是第一位的.<br>c.按照helloworld等例程实操一遍.<br>b.根据实际体验,重新思考这门技术,有哪些概念,组件,组件之间的关系是怎样.各个组件的输入输出怎么样的,怎么组合起来解决了这个问题.此时可以借助时序图,流程图等工具.加深理解.</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-06 01:58:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/99/f4/e0484cac.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>崔伟协</span>
  </div>
  <div class="_2_QraFYR_0">ebpf是图灵完备的吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不是的，eBPF有非常多的限制，内核需要验证所有执行路径是确定的才可以加载（如果去掉验证，只考虑内核中的eBPF虚拟机，我的理解它是图灵完备的）。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-25 15:51:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_ee7838</span>
  </div>
  <div class="_2_QraFYR_0">root@iZbp18xpd2ia89yqcrhi48Z:~&#47;eBPF# ll<br>total 24<br>drwxr-xr-x 2 root root 4096 Sep 20 20:03 .&#47;<br>drwx------ 8 root root 4096 Sep 20 20:49 ..&#47;<br>-rw-r--r-- 1 root root   85 Sep 20 19:14 hello.c<br>-rwxr-xr-x 1 root root  274 Sep 20 19:18 hello.py*<br>-rw-r--r-- 1 root root  753 Sep 20 20:03 trace-open.c<br>-rw-r--r-- 1 root root  725 Sep 20 20:02 trace-open.py<br>root@iZbp18xpd2ia89yqcrhi48Z:~&#47;eBPF# sudo strace -v -f -ebpf .&#47;hello.py<br>strace: exec: Exec format error<br>+++ exited with 1 +++<br>为什么我的strace跟踪不了ebpf数据包，strace版本是5.16的，我查看命令帮助似乎并没有-ebpf的命令参数</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-20 21:08:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1c/bb/50/c8ebd5e1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>MoGeJiEr🐔</span>
  </div>
  <div class="_2_QraFYR_0">“R1-R5寄存器用于函数调用的参数，因此这里的函数调用参数不能超过5个”是指bpf-helper函数的参数不能超过5个嘛？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-14 22:31:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/5f/43/3799a0f3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>magina</span>
  </div>
  <div class="_2_QraFYR_0">eBPF 运行时图中，不支持JIT是什么过程呢？文中也没有提什么情况下走JIT，什么情况不走JIT？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-25 11:34:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/17/18/e4382a8e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>有识之士</span>
  </div>
  <div class="_2_QraFYR_0">我的报错<br><br>root@3358d0bf1a7b:~# bpftool prog list<br>WARNING: bpftool not found for kernel 5.10.25<br><br>  You may need to install the following packages for this specific kernel:<br>    linux-tools-5.10.25-linuxkit<br>    linux-cloud-tools-5.10.25-linuxkit<br><br>  You may also want to install one of the following packages to keep up to date:<br>    linux-tools-linuxkit<br>    linux-cloud-tools-linuxkit<br><br>root@3358d0bf1a7b:~# bpftool<br>WARNING: bpftool not found for kernel 5.10.25<br><br>  You may need to install the following packages for this specific kernel:<br>    linux-tools-5.10.25-linuxkit<br>    linux-cloud-tools-5.10.25-linuxkit<br><br>  You may also want to install one of the following packages to keep up to date:<br>    linux-tools-linuxkit<br>    linux-cloud-tools-linuxkit</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-22 22:15:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/17/18/e4382a8e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>有识之士</span>
  </div>
  <div class="_2_QraFYR_0">请教选，这个ebpf 虚拟机功能类似  java jvm 编译成class 字节码？还是有很大的不同点？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-22 21:17:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/28/04/7a/1306bf9c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>cmz</span>
  </div>
  <div class="_2_QraFYR_0">ebpf为什么要设计成虚拟机的形式</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-07 16:19:59</div>
  </div>
</div>
</div>
</li>
</ul>