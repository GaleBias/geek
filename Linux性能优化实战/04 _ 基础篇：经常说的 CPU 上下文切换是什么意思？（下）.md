<audio title="04 _ 基础篇：经常说的 CPU 上下文切换是什么意思？（下）" src="https://static001.geekbang.org/resource/audio/68/58/68653d49fda5636f3cdd6063f2accf58.mp3" controls="controls"></audio> 
<p>你好，我是倪朋飞。</p><p>上一节，我给你讲了CPU上下文切换的工作原理。简单回顾一下，CPU 上下文切换是保证 Linux 系统正常工作的一个核心功能，按照不同场景，可以分为进程上下文切换、线程上下文切换和中断上下文切换。具体的概念和区别，你也要在脑海中过一遍，忘了的话及时查看上一篇。</p><p>今天我们就接着来看，究竟怎么分析CPU上下文切换的问题。</p><h2>怎么查看系统的上下文切换情况</h2><p>通过前面学习我们知道，过多的上下文切换，会把CPU 时间消耗在寄存器、内核栈以及虚拟内存等数据的保存和恢复上，缩短进程真正运行的时间，成了系统性能大幅下降的一个元凶。</p><p>既然上下文切换对系统性能影响那么大，你肯定迫不及待想知道，到底要怎么查看上下文切换呢？在这里，我们可以使用 vmstat 这个工具，来查询系统的上下文切换情况。</p><p>vmstat 是一个常用的系统性能分析工具，主要用来分析系统的内存使用情况，也常用来分析 CPU 上下文切换和中断的次数。</p><p>比如，下面就是一个 vmstat 的使用示例：</p><pre><code># 每隔5秒输出1组数据
$ vmstat 5
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 0  0      0 7005360  91564 818900    0    0     0     0   25   33  0  0 100  0  0
</code></pre><p>我们一起来看这个结果，你可以先试着自己解读每列的含义。在这里，我重点强调下，需要特别关注的四列内容：</p><ul>
<li>
<p>cs（context switch）是每秒上下文切换的次数。</p>
</li>
<li>
<p>in（interrupt）则是每秒中断的次数。</p>
</li>
<li>
<p>r（Running or Runnable）是就绪队列的长度，也就是正在运行和等待CPU的进程数。</p>
</li>
<li>
<p>b（Blocked）则是处于不可中断睡眠状态的进程数。</p>
</li>
</ul><!-- [[[read_end]]] --><p>可以看到，这个例子中的上下文切换次数 cs 是33次，而系统中断次数 in 则是25次，而就绪队列长度r和不可中断状态进程数b都是0。</p><p>vmstat 只给出了系统总体的上下文切换情况，要想查看每个进程的详细情况，就需要使用我们前面提到过的 pidstat  了。给它加上 -w 选项，你就可以查看每个进程上下文切换的情况了。</p><p>比如说：</p><pre><code># 每隔5秒输出1组数据
$ pidstat -w 5
Linux 4.15.0 (ubuntu)  09/23/18  _x86_64_  (2 CPU)

