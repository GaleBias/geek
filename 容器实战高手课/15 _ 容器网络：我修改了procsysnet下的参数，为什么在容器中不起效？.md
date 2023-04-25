<audio title="15 _ 容器网络：我修改了procsysnet下的参数，为什么在容器中不起效？" src="https://static001.geekbang.org/resource/audio/ee/55/ee4cff36d3af749890cb4672ae0fc455.mp3" controls="controls"></audio> 
<p>你好，我是程远。</p><p>从这一讲开始，我们进入到了容器网络这个模块。容器网络最明显的一个特征就是它有自己的Network Namespace了。你还记得，在我们这个课程的<a href="https://time.geekbang.org/column/article/308108">第一讲</a>里，我们就提到过Network Namespace负责管理网络环境的隔离。</p><p>今天呢，我们更深入地讨论一下和Network Namespace相关的一个问题——容器中的网络参数。</p><p>和之前的思路一样，我们先来看一个问题。然后在解决问题的过程中，更深入地理解容器的网络参数配置。</p><h2>问题再现</h2><p>在容器中运行的应用程序，如果需要用到tcp/ip协议栈的话，常常需要修改一些网络参数（内核中网络协议栈的参数）。</p><p>很大一部分网络参数都在/proc文件系统下的<a href="https://www.kernel.org/doc/Documentation/sysctl/net.txt">/proc/sys/net/</a>目录里。</p><p><strong>修改这些参数主要有两种方法：一种方法是直接到/proc文件系统下的"/proc/sys/net/"目录里对参数做修改；还有一种方法是使用<a href="https://man7.org/linux/man-pages/man8/sysctl.8.html">sysctl</a>这个工具来修改。</strong></p><p>在启动容器之前呢，根据我们的需要我们在宿主机上已经修改过了几个参数，也就是说这些参数的值已经不是内核里原来的缺省值了.</p><p>比如我们改了下面的几个参数：</p><pre><code class="language-shell"># # The default value:
# cat /proc/sys/net/ipv4/tcp_congestion_control
cubic
# cat /proc/sys/net/ipv4/tcp_keepalive_time
7200
# cat /proc/sys/net/ipv4/tcp_keepalive_intvl
75
# cat /proc/sys/net/ipv4/tcp_keepalive_probes
9
 
# # To update the value:
# echo bbr &gt; /proc/sys/net/ipv4/tcp_congestion_control
# echo 600 &gt; /proc/sys/net/ipv4/tcp_keepalive_time
# echo 10 &gt; /proc/sys/net/ipv4/tcp_keepalive_intvl
# echo 6 &gt; /proc/sys/net/ipv4/tcp_keepalive_probes
#
 
