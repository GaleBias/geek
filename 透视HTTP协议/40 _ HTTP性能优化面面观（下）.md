<audio title="40 _ HTTP性能优化面面观（下）" src="https://static001.geekbang.org/resource/audio/0d/73/0d24df4c27787e99c654ac8f28969873.mp3" controls="controls"></audio> 
<p>今天我们继续上次的话题，看看HTTP性能优化有哪些行之有效的手段。</p><p>上一讲里我说到了，在整个HTTP系统里有三个可优化的环节，分别是<strong>服务器</strong>、<strong>客户端</strong>和<strong>传输链路</strong>（“第一公里”和“中间一公里”）。但因为我们是无法完全控制客户端的，所以实际上的优化工作通常是在服务器端。这里又可以细分为后端和前端，后端是指网站的后台服务，而前端就是HTML、CSS、图片等展现在客户端的代码和数据。</p><p>知道了大致的方向，HTTP性能优化具体应该怎么做呢？</p><p>总的来说，任何计算机系统的优化都可以分成这么几类：硬件软件、内部外部、花钱不花钱。</p><p><strong>投资购买现成的硬件</strong>最简单的优化方式，比如换上更强的CPU、更快的网卡、更大的带宽、更多的服务器，效果也会“立竿见影”，直接提升网站的服务能力，也就实现了HTTP优化。</p><p>另外，<strong>花钱购买外部的软件或者服务</strong>也是一种行之有效的优化方式，最“物有所值”的应该算是CDN了（参见<a href="https://time.geekbang.org/column/article/120664">第37讲</a>）。CDN专注于网络内容交付，帮助网站解决“中间一公里”的问题，还有很多其他非常专业的优化功能。把网站交给CDN运营，就好像是“让网站坐上了喷气飞机”，能够直达用户，几乎不需要费什么力气就能够达成很好的优化效果。</p><p>不过这些“花钱”的手段实在是太没有“技术含量”了，属于“懒人”（无贬义）的做法，所以我就不再细说，接下来重点就讲讲在网站内部、“不花钱”的软件优化。</p><!-- [[[read_end]]] --><p>我把这方面的HTTP性能优化概括为三个关键词：<strong>开源</strong>、<strong>节流</strong>、<strong>缓存</strong>。</p><h2>开源</h2><p>这个“开源”可不是Open Source，而是指抓“源头”，开发网站服务器自身的潜力，在现有条件不变的情况下尽量挖掘出更多的服务能力。</p><p>首先，我们应该选用高性能的Web服务器，最佳选择当然就是Nginx/OpenResty了，尽量不要选择基于Java、Python、Ruby的其他服务器，它们用来做后面的业务逻辑服务器更好。利用Nginx强大的反向代理能力实现“动静分离”，动态页面交给Tomcat、Django、Rails，图片、样式表等静态资源交给Nginx。</p><p>Nginx或者OpenResty自身也有很多配置参数可以用来进一步调优，举几个例子，比如说禁用负载均衡锁、增大连接池，绑定CPU等等，相关的资料有很多。</p><p>特别要说的是，对于HTTP协议一定要<strong>启用长连接</strong>。在<a href="https://time.geekbang.org/column/article/126374">第39讲</a>里你也看到了，TCP和SSL建立新连接的成本是非常高的，有可能会占到客户端总延迟的一半以上。长连接虽然不能优化连接握手，但可以把成本“均摊”到多次请求里，这样只有第一次请求会有延迟，之后的请求就不会有连接延迟，总体的延迟也就降低了。</p><p>另外，在现代操作系统上都已经支持TCP的新特性“<strong>TCP Fast Open</strong>”（Win10、iOS9、Linux 4.1），它的效果类似TLS的“False Start”，可以在初次握手的时候就传输数据，也就是0-RTT，所以我们应该尽可能在操作系统和Nginx里开启这个特性，减少外网和内网里的握手延迟。</p><p>下面给出一个简短的Nginx配置示例，启用了长连接等优化参数，实现了动静分离。</p><pre><code>server {
  listen 80 deferred reuseport backlog=4096 fastopen=1024; 


  keepalive_timeout  60;
  keepalive_requests 10000;
  
  location ~* \.(png)$ {
    root /var/images/png/;
  }
  
  location ~* \.(php)$ {
    proxy_pass http://php_back_end;
  }
}
</code></pre><h2>节流</h2><p>“节流”是指减少客户端和服务器之间收发的数据量，在有限的带宽里传输更多的内容。</p><p>“节流”最基本的做法就是使用HTTP协议内置的“数据压缩”编码，不仅可以选择标准的gzip，还可以积极尝试新的压缩算法br，它有更好的压缩效果。</p><p>不过在数据压缩的时候应当注意选择适当的压缩率，不要追求最高压缩比，否则会耗费服务器的计算资源，增加响应时间，降低服务能力，反而会“得不偿失”。</p><p>gzip和br是通用的压缩算法，对于HTTP协议传输的各种格式数据，我们还可以有针对性地采用特殊的压缩方式。</p><p>HTML/CSS/JavaScript属于纯文本，就可以采用特殊的“压缩”，去掉源码里多余的空格、换行、注释等元素。这样“压缩”之后的文本虽然看起来很混乱，对“人类”不友好，但计算机仍然能够毫无障碍地阅读，不影响浏览器上的运行效果。</p><p>图片在HTTP传输里占有非常高的比例，虽然它本身已经被压缩过了，不能被gzip、br处理，但仍然有优化的空间。比如说，去除图片里的拍摄时间、地点、机型等元数据，适当降低分辨率，缩小尺寸。图片的格式也很关键，尽量选择高压缩率的格式，有损格式应该用JPEG，无损格式应该用Webp格式。</p><p>对于小文本或者小图片，还有一种叫做“资源合并”（Concatenation）的优化方式，就是把许多小资源合并成一个大资源，用一个请求全下载到客户端，然后客户端再用JavaScript、CSS切分后使用，好处是节省了请求次数，但缺点是处理比较麻烦。</p><p>刚才说的几种数据压缩针对的都是HTTP报文里的body，在HTTP/1里没有办法可以压缩header，但我们也可以采取一些手段来减少header的大小，不必要的字段就尽量不发（例如Server、X-Powered-By）。</p><p>网站经常会使用Cookie来记录用户的数据，浏览器访问网站时每次都会带上Cookie，冗余度很高。所以应当少使用Cookie，减少Cookie记录的数据量，总使用domain和path属性限定Cookie的作用域，尽可能减少Cookie的传输。如果客户端是现代浏览器，还可以使用HTML5里定义的Web Local Storage，避免使用Cookie。</p><p>压缩之外，“节流”还有两个优化点，就是<strong>域名</strong>和<strong>重定向</strong>。</p><p>DNS解析域名会耗费不少的时间，如果网站拥有多个域名，那么域名解析获取IP地址就是一个不小的成本，所以应当适当“收缩”域名，限制在两三个左右，减少解析完整域名所需的时间，让客户端尽快从系统缓存里获取解析结果。</p><p>重定向引发的客户端延迟也很高，它不仅增加了一次请求往返，还有可能导致新域名的DNS解析，是HTTP前端性能优化的“大忌”。除非必要，应当尽量不使用重定向，或者使用Web服务器的“内部重定向”。</p><h2>缓存</h2><p>在<a href="https://time.geekbang.org/column/article/106804">第20讲</a>里，我就说到了“缓存”，它不仅是HTTP，也是任何计算机系统性能优化的“法宝”，把它和上面的“开源”“节流”搭配起来应用于传输链路，就能够让HTTP的性能再上一个台阶。</p><p>在“第零公里”，也就是网站系统内部，可以使用Memcache、Redis、Varnish等专门的缓存服务，把计算的中间结果和资源存储在内存或者硬盘里，Web服务器首先检查缓存系统，如果有数据就立即返回给客户端，省去了访问后台服务的时间。</p><p>在“中间一公里”，缓存更是性能优化的重要手段，CDN的网络加速功能就是建立在缓存的基础之上的，可以这么说，如果没有缓存，那就没有CDN。</p><p>利用好缓存功能的关键是理解它的工作原理（参见<a href="https://time.geekbang.org/column/article/106804">第20讲</a>和<a href="https://time.geekbang.org/column/article/108313">第22讲</a>），为每个资源都添加ETag和Last-modified字段，再用Cache-Control、Expires设置好缓存控制属性。</p><p>其中最基本的是max-age有效期，标记资源可缓存的时间。对于图片、CSS等静态资源可以设置较长的时间，比如一天或者一个月，对于动态资源，除非是实时性非常高，也可以设置一个较短的时间，比如1秒或者5秒。</p><p>这样一旦资源到达客户端，就会被缓存起来，在有效期内都不会再向服务器发送请求，也就是：“<strong>没有请求的请求，才是最快的请求。</strong>”</p><h2>HTTP/2</h2><p>在“开源”“节流”和“缓存”这三大策略之外，HTTP性能优化还有一个选择，那就是把协议由HTTP/1升级到HTTP/2。</p><p>通过“飞翔篇”的学习，你已经知道了HTTP/2的很多优点，它消除了应用层的队头阻塞，拥有头部压缩、二进制帧、多路复用、流量控制、服务器推送等许多新特性，大幅度提升了HTTP的传输效率。</p><p>实际上这些特性也是在“开源”和“节流”这两点上做文章，但因为这些都已经内置在了协议内，所以只要换上HTTP/2，网站就能够立刻获得显著的性能提升。</p><p>不过你要注意，一些在HTTP/1里的优化手段到了HTTP/2里会有“反效果”。</p><p>对于HTTP/2来说，一个域名使用一个TCP连接才能够获得最佳性能，如果开多个域名，就会浪费带宽和服务器资源，也会降低HTTP/2的效率，所以“域名收缩”在HTTP/2里是必须要做的。</p><p>“资源合并”在HTTP/1里减少了多次请求的成本，但在HTTP/2里因为有头部压缩和多路复用，传输小文件的成本很低，所以合并就失去了意义。而且“资源合并”还有一个缺点，就是降低了缓存的可用性，只要一个小文件更新，整个缓存就完全失效，必须重新下载。</p><p>所以在现在的大带宽和CDN应用场景下，应当尽量少用资源合并（JavaScript、CSS图片合并，数据内嵌），让资源的粒度尽可能地小，才能更好地发挥缓存的作用。</p><h2>小结</h2><ol>
<li><span class="orange">花钱购买硬件、软件或者服务可以直接提升网站的服务能力，其中最有价值的是CDN；</span></li>
<li><span class="orange">不花钱也可以优化HTTP，三个关键词是“开源”“节流”和“缓存”；</span></li>
<li><span class="orange">后端应该选用高性能的Web服务器，开启长连接，提升TCP的传输效率；</span></li>
<li><span class="orange">前端应该启用gzip、br压缩，减小文本、图片的体积，尽量少传不必要的头字段；</span></li>
<li><span class="orange">缓存是无论何时都不能忘记的性能优化利器，应该总使用Etag或Last-modified字段标记资源；</span></li>
<li><span class="orange">升级到HTTP/2能够直接获得许多方面的性能提升，但要留意一些HTTP/1的“反模式”。</span></li>
</ol><p>到这里，专栏的全部课程就学完了，在这三个月的时间里你是否有了很多的收获呢？</p><p>接下来，就请在广阔的网络世界里去实践这些知识吧，祝你成功！</p><p><img src="https://static001.geekbang.org/resource/image/7b/8a/7b2351d7175e815710de646d53d7958a.png?wh=1769*2779" alt="unpreview"></p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/30/8a/b5ca7286.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>业余草</span>
  </div>
  <div class="_2_QraFYR_0">70门课程，看完了50门。后面空闲下来继续看2遍！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 好好学习天天向上。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-03 21:12:33</div>
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
  <div class="_2_QraFYR_0">这是我30门极客时间课程中第一门完整学完的课程，感谢老师通俗易懂的讲解，让我又新学到很多关于HTTP的知识点。跟着这门课程记录笔记，整理完后，整个结构清晰明了。老师按功能讲解字段的方法，让我知道了哪些功能会涉及哪些字段，不会再像以前那么模糊了，期待老师下一门课程</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 期待与你的再次相会。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-03 09:58:04</div>
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
  <div class="_2_QraFYR_0">哈哈，都是夸老师的，在极客时间买了一堆专栏，别的不敢说，判断一个专栏的优劣的能力是给锻炼出来了。和买东西很类似，销量多、评论多（如果引起不了共鸣老实讲会懒得评论）、老师回复多、且比较有耐心（能感受到）这课程绝对值，如果能通俗易懂的讲出来，那功力就更强了，另外，姜还是老的辣，年轻又优秀的也非常多，不过有时能感觉到功力还是不行。说了这么多，最后想说的是老师在佼佼者中也是佼佼者。<br>感谢分享，这钱花的太值了（其实钱不多也不重要，时间和精力才是最宝贵的，如果学了感觉啥用没有就太沮丧了）</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可能是批评的文字没被显示出来吧。<br><br>虽然批评刺耳（人性使然），但也能提醒自己，不然很容易被“捧杀”。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-02 21:31:07</div>
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
  <div class="_2_QraFYR_0">老师写得太好了，学到了很多很多东西。靠着在这篇专栏学到的 HTTP 知识，我在几天前也是拿到了一个大公司的offer，太高兴了。<br>另外这篇专栏也是我第一个从头到尾没有掉队的专栏，写得真的不错！感谢老师带给我们这么棒的专栏！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 能帮到你也是我的荣幸。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-28 23:06:23</div>
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
  <div class="_2_QraFYR_0">老师的课讲的非常好，每节课都看的非常明白，而且每个问题都很认真的回答，想问下老师接下来还打算开什么课吗？个人很希望老师讲讲 tcp ip 有关的。或者还有什么其它渠道能关注到老师吗？比如博客？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.tcp&#47;ip在极客时间上已经有非常好的课程了，可以去看看。<br><br>2.接下来可能要休息一下，也许以后还有机会，感谢你的关注。<br><br>3.博客没有，也没有公众号，个人比较“懒散”，不太愿意在网络上抛头露面，还是在GitHub上交流吧。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-29 18:32:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/db/64/06d54a80.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>中年男子</span>
  </div>
  <div class="_2_QraFYR_0">老师，不太理解 为什么一个网站有多个域名，解析域名获得ip会是不小的成本，需要收缩，是因为访问一个url内部会出现多次访问不同的地址的原因吗？<br>不是web开发，不太理解这些。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 有很多原因，早期域名分片是原因之一，可以让浏览器并发多个连接，加快数据的访问速度。<br><br>另外的原因比如防止域名抢注，增加入口等等，比如Google就注册过多个短域名。<br><br>总之，域名就像是互联网世界的名片，多了总不是坏事。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-28 23:08:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/10/bb/f1061601.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Demon.Lee</span>
  </div>
  <div class="_2_QraFYR_0">c++11、Boost、OpenResty，不过还要看有多少读者想要了<br>---<br>I want! <br>老师尽管出，我一定支持！（老师也要注意休息，身体第一！）<br>感恩！！<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 多谢关心，大家都要多锻炼，健康第一。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-07 09:09:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/7b/10/723a149c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>何某人</span>
  </div>
  <div class="_2_QraFYR_0">吹爆Chrono老师，课程又专业又通俗易懂。让身为网络小白的我受益匪浅。期待老师的新一门课程！！！另外看了老师的课后老师的书我也都买了哈哈。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 支持大感谢。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-29 10:28:09</div>
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
  <div class="_2_QraFYR_0">谢谢老师，我学完了，4周记了70页的笔记。作为一个前端我的http终于入门了。感谢老师的耐心回复让我及时解惑，让我觉得学习的路上有人陪伴，从不孤独。谢谢老师，希望老师能出更多的精品课程，让知识能够改变更多人的生活！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: It&#39;s my pleasure.</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-26 13:04:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_3e1530</span>
  </div>
  <div class="_2_QraFYR_0">老师，赶紧出一个关于nginx和openrestry的课吧</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 极客时间上这两个都有啊，难道有什么特殊需求吗？可以给编辑提，有条件当然可以上。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-14 11:32:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/cd/94/0d44361e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Jerryz</span>
  </div>
  <div class="_2_QraFYR_0">首先，文字+音频的专栏确实比视频的专栏信息密度更高。也更适合学习。<br>再则，作为一个前端工程师学习 HTTP 原理确实和后端谈接口的时候更有信心了。<br>老师的专栏质量很高，也确实做了深入浅出。<br>点个赞。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Many thanks。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-09 08:58:27</div>
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
  <div class="_2_QraFYR_0">完结～ 认真的学完了第一遍～ 第二遍继续～</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 认真的态度值得表扬。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-24 21:22:23</div>
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
  <div class="_2_QraFYR_0">老师，这里指的是多级域名的意思嘛？<br><br>DNS 解析域名会耗费不少的时间，如果网站拥有多个域名，那么域名解析获取 IP 地址就是一个不小的成本，所以应当适当“收缩”域名，限制在两三个左右，减少解析完整域名所需的时间，让客户端尽快从系统缓存里获取解析结果。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不是的，意思是一个网站可能会有多个域名，但这些域名都指向同一个服务器。<br><br>例如Google，有www.google.com，还可能有goog.le等等。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-01 21:03:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/21/cf/f47e092d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>咕咕咕</span>
  </div>
  <div class="_2_QraFYR_0">非常好的专栏，让我对于HTTP知识有了一个系统性的认知，但是后面还需要结合实际经验反复学习加深印象。非常期待老师的下一门专栏！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 说的对，目前这个专题还是比较偏理论一些，要多结合自己的实际情况去实践。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-21 17:47:03</div>
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
  <div class="_2_QraFYR_0">感谢老师的分享，感谢老师的陪伴，学到了很多东西</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 共同进步。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-28 10:09:18</div>
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
  <div class="_2_QraFYR_0">感谢老师，学到了很多知识，长见识了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: my pleasure.</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-08 16:53:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKq0oQVibKcmYJqmpqaNNQibVgia7EsEgW65LZJIpDZBMc7FyMcs7J1JmFCtp06pY8ibbcpW4ibRtG7Frg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>zhoufeng</span>
  </div>
  <div class="_2_QraFYR_0">老师好，<br>课外小贴士里说的第1点，“增大初始拥塞窗口”这个是怎么做的，接收窗口倒是可以通过内核参数调大，包括window_scaling也是针对的接收窗口。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 时间比较久，记不太清了，应该有很多相关的资料，这些都是Linux通用的调优手段。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-27 11:37:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/9a/ab/fd201314.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小耿</span>
  </div>
  <div class="_2_QraFYR_0">感谢老师，学到很多！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: my pleasure.</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-12 00:23:09</div>
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
  <div class="_2_QraFYR_0">我感觉解耦和缓存真的是计算机两个法宝。解耦可以试代码的逻辑性变强，比如计算机网络的分层，前后端的分离。而缓存就是加快速度的一个方法</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 一个是中间层原理，一个是空间换时间。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-31 09:48:23</div>
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
  <div class="_2_QraFYR_0">对于 HTTP&#47;2 来说，一个域名使用一个 TCP 连接才能够获得最佳性能，如果开多个域名，就会浪费带宽和服务器资源，也会降低 HTTP&#47;2 的效率<br>这里说的不太对吧  http1里的并发连接不也是链接到同一个域名吗  域名分片是为了使服务器可以支持更多的连接 所以对同一个ip注册了多个域名 这样服务器总的最大并发连接数就是它们的和</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 建立多个连接会重复慢启动、hpack字典积累等动作，是不必要的资源浪费，http&#47;2有多路复用，一个连接里的效果会更好。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-06 11:52:34</div>
  </div>
</div>
</div>
</li>
</ul>