08:18:26      UID       PID   cswch/s nvcswch/s  Command
08:18:31        0         1      0.20      0.00  systemd
08:18:31        0         8      5.40      0.00  rcu_sched
...
</code></pre><p>这个结果中有两列内容是我们的重点关注对象。一个是  cswch  ，表示每秒自愿上下文切换（voluntary context switches）的次数，另一个则是  nvcswch  ，表示每秒非自愿上下文切换（non voluntary context switches）的次数。</p><p>这两个概念你一定要牢牢记住，因为它们意味着不同的性能问题：</p><ul>
<li>
<p>所谓<strong>自愿上下文切换，是指进程无法获取所需资源，导致的上下文切换</strong>。比如说， I/O、内存等系统资源不足时，就会发生自愿上下文切换。</p>
</li>
<li>
<p>而<strong>非自愿上下文切换，则是指进程由于时间片已到等原因，被系统强制调度，进而发生的上下文切换</strong>。比如说，大量进程都在争抢 CPU 时，就容易发生非自愿上下文切换。</p>
</li>
</ul><h2>案例分析</h2><p>知道了怎么查看这些指标，另一个问题又来了，上下文切换频率是多少次才算正常呢？别急着要答案，同样的，我们先来看一个上下文切换的案例。通过案例实战演练，你自己就可以分析并找出这个标准了。</p><h3>你的准备</h3><p>今天的案例，我们将使用 sysbench 来模拟系统多线程调度切换的情况。</p><p>sysbench 是一个多线程的基准测试工具，一般用来评估不同系统参数下的数据库负载情况。当然，在这次案例中，我们只把它当成一个异常进程来看，作用是模拟上下文切换过多的问题。</p><p>下面的案例基于 Ubuntu 18.04，当然，其他的 Linux 系统同样适用。我使用的案例环境如下所示：</p><ul>
<li>
<p>机器配置：2 CPU，8GB 内存</p>
</li>
<li>
<p>预先安装 sysbench 和 sysstat 包，如 apt install sysbench sysstat</p>
</li>
</ul><p>正式操作开始前，你需要打开三个终端，登录到同一台 Linux 机器中，并安装好上面提到的两个软件包。包的安装，可以先Google一下自行解决，如果仍然有问题的，在留言区写下你的情况。</p><p>另外注意，下面所有命令，都<strong>默认以 root 用户运行</strong>。所以，如果你是用普通用户登陆的系统，记住先运行 sudo su root 命令切换到 root 用户。</p><p>安装完成后，你可以先用 vmstat 看一下空闲系统的上下文切换次数：</p><pre><code># 间隔1秒后输出1组数据
$ vmstat 1 1
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 0  0      0 6984064  92668 830896    0    0     2    19   19   35  1  0 99  0  0
</code></pre><p>这里你可以看到，现在的上下文切换次数 cs 是35，而中断次数 in 是19，r和b都是0。因为这会儿我并没有运行其他任务，所以它们就是空闲系统的上下文切换次数。</p><h3>操作和分析</h3><p>接下来，我们正式进入实战操作。</p><p>首先，在第一个终端里运行 sysbench  ，模拟系统多线程调度的瓶颈：</p><pre><code># 以10个线程运行5分钟的基准测试，模拟多线程切换的问题
$ sysbench --threads=10 --max-time=300 threads run
</code></pre><p>接着，在第二个终端运行 vmstat  ，观察上下文切换情况：</p><pre><code># 每隔1秒输出1组数据（需要Ctrl+C才结束）
$ vmstat 1
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 6  0      0 6487428 118240 1292772    0    0     0     0 9019 1398830 16 84  0  0  0
 8  0      0 6487428 118240 1292772    0    0     0     0 10191 1392312 16 84  0  0  0
</code></pre><p>你应该可以发现，cs 列的上下文切换次数从之前的 35 骤然上升到了 139 万。同时，注意观察其他几个指标：</p><ul>
<li>
<p>r 列：就绪队列的长度已经到了 8，远远超过了系统 CPU 的个数 2，所以肯定会有大量的 CPU 竞争。</p>
</li>
<li>
<p>us（user）和 sy（system）列：这两列的CPU 使用率加起来上升到了 100%，其中系统 CPU 使用率，也就是 sy 列高达 84%，说明 CPU 主要是被内核占用了。</p>
</li>
<li>
<p>in  列：中断次数也上升到了1万左右，说明中断处理也是个潜在的问题。</p>
</li>
</ul><p>综合这几个指标，我们可以知道，系统的就绪队列过长，也就是正在运行和等待CPU的进程数过多，导致了大量的上下文切换，而上下文切换又导致了系统 CPU 的占用率升高。</p><p>那么到底是什么进程导致了这些问题呢？</p><p>我们继续分析，在第三个终端再用 pidstat 来看一下， CPU 和进程上下文切换的情况：</p><pre><code># 每隔1秒输出1组数据（需要 Ctrl+C 才结束）
# -w参数表示输出进程切换指标，而-u参数则表示输出CPU使用指标
$ pidstat -w -u 1
08:06:33      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
08:06:34        0     10488   30.00  100.00    0.00    0.00  100.00     0  sysbench
08:06:34        0     26326    0.00    1.00    0.00    0.00    1.00     0  kworker/u4:2

