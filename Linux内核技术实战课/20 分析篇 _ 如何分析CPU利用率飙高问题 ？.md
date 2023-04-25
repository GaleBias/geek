<audio title="20 分析篇 _ 如何分析CPU利用率飙高问题 ？" src="https://static001.geekbang.org/resource/audio/19/60/1966ed0049810c56aa6cd161453ae260.mp3" controls="controls"></audio> 
<p>你好，我是邵亚方。</p><p>如果你是一名应用开发者，那你应该知道如何去分析应用逻辑，对于如何优化应用代码提升系统性能也应该有自己的一套经验。而我们这节课想要讨论的是，如何拓展你的边界，让你能够分析代码之外的模块，以及对你而言几乎是黑盒的Linux内核。</p><p>在很多情况下，应用的性能问题都需要通过分析内核行为来解决，因此，内核提供了非常多的指标供应用程序参考。当应用出现问题时，我们可以查看到底是哪些指标出现了异常，然后再做进一步分析。不过，这些内核导出的指标并不能覆盖所有的场景，我们面临的问题可能更加棘手：应用出现性能问题，可是系统中所有的指标都看起来没有异常。相信很多人都为此抓狂过。那出现这种情况时，内核到底有没有问题呢，它究竟在搞什么鬼？这节课我就带你探讨一下如何分析这类问题。</p><p>我们知道，对于应用开发者而言，应用程序的边界是系统调用，进入到系统调用中就是Linux内核了。所以，要想拓展分析问题的边界，你首先需要知道该怎么去分析应用程序使用的系统调用函数。对于内核开发者而言，边界同样是系统调用，系统调用之外是应用程序。如果内核开发者想要拓展分析问题的边界，也需要知道如何利用系统调用去追踪应用程序的逻辑。</p><!-- [[[read_end]]] --><h2>如何拓展你分析问题的边界？</h2><p>作为一名内核开发者，我对应用程序逻辑的了解没有对内核的了解那么深。不过，当应用开发者向我寻求帮助时，尽管我对他们的应用逻辑一无所知，但这并不影响我对问题的分析，因为我知道如何借助分析工具追踪应用程序的逻辑。经过一系列追踪之后，我就能对应用程序有一个大概的认识。</p><p>我常用来追踪应用逻辑的工具之一就是strace。strace可以用来分析应用和内核的“边界”——系统调用。借助strace，我们不仅能够了解应用执行的逻辑，还可以了解内核逻辑。那么，作为应用开发者的你，就可以借助这个工具来拓展你分析应用问题的边界。</p><p>strace可以跟踪进程的系统调用、特定的系统调用以及系统调用的执行时间。很多时候，我们通过系统调用的执行时间，就能判断出业务延迟发生在哪里。比如我们想要跟踪一个多线程程序的系统调用情况，那就可以这样使用strace：</p><blockquote>
<p>$ strace -T -tt -ff -p pid -o strace.out</p>
</blockquote><p>不过，在使用strace跟踪进程之前，我希望你可以先明白strace的工作原理，这也是我们这节课的目的：你不只要知道怎样使用工具，更要明白工具的原理，这样在出现问题时，你就能明白该工具是否适用了。</p><h2>了解工具的原理，不要局限于如何使用它</h2><p>strace工具的原理如下图所示（我们以上面的那个命令为例来说明）：</p><p><img src="https://static001.geekbang.org/resource/image/bc/6f/bc04236262f16e0b69842dafd503616f.jpg?wh=4871*3293" alt="" title="strace基本原理"></p><p>我们从图中可以看到，对于正在运行的进程而言，strace可以attach到目标进程上，这是通过ptrace这个系统调用实现的（gdb工具也是如此）。ptrace的PTRACE_SYSCALL会去追踪目标进程的系统调用；目标进程被追踪后，每次进入syscall，都会产生SIGTRAP信号并暂停执行；追踪者通过目标进程触发的SIGTRAP信号，就可以知道目标进程进入了系统调用，然后追踪者会去处理该系统调用，我们用strace命令观察到的信息输出就是该处理的结果；追踪者处理完该系统调用后，就会恢复目标进程的执行。被恢复的目标进程会一直执行下去，直到下一个系统调用。</p><p>你可以发现，目标进程每执行一次系统调用都会被打断，等strace处理完后，目标进程才能继续执行，这就会给目标进程带来比较明显的延迟。因此，在生产环境中我不建议使用该命令，如果你要使用该命令来追踪生产环境的问题，那就一定要做好预案。</p><p>假设我们使用strace跟踪到，线程延迟抖动是由某一个系统调用耗时长导致的，那么接下来我们该怎么继续追踪呢？这就到了应用开发者和运维人员需要拓展分析边界的时刻了，对内核开发者来说，这才算是分析问题的开始。</p><h2>学会使用内核开发者常用的分析工具</h2><p>我们以一个实际案例来说明吧。有一次，业务开发者反馈说他们用strace追踪发现业务的pread(2)系统调用耗时很长，经常会有几十毫秒（ms）的情况，甚至能够达到秒级，但是不清楚接下来该如何分析，因此让我帮他们分析一下。</p><p>因为已经明确了问题是由pread(2)这个系统调用引起的，所以对内核开发者而言，后续的分析就相对容易了。分析这类问题最合适的工具是ftrace，我们可以使用ftrace的function_trace功能来追踪pread(2)这个系统调用到底是在哪里耗费了这么长的时间。</p><p>要想追踪pread(2)究竟在哪里耗时长，我们就需要知道该系统调用对应的内核函数是什么。我们有两种途径可以方便地获取到系统调用对应的内核函数：</p><ul>
<li>查看<a href="https://elixir.bootlin.com/linux/v5.9-rc6/source/include/linux/syscalls.h">include/linux/syscalls.h</a>文件里的内核函数：</li>
</ul><p>你可以看到，与pread有关的函数有多个，由于我们的系统是64bit的，只需关注64bit相关的系统调用就可以了，所以我们锁定在ksys_pread64和sys_read64这两个函数上。<a href="https://elixir.bootlin.com/linux/v5.9-rc6/source/include/linux/syscalls.h#L1234">通过该头文件里的注释</a>我们能知道，前者是内核使用的，后者是导出给用户的。那么在内核里，我们就需要去追踪前者。另外，请注意，不同内核版本对应的函数可能不一致，我们这里是以最新内核代码(5.9-rc)为例来说明的。</p><ul>
<li>通过/proc/kallsyms这个文件来查找：</li>
</ul><blockquote>
<p>$ cat /proc/kallsyms | grep pread64<br>
…<br>
ffffffffa02ef3d0 T ksys_pread64<br>
…</p>
</blockquote><p>/proc/kallsyms里的每一行都是一个符号，其中第一列是符号地址，第二列是符号的属性，第三列是符号名字，比如上面这个信息中的T就表示全局代码符号，我们可以追踪这类的符号。关于这些符号属性的含义，你可以通过<a href="https://man7.org/linux/man-pages/man1/nm.1p.html">man nm</a>来查看。</p><p>接下来我们就使用ftrace的function_graph功能来追踪ksys_pread64这个函数，看看究竟是内核的哪里耗时这么久。function_graph的使用方式如下：</p><pre><code># 首先设置要追踪的函数
$ echo ksys_pread64 &gt; /sys/kernel/debug/tracing/set_graph_function

