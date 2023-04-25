<audio title="第33讲 _ 基于XML的SOAP协议：不要说NBA，请说美国职业篮球联赛" src="https://static001.geekbang.org/resource/audio/7e/9a/7e3f7b2cbb61438d484ab627a7dcf39a.mp3" controls="controls"></audio> 
<p>上一节我们讲了RPC的经典模型和设计要点，并用最早期的ONC RPC为例子，详述了具体的实现。</p>
<h2>ONC RPC存在哪些问题？</h2>
<p>ONC RPC将客户端要发送的参数，以及服务端要发送的回复，都压缩为一个二进制串，这样固然能够解决双方的协议约定问题，但是存在一定的不方便。</p>
<p>首先，<strong>需要双方的压缩格式完全一致</strong>，一点都不能差。一旦有少许的差错，多一位，少一位或者错一位，都可能造成无法解压缩。当然，我们可以用传输层的可靠性以及加入校验值等方式，来减少传输过程中的差错。</p>
<p>其次，<strong>协议修改不灵活</strong>。如果不是传输过程中造成的差错，而是客户端因为业务逻辑的改变，添加或者删除了字段，或者服务端添加或者删除了字段，而双方没有及时通知，或者线上系统没有及时升级，就会造成解压缩不成功。</p>
<p>因而，当业务发生改变，需要多传输一些参数或者少传输一些参数的时候，都需要及时通知对方，并且根据约定好的协议文件重新生成双方的Stub程序。自然，这样灵活性比较差。</p>
<p>如果仅仅是沟通的问题也还好解决，其实更难弄的还有<strong>版本的问题</strong>。比如在服务端提供一个服务，参数的格式是版本一的，已经有50个客户端在线上调用了。现在有一个客户端有个需求，要加一个字段，怎么办呢？这可是一个大工程，所有的客户端都要适配这个，需要重新写程序，加上这个字段，但是传输值是0，不需要这个字段的客户端很“冤”，本来没我啥事儿，为啥让我也忙活？</p><!-- [[[read_end]]] -->
<p>最后，<strong>ONC RPC的设计明显是面向函数的，而非面向对象</strong>。而当前面向对象的业务逻辑设计与实现方式已经成为主流。</p>
<p>这一切的根源就在于压缩。这就像平时我们爱用缩略语。如果是篮球爱好者，你直接说NBA，他马上就知道什么意思，但是如果你给一个大妈说NBA，她可能就不知所云。</p>
<p>所以，这种RPC框架只能用于客户端和服务端全由一拨人开发的场景，或者至少客户端和服务端的开发人员要密切沟通，相互合作，有大量的共同语言，才能按照既定的协议顺畅地进行工作。</p>
<h2>XML与SOAP</h2>
<p>但是，一般情况下，我们做一个服务，都是要提供给陌生人用的，你和客户不会经常沟通，也没有什么共同语言。就像你给别人介绍NBA，你要说美国职业篮球赛，这样不管他是干啥的，都能听得懂。</p>
<p>放到我们的场景中，对应的就是用<strong>文本类</strong>的方式进行传输。无论哪个客户端获得这个文本，都能够知道它的意义。</p>
<p>一种常见的文本类格式是XML。我们这里举个例子来看。</p>
<pre><code>&lt;?xml version=&quot;1.0&quot; encoding=&quot;UTF-8&quot;?&gt;
&lt;geek:purchaseOrder xmlns:xsi=&quot;http://www.w3.org/2001/XMLSchema-instance&quot; xmlns:geek=&quot;http://www.example.com/geek&quot;&gt;
    &lt;order&gt;
        &lt;date&gt;2018-07-01&lt;/date&gt;
        &lt;className&gt;趣谈网络协议&lt;/className&gt;
        &lt;Author&gt;刘超&lt;/Author&gt;
        &lt;price&gt;68&lt;/price&gt;
    &lt;/order&gt;
