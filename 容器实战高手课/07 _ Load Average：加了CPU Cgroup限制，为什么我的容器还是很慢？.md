<audio title="07 _ Load Average：加了CPU Cgroup限制，为什么我的容器还是很慢？" src="https://static001.geekbang.org/resource/audio/81/7e/81a0355018d9842a599f5e2e0dd3577e.mp3" controls="controls"></audio> 
<p>你好，我是程远。今天我想聊一聊平均负载（Load Average）的话题。</p><p>在上一讲中，我们提到过CPU Cgroup可以限制进程的CPU资源使用，但是CPU Cgroup对容器的资源限制是存在盲点的。</p><p>什么盲点呢？就是无法通过CPU Cgroup来控制Load Average的平均负载。而没有这个限制，就会影响我们系统资源的合理调度，很可能导致我们的系统变得很慢。</p><p>那么今天这一讲，我们要来讲一下为什么加了CPU Cgroup的配置后，即使保证了容器的CPU资源，容器中的进程还是会运行得很慢？</p><h2>问题再现</h2><p>在Linux的系统维护中，我们需要经常查看CPU使用情况，再根据这个情况分析系统整体的运行状态。有时候你可能会发现，明明容器里所有进程的CPU使用率都很低，甚至整个宿主机的CPU使用率都很低，而机器的Load Average里的值却很高，容器里进程运行得也很慢。</p><p>这么说有些抽象，我们一起动手再现一下这个情况，这样你就能更好地理解这个问题了。</p><p>比如说下面的top输出，第三行可以显示当前的CPU使用情况，我们可以看到整个机器的CPU Usage几乎为0，因为"id"显示99.9%，这说明CPU是处于空闲状态的。</p><p>但是请你注意，这里1分钟的"load average"的值却高达9.09，这里的数值9几乎就意味着使用了9个CPU了，这样CPU Usage和Load Average的数值看上去就很矛盾了。</p><!-- [[[read_end]]] --><p><img src="https://static001.geekbang.org/resource/image/50/be/507d6732efd350d47161174bc0a8e9be.png?wh=1858*278" alt=""></p><p>那问题来了，我们在看一个系统里CPU使用情况时，到底是看CPU Usage还是Load Average呢？</p><p>这里就涉及到今天要解决的两大问题：</p><ol>
<li>Load Average到底是什么，CPU Usage和Load Average有什么差别？</li>
<li>如果Load Average值升高，应用的性能下降了，这背后的原因是什么呢？</li>
</ol><p>好了，这一讲我们就带着这两个问题，一起去揭开谜底。</p><h2>什么是Load Average?</h2><p>要回答前面的问题，很显然我们要搞明白这个Linux里的"load average"这个值是什么意思，又是怎样计算的。</p><p>Load Average这个概念，你可能在使用Linux的时候就已经注意到了，无论你是运行uptime, 还是top，都可以看到类似这个输出"load average：2.02, 1.83, 1.20"。那么这一串输出到底是什么意思呢？</p><p>最直接的办法当然是看手册了，如果我们用"Linux manual page"搜索uptime或者top，就会看到对这个"load average"和后面三个数字的解释是"the system load averages for the past 1, 5, and 15 minutes"。</p><p>这个解释就是说，后面的三个数值分别代表过去1分钟，5分钟，15分钟在这个节点上的Load Average，但是看了手册上的解释，我们还是不能理解什么是Load Average。</p><p>这个时候，你如果再去网上找资料，就会发现Load Average是一个很古老的概念了。上个世纪70年代，早期的Unix系统上就已经有了这个Load Average，IETF还有一个<a href="https://tools.ietf.org/html/rfc546">RFC546</a>定义了Load Average，这里定义的Load Average是<strong>一种CPU资源需求的度量。</strong></p><p>举个例子，对于一个单个CPU的系统，如果在1分钟的时间里，处理器上始终有一个进程在运行，同时操作系统的进程可运行队列中始终都有9个进程在等待获取CPU资源。那么对于这1分钟的时间来说，系统的"load average"就是1+9=10，这个定义对绝大部分的Unix系统都适用。</p><p>对于Linux来说，如果只考虑CPU的资源，Load Averag等于单位时间内正在运行的进程加上可运行队列的进程，这个定义也是成立的。通过这个定义和我自己的观察，我给你归纳了下面三点对Load Average的理解。</p><p>第一，不论计算机CPU是空闲还是满负载，Load Average都是Linux进程调度器中<strong>可运行队列（Running Queue）里的一段时间的平均进程数目。</strong></p><p>第二，计算机上的CPU还有空闲的情况下，CPU Usage可以直接反映到"load average"上，什么是CPU还有空闲呢？具体来说就是可运行队列中的进程数目小于CPU个数，这种情况下，单位时间进程CPU Usage相加的平均值应该就是"load average"的值。</p><p>第三，计算机上的CPU满负载的情况下，计算机上的CPU已经是满负载了，同时还有更多的进程在排队需要CPU资源。这时"load average"就不能和CPU Usage等同了。</p><p>比如对于单个CPU的系统，CPU Usage最大只是有100%，也就1个CPU；而"load average"的值可以远远大于1，因为"load average"看的是操作系统中可运行队列中进程的个数。</p><p>这样的解释可能太抽象了，为了方便你理解，我们一起动手验证一下。</p><p>怎么验证呢？我们可以执行个程序来模拟一下,先准备好一个可以消耗任意CPU Usage的<a href="https://github.com/chengyli/training/tree/master/cpu/cgroup_cpu/threads-cpu">程序</a>，在执行这个程序的时候，后面加个数字作为参数，</p><p>比如下面的设置，参数是2，就是说这个进程会创建出两个线程，并且每个线程都跑满100%的CPU，2个线程就是2 * 100% = 200%的CPU Usage，也就是消耗了整整两个CPU的资源。</p><pre><code># ./threads-cpu 2
</code></pre><p>准备好了这个CPU Usage的模拟程序，我们就可以用它来查看CPU Usage和Load Average之间的关系了。</p><p>接下来我们一起跑两个例子，第一个例子是执行2个满负载的线程，第二个例子执行6个满负载的线程，同样都是在一台4个CPU的节点上。</p><p>先来看第一个例子，我们在一台4个CPU的计算机节点上运行刚才这个模拟程序，还是设置参数为2，也就是使用2个CPU Usage。在这个程序运行了几分钟之后，我们运行top来查看一下CPU Usage和Load Average。</p><p>我们可以看到两个threads-cpu各自都占了将近100%的CPU，两个就是200%，2个CPU，对于4个CPU的计算机来说，CPU Usage占了50%，空闲了一半，这个我们也可以从 idle （id）：49.9%得到印证。</p><p>这时候，Load Average里第一项（也就是前1分钟的数值）为1.98，近似于2。这个值和我们一直运行的200%CPU  Usage相对应，也验证了我们之前归纳的第二点——<strong>CPU Usage可以反映到Load Average上。</strong></p><p>因为运行的时间不够，前5分钟，前15分钟的Load Average还没有到2，而且后面我们的例子程序一般都只会运行几分钟，所以这里我们只看前1分钟的Load Average值就行。</p><p>另外，Linux内核中不使用浮点计算，这导致Load Average里的1分钟，5分钟，15分钟的时间值并不精确，但这不影响我们查看Load Average的数值，所以先不用管这个时间的准确性。</p><p><img src="https://static001.geekbang.org/resource/image/11/52/110f67d3b31a62d3d7f27bb4a28aa552.png?wh=1606*420" alt=""><br>
那我们再来跑第二个例子，同样在这个4个CPU的计算机节点上，如果我们执行CPU Usage模拟程序threads-cpu，设置参数为6，让这个进程建出6个线程，这样每个线程都会尽量去抢占CPU，但是计算机总共只有4个CPU，所以这6个线程的CPU Usage加起来只是400%。</p><p>显然这时候4个CPU都被占满了，我们可以看到整个节点的idle（id）也已经是0.0%了。</p><p>但这个时候，我们看看前1分钟的Load Average，数值不是4而是5.93接近6，我们正好模拟了6个高CPU需求的线程。这也告诉我们，Load Average表示的是一段时间里运行队列中需要被调度的进程/线程平均数目。</p><p><img src="https://static001.geekbang.org/resource/image/3c/26/3caf637cb3bc20yy5610b3d0bf59bd26.png?wh=1628*614" alt=""></p><p>讲到这里，我们是不是就可以认定Load Average就代表一段时间里运行队列中需要被调度的进程或者线程平均数目了呢? 或许对其他的Unix系统来说，这个理解已经够了，但是对于Linux系统还不能这么认定。</p><p>为什么这么说呢？故事还要从Linux早期的历史说起，那时开发者Matthias有这么一个发现，比如把快速的磁盘换成了慢速的磁盘，运行同样的负载，系统的性能是下降的，但是Load Average却没有反映出来。</p><p>他发现这是因为Load Average只考虑运行态的进程数目，而没有考虑等待I/O的进程。所以，他认为Load Average如果只是考虑进程运行队列中需要被调度的进程或线程平均数目是不够的，因为对于处于I/O资源等待的进程都是处于TASK_UNINTERRUPTIBLE状态的。</p><p>那他是怎么处理这件事的呢？估计你也猜到了，他给内核加一个patch（补丁），把处于TASK_UNINTERRUPTIBLE状态的进程数目也计入了Load Average中。</p><p>在这里我们又提到了TASK_UNINTERRUPTIBLE状态的进程，在前面的章节中我们介绍过，我再给你强调一下，<strong>TASK_UNINTERRUPTIBLE是Linux进程状态的一种，是进程为等待某个系统资源而进入了睡眠的状态，并且这种睡眠的状态是不能被信号打断的。</strong></p><p>下面就是1993年Matthias的kernel patch，你有兴趣的话，可以读一下。</p><pre><code class="language-shell">From: Matthias Urlichs &lt;urlichs@smurf.sub.org&gt;
Subject: Load average broken ?
Date: Fri, 29 Oct 1993 11:37:23 +0200

