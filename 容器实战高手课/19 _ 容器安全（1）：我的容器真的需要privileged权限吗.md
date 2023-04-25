<audio title="19 _ 容器安全（1）：我的容器真的需要privileged权限吗" src="https://static001.geekbang.org/resource/audio/92/4f/9263499af034a454b28bcd888190c14f.mp3" controls="controls"></audio> 
<p>你好，我是程远。从今天这一讲，我们进入到了容器安全的模块。</p><p>容器安全是一个很大的话题，容器的安全性很大程度是由容器的架构特性所决定的。比如容器与宿主机共享Linux内核，通过Namespace来做资源的隔离，通过shim/runC的方式来启动等等。</p><p>这些容器架构特性，在你选择使用容器之后，作为使用容器的用户，其实你已经没有多少能力去对架构这个层面做安全上的改动了。你可能会说用<a href="https://katacontainers.io/">Kata Container</a>、<a href="https://gvisor.dev/">gVisor</a> 就是安全“容器”了。不过，Kata或者gVisor只是兼容了容器接口标准，而内部的实现完全是另外的技术了。</p><p>那么对于使用容器的用户，在运行容器的时候，在安全方面可以做些什么呢？我们主要可以从这两个方面来考虑：第一是赋予容器合理的capabilities，第二是在容器中以非root用户来运行程序。</p><p>为什么是这两点呢？我通过两讲的内容和你讨论一下，这一讲我们先来看容器的capabilities的问题。</p><h2>问题再现</h2><p>刚刚使用容器的同学，往往会发现用缺省 <code>docker run</code>的方式启动容器后，在容器里很多操作都是不允许的，即使是以root用户来运行程序也不行。</p><p>我们用下面的<a href="https://github.com/chengyli/training/tree/main/security/capability">例子</a>来重现一下这个问题。我们先运行<code>make image</code> 做个容器镜像，然后运行下面的脚本：</p><!-- [[[read_end]]] --><pre><code class="language-shell"># docker run --name iptables -it registry/iptables:v1 bash
[root@0b88d6486149 /]# iptables -L
iptables v1.8.4 (nf_tables): Could not fetch rule set generation id: Permission denied (you must be root)
 
[root@0b88d6486149 /]# id
uid=0(root) gid=0(root) groups=0(root)
</code></pre><p>在这里，我们想在容器中运行 <code>iptables</code> 这个命令，来查看一下防火墙的规则，但是执行命令之后，你会发现结果输出中给出了"Permission denied (you must be root)"的错误提示，这个提示要求我们用root用户来运行。</p><p>不过在容器中，我们现在已经是以root用户来运行了，么为什么还是不可以运行"iptables"这条命令呢？</p><p>你肯定会想到，是不是容器中又做了别的权限限制？如果你去查一下资料，就会看到启动容器有一个"privileged"的参数。我们可以试一下用上这个参数，没错，我们用了这个参数之后，iptables这个命令就执行成功了。</p><pre><code class="language-shell"># docker stop iptables;docker rm iptables
iptables
iptables
# docker run --name iptables --privileged -it registry/iptables:v1 bash
[root@44168f4b9b24 /]# iptables -L
Chain INPUT (policy ACCEPT)
target     prot opt source               destination
 
Chain FORWARD (policy ACCEPT)
target     prot opt source               destination
 
Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
</code></pre><p>看上去，我们用了一个配置参数就已经解决了问题，似乎很容易。不过这里我们可以进一步想想，用"privileged"参数来解决问题，是不是一个合理的方法呢？用它会有什么问题吗？</p><p>要回答这些问题，我们先来了解一下"privileged"是什么意思。从Docker的<a href="https://github.com/moby/moby/blob/17.03.x/daemon/exec_linux.go#L25">代码</a>里，我们可以看到，如果配置了privileged的参数的话，就会获取所有的capabilities，那什么是capabilities呢？</p><pre><code class="language-shell">            if ec.Privileged {
                        p.Capabilities = caps.GetAllCapabilities()
            }
