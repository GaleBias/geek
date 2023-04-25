<audio title="05 _ 编程接口：eBPF程序是怎么跟内核进行交互的？" src="https://static001.geekbang.org/resource/audio/53/ef/53599469acc12bcb9d5452680bc358ef.mp3" controls="controls"></audio> 
<p>你好，我是倪朋飞。</p><p>上一讲，我带你一起梳理了 eBPF 在内核中的实现原理，以及 BPF 指令的具体格式。</p><p>用高级语言开发的 eBPF 程序，需要首先编译为 BPF 字节码，然后借助 bpf 系统调用加载到内核中，最后再通过性能监控等接口与具体的内核事件进行绑定。这样，内核的性能监控模块才会在内核事件发生时，自动执行我们开发的 eBPF 程序。</p><p>那么，eBPF 程序到底是如何跟内核事件进行绑定的？又该如何跟内核中的其他模块进行交互呢？今天，我就带你一起看看 eBPF 程序的编程接口。</p><h2>BPF 系统调用</h2><p>如下图（图片来自<a href="https://www.brendangregg.com/ebpf.html">brendangregg.com</a>）所示，一个完整的 eBPF 程序通常包含用户态和内核态两部分。其中，用户态负责 eBPF 程序的加载、事件绑定以及 eBPF 程序运行结果的汇总输出；内核态运行在 eBPF 虚拟机中，负责定制和控制系统的运行状态。</p><p><img src="https://static001.geekbang.org/resource/image/a7/6a/a7165eea1fd9fc24090a3a1e8987986a.png?wh=1500x550" alt="图片"></p><p>对于用户态程序来说，我想你已经了解，<strong>它们与内核进行交互时必须要通过系统调用来完成</strong>。而对应到 eBPF 程序中，我们最常用到的就是 <a href="https://man7.org/linux/man-pages/man2/bpf.2.html">bpf 系统调用</a>。</p><p>在命令行中输入 <code>man bpf</code>&nbsp;，就可以查询到 BPF 系统调用的调用格式：</p><pre><code class="language-plain">#include &lt;linux/bpf.h&gt;