# # Double check the value after update:
# cat /proc/sys/net/ipv4/tcp_congestion_control
bbr
# cat /proc/sys/net/ipv4/tcp_keepalive_time
600
# cat /proc/sys/net/ipv4/tcp_keepalive_intvl
10
# cat /proc/sys/net/ipv4/tcp_keepalive_probes
6
</code></pre><!-- [[[read_end]]] --><p>然后我们启动一个容器， 再来查看一下容器里这些参数的值。</p><p>你可以先想想，容器里这些参数的值会是什么？我最初觉得容器里参数值应该会继承宿主机Network Namesapce里的值，实际上是不是这样呢？</p><p>我们还是先按下面的脚本，启动容器，然后运行 <code>docker exec</code> 命令一起看一下：</p><pre><code class="language-shell"># docker run -d --name net_para centos:8.1.1911 sleep 3600
deec6082bac7b336fa28d0f87d20e1af21a784e4ef11addfc2b9146a9fa77e95
# docker exec -it net_para bash
[root@deec6082bac7 /]# cat /proc/sys/net/ipv4/tcp_congestion_control
bbr
[root@deec6082bac7 /]# cat /proc/sys/net/ipv4/tcp_keepalive_time
7200
[root@deec6082bac7 /]# cat /proc/sys/net/ipv4/tcp_keepalive_intvl
75
[root@deec6082bac7 /]# cat /proc/sys/net/ipv4/tcp_keepalive_probes
9
</code></pre><p>从这个结果我们看到，tcp_congestion_control的值是bbr，和宿主机Network Namespace里的值是一样的，而其他三个tcp keepalive相关的值，都不是宿主机Network Namespace里设置的值，而是原来系统里的缺省值了。</p><p>那为什么会这样呢？在分析这个问题之前，我们需要先来看看Network Namespace这个概念。</p><h2>知识详解</h2><h3>如何理解Network Namespace？</h3><p>对于Network Namespace，我们从字面上去理解的话，可以知道它是在一台Linux节点上对网络的隔离，不过它具体到底隔离了哪部分的网络资源呢？</p><p>我们还是先来看看操作手册，在<a href="https://man7.org/linux/man-pages/man7/network_namespaces.7.html">Linux Programmer’s Manual</a>里对Network Namespace有一个段简短的描述，在里面就列出了最主要的几部分资源，它们都是通过Network Namespace隔离的。</p><p>我把这些资源给你做了一个梳理：</p><p>第一种，网络设备，这里指的是lo，eth0等网络设备。你可以通过 <code>ip link</code>命令看到它们。</p><p>第二种是IPv4和IPv6协议栈。从这里我们可以知道，IP层以及上面的TCP和UDP协议栈也是每个Namespace独立工作的。</p><p>所以IP、TCP、UDP的很多协议，它们的相关参数也是每个Namespace独立的，这些参数大多数都在 /proc/sys/net/目录下面，同时也包括了TCP和UDP的port资源。</p><p>第三种，IP路由表，这个资源也是比较好理解的，你可以在不同的Network Namespace运行 <code>ip route</code> 命令，就能看到不同的路由表了。</p><p>第四种是防火墙规则，其实这里说的就是iptables规则了，每个Namespace里都可以独立配置iptables规则。</p><p>最后一种是网络的状态信息，这些信息你可以从/proc/net 和/sys/class/net里得到，这里的状态基本上包括了前面4种资源的的状态信息。</p><h3>Namespace的操作</h3><p>那我们怎么建立一个新的Network Namespace呢？</p><p><strong>我们可以通过系统调用clone()或者unshare()这两个函数来建立新的Network Namespace。</strong></p><p>下面我们会讲两个例子，带你体会一下这两个方法具体怎么用。</p><p>第一种方法呢，是在新的进程创建的时候，伴随新进程建立，同时也建立出新的Network Namespace。这个方法，其实就是通过clone()系统调用带上CLONE_NEWNET flag来实现的。</p><p>Clone建立出来一个新的进程，这个新的进程所在的Network Namespace也是新的。然后我们执行 <code>ip link</code> 命令查看Namespace里的网络设备，就可以确认一个新的Network Namespace已经建立好了。</p><p>具体操作你可以看一下<a href="https://github.com/chengyli/training/blob/master/net/namespace/clone-ns.c">这段代码</a>。</p><pre><code class="language-shell">int new_netns(void *para)
{
            printf("New Namespace Devices:\n");
            system("ip link");
            printf("\n\n");
 
            sleep(100);
            return 0;
}
 
