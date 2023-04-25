<audio title="第18讲 _ DNS协议：网络世界的地址簿" src="https://static001.geekbang.org/resource/audio/c5/34/c56a67b3e6e791b6677d9d327b3fd834.mp3" controls="controls"></audio> 
<p>前面我们讲了平时常见的看新闻、支付、直播、下载等场景，现在网站的数目非常多，常用的网站就有二三十个，如果全部用IP地址进行访问，恐怕很难记住。于是，就需要一个地址簿，根据名称，就可以查看具体的地址。</p><p>例如，我要去西湖边的“外婆家”，这就是名称，然后通过地址簿，查看到底是哪条路多少号。</p><h2>DNS服务器</h2><p>在网络世界，也是这样的。你肯定记得住网站的名称，但是很难记住网站的IP地址，因而也需要一个地址簿，就是<strong>DNS服务器</strong>。</p><p>由此可见，DNS在日常生活中多么重要。每个人上网，都需要访问它，但是同时，这对它来讲也是非常大的挑战。一旦它出了故障，整个互联网都将瘫痪。另外，上网的人分布在全世界各地，如果大家都去同一个地方访问某一台服务器，时延将会非常大。因而，<strong>DNS服务器，一定要设置成高可用、高并发和分布式的</strong>。</p><p>于是，就有了这样<strong>树状的层次结构</strong>。</p><p><img src="https://static001.geekbang.org/resource/image/89/4d/890ff98fde625c6a60fb71yy22d8184d.jpg?wh=1512*984" alt="">-   根DNS服务器 ：返回顶级域DNS服务器的IP地址</p><ul>
<li>
<p>顶级域DNS服务器：返回权威DNS服务器的IP地址</p>
</li>
<li>
<p>权威DNS服务器 ：返回相应主机的IP地址</p>
</li>
</ul><h2>DNS解析流程</h2><p>为了提高DNS的解析性能，很多网络都会就近部署DNS缓存服务器。于是，就有了以下的DNS解析流程。</p><ol>
<li>
<p>电脑客户端会发出一个DNS请求，问www.163.com的IP是啥啊，并发给本地域名服务器 (本地DNS)。那本地域名服务器 (本地DNS) 是什么呢？如果是通过DHCP配置，本地DNS由你的网络服务商（ISP），如电信、移动等自动分配，它通常就在你网络服务商的某个机房。</p>
</li>
<li>
<p>本地DNS收到来自客户端的请求。你可以想象这台服务器上缓存了一张域名与之对应IP地址的大表格。如果能找到 www.163.com，它就直接返回IP地址。如果没有，本地DNS会去问它的根域名服务器：“老大，能告诉我www.163.com的IP地址吗？”根域名服务器是最高层次的，全球共有13套。它不直接用于域名解析，但能指明一条道路。</p>
</li>
<li>
<p>根DNS收到来自本地DNS的请求，发现后缀是 .com，说：“哦，www.163.com啊，这个域名是由.com区域管理，我给你它的顶级域名服务器的地址，你去问问它吧。”</p>
</li>
<li>
<p>本地DNS转向问顶级域名服务器：“老二，你能告诉我www.163.com的IP地址吗？”顶级域名服务器就是大名鼎鼎的比如 .com、.net、 .org这些一级域名，它负责管理二级域名，比如 163.com，所以它能提供一条更清晰的方向。</p>
</li>
<li>
<p>顶级域名服务器说：“我给你负责 www.163.com 区域的权威DNS服务器的地址，你去问它应该能问到。”</p>
</li>
<li>
<p>本地DNS转向问权威DNS服务器：“您好，www.163.com 对应的IP是啥呀？”163.com的权威DNS服务器，它是域名解析结果的原出处。为啥叫权威呢？就是我的域名我做主。</p>
</li>
<li>
<p>权威DNS服务器查询后将对应的IP地址X.X.X.X告诉本地DNS。</p>
</li>
<li>
<p>本地DNS再将IP地址返回客户端，客户端和目标建立连接。</p>
</li>
</ol><!-- [[[read_end]]] --><p>至此，我们完成了DNS的解析过程。现在总结一下，整个过程我画成了一个图。</p><p><img src="https://static001.geekbang.org/resource/image/71/e8/718e3a1a1a7927302b6a0f836409e8e8.jpg?wh=1456*1212" alt=""></p><h2>负载均衡</h2><p>站在客户端角度，这是一次<strong>DNS递归查询过程。<strong>因为本地DNS全权为它效劳，它只要坐等结果即可。在这个过程中，DNS除了可以通过名称映射为IP地址，它还可以做另外一件事，就是</strong>负载均衡</strong>。</p><p>还是以访问“外婆家”为例，还是我们开头的“外婆家”，但是，它可能有很多地址，因为它在杭州可以有很多家。所以，如果一个人想去吃“外婆家”，他可以就近找一家店，而不用大家都去同一家，这就是负载均衡。</p><p>DNS首先可以做<strong>内部负载均衡</strong>。</p><p>例如，一个应用要访问数据库，在这个应用里面应该配置这个数据库的IP地址，还是应该配置这个数据库的域名呢？显然应该配置域名，因为一旦这个数据库，因为某种原因，换到了另外一台机器上，而如果有多个应用都配置了这台数据库的话，一换IP地址，就需要将这些应用全部修改一遍。但是如果配置了域名，则只要在DNS服务器里，将域名映射为新的IP地址，这个工作就完成了，大大简化了运维。</p><p>在这个基础上，我们可以再进一步。例如，某个应用要访问另外一个应用，如果配置另外一个应用的IP地址，那么这个访问就是一对一的。但是当被访问的应用撑不住的时候，我们其实可以部署多个。但是，访问它的应用，如何在多个之间进行负载均衡？只要配置成为域名就可以了。在域名解析的时候，我们只要配置策略，这次返回第一个IP，下次返回第二个IP，就可以实现负载均衡了。</p><p>另外一个更加重要的是，DNS还可以做<strong>全局负载均衡</strong>。</p><p>为了保证我们的应用高可用，往往会部署在多个机房，每个地方都会有自己的IP地址。当用户访问某个域名的时候，这个IP地址可以轮询访问多个数据中心。如果一个数据中心因为某种原因挂了，只要在DNS服务器里面，将这个数据中心对应的IP地址删除，就可以实现一定的高可用。</p><p>另外，我们肯定希望北京的用户访问北京的数据中心，上海的用户访问上海的数据中心，这样，客户体验就会非常好，访问速度就会超快。这就是全局负载均衡的概念。</p><h2>示例：DNS访问数据中心中对象存储上的静态资源</h2><p>我们通过DNS访问数据中心中对象存储上的静态资源为例，看一看整个过程。</p><p>假设全国有多个数据中心，托管在多个运营商，每个数据中心三个可用区（Available Zone）。对象存储通过跨可用区部署，实现高可用性。在每个数据中心中，都至少部署两个内部负载均衡器，内部负载均衡器后面对接多个对象存储的前置服务器（Proxy-server）。</p><p><img src="https://static001.geekbang.org/resource/image/0b/b6/0b241afef775a1c942c5728364b302b6.jpg?wh=2200*2262" alt=""></p><ol>
<li>
<p>当一个客户端要访问object.yourcompany.com的时候，需要将域名转换为IP地址进行访问，所以它要请求本地DNS解析器。</p>
</li>
<li>
<p>本地DNS解析器先查看看本地的缓存是否有这个记录。如果有则直接使用，因为上面的过程太复杂了，如果每次都要递归解析，就太麻烦了。</p>
</li>
<li>
<p>如果本地无缓存，则需要请求本地的DNS服务器。</p>
</li>
<li>
<p>本地的DNS服务器一般部署在你的数据中心或者你所在的运营商的网络中，本地DNS服务器也需要看本地是否有缓存，如果有则返回，因为它也不想把上面的递归过程再走一遍。</p>
</li>
<li>
<p>至 7. 如果本地没有，本地DNS才需要递归地从根DNS服务器，查到.com的顶级域名服务器，最终查到 yourcompany.com 的权威DNS服务器，给本地DNS服务器，权威DNS服务器按说会返回真实要访问的IP地址。</p>
</li>
</ol><p>对于不需要做全局负载均衡的简单应用来讲，yourcompany.com的权威DNS服务器可以直接将 object.yourcompany.com这个域名解析为一个或者多个IP地址，然后客户端可以通过多个IP地址，进行简单的轮询，实现简单的负载均衡。</p><p>但是对于复杂的应用，尤其是跨地域跨运营商的大型应用，则需要更加复杂的全局负载均衡机制，因而需要专门的设备或者服务器来做这件事情，这就是<strong>全局负载均衡器</strong>（<strong>GSLB</strong>，<strong>Global Server Load Balance</strong>）。</p><p>在yourcompany.com的DNS服务器中，一般是通过配置CNAME的方式，给 object.yourcompany.com起一个别名，例如 object.vip.yourcomany.com，然后告诉本地DNS服务器，让它请求GSLB解析这个域名，GSLB就可以在解析这个域名的过程中，通过自己的策略实现负载均衡。</p><p>图中画了两层的GSLB，是因为分运营商和地域。我们希望不同运营商的客户，可以访问相同运营商机房中的资源，这样不跨运营商访问，有利于提高吞吐量，减少时延。</p><ol>
<li>
<p>第一层GSLB，通过查看请求它的本地DNS服务器所在的运营商，就知道用户所在的运营商。假设是移动，通过CNAME的方式，通过另一个别名 object.yd.yourcompany.com，告诉本地DNS服务器去请求第二层的GSLB。</p>
</li>
<li>
<p>第二层GSLB，通过查看请求它的本地DNS服务器所在的地址，就知道用户所在的地理位置，然后将距离用户位置比较近的Region里面，六个<strong>内部负载均衡</strong>（<strong>SLB</strong>，S<strong>erver Load Balancer</strong>）的地址，返回给本地DNS服务器。</p>
</li>
<li>
<p>本地DNS服务器将结果返回给本地DNS解析器。</p>
</li>
<li>
<p>本地DNS解析器将结果缓存后，返回给客户端。</p>
</li>
<li>
<p>客户端开始访问属于相同运营商的距离较近的Region 1中的对象存储，当然客户端得到了六个IP地址，它可以通过负载均衡的方式，随机或者轮询选择一个可用区进行访问。对象存储一般会有三个备份，从而可以实现对存储读写的负载均衡。</p>
</li>
</ol><h2>小结</h2><p>好了，这节内容就到这里了，我们来总结一下：</p><ul>
<li>
<p>DNS是网络世界的地址簿，可以通过域名查地址，因为域名服务器是按照树状结构组织的，因而域名查找是使用递归的方法，并通过缓存的方式增强性能；</p>
</li>
<li>
<p>在域名和IP的映射过程中，给了应用基于域名做负载均衡的机会，可以是简单的负载均衡，也可以根据地址和运营商做全局的负载均衡。</p>
</li>
</ul><p>最后，给你留两个思考题：</p><ol>
<li>
<p>全局负载均衡为什么要分地址和运营商呢？</p>
</li>
<li>
<p>全局负载均衡使用过程中，常常遇到失灵的情况，你知道具体有哪些情况吗？对应应该怎么来解决呢？</p>
</li>
</ol><p>我们的专栏更新过半了，不知你掌握得如何？每节课后我留的思考题，你都有没有认真思考，并在留言区写下答案呢？我会从<strong>已发布的文章中选出一批认真留言的同学</strong>，赠送**<span class="orange">学习奖励礼券</span><strong>和我整理的</strong><span class="orange">独家网络协议知识图谱</span>**。</p><p>欢迎你留言和我讨论。趣谈网络协议，我们下期见！</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/b0/c8/342a063f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>iiwoai</span>
  </div>
  <div class="_2_QraFYR_0">图谱应该都给赠送才对啊</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-27 08:44:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/21/0f/b61dc67e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>^_^</span>
  </div>
  <div class="_2_QraFYR_0">老师，独家网络协议知识图谱，这么好的东西，应该阳光普照，哈哈</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-27 07:32:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/6a/a8/ee414a25.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>公告-SRE运维实践</span>
  </div>
  <div class="_2_QraFYR_0">分地址和运营商主要是为了返回最优的ip，也就是离用户最近的ip，提高用户访问的速度，分运营商也是返回最快的一条路径。gslb失灵一般是因为一个ns请求gslb的时候，看不到用户真实的ip，从而不一定是最优的，而且这个返回的结果可能给一个用户或者一万个用户，可以通过流量监测来缓解。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-19 14:11:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/53/fa/6fb4e123.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>coldfire</span>
  </div>
  <div class="_2_QraFYR_0">从真名到别名怎么完成均衡负载，感觉没讲清楚，对于小白来说这就是一笔带过留了个影响</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-27 09:18:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/f2/46/09c457eb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Garwen</span>
  </div>
  <div class="_2_QraFYR_0">刘超老师，根域名服务器的访问不还是无法逃离高并发访问的场景吗，虽说本地的DNS已经可以解决80%请求的地址解析，但还是会有极其庞大数量的访问需要去找根域名服务器的吧，这里的根域名服务器的高并发场景不比12306小吧，是如何解决的呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 一方面，dns要求有缓存。另一方面，根服务器没有12306业务逻辑这么复杂，什么各种高铁段的计算啊，而且涉及事务，两个人不能同一个位置，涉及支付。根服务器基本只需要维护下一级别的就可以了，而下一个级别的相对稳定，而且都是静态的。所以支撑过双十一我们就知道，凡是静态的，我有一万种方式可以搞定，就怕落数据库</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-05 11:25:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/5a/06/ca4cd46e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>知识改变命运</span>
  </div>
  <div class="_2_QraFYR_0">感谢刘老师的讲解！这几次课的理解程度好多了！话说不是不愿意回答思考题，而是小白无力回答~最后，同求知识图谱~多谢多谢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-27 10:22:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/43/c1/576c01a1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>upstream</span>
  </div>
  <div class="_2_QraFYR_0">问题1，分成两级还是在于国内网络跨运营商访问慢<br>问题2，如果用户是上海电信，用户本地dns不是用的电信dns服务器，负载均衡调度会不准确。针对app的可以httpdns<br>针对网站的除非能支持edns协议才比较好。<br>另外当用户使用阿里云这类anycast 地址时，不清楚gslb调度策略如何的</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-27 09:15:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/43/c1/576c01a1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>upstream</span>
  </div>
  <div class="_2_QraFYR_0">虽然有了域名，但对于大多数人而言，能记住的就那么一两个网站地址。于是，打开baidu.com 搜索想要打开的网站，也就不奇怪了。打开baidu.com 搜索谷歌，再去访问Google 也就成为很多人的默认行为。说白了，也还是记不住域名。<br>Baidu.com 搜索qq邮箱的大有人在</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-27 08:01:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/67/61/ea54229a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jim</span>
  </div>
  <div class="_2_QraFYR_0">浏览器，路由器也有缓存吧？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-28 22:47:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/88/18/a88cdaf5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>alexgzh</span>
  </div>
  <div class="_2_QraFYR_0">以前只知道DNS是方便記憶。今天學到了DNS其實是解耦應用和被訪問應用的IP地址，還有局部負載均衡和全局負載均衡的功能, 學到了！！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对的，负载均衡</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-20 13:30:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/78/ac/e5e6e7f3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>古夜</span>
  </div>
  <div class="_2_QraFYR_0">这一课看懂了，还联想到了ping百度域名有时候返回ip不一样的问题，感谢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-19 09:21:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/e2/58/8c8897c8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>杨春鹏</span>
  </div>
  <div class="_2_QraFYR_0">当访问网站的时候，是先去访问host文件还是，本地DNS缓存？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: host</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-12 16:49:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/51/d2/dddffa12.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>胸怀宇宙瓜皮李</span>
  </div>
  <div class="_2_QraFYR_0">确实不是很理解为什么需要两次的全局负载均衡？为什么不在第一次的负载均衡中，直接获取DNS服务器所在地址和DNS服务提供商？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不是所有的人都要两层</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-14 15:53:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/27/d5/0fd21753.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>一粟</span>
  </div>
  <div class="_2_QraFYR_0">要解析本地DNS缓存，需要启动DNS Sclient服务，此服务不开启hosts配置会失效。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-07-15 16:26:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/7f/0c/392ce255.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>极客时间</span>
  </div>
  <div class="_2_QraFYR_0">引用原文中的话：<br>“在 yourcompany.com 的 DNS 服务器中，一般是通过配置 CNAME 的方式，给<br>object.yourcompany.com 起一个别名，例如 object.vip.yourcomany.com，然后告诉本地 DNS<br>服务器，让它请求 GSLB 解析这个域名，GSLB 就可以在解析这个域名的过程中，通过自己的策略实<br>现负载均衡”<br>这里有一句没明白，告诉本地服务器，让object.vip.yourcomany.com域名去GSLB解析，这个应该如何告诉呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 返回给他就可以了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-17 21:07:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/9e/80/b87f8b49.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>LongXiaJun</span>
  </div>
  <div class="_2_QraFYR_0">😃😃😃现在一周三天都在期待专栏更新~</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-27 07:47:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/27/f5/6e/55ec1ed1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>C++大雄</span>
  </div>
  <div class="_2_QraFYR_0">文中有一处不严谨，DNS分为迭代查询和递归查询，图三中5-9的查询应该是迭代，3是递归</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-07 09:01:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>起风了001</span>
  </div>
  <div class="_2_QraFYR_0">好像设置GSLB是通过添加NS记录实现的, 不是CNAME呀.</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 通过添加NS也是可以的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-22 18:19:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/ec/2a/b11d5ad8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>曾经瘦过</span>
  </div>
  <div class="_2_QraFYR_0">报名太晚了 不知道是否还可以获得独家网络知识图谱和学习奖励券</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 图谱最后一篇里面有啊</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-18 14:30:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/pTD8nS0SsORKiaRD3wB0NK9Bpd0wFnPWtYLPfBRBhvZ68iaJErMlM2NNSeEibwQfY7GReILSIYZXfT9o8iaicibcyw3g/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>雷刚</span>
  </div>
  <div class="_2_QraFYR_0">ISO 七层协议中每一层都有自己的寻址方式：应用层用过域名（主机名）寻址，网络层通过 IP 寻址，链路层通过MAC 寻址。这样每一层都是独立的，但不同层之间交流时就需要进行地址转换：<br>1. DNS 协议：将域名转换为 IP 地址。<br>2. ARP 协议：将 IP 转换为 MAC 地址。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-10 11:31:09</div>
  </div>
</div>
</div>
</li>
</ul>