int bpf(int cmd, union bpf_attr *attr, unsigned int size);
</code></pre><!-- [[[read_end]]] --><p>BPF 系统调用接受三个参数：</p><ul>
<li>第一个，cmd ，代表操作命令，比如上一讲中我们看到的 BPF_PROG_LOAD 就是加载 eBPF 程序；</li>
<li>第二个，attr，代表 bpf_attr 类型的 eBPF 属性指针，不同类型的操作命令需要传入不同的属性参数；</li>
<li>第三个，size ，代表属性的大小。</li>
</ul><p>注意，<strong>不同版本的内核所支持的 BPF 命令是不同的</strong>，具体支持的命令列表可以参考内核头文件 include/uapi/linux/bpf.h 中  <code>bpf_cmd</code> 的定义。比如，v5.13 内核已经支持 <a href="https://elixir.bootlin.com/linux/v5.13/source/include/uapi/linux/bpf.h#L828">36 个 BPF 命令</a>：</p><pre><code class="language-plain">enum bpf_cmd {
&nbsp; BPF_MAP_CREATE,
&nbsp; BPF_MAP_LOOKUP_ELEM,
&nbsp; BPF_MAP_UPDATE_ELEM,
&nbsp; BPF_MAP_DELETE_ELEM,
&nbsp; BPF_MAP_GET_NEXT_KEY,
&nbsp; BPF_PROG_LOAD,
&nbsp; BPF_OBJ_PIN,
&nbsp; BPF_OBJ_GET,
&nbsp; BPF_PROG_ATTACH,
&nbsp; BPF_PROG_DETACH,
&nbsp; BPF_PROG_TEST_RUN,
&nbsp; BPF_PROG_GET_NEXT_ID,
&nbsp; BPF_MAP_GET_NEXT_ID,
&nbsp; BPF_PROG_GET_FD_BY_ID,
&nbsp; BPF_MAP_GET_FD_BY_ID,
&nbsp; BPF_OBJ_GET_INFO_BY_FD,
&nbsp; BPF_PROG_QUERY,
&nbsp; BPF_RAW_TRACEPOINT_OPEN,
&nbsp; BPF_BTF_LOAD,
&nbsp; BPF_BTF_GET_FD_BY_ID,
&nbsp; BPF_TASK_FD_QUERY,
&nbsp; BPF_MAP_LOOKUP_AND_DELETE_ELEM,
&nbsp; BPF_MAP_FREEZE,
&nbsp; BPF_BTF_GET_NEXT_ID,
&nbsp; BPF_MAP_LOOKUP_BATCH,
&nbsp; BPF_MAP_LOOKUP_AND_DELETE_BATCH,
&nbsp; BPF_MAP_UPDATE_BATCH,
&nbsp; BPF_MAP_DELETE_BATCH,
&nbsp; BPF_LINK_CREATE,
&nbsp; BPF_LINK_UPDATE,
&nbsp; BPF_LINK_GET_FD_BY_ID,
&nbsp; BPF_LINK_GET_NEXT_ID,
&nbsp; BPF_ENABLE_STATS,
&nbsp; BPF_ITER_CREATE,
&nbsp; BPF_LINK_DETACH,
&nbsp; BPF_PROG_BIND_MAP,
};
</code></pre><p>为了方便你掌握，我把用户程序中常用的命令整理成了一个表格，你可以在需要时参考：<br>
<img src="https://static001.geekbang.org/resource/image/7c/88/7cc33d0bdd8a3ba0dda7f533f3375b88.jpg?wh=1920x1080" alt="图片"></p><h2>BPF 辅助函数</h2><p>说完用户态程序的 bpf 系统调用格式，我们再来看看内核态的 eBPF 程序。</p><p>eBPF 程序并不能随意调用内核函数，因此，内核定义了一系列的辅助函数，用于 eBPF 程序与内核其他模块进行交互。比如，上一讲的 Hello World 示例中使用的 bpf_trace_printk() 就是最常用的一个辅助函数，用于向调试文件系统（/sys/kernel/debug/tracing/trace_pipe）写入调试信息。</p><p>这里补充一个知识点：从内核 5.13 版本开始，部分内核函数（如&nbsp;<code>tcp_slow_start()</code>、<code>tcp_reno_ssthresh()</code>&nbsp;等）也可以被 BPF 程序直接调用了，具体你可以查看<a href="https://lwn.net/Articles/856005">这个链接</a>。 不过，这些函数只能在 TCP 拥塞控制算法的 BPF 程序中调用，所以本课程不会过多展开。</p><p>需要注意的是，并不是所有的辅助函数都可以在 eBPF 程序中随意使用，不同类型的 eBPF 程序所支持的辅助函数是不同的。比如，对于 Hello World 示例这类内核探针（kprobe）类型的 eBPF 程序，你可以在命令行中执行  <code>bpftool feature probe</code>&nbsp;，来查询当前系统支持的辅助函数列表：</p><pre><code class="language-plain">$ bpftool feature probe
...
eBPF helpers supported for program type kprobe:
	- bpf_map_lookup_elem
	- bpf_map_update_elem
	- bpf_map_delete_elem
	- bpf_probe_read
	- bpf_ktime_get_ns
	- bpf_get_prandom_u32
	- bpf_get_smp_processor_id
	- bpf_tail_call
	- bpf_get_current_pid_tgid
	- bpf_get_current_uid_gid
	- bpf_get_current_comm
	- bpf_perf_event_read
	- bpf_perf_event_output
	- bpf_get_stackid
	- bpf_get_current_task
	- bpf_current_task_under_cgroup
	- bpf_get_numa_node_id
	- bpf_probe_read_str
	- bpf_perf_event_read_value
	- bpf_override_return
	- bpf_get_stack
	- bpf_get_current_cgroup_id
	- bpf_map_push_elem
	- bpf_map_pop_elem
	- bpf_map_peek_elem
	- bpf_send_signal
	- bpf_probe_read_user
	- bpf_probe_read_kernel
	- bpf_probe_read_user_str
	- bpf_probe_read_kernel_str
