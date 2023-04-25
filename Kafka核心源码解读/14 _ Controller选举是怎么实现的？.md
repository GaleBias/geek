<audio title="14 _ Controller选举是怎么实现的？" src="https://static001.geekbang.org/resource/audio/aa/18/aab8712a07a0638527505d3e3574df18.mp3" controls="controls"></audio> 
<p>你好，我是胡夕。</p><p>上节课，我们学习了单线程事件队列模型处理Controller事件的代码。Controller组件通过ControllerEventManager类构造了一个阻塞队列，同时配以专属的事件处理线程，实现了对各类ControllerEvent的处理。</p><p>这种设计思路既保证了多线程访问所需的线程安全，还简化了Controller端的代码结构，极大地提升了代码的可维护性。</p><p>今天，我们学习下Controller选举部分的源码。</p><p>还记得我在<a href="https://time.geekbang.org/column/article/235562">第11节课</a>的案例中提到的“恢复大法”——删除ZooKeeper的/controller节点吗？当时，我们靠着这个“秘籍”涉险过关，既恢复了错误的集群状态，又避免了重启整个生产环境。</p><p>但你有没有想过，为什么删除/controller节点能够令集群元数据重新保持同步呢？如果不了解这背后的原理，我们是不敢贸然在生产环境做这种操作的。今天，我们要学习的就是这背后的一整套实现逻辑，重点关注下Controller是怎么被选举出来的。</p><p>我始终认为，只有掌握了这些知识，才算真正入门Kafka服务器端的代码了。作为Broker端最重要的组件之一，Controller在Kafka中的地位无可替代。整个Kafka集群就只有一个Controller，从某种意义上来说，它是目前Kafka这个分布式系统中唯一的“单点”。</p><!-- [[[read_end]]] --><p>因此，了解这个“单点”的选举触发场景，以及如何被选举出来的，对于我们后面深入理解Controller在集群中的作用非常有帮助。毕竟，Controller对外提供的一些服务也是采用了类似的实现原理。</p><h2>概览</h2><h3>ZooKeeper /controller节点</h3><p>再次强调下，<strong>在一个Kafka集群中，某段时间内只能有一台Broker被选举为Controller。随着时间的推移，可能会有不同的Broker陆续担任过Controller的角色，但是在某一时刻，Controller只能由一个Broker担任</strong>。</p><p>那选择哪个Broker充当Controller呢？当前，Controller的选举过程依赖ZooKeeper完成。ZooKeeper除了扮演集群元数据的“真理之源”角色，还定义了/controller临时节点（Ephemeral Node），以协助完成Controller的选举。</p><p>下面这段代码展示的是一个双Broker的Kafka集群上的ZooKeeper中/controller节点：</p><pre><code>{&quot;version&quot;:1,&quot;brokerid&quot;:0,&quot;timestamp&quot;:&quot;1585098432431&quot;}
cZxid = 0x1a
ctime = Wed Mar 25 09:07:12 CST 2020
mZxid = 0x1a
mtime = Wed Mar 25 09:07:12 CST 2020
pZxid = 0x1a
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x100002d3a1f0000
dataLength = 54
numChildren = 0
</code></pre><p>有两个地方的内容，你要重点关注一下。</p><ul>
<li>Controller Broker Id是0，表示序号为0的Broker是集群Controller。</li>
<li>ephemeralOwner字段不是0x0，说明这是一个临时节点。</li>
</ul><p>既然是临时节点，那么，一旦Broker与ZooKeeper的会话终止，该节点就会消失。Controller选举就依靠了这个特性。每个Broker都会监听/controller节点随时准备应聘Controller角色。下图展示了Broker与/controller节点的交互关系：</p><p><img src="https://static001.geekbang.org/resource/image/2e/83/2e75cdbfb68c86169ec83f58e59e1283.jpg?wh=1948*1252" alt=""></p><p>如图所示，集群上所有的Broker都在实时监听ZooKeeper上的这个节点。这里的“监听”有两个含义。</p><ul>
<li><strong>监听这个节点是否存在</strong>。倘若发现这个节点不存在，Broker会立即“抢注”该节点，即创建/controller节点。创建成功的那个Broker，即当选为新一届的Controller。</li>
<li><strong>监听这个节点数据是否发生了变更</strong>。同样，一旦发现该节点的内容发生了变化，Broker也会立即启动新一轮的Controller选举。</li>
</ul><p>掌握了这些基础之后，下面我们来阅读具体的源码文件：KafkaController.scala。这是一个2200行的大文件。我先向你介绍一下这个文件的大致结构，以免你陷入到一些繁枝末节中。</p><h3>源码结构</h3><p>KafkaController文件的代码结构如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/7e/88/7e5ddb69df585b5bbbcc91336ab8f588.jpg?wh=2250*1532" alt=""></p><p>整体而言，该文件大致由五部分组成。</p><ul>
<li><strong>选举触发器</strong>（ElectionTrigger）：这里的选举不是指Controller选举，而是指主题分区副本的选举，即为哪些分区选择Leader副本。后面在学习副本管理器和分区管理器时，我们会讲到它。</li>
<li><strong>KafkaController Object</strong>：KafkaController伴生对象，仅仅定义了一些常量和回调函数类型。</li>
<li><strong>ControllerEvent</strong>：定义Controller事件类型。上节课我们详细学习过Controller事件以及基于事件的单线程事件队列模型。这部分的代码看着很多，但实际上都是千篇一律的。你看懂了一个事件的定义，其他的也就不在话下了。</li>
<li><strong>各种ZooKeeper监听器</strong>：定义ZooKeeper监听器，去监听ZooKeeper中各个节点的变更。今天，我们重点关注监听/controller节点的那个监听器。</li>
<li><strong>KafkaController Class</strong>：定义KafkaController类以及实际的处理逻辑。这是我们今天的重点学习对象。</li>
</ul><p>接下来，我会给你重点介绍KafkaController类、ZooKeeper监听器和Controller选举这三大部分。在众多的ZooKeeper监听器中，我会详细介绍监听Controller变更的监听器，它也是我们了解Controller选举流程的核心环节。</p><h2>KafkaController类</h2><p>这个类大约有1900行代码，里面定义了非常多的变量和方法。这些方法大多是处理不同Controller事件的。后面讲到选举流程的时候，我会挑一些有代表性的来介绍。我希望你能举一反三，借此吃透其他方法的代码。毕竟，它们做的事情大同小异，至少代码风格非常相似。</p><p>在学习重要的方法之前，我们必须要先掌握KafkaController类的定义。接下来，我们从4个维度来进行学习，分别是原生字段、辅助字段、各类ZooKeeper监听器字段和统计字段。</p><p>弄明白了这些字段的含义之后，再去看操作这些字段的方法，会更加有的放矢，理解起来也会更加容易。</p><h3>原生字段</h3><p>首先来看原生字段。所谓的原生字段，是指在创建一个KafkaController实例时，需要指定的字段。</p><p>先来看下KafkaController类的定义代码：</p><pre><code>// 字段含义：
// config：Kafka配置信息，通过它，你能拿到Broker端所有参数的值
// zkClient：ZooKeeper客户端，Controller与ZooKeeper的所有交互均通过该属性完成
// time：提供时间服务(如获取当前时间)的工具类
// metrics：实现指标监控服务(如创建监控指标)的工具类
// initialBrokerInfo：Broker节点信息，包括主机名、端口号，所用监听器等
// initialBrokerEpoch：Broker Epoch值，用于隔离老Controller发送的请求
// tokenManager：实现Delegation token管理的工具类。Delegation token是一种轻量级的认证机制
// threadNamePrefix：Controller端事件处理线程名字前缀
class KafkaController(val config: KafkaConfig,
                      zkClient: KafkaZkClient,
                      time: Time,
                      metrics: Metrics,
                      initialBrokerInfo: BrokerInfo,
                      initialBrokerEpoch: Long,
                      tokenManager: DelegationTokenManager,
                      threadNamePrefix: Option[String] = None)
  extends ControllerEventProcessor with Logging with KafkaMetricsGroup {
  ......
}
</code></pre><p>就像我上节课说过的，KafkaController实现了ControllerEventProcessor接口，因而也就实现了处理Controller事件的process方法。这里面比较重要的字段有3个。</p><ul>
<li><strong>config</strong>：KafkaConfig类实例，里面封装了Broker端所有参数的值。</li>
<li><strong>zkClient</strong>：ZooKeeper客户端类，定义了与ZooKeeper交互的所有方法。</li>
<li><strong>initialBrokerEpoch</strong>：Controller所在Broker的Epoch值。Kafka使用它来确保Broker不会处理老Controller发来的请求。</li>
</ul><p>其他字段要么是像time、metrics一样，是工具类字段，要么是像initialBrokerInfo、tokenManager字段一样，使用场景很有限，我就不展开讲了。</p><h3>辅助字段</h3><p>除了原生字段之外，KafkaController还定义了很多辅助字段，帮助实现Controller的各类功能。</p><p>我们来看一些重要的辅助字段：</p><pre><code>......
// 集群元数据类，保存集群所有元数据
val controllerContext = new ControllerContext
// Controller端通道管理器类，负责Controller向Broker发送请求
var controllerChannelManager = new ControllerChannelManager(controllerContext, config, time, metrics,
  stateChangeLogger, threadNamePrefix)
