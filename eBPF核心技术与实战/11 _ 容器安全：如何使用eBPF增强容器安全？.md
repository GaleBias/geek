<audio title="11 _ 容器安全：如何使用eBPF增强容器安全？" src="https://static001.geekbang.org/resource/audio/f5/ea/f5ebe3154cea5dee99dc51fe1dddc5ea.mp3" controls="controls"></audio> 
<p>你好，我是倪朋飞。</p><p>上一讲，我以最常见的网络丢包为例，带你一起梳理了 eBPF 所提供的网络功能特性，并教你使用 bpftrace 开发了一个跟踪内核网络协议栈的 eBPF 程序。虽然 eBPF 起源于网络过滤，并且网络过滤也是 eBPF 应用最为广泛的一个领域，但其实 eBPF 的应用已经远远超出了这一范畴。故障诊断、网络优化、安全控制、性能监控等，都已是 eBPF 的主战场。</p><p>随着容器和云原生技术的普及，由于容器天生共享内核的特性，容器的安全和隔离就是所有容器平台头上的“紧箍咒”。因此，如何快速定位容器安全问题，如何确保容器的隔离，以及如何预防容器安全漏洞等，是每个容器平台都需要解决的头号问题。</p><p>既然容器是共享内核的，这些安全问题的解决自然就可以从内核的角度进行考虑。除了容器自身所强依赖的命名空间、cgroups、Linux 权限控制 Capabilities 之外，可以动态跟踪和扩展内核的 eBPF 就成为了安全监控和安全控制的主要手段之一。 Sysdig、Aqua Security、Datadog 等业内知名的容器安全解决方案，都基于 eBPF 构建了丰富的安全特性。</p><p>那么，到底如何使用 eBPF 来监控容器的安全问题，又如何使用 eBPF 阻止容器中的恶意行为呢？今天，我就带你一起来看看如何借助 eBPF 来增强容器安全。</p><!-- [[[read_end]]] --><h2>eBPF 都有哪些安全能力？</h2><p>安全这个词通常包含了非常广泛的任务，包括安全问题的分析与诊断、安全事件的检测，以及安全策略的执行等。针对这些任务，eBPF 又提供了哪些安全能力呢？</p><p>首先，对于<strong>安全问题的分析与诊断</strong>，eBPF 无需修改并重新编译内核和应用就可以动态分析内核及应用的行为。这在很多需要保留安全问题现场的情况下非常有用。特别是在紧急安全事件的处理过程中，eBPF 可以实时探测进程或内核中的可疑行为，进而帮你更快地定位安全问题的根源。</p><p>比如，Aqua Security 开源的 <a href="https://aquasecurity.github.io/tracee/dev/">Tracee</a> 项目就利用 eBPF，动态跟踪系统和应用的可疑行为模式，再与不断丰富的特征检测库进行匹配，就可以分析出容器应用中的安全问题。下面是 Tracee 的工作原理示意图（图片来自 <a href="https://aquasecurity.github.io/tracee/dev/architecture/">Tracee 文档</a>）：</p><p><img src="https://static001.geekbang.org/resource/image/a9/6f/a97a6b3e5077c13fbcb346f217c0a96f.png?wh=1340x853" alt="图片" title="Tracee 原理示意图"></p><p>其次，对于<strong>安全事件的监测</strong>，eBPF 的事件触发机制不仅可以用极低的开销，动态监测内核和应用程序中的各类安全事件，更可以避免其他基于采样或统计机制的安全检测系统中经常发生的安全事件漏检问题。</p><p>如下图（图片来自<a href="https://www.brendangregg.com/Slides/BSidesSF2017_BPF_security_monitoring.pdf">brendangregg.com</a>）所示，kprobe、uprobe、tracepoint、USDT 等各类探针已经非常全面地涵盖了从应用到内核中的各类安全问题相关的跟踪点：</p><p><img src="https://static001.geekbang.org/resource/image/5e/6e/5e178ebbc7928a0cd2aff7243f24e96e.png?wh=1089x693" alt="图片" title="eBPF 安全跟踪点"></p><p>比如，Sysdig 贡献给 CNCF 基金会的 <a href="https://falco.org/">Falco</a> 项目，就利用 eBPF 在运行时监测内核和应用中是否发生了诸如特权提升、SHELL 命令执行、系统文件（比如 <code>/etc/passwd</code>）修改、SSH 登录等异常行为，再通过告警系统实时将这些安全事件及时推送给你。</p><p>最后，对于<strong>安全策略的执行</strong>，安全计算（seccomp）、Linux 安全模块（LSM）等 Linux 已有的安全机制，均可以通过 eBPF 来进行安全审计和决策执行，阻止系统和进程中的非法操作，甚至可以通过辅助函数 <code>bpf_send_signal()</code> 直接把有安全隐患的进程杀死。在网络安全方面，eBPF 不仅可以用来实时探测和跟踪网络请求，更可以在探测到网络攻击等危害行为时直接在网络协议栈之前丢弃数据包，从而实现极高的网络性能。</p><p>如下图所示（图片来自 KubeArmor <a href="https://docs.kubearmor.com/kubearmor/kubearmor-design">文档</a>），容器运行时安全系统 <a href="https://kubearmor.com/">KubeArmor</a> 就利用 LSM 和 eBPF，限制容器中进程执行、文件访问、网络连接等各种违反安全策略的操作。</p><p><img src="https://static001.geekbang.org/resource/image/49/6d/49bd8dbce937eff08bcc40c12559df6d.png?wh=932x518" alt="图片" title="KubeArmor 工作原理"></p><h2>如何使用 eBPF 分析容器的安全问题？</h2><p>既然容器还是共享内核的，运行在内核中的 eBPF 程序自然也能够跟踪和分析容器中的应用程序。但由于容器利用 Linux 的 namespace 机制进行了隔离，其跟踪和分析方法又跟直接运行在主机内的进程有些不同。</p><p>以跟踪恶意程序的执行为例，为了躲避安全监控，很多恶意程序并不是在容器一开始启动的时候就运行了恶意进程，而是先启动一个正常程序，之后再创建新的恶意进程。这种场景特别容易出现在容器安全漏洞被恶意软件侵入的场景。</p><p>那么，如何用 eBPF 来分析这类安全问题呢？我想，你可能已经想起来了，我们课程的 <a href="https://time.geekbang.org/column/article/484207">07 讲</a> 其实已经实现了对新进程创建的跟踪程序，即跟踪系统调用 <code>execve</code>。比如，执行下面的 bpftrace 命令，就可以跟踪新创建的进程：</p><pre><code class="language-bash">sudo bpftrace -e 'tracepoint:syscalls:sys_enter_execve { printf("%-6d %-8s", pid, comm); join(args-&gt;argv);}'
</code></pre><p>打开一个新终端，执行一条 <code>ls</code> 命令，然后你就会看到如下的输出：</p><pre><code class="language-plain">8964   bash    ls --color=auto
</code></pre><p>接下来，参考 <a href="https://docs.docker.com/engine/install/">Docker 官方文档</a>安装 Docker 之后，再执行下面的命令，启动一个 Ubuntu 容器：</p><pre><code class="language-bash"># -it表示进入容器终端，--rm表示终端关闭后自动清理容器
docker run -it --rm --name bash --hostname bash ubuntu:impish
</code></pre><p>在容器中执行 <code>ls</code> 命令，忽略容器启动过程中的进程跟踪信息（Docker在启动容器过程中也会执行大量的命令），你会看到跟刚才类似的输出：</p><pre><code class="language-plain">9018   bash    ls --color=auto
</code></pre><p>这个输出跟刚才在主机中执行 <code>ls</code> 后的结果是一样的，只根据这个输出，我们显然没法区分 <code>ls</code> 是不是运行在容器中。</p><p>实际上，虽然所有容器都是共享内核的，但不同的容器之间还是通过命名空间进行了隔离。你可以使用 <code>lsns</code> 命令来查询容器或者主机的命名空间。比如，在刚才的容器终端中执行 <code>lsns</code> 命令，就可以看到如下的输出：</p><pre><code class="language-plain">        NS TYPE   NPROCS PID USER COMMAND
4026531834 time        2   1 root bash
4026531835 cgroup      2   1 root bash
4026531837 user        2   1 root bash
4026532530 mnt         2   1 root bash
4026532531 uts         2   1 root bash
4026532532 ipc         2   1 root bash
4026532533 pid         2   1 root bash
4026532535 net         2   1 root bash
</code></pre><p>关于这些命名空间的含义，如果你还不了解，可以参考<a href="https://man7.org/linux/man-pages/man7/namespaces.7.html">这里</a>的 Linux 手册。</p><p>在内核中，进程的基本信息都保存在 <a href="https://elixir.bootlin.com/linux/v5.13/source/include/linux/sched.h#L657">task_struct</a> 结构体中，其中也包括了包含命名空间信息的  <a href="https://elixir.bootlin.com/linux/v5.13/source/include/linux/nsproxy.h#L31">nsproxy</a> 结构体。<code>nsproxy</code> 结构体的定义如下所示：</p><pre><code class="language-c++">struct nsproxy {
	atomic_t count;
	struct uts_namespace *uts_ns;
	struct ipc_namespace *ipc_ns;
	struct mnt_namespace *mnt_ns;
	struct pid_namespace *pid_ns_for_children;
	struct net 	     *net_ns;
	struct time_namespace *time_ns;
	struct time_namespace *time_ns_for_children;
	struct cgroup_namespace *cgroup_ns;
};
</code></pre><p>为了区分一个进程是属于容器还是主机，我们可以在跟踪结果中输出 PID 命名空间和 UTS 命名空间中的主机名。</p><p>bpftrace 内置了表示进程结构体的 <a href="https://github.com/iovisor/bpftrace/blob/master/docs/reference_guide.md#1-builtins">curtask</a>，因而对前面的 bpftrace 脚本，我们可以进行下面的改进：</p><pre><code class="language-c++">tracepoint:syscalls:sys_enter_execve {
  /* 1. 获取task_struct结构体 */
  $task = (struct task_struct *)curtask;
  /* 2. 获取PID命名空间 */
  $pidns = $task-&gt;nsproxy-&gt;pid_ns_for_children-&gt;ns.inum;
  /* 3. 获取主机名 */
  $cname = $task-&gt;nsproxy-&gt;uts_ns-&gt;name.nodename;
  /* 4. 输出PID命名空间、主机名和进程基本信息 */
  printf("%-12ld %-8s %-6d %-6d %-8s", (uint64)$pidns, $cname, curtask-&gt;parent-&gt;pid, pid, comm); join(args-&gt;argv);
}
</code></pre><p>这段代码中的具体内容含义如下：</p><ul>
<li>第 1 处，把内置变量 <code>curtask</code> 转换为我们想要的 <code>task_struct</code> 结构体；</li>
<li>第 2 处，从进程信息的 nsproxy 中读取 PID 命名空间编号；</li>
<li>第 3 处，从进程信息的 nsproxy 中读取 UTS 命名空间的主机名（也就是在容器中执行 <code>hostname</code> 命令后的输出）；</li>
<li>第 4 处你已经非常熟悉了，就是把刚才获取的信息输出，以便我们观察。</li>
</ul><p>由于这儿用到了很多内核数据结构，在运行之前，还需要给它引入相关数据结构定义的头文件：</p><pre><code class="language-c++">#include &lt;linux/sched.h&gt;
#include &lt;linux/nsproxy.h&gt;
#include &lt;linux/utsname.h&gt;
#include &lt;linux/pid_namespace.h&gt;
</code></pre><p>同时，由于输出的内容比较多，为了便于理解，你还可以在脚本运行开始的时候输出一个表头，表示每个输出的含义：</p><pre><code class="language-plain">BEGIN {
  printf("%-12s %-8s %-6s %-6s %-8s %s\n", "PIDNS", "CONTAINER", "PPID", "PID", "COMM", "ARGS");
}
</code></pre><p>把头文件引入和改进后的 bpftrace 脚本保存到 <code>execsnoop-container.bt</code> 文件中（你也可以在 <a href="https://github.com/feiskyer/ebpf-apps/blob/main/bpftrace/execsnoop-container.bt">GitHub</a> 上找到完整代码），然后打开一个新终端，运行下面的命令来执行：</p><pre><code class="language-bash">sudo bpftrace execsnoop-container.bt
</code></pre><p>接下来，分别在容器终端和主机终端中执行一个 <code>ls</code> 命令，就可以得到如下的输出：</p><pre><code class="language-bash">PIDNS        CONTAINER PPID   PID    COMM     ARGS
# 容器ls命令跟踪结果
4026532533   bash     41046  41335  bash    ls --color=auto

