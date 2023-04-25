<audio title="第34讲 _ 基于JSON的RESTful接口协议：我不关心过程，请给我结果" src="https://static001.geekbang.org/resource/audio/7a/cb/7a7e5b3b70e60c69b8a533659260dacb.mp3" controls="controls"></audio> 
<p>上一节我们讲了基于XML的SOAP协议，SOAP的S是啥意思来着？是Simple，但是好像一点儿都不简单啊！</p>
<p>你会发现，对于SOAP来讲，无论XML中调用的是什么函数，多是通过HTTP的POST方法发送的。但是咱们原来学HTTP的时候，我们知道HTTP除了POST，还有PUT、DELETE、GET等方法，这些也可以代表一个个动作，而且基本满足增、删、查、改的需求，比如增是POST，删是DELETE，查是GET，改是PUT。</p>
<h2>传输协议问题</h2>
<p>对于SOAP来讲，比如我创建一个订单，用POST，在XML里面写明动作是CreateOrder；删除一个订单，还是用POST，在XML里面写明了动作是DeleteOrder。其实创建订单完全可以使用POST动作，然后在XML里面放一个订单<!-- [[[read_end]]] -->的信息就可以了，而删除用DELETE动作，然后在XML里面放一个订单的ID就可以了。</p>
<p>于是上面的那个SOAP就变成下面这个简单的模样。</p>
<pre><code>POST /purchaseOrder HTTP/1.1
Host: www.geektime.com
Content-Type: application/xml; charset=utf-8
Content-Length: nnn

&lt;?xml version=&quot;1.0&quot;?&gt;
 &lt;order&gt;
     &lt;date&gt;2018-07-01&lt;/date&gt;
      &lt;className&gt;趣谈网络协议&lt;/className&gt;
       &lt;Author&gt;刘超&lt;/Author&gt;
       &lt;price&gt;68&lt;/price&gt;
  &lt;/order&gt;
</code></pre>
<p>而且XML的格式也可以改成另外一种简单的文本化的对象表示格式JSON。</p>
<pre><code>POST /purchaseOrder HTTP/1.1
Host: www.geektime.com
Content-Type: application/json; charset=utf-8
Content-Length: nnn

