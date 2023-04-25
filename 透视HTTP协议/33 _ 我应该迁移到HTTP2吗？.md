<audio title="33 _ 我应该迁移到HTTP2吗？" src="https://static001.geekbang.org/resource/audio/41/28/413e8241fbe9202ebe252b2b3dd0ad28.mp3" controls="controls"></audio> 
<p>这一讲是“飞翔篇”的最后一讲，而HTTP的所有知识也差不多快学完了。</p><p>前面你已经看到了新的HTTP/2和HTTP/3协议，了解了它们的特点和工作原理，如果再联系上前几天“安全篇”的HTTPS，你可能又会发出疑问：</p><p>“刚费了好大的力气升级到HTTPS，这又出了一个HTTP/2，还有再次升级的必要吗？”</p><p>与各大浏览器“强推”HTTPS的待遇不一样，HTTP/2的公布可谓是“波澜不惊”。虽然它是HTTP协议的一个重大升级，但Apple、Google等科技巨头并没有像HTTPS那样给予大量资源的支持。</p><p>直到今天，HTTP/2在互联网上还是处于“不温不火”的状态，虽然已经有了不少的网站改造升级到了HTTP/2，但普及的速度远不及HTTPS。</p><p>所以，你有这样的疑问也是很自然的，升级到HTTP/2究竟能给我们带来多少好处呢？到底“值不值”呢？</p><h2>HTTP/2的优点</h2><p>前面的几讲主要关注了HTTP/2的内部实现，今天我们就来看看它有哪些优点和缺点。</p><p>首先要说的是，HTTP/2最大的一个优点是<strong>完全保持了与HTTP/1的兼容</strong>，在语义上没有任何变化，之前在HTTP上的所有投入都不会浪费。</p><p>因为兼容HTTP/1，所以HTTP/2也具有HTTP/1的所有优点，并且“基本”解决了HTTP/1的所有缺点，安全与性能兼顾，可以认为是“<span class="orange">更安全的HTTP、更快的HTTPS</span>”。</p><!-- [[[read_end]]] --><p>在安全上，HTTP/2对HTTPS在各方面都做了强化。下层的TLS至少是1.2，而且只能使用前向安全的密码套件（即ECDHE），这同时也就默认实现了“TLS False Start”，支持1-RTT握手，所以不需要再加额外的配置就可以自动实现HTTPS加速。</p><p>安全有了保障，再来看HTTP/2在性能方面的改进。</p><p>你应该知道，影响网络速度的两个关键因素是“<strong>带宽</strong>”和“<strong>延迟</strong>”，HTTP/2的头部压缩、多路复用、流优先级、服务器推送等手段其实都是针对这两个要点。</p><p>所谓的“带宽”就是网络的传输速度。从最早的56K/s，到如今的100M/s，虽然网速已经是“今非昔比”，比从前快了几十倍、几百倍，但仍然是“稀缺资源”，图片、视频这样的多媒体数据很容易会把带宽用尽。</p><p>节约带宽的基本手段就是压缩，在HTTP/1里只能压缩body，而HTTP/2则可以用HPACK算法压缩header，这对高流量的网站非常有价值，有数据表明能节省大概5%~10%的流量，这是实实在在的“真金白银”。</p><p>与HTTP/1“并发多个连接”不同，HTTP/2的“多路复用”特性要求对<strong>一个域名（或者IP）只用一个TCP连接</strong>，所有的数据都在这一个连接上传输，这样不仅节约了客户端、服务器和网络的资源，还可以把带宽跑满，让TCP充分“吃饱”。</p><p>这是为什么呢？</p><p>我们来看一下在HTTP/1里的长连接，虽然是双向通信，但任意一个时间点实际上还是单向的：上行请求时下行空闲，下行响应时上行空闲，再加上“队头阻塞”，实际的带宽打了个“对折”还不止（可参考<a href="https://time.geekbang.org/column/article/104949">第17讲</a>）。</p><p>而在HTTP/2里，“多路复用”则让TCP开足了马力，“全速狂奔”，多个请求响应并发，每时每刻上下行方向上都有流在传输数据，没有空闲的时候，带宽的利用率能够接近100%。所以，HTTP/2只使用一个连接，就能抵得过HTTP/1里的五六个连接。</p><p>不过流也可能会有依赖关系，可能会存在等待导致的阻塞，这就是“延迟”，所以HTTP/2的其他特性就派上了用场。</p><p>“优先级”可以让客户端告诉服务器，哪个文件更重要，更需要优先传输，服务器就可以调高流的优先级，合理地分配有限的带宽资源，让高优先级的HTML、图片更快地到达客户端，尽早加载显示。</p><p>“服务器推送”也是降低延迟的有效手段，它不需要客户端预先请求，服务器直接就发给客户端，这就省去了客户端解析HTML再请求的时间。</p><h2>HTTP/2的缺点</h2><p>说了一大堆HTTP/2的优点，再来看看它有什么缺点吧。</p><p>听过上一讲HTTP/3的介绍，你就知道HTTP/2在TCP级别还是存在“队头阻塞”的问题。所以，如果网络连接质量差，发生丢包，那么TCP会等待重传，传输速度就会降低。</p><p>另外，在移动网络中发生IP地址切换的时候，下层的TCP必须重新建连，要再次“握手”，经历“慢启动”，而且之前连接里积累的HPACK字典也都消失了，必须重头开始计算，导致带宽浪费和时延。</p><p>刚才也说了，HTTP/2对一个域名只开一个连接，所以一旦这个连接出问题，那么整个网站的体验也就变差了。</p><p>而这些情况下HTTP/1反而不会受到影响，因为它“本来就慢”，而且还会对一个域名开6~8个连接，顶多其中的一两个连接会“更慢”，其他的连接不会受到影响。</p><h2>应该迁移到HTTP/2吗？</h2><p>说到这里，你对迁移到HTTP/2是否已经有了自己的判断呢？</p><p>在我看来，HTTP/2处于一个略“尴尬”的位置，前面有“老前辈”HTTP/1，后面有“新来者”HTTP/3，即有“老前辈”的“打压”，又有“新来者”的“追赶”，也就难怪没有获得市场的大力“吹捧”了。</p><p>但这绝不是说HTTP/2“一无是处”，实际上HTTP/2的性能改进效果是非常明显的，Top 1000的网站中已经有超过40%运行在了HTTP/2上，包括知名的Apple、Facebook、Google、Twitter等等。仅用了四年的时间，HTTP/2就拥有了这么大的市场份额和巨头的认可，足以证明它的价值。</p><p>因为HTTP/2的侧重点是“性能”，所以“是否迁移”就需要在这方面进行评估。如果网站的流量很大，那么HTTP/2就可以带来可观的收益；反之，如果网站流量比较小，那么升级到HTTP/2就没有太多必要了，只要利用现有的HTTP再优化就足矣。</p><p>不过如果你是新建网站，我觉得完全可以跳过HTTP/1、HTTPS，直接“一步到位”，上HTTP/2，这样不仅可以获得性能提升，还免去了老旧的“历史包袱”，日后也不会再有迁移的烦恼。</p><p>顺便再多嘴一句，HTTP/2毕竟是“下一代”HTTP协议，它的很多特性也延续到了HTTP/3，提早升级到HTTP/2还可以让你在HTTP/3到来时有更多的技术积累和储备，不至于落后于时代。</p><h2>配置HTTP/2</h2><p>假设你已经决定要使用HTTP/2，应该如何搭建服务呢？</p><p>因为HTTP/2“事实上”是加密的，所以如果你已经在“安全篇”里成功迁移到了HTTPS，那么在Nginx里启用HTTP/2简直可以说是“不费吹灰之力”，只需要在server配置里再多加一个参数就可以搞定了。</p><pre><code>server {
    listen       443 ssl http2;


    server_name  www.xxx.net;


    ssl_certificate         xxx.crt;
    ssl_certificate_key     xxx.key;
</code></pre><p>注意“listen”指令，在“ssl”后面多了一个“http2”，这就表示在443端口上开启了SSL加密，然后再启用HTTP/2。</p><p>配置服务器推送特性可以使用指令“http2_push”和“http2_push_preload”：</p><pre><code>http2_push         /style/xxx.css;
http2_push_preload on;
</code></pre><p>不过如何合理地配置推送是个难题，如果推送给浏览器不需要的资源，反而浪费了带宽。</p><p>这方面暂时没有一般性的原则指导，你必须根据自己网站的实际情况去“猜测”客户端最需要的数据。</p><p>优化方面，HTTPS的一些策略依然适用，比如精简密码套件、ECC证书、会话复用、HSTS减少重定向跳转等等。</p><p>但还有一些优化手段在HTTP/2里是不适用的，而且还会有反效果，比如说常见的精灵图（Spriting）、资源内联（inlining）、域名分片（Sharding）等，至于原因是什么，我把它留给你自己去思考（提示，与缓存有关）。</p><p>还要注意一点，HTTP/2默认启用header压缩（HPACK），但并没有默认启用body压缩，所以不要忘了在Nginx配置文件里加上“gzip”指令，压缩HTML、JS等文本数据。</p><h2>应用层协议协商（ALPN）</h2><p>最后说一下HTTP/2的“服务发现”吧。</p><p>你有没有想过，在URI里用的都是HTTPS协议名，没有版本标记，浏览器怎么知道服务器支持HTTP/2呢？为什么上来就能用HTTP/2，而不是用HTTP/1通信呢？</p><p>答案在TLS的扩展里，有一个叫“<strong>ALPN</strong>”（Application Layer Protocol Negotiation）的东西，用来与服务器就TLS上跑的应用协议进行“协商”。</p><p>客户端在发起“Client Hello”握手的时候，后面会带上一个“ALPN”扩展，里面按照优先顺序列出客户端支持的应用协议。</p><p>就像下图这样，最优先的是“h2”，其次是“http/1.1”，以前还有“spdy”，以后还可能会有“h3”。</p><p><img src="https://static001.geekbang.org/resource/image/d8/b0/d8f8606948bbd63c31466e464c1956b0.png?wh=1233*545" alt=""></p><p>服务器看到ALPN扩展以后就可以从列表里选择一种应用协议，在“Server Hello”里也带上“ALPN”扩展，告诉客户端服务器决定使用的是哪一种。因为我们在Nginx配置里使用了HTTP/2协议，所以在这里它选择的就是“h2”。</p><p><img src="https://static001.geekbang.org/resource/image/19/a7/19be1138574589458c96040e1a23b3a7.png?wh=1197*361" alt=""></p><p>这样在TLS握手结束后，客户端和服务器就通过“ALPN”完成了应用层的协议协商，后面就可以使用HTTP/2通信了。</p><h2>小结</h2><p>今天我们讨论了是否应该迁移到HTTP/2，还有应该如何迁移到HTTP/2。</p><ol>
<li><span class="orange">HTTP/2完全兼容HTTP/1，是“更安全的HTTP、更快的HTTPS”，头部压缩、多路复用等技术可以充分利用带宽，降低延迟，从而大幅度提高上网体验；</span></li>
<li><span class="orange">TCP协议存在“队头阻塞”，所以HTTP/2在弱网或者移动网络下的性能表现会不如HTTP/1；</span></li>
<li><span class="orange">迁移到HTTP/2肯定会有性能提升，但高流量网站效果会更显著；</span></li>
<li><span class="orange">如果已经升级到了HTTPS，那么再升级到HTTP/2会很简单；</span></li>
<li><span class="orange">TLS协议提供“ALPN”扩展，让客户端和服务器协商使用的应用层协议，“发现”HTTP/2服务。</span></li>
</ol><h2>课下作业</h2><ol>
<li>和“安全篇”的第29讲类似，结合自己的实际情况，分析一下是否应该迁移到HTTP/2，有没有难点？</li>
<li>精灵图（Spriting）、资源内联（inlining）、域名分片（Sharding）这些手段为什么会对HTTP/2的性能优化造成反效果呢？</li>
</ol><p>欢迎你把自己的学习体会写在留言区，与我和其他同学一起讨论。如果你觉得有所收获，也欢迎把文章分享给你的朋友。</p><p><img src="https://static001.geekbang.org/resource/image/fb/55/fb986a7575ec902c86c17a937dbca655.png?wh=1769*3444" alt="unpreview"></p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/af/e6/9c77acff.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我行我素</span>
  </div>
  <div class="_2_QraFYR_0">2.因为HTTP&#47;2中使用小颗粒化的资源，优化了缓存，而使用精灵图就相当于传输大文件，但是大文件会延迟客户端的处理执行，并且缓存失效的开销很昂贵，很少数量的数据更新就会使整个精灵图失效，需要重新下载(http1中使用精灵图是为了减少请求)；<br>HTTP1中使用内联资源也是为了减少请求，内联资源没有办法独立缓存，破坏了HTTP&#47;2的多路复用和优先级策略；<br>域名分片在HTTP1中是为了突破浏览器每个域名下同时连接数，但是这在HTTP&#47;2中使用多路复用解决了这个问题，如果使用域名分片反而会限制HTTP2的自由发挥</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 回答的非常好。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-12 11:02:56</div>
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
  <div class="_2_QraFYR_0">课下作业的第二题的个人理解。<br><br>问： 精灵图（Spriting）、资源内联（inlining）、域名分片（Sharding）这些手段为什么会对 HTTP&#47;2 的性能优化造成反效果呢？<br><br>答：主要是缓存和请求速度的原因。使用 HTTP&#47;2 后，请求就可以做到乱序收发、多路复用。<br><br>1. 图片就算很多，在 HTTP&#47;2 也可以做到 “并发”。使用了精灵图的话，首先文件变大了，在 HTTP&#47;2 中相比分开请求要更慢，而且不利于缓存（比如修改了其中几个图片）。<br><br>2. 资源内联，是指将一个资源作为另一个资源的一部分，使二者作为一个整体的资源来请求，比如 HTML 文件里嵌入 base64 的图片，该方案是为了减少 HTTP&#47;1 下的请求数，加快网页响应时间。HTTP&#47;2 不存在网页加载变慢的情况，而且不内联的话，能更好地发挥缓存的优势（比如图片是固定的，但 HTML 是动态的）。<br><br>3. 域名分片。这个不是很懂，老师在本文也说到： “HTTP&#47;2 对一个域名只开一个连接，所以一旦这个连接出问题，那么整个网站的体验也就变差了”。这么来说，域名分片建立多个连接，貌似就可以解决这个问题？如果不考虑这个问题的话，因为多路复用的原因，就算多了一个连接，也只是变成了两个多路复用，并没有提高多少效率，倒不是很有必要，而且代码实现也会比较麻烦。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 回答的很好。<br><br>对于第三点，域名分片对于http&#47;2会增加连接成本、HPack字典、慢启动等多个不利因素，所以应该少用。<br><br>最后的综合篇还会再讲一下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-13 00:14:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/77/e5/8b5844df.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>丶景</span>
  </div>
  <div class="_2_QraFYR_0">老师这么理解有错吗？<br>http1 最好把多个请求合成一个请求（比如精灵图，资源内联，域名分片等），原因是 http1 存在 http 的队头阻塞，每次发送新的请求都又需要带上没有压缩的请求头，DNS及时有缓存也需要查找，而且有时候会被清空。http2 某些情况比如精灵图，资源内联就不需要合成一个请求，因为 http2 是基于流和帧的，没有了 http1 的队头阻塞，可以并发多个请求，而且 http2 的请求头使用了 hpage 压缩算法，下一次请求时请求头的长度会非常短（比如 65 一个数字就能表示以前的一长串 UA）。而且还加入了服务器推送，服务器会预先把可能需要的资源先推送给你。如果还是使用一个请求，可能会因为只更新一点点资源而更新整个缓存（比如更新精灵图或资源内联其中的一小部分）。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 基本正确。<br><br>http&#47;1的问题一个是队头阻塞，另一个是报文头数据冗余，多个小请求会浪费带宽，所以资源合并、内联就很有必要。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-13 14:48:28</div>
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
  <div class="_2_QraFYR_0">分析一下是否应该迁移到 HTTP&#47;2，有没有难点？<br>从慢、贵、难三个角度来分析<br>速度快，免费，部署简单，具体分析过程就不写了，我觉得应该立即迁移到 HTTP&#47;2。<br><br>精灵图（Spriting）、资源内联（inlining）、域名分片（Sharding）这些手段为什么会对 HTTP&#47;2 的性能优化造成反效果呢？<br>精灵图、资源内联可以减少HTTP请求数但加大单个请求的大小、域名分片为了突破与同一域名最多建立6个连接的限制，HTTP&#47;2使用流并发请求响应多个资源，完全不需要此优化，相反还会多建立TCP连接浪费资源。<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 迁移到http&#47;2还需要考虑运维、部署等成本，技术上的难度倒是不大。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-12 15:02:01</div>
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
  <div class="_2_QraFYR_0">HTTP&#47;1 里实现了长连接，为啥还会对一个域名开 6~8 个连接，是不是http为了解决http自身的对头阻塞，不要让http等待完响应之后，才发出下一个请求，而打开多个连接？这些打开了的连接会得到重复利用，就是一段时间内不会关闭？这样才会比HTTP（短连接）高效？<br>可以解析一下内联资源，域名分片，什么意思？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.可以参考一下第17讲，里面解释了域名分片。<br><br>2.资源内联，就是把比较小的图片、js等资源编码成base64，以文本的形式嵌入进html，这样通过一次请求就可以下载到更多的数据。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-12 14:37:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/5c/3f/34e5c750.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>看，有只猪</span>
  </div>
  <div class="_2_QraFYR_0">老师，我还有一个问题，既然通过ALPN协商了通信采用的协议，那建立好TLS连接后，后续操作直接采用HTTP2即可，为什么还需要发送Magic呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，这个magic的确是显得有点“多余”，但协议标准里就是这么规定的，只能遵守。<br><br>在rfc里，对这个的说明是对协议的最终确认，相当于是一个简单的校验机制，也可以理解成是正式收发数据前的“握手”，确认是http&#47;2而不是其他的协议。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-28 09:44:24</div>
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
  <div class="_2_QraFYR_0">看了下tmall.com，协议写的是https</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-04 10:41:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/62/10/c1b4770d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>球魁</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，目前我们团队正在尝试升级HTTP&#47;2协议，考虑到原域名下有很多其它团队的接口服务，我们使用一个新的域名来升级HTTP&#47;2，然后把接口逐步迁移过去。最近打算迁移一个流量很大的接口大概2000qps，前期对其做了5%小流量实验，但是结果很不理想，性能并没有提升反而劣化了。跑了一下每个阶段的耗时，发现服务端耗时大概有15-20ms的下降（接口平均耗时900ms），tcp有10ms的下降，ssl也有5-10ms的下降，请问有什么好的优化升级建议嘛？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我是做研发的，对于这种比较复杂的应用场景没有太多实际经验，还是得根据自己的情况去摸索，不太可能给出普适的解决方案，抱歉了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-22 15:13:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJayib1ZcRfOaoLsdsWZokiaO5tLAdC4uNAicQJRIVXrz9fIchib7QwXibnRrsJaoh5TUlia7faUf36g8Bw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>明月</span>
  </div>
  <div class="_2_QraFYR_0">精灵图、资源内联在http2效果不好 终极原因也还是http2只能建立一个连接 这样一个大文件传输会占用这唯一的连接 缓存失效后又要更新占用这个连接更新资源 我理解的对吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不完全对。http&#47;2传输大文件不是问题，问题在于这一个大文件里包含了很多的小文件，缓存失效后重传会浪费带宽，不划算。<br><br>这个与连接没有关系，http&#47;2在一个连接上可以开多个流，不是独占传输数据的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-08 13:02:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/5c/3f/34e5c750.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>看，有只猪</span>
  </div>
  <div class="_2_QraFYR_0">老师，请问我对服务发现的理解对吗？<br>针对加密版本的HTTP&#47;2，客户端需要`Client Hello`的`ALPN`中添加支持的协议，由服务器选择，服务器也通过`ALPN`告知客户端后续通信采用的协议。<br>非加密版本的HTTP&#47;1中，客户端建立TCP连接后，将发送Magic。服务器识别到后，后续通信采用HTTP&#47;2。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 加密版本你理解的是正确的。<br><br>对于非加密版本，http&#47;2要求使用upgrade机制升级到http&#47;2，步骤比alpn要复杂一些，可以参考rfc。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-27 20:57:59</div>
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
  <div class="_2_QraFYR_0">TCP 协议存在“队头阻塞”，所以 HTTP&#47;2 在弱网或者移动网络下的性能表现会不如 HTTP&#47;1；<br>老师，HTTP&#47;1，HTTP&#47;2都是基于TCP协议的，在同样“队头阻塞”的情况下，为什么HTTP&#47;1的性能要优于HTTP&#47;2 ？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: http&#47;1开多个连接，而http&#47;2只有一个连接，弱网下多个连接显然要比一个连接的传输效果好。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-03 10:20:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/cd/77/b2ab5d44.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>👻 小二</span>
  </div>
  <div class="_2_QraFYR_0">有一点我不明白， http2只有一条连接， 那在多线程并发时， 应该需要锁吧， 也就是同一瞬间只能一个写入，多几条连接， 不是能并发吗？<br>还是说网卡层同一瞬间也只能处理一个写的请求， 所以并不并发都没关系。不过在tcp层面上的队头阻塞还是有关系的吧， 多 开一个连接， 就不会都阻塞住了<br><br>如果有多个网卡呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.在http&#47;2协议里没有操作系统线程的概念，应用发送数据需要自己处理socket的并发读写，但数据写入socket后就会由http&#47;2自己分帧管理，实现多路复用。<br><br>2.http&#47;2用一个连接能够实现最佳性能，可以省去连接、慢启动等成本，当然因为tcp层有队头阻塞，所以在网络质量差时会有性能损失。<br><br>3.多个网卡就是多个ip地址，就可以开多个连接，但每个http&#47;2连接就会重建hpack动态表、慢启动，效率会低一点。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-23 16:17:59</div>
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
  <div class="_2_QraFYR_0">为什么在使用http2的时候不可以在一个网站内多开几个tcp连接，这岂不是更高效</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 当然可以，但效果不一定好，因为每个tcp连接都会重建字典，会浪费，而且http&#47;2有了多路复用，传输效率高很多。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-13 23:40:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q3auHgzwzM7JkdLdZXNYZeopVSxeI8ml4MptQMCWI7oIHaJpfYuYjlV9Efic7x19lWickckLQzmTuFlgCVmVicZ9A/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_0e3b40</span>
  </div>
  <div class="_2_QraFYR_0">老师，请教一个关于http&#47;2的问题：<br>如果客户端向服务端同时发起多个请求，那服务端收到请求的顺序会和客户端发起的顺序一样吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 同一个流里的请求是顺序的，不同流是乱序的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-09 21:22:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83erMrXia5kb1AXUJoiccmIQxSQ7ib5SkibsQqd9FZInQcwYeNbZXp7CCtMibtg0RLHoza1NVo8A5M3uIluA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_8476da</span>
  </div>
  <div class="_2_QraFYR_0">学习http到最好<br>好的方法应该是实践！！！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-02 21:03:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/58/95/640b6465.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>fmouse</span>
  </div>
  <div class="_2_QraFYR_0">老师好，对于 HTTP&#47;2 的“服务发现”用 TLS 的扩展里“ALPN”。<br>在 《30｜时代之风（上）：HTTP&#47;2特性概览》提到“在 HTTP&#47;2 标准制定的时候（2015 年）已经发现了很多 SSL&#47;TLS 的弱点，而新的 TLS1.3 还未发布。”<br>TLS1.2+的时候已经支持扩展特性了是吗。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，可以去看rfc。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-22 19:53:23</div>
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
  <div class="_2_QraFYR_0">老师，http2 多开几个连接，岂不是很厉害吗，为啥要限制一个，之前还6-8个呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 因为一个http&amp;#47;2连接的传输效率已经很高了，再多的连接反而会浪费资源，虽然理论上说会增加传输能力，但综合性价比不高。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-25 09:45:39</div>
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
  <div class="_2_QraFYR_0">老师课外小贴士第四条connection：upgrade应该是明文的http 1携带的字段吧  升级到HTTP2之前客户端和服务器不应该是通过HTTP1通信吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，明文的http&#47;2就只能从http&#47;1升级。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-13 19:55:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/24/22/4e/2e081d9c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>hao</span>
  </div>
  <div class="_2_QraFYR_0">精灵图（Spriting）、资源内联（inlining）、域名分片（Sharding）这些手段为什么会对HTTP&#47;2的性能优化造成反效果呢 ？<br><br><br>- 精灵图（Spriting）和资源内联（inlining）减少了请求数但增加了每次请求的报文大小，但  `不利于缓存`（精灵图中某个图标改动就要重新请求整个精灵图，资源内联同理）<br><br>  &gt; 资源内联：内嵌css  、js等资源<br><br><br>  - 对于HTTP&#47;1来说，因为它存在  `队头阻塞`  ，所以将多个小的  `资源合并`  可以有效的缓解这种情况的出现<br>  - 但对于HTTP&#47;2来说，它已经引入了流的概念实现了基于单个TCP连接的多路复用，也就是说解决了  `HTTP的队头阻塞`，它不需要通过资源合并来缓解<br><br>- 域名分片是指利用多个域名和同一个IP地址建立TCP连接，巧妙地避开了浏览器对并发连接数的限制<br><br><br>  - 对于HTTP&#47;1来说，因为它没有  `多路复用`  ，所以这样能很好的缓解因为  `丢包重发`  而导致的  `队头阻塞`  <br>  - 但对于HTTP&#47;2来说，多建立的TCP连接完全是浪费资源（两端的静态表和动态表，TCP连接的成本等）  </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-05 15:43:24</div>
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
  <div class="_2_QraFYR_0">迁移到HTTP&#47;2感觉成本小收益高，好像没有什么理由不迁移一下，除了弱网环境下性能差一些，是否还有什么不利影响呢？或者不想改变也是一种原因，谁知道新东西是否有什么坑呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 协议本身很好，但相关的运维、测试、兼容成本必须要考虑。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-04 22:45:23</div>
  </div>
</div>
</div>
</li>
</ul>