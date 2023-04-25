<audio title="11 _ Controller元数据：Controller都保存有哪些东西？有几种状态？" src="https://static001.geekbang.org/resource/audio/5f/8b/5f0417c8727827798cb3225eba50f98b.mp3" controls="controls"></audio> 
<p>你好，我是胡夕。从今天开始，我们正式进入到第三大模块的学习：控制器（Controller）模块 。</p><p>提起Kafka中的Controller组件，我相信你一定不陌生。从某种意义上说，它是Kafka最核心的组件。一方面，它要为集群中的所有主题分区选举领导者副本；另一方面，它还承载着集群的全部元数据信息，并负责将这些元数据信息同步到其他Broker上。既然我们是Kafka源码解读课，那就绝对不能错过这么重量级的组件。</p><p>我画了一张图片，希望借助它帮你建立起对这个模块的整体认知。今天，我们先学习下Controller元数据。</p><p><img src="https://static001.geekbang.org/resource/image/13/5f/13c0d8b3f52c295c70c71a154dae185f.jpg?wh=2447*1412" alt=""></p><h2>案例分享</h2><p>在正式学习源码之前，我想向你分享一个真实的案例。</p><p>在我们公司的Kafka集群环境上，曾经出现了一个比较“诡异”的问题：某些核心业务的主题分区一直处于“不可用”状态。</p><p>通过使用“kafka-topics”命令查询，我们发现，这些分区的Leader显示是-1。之前，这些Leader所在的Broker机器因为负载高宕机了，当Broker重启回来后，Controller竟然无法成功地为这些分区选举Leader，因此，它们一直处于“不可用”状态。</p><p>由于是生产环境，我们的当务之急是马上恢复受损分区，然后才能调研问题的原因。有人提出，重启这些分区旧Leader所在的所有Broker机器——这很容易想到，毕竟“重启大法”一直很好用。但是，这一次竟然没有任何作用。</p><!-- [[[read_end]]] --><p>之后，有人建议升级重启大法，即重启集群的所有Broker——这在当时是不能接受的。且不说有很多业务依然在运行着，单是重启Kafka集群本身，就是一件非常缺乏计划性的事情。毕竟，生产环境怎么能随意重启呢？！</p><p>后来，我突然想到了Controller组件中重新选举Controller的代码。一旦Controller被选举出来，它就会向所有Broker更新集群元数据，也就是说，会“重刷”这些分区的状态。</p><p>那么问题来了，我们如何在避免重启集群的情况下，干掉已有Controller并执行新的Controller选举呢？答案就在源码中的<strong>ControllerZNode.path</strong>上，也就是ZooKeeper的/controller节点。倘若我们手动删除了/controller节点，Kafka集群就会触发Controller选举。于是，我们马上实施了这个方案，效果出奇得好：之前的受损分区全部恢复正常，业务数据得以正常生产和消费。</p><p>当然，给你分享这个案例的目的，并不是让你记住可以随意干掉/controller节点——这个操作其实是有一点危险的。事实上，我只是想通过这个真实的例子，向你说明，很多打开“精通Kafka之门”的钥匙是隐藏在源码中的。那么，接下来，我们就开始找“钥匙”吧。</p><h2>集群元数据</h2><p>想要完整地了解Controller的工作原理，我们首先就要学习它管理了哪些数据。毕竟，Controller的很多代码仅仅是做数据的管理操作而已。今天，我们就来重点学习Kafka集群元数据都有哪些。</p><p>如果说ZooKeeper是整个Kafka集群元数据的“真理之源（Source of Truth）”，那么Controller可以说是集群元数据的“真理之源副本（Backup Source of Truth）”。好吧，后面这个词是我自己发明的。你只需要理解，Controller承载了ZooKeeper上的所有元数据即可。</p><p>事实上，集群Broker是不会与ZooKeeper直接交互去获取元数据的。相反地，它们总是与Controller进行通信，获取和更新最新的集群数据。而且社区已经打算把ZooKeeper“干掉”了（我会在之后的“特别放送”里具体给你解释社区干掉ZooKeeper的操作），以后Controller将成为新的“真理之源”。</p><p>我们总说元数据，那么，到底什么是集群的元数据，或者说，Kafka集群的元数据都定义了哪些内容呢？我用一张图给你完整地展示一下，当前Kafka定义的所有集群元数据信息。</p><p><img src="https://static001.geekbang.org/resource/image/f1/54/f146aceb78a5da31d887618303b5ff54.jpg?wh=2284*2069" alt=""></p><p>可以看到，目前，Controller定义的元数据有17项之多。不过，并非所有的元数据都同等重要，你也不用完整地记住它们，我们只需要重点关注那些最重要的元数据，并结合源代码来了解下这些元数据都是用来做什么的。</p><p>在了解具体的元数据之前，我要先介绍下ControllerContext类。刚刚我们提到的这些元数据信息全部封装在这个类里。应该这么说，<strong>这个类是Controller组件的数据容器类</strong>。</p><h2>ControllerContext</h2><p>Controller组件的源代码位于core包的src/main/scala/kafka/controller路径下，这里面有很多Scala源文件，<strong>ControllerContext类就位于这个路径下的ControllerContext.scala文件中。</strong></p><p>该文件只有几百行代码，其中，最重要的数据结构就是ControllerContext类。前面说过，<strong>它定义了前面提到的所有元数据信息，以及许多实用的工具方法</strong>。比如，获取集群上所有主题分区对象的allPartitions方法、获取某主题分区副本列表的partitionReplicaAssignment方法，等等。</p><p>首先，我们来看下ControllerContext类的定义，如下所示：</p><pre><code>class ControllerContext {
  val stats = new ControllerStats // Controller统计信息类 
  var offlinePartitionCount = 0   // 离线分区计数器
  val shuttingDownBrokerIds = mutable.Set.empty[Int]  // 关闭中Broker的Id列表
  private val liveBrokers = mutable.Set.empty[Broker] // 当前运行中Broker对象列表
  private val liveBrokerEpochs = mutable.Map.empty[Int, Long] 	// 运行中Broker Epoch列表
  var epoch: Int = KafkaController.InitialControllerEpoch   // Controller当前Epoch值
  var epochZkVersion: Int = KafkaController.InitialControllerEpochZkVersion	// Controller对应ZooKeeper节点的Epoch值
  val allTopics = mutable.Set.empty[String]	// 集群主题列表
  val partitionAssignments = mutable.Map.empty[String, mutable.Map[Int, ReplicaAssignment]]	// 主题分区的副本列表
  val partitionLeadershipInfo = mutable.Map.empty[TopicPartition, LeaderIsrAndControllerEpoch]	// 主题分区的Leader/ISR副本信息
  val partitionsBeingReassigned = mutable.Set.empty[TopicPartition]	// 正处于副本重分配过程的主题分区列表
  val partitionStates = mutable.Map.empty[TopicPartition, PartitionState] // 主题分区状态列表 
  val replicaStates = mutable.Map.empty[PartitionAndReplica, ReplicaState]	// 主题分区的副本状态列表
  val replicasOnOfflineDirs = mutable.Map.empty[Int, Set[TopicPartition]]	// 不可用磁盘路径上的副本列表
  val topicsToBeDeleted = mutable.Set.empty[String]	// 待删除主题列表
  val topicsWithDeletionStarted = mutable.Set.empty[String]	// 已开启删除的主题列表
  val topicsIneligibleForDeletion = mutable.Set.empty[String]	// 暂时无法执行删除的主题列表
  ......
}
</code></pre><p>不多不少，这段代码中定义的字段正好17个，它们一一对应着上图中的那些元数据信息。下面，我选取一些重要的元数据，来详细解释下它们的含义。</p><p>这些元数据理解起来还是比较简单的，掌握了它们之后，你在理解MetadataCache，也就是元数据缓存的时候，就容易得多了。比如，接下来我要讲到的liveBrokers信息，就是Controller通过UpdateMetadataRequest请求同步给其他Broker的MetadataCache的。</p><h3>ControllerStats</h3><p>第一个是ControllerStats类的变量。它的完整代码如下：</p><pre><code>private[controller] class ControllerStats extends KafkaMetricsGroup {
  // 统计每秒发生的Unclean Leader选举次数
  val uncleanLeaderElectionRate = newMeter(&quot;UncleanLeaderElectionsPerSec&quot;, &quot;elections&quot;, TimeUnit.SECONDS)
  // Controller事件通用的统计速率指标的方法
  val rateAndTimeMetrics: Map[ControllerState, KafkaTimer] = ControllerState.values.flatMap { state =&gt;
    state.rateAndTimeMetricName.map { metricName =&gt;
      state -&gt; new KafkaTimer(newTimer(metricName, TimeUnit.MILLISECONDS, TimeUnit.SECONDS))
    }
  }.toMap
}
</code></pre><p>顾名思义，它表征的是Controller的一些统计信息。目前，源码中定义了两大类统计指标：<strong>UncleanLeaderElectionsPerSec和所有Controller事件状态的执行速率与时间</strong>。</p><p>其中，<strong>前者是计算Controller每秒执行的Unclean Leader选举数量，通常情况下，执行Unclean Leader选举可能造成数据丢失，一般不建议开启它</strong>。一旦开启，你就需要时刻关注这个监控指标的值，确保Unclean Leader选举的速率维持在一个很低的水平，否则会出现很多数据丢失的情况。</p><p><strong>后者是统计所有Controller状态的速率和时间信息</strong>，单位是毫秒。当前，Controller定义了很多事件，比如，TopicDeletion是执行主题删除的Controller事件、ControllerChange是执行Controller重选举的事件。ControllerStats的这个指标通过在每个事件名后拼接字符串RateAndTimeMs的方式，为每类Controller事件都创建了对应的速率监控指标。</p><p>由于Controller事件有很多种，对应的速率监控指标也有很多，有一些Controller事件是需要你额外关注的。</p><p>举个例子，IsrChangeNotification事件是标志ISR列表变更的事件，如果这个事件经常出现，说明副本的ISR列表经常发生变化，而这通常被认为是非正常情况，因此，你最好关注下这个事件的速率监控指标。</p><h3>offlinePartitionCount</h3><p><strong>该字段统计集群中所有离线或处于不可用状态的主题分区数量</strong>。所谓的不可用状态，就是我最开始举的例子中“Leader=-1”的情况。</p><p>ControllerContext中的updatePartitionStateMetrics方法根据<strong>给定主题分区的当前状态和目标状态</strong>，来判断该分区是否是离线状态的分区。如果是，则累加offlinePartitionCount字段的值，否则递减该值。方法代码如下：</p><pre><code>// 更新offlinePartitionCount元数据
private def updatePartitionStateMetrics(
  partition: TopicPartition, 
  currentState: PartitionState,
  targetState: PartitionState): Unit = {
  // 如果该主题当前并未处于删除中状态
  if (!isTopicDeletionInProgress(partition.topic)) {
    // targetState表示该分区要变更到的状态
    // 如果当前状态不是OfflinePartition，即离线状态并且目标状态是离线状态
    // 这个if语句判断是否要将该主题分区状态转换到离线状态
    if (currentState != OfflinePartition &amp;&amp; targetState == OfflinePartition) {
      offlinePartitionCount = offlinePartitionCount + 1
    // 如果当前状态已经是离线状态，但targetState不是
    // 这个else if语句判断是否要将该主题分区状态转换到非离线状态
    } else if (currentState == OfflinePartition &amp;&amp; targetState != OfflinePartition) {
      offlinePartitionCount = offlinePartitionCount - 1
    }
  }
}
</code></pre><p>该方法首先要判断，此分区所属的主题当前是否处于删除操作的过程中。如果是的话，Kafka就不能修改这个分区的状态，那么代码什么都不做，直接返回。否则，代码会判断该分区是否要转换到离线状态。如果targetState是OfflinePartition，那么就将offlinePartitionCount值加1，毕竟多了一个离线状态的分区。相反地，如果currentState是offlinePartition，而targetState反而不是，那么就将offlinePartitionCount值减1。</p><h3>shuttingDownBrokerIds</h3><p>顾名思义，<strong>该字段保存所有正在关闭中的Broker ID列表</strong>。当Controller在管理集群Broker时，它要依靠这个字段来甄别Broker当前是否已关闭，因为处于关闭状态的Broker是不适合执行某些操作的，如分区重分配（Reassignment）以及主题删除等。</p><p>另外，Kafka必须要为这些关闭中的Broker执行很多清扫工作，Controller定义了一个onBrokerFailure方法，它就是用来做这个的。代码如下：</p><pre><code>private def onBrokerFailure(deadBrokers: Seq[Int]): Unit = {
  info(s&quot;Broker failure callback for ${deadBrokers.mkString(&quot;,&quot;)}&quot;)
  // deadBrokers：给定的一组已终止运行的Broker Id列表
  // 更新Controller元数据信息，将给定Broker从元数据的replicasOnOfflineDirs中移除
  deadBrokers.foreach(controllerContext.replicasOnOfflineDirs.remove)
  // 找出这些Broker上的所有副本对象
  val deadBrokersThatWereShuttingDown =
    deadBrokers.filter(id =&gt; controllerContext.shuttingDownBrokerIds.remove(id))
  if (deadBrokersThatWereShuttingDown.nonEmpty)
    info(s&quot;Removed ${deadBrokersThatWereShuttingDown.mkString(&quot;,&quot;)} from list of shutting down brokers.&quot;)
  // 执行副本清扫工作
  val allReplicasOnDeadBrokers = controllerContext.replicasOnBrokers(deadBrokers.toSet)
  onReplicasBecomeOffline(allReplicasOnDeadBrokers)
  // 取消这些Broker上注册的ZooKeeper监听器
  unregisterBrokerModificationsHandler(deadBrokers)
}
</code></pre><p>该方法接收一组已终止运行的Broker ID列表，首先是更新Controller元数据信息，将给定Broker从元数据的replicasOnOfflineDirs和shuttingDownBrokerIds中移除，然后为这组Broker执行必要的副本清扫工作，也就是onReplicasBecomeOffline方法做的事情。</p><p>该方法主要依赖于分区状态机和副本状态机来完成对应的工作。在后面的课程中，我们会专门讨论副本状态机和分区状态机，这里你只要简单了解下它要做的事情就行了。后面等我们学完了这两个状态机之后，你可以再看下这个方法的具体实现原理。</p><p>这个方法的主要目的是把给定的副本标记成Offline状态，即不可用状态。具体分为以下这几个步骤：</p><ol>
<li>利用分区状态机将给定副本所在的分区标记为Offline状态；</li>
<li>将集群上所有新分区和Offline分区状态变更为Online状态；</li>
<li>将相应的副本对象状态变更为Offline。</li>
</ol><h3>liveBrokers</h3><p><strong>该字段保存当前所有运行中的Broker对象</strong>。每个Broker对象就是一个&lt;Id，EndPoint，机架信息&gt;的三元组。ControllerContext中定义了很多方法来管理该字段，如addLiveBrokersAndEpochs、removeLiveBrokers和updateBrokerMetadata等。我拿updateBrokerMetadata方法进行说明，以下是源码：</p><pre><code>def updateBrokerMetadata(oldMetadata: Broker, newMetadata: Broker): Unit = {
    liveBrokers -= oldMetadata
    liveBrokers += newMetadata
  }
