<audio title="37 _ CDN：加速我们的网络服务" src="https://static001.geekbang.org/resource/audio/44/4f/44e3c0a62c765e9df59d0447ec226a4f.mp3" controls="controls"></audio> 
<p>在正式开讲前，我们先来看看到现在为止HTTP手头都有了哪些“武器”。</p><p>协议方面，HTTPS强化通信链路安全、HTTP/2优化传输效率；应用方面，Nginx/OpenResty提升网站服务能力，WAF抵御网站入侵攻击，讲到这里，你是不是感觉还少了点什么？</p><p>没错，在应用领域，还缺一个在外部加速HTTP协议的服务，这个就是我们今天要说的CDN（Content Delivery Network或Content Distribution Network），中文名叫“内容分发网络”。</p><h2>为什么要有网络加速？</h2><p>你可能要问了，HTTP的传输速度也不算差啊，而且还有更好的HTTP/2，为什么还要再有一个额外的CDN来加速呢？是不是有点“多此一举”呢？</p><p>这里我们就必须要考虑现实中会遇到的问题了。你一定知道，光速是有限的，虽然每秒30万公里，但这只是真空中的上限，在实际的电缆、光缆中的速度会下降到原本的三分之二左右，也就是20万公里/秒，这样一来，地理位置的距离导致的传输延迟就会变得比较明显了。</p><p>比如，北京到广州直线距离大约是2000公里，按照刚才的20万公里/秒来算的话，发送一个请求单程就要10毫秒，往返要20毫秒，即使什么都不干，这个“硬性”的时延也是躲不过的。</p><!-- [[[read_end]]] --><p>另外不要忘了， 互联网从逻辑上看是一张大网，但实际上是由许多小网络组成的，这其中就有小网络“互连互通”的问题，典型的就是各个电信运营商的网络，比如国内的电信、联通、移动三大家。</p><p><img src="https://static001.geekbang.org/resource/image/41/b9/413605355db69278cb137b318b70b3b9.png?wh=1239*746" alt=""></p><p>这些小网络内部的沟通很顺畅，但网络之间却只有很少的联通点。如果你在A网络，而网站在C网络，那么就必须“跨网”传输，和成千上万的其他用户一起去“挤”连接点的“独木桥”。而带宽终究是有限的，能抢到多少只能看你的运气。</p><p>还有，网络中还存在许多的路由器、网关，数据每经过一个节点，都要停顿一下，在二层、三层解析转发，这也会消耗一定的时间，带来延迟。</p><p>把这些因素再放到全球来看，地理距离、运营商网络、路由转发的影响就会成倍增加。想象一下，你在北京，访问旧金山的网站，要跨越半个地球，中间会有多少环节，会增加多少时延？</p><p>最终结果就是，如果仅用现有的HTTP传输方式，大多数网站都会访问速度缓慢、用户体验糟糕。</p><h2>什么是CDN？</h2><p>这个时候CDN就出现了，它就是专门为解决“长距离”上网络访问速度慢而诞生的一种网络应用服务。</p><p>从名字上看，CDN有三个关键词：“<strong>内容</strong>”“<strong>分发</strong>”和“<strong>网络</strong>”。</p><p>先看一下“网络”的含义。CDN的最核心原则是“<strong>就近访问</strong>”，如果用户能够在本地几十公里的距离之内获取到数据，那么时延就基本上变成0了。</p><p>所以CDN投入了大笔资金，在全国、乃至全球的各个大枢纽城市都建立了机房，部署了大量拥有高存储高带宽的节点，构建了一个专用网络。这个网络是跨运营商、跨地域的，虽然内部也划分成多个小网络，但它们之间用高速专有线路连接，是真正的“信息高速公路”，基本上可以认为不存在网络拥堵。</p><p>有了这个高速的专用网之后，CDN就要“分发”源站的“内容”了，用到的就是在<a href="https://time.geekbang.org/column/article/108313">第22讲</a>说过的“<strong>缓存代理</strong>”技术。使用“推”或者“拉”的手段，把源站的内容逐级缓存到网络的每一个节点上。</p><p>于是，用户在上网的时候就不直接访问源站，而是访问离他“最近的”一个CDN节点，术语叫“<strong>边缘节点</strong>”（edge node），其实就是缓存了源站内容的代理服务器，这样一来就省去了“长途跋涉”的时间成本，实现了“网络加速”。</p><p><img src="https://static001.geekbang.org/resource/image/46/5b/46d1dbbb545fcf3cfb53407e0afe9a5b.png?wh=1291*787" alt=""></p><p>那么，CDN都能加速什么样的“内容”呢？</p><p>在CDN领域里，“内容”其实就是HTTP协议里的“资源”，比如超文本、图片、视频、应用程序安装包等等。</p><p>资源按照是否可缓存又分为“<strong>静态资源</strong>”和“<strong>动态资源</strong>”。所谓的“静态资源”是指数据内容“静态不变”，任何时候来访问都是一样的，比如图片、音频。所谓的“动态资源”是指数据内容是“动态变化”的，也就是由后台服务计算生成的，每次访问都不一样，比如商品的库存、微博的粉丝数等。</p><p>很显然，只有静态资源才能够被缓存加速、就近访问，而动态资源只能由源站实时生成，即使缓存了也没有意义。不过，如果动态资源指定了“Cache-Control”，允许缓存短暂的时间，那它在这段时间里也就变成了“静态资源”，可以被CDN缓存加速。</p><p>套用一句广告词来形容CDN吧，我觉得非常恰当：“<strong>我们不生产内容，我们只是内容的搬运工。</strong>”</p><p>CDN，正是把“数据传输”这件看似简单的事情“做大做强”“做专做精”，就像专门的快递公司一样，在互联网世界里实现了它的价值。</p><h2>CDN的负载均衡</h2><p>我们再来看看CDN是具体怎么运行的，它有两个关键组成部分：<strong>全局负载均衡</strong>和<strong>缓存系统</strong>，对应的是DNS（<a href="https://time.geekbang.org/column/article/99665">第6讲</a>）和缓存代理（<a href="https://time.geekbang.org/column/article/107577">第21讲</a>、<a href="https://time.geekbang.org/column/article/108313">第22讲</a>）技术。</p><p>全局负载均衡（Global Sever Load Balance）一般简称为GSLB，它是CDN的“大脑”，主要的职责是当用户接入网络的时候在CDN专网中挑选出一个“最佳”节点提供服务，解决的是用户如何找到“最近的”边缘节点，对整个CDN网络进行“负载均衡”。</p><p><img src="https://static001.geekbang.org/resource/image/6c/ca/6c39e76d58d9f17872c83ae72908faca.png?wh=1281*664" alt=""></p><p>GSLB最常见的实现方式是“<strong>DNS负载均衡</strong>”，这个在<a href="https://time.geekbang.org/column/article/99665">第6讲</a>里也说过，不过GSLB的方式要略微复杂一些。</p><p>原来没有CDN的时候，权威DNS返回的是网站自己服务器的实际IP地址，浏览器收到DNS解析结果后直连网站。</p><p>但加入CDN后就不一样了，权威DNS返回的不是IP地址，而是一个CNAME( Canonical Name )别名记录，指向的就是CDN的GSLB。它有点像是HTTP/2里“Alt-Svc”的意思，告诉外面：“我这里暂时没法给你真正的地址，你去另外一个地方再查查看吧。”</p><p>因为没拿到IP地址，于是本地DNS就会向GSLB再发起请求，这样就进入了CDN的全局负载均衡系统，开始“智能调度”，主要的依据有这么几个：</p><ol>
<li>看用户的IP地址，查表得知地理位置，找相对最近的边缘节点；</li>
<li>看用户所在的运营商网络，找相同网络的边缘节点；</li>
<li>检查边缘节点的负载情况，找负载较轻的节点；</li>
<li>其他，比如节点的“健康状况”、服务能力、带宽、响应时间等。</li>
</ol><p>GSLB把这些因素综合起来，用一个复杂的算法，最后找出一台“最合适”的边缘节点，把这个节点的IP地址返回给用户，用户就可以“就近”访问CDN的缓存代理了。</p><h2>CDN的缓存代理</h2><p>缓存系统是CDN的另一个关键组成部分，相当于CDN的“心脏”。如果缓存系统的服务能力不够，不能很好地满足用户的需求，那GSLB调度算法再优秀也没有用。</p><p>但互联网上的资源是无穷无尽的，不管CDN厂商有多大的实力，也不可能把所有资源都缓存起来。所以，缓存系统只能有选择地缓存那些最常用的那些资源。</p><p>这里就有两个CDN的关键概念：“<strong>命中</strong>”和“<strong>回源</strong>”。</p><p>“命中”就是指用户访问的资源恰好在缓存系统里，可以直接返回给用户；“回源”则正相反，缓存里没有，必须用代理的方式回源站取。</p><p>相应地，也就有了两个衡量CDN服务质量的指标：“<strong>命中率</strong>”和“<strong>回源率</strong>”。命中率就是命中次数与所有访问次数之比，回源率是回源次数与所有访问次数之比。显然，好的CDN应该是命中率越高越好，回源率越低越好。现在的商业CDN命中率都在90%以上，相当于把源站的服务能力放大了10倍以上。</p><p>怎么样才能尽可能地提高命中率、降低回源率呢？</p><p>首先，最基本的方式就是在存储系统上下功夫，硬件用高速CPU、大内存、万兆网卡，再搭配TB级别的硬盘和快速的SSD。软件方面则不断“求新求变”，各种新的存储软件都会拿来尝试，比如Memcache、Redis、Ceph，尽可能地高效利用存储，存下更多的内容。</p><p>其次，缓存系统也可以划分出层次，分成一级缓存节点和二级缓存节点。一级缓存配置高一些，直连源站，二级缓存配置低一些，直连用户。回源的时候二级缓存只找一级缓存，一级缓存没有才回源站，这样最终“扇入度”就缩小了，可以有效地减少真正的回源。</p><p>第三个就是使用高性能的缓存服务，据我所知，目前国内的CDN厂商内部都是基于开源软件定制的。最常用的是专门的缓存代理软件Squid、Varnish，还有新兴的ATS（Apache Traffic Server），而Nginx和OpenResty作为Web服务器领域的“多面手”，凭借着强大的反向代理能力和模块化、易于扩展的优点，也在CDN里占据了不少的份额。</p><h2>小结</h2><p>CDN发展到现在已经有二十来年的历史了，早期的CDN功能比较简单，只能加速静态资源。随着这些年Web 2.0、HTTPS、视频、直播等新技术、新业务的崛起，它也在不断进步，增加了很多的新功能，比如SSL加速、内容优化（数据压缩、图片格式转换、视频转码）、资源防盗链、WAF安全防护等等。</p><p>现在，再说CDN是“搬运工”已经不太准确了，它更像是一个“无微不至”的“网站保姆”，让网站只安心生产优质的内容，其他的“杂事”都由它去代劳。</p><ol>
<li>由于客观地理距离的存在，直连网站访问速度会很慢，所以就出现了CDN；</li>
<li>CDN构建了全国、全球级别的专网，让用户就近访问专网里的边缘节点，降低了传输延迟，实现了网站加速；</li>
<li>GSLB是CDN的“大脑”，使用DNS负载均衡技术，智能调度边缘节点提供服务；</li>
<li>缓存系统是CDN的“心脏”，使用HTTP缓存代理技术，缓存命中就返回给用户，否则就要回源。</li>
</ol><h2>课下作业</h2><ol>
<li>网站也可以自建同城、异地多处机房，构建集群来提高服务能力，为什么非要选择CDN呢？</li>
<li>对于无法缓存的动态资源，你觉得CDN也能有加速效果吗？</li>
</ol><p>欢迎你把自己的学习体会写在留言区，与我和其他同学一起讨论。如果你觉得有所收获，也欢迎把文章分享给你的朋友。</p><p><img src="https://static001.geekbang.org/resource/image/bc/51/bc1805a7c49977c7838b29602f3bba51.png?wh=1769*3498" alt="unpreview"></p><p></p>
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
  <div class="_2_QraFYR_0">1.自建成本太高，一般的公司玩不起<br>2.cache-control允许缓存的动态资源可以被CDN缓存。不允许缓存的动态资源会回源，虽然老师课上没讲，感觉回源的路径会被优化。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.对<br><br>2.cdn一般有专用的高速网络直连源站，或者是动态路径优化，所以动态资源回源要比通过公网速度快很多。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-21 07:08:54</div>
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
  <div class="_2_QraFYR_0">如果请求的是动态资源，走 CDN 貌似会更慢（因为无法缓存，CDN到源站多了握手过程）？<br><br>我想到两个方案：<br>（1）静态资源都放到一个域名里，然后这个域名使用 CDN 缓存加速。动态资源则使用另一个域名，不进行 CDN 缓存，直接访问源站。<br>（2）让动态资源也走 CDN，像老师说的那样， CDN 有高速网络直连源站，速度也能提高，但这里貌似也是会有不少带宽支出。<br><br>我想知道实际中，请求动态资源具体到底是怎么考虑，一般使用什么方案？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 现在cdn有种叫“边缘计算”的技术，就是把计算动态资源的代码和数据也放在cdn的节点上，这样就可以在cdn里获取动态资源不用回源站了。<br><br>cdn与源站通常都有专网连接，所以走cdn也比走公网快。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-23 01:17:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/11/e7/044a9a6c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>book尾汁</span>
  </div>
  <div class="_2_QraFYR_0">1 是的自建成本很高<br>2 虽然是动态资源会回源，但是用户通过GSLB得到的边缘节点的ip也是离用户较近的，边缘节点到中心 中心节点到源站的网络也是很快的，综合来说还是要比用户直接访问快点。支持动态加速的CDN就更快了，直接规划出一条到源站的最优路径<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 说的很好。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-04 21:53:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/1a/66/2d9db9ed.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>苦行僧</span>
  </div>
  <div class="_2_QraFYR_0">我们公司就是生产大量视频 每天几T视频 不用cdn成本可想而知 不过每次招标换厂家都涉及视频搬迁</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 在视频、直播领域cdn非常有用。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-28 08:36:36</div>
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
  <div class="_2_QraFYR_0">CDN不仅有专线，还可以进行边缘计算。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-22 00:25:44</div>
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
  <div class="_2_QraFYR_0">网站也可以自建同城、异地多处机房，构建集群来提高服务能力，为什么非要选择 CDN 呢？<br>确实可以，不过建大了，就成了一个CDN。<br><br>对于无法缓存的动态资源，你觉得 CDN 也能有加速效果吗？<br>也有加速效果，不过要注意设置好缓存过期时间，不宜太长。如果不设缓存，内容由CDN回源去源站拉取，链路上专用网络可能快一些。对于缓存不可接受的业务，最好就不要用CDN了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-21 12:00:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/d9/78/8a328299.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>佳佳大魔王</span>
  </div>
  <div class="_2_QraFYR_0">问题一，我觉得会造成数据同步困难的问题，另外在网站更新的时候，消耗的资源也比较多</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对，自己搭建机房费时费力费钱费人工，对于一般的中小型网站来说性价比不高，cdn是专门做这个的，交给他们做效果会更好。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-21 07:55:44</div>
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
  <div class="_2_QraFYR_0">1：网站也可以自建同城、异地多处机房，构建集群来提高服务能力，为什么非要选择 CDN 呢？<br>钱决定的，成本收益比导致选择CDN，假如自建更省钱那一定会选择自建。首先，小公司自建不起，没有这么多人力、财力、技术，用多少买多少多香。大公司觉得自建更好，此时就变成了使用自己的CDN。<br><br>2：对于无法缓存的动态资源，你觉得 CDN 也能有加速效果吗？<br>无法缓存的资源，只能回源获取，正好能够利用CDN与源服务器之间的高速通道（看评论才知道），也能起到加速的效果吧！<br>话说什么动态资源不能够缓存？自建机房，异地多机房不是什么都有了嘛？只是又引入了一致性问题。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 回答的很好。<br><br>动态资源的关键在于它的实时性，如果缓存可能会导致数据不一致，比如账户余额。<br><br>如果实时性要求低，那么就可以缓存较短的时间。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-05 12:33:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/67/8a/babd74dc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>锦</span>
  </div>
  <div class="_2_QraFYR_0">现实中为了减少回源率，降低宽带费用，一般cdn厂商也会提供伪源，那云厂商是怎么解决这个回源问题的呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你说的这个“伪源”可能就是cdn的一级节点吧，二级节点不直接访问源站，而是访问cdn内部的一级节点，而一级节点就充当了源站。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-21 09:36:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/57/6e/b6795c44.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>夜空中最亮的星</span>
  </div>
  <div class="_2_QraFYR_0">老师，我又回来啦，😄</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: welcome back。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-21 09:10:02</div>
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
  <div class="_2_QraFYR_0">1.网站也可以自建同城、异地多处机房，构建集群来提高服务能力，为什么非要选择 CDN 呢？<br>因为CDN有大量边缘节点，网站只需要专注于自身业务，无需关心早专业的CDN复杂调度逻辑。<br>2.对于无法缓存的动态资源，你觉得 CDN 也能有加速效果吗？<br>有一定的效果，因为动态资源可以走cdn专线回源。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: very good。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-21 08:28:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/75/aa/21275b9d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>闫飞</span>
  </div>
  <div class="_2_QraFYR_0">TB级别的硬盘看起来也不是特别高端厉害的样子，因为普通的PC机也标配TB级SATA盘或者512GB SSD了吧。我猜CDN厂商是不是需要用专有的服务器硬盘保证比较长的服务寿命，还是大家就到市面上买买普通的硬盘挂上去就用了?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 家用级别和企业级别的硬盘还是有区别的，用于服务器的硬盘质量肯定会更高一些。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-21 08:08:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/d9/78/8a328299.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>佳佳大魔王</span>
  </div>
  <div class="_2_QraFYR_0">问题二，文章指出了如果动态资源指定了 Cache-Control  那么也可以在很短时间内缓存<br>我觉得在这种情况比较有用：打开一个网站，打到一半的时候关闭，此时服务器已经开始了运算，当我们在很短时间内再次打开相同的网站时，很快就进入了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，这些数据已经下载到了本地，而且依然在有效期，那么就不用再次向服务器请求，可以直接在本地获取。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-21 07:59:26</div>
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
  <div class="_2_QraFYR_0">学习了，干货满满</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-07 10:44:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/f2/35/c3589990.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>北冥有鱼</span>
  </div>
  <div class="_2_QraFYR_0">1.自建多地缓存集群，成本巨高，只有少数top级互联网公司承担的起。<br>2.CDN依托于自己搭建的网络，通过计算最优路由来实现加速效果。另外还能优化跨运营商访问，省去了源站的BGP带宽开销</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-25 00:09:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/23/52/66/3e4d4846.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>includestdio.h</span>
  </div>
  <div class="_2_QraFYR_0">1.成本高，维护困难；<br><br>2.数据一致性要求不高，同步慢等，可以设置短时间缓存</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-28 10:34:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>牛</span>
  </div>
  <div class="_2_QraFYR_0">CDN已经是一项互联网领域的基本服务了。对于受众分布广泛，比如全国性、全世界性的互联网产品，都用得上，而且费用还不便宜。那些个视频网站，这项开支巨大。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对，CDN已经是如今互联网的基础设施了，无法想象没有它的世界会是什么样。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-02 21:14:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/c1/60/fc3689d0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小谢同学</span>
  </div>
  <div class="_2_QraFYR_0">问题1:主要是互联网形态的业务终端用户分布范围太广，还需要考虑一些边远地区甚至三大运营商覆盖不到的地方，这时候考验cdn的节点和运营商覆盖了，此外尽可能优化终端体验也是产品的一大竞争优势<br>问题2:对于动态内容可以采用动态加速或者所谓的“全站加速”，即通过cdn服务商提供动态内容请求的最优回源路线</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-08 19:47:32</div>
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
  <div class="_2_QraFYR_0">1、为了省钱<br>2、线路优化</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 言简意赅，good。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-04 08:06:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/df/43/1aa8708a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>子杨</span>
  </div>
  <div class="_2_QraFYR_0">老师好，对于 CDN 而言，如何判断是源站造成的异常还是 CDN 的异常？为了监控源站是否异常，我们目前采用的是在源站放一个文件，不断的去探测文件的长度，一致的话就表示正常。总觉得会有更好的办法。。。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个好像是最基本的，也挺有效，这样就保证源站的http服务是正常的，其他很多网站都是这么干，像let&#39;s encrypt。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-23 17:39:18</div>
  </div>
</div>
</div>
</li>
</ul>