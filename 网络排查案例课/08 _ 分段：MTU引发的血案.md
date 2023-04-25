<audio title="08 _ 分段：MTU引发的血案" src="https://static001.geekbang.org/resource/audio/51/af/514795d0a98dcda2c29dfb7ed988e2af.mp3" controls="controls"></audio> 
<p>你好，我是胜辉。</p><p>在<a href="https://time.geekbang.org/column/article/477510">第1讲</a>里，我给你介绍过TCP segment（TCP段）作为“部分”，是从“整体”里面切分出来的。这种切分机制在网络设计里面很常见，但同时也容易引起问题。更麻烦的是，这些概念因为看起来都很像，特别容易引起混淆。比如，你可能也听说过下面这些概念：</p><ul>
<li>TCP分段（segmentation）</li>
<li>IP分片（fragmentation）</li>
<li>MTU（最大传输单元）</li>
<li>MSS（最大分段大小）</li>
<li>TSO（TCP分段卸载）</li>
<li>……</li>
</ul><p>所以这节课，我就通过一个案例，来帮助你彻底搞清楚这些概念的联系和区别，这样你以后遇到跟MTU、MSS、分片、分段等相关的问题的时候，就不会再茫然失措，也不会再张冠李戴了，而是能清晰地知道问题在哪里，并能针对性地搞定它。</p><h2>案例：重传失败导致应用压测报错</h2><p>我先来给你介绍下案例背景。</p><p>在公有云服务的时候，一个客户对我们公有云的软件负载均衡（LB）进行压力测试，结果遇到了大量报错。要知道，这是一个比较大的客户，这样的压测失败，意味着可能这个大客户要流失，所以我们打起十二分的精神，投入了排查工作。</p><p>首先，我们看一下这个客户的压测环境拓扑图：</p><p><img src="https://static001.geekbang.org/resource/image/9a/e0/9ae5998eb5922295ebbdab9147eff0e0.jpg?wh=1821x692" alt=""></p><p>这里的香港和北京，都是指客户在我们平台上租赁的云计算资源。从香港的客户端机器，发起对北京LB上的VIP的压力测试，也就是短时间内有成千上万的请求会发送过来，北京LB就分发这些请求到后端的那些同时在北京的服务器上。照理说，我们的云LB的性能十分出色，承受数十万的连接没有问题。不夸张地说，就算客户端垮了，LB都能正常工作。</p><!-- [[[read_end]]] --><p>但是在这次压测中，客户发现有大量的HTTP报错。我们看了具体情况，这个压测对每个请求的超时时间设置为1秒。也就是说，如果请求不能在1秒内得到返回，就会报错。而问题症状就是出现了大量这样的超时报错。</p><p>既然通过LB做压测有问题，客户就绕过LB，从香港客户端直接对北京服务器进行测试，结果发现是正常的，压测可以顺利完成。当然，因为单台服务器的能力比不上LB加多台服务器的配置，所以压测数据不会太高，但至少没有报错了。</p><p>那这样的话，客户是用不了我们的LB了？</p><p>我们还是用<strong>抓包分析</strong>来查看这个问题。显然，我们知道了两种场景，一种是正常的，一种是异常的。那么我们就在两种场景下分别做了抓包。</p><h3>绕过LB的压测</h3><p>我们来看一下绕过LB的话，请求是如何到服务器的。香港机房的客户端发起HTTP请求，通过专线进入广州机房，然后经由广州-北京专线，直达北京机房的服务器。</p><p><img src="https://static001.geekbang.org/resource/image/70/66/70b3b3925eyy21baf5b7875b625e9a66.jpg?wh=1824x690" alt=""></p><p>我们来看一下这种场景下的抓包文件：</p><blockquote>
<p>抓包文件已经脱敏后上传至<a href="https://gitee.com/steelvictor/network-analysis/blob/master/08/bypassLB.pcap">gitee</a>。</p>
</blockquote><p><img src="https://static001.geekbang.org/resource/image/fc/18/fc78697920a60ff6190808d4234f9218.jpg?wh=1724x1420" alt=""></p><p>考虑到压测的问题是关于HTTP的，我们需要查看一下HTTP事务的报文。这里只需要输入过滤器http就可以了：</p><p><img src="https://static001.geekbang.org/resource/image/cd/58/cd91d720450c3c3532ac0fa4434da758.jpg?wh=999x253" alt="图片"></p><p>我们选中任意一个这样的报文，然后Follow -&gt; TCP Stream，来看看整个过程：</p><p><img src="https://static001.geekbang.org/resource/image/a4/7c/a4f5858b2f5d81f9b64367fe9aef227c.jpg?wh=1094x319" alt="图片"></p><p>从这个正常的TCP流里，我们可以获取到以下信息：</p><ul>
<li><strong>时延（往返时间）在36.8ms</strong>，因为SYN+ACK（2号报文）和ACK（9号报文）之间，就是隔了0.036753秒，即36.8ms。</li>
<li><strong>HTTP处理时间也很快</strong>，从服务端收到HTTP请求（在10号报文），到服务端回复HTTP响应（14到16号报文），一共只花费了大约6ms。</li>
<li>在收到HTTP响应后，客户端在250ms（324号报文）后，发起了TCP挥手。</li>
</ul><p>看起来，服务器确实没有问题，整个TCP交互的过程也十分正常。那么，我们再来看一下“经过LB”的失败场景又是什么样的，然后在报文中“顺藤摸瓜”，进一步就能逼近根因了。</p><h3>经过LB的压测</h3><p>我们打开绕过LB的压测的抓包文件，在<a href="https://time.geekbang.org/column/article/482610">第7讲</a>中，我提到过利用Expert Information（专家信息）来展开排查的小技巧。那么这里也是如此。</p><blockquote>
<p>抓包文件已经脱敏后上传至<a href="https://gitee.com/steelvictor/network-analysis/blob/master/08/viaLB.pcap">gitee</a>。</p>
</blockquote><p><img src="https://static001.geekbang.org/resource/image/52/eb/52c0fb3e5bc9eb410c3fbee0d036abeb.jpg?wh=1792x1198" alt=""></p><p>可以看到，里面有标黄色的Warning级别的信息，有50个需要我们注意的RST报文：</p><p><img src="https://static001.geekbang.org/resource/image/40/16/4040e386a4642a2da06faa61f332cb16.jpg?wh=1816x306" alt=""></p><p>我们展开Warning Connection reset (RST)，选一个报文然后找到它的TCP流来看一下：</p><p><img src="https://static001.geekbang.org/resource/image/0b/e8/0b98ebd04e4ce8dc33c6e241d8eed7e8.jpg?wh=1890x1016" alt="图片"></p><p>比如上图中，我们选中575号报文，然后Follow -&gt; TCP Stream，就来到了这条TCP流：</p><p><img src="https://static001.geekbang.org/resource/image/2a/02/2ab7de2bfc73ffc0304596f63e15bb02.jpg?wh=1920x623" alt="图片"></p><p>不过这里，我先考考你，<strong>目前这个抓包是在客户端还是服务端抓取的呢？</strong></p><p>如果你对<a href="https://time.geekbang.org/column/article/478189">第2讲</a>还有印象，应该已经知道如何根据TTL来做这个判断了。没错，因为SYN+ACK报文的TTL是64，所以这个抓包就是在发送SYN+ACK的一端，也就是服务端做的。</p><p>接下来就是重点了。显然，这个TCP流里面，有两个重复确认（54和55号报文），还有两个重传（154和410号报文），在它们之后，就是客户端（源端口53362）发起了TCP挥手，最终以客户端发出的RST结束。</p><p>这里隐含着两个疑问，我们能回答这两个疑问了，那么问题的根因也就找到了。</p><h4>第一个疑问：为什么有重复确认（DupAck）？</h4><p>重复确认在TCP里面有很重要的价值，它的出现，一般意味着传输中出现了丢包、乱序等情况。我们来看看这两个重复确认报文的细节。</p><p><img src="https://static001.geekbang.org/resource/image/9c/bc/9cb992205f77253dac247a31f3e8cdbc.jpg?wh=996x74" alt="图片"></p><p>我们很容易发现，这两个DupAck报文的确认号是1。这意味着什么呢？你现在对TCP握手已经挺熟悉了，显然应该能想到，这个1的确认号，其实就是<strong>握手阶段完成时候的确认号</strong>。也就是说，客户端其实并没有收到握手后服务端发送的第一个数据报文，所以确认号“停留”在1。</p><p>那么，为什么是两个重复确认报文呢？我们把视线从2个DupAck报文往上挪，关注到整个TCP流的情况。</p><p><img src="https://static001.geekbang.org/resource/image/cd/9e/cd0ef78f432565b472a8fcf39595d69e.jpg?wh=786x294" alt=""></p><p>握手完成后，客户端就发送了POST请求，然后服务端先回复了一个ACK，确认收到了这个请求。之后有连续3个报文作为HTTP响应，返回给客户端。</p><p>按照TCP的机制，它可以收一个报文，就发送一个确认报文；也可以收多个报文，发送一个确认报文。反过来说，一端发送几次确认报文，就意味着它收到了至少同样数量的数据报文。</p><p>在当前的例子里，因为有2个DupAck报文，那么客户端一定至少收到了2个数据报文。是哪两个呢？一定是连续3个报文的第二和第三个，也就是1388字节报文的后面两个。因为如果是收到了1388字节那个，那确认号就一定不是1，而是1389（1388+1）了。</p><p>我们再把视线从2个DupAck往下挪，这里有2个TCP重传。</p><p>我们关注一下Time列，第一个重传是隔了大约200ms，第二次重传隔了大约472毫秒。这就是<strong>TCP的超时重传机制</strong>引发的行为。关于重传这个话题，后续课程里会有大量的展开，你可以期待一下。</p><p><img src="https://static001.geekbang.org/resource/image/0f/64/0f6c4a19bca1918d75099420de2c6164.jpg?wh=1798x152" alt="图片"></p><p>那么结合上面这些信息，我们也就理解了“通过LB压测失败”的整个过程，在TCP里面具体发生了什么。我还是用示意图来展示一下：</p><p><img src="https://static001.geekbang.org/resource/image/e2/97/e2973c5380261123626b5a3754626c97.jpg?wh=2000x1125" alt=""></p><p>不过你也许会问：“每次这样画一个示意图，好像比较麻烦啊？难道Wireshark就不能提供类似的功能吗？”</p><p>Wireshark主窗口里展示的报文，确实有点类似“一维”，也就是从上到下依次排列，在解读通信双方的具体行为时，如果能添加上另外一个“维度”，比如增加向左和向右的箭头，是不是可以让我们更容易理解呢？</p><p>其实，我们能想到的，Wireshark的聪明的开发者也想到了，Wireshark里确实有一个小工具可以起到这个作用，它就是<strong>Flow Graph</strong>。</p><p>你可以这样找到它：点开Statistics菜单，在下拉菜单中找到Flow Graph，点击它，就可以看到这个抓包文件的“二维图”了。不过，因为我们要查看的是过滤出来的TCP流，而Flow Graph只会展示抓包文件里所有的报文，所以，我们需要这么做：</p><ul>
<li>先把过滤出来的报文，保存为一个新的抓包文件；</li>
<li>然后打开那个新文件，再查看Flow Graph。</li>
</ul><p>比如这次我就可以看到下面这个Flow Graph：</p><p><img src="https://static001.geekbang.org/resource/image/03/5f/03ab05db4962c3b2d81e0471a083bf5f.jpg?wh=1470x1206" alt=""></p><p>上图读起来，是不是感觉信息量要比主界面要多一些？特别是有了左右方向箭头，给我们大脑形成了“第二个维度”，报文的流向可以直接看出来，而不再去看端口或者IP去推导出流向了。</p><p>好了，“为什么会有重复确认”的问题，我们搞清楚了，它就是由于三个报文中，第一个报文没有到达客户端，而后两个到达的报文触发了客户端发送两次重复确认。我们接下来看更为关键的问题。</p><h4>第二个疑问：为什么重传没有成功？</h4><p>第一个报文就算暂时丢失，后续也有两次重传，为什么这些重传都没成功呢？既然我们同时有成功情况和不成功情况下的抓包文件，那我们直接比较，也许就能找到原因了。</p><p>让我们把两个文件中的类似的TCP流对比一下：</p><p><img src="https://static001.geekbang.org/resource/image/87/74/873c5cb8372a68fe88489d9383848a74.jpg?wh=1920x454" alt="图片"></p><p>你能发现其中的不同吗？这应该还是比较容易发现的，它就是：<strong>HTTP响应报文的大小</strong>。两次测试中，虽然HTTP响应报文都分成了3个TCP报文，但最大报文大小不同：左边是1348，右边是1388，相差有40字节。既然已经提到了报文大小，那你应该会联想到我们这节课的主题，MTU了吧？</p><p><strong>MTU，中文叫最大传输单元，也就是第三层的报文大小的上限。</strong>我们知道，网络路径中，小的报文相对容易传输，而大的报文遇到路径中某个MTU限制的可能会更大。那么在这里，假如这个问题真的是MTU限制导致的，显然，1388会比1348更容易遇到这个问题！</p><p><img src="https://static001.geekbang.org/resource/image/83/1b/83b1ac2130fecb5f46d719c30184671b.jpg?wh=2000x799" alt=""></p><p>就像上面示意图展示的那样，如果路径中有一个偏小的MTU环节，那么完全有可能导致1388字节的报文无法通过，而1348字节的报文就可以通过。</p><p>而且，因为MTU是一个静态设置，在同样的路径上，一旦某个尺寸的报文一次没通过，后续的这个尺寸的报文全都不能通过。这样的话，后续重传的两次1388字节的报文也都失败这个事实，也就可以解释了。</p><p>既然问题跟MTU有关，我们就检查了客户端到服务端之间的一整条链路，发现了一个之前没注意到的情况：除了广州到北京之间有一条隧道，在北京LB到服务端之间，还有一条额外的隧道。我们在<a href="https://time.geekbang.org/column/article/481042">第5讲</a>里学习过，<strong>隧道会增加报文的大小</strong>。而正是这条额外隧道，造成了报文被封装后，超过了路径最小MTU的大小！从下面的示意图中，我们能看到两次路径上的区别所在：</p><p><img src="https://static001.geekbang.org/resource/image/61/c3/61998617c7ae9cef8c2a7f66fdb634c3.jpg?wh=2000x474" alt=""></p><p>经过LB的时候，报文需要做2次封装（Tunnel 1和Tunnel 2），而绕过LB就只要做1次封装（只有Tunnel 1）。跟生活中的例子一样，同样体型的两个人，穿两件衣服的那个看起来比穿单衣的那个要显胖一点，也是理所当然。要显瘦，穿薄点。或者实在要穿两件，那只好自己锻炼瘦身（改小自己的MTU）了！</p><p>另外，由于Tunnel 1比Tunnel 2的封装更大一些，所以服务端选择了不同的传输尺寸，一个是1388，一个是1348。</p><h4>第三个疑问：为什么重传只有两次？</h4><p>一般我们印象里TCP重传会有很多次，为什么这个案例里只有两次呢？如果你能联想到<a href="https://time.geekbang.org/column/article/479163">第3讲</a>里提到的多个内核TCP配置参数，那可能你会想到<code>net.ipv4.tcp_retries2</code>这个参数。确实，通过这个参数的调整，是可以把重传次数改小，比如改为两次的。不过在这个案例里不太可能。一方面，除非有必要，没人会特地去改动这个值；另外一个原因，是因为我们找到了更合理的解释。</p><p>这个解释就是<strong>客户端超时</strong>，这一点其实我在前面介绍案例的时候就提到过。从TCP流来看，从发送POST请求开始到FIN结束，一共耗时正好在1秒左右。我们可以把Time列从显示时间差（delta time）改为显示绝对时间（absolute time），得到下图：</p><p><img src="https://static001.geekbang.org/resource/image/b7/f8/b714802046bfd8220d580155f621dbf8.jpg?wh=1480x638" alt="图片"></p><p>可见，客户端在0.72秒发出了POST请求，在1.72秒发出了TCP挥手（第一个FIN），相差正好1秒，更多的重传还来不及发生，连接就结束了。</p><p>这种“整数值”，一般是跟某种特定的（有意的）配置有关，而不是偶然。那么显然，这个案例里，客户端压测程序配置了1秒超时，目的也容易理解：这样可以保证即使一些请求没有得到回复，客户端还是可以快速释放资源，开启下一个测试请求。</p><h2>一般对策</h2><p>其实，我估计你在日常工作中也可能遇到过这种MTU引发的问题。那一般来说，我们的对策是把两端的MTU往下调整，使得报文发出的时候的尺寸就小于路径最小MTU，这样就可以规避掉这类问题了。</p><p>举个例子，在我的测试机上，执行<strong>ip addr</strong>命令，就可以查看到各个接口的MTU，比如下面的输出里，enp0s3口的当前MTU是1500：</p><pre><code class="language-plain">$ ip addr
1: enp0s3: &lt;BROADCAST,MULTICAST,UP,LOWER_UP&gt; mtu 1500 qdisc fq_codel state UP group default qlen 1000
&nbsp; &nbsp; link/ether 08:00:27:09:92:f9 brd ff:ff:ff:ff:ff:ff
&nbsp; &nbsp; inet 192.168.2.29/24 brd 192.168.2.255 scope global dynamic enp0s3
&nbsp; &nbsp; &nbsp; &nbsp;valid_lft 82555sec preferred_lft 82555sec
&nbsp; &nbsp; inet6 fe80::a00:27ff:fe09:92f9/64 scope link
&nbsp; &nbsp; &nbsp; &nbsp;valid_lft forever preferred_lft forever
</code></pre><p>而假如，路径上有一个比1500更小的MTU，那为了适配这个状况，我们就需要调小MTU。这么做很简单，比如执行以下命令，就可以把MTU调整为1400字节：</p><pre><code class="language-plain">$ sudo ip link set enp0s3 mtu 1400
</code></pre><h2>“暗箱操作”</h2><p>那除了这个方法，是不是就没有别的方法了呢？其实，我喜欢网络的一个重要原因是，它有很强的“可玩性”。<strong>只要我们有可能拆解网络报文，然后遵照协议规范做事情，那还是有不少灵活的操作空间的。</strong>你可能会好奇：这听起来有点像“灰色地带”一样，难道网络还能玩“潜规则”吗？</p><p>比如这次的案例，网络环节都是软件路由和软件网关，所以“暗箱操作”也成了可能，我们不需要修改两端MTU就能解决这个问题。是不是有点神奇？不过，你理解了TCP和MTU的关系，就会明白这是如何做到的了。</p><p>MTU本身是三层的概念，而在第四层的TCP层面，有个对应的概念叫<strong>MSS，Maximum Segment Size（最大分段尺寸），也就是单纯的TCP载荷的最大尺寸</strong>。MTU是三层报文的大小，在MTU的基础上刨去IP头部20字节和TCP头部20字节，就得到了最常见的MSS 1460字节。如果你之前对MTU和MSS还分不清楚的话，现在应该能搞清楚了。</p><p><img src="https://static001.geekbang.org/resource/image/13/64/1328c6cdda41681e7a199c06dcaa6964.jpg?wh=1655x495" alt=""></p><p>MSS在TCP里是怎么体现的呢？其实我在TCP握手那一讲里提到过 <a href="https://time.geekbang.org/column/article/479163">Window Scale</a>，你很容易能联想到，MSS其实也是在握手阶段完成“通知”的。在SYN报文里，客户端向服务端通报了自己的MSS。而在SYN+ACK里，服务端也做了类似的事情。这样，两端就知道了对端的MSS，在这条连接里发送报文的时候，双方发送的TCP载荷都不会超过对方声明的MSS。</p><p>当然，如果发送端本地网口的MTU值，比对方的MSS + IP header + TCP header更低，那么会以本地MTU为准，这一点也不难理解。这里借用一下 <a href="https://datatracker.ietf.org/doc/html/rfc879">RFC879</a> 里的公式：</p><blockquote>
<p><strong>SndMaxSegSiz = MIN((MTU - sizeof(TCPHDR) - sizeof(IPHDR)), MSS)</strong></p>
</blockquote><p>MTU是两端的静态配置，除非我们登录机器，否则改不了它们的MTU。但是，它们的TCP报文却是在网络上传送的，而我们做“暗箱操作”的机会在于：<strong>TCP本身不加密，这就使得它可以被改变！</strong>也就是我们可以在中间环节修改TCP报文，让其中的MSS变为我们想要的值，比如把它调小。</p><p>这里立功的又是一张熟悉的面孔：<strong>iptables</strong>。在中间环节（比如某个软件路由或者软件网关）上，在iptabes的FORWARD链这个位置，我们可以添加规则，修改报文的MSS值。比如在这个案例里，我们通过下面这条命令，把经过这个网络环节的TCP握手报文里的MSS，改为1400字节：</p><pre><code class="language-plain">iptables -A FORWARD -p tcp --tcp-flags SYN SYN -j TCPMSS --set-mss 1400
</code></pre><p>它工作起来就是下图这样，是不是很巧妙？通过这种途中的修改，两端就以修改后的MSS来工作了，这样就避免了用原先过大的MSS引发的问题。我称之为“暗箱操作”，就是因为这是通信双方都不知道的一个操作，而正是这个操作不动声色地解决了问题。</p><p><img src="https://static001.geekbang.org/resource/image/yy/c5/yy96d3041c813231222a87fc3f77d5c5.jpg?wh=2000x1125" alt=""></p><h2>什么是TSO？</h2><p>前面说的都是操作系统会做TCP分段的情况。但是，这个工作其实还是有一些CPU的开销的，毕竟需要把应用层消息切分为多个分段，然后给它们组装TCP头部等。而为了提高性能，网卡厂商们提供了一个特性，就是让这个分段的工作<strong>从内核下沉到网卡上来完成</strong>，这个特性就是<strong>TCP Segmentation Offload</strong>。</p><p>这里的offload，如果仅仅翻译成“卸载”，可能还是有点晦涩。其实，它是off + load，那什么是load呢？就是CPU的开销。如果网卡硬件芯片完成了这部分计算任务，那么CPU就减轻负担了，这就是offload一词的真正含义。</p><p>TSO启用后，发送出去的报文可能会超过MSS。同样的，在接收报文的方向，我们也可以启用GRO（Generic Receive Offload）。比如下图中，TCP载荷就有2800字节，这并不是说这些报文真的是以2800字节这个尺寸从网络上传输过来的，而是由于接收端启用了GRO，由接收端的网卡负责把几个小报文“拼接”成了2800字节。</p><p><img src="https://static001.geekbang.org/resource/image/d4/67/d45a675de246069fed22867335a54567.jpg?wh=1834x344" alt="图片"></p><p>所以，如果以后你在Wireshark里看到这种超过1460字节的TCP段长度，不要觉得奇怪了，这只是因为你启用了TSO（发送方向），或者是GRO（接收方向），而不是TCP报文真的就有这么大！</p><p>想要确认你的网卡是否启用了这些特性，可以用ethtool命令，比如下面这样：</p><pre><code class="language-plain">$ ethtool -k enp0s3 | grep offload
tcp-segmentation-offload: on
generic-segmentation-offload: on
generic-receive-offload: on
large-receive-offload: off [fixed]
rx-vlan-offload: on
tx-vlan-offload: on [fixed]
l2-fwd-offload: off [fixed]
hw-tc-offload: off [fixed]
esp-hw-offload: off [fixed]
esp-tx-csum-hw-offload: off [fixed]
rx-udp_tunnel-port-offload: off [fixed]
tls-hw-tx-offload: off [fixed]
tls-hw-rx-offload: off [fixed]
</code></pre><p>当然，在上面的输出中，你也能看到有好几种别的offload。如果你感兴趣，可以自己搜索研究下，这里就不展开了。</p><p>对了，要想启用或者关闭TSO/GRO，也是用ethtool命令，比如这样：</p><pre><code class="language-plain">$ sudo ethtool -K enp0s3 tso off
$ sudo ethtool -k enp0s3 | grep offload
tcp-segmentation-offload: off
</code></pre><h2>IP分片</h2><p>IP层也有跟TCP分段类似的机制，它就是IP分片。很多人搞不清IP分片和TCP分段的区别，甚至经常混为一谈。事实上，它们是两个在不同层面的分包机制，互不影响。</p><p><strong>在TCP这一层，分段的对象是应用层发给TCP的消息体</strong>（message）。比如应用给TCP协议栈发送了3000字节的消息，那么TCP发现这个消息超过了MSS（常见值为1460），就必须要进行分段，比如可能分成1460，1460，80这三个TCP段。</p><p><img src="https://static001.geekbang.org/resource/image/e6/b4/e6b8bb5cd1721d8b0be961a566fa6db4.jpg?wh=1787x516" alt=""></p><p><strong>在IP这一层，分片的对象是IP包的载荷</strong>，它可以是TCP报文，也可以是UDP报文，还可以是IP层自己的报文比如ICMP。</p><p>为了帮助你理解segmentation和fragmentation的区别，我现在假设一个“奇葩”的场景，也就是MSS为1460字节，而MTU却只有1000字节，那么segmentation和fragmentation将按照如下示意图来工作：</p><p><img src="https://static001.geekbang.org/resource/image/ec/cf/ec7a1c58e680061cfb00e71922c135cf.jpg?wh=1830x1013" alt=""></p><blockquote>
<p>补充：为了方便讨论，我们假设TCP头部就是没有Option扩展的20字节。但实际场景里，很可能MSS小于1460字节，而TCP头部也超过20字节。</p>
</blockquote><p>当然，实际的操作系统不太会做这种自我矛盾的傻事，这是因为它自身会解决好MSS跟MTU的关系，比如一般来说，MSS会自动调整为MTU减去40字节。但是我们如果把视野扩大到局域网，也就是主机再加上网络设备，那么就有可能发生这样的情况：1460字节的TCP分段由这台主机完成，1000字节的IP分片由路径中某台MTU为1000的网络设备完成。</p><p>这里其实也有个隐含的条件，就是主机发出的1500字节的报文，不能设置 <strong>DF（Don’t Fragment）位</strong>，否则它既超过了1000这个路径最小MTU，又不允许分片，那么网络设备只能把它丢弃。</p><p>在Wireshark里，我们可以清楚地看到IP报文的这几个标志位：</p><p><img src="https://static001.geekbang.org/resource/image/ec/77/ec65ebe4f5cf7aa476286aa300904a77.jpg?wh=1268x464" alt="图片"></p><p>现在我们假设主机发出的报文是不带DF位的，那么在这种情况下，这台网络设备会把它切分为一个1000（也就是960+20+20）字节的报文和一个520（也就是500+20）字节的报文。1000字节的IP报文的 <strong>MF位（More Fragment）</strong>会设置为1，表示后续还有更多分片，而520字节的IP报文的MF字段为0。</p><p>这样的话，接收端收到第一个IP报文时发现MF是1，就会等第二个IP报文到达，又因为第二个报文的MF是0，那么结合第二个报文的fragment offset信息（这个报文在分片流中的位置），就把这两个报文重组为一个新的完整的IP报文，然后进入正常处理流程，也就是上报给TCP。</p><p>不过在现实场景里，<strong>IP分片是需要尽量避免的</strong>，原因有很多，主要是因为互联网是一个松散的架构，这就导致路径中的各个环节未必会完全遵照所有的约定。比如你发出了大于PMTU的报文，寄希望于MTU较小的那个网络环节为你做分片，但事实上它可能不做分片，而是直接丢弃，比如下面两种情况：</p><ul>
<li>它考虑到开销等问题，未必做分片，所以直接丢弃。</li>
<li>如果你的报文有DF标志位，那么也是直接丢弃。</li>
</ul><p>即使它帮你做了分片，但因为开销比较大，增加的时延对性能也是一个不利因素。</p><p>另外一个原因是，分片后，TCP报文头部只在第一个IP分片中，后续分片不带TCP头部，那么防火墙就不知道后面这几个报文用的传输层协议是什么，可能判断为有害报文而丢弃。</p><p>总之，为了避免这些麻烦，我们还是不要开启IP分片功能。事实上，Linux默认的配置就是，发出的IP报文都设置了DF位，就是明确告诉每个三层设备：“不要对我的报文做分片，如果超出了你的MTU，那就直接丢弃，好过你慢腾腾地做分片，反而降低了网络性能”。</p><h2>小结</h2><p>这节课，我们通过拆解一个典型的MTU引发的传输问题，学习了MTU和MSS、分段和分片、各种卸载（offload）机制等概念。这里，我帮你再提炼几个要点：</p><ul>
<li>在案例分析的过程中，我们解读了Wireshark里的信息，特别是两次DupAck和两次重传，推导出了问题的根因。这里，你需要了解 <strong>200ms超时重传这个知识点</strong>，这在平时排查重传问题时也经常用到。</li>
<li>借助 <strong>Wireshark的Flow graph</strong>，我们可以更加清晰地看到两端报文的流动过程，这对我们推导问题提供了便利。</li>
<li>如果能稳定重现成功和失败这两种不同场景，那就对我们排查工作提供了极大的便利。我们通过<strong>对比成功和失败两种场景下的不同的抓包文件</strong>，能比较快地定位到问题根因。</li>
<li>如果排查中遇到<strong>有“整数值”出现</strong>，可以重点查一下，一般这跟人为的设置有关系，也有可能就是根因，或者与根因有关。</li>
<li>如果你对网络中间环节（包括LB、网关、防火墙等）有权限，又不想改动两端机器的MTU，那么可以选择在中间环节实施“暗箱操作”，也就是<strong>用iptables规则改动双方的MSS</strong>，从而间接地达到“双方不发送超过MTU的报文”的目的。</li>
<li>我们也学习了如何<strong>用ethtool工具查看offload相关特性</strong>，包括TSO、LRO、GRO等等。同样通过ethtool，我们还可以对这些特性进行启用或者禁用，这为我们的排查和调优工作提供了更大的余地。</li>
</ul><h2>思考题</h2><p>最后再给你留两道思考题：</p><ul>
<li>在LB或者网关上修改MSS，虽然可以减小MSS，从而达到让通信成功这个目的，但是这个方案有没有什么劣势或者不足，也同样需要我们认真考量呢？可以从运维和可用性的角度来思考。</li>
<li>你有没有遇到过MTU引发的问题呢？欢迎你分享到留言区，一同交流。</li>
</ul><h2>附录</h2><p>抓包示例文件：<a href="https://gitee.com/steelvictor/network-analysis/tree/master/08">https://gitee.com/steelvictor/network-analysis/tree/master/08</a></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/21/b8/aca814dd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>江山如画</span>
  </div>
  <div class="_2_QraFYR_0">问题1:<br>从可用性角度分析，通过 iptables 修改 mss 只对 tcp 报文生效，对 udp 报文不生效。由于udp 报文传输时没有协商 mss 的过程，如果发现 udp 负载长度比 mtu 大，会交给网络层分片处理，分片传输途中只要有一个分片丢了，由于 udp 没有反馈给发送端具体是哪个分片丢了的能力，只能重新传整个包，传输效率会变低。<br><br>从运维角度分析，由于修改中间环节某个服务器的 iptables ，对于其它侧是透明的，可能会对定位问题带来困扰。<br><br>问题2:<br>遇到过 mtu 引发的问题。之前手动创建了虚拟网卡，和物理网卡之间做了流量的桥接，发现有些报文在二者之间转发时会被丢掉，分析发现虚拟网卡的 mtu 设置过大，并且报文 DF 位设置为了 1，通过 ifconfig 命令把虚拟网卡的 mtu 改小，报文就可以正常转发了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 回答的很详细，很明显经过了仔细的思考，不是随便写的，不光为你的答案，我更要为你的学习态度点赞！<br>udp这部分，协议要求是udp载荷不要超过512字节，比常见MTU小很多，所以可能这个超出MTU的问题没有tcp那样严重。但是就像你说的，udp丢包的话，相应的容错和重传，协议栈本身是不做的，这就给应用程序提出了要求。<br>从运维角度分析的答案也很好，我也是很看重这一点：如果把配置“藏”在各个地方而其他人不知道，就等于是给其他运维和开发人员“埋雷”，我们应该尽量避免。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-07 19:46:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/vM2NmFZQYmb4kkk8Cba1fGP4bhK9diaeXb2LXFWXJWfOPuibK3aib24qujweqciaxt43btqicSz9gDDlJkUQ12RDfJQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_955506</span>
  </div>
  <div class="_2_QraFYR_0">Flow Graph 展示页的左下角有一个复选框 &quot;limt to display filter&quot;，勾上之后就只展示过滤的内容了，不需要再单独保存展示</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 多谢补充，工具的使用方面有很多小的细节：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-05 19:16:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/28/73/9a/5197bbbd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张靳</span>
  </div>
  <div class="_2_QraFYR_0">开始对[为什么有重复确认（DupAck）]这个小节标志位Dup ACK的两个报文是载荷为688和57的两个报文的确认报文不是很理解，在杨老师指导下豁然开朗，做一下我的理解记录：<br>最开始我对wirshark的符号有误解，我选中Dup ACK报文发现是7号报文（三次握手的ack报文）的重复确认（有两个对勾），我就以为是7号报文的确认报文，这个理解本身有问题，因为ack本身没有ack了，不然就没完没了了。查阅了Dup ACK的定义，当发现丢包或者乱序的时候接收方会收到一些Seq序列号比期望值大的包，每收到这种包就会ack一次期望的Seq值。<br>所以我们知道可能有丢包才会有重复确认，且确认得是对端发过来得报文。那么结合整个tcp流来看，发生重传得报文是没收到的，然后确认了688和57载荷得报文。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 很好，自己想明白了这知识就真的是你的了：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-12 13:46:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek3340</span>
  </div>
  <div class="_2_QraFYR_0">老师，有一点不是很理解：<br>由于 Tunnel 1 比 Tunnel 2 的封装更大一些，所以服务端选择了不同的传输尺寸，一个是 1388，一个是 1348。<br>为啥会有这种选择呢，按照理解，MSS会自动从MTU-40来计算，不太理解为啥中间的IPIP隧道会影响到MSS的分段<br><br><br>经过LB的，1388 + 40(TCPIP) + IP(20) + IP(20) = 1468<br>不经过LB的，1348 + 40(TCPIP) + IP(20) = 1408 <br><br>在本次案例中，以下命令能解决问题，应该如何理解呢？我理解1400应该是调整小了MSS到1400，但是之前的1388也没有超限呀，不理解：<br>iptables -A FORWARD -p tcp --tcp-flags SYN SYN -j TCPMSS --set-mss 1400<br><br><br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 您好，图中tunnel1也是用在gz和bj服务器之间的，并不是仅仅在gz和bj lb之间。<br>第二个问题，mss改为1400是一个示例，并不是说当时这个案例就是用了这个数值。<br>有任何其他疑问也都可以提哈</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-08 11:15:33</div>
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
  <div class="_2_QraFYR_0">杨老师，中间设备不是只是转发作用吗，协商mss也参与吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 中间设备负责转发，但因为IP和TCP报文是不加密的，所以可以在中间环节修改这些转发报文的MSS字段，这就起到了让通信两端都按照修改后的MSS进行传输的效果。<br>理论上说，不光MSS，几乎其他任何IP头部和TCP头部字段都可以被中间设备修改。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-07 13:32:40</div>
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
  <div class="_2_QraFYR_0">包的收发路径不一致的时候,  MSS 协商是不是就失效了?  运营商网络, 如果出现链路调整, 之前协商的MSS 是不是也会可能失效了?  特别是长链接场景, 这种情况, 是不是要预先设置一个比较保守的值?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Linux对MSS会随时间发生变化这一点也是有机制来保护的，比如依赖PMTU报文进行调整。PMTU是网络设备会遵循的协议，当设备收到一个超过其MTU的报文时，一方面会丢弃，一方面会返回一个Destination Unreachable的ICMP消息，具体可以参考RFC 1191：<br>https:&#47;&#47;www.rfc-editor.org&#47;rfc&#47;rfc1191<br><br>由于网络的复杂性，这种ICMP消息经常会被拦截，导致发送端压根不知道自己的报文因为MTU超限而被丢弃了，就会导致各种问题。<br>在已经知道有隧道的情况下，尽量设置合理的相对低的MTU值，是保障网络通信的一个经验。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-03 11:19:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83ep8bWkhQUsZRYaoWhWpTKsoSSx7avG2GlvGXYGrfiaiaur9LkTFWeHnuTvyqQN1W1ibJ5pnamB2a92lQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_cf2028</span>
  </div>
  <div class="_2_QraFYR_0">这个案例确实经典，我也遇到过，中间过了ipsec设备加了一层封装导致报文变长，通过修改mss解决了，通过这个案例才明白icmp不是只有ping才有。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-20 22:13:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/96/47/93838ff7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>青鸟飞鱼</span>
  </div>
  <div class="_2_QraFYR_0">TCP，三次握手时，会相互沟通MSS，MSS是怎么来的呢？经过网络各种设备后，不能保证MSS比MTU小，是不是随时得关注MTU、MSS吗</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-03 12:35:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/f1/d7/a52e390d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Pantheon</span>
  </div>
  <div class="_2_QraFYR_0">每一个字都值得分析,老师的课写的很棒,实战派</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢支持，也希望大家坚持学习和反馈，通过这门课程，提升对网络的理解，和网络排查的水平~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-22 00:02:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/6b/20/004af747.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>志强</span>
  </div>
  <div class="_2_QraFYR_0">老师有几个疑问请教：<br>问题1.&quot;我们选中 575 号报文&quot;下边的图中Dup ACK报文是31号，&quot;为什么是两个重复确认报文呢？我们把视线从 2 个 DupAck 报文往上挪&quot;下边的图中dup ack 就变成了3号，是人为修改的还是怎么回事？<br>问题2.有两次31号报文的dup ack，是因为收到了额外两次17号报文吧，有人肯能会问那为啥看不到17号报文的重传呢，这个我也不太清楚，可能是处理重传的位置在捕获抓包之后吧，要是在客户端抓包就能看到17号报文的重传，请老师指正<br>问题3.无论是握手的ack 还是数据的ack，这个ack是谁给回的，知道是内核给回复，具体哪一层的什么函数处理过后给回复的ack；kcp老师了解吗，是应用层在再给回复还是也是内核给的恢复<br>谢谢老师，期待您的详细解答</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 关于这3个问题：<br>1. 你看到DupAck to从#31变成了#3，是因为截图的问题，#3那个是另外单独保存的抓包文件，所以编号不一样，但是TCP流确实是同一个。这个截图我们刚替换成统一#31的，也感谢你的细心！<br>2. 这两次对#31报文的DupAck，只是因为握手之后的第一个数据报文（也就是1388字节那个）没有到客户端，但后面两个报文到了，所以客户端回复的两次ack号只能停留在握手结束的时候。这不是说客户端收到了两次SYN+ACK。DupAck的意思是：“我需要你那边跟这个ack号等值的seq号的报文”，而不是“我需要你再发一次我刚刚ack过的报文”，这里正好差了一个“身位”，你再想想？<br>3. ack是内核协议栈回复的，你可以从github上git clone一下linux内核代码，然后在IDE里，搜一下TCPHDR_ACK在代码中的位置。大体上说，ACK标志位的设置是一类函数，比如SYN、FIN、RST（不含被动RST）、传输阶段等报文都是在各自不同的函数里面设置了ACK标志位，而它们的发送都会走到tcp_transmit_skb()来完成。<br>kcp没了解，也没用过:(</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-08 09:43:37</div>
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
  <div class="_2_QraFYR_0">MTU我看有的文章说是二层的，不知道到底算那层的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: MTU当然是三层的，属于IP报文的长度，一般是1500字节。<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-07 09:40:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/50/33/9dcd30c4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>斯蒂芬.赵</span>
  </div>
  <div class="_2_QraFYR_0">不明白，如果传入的tcp载荷超过了mtu值，不应该根据mss值自动的分段么</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，不过这里的麻烦是由于很多网络限制，通信两端其实经常无法收到PMTU ICMP消息，也就导致无法及时调整MSS来适配网络状况，也是文中案例的诱因之一。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-11 08:50:59</div>
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
  <div class="_2_QraFYR_0">关于怎么看包是在服务器抓的还是在客户端抓的，还有一个方法，就是看端口号，服务器端都是固定的公认端口，客户端都是随机的高端口。<br><br>另外，请教下老师，TCP MSS有没有类似PMTUD的路径发现机制？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 看端口号可能不行哦，因为无论哪端抓取的，都可以抓取到双向的报文。<br>pmtu机制本身主要就是为了帮助传输层设置合理的mss</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-29 17:41:57</div>
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
  <div class="_2_QraFYR_0">经过 LB 的时候，报文需要做 2 次封装（Tunnel 1 和 Tunnel 2），而绕过 LB 就只要做 1 次封装<br>（只有 Tunnel 1）。另外，由于 Tunnel 1 比 Tunnel 2 的封装更大一些，所以服务端选择了不同的传<br>输尺寸，一个是 1388，一个是 1348。<br><br>杨老师，请教个问题，“由于 Tunnel 1 比 Tunnel 2 的封装更大一些，所以服务端选择了不同的传<br>输尺寸，一个是 1388，一个是 1348。”这句话不理解，还望老师解答</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-05 23:54:56</div>
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
  <div class="_2_QraFYR_0">杨老师，我看了前面Geek3340 这位同学的问题，我也有类似的问题：<br><br>1. 广州到LB 之间应该会商量一个MTU1然后LB 和后端Server 之间也会商量一个MTU2,  造成这个问题的原因是广州到LB 之间的包设置了DF 标记导致LB 在转发到后端server 时候直接丢弃超出MTU2的包吗？<br><br>2. 现在的情况是我们有LB 的权限，所以可以在LB 上用iptables来“暗箱操作”, 但是如果客户端到服务端之间经过的网络设备因为MTU 导致问题，而我们又没有权限那是不是就只剩下在客户端直接降低MTU 了呢？如果答案是Yes , 这又导致一个问题，在现在应用和基础架构分离的趋势下，应用开发人员越来越不关心基础架构，那基础架构维护OS 的人就必须保证提供的vm 或者docker image 提供合适的MTU 配置？写到这里还是觉得不设置DF 让网络协议自己商量MTU 似乎更好一些？但是这样会碰到你说的防火墙不认拆分后的后续包的问题，我这样理解对吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-20 08:24:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/16/2a/68913d36.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>杨震</span>
  </div>
  <div class="_2_QraFYR_0">每节课都要看几遍消化消化，老师写的非常好，图文清晰。期待后续老师更多栏目</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢你的支持：）图解确实是一个很好的教学方法，我会尽量多一些这样的方式，帮助大家学的更好~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-03 08:07:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/a8/60/7abaaa62.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>夜、</span>
  </div>
  <div class="_2_QraFYR_0">好像没有直接说明第一个包为啥没发送成功？  就算超过MTU了不是还会IP分片么</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是否进行IP分片取决于IP报文的DF位，如果是1就不允许分片，如果超过MTU，就丢弃这个报文，同时回复一个PMTU ICMP报文给发送端（但这个ICMP报文也经常丢失）。如果是0，允许分片。<br>在后半段我有介绍分片和分段的联系和区别~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-27 18:14:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/27/ce/58/71ed845f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Dexter</span>
  </div>
  <div class="_2_QraFYR_0">TCP分段的计算公司，那个IP header length和TCP header length是如何的得知的？难道都是按照默认的IP HDR =20, TCP HDR =20吗？这不是很可靠吧。 还有TCP segmentation probe功能开启是不是能够解决案例中的问题？<br>[root@master03 ipv4]# cat tcp_mtu_probing<br>0<br>[root@master03 ipv4]#<br><br><br>问题2：如果启用了TSO或者GRO，为什么经常在抓包中看到TCP segment还是1460?<br><br>问题3： LRO是什么？老师能帮忙解答下吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 您好~<br>1. 你问的是RFC879里面的这个公式吗？SndMaxSegSiz = MIN((MTU - sizeof(TCPHDR) - sizeof(IPHDR)), MSS) 。MSS是每个TCP连接各自维护的，所以是根据这条连接的实际情况来决定大小。比如在握手阶段，已经确定要使用TCP option的timestamp，那么这里的TCPHDR就不再是20字节了，而是更大，得出的MSS就更小。<br>mtu_probing没有在这个案例里使用，遗憾的是现场早就没有了，要不然可以试试。<br>2. 启用了TSO或者GRO，还是看到1460？这我倒没注意，一般启用以后，常见的是2920、2800或者更大的值。你依然看到1460，有一个可能性是网卡并没有严格去执行offload。<br>3. LRO是Large Receive Offload，跟GRO类似。LRO&#47;GRO启用后，tcpdump抓取到的接收方向的报文就是合并过的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-26 20:59:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/27/ce/58/71ed845f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Dexter</span>
  </div>
  <div class="_2_QraFYR_0">iptables -A FORWARD -p tcp --tcp-flags SYN SYN -j TCPMSS --set-mss 1400   --- 文中说是在nat表，不过这个command没有指定nat表，而且nat表中应该没有forward chain</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯这里有误，我更正一下，是默认（没有指定-t）的filter表，谢谢：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-26 20:47:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/e4/79/0f0114ba.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>taochao_zs</span>
  </div>
  <div class="_2_QraFYR_0">老师，你的wireshark是什么版本的，为啥3.4.6版本没法再显示列这里添加ttl的字段。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 应该不是版本限制。你可以参考第6讲如何设置自定义列，简单来说就是打开任意报文的IP详情，找到TTL字段，右单击选择apply as column</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-09 09:43:21</div>
  </div>
</div>
</div>
</li>
</ul>