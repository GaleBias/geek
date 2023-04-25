<audio title="38 _ 案例篇：怎么使用 tcpdump 和 Wireshark 分析网络流量？" src="https://static001.geekbang.org/resource/audio/93/a6/935e147b172f260ea55bede88c7ae8a6.mp3" controls="controls"></audio> 
<p>你好，我是倪朋飞。</p><p>上一节，我们学习了 DNS 性能问题的分析和优化方法。简单回顾一下，DNS 可以提供域名和 IP 地址的映射关系，也是一种常用的全局负载均衡（GSLB）实现方法。</p><p>通常，需要暴露到公网的服务，都会绑定一个域名，既方便了人们记忆，也避免了后台服务 IP 地址的变更影响到用户。</p><p>不过要注意，DNS 解析受到各种网络状况的影响，性能可能不稳定。比如公网延迟增大，缓存过期导致要重新去上游服务器请求，或者流量高峰时 DNS 服务器性能不足等，都会导致 DNS 响应的延迟增大。</p><p>此时，可以借助 nslookup 或者 dig 的调试功能，分析 DNS 的解析过程，再配合 ping 等工具调试 DNS 服务器的延迟，从而定位出性能瓶颈。通常，你可以用缓存、预取、HTTPDNS 等方法，优化 DNS 的性能。</p><p>上一节我们用到的ping，是一个最常用的测试服务延迟的工具。很多情况下，ping 可以帮我们定位出延迟问题，不过有时候， ping 本身也会出现意想不到的问题。这时，就需要我们抓取ping 命令执行时收发的网络包，然后分析这些网络包，进而找出问题根源。</p><p>tcpdump 和 Wireshark 就是最常用的网络抓包和分析工具，更是分析网络性能必不可少的利器。</p><!-- [[[read_end]]] --><ul>
<li>
<p>tcpdump 仅支持命令行格式使用，常用在服务器中抓取和分析网络包。</p>
</li>
<li>
<p>Wireshark 除了可以抓包外，还提供了强大的图形界面和汇总分析工具，在分析复杂的网络情景时，尤为简单和实用。</p>
</li>
</ul><p>因而，在实际分析网络性能时，先用 tcpdump 抓包，后用 Wireshark 分析，也是一种常用的方法。</p><p>今天，我就带你一起看看，怎么使用 tcpdump 和 Wireshark ，来分析网络的性能问题。</p><h2>案例准备</h2><p>本次案例还是基于 Ubuntu 18.04，同样适用于其他的 Linux 系统。我使用的案例环境是这样的：</p><ul>
<li>
<p>机器配置：2 CPU，8GB 内存。</p>
</li>
<li>
<p>预先安装 tcpdump、Wireshark 等工具，如：</p>
</li>
</ul><pre><code># Ubuntu
apt-get install tcpdump wireshark

# CentOS
yum install -y tcpdump wireshark
</code></pre><p>由于 Wireshark 的图形界面，并不能通过 SSH 使用，所以我推荐你在本地机器（比如 Windows）中安装。你可以到 <a href="https://www.wireshark.org/">https://www.wireshark.org/</a> 下载并安装 Wireshark。</p><blockquote>
<p>跟以前一样，案例中所有命令，都默认以 root 用户（在Windows中，运行Wireshark时除外）运行。如果你是用普通用户身份登陆系统，请运行 sudo su root 命令切换到 root 用户。</p>
</blockquote><h2>再探 ping</h2><p>前面讲过，ping 是一种最常用的网络工具，常用来探测网络主机之间的连通性以及延迟。关于 ping 的原理和使用方法，我在前面的 <a href="https://time.geekbang.org/column/article/81057">Linux网络基础篇</a>  已经简单介绍过，而 DNS 缓慢的案例中，也多次用到了 ping 测试 DNS 服务器的延迟（RTT）。</p><p>不过，虽然 ping 比较简单，但有时候你会发现，ping 工具本身也可能出现异常，比如运行缓慢，但实际网络延迟却并不大的情况。</p><p>接下来，我们打开一个终端，SSH 登录到案例机器中，执行下面的命令，来测试案例机器与极客邦科技官网的连通性和延迟。如果一切正常，你会看到下面这个输出：</p><pre><code># ping 3 次（默认每次发送间隔1秒）
# 假设DNS服务器还是上一期配置的114.114.114.114
$ ping -c3 geektime.org
PING geektime.org (35.190.27.188) 56(84) bytes of data.
64 bytes from 35.190.27.188 (35.190.27.188): icmp_seq=1 ttl=43 time=36.8 ms
64 bytes from 35.190.27.188 (35.190.27.188): icmp_seq=2 ttl=43 time=31.1 ms
64 bytes from 35.190.27.188 (35.190.27.188): icmp_seq=3 ttl=43 time=31.2 ms

