<audio title="34 _ 关于 Linux 网络，你必须知道这些（下）" src="https://static001.geekbang.org/resource/audio/d1/da/d15ac64e53110f242c6fc7c1b8f121da.mp3" controls="controls"></audio> 
<p>你好，我是倪朋飞。</p><p>上一节，我带你学习了 Linux 网络的基础原理。简单回顾一下，Linux 网络根据 TCP/IP 模型，构建其网络协议栈。TCP/IP 模型由应用层、传输层、网络层、网络接口层等四层组成，这也是 Linux 网络栈最核心的构成部分。</p><p>应用程序通过套接字接口发送数据包时，先要在网络协议栈中从上到下逐层处理，然后才最终送到网卡发送出去；而接收数据包时，也要先经过网络栈从下到上的逐层处理，最后送到应用程序。</p><p>了解Linux 网络的基本原理和收发流程后，你肯定迫不及待想知道，如何去观察网络的性能情况。具体而言，哪些指标可以用来衡量 Linux 的网络性能呢？</p><h2>性能指标</h2><p>实际上，我们通常用带宽、吞吐量、延时、PPS（Packet Per Second）等指标衡量网络的性能。</p><ul>
<li>
<p><strong>带宽</strong>，表示链路的最大传输速率，单位通常为 b/s （比特/秒）。</p>
</li>
<li>
<p><strong>吞吐量</strong>，表示单位时间内成功传输的数据量，单位通常为 b/s（比特/秒）或者 B/s（字节/秒）。吞吐量受带宽限制，而吞吐量/带宽，也就是该网络的使用率。</p>
</li>
<li>
<p><strong>延时</strong>，表示从网络请求发出后，一直到收到远端响应，所需要的时间延迟。在不同场景中，这一指标可能会有不同含义。比如，它可以表示，建立连接需要的时间（比如 TCP 握手延时），或一个数据包往返所需的时间（比如 RTT）。</p>
</li>
<li>
<p><strong>PPS</strong>，是 Packet Per Second（包/秒）的缩写，表示以网络包为单位的传输速率。PPS 通常用来评估网络的转发能力，比如硬件交换机，通常可以达到线性转发（即 PPS 可以达到或者接近理论最大值）。而基于 Linux 服务器的转发，则容易受网络包大小的影响。</p>
</li>
</ul><!-- [[[read_end]]] --><p>除了这些指标，<strong>网络的可用性</strong>（网络能否正常通信）、<strong>并发连接数</strong>（TCP连接数量）、<strong>丢包率</strong>（丢包百分比）、<strong>重传率</strong>（重新传输的网络包比例）等也是常用的性能指标。</p><p>接下来，请你打开一个终端，SSH登录到服务器上，然后跟我一起来探索、观测这些性能指标。</p><h2><strong>网络配置</strong></h2><p>分析网络问题的第一步，通常是查看网络接口的配置和状态。你可以使用 ifconfig 或者 ip 命令，来查看网络的配置。我个人更推荐使用 ip 工具，因为它提供了更丰富的功能和更易用的接口。</p><blockquote>
<p>ifconfig 和 ip 分别属于软件包 net-tools 和 iproute2，iproute2 是 net-tools 的下一代。通常情况下它们会在发行版中默认安装。但如果你找不到 ifconfig 或者 ip 命令，可以安装这两个软件包。</p>
</blockquote><p>以网络接口 eth0 为例，你可以运行下面的两个命令，查看它的配置和状态：</p><pre><code>$ ifconfig eth0
eth0: flags=4163&lt;UP,BROADCAST,RUNNING,MULTICAST&gt; mtu 1500
      inet 10.240.0.30 netmask 255.240.0.0 broadcast 10.255.255.255
      inet6 fe80::20d:3aff:fe07:cf2a prefixlen 64 scopeid 0x20&lt;link&gt;
      ether 78:0d:3a:07:cf:3a txqueuelen 1000 (Ethernet)
      RX packets 40809142 bytes 9542369803 (9.5 GB)
      RX errors 0 dropped 0 overruns 0 frame 0
      TX packets 32637401 bytes 4815573306 (4.8 GB)
      TX errors 0 dropped 0 overruns 0 carrier 0 collisions 0