...
</code></pre><p>对于这些辅助函数的详细定义，你可以在命令行中执行  <code>man bpf-helpers</code>&nbsp;，或者参考内核头文件 <a href="https://elixir.bootlin.com/linux/v5.13/source/include/uapi/linux/bpf.h#L1463">include/uapi/linux/bpf.h</a>&nbsp;，来查看它们的详细定义和使用说明。为了方便你掌握，我把常用的辅助函数整理成了一个表格，你可以在需要时参考：<br>
<img src="https://static001.geekbang.org/resource/image/0b/cb/0b3edac18276a1236dde7135b961d8cb.jpg?wh=1920x1080" alt="图片"></p><p>这其中，需要你特别注意的是以<code>bpf_probe_read</code>  开头的一系列函数。我在上一讲中已经提到，eBPF 内部的内存空间只有寄存器和栈。所以，要访问其他的内核空间或用户空间地址，就需要借助&nbsp;<code>bpf_probe_read</code>  这一系列的辅助函数。这些函数会进行安全性检查，并禁止缺页中断的发生。</p><p>而在 eBPF 程序需要大块存储时，就不能像常规的内核代码那样去直接分配内存了，而是必须通过 BPF 映射（BPF Map）来完成。接下来，我带你看看 BPF 映射的具体原理。</p><h2>BPF 映射</h2><p>BPF 映射用于提供大块的键值存储，这些存储可被用户空间程序访问，进而获取 eBPF 程序的运行状态。eBPF 程序最多可以访问 64 个不同的 BPF 映射，并且不同的 eBPF 程序也可以通过相同的 BPF 映射来共享它们的状态。下图（图片来自<a href="https://docs.cilium.io/en/stable/bpf">docs.cilium.io</a>）展示了&nbsp;BPF 映射的基本使用方法。</p><p><img src="https://static001.geekbang.org/resource/image/d8/11/d87b409fa85d3a07973a8689b228cf11.png?wh=576x383" alt="图片" title="BPF 映射"></p><p>在前面的 BPF 系统调用和辅助函数小节中，你也看到，有很多系统调用命令和辅助函数都是用来访问 BPF 映射的。我相信细心的你已经发现了：BPF 辅助函数中并没有 BPF 映射的创建函数，BPF 映射只能通过用户态程序的系统调用来创建。比如，你可以通过下面的示例代码来创建一个 BPF 映射，并返回映射的文件描述符：</p><pre><code class="language-c++">int bpf_create_map(enum bpf_map_type map_type,
		&nbsp; &nbsp;unsigned int key_size,
		&nbsp; &nbsp;unsigned int value_size, unsigned int max_entries)
{
&nbsp; union bpf_attr attr = {
&nbsp; &nbsp; .map_type = map_type,
&nbsp; &nbsp; .key_size = key_size,
&nbsp; &nbsp; .value_size = value_size,
&nbsp; &nbsp; .max_entries = max_entries
&nbsp; };
&nbsp; return bpf(BPF_MAP_CREATE, &amp;attr, sizeof(attr));
}
</code></pre><p>这其中，最关键的是设置映射的类型。内核头文件<a href="https://elixir.bootlin.com/linux/v5.13/source/include/uapi/linux/bpf.h#L867"> include/uapi/linux/bpf.h </a>中的  <code>bpf_map_type</code> 定义了所有支持的映射类型，你可以使用如下的 bpftool 命令，来查询当前系统支持哪些映射类型：</p><pre><code class="language-bash">$ bpftool feature probe | grep map_type
eBPF map_type hash is available
eBPF map_type array is available
eBPF map_type prog_array is available
eBPF map_type perf_event_array is available
eBPF map_type percpu_hash is available
eBPF map_type percpu_array is available
eBPF map_type stack_trace is available
...
</code></pre><p>在下面的表格中，我给你整理了几种最常用的映射类型及其功能和使用场景：<br>
<img src="https://static001.geekbang.org/resource/image/f3/d7/f3210199e6689e7659057a935e7fc5d7.jpg?wh=1920x1080" alt="图片"><br>
如果你的 eBPF 程序使用了 BCC 库，你还可以使用预定义的宏来简化 BPF 映射的创建过程。比如，对哈希表映射来说，BCC 定义了  <code>BPF_HASH(name, key_type=u64, leaf_type=u64, size=10240)</code>，因此，你就可以通过下面的几种方法来创建一个哈希表映射：</p><pre><code class="language-c++">// 使用默认参数 key_type=u64, leaf_type=u64, size=10240
BPF_HASH(stats);

