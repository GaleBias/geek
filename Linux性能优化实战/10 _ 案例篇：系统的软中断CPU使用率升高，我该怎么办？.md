<audio title="10 _ 案例篇：系统的软中断CPU使用率升高，我该怎么办？" src="https://static001.geekbang.org/resource/audio/01/c2/019804f0aadcebd7e9f44d3d31b2a0c2.mp3" controls="controls"></audio> 
<p>你好，我是倪朋飞。</p><p>上一期我给你讲了软中断的基本原理，我们先来简单复习下。</p><p>中断是一种异步的事件处理机制，用来提高系统的并发处理能力。中断事件发生，会触发执行中断处理程序，而中断处理程序被分为上半部和下半部这两个部分。</p><ul>
<li>
<p>上半部对应硬中断，用来快速处理中断；</p>
</li>
<li>
<p>下半部对应软中断，用来异步处理上半部未完成的工作。</p>
</li>
</ul><p>Linux 中的软中断包括网络收发、定时、调度、RCU锁等各种类型，我们可以查看 proc 文件系统中的 /proc/softirqs  ，观察软中断的运行情况。</p><p>在 Linux 中，每个 CPU 都对应一个软中断内核线程，名字是 ksoftirqd/CPU编号。当软中断事件的频率过高时，内核线程也会因为CPU 使用率过高而导致软中断处理不及时，进而引发网络收发延迟、调度缓慢等性能问题。</p><p>软中断 CPU 使用率过高也是一种最常见的性能问题。今天，我就用最常见的反向代理服务器 Nginx 的案例，教你学会分析这种情况。</p><h2>案例</h2><h3>你的准备</h3><p>接下来的案例基于 Ubuntu 18.04，也同样适用于其他的 Linux 系统。我使用的案例环境是这样的：</p><ul>
<li>
<p>机器配置：2 CPU、8 GB 内存。</p>
</li>
<li>
<p>预先安装 docker、sysstat、sar 、hping3、tcpdump 等工具，比如 apt-get install <a href="http://docker.io">docker.io</a> sysstat hping3 tcpdump。</p>
</li>
</ul><!-- [[[read_end]]] --><p>这里我又用到了三个新工具，sar、  hping3 和 tcpdump，先简单介绍一下：</p><ul>
<li>
<p>sar 是一个系统活动报告工具，既可以实时查看系统的当前活动，又可以配置保存和报告历史统计数据。</p>
</li>
<li>
<p>hping3 是一个可以构造 TCP/IP 协议数据包的工具，可以对系统进行安全审计、防火墙测试等。</p>
</li>
<li>
<p>tcpdump 是一个常用的网络抓包工具，常用来分析各种网络问题。</p>
</li>
</ul><p>本次案例用到两台虚拟机，我画了一张图来表示它们的关系。</p><p><img src="https://static001.geekbang.org/resource/image/5f/96/5f9487847e937f955ebc2ec86d490b96.png?wh=1632*1032" alt=""></p><p>你可以看到，其中一台虚拟机运行 Nginx ，用来模拟待分析的 Web 服务器；而另一台当作Web 服务器的客户端，用来给 Nginx 增加压力请求。使用两台虚拟机的目的，是为了相互隔离，避免“交叉感染”。</p><p>接下来，我们打开两个终端，分别 SSH 登录到两台机器上，并安装上面提到的这些工具。</p><p>同以前的案例一样，下面的所有命令都默认以 root 用户运行，如果你是用普通用户身份登陆系统，请运行 sudo su root 命令切换到 root 用户。</p><p>如果安装过程中有什么问题，同样鼓励你先自己搜索解决，解决不了的，可以在留言区向我提问。如果你以前已经安装过了，就可以忽略这一点了。</p><h3>操作和分析</h3><p>安装完成后，我们先在第一个终端，执行下面的命令运行案例，也就是一个最基本的 Nginx 应用：</p><pre><code># 运行Nginx服务并对外开放80端口
$ docker run -itd --name=nginx -p 80:80 nginx
</code></pre><p>然后，在第二个终端，使用 curl 访问 Nginx 监听的端口，确认 Nginx 正常启动。假设 192.168.0.30 是 Nginx 所在虚拟机的 IP 地址，运行 curl 命令后你应该会看到下面这个输出界面：</p><pre><code>$ curl http://192.168.0.30/
&lt;!DOCTYPE html&gt;
&lt;html&gt;
&lt;head&gt;
&lt;title&gt;Welcome to nginx!&lt;/title&gt;
...
</code></pre><p>接着，还是在第二个终端，我们运行 hping3 命令，来模拟 Nginx 的客户端请求：</p><pre><code># -S参数表示设置TCP协议的SYN（同步序列号），-p表示目的端口为80
# -i u100表示每隔100微秒发送一个网络帧
# 注：如果你在实践过程中现象不明显，可以尝试把100调小，比如调成10甚至1
$ hping3 -S -p 80 -i u100 192.168.0.30
</code></pre><p>现在我们再回到第一个终端，你应该发现了异常。是不是感觉系统响应明显变慢了，即便只是在终端中敲几个回车，都得很久才能得到响应？这个时候应该怎么办呢？</p><p>虽然在运行 hping3 命令时，我就已经告诉你，这是一个 SYN FLOOD 攻击，你肯定也会想到从网络方面入手，来分析这个问题。不过，在实际的生产环境中，没人直接告诉你原因。</p><p>所以，我希望你把 hping3 模拟 SYN FLOOD 这个操作暂时忘掉，然后重新从观察到的问题开始，分析系统的资源使用情况，逐步找出问题的根源。</p><p>那么，该从什么地方入手呢？刚才我们发现，简单的 SHELL 命令都明显变慢了，先看看系统的整体资源使用情况应该是个不错的注意，比如执行下 top 看看是不是出现了 CPU 的瓶颈。我们在第一个终端运行 top 命令，看一下系统整体的资源使用情况。</p><pre><code># top运行后按数字1切换到显示所有CPU
$ top
top - 10:50:58 up 1 days, 22:10,  1 user,  load average: 0.00, 0.00, 0.00
Tasks: 122 total,   1 running,  71 sleeping,   0 stopped,   0 zombie
%Cpu0  :  0.0 us,  0.0 sy,  0.0 ni, 96.7 id,  0.0 wa,  0.0 hi,  3.3 si,  0.0 st
%Cpu1  :  0.0 us,  0.0 sy,  0.0 ni, 95.6 id,  0.0 wa,  0.0 hi,  4.4 si,  0.0 st
...

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
    7 root      20   0       0      0      0 S   0.3  0.0   0:01.64 ksoftirqd/0
   16 root      20   0       0      0      0 S   0.3  0.0   0:01.97 ksoftirqd/1
 2663 root      20   0  923480  28292  13996 S   0.3  0.3   4:58.66 docker-containe
 3699 root      20   0       0      0      0 I   0.3  0.0   0:00.13 kworker/u4:0
 3708 root      20   0   44572   4176   3512 R   0.3  0.1   0:00.07 top
    1 root      20   0  225384   9136   6724 S   0.0  0.1   0:23.25 systemd
    2 root      20   0       0      0      0 S   0.0  0.0   0:00.03 kthreadd
