<audio title="07 _ SocketServer（上）：Kafka到底是怎么应用NIO实现网络通信的？" src="https://static001.geekbang.org/resource/audio/3f/8c/3f354d9a26a03aeb946bbefca5cbd88c.mp3" controls="controls"></audio> 
<p>你好，我是胡夕。这节课我们来说说Kafka底层的NIO通信机制源码。</p><p>在谈到Kafka高性能、高吞吐量实现原理的时候，很多人都对它使用了Java NIO这件事津津乐道。实际上，搞懂“Kafka究竟是怎么应用NIO来实现网络通信的”，不仅是我们掌握Kafka请求全流程处理的前提条件，对我们了解Reactor模式的实现大有裨益，而且还能帮助我们解决很多实际问题。</p><p>比如说，当Broker处理速度很慢、需要优化的时候，你只有明确知道SocketServer组件的工作原理，才能制定出恰当的解决方案，并有针对性地给出对应的调优参数。</p><p>那么，今天，我们就一起拿下这个至关重要的NIO通信机制吧。</p><h2>网络通信层</h2><p>在深入学习Kafka各个网络组件之前，我们先从整体上看一下完整的网络通信层架构，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/52/e8/52c3226ad4736751b4b1ccfcb2a09ee8.jpg?wh=2079*2053" alt=""></p><p>可以看出，Kafka网络通信组件主要由两大部分构成：<strong>SocketServer</strong>和<strong>KafkaRequestHandlerPool</strong>。</p><p><strong>SocketServer组件是核心</strong>，主要实现了Reactor模式，用于处理外部多个Clients（这里的Clients指的是广义的Clients，可能包含Producer、Consumer或其他Broker）的并发请求，并负责将处理结果封装进Response中，返还给Clients。</p><!-- [[[read_end]]] --><p><strong>KafkaRequestHandlerPool组件就是我们常说的I/O线程池</strong>，里面定义了若干个I/O线程，用于执行真实的请求处理逻辑。</p><p>两者的交互点在于SocketServer中定义的RequestChannel对象和Processor线程。对了，我所说的线程，在代码中本质上都是Runnable类型，不管是Acceptor类、Processor类，还是后面我们会单独讨论的KafkaRequestHandler类。</p><p>讲到这里，我稍微提示你一下。在第9节课，我会给出KafkaRequestHandlerPool线程池的详细介绍。但你现在需要知道的是，KafkaRequestHandlerPool线程池定义了多个KafkaRequestHandler线程，而KafkaRequestHandler线程是真正处理请求逻辑的地方。和KafkaRequestHandler相比，今天所说的Acceptor和Processor线程从某种意义上来说，只能算是请求和响应的“搬运工”罢了。</p><p>了解了完整的网络通信层架构之后，我们要重点关注一下SocketServer组件。<strong>这个组件是Kafka网络通信层中最重要的子模块。它下辖的Acceptor线程、Processor线程和RequestChannel等对象，都是实施网络通信的重要组成部分</strong>。你可能会感到意外的是，这套线程组合在源码中有多套，分别具有不同的用途。在下节课，我会具体跟你分享一下，不同的线程组合会被应用到哪些实际场景中。</p><p>下面我们进入到SocketServer组件的学习。</p><h2>SocketServer概览</h2><p>SocketServer组件的源码位于Kafka工程的core包下，具体位置是src/main/scala/kafka/network路径下的SocketServer.scala文件。</p><p>SocketServer.scala可谓是元老级的源码文件了。在Kafka的源码演进历史中，很多代码文件进进出出，这个文件却一直“坚强地活着”，而且还在不断完善。如果翻开它的Git修改历史，你会发现，它最早的修改提交历史可回溯到2011年8月，足见它的资历之老。</p><p>目前，SocketServer.scala文件是一个近2000行的大文件，共有8个代码部分。我使用一张思维导图帮你梳理下：</p><p><img src="https://static001.geekbang.org/resource/image/18/18/18b39a0fe7817bf6c2344bf6b49eaa18.jpg?wh=2569*3387" alt=""></p><p>乍一看组件有很多，但你也不必担心，我先对这些组件做个简单的介绍，然后我们重点学习一下Acceptor类和Processor类的源码。毕竟，<strong>这两个类是实现网络通信的关键部件</strong>。另外，今天我给出的都是SocketServer组件的基本情况介绍，下节课我再详细向你展示它的定义。</p><p>1.<strong>AbstractServerThread类</strong>：这是Acceptor线程和Processor线程的抽象基类，定义了这两个线程的公有方法，如shutdown（关闭线程）等。我不会重点展开这个抽象类的代码，但你要重点关注下CountDownLatch类在线程启动和线程关闭时的作用。</p><p>如果你苦于寻找Java线程安全编程的最佳实践案例，那一定不要错过CountDownLatch这个类。Kafka中的线程控制代码大量使用了基于CountDownLatch的编程技术，依托于它来实现优雅的线程启动、线程关闭等操作。因此，我建议你熟练掌握它们，并应用到你日后的工作当中去。</p><p>2.<strong>Acceptor线程类</strong>：这是接收和创建外部TCP连接的线程。每个SocketServer实例只会创建一个Acceptor线程。它的唯一目的就是创建连接，并将接收到的Request传递给下游的Processor线程处理。</p><p>3.<strong>Processor线程类</strong>：这是处理单个TCP连接上所有请求的线程。每个SocketServer实例默认创建若干个（num.network.threads）Processor线程。Processor线程负责将接收到的Request添加到RequestChannel的Request队列上，同时还负责将Response返还给Request发送方。</p><p>4.<strong>Processor伴生对象类</strong>：仅仅定义了一些与Processor线程相关的常见监控指标和常量等，如Processor线程空闲率等。</p><p>5.<strong>ConnectionQuotas类</strong>：是控制连接数配额的类。我们能够设置单个IP创建Broker连接的最大数量，以及单个Broker能够允许的最大连接数。</p><p>6.<strong>TooManyConnectionsException类</strong>：SocketServer定义的一个异常类，用于标识连接数配额超限情况。</p><p>7.<strong>SocketServer类</strong>：实现了对以上所有组件的管理和操作，如创建和关闭Acceptor、Processor线程等。</p><p>8.<strong>SocketServer伴生对象类</strong>：定义了一些有用的常量，同时明确了SocketServer组件中的哪些参数是允许动态修改的。</p><h2>Acceptor线程</h2><p>经典的Reactor模式有个Dispatcher的角色，接收外部请求并分发给下面的实际处理线程。在Kafka中，这个Dispatcher就是Acceptor线程。</p><p>我们看下它的定义：</p><pre><code>private[kafka] class Acceptor(val endPoint: EndPoint,
                              val sendBufferSize: Int,
                              val recvBufferSize: Int,
                              brokerId: Int,
                              connectionQuotas: ConnectionQuotas,
                              metricPrefix: String) extends AbstractServerThread(connectionQuotas) with KafkaMetricsGroup {
  // 创建底层的NIO Selector对象
  // Selector对象负责执行底层实际I/O操作，如监听连接创建请求、读写请求等
  private val nioSelector = NSelector.open() 
  // Broker端创建对应的ServerSocketChannel实例
  // 后续把该Channel向上一步的Selector对象注册
  val serverChannel = openServerSocket(endPoint.host, endPoint.port)
  // 创建Processor线程池，实际上是Processor线程数组
  private val processors = new ArrayBuffer[Processor]()
  private val processorsStarted = new AtomicBoolean

  private val blockedPercentMeter = newMeter(s&quot;${metricPrefix}AcceptorBlockedPercent&quot;,
    &quot;blocked time&quot;, TimeUnit.NANOSECONDS, Map(ListenerMetricTag -&gt; endPoint.listenerName.value))
  ......
}
</code></pre><p>从定义来看，Acceptor线程接收5个参数，其中比较重要的有3个。</p><ul>
<li><strong>endPoint</strong>。它就是你定义的Kafka Broker连接信息，比如PLAINTEXT://localhost:9092。Acceptor需要用到endPoint包含的主机名和端口信息创建Server Socket。</li>
<li><strong>sendBufferSize</strong>。它设置的是SocketOptions的SO_SNDBUF，即用于设置出站（Outbound）网络I/O的底层缓冲区大小。该值默认是Broker端参数socket.send.buffer.bytes的值，即100KB。</li>
<li><strong>recvBufferSize</strong>。它设置的是SocketOptions的SO_RCVBUF，即用于设置入站（Inbound）网络I/O的底层缓冲区大小。该值默认是Broker端参数socket.receive.buffer.bytes的值，即100KB。</li>
</ul><p>说到这儿，我想给你提一个优化建议。如果在你的生产环境中，Clients与Broker的通信网络延迟很大（比如RTT&gt;10ms），那么我建议你调大控制缓冲区大小的两个参数，也就是sendBufferSize和recvBufferSize。通常来说，默认值100KB太小了。</p><p>除了类定义的字段，Acceptor线程还有两个非常关键的自定义属性。</p><ul>
<li><strong>nioSelector</strong>：是Java NIO库的Selector对象实例，也是后续所有网络通信组件实现Java NIO机制的基础。如果你不熟悉Java NIO，那么我推荐你学习这个系列教程：<a href="http://tutorials.jenkov.com/java-nio/index.html">Java NIO</a>。</li>
<li><strong>processors</strong>：网络Processor线程池。Acceptor线程在初始化时，需要创建对应的网络Processor线程池。可见，Processor线程是在Acceptor线程中管理和维护的。</li>
</ul><p>既然如此，那它就必须要定义相关的方法。Acceptor代码中，提供了3个与Processor相关的方法，分别是addProcessors、startProcessors和removeProcessors。鉴于它们的代码都非常简单，我用注释的方式给出主体逻辑的步骤：</p><h3>addProcessors</h3><pre><code>private[network] def addProcessors(
  newProcessors: Buffer[Processor], processorThreadPrefix: String): Unit = synchronized {
  processors ++= newProcessors // 添加一组新的Processor线程
  if (processorsStarted.get) // 如果Processor线程池已经启动
    startProcessors(newProcessors, processorThreadPrefix) // 启动新的Processor线程
}
</code></pre><h3>startProcessors</h3><pre><code>private[network] def startProcessors(processorThreadPrefix: String): Unit = synchronized {
    if (!processorsStarted.getAndSet(true)) {  // 如果Processor线程池未启动
      startProcessors(processors, processorThreadPrefix) // 启动给定的Processor线程
    }
}