--- geektime.org ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 11049ms
rtt min/avg/max/mdev = 31.146/33.074/36.809/2.649 ms
</code></pre><p>ping 的输出界面，  <a href="https://time.geekbang.org/column/article/81057">Linux网络基础篇</a> 中我们已经学过，你可以先复习一下，自己解读并且分析这次的输出。</p><p>不过要注意，假如你运行时发现 ping 很快就结束了，那就执行下面的命令，再重试一下。至于这条命令的含义，稍后我们再做解释。</p><pre><code># 禁止接收从DNS服务器发送过来并包含googleusercontent的包
$ iptables -I INPUT -p udp --sport 53 -m string --string googleusercontent --algo bm -j DROP
</code></pre><p>根据 ping 的输出，你可以发现，geektime.org 解析后的 IP 地址是 35.190.27.188，而后三次 ping 请求都得到了响应，延迟（RTT）都是 30ms 多一点。</p><p>但汇总的地方，就有点儿意思了。3次发送，收到3次响应，没有丢包，但三次发送和接受的总时间居然超过了 11s（11049ms），这就有些不可思议了吧。</p><p>会想起上一节的 DNS 解析问题，你可能会怀疑，这可能是 DNS 解析缓慢的问题。但到底是不是呢？</p><p>再回去看 ping 的输出，三次 ping 请求中，用的都是 IP 地址，说明 ping 只需要在最开始运行时，解析一次得到 IP，后面就可以只用 IP了。</p><p>我们再用 nslookup 试试。在终端中执行下面的 nslookup 命令，注意，这次我们同样加了 time 命令，输出 nslookup 的执行时间：</p><pre><code>$ time nslookup geektime.org
Server:		114.114.114.114
Address:	114.114.114.114#53

Non-authoritative answer:
Name:	geektime.org
Address: 35.190.27.188


