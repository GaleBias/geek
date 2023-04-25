<audio title="06 _ RPC实战：剖析gRPC源码，动手实现一个完整的RPC" src="https://static001.geekbang.org/resource/audio/a0/cc/a0661ac1c3e1930a02f4c43bcb321ccc.mp3" controls="controls"></audio> 
<p>你好，我是何小锋。上一讲我分享了动态代理，其作用总结起来就是一句话：“我们可以通过动态代理技术，屏蔽 RPC 调用的细节，从而让使用者能够面向接口编程。”</p><p>到今天为止，我们已经把 RPC 通信过程中要用到的所有基础知识都讲了一遍，但这些内容多属于理论。<strong>这一讲我们就来实战一下，看看具体落实到代码上，我们应该怎么实现一个 RPC 框架？</strong></p><p>为了能让咱们快速达成共识，我选择剖析 gRPC 源码（源码地址：<a href="https://github.com/grpc/grpc-java">https://github.com/grpc/grpc-java</a>）。通过分析 gRPC 的通信过程，我们可以清楚地知道在 gRPC 里面这些知识点是怎么落地到具体代码上的。</p><p>gRPC 是由 Google 开发并且开源的一款高性能、跨语言的 RPC 框架，当前支持 C、Java 和 Go 等语言，当前 Java 版本最新 Release 版为 1.27.0。gRPC 有很多特点，比如跨语言，通信协议是基于标准的 HTTP/2 设计的，序列化支持 PB（Protocol Buffer）和 JSON，整个调用示例如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/86/0d/8671942cd89feea3a2544d3530da450d.jpg?wh=3183*1769" alt="" title="gRPC调用示例图"></p><p>如果你想快速地了解一个全新框架的工作原理，我个人认为最快的方式就是从使用示例开始，所以现在我们就以最简单的 HelloWord 为例开始了解。</p><p>在这个例子里面，我们会定义一个 say 方法，调用方通过 gRPC 调用服务提供方，然后服务提供方会返回一个字符串给调用方。</p><!-- [[[read_end]]] --><p>为了保证调用方和服务提供方能够正常通信，我们需要先约定一个通信过程中的契约，也就是我们在 Java  里面说的定义一个接口，这个接口里面只会包含一个 say 方法。在 gRPC 里面定义接口是通过写 Protocol Buffer 代码，从而把接口的定义信息通过 Protocol Buffer 语义表达出来。HelloWord 的 Protocol Buffer 代码如下所示：</p><pre><code>syntax = &quot;proto3&quot;;

option java_multiple_files = true;
option java_package = &quot;io.grpc.hello&quot;;
option java_outer_classname = &quot;HelloProto&quot;;
option objc_class_prefix = &quot;HLW&quot;;

package hello;

service HelloService{
rpc Say(HelloRequest) returns (HelloReply) {}
}

message HelloRequest {
string name = 1;
}

message HelloReply {
string message = 1;
}
</code></pre><p>有了这段代码，我们就可以为客户端和服务器端生成消息对象和 RPC 基础代码。我们可以利用 Protocol Buffer 的编译器 protoc，再配合 gRPC Java 插件（protoc-gen-grpc-java），通过命令行 protoc3 加上 plugin 和 proto 目录地址参数，我们就可以生成消息对象和 gRPC 通信所需要的基础代码。如果你的项目是 Maven 工程的话，你还可以直接选择使用 Maven  插件来生成同样的代码。</p><h2>发送原理</h2><p>生成完基础代码以后，我们就可以基于生成的代码写下调用端代码，具体如下：</p><pre><code>package io.grpc.hello;

import io.grpc.ManagedChannel;
import io.grpc.ManagedChannelBuilder;
import io.grpc.StatusRuntimeException;


import java.util.concurrent.TimeUnit;

public class HelloWorldClient {

    private final ManagedChannel channel;
    private final HelloServiceGrpc.HelloServiceBlockingStub blockingStub;
    /**
    * 构建Channel连接
    **/
    public HelloWorldClient(String host, int port) {
        this(ManagedChannelBuilder.forAddress(host, port)
                .usePlaintext()
                .build());
    }

    /**
    * 构建Stub用于发请求
    **/
    HelloWorldClient(ManagedChannel channel) {
        this.channel = channel;
        blockingStub = HelloServiceGrpc.newBlockingStub(channel);
    }
    
