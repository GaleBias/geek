<audio title="06 _ 请求通道：如何实现Kafka请求队列？" src="https://static001.geekbang.org/resource/audio/10/27/10062b1488ba1dacaf12e89c3f717f27.mp3" controls="controls"></audio> 
<p>你好，我是胡夕。日志模块我们已经讲完了，你掌握得怎样了呢？如果你在探索源码的过程中碰到了问题，记得在留言区里写下你的困惑，我保证做到知无不言。</p><p>现在，让我们开启全新的“请求处理模块”的源码学习之旅。坦率地讲，这是我自己给Kafka源码划分的模块，在源码中可没有所谓的“请求处理模块”的提法。但我始终觉得，这样划分能够帮助你清晰地界定不同部分的源码的作用，可以让你的阅读更有针对性，学习效果也会事半功倍。</p><p>在这个模块，我会带你了解请求处理相关的重点内容，包括请求处理通道、请求处理全流程分析和请求入口类分析等。今天，我们先来学习下Kafka是如何实现请求队列的。源码中位于core/src/main/scala/kafka/network下的RequestChannel.scala文件，是主要的实现类。</p><p>当我们说到Kafka服务器端，也就是Broker的时候，往往会说它承担着消息持久化的功能，但本质上，它其实就是<strong>一个不断接收外部请求、处理请求，然后发送处理结果的Java进程</strong>。</p><p>你可能会觉得奇怪，Broker不是用Scala语言编写的吗，怎么这里又是Java进程了呢？这是因为，Scala代码被编译之后生成.class文件，它和Java代码被编译后的效果是一样的，因此，Broker启动后也仍然是一个普通的Java进程。</p><!-- [[[read_end]]] --><p><strong>高效地保存排队中的请求，是确保Broker高处理性能的关键</strong>。既然这样，那你一定很想知道，Broker上的请求队列是怎么实现的呢？接下来，我们就一起看下<strong>Broker底层请求对象的建模</strong>和<strong>请求队列的实现原理</strong>，以及Broker<strong>请求处理方面的核心监控指标</strong>。</p><p>目前，Broker与Clients进行交互主要是基于<span class="orange">Request/Response机制</span>，所以，我们很有必要学习一下源码是如何建模或定义Request和Response的。</p><h2>请求（Request）</h2><p>我们先来看一下RequestChannel源码中的Request定义代码。</p><pre><code>sealed trait BaseRequest
case object ShutdownRequest extends BaseRequest