private def startProcessors(processors: Seq[Processor], processorThreadPrefix: String): Unit = synchronized {
  processors.foreach { processor =&gt; // 依次创建并启动Processor线程
  // 线程命名规范：processor线程前缀-kafka-network-thread-broker序号-监听器名称-安全协议-Processor序号
  // 假设为序号为0的Broker设置PLAINTEXT://localhost:9092作为连接信息，那么3个Processor线程名称分别为：
  // data-plane-kafka-network-thread-0-ListenerName(PLAINTEXT)-PLAINTEXT-0
  // data-plane-kafka-network-thread-0-ListenerName(PLAINTEXT)-PLAINTEXT-1
  // data-plane-kafka-network-thread-0-ListenerName(PLAINTEXT)-PLAINTEXT-2
  KafkaThread.nonDaemon(s&quot;${processorThreadPrefix}-kafka-network-thread-$brokerId-${endPoint.listenerName}-${endPoint.securityProtocol}-${processor.id}&quot;, processor).start()
  }
}

</code></pre><h3>removeProcessors</h3><pre><code>private[network] def removeProcessors(removeCount: Int, requestChannel: RequestChannel): Unit = synchronized {
  // 获取Processor线程池中最后removeCount个线程
  val toRemove = processors.takeRight(removeCount)
  // 移除最后removeCount个线程
  processors.remove(processors.size - removeCount, removeCount)
  // 关闭最后removeCount个线程
  toRemove.foreach(_.shutdown())
  // 在RequestChannel中移除这些Processor
  toRemove.foreach(processor =&gt; requestChannel.removeProcessor(processor.id))
}
</code></pre><p>为了更加形象地展示这些方法的逻辑，我画了一张图，它同时包含了这3个方法的执行流程，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/49/2b/494e5bac80f19a2533bb9e7b30003e2b.jpg?wh=3110*1774" alt=""></p><p>刚才我们学到的addProcessors、startProcessors和removeProcessors方法是管理Processor线程用的。应该这么说，有了这三个方法，Acceptor类就具备了基本的Processor线程池管理功能。不过，<strong>Acceptor类逻辑的重头戏其实是run方法，它是处理Reactor模式中分发逻辑的主要实现方法</strong>。下面我使用注释的方式给出run方法的大体运行逻辑，如下所示：</p><pre><code>def run(): Unit = {
  //注册OP_ACCEPT事件
  serverChannel.register(nioSelector, SelectionKey.OP_ACCEPT)
  // 等待Acceptor线程启动完成
  startupComplete()
  try {
    // 当前使用的Processor序号，从0开始，最大值是num.network.threads - 1
    var currentProcessorIndex = 0
    while (isRunning) {
      try {
        // 每500毫秒获取一次就绪I/O事件
        val ready = nioSelector.select(500)
        if (ready &gt; 0) { // 如果有I/O事件准备就绪
          val keys = nioSelector.selectedKeys()
          val iter = keys.iterator()
          while (iter.hasNext &amp;&amp; isRunning) {
            try {
              val key = iter.next
              iter.remove()
              if (key.isAcceptable) {
                // 调用accept方法创建Socket连接
                accept(key).foreach { socketChannel =&gt;
                  var retriesLeft = synchronized(processors.length)
                  var processor: Processor = null
                  do {
                    retriesLeft -= 1
                    // 指定由哪个Processor线程进行处理
                    processor = synchronized {
                      currentProcessorIndex = currentProcessorIndex % processors.length
                      processors(currentProcessorIndex)
                    }
                    // 更新Processor线程序号
                    currentProcessorIndex += 1
                  } while (!assignNewConnection(socketChannel, processor, retriesLeft == 0)) // Processor是否接受了该连接
                }
              } else
                throw new IllegalStateException(&quot;Unrecognized key state for acceptor thread.&quot;)
            } catch {
              case e: Throwable =&gt; error(&quot;Error while accepting connection&quot;, e)
            }
          }
        }
      }
      catch {
        case e: ControlThrowable =&gt; throw e
        case e: Throwable =&gt; error(&quot;Error occurred&quot;, e)
      }
    }
  } finally { // 执行各种资源关闭逻辑
    debug(&quot;Closing server socket and selector.&quot;)
    CoreUtils.swallow(serverChannel.close(), this, Level.ERROR)
    CoreUtils.swallow(nioSelector.close(), this, Level.ERROR)
    shutdownComplete()
  }
}

