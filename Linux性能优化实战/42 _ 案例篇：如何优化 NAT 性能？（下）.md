<audio title="42 _ 案例篇：如何优化 NAT 性能？（下）" src="https://static001.geekbang.org/resource/audio/30/bd/30b5d89dee30ee92a0400cba23d78bbd.mp3" controls="controls"></audio> 
<p>你好，我是倪朋飞。</p><p>上一节，我们学习了 NAT 的原理，明白了如何在 Linux 中管理 NAT 规则。先来简单复习一下。</p><p>NAT 技术能够重写 IP 数据包的源 IP 或目的 IP，所以普遍用来解决公网 IP 地址短缺的问题。它可以让网络中的多台主机，通过共享同一个公网 IP 地址，来访问外网资源。同时，由于 NAT 屏蔽了内网网络，也为局域网中机器起到安全隔离的作用。</p><p>Linux 中的NAT ，基于内核的连接跟踪模块实现。所以，它维护每个连接状态的同时，也对网络性能有一定影响。那么，碰到 NAT 性能问题时，我们又该怎么办呢？</p><p>接下来，我就通过一个案例，带你学习 NAT 性能问题的分析思路。</p><h2>案例准备</h2><p>下面的案例仍然基于 Ubuntu 18.04，同样适用于其他的 Linux 系统。我使用的案例环境是这样的：</p><ul>
<li>
<p>机器配置：2 CPU，8GB 内存。</p>
</li>
<li>
<p>预先安装 docker、tcpdump、curl、ab、SystemTap 等工具，比如</p>
</li>
</ul><pre><code>  # Ubuntu
  $ apt-get install -y docker.io tcpdump curl apache2-utils
  
  # CentOS
  $ curl -fsSL https://get.docker.com | sh
  $ yum install -y tcpdump curl httpd-tools
</code></pre><p>大部分工具，你应该都比较熟悉，这里我简单介绍一下 SystemTap 。</p><p><a href="https://sourceware.org/systemtap/">SystemTap</a> 是 Linux 的一种动态追踪框架，它把用户提供的脚本，转换为内核模块来执行，用来监测和跟踪内核的行为。关于它的原理，你暂时不用深究，后面的内容还会介绍到。这里你只要知道怎么安装就可以了：</p><!-- [[[read_end]]] --><pre><code># Ubuntu
apt-get install -y systemtap-runtime systemtap
# Configure ddebs source
echo &quot;deb http://ddebs.ubuntu.com $(lsb_release -cs) main restricted universe multiverse
deb http://ddebs.ubuntu.com $(lsb_release -cs)-updates main restricted universe multiverse
deb http://ddebs.ubuntu.com $(lsb_release -cs)-proposed main restricted universe multiverse&quot; | \
sudo tee -a /etc/apt/sources.list.d/ddebs.list
# Install dbgsym
apt-key adv --keyserver keyserver.ubuntu.com --recv-keys F2EDC64DC5AEE1F6B9C621F0C8CAB6595FDFF622
apt-get update
apt install ubuntu-dbgsym-keyring
stap-prep
apt-get install linux-image-`uname -r`-dbgsym