class Request(val processor: Int,
              val context: RequestContext,
              val startTimeNanos: Long,
              memoryPool: MemoryPool,
              @volatile private var buffer: ByteBuffer,
              metrics: RequestChannel.Metrics) extends BaseRequest {
  ......
}
</code></pre><p>简单提一句，Scala语言中的“trait”关键字，大致类似于Java中的interface（接口）。从代码中，我们可以知道，BaseRequest是一个trait接口，定义了基础的请求类型。它有两个实现类：<strong>ShutdownRequest类</strong>和<strong>Request类</strong>。</p><p>ShutdownRequest仅仅起到一个标志位的作用。当Broker进程关闭时，请求处理器（RequestHandler，在第9讲我会具体讲到它）会发送ShutdownRequest到专属的请求处理线程。该线程接收到此请求后，会主动触发一系列的Broker关闭逻辑。</p><p>Request则是真正定义各类Clients端或Broker端请求的实现类。它定义的属性包括processor、context、startTimeNanos、memoryPool、buffer和metrics。下面我们一一来看。</p><h3>processor</h3><p>processor是Processor线程的序号，即<strong>这个请求是由哪个Processor线程接收处理的</strong>。Broker端参数num.network.threads控制了Broker每个监听器上创建的Processor线程数。</p><p>假设你的listeners配置为PLAINTEXT://localhost:9092,SSL://localhost:9093，那么，在默认情况下，Broker启动时会创建6个Processor线程，每3个为一组，分别给listeners参数中设置的两个监听器使用，每组的序号分别是0、1、2。</p><p>你可能会问，为什么要保存Processor线程序号呢？这是因为，当Request被后面的I/O线程处理完成后，还要依靠Processor线程发送Response给请求发送方，因此，Request中必须记录它之前是被哪个Processor线程接收的。另外，这里我们要先明确一点：<strong>Processor线程仅仅是网络接收线程，不会执行真正的Request请求处理逻辑</strong>，那是I/O线程负责的事情。</p><h3>context</h3><p><strong>context是用来标识请求上下文信息的</strong>。Kafka源码中定义了RequestContext类，顾名思义，它保存了有关Request的所有上下文信息。RequestContext类定义在clients工程中，下面是它主要的逻辑代码。我用注释的方式解释下主体代码的含义。</p><pre><code>public class RequestContext implements AuthorizableRequestContext {
    public final RequestHeader header; // Request头部数据，主要是一些对用户不可见的元数据信息，如Request类型、Request API版本、clientId等
    public final String connectionId; // Request发送方的TCP连接串标识，由Kafka根据一定规则定义，主要用于表示TCP连接
    public final InetAddress clientAddress; // Request发送方IP地址
    public final KafkaPrincipal principal;  // Kafka用户认证类，用于认证授权
    public final ListenerName listenerName; // 监听器名称，可以是预定义的监听器（如PLAINTEXT），也可自行定义
    public final SecurityProtocol securityProtocol; // 安全协议类型，目前支持4种：PLAINTEXT、SSL、SASL_PLAINTEXT、SASL_SSL
    public final ClientInformation clientInformation; // 用户自定义的一些连接方信息
    // 从给定的ByteBuffer中提取出Request和对应的Size值
    public RequestAndSize parseRequest(ByteBuffer buffer) {
             ......
    }
    // 其他Getter方法
    ......
}
</code></pre><h3>startTimeNanos</h3><p><strong>startTimeNanos记录了Request对象被创建的时间，主要用于各种时间统计指标的计算</strong>。</p><p>请求对象中的很多JMX指标，特别是时间类的统计指标，都需要使用startTimeNanos字段。你要注意的是，<strong>它是以纳秒为单位的时间戳信息，可以实现非常细粒度的时间统计精度</strong>。</p><h3>memoryPool</h3><p>memoryPool表示源码定义的一个非阻塞式的内存缓冲区，主要作用是<strong>避免Request对象无限使用内存</strong>。</p><p>当前，该内存缓冲区的接口类和实现类，分别是MemoryPool和SimpleMemoryPool。你可以重点关注下SimpleMemoryPool的tryAllocate方法，看看它是怎么为Request对象分配内存的。</p><h3>buffer</h3><p>buffer是真正保存Request对象内容的字节缓冲区。Request发送方必须按照Kafka RPC协议规定的格式向该缓冲区写入字节，否则将抛出InvalidRequestException异常。<strong>这个逻辑主要是由RequestContext的parseRequest方法实现的</strong>。</p><pre><code>public RequestAndSize parseRequest(ByteBuffer buffer) {
    if (isUnsupportedApiVersionsRequest()) {
        // 不支持的ApiVersions请求类型被视为是V0版本的请求，并且不做解析操作，直接返回
        ApiVersionsRequest apiVersionsRequest = new ApiVersionsRequest(new ApiVersionsRequestData(), (short) 0, header.apiVersion());
        return new RequestAndSize(apiVersionsRequest, 0);
    } else {
        // 从请求头部数据中获取ApiKeys信息
        ApiKeys apiKey = header.apiKey();
        try {
            // 从请求头部数据中获取版本信息
            short apiVersion = header.apiVersion();
            // 解析请求
            Struct struct = apiKey.parseRequest(apiVersion, buffer);
            AbstractRequest body = AbstractRequest.parseRequest(apiKey, apiVersion, struct);
            // 封装解析后的请求对象以及请求大小返回
            return new RequestAndSize(body, struct.sizeOf());
        } catch (Throwable ex) {
            // 解析过程中出现任何问题都视为无效请求，抛出异常
            throw new InvalidRequestException(&quot;Error getting request for apiKey: &quot; + apiKey +
                    &quot;, apiVersion: &quot; + header.apiVersion() +
                    &quot;, connectionId: &quot; + connectionId +
                    &quot;, listenerName: &quot; + listenerName +
                    &quot;, principal: &quot; + principal, ex);
        }
    }
}
</code></pre><p>就像前面说过的，这个方法的主要目的是<strong>从ByteBuffer中提取对应的Request对象以及它的大小</strong>。</p><p>首先，代码会判断该Request是不是Kafka不支持的ApiVersions请求版本。如果是不支持的，就直接构造一个V0版本的ApiVersions请求，然后返回。否则的话，就继续下面的代码。</p><p>这里我稍微解释一下ApiVersions请求的作用。当Broker接收到一个ApiVersionsRequest的时候，它会返回Broker当前支持的请求类型列表，包括请求类型名称、支持的最早版本号和最新版本号。如果你查看Kafka的bin目录的话，你应该能找到一个名为kafka-broker-api-versions.sh的脚本工具。它的实现原理就是，构造ApiVersionsRequest对象，然后发送给对应的Broker。</p><p>你可能会问，如果是ApiVersions类型的请求，代码中为什么要判断一下它的版本呢？这是因为，和处理其他类型请求不同的是，Kafka必须保证版本号比最新支持版本还要高的ApiVersions请求也能被处理。这主要是考虑到了客户端和服务器端版本的兼容问题。客户端发送请求给Broker的时候，很可能不知道Broker到底支持哪些版本的请求，它需要使用ApiVersionsRequest去获取完整的请求版本支持列表。但是，如果不做这个判断，Broker可能无法处理客户端发送的ApiVersionsRequest。</p><p>通过这个检查之后，代码会从请求头部数据中获取ApiKeys信息以及对应的版本信息，然后解析请求，最后封装解析后的请求对象以及请求大小，并返回。</p><h3>metrics</h3><p>metrics是Request相关的各种监控指标的一个管理类。它里面构建了一个Map，封装了所有的请求JMX指标。除了上面这些重要的字段属性之外，Request类中的大部分代码都是与监控指标相关的，后面我们再详细说。</p><h2>响应（Response）</h2><p>说完了Request代码，我们再来说下Response。Kafka为Response定义了1个抽象父类和5个具体子类，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/a0/6a/a03ecdba3c118efbc3910b5a1badc96a.jpg?wh=3640*3815" alt=""></p><p>看到这么多类，你可能会有点蒙，这些都是干吗的呢？别着急，现在我分别给你介绍下各个类的作用。</p><ul>
<li>Response：定义Response的抽象基类。每个Response对象都包含了对应的Request对象。这个类里最重要的方法是onComplete方法，用来实现每类Response被处理后需要执行的回调逻辑。</li>
<li>SendResponse：Kafka大多数Request处理完成后都需要执行一段回调逻辑，SendResponse就是保存返回结果的Response子类。里面最重要的字段是<strong>onCompletionCallback</strong>，即<strong>指定处理完成之后的回调逻辑</strong>。</li>
<li>NoResponse：有些Request处理完成后无需单独执行额外的回调逻辑。NoResponse就是为这类Response准备的。</li>
<li>CloseConnectionResponse：用于出错后需要关闭TCP连接的场景，此时返回CloseConnectionResponse给Request发送方，显式地通知它关闭连接。</li>
<li>StartThrottlingResponse：用于通知Broker的Socket Server组件（后面几节课我会讲到它）某个TCP连接通信通道开始被限流（throttling）。</li>
<li>EndThrottlingResponse：与StartThrottlingResponse对应，通知Broker的SocketServer组件某个TCP连接通信通道的限流已结束。</li>
</ul><p>你可能又会问了：“这么多类，我都要掌握吗？”其实是不用的。你只要了解SendResponse表示正常需要发送Response，而NoResponse表示无需发送Response就可以了。至于CloseConnectionResponse，它是用于标识关闭连接通道的Response。而后面两个Response类不是很常用，它们仅仅在对Socket连接进行限流时，才会派上用场，这里我就不具体展开讲了。</p><p>Okay，现在，我们看下Response相关的代码部分。</p><pre><code>abstract class Response(val request: Request) {
  locally {
    val nowNs = Time.SYSTEM.nanoseconds
    request.responseCompleteTimeNanos = nowNs
    if (request.apiLocalCompleteTimeNanos == -1L)
      request.apiLocalCompleteTimeNanos = nowNs
  }
  def processor: Int = request.processor
  def responseString: Option[String] = Some(&quot;&quot;)
  def onComplete: Option[Send =&gt; Unit] = None
  override def toString: String
}
</code></pre><p>这个抽象基类只有一个属性字段：request。这就是说，<strong>每个Response对象都要保存它对应的Request对象</strong>。我在前面说过，onComplete方法是调用指定回调逻辑的地方。SendResponse类就是复写（Override）了这个方法，如下所示：</p><pre><code>class SendResponse(request: Request,
                     val responseSend: Send,
                     val responseAsString: Option[String],
                     val onCompleteCallback: Option[Send =&gt; Unit]) 
  extends Response(request) {
    ......
    override def onComplete: Option[Send =&gt; Unit] = onCompleteCallback
}
</code></pre><p>这里的SendResponse类继承了Response父类，并重新定义了onComplete方法。复写的逻辑很简单，就是指定输入参数onCompleteCallback。其实方法本身没有什么可讲的，反倒是这里的Scala语法值得多说几句。</p><p>Scala中的Unit类似于Java中的void，而“Send =&gt; Unit”表示一个方法。这个方法接收一个Send类实例，然后执行一段代码逻辑。Scala是函数式编程语言，函数在Scala中是“一等公民”，因此，你可以把一个函数作为一个参数传给另一个函数，也可以把函数作为结果返回。这里的onComplete方法就应用了第二种用法，也就是把函数赋值给另一个函数，并作为结果返回。这样做的好处在于，你可以灵活地变更onCompleteCallback来实现不同的回调逻辑。</p><h2>RequestChannel</h2><p>RequestChannel，顾名思义，就是传输Request/Response的通道。有了Request和Response的基础，下面我们可以学习RequestChannel类的实现了。</p><p>我们先看下RequestChannel类的定义和重要的字段属性。</p><pre><code>class RequestChannel(val queueSize: Int, val metricNamePrefix : String) extends KafkaMetricsGroup {
  import RequestChannel._
  val metrics = new RequestChannel.Metrics
  private val requestQueue = new ArrayBlockingQueue[BaseRequest](queueSize)
  private val processors = new ConcurrentHashMap[Int, Processor]()
  val requestQueueSizeMetricName = metricNamePrefix.concat(RequestQueueSizeMetric)
  val responseQueueSizeMetricName = metricNamePrefix.concat(ResponseQueueSizeMetric)

  ......
}
</code></pre><p>RequestChannel类实现了KafkaMetricsGroup trait，后者封装了许多实用的指标监控方法，比如，newGauge方法用于创建数值型的监控指标，newHistogram方法用于创建直方图型的监控指标。</p><p><strong>就RequestChannel类本身的主体功能而言，它定义了最核心的3个属性：requestQueue、queueSize和processors</strong>。下面我分别解释下它们的含义。</p><p>每个RequestChannel对象实例创建时，会定义一个队列来保存Broker接收到的各类请求，这个队列被称为请求队列或Request队列。Kafka使用<strong>Java提供的阻塞队列ArrayBlockingQueue</strong>实现这个请求队列，并利用它天然提供的线程安全性来保证多个线程能够并发安全高效地访问请求队列。在代码中，这个队列由变量<span class="orange">requestQueue</span>定义。</p><p>而字段queueSize就是Request队列的最大长度。当Broker启动时，SocketServer组件会创建RequestChannel对象，并把Broker端参数queued.max.requests赋值给queueSize。因此，在默认情况下，每个RequestChannel上的队列长度是500。</p><p>字段processors封装的是RequestChannel下辖的Processor线程池。每个Processor线程负责具体的请求处理逻辑。下面我详细说说Processor的管理。</p><h3>Processor管理</h3><p>上面代码中的第六行创建了一个Processor线程池——当然，它是用Java的ConcurrentHashMap数据结构去保存的。Map中的Key就是前面我们说的processor序号，而Value则对应具体的Processor线程对象。</p><p>这个线程池的存在告诉了我们一个事实：<strong>当前Kafka Broker端所有网络线程都是在RequestChannel中维护的</strong>。既然创建了线程池，代码中必然要有管理线程池的操作。RequestChannel中的addProcessor和removeProcessor方法就是做这些事的。</p><pre><code>def addProcessor(processor: Processor): Unit = {
  // 添加Processor到Processor线程池  
  if (processors.putIfAbsent(processor.id, processor) != null)
    warn(s&quot;Unexpected processor with processorId ${processor.id}&quot;)
    newGauge(responseQueueSizeMetricName, 
      () =&gt; processor.responseQueueSize,
      // 为给定Processor对象创建对应的监控指标
      Map(ProcessorMetricTag -&gt; processor.id.toString))
}