// 使用自定义key类型，保持默认 leaf_type=u64, size=10240
struct key_t {
&nbsp; char c[80];
};
BPF_HASH(counts, struct key_t);

// 自定义所有参数
BPF_HASH(cpu_time, uint64_t, uint64_t, 4096);
</code></pre><p>除了创建之外，映射的删除也需要你特别注意。BPF 系统调用中并没有删除映射的命令，这是因为 <strong>BPF 映射会在用户态程序关闭文件描述符的时候自动删除</strong>（即<code>close(fd)</code> ）。 如果你想在程序退出后还保留映射，就需要调用  <code>BPF_OBJ_PIN</code> 命令，将映射挂载到 /sys/fs/bpf 中。</p><p>在调试 BPF 映射相关的问题时，你还可以通过 bpftool 来查看或操作映射的具体内容。比如，你可以通过下面这些命令创建、更新、输出以及删除映射：</p><pre><code class="language-c++">//创建一个哈希表映射，并挂载到/sys/fs/bpf/stats_map(Key和Value的大小都是2字节)
bpftool map create /sys/fs/bpf/stats_map type hash key 2 value 2 entries 8 name stats_map

//查询系统中的所有映射
bpftool map
//示例输出
//340: hash&nbsp; name stats_map&nbsp; flags 0x0
//&nbsp; &nbsp; &nbsp; &nbsp; key 2B&nbsp; value 2B&nbsp; max_entries 8&nbsp; memlock 4096B

//向哈希表映射中插入数据
bpftool map update name stats_map key 0xc1 0xc2 value 0xa1 0xa2

//查询哈希表映射中的所有数据
 
bpftool map dump name stats_map
//示例输出
//key: c1 c2&nbsp; value: a1 a2
//Found 1 element

