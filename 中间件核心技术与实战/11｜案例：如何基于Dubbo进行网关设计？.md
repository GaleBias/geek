<audio title="11｜案例：如何基于Dubbo进行网关设计？" src="https://static001.geekbang.org/resource/audio/de/cf/de05ff84094f470173d092947c18c5cf.mp3" controls="controls"></audio> 
<p>你好，我是丁威。</p><p>这节课我们通过一个真实的业务场景来看看Dubbo网关（开放平台）的设计要领。</p><h2>设计背景</h2><p>要设计一个网关，我们首先要知道它的设计背景。</p><p>2017年，我从传统行业脱身，正式进入物流行业。说来也非常巧，我当时加入的是公司的网关项目组，主要解决泛化调用与协议转换代码的开发问题。刚进公司不久，网关项目组就遇到了技术难题。快递物流行业的业务量可以比肩互联网，从那时候开始，我的传统技术思维开始向互联网技术思维转变。</p><p>当时网关项目组的核心任务就是确保能够快速接入各个电商平台。我来简单说明一下具体的场景。</p><p><img src="https://static001.geekbang.org/resource/image/yy/af/yya8c848bc0c0870d5bb5b6bb41268af.jpg?wh=1920x539" alt="图片"></p><p>解释一下上面这个图。</p><p>物流公司内部已经基于Dubbo构建了订单中心微服务域，其中创建订单接口的定义如下：​</p><p><img src="https://static001.geekbang.org/resource/image/c4/61/c4yy4a46bcb07cbbfb6f0fe750de8861.jpg?wh=1920x936" alt="图片"></p><p>外部电商平台众多，每一家电商平台内部都有自己的标准，并不会遵循统一的标准。例如在淘宝中，当用户购买商品后，淘宝内部会定义一个统一的订单外派接口。它的请求包可能是这样的：</p><pre><code class="language-plain">{
 &nbsp;"seller_id":189,
 &nbsp;"buyer":"dingwei",
 &nbsp;"order":[
 &nbsp;  {
 &nbsp; &nbsp; &nbsp;"goods_name":"华为笔记本",
 &nbsp; &nbsp; &nbsp;"num":1,
 &nbsp; &nbsp; &nbsp;"price":500000
 &nbsp;  },
 &nbsp;  {
 &nbsp; &nbsp; &nbsp;"goods_name":"华为手表",
 &nbsp; &nbsp; &nbsp;"num":1,
 &nbsp; &nbsp; &nbsp;"price":200000
 &nbsp;  }
  ]
}
</code></pre><!-- [[[read_end]]] --><p>但拼多多内部定义的订单外派接口，它的请求包可能是下面这样的：</p><pre><code class="language-plain">&lt;order&gt;
 &nbsp;&lt;seller_uid&gt;189&lt;/seller_uid&gt;
 &nbsp;&lt;buyer_uid&gt;dingwei&lt;/buyer_uid&gt;
 &nbsp;&lt;order_items&gt;
 &nbsp; &nbsp;&lt;order_item&gt;
 &nbsp; &nbsp; &nbsp;&lt;goods_name&gt;华为笔记本&lt;/goods_name&gt;
 &nbsp; &nbsp; &nbsp;&lt;num&gt;1&lt;/num&gt;
 &nbsp; &nbsp; &nbsp;&lt;price&gt;500000&lt;/price&gt;
 &nbsp; &nbsp;&lt;/order_item&gt;
 &nbsp; &nbsp;&lt;order_item&gt;
 &nbsp; &nbsp; &nbsp;&lt;goods_name&gt;华为手表&lt;/goods_name&gt;
 &nbsp; &nbsp; &nbsp;&lt;num&gt;1&lt;/num&gt;
 &nbsp; &nbsp; &nbsp;&lt;price&gt;200000&lt;/price&gt;
 &nbsp; &nbsp;&lt;/order_item&gt;
 &nbsp;&lt;/order_items&gt;
