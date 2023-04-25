<audio title="第36讲 _ 跨语言类RPC协议：交流之前，双方先来个专业术语表" src="https://static001.geekbang.org/resource/audio/fc/ce/fc48ac4a5474ab681c2d9e50118924ce.mp3" controls="controls"></audio> 
<p>到目前为止，咱们讲了四种RPC，分别是ONC RPC、基于XML的SOAP、基于JSON的RESTful和Hessian2。</p><p>通过学习，我们知道，二进制的传输性能好，文本类的传输性能差一些；二进制的难以跨语言，文本类的可以跨语言；要写协议文件的严谨一些，不写协议文件的灵活一些。虽然都有服务发现机制，有的可以进行服务治理，有的则没有。</p><p>我们也看到了RPC从最初的客户端服务器模式，最终演进到微服务。对于RPC框架的要求越来越多了，具体有哪些要求呢？</p><ul>
<li>
<p>首先，传输性能很重要。因为服务之间的调用如此频繁了，还是二进制的越快越好。</p>
</li>
<li>
<p>其次，跨语言很重要。因为服务多了，什么语言写成的都有，而且不同的场景适宜用不同的语言，不能一个语言走到底。</p>
</li>
<li>
<p>最好既严谨又灵活，添加个字段不用重新编译和发布程序。</p>
</li>
<li>
<p>最好既有服务发现，也有服务治理，就像Dubbo和Spring Cloud一样。</p>
</li>
</ul><h2>Protocol Buffers</h2><p>这是要多快好省地建设社会主义啊。理想还是要有的嘛，这里我就来介绍一个向“理想”迈进的GRPC。</p><p>GRPC首先满足二进制和跨语言这两条，二进制说明压缩效率高，跨语言说明更灵活。但是又是二进制，又是跨语言，这就相当于两个人沟通，你不但说方言，还说缩略语，人家怎么听懂呢？所以，最好双方弄一个协议约定文件，里面规定好双方沟通的专业术语，这样沟通就顺畅多了。</p><!-- [[[read_end]]] --><p>对于GRPC来讲，二进制序列化协议是Protocol Buffers。首先，需要定义一个协议文件.proto。</p><p>我们还看买极客时间专栏的这个例子。</p><pre><code>syntax = “proto3”;
package com.geektime.grpc
option java_package = “com.geektime.grpc”;
message Order {
  required string date = 1;
  required string classname = 2;
  required string author = 3;
  required int price = 4;
}

message OrderResponse {
  required string message = 1;
}