&lt;/geek:purchaseOrder&gt;
</code></pre>
<p>我这里不准备详细讲述XML的语法规则，但是你相信我，看完下面的内容，即便你没有学过XML，也能一看就懂，这段XML描述的是什么，不像全面的二进制，你看到的都是010101，不知所云。</p>
<p>有了这个，刚才我们说的那几个问题就都不是问题了。</p>
<p>首先，<strong>格式没必要完全一致</strong>。比如如果我们把price和author换个位置，并不影响客户端和服务端解析这个文本，也根本不会误会，说这个作者的名字叫68。</p>
<p>如果有的客户端想增加一个字段，例如添加一个推荐人字段，只需要在上面的文件中加一行：</p>
<pre><code>&lt;recommended&gt; Gary &lt;/recommended&gt; 
</code></pre>
<p>对于不需要这个字段的客户端，只要不解析这一行就是了。只要用简单的处理，就不会出现错误。</p>
<p>另外，这种表述方式显然是描述一个订单对象的，是一种面向对象的、更加接近用户场景的表示方式。</p>
<p>既然XML这么好，接下来我们来看看怎么把它用在RPC中。</p>
<h3>传输协议问题</h3>
<p>我们先解决第一个，传输协议的问题。</p>
<p>基于XML的最著名的通信协议就是<strong>SOAP</strong>了，全称<strong>简单对象访问协议</strong>（Simple Object Access Protocol）。它使用XML编写简单的请求和回复消息，并用HTTP协议进行传输。</p>
<p>SOAP将请求和回复放在一个信封里面，就像传递一个邮件一样。信封里面的信分<strong>抬头</strong>和<strong>正文</strong>。</p>
<pre><code>POST /purchaseOrder HTTP/1.1
Host: www.geektime.com
Content-Type: application/soap+xml; charset=utf-8
Content-Length: nnn
</code></pre>
<pre><code>&lt;?xml version=&quot;1.0&quot;?&gt;
&lt;soap:Envelope xmlns:soap=&quot;http://www.w3.org/2001/12/soap-envelope&quot;
soap:encodingStyle=&quot;http://www.w3.org/2001/12/soap-encoding&quot;&gt;
    &lt;soap:Header&gt;
        &lt;m:Trans xmlns:m=&quot;http://www.w3schools.com/transaction/&quot;
          soap:mustUnderstand=&quot;1&quot;&gt;1234
        &lt;/m:Trans&gt;
    &lt;/soap:Header&gt;
    &lt;soap:Body xmlns:m=&quot;http://www.geektime.com/perchaseOrder&quot;&gt;
        &lt;m:purchaseOrder&quot;&gt;
            &lt;order&gt;
                &lt;date&gt;2018-07-01&lt;/date&gt;
                &lt;className&gt;趣谈网络协议&lt;/className&gt;
                &lt;Author&gt;刘超&lt;/Author&gt;
                &lt;price&gt;68&lt;/price&gt;
            &lt;/order&gt;
        &lt;/m:purchaseOrder&gt;
    &lt;/soap:Body&gt;
&lt;/soap:Envelope&gt;
</code></pre>
<p>HTTP协议我们学过，这个请求使用POST方法，发送一个格式为 application/soap + xml 的XML正文给 <a href="http://www.geektime.com">www.geektime.com</a>，从而下一个单，这个订单封装在SOAP的信封里面，并且表明这是一笔交易（transaction），而且订单的详情都已经写明了。</p>
<h3>协议约定问题</h3>
<p>接下来我们解决第二个问题，就是双方的协议约定是什么样的？</p>
<p>因为服务开发出来是给陌生人用的，就像上面下单的那个XML文件，对于客户端来说，它如何知道应该拼装成上面的格式呢？这就需要对于服务进行描述，因为调用的人不认识你，所以没办法找到你，问你的服务应该如何调用。</p>
<p>当然你可以写文档，然后放在官方网站上，但是你的文档不一定更新得那么及时，而且你也写的文档也不一定那么严谨，所以常常会有调试不成功的情况。因而，我们需要一种相对比较严谨的<strong>Web服务描述语言</strong>，<strong>WSDL</strong>（Web Service Description Languages）。它也是一个XML文件。</p>
<p>在这个文件中，要定义一个类型order，与上面的XML对应起来。</p>
<pre><code> &lt;wsdl:types&gt;
  &lt;xsd:schema targetNamespace=&quot;http://www.example.org/geektime&quot;&gt;
   &lt;xsd:complexType name=&quot;order&quot;&gt;
    &lt;xsd:element name=&quot;date&quot; type=&quot;xsd:string&quot;&gt;&lt;/xsd:element&gt;
