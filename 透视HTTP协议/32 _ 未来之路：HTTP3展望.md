<audio title="32 _ 未来之路：HTTP3展望" src="https://static001.geekbang.org/resource/audio/3e/4d/3e41a53d39f155573182179cc6c8634d.mp3" controls="controls"></audio> 
<p>在前面的两讲里，我们一起学习了HTTP/2，你也应该看到了HTTP/2做出的许多努力，比如头部压缩、二进制分帧、虚拟的“流”与多路复用，性能方面比HTTP/1有了很大的提升，“基本上”解决了“队头阻塞”这个“老大难”问题。</p><h2>HTTP/2的“队头阻塞”</h2><p>等等，你可能要发出疑问了：为什么说是“基本上”，而不是“完全”解决了呢？</p><p>这是因为HTTP/2虽然使用“帧”“流”“多路复用”，没有了“队头阻塞”，但这些手段都是在应用层里，而在下层，也就是TCP协议里，还是会发生“队头阻塞”。</p><p>这是怎么回事呢？</p><p>让我们从协议栈的角度来仔细看一下。在HTTP/2把多个“请求-响应”分解成流，交给TCP后，TCP会再拆成更小的包依次发送（其实在TCP里应该叫segment，也就是“段”）。</p><p>在网络良好的情况下，包可以很快送达目的地。但如果网络质量比较差，像手机上网的时候，就有可能会丢包。而TCP为了保证可靠传输，有个特别的“丢包重传”机制，丢失的包必须要等待重新传输确认，其他的包即使已经收到了，也只能放在缓冲区里，上层的应用拿不出来，只能“干着急”。</p><p>我举个简单的例子：</p><p>客户端用TCP发送了三个包，但服务器所在的操作系统只收到了后两个包，第一个包丢了。那么内核里的TCP协议栈就只能把已经收到的包暂存起来，“停下”等着客户端重传那个丢失的包，这样就又出现了“队头阻塞”。</p><!-- [[[read_end]]] --><p>由于这种“队头阻塞”是TCP协议固有的，所以HTTP/2即使设计出再多的“花样”也无法解决。</p><p>Google在推SPDY的时候就已经意识到了这个问题，于是就又发明了一个新的“QUIC”协议，让HTTP跑在QUIC上而不是TCP上。</p><p>而这个“HTTP over QUIC”就是HTTP协议的下一个大版本，<strong>HTTP/3</strong>。它在HTTP/2的基础上又实现了质的飞跃，真正“完美”地解决了“队头阻塞”问题。</p><p>不过HTTP/3目前还处于草案阶段，正式发布前可能会有变动，所以今天我尽量不谈那些不稳定的细节。</p><p>这里先贴一下HTTP/3的协议栈图，让你对它有个大概的了解。</p><p><img src="https://static001.geekbang.org/resource/image/d2/03/d263202e431c84db0fd6c7e6b1980f03.png?wh=1235*545" alt=""></p><h2>QUIC协议</h2><p>从这张图里，你可以看到HTTP/3有一个关键的改变，那就是它把下层的TCP“抽掉”了，换成了UDP。因为UDP是无序的，包之间没有依赖关系，所以就从根本上解决了“队头阻塞”。</p><p>你一定知道，UDP是一个简单、不可靠的传输协议，只是对IP协议的一层很薄的包装，和TCP相比，它实际应用的较少。</p><p>不过正是因为它简单，不需要建连和断连，通信成本低，也就非常灵活、高效，“可塑性”很强。</p><p>所以，QUIC就选定了UDP，在它之上把TCP的那一套连接管理、拥塞窗口、流量控制等“搬”了过来，“去其糟粕，取其精华”，打造出了一个全新的可靠传输协议，可以认为是“<strong>新时代的TCP</strong>”。</p><p><img src="https://static001.geekbang.org/resource/image/fd/7a/fd99221ede55272a998760cc6aaa037a.png?wh=1142*388" alt="unpreview"></p><p>QUIC最早是由Google发明的，被称为gQUIC。而当前正在由IETF标准化的QUIC被称为iQUIC。两者的差异非常大，甚至比当年的SPDY与HTTP/2的差异还要大。</p><p>gQUIC混合了UDP、TLS、HTTP，是一个应用层的协议。而IETF则对gQUIC做了“清理”，把应用部分分离出来，形成了HTTP/3，原来的UDP部分“下放”到了传输层，所以iQUIC有时候也叫“QUIC-transport”。</p><p>接下来要说的QUIC都是指iQUIC，要记住，它与早期的gQUIC不同，是一个传输层的协议，和TCP是平级的。</p><h2>QUIC的特点</h2><p>QUIC基于UDP，而UDP是“无连接”的，根本就不需要“握手”和“挥手”，所以天生就要比TCP快。</p><p>就像TCP在IP的基础上实现了可靠传输一样，QUIC也基于UDP实现了可靠传输，保证数据一定能够抵达目的地。它还引入了类似HTTP/2的“流”和“多路复用”，单个“流”是有序的，可能会因为丢包而阻塞，但其他“流”不会受到影响。</p><p>为了防止网络上的中间设备（Middle Box）识别协议的细节，QUIC全面采用加密通信，可以很好地抵御窜改和“协议僵化”（ossification）。</p><p>而且，因为TLS1.3已经在去年（2018）正式发布，所以QUIC就直接应用了TLS1.3，顺便也就获得了0-RTT、1-RTT连接的好处。</p><p>但QUIC并不是建立在TLS之上，而是内部“包含”了TLS。它使用自己的帧“接管”了TLS里的“记录”，握手消息、警报消息都不使用TLS记录，直接封装成QUIC的帧发送，省掉了一次开销。</p><h2>QUIC内部细节</h2><p>由于QUIC在协议栈里比较偏底层，所以我只简略介绍两个内部的关键知识点。</p><p>QUIC的基本数据传输单位是<strong>包</strong>（packet）和<strong>帧</strong>（frame），一个包由多个帧组成，包面向的是“连接”，帧面向的是“流”。</p><p>QUIC使用不透明的“<strong>连接ID</strong>”来标记通信的两个端点，客户端和服务器可以自行选择一组ID来标记自己，这样就解除了TCP里连接对“IP地址+端口”（即常说的四元组）的强绑定，支持“<strong>连接迁移</strong>”（Connection Migration）。</p><p><img src="https://static001.geekbang.org/resource/image/ae/3b/ae0c482ea0c3b8ebc71924b19feb9b3b.png?wh=1237*715" alt=""></p><p>比如你下班回家，手机会自动由4G切换到WiFi。这时IP地址会发生变化，TCP就必须重新建立连接。而QUIC连接里的两端连接ID不会变，所以连接在“逻辑上”没有中断，它就可以在新的IP地址上继续使用之前的连接，消除重连的成本，实现连接的无缝迁移。</p><p>QUIC的帧里有多种类型，PING、ACK等帧用于管理连接，而STREAM帧专门用来实现流。</p><p>QUIC里的流与HTTP/2的流非常相似，也是帧的序列，你可以对比着来理解。但HTTP/2里的流都是双向的，而QUIC则分为双向流和单向流。</p><p><img src="https://static001.geekbang.org/resource/image/9a/10/9ab3858bf918dffafa275c400d78d910.png?wh=1270*619" alt=""></p><p>QUIC帧普遍采用变长编码，最少只要1个字节，最多有8个字节。流ID的最大可用位数是62，数量上比HTTP/2的2^31大大增加。</p><p>流ID还保留了最低两位用作标志，第1位标记流的发起者，0表示客户端，1表示服务器；第2位标记流的方向，0表示双向流，1表示单向流。</p><p>所以QUIC流ID的奇偶性质和HTTP/2刚好相反，客户端的ID是偶数，从0开始计数。</p><h2>HTTP/3协议</h2><p>了解了QUIC之后，再来看HTTP/3就容易多了。</p><p>因为QUIC本身就已经支持了加密、流和多路复用，所以HTTP/3的工作减轻了很多，把流控制都交给QUIC去做。调用的不再是TLS的安全接口，也不是Socket API，而是专门的QUIC函数。不过这个“QUIC函数”还没有形成标准，必须要绑定到某一个具体的实现库。</p><p>HTTP/3里仍然使用流来发送“请求-响应”，但它自身不需要像HTTP/2那样再去定义流，而是直接使用QUIC的流，相当于做了一个“概念映射”。</p><p>HTTP/3里的“双向流”可以完全对应到HTTP/2的流，而“单向流”在HTTP/3里用来实现控制和推送，近似地对应HTTP/2的0号流。</p><p>由于流管理被“下放”到了QUIC，所以HTTP/3里帧的结构也变简单了。</p><p>帧头只有两个字段：类型和长度，而且同样都采用变长编码，最小只需要两个字节。</p><p><img src="https://static001.geekbang.org/resource/image/26/5b/2606cbaa1a2e606a3640cc1825f5605b.png?wh=1262*430" alt=""></p><p>HTTP/3里的帧仍然分成数据帧和控制帧两类，HEADERS帧和DATA帧传输数据，但其他一些帧因为在下层的QUIC里有了替代，所以在HTTP/3里就都消失了，比如RST_STREAM、WINDOW_UPDATE、PING等。</p><p>头部压缩算法在HTTP/3里升级成了“<strong>QPACK</strong>”，使用方式上也做了改变。虽然也分成静态表和动态表，但在流上发送HEADERS帧时不能更新字段，只能引用，索引表的更新需要在专门的单向流上发送指令来管理，解决了HPACK的“队头阻塞”问题。</p><p>另外，QPACK的字典也做了优化，静态表由之前的61个增加到了98个，而且序号从0开始，也就是说“:authority”的编号是0。</p><h2>HTTP/3服务发现</h2><p>讲了这么多，不知道你注意到了没有：HTTP/3没有指定默认的端口号，也就是说不一定非要在UDP的80或者443上提供HTTP/3服务。</p><p>那么，该怎么“发现”HTTP/3呢？</p><p>这就要用到HTTP/2里的“扩展帧”了。浏览器需要先用HTTP/2协议连接服务器，然后服务器可以在启动HTTP/2连接后发送一个“<strong>Alt-Svc</strong>”帧，包含一个“h3=host:port”的字符串，告诉浏览器在另一个端点上提供等价的HTTP/3服务。</p><p>浏览器收到“Alt-Svc”帧，会使用QUIC异步连接指定的端口，如果连接成功，就会断开HTTP/2连接，改用新的HTTP/3收发数据。</p><h2>小结</h2><p>HTTP/3综合了我们之前讲的所有技术（HTTP/1、SSL/TLS、HTTP/2），包含知识点很多，比如队头阻塞、0-RTT握手、虚拟的“流”、多路复用，算得上是“集大成之作”，需要多下些功夫好好体会。</p><ol>
<li><span class="orange">HTTP/3基于QUIC协议，完全解决了“队头阻塞”问题，弱网环境下的表现会优于HTTP/2；</span></li>
<li><span class="orange">QUIC是一个新的传输层协议，建立在UDP之上，实现了可靠传输；</span></li>
<li><span class="orange">QUIC内含了TLS1.3，只能加密通信，支持0-RTT快速建连；</span></li>
<li><span class="orange">QUIC的连接使用“不透明”的连接ID，不绑定在“IP地址+端口”上，支持“连接迁移”；</span></li>
<li><span class="orange">QUIC的流与HTTP/2的流很相似，但分为双向流和单向流；</span></li>
<li><span class="orange">HTTP/3没有指定默认端口号，需要用HTTP/2的扩展帧“Alt-Svc”来发现。</span></li>
</ol><h2>课下作业</h2><ol>
<li>IP协议要比UDP协议省去8个字节的成本，也更通用，QUIC为什么不构建在IP协议之上呢？</li>
<li>说一说你理解的QUIC、HTTP/3的好处。</li>
<li>对比一下HTTP/3和HTTP/2各自的流、帧，有什么相同点和不同点。</li>
</ol><p>欢迎你把自己的学习体会写在留言区，与我和其他同学一起讨论。如果你觉得有所收获，也欢迎把文章分享给你的朋友。</p><p><img src="https://static001.geekbang.org/resource/image/58/df/5857f14a3b06b6c0dd38e00b4a6124df.png?wh=1769*3848" alt="unpreview"></p><p></p>
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
  <div class="_2_QraFYR_0">IP 协议要比 UDP 协议省去 8 个字节的成本，也更通用，QUIC 为什么不构建在 IP 协议之上呢？<br>直接利用UDP，兼容性好。<br>说一说你理解的 QUIC、HTTP&#47;3 的好处。<br>彻底解决队头阻塞，用户态定义流量控制、拥塞避免等算法，优化慢启动、弱网、重建连接等问题。<br>对比一下 HTTP&#47;3 和 HTTP&#47;2 各自的流、帧，有什么相同点和不同点。<br>HTTP&#47;3在QUIC层定义流、帧，真正解决队头阻塞，HTTP&#47;2流、帧是在TCP层上抽象出的逻辑概念。<br>相同点是在逻辑理解上是基本一致的，流由帧组成，多个流可以并发传输互不影响。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-09 13:49:11</div>
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
  <div class="_2_QraFYR_0">老师，以下问题，麻烦回答一下，谢谢：<br><br>1.它使用自己的帧“接管”了 TLS 里的“记录”，握手消息、警报消息都不使用 TLS 记录，直接封装成 QUIC 的帧发送，省掉了一次开销。省掉的一次开销是什么？<br><br>2.解决了 HPACK 的“队头阻塞”问题。 没明白这句话。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.省去了TLS封装步骤，也就没有了TLS帧头等相关信息，字节数等成本就减少了。<br><br>2.HPACK的字典需要双方传输同步，如果有的字典没有传过来就会阻塞，而HTTP&#47;3解决了这个问题。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-25 10:49:01</div>
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
  <div class="_2_QraFYR_0">（1）http的队头阻塞，和tcp的队头阻塞，怎么理解 ？是由于tcp队头阻塞导致http对头阻塞，还是http本身的实现就会造成队头阻塞，还是都有。感觉有点模糊？<br>（2）看完了QUIC，其流内部还是会产生队头阻塞，感觉没啥区别，QUIC内部还不是要实现tcp的重传那一套东西。QUIC没看出来比tcp好在哪里。<br>（3）队头阻塞在http，tcp，流等这几个概念中是怎么理解和区分的，很迷惑。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.队头阻塞在tcp和http层都存在，原因不同。http&#47;2解决了http的队头阻塞，在http层是没有问题了，但在tcp层还有队头阻塞，所以会影响传输效率。<br><br>2.一个流就是一个请求响应，它阻塞不会影响其他流，所以不会发生队头阻塞。<br><br>3.QUIC有很多优于tcp的新特性，例如连接迁移、多路复用，加密。<br><br>4.队头阻塞确实不太好理解，可以再看看之前的课程，结合示意图来加深体会。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-09 12:07:40</div>
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
  <div class="_2_QraFYR_0">1.传输层TCP和UDP就够了，在多加会提高复杂度，基于UDP向前兼容会好一些。<br>2.在传输层解决了队首阻塞，基于UDP协议，在网络拥堵的情况下，提高传输效率<br>3.http3在传输层基于UDP真正解决了队头阻塞。http2只是部分解决。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-09 08:42:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/62/dc/8876c73b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>moooofly</span>
  </div>
  <div class="_2_QraFYR_0">我是不是可以这样理解，QUIC 之所以解决了队头阻塞，是基于UDP的乱序，无连接，以包为单位进行传出的特性，即当发生丢包时，当前流中对应的请求或应答就彻底“丢失”了，之后只需要通过在UDP基础实现的“可靠传输”功能，重传就好了，这样就避免了接收端死等尚未接收到的数据的“干着急”状态；</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 基本正确，udp的各个包之间没有依赖关系，所以就不会出现队头阻塞。<br><br>quic里的另一个关键是流概念，它和http&#47;2的一样，把多个请求响应解耦，互不干扰。<br><br>这样在上层应用层和下层的传输层就都没有了队头阻塞，但因为丢包而重传该等还是会等，只是不会影响其他的流。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-31 14:06:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/29/b0/d3/200e82ff.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>功夫熊猫</span>
  </div>
  <div class="_2_QraFYR_0">udp虽然可以节省时间和速度比tcp快，但是如果传输的是那种很机密的东西的时候，但是如何保证udp传输的数据是没有丢失的，（所以udp一般是传输视频，图片之类的东西吧）是换tcp还是对udp进行改装，还是http&#47;3有什么特殊的方法</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 在udp之上，quic有自己的丢包重传机制。<br><br>其实在tcp里也是有丢包的，它自己来保证数据的完整可达，类似可以去理解quic。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-31 09:28:30</div>
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
  <div class="_2_QraFYR_0">老师 请问下这句话怎么理解呢  HTTP&#47;2 那样再去定义流</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 在quic里已经有流的概念了，所以http&#47;3就不需要再用一大段文字来定义流的格式、行为等，直接引用quic就可以。<br><br>而http&#47;2里的流概念是全新的，下面的tcp、tls没有，就必须花上很多篇幅去说明。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-05 22:33:18</div>
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
  <div class="_2_QraFYR_0">gQUIC 混合了 UDP、TLS、HTTP，是一个应用层的协议。而 IETF 则对 gQUIC 做了“清理”，把应用部分分离出来，形成了 HTTP&#47;3，原来的 UDP 部分“下放”到了传输层，所以 iQUIC 有时候也叫“QUIC-transport”。接下来要说的 QUIC 都是指 iQUIC，要记住，它与早期的 gQUIC 不同，是一个传输层的协议，和 TCP 是平级的<br><br>老师问下这一段最后为什么说iQUIC是传输层协议？本来gQUIC是应用层协议，去掉传输层部分后反而变成了传输层协议吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: gQUIC把udp&#47;tls&#47;http混合在一起，所以是应用层协议。而iQUIC只专注于数据传输，是gQUIC的udp+tls部分，http那部分变成了http&#47;3。<br><br>可以去看rfc9000，里面写的很清楚。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-12 16:58:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/e8/43/f9c0faed.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小童</span>
  </div>
  <div class="_2_QraFYR_0">老师这个QUIC 是如何保证UDP的 可靠传输？还是没看明白。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 底层的细节比较复杂，一下子也说不清楚，可以类比一下tcp和ip，quic就是在udp之上加了tcp的那套重传校验机制。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-28 05:29:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/ee/c2/873cc8d9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Rick</span>
  </div>
  <div class="_2_QraFYR_0">请问连接迁移是如何做到的?毕竟它依赖于udp，而udp使用了ip&#47;port。当一个连接的一端从一个ip&#47;port转移到另外一个ip&#47;port上的时候，怎么通知对端呢？需要使用QUIC的控制帧来完成吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个跟控制端无关，因为在quic里标记连接的是连接id，所以两边都用这个id来标记连接，在quic这层是看不到ip+port的，只有id，这个道理和四元组是一样的。<br><br>至于具体如何做到，那就要看rfc了，不过我个人觉得如果是做应用，了解的太细不是很有必要。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-20 18:27:09</div>
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
  <div class="_2_QraFYR_0">老师您好，想向老师请教一下学习方法的问题<br>学习HTTP协议一直学习到这里，发现老师学识太渊博了，这得需要好多年的积累吧<br>像我这样初学网络、HTTP协议的，老师有什么好的建议么<br>感谢老师的回复</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感觉学习好像也没有捷径啊，结合自身的实际情况，看好大方向，找准几个关键点去学吧。<br><br>还可以看极客时间上其他一些老师的感悟，路还是要自己走。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-16 14:26:53</div>
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
  <div class="_2_QraFYR_0">浏览器需要先用 HTTP&#47;2 协议连接服务器，然后服务器可以在启动 HTTP&#47;2 连接后发送一个“Alt-Svc”帧，包含一个“h3=host:port”的字符串，告诉浏览器在另一个端点上提供等价的 HTTP&#47;3 服务。<br>老师，这里的意思是指HTTP&#47;3包含了HTTP&#47;2的这部分功能，还是HTTP&#47;3的使用必须依赖HTTP&#47;2？<br>另外，QUIC中的包是一个完整的请求或响应报文？否则多个包的内容才能组成一个完整的请求或响应报文，必然也需要等待所有包都到齐了，组装一下吧？假如你一个包，这个包得多大？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.目前的http&#47;3草案来看，这个是http&#47;3建立连接的要求，也就是要从http&#47;2升级。<br><br>2.quic的一个包里有多个帧，多个帧组成一个报文，包只是把帧整合在一起而已，实际上用的还是帧。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-04 17:59:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/42/44/d3d67640.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Hills录</span>
  </div>
  <div class="_2_QraFYR_0">课后1：QUIC 不基于 IP 协议，是因为没有设备认识它<br>课后2：HTTP&#47;3 端口不固定、内容天然加密、连接迁移等特性，让互联网回归自由</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-27 10:00:52</div>
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
  <div class="_2_QraFYR_0">老师，文中有一张包的结构图，Quic Package Payload 里面说『实际传输的数据是多个帧构成的流』，这里怎么理解呢？<br>是这样吗，Quic里面有帧、流、包的概念，流上传输的是帧，Quic是把多个流结合为包然后传递给UDP吗，因为每个流是一个消息，丢包的时候会一次丢多个消息吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: quic里的流由多个帧组成，这些帧会组合在一个包里，也就是说一个包里会有多个流的帧，但不是一个包就包含了完整的流，而是会有多个包（里面多个帧）然后才能是一个完整的流。<br><br>丢包只会丢失部分帧，quic也只会重传这些丢失的帧。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-22 11:50:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/62/dc/8876c73b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>moooofly</span>
  </div>
  <div class="_2_QraFYR_0">“下班回家，手机会自动由 4G 切换到 WiFi。这时 IP 地址会发生变化，TCP 就必须重新建立连接。而 QUIC 连接里的两端连接 ID 不会变，所以连接在“逻辑上”没有中断，它就可以在新的 IP 地址上继续使用之前的连接，消除重连的成本，实现连接的无缝迁移”，我觉得这里是不是应该强调一下，QUIC 是基于无连接概念的 UDP 协议，因此也就没有所谓的“中断”和“重连”概念，进而才能实现在新的 ip 地址上的无缝迁移；</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 在udp协议之上也可以实现有连接的协议，比如有的公司的自定义协议。<br><br>quic底层的udp提供了无连接的基础，但它是有连接的，连接迁移的功能还是quic自己实现的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-31 14:34:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/62/dc/8876c73b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>moooofly</span>
  </div>
  <div class="_2_QraFYR_0">一个小建议：既然 TLS1.3 是被“包含”在 QUIC 协议中的，那么文章中给出的 HTTP&#47;3 协议栈图，就有点容易让人产生误会，图示给人的感觉是 QUIC 和 TLS1.3 是一个级别的对等存在，让人感觉 QPack 是基于 QUIC 的，而 Stream 是基于 TLS1.3 实现的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，quic和tls的关系比较复杂，不是直接的上下级关系，但画图为了“美观”给画成了这样，所以示意图只是“示意”，还是要结合文字来全面理解。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-31 14:10:12</div>
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
  <div class="_2_QraFYR_0">老师好!<br>协议处在哪一次有什么划分标准么?<br>mac层和ip成感觉一般不怎么会变<br>传输层和应用层搞不太清楚</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 传输层就是只负责传输数据，不关心数据的具体内容。<br><br>应用层通常不关心数据传输的细节，关心的是如何处理数据，解析数据格式。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-09 10:00:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/7b/57/a9b04544.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>QQ怪</span>
  </div>
  <div class="_2_QraFYR_0">老师能否分享下要更新换代http3，其上层的服务协议是否也要更新还是都能够兼容？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 和http&#47;2一样，http&#47;3是完全兼容的，因为语义还是保持不变。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-09 08:52:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/c0/f0/1aabc056.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Jiantao</span>
  </div>
  <div class="_2_QraFYR_0">老师，文中提到h3&#47;quic是包含tls的，意思是要使用h3&#47;quic 必须要求域名是https吗<br><br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，http&#47;3强制加密，所以一定是https。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-07 09:56:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q3auHgzwzM74658w9PQeTM4TcM14BzfpJnVLrsciaS26ibRwRbCE09ydI6UlZhFrJh7iaVLp2xxhBppVDKLyRRg9Q/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_21a73c</span>
  </div>
  <div class="_2_QraFYR_0">老师，基于UDP的QUIC如何保证数据一定抵达目的地呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 道理和IP层发数据包是一样的，QUIC层面发现丢包会重传。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-20 23:13:33</div>
  </div>
</div>
</div>
</li>
</ul>