service PurchaseOrder {
  rpc Purchase (Order) returns (OrderResponse) {}
}
</code></pre><p>在这个协议文件中，我们首先指定使用proto3的语法，然后我们使用Protocol Buffers的语法，定义两个消息的类型，一个是发出去的参数，一个是返回的结果。里面的每一个字段，例如date、classname、author、price都有唯一的一个数字标识，这样在压缩的时候，就不用传输字段名称了，只传输这个数字标识就行了，能节省很多空间。</p><p>最后定义一个Service，里面会有一个RPC调用的声明。</p><p>无论使用什么语言，都有相应的工具生成客户端和服务端的Stub程序，这样客户端就可以像调用本地一样，调用远程的服务了。</p><h2>协议约定问题</h2><p>Protocol Buffers是一款压缩效率极高的序列化协议，有很多设计精巧的序列化方法。</p><p>对于int类型32位的，一般都需要4个Byte进行存储。在Protocol Buffers中，使用的是变长整数的形式。对于每一个Byte的8位，最高位都有特殊的含义。</p><p>如果该位为 1，表示这个数字没完，后续的Byte也属于这个数字；如果该位为 0，则这个数字到此结束。其他的7个Bit才是用来表示数字的内容。因此，小于128的数字都可以用一个Byte表示；大于128的数字，比如130，会用两个字节来表示。</p><p>对于每一个字段，使用的是TLV（Tag，Length，Value）的存储办法。</p><p>其中Tag = (field_num &lt;&lt; 3) | wire_type。field_num就是在proto文件中，给每个字段指定唯一的数字标识，而wire_type用于标识后面的数据类型。</p><p><img src="https://static001.geekbang.org/resource/image/a6/10/a66aa9ca6c6575f4b335881ae786ba10.jpg?wh=1220*540" alt=""></p><p>例如，对于string author = 3，在这里field_num为3，string的wire_type为2，于是 (field_num &lt;&lt; 3) | wire_type = (11000) | 10 = 11010 = 26；接下来是Length，最后是Value为“liuchao”，如果使用UTF-8编码，长度为7个字符，因而Length为7。</p><p>可见，在序列化效率方面，Protocol Buffers简直做到了极致。</p><p>在灵活性方面，这种基于协议文件的二进制压缩协议往往存在更新不方便的问题。例如，客户端和服务器因为需求的改变需要添加或者删除字段。</p><p>这一点上，Protocol Buffers考虑了兼容性。在上面的协议文件中，每一个字段都有修饰符。比如：</p><ul>
<li>
<p>required：这个值不能为空，一定要有这么一个字段出现；</p>
</li>
<li>
<p>optional：可选字段，可以设置，也可以不设置，如果不设置，则使用默认值；</p>
</li>
<li>
<p>repeated：可以重复0到多次。</p>
</li>
</ul><p>如果我们想修改协议文件，对于赋给某个标签的数字，例如string author=3，这个就不要改变了，改变了就不认了；也不要添加或者删除required字段，因为解析的时候，发现没有这个字段就会报错。对于optional和repeated字段，可以删除，也可以添加。这就给了客户端和服务端升级的可能性。</p><p>例如，我们在协议里面新增一个string recommended字段，表示这个课程是谁推荐的，就将这个字段设置为optional。我们可以先升级服务端，当客户端发过来消息的时候，是没有这个值的，将它设置为一个默认值。我们也可以先升级客户端，当客户端发过来消息的时候，是有这个值的，那它将被服务端忽略。</p><p>至此，我们解决了协议约定的问题。</p><h2>网络传输问题</h2><p>接下来，我们来看网络传输的问题。</p><p>如果是Java技术栈，GRPC的客户端和服务器之间通过Netty Channel作为数据通道，每个请求都被封装成HTTP  2.0的Stream。</p><p>Netty是一个高效的基于异步IO的网络传输框架，这个上一节我们已经介绍过了。HTTP  2.0在<a href="https://time.geekbang.org/column/article/9410">第14讲</a>，我们也介绍过。HTTP 2.0协议将一个TCP的连接，切分成多个流，每个流都有自己的ID，而且流是有优先级的。流可以是客户端发往服务端，也可以是服务端发往客户端。它其实只是一个虚拟的通道。</p><p>HTTP 2.0还将所有的传输信息分割为更小的消息和帧，并对它们采用二进制格式编码。</p><p>通过这两种机制，HTTP 2.0的客户端可以将多个请求分到不同的流中，然后将请求内容拆成帧，进行二进制传输。这些帧可以打散乱序发送， 然后根据每个帧首部的流标识符重新组装，并且可以根据优先级，决定优先处理哪个流的数据。</p><p><img src="https://static001.geekbang.org/resource/image/03/dd/03d4a216c024a9e761ed43c6787bf7dd.jpg?wh=1357*687" alt=""></p><p>由于基于HTTP 2.0，GRPC和其他的RPC不同，可以定义四种服务方法。</p><p>第一种，也是最常用的方式是<strong>单向RPC</strong>，即客户端发送一个请求给服务端，从服务端获取一个应答，就像一次普通的函数调用。</p><pre><code>rpc SayHello(HelloRequest) returns (HelloResponse){}
</code></pre><p>第二种方式是<strong>服务端流式RPC</strong>，即服务端返回的不是一个结果，而是一批。客户端发送一个请求给服务端，可获取一个数据流用来读取一系列消息。客户端从返回的数据流里一直读取，直到没有更多消息为止。</p><pre><code>rpc LotsOfReplies(HelloRequest) returns (stream HelloResponse){}
</code></pre><p>第三种方式为<strong>客户端流式RPC</strong>，也即客户端的请求不是一个，而是一批。客户端用提供的一个数据流写入并发送一系列消息给服务端。一旦客户端完成消息写入，就等待服务端读取这些消息并返回应答。</p><pre><code>rpc LotsOfGreetings(stream HelloRequest) returns (HelloResponse) {}
</code></pre><p>第四种方式为<strong>双向流式 RPC</strong>，即两边都可以分别通过一个读写数据流来发送一系列消息。这两个数据流操作是相互独立的，所以客户端和服务端能按其希望的任意顺序读写，服务端可以在写应答前等待所有的客户端消息，或者它可以先读一个消息再写一个消息，或者读写相结合的其他方式。每个数据流里消息的顺序会被保持。</p><pre><code>rpc BidiHello(stream HelloRequest) returns (stream HelloResponse){}
</code></pre><p>如果基于HTTP  2.0，客户端和服务器之间的交互方式要丰富得多，不仅可以单方向远程调用，还可以实现当服务端状态改变的时候，主动通知客户端。</p><p>至此，传输问题得到了解决。</p><h2>服务发现与治理问题</h2><p>最后是服务发现与服务治理的问题。</p><p>GRPC本身没有提供服务发现的机制，需要借助其他的组件，发现要访问的服务端，在多个服务端之间进行容错和负载均衡。</p><p>其实负载均衡本身比较简单，LVS、HAProxy、Nginx都可以做，关键问题是如何发现服务端，并根据服务端的变化，动态修改负载均衡器的配置。</p><p>在这里我们介绍一种对于GRPC支持比较好的负载均衡器Envoy。其实Envoy不仅仅是负载均衡器，它还是一个高性能的C++写的Proxy转发器，可以配置非常灵活的转发规则。</p><p>这些规则可以是静态的，放在配置文件中的，在启动的时候加载。要想重新加载，一般需要重新启动，但是Envoy支持热加载和热重启，这在一定程度上缓解了这个问题。</p><p>当然，最好的方式是将规则设置为动态的，放在统一的地方维护。这个统一的地方在Envoy眼中被称为服务发现（Discovery Service），过一段时间去这里拿一下配置，就修改了转发策略。</p><p>无论是静态的，还是动态的，在配置里面往往会配置四个东西。</p><p>第一个是listener。Envoy既然是Proxy，专门做转发，就得监听一个端口，接入请求，然后才能够根据策略转发，这个监听的端口就称为listener。</p><p>第二个是endpoint，是目标的IP地址和端口。这个是Proxy最终将请求转发到的地方。</p><p>第三个是cluster。一个cluster是具有完全相同行为的多个endpoint，也即如果有三个服务端在运行，就会有三个IP和端口，但是部署的是完全相同的三个服务，它们组成一个cluster，从cluster到endpoint的过程称为负载均衡，可以轮询。</p><p>第四个是route。有时候多个cluster具有类似的功能，但是是不同的版本号，可以通过route规则，选择将请求路由到某一个版本号，也即某一个cluster。</p><p>如果是静态的，则将后端的服务端的IP地址拿到，然后放在配置文件里面就可以了。</p><p>如果是动态的，就需要配置一个服务发现中心，这个服务发现中心要实现Envoy的API，Envoy可以主动去服务发现中心拉取转发策略。</p><p><img src="https://static001.geekbang.org/resource/image/d0/c7/d0e559056729a4618e76feb2150ef9c7.jpg?wh=1465*946" alt=""></p><p>看来，Envoy进程和服务发现中心之间要经常相互通信，互相推送数据，所以Envoy在控制面和服务发现中心沟通的时候，就可以使用GRPC，也就天然具备在用户面支撑GRPC的能力。</p><p>Envoy如果复杂的配置，都能干什么事呢？</p><p>一种常见的规则是<strong>配置路由策略</strong>。例如后端的服务有两个版本，可以通过配置Envoy的route，来设置两个版本之间，也即两个cluster之间的route规则，一个占99%的流量，一个占1%的流量。</p><p>另一种常见的规则就是<strong>负载均衡策略</strong>。对于一个cluster下的多个endpoint，可以配置负载均衡机制和健康检查机制，当服务端新增了一个，或者挂了一个，都能够及时配置Envoy，进行负载均衡。</p><p><img src="https://static001.geekbang.org/resource/image/72/32/722d4a1722e551be8c9583d6f5494032.jpg?wh=1516*1392" alt=""></p><p>所有这些节点的变化都会上传到注册中心，所有这些策略都可以通过注册中心进行下发，所以，更严格的意义上讲，注册中心可以称为<strong>注册治理中心</strong>。</p><p>Envoy这么牛，是不是能够将服务之间的相互调用全部由它代理？如果这样，服务也不用像Dubbo，或者Spring Cloud一样，自己感知到注册中心，自己注册，自己治理，对应用干预比较大。</p><p>如果我们的应用能够意识不到服务治理的存在，就可以直接进行GRPC的调用。</p><p>这就是未来服务治理的趋势<strong>Serivce Mesh</strong>，也即应用之间的相互调用全部由Envoy进行代理，服务之间的治理也被Envoy进行代理，完全将服务治理抽象出来，到平台层解决。</p><p><img src="https://static001.geekbang.org/resource/image/b5/5f/b5e7f54b07fea522ebc2469bbb081b5f.jpg?wh=1565*1312" alt=""><br>
至此RPC框架中有治理功能的Dubbo、Spring  Cloud、Service Mesh就聚齐了。</p><h2>小结</h2><p>好了，这一节就到这里了，我们来总结一下。</p><ul>
<li>
<p>GRPC是一种二进制，性能好，跨语言，还灵活，同时可以进行服务治理的多快好省的RPC框架，唯一不足就是还是要写协议文件。</p>
</li>
<li>
<p>GRPC序列化使用Protocol Buffers，网络传输使用HTTP  2.0，服务治理可以使用基于Envoy的Service  Mesh。</p>
</li>
</ul><p>最后，给你留一个思考题吧。</p><p>在讲述Service Mesh的时候，我们说了，希望Envoy能够在服务不感知的情况下，将服务之间的调用全部代理了，你知道怎么做到这一点吗？</p><p>我们《趣谈网络协议》专栏已经接近尾声了。你还记得专栏开始，我们讲过的那个“双十一”下单的故事吗？</p><p>下节开始，我会将这个过程涉及的网络协议细节，全部串联起来，给你还原一个完整的网络协议使用场景。信息量会很大，做好准备哦，我们下期见！</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/60/f8/3f0ba76d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>sam</span>
  </div>
  <div class="_2_QraFYR_0">我觉得是极客目前最好的专栏</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-08 17:36:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/20/5d/69170b96.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>灰灰</span>
  </div>
  <div class="_2_QraFYR_0">讲的太棒了，绝对是大师级人物。快结束了，意犹未尽，重新看一遍。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-08 10:23:10</div>
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
  <div class="_2_QraFYR_0">题目：在讲述 Service Mesh 的时候，我们说了，希望 Envoy 能够在服务不感知的情况下，将服务之间的调用全部代理了，你知道怎么做到这一点吗？<br>答：在Service Mesh模式中，每个服务都配备了一个代理sidecar（Envoy代理），用于服务之间的通信。这些代理通常与应用程序一起部署，代理不会被应用程序感知。这些代理组织起来形成服务网格。<br>Envoy是Service Mesh中一个非常优秀的sidecar的来源实现。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-08 09:47:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/d7/39/6698b6a9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Hector</span>
  </div>
  <div class="_2_QraFYR_0">Envoy就是lstio的核心支持，越来越觉得kubernetes真的是集大成者，服务治理，面向未来的声明式API架构，资源调度与编排，网络的多种优秀方案。怪不得可以在极端的时间里力压docker</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-07 14:24:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/07/40/04bee70c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jedi knight</span>
  </div>
  <div class="_2_QraFYR_0">学java的应该可以跳过spring cloud了，感觉envoy + grpc + kubernetes是趋势</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-09 17:53:28</div>
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
  <div class="_2_QraFYR_0">对思考题的解答<br>容器系统中，是通过 sidecar 模式来解决的，服务容器都是直接和 envoy sidecar 互通，envoy 的配置变化，网络拓扑的改变对服务容器都是不可感知的。service mesh 还更进一步的发展，istio 和 conduit，他们都是在 sidecar 基础上，又加了一个总的数据控制平面，来加强 service mesh 的掌控能力。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-09 09:08:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJNxMqDoNODfL1aFpkJboxlZPyFANAt9ovzksYtMUIGo3KbHr4Qy8j20Txrpiau6iaJGgAr1nCfpy9w/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>爱学习的老吴</span>
  </div>
  <div class="_2_QraFYR_0">老师，我有点搞不懂了，grpc用的是http2.0协议来传输的话，那它和普通的http服务有什么区别了吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-24 21:39:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/65/03/973b24ec.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>谢晋</span>
  </div>
  <div class="_2_QraFYR_0">我觉得是极客目前最好的专栏<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-12 23:22:44</div>
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
  <div class="_2_QraFYR_0">通过使用iptables程序配置内核中的netfilter，实现流量劫持转发，把指定入口流量都转发到envoy，出口流量也可以使用两样的方法实现</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-08 09:28:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/ae/e0/3e636955.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>李博越</span>
  </div>
  <div class="_2_QraFYR_0">想问一下老师，GRPC是否支持client通过代理服务器访问到server端？官网上找了半天没有找到相关答案，我这边现在的场景需要经典网络的管控节点访问到vpc中的服务节点，由于vpc中服务节点太多，不可能打太多的洞，所以想在vpc中部署nginx服务作为代理服务器，转发经典网络发来的所有请求到server端</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 试一下envoy</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-09 11:24:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/49/3c/5d54c510.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>incorporate</span>
  </div>
  <div class="_2_QraFYR_0">刘超老师，问个问题，如果一个服务在容器中，一个服务在物理机上，怎么进行rpc通信，此时容器是可以调用物理机的，但是物理机是无法调用容器中的服务了，例如dubbo，获取到的是容器中的ip，物理机调用的时候，通过容器中的ip是无法调用到容器中的，这个有什么解决办法么</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-11 10:20:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/12/b3/9e8f4b4f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>MoFanDon</span>
  </div>
  <div class="_2_QraFYR_0">已经完结这么久了，我才终于看到了这里。收获很多。感谢。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-08 01:02:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/a8/1b/ced1d171.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>空档滑行</span>
  </div>
  <div class="_2_QraFYR_0">其实也不是完全无感知，服务还是需要知道service mesh的存在，只是一般是sidebar方式的部署，每个服务只需要知道自己的enovy在哪里就可以了，所有网络交互通过它来转发</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-08 09:39:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/01/87/c45e7044.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>崔朝普🐳📡</span>
  </div>
  <div class="_2_QraFYR_0">赞，干货满满</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-08 09:00:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/7b/2a/7d8b5943.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>LH</span>
  </div>
  <div class="_2_QraFYR_0">envoy需要拦截所有进出服务的流量，可以通过配置iptable转发规则实现</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-29 23:51:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/1d/64/52a5863b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>大土豆</span>
  </div>
  <div class="_2_QraFYR_0">dubbo没法跨语言，是因为内置了netty吗？Hessian2协议本身是语言无关的。所以grpc选择了http2.0，就是为了跨语言吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-02 22:31:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/de/3c/b1fe1f52.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>wsjx</span>
  </div>
  <div class="_2_QraFYR_0">慢慢学</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-29 18:46:52</div>
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
  <div class="_2_QraFYR_0">问题中描述的“希望 Envoy 能够在服务不感知的情况下，将服务之间的调用全部代理了”不禁让我想起了网络中的路由器发现相关算法，参照Jay的评论，我想为每个服务配置一个代理(路由器)，组成服务网格(AS)，将RPC的相关操作独立成一层，让应用如TCP调用IP层般进行夸服务、语言的调用，以达到对应用层的透明。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-03 15:02:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/57/04/07eb8cbb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>hhq</span>
  </div>
  <div class="_2_QraFYR_0">对grpc有了基本的认识，包括协议定义，传输封装等。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-11 17:47:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/f7/17/f94e987f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Shawn.C</span>
  </div>
  <div class="_2_QraFYR_0">服务的注册不感知是因为使用了容器平台的发现能力。服务自身不感知，是因为envoy 作为sidecar 的方式劫持了网络流量。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-09 08:55:11</div>
  </div>
</div>
</div>
</li>
</ul>