&lt;xsd:element name=&quot;className&quot; type=&quot;xsd:string&quot;&gt;&lt;/xsd:element&gt;
&lt;xsd:element name=&quot;Author&quot; type=&quot;xsd:string&quot;&gt;&lt;/xsd:element&gt;
    &lt;xsd:element name=&quot;price&quot; type=&quot;xsd:int&quot;&gt;&lt;/xsd:element&gt;
   &lt;/xsd:complexType&gt;
  &lt;/xsd:schema&gt;
 &lt;/wsdl:types&gt;
</code></pre>
<p>接下来，需要定义一个message的结构。</p>
<pre><code> &lt;wsdl:message name=&quot;purchase&quot;&gt;
  &lt;wsdl:part name=&quot;purchaseOrder&quot; element=&quot;tns:order&quot;&gt;&lt;/wsdl:part&gt;
 &lt;/wsdl:message&gt;
</code></pre>
<p>接下来，应该暴露一个端口。</p>
<pre><code> &lt;wsdl:portType name=&quot;PurchaseOrderService&quot;&gt;
  &lt;wsdl:operation name=&quot;purchase&quot;&gt;
   &lt;wsdl:input message=&quot;tns:purchase&quot;&gt;&lt;/wsdl:input&gt;
   &lt;wsdl:output message=&quot;......&quot;&gt;&lt;/wsdl:output&gt;
  &lt;/wsdl:operation&gt;
 &lt;/wsdl:portType&gt;
</code></pre>
<p>然后，我们来编写一个binding，将上面定义的信息绑定到SOAP请求的body里面。</p>
<pre><code> &lt;wsdl:binding name=&quot;purchaseOrderServiceSOAP&quot; type=&quot;tns:PurchaseOrderService&quot;&gt;
  &lt;soap:binding style=&quot;rpc&quot;
   transport=&quot;http://schemas.xmlsoap.org/soap/http&quot; /&gt;
  &lt;wsdl:operation name=&quot;purchase&quot;&gt;
   &lt;wsdl:input&gt;
    &lt;soap:body use=&quot;literal&quot; /&gt;
   &lt;/wsdl:input&gt;
   &lt;wsdl:output&gt;
    &lt;soap:body use=&quot;literal&quot; /&gt;
   &lt;/wsdl:output&gt;
  &lt;/wsdl:operation&gt;
 &lt;/wsdl:binding&gt;
</code></pre>
<p>最后，我们需要编写service。</p>
<pre><code> &lt;wsdl:service name=&quot;PurchaseOrderServiceImplService&quot;&gt;
  &lt;wsdl:port binding=&quot;tns:purchaseOrderServiceSOAP&quot; name=&quot;PurchaseOrderServiceImplPort&quot;&gt;
   &lt;soap:address location=&quot;http://www.geektime.com:8080/purchaseOrder&quot; /&gt;
  &lt;/wsdl:port&gt;
 &lt;/wsdl:service&gt;
