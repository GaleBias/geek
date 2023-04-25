<audio title="13 _ 答疑（一）：无法模拟出 RES 中断的问题，怎么办？" src="https://static001.geekbang.org/resource/audio/61/4d/614e31077f92dcc9a8c7850ecb6e3c4d.mp3" controls="controls"></audio> 
<p>你好，我是倪朋飞。</p><p>专栏更新至今，四大基础模块之一的CPU性能篇，我们就已经学完了。很开心过半数同学还没有掉队，仍然在学习、积极实践操作，并且热情地留下了大量的留言。</p><p>这些留言中，我非常高兴地看到，很多同学已经做到了活学活用，用学过的案例思路，分析出了线上应用的性能瓶颈，解决了实际工作中的性能问题。 还有同学能够反复推敲思考，指出文章中某些不当或不严谨的叙述，我也十分感谢你，同时很乐意和你探讨。</p><p>此外，很多留言提出的问题也很有价值，大部分我都已经在app里回复，一些手机上不方便回复的或者很有价值的典型问题，我专门摘了出来，作为今天的答疑内容，集中回复。另一方面，也是为了保证所有人都能不漏掉任何一个重点。</p><p>今天是性能优化答疑的第一期。为了便于你学习理解，它们并不是严格按照文章顺序排列的。每个问题，我都附上了留言区提问的截屏。如果你需要回顾内容原文，可以扫描每个问题右下方的二维码查看。</p><h2>问题1：性能工具版本太低，导致指标不全</h2><p><img src="https://static001.geekbang.org/resource/image/19/ba/19084718d4682168fea4bb6cb27c4fba.png?wh=750*747" alt=""></p><p>这是使用 CentOS 的同学普遍碰到的问题。在文章中，我的 pidstat 输出里有一个 %wait 指标，代表进程等待 CPU 的时间百分比，这是 systat 11.5.5 版本才引入的新指标，旧版本没有这一项。而CentOS 软件库里的 sysstat 版本刚好比这个低，所以没有这项指标。</p><!-- [[[read_end]]] --><p>不过，你也不用担心。前面我就强调过，工具只是查找分析的手段，指标才是我们重点分析的对象。如果你的pidstat 里没有显示，自然还有其他手段能找到这个指标。</p><p>比如说，在讲解系统原理和性能工具时，我一般会介绍一些 <strong>proc 文件系统</strong>的知识，教你看懂 proc 文件系统提供的各项指标。之所以这么做，一方面，当然是为了让你更直观地理解系统的工作原理；另一方面，其实是想给你展示，性能工具上能看到的各项性能指标的原始数据来源。</p><p>这样，在实际生产环境中，即使你很可能需要运行老版本的操作系统，还没有权限安装新的软件包，你也可以查看 proc 文件系统，获取自己想要的指标。</p><p>但是，性能分析的学习，我还是建议你要用最新的性能工具来学。新工具有更全面的指标，让你更容易上手分析。这个绝对的优势，可以让你更直观地得到想要的数据，也不容易让你打退堂鼓。</p><p>当然，初学时，你最好试着去理解性能工具的原理，或者熟悉了使用方法后，再回过头重新学习原理。这样，即使是在无法安装新工具的环境中，你仍然可以从 proc 文件系统或者其他地方，获得同样的指标，进行有效的分析。</p><h2>问题2：使用 stress 命令，无法模拟 iowait 高的场景</h2><p><img src="https://static001.geekbang.org/resource/image/34/43/34a354b22e351571e7f6a532e719fd43.png?wh=750*856" alt="">  <img src="https://static001.geekbang.org/resource/image/e7/c5/e7ffb84e4c22b08c0b2db14e2f61fdc5.jpg?wh=750*856" alt=""></p><p>使用 stress 无法模拟 iowait 升高，但是却看到了 sys 升高。这是因为案例中 的stress -i 参数，它表示通过系统调用 sync() 来模拟 I/O 的问题，但这种方法实际上并不可靠。</p><p>因为 sync() 的本意是刷新内存缓冲区的数据到磁盘中，以确保同步。如果缓冲区内本来就没多少数据，那读写到磁盘中的数据也就不多，也就没法产生 I/O 压力。</p><p>这一点，在使用 SSD 磁盘的环境中尤为明显，很可能你的 iowait 总是 0，却单纯因为大量的系统调用，导致了系统CPU使用率 sys 升高。</p><p>这种情况，我在留言中也回复过，推荐使用 stress-ng 来代替 stress。担心你没有看到留言，所以这里我再强调一遍。</p><p>你可以运行下面的命令，来模拟 iowait 的问题。</p><pre><code># -i的含义还是调用sync，而—hdd则表示读写临时文件
$ stress-ng -i 1 --hdd 1 --timeout 600
</code></pre><h2>问题3：无法模拟出 RES 中断的问题</h2><p><img src="https://static001.geekbang.org/resource/image/22/be/22d09f0924f7ae09a9dbcb7253b5b6be.jpg?wh=750*1053" alt=""></p><p>这个问题是说，即使运行了大量的线程，也无法模拟出重调度中断 RES 升高的问题。</p><p>其实我在 CPU 上下文切换的案例中已经提到，重调度中断是调度器用来分散任务到不同 CPU 的机制，也就是可以唤醒空闲状态的 CPU ，来调度新任务运行，而这通常借助<strong>处理器间中断</strong>（Inter-Processor Interrupts，IPI）来实现。</p><p>所以，这个中断在单核（只有一个逻辑 CPU）的机器上当然就没有意义了，因为压根儿就不会发生重调度的情况。</p><p>不过，正如留言所说，上下文切换的问题依然存在，所以你会看到， cs（context switch）从几百增加到十几万，同时 sysbench 线程的自愿上下文切换和非自愿上下文切换也都会大幅上升，特别是非自愿上下文切换，会上升到十几万。根据非自愿上下文的含义，我们都知道，这是过多的线程在争抢 CPU。</p><p>其实这个结论也可以从另一个角度获得。比如，你可以在 pidstat 的选项中，加入 -u 和 -t 参数，输出线程的 CPU 使用情况，你会看到下面的界面：</p><pre><code>$ pidstat -u -t 1