08:06:33      UID       PID   cswch/s nvcswch/s  Command
08:06:34        0         8     11.00      0.00  rcu_sched
08:06:34        0        16      1.00      0.00  ksoftirqd/1
08:06:34        0       471      1.00      0.00  hv_balloon
08:06:34        0      1230      1.00      0.00  iscsid
08:06:34        0      4089      1.00      0.00  kworker/1:5
08:06:34        0      4333      1.00      0.00  kworker/0:3
08:06:34        0     10499      1.00    224.00  pidstat
08:06:34        0     26326    236.00      0.00  kworker/u4:2
08:06:34     1000     26784    223.00      0.00  sshd
</code></pre><p>从pidstat的输出你可以发现，CPU 使用率的升高果然是 sysbench 导致的，它的 CPU 使用率已经达到了 100%。但上下文切换则是来自其他进程，包括非自愿上下文切换频率最高的 pidstat  ，以及自愿上下文切换频率最高的内核线程 kworker 和 sshd。</p><p>不过，细心的你肯定也发现了一个怪异的事儿：pidstat 输出的上下文切换次数，加起来也就几百，比 vmstat 的 139 万明显小了太多。这是怎么回事呢？难道是工具本身出了错吗？</p><p>别着急，在怀疑工具之前，我们再来回想一下，前面讲到的几种上下文切换场景。其中有一点提到， Linux 调度的基本单位实际上是线程，而我们的场景 sysbench 模拟的也是线程的调度问题，那么，是不是 pidstat 忽略了线程的数据呢？</p><p>通过运行 man pidstat  ，你会发现，pidstat 默认显示进程的指标数据，加上 -t 参数后，才会输出线程的指标。</p><p>所以，我们可以在第三个终端里，  Ctrl+C 停止刚才的 pidstat 命令，再加上 -t 参数，重试一下看看：</p><pre><code># 每隔1秒输出一组数据（需要 Ctrl+C 才结束）
# -wt 参数表示输出线程的上下文切换指标
$ pidstat -wt 1
08:14:05      UID      TGID       TID   cswch/s nvcswch/s  Command
...
08:14:05        0     10551         -      6.00      0.00  sysbench
08:14:05        0         -     10551      6.00      0.00  |__sysbench
08:14:05        0         -     10552  18911.00 103740.00  |__sysbench
08:14:05        0         -     10553  18915.00 100955.00  |__sysbench
08:14:05        0         -     10554  18827.00 103954.00  |__sysbench
...
</code></pre><p>现在你就能看到了，虽然 sysbench 进程（也就是主线程）的上下文切换次数看起来并不多，但它的子线程的上下文切换次数却有很多。看来，上下文切换罪魁祸首，还是过多的 sysbench 线程。</p><p>我们已经找到了上下文切换次数增多的根源，那是不是到这儿就可以结束了呢？</p><p>当然不是。不知道你还记不记得，前面在观察系统指标时，除了上下文切换频率骤然升高，还有一个指标也有很大的变化。是的，正是中断次数。中断次数也上升到了1万，但到底是什么类型的中断上升了，现在还不清楚。我们接下来继续抽丝剥茧找源头。</p><p>既然是中断，我们都知道，它只发生在内核态，而 pidstat 只是一个进程的性能分析工具，并不提供任何关于中断的详细信息，怎样才能知道中断发生的类型呢？</p><p>没错，那就是从 /proc/interrupts 这个只读文件中读取。/proc 实际上是 Linux 的一个虚拟文件系统，用于内核空间与用户空间之间的通信。/proc/interrupts 就是这种通信机制的一部分，提供了一个只读的中断使用情况。</p><p>我们还是在第三个终端里，  Ctrl+C 停止刚才的 pidstat 命令，然后运行下面的命令，观察中断的变化情况：</p><pre><code># -d 参数表示高亮显示变化的区域
$ watch -d cat /proc/interrupts
           CPU0       CPU1