​
$ ip -s addr show dev eth0
2: eth0: &lt;BROADCAST,MULTICAST,UP,LOWER_UP&gt; mtu 1500 qdisc mq state UP group default qlen 1000
  link/ether 78:0d:3a:07:cf:3a brd ff:ff:ff:ff:ff:ff
  inet 10.240.0.30/12 brd 10.255.255.255 scope global eth0
      valid_lft forever preferred_lft forever
  inet6 fe80::20d:3aff:fe07:cf2a/64 scope link
      valid_lft forever preferred_lft forever
  RX: bytes packets errors dropped overrun mcast
   9542432350 40809397 0       0       0       193
  TX: bytes packets errors dropped carrier collsns
   4815625265 32637658 0       0       0       0
</code></pre><p>你可以看到，ifconfig 和 ip 命令输出的指标基本相同，只是显示格式略微不同。比如，它们都包括了网络接口的状态标志、MTU 大小、IP、子网、MAC 地址以及网络包收发的统计信息。</p><p>这些具体指标的含义，在文档中都有详细的说明，不过，这里有几个跟网络性能密切相关的指标，需要你特别关注一下。</p><p>第一，网络接口的状态标志。ifconfig 输出中的 RUNNING ，或 ip 输出中的 LOWER_UP ，都表示物理网络是连通的，即网卡已经连接到了交换机或者路由器中。如果你看不到它们，通常表示网线被拔掉了。</p><p>第二，MTU 的大小。MTU 默认大小是 1500，根据网络架构的不同（比如是否使用了 VXLAN 等叠加网络），你可能需要调大或者调小 MTU 的数值。</p><p>第三，网络接口的 IP 地址、子网以及 MAC 地址。这些都是保障网络功能正常工作所必需的，你需要确保配置正确。</p><p>第四，网络收发的字节数、包数、错误数以及丢包情况，特别是 TX 和 RX 部分的 errors、dropped、overruns、carrier 以及 collisions 等指标不为 0 时，通常表示出现了网络 I/O 问题。其中：</p><ul>
<li>
<p>errors 表示发生错误的数据包数，比如校验错误、帧同步错误等；</p>
</li>
<li>
<p>dropped 表示丢弃的数据包数，即数据包已经收到了 Ring Buffer，但因为内存不足等原因丢包；</p>
</li>
<li>
<p>overruns 表示超限数据包数，即网络 I/O 速度过快，导致 Ring Buffer 中的数据包来不及处理（队列满）而导致的丢包；</p>
</li>
<li>
<p>carrier 表示发生 carrirer 错误的数据包数，比如双工模式不匹配、物理电缆出现问题等；</p>
</li>
<li>
<p>collisions 表示碰撞数据包数。</p>
</li>
</ul><h2><strong>套接字信息</strong></h2><p>ifconfig 和 ip 只显示了网络接口收发数据包的统计信息，但在实际的性能问题中，网络协议栈中的统计信息，我们也必须关注。你可以用 netstat 或者 ss ，来查看套接字、网络栈、网络接口以及路由表的信息。</p><p>我个人更推荐，使用 ss 来查询网络的连接信息，因为它比 netstat 提供了更好的性能（速度更快）。</p><p>比如，你可以执行下面的命令，查询套接字信息：</p><pre><code># head -n 3 表示只显示前面3行
# -l 表示只显示监听套接字
# -n 表示显示数字地址和端口(而不是名字)
# -p 表示显示进程信息
$ netstat -nlp | head -n 3
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      840/systemd-resolve