int main(void)
{
            pid_t pid;
 
            printf("Host Namespace Devices:\n");
            system("ip link");
            printf("\n\n");
 
            pid =
                clone(new_netns, stack + STACK_SIZE, CLONE_NEWNET | SIGCHLD, NULL);
            if (pid == -1)
                        errExit("clone");
 
            if (waitpid(pid, NULL, 0) == -1)
                        errExit("waitpid");
 
            return 0;
}
</code></pre><p>第二种方法呢，就是调用unshare()这个系统调用来直接改变当前进程的Network Namespace，你可以看一下<a href="https://github.com/chengyli/training/blob/master/net/namespace/unshare-ns.c">这段代码</a>。</p><pre><code class="language-shell">int main(void)
{
            pid_t pid;
 
            printf("Host Namespace Devices:\n");
            system("ip link");
            printf("\n\n");
 
            if (unshare(CLONE_NEWNET) == -1)
                        errExit("unshare");
 
            printf("New Namespace Devices:\n");
            system("ip link");
            printf("\n\n");
 
            return 0;
}
</code></pre><p>其实呢，不仅是Network Namespace，其它的Namespace也是通过clone()或者unshare()系统调用来建立的。</p><p>而创建容器的程序，比如<a href="https://github.com/opencontainers/runc">runC</a>也是用unshare()给新建的容器建立Namespace的。</p><p>这里我简单地说一下runC是什么，我们用Docker或者containerd去启动容器，最后都会调用runC在Linux中把容器启动起来。</p><p>除了在代码中用系统调用来建立Network Namespace，我们也可以用命令行工具来建立Network Namespace。比如用 <code>ip netns</code> 命令，在下一讲学习容器网络配置的时候呢，我们会用到 <code>ip netns</code>，这里你先有个印象就行。</p><p>在Network Namespace 创建好了之后呢，我们可以在宿主机上运行 <code>lsns -t net</code> 这个命令来查看系统里已有的Network Namespace。当然，<code>lsns</code>也可以用来查看其它Namespace。</p><p>用 <code>lsns</code> 查看已有的Namespace后，我们还可以用 <code>nsenter</code> 这个命令进入到某个Network Namespace里，具体去查看这个Namespace里的网络配置。</p><p>比如下面的这个例子，用我们之前的clone()的例子里的代码，编译出clone-ns这个程序，运行后，再使用 <code>lsns</code> 查看新建的Network Namespace，并且用<code>nsenter</code>进入到这个Namespace，查看里面的lo device。</p><p>具体操作你可以参考下面的代码：</p><pre><code class="language-shell"># ./clone-ns &amp;
[1] 7732
# Host Namespace Devices:
1: lo: &lt;LOOPBACK,UP,LOWER_UP&gt; mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: &lt;BROADCAST,MULTICAST,UP,LOWER_UP&gt; mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 74:db:d1:80:54:14 brd ff:ff:ff:ff:ff:ff
3: docker0: &lt;NO-CARRIER,BROADCAST,MULTICAST,UP&gt; mtu 1500 qdisc noqueue state DOWN mode DEFAULT group default
    link/ether 02:42:0c:ff:2b:77 brd ff:ff:ff:ff:ff:ff
 
 
New Namespace Devices:
1: lo: &lt;LOOPBACK&gt; mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
 
# lsns -t net
        NS TYPE NPROCS   PID USER    NETNSID NSFS COMMAND