// 线程调度器，当前唯一负责定期执行Leader重选举
private[controller] val kafkaScheduler = new KafkaScheduler(1)
// Controller事件管理器，负责管理事件处理线程
private[controller] val eventManager = new ControllerEventManager(config.brokerId, this, time,
  controllerContext.stats.rateAndTimeMetrics)
......
// 副本状态机，负责副本状态转换
val replicaStateMachine: ReplicaStateMachine = new ZkReplicaStateMachine(config, stateChangeLogger, controllerContext, zkClient,
  new ControllerBrokerRequestBatch(config, controllerChannelManager, eventManager, controllerContext, stateChangeLogger))
// 分区状态机，负责分区状态转换
val partitionStateMachine: PartitionStateMachine = new ZkPartitionStateMachine(config, stateChangeLogger, controllerContext, zkClient,
  new ControllerBrokerRequestBatch(config, controllerChannelManager, eventManager, controllerContext, stateChangeLogger))
// 主题删除管理器，负责删除主题及日志
val topicDeletionManager = new TopicDeletionManager(config, controllerContext, replicaStateMachine,
  partitionStateMachine, new ControllerDeletionClient(this, zkClient))
......
</code></pre><p>其中，有7个字段是重中之重。</p><ul>
<li><strong>controllerContext</strong>：集群元数据类，保存集群所有元数据。</li>
<li><strong>controllerChannelManager</strong>：Controller端通道管理器类，负责Controller向Broker发送请求。</li>
<li><strong>kafkaScheduler</strong>：线程调度器，当前唯一负责定期执行分区重平衡Leader选举。</li>
<li><strong>eventManager</strong>：Controller事件管理器，负责管理事件处理线程。</li>
<li><strong>replicaStateMachine</strong>：副本状态机，负责副本状态转换。</li>
<li><strong>partitionStateMachine</strong>：分区状态机，负责分区状态转换。</li>
<li><strong>topicDeletionManager</strong>：主题删除管理器，负责删除主题及日志。</li>
</ul><h3>各类ZooKeeper监听器</h3><p>我们今天开头学到的ControllerChangeHandler仅仅是其中的一种。实际上，该类定义了很多监听器，如下所示：</p><pre><code>// Controller节点ZooKeeper监听器
private val controllerChangeHandler = new ControllerChangeHandler(eventManager)
// Broker数量ZooKeeper监听器
private val brokerChangeHandler = new BrokerChangeHandler(eventManager)
// Broker信息变更ZooKeeper监听器集合
private val brokerModificationsHandlers: mutable.Map[Int, BrokerModificationsHandler] = mutable.Map.empty
// 主题数量ZooKeeper监听器
private val topicChangeHandler = new TopicChangeHandler(eventManager)
// 主题删除ZooKeeper监听器
private val topicDeletionHandler = new TopicDeletionHandler(eventManager)
// 主题分区变更ZooKeeper监听器
private val partitionModificationsHandlers: mutable.Map[String, PartitionModificationsHandler] = mutable.Map.empty
// 主题分区重分配ZooKeeper监听器
private val partitionReassignmentHandler = new PartitionReassignmentHandler(eventManager)
// Preferred Leader选举ZooKeeper监听器
private val preferredReplicaElectionHandler = new PreferredReplicaElectionHandler(eventManager)
// ISR副本集合变更ZooKeeper监听器
private val isrChangeNotificationHandler = new IsrChangeNotificationHandler(eventManager)
// 日志路径变更ZooKeeper监听器
private val logDirEventNotificationHandler = new LogDirEventNotificationHandler(eventManager)
</code></pre><p>我分别解释一下这些ZooKeeper监听器的作用：</p><ul>
<li><strong>controllerChangeHandler</strong>：前面说过，它是监听/controller节点变更的。这种变更包括节点创建、删除以及数据变更。</li>
<li><strong>brokerChangeHandler</strong>：监听Broker的数量变化。</li>
<li><strong>brokerModificationsHandlers</strong>：监听Broker的数据变更，比如Broker的配置信息发生的变化。</li>
<li><strong>topicChangeHandler</strong>：监控主题数量变更。</li>
<li><strong>topicDeletionHandler</strong>：监听主题删除节点/admin/delete_topics的子节点数量变更。</li>
<li><strong>partitionModificationsHandlers</strong>：监控主题分区数据变更的监听器，比如，新增加了副本、分区更换了Leader副本。</li>
<li><strong>partitionReassignmentHandler</strong>：监听分区副本重分配任务。一旦发现新提交的任务，就为目标分区执行副本重分配。</li>
<li><strong>preferredReplicaElectionHandler</strong>：监听Preferred Leader选举任务。一旦发现新提交的任务，就为目标主题执行Preferred Leader选举。</li>
<li><strong>isrChangeNotificationHandler</strong>：监听ISR副本集合变更。一旦被触发，就需要获取ISR发生变更的分区列表，然后更新Controller端对应的Leader和ISR缓存元数据。</li>
<li><strong>logDirEventNotificationHandler</strong>：监听日志路径变更。一旦被触发，需要获取受影响的Broker列表，然后处理这些Broker上失效的日志路径。</li>
</ul><p>我画了一张脑图，希望可以帮助你更高效地记住这些ZooKeeper监听器：</p><p><img src="https://static001.geekbang.org/resource/image/8f/31/8feed623165ab6e50b31614e67498c31.jpg?wh=2250*2963" alt=""></p><h3>统计字段</h3><p>最后，我们来看统计字段。</p><p>这些统计字段大多用于计算统计指标。有的监控指标甚至是非常重要的Controller监控项，比如ActiveControllerCount指标。下面，我们来了解下KafkaController都定义了哪些统计字段。这些指标的含义一目了然，非常清晰，我用注释的方式给出每个字段的含义：</p><pre><code>// 当前Controller所在Broker Id
@volatile private var activeControllerId = -1
// 离线分区总数
@volatile private var offlinePartitionCount = 0
// 满足Preferred Leader选举条件的总分区数
@volatile private var preferredReplicaImbalanceCount = 0
// 总主题数
@volatile private var globalTopicCount = 0
// 总主题分区数
@volatile private var globalPartitionCount = 0
// 待删除主题数
@volatile private var topicsToDeleteCount = 0
//待删除副本数
@volatile private var replicasToDeleteCount = 0
// 暂时无法删除的主题数
@volatile private var ineligibleTopicsToDeleteCount = 0
// 暂时无法删除的副本数
@volatile private var ineligibleReplicasToDeleteCount = 0
</code></pre><p>好了，KafkaController类的定义我们就全部介绍完了。再次强调下，因为KafkaController类的代码很多，我强烈建议你熟练掌握这些字段的含义，因为后面的所有方法都是围绕着这些字段进行操作的。</p><p>接下来，我以Controller的选举流程为例，引出KafkaController的一些方法的实现原理。不过，在此之前，我们要学习监听Controller变更的ZooKeeper监听器：ControllerChangeHandler的源码。</p><h2>ControllerChangeHandler监听器</h2><p>就像我前面说到的，KafkaController定义了十几种ZooKeeper监听器。和Controller相关的监听器是ControllerChangeHandler，用于监听Controller的变更，定义代码如下：</p><pre><code>class ControllerChangeHandler(eventManager: ControllerEventManager) extends ZNodeChangeHandler {
  // ZooKeeper中Controller节点路径，即/controller
  override val path: String = ControllerZNode.path
  // 监听/controller节点创建事件
  override def handleCreation(): Unit = eventManager.put(ControllerChange)
  // 监听/controller节点被删除事件
  override def handleDeletion(): Unit = eventManager.put(Reelect)
  // 监听/controller节点数据变更事件
  override def handleDataChange(): Unit = eventManager.put(ControllerChange)
}
</code></pre><p>该监听器接收ControllerEventManager实例，实现了ZNodeChangeHandler接口的三个方法：<strong>handleCreation</strong>、<strong>handleDeletion</strong>和<strong>handleDataChange</strong>。该监听器下的path变量，实际上就是/controller字符串，表示它监听ZooKeeper的这个节点。</p><p>3个handle方法都用于监听/controller节点的变更，但实现细节上稍有不同。</p><p>handleCreation和handleDataChange的处理方式是向事件队列写入ControllerChange事件；handleDeletion的处理方式是向事件队列写入Reelect事件。</p><p>Deletion表明ZooKeeper中/controller节点不存在了，即Kafka集群中的Controller暂时空缺了。因为它和Creation和DataChange是不同的状态，需要区别对待，因此，Reelect事件做的事情要比ControllerChange的多：处理ControllerChange事件，只需要当前Broker执行“卸任Controller”的逻辑即可，而Reelect事件是重选举，除了Broker执行卸任逻辑之外，还要求Broker参与到重选举中来。</p><p>由于KafkaController的process方法代码非常长，因此，我节选了刚刚提到的那两个事件的处理代码：</p><pre><code>// process方法(部分)
override def process(event: ControllerEvent): Unit = {
    try {
      event match {
       ......
       // ControllerChange事件
       case ControllerChange =&gt;
          processControllerChange()
       // Reelect事件
       case Reelect =&gt;
          processReelect()
        ......
      }
    }
    ......
}
// 如果是ControllerChange事件，仅执行卸任逻辑即可
private def processControllerChange(): Unit = {
    maybeResign()
  }
