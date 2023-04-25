<audio title="05｜容器CPU（1）：怎么限制容器的CPU使用？" src="https://static001.geekbang.org/resource/audio/7b/d3/7b9ab682cf747b0561cb6b3ef87997d3.mp3" controls="controls"></audio> 
<p>你好，我是程远。从这一讲开始，我们进入容器CPU这个模块。</p><p>我在第一讲中给你讲过，容器在Linux系统中最核心的两个概念是Namespace和Cgroups。我们可以通过Cgroups技术限制资源。这个资源可以分为很多类型，比如CPU，Memory，Storage，Network等等。而计算资源是最基本的一种资源，所有的容器都需要这种资源。</p><p>那么，今天我们就先聊一聊，怎么限制容器的CPU使用？</p><p>我们拿Kubernetes平台做例子，具体来看下面这个pod/container里的spec定义，在CPU资源相关的定义中有两项内容，分别是 <strong>Request CPU</strong> 和 <strong>Limit CPU</strong>。</p><pre><code>apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  containers:
  - name: app
    image: images.my-company.example/app:v4
    env:
    resources:
      requests:
        memory: &quot;64Mi&quot;
        cpu: &quot;1&quot;
      limits:
        memory: &quot;128Mi&quot;
        cpu: &quot;2&quot;
