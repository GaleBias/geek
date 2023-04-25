<audio title="10 _ 窗口：TCP Window Full会影响传输效率吗？" src="https://static001.geekbang.org/resource/audio/6c/3b/6c8d8f26b717d997bd983c8efb59fc3b.mp3" controls="controls"></audio> 
<p>你好，我是胜辉。</p><p>有时候，不少知识点在过段时间重新回看的时候，又会有新的体会和发现。比如在<a href="https://time.geekbang.org/column/article/484667">第8讲</a>里，我们回顾了一个MTU造成传输失败的案例，虽然整个排查过程的步骤不算很多，但也算是TCP传输问题的一个缩影了。尤其是其中那个失败的TCP流中的一些现象，比如客户端发出的重复确认（DupAck），还有服务端启动的超时重传，都值得我们继续深挖，所以我会在后续的课程里继续这个话题。</p><p>然后在上节课里，我们还探讨了传输速度的相关知识，也初步学习了窗口的概念。最后，我们终于推导出了TCP传输的核心公式：速度=窗口/往返时间。这个公式，对于我们理解传输本质和排查传输问题，都有很强的指导意义。</p><p>然而，如果你足够细心的话，其实可能会对上节课里的细节有一些疑问，比如：既然接收窗口满了，那为什么当时没有看到TCP Window Full这种提示呢？</p><p>其实，我这边也有不少内容按住没有展开，包括核心公式的理解，我们在这节课里将有一个新的认识。另外，我也将带你继续挖掘窗口这个细分领域，这样你以后遇到跟窗口相关的问题，就知道如何破解了。</p><h2>案例：TCP Window Full是导致异地拷贝速度低的原因吗？</h2><p>也是在公有云服务的时候，有个客户有这么一个需求，就是要把文件从北京机房拷贝到上海机房。但是他们发现传输速度比较慢，就做了抓包。在查看抓包文件的时候，发现Wireshark有很多<strong>TCP Window Full</strong>这样的提示，不明白这些是否跟速度慢有关系，于是找我们来协助分析。</p><!-- [[[read_end]]] --><h3>解读Expert Information</h3><p>我们先要了解一下抓包文件的整体状况。怎么看呢？当然是看Expert Information了：</p><p><img src="https://static001.geekbang.org/resource/image/72/8c/7285f181c54e6f464836e8c44f43c88c.jpg?wh=1818x234" alt="图片"></p><p>它确实提醒我们，有69个Warning级别报文，它们的问题是TCP Window specified by the receiver is now completely Full。在展开进一步排查之前，我们先对这个信息做一下解读。</p><ul>
<li>TCP Window：在上节课里，我介绍了TCP的三种窗口，分别是发送窗口、接收窗口、拥塞窗口。那么这里说的是哪个窗口呢？一般说到TCP Window，<strong>如果没有特别指明，就是指接收窗口</strong>。</li>
<li>specified by the receiver：这也很明确，这个窗口是接收方的，其实就是佐证了这个窗口就是接收窗口。</li>
<li>is now completely Full：窗口满了，这又怎么理解呢？你还记得在上节课里学过的在途数据，也就是Bytes in flight吗？<strong>当在途数据的大小等于接收窗口的大小时，这个窗口就是“满了”</strong>。</li>
</ul><p>好了，这个信息解读完毕，一句话说就是：发生了69次在途数据等于接收窗口的情况。</p><p>接下来我们看看TCP Window Full具体是个什么样子。比如我们选中224号报文，主界面也自动定位到了这个报文：</p><p><img src="https://static001.geekbang.org/resource/image/49/4f/4957d1a108c78d3f73eee9b85213864f.jpg?wh=1920x960" alt="图片"></p><p>我们可以看到，除了224报文，也确实有很多其他报文也报了TCP Window Full的警告信息。</p><h3>解读TCP Window Full</h3><p>TCP Window Full这个信息非常直接明了，就是说“接收窗口满了”。不过，你可别以为这个信息是TCP报文里的某个字段。其实，它只是Wireshark通过分析得出的信息。你有没有注意到，TCP Window Full前后是有方括号的。一般来说，<strong>Wireshark自己分析得到的信息，都会用方括号括起来</strong>，而TCP报文本身的字段，是不会带这种方括号的。我们来看一个截图：</p><p><img src="https://static001.geekbang.org/resource/image/b7/a5/b76369d4ef7fe6f9dd49816247351fa5.jpg?wh=1630x1062" alt="图片"></p><p>上面是224号报文的TCP详情，里面有不少信息也带上了方括号，比如其中的 [Bytes in flight: 112000]，这也是解读出来的，而且它跟Window Full关系很大。</p><p>前面提到过，在途数据（或者叫在途字节数，Bytes in flight）等于接收窗口大小的时候，Wireshark就会解读为TCP Window Full了。不过，如果你在上图中找一下Window size，会发现它是19200，而不是Bytes in flight的112000，这又是为什么呢？</p><p><img src="https://static001.geekbang.org/resource/image/82/98/829ee4eda487862ed761cd0bafff8d98.jpg?wh=674x396" alt="图片"></p><p>这是因为，我们把发送方和接收方的接收窗口搞混啦。这里你需要搞清楚：如果说在途数据的发送方是A，接收方是B，那么这里Window Full的窗口，<strong>是B的接收窗口</strong>，而不是A的接收窗口。上图是A的报文，自然没有我们要找的B的接收窗口信息了，那怎么找到B的接收窗口呢？</p><p>因为这次通信是SCP文件传输，那么A就是客户端，它的端口是38979；B就是服务端，它的端口是22。我们的具体做法是：在抓包文件里，找到B（也就是源端口为22）的报文，而且应该是这个TCP Window Full报文之前的最近的一个，在这个报文里就有B的最近的接收窗口值。</p><p>单纯用文字，可能未必容易理解，我给你画了一张示意图：</p><p><img src="https://static001.geekbang.org/resource/image/2e/95/2e594ee968a7a62b307e5f22c08c4095.jpg?wh=2000x1125" alt=""></p><p>上图中，我还是用A指代发送端，用B指代接收端。当A的在途数据跟B的接收窗口大小相等时，Wireshark就会判断出，这个接收窗口满了，这意味着：A无法再从自己的发送缓冲区把数据发送出来了。只有当B回复ACK，确认了n字节的数据后，A才有可能发送最多n字节的数据（如果缓冲区有足够多的待发数据的话）。</p><p>让我们回到Wireshark窗口，找到离这个TCP Window Full最近的，从源端口22发送来的报文。我们发现，它就是下图这个222号报文：</p><p><img src="https://static001.geekbang.org/resource/image/30/60/30d13a6c4e4730742fe1bed79b6b0a60.jpg?wh=1836x1404" alt="图片"></p><p>可以看到，这个报文的接收窗口就是112000，正好等于前面224号报文的Bytes in flight的112000字节。所以，我把前面的示意图改进一下供你参考。这里面的信息比较多，建议你耐心多花几分钟时间来充分理解其中的机制：</p><p><img src="https://static001.geekbang.org/resource/image/84/0f/84453f815b643f6ac29062a3b9ccf90f.jpg?wh=2000x1125" alt=""></p><p>整个过程是这样的：</p><ul>
<li>B发送了报文222给A，其中带有B自己的接收窗口112000字节。由于这是一个纯的确认报文，所以没有TCP载荷，也没有在途数据。</li>
<li>报文抵达A端，进入A的接收缓冲区。</li>
<li>A从222号报文中得知，B现在的接收窗口是112000字节，由于发送缓冲区有足够多的待发的数据，A选择用满这个接收窗口，也就是连续发送112000字节。</li>
<li>A把这112000字节的数据发送出来，成为报文224，其中还带有A自己的接收窗口值19200字节，不过，由于这次主要是A向B传送数据，所以B发给A的基本都是纯确认报文，这些报文的载荷都是0。极端情况下，即使A的接收窗口为0，只要B回复的报文没有载荷，它们也是可以持续通信的。</li>
<li>224报文抵达B端，正好填满B的接收窗口112000字节。Wireshark分别从222报文中读取到B的接收窗口值，从224报文中读取到在途字节数，由于两者相等，所以Wiresahrk提示TCP Window Full。而这个信息是被Wiresahrk展示在224报文中的。</li>
</ul><h2>自己验证TCP Window Full</h2><p>对于在途数据，既然Wireshark可以解读出来，那只要理解了TCP的原理，我们同样可以自己来计算，这不仅可以考查我们对TCP知识的掌握程度，同时对日常排查也有帮助。有这么多好处，你是不是跃跃欲试了呢？不过在开始之前，我们要先学习一个新的概念。</p><h3>下个序列号</h3><p>下个序列号，也就是<strong>Next Sequence Number</strong>，缩写是<strong>NextSeq</strong>。它是指当前TCP段的结尾字节的位置，但不包含这个结尾字节本身。很显然，下个序列号的值就是当前序列号加上当前TCP段的长度，也就是<strong>NextSeq = Seq + Len</strong>。</p><p>这也不难理解，因为TCP字节流是连续的，那么既然Seq + Len是这个报文的数据截止点，自然也是下一个报文的起始点，你可以参考这个示意图：</p><p><img src="https://static001.geekbang.org/resource/image/f7/bf/f7864509fd7ff569ebbfa54e14a9e6bf.jpg?wh=2000x910" alt=""></p><p>在Wireshark里，我们也可以找到NextSeq这个解读值，比如下图这样：</p><p><img src="https://static001.geekbang.org/resource/image/bd/7e/bd1093a136119ebf4166989e64b3637e.jpg?wh=756x201" alt="图片"></p><p>明白了NextSeq，我们来看如何手工验证TCP Window Full。比如，还是分析224号报文的这次TCP Window Full，我们可以这么做，来验证一下在途数据是否真的是112000字节。</p><p><img src="https://static001.geekbang.org/resource/image/0e/28/0ec8c879acb7526bbff8eb9650cyy928.jpg?wh=1558x550" alt="图片"></p><p>首先，跟上面的步骤类似，我们要找到222号报文。在这个报文里，服务端（源端口22）告诉客户端：“我确认你发送来的198854字节的数据”。我们先把这个数字记为X。</p><p>然后，我们查看224号报文里，客户端发送的数据到了哪个位置：</p><p><img src="https://static001.geekbang.org/resource/image/22/2e/22f94b3933fb841705b28d11a3106b2e.jpg?wh=1552x984" alt="图片"></p><p>我们可以在224号报文的TCP详情页，看到Next sequence number: 310854，而这个数字，就是客户端发送的数据的最新的位置。我们把这个数字记为Y。当然，你也可以像前面说的那样，把Seq和Len加起来，也就是308054 + 2800，得到的自然也是310854。</p><p>最后，我们做一个最简单的减法：Y - X = 310854 - 198854 = 112000！这正是前面说的在途数据的大小。</p><p>恭喜你，你已经学会了如何手工计算在途数据的方法，这也意味着你对TCP的了解又更深入了一点。你可以这么来总结计算在途数据的方法：</p><blockquote>
<p><strong>Bytes_in_flight = latest_nextSeq - latest_ack_from_receiver</strong></p>
</blockquote><p>不过，你会不会觉得，虽然这个计算方法对理解窗口有帮助，但是既然Wireshark会给我们提示，那这种计算也主要是自我练习而已，应该不会真的用得上吧？</p><p>这还真不好说。因为，Wireshark在不少场景下并不会给你提示。比如，在接收窗口接近满但又不是完全满的时候，哪怕是离窗口满只差1个字节，Wireshark也不会提示TCP Window Full了。但是，在途数据都已经逼近接收窗口的99.9%了，你还觉得这个肯定没有问题，或者一定没有隐患吗？</p><p>要知道，这种临界状况也很可能跟问题根因有关。那么你掌握了这个方法，就可以把排查做得更彻底了。或者，如果你想预防性能瓶颈，那么提前找到这种窗口临界满的状况，也是有益的。</p><p>到这里，我们可以回答开始时候的问题了：为什么上节课里，没有看到TCP Window Full这种提示呢？我们看一下当时的报文状况吧。</p><p>在22:30:39.067477，接收端的确认号为7105632：</p><p><img src="https://static001.geekbang.org/resource/image/95/15/95c7e978090178984b6cb27355021e15.jpg?wh=896x61" alt="图片"></p><p>然后在22:30:39.209712，接收端的确认号为7169872：</p><p><img src="https://static001.geekbang.org/resource/image/4a/77/4ae7928be673fd6d2c43c7289aed0077.jpg?wh=901x59" alt="图片"></p><p>这两个报文的时间跨度正好是141ms左右，也就是这次传输里面的<strong>往返时间</strong>。在这个往返时间里，接收端确认了多少数据呢？是7169872-7105632=64240，也就是64KB。这个就是99.9%逼近TCP Window Full了，但是因为还差小几十个字节，所以Wireshark并没有提示TCP Window Full！</p><p>你可能还想追问：那为什么不把这剩余的0.1%的窗口“榨干”，非要留一点呢？我们看一下当时的接收窗口和在途数据的具体情况，就以上面选择的5864报文附近为例：</p><p><img src="https://static001.geekbang.org/resource/image/14/d7/1439256885a585cebc96a139e41495d7.jpg?wh=900x91" alt="图片"></p><p>接收端（源端口为22）的接收窗口为65728，发送端（源端口为59159）的在途数据为65700，两者相差只有28字节。对于发送端来说，没有必要为了这区区28字节再发送一个小报文了，等接收窗口空余出多一点的空间后再动身不迟。</p><blockquote>
<p>如果你还没学习过上节课的内容，可能会对这些信息感到疑惑，建议先去把上节课学完，再来学习这一讲，效果更好。</p>
</blockquote><h2>TCP Window Full对传输的影响</h2><p>好了，现在我们已经对TCP Window Full做了充分的分析，而且也明白了：这就是<strong>接收端的接收窗口小于发送端的发送能力而出现的状况</strong>。我们也很容易得出推论：瓶颈在接收端，<strong>TCP Window Full也确实会影响传输速度</strong>。</p><p>春节刚过，你可能对高速公路上的状况也感受深刻吧！很多路段出现了堵车，这就相当于TCP Window Full，更多的车辆上不了高速了，只好堵在外面。如果高速公路的路更宽、车速更快，那么就相当于接收窗口变得更大，车辆就能进更多，也就相当于Bytes in flight更大了。这么说来，TCP流量控制和高速车流控制这两个领域也有不少共通之处，说不定双方都互有借鉴呢。</p><p>回到客户这次的案例。我们看看，这次的传输速度是多少呢？在上节课里，我介绍了在Wireshark里查看TCP传输速度的两种方法。比如，我们现在用I/O Graph来看一下：</p><p><img src="https://static001.geekbang.org/resource/image/50/5f/50bb15443878c7458035155f5a19425f.jpg?wh=1920x1287" alt="图片"></p><blockquote>
<p>补充：如果你的I/O Graph显示的不是这种图，那需要像图中这样：</p>
<ul>
<li>选中All Bytes指标；</li>
<li>Y轴的单位选为Bytes。</li>
</ul>
</blockquote><p>这个图不能说不对，但柱子比较粗，看起来不是很精确。这是因为，它默认是以1秒为间隔而计算的速度。但是TCP传输中途，很可能每过几毫秒都有所变化，所以，如果我们要看更加精细的图，可以调整一下粒度，把Interval从1sec改为100ms，看看会怎么样：</p><p><img src="https://static001.geekbang.org/resource/image/91/e1/91b410903c0b1f35fa0b88666ae012e1.jpg?wh=308x328" alt="图片"></p><p><img src="https://static001.geekbang.org/resource/image/55/5b/5575a289a1aba0d303a0be0db968175b.jpg?wh=1920x1284" alt="图片"></p><p>这样看起来精确了很多。我已经把这个抓包文件上传到<a href="https://gitee.com/steelvictor/network-analysis/tree/master/10">Gitee</a>，你可以下载后，依照这里的步骤，把我做过的排查工作也演练一遍，这对你掌握课程知识是很有帮助的。</p><blockquote>
<p>补充：如果你用Wireshark查看到的I/O Graph跟我这里的不同，那可以对比一下Interval和SMA period这两个配置是否跟图中的一致。</p>
</blockquote><p>这里有个小的注意点：因为我们选择的间隔是100ms，所以Y轴的数字就是Bytes/100ms，换算成Bytes/s的话，要把Y轴的数字再<strong>乘以10</strong>。从图上看，在一开始的8秒几乎没有数据传输发生，从第8秒开始速度上到了400KB/s（就是图上的40K*10）左右，一直到结束都大致维持在300~400KB/s这个区间里。</p><h2>继续深挖窗口</h2><p>一般来说，接收窗口、拥塞窗口、发送窗口，这些都不是一上来就是一个很大的值的，而是跟汽车起步阶段类似，逐步跑起来的。那么这就产生了一个很有意思的话题：这些窗口之间都是怎么协调的呢？直观上感觉，无论哪个更快了，另外两个就要受影响。</p><p>我们很容易理解，假设起始值相同，如果接收窗口增长的速度小于拥塞窗口的增长速度，那么接收窗口就成了瓶颈；反过来说，拥塞窗口增速更小，那么它就成了瓶颈。</p><p>当接收窗口成为瓶颈的时候，很容易就出现这里的TCP Window Full的现象。不过，我们这么多讨论都是基于文字，如果有更加直观的方式，让我们理解这个现象就更好了。这里，我们就可以再学一个Wireshark的小工具：TCP Stream Graphs里面的 <strong>Window Scaling</strong>。</p><p>我们还是打开Wireshark的Statistics下拉菜单，找到TCP Stream Graphs，在二级菜单中，选择Window Scaling：</p><p><img src="https://static001.geekbang.org/resource/image/cf/e4/cfa2f41e1e0a52f9yyd4d226c441a4e4.jpg?wh=462x552" alt="图片"></p><p>这时候就能看到Windows Scaling的趋势图了：</p><p><img src="https://static001.geekbang.org/resource/image/0c/82/0cda89fe0232b8515007fe1fc5e7aa82.jpg?wh=922x817" alt="图片"></p><p>我就直接给你把关键信息标注出来了。这里主要是两个关键点。</p><ul>
<li><strong>数据流的方向要找对</strong>：比如这次传输是从客户端向SSH服务端发送数据，所以要确认这是从一个高端口向22端口发送数据的流向。如果搞反了，那图就变成了SSH服务端回复的ACK报文了，不是你要分析的传输速度了。</li>
<li><strong>定位TCP Window Full</strong>：在这里，Receive Window是“阶梯”式的，每次变化后会保持在一个“平台”一小段时间，那么这时候Bytes Out（发送的数据，也就是Bytes in flight）就有可能触及这个“平台”，每次真的碰上的时候，就是一次TCP Window Full。</li>
</ul><p>我们可以看一个例子。图中的蓝线代表Bytes Out，绿线代表Receive Window。你可以像我这样，在这几个“平台”区域，找到蓝线和绿线的汇合点，然后在这些点上点击鼠标左键，就能定位到TCP Window Full事件了。</p><p><img src="https://static001.geekbang.org/resource/image/37/3e/37cb41aa86cce8a437c13db7374d2a3e.jpg?wh=832x573" alt="图片"></p><p>上图中，我用鼠标放大了一个“平台”，然后选中了一个Receive Window和Bytes Out重合的点，它是1200号报文，主窗口也自动定位到了这个报文，果然它也是一次TCP Window Full。</p><h2>验证传输公式</h2><p>在上节课里，我们推导出了TCP传输的核心公式：<strong>速度=窗口/往返时间</strong>。既然当前案例里TCP Window Full的时候，Bytes in flight跟接收窗口相等，那么在这个公式里的窗口，是否就是Bytes in flight呢？我们来验证一下。</p><p>还是在Wireshark窗口里，我们要添加这么几个自定义列，以便进行数据比对：</p><ul>
<li>Acknowledgement Number：确认号</li>
<li>Next Sequence Number：下个序列号</li>
<li>Caculated Window Size：计算后的接收窗口</li>
<li>Bytes in flight：在途字节数</li>
</ul><blockquote>
<p>补充：如果对怎么添加自定义列还不清楚，可以复习<a href="https://time.geekbang.org/column/article/481042">第5讲</a>中的添加TTL自定义列的部分。</p>
</blockquote><p>另外，因为我们要集中检查发送端的Bytes in flight，就需要把源端口38979的报文过滤出来，这样就不会被另一个方向的报文给干扰了。</p><p><img src="https://static001.geekbang.org/resource/image/eb/52/ebf76c39e874a4350f8423485f0acd52.jpg?wh=1231x315" alt="图片"></p><p>在这里，我们看到的Bytes in flight是112000字节左右。从右边滚动条的位置来看，这是在传输过程的初期。让我们滚动到中间和后期，看看这些在途字节数是多少：</p><p><img src="https://static001.geekbang.org/resource/image/56/eb/5633292b85320c4a309d67f06c94b0eb.jpg?wh=1152x315" alt="图片"></p><p>中期这里的Calculated Window Size明显增大了，到了445312字节。再看看后半程：</p><p><img src="https://static001.geekbang.org/resource/image/55/d7/55cecd4faa33088df661d5058df379d7.jpg?wh=1158x314" alt="图片"></p><p>最后阶段已经达到863800字节。综合这三个阶段来看，折中值差不多在400KB左右，我们把它除以RTT 0.029秒，得到的是400KB/0.029s=13790KB/s。显然，这个数值远超过前面I/O Graph里看到的300~400KB/s。这是怎么回事呢？难道我们的核心公式是错的吗？</p><p>不知道你有没有考虑到这个问题：Bytes in flight是指真的一直在网络上两头不着吗？一般来说，数据到了接收端，接收端就发送ACK确认这部分数据，然后TCP Window就往下降了。比如ACK 300字节，那么TCP Window就又空出来300字节，也就是发送端又可以新发送300字节了。</p><p><img src="https://static001.geekbang.org/resource/image/09/a6/097c2ccce8c3316137f4408yy24d95a6.jpg?wh=2000x1125" alt=""></p><p>像图上这种情况：</p><ul>
<li>B通知A：“我的接收窗口是1000”；</li>
<li>A向B发送了1000字节，此时B的接收窗口满；</li>
<li>B向A确认了300字节的数据；</li>
<li>A的在途字节数也从1000字节变成700字节，因为刚刚有300字节被B确认了。</li>
</ul><p>图中的t1到t4表示时间点。t2到t4就是一次往返的时间，在这个往返时间内，被传输的数据是1000字节吗？不是。因为被确认的只有300字节，所以传输完的也只有这300字节，速度也就不是1000/RTT，而是300/RTT！我们可以把核心公式做一下改进，变成下面这个：</p><blockquote>
<p><strong>velocity = acked_data/RTT</strong></p>
</blockquote><p>我们再用改进后的公式来计算这次的速度。我们可以选择传输中间偏后面一点的报文来做分析。比如下图中，我们选择1337号和1357号报文为起始和截止点，计算NextSeq的差值，还有时间的差值，然后两个差值相除。</p><p><img src="https://static001.geekbang.org/resource/image/d7/ec/d7174698a1e5c56357f6bd559bbb83ec.jpg?wh=978x289" alt="图片"></p><p>33600/0.094 = <strong>357KB/s</strong>。是不是很接近I/O Graph的值了？看来这样计算才是正确的！</p><p>那为什么在上节课里我们就可以用 <strong>窗口/往返时间</strong> 来计算速度，而且数值也很准呢？而这种方法用到这里就完全不对呢？</p><p>这是因为上节课的案例，在途数据一旦到了接收端，都被及时确认了。而当前这个案例里面并没有这样。也就是说，这次的案例，出现了“滞留”现象。</p><p><img src="https://static001.geekbang.org/resource/image/4e/e0/4e02af20ef67d67490ba5b2c00bb7de0.jpg?wh=1000x512" alt="图片"></p><p>还是1337到1357号报文，我们去掉了过滤器 <code>tcp.srcport eq 38979</code>，这样就展示了双向报文。可以看到，服务端（B端）在这段时间内，只确认了22400字节（1495254 - 1472854），而同样时刻的在途数据，却一直维持在一个比较高的数字，在660KB上下。所以，<strong>真正完成了传输的数据量，是前者22400B，而不是“虚浮”的660KB</strong>。</p><p>那你可能又要问了：既然已经确认了22400字节，为什么客户端的在途字节数还是没有变化呢？</p><p>这是因为，客户端被确认了22400字节的数据，马上又把这个尺寸的数据发送出去了，事实上就<strong>维持了这个在途字节数的尺寸</strong>。</p><p>我可以再做一个比喻帮助你理解这个现象。我们如果去银行的一个窗口（这可不是TCP窗口）排队办业务，现在排队人数为10人，相当于Bytes in flight为10。每分钟都有一个人能完成业务办理，原以为队列会减小为9人，结果每当有一个人出来，保安就喊：“下一个！”于是就立刻又补进来一个人，所以队伍还是维持在10人这么长。</p><p>那么，窗口的业务员的办理速度是多少呢？显然不是10人/分钟，而是1人/分钟了。这样是不是理解起来容易多了？而上节课的情况，相当于这里的“每次就处理一个人”，所以处理速度就是1人/分钟，也就可以用“速度=窗口/往返时间”来计算了。</p><h2>小结</h2><p>这节课，我们集中讨论了TCP传输中的窗口相关的知识，特别是围绕TCP Window Full这个Wiresahrk中比较常见的信息，展开了深入的讨论。你需要重点掌握以下这些知识点：</p><ul>
<li>Wireshark报告TCP Window Full是因为，一端的在途数据跟另一端的接收窗口相等。</li>
<li>TCP的下个序列号（Next Sequence Number）等于序列号和段长度之和，即<strong>NextSeq = Seq + Len</strong>。</li>
<li>我们可以自己人工计算出在途字节数，公式是<strong>Bytes_in_flight = latest_nextSeq - latest_ack_from_receiver</strong>。</li>
<li>人工计算在途字节数的方法是：
<ul>
<li>找到最近一次发送出去的报文的NextSeq，记为X；</li>
<li>找到在这次发送之前收到的最近的ACK，记录它的ACK，记为Y；</li>
<li>X－Y，得到在途字节数。</li>
</ul>
</li>
<li>另外我们也知道了，TCP Window Full确实会影响到传输速度。</li>
</ul><p>除了技术知识点，我也带你学习了下面这些Wireshark工具使用技巧：</p><ul>
<li>在Statistics下拉菜单下的I/O Graph工具，可以直观地展示传输速度图。</li>
<li>同是Statistics菜单下的TCP Stream Graphs的Window Scaling工具，可以直观的展示TCP Window Full历史曲线图。</li>
</ul><p>最后，我们还发现这个案例里的接收端有“数据滞留”的现象，这就导致“速度=窗口/往返时间”的公式遇到了挑战，而我们可以进一步优化为新的公式：<strong>速度=确认数据/往返时间</strong>。这个改进后的公式，可以兼容这种有“数据滞留”现象的传输场景。</p><h2>思考题</h2><p>给你留两道思考题：</p><ul>
<li>TCP的序列号和确认号，最大可以到多少？</li>
<li>接收端只确认部分数据，导致了“数据滞留”现象，这个现象背后的原因可能是什么呢？</li>
</ul><p>欢迎你把答案分享到留言区，我们一同交流和进步。</p><h2>附录</h2><p>抓包示例文件：<a href="https://gitee.com/steelvictor/network-analysis/tree/master/10">https://gitee.com/steelvictor/network-analysis/tree/master/10</a></p>
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
  <div class="_2_QraFYR_0">问题1:<br>序列号和确认号都占用 32bit，空间范围从 [0, 2**32-1], 最大是 2**32-1<br><br>问题2:<br>感觉接收端确认的这部分数据，就是用户进程从内核接收缓冲区中读取的数据。出现数据滞留，说明写接收缓冲区的速度大于读接收缓冲区的速度，基于此，可以从接收缓冲区的大小(接收窗口的大小)，写接收缓冲区的速度，读接收缓冲区的速度三方面考虑。<br><br>从接收缓冲区的自身的大小考虑，出现数据滞留，可能是接收窗口太小了，比如 &#47;proc&#47;sys&#47;net&#47;ipv4&#47;tcp_window_scaling 设置为了0，导致窗口上限就是 65535。<br><br>从读接收缓冲区的角度考虑，出现数据滞留，可能是用户进程读取速度太慢了，用户进程读取的时候也可能一次也读区不完，需要读取多次，这里取决于用户进程的代码逻辑，极端情况下，比如每调用一次 recv，就 sleep 一段时间。<br><br>写接收缓冲区的速度由tcp协议栈维护，取决于接收窗口大小和读接收缓冲区的速度。<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 正确。而且你的答案非常细致了，说明你经过了深入的思考，很棒：）带着问题去学习，一直是非常高效的学习方法，比如这个问题里面，涉及缓冲区的概念、TCP窗口的概念等等。每次这样的思考，就让我们的理解，更加深入了一点~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-11 18:17:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/1f/51/5e2c484e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>静静同学</span>
  </div>
  <div class="_2_QraFYR_0">老师，我们工作中有同事遇到了tcp window zero，我当时没参与这个任务，没去看具体的抓包数据，但好像也是接收端窗口满了的原因。请问window zero 和window full的区别是什么呢？是视角不同吗？zero是接收端视角，full是发送端视角？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你好，这两个信息都是关于window的，确实很容易搞混，我这里解释一下。其实跟视角没关系。<br>tcp window zero是指，这个tcp报文的window字段明确就是0，也就意味着这个报文的发送方的接收缓冲区已经变成0了，另一方就会停止发送数据。当然，为了避免无限等待，另一方会有探测机制，定制发送一个零载荷的报文，让window zero这一端回复报文，从这个回复报文里读取到最新的window值。<br>tcp window full是指，这个报文发送后，另一端的接收窗口就满了。我举个例子，现在A和B在通信，A的一个报文被wireshark标记为tcp window full，指的是A的未被确认的数据量（也就是在途字节数）跟B的接收窗口相等。这意味着A不会发送更多报文（即使B没有发送window zero的报文），因为A知道发送出去的数据量已经“填满”了B的接收缓冲区~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-26 17:10:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/18/81/83b6ade2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>好吃不贵</span>
  </div>
  <div class="_2_QraFYR_0">老师好，请教下，如果是长连接，没法抓到TCP之前的SYN报文了，在抓到的报文里，window size scaling factor字段显示为-1(unknown)，有什么办法解决吗？多谢。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个没有办法，ws系数只在握手包里。不过，你可以对比实际的bytes in flight，间接的推测这个系数，这就不是很准了，但是如果还有tcp window full信息出现，那就可以确定了。你试试看</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-19 20:02:30</div>
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
  <div class="_2_QraFYR_0">第一题：tcp头的序列号是32位，所以最大值是2^32。 如果短时间内发送大量数据，会有序列号回绕的问题。<br>第二题，接收端只确认了部分数据，可能接收程序处理慢，没有及时响应网卡缓冲区的irq，接收了部分数据。抑或网卡多队列导致接收不及时。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 恩差不多。第一题确切说是2^32-1 ：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-11 10:26:25</div>
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
  <div class="_2_QraFYR_0">如果把老师这个课程里的案例都实际操作一遍，相信一定对于网络抓包分析问题达到实操水平的。对于我们部门的分布式防火墙和分布式负载均衡系统的问题排查我也相信用同样的方法可以批量的培训人。<br><br>更难或者更重要的是：怎么才能让更多的人有像老师这样的心和动力去 准备这样的培训呢？人性的善需要被激发，而激发的方法是多样的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 确实是可以实操的，我也推荐大家去gitee上下载这些课件，对照我的分析过程做一遍，会比单纯看文字的收获要大得多。毕竟，抓包分析是实践性很强的技术，不动手是很难学到真本事的。<br>另外关于分享，我觉得每个人都有自己的独特优势，如果一个人本身乐于分享，那么做培训的过程也是令他享受的。此时我似乎明白了“享”字的双关之义，哈哈</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-15 09:57:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/48/8f/b728f820.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>AlohaJack</span>
  </div>
  <div class="_2_QraFYR_0">&quot;不知道你有没有考虑到这个问题：Bytes in flight 是指真的一直在网络上两头不着吗？一般来说，数据到了接收端，接收端就发送 ACK 确认这部分数据，然后 TCP Window 就往下降了。比如 ACK 300 字节，那么 TCP Window 就又空出来 300 字节，也就是发送端又可以新发送 300 字节了。&quot;<br><br>老师，这里对应的图里面，t3时刻回的ack包里面，窗口应该是从0变成了300吧，图里面ack的window=1000，这里不是很直观。<br>t3时刻ack的包里window是300<br>t4时刻A收到了这个ack，才可以继续发送300B<br>图里面改成300可能要直观点，还是我理解有偏差</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果你看一下本课的示例文件，会发现接收端在Window Full前后，receive window确实是保持不变的，对比到这个例子里就是保持在win=1000。不过这个其实并不重要。关键点在于理解bytes in flight的变化。<br>在t2时刻，发送端认为有1000字节的bytes in flight;<br>在t4时刻，由于300字节被确认了，所以发送端认为bytes in flight是1000-300=700字节。<br>也就是在t4-t2这一个RTT内，被确认的数据是300字节，而不是700字节或者1000字节~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-18 16:08:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/4f/ca/3ac60658.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>orange</span>
  </div>
  <div class="_2_QraFYR_0">我们公司之前遇到一个问题，访问特别慢，排查到最后，ss命令发现是 被攻击了，8081端口的 接收队列满了，默认是128</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯新的连接进不来了：）可以调大tcp_max_syn_backlog值看看~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-12 12:22:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/7c/25/70134099.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>许凯</span>
  </div>
  <div class="_2_QraFYR_0">如果高速公路的路更宽、车速更快，那么就相当于接收窗口变得更大，车辆就能进更多，也就相当于 Bytes in flight 更大了。      <br><br>老师这句话是否不妥哈？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢：）你是不是觉得“车速更快”的表述有点多余？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-24 08:30:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/76/f5/e3f5bd8d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>宝仔</span>
  </div>
  <div class="_2_QraFYR_0">老师我想问下，你这个案例里用scp从北京机房拷贝到上海机房，为什么服务端数据确认这么慢，这个你们后来查到是为啥吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 您好，当时我们作为云服务商，帮助排查了云平台网络，确认没有这方面问题，后来客户自己去检查操作系统方面的原因了，更近一步的信息后来就没有沟通了。我这里介绍这个案例主要想让大家学会如何做网络分析，特别是传输速度和窗口、确认等这些关键环节的关系。<br>就那个主机的原因来说，可能有一定的特殊性，而网络排查过程相对来说比较普适，希望对大家平时的工作有帮助：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-15 21:19:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJnugUNWBtcszhJg3Q0hqEMSHftKco2TqCG78blZ3ibjncjZ64NbibGia5l4NB0DUibIq0BCZ03JvkoNA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek__e15575f5b6ec</span>
  </div>
  <div class="_2_QraFYR_0">(2) 有时候发送的时候拆包了 接收端接收的时候  需要完整数据  所以只能等到后续的数据<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯如果有做IP分片，确实要在接收端组装的~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-13 21:19:15</div>
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
  <div class="_2_QraFYR_0">1. 默认最大2的32次方，好像也有一些机制，比如使用timestaps，用来防止序列号环绕，是不是一定程度上也突破了这个最大限制？<br>2 接收端，用户程序处理速度过慢，或者网卡出现故障。<br><br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. 是的，最大是2^32-1。timestamp会起到PAWS的作用，但是不增加序列号取值范围本身。<br>2. 嗯“滞留”或者确认慢的现象，可能是多种原因导致的，这是一个开放式的问题，鼓励大家多思考各种可能的原因，我们经常这样思考的话，遇到问题时候的排查思路就会不知不觉的宽广很多：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-11 22:31:46</div>
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
  <div class="_2_QraFYR_0">2 ** 32 -1</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的～</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-11 10:21:50</div>
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
  <div class="_2_QraFYR_0">从老师的文章中学了不少工具的使用方法，以前只会傻傻的看每个报文。马上就把它给用起来，😄</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 哈哈 好的 多利用工具，效率继续提升</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-11 00:56:52</div>
  </div>
</div>
</div>
</li>
</ul>