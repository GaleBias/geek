<audio title="34 _ 动手实现一个简单的RPC框架（四）：服务端" src="https://static001.geekbang.org/resource/audio/c5/32/c5e8030aa04e3ae3db66015dba8dd932.mp3" controls="controls"></audio> 
<p>你好，我是李玥。</p><p>上节课我们一起学习了如何来构建这个RPC框架中最关键的部分，也就是：在客户端，如何根据用户注册的服务接口来动态生成桩的方法。在这里，除了和语言特性相关的一些动态编译小技巧之外，你更应该掌握的是其中动态代理这种设计思想，它的使用场景以及实现方法。</p><p>这节课我们一起来实现这个框架的最后一部分：服务端。对于我们这个RPC框架来说，服务端可以分为两个部分：注册中心和RPC服务。其中，注册中心的作用是帮助客户端来寻址，找到对应RPC服务的物理地址，RPC服务用于接收客户端桩的请求，调用业务服务的方法，并返回结果。</p><h2>注册中心是如何实现的？</h2><p>我们先来看看注册中心是如何实现的。一般来说，一个完整的注册中心也是分为客户端和服务端两部分的，客户端给调用方提供API，并实现与服务端的通信；服务端提供真正的业务功能，记录每个RPC服务发来的注册信息，并保存到它的元数据中。当有客户端来查询服务地址的时候，它会从元数据中获取服务地址，返回给客户端。</p><p>由于注册中心并不是这个RPC框架的重点内容，所以在这里，我们只实现了一个单机版的注册中心，它只有客户端没有服务端，所有的客户端依靠读写同一个元数据文件来实现元数据共享。所以，我们这个注册中心只能支持单机运行，并不支持跨服务器调用。</p><!-- [[[read_end]]] --><p>但是，我们在这里，同样采用的是“面向接口编程”的设计模式，这样，你可以在不改动一行代码的情况下，就可以通过增加一个SPI插件的方式，提供一个可以跨服务器调用的真正的注册中心实现，比如说，一个基于HTTP协议实现的注册中心。我们再来复习一下，这种面向接口编程的设计是如何在注册中心中来应用的。</p><p>首先，我们在RPC服务的接入点，接口RpcAccessPoint中增加一个获取注册中心实例的方法：</p><pre><code>public interface RpcAccessPoint extends Closeable{
    /**
     * 获取注册中心的引用
     * @param nameServiceUri 注册中心URI
     * @return 注册中心引用
     */
    NameService getNameService(URI nameServiceUri);

    // ...
}
</code></pre><p>这个方法的参数就是注册中心的URI，也就是它的地址，返回值就是访问这个注册中心的实例。然后我们再给NameService接口增加两个方法：</p><pre><code>public interface NameService {

    /**
     * 所有支持的协议
     */
    Collection&lt;String&gt; supportedSchemes();

    /**
     * 连接注册中心
     * @param nameServiceUri 注册中心地址
     */
    void connect(URI nameServiceUri);

    // ...
}
</code></pre><p>其中supportedSchemes方法，返回可以支持的所有协议，比如我们在这个例子中的实现，它的协议是“file”。connect方法就是给定注册中心服务端的URI，去建立与注册中心服务端的连接。</p><p>下面我们来看获取注册中心的方法getNameService的实现，它的实现也很简单，就是通过SPI机制加载所有的NameService的实现类，然后根据给定的URI中的协议，去匹配支持这个协议的实现类，然后返回这个实现的引用就可以了。由于这部分实现是通用并且不会改变的，我们直接把实现代码放在RpcAccessPoint这个接口中。</p><p>这样我们就实现了一个可扩展的注册中心接口，系统可以根据URI中的协议，动态地来选择不同的注册中心实现。增加一种注册中心的实现，也不需要修改任何代码，只要按照SPI的规范，把协议的实现加入到运行时CLASSPATH中就可以了。（这里设置CLASSPATH的目的，在于告诉Java执行环境，在哪些目录下可以找到你所要执行的Java程序所需要的类或者包。）</p><p>我们这个例子中注册中心的实现类是LocalFileNameService，它的实现比较简单，就是去读写一个本地文件，实现注册服务registerService方法时，把服务提供者保存到本地文件中；实现查找服务lookupService时，就是去本地文件中读出所有的服务提供者，找到对应的服务提供者，然后返回。</p><p>这里面有一点需要注意的是，由于这个本地文件它是一个共享资源，它会被RPC框架所有的客户端和服务端并发读写。所以，这时你要怎么做呢？对，<strong>必须要加锁！</strong></p><p>由于我们这个文件可能被多个进程读写，所以这里不能使用我们之前讲过的，编程语言提供的那些锁，原因是这些锁只能在进程内起作用，它锁不住其他进程。我们这里面必须使用由操作系统提供的文件锁。这个锁的使用和其他的锁并没有什么区别，同样是在访问共享文件之前先获取锁，访问共享资源结束后必须释放锁。具体的代码你可以去查看LocalFileNameService这个实现类。</p><h2>RPC服务是怎么实现的？</h2><p>接下来，我们再来看看RPC服务是怎么实现的。RPC服务也就是RPC框架的服务端。我们在之前讲解这个RPC框架的实现原理时讲到过，RPC框架的服务端主要需要实现下面这两个功能：</p><ol>
<li>服务端的业务代码把服务的实现类注册到RPC框架中;</li>
<li>接收客户端桩发出的请求，调用服务的实现类并返回结果。</li>
</ol><p>把服务的实现类注册到RPC框架中，这个逻辑的实现很简单，我们只要使用一个合适的数据结构，记录下所有注册的实例就可以了，后面在处理客户端请求的时候，会用到这个数据结构来查找服务实例。</p><p>然后我们来看，RPC框架的服务端如何来处理客户端发送的RPC请求。首先来看服务端中，使用Netty接收所有请求数据的处理类RequestInvocation的channelRead0方法。</p><pre><code>@Override
protected void channelRead0(ChannelHandlerContext channelHandlerContext, Command request) throws Exception {
    RequestHandler handler = requestHandlerRegistry.get(request.getHeader().getType());
    if(null != handler) {
        Command response = handler.handle(request);
        if(null != response) {
            channelHandlerContext.writeAndFlush(response).addListener((ChannelFutureListener) channelFuture -&gt; {
                if (!channelFuture.isSuccess()) {
                    logger.warn(&quot;Write response failed!&quot;, channelFuture.cause());
                    channelHandlerContext.channel().close();
                }
            });
        } else {
            logger.warn(&quot;Response is null!&quot;);
        }
    } else {
        throw new Exception(String.format(&quot;No handler for request with type: %d!&quot;, request.getHeader().getType()));
    }
}
</code></pre><p>这段代码的处理逻辑就是，根据请求命令的Header中的请求类型type，去requestHandlerRegistry中查找对应的请求处理器RequestHandler，然后调用请求处理器去处理请求，最后把结果发送给客户端。</p><p>这种通过“请求中的类型”，把请求分发到对应的处理类或者处理方法的设计，我们在RocketMQ和Kafka的源代码中都见到过，在服务端处理请求的场景中，这是一个很常用的方法。我们这里使用的也是同样的设计，不同的是，我们使用了一个命令注册机制，让这个路由分发的过程省略了大量的if-else或者是switch代码。这样做的好处是，可以很方便地扩展命令处理器，而不用修改路由分发的方法，并且代码看起来更加优雅。这个命令注册机制的实现类是RequestHandlerRegistry，你可以自行去查看。</p><p>因为我们这个RPC框架中只需要处理一种类型的请求：RPC请求，所以我们只实现了一个命令处理器：RpcRequestHandler。这部分代码是这个RPC框架服务端最核心的部分，你需要重点掌握。另外，为了便于你理解，在这里我只保留了核心业务逻辑，你在充分理解这部分核心业务逻辑之后，可以再去查看项目中完整的源代码，补全错误处理部分。</p><p>我们先来看它处理客户端请求，也就是这个handle方法的实现。</p><pre><code>@Override
public Command handle(Command requestCommand) {
    Header header = requestCommand.getHeader();
    // 从payload中反序列化RpcRequest
    RpcRequest rpcRequest = SerializeSupport.parse(requestCommand.getPayload());
    // 查找所有已注册的服务提供方，寻找rpcRequest中需要的服务
    Object serviceProvider = serviceProviders.get(rpcRequest.getInterfaceName());
    // 找到服务提供者，利用Java反射机制调用服务的对应方法
    String arg = SerializeSupport.parse(rpcRequest.getSerializedArguments());
    Method method = serviceProvider.getClass().getMethod(rpcRequest.getMethodName(), String.class);
    String result = (String ) method.invoke(serviceProvider, arg);
    // 把结果封装成响应命令并返回
    return new Command(new ResponseHeader(type(), header.getVersion(), header.getRequestId()), SerializeSupport.serialize(result));
    // ...
}
</code></pre><ol>
<li>把requestCommand的payload属性反序列化成为RpcRequest；</li>
<li>根据rpcRequest中的服务名，去成员变量serviceProviders中查找已注册服务实现类的实例；</li>
<li>找到服务提供者之后，利用Java反射机制调用服务的对应方法；</li>
<li>把结果封装成响应命令并返回，在RequestInvocation中，它会把这个响应命令发送给客户端。</li>
</ol><p>再来看成员变量serviceProviders，它的定义是：Map&lt;String/<em>service name</em>/, Object/<em>service provider</em>/&gt; serviceProviders。它实际上就是一个Map，Key就是服务名，Value就是服务提供方，也就是服务实现类的实例。这个Map的数据从哪儿来的呢？我们来看一下RpcRequestHandler这个类的定义：</p><pre><code>@Singleton
public class RpcRequestHandler implements RequestHandler, ServiceProviderRegistry {
    @Override
    public synchronized &lt;T&gt; void addServiceProvider(Class&lt;? extends T&gt; serviceClass, T serviceProvider) {
        serviceProviders.put(serviceClass.getCanonicalName(), serviceProvider);
        logger.info(&quot;Add service: {}, provider: {}.&quot;,
                serviceClass.getCanonicalName(),
                serviceProvider.getClass().getCanonicalName());
    }
    // ...
}
</code></pre><p>可以看到，这个类不仅实现了处理客户端请求的RequestHandler接口，同时还实现了注册RPC服务ServiceProviderRegistry接口，也就是说，RPC框架服务端需要实现的两个功能——注册RPC服务和处理客户端RPC请求，都是在这一个类RpcRequestHandler中实现的，所以说，这个类是这个RPC框架服务端最核心的部分。成员变量serviceProviders这个Map中的数据，也就是在addServiceProvider这个方法的实现中添加进去的。</p><p>还有一点需要注意的是，我们RpcRequestHandler上增加了一个注解@Singleton，限定这个类它是一个单例模式，这样确保在进程中任何一个地方，无论通过ServiceSupport获取RequestHandler或者ServiceProviderRegistry这两个接口的实现类，拿到的都是RpcRequestHandler这个类的唯一的一个实例。这个@Singleton的注解和获取单例的实现在ServiceSupport中，你可以自行查看代码。顺便说一句，在Spring中，也提供了单例Bean的支持，它的实现原理也是类似的。</p><h2>小结</h2><p>以上就是实现这个RPC框架服务端的全部核心内容，照例我们来做一个总结。</p><p>首先我们一起来实现了一个注册中心，注册中心的接口设计采用了依赖倒置的设计原则（也就是“面向接口编程”的设计），并且还提供了一个“根据URI协议，自动加载对应实现类”的机制，使得我们可以通过扩展不同的协议，增加不同的注册中心实现。</p><p>这种“通过请求参数中的类型，来动态加载对应实现”的设计，在我们这个RPC框架中不止这一处用到，在“处理客户端命令并路由到对应的处理类”这部分代码中，使用的也是这样一种设计。</p><p>在RPC框架的服务端处理客户端请求的业务逻辑中，我们分两层做了两次请求分发：</p><ol>
<li>在RequestInvocation类中，根据请求命令中的请求类型(command.getHeader().getType())，分发到对应的请求处理器RequestHandler中；</li>
<li>RpcRequestHandler类中，根据RPC请求中的服务名，把RPC请求分发到对应的服务实现类的实例中去。</li>
</ol><p>这两次分发采用的设计是差不多的，但你需要注意的是，这并不是一种过度设计。原因是，我们这两次分发分别是在不同的业务抽象分层中，第一次分发是在服务端的网络传输层抽象中，它是网络传输的一部分，而第二次分发是RPC框架服务端的业务层，是RPC框架服务端的一部分。良好的分层设计，目的也是让系统各部分更加的“松耦合，高内聚”。</p><h2>思考题</h2><p>这节课的课后作业，我们来继续写代码。需要你实现一个JDBC协议的注册中心，并加入到我们的RPC框架中。加入后，我们的注册中心就可以使用一个支持JDBC协议的数据库（比如MySQL）作为注册中心的服务端，实现跨服务器的服务注册和查询。要求：</p><ol>
<li>调用RpcAccessPoint.getNameService()方法，获取注册中心实例时，传入的参数就是JDBC的URL，比如：“jdbc:mysql://127.0.0.1/mydb”;</li>
<li>不能修改RPC框架的源代码;</li>
<li>实现必须具有通用性，可以支持任意一种JDBC数据库。</li>
</ol><p>欢迎你在评论区留言，分享你的代码。</p><p>感谢阅读，如果你觉得这篇文章对你有一些启发，也欢迎把它分享给你的朋友。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4b/4d/0239bc19.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>益军</span>
  </div>
  <div class="_2_QraFYR_0">您好哈，提个疑问:<br>服务端业务方法不应该在channelRead0中执行，会导致netty ioEvent线程阻塞，应该异步提交到业务线程池执行<br>workers.execute(() -&gt; {<br>    Command response = handler.handle(request);<br>    if (null != response) {<br>        channelHandlerContext.writeAndFlush(response).addListener((ChannelFutureListener) channelFuture -&gt; {<br>            if (!channelFuture.isSuccess()) {<br>                logger.warn(&quot;Write response failed!&quot;, channelFuture.cause());<br>                channelHandlerContext.channel().close();<br>            }<br>        });<br>    } else {<br>        logger.warn(&quot;Response is null!&quot;);<br>    }<br>});<br> Command response = handler.handle(request);</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 在实际生产过程中，很多情况是需要这样处理的。避免阻塞IO线程。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-10 18:04:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/04/71/853b2292.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Gred</span>
  </div>
  <div class="_2_QraFYR_0">来交作业啦【https:&#47;&#47;github.com&#47;Gred01&#47;simple-rpc-framework&#47;tree&#47;nameservice】，其实本应该在周五就写好了，现在写的demo支持mysql和oracle，初始化sql在rpc-netty底下init-sql.md。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 交作业的都是好同学，👍！<br><br>一个建议：使用PreparedStatement，不要用字符出拼SQL，会有SQL注入漏洞。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-28 17:10:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/ac/62/37912d51.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>东方奇骥</span>
  </div>
  <div class="_2_QraFYR_0">这里注册中心信息是保存在本地文件中，如果保存在数据库，就要用数据库锁，或者zookeeper、redis的分布式锁。感谢老师，工作三年了，没写过框架，这个专栏收获不小！实战篇就超值了。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-21 15:09:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/05/06/f5979d65.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>亚洲舞王.尼古拉斯赵四</span>
  </div>
  <div class="_2_QraFYR_0">看过了这四节rpc的课程之后，再去看了看中KafkaClient及实现的源码，和本篇讲述思想都是相似的，玥哥厉害！还是思想重要，同时觉得期末测试中的状态转换图也是中间件的基本中的基本，重要中的重要的点，KafkaClient这个接口中很多方法都是基于状态字段来给予的返回。（为什么提起状态转换图我就想到了线程的状态转换~~☺️</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-11 17:35:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/d8/d6/47da34bf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>任鹏斌</span>
  </div>
  <div class="_2_QraFYR_0">代码拿下来刚消化了一部分，慢慢消化，希望能做一些扩展，一转眼课程要结束了，老师辛苦！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-13 22:09:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/05/7f/d35ab9a1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>z.l</span>
  </div>
  <div class="_2_QraFYR_0">请教下老师，如果要实现一个可供生产环境大规模集群使用的注册中心，JDBC协议是不是就不太合适了？这种情况下一个注册中心要满足哪些要求呢？个人盲猜：可多台部署、基于tcp协议最好也是netty实现、彼此之间要保证数据一致性（好像也不用强一致），不知道理解对不对</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 大致的要求就是：<br><br>可用性：不能因为某个节点故障导致注册中心不可以；<br>数据一致性：能保证顺序一致性一般就可以满足需求了。<br>性能：server节点状态变更后，能尽快更新。<br><br>另外，大规模集群情况下，需要考虑业务系统上下线时，注册中心的性能问题。比如：一个几千个节点的微服务上线，会重启几千个Server节点，这时候注册中心需要更新大量的数据。<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-01 22:47:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKlwpFM3tkeG15YqyJTYWkfqkdmro9POq6SicYm57TaEFDOUZCXjoe0Z0Iz6UibGQqic3icJRsHdFzibtw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>zero</span>
  </div>
  <div class="_2_QraFYR_0">如果服务端挂掉了，怎么通知NameService呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个问题在我们的demo中并没有解决。实际在生产环境中，有些注册中心会定期检查服务端的状态，但是即使这样也不能保证注册中心中维护的状态和服务端实际的状态实时同步。<br><br>解决这个问题，更多的是依靠rpc框架客户端的自动重试策略，比如，请求某个服务端实例超时后，立即换一个实例自动重试。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-30 11:24:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/15/69/187b9968.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>南山</span>
  </div>
  <div class="_2_QraFYR_0">抓耳挠腮了两天，还没开始动手，不知道怎么下手～</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 先把课后思考题完成了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-12 15:43:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/69/af/ceb4450c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Asha</span>
  </div>
  <div class="_2_QraFYR_0">老师，有个问题，不知道还能不能到达你那，我们在进阶篇中讲到了应用层协议，其实就是半包处理，为什么在rpc这里没有这部分的处理</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-23 16:17:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/5a/5e/a897cb0d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>grey927</span>
  </div>
  <div class="_2_QraFYR_0">老师，单例模式的实现，如果用Effective Java中推荐的枚举单例是否会更好一些。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-12 16:55:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/aa/b1/c834aab5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张小勋</span>
  </div>
  <div class="_2_QraFYR_0">老师 你好~ 本人是net的 最近刚下了 git 的代码 去看了下~有些写法java和net 还是有区别的  问几个问题 希望老师能够作答一下<br>首先：helloServiceapi 项目中 建了一个接口   这个接口在服务端去实现   客户端也用到了~ 问下这个接口 在正常的使用中 都是服务端定义好  去给客户端 去使用的么  <br>第二点：就是在服务端  服务端启动的时候  这个demo 中 是自己去实例化了HelloServiceImpl对象  如果在生产环境中 是不是启动的时候 通过反射 去实例化 那些特定的对外提供的服务  HelloService这样的接口 是不是也要做下标识  是这个思路么<br>最后 给自己立下个flag  这个月 自己会去 用netcore+netty+zookeeper 去实现一个RPC  多谢老师 </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复:  准确的说，应该是实现定义好接口，服务端和客户端都需要使用这个接口；<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-08 16:32:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/jDG4trQm3t6licZym9TayWRib15Z3auNv4ictblMvkHxzYqOwuD3HibGs8ktGBrPefDic5ZYH85lg9CroynSMlJxcpw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>kim118000</span>
  </div>
  <div class="_2_QraFYR_0">Method method = serviceProvider.getClass().getMethod(rpcRequest.getMethodName(), String.class);<br>每次请求都用反射获取method，有没有性能损耗，是不是存起来，请问老师这部分生产级别怎么处理的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: JVM对反射的优化已经很好了，这部分的性能损耗并不是很大，所以一般不用特别的优化。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-15 08:20:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/34/df/64e3d533.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>leslie</span>
  </div>
  <div class="_2_QraFYR_0">动手要欠账了：大致明白了RPC整个过程需要什么了，啃几遍梳理一下-看看各个是怎么实现的啃明白再去思考Go怎么实现。。。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-12 09:46:03</div>
  </div>
</div>
</div>
</li>
</ul>