# CentOS
yum install systemtap kernel-devel yum-utils kernel
stab-prep
</code></pre><p>本次案例还是我们最常见的 Nginx，并且会用 ab 作为它的客户端，进行压力测试。案例中总共用到两台虚拟机，我画了一张图来表示它们的关系。</p><p><img src="https://static001.geekbang.org/resource/image/70/c6/7081ad1b72535107e94f852ac41e0dc6.png?wh=1632*1032" alt=""></p><p>接下来，我们打开两个终端，分别 SSH 登录到两台机器上（以下步骤，假设终端编号与图示VM 编号一致），并安装上面提到的这些工具。注意，curl 和 ab 只需要在客户端 VM（即 VM2）中安装。</p><p>同以前的案例一样，下面的所有命令都默认以 root 用户运行。如果你是用普通用户身份登陆系统，请运行 sudo su root 命令，切换到 root 用户。</p><blockquote>
<p>如果安装过程中有什么问题，同样鼓励你先自己搜索解决，解决不了的，可以在留言区向我提问。如果你以前已经安装过了，就可以忽略这一点了。</p>
</blockquote><p>接下来，我们就进入到案例环节。</p><h2>案例分析</h2><p>为了对比 NAT 带来的性能问题，我们首先运行一个不用 NAT 的 Nginx 服务，并用 ab 测试它的性能。</p><p>在终端一中，执行下面的命令，启动 Nginx，注意选项 --network=host ，表示容器使用 Host 网络模式，即不使用 NAT：</p><pre><code>$ docker run --name nginx-hostnet --privileged --network=host -itd feisky/nginx:80
</code></pre><p>然后到终端二中，执行 curl 命令，确认 Nginx 正常启动：</p><pre><code>$ curl http://192.168.0.30/
...
&lt;p&gt;&lt;em&gt;Thank you for using nginx.&lt;/em&gt;&lt;/p&gt;
&lt;/body&gt;
&lt;/html&gt;
</code></pre><p>继续在终端二中，执行 ab 命令，对 Nginx 进行压力测试。不过在测试前要注意，Linux 默认允许打开的文件描述数比较小，比如在我的机器中，这个值只有 1024：</p><pre><code># open files
$ ulimit -n
1024
</code></pre><p>所以，执行 ab 前，先要把这个选项调大，比如调成 65536:</p><pre><code># 临时增大当前会话的最大文件描述符数
$ ulimit -n 65536
</code></pre><p>接下来，再去执行 ab 命令，进行压力测试：</p><pre><code># -c表示并发请求数为5000，-n表示总的请求数为10万
# -r表示套接字接收错误时仍然继续执行，-s表示设置每个请求的超时时间为2s
$ ab -c 5000 -n 100000 -r -s 2 http://192.168.0.30/
...
Requests per second:    6576.21 [#/sec] (mean)
Time per request:       760.317 [ms] (mean)
Time per request:       0.152 [ms] (mean, across all concurrent requests)
Transfer rate:          5390.19 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0  177 714.3      9    7338
Processing:     0   27  39.8     19     961
Waiting:        0   23  39.5     16     951
Total:          1  204 716.3     28    7349
...
</code></pre><p>关于 ab 输出界面的含义，我已经在 <a href="https://time.geekbang.org/column/article/81497">怎么评估系统的网络性能</a> 文章中介绍过，忘了的话自己先去复习。从这次的界面，你可以看出：</p><ul>
<li>
<p>每秒请求数（Requests  per second）为 6576；</p>
</li>
<li>
<p>每个请求的平均延迟（Time per request）为 760ms；</p>
</li>
<li>
<p>建立连接的平均延迟（Connect）为 177ms。</p>
</li>
</ul><p>记住这几个数值，这将是接下来案例的基准指标。</p><blockquote>
<p>注意，你的机器中，运行结果跟我的可能不一样，不过没关系，并不影响接下来的案例分析思路。</p>
</blockquote><p>接着，回到终端一，停止这个未使用NAT的Nginx应用：</p><pre><code>$ docker rm -f nginx-hostnet
</code></pre><p>再执行下面的命令，启动今天的案例应用。案例应用监听在 8080 端口，并且使用了 DNAT ，来实现 Host 的 8080 端口，到容器的 8080 端口的映射关系：</p><pre><code>$ docker run --name nginx --privileged -p 8080:8080 -itd feisky/nginx:nat
</code></pre><p>Nginx 启动后，你可以执行 iptables 命令，确认 DNAT 规则已经创建：</p><pre><code>$ iptables -nL -t nat
Chain PREROUTING (policy ACCEPT)
target     prot opt source               destination
DOCKER     all  --  0.0.0.0/0            0.0.0.0/0            ADDRTYPE match dst-type LOCAL

...

Chain DOCKER (2 references)
target     prot opt source               destination
RETURN     all  --  0.0.0.0/0            0.0.0.0/0
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:8080 to:172.17.0.2:8080
</code></pre><p>你可以看到，在 PREROUTING 链中，目的为本地的请求，会转到 DOCKER 链；而在 DOCKER 链中，目的端口为 8080 的 tcp 请求，会被 DNAT 到 172.17.0.2 的 8080 端口。其中，172.17.0.2 就是 Nginx 容器的 IP 地址。</p><p>接下来，我们切换到终端二中，执行 curl 命令，确认 Nginx 已经正常启动：</p><pre><code>$ curl http://192.168.0.30:8080/
...
&lt;p&gt;&lt;em&gt;Thank you for using nginx.&lt;/em&gt;&lt;/p&gt;
&lt;/body&gt;
&lt;/html&gt;
</code></pre><p>然后，再次执行上述的 ab 命令，不过这次注意，要把请求的端口号换成 8080：</p><pre><code># -c表示并发请求数为5000，-n表示总的请求数为10万
# -r表示套接字接收错误时仍然继续执行，-s表示设置每个请求的超时时间为2s
$ ab -c 5000 -n 100000 -r -s 2 http://192.168.0.30:8080/
...
apr_pollset_poll: The timeout specified has expired (70007)
Total of 5602 requests completed
</code></pre><p>果然，刚才正常运行的 ab ，现在失败了，还报了连接超时的错误。运行 ab 时的-s 参数，设置了每个请求的超时时间为 2s，而从输出可以看到，这次只完成了 5602 个请求。</p><p>既然是为了得到 ab 的测试结果，我们不妨把超时时间延长一下试试，比如延长到 30s。延迟增大意味着要等更长时间，为了快点得到结果，我们可以同时把总测试次数，也减少到 10000:</p><pre><code>$ ab -c 5000 -n 10000 -r -s 30 http://192.168.0.30:8080/
...
Requests per second:    76.47 [#/sec] (mean)
Time per request:       65380.868 [ms] (mean)
Time per request:       13.076 [ms] (mean, across all concurrent requests)
Transfer rate:          44.79 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0 1300 5578.0      1   65184
Processing:     0 37916 59283.2      1  130682
Waiting:        0    2   8.7      1     414
Total:          1 39216 58711.6   1021  130682
...
</code></pre><p>再重新看看 ab 的输出，这次的结果显示：</p><ul>
<li>
<p>每秒请求数（Requests per second）为 76；</p>
</li>
<li>
<p>每个请求的延迟（Time per request）为 65s；</p>
</li>
<li>
<p>建立连接的延迟（Connect）为 1300ms。</p>
</li>
</ul><p>显然，每个指标都比前面差了很多。</p><p>那么，碰到这种问题时，你会怎么办呢？你可以根据前面的讲解，先自己分析一下，再继续学习下面的内容。</p><p>在上一节，我们使用 tcpdump 抓包的方法，找出了延迟增大的根源。那么今天的案例，我们仍然可以用类似的方法寻找线索。不过，现在换个思路，因为今天我们已经事先知道了问题的根源——那就是 NAT。</p><p>回忆一下Netfilter 中，网络包的流向以及 NAT 的原理，你会发现，要保证 NAT 正常工作，就至少需要两个步骤：</p><ul>
<li>
<p>第一，利用 Netfilter 中的钩子函数（Hook），修改源地址或者目的地址。</p>
</li>
<li>
<p>第二，利用连接跟踪模块 conntrack ，关联同一个连接的请求和响应。</p>
</li>
</ul><p>是不是这两个地方出现了问题呢？我们用前面提到的动态追踪工具 SystemTap 来试试。</p><p>由于今天案例是在压测场景下，并发请求数大大降低，并且我们清楚知道 NAT 是罪魁祸首。所以，我们有理由怀疑，内核中发生了丢包现象。</p><p>我们可以回到终端一中，创建一个 dropwatch.stp 的脚本文件，并写入下面的内容：</p><pre><code>#! /usr/bin/env stap

############################################################
# Dropwatch.stp
# Author: Neil Horman &lt;nhorman@redhat.com&gt;
# An example script to mimic the behavior of the dropwatch utility
# http://fedorahosted.org/dropwatch
############################################################

# Array to hold the list of drop points we find
global locations

# Note when we turn the monitor on and off
probe begin { printf(&quot;Monitoring for dropped packets\n&quot;) }
probe end { printf(&quot;Stopping dropped packet monitor\n&quot;) }

# increment a drop counter for every location we drop at
probe kernel.trace(&quot;kfree_skb&quot;) { locations[$location] &lt;&lt;&lt; 1 }

# Every 5 seconds report our drop locations
probe timer.sec(5)
{
  printf(&quot;\n&quot;)
  foreach (l in locations-) {
    printf(&quot;%d packets dropped at %s\n&quot;,
           @count(locations[l]), symname(l))
  }
  delete locations
}
</code></pre><p>这个脚本，跟踪内核函数 kfree_skb() 的调用，并统计丢包的位置。文件保存好后，执行下面的 stap 命令，就可以运行丢包跟踪脚本。这里的stap，是 SystemTap 的命令行工具：</p><pre><code>$ stap --all-modules dropwatch.stp
Monitoring for dropped packets
</code></pre><p>当你看到 probe begin 输出的 “Monitoring for dropped packets” 时，表明 SystemTap 已经将脚本编译为内核模块，并启动运行了。</p><p>接着，我们切换到终端二中，再次执行 ab 命令：</p><pre><code>$ ab -c 5000 -n 10000 -r -s 30 http://192.168.0.30:8080/
</code></pre><p>然后，再次回到终端一中，观察 stap 命令的输出：</p><pre><code>10031 packets dropped at nf_hook_slow
676 packets dropped at tcp_v4_rcv

7284 packets dropped at nf_hook_slow
268 packets dropped at tcp_v4_rcv
</code></pre><p>你会发现，大量丢包都发生在 nf_hook_slow 位置。看到这个名字，你应该能想到，这是在 Netfilter Hook 的钩子函数中，出现丢包问题了。但是不是 NAT，还不能确定。接下来，我们还得再跟踪  nf_hook_slow 的执行过程，这一步可以通过 perf 来完成。</p><p>我们切换到终端二中，再次执行 ab 命令：</p><pre><code>$ ab -c 5000 -n 10000 -r -s 30 http://192.168.0.30:8080/
</code></pre><p>然后，再次切换回终端一，执行 perf record 和 perf report 命令</p><pre><code># 记录一会（比如30s）后按Ctrl+C结束
$ perf record -a -g -- sleep 30

# 输出报告
$ perf report -g graph,0
</code></pre><p>在 perf report 界面中，输入查找命令 / 然后，在弹出的对话框中，输入 nf_hook_slow；最后再展开调用栈，就可以得到下面这个调用图：</p><p><img src="https://static001.geekbang.org/resource/image/0e/3c/0e844a471ff1062a1db70a303add943c.png?wh=587*851" alt=""></p><p>从这个图我们可以看到，nf_hook_slow 调用最多的有三个地方，分别是 ipv4_conntrack_in、br_nf_pre_routing 以及 iptable_nat_ipv4_in。换言之，nf_hook_slow 主要在执行三个动作。</p><ul>
<li>
<p>第一，接收网络包时，在连接跟踪表中查找连接，并为新的连接分配跟踪对象（Bucket）。</p>
</li>
<li>
<p>第二，在 Linux 网桥中转发包。这是因为案例 Nginx 是一个 Docker 容器，而容器的网络通过网桥来实现；</p>
</li>
<li>
<p>第三，接收网络包时，执行 DNAT，即把 8080 端口收到的包转发给容器。</p>
</li>
</ul><p>到这里，我们其实就找到了性能下降的三个来源。这三个来源，都是 Linux 的内核机制，所以接下来的优化，自然也是要从内核入手。</p><p>根据以前各个资源模块的内容，我们知道，Linux 内核为用户提供了大量的可配置选项，这些选项可以通过 proc 文件系统，或者 sys 文件系统，来查看和修改。除此之外，你还可以用 sysctl 这个命令行工具，来查看和修改内核配置。</p><p>比如，我们今天的主题是 DNAT，而 DNAT 的基础是 conntrack，所以我们可以先看看，内核提供了哪些 conntrack 的配置选项。</p><p>我们在终端一中，继续执行下面的命令：</p><pre><code>$ sysctl -a | grep conntrack
net.netfilter.nf_conntrack_count = 180
net.netfilter.nf_conntrack_max = 1000
net.netfilter.nf_conntrack_buckets = 65536
net.netfilter.nf_conntrack_tcp_timeout_syn_recv = 60
net.netfilter.nf_conntrack_tcp_timeout_syn_sent = 120
net.netfilter.nf_conntrack_tcp_timeout_time_wait = 120
...
</code></pre><p>你可以看到，这里最重要的三个指标：</p><ul>
<li>
<p>net.netfilter.nf_conntrack_count，表示当前连接跟踪数；</p>
</li>
<li>
<p>net.netfilter.nf_conntrack_max，表示最大连接跟踪数；</p>
</li>
<li>
<p>net.netfilter.nf_conntrack_buckets，表示连接跟踪表的大小。</p>
</li>
</ul><p>所以，这个输出告诉我们，当前连接跟踪数是 180，最大连接跟踪数是 1000，连接跟踪表的大小，则是 65536。</p><p>回想一下前面的 ab 命令，并发请求数是 5000，而请求数是 100000。显然，跟踪表设置成，只记录 1000 个连接，是远远不够的。</p><p>实际上，内核在工作异常时，会把异常信息记录到日志中。比如前面的 ab 测试，内核已经在日志中报出了 “nf_conntrack: table full” 的错误。执行 dmesg 命令，你就可以看到：</p><pre><code>$ dmesg | tail
[104235.156774] nf_conntrack: nf_conntrack: table full, dropping packet
[104243.800401] net_ratelimit: 3939 callbacks suppressed
[104243.800401] nf_conntrack: nf_conntrack: table full, dropping packet
[104262.962157] nf_conntrack: nf_conntrack: table full, dropping packet
</code></pre><p>其中，net_ratelimit 表示有大量的日志被压缩掉了，这是内核预防日志攻击的一种措施。而当你看到 “nf_conntrack: table full” 的错误时，就表明 nf_conntrack_max 太小了。</p><p>那是不是，直接把连接跟踪表调大就可以了呢？调节前，你先得明白，连接跟踪表，实际上是内存中的一个哈希表。如果连接跟踪数过大，也会耗费大量内存。</p><p>其实，我们上面看到的 nf_conntrack_buckets，就是哈希表的大小。哈希表中的每一项，都是一个链表（称为 Bucket），而链表长度，就等于 nf_conntrack_max 除以 nf_conntrack_buckets。</p><p>比如，我们可以估算一下，上述配置的连接跟踪表占用的内存大小：</p><pre><code># 连接跟踪对象大小为376，链表项大小为16
nf_conntrack_max*连接跟踪对象大小+nf_conntrack_buckets*链表项大小 
= 1000*376+65536*16 B
= 1.4 MB
</code></pre><p>接下来，我们将 nf_conntrack_max 改大一些，比如改成 131072（即nf_conntrack_buckets的2倍）：</p><pre><code>$ sysctl -w net.netfilter.nf_conntrack_max=131072
$ sysctl -w net.netfilter.nf_conntrack_buckets=65536
</code></pre><p>然后再切换到终端二中，重新执行 ab 命令。注意，这次我们把超时时间也改回原来的 2s：</p><pre><code>$ ab -c 5000 -n 100000 -r -s 2 http://192.168.0.30:8080/
...
Requests per second:    6315.99 [#/sec] (mean)
Time per request:       791.641 [ms] (mean)
Time per request:       0.158 [ms] (mean, across all concurrent requests)
Transfer rate:          4985.15 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0  355 793.7     29    7352
Processing:     8  311 855.9     51   14481
Waiting:        0  292 851.5     36   14481
Total:         15  666 1216.3    148   14645
</code></pre><p>果然，现在你可以看到：</p><ul>
<li>
<p>每秒请求数（Requests per second）为 6315（不用NAT时为6576）；</p>
</li>
<li>
<p>每个请求的延迟（Time per request）为 791ms（不用NAT时为760ms）；</p>
</li>
<li>
<p>建立连接的延迟（Connect）为 355ms（不用NAT时为177ms）。</p>
</li>
</ul><p>这个结果，已经比刚才的测试好了很多，也很接近最初不用 NAT 时的基准结果了。</p><p>不过，你可能还是很好奇，连接跟踪表里，到底都包含了哪些东西？这里的东西，又是怎么刷新的呢？</p><p>实际上，你可以用 conntrack 命令行工具，来查看连接跟踪表的内容。比如：</p><pre><code># -L表示列表，-o表示以扩展格式显示
$ conntrack -L -o extended | head
ipv4     2 tcp      6 7 TIME_WAIT src=192.168.0.2 dst=192.168.0.96 sport=51744 dport=8080 src=172.17.0.2 dst=192.168.0.2 sport=8080 dport=51744 [ASSURED] mark=0 use=1
ipv4     2 tcp      6 6 TIME_WAIT src=192.168.0.2 dst=192.168.0.96 sport=51524 dport=8080 src=172.17.0.2 dst=192.168.0.2 sport=8080 dport=51524 [ASSURED] mark=0 use=1
</code></pre><p>从这里你可以发现，连接跟踪表里的对象，包括了协议、连接状态、源IP、源端口、目的IP、目的端口、跟踪状态等。由于这个格式是固定的，所以我们可以用 awk、sort 等工具，对其进行统计分析。</p><p>比如，我们还是以 ab 为例。在终端二启动 ab 命令后，再回到终端一中，执行下面的命令：</p><pre><code># 统计总的连接跟踪数
$ conntrack -L -o extended | wc -l
14289

# 统计TCP协议各个状态的连接跟踪数
$ conntrack -L -o extended | awk '/^.*tcp.*$/ {sum[$6]++} END {for(i in sum) print i, sum[i]}'
SYN_RECV 4
CLOSE_WAIT 9
ESTABLISHED 2877
FIN_WAIT 3
SYN_SENT 2113
TIME_WAIT 9283

# 统计各个源IP的连接跟踪数
$ conntrack -L -o extended | awk '{print $7}' | cut -d &quot;=&quot; -f 2 | sort | uniq -c | sort -nr | head -n 10
  14116 192.168.0.2
    172 192.168.0.96
</code></pre><p>这里统计了总连接跟踪数，TCP协议各个状态的连接跟踪数，以及各个源IP的连接跟踪数。你可以看到，大部分 TCP 的连接跟踪，都处于 TIME_WAIT 状态，并且它们大都来自于 192.168.0.2 这个 IP 地址（也就是运行 ab 命令的 VM2）。</p><p>这些处于 TIME_WAIT 的连接跟踪记录，会在超时后清理，而默认的超时时间是 120s，你可以执行下面的命令来查看：</p><pre><code>$ sysctl net.netfilter.nf_conntrack_tcp_timeout_time_wait
net.netfilter.nf_conntrack_tcp_timeout_time_wait = 120
</code></pre><p>所以，如果你的连接数非常大，确实也应该考虑，适当减小超时时间。</p><p>除了上面这些常见配置，conntrack 还包含了其他很多配置选项，你可以根据实际需要，参考 nf_conntrack 的<a href="https://www.kernel.org/doc/Documentation/networking/nf_conntrack-sysctl.txt">文档</a>来配置。</p><h2>小结</h2><p>今天，我带你一起学习了，如何排查和优化 NAT 带来的性能问题。</p><p>由于 NAT 基于 Linux 内核的连接跟踪机制来实现。所以，在分析 NAT 性能问题时，我们可以先从 conntrack 角度来分析，比如用 systemtap、perf 等，分析内核中 conntrack 的行文；然后，通过调整 netfilter 内核选项的参数，来进行优化。</p><p>其实，Linux 这种通过连接跟踪机制实现的 NAT，也常被称为有状态的 NAT，而维护状态，也带来了很高的性能成本。</p><p>所以，除了调整内核行为外，在不需要状态跟踪的场景下（比如只需要按预定的IP和端口进行映射，而不需要动态映射），我们也可以使用无状态的 NAT （比如用 tc 或基于 DPDK 开发），来进一步提升性能。</p><h2>思考</h2><p>最后，给你留一个思考题。你有没有碰到过 NAT 带来的性能问题？你是怎么定位和分析它的根源的？最后，又是通过什么方法来优化解决的？你可以结合今天的案例，总结自己的思路。</p><p>欢迎在留言区和我讨论，也欢迎你把这篇文章分享给你的同事、朋友。我们一起在实战中演练，在交流中进步。</p><p></p>
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
  <div class="_2_QraFYR_0">https:&#47;&#47;mp.weixin.qq.com&#47;s&#47;VYBs8iqf0HsNg9WAxktzYQ：（多个容器snat时因为搜索本地可用端口（都从1025开始，到找到可用端口并插入到conntrack表是一个非事务并且有时延--第二个插入会失败，进而导致第一个syn包被扔掉的错误，扔掉后重传找到新的可用端口，表现就是时延偶尔为1秒或者3秒）<br><br>这篇文章是我见过诊断NAT问题最专业的，大家要多学习一下里面的思路和手段</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍 谢谢分享。内核问题的分析和排查一般都比较耗时间，对基础知识的要求也会高一些。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-01 16:15:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/3d/3d/66512c23.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ben</span>
  </div>
  <div class="_2_QraFYR_0">其实遇到很多问题的时候多看看内核日志就知道了，linux很智能的，很多报错信息都在日志里面，越遇到系统优化层面，就多要看看内核日志，我一般是使用journalctl -k -f来查看，有报错信息就Google，线上遇到nf_conntrack: table full，就是这样排查出来的，查看内核日志真的很重要，特别应用日志没看出什么来的时候</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-11 17:47:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTL9hlAIKQ1sGDu16oWLOHyCSicr18XibygQSMLMjuDvKk73deDlH9aMphFsj41WYJh121aniaqBLiaMNg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>腾达</span>
  </div>
  <div class="_2_QraFYR_0">这个案例，能不能讲讲怎么找到是NAT问题？这个很关键，但文章里直接点明说是NAT问题，这个就不好了，一般看cpu,看其他指标很难想到是nat问题，真实场景里，怎么会想到是nat问题呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯嗯，是个好问题。下一模块中有内核线程的分析思路，到时候可以看到分析的方法</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-01 10:19:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c8/74/a1bd2307.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>vvccoe</span>
  </div>
  <div class="_2_QraFYR_0">‘# 连接跟踪对象大小为 376，链表项大小为 16<br>nf_conntrack_max* 连接跟踪对象大小 +nf_conntrack_buckets* 链表项大小 <br>= 1000*376+65536*16 B<br>= 1.4 MB’   <br>老师 上面的376和16 是固定值吗？<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这是内核数据结构的大小，一般不会变化</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-27 10:20:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/63/2e/e49116d1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_007</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，有两个问题想请教一下。<br>第一点，了解到ip_conntrack模块既然会限制链接，且调大会导致占用内存，而且调大了也不一定能解决大流量服务器的网络性能问题，我理解是不是应该关掉ip_conntrack模块，因为业务服务器按理说是不需要状态追踪的。<br>第二点，如果我关掉ip_conntrack，会不会因为我执行iptables命令导致该模块被加载，或者执行conntrack命令导致模块被加载。。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. 有很多服务是依赖conntrack的，所以要看实际需求确定<br>2. 要看iptables规则是不是用到了conntrack功能</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-10 21:51:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>无名老卒</span>
  </div>
  <div class="_2_QraFYR_0">很惊讶，之前在线上环境中就出现了kernel: nf_conntrack: table full, dropping packet.的报错，当时就认为是conntrack_max导致的，后面调整了这个值之后就恢复了，但其实那次故障也不应该会加载nf_conntrack模块，因为iptables规则只是设置了几个IP允许登陆服务器，当时也不清楚为什么会去加载这个模块了。<br><br>同时，conntrack_max和conntrack_buckets有没有什么联系呢？从描述中，感觉conntrack_buckets应该要大于conntrack_max才对，但实际 上不是这样，请老师解惑下。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 应该是反过来，nf_conntrack_max是最大连接跟踪数，nf_conntrack_buckets是连接跟踪表（哈希表）大小。哈希表最大也就是最大连接跟踪数</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-01 23:08:53</div>
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
  <div class="_2_QraFYR_0">systemtap这个真牛，还可以追踪内核模块执行，长见识咯！iptables需要学习下</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-01 10:18:41</div>
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
  <div class="_2_QraFYR_0">[D42打卡]<br>今天的内容只能围观了.<br>居然还用了内核动态追踪工具,统计丢包位置.<br>对于我这种完全不了解内核的人来说, 只当是开了眼界.<br><br>对于我来说,目前知道 NAT会带来性能损耗 就行了.🤦‍♀️<br>能避免就避免使用,不能避免了就在请求数较多的场景下调些参数.<br><br>`# 连接跟踪对象大小为 376，链表项大小为 16`<br>这应该是c结构体的大小吧.<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-27 13:43:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/4f/2d/7b5ed6c2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>章星</span>
  </div>
  <div class="_2_QraFYR_0">执行第二个8080的容器的时候提示：<br>&#47;bin&#47;sh: 1: cannot create &#47;proc&#47;sys&#47;net&#47;netfilter&#47;nf_conntrack_max: Permission denied  <br>不知道有没有人碰到，怎么解决呢<br>root@vm101:&#47;home&#47;zhang# docker logs nginx         <br>&#47;bin&#47;sh: 1: cannot create &#47;proc&#47;sys&#47;net&#47;netfilter&#47;nf_conntrack_max: Permission denied<br>root@vm101:&#47;home&#47;zhang# docker ps -a<br>CONTAINER ID   IMAGE              COMMAND                  CREATED          STATUS                      PORTS     NAMES<br>79b0ea8bb408   feisky&#47;nginx:nat   &quot;&#47;bin&#47;sh -c &#39;echo 10…&quot;   11 minutes ago   Exited (2) 11 minutes ago             nginx</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-12 22:36:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/93/1b/bf902c9d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>aliuql</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，docker，是不是必然会用到dnat的端口映射？linux服务器内 nat本身应该也有流量限制吧！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不一定，比如使用hostnetwork就会直接使用host的网络。NAT支持流量限制，不过需要额外的配置</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-31 07:31:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/57/6e/b6795c44.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>夜空中最亮的星</span>
  </div>
  <div class="_2_QraFYR_0">以前只是知道net性能不好，今天通过老师的讲解彻底明白了来龙去脉。<br>公司内部上网用的就是net 人一多就特别慢。<br>业务基本上没用过net。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: NAT，不是net😊</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-27 16:04:45</div>
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
  <div class="_2_QraFYR_0">今天是跟不上了，没有网络基础，进入到网络模块就开始觉得吃力了😂</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 先尝试尝试案例，再回去补补基础</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-27 08:29:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/39/f8/b5a1b669.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Brilliant</span>
  </div>
  <div class="_2_QraFYR_0">$ docker run --name nginx --privileged -p 8080:8080 -itd feisky&#47;nginx:nat<br>执行后容器没有启动起来，日志显示<br>&#47;bin&#47;sh: 1: cannot create &#47;proc&#47;sys&#47;net&#47;netfilter&#47;nf_conntrack_max: Permission denied<br><br>需要为容器中的入口指令提权，如下：<br>$ docker run --name nginx --privileged=true -u=root -p 8080:8080 -itd feisky&#47;nginx:nat &#47;bin&#47;sh<br><br>附docker版本为：20.10.17</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-09 09:49:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/ec/18/bf7254d3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>肥low</span>
  </div>
  <div class="_2_QraFYR_0">这个基础知识应该去哪里学习才能不走弯路呢……</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-05 21:29:11</div>
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
  <div class="_2_QraFYR_0"><br># 连接跟踪对象大小为376，链表项大小为16<br>nf_conntrack_max*连接跟踪对象大小+nf_conntrack_buckets*链表项大小 <br>= 1000*376+65536*16 B<br>= 1.4 MB<br><br>在这样的设置下，每一个bucket 所对应的list 大小是 1000&#47;65536 ? 那就是不到1,  意思是大部门情况都不需要单向链表遍历了吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-02 22:19:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/5d/51/87fc7ef9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Issac慜</span>
  </div>
  <div class="_2_QraFYR_0">在测试系统的并发和新建时，出现过指标上不去的问题。最终就是发现最大连接跟踪数设置的比较小，导致测试不通过</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-05 08:12:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小刚</span>
  </div>
  <div class="_2_QraFYR_0">这篇文章仔细看了几遍，也操作了文中的实际案例。期间遇到的问题及解决方法顺便记录一下：<br><br>1.带NAT的docker启动不起来，docker logs 提示权限不足，修改Dockerfile，进入docker发现实际是 &#47;proc&#47;sys&#47;net&#47;netfilter&#47;nf_conntrack_max文件无写权限<br>2.systemtap按文中步骤安装后，执行脚本报符号找不到，尝试从源码安装仍不可用<br><br>我用的ubuntu版本为最新下载的18.04.5，原因为ubuntu18.04.5的内核版本(5.4)过高导致的兼容性问题，安装18.04.1(内核4.15.0)即可解决这两问题，其中systemtap问题要注意下systemtap和内核版本兼容。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-12 12:18:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/75/3a/a7596c06.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>大大</span>
  </div>
  <div class="_2_QraFYR_0">不知道老师现在还能回答问题么，我们线上出现一个问题，内核比较新5.4，ipvs模式下，只有一台物理机上出现问题，pod内通过service访问服务，当后端的pod在本机时百分之百超时失败，经抓包发现svc后面的pod返回tcp三次握手的第二个包时原ip没有改成service  ipvs 导致client  直接reset</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-20 21:01:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/01/bb/7d15d44a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>欧雄虎(Badguy）</span>
  </div>
  <div class="_2_QraFYR_0">老师我那个脚本不能执行成功，辛苦帮看下，自己百度没解决<br>root@VM-0-5-ubuntu:~# stap --all-modules dropwatch.stp<br>semantic error: while resolving probe point: identifier &#39;kernel&#39; at dropwatch.stp:19:7<br>        source: probe kernel.trace(&quot;kfree_skb&quot;) { locations[$location] &lt;&lt;&lt; 1 }<br>                      ^<br><br>semantic error: no match (similar tracepoints: map, kfree, rdpmc, unmap, read_msr)<br><br>Pass 2: analysis failed.  [man error::pass2]<br>Tip: &#47;usr&#47;share&#47;doc&#47;systemtap&#47;README.Debian should help you get started.<br>root@VM-0-5-ubuntu:~# uname -r<br>4.15.0-54-generic<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-26 17:37:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/42/44/d3d67640.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Hills录</span>
  </div>
  <div class="_2_QraFYR_0">这篇看的太爽了，期待自己早日能够如此这般～</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-26 22:53:18</div>
  </div>
</div>
</div>
</li>
</ul>