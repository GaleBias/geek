<audio title="第15讲 _ HTTPS协议：点外卖的过程原来这么复杂" src="https://static001.geekbang.org/resource/audio/f3/8d/f32cb9ab428dd51e209d6c1ce260ae8d.mp3" controls="controls"></audio> 
<p>用HTTP协议，看个新闻还没有问题，但是换到更加严肃的场景中，就存在很多的安全风险。例如，你要下单做一次支付，如果还是使用普通的HTTP协议，那你很可能会被黑客盯上。</p><p>你发送一个请求，说我要点个外卖，但是这个网络包被截获了，于是在服务器回复你之前，黑客先假装自己就是外卖网站，然后给你回复一个假的消息说：“好啊好啊，来来来，银行卡号、密码拿来。”如果这时候你真把银行卡密码发给它，那你就真的上套了。</p><p>那怎么解决这个问题呢？当然一般的思路就是<strong>加密</strong>。加密分为两种方式一种是<strong>对称加密</strong>，一种是<strong>非对称加密</strong>。</p><p>在对称加密算法中，加密和解密使用的密钥是相同的。也就是说，加密和解密使用的是同一个密钥。因此，对称加密算法要保证安全性的话，密钥要做好保密。只能让使用的人知道，不能对外公开。</p><p>在非对称加密算法中，加密使用的密钥和解密使用的密钥是不相同的。一把是作为公开的公钥，另一把是作为谁都不能给的私钥。公钥加密的信息，只有私钥才能解密。私钥加密的信息，只有公钥才能解密。</p><p>因为对称加密算法相比非对称加密算法来说，效率要高得多，性能也好，所以交互的场景下多用对称加密。</p><h2>对称加密</h2><p>假设你和外卖网站约定了一个密钥，你发送请求的时候用这个密钥进行加密，外卖网站用同样的密钥进行解密。这样就算中间的黑客截获了你的请求，但是它没有密钥，还是破解不了。</p><!-- [[[read_end]]] --><p>这看起来很完美，但是中间有个问题，你们两个怎么来约定这个密钥呢？如果这个密钥在互联网上传输，也是很有可能让黑客截获的。黑客一旦截获这个秘钥，它可以佯作不知，静静地等着你们两个交互。这时候你们之间互通的任何消息，它都能截获并且查看，就等你把银行卡账号和密码发出来。</p><p>我们在谍战剧里面经常看到这样的场景，就是特工破译的密码会有个密码本，截获无线电台，通过密码本就能将原文破解出来。怎么把密码本给对方呢？只能通过<strong>线下传输</strong>。</p><p>比如，你和外卖网站偷偷约定时间地点，它给你一个纸条，上面写着你们两个的密钥，然后说以后就用这个密钥在互联网上定外卖了。当然你们接头的时候，也会先约定一个口号，什么“天王盖地虎”之类的，口号对上了，才能把纸条给它。但是，“天王盖地虎”同样也是对称加密密钥，同样存在如何把“天王盖地虎”约定成口号的问题。而且在谍战剧中一对一接头可能还可以，在互联网应用中，客户太多，这样是不行的。</p><h2>非对称加密</h2><p>所以，只要是对称加密，就会永远在这个死循环里出不来，这个时候，就需要非对称加密介入进来。</p><p>非对称加密的私钥放在外卖网站这里，不会在互联网上传输，这样就能保证这个密钥的私密性。但是，对应私钥的公钥，是可以在互联网上随意传播的，只要外卖网站把这个公钥给你，你们就可以愉快地互通了。</p><p>比如说你用公钥加密，说“我要定外卖”，黑客在中间就算截获了这个报文，因为它没有私钥也是解不开的，所以这个报文可以顺利到达外卖网站，外卖网站用私钥把这个报文解出来，然后回复，“那给我银行卡和支付密码吧”。</p><p>先别太乐观，这里还是有问题的。回复的这句话，是外卖网站拿私钥加密的，互联网上人人都可以把它打开，当然包括黑客。那外卖网站可以拿公钥加密吗？当然不能，因为它自己的私钥只有它自己知道，谁也解不开。</p><p>另外，这个过程还有一个问题，黑客也可以模拟发送“我要定外卖”这个过程的，因为它也有外卖网站的公钥。</p><p>为了解决这个问题，看来一对公钥私钥是不够的，客户端也需要有自己的公钥和私钥，并且客户端要把自己的公钥，给外卖网站。</p><p>这样，客户端给外卖网站发送的时候，用外卖网站的公钥加密。而外卖网站给客户端发送消息的时候，使用客户端的公钥。这样就算有黑客企图模拟客户端获取一些信息，或者半路截获回复信息，但是由于它没有私钥，这些信息它还是打不开。</p><h2>数字证书</h2><p>不对称加密也会有同样的问题，如何将不对称加密的公钥给对方呢？一种是放在一个公网的地址上，让对方下载；另一种就是在建立连接的时候，传给对方。</p><p>这两种方法有相同的问题，那就是，作为一个普通网民，你怎么鉴别别人给你的公钥是对的。会不会有人冒充外卖网站，发给你一个它的公钥。接下来，你和它所有的互通，看起来都是没有任何问题的。毕竟每个人都可以创建自己的公钥和私钥。</p><p>例如，我自己搭建了一个网站cliu8site，可以通过这个命令先创建私钥。</p><pre><code>openssl genrsa -out cliu8siteprivate.key 1024
</code></pre><p>然后，再根据这个私钥，创建对应的公钥。</p><pre><code>openssl rsa -in cliu8siteprivate.key -pubout -outcliu8sitepublic.pem
</code></pre><p>这个时候就需要权威部门的介入了，就像每个人都可以打印自己的简历，说自己是谁，但是有公安局盖章的，就只有户口本，这个才能证明你是你。这个由权威部门颁发的称为<strong>证书</strong>（<strong>Certificate</strong>）。</p><p>证书里面有什么呢？当然应该有<strong>公钥</strong>，这是最重要的；还有证书的<strong>所有者</strong>，就像户口本上有你的姓名和身份证号，说明这个户口本是你的；另外还有证书的<strong>发布机构</strong>和证书的<strong>有效期</strong>，这个有点像身份证上的机构是哪个区公安局，有效期到多少年。</p><p>这个证书是怎么生成的呢？会不会有人假冒权威机构颁发证书呢？就像有假身份证、假户口本一样。生成证书需要发起一个证书请求，然后将这个请求发给一个权威机构去认证，这个权威机构我们称为<strong>CA</strong>（ <strong>Certificate Authority</strong>）。</p><p>证书请求可以通过这个命令生成。</p><pre><code>openssl req -key cliu8siteprivate.key -new -out cliu8sitecertificate.req
</code></pre><p>将这个请求发给权威机构，权威机构会给这个证书卡一个章，我们称为<strong>签名算法。</strong>问题又来了，那怎么签名才能保证是真的权威机构签名的呢？当然只有用只掌握在权威机构手里的东西签名了才行，这就是CA的私钥。</p><p>签名算法大概是这样工作的：一般是对信息做一个Hash计算，得到一个Hash值，这个过程是不可逆的，也就是说无法通过Hash值得出原来的信息内容。在把信息发送出去时，把这个Hash值加密后，作为一个签名和信息一起发出去。</p><p>权威机构给证书签名的命令是这样的。</p><pre><code>openssl x509 -req -in cliu8sitecertificate.req -CA cacertificate.pem -CAkey caprivate.key -out cliu8sitecertificate.pem
</code></pre><p>这个命令会返回Signature ok，而cliu8sitecertificate.pem就是签过名的证书。CA用自己的私钥给外卖网站的公钥签名，就相当于给外卖网站背书，形成了外卖网站的证书。</p><p>我们来查看这个证书的内容。</p><pre><code>openssl x509 -in cliu8sitecertificate.pem -noout -text 
</code></pre><p>这里面有个Issuer，也即证书是谁颁发的；Subject，就是证书颁发给谁；Validity是证书期限；Public-key是公钥内容；Signature Algorithm是签名算法。</p><p>这下好了，你不会从外卖网站上得到一个公钥，而是会得到一个证书，这个证书有个发布机构CA，你只要得到这个发布机构CA的公钥，去解密外卖网站证书的签名，如果解密成功了，Hash也对的上，就说明这个外卖网站的公钥没有啥问题。</p><p>你有没有发现，又有新问题了。要想验证证书，需要CA的公钥，问题是，你怎么确定CA的公钥就是对的呢？</p><p>所以，CA的公钥也需要更牛的CA给它签名，然后形成CA的证书。要想知道某个CA的证书是否可靠，要看CA的上级证书的公钥，能不能解开这个CA的签名。就像你不相信区公安局，可以打电话问市公安局，让市公安局确认区公安局的合法性。这样层层上去，直到全球皆知的几个著名大CA，称为<strong>root CA</strong>，做最后的背书。通过这种<strong>层层授信背书</strong>的方式，从而保证了非对称加密模式的正常运转。</p><p>除此之外，还有一种证书，称为<strong>Self-Signed Certificate</strong>，就是自己给自己签名。这个给人一种“我就是我，你爱信不信”的感觉。这里我就不多说了。</p><h2>HTTPS的工作模式</h2><p>我们可以知道，非对称加密在性能上不如对称加密，那是否能将两者结合起来呢？例如，公钥私钥主要用于传输对称加密的秘钥，而真正的双方大数据量的通信都是通过对称加密进行的。</p><p>当然是可以的。这就是HTTPS协议的总体思路。</p><p><img src="https://static001.geekbang.org/resource/image/df/b4/df1685dd308cef1db97e91493f911ab4.jpg?wh=2285*4076" alt=""></p><p>当你登录一个外卖网站的时候，由于是HTTPS，客户端会发送Client Hello消息到服务器，以明文传输TLS版本信息、加密套件候选列表、压缩算法候选列表等信息。另外，还会有一个随机数，在协商对称密钥的时候使用。</p><p>这就类似在说：“您好，我想定外卖，但你要保密我吃的是什么。这是我的加密套路，再给你个随机数，你留着。”</p><p>然后，外卖网站返回Server Hello消息, 告诉客户端，服务器选择使用的协议版本、加密套件、压缩算法等，还有一个随机数，用于后续的密钥协商。</p><p>这就类似在说：“您好，保密没问题，你的加密套路还挺多，咱们就按套路2来吧，我这里也有个随机数，你也留着。”</p><p>然后，外卖网站会给你一个服务器端的证书，然后说：“Server Hello Done，我这里就这些信息了。”</p><p>你当然不相信这个证书，于是你从自己信任的CA仓库中，拿CA的证书里面的公钥去解密外卖网站的证书。如果能够成功，则说明外卖网站是可信的。这个过程中，你可能会不断往上追溯CA、CA的CA、CA的CA的CA，反正直到一个授信的CA，就可以了。</p><p>证书验证完毕之后，觉得这个外卖网站可信，于是客户端计算产生随机数字Pre-master，发送Client Key Exchange，用证书中的公钥加密，再发送给服务器，服务器可以通过私钥解密出来。</p><p>到目前为止，无论是客户端还是服务器，都有了三个随机数，分别是：自己的、对端的，以及刚生成的Pre-Master随机数。通过这三个随机数，可以在客户端和服务器产生相同的对称密钥。</p><p>有了对称密钥，客户端就可以说：“Change Cipher Spec，咱们以后都采用协商的通信密钥和加密算法进行加密通信了。”</p><p>然后发送一个Encrypted Handshake Message，将已经商定好的参数等，采用协商密钥进行加密，发送给服务器用于数据与握手验证。</p><p>同样，服务器也可以发送Change Cipher Spec，说：“没问题，咱们以后都采用协商的通信密钥和加密算法进行加密通信了”，并且也发送Encrypted Handshake Message的消息试试。当双方握手结束之后，就可以通过对称密钥进行加密传输了。</p><p>这个过程除了加密解密之外，其他的过程和HTTP是一样的，过程也非常复杂。</p><p>上面的过程只包含了HTTPS的单向认证，也即客户端验证服务端的证书，是大部分的场景，也可以在更加严格安全要求的情况下，启用双向认证，双方互相验证证书。</p><h2>重放与篡改</h2><p>其实，这里还有一些没有解决的问题，例如重放和篡改的问题。</p><p>没错，有了加密和解密，黑客截获了包也打不开了，但是它可以发送N次。这个往往通过Timestamp和Nonce随机数联合起来，然后做一个不可逆的签名来保证。</p><p>Nonce随机数保证唯一，或者Timestamp和Nonce合起来保证唯一，同样的，请求只接受一次，于是服务器多次收到相同的Timestamp和Nonce，则视为无效即可。</p><p>如果有人想篡改Timestamp和Nonce，还有签名保证不可篡改性，如果改了用签名算法解出来，就对不上了，可以丢弃了。</p><h2>小结</h2><p>好了，这一节就到这里了，我们来总结一下。</p><ul>
<li>
<p><span class="orange">加密分对称加密和非对称加密。对称加密效率高，但是解决不了密钥传输问题；非对称加密可以解决这个问题，但是效率不高。</span></p>
</li>
<li>
<p><span class="orange"> 非对称加密需要通过证书和权威机构来验证公钥的合法性。</span></p>
</li>
<li>
<p><span class="orange">HTTPS是综合了对称加密和非对称加密算法的HTTP协议。既保证传输安全，也保证传输效率。</span></p>
</li>
</ul><p>最后，给你留两个思考题：</p><ol>
<li>
<p>HTTPS协议比较复杂，沟通过程太繁复，这样会导致效率问题，那你知道有哪些手段可以解决这些问题吗？</p>
</li>
<li>
<p>HTTP和HTTPS协议的正文部分传输个JSON什么的还好，如果播放视频，就有问题了，那这个时候，应该使用什么协议呢？</p>
</li>
</ol><p>欢迎你留言和我讨论。趣谈网络协议，我们下期见！</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/1f/66/59e0647a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>万历十五年</span>
  </div>
  <div class="_2_QraFYR_0">各大CA机构的公钥是默认安装在操作系统里的。所以不要安装来路不明的操作系统，否则相当于裸奔</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-22 20:52:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/13/00/a4a2065f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Michael</span>
  </div>
  <div class="_2_QraFYR_0">看了好几遍，感觉算是明白 https 非对称加密通信的过程了，总结一下，大家看看是不是这样的：<br><br>首先，服务端需要向证书颁发机构申请一个自己的证书，这个证书里面会包含此该站点的基本信息，个人啊，公司啊，组织什么呢，我记得CA证书好像分三类的，然后还有该证书的 签名 以及 hash 值用于在通信中客户端鉴别此证书是否合法。<br><br>https 通信分为四个步骤：<br><br>1. c-&gt;s,客户端发起加密通信请求，这个请求通常叫做 ClientHello请求，告知自己支持的协议版本号，加密算法，压缩算法，以及一个用于生成后续通信密钥的随机数；<br>2. s-&gt;c,服务端响应，也叫作 ServerHello，确认加密通信协议，加密算法，以及一个用于生成后续通信密钥的随机数，还有网站证书；<br>3. c-&gt;s,客户端在收到上一步服务端的响应之后，首先会检查证书的颁发者是否可信任，是否过期，域名是否一致，并且从操作系统的证书链中找出该证书的上一级证书，并拿出服务端证书的公钥，然后验证签名和hash，如果验证失败，就会显示警告，我们经常在Chrome里面看到，“此网站有风险，是否继续什么的”。如果验证通过，客户端会向服务端发送一个称作 “pre-master-key” 的随机数，该随机数使用证书的公钥加密，以及编码改变通知（以后咋们就用协商的密钥堆成加密通信了），客户端完成握手。<br>4. 服务端在收到上一步客户端请求之后，也会确认我以后发给你的信息可就加密了哦，并且完成握手。<br><br>此时，客户端有第一步自己生成的随机数，第二步收到服务端的随机数，第三步的 pre-master-key，服务端也是如此，他们就可以用这三个随机数使用约定的算法生成同一个密钥来加密以后的通信数据了。<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-11 13:50:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/pc41FOKAiabVaaKiawibEm7zglvnsYBnYeRiaSAElf9ciczovXmXmI0hOeR6U9RULFtMoqX5kobNttvwXCLsUM9Hbcg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>monkay</span>
  </div>
  <div class="_2_QraFYR_0">会不会存在浏览器第一次向外卖网站请求证书时就被黑客拦截了，然后黑客用自己服务器去权威机构请求正规的证书，然后发给用户，之后的交互，用户以为是跟外卖网站交互，其实背后是跟黑客的网站交互</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-21 07:03:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/ef/a2/6ea5bb9e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>LEON</span>
  </div>
  <div class="_2_QraFYR_0">这张SSL握手的图，真的棒！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-20 08:30:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c9/20/e4f1b17c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>zj</span>
  </div>
  <div class="_2_QraFYR_0">Nonce是由服务器生成的一个随机数，在客户端第一次请求页面时将其发回客户端；客户端拿到这个Nonce，将其与用户密码串联在一起并进行非可逆加密（MD5、SHA1等等），然后将这个加密后的字符串和用户名、Nonce、加密算法名称一起发回服务器；服务器使用接收到的用户名到数据库搜索密码，然后跟客户端使用同样的算法对其进行加密，接着将其与客户端提交上来的加密字符串进行比较，如果两个字符串一致就表示用户身份有效。这样就解决了用户密码明文被窃取的问题，攻击者就算知道了算法名和nonce也无法解密出密码。<br><br> <br><br>每个nonce只能供一个用户使用一次，这样就可以防止攻击者使用重放攻击，因为该Http报文已经无效。可选的实现方式是把每一次请求的Nonce保存到数据库，客户端再一次提交请求时将请求头中得Nonce与数据库中得数据作比较，如果已存在该Nonce，则证明该请求有可能是恶意的。然而这种解决方案也有个问题，很有可能在两次正常的资源请求中，产生的随机数是一样的，这样就造成正常的请求也被当成了攻击，随着数据库中保存的随机数不断增多，这个问题就会变得很明显。所以，还需要加上另外一个参数Timestamp（时间戳）。<br><br> <br><br>Timestamp是根据服务器当前时间生成的一个字符串，与nonce放在一起，可以表示服务器在某个时间点生成的随机数。这样就算生成的随机数相同，但因为它们生成的时间点不一样，所以也算有效的随机数</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-23 22:55:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/60/36/1848c2b7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>dovefi</span>
  </div>
  <div class="_2_QraFYR_0">https 核心思想，首先数字证书保证了服务端公钥的可靠性，客户端确认完可靠性之后，就用这个公钥去加密pre-master，服务端收到加密后的数据，用自己的私钥去解开，得到pre-master，然后两边都有了①客户端的随机数 ②服务端的随机数 ③pre-master  然后就各自通过计算得到一样的值，就当做公钥。后面就是对称加密的数据传输。这个过程中，对称加密的公钥从来没有在网络上传输过，所以不存在暴露的微信，ca证书中的解密是通过不对称的加密方式实现的，而且保证了公钥的可靠性，所以这里不怕别人拿到公钥，因为私钥他没有，也不怕别人伪造私钥和公钥，因为ca保证了公钥是可靠的</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-07 18:50:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/THkFNC52F0kYs2XI1fwxOvCck0Pibwnia4z6fzCPMRg2qYQLlt57qW4caJZ6uj9lWROc7t1OHFmIdKmiaEIP2GXpg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>isaac</span>
  </div>
  <div class="_2_QraFYR_0">单向认证中的三个随机数可能是为了保证完全随机吧，毕竟所有的机器都是伪随机的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞，对的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-28 18:32:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/19/ee/e395a35e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小先生</span>
  </div>
  <div class="_2_QraFYR_0">新学到的一点，CA 证书的作用，是保证服务器的公钥的来历。其做法是对公钥进行哈希摘要算法，然后用 CA 私钥加密，伴随公钥一起发送出去。<br>客户端收到后，用 CA 公钥解密，然后对公钥做哈希，比对哈希值是否一致来做到的。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-29 23:47:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c6/e4/ec572f55.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>沙亮亮</span>
  </div>
  <div class="_2_QraFYR_0">https还是可以被抓包，可以讲下抓https包的原理</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-20 08:26:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/8c/15/f85c0121.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>飞龙在天</span>
  </div>
  <div class="_2_QraFYR_0">老师你好：<br>https协议中，客户端和服务端信息传输使用了对称加密，疑惑的是服务器端是怎样识别多个客户端的公钥呢？项目中并没有对此做特殊的处理，希望老师解惑o(^o^)o。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 会建立session</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-20 10:08:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKSVGSo9leSm0vhtHgzIOL7uaJhOcaImuIzLIrVhXUNPmhd9HGIxs0nWIQm5RTCEjwJ6IuG3moOdQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_f1933b</span>
  </div>
  <div class="_2_QraFYR_0">对老师的证书生成的签名算法和用户校验再细化一下，有问题请大家指教哈<br>证书生成的签名算法：<br>1、服务器公钥 经过数字摘要算法 生成数字指纹<br>2、把生成的数字指纹 在用认证机构的私钥加密 生成数字签名<br>用户校验证书：<br>1、客户端取出提前内置在系统内部的认证机构的公钥<br>2、用认证机构的公钥去解密公钥证书里的数字签名 从而得到数字指纹<br>3、客户端对公钥证书的服务器公钥进行 数字摘要算法 从而生成数字指纹<br>4、对比客户端自己生成的数字指纹(第3步)和解密得到的数字指纹(第2步)是否一致 如果一致则公钥证书验证通过 就可以进行接下来的握手步骤了<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-08 12:24:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/9e/ef/a74049d9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>森码</span>
  </div>
  <div class="_2_QraFYR_0">客户端和服务端为什么要分别传输一个随机数呢？通过公钥加密Pre-master不是就可以用吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果双向认证，其实是可以的，但是不都是双向认证</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-07-11 23:42:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>shupian418</span>
  </div>
  <div class="_2_QraFYR_0">https是要解决http中的问题：<br>1 明文传输，很容易被窃听<br>2 没有验证数据的完整性，容易被篡改<br>3 没有验证对端的身份，容易被伪装</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-20 17:52:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/a9/cb/a431bde5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>木头发芽</span>
  </div>
  <div class="_2_QraFYR_0">@monkay:证书里还带了域名，如果浏览器联的是外卖网站的域名，被黑客拦截发回了黑客的ca证书，校验证书的时候会发现域名不对从而证书认证失败</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-12 21:43:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJTL1kXMgeyk9hcCTDWbrwuAW2OiaJmHLTN3ZppVzKJXIXicia8D9a38Nb8vjfCEepFJyJJUqAknib2UQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>酷哥居士</span>
  </div>
  <div class="_2_QraFYR_0">想请教老师一下，上面的HTTPS请求，客户端和服务端之间发送证书，验证证书，传输秘钥等等这些操作，是在HTTPS的TCP连接建立成功后做的吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-23 13:50:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/87/62/f99b5b05.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>曙光</span>
  </div>
  <div class="_2_QraFYR_0">这张就是双向认证的流程图吧，因为客户端发证书给服务器，单向认证客户端证书可发可不发；但双向认证，服务器一定需要客户端的证书。流程图画了客户端的证书，但文章描述说是单向认证，感觉有些歧义</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-13 10:13:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/cf/22/2091d425.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>李简单</span>
  </div>
  <div class="_2_QraFYR_0">证书解密后，hash怎么算对上了呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-08 13:54:25</div>
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
  <div class="_2_QraFYR_0">老师，可以补充一下https双向认证的流程图吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 好的，后期补充一下</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-07-17 09:07:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/88/9c/cbc463e6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>仰望星空</span>
  </div>
  <div class="_2_QraFYR_0">如果没有听到老师讲的，可以参考：https:&#47;&#47;blog.csdn.net&#47;guolin_blog&#47;article&#47;details&#47;104546558，写的非常好</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-27 14:36:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/41/be/4893497a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>野猪佩奇</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，我之前听说获取CA证书都是需要给钱的，这边怎么一条命令就能获取一个证书啊？望解惑，谢谢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我自己做ca当然可以啊，只不过拿出去不算</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-07-17 22:17:30</div>
  </div>
</div>
</div>
</li>
</ul>