&lt;/order&gt;
</code></pre><p>当电商的快递件占据快递公司总业务量的大半时，电商平台的话语权是高于快递公司的。也就是说，电商平台不管下游对接哪家物流公司，都会下发自己公司内部定义的订单派发接口，适配工作需要由物流公司自己来承担。</p><p>那站在物流公司的角度，应该怎么做呢？总不能每接入一个电商平台就为它们开发一套下单服务吧？那样的话，随着越来越多的电商平台接入，系统的复杂度会越来越高，可维护性将越来越差。</p><h2>设计方案</h2><p>正是在这样的背景下，网关平台被立项开发出来了。这个网关平台是怎么设计的呢？在设计的过程中需要解决哪些常见的问题？</p><p>我认为，网关的设计至少需要包括三个方面，分别是签名验证、服务配置和限流。</p><p>先说签名验证。保证请求的安全是系统设计需要优先考虑的。业界有一种非常经典的通信安全校验机制：<strong>验证签名。</strong></p><p>这种机制的做法是，客户端与服务端会首先采用HTTPS进行通信，确保传输过程的私密性。</p><p>客户端在发送请求时，先将请求参数按参数名称进行排序，然后按顺序拼接成字符串，格式为key1=a &amp; key2=b。接下来，客户端使用一个约定的密钥对拼接出来的参数字符串进行签名，生成签名字符串（我们用sign表示签名字符串）并追加到URL。通常，还会在URL中追加一个发送时间戳（时间戳不参与签名验证）。</p><p>服务端在接收到客户端的请求后，先从请求中解析出所有的参数，同样按照参数名对参数进行排序，然后使用同样的密钥对参数进行签名。得到的签名字符串需要与客户端计算的签名字符串进行对比，如果两者不同，则请求无效。与此同时，通常我们还需要将服务端当前的时间戳与客户端时间戳进行对比，如果相差超过一定的时间，同样认为请求无效，这个操作主要是为了避免使用同一个连接对网络进行连续攻击。</p><p>这整个过程里有一个非常重要的点，就是密钥自始至终并没有在网络上进行过传播，它的安全性可以得到十足的保证。签名验证的流程大概可以用下面这张图表示：</p><p><img src="https://static001.geekbang.org/resource/image/cb/7d/cb6c980fabc1afd0e76c4fe6627ce87d.jpg?wh=1920x1367" alt="图片"></p><p>如果要对验证签名进行产品化设计，我们通常需要：</p><ol>
<li>为不同的接入端（电商平台）创建不同的密钥，并通过安全的方式告知他们；</li>
<li>为不同的接入端（电商平台）配置签名算法。</li>
</ol><p>在确保能够安全通信后，接下来就是网关设计最核心的部分了：<strong>服务接口配置化。</strong>它主要包括两个要点：微服务调用协议（Dubbo服务描述）和接口定义与参数映射。</p><p>我们先来看一下微服务调用协议的配置，设计的原型界面如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/6e/59/6eed0bd5cbe4891082bb3e0678cb7b59.jpg?wh=1920x668" alt="图片"></p><p>将所有的微服务（细化到方法级名称）维护到网关系统中，网关应用就可以使用Dubbo提供的编程API，根据这些元信息动态构建一个个消费者（服务调用者），进而通过创建的服务调用客户端发起RPC远程调用，最终实现网关应用的Dubbo服务调用。</p><p>基于这些元信息构建消费者对象的关键代码如下：</p><pre><code class="language-plain">public static GenericService getInvoker(String serviceInterface, String version, List&lt;String&gt; methods, int retry, String registryAddr ) {
 &nbsp; &nbsp; &nbsp; &nbsp;ReferenceConfig referenceConfig = new ReferenceConfig();
 &nbsp; &nbsp; &nbsp; &nbsp;// 关于消费者通用参数，可以从配置文件中获取，本示例取消
 &nbsp; &nbsp; &nbsp; &nbsp;ConsumerConfig consumerConfig = new ConsumerConfig();
 &nbsp; &nbsp; &nbsp; &nbsp;consumerConfig.setTimeout(3000);
 &nbsp; &nbsp; &nbsp; &nbsp;consumerConfig.setRetries(2);
 &nbsp; &nbsp; &nbsp; &nbsp;referenceConfig.setConsumer(consumerConfig);
 &nbsp; &nbsp; &nbsp; &nbsp;//应用程序名称
 &nbsp; &nbsp; &nbsp; &nbsp;ApplicationConfig applicationConfig = new ApplicationConfig();
 &nbsp; &nbsp; &nbsp; &nbsp;applicationConfig.setName("GateWay");
 &nbsp; &nbsp; &nbsp; &nbsp;referenceConfig.setApplication(applicationConfig);
 &nbsp; &nbsp; &nbsp; &nbsp;// 注册中心
 &nbsp; &nbsp; &nbsp; &nbsp;RegistryConfig registry = new RegistryConfig();
 &nbsp; &nbsp; &nbsp; &nbsp;registry.setAddress(registryAddr);
 &nbsp; &nbsp; &nbsp; &nbsp;registry.setProtocol("zookeeper");
 &nbsp; &nbsp; &nbsp; &nbsp;referenceConfig.setRegistry(registry);
 &nbsp; &nbsp; &nbsp; &nbsp;// 设置服务接口名称
 &nbsp; &nbsp; &nbsp; &nbsp;referenceConfig.setInterface(serviceInterface);
 &nbsp; &nbsp; &nbsp; &nbsp;// 设置服务版本
 &nbsp; &nbsp; &nbsp; &nbsp;referenceConfig.setVersion(version);
 &nbsp; &nbsp; &nbsp; &nbsp;referenceConfig.setMethods(new ArrayList&lt;MethodConfig&gt;());
 &nbsp; &nbsp; &nbsp; &nbsp;for(String method : methods) {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;MethodConfig methodConfig = new MethodConfig();
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;methodConfig.setName(method);
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;referenceConfig.getMethods().add(methodConfig);
 &nbsp; &nbsp; &nbsp;  }
 &nbsp; &nbsp; &nbsp; &nbsp;referenceConfig.setGeneric("true");// 开启dubbo的泛化调用
 &nbsp; &nbsp; &nbsp; &nbsp;return (GenericService) referenceConfig.get();
 &nbsp;  }