...
RES:    2450431    5279697   Rescheduling interrupts
...
</code></pre><p>观察一段时间，你可以发现，变化速度最快的是<strong>重调度中断</strong>（RES），这个中断类型表示，唤醒空闲状态的 CPU 来调度新的任务运行。这是多处理器系统（SMP）中，调度器用来分散任务到不同 CPU 的机制，通常也被称为<strong>处理器间中断</strong>（Inter-Processor Interrupts，IPI）。</p><p>所以，这里的中断升高还是因为过多任务的调度问题，跟前面上下文切换次数的分析结果是一致的。</p><p>通过这个案例，你应该也发现了多工具、多方面指标对比观测的好处。如果最开始时，我们只用了  pidstat 观测，这些很严重的上下文切换线程，压根儿就发现不了了。</p><p>现在再回到最初的问题，每秒上下文切换多少次才算正常呢？</p><p><strong>这个数值其实取决于系统本身的 CPU 性能</strong>。在我看来，如果系统的上下文切换次数比较稳定，那么从数百到一万以内，都应该算是正常的。但当上下文切换次数超过一万次，或者切换次数出现数量级的增长时，就很可能已经出现了性能问题。</p><p>这时，你还需要根据上下文切换的类型，再做具体分析。比方说：</p><ul>
<li>
<p>自愿上下文切换变多了，说明进程都在等待资源，有可能发生了 I/O 等其他问题；</p>
</li>
<li>
<p>非自愿上下文切换变多了，说明进程都在被强制调度，也就是都在争抢 CPU，说明 CPU 的确成了瓶颈；</p>
</li>
<li>
<p>中断次数变多了，说明 CPU 被中断处理程序占用，还需要通过查看 /proc/interrupts 文件来分析具体的中断类型。</p>
</li>
</ul><h2>小结</h2><p>今天，我通过一个sysbench的案例，给你讲了上下文切换问题的分析思路。碰到上下文切换次数过多的问题时，<strong>我们可以借助 vmstat  、  pidstat 和 /proc/interrupts 等工具</strong>，来辅助排查性能问题的根源。</p><h2>思考</h2><p>最后，我想请你一起来聊聊，你之前是怎么分析和排查上下文切换问题的。你可以结合这两节的内容和你自己的实际操作，来总结自己的思路。</p><p>欢迎在留言区和我讨论，也欢迎把这篇文章分享给你的同事、朋友。我们一起在实战中演练，在交流中学习。</p><p><img src="https://static001.geekbang.org/resource/image/56/52/565d66d658ad23b2f4997551db153852.jpg?wh=1110*549" alt=""></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/3b/36/2d61e080.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>行者</span>
  </div>
  <div class="_2_QraFYR_0">结合前两节，首先通过uptime查看系统负载，然后使用mpstat结合pidstat来初步判断到底是cpu计算量大还是进程争抢过大或者是io过多，接着使用vmstat分析切换次数，以及切换类型，来进一步判断到底是io过多导致问题还是进程争抢激烈导致问题。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-28 00:28:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/65/ca/38dcd55a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lupguo</span>
  </div>
  <div class="_2_QraFYR_0">1. stress和sysbench两个工具在压测过程中的对比发现：<br>stress基于多进程的，会fork多个进程，导致进程上下文切换，导致us开销很高；<br>sysbench基于多线程的，会创建多个线程，单一进程基于内核线程切换，导致sy的内核开销很高；<br>具体可以通过vmstat对比<br>stress -c 8 -i 16 -t 600<br>vmstat 1 5<br>sysbench --threads=20 --time=300 threads run<br>vmstat 1 5<br>2. 和鸟哥说的一样，不懂多看man，see also的命令基本涵盖了老师讲解的<br>3. 建议结合操作系统完成后，再看这块教程，整体会更系统性，看问题会更加客观<br><br>希望对大家有帮助</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-03 19:58:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/36/d2/c7357723.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>发条橙子 。</span>
  </div>
  <div class="_2_QraFYR_0">案例分析 ：<br><br>登录到服务器，现在系统负载怎么样 。 高的话有三种情况，首先是cpu使用率 ，其次是io使用率 ，之后就是两者都高 。 <br><br>cpu 使用率高，可能确实是使用率高， 也的可能实际处理不高而是进程太多切换上下文频繁 ， 也可能是进程内线程的上下文切换频繁<br><br>io 使用率高 ， 说明 io 请求比较大， 可能是 文件io 、 网络io 。  <br><br>工具 ：<br>系统负载 ：  uptime   （ watch -d uptime）看三个阶段平均负载<br>系统整体情况 ：  mpstat （mpstat -p ALL 3） 查看 每个cpu当前的整体状况，可以重点看用户态、内核态、以及io等待三个参数<br>系统整体的平均上下文切换情况 ： vmstat   (vmstat 3) 可以重点看 r （进行或等待进行的进程）、b （不可中断进程&#47;io进程） 、in （中断次数） 、cs（上下文切换次数）  <br>查看详细的上下文切换情况 ： pidstat （pidstat -w(进程切换指标)&#47;-u（cpu使用指标）&#47;-wt(线程上下文切换指标)） 注意看是自愿上下文切换、还是被动上下文切换<br>io使用情况 ： iostat<br><br>模拟场景工具 ：<br>stress ： 模拟进程 、 io<br>sysbench ： 模拟线程数</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 总结的很好，继续保持👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-02 10:28:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/51/0f/17bdfdaf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>酱油侠</span>
  </div>
  <div class="_2_QraFYR_0">我用的centos，yum装的sysbench。执行后很快完事了的可以设置下max-requests，默认max-requests是1w所以很快就结束了。<br>sysbench --num-threads=10 --max-time=300 --max-requests=10000000 --test=threads run<br>有的朋友&#47;proc&#47;interrupts时看不见RES是因为窗口开太小了RES在最下面。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍 谢谢分享经验，其他用centos的同学可以参考一下</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-29 18:04:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/0c/e1/e54540b9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>冯宇</span>
  </div>
  <div class="_2_QraFYR_0">友情提醒，sudo -i就可以快速切换到root啦😄 不加-i的话是以非登录模式切换，不会拿到root的环境变量</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-30 10:27:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/5b/e0/d27145c8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>discoverer-tab</span>
  </div>
  <div class="_2_QraFYR_0">过多上下文切换会缩短进程运行时间vmstat 1 1：分析内存使用情况、cpu上下文切换和中断的次数。cs每秒上下文切换的次数，in每秒中断的次数，r运行或等待cpu的进程数，b中断睡眠状态的进程数。pidstat -w 5：查看每个进程详细情况。cswch（每秒自愿）上下文切换次数，如系统资源不足导致，nvcswch每秒非自愿上下文切换次数，如cpu时间片用完或高优先级线程案例分析：sysbench：多线程的基准测试工具，模拟context switch终端1：sysbench --threads=10 --max-time=300 threads run终端2：vmstat 1：sys列占用84%说明主要被内核占用，ur占用16%；r就绪队列8；in中断处理1w，cs切换139w==&gt;等待进程过多，频繁上下文切换，内核cpu占用率升高终端3：pidstat -w -u 1：sysbench的cpu占用100%（-wt发现子线程切换过多），其他进程导致上下文切换watch -d cat &#47;proc&#47;interupts ：查看另一个指标中断次数，在&#47;proc&#47;interupts中读取，发现重调度中断res变化速度最快总结：cswch过多说明资源IO问题，nvcswch过多说明调度争抢cpu过多，中断次数变多说明cpu被中断程序调用</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 总结的好👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-28 16:42:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/78/e9/9d807269.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>miracle</span>
  </div>
  <div class="_2_QraFYR_0">发现一个不太严谨的地方，即使没有开sysbench，用watch -d &#47;proc&#47;interrupts的时候 RES的变化也是最大的，这个时候in跟cs都不高</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 中断次数是多少？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-28 10:30:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/f6/4e/0066303c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>cuikt</span>
  </div>
  <div class="_2_QraFYR_0">可以通过以下指令进行排序，观察RES。<br>watch -d &#39;cat &#47;proc&#47;interrupts | sort -nr -k 2 &#39;</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢分享，sort、cat 都是所有人都需要掌握的基础工具</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-30 15:54:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/1a/82/ca3ef12c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Haku</span>
  </div>
  <div class="_2_QraFYR_0">Ubuntu16.04LTS下:<br># 以 10 个线程运行 5 分钟的基准测试，模拟多线程切换的问题<br>$ sysbench --num-threads=10 --max-time=300 --test=threads run</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-28 09:55:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/64/86/f5a9403a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>多襄丸</span>
  </div>
  <div class="_2_QraFYR_0">说RES=0的同学，你一定是设置了CPU个数=1。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-01 13:55:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c7/30/2f8b78e9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>CYH</span>
  </div>
  <div class="_2_QraFYR_0">老师请教一下：我的是centos7系统，执行sysbetch后，watch -d cat &#47;proc&#47;interrupts并没有发现您文中描述的重调度中断指标呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-28 22:49:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/PiajxSqBRaEIHvQHWCJ6cjAjFthVAADNWx0uaZicm4UDJCbVbvcian6gFI1gqWWX2tG3lEB2nVNtaVXtwibGOYiauCA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>echo</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，有个难题希望能指导一下排查<br>我有个系统，跑的是32核48G的云主机，load经常超过CPU核数，峰值时load5可达到CPU核数的3倍， 但是CPU利用率不超过50%左右。<br>其他关键数据：I&#47;O wait 不超过0.1， 网络流量没超出网卡QOS，R状态的进程数也就一两个，没有D状态的进程。系统只要跑一个CPU密集型的Java进程，线程数2-3k。另外load、CPU、网卡流量的曲线是一致的。<br><br>通读了你的第二篇文章，按文章指导能排查的都排查了，接下来应该从哪方面着手定位load高的根因呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: CPU使用率都是哪种高？具体到每个线程，又是哪种CPU使用率高？先找出最高的类型再继续分析</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-28 10:01:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/15/0f/954be2db.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>茴香根</span>
  </div>
  <div class="_2_QraFYR_0">打卡本节课程，在使用Linux一些监控命令行时候常常碰到列宽和下面的的数据错位的情况，比如数据过大，占了两列，导致数据错位，不方便观察，不知老师可有好的工具或方法解决。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这是常见的问题，一般用宽显示器会好些</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-28 09:02:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/9b/ba/333b59e5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Linuxer</span>
  </div>
  <div class="_2_QraFYR_0">我们之前是如果系统CPU不高根本不会去关注上下文切换，但是这种情况下以前也观测到cs有几十万的情况，所以我想请教一个问题，什么情况下需要关注上下文切换呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，cs值不是绝对的，所以最好是监控起来，看变化情况，比如是不是数量级的增长</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-28 08:28:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/20/1d/0c1a184c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>罗辑思维</span>
  </div>
  <div class="_2_QraFYR_0"><br>CPU上下文切换<br>三个场景识切换：进程上下文切换、线程上下文切换以及中断上下文切换。<br>二个概念牢牢记：自愿上下文切换、非自愿上下文切换。<br>一个中断不要忘：中断次数。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-19 19:38:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/3e/22/f91219a8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>wanlinwang</span>
  </div>
  <div class="_2_QraFYR_0">进程状态说明<br><br>R (task_running) : 可执行状态<br><br>S (task_interruptible): 可中断的睡眠状态<br><br>D (task_uninterruptible): 不可中断的睡眠状态<br><br>T(task_stopped or task_traced)：暂停状态或跟踪状态<br><br>Z (task_dead - exit_zombie)：退出状态，进程成为僵尸进程<br><br>X (task_dead - exit_dead)：退出状态，进程即将被销毁<br><br>文章中写的是b状态是不可中断的睡眠状态，哪个是正确的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不同工具的输出格式不完全一样，文章中的b其实就是D状态的进程</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-16 20:08:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/1b/82/69581d8a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>姜小鱼</span>
  </div>
  <div class="_2_QraFYR_0">老师 为什么我执行sysbench之后很快就结束了？sysbench --num-threads=10 --max-time=600 --test=threads run 我用的是ubuntu16</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 有没有报错误？可以加上—debug看看有没有错误消息</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-28 10:41:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/85/3a/bb885be4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>glinuxer</span>
  </div>
  <div class="_2_QraFYR_0">Pidstats确实是把利器啊</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-28 05:54:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTIbas4S4X5W15njMeoEPPSyBZRX37nrTXbMFFeHghXl4Slk6WXE7oq5yxoNnukYfcOQs00RAvUmEA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_5258f8</span>
  </div>
  <div class="_2_QraFYR_0">老师，请教一个问题:线程多了会带来性能下降。我们假定一个理想境，所有线程优先级相同，即除了时间片到期切换外，无其他触发条件。那1s内发生的线程切换次数是相同的，即1s&#47;20ms次。从这个角度思考，线程的多少并不影响cpu的整体吞吐量。只是影响对某单个线程的响应时间。我的理解对吗？(这里也去掉了cache失效的影响)</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-02 21:09:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/dd/9b/0bc44a78.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>yyl</span>
  </div>
  <div class="_2_QraFYR_0">“自愿上下文切换变多了，说明进程都在等待资源，有可能发生了 I&#47;O 等其他问题；<br>非自愿上下文切换变多了，说明进程都在被强制调度，也就是都在争抢 CPU，说明 CPU 的确成了瓶颈；”<br>是否可以这样理解：<br>自愿上下文切换，是任务在竞争除CPU的其他资源，其他资源成为瓶颈；<br>非自愿上下文切换，是任务在竞争CPU，由于执行优先级 或 执行时间片达到等导致的强制释放CPU资源</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-01 11:01:06</div>
  </div>
</div>
</div>
</li>
</ul>