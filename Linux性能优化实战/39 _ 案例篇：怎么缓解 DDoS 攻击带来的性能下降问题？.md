<audio title="39 _ 案例篇：怎么缓解 DDoS 攻击带来的性能下降问题？" src="https://static001.geekbang.org/resource/audio/15/c6/15eb08485d456e1660aa901e5fb046c6.mp3" controls="controls"></audio> 
<p>你好，我是倪朋飞。</p><p>上一节，我带你学习了tcpdump 和 Wireshark 的使用方法，并通过几个案例，带你用这两个工具实际分析了网络的收发过程。碰到网络性能问题，不要忘记可以用 tcpdump 和 Wireshark 这两个大杀器，抓取实际传输的网络包，排查潜在的性能问题。</p><p>今天，我们一起来看另外一个问题，怎么缓解 DDoS（Distributed Denial of Service）带来的性能下降问题。</p><h2>DDoS 简介</h2><p>DDoS 的前身是 DoS（Denail of Service），即拒绝服务攻击，指利用大量的合理请求，来占用过多的目标资源，从而使目标服务无法响应正常请求。</p><p>DDoS（Distributed Denial of Service） 则是在 DoS 的基础上，采用了分布式架构，利用多台主机同时攻击目标主机。这样，即使目标服务部署了网络防御设备，面对大量网络请求时，还是无力应对。</p><p>比如，目前已知的最大流量攻击，正是去年 Github 遭受的 <a href="https://githubengineering.com/ddos-incident-report/">DDoS 攻击</a>，其峰值流量已经达到了 1.35Tbps，PPS 更是超过了 1.2 亿（126.9 million）。</p><p>从攻击的原理上来看，DDoS 可以分为下面几种类型。</p><p>第一种，耗尽带宽。无论是服务器还是路由器、交换机等网络设备，带宽都有固定的上限。带宽耗尽后，就会发生网络拥堵，从而无法传输其他正常的网络报文。</p><!-- [[[read_end]]] --><p>第二种，耗尽操作系统的资源。网络服务的正常运行，都需要一定的系统资源，像是CPU、内存等物理资源，以及连接表等软件资源。一旦资源耗尽，系统就不能处理其他正常的网络连接。</p><p>第三种，消耗应用程序的运行资源。应用程序的运行，通常还需要跟其他的资源或系统交互。如果应用程序一直忙于处理无效请求，也会导致正常请求的处理变慢，甚至得不到响应。</p><p>比如，构造大量不同的域名来攻击 DNS 服务器，就会导致 DNS 服务器不停执行迭代查询，并更新缓存。这会极大地消耗 DNS 服务器的资源，使 DNS 的响应变慢。</p><p>无论是哪一种类型的 DDoS，危害都是巨大的。那么，如何可以发现系统遭受了 DDoS 攻击，又该如何应对这种攻击呢？接下来，我们就通过一个案例，一起来看看这些问题。</p><h2>案例准备</h2><p>下面的案例仍然基于 Ubuntu 18.04，同样适用于其他的 Linux 系统。我使用的案例环境是这样的：</p><ul>
<li>
<p>机器配置：2 CPU，8GB 内存。</p>
</li>
<li>
<p>预先安装 docker、sar 、hping3、tcpdump、curl 等工具，比如 apt-get install docker.io hping3 tcpdump curl。</p>
</li>
</ul><p>这些工具你应该都比较熟悉了。其中，hping3 在 <a href="https://time.geekbang.org/column/article/72147">系统的软中断CPU使用率升高案例</a> 中曾经介绍过，它可以构造 TCP/IP 协议数据包，对系统进行安全审计、防火墙测试、DoS 攻击测试等。</p><p>本次案例用到三台虚拟机，我画了一张图来表示它们之间的关系。</p><p><img src="https://static001.geekbang.org/resource/image/d6/12/d64dd4603a4bd90d110f382d313d8c12.png?wh=826*720" alt=""></p><p>你可以看到，其中一台虚拟机运行 Nginx ，用来模拟待分析的 Web 服务器；而另外两台作为 Web 服务器的客户端，其中一台用作 DoS 攻击，而另一台则是正常的客户端。使用多台虚拟机的目的，自然还是为了相互隔离，避免“交叉感染”。</p><blockquote>
<p>由于案例只使用了一台机器作为攻击源，所以这里的攻击，实际上还是传统的 DoS ，而非 DDoS。</p>
</blockquote><p>接下来，我们打开三个终端，分别 SSH 登录到三台机器上（下面的步骤，都假设终端编号与图示VM 编号一致），并安装上面提到的这些工具。</p><p>同以前的案例一样，下面的所有命令，都默认以 root 用户运行。如果你是用普通用户身份登陆系统，请运行 sudo su root 命令切换到 root 用户。</p><p>接下来，我们就进入到案例操作环节。</p><h2>案例分析</h2><p>首先，在终端一中，执行下面的命令运行案例，也就是启动一个最基本的 Nginx 应用：</p><pre><code># 运行Nginx服务并对外开放80端口
# --network=host表示使用主机网络（这是为了方便后面排查问题）
$ docker run -itd --name=nginx --network=host nginx
</code></pre><p>然后，在终端二和终端三中，使用 curl 访问 Nginx 监听的端口，确认 Nginx 正常启动。假设 192.168.0.30 是 Nginx 所在虚拟机的 IP 地址，那么运行 curl 命令后，你应该会看到下面这个输出界面：</p><pre><code># -w表示只输出HTTP状态码及总时间，-o表示将响应重定向到/dev/null
$ curl -s -w 'Http code: %{http_code}\nTotal time:%{time_total}s\n' -o /dev/null http://192.168.0.30/
...
Http code: 200
Total time:0.002s
</code></pre><p>从这里可以看到，正常情况下，我们访问 Nginx 只需要 2ms（0.002s）。</p><p>接着，在终端二中，运行 hping3 命令，来模拟 DoS 攻击：</p><pre><code># -S参数表示设置TCP协议的SYN（同步序列号），-p表示目的端口为80
# -i u10表示每隔10微秒发送一个网络帧
$ hping3 -S -p 80 -i u10 192.168.0.30
</code></pre><p>现在，再回到终端一，你就会发现，现在不管执行什么命令，都慢了很多。不过，在实践时要注意：</p><ul>
<li>
<p>如果你的现象不那么明显，那么请尝试把参数里面的 u10 调小（比如调成 u1），或者加上–flood选项；</p>
</li>
<li>
<p>如果你的终端一完全没有响应了，那么请适当调大 u10（比如调成 u30），否则后面就不能通过 SSH 操作 VM1。</p>
</li>
</ul><p>然后，到终端三中，执行下面的命令，模拟正常客户端的连接：</p><pre><code># --connect-timeout表示连接超时时间
$ curl -w 'Http code: %{http_code}\nTotal time:%{time_total}s\n' -o /dev/null --connect-timeout 10 http://192.168.0.30
...
Http code: 000
Total time:10.001s
curl: (28) Connection timed out after 10000 milliseconds
</code></pre><p>你可以发现，在终端三中，正常客户端的连接超时了，并没有收到 Nginx 服务的响应。</p><p>这是发生了什么问题呢？我们再回到终端一中，检查网络状况。你应该还记得我们多次用过的 sar，它既可以观察 PPS（每秒收发的报文数），还可以观察 BPS（每秒收发的字节数）。</p><p>我们可以回到终端一中，执行下面的命令：</p><pre><code>$ sar -n DEV 1
08:55:49        IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s   %ifutil
08:55:50      docker0      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
08:55:50         eth0  22274.00    629.00   1174.64     37.78      0.00      0.00      0.00      0.02
08:55:50           lo      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
</code></pre><p>关于 sar 输出中的各列含义，我在前面的 Linux 网络基础中已经介绍过，你可以点击 <a href="https://time.geekbang.org/column/article/81057">这里</a> 查看，或者执行 man sar 查询手册。</p><p>从这次 sar 的输出中，你可以看到，网络接收的 PPS 已经达到了 20000 多，但是 BPS 却只有 1174 kB，这样每个包的大小就只有 54B（1174*1024/22274=54）。</p><p>这明显就是个小包了，不过具体是个什么样的包呢？那我们就用 tcpdump 抓包看看吧。</p><p>在终端一中，执行下面的 tcpdump 命令：</p><pre><code># -i eth0 只抓取eth0网卡，-n不解析协议名和主机名
# tcp port 80表示只抓取tcp协议并且端口号为80的网络帧
$ tcpdump -i eth0 -n tcp port 80
09:15:48.287047 IP 192.168.0.2.27095 &gt; 192.168.0.30: Flags [S], seq 1288268370, win 512, length 0
09:15:48.287050 IP 192.168.0.2.27131 &gt; 192.168.0.30: Flags [S], seq 2084255254, win 512, length 0
09:15:48.287052 IP 192.168.0.2.27116 &gt; 192.168.0.30: Flags [S], seq 677393791, win 512, length 0
09:15:48.287055 IP 192.168.0.2.27141 &gt; 192.168.0.30: Flags [S], seq 1276451587, win 512, length 0
09:15:48.287068 IP 192.168.0.2.27154 &gt; 192.168.0.30: Flags [S], seq 1851495339, win 512, length 0
...
</code></pre><p>这个输出中，Flags [S] 表示这是一个 SYN 包。大量的 SYN 包表明，这是一个 SYN Flood 攻击。如果你用上一节讲过的 Wireshark 来观察，则可以更直观地看到 SYN Flood 的过程：</p><p><img src="https://static001.geekbang.org/resource/image/f3/13/f397305c87be6ae43e065d3262ec9113.png?wh=1048*574" alt=""></p><p>实际上，SYN Flood  正是互联网中最经典的 DDoS 攻击方式。从上面这个图，你也可以看到它的原理：</p><ul>
<li>
<p>即客户端构造大量的 SYN 包，请求建立 TCP 连接；</p>
</li>
<li>
<p>而服务器收到包后，会向源 IP 发送 SYN+ACK 报文，并等待三次握手的最后一次ACK报文，直到超时。</p>
</li>
</ul><p>这种等待状态的 TCP 连接，通常也称为半开连接。由于连接表的大小有限，大量的半开连接就会导致连接表迅速占满，从而无法建立新的 TCP 连接。</p><p>参考下面这张 TCP 状态图，你能看到，此时，服务器端的 TCP 连接，会处于 SYN_RECEIVED 状态：</p><p><img src="https://static001.geekbang.org/resource/image/86/a2/86dabf9cc66b29133fa6a239cfee38a2.png?wh=796*600" alt=""></p><p>（图片来自 <a href="https://en.wikipedia.org/wiki/File:Tcp_state_diagram.png">Wikipedia</a>）</p><p>这其实提示了我们，查看 TCP 半开连接的方法，关键在于 SYN_RECEIVED 状态的连接。我们可以使用 netstat ，来查看所有连接的状态，不过要注意，SYN_REVEIVED 的状态，通常被缩写为 SYN_RECV。</p><p>我们继续在终端一中，执行下面的 netstat 命令：</p><pre><code># -n表示不解析名字，-p表示显示连接所属进程
$ netstat -n -p | grep SYN_REC
tcp        0      0 192.168.0.30:80          192.168.0.2:12503      SYN_RECV    -
tcp        0      0 192.168.0.30:80          192.168.0.2:13502      SYN_RECV    -
tcp        0      0 192.168.0.30:80          192.168.0.2:15256      SYN_RECV    -
tcp        0      0 192.168.0.30:80          192.168.0.2:18117      SYN_RECV    -
...
</code></pre><p>从结果中，你可以发现大量 SYN_RECV 状态的连接，并且源IP地址为 192.168.0.2。</p><p>进一步，我们还可以通过 wc 工具，来统计所有 SYN_RECV 状态的连接数：</p><pre><code>$ netstat -n -p | grep SYN_REC | wc -l
193
</code></pre><p>找出源 IP 后，要解决 SYN 攻击的问题，只要丢掉相关的包就可以。这时，iptables 可以帮你完成这个任务。你可以在终端一中，执行下面的 iptables 命令：</p><pre><code>$ iptables -I INPUT -s 192.168.0.2 -p tcp -j REJECT
</code></pre><p>然后回到终端三中，再次执行 curl 命令，查看正常用户访问 Nginx 的情况：</p><pre><code>$ curl -w 'Http code: %{http_code}\nTotal time:%{time_total}s\n' -o /dev/null --connect-timeout 10 http://192.168.0.30
Http code: 200
Total time:1.572171s
</code></pre><p>现在，你可以发现，正常用户也可以访问 Nginx 了，只是响应比较慢，从原来的 2ms 变成了现在的 1.5s。</p><p>不过，一般来说，SYN Flood 攻击中的源 IP 并不是固定的。比如，你可以在 hping3 命令中，加入 --rand-source 选项，来随机化源 IP。不过，这时，刚才的方法就不适用了。</p><p>幸好，我们还有很多其他方法，实现类似的目标。比如，你可以用以下两种方法，来限制 syn 包的速率：</p><pre><code># 限制syn并发数为每秒1次
$ iptables -A INPUT -p tcp --syn -m limit --limit 1/s -j ACCEPT

