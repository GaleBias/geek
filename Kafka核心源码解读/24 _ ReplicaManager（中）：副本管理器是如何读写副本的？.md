<audio title="24 _ ReplicaManager（中）：副本管理器是如何读写副本的？" src="https://static001.geekbang.org/resource/audio/8c/1d/8c518f523d1ab848c19dd198b6b17f1d.mp3" controls="controls"></audio> 
<p>你好，我是胡夕。上节课，我们学习了ReplicaManager类的定义和重要字段，今天我们接着学习这个类中的读写副本对象部分的源码。无论是读取副本还是写入副本，都是通过底层的Partition对象完成的，而这些分区对象全部保存在上节课所学的allPartitions字段中。可以说，理解这些字段的用途，是后续我们探索副本管理器类功能的重要前提。</p><p>现在，我们就来学习下副本读写功能。整个Kafka的同步机制，本质上就是副本读取+副本写入，搞懂了这两个功能，你就知道了Follower副本是如何同步Leader副本数据的。</p><h2>副本写入：appendRecords</h2><p>所谓的副本写入，是指向副本底层日志写入消息。在ReplicaManager类中，实现副本写入的方法叫appendRecords。</p><p>放眼整个Kafka源码世界，需要副本写入的场景有4个。</p><ul>
<li>场景一：生产者向Leader副本写入消息；</li>
<li>场景二：Follower副本拉取消息后写入副本；</li>
<li>场景三：消费者组写入组信息；</li>
<li>场景四：事务管理器写入事务信息（包括事务标记、事务元数据等）。</li>
</ul><p>除了第二个场景是直接调用Partition对象的方法实现之外，其他3个都是调用appendRecords来完成的。</p><!-- [[[read_end]]] --><p>该方法将给定一组分区的消息写入到对应的Leader副本中，并且根据PRODUCE请求中acks设置的不同，有选择地等待其他副本写入完成。然后，调用指定的回调逻辑。</p><p>我们先来看下它的方法签名：</p><pre><code>def appendRecords(
  timeout: Long,  // 请求处理超时时间
  requiredAcks: Short,  // 请求acks设置
  internalTopicsAllowed: Boolean,  // 是否允许写入内部主题
  origin: AppendOrigin,  // 写入方来源
  entriesPerPartition: Map[TopicPartition, MemoryRecords], // 待写入消息
  // 回调逻辑 
  responseCallback: Map[TopicPartition, PartitionResponse] =&gt; Unit,
  delayedProduceLock: Option[Lock] = None,
  recordConversionStatsCallback: 
    Map[TopicPartition, RecordConversionStats] =&gt; Unit = _ =&gt; ())
  : Unit = {
  ......
}
</code></pre><p>输入参数有很多，而且都很重要，我一个一个地说。</p><ul>
<li><strong>timeout</strong>：请求处理超时时间。对于生产者来说，它就是request.timeout.ms参数值。</li>
<li><strong>requiredAcks</strong>：是否需要等待其他副本写入。对于生产者而言，它就是acks参数的值。而在其他场景中，Kafka默认使用-1，表示等待其他副本全部写入成功再返回。</li>
<li><strong>internalTopicsAllowed</strong>：是否允许向内部主题写入消息。对于普通的生产者而言，该字段是False，即不允许写入内部主题。对于Coordinator组件，特别是消费者组GroupCoordinator组件来说，它的职责之一就是向内部位移主题写入消息，因此，此时，该字段值是True。</li>
<li><strong>origin</strong>：AppendOrigin是一个接口，表示写入方来源。当前，它定义了3类写入方，分别是Replication、Coordinator和Client。Replication表示写入请求是由Follower副本发出的，它要将从Leader副本获取到的消息写入到底层的消息日志中。Coordinator表示这些写入由Coordinator发起，它既可以是管理消费者组的GroupCooridnator，也可以是管理事务的TransactionCoordinator。Client表示本次写入由客户端发起。前面我们说过了，Follower副本同步过程不调用appendRecords方法，因此，这里的origin值只可能是Replication或Coordinator。</li>
<li><strong>entriesPerPartitio</strong>n：按分区分组的、实际要写入的消息集合。</li>
<li><strong>responseCallback</strong>：写入成功之后，要调用的回调逻辑函数。</li>
<li><strong>delayedProduceLock</strong>：专门用来保护消费者组操作线程安全的锁对象，在其他场景中用不到。</li>
<li><strong>recordConversionStatsCallback</strong>：消息格式转换操作的回调统计逻辑，主要用于统计消息格式转换操作过程中的一些数据指标，比如总共转换了多少条消息，花费了多长时间。</li>
</ul><p>接下来，我们就看看，appendRecords如何利用这些输入参数向副本日志写入消息。我把它的完整代码贴出来。对于重要的步骤，我标注了注释：</p><pre><code>// requiredAcks合法取值是-1，0，1，否则视为非法
if (isValidRequiredAcks(requiredAcks)) {
  val sTime = time.milliseconds
  // 调用appendToLocalLog方法写入消息集合到本地日志
  val localProduceResults = appendToLocalLog(
    internalTopicsAllowed = internalTopicsAllowed,
    origin, entriesPerPartition, requiredAcks)
  debug(&quot;Produce to local log in %d ms&quot;.format(time.milliseconds - sTime))
  val produceStatus = localProduceResults.map { case (topicPartition, result) =&gt;
    topicPartition -&gt;
            ProducePartitionStatus(
              result.info.lastOffset + 1, // 设置下一条待写入消息的位移值
              // 构建PartitionResponse封装写入结果
              new PartitionResponse(result.error, result.info.firstOffset.getOrElse(-1), result.info.logAppendTime,
                result.info.logStartOffset, result.info.recordErrors.asJava, result.info.errorMessage))
  }
  // 尝试更新消息格式转换的指标数据
  recordConversionStatsCallback(localProduceResults.map { case (k, v) =&gt; k -&gt; v.info.recordConversionStats })
  // 需要等待其他副本完成写入
  if (delayedProduceRequestRequired(
    requiredAcks, entriesPerPartition, localProduceResults)) {
    val produceMetadata = ProduceMetadata(requiredAcks, produceStatus)
    // 创建DelayedProduce延时请求对象
    val delayedProduce = new DelayedProduce(timeout, produceMetadata, this, responseCallback, delayedProduceLock)
    val producerRequestKeys = entriesPerPartition.keys.map(TopicPartitionOperationKey(_)).toSeq
    // 再一次尝试完成该延时请求
    // 如果暂时无法完成，则将对象放入到相应的Purgatory中等待后续处理
    delayedProducePurgatory.tryCompleteElseWatch(delayedProduce, producerRequestKeys)
  } else { // 无需等待其他副本写入完成，可以立即发送Response 
    val produceResponseStatus = produceStatus.map { case (k, status) =&gt; k -&gt; status.responseStatus }
    // 调用回调逻辑然后返回即可
    responseCallback(produceResponseStatus)
  }
} else { // 如果requiredAcks值不合法
  val responseStatus = entriesPerPartition.map { case (topicPartition, _) =&gt;
    topicPartition -&gt; new PartitionResponse(Errors.INVALID_REQUIRED_ACKS,
      LogAppendInfo.UnknownLogAppendInfo.firstOffset.getOrElse(-1), RecordBatch.NO_TIMESTAMP, LogAppendInfo.UnknownLogAppendInfo.logStartOffset)
  }
  // 构造INVALID_REQUIRED_ACKS异常并封装进回调函数调用中
  responseCallback(responseStatus)
}
</code></pre><p>为了帮助你更好地理解，我再用一张图说明一下appendRecords方法的完整流程。</p><p><img src="https://static001.geekbang.org/resource/image/52/d4/52f1dc751ecfc95f509d1f001ff551d4.jpg?wh=2380*3385" alt=""></p><p>我再给你解释一下它的执行流程。</p><p>首先，它会判断requiredAcks的取值是否在合理范围内，也就是“是否是-1、0、1这3个数值中的一个”。如果不是合理取值，代码就进入到外层的else分支，构造名为INVALID_REQUIRED_ACKS的异常，并将其封装进回调函数中执行，然后返回结果。否则的话，代码进入到外层的if分支下。</p><p>进入到if分支后，代码调用<strong>appendToLocalLog</strong>方法，将要写入的消息集合保存到副本的本地日志上。然后构造PartitionResponse对象实例，来封装写入结果以及一些重要的元数据信息，比如本次写入有没有错误（errorMessage）、下一条待写入消息的位移值、本次写入消息集合首条消息的位移值，等等。待这些做完了之后，代码会尝试更新消息格式转换的指标数据。此时，源码需要调用delayedProduceRequestRequired方法，来判断本次写入是否算是成功了。</p><p>如果还需要等待其他副本同步完成消息写入，那么就不能立即返回，代码要创建DelayedProduce延时请求对象，并把该对象交由Purgatory来管理。DelayedProduce是生产者端的延时发送请求，对应的Purgatory就是ReplicaManager类构造函数中的delayedProducePurgatory。所谓的Purgatory管理，主要是调用tryCompleteElseWatch方法尝试完成延时发送请求。如果暂时无法完成，就将对象放入到相应的Purgatory中，等待后续处理。</p><p>如果无需等待其他副本同步完成消息写入，那么，appendRecords方法会构造响应的Response，并调用回调逻辑函数，至此，方法结束。</p><p>从刚刚的分析中，我们可以知道，appendRecords实现消息写入的方法是<strong>appendToLocalLog</strong>，用于判断是否需要等待其他副本写入的方法是<strong>delayedProduceRequestRequired</strong>。下面我们就深入地学习下这两个方法的代码。</p><p>首先来看appendToLocalLog。从它的名字来看，就是写入副本本地日志。我们来看一下该方法的主要代码片段。</p><pre><code>private def appendToLocalLog(
  internalTopicsAllowed: Boolean,
  origin: AppendOrigin,
  entriesPerPartition: Map[TopicPartition, MemoryRecords],
  requiredAcks: Short): Map[TopicPartition, LogAppendResult] = {
  ......
  entriesPerPartition.map { case (topicPartition, records) =&gt;
    brokerTopicStats.topicStats(topicPartition.topic)
      .totalProduceRequestRate.mark()
    brokerTopicStats.allTopicsStats.totalProduceRequestRate.mark()
    // 如果要写入的主题是内部主题，而internalTopicsAllowed=false，则返回错误
    if (Topic.isInternal(topicPartition.topic) 
      &amp;&amp; !internalTopicsAllowed) {
      (topicPartition, LogAppendResult(
        LogAppendInfo.UnknownLogAppendInfo,
        Some(new InvalidTopicException(s&quot;Cannot append to internal topic ${topicPartition.topic}&quot;))))
    } else {
      try {
        // 获取分区对象
        val partition = getPartitionOrException(topicPartition, expectLeader = true)
        // 向该分区对象写入消息集合
        val info = partition.appendRecordsToLeader(records, origin, requiredAcks)
        ......
        // 返回写入结果
        (topicPartition, LogAppendResult(info))
      } catch {
        ......
      }
    }
  }
}
</code></pre><p>我忽略了很多打日志以及错误处理的代码。你可以看到，该方法主要就是利用Partition的appendRecordsToLeader方法写入消息集合，而后者就是利用我们在<a href="https://time.geekbang.org/column/article/225993">第3节课</a>学到的appendAsLeader方法写入本地日志的。总体来说，appendToLocalLog的逻辑不复杂，你应该很容易理解。</p><p>下面我们看下delayedProduceRequestRequired方法的源码。它用于判断消息集合被写入到日志之后，是否需要等待其他副本也写入成功。我们看下它的代码：</p><pre><code>private def delayedProduceRequestRequired(
  requiredAcks: Short,
  entriesPerPartition: Map[TopicPartition, MemoryRecords],
  localProduceResults: Map[TopicPartition, LogAppendResult]): Boolean = {
  requiredAcks == -1 &amp;&amp; entriesPerPartition.nonEmpty &amp;&amp; 
    localProduceResults.values.count(_.exception.isDefined) &lt; entriesPerPartition.size
}
</code></pre><p>该方法返回一个布尔值，True表示需要等待其他副本完成；False表示无需等待。上面的代码表明，如果需要等待其他副本的写入，就必须同时满足3个条件：</p><ol>
<li>requiredAcks必须等于-1；</li>
<li>依然有数据尚未写完；</li>
<li>至少有一个分区的消息已经成功地被写入到本地日志。</li>
</ol><p>其实，你可以把条件2和3联合在一起来看。如果所有分区的数据写入都不成功，就表明可能出现了很严重的错误，此时，比较明智的做法是不再等待，而是直接返回错误给发送方。相反地，如果有部分分区成功写入，而部分分区写入失败了，就表明可能是由偶发的瞬时错误导致的。此时，不妨将本次写入请求放入Purgatory，再给它一个重试的机会。</p><h2>副本读取：fetchMessages</h2><p>好了，说完了副本的写入，下面我们进入到副本读取的源码学习。</p><p>在ReplicaManager类中，负责读取副本数据的方法是fetchMessages。不论是Java消费者API，还是Follower副本，它们拉取消息的主要途径都是向Broker发送FETCH请求，Broker端接收到该请求后，调用fetchMessages方法从底层的Leader副本取出消息。</p><p>和appendRecords方法类似，fetchMessages方法也可能会延时处理FETCH请求，因为Broker端必须要累积足够多的数据之后，才会返回Response给请求发送方。</p><p>可以看一下下面的这张流程图，它展示了fetchMessages方法的主要逻辑。</p><p><img src="https://static001.geekbang.org/resource/image/0f/2c/0f4b45008bdf0b83d0865c7db6d5452c.jpg?wh=1760*2793" alt=""></p><p>我们来看下该方法的签名：</p><pre><code>def fetchMessages(timeout: Long,
                  replicaId: Int,
                  fetchMinBytes: Int,
                  fetchMaxBytes: Int,
                  hardMaxBytesLimit: Boolean,
                  fetchInfos: Seq[(TopicPartition, PartitionData)],
                  quota: ReplicaQuota,
                  responseCallback: Seq[(TopicPartition, FetchPartitionData)] =&gt; Unit,
                  isolationLevel: IsolationLevel,
                  clientMetadata: Option[ClientMetadata]): Unit = {
  ......
}
</code></pre><p>这些输入参数都是我们理解下面的重要方法的基础，所以，我们来逐个分析一下。</p><ul>
<li><strong>timeout</strong>：请求处理超时时间。对于消费者而言，该值就是request.timeout.ms参数值；对于Follower副本而言，该值是Broker端参数replica.fetch.wait.max.ms的值。</li>
<li><strong>replicaId</strong>：副本ID。对于消费者而言，该参数值是-1；对于Follower副本而言，该值就是Follower副本所在的Broker ID。</li>
<li><strong>fetchMinBytes &amp; fetchMaxBytes</strong>：能够获取的最小字节数和最大字节数。对于消费者而言，它们分别对应于Consumer端参数fetch.min.bytes和fetch.max.bytes值；对于Follower副本而言，它们分别对应于Broker端参数replica.fetch.min.bytes和replica.fetch.max.bytes值。</li>
<li><strong>hardMaxBytesLimit</strong>：对能否超过最大字节数做硬限制。如果hardMaxBytesLimit=True，就表示，读取请求返回的数据字节数绝不允许超过最大字节数。</li>
<li><strong>fetchInfos</strong>：规定了读取分区的信息，比如要读取哪些分区、从这些分区的哪个位移值开始读、最多可以读多少字节，等等。</li>
<li><strong>quota</strong>：这是一个配额控制类，主要是为了判断是否需要在读取的过程中做限速控制。</li>
<li><strong>responseCallback</strong>：Response回调逻辑函数。当请求被处理完成后，调用该方法执行收尾逻辑。</li>
</ul><p>有了这些铺垫之后，我们进入到方法代码的学习。为了便于学习，我将整个方法的代码分成两部分：第一部分是读取本地日志；第二部分是根据读取结果确定Response。</p><p>我们先看第一部分的源码：</p><pre><code>// 判断该读取请求是否来自于Follower副本或Consumer
val isFromFollower = Request.isValidBrokerId(replicaId)
val isFromConsumer = !(isFromFollower || replicaId == Request.FutureLocalReplicaId)
// 根据请求发送方判断可读取范围
// 如果请求来自于普通消费者，那么可以读到高水位值
// 如果请求来自于配置了READ_COMMITTED的消费者，那么可以读到Log Stable Offset值
// 如果请求来自于Follower副本，那么可以读到LEO值
val fetchIsolation = if (!isFromConsumer)
  FetchLogEnd