...
</code></pre><p>这里你有没有发现异常的现象？我们从第一行开始，逐个看一下：</p><ul>
<li>
<p>平均负载全是0，就绪队列里面只有一个进程（1 running）。</p>
</li>
<li>
<p>每个CPU的使用率都挺低，最高的CPU1的使用率也只有4.4%，并不算高。</p>
</li>
<li>
<p>再看进程列表，CPU使用率最高的进程也只有 0.3%，还是不高呀。</p>
</li>
</ul><p>那为什么系统的响应变慢了呢？既然每个指标的数值都不大，那我们就再来看看，这些指标对应的更具体的含义。毕竟，哪怕是同一个指标，用在系统的不同部位和场景上，都有可能对应着不同的性能问题。</p><p>仔细看 top 的输出，两个 CPU的使用率虽然分别只有 3.3%和4.4%，但都用在了软中断上；而从进程列表上也可以看到，CPU使用率最高的也是软中断进程 ksoftirqd。看起来，软中断有点可疑了。</p><p>根据上一期的内容，既然软中断可能有问题，那你先要知道，究竟是哪类软中断的问题。停下来想想，上一节我们用了什么方法，来判断软中断类型呢？没错，还是proc文件系统。观察 /proc/softirqs 文件的内容，你就能知道各种软中断类型的次数。</p><p>不过，这里的各类软中断次数，又是什么时间段里的次数呢？它是系统运行以来的<strong>累积中断次数</strong>。所以我们直接查看文件内容，得到的只是累积中断次数，对这里的问题并没有直接参考意义。因为，这些<strong>中断次数的变化速率</strong>才是我们需要关注的。</p><p>那什么工具可以观察命令输出的变化情况呢？我想你应该想起来了，在前面案例中用过的  watch 命令，就可以定期运行一个命令来查看输出；如果再加上 -d 参数，还可以高亮出变化的部分，从高亮部分我们就可以直观看出，哪些内容变化得更快。</p><p>比如，还是在第一个终端，我们运行下面的命令：</p><pre><code>$ watch -d cat /proc/softirqs
                    CPU0       CPU1
          HI:          0          0
       TIMER:    1083906    2368646
      NET_TX:         53          9
      NET_RX:    1550643    1916776
       BLOCK:          0          0
    IRQ_POLL:          0          0
     TASKLET:     333637       3930
       SCHED:     963675    2293171
     HRTIMER:          0          0
         RCU:    1542111    1590625