// 如果是Reelect事件，还需要执行elect方法参与新一轮的选举
private def processReelect(): Unit = {
    maybeResign()
    elect()
}
</code></pre><p>可以看到，虽然代码非常长，但整体结构却工整清晰，全部都是基于模式匹配的事件处理。process方法会根据给定的Controller事件类型，调用对应的process***方法处理该事件。这里只列举了ZooKeeper端/controller节点监听器监听的两类事件，以及对应的处理方法。</p><p>对于ControllerChange事件而言，处理方式是调用maybeResign去执行Controller的卸任逻辑。如果是Reelect事件，除了执行卸任逻辑之外，还要额外执行elect方法进行新一轮的Controller选举。</p><h2>Controller选举流程</h2><p>说完了ControllerChangeHandler源码，我们来看下Controller的选举。所谓的Controller选举，是指Kafka选择集群中一台Broker行使Controller职责。整个选举过程分为两个步骤：触发选举和开始选举。</p><h3>触发选举</h3><p>我先用一张图展示下可能触发Controller选举的三个场景。</p><p><img src="https://static001.geekbang.org/resource/image/a8/98/a8cbc562518f93f9befc6bd7a87d5b98.jpg?wh=2276*2297" alt=""></p><p>这三个场景是：</p><ol>
<li>集群从零启动时；</li>
<li>Broker侦测/controller节点消失时；</li>
<li>Broker侦测到/controller节点数据发生变更时。</li>
</ol><p>这三个场景殊途同归，最后都要执行选举Controller的动作。我来一一解释下这三个场景，然后再介绍选举Controller的具体操作。</p><h4>场景一：集群从零启动</h4><p>集群首次启动时，Controller尚未被选举出来。于是，Broker启动后，首先将Startup这个ControllerEvent写入到事件队列中，然后启动对应的事件处理线程和ControllerChangeHandler ZooKeeper监听器，最后依赖事件处理线程进行Controller的选举。</p><p>在源码中，KafkaController类的startup方法就是做这些事情的。当Broker启动时，它会调用这个方法启动ControllerEventThread线程。值得注意的是，<strong>每个Broker都需要做这些事情，不是说只有Controller所在的Broker才需要执行这些逻辑</strong>。</p><p>startup方法的主体代码如下：</p><pre><code>def startup() = {
  // 第1步：注册ZooKeeper状态变更监听器，它是用于监听Zookeeper会话过期的
  zkClient.registerStateChangeHandler(new StateChangeHandler {
    override val name: String = StateChangeHandlers.ControllerHandler
    override def afterInitializingSession(): Unit = {
      eventManager.put(RegisterBrokerAndReelect)
    }
    override def beforeInitializingSession(): Unit = {
      val queuedEvent = eventManager.clearAndPut(Expire)
      queuedEvent.awaitProcessing()
    }
  })
  // 第2步：写入Startup事件到事件队列
  eventManager.put(Startup)
  // 第3步：启动ControllerEventThread线程，开始处理事件队列中的ControllerEvent
  eventManager.start()
}

