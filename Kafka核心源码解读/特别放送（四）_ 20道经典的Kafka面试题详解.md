<audio title="特别放送（四）_ 20道经典的Kafka面试题详解" src="https://static001.geekbang.org/resource/audio/a8/0b/a82d1f9a1c21f46d4f9350ba6574f10b.mp3" controls="controls"></audio> 
<p>你好，我是胡夕。这一期的“特别放送”，我想跟你分享一些常见的Kafka面试题。</p><p>无论是作为面试官，还是应聘者，我都接触过很多Kafka面试题。有的题目侧重于基础的概念考核，有的关注实际场景下的解决方案，有的属于“炫技式”，有的可算是深入思考后的“灵魂拷问”。“炫技”类的问题属于冷门的Kafka组件知识考核，而“灵魂拷问”类的问题大多是对Kafka设计原理的深入思考，有很高的技术难度。</p><p>每类题目的应对方法其实不太一样。今天，我就按照这4种类别，具体讲解20道面试题。不过，我不打算只给出答案，我会把面试题的考核初衷也一并写出来。同时，我还会给你分享一些面试小技巧，希望能够帮你更顺利地获取心仪的offer。</p><p>那么，话不多说，我们现在开始吧。</p><h2>基础题目</h2><h3>1.Apache Kafka是什么？</h3><p>这是一道很常见的题目，看似很无聊，其实考核的知识点很多。</p><p>首先，它考验的是，你对Kafka的定位认知是否准确。Apache Kafka一路发展到现在，已经由最初的分布式提交日志系统逐渐演变成了实时流处理框架。因此，这道题你最好这么回答：<strong>Apach Kafka是一款分布式流处理框架，用于实时构建流处理应用。它有一个核心的功能广为人知，即作为企业级的消息引擎被广泛使用。</strong></p><!-- [[[read_end]]] --><p>其实，这里暗含了一个小技巧。Kafka被定位为实时流处理框架，在国内的接受度还不是很高，因此，在回答这道题的时候，你一定要先明确它的流处理框架地位，这样能给面试官留下一个很专业的印象。</p><h3>2.什么是消费者组？</h3><p>从某种程度上说，这可是个“送命题”。消费者组是Kafka独有的概念，如果面试官问这个，就说明他对此是有一定了解的。我先给出标准答案：<strong>关于它的定义，官网上的介绍言简意赅，即消费者组是Kafka提供的可扩展且具有容错性的消费者机制。</strong>切记，一定要加上前面那句，以显示你对官网很熟悉。</p><p>另外，你最好再解释下消费者组的原理：<strong>在Kafka中，消费者组是一个由多个消费者实例构成的组。多个实例共同订阅若干个主题，实现共同消费。同一个组下的每个实例都配置有相同的组ID，被分配不同的订阅分区。当某个实例挂掉的时候，其他实例会自动地承担起它负责消费的分区。</strong></p><p>此时，又有一个小技巧给到你：消费者组的题目，能够帮你在某种程度上掌控下面的面试方向。</p><ul>
<li>如果你擅长<strong>位移值原理</strong>，就不妨再提一下消费者组的位移提交机制；</li>
<li>如果你擅长Kafka <strong>Broker</strong>，可以提一下消费者组与Broker之间的交互；</li>
<li>如果你擅长与消费者组完全不相关的<strong>Producer</strong>，那么就可以这么说：“消费者组要消费的数据完全来自于Producer端生产的消息，我对Producer还是比较熟悉的。”</li>
</ul><p>使用这个策略的话，面试官可能会被你的话术所影响，偏离他之前想问的知识路径。当然了，如果你什么都不擅长，那就继续往下看题目吧。</p><h3>3.在Kafka中，ZooKeeper的作用是什么？</h3><p>这是一道能够帮助你脱颖而出的题目。碰到这个题目，请在心中暗笑三声。</p><p>先说标准答案：<strong>目前，Kafka使用ZooKeeper存放集群元数据、成员管理、Controller选举，以及其他一些管理类任务。之后，等KIP-500提案完成后，Kafka将完全不再依赖于ZooKeeper。</strong></p><p>记住，<strong>一定要突出“目前”</strong>，以彰显你非常了解社区的演进计划。“存放元数据”是指主题分区的所有数据都保存在ZooKeeper中，且以它保存的数据为权威，其他“人”都要与它保持对齐。“成员管理”是指Broker节点的注册、注销以及属性变更，等等。“Controller选举”是指选举集群Controller，而其他管理类任务包括但不限于主题删除、参数配置等。</p><p>不过，抛出KIP-500也可能是个双刃剑。碰到非常资深的面试官，他可能会进一步追问你KIP-500是做的。一言以蔽之：<strong>KIP-500思想，是使用社区自研的基于Raft的共识算法，替代ZooKeeper，实现Controller自选举</strong>。你可能会担心，如果他继续追问的话，该怎么办呢？别怕，在下一期“特别发送”，我会专门讨论这件事。</p><h3>4.解释下Kafka中位移（offset）的作用</h3><p>这也是一道常见的面试题。位移概念本身并不复杂，你可以这么回答：<strong>在Kafka中，每个主题分区下的每条消息都被赋予了一个唯一的ID数值，用于标识它在分区中的位置。这个ID数值，就被称为位移，或者叫偏移量。一旦消息被写入到分区日志，它的位移值将不能被修改。</strong></p><p>答完这些之后，你还可以把整个面试方向转移到你希望的地方。常见方法有以下3种：</p><ul>
<li>如果你深谙Broker底层日志写入的逻辑，可以强调下消息在日志中的存放格式；</li>
<li>如果你明白位移值一旦被确定不能修改，可以强调下“Log Cleaner组件都不能影响位移值”这件事情；</li>
<li>如果你对消费者的概念还算熟悉，可以再详细说说位移值和消费者位移值之间的区别。</li>
</ul><h3>5.阐述下Kafka中的领导者副本（Leader Replica）和追随者副本（Follower Replica）的区别</h3><p>这道题表面上是考核你对Leader和Follower区别的理解，但很容易引申到Kafka的同步机制上。因此，我建议你主动出击，一次性地把隐含的考点也答出来，也许能够暂时把面试官“唬住”，并体现你的专业性。</p><p>你可以这么回答：<strong>Kafka副本当前分为领导者副本和追随者副本。只有Leader副本才能对外提供读写服务，响应Clients端的请求。Follower副本只是采用拉（PULL）的方式，被动地同步Leader副本中的数据，并且在Leader副本所在的Broker宕机后，随时准备应聘Leader副本。</strong></p><p>通常来说，回答到这个程度，其实才只说了60%，因此，我建议你再回答两个额外的加分项。</p><ul>
<li><strong>强调Follower副本也能对外提供读服务</strong>。自Kafka 2.4版本开始，社区通过引入新的Broker端参数，允许Follower副本有限度地提供读服务。</li>
<li><strong>强调Leader和Follower的消息序列在实际场景中不一致</strong>。很多原因都可能造成Leader和Follower保存的消息序列不一致，比如程序Bug、网络问题等。这是很严重的错误，必须要完全规避。你可以补充下，之前确保一致性的主要手段是高水位机制，但高水位值无法保证Leader连续变更场景下的数据一致性，因此，社区引入了Leader Epoch机制，来修复高水位值的弊端。关于“Leader Epoch机制”，国内的资料不是很多，它的普及度远不如高水位，不妨大胆地把这个概念秀出来，力求惊艳一把。上一季专栏的<a href="https://time.geekbang.org/column/article/112118">第27节课</a>讲的就是Leader Epoch机制的原理，推荐你赶紧去学习下。</li>
</ul><h2>实操题目</h2><h3>6.如何设置Kafka能接收的最大消息的大小？</h3><p>这道题除了要回答消费者端的参数设置之外，一定要加上Broker端的设置，这样才算完整。毕竟，如果Producer都不能向Broker端发送数据很大的消息，又何来消费一说呢？因此，你需要同时设置Broker端参数和Consumer端参数。</p><ul>
<li>Broker端参数：message.max.bytes、max.message.bytes（主题级别）和replica.fetch.max.bytes。</li>
<li>Consumer端参数：fetch.message.max.bytes。</li>
</ul><p>Broker端的最后一个参数比较容易遗漏。我们必须调整Follower副本能够接收的最大消息的大小，否则，副本同步就会失败。因此，把这个答出来的话，就是一个加分项。</p><h3>7.监控Kafka的框架都有哪些？</h3><p>其实，目前业界并没有公认的解决方案，各家都有各自的监控之道。所以，面试官其实是在考察你对监控框架的了解广度，或者说，你是否知道很多能监控Kafka的框架或方法。下面这些就是Kafka发展历程上比较有名气的监控系统。</p><ul>
<li><strong>Kafka Manager</strong>：应该算是最有名的专属Kafka监控框架了，是独立的监控系统。</li>
<li><strong>Kafka Monitor</strong>：LinkedIn开源的免费框架，支持对集群进行系统测试，并实时监控测试结果。</li>
<li><strong>CruiseControl</strong>：也是LinkedIn公司开源的监控框架，用于实时监测资源使用率，以及提供常用运维操作等。无UI界面，只提供REST API。</li>
<li><strong>JMX监控</strong>：由于Kafka提供的监控指标都是基于JMX的，因此，市面上任何能够集成JMX的框架都可以使用，比如Zabbix和Prometheus。</li>
<li><strong>已有大数据平台自己的监控体系</strong>：像Cloudera提供的CDH这类大数据平台，天然就提供Kafka监控方案。</li>
<li><strong>JMXTool</strong>：社区提供的命令行工具，能够实时监控JMX指标。答上这一条，属于绝对的加分项，因为知道的人很少，而且会给人一种你对Kafka工具非常熟悉的感觉。如果你暂时不了解它的用法，可以在命令行以无参数方式执行一下<code>kafka-run-class.sh kafka.tools.JmxTool</code>，学习下它的用法。</li>
</ul><h3>8.Broker的Heap Size如何设置？</h3><p>如何设置Heap Size的问题，其实和Kafka关系不大，它是一类非常通用的面试题目。一旦你应对不当，面试方向很有可能被引到JVM和GC上去，那样的话，你被问住的几率就会增大。因此，我建议你简单地介绍一下Heap Size的设置方法，并把重点放在Kafka Broker堆大小设置的最佳实践上。</p><p>比如，你可以这样回复：<strong>任何Java进程JVM堆大小的设置都需要仔细地进行考量和测试。一个常见的做法是，以默认的初始JVM堆大小运行程序，当系统达到稳定状态后，手动触发一次Full GC，然后通过JVM工具查看GC后的存活对象大小。之后，将堆大小设置成存活对象总大小的1.5~2倍。对于Kafka而言，这个方法也是适用的。不过，业界有个最佳实践，那就是将Broker的Heap Size固定为6GB。经过很多公司的验证，这个大小是足够且良好的</strong>。</p><h3>9.如何估算Kafka集群的机器数量？</h3><p>这道题目考查的是<strong>机器数量和所用资源之间的关联关系</strong>。所谓资源，也就是CPU、内存、磁盘和带宽。</p><p>通常来说，CPU和内存资源的充足是比较容易保证的，因此，你需要从磁盘空间和带宽占用两个维度去评估机器数量。</p><p>在预估磁盘的占用时，你一定不要忘记计算副本同步的开销。如果一条消息占用1KB的磁盘空间，那么，在有3个副本的主题中，你就需要3KB的总空间来保存这条消息。显式地将这些考虑因素答出来，能够彰显你考虑问题的全面性，是一个难得的加分项。</p><p>对于评估带宽来说，常见的带宽有1Gbps和10Gbps，但你要切记，<strong>这两个数字仅仅是最大值</strong>。因此，你最好和面试官确认一下给定的带宽是多少。然后，明确阐述出当带宽占用接近总带宽的90%时，丢包情形就会发生。这样能显示出你的网络基本功。</p><h3>10.Leader总是-1，怎么破？</h3><p>在生产环境中，你一定碰到过“某个主题分区不能工作了”的情形。使用命令行查看状态的话，会发现Leader是-1，于是，你使用各种命令都无济于事，最后只能用“重启大法”。</p><p>但是，有没有什么办法，可以不重启集群，就能解决此事呢？这就是此题的由来。</p><p>我直接给答案：<strong>删除ZooKeeper节点/controller，触发Controller重选举。Controller重选举能够为所有主题分区重刷分区状态，可以有效解决因不一致导致的Leader不可用问题</strong>。我几乎可以断定，当面试官问出此题时，要么就是他真的不知道怎么解决在向你寻求答案，要么他就是在等你说出这个答案。所以，<strong>千万别一上来就说“来个重启”之类的话</strong>。</p><h2>炫技式题目</h2><h3>11.LEO、LSO、AR、ISR、HW都表示什么含义？</h3><p>在我看来，这纯属无聊的炫技。试问我不知道又能怎样呢？！不过既然问到了，我们就统一说一说。</p><ul>
<li><strong>LEO</strong>：Log End Offset。日志末端位移值或末端偏移量，表示日志下一条待插入消息的位移值。举个例子，如果日志有10条消息，位移值从0开始，那么，第10条消息的位移值就是9。此时，LEO = 10。</li>
<li><strong>LSO</strong>：Log Stable Offset。这是Kafka事务的概念。如果你没有使用到事务，那么这个值不存在（其实也不是不存在，只是设置成一个无意义的值）。该值控制了事务型消费者能够看到的消息范围。它经常与Log Start Offset，即日志起始位移值相混淆，因为有些人将后者缩写成LSO，这是不对的。在Kafka中，LSO就是指代Log Stable Offset。</li>
<li><strong>AR</strong>：Assigned Replicas。AR是主题被创建后，分区创建时被分配的副本集合，副本个数由副本因子决定。</li>
<li><strong>ISR</strong>：In-Sync Replicas。Kafka中特别重要的概念，指代的是AR中那些与Leader保持同步的副本集合。在AR中的副本可能不在ISR中，但Leader副本天然就包含在ISR中。关于ISR，还有一个常见的面试题目是<strong>如何判断副本是否应该属于ISR</strong>。目前的判断依据是：<strong>Follower副本的LEO落后Leader LEO的时间，是否超过了Broker端参数replica.lag.time.max.ms值</strong>。如果超过了，副本就会被从ISR中移除。</li>
<li><strong>HW</strong>：高水位值（High watermark）。这是控制消费者可读取消息范围的重要字段。一个普通消费者只能“看到”Leader副本上介于Log Start Offset和HW（不含）之间的所有消息。水位以上的消息是对消费者不可见的。关于HW，问法有很多，我能想到的最高级的问法，就是让你完整地梳理下Follower副本拉取Leader副本、执行同步机制的详细步骤。这就是我们的第20道题的题目，一会儿我会给出答案和解析。</li>
</ul><h3>12.Kafka能手动删除消息吗？</h3><p>其实，Kafka不需要用户手动删除消息。它本身提供了留存策略，能够自动删除过期消息。当然，它是支持手动删除消息的。因此，你最好从这两个维度去回答。</p><ul>
<li>对于设置了Key且参数cleanup.policy=compact的主题而言，我们可以构造一条&lt;Key，null&gt;的消息发送给Broker，依靠Log Cleaner组件提供的功能删除掉该Key的消息。</li>
<li>对于普通主题而言，我们可以使用kafka-delete-records命令，或编写程序调用Admin.deleteRecords方法来删除消息。这两种方法殊途同归，底层都是调用Admin的deleteRecords方法，通过将分区Log Start Offset值抬高的方式间接删除消息。</li>
</ul><h3>13.__consumer_offsets是做什么用的？</h3><p>这是一个内部主题，公开的官网资料很少涉及到。因此，我认为，此题属于面试官炫技一类的题目。你要小心这里的考点：该主题有3个重要的知识点，你一定要全部答出来，才会显得对这块知识非常熟悉。</p><ul>
<li>它是一个内部主题，无需手动干预，由Kafka自行管理。当然，我们可以创建该主题。</li>
<li>它的主要作用是负责注册消费者以及保存位移值。可能你对保存位移值的功能很熟悉，但其实<strong>该主题也是保存消费者元数据的地方。千万记得把这一点也回答上</strong>。另外，这里的消费者泛指消费者组和独立消费者，而不仅仅是消费者组。</li>
<li>Kafka的GroupCoordinator组件提供对该主题完整的管理功能，包括该主题的创建、写入、读取和Leader维护等。</li>
</ul><h3>14.分区Leader选举策略有几种？</h3><p>分区的Leader副本选举对用户是完全透明的，它是由Controller独立完成的。你需要回答的是，在哪些场景下，需要执行分区Leader选举。每一种场景对应于一种选举策略。当前，Kafka有4种分区Leader选举策略。</p><ul>
<li><strong>OfflinePartition Leader选举</strong>：每当有分区上线时，就需要执行Leader选举。所谓的分区上线，可能是创建了新分区，也可能是之前的下线分区重新上线。这是最常见的分区Leader选举场景。</li>
<li><strong>ReassignPartition Leader选举</strong>：当你手动运行kafka-reassign-partitions命令，或者是调用Admin的alterPartitionReassignments方法执行分区副本重分配时，可能触发此类选举。假设原来的AR是[1，2，3]，Leader是1，当执行副本重分配后，副本集合AR被设置成[4，5，6]，显然，Leader必须要变更，此时会发生Reassign Partition Leader选举。</li>
<li><strong>PreferredReplicaPartition Leader选举</strong>：当你手动运行kafka-preferred-replica-election命令，或自动触发了Preferred Leader选举时，该类策略被激活。所谓的Preferred Leader，指的是AR中的第一个副本。比如AR是[3，2，1]，那么，Preferred Leader就是3。</li>
<li><strong>ControlledShutdownPartition Leader选举</strong>：当Broker正常关闭时，该Broker上的所有Leader副本都会下线，因此，需要为受影响的分区执行相应的Leader选举。</li>
</ul><p>这4类选举策略的大致思想是类似的，即从AR中挑选首个在ISR中的副本，作为新Leader。当然，个别策略有些微小差异。不过，回答到这种程度，应该足以应付面试官了。毕竟，微小差别对选举Leader这件事的影响很小。</p><h3>15.Kafka的哪些场景中使用了零拷贝（Zero Copy）？</h3><p>Zero Copy是特别容易被问到的高阶题目。在Kafka中，体现Zero Copy使用场景的地方有两处：<strong>基于mmap的索引</strong>和<strong>日志文件读写所用的TransportLayer</strong>。</p><p>先说第一个。索引都是基于MappedByteBuffer的，也就是让用户态和内核态共享内核态的数据缓冲区，此时，数据不需要复制到用户态空间。不过，mmap虽然避免了不必要的拷贝，但不一定就能保证很高的性能。在不同的操作系统下，mmap的创建和销毁成本可能是不一样的。很高的创建和销毁开销会抵消Zero Copy带来的性能优势。由于这种不确定性，在Kafka中，只有索引应用了mmap，最核心的日志并未使用mmap机制。</p><p>再说第二个。TransportLayer是Kafka传输层的接口。它的某个实现类使用了FileChannel的transferTo方法。该方法底层使用sendfile实现了Zero Copy。对Kafka而言，如果I/O通道使用普通的PLAINTEXT，那么，Kafka就可以利用Zero Copy特性，直接将页缓存中的数据发送到网卡的Buffer中，避免中间的多次拷贝。相反，如果I/O通道启用了SSL，那么，Kafka便无法利用Zero Copy特性了。</p><h2>深度思考题</h2><h3>16.Kafka为什么不支持读写分离？</h3><p>这道题目考察的是你对Leader/Follower模型的思考。</p><p>Leader/Follower模型并没有规定Follower副本不可以对外提供读服务。很多框架都是允许这么做的，只是Kafka最初为了避免不一致性的问题，而采用了让Leader统一提供服务的方式。</p><p>不过，在开始回答这道题时，你可以率先亮出观点：<strong>自Kafka 2.4之后，Kafka提供了有限度的读写分离，也就是说，Follower副本能够对外提供读服务。</strong></p><p>说完这些之后，你可以再给出之前的版本不支持读写分离的理由。</p><ul>
<li><strong>场景不适用</strong>。读写分离适用于那种读负载很大，而写操作相对不频繁的场景，可Kafka不属于这样的场景。</li>
<li><strong>同步机制</strong>。Kafka采用PULL方式实现Follower的同步，因此，Follower与Leader存在不一致性窗口。如果允许读Follower副本，就势必要处理消息滞后（Lagging）的问题。</li>
</ul><h3>17.如何调优Kafka？</h3><p>回答任何调优问题的第一步，就是<strong>确定优化目标，并且定量给出目标</strong>！这点特别重要。对于Kafka而言，常见的优化目标是吞吐量、延时、持久性和可用性。每一个方向的优化思路都是不同的，甚至是相反的。</p><p>确定了目标之后，还要明确优化的维度。有些调优属于通用的优化思路，比如对操作系统、JVM等的优化；有些则是有针对性的，比如要优化Kafka的TPS。我们需要从3个方向去考虑。</p><ul>
<li><strong>Producer端</strong>：增加batch.size、linger.ms，启用压缩，关闭重试等。</li>
<li><strong>Broker端</strong>：增加num.replica.fetchers，提升Follower同步TPS，避免Broker Full GC等。</li>
<li><strong>Consumer</strong>：增加fetch.min.bytes等</li>
</ul><h3>18.Controller发生网络分区（Network Partitioning）时，Kafka会怎么样？</h3><p>这道题目能够诱发我们对分布式系统设计、CAP理论、一致性等多方面的思考。不过，针对故障定位和分析的这类问题，我建议你首先言明“实用至上”的观点，即不论怎么进行理论分析，永远都要以实际结果为准。一旦发生Controller网络分区，那么，第一要务就是查看集群是否出现“脑裂”，即同时出现两个甚至是多个Controller组件。这可以根据Broker端监控指标ActiveControllerCount来判断。</p><p>现在，我们分析下，一旦出现这种情况，Kafka会怎么样。</p><p>由于Controller会给Broker发送3类请求，即LeaderAndIsrRequest、StopReplicaRequest和UpdateMetadataRequest，因此，一旦出现网络分区，这些请求将不能顺利到达Broker端。这将影响主题的创建、修改、删除操作的信息同步，表现为集群仿佛僵住了一样，无法感知到后面的所有操作。因此，网络分区通常都是非常严重的问题，要赶快修复。</p><h3>19.Java Consumer为什么采用单线程来获取消息？</h3><p>在回答之前，如果先把这句话说出来，一定会加分：<strong>Java Consumer是双线程的设计。一个线程是用户主线程，负责获取消息；另一个线程是心跳线程，负责向Kafka汇报消费者存活情况。将心跳单独放入专属的线程，能够有效地规避因消息处理速度慢而被视为下线的“假死”情况。</strong></p><p>单线程获取消息的设计能够避免阻塞式的消息获取方式。单线程轮询方式容易实现异步非阻塞式，这样便于将消费者扩展成支持实时流处理的操作算子。因为很多实时流处理操作算子都不能是阻塞式的。另外一个可能的好处是，可以简化代码的开发。多线程交互的代码是非常容易出错的。</p><h3>20.简述Follower副本消息同步的完整流程</h3><p>首先，Follower发送FETCH请求给Leader。接着，Leader会读取底层日志文件中的消息数据，再更新它内存中的Follower副本的LEO值，更新为FETCH请求中的fetchOffset值。最后，尝试更新分区高水位值。Follower接收到FETCH响应之后，会把消息写入到底层日志，接着更新LEO和HW值。</p><p>Leader和Follower的HW值更新时机是不同的，Follower的HW更新永远落后于Leader的HW。这种时间上的错配是造成各种不一致的原因。</p><p>好了，今天的面试题分享就先到这里啦。你有遇到过什么经典的面试题吗？或者是有什么好的面试经验吗？</p><p>欢迎你在留言区分享。如果你觉得今天的内容对你有所帮助，也欢迎分享给你的朋友。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/54/b5/310dae7b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小刀</span>
  </div>
  <div class="_2_QraFYR_0">多谢胡老师，两门课程下来我给号称精通Kafka的候选者准备了上百道Kafka面试题。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 牛</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-04 15:40:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/7b/57/a9b04544.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>QQ怪</span>
  </div>
  <div class="_2_QraFYR_0">想问下自 Kafka 2.4 之后，Kafka 提供了有限度的读写分离，这个有限度具体指的是哪些</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 用户需要设定哪些follower副本可以对外提供服务网</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-17 17:06:32</div>
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
  <div class="_2_QraFYR_0">胡大大给力！为了我们面试不被虐真是用心良苦！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯嗯，加油加油！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-11 10:37:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/6b/e9/7620ae7e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>雨落～紫竹</span>
  </div>
  <div class="_2_QraFYR_0">这是我看的最细致的一章</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-10 11:22:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/07/41/2d477385.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>柠檬C</span>
  </div>
  <div class="_2_QraFYR_0">单线程获取消息是否也有提交offset方面的考虑？<br>比如2个线程1、2分别获取0~10和11~20offset的消息，结果线程2先处理完了，却需要顾及线程1的处理结果而不敢提交<br>而且kafka本身没有实现重试队列，不能像rocketmq那样，把消费失败的消息丢到重试队列里</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 单线程消费是指单个线程消费一个分区下的数据，在提交位移时不需要考虑其他线程消费同一个分区下数据的进度</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-31 19:49:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/38/56/73f6bafb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>周大培</span>
  </div>
  <div class="_2_QraFYR_0">老师，controller的网络分区概念不太理解，为什么会导致出现两个controller呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 网络分区导致ZooKeeper出现脑裂，有可能出现两个controller的情形。Kafka端可通过ISR相关配置进行缓解</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-26 15:17:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/f9/d1/cd079d95.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Satellite</span>
  </div>
  <div class="_2_QraFYR_0">胡老师你好，看到有设置单条消息大小的题，想到个问题 大小应该也是有个限制的吧，例如消息都是10M以上的，这种合理么 会不会对集群压力很大（broker异常 复制副本等）</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 确实压力会很大，通常也不是很建议传输这么大的消息。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-15 17:31:17</div>
  </div>
</div>
</div>
</li>
</ul>