real	0m0.044s
user	0m0.006s
sys	0m0.003s
</code></pre><p>可以看到，域名解析还是很快的，只需要 44ms，显然比 11s 短了很多。</p><p>到这里，再往后该怎么分析呢？其实，这时候就可以用 tcpdump 抓包，查看 ping 在收发哪些网络包。</p><p>我们再打开另一个终端（终端二），SSH 登录案例机器后，执行下面的命令：</p><pre><code>$ tcpdump -nn udp port 53 or host 35.190.27.188
</code></pre><p>当然，你可以直接用 tcpdump 不加任何参数来抓包，但那样的话，就可能抓取到很多不相干的包。由于我们已经执行过 ping 命令，知道了 geekbang.org 的 IP 地址是35.190.27.188，也知道 ping 命令会执行 DNS 查询。所以，上面这条命令，就是基于这个规则进行过滤。</p><p>我来具体解释一下这条命令。</p><ul>
<li>
<p>-nn ，表示不解析抓包中的域名（即不反向解析）、协议以及端口号。</p>
</li>
<li>
<p>udp port 53 ，表示只显示 UDP协议的端口号（包括源端口和目的端口）为53的包。</p>
</li>
<li>
<p>host 35.190.27.188 ，表示只显示 IP 地址（包括源地址和目的地址）为35.190.27.188的包。</p>
</li>
<li>
<p>这两个过滤条件中间的“ or ”，表示或的关系，也就是说，只要满足上面两个条件中的任一个，就可以展示出来。</p>
</li>
</ul><p>接下来，回到终端一，执行相同的 ping 命令：</p><pre><code>$ ping -c3 geektime.org
...
--- geektime.org ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 11095ms
rtt min/avg/max/mdev = 81.473/81.572/81.757/0.130 ms
</code></pre><p>命令结束后，再回到终端二中，查看 tcpdump 的输出：</p><pre><code>tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
14:02:31.100564 IP 172.16.3.4.56669 &gt; 114.114.114.114.53: 36909+ A? geektime.org. (30)
14:02:31.507699 IP 114.114.114.114.53 &gt; 172.16.3.4.56669: 36909 1/0/0 A 35.190.27.188 (46)
14:02:31.508164 IP 172.16.3.4 &gt; 35.190.27.188: ICMP echo request, id 4356, seq 1, length 64
14:02:31.539667 IP 35.190.27.188 &gt; 172.16.3.4: ICMP echo reply, id 4356, seq 1, length 64
14:02:31.539995 IP 172.16.3.4.60254 &gt; 114.114.114.114.53: 49932+ PTR? 188.27.190.35.in-addr.arpa. (44)
14:02:36.545104 IP 172.16.3.4.60254 &gt; 114.114.114.114.53: 49932+ PTR? 188.27.190.35.in-addr.arpa. (44)
14:02:41.551284 IP 172.16.3.4 &gt; 35.190.27.188: ICMP echo request, id 4356, seq 2, length 64
14:02:41.582363 IP 35.190.27.188 &gt; 172.16.3.4: ICMP echo reply, id 4356, seq 2, length 64
14:02:42.552506 IP 172.16.3.4 &gt; 35.190.27.188: ICMP echo request, id 4356, seq 3, length 64
14:02:42.583646 IP 35.190.27.188 &gt; 172.16.3.4: ICMP echo reply, id 4356, seq 3, length 64
</code></pre><p>这次输出中，前两行，表示 tcpdump 的选项以及接口的基本信息；从第三行开始，就是抓取到的网络包的输出。这些输出的格式，都是 <code>时间戳 协议 源地址.源端口 &gt; 目的地址.目的端口 网络包详细信息</code>（这是最基本的格式，可以通过选项增加其他字段）。</p><p>前面的字段，都比较好理解。但网络包的详细信息，本身根据协议的不同而不同。所以，要理解这些网络包的详细含义，就要对常用网络协议的基本格式以及交互原理，有基本的了解。</p><p>当然，实际上，这些内容都会记录在 IETF（ 互联网工程任务组）发布的 <a href="https://tools.ietf.org/rfc/index">RFC</a>（请求意见稿）中。</p><p>比如，第一条就表示，从本地 IP 发送到 114.114.114.114 的 A 记录查询请求，它的报文格式记录在 RFC1035 中，你可以点击<a href="https://www.ietf.org/rfc/rfc1035.txt">这里</a>查看。在这个 tcpdump 的输出中，</p><ul>
<li>
<p>36909+ 表示查询标识值，它也会出现在响应中，加号表示启用递归查询。</p>
</li>
<li>
<p>A? 表示查询 A 记录。</p>
</li>
<li>
<p>geektime.org. 表示待查询的域名。</p>
</li>
<li>
<p>30 表示报文长度。</p>
</li>
</ul><p>接下来的一条，则是从 114.114.114.114 发送回来的 DNS 响应——域名 geektime.org. 的 A 记录值为 35.190.27.188。</p><p>第三条和第四条，是 ICMP echo request 和 ICMP echo reply，响应包的时间戳 14:02:31.539667，减去请求包的时间戳 14:02:31.508164 ，就可以得到，这次 ICMP 所用时间为 30ms。这看起来并没有问题。</p><p>但随后的两条反向地址解析 PTR 请求，就比较可疑了。因为我们只看到了请求包，却没有应答包。仔细观察它们的时间，你会发现，这两条记录都是发出后 5s 才出现下一个网络包，两条 PTR 记录就消耗了 10s。</p><p>再往下看，最后的四个包，则是两次正常的 ICMP 请求和响应，根据时间戳计算其延迟，也是 30ms。</p><p>到这里，其实我们也就找到了 ping 缓慢的根源，正是两次 PTR 请求没有得到响应而超时导致的。PTR 反向地址解析的目的，是从 IP 地址反查出域名，但事实上，并非所有IP 地址都会定义 PTR 记录，所以 PTR 查询很可能会失败。</p><p>所以，在你使用 ping 时，如果发现结果中的延迟并不大，而 ping 命令本身却很慢，不要慌，有可能是背后的 PTR 在搞鬼。</p><p>知道问题后，解决起来就比较简单了，只要禁止 PTR 就可以。还是老路子，执行 man ping 命令，查询使用手册，就可以找出相应的方法，即加上 -n 选项禁止名称解析。比如，我们可以在终端中执行如下命令：</p><pre><code>$ ping -n -c3 geektime.org
PING geektime.org (35.190.27.188) 56(84) bytes of data.
64 bytes from 35.190.27.188: icmp_seq=1 ttl=43 time=33.5 ms
64 bytes from 35.190.27.188: icmp_seq=2 ttl=43 time=39.0 ms
64 bytes from 35.190.27.188: icmp_seq=3 ttl=43 time=32.8 ms

