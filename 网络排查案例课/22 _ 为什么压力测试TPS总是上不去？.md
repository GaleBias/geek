<audio title="22 _ 为什么压力测试TPS总是上不去？" src="https://static001.geekbang.org/resource/audio/3f/6c/3f3b9222c5ee212b48d2426749b5046c.mp3" controls="controls"></audio> 
<p>你好，我是胜辉。</p><p>在上一讲里，我们排查了一个跟操作系统紧密相关的性能问题。我们结合top和strace这两个工具，抓住了关键点，从而解决了问题。性能问题，确实也是我们日常技术工作中的一个重要话题。在出现性能问题以后，我们要有能力搞定它；而在出现性能问题之前，最好能提前预见到它。而要“预见”性能瓶颈，最好的方法就是做<strong>压力测试</strong>。</p><p>但是，我们在做压力测试的过程中也时常出现预料不到的情况。比如在离预期的瓶颈值还很远的时候，系统就出现了各种意外，影响到压力测试的继续进行。</p><p>那么在这节课里，我们会回顾几个典型的压力测试场景中的网络问题，一起来学习其中的关键要点。同时，我们还会学习一系列跟网络性能相关的压测工具和检测工具，这样以后你遇到类似的问题时，就有所准备了。</p><h2>压测要做什么？</h2><p>压力测试的诉求实际上多种多样，不过大体上可以分为这几种。</p><p><strong>应用的承受能力</strong>：这主要在第七层应用层，比如发起了压测，把服务端的CPU打到95%甚至100%，观察这时候的请求的TPS、请求耗时、并发量等等。而这些对于不同的业务场景，又会有不同的侧重点。比如：</p><ul>
<li>对于时间敏感型业务来说，请求耗时（Latency）这个指标就是关键了。</li>
<li>对于经常做秒杀的电商来说，并发处理量TPS（Transaction Per Second）就是一个核心关注点了。</li>
</ul><!-- [[[read_end]]] --><p><strong>LB的连接处理能力</strong>：这主要在第四层TCP，看LB能最大支持的TCP并发连接数。这时候，发起压测的客户端一般会指定比较大的并发数，这样就可以发起尽量多的TCP连接。</p><p><strong>网络的承受能力</strong>：这可能主要在第三层IP层了，比如测试上行和下行带宽能否跑满、是否有丢包和额外的延迟，等等。特别是对于一些流量比较大的场景，很可能服务端计算能力都还在，但带宽已经不够用了，所以我们要提前发现这些隐患。</p><h2>案例1：压测TPS上不去</h2><p>我们有个客户是传统企业转型做电商，有一次准备搞大促。为了确保大促顺利，他们要提前对网站进行压测。</p><p>客户用的压测工具比较简单，是Apache ab。ab是Apache Benchmark的缩写，它的用途就是对HTTP服务端发起测试，以获得性能指标（Benchmark）。ab本身不是独立安装的，而是在apache2-utils工具包里，所以你可以这样来安装它：</p><pre><code class="language-bash">apt install apache2-utils
</code></pre><p>ab是一个轻量级的工具，因为相对其他重量级的工具比如LoadRunner或者JMeter来说，ab只要一行命令就可以发起压测了，是不是很省事？比如你可以这样：</p><pre><code class="language-bash">ab -c 100 -n 10000 目标URL
</code></pre><p>通过上面的命令，你就用-c 100这个参数，让ab发起了100个并发的请求，而-n 10000指定了总共发送的请求量。</p><blockquote>
<p>补充：这里有一个小的注意点。如果目标URL只是站点名本身，还是需要在结尾处加上“<code>&gt;/</code>”，要不然ab会报这个错误：<code>ab: invalid URL</code></p>
</blockquote><p>比如我用下面这条ab命令，对一个著名网站发起“压测”，当然我的参数选择的很小，只有10个并发，一共100次请求，尽量避免打扰到这个网站。我们可以看一下输出：</p><pre><code class="language-bash">$ ab -c 10 -n 100 https://www.baidu.com/abc
This is ApacheBench, Version 2.3 &lt;$Revision: 1843412 $&gt;
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking www.baidu.com (be patient).....done


Server Software:&nbsp; &nbsp; &nbsp; &nbsp; Apache
Server Hostname:&nbsp; &nbsp; &nbsp; &nbsp; www.baidu.com
Server Port:&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; 443
SSL/TLS Protocol:&nbsp; &nbsp; &nbsp; &nbsp;TLSv1.2,ECDHE-RSA-AES128-GCM-SHA256,2048,128
Server Temp Key:&nbsp; &nbsp; &nbsp; &nbsp; ECDH P-256 256 bits
TLS Server Name:&nbsp; &nbsp; &nbsp; &nbsp; www.baidu.com