# 主机ls命令跟踪结果
4026531836   ubuntu.localdomain 40958  41356  bash    ls --color=auto
</code></pre><p>在输出中，容器 <code>ls</code> 命令跟踪结果中的 PID 命名空间 <code>4026532533</code> 跟上述容器中 <code>lsns</code> 结果是一致的，而主机名 <code>bash</code> 也跟运行容器时设置的 <code>--hostname name</code> 一致，因而我们很容易区分这条 <code>ls</code> 命令的来源。</p><p>你可以发现，<strong>只要理解了容器的基本原理，在跟踪过程中加入容器的基本信息，容器内外进程的跟踪和分析并没有本质的区别</strong>。</p><p>也许看到这里你会有疑问：这儿讲到的是内核态的跟踪，容器内外没有区别很正常。但如果是用户态的进程跟踪呢？你可以先思考一下，再继续下面的内容。</p><p>实际上，用户态进程的跟踪也是一样的，唯一需要注意的就是找到容器内二进制文件的正确路径。虽然容器文件系统在不同的 mount 命令空间中，但对于每个进程来说，Linux 都在 <code>/proc/[pid]/root</code> 处创建了一个链接（详细信息请参考 <a href="https://man7.org/linux/man-pages/man5/proc.5.html">proc 文件系统手册</a>）。因而，容器内的文件就可以通过 <code>/proc/[pid]/root</code> 在主机中访问。</p><p>你可以执行下面的命令，查询容器的 PID，进而再查询 bash 的 uprobe 列表：</p><pre><code class="language-bash"># 查询容器进程在主机命名空间中的PID
PID=$(docker inspect -f '{{.State.Pid}}' bash)