# 其次设置要追踪的线程的pid，如果有多个线程，那需要将每个线程都逐个写入
$ echo 6577 &gt; /sys/kernel/debug/tracing/set_ftrace_pid
$ echo 6589 &gt;&gt; /sys/kernel/debug/tracing/set_ftrace_pid

# 将function_graph设置为当前的tracer，来追踪函数调用情况
$ echo function_graph &gt; /sys/kernel/debug/tracing/current_trace
</code></pre><p>然后我们就可以通过/sys/kernel/debug/tracing/trace_pipe来查看它的输出了，下面就是我追踪到的耗时情况：</p><p><img src="https://static001.geekbang.org/resource/image/68/fc/689eacfa3ef10c236221f1b2051ab5fc.png?wh=1352*1238" alt=""></p><p>我们可以发现pread(2)有102ms是阻塞在io_schedule()这个函数里的，io_schedule()的意思是，该线程因I/O阻塞而被调度走，线程需要等待I/O完成才能继续执行。在function_graph里，我们同样也能看到<strong>pread</strong>**(<strong><strong>2</strong></strong>)**是如何一步步执行到io_schedule的，由于整个流程比较长，我在这里只把关键的调用逻辑贴出来：</p><pre><code> 21)               |            __lock_page_killable() {
 21)   0.073 us    |              page_waitqueue();
 21)               |              __wait_on_bit_lock() {
 21)               |                prepare_to_wait_exclusive() {
 21)   0.186 us    |                  _raw_spin_lock_irqsave();
 21)   0.051 us    |                  _raw_spin_unlock_irqrestore();
 21)   1.339 us    |                }
 21)               |                bit_wait_io() {
 21)               |                  io_schedule() {
</code></pre><p>我们可以看到，<strong>pread（2）</strong>是从__lock_page_killable这个函数调用下来的。当pread(2)从磁盘中读文件到内存页（page）时，会先lock该page，读完后再unlock。如果该page已经被别的线程lock了，比如在I/O过程中被lock，那么pread(2)就需要等待。等该page被I/O线程unlock后，pread(2)才能继续把文件内容读到这个page中。我们当时遇到的情况是：在pread(2)从磁盘中读取文件内容到一个page中的时候，该page已经被lock了，于是调用pread(2)的线程就在这里等待。这其实是合理的内核逻辑，没有什么问题。接下来，我们就需要看看为什么该page会被lock了这么久。</p><p>因为线程是阻塞在磁盘I/O里的，所以我们需要查看一下系统的磁盘I/O情况，我们可以使用iostat来观察：</p><blockquote>
<p>$ iostat -dxm 1</p>
</blockquote><p>追踪信息如下：</p><p><img src="https://static001.geekbang.org/resource/image/ca/04/ca94121ff716f75c171e2a3380d14d04.png?wh=1920*837" alt=""></p><p>其中，sdb是业务pread(2)读取的磁盘所在的文件，通常情况下它的读写量很小，但是我们从上图中可以看到，磁盘利用率（%util）会随机出现比较高的情况，接近100%。而且avgrq-sz很大，也就是说出现了很多I/O排队的情况。另外，w/s比平时也要高很多。我们还可以看到，由于此时存在大量的I/O写操作，磁盘I/O排队严重，磁盘I/O利用率也很高。根据这些信息我们可以判断，之所以pread(2)读磁盘文件耗时较长，很可能是因为被写操作饿死导致的。因此，我们接下来需要排查到底是谁在进行写I/O操作。</p><p>通过iotop观察I/O行为，我们发现并没有用户线程在进行I/O写操作，写操作几乎都是内核线程kworker来执行的，也就是说用户线程把内容写在了Page Cache里，然后kwoker将这些Page Cache中的内容再同步到磁盘中。这就涉及到了我们这门课程第一个模块的内容了：如何观测Page Cache的行为。</p><h2>自己写分析工具</h2><p>如果你现在还不清楚该如何来观测Page Cache的行为，那我建议你再从头仔细看一遍我们这门课程的第一个模块，我在这里就不细说了。不过，我要提一下在Page Cache模块中未曾提到的一些方法，这些方法用于判断内存中都有哪些文件以及这些文件的大小。</p><p>常规方式是用fincore和mincore，不过它们都比较低效。这里有一个更加高效的方式：通过写一个内核模块遍历inode来查看Page Cache的组成。该模块的代码较多，我只说一下核心的思想，伪代码大致如下：</p><pre><code>iterate_supers // 遍历super block
  iterate_pagecache_sb // 遍历superblock里的inode
      list_for_each_entry(inode, &amp;sb-&gt;s_inodes, i_sb_list)
        // 记录该inode的pagecache大小 
        nrpages = inode-&gt;i_mapping-&gt;nrpages; 
        /* 获取该inode对应的dentry，然后根据该dentry来查找文件路径；
         * 请注意inode可能没有对应的dentry，因为dentry可能被回收掉了，
         * 此时就无法查看该inode对应的文件名了。
         */
        dentry = dentry_from_inode(inode); 
        dentry_path_raw(dentry, filename, PATH_MAX);
</code></pre><p>使用这种方式不仅可以查看进程正在打开的文件，也能查看文件已经被进程关闭，但文件内容还在内存中的情况。所以这种方式分析起来会更全面。</p><p>通过查看Page Cache的文件内容，我们发现某些特定的文件占用的内存特别大，但是这些文件都是一些离线业务的文件，也就是不重要业务的文件。因为离线业务占用了大量的Page Cache，导致该在线业务的workingset大大减小，所以pread(2)在读文件内容时经常命中不了Page Cache，进而需要从磁盘来读文件，也就是说该在线业务存在大量的pagein和pageout。</p><p>至此，问题的解决方案也就有了：我们可以通过限制离线业务的Page Cache大小，来保障在线业务的workingset，防止它出现较多的refault。经过这样调整后，业务再也没有出现这种性能抖动了。</p><p>你是不是对我上面提到的这些名字感到困惑呢？也不清楚inode和Page Cache是什么关系？如果是的话，那就说明你没有好好学习我们这门课程的Page Cache模块，我建议你从头再仔细学习一遍。</p><p>好了，我们这节课就讲到这里。</p><h2>课堂总结</h2><p>我们这节课的内容，对于应用开发者和运维人员而言是有些难度的。我之所以讲这些有难度的内容，就是希望你可以拓展分析问题的边界。这节课的内容对内核开发者而言基本都是基础知识，如果你看不太明白，说明你对内核的理解还不够，你需要花更多的时间好好学习它。我研究内核已经有很多年了，尽管如此，我还是觉得自己对它的理解仍然不够深刻，需要持续不断地学习和研究，而我也一直在这么做。</p><p>我们现在回顾一下这节课的重点：</p><ul>
<li>strace工具是应用和内核的边界，如果你是一名应用开发者，并且想去拓展分析问题的边界，那你就需要去了解strace的原理，还需要了解如何去分析strace发现的问题；</li>
<li>ftrace是分析内核问题的利器，你需要去了解它；</li>
<li>你需要根据自己的问题来实现特定的问题分析工具，要想更好地实现这些分析工具，你必须掌握很多内核细节。</li>
</ul><h2>课后作业</h2><p>关于我们这节课的“自己写分析工具”这部分，我给你留一个作业，这也是我没有精力和时间去做的一件事：请你在sysrq里实现一个功能，让它可以显示出系统中所有R和D状态的任务，以此来帮助开发者分析系统load飙高的问题。</p><p>我在我们的内核里已经实现了该功能，不过在推给Linux内核时，maintainer希望我可以用另一种方式来实现。由于那个时候我在忙其他事情，这件事便被搁置了下来。如果你实现得比较好，你可以把它提交给Linux内核，提交的时候你也可以cc一下我（laoar.shao@gmail.com）。对了，你在实现时，也可以参考我之前的提交记录：<a href="https://lore.kernel.org/patchwork/patch/818962/">scheduler: enhancement to show_state_filter and SysRq</a>。欢迎你在留言区与我讨论。</p><p>最后，感谢你的阅读，如果你认为这节课的内容有收获，也欢迎把它分享给你的朋友。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/9b/ba/333b59e5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Linuxer</span>
  </div>
  <div class="_2_QraFYR_0">邵老师，你好，有个负载高的问题，我判断是因为开的线程数过多导致的，我想请教一下是下面的这些数据是不是有说服力或者有什么手段进一步确认? 谢谢！<br>top<br>top - 18:46:33 up 186 days,  4:31,  3 users,  load average: 67.47, 55.78, 61.19<br>Tasks: 377 total,   1 running, 376 sleeping,   0 stopped,   0 zombie<br>%Cpu(s): 51.0 us, 30.2 sy,  0.0 ni,  8.0 id,  0.0 wa,  0.0 hi, 10.8 si,  0.0 st<br>KiB Mem : 13173332+total, 21147572 free,  6840020 used, 10374573+buff&#47;cache<br><br>grep procs_running &#47;proc&#47;stat<br>procs_running 70<br>grep procs_running &#47;proc&#47;stat<br>procs_running 90<br>grep procs_running &#47;proc&#47;stat<br>procs_running 119<br><br> perf top -U<br>   4.41%  [kernel]          [k] system_call_after_swapgs<br>   3.49%  [kernel]          [k] do_select<br>   3.23%  [kernel]          [k] copy_user_enhanced_fast_string<br>   2.98%  [kernel]          [k] sysret_check<br>   2.78%  [kernel]          [k] __schedule<br>   1.82%  [kernel]          [k] __check_object_size<br>   1.67%  [kernel]          [k] fget_light<br>   1.22%  [kernel]          [k] tcp_ack<br>   1.21%  [kernel]          [k] __audit_syscall_exit<br>   1.16%  [kernel]          [k] __x86_indirect_thunk_rax<br>   1.15%  [kernel]          [k] tcp_poll<br>   1.13%  [kernel]          [k] _raw_spin_lock_irqsave<br>   1.12%  [kernel]          [k] __switch_to<br> <br><br><br> perf stat<br> Performance counter stats for &#39;system wide&#39;:<br><br>     376801.028719      cpu-clock (msec)          #   31.997 CPUs utilized          <br>         7,323,807      context-switches          #    0.019 M&#47;sec                  <br>           824,699      cpu-migrations            #    0.002 M&#47;sec                  <br>           100,337      page-faults               #    0.266 K&#47;sec                  <br>   808,730,622,944      cycles                    #    2.146 GHz                    <br>   429,965,110,114      instructions              #    0.53  insn per cycle         <br>    90,908,416,046      branches                  #  241.264 M&#47;sec                  <br>     2,554,107,830      branch-misses             #    2.81% of all branches        <br><br>      11.776125779 seconds time elapsed<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 从你提供的top信息来看，此时的cpu利用率已经很高了：<br>%Cpu(s): 51.0 us, 30.2 sy,  0.0 ni,  8.0 id,  0.0 wa,  0.0 hi, 10.8 si,  0.0 st<br><br>51.0 us, 30.2 sy， 10.8 si 加起来有90%，这么高的cpu利用率和running的线程数太多有关系，需要将cpu利用率降下来。另外你可以看到，有30%的sys和10%的si，看起来是网卡软中断太多导致的，可能你的系统里存在非常多的网络连接，如果是这样的话，需要去评估下是否有必要控制连接数。<br><br>ss -s可以查看系统里连接数统计信息。<br><br>从perf热点：<br>  3.49%  [kernel]          [k] do_select<br><br>select(2)系统调用是个热点，也跟网络连接较多或者网络请求较多有关系。<br><br><br><br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-06 20:06:39</div>
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
  <div class="_2_QraFYR_0">推荐使用 ftrace 前端 trace-cmd，直接操作 tracefs 文件略显繁琐。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: trace-cmd使用起来更简单些，但是它不灵活，很多时候需要根据实际情况来观测一些细节，这个时候就需要tracefs+手写Python进行分析了。<br>我个人不太喜欢使用tracep-cmd，我更喜欢用Python来写问题分析工具。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-21 22:51:57</div>
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
  <div class="_2_QraFYR_0">iotop 没有发现写 I&#47;O 的用户进程，觉得可以往上走，在文件系统层面做些分析，page 紧张的情况下，文件读写操作会变得相对慢一些。借助 BCC 现有工具 ext4slower、ext4dist（假设是 ext4 fs）分析，应该可以看到具体的用户进程。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-17 19:04:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/e9/6b/a52282b8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>会飞的鱼</span>
  </div>
  <div class="_2_QraFYR_0">老师好，我的服务器内核空间的cpu使用率达到了100%，重启了几次依然没有解决，然后我用top查看，没有发现cpu占用很高的进程，用execsnoop也没发现异常，然后用pidstat -w查看，发现rcu_sched的上下文切换很频繁，而且是自愿上下文切换，然后我再用perf top -g -p 9追踪，发现有schedule,finish_task_switch等系统调用，但我还是不清楚是什么原因导致的内核空间cpu使用率高，服务器上也没跑什么程序，这种情况怎么进一步解决呢？ 可以加您微信进一步求教下吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: cpu的利用率是100%，能否细化一下具体是什么的利用率高，比如usr，sys，wait，siq等？top就可以看到。明确了哪一项高后，就可以针对性的分析了。<br>你可以把观察到的具体现象发到我邮箱：laoar.shao@gmail.com, 我在开篇词里也有提到我的邮箱。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-25 22:43:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_9bf0b0</span>
  </div>
  <div class="_2_QraFYR_0">邵老师，关于在 sysrq 里实现显示出系统中所有 R 和 D 状态的任务的功能，我的想法是将<br><br>show_state_filter()接口的state_filter参数改成指针类型，传递参数NULL时表示显示所有进程信息，<br>这样避免与TASK_RUNNING冲突，代码修改量也小，也相对优雅。<br><br>-extern void show_state_filter(unsigned long state_filter);<br>+extern void show_state_filter(unsigned long *state_filter);<br><br> static inline void show_state(void)<br> {<br>- show_state_filter(0);<br>+ show_state_filter(NULL);<br> }<br><br> static void sysrq_handle_showstate_blocked(int key)<br> {<br>- show_state_filter(TASK_UNINTERRUPTIBLE);<br>+ unsigned long filter = TASK_UNINTERRUPTIBLE;<br>+<br>+ show_state_filter(&amp;filter);<br> }<br><br><br>然后调用unregister_sysrq_key()移除sysrq_showstate_blocked_op，<br>接着register_sysrq_key()接口注册定制化的op，<br>定制化的回调函数handle可以实现如下：<br>static void sysrq_handle_showstate_load(int key)<br>{<br>    unsigned long filter;<br><br>    filter = TASK_RUNNING;<br>    show_state_filter(&amp;filter);<br><br>    filter = TASK_UNINTERRUPTIBLE;<br>    show_state_filter(&amp;filter);<br>}<br><br>关于注册定制化的sysrq，可以通过early_param或者模块的方式。<br>以模块的方式注册定制化的sysrq时需要导出内核符号show_state_filter。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复:     filter = TASK_RUNNING;<br>    show_state_filter(&amp;filter);<br><br>    filter = TASK_UNINTERRUPTIBLE;<br>    show_state_filter(&amp;filter);<br><br>这里调用两次看起来不太好，因为每次都要检查所有task，效率低。最好只调用一次，比如以(TASK_UNINTERRUPTIBLE | TASK_RUNNING)的方式来调用。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-10 16:06:36</div>
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
  <div class="_2_QraFYR_0"># 老师文中提到的`查看 Page Cache 的组成`这个功能,感觉很吸引人啊!<br><br># 根据老师的提示,也只找到了这两个方式:<br>## [Is it possible to list the files that are cached?](https:&#47;&#47;serverfault.com&#47;a&#47;782640)<br>原理也比较简单:<br>借助`ps`找出`rss`的top10进程,然后根据`lsof`找出进程引用的文件,最后借助`linux-fincore`查看这些文件在PageCache中的信息.<br><br>感觉这个脚本也有局限性.<br>由于`linux-fincore`只能查看列出的文件,所以是无法查看已经不被进程占用的`Page Cache`中的文件及大小的.<br><br>## [pcstat](https:&#47;&#47;github.com&#47;tobert&#47;pcstat)<br>这个工具是借助`mincore`来查看的Page Cache信息,但是也是需要列出具体的文件名.<br><br># 疑问<br>还是老师的这个方法好,不需要提供文件列表,也可以查看内存中都有哪些文件以及这些文件的大小.<br>请问,老师的这个工具有开源版本么?<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个方法需要修改内核来实现，或者写一个内核模块来实现。 主要思路我已经写在文章里了，你可以思考下如何来实现。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-03 18:10:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/13/fb/7f31dfd4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Bin</span>
  </div>
  <div class="_2_QraFYR_0">你好，邵老师， 这篇文文章标题写的 cpu利用率高因为系统调用的问题，而这个系统调用主要卡在io wait上，但io wait 跟cpu利用率没关系吧，cpu利用率(用户态加系统态)应该不受影响才对</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-09 13:55:27</div>
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
  <div class="_2_QraFYR_0">我是一名系统运维工程师，对内核还不太了解。但感觉这节课还是收货了很多，尤其是如何借助ftrace去做进一步分析，以及如何查看page cache中都是哪些文件及文件大小。之前只知道使用cachestat和cachetop去查看缓存命中情况。希望以后工作中能够运用上。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-24 11:02:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/e3/cc/0947ff0b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>nestle</span>
  </div>
  <div class="_2_QraFYR_0">老是，IO排队是不是应该看avgqu-sz这个字段？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对的 它表示队列长度</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-14 22:01:46</div>
  </div>
</div>
</div>
</li>
</ul>