--- geektime.org ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2002ms
rtt min/avg/max/mdev = 32.879/35.160/39.030/2.755 ms
</code></pre><p>你可以发现，现在只需要 2s 就可以结束，比刚才的 11s 可是快多了。</p><p>到这里， 我就带你一起使用 tcpdump ，解决了一个最常见的 ping 工作缓慢的问题。</p><p>案例最后，如果你在开始时，执行了 iptables 命令，那也不要忘了删掉它：</p><pre><code>$ iptables -D INPUT -p udp --sport 53 -m string --string googleusercontent --algo bm -j DROP
</code></pre><p>不过，删除后你肯定还有疑问，明明我们的案例跟 Google 没啥关系，为什么要根据 googleusercontent ，这个毫不相关的字符串来过滤包呢？</p><p>实际上，如果换一个 DNS 服务器，就可以用 PTR 反查到 35.190.27.188 所对应的域名：</p><pre><code> $ nslookup -type=PTR 35.190.27.188 8.8.8.8
Server:	8.8.8.8
Address:	8.8.8.8#53
Non-authoritative answer:
188.27.190.35.in-addr.arpa	name = 188.27.190.35.bc.googleusercontent.com.
Authoritative answers can be found from:
</code></pre><p>你看，虽然查到了 PTR 记录，但结果并非 geekbang.org，而是 188.27.190.35.bc.googleusercontent.com。其实，这也是为什么，案例开始时将包含 googleusercontent 的丢弃后，ping 就慢了。因为 iptables ，实际上是把 PTR 响应给丢了，所以会导致 PTR 请求超时。</p><p>tcpdump 可以说是网络性能分析最有效的利器。接下来，我再带你一起看看 tcpdump 的更多使用方法。</p><h2>tcpdump</h2><p>我们知道，tcpdump 也是最常用的一个网络分析工具。它基于 <a href="https://www.tcpdump.org/">libpcap</a>  ，利用内核中的 AF_PACKET 套接字，抓取网络接口中传输的网络包；并提供了强大的过滤规则，帮你从大量的网络包中，挑出最想关注的信息。</p><p>tcpdump 为你展示了每个网络包的详细细节，这就要求，在使用前，你必须要对网络协议有基本了解。而要了解网络协议的详细设计和实现细节， <a href="https://www.rfc-editor.org/rfc-index.html">RFC</a> 当然是最权威的资料。</p><p>不过，RFC 的内容，对初学者来说可能并不友好。如果你对网络协议还不太了解，推荐你去学《TCP/IP详解》，特别是第一卷的 TCP/IP 协议族。这是每个程序员都要掌握的核心基础知识。</p><p>再回到 tcpdump工具本身，它的基本使用方法，还是比较简单的，也就是 <strong>tcpdump [选项] [过滤表达式]</strong>。当然，选项和表达式的外面都加了中括号，表明它们都是可选的。</p><blockquote>
<p>提示：在 Linux 工具中，如果你在文档中看到，选项放在中括号里，就说明这是一个可选选项。这时候就要留意一下，这些选项是不是有默认值。</p>
</blockquote><p>查看 tcpdump 的 <a href="https://www.tcpdump.org/manpages/tcpdump.1.html">手册</a>  ，以及 pcap-filter 的<a href="https://www.tcpdump.org/manpages/pcap-filter.7.html">手册</a>，你会发现，tcpdump 提供了大量的选项以及各式各样的过滤表达式。不过不要担心，只需要掌握一些常用选项和过滤表达式，就可以满足大部分场景的需要了。</p><p>为了帮你更快上手 tcpdump 的使用，我在这里也帮你整理了一些最常见的用法，并且绘制成了表格，你可以参考使用。</p><p>首先，来看一下常用的几个选项。在上面的ping 案例中，我们用过  <strong>-nn</strong> 选项，表示不用对 IP 地址和端口号进行名称解析。其他常用选项，我用下面这张表格来解释。</p><p><img src="https://static001.geekbang.org/resource/image/85/ff/859d3b5c0071335429620a3fcdde4fff.png?wh=1655*994" alt=""></p><p>接下来，我们再来看常用的过滤表达式。刚刚用过的是  udp port 53 or host 35.190.27.188 ，表示抓取 DNS 协议的请求和响应包，以及源地址或目的地址为 35.190.27.188 的包。</p><p>其他常用的过滤选项，我也整理成了下面这个表格。</p><p><img src="https://static001.geekbang.org/resource/image/48/b3/4870a28c032bdd2a26561604ae2f7cb3.png?wh=1660*1129" alt=""></p><p>最后，再次强调 tcpdump 的输出格式，我在前面已经介绍了它的基本格式：</p><pre><code>时间戳 协议 源地址.源端口 &gt; 目的地址.目的端口 网络包详细信息
</code></pre><p>其中，网络包的详细信息取决于协议，不同协议展示的格式也不同。所以，更详细的使用方法，还是需要你去查询 tcpdump 的 <a href="https://www.tcpdump.org/manpages/tcpdump.1.html">man</a> 手册（执行 man tcpdump 也可以得到）。</p><p>不过，讲了这么多，你应该也发现了。tcpdump 虽然功能强大，可是输出格式却并不直观。特别是，当系统中网络包数比较多（比如PPS 超过几千）的时候，你想从 tcpdump 抓取的网络包中分析问题，实在不容易。</p><p>对比之下，Wireshark 则通过图形界面，以及一系列的汇总分析工具，提供了更友好的使用界面，让你可以用更快的速度，摆平网络性能问题。接下来，我们就详细来看看它。</p><h2>Wireshark</h2><p>Wireshark 也是最流行的一个网络分析工具，它最大的好处就是提供了跨平台的图形界面。跟 tcpdump 类似，Wireshark 也提供了强大的过滤规则表达式，同时，还内置了一系列的汇总分析工具。</p><p>比如，拿刚刚的 ping 案例来说，你可以执行下面的命令，把抓取的网络包保存到 ping.pcap 文件中：</p><pre><code>$ tcpdump -nn udp port 53 or host 35.190.27.188 -w ping.pcap
</code></pre><p>接着，把它拷贝到你安装有 Wireshark 的机器中，比如你可以用 scp 把它拷贝到本地来：</p><pre><code>$ scp host-ip/path/ping.pcap .
</code></pre><p>然后，再用 Wireshark 打开它。打开后，你就可以看到下面这个界面：</p><p><img src="https://static001.geekbang.org/resource/image/6b/2c/6b854703dcfcccf64c0a69adecf2f42c.png?wh=2316*400" alt=""></p><p>从 Wireshark 的界面里，你可以发现，它不仅以更规整的格式，展示了各个网络包的头部信息；还用了不同颜色，展示 DNS 和 ICMP 这两种不同的协议。你也可以一眼看出，中间的两条 PTR 查询并没有响应包。</p><p>接着，在网络包列表中选择某一个网络包后，在其下方的网络包详情中，你还可以看到，这个包在协议栈各层的详细信息。比如，以编号为 5 的 PTR 包为例：</p><p><img src="https://static001.geekbang.org/resource/image/59/25/59781a5dc7b1b9234643991365bfc925.png?wh=2230*864" alt=""></p><p>你可以看到，IP 层（Internet Protocol）的源地址和目的地址、传输层的 UDP 协议（User Datagram Protocol）、应用层的 DNS 协议（Domain Name System）的概要信息。</p><p>继续点击每层左边的箭头，就可以看到该层协议头的所有信息。比如点击 DNS 后，就可以看到 Transaction ID、Flags、Queries 等 DNS 协议各个字段的数值以及含义。</p><p>当然，Wireshark 的功能远不止如此。接下来我再带你一起，看一个 HTTP 的例子，并理解 TCP 三次握手和四次挥手的工作原理。</p><p>这个案例我们将要访问的是 <a href="http://example.com/">http://example.com/</a> 。进入终端一，执行下面的命令，首先查出 example.com 的 IP。然后，执行 tcpdump 命令，过滤得到的 IP 地址，并将结果保存到 web.pcap 中。</p><pre><code>$ dig +short example.com
93.184.216.34
$ tcpdump -nn host 93.184.216.34 -w web.pcap
</code></pre><blockquote>
<p>实际上，你可以在 host 表达式中，直接使用域名，即 <strong>tcpdump -nn host example.com -w web.pcap</strong>。</p>
</blockquote><p>接下来，切换到终端二，执行下面的 curl 命令，访问 <a href="http://example.com">http://example.com</a>：</p><pre><code>$ curl http://example.com
</code></pre><p>最后，再回到终端一，按下 Ctrl+C 停止 tcpdump，并把得到的 web.pcap 拷贝出来。</p><p>使用 Wireshark 打开 web.pcap 后，你就可以在 Wireshark 中，看到如下的界面：</p><p><img src="https://static001.geekbang.org/resource/image/07/9d/07bcdba5b563ebae36f5b5b453aacd9d.png?wh=2360*398" alt=""></p><p>由于 HTTP 基于 TCP ，所以你最先看到的三个包，分别是 TCP 三次握手的包。接下来，中间的才是 HTTP 请求和响应包，而最后的三个包，则是 TCP 连接断开时的三次挥手包。</p><p>从菜单栏中，点击 Statistics -&gt; Flow Graph，然后，在弹出的界面中的 Flow type 选择 TCP Flows，你可以更清晰的看到，整个过程中 TCP 流的执行过程：</p><p><img src="https://static001.geekbang.org/resource/image/4e/bb/4ec784752fdbc0cc5ead036a6419cbbb.png?wh=1526*596" alt=""></p><p>这其实跟各种教程上讲到的，TCP 三次握手和四次挥手很类似，作为对比， 你通常看到的 TCP 三次握手和四次挥手的流程，基本是这样的：</p><p><img src="https://static001.geekbang.org/resource/image/52/e8/5230fb678fcd3ca6b55d4644881811e8.png?wh=875*976" alt=""></p><p>(图片来自<a href="https://coolshell.cn/articles/11564.html">酷壳</a>)</p><p>不过，对比这两张图，你会发现，这里抓到的包跟上面的四次挥手，并不完全一样，实际挥手过程只有三个包，而不是四个。</p><p>其实，之所以有三个包，是因为服务器端收到客户端的 FIN 后，服务器端同时也要关闭连接，这样就可以把 ACK 和 FIN 合并到一起发送，节省了一个包，变成了“三次挥手”。</p><p>而通常情况下，服务器端收到客户端的 FIN 后，很可能还没发送完数据，所以就会先回复客户端一个 ACK 包。稍等一会儿，完成所有数据包的发送后，才会发送 FIN 包。这也就是四次挥手了。</p><p>抓包后， Wireshark 中就会显示下面这个界面（原始网络包来自 Wireshark TCP 4-times close 示例，你可以点击 <a href="https://wiki.wireshark.org/TCP%204-times%20close">这里</a> 下载）：</p><p><img src="https://static001.geekbang.org/resource/image/0e/99/0ecb6d11e5e7725107c0291c45aa7e99.png?wh=1898*186" alt=""></p><p>当然，Wireshark 的使用方法绝不只有这些，更多的使用方法，同样可以参考 <a href="https://www.wireshark.org/docs/">官方文档</a> 以及 <a href="https://wiki.wireshark.org/">WIKI</a>。</p><h2>小结</h2><p>今天，我们一起学了 tcpdump 和 Wireshark 的使用方法，并通过几个案例，学会了如何运用这两个工具来分析网络的收发过程，并找出潜在的性能问题。</p><p>当你发现针对相同的网络服务，使用 IP 地址快而换成域名却慢很多时，就要想到，有可能是 DNS 在捣鬼。DNS 的解析，不仅包括从域名解析出 IP 地址的 A 记录请求，还包括性能工具帮你，“聪明”地从 IP 地址反查域名的 PTR 请求。</p><p>实际上，<strong>根据 IP 地址反查域名、根据端口号反查协议名称，是很多网络工具默认的行为，而这往往会导致性能工具的工作缓慢</strong>。所以，通常，网络性能工具都会提供一个选项（比如 -n 或者 -nn），来禁止名称解析。</p><p>在工作中，当你碰到网络性能问题时，不要忘记tcpdump 和 Wireshark 这两个大杀器。你可以用它们抓取实际传输的网络包，再排查是否有潜在的性能问题。</p><h2>思考</h2><p>最后，我想请你来聊一聊，你是如何使用 tcpdump 和 Wireshark 的。你用 tcpdump 或者 Wireshark 解决过哪些网络问题呢？你又是如何排查、分析并解决的呢？你可以结合今天学到的网络知识，总结自己的思路。</p><p>欢迎在留言区和我讨论，也欢迎你把这篇文章分享给你的同事、朋友。我们一起在实战中演练，在交流中进步。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/f4/5c/e0a56bbf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>1+1</span>
  </div>
  <div class="_2_QraFYR_0">wireshark的使用推荐阅读林沛满的《Wireshark网络分析就这么简单》和《Wireshark网络分析的艺术》</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍 这两本书都不错</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-18 08:48:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/mnBC29lF1RibHdwkZdPdGM9QRAl7Y7Aicad8vpmIEialjia93IEVSAHibkKdwHwfZr6qedVHiafKUD8Yk1v2eiaibj8l0w/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>xierongfei</span>
  </div>
  <div class="_2_QraFYR_0">之前公司一个内部应用出现页面卡顿，而且每次都是1-2用户反馈（随机），排出了应用本身，服务器，客户端网络问题后，然后让it在用户端抓包传给我，然后用Wireshark分析后，发现有大量虚假重传，后面分析后发现，是用户都在一个Nat网络后面，部分用户时间不一致，同时我们服务器开启了tcp快速回收，导致连接被回收了。后面关闭tcp快速回收后解决。也是第一次用工具分析这种比较复杂的问题。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-13 15:37:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/20/b0/0a1551c4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>日拱一卒</span>
  </div>
  <div class="_2_QraFYR_0">林沛满的书都看过，确实写的相当好，都是案例驱动。<br>把协议讲的生动有趣就数他。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-18 09:10:02</div>
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
  <div class="_2_QraFYR_0">&quot;如果看了这个你还是不会用Wireshark，那就来找我吧&quot; ----这是在网上可以找到最牛逼的资料<br><br>https:&#47;&#47;www.dell.com&#47;community&#47;%E6%95%B0%E6%8D%AE%E5%AD%98%E5%82%A8%E8%AE%A8%E8%AE%BA%E5%8C%BA&#47;%E5%A6%82%E6%9E%9C%E7%9C%8B%E4%BA%86%E8%BF%99%E4%B8%AA%E4%BD%A0%E8%BF%98%E6%98%AF%E4%B8%8D%E4%BC%9A%E7%94%A8Wireshark-%E9%82%A3%E5%B0%B1%E6%9D%A5%E6%89%BE%E6%88%91%E5%90%A7-8%E6%9C%886%E6%97%A5%E5%AE%8C%E7%BB%93&#47;td-p&#47;7007033</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢分享</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-06 22:37:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/44/f8/6fdb50ab.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>肖飞码字</span>
  </div>
  <div class="_2_QraFYR_0">tcpdump抓包可以用来处理一些疑难问题的。 如受到了什么类型的攻击，执行了mysql的什么命令，接收以及发送出去了什么数据包通通都可以。像入侵检测如snort工具之类的应该也是对数据包进行抓包分析的，很实用，很强大。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍 谢谢分享</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-19 17:07:54</div>
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
  <div class="_2_QraFYR_0">老师，这个案例写的极其生动:D, 就是问一句，现在我们的项目都是https，那么如果抓包https，tcpdump或者wireshark是否可以解密？因为我看到wireshark解密需要private key，但是private key涉及安全问题，肯定都拿不到，那么你们遇到抓包https后解析是怎么做的呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，证书解密是最简单的方法，也可以使用 MitM（Man-in-the-middle）方法</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-05 10:50:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/25/66/4835d92e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>潘政宇</span>
  </div>
  <div class="_2_QraFYR_0">老师好，为什么ping命令使用PTR信息，ping一个域名的时候，直接dns查询得到A记录ip地址，然后ping这个ip就行啊，为什么使用反向解析？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不只是ping，大部分输出中包含名字的工具都支持反向解析，这是为了更直观展示结果（毕竟我们熟悉的都是域名而不是IP）。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-21 10:07:08</div>
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
  <div class="_2_QraFYR_0">老师wireshark有个命令行版本tshark，wireshark是通过一个叫dump的进程抓包管道方式发送给tshark解析的，不过tshark的命令项有点多，不是太好用</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍 没有图形界面的时候，tshark也是个选择</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-18 08:19:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKzqiaZnBw2myRWY802u48Rw3W2zDtKoFQ6vN63m4FdyjibM21FfaOYe8MbMpemUdxXJeQH6fRdVbZA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>kissingers</span>
  </div>
  <div class="_2_QraFYR_0">林沛满的书不错，EMC 大牛值得推荐。<br>Fiddler，工具也了解过，微软的人写的，支持https （man in middle），能修改请求和响应数据包。<br>另外老师能讲讲：linux 主机上怎么提高数据转发性能吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 优化方法里面有提到</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-07 10:47:55</div>
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
  <div class="_2_QraFYR_0">如果换一个 DNS 服务器，就可以用 PTR 反查到 35.190.27.188 所对应的域名，到最后也没讲到为什么是googleaccount域名。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-30 06:54:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/97/61/34a0da09.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Griffin</span>
  </div>
  <div class="_2_QraFYR_0">Web的问题推荐 MITM啊</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯嗯，谢谢补充</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-24 12:47:29</div>
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
  <div class="_2_QraFYR_0">打卡day40<br>使用姿势跟老师的一样，都是tcpdump抓包后，拿到图形界面分析，如果是web问题，还会用下fiddler来分析</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯嗯，fiddler主要用于HTTP</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-22 08:20:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/83/4b/0e96fcae.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>sky</span>
  </div>
  <div class="_2_QraFYR_0">实践的时候出现了以下错误，也没找到解决方案，有朋友遇到过吗<br><br>配置：<br>腾讯云 OS：ubuntu18.04 <br>CPU：2 <br>Mem：4G<br>Bandwidth：8Mbps<br><br>resolv.conf<br>nameserver 114.114.114.114<br>options edns0<br><br>root@VM-4-13-ubuntu:&#47;home&#47;ubuntu# nslookup -type=PTR 39.106.233.176 8.8.8.8<br>Server:         8.8.8.8<br>Address:        8.8.8.8#53<br><br>** server can&#39;t find 176.233.106.39.in-addr.arpa: NXDOMAIN</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-16 09:24:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/18/5e/abb7bfe3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>哈哈哈哈哈哈哈哈</span>
  </div>
  <div class="_2_QraFYR_0">老师 对线上服务进行tcpdump会影响性能吗</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-15 08:34:15</div>
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
  <div class="_2_QraFYR_0">[D38打卡]<br>以前都是在windows上用Wireshark,想不到linux下也有Wireshark.<br>不过平常维护的linux只能用终端登录,在终端中还是用tcpdump.<br><br>今天看了专栏,才发现,可以用tcpdump抓包,然后用Wireshark来展示.这个厉害了.<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯 复杂的场景还是用Wireshark更方便些</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-18 10:02:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/de/00/c646dc88.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>丫头</span>
  </div>
  <div class="_2_QraFYR_0">怎么安装tcpdump</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-20 21:07:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/c4/03/f753fda7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>JianXu</span>
  </div>
  <div class="_2_QraFYR_0">我们部门现在使用iPvS 和 envoy 作分布式负载均衡，有人说这种情况用log 分析就可以了，没有必要使用tcpdump, 也有人说需要使用tcpdump,  为什么支持类似Netscalar nstrace 的能力，需要改动envoy 源代码，加载自定义的kernel module , 还要暂存key … 不知道兄弟公司有这样的需求吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-30 09:10:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/82/3d/356fc3d6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>忆水寒</span>
  </div>
  <div class="_2_QraFYR_0">分析过很多问题，比如高峰期的时候，网络断连。<br><br>https:&#47;&#47;mp.weixin.qq.com&#47;s&#47;XS3Jnn3Xl4_12gzg0zrSTA</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-01 23:00:50</div>
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
  <div class="_2_QraFYR_0">ping -c3 geektime.org<br>ping -n -c3 geektime.org<br>iptables -I INPUT -p udp --sport 53 -m string --string googleusercontent --algo bm -j DROP<br>iptables -D INPUT -p udp --sport 53 -m string --string googleusercontent --algo bm -j DROP<br>time nslookup geektime.org<br> nslookup -type=PTR 35.190.27.188 8.8.8.8<br>tcpdump -nn udp port 53 or host 35.190.27.188<br>tcpdump -nn udp port 53 or host 35.190.27.188 -w ping.pcap<br>dig +short example.com<br>tcpdump -nn host 93.184.216.34 -w web.pcap</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-30 16:25:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/02/75/2839c0bb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>谨兮谨兮</span>
  </div>
  <div class="_2_QraFYR_0">为什么ping域名出现ptr呢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-15 14:26:44</div>
  </div>
</div>
</div>
</li>
</ul>