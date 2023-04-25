<audio title="28 _ 连接太慢该怎么办：HTTPS的优化" src="https://static001.geekbang.org/resource/audio/3e/02/3e5a61bf873fe75c1dc93dfc5c08d602.mp3" controls="controls"></audio> 
<p>你可能或多或少听别人说过，“HTTPS的连接很慢”。那么“慢”的原因是什么呢？</p><p>通过前两讲的学习，你可以看到，HTTPS连接大致上可以划分为两个部分，第一个是建立连接时的<strong>非对称加密握手</strong>，第二个是握手后的<strong>对称加密报文传输</strong>。</p><p>由于目前流行的AES、ChaCha20性能都很好，还有硬件优化，报文传输的性能损耗可以说是非常地小，小到几乎可以忽略不计了。所以，通常所说的“HTTPS连接慢”指的就是刚开始建立连接的那段时间。</p><p>在TCP建连之后，正式数据传输之前，HTTPS比HTTP增加了一个TLS握手的步骤，这个步骤最长可以花费两个消息往返，也就是2-RTT。而且在握手消息的网络耗时之外，还会有其他的一些“隐形”消耗，比如：</p><ul>
<li>产生用于密钥交换的临时公私钥对（ECDHE）；</li>
<li>验证证书时访问CA获取CRL或者OCSP；</li>
<li>非对称加密解密处理“Pre-Master”。</li>
</ul><p>在最差的情况下，也就是不做任何的优化措施，HTTPS建立连接可能会比HTTP慢上几百毫秒甚至几秒，这其中既有网络耗时，也有计算耗时，就会让人产生“打开一个HTTPS网站好慢啊”的感觉。</p><p>不过刚才说的情况早就是“过去时”了，现在已经有了很多行之有效的HTTPS优化手段，运用得好可以把连接的额外耗时降低到几十毫秒甚至是“零”。</p><!-- [[[read_end]]] --><p>我画了一张图，把TLS握手过程中影响性能的部分都标记了出来，对照着它就可以“有的放矢”地来优化HTTPS。</p><p><img src="https://static001.geekbang.org/resource/image/c4/ed/c41da1f1b1bdf4dc92c46330542c5ded.png?wh=2017*1754" alt=""></p><h2>硬件优化</h2><p>在计算机世界里的“优化”可以分成“硬件优化”和“软件优化”两种方式，先来看看有哪些硬件的手段。</p><p>硬件优化，说白了就是“花钱”。但花钱也是有门道的，要“有钱用在刀刃上”，不能大把的银子撒出去“只听见响”。</p><p>HTTPS连接是计算密集型，而不是I/O密集型。所以，如果你花大价钱去买网卡、带宽、SSD存储就是“南辕北辙”了，起不到优化的效果。</p><p>那该用什么样的硬件来做优化呢？</p><p>首先，你可以选择<strong>更快的CPU</strong>，最好还内建AES优化，这样即可以加速握手，也可以加速传输。</p><p>其次，你可以选择“<strong>SSL加速卡</strong>”，加解密时调用它的API，让专门的硬件来做非对称加解密，分担CPU的计算压力。</p><p>不过“SSL加速卡”也有一些缺点，比如升级慢、支持算法有限，不能灵活定制解决方案等。</p><p>所以，就出现了第三种硬件加速方式：“<strong>SSL加速服务器</strong>”，用专门的服务器集群来彻底“卸载”TLS握手时的加密解密计算，性能自然要比单纯的“加速卡”要强大的多。</p><h2>软件优化</h2><p>不过硬件优化方式中除了CPU，其他的通常可不是靠简单花钱就能买到的，还要有一些开发适配工作，有一定的实施难度。比如，“加速服务器”中关键的一点是通信必须是“异步”的，不能阻塞应用服务器，否则加速就没有意义了。</p><p>所以，软件优化的方式相对来说更可行一些，性价比高，能够“少花钱，多办事”。</p><p>软件方面的优化还可以再分成两部分：一个是<strong>软件升级</strong>，一个是<strong>协议优化</strong>。</p><p>软件升级实施起来比较简单，就是把现在正在使用的软件尽量升级到最新版本，比如把Linux内核由2.x升级到4.x，把Nginx由1.6升级到1.16，把OpenSSL由1.0.1升级到1.1.0/1.1.1。</p><p>由于这些软件在更新版本的时候都会做性能优化、修复错误，只要运维能够主动配合，这种软件优化是最容易做的，也是最容易达成优化效果的。</p><p>但对于很多大中型公司来说，硬件升级或软件升级都是个棘手的问题，有成千上万台各种型号的机器遍布各个机房，逐一升级不仅需要大量人手，而且有较高的风险，可能会影响正常的线上服务。</p><p>所以，在软硬件升级都不可行的情况下，我们最常用的优化方式就是在现有的环境下挖掘协议自身的潜力。</p><h2>协议优化</h2><p>从刚才的TLS握手图中你可以看到影响性能的一些环节，协议优化就要从这些方面着手，先来看看核心的密钥交换过程。</p><p>如果有可能，应当尽量采用TLS1.3，它大幅度简化了握手的过程，完全握手只要1-RTT，而且更加安全。</p><p>如果暂时不能升级到1.3，只能用1.2，那么握手时使用的密钥交换协议应当尽量选用椭圆曲线的ECDHE算法。它不仅运算速度快，安全性高，还支持“False Start”，能够把握手的消息往返由2-RTT减少到1-RTT，达到与TLS1.3类似的效果。</p><p>另外，椭圆曲线也要选择高性能的曲线，最好是x25519，次优选择是P-256。对称加密算法方面，也可以选用“AES_128_GCM”，它能比“AES_256_GCM”略快一点点。</p><p>在Nginx里可以用“ssl_ciphers”“ssl_ecdh_curve”等指令配置服务器使用的密码套件和椭圆曲线，把优先使用的放在前面，例如：</p><pre><code>ssl_ciphers   TLS13-AES-256-GCM-SHA384:TLS13-CHACHA20-POLY1305-SHA256:EECDH+CHACHA20；
ssl_ecdh_curve              X25519:P-256;
</code></pre><h2>证书优化</h2><p>除了密钥交换，握手过程中的证书验证也是一个比较耗时的操作，服务器需要把自己的证书链全发给客户端，然后客户端接收后再逐一验证。</p><p>这里就有两个优化点，一个是<strong>证书传输</strong>，一个是<strong>证书验证</strong>。</p><p>服务器的证书可以选择椭圆曲线（ECDSA）证书而不是RSA证书，因为224位的ECC相当于2048位的RSA，所以椭圆曲线证书的“个头”要比RSA小很多，即能够节约带宽也能减少客户端的运算量，可谓“一举两得”。</p><p>客户端的证书验证其实是个很复杂的操作，除了要公钥解密验证多个证书签名外，因为证书还有可能会被撤销失效，客户端有时还会再去访问CA，下载CRL或者OCSP数据，这又会产生DNS查询、建立连接、收发数据等一系列网络通信，增加好几个RTT。</p><p>CRL（Certificate revocation list，证书吊销列表）由CA定期发布，里面是所有被撤销信任的证书序号，查询这个列表就可以知道证书是否有效。</p><p>但CRL因为是“定期”发布，就有“时间窗口”的安全隐患，而且随着吊销证书的增多，列表会越来越大，一个CRL经常会上MB。想象一下，每次需要预先下载几M的“无用数据”才能连接网站，实用性实在是太低了。</p><p>所以，现在CRL基本上不用了，取而代之的是OCSP（在线证书状态协议，Online Certificate Status Protocol），向CA发送查询请求，让CA返回证书的有效状态。</p><p>但OCSP也要多出一次网络请求的消耗，而且还依赖于CA服务器，如果CA服务器很忙，那响应延迟也是等不起的。</p><p>于是又出来了一个“补丁”，叫“OCSP Stapling”（OCSP装订），它可以让服务器预先访问CA获取OCSP响应，然后在握手时随着证书一起发给客户端，免去了客户端连接CA服务器查询的时间。</p><h2>会话复用</h2><p>到这里，我们已经讨论了四种HTTPS优化手段（硬件优化、软件优化、协议优化、证书优化），那么，还有没有其他更好的方式呢？</p><p>我们再回想一下HTTPS建立连接的过程：先是TCP三次握手，然后是TLS一次握手。这后一次握手的重点是算出主密钥“Master Secret”，而主密钥每次连接都要重新计算，未免有点太浪费了，如果能够把“辛辛苦苦”算出来的主密钥缓存一下“重用”，不就可以免去了握手和计算的成本了吗？</p><p>这种做法就叫“<strong>会话复用</strong>”（TLS session resumption），和HTTP Cache一样，也是提高HTTPS性能的“大杀器”，被浏览器和服务器广泛应用。</p><p>会话复用分两种，第一种叫“<strong>Session ID</strong>”，就是客户端和服务器首次连接后各自保存一个会话的ID号，内存里存储主密钥和其他相关的信息。当客户端再次连接时发一个ID过来，服务器就在内存里找，找到就直接用主密钥恢复会话状态，跳过证书验证和密钥交换，只用一个消息往返就可以建立安全通信。</p><p>实验环境的端口441实现了“Session ID”的会话复用，你可以访问URI<br>
“<a href="https://www.chrono.com:441/28-1">https://www.chrono.com:441/28-1</a>”，刷新几次，用Wireshark抓包看看实际的效果。</p><pre><code>Handshake Protocol: Client Hello
    Version: TLS 1.2 (0x0303)
    Session ID: 13564734eeec0a658830cd…
    Cipher Suites Length: 34


