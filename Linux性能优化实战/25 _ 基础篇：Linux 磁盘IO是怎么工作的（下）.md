<audio title="25 _ 基础篇：Linux 磁盘IO是怎么工作的（下）" src="https://static001.geekbang.org/resource/audio/74/14/7441a367d3701e7b5897132699baf314.mp3" controls="controls"></audio> 
<p>你好，我是倪朋飞。</p><p>上一节我们学习了 Linux 磁盘 I/O 的工作原理，并了解了由文件系统层、通用块层和设备层构成的 Linux 存储系统 I/O 栈。</p><p>其中，通用块层是 Linux 磁盘 I/O 的核心。向上，它为文件系统和应用程序，提供访问了块设备的标准接口；向下，把各种异构的磁盘设备，抽象为统一的块设备，并会对文件系统和应用程序发来的 I/O 请求，进行重新排序、请求合并等，提高了磁盘访问的效率。</p><p>掌握了磁盘 I/O 的工作原理，你估计迫不及待想知道，怎么才能衡量磁盘的 I/O 性能。</p><p>接下来，我们就来看看，磁盘的性能指标，以及观测这些指标的方法。</p><h2>磁盘性能指标</h2><p>说到磁盘性能的衡量标准，必须要提到五个常见指标，也就是我们经常用到的，使用率、饱和度、IOPS、吞吐量以及响应时间等。这五个指标，是衡量磁盘性能的基本指标。</p><ul>
<li>
<p>使用率，是指磁盘处理I/O的时间百分比。过高的使用率（比如超过80%），通常意味着磁盘 I/O 存在性能瓶颈。</p>
</li>
<li>
<p>饱和度，是指磁盘处理 I/O 的繁忙程度。过高的饱和度，意味着磁盘存在严重的性能瓶颈。当饱和度为 100% 时，磁盘无法接受新的 I/O 请求。</p>
</li>
<li>
<p>IOPS（Input/Output Per Second），是指每秒的 I/O 请求数。</p>
</li>
<li>
<p>吞吐量，是指每秒的 I/O 请求大小。</p>
</li>
<li>
<p>响应时间，是指 I/O 请求从发出到收到响应的间隔时间。</p>
</li>
</ul><!-- [[[read_end]]] --><p>这里要注意的是，使用率只考虑有没有 I/O，而不考虑 I/O 的大小。换句话说，当使用率是 100% 的时候，磁盘依然有可能接受新的 I/O 请求。</p><p>这些指标，很可能是你经常挂在嘴边的，一讨论磁盘性能必定提起的对象。不过我还是要强调一点，不要孤立地去比较某一指标，而要结合读写比例、I/O类型（随机还是连续）以及 I/O 的大小，综合来分析。</p><p>举个例子，在数据库、大量小文件等这类随机读写比较多的场景中，IOPS 更能反映系统的整体性能；而在多媒体等顺序读写较多的场景中，吞吐量才更能反映系统的整体性能。</p><p>一般来说，我们在为应用程序的服务器选型时，要先对磁盘的 I/O 性能进行基准测试，以便可以准确评估，磁盘性能是否可以满足应用程序的需求。</p><p>这一方面，我推荐用性能测试工具 fio ，来测试磁盘的IOPS、吞吐量以及响应时间等核心指标。但还是那句话，因地制宜，灵活选取。在基准测试时，一定要注意根据应用程序 I/O 的特点，来具体评估指标。</p><p>当然，这就需要你测试出，不同 I/O 大小（一般是 512B 至 1MB 中间的若干值）分别在随机读、顺序读、随机写、顺序写等各种场景下的性能情况。</p><p>用性能工具得到的这些指标，可以作为后续分析应用程序性能的依据。一旦发生性能问题，你就可以把它们作为磁盘性能的极限值，进而评估磁盘 I/O 的使用情况。</p><p>了解磁盘的性能指标，只是我们I/O性能测试的第一步。接下来，又该用什么方法来观测它们呢？这里，我给你介绍几个常用的I/O性能观测方法。</p><h2><strong>磁盘I/O观测</strong></h2><p>第一个要观测的，是每块磁盘的使用情况。</p><p>iostat 是最常用的磁盘I/O性能观测工具，它提供了每个磁盘的使用率、IOPS、吞吐量等各种常见的性能指标，当然，这些指标实际上来自  /proc/diskstats。</p><p>iostat 的输出界面如下。</p><pre><code># -d -x表示显示所有磁盘I/O的指标
$ iostat -d -x 1 
Device            r/s     w/s     rkB/s     wkB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz  svctm  %util 
loop0            0.00    0.00      0.00      0.00     0.00     0.00   0.00   0.00    0.00    0.00   0.00     0.00     0.00   0.00   0.00 
loop1            0.00    0.00      0.00      0.00     0.00     0.00   0.00   0.00    0.00    0.00   0.00     0.00     0.00   0.00   0.00 
sda              0.00    0.00      0.00      0.00     0.00     0.00   0.00   0.00    0.00    0.00   0.00     0.00     0.00   0.00   0.00 
sdb              0.00    0.00      0.00      0.00     0.00     0.00   0.00   0.00    0.00    0.00   0.00     0.00     0.00   0.00   0.00 
</code></pre><p>从这里你可以看到，iostat 提供了非常丰富的性能指标。第一列的 Device 表示磁盘设备的名字，其他各列指标，虽然数量较多，但是每个指标的含义都很重要。为了方便你理解，我把它们总结成了一个表格。</p><p><img src="https://static001.geekbang.org/resource/image/cf/8d/cff31e715af51c9cb8085ce1bb48318d.png?wh=1712*1791" alt=""></p><p>这些指标中，你要注意：</p><ul>
<li>
<p>%util  ，就是我们前面提到的磁盘I/O使用率；</p>
</li>
<li>
<p>r/s+  w/s  ，就是 IOPS；</p>
</li>
<li>
<p>rkB/s+wkB/s ，就是吞吐量；</p>
</li>
<li>
<p>r_await+w_await ，就是响应时间。</p>
</li>
</ul><p>在观测指标时，也别忘了结合请求的大小（ rareq-sz 和wareq-sz）一起分析。</p><p>你可能注意到，从 iostat 并不能直接得到磁盘饱和度。事实上，饱和度通常也没有其他简单的观测方法，不过，你可以把观测到的，平均请求队列长度或者读写请求完成的等待时间，跟基准测试的结果（比如通过 fio）进行对比，综合评估磁盘的饱和情况。</p><h2><strong>进程I/O观测</strong></h2><p>除了每块磁盘的 I/O 情况，每个进程的 I/O 情况也是我们需要关注的重点。</p><p>上面提到的 iostat 只提供磁盘整体的 I/O 性能数据，缺点在于，并不能知道具体是哪些进程在进行磁盘读写。要观察进程的I/O情况，你还可以使用 pidstat 和 iotop 这两个工具。</p><p>pidstat 是我们的老朋友了，这里我就不再啰嗦它的功能了。给它加上 -d 参数，你就可以看到进程的I/O情况，如下所示：</p><pre><code>$ pidstat -d 1 
13:39:51      UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command 
13:39:52      102       916      0.00      4.00      0.00       0  rsyslogd
</code></pre><p>从pidstat的输出你能看到，它可以实时查看每个进程的I/O情况，包括下面这些内容。</p><ul>
<li>
<p>用户ID（UID）和进程ID（PID）  。</p>
</li>
<li>
<p>每秒读取的数据大小（kB_rd/s） ，单位是 KB。</p>
</li>
<li>
<p>每秒发出的写请求数据大小（kB_wr/s） ，单位是 KB。</p>
</li>
<li>
<p>每秒取消的写请求数据大小（kB_ccwr/s） ，单位是 KB。</p>
</li>
<li>
<p>块I/O延迟（iodelay），包括等待同步块I/O和换入块I/O结束的时间，单位是时钟周期。</p>
</li>
</ul><p>除了可以用 pidstat 实时查看，根据 I/O 大小对进程排序，也是性能分析中一个常用的方法。这一点，我推荐另一个工具， iotop。它是一个类似于 top 的工具，你可以按照 I/O 大小对进程排序，然后找到I/O较大的那些进程。</p><p>iotop 的输出如下所示：</p><pre><code>$ iotop
Total DISK READ :       0.00 B/s | Total DISK WRITE :       7.85 K/s 
Actual DISK READ:       0.00 B/s | Actual DISK WRITE:       0.00 B/s 
  TID  PRIO  USER     DISK READ  DISK WRITE  SWAPIN     IO&gt;    COMMAND 