def removeProcessor(processorId: Int): Unit = {
  processors.remove(processorId) // 从Processor线程池中移除给定Processor线程
  removeMetric(responseQueueSizeMetricName, Map(ProcessorMetricTag -&gt; processorId.toString)) // 移除对应Processor的监控指标
}
</code></pre><p>代码很简单，基本上就是调用ConcurrentHashMap的putIfAbsent和remove方法分别实现增加和移除线程。每当Broker启动时，它都会调用addProcessor方法，向RequestChannel对象添加num.network.threads个Processor线程。</p><p>如果查询Kafka官方文档的话，你就会发现，num.network.threads这个参数的更新模式（Update Mode）是Cluster-wide。这就说明，Kafka允许你动态地修改此参数值。比如，Broker启动时指定num.network.threads为8，之后你通过kafka-configs命令将其修改为3。显然，这个操作会减少Processor线程池中的线程数量。在这个场景下，removeProcessor方法会被调用。</p><h3>处理Request和Response</h3><p>除了Processor的管理之外，RequestChannel的另一个重要功能，是处理<strong>Request和Response</strong>，具体表现为收发Request和发送Response。比如，收发Request的方法有sendRequest和receiveRequest：</p><pre><code>def sendRequest(request: RequestChannel.Request): Unit = {
    requestQueue.put(request)
}
def receiveRequest(timeout: Long): RequestChannel.BaseRequest =
    requestQueue.poll(timeout, TimeUnit.MILLISECONDS)