</code></pre><h2>基本概念</h2><h3>Linux capabilities</h3><p>要了解Linux capabilities的定义，我们可以先查看一下"Linux Programmer's Manual"中关于<a href="https://man7.org/linux/man-pages/man7/capabilities.7.html">Linux capabilities</a>的描述。</p><p>在Linux capabilities出现前，进程的权限可以简单分为两类，第一类是特权用户的进程（进程的有效用户ID是0，简单来说，你可以认为它就是root用户的进程），第二类是非特权用户的进程（进程的有效用户ID是非0，可以理解为非root用户进程）。</p><p>特权用户进程可以执行Linux系统上的所有操作，而非特权用户在执行某些操作的时候就会被内核限制执行。其实这个概念，也是我们通常对Linux中root用户与非root用户的理解。</p><p>从kernel 2.2开始，Linux把特权用户所有的这些“特权”做了更详细的划分，这样被划分出来的每个单元就被称为capability。</p><p>所有的capabilities都在<a href="https://man7.org/linux/man-pages/man7/capabilities.7.html">Linux capabilities</a>的手册列出来了，你也可以在内核的文件<a href="https://github.com/torvalds/linux/blob/v5.4/include/uapi/linux/capability.h#L113">capability.h</a>中看到所有capabilities的定义。</p><p>对于任意一个进程，在做任意一个特权操作的时候，都需要有这个特权操作对应的capability。</p><p>比如说，运行iptables命令，对应的进程需要有CAP_NET_ADMIN这个capability。如果要mount一个文件系统，那么对应的进程需要有CAP_SYS_ADMIN这个capability。</p><p>我还要提醒你的是，CAP_SYS_ADMIN这个capability里允许了大量的特权操作，包括文件系统，交换空间，还有对各种设备的操作，以及系统调试相关的调用等等。</p><p>在普通Linux节点上，非root用户启动的进程缺省没有任何Linux capabilities，而root用户启动的进程缺省包含了所有的Linux capabilities。</p><p>我们可以做个试验，对于root用户启动的进程，如果把CAP_NET_ADMIN这个capability移除，看看它是否还可以运行iptables。</p><p>在这里我们要用到<a href="https://man7.org/linux/man-pages/man1/capsh.1.html">capsh</a>这个工具，对这个工具不熟悉的同学可以查看超链接。接下来，我们就用capsh执行下面的这个命令：</p><pre><code class="language-shell"># sudo /usr/sbin/capsh --keep=1 --user=root   --drop=cap_net_admin  --   -c './iptables -L;sleep 100'
Chain INPUT (policy ACCEPT)
target     prot opt source               destination
 
Chain FORWARD (policy ACCEPT)
target     prot opt source               destination
 
Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
iptables: Permission denied (you must be root).
</code></pre><p>这时候，我们可以看到即使是root用户，如果把"CAP_NET_ADMIN"给移除了，那么在执行iptables的时候就会看到"Permission denied (you must be root)."的提示信息。</p><p>同时，我们可以通过/proc文件系统找到对应进程的status，这样就能确认进程中的CAP_NET_ADMIN是否已经被移除了。</p><pre><code class="language-shell"># ps -ef | grep sleep
root     22603 22275  0 19:44 pts/1    00:00:00 sudo /usr/sbin/capsh --keep=1 --user=root --drop=cap_net_admin -- -c ./iptables -L;sleep 100
root     22604 22603  0 19:44 pts/1    00:00:00 /bin/bash -c ./iptables -L;sleep 100
 