</code></pre><p>通过 /proc/softirqs 文件内容的变化情况，你可以发现， TIMER（定时中断）、NET_RX（网络接收）、SCHED（内核调度）、RCU（RCU锁）等这几个软中断都在不停变化。</p><p>其中，NET_RX，也就是网络数据包接收软中断的变化速率最快。而其他几种类型的软中断，是保证 Linux 调度、时钟和临界区保护这些正常工作所必需的，所以它们有一定的变化倒是正常的。</p><p>那么接下来，我们就从网络接收的软中断着手，继续分析。既然是网络接收的软中断，第一步应该就是观察系统的网络接收情况。这里你可能想起了很多网络工具，不过，我推荐今天的主人公工具  sar  。</p><p>sar  可以用来查看系统的网络收发情况，还有一个好处是，不仅可以观察网络收发的吞吐量（BPS，每秒收发的字节数），还可以观察网络收发的 PPS，即每秒收发的网络帧数。</p><p>我们在第一个终端中运行 sar 命令，并添加 -n DEV 参数显示网络收发的报告：</p><pre><code># -n DEV 表示显示网络收发的报告，间隔1秒输出一组数据
$ sar -n DEV 1
15:03:46        IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s   %ifutil
15:03:47         eth0  12607.00   6304.00    664.86    358.11      0.00      0.00      0.00      0.01
15:03:47      docker0   6302.00  12604.00    270.79    664.66      0.00      0.00      0.00      0.00
15:03:47           lo      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
15:03:47    veth9f6bbcd   6302.00  12604.00    356.95    664.66      0.00      0.00      0.00      0.05
</code></pre><p>对于 sar 的输出界面，我先来简单介绍一下，从左往右依次是：</p><ul>
<li>
<p>第一列：表示报告的时间。</p>
</li>
<li>
<p>第二列：IFACE 表示网卡。</p>
</li>
<li>
<p>第三、四列：rxpck/s 和 txpck/s 分别表示每秒接收、发送的网络帧数，也就是  PPS。</p>
</li>
<li>
<p>第五、六列：rxkB/s 和 txkB/s 分别表示每秒接收、发送的千字节数，也就是  BPS。</p>
</li>
<li>
<p>后面的其他参数基本接近0，显然跟今天的问题没有直接关系，你可以先忽略掉。</p>
</li>
</ul><p>我们具体来看输出的内容，你可以发现：</p><ul>
<li>
<p>对网卡 eth0来说，每秒接收的网络帧数比较大，达到了 12607，而发送的网络帧数则比较小，只有 6304；每秒接收的千字节数只有 664 KB，而发送的千字节数更小，只有 358 KB。</p>
</li>
<li>
<p>docker0 和 veth9f6bbcd 的数据跟 eth0 基本一致，只是发送和接收相反，发送的数据较大而接收的数据较小。这是 Linux 内部网桥转发导致的，你暂且不用深究，只要知道这是系统把 eth0 收到的包转发给 Nginx 服务即可。具体工作原理，我会在后面的网络部分详细介绍。</p>
</li>
</ul><p>从这些数据，你有没有发现什么异常的地方？</p><p>既然怀疑是网络接收中断的问题，我们还是重点来看 eth0 ：接收的 PPS 比较大，达到 12607，而接收的 BPS 却很小，只有 664 KB。直观来看网络帧应该都是比较小的，我们稍微计算一下，664*1024/12607 = 54 字节，说明平均每个网络帧只有 54 字节，这显然是很小的网络帧，也就是我们通常所说的小包问题。</p><p>那么，有没有办法知道这是一个什么样的网络帧，以及从哪里发过来的呢？</p><p>使用 tcpdump 抓取 eth0 上的包就可以了。我们事先已经知道， Nginx 监听在 80 端口，它所提供的 HTTP 服务是基于 TCP 协议的，所以我们可以指定 TCP 协议和 80 端口精确抓包。</p><p>接下来，我们在第一个终端中运行 tcpdump 命令，通过 -i eth0 选项指定网卡 eth0，并通过 tcp port 80 选项指定 TCP 协议的 80 端口：</p><pre><code># -i eth0 只抓取eth0网卡，-n不解析协议名和主机名
# tcp port 80表示只抓取tcp协议并且端口号为80的网络帧
$ tcpdump -i eth0 -n tcp port 80
15:11:32.678966 IP 192.168.0.2.18238 &gt; 192.168.0.30.80: Flags [S], seq 458303614, win 512, length 0
...
</code></pre><p>从 tcpdump 的输出中，你可以发现</p><ul>
<li>
<p>192.168.0.2.18238 &gt; 192.168.0.30.80  ，表示网络帧从 192.168.0.2 的 18238 端口发送到 192.168.0.30 的 80 端口，也就是从运行 hping3 机器的 18238 端口发送网络帧，目的为 Nginx 所在机器的 80 端口。</p>
</li>
<li>
<p>Flags [S] 则表示这是一个 SYN 包。</p>
</li>
</ul><p>再加上前面用 sar 发现的， PPS 超过 12000的现象，现在我们可以确认，这就是从 192.168.0.2 这个地址发送过来的 SYN FLOOD 攻击。</p><p>到这里，我们已经做了全套的性能诊断和分析。从系统的软中断使用率高这个现象出发，通过观察 /proc/softirqs 文件的变化情况，判断出软中断类型是网络接收中断；再通过 sar 和 tcpdump  ，确认这是一个 SYN FLOOD 问题。</p><p>SYN FLOOD 问题最简单的解决方法，就是从交换机或者硬件防火墙中封掉来源 IP，这样 SYN FLOOD 网络帧就不会发送到服务器中。</p><p>至于 SYN FLOOD 的原理和更多解决思路，你暂时不需要过多关注，后面的网络章节里我们都会学到。</p><p>案例结束后，也不要忘了收尾，记得停止最开始启动的 Nginx 服务以及 hping3 命令。</p><p>在第一个终端中，运行下面的命令就可以停止 Nginx 了：</p><pre><code># 停止 Nginx 服务
$ docker rm -f nginx
</code></pre><p>然后到第二个终端中按下 Ctrl+C 就可以停止 hping3。</p><h2>小结</h2><p>软中断CPU使用率（softirq）升高是一种很常见的性能问题。虽然软中断的类型很多，但实际生产中，我们遇到的性能瓶颈大多是网络收发类型的软中断，特别是网络接收的软中断。</p><p>在碰到这类问题时，你可以借用 sar、tcpdump 等工具，做进一步分析。不要害怕网络性能，后面我会教你更多的分析方法。</p><h2>思考</h2><p>最后，我想请你一起来聊聊，你所碰到的软中断问题。你所碰到的软中问题是哪种类型，是不是这个案例中的小包问题？你又是怎么分析它们的来源并解决的呢？可以结合今天的案例，总结你自己的思路和感受。如果遇到过其他问题，也可以留言给我一起解决。</p><p>欢迎在留言区和我讨论，也欢迎你把这篇文章分享给你的同事、朋友。我们一起在实战中演练，在交流中进步。</p><p><img src="https://static001.geekbang.org/resource/image/56/52/565d66d658ad23b2f4997551db153852.jpg?wh=1110*549" alt=""></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/47/42/5b55bd1a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>倪朋飞</span>
  </div>
  <div class="_2_QraFYR_0">统一回复一下终端卡顿的问题，这个是由于网络延迟增大（甚至是丢包）导致的。比如你可以再拿另外一台机器（也就是第三台）在 hping3 运行的前后 ping 一下案例机器，ping -c3 &lt;ip&gt;<br><br>hping3 运行前，你可能看到最长的也不超过 1 ms：<br><br>3 packets transmitted, 3 received, 0% packet loss, time 2028ms<br>rtt min&#47;avg&#47;max&#47;mdev = 0.815&#47;0.914&#47;0.989&#47;0.081 ms<br><br>而 hping3 运行时，不仅平均延迟增长到了 245 ms，而且还会有丢包的发生：<br><br>3 packets transmitted, 2 received, 33% packet loss, time 2026ms<br>rtt min&#47;avg&#47;max&#47;mdev = 240.637&#47;245.758&#47;250.880&#47;5.145 ms<br><br>网络问题的排查方法在后面的文章中还会讲，这儿只是从 CPU 利用率的角度出发，你可以发现也有可能是网络导致的问题。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-12 17:02:10</div>
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
  <div class="_2_QraFYR_0">软终端不高导致系统卡顿，我的理解是这样的，其实不是系统卡顿，而是由于老师用的ssh远程登录，在这期间hping3大量发包，导致其他网络连接延迟，ssh通过网络连接，使ssh客户端感觉卡顿现象。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 正解👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-13 09:49:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/de/70/52b1d5ab.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>不去会死</span>
  </div>
  <div class="_2_QraFYR_0">搞运维好些年了。一些底层性能的东西，感觉自己始终是一知半解，通过这个专栏了解的更深入了，确实学到了很多。而且老师也一直在积极回复同学们的问题，相比某些专栏的老师发出来就不管的状态好太多。给老师点赞。<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 也很高兴看到大家有所收获😊</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-27 17:42:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/5c/2b/25c1c14c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>卿卿子衿</span>
  </div>
  <div class="_2_QraFYR_0">有同学说在查看软中断数据时会显示128个核的数据，我的也是，虽然只有一个核，但是会显示128个核的信息，用下面的命令可以提取有数据的核，我的1核，所以这个命令只能显示1核，多核需要做下修改<br><br>watch -d &quot;&#47;bin&#47;cat &#47;proc&#47;softirqs | &#47;usr&#47;bin&#47;awk &#39;NR == 1{printf  \&quot;%13s %s\n\&quot;,\&quot; \&quot;,\$1}; NR &gt; 1{printf \&quot;%13s %s\n\&quot;,\$1,\$2}&#39;&quot;</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢分享</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-12 14:29:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/61/bc/88a905a5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>亮点</span>
  </div>
  <div class="_2_QraFYR_0">倪老师，案例中硬中断CPU占用率为啥是0呢，硬中断和软中断次数不是基本一致的吗?</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-12 12:54:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/a0/39/ddcf26ac.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>bruceding</span>
  </div>
  <div class="_2_QraFYR_0">找网络相关的错误，可以有几种方式。<br>1. 找系统类的错误， dmesg | tail <br>2. 直接的网络错误   sar -n ETCP 1 或者 sar -n EDEV 1 <br>3.查看网络状态， netstat -s  或者 watch -d netstat -s <br>4.网络状态的统计  ss -ant | awk &#39;{++s[$1]} END {for(k in s) print k,s[k]}&#39;</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-12 09:00:57</div>
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
  <div class="_2_QraFYR_0">软中断对系统的影响排查思路：<br>1. 当发现服务器或业务卡顿的时候，首先通过top命令来查看服务器负载，cpu使用率和各个cpu参数，然后排查cpu占用较高的进程；<br>2. 如果发现cpu使用率并不高，但是si很高，而且ksoftirqd进程cpu占用率高，则说明服务器一直在发生软中断；<br>3. 通过cat &#47;proc&#47;softirqs来分析是哪方面的软中断次数最多，可以通过watch命令来查看变化最快的值；<br>4. 一般情况下网络发生中断的情况会比较多，如果发现是网络收发包导致的软中断，则通过sar命令来查看此时的收发包速率和收发包数据量来验证是否真的是网络收发包过多导致，也可以计算每个包的大小，以此来计算服务器是否收到了flood攻击；<br>5. 通过tcpdump来抓包分析数据包来源ip和数据包类型，通过抓包数据中的Flags来分析数据包类型；<br>6. 通过防火墙封堵异常ip；</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-01 09:52:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKsz8j0bAayjSne9iakvjzUmvUdxWEbsM9iasQ74spGFayIgbSE232sH2LOWmaKtx1WqAFDiaYgVPwIQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>2xshu</span>
  </div>
  <div class="_2_QraFYR_0">老师，网络软中断明明只占了百分之四左右。为什么终端会感觉那么卡呢？不是很理解这点呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 参考置顶回复</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-12 08:38:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/84/02/e9937c31.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>陈文庆</span>
  </div>
  <div class="_2_QraFYR_0">很喜欢这个专栏，只有干货，诚意满满。做业务研发的天天堆业务代码，对系统底层一直不了解，出了问题总是抓瞎，临时网上找的方法不系统，不深入讲原理，多年也不进步，能通过专栏系统学习，真是打开了一扇门，感谢老师！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-22 21:53:02</div>
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
  <div class="_2_QraFYR_0">[D10打卡]<br> &quot;hping3 -S -p 80 -i u100 192.168.0.30&quot; 这里的u100改为了1 也没觉得终端卡,top的软中断%si倒是从4%上升了不少,吃满了一个cpu.<br>可能是我直接在宿主机上开终端的原因,本身两个虚拟机都在这个宿主机上,都是走的本地网络.<br>本地网卡可能还处理的过来.<br>-----------<br>在工作中,倒是没有遇到小包导致的性能问题.<br>也许是用户数太少,流量不够.[才二三十兆带宽], 也许是之前发生了,自己并不知道.<br>在工作中遇到的软中间导致的性能问题就是上期说的usleep(1)了.<br>-----------<br>本期又学到新东西了:<br>1.sar 原来可以这么方便的看各网卡流量,甚至是网络帧数.<br>到目前为止,我都是用的最原始的方法:在网上找的一个脚本,分析ifconfig中的数据,来统计某个网卡的流量.一来需要指定某个网卡(默认eth0),二来显示的数据不太准确且不友好(sleep 1做差值).<br>2.nping3 居然可以用来模拟SYN FLOOD. (不要用来做坏事哦)<br>3.tcpdump 之前有所耳闻. 用的不多. 平常有解包需求,都是在windows下用wireshark,毕竟是图形界面.<br>-----------<br>有同学说&quot;仅凭tcpdump发现一个syn包就断定是SYN FLOOD，感觉有些草断&quot;<br>我是这样认为的:<br>你tcpdump 截取一段时间的日志, 除去正常的流量, 着重分析异常的,再根据ip来统计出现的次数, 还是可以合理推理出来老师结论的.<br>毕竟平常不会有哪个ip每秒产生这么多的syn,且持续这么长时间.</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍<br><br>最后一个问题其实前面已经看到PPS了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-12 10:55:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/75/dd/9ead6e69.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>黄海峰</span>
  </div>
  <div class="_2_QraFYR_0">这真是非常干货和务实的一个专栏，这么便宜，太值了。。。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 😊 </p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-12 09:24:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/54/98/52ca7053.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Vicky🐣🐣🐣</span>
  </div>
  <div class="_2_QraFYR_0">1. 网络收发软中断过多导致命令行比较卡，是因为键盘敲击命令行属于硬中断，内核也需要去处理的原因吗？<br>2. 观察&#47;proc&#47;softirqs，发现变化的值是TIMER、NET_RX、BLOCK、RCU，奇怪的是SCHED一直为0，求老师解答</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我们是SSH登陆的机器，还是走网络而不是键盘中断😓</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-12 11:40:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/11/4b/fa64f061.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>xfan</span>
  </div>
  <div class="_2_QraFYR_0">ssh的tty其实也是通过网络传输的，既然是经过网卡，当然会卡，这就是攻击所带来的结果</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-12 22:52:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/f2/49/43f222e4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>kyra</span>
  </div>
  <div class="_2_QraFYR_0">最近刚遇到这个问题，si高达80%，试着分析了下，但是不知道分析思路对不对<br>环境：1C1G的虚拟机上启动docker容器的nginx<br>场景：在物理机上用jmeter，1线程间隔1秒，永久循环压测nginx<br>结果：服务器si高达80%，TPS达到1200多，响应时间1ms以下<br>分析：<br>开始压测的时候就是有top监控服务器的情况，发现si很快飙到80%，并且一直维持在这个比例<br>然后查看&#47;proc&#47;softirqs里的数据，NET_RX数据变化最快，根据文章提供的办法，所以分析网络，使用sar分析网络，通过计算发现每帧字节数在115字节左右，不确定是不是小包，但是感觉直接关闭压测工具，这里不是很理解，万一是正常压测呢？<br>然后我又使用vmstat查看了cpu的情况，发现nginx的cs变化比较大，没有压测的时候是80多，压测的时候达到了20000多，然后通过pidstat -wt查看发现nginx和docker 容器的上下文切换废材频繁，且都是主动上下文切换，所以考虑是压测过快，响应时间才1ms不到，CPU资源不足造成的如此频繁的上下文切换，但是又总觉得哪里不太对，希望老师能够指正一下</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-23 10:37:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/14/3a/94e25d0c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>男人十八一枝花</span>
  </div>
  <div class="_2_QraFYR_0">cat &#47;proc&#47;softirqs时我有4个cpu，可用<br>watch -d &quot;&#47;bin&#47;cat &#47;proc&#47;softirqs | &#47;usr&#47;bin&#47;awk &#39;NR == 1{printf \&quot;%-15s %-15s %-15s %-15s %-15s\n\&quot;,\&quot; \&quot;,\$1,\$2,\$3,\$4}; NR &gt; 1{printf \&quot;%-15s %-15s %-15s %-15s %-15s\n\&quot;,\$1,\$2,\$3,\$4,\$5}&#39;&quot;<br>查看</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢分享👍 不懂awk的赶紧去学习😊</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-17 17:32:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Um0fKCDsGBRStZBF1M4HLPSq8jiancnNoYKiaYyYldFX0NObkyUFmnVKTgjm6Y7wUiaCQ3Vm9Ic213l65kJfUzq4w/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_c92584</span>
  </div>
  <div class="_2_QraFYR_0">老师说的这些工具的使用是基础中的基础，值得反复实践。<br><br>另外推荐 netdata 这个监控神器，下载了netdata&#47;netdata docker 镜像的前提下，可以瞬间一个从监控指标采集、存储、到展示的整套监控系统，老师说的这些案例可以直接用其分析。<br><br>这套系统可以和其他外围的监控系统整合。并且，通过 netdata registery 整合管理多个 netdata 示例。值得一试！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-08 12:52:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJtl3p4gcguAZy580SyoQAic79Z7QAvTcibnicV9K8x2Yzbxa8BlknwhquzTPPklaWPDDbrECQG3uurg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lumence</span>
  </div>
  <div class="_2_QraFYR_0">是不是不能用内网网卡来测。我docker的ip是 172.17.0.2。可是我开启 -i u1 或者 --flood 依然没有卡顿现象<br>sudo hping3 -S -p 80 --flood 172.17.0.2<br>HPING 172.17.0.2 (docker0 172.17.0.2): S set, 40 headers + 0 data bytes<br>hping in flood mode, no replies will be shown</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-15 15:01:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/a7/fa/1dca9fd5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>王星旗</span>
  </div>
  <div class="_2_QraFYR_0">hping3 -S --flood  -p 80 ip，这样压力更大，哈哈</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-11 16:29:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/7a/8e/934cbcbc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>几叶星辰</span>
  </div>
  <div class="_2_QraFYR_0">怎么让网卡中断平衡呢，可以请教下linux 2.6.40。中断平衡问题吗，以及内核版本更高的版本？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 配置 smp_affinity 或者开启 irqbalance 服务</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-24 15:46:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/54/98/52ca7053.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Vicky🐣🐣🐣</span>
  </div>
  <div class="_2_QraFYR_0">执行了一下hping3，机器直接卡死了，登录不上去了，哈哈</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可能太猛了，调整下参数再试试</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-12 11:01:29</div>
  </div>
</div>
</div>
</li>
</ul>