</code></pre><p>首先，startup方法会注册ZooKeeper状态变更监听器，用于监听Broker与ZooKeeper之间的会话是否过期。接着，写入Startup事件到事件队列，然后启动ControllerEventThread线程，开始处理事件队列中的Startup事件。</p><p>接下来，我们来学习下KafkaController的process方法处理Startup事件的方法：</p><pre><code>// KafkaController的process方法，
override def process(event: ControllerEvent): Unit = {
    try {
      event match {
       ......
       case Startup =&gt;
          processStartup() // 处理Startup事件
      }
    }
    ......
}
private def processStartup(): Unit = {
   // 注册ControllerChangeHandler ZooKeeper监听器
   zkClient.registerZNodeChangeHandlerAndCheckExistence(
    controllerChangeHandler)
   // 执行Controller选举
   elect()
}
</code></pre><p>从这段代码可知，process方法调用processStartup方法去处理Startup事件。而processStartup方法又会调用zkClient的registerZNodeChangeHandlerAndCheckExistence方法注册ControllerChangeHandler监听器。</p><p>值得注意的是，虽然前面的三个场景是并列的关系，但实际上，后面的两个场景必须要等场景一的这一步成功执行之后，才能被触发。</p><p>这三种场景都要选举Controller，因此，我们最后统一学习elect方法的代码实现。</p><p>总体来说，集群启动时，Broker通过向事件队列“塞入”Startup事件的方式，来触发Controller的竞选。</p><h4>场景二：/controller节点消失</h4><p>Broker检测到/controller节点消失时，就意味着，此时整个集群中没有Controller。因此，所有检测到/controller节点消失的Broker，都会立即调用elect方法执行竞选逻辑。</p><p>你可能会问：“Broker是怎么侦测到ZooKeeper上的这一变化的呢？”实际上，这是ZooKeeper监听器提供的功能，换句话说，这是Apache ZooKeeper自己实现的功能，所以我们才说，Kafka依赖ZooKeeper完成Controller的选举。</p><p>讲到这里，我说点题外话，社区最近正在酝酿彻底移除ZooKeeper依赖。具体到Controller端的变化，就是在Kafka内部实现一个类似于Raft的共识算法来选举Controller。我会在后面的特别放送里详细讲一下社区移除ZooKeeper的全盘计划。</p><h4>场景三：/controller节点数据变更</h4><p>Broker检测到/controller节点数据发生变化，通常表明，Controller“易主”了，这就分为两种情况：</p><ul>
<li>如果Broker之前是Controller，那么该Broker需要首先执行卸任操作，然后再尝试竞选；</li>
<li>如果Broker之前不是Controller，那么，该Broker直接去竞选新Controller。</li>
</ul><p>具体到代码层面，maybeResign方法形象地说明了这两种情况。你要注意方法中的maybe字样，这表明，Broker可能需要执行卸任操作，也可能不需要。Kafka源码非常喜欢用maybe***来命名方法名，以表示那些在特定条件下才需要执行的逻辑。以下是maybeResign的实现：</p><pre><code>private def maybeResign(): Unit = {
  // 非常关键的一步！这是判断是否需要执行卸任逻辑的重要依据！
  // 判断该Broker之前是否是Controller
  val wasActiveBeforeChange = isActive
  // 注册ControllerChangeHandler监听器  
  zkClient.registerZNodeChangeHandlerAndCheckExistence(
    controllerChangeHandler)
  // 获取当前集群Controller所在的Broker Id，如果没有Controller则返回-1
  activeControllerId = zkClient.getControllerId.getOrElse(-1)
  // 如果该Broker之前是Controller但现在不是了
  if (wasActiveBeforeChange &amp;&amp; !isActive) {
    onControllerResignation() // 执行卸任逻辑
  }
}
</code></pre><p>代码的第一行非常关键，它是决定是否需要执行卸任的重要依据。毕竟，如果Broker之前不是Controller，那何来“卸任”一说呢？之后代码要注册ControllerChangeHandler监听器，获取当前集群Controller所在的Broker ID，如果没有Controller，则返回-1。有了这些数据之后，maybeResign方法需要判断该Broker是否之前是Controller但现在不是了。如果是这种情况的话，则调用onControllerResignation方法执行Controller卸任逻辑。</p><p>说到“卸任”，你可能会问：“卸任逻辑是由哪个方法执行的呢？”实际上，这是由onControllerResignation方法执行的，它主要是用于清空各种数据结构的值、取消ZooKeeper监听器、关闭各种状态机以及管理器，等等。我用注释的方式给出它的逻辑实现：</p><pre><code>private def onControllerResignation(): Unit = {
  debug(&quot;Resigning&quot;)
  // 取消ZooKeeper监听器的注册
  zkClient.unregisterZNodeChildChangeHandler(
    isrChangeNotificationHandler.path)
  zkClient.unregisterZNodeChangeHandler(
    partitionReassignmentHandler.path)
  zkClient.unregisterZNodeChangeHandler(
    preferredReplicaElectionHandler.path)
  zkClient.unregisterZNodeChildChangeHandler(
    logDirEventNotificationHandler.path)
  unregisterBrokerModificationsHandler(
    brokerModificationsHandlers.keySet)
  // 关闭Kafka线程调度器，其实就是取消定期的Leader重选举
  kafkaScheduler.shutdown()
  // 将统计字段全部清0
  offlinePartitionCount = 0
  preferredReplicaImbalanceCount = 0
  globalTopicCount = 0
  globalPartitionCount = 0
  topicsToDeleteCount = 0
  replicasToDeleteCount = 0
  ineligibleTopicsToDeleteCount = 0
  ineligibleReplicasToDeleteCount = 0
  // 关闭Token过期检查调度器
  if (tokenCleanScheduler.isStarted)
    tokenCleanScheduler.shutdown()
  // 取消分区重分配监听器的注册
  unregisterPartitionReassignmentIsrChangeHandlers()
  // 关闭分区状态机
  partitionStateMachine.shutdown()
  // 取消主题变更监听器的注册
  zkClient.unregisterZNodeChildChangeHandler(topicChangeHandler.path)
  // 取消分区变更监听器的注册
  unregisterPartitionModificationsHandlers(
    partitionModificationsHandlers.keys.toSeq)
  // 取消主题删除监听器的注册
  zkClient.unregisterZNodeChildChangeHandler(
    topicDeletionHandler.path)
  // 关闭副本状态机
  replicaStateMachine.shutdown()
  // 取消Broker变更监听器的注册
  zkClient.unregisterZNodeChildChangeHandler(brokerChangeHandler.path)
  // 关闭Controller通道管理器
  controllerChannelManager.shutdown()
  // 清空集群元数据
  controllerContext.resetContext()
  info(&quot;Resigned&quot;)
}