# 查询uprobe
sudo bpftrace -l "uprobe:/proc/$PID/root/usr/bin/bash:*"

# 跟踪bash:readline的结果
sudo bpftrace -e "uretprobe:/proc/$PID/root/usr/bin/bash:readline { printf(\"User %d executed %s in container\n\", uid, str(retval)); }"
</code></pre><h2>如何使用 eBPF 阻止容器的异常行为？</h2><p>了解了如何分析容器中的安全问题之后，我们再来看看如何阻止容器中的异常行为。</p><p>在 eBPF 之前，其实已经有很多的机制来阻止容器的异常行为，比如 <a href="https://docs.docker.com/engine/security/apparmor/">AppArmor</a>、<a href="https://docs.docker.com/engine/security/seccomp/">Seccomp</a>、<a href="https://man7.org/linux/man-pages/man7/capabilities.7.html">Capabilities</a> 等。这些机制虽然好用，但它们的缺点也很明显，那就是它们都需要预先配置好相关的安全策略。在检测到安全隐患之后，更新策略到策略生效中间总有一定的延迟。</p><p>而 eBPF 则可以在检测到安全隐患时，实时去触发安全策略的执行。比如，Linux 5.3 版本中新增的辅助函数 <code>bpf_send_signal()</code> 可以直接把有安全隐患的进程杀死，而 Linux 5.7 新增的 LSM 插桩则可以扩展 Linux 安全模块的审计和策略执行（想了解 LSM 钩子的详细信息，你可以参考内核头文件 <a href="https://elixir.bootlin.com/linux/v5.13/source/include/linux/lsm_hooks.h">include/linux/lsm_hooks.h</a>）。在网络安全方面，XDP 程序还可以在网络协议栈之前丢弃数据包，减少非法网络请求对系统资源的消耗。</p><p>以上一小节的二进制命令执行为例，你可以在 bpftrace 中调用 <code>signal()</code> 函数，给当前进程发送特定信号：</p><pre><code class="language-c++">tracepoint:syscalls:sys_enter_execve
/comm == "bash"/ {
  $task = (struct task_struct *)curtask;
  $cname = $task-&gt;nsproxy-&gt;uts_ns-&gt;name.nodename;
  printf("Killing shell command in container %s: %s ", $cname, $pidns, comm); join(args-&gt;argv);
 signal(9); /* SIGKILL */
}
</code></pre><p>这段代码中的具体内容含义如下：</p><ul>
<li><code>/comm == "bash"/</code> 表示对进程名称进行过滤，只处理 Bash 进程；</li>
<li><code>signal(9)</code> 表示向进程发送 SIGKILL 信号，即杀死进程。</li>
</ul><p>加入与上一小节中相同的头文件，然后把它保存到 <code>block-container-shell.bt</code> 文件（你也可以在 <a href="https://github.com/feiskyer/ebpf-apps/blob/main/bpftrace/block-container-shell.bt">GitHub</a> 上找到完整代码）中，然后运行下面的命令来执行：</p><pre><code class="language-bash">sudo bpftrace --unsafe block-container-shell.bt
</code></pre><p>接着，回到容器终端，执行任意命令都会失败，并且失败信息都是 <code>Killed</code> 。</p><p>需要注意，这种直接杀死进程的方法实际上比较危险，如果出现误杀，可能会导致大量进程都无法正常启动。所以 bpftrace 要求你加上 <code>--unsafe</code>选项后才可以正常运行。</p><h2>小结</h2><p>今天，我带你一起梳理了 eBPF 的安全能力，并以容器应用为例，带你用 eBPF 分析并阻止了容器中的命令执行。</p><p>eBPF 所支持的 kprobe、uprobe、tracepoint 等各类探针已经非常全面地涵盖了从应用到内核中的各类安全问题相关的跟踪点，再配合其开销低、实时性好，且不需要修改并重新编译内核和应用等特性，使 eBPF 特别适合用于安全事件的动态监测和实时分析诊断。对于安全策略的执行，eBPF 也已经支持了 LSM 钩子、向进程发送信号、丢弃网络数据包等各类操作。</p><p>在这一讲的最后，我想提醒你：既然 eBPF 提供了这么强大的功能，在用好 eBPF 这些丰富特性的同时，你也要特别留意 eBPF 程序自身的安全性。比如，禁止普通用户和普通容器应用运行 eBPF 程序，而只允许管理员和系统服务运行。对容器来说，你可以利用 Capabilities 禁止普通容器的 <code>CAP_PERFMON</code>、<code>CAP_BPF</code>和 <code>CAP_SYS_ADMIN</code>等权限。</p><h2>思考题</h2><p>最后，我想邀请你来聊一聊：</p><ol>
<li>在了解 eBPF 之前，你是如何监测、分析和阻止容器的安全问题的？</li>
<li>在阻止容器 Bash 命令执行的案例中，除了杀死进程之外，还有哪些其他的方法？</li>
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
  <div class="_2_QraFYR_0">另外分享一个经验，结构体 nsproxy 的 net 字段（代表进程所在的网络命名空间）在追踪网络包在宿主机 -&gt; 容器传递过程时尤其有用，可以比较清楚的看到网络包从宿主机的网络命名空间传递到容器独立的网络命名空间之中，协助更好的理解容器网络模型。（把网络设备名字也打印出来会更好。）<br><br>倪老师文中的示例程序 execsnoop-container.bt 使用了结构体 nsproxy 的 pid_ns_for_children 获取 PID 命名空间，稍微改动了一下，换成 NET 命名空间，把具体的 id 打出来。<br><br>输出示例（shell1 执行 docker run ... bash, shell2 追踪 execve 事件，可以看到 bash 进程在一个新的网络命名空间下执行。）：<br><br>shell 1 # docker exec -ti vibrant_jepsen bash<br>root@c340ba5cb9de:&#47;#<br><br>shell2 # # .&#47;execve.bt<br>Attaching 3 probes...<br>NETNS          CONTAINER              PPID     PID     COMM         ARGS<br>4026531993   VM-56-211-centos   799603 181059 bash            docker exec -ti vibrant_jepsen bash<br>...<br>4026532351   c340ba5cb9de         181078   181083 runc:[2:INIT]   bash<br><br>具体源码：<br>======================================<br><br>#!&#47;usr&#47;bin&#47;env bpftrace<br><br>#include &lt;linux&#47;sched.h&gt;<br>#include &lt;linux&#47;nsproxy.h&gt;<br>#include &lt;linux&#47;utsname.h&gt;<br>&#47;&#47;#include &lt;linux&#47;pid_namespace.h&gt;<br>#include &lt;net&#47;net_namespace.h&gt;<br><br>BEGIN <br>{<br>  printf(&quot;%-12s %-18s %-6s %-6s %-16s %s\n&quot;, &quot;NETNS&quot;, &quot;CONTAINER&quot;, &quot;PPID&quot;, &quot;PID&quot;, &quot;COMM&quot;, &quot;ARGS&quot;);<br>}<br><br>tracepoint:syscalls:sys_enter_execve,<br>tracepoint:syscalls:sys_enter_execveat<br>{<br>  $task = (struct task_struct *)curtask;<br>  $netns = $task-&gt;nsproxy-&gt;net_ns-&gt;ns.inum;<br>  &#47;&#47;$pidns = $task-&gt;nsproxy-&gt;pid_ns_for_children-&gt;ns.inum;<br>  $cname = $task-&gt;nsproxy-&gt;uts_ns-&gt;name.nodename;<br>  printf(&quot;%-12ld %-18s %-6d %-6d %-16s&quot;, (uint64)$netns, $cname, curtask-&gt;parent-&gt;pid, pid, comm);<br>  join(args-&gt;argv);<br>}</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 非常棒的示例，谢谢分享👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-10 14:54:34</div>
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
  <div class="_2_QraFYR_0">1、曾使用过 sysdig，老版本通过插入内核模块的方式进行安全审计。后来 sysdig 支持了 eBPF driver，主要通过追踪系统调用分析可能的安全隐患。sysdig eBPF driver 实现比较简单，一共十几个 program，统一放在 probe.c 源文件，里面的思路借鉴下还是不错的。<br>2、觉得需要在 bash 自身上做手脚，比如把 bash 软链接到一个脚本文件，记录下 bash 执行记录，然后 直接 exit。或者修改 bash 配置文件 .bashrc，文件末尾直接 exit。两种方法都有局限性，如果被识破可以很容易绕过去。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. 谢谢分享sysdig的使用经验！<br>2. 嗯嗯，这是一种不错的方法。或者考虑极端一些，完全禁止掉Bash（比如所有进程都跑在容器中，容器内不允许安装任何SHELL）。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-10 12:10:08</div>
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
  <div class="_2_QraFYR_0">本节课是关于eBPF在安全方面的使用，微博上看到有人发的这个rookit：https:&#47;&#47;weibo.com&#47;tv&#47;show&#47;1034:4718242451882158?from=old_pc_videoshow，因此我对eBPF如何保证自身安全性很好奇，想请教下老师对于系统来说eBPF是一个两面利器，它自身设计与实现，还有在使用中该如何注意避免引入安全问题呢？<br><br>另外像bpf_send_signal辅助函数能够通过信号杀死进程，这是不是一种类似变成语言的unsafe方法，应该在实际中谨慎使用呢？<br><br>谢谢老师。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你说的非常正确，eBPF如果被恶意应用利用，也同样会带来不可预料的损失。所以，在系统维护中，还是需要对进程的权限有所限制。比如，不要用root用户运行进程、不要给容器特权或者ADMIN capability等等。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-09 20:57:58</div>
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
  <div class="_2_QraFYR_0">如果运行 bpftrace 找不到 BEGIN symbol，例如出现：<br>ERROR: Could not resolve symbol: &#47;proc&#47;self&#47;exe:BEGIN_trigger<br>就运行 <br>sudo apt install bpftrace-dbgsym<br>就行了哈</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-09 11:36:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/32/a8/d5bf5445.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>郑海成</span>
  </div>
  <div class="_2_QraFYR_0">https:&#47;&#47;elixir.bootlin.com&#47;linux&#47;v5.13&#47;source&#47;include&#47;linux&#47;sched.h#L657 老师，bootlin这个网站是有什么要求吗？为什么我所有的代码link都显示 &quot;This file does not exist.&quot;呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-13 17:29:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTIsEia7TcYPiaO53QIydh4tPonwnpktgrhLeJqg4sNa8s11XNVTVajrI9jKibHs0FYn0EW8d8t3EM8ibQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_89541f</span>
  </div>
  <div class="_2_QraFYR_0">请问我在阿里云的虚拟机（centos系统，内核版本4.18)中使用ip link加载xdp程序，若指定xdpdrv模式会报错，xdpgeneric可以。这是为啥？虚拟网卡只能使用generic模式吗？这样性能不能满足生产需求。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: xdpdrv模式需要网卡驱动的支持的，具体的驱动支持情况可以参考 https:&#47;&#47;github.com&#47;iovisor&#47;bcc&#47;blob&#47;master&#47;docs&#47;kernel-versions.md#xdp</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-09 19:25:06</div>
  </div>
</div>
</div>
</li>
</ul>