</code></pre><p>看上去代码似乎有点多，我再用一张图来说明一下run方法的主要处理逻辑吧。这里的关键点在于，Acceptor线程会先为每个入站请求确定要处理它的Processor线程，然后调用assignNewConnection方法令Processor线程创建与发送方的连接。</p><p><img src="https://static001.geekbang.org/resource/image/1c/55/1c8320c702c1e18b37b992cadc61d555.jpg?wh=1463*2604" alt=""></p><p>基本上，Acceptor线程使用Java NIO的Selector + SocketChannel的方式循环地轮询准备就绪的I/O事件。这里的I/O事件，主要是指网络连接创建事件，即代码中的SelectionKey.OP_ACCEPT。一旦接收到外部连接请求，Acceptor就会指定一个Processor线程，并将该请求交由它，让它创建真正的网络连接。总的来说，Acceptor线程就做这么点事。</p><h2>Processor线程</h2><p>下面我们进入到Processor线程源码的学习。</p><p><strong>如果说Acceptor是做入站连接处理的，那么，Processor代码则是真正创建连接以及分发请求的地方</strong>。显然，它要做的事情远比Acceptor要多得多。我先给出Processor线程的run方法，你大致感受一下：</p><pre><code>override def run(): Unit = {
    startupComplete() // 等待Processor线程启动完成
    try {
      while (isRunning) {
        try {
          configureNewConnections() // 创建新连接
          // register any new responses for writing
          processNewResponses() // 发送Response，并将Response放入到inflightResponses临时队列
          poll() // 执行NIO poll，获取对应SocketChannel上准备就绪的I/O操作
          processCompletedReceives() // 将接收到的Request放入Request队列
          processCompletedSends() // 为临时Response队列中的Response执行回调逻辑
          processDisconnected() // 处理因发送失败而导致的连接断开
          closeExcessConnections() // 关闭超过配额限制部分的连接
        } catch {
          case e: Throwable =&gt; processException(&quot;Processor got uncaught exception.&quot;, e)
        }
      }
    } finally { // 关闭底层资源
      debug(s&quot;Closing selector - processor $id&quot;)
      CoreUtils.swallow(closeAll(), this, Level.ERROR)
      shutdownComplete()
    }
}
</code></pre><p>run方法逻辑被切割得相当好，各个子方法的边界非常清楚。因此，从整体上看，该方法呈现出了面向对象领域中非常难得的封装特性。我使用一张图来展示下该方法要做的事情：</p><p><img src="https://static001.geekbang.org/resource/image/1d/42/1d6f59036ea2797bfc1591f57980df42.jpg?wh=1988*6258" alt=""></p><p>在详细说run方法之前，我们先来看下Processor线程初始化时要做的事情。</p><p>每个Processor线程在创建时都会创建3个队列。注意，这里的队列是广义的队列，其底层使用的数据结构可能是阻塞队列，也可能是一个Map对象而已，如下所示：</p><pre><code>private val newConnections = new ArrayBlockingQueue[SocketChannel](connectionQueueSize)
private val inflightResponses = mutable.Map[String, RequestChannel.Response]()
private val responseQueue = new LinkedBlockingDeque[RequestChannel.Response]()
</code></pre><p><strong>队列一：newConnections</strong></p><p><strong>它保存的是要创建的新连接信息</strong>，具体来说，就是SocketChannel对象。这是一个默认上限是20的队列，而且，目前代码中硬编码了队列的长度，因此，你无法变更这个队列的长度。</p><p>每当Processor线程接收新的连接请求时，都会将对应的SocketChannel放入这个队列。后面在创建连接时（也就是调用configureNewConnections时），就从该队列中取出SocketChannel，然后注册新的连接。</p><p><strong>队列二：inflightResponses</strong></p><p>严格来说，这是一个临时Response队列。当Processor线程将Response返还给Request发送方之后，还要将Response放入这个临时队列。</p><p>为什么需要这个临时队列呢？这是因为，有些Response回调逻辑要在Response被发送回发送方之后，才能执行，因此需要暂存在一个临时队列里面。这就是inflightResponses存在的意义。</p><p><strong>队列三：responseQueue</strong></p><p>看名字我们就可以知道，这是Response队列，而不是Request队列。这告诉了我们一个事实：<strong>每个Processor线程都会维护自己的Response队列</strong>，而不是像网上的某些文章说的，Response队列是线程共享的或是保存在RequestChannel中的。Response队列里面保存着需要被返还给发送方的所有Response对象。</p><p>好了，了解了这些之后，现在我们来深入地查看一下Processor线程的工作逻辑。根据run方法中的方法调用顺序，我先来介绍下configureNewConnections方法。</p><h3>configureNewConnections</h3><p>就像我前面所说的，configureNewConnections负责处理新连接请求。接下来，我用注释的方式给出这个方法的主体逻辑：</p><pre><code>private def configureNewConnections(): Unit = {
    var connectionsProcessed = 0 // 当前已配置的连接数计数器
    while (connectionsProcessed &lt; connectionQueueSize &amp;&amp; !newConnections.isEmpty) { // 如果没超配额并且有待处理新连接
      val channel = newConnections.poll() // 从连接队列中取出SocketChannel
      try {
        debug(s&quot;Processor $id listening to new connection from ${channel.socket.getRemoteSocketAddress}&quot;)
        // 用给定Selector注册该Channel
        // 底层就是调用Java NIO的SocketChannel.register(selector, SelectionKey.OP_READ)
        selector.register(connectionId(channel.socket), channel)
        connectionsProcessed += 1 // 更新计数器
      } catch {
        case e: Throwable =&gt;
          val remoteAddress = channel.socket.getRemoteSocketAddress
          close(listenerName, channel)
          processException(s&quot;Processor $id closed connection from $remoteAddress&quot;, e)
      }
    }
}

