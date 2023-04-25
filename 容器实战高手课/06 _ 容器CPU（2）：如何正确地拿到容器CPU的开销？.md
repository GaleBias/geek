<audio title="06 _ 容器CPU（2）：如何正确地拿到容器CPU的开销？" src="https://static001.geekbang.org/resource/audio/6a/7e/6a544263ae275a2e6197ef02be61007e.mp3" controls="controls"></audio> 
<p>你好，我是程远。今天我们聊一聊，如何正确地拿到容器CPU的开销。</p><p>为啥要解决这个问题呢，还是来源于实际工作中的需要。</p><p>无论是容器的所有者还是容器平台的管理者，我们想要精准地对运行着众多容器的云平台做监控，快速排查例如应用的处理能力下降，节点负载过高等问题，就绕不开容器CPU开销。<strong>因为CPU开销的异常，往往是程序异常最明显的一个指标。</strong></p><p>在一台物理机器或者虚拟机里，如果你想得到这个节点的CPU使用率，最常用的命令就是top了吧？top一下子就能看到整个节点当前的CPU使用情况。</p><p>那么在容器里，top命令也可以做到这点吗？想要知道答案，我们还是得实际动手试一试。</p><h2>问题重现</h2><p>实际上，你在使用容器的时候，如果运行top命令来查看当前容器总共使用了多少CPU，你肯定马上就会失望了。</p><p>这是因为我们在容器中运行top命令，虽然可以看到容器中每个进程的CPU使用率，但是top中"%Cpu(s)"那一行中显示的数值，并不是这个容器的CPU整体使用率，而是容器宿主机的CPU使用率。</p><p>就像下面的这个例子，我们在一个12个CPU的宿主机上，启动一个容器，然后在容器里运行top命令。</p><p>这时我们可以看到，容器里有两个进程threads-cpu，总共消耗了200%的CPU（2 CPU  Usage），而"%Cpu(s)"那一行的"us cpu"是58.5%。对于12CPU的系统来说，12 * 58.5%=7.02，也就是说这里显示总共消耗了7个CPU，远远大于容器中2个CPU的消耗。</p><!-- [[[read_end]]] --><p><img src="https://static001.geekbang.org/resource/image/by/fb/byy517a2ee2e041c5ce0ae4a30753cfb.png?wh=1686*568" alt="" title="运行top的示意图"></p><p>这个例子说明，top这个工具虽然在物理机或者虚拟机上看得到系统CPU开销，但是如果是放在容器环境下，运行top就无法得到容器中总的CPU使用率。那么，我们还有什么其他的办法吗？</p><h2>进程CPU使用率和系统CPU使用率</h2><p>通过问题重现，我们发现top工具主要显示了宿主机系统整体的CPU使用率，以及单个进程的CPU使用率。既然没有现成的工具可以得到容器CPU开销，那我们需要自己开发一个工具来解决问题了。</p><p>其实我们自己推导，也没有那么难。我认为，最有效的思路还是从原理上去理解问题。</p><p>所以，在解决怎样得到单个容器整体的CPU使用率这个问题之前，我们先来学习一下，在Linux中到底是如何计算单个进程的CPU使用率，还有整个系统的CPU使用率的。</p><h3>进程CPU使用率</h3><p>Linux中每个进程的CPU使用率，我们都可以用top命令查看。</p><p>对照我们前面的那张示意图，我们可以发现，每个进程在top命令输出中都有对应的一行，然后“%CPU”的那一列就是这个进程的实时CPU使用率了。</p><p>比如说，100%就表示这个进程在这个瞬时使用了1个CPU，200%就是使用了2个CPU。那么这个百分比的数值是怎么得到呢？</p><p>最直接的方法，就是从源头开始寻找答案。因为是top命令的输出，我们可以去看一下top命令的<a href="https://gitlab.com/procps-ng/procps">源代码</a>。在代码中你会看到对于每个进程，top都会从proc文件系统中每个进程对应的stat文件中读取2个数值。我们先来看这个文件，再来解读文件中具体的两个数值。</p><p>这个stat文件就是 <code>/proc/[pid]/stat</code> ， <code>[pid]</code> 就是替换成具体一个进程的PID值。比如PID值为1的进程，这个文件就是 <code>/proc/1/stat</code> ，那么这个 <code>/proc/[pid]/stat</code> 文件里有什么信息呢？</p><p>其实这个stat文件实时输出了进程的状态信息，比如进程的运行态（Running还是 Sleeping）、父进程PID、进程优先级、进程使用的内存等等总共50多项。</p><p>完整的stat文件内容和格式在proc文件系统的 <a href="https://man7.org/linux/man-pages/man5/proc.5.html">Linux programmer’s manual</a> 里定义了。在这里，我们只需要重点关注这两项数值，stat文件中的第14项utime和第15项stime。</p><p><img src="https://static001.geekbang.org/resource/image/53/b1/53c409d607942e8189f73636aafe3cb1.png?wh=790*333" alt=""></p><p>那么这两项数值utime和stime是什么含义呢？utime是表示进程的用户态部分在Linux调度中获得CPU的ticks，stime是表示进程的内核态部分在Linux调度中获得CPU的ticks。</p><p>看到这个解释，你可能又冒出一个新问题，疑惑ticks是什么?这个ticks就是Linux操作系统中的一个时间单位，你可以理解成类似秒，毫秒的概念。</p><p>在Linux中有个自己的时钟，它会周期性地产生中断。每次中断都会触发Linux内核去做一次进程调度，而这一次中断就是一个tick。因为是周期性的中断，比如1秒钟100次中断，那么一个tick作为一个时间单位看的话，也就是1/100秒。</p><p>我给你举个例子说明，假如进程的utime是130ticks，就相当于130 * 1/100=1.3秒，也就是进程从启动开始在用户态总共运行了1.3秒钟。</p><p>这里需要你注意，utime和stime都是一个累计值，也就是说从进程启动开始，这两个值就是一直在累积增长的。</p><p>那么我们怎么计算，才能知道某一进程在用户态和内核态中，分别获得了多少CPU的ticks呢？</p><p>首先，我们可以假设这个瞬时是1秒钟，这1秒是T1时刻到T2时刻之间的，那么这样我们就能获得 T1 时刻的utime_1 和stime_1，同时获得T2时刻的utime_2 和 stime_2。</p><p>在这1秒的瞬时，进程用户态获得的CPU ticks就是 (utime_2 – utime_1), 进程内核态获得的CPU ticks就是 (stime_2 – stime_1)。</p><p>那么我们可以推导出，进程CPU总的开销就是用户态加上内核态，也就是在1秒瞬时进程总的CPU ticks等于 (utime_2 – utime_1) + (stime_2 – stime_1)。</p><p>好了，现在我们得到了进程以ticks为单位的CPU开销，接下来还要做个转化。我们怎样才能把这个值转化成我们熟悉的百分比值呢？其实也不难，我们还是可以去top的<a href="https://gitlab.com/procps-ng/procps">源代码</a>里得到这个百分比的计算公式。</p><p>简单总结一下，这个公式是这样的：</p><p><strong>进程的CPU使用率=((utime_2 – utime_1) + (stime_2 – stime_1)) * 100.0 / (HZ * et * 1 )</strong></p><p>接下来，我再给你讲一下，这个公式里每一个部分的含义。</p><p>首先， ((utime_2 – utime_1) + (stime_2 – stime_1))是瞬时进程总的CPU ticks。这个我们已经在前面解释过了。</p><p>其次，我们来看100.0，这里乘以100.0的目的是产生百分比数值。</p><p>最后，我再讲一下  <strong>(HZ * et * 1)</strong>。这是被除数这里的三个参数，我给你详细解释一下。</p><p>第一个HZ是什么意思呢？前面我们介绍ticks里说了，ticks是按照固定频率发生的，在我们的Linux系统里1秒钟是100次，那么HZ就是1秒钟里ticks的次数，这里值是100。</p><p>第二个参数et是我们刚才说的那个“瞬时”的时间，也就是得到utime_1和utime_2这两个值的时间间隔。</p><p>第三个“1”, 就更容易理解了，就是1个CPU。那么这三个值相乘，你是不是也知道了它的意思呢？就是在这“瞬时”的时间（et）里，1个CPU所包含的ticks数目。</p><p>解释了这些参数，我们可以把这个公式简化一下，就是下面这样：</p><p>进程的CPU使用率=（进程的ticks/单个CPU总ticks）*100.0</p><p>知道了这个公式，就需要上手来验证一下这个方法对不对，怎么验证呢？我们可以启动一个消耗CPU的小程序，然后读取一下进程对应的/proc/[pid]/stat中的utime和stime，然后用这个方法来计算一下进程使用率这个百分比值，并且和top的输出对比一下，看看是否一致。</p><p>先启动一个消耗200%的小程序，它的PID是10021，CPU使用率是200%。<br>
<img src="https://static001.geekbang.org/resource/image/yy/f9/yy5f601a16cd9c5f8b01157daea0d4f9.png?wh=1644*364" alt=""></p><p>然后，我们查看这个进程对应的stat文件/proc/10021/stat，间隔1秒钟输出第二次，因为stat文件内容很多，我们知道utime和stime第14和15项，所以我们这里只截取了前15项的输出。这里可以看到，utime_1 = 399，stime_1=0，utime_2=600，stime_2=0。</p><p><img src="https://static001.geekbang.org/resource/image/29/b0/29609d255bfd22eab6697f6ccd4923b0.png?wh=1556*102" alt=""></p><p>根据前面的公式，我们计算一下进程threads-cpu的CPU使用率。套用前面的公式，计算的过程是：</p><p>((600 – 399) + (0 – 0)) * 100.0 / (100 * 1 * 1) =201，也就是201%。你会发现这个值和我们运行top里的值是一样的。同时，我们也就验证了这个公式是没问题的。</p><h3>系统CPU使用率</h3><p>前面我们介绍了Linux中如何获取单个进程的CPU使用率，下面我们再来看看Linux里是怎么计算系统的整体CPU使用率的。</p><p>其实知道了如何计算单个进程的CPU使用率之后，要理解系统整体的CPU使用率计算方法就简单多了。</p><p>同样，我们要计算CPU使用率，首先需要拿到数据，数据源也同样可以从proc文件系统里得到，对于整个系统的CPU使用率，这个文件就是/proc/stat。</p><p>在/proc/stat 文件的 <code>cpu</code> 这行有10列数据，同样我们可以在proc文件系统的 <a href="https://man7.org/linux/man-pages/man5/proc.5.html"></a><a href="https://man7.org/linux/man-pages/man5/proc.5.html">Linux programmer’s manual</a> 里，找到每一列数据的定义，而前8列数据正好对应top输出中"%Cpu(s)"那一行里的8项数据，也就是在上一讲中，我们介绍过的user/system/nice/idle/iowait/irq/softirq/steal 这8项。</p><p><img src="https://static001.geekbang.org/resource/image/7d/9d/7da412b1b9e4f4b235e5dae6c997859d.png?wh=1748*62" alt=""></p><p>而在/proc/stat里的每一项的数值，就是系统自启动开始的ticks。那么要计算出“瞬时”的CPU使用率，首先就要算出这个“瞬时”的ticks，比如1秒钟的“瞬时”，我们可以记录开始时刻T1的ticks, 然后再记录1秒钟后T2时刻的ticks，再把这两者相减，就可以得到这1秒钟的ticks了。</p><p><img src="https://static001.geekbang.org/resource/image/fa/60/fa59040d49cb20d808619957ae1e2760.png?wh=1452*140" alt=""></p><p>这里我们可以得到，在这1秒钟里每个CPU使用率的ticks：</p><p><img src="https://static001.geekbang.org/resource/image/f5/bd/f550d24358e7cc909ff337f4840512bd.jpeg?wh=3200*1800" alt=""></p><p>我们想要计算每一种CPU使用率的百分比，其实也很简单。我们只需要把所有在这1秒里的ticks相加得到一个总值，然后拿某一项的ticks值，除以这个总值。比如说计算idle CPU的使用率就是：</p><p>1203 /（ 0 + 0 + 0 + 1203 + 0 + 0 + 0 + 0）=100%</p><p>好了，我们现在来整体梳理一下，我们通过Linux里的工具，要怎样计算进程的CPU使用率和系统的CPU使用率。</p><p>对于单个进程的CPU使用率计算，我们需要读取对应进程的/proc/[pid]/stat文件，将进程瞬时用户态和内核态的ticks数相加，就能得到进程的总ticks。</p><p>然后我们运用公式“(进程的ticks / 单个CPU总ticks) * 100.0”计算出进程CPU使用率的百分比值。</p><p>对于系统的CPU使用率，需要读取/proc/stat文件，得到瞬时各项CPU使用率的ticks值，相加得到一个总值，单项值除以总值就是各项CPU的使用率。</p><h2>解决问题</h2><p>前面我们学习了在Linux中，top工具是怎样计算每个进程的CPU使用率，以及系统总的CPU使用率。现在我们再来看最初的问题：为什么在容器中运行top命令不能得到容器中总的CPU使用率？</p><p>这就比较好解释了，对于系统总的CPU使用率，需要读取/proc/stat文件，但是这个文件中的各项CPU ticks是反映整个节点的，并且这个/proc/stat文件也不包含在任意一个Namespace里。</p><p>那么，<strong>对于top命令来说，它只能显示整个节点中各项CPU的使用率，不能显示单个容器的各项CPU的使用率</strong>。既然top命令不行，我们还有没有办法得到整个容器的CPU使用率呢？</p><p>我们之前已经学习过了CPU Cgroup，每个容器都会有一个CPU Cgroup的控制组。在这个控制组目录下面有很多参数文件，有的参数可以决定这个控制组里最大的CPU可使用率外，除了它们之外，目录下面还有一个可读项cpuacct.stat。</p><p>这里包含了两个统计值，这两个值分别是<strong>这个控制组里所有进程的内核态ticks和用户态的ticks</strong>，那么我们就可以用前面讲过的公式，也就是计算进程CPU使用率的公式，去计算整个容器的CPU使用率：</p><p>CPU使用率=((utime_2 – utime_1) + (stime_2 – stime_1)) * 100.0 / (HZ * et * 1 )</p><p>我们还是以问题重现中的例子说明，也就是最开始启动容器里的那两个容器threads-cpu进程。</p><p>就像下图显示的这样，整个容器的CPU使用率的百分比就是 ( (174021 - 173820) + (4 – 4)) * 100.0 / (100 * 1 * 1) = 201, 也就是201%。<strong>所以，我们从每个容器的CPU Cgroup控制组里的cpuacct.stat的统计值中，</strong>可以比较快地得到整个容器的CPU使用率。</p><p><img src="https://static001.geekbang.org/resource/image/fd/95/fdcfd5yy6e593cccea1bed7bfe4e5095.png?wh=1920*321" alt=""></p><h2>重点总结</h2><p>Linux里获取CPU使用率的工具，比如top，都是通过读取proc文件系统下的stat文件来得到CPU使用了多少ticks。而这里的ticks，是Linux操作系统里的一个时间单位，可以理解成类似秒，毫秒的概念。</p><p>对于每个进程来说，它的stat文件是/proc/[pid]/stat，里面包含了进程用户态和内核态的ticks数目；对于整个节点，它的stat文件是 /proc/stat，里面包含了user/system/nice/idle/iowait等不同CPU开销类型的ticks。</p><p><strong>由于/proc/stat文件是整个节点全局的状态文件，不属于任何一个Namespace，因此在容器中无法通过读取/proc/stat文件来获取单个容器的CPU使用率。</strong></p><p>所以要得到单个容器的CPU使用率，我们可以从CPU Cgroup每个控制组里的统计文件cpuacct.stat中获取。<strong>单个容器CPU使用率=((utime_2 – utime_1) + (stime_2 – stime_1)) * 100.0 / (HZ * et * 1 )。</strong></p><p>得到单个容器的CPU的使用率，那么当宿主机上负载变高的时候，就可以很快知道是哪个容器引起的问题。同时，用户在管理自己成百上千的容器的时候，也可以很快发现CPU使用率异常的容器，这样就能及早地介入去解决问题。</p><h2>思考题</h2><p>写一个小程序，在容器中执行，它可以显示当前容器中所有进程总的CPU使用率。</p><p>欢迎在留言区和我互动，一起探讨容器CPU的相关问题。如果这篇文章让你有所收获，也欢迎你分享给更多的朋友，一起学习进步。</p>
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
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTIz9dKN1C8rKQoaVtmEGdzObhlia6zAfTsPYOm4ibz39VjTbu7Aia1LyeedHR26b6nxUtcCufpichcYgw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>上邪忘川</span>
  </div>
  <div class="_2_QraFYR_0">随便写了一个，比较粗糙<br>#!&#47;bin&#47;bash<br>cpuinfo1=$(cat &#47;sys&#47;fs&#47;cgroup&#47;cpu,cpuacct&#47;cpuacct.stat)<br>utime1=$(echo $cpuinfo1|awk &#39;{print $2}&#39;)<br>stime1=$(echo $cpuinfo1|awk &#39;{print $4}&#39;)<br>sleep 1<br>cpuinfo2=$(cat &#47;sys&#47;fs&#47;cgroup&#47;cpu,cpuacct&#47;cpuacct.stat)<br>utime2=$(echo $cpuinfo2|awk &#39;{print $2}&#39;)<br>stime2=$(echo $cpuinfo2|awk &#39;{print $4}&#39;)<br>cpus=$((utime2+stime2-utime1-stime1))<br>echo &quot;${cpus}%&quot;<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-29 23:38:13</div>
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
  <div class="_2_QraFYR_0">多谢老师的分享，之前想去搞明白进程CPU时间的计算，一直没有去花时间研究，今天终于透彻地明白了。<br><br>我有个问题不太明白，想请教下，容器运行时比如docker，在做容器化的时候，有没有办法构造出一个和物理机一样的proc文件系统呢？这样的话，容器环境和虚拟机也没啥差别了，物理机上的应用也可以无障碍运行在容器环境中。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 很高兴，这篇文章对你有帮助。<br><br>https:&#47;&#47;github.com&#47;lxc&#47;lxcfs， lxcfs可以为每个容器虚拟一些&#47;proc下的文件，比如&#47;proc&#47;stat</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-27 11:24:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/ac/62/37912d51.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>东方奇骥</span>
  </div>
  <div class="_2_QraFYR_0">老师，ticks 1s中是100次，这个怎么查呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: USER_HZ, 可以用命令 &quot;getconf CLK_TCK&quot;拿</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-27 11:13:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/bf/aa/32a6449c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>蒋悦</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，<br>我有一个问题。根据我的理解，容器的cpu使用就必须要在容器内进行，从宿主机是无法计算的，是这样吗？如果是的，那么，这个监控cpu的代码就需要侵入程序代码(容器中跑的业务代码)，这会不会有些无奈啊？另外，这个侵入的代码，所在的线程如果不能被实时调度，则瞬时速度就算的不准确了吧？<br><br>望解答。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: @蒋悦<br>&gt; 容器的cpu使用就必须要在容器内进行，从宿主机是无法计算的，是这样吗?<br><br>不是这样的，从宿主机也可以得到容器对应的CPU Cgroup里的值。 </p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-27 13:19:22</div>
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
  <div class="_2_QraFYR_0">老师，非常感谢，之前都是模糊知道大概这个意思，这次明白了<br><br>另外有个问题：<br>我当然做监控的时候发现docker stats 和cadvisor(或通过cgroup直接计算)通过cgroup拿到的cpu使用率，内存使用率都是不一样的，您这边知道根本的原因吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: @争光 Alan<br>你是指 cpu&#47;memory 使用率在分别在docker stats 和cadvisor里是不一样的？<br>我们用cadvisor, 看到的结果大致和cgroup差不多，docker stats也是通过cgroup计算的。<br>你看到的差异有多大？或者你看到的结果，和你自己通过cgroup里的值计算比较一下，哪个更加接近一些？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-02 08:42:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/8f/5f/11ba01f2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>路一直在</span>
  </div>
  <div class="_2_QraFYR_0">老师，如果计算容器整体的cpu使用率，使用cpuacct.stat是否完整，因为cpuacct.stat中只有us和sys的ticks，其他的类似iowait、idle等的ticks不用计算吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 其他的cpu usage, 如io&#47;hi&#47;si等更多的是从宿主机角度统计的，也不能知道这些开销是属于哪个进程的，也就很难放到每一个cgroup里。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-04 17:23:16</div>
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
  <div class="_2_QraFYR_0">老师，&#47;sys&#47;fs&#47;cgroup&#47;cpu 和 cpuacc，这2个有什么区别<br>还有这个获取容器中是获取容器的CPU使用，宿主机是获取宿主机的CPU使用吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这两个目录是一样的，<br>lrwxrwxrwx 1 root root 11 Dec 14 23:51 cpu -&gt; cpu,cpuacct<br>lrwxrwxrwx 1 root root 11 Dec 14 23:51 cpuacct -&gt; cpu,cpuacct<br>dr-xr-xr-x 5 root root  0 Dec 14 23:51 cpu,cpuacct</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-15 15:14:05</div>
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
  <div class="_2_QraFYR_0">老师问一下，像Prometheus或者k8s自己的metrics server获取到的pod各个资源使用率(cpu men net io)也是这样通过查看进程&#47;proc来计算得出的吗？如果是这样的话，指标数据太多，运算太大会不会也会消耗节点资源呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: cAdvisor就是通过读取sys cgroup 或者 &#47;proc下的状态信息来得到container cpu&#47;mem&#47;net&#47;io metrics的。<br>一般一个节点上的container数量是小于1000的，metrics获取的周期在10s 的情况下， 这个资源消耗不会很大。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-27 15:05:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/23/9a/39/a762e79d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>JZ</span>
  </div>
  <div class="_2_QraFYR_0">老师，获取cpu的开销可以使用kubectl top这个工具吧，这个拿到的值和文章中公式计算的应该是一致的吧</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-11 10:33:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/a9/bc/d3e25886.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>GCC?</span>
  </div>
  <div class="_2_QraFYR_0">老师，在k8s集群中，使用metrics查看到整个pod的资源用量，这个粒度已经够了吧？我的理解是如果pod内不是单一容器，或者容器内有多个进程，这个时候才应该考虑进程级的cpu用量。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-02 00:09:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/b7/24/17f6c240.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>janey</span>
  </div>
  <div class="_2_QraFYR_0">感觉这种计算方法不是很简便，节点上容器多的话，不好判断是哪个容器消耗CPU最多吧</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-02 09:21:15</div>
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
  <div class="_2_QraFYR_0">老师你好，我想请问以下docker stats显示的cpu%依旧是不正确的吗，如果是文中的统计方法根据我的测试是不正确的。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-13 23:49:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/20/7e/e0/ff046b55.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>李兵</span>
  </div>
  <div class="_2_QraFYR_0">老师，如已经是高负载的情况下。应该会有很多正常的进程被安排在D进程中，如何区分造成高负载的D进程呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果是单纯的高计算的进程，状态多为R.</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-22 00:48:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/e3/9e/b26da70d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>closer</span>
  </div>
  <div class="_2_QraFYR_0">问题1）生产上面 如果把nginx放入到pod中，是不是会碰到目前为止老师说的那些所有问题 比如关闭容器sigterm主进程nginx master。所有的nginx worker都被sigkill。<br>问题2）然后容器里面的各个进程用的cpu总合是会被k8s metrics计算到的吧。<br>谢谢。期待老师百忙中回复</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: &gt; 问题1<br>nginx 似乎有自己的master process 和 client processes之间的socket连接。如果nginx master作为容器里的pid 1, 当关闭容器的时候master收到SIGTERM, 而其他的client子进程是看到socket连接断开之后，主动退出的。<br><br>&gt; 问题2<br>是的， 用cAdvisor可以统计整个容器的cpu.</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-29 09:55:36</div>
  </div>
</div>
</div>
</li>
</ul>