# -l 表示只显示监听套接字
# -t 表示只显示 TCP 套接字
# -n 表示显示数字地址和端口(而不是名字)
# -p 表示显示进程信息
$ ss -ltnp | head -n 3
State    Recv-Q    Send-Q        Local Address:Port        Peer Address:Port
LISTEN   0         128           127.0.0.53%lo:53               0.0.0.0:*        users:((&quot;systemd-resolve&quot;,pid=840,fd=13))
LISTEN   0         128                 0.0.0.0:22               0.0.0.0:*        users:((&quot;sshd&quot;,pid=1459,fd=3))
</code></pre><p>netstat 和 ss 的输出也是类似的，都展示了套接字的状态、接收队列、发送队列、本地地址、远端地址、进程 PID 和进程名称等。</p><p>其中，接收队列（Recv-Q）和发送队列（Send-Q）需要你特别关注，它们通常应该是 0。当你发现它们不是 0 时，说明有网络包的堆积发生。当然还要注意，在不同套接字状态下，它们的含义不同。</p><p>当套接字处于连接状态（Established）时，</p><ul>
<li>
<p>Recv-Q 表示套接字缓冲还没有被应用程序取走的字节数（即接收队列长度）。</p>
</li>
<li>
<p>而 Send-Q 表示还没有被远端主机确认的字节数（即发送队列长度）。</p>
</li>
</ul><p>当套接字处于监听状态（Listening）时，</p><ul>
<li>
<p>Recv-Q 表示全连接队列的长度。</p>
</li>
<li>
<p>而 Send-Q 表示全连接队列的最大长度。</p>
</li>
</ul><p>所谓全连接，是指服务器收到了客户端的 ACK，完成了 TCP 三次握手，然后就会把这个连接挪到全连接队列中。这些全连接中的套接字，还需要被 accept() 系统调用取走，服务器才可以开始真正处理客户端的请求。</p><p>与全连接队列相对应的，还有一个半连接队列。所谓半连接是指还没有完成 TCP 三次握手的连接，连接只进行了一半。服务器收到了客户端的 SYN 包后，就会把这个连接放到半连接队列中，然后再向客户端发送 SYN+ACK 包。</p><h2><strong>协议栈统计信息</strong></h2><p>类似的，使用 netstat 或 ss ，也可以查看协议栈的信息：</p><pre><code>$ netstat -s
...
Tcp:
    3244906 active connection openings
    23143 passive connection openings
    115732 failed connection attempts
    2964 connection resets received
    1 connections established
    13025010 segments received
    17606946 segments sent out
    44438 segments retransmitted
    42 bad segments received
    5315 resets sent
    InCsumErrors: 42
...

$ ss -s
Total: 186 (kernel 1446)
TCP:   4 (estab 1, closed 0, orphaned 0, synrecv 0, timewait 0/0), ports 0

Transport Total     IP        IPv6
*	  1446      -         -
RAW	  2         1         1
UDP	  2         2         0
TCP	  4         3         1
...
</code></pre><p>这些协议栈的统计信息都很直观。ss 只显示已经连接、关闭、孤儿套接字等简要统计，而netstat 则提供的是更详细的网络协议栈信息。</p><p>比如，上面 netstat 的输出示例，就展示了 TCP 协议的主动连接、被动连接、失败重试、发送和接收的分段数量等各种信息。</p><h2><strong>网络吞吐和 PPS</strong></h2><p>接下来，我们再来看看，如何查看系统当前的网络吞吐量和 PPS。在这里，我推荐使用我们的老朋友 sar，在前面的 CPU、内存和 I/O 模块中，我们已经多次用到它。</p><p>给 sar 增加 -n 参数就可以查看网络的统计信息，比如网络接口（DEV）、网络接口错误（EDEV）、TCP、UDP、ICMP 等等。执行下面的命令，你就可以得到网络接口统计信息：</p><pre><code># 数字1表示每隔1秒输出一组数据
$ sar -n DEV 1
Linux 4.15.0-1035 (ubuntu) 	01/06/19 	_x86_64_	(2 CPU)

13:21:40        IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s   %ifutil
13:21:41         eth0     18.00     20.00      5.79      4.25      0.00      0.00      0.00      0.00
13:21:41      docker0      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
13:21:41           lo      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
</code></pre><p>这儿输出的指标比较多，我来简单解释下它们的含义。</p><ul>
<li>
<p>rxpck/s 和 txpck/s 分别是接收和发送的 PPS，单位为包/秒。</p>
</li>
<li>
<p>rxkB/s 和 txkB/s 分别是接收和发送的吞吐量，单位是KB/秒。</p>
</li>
<li>
<p>rxcmp/s 和 txcmp/s 分别是接收和发送的压缩数据包数，单位是包/秒。</p>
</li>
<li>
<p>%ifutil 是网络接口的使用率，即半双工模式下为 (rxkB/s+txkB/s)/Bandwidth，而全双工模式下为 max(rxkB/s, txkB/s)/Bandwidth。</p>
</li>
</ul><p>其中，Bandwidth 可以用 ethtool 来查询，它的单位通常是 Gb/s 或者 Mb/s，不过注意这里小写字母 b ，表示比特而不是字节。我们通常提到的千兆网卡、万兆网卡等，单位也都是比特。如下你可以看到，我的 eth0 网卡就是一个千兆网卡：</p><pre><code>$ ethtool eth0 | grep Speed
	Speed: 1000Mb/s
