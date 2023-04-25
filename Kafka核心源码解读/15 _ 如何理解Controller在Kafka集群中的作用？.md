<audio title="15 _ 如何理解Controller在Kafka集群中的作用？" src="https://static001.geekbang.org/resource/audio/5a/41/5afb796241d45b2e75d1f295067f0441.mp3" controls="controls"></audio> 
<p>你好，我是胡夕。</p><p>上节课，我们学习了Controller选举的源码，了解了Controller组件的选举触发场景，以及它是如何被选举出来的。Controller就绪之后，就会行使它作为控制器的重要权利了，包括管理集群成员、维护主题、操作元数据，等等。</p><p>之前在学习Kafka的时候，我一直很好奇，新启动的Broker是如何加入到集群中的。官方文档里的解释是：“Adding servers to a Kafka cluster is easy, just assign them a unique broker id and start up Kafka on your new servers.”显然，你只要启动Broker进程，就可以实现集群的扩展，甚至包括集群元数据信息的同步。</p><p>不过，你是否思考过，这一切是怎么做到的呢？其实，这就是Controller组件源码提供的一个重要功能：管理新集群成员。</p><p>当然，作为核心组件，Controller提供的功能非常多。除了集群成员管理，主题管理也是一个极其重要的功能。今天，我就带你深入了解下它们的实现代码。可以说，这是Controller最核心的两个功能，它们几乎涉及到了集群元数据中的所有重要数据。掌握了这些，之后你在探索Controller的其他代码时，会更加游刃有余。</p><!-- [[[read_end]]] --><h2>集群成员管理</h2><p>首先，我们来看Controller管理集群成员部分的代码。这里的成员管理包含两个方面：</p><ol>
<li>成员数量的管理，主要体现在新增成员和移除现有成员；</li>
<li>单个成员的管理，如变更单个Broker的数据等。</li>
</ol><h3>成员数量管理</h3><p>每个Broker在启动的时候，会在ZooKeeper的/brokers/ids节点下创建一个名为broker.id参数值的临时节点。</p><p>举个例子，假设Broker的broker.id参数值设置为1001，那么，当Broker启动后，你会在ZooKeeper的/brokers/ids下观测到一个名为1001的子节点。该节点的内容包括了Broker配置的主机名、端口号以及所用监听器的信息（注意：这里的监听器和上面说的ZooKeeper监听器不是一回事）。</p><p>当该Broker正常关闭或意外退出时，ZooKeeper上对应的临时节点会自动消失。</p><p>基于这种临时节点的机制，Controller定义了BrokerChangeHandler监听器，专门负责监听/brokers/ids下的子节点数量变化。</p><p>一旦发现新增或删除Broker，/brokers/ids下的子节点数目一定会发生变化。这会被Controller侦测到，进而触发BrokerChangeHandler的处理方法，即handleChildChange方法。</p><p>我给出BrokerChangeHandler的代码。可以看到，这里面定义了handleChildChange方法：</p><pre><code>class BrokerChangeHandler(eventManager: ControllerEventManager) extends ZNodeChildChangeHandler {
  // Broker ZooKeeper ZNode: /brokers/ids 
  override val path: String = BrokerIdsZNode.path
  override def handleChildChange(): Unit = {
    eventManager.put(BrokerChange) // 仅仅是向事件队列写入BrokerChange事件
  }
}
</code></pre><p>该方法的作用就是向Controller事件队列写入一个BrokerChange事件。<strong>事实上，Controller端定义的所有Handler的处理逻辑，都是向事件队列写入相应的ControllerEvent，真正的事件处理逻辑位于KafkaController类的process方法中。</strong></p><p>那么，接下来，我们就来看process方法。你会发现，处理BrokerChange事件的方法实际上是processBrokerChange，代码如下：</p><pre><code>private def processBrokerChange(): Unit = {
  // 如果该Broker不是Controller，自然无权处理，直接返回
  if (!isActive) return
  // 第1步：从ZooKeeper中获取集群Broker列表
  val curBrokerAndEpochs = zkClient.getAllBrokerAndEpochsInCluster
  val curBrokerIdAndEpochs = curBrokerAndEpochs map { case (broker, epoch) =&gt; (broker.id, epoch) }
  val curBrokerIds = curBrokerIdAndEpochs.keySet
  // 第2步：获取Controller当前保存的Broker列表
  val liveOrShuttingDownBrokerIds = controllerContext.liveOrShuttingDownBrokerIds
  // 第3步：比较两个列表，获取新增Broker列表、待移除Broker列表、
  // 已重启Broker列表和当前运行中的Broker列表
  val newBrokerIds = curBrokerIds.diff(liveOrShuttingDownBrokerIds)
  val deadBrokerIds = liveOrShuttingDownBrokerIds.diff(curBrokerIds)
  val bouncedBrokerIds = (curBrokerIds &amp; liveOrShuttingDownBrokerIds)
    .filter(brokerId =&gt; curBrokerIdAndEpochs(brokerId) &gt; controllerContext.liveBrokerIdAndEpochs(brokerId))
  val newBrokerAndEpochs = curBrokerAndEpochs.filter { case (broker, _) =&gt; newBrokerIds.contains(broker.id) }
  val bouncedBrokerAndEpochs = curBrokerAndEpochs.filter { case (broker, _) =&gt; bouncedBrokerIds.contains(broker.id) }
  val newBrokerIdsSorted = newBrokerIds.toSeq.sorted
  val deadBrokerIdsSorted = deadBrokerIds.toSeq.sorted
  val liveBrokerIdsSorted = curBrokerIds.toSeq.sorted
  val bouncedBrokerIdsSorted = bouncedBrokerIds.toSeq.sorted
  info(s&quot;Newly added brokers: ${newBrokerIdsSorted.mkString(&quot;,&quot;)}, &quot; +
    s&quot;deleted brokers: ${deadBrokerIdsSorted.mkString(&quot;,&quot;)}, &quot; +
    s&quot;bounced brokers: ${bouncedBrokerIdsSorted.mkString(&quot;,&quot;)}, &quot; +
    s&quot;all live brokers: ${liveBrokerIdsSorted.mkString(&quot;,&quot;)}&quot;)
  // 第4步：为每个新增Broker创建与之连接的通道管理器和
  // 底层的请求发送线程（RequestSendThread）
  newBrokerAndEpochs.keySet.foreach(
    controllerChannelManager.addBroker)
  // 第5步：为每个已重启的Broker移除它们现有的配套资源
  //（通道管理器、RequestSendThread等），并重新添加它们
  bouncedBrokerIds.foreach(controllerChannelManager.removeBroker)
  bouncedBrokerAndEpochs.keySet.foreach(
    controllerChannelManager.addBroker)
  // 第6步：为每个待移除Broker移除对应的配套资源
  deadBrokerIds.foreach(controllerChannelManager.removeBroker)
  // 第7步：为新增Broker执行更新Controller元数据和Broker启动逻辑
  if (newBrokerIds.nonEmpty) {
    controllerContext.addLiveBrokers(newBrokerAndEpochs)
    onBrokerStartup(newBrokerIdsSorted)
  }
  // 第8步：为已重启Broker执行重添加逻辑，包含
  // 更新ControllerContext、执行Broker重启动逻辑
  if (bouncedBrokerIds.nonEmpty) {
    controllerContext.removeLiveBrokers(bouncedBrokerIds)
    onBrokerFailure(bouncedBrokerIdsSorted)
    controllerContext.addLiveBrokers(bouncedBrokerAndEpochs)
    onBrokerStartup(bouncedBrokerIdsSorted)
  }
  // 第9步：为待移除Broker执行移除ControllerContext和Broker终止逻辑
  if (deadBrokerIds.nonEmpty) {
    controllerContext.removeLiveBrokers(deadBrokerIds)
    onBrokerFailure(deadBrokerIdsSorted)
  }
  if (newBrokerIds.nonEmpty || deadBrokerIds.nonEmpty ||
   bouncedBrokerIds.nonEmpty) {
    info(s&quot;Updated broker epochs cache: ${controllerContext.liveBrokerIdAndEpochs}&quot;)
  }
}
</code></pre><p>代码有点长，你可以看下我添加的重点注释。同时，我再画一张图，帮你梳理下这个方法做的事情。</p><p><img src="https://static001.geekbang.org/resource/image/ff/d3/fffc8456d8ede9219462e607fa4241d3.jpg?wh=2800*1784" alt=""></p><p>整个方法共有9步。</p><p>第1~3步：</p><p>前两步分别是从ZooKeeper和ControllerContext中获取Broker列表；第3步是获取4个Broker列表：新增Broker列表、待移除Broker列表、已重启的Broker列表和当前运行中的Broker列表。</p><p>假设前两步中的Broker列表分别用A和B表示，由于Kafka以ZooKeeper上的数据为权威数据，因此，A就是最新的运行中Broker列表，“A-B”就表示新增的Broker，“B-A”就表示待移除的Broker。</p><p>已重启的Broker的判断逻辑要复杂一些，它判断的是A∧B集合中的那些Epoch值变更了的Broker。你大体上可以把Epoch值理解为Broker的版本或重启的次数。显然，Epoch值变更了，就说明Broker发生了重启行为。</p><p>第4~9步：</p><p>拿到这些集合之后，Controller会分别为这4个Broker列表执行相应的操作，也就是这个方法中第4~9步要做的事情。总体上，这些相应的操作分为3类。</p><ul>
<li>执行元数据更新操作：调用ControllerContext类的各个方法，更新不同的集群元数据信息。比如需要将新增Broker加入到集群元数据，将待移除Broker从元数据中移除等。</li>
<li>执行Broker终止操作：为待移除Broker和已重启Broker调用onBrokerFailure方法。</li>
<li>执行Broker启动操作：为已重启Broker和新增Broker调用onBrokerStartup方法。</li>
</ul><p>下面我们深入了解下onBrokerFailure和onBrokerStartup方法的逻辑。相比于其他方法，这两个方法的代码逻辑有些复杂，要做的事情也很多，因此，我们重点研究下它们。</p><p>首先是处理Broker终止逻辑的onBrokerFailure方法，代码如下：</p><pre><code>private def onBrokerFailure(deadBrokers: Seq[Int]): Unit = {
  info(s&quot;Broker failure callback for ${deadBrokers.mkString(&quot;,&quot;)}&quot;)
  // 第1步：为每个待移除Broker，删除元数据对象中的相关项
  deadBrokers.foreach(controllerContext.replicasOnOfflineDirs.remove
  // 第2步：将待移除Broker从元数据对象中处于已关闭状态的Broker列表中去除             
  val deadBrokersThatWereShuttingDown =
    deadBrokers.filter(id =&gt; controllerContext.shuttingDownBrokerIds.remove(id))
  if (deadBrokersThatWereShuttingDown.nonEmpty)
    info(s&quot;Removed ${deadBrokersThatWereShuttingDown.mkString(&quot;,&quot;)} from list of shutting down brokers.&quot;)
  // 第3步：找出待移除Broker上的所有副本对象，执行相应操作，
  // 将其置为“不可用状态”（即Offline）  
  val allReplicasOnDeadBrokers = controllerContext.replicasOnBrokers(deadBrokers.toSet)
  onReplicasBecomeOffline(allReplicasOnDeadBrokers)
  // 第4步：注销注册的BrokerModificationsHandler监听器
  unregisterBrokerModificationsHandler(deadBrokers)
}
</code></pre><p>Broker终止，意味着我们必须要删除Controller元数据缓存中与之相关的所有项，还要处理这些Broker上保存的副本。最后，我们还要注销之前为该Broker注册的BrokerModificationsHandler监听器。</p><p>其实，主体逻辑在于如何处理Broker上的副本对象，即onReplicasBecomeOffline方法。该方法大量调用了Kafka副本管理器和分区管理器的相关功能，后面我们会专门学习这两个管理器，因此这里我就不展开讲了。</p><p>现在，我们看onBrokerStartup方法。它是处理Broker启动的方法，也就是Controller端应对集群新增Broker启动的方法。同样，我先给出带注释的完整方法代码：</p><pre><code>private def onBrokerStartup(newBrokers: Seq[Int]): Unit = {
  info(s&quot;New broker startup callback for ${newBrokers.mkString(&quot;,&quot;)}&quot;)
  // 第1步：移除元数据中新增Broker对应的副本集合
  newBrokers.foreach(controllerContext.replicasOnOfflineDirs.remove)
  val newBrokersSet = newBrokers.toSet
  val existingBrokers = controllerContext.
    liveOrShuttingDownBrokerIds.diff(newBrokersSet)
  // 第2步：给集群现有Broker发送元数据更新请求，令它们感知到新增Broker的到来
  sendUpdateMetadataRequest(existingBrokers.toSeq, Set.empty)
  // 第3步：给新增Broker发送元数据更新请求，令它们同步集群当前的所有分区数据
  sendUpdateMetadataRequest(newBrokers, controllerContext.partitionLeadershipInfo.keySet)
  val allReplicasOnNewBrokers = controllerContext.replicasOnBrokers(newBrokersSet)
  // 第4步：将新增Broker上的所有副本设置为Online状态，即可用状态
  replicaStateMachine.handleStateChanges(
    allReplicasOnNewBrokers.toSeq, OnlineReplica)
  partitionStateMachine.triggerOnlinePartitionStateChange()
  // 第5步：重启之前暂停的副本迁移操作
  maybeResumeReassignments { (_, assignment) =&gt;
    assignment.targetReplicas.exists(newBrokersSet.contains)
  }
  val replicasForTopicsToBeDeleted = allReplicasOnNewBrokers.filter(p =&gt; topicDeletionManager.isTopicQueuedUpForDeletion(p.topic))
  // 第6步：重启之前暂停的主题删除操作
  if (replicasForTopicsToBeDeleted.nonEmpty) {
    info(s&quot;Some replicas ${replicasForTopicsToBeDeleted.mkString(&quot;,&quot;)} for topics scheduled for deletion &quot; +
      s&quot;${controllerContext.topicsToBeDeleted.mkString(&quot;,&quot;)} are on the newly restarted brokers &quot; +
      s&quot;${newBrokers.mkString(&quot;,&quot;)}. Signaling restart of topic deletion for these topics&quot;)
   topicDeletionManager.resumeDeletionForTopics(
     replicasForTopicsToBeDeleted.map(_.topic))
  }
  // 第7步：为新增Broker注册BrokerModificationsHandler监听器
  registerBrokerModificationsHandler(newBrokers)
}
</code></pre><p>如代码所示，第1步是移除新增Broker在元数据缓存中的信息。你可能会问：“这些Broker不都是新增的吗？元数据缓存中有它们的数据吗？”实际上，这里的newBrokers仅仅表示新启动的Broker，它们不一定是全新的Broker。因此，这里的删除元数据缓存是非常安全的做法。</p><p>第2、3步：分别给集群的已有Broker和新增Broker发送更新元数据请求。这样一来，整个集群上的Broker就可以互相感知到彼此，而且最终所有的Broker都能保存相同的分区数据。</p><p>第4步：将新增Broker上的副本状态置为Online状态。Online状态表示这些副本正常提供服务，即Leader副本对外提供读写服务，Follower副本自动向Leader副本同步消息。</p><p>第5、6步：分别重启可能因为新增Broker启动、而能够重新被执行的副本迁移和主题删除操作。</p><p>第7步：为所有新增Broker注册BrokerModificationsHandler监听器，允许Controller监控它们在ZooKeeper上的节点的数据变更。</p><h3>成员信息管理</h3><p>了解了Controller管理集群成员数量的机制之后，接下来，我们要重点学习下Controller如何监听Broker端信息的变更，以及具体的操作。</p><p>和管理集群成员类似，Controller也是通过ZooKeeper监听器的方式来应对Broker的变化。这个监听器就是BrokerModificationsHandler。一旦Broker的信息发生变更，该监听器的handleDataChange方法就会被调用，向事件队列写入BrokerModifications事件。</p><p>KafkaController类的processBrokerModification方法负责处理这类事件，代码如下：</p><pre><code>private def processBrokerModification(brokerId: Int): Unit = {
  if (!isActive) return
  // 第1步：获取目标Broker的详细数据，
  // 包括每套监听器配置的主机名、端口号以及所使用的安全协议等
  val newMetadataOpt = zkClient.getBroker(brokerId)
  // 第2步：从元数据缓存中获得目标Broker的详细数据
  val oldMetadataOpt = controllerContext.liveOrShuttingDownBroker(brokerId)
  if (newMetadataOpt.nonEmpty &amp;&amp; oldMetadataOpt.nonEmpty) {
    val oldMetadata = oldMetadataOpt.get
    val newMetadata = newMetadataOpt.get
    // 第3步：如果两者不相等，说明Broker数据发生了变更
    // 那么，更新元数据缓存，以及执行onBrokerUpdate方法处理Broker更新的逻辑
    if (newMetadata.endPoints != oldMetadata.endPoints) {
      info(s&quot;Updated broker metadata: $oldMetadata -&gt; $newMetadata&quot;)
      controllerContext.updateBrokerMetadata(oldMetadata, newMetadata)
      onBrokerUpdate(brokerId)
    }
  }
}

</code></pre><p>该方法首先获取ZooKeeper上最权威的Broker数据，将其与元数据缓存上的数据进行比对。如果发现两者不一致，就会更新元数据缓存，同时调用onBrokerUpdate方法执行更新逻辑。</p><p>那么，onBrokerUpdate方法又是如何实现的呢？我们先看下代码：</p><pre><code>private def onBrokerUpdate(updatedBrokerId: Int): Unit = {
  info(s&quot;Broker info update callback for $updatedBrokerId&quot;)
  // 给集群所有Broker发送UpdateMetadataRequest，让她它们去更新元数据
  sendUpdateMetadataRequest(
    controllerContext.liveOrShuttingDownBrokerIds.toSeq, Set.empty)
}
</code></pre><p>可以看到，onBrokerUpdate就是向集群所有Broker发送更新元数据信息请求，把变更信息广播出去。</p><p>怎么样，应对Broker信息变更的方法还是比较简单的吧？</p><h2>主题管理</h2><p>除了维护集群成员之外，Controller还有一个重要的任务，那就是对所有主题进行管理，主要包括主题的创建、变更与删除。</p><p>掌握了前面集群成员管理的方法，在学习下面的内容时会轻松很多。因为它们的实现机制是一脉相承的，几乎没有任何差异。</p><h3>主题创建/变更</h3><p>我们重点学习下主题是如何被创建的。实际上，主题变更与创建是相同的逻辑，因此，源码使用了一套监听器统一处理这两种情况。</p><p>你一定使用过Kafka的kafka-topics脚本或AdminClient创建主题吧？实际上，这些工具仅仅是向ZooKeeper对应的目录下写入相应的数据而已，那么，Controller，或者说Kafka集群是如何感知到新创建的主题的呢？</p><p>这当然要归功于监听主题路径的ZooKeeper监听器：TopicChangeHandler。代码如下：</p><pre><code>class TopicChangeHandler(eventManager: ControllerEventManager) extends ZNodeChildChangeHandler {
  // ZooKeeper节点：/brokers/topics
  override val path: String = TopicsZNode.path
  // 向事件队列写入TopicChange事件
  override def handleChildChange(): Unit = eventManager.put(TopicChange)
}
</code></pre><p>代码中的TopicsZNode.path就是ZooKeeper下/brokers/topics节点。一旦该节点下新增了主题信息，该监听器的handleChildChange就会被触发，Controller通过ControllerEventManager对象，向事件队列写入TopicChange事件。</p><p>KafkaController的process方法接到该事件后，调用processTopicChange方法执行主题创建。代码如下：</p><pre><code>private def processTopicChange(): Unit = {
  if (!isActive) return
  // 第1步：从ZooKeeper中获取所有主题
  val topics = zkClient.getAllTopicsInCluster(true)
  // 第2步：与元数据缓存比对，找出新增主题列表与已删除主题列表
  val newTopics = topics -- controllerContext.allTopics
  val deletedTopics = controllerContext.allTopics.diff(topics)
  // 第3步：使用ZooKeeper中的主题列表更新元数据缓存
  controllerContext.setAllTopics(topics)
  // 第4步：为新增主题注册分区变更监听器
  // 分区变更监听器是监听主题分区变更的
  registerPartitionModificationsHandlers(newTopics.toSeq)
  // 第5步：从ZooKeeper中获取新增主题的副本分配情况
  val addedPartitionReplicaAssignment = zkClient.getFullReplicaAssignmentForTopics(newTopics)
  // 第6步：清除元数据缓存中属于已删除主题的缓存项
  deletedTopics.foreach(controllerContext.removeTopic)
  // 第7步：为新增主题更新元数据缓存中的副本分配条目
  addedPartitionReplicaAssignment.foreach {
    case (topicAndPartition, newReplicaAssignment) =&gt; controllerContext.updatePartitionFullReplicaAssignment(topicAndPartition, newReplicaAssignment)
  }
  info(s&quot;New topics: [$newTopics], deleted topics: [$deletedTopics], new partition replica assignment &quot; +
    s&quot;[$addedPartitionReplicaAssignment]&quot;)
  // 第8步：调整新增主题所有分区以及所属所有副本的运行状态为“上线”状态
  if (addedPartitionReplicaAssignment.nonEmpty)
    onNewPartitionCreation(addedPartitionReplicaAssignment.keySet)
}

</code></pre><p>虽然一共有8步，但大部分的逻辑都与更新元数据缓存项有关，因此，处理逻辑总体上还是比较简单的。需要注意的是，第8步涉及到了使用分区管理器和副本管理器来调整分区和副本状态。后面我们会详细介绍。这里你只需要知道，分区和副本处于“上线”状态，就表明它们能够正常工作，就足够了。</p><h3>主题删除</h3><p>和主题创建或变更类似，删除主题也依赖ZooKeeper监听器完成。</p><p>Controller定义了TopicDeletionHandler，用它来实现对删除主题的监听，代码如下：</p><pre><code>class TopicDeletionHandler(eventManager: ControllerEventManager) extends ZNodeChildChangeHandler {
  // ZooKeeper节点：/admin/delete_topics
  override val path: String = DeleteTopicsZNode.path
  // 向事件队列写入TopicDeletion事件
  override def handleChildChange(): Unit = eventManager.put(TopicDeletion)
}
</code></pre><p>这里的DeleteTopicsZNode.path指的是/admin/delete_topics节点。目前，无论是kafka-topics脚本，还是AdminClient，删除主题都是在/admin/delete_topics节点下创建名为待删除主题名的子节点。</p><p>比如，如果我要删除test-topic主题，那么，Kafka的删除命令仅仅是在ZooKeeper上创建/admin/delete_topics/test-topic节点。一旦监听到该节点被创建，TopicDeletionHandler的handleChildChange方法就会被触发，Controller会向事件队列写入TopicDeletion事件。</p><p>处理TopicDeletion事件的方法是processTopicDeletion，代码如下：</p><pre><code>private def processTopicDeletion(): Unit = {
  if (!isActive) return
  // 从ZooKeeper中获取待删除主题列表
  var topicsToBeDeleted = zkClient.getTopicDeletions.toSet
  debug(s&quot;Delete topics listener fired for topics ${topicsToBeDeleted.mkString(&quot;,&quot;)} to be deleted&quot;)
  // 找出不存在的主题列表
  val nonExistentTopics = topicsToBeDeleted -- controllerContext.allTopics
  if (nonExistentTopics.nonEmpty) {
    warn(s&quot;Ignoring request to delete non-existing topics ${nonExistentTopics.mkString(&quot;,&quot;)}&quot;)
    zkClient.deleteTopicDeletions(nonExistentTopics.toSeq, controllerContext.epochZkVersion)
  }
  topicsToBeDeleted --= nonExistentTopics
  // 如果delete.topic.enable参数设置成true
  if (config.deleteTopicEnable) {
    if (topicsToBeDeleted.nonEmpty) {
      info(s&quot;Starting topic deletion for topics ${topicsToBeDeleted.mkString(&quot;,&quot;)}&quot;)
      topicsToBeDeleted.foreach { topic =&gt;
        val partitionReassignmentInProgress = controllerContext.partitionsBeingReassigned.map(_.topic).contains(topic)
        if (partitionReassignmentInProgress)
          topicDeletionManager.markTopicIneligibleForDeletion(
            Set(topic), reason = &quot;topic reassignment in progress&quot;)
      }
      // 将待删除主题插入到删除等待集合交由TopicDeletionManager处理
      topicDeletionManager.enqueueTopicsForDeletion(topicsToBeDeleted)
    }
  } else { // 不允许删除主题
    info(s&quot;Removing $topicsToBeDeleted since delete topic is disabled&quot;)
    // 清除ZooKeeper下/admin/delete_topics下的子节点
    zkClient.deleteTopicDeletions(topicsToBeDeleted.toSeq, controllerContext.epochZkVersion)
  }
}
</code></pre><p>为了帮助你更直观地理解，我再画一张图来说明下：</p><p><img src="https://static001.geekbang.org/resource/image/97/c9/976d35f7771f4cd5ef94eda856fb53c9.jpg?wh=2400*2985" alt=""></p><p>首先，代码从ZooKeeper的/admin/delete_topics下获取子节点列表，即待删除主题列表。</p><p>之后，比对元数据缓存中的主题列表，获知压根不存在的主题列表。如果确实有不存在的主题，删除/admin/delete_topics下对应的子节点就行了。同时，代码会更新待删除主题列表，将这些不存在的主题剔除掉。</p><p>接着，代码会检查Broker端参数delete.topic.enable的值。如果该参数为false，即不允许删除主题，代码就会清除ZooKeeper下的对应子节点，不会做其他操作。反之，代码会遍历待删除主题列表，将那些正在执行分区迁移的主题暂时设置成“不可删除”状态。</p><p>最后，把剩下可以删除的主题交由TopicDeletionManager，由它执行真正的删除逻辑。</p><p>这里的TopicDeletionManager是Kafka专门负责删除主题的管理器，下节课我会详细讲解它的代码实现。</p><h2>总结</h2><p>今天，我们学习了Controller的两个主要功能：管理集群Broker成员和主题。这两个功能是Controller端提供的重要服务。我建议你仔细地查看这两部分的源码，弄明白Controller是如何管理集群中的重要资源的。</p><p>针对这些内容，我总结了几个重点，希望可以帮助你更好地理解和记忆。</p><ul>
<li>集群成员管理：Controller负责对集群所有成员进行有效管理，包括自动发现新增Broker、自动处理下线Broker，以及及时响应Broker数据的变更。</li>
<li>主题管理：Controller负责对集群上的所有主题进行高效管理，包括创建主题、变更主题以及删除主题，等等。对于删除主题而言，实际的删除操作由底层的TopicDeletionManager完成。</li>
</ul><p><img src="https://static001.geekbang.org/resource/image/00/37/0035a579a02def8f5234831bf0857f37.jpg?wh=2250*1149" alt=""></p><p>接下来，我们将进入到下一个模块：状态机模块。在该模块中，我们将系统学习Kafka提供的三大状态机或管理器。Controller非常依赖这些状态机对下辖的所有Kafka对象进行管理。在下一个模块中，我将带你深入了解分区或副本在底层的状态流转是怎么样的，你一定不要错过。</p><h2>课后讨论</h2><p>如果我们想要使用脚本命令增加一个主题的分区，你知道应该用KafkaController类中的哪个方法吗？</p><p>欢迎你在留言区畅所欲言，跟我交流讨论，也欢迎你把今天的内容分享给你的朋友。</p>
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
  <div class="_2_QraFYR_0">你好，我是胡夕。我来公布上节课的“课后讨论”题答案啦～<br><br>上节课，咱们重点了解了Controller选举部分的代码。课后我请你思考这样一个问题：源码中当Controller选举之后哪里更新的元数据请求？ 其实这是在onControllerFailover方法中完成的。该方法中调用：<br>sendUpdateMetadataRequest(controllerContext.liveOrShuttingDownBrokerIds.toSeq, Set.empty)令集群所有Broker更新元数据<br><br>okay，你同意这个说法吗？或者说你有其他的看法吗？我们可以一起讨论下。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-29 10:28:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTIZcwnxGkjzENwyeEd3RZsh9tpZDeYmnT51iciaMiaLV2XRfzrJolZvUWjf3L5DuE3BmBg7uCKg3iaSzQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>RonnieXie</span>
  </div>
  <div class="_2_QraFYR_0">老师，学习了关于Controller源码课，我再深入看相关源码，梳理了流程图，您看这流程图是否正确<br>http:&#47;&#47;note.youdao.com&#47;noteshare?id=1f999786b77ada75fbb58b2831770e47&amp;sub=554F637E4F984C7B8036365C51E0758B</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 很棒的总结啊！我自己也学到了一些：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-27 07:51:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTI34ZlT6HSOtJBeTvTvfNLfYECDdJXnHCMj2BHdrRaqRLnZiafnxmKQ2aXoQkW1RLQOyt0tlyzEWIA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ahu0605</span>
  </div>
  <div class="_2_QraFYR_0">这节课应该往前提，提到这个主题的前面，讲的太好了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-18 09:54:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/f7/8a/09a4c107.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>贝</span>
  </div>
  <div class="_2_QraFYR_0">胡夕老师，最近遇到个问题，向请教一下您：<br>我下线了controller节点1后，2被选为新的controller。但是发现有个topic A的partition1元数据信息不对，原先它的三个副本在2，3，4（2为leader）。但是看打印的日志中发现新controller 2 告诉6节点说topic A的partition1的副本是3，2，6，但并没有向2，3，4同步元数据的变更，导致6节点无故新增了一副本并向leader拉取一直失败，以下是日志：您帮我看看这是为什么呢？感觉像是controller的下线导致元数据错乱了<br><br>6机器上的日志：<br>[2021-03-16 14:12:39.979] TRACE [Broker id=6] Cached leader info UpdateMetadataPartitionState(topicName=&#39;A&#39;, p<br>artitionIndex=1, controllerEpoch=29, leader=2, leaderEpoch=13, isr=[2, 3, 4], zkVersion=25, replicas=[2, 3, 4], offlineReplicas=[<br>]) for partition A-1 in response to UpdateMetadata request sent by controller 1 epoch 29 with correlation id 0 (state.change.logger)<br><br>[2021-03-16 15:00:46.600] TRACE [Broker id=6] Cached leader info UpdateMetadataPartitionState(topicName=&#39;A&#39;, p<br>artitionIndex=1, controllerEpoch=29, leader=2, leaderEpoch=13, isr=[2, 3, 4], zkVersion=25, replicas=[3, 2, 6], offlineReplicas=[<br>]) for partition A-1 in response to UpdateMetadata request sent by controller 2 epoch 30 with correlation id 0 (state.change.logger)<br><br>2机器上的日志：<br>[2021-03-16 15:07:26,164] WARN [ReplicaManager broker=2] Leader 2 failed to record follower 6&#39;s position 11679162 since the replica is not recognized to be one of the assigned replicas 3,4,2 for partition A-1. Empty records will be returned for this partition. (kafka.server.ReplicaManager)</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-18 15:22:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLSEtGul3OLHfbkbq5qOywnsCmZ68icDaFcvghXyvmAicTUtLr9q18dmJHH3ZWq3QLGcaicHo1ARn5gA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>雕</span>
  </div>
  <div class="_2_QraFYR_0">老师，在看 Controller 处理各种事件的时候，发现相关的处理方法中都没有重新注册watcher。比如新启动节点触发了 BrokerChange 事件，处理流程中并没有重新注册对&#47;brokers&#47;ids 子节点的监听。这个是在哪里进行重新注册的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 在KafkaController的onControllerFailover方法中会注册</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-22 11:21:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/76/53/21d62a23.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>鲁·本</span>
  </div>
  <div class="_2_QraFYR_0">增加一个主题的分区调用的是 KafkaController 的方法: processTopicChange -&gt; onNewPartitionCreation </div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-01 18:47:43</div>
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
  <div class="_2_QraFYR_0"> 调用的是onNewPartitionCreation方法，源码如下<br>private def onNewPartitionCreation(newPartitions: Set[TopicPartition]): Unit = {<br>    info(s&quot;New partition creation callback for ${newPartitions.mkString(&quot;,&quot;)}&quot;)<br>    partitionStateMachine.handleStateChanges(newPartitions.toSeq, NewPartition)<br>    replicaStateMachine.handleStateChanges(controllerContext.replicasForPartition(newPartitions).toSeq, NewReplica)<br>    partitionStateMachine.handleStateChanges(<br>      newPartitions.toSeq,<br>      OnlinePartition,<br>      Some(OfflinePartitionLeaderElectionStrategy(false))<br>    )<br>    replicaStateMachine.handleStateChanges(controllerContext.replicasForPartition(newPartitions).toSeq, OnlineReplica)<br>  }</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-25 14:14:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTIZcwnxGkjzENwyeEd3RZsh9tpZDeYmnT51iciaMiaLV2XRfzrJolZvUWjf3L5DuE3BmBg7uCKg3iaSzQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>RonnieXie</span>
  </div>
  <div class="_2_QraFYR_0">	胡夕老师，我们项目最近在预研使用图数据库，刚看碰巧在一篇文章（https:&#47;&#47;zhuanlan.zhihu.com&#47;p&#47;99381529）<br>	看到你们公司使用JanusGraph，想问下这数据库稳定性如何，同时也看到另外一款百度开源的HugeGraph，<br>	你们选型的JanusGraph看中的优势是什么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 比较稳定，API也算好用。没怎么用过百度的图数据库。另外我们公司用图数据库很长时间了，那时HugeGraph还没有开源。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-27 07:54:49</div>
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
  <div class="_2_QraFYR_0">调用脚本kafka-topics.sh<br>	exec $(dirname $0)&#47;kafka-run-class.sh kafka.admin.TopicCommand &quot;$@&quot;<br>对应的类object TopicCommand 的topicService.createTopic(opts)根据是否定义ZK信息走实现类ZookeeperTopicService或者AdminClientTopicService然后 ZookeeperTopicService里面<br>adminZkClient.createTopic 然后就可以触发topicChange事件...</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-24 22:07:56</div>
  </div>
</div>
</div>
</li>
</ul>