</code></pre><p>每当新增或移除已有Broker时，ZooKeeper就会更新其保存的Broker数据，从而引发Controller修改元数据，也就是会调用updateBrokerMetadata方法来增减Broker列表中的对象。怎么样，超简单吧？！</p><h3>liveBrokerEpochs</h3><p><strong>该字段保存所有运行中Broker的Epoch信息</strong>。Kafka使用Epoch数据防止Zombie Broker，即一个非常老的Broker被选举成为Controller。</p><p>另外，源码大多使用这个字段来获取所有运行中Broker的ID序号，如下面这个方法定义的那样：</p><pre><code>def liveBrokerIds: Set[Int] = liveBrokerEpochs.keySet -- shuttingDownBrokerIds
</code></pre><p>liveBrokerEpochs的keySet方法返回Broker序号列表，然后从中移除关闭中的Broker序号，剩下的自然就是处于运行中的Broker序号列表了。</p><h3>epoch &amp; epochZkVersion</h3><p>这两个字段一起说，因为它们都有“epoch”字眼，放在一起说，可以帮助你更好地理解两者的区别。epoch实际上就是ZooKeeper中/controller_epoch节点的值，你可以认为它就是Controller在整个Kafka集群的版本号，而epochZkVersion实际上是/controller_epoch节点的dataVersion值。</p><p>Kafka使用epochZkVersion来判断和防止Zombie Controller。这也就是说，原先在老Controller任期内的Controller操作在新Controller不能成功执行，因为新Controller的epochZkVersion要比老Controller的大。</p><p>另外，你可能会问：“这里的两个Epoch和上面的liveBrokerEpochs有啥区别呢？”实际上，这里的两个Epoch值都是属于Controller侧的数据，而liveBrokerEpochs是每个Broker自己的Epoch值。</p><h3>allTopics</h3><p><strong>该字段保存集群上所有的主题名称</strong>。每当有主题的增减，Controller就要更新该字段的值。</p><p>比如Controller有个processTopicChange方法，从名字上来看，它就是处理主题变更的。我们来看下它的代码实现，我把主要逻辑以注释的方式标注了出来：</p><pre><code>private def processTopicChange(): Unit = {
    if (!isActive) return // 如果Contorller已经关闭，直接返回
    val topics = zkClient.getAllTopicsInCluster(true) // 从ZooKeeper中获取当前所有主题列表
    val newTopics = topics -- controllerContext.allTopics // 找出当前元数据中不存在、ZooKeeper中存在的主题，视为新增主题
    val deletedTopics = controllerContext.allTopics -- topics // 找出当前元数据中存在、ZooKeeper中不存在的主题，视为已删除主题
    controllerContext.allTopics = topics // 更新Controller元数据
    // 为新增主题和已删除主题执行后续处理操作
    registerPartitionModificationsHandlers(newTopics.toSeq)
    val addedPartitionReplicaAssignment = zkClient.getFullReplicaAssignmentForTopics(newTopics)
    deletedTopics.foreach(controllerContext.removeTopic)
    addedPartitionReplicaAssignment.foreach {
      case (topicAndPartition, newReplicaAssignment) =&gt; controllerContext.updatePartitionFullReplicaAssignment(topicAndPartition, newReplicaAssignment)
    }
    info(s&quot;New topics: [$newTopics], deleted topics: [$deletedTopics], new partition replica assignment &quot; +
      s&quot;[$addedPartitionReplicaAssignment]&quot;)
    if (addedPartitionReplicaAssignment.nonEmpty)
      onNewPartitionCreation(addedPartitionReplicaAssignment.keySet)
  }