else if (isolationLevel == IsolationLevel.READ_COMMITTED)
  FetchTxnCommitted
else
  FetchHighWatermark
val fetchOnlyFromLeader = isFromFollower || (isFromConsumer &amp;&amp; clientMetadata.isEmpty)
// 定义readFromLog方法读取底层日志中的消息
def readFromLog(): Seq[(TopicPartition, LogReadResult)] = {
  val result = readFromLocalLog(
    replicaId = replicaId,
    fetchOnlyFromLeader = fetchOnlyFromLeader,
    fetchIsolation = fetchIsolation,
    fetchMaxBytes = fetchMaxBytes,
    hardMaxBytesLimit = hardMaxBytesLimit,
    readPartitionInfo = fetchInfos,
    quota = quota,
    clientMetadata = clientMetadata)
  if (isFromFollower) updateFollowerFetchState(replicaId, result)
  else result
}
// 读取消息并返回日志读取结果
val logReadResults = readFromLog()
</code></pre><p>这部分代码首先会判断，读取消息的请求方到底是Follower副本，还是普通的Consumer。判断的依据就是看<strong>replicaId字段是否大于0</strong>。Consumer的replicaId是-1，而Follower副本的则是大于0的数。一旦确定了请求方，代码就能确定可读取范围。</p><p>这里的fetchIsolation是读取隔离级别的意思。对于Follower副本而言，它能读取到Leader副本LEO值以下的所有消息；对于普通Consumer而言，它只能“看到”Leader副本高水位值以下的消息。</p><p>待确定了可读取范围后，fetchMessages方法会调用它的内部方法<strong>readFromLog</strong>，读取本地日志上的消息数据，并将结果赋值给logReadResults变量。readFromLog方法的主要实现是调用readFromLocalLog方法，而后者就是在待读取分区上依次调用其日志对象的read方法执行实际的消息读取。</p><p>fetchMessages方法的第二部分，是根据上一步的读取结果创建对应的Response。我们看下具体实现：</p><pre><code>var bytesReadable: Long = 0
var errorReadingData = false
val logReadResultMap = new mutable.HashMap[TopicPartition, LogReadResult]
// 统计总共可读取的字节数
logReadResults.foreach { case (topicPartition, logReadResult) =&gt;
 brokerTopicStats.topicStats(topicPartition.topic).totalFetchRequestRate.mark()
  brokerTopicStats.allTopicsStats.totalFetchRequestRate.mark()
  if (logReadResult.error != Errors.NONE)
    errorReadingData = true
  bytesReadable = bytesReadable + logReadResult.info.records.sizeInBytes
  logReadResultMap.put(topicPartition, logReadResult)
}
// 判断是否能够立即返回Reponse，满足以下4个条件中的任意一个即可：
// 1. 请求没有设置超时时间，说明请求方想让请求被处理后立即返回
// 2. 未获取到任何数据
// 3. 已累积到足够多的数据
// 4. 读取过程中出错
if (timeout &lt;= 0 || fetchInfos.isEmpty || bytesReadable &gt;= fetchMinBytes || errorReadingData) {
  // 构建返回结果
  val fetchPartitionData = logReadResults.map { case (tp, result) =&gt;
    tp -&gt; FetchPartitionData(result.error, result.highWatermark, result.leaderLogStartOffset, result.info.records,
      result.lastStableOffset, result.info.abortedTransactions, result.preferredReadReplica, isFromFollower &amp;&amp; isAddingReplica(tp, replicaId))
  }
  // 调用回调函数
  responseCallback(fetchPartitionData)
} else { // 如果无法立即完成请求
  val fetchPartitionStatus = new mutable.ArrayBuffer[(TopicPartition, FetchPartitionStatus)]
  fetchInfos.foreach { case (topicPartition, partitionData) =&gt;
    logReadResultMap.get(topicPartition).foreach(logReadResult =&gt; {
      val logOffsetMetadata = logReadResult.info.fetchOffsetMetadata
      fetchPartitionStatus += (topicPartition -&gt; FetchPartitionStatus(logOffsetMetadata, partitionData))
    })
  }
  val fetchMetadata: SFetchMetadata = SFetchMetadata(fetchMinBytes, fetchMaxBytes, hardMaxBytesLimit,
    fetchOnlyFromLeader, fetchIsolation, isFromFollower, replicaId, fetchPartitionStatus)
  // 构建DelayedFetch延时请求对象  
  val delayedFetch = new DelayedFetch(timeout, fetchMetadata, this, quota, clientMetadata,
    responseCallback)
  val delayedFetchKeys = fetchPartitionStatus.map { case (tp, _) =&gt; TopicPartitionOperationKey(tp) }
  // 再一次尝试完成请求，如果依然不能完成，则交由Purgatory等待后续处理
  delayedFetchPurgatory.tryCompleteElseWatch(delayedFetch, delayedFetchKeys)
}
</code></pre><p>这部分代码首先会根据上一步得到的读取结果，统计可读取的总字节数，之后，判断此时是否能够立即返回Reponse。那么，怎么判断是否能够立即返回Response呢？实际上，只要满足以下4个条件中的任意一个即可：</p><ol>
<li>请求没有设置超时时间，说明请求方想让请求被处理后立即返回；</li>
<li>未获取到任何数据；</li>
<li>已累积到足够多数据；</li>
<li>读取过程中出错。</li>
</ol><p>如果这4个条件一个都不满足，就需要进行延时处理了。具体来说，就是构建DelayedFetch对象，然后把该延时对象交由delayedFetchPurgatory后续自动处理。</p><p>至此，关于副本管理器读写副本的两个方法appendRecords和fetchMessages，我们就学完了。本质上，它们在底层分别调用Log的append和read方法，以实现本地日志的读写操作。当完成读写操作之后，这两个方法还定义了延时处理的条件。一旦发现满足了延时处理的条件，就交给对应的Purgatory进行处理。</p><p>从这两个方法中，我们已经看到了之前课程中单个组件融合在一起的趋势。就像我在开篇词里面说的，虽然我们学习单个源码文件的顺序是自上而下，但串联Kafka主要组件功能的路径却是自下而上。</p><p>就拿这节课的副本写入操作来说，日志对象的append方法被上一层Partition对象中的方法调用，而后者又进一步被副本管理器中的方法调用。我们是按照自上而下的方式阅读副本管理器、日志对象等单个组件的代码，了解它们各自的独立功能的，现在，我们开始慢慢地把它们融合在一起，勾勒出了Kafka操作分区副本日志对象的完整调用路径。咱们同时采用这两种方式来阅读源码，就可以更快、更深入地搞懂Kafka源码的原理了。</p><h2>总结</h2><p>今天，我们学习了Kafka副本状态机类ReplicaManager是如何读写副本的，重点学习了它的两个重要方法appendRecords和fetchMessages。我们再简单回顾一下。</p><ul>
<li>appendRecords：向副本写入消息的方法，主要利用Log的append方法和Purgatory机制，共同实现Follower副本向Leader副本获取消息后的数据同步工作。</li>
<li>fetchMessages：从副本读取消息的方法，为普通Consumer和Follower副本所使用。当它们向Broker发送FETCH请求时，Broker上的副本管理器调用该方法从本地日志中获取指定消息。</li>
</ul><p><img src="https://static001.geekbang.org/resource/image/29/b3/295faae205df4255d2861d658df10db3.jpg?wh=2250*2068" alt=""><br>
下节课中，我们要把重心转移到副本管理器对副本和分区对象的管理上。这是除了读写副本之外，副本管理器另一大核心功能，你一定不要错过！</p><h2>课后讨论</h2><p>appendRecords参数列表中有个origin。我想请你思考一下，在写入本地日志的过程中，这个参数的作用是什么？你能找出最终使用origin参数的具体源码位置吗？</p><p>欢迎在留言区写下你的思考和答案，跟我交流讨论，也欢迎你把今天的内容分享给你的朋友。</p>
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
  <div class="_2_QraFYR_0">你好，我是胡夕。我来公布上节课的“课后讨论”题答案啦～<br><br>上节课，我们重点学习了ReplicaManager类的类定义和核心字段。课后我请你自行写一个统计Online状态分区数的方法。我的代码如下：<br>private def onlinePartitionCount: Int = {<br>    allPartitions.values.iterator.count(_ == HostedPartition.Online)<br>  }<br><br>okay，你同意这个说法吗？或者说你有其他的看法吗？我们可以一起讨论下。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-23 10:48:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/a1/ea/4afba3f1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>云端漫漫步</span>
  </div>
  <div class="_2_QraFYR_0">origin是标识消息来源的，在写入向Leader写入本地日志时候层层传递，最终在analyzeAndValidateRecords中使用；<br>作用是根据origin的值，判断消息格式是否合法：如果是AppendOrigin.Client，消息格式Version 2，且起始batch位移值不为0，抛出InvalidRecordException；</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-10 09:54:58</div>
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
  <div class="_2_QraFYR_0">这里的requiredAcks就是参数request.required.acks的值吗？如果是的话，request.required.acks=-1需要等待其他broker上的副本写入完成才能返回写入成功，这块逻辑在哪里啊？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-15 00:04:12</div>
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
  <div class="_2_QraFYR_0">origin表示写入方来源，在写入日志之前，需要对不同的写入方做不同程度的校验<br>例如，client的需要做full validation：producerEpoch+appendFirstSeq+offset</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-02 22:41:00</div>
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
  <div class="_2_QraFYR_0">场景二：Follower 副本拉取消息后写入副本不调用appendRecords；这样origin 值不就只可能是 Client 或 Coordinator了吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 其他写入方法也需要传入origin，比如Log的append方法。在那里指定了origin=Replication<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-15 00:24:14</div>
  </div>
</div>
</div>
</li>
</ul>