<audio title="09 _ 基础篇：怎么理解Linux软中断？" src="https://static001.geekbang.org/resource/audio/61/bb/61b8d75070507d09726fafc5999521bb.mp3" controls="controls"></audio> 
<p>你好，我是倪朋飞。</p><p>上一期，我用一个不可中断进程的案例，带你学习了iowait（也就是等待I/O的CPU使用率）升高时的分析方法。这里你要记住，进程的不可中断状态是系统的一种保护机制，可以保证硬件的交互过程不被意外打断。所以，短时间的不可中断状态是很正常的。</p><p>但是，当进程长时间都处于不可中断状态时，你就得当心了。这时，你可以使用 dstat、pidstat 等工具，确认是不是磁盘 I/O 的问题，进而排查相关的进程和磁盘设备。关于磁盘 I/O 的性能问题，你暂且不用专门去背，我会在后续的 I/O 部分详细介绍，到时候理解了也就记住了。</p><p>其实除了 iowait，软中断（softirq）CPU使用率升高也是最常见的一种性能问题。接下来的两节课，我们就来学习软中断的内容，我还会以最常见的反向代理服务器 Nginx 的案例，带你分析这种情况。</p><h2>从“取外卖”看中断</h2><p>说到中断，我在前面<a href="https://time.geekbang.org/column/article/69859">关于“上下文切换”的文章</a>，简单说过中断的含义，先来回顾一下。中断是系统用来响应硬件设备请求的一种机制，它会打断进程的正常调度和执行，然后调用内核中的中断处理程序来响应设备的请求。</p><p>你可能要问了，为什么要有中断呢？我可以举个生活中的例子，让你感受一下中断的魅力。</p><!-- [[[read_end]]] --><p>比如说你订了一份外卖，但是不确定外卖什么时候送到，也没有别的方法了解外卖的进度，但是，配送员送外卖是不等人的，到了你这儿没人取的话，就直接走人了。所以你只能苦苦等着，时不时去门口看看外卖送到没，而不能干其他事情。</p><p>不过呢，如果在订外卖的时候，你就跟配送员约定好，让他送到后给你打个电话，那你就不用苦苦等待了，就可以去忙别的事情，直到电话一响，接电话、取外卖就可以了。</p><p>这里的“打电话”，其实就是一个中断。没接到电话的时候，你可以做其他的事情；只有接到了电话（也就是发生中断），你才要进行另一个动作：取外卖。</p><p>这个例子你就可以发现，<strong>中断其实是一种异步的事件处理机制，可以提高系统的并发处理能力</strong>。</p><p>由于中断处理程序会打断其他进程的运行，所以，<strong>为了减少对正常进程运行调度的影响，中断处理程序就需要尽可能快地运行</strong>。如果中断本身要做的事情不多，那么处理起来也不会有太大问题；但如果中断要处理的事情很多，中断服务程序就有可能要运行很长时间。</p><p>特别是，中断处理程序在响应中断时，还会临时关闭中断。这就会导致上一次中断处理完成之前，其他中断都不能响应，也就是说中断有可能会丢失。</p><p>那么还是以取外卖为例。假如你订了 2 份外卖，一份主食和一份饮料，并且是由2个不同的配送员来配送。这次你不用时时等待着，两份外卖都约定了电话取外卖的方式。但是，问题又来了。</p><p>当第一份外卖送到时，配送员给你打了个长长的电话，商量发票的处理方式。与此同时，第二个配送员也到了，也想给你打电话。</p><p>但是很明显，因为电话占线（也就是关闭了中断响应），第二个配送员的电话是打不通的。所以，第二个配送员很可能试几次后就走掉了（也就是丢失了一次中断）。</p><h2>软中断</h2><p>如果你弄清楚了“取外卖”的模式，那对系统的中断机制就很容易理解了。事实上，为了解决中断处理程序执行过长和中断丢失的问题，Linux 将中断处理过程分成了两个阶段，也就是<strong>上半部和下半部</strong>：</p><ul>
<li>
<p><strong>上半部用来快速处理中断</strong>，它在中断禁止模式下运行，主要处理跟硬件紧密相关的或时间敏感的工作。</p>
</li>
<li>
<p><strong>下半部用来延迟处理上半部未完成的工作，通常以内核线程的方式运行</strong>。</p>
</li>
</ul><p>比如说前面取外卖的例子，上半部就是你接听电话，告诉配送员你已经知道了，其他事儿见面再说，然后电话就可以挂断了；下半部才是取外卖的动作，以及见面后商量发票处理的动作。</p><p>这样，第一个配送员不会占用你太多时间，当第二个配送员过来时，照样能正常打通你的电话。</p><p>除了取外卖，我再举个最常见的网卡接收数据包的例子，让你更好地理解。</p><p>网卡接收到数据包后，会通过<strong>硬件中断</strong>的方式，通知内核有新的数据到了。这时，内核就应该调用中断处理程序来响应它。你可以自己先想一下，这种情况下的上半部和下半部分别负责什么工作呢？</p><p>对上半部来说，既然是快速处理，其实就是要把网卡的数据读到内存中，然后更新一下硬件寄存器的状态（表示数据已经读好了），最后再发送一个<strong>软中断</strong>信号，通知下半部做进一步的处理。</p><p>而下半部被软中断信号唤醒后，需要从内存中找到网络数据，再按照网络协议栈，对数据进行逐层解析和处理，直到把它送给应用程序。</p><p>所以，这两个阶段你也可以这样理解：</p><ul>
<li>
<p>上半部直接处理硬件请求，也就是我们常说的硬中断，特点是快速执行；</p>
</li>
<li>
<p>而下半部则是由内核触发，也就是我们常说的软中断，特点是延迟执行。</p>
</li>
</ul><p>实际上，上半部会打断CPU正在执行的任务，然后立即执行中断处理程序。而下半部以内核线程的方式执行，并且每个 CPU 都对应一个软中断内核线程，名字为 “ksoftirqd/CPU编号”，比如说， 0 号CPU对应的软中断内核线程的名字就是 ksoftirqd/0。</p><p>不过要注意的是，软中断不只包括了刚刚所讲的硬件设备中断处理程序的下半部，一些内核自定义的事件也属于软中断，比如内核调度和RCU锁（Read-Copy Update 的缩写，RCU 是 Linux 内核中最常用的锁之一）等。</p><p>那要怎么知道你的系统里有哪些软中断呢？</p><h2>查看软中断和内核线程</h2><p>不知道你还记不记得，前面提到过的 proc 文件系统。它是一种内核空间和用户空间进行通信的机制，可以用来查看内核的数据结构，或者用来动态修改内核的配置。其中：</p><ul>
<li>
<p>/proc/softirqs 提供了软中断的运行情况；</p>
</li>
<li>
<p>/proc/interrupts 提供了硬中断的运行情况。</p>
</li>
</ul><p>运行下面的命令，查看 /proc/softirqs 文件的内容，你就可以看到各种类型软中断在不同 CPU 上的累积运行次数：</p><pre><code>$ cat /proc/softirqs
                    CPU0       CPU1
          HI:          0          0
       TIMER:     811613    1972736
      NET_TX:         49          7
      NET_RX:    1136736    1506885
       BLOCK:          0          0
    IRQ_POLL:          0          0
     TASKLET:     304787       3691
       SCHED:     689718    1897539
     HRTIMER:          0          0
         RCU:    1330771    1354737
