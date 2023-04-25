<audio title="38 _ WebSocket：沙盒里的TCP" src="https://static001.geekbang.org/resource/audio/5e/e2/5e9e27f590f3fd65f21975e334447ee2.mp3" controls="controls"></audio> 
<p>在之前讲TCP/IP协议栈的时候，我说过有“TCP Socket”，它实际上是一种功能接口，通过这些接口就可以使用TCP/IP协议栈在传输层收发数据。</p><p>那么，你知道还有一种东西叫“WebSocket”吗？</p><p>单从名字上看，“Web”指的是HTTP，“Socket”是套接字调用，那么这两个连起来又是什么意思呢？</p><p>所谓“望文生义”，大概你也能猜出来，“WebSocket”就是运行在“Web”，也就是HTTP上的Socket通信规范，提供与“TCP Socket”类似的功能，使用它就可以像“TCP Socket”一样调用下层协议栈，任意地收发数据。</p><p><img src="https://static001.geekbang.org/resource/image/ee/28/ee6685c7d3c673b95e46d582828eee28.png?wh=1142*502" alt=""></p><p>更准确地说，“WebSocket”是一种基于TCP的轻量级网络通信协议，在地位上是与HTTP“平级”的。</p><h2>为什么要有WebSocket</h2><p>不过，已经有了被广泛应用的HTTP协议，为什么要再出一个WebSocket呢？它有哪些好处呢？</p><p>其实WebSocket与HTTP/2一样，都是为了解决HTTP某方面的缺陷而诞生的。HTTP/2针对的是“队头阻塞”，而WebSocket针对的是“请求-应答”通信模式。</p><p>那么，“请求-应答”有什么不好的地方呢？</p><p>“请求-应答”是一种“<strong>半双工</strong>”的通信模式，虽然可以双向收发数据，但同一时刻只能一个方向上有动作，传输效率低。更关键的一点，它是一种“被动”通信模式，服务器只能“被动”响应客户端的请求，无法主动向客户端发送数据。</p><!-- [[[read_end]]] --><p>虽然后来的HTTP/2、HTTP/3新增了Stream、Server Push等特性，但“请求-应答”依然是主要的工作方式。这就导致HTTP难以应用在动态页面、即时消息、网络游戏等要求“<strong>实时通信</strong>”的领域。</p><p>在WebSocket出现之前，在浏览器环境里用JavaScript开发实时Web应用很麻烦。因为浏览器是一个“受限的沙盒”，不能用TCP，只有HTTP协议可用，所以就出现了很多“变通”的技术，“<strong>轮询</strong>”（polling）就是比较常用的的一种。</p><p>简单地说，轮询就是不停地向服务器发送HTTP请求，问有没有数据，有数据的话服务器就用响应报文回应。如果轮询的频率比较高，那么就可以近似地实现“实时通信”的效果。</p><p>但轮询的缺点也很明显，反复发送无效查询请求耗费了大量的带宽和CPU资源，非常不经济。</p><p>所以，为了克服HTTP“请求-应答”模式的缺点，WebSocket就“应运而生”了。它原来是HTML5的一部分，后来“自立门户”，形成了一个单独的标准，RFC文档编号是6455。</p><h2>WebSocket的特点</h2><p>WebSocket是一个真正“<strong>全双工</strong>”的通信协议，与TCP一样，客户端和服务器都可以随时向对方发送数据，而不用像HTTP“你拍一，我拍一”那么“客套”。于是，服务器就可以变得更加“主动”了。一旦后台有新的数据，就可以立即“推送”给客户端，不需要客户端轮询，“实时通信”的效率也就提高了。</p><p>WebSocket采用了二进制帧结构，语法、语义与HTTP完全不兼容，但因为它的主要运行环境是浏览器，为了便于推广和应用，就不得不“搭便车”，在使用习惯上尽量向HTTP靠拢，这就是它名字里“Web”的含义。</p><p>服务发现方面，WebSocket没有使用TCP的“IP地址+端口号”，而是延用了HTTP的URI格式，但开头的协议名不是“http”，引入的是两个新的名字：“<strong>ws</strong>”和“<strong>wss</strong>”，分别表示明文和加密的WebSocket协议。</p><p>WebSocket的默认端口也选择了80和443，因为现在互联网上的防火墙屏蔽了绝大多数的端口，只对HTTP的80、443端口“放行”，所以WebSocket就可以“伪装”成HTTP协议，比较容易地“穿透”防火墙，与服务器建立连接。具体是怎么“伪装”的，我稍后再讲。</p><p>下面我举几个WebSocket服务的例子，你看看，是不是和HTTP几乎一模一样：</p><pre><code>ws://www.chrono.com
ws://www.chrono.com:8080/srv
wss://www.chrono.com:445/im?user_id=xxx
</code></pre><p>要注意的一点是，WebSocket的名字容易让人产生误解，虽然大多数情况下我们会在浏览器里调用API来使用WebSocket，但它不是一个“调用接口的集合”，而是一个通信协议，所以我觉得把它理解成“<strong>TCP over Web</strong>”会更恰当一些。</p><h2>WebSocket的帧结构</h2><p>刚才说了，WebSocket用的也是二进制帧，有之前HTTP/2、HTTP/3的经验，相信你这次也能很快掌握WebSocket的报文结构。</p><p>不过WebSocket和HTTP/2的关注点不同，WebSocket更<strong>侧重于“实时通信”</strong>，而HTTP/2更侧重于提高传输效率，所以两者的帧结构也有很大的区别。</p><p>WebSocket虽然有“帧”，但却没有像HTTP/2那样定义“流”，也就不存在“多路复用”“优先级”等复杂的特性，而它自身就是“全双工”的，也就不需要“服务器推送”。所以综合起来，WebSocket的帧学习起来会简单一些。</p><p>下图就是WebSocket的帧结构定义，长度不固定，最少2个字节，最多14字节，看着好像很复杂，实际非常简单。</p><p><img src="https://static001.geekbang.org/resource/image/29/c4/29d33e972dda5a27aa4773eea896a8c4.png?wh=2468*1523" alt=""></p><p>开头的两个字节是必须的，也是最关键的。</p><p>第一个字节的第一位“<strong>FIN</strong>”是消息结束的标志位，相当于HTTP/2里的“END_STREAM”，表示数据发送完毕。一个消息可以拆成多个帧，接收方看到“FIN”后，就可以把前面的帧拼起来，组成完整的消息。</p><p>“FIN”后面的三个位是保留位，目前没有任何意义，但必须是0。</p><p>第一个字节的后4位很重要，叫<strong>“Opcode</strong>”，操作码，其实就是帧类型，比如1表示帧内容是纯文本，2表示帧内容是二进制数据，8是关闭连接，9和10分别是连接保活的PING和PONG。</p><p>第二个字节第一位是掩码标志位“<strong>MASK</strong>”，表示帧内容是否使用异或操作（xor）做简单的加密。目前的WebSocket标准规定，客户端发送数据必须使用掩码，而服务器发送则必须不使用掩码。</p><p>第二个字节后7位是“<strong>Payload len</strong>”，表示帧内容的长度。它是另一种变长编码，最少7位，最多是7+64位，也就是额外增加8个字节，所以一个WebSocket帧最大是2^64。</p><p>长度字段后面是“<strong>Masking-key</strong>”，掩码密钥，它是由上面的标志位“MASK”决定的，如果使用掩码就是4个字节的随机数，否则就不存在。</p><p>这么分析下来，其实WebSocket的帧头就四个部分：“<strong>结束标志位+操作码+帧长度+掩码</strong>”，只是使用了变长编码的“小花招”，不像HTTP/2定长报文头那么简单明了。</p><p>我们的实验环境利用OpenResty的“lua-resty-websocket”库，实现了一个简单的WebSocket通信，你可以访问URI“/38-1”，它会连接后端的WebSocket服务“ws://127.0.0.1/38-0”，用Wireshark抓包就可以看到WebSocket的整个通信过程。</p><p>下面的截图是其中的一个文本帧，因为它是客户端发出的，所以需要掩码，报文头就在两个字节之外多了四个字节的“Masking-key”，总共是6个字节。</p><p><img src="https://static001.geekbang.org/resource/image/c9/94/c91ee4815097f5f9059ab798bb841594.png?wh=1269*634" alt=""></p><p>而报文内容经过掩码，不是直接可见的明文，但掩码的安全强度几乎是零，用“Masking-key”简单地异或一下就可以转换出明文。</p><h2>WebSocket的握手</h2><p>和TCP、TLS一样，WebSocket也要有一个握手过程，然后才能正式收发数据。</p><p>这里它还是搭上了HTTP的“便车”，利用了HTTP本身的“协议升级”特性，“伪装”成HTTP，这样就能绕过浏览器沙盒、网络防火墙等等限制，这也是WebSocket与HTTP的另一个重要关联点。</p><p>WebSocket的握手是一个标准的HTTP GET请求，但要带上两个协议升级的专用头字段：</p><ul>
<li>“Connection: Upgrade”，表示要求协议“升级”；</li>
<li>“Upgrade: websocket”，表示要“升级”成WebSocket协议。</li>
</ul><p>另外，为了防止普通的HTTP消息被“意外”识别成WebSocket，握手消息还增加了两个额外的认证用头字段（所谓的“挑战”，Challenge）：</p><ul>
<li>Sec-WebSocket-Key：一个Base64编码的16字节随机数，作为简单的认证密钥；</li>
<li>Sec-WebSocket-Version：协议的版本号，当前必须是13。</li>
</ul><p><img src="https://static001.geekbang.org/resource/image/8f/97/8f007bb0e403b6cc28493565f709c997.png?wh=2096*865" alt=""></p><p>服务器收到HTTP请求报文，看到上面的四个字段，就知道这不是一个普通的GET请求，而是WebSocket的升级请求，于是就不走普通的HTTP处理流程，而是构造一个特殊的“101 Switching Protocols”响应报文，通知客户端，接下来就不用HTTP了，全改用WebSocket协议通信。（有点像TLS的“Change Cipher Spec”）</p><p>WebSocket的握手响应报文也是有特殊格式的，要用字段“Sec-WebSocket-Accept”验证客户端请求报文，同样也是为了防止误连接。</p><p>具体的做法是把请求头里“Sec-WebSocket-Key”的值，加上一个专用的UUID “258EAFA5-E914-47DA-95CA-C5AB0DC85B11”，再计算SHA-1摘要。</p><pre><code>encode_base64(
  sha1( 
    Sec-WebSocket-Key + '258EAFA5-E914-47DA-95CA-C5AB0DC85B11' ))