</code></pre><p><strong>该方法最重要的逻辑是调用selector的register来注册SocketChannel</strong>。每个Processor线程都维护了一个Selector类实例。Selector类是社区提供的一个基于Java NIO Selector的接口，用于执行非阻塞多通道的网络I/O操作。在核心功能上，Kafka提供的Selector和Java提供的是一致的。</p><h3>processNewResponses</h3><p>它负责发送Response给Request发送方，并且将Response放入临时Response队列。处理逻辑如下：</p><pre><code>private def processNewResponses(): Unit = {
    var currentResponse: RequestChannel.Response = null
    while ({currentResponse = dequeueResponse(); currentResponse != null}) { // Response队列中存在待处理Response
      val channelId = currentResponse.request.context.connectionId // 获取连接通道ID
      try {
        currentResponse match {
          case response: NoOpResponse =&gt; // 无需发送Response
            updateRequestMetrics(response)
            trace(s&quot;Socket server received empty response to send, registering for read: $response&quot;)
            handleChannelMuteEvent(channelId, ChannelMuteEvent.RESPONSE_SENT)
            tryUnmuteChannel(channelId)
          case response: SendResponse =&gt; // 发送Response并将Response放入inflightResponses
            sendResponse(response, response.responseSend)
          case response: CloseConnectionResponse =&gt; // 关闭对应的连接
            updateRequestMetrics(response)
            trace(&quot;Closing socket connection actively according to the response code.&quot;)
            close(channelId)
          case _: StartThrottlingResponse =&gt;
            handleChannelMuteEvent(channelId, ChannelMuteEvent.THROTTLE_STARTED)
          case _: EndThrottlingResponse =&gt;
            handleChannelMuteEvent(channelId, ChannelMuteEvent.THROTTLE_ENDED)
            tryUnmuteChannel(channelId)
          case _ =&gt;
            throw new IllegalArgumentException(s&quot;Unknown response type: ${currentResponse.getClass}&quot;)
        }
      } catch {
        case e: Throwable =&gt;
          processChannelException(channelId, s&quot;Exception while processing response for $channelId&quot;, e)
      }
    }
}
</code></pre><p>这里的关键是<strong>SendResponse分支上的sendResponse方法</strong>。这个方法的核心代码其实只有三行：</p><pre><code>if (openOrClosingChannel(connectionId).isDefined) { // 如果该连接处于可连接状态
  selector.send(responseSend) // 发送Response
  inflightResponses += (connectionId -&gt; response) // 将Response加入到inflightResponses队列
}
</code></pre><h3>poll</h3><p>严格来说，上面提到的所有发送的逻辑都不是执行真正的发送。真正执行I/O动作的方法是这里的poll方法。</p><p>poll方法的核心代码就只有1行：<strong>selector.poll(pollTimeout)</strong>。在底层，它实际上调用的是Java NIO Selector的select方法去执行那些准备就绪的I/O操作，不管是接收Request，还是发送Response。因此，你需要记住的是，<strong>poll方法才是真正执行I/O操作逻辑的地方</strong>。</p><h3>processCompletedReceives</h3><p>它是接收和处理Request的。代码如下：</p><pre><code>private def processCompletedReceives(): Unit = {
  // 遍历所有已接收的Request
  selector.completedReceives.asScala.foreach { receive =&gt;
    try {
      // 保证对应连接通道已经建立
      openOrClosingChannel(receive.source) match {
        case Some(channel) =&gt;
          val header = RequestHeader.parse(receive.payload)
          if (header.apiKey == ApiKeys.SASL_HANDSHAKE &amp;&amp; channel.maybeBeginServerReauthentication(receive, nowNanosSupplier))
            trace(s&quot;Begin re-authentication: $channel&quot;)
          else {
            val nowNanos = time.nanoseconds()
            // 如果认证会话已过期，则关闭连接
            if (channel.serverAuthenticationSessionExpired(nowNanos)) {
              debug(s&quot;Disconnecting expired channel: $channel : $header&quot;)
              close(channel.id)
              expiredConnectionsKilledCount.record(null, 1, 0)
            } else {
              val connectionId = receive.source
              val context = new RequestContext(header, connectionId, channel.socketAddress,
                channel.principal, listenerName, securityProtocol,
                channel.channelMetadataRegistry.clientInformation)
              val req = new RequestChannel.Request(processor = id, context = context,
                startTimeNanos = nowNanos, memoryPool, receive.payload, requestChannel.metrics)
              if (header.apiKey == ApiKeys.API_VERSIONS) {
                val apiVersionsRequest = req.body[ApiVersionsRequest]
                if (apiVersionsRequest.isValid) {
                  channel.channelMetadataRegistry.registerClientInformation(new ClientInformation(
                    apiVersionsRequest.data.clientSoftwareName,
                    apiVersionsRequest.data.clientSoftwareVersion))
                }
              }
              // 核心代码：将Request添加到Request队列
              requestChannel.sendRequest(req)
              selector.mute(connectionId)
              handleChannelMuteEvent(connectionId, ChannelMuteEvent.REQUEST_RECEIVED)
            }
          }
        case None =&gt;
          throw new IllegalStateException(s&quot;Channel ${receive.source} removed from selector before processing completed receive&quot;)
      }
    } catch {
      case e: Throwable =&gt;
        processChannelException(receive.source, s&quot;Exception while processing request from ${receive.source}&quot;, e)
    }
  }
}

