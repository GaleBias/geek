<audio title="23 _ HTTPS是什么？SSLTLS又是什么？" src="https://static001.geekbang.org/resource/audio/60/27/606366f47954111645ec28ffab6f4127.mp3" controls="controls"></audio> 
<p>从今天开始，我们开始进入全新的“安全篇”，聊聊与安全相关的HTTPS、SSL、TLS。</p><p>在<a href="https://time.geekbang.org/column/article/103746">第14讲</a>中，我曾经谈到过HTTP的一些缺点，其中的“无状态”在加入Cookie后得到了解决，而另两个缺点——“明文”和“不安全”仅凭HTTP自身是无力解决的，需要引入新的HTTPS协议。</p><h2>为什么要有HTTPS？</h2><p>简单的回答是“<strong>因为HTTP不安全</strong>”。</p><p>由于HTTP天生“明文”的特点，整个传输过程完全透明，任何人都能够在链路中截获、修改或者伪造请求/响应报文，数据不具有可信性。</p><p>比如，前几讲中说过的“代理服务”。它作为HTTP通信的中间人，在数据上下行的时候可以添加或删除部分头字段，也可以使用黑白名单过滤body里的关键字，甚至直接发送虚假的请求、响应，而浏览器和源服务器都没有办法判断报文的真伪。</p><p>这对于网络购物、网上银行、证券交易等需要高度信任的应用场景来说是非常致命的。如果没有基本的安全保护，使用互联网进行各种电子商务、电子政务就根本无从谈起。</p><p>对于安全性要求不那么高的新闻、视频、搜索等网站来说，由于互联网上的恶意用户、恶意代理越来越多，也很容易遭到“流量劫持”的攻击，在页面里强行嵌入广告，或者分流用户，导致各种利益损失。</p><!-- [[[read_end]]] --><p>对于你我这样的普通网民来说，HTTP不安全的隐患就更大了，上网的记录会被轻易截获，网站是否真实也无法验证，黑客可以伪装成银行网站，盗取真实姓名、密码、银行卡等敏感信息，威胁人身安全和财产安全。</p><p>总的来说，今天的互联网已经不再是早期的“田园牧歌”时代，而是进入了“黑暗森林”状态。上网的时候必须步步为营、处处小心，否则就会被不知道埋伏在哪里的黑客所“猎杀”。</p><h2>什么是安全？</h2><p>既然HTTP“不安全”，那什么样的通信过程才是安全的呢？</p><p>通常认为，<span class="orange">如果通信过程具备了四个特性，就可以认为是“安全”的</span>，这四个特性是：<span class="orange">机密性</span>、<span class="orange">完整性</span>，<span class="orange">身份认证</span>和<span class="orange">不可否认</span>。</p><p><strong>机密性</strong>（Secrecy/Confidentiality）是指对数据的“保密”，只能由可信的人访问，对其他人是不可见的“秘密”，简单来说就是不能让不相关的人看到不该看的东西。</p><p>比如小明和小红私下聊天，但“隔墙有耳”，被小强在旁边的房间里全偷听到了，这就是没有机密性。我们之前一直用的Wireshark ，实际上也是利用了HTTP的这个特点，捕获了传输过程中的所有数据。</p><p><strong>完整性</strong>（Integrity，也叫一致性）是指数据在传输过程中没有被篡改，不多也不少，“完完整整”地保持着原状。</p><p>机密性虽然可以让数据成为“秘密”，但不能防止黑客对数据的修改，黑客可以替换数据，调整数据的顺序，或者增加、删除部分数据，破坏通信过程。</p><p>比如，小明给小红写了张纸条：“明天公园见”。小强把“公园”划掉，模仿小明的笔迹把这句话改成了“明天广场见”。小红收到后无法验证完整性，信以为真，第二天的约会就告吹了。</p><p><strong>身份认证</strong>（Authentication）是指确认对方的真实身份，也就是“证明你真的是你”，保证消息只能发送给可信的人。</p><p>如果通信时另一方是假冒的网站，那么数据再保密也没有用，黑客完全可以使用冒充的身份“套”出各种信息，加密和没加密一样。</p><p>比如，小明给小红写了封情书：“我喜欢你”，但不留心发给了小强。小强将错就错，假冒小红回复了一个“白日做梦”，小明不知道这其实是小强的话，误以为是小红的，后果可想而知。</p><p>第四个特性是<strong>不可否认</strong>（Non-repudiation/Undeniable），也叫不可抵赖，意思是不能否认已经发生过的行为，不能“说话不算数”“耍赖皮”。</p><p>使用前三个特性，可以解决安全通信的大部分问题，但如果缺了不可否认，那通信的事务真实性就得不到保证，有可能出现“老赖”。</p><p>比如，小明借了小红一千元，没写借条，第二天矢口否认，小红也确实拿不出借钱的证据，只能认倒霉。另一种情况是小明借钱后还了小红，但没写收条，小红于是不承认小明还钱的事，说根本没还，要小明再掏出一千元。</p><p>所以，只有同时具备了机密性、完整性、身份认证、不可否认这四个特性，通信双方的利益才能有保障，才能算得上是真正的安全。</p><h2>什么是HTTPS？</h2><p>说到这里，终于轮到今天的主角HTTPS出场了，它为HTTP增加了刚才所说的四大安全特性。</p><p>HTTPS其实是一个“非常简单”的协议，RFC文档很小，只有短短的7页，里面规定了<strong>新的协议名“https”，默认端口号443</strong>，至于其他的什么请求-应答模式、报文结构、请求方法、URI、头字段、连接管理等等都完全沿用HTTP，没有任何新的东西。</p><p>也就是说，除了协议名“http”和端口号80这两点不同，HTTPS协议在语法、语义上和HTTP完全一样，优缺点也“照单全收”（当然要除去“明文”和“不安全”）。</p><p>不信你可以用URI“<a href="https://www.chrono.com">https://www.chrono.com</a>”访问之前08至21讲的所有示例，看看它的响应报文是否与HTTP一样。</p><pre><code>https://www.chrono.com
https://www.chrono.com/11-1
https://www.chrono.com/15-1?name=a.json
https://www.chrono.com/16-1
</code></pre><p><img src="https://static001.geekbang.org/resource/image/40/b0/40fbb989a9fd2217320ab287e80e1fb0.png?wh=1397*1001" alt=""></p><p>你肯定已经注意到了，在用HTTPS访问实验环境时Chrome会有不安全提示，必须点击“高级-继续前往”才能顺利显示页面。而且如果用Wireshark抓包，也会发现与HTTP不一样，不再是简单可见的明文，多了“Client Hello”“Server Hello”等新的数据包。</p><p>这就是HTTPS与HTTP最大的区别，它能够鉴别危险的网站，并且尽最大可能保证你的上网安全，防御黑客对信息的窃听、篡改或者“钓鱼”、伪造。</p><p>你可能要问了，既然没有新东西，HTTPS凭什么就能做到机密性、完整性这些安全特性呢？</p><p>秘密就在于HTTPS名字里的“S”，它把HTTP下层的传输协议由TCP/IP换成了SSL/TLS，由“<strong>HTTP over TCP/IP</strong>”变成了“<strong>HTTP over SSL/TLS</strong>”，让HTTP运行在了安全的SSL/TLS协议上（可参考第4讲和第5讲），收发报文不再使用Socket API，而是调用专门的安全接口。</p><p><img src="https://static001.geekbang.org/resource/image/50/a3/50d57e18813e18270747806d5d73f0a3.png?wh=2057*810" alt=""></p><p>所以说，HTTPS本身并没有什么“惊世骇俗”的本事，全是靠着后面的SSL/TLS“撑腰”。只要学会了SSL/TLS，HTTPS自然就“手到擒来”。</p><h2>SSL/TLS</h2><p>现在我们就来看看SSL/TLS，它到底是个什么来历。</p><p>SSL即安全套接层（Secure Sockets Layer），在OSI模型中处于第5层（会话层），由网景公司于1994年发明，有v2和v3两个版本，而v1因为有严重的缺陷从未公开过。</p><p>SSL发展到v3时已经证明了它自身是一个非常好的安全通信协议，于是互联网工程组IETF在1999年把它改名为TLS（传输层安全，Transport Layer Security），正式标准化，版本号从1.0重新算起，所以TLS1.0实际上就是SSLv3.1。</p><p>到今天TLS已经发展出了三个版本，分别是2006年的1.1、2008年的1.2和去年（2018）的1.3，每个新版本都紧跟密码学的发展和互联网的现状，持续强化安全和性能，已经成为了信息安全领域中的权威标准。</p><p>目前应用的最广泛的TLS是1.2，而之前的协议（TLS1.1/1.0、SSLv3/v2）都已经被认为是不安全的，各大浏览器即将在2020年左右停止支持，所以接下来的讲解都针对的是TLS1.2。</p><p>TLS由记录协议、握手协议、警告协议、变更密码规范协议、扩展协议等几个子协议组成，综合使用了对称加密、非对称加密、身份认证等许多密码学前沿技术。</p><p>浏览器和服务器在使用TLS建立连接时需要选择一组恰当的加密算法来实现安全通信，这些算法的组合被称为“密码套件”（cipher suite，也叫加密套件）。</p><p>你可以访问实验环境的URI“/23-1”，对TLS和密码套件有个感性的认识。</p><p><img src="https://static001.geekbang.org/resource/image/5e/24/5ead57e03f127ea8f244d715186adb24.png?wh=1300*1182" alt=""></p><p>你可以看到，实验环境使用的TLS是1.2，客户端和服务器都支持非常多的密码套件，而最后协商选定的是“ECDHE-RSA-AES256-GCM-SHA384”。</p><p>这么长的名字看着有点晕吧，不用怕，其实TLS的密码套件命名非常规范，格式很固定。基本的形式是“密钥交换算法+签名算法+对称加密算法+摘要算法”，比如刚才的密码套件的意思就是：</p><p>“握手时使用ECDHE算法进行密钥交换，用RSA签名和身份认证，握手后的通信使用AES对称算法，密钥长度256位，分组模式是GCM，摘要算法SHA384用于消息认证和产生随机数。”</p><h2>OpenSSL</h2><p>说到TLS，就不能不谈到OpenSSL，它是一个著名的开源密码学程序库和工具包，几乎支持所有公开的加密算法和协议，已经成为了事实上的标准，许多应用软件都会使用它作为底层库来实现TLS功能，包括常用的Web服务器Apache、Nginx等。</p><p>OpenSSL是从另一个开源库SSLeay发展出来的，曾经考虑命名为“OpenTLS”，但当时（1998年）TLS还未正式确立，而SSL早已广为人知，所以最终使用了“OpenSSL”的名字。</p><p>OpenSSL目前有三个主要的分支，1.0.2和1.1.0都将在今年（2019）年底不再维护，最新的长期支持版本是1.1.1，我们的实验环境使用的OpenSSL是“1.1.0j”。</p><p>由于OpenSSL是开源的，所以它还有一些代码分支，比如Google的BoringSSL、OpenBSD的LibreSSL，这些分支在OpenSSL的基础上删除了一些老旧代码，也增加了一些新特性，虽然背后有“大金主”，但离取代OpenSSL还差得很远。</p><h2>小结</h2><ol>
<li><span class="orange">因为HTTP是明文传输，所以不安全，容易被黑客窃听或篡改；</span></li>
<li><span class="orange">通信安全必须同时具备机密性、完整性、身份认证和不可否认这四个特性；</span></li>
<li><span class="orange">HTTPS的语法、语义仍然是HTTP，但把下层的协议由TCP/IP换成了SSL/TLS；</span></li>
<li><span class="orange">SSL/TLS是信息安全领域中的权威标准，采用多种先进的加密技术保证通信安全；</span></li>
<li><span class="orange">OpenSSL是著名的开源密码学工具包，是SSL/TLS的具体实现。</span></li>
</ol><h2>课下作业</h2><ol>
<li>你能说出HTTPS与HTTP有哪些区别吗？</li>
<li>你知道有哪些方法能够实现机密性、完整性等安全特性呢？</li>
</ol><p>欢迎你把自己的学习体会写在留言区，与我和其他同学一起讨论。如果你觉得有所收获，也欢迎把文章分享给你的朋友。</p><p><img src="https://static001.geekbang.org/resource/image/05/4a/052e28eaa90a37f21ae4052135750a4a.png?wh=1769*3198" alt="unpreview"></p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/a5/98/a65ff31a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>djfhchdh</span>
  </div>
  <div class="_2_QraFYR_0">机密性由对称加密AES保证，完整性由SHA384摘要算法保证，身份认证和不可否认由RSA非对称加密保证</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: √</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-19 10:37:35</div>
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
  <div class="_2_QraFYR_0">老师好!有个问题，之前调用第三方的支付走https协议都需要本地配置一个证书。为啥最近有个项目也是用的https协议(url里会放token)。直接和http一样调用就好了，不需要本地配置证书了呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 本地证书是用来做双向认证的，服务器用客户端的证书来验证客户端的证书。<br><br>通常我们上网是单向认证，只验证服务器的身份，客户端（也就是用户）的身份不用证书验证。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-25 10:40:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/f2/70/8159901c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>David Mao</span>
  </div>
  <div class="_2_QraFYR_0">老师，请教一下，我们现在正在申请SSL证书，SSL证书有专门的机构颁发，文中老师提到HTTPS能够鉴别危险网站，防止黑客篡改，这些具体是怎么做到的呢？由专门机构颁发的原因是什么？谢谢老师。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果网站是http而不是https，那么浏览器就会认为网站不安全，有风险。<br><br>如果证书内容不完善，或者被列入了黑名单，那么浏览器也可以提示用户有危险。<br><br>这些的关键都是证书，用证书里的信息，来验证网站的有效性、真实性。<br><br>因为证书用来证明网站的身份，就像身份证、学位证一样，如果随便颁发，那么它的可靠性就得不到保证，所以必须要由指定信任的专门机构来颁发，由ca来“背书”，保证证书和它关联的网站是安全可靠的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-20 08:14:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/08/17/e63e50f3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>彩色的沙漠</span>
  </div>
  <div class="_2_QraFYR_0">1、HTTPS相对于HTTP具有机密性，完整性，身份认证和不可否认的特性,HTTPS是HTTP over SSL&#47;TLS,HTTP&gt; HTTP over TCP&#47;IP<br>2、实现机密性可以采用加密手段，接口签名实现完整性，数字签名用于身份认证</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: √</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-19 08:44:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/54/b2/5ea0b709.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Danpier</span>
  </div>
  <div class="_2_QraFYR_0">有个疑问，维基百科 OSI 模型图表把 SSL\TLS 归到第6层（表示层），文中说 SSL 属于第5层（会话层），这里是不是写错了？ 附：<br>https:&#47;&#47;en.wikipedia.org&#47;wiki&#47;OSI_model#Layer_6:_Presentation_Layer</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个分层没有统一的定论，ssl&#47;tls在tcp&#47;ip里属于应用层，但不能准确对应到osi的某一层，因为它即有会话功能，又有加解密表示。<br><br>我们只要会用、理解就行，不要过于拘泥于学术。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-12 16:14:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/80/f4/564209ea.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>纳兰容若</span>
  </div>
  <div class="_2_QraFYR_0">老师您好<br>一直以来不太明白openssl的各版本，我看官网上还有2.0和3.0的，还有后面还有t、h、j字母跟在后面，这些大概有什么区别，正常使用不知道选择什么版本好，老师有什么建议么<br>感谢老师回复</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: OpenSSL的官网上对各个版本写的很清楚，最早它是由ssleay发展来的，所以就沿用了0.98的版本号，后来又有好几个系列，比如1.01、1.10等等，小版本号用字母表示。<br><br>目前OpenSSL准备跳过2.0，直接出3.0，我们用最新的1.11就好了，之前的版本都将不再维护。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-04 09:53:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/97/1a/389eab84.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>而立斋</span>
  </div>
  <div class="_2_QraFYR_0">1、https与http协议相比，最重要的是增加安全性，这种安全性的实现主要是依赖于两个协议底层依赖的协议是不同的，https在传输的应用层与传输层协议之间增加了ssl&#47;tls,这就使得http在固有协议之上增加一层专用用于处理数据安全的工具。<br><br>2、机密性：数据使用非对称加密传输<br>完整性：数据用公钥加密，私钥解密，数据生成摘要算法，同步传输</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.正确。<br><br>2.机密性主要用对称加密实现，非对称加密虽然也可以，但是效率太低，不实用。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-08 09:49:16</div>
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
  <div class="_2_QraFYR_0">P-256 是 NIST（美国国家标准技术研究所）和 NSA（美国国家安全局）推荐使用的曲线。而密码学界不信任这两个机构，所以 P-256 是有可能被秘密破解但出于政治考虑而未公开？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，可能有这个隐患，就跟des一样。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-22 08:38:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/6f/10/bfbf81dc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>海绵薇薇</span>
  </div>
  <div class="_2_QraFYR_0">老师好，我想问下，在HTTPS协议上传输的报文，是怎么被缓存的，因为传输的内容应该都是加密的，那么如何做到If-Match或者If-Modified-Since判断的呢？CDN可以解析加密过后的内容吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.https传输过程加密，但在两边是解密的，所以可以看到头字段，然后再走正常的判断处理流程。<br><br>2.如果cdn有证书和私钥，就可以解密数据，否则就只能做透明代理，在tcp层面转发。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-29 22:41:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/1a/ac/cec17283.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>zhangdroid</span>
  </div>
  <div class="_2_QraFYR_0">HTTPS：即HTTP over SSL&#47;TLS，用来解决HTTP明文传输导致的不安全问题。流程大致为：<br>使用对称加密算法加解密报文，保证机密性；使用摘要算法保证数据完整性；使用证书CA来进行身份认证;而不可否认则由非对称加密算法来实现。由于非对称加密算法耗时比对称加密算法长，所以用非对称加密算法来加解密给报文加密的对称算法的秘钥：即使用公钥对对称加密算法秘钥进行加密，私钥用来相应地解密。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-11 16:50:15</div>
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
  <div class="_2_QraFYR_0">老师，以下问题，麻烦解答：<br>1. 这就是 HTTPS 与 HTTP 最大的区别，它能够鉴别危险的网站?这个仅仅从浏览器弹出不安全的提示来说的嘛？或者说怎么个鉴别法？<br>2. 网站是否真实也无法验证。加了https的网站也有可能是钓鱼网站吧？也没法验证啊？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.如果网站的证书不可信（过期、失效、被废除、伪造），那么就可以说明网站是不安全的，而http不能对网站有任何的认证措施。<br><br>2.证书有dv、ov、ev三种，能从ca的层面证明网站的所有者。<br><br>3.当然，如果网站故意作恶，https也无法制止，它只能证明网站确实是如证书所声明的，不是假冒的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-11 09:57:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/d7/a7/2c979c01.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>蒋润</span>
  </div>
  <div class="_2_QraFYR_0">老师你好  https能有效防止抓包然后篡改报文数据,防止xxs攻击吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: https传输的内容是加密的，所以抓包后看不到明文，是无法窜改数据的。<br><br>但xss属于内容攻击，报文本身是合法的，所以它不能防止。<br><br>https只能保证数据传输安全，但在链接的两端不能提供保护。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-24 10:36:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/57/0e/5f0bb588.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>懒人一枚</span>
  </div>
  <div class="_2_QraFYR_0">觉得大佬还可以再深入一些，比如浏览器是如何验证证书的，证书对浏览器对作用是什么，这个证书和我们平时接口调用使用的证书有什么区别呢，为什么有的网站有证书，浏览器却不认</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个问题跟浏览器实现有关，而与http协议就无关了，我个人对浏览器的工作原理不是很了解，只能说抱歉了。<br><br>其实对于http协议来说，浏览器就是一个普通的客户端，与wget、curl没有区别。而不认证书，那就是信任发生了问题，要么是过期失效，要么是上级证书有问题。<br><br>其他疑问可以看后面几讲，有不明白的随时问。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-18 21:05:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/79/e3/0ec0b681.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>mini💝</span>
  </div>
  <div class="_2_QraFYR_0">请问老师 端口的作用是什么呢？为什么http和https的默认端口是不一样的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个属于tcp&#47;ip层次的知识了，我简单说一下吧。<br><br>互联网上的机器都用ip地址来标记，但只有ip还不够，一台机器上会有很多不同的服务，为了区分，就要再加上端口，这样才能完整地标记一个网络服务。<br><br>比如有台主机的地址是10.1.1.1，它在端口22上开了ssh服务，21上开了ftp服务，80是http，443是https，这样客户端就可以用地址加上不同的端口去访问主机上的服务了。<br><br>http和https的默认端口都是国际标准化组织分配的，比如etcd就用了2379。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-03 09:51:09</div>
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
  <div class="_2_QraFYR_0">1：你能说出 HTTPS 与 HTTP 有哪些区别吗？<br>       正如文中所言HTTPS比HTTP多了一个S，这个S代表安全，是基于SSL&#47;TSL实现的，SSL&#47;TSL是专门用于安全传输的，具体咋实现的比较复杂还没弄明白，主要就是各种加密算法的应用，后面继续看。<br><br>2：你知道有哪些方法能够实现机密性、完整性等安全特性呢？<br>这个问题不知如何回答，不会，看答案如下。<br>机密性由对称加密AES保证<br>安全性由SHA384摘要算法保证<br>身份认证由RSA非对称加密算法保证<br>不可否认由RSA非对称加密算法保证<br>符合以上四点的才算是安全的通信方式，实现安全性看样子很不容易啊！<br>这些加密算法，他的发明者是否比较容易破解呢？还是说加密之后即使是发明者也无能为力，那如果解密的东西丢啦咋弄？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 加密算法设计出来就是要任何人都无法破解的，否则就是有后门。<br><br>这个是密码学的基本原则，必须有密钥才能解密，至于如何保管就是另外的事情。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-30 21:05:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/30/8a/b5ca7286.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>业余草</span>
  </div>
  <div class="_2_QraFYR_0">老师，我的个人网站：https:&#47;&#47;www.xttblog.com  在mac上的谷歌浏览器最新版中控制台总是会报一个错误，而我已经是https了，这个问题，空扰了我很久</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 什么错误，说出来看看。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-05 08:51:53</div>
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
  <div class="_2_QraFYR_0">“明文”和“不安全”仅凭 HTTP 自身是无力解决的，需要引入新的 HTTPS 协议  老师，请问下明文和不安全不是同一个东西么？为什么要分开说呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 明文只是消息可见，而不安全的范围要更广一些，比如身份认证、防抵赖。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-23 17:57:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/4faqHgQSawd4VzAtSv0IWDddm9NucYWibRpxejWPH5RUO310qv8pAFmc0rh0Qu6QiahlTutGZpia8VaqP2w6icybiag/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>爱编程的运维</span>
  </div>
  <div class="_2_QraFYR_0">像一般的web网站存储用户的密码，密码存在数据库表中都是加密后的，这个加密跟https中的加密有啥区别？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复:   我们通常所说的加密和密码学里的加密不是一个概念。<br><br>密码的正式名称应该叫password，存在数据库里实际上是做了摘要hash，比如md5、sha1，不是密码学里的用密钥算法加密。<br><br>而https里的加密是真正的密码学，里面有对称算法、非对称算法、摘要算法、证书等等，非常复杂。<br><br>可以再看后面的课程来加深理解。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-07 19:33:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/2d/ca/02b0e397.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>fomy</span>
  </div>
  <div class="_2_QraFYR_0">如果每个人都可以生成一个证书，钓鱼网站也可以申请呀，不就没有什么安全性可言了吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，但ca颁发证书有验证机制，钓鱼网站是无法伪装成官方网站的，因为它不拥有官方域名。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-23 18:43:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/e9/0b/1171ac71.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>WL</span>
  </div>
  <div class="_2_QraFYR_0">请问一下老师我这边用WireShark抓包，发现两个TLS请求和响应之间和两个HTTP请求和响应之间有很多个TCP的包，请问一下这些TCP的包是一个HTTP的响应没有发完后续一致在通过TCP包发HTTP响应的responseBody吗？ </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 应该在wireshark里看一下这些tcp包的端口、发送方向，应该不是https相关的包，可以过滤一下试试。<br><br>https必须在ssl&#47;tls握手之后才能发送http报文。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-19 19:16:38</div>
  </div>
</div>
</div>
</li>
</ul>