Handshake Protocol: Server Hello
    Version: TLS 1.2 (0x0303)
    Session ID: 13564734eeec0a658830cd…
    Cipher Suite: TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384 (0xc030)
</code></pre><p>通过抓包可以看到，服务器在“ServerHello”消息后直接发送了“Change Cipher Spec”和“Finished”消息，复用会话完成了握手。</p><p><img src="https://static001.geekbang.org/resource/image/12/ac/125fe443a147ed38a97a4492045d98ac.png?wh=2004*2892" alt=""></p><h2>会话票证</h2><p>“Session ID”是最早出现的会话复用技术，也是应用最广的，但它也有缺点，服务器必须保存每一个客户端的会话数据，对于拥有百万、千万级别用户的网站来说存储量就成了大问题，加重了服务器的负担。</p><p>于是，又出现了第二种“<strong>Session Ticket</strong>”方案。</p><p>它有点类似HTTP的Cookie，存储的责任由服务器转移到了客户端，服务器加密会话信息，用“New Session Ticket”消息发给客户端，让客户端保存。</p><p>重连的时候，客户端使用扩展“<strong>session_ticket</strong>”发送“Ticket”而不是“Session ID”，服务器解密后验证有效期，就可以恢复会话，开始加密通信。</p><p>这个过程也可以在实验环境里测试，端口号是442，URI是“<a href="https://www.chrono.com:442/28-1">https://www.chrono.com:442/28-1</a>”。</p><p>不过“Session Ticket”方案需要使用一个固定的密钥文件（ticket_key）来加密Ticket，为了防止密钥被破解，保证“前向安全”，密钥文件需要定期轮换，比如设置为一小时或者一天。</p><h2>预共享密钥</h2><p>“False Start”“Session ID”“Session Ticket”等方式只能实现1-RTT，而TLS1.3更进一步实现了“<strong>0-RTT</strong>”，原理和“Session Ticket”差不多，但在发送Ticket的同时会带上应用数据（Early Data），免去了1.2里的服务器确认步骤，这种方式叫“<strong>Pre-shared Key</strong>”，简称为“PSK”。</p><p><img src="https://static001.geekbang.org/resource/image/11/ab/119cfd261db49550411a12b1f6d826ab.png?wh=2001*1446" alt=""></p><p>但“PSK”也不是完美的，它为了追求效率而牺牲了一点安全性，容易受到“重放攻击”（Replay attack）的威胁。黑客可以截获“PSK”的数据，像复读机那样反复向服务器发送。</p><p>解决的办法是只允许安全的GET/HEAD方法（参见<a href="https://time.geekbang.org/column/article/101518">第10讲</a>），在消息里加入时间戳、“nonce”验证，或者“一次性票证”限制重放。</p><h2>小结</h2><ol>
<li><span class="orange">可以有多种硬件和软件手段减少网络耗时和计算耗时，让HTTPS变得和HTTP一样快，最可行的是软件优化；</span></li>
<li><span class="orange">应当尽量使用ECDHE椭圆曲线密码套件，节约带宽和计算量，还能实现“False Start”；</span></li>
<li><span class="orange"> 服务器端应当开启“OCSP Stapling”功能，避免客户端访问CA去验证证书；</span></li>
<li><span class="orange">会话复用的效果类似Cache，前提是客户端必须之前成功建立连接，后面就可以用“Session ID”“Session Ticket”等凭据跳过密钥交换、证书验证等步骤，直接开始加密通信。</span></li>
</ol><h2>课下作业</h2><ol>
<li>你能比较一下“Session ID”“Session Ticket”“PSK”这三种会话复用手段的异同吗？</li>
<li>你觉得哪些优化手段是你在实际工作中能用到的？应该怎样去用？</li>
</ol><p>欢迎你把自己的学习体会写在留言区，与我和其他同学一起讨论。如果你觉得有所收获，也欢迎把文章分享给你的朋友。</p><p><img src="https://static001.geekbang.org/resource/image/a2/ab/a251606fb0637c6db45b7fd6660af5ab.png?wh=1769*3265" alt="unpreview"></p><p></p>
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
  <div class="_2_QraFYR_0">Session ID:会话复用压力在服务端<br>Session Ticket:压力在客户端，客户端不安全所以要频繁换密钥文件<br>PSK:验证阶段把数据也带上，少一次请求<br>前两个都是缓存复用的思想，重用之前计算好的结果，达到降低CPU的目的。第三个就是少一次链接减少网络开销。<br>感觉都可以把开销大的东西缓存起来复用，缓存真个好东西，空间局部命中和时间局部命中定理太牛逼了。不过最关键的还是找性能瓶颈，精确定位性能瓶颈比较重要，然后针对瓶颈优化，空间换时间或者时间换空间。这个时候算法的价值就体现出来了。可惜这些我都不会╯﹏╰</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: psk实际上是Session Ticket的强化版，本身也是缓存，但它简化了Session Ticket的协商过程，省掉了一次RTT。<br><br>多在实践中学，多看些开源项目，就能逐渐掌握了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-31 09:15:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/f4/87/644c0c5d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>俊伟</span>
  </div>
  <div class="_2_QraFYR_0">session id 把会话信息存在服务端<br>session ticket 把会话信息存在客户端<br>psk 在session ticket 的基础上，使用early data顺便再发送一下服务端的数据</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 态度认真、积极，good。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-17 10:32:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/0d/40/f70e5653.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>前端西瓜哥</span>
  </div>
  <div class="_2_QraFYR_0">1. 你能比较一下“Session ID”“Session Ticket”“PSK”这三种会话复用手段的异同吗？<br><br>答：<br>（1）Session ID 类似网站开发中用来验证用户的 cookie，服务器会保存 Session ID对应的主密钥，需要用到服务器的存储空间。<br>（2）Session Ticket 貌似有点类似网站开发中的 JWT（JSON Web Token），JWT的做法是服务器将必要的信息（主密钥和过期时间）加上密钥进行 HMAC 加密，然后将生成的密文和原文相连得到 JWT 字符串，交给客户端。当客户端发送 JWT 给服务端后，服务器会取出其中的原文和自己的密钥进行 HMAC 运算，如果得到的结果和 JWT 中的密文一样，就说明是服务端颁发的 JWT，服务器就会认为 JWT 存储 的主密钥和有效时间是有效的。另外，JWT 中不应该存放用户的敏感信息，明文部分任何人可见（不知道 Session Ticket 的实现是不是也是这样？）<br>（3）PSK 不是很懂，貌似是在 tcp 握手的时候，就直接给出了 Ticket（可是这样 Ticket 好像没有加密呢）。<br><br>总的来说，Session ID 需要服务器来存储会话；而 Session Ticket 则不需要服务器使用存储空间，但要保护好密钥。另外为了做到“前向安全”，需要经常更换密钥。PSK相比 Session Tick，直接在第一次握手时，就将 ticket 发送过去了，减少了握手次数。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 说的挺好，PSK其实就是Session Ticket的强化版，也有ticket，但应用数据随ticket一起发给服务器。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-01 01:36:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/26/7e/823c083e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Wr</span>
  </div>
  <div class="_2_QraFYR_0">1、相同点：都是会话复用技术<br>区别：<br>Seesion ID：会话数据缓存在服务端，如果服务器客户量大，对服务器会造成很大压力<br>Seeion Ticket：会话数据缓存在客户端<br>PAK：在Seesion Ticket的基础上，应用数据和Session Ticket一起发送给服务器，省去了中间服务器与客户端的确认步骤<br><br>2、暂无</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 居然是三连……</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-14 22:12:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/99/af/d29273e2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>饭粒</span>
  </div>
  <div class="_2_QraFYR_0">1.抓包看了下 442 ，复用会话时 Client Hello session_ticket 确实有数据，Sessioin Id 好像也还会复用。<br>Transport Layer Security<br>    TLSv1.2 Record Layer: Handshake Protocol: Client Hello<br>        Content Type: Handshake (22)<br>        Version: TLS 1.0 (0x0301)<br>        Length: 512<br>        Handshake Protocol: Client Hello<br>            ...<br>            Random: 9474888cafdce89fd32eac247a8b464f842efbac706d8930…<br>            Session ID Length: 32<br>            Session ID: a4a0caef10dee7a6f44aa522a35f6c799101d5eced01eb32…   # Session ID<br>            ...<br>            Extension: session_ticket (len=192)     # session_ticket<br>                Type: session_ticket (35)<br>                Length: 192<br>                Data (192 bytes)<br>            ...<br><br>Transport Layer Security<br>    TLSv1.2 Record Layer: Handshake Protocol: Server Hello<br>        Content Type: Handshake (22)<br>        Version: TLS 1.2 (0x0303)<br>        Length: 100<br>        Handshake Protocol: Server Hello<br>            ...<br>            Random: e82519e9e8bfcbd40e1da7a202bd50ff993d5ef0cbc33378…<br>            Session ID Length: 32<br>            Session ID: a4a0caef10dee7a6f44aa522a35f6c799101d5eced01eb32…   # Session ID<br>            Cipher Suite: TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384 (0xc030)<br>            ...<br>    TLSv1.2 Record Layer: Change Cipher Spec Protocol: Change Cipher Spec<br>    TLSv1.2 Record Layer: Handshake Protocol: Finished   <br><br>2.有个疑问：会话复用技术，保存会话数据的一端使用对端传过来的 Sessin Id 查询到之前的 master secret，但它如何安全的把这个 master secret 传递给对端（对端应该只有 Sessin Id 吧）？<br><br>3.抓包过程用 Chrome 发请求，前三次 Client Hello, Server Hello 握手过程都因为 Alert (Level: Fatal, Description: Certificate Unknown) 失败了，第四次才成功。 而使用 FireFox 发请则不会出现。这是因为 Chrome 内置的证书更少吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.有ticket就不会用id。<br><br>2.master secret都保存在两端各自的内存里，不需要，也不允许在网络上传递，两端用id对一下，一致就行了。<br><br>3.实验环境的证书是自签名证书，需要提前信任才行，不然就会告警。这个跟浏览器的安全策略有关，可能Firefox的处理逻辑不一样。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-05 11:51:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/9d/a4/e481ae48.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lesserror</span>
  </div>
  <div class="_2_QraFYR_0">老师，以下问题麻烦请回答一下：<br>1.客户端有时还会再去访问 CA，下载 CRL 或者 OCSP 数据，这又会产生 DNS 查询、建立连接、收发数据等一系列网络通信，增加好几个 RTT。这个CRL 或者 OCSP是对应到某个网址上面的嘛？客户端根据网址访问？<br><br>2.它可以让服务器预先访问 CA 获取 OCSP 响应，然后在握手时随着证书一起发给客户端，免去了客户端连接 CA 服务器查询的时间。这里不是客户端自己去验证的会不会有问题？服务器自己代做了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.crl和ocsp是一个很大的列表，包含所有过期或者作废的证书序列号，不关联到具体的网址。客户端只要在这个列表里查一下证书序列号是否在这里面就行。而证书里面是包含网址的。<br><br>2.ocsp都是经过ca签名的，所以不会被窜改，保证肯定是ca发出的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-21 10:28:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/26/eb/d7/90391376.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ifelse</span>
  </div>
  <div class="_2_QraFYR_0">good good study，day day up.</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: nice</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-31 19:01:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/6c/d4/85ef1463.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>路漫漫</span>
  </div>
  <div class="_2_QraFYR_0">老师，文章里说，这后一次握手的重点是算出主密钥“Master Secret”，而主密钥每次连接都要重新计算，未免有点太浪费了。难道每次连接的主密钥是一样的？不是每次连接的ecdhe的公钥私钥都是不同的吗？怎么还可以缓存呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: ecdhe的公钥私钥每次都是临时生成的，但它只是用在握手过程中。<br><br>而master secret是用在会话过程中的，由client random&amp;#47;server random&amp;#47;pre-master生成的，而这三个都是随机数，所以master secret必然每次都不同。<br><br>但算master secret需要握手，成本高，所以为了提高效率，就可以用session id、session ticket来复用，并不是简单的缓存。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-23 19:33:08</div>
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
  <div class="_2_QraFYR_0">1：你能比较一下“Session ID”“Session Ticket”“PSK”这三种会话复用手段的异同吗？<br>1-1：回话复用<br>核心是缓存主密钥，为啥要缓存？因为计算出主密钥比较费劲，如果能重复利用，重复计算的活就免了，这是在拿空间换时间。<br>1-2：Session ID<br>可以认为是缓存主密钥的key，在客户端和服务器都有存储，通过传递这个key来获取和重用主密钥。<br>缺点是太费服务器的村村资源，因为每一个客户端的回话数据都需要保存，如果客户端有百万甚至千万基本，那存储空间使用的就有些多啦！<br>1-3：Session ticket<br>这个方案可以解决服务器存储空间压力山大的问题，核心是把信息放在客户端存储，当然是加密后的信息，服务器侧需要解码后再使用。<br>1-44：PSK<br>和Session ticket类似，为了效率加大了一些安全风险，ticket中带上了应用数据的信息，这样能省去服务器的确认步骤。为了加强安全性，使用上做了一些限制。<br>有此可见，没有完美的解决方案，具体想要什么需要自己权衡对待。想要实现多快好省，那要么是一句口号，要么必须付出其他的代价。<br><br>2：你觉得哪些优化手段是你在实际工作中能用到的？应该怎样去用？<br>软件优化这个估计最常用，也能用到，一些安全漏洞或性能优化常这么玩。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: psk理解的稍微有点偏差，ticket里是加密的会话密钥，应用数据在psk后面。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-04 13:06:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/53/03/24a892b7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>书生依旧</span>
  </div>
  <div class="_2_QraFYR_0">PSK 在发送 Ticket 的同时会带上应用数据，免去了 1.2 里面的服务器确认步骤。<br>这句话有点不太理解，请问老师：<br>1. 看图上 pre_shared_key 是在 Hello 中发送的，Session Ticket 也是在 Hello 中发送的吗？<br>2. 带上应用数据是什么数据?<br>3. 1.2 里面的服务器验证指的是哪个步骤？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.是的，hello消息里有个pre_share_key扩展，就是ticket。<br><br>2.在https里应用数据就是http报文了。<br><br>3.在tls1.2的会话复用里，必须在服务器发送server hello确认建立加密连接之后才能发送应用数据，而tls1.3就不需要这个确认步骤。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-16 10:53:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_11246e</span>
  </div>
  <div class="_2_QraFYR_0">请问异步通信可以讲下吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: http协议里没有涉及异步通信吧，不知道是哪方面的内容，sorry。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-28 11:02:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/ojfRyNRvy1x3Mia0nssz6CNPHrHXwPPmibvds1URgoHQuKXrGiaxrEbsT6sAvuK4N4AOicySh8S9iaWcOLjteOl6Kgg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>泥鳅儿</span>
  </div>
  <div class="_2_QraFYR_0">老师，如果安装的ssl证书过期了，之后进行的http数据传输还是加密的吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 证书过期前面的握手步骤就通不过，也不会有后面的加密过程。<br>不过证书的过期其实与密码学无关，这点要理解。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-09 17:15:16</div>
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
  <div class="_2_QraFYR_0">老师请问下    非对称加密解密处理“Pre-Master” 这计划的意思就是生成 “Pre-Master” 的意思么</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对，因为交换pre-master需要非对称算法的参与。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-27 19:39:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKiay65IMyD82E59Xnbp370ChMG3N9XQuXoKwfhZJ19zotzKMlJhwzBDxE61bp26jdkf54NY9L41yg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_5227ac</span>
  </div>
  <div class="_2_QraFYR_0">罗老师，本节最后提到PSK有遭重放隐患，但为什么TLS开始握手阶段的明文传输没有重放危险呢，是因为有携带应用数据才谈重放吗？另外HTTPS是先有tcp连接了再来TLS握手，那有没可能黑客通过不断地先进行TCP连接再TLS握手来很快消耗掉服务器提供正常服务的连接资源？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.握手时每次都是随机生成的密钥，不重复，所以不会有重放的危险。而psk直接放行，就容易被冒充。<br><br>2.这个就是dos攻击了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-24 11:05:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/86/e3/a31f6869.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span> 尿布</span>
  </div>
  <div class="_2_QraFYR_0">服务器端应当开启“OCSP Stapling”功能，避免客户端访问CA去验证证书<br><br>在Nginx里可以用指令“ssl_stapling on”开启“OCSP Stapling”，而在OpenResty里更可以编写Lua代码灵活定制</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-10 10:44:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_e4a9c5</span>
  </div>
  <div class="_2_QraFYR_0">接之前提的问题，可是session id和session ticket不也是一段时间内固定的吗，psk只是加上了https报文信息，还是不太理解“更容易”被攻击的原因，万分感谢老师解答。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: session id的数据在内存里，所以不会有危险。<br><br>session ticket同样会受到重放攻击，本质上和psk是一样的，所以需要用一些手段来防止。<br><br>正文里没有说太清楚，可能造成了误解，sorry。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-26 09:59:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_e4a9c5</span>
  </div>
  <div class="_2_QraFYR_0">请问PSK 为什么容易受到重放攻击呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 因为PSK每次是固定的，这就容易被黑客截取，然后伪装成客户发送给服务器，如果没有鉴别手段就会被冒充。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-25 17:59:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/a5/f0/8648c464.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Joker</span>
  </div>
  <div class="_2_QraFYR_0">特来请教老师，第二个问题，有什么开源的项目里面有实现了这些吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 现在流行的Apache、Nginx都支持，因为这都是TLS协议标准规定的，可以参考它们的文档。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-14 20:33:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/50/67/beca0050.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Amberlo</span>
  </div>
  <div class="_2_QraFYR_0">老师好，关于 证书优化，获取CRL列表的时候会不会存在中间人攻击呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: crl有ca的签名，不会被伪造，所以是安全的。<br><br>不过如果有恶意ca，恶意签发假证书、假crl就是另外一回事了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-07 10:37:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/81/51/4999f121.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>qzmone</span>
  </div>
  <div class="_2_QraFYR_0">老师，https:&#47;&#47;www.chrono.com:441&#47;28-1 这个我抓包看到client和server的session-ID不一样，而且每次也都是server发了变更密码规范消息后才发加密的数据，跟您说的不一致？不知什么原因</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 第一次握手肯定是没有会话复用的，到第二次就会会话复用。<br><br>第一次后实验环境在浏览器会显示出“reused? false”，这就是没有复用。然后wireshark重新开始抓包，刷新一下页面，会显示为“reused? true”，这个时候再看抓包。<br><br>我又做了一次试验，是可以的，你再操作看看。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-16 11:08:58</div>
  </div>
</div>
</div>
</li>
</ul>