</code></pre><p>通过getInvoker方法发起调用远程RPC服务，这样，<strong>网关应用就成为了对应服务的消费者</strong>。</p><p>因为网关应用引入服务规约（API包）不太现实，所以这里使用的是泛化调用，这样方便网关应用不受约束地构建消费者对象。</p><p>值得注意的是，ReferenceConfig实例很重，它封装了与注册中心的连接以及所有服务提供者的连接，需要被缓存起来。因此，在真实的生产实践中，我们需要将ReferenceConfig对象存储到缓存中。否则，重复生成的ReferenceConfig可能造成性能问题并伴随着内存和连接泄漏。</p><p>除了ReferenceConfig，其实getInvoker生成对象也可以进行缓存，缓存的key通常为接口名称、版本和注册中心。</p><p>那如果配置信息动态发生了变化，例如需要添加新的服务，这时候网关应用如何做到动态感知呢？我们通常可以用基于MQ的方式来解决这个问题。具体的解决方案如下：</p><p><img src="https://static001.geekbang.org/resource/image/01/0e/01a98d71ee4b0a1c56a390eb8e338f0e.jpg?wh=1920x619" alt="图片"></p><p>也就是说，用户如果在网关运营平台上修改原有服务协议（Dubbo服务）或者添加新的服务协议，变动后的协议会首先存储到数据库中，然后运营平台发送一条消息到MQ，紧接着Gateway的后台进程以广播模式进行订阅。这样，所有后台网关进程都可以感知。</p><p>如果是对已有服务协议进行修改，在具体实践时有一个小细节请你一定注意。我们先看看这段代码：</p><pre><code class="language-plain">Map&lt;String /* 缓存key */,GenericService&gt; invokerCache;
GenericService newInvoker = getInvoker(...);//参数省略
GenericService oldInvoker = invokerCache.get(key);
invokerCache.put(newInvoker);//先缓存新的invoker
// 然后再销毁旧的invoker对象
oldInvoker.destory();
</code></pre><p>如果已经存在对应的Invoker对象，为了不影响现有调用，应该先用新的Invoker对象去更新缓存，然后再销毁旧的Invoker对象。</p><p>上面的方法解决了网关调用公司内部的Dubbo微服务问题，但还有另外一个非常重要的问题，怎么配置服务接口相关参数呢？</p><p>联系这节课前面的场景，我们需要在页面上配置公司内部Dubbo服务与外部电商的接口映射。</p><p><img src="https://static001.geekbang.org/resource/image/23/79/233fc90464a70ab461d91766789df579.png?wh=1920x596" alt="图片"></p><p>为此，我们专门建立了一条参数映射协议：</p><p><img src="https://static001.geekbang.org/resource/image/51/af/51f750f5cb40a27a69e49a186b028faf.jpg?wh=1920x774" alt="图片"></p><p>参数映射设计的说明如下。</p><ul>
<li>请求类型：主要分为请求参数与响应参数；</li>
<li>字段名称：Dubbo服务对应的字段名称；</li>
<li>字段类型：Dubbo服务对应字段的属性；</li>
<li>字段所属类：Dubbo服务对应字段所属类型；</li>
<li>节点名称：外部请求接口对应的字段名称；</li>
<li>显示顺序：排序字段。</li>
</ul><p>由于网关采取了泛化调用，在编写转换代码时，主要是遍历传入的参数，根据每一个字段查询对应的转换规则，然后转换为Map，返回值则刚好相反，是将Map转换为XML或者JSON。</p><p>在真正请求调用时，根据映射规则构建出请求参数Map后，通过Dubbo的泛化调用执行真正的调用：</p><pre><code class="language-plain">GenericService genericService = (GenericService) invokeBean;
Map invokerPams;//省略转换过程
// 参数类型数组
String[] paramTypes = new String[1];
paramTypes[0]="java.util.Map";
// 参数值数组
Object[] paramValues = new Object[1];
​
invokerPams.put("class", "net.codingw.oms.vo.OrderItemVo");
paramValues[0] = invokerPams;
//由于我们已经转化为java.util.Map，并且Map中，需要有一个key为class的，表示服务端需要转化的类型，这个从协议转换器中获取
Object result = genericService.$invoke(this.getInvokeMethod(), paramTypes, paramValues);
</code></pre><p>这样，网关就具备了高扩展性和稳定性，可以非常灵活地支撑业务的扩展，为不同的电商平台配置不同的参数转换，从而在内部只需要开发一套接口就可以非常灵活地支撑业务的扩展，基本做到网关代码零修改。</p><h2>总结</h2><p>这节课，我通过一个真实的场景，详细介绍了网关设计的需求背景，然后针对网关设计的痛点给出了设计方案。通过对这个方案中关键代码的解读，你应该能够更加深刻地理解Dubbo泛化调用背后的逻辑，真正做到理论与实际相结合。</p><p>值得注意的是，我们这节课提到的转换协议也是一绝，它使用中括号来定义多层嵌套结构，使得该协议具有普适性。</p><h2>课后题</h2><p>检测对知识的掌握程度最好的方式是自己写出来。所以，我建议你将我们这节课所讲的方案落到实处，尝试自己实现一个demo级的网关设计。</p><p>如果你想听听我的意见，可以提交一个 <a href="https://github.com/dingwpmz/infoq_question">GitHub</a>的push请求或issues，并把对应地址贴到留言里。我们下节课见！</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/56/2f/4518f8e1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>放不下荣华富贵</span>
  </div>
  <div class="_2_QraFYR_0">数字签名部分：<br>计算完sign再追加时间戳，时间戳就可能篡改，那还是避免不了重放攻击吧？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对的，数字签名主要还是进行身份验证与防止数据被篡改，重放攻击通常在服务端结合redis等缓存组件，进行处理。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-26 08:44:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/78/b9/d8bb4c45.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>赤红热血</span>
  </div>
  <div class="_2_QraFYR_0">根据这些元信息动态构建一个个消费者（服务调用者），进而通过创建的服务调用客户端发起 RPC 远程调用，最终实现网关应用的 Dubbo 服务调用。<br>1.这句话里面，消费者为啥是服务调用者？<br>2.啥叫“创建的服务调用客户端发起rpc远程调用”？这句话读的通吗</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-09 11:06:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/0d/eb/c3ff1e85.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>🐝</span>
  </div>
  <div class="_2_QraFYR_0">如何进行协议转换呢，有什么方案</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-28 09:13:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/7c/89/93a4019e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>fantasyer</span>
  </div>
  <div class="_2_QraFYR_0">老师，关于网关这块有三个问题想咨询一下<br>1、验证签名这块提到，时间戳不参与签名验证，为什么时间戳不参与签名验证，我理解时间戳参与签名可以防止时间戳被恶意篡改，进而可以防止交易恶意重复提交<br>2、代码invokerPams.put(&quot;class&quot;, &quot;net.codingw.oms.vo.OrderItemVo&quot;);中这里的对象是不是应该是net.codingw.oms.vo.OrderVo，整个报文映射的类？<br>3、网关这部分设计重点讲了网关内部如何通过泛化调用后端的dubbo服务，想咨询下老师，这个案例中网关对外的接口部分是如何设计的？即给不同的电商是如何暴露接口服务的，从而灵活做到基本不用修改、快速扩展支持？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-20 08:53:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/77/b3/991f3f9b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>公号-技术夜未眠</span>
  </div>
  <div class="_2_QraFYR_0">如果已经存在对应的 Invoker 对象，为了不影响现有调用，应该先用新的 Invoker 对象去更新缓存，然后再销毁旧的 Invoker 对象。<br>其中，不影响现有调用，请问如何理解？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-12 08:53:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/6b/e9/7620ae7e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>雨落～紫竹</span>
  </div>
  <div class="_2_QraFYR_0">其实这种配置话实体 用起来很灵活 比如我们对接第三方云厂商 就是将 接口传参 请求路径 返回结构 解析结构 鉴权 全部做成了配置话 之后接入其他云厂商 也就只是 配置和联调的工作</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-09 20:01:31</div>
  </div>
</div>
</div>
</li>
</ul>