<audio title="18 _ 如何通过gRPC实现高效远程过程调用？" src="https://static001.geekbang.org/resource/audio/e8/e8/e894713ff9692c1c167f23df35b9b8e8.mp3" controls="controls"></audio> 
<p>你好，我是陶辉。</p><p>这一讲我们将以一个实战案例，基于前两讲提到的HTTP/2和ProtoBuf协议，看看gRPC如何将结构化消息编码为网络报文。</p><p>直接操作网络协议编程，容易让业务开发过程陷入复杂的网络处理细节。RPC框架以编程语言中的本地函数调用形式，向应用开发者提供网络访问能力，这既封装了消息的编解码，也通过线程模型封装了多路复用，对业务开发很友好。</p><p>其中，Google推出的gRPC是性能最好的RPC框架之一，它支持Java、JavaScript、Python、GoLang、C++、Object-C、Android、Ruby等多种编程语言，还支持安全验证等特性，得到了广泛的应用，比如微服务中的Envoy、分布式机器学习中的TensorFlow，甚至华为去年推出重构互联网的New IP技术，都使用了gRPC框架。</p><p>然而，网络上教你使用gRPC框架的教程很多，却很少去谈gRPC是如何编码消息的。这样，一旦在大型分布式系统中出现疑难杂症，需要通过网络报文去定位问题发生在哪个系统、主机、进程中时，你就会毫无头绪。即使我们掌握了HTTP/2和Protobuf协议，但若不清楚gRPC的编码规则，还是无法分析抓取到的gRPC报文。而且，gRPC支持单向、双向的流式RPC调用，编程相对复杂一些，定位流式RPC调用引发的bug时，更需要我们掌握gRPC的编码原理。</p><!-- [[[read_end]]] --><p>这一讲，我就将以gRPC官方提供的example：<a href="https://github.com/grpc/grpc/tree/master/examples/python/data_transmission">data_transmisstion</a> 为例，介绍gRPC的编码流程。在这一过程中，会顺带回顾HTTP/2和Protobuf协议，加深你对它们的理解。虽然这个示例使用的是Python语言，但基于gRPC框架，你可以轻松地将它们转换为其他编程语言。</p><h2>如何使用gRPC框架实现远程调用？</h2><p>我们先来简单地看下gRPC框架到底是什么。RPC的全称是Remote Procedure Call，即远程过程调用，它通过本地函数调用，封装了跨网络、跨平台、跨语言的服务访问，大大简化了应用层编程。其中，函数的入参是请求，而函数的返回值则是响应。</p><p>gRPC就是一种RPC框架，在你定义好消息格式后，针对你选择的编程语言，gRPC为客户端生成发起RPC请求的Stub类，以及为服务器生成处理RPC请求的Service类（服务器只需要继承、实现类中处理请求的函数即可）。如下图所示，很明显，gRPC主要服务于面向对象的编程语言。</p><p><img src="https://static001.geekbang.org/resource/image/c2/a1/c20e6974a05b5e71823aec618fc824a1.jpg?wh=1682*1182" alt=""></p><p>gRPC支持QUIC、HTTP/1等多种协议，但鉴于HTTP/2协议性能好，应用场景又广泛，因此HTTP/2是gRPC的默认传输协议。gRPC也支持JSON编码格式，但在忽略编码细节的RPC调用中，高效的Protobuf才是最佳选择！因此，这一讲仅基于HTTP/2和Protobuf，介绍gRPC的用法。</p><p>gRPC可以简单地分为三层，包括底层的数据传输层，中间的框架层（框架层又包括C语言实现的核心功能，以及上层的编程语言框架），以及最上层由框架层自动生成的Stub和Service类，如下图所示：</p><p><a href="https://platformlab.stanford.edu/Seminar%20Talks/gRPC.pdf"><img src="https://static001.geekbang.org/resource/image/2a/4a/2a3f82f3eaabd440bf1ee449e532944a.png?wh=1380*557" alt="" title="图片来源：https://platformlab.stanford.edu/Seminar%20Talks/gRPC.pdf"></a></p><p>接下来我们以官网上的<a href="https://github.com/grpc/grpc/tree/master/examples/python/data_transmission">data_transmisstion</a> 为例，先看看如何使用gRPC。</p><p>构建Python语言的gRPC环境很简单，你可以参考官网上的<a href="https://grpc.io/docs/quickstart/python/">QuickStart</a>。</p><p>使用gRPC前，先要根据Protobuf语法，编写定义消息格式的proto文件。在这个例子中只有1种请求和1种响应，且它们很相似，各含有1个整型数字和1个字符串，如下所示：</p><pre><code>package demo;