…
</code></pre><p>很多刚刚使用Kubernetes的同学，可能一开始并不理解这两个参数有什么作用。</p><p>这里我先给你说结论，在Pod Spec里的"Request CPU"和"Limit CPU"的值，最后会通过CPU Cgroup的配置，来实现控制容器CPU资源的作用。</p><p>那接下来我会先从进程的CPU使用讲起，然后带你在CPU Cgroup子系统中建立几个控制组，用这个例子为你讲解CPU Cgroup中的三个最重要的参数"cpu.cfs_quota_us""cpu.cfs_period_us""cpu.shares"。</p><!-- [[[read_end]]] --><p>相信理解了这三个参数后，你就会明白我们要怎样限制容器CPU的使用了。</p><h2>如何理解CPU使用和CPU Cgroup？</h2><p>既然我们需要理解CPU  Cgroup，那么就有必要先来看一下Linux里的CPU使用的概念，这是因为CPU Cgroup最大的作用就是限制CPU使用。</p><h3>CPU使用的分类</h3><p>如果你想查看Linux系统的CPU使用的话，会用什么方法呢？最常用的肯定是运行Top了。</p><p>我们对照下图的Top运行界面，在截图第三行，"%Cpu(s)"开头的这一行，你会看到一串数值，也就是"0.0 us, 0.0 sy, 0.0 ni, 99.9 id, 0.0 wa, 0.0 hi, 0.0 si, 0.0 st"，那么这里的每一项值都是什么含义呢？</p><p><img src="https://static001.geekbang.org/resource/image/a6/c2/a67fae56ce2f4e7078d552c58c9f9dc2.png?wh=1696*412" alt=""></p><p>下面这张图里最长的带箭头横轴，我们可以把它看成一个时间轴。同时，它的上半部分代表Linux用户态（User space），下半部分代表内核态（Kernel space）。这里为了方便你理解，我们先假设只有一个CPU吧。</p><p><img src="https://static001.geekbang.org/resource/image/7d/99/7dbd023628f5f4165abc23c1d67aca99.jpeg?wh=3200*1800" alt=""></p><p>我们可以用上面这张图，把这些值挨个解释一下。</p><p>假设一个用户程序开始运行了，那么就对应着第一个"us"框，"us"是"user"的缩写，代表Linux的用户态CPU Usage。普通用户程序代码中，只要不是调用系统调用（System Call），这些代码的指令消耗的CPU就都属于"us"。</p><p>当这个用户程序代码中调用了系统调用，比如说read()去读取一个文件，这时候这个用户进程就会从用户态切换到内核态。</p><p>内核态read()系统调用在读到真正disk上的文件前，就会进行一些文件系统层的操作。那么这些代码指令的消耗就属于"sy"，这里就对应上面图里的第二个框。"sy"是 "system"的缩写，代表内核态CPU使用。</p><p>接下来，这个read()系统调用会向Linux的Block Layer发出一个I/O Request，触发一个真正的磁盘读取操作。</p><p>这时候，这个进程一般会被置为TASK_UNINTERRUPTIBLE。而Linux会把这段时间标示成"wa"，对应图中的第三个框。"wa"是"iowait"的缩写，代表等待I/O的时间，这里的I/O是指Disk I/O。</p><p>紧接着，当磁盘返回数据时，进程在内核态拿到数据，这里仍旧是内核态的CPU使用中的"sy"，也就是图中的第四个框。</p><p>然后，进程再从内核态切换回用户态，在用户态得到文件数据，这里进程又回到用户态的CPU使用，"us"，对应图中第五个框。</p><p>好，这里我们假设一下，这个用户进程在读取数据之后，没事可做就休眠了。并且我们可以进一步假设，这时在这个CPU上也没有其他需要运行的进程了，那么系统就会进入"id"这个步骤，也就是第六个框。"id"是"idle"的缩写，代表系统处于空闲状态。</p><p>如果这时这台机器在网络收到一个网络数据包，网卡就会发出一个中断（interrupt）。相应地，CPU会响应中断，然后进入中断服务程序。</p><p>这时，CPU就会进入"hi"，也就是第七个框。"hi"是"hardware irq"的缩写，代表CPU处理硬中断的开销。由于我们的中断服务处理需要关闭中断，所以这个硬中断的时间不能太长。</p><p>但是，发生中断后的工作是必须要完成的，如果这些工作比较耗时那怎么办呢？Linux中有一个软中断的概念（softirq），它可以完成这些耗时比较长的工作。</p><p>你可以这样理解这个软中断，从网卡收到数据包的大部分工作，都是通过软中断来处理的。那么，CPU就会进入到第八个框，"si"。这里"si"是"softirq"的缩写，代表CPU处理软中断的开销。</p><p>这里你要注意，无论是"hi"还是"si"，它们的CPU时间都不会计入进程的CPU时间。<strong>这是因为本身它们在处理的时候就不属于任何一个进程。</strong></p><p>好了，通过这个场景假设，我们介绍了大部分的Linux CPU使用。</p><p>不过，我们还剩两个类型的CPU使用没讲到，我想给你做个补充，一次性带你做个全面了解。这样以后你解决相关问题时，就不会再犹豫，这些值到底影不影响CPU Cgroup中的限制了。下面我给你具体讲一下。</p><p>一个是"ni"，是"nice"的缩写，这里表示如果进程的nice值是正值（1-19），代表优先级比较低的进程运行时所占用的CPU。</p><p>另外一个是"st"，"st"是"steal"的缩写，是在虚拟机里用的一个CPU使用类型，表示有多少时间是被同一个宿主机上的其他虚拟机抢走的。</p><p>综合前面的内容，我再用表格为你总结一下：<br>
<img src="https://static001.geekbang.org/resource/image/a4/a3/a4f537187a16e872ebcc605d972672a3.jpeg?wh=3200*1800" alt=""></p><h3>CPU Cgroup</h3><p>在第一讲中，我们提到过Cgroups是对指定进程做计算机资源限制的，CPU  Cgroup是Cgroups其中的一个Cgroups子系统，它是用来限制进程的CPU使用的。</p><p>对于进程的CPU使用, 通过前面的Linux CPU使用分类的介绍，我们知道它只包含两部分: 一个是用户态，这里的用户态包含了us和ni；还有一部分是内核态，也就是sy。</p><p>至于wa、hi、si，这些I/O或者中断相关的CPU使用，CPU Cgroup不会去做限制，那么接下来我们就来看看CPU  Cgoup是怎么工作的？</p><p>每个Cgroups子系统都是通过一个虚拟文件系统挂载点的方式，挂到一个缺省的目录下，CPU  Cgroup 一般在Linux 发行版里会放在 <code>/sys/fs/cgroup/cpu</code> 这个目录下。</p><p>在这个子系统的目录下，每个控制组（Control Group） 都是一个子目录，各个控制组之间的关系就是一个树状的层级关系（hierarchy）。</p><p>比如说，我们在子系统的最顶层开始建立两个控制组（也就是建立两个目录）group1 和 group2，然后再在group2的下面再建立两个控制组group3和group4。</p><p>这样操作以后，我们就建立了一个树状的控制组层级，你可以参考下面的示意图。<br>
<img src="https://static001.geekbang.org/resource/image/8b/54/8b86bc86706b0bbfe8fe157ee21b6454.jpeg?wh=3200*1800" alt=""></p><p>那么我们的每个控制组里，都有哪些CPU  Cgroup相关的控制信息呢？这里我们需要看一下每个控制组目录中的内容：</p><pre><code class="language-shell"> # pwd
/sys/fs/cgroup/cpu
# mkdir group1 group2
# cd group2
# mkdir group3 group4
# cd group3
# ls cpu.*
cpu.cfs_period_us  cpu.cfs_quota_us  cpu.rt_period_us  cpu.rt_runtime_us  cpu.shares  cpu.stat 
</code></pre><p>考虑到在云平台里呢，大部分程序都不是实时调度的进程，而是普通调度（SCHED_NORMAL）类型进程，那什么是普通调度类型呢？</p><p>因为普通调度的算法在Linux中目前是CFS （Completely Fair Scheduler，即完全公平调度器）。为了方便你理解，我们就直接来看CPU Cgroup和CFS相关的参数，一共有三个。</p><p>第一个参数是 <strong>cpu.cfs_period_us</strong>，它是CFS算法的一个调度周期，一般它的值是100000，以microseconds为单位，也就100ms。</p><p>第二个参数是 <strong>cpu.cfs_quota_us</strong>，它“表示CFS算法中，在一个调度周期里这个控制组被允许的运行时间，比如这个值为50000时，就是50ms。</p><p>如果用这个值去除以调度周期（也就是cpu.cfs_period_us），50ms/100ms = 0.5，这样这个控制组被允许使用的CPU最大配额就是0.5个CPU。</p><p>从这里能够看出，cpu.cfs_quota_us是一个绝对值。如果这个值是200000，也就是200ms，那么它除以period，也就是200ms/100ms=2。</p><p>你看，结果超过了1个CPU，这就意味着这时控制组需要2个CPU的资源配额。</p><p>我们再来看看第三个参数， <strong>cpu.shares</strong>。这个值是CPU  Cgroup对于控制组之间的CPU分配比例，它的缺省值是1024。</p><p>假设我们前面创建的group3中的cpu.shares是1024，而group4中的cpu.shares是3072，那么group3:group4=1:3。</p><p>这个比例是什么意思呢？我还是举个具体的例子来说明吧。</p><p>在一台4个CPU的机器上，当group3和group4都需要4个CPU的时候，它们实际分配到的CPU分别是这样的：group3是1个，group4是3个。</p><p>我们刚才讲了CPU Cgroup里的三个关键参数，接下来我们就通过几个例子来进一步理解一下，代码你可以在<a href="https://github.com/chengyli/training/tree/master/cpu/cgroup_cpu">这里</a>找到。</p><p>第一个例子，我们启动一个消耗2个CPU（200%）的程序threads-cpu，然后把这个程序的pid加入到group3的控制组里：</p><pre><code>./threads-cpu/threads-cpu 2 &amp;
echo $! &gt; /sys/fs/cgroup/cpu/group2/group3/cgroup.procs 
</code></pre><p>在我们没有修改cpu.cfs_quota_us前，用top命令可以看到threads-cpu这个进程的CPU  使用是199%，近似2个CPU。</p><p><img src="https://static001.geekbang.org/resource/image/1e/b8/1e95db3f15fc4cf1573f8ebe22db38b8.png?wh=1640*142" alt=""></p><p>然后，我们更新这个控制组里的cpu.cfs_quota_us，把它设置为150000（150ms）。把这个值除以cpu.cfs_period_us，计算过程是150ms/100ms=1.5, 也就是1.5个CPU，同时我们也把cpu.shares设置为1024。</p><pre><code>echo 150000 &gt; /sys/fs/cgroup/cpu/group2/group3/cpu.cfs_quota_us
echo 1024 &gt; /sys/fs/cgroup/cpu/group2/group3/cpu.shares
</code></pre><p>这时候我们再运行top，就会发现threads-cpu进程的CPU使用减小到了150%。这是因为我们设置的cpu.cfs_quota_us起了作用，限制了进程CPU的绝对值。</p><p>但这时候cpu.shares的作用还没有发挥出来，因为cpu.shares是几个控制组之间的CPU分配比例，而且一定要到整个节点中所有的CPU都跑满的时候，它才能发挥作用。</p><p><img src="https://static001.geekbang.org/resource/image/3c/7e/3c153bba9d7668c22048602d730d627e.png?wh=1670*142" alt=""></p><p>好，下面我们再来运行第二个例子来理解cpu.shares。我们先把第一个例子里的程序启动，同时按前面的内容，一步步设置好group3里cpu.cfs_quota_us 和cpu.shares。</p><p>设置完成后，我们再启动第二个程序，并且设置好group4里的cpu.cfs_quota_us 和 cpu.shares。</p><p>group3：</p><pre><code>./threads-cpu/threads-cpu 2 &amp;  # 启动一个消耗2个CPU的程序
echo $! &gt; /sys/fs/cgroup/cpu/group2/group3/cgroup.procs #把程序的pid加入到控制组
echo 150000 &gt; /sys/fs/cgroup/cpu/group2/group3/cpu.cfs_quota_us #限制CPU为1.5CPU
echo 1024 &gt; /sys/fs/cgroup/cpu/group2/group3/cpu.shares 