def receiveRequest(): RequestChannel.BaseRequest =
    requestQueue.take()
</code></pre><p>所谓的发送Request，仅仅是将Request对象放置在Request队列中而已，而接收Request则是从队列中取出Request。整个流程构成了一个迷你版的“生产者-消费者”模式，然后依靠ArrayBlockingQueue的线程安全性来确保整个过程的线程安全，如下所示：</p><p><img src="https://static001.geekbang.org/resource/image/b8/cc/b83a2856a7f8e7b895f47e277e007ecc.jpg?wh=2054*915" alt=""></p><p>对于Response而言，则没有所谓的接收Response，只有发送Response，即sendResponse方法。sendResponse是啥意思呢？其实就是把Response对象发送出去，也就是将Response添加到Response队列的过程。</p><pre><code>def sendResponse(response: RequestChannel.Response): Unit = {
    if (isTraceEnabled) {  // 构造Trace日志输出字符串
      val requestHeader = response.request.header
      val message = response match {
        case sendResponse: SendResponse =&gt;
          s&quot;Sending ${requestHeader.apiKey} response to client ${requestHeader.clientId} of ${sendResponse.responseSend.size} bytes.&quot;
        case _: NoOpResponse =&gt;
          s&quot;Not sending ${requestHeader.apiKey} response to client ${requestHeader.clientId} as it's not required.&quot;
        case _: CloseConnectionResponse =&gt;
          s&quot;Closing connection for client ${requestHeader.clientId} due to error during ${requestHeader.apiKey}.&quot;
        case _: StartThrottlingResponse =&gt;
          s&quot;Notifying channel throttling has started for client ${requestHeader.clientId} for ${requestHeader.apiKey}&quot;
        case _: EndThrottlingResponse =&gt;
          s&quot;Notifying channel throttling has ended for client ${requestHeader.clientId} for ${requestHeader.apiKey}&quot;
      }
      trace(message)
    }
    // 找出response对应的Processor线程，即request当初是由哪个Processor线程处理的
    val processor = processors.get(response.processor)
    // 将response对象放置到对应Processor线程的Response队列中
    if (processor != null) {
      processor.enqueueResponse(response)
    }
}
</code></pre><p>sendResponse方法的逻辑其实非常简单。</p><p>前面的一大段if代码块仅仅是构造Trace日志要输出的内容。根据不同类型的Response，代码需要确定要输出的Trace日志内容。</p><p>接着，代码会找出Response对象对应的Processor线程。当Processor处理完某个Request后，会把自己的序号封装进对应的Response对象。一旦找出了之前是由哪个Processor线程处理的，代码直接调用该Processor的enqueueResponse方法，将Response放入Response队列中，等待后续发送。</p><h2>监控指标实现</h2><p>RequestChannel类还定义了丰富的监控指标，用于实时动态地监测Request和Response的性能表现。我们来看下具体指标项都有哪些。</p><pre><code>object RequestMetrics {
  val consumerFetchMetricName = ApiKeys.FETCH.name + &quot;Consumer&quot;
  val followFetchMetricName = ApiKeys.FETCH.name + &quot;Follower&quot;
  val RequestsPerSec = &quot;RequestsPerSec&quot;
  val RequestQueueTimeMs = &quot;RequestQueueTimeMs&quot;
  val LocalTimeMs = &quot;LocalTimeMs&quot;
  val RemoteTimeMs = &quot;RemoteTimeMs&quot;
  val ThrottleTimeMs = &quot;ThrottleTimeMs&quot;
  val ResponseQueueTimeMs = &quot;ResponseQueueTimeMs&quot;
  val ResponseSendTimeMs = &quot;ResponseSendTimeMs&quot;
  val TotalTimeMs = &quot;TotalTimeMs&quot;
  val RequestBytes = &quot;RequestBytes&quot;
  val MessageConversionsTimeMs = &quot;MessageConversionsTimeMs&quot;
  val TemporaryMemoryBytes = &quot;TemporaryMemoryBytes&quot;
  val ErrorsPerSec = &quot;ErrorsPerSec&quot;
}
</code></pre><p>可以看到，指标有很多，不过别有压力，我们只要掌握几个重要的就行了。</p><ul>
<li><strong>RequestsPerSec</strong>：每秒处理的Request数，用来评估Broker的繁忙状态。</li>
<li><strong>RequestQueueTimeMs</strong>：计算Request在Request队列中的平均等候时间，单位是毫秒。倘若Request在队列的等待时间过长，你通常需要增加后端I/O线程的数量，来加快队列中Request的拿取速度。</li>
<li><strong>LocalTimeMs</strong>：计算Request实际被处理的时间，单位是毫秒。一旦定位到这个监控项的值很大，你就需要进一步研究Request被处理的逻辑了，具体分析到底是哪一步消耗了过多的时间。</li>
<li><strong>RemoteTimeMs</strong>：Kafka的读写请求（PRODUCE请求和FETCH请求）逻辑涉及等待其他Broker操作的步骤。RemoteTimeMs计算的，就是等待其他Broker完成指定逻辑的时间。因为等待的是其他Broker，因此被称为Remote Time。这个监控项非常重要！Kafka生产环境中设置acks=all的Producer程序发送消息延时高的主要原因，往往就是Remote Time高。因此，如果你也碰到了这样的问题，不妨先定位一下Remote Time是不是瓶颈。</li>
<li><strong>TotalTimeMs</strong>：计算Request被处理的完整流程时间。<strong>这是最实用的监控指标，没有之一！</strong>毕竟，我们通常都是根据TotalTimeMs来判断系统是否出现问题的。一旦发现了问题，我们才会利用前面的几个监控项进一步定位问题的原因。</li>
</ul><p>RequestChannel定义了updateMetrics方法，用于实现监控项的更新，因为逻辑非常简单，我就不展开说了，你可以自己阅读一下。</p><h2>总结</h2><p>好了，又到了总结时间。</p><p>今天，我带你阅读了Kafka请求队列的实现源码。围绕这个问题，我们学习了几个重点内容。</p><ul>
<li>Request：定义了Kafka Broker支持的各类请求。</li>
<li>Response：定义了与Request对应的各类响应。</li>
<li>RequestChannel：实现了Kafka Request队列。</li>
<li>监控指标：封装了与Request队列相关的重要监控指标。</li>
</ul><p><img src="https://static001.geekbang.org/resource/image/4b/15/4bf7b31f368743496018b3f21a528b15.jpg?wh=2284*1921" alt=""></p><p>希望你结合今天所讲的内容思考一下，Request和Response在请求通道甚至是SocketServer中的流转过程，这将极大地帮助你了解Kafka是如何处理外部发送的请求的。当然，如果你觉得这个有难度，也不必着急，因为后面我会专门用一节课来告诉你这些内容。</p><h2>课后讨论</h2><p>如果我想监控Request队列当前的使用情况（比如当前已保存了多少个Request），你可以结合源码指出，应该使用哪个监控指标吗？</p><p>欢迎你在留言区畅所欲言，跟我交流讨论，也欢迎你把今天的内容分享给你的朋友。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/a7/9a/495cb99a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>胡夕</span>
  </div>
  <div class="_2_QraFYR_0">你好，我是胡夕。我来公布上节课的“课后讨论”题答案啦～<br><br>上节课，咱们重点了解了Kafka中位移索引和时间戳索引的不同。课后我让你尝试自行编写一个函数，该函数实现类似于CEILING 函数的位移查找逻辑，即返回不小于给定位移值 targetOffset 的最小位移值和对应的物理文件位置。以下是我给出的代码：<br>def ceilingLookup(targetOffset: Long): OffsetPosition = {<br>    maybeLock(lock) {<br>      val idx = mmap.duplicate<br>      val slot = smallestUpperBoundSlotFor(idx, targetOffset, IndexSearchType.KEY)<br>      if (slot == -1)<br>        OffsetPosition(baseOffset, 0)<br>      else<br>        parseEntry(idx, slot)<br>    }<br>  }<br><br><br>okay，你是怎么考虑的呢？我们可以一起讨论下。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-06 13:49:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/aa/6b/ab9a072a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>对与错</span>
  </div>
  <div class="_2_QraFYR_0">Kafka的异步处理机制应该不能保证消息的有序吧，比如哪怕只有一个分区，生产者发送10个消息，通过acceptor轮询给不同的Processor去处理，然后Processor最终处理的顺序不同，发送给RequestChannel的顺序也不同，那最后消费的顺序岂不是也不相同了?</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-21 18:31:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/23/37/61/51e10a30.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>QAQ</span>
  </div>
  <div class="_2_QraFYR_0">为啥这里使用的队列是arrayblockqueue，是否可以考虑下阻塞的双端队列，一边进另一边出，降低竞争</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-14 00:20:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/37/2b/b32f1d66.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Ball</span>
  </div>
  <div class="_2_QraFYR_0">思考题：监控 Request 队列当前的使用情况，应该是 newGauge(requestQueueSizeMetricName, () =&gt; requestQueue.size) 这个方法里面，requestQueueSizeMetricName 这个指标。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个指标名称实际上是RequestQueueSize：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-04 07:46:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/84/4b/e4738ba8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>delete is create</span>
  </div>
  <div class="_2_QraFYR_0">感觉kafka中大量使用协调器（findCoordinator）      老师协调器到底是做什么的  我今天看一个kafkaAdminClient中获取消费者组信息里面也有这个东西  </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 协调器是做协调用的：） hmmm... sorry，哈哈哈<br>目前Kafka中有两类协调器，GroupCoordinator和TransactionCoordinator。前者用于管理消费者组，后者是用于事务管理的。就以消费者组来说，Coordinator需要生成消费者组的元数据信息，负责维护组的位移提交和获取以及全套的rebalance流程支持。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-28 15:33:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/21/63/6e/6b971571.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Z宇锤锤</span>
  </div>
  <div class="_2_QraFYR_0">RequestChannel中定义了broker接收到的请求和response。这里有一个有趣的点就是RequestChannel类似于一个中继管理者或者枢纽的形象。因为它管理着请求和response的来往，但是不执行具体IO。<br>还有一个细节就是所有的请求使用一个阻塞队列来实现，response分processId存储在不同的阻塞队列上。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-22 20:00:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTI3lJFX3XBF3C0qriazEcv7rPsdJZCwz0bkmP5M37pa1IJr7G5LNQevXLFBzPpOZLzNZnybN0bNPhg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lucas</span>
  </div>
  <div class="_2_QraFYR_0">老师好，请教一个问题，我flink1.12.1写入kafka2.3.0的时候报错，而且是运行几个小时之后才会报<br>Transiting to fatal error state due to org.apache.kafka.common.errors.InvalidPidMappingException: The producer attempted to use a producer id which is not currently assigned to its transactional id.<br>Transiting to fatal error state due to org.apache.kafka.common.errors.UnsupportedVersionException: Attempted to write a non-default producerId at version 1</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 还是版本不匹配的问题</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-28 12:12:30</div>
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
  <div class="_2_QraFYR_0">思考题：（RequestQueueTimeMs&#47;1000）* RequestsPerSec</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-08 21:43:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/I6CEbicmiag9icicg9icJfUiajZ0zuZTiciaYhwMyfo6VMfLtqrxicOIvmmibIwRDFpGRBO0VWeMZCAUZ6Jbv22emZRCoYAQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_ae1f59</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，我们kafka集群responseSendTimeMs监控指标非常大，而且部分节点还抛出异常：java.io.IOException: Connection to xx was disconnected before the response was read，关于这个有什么排查思路吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 排查网络的问题吧</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-11 09:12:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/88/d7/07f8bc6c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>sljoai</span>
  </div>
  <div class="_2_QraFYR_0">“StartThrottlingResponse：用于通知 Broker 的 Socket Server 组件某个 TCP 连接通信通道开始被限流（throttling). ”这里的限流如何理解呢？是物理网络上的限制，还是可以在Kafka中设置的？！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Kafka实现的。基本上可以认为是人为引入延时</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-10 23:19:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/76/23/31e5e984.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>空知</span>
  </div>
  <div class="_2_QraFYR_0"> 用kafka-configs 命令把Processor 线程池中的线程数量减少 会导致响应丢失的吧，这个不用处理或者警告限制嘛？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 的确可能丢失，到时候客户端自行处理。不过我同意应该加个告警。如果你有兴趣可以提交一个patch给社区，修改RequestChannel.scala的sendResponse方法代码即可：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-05 20:36:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/dc/3e/abb7bfe3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小崔</span>
  </div>
  <div class="_2_QraFYR_0">对于一个特定的请求，TotalTimeMs = RequestQueueTimeMs + LocalTimeMs + RemoteTimeMs，对吗？还是说LocalTimeMs包含了（RemoteTimeMs + 本地处理时间）？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 前一个是对的。local time是指本地broker写入磁盘的时间</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-05 16:47:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/qsmAdOC3R3twep9xwiboiaNF6u3fk5jNZGibKrBuILKgyNMH0DAQMg3liaWQ7ntVAFGEBCg5uB9y9KdKrhD65TyGgQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>镜子中间</span>
  </div>
  <div class="_2_QraFYR_0">这节课的信息量很大，干货满满，就是听完下来云里雾里，打算对照源码和老师的讲解梳理一遍</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 加油加油，如果有问题就提出来， 我第一时间回复</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-01 22:12:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/97/18/a5218104.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>🐾</span>
  </div>
  <div class="_2_QraFYR_0">胡总下午好，专栏后面会介绍如何在IDEA进行调试吗？想看下客户端发送消息到服务端的整个流程，再结合文章介绍的功能点，功力提升效果会比较好些</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: hmmm，实话说本专栏聚焦于Broker端代码。客户端染指的很少。不过你可以将整个流程拆成broker端调试+clients端调试。特别是broker端调试，直接在KafkaApis.scala中打断点就可以</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-29 14:43:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/78/d3/1dc40aa2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Jonah</span>
  </div>
  <div class="_2_QraFYR_0">指标应该就是metricNamePrefix + RequestQueueSize</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以再思考看看，后面我也给出我的答案：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-29 11:41:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>kai</span>
  </div>
  <div class="_2_QraFYR_0">请问，如果定位到TotalTimeMs波动比较大，尖峰比较多，而且一般都在5s左右，然后深入发现，时间主要是 reaponse queue time ，看了一下网络线程池空闲比例也在80%以上，说明网络线程也不是很忙碌，这个有可能是因为什么原因呢？如何判断集群是否正常？TotalTimeMs 一般如何表现才是正常的？谢谢老师</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 看下local time和remote time，另外网络传输时间也需要监控下。这些都是构成total time的要素</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-28 07:56:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/84/4b/e4738ba8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>delete is create</span>
  </div>
  <div class="_2_QraFYR_0">老师 使用kafkaAdminClient获取消费者连接状态超时是什么问题?  kafka服务器的内存和cpu都占用不高<br>DescribeConsumerGroupsResult result = getAdminClient().describeConsumerGroups(groupIdList);<br>Map&lt;String, ConsumerGroupDescription&gt; stringConsumerGroupDescriptionMap = result.all().get(20, TimeUnit.SECONDS);&#47;&#47; get方法报了超时异常TimeOutException </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 希望给出完整的stacktrace。通常情况下都是因为无法连接到Coordinator所在的broker导致的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-28 05:43:45</div>
  </div>
</div>
</div>
</li>
</ul>