</code></pre><p>客户端收到响应报文，就可以用同样的算法，比对值是否相等，如果相等，就说明返回的报文确实是刚才握手时连接的服务器，认证成功。</p><p>握手完成，后续传输的数据就不再是HTTP报文，而是WebSocket格式的二进制帧了。</p><p><img src="https://static001.geekbang.org/resource/image/84/03/84e9fa337f2b4c2c9f14760feb41c903.png?wh=1254*707" alt=""></p><h2>小结</h2><p>浏览器是一个“沙盒”环境，有很多的限制，不允许建立TCP连接收发数据，而有了WebSocket，我们就可以在浏览器里与服务器直接建立“TCP连接”，获得更多的自由。</p><p>不过自由也是有代价的，WebSocket虽然是在应用层，但使用方式却与“TCP Socket”差不多，过于“原始”，用户必须自己管理连接、缓存、状态，开发上比HTTP复杂的多，所以是否要在项目中引入WebSocket必须慎重考虑。</p><ol>
<li><span class="orange">HTTP的“请求-应答”模式不适合开发“实时通信”应用，效率低，难以实现动态页面，所以出现了WebSocket；</span></li>
<li><span class="orange">WebSocket是一个“全双工”的通信协议，相当于对TCP做了一层“薄薄的包装”，让它运行在浏览器环境里；</span></li>
<li><span class="orange">WebSocket使用兼容HTTP的URI来发现服务，但定义了新的协议名“ws”和“wss”，端口号也沿用了80和443；</span></li>
<li><span class="orange">WebSocket使用二进制帧，结构比较简单，特殊的地方是有个“掩码”操作，客户端发数据必须掩码，服务器则不用；</span></li>
<li><span class="orange">WebSocket利用HTTP协议实现连接握手，发送GET请求要求“协议升级”，握手过程中有个非常简单的认证机制，目的是防止误连接。</span></li>
</ol><h2>课下作业</h2><ol>
<li>WebSocket与HTTP/2有很多相似点，比如都可以从HTTP/1升级，都采用二进制帧结构，你能比较一下这两个协议吗？</li>
<li>试着自己解释一下WebSocket里的”Web“和”Socket“的含义。</li>
<li>结合自己的实际工作，你觉得WebSocket适合用在哪些场景里？</li>
</ol><p>欢迎你把自己的学习体会写在留言区，与我和其他同学一起讨论。如果你觉得有所收获，也欢迎把文章分享给你的朋友。</p><p><img src="https://static001.geekbang.org/resource/image/4b/5b/4b81de6b5c57db92ed7808344482ef5b.png?wh=1769*3348" alt="unpreview"></p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4d/fd/0aa0e39f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>许童童</span>
  </div>
  <div class="_2_QraFYR_0">思考题：<br>1.WebSocket 和 HTTP&#47;2 都是用来弥补HTTP协议的一些缺陷和不足，WebSocket 主要解决双向通信、全双工问题，HTTP&#47;2 主要解决传输效率的问题，两者在二进制帧的格式上也不太一样，HTTP&#47;2 有多路复用、优先级和流的概念。<br><br>2.试着自己解释一下 WebSocket 里的”Web“和”Socket“的含义。<br>Web就是HTTP的意思，Socket就是网络编程里的套接字，也就是HTTP协议上的网络套接字，可以任意双向通信。<br><br>3.结合自己的实际工作，你觉得 WebSocket 适合用在哪些场景里？<br>IM通信，实时互动，回调响应，数据实时同步。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-23 11:59:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTI3F4IdQuDZrhN8ThibP85eCiaSWTYpTrcC6QB9EoAkw3IIj6otMibb1CgrS1uzITAnJmGLXQ2tgIkAQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>cugphoenix</span>
  </div>
  <div class="_2_QraFYR_0">是不是可以这样理解：HTTP是基于TCP的，通过TCP收发的消息用HTTP的应用层协议解析。WebSocket是首先通过HTTP协议把TCP链接建好，然后通过Upgrade字段进行协议转换，在收到服务器的101 Switching Protocols应答之后，后续的TCP消息就通过WebSocket协议解析。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 理解的基本正确，把WebSocket和http、tcp的关系理顺了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-07 22:10:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/9b/a8/6a391c66.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Leon📷</span>
  </div>
  <div class="_2_QraFYR_0">工作场景遇到过用户订阅股票的股价，股价波动时实时推送给海量订阅的用户，面试场景被问到两次，一 千万粉丝的明星发布动态如何推送给粉丝 二 海量用户的主播直播如何推送弹幕 当时回答消息队列，其实web socket才是比较好的方案</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: WebSocket适合实时通信交互的场景，和消息队列其实是两个领域，不冲突，可以互相结合使用。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-23 06:42:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/1d/64/52a5863b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>大土豆</span>
  </div>
  <div class="_2_QraFYR_0">WebSocket定义为tcp over web，我个人感觉是不妥的，应该是&quot;可靠有序的传输层 + 实现组包协议的应用层的长连接方案 over web&quot;。WebSocket和HTTP已经有包的结构了,业务直接可以用了，浏览器A发一个websocket包，比如说数据是1234，给服务器，服务器可以获取这个包，数据是1234，和http一样了。而tcp还是原始的没头没尾的字节流，想要通讯，还得再自定义一个应用层的协议。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 说的很对。<br><br>正文里的tcp over web只是一种比喻，强调了WebSocket与http的区别，与tcp的相似性，不是那么严谨。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-28 12:37:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/ba/bf/e9a44c63.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>chao</span>
  </div>
  <div class="_2_QraFYR_0">1、第二个字节后 7 位是“Payload len”，表示帧内容的长度。它是另一种变长编码，最少 7 位，最多是 7+64 位，也就是额外增加 8 个字节，所以一个 WebSocket 帧最大是 2^64。<br>2、如果数据的长度小于等于125个字节，则用默认的7个bit来标示数据的长度；<br>如果数据的长度为126个字节，则用后面相邻的2个字节来保存一个16bit位的无符号整数作为数据的长度；<br>如果数据的长度大于等于127个字节，则用后面相邻的8个字节来保存一个64bit位的无符号整数作为数据的长度；<br>老师，2是其它地方看到的，Payload len 这样设计的原因是什么，以及没明白为啥126个字节的长度要用16bit来表示</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我个人也觉得WebSocket的变长编码设计的很奇怪。<br><br>第二个字节最高位被mask占用，所以低7位表示长度，最多127。<br><br>那么125一下在低7位就够了，126用作标志位，表示后续使用两个字节，127又是另外一个标志位，表示后面是四个字节。<br><br>所以超过125后低7位就不再是长度的含义了，而是标志位：126=&gt;2 bytes， 127=&gt; 4 bytes。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-18 22:09:03</div>
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
  <div class="_2_QraFYR_0">两年前我实习的时候公司项目用过，到现在我才搞清楚和http的区别，惭愧…</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这两个确实很像，第一次接触的人（包括我）也是容易弄糊涂。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-10 23:45:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/23/40/c8/17af3598.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Evan Xia</span>
  </div>
  <div class="_2_QraFYR_0">我们的业务场景是在下黑白棋的过程双方都能实时收到对方的落子， 用的是一个封装好的Centrifugo库，期间遇到最多的就是网络不好重连的问题</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复:  这个场景用websocket还是挺合适的，不过网络的问题就是协议之外的事情了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-20 14:18:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/e8/91/e05a03a0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ccx</span>
  </div>
  <div class="_2_QraFYR_0">在 FINTECH 领域工作了几年了，自研发的外汇&#47;数字货币交易系统的行情模块基本都是 websocket 实现的；另外还遇到一个有意思的场景，就是 discord 的机器人 Slash Commands 的实现也是基于 websocket  的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: websocket在如今的互联网大环境下非常有用，基于http握手建连让它可以很容易运行在各种web服务上。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-30 12:20:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/ed/a9/662318ab.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我母鸡啊！</span>
  </div>
  <div class="_2_QraFYR_0">3.结合自己的实际工作，你觉得 WebSocket 适合用在哪些场景里？<br>最近在做直播功能的项目，就用到了websovket的去做用户评论的推送，观看人数等的功能。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: nice</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-24 20:49:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_5b0e47</span>
  </div>
  <div class="_2_QraFYR_0">向客户端监控屏推送时实更新数据，可以使用websocket</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: nice，实时推送用WebSocket很好使。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-27 15:10:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/60/4f/db0e62b3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Daiver</span>
  </div>
  <div class="_2_QraFYR_0">还有socket.io，算是websocket的超集了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: socket.io是一个开发框架，而WebSocket是传输协议，两者虽然有联系，但不能混在一起。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-23 17:32:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/7b/f0/269139d5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Cris</span>
  </div>
  <div class="_2_QraFYR_0">老师，我想问下，uri里的端口号，有什么用？为什么它是和协议对应的（http默认80，https默认443），却又写在域名的后面？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以参考一下第6讲，端口号是跟tcp协议相关的概念。<br><br>因为域名实际上是ip地址的等价替换，所以端口号就可以跟在域名后面。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-23 08:58:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/75/00/618b20da.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>徐海浪</span>
  </div>
  <div class="_2_QraFYR_0">1. WebSocket 与 HTTP&#47;2 有很多相似点，比如都可以从 HTTP&#47;1 升级，都采用二进制帧结构，你能比较一下这两个协议吗？<br>差别：HTTP&#47;2是请求与响应的模式，而WebSocket是双向的，服务器也可以主动向客户端发起请求。<br>2. 试着自己解释一下 WebSocket 里的”Web“和”Socket“的含义。<br>是基于web服务器，类似于tcp的socket方式来使用的协议。<br>3. 结合自己的实际工作，你觉得 WebSocket 适合用在哪些场景里？<br>我在实际工作中还没有用到WebSocket，觉得适合服务器主动推送的客户端的场景，比如站内信或者站内聊天，或者在线页游？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.在WebSocket里没有请求响应的概念，收发的都是数据帧，通信的双方可以自己解释帧的含义。<br><br>2.应该是基于web，也就是http协议。<br><br>3.对。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-23 08:27:33</div>
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
  <div class="_2_QraFYR_0">老师好!websocket单机服务器能支持多少链接啊?之前没用过websocket。看帖子好像是通过key-value形式存储所有链接。需要用得时候通过key拿到链接往外写数据。希望老师科普下web socket的简单应用和实现，性能分析。<br>需要服务器主动推的感觉都可以用websocket做。<br>聊天工具:用户A，用户B，<br>A-&gt;服务器(保存聊天记录)-&gt;B;B-&gt;服务器(保存聊天记录)-&gt;A;是这样么?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>WebSocket其实就是给tcp加了一层简单的包装，所以它的并发能力取决于服务器，并不是kv的形式，你应该把它理解成运行在http上的tcp，用tcp的思路去考虑它。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-23 08:00:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/fa/5d/735fdc76.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>╭(╯ε╰)╮</span>
  </div>
  <div class="_2_QraFYR_0">websocket包装成http协议不应该就是按照http协议的“请求-应答”模式收发数据了嘛，还是没理解怎么实现的全双工？服务器向客户端推消息先封装成websocket帧，帧再模拟成fttp协议，然后http协议不支持推送，这不是没办法进行下去了嘛？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: websocket只是在握手建连阶段使用的http，之后就完全是自己的协议了，与http没有关系，通信方式是和tcp一样的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-20 18:24:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/fa/5d/735fdc76.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>╭(╯ε╰)╮</span>
  </div>
  <div class="_2_QraFYR_0">有个问题 30课介绍http2的时候不能重复使用443端口，所以重新用了8443端口。这节课websocket为什么就能跟http一起复用443端口呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: websocket基于http&#47;1.1，所以它可以和443，也就是https在一起。<br><br>而http&#47;2与http&#47;1.1完全不同，就没有办法用同一个端口。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-20 18:17:39</div>
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
  <div class="_2_QraFYR_0">请问下老师，握手的请求只能用 GET 吗？能用POST吗？如果不能是因为什么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个是websocket协议的规定，必须用这个。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-09 19:26:53</div>
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
  <div class="_2_QraFYR_0">第二个字节后 7 位是“Payload len”，表示帧内容的长度。它是另一种变长编码，最少 7 位，最多是 7+64 位，也就是额外增加 8 个字节，所以一个 WebSocket 帧最大是 2^64。——这里最大的帧为什么不说是2^71呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 前面的7位是标志位，已经被占用了，所以只能用64位。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-22 13:38:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/f2/f5/b82f410d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Unknown element</span>
  </div>
  <div class="_2_QraFYR_0">老师问下<br>1.Web socket既然没有流的概念那具体是怎么实现全双工的呢  两边都同时收发那包的顺序就乱了吧<br>2.Web socket帧里帧长度最大是7+64位那最长不应该是2^71位吗  为什么课程里说最长是2^64？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.websocket本质上和tcp是一样的，双工是指通信的双方可以同时发送数据，但一个方向上的数据包还是有序的。<br><br>2.注意它用了变长编码，超过127才会用8个字节来表示长度，所以最大长度是2^64。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-04 00:22:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c0/5c/10111544.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张峰</span>
  </div>
  <div class="_2_QraFYR_0">服务器发送事件（Server-sent Events）是基于 WebSocket 协议的一种服务器向客户端发送事件和数据的单向通讯。<br>HTML5 服务器发送事件（server-sent event）允许网页获得来自服务器的更新。<br><br>老师能讲解一下 Server-sent Events 和 websocket的相同和不同。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对这两个不是太了解，感觉像是js里的编程概念吧，与传输协议本身没有关系。<br><br>抱歉了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-20 15:34:47</div>
  </div>
</div>
</div>
</li>
</ul>