</code></pre><p>group4：</p><pre><code>./threads-cpu/threads-cpu 4 &amp;  # 启动一个消耗4个CPU的程序
echo $! &gt; /sys/fs/cgroup/cpu/group2/group4/cgroup.procs #把程序的pid加入到控制组
echo 350000 &gt; /sys/fs/cgroup/cpu/group2/group4/cpu.cfs_quota_us  #限制CPU为3.5CPU
echo 3072 &gt; /sys/fs/cgroup/cpu/group2/group3/cpu.shares # shares 比例 group4: group3 = 3:1
</code></pre><p>好了，现在我们的节点上总共有4个CPU，而group3的程序需要消耗2个CPU，group4里的程序要消耗4个CPU。</p><p>即使cpu.cfs_quota_us已经限制了进程CPU使用的绝对值，group3的限制是1.5CPU，group4是3.5CPU，1.5+3.5=5，这个结果还是超过了节点上的4个CPU。</p><p>好了，说到这里，我们发现在这种情况下，cpu.shares终于开始起作用了。</p><p>在这里shares比例是group4:group3=3:1，在总共4个CPU的节点上，按照比例，group4里的进程应该分配到3个CPU，而group3里的进程会分配到1个CPU。</p><p>我们用top可以看一下，结果和我们预期的一样。</p><p><img src="https://static001.geekbang.org/resource/image/84/a3/8424b7fb4c84679412f75774060fcca3.png?wh=1630*152" alt=""></p><p>好了，我们对CPU Cgroup的参数做一个梳理。</p><p>第一点，cpu.cfs_quota_us和cpu.cfs_period_us这两个值决定了<strong>每个控制组中所有进程的可使用CPU资源的最大值。</strong></p><p>第二点，cpu.shares这个值决定了<strong>CPU Cgroup子系统下控制组可用CPU的相对比例</strong>，不过只有当系统上CPU完全被占满的时候，这个比例才会在各个控制组间起作用。</p><h2>现象解释</h2><p>在解释了Linux CPU Usage和CPU Cgroup这两个基本概念之后，我们再回到我们最初的问题 “怎么限制容器的CPU使用”。有了基础知识的铺垫，这个问题就比较好解释了。</p><p>首先，Kubernetes会为每个容器都在CPUCgroup的子系统中建立一个控制组，然后把容器中进程写入到这个控制组里。</p><p>这时候"Limit CPU"就需要为容器设置可用CPU的上限。结合前面我们讲的几个参数么，我们就能知道容器的CPU上限具体如何计算了。</p><p>容器CPU的上限由cpu.cfs_quota_us除以cpu.cfs_period_us得出的值来决定的。而且，在操作系统里，cpu.cfs_period_us的值一般是个固定值，Kubernetes不会去修改它，所以我们就是只修改cpu.cfs_quota_us。</p><p>而"Request CPU"就是无论其他容器申请多少CPU资源，即使运行时整个节点的CPU都被占满的情况下，我的这个容器还是可以保证获得需要的CPU数目，那么这个设置具体要怎么实现呢？</p><p>显然我们需要设置cpu.shares这个参数：<strong>在CPU Cgroup中cpu.shares == 1024表示1个CPU的比例，那么Request CPU的值就是n，给cpu.shares的赋值对应就是n*1024。</strong></p><h2>重点总结</h2><p>首先，我带你了解了Linux下CPU Usage的种类.</p><p>这里你要注意的是<strong>每个进程的CPU Usage只包含用户态（us或ni）和内核态（sy）两部分，其他的系统CPU开销并不包含在进程的CPU使用中，而CPU Cgroup只是对进程的CPU使用做了限制。</strong></p><p>其实这一讲我们开篇的问题“怎么限制容器的CPU使用”，这个问题背后隐藏了另一个问题，也就是容器是如何设置它的CPU Cgroup中参数值的？想解决这个问题，就要先知道CPU Cgroup都有哪些参数。</p><p>所以，我详细给你介绍了CPU Cgroup中的主要参数，包括这三个：<strong>cpu.cfs_quota_us，cpu.cfs_period_us 还有cpu.shares。</strong></p><p>其中，cpu.cfs_quota_us（一个调度周期里这个控制组被允许的运行时间）除以cpu.cfs_period_us（用于设置调度周期）得到的这个值决定了CPU  Cgroup每个控制组中CPU使用的上限值。</p><p>你还需要掌握一个cpu.shares参数，正是这个值决定了CPU  Cgroup子系统下控制组可用CPU的相对比例，当系统上CPU完全被占满的时候，这个比例才会在各个控制组间起效。</p><p>最后，我们明白了CPU Cgroup关键参数是什么含义后，Kubernetes中"Limit CPU"和 "Request CPU"也就很好解释了:</p><p><strong> Limit CPU就是容器所在Cgroup控制组中的CPU上限值，Request CPU的值就是控制组中的cpu.shares的值。</strong></p><h2>思考题</h2><p>我们还是按照文档中定义的控制组目录层次结构图，然后按序执行这几个脚本：</p><ul>
<li><a href="https://github.com/chengyli/training/blob/main/cpu/cgroup_cpu/create_groups.sh">create_groups.sh</a></li>
<li><a href="https://github.com/chengyli/training/blob/main/cpu/cgroup_cpu/update_group1.sh">update_group1.sh</a></li>
<li><a href="https://github.com/chengyli/training/blob/main/cpu/cgroup_cpu/update_group4.sh">update_group4.sh</a></li>
<li><a href="https://github.com/chengyli/training/blob/main/cpu/cgroup_cpu/update_group3.sh">update_group3.sh</a></li>
</ul><p>那么，在一个4个CPU的节点上，group1/group3/group4里的进程，分别会被分配到多少CPU呢?</p><p>欢迎留言和我分享你的思考和疑问。如果你有所收获，也欢迎分享给朋友，一起学习和交流。</p>
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
  <div class="_2_QraFYR_0">CPU 使用率分解图画的挺有创意👍</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-25 08:21:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/e8/c2/77a413a7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Action</span>
  </div>
  <div class="_2_QraFYR_0">为什么说“云平台里呢，大部分程序都不是实时调度的进程，而是普通调度（SCHED_NORMAL）类型进程”？这块不是很明白</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 进程如果设置为SCHED_FIFO 或者SCHED_RR实时调度类型，那么只要进程任务不结束，就不会把cpu资源让给SCHED_NORMAL进程。这种实时的进程，在实时性要比较高的嵌入式系统中会用到，但是云平台中提供互联网服务的应用中不太会去用实时调度。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-25 15:21:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/64/be/546665e3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>心随缘</span>
  </div>
  <div class="_2_QraFYR_0">老师的时间轴讲解TOP非常棒！CPU 使用分解的很到位~赞!</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-10 11:18:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/ce/b2/1f914527.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>海盗船长</span>
  </div>
  <div class="_2_QraFYR_0">老师，不太明白  “Request CPU就是无论其他容器申请多少 CPU 资源，即使运行时整个节点的 CPU 都被占满的情况下，我的这个容器还是可以保证获得需要的 CPU 数目”，这句话改怎么理解呢？当节点cpu都被占满的情况下，我的这个容器会去抢占吗？ 另外cpu.shares是个权重，如何去保证Request CPU的数量？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 首先这个容器已经在节点上运行了，并且其他的容器也都配置了cpu.shares。<br>比如shares比例， group4: group3=3:1， 在一个4个CPU的节点上，我们先不用考虑limit, 对于group4,如果没有group3的程序运行，那么group4里的4个线程运行的时候可以占用4个CPU。当group3里的2个线程也运行起来了，即使2个线程最大可以消耗2个CPU,但是由于有了shares比例分配，那么group4里的线程仍然可以保证有3个CPU, 而分配给group3的只有1个CPU。<br><br>cpu.shares是个相对值，但是在Linux节点上一般的约定是以值1024为1个CPU的比例，当所有的配置都遵守这个约定的时候，那么给值N*1024, 就表示N个CPU的数量了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-25 10:05:23</div>
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
  <div class="_2_QraFYR_0">group1:group2是1比1，由于group1 limit是3.5，那group1分到的只能是两个核，剩余的2个核给group3和group4，group4:group3是3比1，那么得出group4与group3各分配的核就是1.5核与0.5核</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-26 20:58:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/9c/aa/6f780187.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>言希</span>
  </div>
  <div class="_2_QraFYR_0">请问老师，我在环境上遇到 cpu.cfs_quota_us 取值为 -1 的，这种是不是代表的不限制CPU的使用 ？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对的，不限制。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-29 10:08:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/e8/c2/77a413a7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Action</span>
  </div>
  <div class="_2_QraFYR_0">老师，假如我有10个节点，每个节点的cpu核心数是40，只是调度pod，那么limit.cpu 可以设置为400吧？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 一个pod只能在一个节点上运行，如果一个节点上的cpu是40， 那么limit.cpu最大只能是40</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-25 15:01:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/ce/15/51187703.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>瓜蛋</span>
  </div>
  <div class="_2_QraFYR_0">请问下关于wa和hi&#47;si的问题：<br>1. 例子中，wa是等待磁盘IO的状态，那等待网络IO时，是不是wa呢？<br>2. 例子中，hi&#47;si是收到网卡中断，那收到磁盘中断时，是不是也是hi&#47;si？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 网络I&#47;O 没有wa状态部分。<br>hi&#47;si 对于磁盘都是有的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-29 22:23:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTIz9dKN1C8rKQoaVtmEGdzObhlia6zAfTsPYOm4ibz39VjTbu7Aia1LyeedHR26b6nxUtcCufpichcYgw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>上邪忘川</span>
  </div>
  <div class="_2_QraFYR_0">分别为2,1.5,0.5<br>group1和group2分配的cpu配额已经超过4个总cpu，那么就会按照cpu.shares的比例去分配cpu。其中，group1:group2=1:1，group3:group4=1:3，group2=group3+group4，得出group1:group3:group4=4:1:3</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-25 21:48:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/c5/a7/e32dcfe7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>谦寻</span>
  </div>
  <div class="_2_QraFYR_0">老师，您好，如果一个容器里面有多个进程，这个限制是针对所有进程，还是只是pid是1的进程？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 一般对于容器，是一个cgroup控制组里限制容器中的所有进程。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-07 11:05:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/f5/9d/104bb8ea.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek2014</span>
  </div>
  <div class="_2_QraFYR_0">尝试胡乱分析下：<br>group1 的shares为1024，quota 3.5，尝试使用4，<br>group2的shares默认为1024，quota设置为-1，不受限制，也即是，如果CPU上只有group2的话，那么group2可以使用完所有的CPU（实际上根据group3和group4，group2最多也就能用到1.5+3.5 core）<br><br>故而，group1和group2各分配到2<br><br>把group2分到的2CPU，看作总量，再次分析group3和group4<br>group3和group3尝试使用的总量超过2，所以按照shares比例分配，group3使用1&#47;(1+3) * 2 = 0.5，group4使用3&#47;(1+3) * 2 = 1.5</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-26 10:51:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/b2/91/714c0f07.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>zero</span>
  </div>
  <div class="_2_QraFYR_0">这节真心不错</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-16 11:12:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>垂死挣扎的咸鱼</span>
  </div>
  <div class="_2_QraFYR_0">老师，这边想请教一个问题，两个容器a,b进程分别设置了cpuset以及cpu.share，其中两个进程绑定了相同的16核，a设置较大的cpu.share值而b设置较小的cpu.share值，比例为4比1，同时将这两个进程压测到80%左右的cpu使用率，发现他俩各占用了8核左右的cpu就好像cpu.share值没有生效，可以请教一下老师可能是什么原因造成cpu.share没有生效</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-28 15:45:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/dc/21/34c72e67.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>cyz</span>
  </div>
  <div class="_2_QraFYR_0">评论解答依然精彩</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-06 22:44:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/ac/33/110437cc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>不二</span>
  </div>
  <div class="_2_QraFYR_0">请问老师，cpu.rt_runtime_us一般是不是用不上，Linux系统中哪些程序会被配置为实时调度程序呢？如果没有实时调度程序这个参数也就没有了存在的必要吧</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 一些嵌入式实时的程序需要实时调度的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-14 14:36:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/28/8d/c6/9afdffaa.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Andylin</span>
  </div>
  <div class="_2_QraFYR_0">在group4：中 echo 3072 &gt; &#47;sys&#47;fs&#47;cgroup&#47;cpu&#47;group2&#47;group3&#47;cpu.shares   应该调整为echo 3072 &gt; &#47;sys&#47;fs&#47;cgroup&#47;cpu&#47;group2&#47;group4&#47;cpu.shares 才是对的</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-17 17:56:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2d/d5/e5/101e13b5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>HF</span>
  </div>
  <div class="_2_QraFYR_0">echo 3072 &gt; &#47;sys&#47;fs&#47;cgroup&#47;cpu&#47;group2&#47;group3&#47;cpu.shares # shares 比例 group4: group3 = 3:1 这个地方是否有误</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-09 14:56:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/54/7b/780c04ff.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>史努比</span>
  </div>
  <div class="_2_QraFYR_0">我尝试回答一下。不对之处，感谢指正：<br><br>1.当执行完create_groups.sh后，创建出了四个CPU控制组，分别是group1、group2、group3和group4，其中group3和group4属于group2层级下；<br><br>2.执行完update_group1.sh后，group1控制组设置的CPU限制是3.5核(cpu.cfs_quota_us为350000)，且节点无其他控制器组进程运行，group1控制组的进程最多能够使用到CPU配额上限，即3.5个CPU；<br><br>3.执行完update_group4.sh后，两个控制组的进程将占满节点CPU，此时节点cpu资源会被group1下和group4下的进程完全瓜分。由于group1和group4两个控制组的cpu.shares分别是1024和3072，也就是1:3，所以两个控制组下的进程能够分得的CPU资源是1核和3核；<br><br>4.最后是update_group3.sh，三个控制组(group1、group4、group3)的进程占满节点CPU，这里前后两个阶段各控制组下进程的cpu分配无变化：首先是刚开始为group3设置cpu.cfs_quota_us为-1(不限制cpu上限)，且未设置cpu.shares（默认就是1024）。此时group1、group4和group3控制组下的进程最多能够使用节点cpu资源比例由各自cpu.shares在总的cpu.shares值的比例决定，因此分别是1:3:1，最后分别能使用0.8、2.4和0.8个CPU；而后将group3的cpu.cfs_quota_us设置为150000，cpu.shares为1024。由于节点cpu占满，各控制值的cpu.shares比例未发生改变，且group3的cpu限制是大于0.8个cpu（1.5个cpu），因此group1、group4和group3控制组下进程仍然是分得0.8、2.4和0.8个CPU。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-20 21:23:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2c/45/44/8df79d3c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>事已至此开始撤退</span>
  </div>
  <div class="_2_QraFYR_0">感觉上面讲的和下面的没有啥联系呀，直接看下面的也可以设置好CPU的cgroup，即使完整看完了上面，也和它实现的底层逻辑没有什么关联，老师可能确实自己懂很多，但是没有很好的展现出来</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-19 14:32:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/8d/6d/8c0a487b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>册书一幕</span>
  </div>
  <div class="_2_QraFYR_0">request只有满的时候才有用，是否意味着CPU不满的时候，request可以任意定义，容器都能使用到超出request定义的部分</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-04 17:26:08</div>
  </div>
</div>
</div>
</li>
</ul>