# cat /proc/22604/status | grep Cap
CapInh:            0000000000000000
CapPrm:          0000003fffffefff
CapEff:             0000003fffffefff
CapBnd:          0000003fffffefff
CapAmb:         0000000000000000
</code></pre><p>运行上面的命令查看 /proc/<pid>/status里Linux capabilities的相关参数之后，我们可以发现，输出结果中包含5个Cap参数。</pid></p><p>这里我给你解释一下， 对于当前进程，直接影响某个特权操作是否可以被执行的参数，是"CapEff"，也就是"Effective capability sets"，这是一个bitmap，每一个bit代表一项capability是否被打开。</p><p>在Linux内核<a href="https://github.com/torvalds/linux/blob/v5.4/include/uapi/linux/capability.h#L203">capability.h</a>里把CAP_NET_ADMIN的值定义成12，所以我们可以看到"CapEff"的值是"0000003fffffefff"，第4个数值是16进制的"e"，而不是f。</p><p>这表示CAP_NET_ADMIN对应的第12-bit没有被置位了（0xefff = 0xffff &amp; (~(1 &lt;&lt; 12))），所以这个进程也就没有执行iptables命令的权限了。</p><p>对于进程status中其他几个capabilities相关的参数，它们还需要和应用程序文件属性中的capabilities协同工作，这样才能得到新启动的进程最终的capabilities参数的值。</p><p>我们看下面的图，结合这张图看后面的讲解：</p><p><img src="https://static001.geekbang.org/resource/image/90/5c/906a996776f84d8f856cc7f62589095c.jpeg?wh=1920*1080" alt=""></p><p>如果我们要新启动一个程序，在Linux里的过程就是先通过fork()来创建出一个子进程，然后调用execve()系统调用读取文件系统里的程序文件，把程序文件加载到进程的代码段中开始运行。</p><p>就像图片所描绘的那样，这个新运行的进程里的相关capabilities参数的值，是由它的父进程以及程序文件中的capabilities参数值计算得来的。</p><p>具体的计算过程你可以看<a href="https://man7.org/linux/man-pages/man7/capabilities.7.html">Linux capabilities</a>的手册中的描述，也可以读一下网上的这两篇文章：</p><ul>
<li><a href="https://blog.container-solutions.com/linux-capabilities-why-they-exist-and-how-they-work">Capabilities: Why They Exist and How They Work</a></li>
<li><a href="https://blog.container-solutions.com/linux-capabilities-in-practice">Linux Capabilities in Practice</a></li>
</ul><p>我就不对所有的进程和文件的capabilities集合参数和算法挨个做解释了，感兴趣的话你可以自己详细去看看。</p><p>这里你只要记住最重要的一点，<strong>文件中可以设置capabilities参数值，并且这个值会影响到最后运行它的进程。</strong>比如，我们如果把iptables的应用程序加上 CAP_NET_ADMIN的capability，那么即使是非root用户也有执行iptables的权限了。</p><pre><code class="language-shell">$ id
uid=1000(centos) gid=1000(centos) groups=1000(centos),10(wheel)
$ sudo setcap cap_net_admin+ep ./iptables
$ getcap ./iptables
./iptables = cap_net_admin+ep
$./iptables -L
Chain INPUT (policy ACCEPT)
target     prot opt source               destination
 
Chain FORWARD (policy ACCEPT)
target     prot opt source               destination
DOCKER-USER  all  --  anywhere             anywhere
DOCKER-ISOLATION-STAGE-1  all  --  anywhere             anywhere
ACCEPT     all  --  anywhere             anywhere             ctstate RELATED,ESTABLISHED
DOCKER     all  --  anywhere             anywhere
ACCEPT     all  --  anywhere             anywhere
ACCEPT     all  --  anywhere             anywhere
…
</code></pre><p>好了，关于Linux capabilities的内容到这里我们就讲完了，其实它就是把Linux root用户原来所有的特权做了细化，可以更加细粒度地给进程赋予不同权限。</p><h2>解决问题</h2><p>我们搞懂了Linux capabilities之后，那么对privileged的容器也很容易理解了。<strong>Privileged的容器也就是允许容器中的进程可以执行所有的特权操作。</strong></p><p>因为安全方面的考虑，容器缺省启动的时候，哪怕是容器中root用户的进程，系统也只允许了15个capabilities。这个你可以查看<a href="https://github.com/opencontainers/runc/blob/v1.0.0-rc92/libcontainer/SPEC.md#security">runC spec文档中的security</a> 部分，你也可以查看容器init进程status里的Cap参数，看一下容器中缺省的capabilities。</p><pre><code class="language-shell"># docker run --name iptables -it registry/iptables:v1 bash
[root@e54694652a42 /]# cat /proc/1/status  |grep Cap
CapInh:            00000000a80425fb
CapPrm:          00000000a80425fb
CapEff:              00000000a80425fb
CapBnd:          00000000a80425fb
CapAmb:         0000000000000000
</code></pre><p>我想提醒你，当我们发现容器中运行某个程序的权限不够的时候，并不能“偷懒”把容器设置为"privileged"，也就是把所有的capabilities都赋予了容器。</p><p>因为容器中的权限越高，对系统安全的威胁显然也是越大的。比如说，如果容器中的进程有了CAP_SYS_ADMIN的特权之后，那么这些进程就可以在容器里直接访问磁盘设备，直接可以读取或者修改宿主机上的所有文件了。</p><p>所以，在容器平台上是基本不允许把容器直接设置为"privileged"的，我们需要根据容器中进程需要的最少特权来赋予capabilities。</p><p>我们结合这一讲开始的例子来说说。在开头的例子中，容器里需要使用iptables。因为使用iptables命令，只需要设置CAP_NET_ADMIN这个capability就行。那么我们只要在运行Docker的时候，给这个容器再多加一个NET_ADMIN参数就可以了。</p><pre><code class="language-shell"># docker run --name iptables --cap-add NET_ADMIN -it registry/iptables:v1 bash
[root@cfedf124dcf1 /]# iptables -L
Chain INPUT (policy ACCEPT)
target     prot opt source               destination
 