Document Path:&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; /abc
Document Length:&nbsp; &nbsp; &nbsp; &nbsp; 201 bytes

Concurrency Level:&nbsp; &nbsp; &nbsp; 10
Time taken for tests:&nbsp; &nbsp;9.091 seconds
Complete requests:&nbsp; &nbsp; &nbsp; 100
Failed requests:&nbsp; &nbsp; &nbsp; &nbsp; 0
Non-2xx responses:&nbsp; &nbsp; &nbsp; 100
Total transferred:&nbsp; &nbsp; &nbsp; 34600 bytes
HTML transferred:&nbsp; &nbsp; &nbsp; &nbsp;20100 bytes
Requests per second:&nbsp; &nbsp; 11.00 [#/sec] (mean)
Time per request:&nbsp; &nbsp; &nbsp; &nbsp;909.051 [ms] (mean)
Time per request:&nbsp; &nbsp; &nbsp; &nbsp;90.905 [ms] (mean, across all concurrent requests)
Transfer rate:&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; 3.72 [Kbytes/sec] received

Connection Times (ms)
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; min&nbsp; mean[+/-sd] median&nbsp; &nbsp;max
Connect:&nbsp; &nbsp; &nbsp; 141&nbsp; 799 429.9&nbsp; &nbsp; 738&nbsp; &nbsp; 4645
Processing:&nbsp; &nbsp; 19&nbsp; &nbsp;67 102.0&nbsp; &nbsp; &nbsp;23&nbsp; &nbsp; &nbsp;416
Waiting:&nbsp; &nbsp; &nbsp; &nbsp;17&nbsp; &nbsp;67 101.8&nbsp; &nbsp; &nbsp;23&nbsp; &nbsp; &nbsp;416
Total:&nbsp; &nbsp; &nbsp; &nbsp; 162&nbsp; 866 439.7&nbsp; &nbsp; 796&nbsp; &nbsp; 4666

Percentage of the requests served within a certain time (ms)
&nbsp; 50%&nbsp; &nbsp; 796
&nbsp; 66%&nbsp; &nbsp; 877
&nbsp; 75%&nbsp; &nbsp; 944
&nbsp; 80%&nbsp; &nbsp;1035
&nbsp; 90%&nbsp; &nbsp;1093
&nbsp; 95%&nbsp; &nbsp;1339
&nbsp; 98%&nbsp; &nbsp;1530
&nbsp; 99%&nbsp; &nbsp;4666
&nbsp;100%&nbsp; &nbsp;4666 (longest request)
</code></pre><p>结尾部分的多个Percentage，确切地说就是Percentile，也就是百分位。比如50% 796这一行的意思是：第50个请求（因为总数是100个）的耗时小于等于796毫秒，另外50个请求大于796毫秒。那么我们也可以知道，耗时最长的那个就是排最后一名的4666，它的耗时是4666毫秒。</p><p>这次，客户用ab时指定的具体参数是这样的：</p><pre><code class="language-bash">ab -k -c 500 -n 100000 http://site.name.com/path
</code></pre><p>也就是并发数为500，总次数为10万，而-k参数是启用长连接。然后就是查看以下两个主要指标：</p><ul>
<li>ab这个客户端的耗时分布，也就是前面刚介绍过的各个百分位的耗时数值。</li>
<li>服务端的性能指标，也就是在这个压力下面，服务端的CPU、内存、网络丢包率等的统计数值。</li>
</ul><p><img src="https://static001.geekbang.org/resource/image/8c/9d/8ce9cb87baccc568020963321f3ff39d.jpg?wh=1575x445" alt=""></p><p>压测过程中，客户发现测试端的带宽用到400Mbps后，TPS就上不去了，无论把并发量或者总量的数值进行怎样的调整，TPS都会维持在一个稳定的数值。</p><pre><code class="language-bash">Transfer rate:&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; 52087.82 [Kbytes/sec] received
</code></pre><p>那究竟是不是购买的服务端主机的性能不够的原因呢？要知道，客户端和服务端都是1Gbps的网卡，客户是想压满100%的带宽，现在只用到了40%的带宽，TPS就上不去了。</p><p>为什么会这样呢？</p><p>我们就跟客户配合，一边做ab压测，一边做了4秒钟的tcpdump抓包，这对采样分析来说也够用了。然后打开抓包文件，查看专家信息（Expert Information），如下：</p><blockquote>
<p>补充：抓包示例文件已经上传至<a href="https://gitee.com/steelvictor/network-analysis/tree/master/22">Gitee</a>，建议结合抓包文件和文稿一起学习。</p>
</blockquote><p><img src="https://static001.geekbang.org/resource/image/d6/00/d64b27b7a79ce3edcd702d0fcffdb900.jpg?wh=1822x592" alt="图片"></p><p>这次我就不做标记了，你自己先找找看，能否找到一些问题？</p><p>你有没有发现，GET有1603次（倒数第三行），而RST有1982次（第三行），比GET这种HTTP请求的次数还更多。也就是说，平摊的话，每次GET请求对应了一次以上的RST。这个现象会不会跟TPS上不去的问题有关系呢？</p><p>我们选一个RST来看一下情况。比如14031号报文：</p><p><img src="https://static001.geekbang.org/resource/image/46/28/4610b0fd0352d03523f448480c256428.jpg?wh=1920x847" alt="图片"></p><p>然后Follow -&gt; TCP Stream，就来到了这条TCP流：</p><p><img src="https://static001.geekbang.org/resource/image/fa/d0/fa6d59359e1044a4844b6a29c3a260d0.jpg?wh=1920x782" alt="图片"></p><p>在这里，你有没有发现两个重传报文？一个是6504号报文，一个是7340号报文。我们再进一步看一下这两个的重传的原因是什么。一般来说，判断一个报文是否是重传，最方便的就是借助Wireshark本身提供的提示，Wireshark说是Retransmission，那就是了。</p><p>那么假设世界上还没有Wireshark，你又<strong>如何判断一个报文是否是重传呢？</strong>其实可以根据两个关键信息：<strong>序列号、载荷长度</strong>。</p><p>我在之前的课程里提到过，序列号本身反映的就是字节位置，载荷的长度就表示这段报文的实际字节长度，这两个信息就确定了信息的<strong>起点和终点</strong>，也就是决定了一个报文是否是之前报文的重传。</p><p>在上面的截图里，我们根据序列号（Sequence Number列）和载荷长度（TCP Seglen列），判断出下面两次重传：</p><p><img src="https://static001.geekbang.org/resource/image/10/fa/10e8909974fd01608f6f7a976949b7fa.jpg?wh=1966x550" alt=""></p><p>可见，7340是3155报文的重传，因为它们的序列号都是9593，载荷长度都是1061。同样的道理，6504和3156也是一对重传关系。</p><p>当然，我们也可以直接在Wireshark里选中7340号报文，它的基础报文也就是3155的<strong>前面会出现一个小圆点</strong>，就像下图这样：</p><p><img src="https://static001.geekbang.org/resource/image/e4/57/e460a19638fb3b633e1315cc02b07857.jpg?wh=1920x354" alt="图片"></p><p>总之，我们确认TCP传输中是有丢包和重传现象的，我们可以回头从专家信息那里得到证实，这里显示有184个Suprious重传和2274个（超时）重传。</p><p><img src="https://static001.geekbang.org/resource/image/2d/9d/2d44ff9320c8bf3e72ff8e907a9eyy9d.jpg?wh=1794x78" alt="图片"></p><blockquote>
<p>补充：而出现RST的原因，是客户端在已经没有了这个TCP连接的情况下，收到了服务端的ACK报文。从现象来看，客户端应该是做了TIME_WAIT优化的设置，我们后面的实验里也会带到。</p>
</blockquote><p>考虑到这是在压测场景中出现的丢包现象，我们根据经验，想到了<strong>网卡包量</strong>的问题。</p><p>一般说到网络性能，我们会讨论的就是带宽、时延、网速等等这些指标。实际上，另外一个性能指标常常在达到带宽极限之前就已经触顶了，它就是包量。</p><p><strong>包量是对PPS（Packet Per Second）的简称，一般用来衡量一台主机的网络处理能力。</strong>那么，这次是不是触及了服务端主机的包量上限了呢？我们又如何检查包量指标呢？</p><p>我们需要用的工具是sar。</p><h3>性能工具sar</h3><p>sar是sysstat工具集的一部分。这个工具集包含了一些很有用的工具，除了sar，还有mpstat、iostat等等。如果你平时经常做操作系统维护方面的工作，应该对它们比较熟悉了。这些工具还有一个共同的特点，运行的时候一般都是加上“间隔 次数”这样的参数的。也就是下面这样：</p><pre><code class="language-bash">sar -n DEV 1 10   #查看网卡性能
iostat 1 10       #查看IO性能
mpstat 2 5        #查看CPU性能
</code></pre><p>这些工具会按指定的时间间隔，运行指定的次数后再退出。这样我们就可以观察到这段时间内的多次输出了，很适合通过这种多次的观察，得到比较准确的性能数据趋势。</p><p>这次我们也在被压测的服务端云主机上运行了 <code>sar -n DEV</code>，得到了以下的性能数据：</p><p><img src="https://static001.geekbang.org/resource/image/55/be/5523ee0f7eba2cce7979620e4abccabe.jpg?wh=749x338" alt="图片"></p><p>前面说的包量，就是体现在两个pck/s指标中，一个是 <strong>rxpck/s</strong>，就是接收方向的包量；一个是 <strong>txpck/s</strong>，就是发送方向的包量。显然，图中的数值已经在5万左右，这个正是当时我们云主机性能的上限。难怪服务端已经无法提供更高的TPS了，因为网络包的处理都来不及做了。</p><p>那你可能会问了：“网络包也有大有小，这个包量指标说的是大包还是小包呢？”</p><p>其实，一般的包量测试，不是随便什么大小的报文都可以测试，而是普遍使用64字节长度的IP报文。另外，我们也要认识到，包的大小对包量性能的影响也不是很大。这是因为，对于网络处理来说，主要的开销在包的头部的处理上，而载荷本身的处理是很快的。</p><p>那么，对于客户遇到的这种包量达到上限的情况，我们可以选择的应对办法是这样的：</p><ul>
<li>选择更高网络性能的主机，比如硬件的RSS、软件的RPS等特性，都会大幅提升包量处理性能。</li>
<li>对服务集群进行水平扩展，也就是在LB后面增加服务器，这样VIP作为一个整体提供的包，处理能力也就提升了。</li>
</ul><h2>案例2：LoadRunner压测发现部分失败</h2><p>这是另外一个客户，他们没有用ab这样的简单工具，而是用了LoadRunner这样一个企业级的测试软件。LoadRunner的厉害之处在于，不仅可以发起巨大的请求量，而且可以模拟用户的复杂行为，比如登录、浏览、加入购物车等等。这一系列事务有前后状态关系，这就不是简单的ab可以做到的了。</p><p>但是测试结果中的小几十个Failed和Error引起了客户的疑虑。他们怀疑：是不是我们公有云的机器或者网络质量不行，所以才导致了这些失败呢？</p><p><img src="https://static001.geekbang.org/resource/image/91/cb/9185cd1c33463aa7594bf290547fb1cb.jpg?wh=450x248" alt=""></p><p>我们就尝试复现这个压测场景，同时也对测试机上的TCP资源情况做检查，其中最主要的就是源端口了。</p><p>为什么会想到这个呢？因为压测发起的请求，都是依托于TCP连接的。我们在<a href="https://time.geekbang.org/column/article/477510">第1讲</a>里就提到过：<strong>TCP连接是基于五元组的</strong>。那么对于客户端来说，源IP、目的IP、目的端口、协议，这四个元素都不会变化，<strong>唯一会变的就是自己的源端口</strong>了。那么我们来看看，当时测试机的源端口情况。</p><p>这是一台Window机器，跟Linux类似，我们执行这条命令：</p><pre><code class="language-bash">netstat -ant
</code></pre><p>输出如下图：</p><p><img src="https://static001.geekbang.org/resource/image/ce/a4/cee3b9c775369dba1cfd2aaa2c6d8fa4.jpg?wh=550x328" alt=""></p><p>原来，确实有大量的TCP连接都在TIME_WAIT状态，尤其是源端口已经用到了<strong>65534</strong>。所以可以肯定，这次压测中出现失败的原因，就在于源端口耗尽。</p><p>那要如何处理呢？显然我们也变不出更多的端口来。这个时候，我们可以<strong>调整压测软件的设置，比如从短连接改成长连接</strong>。这样就可以避免源端口耗尽的情况，因为所有的请求是在长连接里完成的，只要连接池本身设置合理，源端口就不会被用完。</p><h2>案例3：压测报cannot assign requested address错误</h2><p>我们内部有一个团队在做业务压力测试，结果遇到了一个奇怪的报错：cannot assign requested address。这次的场景是这样的：这个团队从多台客户端机器向LB上的一个VIP，发送大量的请求，而LB的后面就是很多的服务器。</p><p><img src="https://static001.geekbang.org/resource/image/6d/fb/6d0449c9c02b508eaa5bc7956c0194fb.jpg?wh=1858x711" alt=""></p><p>但是，还没等到这些服务器的负载跑起来，客户端那边提前报错了。这是一个Go语言的程序，具体的报错信息如下：</p><pre><code class="language-plain">https://test.vip/a": dial tcp 10.123.123.12:443: connect: cannot assign requested address
http post Post \"https://test.vip/b": dial tcp 10.123.123.12:443: connect: cannot assign requested address retry 1 times
</code></pre><p>于是他们就怀疑：既然报错是连接出错，那是不是LB有什么问题呢？比如，是否LB本身处理能力不够，不能服务这么大量的连接请求？否则为什么客户端会报connect的错误呢？要知道，用来发起TCP连接的系统调用（syscall）就是connect。</p><p>确实，这个报错信息“dial tcp 10.123.123.12:443: connect: cannot assign requested address”说得不太清楚。表面上看，就是客户端往10.123.123.12:443这个VIP发起连接请求（用connect）然后遇到了报错，也难怪测试团队会找到我们来查看LB的问题。</p><p>我们检查了LB，发现一切正常。于是把排查方向换到客户端。在这次测试中，有一个参数是关于HTTP Keep-alive的。如果你对课程前面的内容还有印象的话，应该还记得我们在第7讲“<a href="https://time.geekbang.org/column/article/482610">保活机制</a>”里，深入讨论过这个问题。</p><p>简单来说，这次的测试配置里，HTTP Keep-alive没有打开，导致这些TCP连接被视作短连接来处理了，也就是一次HTTP请求和响应完成后，这条连接就关闭了。由于发起关闭的是客户端自己，于是这条连接也就进入了TIME_WAIT状态。</p><p>而要查看TIME_WAIT状态的连接数量，我们可以用 <strong>netstat命令</strong>，配合管道和awk来完成统计。比如，这次我在一台客户端机器上执行了下面的命令：</p><pre><code class="language-bash">$ netstat -ant |awk '{++a[$6]} END{for (i in a) print i, a[i]}'
TIME_WAIT 28231
</code></pre><p>或者用下面的 <strong>ss命令</strong>会更快：</p><pre><code class="language-bash">ss -ant | awk '{++s[$1]}END{for(k in s) print k,s[k]}'
TIME-WAIT 28231
</code></pre><p>可见，处于TIME_WAIT状态的连接数接近3万个，差不多就是Linux的本地动态端口的范围了。我们随后检查这台Linux机器的本地源端口范围，执行了下面的命令：</p><pre><code class="language-plain">$ cat /proc/sys/net/ipv4/ip_local_port_range
32768	60999
</code></pre><p>于是发现它的下限是32768，上限是60999，范围正好就是28231，跟TIME_WAIT的数量一致，显然也是一次源端口耗尽导致的压测问题。</p><p>当然，用sysctl也一样可以查看这个范围：</p><pre><code class="language-bash">sysctl net.ipv4.ip_local_port_range
</code></pre><p>不过你可能会问：“难道压测中的网络排查，除了包量和源端口，就没有别的问题了吗？”</p><p>我们来看一个不一样的案例。</p><h2>案例4：压测遇到connection reset by peer</h2><p>这次的场景是一个应用团队用netty http client去调用一个VIP做压测。具体来说是9台客户端机器，向同一个LB VIP发起大量的请求。</p><p><img src="https://static001.geekbang.org/resource/image/c4/9e/c42770f906ab7yyf9b964757d120069e.jpg?wh=1866x711" alt=""></p><p>结果遇到了connection reset by peer。蹊跷的是，这个报错是零零星星出现的。而像之前的例子，如果真的源端口用尽了，那么接下来一段时间内，这些请求都会因为本地没有源端口可用而宣告失败，也就是报错会是大面积出现，而不会零星出现。</p><p><img src="https://static001.geekbang.org/resource/image/a0/72/a086cf71e10b0c1f41f79978f6bd1a72.png?wh=1920x1120" alt="图片"></p><p>从图上看，报错数量不超过50个，主要集中在11点20分到11点35分这个时间段，这正是压测的时段。因为报错只有50来个，这相比于这次压测发起的成千上万的请求来说，是很小的比例了。</p><blockquote>
<p>补充：Y轴就是报错的个数，最高值是10，我们把几个柱体的高度加起来，就是报错的总数量了。</p>
</blockquote><p>我们了解到，压测期间每个客户端的请求频率是700TPS，所以9台客户端一共会发起6300TPS的请求量。这个问题诡异的地方就是，大部分的请求都能得到正确及时的回复，但是隔了两三分钟，就会出现几次这种connection reset by peer的问题。</p><p>那我们需要理解一下，<strong>为什么会reset</strong>。</p><p>我们在做网络排查的时候，如果在Wireshark里看到TCP RST，往往会觉得它不是一个好的征兆。确实，有时候是RST引起了故障，有时候又是网络故障迫使TCP用RST来结束连接。无论RST是因还是果，它总是跟问题本身逃不脱关系。</p><p>那干脆在TCP里取消RST，是不是很多问题就会被解决呢？当然不是，RST在TCP里面是一个非常必要的组成部分。没有RST，其实就没有“坏”的结束，也就没有“好”的开始。</p><p>大体上，TCP RST的原因可以分为这么几个大类：</p><ul>
<li>找不到相关连接，那么接收端可以放心地直接发送RST。</li>
<li>找到了相关连接，但收到的报文不符合TCP规范，那么接收端也可以发送RST。</li>
<li>找到了相关连接，但传输状况恶劣，内核选择及时“止损”，发送RST。</li>
</ul><p><img src="https://static001.geekbang.org/resource/image/de/d6/de7ec3cb7ccdc5c7da395f271dc935d6.jpg?wh=1861x890" alt=""></p><p>而这次案例里面的情况，就符合第一种。这是因为：</p><ul>
<li>我们的LB上的VIP有一个idle timeout的设置，如果客户端在一定时限内不发任何报文，那么这条连接将被回收。这个时限是180秒，而且回收时不向客户端发送FIN或者RST报文。</li>
<li>这次的压测客户端框架里，有一个设置值也是关于idle timeout的。不过这个值设置的是360秒。</li>
</ul><p>你有没有发现问题？这两个idle timeout值一边大一边小，配合不当，就会出现下面这样的问题：</p><p><img src="https://static001.geekbang.org/resource/image/02/9d/02372ef7e9f63a76aa9f1c8e82957d9d.jpg?wh=2000x1125" alt=""></p><p>某条TCP连接中完成一次HTTP请求和响应之后，连接没有被关闭。过了180秒，LB这一侧的连接被回收了，但客户端那边还没到360秒，所以还认为这条连接是活着的，于是在180秒之后发起了一次请求。报文到达LB，后者在连接表里没有查到这条连接，于是回复了RST。</p><p>根因清楚了，解决起来异常简单：把客户端的Idle timeout参数，从原先的360秒，改成比LB的180秒更低的值就好了。这次改到了120秒，RST就完全消失了。</p><p>你看，虽然在案例2里，我们知道了压测最好使用长连接，这样可以避免源端口耗尽的问题。但是不等于你设置了长连接就一劳永逸了，还需要考虑idle timeout的问题。这次压测的关键在于：要确保长连接本身是真正有效的，你需要<strong>确保客户端的idle timeout小于服务端的idle timeout，这样才能避免连接失效导致的RST</strong>。</p><blockquote>
<p>补充：这个库是Ractor-Netty，idle timeout对应的是<a href="https://projectreactor.io/docs/netty/release/api/reactor/netty/resources/ConnectionProvider.ConnectionPoolSpec.html#evictInBackground-java.time.Duration-">evictInBackground</a> + <a href="https://projectreactor.io/docs/netty/release/api/reactor/netty/resources/ConnectionProvider.ConnectionPoolSpec.html#maxLifeTime-java.time.Duration-">maxLifeTime</a>。</p>
</blockquote><h2>实验</h2><p>回顾完这些压测案例，你可能会发现其实难度并不太大，是不是也跃跃欲试了呢？接下来，我们就用ab来做个小实验，这次还捎带一个小任务：<strong>控制TIME_WAIT状态的连接的数量</strong>。</p><p>在案例2和3里面，源端口耗尽的问题，本质是TIME_WAIT需要停留2MSL的时长。要解决TIME_WAIT连接过多的问题，方法有很多。这次我们只实验其中的一种方法，就是修改 <code>net.ipv4.tcp_max_tw_buckets</code> 这个内核参数。</p><p>它的默认值为16384，改为更小的值后，超过这个数值的TIME_WAIT连接会被清除，这样，TIME_WAIT连接数就被控制住了。</p><p>这个实验在本地就可以完成，比如在你的笔记本上安装VirtualBox，然后在这个虚拟机里面完成实验。</p><p>步骤一：安装Nginx和Apache ab。</p><pre><code class="language-bash">apt install nginx
apt install apache2-utils
</code></pre><p>步骤二：启动ab测试，执行下面的命令。</p><pre><code class="language-bash">ab -c 500 -n 10000 http://localhost/
</code></pre><p>步骤三：观察TCP连接的各种状态的统计数。</p><pre><code class="language-bash">ss -ant | awk '{++s[$1]}END{for(k in s) print k,s[k]}'
</code></pre><p>此时你应该会看到1万多个TIME_WAIT的连接，然后等待2分钟后继续步骤四。</p><p>步骤四：修改内核参数 <code>tcp_max_tw_buckets</code> 为100。</p><pre><code class="language-bash">sysctl net.ipv4.tcp_max_tw_buckets=100
</code></pre><p>步骤五：再次ab测试。</p><pre><code class="language-bash">ab -c 500 -n 10000 http://localhost/
</code></pre><p>步骤六：再次观察TCP连接的各种状态的统计数。</p><pre><code class="language-bash">ss -ant | awk '{++s[$1]}END{for(k in s) print k,s[k]}'
</code></pre><p>此时TIME_WAIT应该只有100了。</p><p>当然，TIME_WAIT本身被设计出来是有原因的，建议你在改动它之前，把自己的场景想清楚，然后再改不迟。</p><h2>小结</h2><p>这节课，我们回顾了几个典型的压力测试场景中的网络问题，你需要重点掌握以下这些知识点。</p><p><strong>关于压测指标和工具。</strong>轻量级的工具是ab，它十分方便，而且也可以发起大量的请求，在简单的压测场景下很有用。比如下面的命令：</p><pre><code class="language-bash">ab -k -c 100 -n 10000 目标URL
</code></pre><p>重量级工具是LoadRunner和JMeter，它们可以实现复杂的交互行为，也提供更加详细的报告。</p><p>在网络性能领域，<strong>包量是对PPS的简称，一般用来衡量一台主机的网络处理能力。</strong>而评估包量的工具是sar，我们可以运行下面的命令获取到实时的包量值：</p><pre><code class="language-bash">sar -n DEV 1 10
</code></pre><p><strong>关于TCP的知识。</strong>这节课里我们也再次温习了TCP相关的知识，包括：</p><ul>
<li>判断一个报文是否是之前报文的重传，可以根据两个关键信息：<strong>序列号、载荷长度</strong>。这两个值分别相同的多个报文，互相就是重传关系了。</li>
<li>压测数据起不来的一个常见原因是<strong>源端口耗尽</strong>，要验证这一点，可以执行以下命令，来查看TIME_WAIT或者ESTABLISHED状态的连接数是否到达了上限：</li>
</ul><pre><code class="language-bash">ss -ant | awk '{++s[$1]}END{for(k in s) print k,s[k]}'
</code></pre><p>或者：</p><pre><code class="language-bash">netstat -ant |awk '{++a[$6]} END{for (i in a) print i, a[i]}'
</code></pre><ul>
<li>而连接数到达上限值的问题，往往跟可用的<strong>本地源端口范围</strong>有关。要查看端口范围，我们可以执行这条命令：</li>
</ul><pre><code class="language-bash">sysctl net.ipv4.ip_local_port_range
</code></pre><ul>
<li>压测中遇到RST的原因，可能跟两端的idle timeout不合理有关，建议把客户端的idle timeout设置为比服务端idle timeout更低的值。</li>
</ul><h2>思考题</h2><p>最后还是再给你留两道思考题：</p><ul>
<li>测试HTTP性能经常用ab，那测试带宽本身，一般用什么工具呢？</li>
<li>要把TIME_WAIT停留的时间2MSL改小，你知道用什么方法吗？</li>
</ul><p>欢迎在留言区分享你的答案，也欢迎你把今天的内容分享给更多的朋友。</p><h2>附录</h2><p>抓包示例文件：<a href="https://gitee.com/steelvictor/network-analysis/tree/master/22">https://gitee.com/steelvictor/network-analysis/tree/master/22</a></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/c4/03/f753fda7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>JianXu</span>
  </div>
  <div class="_2_QraFYR_0">当年蒋公领导Cms 项目，压测必须见到cpu , mem 或者IO 之一见顶，不然不算过。客户端压测必须有多机并发避免受限单机性能。需要做大小包测试，对应到cms 就是小的文档和大的文档压测。目标机也需要多机集群，因为牵涉到服务端同步。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这些都很全面了：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-12 14:48:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/8f/cf/890f82d6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>那时刻</span>
  </div>
  <div class="_2_QraFYR_0">请问老师，文中提到：对于网络处理来说，主要的开销在包的头部的处理上，而载荷本身的处理是很快的。指的是内核需要处理头部信息所消耗的开销么？而载荷是由应用程序处理，不计入网络处理？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 因为对操作系统来说，处理报文的开销主要就是对头部的拆解和封装，而对载荷做的结构化处理就很少，对载荷的操作主要是内存复制。比如一个64字节的报文和一个640字节的报文，对CPU的开销差别不大，但是带宽上的差异就大了，所以大报文测试可能先触及带宽瓶颈；小报文测试，可能先触及CPU瓶颈。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-11 10:29:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>holiday</span>
  </div>
  <div class="_2_QraFYR_0">案例一的5万pck&#47;s达到这台云主机的上限，这个老师能否详细解释一下？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你好，这个数值是我们已知的瓶颈值，你们也可以用这种方式发起压测，然后用sar -n DEV查看包量。如果无论怎么压测，这个包量值就维持在某个数值（甚至有时候还会降低一些），那说明这个就是这台主机的瓶颈值了~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-02 09:46:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2a/14/76/304fb890.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>陈海松</span>
  </div>
  <div class="_2_QraFYR_0">请问老师，最近我们这边遇到一个问题，2个系统建立tCP连接总是在1500个左右，超不过1600.遇到业务高峰期，就会出现连运维中心维护终端SSH都连接不上或者需要多次尝试才能登录系统。系统允许打开文件数、文件句柄数调整了，没有效果。系统负荷、内存占用率都不高，网络带宽也足够，麻烦老师协助帮忙分析分项，盼复，谢谢！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 您好，你提到高峰期TCP连接数不超过1600，应该是指ESTABLISHED状态的连接数吗？还是指活跃的并发连接数（比如一些网络IO库提供的连接池设置）？你可以看看出问题时候是不是有大量的连接建立和销毁，这部分隐形的开销也可能引起问题。另外可以查看SYN queue是否有溢出的情况。单纯从文字来看比较难给出准确的意见，有必要的话也可以到学习群详细讨论~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-14 13:07:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/4e/bd/7fc7c14f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>汤玉民</span>
  </div>
  <div class="_2_QraFYR_0">包量的上限是算的还是测试得到的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你好，测试到极限后，无论客户端再增加多少请求量，服务端的包量数值就再也上不去了，这个时候就是包量的极限值了。也就是测出来的~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-30 07:43:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/7f/d3/b5896293.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Realm</span>
  </div>
  <div class="_2_QraFYR_0"><br>1 iperf 和 netperf 都是最常用的网络性能测试工具；<br><br>2 也可以通过修改内核参数，优化tw的问题<br>#启用 timewait 快速回收。<br>net.ipv4.tcp_tw_recycle = 1<br><br>#开启重用。允许将 TIME-WAIT sockets 重新用于新的 TCP 连接。<br>net.ipv4.tcp_tw_reuse = 1</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你的补充很好：）<br>不过，tw_recycle这个配置从4.12内核开始被移除了。关于这个变更，这里有详细的说明：https:&#47;&#47;git.kernel.org&#47;pub&#47;scm&#47;linux&#47;kernel&#47;git&#47;torvalds&#47;linux.git&#47;commit&#47;?id=4396e46187ca5070219b81773c4e65088dac50cc</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-11 17:20:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/8f/cf/890f82d6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>那时刻</span>
  </div>
  <div class="_2_QraFYR_0">1.测试带宽，可以使用iPerf工具 <br>2.Linux的TIME_WAIT貌似是hard code为60秒。而阿里云的机器可以通过 sysctl -w &quot;net.ipv4.tcp_tw_timeout=[$TIME_VALUE] 来修改TIME_WAIT。<br>window下可以通过修改注册表 HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\TCPIP\Parameters] &quot;TcpTimedWaitDelay&quot;=dword:0000001E </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1.是的<br>2.确实，Linux的这个值要修改只能重新编译内核了，阿里云的机器提供了方便的sysctl接口来修改这个值~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-11 10:53:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/eb/09/ba5f0135.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Chao</span>
  </div>
  <div class="_2_QraFYR_0">Idle timeout  导致连接被rest。 遇到一个相同案例， 浏览器做法是遇到这种情况 进行了重试。 虽然有http headers 里可以申明 keepalive 超时时间 。 但好像没有客户端实现其读取并主动断开。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对，现代浏览器都有重试机制，可以对不少网络状况进行容错~<br>关于你提到的服务端返回的Keep-Alive头部，我们可以看下MDN的解释：<br>https:&#47;&#47;developer.mozilla.org&#47;en-US&#47;docs&#47;Web&#47;HTTP&#47;Headers&#47;Keep-Alive<br><br>timeout: An integer that is the time in seconds that the host will allow an idle connection to remain open before it is closed. A connection is idle if no data is sent or received by a host. A host may keep an idle connection open for longer than timeout seconds, but the host should attempt to retain a connection for at least timeout seconds.<br>max: An integer that is the maximum number of requests that can be sent on this connection before closing it. Unless 0, this value is ignored for non-pipelined connections as another request will be sent in the next response. An HTTP pipeline can use it to limit the pipelining.<br><br>也就是说，timeout值是连接保持的最短时间，但可以超过这个时间。<br>max是指开启了http pipeline以后才有意义，但大部分http场景是不启用Pipeline的，所以这个值一般就是被忽略了：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-11 10:24:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/b3/c5/7fc124e2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Liam</span>
  </div>
  <div class="_2_QraFYR_0">带宽测试：iperf, pktgen<br>修改timewait的时间： 修改TCP_TIME_WAIT, 重新编译内核。默认2MSL是60s<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，修改TW时间需要重新编译内核。国内有云厂商提供了定制版的linux，可以用sysctl来动态修改这个参数，就方便很多了~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-11 08:25:53</div>
  </div>
</div>
</div>
</li>
</ul>