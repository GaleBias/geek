<audio title="44 _ 先睹为快：HTTP3实验版本长什么样子？" src="https://static001.geekbang.org/resource/audio/75/4a/756326afb9b24943b205e78feae33b4a.mp3" controls="controls"></audio> 
<p>你好，我是Chrono。</p><p>不知不觉，《透视HTTP协议》这个专栏马上就要两周岁了。前几天我看了一下专栏的相关信息，订阅数刚好破万，非常感谢包括你在内的所有朋友们的关心、支持、鼓励和鞭策。</p><p>在专栏的结束语里我曾经说过，希望HTTP/3发布之时能够再相会。而如今虽然它还没有发布，但也为时不远了。</p><p>所以今天呢，我就来和你聊聊HTTP/3的一些事，就当是“尝尝鲜”吧。</p><h2>HTTP/3的现状</h2><p>从2019到2021的这两年间，大家对HTTP协议的关注重点差不多全都是放在HTTP/3标准的制订上。</p><p>最初专栏开始的时候，HTTP/3草案还是第20版，而现在则已经是第34版了，发展的速度可以说是非常快的，里面的内容也变动得非常多。很有可能最多再过一年，甚至是今年内，我们就可以看到正式的标准。</p><p>在标准文档的制订过程中，互联网业届也没有闲着，也在积极地为此做准备，以草案为基础做各种实验性质的开发。</p><p>这其中比较引人瞩目的要数CDN大厂Cloudflare，还有Web Server领头羊Nginx（而另一个Web Server Apache好像没什么动静）了。</p><p>Cloudflare公司用Rust语言编写了一个QUIC支持库，名字叫“quiche”，然后在上面加了一层薄薄的封装，由此能够以一个C模块的形式加入进Nginx框架，为Nginx提供了HTTP/3的功能。（可以参考这篇文章：<a href="https://blog.cloudflare.com/zh-cn/http3-the-past-present-and-future-zh-cn/">HTTP/3：过去，现在，还有未来</a>）</p><!-- [[[read_end]]] --><p>不过Cloudflare的这个QUIC支持库属于“民间行为”，没有得到Nginx的认可。Nginx的官方HTTP/3模块其实一直在“秘密”开发中，在去年的6月份，这个模块终于正式公布了，名字就叫“http_v3_module”。（可以参考这篇文章：<a href="https://www.nginx.com/blog/introducing-technology-preview-nginx-support-for-quic-http-3/">Introducing a Technology Preview of NGINX Support for QUIC and HTTP/3</a>）</p><p>目前，http_v3_module已经度过了Alpha阶段，处于Beta状态，但支持的草案版本是29，而不是最新的34。</p><p>这当然也有情可原。相比于HTTP/2，HTTP/3的变化太大了，Nginx团队的精力还是集中在核心功能实现上，选择一个稳定的版本更有利于开发，而且29后面的多个版本标准其实差异非常小（仅文字编辑变更）。</p><p>Nginx也为测试HTTP/3专门搭建了一个网站：<a href="https://quic.nginx.org/">quic.nginx.org</a>，任何人都可以上去测试验证HTTP/3协议。</p><p>所以，接下来我们就用它来看看HTTP/3到底长什么样。</p><h2>初识HTTP/3</h2><p>在体验之前，得先说一下浏览器，这是测试QUIC和HTTP/3的关键：最好使用最新版本的Chrome或者Firefox，这里我用的是Chrome88。</p><p>打开浏览器窗口，输入测试网站的URI（<a href="https://quic.nginx.org/">https://quic.nginx.org/</a>），如果“运气好”，刷新几次就能够在网页上看到大大的QUIC标志了。</p><p><img src="https://static001.geekbang.org/resource/image/82/e0/827d261f49f6a20eb227f851dec667e0.png?wh=986*398" alt=""></p><p>不过你很可能“运气”没有这么好，在网页上看到的QUIC标志是灰色的。这意味着暂时没有应用QUIC和HTTP/3，这就需要对Chrome做一点设置，开启QUIC的支持。</p><p>首先要在地址栏输入“chrome://flags”，打开设置页面，然后搜索“QUIC”，找到启用QUIC的选项，把它改为“Enabled”，具体可以参考下面的图片。</p><p><img src="https://static001.geekbang.org/resource/image/67/65/674ff32bf05b5859fb6985e29f8c1e65.png?wh=1724*716" alt=""></p><p>接下来，我们要在命令行里启动Chrome浏览器，在命令行里传递“enable-quic”“quic-version”等参数来要求Chrome使用特定的草案版本。</p><p>下面的示例就是我在macOS上运行Chrome的命令行。你也可以参考Nginx网站上的README文档，在Windows或者Linux上用类似的形式运行Chrome的命令行：</p><pre><code>/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome \
--enable-quic --quic-version=h3-29 \
--origin-to-force-quic-on=quic.nginx.org:443
</code></pre><p>如果这样操作之后网页上仍然是显示灰色标志也不要紧，你还可以用“F12”打开Chrome的开发者工具面板，查看protocol一栏。</p><p>应该可以看到大部分资源显示的还是“h2”，表示使用的是HTTP/2协议，但有一小部分资源显示的是“h3-29”，这就表示它是使用HTTP/3协议传输的，而后面的“29”后缀，意思是基于第29版草案，也就是说启用了QUIC+HTTP/3协议。</p><p><img src="https://static001.geekbang.org/resource/image/c3/e6/c3a532736412a4457ee81a280fc76be6.png?wh=1664*1320" alt=""></p><h2>Wireshark抓包分析</h2><p>好了，大概看了HTTP/3是什么样，有了感性认识，我们就可以进一步来抓包分析。</p><p>网络抓包工具Wireshark你一定已经比较熟悉了，这里同样要用最新的，不然可能识别不了QUIC和HTTP/3的数据包，比如我用的就是3.4.3。</p><p>QUIC的底层是UDP，所以在抓包的时候过滤器要设置成“udp port 443”，然后启动就可以了。这次我抓的包也放到了GitHub的<a href="https://github.com/chronolaw/http_study/tree/master/wireshark">Wireshark目录</a>，文件名是“44-1.pcapng”。</p><p><img src="https://static001.geekbang.org/resource/image/6d/d4/6d217ee87e1f777d432059f81fc2f5d4.png?wh=1758*1132" alt=""></p><p>因为HTTP/3内置了TLS加密（可参考之前的<a href="https://time.geekbang.org/column/article/115564">第32讲</a>），所以用Wireshark抓包后看到的数据大部分都是乱码，想要解密看到真实数据就必须设置SSLKEYLOG（参考<a href="https://time.geekbang.org/column/article/110354">第26讲</a>）。</p><p>不过非常遗憾，不知道是什么原因，虽然我导出了SSLKEYLOG，但在Wireshark里还是无法解密HTTP/3的数据，显示解密错误。但同样的设置和操作步骤，抓包解密HTTPS和HTTP/2却是正常的，估计可能是目前Wireshark自身对HTTP/3的支持还不太完善吧。</p><p>所以今天我也就只能带你一起来看QUIC的握手阶段了。这个其实与TLS1.3非常接近，只不过是内嵌在了QUIC协议里，如果你学过了“安全篇”“飞翔篇”的课程，看QUIC应该是不会费什么力气。</p><p>首先我们来看Header数据：</p><pre><code>[Packet Length: 1350]
1... .... = Header Form: Long Header (1)
.1.. .... = Fixed Bit: True
..00 .... = Packet Type: Initial (0)
.... 00.. = Reserved: 0
.... ..00 = Packet Number Length: 1 bytes (0)
Version: draft-29 (0xff00001d)
Destination Connection ID Length: 20
Destination Connection ID: 3ae893fa047246e55f963ea14fc5ecac3774f61e
Source Connection ID Length: 0
</code></pre><p>QUIC包头的第一个字节是标志位，可以看到最开始建立连接会发一个长包（Long Header），包类型是初始化（Initial）。</p><p>标志位字节后面是4字节的版本号，因为目前还是草案，所以显示的是“draft-29”。再后面，是QUIC的特性之一“连接ID”，长度为20字节的十六进制字符串。</p><p>这里我要特别提醒你注意，因为标准版本的演变，这个格式已经与当初<a href="https://time.geekbang.org/column/article/115564">第32讲</a>的内容（draft-20）完全不一样了，在分析查看的时候一定要使用<a href="https://tools.ietf.org/html/draft-ietf-quic-transport-28#section-17.2">对应的RFC文档</a>。</p><p>往下再看，是QUIC的CRYPTO帧，用来传输握手消息，帧类型是0x06：</p><pre><code>TLSv1.3 Record Layer: Handshake Protocol: Client Hello
    Frame Type: CRYPTO (0x0000000000000006)
    Offset: 0
    Length: 309
    Crypto Data
    Handshake Protocol: Client Hello