4026531992 net     283     1 root unassigned      /usr/lib/systemd/systemd --switched-root --system --deserialize 16
4026532241 net       1  7734 root unassigned      ./clone-ns
# nsenter -t 7734 -n ip addr
1: lo: &lt;LOOPBACK&gt; mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
</code></pre><h2>解决问题</h2><p>那理解了Network Namespace之后，我们再来看看这一讲最开始的问题，我们应该怎么来设置容器里的网络相关参数呢？</p><p>首先你要避免走入误区。从我们一开始的例子里，也可以看到，容器里Network Namespace的网络参数并不是完全从宿主机Host Namespace里继承的，也不是完全在新的Network Namespace建立的时候重新初始化的。</p><p>其实呢，这一点我们只要看一下内核代码中对协议栈的初始化函数，很快就可以知道为什么会有这样的情况。</p><p>在我们的例子里tcp_congestion_control的值是从Host Namespace里继承的，而tcp_keepalive相关的几个值会被重新初始化了。</p><p>在函数<a href="https://github.com/torvalds/linux/blob/v5.4/net/ipv4/tcp_ipv4.c#L2631">tcp_sk_init</a>()里，tcp_keepalive的三个参数都是重新初始化的，而tcp_congestion_control 的值是从Host Namespace里复制过来的。</p><pre><code class="language-shell">static int __net_init tcp_sk_init(struct net *net)
{
…
        net-&gt;ipv4.sysctl_tcp_keepalive_time = TCP_KEEPALIVE_TIME;
        net-&gt;ipv4.sysctl_tcp_keepalive_probes = TCP_KEEPALIVE_PROBES;
        net-&gt;ipv4.sysctl_tcp_keepalive_intvl = TCP_KEEPALIVE_INTVL;
 
…
        /* Reno is always built in */
        if (!net_eq(net, &amp;init_net) &amp;&amp;
            try_module_get(init_net.ipv4.tcp_congestion_control-&gt;owner))
                net-&gt;ipv4.tcp_congestion_control = init_net.ipv4.tcp_congestion_control;
        else
                net-&gt;ipv4.tcp_congestion_control = &amp;tcp_reno;
 
…
 
}

