<audio title="16 _ TopicDeletionManager： Topic是怎么被删除的？" src="https://static001.geekbang.org/resource/audio/9f/22/9fdf4de216c6431cc1324361c5132222.mp3" controls="controls"></audio> 
<p>你好，我是胡夕。今天，我们正式进入到第四大模块“状态机”的学习。</p><p>Kafka源码中有很多状态机和管理器，比如之前我们学过的Controller通道管理器ControllerChannelManager、处理Controller事件的ControllerEventManager，等等。这些管理器和状态机，大多与各自的“宿主”组件关系密切，可以说是大小不同、功能各异。就比如Controller的这两个管理器，必须要与Controller组件紧耦合在一起才能实现各自的功能。</p><p>不过，Kafka中还是有一些状态机和管理器具有相对独立的功能框架，不严重依赖使用方，也就是我在这个模块为你精选的TopicDeletionManager（主题删除管理器）、ReplicaStateMachine（副本状态机）和PartitionStateMachine（分区状态机）。</p><ul>
<li>TopicDeletionManager：负责对指定Kafka主题执行删除操作，清除待删除主题在集群上的各类“痕迹”。</li>
<li>ReplicaStateMachine：负责定义Kafka副本状态、合法的状态转换，以及管理状态之间的转换。</li>
<li>PartitionStateMachine：负责定义Kafka分区状态、合法的状态转换，以及管理状态之间的转换。</li>
</ul><!-- [[[read_end]]] --><p>无论是主题、分区，还是副本，它们在Kafka中的生命周期通常都有多个状态。而这3个状态机，就是来管理这些状态的。而如何实现正确、高效的管理，就是源码要解决的核心问题。</p><p>今天，我们先来学习TopicDeletionManager，看一下Kafka是如何删除一个主题的。</p><h2>课前导读</h2><p>刚开始学习Kafka的时候，我对Kafka删除主题的认识非常“浅薄”。之前我以为成功执行了kafka-topics.sh --delete命令后，主题就会被删除。我相信，很多人可能都有过这样的错误理解。</p><p>这种不正确的认知产生的一个结果就是，我们经常发现主题没有被删除干净。于是，网上流传着一套终极“武林秘籍”：手动删除磁盘上的日志文件，以及手动删除ZooKeeper下关于主题的各个节点。</p><p>就我个人而言，我始终不推荐你使用这套“秘籍”，理由有二：</p><ul>
<li>它并不完整。事实上，除非你重启Broker，否则，这套“秘籍”无法清理Controller端和各个Broker上元数据缓存中的待删除主题的相关条目。</li>
<li>它并没有被官方所认证，换句话说就是后果自负。从某种程度上说，它会带来什么不好的结果，是你没法把控的。</li>
</ul><p>所谓“本事大不如不摊上”，我们与其琢磨删除主题失败之后怎么自救，不如踏踏实实地研究下Kafka底层是怎么执行这个操作的。搞明白它的原理之后，再有针对性地使用“秘籍”，才能做到有的放矢。你说是不是？</p><h2>TopicDeletionManager概览</h2><p>好了，我们就正式开始学习TopicDeletionManager吧。</p><p>这个管理器位于kafka.controller包下，文件名是TopicDeletionManager.scala。在总共不到400行的源码中，它定义了3个类结构以及20多个方法。总体而言，它还是比较容易学习的。</p><p>为了让你先有个感性认识，我画了一张TopicDeletionManager.scala的代码UML图：</p><p><img src="https://static001.geekbang.org/resource/image/52/f9/52032b5ccf820b10090b5a4ad78d8ff9.jpg?wh=3208*2957" alt=""></p><p>TopicDeletionManager.scala这个源文件，包括3个部分。</p><ul>
<li>DeletionClient接口：负责实现删除主题以及后续的动作，比如更新元数据等。这个接口里定义了4个方法，分别是deleteTopic、deleteTopicDeletions、mutePartitionModifications和sendMetadataUpdate。我们后面再详细学习它们的代码。</li>
<li>ControllerDeletionClient类：实现DeletionClient接口的类，分别实现了刚刚说到的那4个方法。</li>
<li>TopicDeletionManager类：主题删除管理器类，定义了若干个方法维护主题删除前后集群状态的正确性。比如，什么时候才能删除主题、什么时候主题不能被删除、主题删除过程中要规避哪些操作，等等。</li>
</ul><h2>DeletionClient接口及其实现</h2><p>接下来，我们逐一讨论下这3部分。首先是DeletionClient接口及其实现类。</p><p>就像前面说的，DeletionClient接口定义的方法用于删除主题，并将删除主题这件事儿同步给其他Broker。</p><p>目前，DeletionClient这个接口只有一个实现类，即ControllerDeletionClient。我们看下这个实现类的代码：</p><pre><code>class ControllerDeletionClient(controller: KafkaController, zkClient: KafkaZkClient) extends DeletionClient {
  // 删除给定主题
  override def deleteTopic(topic: String, epochZkVersion: Int): Unit = {
    // 删除/brokers/topics/&lt;topic&gt;节点
    zkClient.deleteTopicZNode(topic, epochZkVersion)
    // 删除/config/topics/&lt;topic&gt;节点
    zkClient.deleteTopicConfigs(Seq(topic), epochZkVersion)
    // 删除/admin/delete_topics/&lt;topic&gt;节点
    zkClient.deleteTopicDeletions(Seq(topic), epochZkVersion)
  }
  // 删除/admin/delete_topics下的给定topic子节点
  override def deleteTopicDeletions(topics: Seq[String], epochZkVersion: Int): Unit = {
    zkClient.deleteTopicDeletions(topics, epochZkVersion)
  }
  // 取消/brokers/topics/&lt;topic&gt;节点数据变更的监听
  override def mutePartitionModifications(topic: String): Unit = {
    controller.unregisterPartitionModificationsHandlers(Seq(topic))
  }
  // 向集群Broker发送指定分区的元数据更新请求
  override def sendMetadataUpdate(partitions: Set[TopicPartition]): Unit = {
    controller.sendUpdateMetadataRequest(
      controller.controllerContext.liveOrShuttingDownBrokerIds.toSeq, partitions)
  }
}
</code></pre><p>这个类的构造函数接收两个字段。同时，由于是DeletionClient接口的实现类，因而该类实现了DeletionClient接口定义的四个方法。</p><p>先来说构造函数的两个字段：KafkaController实例和KafkaZkClient实例。KafkaController实例，我们已经很熟悉了，就是Controller组件对象；而KafkaZkClient实例，就是Kafka与ZooKeeper交互的客户端对象。</p><p>接下来，我们再结合代码看下DeletionClient接口实现类ControllerDeletionClient定义的4个方法。我来简单介绍下这4个方法大致是做什么的。</p><p><strong>1.deleteTopic</strong></p><p>它用于删除主题在ZooKeeper上的所有“痕迹”。具体方法是，分别调用KafkaZkClient的3个方法去删除ZooKeeper下/brokers/topics/<topic>节点、/config/topics/<topic>节点和/admin/delete_topics/<topic>节点。</topic></topic></topic></p><p><strong>2.deleteTopicDeletions</strong></p><p>它用于删除ZooKeeper下待删除主题的标记节点。具体方法是，调用KafkaZkClient的deleteTopicDeletions方法，批量删除一组主题在/admin/delete_topics下的子节点。注意，deleteTopicDeletions这个方法名结尾的Deletions，表示/admin/delete_topics下的子节点。所以，deleteTopic是删除主题，deleteTopicDeletions是删除/admin/delete_topics下的对应子节点。</p><p>到这里，我们还要注意的一点是，这两个方法里都有一个epochZkVersion的字段，代表期望的Controller Epoch版本号。如果你使用一个旧的Epoch版本号执行这些方法，ZooKeeper会拒绝，因为和它自己保存的版本号不匹配。如果一个Controller的Epoch值小于ZooKeeper中保存的，那么这个Controller很可能是已经过期的Controller。这种Controller就被称为Zombie Controller。epochZkVersion字段的作用，就是隔离Zombie Controller发送的操作。</p><p><strong>3.mutePartitionModifications</strong></p><p>它的作用是屏蔽主题分区数据变更监听器，具体实现原理其实就是取消/brokers/topics/<topic>节点数据变更的监听。这样当该主题的分区数据发生变更后，由于对应的ZooKeeper监听器已经被取消了，因此不会触发Controller相应的处理逻辑。</topic></p><p>那为什么要取消这个监听器呢？其实，主要是为了避免操作之间的相互干扰。设想下，用户A发起了主题删除，而同时用户B为这个主题新增了分区。此时，这两个操作就会相互冲突，如果允许Controller同时处理这两个操作，势必会造成逻辑上的混乱以及状态的不一致。为了应对这种情况，在移除主题副本和分区对象前，代码要先执行这个方法，以确保不再响应用户对该主题的其他操作。</p><p>mutePartitionModifications方法的实现原理很简单，它会调用unregisterPartitionModificationsHandlers，并接着调用KafkaZkClient的unregisterZNodeChangeHandler方法，取消ZooKeeper上对给定主题的分区节点数据变更的监听。</p><p><strong>4.sendMetadataUpdate</strong></p><p>它会调用KafkaController的sendUpdateMetadataRequest方法，给集群所有Broker发送更新请求，告诉它们不要再为已删除主题的分区提供服务。代码如下：</p><pre><code>override def sendMetadataUpdate(partitions: Set[TopicPartition]): Unit = {
  // 给集群所有Broker发送UpdateMetadataRequest
  // 通知它们给定partitions的状态变化
  controller.sendUpdateMetadataRequest(
    controller.controllerContext.liveOrShuttingDownBrokerIds.toSeq, partitions)
}
</code></pre><p>该方法会给集群中的所有Broker发送更新元数据请求，告知它们要同步给定分区的状态。</p><h2>TopicDeletionManager定义及初始化</h2><p>有了这些铺垫，我们再来看主题删除管理器的主要入口：TopicDeletionManager类。这个类的定义代码，如下：</p><pre><code>class TopicDeletionManager(
  // KafkaConfig类，保存Broker端参数
  config: KafkaConfig, 
  // 集群元数据
  controllerContext: ControllerContext,
  // 副本状态机，用于设置副本状态
  replicaStateMachine: ReplicaStateMachine,
  // 分区状态机，用于设置分区状态
  partitionStateMachine: PartitionStateMachine,
  // DeletionClient接口，实现主题删除
  client: DeletionClient) extends Logging {
  this.logIdent = s&quot;[Topic Deletion Manager ${config.brokerId}] &quot;
  // 是否允许删除主题
  val isDeleteTopicEnabled: Boolean = config.deleteTopicEnable
  ......
}
</code></pre><p>该类主要的属性有6个，我们分别来看看。</p><ul>
<li>config：KafkaConfig实例，可以用作获取Broker端参数delete.topic.enable的值。该参数用于控制是否允许删除主题，默认值是true，即Kafka默认允许用户删除主题。</li>
<li>controllerContext：Controller端保存的元数据信息。删除主题必然要变更集群元数据信息，因此TopicDeletionManager需要用到controllerContext的方法，去更新它保存的数据。</li>
<li>replicaStateMachine和partitionStateMachine：副本状态机和分区状态机。它们各自负责副本和分区的状态转换，以保持副本对象和分区对象在集群上的一致性状态。这两个状态机是后面两讲的重要知识点。</li>
<li>client：前面介绍的DeletionClient接口。TopicDeletionManager通过该接口执行ZooKeeper上节点的相应更新。</li>
<li>isDeleteTopicEnabled：表明主题是否允许被删除。它是Broker端参数delete.topic.enable的值，默认是true，表示Kafka允许删除主题。源码中大量使用这个字段判断主题的可删除性。前面的config参数的主要目的就是设置这个字段的值。被设定之后，config就不再被源码使用了。</li>
</ul><p>好了，知道这些字段的含义，我们再看看TopicDeletionManager类实例是如何被创建的。</p><p>实际上，这个实例是在KafkaController类初始化时被创建的。在KafkaController类的源码中，你可以很容易地找到这行代码：</p><pre><code>val topicDeletionManager = new TopicDeletionManager(config, controllerContext, replicaStateMachine,
  partitionStateMachine, new ControllerDeletionClient(this, zkClient))