Chain FORWARD (policy ACCEPT)
target     prot opt source               destination
 
Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination
</code></pre><h2>重点小结</h2><p>这一讲我们主要学习了如何给容器赋予合理的capabilities。</p><p>那么，我们自然需要先来理解什么是Linux capabilities。<strong>其实Linux capabilities就是把Linux root用户原来所有的特权做了细化，可以更加细粒度地给进程赋予不同权限。</strong></p><p>对于Linux中的每一个特权操作都有一个对应的capability，对于一个capability，有的对应一个特权操作，有的可以对应很多个特权操作。</p><p>每个Linux进程有5个capabilities集合参数，其中Effective集合里的capabilities决定了当前进程可以做哪些特权操作，而其他集合参数会和应用程序文件的capabilities集合参数一起来决定新启动程序的capabilities集合参数。</p><p>对于容器的root用户，缺省只赋予了15个capabilities。如果我们发现容器中进程的权限不够，就需要分析它需要的最小capabilities集合，而不是直接赋予容器"privileged"。</p><p>因为"privileged"包含了所有的Linux capabilities, 这样"privileged"就可以轻易获取宿主机上的所有资源，这会对宿主机的安全产生威胁。所以，我们要根据容器中进程需要的最少特权来赋予capabilities。</p><h2>思考题</h2><p>你可以查看一下你的Linux系统里ping程序文件有哪些capabilities，看看有什么办法，能让Linux普通用户没有执行ping的能力。</p><p>欢迎你在留言区和我交流互动。如果学完这一讲让你有所收获，也欢迎转发给你的同事、或者朋友，一起交流探讨容器安全的问题。</p>
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
  <div class="_2_QraFYR_0">getcap $(which ping)<br>setcap -r $(which ping)<br><br>顺便举个之前使用过的例子：普通用户默认没有 tcpdump 抓包权限，可添加 net_raw、net_admin caps：<br>sudo setcap cap_net_raw,cap_net_admin+ep $(which tcpdump)</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-29 08:01:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/23/7e/c0/1c3fd7dd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>朱新威</span>
  </div>
  <div class="_2_QraFYR_0">已经更新20讲了，莫名有点心慌，生怕这么好的专栏结束了🤪</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">编辑回复: 哈哈别慌哦，咱们后续还有专题加餐，等更新的时间里，你还可以复习已有内容哦。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-28 19:03:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/83/09/4a4b0cf2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>老酒馆</span>
  </div>
  <div class="_2_QraFYR_0">getcap &#47;usr&#47;bin&#47;ping 查看ping进程当前cap<br>setcap cap_net_admin,cap_net_raw+p &#47;usr&#47;bin&#47;ping 设置ping进程cap</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-28 17:08:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/48/39/4e95e7b9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>morse</span>
  </div>
  <div class="_2_QraFYR_0">老师, 您好, 我在 Ubuntu20.04 下删除 ping 的 capabilities 后, 切换别的用户后, 还是可以正常使用 ping 的, 我进行了以下操作. <br># getcap &#47;usr&#47;bin&#47;ping # 发现ping 具有cap_net_raw capability<br><br>&#47;usr&#47;bin&#47;ping = cap_net_raw+ep<br># 删除全部 capabilites<br># sudo setcap -r &#47;usr&#47;bin&#47;ping<br># 切换普通用户 sudo su - appuser<br># ping localhost # 可以正常工作<br># capsh --print -- -c &quot;&#47;bin&#47;ping -c 1 localhost&quot; #可以看到还是具有cap_net_raw<br>Current: =<br>Bounding set =cap_chown,cap_dac_override,cap_dac_read_search,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_linux_immutable,cap_net_bind_service,cap_net_broadcast,cap_net_admin,cap_net_raw,cap_ipc_lock,cap_ipc_owner,cap_sys_module,cap_sys_rawio,cap_sys_chroot,cap_sys_ptrace,cap_sys_pacct,cap_sys_admin,cap_sys_boot,cap_sys_nice,cap_sys_resource,cap_sys_time,cap_sys_tty_config,cap_mknod,cap_lease,cap_audit_write,cap_audit_control,cap_setfcap,cap_mac_override,cap_mac_admin,cap_syslog,cap_wake_alarm,cap_block_suspend,cap_audit_read<br>....<br><br># capsh --print # 打印当前用户全部 capabilities, 发现当前用户是具有 cap_net_raw <br><br>给我的感觉, 在 Ubuntu 20.04 中, useradd appuser 创建好的用户, 初始就具有一定的 capabilites, 所以在运行程序的时候, 用户自身的 capabilities+程序的 capabliities 是最终的. 所以就算把文件的 capabilities 删除, 只要用户还具有这个能力, 那么还是可以正常执行的. <br><br>那么我的问题来了, 我没有找到, 如何对一个用户限制这种 capabilities, 即我 useradd 一个 用户, 怎么限制这个用户的 capabilities.<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: @morse<br>你用capsh看到的cap_net_raw 应该是在Binding Cap而不是在Effective Cap里。<br><br>ubuntu20.04 里安装的ping程序本身允许普通用户ping ICMP.<br>https:&#47;&#47;unix.stackexchange.com&#47;questions&#47;617927&#47;why-ping-works-without-capability-and-setuid</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-11 12:53:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83eq30mvo0eATZ3Yfm5POktwic3NJSRkiagtJt1vaxyvCS22PJRm8xrulXqaLJRWQWb6zNI4zL0G2QkCA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>heyhd9475</span>
  </div>
  <div class="_2_QraFYR_0">老师想问一下文件的capabilities是保存在什么地方呢，getcap应该也是从什么地方读取的这些信息吧，是inode,file,dentry？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-11 15:43:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/d0/51/f1c9ae2d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Christopher</span>
  </div>
  <div class="_2_QraFYR_0">selinux是不是实际上就是限制cap权限的操作</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: selinux 是通过对object, 例如进程\文件, 打上label来控制这些object的相互作用。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-02 10:57:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/21/4a/df/c1eb99cf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Tony</span>
  </div>
  <div class="_2_QraFYR_0">老师你好。请问在Linux中（比如centos），在允许普通用户使用docker以后，如何如何限制用户不能读取，宿主机上非该普通用户的文件？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我想你问的问题是关于rootless container的？普通用户可以启动容器，但是用户权限并没有提高，非该用户的文件还是不能读写。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-28 07:52:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2c/90/9b/0013cf16.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>段殷澄</span>
  </div>
  <div class="_2_QraFYR_0">我ubuntu的root用户怎么默认只赋予了15个capabilities</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-07 15:21:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJk3PElN2J96DtyWuIg6xPSs3zRFsIMibOvIn5kuRkESORsRIkDJMUekymI2wiaYiaP0UzibXWEl0aLYw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Bill</span>
  </div>
  <div class="_2_QraFYR_0">Best ever</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-25 14:42:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/97/c5/84491beb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>罗峰</span>
  </div>
  <div class="_2_QraFYR_0">有的gpu容器需要privileged这个是啥原因呢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-30 08:22:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/9f/98/b6f20c10.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>王皓月</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，我想要在容器中使用systemctl，除了启用特权模式还有别的办法吗？看到过大牛在docker run的时候加了&#47;sys&#47;fs&#47;cgroup:&#47;sys&#47;fs&#47;cgroup就可以在容器内使用systemctl，这个和特权模式有什么区别？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-25 13:25:54</div>
  </div>
</div>
</div>
</li>
</ul>