</code></pre><p>CRYPTO帧里的数据，就是QUIC内置的TLS “Client Hello”了，我把里面的一些重要信息摘了出来：</p><pre><code>Handshake Protocol: Client Hello
   Handshake Type: Client Hello (1)
   Version: TLS 1.2 (0x0303)
   Random: b4613d...
   Cipher Suites (3 suites)
       Cipher Suite: TLS_AES_128_GCM_SHA256 (0x1301)
       Cipher Suite: TLS_AES_256_GCM_SHA384 (0x1302)
       Cipher Suite: TLS_CHACHA20_POLY1305_SHA256 (0x1303)
   Extension: server_name (len=19)
       Type: server_name (0)
       Server Name Indication extension
           Server Name: quic.nginx.org
   Extension: application_layer_protocol_negotiation (len=8)
       Type: application_layer_protocol_negotiation (16)
       ALPN Protocol
           ALPN Next Protocol: h3-29
   Extension: key_share (len=38)
       Key Share extension
   Extension: supported_versions (len=3)
       Type: supported_versions (43)
       Supported Version: TLS 1.3 (0x0304)
</code></pre><p>你看，这个就是标准的TLS1.3数据（伪装成了TLS1.2），支持AES128、AES256、CHACHA20三个密码套件，SNI是“quic.nginx.org”，ALPN是“h3-29”。</p><p>浏览器发送完Initial消息之后，服务器回复Handshake，用一个RTT就完成了握手，包的格式基本一样，用了一个CRYPTO帧和ACK帧，我就不细分析了（可参考<a href="https://tools.ietf.org/html/draft-ietf-quic-transport-28#section-17.2">相应的RFC</a>），只贴一下里面的“Server Hello”信息：</p><pre><code>Handshake Protocol: Server Hello
    Handshake Type: Server Hello (2)
    Version: TLS 1.2 (0x0303)
    Random: d6aede...
    Cipher Suite: TLS_AES_128_GCM_SHA256 (0x1301)
    Extension: key_share (len=36)
        Key Share extension
    Extension: supported_versions (len=2)
        Type: supported_versions (43)
        Supported Version: TLS 1.3 (0x0304)
