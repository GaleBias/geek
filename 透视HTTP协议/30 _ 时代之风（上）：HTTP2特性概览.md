<audio title="30 _ 时代之风（上）：HTTP2特性概览" src="https://static001.geekbang.org/resource/audio/eb/d5/ebc13babfb4a88680afcf18dde166fd5.mp3" controls="controls"></audio> 
<p>在<a href="https://time.geekbang.org/column/article/103746">第14讲</a>里，我们看到HTTP有两个主要的缺点：安全不足和性能不高。</p><p>刚结束的“安全篇”里的HTTPS，通过引入SSL/TLS在安全上达到了“极致”，但在性能提升方面却是乏善可陈，只优化了握手加密的环节，对于整体的数据传输没有提出更好的改进方案，还只能依赖于“长连接”这种“落后”的技术（参见<a href="https://time.geekbang.org/column/article/104949">第17讲</a>）。</p><p>所以，在HTTPS逐渐成熟之后，HTTP就向着性能方面开始“发力”，走出了另一条进化的道路。</p><p>在<a href="https://time.geekbang.org/column/article/97837">第1讲</a>的HTTP历史中你也看到了，“秦失其鹿，天下共逐之”，Google率先发明了SPDY协议，并应用于自家的浏览器Chrome，打响了HTTP性能优化的“第一枪”。</p><p>随后互联网标准化组织IETF以SPDY为基础，综合其他多方的意见，终于推出了HTTP/1的继任者，也就是今天的主角“HTTP/2”，在性能方面有了一个大的飞跃。</p><h2>为什么不是HTTP/2.0</h2><p>你一定很想知道，为什么HTTP/2不像之前的“1.0”“1.1”那样叫“2.0”呢？</p><p>这个也是很多初次接触HTTP/2的人问的最多的一个问题，对此HTTP/2工作组特别给出了解释。</p><p>他们认为以前的“1.0”“1.1”造成了很多的混乱和误解，让人在实际的使用中难以区分差异，所以就决定HTTP协议不再使用小版本号（minor version），只使用大版本号（major version），从今往后HTTP协议不会出现HTTP/2.0、2.1，只会有“HTTP/2”“HTTP/3”……</p><!-- [[[read_end]]] --><p>这样就可以明确无误地辨别出协议版本的“跃进程度”，让协议在一段较长的时期内保持稳定，每当发布新版本的HTTP协议都会有本质的不同，绝不会有“零敲碎打”的小改良。</p><h2>兼容HTTP/1</h2><p>由于HTTPS已经在安全方面做的非常好了，所以HTTP/2的唯一目标就是改进性能。</p><p>但它不仅背负着众多的期待，同时还背负着HTTP/1庞大的历史包袱，所以协议的修改必须小心谨慎，兼容性是首要考虑的目标，否则就会破坏互联网上无数现有的资产，这方面TLS已经有了先例（为了兼容TLS1.2不得不进行“伪装”）。</p><p>那么，HTTP/2是怎么做的呢？</p><p>因为必须要保持功能上的兼容，所以HTTP/2把HTTP分解成了“<span class="orange">语义</span>”和“<span class="orange">语法</span>”两个部分，“语义”层不做改动，与HTTP/1完全一致（即RFC7231）。比如请求方法、URI、状态码、头字段等概念都保留不变，这样就消除了再学习的成本，基于HTTP的上层应用也不需要做任何修改，可以无缝转换到HTTP/2。</p><p>特别要说的是，与HTTPS不同，HTTP/2没有在URI里引入新的协议名，仍然用“http”表示明文协议，用“https”表示加密协议。</p><p>这是一个非常了不起的决定，可以让浏览器或者服务器去自动升级或降级协议，免去了选择的麻烦，让用户在上网的时候都意识不到协议的切换，实现平滑过渡。</p><p>在“语义”保持稳定之后，HTTP/2在“语法”层做了“天翻地覆”的改造，完全变更了HTTP报文的传输格式。</p><h2>头部压缩</h2><p>首先，HTTP/2对报文的头部做了一个“大手术”。</p><p>通过“进阶篇”的学习你应该知道，HTTP/1里可以用头字段“Content-Encoding”指定Body的编码方式，比如用gzip压缩来节约带宽，但报文的另一个组成部分——Header却被无视了，没有针对它的优化手段。</p><p>由于报文Header一般会携带“User Agent”“Cookie”“Accept”“Server”等许多固定的头字段，多达几百字节甚至上千字节，但Body却经常只有几十字节（比如GET请求、204/301/304响应），成了不折不扣的“大头儿子”。更要命的是，成千上万的请求响应报文里有很多字段值都是重复的，非常浪费，“长尾效应”导致大量带宽消耗在了这些冗余度极高的数据上。</p><p>所以，HTTP/2把“<strong>头部压缩</strong>”作为性能改进的一个重点，优化的方式你也肯定能想到，还是“压缩”。</p><p>不过HTTP/2并没有使用传统的压缩算法，而是开发了专门的“<strong>HPACK</strong>”算法，在客户端和服务器两端建立“字典”，用索引号表示重复的字符串，还釆用哈夫曼编码来压缩整数和字符串，可以达到50%~90%的高压缩率。</p><h2>二进制格式</h2><p>你可能已经很习惯于HTTP/1里纯文本形式的报文了，它的优点是“一目了然”，用最简单的工具就可以开发调试，非常方便。</p><p>但HTTP/2在这方面没有“妥协”，决定改变延续了十多年的现状，不再使用肉眼可见的ASCII码，而是向下层的TCP/IP协议“靠拢”，全面采用二进制格式。</p><p>这样虽然对人不友好，但却大大方便了计算机的解析。原来使用纯文本的时候容易出现多义性，比如大小写、空白字符、回车换行、多字少字等等，程序在处理时必须用复杂的状态机，效率低，还麻烦。</p><p>而二进制里只有“0”和“1”，可以严格规定字段大小、顺序、标志位等格式，“对就是对，错就是错”，解析起来没有歧义，实现简单，而且体积小、速度快，做到“内部提效”。</p><p>以二进制格式为基础，HTTP/2就开始了“大刀阔斧”的改革。</p><p>它把TCP协议的部分特性挪到了应用层，把原来的“Header+Body”的消息“打散”为数个小片的<strong>二进制“帧”</strong>（Frame），用“HEADERS”帧存放头数据、“DATA”帧存放实体数据。</p><p>这种做法有点像是“Chunked”分块编码的方式（参见<a href="https://time.geekbang.org/column/article/104456">第16讲</a>），也是“化整为零”的思路，但HTTP/2数据分帧后“Header+Body”的报文结构就完全消失了，协议看到的只是一个个的“碎片”。</p><p><img src="https://static001.geekbang.org/resource/image/8f/96/8fe2cbd57410299a1a36d7eb105ea896.png?wh=1236*366" alt=""></p><h2>虚拟的“流”</h2><p>消息的“碎片”到达目的地后应该怎么组装起来呢？</p><p>HTTP/2为此定义了一个“<strong>流</strong>”（Stream）的概念，<strong>它是二进制帧的双向传输序列</strong>，同一个消息往返的帧会分配一个唯一的流ID。你可以把它想象成是一个虚拟的“数据流”，在里面流动的是一串有先后顺序的数据帧，这些数据帧按照次序组装起来就是HTTP/1里的请求报文和响应报文。</p><p>因为“流”是虚拟的，实际上并不存在，所以HTTP/2就可以在一个TCP连接上用“<strong>流</strong>”同时发送多个“碎片化”的消息，这就是常说的“<strong>多路复用</strong>”（ Multiplexing）——多个往返通信都复用一个连接来处理。</p><p>在“流”的层面上看，消息是一些有序的“帧”序列，而在“连接”的层面上看，消息却是乱序收发的“帧”。多个请求/响应之间没有了顺序关系，不需要排队等待，也就不会再出现“队头阻塞”问题，降低了延迟，大幅度提高了连接的利用率。</p><p><img src="https://static001.geekbang.org/resource/image/d8/bc/d8fd32a4d044f2078b3a260e4478c5bc.png?wh=1231*842" alt=""></p><p>为了更好地利用连接，加大吞吐量，HTTP/2还添加了一些控制帧来管理虚拟的“流”，实现了优先级和流量控制，这些特性也和TCP协议非常相似。</p><p>HTTP/2还在一定程度上改变了传统的“请求-应答”工作模式，服务器不再是完全被动地响应请求，也可以新建“流”主动向客户端发送消息。比如，在浏览器刚请求HTML的时候就提前把可能会用到的JS、CSS文件发给客户端，减少等待的延迟，这被称为“<strong>服务器推送</strong>”（Server Push，也叫Cache Push）。</p><h2>强化安全</h2><p>出于兼容的考虑，HTTP/2延续了HTTP/1的“明文”特点，可以像以前一样使用明文传输数据，不强制使用加密通信，不过格式还是二进制，只是不需要解密。</p><p>但由于HTTPS已经是大势所趋，而且主流的浏览器Chrome、Firefox等都公开宣布只支持加密的HTTP/2，所以“事实上”的HTTP/2是加密的。也就是说，互联网上通常所能见到的HTTP/2都是使用“https”协议名，跑在TLS上面。</p><p>为了区分“加密”和“明文”这两个不同的版本，HTTP/2协议定义了两个字符串标识符：“<span class="orange">h2</span>”表示加密的HTTP/2，“<span class="orange">h2c</span>”表示明文的HTTP/2，多出的那个字母“c”的意思是“clear text”。</p><p>在HTTP/2标准制定的时候（2015年）已经发现了很多SSL/TLS的弱点，而新的TLS1.3还未发布，所以加密版本的HTTP/2在安全方面做了强化，要求下层的通信协议必须是TLS1.2以上，还要支持前向安全和SNI，并且把几百个弱密码套件列入了“黑名单”，比如DES、RC4、CBC、SHA-1都不能在HTTP/2里使用，相当于底层用的是“TLS1.25”。</p><h2>协议栈</h2><p>下面的这张图对比了HTTP/1、HTTPS和HTTP/2的协议栈，你可以清晰地看到，HTTP/2是建立在“HPack”“Stream”“TLS1.2”基础之上的，比HTTP/1、HTTPS复杂了一些。</p><p><img src="https://static001.geekbang.org/resource/image/83/1a/83c9f0ecad361ba8ef8f3b73d6872f1a.png?wh=1227*632" alt=""></p><p>虽然HTTP/2的底层实现很复杂，但它的“语义”还是简单的HTTP/1，之前学习的知识不会过时，仍然能够用得上。</p><p>我们的实验环境在新的域名“<strong>www.metroid.net</strong>”上启用了HTTP/2协议，你可以把之前“进阶篇”“安全篇”的测试用例都走一遍，再用Wireshark抓一下包，实际看看HTTP/2的效果和对老协议的兼容性（例如“<a href="http://www.metroid.net/11-1">http://www.metroid.net/11-1</a>”）。</p><p>在今天这节课专用的URI“/30-1”里，你还可以看到服务器输出了HTTP的版本号“2”和标识符“h2”，表示这是加密的HTTP/2，如果改用“<a href="https://www.chrono.com/30-1">https://www.chrono.com/30-1</a>”访问就会是“1.1”和空。</p><p><img src="https://static001.geekbang.org/resource/image/fd/d1/fdf1a6916c3ac22b6fb7628de3d7ddd1.png?wh=1142*1139" alt=""></p><p>你可能还会注意到URI里的一个小变化，端口使用的是“8443”而不是“443”。这是因为443端口已经被“www.chrono.com”的HTTPS协议占用，Nginx不允许在同一个端口上根据域名选择性开启HTTP/2，所以就不得不改用了“8443”。</p><h2>小结</h2><p>今天我简略介绍了HTTP/2的一些重要特性，比较偏重理论，下一次我会用Wireshark抓包，具体讲解HTTP/2的头部压缩、二进制帧和流等特性。</p><ol>
<li><span class="orange">HTTP协议取消了小版本号，所以HTTP/2的正式名字不是2.0；</span></li>
<li><span class="orange">HTTP/2在“语义”上兼容HTTP/1，保留了请求方法、URI等传统概念；</span></li>
<li><span class="orange">HTTP/2使用“HPACK”算法压缩头部信息，消除冗余数据节约带宽；</span></li>
<li><span class="orange">HTTP/2的消息不再是“Header+Body”的形式，而是分散为多个二进制“帧”；</span></li>
<li><span class="orange">HTTP/2使用虚拟的“流”传输消息，解决了困扰多年的“队头阻塞”问题，同时实现了“多路复用”，提高连接的利用率；</span></li>
<li><span class="orange">HTTP/2也增强了安全性，要求至少是TLS1.2，而且禁用了很多不安全的密码套件。</span></li>
</ol><h2>课下作业</h2><ol>
<li>你觉得明文形式的HTTP/2（h2c）有什么好处，应该如何使用呢？</li>
<li>你觉得应该怎样理解HTTP/2里的“流”，为什么它是“虚拟”的？</li>
<li>你能对比一下HTTP/2与HTTP/1、HTTPS的相同点和不同点吗？</li>
</ol><p>欢迎你把自己的学习体会写在留言区，与我和其他同学一起讨论。如果你觉得有所收获，也欢迎把文章分享给你的朋友。</p><p><img src="https://static001.geekbang.org/resource/image/78/42/781da6191d342d71d3be2675cb610742.png?wh=1769*3769" alt="unpreview"></p><p><img src="https://static001.geekbang.org/resource/image/56/63/56d766fc04654a31536f554b8bde7b63.jpg?wh=1110*659" alt="unpreview"></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/f4/87/644c0c5d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>俊伟</span>
  </div>
  <div class="_2_QraFYR_0">1.h2c使用明文传输，速度更快，不需要TLS握手。<br>2.客户端将多个请求分成不同的流，然后每个流里面在切成一个个帧，发送的时候是按帧发送的。每个帧存着一个流ID来表示它属于的流。服务端收到请求的时候将帧按流ID进行拼接。从传输的角度来看流是不存在的，只是看到了一个个帧，所以说流是虚拟的。<br>3.相同点：都是基于TCP和TLS的，url格式都是相同的。都是基于header+body的形式。都是请求-应答模型。<br>4.不同点： 1.使用了HPACK进行头部压缩。<br>                2.使用的是二进制的方式进行传输。<br>                3.将多个请求切分成帧发送，实现了多路复用。这个感觉上利用了多道程序设计的思想。<br>                4.服务器可以主动向客户端推送消息。充分利用了TCP的全功双通道。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 总结的非常好，great。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-19 10:46:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/55/45/e4314bc6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>magicnum</span>
  </div>
  <div class="_2_QraFYR_0">h2c优点是性能，不需要TLS握手以及加解密。可以通过curl工具构造h2c请求；<br>h2的流是虚拟的因为它是使用帧传输数据的，相同streamid的帧组成了虚拟消息以及流；<br>相同点：都是基于tcp或TLS，并且是基于请求-响应模型，schema还是http或https不会有http2。<br>不同点：h2使用二进制传输消息并且通过HPACK压缩请求头，实现流多路复用、服务器推送<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-05 12:24:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/35/51/c616f95a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>阿锋</span>
  </div>
  <div class="_2_QraFYR_0">突然想起了一个问题，get和post请求其中一个区别是，post请求会把请求的数据放入请求体（body）中，而get请求是拼接到url后面。get请求是不是一定不能往请求体（body）中放入数据。还是这些都只是客户端和服务端的约定，可以灵活的自定义，没有强制的要求。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: get也可以有body，post也可以用query参数，区别的关键在于动作语义，一个是取一个是存。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-05 15:12:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/5a/13/3160996d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>nb Ack</span>
  </div>
  <div class="_2_QraFYR_0">老师好。我想问一下，http2的多路复用和http的长连接效果不是一样吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 完全不一样。<br><br>多路复用多个请求没有顺序，而长连接多个请求必须排队，就会队头阻塞。<br><br>可以再看看示意图体会一下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-05 12:43:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/81/4e/d71092f4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>夏目</span>
  </div>
  <div class="_2_QraFYR_0">流就是逻辑上将数据帧按id分组了，同组有序，组间无序，本质就是id相同的几个数据帧所以流是虚拟的。在tcp层面还是队首阻塞的吧？需要等待ack</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，理解的非常正确。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-09 20:37:06</div>
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
  <div class="_2_QraFYR_0">1.明文传输时不需要进行加密解密动作，不需要TLS握手，能节约性能。适用于对数据传输安全性要求不高的场景。<br>2.http2改变了http1.1的“请求-应答”模式，将head+body的请求报文在传输过程中改为 head帧 + data帧，在同一个TCP&#47;IP中，可以将多个请求分解为多个帧，从连接层面来看，这些帧是无序的，为了让接受端准确的将这些帧还原为一个一个独立的请求或响应，就给了每一个帧分配了streamid，streamid相同的即为同一个请求或响应的数据。因此，此处的流并不是真实有序的二进制字节，所以叫‘虚拟流’。<br>3.http1.1解决的是在万维网中，计算机之间的信息通信的一套规范，包括定义其属于应用层协议，建立在tcp&#47;ip之上，请求响应的报文结构等。https不改变http1.1的原有属性，是在其之上新增了对数据安全性和有效性的特性，解决的是数据安全的问题，通过使用加密解密，数字证书，TLS握手等过程保证了这一点。http2解决的是性能问题，通过头部压缩，使用二进制传输，多路复用，服务器推送等策略使得http的性能更好。http2和https本质上都是对http1.1的扩展和延伸。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 理解的很透彻，great。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-26 12:54:07</div>
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
  <div class="_2_QraFYR_0">课后习题出的很好。可惜我不会坐等答案<br>1.内网用h2c会比https快么?<br>2.感觉回答虚拟流之前给先回答啥是真真的流。我对流的理解是有序，切只能读一次。http2支持乱序发，我猜也支持，部分帧重发，所以就是虚拟的了。<br>3.共同，都是应用层协议，传输成都用的TCP。<br>不同:https=TLS+HTTP&#47;HTTP2，安全。<br>http2:二进制传输，对header压缩，通过二进制分帧解决了队头阻塞，传输效率更高，服务端可推数据<br>http:明文，队头阻塞，半双工。<br>问题1:一个TCP链接可以打开很多channel是吧，每一个channel都可以传输数据。底层具体怎么实现的啊，是怎么区分channel里的数据谁是谁的?<br>问题2:我之前看见TPC好像是通过服务端IP,服务端端口号,客户端端IP,客户端端口号。来唯一标识一个链接的。http1的时候队头阻塞，继续要多建http链接。每建立一个链接客户端就用一个不同的端口号么?<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.当然，省去了加密的成本。<br><br>2.所谓“虚拟的流”，是指流实际上是多个同一序号的帧，并没有真正的流数据结构，这与连接不同。<br><br>3.正确。<br><br>4.你说的channel应该是http&#47;2里的“流”吧，http&#47;2里没有channel。流是由帧组成的，帧头里有流id标记所属的流，马上会讲具体的细节。<br><br>5.标记一个tcp连接要用四元组（客户端ip端口+服务器ip端口），所以肯定要用一个新的端口号，在客户端这是临时分配的，而服务器是固定的端口。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-05 21:54:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/55/cb/1efe460a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>渴望做梦</span>
  </div>
  <div class="_2_QraFYR_0">老师，我有个疑问，既然http2是二进制的格式，那我们还能用chrome自带的工具调试吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以的，Chrome会把二进制解码，还原为http&#47;1的文本形式，你可以自己试一下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-29 08:21:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/18/65/35361f02.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>潇潇雨歇</span>
  </div>
  <div class="_2_QraFYR_0">1、明文传输性能更好，省去了加密相关操作<br>2、流和请求&#47;应答一样，但是流是相同流id的帧组合，不同流可以无序，相同流有序。整个看起来是无序的，请求之间不受影响。这也解决了http1.1的队头阻塞。<br>3、三者都是基于tcp的，基本语义是一样的。http2在性能上做了提升，比如二进制帧，流，服务器推送，HPACK算法等；https在安全上做了提升，下层多了TLS&#47;SSL，要多做一些握手加密证书验证等操作。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 回答的非常好！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-30 13:38:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/44/c0/cd2cd082.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>BoyiKia</span>
  </div>
  <div class="_2_QraFYR_0">http2  优点<br>1.兼容性<br>   兼容以前的http1.1 ，https等。<br><br>2.性能提升<br>   报文变成了 二进制数据帧，提高传输效率，和减少歧义。<br>    ①header 采用了头部压缩，来减小传输体积。<br>    ②body数据 放到了 data帧。<br>   a.同一请求或响应的数据帧具有相同的帧标识(流ID)，两端接受到的帧数据可以通过同一帧标识，重新组装成请求或响应数据。<br> b.不同请求&#47;响应的数据帧可以乱序发，避免生成请求队列造成的队头阻塞。<br>c.同一个TCP连接上，可以并行发送多种流的数据帧(多路复用，PS： http1的 多路复用是分母效应，同一连接串行增加http通信 )。<br>d.强化了请求响应模式，服务器可以主动发送信息-服务器推送。<br>3.安全性<br>①.要求下层必须是 TlS1.2以上，支持前向安全，废除安全性比较低的密码套件。<br><br><br>   <br> </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: awesome!</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-18 20:59:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/f5/0b/73628618.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>兔嘟嘟</span>
  </div>
  <div class="_2_QraFYR_0">老师好，我不太理解为什么二进制帧可以提高解析效率，我的理解是这样的：<br>在HTTP&#47;1.1中，请求方的字符串在TCP层被解码为Unicode二进制，然后应答方在HTTP层编码为utf-8字符串。<br>而在HTTP&#47;2中，请求方的字符串在HTTP层被解码为二进制，然后应答方在浏览器处编码为字符串。所以好像没有省去时间或者资源。请老师赐教</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 二进制的好处显而易见，用位来表示信息，要比字符串表示简单，比如用01表示host，而用字符串就需要4个字节，而且要用状态机去检测单词，非常麻烦。<br><br>http&#47;2在底层是二进制，解析起来快速方便，然后再到应用层对字符串做个映射就行了，不是再编码。<br><br>可以再用hpack来理解一下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-29 11:12:30</div>
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
  <div class="_2_QraFYR_0">老师我又回来了，按之前的理解是，http2是对同一域名使用单一的TCP连接进行数据传输，多个请求同时进行，既然如此，为什么在chrome调试面板中还能看到资源还是是有请求排队时间的呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: http&#47;2里面的流，就相当于http&#47;1里的并发连接，要开一个新流同样也要有一些准备的工作。Chrome里的排队属于它自己的调度工作，与http&#47;2协议是无关的。<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-20 19:57:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/66/51/052c7b30.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>谢一</span>
  </div>
  <div class="_2_QraFYR_0">老师，既然在连接层，是无序的，那在http&#47;2中是怎么保证frame的有序性的呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: tcp层是有序的，所以一个流里的多个帧会按照顺序依次到达，接收方只要依次接收就可以了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-09 09:41:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c6/73/abb7bfe3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>疯琴</span>
  </div>
  <div class="_2_QraFYR_0">请问老师，同一个流里面不同序号的帧可以乱序到达统一组装么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不会的，tcp会保证有序送达，多个流是并行乱序发，但看单个流，它里面的帧还是有序的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-16 09:12:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/01/c5/b48d25da.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>cake</span>
  </div>
  <div class="_2_QraFYR_0">老师请问下，而在“连接”的层面上看，消息却是乱序收发的“帧”，为什么HTTP3 Over UDP 连接层面是包，这个HTTP2连接层面为啥是帧呢？不应该叫报文段么</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我觉得这只是称呼的问题，都是一段段的数据，只是协议里起了个专用名词，把它的作用功能弄清楚就行了，不要太纠结名字。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-24 12:02:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>思维决定未来</span>
  </div>
  <div class="_2_QraFYR_0">http2的事实标准就是加密传输的，那是不是跟https重复了？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 前面说过，对http的改进有两个方向，一个是安全，一个是效率，http&#47;2是安全和效率兼顾，而https只是传输安全，效率上没有改进。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-07 19:23:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/64/8a/bc8cb43c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>路边的猪</span>
  </div>
  <div class="_2_QraFYR_0">想知道对于 HTTP&#47;2 这种大的版本，以及包括之前的http&#47;1.1 <br>是怎么一步步在全世界范围内被广泛应用的？协议颁发出来，是会有一些大头企业领头去实现基于HTTP&#47;2的协议吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是啊，http&#47;2、http&#47;3都是Google先做出个原型，然后在自家的Chrome里试用推广，再不断根据反馈修正，就成为了事实标准，最后提交IETF标准化。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-13 17:29:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/98/1c/d7a1439e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>KaKaKa</span>
  </div>
  <div class="_2_QraFYR_0">老师，重新复习的时候，我突然有点疑惑：<br>1.同一个TCP连接中，多个流是怎么并发请求的？再怎么并发，不都是需要这个TCP连接去一个个数据包进行传输吗？那为什么还需要有多个流？<br>2.多个TCP连接，每个连接都能单独去发送数据包，这种形式不是更快吗？<br>本来我以为我了解这章了，现在重新复习的时候，有些点还是不太了解</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1. 流是逻辑上的概念，实际上就是一些数据包，但这些包在tcp层面是无序的，也就解决了队头阻塞问题，在http&#47;2层面把这些包整理成流，每个流对应一个http请求响应。<br><br>2.多个连接当然很快，但问题是服务器不允许客户端建立任意多的连接。<br><br>3.可以在看一下之前讲http&#47;2历史的课程，理解它针对的痛点。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-10 14:49:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83eqf54z1ZmqQY1kmJ6t1HAnrqMM3j6WKf0oDeVLhtnA2ZUKY6AX9MK6RjvcO8SiczXy3uU0IzBQ3tpw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_68d3d2</span>
  </div>
  <div class="_2_QraFYR_0">请问同一个流可以在一个tcp连接里面发送吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 当然了，流实际上就是多个无序的数据帧，在tcp连接里乱序发送，到了目的地再按照序号组装，就形成了流。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-09 15:50:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/83/98/fab9bd2a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Mingyan</span>
  </div>
  <div class="_2_QraFYR_0">我有疑问，http2.0如果遇到服务器主动关闭tcp链接会理会回ack，再发fin等ack去关闭tcp链接还是不理会继续使用被关闭的tcp链接了？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: tcp连接关闭后就消失了，再建连就是一个新的连接，不存在复用。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-04 18:52:17</div>
  </div>
</div>
</div>
</li>
</ul>