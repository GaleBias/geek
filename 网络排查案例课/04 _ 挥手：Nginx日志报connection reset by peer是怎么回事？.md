<audio title="04 _ 挥手：Nginx日志报connection reset by peer是怎么回事？" src="https://static001.geekbang.org/resource/audio/88/2c/88f37c17501dd5aa2d717157fc40f22c.mp3" controls="controls"></audio> 
<p>你好，我是胜辉。今天这节课，我们要通过实际的案例，来学习下TCP挥手的知识，在实战中加深对这些知识的理解。</p><p>我们在做一些应用排查的时候，时常会在日志里看到跟TCP有关的报错。比如在Nginx的日志里面，可能就有connection reset by peer这种报错。“连接被对端reset（重置）”，这个字面上的意思是看明白了。但是，心里不免发毛：</p><ul>
<li>这个reset会影响我们的业务吗，这次事务到底有没有成功呢?</li>
<li>这个reset发生在具体什么阶段，属于TCP的正常断连吗？</li>
<li>我们要怎么做才能避免这种reset呢？</li>
</ul><p>要回答这类追问，Nginx日志可能就不够用了。</p><p>事实上，网络分层的好处是在于每一层都专心做好自己的事情就行了。而坏处也不是没有，这种情况就是如此：应用层只知道操作系统告诉它，“喂，你的连接被reset了”。但是为什么会被reset呢？应用层无法知道，只有操作系统知道，但是操作系统只是把事情处理掉，往内部reset计数器里加1，但也不记录这次reset的前后上下文。</p><p>所以，为了搞清楚connection reset by peer时具体发生了什么，我们需要<strong>突破应用层这口井，跳出来看到更大的网络世界</strong>。</p><h2>在应用层和网络层之间搭建桥梁</h2><!-- [[[read_end]]] --><p>首先，你需要理解下connection reset by peer的含义。熟悉TCP的话，你应该会想到这大概是对端（peer）回复了TCP RST（也就是这里的reset），终止了一次TCP连接。其实，这也是我们<strong>做网络排查的第一个要点：把应用层的信息，“翻译”成传输层和网络层的信息</strong>。</p><p>或者说，我们需要完成一件有时候比较有挑战的事情：把应用层的信息，跟位于它下面的传输层和网络层的信息联系起来。</p><p><img src="https://static001.geekbang.org/resource/image/86/0a/869e8f5d68484b4d2a4cac41c901070a.jpg?wh=2000x830" alt=""></p><p>这里我说的“应用层信息”，可能是以下这些：</p><ul>
<li><strong>应用层日志</strong>，包括成功日志、报错日志，等等；</li>
<li><strong>应用层性能数据</strong>，比如RPS（每秒请求数），transaction time（处理时间）等；</li>
<li><strong>应用层载荷</strong>，比如HTTP请求和响应的header、body等。</li>
</ul><p>而“传输层/网络层信息”，可能是以下种种：</p><ul>
<li><strong>传输层</strong>：TCP序列号（Sequence Number）、确认号（Acknowledgement Number）、MSS（Maximum Segment Size）、接收窗口（Receive Window）、拥塞窗口（Congestion Window）、时延（Latency）、重复确认（DupAck）、选择性确认（Selective Ack）、重传（Retransmission）、丢包（Packet loss）等。</li>
<li><strong>网络层</strong>：IP的TTL、MTU、跳数（hops）、路由表等。</li>
</ul><p>可见，这两大类（应用vs网络）信息的视角和度量标准完全不同，所以几乎没办法直接挂钩。而这，也就造成了问题排查方面的两大鸿沟。</p><ul>
<li><strong>应用现象跟网络现象之间的鸿沟</strong>：你可能看得懂应用层的日志，但是不知道网络上具体发生了什么。</li>
<li><strong>工具提示跟协议理解之间的鸿沟</strong>：你看得懂Wireshark、tcpdump这类工具的输出信息的含义，但就是无法真正地把它们跟你对协议的理解对应起来。</li>
</ul><p>也就是说，<strong>你需要具备把两大鸿沟填平的能力</strong>，有了这个能力，你也就有了<strong>能把两大类信息（应用信息和网络信息）联通起来的“翻译”的能力。</strong>这正是网络排查的核心能力。</p><p>既然我们是案例实战课程，这些知识从案例里面学，是最高效的方法了。接下来，就让我们一起看两个案例吧。</p><h2>案例1：connection reset by peer?</h2><p>前几年我在UCloud服务的时候，有个客户也是反馈，他们的Nginx服务器上遇到了很多connection reset by peer的报错。他们担心这个问题对业务产生了影响，希望我们协助查清原因。客户的应用是一个普通的Web服务，架设在Nginx上，而他们的另外一组机器是作为客户端，去调用这个Nginx上面的Web服务。</p><p>架构简图如下：</p><p><img src="https://static001.geekbang.org/resource/image/2d/32/2d02ff0c0d5416335cf7b6b749715832.jpg?wh=2000x539" alt=""></p><p>前面我说过，单纯从应用层日志来看的话，几乎难以确定connection reset by peer的底层原因。所以，我们就展开了抓包工作。具体做法是：</p><ul>
<li>我们需要选择一端做抓包，这次是客户端；</li>
<li>检查应用日志，发现没几分钟就出现了connection reset by peer的报错；</li>
<li>对照报错日志和抓包文件，寻找线索。</li>
</ul><p>我们先看一下，这些报错日志长什么样子：</p><pre><code class="language-plain">2015/12/01 15:49:48 [info] 20521#0: *55077498 recv() failed (104: Connection reset by peer) while sending to client, client: 10.255.252.31, server: manager.example.com, request: "POST /WebPageAlipay/weixin/notify_url.htm HTTP/1.1", upstream: "http:/10.4.36.207:8080/WebPageAlipay/weixin/notify_url.htm", host: "manager.example.com"
2015/12/01 15:49:54 [info] 20523#0: *55077722 recv() failed (104: Connection reset by peer) while sending to client, client: 10.255.252.31, server: manager.example.com, request: "POST /WebPageAlipay/app/notify_url.htm HTTP/1.1", upstream: "http:/10.4.36.207:8080/WebPageAlipay/app/notify_url.htm", host: "manager.example.com"
2015/12/01 15:49:54 [info] 20523#0: *55077710 recv() failed (104: Connection reset by peer) while sending to client, client: 10.255.252.31, server: manager.example.com, request: "POST /WebPageAlipay/app/notify_url.htm HTTP/1.1", upstream: "http:/10.4.36.207:8080/WebPageAlipay/app/notify_url.htm", host: "manager.example.com"
2015/12/01 15:49:58 [info] 20522#0: *55077946 recv() failed (104: Connection reset by peer) while sending to client, client: 10.255.252.31, server: manager.example.com, request: "POST /WebPageAlipay/app/notify_url.htm HTTP/1.1", upstream: "http:/10.4.36.207:8080/WebPageAlipay/app/notify_url.htm", host: "manager.example.com"
2015/12/01 15:49:58 [info] 20522#0: *55077965 recv() failed (104: Connection reset by peer) while sending to client, client: 10.255.252.31, server: manager.example.com, request: "POST /WebPageAlipay/app/notify_url.htm HTTP/1.1", upstream: "http:/10.4.36.207:8080/WebPageAlipay/app/notify_url.htm", host: "manager.example.com"
</code></pre><blockquote>
<p>补充：因为日志涉及客户数据安全和隐私，我已经做了脱敏处理。</p>
</blockquote><p>看起来最“显眼”的，应该就是那句connection reset by peer。另外，我们其实也可以关注一下报错日志里面的其他信息，这也可以帮助我们获取更全面的上下文。</p><ul>
<li><strong>recv() failed</strong>：这里的recv()是一个系统调用，也就是Linux网络编程接口。它的作用呢，看字面就很容易理解，就是用来接收数据的。我们可以直接<strong>man recv</strong>，看到这个系统调用的详细信息，也包括它的各种异常状态码。</li>
<li><strong>104</strong>：这个数字也是跟系统调用有关的，它就是recv()调用出现异常时的一个状态码，这是操作系统给出的。在Linux系统里，104对应的是ECONNRESET，也正是一个TCP连接被RST报文异常关闭的情况。</li>
<li><strong>upstream</strong>：在Nginx等反向代理软件的术语里，upstream是指后端的服务器。也就是说，客户端把请求发到Nginx，Nginx会把请求转发到upstream，等后者回复HTTP响应后，Nginx把这个响应回复给客户端。注意，这里的“客户端&lt;-&gt;Nginx”和“Nginx&lt;-&gt;upstream”是两条独立的TCP连接，也就是下图这样：</li>
</ul><p><img src="https://static001.geekbang.org/resource/image/cc/70/cc2e1bc3669138f6bc0f9222ae4cf770.jpg?wh=2000x504" alt=""></p><blockquote>
<p>补充：你可能觉得奇怪，明明数据是从外面进入到里面的，为什么里面的反而叫upstream？其实是这样的：<strong>在网络运维的视角上，我们更关注网络报文的流向</strong>，因为HTTP报文是从外部进来的，那么我们认为其上游（upstream）是客户端；<strong>但是在应用的视角上，更关注的是数据的流向</strong>，一般来说HTTP数据是从内部往外发送的，那么在这种视角下，数据的上游（upstream）就是后端服务器了。同样，在HTTP协议规范RFC中，upstream也是指服务端。<br>
&nbsp;<br>
Nginx、Envoy都属于应用网关，所以在它们的术语里，upstream指的是后端环节。这里没有对错之分，你只要知道并且遵照这个约定就好了。</p>
</blockquote><p>到这里，我们既然已经解读清楚报错日志了，接下来就进入到抓包文件的分析里吧。</p><h3>先写过滤器</h3><p>虽然在上节课，我们也使用Wireshark对握手相关的案例做了不少分析，但对它的使用还是相对简单的。那今天这节课开始，我们就要深度使用Wireshark了。比如在接下来的内容里，我会用到很多Wireshark的过滤器（也可以叫过滤表达式或者过滤条件）。因为步骤稍多，所以我会多花一些时间来讲解，希望能给你讲透。</p><p>一般来说，在我们抓到的原始抓包文件里，我们真正关心的报文只占整体的一小部分。那么，如何从中定位跟问题相关的报文，就是个学问了。</p><p>就当前这个案例而言，我们既然有应用层日志，也有相关的IP地址等明确的信息，这些就为我们做报文过滤创造了条件。我们要写一个过滤器，这个过滤器以IP为条件，先从原始文件中过滤出<strong>跟这个IP相关的报文</strong>。</p><p>在Wireshark中，以IP为条件的常用过滤器语法，主要有以下几种：</p><pre><code class="language-plain">ip.addr eq my_ip：过滤出源IP或者目的IP为my_ip的报文
ip.src eq my_ip：过滤出源IP为my_ip的报文
ip.dst eq my_ip：过滤出目的IP为my_ip的报文
</code></pre><p>不过，这还只是第一个过滤条件，仅通过它过滤的话，出来的报文数量仍然比我们真正关心的报文要多很多。我们还需要第二个过滤条件，也就是要<strong>找到TCP RST报文</strong>。这就要用到另外一类过滤器了，也就是tcp.flags，而这里的flags，就是SYN、ACK、FIN、PSH、RST等TCP标志位。</p><p>对于RST报文，过滤条件就是：</p><pre><code class="language-plain">tcp.flags.reset eq 1
</code></pre><p>我们可以选中任意一个报文，注意其TCP的Flags部分：</p><p><img src="https://static001.geekbang.org/resource/image/82/76/8297712dde99ecaef1b27db608561176.jpg?wh=505x214" alt=""></p><p>好了，我们打开抓包文件，输入这个过滤条件：</p><pre><code class="language-plain">ip.addr eq 10.255.252.31 and tcp.flags.reset eq 1
</code></pre><p>我们会发现有很多RST报文：</p><p><img src="https://static001.geekbang.org/resource/image/69/2c/69196eb1f280bfed8c8997e869899c2c.jpg?wh=781x522" alt=""></p><p>在Wirershark窗口的右下角，就有符合过滤条件的报文个数，这里有9122个，占所有报文的4%，确实是非常多。由此推测，日志里的很多报错估计应该就是其中一些RST引起的。我们选一个先看一下。</p><p>我们在<a href="https://time.geekbang.org/column/article/478189">第2讲</a>的时候，就学习了如何在Wireshark中，基于一个报文，找到它所在的整个TCP流的所有其他报文。这里呢，我们选择172号报文，右单击，选中Follow -&gt; TCP Stream，就找到了它所属的整个TCP流的报文：</p><p><img src="https://static001.geekbang.org/resource/image/35/01/35bf3f90770f808515a112ec5694eb01.jpg?wh=858x101" alt=""></p><p>咦，这个RST处在握手阶段？由于这个RST是握手阶段里的第三个报文，但它又不是期望的ACK，而是RST+ACK，所以握手失败了。</p><p>不过，你也许会问：<strong>这种握手阶段的RST，会不会也跟Nginx日志里的connection reset by peer有关系呢？</strong></p><p>要回答这个问题，我们就要先了解应用程序是怎么跟内核的TCP协议栈交互的。一般来说，客户端发起连接，依次调用的是这几个系统调用：</p><ul>
<li>socket()</li>
<li>connect()</li>
</ul><p>而服务端监听端口并提供服务，那么要依次调用的就是以下几个系统调用：</p><ul>
<li>socket()</li>
<li>bind()</li>
<li>listen()</li>
<li>accept()</li>
</ul><p>服务端的用户空间程序要使用TCP连接来接收请求，首先要获得上面最后一个接口，也就是<strong>accept()</strong>调用的返回。而accept()调用能成功返回的前提呢，是正常完成三次握手。</p><p>你看，这次客户端在握手中的第三个包不是ACK，而是RST（或者RST+ACK），握手不是失败了吗？那么自然地，这次失败的握手，也不会转化为一次有效的连接了，所以<strong>Nginx都不知道还存在过这么一次失败的握手</strong>。</p><p>当然，在客户端日志里，是可以记录到这次握手失败的。这是因为，客户端是TCP连接的发起方，它调用connect()，而connect()失败的话，其ECONNRESET返回码，还是可以通知给应用程序的。</p><p>我们再来看一下这张系统调用跟TCP状态关系的示意图：</p><p><img src="https://static001.geekbang.org/resource/image/36/97/3623f2877b7464yy630yy5b14b214697.jpg?wh=1268x456" alt=""></p><p>所以，上面这个虽然也是RST，但并不是我们要找的那种<strong>“在连接建立后发生的RST”</strong>。</p><h3>继续打磨过滤器</h3><p>看来，我们还需要进一步打磨一下过滤条件，把握手阶段的RST给排除。要做到这一点，首先要搞清楚：<strong>什么是握手阶段的RST的特征呢？</strong></p><p>我们关注一下上面的截图，其实会发现：这个RST的序列号是1，确认号也是1。因此，我们可以在原先的过滤条件后面，再加上这个条件：</p><pre><code class="language-plain">tcp.seq eq 1 and tcp.ack eq 1
</code></pre><p>于是过滤条件变成：</p><pre><code class="language-plain">ip.addr eq 10.255.252.31 and tcp.flags.reset eq 1 and !(tcp.seq eq 1 and tcp.ack eq 1)
</code></pre><p>注意，这里的<code>(tcp.seq eq 1 and tcp.ack eq 1)</code>前面是一个感叹号（用not也一样），起到“取反”的作用，也就是排除这类报文。</p><p>让我们看下，现在过滤出来的报文是怎样的：</p><p><img src="https://static001.geekbang.org/resource/image/52/b2/52881809f2d090e658314945f8de83b2.jpg?wh=933x455" alt=""></p><p>我们又发现了序列号为2的很多RST报文，这些又是什么呢？我们选包号115，然后Follow -&gt; TCP Stream看一下：</p><p><img src="https://static001.geekbang.org/resource/image/e2/b4/e2ebcf2e81c8cb6759b254676f69a7b4.jpg?wh=934x78" alt=""></p><p>原来这是挥手阶段的RST，并且没有抓取到数据交互阶段，那跟日志里的报错也没关系，也可以排除。这样的话，我们可以把前面的过滤条件中的and改成or，就可以同时排除握手阶段和挥手阶段的RST报文了。我们输入过滤器：</p><pre><code>ip.addr eq 10.255.252.31 and tcp.flags.reset eq 1 and !(tcp.seq eq 1 or tcp.ack eq 1)
</code></pre><p>得到下面这些报文：</p><p><img src="https://static001.geekbang.org/resource/image/bd/65/bdf63b498d1b258879ef977a1bfd5265.jpg?wh=856x333" alt=""></p><p>虽然排除了握手阶段的RST报文，但是剩下的也还是太多，我们要找的“造成Nginx日志报错”的RST在哪里呢？</p><p>为了找到它们，我们需要再增加一些明确的搜索条件。还记得我提到过的两大鸿沟吗？一个是应用现象跟网络现象之间的鸿沟，一个是工具提示跟协议理解之间的鸿沟。</p><p>现在为了跨越第一个鸿沟，我们需要把搜索条件落实具体，针对当前案例来说，就是基于以下条件寻找数据包：</p><ul>
<li>既然这些网络报文跟应用层的事务有直接关系，那么报文中应该就包含了请求相关的数据，比如字符串、数值等。当然，这个前提是数据本身没有做过特定的编码，否则的话，报文中的二进制数据，跟应用层解码后看到的数据就会完全不同。</li>
</ul><blockquote>
<p>补充：编码的最典型的场景就是TLS。如果我们不做解密，那么直接tcpdump或者Wireshark抓取到的报文就是加密过的，跟应用层（比如HTTP）的数据完全不同，这也给排查工作带来了不小的困难。关于如何对TLS抓包数据进行解密，我在“实战二”的TLS排查的课程里会提到，你可以期待一下。</p>
</blockquote><ul>
<li>这些报文的发送时间，应该跟日志的时间是吻合的。</li>
</ul><p>对于条件1，我们可以利用Nginx日志中的URL等信息；对于条件2，我们就要利用日志的时间。其实，在开头部分展示的Nginx日志中，就有明确的时间（2015/12/01 15:49:48），虽然只是精确到秒，但很多时候已经足以帮助我们进一步缩小范围了。</p><p>那么，在Wireshark中搜索“特定时间段内的报文”，又要如何做到呢？这就是我要介绍给你的又一个<strong>搜索技巧：使用frame.time过滤器</strong>。比如下面这样：</p><pre><code class="language-plain">frame.time &gt;="dec 01, 2015 15:49:48" and frame.time &lt;="dec 01, 2015 15:49:49"
</code></pre><p>这就可以帮助我们定位到跟上面Nginx日志中，第一条日志的时间匹配的报文了。为了方便你理解，我直接把这条日志复制到这里给你参考：</p><pre><code class="language-plain">2015/12/01 15:49:48 [info] 20521#0: *55077498 recv() failed (104: Connection reset by peer) while sending to client, client: 10.255.252.31, server: manager.example.com, request: "POST /WebPageAlipay/weixin/notify_url.htm HTTP/1.1", upstream: "http:/10.4.36.207:8080/WebPageAlipay/weixin/notify_url.htm", host: "manager.example.com"
</code></pre><p>我们再结合前面的搜索条件，就得到了下面这个更加精确的过滤条件：</p><pre><code class="language-plain">frame.time &gt;="dec 01, 2015 15:49:48" and frame.time &lt;="dec 01, 2015 15:49:49" and ip.addr eq 10.255.252.31 and tcp.flags.reset eq 1 and !(tcp.seq eq 1 or tcp.ack eq 1)
</code></pre><p>好长的一个过滤器！不过没关系，人读着觉得长，Wireshark就未必这么觉得了，也许还觉得很顺眼呢。就好比机器语言，人读着感觉是天书，机器却觉得好亲近，“这可是我的母语啊！”</p><p>好，这次我们终于非常成功地锁定到只有3个RST报文了：</p><p><img src="https://static001.geekbang.org/resource/image/2b/f0/2b370fc667acf40c88d2502536fcf3f0.jpg?wh=1088x108" alt=""></p><p>接下来要做的事情就会简单很多：只要把这三个RST所在的TCP流里的应用层数据（也就是HTTP请求和返回）跟Nginx日志中的请求和返回进行对比，就能找到是哪个RST引起了Nginx报错了。</p><h3>对问题报文的深入分析</h3><p>我们先来看看，11393号报文所属的流是什么情况？</p><p><img src="https://static001.geekbang.org/resource/image/04/ce/043f9bc2bcc71eda60b891fedb7285ce.jpg?wh=1028x211" alt=""></p><p>然后我们来看一下11448号报文所属的TCP流。</p><p><img src="https://static001.geekbang.org/resource/image/4a/8b/4aaef0f94ccd160269c3a87fe1bed78b.jpg?wh=1027x262" alt=""></p><p>原来，11448跟11450是在同一个流里面的。现在清楚了，3个RST，分别属于2个HTTP事务。</p><p>你再仔细对比一下两个图中的红框部分，是不是不一样？它们分别是对应了一个URL里带“weixin”字符串的请求，和一个URL里带“app”字符串的请求。那么，在这个时间点（15:49:48）对应的日志是关于哪一个URL的呢？</p><pre><code class="language-plain">2015/12/01 15:49:48 [info] 20521#0: *55077498 recv() failed (104: Connection reset by peer) while sending to client, client: 10.255.252.31, server: manager.example.com, request: "POST /WebPageAlipay/weixin/notify_url.htm HTTP/1.1", upstream: "http:/10.4.36.207:8080/WebPageAlipay/weixin/notify_url.htm", host: "manager.example.com"
</code></pre><p>你只要往右拖动一下鼠标，就能看到POST URL里的“weixin”字符串了。而包号11448和11450这两个RST所在的TCP流的请求，也是带“weixin”字符串的，所以它们就是匹配上面这条日志的RST！</p><p>如果你还没有完全理解，我这里帮你小结一下，为什么我们可以确定这个TCP流就是对应这条日志的，主要三点原因：</p><ul>
<li>时间吻合；</li>
<li>RST行为吻合；</li>
<li>URL路径吻合。</li>
</ul><p>通过解读上面的TCP流，我们终于跨过了这道“应用现象跟网络报文”之间的鸿沟：</p><p><img src="https://static001.geekbang.org/resource/image/30/e2/30d7677724c5f6ffc8635f2ab9fa0fe2.jpg?wh=2000x562" alt=""></p><p>再进一步，我给你画一下这个HTTP事务的整体过程，帮你进一步搞清楚为什么这个RST，会引起Nginx记录connection reset by peer的报错：</p><p><img src="https://static001.geekbang.org/resource/image/7c/2c/7c193479ac85275f411a3de778a1ae2c.jpg?wh=2000x1125" alt=""></p><p>也就是说，握手和HTTP POST请求和响应都正常，但是客户端在对HTTP 200这个响应做了ACK后，随即发送了RST+ACK，而正是这个行为<strong>破坏了正常的TCP四次挥手</strong>。也正是这个RST，导致服务端Nginx的recv()调用收到了ECONNRESET报错，从而进入了Nginx日志，成为一条connection reset by peer。</p><p><strong>这个对应用产生了什么影响呢？</strong>对于服务端来说，表面上至少是记录了一次报错日志。但是有意思的是，这个POST还是成功了，已经被正常处理完了，要不然Nginx也不会回复HTTP 200。</p><p>对于客户端呢？还不好说，因为我们并没有客户端的日志，也不排除客户端认为这次是失败，可能会有重试等等。</p><p>我们把这个结论告诉给了客户，他们悬着的心稍稍放下了：至少POST的数据都被服务端处理了。当然，他们还需要查找客户端代码的问题，把这个不正常的RST行为给修复掉，但是至少已经不用担心数据是否完整、事务是否正常了。</p><p>现在，回到我们开头的三连问：</p><ul>
<li>这个reset会影响我们的业务吗，这次事务到底有没有成功呢?</li>
<li>这个reset发生在具体什么阶段，属于TCP的正常断连吗？</li>
<li>我们要怎么做才能避免这种reset呢？</li>
</ul><p>我们现在就可以回答了：</p><ul>
<li>这个reset是否影响业务，还要继续查客户端应用，但服务端事务是成功被处理了。</li>
<li>这个reset发生在事务处理完成后，但不属于TCP正常断连，还需要继续查客户端代码问题。</li>
<li>要避免这种reset，需要客户端代码进行修复。</li>
</ul><blockquote>
<p>补充：客户端用RST来断开连接并不妥当，需要从代码上找原因。比如客户端在Receive Buffer里还有数据未被读取的情况下，就调用了close()。对应用的影响究竟如何，就要看具体的应用逻辑了。</p>
</blockquote><p>网络中的环节很多，包括客户端、服务端、中间路由交换设备、防火墙、LB或者反向代理等等。如何在这么多环节中定位到具体的问题节点，一直以来是很多工程师的痛点。比如，网络不稳定，或者防火墙来几个RST，也都有可能导致类似的connection reset by peer的问题。</p><p>通过抓包分析，我们抽丝剥茧，定位到具体的问题环节不在Nginx，也不在网络本身，而是在客户端代码这里。也正因为有了这样的分析，写代码的同学就可以专心做代码修复，而不用一直怀疑问题在其他环节了。</p><p>好，讨论完RST，你可能会问了：TCP挥手一般是用FIN的，这个知识点还没讨论呢。别急，这第二个案例就是关于<strong>FIN</strong>的。</p><h2>案例2：一个FIN就完成了TCP挥手？</h2><p>你应该知道，TCP挥手是“四次”，这几乎也是老生常谈的知识点了。不过这里，我也再带你来看一下常规的四次挥手的过程：</p><p><img src="https://static001.geekbang.org/resource/image/82/71/82d23321527b3530d40c67yy37670f71.jpg?wh=2000x908" alt=""></p><p>我在图上没有用“客户端”和“服务端”这种名称，而是叫“发起端”和“接收端”。这是因为，<strong>TCP的挥手是任意一端都可以主动发起的</strong>。也就是说，挥手的发起权并不固定给客户端或者服务端。这跟TCP握手不同：握手是客户端发起的。或者换个说法：<strong>发起握手的就是客户端</strong>。在握手阶段，角色分工十分明确。</p><p>另外，FIN和ACK都各有两次，这也是十分明确的。</p><p>可是有一次，一个客户向我报告这么一个奇怪的现象：他们偶然发现，他们的应用在TCP关闭阶段，只有一个FIN，而不是两个FIN。这好像不符合常理啊。我也觉得有意思，就一起看了他们这个抓包文件：</p><p><img src="https://static001.geekbang.org/resource/image/cd/76/cd8e66de72bab07244442ae290940f76.jpg?wh=1034x173" alt=""></p><p>确实奇怪，真的只有一个FIN。这两端的操作系统竟然能容忍这种事情发生？瞬间感觉“塌房”了：难道一向严谨的TCP，它的分手也可以这么随意吗？“当初是你要分开，分开就分开，一个FIN，就足够，眼泪落下来”？</p><p>玩笑归玩笑，很快，我就意识到还有一种可能性。在上节课我介绍TCP握手的时候提到过，TCP里一个报文可以搭另一个报文的顺风车（Piggybacking），以提高TCP传输的运载效率。所以，TCP挥手倒不是一定要四个报文，Piggybacking后，就可能是3个报文了。看起来就类似三次挥手：</p><p><img src="https://static001.geekbang.org/resource/image/be/e1/be46316edd5128c572a991b76fe9dbe1.jpg?wh=2000x639" alt=""></p><p>那这次的案例，我们在Wireshark中看到了后两个报文，即接收端回复的FIN+ACK和发起端的最后一个ACK。<strong>那么，第一个FIN在哪里呢？</strong>从Wireshark的截图中，确实看不出来。</p><p>当然，从Wireshark的图里，我们甚至可以认为，这次连接是服务端发起的，它发送了FIN+ACK，而客户端只回复了一个ACK，这条连接就结束了。这样的解读更加诡异，却也符合Wireshark的展示。</p><p><img src="https://static001.geekbang.org/resource/image/74/75/741c44e6e20b7d75a316f7b3a0cf8175.jpg?wh=2008x384" alt=""><br>
<img src="https://static001.geekbang.org/resource/image/bb/f8/bbfdfc004bd5aa7a0e8d6db1979bcaf8.jpg?wh=2000x746" alt=""></p><p>但是，Wireshark的主界面还有个特点，就是当它的<strong>Information列展示的是应用层信息时，这个报文的TCP层面的控制信息就不显示了</strong>。</p><p>所以，上面的POST请求报文，其Information列就是POST方法加上具体的URL。它的TCP信息，包括序列号、确认号、标志位等，都需要到详情里面去找。</p><p>我们先选中这个POST报文，然后到界面中间的TCP详情部分去看看：</p><p><img src="https://static001.geekbang.org/resource/image/fc/b9/fc400e5bc462d752846a921938cae1b9.jpg?wh=999x702" alt=""></p><p>原来，第一个FIN控制报文，并没有像常规的那样单独出现，而是<strong>合并（Piggybacking）在POST报文里</strong>！所以，整个挥手过程，其实依然十分标准，完全遵循了协议规范。仅仅是因为Wireshark的显示问题，带来了一场小小的误会。虽然还有一个“为什么没有HTTP响应报文”的问题，但是TCP挥手方面的问题，已经得到了合理的解释了。</p><p><img src="https://static001.geekbang.org/resource/image/23/2c/2359b320a409263092b5b9ayybd7802c.jpg?wh=2000x864" alt=""></p><p>这也提醒我们，理解TCP知识点的时候需要真正理解，而不是生搬硬套。这一方面需要对协议的仔细研读，另一方面也离不开实际案例的积累和融会贯通，从量变引起质变。</p><p>我们自己也要有个态度：大部分时候，当看到TCP有什么好像“不合规的行为”，<strong>我们最好先反思自己是不是对TCP的掌握还不够深入，而不是先去怀疑TCP</strong>，毕竟它也久经考验，它正确的概率比我们高得多，那我们做“自我检讨”，其实是笔划算的买卖，基本“稳赢”。</p><h2>小结</h2><p>在这节课里，我们通过回顾案例，把TCP挥手的相关技术细节给梳理了一遍。在案例1里面，我们用抓包分析的方法，打通了“应用症状跟网络现象”以及“工具提示与协议理解”这两大鸿沟，你可以再重点关注一下这里面用到的推进技巧：</p><ul>
<li>首先根据应用层的表象信息，抽取出IP和RST报文这两个过滤条件，启动了报文过滤的工作。</li>
<li>分析第一遍的过滤结果，得到进一步推进的过滤条件（在这个案例里是排除握手阶段的RST）。</li>
<li>结合日志时间范围，继续缩小范围到3个RST报文，这个范围足够小，我们可以展开分析，最终找到报错相关的TCP流。这种“迭代式”的过滤可以反复好几轮，直到你定位到问题报文。</li>
<li>在这个TCP流里，结合对TCP协议和HTTP的理解，定位到问题所在。</li>
</ul><p>此外，通过这个案例，我也给你介绍了一些Wireshark的使用技巧，特别是各种过滤器：</p><ul>
<li>通过<strong>ip.addr eq my_ip</strong>或<strong>ip.src eq my_ip</strong>，再或者<strong>ip.dst eq my_ip</strong>，可以找到跟my_ip相关的报文。</li>
<li>通过<strong>tcp.flags.reset eq 1</strong>可以找到RST报文，其他TCP标志位，依此类推。</li>
<li>通过<strong>tcp.ack eq my_num</strong>可以找到确认号为my_num的报文，对序列号的搜索，同理可用<strong>tcp.seq eq my_num</strong>。</li>
<li>一个过滤表达式之前加上“<strong>!</strong>”或者<strong>not</strong>起到取反的作用，也就是排除掉这些报文。</li>
<li>通过frame.time &gt;="dec 01, 2015 15:49:48"这种形式的过滤器，我们可以根据时间来过滤报文。</li>
<li>多个过滤条件之间可以用and或者or来形成复合过滤器。</li>
<li>通过把<strong>应用日志中的信息</strong>（比如URL路径等）和Wireshark里的<strong>TCP载荷的信息</strong>进行对比，可以帮助我们定位到跟这个日志相关的网络报文。</li>
</ul><p>而在案例2里面，我们对“四次挥手”又有了新的认识。通过这个真实案例，我希望你能够了解到：</p><ul>
<li>实际上TCP挥手可能不是表面上的四次报文，因为并包也就是Piggybacking的存在，它可能<strong>看起来是三次</strong>。</li>
<li>在某些特殊情况下，在Wireshark里看不到第一个FIN。这个时候你不要真的把后面那个被Wireshark直接展示的FIN当作是第一个FIN。你需要选中挥手阶段附近的报文，<strong>在TCP详情里面查看是否有报文携带了FIN标志位</strong>。这确实是个非常容易掉坑的地方，所以我要提醒你一下。</li>
</ul><h2>思考题</h2><p>好了，挥手相关的知识点给你复习到这里。给你留几道思考题：</p><ol>
<li>如果要在Wireshark中搜索到挥手阶段出现的RST+ACK报文，那么这个过滤器该如何写呢？</li>
<li>你有没有通过抓包分析，解决过应用层的奇怪问题呢？你是怎么做的呢？</li>
</ol><p>欢迎在留言区分享你的答案和思考，我们一起交流探讨。如果觉得有收获，也欢迎你把今天的内容分享给更多的朋友。</p><h2>扩展知识：聊一聊挥手的常见误区</h2><p>我们案例也讲了两个了，相信你也对非正常挥手（RST）和正常挥手（FIN）有了更加深入的认识了。接下来，我再给你介绍几个常见误区，希望给你起到“有则改之，无则加勉”的效果。</p><h3>连接关闭由客户端发起？</h3><p>其实不对，连接关闭可以是客户端，也可以是服务端发起。造成这个误解的原因，其实也跟这张图有关系：</p><p><img src="https://static001.geekbang.org/resource/image/a2/83/a2d8d9a106760e8a0de2d880af0bd883.jpg?wh=1370x1184" alt=""></p><p>你有没有发现，图中第一个FIN是从客户端发起的。但服务端就不会主动发起关闭/挥手吗？当然会，只是图中没有标明这种情况。挥手跟握手不同，握手一定是客户端发起的（所以才叫客户端），但挥手是双方都可以。</p><p>其实上节课我们也讲到过这张图，它出自Richard Stevens的<a href="https://book.douban.com/subject/4859464">《UNIX网络编程：套接字联网API》</a>。那是不是Stevens自己就搞错了呢？我觉得，这个可能性比我中彩票的概率还要低好几个数量级。</p><p>Stevens当然清楚双方都可以发起挥手，他只是为了突出重点，就没有把多种情况都画到同一张图里，因为这张图的重点是<strong>把TCP连接状态的变迁展示清楚</strong>，而不是要突出“谁可以发起挥手”这个细节。</p><h3>挥手不能同时发起？</h3><p>有的同学觉得挥手是客户端发起的，或者是服务端发起，反正就不能是双方同时发起。事实上，如果双方同时都主动发起了关闭，TCP会怎么处理这种情况呢？我们看下图：</p><p><img src="https://static001.geekbang.org/resource/image/c2/c4/c22911ba09ca2300f1377387b4a41cc4.jpg?wh=2000x757" alt=""></p><ul>
<li>双方同时发起关闭后，也同时进入了FIN_WAIT_1状态；</li>
<li>然后也因为收到了对方的FIN，也都进入了CLOSING状态；</li>
<li>当双方都收到对方的ACK后，最终都进入了TIME_WAIT状态。</li>
</ul><p>这也意味着，两端都需要等待2MSL的时间，才能复用这个五元组TCP连接。这种情况是比较少见的，但是协议设计需要考虑各种边界条件下的实现，比普通的应用程序所要考虑的事情要多不少。所以也许<strong>有些RFC看似简单，但背后其实都十分不简单</strong>。</p><h3>TCP挥手时双方同时停止发送数据？</h3><p>一方发送FIN，表示这个连接开始关闭了，双方就都不会发送新的数据了？这也是很常见的误区。</p><p>实际上，一方发送FIN只是表示这一方不再发送新的数据，但对方仍可以发送数据。</p><p>还是在Richard Stevens的<a href="https://book.douban.com/subject/1088054">《TCP/IP详解（第一卷）》</a>中，明确提到TCP可以有“半关闭”的做法，也就是：</p><ul>
<li>一端（A）发送FIN，表示“我要关闭，不再发送新的数据了，但我可以接收新的数据”。</li>
<li>另一端（B）可以回复ACK，表示“我知道你那头不会再发送了，我这头未必哦”。</li>
<li>B可以继续发送新的数据给A，A也会回复ACK表示确认收到新数据。</li>
<li>在发送完这些新数据后，B才启动了自己的关闭过程，也就是发送FIN给A，表示“我的事情终于忙好了，我也要关闭，不会再发送新数据了”。</li>
<li>这时候才是真正的两端都关闭了连接。</li>
</ul><p>还是搬运了Stevens的图过来供你参考，也再次致敬Stevens大师！</p><p><img src="https://static001.geekbang.org/resource/image/12/2c/128cea235e031fd471b501f7532a372c.jpg?wh=1190x876" alt=""></p>
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
  <div class="_2_QraFYR_0">tcpdump立奇功，《如何定位上亿次调用才出现一次的Bug》https:&#47;&#47;plantegg.github.io&#47;2018&#47;04&#47;26&#47;%E5%A6%82%E4%BD%95%E5%AE%9A%E4%BD%8D%E4%B8%8A%E4%BA%BF%E6%AC%A1%E8%B0%83%E7%94%A8%E6%89%8D%E5%87%BA%E7%8E%B0%E4%B8%80%E6%AC%A1%E7%9A%84Bug&#47; 扯皮的时候没有人敢否认tcpdump数据</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-20 18:13:52</div>
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
  <div class="_2_QraFYR_0">请问老师，客户端用 RST 来断开连接并不妥当，需要从代码上找原因。比如客户端在 Receive Buffer 里还有数据未被读取的情况下，就调用了 close()。调用close，是不是应该发fin包呢？什么时候会发rst包？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这是内核的行为。close()系统调用会走到tcp_close()，在这个函数里，会做判断，如果buffer里还有数据未读，它就直接调用tcp_send_active_reset()，发出RST，并把这个连接直接设置到CLOSED状态，也就是不进入TIME_WAIT。<br>对于这种情况，为了避免close()时候发出RST，就需要检查业务代码，确保在调用close()之前，把接收缓冲里的数据读取掉。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-19 11:01:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2c/45/c6/28dfdbc9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>*</span>
  </div>
  <div class="_2_QraFYR_0">老师，tcp.ack和tcp.flags.ack有什么不同，为什么问题1的答案是tcp.ack==1 and tcp.flags.reset == 1</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 答案是tcp.flags.ack == 1 and tcp.flags.reset == 1<br>tcp.ack是指报文的确认号，是一个数字<br>tcp.flags.ack和tcp.flags.reset都是报文的TCP标志位,不是0就是1</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-27 14:40:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/f5/d1/cc6f82eb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>kaixiao7</span>
  </div>
  <div class="_2_QraFYR_0">老师，在客户端握手的第三个RST+ACK报文中，为什么会出现RST呢？有哪些情况会出现呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你好，这个要看具体的场景逻辑，有一种场景就是握手的第三个报文是RST+ACK的，那就是健康检查，也就是对服务端口的可达性的检查。发起端发送SYN，服务端回复SYN+ACK，发起端此时知道这个服务端口是可用的（健康的），此时探测任务已经完成了。而用RST+ACK的话可以明确的让服务端关闭这个连接请求（释放资源），在发起端（客户端）也是可以回收资源，对双方都有利。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-03 14:41:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/37/a0/032d0828.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>上杉夏香</span>
  </div>
  <div class="_2_QraFYR_0">还真遇到过。<br>问题背景：搭建用标签体系，使用redis的扩展redis-search&#47;redis-json。<br><br>出错场景：标签建立之后，使用redis-search进行搜索。我用的是go语言，实现搜索，使用官方的golang client。在关于数值的开&#47;闭区间查找时，直接报错。<br>抓包发现，是这个golang client将redis命令解析失败，不符合redis的通信协议RESP，redis服务器接收请求，直接报错。<br><br>这个属于golang client实现的问题，提了人生第一个PR。<br><br>pr地址<br>https:&#47;&#47;github.com&#47;RediSearch&#47;redisearch-go&#47;pull&#47;139</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-16 17:36:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/57/4f/6fb51ff1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>一步</span>
  </div>
  <div class="_2_QraFYR_0">最后一个图 有个疑问的， 为什么客户端在初始化关闭的时候 除了发送一个 FIN ，还有个 ACK呢，是为了回复上一个服务端发送的数据吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，就是确认它收到的上一个报文。在TCP里面，基本上除了握手的第一个报文SYN，其他报文都带ACK标志位</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-25 22:32:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/BQaJmDRTUXyNIzH6SwM27kXQibyDUlJ3vibiaIUmRe9j1I4K5fLDnOf6Uicibj2gBsSeWA4zKoUMN803oFD4gAHuiblA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lxj</span>
  </div>
  <div class="_2_QraFYR_0">有一点点小提议，分析一个问题能把如何抓包，在哪端抓包，抓包时间点，过滤条件再交代详细点嘛，感觉分析的粒度太粗了，上下文没有交代清楚</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你好，这次的抓包细节是这样的：<br>“所以，我们就展开了抓包工作。具体做法是：我们需要选择一端做抓包，这次是客户端；检查应用日志，发现没几分钟就出现了 connection reset by peer 的报错；对照报错日志和抓包文件，寻找线索。”<br><br>对应你的几个问题：<br>如何抓包：用tcpdump<br>在哪端抓包：客户端<br>抓包时间点：这个没有特别的讲究，因为问题一直在发生，随时可以抓<br>过滤条件： tcpdump host 服务端IP<br>抓了几分钟就发现日志里有我们找的报错，此时就可以把抓包停止了。因为很容易复现，所以抓包时间不长，只有3分25秒。具体的抓包示例文件我很快上传：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-04 12:41:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/c1/23/fa377732.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>乌龟爱上金鱼</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，对端向我发了fin，我这边ack后没有close，进入closewait，对端现在应该是finwait2，按理来说对端现在只是不向我发送数据了，但是能接受数据，但是我继续向对端发送数据后，对端回复的是reset，这是为什么呀？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 您好，这确实是一个有意思的话题。TCP确实可以做“半关闭”行为，但是仅根据对端发出的一个FIN报文是无法确切的判定对端是全关闭（同时关闭读和写）还是半关闭（只关闭写）的。但是对端在FIN报文后的行为倒是可以帮助我们做进一步的判断。<br><br>比如你遇到的情况是，收到FIN后，你继续发送数据，对端回复了RST。这大概率说明对端是同时关闭了读和写了。也就是说，对端发出FIN的同时，它这个socket已经既不能写入数据（也就是发给你数据），也不能接受数据（也就是接收你在收到FIN后发送的更多的数据报文）了。由于此时对端这个socket处于这种不可读写的状态，所以当对端的内核收到了你的数据报文时，它就用RST做了回复。<br><br>当然，如果对端的FIN是由shutdown(sockfd, SHUT_WR)触发的，那就只是关闭写的方向，也会同样发送FIN给你，而此时你继续发送数据报文，对端还是有可能会接收的。总之还是看具体的代码场景了~<br><br>所以我推测，对端当时是用shutdown(sockfd, SHUT_RDWR)或者close(sockfd)来关闭这个连接的。供你参考。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-19 23:17:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/37/a0/032d0828.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>上杉夏香</span>
  </div>
  <div class="_2_QraFYR_0">这篇文章的最后一张图，是不是有些瑕疵？进入半连接状态后，服务端继续发送数据，当服务器发送FIN报文的时候，它的seq应该不再是L？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个图是Stevens的书中的原图。之前没留意，你这么说，我觉得真是个问题。服务器发送的FIN的seq应该是L加上最后一次data的length。你好仔细！~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-27 10:34:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/22/73/76/a08f9af2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>sysho</span>
  </div>
  <div class="_2_QraFYR_0">这两个月刚好排查了几个抓包问题，觉得老师这节课讲得真好，看得很爽。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，抓包分析入门后会发现越来越多的乐趣：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-20 22:16:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/69/21/7db9faf1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>简迷离</span>
  </div>
  <div class="_2_QraFYR_0">老师，案例中的抓包文件能提供下吗？这样单独的看文章分析效果还是有限，能提供下案例抓包文件边看边分析数据包效果更佳，还请老师提供下，谢谢您！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 您好，后面从第6讲开始大部分都有抓包示例文件了，供你参考~<br>这一讲的文件我整理一下：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-27 18:30:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83eoSffTqghG7YNWv5b4mEibbQh0ARr5Ar5U9qIE2BSCOjlmbfflhLDjQKsok9TvhCcV4PrVc1SsAD9w/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>仲晶晶</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，我这边做openresty测试的时候遇到个现象：<br>jemter通过F5压测到openresty网关的时候（TPS 5k-6k）左右，这个时候用的都是短链接，我就看到很多TW，这说明断链都是在网关这边断链的，我比较疑惑什么情况下服务端（网关）会发生断链呢（发FIN）？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: openrestry我了解不多，不过从协议来看，你在openrestry网关上看到很多TW，那就意味着网关主动发起了关闭，你的这个判断是正确的。<br>你的测试场景是不是这样：Jmeter -&gt; F5 -&gt; openrestry<br>你这里提到“用的都是短连接”，应该是指JMeter或者F5做了这个配置对吧，那么按照http协议规范，如果客户端（这里是Jmeter或者F5）发送的http请求里面携带Connection: close头部，web server（这里是openrestry)有可能会主动发起关闭。我这里说“有可能”是因为我也注意到有些web server并不会那么做，也许是考虑到了主动关闭会引发自己进入TW状态，而这种状态的连接越多，对web server自己越不利。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-27 21:45:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/98/b1/f89a84d0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Tokamak</span>
  </div>
  <div class="_2_QraFYR_0">老师请问一下，TIME_WAIT 需要等待2个MSL, 如果不等待2个MSL，此时直接使用该端口连接服务器，在建立连接的过程中，如果收到了服务端重发的FIN+ACK,此时客户端会做什么处理呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不等待2个MSL，又直接使用该端口？你是指操作系统此时也已经丢弃这个连接了吗？此时客户端发送SYN后收到了服务端迟到的FIN+ACK，会回复RST给服务端。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-20 13:25:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/23/52/66/3e4d4846.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>includestdio.h</span>
  </div>
  <div class="_2_QraFYR_0">1.不太了解挥手阶段的RST特征，不确定对不对。tcp.flags.reset eq 1 and tcp.fin eq 1，感觉差点啥<br><br>2.实际工作中对抓包一知半解，基本只会通过RST分析问题大致在哪端，后续就不会了，希望通过专栏能得到提升</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. 用RST的方式断开连接，用意是表示异常，至少让对端知道：我这儿多少有点问题，或者我觉得你这次连接中的表现有点问题，所以我用RST来关闭连接。<br>而用FIN关闭就意味着：我认为这次咱们通话顺利正常，我准备用安全标准的方式来关闭连接。<br><br>2. 抓包分析除了网络技术的学习，更多的是靠实践，在后续的课程里会有更多的实际案例，我会尽量对每个关键步骤做详细的说明，让大家明白怎么做，为什么这么做。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-19 12:21:45</div>
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
  <div class="_2_QraFYR_0">前面提到的挥手时搭便车，指的是第二次和第三次挥手合并在一起吗，总共减少了一次挥手</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯 挥手时候的四个报文是<br>1. 发起端：FIN<br>2. 对端：ACK<br>3. 对端：FIN<br>4. 发起端：ACK<br>其中，2和3经常会在一起发送。除此以外，1和经常和发起端的其他报文一起发送（也就是课程里面介绍的情况）。<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-19 08:48:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/b7/24/17f6c240.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>janey</span>
  </div>
  <div class="_2_QraFYR_0">这里有疑问，上一句说“Nginx 都不知道还存在过这么一次失败的握手”下一句又说“在客户端日志里，是可以记录到这次握手失败的。” 那怎么知道“这个虽然也是 RST，但并不是我们要找的那种“在连接建立后发生的 RST””的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 失败的握手，在客户端和服务端的表现不同：<br>1. 在客户端是直接体现为connect()调用的失败，所以用户空间程序是可以记录下这次失败的。<br>2. 在服务端，握手是内核直接处理的，用户空间程序的accept()接收到的是握手成功后的连接，对于握手过程失败的情况无法感知，也就不能在应用日志里看到这种失败。<br><br>以上说的都是应用程序方面的情况。tcpdump抓包是直接在内核处理之前抓取到报文的，所以当然可以看到所有RST等异常报文。<br><br>总结一下：服务端应用程序不知道握手阶段的RST，但服务端tcpdump可以知道。<br>这样说是否清晰一些？<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-09 09:13:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/b3/e3/a66c8c75.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>邓坤坤</span>
  </div>
  <div class="_2_QraFYR_0">老师，请问应用层如何编码才能在发送数据时带上fin flag？从来没见过这种</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 一般来说，应用层是通过系统调用（像connect, socket, bind, send, close, shutdown等）来跟内核网络做交互的。TCP标志位是无法直接操作的，只能通过系统调用来“间接”的达到目的。比如你要发送FIN flag，那就对socket描述符调用close()。<br>另外就是用raw socket直接构造报文，但未必是你想要的吧<br>如果要直接操纵报文，还有一种办法是用</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-01 10:51:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/47/00/3202bdf0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>piboye</span>
  </div>
  <div class="_2_QraFYR_0">sync 包的 seq 我想的是 ISN 现在已经随机化了,  怎么会是1呢? 仔细看 Wireshark的detail 部分, 才发现wireshark 帮我们计算成相对seq了,  棒</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯 如果要看原始序列号（裸序列号），就可以在wireshark的preference里把relative sequence number选项取消~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-25 16:28:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83eo5vic8QksE4b8ricXxKrEWJyOX9pwiadhk3kvHYoLXoKRTWvbFCxibFTbExNQWDG4nvNfpic9t1umibKww/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>江楠大盗</span>
  </div>
  <div class="_2_QraFYR_0">如果线上请求量很大，而异常关闭可能一个小时才发生一次，为了定位问题，得一直开着tcpdump，那么.pcap文件岂不是非常大，这个怎么处理？还望老师能解答</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 您好，这个话题正好课程里有提及，在《实战二》部分的第18讲，就是关于偶发性问题如何排查的思路和技巧。你也可以去参考那一讲，有问题可以留言，我会解答：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-29 17:53:37</div>
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
  <div class="_2_QraFYR_0">第一个案例 第一次filter 掉的 握手时候客户端发往服务端的RST SEQ=1, ACK=1为什么会出现呢？ 没有办法理解为什么客户端刚发SYN,  收到服务端ACK 后立即RST <br><br>第二次filter 掉的挥手那张图，看上去也是客户端给服务端的443 端口直接先发一个FIN , 然后接着就是 RST SEQ=2, ACK=1,  那这个对应客户端应用代码大概会是什么样呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 第一个问题，从现象来说，这应该是TCP健康检查的报文，也就是客户端收到对端回复SYN+ACK就能知道这个TCP端口是通的（检查成功），然后用RST关闭这个连接。当然，更加友好的方式是用FIN来结束，但用RST结束也是可以的，我们在其他LB场景里也能观察到同样的现象。<br>第二个问题，指的应该是tcp.stream eq 20过滤出来的那两个报文。从现象来看，第一个报文FIN+ACK是关闭自己的发送方向，也就是说“我不会发更多数据了，但还可以接收新的数据”，这个比较符合shutdown(fd, SHUT_WR)；第二个报文是RST+ACK，这个一般是在接受缓冲区还有数据没有读走时调用close()导致的，所以客户端触发这段行为的代码是：<br>&#47;&#47;don&#39;t call read() to fetch the data from receive buffer<br>shutdown(fd, SHUT_WR);  &#47;&#47;trigger FIN+ACK<br>close(fd);  &#47;&#47;trigger RST+ACK</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-12 22:48:30</div>
  </div>
</div>
</div>
</li>
</ul>