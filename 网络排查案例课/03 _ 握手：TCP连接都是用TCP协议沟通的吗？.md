<audio title="03 _ 握手：TCP连接都是用TCP协议沟通的吗？" src="https://static001.geekbang.org/resource/audio/cf/46/cf9442a3b351517b48e7a905085b3146.mp3" controls="controls"></audio> 
<p>你好，我是胜辉。</p><p>在前面预习篇的两节课里，我们一起回顾和学习了网络分层模型与排查工具，也初步学习了一下抓包分析技术。相信现在的你，已经比刚开始的时候多了不少底气了。那么从今天开始，我们就要正式进入TCP这本大部头，而首先要攻破的，就是握手和挥手。</p><p>TCP的三次握手非常有名，我们工作中也时常能用到，所以这块知识的实用性是很强的。更不用说，技术面试里面，无论是什么岗位，似乎只要是技术岗，都可能会问到TCP握手。可见，它跟操作系统基础、编程基础等类似，同属于计算机技术的底座之一。</p><p>握手，说简单也简单，不就是三次握手嘛。说复杂也复杂，别看只是三次握手，中间还是有不少学问的，有些看似复杂的问题，也能用握手的技术来解决。不信你就跟我看这几个案例。</p><h2>TCP连接都是用TCP协议沟通的吗？</h2><p>看到这个小标题，可能你都觉得奇怪了：TCP连接不用TCP协议沟通还用什么呢？</p><p>确实，一般来说TCP连接是标准的TCP三次握手完成的：</p><ol>
<li>客户端发送SYN；</li>
<li>服务端收到SYN后，回复SYN+ACK；</li>
<li>客户端收到SYN+ACK后，回复ACK。</li>
</ol><p>这里面SYN会在两端各发送一次，表示“我准备好了，可以开始连接了”。ACK也是两端各发送了一次，表示“我知道你准备好了，我们开始通信吧”。</p><!-- [[[read_end]]] --><p>那既然是4个报文，为什么是三次发送呢？显然，服务端的SYN和ACK是合并在一起发送的，就节省了一次发送。这个在英文里叫Piggybacking，就是背着走，搭顺风车的意思。</p><p>如果服务端不想接受这次握手，它会怎么做呢？可能会出现这么几种情况：</p><ol>
<li>不搭理这次连接，就当什么都没收到，什么都没发生。这种行为，也可以说是“装聋作哑”。</li>
<li>给予回复，明确拒绝。相当于有人伸手过来想握手，你一巴掌拍掉，真的是非常刚了。</li>
</ol><p>第一种情况，因为服务端做了“<strong>静默丢包</strong>”，也就是虽然收到了SYN，但是它直接丢弃了，也不给客户端回复任何消息。这也导致了一个问题，就是客户端无法分清楚这个SYN到底是下面哪种情况：</p><ol>
<li>在网络上丢失了，服务端收不到，自然不会有回复；</li>
<li>对端收到了但没回，就是刚才说的“静默丢包”；</li>
<li>对端收到了也回了，但这个回包在网络中丢了。</li>
</ol><p><img src="https://static001.geekbang.org/resource/image/43/5a/43eaba402d816fb9eac71de24062695a.jpg?wh=2000x777" alt=""></p><p>你看，就这么简单的一个SYN，还能引申出三种状况出来。感觉什么东西一沾上网络，就要变成麻烦事啊。所以，跟我们在<a href="https://time.geekbang.org/column/article/477510">第1讲</a>里学过的一样：设计网络协议真的不简单。</p><p>那么，从客户端的角度，对于SYN包发出去之后迟迟没有回应的情况，它的策略是做<strong>重试</strong>，而且不止一次。那会重试几次呢？重试多久呢？这个问题，一下子还不太好回答。不过，有tcpdump帮忙，我们可以搞清楚重试的问题，也可以搞清楚“TCP连接是否都用TCP协议沟通”的问题。</p><h3>动手实验</h3><p>你可以借助iptables和tcpdump做个实验，来验证这件事。你需要一台测试用的服务端，安装Ubuntu等Linux类系统，然后用你的笔记本作为客户端发起测试。这里我也放了一个视频，展示了这个实验过程，你可以结合着对照来看。</p><p><video poster="https://media001.geekbang.org/8182de625b084859a9c4a0b294528730/snapshots/2423524caf814063bfc94b8e03df7bf6-00005.jpg" preload="none" controls=""><source src="https://media001.geekbang.org/customerTrans/7e27d07d27d407ebcc195a0e78395f55/3ecf30cd-17e63c17e92-0000-0000-01d-dbacd.mp4" type="video/mp4"><source src=" https://media001.geekbang.org/6d45e3e01cdc4a38a52f112f2a9094ba/afa53eaae3254628bfb9f4499232d026-a5eb420ed8725cc8ca8e05545e274b0b-sd.m3u8" type="application/x-mpegURL"></video></p><blockquote>
<p>注意：在这个视频中，我是直接在tcpdump窗口里解读抓包结果的，而在下面我们是用Wireshark来解读，思路其实是一样的，只是操作方式略有不同，正好你可以都学习一下。</p>
</blockquote><p>第一步，在服务端，执行下面的这条命令，让iptables静默丢弃掉发往自己80端口的数据包：</p><pre><code class="language-plain">iptables -I INPUT -p tcp --dport 80 -j DROP
</code></pre><p>第二步，在客户端启动tcpdump抓包：</p><pre><code class="language-plain">sudo tcpdump -i any -w telnet-80.pcap port 80
</code></pre><p>第三步，从客户端发起一次telnet：</p><pre><code class="language-plain">telnet 服务端IP 80
</code></pre><p>这个时候，这个telnet会挂起：</p><p><img src="https://static001.geekbang.org/resource/image/be/79/bea7a95e1c15c9abcd6d14bcb2a90979.jpg?wh=644x104" alt=""></p><p>大约一两分钟后才会失败退出，你随后就会明白背后发生了什么。</p><p>这时，你可以把客户端的tcpdump停掉了（按下Ctrl+C）。然后用Wireshark打开这个抓包文件，看看里面是什么：</p><p><img src="https://static001.geekbang.org/resource/image/55/8d/552e7cb1263ffe018c3bc5cf946b3f8d.jpg?wh=1740x330" alt=""></p><p>telnet挂起的原因就在这里：<strong>握手请求一直没成功</strong>。客户端一共有7个SYN包发出，或者说，除了第一次SYN，后续还有6次重试。客户端当然也不是“傻子”，这么多次都失败，就放弃了连接尝试，把失败的消息传递给了用户空间程序，然后就是telnet退出。</p><p>这里有个信息很值得我们关注。第二列是数据包之间的时间间隔，也就是1秒，2秒，4.2秒，8.2秒，16.1秒，33秒，每个间隔是上一个的<strong>两倍</strong>左右。到第6次重试失败后，客户端就彻底放弃了。</p><p>显然，这里的翻倍时间，就是“<strong>指数退避</strong>”（<a href="https://en.wikipedia.org/wiki/Exponential_backoff">Exponential backoff</a>）原则的体现。这里的时间不是精确的整秒，因为指数退避原则本身就不建议在精确的整秒做重试，最好是有所浮动，这样可以让重试成功的机会变得更大一些。</p><p>这里实际上也是一个知识点了：<strong>TCP握手没响应的话，操作系统会做重试</strong>。在Linux中，这个设置是由内核参数net.ipv4.tcp_syn_retries控制的，默认值为6，也就是我们前面刚观察到的现象。以下就是我的Ubuntu 20.04测试机的配置：</p><pre><code class="language-plain">$ sudo sysctl net.ipv4.tcp_syn_retries
net.ipv4.tcp_syn_retries = 6
</code></pre><p>还有另外好几个有关TCP重试的设置值，也都可以调整。更全面的内容呢，你可以直接<strong>man tcp</strong>，查看tcp的内核手册的信息。比如下面就是对于tcp_syn_retries的解释：</p><pre><code class="language-plain">tcp_syn_retries (integer; default: 5; since Linux 2.2)
&nbsp; &nbsp; &nbsp; &nbsp;The&nbsp; maximum&nbsp; number of times initial SYNs for an active TCP connection attempt will be retransmitted.&nbsp; This value should not be higher than 255.&nbsp; The default value is 5, which corresponds to approximately 180 seconds.
</code></pre><p>既然静默丢包会引起客户端空等待的问题，那我们直接拒绝，应该就能解决这个问题了吧？</p><p>正好，iptables的规则动作有好几种，前面我们用DROP，那这次我们用REJECT，这应该能让客户端立刻退出了。执行下面的这条命令，让iptables拒绝发到80端口的数据包：</p><pre><code class="language-plain">iptables -I INPUT -p tcp --dport 80 -j REJECT
</code></pre><p>跟前面的实验一样，我们在客户端发起telnet 服务端IP 80。果然，telnet立刻退出，显示：</p><pre><code class="language-plain">$ telnet 47.94.129.219 80
Trying 47.94.129.219...
telnet: connect to address 47.94.129.219: Connection refused
telnet: Unable to connect to remote host
</code></pre><p>可见，连接请求确实被拒绝了。我在telnet同时也抓了包，我们来看一下抓包文件：</p><p><img src="https://static001.geekbang.org/resource/image/cd/36/cd7228b8a38ec807263d980d5887d136.jpg?wh=1556x184" alt=""></p><p>奇怪，抓包文件里并没有期望的TCP RST？是我们抓包命令没写对吗？下面是这条命令，你已经初步学过tcpdump抓包命令了，看看有没有什么问题？</p><pre><code class="language-plain">sudo tcpdump -i any -w telnet-80-reject.pcap host 47.94.129.219 and port 80
</code></pre><p>命令语法没问题，要不然命令都无法执行。那过滤条件呢？指定了远端IP和端口，这是很常见的用法，应该也没什么问题。</p><p>但是，<strong>这里隐藏了一个假设的前提</strong>，也就是我们认为，这次握手的所有过程都是通过这个80端口进行的。但事实上呢？我们稍微改一下抓包条件，只保留远端IP，去掉端口的限制：</p><pre><code class="language-plain">sudo tcpdump -i any -w telnet-80-reject.pcap host 47.94.129.219
</code></pre><p>然后再来看看，我们抓到的报文是怎样的：</p><p><img src="https://static001.geekbang.org/resource/image/e2/83/e2b17ee170d80dbdbb0efb9d0828a183.jpg?wh=1542x126" alt=""></p><p>很意外，居然对端回复了一个<strong>ICMP消息：Destination unreachable (Port unreachable)</strong>。这还不是最意外的，我们选中这个报文，进一步看它的详情，可能会更惊讶：</p><p><img src="https://static001.geekbang.org/resource/image/1e/e9/1e63ce285a59f917149d4d55fed5a1e9.jpg?wh=1646x1200" alt=""></p><p>原来，这个ICMP消息不仅通过type=3表示，这是一个“端口不可达”的错误消息，而且在它的payload里面，还携带了完整的TCP握手包的信息。而这个握手包，可是客户端发过来的。</p><blockquote>
<p>补充一下：如果我们回头再检查一下前面生成的iptables规则，它是这样的：</p>
</blockquote><pre><code>-A INPUT -p tcp -m tcp --dport 80 -j REJECT --reject-with icmp-port-unreachable
</code></pre><blockquote>
<p>原来，它自动补上了–reject-with icmp-port-unreachable，也就是说确实用ICMP消息做了回复。当然，你还可以把这个动作定义为–reject-with tcp-reset，那样的话就符合我们一开始的期望了。<br>
&nbsp;<br>
事实上，无论是收到TCP RST还是ICMP port unreachable消息，客户端的connect()调用都是返回ECONNREFUSED，这就是telnet都报“connection refused”的深层次原因。</p>
</blockquote><p>所以，这个握手失败的情况终于搞清楚了，它是这么发生的：</p><p><img src="https://static001.geekbang.org/resource/image/3a/29/3af3ae7d7ed9a6662dd66a6beafd1829.jpg?wh=2000x702" alt=""></p><p>TCP握手拒绝这个事，竟然可以是<strong>ICMP报文</strong>来达成的。“握手过程用TCP协议做沟通”，看起来这么理所当然的事情居然也会反转，你是不是也有点自我怀疑了：是不是其他网络知识，也未必是我自己认为的那样呢？</p><p>这个知识点，其实是几年前我在处理一个客户的TCP连接问题时遇到的。剧情么，前面已经给你“演”过一遍了。当时我也深感TCP的水太深，快没过脖子了，甚至有点喘不过气来……从此以后，我再也不敢小看任何知识点，同时也领教了tcpdump和Wireshark在网络分析方面的威力。有了这两个大杀器的帮助，我的网络水平提高很快。这个经验我也分享给你，相信你也一定能从中受益。</p><h2>Windows服务器加域报RPC service unavailable?</h2><p>虽然tcpdump + Wireshark的组合威力强大，但用起来总是会稍微花点时间。<strong>有没有不用抓包分析，也能做排查TCP连接问题的方法呢？</strong>这样也好快一点啊。接下来这个例子，就是这样的。</p><p>我们eBay也有不少Windows服务器，这些机器都由Active Directory（简称AD）管理。有一次，我们有一台Windows服务器加入AD失败，相关同事已经排查了好久，一直没找到原因。操作过程就是最普通的加域动作：</p><p><img src="https://static001.geekbang.org/resource/image/76/aa/76770d2be1c188dfafd69c906cb5b1aa.jpg?wh=325x365" alt=""></p><p>然后，一开始显示加域成功，但是过一两分钟后，又会来个“回马枪”，冒出来一个The RPC server is unavailable的报错：</p><p><img src="https://static001.geekbang.org/resource/image/59/7f/59793b82eyy0e8855e1879e5943f057f.jpg?wh=255x138" alt=""></p><p>在Windows的体系里面，这个报错大体意思是连不上RPC服务器。同事检查过RPC服务端并没有问题，然后其他Windows客户端加域呢，也都正常，唯独这台就不行。</p><p>单独一台机器加不了域，本身也不是特别大的麻烦，但是同事还是想找一下根因，于是就让我帮忙。很幸运，当时我只用了大概十分钟就找到了原因（这里我有点不谦虚了，我对你扔过来的鸡蛋和番茄表示接受）。</p><p>这倒不是我对Windows多么精通，<strong>主要是正确的排查思路帮助了我</strong>。给你分享一下我当时的思路：</p><ol>
<li>既然报错是RPC unavailable，那可能意味着有一个RPC服务没有得到响应。</li>
<li>没有得到服务端的响应，那多半是跟网络有关系，特别是跟端口的连通性有关系。</li>
<li>要知道，RPC使用的是动态端口，每次连接都可能连接到不同的服务端口。所以，我也<strong>没办法预先知道是具体哪几个端口</strong>，如果我知道的话，直接找防火墙团队去把那几个服务端口打开就好了，但这个做不到。这一点也是同事卡了许久的原因之一，他也不知道如何找到这些“动态会变的RPC端口”。</li>
<li>要找到<strong>实时在用</strong>的动态RPC端口，最方便的方法就是<strong>运行netstat命令</strong>。无论连接是处在什么状态，比如是在传输数据的ESTABLISHED状态、新近关闭端口的TIME_WAIT状态，都可以用netstat命令看到。</li>
<li>我运行了netstat，在当时的命令输出中，我注意到有一个 <strong>SYN_SENT状态</strong>的连接，它要连的就是服务端的一个高端口。</li>
</ol><p>那么，这个SYN_SENT状态究竟说明了什么呢？</p><p><img src="https://static001.geekbang.org/resource/image/bd/46/bdd5af2530ef0b810863947af22be046.jpg?wh=633x23" alt=""></p><p>SYN_SENT是TCP的11个状态之一。要理解SYN_SENT的含义，我们首先要把整个TCP状态机的机制搞清楚。关于TCP状态机，目前流传比较广的是下面这张图。我没有考证过这张图的出处，不过在Stevens的<a href="https://book.douban.com/subject/4859464">《UNIX网络编程：套接字联网API》</a>里就有这张图，很有可能最早就是来自于Stevens：</p><p><img src="https://static001.geekbang.org/resource/image/69/55/6918091f084186bb968180c2e7aacf55.jpg?wh=754x858" alt=""></p><p>这张图浓缩了TCP状态转换的所有知识点，确实值得反复研读。不过，我鸡蛋里挑个骨头：这张图也有个小小的问题，就是对于初学者来说，它并不容易理解。</p><p>比如，多年前我自己在学习TCP的时候，就一直没有彻底看懂这张图。好笑的是，我经常假装自己看懂了，还拿这张图跟别人侃侃而谈，而对方还被我唬住了呢。所以你也要学会了：当大家都不是很懂的时候，你对自己的话越相信，你就越有说服力哦。</p><p>好了，当然是跟你开个玩笑，做学问还是要严谨。那么，这张图的难点在哪呢？我觉得主要是视角不固定，一会是发送方，一会是接收方，对初学者来说很容易混淆。实际上，在Stevens的这本书中，还有另外一张图，我认为更加清晰明了，也是我想推荐给你的：</p><p><img src="https://static001.geekbang.org/resource/image/4b/99/4b5dbdf4365d467cce222c9cf6707099.jpg?wh=607x513" alt=""></p><p>在上面这张图里，无论是客户端还是服务端，我们从上往下看，它要经历的各个TCP状态，都展示得十分清楚。我把这个过程解读如下：</p><p><img src="https://static001.geekbang.org/resource/image/ec/ce/ec2f7ee67f560420bced033059c31ece.jpg?wh=2000x1125" alt=""></p><p>后续的过程，不用我继续解读，你也会看得很清楚了：<strong>分别沿着左边和右边的垂直线从上往下看，就经历了客户端和服务端的TCP生命周期里的各种状态</strong>，这个过程中，视角保持一致。你觉得是否比前面那张转换图，更加容易理解呢？</p><p>看懂了这张图，你应该就明白了：SYN_SENT这个状态，意味着当时这个连接请求（SYN包），已经从这台Windows服务器发出，试图跟远端的AD域控制器进行连接。但由于对端迟迟没有回应SYN+ACK报文，那么客户端这个连接的状态，就只能“停留”在SYN_SENT状态，无法转化为ESTABLISHED状态。</p><p>等到达了SYN timeout时间后，Windows操作系统会放弃这次连接，而这个SYN_SENT状态的连接也会消失不见。所以，前面提到的“<strong>实时</strong>”两字，也是很关键的。如果不是在问题发生时运行netstat，哪怕是过了几分钟再去运行netstat，错过了这个SYN_SENT，我也不能发现这个失败的TCP连接企图，也就无法定位到真正的原因了。</p><p><img src="https://static001.geekbang.org/resource/image/83/18/8357d8df335042e2d781f540c9e85a18.jpg?wh=2000x920" alt=""></p><p>然后我们拿着这个端口去找防火墙团队，对方检查了配置，发现这个端口确实是禁止的。在开通后，问题就解决了。</p><p>所以说，<strong>真的不要小看任何知识点和小工具，你掌握以后，完全可以起到关键性的作用</strong>（对了，排查防火墙也时常是我们工作的痛点，我在第5和第6讲会专门讲解这方面的排查技巧，敬请期待）。</p><p>这里还有一个技术点我想给你展开一下。我们在前面已经讨论过了SYN重试的问题，显然，这次Windows的SYN_SENT的背后，我们相信，应该也是有数次的SYN重试的情况。同时，因为我观察到，这个SYN_SENT停留了大约有十几二十秒，所以我判断应该也有指数退避的存在，所以这个状态才保留了那么长时间。</p><p>也就是说，无论是Linux还是Windows，都实现了类似的TCP握手方面的容错手段。还是那句话：设计网络不容易。<strong>理解了设计者的初心</strong>，很多问题就不会那么模糊了，可能你一下子就能看清。</p><h2>发送的数据还能超过接收窗口？</h2><p>最后一个案例表面上并不直接跟握手相关，但背地里就……不剧透了，看剧情。</p><p>前段时间，有个朋友找到我咨询一个问题。他们最近处理了一个Redis相关的技术问题，让他们既开心又“闹心”。开心的是整体分析是正确的，问题也得以解决；“闹心”的是，唯独有个技术点好像无法自圆其说，所以想让我看看到底是怎么回事。</p><p>这个问题是：Redis服务告诉客户端它的接收窗口是190字节，但是客户端居然会发送308字节，<strong>大大超出了接收窗口</strong>。下图是他们用Wireshark打开抓包文件后的界面：</p><p><img src="https://static001.geekbang.org/resource/image/10/yy/1042be623e0390e3c96a9675a73a45yy.jpg?wh=2122x522" alt=""></p><p>我一开始也懵了：难道TCP的深水又到我脖子这儿了？在我多年的抓包分析经历中，数据超过接收窗口的情况，好像还没有遇到过，这次算是TCP准备再次让我“开开眼”吗？</p><p>不过我很快又稳定了下来，因为我想到了一个朋友他们没有注意到的细节。在说到TCP窗口的时候，一般都会提到一个很重要的概念：<strong>Window Scale</strong>。这是因为，TCP最初是七八十年代的产物，1981年9月定稿的<a href="https://datatracker.ietf.org/doc/html/rfc793">RFC793</a>才第一次正式确定了TCP的标准。当时的网络带宽还处于“石器时代”，机器的带宽只有现在的百分之一，那么TCP接收窗口自然也没必要很大，2个字节长度代表的65535字节的窗口足矣。</p><p>但是后来网络带宽越来越大，65535字节的窗口慢慢就不够用了，于是设计者们又想出了一个巧妙的办法。原先的Window字段还是保持不变，在TCP扩展部分也就是TCP Options里面，增加一个Window Scale的字段，它表示原始Window值的左移位数，最高可以左移14位。</p><p>如果你还没有完全忘记计算机课的基本知识，那么应该明白这是一个非常大的提升了（扩大了2的14次方，即16384倍）。16384乘以65535，这个数字就是1G字节，也就是说，一个启用了Window Scale特性的TCP连接，最大的接收窗口可以达到1GB。可以说，这个数字至今都是够用的。</p><p>说了这么多，我们用Wireshark来看看它究竟长啥样。找一个包含了SYN报文的抓包文件，选中SYN报文，在Wireshark窗口中部找到TCP的部分，展开Options就能看到了：</p><p><img src="https://static001.geekbang.org/resource/image/88/17/8818106efb1bd4313928bd66302ef417.jpg?wh=865x626" alt=""></p><p>我们逐一理解下。</p><ul>
<li>Kind：这个值是3，每个TCP Option都有自己的编号，3代表这是Window Scale类型。</li>
<li>Length：3字节，含Kind、Length（自己）、Shift count。</li>
<li>Shift count：6，也就是我们最为关心的窗口将要被左移的位数，2的6次方就是64。</li>
</ul><blockquote>
<p>小小提醒：SYN包里的Window是不会被Scale放大的，只有握手后的报文才会。</p>
</blockquote><p>当然，TCP的窗口也是TCP知识体系里一块挺大的分支领域，我会在当前这个“实战一”模块的传输效率部分，也就是第9~11讲里，详细讲解这方面的知识，帮你把这块的东西真正搞透。</p><p>回到握手。既然Window Scale这么有用，那每个TCP报文应该都是带上这个信息的吧，因为它在TCP头部里面嘛，而每个TCP报文都有头部的，不是吗？</p><p>你要这样想就错了。事实上，Window Scale只出现在TCP握手里面。你再想想就明白了：这个是“控制面”的信息，说一次让双方明白就够了，每次都说，不光显得“话痨”，也很浪费带宽啊。一般传输过程中的报文，完全不需要再浪费这3个字节来传送一个已经同步过的信息。所以，握手之后的TCP报文里面，是不带Window Scale的。</p><p>比如，我们来看一个抓取到握手阶段的抓包文件。下图是客户端在数据传输阶段发送的报文，它是一个TLS Client Hello报文。</p><p><img src="https://static001.geekbang.org/resource/image/b8/fe/b8b79b774695579a065b2da5f62a33fe.jpg?wh=1478x800" alt=""></p><p>可见，原始窗口502字节，放大128倍后就是64256字节了。</p><p>说到这里，想必你已经明白了：我朋友这次的疑惑，其实就是<strong>缺少TCP握手包</strong>造成的。要知道，Wireshark也一样要依赖握手包，才能了解到这次连接用的Window Scale值，然后才好在原始Window值的基础上，对Window值进行左移（放大），得出真正的窗口值。于是，因为这次他们的抓包没有抓取到握手报文，所以Wireshark里看到的窗口，就是190字节，而不是190字节的某个倍数了！</p><p>当时通信的另一端当然知道这个信息，所以它发送308字节一点都不意外，因为这个值根本就没超出接收窗口。</p><p>那么，<strong>是不是没有抓取到握手包的话，Wireshark里读取到的Window就一定不对呢？</strong>大部分时候是这样的。不过，还有一部分老系统的TCP栈并没有启用Window Scale，那么抓包文件中有没有握手包都没关系，只要看基本Window就好了。</p><p>说到这里，你对TCP握手的印象，是不是又有改变呢？它简单，也丰富；它靠谱，也调皮。你只有真的读懂它，才不会被它牵着鼻子走。而读懂它的方法是什么呢？</p><p>就是多读些TCP理论，就是多做些抓包分析，就是多处理些案例，更是多走走，多看看。只要有心，你总有机会可以学会，可以成长。</p><h2>小结</h2><p>作为这个模块的第一课，这次我们围绕TCP握手展开了几个有趣的案例，并从中梳理了以下知识点：</p><ol>
<li>客户端发起的连接请求可能因为各种原因没有回复，这时客户端会做<strong>重试</strong>。一般在Linux里，重试次数默认是6次，内核参数是net.ipv4.tcp_syn_retries。重试间隔遵循了<strong>指数退避原则</strong>。</li>
<li>服务端拒绝TCP握手，除了用TCP RST，另外一种方式是通过<strong>ICMP Destination unreachable（Port unreachable）消息</strong>。从客户端应用程序看，这两种回复都属于“对端拒绝”，所以应用表面看不出区别，但我们在抓包的时候要注意，如果单纯抓取服务端口的报文，就会漏过这个ICMP消息，可能对排查不利。</li>
<li>对于连通性相关的问题，除了用tcpdump+Wireshark这个黄金组合，我们还可以在理解TCP握手原理的基础上，使用小工具（比如netstat）来排查。特别是对于RPC服务场景，在问题发生时及时<strong>执行netstat -ant，找到SYN_SENT状态的连接</strong>，这个很可能是突破口。</li>
<li>我们也学习了如何在Wireshark中查看Window Scale。握手包中的Window Scale信息十分重要，这会帮助我们知道正确的接收窗口。在分析抓包文件时，<strong>要注意是否连接的握手包被抓取到</strong>，没有握手包，这个Window值一般就不准。</li>
</ol><p>可以说，<strong>应用都靠连接，连接都靠握手。</strong>掌握好了握手，你的TCP就算入门了。学完这节课之后，你有没有觉得，今天的你比昨天的你，要强一些了呢？加油！后面更多的知识在等你来发现。</p><h2>思考题</h2><p>最后，还是按照惯例，还是给你留几道思考题：</p><ol>
<li>在Linux中，还有一个内核参数也是关于握手的，net.ipv4.tcp_synack_retries。你知道这个参数是用来做什么的吗？</li>
<li>如果握手双方，一方支持Window Scale，一方不支持，那么在这个连接里，Window Scale最终会被启用吗？你可以参考<a href="https://datatracker.ietf.org/doc/html/rfc1323">RFC1323</a>，给出你的解答。</li>
</ol><p>欢迎在留言区分享你的答案，如果觉得有收获，也欢迎你把今天的内容分享给更多的朋友。</p><h2>扩展知识：聊聊几个常见误区</h2><p>很多时候，我们的成长不仅是由于学到了正确的知识，更是由于纠正了“错误的认知”。下面列几个常见误区，你看看自己有没有“中招”。</p><h3>UDP也有握手?</h3><p>有些同学会有这个误解，可能是跟nc这个命令有关。我们来看一个TCP端口22的测试：</p><pre><code class="language-plain">victor@victorebpf:~$ nc -v -w 2 47.94.129.219 22
Connection to 47.94.129.219 22 port [tcp/ssh] succeeded!
</code></pre><p>同一时间的tcpdump抓包，显示这个TCP经历了成功的握手和挥手：</p><pre><code class="language-plain">$ sudo tcpdump -i any host 47.94.129.219
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on any, link-type LINUX_SLL (Linux cooked v1), capture size 262144 bytes
11:58:10.749372 IP victorebpf.51072 &gt; 47.94.129.219.ssh: Flags [S], seq 966857509, win 64240, options [mss 1460,sackOK,TS val 1565897461 ecr 0,nop,wscale 7], length 0
11:58:10.781734 IP 47.94.129.219.ssh &gt; victorebpf.51072: Flags [S.], seq 3170176001, ack 966857510, win 65535, options [mss 1460], length 0
11:58:10.781880 IP victorebpf.51072 &gt; 47.94.129.219.ssh: Flags [.], ack 1, win 64240, length 0
11:58:10.782344 IP victorebpf.51072 &gt; 47.94.129.219.ssh: Flags [F.], seq 1, ack 1, win 64240, length 0
11:58:10.782586 IP 47.94.129.219.ssh &gt; victorebpf.51072: Flags [.], ack 2, win 65535, length 0
11:58:10.821202 IP 47.94.129.219.ssh &gt; victorebpf.51072: Flags [P.], seq 1:42, ack 2, win 65535, length 41
11:58:10.821292 IP victorebpf.51072 &gt; 47.94.129.219.ssh: Flags [R], seq 966857511, win 0, length 0
</code></pre><p>如果我们用nc测试 <strong>UDP</strong> 22端口，看看会发生什么。注意，UDP 22是没有服务在监听的。但是nc一样告诉我们succeeded！这似乎在告诉我们，这个UDP 22端口确实是在监听的：</p><pre><code class="language-plain">$ nc -v -w 2 47.94.129.219 22
Connection to 47.94.129.219 22 port [tcp/ssh] succeeded!
victor@victorebpf:~$ nc -v -w 2 47.94.129.219 -u 22
Connection to 47.94.129.219 22 port [udp/*] succeeded!
</code></pre><p>同一时间的抓包，显示客户端发送了4个UDP报文，但服务端没有任何回复：</p><pre><code class="language-plain">11:59:05.605556 IP victorebpf.54145 &gt; 47.94.129.219.22: UDP, length 1
11:59:05.605995 IP victorebpf.54145 &gt; 47.94.129.219.22: UDP, length 1
11:59:06.606436 IP victorebpf.54145 &gt; 47.94.129.219.22: UDP, length 1
11:59:07.607134 IP victorebpf.54145 &gt; 47.94.129.219.22: UDP, length 1
</code></pre><p>从表象上看，nc告诉我们：这个跟UDP 22端口的“连接”是成功的，这是nc的Bug吗？可能并不算是。原因就在于，<strong>UDP本身不是面向连接的</strong>，所以没有一个确定的UDP协议层面的“答复”。这种答复，需要由调用UDP的应用程序自己去实现。</p><p>那为什么在这里，nc还是要告诉我们成功呢？可能只是因为对端没有回复ICMP port unreachable。nc的逻辑是：</p><ul>
<li>对于UDP来说，除非明确拒绝，否则可视为“连通”；</li>
<li>对TCP来说，除非明确接受，否则视为“不连通”。</li>
</ul><p>所以，当你下次用nc探测UDP端口，不通的结果是可信的，而能通（succeeded）的结果并不准确，只能作为参考。</p><h3>一台机器最多65535个TCP连接?</h3><p>这也是很常见的误区了。我还是小白的时候，也曾经深信不疑。当时读到一篇讨论服务器可以承受多少TCP连接（就是C10k问题）的文章时，还觉得奇怪，不是端口范围只有0~65535吗？为什么还会有几十万上百万连接呢？</p><p>这就是没有意识到，连接是四元组（咱们在<a href="https://time.geekbang.org/column/article/477510">第一节课</a>讲到过），并不是单纯的源端口或者目的端口。那么多个数相乘，这个乘积当然可以远远超过65535了。先不谈论海量级网站的场景，就算我们维护一台Web服务器，假如当前有10万台客户端连着你，平均每个客户端跟你有6个连接（这很常见），那么就是60万个连接了，是不是也早就超过6万了？</p><p>当然，在限定场景下，一个客户端（假设只有一个出口IP）和一个服务端（假设也只有一个IP和一个服务端口），那么确实只能最多发起6万多个连接。但你自己也已经明白，这跟前面的误解，已经是两回事了。</p><h3>不能同时发起握手?</h3><p>如果两端同时发送了SYN给对方，也就是双方都收到了一个SYN，那么接下来，它们会进入什么状态呢？你可能觉得这应该不行。</p><p>其实，通信双方还真的可以同时向对方发送SYN，也能建立起连接。你可以参考这节课里我提到的<strong>TCP状态转换图</strong>。在Richard Stevens的<a href="https://book.douban.com/subject/1088054">《TCP/IP详解（第一卷）》</a>里，也提到了这个知识点，参考下图：</p><p><img src="https://static001.geekbang.org/resource/image/02/12/0218c45cf90c06983e6e0fa6211f6912.jpg?wh=1384x614" alt=""></p><p>当然，这种情况是很罕见的，你可以参考一下，也丰富一下你对TCP握手的理解。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/d3/cc/bc01a9ef.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>某人</span>
  </div>
  <div class="_2_QraFYR_0">数退避原则本身就不建议在精确的整秒做重试，为什么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 假设这样一个场景：<br>1. 接收端有个定时任务，比如每隔1秒、2秒、4秒。。。这样反复运行，正好发送端的SYN包卡在第1秒，因为定时任务占用了全部CPU资源，导致SYN+ACK没有及时发出<br>2. 发送端接着正好也卡着2秒、4秒。。。这种时间点，每次都赶上对端那个定时任务，所以每次SYN都没有响应<br>3. 握手失败<br><br>这就是双方都精确翻倍的话，可能遇到的问题。同样的道理，对于一般的应用逻辑，其“重试”也最好是随机时间，或者是翻倍时间附近有一定的浮动，这样可以避免每次都“撞车”。不知道我这样说清楚没有：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-18 12:07:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/24/b7/08/a6493073.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>MeowTvづ</span>
  </div>
  <div class="_2_QraFYR_0">对于服务器端最多65535确实是个误区，其实这个跟很多都有关系的，比如服务器端的CPU、内存、fd数以及连接的情况，fd数是前提。一个连接会牵扯到服务端的接收缓冲区(net.ipv4.tcp_rmem)以及发送缓冲区(net.ipv4.tcp_wmem)，一个空的TCP连接会消耗3.3KB左右的内存，如果发数据的话，一个连接占用的内存会更大。所以理论上4GB的机器理论上支持的空TCP连接可以达到100W个。此外数据经过内核协议栈的处理需要CPU，所以CPU的好坏也会影响连接数。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，所以C10K, C100K等经典问题就是讨论了这个主题~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-09 17:48:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_cad238</span>
  </div>
  <div class="_2_QraFYR_0">其实Window Scale是常识，并不是冷门，😂，关于这个，在林沛满大佬的《wireshark网络分析就这么简单》一书里有详细说明，大家可以一看。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍林老师是这方面的先行者，非常厉害！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-20 20:18:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/a1/bc/ef0f26fa.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>首富手记</span>
  </div>
  <div class="_2_QraFYR_0">一个客户端（假设只有一个出口 IP）和一个服务端（假设也只有一个 IP 和一个服务端口），那么确实只能最多发起 6 万多个连接。针对这句话，在centos 和ubuntu系统默认的情况下，tcp是没有办法建立起6万多个链接的，因为 net.ipv4.ip_local_port_range 这个参数固定了机器当做client 发起请求的时候使用的端口范围，所以默认的情况下，单向智能建立28231 个链接，这个是我们真实生产服务器上发生过的问题；<br>因为程序释放tcp有问题，所以机器上的timewite 过多，然后把这两万多个端口用完了，导致了服务之间链接异常；</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯 6万多连接是纯理论的讨论，从TCP协议规范来说，最大确实有这么大。操作系统也在进化，在很早以前，1024以上的端口都可以用作动态源端口，但现在确实窄了很多了。<br>理论知识（比如TCP的端口最大范围），现实知识（比如当前linux普遍支持的local port range）并不矛盾，也最好都知道。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-18 06:32:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/5b/96/57d4970d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>魏玉会 Gabby</span>
  </div>
  <div class="_2_QraFYR_0">老师讲的真好，我一个前端人员也能看的懂</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对我也是很好的鼓励，大家一起加油：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-20 20:13:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/c3/eb/528a1d68.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>beanSeedling</span>
  </div>
  <div class="_2_QraFYR_0"><br>1.<br>第二次握手最大重传次数<br>tcp_synack_retries (integer; default: 5; since Linux 2.2)<br>              The maximum number of times a SYN&#47;ACK segment for a<br>              passive TCP connection will be retransmitted.  This number<br>              should not be higher than 255.<br>2.<br>This option is an offer, not a promise; both sides must send Window Scale options in their SYN segments to enable window scaling in either direction.<br>This option may be sent in an initial &lt;SYN&gt; segment (i.e., a segment with the SYN bit on and the ACK bit off).  It may also be sent in a &lt;SYN,ACK&gt; segment, but only if a Window Scale option was received in the initial &lt;SYN&gt; segment. <br>不会，从上诉RFC原文可以看出是必须双方都支持Window Scale，才会启用<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 很好，也看得出来做了功课：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-18 18:19:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/d0/2b/c571c59f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>steven</span>
  </div>
  <div class="_2_QraFYR_0">在握手期间window是不会被scale放大的，但是我发现在传输过程中的window有260000+，但是握手的时候我客户端最大只有65535，服务端只有8000+，和我理解的有点不太一样呀</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对呀，握手期间，也就是发送SYN, SYN+ACK, ACK这段时间里，窗口是不放大的，也就是最大也只有65535。握手完成后也就是传输开始后，就可以应用这个放大系数了，跟你描述的现象很符合啊：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-12 13:23:50</div>
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
  <div class="_2_QraFYR_0">CentOS Linux release 7.6 , net.ipv4.tcp_syn_retries = 6  设置静默丢包，客户端重试的时候，发现尝试了 11次，前5次是每隔1s 后面几次就根据指数退避原则了，我这个环境为什么会多了 4次呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个问题有意思，我找个时间搭个环境测试一下。不过我推测，有可能其中几次是TCP retransmission，是syn_retries控制次数以外的~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-20 22:16:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/21/b8/aca814dd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>江山如画</span>
  </div>
  <div class="_2_QraFYR_0">问题1:<br>因为 net.ipv4.tcp_syn_retries 是三次握手的 SYN 包的重试次数，猜测 net.ipv4.tcp_synack_retries 是 SYN+ACK 包的重试次数，在 centos 上看了下，这个值默认是2.<br><br>问题2:<br>在RFC1332的2.2节 Window Scale Option中有一段 &quot;Upon receiving a SYN segment with a Window Scale option containing shift.cnt = S, a TCP sets Snd.Wind.Scale to S and sets Rcv.Wind.Scale to R; otherwise, it sets both Snd.Wind.Scale and Rcv.Wind.Scale to zero.&quot;<br><br>所以当通信双方一方支持 windows scale, 另一方不支持时，再之后的通信中，发送窗口和接口窗口的缩放比例都是0，相当于双方的 Shift Count 的值都是0.</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 回答正确！<br>对于问题2，RFC1323的以下陈述可能更加明确一些：只有SYN里带这个option，那么接收端回复的SYN+ACK里才可以带上这个option；如果SYN里没有，SYN+ACK是不应该带这个option的（当然，因为客户度本身不支持，所以SYN+ACK带不带，效果都一样）：<br><br>This option may be sent in an initial &lt;SYN&gt; segment (i.e., a segment with the SYN bit on and the ACK bit off). It may also be sent in a &lt;SYN,ACK&gt; segment, but only if a Window Scale option was received in the initial &lt;SYN&gt; segment. A Window Scale option in a segment without a SYN bit should be ignored.</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-17 21:37:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/24/da/54/9a419711.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>海风极客</span>
  </div>
  <div class="_2_QraFYR_0">我看其他文章说<br>“最大 TCP 连接数 = 客户端 IP 数×客户端端口数。对于 IPv4，客户端的 IP 数最多为 2 的 32 次方，客户端的端口数最多为 2 的 16 次方，也就是服务端单机最大 TCP 连接数约为 2 的 48 次方。”<br>跟老师您文章里说的65535不太一样，我们都知道端口只能被一个进程使用，但是又能被多个线程连接，所以我想问下到底是哪个正确呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-17 09:53:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_0d1ecd</span>
  </div>
  <div class="_2_QraFYR_0">对我们7年运维，5年TAM静下心看还是收获满满，已经强推荐给伙伴，不介意加个微信，讨论下一个10年</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢支持！有具体的案例讨论的话，可以通过专栏首页的信息，加入学习讨论群~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-05 10:18:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/97/ad/cdb3c420.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>kakashi</span>
  </div>
  <div class="_2_QraFYR_0">“一台机器最多65535个TCP连接”，确实让我懵逼了下，差点就着了你的道了哈哈，确实，一台机器有65535个端口，但不表示最多有65535个连接，连接数是按五元组定义的，一个五元组表示一个连接。不知道我理解的对不对(✿◡‿◡)</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，这个是误解，拎出来批判的：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-23 16:27:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/9f/a1/d75219ee.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>po</span>
  </div>
  <div class="_2_QraFYR_0">对于redis的案例有几个疑问，1.原始的窗口65535字节不是已经满足308字节大小了吗？还需要用到Windows scale？2.为什么服务端是通知190字节呢？为什么不是240或者其他字节大小呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你好，关于你的两个疑问：<br>1. 原始窗口最大确实是65535，但窗口是动态的，随着读写的情况以及资源情况不断调整，也就是不是每次都宣告这么大的窗口的，比如截图里就是190字节（当然这还要乘以系数）。我的朋友的疑问就在于：他以为接收窗口就是没有乘上系数的原始的190字节，而对方发了308字节，显然308&gt;190，貌似违规了。<br>2. 刚才说过，服务端（也包括客户端）宣告的接收窗口是可以动态变化的，具体宣告多少，主要取决于多个因素，比如：<br>a. 系统配置的TCP接收缓冲区参数（最小值&#47;中间值&#47;最大值）<br>b. 当时的系统资源的情况<br>c. 收到的报文大小等等<br>为什么不是240或者其他字节，这是系统根据实际情况做的决定。我们只能设置上面说的阈值参数，但不能决定动态运行中每个时刻的窗口大小。<br>关于TCP窗口的知识点，在09、10、11讲里有详细的展开，你学到那里应该能解开很多疑惑：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-12 10:36:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/38/5e/f0394817.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>费曼先生</span>
  </div>
  <div class="_2_QraFYR_0">看的出来，老师写的专栏真的很用心，内容真的很充实，收获很多！！！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-21 22:41:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/b4/94/2796de72.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>追风筝的人</span>
  </div>
  <div class="_2_QraFYR_0">1.第二次握手最大重传次数<br>tcp_synack_retries (integer; default: 5; since Linux 2.2)<br>              The maximum number of times a SYN&#47;ACK segment for a<br>              passive TCP connection will be retransmitted. This number<br>              should not be higher than 255.<br><br>2. 不会     both sides must support window Scale</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 正确，而且你去查手册确认了这些知识，赞～</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-18 17:39:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/22/0f/f9/95d1537d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>氢气球</span>
  </div>
  <div class="_2_QraFYR_0">老师讲的真好，收益匪浅！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 坚持学习，一定有收获，我们一起进步！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-18 07:57:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJcwXucibksEYRSYg6icjibzGa7efcMrCsGec2UwibjTd57icqDz0zzkEEOM2pXVju60dibzcnQKPfRkN9g/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_93970d</span>
  </div>
  <div class="_2_QraFYR_0">sudo sysctl net.ipv4.tcp_synack_retries<br>net.ipv4.tcp_synack_retries = 5</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-07 18:01:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/e4/29/07400492.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>bochs</span>
  </div>
  <div class="_2_QraFYR_0">wireshark 中tcp 的窗口和 长度之间有直接联系吗？窗口是针对 通信2端操作系统的缓冲区大小设置，通信过程中传输数据的长度始终不应大于路径中通信设备最小的mtu值 。没太明白wireshark 中lengh列和报文中的len  有必然联系吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-17 11:19:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/be/f5/e98a2199.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>稻香</span>
  </div>
  <div class="_2_QraFYR_0">老师，你好。这里有点不太明白。根据四元祖，这句话是不是有点问题呢？<br>在限定场景下，一个客户端（假设只有一个出口 IP）和一个服务端（假设也只有一个 IP 和一个服务端口），那么确实只能最多发起 6 万多个连接。<br>这种情况，源端端口不是可以变化吗？端口变化后不就是一个新的连接吗？<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 其实我没太明白你困惑的点事什么，感觉你已经懂了：）我试图再解释一遍：<br>第五元是协议（TCP或者UDP），其实在某个确定服务协议下，这个TCP或者UDP传输协议也确定了，所以可以只关注四元：源IP，源端口，目的IP，目的端口。<br>当源IP、目的IP、目的端口这三者都确定后，唯一的变量就是源端口了。理论最大能用的也就是6万多个端口号，也就是可能有6万多个连接。<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-06 20:14:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/vP9GOCAlhOic6cYPRlyGibBt7qQUVROUIfQ7jF4BKOabSFrfegr93rHQEibM5R9lvbZXIlypw1XznGRmCzOd4PLwA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jack.mark</span>
  </div>
  <div class="_2_QraFYR_0">TCP协议讲的非常好，继续阅览中</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-03 14:51:24</div>
  </div>
</div>
</div>
</li>
</ul>