The kernel only counts "runnable" processes when computing the load average.
I don't like that; the problem is that processes which are swapping or
waiting on "fast", i.e. noninterruptible, I/O, also consume resources.

It seems somewhat nonintuitive that the load average goes down when you
replace your fast swap disk with a slow swap disk...

Anyway, the following patch seems to make the load average much more
consistent WRT the subjective speed of the system. And, most important, the
load is still zero when nobody is doing anything. ;-)

--- kernel/sched.c.orig Fri Oct 29 10:31:11 1993
+++ kernel/sched.c Fri Oct 29 10:32:51 1993
@@ -414,7 +414,9 @@
unsigned long nr = 0;

    for(p = &amp;LAST_TASK; p &gt; &amp;FIRST_TASK; --p)
-       if (*p &amp;&amp; (*p)-&gt;state == TASK_RUNNING)
+       if (*p &amp;&amp; ((*p)-&gt;state == TASK_RUNNING) ||
+                  (*p)-&gt;state == TASK_UNINTERRUPTIBLE) ||
+                  (*p)-&gt;state == TASK_SWAPPING))
            nr += FIXED_1;
    return nr;
 }
</code></pre><p>那么对于Linux的Load Average来说，除了可运行队列中的进程数目，等待队列中的UNINTERRUPTIBLE进程数目也会增加Load Average。</p><p>为了验证这一点，我们可以模拟一下UNINTERRUPTIBLE的进程，来看看Load Average的变化。</p><p>这里我们做一个<a href="https://github.com/chengyli/training/tree/master/cpu/load_average/uninterruptable/kmod">kernel module</a>，通过一个/proc文件系统给用户程序提供一个读取的接口，只要用户进程读取了这个接口就会进入UNINTERRUPTIBLE。这样我们就可以模拟两个处于UNINTERRUPTIBLE状态的进程，然后查看一下Load Average有没有增加。</p><p>我们发现程序跑了几分钟之后，前1分钟的Load Average差不多从0增加到了2.16，节点上CPU  Usage几乎为0，idle为99.8%。</p><p>可以看到，可运行队列（Running Queue）中的进程数目是0，只有休眠队列（Sleeping Queue）中有两个进程，并且这两个进程显示为D state进程，这个D state进程也就是我们模拟出来的TASK_UNINTERRUPTIBLE状态的进程。</p><p>这个例子证明了Linux将TASK_UNINTERRUPTIBLE状态的进程数目计入了Load Average中，所以即使CPU上不做任何的计算，Load Average仍然会升高。如果TASK_UNINTERRUPTIBLE状态的进程数目有几百几千个，那么Load Average的数值也可以达到几百几千。</p><p><img src="https://static001.geekbang.org/resource/image/e0/a9/e031338191bcec89b7fb02c19af843a9.png?wh=1622*438" alt=""></p><p>好了，到这里我们就可以准确定义Linux系统里的Load Average了，其实也很简单，你只需要记住，平均负载统计了这两种情况的进程：</p><p>第一种是Linux进程调度器中可运行队列（Running Queue）一段时间（1分钟，5分钟，15分钟）的进程平均数。</p><p>第二种是Linux进程调度器中休眠队列（Sleeping Queue）里的一段时间的TASK_UNINTERRUPTIBLE状态下的进程平均数。</p><p>所以，最后的公式就是：<strong>Load Average=可运行队列进程平均数+休眠队列中不可打断的进程平均数</strong></p><p>如果打个比方来说明Load Average的统计原理。你可以想象每个CPU就是一条道路，每个进程都是一辆车，怎么科学统计道路的平均负载呢？就是看单位时间通过的车辆，一条道上的车越多，那么这条道路的负载也就越高。</p><p>此外，Linux计算系统负载的时候，还额外做了个补丁把TASK_UNINTERRUPTIBLE状态的进程也考虑了，这个就像道路中要把红绿灯情况也考虑进去。一旦有了红灯，汽车就要停下来排队，那么即使道路很空，但是红灯多了，汽车也要排队等待，也开不快。</p><h2>现象解释：为什么Load Average会升高？</h2><p>解释了Load Average这个概念，我们再回到这一讲最开始的问题，为什么对容器已经用CPU  Cgroup限制了它的CPU  Usage，容器里的进程还是可以造成整个系统很高的Load Average。</p><p>我们理解了Load Average这个概念之后，就能区分出Load Averge和CPU使用率的区别了。那么这个看似矛盾的问题也就很好回答了，因为<strong>Linux下的Load Averge不仅仅计算了CPU Usage的部分，它还计算了系统中TASK_UNINTERRUPTIBLE状态的进程数目。</strong></p><p>讲到这里为止，我们找到了第一个问题的答案，那么现在我们再看第二个问题：如果Load Average值升高，应用的性能已经下降了，真正的原因是什么？问题就出在TASK_UNINTERRUPTIBLE状态的进程上了。</p><p>怎么验证这个判断呢？这时候我们只要运行 <code>ps aux | grep “ D ”</code> ，就可以看到容器中有多少TASK_UNINTERRUPTIBLE状态（在ps命令中这个状态的进程标示为"D"状态）的进程，为了方便理解，后面我们简称为D状态进程。而正是这些D状态进程引起了Load Average的升高。</p><p>找到了Load Average升高的问题出在D状态进程了，我们想要真正解决问题，还有必要了解D状态进程产生的本质是什么？</p><p>在Linux内核中有数百处调用点，它们会把进程设置为D状态，主要集中在disk I/O 的访问和信号量（Semaphore）锁的访问上，因此D状态的进程在Linux里是很常见的。</p><p><strong>无论是对disk I/O的访问还是对信号量的访问，都是对Linux系统里的资源的一种竞争。</strong>当进程处于D状态时，就说明进程还没获得资源，这会在应用程序的最终性能上体现出来，也就是说用户会发觉应用的性能下降了。</p><p>那么D状态进程导致了性能下降，我们肯定是想方设法去做调试的。但目前D状态进程引起的容器中进程性能下降问题，Cgroups还不能解决，这也就是为什么我们用Cgroups做了配置，即使保证了容器的CPU资源， 容器中的进程还是运行很慢的根本原因。</p><p>这里我们进一步做分析，为什么CPU Cgroups不能解决这个问题呢？就是因为Cgroups更多的是以进程为单位进行隔离，而D状态进程是内核中系统全局资源引入的，所以Cgroups影响不了它。</p><p>所以我们可以做的是，在生产环境中监控容器的宿主机节点里D状态的进程数量，然后对D状态进程数目异常的节点进行分析，比如磁盘硬件出现问题引起D状态进程数目增加，这时就需要更换硬盘。</p><h2>重点总结</h2><p>这一讲我们从CPU  Usage和Load Average差异这个现象讲起，最主要的目的是讲清楚Linux下的Load Average这个概念。</p><p>在其他Unix操作系统里Load Average只考虑CPU部分，Load Average计算的是进程调度器中可运行队列（Running Queue）里的一段时间（1分钟，5分钟，15分钟）的平均进程数目，而Linux在这个基础上，又加上了进程调度器中休眠队列（Sleeping Queue）里的一段时间的TASK_UNINTERRUPTIBLE状态的平均进程数目。</p><p>这里你需要重点掌握Load Average的计算公式，如下图。</p><p><img src="https://static001.geekbang.org/resource/image/e6/4e/e672a6a35248420d623e55d7f7ddf34e.jpeg?wh=3200*1800" alt=""></p><p>因为TASK_UNINTERRUPTIBLE状态的进程同样也会竞争系统资源，所以它会影响到应用程序的性能。我们可以在容器宿主机的节点对D状态进程做监控，定向分析解决。</p><p>最后，我还想强调一下，这一讲中提到的对D状态进程进行监控也很重要，因为这是通用系统性能的监控方法。</p><h2>思考题</h2><p>结合今天的学习，你可以自己动手感受一下Load Average是怎么产生的，请你创建一个容器，在容器中运行一个消耗100%CPU的进程，运行10分钟后，然后查看Load Average的值。</p><p>欢迎在留言区晒出你的经历和疑问。如果有收获，也欢迎你把这篇文章分享给你的朋友，一起学习和讨论。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/23/7e/c0/1c3fd7dd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>朱新威</span>
  </div>
  <div class="_2_QraFYR_0">做个不太恰当的比喻：<br><br>cpu：比做一条高速公路<br><br>进程占用：比做一辆辆汽车<br><br>高速公路上的拥堵情况（load average）= 正在跑的汽车 + 收费站排队等待进入的汽车 + 服务站加油、吃饭的汽车（D进程）；<br><br>把D进程考虑进去是因为，服务站里的汽车随时可能进入高速公路，造成拥堵程度增加</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-12 20:49:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTIYdf5Z0Iicn5mGECKewrGxaY3sPlTEk3PcjImA7as2H7n7L6Krly8dzjwfOQIUYaZwdBBtfCuc0Sg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>garnett</span>
  </div>
  <div class="_2_QraFYR_0">请问老师，引入为 TASK_UNINTERRUPTIBLE 状态的进程的案例，top 输出中为什么wa使用率这一项没有增长？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: TASK_UNINTERRUPTIBLE 状态的进程不一定是做I&#47;O, 比如等待信号量的进程也会进入到这个状态。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-30 11:26:33</div>
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
  <div class="_2_QraFYR_0">推荐 stress 压测工具：stress -c 1 -t 600<br><br>平均负载计算公式（nr_active 表示单位时间内平均活跃进程个数，每个 CPU 对应一个 运行队列 rq，rq-&gt;nr_running、rq-&gt;nr_uninterruptible 分别表示该运行队列上可运行进程、不可中断进程的个数。累积的 nr_active 再进行指数衰减平均得到最终的平均负载）<br><br>&#47;* <br>* The global load average is an exponentially decaying average of nr_running +<br> * nr_uninterruptible.<br> *&#47;<br>nr_active = 0;<br>for_each_possible_cpu(cpu)<br>    nr_active += cpu_of(cpu)-&gt;nr_running + cpu_of(cpu)-&gt;nr_uninterruptible;</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: @莫名，<br>是的，stress是很好的负载模拟工具！<br><br>kernel&#47;sched&#47;loadavg.c 里有完整的load average代码</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-30 08:50:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/64/08/0287f41f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>争光 Alan</span>
  </div>
  <div class="_2_QraFYR_0">感谢分享<br><br>我想问下<br>1.如果出现就D进程，我为什么好的故障方法排查在等待什么吗？<br>2.一直有个疑问，是不是linux的进程数不能太多？太多会有很多的调度时间造成很卡？<br>3.容器目前每个节点官方推荐是110个pod，openshift是250个，能问下你们这边的最佳实践不引起性能下降的前提下节点最大pod是多少个吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: @争光 Alan<br>&gt; 1<br>可以 `cat &#47;proc&#47;&lt;pid&gt;&#47;stack`, 看到进程停在内核中的哪个函数上，结合内核的代码，可以“猜一下”大概是在哪个信号量上。<br><br>&gt; 2<br>进程太多会有问题的。<br><br>&gt; 3<br>我们用的缺省110个pod, 不过对pod&#47;container要做一下max pid的限制， 同时需要监控cpu&#47;memory&#47;disk io&#47; network&#47;D process number&#47;max pid&#47;max fd 等等</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-02 09:29:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/23/86/fe/0fb57311.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>游离的鱼</span>
  </div>
  <div class="_2_QraFYR_0">首先很感谢老师，然后我还是有一些些不太明白，第一个: 处于task_interruptible的进程，虽然它在等信号量和等待io上，但是我理解这个时候其实cpu是空闲的，为什么不把cpu资源让出来，等io完成或者有信号量时再把它放入可运行的队列中去等待调用呢，类似于回调函数那样的思想。第二个: 如果是我的机器长期平均负载过高，是不是一定是D状态的进程或线程引起的。 第三个: 我有四个cpu的机器，现有五个进程，有四个在cpu中运行，其中三个是处于运行状态，另一个是处于task_interruptable状态，也就是D状态，还有一个在排队，那这个时候的负载是不是就是5？如果除了刚刚的D状态的进程其他的进程都运行完了，负载是不是又变成1了。 第四个: 根据老师的定义和公式，平均负载是...的平均进程数，我感觉平均进程数是一个整数，为什么我们看到的平均负载都是带小数的。希望老师帮忙解答一下，帮我解除疑惑。万分感谢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: &gt; 第一个<br>这时候cpu仍然是空闲的，cpu也可以用来调度别的进程，只是需要竞争信号量的几个进程间相互在等待中。<br><br>&gt; 第二个<br>这个就是我在文章中讲的，引起load average增高就是两个原因一个running 队列里的进程，一个是D进程。<br><br>&gt; 第三个<br>如果在较长的一段时间里，都是处于这种状态，那么load average是5。只是处于D状态的进程，其实是不用cpu的，因此其他的四个进程应该都在运行。当其他的四个进程都退出了，只剩D进程，那么等待相当长的一段时间后，load average变成1.<br><br>&gt; 第四个<br>因为load average是过去1分钟&#47;5分钟&#47;15分钟的一个平均值</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-01 09:16:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/d5/d1/42de06de.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Harold</span>
  </div>
  <div class="_2_QraFYR_0">对于 load average 的值还是有些模糊，不考虑 D 进程的情况下，1台8核的机器有 16个 running 的进程，cpu并没有占满100%的情况下，load average取的值是 16*(1 - cpu idle) 还是 16？换句话说，是不是只要 cpu idle 不是0，取的都是 cpu usage 的值。望解答。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: &gt; 换句话说，是不是只要 cpu idle 不是0，取的都是 cpu usage 的值.<br>没有D进程的情况下，是的。<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-12 19:18:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/9f/a1/d75219ee.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>po</span>
  </div>
  <div class="_2_QraFYR_0">有事，好几天没来留言了，我有个确认点：<br>1. 如果一台2个cpu的机器，跑了8个进程，每个进程使用一个cpu的10%，那么load average应该是0.8吧？<br>2. 平常说的物理机CPU，比如2核4线程，这个在Linux里面看是几个cpu呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: @po,<br>&gt;1, 是的<br>&gt;2, 对于处理器里的hyper-threading, 从Linux角度看到的也是1个cpu。2核4线程(hyper-threading), 看到的是4个cpu.<br>你可以 sys&#47;, proc&#47;下看到cpu信息。<br>&#47;sys&#47;devices&#47;system&#47;cpu&#47;online<br>&#47;proc&#47;cpuinfo<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-09 23:37:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/e7/40/e592d386.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Genius</span>
  </div>
  <div class="_2_QraFYR_0">老师好，一个CPU只能同时处理一个进程，为什么还能把CPU分为0.5C的单位呢，这个cpu的单位是怎么理解的呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: CPU是分时处理进程的。有两个进程A，B在一个CPU的机器上运行， 每一秒时间里，A可以运行0.5秒，B可以运行0.5秒，那么A拿到的CPU资源就是0.5 cpu</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-06 13:05:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/f5/97/9a7ee7b3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek4329</span>
  </div>
  <div class="_2_QraFYR_0">感谢老师，受益匪浅！<br><br>有一点不是很懂，想请教下：<br>“这里我们做一个kernel module，通过一个 &#47;proc 文件系统给用户程序提供一个读取的接口，只要用户进程读取了这个接口就会进入 UNINTERRUPTIBLE。”<br><br>老师上面给的kernel module中，我的理解是只调用sleep，然后用户调用这个接口就进入D state，算是模拟DISK IO状态时获取不到资源时的状态吗？老师能不能给个参考，用户进程读是如何取了这个接口呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: @Geek4329<br>内核模块里调用的是 msleep(), 这个函数会把进程的状态设置为TASK_UNINTERRUPTIBLE。 就用它来模拟一下。<br><br>void msleep(unsigned int msecs)<br>{<br>        unsigned long timeout = msecs_to_jiffies(msecs) + 1;<br><br>        while (timeout)<br>                timeout = schedule_timeout_uninterruptible(timeout);<br>}<br><br>signed long __sched schedule_timeout_uninterruptible(signed long timeout)<br>{<br>        __set_current_state(TASK_UNINTERRUPTIBLE);<br>        return schedule_timeout(timeout);<br>}<br><br><br>用户程序读取&#47;proc的接口的例子程序：<br>https:&#47;&#47;github.com&#47;chengyli&#47;training&#47;blob&#47;main&#47;cpu&#47;load_average&#47;uninterruptable&#47;app-test.c</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-30 10:56:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/15/4d/bbfda6b7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>笃定</span>
  </div>
  <div class="_2_QraFYR_0">文中的第二个实验，四个cpu的系统，运行六个进程。理论上六个进程同一时刻不可能都处于R状态吧？一个cpu同一时刻不是只能处理一个进程吗？我的理解top输出应该是四个R状态两个S状态</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: R状态的进程不是单指正在运行的进程，还指在runqueue上可以随时得到调度时间片而可以运行的进程。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-28 15:33:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/b4/f6/e39d5af1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>钱米</span>
  </div>
  <div class="_2_QraFYR_0">请问老师，我的宿主机的memory cgroup很多，处于上升趋势，未下降<br> ~# cat &#47;proc&#47;cgroups<br>#subsys_name	hierarchy	num_cgroups	enabled<br>cpuset	3	30	1<br>cpu	2	101	1<br>cpuacct	2	101	1<br>blkio	6	99	1<br>memory	7	1013	1<br>devices	4	99	1<br>freezer	9	30	1<br>net_cls	10	30	1<br>perf_event	8	30	1<br>net_prio	10	30	1<br>pids	5	100	1<br>这个影响了api对容器的创建和启动，改如何处理呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果被删的memory cgroup还有cache memory, 对应的memroy cgroup 控制块不会随 &#47;sys&#47;fs&#47;cgroup&#47;memory&#47;...&#47;&lt;memory_cgroup&gt; 的删除而马上删掉。<br><br>你可以用 “echo 3 &gt; &#47;proc&#47;sys&#47;vm&#47;drop_caches” 释放cache之后， memory cgroup num_cgroups的数值应该会下降一些。<br><br>&gt; 这个影响了api对容器的创建和启动<br>这个有什么样的影响？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-22 14:26:29</div>
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
  <div class="_2_QraFYR_0">“在 Linux 的系统维护中，我们需要经常查看 CPU 使用情况，再根据这个情况分析系统整体的运行状态。有时候你可能会发现，明明容器里所有进程的 CPU 使用率都很低，甚至整个宿主机的 CPU 使用率都很低，而机器的 Load Average 里的值却很高，容器里进程运行得也很慢。”<br><br>老师，这个问题，我是不是可以这么理解<br><br>    假设系统4CPU，平均负载很高为8，每个任务使用100% CPU可发挥最高性能<br>        1. 如果都是CPU密集型负载，那么CPU使用率不超过50%，应用性能肯定下降<br>        2. 如果全部是IO密集型负载，那么都在竞争IO资源，应用性能肯定下降<br>        3. 假设有小于4个的CPU密集型负载，其他都是IO密集型负载，那么CPU密集型负载的性能应该不受影响。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: @Geek2014, <br>对于1，2，3点，我的理解和你一样。<br><br>不过还是要说一下，I&#47;O高的应用不一定是D进程，而只有D状态才是处于等待资源，才会引起load average增高。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-01 15:58:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/f7/b1/982ea185.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Frank</span>
  </div>
  <div class="_2_QraFYR_0">请问下：  D状态进程如何监控和分析呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-04 11:56:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek4437</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，结合上一节我有个疑问：容器中 top&#47;uptime 中的 load average 是宿主机的还是容器的呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-31 11:21:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/aa/9a/92d2df36.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>tianfeiyu</span>
  </div>
  <div class="_2_QraFYR_0">老师，我想问一下“休眠队列中不可打断的进程平均数”虽然会影响系统的负载，但是对系统的性能影响会非常大吗，它们之前有什么样的关系，因为在线上看到过一些机器确实有D进程出现，甚至负载达到了上千但是系统在正常运行，业务进程并不受影响。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-24 09:30:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/2a/ff/a9d72102.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>BertGeek</span>
  </div>
  <div class="_2_QraFYR_0">原来Load Average 是这么统计的<br>第一种是 Linux 进程调度器中可运行队列（Running Queue）一段时间（1 分钟，5 分钟，15 分钟）的进程平均数。<br>第二种是 Linux 进程调度器中休眠队列（Sleeping Queue）里的一段时间的 TASK_UNINTERRUPTIBLE 状态下的进程平均数。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-29 15:20:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/c5/d5/90ca8efe.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>拉可里啦</span>
  </div>
  <div class="_2_QraFYR_0">老师，我使用top命令后，进程状态是S，但是为什么还占用CPU呢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-31 10:56:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/96/ed/56731acd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>瑞</span>
  </div>
  <div class="_2_QraFYR_0">文首提到的Cgroup无法限制LoadAverage，从而可能导致整个系统性能下降。指的是处于D状态进程的性能下降，而D状态进程对同一资源的竞争越来越多，整体表现出来就是系统性能下降。这样理解对吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-08 08:41:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/9b/08/27ac7ecd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>水蒸蛋</span>
  </div>
  <div class="_2_QraFYR_0">老师cpu平均是活动进程&#47;cpu核数，sleep平均也是sleep&#47;CPUcore数量？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: @水蒸蛋<br>这里的平均数目是指1分钟&#47;5分钟&#47;15分钟队列中的平均数</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-01 18:50:16</div>
  </div>
</div>
</div>
</li>
</ul>