15055 be/3 root        0.00 B/s    7.85 K/s  0.00 %  0.00 % systemd-journald 
</code></pre><p>从这个输出，你可以看到，前两行分别表示，进程的磁盘读写大小总数和磁盘真实的读写大小总数。因为缓存、缓冲区、I/O合并等因素的影响，它们可能并不相等。</p><p>剩下的部分，则是从各个角度来分别表示进程的I/O情况，包括线程ID、I/O优先级、每秒读磁盘的大小、每秒写磁盘的大小、换入和等待I/O的时钟百分比等。</p><p>这两个工具，是我们分析磁盘 I/O 性能时最常用到的。你先了解它们的功能和指标含义，具体的使用方法，接下来的案例实战中我们一起学习。</p><h2>小结</h2><p>今天，我们梳理了 Linux 磁盘 I/O 的性能指标和性能工具。我们通常用IOPS、吞吐量、使用率、饱和度以及响应时间等几个指标，来评估磁盘的 I/O 性能。</p><p>你可以用 iostat 获得磁盘的 I/O 情况，也可以用 pidstat、iotop 等观察进程的 I/O 情况。不过在分析这些性能指标时，你要注意结合读写比例、I/O 类型以及 I/O 大小等，进行综合分析。</p><h2>思考</h2><p>最后，我想请你一起来聊聊，你碰到过的磁盘 I/O 问题。在碰到磁盘 I/O 性能问题时，你是怎么分析和定位的呢？你可以结合今天学到的磁盘 I/O 指标和工具，以及上一节学过的磁盘 I/O 原理，来总结你的思路。</p><p>欢迎在留言区和我讨论，也欢迎把这篇文章分享给你的同事、朋友。我们一起在实战中演练，在交流中进步。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/54/9a/76c0af70.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>每天晒白牙</span>
  </div>
  <div class="_2_QraFYR_0">【D25打卡】<br>总结：<br>磁盘性能检测指标：<br>使用率：磁盘处理I&#47;O的时间百分比，使用率只考虑有没有I&#47;O，不考虑I&#47;O的大小。注意当使用率为100%时，由于可能存在并行I&#47;O，磁盘并不一定饱和，所以磁盘仍然可能接收新的I&#47;O请求<br>饱和度：磁盘处理I&#47;O的繁忙程度，注意当饱和度为100%时，磁盘不能接收新的I&#47;O请求<br>吞吐量：每秒I&#47;O请求大小<br>IOPS：Input&#47;Output Per Second 每秒的I&#47;O请求数<br>响应时间：I&#47;O请求从发出到收到响应的间隔时间<br><br>不孤立比较某项指标，结合读写比例、I&#47;O类型（随机还是连续）以及I&#47;O大小综合分析<br>例如：随机读写：多关注IOPS<br>          连续读写：多关注吞吐量<br><br>服务器选型时，对磁盘I&#47;O性能进行基础测试，使用 fio<br>磁盘I&#47;O观测：iostat<br>进程I&#47;O观测：pidstat,iotop<br>指导：遇到I&#47;O性能时，先通过iostat查看磁盘整体性能，然后用pidstat或iotop定位到具体的进程<br><br>疑惑：<br>对磁盘的使用率和饱和度还是没太理解，比如说磁盘的使用率达到100%，由于并行I&#47;O，不一定饱和了，所以还可能接收新的I&#47;O请求，还希望老师再指点下。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 使用率是从时间角度衡量I&#47;O，但是磁盘还可以支持并行写，所以即使使用率100%，有可能还可以接收新的I&#47;O（不饱和）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-17 08:26:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/9a/64/ad837224.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Christmas</span>
  </div>
  <div class="_2_QraFYR_0">一趟调度法，电梯调度法等调度是发生在磁盘控制器硬件上的吗？通用块层的调度是os级别的对吧？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-16 10:33:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/e6/ee/e3c4c9b3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Cranliu</span>
  </div>
  <div class="_2_QraFYR_0">关于磁盘的饱和度，有没有经验值可以参考下呢？谢谢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 饱和度一般没法直接观测到，所以一般是通过实际观测值跟基准测试结果对比来分析</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-16 08:00:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://wx.qlogo.cn/mmopen/vi_32/PiajxSqBRaEKQMM4m7NHuicr55aRiblTSEWIYe0QqbpyHweaoAbG7j2v7UUElqqeP3Ihrm3UfDPDRb1Hv8LvPwXqA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ninuxer</span>
  </div>
  <div class="_2_QraFYR_0">day26打卡<br>之前都没用过fio测试磁盘实际性能，基本都是依赖磁盘型号查官网数据作为依据～<br>iostat和iotop倒是会经常用，之前有几列输出的内容自己理解有偏差，这下算是纠正过来了💪</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-16 08:17:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/65/ae/9727318e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Boy-struggle</span>
  </div>
  <div class="_2_QraFYR_0">老师，如何根据系统调用判断IO为随机还是顺序，IO 的位置怎么体现，希望老师可以结合案例具体讲解一下，多谢！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 最简单的方法是根据系统调用判断I&#47;O读写的相对位置</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-03 08:57:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/0f/84/d8e63885.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>仲鬼</span>
  </div>
  <div class="_2_QraFYR_0">&quot;r_await+w_await ，就是响应时间&quot;<br>对这句表述有怀疑。<br>r_await、w_await分别是读、写请求的平均等待时间，二者相加什么都不是。因为a&#47;b + c&#47;d不等于(a+c)&#47;(b+d)。<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 从公式上是这样，但间隔时间相同的时候呢？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-18 13:32:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/1e/b7/b20ab184.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>麋鹿在泛舟</span>
  </div>
  <div class="_2_QraFYR_0">仲鬼<br>2019-01-25<br><br>2<br>&quot;r_await+w_await ，就是响应时间&quot;<br>对这句表述有怀疑。<br>r_await、w_await分别是读、写请求的平均等待时间，二者相加什么都不是。因为a&#47;b + c&#47;d不等于(a+c)&#47;(b+d)。<br>展开<br>作者回复: 从公式上是这样，但间隔时间相同的时候呢？<br><br>man手册解释await是平均等待时间，我理解意思是toal wait time &#47; total req number，跟间隔时间无关<br>-----------------------------------------------<br>&quot;r_await、w_await分别是读、写请求的平均等待时间&quot;基于读写的平均等待时间没错，但是结果也是基于一定的时间范围内的，比如说过去1s，过去5s，显然间隔时间无论设置成多少，都是一样的.<br>即a&#47;t + b&#47;t = (a+b)&#47;t</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-10 21:11:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/0f/84/d8e63885.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>仲鬼</span>
  </div>
  <div class="_2_QraFYR_0">&quot;r_await+w_await ，就是响应时间&quot;<br>对这句表述有怀疑。<br>r_await、w_await分别是读、写请求的平均等待时间，二者相加什么都不是。因为a&#47;b + c&#47;d不等于(a+c)&#47;(b+d)。<br>展开<br>作者回复: 从公式上是这样，但间隔时间相同的时候呢？<br><br>man手册解释await是平均等待时间，我理解意思是toal wait time &#47; total req number，跟间隔时间无关</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-25 11:37:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/f0/27/6679da14.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>程序员历小冰</span>
  </div>
  <div class="_2_QraFYR_0">请问作者对《性能之垫-洞悉系统、企业和云计算》这本书的看法？适合作为工具书，用于查阅；还是可以进行通篇学习</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 建议学习一下各个章节的基本原理和思路，剩下的工具部分作为手册参考。不过有些工具过时了，使用的时候要注意</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-21 18:45:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/25/00/3afbab43.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>88591</span>
  </div>
  <div class="_2_QraFYR_0">老师 ，应用程序可以控制磁盘的顺序写吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-15 16:21:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_e2b0f9</span>
  </div>
  <div class="_2_QraFYR_0">怎么分析io类型 是连续还是随机</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-07 09:06:16</div>
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
  <div class="_2_QraFYR_0">请问aqu-sz指的是通用块层中的队列长度吗?</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-08 00:02:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/50/01/128bdef2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Black🐯</span>
  </div>
  <div class="_2_QraFYR_0">使用率，是指磁盘处理 I&#47;O 的时间百分比。过高的使用率（比如超过 80%），通常意味着磁盘 I&#47;O 存在性能瓶颈。<br>%util ，就是我们前面提到的磁盘 I&#47;O 使用率；<br>---------------------------------------<br>man手册对%util的解释：Percentage  of elapsed time during which I&#47;O requests were issued to the device (bandwidth utilization for the device). Device saturation occurs when this value is close to 100%.<br>向设备发出I&#47;O请求所用的时间百分比（设备的带宽利用率）。当该值接近100%时，设备饱和。<br>换成公式的话，就是： 请求时间&#47;（请求时间+io处理时间）</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-17 11:32:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/50/01/128bdef2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Black🐯</span>
  </div>
  <div class="_2_QraFYR_0">%util ，就是我们前面提到的磁盘 I&#47;O 使用率；<br>-----------------------------<br>man手册对%util的解释</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-17 11:30:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/21/5f/dc/3923dc67.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>蓝胖子的编程梦</span>
  </div>
  <div class="_2_QraFYR_0">在用k8s里面搭建prometheus operator，自带了面板用了node_disk_io_time_weighted_seconds:rate1m去观察磁盘饱和度，这个值是准确的吗</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-30 11:11:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/VX0ib4CV0m7fwxB2xFcIJaYYWXXpfxxYbfBErqBej9395hgZszqS3dz9bThCxOuFfJ8Xibx9HbdNmZJwL5m33wIw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>chaoxifuchen</span>
  </div>
  <div class="_2_QraFYR_0">现在业务应用程序对数据的操作有数据库，缓存中间件，一般只有日志直接涉及io读写，对io性能比较敏感的主要有数据库，es,prometheus等应用</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-08 22:49:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/99/7a/558666a5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>AceslupK</span>
  </div>
  <div class="_2_QraFYR_0">IOPS vs 吞吐量 <br>每秒IO请求数，每秒IO请求大小，这该咋区分</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-02 19:16:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/f0/17/796a3d20.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>言十年</span>
  </div>
  <div class="_2_QraFYR_0">工作多年，没有关注过IO这个问题。<br>观测命令。到是让我觉得，可以看一个进程打日志的速率。评估下磁盘容量了。或者优化程序中没必要的日志打印。<br>还有看打日志多的进程。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-29 17:54:33</div>
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
  <div class="_2_QraFYR_0">老师您好，我请假一个磁盘性能问题<br>我在压测Kafka写性能的时候发现如下问题<br>物理机挂载多块HDD盘，应用程序采用 内存映射的方式来写文件<br>现象：当随机性越强，多块盘的utils使用率会下降的很明显，增加客户端去压也压不上去<br>个人感觉瓶颈出在OS将数据从PageCache刷到磁盘，导致应用程序写PageCache会阻塞<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-09 17:21:19</div>
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
  <div class="_2_QraFYR_0">老师请问下：网络存储在创建目录和删除文件时比较慢，如何定位呀？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-16 07:23:18</div>
  </div>
</div>
</div>
</li>
</ul>