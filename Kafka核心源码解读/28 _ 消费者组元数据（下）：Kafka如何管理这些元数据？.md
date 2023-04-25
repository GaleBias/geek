<audio title="28 _ 消费者组元数据（下）：Kafka如何管理这些元数据？" src="https://static001.geekbang.org/resource/audio/12/84/129fd863d8d850f4ab5b6a0603e02784.mp3" controls="controls"></audio> 
<p>你好，我是胡夕。今天我们继续学习消费者组元数据。</p><p>学完上节课之后，我们知道，Kafka定义了非常多的元数据，那么，这就必然涉及到对元数据的管理问题了。</p><p>这些元数据的类型不同，管理策略也就不一样。这节课，我将从消费者组状态、成员、位移和分区分配策略四个维度，对这些元数据进行拆解，带你一起了解下Kafka管理这些元数据的方法。</p><p>这些方法定义在MemberMetadata和GroupMetadata这两个类中，其中，GroupMetadata类中的方法最为重要，是我们要重点学习的对象。在后面的课程中，你会看到，这些方法会被上层组件GroupCoordinator频繁调用，因此，它们是我们学习Coordinator组件代码的前提条件，你一定要多花些精力搞懂它们。</p><h2>消费者组状态管理方法</h2><p>消费者组状态是很重要的一类元数据。管理状态的方法，要做的事情也就是设置和查询。这些方法大多比较简单，所以我把它们汇总在一起，直接介绍给你。</p><pre><code>// GroupMetadata.scala
// 设置/更新状态
def transitionTo(groupState: GroupState): Unit = {
  assertValidTransition(groupState) // 确保是合法的状态转换
  state = groupState  // 设置状态到给定状态
  currentStateTimestamp = Some(time.milliseconds() // 更新状态变更时间戳
// 查询状态
def currentState = state
// 判断消费者组状态是指定状态
def is(groupState: GroupState) = state == groupState
// 判断消费者组状态不是指定状态
def not(groupState: GroupState) = state != groupState
// 消费者组能否Rebalance的条件是当前状态是PreparingRebalance状态的合法前置状态
def canRebalance = PreparingRebalance.validPreviousStates.contains(state)
</code></pre><p><strong>1.transitionTo方法</strong></p><p>transitionTo方法的作用是<strong>将消费者组状态变更成给定状态</strong>。在变更前，代码需要确保这次变更必须是合法的状态转换。这是依靠每个GroupState实现类定义的<strong>validPreviousStates集合</strong>来完成的。只有在这个集合中的状态，才是合法的前置状态。简单来说，只有集合中的这些状态，才能转换到当前状态。</p><!-- [[[read_end]]] --><p>同时，该方法还会<strong>更新状态变更的时间戳字段</strong>。Kafka有个定时任务，会定期清除过期的消费者组位移数据，它就是依靠这个时间戳字段，来判断过期与否的。</p><p><strong>2.canRebalance方法</strong></p><p>它用于判断消费者组是否能够开启Rebalance操作。判断依据是，<strong>当前状态是否是PreparingRebalance状态的合法前置状态</strong>。只有<strong>Stable</strong>、<strong>CompletingRebalance</strong>和<strong>Empty</strong>这3类状态的消费者组，才有资格开启Rebalance。</p><p><strong>3.is和not方法</strong></p><p>至于is和not方法，它们分别判断消费者组的状态与给定状态吻合还是不吻合，主要被用于<strong>执行状态校验</strong>。特别是is方法，被大量用于上层调用代码中，执行各类消费者组管理任务的前置状态校验工作。</p><p>总体来说，管理消费者组状态数据，依靠的就是这些方法，还是很简单的吧？</p><h2>成员管理方法</h2><p>在介绍管理消费者组成员的方法之前，我先帮你回忆下GroupMetadata中保存成员的字段。GroupMetadata中使用members字段保存所有的成员信息。该字段是一个HashMap，Key是成员的member ID字段，Value是MemberMetadata类型，该类型保存了成员的元数据信息。</p><p>所谓的管理成员，也就是添加成员（add方法）、移除成员（remove方法）和查询成员（has、get、size方法等）。接下来，我们就逐一来学习。</p><h3>添加成员</h3><p>先说添加成员的方法：add。add方法的主要逻辑，是将成员对象添加到members字段，同时更新其他一些必要的元数据，比如Leader成员字段、分区分配策略支持票数等。下面是add方法的源码：</p><pre><code>def add(member: MemberMetadata, callback: JoinCallback = null): Unit = {
  // 如果是要添加的第一个消费者组成员
  if (members.isEmpty)
    // 就把该成员的procotolType设置为消费者组的protocolType
    this.protocolType = Some(member.protocolType)
  // 确保成员元数据中的groupId和组Id相同
  assert(groupId == member.groupId)
  // 确保成员元数据中的protoclType和组protocolType相同
  assert(this.protocolType.orNull == member.protocolType)
  // 确保该成员选定的分区分配策略与组选定的分区分配策略相匹配
  assert(supportsProtocols(member.protocolType, MemberMetadata.plainProtocolSet(member.supportedProtocols)))
  // 如果尚未选出Leader成员
  if (leaderId.isEmpty)
    // 把该成员设定为Leader成员
    leaderId = Some(member.memberId)
  // 将该成员添加进members
  members.put(member.memberId, member)
  // 更新分区分配策略支持票数
  member.supportedProtocols.foreach{ case (protocol, _) =&gt; supportedProtocols(protocol) += 1 }
  // 设置成员加入组后的回调逻辑
  member.awaitingJoinCallback = callback
  // 更新已加入组的成员数
  if (member.isAwaitingJoin)
    numMembersAwaitingJoin += 1
}
</code></pre><p>我再画一张流程图，帮助你更直观地理解这个方法的作用。</p><p><img src="https://static001.geekbang.org/resource/image/67/d5/679b34908b9d3cf7e10905fcf96e69d5.jpg?wh=1312*4544" alt=""></p><p>我再具体解释一下这个方法的执行逻辑。</p><p>第一步，add方法要判断members字段是否包含已有成员。如果没有，就说明要添加的成员是该消费者组的第一个成员，那么，就令该成员协议类型（protocolType）成为组的protocolType。我在上节课中讲过，对于普通的消费者而言，protocolType就是字符串"consumer"。如果不是首个成员，就进入到下一步。</p><p>第二步，add方法会连续进行三次校验，分别确保<strong>待添加成员的组ID、protocolType</strong>和组配置一致，以及该成员选定的分区分配策略与组选定的分区分配策略相匹配。如果这些校验有任何一个未通过，就会立即抛出异常。</p><p>第三步，判断消费者组的Leader成员是否已经选出了。如果还没有选出，就将该成员设置成Leader成员。当然了，如果Leader已经选出了，自然就不需要做这一步了。需要注意的是，这里的Leader和我们在学习副本管理器时学到的Leader副本是不同的概念。这里的Leader成员，是指<strong>消费者组下的一个成员</strong>。该成员负责为所有成员制定分区分配方案，制定方法的依据，就是消费者组选定的分区分配策略。</p><p>第四步，更新消费者组分区分配策略支持票数。关于supportedProtocols字段的含义，我在上节课的末尾用一个例子形象地进行了说明，这里就不再重复说了。如果你没有印象了，可以再复习一下。</p><p>最后一步，设置成员加入组后的回调逻辑，同时更新已加入组的成员数。至此，方法结束。</p><p>作为关键的成员管理方法之一，add方法是实现消费者组Rebalance流程至关重要的一环。每当Rebalance开启第一大步——加入组的操作时，本质上就是在利用这个add方法实现新成员入组的逻辑。</p><h3>移除成员</h3><p>有add方法，自然也就有remove方法，下面是remove方法的完整源码：</p><pre><code>def remove(memberId: String): Unit = {
  // 从members中移除给定成员
  members.remove(memberId).foreach { member =&gt;
    // 更新分区分配策略支持票数
    member.supportedProtocols.foreach{ case (protocol, _) =&gt; supportedProtocols(protocol) -= 1 }
    // 更新已加入组的成员数
    if (member.isAwaitingJoin)
      numMembersAwaitingJoin -= 1
  }
  // 如果该成员是Leader，选择剩下成员列表中的第一个作为新的Leader成员
  if (isLeader(memberId))
    leaderId = members.keys.headOption
}
</code></pre><p>remove方法比add要简单一些。<strong>首先</strong>，代码从members中移除给定成员。<strong>之后</strong>，更新分区分配策略支持票数，以及更新已加入组的成员数。<strong>最后</strong>，代码判断该成员是否是Leader成员，如果是的话，就选择成员列表中尚存的第一个成员作为新的Leader成员。</p><h3>查询成员</h3><p>查询members的方法有很多，大多都是很简单的场景。我给你介绍3个比较常见的。</p><pre><code>def has(memberId: String) = members.contains(memberId)
def get(memberId: String) = members(memberId)
def size = members.size
</code></pre><ul>
<li>has方法，判断消费者组是否包含指定成员；</li>
<li>get方法，获取指定成员对象；</li>
<li>size方法，统计总成员数。</li>
</ul><p>其它的查询方法逻辑也都很简单，比如allMemberMetadata、rebalanceTimeoutMs，等等，我就不多讲了。课后你可以自行阅读下，重点是体会这些方法利用members都做了什么事情。</p><h2>位移管理方法</h2><p>除了组状态和成员管理之外，GroupMetadata还有一大类管理功能，就是<strong>管理消费者组的提交位移</strong>（Committed Offsets），主要包括添加和移除位移值。</p><p>不过，在学习管理位移的功能之前，我再带你回顾一下保存位移的offsets字段的定义。毕竟，接下来我们要学习的方法，主要操作的就是这个字段。</p><pre><code>private val offsets = new mutable.HashMap[TopicPartition, CommitRecordMetadataAndOffset]
</code></pre><p>它是HashMap类型，Key是TopicPartition类型，表示一个主题分区，而Value是CommitRecordMetadataAndOffset类型。该类封装了位移提交消息的位移值。</p><p>在详细阅读位移管理方法之前，我先解释下这里的“位移”和“位移提交消息”。</p><p>消费者组需要向Coordinator提交已消费消息的进度，在Kafka中，这个进度有个专门的术语，叫作提交位移。Kafka使用它来定位消费者组要消费的下一条消息。那么，提交位移在Coordinator端是如何保存的呢？它实际上是保存在内部位移主题中。提交的方式是，消费者组成员向内部主题写入符合特定格式的事件消息，这类消息就是所谓的位移提交消息（Commit Record）。关于位移提交消息的事件格式，我会在第30讲具体介绍，这里你可以暂时不用理会。而这里所说的CommitRecordMetadataAndOffset类，就是标识位移提交消息的地方。我们看下它的代码：</p><pre><code>case class CommitRecordMetadataAndOffset(appendedBatchOffset: Option[Long], offsetAndMetadata: OffsetAndMetadata) {
  def olderThan(that: CommitRecordMetadataAndOffset): Boolean = appendedBatchOffset.get &lt; that.appendedBatchOffset.get
}
</code></pre><p>这个类的构造函数有两个参数。</p><ul>
<li>appendedBatchOffset：保存的是位移主题消息自己的位移值；</li>
<li>offsetAndMetadata：保存的是位移提交消息中保存的消费者组的位移值。</li>
</ul><h3>添加位移值</h3><p>在GroupMetadata中，有3个向offsets中添加订阅分区的已消费位移值的方法，分别是initializeOffsets、onOffsetCommitAppend和completePendingTxnOffsetCommit。</p><p>initializeOffsets方法的代码非常简单，如下所示：</p><pre><code>def initializeOffsets(
  offsets: collection.Map[TopicPartition, CommitRecordMetadataAndOffset],
  pendingTxnOffsets: Map[Long, mutable.Map[TopicPartition, CommitRecordMetadataAndOffset]]): Unit = {
  this.offsets ++= offsets
  this.pendingTransactionalOffsetCommits ++= pendingTxnOffsets
}
</code></pre><p>它仅仅是将给定的一组订阅分区提交位移值加到offsets中。当然，同时它还会更新pendingTransactionalOffsetCommits字段。</p><p>不过，由于这个字段是给Kafka事务机制使用的，因此，你只需要关注这个方法的第一行语句就行了。当消费者组的协调者组件启动时，它会创建一个异步任务，定期地读取位移主题中相应消费者组的提交位移数据，并把它们加载到offsets字段中。</p><p>我们再来看第二个方法，onOffsetCommitAppend的代码。</p><pre><code>def onOffsetCommitAppend(topicPartition: TopicPartition, offsetWithCommitRecordMetadata: CommitRecordMetadataAndOffset): Unit = {
  if (pendingOffsetCommits.contains(topicPartition)) {
    if (offsetWithCommitRecordMetadata.appendedBatchOffset.isEmpty)
      throw new IllegalStateException(&quot;Cannot complete offset commit write without providing the metadata of the record &quot; +
        &quot;in the log.&quot;)
    // offsets字段中没有该分区位移提交数据，或者
    // offsets字段中该分区对应的提交位移消息在位移主题中的位移值小于待写入的位移值
    if (!offsets.contains(topicPartition) || offsets(topicPartition).olderThan(offsetWithCommitRecordMetadata))
      // 将该分区对应的提交位移消息添加到offsets中
      offsets.put(topicPartition, offsetWithCommitRecordMetadata)
  }
  pendingOffsetCommits.get(topicPartition) match {
    case Some(stagedOffset) if offsetWithCommitRecordMetadata.offsetAndMetadata == stagedOffset =&gt;
      pendingOffsetCommits.remove(topicPartition)
    case _ =&gt;
  }
}
</code></pre><p>该方法在提交位移消息被成功写入后调用。主要判断的依据，是offsets中是否已包含该主题分区对应的消息值，或者说，offsets字段中该分区对应的提交位移消息在位移主题中的位移值是否小于待写入的位移值。如果是的话，就把该主题已提交的位移值添加到offsets中。</p><p>第三个方法completePendingTxnOffsetCommit的作用是完成一个待决事务（Pending Transaction）的位移提交。所谓的待决事务，就是指正在进行中、还没有完成的事务。在处理待决事务的过程中，可能会出现将待决事务中涉及到的分区的位移值添加到offsets中的情况。不过，由于该方法是与Kafka事务息息相关的，你不需要重点掌握，这里我就不展开说了。</p><h3>移除位移值</h3><p>offsets中订阅分区的已消费位移值也是能够被移除的。你还记得，Kafka主题中的消息有默认的留存时间设置吗？位移主题是普通的Kafka主题，所以也要遵守相应的规定。如果当前时间与已提交位移消息时间戳的差值，超过了Broker端参数offsets.retention.minutes值，Kafka就会将这条记录从offsets字段中移除。这就是方法removeExpiredOffsets要做的事情。</p><p>这个方法的代码有点长，为了方便你掌握，我分块给你介绍下。我先带你了解下它的内部嵌套类方法getExpireOffsets，然后再深入了解它的实现逻辑，这样你就能很轻松地掌握Kafka移除位移值的代码原理了。</p><p>首先，该方法定义了一个内部嵌套方法<strong>getExpiredOffsets</strong>，专门用于获取订阅分区过期的位移值。我们来阅读下源码，看看它是如何做到的。</p><pre><code>def getExpiredOffsets(
  baseTimestamp: CommitRecordMetadataAndOffset =&gt; Long,
  subscribedTopics: Set[String] = Set.empty): Map[TopicPartition, OffsetAndMetadata] = {
  // 遍历offsets中的所有分区，过滤出同时满足以下3个条件的所有分区
  // 条件1：分区所属主题不在订阅主题列表之内
  // 条件2：该主题分区已经完成位移提交
  // 条件3：该主题分区在位移主题中对应消息的存在时间超过了阈值
  offsets.filter {
    case (topicPartition, commitRecordMetadataAndOffset) =&gt;
      !subscribedTopics.contains(topicPartition.topic()) &amp;&amp;
      !pendingOffsetCommits.contains(topicPartition) &amp;&amp; {
        commitRecordMetadataAndOffset
          .offsetAndMetadata.expireTimestamp match {
          case None =&gt;
            currentTimestamp - baseTimestamp(commitRecordMetadataAndOffset) &gt;= offsetRetentionMs
          case Some(expireTimestamp) =&gt;
            currentTimestamp &gt;= expireTimestamp
        }
      }
  }.map {
    // 为满足以上3个条件的分区提取出commitRecordMetadataAndOffset中的位移值
    case (topicPartition, commitRecordOffsetAndMetadata) =&gt;
      (topicPartition, commitRecordOffsetAndMetadata.offsetAndMetadata)
  }.toMap
}
</code></pre><p>该方法接收两个参数。</p><ul>
<li>baseTimestamp：它是一个函数类型，接收CommitRecordMetadataAndOffset类型的字段，然后计算时间戳，并返回；</li>
<li>subscribedTopics：即订阅主题集合，默认是空。</li>
</ul><p>方法开始时，代码从offsets字段中过滤出同时满足3个条件的所有分区。</p><p><strong>条件1</strong>：分区所属主题不在订阅主题列表之内。当方法传入了不为空的主题集合时，就说明该消费者组此时正在消费中，正在消费的主题是不能执行过期位移移除的。</p><p><strong>条件2</strong>：主题分区已经完成位移提交，那种处于提交中状态，也就是保存在pendingOffsetCommits字段中的分区，不予考虑。</p><p><strong>条件3</strong>：该主题分区在位移主题中对应消息的存在时间超过了阈值。老版本的Kafka消息直接指定了过期时间戳，因此，只需要判断当前时间是否越过了这个过期时间。但是，目前，新版Kafka判断过期与否，主要是<strong>基于消费者组状态</strong>。如果是Empty状态，过期的判断依据就是当前时间与组变为Empty状态时间的差值，是否超过Broker端参数offsets.retention.minutes值；如果不是Empty状态，就看当前时间与提交位移消息中的时间戳差值是否超过了offsets.retention.minutes值。如果超过了，就视为已过期，对应的位移值需要被移除；如果没有超过，就不需要移除了。</p><p>当过滤出同时满足这3个条件的分区后，提取出它们对应的位移值对象并返回。</p><p>学过了getExpiredOffsets方法代码的实现之后，removeExpiredOffsets方法剩下的代码就很容易理解了。</p><pre><code>def removeExpiredOffsets(
  currentTimestamp: Long, offsetRetentionMs: Long): Map[TopicPartition, OffsetAndMetadata] = {
  // getExpiredOffsets方法代码......
  // 调用getExpiredOffsets方法获取主题分区的过期位移
  val expiredOffsets: Map[TopicPartition, OffsetAndMetadata] = protocolType match {
    case Some(_) if is(Empty) =&gt;
      getExpiredOffsets(
        commitRecordMetadataAndOffset =&gt; currentStateTimestamp    .getOrElse(commitRecordMetadataAndOffset.offsetAndMetadata.commitTimestamp)
      )
    case Some(ConsumerProtocol.PROTOCOL_TYPE) if subscribedTopics.isDefined =&gt;
      getExpiredOffsets(
        _.offsetAndMetadata.commitTimestamp,
        subscribedTopics.get
      )
    case None =&gt;
      getExpiredOffsets(_.offsetAndMetadata.commitTimestamp)
    case _ =&gt;
      Map()
  }
  if (expiredOffsets.nonEmpty)
    debug(s&quot;Expired offsets from group '$groupId': ${expiredOffsets.keySet}&quot;)
  // 将过期位移对应的主题分区从offsets中移除
  offsets --= expiredOffsets.keySet
  // 返回主题分区对应的过期位移
  expiredOffsets
}
</code></pre><p>代码根据消费者组的protocolType类型和组状态调用getExpiredOffsets方法，同时决定传入什么样的参数：</p><ul>
<li>如果消费者组状态是Empty，就传入组变更为Empty状态的时间，若该时间没有被记录，则使用提交位移消息本身的写入时间戳，来获取过期位移；</li>
<li>如果是普通的消费者组类型，且订阅主题信息已知，就传入提交位移消息本身的写入时间戳和订阅主题集合共同确定过期位移值；</li>
<li>如果protocolType为None，就表示，这个消费者组其实是一个Standalone消费者，依然是传入提交位移消息本身的写入时间戳，来决定过期位移值；</li>
<li>如果消费者组的状态不符合刚刚说的这些情况，那就说明，没有过期位移值需要被移除。</li>
</ul><p>当确定了要被移除的位移值集合后，代码会将它们从offsets中移除，然后返回这些被移除的位移值信息。至此，方法结束。</p><h2>分区分配策略管理方法</h2><p>最后，我们讨论下消费者组分区分配策略的管理，也就是字段supportedProtocols的管理。supportedProtocols是分区分配策略的支持票数，这个票数在添加成员、移除成员时，会进行相应的更新。</p><p>消费者组每次Rebalance的时候，都要重新确认本次Rebalance结束之后，要使用哪个分区分配策略，因此，就需要特定的方法来对这些票数进行统计，把票数最多的那个策略作为新的策略。</p><p>GroupMetadata类中定义了两个方法来做这件事情，分别是candidateProtocols和selectProtocol方法。</p><h3>确认消费者组支持的分区分配策略集</h3><p>首先来看candidateProtocols方法。它的作用是<strong>找出组内所有成员都支持的分区分配策略集</strong>。代码如下：</p><pre><code>private def candidateProtocols: Set[String] = {
  val numMembers = members.size // 获取组内成员数
  // 找出支持票数=总成员数的策略，返回它们的名称
  supportedProtocols.filter(_._2 == numMembers).map(_._1).toSet
}
</code></pre><p>该方法首先会获取组内的总成员数，然后，找出supportedProtocols中那些支持票数等于总成员数的分配策略，并返回它们的名称。<strong>支持票数等于总成员数的意思，等同于所有成员都支持该策略</strong>。</p><h3>选出消费者组的分区消费分配策略</h3><p>接下来，我们看下selectProtocol方法，它的作用是<strong>选出消费者组的分区消费分配策略</strong>。</p><pre><code>def selectProtocol: String = {
  // 如果没有任何成员，自然无法确定选用哪个策略
  if (members.isEmpty)
    throw new IllegalStateException(&quot;Cannot select protocol for empty group&quot;)
  // 获取所有成员都支持的策略集合
  val candidates = candidateProtocols
  // 让每个成员投票，票数最多的那个策略当选
  val (protocol, _) = allMemberMetadata
    .map(_.vote(candidates))
    .groupBy(identity)
    .maxBy { case (_, votes) =&gt; votes.size }
  protocol
}
</code></pre><p>这个方法首先会判断组内是否有成员。如果没有任何成员，自然就无法确定选用哪个策略了，方法就会抛出异常，并退出。否则的话，代码会调用刚才的candidateProtocols方法，获取所有成员都支持的策略集合，然后让每个成员投票，票数最多的那个策略当选。</p><p>你可能会好奇，这里的vote方法是怎么实现的呢？其实，它就是简单地查找而已。我举一个简单的例子，来帮助你理解。</p><p>比如，candidates字段的值是[“策略A”，“策略B”]，成员1支持[“策略B”，“策略A”]，成员2支持[“策略A”，“策略B”，“策略C”]，成员3支持[“策略D”，“策略B”，“策略A”]，那么，vote方法会将candidates与每个成员的支持列表进行比对，找出成员支持列表中第一个包含在candidates中的策略。因此，对于这个例子来说，成员1投票策略B，成员2投票策略A，成员3投票策略B。可以看到，投票的结果是，策略B是两票，策略A是1票。所以，selectProtocol方法返回策略B作为新的策略。</p><p>有一点你需要注意，<strong>成员支持列表中的策略是有顺序的</strong>。这就是说，[“策略B”，“策略A”]和[“策略A”，“策略B”]是不同的，成员会倾向于选择靠前的策略。</p><h2>总结</h2><p>今天，我们结合GroupMetadata源码，学习了Kafka对消费者组元数据的管理，主要包括组状态、成员、位移和分区分配策略四个维度。我建议你在课下再仔细地阅读一下这些管理数据的方法，对照着源码和注释走一遍完整的操作流程。</p><p>另外，在这两节课中，我没有谈及待决成员列表（Pending Members）和待决位移（Pending Offsets）的管理，因为这两个元数据项属于中间临时状态，因此我没有展开讲，不理解这部分代码的话，也不会影响我们理解消费者组元数据以及Coordinator是如何使用它们的。不过，我建议你可以阅读下与它们相关的代码部分。要知道，Kafka是非常喜欢引用中间状态变量来管理各类元数据或状态的。</p><p>现在，我们再来简单回顾下这节课的重点。</p><ul>
<li>消费者组元数据管理：主要包括对组状态、成员、位移和分区分配策略的管理。</li>
<li>组状态管理：transitionTo方法负责设置状态，is、not和get方法用于查询状态。</li>
<li>成员管理：add、remove方法用于增减成员，has和get方法用于查询特定成员。</li>
<li>分区分配策略管理：定义了专属方法selectProtocols，用于在每轮Rebalance时选举分区分配策略。</li>
</ul><p><img src="https://static001.geekbang.org/resource/image/a3/e7/a3eafee6b5d17b97f7661c24ccdcd4e7.jpg?wh=2250*1823" alt=""></p><p>至此，我们花了两节课的时间，详细地学习了消费者组元数据及其管理方法的源码。这些操作元数据的方法被上层调用方GroupCoordinator大量使用，就像我在开头提到的，如果现在我们不彻底掌握这些元数据被操作的手法，等我们学到GroupCoordinator代码时，就会感到有些吃力，所以，你一定要好好地学习这两节课。有了这些基础，等到学习GroupCoordinator源码时，你就能更加深刻地理解它的底层实现原理了。</p><h2>课后讨论</h2><p>在讲到MemberMetadata时，我说过，每个成员都有自己的Rebalance超时时间设置，那么，Kafka是怎么确认消费者组使用哪个成员的超时时间作为整个组的超时时间呢？</p><p>欢迎在留言区写下你的思考和答案，跟我交流讨论，也欢迎你把今天的内容分享给你的朋友。</p>
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
  <div class="_2_QraFYR_0">你好，我是胡夕。我来公布上节课的“课后讨论”题答案啦～<br><br>上节课，我们重点学习了消费者组元数据都有哪些。课后我请你思考这样一个问题：kafka-consumer-groups脚本输出中的ASSIGNMENT-STRATEGY项对应于哪一项元数据。实际上，它对应于GroupMetadata类的protocolName字段，即消费者组分区消费分配策略名称。<br><br>okay，你同意这个说法吗？或者说你有其他的看法吗？我们可以一起讨论下。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-07 22:34:02</div>
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
  <div class="_2_QraFYR_0">每个消费者都有一个自己的rebalance时间，这是为了每个消费者按照自己的需要设置，但是在服务端消费者组需要管理这些组内所有消费者，消费者组也需要一个rebalance时间来平衡消费者，在服务端prepareRebalance方法有个参数delayedRebalance 要么初始化InitialDelayedJoin得到rebalance时间，要么非初始化方法DelayedJoin，而DelayedJoin中参数rebalanceTimeoutMs 从方法timeout.max(member.rebalanceTimeoutMs)计算得到，简单来说就是group中所有consumer的最大的member.rebalanceTimeoutMs。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-02 05:52:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/55/eb/a441eda8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>梁聪明</span>
  </div>
  <div class="_2_QraFYR_0"><br>  def rebalanceTimeoutMs = members.values.foldLeft(0) { (timeout, member) =&gt;<br>    timeout.max(member.rebalanceTimeoutMs)<br>  }</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-06 11:16:58</div>
  </div>
</div>
</div>
</li>
</ul>