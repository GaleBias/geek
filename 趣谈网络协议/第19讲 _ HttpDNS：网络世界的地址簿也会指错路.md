<audio title="第19讲 _ HttpDNS：网络世界的地址簿也会指错路" src="https://static001.geekbang.org/resource/audio/c5/8e/c5878b08f79096cdad80d9f9e656818e.mp3" controls="controls"></audio> 
<p>上一节我们知道了DNS的两项功能，第一是根据名称查到具体的地址，另外一个是可以针对多个地址做负载均衡，而且可以在多个地址中选择一个距离你近的地方访问。</p><p>然而有时候这个地址簿也经常给你指错路，明明距离你500米就有个吃饭的地方，非要把你推荐到5公里外。为什么会出现这样的情况呢？</p><p>还记得吗？当我们发出请求解析DNS的时候，首先，会先连接到运营商本地的DNS服务器，由这个服务器帮我们去整棵DNS树上进行解析，然后将解析的结果返回给客户端。但是本地的DNS服务器，作为一个本地导游，往往有自己的“小心思”。</p><h2>传统DNS存在哪些问题？</h2><h3>1.域名缓存问题</h3><p>它可以在本地做一个缓存，也就是说，不是每一个请求，它都会去访问权威DNS服务器，而是访问过一次就把结果缓存到自己本地，当其他人来问的时候，直接就返回这个缓存数据。</p><p>这就相当于导游去过一个饭店，自己脑子记住了地址，当有一个游客问的时候，他就凭记忆回答了，不用再去查地址簿。这样经常存在的一个问题是，人家那个饭店明明都已经搬了，结果作为导游，他并没有刷新这个缓存，结果你辛辛苦苦到了这个地点，发现饭店已经变成了服装店，你是不是会非常失望？</p><p>另外，有的运营商会把一些静态页面，缓存到本运营商的服务器内，这样用户请求的时候，就不用跨运营商进行访问，这样既加快了速度，也减少了运营商之间流量计算的成本。在域名解析的时候，不会将用户导向真正的网站，而是指向这个缓存的服务器。</p><!-- [[[read_end]]] --><p>很多情况下是看不出问题的，但是当页面更新，用户会访问到老的页面，问题就出来了。例如，你听说一个餐馆推出了一个新菜，你想去尝一下。结果导游告诉你，在这里吃也是一样的。有的游客会觉得没问题，但是对于想尝试新菜的人来说，如果导游说带你去，但其实并没有吃到新菜，你是不是也会非常失望呢？</p><p>再就是本地的缓存，往往使得全局负载均衡失败，因为上次进行缓存的时候，缓存中的地址不一定是这次访问离客户最近的地方，如果把这个地址返回给客户，那肯定就会绕远路。</p><p>就像上一次客户要吃西湖醋鱼的事，导游知道西湖边有一家，因为当时游客就在西湖边，可是，下一次客户在灵隐寺，想吃西湖醋鱼的时候，导游还指向西湖边的那一家，那这就绕得太远了。</p><p><img src="https://static001.geekbang.org/resource/image/8b/eb/8b5e670e22c459e859293db1bed643eb.jpg?wh=1383*965" alt=""></p><h3>2.域名转发问题</h3><p>缓存问题还是说本地域名解析服务，还是会去权威DNS服务器中查找，只不过不是每次都要查找。可以说这还是大导游、大中介。还有一些小导游、小中介，有了请求之后，直接转发给其他运营商去做解析，自己只是外包了出去。</p><p>这样的问题是，如果是A运营商的客户，访问自己运营商的DNS服务器，如果A运营商去权威DNS服务器查询的话，权威DNS服务器知道你是A运营商的，就返回给一个部署在A运营商的网站地址，这样针对相同运营商的访问，速度就会快很多。</p><p>但是A运营商偷懒，将解析的请求转发给B运营商，B运营商去权威DNS服务器查询的话，权威服务器会误认为，你是B运营商的，那就返回给你一个在B运营商的网站地址吧，结果客户的每次访问都要跨运营商，速度就会很慢。</p><p><img src="https://static001.geekbang.org/resource/image/cf/f1/cfde4b3bc5c0aabaaa81f3a26cd99cf1.jpg?wh=1645*1055" alt=""></p><h3>3.出口NAT问题</h3><p>前面讲述网关的时候，我们知道，出口的时候，很多机房都会配置<strong>NAT</strong>，也即<strong>网络地址转换</strong>，使得从这个网关出去的包，都换成新的IP地址，当然请求返回的时候，在这个网关，再将IP地址转换回去，所以对于访问来说是没有任何问题。</p><p>但是一旦做了网络地址的转换，权威的DNS服务器，就没办法通过这个地址，来判断客户到底是来自哪个运营商，而且极有可能因为转换过后的地址，误判运营商，导致跨运营商的访问。</p><h3>4.域名更新问题</h3><p>本地DNS服务器是由不同地区、不同运营商独立部署的。对域名解析缓存的处理上，实现策略也有区别，有的会偷懒，忽略域名解析结果的TTL时间限制，在权威DNS服务器解析变更的时候，解析结果在全网生效的周期非常漫长。但是有的时候，在DNS的切换中，场景对生效时间要求比较高。</p><p>例如双机房部署的时候，跨机房的负载均衡和容灾多使用DNS来做。当一个机房出问题之后，需要修改权威DNS，将域名指向新的IP地址，但是如果更新太慢，那很多用户都会出现访问异常。</p><p>这就像，有的导游比较勤快、敬业，时时刻刻关注酒店、餐馆、交通的变化，问他的时候，往往会得到最新情况。有的导游懒一些，8年前背的导游词就没换过，问他的时候，指的路往往就是错的。</p><h3>5.解析延迟问题</h3><p>从上一节的DNS查询过程来看，DNS的查询过程需要递归遍历多个DNS服务器，才能获得最终的解析结果，这会带来一定的时延，甚至会解析超时。</p><h2>HttpDNS的工作模式</h2><p>既然DNS解析中有这么多问题，那怎么办呢？难不成退回到直接用IP地址？这样显然不合适，所以就有了<strong> HttpDNS</strong>。</p><p>HttpDNS其实就是，不走传统的DNS解析，而是自己搭建基于HTTP协议的DNS服务器集群，分布在多个地点和多个运营商。当客户端需要DNS解析的时候，直接通过HTTP协议进行请求这个服务器集群，得到就近的地址。</p><p>这就相当于每家基于HTTP协议，自己实现自己的域名解析，自己做一个自己的地址簿，而不使用统一的地址簿。但是默认的域名解析都是走DNS的，因而使用HttpDNS需要绕过默认的DNS路径，就不能使用默认的客户端。使用HttpDNS的，往往是手机应用，需要在手机端嵌入支持HttpDNS的客户端SDK。</p><p>通过自己的HttpDNS服务器和自己的SDK，实现了从依赖本地导游，到自己上网查询做旅游攻略，进行自由行，爱怎么玩怎么玩。这样就能够避免依赖导游，而导游又不专业，你还不能把他怎么样的尴尬。</p><p>下面我来解析一下<strong> HttpDNS的工作模式</strong>。</p><p>在客户端的SDK里动态请求服务端，获取HttpDNS服务器的IP列表，缓存到本地。随着不断地解析域名，SDK也会在本地缓存DNS域名解析的结果。</p><p>当手机应用要访问一个地址的时候，首先看是否有本地的缓存，如果有就直接返回。这个缓存和本地DNS的缓存不一样的是，这个是手机应用自己做的，而非整个运营商统一做的。如何更新、何时更新，手机应用的客户端可以和服务器协调来做这件事情。</p><p>如果本地没有，就需要请求HttpDNS的服务器，在本地HttpDNS服务器的IP列表中，选择一个发出HTTP的请求，会返回一个要访问的网站的IP列表。</p><p>请求的方式是这样的。</p><pre><code>curl http://106.2.xxx.xxx/d?dn=c.m.163.com
{&quot;dns&quot;:[{&quot;host&quot;:&quot;c.m.163.com&quot;,&quot;ips&quot;:[&quot;223.252.199.12&quot;],&quot;ttl&quot;:300,&quot;http2&quot;:0}],&quot;client&quot;:{&quot;ip&quot;:&quot;106.2.81.50&quot;,&quot;line&quot;:269692944}}
</code></pre><p>手机客户端自然知道手机在哪个运营商、哪个地址。由于是直接的HTTP通信，HttpDNS服务器能够准确知道这些信息，因而可以做精准的全局负载均衡。</p><p><img src="https://static001.geekbang.org/resource/image/aa/75/aa45cf8a07b563ccea376f712b2e8975.jpg?wh=1581*1261" alt=""></p><p>当然，当所有这些都不工作的时候，可以切换到传统的LocalDNS来解析，慢也比访问不到好。那HttpDNS是如何解决上面的问题的呢？</p><p>其实归结起来就是两大问题。一是解析速度和更新速度的平衡问题，二是智能调度的问题，对应的解决方案是HttpDNS的缓存设计和调度设计。</p><h3>HttpDNS的缓存设计</h3><p>解析DNS过程复杂，通信次数多，对解析速度造成很大影响。为了加快解析，因而有了缓存，但是这又会产生缓存更新速度不及时的问题。最要命的是，这两个方面都掌握在别人手中，也即本地DNS服务器手中，它不会为你定制，你作为客户端干着急没办法。</p><p>而HttpDNS就是将解析速度和更新速度全部掌控在自己手中。一方面，解析的过程，不需要本地DNS服务递归的调用一大圈，一个HTTP的请求直接搞定，要实时更新的时候，马上就能起作用；另一方面为了提高解析速度，本地也有缓存，缓存是在客户端SDK维护的，过期时间、更新时间，都可以自己控制。</p><p>HttpDNS的缓存设计策略也是咱们做应用架构中常用的缓存设计模式，也即分为客户端、缓存、数据源三层。</p><ul>
<li>
<p>对于应用架构来讲，就是应用、缓存、数据库。常见的是Tomcat、Redis、MySQL。</p>
</li>
<li>
<p>对于HttpDNS来讲，就是手机客户端、DNS缓存、HttpDNS服务器。</p>
</li>
</ul><p><img src="https://static001.geekbang.org/resource/image/9e/56/9e0141a2939e7c194689e990859ed456.jpg?wh=871*868" alt=""></p><p>只要是缓存模式，就存在缓存的过期、更新、不一致的问题，解决思路也是很像的。</p><p>例如DNS缓存在内存中，也可以持久化到存储上，从而APP重启之后，能够尽快从存储中加载上次累积的经常访问的网站的解析结果，就不需要每次都全部解析一遍，再变成缓存。这有点像Redis是基于内存的缓存，但是同样提供持久化的能力，使得重启或者主备切换的时候，数据不会完全丢失。</p><p>SDK中的缓存会严格按照缓存过期时间，如果缓存没有命中，或者已经过期，而且客户端不允许使用过期的记录，则会发起一次解析，保障记录是更新的。</p><p>解析可以<strong>同步进行</strong>，也就是直接调用HttpDNS的接口，返回最新的记录，更新缓存；也可以<strong>异步进行</strong>，添加一个解析任务到后台，由后台任务调用HttpDNS的接口。</p><p><strong>同步更新</strong>的<strong>优点</strong>是实时性好，缺点是如果有多个请求都发现过期的时候，同时会请求HttpDNS多次，其实是一种浪费。</p><p>同步更新的方式对应到应用架构中缓存的<strong>Cache-Aside机制</strong>，也即先读缓存，不命中读数据库，同时将结果写入缓存。</p><img style="margin: 0 auto" src="https://static001.geekbang.org/resource/image/34/b4/346a1bf30fb56ef918d708f422dc3bb4.jpg?wh=664*801"><p><strong>异步更新</strong>的<strong>优点</strong>是，可以将多个请求都发现过期的情况，合并为一个对于HttpDNS的请求任务，只执行一次，减少HttpDNS的压力。同时可以在即将过期的时候，就创建一个任务进行预加载，防止过期之后再刷新，称为<strong>预加载</strong>。</p><p>它的<strong>缺点</strong>是当前请求拿到过期数据的时候，如果客户端允许使用过期数据，需要冒一次风险。如果过期的数据还能请求，就没问题；如果不能请求，则失败一次，等下次缓存更新后，再请求方能成功。</p><p><img src="https://static001.geekbang.org/resource/image/df/98/df65059e9b66c32bbbca6febf6ecb298.jpg?wh=472*826" alt=""></p><p>异步更新的机制对应到应用架构中缓存的<strong>Refresh-Ahead机制</strong>，即业务仅仅访问缓存，当过期的时候定期刷新。在著名的应用缓存Guava Cache中，有个RefreshAfterWrite机制，对于并发情况下，多个缓存访问不命中从而引发并发回源的情况，可以采取只有一个请求回源的模式。在应用架构的缓存中，也常常用<strong>数据预热</strong>或者<strong>预加载</strong>的机制。</p><p><img src="https://static001.geekbang.org/resource/image/16/28/161a14f41b3547739982ec551aa26928.jpg?wh=626*770" alt=""></p><h3>HttpDNS的调度设计</h3><p>由于客户端嵌入了SDK，因而就不会因为本地DNS的各种缓存、转发、NAT，让权威DNS服务器误会客户端所在的位置和运营商，而可以拿到第一手资料。</p><p>在<strong>客户端</strong>，可以知道手机是哪个国家、哪个运营商、哪个省，甚至哪个市，HttpDNS服务端可以根据这些信息，选择最佳的服务节点访问。</p><p>如果有多个节点，还会考虑错误率、请求时间、服务器压力、网络状况等，进行综合选择，而非仅仅考虑地理位置。当有一个节点宕机或者性能下降的时候，可以尽快进行切换。</p><p>要做到这一点，需要客户端使用HttpDNS返回的IP访问业务应用。客户端的SDK会收集网络请求数据，如错误率、请求时间等网络请求质量数据，并发送到统计后台，进行分析、聚合，以此查看不同的IP的服务质量。</p><p>在<strong>服务端</strong>，应用可以通过调用HttpDNS的管理接口，配置不同服务质量的优先级、权重。HttpDNS会根据这些策略综合地理位置和线路状况算出一个排序，优先访问当前那些优质的、时延低的IP地址。</p><p>HttpDNS通过智能调度之后返回的结果，也会缓存在客户端。为了不让缓存使得调度失真，客户端可以根据不同的移动网络运营商WIFI的SSID来分维度缓存。不同的运营商或者WIFI解析出来的结果会不同。</p><p><img src="https://static001.geekbang.org/resource/image/3b/ae/3b368yy7a4f2319bd6491bc4e66e55ae.jpg?wh=1937*1634" alt=""></p><h2>小结</h2><p>好了，这节就到这里了，我们来总结一下，你需要记住这两个重点：</p><ul>
<li>
<p>传统的DNS有很多问题，例如解析慢、更新不及时。因为缓存、转发、NAT问题导致客户端误会自己所在的位置和运营商，从而影响流量的调度。</p>
</li>
<li>
<p>HttpDNS通过客户端SDK和服务端，通过HTTP直接调用解析DNS的方式，绕过了传统DNS的这些缺点，实现了智能的调度。</p>
</li>
</ul><p>最后，给你留两个思考题。</p><ol>
<li>
<p>使用HttpDNS，需要向HttpDNS服务器请求解析域名，可是客户端怎么知道HttpDNS服务器的地址或者域名呢？</p>
</li>
<li>
<p>HttpDNS的智能调度，主要是让客户端选择最近的服务器，而有另一种机制，使得资源分发到离客户端更近的位置，从而加快客户端的访问，你知道是什么技术吗？</p>
</li>
</ol><p>我们的专栏更新过半了，不知你掌握得如何？每节课后我留的思考题，你都有没有认真思考，并在留言区写下答案呢？我会从已发布的文章中选出一批认真留言的同学，赠送学习奖励礼券和我整理的独家网络协议知识图谱。</p><p>欢迎你留言和我讨论。趣谈网络协议，我们下期见！</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/61/bc/88a905a5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>亮点</span>
  </div>
  <div class="_2_QraFYR_0">老师能不能写个专题介绍一下国内几大运营商网络状况和互联关系😬 </div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-29 08:32:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/7c/16/4d1e5cc1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>mgxian</span>
  </div>
  <div class="_2_QraFYR_0">1.httpdns服务器的地址一般不变 可以使用dns的方式获取httpdns服务器的ip地址 也可以直接把httpdns服务器的ip地址写死在客户端中。<br><br>2.让用户获取离用户最近的资源 一般是静态资源 这使用的是cdn技术</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-29 08:14:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/80/fb/ffa57759.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>xyz</span>
  </div>
  <div class="_2_QraFYR_0">1. HTTPDNS 服务器的地址或者域名（和网站域名类似）一般不变，所以可以写死在SDK里；<br><br>2..为方便用户近源访问，使用CDN技术（content distribution network，内容分发网络）。<br><br><br>一点题外话：作者的每一期讲解都有好好学习，讲的都挺通俗易懂的，不过如果能加入一些相关的经典的参考链接，应该会更好，授人以鱼的同时也授人以渔。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-29 23:30:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/4b/b2/7f600f9e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>silence</span>
  </div>
  <div class="_2_QraFYR_0">之前在软件公司干活，有个大客户，有很多站点，但是他们原先的系统不支持调度。希望我们给他出个方案，做到智能调度。当时我出的方案是给他们搞一个类似 HTTPDNS 的系统。根据运营商、地理位置、站点负载进行调度。一次返回多个服务列表。让客户端进行缓存，定期更新。今天看了这个文章，在回想起当时我给他们做的方案，当时只做了服务器上的调度，客户端因为无法控制，当时没考虑进行改造，添加反馈。看了文章，看到还是有可以再优化的点的。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-19 16:10:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTIu1n1DhUGGKS1obrPQicEHlhC7A5zuu6fCQVR4PNx06LNEgfITQqic9fRGYicI0TkFMQUfDAnR7YVLA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>quanbove</span>
  </div>
  <div class="_2_QraFYR_0">HTTPNDS 其实就是，不走传统的 DNS 解析，而是自己搭建基于 HTTP 协议的 DNS 服务器集群，分布在多个地点和多个运营商。<br>发现一个笔误的地方，HTTPDNS，而不是HTTPNDS，我是又多无聊[捂脸]</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢指正</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-26 23:05:39</div>
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
  <div class="_2_QraFYR_0">3.出口nat问题这段个人认为说的不准确。客户端的dns服务器指向一般都是一个运营商的localdns，不考虑edns的情况下，权威服务器拿到的是localdns地址。<br>如果递归dns支持edns，权威服务器能看到客户端出口nat地址。<br>只有像长城宽带这类流氓二道贩子，会把用户出口nat地址全国飘</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-29 07:53:21</div>
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
  <div class="_2_QraFYR_0">我想问的是:HttpDns服务器的域名地址簿，也是通过递归根域名，顶级域名，权威域名服务器获取的吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-03 09:21:05</div>
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
  <div class="_2_QraFYR_0">这个文章看起来感觉httpdns能解决传统的dns问题，但是前提是有自己的客户端，这种要对业务进行全部改造。那么如果就是浏览器访问，有没有什么方案，在不使用cdn的情况下</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: httpdns需要自己的客户端</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-21 18:19:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/60/c7/a147b71b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Fisher</span>
  </div>
  <div class="_2_QraFYR_0">最近几篇文章都很赞啊，这一篇讲清楚了为什么跨运营商速度会慢的原因，之前知道这个问题但是不知道答案，其次httpdns这个新技术第一次听说，赞一个</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-29 22:02:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/07/8c/0d886dcc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>蚂蚁内推+v</span>
  </div>
  <div class="_2_QraFYR_0">HTTPDNS能支持HTTPS 不<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-16 18:02:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/a0/a7/db7a7c50.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>送普选</span>
  </div>
  <div class="_2_QraFYR_0">咨询一下httpdns如何获取权威dns上的IP地址？每次有客户端httpdns请求来了就查一下，还是每1秒定期轮询后缓存？谢谢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-14 09:20:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/b4/94/2796de72.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>追风筝的人</span>
  </div>
  <div class="_2_QraFYR_0">非计算机专业同学，老师能送我一份网络协议知识图谱吗？<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 买了课程的，最后一节都有</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-05 17:33:15</div>
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
  <div class="_2_QraFYR_0">1、可以是通过域名解析，也可以是直接配置所有httpdns服务器节点。阿里云采用的是配置ip列表的方式。2、cdn内容分发网络</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-07-03 21:46:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/53/0c/b4907516.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Marco</span>
  </div>
  <div class="_2_QraFYR_0">那httpdns不需要考虑负载均衡么?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 考虑，httpdns里面也可以配置负载均衡策略的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-19 14:12:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>哈哈哈</span>
  </div>
  <div class="_2_QraFYR_0">请问老师：文中的在不同运营商或地区部署httpdns服务，根据客户端位置返回最优应用地址，httpdns服务端部署方式是如何的？如何达到不同运营商或地区效果？httpdns适用场景只有哪些呢？谢谢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 一般是多个数据中心分布式部署，适用于手机客户端的情况</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-14 14:59:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/08/94/2c22bd4e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>克里斯</span>
  </div>
  <div class="_2_QraFYR_0">权威DNS服务器的ip地址不会变吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 很少变</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-08 12:17:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/yy4cUibeUfPHPkXXZQnQwjXY7m5rXY5ib6a7pC1vkupj1icibF305N4pJSdqw0fO1ibvyfKCQ7HWggLhwiaNbbRPBsKg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>桃子妈妈</span>
  </div>
  <div class="_2_QraFYR_0">老师好，很想了解HTTPDNS服务端仍然会使用递归的方式依次通过根域名、顶级域名、权威域名去递归查询吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不会的，直接返回了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-31 14:21:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/b6/43/c23a38b2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>whats your name</span>
  </div>
  <div class="_2_QraFYR_0">老师，有一个疑惑点：HTTPDNS的域名&#47;ip地址簿是从哪里来的呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 管理员配置，反正httpdns服务器也是你自己搭建的，如果你用公有云，他也让你配置的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-13 23:30:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/2b/84/07f0c0d6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>supermouse</span>
  </div>
  <div class="_2_QraFYR_0">想问一下老师，HTTPDNS服务器集群是谁来搭建的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你可以自己搭建，也可以去公有云买</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-11 00:11:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/9e/c3/c321541c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Nicole</span>
  </div>
  <div class="_2_QraFYR_0">1.写死在SDK中，支持DNS over HTTPS方案<br>2.CDN内容分发服务</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-03 08:22:52</div>
  </div>
</div>
</div>
</li>
</ul>