</code></pre><h3>选举Controller</h3><p>讲完了触发场景，接下来，我们就要学习Controller选举的源码了。前面说过了，这三种选举场景最后都会调用elect方法来执行选举逻辑。我们来看下它的实现：</p><pre><code>private def elect(): Unit = {
    // 第1步：获取当前Controller所在Broker的序号，如果Controller不存在，显式标记为-1
    activeControllerId = zkClient.getControllerId.getOrElse(-1)

    // 第2步：如果当前Controller已经选出来了，直接返回即可
    if (activeControllerId != -1) {
      debug(s&quot;Broker $activeControllerId has been elected as the controller, so stopping the election process.&quot;)
      return
    }

    try {
      // 第3步：注册Controller相关信息
      // 主要是创建/controller节点
      val (epoch, epochZkVersion) = zkClient.registerControllerAndIncrementControllerEpoch(config.brokerId)
      controllerContext.epoch = epoch
      controllerContext.epochZkVersion = epochZkVersion
      activeControllerId = config.brokerId

      info(s&quot;${config.brokerId} successfully elected as the controller. Epoch incremented to ${controllerContext.epoch} &quot; +
        s&quot;and epoch zk version is now ${controllerContext.epochZkVersion}&quot;)

      // 第4步：执行当选Controller的后续逻辑
      onControllerFailover()
    } catch {
      case e: ControllerMovedException =&gt;
        maybeResign()

        if (activeControllerId != -1)
          debug(s&quot;Broker $activeControllerId was elected as controller instead of broker ${config.brokerId}&quot;, e)
        else
          warn(&quot;A controller has been elected but just resigned, this will result in another round of election&quot;, e)

      case t: Throwable =&gt;
        error(s&quot;Error while electing or becoming controller on broker ${config.brokerId}. &quot; +
          s&quot;Trigger controller movement immediately&quot;, t)
        triggerControllerMove()
    }
  }
