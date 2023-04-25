<audio title="第20讲 _ CDN：你去小卖部取过快递么？" src="https://static001.geekbang.org/resource/audio/8d/a0/8dafced847a138ec2914b8c5f45b5ea0.mp3" controls="controls"></audio> 
<p>上一节，我们看到了网站的一般访问模式。</p><p>当一个用户想访问一个网站的时候，指定这个网站的域名，DNS就会将这个域名解析为地址，然后用户请求这个地址，返回一个网页。就像你要买个东西，首先要查找商店的位置，然后去商店里面找到自己想要的东西，最后拿着东西回家。</p><p><strong>那这里面还有没有可以优化的地方呢？</strong></p><p>例如你去电商网站下单买个东西，这个东西一定要从电商总部的中心仓库送过来吗？原来基本是这样的，每一单都是单独配送，所以你可能要很久才能收到你的宝贝。但是后来电商网站的物流系统学聪明了，他们在全国各地建立了很多仓库，而不是只有总部的中心仓库才可以发货。</p><p>电商网站根据统计大概知道，北京、上海、广州、深圳、杭州等地，每天能够卖出去多少书籍、卫生纸、包、电器等存放期比较长的物品。这些物品用不着从中心仓库发出，所以平时就可以将它们分布在各地仓库里，客户一下单，就近的仓库发出，第二天就可以收到了。</p><p>这样，用户体验大大提高。当然，这里面也有个难点就是，生鲜这类东西保质期太短，如果提前都备好货，但是没有人下单，那肯定就坏了。这个问题，我后文再说。</p><p><strong>我们先说，我们的网站访问可以借鉴“就近配送”这个思路。</strong></p><p>全球有这么多的数据中心，无论在哪里上网，临近不远的地方基本上都有数据中心。是不是可以在这些数据中心里部署几台机器，形成一个缓存的集群来缓存部分数据，那么用户访问数据的时候，就可以就近访问了呢？</p><!-- [[[read_end]]] --><p>当然是可以的。这些分布在各个地方的各个数据中心的节点，就称为<strong>边缘节点</strong>。</p><p>由于边缘节点数目比较多，但是每个集群规模比较小，不可能缓存下来所有东西，因而可能无法命中，这样就会在边缘节点之上。有区域节点，规模就要更大，缓存的数据会更多，命中的概率也就更大。在区域节点之上是中心节点，规模更大，缓存数据更多。如果还不命中，就只好回源网站访问了。</p><p><img src="https://static001.geekbang.org/resource/image/5f/25/5fbe602d9b85966d9a1748d2e6aa6425.jpeg?wh=1920*1080" alt=""></p><p>这就是<strong>CDN分发系统的架构</strong>。CDN系统的缓存，也是一层一层的，能不访问后端真正的源，就不打扰它。这也是电商网站物流系统的思路，北京局找不到，找华北局，华北局找不到，再找北方局。</p><p>有了这个分发系统之后，接下来，<strong>客户端如何找到相应的边缘节点进行访问呢？</strong></p><p>还记得我们讲过的基于DNS的全局负载均衡吗？这个负载均衡主要用来选择一个就近的同样运营商的服务器进行访问。你会发现，CDN分发网络也是一个分布在多个区域、多个运营商的分布式系统，也可以用相同的思路选择最合适的边缘节点。</p><p><img src="https://static001.geekbang.org/resource/image/c4/24/c4d826188e664605d6f8dfb82e348824.jpeg?wh=1920*1080" alt=""></p><p><strong>在没有CDN的情况下</strong>，用户向浏览器输入www.web.com这个域名，客户端访问本地DNS服务器的时候，如果本地DNS服务器有缓存，则返回网站的地址；如果没有，递归查询到网站的权威DNS服务器，这个权威DNS服务器是负责web.com的，它会返回网站的IP地址。本地DNS服务器缓存下IP地址，将IP地址返回，然后客户端直接访问这个IP地址，就访问到了这个网站。</p><p>然而<strong>有了CDN之后，情况发生了变化</strong>。在web.com这个权威DNS服务器上，会设置一个CNAME别名，指向另外一个域名 www.web.cdn.com，返回给本地DNS服务器。</p><p>当本地DNS服务器拿到这个新的域名时，需要继续解析这个新的域名。这个时候，再访问的就不是web.com的权威DNS服务器了，而是web.cdn.com的权威DNS服务器，这是CDN自己的权威DNS服务器。在这个服务器上，还是会设置一个CNAME，指向另外一个域名，也即CDN网络的全局负载均衡器。</p><p>接下来，本地DNS服务器去请求CDN的全局负载均衡器解析域名，全局负载均衡器会为用户选择一台合适的缓存服务器提供服务，选择的依据包括：</p><ul>
<li>
<p>根据用户IP地址，判断哪一台服务器距用户最近；</p>
</li>
<li>
<p>用户所处的运营商；</p>
</li>
<li>
<p>根据用户所请求的URL中携带的内容名称，判断哪一台服务器上有用户所需的内容；</p>
</li>
<li>
<p>查询各个服务器当前的负载情况，判断哪一台服务器尚有服务能力。</p>
</li>
</ul><p>基于以上这些条件，进行综合分析之后，全局负载均衡器会返回一台缓存服务器的IP地址。</p><p>本地DNS服务器缓存这个IP地址，然后将IP返回给客户端，客户端去访问这个边缘节点，下载资源。缓存服务器响应用户请求，将用户所需内容传送到用户终端。如果这台缓存服务器上并没有用户想要的内容，那么这台服务器就要向它的上一级缓存服务器请求内容，直至追溯到网站的源服务器将内容拉到本地。</p><p><strong>CDN可以进行缓存的内容有很多种。</strong></p><p>保质期长的日用品比较容易缓存，因为不容易过期，对应到就像电商仓库系统里，就是静态页面、图片等，因为这些东西也不怎么变，所以适合缓存。</p><p><img src="https://static001.geekbang.org/resource/image/ca/1d/caec3ba1086557cbf694c621e7e01e1d.jpeg?wh=1254*1080" alt=""></p><p>还记得这个<strong>接入层缓存的架构</strong>吗？在进入数据中心的时候，我们希望通过最外层接入层的缓存，将大部分静态资源的访问拦在边缘。而CDN则更进一步，将这些静态资源缓存到离用户更近的数据中心。越接近客户，访问性能越好，时延越低。</p><p>但是静态内容中，有一种特殊的内容，也大量使用了CDN，这个就是前面讲过的<a href="https://time.geekbang.org/column/article/9688">流媒体</a>。</p><p>CDN支持<strong>流媒体协议</strong>，例如前面讲过的RTMP协议。在很多情况下，这相当于一个代理，从上一级缓存读取内容，转发给用户。由于流媒体往往是连续的，因而可以进行预先缓存的策略，也可以预先推送到用户的客户端。</p><p>对于静态页面来讲，内容的分发往往采取<strong>拉取</strong>的方式，也即当发现未命中的时候，再去上一级进行拉取。但是，流媒体数据量大，如果出现<strong>回源</strong>，压力会比较大，所以往往采取主动<strong>推送</strong>的模式，将热点数据主动推送到边缘节点。</p><p>对于流媒体来讲，很多CDN还提供<strong>预处理服务</strong>，也即文件在分发之前，经过一定的处理。例如将视频转换为不同的码流，以适应不同的网络带宽的用户需求；再如对视频进行分片，降低存储压力，也使得客户端可以选择使用不同的码率加载不同的分片。这就是我们常见的，“我要看超清、标清、流畅等”。</p><p>对于流媒体CDN来讲，有个关键的问题是<strong>防盗链</strong>问题。因为视频是要花大价钱买版权的，为了挣点钱，收点广告费，如果流媒体被其他的网站盗走，在人家的网站播放，那损失可就大了。</p><p>最常用也最简单的方法就是<strong>HTTP头的referer字段</strong>， 当浏览器发送请求的时候，一般会带上referer，告诉服务器是从哪个页面链接过来的，服务器基于此可以获得一些信息用于处理。如果refer信息不是来自本站，就阻止访问或者跳到其它链接。</p><p><strong>referer的机制相对比较容易破解，所以还需要配合其他的机制。</strong></p><p>一种常用的机制是<strong>时间戳防盗链</strong>。使用CDN的管理员可以在配置界面上，和CDN厂商约定一个加密字符串。</p><p>客户端取出当前的时间戳，要访问的资源及其路径，连同加密字符串进行签名算法得到一个字符串，然后生成一个下载链接，带上这个签名字符串和截止时间戳去访问CDN。</p><p>在CDN服务端，根据取出过期时间，和当前 CDN 节点时间进行比较，确认请求是否过期。然后CDN服务端有了资源及路径，时间戳，以及约定的加密字符串，根据相同的签名算法计算签名，如果匹配则一致，访问合法，才会将资源返回给客户。</p><p>然而比如在电商仓库中，我在前面提过，有关生鲜的缓存就是非常麻烦的事情，这对应着就是动态的数据，比较难以缓存。怎么办呢？现在也有<strong>动态CDN，主要有两种模式</strong>。</p><ul>
<li>
<p>一种为<strong>生鲜超市模式</strong>，也即<strong>边缘计算的模式</strong>。既然数据是动态生成的，所以数据的逻辑计算和存储，也相应的放在边缘的节点。其中定时从源数据那里同步存储的数据，然后在边缘进行计算得到结果。就像对生鲜的烹饪是动态的，没办法事先做好缓存，因而将生鲜超市放在你家旁边，既能够送货上门，也能够现场烹饪，也是边缘计算的一种体现。</p>
</li>
<li>
<p>另一种是<strong>冷链运输模式</strong>，也即<strong>路径优化的模式</strong>。数据不是在边缘计算生成的，而是在源站生成的，但是数据的下发则可以通过CDN的网络，对路径进行优化。因为CDN节点较多，能够找到离源站很近的边缘节点，也能找到离用户很近的边缘节点。中间的链路完全由CDN来规划，选择一个更加可靠的路径，使用类似专线的方式进行访问。</p>
</li>
</ul><p>对于常用的TCP连接，在公网上传输的时候经常会丢数据，导致TCP的窗口始终很小，发送速度上不去。根据前面的TCP流量控制和拥塞控制的原理，在CDN加速网络中可以调整TCP的参数，使得TCP可以更加激进地传输数据。</p><p>可以通过多个请求复用一个连接，保证每次动态请求到达时。连接都已经建立了，不必临时三次握手或者建立过多的连接，增加服务器的压力。另外，可以通过对传输数据进行压缩，增加传输效率。</p><p>所有这些手段就像冷链运输，整个物流优化了，全程冷冻高速运输。不管生鲜是从你旁边的超市送到你家的，还是从产地送的，保证到你家是新鲜的。</p><h2>小结</h2><p>好了，这节就到这里了。咱们来总结一下，你记住这两个重点就好。</p><ul>
<li>
<p>CDN和电商系统的分布式仓储系统一样，分为中心节点、区域节点、边缘节点，而数据缓存在离用户最近的位置。</p>
</li>
<li>
<p>CDN最擅长的是缓存静态数据，除此之外还可以缓存流媒体数据，这时候要注意使用防盗链。它也支持动态数据的缓存，一种是边缘计算的生鲜超市模式，另一种是链路优化的冷链运输模式。</p>
</li>
</ul><p>最后，给你留两个思考题：</p><ol>
<li>
<p>这一节讲了CDN使用DNS进行全局负载均衡的例子，CDN如何使用HttpDNS呢？</p>
</li>
<li>
<p>客户端对DNS、HttpDNS、CDN访问了半天，还没进数据中心，你知道数据中心里面什么样吗？</p>
</li>
</ol><p>我们的专栏更新到第20讲，不知你掌握得如何？每节课后我留的思考题，你都有没有认真思考，并在留言区写下答案呢？我会从<strong>已发布的文章中选出一批认真留言的同学</strong>，赠送<span class="orange">学习奖励礼券</span>和我整理的<span class="orange">独家网络协议知识图谱</span>。</p><p>欢迎你留言和我讨论。趣谈网络协议，我们下期见！</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/3b/36/2d61e080.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>行者</span>
  </div>
  <div class="_2_QraFYR_0">1. 参照阿里云CDN HTTPDNS方式；客户端请求服务URL:umc.danuoyi.alicdn.com xxx，参数是客户端ip地址和待解析的域名；然后返回多个ip地址，客户端轮训这些ip地址。<br>2. 如果把边缘节点比作小卖部，那数据中心就是超级市场，里面商品应有尽有，是所有子节点的父集。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-07-03 21:09:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/aa/21/6c3ba9af.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lfn</span>
  </div>
  <div class="_2_QraFYR_0">搜了一圈儿，没找到cdn权威指南这本书，您是不是写错了？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: cdn技术详解</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-20 11:42:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>米兰的小铁匠</span>
  </div>
  <div class="_2_QraFYR_0">我的理解是 CDN 只是节点，网络传输只能走公网的线路。文章中说走 dns 专用线路，难道是像运营商一样自建电缆？希望老师解惑。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-07-08 01:43:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/72/67/aa52812a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>stark</span>
  </div>
  <div class="_2_QraFYR_0">我有个疑问，CDN是内容分发系统，之前在NGINX的学习中，那个老师说主要是为了静态资源，我有点不理解，超哥能稍微解释一下么</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: nginx是在网站的接入层缓存静态资源，CDN是在数据中心之外，离客户端很近的地方缓存静态资源</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-12 20:02:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/d6/12/f799997f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ZACK</span>
  </div>
  <div class="_2_QraFYR_0">由于多dc，静态资源图片同步是个很大的问题，因为网速，后来我们尝试用cdn去解决，但被公司一自称很牛的哥们否掉，原因是cdn只支持外网的，至今没有完全理解是否正确</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 理解正确，cdn其他厂商提供的服务，是指在外网工作</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-15 14:16:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/70/49/d7690979.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>tommyCmd</span>
  </div>
  <div class="_2_QraFYR_0">根据ip地址，怎么判断服务器的远近呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-03 13:35:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/66/34/0508d9e4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>u</span>
  </div>
  <div class="_2_QraFYR_0">专栏前15讲偏基础，后面的偏架构，后面的内容虽然工作中不一定接触的到，但里面的架构思想值得总结学习！超哥厉害！👍🏻👍🏻👍🏻</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-07-04 09:42:46</div>
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
  <div class="_2_QraFYR_0">老师，DNS的作用相当于客户端找到最近最合适DNS运营商，而CDN相当于DNS运营商找到最合适的服务器获取内容。可以这样理解吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-06 09:17:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/wYPdGBibR1FJWWMzFzYBy4tNTd22rMgtlYTgR7wuicFLQjiaozuRWM2VSMFxPYjVdkLJLGvMav2icH3icgz07hmDsDw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>dayspring</span>
  </div>
  <div class="_2_QraFYR_0">超哥，请问一下cdn的权威dns服务器为什么还需要配置cname指向全局负载均衡器的域名呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-24 09:01:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/fc/d7/b102034a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>do it</span>
  </div>
  <div class="_2_QraFYR_0">个人小结<br>CDN(content delivery network)内容分发网络：分为边缘节点、区域节点、中心节点，数据缓存在离用户最近的位置。<br>CDN最擅长的是缓存静态数据，还可以缓存流媒体数据，这时需要注意使用防盗链(refer机制，时间戳防盗链)。也支持动态数据的缓存，一种是边缘计算的生鲜超市模式，另一种是链路优化的冷链运输模式。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-13 08:21:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/22/e8/8e154de6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>圆嗝嗝</span>
  </div>
  <div class="_2_QraFYR_0">客户端发送一个HTTP请求给HTTPDNS，HTTPDNS返回含有请求内容的CDN服务器的IP地址。<br>与基于DNS进行负载均衡相比，少了DNS先递归查询CDN负载均衡服务器IP地址再通过CDN负载均衡服务器获取CDN服务器IP地址两个步骤。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-01 20:39:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/58/2c/92c7ce3b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>易轻尘</span>
  </div>
  <div class="_2_QraFYR_0">貌似cdn要访问的节点比直接的dns更多啊，要是内容命中率不高的话反而得不偿失。所以cdn的边缘节点是有像cache那样的缓存机制吧，最近最少访问之类的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-07-03 12:11:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/ac/a1/43d83698.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>云学</span>
  </div>
  <div class="_2_QraFYR_0">数据中心里面是一排排大机柜</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-07-03 06:54:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/80/ac/37afe559.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小毅(Eric)</span>
  </div>
  <div class="_2_QraFYR_0">听了专栏20集,每天上班中午休息的时候听一级然后做记录. 感觉将的听上去明白,但是其实不明白.由于在工作中其实并没有碰到那么多关于网络优化的实例(仅仅做后开开发的工作),所以有点一知半解.感觉似懂非懂.不知道后面如何继续?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 网上查查资料，尤其是公有云，一般都提供这种服务的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-02 13:11:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/09/42/1f762b72.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Hurt</span>
  </div>
  <div class="_2_QraFYR_0">认真的思考了 但又的确实是自己底子薄 不过会反复的学习</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-07-02 19:58:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_f0bd6b</span>
  </div>
  <div class="_2_QraFYR_0">冷链运输模式，离源近的节点和离客户近的节点之间的链路是怎么实现优化成“专线”的，如果还是承载在公网上，那他们之间的路径本身不就应该是最优的了吗？请教老师解惑，谢谢！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-27 08:08:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/ef/f1/8b06801a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>哇哈哈</span>
  </div>
  <div class="_2_QraFYR_0">其实我还是没有理解 cdn 和 dns 的区别，dns 能帮找到最近的服务器，在这些最近的服务器上做静态资源存储不就可以吗？为什么还要分发到 cdn？我个人的理解是，打个比方，可能web.com 的服务器因为资源问题不会部署到每个城市，例如只在一些大区域部署，例如华东部署一个集群华南区域部署一个集群，但是 cdn 因为基本只有存储逻辑，资源开销比服务集群小，所以可以铺开部署到下一层深入到城市级别，例如上海，广州，这样 cdn 比 web.com 的集群更接近用户，所以这就是 cdn 能存在的意义？<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-23 11:59:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/21/b8/aca814dd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>江山如画</span>
  </div>
  <div class="_2_QraFYR_0">今日学习笔记：<br><br>什么是CDN?<br><br>CDN 内容分发网络，content delivery network，用于加快客户端的数据访问速度。<br><br><br>CDN如何加快数据访问速度?<br><br>边缘节点-&gt;区域节点-&gt;中心节点，类似于多级Cache的设计，CDN 负载均衡器会根据 用户IP，所在运营商，访问内容，当前服务器负载情况 等因素选择一台合适边缘节点的地址供客户端进行访问。<br><br><br>流媒体CDN的特点？<br><br>a. 音视频流数据量比较大，一般会将热点数据从中心节点主动推送到边缘节点进行缓存，以减少被动回源对中心服务器造成的压力。<br>b. 流媒体CDN支持对音视频流进行预处理，比如视频的标清，高清，蓝光，视频会被处理为不同码率，以便适应不同带宽，或者对视频进行分片处理，加载时只加载用户当前访问分片以加快访问速度。<br>c. 流媒体CDN需要注意防盗链，一般在http报头中添加 referer 字段指明访问来自哪里，在通信时服务器端需要根据加解密算法验证访问方是否可靠，也需要验证访问是否已过期。<br><br><br>CDN 如何加快动态数据的访问？<br><br>a. 将数据计算放在边缘节点，边缘节点定期从中心节点同步数据<br>b. 计算仍放在中心节点，但是访问路径进行优化</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-14 00:34:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/88/cd/2c3808ce.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Yangjing</span>
  </div>
  <div class="_2_QraFYR_0">那 CDN 是怎么样把数据推送到到各个节点的呢？ 运营商给的规则吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，缓存规则</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-26 23:20:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/1d/8e/0a546871.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>凡凡</span>
  </div>
  <div class="_2_QraFYR_0">1.需要客户端增加httpdns的sdk包，通过一次http请求完成最近节点的获取，然后请求cdn节点获取数据。相较于dns方式，少了一次cname域名的获取过程，少了dns域名的解析过程，而且能避免dns的偶尔失效问题。不过，现在主流的cdn似是采用的dns方式，比如阿里云和七牛云。2.到了服务地址之后，一般会有防火墙，nat，负载均衡器，网关，rpc等参与请求处理。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-07-03 21:28:05</div>
  </div>
</div>
</div>
</li>
</ul>