</code></pre><p>可以看到，代码实例化了一个全新的ControllerDeletionClient对象，然后利用这个对象实例和replicaStateMachine、partitionStateMachine，一起创建TopicDeletionManager实例。</p><p>为了方便你理解，我再给你画一张流程图：</p><p><img src="https://static001.geekbang.org/resource/image/89/e6/89733b9e03df6e1450ba81e082187ce6.jpg?wh=1449*2175" alt=""></p><h2>TopicDeletionManager重要方法</h2><p>除了类定义和初始化，TopicDeletionManager类还定义了16个方法。在这些方法中，最重要的当属resumeDeletions方法。它是重启主题删除操作过程的方法。</p><p>主题因为某些事件可能一时无法完成删除，比如主题分区正在进行副本重分配等。一旦这些事件完成后，主题重新具备可删除的资格。此时，代码就需要调用resumeDeletions重启删除操作。</p><p>这个方法之所以很重要，是因为它还串联了TopicDeletionManager类的很多方法，如completeDeleteTopic和onTopicDeletion等。因此，你完全可以从resumeDeletions方法开始，逐渐深入到其他方法代码的学习。</p><p>那我们就先学习resumeDeletions的实现代码吧。</p><pre><code>private def resumeDeletions(): Unit = {
  // 从元数据缓存中获取要删除的主题列表
  val topicsQueuedForDeletion = Set.empty[String] ++ controllerContext.topicsToBeDeleted
  // 待重试主题列表
  val topicsEligibleForRetry = mutable.Set.empty[String]
  // 待删除主题列表
  val topicsEligibleForDeletion = mutable.Set.empty[String]
  if (topicsQueuedForDeletion.nonEmpty)
    info(s&quot;Handling deletion for topics ${topicsQueuedForDeletion.mkString(&quot;,&quot;)}&quot;)
  // 遍历每个待删除主题
  topicsQueuedForDeletion.foreach { topic =&gt;
    // 如果该主题所有副本已经是ReplicaDeletionSuccessful状态
    // 即该主题已经被删除  
    if (controllerContext.areAllReplicasInState(topic, ReplicaDeletionSuccessful)) {
      // 调用completeDeleteTopic方法完成后续操作即可
      completeDeleteTopic(topic)
      info(s&quot;Deletion of topic $topic successfully completed&quot;)
     // 如果主题删除尚未开始并且主题当前无法执行删除的话
    } else if (!controllerContext.isAnyReplicaInState(topic, ReplicaDeletionStarted)) {
      if (controllerContext.isAnyReplicaInState(topic, ReplicaDeletionIneligible)) {
        // 把该主题加到待重试主题列表中用于后续重试
        topicsEligibleForRetry += topic
      }
    }
    // 如果该主题能够被删除
    if (isTopicEligibleForDeletion(topic)) {
      info(s&quot;Deletion of topic $topic (re)started&quot;)
      topicsEligibleForDeletion += topic
    }
  }
  // 重试待重试主题列表中的主题删除操作
  if (topicsEligibleForRetry.nonEmpty) {
    retryDeletionForIneligibleReplicas(topicsEligibleForRetry)
  }
  // 调用onTopicDeletion方法，对待删除主题列表中的主题执行删除操作
  if (topicsEligibleForDeletion.nonEmpty) {
    onTopicDeletion(topicsEligibleForDeletion)
  }
}
</code></pre><p>通过代码我们发现，这个方法<strong>首先</strong>从元数据缓存中获取要删除的主题列表，之后定义了两个空的主题列表，分别保存待重试删除主题和待删除主题。</p><p><strong>然后</strong>，代码遍历每个要删除的主题，去看它所有副本的状态。如果副本状态都是ReplicaDeletionSuccessful，就表明该主题已经被成功删除，此时，再调用completeDeleteTopic方法，完成后续的操作就可以了。对于那些删除操作尚未开始，并且暂时无法执行删除的主题，源码会把这类主题加到待重试主题列表中，用于后续重试；如果主题是能够被删除的，就将其加入到待删除列表中。</p><p><strong>最后</strong>，该方法调用retryDeletionForIneligibleReplicas方法，来重试待重试主题列表中的主题删除操作。对于待删除主题列表中的主题则调用onTopicDeletion删除之。</p><p>值得一提的是，retryDeletionForIneligibleReplicas方法用于重试主题删除。这是通过将对应主题副本的状态，从ReplicaDeletionIneligible变更到OfflineReplica来完成的。这样，后续再次调用resumeDeletions时，会尝试重新删除主题。</p><p>看到这里，我们再次发现，Kafka的方法命名真的是非常规范。得益于这一点，很多时候，我们不用深入到方法内部，就能知道这个方法大致是做什么用的。比如：</p><ul>
<li>topicsQueuedForDeletion方法，应该是保存待删除的主题列表；</li>
<li>controllerContext.isAnyReplicaInState方法，应该是判断某个主题下是否有副本处于某种状态；</li>
<li>而onTopicDeletion方法，应该就是执行主题删除操作用的。</li>
</ul><p>这时，你再去阅读这3个方法的源码，就会发现它们的作用确实如其名字标识的那样。这也再次证明了Kafka源码质量是非常不错的。因此，不管你是不是Kafka的使用者，都可以把Kafka的源码作为阅读开源框架源码、提升自己竞争力的一个选择。</p><p>下面，我再用一张图来解释下resumeDeletions方法的执行流程：</p><p><img src="https://static001.geekbang.org/resource/image/d1/34/d165193d4f27ffe18e038643ea050534.jpg?wh=2504*3969" alt=""></p><p>到这里，resumeDeletions方法的逻辑我就讲完了，它果然是串联起了TopicDeletionManger中定义的很多方法。其中，比较关键的两个操作是<strong>completeDeleteTopic</strong>和<strong>onTopicDeletion</strong>。接下来，我们就分别看看。</p><p>先来看completeDeleteTopic方法代码，我给每行代码都加标了注解。</p><pre><code>private def completeDeleteTopic(topic: String): Unit = {
  // 第1步：注销分区变更监听器，防止删除过程中因分区数据变更
  // 导致监听器被触发，引起状态不一致
  client.mutePartitionModifications(topic)
  // 第2步：获取该主题下处于ReplicaDeletionSuccessful状态的所有副本对象，
  // 即所有已经被成功删除的副本对象
  val replicasForDeletedTopic = controllerContext.replicasInState(topic, ReplicaDeletionSuccessful)
  // 第3步：利用副本状态机将这些副本对象转换成NonExistentReplica状态。
  // 等同于在状态机中删除这些副本
  replicaStateMachine.handleStateChanges(
    replicasForDeletedTopic.toSeq, NonExistentReplica)
  // 第4步：更新元数据缓存中的待删除主题列表和已开始删除的主题列表
  // 因为主题已经成功删除了，没有必要出现在这两个列表中了
  controllerContext.topicsToBeDeleted -= topic
  controllerContext.topicsWithDeletionStarted -= topic
  // 第5步：移除ZooKeeper上关于该主题的一切“痕迹”
  client.deleteTopic(topic, controllerContext.epochZkVersion)
  // 第6步：移除元数据缓存中关于该主题的一切“痕迹”
  controllerContext.removeTopic(topic)
}
</code></pre><p>整个过程如行云流水般一气呵成，非常容易理解，我就不多解释了。</p><p>再来看看onTopicDeletion方法的代码：</p><pre><code>private def onTopicDeletion(topics: Set[String]): Unit = {
  // 找出给定待删除主题列表中那些尚未开启删除操作的所有主题
  val unseenTopicsForDeletion = topics.diff(controllerContext.topicsWithDeletionStarted)
  if (unseenTopicsForDeletion.nonEmpty) {
    // 获取到这些主题的所有分区对象
    val unseenPartitionsForDeletion = unseenTopicsForDeletion.flatMap(controllerContext.partitionsForTopic)
    // 将这些分区的状态依次调整成OfflinePartition和NonExistentPartition
    // 等同于将这些分区从分区状态机中删除
    partitionStateMachine.handleStateChanges(
      unseenPartitionsForDeletion.toSeq, OfflinePartition)
    partitionStateMachine.handleStateChanges(
      unseenPartitionsForDeletion.toSeq, NonExistentPartition)
    // 把这些主题加到“已开启删除操作”主题列表中
    controllerContext.beginTopicDeletion(unseenTopicsForDeletion)
  }
  // 给集群所有Broker发送元数据更新请求，告诉它们不要再为这些主题处理数据了
  client.sendMetadataUpdate(
    topics.flatMap(controllerContext.partitionsForTopic))
  // 分区删除操作会执行底层的物理磁盘文件删除动作
  onPartitionDeletion(topics)
}
</code></pre><p>我在代码中用注释的方式，已经把onTopicDeletion方法的逻辑解释清楚了。你可以发现，这个方法也基本是串行化的流程，没什么难理解的。我再给你梳理下其中的核心点。</p><p>onTopicDeletion方法会多次使用分区状态机，来调整待删除主题的分区状态。在后两讲的分区状态机和副本状态机的课里面，我还会和你详细讲解它们，包括它们都定义了哪些状态，这些状态彼此之间的转换规则都是什么，等等。</p><p>onTopicDeletion方法的最后一行调用了onPartitionDeletion方法，来执行真正的底层物理磁盘文件删除。实际上，这是通过副本状态机状态转换操作完成的。下节课，我会和你详细聊聊这个事情。</p><p>学到这里，我还想提醒你的是，在学习TopicDeletionManager方法的时候，非常重要一点的是，你要理解主题删除的脉络。对于其他部分的源码，也是这个道理。一旦你掌握了整体的流程，阅读那些细枝末节的方法代码就没有任何难度了。照着这个方法去读源码，搞懂Kafka源码也不是什么难事！</p><h2>总结</h2><p>今天，我们主要学习了TopicDeletionManager.scala中关于主题删除的代码。这里有几个要点，需要你记住。</p><ul>
<li>在主题删除过程中，Kafka会调整集群中三个地方的数据：ZooKeeper、元数据缓存和磁盘日志文件。删除主题时，ZooKeeper上与该主题相关的所有ZNode节点必须被清除；Controller端元数据缓存中的相关项，也必须要被处理，并且要被同步到集群的其他Broker上；而磁盘日志文件，更是要清理的首要目标。这三个地方必须要统一处理，就好似我们常说的原子性操作一样。现在回想下开篇提到的那个“秘籍”，你就会发现它缺少了非常重要的一环，那就是：<strong>无法清除Controller端的元数据缓存项</strong>。因此，你要尽力避免使用这个“大招”。</li>
<li>DeletionClient接口的作用，主要是操作ZooKeeper，实现ZooKeeper节点的删除等操作。</li>
<li>TopicDeletionManager，是在KafkaController创建过程中被初始化的，主要通过与元数据缓存进行交互的方式，来更新各类数据。</li>
</ul><p><img src="https://static001.geekbang.org/resource/image/3a/3c/3a47958f44b81cef7b502da53495ef3c.jpg?wh=2250*1332" alt=""></p><p>今天我们看到的代码中出现了大量的replicaStateMachine和partitionStateMachine，其实这就是我今天反复提到的副本状态机和分区状态机。接下来两讲，我会逐步带你走进它们的代码世界，让你领略下Kafka通过状态机机制管理副本和分区的风采。</p><h2>课后讨论</h2><p>上节课，我们在学习processTopicDeletion方法的代码时，看到了一个名为markTopicIneligibleForDeletion的方法，这也是和主题删除相关的方法。现在，你能说说，它是做什么用的、它的实现原理又是怎样的吗？</p><p>欢迎你在留言区畅所欲言，跟我交流讨论，也欢迎你把今天的内容分享给你的朋友。</p>
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
  <div class="_2_QraFYR_0">你好，我是胡夕。我来公布上节课的“课后讨论”题答案啦～<br><br>上节课，咱们结合源码重点了解了Controller在集群中的作用。课后我请你思考这样一个问题：如果我们想要使用脚本命令增加一个主题的分区，你知道应该用KafkaController类中的哪个方法吗？其实，KafkaController的onNewPartitionCreation方法是处理新分区增加逻辑的。它包括要将新分区状态调整到Online状态以及将对应副本的状态设置成Online。<br><br>okay，你同意这个说法吗？或者说你有其他的看法吗？我们可以一起讨论下。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-29 10:35:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/a1/46/3136ac25.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>是男人就开巴巴托斯</span>
  </div>
  <div class="_2_QraFYR_0">mutePartitionModifications是为了不要让watcher响应zk topic各个分区子节点逐个被删除的变化。<br>删除是从分区节点到topic节点递归执行的，因为zk不能在有子节点的情况下直接删除父节点。<br><br>新加分区与删除topic节点的互斥还是在handleCreatePartitionsRequest函数里做的check。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-11 10:40:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/66/d1/8664c464.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>flyCoder</span>
  </div>
  <div class="_2_QraFYR_0">胡老师，遇到了这个问题，能帮忙分析下吗？https:&#47;&#47;www.orchome.com&#47;1085</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 打开文件数太多这件事社区曾经有几个bug跟过，在新一点的版本（比如2.0之后）之后已经不太常见了，不妨升级下看看。另外ulimit -n 不妨设置大一点。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-16 11:11:46</div>
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
  <div class="_2_QraFYR_0">def enqueueTopicsForDeletion(topics: Set[String]): Unit = {<br>    if (isDeleteTopicEnabled) {<br>      controllerContext.queueTopicDeletion(topics)<br>      resumeDeletions()<br>    }<br>  }刚去研究了一下enqueueTopicsForDeletion方法源码<br>val isDeleteTopicEnabled: Boolean = config.deleteTopicEnable<br><br> def queueTopicDeletion(topics: Set[String]): Unit = {<br>    topicsToBeDeleted ++= topics<br>}<br>可以删除的topic会添加进删除队列，不可删除的会重新调用resumeDeletions方法<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-25 16:08:49</div>
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
  <div class="_2_QraFYR_0">if (config.deleteTopicEnable) {<br>      if (topicsToBeDeleted.nonEmpty) {<br>        info(s&quot;Starting topic deletion for topics ${topicsToBeDeleted.mkString(&quot;,&quot;)}&quot;)<br>        &#47;&#47; mark topic ineligible for deletion if other state changes are in progress<br>        topicsToBeDeleted.foreach { topic =&gt;<br>          val partitionReassignmentInProgress =<br>            controllerContext.partitionsBeingReassigned.map(_.topic).contains(topic)<br>          if (partitionReassignmentInProgress)<br>            topicDeletionManager.markTopicIneligibleForDeletion(Set(topic),<br>              reason = &quot;topic reassignment in progress&quot;)<br>        }<br>        &#47;&#47; add topic to deletion list<br>        topicDeletionManager.enqueueTopicsForDeletion(topicsToBeDeleted)<br>      }  <br>①根据配置信息是否进行主题删除 是则进行下一步<br>②判断要删除的主题是否为空，不为空则进行下一步<br>③进行迭代判断，主题是否在partitionsBeingReassigned集合中，如果在，则将主题添加进markTopicIneligibleForDeletion 标志为主题正在重新分配 不可删除<br><br>有一点疑问是为什么没有从topicsToBeDeleted集合中把不可删除主题移除的步骤，然后就直接将topicsToBeDeleted添加进删除队列了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-25 15:52:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/54/b5/310dae7b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小刀</span>
  </div>
  <div class="_2_QraFYR_0">stream DSL中除了通过缩短时间窗口可以减小state.dir（&#47;tmp&#47;kafka-streams）的大小之外还有别的方式么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 没有太好的方法</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-04 15:35:52</div>
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
  <div class="_2_QraFYR_0">controllerContext 中的 topicsToBeDeleted （待删除主题列表）和 zk中的 &#47;delete_topics列表 进行 &amp;得到暂时不可删除的topic。<br>暂时不可删除的原因：<br>1、topic存在的部分副本down了<br>2、当前topic的部分分区正在reassignment<br>我理解为是controllerContext缓存中的数据，可能会和当前的状态不一致的情况</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: “controllerContext缓存中的数据，可能会和当前的状态不一致的情况”  比如？能否举个例子？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-27 23:36:49</div>
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
  <div class="_2_QraFYR_0">不符合删除的条件 在这里实现，副本关闭停止删除，正在重新分区（扩容到新的broker）的不可删除。主要做逻辑与“&amp;”操作得到不符合删除的topic。不符合删除的条件 在这里实现，副本关闭停止删除，正在重新分区（扩容到新的broker）的不可删除。主要做逻辑与“&amp;”操作得到不符合删除的topic。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-30 07:58:58</div>
  </div>
</div>
</div>
</li>
</ul>