</code></pre><p>为了帮助你更好地理解这个方法，我再画一张图来进行说明：</p><p><img src="https://static001.geekbang.org/resource/image/23/1b/2331395774956a61f37836c46d65d01b.jpg?wh=2104*2136" alt=""></p><p>该方法首先检查Controller是否已经选出来了。要知道，集群中的所有Broker都要执行这些逻辑，因此，非常有可能出现某些Broker在执行elect方法时，Controller已经被选出来的情况。如果Controller已经选出来了，那么，自然也就不用再做什么了。相反地，如果Controller尚未被选举出来，那么，代码会尝试创建/controller节点去抢注Controller。</p><p>一旦抢注成功，就调用onControllerFailover方法，执行选举成功后的动作。这些动作包括注册各类ZooKeeper监听器、删除日志路径变更和ISR副本变更通知事件、启动Controller通道管理器，以及启动副本状态机和分区状态机。</p><p>如果抢注失败了，代码会抛出ControllerMovedException异常。这通常表明Controller已经被其他Broker抢先占据了，那么，此时代码调用maybeResign方法去执行卸任逻辑。</p><h2>总结</h2><p>今天，我们梳理了Controller选举的全过程，包括Controller如何借助ZooKeeper监听器实现监听Controller节点，以及Controller的选举触发场景和完整流程。我们来回顾一下这节课的重点。</p><ul>
<li>Controller依赖ZooKeeper实现Controller选举，主要是借助于/controller临时节点和ZooKeeper的监听器机制。</li>
<li>Controller触发场景有3种：集群启动时；/controller节点被删除时；/controller节点数据变更时。</li>
<li>源码最终调用elect方法实现Controller选举。</li>
</ul><p><img src="https://static001.geekbang.org/resource/image/e2/74/e28c134e4fd11ff8ed87933aee88d374.jpg?wh=2250*3387" alt=""></p><p>下节课，我将带你学习Controller的其他重要功能，包括它如何管理Broker和副本等。你千万不要错过。</p><h2>课后讨论</h2><p>在这节课刚开始的时候，我提到，删除/controller会触发Controller选举，之后会同步集群元数据信息。那么，你知道源码是在哪里更新的元数据请求吗？</p><p>欢迎你在留言区畅所欲言，跟我交流讨论，也欢迎你把今天的内容分享给你的朋友。</p>
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
  <div class="_2_QraFYR_0">你好，我是胡夕。我来公布上节课的“课后讨论”题答案啦～<br><br>上节课，咱们重点了解了Controller的单线程+阻塞队列消息处理模式。课后我请你思考这样一个问题：ControllerEventManager的put方法代码是否需要加putLock保护。这是一个开放式的问题，就我个人的观点，我认为是不需要的。因为queue本身是线程安全的，queuedEvent又是线程私有的栈上引用，因此这里的线程安全性可完全由queue字段类型LinkedBlockingQueue来保证。<br><br>okay，你同意这个说法吗？或者说你有其他的看法吗？我们可以一起讨论下。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-29 10:22:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/03/1c/c9fe6738.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Kvicii.Y</span>
  </div>
  <div class="_2_QraFYR_0">在最后的elect方法中，如果抢注失败抛出 ControllerMovedException 异常后为什么要执行卸任逻辑逻辑呢？此时这个Broker即使之前是Controller但应该也是执行过卸任逻辑了，为什么还要重复的执行一次呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: maybeResign只是maybe：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-02 00:46:41</div>
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
  <div class="_2_QraFYR_0">这句话前后好矛盾啊，节点数据变化这里说只需要卸任，不选举，后边又说卸任并且选举，这是写错了么？<br><br><br>&quot;Deletion 表明 ZooKeeper 中 &#47;controller 节点不存在了，即 Kafka 集群中的 Controller 暂时空缺了。因为它和 Creation 和 DataChange 是不同的状态，需要区别对待，因此，Reelect 事件做的事情要比 ControllerChange 的多：处理 ControllerChange 事件，只需要当前 Broker 执行“卸任 Controller”的逻辑即可，而 Reelect 事件是重选举，除了 Broker 执行卸任逻辑之外，还要求 Broker 参与到重选举中来。&quot;<br><br><br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-13 17:27:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/8b/fe/f2b2ae63.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>t-mac</span>
  </div>
  <div class="_2_QraFYR_0">老师， 以前的版本有注册&quot;registerSessionExpirationListener()&quot;  新版本不需要这个监听zk session的listener了吗？ 这个listener是不是非必须的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不需要了。新版本Controller没有这个监听器了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-06 11:42:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/f3/f3/3fbb4c38.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>旭杰</span>
  </div>
  <div class="_2_QraFYR_0">触发场景三：controller节点数据发生变更。这种情况下Zookeeper会通知ControllerChange事件，Broker端执行processControllerChange()-&gt;maybeResign()，只是做一些清理工作，该事件本身应该不会触发controller选举。该事件应该属于Controller易主过程中的一部分，最终是由Zookeeper通知Reelect()事件来触发选举流程。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: ������</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-03 15:23:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/f3/f3/3fbb4c38.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>旭杰</span>
  </div>
  <div class="_2_QraFYR_0">&#47;controller 节点数据变更 数据变更指的是?isr集合变更&#47;broker列表变更这种吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 就是controller发生变更，因为&#47;controller节点的数据就是controller所在的broker id</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-21 11:35:50</div>
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
  <div class="_2_QraFYR_0">老师，controller 会出现脑裂吗？或者是在哪个版本这个bug 修复了呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，实际上我更愿意这么说：ZooKeeper出现脑裂导致Kafka集群出现多个controller。Kafka提供了一些参数可以帮忙应对这种场景：unclean.leader.election.enable、replication.factor和min.insync.replicas</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-12 08:55:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/83/c9/5d03981a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>thomas</span>
  </div>
  <div class="_2_QraFYR_0">老师，新建一个topic时，是如何决定选哪一个是领导者副本？假如有两个broker，replica. factor=2</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 一般是按照副本列表中的第一个副本是leader副本。而副本列表一般是按照brokerid顺序排列</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-28 20:13:51</div>
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
  <div class="_2_QraFYR_0">调用onControllerFailover时候<br>sendUpdateMetadataRequest(controllerContext.liveOrShuttingDownBrokerIds.toSeq, Set.empty)<br>向集群里的brokers发送更新元数据请求 </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对，让它们去更新元数据</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-21 12:09:52</div>
  </div>
</div>
</div>
</li>
</ul>