</code></pre><p>在查看 /proc/softirqs 文件内容时，你要特别注意以下这两点。</p><p>第一，要注意软中断的类型，也就是这个界面中第一列的内容。从第一列你可以看到，软中断包括了10个类别，分别对应不同的工作类型。比如 NET_RX 表示网络接收中断，而 NET_TX 表示网络发送中断。</p><p>第二，要注意同一种软中断在不同 CPU 上的分布情况，也就是同一行的内容。正常情况下，同一种中断在不同 CPU 上的累积次数应该差不多。比如这个界面中，NET_RX 在 CPU0 和 CPU1 上的中断次数基本是同一个数量级，相差不大。</p><p>不过你可能发现，TASKLET  在不同CPU上的分布并不均匀。TASKLET 是最常用的软中断实现机制，每个 TASKLET 只运行一次就会结束 ，并且只在调用它的函数所在的 CPU 上运行。</p><p>因此，使用 TASKLET 特别简便，当然也会存在一些问题，比如说由于只在一个CPU上运行导致的调度不均衡，再比如因为不能在多个 CPU 上并行运行带来了性能限制。</p><p>另外，刚刚提到过，软中断实际上是以内核线程的方式运行的，每个 CPU 都对应一个软中断内核线程，这个软中断内核线程就叫做  ksoftirqd/CPU编号。那要怎么查看这些线程的运行状况呢？</p><p>其实用 ps 命令就可以做到，比如执行下面的指令：</p><pre><code>$ ps aux | grep softirq
root         7  0.0  0.0      0     0 ?        S    Oct10   0:01 [ksoftirqd/0]
root        16  0.0  0.0      0     0 ?        S    Oct10   0:01 [ksoftirqd/1]
</code></pre><p>注意，这些线程的名字外面都有中括号，这说明 ps 无法获取它们的命令行参数（cmline）。一般来说，ps  的输出中，名字括在中括号里的，一般都是内核线程。</p><h2>小结</h2><p>Linux 中的中断处理程序分为上半部和下半部：</p><ul>
<li>
<p>上半部对应硬件中断，用来快速处理中断。</p>
</li>
<li>
<p>下半部对应软中断，用来异步处理上半部未完成的工作。</p>
</li>
</ul><p>Linux 中的软中断包括网络收发、定时、调度、RCU锁等各种类型，可以通过查看 /proc/softirqs 来观察软中断的运行情况。</p><h2>思考</h2><p>最后，我想请你一起聊聊，你是怎么理解软中断的？你有没有碰到过因为软中断出现的性能问题？你又是怎么分析它们的瓶颈的呢？你可以结合今天的内容，总结自己的思路，写下自己的问题。</p><p>欢迎在留言区和我讨论，也欢迎把这篇文章分享给你的同事、朋友。我们一起在实战中演练，在交流中进步。</p><p><img src="https://static001.geekbang.org/resource/image/56/52/565d66d658ad23b2f4997551db153852.jpg?wh=1110*549" alt=""></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/64/05/6989dce6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我来也</span>
  </div>
  <div class="_2_QraFYR_0">[D9打卡]<br>======================================<br>问题:怎么理解软中断?<br>我的理解比较简单粗暴, 硬中断是硬件产生的,比如键盘、鼠标的输入，硬盘的写入读取、网卡有数据了；软中断是软件产生的，比如程序内的定时器、[文中提到的RCU锁]。<br>再加上今天的上半部下半部，更好的理解了网卡的处理实际是有硬中断和软中断的。<br>======================================<br>问题:有没有碰到过因为软中断出现的性能问题?<br>有，且是血淋淋的教训。<br>之前的c程序用到了别人写的动态库[如:lib.a]，在物理机上，进程的cpu利用率在0%；而切换到了云服务器，即使空载，单进程的cpu利用率都有30%+。<br>以前没经验嘛,各种自查,无结果,但是自己的进程cpu利用率又那么高,总是不安心.<br>后来才通过vmstat 检测到系统的软中断每秒有100W+次.<br>最后各自百度,找人,才发现是那个动态库在处理网络收发消息时,使用了usleep(1)来休息,每次休息1纳秒,单次中断的耗时都不止1纳秒.<br>--------------------------------------<br>如果是现在,我会如下分析:<br>1.检测是哪个线程占用了cpu: top -H -p XX 1 &#47; pidstat -wut -p XX 1 <br>2.在进程中打印各线程号. 找到是哪个线程.[ 此过程也可以省略 但可以快速定位线程]<br>3.第一步应该可以判断出来中断数过高. 再使用 cat &#47;proc&#47;softirqs 查看是哪种类型的中断数过高.<br>4.不知道perf report -g -p XX 是否可以定位到具体的系统调用函数.<br>5.最终还是要查看源码,定位具体的位置,并加以验证.<br>--------------------------------------<br>感觉现在随便怎么分析都可以快速定位到是动态库的锅.<br>想当初可是好几个月都无能为力啊, 最后还是几个人各种查,搞了一周多才定位到原因.<br>最后再吐槽下,没有root权限的普通账户真是不方便啊,有些工具只能安装在自己的目录下, 还有些好用的工具根本无权限运行,哎...</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯嗯，已经是很有经验的老手了😊<br><br>大多数情况下 root 权限都是必须的，还是准备个root权限的环境实践吧</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-10 12:15:01</div>
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
  <div class="_2_QraFYR_0">经常听同事说大量的网络小包会导致性能问题，一直不太理解，从今天的课程来看，是不是大量的小网络包会导致频繁的硬中断和软中断呢？希望老师给予指点，谢谢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 正解</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-10 08:19:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/82/59/cd10c25b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>冷静</span>
  </div>
  <div class="_2_QraFYR_0">中断不会丢失的，因为有中断控制器，它会pending住所有外部中断。除非pending住的那个中断，CPU还没来得及处理，这时又来了一个同样的中断，这个中断才会丢失。还有就是自linux-2.6.3x开始就完全不支持中断嵌套了。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-13 14:57:38</div>
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
  <div class="_2_QraFYR_0">打卡，day10<br>用外卖的例子，延伸到网卡的例子，非常形象，👍</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-10 08:15:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/54/9a/76c0af70.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>每天晒白牙</span>
  </div>
  <div class="_2_QraFYR_0">【D9打卡】<br>主题:软中断<br>中断:系统用来响应硬件设备请求的一种机制，会打断进程的正常调度和执行，通过调用内核中的中断处理程序来响应设备的请求。<br>1.中断是一种异步的事件处理机制，能提高系统的并发处理能力<br>2.为了减少对正常进程运行进行影响，中断处理程序需要尽快运行。<br>3.中断分为上下两个部分<br>(1)上部分用来快速处理中断，在中断禁止模式下，主要处理跟硬件紧密相关的或时间敏感的工作<br>(2)下部分用来延迟处理上半部分未完成的工作，通常以内核线程的方式运行。<br>小结:<br>上半部分直接处理硬件请求，即硬中断，特点是快速执行<br>下部分由内核触发，即软中断，特点是延迟执行<br>软中断除了上面的下部分，还包括一些内核自定义的事件，如:内核调度 RCU锁 网络收发 定时等<br>软中断内核线程的名字:ksoftirq&#47;cpu编号<br>4.proc文件系统是一种内核空间和用户空间进行通信的机制，可以同时用来查看内核的数据结构又能用了动态修改内核的配置，如:<br>&#47;proc&#47;softirqs 提供软中断的运行情况<br>&#47;proc&#47;interrupts 提供硬中断的运行情况<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-10 07:13:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/2o1Izf2YyJSnnI0ErZ51pYRlnrmibqUTaia3tCU1PjMxuwyXSKOLUYiac2TQ5pd5gNGvS81fVqKWGvDsZLTM8zhWg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>划时代</span>
  </div>
  <div class="_2_QraFYR_0">我的心得：<br>中断过程的工作包括，应答并重新设置硬件，从设备拷贝数据到内存以及反向操作，处理硬件请求，并发送新的硬件请求。<br>中断处理流程：硬件--&gt;特殊电信号（中断）--&gt;中断处理器--&gt;IRQ线（中断号）--&gt;操作系统内核（调用中断处理程序，上半部）--&gt;ksoftirqd&#47;n（内核线程处理软中断，下半部）<br>因为中断打断了其他代码的执行（进程，内核本身，甚至其他中断处理程序），它们必须尽快执行完，所以内核把中断处理切分为两部分：上半部，对于时间非常敏感，硬件相关，保证不被其他中断（特别是相同中断）打断的任务，由中断处理程序处理；其他所有能够被允许稍后完成的工作推迟到下半部（任务尽可能放在下半部）。<br><br>基于Linux内核2.6，稍后执行的中断下半部使用三种方式实现：<br>1、软中断，可在所有处理器上同时执行，同类型也可以，仅网络和SCSI直接使用；<br>2、tasklet，通过软中断实现，同类型不能在处理器上同时执行，大部分下半部处理又tasklet实现；<br>3、工作队列，在进程上下文中执行，允许重新调度甚至睡眠，如获得大量内存、信号量、执行阻塞式I&#47;O非常有用。<br>ksoftirqd&#47;n，每个处理器都有一组辅助处理中断的内核线程，专为出现大量软中断处理而设计，避免跟其他重要任务抢夺资源。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-11 09:35:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83epGMibYc0m7cDHMsNRBUur2NPVnlBZFXoNjWomibfjnHeAO3XRt27VaH3WNtdUX11d3uIT1ZHWCxLeg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>york</span>
  </div>
  <div class="_2_QraFYR_0">老师，可否把&#47;proc&#47;softirqs输出信息中的10个类别解释一下含义？网上自行搜索了一下也没查到。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-27 10:42:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/67/0e/2a51a2df.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Eric</span>
  </div>
  <div class="_2_QraFYR_0">中断不是可以嵌套的吗？在中断处理程序中可以开中断以响应更高优先级的中断，为什么第二次中断会丢失？是指中断隐指令过程吗？？？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-10 00:23:10</div>
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
  <div class="_2_QraFYR_0">打卡，软中断。我想起了定完外卖打王者农药的场景，害怕被外卖电话软中断^_^</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-16 23:40:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Dym</span>
  </div>
  <div class="_2_QraFYR_0">CPU软中断<br><br><br>1、什么是中断<br><br>中断表示我们请求操作硬件操作准备就绪了，例如从磁盘读取数据，我们知道CPU执行速度比磁盘执行速度快几个数量级，因此如果CPU每次check磁盘是否准备就绪了，那么系统的并发能力和性能会大大下降，但是采用中断方式，异步事件驱动方式来提升系统效率，首先会在驱动程序中嵌入中断程序，一旦磁盘准备就绪就会通过驱动程序发生一个中断请求操作，CPU立马停下手里的活来执行中断程序，该中断程序会从磁盘中读取数据到内存中。<br><br>2、如何避免丢失其他中断请求<br><br><br>一定要保证中断程序快速能处理，因为当CPU在处理中断时，是不能响应其他中断请求的，那么就会导致其他中断请求丢失<br><br><br>举个取外卖的例子：我们在app上点外卖，但是这个外卖不知道什么时候到，因为送外卖小哥一旦到了目的地就会放下外卖就走，这个时候你就会一次又一次的check外卖是否到了，然而你什么时候也干不了，仅仅在来来回回看外卖是否到了，导致浪费你的时间。<br><br>改进方式：如果换一种方式，你和外卖小哥约定一个通知方式，例如当外卖到了家门口小哥就打电话通知我，我就出去取外卖。 打电话就是一次中断请求，你就安心的干其他事情，静静等电话。 如果你点了两份外卖，当第一份外卖到了，小哥电话通知你，但是你在电话中沟通发票问题，这个时候第二份外卖到了另外一个小哥给你打电话发现占线，几次尝试后还是失败，这个时候外卖小哥就走了，导致丢失了这次外卖。<br><br>解决：在电话中只回答好，然后沟通发票问题当面说，这个时候就可以接到另外小哥的电话。 所以中断请求分为两个阶段：<br><br>第一阶段(上半部请求)：接受硬件中断请求(从硬件中取完数据后发送一次软断请求，复杂逻辑交给下半部分请求，)，称为硬中断，特点是处理速度快<br><br>第二阶段(下半部请求)： 内核线程接受到上半部分软中断请求，就会异步的继续执行上半部未完成的请求， 称为软中断，特点延迟执行<br><br>举个网络接受数据例子：<br><br>当网卡接收到数据时，首先会发送一个硬中断请求，这个时候CPU就会执行中断处理程序，快速将网卡中数据读取到内存中，完成后会发送一个软中断请求，下半部被软中断信号唤醒后就会按照网络协议栈将内存数据进行解析处理，最终递给应用程序。(处理期间还是可以响应其他硬中断请求的)<br><br>3、查看硬中断和软中断运行情况<br><br><br><br>可以查看 &#47;proc&#47;interrupts 和 &#47;proc&#47;softirq 文件<br><br><br><br><br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-26 13:42:14</div>
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
  <div class="_2_QraFYR_0">老师说软中断是在内核线程ksoftirqd&#47;cpu中运行。思考和疑问，这个内核线程为什么叫线程，不是进程吗？猜测这个线程很特殊，属于内核中的某一进程，而且这个进程还能够直接与挂接的用户空间内存进行交互。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 内核管理的任务一般叫线程是因为它们没有独立的虚拟内存空间，并且它也不会去跟用户空间内存交互。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-10 12:55:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/f6/b4/e4d13c1a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>沙皮狗</span>
  </div>
  <div class="_2_QraFYR_0">老师，有一点很疑惑。在《Linux内核设计与实现》一书中提到&quot;在2.6以后的内核中提到，目前有三种方式实现中断下半部：工作队列，tasklet和软中断，软中断机制并不完全等同于中断下半部，很多人把所有下半部当成是软中断。&quot;请问这部分怎么理解？麻烦老师解答一下</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这儿区分的更细了，tasklet 也是基于软中断的，而工作队列则是用于可以睡眠的下半部处理过程</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-08 10:21:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/7c/59/26b1e65a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>科学Jia</span>
  </div>
  <div class="_2_QraFYR_0">老师老师，女童鞋举手问您，软中断既然是内核方式运行，那就没有可能是应用程序引起，那如果软中断很频繁，作为应用程序该怎么考虑解决内核的问题？很忧伤啊。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 当然可能了，比如还是以网络软中断为例，同样发送1GB数据包，每次发送1KB跟每次发送32B带来的结果肯定是不同的。<br><br>具体优化方法不要太着急，先定位出根源，之后一般都不难找出解决方法。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-24 15:31:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/92/01/c723d180.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>饼子</span>
  </div>
  <div class="_2_QraFYR_0">积累了知识，但是没有遇到优化软中断的情况，不知道在哪些情况下会考虑进去？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 软中断问题在大流量网络中最为常见</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-10 12:52:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/db/26/54f2c164.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>靠人品去赢</span>
  </div>
  <div class="_2_QraFYR_0">问一个很low的问题，我们平常有时候会使用crtl+z ，crtl+c这种挂起或中断的命令，跟文中提到的硬中断，软中断是否有什么关联。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这属于进程间通信的“信号”机制</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-10 10:07:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/56/43/abb7bfe3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>阿巍-豆夫</span>
  </div>
  <div class="_2_QraFYR_0">老师，请教下，一般进程都是多线程运行的。你所说的中断是线程的级别，还是进程的级别？ 送外卖的场景没有解释到多线程的场景。 如果是多线程是不是根本不存在中断的情况？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 中断是内核中的任务，可以理解为跟线程是并列的概念，但它的优先级更高。<br><br>中断中也没有进程的概念，线程才是调度的基本单位。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-10 09:55:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/1b/ca/9afb89a2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Days</span>
  </div>
  <div class="_2_QraFYR_0">而下半部以内核线程的方式执行，并且每个 CPU 都对应一个软中断内核线程，<br>这里我觉得不是所有软中断直接被ksoftirqd处理，只有大量软中断产生，或者处理软终端超时才唤醒ksoftirqd线程</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-10 08:27:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/26/d4/add17f35.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>聰</span>
  </div>
  <div class="_2_QraFYR_0">明明只有两个cpu ，请问老师为何会出现多个CPU呢？百度无果<br>[root@master ~]# grep processor &#47;proc&#47;cpuinfo |wc -l<br>2<br>[root@master ~]# head -1 &#47;proc&#47;softirqs<br>                    CPU0       CPU1       CPU2       CPU3       CPU4       CPU5       CPU6       CPU7       CPU8       CPU9       CPU10      CPU11      CPU12      CPU13      CPU14      CPU15      CPU16      CPU17      CPU18      CPU19      CPU20      CPU21      CPU22      CPU23      CPU24      CPU25      CPU26      CPU27      CPU28      CPU29      CPU30      CPU31      CPU32      CPU33      CPU34      CPU35      CPU36      CPU37      CPU38      CPU39      CPU40      CPU41      CPU42      CPU43      CPU44      CPU45      CPU46      CPU47      CPU48      CPU49      CPU50      CPU51      CPU52      CPU53      CPU54      CPU55      CPU56      CPU57      CPU58      CPU59      CPU60      CPU61      CPU62      CPU63      CPU64      CPU65      CPU66      CPU67      CPU68      CPU69      CPU70      CPU71      CPU72      CPU73      CPU74      CPU75      CPU76      CPU77      CPU78      CPU79      CPU80      CPU81      CPU82      CPU83      CPU84      CPU85      CPU86      CPU87      CPU88      CPU89      CPU90      CPU91      CPU92      CPU93      CPU94      CPU95      CPU96      CPU97      CPU98      CPU99      CPU100     CPU101     CPU102     CPU103     CPU104     CPU105     CPU106     CPU107     CPU108     CPU109     CPU110     CPU111     CPU112     CPU113     CPU114     CPU115     CPU116     CPU117     CPU118     CPU119     CPU120     CPU121     CPU122     CPU123     CPU124     CPU125     CPU126     CPU127<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这是发行版的问题，忽略多余的就可以了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-03 17:12:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/1d/e6/c8f20c9e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>一往而深</span>
  </div>
  <div class="_2_QraFYR_0">我也来理解一下，先来硬的：<br>硬件层面，来事情了cpu就必须过来处理，网卡就能存储那么多数据，如果设计成延迟的话可能有丢数据的坑，所以必须过来把数据拿出来放在容量大的地方，然后就进入下半场，软中断，应该就没那么着急了，有时间就给它处理掉，软中断处理过程中应该也是可以再处理硬中断的。意淫一下 不知道对不对啊</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对的，硬中断优先处理</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-10 20:11:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/35/25/bab760a1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>好好学习</span>
  </div>
  <div class="_2_QraFYR_0">期待第二篇，我有个业务48核心，怎么调整都是只用了前面24核心的软中断，期待更新</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-10 19:10:23</div>
  </div>
</div>
</div>
</li>
</ul>