14:24:03      UID      TGID       TID    %usr %system  %guest   %wait    %CPU   CPU  Command
14:24:04        0         -      2472    0.99    8.91    0.00   77.23    9.90     0  |__sysbench
14:24:04        0         -      2473    0.99    8.91    0.00   68.32    9.90     0  |__sysbench
14:24:04        0         -      2474    0.99    7.92    0.00   75.25    8.91     0  |__sysbench
14:24:04        0         -      2475    2.97    6.93    0.00   70.30    9.90     0  |__sysbench
14:24:04        0         -      2476    2.97    6.93    0.00   68.32    9.90     0  |__sysbench
...
</code></pre><p>从这个 pidstat 的输出界面，你可以发现，每个 stress 线程的 %wait 高达 70%，而 CPU 使用率只有不到 10%。换句话说， stress 线程大部分时间都消耗在了等待 CPU 上，这也表明，确实是过多的线程在争抢 CPU。</p><p>在这里顺便提一下，留言中很常见的一个错误。有些同学会拿 pidstat 中的 %wait 跟 top 中的 iowait% （缩写为wa）对比，其实这是没有意义的，因为它们是完全不相关的两个指标。</p><ul>
<li>
<p>pidstat 中， %wait 表示进程等待 CPU 的时间百分比。</p>
</li>
<li>
<p>top 中 ，iowait% 则表示等待 I/O 的 CPU 时间百分比。</p>
</li>
</ul><p>回忆一下我们学过的进程状态，你应该记得，等待 CPU 的进程已经在 CPU 的就绪队列中，处于运行状态；而等待 I/O 的进程则处于不可中断状态。</p><p>另外，不同版本的 sysbench 运行参数也不是完全一样的。比如，在案例 Ubuntu 18.04 中，运行 sysbench 的格式为：</p><pre><code>$ sysbench --threads=10 --max-time=300 threads run
</code></pre><p>而在 Ubuntu 16.04 中，运行格式则为（感谢 Haku 留言分享的执行命令）：</p><pre><code>$ sysbench --num-threads=10 --max-time=300 --test=threads run
</code></pre><h2>问题4：无法模拟出I/O性能瓶颈，以及I/O压力过大的问题</h2><p><img src="https://static001.geekbang.org/resource/image/9e/d8/9e235aca4e92b68e84dba03881c591d8.png?wh=750*1533" alt=""></p><p>这个问题可以看成是上一个问题的延伸，只是把 stress 命令换成了一个在容器中运行的 app 应用。</p><p>事实上，在 I/O 瓶颈案例中，除了上面这个模拟不成功的留言，还有更多留言的内容刚好相反，说的是案例 I/O 压力过大，导致自己的机器出各种问题，甚至连系统都没响应了。</p><p>之所以这样，其实还是因为每个人的机器配置不同，既包括了 CPU 和内存配置的不同，更是因为磁盘的巨大差异。比如，机械磁盘（HDD）、低端固态磁盘（SSD）与高端固态磁盘相比，性能差异可能达到数倍到数十倍。</p><p>其实，我自己所用的案例机器也只是低端的 SSD，比机械磁盘稍微好一些，但跟高端固态磁盘还是比不了的。所以，相同操作下，我的机器上刚好出现 I/O 瓶颈，但换成一台使用机械磁盘的机器，可能磁盘 I/O 就被压死了（表现为使用率长时间100%），而换上好一些的 SSD 磁盘，可能又无法产生足够的 I/O 压力。</p><p>另外，由于我在案例中只查找了 /dev/xvd 和 /dev/sd 前缀的磁盘，而没有考虑到使用其他前缀磁盘（比如 /dev/nvme）的同学。如果你正好用的是其他前缀，你可能会碰到跟Vicky 类似的问题，也就是app 启动后又很快退出，变成 exited 状态。</p><p><img src="https://static001.geekbang.org/resource/image/a3/38/a30211eeb41194eb9b5aa193cda25238.png?wh=900*1359" alt=""></p><p>在这里，berryfl 同学提供了一个不错的建议：可以在案例中增加一个参数指定块设备，这样有需要的同学就不用自己编译和打包案例应用了。</p><p><img src="https://static001.geekbang.org/resource/image/f3/2c/f351f346cbfc2b3c35d010536b23332c.png?wh=900*1050" alt=""></p><p>所以，在最新的案例中，我为 app 应用增加了三个选项。</p><ul>
<li>
<p>-d 设置要读取的磁盘，默认前缀为 <code>/dev/sd</code> 或者 <code>/dev/xvd</code> 的磁盘。</p>
</li>
<li>
<p>-s 设置每次读取的数据量大小，单位为字节，默认为 67108864（也就是 64MB）。</p>
</li>
<li>
<p>-c 设置每个子进程读取的次数，默认为 20 次，也就是说，读取 20*64MB 数据后，子进程退出。</p>
</li>
</ul><p>你可以点击 <a href="https://github.com/feiskyer/linux-perf-examples/tree/master/high-iowait-process">Github</a> 查看它的源码，使用方法我写在了这里：</p><pre><code>$ docker run --privileged --name=app -itd feisky/app:iowait /app -d /dev/sdb -s 67108864 -c 20
</code></pre><p>案例运行后，你可以执行 docker logs 查看它的日志。正常情况下，你可以看到下面的输出：</p><pre><code>$ docker logs app
Reading data from disk /dev/sdb with buffer size 67108864 and count 20
</code></pre><h2>问题5：性能工具（如 vmstat）输出中，第一行数据跟其他行差别巨大</h2><p><img src="https://static001.geekbang.org/resource/image/ef/0f/efa8186b71c474bd40924a9038016e0f.png?wh=900*1404" alt=""></p><p>这个问题主要是说，在执行 vmstat 时，第一行数据跟其他行相比较，数值相差特别大。我相信不少同学都注意到了这个现象，这里我简单解释一下。</p><p>首先还是要记住，我总强调的那句话，<strong>在碰到直观上解释不了的现象时，要第一时间去查命令手册</strong>。</p><p>比如，运行 man vmstat 命令，你可以在手册中发现下面这句话：</p><pre><code>The first report produced gives averages since the last reboot.  Additional reports give information on a sam‐ 
pling period of length delay.  The process and memory reports are instantaneous in either case. 
</code></pre><p>也就是说，第一行数据是系统启动以来的平均值，其他行才是你在运行 vmstat 命令时，设置的间隔时间的平均值。另外，进程和内存的报告内容都是即时数值。</p><p>你看，这并不是什么不得了的事故，但如果我们不清楚这一点，很可能卡住我们的思维，阻止我们进一步的分析。这里我也不得不提一下，文档的重要作用。</p><p>授之以鱼，不如授之以渔。我们专栏的学习核心，一定是教会你<strong>性能分析的原理和思路</strong>，性能工具只是我们的路径和手段。所以，在提到各种性能工具时，我并没有详细解释每个工具的各种命令行选项的作用，一方面是因为你很容易通过文档查到这些，另一方面就是不同版本、不同系统中，个别选项的含义可能并不相同。</p><p>所以，不管因为哪个因素，自己man一下，一定是最快速并且最准确的方式。特别是，当你发现某些工具的输出不符合常识时，一定记住，第一时间查文档弄明白。实在读不懂文档的话，再上网去搜，或者在专栏里向我提问。</p><p>学习是一个“从薄到厚再变薄”的过程，我们从细节知识入手开始学习，积累到一定程度，需要整理成一个体系来记忆，这其中还要不断地对这个体系进行细节修补。有疑问、有反思才可以达到最佳的学习效果。</p><p>最后，欢迎继续在留言区写下你的疑问，我会持续不断地解答。我的目的仍然不变，希望可以和你一起，把文章的知识变成你的能力，我们不仅仅在实战中演练，也要在交流中进步。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://wx.qlogo.cn/mmopen/vi_32/PiajxSqBRaEKQMM4m7NHuicr55aRiblTSEWIYe0QqbpyHweaoAbG7j2v7UUElqqeP3Ihrm3UfDPDRb1Hv8LvPwXqA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ninuxer</span>
  </div>
  <div class="_2_QraFYR_0">打卡day14<br>之前一直理解有误，感谢指出！<br>pidstat 中， %wait 表示进程等待 CPU 的时间百分比。此时进程是运行状态。<br>top 中 ，iowait% 则表示等待 I&#47;O 的 CPU 时间百分比。此时进程处于不可中断睡眠态。<br>等待 CPU 的进程已经在 CPU 的就绪队列中，处于运行状态；而等待 I&#47;O 的进程则处于不可中断状态。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-19 08:17:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/32/0b/981b4e93.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>念你如昔</span>
  </div>
  <div class="_2_QraFYR_0">非常非常感谢，这钱花的值，之前没有对这些东西形成体系，老是感觉有力使不上的感觉，自从看了老师的文档，终于飘了，都想跳槽了？！。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 😢</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-19 10:27:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/0c/ca/6173350b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>郭江伟</span>
  </div>
  <div class="_2_QraFYR_0">课程很系统，把自己以前的知识都串起来了，后续争取每个案例自己都做一次，并且融合自己的经验改进下案例</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 欢迎分享你的改进经验😊</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-19 11:38:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/53/3d/1189e48a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>微思</span>
  </div>
  <div class="_2_QraFYR_0">老师，等待IO的不可中断进程是否一直占用CPU？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不会，有其他需要CPU的进程运行时会切换过去</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-19 21:04:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/12/b3/9e8f4b4f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>MoFanDon</span>
  </div>
  <div class="_2_QraFYR_0">做了几年运维一直想要掌握，却了解的很零散。这段时间的课程让我学习很多，感谢老师。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 😊 </p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-19 00:27:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/e0/99/5d603697.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>MJ</span>
  </div>
  <div class="_2_QraFYR_0">虽然有很多地方不太懂（不熟Linux原理），但还是要跟下来，多过几遍。<br>同时，私下学Linux原理</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-28 07:33:03</div>
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
  <div class="_2_QraFYR_0">[D13打卡]<br>多谢老师提出来, pidstat 和 top 中的 %wait 含义并不一样.<br>之前只知道top是io的wait, 而新接触的pidstat的倒没有细想过.<br>确实是应该多man一下,看下命令文档.<br>刚开始要把工具用起来, 之后再查看命令的详细文档.</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯嗯，虽然专栏里也有不少使用案例，但并能包括所有细节的知识，这都需要查文档</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-19 09:26:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/99/af/d29273e2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>饭粒</span>
  </div>
  <div class="_2_QraFYR_0">测试跑的Q4 还是无法模拟出 I&#47;O 性能瓶颈，依然是要加了 --device-write-iops --device-read-iops 限制后才可以。<br><br>[root@centos-7 ~]# docker run --privileged --name=app -itd feisky&#47;app:iowait &#47;app -d &#47;dev&#47;sda -s 67108864 -c 20<br>e7a7deddba3d2845030515fbbe25785382306d3c600c474a22de54d543a2cf53<br>[root@centos-7 ~]# docker logs app<br>Reading data from disk &#47;dev&#47;sdb with buffer size 67108864 and count 20<br><br># 内核cpu时间sy，软中断cpu时间si升高明显<br>top - 00:15:15 up 12 min,  2 users,  load average: 23.50, 7.07, 2.49<br>Tasks: 148 total,  43 running, 104 sleeping,   1 stopped,   0 zombie<br>%Cpu0  : 21.1 us, 54.9 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi, 24.0 si,  0.0 st<br>%Cpu1  : 21.8 us, 76.8 sy,  0.0 ni,  0.0 id,  0.0 wa,  0.0 hi,  1.4 si,  0.0 st<br>KiB Mem :  3880792 total,   737592 free,  2880084 used,   263116 buff&#47;cache<br>KiB Swap:        0 total,        0 free,        0 used.   764308 avail Mem <br><br># docker run --privileged --name=app -itd --device-write-iops &#47;dev&#47;sda:3 --device-read-iops &#47;dev&#47;sda:3 feisky&#47;app:iowait &#47;app -d &#47;dev&#47;sda -s 67108864 -c </div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-27 00:31:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/46/1d/83021ef5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>大哈</span>
  </div>
  <div class="_2_QraFYR_0">老师，您好，我想知道像pidstat这样的工具是怎么获取到这些性能数据的，我自己去开发应该从那里获取呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: man 查询工具手册，一般都有介绍。一般数据都是来自 &#47;proc、&#47;sys 等</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-10 12:32:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83eqcQWfsAka5Y9wVIKTKuN1173m9N5MyiadEJ2mdaUv9eHreg2oOYnGKsib9xvLtQ2cdxTKicwF351eTQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>胡莉婷</span>
  </div>
  <div class="_2_QraFYR_0">补充一点。我们的进程都是后台服务，机器1和机器2启动后都没有业务进来</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯 我的第一反应是负载不均匀，后台处理的业务是均匀的吗？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-08 16:58:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/d8/ee/6e7c2264.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Only now</span>
  </div>
  <div class="_2_QraFYR_0">mark </div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-27 22:35:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/00/53/b8ee8918.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>灬 黑 礼服 ~</span>
  </div>
  <div class="_2_QraFYR_0">老师 。。我们这边系统多数都是java 语言开发的。。。也有些系统经常出现cpu暴涨的时候。。。都是根据网上 jstack那种工具 去分析。。。对于java而言？？ 有没有更好的 查找cpu暴涨的时候的信息？ 具体什么原因引起的 ？<br>      同时 ，根据jdk自带的检查工具 jstack之类的  都是瞬时的。。。有没有 连续监控排查的 工具？？有效的查找 java语言 开发的系统引起的cpu暴涨？？？？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-24 15:44:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTIDqHQQByGiaXcAk94MdDn3ftupZLXyR6bAKibxOzMxy5h3uBwZ7QiaCiaIfbCMK0cIQfGNax8iawoiaQAg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>nuan</span>
  </div>
  <div class="_2_QraFYR_0">期待老师出个新版教程！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-18 09:58:07</div>
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
  <div class="_2_QraFYR_0">day13<br>授之以鱼，不如授之以渔</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-03 14:23:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/93/38/71615300.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>DayDayUp</span>
  </div>
  <div class="_2_QraFYR_0">老师，一个机器有很多D state的进程，然后导致机器很慢，有时间讲一下关于“D进程太多的相关诊断”章节吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-28 16:17:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ch_ort</span>
  </div>
  <div class="_2_QraFYR_0">“pidstat 中， %wait 表示进程等待 CPU 的时间百分比。<br> top 中 ，iowait% 则表示等待 I&#47;O 的 CPU 时间百分比。<br>”<br>问题来了，现在通过top和mpstat发现是iowati高导致的，怎么通过pidstat找到iowait高的进程呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-12 08:04:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/63/3b/c5cd68ab.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>生活不如诗</span>
  </div>
  <div class="_2_QraFYR_0">一直以来理解错误的几个误区：<br>1.pidstat 中， %wait 表示进程等待 CPU 的时间百分比。<br>2.top 中 ，iowait% 则表示等待 I&#47;O 的 CPU 时间百分比。<br>3.vmstat命令中第一行数据是系统启动以来的平均值，其他行才是你在运行 vmstat 命令时，设置的间隔时间的平均值。另外，进程和内存的报告内容都是即时数值		<br>4.还是得学会通过man命令查看手册帮助；</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-01 14:19:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://wx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTL3OaibxsOia5ZET1iaBsPwDM6NS43lAUTdItqkwZ66fGaaXtjOQYL73IsvY0foscUZlkaZSQPPQNexA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Yong</span>
  </div>
  <div class="_2_QraFYR_0">虚拟机装的centos7系统，只有2个cpu,大部分命令cpu个数都正常，但cat &#47;proc&#47;softirqs 显示出来的cpu有120个，有同学知道我什么么.<br>搜到的一篇相关的文章proc-softirqs-the-number-of-columns-doesnt-match-the-number-of-cpu-cores 但页面已经不在了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-17 22:48:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/e6/ee/8fdbd5db.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Damoncui</span>
  </div>
  <div class="_2_QraFYR_0">有一个问题 现在系统核心太多。比如256逻辑线程这种级别 通过 &#47;proc查看中断特别不方便。能不能指定核心啊？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-15 22:13:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTK0K9AM6xxDzVV6pF66jyus5NuuxZzT9icad8AQDMKibwUOy3UnoZIZdyKIKd9sA06rgFnIWwiakSeOQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>斑马Z</span>
  </div>
  <div class="_2_QraFYR_0">孜孜不倦，加油！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-27 09:16:35</div>
  </div>
</div>
</div>
</li>
</ul>