</code></pre>
<p>WSDL还是有些复杂的，不过好在有工具可以生成。</p>
<p>对于某个服务，哪怕是一个陌生人，都可以通过在服务地址后面加上“?wsdl”来获取到这个文件，但是这个文件还是比较复杂，比较难以看懂。不过好在也有工具可以根据WSDL生成客户端Stub，让客户端通过Stub进行远程调用，就跟调用本地的方法一样。</p>
<h3>服务发现问题</h3>
<p>最后解决第三个问题，服务发现问题。</p>
<p>这里有一个<strong>UDDI</strong>（Universal Description, Discovery, and Integration），也即<strong>统一描述、发现和集成协议</strong>。它其实是一个注册中心，服务提供方可以将上面的WSDL描述文件，发布到这个注册中心，注册完毕后，服务使用方可以查找到服务的描述，封装为本地的客户端进行调用。</p>
<h2>小结</h2>
<p>好了，这一节就到这里了，我们来总结一下。</p>
<ul>
<li>
<p>原来的二进制RPC有很多缺点，格式要求严格，修改过于复杂，不面向对象，于是产生了基于文本的调用方式——基于XML的SOAP。</p>
</li>
<li>
<p>SOAP有三大要素：协议约定用WSDL、传输协议用HTTP、服务发现用UDDL。</p>
</li>
</ul>
<p>最后，给你留两个思考题：</p>
<ol>
<li>
<p>对于HTTP协议来讲，有多种方法，但是SOAP只用了POST，这样会有什么问题吗？</p>
</li>
<li>
<p>基于文本的RPC虽然解决了二进制的问题，但是SOAP还是有点复杂，还有一种更便捷的接口规则，你知道是什么吗？</p>
</li>
</ol>
<p>我们的专栏更新到第33讲，不知你掌握得如何？每节课后我留的思考题，你都有没有认真思考，并在留言区写下答案呢？我会从<strong>已发布的文章中选出一批认真留言的同学</strong>，赠送<span class="orange">学习奖励礼券</span>和我整理的<span class="orange">独家网络协议知识图谱</span>。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/43/62/cd7d8b3b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>叹息无门</span>
  </div>
  <div class="_2_QraFYR_0">感觉这篇写的不是很严谨:<br>1，首先SOAP并非只能通过HTTP进行传输，关于SOAP binding应该提一下？<br>2，SOAP 的HTTP Binding 支持比较完整的Web Method，http GET&#47;POST都是可以支持的，并且对应不同的模式。大多数情况下只使用POST是具体实现的问题。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，这里说的是通常的使用情况</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-01 09:52:15</div>
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
  <div class="_2_QraFYR_0">1.虽然http协议有post，get，head，put，delete等多种方法，但是平常来说post，get基本足够用。所以soap只支持post方法的差别应该在缺少get方法，get方法可以浏览器直接跳转，post必须借助表单或者ajax等提交。也就限制了soap请求只能在页面内获取或者提交数据。 <br>另外，soap协议规范上是支持get的，但是由于一般xml比较复杂，不适合放在get请求的查询参数里，所以soap协议的服务多采用post请求方法。<br>2.应该要讲restful协议了，一种使用json格式交互数据的，基于http协议的，轻量级网络数据交互规范。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-01 09:15:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/5a/31/14f00b1c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>燃</span>
  </div>
  <div class="_2_QraFYR_0">webservice   soap的初始目标：服务自描述，其实就是没有达成，UDDI早已默默死掉</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，但是作为rpc历史上不能不说的一个环节，也要讲一下</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-16 10:49:12</div>
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
  <div class="_2_QraFYR_0">1.没有充分利用http协议原有的体系 比如get表示获取资源 post表示创建资源 delete表示删除资源 patch表示更新资源<br>2.restful协议</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-01 08:02:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/d5/00/a88c24c0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>雪山飞猪</span>
  </div>
  <div class="_2_QraFYR_0">这套教程真的是比以前看过的好多书都要生动形象，高手出马，化繁为简，非常感谢老师的倾情分享</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-17 23:24:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/4e/b2/57c39f3d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>vloz</span>
  </div>
  <div class="_2_QraFYR_0">面向函数和面向对象在信息交互上的特征是什么？为什么讲onc合适面向函数？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-01 10:43:22</div>
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
  <div class="_2_QraFYR_0">题目1:<br>HTTP请求里面有很多种提交方式，文中只是提到了可以用post，其实还是可以用其他方式的，比如get。<br>题目2:<br>restful，用json格式的数据发送请求和返回数据。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-06 19:15:53</div>
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
  <div class="_2_QraFYR_0">1. POST 请求构造比较麻烦，需要专门的工具，所以调用和调试更费事。<br>2. 更简单的应该就是RESTful 了吧，SOAP 感觉不太好用，复杂度比较高，用起来没有http顺手。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-01 14:20:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/7a/5e/9431165a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>spdia</span>
  </div>
  <div class="_2_QraFYR_0">soap的方言问题过于严重。其实简单场景可以用http rest或者json+http post,或者用比较新的graphql</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-01 07:42:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/40/18/cc3804e2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>沈洪彬</span>
  </div>
  <div class="_2_QraFYR_0">不是网络传输都要变成二进制传输么？  是不是xml也要变成二进制才能传输，这个是谁去做？ stub么？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-16 14:29:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/f7/bb/98a8b8be.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>LHW</span>
  </div>
  <div class="_2_QraFYR_0">不用get是考虑传输的内容长度大小吧，如果是提交文件流的形式，post怎么处理？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-13 08:26:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/37/31/53b449e9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>andy</span>
  </div>
  <div class="_2_QraFYR_0">可以使用类似thrift的DSL来描述服务接口，然后生成服务端和客户端</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-01 07:58:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/96/78/eb86673c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我在你的视线里</span>
  </div>
  <div class="_2_QraFYR_0">webserice和HTTP 有什么区别和联系呢？接口传数据不都是http传输吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-09 13:31:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/0f/d7/31d07471.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>牛年榴莲</span>
  </div>
  <div class="_2_QraFYR_0">现在还有用SOAP协议的吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-11 18:26:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/72/85/c337e9a1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>老兵</span>
  </div>
  <div class="_2_QraFYR_0">1. 用post是因为每次都是到service创建一些资源，之前为什么只用post呢？是不是因为访问soap大部分情况都是需要在服务端保存一些数据之类？ get, post, put, patch, delete也都是http协议里面的method类型。<br>2. restful，grpc</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-27 20:04:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/39/f9/b2fe7b63.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>King-ZJ</span>
  </div>
  <div class="_2_QraFYR_0">在微服务调用这块得多学多看多实践。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-26 11:20:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/7a/32/27a8572a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>渣渣</span>
  </div>
  <div class="_2_QraFYR_0">1.只用http (参数在body中)，如果客户端不是用post接收，就收不到，有一些不太灵活，<br>2.restful</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-27 11:51:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/d1/29/1b1234ed.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>DFighting</span>
  </div>
  <div class="_2_QraFYR_0">读完之后才发现最初的rpc协议并没有做到对调用者的完全透明，不利于扩展</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-02 17:16:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/58/32/535e5c3c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>mlbjay</span>
  </div>
  <div class="_2_QraFYR_0">之前工程里需要调用第三方的SOAP接口，于是研究了一下web servies，但是云里雾里的。<br>这下感觉WSDL，XML，SOAP等技术的关系现在清晰多了。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-25 15:22:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/0c/2f/54f7f676.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Jerry Chan</span>
  </div>
  <div class="_2_QraFYR_0">但是这个二进制格式，怎么转换为xml这种格式呢？过程是怎么解析的呢？主要是这块不清楚，作者能解惑下不？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不会有二进制转换成为xml，是由对象转换成为xml<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-18 13:13:16</div>
  </div>
</div>
</div>
</li>
</ul>