</code></pre><p>看上去代码有很多，但其实最核心的代码就只有1行：<strong>requestChannel.sendRequest(req)</strong>，也就是将此Request放入Request队列。其他代码只是一些常规化的校验和辅助逻辑。</p><p>这个方法的意思是说，<strong>Processor从底层Socket通道不断读取已接收到的网络请求，然后转换成Request实例，并将其放入到Request队列</strong>。整个逻辑还是很简单的，对吧？</p><h3>processCompletedSends</h3><p>它负责处理Response的回调逻辑。我之前说过，Response需要被发送之后才能执行对应的回调逻辑，这便是该方法代码要实现的功能：</p><pre><code>private def processCompletedSends(): Unit = {
  // 遍历底层SocketChannel已发送的Response
  selector.completedSends.asScala.foreach { send =&gt;
    try {
      // 取出对应inflightResponses中的Response
      val response = inflightResponses.remove(send.destination).getOrElse {
        throw new IllegalStateException(s&quot;Send for ${send.destination} completed, but not in `inflightResponses`&quot;)
      }
      updateRequestMetrics(response) // 更新一些统计指标
      // 执行回调逻辑
      response.onComplete.foreach(onComplete =&gt; onComplete(send))
      handleChannelMuteEvent(send.destination, ChannelMuteEvent.RESPONSE_SENT)
      tryUnmuteChannel(send.destination)
    } catch {
      case e: Throwable =&gt; processChannelException(send.destination,
        s&quot;Exception while processing completed send to ${send.destination}&quot;, e)
    }
  }
}