# 限制单个IP在60秒新建立的连接数为10
$ iptables -I INPUT -p tcp --dport 80 --syn -m recent --name SYN_FLOOD --update --seconds 60 --hitcount 10 -j REJECT
</code></pre><p>到这里，我们已经初步限制了 SYN Flood 攻击。不过这还不够，因为我们的案例还只是单个的攻击源。</p><p>如果是多台机器同时发送 SYN Flood，这种方法可能就直接无效了。因为你很可能无法 SSH 登录（SSH 也是基于 TCP 的）到机器上去，更别提执行上述所有的排查命令。</p><p>所以，这还需要你事先对系统做一些 TCP 优化。</p><p>比如，SYN Flood 会导致 SYN_RECV 状态的连接急剧增大。在上面的 netstat 命令中，你也可以看到 190 多个处于半开状态的连接。</p><p>不过，半开状态的连接数是有限制的，执行下面的命令，你就可以看到，默认的半连接容量只有 256：</p><pre><code>$ sysctl net.ipv4.tcp_max_syn_backlog
net.ipv4.tcp_max_syn_backlog = 256
</code></pre><p>换句话说， SYN 包数再稍微增大一些，就不能 SSH 登录机器了。 所以，你还应该增大半连接的容量，比如，你可以用下面的命令，将其增大为 1024：</p><pre><code>$ sysctl -w net.ipv4.tcp_max_syn_backlog=1024
net.ipv4.tcp_max_syn_backlog = 1024
</code></pre><p>另外，连接每个 SYN_RECV 时，如果失败的话，内核还会自动重试，并且默认的重试次数是5次。你可以执行下面的命令，将其减小为 1 次：</p><pre><code>$ sysctl -w net.ipv4.tcp_synack_retries=1
net.ipv4.tcp_synack_retries = 1
</code></pre><p>除此之外，TCP SYN Cookies 也是一种专门防御 SYN Flood 攻击的方法。SYN Cookies 基于连接信息（包括源地址、源端口、目的地址、目的端口等）以及一个加密种子（如系统启动时间），计算出一个哈希值（SHA1），这个哈希值称为 cookie。</p><p>然后，这个 cookie 就被用作序列号，来应答 SYN+ACK 包，并释放连接状态。当客户端发送完三次握手的最后一次 ACK 后，服务器就会再次计算这个哈希值，确认是上次返回的 SYN+ACK 的返回包，才会进入 TCP 的连接状态。</p><p>因而，开启 SYN Cookies 后，就不需要维护半开连接状态了，进而也就没有了半连接数的限制。</p><blockquote>
<p>注意，开启 TCP syncookies 后，内核选项 net.ipv4.tcp_max_syn_backlog 也就无效了。</p>
</blockquote><p>你可以通过下面的命令，开启 TCP SYN Cookies：</p><pre><code>$ sysctl -w net.ipv4.tcp_syncookies=1
net.ipv4.tcp_syncookies = 1
</code></pre><p>注意，上述 sysctl 命令修改的配置都是临时的，重启后这些配置就会丢失。所以，为了保证配置持久化，你还应该把这些配置，写入 /etc/sysctl.conf 文件中。比如：</p><pre><code>$ cat /etc/sysctl.conf
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_synack_retries = 1
net.ipv4.tcp_max_syn_backlog = 1024
</code></pre><p>不过要记得，写入 /etc/sysctl.conf 的配置，需要执行 sysctl -p 命令后，才会动态生效。</p><p>当然案例结束后，别忘了执行 docker rm -f nginx 命令，清理案例开始时启动的 Nginx 应用。</p><h2>DDoS到底该怎么防御</h2><p>到这里，今天的案例就结束了。不过，你肯定还有疑问。你应该注意到了，今天的主题是“缓解”，而不是“解决” DDoS 问题。</p><p>为什么不是解决 DDoS ，而只是缓解呢？而且今天案例中的方法，也只是让 Nginx 服务访问不再超时，但访问延迟还是比一开始时的 2ms 大得多。</p><p>实际上，当 DDoS 报文到达服务器后，Linux 提供的机制只能缓解，而无法彻底解决。即使像是 SYN Flood 这样的小包攻击，其巨大的 PPS ，也会导致 Linux 内核消耗大量资源，进而导致其他网络报文的处理缓慢。</p><p>虽然你可以调整内核参数，缓解 DDoS 带来的性能问题，却也会像案例这样，无法彻底解决它。</p><p>在之前的 <a href="https://time.geekbang.org/column/article/81268">C10K、C100K</a> <a href="https://time.geekbang.org/column/article/81268">文章</a>  中，我也提到过，Linux 内核中冗长的协议栈，在 PPS 很大时，就是一个巨大的负担。对 DDoS 攻击来说，也是一样的道理。</p><p>所以，当时提到的 C10M 的方法，用到这里同样适合。比如，你可以基于 XDP 或者 DPDK，构建 DDoS 方案，在内核网络协议栈前，或者跳过内核协议栈，来识别并丢弃 DDoS 报文，避免DDoS 对系统其他资源的消耗。</p><p>不过，对于流量型的 DDoS 来说，当服务器的带宽被耗尽后，在服务器内部处理就无能为力了。这时，只能在服务器外部的网络设备中，设法识别并阻断流量（当然前提是网络设备要能扛住流量攻击）。比如，购置专业的入侵检测和防御设备，配置流量清洗设备阻断恶意流量等。</p><p>既然 DDoS 这么难防御，这是不是说明， Linux 服务器内部压根儿就不关注这一点，而是全部交给专业的网络设备来处理呢？</p><p>当然不是，因为 DDoS 并不一定是因为大流量或者大 PPS，有时候，慢速的请求也会带来巨大的性能下降（这种情况称为慢速 DDoS）。</p><p>比如，很多针对应用程序的攻击，都会伪装成正常用户来请求资源。这种情况下，请求流量可能本身并不大，但响应流量却可能很大，并且应用程序内部也很可能要耗费大量资源处理。</p><p>这时，就需要应用程序考虑识别，并尽早拒绝掉这些恶意流量，比如合理利用缓存、增加 WAF（Web Application Firewall）、使用 CDN 等等。</p><h2>小结</h2><p>今天，我们学习了分布式拒绝服务（DDoS）时的缓解方法。DDoS 利用大量的伪造请求，使目标服务耗费大量资源，来处理这些无效请求，进而无法正常响应正常的用户请求。</p><p>由于 DDoS 的分布式、大流量、难追踪等特点，目前还没有方法可以完全防御 DDoS 带来的问题，只能设法缓解这个影响。</p><p>比如，你可以购买专业的流量清洗设备和网络防火墙，在网络入口处阻断恶意流量，只保留正常流量进入数据中心的服务器中。</p><p>在 Linux 服务器中，你可以通过内核调优、DPDK、XDP 等多种方法，来增大服务器的抗攻击能力，降低 DDoS 对正常服务的影响。而在应用程序中，你可以利用各级缓存、 WAF、CDN 等方式，缓解 DDoS 对应用程序的影响。</p><h2>思考</h2><p>最后给你留一个思考题。</p><p>看到今天的案例，你可能会觉得眼熟。实际上，它正是在 <a href="https://time.geekbang.org/column/article/72147">系统的软中断CPU使用率升高案例</a> 基础上扩展而来的。当时，我们是从软中断 CPU 使用率的角度来分析的，也就是说，DDoS 会导致软中断 CPU 使用率（softirq）升高。</p><p>回想一下当时的案例和分析思路，再结合今天的案例，你觉得还有没有更好的方法，来检测 DDoS 攻击呢？除了 tcpdump，还有哪些方法查找这些攻击的源地址？</p><p>欢迎在留言区和我讨论，也欢迎把这篇文章分享给你的同事、朋友。我们一起在实战中演练，在交流中进步。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/5c/cb/3ebdcc49.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>怀特</span>
  </div>
  <div class="_2_QraFYR_0">这个专栏太超值，我跟追剧一样的追到现在，收获已经巨大！<br>谢谢倪工的分享。<br>更谢谢倪工对留言一丝不苟的回复---这份对听众的耐心，其他专栏的作者没有一个能比得上的。<br>继续追剧！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 哈哈，谢谢热心回复和支持</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-21 11:14:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4a/15/106eaaa8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>stackWarn</span>
  </div>
  <div class="_2_QraFYR_0">讲的很不错。补充一点，半连接状态不只这一个参数net.ipv4.tcp_max_syn_backlog 控制，实际是 这个，还有系统somaxcon，以及应用程序的backlog，三个一起控制的。具体可以看下相关的源码</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-10 14:31:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/92/e3/1be13204.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ichiro</span>
  </div>
  <div class="_2_QraFYR_0">最近服务会出现干核现象，一个单进程程序发现把一个cpu耗尽，用top发现一个cpu的，软中断很高，通过watch -d cat &#47;proc&#47;softireq，发现网络中断很高，然后配置多网卡队列，把中断分散到是他cpu，缓解了进程cpu的压力，但同时带来担忧，配置多网卡队列绑定，会不会带来cpu切换，缓存失效等负面影响？另外，老师还有别的建议吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 会的，实际上优化网络都会占用更多的cpu和内存。所以还要看优化是不是值得，是不是最需要优化的瓶颈</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-27 21:15:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/4d/49/28e73b9c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>明翼</span>
  </div>
  <div class="_2_QraFYR_0">检测DDOS攻击，我没有什么这方面经验想了下：<br>1、既然攻击，肯定不是正常业务，所以在sar -n DEV 4 命令看运行包的时候，不光要注意收到的包的大小，还要注意平时在监控业务的时候观察正常的业务的收包量，做一个横向比较，确定异常流量；还有一点特别重要，我觉得tcp连接交互基本都是双向的，那么回包数量和发包的数量不能相差太大，如果太大了可能有问题。<br><br>2、至于查看源IP，除了tcpdump外，是不是可以通过netstat 结合wc 统计下各类状态的连接数，如果连接数超过平时的连接数就要注意，关注状态一致的连接数，这里面也有ip信息，当然也可以判断来源。<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，sar和netstat都是最常用的工具</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-20 22:54:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/cb/88/f9ae7bc9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>李燕平²⁰¹⁸</span>
  </div>
  <div class="_2_QraFYR_0">服务器插上网线就像开车上路。<br><br>DDoS攻击可以理解为撞车，提升汽车本身的安全系数会有用。<br>大量DDoS攻击可以理解为撞上大货车，需要护卫车队来保护主车。<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-20 07:55:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/3f/0d/1e8dbb2c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>怀揣梦想的学渣</span>
  </div>
  <div class="_2_QraFYR_0">我观摩过我所在城市的部分学校以及部分教育部门网站的DDOS防护，他们的方案是单个IP发来访问请求，会自动跳转到一个检测的界面，如果这个IP无正常互动响应，直接拉黑1个月，如果想解封，需要自己实名认证发邮件到教育部门的咨询邮箱申请。这个办法是真的挺神奇的。有些好奇，逻辑上外层加了一些防火墙或者防护设备，那内部的服务器还需要配置网络防护的策略吗。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-06 09:34:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/8c/9c/d48473ab.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>dancer</span>
  </div>
  <div class="_2_QraFYR_0">重新开追，前两天着重看了cpu调优，正好线上压测发现了cpu.user飙高的问题，通过perf和pprof查明了是一个复杂json结构解析导致的，学以致用下来印象更深了！这两天开始看网络，后面再看io和内存相关，感谢！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞，很高兴也看到学以致用</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-06 19:58:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/dd/5b/5461afad.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Wen</span>
  </div>
  <div class="_2_QraFYR_0">DDoS简介<br>DDoS（Distributied）的前身是DoS（Denail of Service），即拒绝服务攻击，指利用大量的合理请求，来占用过多的目标资源，从而使目标服务无法响应正常的请求。<br>DDos在Dos的基础上，采用了分布式架构，利用多台主机同时攻击目标主机。<br>DDoS有以下三种类型：<br>a、耗尽宽带。<br>b、消耗操作系统的资源。<br>c、消耗应用程序的运行资源。<br><br>DoS实验模拟：<br>VM1终端一：开启web服务<br> docker run -itd --name=nginx --network=host nginx<br>VM2终端一：对比被攻击前后的web访问时间：<br>curl -s -w &#39;Http code: %{http_code}\nTotal time:%{time_total}s\n&#39; -o &#47;dev&#47;null http:&#47;&#47;10.10.10.2&#47;<br>VM2终端二：模拟Dos攻击<br>hping3 -S -p 80 -i U10 --flood 10.10.10.2<br>VM1终端二：查看网络接收包情况<br>sar -n DEV 1<br>VM1终端三：tcpdump抓包<br>tcpdump -i eth0 -n tcp port 80<br>VM1终端4：统计SYN_REC连接数<br>netstat -n -p | grep SYN_REC | wc -l<br><br><br>解决Dos攻击：<br>防火墙方法1：拒绝源ip访问<br>iptables -I INPUT -s 10.10.10.4 -p tcp -j REJECT<br>防火墙方法2：限制 syn 包的速率<br># 限制syn并发数为每秒1次<br>$ iptables -A INPUT -p tcp --syn -m limit --limit 1&#47;s -j ACCEPT<br># 限制单个IP在60秒新建立的连接数为10<br>$ iptables -I INPUT -p tcp --dport 80 --syn -m recent --name SYN_FLOOD --update --seconds 60 --hitcount 10 -j REJECT<br>优化系统内核TCP参数<br>方法1：增大syn连接数<br>$ sysctl net.ipv4.tcp_max_syn_backlog<br>net.ipv4.tcp_max_syn_backlog = 256<br>$ sysctl -w net.ipv4.tcp_max_syn_backlog=1024<br>net.ipv4.tcp_max_syn_backlog = 1024<br>方法2：减少SYN_REVC重试次数为1，默认是5<br>$ sysctl -w net.ipv4.tcp_synack_retries=1<br>net.ipv4.tcp_synack_retries = 1<br>方法3：开启TCP SYN Cookies<br>注意：开启 TCP syncookies 后，内核选项 net.ipv4.tcp_max_syn_backlog 也就无效了。<br>$ sysctl -w net.ipv4.tcp_syncookies=1<br>net.ipv4.tcp_syncookies = 1<br><br>永久生效方式：写入 &#47;etc&#47;sysctl.conf 文件中：<br>$ cat &#47;etc&#47;sysctl.conf<br>net.ipv4.tcp_syncookies = 1<br>net.ipv4.tcp_synack_retries = 1<br>net.ipv4.tcp_max_syn_backlog = 1024<br> sysctl -p<br><br><br><br><br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-02 09:21:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/8c/2b/3ab96998.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>青石</span>
  </div>
  <div class="_2_QraFYR_0">REJECT攻击IP所有报文，接口响应没什么变化，还是127秒，DROP报文后响应时间才恢复正常，查了REJECT和DROP的区别，REJECT会返回个ICMP包给源IP，有点不太理解为什么一个ICMP包导致这么大的差异。<br><br># hping3命令, u10效果不明显，所以改成了u1测试<br>$ hping3 -S -p 80  172.30.81.136 -I eno1 -i u1<br><br># 调整内核参数测试结果，接口响应时间为127秒<br>$ curl -s -w &#39;Http code: %{http_code}\nTotal time:%{time_total}s\n&#39; -o &#47;dev&#47;null http:&#47;&#47;172.30.81.136&#47; <br>Http code: 000<br>Total time:127.109s<br><br># 调整内核参数、iptables限制syn并发的测试结果，接口响应时间为127秒<br>$ curl -s -w &#39;Http code: %{http_code}\nTotal time:%{time_total}s\n&#39; -o &#47;dev&#47;null http:&#47;&#47;172.30.81.136&#47; <br>Http code: 000<br>Total time:127.106s<br><br># 调整内核参数、iptables限制syn并发、单IP连接数的测试结果，接口响应时间为127秒<br>$ curl -s -w &#39;Http code: %{http_code}\nTotal time:%{time_total}s\n&#39; -o &#47;dev&#47;null http:&#47;&#47;172.30.81.136&#47; <br>Http code: 000<br>Total time:127.097s<br><br># 调整内核参数、iptables限制syn并发、单IP连接数、DROP攻击IP所有报文的测试结果，接口响应时间为127秒<br>$ curl -s -w &#39;Http code: %{http_code}\nTotal time:%{time_total}s\n&#39; -o &#47;dev&#47;null http:&#47;&#47;172.30.81.136&#47; <br>Http code: 200<br>Total time:0.001s</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: REJECT还会给攻击源发送响应，这也是消耗网络资源的；而DROP没有任何回应，直接丢弃</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-17 22:53:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/60/05/3797d774.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>forever</span>
  </div>
  <div class="_2_QraFYR_0">我之前在生产环境中遇到过多次ddos攻击，最好的被打到55G.最初的时候遇到攻击束手无策，还被人勒索过，说给钱就不攻击！哈哈😄因为没经验，后来就打游击战，那时候攻击的是我们负载均衡的ip，当攻击发生时我就换ip这样能暂时缓解，当然提前要把ip给准备好，比如白名单添加等！后来业务壮大，换ip的时间成本比网站不能访问的成本要高，最后我们用了阿里云的高防，就这样攻击就告一段落了！虽然高防很贵，但是比起被攻击的损失，还是值得的！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍谢谢分享</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-25 17:14:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/de/ab/e87f0e5e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>spike</span>
  </div>
  <div class="_2_QraFYR_0">用了hping3 -S -p 80 -i u1 192.168.0.30把公司内网打炸了。。。感谢老师。。认识到了syn泛洪的威力。。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-01 23:13:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/3a/6e/e39e90ca.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>大坏狐狸</span>
  </div>
  <div class="_2_QraFYR_0">1 增大队列SYN最大半连接数sysctl -w et.ipv4.tcp_max_syn_backlog=3000<br>2 减少超时值：通过减少超时重传次数，sysctl -w net.ipv4.tcp_synack_retries=1<br>3 SYN_COOKIE : 开启cookie echo &quot;1&quot; &gt; &#47; proc&#47;sys&#47;net&#47;ipv4&#47;tcp_syncookies （原本对cookie有误解，看了本文才清楚，谢谢老师）<br>4 过滤可疑IP地址:  iptables -I INPUT -s 192.168.0.2 -p tcp -j REJECT  禁止 0.2访问<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-11 10:39:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/05/7f/d35ab9a1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>z.l</span>
  </div>
  <div class="_2_QraFYR_0">请问老师今天讲的三个tcp内核参数调优能否作为通用的提高服务器抗并发能力的优化手段？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-09 00:23:37</div>
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
  <div class="_2_QraFYR_0">Sysc backlog并不是在配置了sync cookie之后就失效了吧，netstat -s 看到的listen状态的recv 和send不是分别表示连接队列配置和当前活跃队列长度吗，比如监听某个端口的进程不响应了，那么这里的recv队列就会慢慢变长然后溢出。溢出之后，linux可以选择drop掉什么也不做，或者返回connection reset通知客户端</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 请参考 syn cookie 和 netstat 的文档，这些配置和字段的含义都有明确的说明</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-01 08:48:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/a0/2d/2fee2d83.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>且听风吟</span>
  </div>
  <div class="_2_QraFYR_0">net.ipv4.tcp_max_tw_buckets <br>net.ipv4.tcp_tw_reuse <br>net.ipv4.tcp_tw_recycle<br>net.ipv4.tcp_keepalive_time <br>net.ipv4.tcp_timestamps<br>期待结合生产环境对这几个内核参数的讲解。目前生产环境下php服务器time_wait特别多，网络包的流程：  NGINX代理&lt;——&gt;PHP服务器——&gt;redis&#47;mysql.. <br>高峰时期php服务器一共50k+的连接。49k+的time_wait.， 主要来源是php作为客户端的角色时连接redis、mysql、给代理NGINX回包、php服务器内部调用。希望老师能给解决问题的思路。<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，后面有的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-20 12:01:52</div>
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
  <div class="_2_QraFYR_0">打卡day42<br>真正的ddos要靠运营商的流量清洗之类的了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，流量型DDoS还是要靠硬件来抗</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-25 08:16:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/cf/5c/d4e19eb6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>安小依</span>
  </div>
  <div class="_2_QraFYR_0">随机化源 IP 和 多台机器分布式攻击 有什么区别，为什么限制 syn 包速率的方式不适用于多台机器分布式的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 总的流量大小不同，比如单机再怎么随机化总是受限于网络带宽，而多机组合产生的流量就要大得多了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-21 12:55:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/11/0e/23d6a88f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Shuai</span>
  </div>
  <div class="_2_QraFYR_0">到服务器的网络如果比喻成车道的话，感觉dos攻击像是恶意阻塞车道的车辆，耗尽带宽、耗尽操作系统的资源、消耗应用程序的运行资源分别相当于主车道被阻塞、服务器家附近的路被阻塞、服务器家门口的停车场被阻塞，导致真正属于服务器的车开不到家门口；<br>于是防治策略一个是 家门口的停车场加上管控，家门口附近的路加上管控，在主干道之前加上WAF、CDN等流量管控，让恶意的车辆进入主干道之前就被过滤掉。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-26 09:38:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/21/5f/f8/1d16434b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>陈麒文</span>
  </div>
  <div class="_2_QraFYR_0">坚持看到这，脑袋瓜有个映像，多个思考的方向</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-19 07:17:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/f6/24/547439f1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ghostwritten</span>
  </div>
  <div class="_2_QraFYR_0">1. DDoS 类型:耗尽带宽、耗尽操作系统的资源、消耗应用程序的运行资源<br>2. curl -s -w &#39;Http code: %{http_code}\nTotal time:%{time_total}s\n&#39; -o &#47;dev&#47;null http:&#47;&#47;192.168.0.30&#47;<br>curl -w &#39;Http code: %{http_code}\nTotal time:%{time_total}s\n&#39; -o &#47;dev&#47;null --connect-timeout 10 http:&#47;&#47;192.168.0.30<br>3. hping3 -S -p 80 -i u10 192.168.0.30<br>4. sar -n DEV 1<br>5. tcpdump -i eth0 -n tcp port 80<br>6. netstat -n -p | grep SYN_REC<br>7. iptables -I INPUT -s 192.168.0.2 -p tcp -j REJECT<br>8. iptables -A INPUT -p tcp --syn -m limit --limit 1&#47;s -j ACCEPT<br>9. iptables -I INPUT -p tcp --dport 80 --syn -m recent --name SYN_FLOOD --update --seconds 60 --hitcount 10 -j REJECT<br>10. sysctl net.ipv4.tcp_max_syn_backlog<br>sysctl -w net.ipv4.tcp_max_syn_backlog=1024<br>sysctl -w net.ipv4.tcp_synack_retries=1<br>sysctl -w net.ipv4.tcp_syncookies=1</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-03 23:06:50</div>
  </div>
</div>
</div>
</li>
</ul>