message Request {
    int64 client_id = 1;
    string request_data = 2;
} 

message Response {
    int64 server_id = 1;
    string response_data = 2;
}
</code></pre><p>请注意，这里的包名demo以及字段序号1、2，都与后续的gRPC报文分析相关。</p><p>接着定义service，所有的RPC方法都要放置在service中，这里将它取名为GRPCDemo。GRPCDemo中有4个方法，后面3个流式访问的例子我们呆会再谈，先来看简单的一元访问模式SimpleMethod 方法，它定义了1个请求对应1个响应的访问形式。其中，SimpleMethod的参数Request是请求，返回值Response是响应。注意，分析报文时会用到这里的类名GRPCDemo以及方法名SimpleMethod。</p><pre><code>service GRPCDemo {
    rpc SimpleMethod (Request) returns (Response);
}
</code></pre><p>用grpc_tools中的protoc命令，就可以针对刚刚定义的service，生成含有GRPCDemoStub类和GRPCDemoServicer类的demo_pb2_grpc.py文件（实际上还包括完成Protobuf编解码的demo_pb2.py），应用层将使用这两个类完成RPC访问。我简化了官网上的Python客户端代码，如下所示：</p><pre><code>with grpc.insecure_channel(&quot;localhost:23333&quot;) as channel:
    stub = demo_pb2_grpc.GRPCDemoStub(channel)
    request = demo_pb2.Request(client_id=1,
            request_data=&quot;called by Python client&quot;)
    response = stub.SimpleMethod(request)
</code></pre><p>示例中客户端与服务器都在同一台机器上，通过23333端口访问。客户端通过Stub对象的SimpleMethod方法完成了RPC访问。而服务器端的实现也很简单，只需要实现GRPCDemoServicer父类的SimpleMethod方法，返回response响应即可：</p><pre><code>class DemoServer(demo_pb2_grpc.GRPCDemoServicer):
    def SimpleMethod(self, request, context):
        response = demo_pb2.Response(
            server_id=1,
            response_data=&quot;Python server SimpleMethod Ok!!!!&quot;)
        return response