</code></pre><p>那么我们现在知道Network Namespace的网络参数是怎么初始化的了，你可能会问了，我在容器里也可以修改这些参数吗？</p><p>我们可以启动一个普通的容器，这里的“普通”呢，我指的不是"privileged"的那种容器，也就是在这个容器中，有很多操作都是不允许做的，比如mount一个文件系统。这个privileged容器概念，我们会在后面容器安全这一讲里详细展开，这里你有个印象。</p><p>那么在启动完一个普通容器后，我们尝试一下在容器里去修改"/proc/sys/net/"下的参数。</p><p>这时候你会看到，容器中"/proc/sys/"是只读mount的，那么在容器里是不能修改"/proc/sys/net/"下面的任何参数了。</p><pre><code class="language-shell"># docker run -d --name net_para centos:8.1.1911 sleep 3600
977bf3f07da90422e9c1e89e56edf7a59fab5edff26317eeb253700c2fa657f7
# docker exec -it net_para bash
[root@977bf3f07da9 /]# echo 600 &gt; /proc/sys/net/ipv4/tcp_keepalive_time
bash: /proc/sys/net/ipv4/tcp_keepalive_time: Read-only file system
[root@977bf3f07da9 /]# cat /proc/mounts | grep "proc/sys"
proc /proc/sys proc ro,relatime 0 0
</code></pre><p>为什么“/proc/sys/” 在容器里是只读mount呢？ 这是因为runC当初出于安全的考虑，把容器中所有/proc和/sys相关的目录缺省都做了read-only mount的处理。详细的说明你可以去看看这两个commits:</p><ul>
<li>
<p><a href="https://github.com/opencontainers/runc/commit/5a6b042e5395660ac8a6e3cc33227ca66df7c835">Mount /proc and /sys read-only, except in privileged containers</a></p>
</li>
<li>
<p><a href="https://github.com/opencontainers/runc/commit/73c607b7ad5cea5c913f96dff17bca668534ad18">Make /proc writable, but not /proc/sys and /proc/sysrq-trigger</a></p>
</li>
</ul><p>那我们应该怎么来修改容器中Network Namespace的网络参数呢？</p><p>当然，如果你有宿主机上的root权限，最简单粗暴的方法就是用我们之前说的"nsenter"工具，用它修改容器里的网络参数的。不过这个方法在生产环境里显然是不会被允许的，因为我们不会允许用户拥有宿主机的登陆权限。</p><p>其次呢，一般来说在容器中的应用已经启动了之后，才会做这样的修改。也就是说，很多tcp链接已经建立好了，那么即使新改了参数，对已经建立好的链接也不会生效了。这就需要重启应用，我们都知道生产环境里通常要避免应用重启，那这样做显然也不合适。</p><p>通过刚刚的排除法，我们推理出了网络参数修改的“正确时机”：想修改Network Namespace里的网络参数，要选择容器刚刚启动，而容器中的应用程序还没启动之前进行。</p><p>其实，runC也在对/proc/sys目录做read-only mount之前，预留出了修改接口，就是用来修改容器里 "/proc/sys"下参数的，同样也是sysctl的参数。</p><p>而Docker的<a href="https://docs.docker.com/engine/reference/commandline/run/#configure-namespaced-kernel-parameters-sysctls-at-runtime">–sysctl</a>或者Kubernetes里的<a href="https://kubernetes.io/docs/tasks/administer-cluster/sysctl-cluster/">allowed-unsafe-sysctls</a>特性也都利用了runC的sysctl参数修改接口，允许容器在启动时修改容器Namespace里的参数。</p><p>比如，我们可以试一下docker –sysctl，这时候我们会发现，在容器的Network Namespace里，/proc/sys/net/ipv4/tcp_keepalive_time这个网络参数终于被修改了！</p><pre><code class="language-shell"># docker run -d --name net_para --sysctl net.ipv4.tcp_keepalive_time=600 centos:8.1.1911 sleep 3600
7efed88a44d64400ff5a6d38fdcc73f2a74a7bdc3dbc7161060f2f7d0be170d1
# docker exec net_para cat /proc/sys/net/ipv4/tcp_keepalive_time
600
</code></pre><h2>重点总结</h2><p>好了，今天的课我们讲完了，那么下面我来给你做个总结。</p><p>今天我们讨论问题是容器中网络参数的问题，因为是问题发生在容器里，又是网络的参数，那么自然就和Network Namespace有关，所以我们首先要理解Network Namespace。</p><p>Network Namespace可以隔离网络设备，ip协议栈，ip路由表，防火墙规则，以及可以显示独立的网络状态信息。</p><p>我们可以通过clone()或者unshare()系统调用来建立新的Network Namespace。</p><p>此外，还有一些工具"ip""netns""unshare""lsns"和"nsenter"，也可以用来操作Network Namespace。</p><p>这些工具的适用条件，我用表格的形式整理如下，你可以做个参考。</p><p><img src="https://static001.geekbang.org/resource/image/6d/cd/6da09e062c0644492af26823343c6ecd.jpeg?wh=3200*1800" alt=""><br>
接着我们分析了如何修改普通容器（非privileged）的网络参数。</p><p>由于安全的原因，普通容器的/proc/sys是read-only mount的，所以在容器启动以后，我们无法在容器内部修改/proc/sys/net下网络相关的参数。</p><p>这时可行的方法是<strong>通过runC sysctl相关的接口，在容器启动的时候对容器内的网络参数做配置。</strong></p><p>这样一来，想要修改网络参数就可以这么做：如果是使用Docker，我们可以加上"—sysctl"这个参数；而如果使用Kubernetes的话，就需要用到"allowed unsaft sysctl"这个特性了。</p><h2>思考题</h2><p>这一讲中，我们提到了可以使用"nsenter"这个工具，从宿主机上修改容器里的/proc/sys/net/下的网络参数，你可以试试看具体怎么修改。</p><p>欢迎你在留言区分享你的收获和疑问。如果这篇文章对你有帮助，也欢迎转发给你的同事和朋友，一起交流探讨。</p>
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
  <div class="_2_QraFYR_0">docker exec、kubectl exec、ip netns exec、nsenter 等命令原理相同，都是基于 setns 系统调用，切换至指定的一个或多个 namespace(s)。 </div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-19 14:17:09</div>
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
  <div class="_2_QraFYR_0"># nsenter -t &lt;pid&gt; -n bash -c &#39;echo 600 &gt; &#47;proc&#47;sys&#47;net&#47;ipv4&#47;tcp_keepalive_time&#39; （root 用户）<br>$ sudo nsenter -t &lt;pid&gt; -n sudo bash -c &#39;echo 600 &gt; &#47;proc&#47;sys&#47;net&#47;ipv4&#47;tcp_keepalive_time&#39; （非 root 用户）<br><br>其中，&lt;pid&gt; 表示容器 init 进程在宿主机上看到的 PID。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-19 14:12:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/23/81/60/71ed6ac7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>谢哈哈</span>
  </div>
  <div class="_2_QraFYR_0">宿主机的进入容器网络地址空间通过nsenter --target $(docker inspect -f {.State.Pid}) --net</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-19 16:14:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83eqYia7VlBNus8kg5daQyb0AywUMMFWH2eUyTIDfBa3tua0Giaxtmx9icxLWyoHTHjo9bFoGOLWMYIdyA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>麻瓜镇</span>
  </div>
  <div class="_2_QraFYR_0">为什么有的参数是从host namespace复制，有的参数直接使用缺省值呢？为什么要这样设计？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-09 14:07:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/94/b0/b073fe8b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>aMaMiMoU</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，有几个问题能否帮忙解答下，谢谢<br>1.在&#47;proc&#47;sys&#47;net 的诸多参数里，如何确认哪些是host level 哪些是容器level的呢？<br>2.对于host level的这些参数，在启动容器的时候通过sysctl能修改么？如果能修改，是不是相当于同时影响了同host里其他容器的运行时参数呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: @AMaMiMou<br>很好的问题，对于&#47;proc&#47;sys&#47;net下的参数，你在容器中看到的，基本都是network namespace下的。<br>对于容器启动的时候，runc只允许修改namesapce下的参数值，而不会修改host相关的值。<br><br>可以参看一下runc的sysctl validation的代码：<br><br>https:&#47;&#47;github.com&#47;opencontainers&#47;runc&#47;blob&#47;ff819c7e9184c13b7c2607fe6c30ae19403a7aff&#47;libcontainer&#47;configs&#47;validate&#47;validator.go#L135</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-26 21:46:43</div>
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
  <div class="_2_QraFYR_0">老师，咨询一个问题，就是我有一个容器里面有两个服务，映射出8000和9000的端口，在容器内会出现8000端口的服务访问宿主机ip：9000的端口不通，但是我service iptables stop ; seriver docker stop ;<br>server docker start ; 就可以访问了。一旦reboot就不行了。请问是怎么样的问题<br> </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: &gt; service iptables stop<br>看上去应该是 iptables stop 后把原来的iptables rules flush了，就可以工作了。<br>你可以查看一下具体是那条iptables规则阻止了访问。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-22 15:40:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/24/62/c4/be92518b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>🐭</span>
  </div>
  <div class="_2_QraFYR_0">既然nsenter与docker exec 原理一样，为啥nsenter修改proc&#47;sys&#47;net不会报错无权限呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: docker exec 同时进入了容器的pid&#47;mnt&#47;net namespace, 而 nsenter 修改参数的时候，只是进入了容器的net namespace.</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-03 22:51:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKJrOl63enWXCRxN0SoucliclBme0qrRb19ATrWIOIvibKIz8UAuVgicBMibIVUznerHnjotI4dm6ibODA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Helios</span>
  </div>
  <div class="_2_QraFYR_0">这些问题文档上都没写，还是老师功力高，场景多。<br>请教个问题，对于proc文件系统的其他目录容器怎么隔离的呢，比如在容器里面free命令看到的是宿主机的内存。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: &#47;proc下的内容大部分是没有隔离的，就想你说的&#47;proc&#47;meminfo, &#47;proc&#47;stat 在容器中看到的都和宿主机上的是一样的。<br>其他有一些因为namespace不同而不同的&#47;proc的内容，<br>比如 &#47;proc&#47;&lt;pid&gt;&#47;， 容器下只能看到自己pid namespace里的进程pid。<br>&#47;proc&#47;mounts的内容和mount namespace相关。<br>还有IPC namesapce相关的一些&#47;proc下的参数，等等。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-22 12:49:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/f2/73/1c7bceae.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>乔纳森</span>
  </div>
  <div class="_2_QraFYR_0">我们是在 initContainers 中 执行 如下来修改容器内的内核参数的，需要privileged: true<br>mount -o remount rw &#47;proc&#47;sys<br>          sysctl -w net.core.somaxconn=65535</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，如果container有privileged的权限，那么在容器中几乎和host有同等权限了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-04 12:06:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/10/bb/f1061601.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Demon.Lee</span>
  </div>
  <div class="_2_QraFYR_0">老师，为啥隔离的这些网络参数不和 &#47;sys&#47;fs&#47;cgroup&#47;net_cls,net_prio,cpu,pid 等一样，统一放在&#47;sys&#47;fs&#47;cgroup&#47;下面，而是跟宿主机共用一套 ？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 一个是namespace, 主要是资源隔离 <br>另外一个是cgroup，主要是资源的划分。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-02 10:22:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/a5/3c/7c0d2e57.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>程序员老王</span>
  </div>
  <div class="_2_QraFYR_0">网卡是通过端口号来区分栈数据吧，命名空间在这里隔离是网络参数配置吗？还是网卡</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: @王传义<br>你这里说的“端口”是指tcp&#47;udp层的端口号吗？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-18 12:20:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/JWoFanyWDk7lWL7g8rLYI0icH1XOVoCyjR9HoMzliauxggPSWWeYVleqKwiaUnBEChfIctoFzVoBqqVT3Lot18Srg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_fd78c0</span>
  </div>
  <div class="_2_QraFYR_0">想问一下，容器启动时网络是桥接模式，启动以后，如何新增容器中端口到host端口的映射？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-24 08:59:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/8b/65/0f1f9a10.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小Y</span>
  </div>
  <div class="_2_QraFYR_0">来到网络的章节基本不太懂，得多听几遍，多补充补充了😆</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">编辑回复: 知道了盲区也是好事，可以去补一补。一定可以的～</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-28 09:36:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/60/a1/8f003697.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>静心</span>
  </div>
  <div class="_2_QraFYR_0">不仅学习到了docker命令的--sysctl参数的用法，还了解到了其原理，真是酣畅淋漓。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-21 11:25:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/d6/43/0704d7db.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>cc</span>
  </div>
  <div class="_2_QraFYR_0">&quot;tcp_congestion_control 的值是 bbr，和宿主机 Network Namespace 里的值是一样的，而其他三个 tcp keepalive 相关的值，都不是宿主机 Network Namespace 里设置的值，而是原来系统里的缺省值了。&quot;<br>------<br>这里比较困惑，为什么要这样设计呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-29 15:35:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/27/ce/58/71ed845f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Dexter</span>
  </div>
  <div class="_2_QraFYR_0">pid =  clone(new_netns, stack + STACK_SIZE, CLONE_NEWNET | SIGCHLD, NULL);  代码例子中的一段， clone系统调用中的参数，new_netns和 CLONE_NEWNET flag这两者有什么关系呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-16 09:45:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/04/3f/f28d76c5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Shone</span>
  </div>
  <div class="_2_QraFYR_0">CY老师，这里我又想起最近遇到的rp_filter 的问题，因为我们的slb用的是ipvs tunnel，需要给应用pod设置tun0设备并且绑定vip，同时还需要设置rp_filter为0或者2。这里我们遇到的问题是有时候node上的这个配置会被改成1，导致ipvs数据面不工作，解决办法是把node上这个值设为0，但是我发现这个时候pod eth0这个值是1，它对应的calico这个值也是1，所以这种情况下好像是node的rp_filter会覆盖其它的设置，对这里不是很理解了。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-10 16:58:57</div>
  </div>
</div>
</div>
</li>
</ul>