</code></pre><p>这里服务器选择了“TLS_AES_128_GCM_SHA256”，然后带回了随机数和key_share，完成了握手阶段的密钥交换。</p><h2>小结</h2><p>好了，QUIC和HTTP/3的“抢鲜体验”就到这里吧，我简单小结一下今天的要点：</p><ol>
<li>HTTP/3的最新草案版本是34，很快就会发布正式版本。</li>
<li>Nginx提供了对HTTP/3的实验性质支持，目前是Beta状态，只能用于测试。</li>
<li>最新版本的Chrome和Firefox都支持QUIC和HTTP/3，但可能需要一些设置工作才能启用。</li>
<li>访问专门的测试网站“quic.nginx.org”可以检查浏览器是否支持QUIC和HTTP/3。</li>
<li>抓包分析QUIC和HTTP/3需要用最新的Wireshark，过滤器用UDP，还要导出SSLKEYLOG才能解密。</li>
</ol><p>希望你看完这一讲后自己实际动手操作一下，访问网站再抓包，如果能正确解密HTTP/3数据，就把资料发出来，和我们分享下。</p><p>如果你觉得有所收获，也欢迎把这一讲的内容分享给你的朋友。</p><p><img src="https://static001.geekbang.org/resource/image/1b/4f/1b4266dcedc5785f3023f47083e4894f.jpg?wh=2800*4221" alt=""></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/94/56/4b8395f6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>CC</span>
  </div>
  <div class="_2_QraFYR_0">惊喜，刚刚好在学习第二遍，看到 HTTP&#47;3 的新文章更新。谢谢老师。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 学而时习之，不亦乐乎。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-02 10:48:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/22/97/d0/95eaabef.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>水手辛伯达</span>
  </div>
  <div class="_2_QraFYR_0">第二遍学习Chrono老师的http协议。这门课分层清晰，环环相扣，由简入繁，对于初中级前端和运维人员去了解http协议帮助是比较大的！最难能可贵的是，老师前后两年时间一直坚持更新，又增加了docker试验环节和http3的发展updates , 同时，老师十分注意和学员的互动，而且几乎和每个留言进行点评和分析，这些课后答疑又极大地分丰富了大家的知识，增涨了经验，十分庆幸在极客时间里面遇到这么优秀的课程！<br><br>祝老师健康，顺利！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢支持，大家共同进步。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-17 08:22:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/a0/a4/b060c723.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>阿斯蒂芬</span>
  </div>
  <div class="_2_QraFYR_0">为老师对课程的持续关注和技术更新的科普点赞！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Thanks</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-19 21:07:13</div>
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
  <div class="_2_QraFYR_0">打卡，真不错</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: nice</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-11 11:50:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/60/4e/1c654d86.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Omooo</span>
  </div>
  <div class="_2_QraFYR_0">牛逼！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-27 15:09:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/63/30/6f4b925c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Luca</span>
  </div>
  <div class="_2_QraFYR_0">更新到了Chrome89.0.4389.90，无需进行额外设置就能够有QUIC支持了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Chrome的行为确实令人迷惑，有的版本就不行，不过相信以后会越来越简单方便。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-23 19:35:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/24/b9/b9/9e4d7aa4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>乘风破浪</span>
  </div>
  <div class="_2_QraFYR_0">实测chrome版本 88.0.4324.190（正式版本）无需设置可以支持quic<br>firefox 86.0需要简单设置一下，具体页面搜firefox，第一条就是<br>wireshark3.4.3抓包结果和大师一样，payload解不出来<br>请问大师，现在学习HTTP&#47;3，现在如果要深入了解HTTP&#47;3,需要看rfc吧？有没有其他好的资源？<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我现在更新到88.0.4324.192，就不能直接显示出quic支持了。<br><br>目前HTTP&#47;3的资料还只有rfc，不过也不用太着急，等正式发布后估计就会有很多其他的分析研究资料了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-02 21:17:30</div>
  </div>
</div>
</div>
</li>
</ul>