</code></pre><p>可见，gRPC的开发效率非常高！接下来我们分析这次RPC调用中，消息是怎样编码的。</p><h2>gRPC消息是如何编码的？</h2><p><strong>定位复杂的网络问题，都需要抓取、分析网络报文。</strong>如果你在Windows上抓取网络报文，可以使用Wireshark工具（可参考<a href="https://time.geekbang.org/course/detail/175-100973">《Web协议详解与抓包实战》第37课</a>），如果在Linux上抓包可以使用tcpdump工具（可参考<a href="https://time.geekbang.org/course/detail/175-118169">第87课</a>）。当然，你也可以从<a href="https://github.com/russelltao/geektime_distrib_perf/blob/master/18-gRPC/data_transmission.pkt">这里</a>下载我抓取好的网络报文，用Wireshark打开它。需要注意，23333不是HTTP常用的80或者443端口，所以Wireshark默认不会把它解析为HTTP/2协议。你需要鼠标右键点击报文，选择“解码为”（Decode as），将23333端口的报文设置为HTTP/2解码器，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/74/f7/743e5038dc22676a0d48b56c453c1af7.png?wh=1241*425" alt=""></p><p>图中蓝色方框中，TCP连接的建立过程请参见<a href="https://time.geekbang.org/column/article/237612">[第9讲]</a>，而HTTP/2会话的建立可参见<a href="https://time.geekbang.org/course/detail/175-105608">《Web协议详解与抓包实战》第52课</a>（还是比较简单的，如果你都清楚就可以直接略过）。我们重点看红色方框中的gRPC请求与响应，点开请求，可以看到下图中的信息：</p><p><img src="https://static001.geekbang.org/resource/image/ba/41/ba4d9e9a6ce212e94a4ced829eeeca41.png?wh=906*844" alt=""></p><p>先来分析蓝色方框中的HTTP/2头部。请求中有2个关键的HTTP头部，path和content-type，它们决定了RPC方法和具体的消息编码格式。path的值为“/demo.GRPCDemo/SimpleMethod”，通过“/包名.服务名/方法名”的形式确定了RPC方法。content-type的值为“application/grpc”，确定消息编码使用Protobuf格式。如果你对其他头部的含义感兴趣，可以看下这个<a href="https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md">文档</a>，注意这里使用了ABNF元数据定义语言（如果你还不了解ABNF，可以看下<a href="https://time.geekbang.org/course/detail/175-93589">《Web协议详解与抓包实战》第4课</a>）。</p><p>HTTP/2包体并不会直接存放Protobuf消息，而是先要添加5个字节的Length-Prefixed Message头部，其中用4个字节明确Protobuf消息的长度（1个字节表示消息是否做过压缩），即上图中的桔色方框。为什么要多此一举呢？这是因为，gRPC支持流式消息，即在HTTP/2的1条Stream中，通过DATA帧发送多个gRPC消息，而Length-Prefixed Message就可以将不同的消息分离开。关于流式消息，我们在介绍完一元模式后，再加以分析。</p><p>最后分析Protobuf消息，这里仅以client_id字段为例，对上一讲的内容做个回顾。在proto文件中client_id字段的序号为1，因此首字节00001000中前5位表示序号为1的client_id字段，后3位表示字段的值类型是varint格式的数字，因此随后的字节00000001表示字段值为1。序号为2的request_data字段请你结合上一讲的内容，试着做一下解析，看看字符串“called by Python client”是怎样编码的。</p><p>再来看服务器发回的响应，点开Wireshark中的响应报文后如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/b8/cf/b8e71a1b956286b2def457c2fae78bcf.png?wh=962*723" alt=""></p><p>其中DATA帧同样包括Length-Prefixed Message和Protobuf，与RPC请求如出一辙，这里就不再赘述了，我们重点看下HTTP/2头部。你可能留意到，响应头部被拆成了2个部分，其中grpc-status和grpc-message是在DATA帧后发送的，这样就允许服务器在发送完消息后再给出错误码。关于gRPC的官方错误码以及message描述信息是如何取值的，你可以参考<a href="https://github.com/grpc/grpc/blob/master/doc/statuscodes.md">这个文档。</a></p><p>这种将部分HTTP头部放在包体后发送的技术叫做Trailer，<a href="https://tools.ietf.org/html/rfc7230#page-39">RFC7230文档</a>对此有详细的介绍。其中，RPC请求中的TE: trailers头部，就说明客户端支持Trailer头部。在RPC响应中，grpc-status头部都会放在最后发送，因此它的帧flags的EndStream标志位为1。</p><p>可以看到，gRPC中的HTTP头部与普通的HTTP请求完全一致，因此，它兼容当下互联网中各种七层负载均衡，这使得gRPC可以轻松地跨越公网使用。</p><h2>gRPC流模式的协议编码</h2><p>说完一元模式，我们再来看流模式RPC调用的编码方式。</p><p>所谓流模式，是指RPC通讯的一方可以在1次RPC调用中，持续不断地发送消息，这对订阅、推送等场景很有用。流模式共有3种类型，包括客户端流模式、服务器端流模式，以及两端双向流模式。在<a href="https://github.com/grpc/grpc/tree/master/examples/python/data_transmission">data_transmisstion</a> 官方示例中，对这3种流模式都定义了RPC方法，如下所示：</p><pre><code>service GRPCDemo {
    rpc ClientStreamingMethod (stream Request) returns Response);
    
    rpc ServerStreamingMethod (Request) returns (stream Response);

    rpc BidirectionalStreamingMethod (stream Request) returns (stream Response);
}
</code></pre><p>不同的编程语言处理流模式的代码很不一样，这里就不一一列举了，但通讯层的流模式消息编码是一样的，而且很简单。这是因为，HTTP/2协议中每个Stream就是天然的1次RPC请求，每个RPC消息又已经通过Length-Prefixed Message头部确立了边界，这样，在Stream中连续地发送多个DATA帧，就可以实现流模式RPC。我画了一张示意图，你可以对照它理解抓取到的流模式报文。</p><p><img src="https://static001.geekbang.org/resource/image/4b/e0/4b1b9301b5cbf0e0544e522c2a8133e0.jpg?wh=1538*1400" alt=""></p><h2>小结</h2><p>这一讲介绍了gRPC怎样使用HTTP/2和Protobuf协议编码消息。</p><p>在定义好消息格式，以及service类中的RPC方法后，gRPC框架可以为编程语言生成Stub和Service类，而类中的方法就封装了网络调用，其中方法的参数是请求，而方法的返回值则是响应。</p><p>发起RPC调用后，我们可以这么分析抓取到的网络报文。首先，分析应用层最外层的HTTP/2帧，根据Stream ID找出一次RPC调用。客户端HTTP头部的path字段指明了service和RPC方法名，而content-type则指明了消息的编码格式。服务器端的HTTP头部被分成2次发送，其中DATA帧发送完毕后，才会发送grpc-status头部，这样可以明确最终的错误码。</p><p>其次，分析包体时，可以通过Stream中Length-Prefixed Message头部，确认DATA帧中含有多少个消息，因此可以确定这是一元模式还是流式调用。在Length-Prefixed Message头部后，则是Protobuf消息，按照上一讲的内容进行分析即可。</p><h2>思考题</h2><p>最后，留给你一道练习题。gRPC默认并不会压缩字符串，你可以通过在获取channel对象时加入grpc.default_compression_algorithm参数的形式，要求gRPC压缩消息，此时Length-Prefixed Message中1个字节的压缩位将会由0变为1。你可以观察下执行压缩后的gRPC消息有何不同，欢迎你在留言区与大家一起探讨。</p><p>感谢阅读，如果你觉得这节课对你有一些启发，也欢迎把它分享给你的朋友。</p>
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
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83eovXluTbBvyjZQ5zY8e3AZLONj6Qx5mcF4G7ZWYVbeicDzOlakFj4dKh6jCFHfqXvrLccuiaxYicmTxg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>远方的风</span>
  </div>
  <div class="_2_QraFYR_0">老师的广度和深度是如何练成的呢，是看书，还是实际工作中都有涉及？一开始搞web开发的，对底层这些研究的就没这么深了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你好远方的风，不用着急，能问出这问题，就表示你在朝这个方向进发，它需要时间。<br>我建议这么加快速度：在实践中遇到问题时，多问自己几个为什么，就会顺着捋下来。在这个过程中，有些超过自己边界的问题，必须要增加广度，才能搞明白“为什么”。<br>比如做gRPC应用开发时，如果想搞明白gRPC为什么可以生成stub代码，就得先增加广度去学一学编译原理</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-15 13:49:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/a9/cb/a431bde5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>木头发芽</span>
  </div>
  <div class="_2_QraFYR_0">公司所有项目(小程序端除外)都选型用了gRPC,不止微服务之间还包括了移动端,web端到服务器的通信.<br>18年开始使用时还遇到不小的阻力,特别是客户端离开了http+json的舒适区转到gRpc+pb 要研究怎么使用.也遇到了不少坑比如:<br>iOS的protoc编出的代码无法建立tls连接一致卡在证书校验步骤.<br>grpc-web在浏览器http1.1转http2的问题.<br>envoy做代理和负载均衡导致大量连接处于closed状态的问题.<br>在走完一遍后,后面就用起来非常的爽了.<br><br>不过流的用法还没实践,通过这节课又加深了一些,找个版本使用一下流方式的接口.</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢木头发芽同学的实战分享！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-08 11:56:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/56/75/6bf38a1e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>坤哥</span>
  </div>
  <div class="_2_QraFYR_0">陶哥，你的技术真不简单了，买了第二门web协议讲课程，希望能跟随你的步伐成长，你比我老大更牛</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢坤哥^_^</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-08 18:27:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/64/9b/d1ab239e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>J.Smile</span>
  </div>
  <div class="_2_QraFYR_0">“分析包体时，可以通过 Stream 中 Length-Prefixed Message 头部，确认 DATA 帧中含有多少个消息，因此可以确定这是一元模式还是流式调用”<br>——————————————————<br>老师好，Length-Prefixed Message 头部不是只有长度和是否压缩吗？怎么能确认有多少个消息的？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-07 09:07:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/7f/91/962eba1a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>唐朝首都</span>
  </div>
  <div class="_2_QraFYR_0">基本原理+工具使用，理论结合实践才能把这些协议搞清楚呀！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 没错！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-16 08:59:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/39/fa/a7edbc72.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>安排</span>
  </div>
  <div class="_2_QraFYR_0">client流模式中是不是每一个rpc请求消息的第一个data帧中才有Length-Prefixed Message ，然后下一个data帧只有protobuf数据，直到这个rpc请求消息发完。然后同一个stream上的下一个rpc消息的第一个data帧再加入Length-Prefixed Message 。以此类推。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-15 08:15:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/24/ee/46/7d65ae37.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>木几丶</span>
  </div>
  <div class="_2_QraFYR_0">老师，从抓包看，grpc三次握手后升级为HTTP2协议，为什么既没有基于TCP的101状态码升级，也没有基于TLS握手升级呢？直接就开始发送MAGIC帧了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-23 19:24:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c6/73/abb7bfe3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>疯琴</span>
  </div>
  <div class="_2_QraFYR_0">老师，我又试了一下，如果消息内容比较大，压缩会缩小尺寸，如果消息内容比较小，压缩反而会增大尺寸。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-09 22:12:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c6/73/abb7bfe3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>疯琴</span>
  </div>
  <div class="_2_QraFYR_0">老师，这两节课受益匪浅，自己实践了一下。我的消息内容只有一个7个字节的字符串，加上序号和类型是9个字节，如果不压缩，消息体就是9个字节，如果压缩，在grpc消息中有这么一条：Message-encoded entity body (gzip): 33 bytes -&gt; 9 bytes，消息体还是一样的，这33字节是怎么来的呢？为什么9个字节的body也没有变化呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-09 21:57:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/64/9b/d1ab239e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>J.Smile</span>
  </div>
  <div class="_2_QraFYR_0">记得以前看到说， 帧的最后4个字节就是StreamId，接收方通过StreamId从乱序的帧中识别出相同StreamId的帧序列，按照顺序组装起来就实现了虚拟的“流”。<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-07 09:12:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/f9/30/54c71bf9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>不会飞的海燕</span>
  </div>
  <div class="_2_QraFYR_0">老师有grpc方面的优化，深度定制案例 介绍吗</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-27 15:35:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/63/5e/799cd6dc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>🎧重返归途</span>
  </div>
  <div class="_2_QraFYR_0">protobuf是属于协议还是文件格式，看到有资料说把protobuf和json和xml作比较？合适么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: protobuf是一种编码格式，从这种角度上，把它与json、XML作比较是合适的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-21 16:01:03</div>
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
  <div class="_2_QraFYR_0">从http1.1 升级到 http2.0 要哪些地方改造呢，就改个versionCode 吗</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-15 16:16:19</div>
  </div>
</div>
</div>
</li>
</ul>