</code></pre><h3>partitionAssignments</h3><p><strong>该字段保存所有主题分区的副本分配情况</strong>。在我看来，<strong>这是Controller最重要的元数据了</strong>。事实上，你可以从这个字段衍生、定义很多实用的方法，来帮助Kafka从各种维度获取数据。</p><p>比如，如果Kafka要获取某个Broker上的所有分区，那么，它可以这样定义：</p><pre><code>partitionAssignments.flatMap {
      case (topic, topicReplicaAssignment) =&gt; topicReplicaAssignment.filter {
        case (_, partitionAssignment) =&gt; partitionAssignment.replicas.contains(brokerId)
      }.map {
        case (partition, _) =&gt; new TopicPartition(topic, partition)
      }
    }.toSet
</code></pre><p>再比如，如果Kafka要获取某个主题的所有分区对象，代码可以这样写：</p><pre><code>partitionAssignments.getOrElse(topic, mutable.Map.empty).map {
      case (partition, _) =&gt; new TopicPartition(topic, partition)
    }.toSet
</code></pre><p>实际上，这两段代码分别是ControllerContext.scala中partitionsOnBroker方法和partitionsForTopic两个方法的主体实现代码。</p><p>讲到这里，9个重要的元数据字段我就介绍完了。前面说过，ControllerContext中一共定义了17个元数据字段，你可以结合这9个字段，把其余8个的定义也过一遍，做到心中有数。<strong>你对Controller元数据掌握得越好，就越能清晰地理解Controller在集群中发挥的作用</strong>。</p><p>值得注意的是，在学习每个元数据字段时，除了它的定义之外，我建议你去搜索一下，与之相关的工具方法都是如何实现的。如果后面你想要新增获取或更新元数据的方法，你要对操作它们的代码有很强的把控力才行。</p><h2>总结</h2><p>今天，我们揭开了Kafka重要组件Controller的学习大幕。我给出了Controller模块的学习路线，还介绍了Controller的重要元数据。</p><ul>
<li>Controller元数据：Controller当前定义了17种元数据，涵盖Kafka集群数据的方方面面。</li>
<li>ControllerContext：定义元数据以及操作它们的类。</li>
<li>关键元数据字段：最重要的元数据包括offlinePartitionCount、liveBrokers、partitionAssignments等。</li>
<li>ControllerContext工具方法：ControllerContext 类定义了很多实用方法来管理这些元数据信息。</li>
</ul><p>下节课，我们将学习Controller是如何给Broker发送请求的。Controller与Broker进行交互与通信，是Controller奠定王者地位的重要一环，我会向你详细解释它是如何做到这一点的。</p><h2>课后讨论</h2><p>我今天并未给出所有的元数据说明，请你自行结合代码分析一下，partitionLeadershipInfo里面保存的是什么数据？</p><p>欢迎你在留言区写下你的思考和答案，跟我交流讨论，也欢迎你把今天的内容分享给你的朋友。</p>
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
  <div class="_2_QraFYR_0">你好，我是胡夕。我来公布上节课的“课后讨论”题答案啦～<br><br>上节课，咱们重点了解了Kafka中重要的源码入口类KafkaApis。课后我请你思考如果一个Consumer要向Broker提交位移，它应该具备什么权限以及声明权限的具体代码位置哪段代码。下面我给出我的答案：消费者要具有GROUP的READ权限和TOPIC的READ权限才能提交位移。具体代码位置在handleOffsetCommitRequest方法的这两行中：<br>if (!authorize(request.context, READ, GROUP, offsetCommitRequest.data.groupId)) {<br>	......<br>} else {<br>	......<br>	val authorizedTopics = filterByAuthorized(request.context, READ, TOPIC, topics)(_.name)<br>	......<br>}<br><br>okay，你同意这个说法吗？或者说你有其他的看法吗？我们可以一起讨论下。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-19 11:16:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/25/7f/473d5a77.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>曾轼麟</span>
  </div>
  <div class="_2_QraFYR_0">我个人比较期待kafka摆脱zookeeper的时候，之前看过一篇文章，对比kafka和RocketMQ的性能对比，其中总结出，kafka的性能会受到topic数量的增加而下降，看了源码后才逐渐明白其实制约kafka的正是zookeeper</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 个人感觉性能的那个问题和ZooKeeper关系不大。主要还是分区路径太分散导致顺序IO变为随机IO</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-31 11:09:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/25/7f/473d5a77.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>曾轼麟</span>
  </div>
  <div class="_2_QraFYR_0">partitionLeadershipInfo存储的主要是leader，leaderEpoch，isr集合，zkVersion，这些都定义在LeaderAndIsr这个类里面</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-31 11:04:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/ca/bd/a51ae4b2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>吃饭饭</span>
  </div>
  <div class="_2_QraFYR_0">z这里怎么理解【将集群上所有新分区和 Offline 分区状态变更为 Online 状态；】直接把 deadBrokers 标记为 Offline 不就可以了吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 没太明白这个问题。到底是online还是offline的问题？<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-05 09:22:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/23/4c/a6/e6f1d773.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>鸣己</span>
  </div>
  <div class="_2_QraFYR_0">我有遇到过一个集群有三个controller;它们好像谁也不服谁，导致好多topic没leader</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-25 17:25:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/9a/25/3e5e942b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张子涵</span>
  </div>
  <div class="_2_QraFYR_0">val partitionLeadershipInfo = mutable.Map.empty[TopicPartition, LeaderIsrAndControllerEpoch]<br><br>def removeTopic(topic: String): Unit = {<br>    allTopics -= topic<br>    partitionAssignments.remove(topic)<br>    partitionLeadershipInfo.foreach {<br>      case (topicPartition, _) if topicPartition.topic == topic =&gt; partitionLeadershipInfo.remove(topicPartition)<br>      case _ =&gt;<br>    }<br>  }<br>根据partitionLeadershipInfo定义，以及在removeTopic方法中partitionLeadershipInfo的应用，可大概理解partitionLeadershipInfo中存储的是分区以及leader epoch  类似变更版本信息</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-24 15:14:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/25/7f/473d5a77.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>曾轼麟</span>
  </div>
  <div class="_2_QraFYR_0">在kafka中我发现经常使用epoch的方式来判断版本新旧，其实epoch这种设计思想类似于乐观锁的方式</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 应该算是token机制的一种</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-31 11:06:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/ef/6b/5e8f6536.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>伯安知心</span>
  </div>
  <div class="_2_QraFYR_0">您说删除的watcher，oncontrollerfailover重新初始化上下文，我感觉代价昂贵了些，就应该做这么多事吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以具体说说哪一步在您看来是多余的，也许是个优化的方向：）<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-15 06:34:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/ef/6b/5e8f6536.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>伯安知心</span>
  </div>
  <div class="_2_QraFYR_0">还有请问partitionLeadershipInfo中controller的epoch是干吗的呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你可以认为是controller换了多少次。比如epoch = 5，说明controller前前后后更换过6次</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-15 06:29:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/ef/6b/5e8f6536.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>伯安知心</span>
  </div>
  <div class="_2_QraFYR_0">partitionLeadershipInfo总的来说每个分区对应的分区主副本，isr集合，还有controller的epoch数，</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-15 06:28:18</div>
  </div>
</div>
</div>
</li>
</ul>