    /**
    * 调用完手动关闭
    **/
    public void shutdown() throws InterruptedException {
        channel.shutdown().awaitTermination(5, TimeUnit.SECONDS);
    }

 
    /**
    * 发送rpc请求
    **/
    public void say(String name) {
        // 构建入参对象
        HelloRequest request = HelloRequest.newBuilder().setName(name).build();
        HelloReply response;
        try {
            // 发送请求
            response = blockingStub.say(request);
        } catch (StatusRuntimeException e) {
            return;
        }
        System.out.println(response);
    }

    public static void main(String[] args) throws Exception {
            HelloWorldClient client = new HelloWorldClient(&quot;127.0.0.1&quot;, 50051);
            try {
                client.say(&quot;world&quot;);
            } finally {
                client.shutdown();
            }
    }
}
</code></pre><p><strong>调用端代码大致分成三个步骤：</strong></p><ul>
<li>首先用 host 和 port 生成 channel 连接；</li>
<li>然后用前面生成的 HelloService gRPC 创建 Stub 类；</li>
<li>最后我们可以用生成的这个 Stub 调用 say 方法发起真正的 RPC 调用，后续其它的 RPC 通信细节就对我们使用者透明了。</li>
</ul><p>为了能看清楚里面具体发生了什么，我们需要进入到 ClientCalls.blockingUnaryCall 方法里面看下逻辑细节。但是为了避免太多的细节影响你理解整体流程，我在下面这张图中只画下了最重要的部分。</p><p><img src="https://static001.geekbang.org/resource/image/57/5d/57c7d1c005c1d426ccee48a84e286b5d.jpg?wh=5429*1517" alt="" title="整体流程图"></p><p>我们可以看到，在调用端代码里面，我们只需要一行（第48行）代码就可以发起一个 RPC 调用，而具体这个请求是怎么发送到服务提供者那端的呢？这对于我们 gRPC 使用者来说是完全透明的，我们只要关注是怎么创建出 stub 对象的就可以了。</p><p>比如入参是一个字符对象，gRPC 是怎么把这个对象传输到服务提供方的呢？因为在<a href="https://time.geekbang.org/column/article/202779">[第 03 讲]</a> 中我们说过，只有二进制才能在网络中传输，但是目前调用端代码的入参是一个字符对象，那在 gRPC 里面我们是怎么把对象转成二进制数据的呢？</p><p>回到上面流程图的第3步，在 writePayload 之前，ClientCallImpl 里面有一行代码就是 method.streamRequest(message)，看方法签名我们大概就知道它是用来把对象转成一个 InputStream，有了 InputStream 我们就很容易获得入参对象的二进制数据。这个方法返回值很有意思，就是为啥不直接返回我们想要的二进制数组，而是返回一个 InputStream 对象呢？你可以先停下来想下原因，我们会在最后继续讨论这个问题。</p><p>我们接着看 streamRequest 方法的拥有者 method 是个什么对象？我们可以看到 method 是 MethodDescriptor 对象关联的一个实例，而 MethodDescriptor 是用来存放要调用 RPC 服务的接口名、方法名、服务调用的方式以及请求和响应的序列化和反序列化实现类。</p><p>大白话说就是，MethodDescriptor 是用来存储一些 RPC 调用过程中的元数据，而在 MethodDescriptor 里面 requestMarshaller 是在绑定请求的时候用来序列化方式对象的，所以当我们调用 method.streamRequest(message) 的时候，实际是调用 requestMarshaller.stream(requestMessage) 方法，而 requestMarshaller 里面会绑定一个 Parser，这个 Parser 才真正地把对象转成了 InputStream 对象。</p><p>讲完序列化在 gRPC 里面的应用后，我们再来看下在 gRPC 里面是怎么完成请求数据“断句”的，就是我们在<a href="https://time.geekbang.org/column/article/199651">[第 02 讲]</a> 中说的那个问题——二进制流经过网络传输后，怎么正确地还原请求前语义？</p><p>我们在 gRPC 文档中可以看到，gRPC 的通信协议是基于标准的 HTTP/2 设计的，而 HTTP/2 相对于常用的 HTTP/1.X 来说，它最大的特点就是多路复用、双向流，该怎么理解这个特点呢？这就好比我们生活中的单行道和双行道，HTTP/1.X 就是单行道，HTTP/2 就是双行道。</p><p>那既然在请求收到后需要进行请求“断句”，那肯定就需要在发送的时候把断句的符号加上，我们看下在 gRPC 里面是怎么加的？</p><p>因为 gRPC 是基于 HTTP/2 协议，而 HTTP/2 传输基本单位是 Frame，Frame 格式是以固定 9 字节长度的 header，后面加上不定长的 payload 组成，协议格式如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/d5/75/d5554644aefe5aaab42718d75a47de75.jpg?wh=3796*1143" alt=""></p><p>那在 gRPC 里面就变成怎么构造一个 HTTP/2 的 Frame 了。</p><p>现在回看我们上面那个流程图的第 4 步，在 write 到 Netty 里面之前，我们看到在 MessageFramer.writePayload 方法里面会间接调用 writeKnownLengthUncompressed 方法，该方法要做的两件事情就是构造 Frame Header 和 Frame Body，然后再把构造的 Frame 发送到 NettyClientHandler，最后将 Frame 写入到 HTTP/2 Stream 中，完成请求消息的发送。</p><h2>接收原理</h2><p>讲完 gRPC 的请求发送原理，我们再来看下服务提供方收到请求后会怎么处理？我们还是接着前面的那个例子，先看下服务提供方代码，具体如下：</p><pre><code>static class HelloServiceImpl extends HelloServiceGrpc.HelloServiceImplBase {

  @Override
  public void say(HelloRequest req, StreamObserver&lt;HelloReply&gt; responseObserver) {
    HelloReply reply = HelloReply.newBuilder().setMessage(&quot;Hello &quot; + req.getName()).build();
    responseObserver.onNext(reply);
    responseObserver.onCompleted();
  }
}
</code></pre><p>上面 HelloServiceImpl 类是按照 gRPC 使用方式实现了 HelloService 接口逻辑，但是对于调用者来说并不能把它调用过来，因为我们没有把这个接口对外暴露，在 gRPC 里面我们是采用 Build 模式对底层服务进行绑定，具体代码如下：</p><pre><code>package io.grpc.hello;

