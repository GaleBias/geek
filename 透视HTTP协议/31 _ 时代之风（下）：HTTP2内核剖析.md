<audio title="31 _ 时代之风（下）：HTTP2内核剖析" src="https://static001.geekbang.org/resource/audio/1a/ba/1a3b423c943a63c41e476a1693c89cba.mp3" controls="controls"></audio> 
<p>今天我们继续上一讲的话题，深入HTTP/2协议的内部，看看它的实现细节。</p><p><img src="https://static001.geekbang.org/resource/image/89/17/8903a45c632b64c220299d5bc64ef717.png?wh=1142*586" alt=""></p><p>这次实验环境的URI是“/31-1”，我用Wireshark把请求响应的过程抓包存了下来，文件放在GitHub的“wireshark”目录。今天我们就对照着抓包来实地讲解HTTP/2的头部压缩、二进制帧等特性。</p><h2>连接前言</h2><p>由于HTTP/2“事实上”是基于TLS，所以在正式收发数据之前，会有TCP握手和TLS握手，这两个步骤相信你一定已经很熟悉了，所以这里就略过去不再细说。</p><p>TLS握手成功之后，客户端必须要发送一个“<strong>连接前言</strong>”（connection preface），用来确认建立HTTP/2连接。</p><p>这个“连接前言”是标准的HTTP/1请求报文，使用纯文本的ASCII码格式，请求方法是特别注册的一个关键字“PRI”，全文只有24个字节：</p><pre><code>PRI * HTTP/2.0\r\n\r\nSM\r\n\r\n
</code></pre><p>在Wireshark里，HTTP/2的“连接前言”被称为“<strong>Magic</strong>”，意思就是“不可知的魔法”。</p><p>所以，就不要问“为什么会是这样”了，只要服务器收到这个“有魔力的字符串”，就知道客户端在TLS上想要的是HTTP/2协议，而不是其他别的协议，后面就会都使用HTTP/2的数据格式。</p><!-- [[[read_end]]] --><h2>头部压缩</h2><p>确立了连接之后，HTTP/2就开始准备请求报文。</p><p>因为语义上它与HTTP/1兼容，所以报文还是由“Header+Body”构成的，但在请求发送前，必须要用“<strong>HPACK</strong>”算法来压缩头部数据。</p><p>“HPACK”算法是专门为压缩HTTP头部定制的算法，与gzip、zlib等压缩算法不同，它是一个“有状态”的算法，需要客户端和服务器各自维护一份“索引表”，也可以说是“字典”（这有点类似brotli），压缩和解压缩就是查表和更新表的操作。</p><p>为了方便管理和压缩，HTTP/2废除了原有的起始行概念，把起始行里面的请求方法、URI、状态码等统一转换成了头字段的形式，并且给这些“不是头字段的头字段”起了个特别的名字——“<strong>伪头字段</strong>”（pseudo-header fields）。而起始行里的版本号和错误原因短语因为没什么大用，顺便也给废除了。</p><p>为了与“真头字段”区分开来，这些“伪头字段”会在名字前加一个“:”，比如“:authority” “:method” “:status”，分别表示的是域名、请求方法和状态码。</p><p>现在HTTP报文头就简单了，全都是“Key-Value”形式的字段，于是HTTP/2就为一些最常用的头字段定义了一个只读的“<strong>静态表</strong>”（Static Table）。</p><p>下面的这个表格列出了“静态表”的一部分，这样只要查表就可以知道字段名和对应的值，比如数字“2”代表“GET”，数字“8”代表状态码200。</p><p><img src="https://static001.geekbang.org/resource/image/76/0c/769dcf953ddafc4573a0b4c3f0321f0c.png?wh=1142*719" alt=""></p><p>但如果表里只有Key没有Value，或者是自定义字段根本找不到该怎么办呢？</p><p>这就要用到“<strong>动态表</strong>”（Dynamic Table），它添加在静态表后面，结构相同，但会在编码解码的时候随时更新。</p><p>比如说，第一次发送请求时的“user-agent”字段长是一百多个字节，用哈夫曼压缩编码发送之后，客户端和服务器都更新自己的动态表，添加一个新的索引号“65”。那么下一次发送的时候就不用再重复发那么多字节了，只要用一个字节发送编号就好。</p><p><img src="https://static001.geekbang.org/resource/image/5f/6f/5fa90e123c68855140e2b40f4f73c56f.png?wh=1142*357" alt=""></p><p>你可以想象得出来，随着在HTTP/2连接上发送的报文越来越多，两边的“字典”也会越来越丰富，最终每次的头部字段都会变成一两个字节的代码，原来上千字节的头用几十个字节就可以表示了，压缩效果比gzip要好得多。</p><h2>二进制帧</h2><p>头部数据压缩之后，HTTP/2就要把报文拆成二进制的帧准备发送。</p><p>HTTP/2的帧结构有点类似TCP的段或者TLS里的记录，但报头很小，只有9字节，非常地节省（可以对比一下TCP头，它最少是20个字节）。</p><p>二进制的格式也保证了不会有歧义，而且使用位运算能够非常简单高效地解析。</p><p><img src="https://static001.geekbang.org/resource/image/61/e3/615b49f9d13de718a34b9b98359066e3.png?wh=1142*575" alt=""></p><p>帧开头是3个字节的<strong>长度</strong>（但不包括头的9个字节），默认上限是2^14，最大是2^24，也就是说HTTP/2的帧通常不超过16K，最大是16M。</p><p>长度后面的一个字节是<strong>帧类型</strong>，大致可以分成<strong>数据帧</strong>和<strong>控制帧</strong>两类，HEADERS帧和DATA帧属于数据帧，存放的是HTTP报文，而SETTINGS、PING、PRIORITY等则是用来管理流的控制帧。</p><p>HTTP/2总共定义了10种类型的帧，但一个字节可以表示最多256种，所以也允许在标准之外定义其他类型实现功能扩展。这就有点像TLS里扩展协议的意思了，比如Google的gRPC就利用了这个特点，定义了几种自用的新帧类型。</p><p>第5个字节是非常重要的<strong>帧标志</strong>信息，可以保存8个标志位，携带简单的控制信息。常用的标志位有<strong>END_HEADERS</strong>表示头数据结束，相当于HTTP/1里头后的空行（“\r\n”），<strong>END_STREAM</strong>表示单方向数据发送结束（即EOS，End of Stream），相当于HTTP/1里Chunked分块结束标志（“0\r\n\r\n”）。</p><p>报文头里最后4个字节是<strong>流标识符</strong>，也就是帧所属的“流”，接收方使用它就可以从乱序的帧里识别出具有相同流ID的帧序列，按顺序组装起来就实现了虚拟的“流”。</p><p>流标识符虽然有4个字节，但最高位被保留不用，所以只有31位可以使用，也就是说，流标识符的上限是2^31，大约是21亿。</p><p>好了，把二进制头理清楚后，我们来看一下Wireshark抓包的帧实例：</p><p><img src="https://static001.geekbang.org/resource/image/57/03/57b0d1814567e6317c8de1e3c04b7503.png?wh=1142*585" alt=""></p><p>在这个帧里，开头的三个字节是“00010a”，表示数据长度是266字节。</p><p>帧类型是1，表示HEADERS帧，负载（payload）里面存放的是被HPACK算法压缩的头部信息。</p><p>标志位是0x25，转换成二进制有3个位被置1。PRIORITY表示设置了流的优先级，END_HEADERS表示这一个帧就是完整的头数据，END_STREAM表示单方向数据发送结束，后续再不会有数据帧（即请求报文完毕，不会再有DATA帧/Body数据）。</p><p>最后4个字节的流标识符是整数1，表示这是客户端发起的第一个流，后面的响应数据帧也会是这个ID，也就是说在stream[1]里完成这个请求响应。</p><h2>流与多路复用</h2><p>弄清楚了帧结构后我们就来看HTTP/2的流与多路复用，它是HTTP/2最核心的部分。</p><p>在上一讲里我简单介绍了流的概念，不知道你“悟”得怎么样了？这里我再重复一遍：<strong>流是二进制帧的双向传输序列</strong>。</p><p>要搞明白流，关键是要理解帧头里的流ID。</p><p>在HTTP/2连接上，虽然帧是乱序收发的，但只要它们都拥有相同的流ID，就都属于一个流，而且在这个流里帧不是无序的，而是有着严格的先后顺序。</p><p>比如在这次的Wireshark抓包里，就有“0、1、3”一共三个流，实际上就是分配了三个流ID号，把这些帧按编号分组，再排一下队，就成了流。</p><p><img src="https://static001.geekbang.org/resource/image/68/33/688630945be2dd51ca62515ae498db33.png?wh=1142*598" alt=""></p><p>在概念上，一个HTTP/2的流就等同于一个HTTP/1里的“请求-应答”。在HTTP/1里一个“请求-响应”报文来回是一次HTTP通信，在HTTP/2里一个流也承载了相同的功能。</p><p>你还可以对照着TCP来理解。TCP运行在IP之上，其实从MAC层、IP层的角度来看，TCP的“连接”概念也是“虚拟”的。但从功能上看，无论是HTTP/2的流，还是TCP的连接，都是实际存在的，所以你以后大可不必再纠结于流的“虚拟”性，把它当做是一个真实存在的实体来理解就好。</p><p>HTTP/2的流有哪些特点呢？我给你简单列了一下：</p><ol>
<li>流是可并发的，一个HTTP/2连接上可以同时发出多个流传输数据，也就是并发多请求，实现“多路复用”；</li>
<li>客户端和服务器都可以创建流，双方互不干扰；</li>
<li>流是双向的，一个流里面客户端和服务器都可以发送或接收数据帧，也就是一个“请求-应答”来回；</li>
<li>流之间没有固定关系，彼此独立，但流内部的帧是有严格顺序的；</li>
<li>流可以设置优先级，让服务器优先处理，比如先传HTML/CSS，后传图片，优化用户体验；</li>
<li>流ID不能重用，只能顺序递增，客户端发起的ID是奇数，服务器端发起的ID是偶数；</li>
<li>在流上发送“RST_STREAM”帧可以随时终止流，取消接收或发送；</li>
<li>第0号流比较特殊，不能关闭，也不能发送数据帧，只能发送控制帧，用于流量控制。</li>
</ol><p>这里我又画了一张图，把上次的图略改了一下，显示了连接中无序的帧是如何依据流ID重组成流的。</p><p><img src="https://static001.geekbang.org/resource/image/b4/7e/b49595a5a425c0e67d46ee17cc212e7e.png?wh=1142*735" alt=""></p><p>从这些特性中，我们还可以推理出一些深层次的知识点。</p><p>比如说，HTTP/2在一个连接上使用多个流收发数据，那么它本身默认就会是长连接，所以永远不需要“Connection”头字段（keepalive或close）。</p><p>你可以再看一下Wireshark的抓包，里面发送了两个请求“/31-1”和“/favicon.ico”，始终用的是“56095&lt;-&gt;8443”这个连接，对比一下<a href="https://time.geekbang.org/column/article/100502">第8讲</a>，你就能够看出差异了。</p><p>又比如，下载大文件的时候想取消接收，在HTTP/1里只能断开TCP连接重新“三次握手”，成本很高，而在HTTP/2里就可以简单地发送一个“RST_STREAM”中断流，而长连接会继续保持。</p><p>再比如，因为客户端和服务器两端都可以创建流，而流ID有奇数偶数和上限的区分，所以大多数的流ID都会是奇数，而且客户端在一个连接里最多只能发出2^30，也就是10亿个请求。</p><p>所以就要问了：ID用完了该怎么办呢？这个时候可以再发一个控制帧“GOAWAY”，真正关闭TCP连接。</p><h2>流状态转换</h2><p>流很重要，也很复杂。为了更好地描述运行机制，HTTP/2借鉴了TCP，根据帧的标志位实现流状态转换。当然，这些状态也是虚拟的，只是为了辅助理解。</p><p>HTTP/2的流也有一个状态转换图，虽然比TCP要简单一点，但也不那么好懂，所以今天我只画了一个简化的图，对应到一个标准的HTTP“请求-应答”。</p><p><img src="https://static001.geekbang.org/resource/image/d3/b4/d389ac436d8100406a4a488a69563cb4.png?wh=1142*941" alt=""></p><p>最开始的时候流都是“<strong>空闲</strong>”（idle）状态，也就是“不存在”，可以理解成是待分配的“号段资源”。</p><p>当客户端发送HEADERS帧后，有了流ID，流就进入了“<strong>打开</strong>”状态，两端都可以收发数据，然后客户端发送一个带“END_STREAM”标志位的帧，流就进入了“<strong>半关闭</strong>”状态。</p><p>这个“半关闭”状态很重要，意味着客户端的请求数据已经发送完了，需要接受响应数据，而服务器端也知道请求数据接收完毕，之后就要内部处理，再发送响应数据。</p><p>响应数据发完了之后，也要带上“END_STREAM”标志位，表示数据发送完毕，这样流两端就都进入了“<strong>关闭</strong>”状态，流就结束了。</p><p>刚才也说过，流ID不能重用，所以流的生命周期就是HTTP/1里的一次完整的“请求-应答”，流关闭就是一次通信结束。</p><p>下一次再发请求就要开一个新流（而不是新连接），流ID不断增加，直到到达上限，发送“GOAWAY”帧开一个新的TCP连接，流ID就又可以重头计数。</p><p>你再看看这张图，是不是和HTTP/1里的标准“请求-应答”过程很像，只不过这是发生在虚拟的“流”上，而不是实际的TCP连接，又因为流可以并发，所以HTTP/2就可以实现无阻塞的多路复用。</p><h2>小结</h2><p>HTTP/2的内容实在是太多了，为了方便学习，我砍掉了一些特性，比如流的优先级、依赖关系、流量控制等。</p><p>但只要你掌握了今天的这些内容，以后再看RFC文档都不会有难度了。</p><ol>
<li><span class="orange">HTTP/2必须先发送一个“连接前言”字符串，然后才能建立正式连接；</span></li>
<li><span class="orange">HTTP/2废除了起始行，统一使用头字段，在两端维护字段“Key-Value”的索引表，使用“HPACK”算法压缩头部；</span></li>
<li><span class="orange">HTTP/2把报文切分为多种类型的二进制帧，报头里最重要的字段是流标识符，标记帧属于哪个流；</span></li>
<li><span class="orange">流是HTTP/2虚拟的概念，是帧的双向传输序列，相当于HTTP/1里的一次“请求-应答”；</span></li>
<li><span class="orange">在一个HTTP/2连接上可以并发多个流，也就是多个“请求-响应”报文，这就是“多路复用”。</span></li>
</ol><h2>课下作业</h2><ol>
<li>HTTP/2的动态表维护、流状态转换很复杂，你认为HTTP/2还是“无状态”的吗？</li>
<li>HTTP/2的帧最大可以达到16M，你觉得大帧好还是小帧好？</li>
<li>结合这两讲，谈谈HTTP/2是如何解决“队头阻塞”问题的。</li>
</ol><p>欢迎你把自己的学习体会写在留言区，与我和其他同学一起讨论。如果你觉得有所收获，也欢迎把文章分享给你的朋友。</p><p><img src="https://static001.geekbang.org/resource/image/3d/49/3dfab162c427fb3a1fa16494456ae449.png?wh=1769*3648" alt="unpreview"></p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/79/4b/740f91ca.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>-W.LI-</span>
  </div>
  <div class="_2_QraFYR_0">1.还是无状态,流状态只是表示流是否建立，单次请求响应的状态。并非会话级的状态保持<br>2.小帧好，少量多次，万一拥堵重复的的少。假设大帧好，只要分流不用分帧了。<br>3.每一个请求响应都是一个流，流和流之间可以并行，流内的帧还是有序串行。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-07 08:42:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/a5/98/a65ff31a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>djfhchdh</span>
  </div>
  <div class="_2_QraFYR_0">3、首先要明确造成“队头阻塞”的原因，因为http1里的请求和应答是没有序号标识的，导致了无法将乱序的请求和应答关联起来，也就是必须等待起始请求的应答先返回，则后续请求的应答都会延迟，这就是“队头阻塞”，而http2采用了虚拟的“流”，每次的请求应答都会分配同一个流id，而同一个流id里的帧又都是有序的，这样根据流id就可以标识出同一次的请求应答，不用再等待起始请求的应答先返回了，解决了“队头阻塞”</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: http&#47;1里的请求都是排队处理的，所以有队头阻塞。<br><br>http&#47;2的请求是乱序的，彼此不依赖，所以没有队头阻塞。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-07 11:38:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/04/71/0b949a4c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>何用</span>
  </div>
  <div class="_2_QraFYR_0">HTTP&#47;2 底层还是依赖 TCP 传输，没有解决队头阻塞的问题啊，这就是为何 HTTP&#47;3 要基于 UDP 来传输</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对，虽然是部分解决，但对于http&#47;1来说已经是一个很大的进步了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-07 20:48:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/04/71/0b949a4c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>何用</span>
  </div>
  <div class="_2_QraFYR_0">服务端是不是要为每一个客户端都单独维护一份索引表？连接的客户端多了的话内存不就OOM了嘛</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，不过动态表也有淘汰机制，服务器可以自己定制策略，不会过度占用内存。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-07 20:31:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/9a/67/73f384f9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>好好好</span>
  </div>
  <div class="_2_QraFYR_0">老师，我还是无法理解HTTP1.X无法实现多路复用的具体原因。如果我在HTTP1.X版本的一个TCP连接下同时发送多个请求，会发生什么情况呢？造成这个情况的具体原因又是什么呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.http协议要求请求-响应必须一来一回，上一个请求没有处理完，下一个请求是不能发出去的。一个tcp连接上的http请求必然是串行。<br><br>2.管道模式可以顺序发出多个请求，但响应也必须顺序响应。这些都是http&#47;1.1里规定的。<br><br>3.再对比http&#47;2，一个tcp连接里有多个流，每个流就是一个请求，所以多个请求可以并发，“复用”在了一个连接里。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-30 10:04:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/c4/8b/a9fbaea6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>想个昵称好难</span>
  </div>
  <div class="_2_QraFYR_0">老师您好,打扰您实在是抱歉,想请教您一个问题,您在文中说HTTP&#47;2会在两端维护“Key-Value”的索引表,静态表应该是一摸一样的,那动态表俩边一样吗?如果一样的话,同步是比较难做的事情吧,我看RFC文档中是这么写的,”When used for bidirectional communication, such as in HTTP, the encoding and decoding dynamic tables maintained by an endpoint are completely independent, i.e., the request and response dynamic tables are separate.“, 所以我的理解是,动态表在客户端和服务器各自都有俩个表,一个是用来保存客户端发送的message的header,另外一个是保存服务器发送的header, 我看stackoverflow中也是这么写的,https:&#47;&#47;stackoverflow.com&#47;questions&#47;53003333&#47;how-does-headers-keep-sync-in-both-client-and-server-side-in-http-2, 如果我有哪个地方理解错了,麻烦下老师指点一下<br><br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，客户端和服务器维护各自的动态表，收发各一张表，但字典里的内容必须是一致的，否则索引号就对不上了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-07 14:53:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/c4/8b/a9fbaea6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>想个昵称好难</span>
  </div>
  <div class="_2_QraFYR_0">还有一个问题想请教下老师,您之前在《HTTP的前世今生》上有一段回复是说,只要是HTTP&#47;1.1，就都是文本格式，虽然里面的数据可能是二进制，但分隔符还是文本，这些都会 在“进阶篇”里讲, 不过我看到现在还是有点迷惑,所二进制协议和文本协议的区别是什么呢?可以按照stackoverflow中https:&#47;&#47;stackoverflow.com&#47;questions&#47;2645009&#47;binary-protocols-v-text-protocols  的回答来理解吗?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 指的是协议本身的数据格式，而不是负载（payload）的格式。<br><br>你看http&#47;1，请求行、头、body里的分隔符，都是ASCII码。<br><br>而http&#47;2，是二进制帧，用字节、位来表示信息，没有ASCII码。<br><br>你可以把自己想象成协议的解析器，你看到的协议头是什么格式，文本还是二进制。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-07 14:58:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/a5/98/a65ff31a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>djfhchdh</span>
  </div>
  <div class="_2_QraFYR_0">1、http的“无状态”是指对事务处理没有记忆，每个请求之间都是独立的，这与HPACK算法里的动态表、流状态转换是两回事。HPACK算法里维护动态表是用于头部压缩，而流状态转换只是表示一次请求应答里流的状态，都不会记录之前事务的信息</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 说的很好，也可以这么表述“语法上有状态，语义上无状态”。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-07 11:18:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4d/fd/0aa0e39f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>许童童</span>
  </div>
  <div class="_2_QraFYR_0">HTTP&#47;2 的动态表维护、流状态转换很复杂，你认为 HTTP&#47;2 还是“无状态”的吗？<br>还是无状态的，对上层应用来说，动态表维护、流状态转换这些操作对它不可见。<br><br>HTTP&#47;2 的帧最大可以达到 16M，你觉得大帧好还是小帧好？<br>大帧好，应该小帧需要很多额外的头信息，有数据冗余。小帧可以当出差错时，只转输出错的帧，细粒度控制。<br><br>结合这两讲，谈谈 HTTP&#47;2 是如何解决“队头阻塞”问题的。<br>因为流可以并发，一个流被阻塞了，并不影响其它的流。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我个人认为小帧比较好，当然如果在某些特定场景里，比如下载大文件，可以适当加大。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-07 14:00:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4e/94/0b22b6a2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Luke</span>
  </div>
  <div class="_2_QraFYR_0">1、一个流中的多个帧是有序的，但是在二进制帧协议中，并没有看到这个序号，请问下老师，这个序号是在哪里？或者一个流中的多个帧是如何保证有序的？<br>2、如果出现丢帧的情况是如何重传的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.由tcp保证帧是顺序到达的，因为tcp是有序的字节流。<br><br>2.同样是由tcp实现，具体细节就太底层了，有兴趣可以研究tcp协议。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-02 15:25:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/9e/c0/69ae8a58.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>sunxu</span>
  </div>
  <div class="_2_QraFYR_0">想问一下，nginx前端采用http2, 反向代理到应用服务使用的http1.1, 这种方式对请求响应有提升吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Nginx作为代理，实际上就把传输链路拆分成了两个部分，下游因为使用了http&#47;2，所以肯定会有性能提升。<br><br>而上游还是http&#47;1，所以瓶颈就在这里，但因为后台系统的服务能力都很强，网络也好，所以不会有太大影响。<br><br>当然，如果全用http&#47;2就更好了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-07 08:54:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTIuTjCibv0afd7SSdLicfNk0f7KO5ga9VMleD1hc2DtQfianK20ht06SekClKV7M8UXLRHqQLm9hJ3ow/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Jasmine</span>
  </div>
  <div class="_2_QraFYR_0">老师，同一个流内部，相同类型的帧，比如DATA[3]和DATA[3]怎么区分先后顺序呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 流里的帧是由tcp来传输的，而tcp是有序的字节流，所以http&#47;2流里的帧必然会是按照发送的顺序先后到达，不会是乱序。<br><br>所以，在tcp层面来看，帧是乱序的，但在流的层面来看，帧是有序的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-08 10:53:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/25/5e/40/dee7906c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张欣</span>
  </div>
  <div class="_2_QraFYR_0">老师，您好：<br>1，http2也是依赖于tcp来保证他自己帧数据的完整性么？他自己有检测数据完整性的功能么<br>2，http2的帧的排序序号也是保存在帧头部里面的么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.当然了，它的下层是tcp，传输的可靠性必须由tcp保证。<br><br>2.看帧结构，里面只有流id。因为帧是顺序发的，顺序由tcp保证，所以不需要序号。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-14 23:25:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/e4/4f/df6d810d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Maske</span>
  </div>
  <div class="_2_QraFYR_0">1.还是无状态，http2虽然实现了多路复用，但是本质没有变，因为服务端永远不会自动记录上次请求的相关数据，客户端的每一次请求都需要表明其身份。<br>2.小帧好。因为http2的一个一个请求都分解为了一个一个帧，在同一个tcp&#47;ip连接层面上表现为无序收发的，小帧有利于提高并行请求或响应。<br>3.通过分解一个一个请求或响应报文数据为多个帧，将同一个请求或响应的帧贴上相同的streamid来作为标识，使得在连接层面上变为无序收发，而且这些帧是并行的，因此不需要等待前一个请求响应结束才能进行下一个请求，实现了多路复用。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 说的非常好。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-26 13:50:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/67/f4/9a1feb59.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>钱</span>
  </div>
  <div class="_2_QraFYR_0">1：HTTP&#47;2 的动态表维护、流状态转换很复杂，你认为 HTTP&#47;2 还是“无状态”的吗？<br>还是无状态的，因为第二次交互并不需要保留或者知道是否在之前已经有过一次交互了，假设后面的服务是个集群第一次请求和第二次请求负载打到的机器不同也没关系，另外动态表应该可以重新创建吧！<br>另外，用代号表示语义是不是也是一种空间换时间的运用，虽然只是减少了传输空间增加了两头的存储空间。<br><br>2：HTTP&#47;2 的帧最大可以达到 16M，你觉得大帧好还是小帧好？<br>大帧好还是小帧好，我觉得应该看场景，如果网络不稳定小帧好因为丢失重传的成本低，也可以传输更多的数据帧提高并行度。不过如果网络稳定，帧小必然帧头多对于而数据是放在帧体区的是不是针对大文件下载之类的需求就有些浪费了呢？<br><br>3：结合这两讲，谈谈 HTTP&#47;2 是如何解决“队头阻塞”问题的。<br>HTTP&#47;2的连接好似一座桥，桥上可以过许多的车辆（数据帧），他们可以并发来走，而车队中的车辆是有顺序的她们组成一个车队（一次请求或响应报文），所以，先出发的车队不一定先到，因为车队之间是并行互不阻塞的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 说的很对。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-04 17:28:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/11/e7/044a9a6c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>book尾汁</span>
  </div>
  <div class="_2_QraFYR_0">1 http的无状态是针对应用层的吧，多个请求之前不会有影响，http2虽然有维护一些状态的信息，但这是针对流的信息，所以我认为http2也还是无状态的<br>2 不知道大小帧哪个好，大帧头部占比会少一点，但在TCP层会需要拆分成小帧，可能会多耗点时间，太大的帧TCP发送缓存区也要设的大一点吧<br>3  http2相比http1.1有了流ID来标识请求响应，因此同一个连接就可以同时进行多个流的传输，但由于TCP的收发窗口的确认机制，并发性还是会受到限制。<br><br>总结：<br>http2连接的建立<br>建立完成TLS连接之后，发送连接前言PRI * HTTP&#47;2.0\r\n\r\nSM\r\n\r\n，这样后面的报文就会使用http2的格式<br>报文格式：<br>帧长度  3byte    默认上限为16k 最大为2^24<br>帧类型  1byte   大致分为数据帧和控制帧，最高可表示256种，可自己扩展<br>标志位   1byte   携带简单的控制信息<br>流ID    4byte  其中最高位为0，客户端发起的流id为奇数，服务器发起的流id为偶数，0号流为控制流<br><br>请求头字段采用HPACK的方式来进行压缩，有静态表来存储二元组（字段 ，value）和index之间的关系，静态表里找不到的key value，可以放在动态表里。<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 总结的不错。<br><br>1.可以说是语义无状态，语法有状态。<br><br>2.通常都认为较小的帧比较好，粒度小，流的并行度就高。<br><br>3.http&#47;2在tcp层有队头阻塞，所以就出了http&#47;3。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-04 00:54:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/75/aa/21275b9d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>闫飞</span>
  </div>
  <div class="_2_QraFYR_0">哈夫曼编码的使用是HTTP2协议里面一个不容忽视的要素，该编码方式很好地保证了所有信息的平均码长最短而互相不构成前缀关系，易于解码。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-14 08:16:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/79/4b/740f91ca.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>-W.LI-</span>
  </div>
  <div class="_2_QraFYR_0">老师好!TCP网络不好的时候会降速，http2的话是一个帧没收到就会导致TCP降速么?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，会发生tcp层的队头阻塞，下一次http&#47;3会讲。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-07 08:44:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/9a/67/73f384f9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>好好好</span>
  </div>
  <div class="_2_QraFYR_0">想请问下老师，我在别的地方看到http1.1有管道机制可以同时发送请求，服务端会根据请求顺序返回内容。那为什么还说http1.x如果同时发送请求，服务端无法区分数据是哪个请求所以实现不了多路复用呢？希望老师解答下。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.http&#47;1.1的管道虽然可以发多个请求，但仍然是顺序处理，有队头阻塞，所以很少有服务器、浏览器支持，基本上没用。<br><br>2.http&#47;1.1既然是顺序请求，那么就只能是串行处理，也就无法像http&#47;2那样多路复用了。<br><br>3.多个并发的http&#47;1.1请求，服务器可以根据tcp连接来区分，不可能是“无法区分”的，这个说法不知道是哪里来的，我认为这个是错误的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-30 02:03:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/29/55/03/1092fb6a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>假于物</span>
  </div>
  <div class="_2_QraFYR_0">老师，帧头和数据帧对应关系是怎么样的？<br>1对1还是1对多？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 当然是一个帧头只能对应一个帧体了，但流里可以有多个数据帧。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-04 17:34:43</div>
  </div>
</div>
</div>
</li>
</ul>