</code></pre><h2><strong>连通性和延时</strong></h2><p>最后，我们通常使用 ping ，来测试远程主机的连通性和延时，而这基于 ICMP 协议。比如，执行下面的命令，你就可以测试本机到 114.114.114.114 这个 IP 地址的连通性和延时：</p><pre><code># -c3表示发送三次ICMP包后停止
$ ping -c3 114.114.114.114
PING 114.114.114.114 (114.114.114.114) 56(84) bytes of data.
64 bytes from 114.114.114.114: icmp_seq=1 ttl=54 time=244 ms
64 bytes from 114.114.114.114: icmp_seq=2 ttl=47 time=244 ms
64 bytes from 114.114.114.114: icmp_seq=3 ttl=67 time=244 ms

--- 114.114.114.114 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2001ms
rtt min/avg/max/mdev = 244.023/244.070/244.105/0.034 ms
</code></pre><p>ping 的输出，可以分为两部分。</p><ul>
<li>
<p>第一部分，是每个 ICMP 请求的信息，包括 ICMP 序列号（icmp_seq）、TTL（生存时间，或者跳数）以及往返延时。</p>
</li>
<li>
<p>第二部分，则是三次 ICMP 请求的汇总。</p>
</li>
</ul><p>比如上面的示例显示，发送了 3 个网络包，并且接收到 3 个响应，没有丢包发生，这说明测试主机到 114.114.114.114 是连通的；平均往返延时（RTT）是 244ms，也就是从发送 ICMP 开始，到接收到 114.114.114.114 回复的确认，总共经历 244ms。</p><h2>小结</h2><p>我们通常使用带宽、吞吐量、延时等指标，来衡量网络的性能；相应的，你可以用 ifconfig、netstat、ss、sar、ping 等工具，来查看这些网络的性能指标。</p><p>在下一节中，我将以经典的 C10K 和 C100K 问题，带你进一步深入 Linux 网络的工作原理。</p><h2>思考</h2><p>最后，我想请你来聊聊，你理解的 Linux 网络性能。你常用什么指标来衡量网络的性能？又用什么思路分析相应性能问题呢？你可以结合今天学到的知识，提出自己的观点。</p><p>欢迎在留言区和我讨论，也欢迎你把这篇文章分享给你的同事、朋友。我们一起在实战中演练，在交流中进步。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/60/71/895ee6cf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>分清云淡</span>
  </div>
  <div class="_2_QraFYR_0">小狗同学问到： 老师，您好 ss —lntp 这个 当session处于listening中 rec-q 确定是 syn的backlog吗？  <br>A:  Recv-Q为全连接队列当前使用了多少。 中文资料里这个问题讲得最明白的文章：https:&#47;&#47;mp.weixin.qq.com&#47;s&#47;yH3PzGEFopbpA-jw4MythQ</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍谢谢分享</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-01 13:41:59</div>
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
  <div class="_2_QraFYR_0">今天查一个问题，花了半天功夫。<br>如果文中的一句话还记得的话，可能就只需要几分钟了。<br><br>而 Send-Q 表示还没有被远端主机确认的字节数（即发送队列长度）。<br><br>原本ngnix在物理机上是没问题的，结果有个环境中放到了docker里，且还是走的普通端口映射模式，还不是—net=host模式，结果websocket发往ngnix的数据过大后就被阻塞了。<br>最后发现相应被阻塞的端口 send-q一直很大。<br>最后docker尝试使用net=host后故障消除。<br>具体的原因还没有细究。<br><br>留个言，给后面看专栏的同学一个教训吧。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-21 23:34:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/47/e7/53416498.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>曹龙飞</span>
  </div>
  <div class="_2_QraFYR_0">看了源码发现，这个地方讲的有问题.关于ss输出中listen状态套接字的Recv-Q表示全连接队列当前使用了多少,也就是全连接队列的当前长度,而Send-Q表示全连接队列的最大长度</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-09 18:56:30</div>
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
  <div class="_2_QraFYR_0">老师春节不休息，大赞啊，老师可否讲解一下一个包从网卡接收，发送在内核协议栈的整个流程，这样性能分析的时候，更好的理解数据包阻塞在哪里？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这些在后面的案例中会涉及</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-09 08:42:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/e0/4e/c266bdb4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>[小狗]</span>
  </div>
  <div class="_2_QraFYR_0">老师，您好  ss —lntp  这个  当session处于listening中  rec-q  确定是  syn的backlog吗？  我之前都是当做全队列的长度</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-09 22:53:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/8OPzdpDraQMvCNWAicicDt54sDaIYJZicBLfMyibXVs4V0ZibEdkZlbzxxL7aGpRoeyvibag5LaAaaGKSdwYQMY2hUrQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>code2</span>
  </div>
  <div class="_2_QraFYR_0">每期读两遍，看看别人怎么做!</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-10 12:00:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/1e/86/33ccfeb7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Aoser</span>
  </div>
  <div class="_2_QraFYR_0">老师，listening状态下，Recv-Q和Send-Q应该分别指的是，当前全连接的使用数量和最大全连接可用数量，但是文中说成了是半连接。参考阿里技术博客：http:&#47;&#47;jm.taobao.org&#47;2017&#47;05&#47;25&#47;525-1&#47;</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-10 12:55:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/u8g8UoUaBZTGpDgAQKDwL5IzicBILfpDmz0tOXibSQWkmTjr3m57ofUKVPgRiaFTRYZkE7dBxOmmJBhqCpYJWd2GA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>mfj1st</span>
  </div>
  <div class="_2_QraFYR_0">倪老师，虚拟机用ethtool获取不到网卡带宽，有其他方法吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-29 20:16:58</div>
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
  <div class="_2_QraFYR_0">[D34打卡]<br>平常只用netstat 和 ifconfig ，前面的专栏里学了sar观测网络指标，今天又接触了两个类似的：ss和ip。<br>平常遇到的网络问题比较简单，先看能否正常连上，再看看并发连接数。有时忘记执行ulimit -n会导致默认账号的一个进程同时打开文件数只有1024。<br>除了带宽没买够，平常不会遇到网络方面的瓶颈，毕竟我们的业务处理数据的消耗比收发数据的消耗大得多。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-09 15:00:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/a1/29/ac0f2c83.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>芥菜</span>
  </div>
  <div class="_2_QraFYR_0">春节期间终于跟上节奏，春节里做到只长知识不长肉：）</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-08 15:42:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/e2/4f/0748c63e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Gaoyc</span>
  </div>
  <div class="_2_QraFYR_0">通过ifconfig和ss看到的错误包或丢弃包等的一些错误是累加的嘛？是否可以清空这些错误包信息？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，都是累加值，所以不建议清空这些统计信息。并且，真正要清的化，也需要停止网卡并且卸载（rmmod）网卡内核模块，这在实际环境中通常是不允许的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-22 08:10:39</div>
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
  <div class="_2_QraFYR_0">老师， 网卡  dropped  不为零，而且还在增加，怎么排查问题哦？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-15 16:54:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/a0/b7/1327ae60.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>hellojd_gk</span>
  </div>
  <div class="_2_QraFYR_0">在阿里云ECS上执行如下命令，怎么啥也不返回<br>[root@iZ8vb31waukedizc39ffofZ ~]# ethtool eth0<br>Settings for eth0:<br>        Link detected: yes</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-29 18:10:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/15/69/187b9968.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>南山</span>
  </div>
  <div class="_2_QraFYR_0">有点看到本质的感觉，赞</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-29 16:27:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/76/93/64ed7385.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>arch</span>
  </div>
  <div class="_2_QraFYR_0">&lt;b&gt;ethtool eth0 | grep Speed&lt;&#47;b&gt; 这个命令不管用，查不出带宽大小？ </div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-26 00:45:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKq0oQVibKcmYJqmpqaNNQibVgia7EsEgW65LZJIpDZBMc7FyMcs7J1JmFCtp06pY8ibbcpW4ibRtG7Frg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>zhoufeng</span>
  </div>
  <div class="_2_QraFYR_0">老师好，再请教一个问题，查看max_syn_backlog值为2048，表示同时最大只能接受2048个请求吗？<br># cat &#47;proc&#47;sys&#47;net&#47;ipv4&#47;tcp_max_syn_backlog <br>2048<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不是，这只是半连接的最大数量</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-21 00:21:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKq0oQVibKcmYJqmpqaNNQibVgia7EsEgW65LZJIpDZBMc7FyMcs7J1JmFCtp06pY8ibbcpW4ibRtG7Frg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>zhoufeng</span>
  </div>
  <div class="_2_QraFYR_0">老师好，<br>“当套接字处于listen状态时，使用ss命令看到的Send-Q表示最大的syn_backlog值”<br>但是和&#47;proc&#47;sys&#47;net&#47;ipv4&#47;tcp_max_syn_backlog   查看到的值不一致，是我理解错了吗？<br>谢谢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 前一个是套接字级的，后一个是系统级的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-21 00:17:48</div>
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
  <div class="_2_QraFYR_0">老师，有一事疑惑，希望帮忙解惑。<br><br>一台64个千兆端口的交换机，全双工模式，交换容量计算：64*1000*2<br>包转发速率计算：64*1.488Mpp<br><br>交换容量有个乘以2，为什么包转发速率不需要乘以2？<br>（我理解，端口速率1000bps，在全双工模式下，按照双向速率计算，即总速率是2000bps）</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯嗯，说转发性能的时候一般都是指一个方向的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-12 09:43:31</div>
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
  <div class="_2_QraFYR_0">eth0: flags=4163<br>这个什么意思，有点好奇</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 网络接口的一些标志，含义在尖括号中</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-09 22:06:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/f5/a0/c342c50e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>bzadhere</span>
  </div>
  <div class="_2_QraFYR_0">netstat -nta  命令看到Listening状态下的Send-Q 值都是0，用man netstat 看到说明和实际情况不一样； 然后用ss -lnt 看到Send-Q  非0，应该怎么理解？<br><br>[root@localhost ~]# man netstat<br>......<br>   Recv-Q<br>       Established:  The count of bytes not copied by the user program connected to this socket.  Listening: Since<br>       Kernel 2.6.18 this column contains the current syn backlog.<br><br>   Send-Q<br>       Established: The count of bytes not acknowledged by the remote host.  Listening: Since Kernel  2.6.18  this<br>       column contains the maximum size of the syn backlog.<br>.......<br><br>[root@localhost ~]# netstat -tna<br>Active Internet connections (servers and established)<br>Proto Recv-Q Send-Q Local Address           Foreign Address         State      <br>tcp        0      0 0.0.0.0:21              0.0.0.0:*               LISTEN     <br>tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN     <br>tcp        0      0 127.0.0.1:25            0.0.0.0:*               LISTEN     <br>tcp        0      0 192.168.137.20:22       192.168.137.1:52521     ESTABLISHED<br>tcp6       0      0 :::22                   :::*                    LISTEN     <br>tcp6       0      0 ::1:25                  :::*                    LISTEN     <br><br>[root@localhost ~]# ss -lnt<br>State       Recv-Q Send-Q             Local Address:Port                            Peer Address:Port              <br>LISTEN      0      32                             *:21                                         *:*                  <br>LISTEN      0      128                            *:22                                         *:*                  <br>LISTEN      0      100                    127.0.0.1:25                                         *:*                  <br>LISTEN      0      128                           :::22                                        :::*                  <br>LISTEN      0      100                          ::1:25                                        :::*  </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可能是版本问题，可以查查 ss 的 manual 上含义是一样的吗</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-07 10:52:42</div>
  </div>
</div>
</div>
</li>
</ul>