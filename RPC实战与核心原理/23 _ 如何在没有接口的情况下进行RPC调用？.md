<audio title="23 _ 如何在没有接口的情况下进行RPC调用？" src="https://static001.geekbang.org/resource/audio/fa/44/fa88615ff18b8a21dbec7d9e7e4d6944.mp3" controls="controls"></audio> 
<p>你好，我是何小锋。上一讲我们学习了RPC如何通过动态分组来实现秒级扩缩容，其关键点就是“动态”与“隔离”。今天我们来聊聊如何在没有接口的情况下进行RPC调用。</p><h2>应用场景有哪些？</h2><p>在RPC运营的过程中，让调用端在没有接口API的情况下发起RPC调用的需求，不只是一个业务方和我提过，这里我列举两个非常典型的场景例子。</p><p><strong>场景一：</strong>我们要搭建一个统一的测试平台，可以让各个业务方在测试平台中通过输入接口、分组名、方法名以及参数值，在线测试自己发布的RPC服务。这时我们就有一个问题要解决，我们搭建统一的测试平台实际上是作为各个RPC服务的调用端，而在RPC框架的使用中，调用端是需要依赖服务提供方提供的接口API的，而统一测试平台不可能依赖所有服务提供方的接口API。我们不能因为每有一个新的服务发布，就去修改平台的代码以及重新上线。这时我们就需要让调用端在没有服务提供方提供接口的情况下，仍然可以正常地发起RPC调用。</p><p><img src="https://static001.geekbang.org/resource/image/fc/bc/fc0027ad042768d9aabf68182de5d2bc.jpg?wh=2792*1137" alt="" title="示意图"></p><p><strong>场景二：</strong>我们要搭建一个轻量级的服务网关，可以让各个业务方用HTTP的方式，通过服务网关调用其它服务。这时就有与场景一相同的问题，服务网关要作为所有RPC服务的调用端，是不能依赖所有服务提供方的接口API的，也需要调用端在没有服务提供方提供接口的情况下，仍然可以正常地发起RPC调用。</p><!-- [[[read_end]]] --><p><img src="https://static001.geekbang.org/resource/image/09/c5/09bd6312f3bdb5d4e9276bd0cb0025c5.jpg?wh=2769*1138" alt="" title="示意图"></p><p>这两个场景都是我们经常会碰到的，而让调用端在没有服务提供方提供接口API的情况下仍然可以发起RPC调用的功能，在RPC框架中也是非常有价值的。</p><h2>怎么做？</h2><p>RPC框架要实现这个功能，我们可以使用泛化调用。那什么是泛化调用呢？我们带着这个问题，先学习下如何在没有接口的情况下进行RPC调用。</p><p>我们先回想下我在基础篇讲过的内容，通过前面的学习我们了解到，在RPC调用的过程中，调用端向服务端发起请求，首先要通过动态代理，正如<a href="https://time.geekbang.org/column/article/205910">[第 05 讲]</a> 中我说过的，动态代理可以帮助我们屏蔽RPC处理流程，真正地让我们发起远程调用就像调用本地一样。</p><p>那么在RPC调用的过程中，既然调用端是通过动态代理向服务端发起远程调用的，那么在调用端的程序中就一定要依赖服务提供方提供的接口API，因为调用端是通过这个接口API自动生成动态代理的。那如果没有接口API呢？我们该如何让调用端仍然能够发起RPC调用呢？</p><p>所谓的RPC调用，本质上就是调用端向服务端发送一条请求消息，服务端接收并处理，之后向调用端发送一条响应消息，调用端处理完响应消息之后，一次RPC调用就完成了。那是不是说我们只要能够让调用端在没有服务提供方提供接口的情况下，仍然能够向服务端发送正确的请求消息，就能够解决这个问题了呢？</p><p>没错，只要调用端将服务端需要知道的信息，如接口名、业务分组名、方法名以及参数信息等封装成请求消息发送给服务端，服务端就能够解析并处理这条请求消息，这样问题就解决了。过程如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/a3/89/a3c5ddba4960645b77d73e503da34b89.jpg?wh=2794*801" alt="" title="示意图"></p><p>现在我们已经清楚了解决问题的关键，但RPC的调用端向服务端发送消息是需要以动态代理作为入口的，我们现在得继续想办法让调用端发送我刚才讲过的那条请求消息。</p><p>我们可以定义一个统一的接口（GenericService），调用端在创建GenericService代理时指定真正需要调用的接口的接口名以及分组名，而GenericService接口的$invoke方法的入参就是方法名以及参数信息。</p><p>这样我们传递给服务端所需要的所有信息，包括接口名、业务分组名、方法名以及参数信息等都可以通过调用GenericService代理的$invoke方法来传递。具体的接口定义如下：</p><pre><code>class GenericService {

  Object $invoke(String methodName, String[] paramTypes, Object[] params);
  
}
</code></pre><p>这个通过统一的GenericService接口类生成的动态代理，来实现在没有接口的情况下进行RPC调用的功能，我们就称之为泛化调用。</p><p>通过泛化调用功能，我们可以解决在没有服务提供方提供接口API的情况下进行RPC调用，那么这个功能是否就完美了呢？</p><p>回顾下<a href="https://time.geekbang.org/column/article/216803">[第 17 讲]</a> 我过的内容，RPC框架可以通过异步的方式提升吞吐量，还有如何实现全异步的RPC框架，其关键点就是RPC框架对CompletableFuture的支持，那么我们的泛化调用是否也可以支持异步呢？</p><p>当然可以。我们可以给GenericService接口再添加一个异步方法$asyncInvoke，方法的返回值就是CompletableFuture，GenericService接口的具体定义如下：</p><pre><code>class GenericService {

  Object $invoke(String methodName, String[] paramTypes, Object[] params);

  CompletableFuture&lt;Object&gt; $asyncInvoke(String methodName, String[] paramTypes, Object[] params);

}
</code></pre><p>学到这里相信你已经对泛化调用的功能有一定的了解了，那你有没有想过这样一个问题？在没有服务提供方提供接口API的情况下，我们可以用泛化调用的方式实现RPC调用，但是如果没有服务提供方提供接口API，我们就没法得到入参以及返回值的Class类，也就不能对入参对象进行正常的序列化。这时我们会面临两个问题：</p><p><strong>问题1：</strong>调用端不能对入参对象进行正常的序列化，那调用端、服务端在接收到请求消息后，入参对象又该如何序列化与反序列化呢？</p><p>回想下<a href="https://time.geekbang.org/column/article/207137">[第 07 讲]</a>，在这一讲中我讲解了如何设计可扩展的RPC框架，我们通过插件体系来提高RPC框架的可扩展性，在RPC框架的整体架构中就包括了序列化插件，我们可以为泛化调用提供专属的序列化插件，通过这个插件，解决泛化调用中的序列化与反序列化问题。</p><p><strong>问题2：</strong>调用端的入参对象（params）与返回值应该是什么类型呢？</p><p>在服务提供方提供的接口API中，被调用的方法的入参类型是一个对象，那么使用泛化调用功能的调用端，可以使用Map类型的对象，之后通过泛化调用专属的序列化方式对这个Map对象进行序列化，服务端收到消息后，再通过泛化调用专属的序列化方式将其反序列成对象。</p><h2>总结</h2><p>今天我们主要讲解了如何在没有接口的情况下进行RPC调用，泛化调用的功能可以实现这一目的。</p><p>这个功能的实现原理，就是RPC框架提供统一的泛化调用接口（GenericService），调用端在创建GenericService代理时指定真正需要调用的接口的接口名以及分组名，通过调用GenericService代理的$invoke方法将服务端所需要的所有信息，包括接口名、业务分组名、方法名以及参数信息等封装成请求消息，发送给服务端，实现在没有接口的情况下进行RPC调用的功能。</p><p>而通过泛化调用的方式发起调用，由于调用端没有服务端提供方提供的接口API，不能正常地进行序列化与反序列化，我们可以为泛化调用提供专属的序列化插件，来解决实际问题。</p><h2>课后思考</h2><p>在讲解泛化调用时，我讲到服务端在收到调用端通过泛化调用的方式发送过来的请求时，会使用泛化调用专属的序列化插件实现对其进行反序列化，那么服务端是如何判定这个请求消息是通过泛化调用的方式发送过来的消息呢？</p><p>欢迎留言和我分享你的答案，也欢迎你把文章分享给你的朋友，邀请他加入学习。我们下节课再见！</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1c/ab/72/c3a5eff3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Reason</span>
  </div>
  <div class="_2_QraFYR_0">能想到两种解决方法：<br>1. 通过泛化调用的接口名或者方法名，判断是否是泛化请求<br>2. 客户端发起调用时一定知道请求是泛化请求，因此可以在请求信息的附加字段中标识该请求为泛化请求</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-13 10:38:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/67/12/51b78d88.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>🐠</span>
  </div>
  <div class="_2_QraFYR_0">对于Dubbo来说，根据org.apache.dubbo.config.AbstractReferenceConfig#generic 字段来标识该引用是否为泛化调用，所以根据该字段来使用对应的序列化插件就可以了，而且Dubbo对于POJO参数和返回值，统一都是用Map来接收的，可以看看官方文档https:&#47;&#47;dubbo.apache.org&#47;zh&#47;docs&#47;v2.7&#47;user&#47;examples&#47;generic-reference&#47;</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-20 22:32:59</div>
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
  <div class="_2_QraFYR_0">没有接口API，调用者怎么使用插件完成序列化&#47;反序列化呢，总得知道序列化反序列化的目标Class才能进行吧</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对于consumer来说，返回得到的值肯定类似一个map</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-14 08:16:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/26/38/ef063dc2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Darren</span>
  </div>
  <div class="_2_QraFYR_0">方法名称需要特殊处理一下，参数不是很好，因为有的方法是没有参数的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是一种方案</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-13 11:02:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/f8/38/5a857c96.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ivan</span>
  </div>
  <div class="_2_QraFYR_0">消息协议中定义泛化标示</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-11 00:15:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/24/da/54/9a419711.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>海风极客</span>
  </div>
  <div class="_2_QraFYR_0">grpc的unknownservice是不是个这个差不多呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-31 08:08:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/81/66/31c7969e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>随风而逝</span>
  </div>
  <div class="_2_QraFYR_0">也可以在定义传输协议时增加一位用来标识数据的请求类型</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-28 12:05:48</div>
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
  <div class="_2_QraFYR_0">泛化序列化的具体细节怎么实现呢？二进制流总要变成对应的对象吧？还是需要一个对象来承载调用入参的，使用map？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-17 10:45:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/17/27/ec30d30a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Jxin</span>
  </div>
  <div class="_2_QraFYR_0">1.在原有的请求处理器里面加判断逻辑（不合理）。<br>2.单独为泛化请求添加请求处理器，在请求解析完，根据解析出来的数据决定走哪个处理器（泛化接口处理器，常规接口处理器）（合理）。<br><br><br>泛化请求不属于常规接口请求，它与常规请求应是平级的两种请求类型，故而认为应该将两种数据流分离，在编码层面就做好隔离，也为以后差异化迭代埋好扩展点。既增强语义准确性，也做前瞻性的需求预留。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 分开处理后代码是不是有的复杂啊？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-14 18:13:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1c/a7/90/9a0da433.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小哇</span>
  </div>
  <div class="_2_QraFYR_0">老师好，我们这边是服务方也使用map做入参，然后在方法里再转成对象，就没有序列化问题，但感觉冗余。看到今天老师说的，意思是不是专属的序列化方式可以在调用方法前反序列为对象。这样做服务方的方法就可以使用对象入参而不用map做入参？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: map更多是扩展来用吧，否则强类型的语言就没有意义了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-14 17:31:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1c/ab/72/c3a5eff3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Reason</span>
  </div>
  <div class="_2_QraFYR_0">有个问题请教下老师，希望可以得到解答：<br>文中说泛化调用用于统一测试平台时，可以不需要修改平台代码重新上线。不修改平台代码重新上线，怎么编写相应的泛化调用代码呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 泛化调用的参数是动态的，不跟接口定义绑定</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-13 10:41:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/0f/bf/ee93c4cf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>雨霖铃声声慢</span>
  </div>
  <div class="_2_QraFYR_0">这个泛化调用好抽象，对于区分泛化调用和其他调用，一个是泛化调用的函数名和第一参数方法名，可以在这两个上做文章来分区泛化方法和其他方法</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 理解接口定义在rpc里面的作用后，理解起来就比较简单了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-13 09:02:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/01/37/12e4c9c9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>高源</span>
  </div>
  <div class="_2_QraFYR_0">老师理论讲的很到位，深入理解还得去看代码，老师能否推荐一下学习rpc地方，理论听的明白，动手还差很多啊😊</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-13 05:51:53</div>
  </div>
</div>
</div>
</li>
</ul>