</code></pre><p>这里通过调用Response对象的onComplete方法，来实现回调函数的执行。</p><h3>processDisconnected</h3><p>顾名思义，它就是处理已断开连接的。该方法的逻辑很简单，我用注释标注了主要的执行步骤：</p><pre><code>private def processDisconnected(): Unit = {
  // 遍历底层SocketChannel的那些已经断开的连接
  selector.disconnected.keySet.asScala.foreach { connectionId =&gt;
    try {
      // 获取断开连接的远端主机名信息
      val remoteHost = ConnectionId.fromString(connectionId).getOrElse {
        throw new IllegalStateException(s&quot;connectionId has unexpected format: $connectionId&quot;)
      }.remoteHost
  // 将该连接从inflightResponses中移除，同时更新一些监控指标
  inflightResponses.remove(connectionId).foreach(updateRequestMetrics)
  // 更新配额数据
  connectionQuotas.dec(listenerName, InetAddress.getByName(remoteHost))
    } catch {
      case e: Throwable =&gt; processException(s&quot;Exception while processing disconnection of $connectionId&quot;, e)
    }
  }
}

</code></pre><p>比较关键的代码是需要从底层Selector中获取那些已经断开的连接，之后把它们从inflightResponses中移除掉，同时也要更新它们的配额数据。</p><h3>closeExcessConnections</h3><p>这是Processor线程的run方法执行的最后一步，即<strong>关闭超限连接</strong>。代码很简单：</p><pre><code>private def closeExcessConnections(): Unit = {
    // 如果配额超限了
    if (connectionQuotas.maxConnectionsExceeded(listenerName)) {
      // 找出优先关闭的那个连接
      val channel = selector.lowestPriorityChannel() 
      if (channel != null)
        close(channel.id) // 关闭该连接
    }
}
</code></pre><p>所谓优先关闭，是指在诸多TCP连接中找出最近未被使用的那个。这里“未被使用”就是说，在最近一段时间内，没有任何Request经由这个连接被发送到Processor线程。</p><h2>总结</h2><p>今天，我带你了解了Kafka网络通信层的全貌，大致介绍了核心组件SocketServer，还花了相当多的时间研究SocketServer下的Acceptor和Processor线程代码。我们来简单总结一下。</p><ul>
<li>网络通信层由SocketServer组件和KafkaRequestHandlerPool组件构成。</li>
<li>SocketServer实现了Reactor模式，用于高性能地并发处理I/O请求。</li>
<li>SocketServer底层使用了Java的Selector实现NIO通信。</li>
</ul><p><img src="https://static001.geekbang.org/resource/image/41/51/41317d400ed096bbca8efadf43186f51.jpg?wh=2349*2028" alt=""></p><p>在下节课，我会重点介绍SocketServer处理不同类型Request所做的设计及其对应的代码。这是社区为了提高Broker处理控制类请求的重大举措，也是为了改善Broker一致性所做的努力，非常值得我们重点关注。</p><h2>课后讨论</h2><p>最后，请思考这样一个问题：为什么Request队列被设计成线程共享的，而Response队列则是每个Processor线程专属的？</p><p>欢迎你在留言区畅所欲言，跟我交流讨论，也欢迎你把今天的内容分享给你的朋友。</p>
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
  <div class="_2_QraFYR_0">你好，我是胡夕。我来公布上节课的“课后讨论”题答案啦～<br><br>上节课，咱们重点了解了Kafka中的请求通道：RequestChannel类。课后我让你去源码中寻找监控Request队列当前使用情况的监控指标。下面我给出我的答案。这个监控指标就是RequestQueueSize。完整的MBean写法是kafka.network:type=RequestChannel,name=RequestQueueSize。你可以在RequestChannel.scala中找到这行源码：newGauge(requestQueueSizeMetricName, () =&gt; requestQueue.size)<br>这里的requestQueueSizeMetricName就对应于RequestQueueSize<br><br><br>okay，你找到了吗？我们可以一起讨论下。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-06 13:56:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/54/9a/76c0af70.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>每天晒白牙</span>
  </div>
  <div class="_2_QraFYR_0">去年自己在工作中接触了 Kafka，然后写了篇网络层的源码分析(应该叫源码注释 + 画图)<br>https:&#47;&#47;mp.weixin.qq.com&#47;s&#47;-VzDU0V8J2guNXwhiBEEyg<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-30 07:11:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/fc/84/7c740988.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_d1cc4d</span>
  </div>
  <div class="_2_QraFYR_0">Request队列线程共享，这样不同线程的workload才不会发生倾斜，不然可能会发生一边的线程空闲，一边的线程队列满</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 很有道理：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-30 00:49:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/4f/37/ad1ca21d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>在路上</span>
  </div>
  <div class="_2_QraFYR_0">课后题思考：<br>（1）Request 队列在被 IO 线程取走处理时不需要关心 Request 被哪个 IO 线程处理了。处理完放入到对应的 Response 队列就好。<br>（2）Response 队列是线程私有的，因为 Processor 管理着某一组 SocketChannel，SocketChannel 发出的某个 Request 处理完成后生成 Response 还需要在该 Processor Send 出去。<br>（3）如果 Response 设计成跟 Request 一样在共享队列中， Processor 需要主动去 take 出来，这时还需要判定对应的 SocketChannel 是否属于这个 Processor。<br>    或者设计成 由一个专门的线程取分发给不同的 Processor。那还不如 IO 线程处理完直接发给不同的 Response 队列划算也没有锁的竞争。<br><br>表达的核心逻辑就是 ：Processor 线程的 selector 注册了该线程所管理的一组 SocketChannel，收发请求只能由该 Processor 处理。所以由 Processor 管理独属的 Response 队列。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-09 11:24:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/e3/13/feaf21e4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>__Unsafe</span>
  </div>
  <div class="_2_QraFYR_0">老师，想请教一个问题，kafka是如何处理nio 的 epoll空轮训bug的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Kafka没做处理。毕竟不属于它应做的事情</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-02 08:58:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/1b/7e/17c2eca1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>hc</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，我这边生产环境kafka0.10版本生产者偶然会出现kafka.common.FailedToSendMessageException: Failed to send messages after 3 tries的问题，问一下您这边有什么好的定位思路吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 看着像是非常古老的异常了。通常都是listeners配置或是clients的&#47;etc&#47;hosts配置的问题。总之是连不上Broker导致的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-20 17:06:57</div>
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
  <div class="_2_QraFYR_0">为什么Clients 与 Broker 的通信网络延迟很大，建议调大缓冲区呢？背后原理是什么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果RTT很大，我们最好是让Socket buffer累积更多的数据之后一次性发送，节省网络I&#47;O带宽。就比如送快递，如果目的地很远，快递员最好先积攒多一点待发送的物件之后再去发送</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-07 20:41:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/a0/cb/aab3b3e7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张三丰</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，请问这句话是不是有歧义？因为对于kafka来说，是可以配置多个listener监听的，一个监听对应一个cceptor线程，那么意味着一个SocketServer实例可能对应多个ccepto线程，而不是一个cceptor线程。<br><br>&quot;2.cceptor 线程类：这是接收和创建外部 TCP 连接的线程。每个 SocketServer 实例只会创建一个 cceptor 线程&quot;</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-27 19:35:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/QicTra5HbNEwGxeG49XibcUibB82I2hpBnp8tviaiaicvFAojuMtRiaLCyl6syzzoS546H2hJibNAQ3h9XM097iapiaamcEQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>邀月对影</span>
  </div>
  <div class="_2_QraFYR_0">老师您好 kafka客户端发送请求1 请求2 请求3 服务端还回响应1 响应2 响应3 响应1是怎么对应请求1的而不会到请求2上 看源码这方面没看明白</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-27 13:17:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/69/26/7968c18f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>xiao儿</span>
  </div>
  <div class="_2_QraFYR_0">kafka连接集群如果只写一个broker地址，当zookeeper挂掉后，访问会存在问题吗？（2.5版本）</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-16 14:10:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/47/1b/64262861.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>胡小禾</span>
  </div>
  <div class="_2_QraFYR_0">老师关于 startupComplete() 的注释不太对. 它并不是为了 等待初始化完成,而是为了通知别的线程初始化完成.</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-04 23:30:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/76/09/62a10668.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>傻傻的帅</span>
  </div>
  <div class="_2_QraFYR_0">课后问题：因为Acceptor和Processor已经完成高吞吐、负载均衡等网络性能瓶颈的点和读写分离，而且I&#47;O是属于重逻辑操作，按照谁有空谁处理的逻辑，因此只需要一个Request 队列作为缓冲区，空闲I&#47;O线程从队列拿出执行即可。Acceptor线程只负责处理连接没有能力处理响应，按照谁处理谁响应的原则，同时连接到Processor线程的时候已经负载均衡了，因为由对应的负责接收的process负责响应是再自然不过了，因为Response队列是每个processor线程都会有一个，这样设计逻辑处理更优雅</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-07 11:11:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/96/ee/9b21c199.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>咸</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，看过这块的网络模型后是不是可以理解为kafka的broker端采用的是reactor中的多线程模型，也就是说一个reactor负责网络上的链接，接收等操作，处理资源池是多个，负责读数据，处理业务(这里也就是放在队列里)，这样理解对吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯嗯，是这样的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-28 22:01:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/e3/8b/27f875ba.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Bryant.C</span>
  </div>
  <div class="_2_QraFYR_0">老版本的response Queue好像确实是保存在requestChannel中，用processor id去索引的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 哈哈，这个要考证下了：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-20 08:39:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/0c/52/f25c3636.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>长脖子树</span>
  </div>
  <div class="_2_QraFYR_0">总结了下 acceptor 和 processor 的处理逻辑<br>Acceptor 部分:<br>1. 建立一个 ServerSocketChannel , 并注册监听 OP_ACCEPT 事件<br>2. 循环调用 select 方法, 获取事件通知,  这时会拿到一个 socketChannel , 然后使用轮询选择一个空闲的 processor ,<br> 将这个 socketChannel 放入 processor 的 newConnections 队列中<br><br>Processor 部分:<br>1. 给 processor 的 newConnections 队列中的 channel 注册 OP_READ<br>2. 从 responseQueue 中拉取 response 对象, 发送 response , 并将 发送完的 response 放入 inflightResponses 队列中<br>3. 获取对应socketChannel 上准备就绪的 IO 操作, 将获取到的字节, 封装成 NetworkReceive , 并放入 completedReceives map 中<br>4. 从 completedReceives 中获取 NetWorkReceive , 将其封装成 Request , 放入所有 processor 共享的请求队列 requestQueue 中<br>5. 取出 inflightResponses  队列中的response , 对其调用回调逻辑<br>6. 处理连接断开后的事情<br>7. 关闭超过配额限制部分的连接</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-23 14:27:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/0c/52/f25c3636.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>长脖子树</span>
  </div>
  <div class="_2_QraFYR_0">文中说:<br> 每当 Processor 线程接收新的连接请求时，都会将对应的 SocketChannel 放入这个队列。后面在创建连接时（也就是调用 configureNewConnections 时），就从该队列中取出 SocketChannel，然后注册新的连接。<br>按照我的理解, tcp 三次握手是在 Acceptor 调用 accept()  完成之前就已经完成了的.  而并不是 processor 的工作. processor 的 configureNewConnections 方法只是将 已经建立的连接取出来再次注册 OP_READ 事件而已<br><br>老师怎么看? 大家有什么想法么?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯嗯，你说的是对的。只是我们对于“创建连接”的定义可能不太一样。通常情况下，除了创建TCP连接，Kafka还需要创建所需的通道等对象，这在Kafka看来都是正常工作前的准备工作：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-23 14:23:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/0c/52/f25c3636.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>长脖子树</span>
  </div>
  <div class="_2_QraFYR_0">Processor 中 run() 中会调用 processCompletedReceives() 这个方法, 将 request  放入 requestQueue 中, 还以为它没加锁, 多个 Processor 会导致问题.  最后发现 requestQueue 是 ArrayBlockingQueue , 里面用了可重入锁.  哎哎哎 并发容器又忘得差不多了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 😃</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-22 18:09:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/26/d9/f7e96590.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>yes</span>
  </div>
  <div class="_2_QraFYR_0">对于老师一直讲述的网络线程和IO线程我一直觉得有点别捏，单单是因为参数的名称所以这样叫嘛？还是官方就是这样命名的？<br>在我的印象中，对于kafka的processor线程所做的事，我认为processor应该称为IO线程，读写socket，而KafkaRequestHandlerPool做的事，对于Broker来说不正是Broker所需要的做的&quot;业务&quot;么？所以称之为业务线程而不是IO线程。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个提法呢是Kafka这边的定义，不是我自己命名的，源码中并无业务线程的提法。不过我们理解意思就行，不用太纠结于称谓</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-29 22:13:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/50/81/e9c5a274.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>huldar</span>
  </div>
  <div class="_2_QraFYR_0">老师，您好。如何正确运行ApiKeys中的main方法呢？直接在idea中点击run按钮会报ClassNotFoundException。查资料后分析应该使用gradle的task运行，但是在clients中没有找到应该是哪个task。谢谢。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是碰到这个错误了吗？ java.lang.ClassNotFoundException: com.fasterxml.jackson.databind.JsonNode<br><br>如果是，我倾向于认为这是一个bug</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-16 00:51:14</div>
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
  <div class="_2_QraFYR_0">接上条老师回复，“ 如果RTT很大，我们最好是让Socket buffer累积更多的数据之后一次性发送，节省网络I&#47;O带宽。”<br>但这里调大的缓冲区是broker端的，broker是接收端，根本措施不应该是调大clients一次发送的的数据量么？<br>还是说这里的broker既是接收端，又是其他broker的clients？文中rtt指broker之间的通讯？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: producer、consumer和broker端都需要调整啊，都有各自的参数</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-09 09:40:33</div>
  </div>
</div>
</div>
</li>
</ul>