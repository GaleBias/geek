<audio title="21 _ Kafka Consumer源码分析：消息消费的实现过程" src="https://static001.geekbang.org/resource/audio/47/36/476b77f5b0d112d5d932aaf851075236.mp3" controls="controls"></audio> 
<p>你好，我是李玥。</p><p>我们在上节课中提到过，用于解决消息队列一些常见问题的知识和原理，最终落地到代码上，都包含在收、发消息这两个流程中。对于消息队列的生产和消费这两个核心流程，在大部分消息队列中，它实现的主要流程都是一样的，所以，通过这两节课的学习之后，掌握了这两个流程的实现过程。无论你使用的是哪种消息队列，遇到收发消息的问题，你都可以用同样的思路去分析和解决问题。</p><p>上一节课我和你一起通过分析源代码学习了RocketMQ消息生产的实现过程，本节课我们来看一下Kafka消费者的源代码，理清Kafka消费的实现过程，并且能从中学习到一些Kafka的优秀设计思路和编码技巧。</p><p>在开始分析源码之前，我们一起来回顾一下Kafka消费模型的几个要点：</p><ul>
<li>Kafka的每个Consumer（消费者）实例属于一个ConsumerGroup（消费组）；</li>
<li>在消费时，ConsumerGroup中的每个Consumer独占一个或多个Partition（分区）；</li>
<li>对于每个ConsumerGroup，在任意时刻，每个Partition至多有1个Consumer在消费；</li>
<li>每个ConsumerGroup都有一个Coordinator(协调者）负责分配Consumer和Partition的对应关系，当Partition或是Consumer发生变更时，会触发rebalance（重新分配）过程，重新分配Consumer与Partition的对应关系；</li>
<li>Consumer维护与Coordinator之间的心跳，这样Coordinator就能感知到Consumer的状态，在Consumer故障的时候及时触发rebalance。</li>
</ul><!-- [[[read_end]]] --><p>掌握并理解Kafka的消费模型，对于接下来理解其消费的实现过程是至关重要的，如果你对上面的这些要点还有不清楚的地方，建议回顾一下之前的课程或者看一下Kafka相关的文档，然后再继续接下来的内容。</p><p>我们使用当前最新的版本2.2进行分析，使用Git在GitHub上直接下载源码到本地：</p><pre><code>git clone git@github.com:apache/kafka.git
cd kafka
git checkout 2.2
</code></pre><p>在《<a href="https://time.geekbang.org/column/article/115519">09 | 学习开源代码该如何入手？</a>》这节课中，我讲过，分析国外源码最好的方式就是从文档入手，接下来我们就找一下Kafka的文档，看看从哪儿来入手开启我们的分析流程。</p><p>Kafka的Consumer入口类<a href="https://kafka.apache.org/10/javadoc/?org/apache/kafka/clients/consumer/KafkaConsumer.html">KafkaConsumer的JavaDoc</a>，给出了关于如何使用KafkaConsumer非常详细的说明文档，并且给出了一个使用Consumer消费的最简代码示例：</p><pre><code>     // 设置必要的配置信息
     Properties props = new Properties();
     props.put(&quot;bootstrap.servers&quot;, &quot;localhost:9092&quot;);
     props.put(&quot;group.id&quot;, &quot;test&quot;);
     props.put(&quot;enable.auto.commit&quot;, &quot;true&quot;);
     props.put(&quot;auto.commit.interval.ms&quot;, &quot;1000&quot;);
     props.put(&quot;key.deserializer&quot;, &quot;org.apache.kafka.common.serialization.StringDeserializer&quot;);
     props.put(&quot;value.deserializer&quot;, &quot;org.apache.kafka.common.serialization.StringDeserializer&quot;);

     // 创建Consumer实例
     KafkaConsumer&lt;String, String&gt; consumer = new KafkaConsumer&lt;&gt;(props);

     // 订阅Topic
     consumer.subscribe(Arrays.asList(&quot;foo&quot;, &quot;bar&quot;));

     // 循环拉消息
     while (true) {
         ConsumerRecords&lt;String, String&gt; records = consumer.poll(100);
         for (ConsumerRecord&lt;String, String&gt; record : records)
             System.out.printf(&quot;offset = %d, key = %s, value = %s%n&quot;, record.offset(), record.key(), record.value());
     }
</code></pre><p>这段代码主要的主要流程是：</p><ol>
<li>设置必要的配置信息，包括：起始连接的Broker地址，Consumer Group的ID，自动提交消费位置的配置和序列化配置；</li>
<li>创建Consumer实例；</li>
<li>订阅了2个Topic：foo 和 bar；</li>
<li>循环拉取消息并打印在控制台上。</li>
</ol><p>通过上面的代码实例我们可以看到，消费这个大的流程，在Kafka中实际上是被分成了“订阅”和“拉取消息”这两个小的流程。另外，我在之前的课程中反复提到过，Kafka在消费过程中，每个Consumer实例是绑定到一个分区上的，那Consumer是如何确定，绑定到哪一个分区上的呢？这个问题也是可以通过分析消费流程来找到答案的。所以，我们分析整个消费流程主要聚焦在三个问题上：</p><ol>
<li>订阅过程是如何实现的？</li>
<li>Consumer是如何与Coordinator协商，确定消费哪些Partition的？</li>
<li>拉取消息的过程是如何实现的？</li>
</ol><p>了解前两个问题，有助于你充分理解Kafka的元数据模型，以及Kafka是如何在客户端和服务端之间来交换元数据的。最后一个问题，拉取消息的实现过程，实际上就是消费的主要流程，我们上节课讲过，这是消息队列最核心的两个流程之一，也是必须重点掌握的。我们就带着这三个问题，来分析Kafka的订阅和拉取消息的过程如何实现。</p><h2>订阅过程如何实现？</h2><p>我们先来看看订阅的实现流程。从上面的例子跟踪到订阅的主流程方法：</p><pre><code>  public void subscribe(Collection&lt;String&gt; topics, ConsumerRebalanceListener listener) {
      acquireAndEnsureOpen();
      try {
          // 省略部分代码

          // 重置订阅状态
          this.subscriptions.subscribe(new HashSet&lt;&gt;(topics), listener);

          // 更新元数据
          metadata.setTopics(subscriptions.groupSubscription());
      } finally {
          release();
      }
  }
</code></pre><p>在这个代码中，我们先忽略掉各种参数和状态检查的分支代码，订阅的主流程主要更新了两个属性：一个是订阅状态subscriptions，另一个是更新元数据中的topic信息。订阅状态subscriptions主要维护了订阅的topic和patition的消费位置等状态信息。属性metadata中维护了Kafka集群元数据的一个子集，包括集群的Broker节点、Topic和Partition在节点上分布，以及我们聚焦的第二个问题：Coordinator给Consumer分配的Partition信息。</p><p>请注意一下，这个subscribe()方法的实现有一个非常值得大家学习的地方：就是开始的acquireAndEnsureOpen()和try-finally release()，作用就是保护这个方法只能单线程调用。</p><p>Kafka在文档中明确地注明了Consumer不是线程安全的，意味着Consumer被并发调用时会出现不可预期的结果。为了避免这种情况发生，Kafka做了主动的检测并抛出异常，而不是放任系统产生不可预期的情况。</p><p>Kafka“<strong>主动检测不支持的情况并抛出异常，避免系统产生不可预期的行为</strong>”这种模式，对于增强的系统的健壮性是一种非常有效的做法。如果你的系统不支持用户的某种操作，正确的做法是，检测不支持的操作，直接拒绝用户操作，并给出明确的错误提示，而不应该只是在文档中写上“不要这样做”，却放任用户错误的操作，产生一些不可预期的、奇怪的错误结果。</p><p>具体Kafka是如何实现的并发检测，大家可以看一下方法acquireAndEnsureOpen()的实现，很简单也很经典，我们就不再展开讲解了。</p><p>继续跟进到更新元数据的方法metadata.setTopics()里面，这个方法的实现除了更新元数据类Metadata中的topic相关的一些属性以外，还调用了Metadata.requestUpdate()方法请求更新元数据。</p><pre><code>    public synchronized int requestUpdate() {
        this.needUpdate = true;
        return this.updateVersion;
    }
</code></pre><p>跟进到requestUpdate()的方法里面我们会发现，这里面并没有真正发送更新元数据的请求，只是将需要更新元数据的标志位needUpdate设置为true就结束了。Kafka必须确保在第一次拉消息之前元数据是可用的，也就是说在第一次拉消息之前必须更新一次元数据，否则Consumer就不知道它应该去哪个Broker上去拉哪个Partition的消息。</p><p>分析完订阅相关的代码，我们来总结一下：在订阅的实现过程中，Kafka更新了订阅状态subscriptions和元数据metadata中的相关topic的一些属性，将元数据状态置为“需要立即更新”，但是并没有真正发送更新元数据的请求，整个过程没有和集群有任何网络数据交换。</p><p>那这个元数据会在什么时候真正做一次更新呢？我们可以先带着这个问题接着看代码。</p><h2>拉取消息的过程如何实现？</h2><p>接下来，我们分析拉取消息的流程。这个流程的时序图如下（点击图片可放大查看）：</p><p><img src="https://static001.geekbang.org/resource/image/4b/96/4be9e6aa9890e66bc4e26f0c318f8d96.png?wh=996*473" alt=""></p><p>我们对着时序图来分析它的实现流程。在KafkaConsumer.poll()方法(对应源码1179行)的实现里面，可以看到主要是先后调用了2个私有方法：</p><ol>
<li>updateAssignmentMetadataIfNeeded(): 更新元数据。</li>
<li>pollForFetches()：拉取消息。</li>
</ol><p>方法updateAssignmentMetadataIfNeeded()中，调用了coordinator.poll()方法，poll()方法里面又调用了client.ensureFreshMetadata()方法，在client.ensureFreshMetadata()方法中又调用了client.poll()方法，实现了与Cluster通信，在Coordinator上注册Consumer并拉取和更新元数据。至此，“元数据会在什么时候真正做一次更新”这个问题也有了答案。</p><p>类ConsumerNetworkClient封装了Consumer和Cluster之间所有的网络通信的实现，这个类是一个非常彻底的异步实现。它没有维护任何的线程，所有待发送的Request都存放在属性unsent中，返回的Response存放在属性pendingCompletion中。每次调用poll()方法的时候，在当前线程中发送所有待发送的Request，处理所有收到的Response。</p><p>我们在之前的课程中讲到过，这种异步设计的优势就是用很少的线程实现高吞吐量，劣势也非常明显，极大增加了代码的复杂度。对比上节课我们分析的RocketMQ的代码，Producer和Consumer在主要收发消息流程上功能的复杂度是差不多的，但是你可以很明显地感受到Kafka的代码实现要比RocketMQ的代码实现更加的复杂难于理解。</p><p>我们继续分析方法pollForFetches()的实现。</p><pre><code>    private Map&lt;TopicPartition, List&lt;ConsumerRecord&lt;K, V&gt;&gt;&gt; pollForFetches(Timer timer) {
        // 省略部分代码
        // 如果缓存里面有未读取的消息，直接返回这些消息
        final Map&lt;TopicPartition, List&lt;ConsumerRecord&lt;K, V&gt;&gt;&gt; records = fetcher.fetchedRecords();
        if (!records.isEmpty()) {
            return records;
        }
        // 构造拉取消息请求，并发送
        fetcher.sendFetches();
        // 省略部分代码
        // 发送网络请求拉取消息，等待直到有消息返回或者超时
        client.poll(pollTimer, () -&gt; {
            return !fetcher.hasCompletedFetches();
        });
        // 省略部分代码
        // 返回拉到的消息
        return fetcher.fetchedRecords();
    }
</code></pre><p>这段代码的主要实现逻辑是：</p><ol>
<li>如果缓存里面有未读取的消息，直接返回这些消息；</li>
<li>构造拉取消息请求，并发送；</li>
<li>发送网络请求并拉取消息，等待直到有消息返回或者超时；</li>
<li>返回拉到的消息。</li>
</ol><p>在方法fetcher.sendFetches()的实现里面，Kafka根据元数据的信息，构造到所有需要的Broker的拉消息的Request，然后调用client.Send()方法将这些请求异步发送出去。并且，注册了一个回调类来处理返回的Response，所有返回的Response被暂时存放在Fetcher.completedFetches中。需要注意的是，这时的Request并没有被真正发给各个Broker，而是被暂存在了client.unsend中等待被发送。</p><p>然后，在调用client.poll()方法时，会真正将之前构造的所有Request发送出去，并处理收到的Response。</p><p>最后，fetcher.fetchedRecords()方法中，将返回的Response反序列化后转换为消息列表，返回给调用者。</p><p>综合上面的实现分析，我在这里给出整个拉取消息的流程涉及到的相关类的类图，在这个类图中，为了便于你理解，我并没有把所有类都绘制上去，只是把本节课两个流程相关的主要类和这些类里的关键属性画在了图中。你可以配合这个类图和上面的时序图进行代码阅读。</p><p>类图（点击图片可放大查看）：</p><p><img src="https://static001.geekbang.org/resource/image/7b/a2/7b9e26b50308a377c62e741f844c0fa2.png?wh=1215*439" alt=""></p><h2>小结</h2><p>本节课我们一起分析了Kafka Consumer消费消息的实现过程。大家来分析代码过程中，不仅仅是要掌握Kafka整个消费的流程是是如何实现的，更重要的是理解它这种完全异步的设计思想。</p><p>发送请求时，构建Request对象，暂存入发送队列，但不立即发送，而是等待合适的时机批量发送。并且，用回调或者RequestFeuture方式，预先定义好如何处理响应的逻辑。在收到Broker返回的响应之后，也不会立即处理，而是暂存在队列中，择机处理。那这个择机策略就比较复杂了，有可能是需要读取响应的时候，也有可能是缓冲区满了或是时间到了，都有可能触发一次真正的网络请求，也就是在poll()方法中发送所有待发送Request并处理所有Response。</p><p>这种设计的好处是，不需要维护用于异步发送的和处理响应的线程，并且能充分发挥批量处理的优势，这也是Kafka的性能非常好的原因之一。这种设计的缺点也非常的明显，就是实现的复杂度太大了，如果没有深厚的代码功力，很难驾驭这么复杂的设计，并且后续维护的成本也很高。</p><p>总体来说，不推荐大家把代码设计得这么复杂。代码结构简单、清晰、易维护是是我们在设计过程中需要考虑的一个非常重要的因素。很多时候，为了获得较好的代码结构，在可接受的范围内，去牺牲一些性能，也是划算的。</p><h2>思考题</h2><p>我们知道，Kafka Consumer在消费过程中是需要维护消费位置的，Consumer每次从当前消费位置拉取一批消息，这些消息都被正常消费后，Consumer会给Coordinator发一个提交位置的请求，然后消费位置会向后移动，完成一批消费过程。那kafka Consumer是如何维护和提交这个消费位置的呢？请你带着这个问题再回顾一下Consumer的代码，尝试独立分析代码并找到答案。欢迎在留言区与我分享讨论。</p><p>感谢阅读，如果你觉得这篇文章对你有一些启发，也欢迎把它分享给你的朋友。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/30/8a/b5ca7286.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>业余草</span>
  </div>
  <div class="_2_QraFYR_0">我们需要一些速成的实战经验，比如消息队列突然延迟2个小时该如何解决？<br>希望带着这些类似的问题，结合设计原理，帮助我们层层解析。。。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-12 08:33:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/57/f6/2c7ac1ad.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Peter</span>
  </div>
  <div class="_2_QraFYR_0">课后作业，希望老师指正。<br>在基础篇03的时候讲过消费位置是消息队列服务器针对每个消费组和每个队列维护的一个位置变量。那么也就是说最终真正更新这个位置变量应该是交由服务器去执行的，而Consumer只是发送一个请求。那么顺着这个思路，我猜应该是在更新元数据的时候就应该发送这个请求，原因很简单：消费者需要知道“从哪发起”并且“发多少”，因此这时就已经知道了应该将消费位置更新为多少了，所以这时候就可以发送这个请求了。至于服务器最终会将消费位置更新为多少，还取决于客户端返回的结果。<br>在方法updateAssignmentMetadataIfNeeded中，最后一行return updateFetchPositions(timer);<br>从updateFetchPositions这个方法点进去，看到coordinator.refreshCommittedOffsetsIfNeeded(timer)<br>这个方法点进去之后会看到fetchCommittedOffsets方法，进这个方法，找到sendOffsetFetchRequest，点进去，最终会发现  client.send(coordinator, requestBuilder)</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 细致👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-01 11:20:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/d1/29/1b1234ed.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>DFighting</span>
  </div>
  <div class="_2_QraFYR_0">offset的提交不知道是不是在kafkaConsumer.commitAsync中调用coordinator.commitOffsetsAsync(offsets,callback)<br>1、这里设计成异步方式一开始我是比较奇怪的，他是如何保证offset不丢失呢？看了代码才知道在异步返回前会等待ConcurrentLinkQueue&lt;offsetCommitCompletetion&gt;中没有其他的待处理的其他的offset的commit后，才会返回，这里的非阻塞队列是线程安全的，可以避免当前提交冲掉其他的offset的提交<br>2、真正进行提交的时候也不是调用什么具体操作net的接口，而是向另一个ConcurrentLinkedQueue中注册了一个RequestFutureListener的监听者，当然注册之前使用了AtomicInteger来保证并发安全。<br>3、每个监听者应该都会由相应的Coordinator轮询处理队列中的待提交请求，将offset提交从具体的Consumer中解耦到每个组的Coordinator中。<br>当然以上只是个人理解，如有不当欢迎指正。<br>读完这个代码，我发现kafka在这里保证并发数据一致性时，使用了安全的数据结构+CAS的数据访问，灵活且大大降低了锁机制的粗粒度带来的性能损耗，只是这个代码真不容易写好，真是大牛作品！！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-17 09:50:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/fe/14/f1532dec.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>鲁班大师</span>
  </div>
  <div class="_2_QraFYR_0">老师，kafak consumer 在reblance期间，如何实现不重复消费</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 实现“不重复消费”是非常困难的，你需要做的就是让你的consumer具备幂等性，这样即使发生重复消费，也不会对系统数据产生任何影响。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-01 09:28:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/1R7lHGBvwPTVfb3BAQrPX3AhsYWnXyicbUJUYDgWagWxMGTnsNFvKibzeJ8v7fF2vJLQGf2EY9dV07rnn5Mhv9Uw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>山头</span>
  </div>
  <div class="_2_QraFYR_0">老师，你好，broker和消费端都重启了，消费端还知道从哪个offset开始消费吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 从服务端的协调者获取。服务端的协调者会记录主题的每个消费组的每个分区当前的消费位置</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-30 21:43:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/87/51/4155b021.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>于成龙</span>
  </div>
  <div class="_2_QraFYR_0">分析了下acquireAndEnsureOpen如何加锁，供大家参考<br><br>private void acquireAndEnsureOpen() {<br>    acquire();<br>    &#47;&#47;KafkaConsumer成员变量，初始值为false，调用close(Duration)方法后才会置为true<br>    if (this.closed) {<br>        release();<br>        throw new IllegalStateException(&quot;This consumer has already been closed.&quot;);<br>    }<br>}<br><br>&#47;&#47;变量声明<br>private static final long NO_CURRENT_THREAD = -1L;<br><br>private void acquire() {<br>    &#47;&#47;拿到当前线程的线程id<br>    long threadId = Thread.currentThread().getId();<br>    &#47;*if threadId与当前正执行的线程的id不一致（并发，多线程访问）&amp;&amp; threadId对应的线程没有争抢到锁<br>    	 then 抛出异常<br>   	举例：<br>   		现在有两个KafkaConsumer线程，线程id分别是thread1, thread2，要执行acquire()方法。<br>		thread1先启动，执行完上面这条语句、赋值threadId后， thread1栈帧中threadId=thread1，此时CPU线程调度、执行thread2，<br>		thread2也走到if语句时，在thread2的栈帧中，threadId已经赋值为thread2，走到这里，currentThread作为成员变量，初始值为NO_CURRENT_THREAD（-1），因此必然不相等，继续走第二判断条件，即利用AtomicInteger的CAS操作，将当前线程id threadId(thread2)赋值给currentThread这个AtomicInteger，必然返回true，因此会继续执行，使得refcount加1；<br>		接着，此时执行thread1，那么再继续执行if，threadId(thread1) != currentThread.get() (thread2)能满足，但是currentThread的CAS赋值将会失败，因此此时currentThread的值并不是NO_CURRENT_THREAD。<br>		<br>		refcount用于记录重入锁的情况，参见release()方法，当refcount=0时，currentThread将重新赋值为NO_CURRENT_THREAD，保证彻底解锁。<br>    *&#47;<br>    if (threadId != currentThread.get() &amp;&amp; !currentThread.compareAndSet(NO_CURRENT_THREAD, threadId))<br>        throw new ConcurrentModificationException(&quot;KafkaConsumer is not safe for multi-threaded access&quot;);<br>    refcount.incrementAndGet();<br>}</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-10 16:09:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/fa/20/0f06b080.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>凌空飞起的剪刀腿</span>
  </div>
  <div class="_2_QraFYR_0">老师您好：<br>          kafka consumer中没有分析到心跳线程是怎么处理的，我看源代码上写的是单独开了一个后台线程负责心跳，这样处理的优势是什么啊？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Kafka Consumer与服务端的协调者维护心跳，而协调者所在的Broker不一定和接收消息的Broker是同一个实例。<br>所以，必须得分开。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-13 13:47:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/fe/14/f1532dec.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>鲁班大师</span>
  </div>
  <div class="_2_QraFYR_0">每个 ConsumerGroup 都有一个 Coordinator(协调者）负责分配 Consumer 和 Partition 的对应关系，当 Partition 或是 Consumer 发生变更时，会触发 rebalance（重新分配）过程，重新分配 Consumer 与 Partition 的对应关系；……在rebalance期间应该是不能消费的吧</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，消费会暂停，直到Rebalance完成。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-21 08:28:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/85/e9/3854e59a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>SKang</span>
  </div>
  <div class="_2_QraFYR_0">老师 我看完之后 可以理解为 消费组A 消费一个cc主题的消息，然后过程中我将消费组A 的名字改成消费组B后，不会出现重复消费，只会接着A的 继续消费剩下的吧 我认为毕竟A已经成功消费了 偏移量已经成功被更新了吧</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不是的，不同消费组的偏移量是分开记录的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-19 01:08:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/1R7lHGBvwPTVfb3BAQrPX3AhsYWnXyicbUJUYDgWagWxMGTnsNFvKibzeJ8v7fF2vJLQGf2EY9dV07rnn5Mhv9Uw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>山头</span>
  </div>
  <div class="_2_QraFYR_0">消费者如何从服务端拉取消息的，用for循环效率太低吧，能否说说实际的代码</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 同学，我们这节课通篇就是讲得这个问题啊，给你们讲解使用的就是实际的代码。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-13 09:45:52</div>
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
  <div class="_2_QraFYR_0">   先打卡：代码慢慢研究；老师今天讲述了研究代码的目的：<br>   1.消息队列实现的主要流程都一样，掌握流程的实现过程；遇到收发消息的问题，都可以用同样的思路去分析和解决问题。<br>   2.看一下源代码，理清消费生产的实现过程，从中学习一些优秀的设计思路和编码技巧。<br>   这个算是一个小的总结吧：明天中秋休息刚好可以啃代码，好好研究代码去体会。<br>   明天是中秋佳节：愿老师节日快乐^_^</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-12 10:27:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/1c/f6/b5394713.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小杨</span>
  </div>
  <div class="_2_QraFYR_0">我有个疑问，1个topic，3个partition，增加consumer数量能提升消费速度么？或者说kafka应该如何提升消费能力。期待老师解答。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-20 20:31:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/21/03/c5/600fd645.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>tianbingJ</span>
  </div>
  <div class="_2_QraFYR_0">大部分内容都在讨论网络、CAS、异步啊等等知识对于实现一个系统多么重要；但是，各种框架实现的场景都会用到这些内容，总不能所有的课程都先罗列一遍这些内容吧。<br>即使介绍，花个两三节课差不多就行了，占比过大；结果就是跑题了，没有聚焦在MQ上。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-01 11:08:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/fa/64/457325e6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Sam Fu</span>
  </div>
  <div class="_2_QraFYR_0">老师 我最近看了rocketmq消费的源码，您看看我的理解对不对。<br>rocketmq consumer消费完消息后，其实不管成功或失败都会提交这批信息的最大位移。 如果存在失败的消息，则会将整个这一批消息全部发到重试队列去。这样的话，之前消费过的消息就会重复消费了。<br>所以每批拉取的消息设置的不能太大，否则有一条失败整个都得重试，重试率会增高</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-24 13:26:09</div>
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
  <div class="_2_QraFYR_0">请问kafka消费者使用手动提交位移的方式，当前消费进度为10，,然后消费几条失败之后，提交位移失败，后面消费新的消息成功之后，当前消费进度被更新为15，那中间消费失败的几条消息会随着重启消费者而重新消费吗？位移主题里面的消费进度会随着重启消费者而被删除吗?如果不被删除，那应该不会重新消费失败的那几条消息吧？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-13 18:06:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/rHcOA80Xqhe4PJ9S38AzqD2zhrMjK92D7lvH8D3feuHkjiaHTIks5LQvOYjLWZr9mjFklv04jI2Wciahk7x2o8YA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_411c57</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，我想问下: 客户端拉取消息的时候是同步拉取吗？如果是同步拉取的话应该会占用连接吧？为什么不是用异步监听io事件呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-21 09:11:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/2d/47/7c3baa15.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>蛤蟆先生</span>
  </div>
  <div class="_2_QraFYR_0">有个问题请教一下老师，目前我们公司某个应用在生产环境一共有两台机器，这时候有一台机器挂了，但是某个消息还是会经常消费到这台挂的机器上，导致消息没有消费成功，这是为什么呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以贴一下Topic的具体配置，大家一起帮你分析一下原因。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-15 11:08:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/fe/14/f1532dec.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>鲁班大师</span>
  </div>
  <div class="_2_QraFYR_0">多个consumer消费同一个partition会有什么问题么</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果是不同的group.id，互相之间没有影响。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-21 08:26:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/f8/3a/e0c14cb3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lizhibo</span>
  </div>
  <div class="_2_QraFYR_0">老师好，kafka消息要是在消费端消费出现异常了怎么办，他没有再次消费的机制，比如1分之后再去消费，这个怎么实现</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 所以消费的时候，如果不能接受丢消息，一定不要设置成自动提交消费位置。这样下次拉取的时候，还会拉到这个位置的消息。<br><br>ConsumerRecords&lt;String, String&gt; records = consumer.poll(Duration.ofMillis(1000L));<br>&#47;&#47; 执行消费业务逻辑，然后再提交消费位置。<br>consumer.commitSync();<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-13 13:04:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/43/50/abb4ca1e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>凡</span>
  </div>
  <div class="_2_QraFYR_0">提交位置是在ConsumerCoordinator类提供了同步异步提交方法，具体提交位置可以查看到调用这几个方法的位置 </div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-24 23:41:20</div>
  </div>
</div>
</div>
</li>
</ul>