import io.grpc.Server;
import io.grpc.ServerBuilder;
import io.grpc.stub.StreamObserver;

import java.io.IOException;


public class HelloWorldServer {

  private Server server;

  /**
  * 对外暴露服务
  **/
  private void start() throws IOException {
    int port = 50051;
    server = ServerBuilder.forPort(port)
        .addService(new HelloServiceImpl())
        .build()
        .start();
    Runtime.getRuntime().addShutdownHook(new Thread() {
      @Override
      public void run() {
        HelloWorldServer.this.stop();
      }
    });
  }

  /**
  * 关闭端口
  **/
  private void stop() {
    if (server != null) {
      server.shutdown();
    }
  }

  /**
  * 优雅关闭
  **/
  private void blockUntilShutdown() throws InterruptedException {
    if (server != null) {
      server.awaitTermination();
    }
  }


  public static void main(String[] args) throws IOException, InterruptedException {
    final HelloWorldServer server = new HelloWorldServer();
    server.start();
    server.blockUntilShutdown();
  }
  
}
</code></pre><p>服务对外暴露的目的是让过来的请求在被还原成信息后，能找到对应接口的实现。在这之前，我们需要先保证能正常接收请求，通俗地讲就是要先开启一个 TCP 端口，让调用方可以建立连接，并把二进制数据发送到这个连接通道里面，这里依然只展示最重要的部分。</p><p><img src="https://static001.geekbang.org/resource/image/b4/cd/b43a3fb6d6929bb862893aebd7cd40cd.jpg?wh=4614*1541" alt=""></p><p>这四个步骤是用来开启一个 Netty Server，并绑定编解码逻辑的，如果你暂时看不懂，没关系的，我们可以先忽略细节。我们重点看下 NettyServerHandler 就行了，在这个 Handler  里面会绑定一个 FrameListener，gRPC 会在这个 Listener 里面处理收到数据请求的 Header 和 Body，并且也会处理 Ping、RST 命令等，具体流程如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/98/f5/98d0ecec8e2c9b75159e86253c6c3cf5.jpg?wh=4606*1527" alt=""></p><p>在收到 Header 或者 Body 二进制数据后，NettyServerHandler 上绑定的FrameListener 会把这些二进制数据转到 MessageDeframer 里面，从而实现 gRPC 协议消息的解析 。</p><p>那你可能会问，这些 Header 和 Body 数据是怎么分离出来的呢？按照我们前面说的，调用方发过来的是一串二进制数据，这就是我们前面开启 Netty Server 的时候绑定 Default HTTP/2FrameReader 的作用，它能帮助我们按照 HTTP/2 协议的格式自动切分出 Header 和 Body 数据来，而对我们上层应用 gRPC  来说，它可以直接拿拆分后的数据来用。</p><h2>总结</h2><p>这是我们基础篇的最后一讲，我们采用剖析 gRPC 源码的方式来学习如何实现一个完整的 RPC。当然整个 gRPC 的代码量可比这多得多，但今天的主要目就是想让你把前面所学的序列化、协议等方面的知识落实到具体代码上，所以我们这儿只分析了 gRPC 收发请求两个过程。</p><p>实现了这两个过程，我们就可以完成一个点对点的 RPC 功能，但在实际使用的时候，我们的服务提供方通常都是以一个集群的方式对外提供服务的，所以在 gRPC 里面你还可以看到负载均衡、服务发现等功能。而且 gRPC 采用的是 HTTP/2 协议，我们还可以通过 Stream 方式来调用服务，以提升调用性能。</p><p>总的来说，其实我们可以简单地认为<strong>gRPC 就是采用 HTTP/2 协议，并且默认采用 PB 序列化方式的一种 RPC</strong>，它充分利用了 HTTP/2 的多路复用特性，使得我们可以在同一条链路上双向发送不同的 Stream 数据，以解决 HTTP/1.X 存在的性能问题。</p><h2>课后思考</h2><p>我们讲到，在 gRPC 调用的时候，我们有一个关键步骤就是把对象转成可传输的二进制，但是在 gRPC 里面，我们并没有直接转成二进制数组，而是返回一个 InputStream，你知道这样做的好处是什么吗？</p><p>欢迎留言和我分享你的答案，也欢迎你把文章分享给你的朋友，邀请他加入学习。我们下节课再见！</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/0f/bf/ee93c4cf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>雨霖铃声声慢</span>
  </div>
  <div class="_2_QraFYR_0">InputStream封装了底层传输的字节缓冲区实现，它通常是一组通过指针连接起来的内存块的集合，这些内存块由网络的零拷贝获取的。由于不能保证能够从内存块中获取一个byte[]，我们不能传递一个简单的byte[]或byte[][]，并且可能需要一个目标byte[]来从缓冲区中获取数据。<br>另外byte[]的缺点是需要从缓冲区中复制一个大的、连续的数据，而实际上没有什么方法可以使它执行得更好。当使用压缩时，我们也不知道消息未压缩的长度，它是动态解压缩的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 没错，避免二次拷贝</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-03 08:51:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/89/04/c1474869.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>tulip</span>
  </div>
  <div class="_2_QraFYR_0">老师可以出一期go 的嘛</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-29 20:38:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/78/c7/083a3a0b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>新世界</span>
  </div>
  <div class="_2_QraFYR_0">老师能否把代码提交到github上，我们可以下载下来跑一跑，看一看</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-02 11:55:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/7b/98/8f1aecf4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>楼下小黑哥</span>
  </div>
  <div class="_2_QraFYR_0">网上搜索资料查到，分享一下：<br>A stream also has the advantage that you don&#39;t have to have all bytes in memory at the same time, which is convenient if the size of the data is large and can easily be handled in small chunks.</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-06 08:07:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/04/51/da465a93.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>超威丶</span>
  </div>
  <div class="_2_QraFYR_0">难道http2的核心实现不就是基于流实现？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: stream传输是建立在多路复用的基础上</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-02 14:56:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/22/f4/9fd6f8f0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>核桃</span>
  </div>
  <div class="_2_QraFYR_0">这里说一下对inpustream的理解。首先要理解，当网卡接收到数据之后是存放到缓冲区的，这里其实已经分配过空间了。而上层应用想拿到这些数据，可以再次分配一段空间，然后把数据从缓冲区复制到这里，但是这样就多了一次复制了。而为了解决这个问题，inputstream本质上就是一段指针，指向了缓冲区的数据，那样可以直接使用这段地址的数据。不管用什么语言实现，本质思想都是类似的。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-09 20:19:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/59/1d/c89abcd8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>四喜</span>
  </div>
  <div class="_2_QraFYR_0">分享一下Python的Tutorial：<br><br>gRPC Basics - Python：https:&#47;&#47;grpc.io&#47;docs&#47;tutorials&#47;basic&#47;python&#47;<br><br>gRPC Python Quick Start：https:&#47;&#47;grpc.io&#47;docs&#47;quickstart&#47;python&#47;</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-10 17:54:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/82/3d/356fc3d6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>忆水寒</span>
  </div>
  <div class="_2_QraFYR_0">老师，有个疑问。按道理客户端发起一次rpc调用，通过序列化、网络传输、服务端处理再响应，这期间有时间差的。上面客户端代码除非是阻塞的，否则不可能立马得到结果吧。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，例子里面的需要在客户端等待</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-02 18:24:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/4e/1b/f4b786b9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>飞翔</span>
  </div>
  <div class="_2_QraFYR_0">老师 有说法是内部调用用rpc 外部用http 这是为什么呀</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 内部应用之间通信更强调性能</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-04 13:11:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/24/ba/99/da85915f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Hzº</span>
  </div>
  <div class="_2_QraFYR_0">感觉对Java不熟悉的同学，不太友好</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-27 15:39:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/67/0e/c77ad9b1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>eason2017</span>
  </div>
  <div class="_2_QraFYR_0">老师好，可以把 为客户端和服务器端生成消息对象和 RPC 基础代码 的命令提供出来吗？谢谢。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以看下grpc官网或者搜索一下</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-29 14:20:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/48/f3/f1034ffd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>cricket1981</span>
  </div>
  <div class="_2_QraFYR_0">以流的方式处理请求数据，适合请求数据数据量大情况。好比Sax和Dom区别。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 在传输层也需要用stream，避免二次拷贝</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-02 08:15:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_757cbc</span>
  </div>
  <div class="_2_QraFYR_0">static class HelloServiceImpl 报错，去掉static调试成功</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-14 17:35:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/22/7a/22/45307c91.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>墨白</span>
  </div>
  <div class="_2_QraFYR_0">老师httpclient底层也是基于socket封装，为什么没有基于FileChannelimp零拷贝的实现</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-18 17:11:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/0c/ad/d1ab1995.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Di Yu</span>
  </div>
  <div class="_2_QraFYR_0">gRPC看起来没有用到动态代理吗？在调用端代码里边我们手动的建立了channel，然后手动的call blockingStub.say(request)。这是为了更好的性能吧？如果要用动态代理，那我们会把建立channel等事情放到代理类里边，这样调用端代码就简化一些了，这样理解对吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-05 16:14:41</div>
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
  <div class="_2_QraFYR_0">我们讲到，在 gRPC 调用的时候，我们有一个关键步骤就是把对象转成可传输的二进制，但是在 gRPC 里面，我们并没有直接转成二进制数组，而是返回一个 InputStream，你知道这样做的好处是什么吗？<br><br>这个不知道哎😂<br>猜测是为性能故，看评论区的讨论，猜测是正确的，不过细节还是不太清楚，需要后补一下。<br><br>Inputstream——避免二次拷贝（序列化＋encode）——更高的性能。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-13 09:38:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/30/c1/2dde6700.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>密码123456</span>
  </div>
  <div class="_2_QraFYR_0">为什么能避免二次拷贝？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 序列化和encode</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-27 21:48:09</div>
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
  <div class="_2_QraFYR_0">这样跟HTTP2有什么区别呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: grpc是用http2来传输的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-06 08:42:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/3b/47/f6c772a1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Jackey</span>
  </div>
  <div class="_2_QraFYR_0">精彩</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-02 23:41:41</div>
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
  <div class="_2_QraFYR_0">我猜的是网络字节序问题吧😊</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-02 06:47:18</div>
  </div>
</div>
</div>
</li>
</ul>