{
 &quot;order&quot;: {
  &quot;date&quot;: &quot;2018-07-01&quot;,
  &quot;className&quot;: &quot;趣谈网络协议&quot;,
  &quot;Author&quot;: &quot;刘超&quot;,
  &quot;price&quot;: &quot;68&quot;
 }
}
</code></pre>
<p>经常写Web应用的应该已经发现，这就是RESTful格式的API的样子。</p>
<h2>协议约定问题</h2>
<p>然而RESTful可不仅仅是指API，而是一种架构风格，全称Representational State Transfer，表述性状态转移，来自一篇重要的论文《架构风格与基于网络的软件架构设计》（Architectural Styles and the Design of Network-based Software Architectures）。</p>
<p>这篇文章从深层次，更加抽象地论证了一个互联网应用应该有的设计要点，而这些设计要点，成为后来我们能看到的所有高并发应用设计都必须要考虑的问题，再加上REST API比较简单直接，所以后来几乎成为互联网应用的标准接口。</p>
<p>因此，和SOAP不一样，REST不是一种严格规定的标准，它其实是一种设计风格。如果按这种风格进行设计，RESTful接口和SOAP接口都能做到，只不过后面的架构是REST倡导的，而SOAP相对比较关注前面的接口。</p>
<p>而且由于能够通过WSDL生成客户端的Stub，因而SOAP常常被用于类似传统的RPC方式，也即调用远端和调用本地是一样的。</p>
<p>然而本地调用和远程跨网络调用毕竟不一样，这里的不一样还不仅仅是因为有网络而导致的客户端和服务端的分离，从而带来的网络性能问题。更重要的问题是，客户端和服务端谁来维护状态。所谓的状态就是对某个数据当前处理到什么程度了。</p>
<p>这里举几个例子，例如，我浏览到哪个目录了，我看到第几页了，我要买个东西，需要扣减一下库存，这些都是状态。本地调用其实没有人纠结这个问题，因为数据都在本地，谁处理都一样，而且一边处理了，另一边马上就能看到。</p>
<p>当有了RPC之后，我们本来期望对上层透明，就像上一节说的“远在天边，尽在眼前”。于是使用RPC的时候，对于状态的问题也没有太多的考虑。</p>
<p>就像NFS一样，客户端会告诉服务端，我要进入哪个目录，服务端必须要为某个客户端维护一个状态，就是当前这个客户端浏览到哪个目录了。例如，客户端输入cd hello，服务端要在某个地方记住，上次浏览到/root/liuchao了，因而客户的这次输入，应该给它显示/root/liuchao/hello下面的文件列表。而如果有另一个客户端，同样输入cd hello，服务端也在某个地方记住，上次浏览到/var/lib，因而要给客户显示的是/var/lib/hello。</p>
<p>不光NFS，如果浏览翻页，我们经常要实现函数next()，在一个列表中取下一页，但是这就需要服务端记住，客户端A上次浏览到20～30页了，那它调用next()，应该显示30～40页，而客户端B上次浏览到100～110页了，调用next()应该显示110～120页。</p>
<p>上面的例子都是在RPC场景下，由服务端来维护状态，很多SOAP接口设计的时候，也常常按这种模式。这种模式原来没有问题，是因为客户端和服务端之间的比例没有失衡。因为一般不会同时有太多的客户端同时连上来，所以NFS还能把每个客户端的状态都记住。</p>
<p>公司内部使用的ERP系统，如果使用SOAP的方式实现，并且服务端为每个登录的用户维护浏览到报表那一页的状态，由于一个公司内部的人也不会太多，把ERP放在一个强大的物理机上，也能记得过来。</p>
<p>但是互联网场景下，客户端和服务端就彻底失衡了。你可以想象“双十一”，多少人同时来购物，作为服务端，它能记得过来吗？当然不可能，只好多个服务端同时提供服务，大家分担一下。但是这就存在一个问题，服务端怎么把自己记住的客户端状态告诉另一个服务端呢？或者说，你让我给你分担工作，你也要把工作的前因后果给我说清楚啊！</p>
<p>那服务端索性就要想了，既然这么多客户端，那大家就分分工吧。服务端就只记录资源的状态，例如文件的状态，报表的状态，库存的状态，而客户端自己维护自己的状态。比如，你访问到哪个目录了啊，报表的哪一页了啊，等等。</p>
<p>这样对于API也有影响，也就是说，当客户端维护了自己的状态，就不能这样调用服务端了。例如客户端说，我想访问当前目录下的hello路径。服务端说，我怎么知道你的当前路径。所以客户端要先看看自己当前路径是/root/liuchao，然后告诉服务端说，我想访问/root/liuchao/hello路径。</p>
<p>再比如，客户端说我想访问下一页，服务端说，我怎么知道你当前访问到哪一页了。所以客户端要先看看自己访问到了100～110页，然后告诉服务器说，我想访问110～120页。</p>
<p>这就是服务端的无状态化。这样服务端就可以横向扩展了，一百个人一起服务，不用交接，每个人都能处理。</p>
<p>所谓的无状态，其实是服务端维护资源的状态，客户端维护会话的状态。对于服务端来讲，只有资源的状态改变了，客户端才调用POST、PUT、DELETE方法来找我；如果资源的状态没变，只是客户端的状态变了，就不用告诉我了，对于我来说都是统一的GET。</p>
<p>虽然这只改进了GET，但是已经带来了很大的进步。因为对于互联网应用，大多数是读多写少的。而且只要服务端的资源状态不变，就给了我们缓存的可能。例如可以将状态缓存到接入层，甚至缓存到CDN的边缘节点，这都是资源状态不变的好处。</p>
<p>按照这种思路，对于API的设计，就慢慢变成了以资源为核心，而非以过程为核心。也就是说，客户端只要告诉服务端你想让资源状态最终变成什么样就可以了，而不用告诉我过程，不用告诉我动作。</p>
<p>还是文件目录的例子。客户端应该访问哪个绝对路径，而非一个动作，我就要进入某个路径。再如，库存的调用，应该查看当前的库存数目，然后减去购买的数量，得到结果的库存数。这个时候应该设置为目标库存数（但是当前库存数要匹配），而非告知减去多少库存。</p>
<p>这种API的设计需要实现幂等，因为网络不稳定，就会经常出错，因而需要重试，但是一旦重试，就会存在幂等的问题，也就是同一个调用，多次调用的结果应该一样，不能一次支付调用，因为调用三次变成了支付三次。不能进入cd a，做了三次，就变成了cd a/a/a。也不能扣减库存，调用了三次，就扣减三次库存。</p>
<p>当然按照这种设计模式，无论RESTful API还是SOAP API都可以将架构实现成无状态的，面向资源的、幂等的、横向扩展的、可缓存的。</p>
<p>但是SOAP的XML正文中，是可以放任何动作的。例如XML里面可以写&lt; ADD &gt;，&lt; MINUS &gt;等。这就方便使用SOAP的人，将大量的动作放在API里面。</p>
<p>RESTful没这么复杂，也没给客户提供这么多的可能性，正文里的JSON基本描述的就是资源的状态，没办法描述动作，而且能够出发的动作只有CRUD，也即POST、GET、PUT、DELETE，也就是对于状态的改变。</p>
<p>所以，从接口角度，就让你死了这条心。当然也有很多技巧的方法，在使用RESTful API的情况下，依然提供基于动作的有状态请求，这属于反模式了。</p>
<h2>服务发现问题</h2>
<p>对于RESTful API来讲，我们已经解决了传输协议的问题——基于HTTP，协议约定问题——基于JSON，最后要解决的是服务发现问题。</p>
<p>有个著名的基于RESTful API的跨系统调用框架叫Spring  Cloud。在Spring  Cloud中有一个组件叫 Eureka。传说，阿基米德在洗澡时发现浮力原理，高兴得来不及穿上裤子，跑到街上大喊：“Eureka（我找到了）！”所以Eureka是用来实现注册中心的，负责维护注册的服务列表。</p>
<p>服务分服务提供方，它向Eureka做服务注册、续约和下线等操作，注册的主要数据包括服务名、机器IP、端口号、域名等等。</p>
<p>另外一方是服务消费方，向Eureka获取服务提供方的注册信息。为了实现负载均衡和容错，服务提供方可以注册多个。</p>
<p>当消费方要调用服务的时候，会从注册中心读出多个服务来，那怎么调用呢？当然是RESTful方式了。</p>
<p>Spring Cloud提供一个RestTemplate工具，用于将请求对象转换为JSON，并发起Rest调用，RestTemplate的调用也是分POST、PUT、GET、  DELETE的，当结果返回的时候，根据返回的JSON解析成对象。</p>
<p>通过这样封装，调用起来也很方便。</p>
<h2>小结</h2>
<p>好了，这一节就到这里了，我们来总结一下。</p>
<ul>
<li>
<p>SOAP过于复杂，而且设计是面向动作的，因而往往因为架构问题导致并发量上不去。</p>
</li>
<li>
<p>RESTful不仅仅是一个API，而且是一种架构模式，主要面向资源，提供无状态服务，有利于横向扩展应对高并发。</p>
</li>
</ul>
<p>最后，给你留两个思考题：</p>
<ol>
<li>
<p>在讨论RESTful模型的时候，举了一个库存的例子，但是这种方法有很大问题，那你知道为什么要这样设计吗？</p>
</li>
<li>
<p>基于文本的RPC虽然解决了二进制的问题，但是它本身也有问题，你能举出一些例子吗？</p>
</li>
</ol>
<p>我们的专栏更新到第34讲，不知你掌握得如何？每节课后我留的思考题，你都有没有认真思考，并在留言区写下答案呢？我会从<strong>已发布的文章中选出一批认真留言的同学</strong>，赠送<span class="orange">学习奖励礼券</span>和我整理的<span class="orange">独家网络协议知识图谱</span>。</p>
<p>欢迎你留言和我讨论。趣谈网络协议，我们下期见！</p>
<p></p>

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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/de/17/75e2b624.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>feifei</span>
  </div>
  <div class="_2_QraFYR_0">在讨论 RESTful 模型的时候，举了一个库存的例子，但是这种方法有很大问题，那你知道为什么要这样设计吗？<br><br>此方法的问题在于，不是解决问题，而是将数据状态进行了转移，将状态交给存储，这样业务将可以无状态化运行，这种设计可以很好的解决扩展的问题，因为无状态，可以进行负载均衡！使用集群化来解决单机的问题。<br><br>基于文本的 RPC 虽然解决了二进制的问题，但是它本身也有问题，你能举出一些例子吗？<br><br>1，效率问题，程序与文本之间转换效率低，因而不适合内部大数据交换，因为文本利用阅读，对外采用较好<br><br>2，相比于二进制rpc,传输需要的带宽更大，二进制的rpc因为可以使用专用的客户短和服务器代码，可以更好的压缩数据，以提供更大的吞吐量</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-03 07:53:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/6f/63/abb7bfe3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>扬～</span>
  </div>
  <div class="_2_QraFYR_0">文本传输最终都会转化为二进制流啊，为什么文本要比二进制rpc占用带宽？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 数字2如果用int传输用几个bit，如果是字符串呢？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-10 11:08:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/5c/b5/0737c1f2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>kuzan</span>
  </div>
  <div class="_2_QraFYR_0">crud的语义离业务有点远，客户端往往不想关心crud，客户端关注的语义是业务，比如审批、下单，添加好友。感觉用http这几个method做语义就把服务变成了dao，一个贫血服务</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-08 22:59:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/23/5b/983408b9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>悟空聊架构</span>
  </div>
  <div class="_2_QraFYR_0">题目1:<br>库存的问题是：存在并发，导致库存可能为负值。<br>用Restful来进行无状态的访问，库存量即状态由业务层来解决。<br>题目2:<br>用文本来进行RPC的请求和响应，占用的字节数大。<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-06 19:38:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/86/e3/28d1330a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>fsj</span>
  </div>
  <div class="_2_QraFYR_0">没有restful api之前的json api 是什么样子的？感觉对于客户端同学，不了解之前是什么样子，很难体会到restful api的有点。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-08 07:24:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/5e/2b/df3983e2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>朱显杰</span>
  </div>
  <div class="_2_QraFYR_0">能不能谈谈dubbo和springcloud在服务发现方面的优缺点？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-03 10:53:46</div>
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
  <div class="_2_QraFYR_0">文本传输最终都会转化为二进制流啊，为什么文本要比二进制rpc占用带宽？<br><br>1<br>2018-11-10<br>作者回复: 数字2如果用int传输用几个bit，如果是字符串呢？<br><br>这个有点疑问, 用int传输需要32或者64位; 字符串的话, 看编码, 如果是utf8编码的话, 还是1个字节8bit即可表示.....</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: int当然不会用32位编码呀</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-24 17:27:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/c2/e0/7188aa0a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>blackpiglet</span>
  </div>
  <div class="_2_QraFYR_0">第一题，我的理解是，资源最终还是有状态的，所以 rest 方式，是把状态转移到了数据库和客户端。那么数据库的抗压能力和稳定性就非常重要了，这也是为什么最近有这么多内存数据库和键值数据库的原因。另外客户端有状态也会造成很多麻烦，毕竟这是不受开发人员控制的，如果逻辑没有切割清楚，升级会非常痛苦。<br>第二题，能想到的主要是性能的损耗，毕竟传输的内容更多了，单位带宽下能传输的信息总量会有明显下降。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-07 21:32:59</div>
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
  <div class="_2_QraFYR_0">第一个问题有点模糊，感觉这个设计没有问题，任何传输方式，一旦经过网络，都会发生很多可能性，必须做幂等处理。如果指的是服务端无状态的话，原因就是提升扩展性，应对C端用户的大规模和并发。<br><br>第二个问题，主要在于序列化和传输。序列化方面由于有格式就不如二进制紧凑，传输的数据量相对来说要大。另一个是二进制可以自定义规范，或者编码方案，传输路径上，数据被截获，也不容易解析。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-03 09:42:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/ed/f1/e31585a8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jonhey</span>
  </div>
  <div class="_2_QraFYR_0">看到这里，强烈建议老刘基于此教程，丰富一下内容，写本书，估计能成为网络、云计算、网络编程、微服务领域集大成的经典教材</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不敢不敢</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-18 16:56:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/32/fc/5d901185.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>vic</span>
  </div>
  <div class="_2_QraFYR_0">楼上那位，登陆动作可以看作是对session的CRUD操作</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-04 22:48:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/85/7c/03a268fe.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>leohuachao</span>
  </div>
  <div class="_2_QraFYR_0">我觉得RESTFul架构流行，也得益于前端框架的丰富吧，要不然维护客户端会话也够难实现</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，简单</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-04 19:11:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/f3/1f/05f97f0f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>葱本</span>
  </div>
  <div class="_2_QraFYR_0">问个问题，购买这个动作，是告诉服务端减一，难受没有办法做到只告诉服务端结果吧</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 购买可不止减库存，服务端可以有个状态机的，要不然重复购买怎么办。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-04 09:25:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_f6f02b</span>
  </div>
  <div class="_2_QraFYR_0">RESTful 模型如何实现幂等，这个表示没有想通，就是好比之前SOAP，你告诉我减库存，我就执行减，减后还剩多少，是否成功返回，但是如果是客户端直接减去库存，然后告诉我说，将库存设置成这么多，服务端只要告诉我是否成功，并发了怎么解决，多个人同时请求，如果库存为10，5个请求都是减1，如果前面失败，但是后面成功，前面分别说将库存设置成9、8、7、6，都失败了，最后一个说设置成5成功了，感觉会有问题。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你这种减是有问题的，并发怎么办，后面5个只有一个成功是对的，其他还可以重试。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-25 20:27:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2e/27/b4/df65c0f7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>| ~浑蛋~</span>
  </div>
  <div class="_2_QraFYR_0">基于文本的 RPC怎么做文件传输，base64编码吗</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-20 17:32:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2c/5e/7b/0e1eb97a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>luojielu</span>
  </div>
  <div class="_2_QraFYR_0">存在并冲突的问题，比如两个都是出库的操作，两个操作都在未减库存前的时间点查询了库存，然后都返回了服务端减去库存后的数量，正确的结果应该是分开执行</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-20 15:39:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/4a/8a/c1069412.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>makermade</span>
  </div>
  <div class="_2_QraFYR_0">讲得很好</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-27 23:17:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/24/df/645f8087.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Yayu</span>
  </div>
  <div class="_2_QraFYR_0">JSON-RESTful 算是一种协议吗？把它理解成一种规范会更好吧？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 没必要纠结吧</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-15 15:02:14</div>
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
  <div class="_2_QraFYR_0">这篇把无状态服务讲透彻了，太棒了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-07 09:46:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/50/2b/2344cdaa.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>第一装甲集群司令克莱斯特</span>
  </div>
  <div class="_2_QraFYR_0">太赞了！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-26 11:10:06</div>
  </div>
</div>
</div>
</li>
</ul>