//删除哈希表映射
rm /sys/fs/bpf/stats_map
</code></pre><h2><strong>BPF 类型格式 (BTF)</strong></h2><p>了解过 BPF 辅助函数和映射之后，我们再来看一个开发 eBPF 程序时最常碰到的问题：内核数据结构的定义。</p><p>在安装 BCC 工具的时候，你可能就注意到了，内核头文件 <code>linux-headers-$(uname -r)</code> 也是必须要安装的一个依赖项。这是因为 BCC 在编译 eBPF 程序时，需要从内核头文件中找到相应的内核数据结构定义。这样，你在调用 <code>bpf_probe_read</code> 时，才能从内存地址中提取到正确的数据类型。</p><p>但是，编译时依赖内核头文件也会带来很多问题。主要有这三个方面：</p><ul>
<li>首先，在开发 eBPF 程序时，为了获得内核数据结构的定义，就需要引入一大堆的内核头文件；</li>
<li>其次，内核头文件的路径和数据结构定义在不同内核版本中很可能不同。因此，你在升级内核版本时，就会遇到找不到头文件和数据结构定义错误的问题；</li>
<li>最后，在很多生产环境的机器中，出于安全考虑，并不允许安装内核头文件，这时就无法得到内核数据结构的定义。<strong>在程序中重定义数据结构</strong>虽然可以暂时解决这个问题，但也很容易把使用着错误数据结构的 eBPF 程序带入新版本内核中运行。</li>
</ul><p>那么，这么多的问题该怎么解决呢？不用担心，BPF 类型格式（BPF Type Format, BTF）的诞生正是为了解决这些问题。从内核 5.2 开始，只要开启了 <code>CONFIG_DEBUG_INFO_BTF</code>，在编译内核时，内核数据结构的定义就会自动内嵌在内核二进制文件 vmlinux 中。并且，你还可以借助下面的命令，把这些数据结构的定义导出到一个头文件中（通常命名为 <code>vmlinux.h</code>）:</p><pre><code class="language-plain">bpftool btf dump file /sys/kernel/btf/vmlinux format c &gt; vmlinux.h
</code></pre><p>如下图（图片来自<a href="https://www.grant.pizza/blog/vmlinux-header">GRANT SELTZER博客</a>）所示，有了内核数据结构的定义，你在开发 eBPF 程序时只需要引入一个 <code>vmlinux.h</code> 即可，不用再引入一大堆的内核头文件了。</p><p><img src="https://static001.geekbang.org/resource/image/45/20/45bbf696e8620d322d857ceab3871720.jpg?wh=1920x1204" alt="" title="vmlinux.h的使用示意图"></p><p>同时，借助 BTF、bpftool 等工具，我们也可以更好地了解 BPF 程序的内部信息，这也会让调试变得更加方便。比如，在查看 BPF 映射的内容时，你可以直接看到结构化的数据，而不只是十六进制数值：</p><pre><code class="language-c++"># bpftool map dump id 386
[
&nbsp; {
&nbsp; &nbsp; &nbsp; "key": 0,
&nbsp; &nbsp; &nbsp; "value": {
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; "eth0": {
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; "value": 0,
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; "ifindex": 0,
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; "mac": []
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; }
&nbsp; &nbsp; &nbsp; }
&nbsp; }
]
</code></pre><p>解决了内核数据结构的定义问题，接下来的问题就是，<strong>如何让 eBPF 程序在内核升级之后，不需要重新编译就可以直接运行</strong>。eBPF 的一次编译到处执行（Compile Once Run Everywhere，简称 CO-RE）项目借助了 BTF 提供的调试信息，再通过下面的两个步骤，使得 eBPF 程序可以适配不同版本的内核：</p><ul>
<li>第一，通过对 BPF 代码中的访问偏移量进行重写，解决了不同内核版本中数据结构偏移量不同的问题；</li>
<li>第二，在 libbpf 中预定义不同内核版本中的数据结构的修改，解决了不同内核中数据结构不兼容的问题。</li>
</ul><p>BTF和一次编译到处执行带来了很多的好处，但你也需要注意这一点：它们都要求比较新的内核版本（&gt;=5.2），并且需要非常新的发行版（如 Ubuntu 20.10+、RHEL 8.2+ 等）才会默认打开内核配置 <code>CONFIG_DEBUG_INFO_BTF</code>。对于旧版本的内核，虽然它们不会再去内置 BTF 的支持，但开源社区正在尝试通过 <a href="https://github.com/aquasecurity/btfhub">BTFHub</a> 等方法，为它们提供 BTF 调试信息。</p><h2>小结</h2><p>今天，我带你一起梳理了 eBPF 程序跟内核交互的基本方法。</p><p>一个完整的 eBPF 程序，通常包含用户态和内核态两部分：用户态程序需要通过 BPF 系统调用跟内核进行交互，进而完成 eBPF 程序加载、事件挂载以及映射创建和更新等任务；而在内核态中，eBPF 程序也不能任意调用内核函数，而是需要通过 BPF 辅助函数完成所需的任务。尤其是在访问内存地址的时候，必须要借助  <code>bpf_probe_read</code> 系列函数读取内存数据，以确保内存的安全和高效访问。</p><p>在 eBPF 程序需要大块存储时，我们还需要根据应用场景，引入特定类型的 BPF 映射，并借助它向用户空间的程序提供运行状态的数据。</p><p>这一讲的最后，我还带你一起了解了 BTF 和 CO-RE 项目，它们在提供轻量级调试信息的同时，还解决了跨内核版本的兼容性问题。很多开源社区的 eBPF 项目（如 BCC 等）也都在向 BTF 进行迁移。</p><h2>思考题</h2><p>最后，我想邀请你来聊一聊：</p><ol>
<li>你是如何理解 BPF 系统调用和 BPF 辅助函数的？</li>
<li>除了今天讲到的内容，bpftool 还提供了哪些有趣的功能呢？给你一个小提示：可以使用 man bpftool 查询它的使用文档。</li>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/5e/96/a03175bc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>莫名</span>
  </div>
  <div class="_2_QraFYR_0">倪老师留的思考题比较基础，尝试从另一个角度做下觉得有趣的对比：<br><br>1、bpf 系统调用一定程度上参考了 perf_event_open 设计，bpf_attr、perf_event_attr 均包含了大量 union，用于适配不同的 cmd 或者性能监控事件类型。因此，bpf、perf_event_open 接口看似简单，其实是大杂烩。<br><br>bpf(int cmd, union bpf_attr *attr, ...) <br>perf_event_open(struct perf_event_attr *attr, ...)<br><br>2、bpftool 命令设计风格重度参考了 iproute2 软件包中的 ip 命令，用法基本上一致。<br><br>Usage: bpftool [OPTIONS] OBJECT { COMMAND | help }<br>Usage: ip [ OPTIONS ] OBJECT { COMMAND | help }<br><br>无论内核开发还是工具开发，都可以看到设计思路借鉴的影子。工作之余不妨多些学习与思考，也许就可以把大师们比较好的设计思路随手用于手头的任务之中。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 非常棒的思考思路👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-26 15:11:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_e8988c</span>
  </div>
  <div class="_2_QraFYR_0">飞哥，你好，看完前两章感觉比较迷糊。没能做到学以致用，或者说举一反三。<br>例如第一个ebpf程序，通过trace open系统调用来监控应用的open动作。<br>那我实际中，有没有什么快速的方案&#47;套路来完成一些其他的trace，例如我想知道监控那些程序使用了socket&#47;bind(尤其适用于有些短暂进程使用udp发送了一些报文就立马退出了）。<br>当我想新增一个trace事件，我的.c需要去包含那些头文件，我的.py需要跟踪那个系统调用。<br>这些头文件与系统调用在哪可以找到，如果去找？<br><br>希望飞哥大佬能给个大体的框架，思路，谢谢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 非常欣慰你在看到一个入门示例就能思考这么多的问题。基础入门篇的内容主要侧重在 eBPF 本身，而其应用的方法都包含在实战进阶篇（其实看看目录就知道这些问题都包含在实战进阶篇中）。<br><br>不过既然问到了，我就提前先给你介绍一个方便的工具 https:&#47;&#47;github.com&#47;iovisor&#47;bpftrace。想你问到的查询跟踪点函数、查询系统调用格式、快速跟踪一个系统调用、网络socket的跟踪等等它都是支持的，并且用起来就像SHELL脚本一样方便。你可以先学习一下，我们后面的案例中还会讲到它的详细使用方法。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-26 16:18:07</div>
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
  <div class="_2_QraFYR_0">想请教下老师关于bpf内核态程序的成名周期，比如前几节课的例子，如果我通过ctrl-c终端用户态程序的时候在bpf虚拟机会发生什么呢？我理解应该会结束已经加载的bpf指令，清理map之类的资源等等，具体会有哪些，这些操作是如何在我们程序中自动插入和实现，请老师指点下。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 非常好的问题。ctrl-c其实终止的是用户空间的程序，终止后所有的文件描述符都会释放掉，对应的map和ebpf程序引用都会减到0，进而它们也就被自动回收了。这些都是自动的，不需要在程序里面增加额外的处理逻辑。<br><br>当然，在需要长久保持eBPF程序的情境中（比如XDP和TC程序），eBPF程序和map的生命周期是可以在程序内控制的，比如通过 tc、ip 以及 bpftool 等工具（当然它们也都有相应的系统调用）。我们课程后面还会讲到相关的案例。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-29 21:39:26</div>
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
  <div class="_2_QraFYR_0">BPF 系统调用  --&gt; 发生在 用户态<br>BPF 辅助函数  --&gt; 内核态<br>---<br>root@ubuntu-impish:&#47;proc# bpftool perf<br>pid 38110  fd 6: prog_id 572  kprobe  func do_sys_openat2  offset 0<br><br>root@ubuntu-impish:&#47;proc# ps -ef|grep 38110<br>root       38110   38109  0 03:41 pts&#47;2    00:00:02 python3 .&#47;trace_open.py</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不错的总结👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-28 11:53:06</div>
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
  <div class="_2_QraFYR_0">BTF那块是不是没说完阿？不是只介绍了可以使用bpftool生成vmlinux.h头文件嘛？ 怎么后面CO-RE就可以借助BTF的调试信息？这个调试信息哪里来？一头雾水。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-15 23:06:17</div>
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
  <div class="_2_QraFYR_0">老师， bpf 的 map 怎么移除啊？ 没看到 bpftools 的命令</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: map是自动删除的，跟它关联的文件描述符关闭后就会删除</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-15 17:34:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJlaNica7xRH6LlMNJtrbK0toc1od8YdqLZOD2AbnOZ2QyKC13gvrrL9cOx5dyYNcsHnJkR6K4ibxZQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_59a6f9</span>
  </div>
  <div class="_2_QraFYR_0">在高版本内核编译运行的ebpf程序，移植到低版本也能直接运行吗？低版本的libbpf 如何感知到高版本内核的修改，从而预定义不同内核版本的数据结构吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 从低版本到高版本的兼容性可以用CO-RE解决（CO-RE只在新版本内核才支持），而反过来就需要在eBPF内部加上额外的代码来处理了。比如根据内核版本作条件编译，低版本内核执行不同的逻辑。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-04 14:10:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/33/ae/5e05b9dc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>朝东</span>
  </div>
  <div class="_2_QraFYR_0">你好，偶尔遇到个问题，在bpf加载成功后，没看到prink 到pipe 调试打印，重启系统就恢复，不知什么原因？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-12 18:03:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/23/3b/1d/cbfbd93e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小印_zoe</span>
  </div>
  <div class="_2_QraFYR_0">有没有详细的付费学习群？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-10 10:23:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/25/6f/a7/565214bc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>│．Sk</span>
  </div>
  <div class="_2_QraFYR_0">倪老师好，<br><br>想请教一下，在支持 BTF 的内核中，利用 CO-RE 编译出的 ebpf 程序可执行文件，直接 copy 到低版本不支持 BTF 的内核版本的系统上能正常运行吗？ 谢谢！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不行的，它需要运行的内核也支持BTF。<br><br>不过现在最新的bpftool已经支持导出BTF，跟eBPF程序一通分发后也可以执行了。具体可以参考这个博客 https:&#47;&#47;www.inspektor-gadget.io&#47;&#47;blog&#47;2022&#47;03&#47;btfgen-one-step-closer-to-truly-portable-ebpf-programs&#47;</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-31 23:57:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/25/3c/92/81fa306d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张Dave</span>
  </div>
  <div class="_2_QraFYR_0">老师，请问下，一次编译到处执行，哪里还有更详细的解释呢？<br>没太明白文中总结的两点具体是怎么做的？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-24 08:53:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJOBwR7MCVqwZbPA5RQ2mjUjd571jUXUcBCE7lY5vSMibWn8D5S4PzDZMaAhRPdnRBqYbVOBTJibhJg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ヾ(◍°∇°◍)ﾉﾞ</span>
  </div>
  <div class="_2_QraFYR_0">老师，请教个问题，就是在比如在jdbc连接的时候，有时候一些防火墙或者VPN什么瞬间的原因会导致这个连接要卡住很长时间（通常是2小时15分），有什么办法可以快速探测并结束这种TCP反馈给客户端，ebpf这个手段可以嘛？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-07 12:00:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/b5/02/3bf0509e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>reonard</span>
  </div>
  <div class="_2_QraFYR_0">倪老师好，我试了4.18的内核，也可以用bpftool btf dump file &#47;sys&#47;kernel&#47;btf&#47;vmlinux format c 获得vmlinux.h，是不是就不需要5.2的内核了？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-25 00:47:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>woJA1wCgAAbjKldokPvO1h9ZEJTUP8ug</span>
  </div>
  <div class="_2_QraFYR_0">老师，结构化的数据需要怎么看，bpftool map dump id mapid和bpftool map dump name mapname结果都是一样的16进制数</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-21 20:29:43</div>
  </div>
</div>
</div>
</li>
</ul>