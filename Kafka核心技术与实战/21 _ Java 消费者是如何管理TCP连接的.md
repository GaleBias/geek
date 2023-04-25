<audio title="21 _ Java 消费者是如何管理TCP连接的" src="https://static001.geekbang.org/resource/audio/24/96/2481b5d4d26b94ef7b6ba91dc3ac7b96.mp3" controls="controls"></audio> 
<p>你好，我是胡夕。今天我要和你分享的主题是：Kafka的Java消费者是如何管理TCP连接的。</p><p>在专栏<a href="https://time.geekbang.org/column/article/103844">第13讲</a>中，我们专门聊过“Java<strong>生产者</strong>是如何管理TCP连接资源的”这个话题，你应该还有印象吧？今天算是它的姊妹篇，我们一起来研究下Kafka的Java<strong>消费者</strong>管理TCP或Socket资源的机制。只有完成了今天的讨论，我们才算是对Kafka客户端的TCP连接管理机制有了全面的了解。</p><p>和之前一样，我今天会无差别地混用TCP和Socket两个术语。毕竟，在Kafka的世界中，无论是ServerSocket，还是SocketChannel，它们实现的都是TCP协议。或者这么说，Kafka的网络传输是基于TCP协议的，而不是基于UDP协议，因此，当我今天说到TCP连接或Socket资源时，我指的是同一个东西。</p><h2>何时创建TCP连接？</h2><p>我们先从消费者创建TCP连接开始讨论。消费者端主要的程序入口是KafkaConsumer类。<strong>和生产者不同的是，构建KafkaConsumer实例时是不会创建任何TCP连接的</strong>，也就是说，当你执行完new KafkaConsumer(properties)语句后，你会发现，没有Socket连接被创建出来。这一点和Java生产者是有区别的，主要原因就是生产者入口类KafkaProducer在构建实例的时候，会在后台默默地启动一个Sender线程，这个Sender线程负责Socket连接的创建。</p><!-- [[[read_end]]] --><p>从这一点上来看，我个人认为KafkaConsumer的设计比KafkaProducer要好。就像我在第13讲中所说的，在Java构造函数中启动线程，会造成this指针的逃逸，这始终是一个隐患。</p><p>如果Socket不是在构造函数中创建的，那么是在KafkaConsumer.subscribe或KafkaConsumer.assign方法中创建的吗？严格来说也不是。我还是直接给出答案吧：<strong>TCP连接是在调用KafkaConsumer.poll方法时被创建的</strong>。再细粒度地说，在poll方法内部有3个时机可以创建TCP连接。</p><p>1.<strong>发起FindCoordinator请求时</strong>。</p><p>还记得消费者端有个组件叫协调者（Coordinator）吗？它驻留在Broker端的内存中，负责消费者组的组成员管理和各个消费者的位移提交管理。当消费者程序首次启动调用poll方法时，它需要向Kafka集群发送一个名为FindCoordinator的请求，希望Kafka集群告诉它哪个Broker是管理它的协调者。</p><p>不过，消费者应该向哪个Broker发送这类请求呢？理论上任何一个Broker都能回答这个问题，也就是说消费者可以发送FindCoordinator请求给集群中的任意服务器。在这个问题上，社区做了一点点优化：消费者程序会向集群中当前负载最小的那台Broker发送请求。负载是如何评估的呢？其实很简单，就是看消费者连接的所有Broker中，谁的待发送请求最少。当然了，这种评估显然是消费者端的单向评估，并非是站在全局角度，因此有的时候也不一定是最优解。不过这不并影响我们的讨论。总之，在这一步，消费者会创建一个Socket连接。</p><p>2.<strong>连接协调者时。</strong></p><p>Broker处理完上一步发送的FindCoordinator请求之后，会返还对应的响应结果（Response），显式地告诉消费者哪个Broker是真正的协调者，因此在这一步，消费者知晓了真正的协调者后，会创建连向该Broker的Socket连接。只有成功连入协调者，协调者才能开启正常的组协调操作，比如加入组、等待组分配方案、心跳请求处理、位移获取、位移提交等。</p><p>3.<strong>消费数据时。</strong></p><p>消费者会为每个要消费的分区创建与该分区领导者副本所在Broker连接的TCP。举个例子，假设消费者要消费5个分区的数据，这5个分区各自的领导者副本分布在4台Broker上，那么该消费者在消费时会创建与这4台Broker的Socket连接。</p><h2>创建多少个TCP连接？</h2><p>下面我们来说说消费者创建TCP连接的数量。你可以先思考一下大致需要的连接数量，然后我们结合具体的Kafka日志，来验证下结果是否和你想的一致。</p><p>我们来看看这段日志。</p><blockquote>
<p><em>[2019-05-27 10:00:54,142] DEBUG [Consumer clientId=consumer-1, groupId=test] Initiating connection to node <span class="orange">localhost:9092</span> (id: -1 rack: null) using address localhost/127.0.0.1 (org.apache.kafka.clients.NetworkClient:944)</em></p>
</blockquote><blockquote>
<p><em>......</em></p>
</blockquote><blockquote>
<p><em>[2019-05-27 10:00:54,188] DEBUG [Consumer clientId=consumer-1, groupId=test] <span class="orange">Sending metadata request</span>  MetadataRequestData(topics=[MetadataRequestTopic(name='t4')], allowAutoTopicCreation=true, includeClusterAuthorizedOperations=false, includeTopicAuthorizedOperations=false) to node <span class="orange">localhost:9092</span> (id: -1 rack: null) (org.apache.kafka.clients.NetworkClient:1097)</em></p>
</blockquote><blockquote>
<p><em>......</em></p>
</blockquote><blockquote>
<p><em>[2019-05-27 10:00:54,188] TRACE [Consumer clientId=consumer-1, groupId=test] <span class="orange">Sending FIND_COORDINATOR</span> {key=test,key_type=0} with correlation id 0 to node -1 (org.apache.kafka.clients.NetworkClient:496)</em></p>
</blockquote><blockquote>
<p><em>[2019-05-27 10:00:54,203] TRACE [Consumer clientId=consumer-1, groupId=test] Completed receive from node -1 for FIND_COORDINATOR with correlation id 0, received {throttle_time_ms=0,error_code=0,error_message=null, <span class="orange">node_id=2,host=localhost,port=9094</span>} (org.apache.kafka.clients.NetworkClient:837)</em></p>
</blockquote><blockquote>
<p><em>......</em></p>
</blockquote><blockquote>
<p><em>[2019-05-27 10:00:54,204] DEBUG [Consumer clientId=consumer-1, groupId=test] Initiating connection to node <span class="orange">localhost:9094</span> (id: 2147483645 rack: null) using address localhost/127.0.0.1 (org.apache.kafka.clients.NetworkClient:944)</em></p>
</blockquote><blockquote>
<p><em>......</em></p>
</blockquote><blockquote>
<p><em>[2019-05-27 10:00:54,237] DEBUG [Consumer clientId=consumer-1, groupId=test] Initiating connection to node <span class="orange">localhost:9094</span> (id: 2 rack: null) using address localhost/127.0.0.1 (org.apache.kafka.clients.NetworkClient:944)</em></p>
</blockquote><blockquote>
<p><em>[2019-05-27 10:00:54,237] DEBUG [Consumer clientId=consumer-1, groupId=test] Initiating connection to node<span class="orange"> localhost:9092</span> (id: 0 rack: null) using address localhost/127.0.0.1 (org.apache.kafka.clients.NetworkClient:944)</em></p>
</blockquote><blockquote>
<p><em>[2019-05-27 10:00:54,238] DEBUG [Consumer clientId=consumer-1, groupId=test] Initiating connection to node <span class="orange">localhost:9093</span> (id: 1 rack: null) using address localhost/127.0.0.1 (org.apache.kafka.clients.NetworkClient:944)</em></p>
</blockquote><p>这里我稍微解释一下，日志的第一行是消费者程序创建的第一个TCP连接，就像我们前面说的，这个Socket用于发送FindCoordinator请求。由于这是消费者程序创建的第一个连接，此时消费者对于要连接的Kafka集群一无所知，因此它连接的Broker节点的ID是-1，表示消费者根本不知道要连接的Kafka Broker的任何信息。</p><p>值得注意的是日志的第二行，消费者复用了刚才创建的那个Socket连接，向Kafka集群发送元数据请求以获取整个集群的信息。</p><p>日志的第三行表明，消费者程序开始发送FindCoordinator请求给第一步中连接的Broker，即localhost:9092，也就是nodeId等于-1的那个。在十几毫秒之后，消费者程序成功地获悉协调者所在的Broker信息，也就是第四行标为橙色的“node_id = 2”。</p><p>完成这些之后，消费者就已经知道协调者Broker的连接信息了，因此在日志的第五行发起了第二个Socket连接，创建了连向localhost:9094的TCP。只有连接了协调者，消费者进程才能正常地开启消费者组的各种功能以及后续的消息消费。</p><p>在日志的最后三行中，消费者又分别创建了新的TCP连接，主要用于实际的消息获取。还记得我刚才说的吗？要消费的分区的领导者副本在哪台Broker上，消费者就要创建连向哪台Broker的TCP。在我举的这个例子中，localhost:9092，localhost:9093和localhost:9094这3台Broker上都有要消费的分区，因此消费者创建了3个TCP连接。</p><p>看完这段日志，你应该会发现日志中的这些Broker节点的ID在不断变化。有时候是-1，有时候是2147483645，只有在最后的时候才回归正常值0、1和2。这又是怎么回事呢？</p><p>前面我们说过了-1的来由，即消费者程序（其实也不光是消费者，生产者也是这样的机制）首次启动时，对Kafka集群一无所知，因此用-1来表示尚未获取到Broker数据。</p><p>那么2147483645是怎么来的呢？它是<strong>由Integer.MAX_VALUE减去协调者所在Broker的真实ID计算得来的</strong>。看第四行标为橙色的内容，我们可以知道协调者ID是2，因此这个Socket连接的节点ID就是Integer.MAX_VALUE减去2，即2147483647减去2，也就是2147483645。这种节点ID的标记方式是Kafka社区特意为之的结果，目的就是要让组协调请求和真正的数据获取请求使用不同的Socket连接。</p><p>至于后面的0、1、2，那就很好解释了。它们表征了真实的Broker ID，也就是我们在server.properties中配置的broker.id值。</p><p>我们来简单总结一下上面的内容。通常来说，消费者程序会创建3类TCP连接：</p><ol>
<li>确定协调者和获取集群元数据。</li>
<li>连接协调者，令其执行组成员管理操作。</li>
<li>执行实际的消息获取。</li>
</ol><p>那么，这三类TCP请求的生命周期都是相同的吗？换句话说，这些TCP连接是何时被关闭的呢？</p><h2>何时关闭TCP连接？</h2><p>和生产者类似，消费者关闭Socket也分为主动关闭和Kafka自动关闭。主动关闭是指你显式地调用消费者API的方法去关闭消费者，具体方式就是<strong>手动调用KafkaConsumer.close()方法，或者是执行Kill命令</strong>，不论是Kill -2还是Kill -9；而Kafka自动关闭是由<strong>消费者端参数connection.max.idle.ms</strong>控制的，该参数现在的默认值是9分钟，即如果某个Socket连接上连续9分钟都没有任何请求“过境”的话，那么消费者会强行“杀掉”这个Socket连接。</p><p>不过，和生产者有些不同的是，如果在编写消费者程序时，你使用了循环的方式来调用poll方法消费消息，那么上面提到的所有请求都会被定期发送到Broker，因此这些Socket连接上总是能保证有请求在发送，从而也就实现了“长连接”的效果。</p><p>针对上面提到的三类TCP连接，你需要注意的是，<strong>当第三类TCP连接成功创建后，消费者程序就会废弃第一类TCP连接</strong>，之后在定期请求元数据时，它会改为使用第三类TCP连接。也就是说，最终你会发现，第一类TCP连接会在后台被默默地关闭掉。对一个运行了一段时间的消费者程序来说，只会有后面两类TCP连接存在。</p><h2>可能的问题</h2><p>从理论上说，Kafka Java消费者管理TCP资源的机制我已经说清楚了，但如果仔细推敲这里面的设计原理，还是会发现一些问题。</p><p>我们刚刚讲过，第一类TCP连接仅仅是为了首次获取元数据而创建的，后面就会被废弃掉。最根本的原因是，消费者在启动时还不知道Kafka集群的信息，只能使用一个“假”的ID去注册，即使消费者获取了真实的Broker ID，它依旧无法区分这个“假”ID对应的是哪台Broker，因此也就无法重用这个Socket连接，只能再重新创建一个新的连接。</p><p>为什么会出现这种情况呢？主要是因为目前Kafka仅仅使用ID这一个维度的数据来表征Socket连接信息。这点信息明显不足以确定连接的是哪台Broker，也许在未来，社区应该考虑使用<strong>&lt;主机名、端口、ID&gt;</strong>三元组的方式来定位Socket资源，这样或许能够让消费者程序少创建一些TCP连接。</p><p>也许你会问，反正Kafka有定时关闭机制，这算多大点事呢？其实，在实际场景中，我见过很多将connection.max.idle.ms设置成-1，即禁用定时关闭的案例，如果是这样的话，这些TCP连接将不会被定期清除，只会成为永久的“僵尸”连接。基于这个原因，社区应该考虑更好的解决方案。</p><h2>小结</h2><p>好了，今天我们补齐了Kafka Java客户端管理TCP连接的“拼图”。我们不仅详细描述了Java消费者是怎么创建和关闭TCP连接的，还对目前的设计方案提出了一些自己的思考。希望今后你能将这些知识应用到自己的业务场景中，并对实际生产环境中的Socket管理做到心中有数。</p><p><img src="https://static001.geekbang.org/resource/image/f1/04/f13d7008d7b251df0e6e6a89077d7604.jpg?wh=2069*2535" alt=""></p><h2>开放讨论</h2><p>假设有个Kafka集群由2台Broker组成，有个主题有5个分区，当一个消费该主题的消费者程序启动时，你认为该程序会创建多少个Socket连接？为什么？</p><p>欢迎写下你的思考和答案，我们一起讨论。如果你觉得有所收获，也欢迎把文章分享给你的朋友。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/5f/e9/95ef44f3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>常超</span>
  </div>
  <div class="_2_QraFYR_0">整个生命周期里会建立4个连接，进入稳定的消费过程后，同时保持3个连接，以下是详细。<br>第一类连接：确定协调者和获取集群元数据。 <br> 一个，初期的时候建立，当第三类连接建立起来之后，这个连接会被关闭。<br><br>第二类连接：连接协调者，令其执行组成员管理操作。<br> 一个<br><br>第三类连接：执行实际的消息获取。<br>两个分别会跟两台broker机器建立一个连接，总共两个TCP连接，同一个broker机器的不同分区可以复用一个socket。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-20 06:42:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/fd/fd/326be9bb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>注定非凡</span>
  </div>
  <div class="_2_QraFYR_0">1，何时创建<br>	A ：消费者和生产者不同，在创建KafkaConsumer实例时不会创建任何TCP连接。<br>		原因：是因为生产者入口类KafkaProducer在构建实例时，会在后台启动一个Sender线程，这个线程是负责Socket连接创建的。<br><br>	B ：TCP连接是在调用KafkaConsumer.poll方法时被创建。在poll方法内部有3个时机创建TCP连接<br>	（1）发起findCoordinator请求时创建<br>		Coordinator（协调者）消费者端主键，驻留在Broker端的内存中，负责消费者组的组成员管理和各个消费者的位移提交管理。<br>		当消费者程序首次启动调用poll方法时，它需要向Kafka集群发送一个名为FindCoordinator的请求，确认哪个Broker是管理它的协调者。<br><br>	（2）连接协调者时<br>		Broker处理了消费者发来的FindCoordinator请求后，返回响应显式的告诉消费者哪个Broker是真正的协调者。<br>		当消费者知晓真正的协调者后，会创建连向该Broker的socket连接。<br>		只有成功连入协调者，协调者才能开启正常的组协调操作。<br><br>	（3）消费数据时<br>		消费者会为每个要消费的分区创建与该分区领导者副本所在的Broker连接的TCP.<br><br>2 创建多少<br>	消费者程序会创建3类TCP连接：<br>	（1） ：确定协调者和获取集群元数据<br>	（2）：连接协调者，令其执行组成员管理操作<br>	（3） ：执行实际的消息获取<br><br>3 何时关闭TCP连接<br>	A ：和生产者相似，消费者关闭Socket也分为主动关闭和Kafka自动关闭。<br>	B ：主动关闭指通过KafkaConsumer.close()方法，或者执行kill命令，显示地调用消费者API的方法去关闭消费者。<br>	C ：自动关闭指消费者端参数connection.max.idle.ms控制的，默认为9分钟，即如果某个socket连接上连续9分钟都没有任何请求通过，那么消费者会强行杀死这个连接。<br>	D ：若消费者程序中使用了循环的方式来调用poll方法消息消息，以上的请求都会被定期的发送到Broker，所以这些socket连接上总是能保证有请求在发送，从而实现“长连接”的效果。<br>	E ：当第三类TCP连接成功创建后，消费者程序就会废弃第一类TCP连接，之后在定期请求元数据时，会改为使用第三类TCP连接。对于一个运行了一段时间的消费者程序来讲，只会有后面两种的TCP连接。<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-06 08:33:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/3b/aa/e8dfcd7e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>AAA_叶子</span>
  </div>
  <div class="_2_QraFYR_0">消费者tcp连接一旦断开，就会导致rebalance，实际开发过程中，是不是需要尽量保证长连接的模式？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，如果就是要长时间的消费，维持一个长连接是不错的选择</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-18 11:20:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/b8/5b/dbe74486.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>taj3991</span>
  </div>
  <div class="_2_QraFYR_0">老师同一个消费组的客户端都只会连接到一个协调者吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的。每个group都有一个与之对应的coordinator</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-22 21:13:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/87/57/e28ba87b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Williamzhang</span>
  </div>
  <div class="_2_QraFYR_0">我觉得作者可以跟学员的留言互动，然后每期课后思考可以在下期中贴出答案及分析，其实留言讨论也是一个非常让人有收获的地方</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-22 09:05:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83eol8MiawYVfCtkaFL9DFGoWpuajsKicwyt7IWm07JfrLMDuksEZJqia4Rbicw0biayokhgvSK0rUXIAngQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_ab3d9a</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，请问k8s这样的容器平台，适合部署kafka的消费者吗？如果容器平台起了二个一模一样的消费者，对kafka来说会不会不知道自己通信的哪一个消费者?  kafka通过什么来判断不同的客户端?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 每个客户端至少主机名和端口是不一样的，我是指在TCP连接这个层面。另外对于producer而言，其实Kafka也不用区分它们，反正知道它们都再向集群发送消息就行了。对于consumer而言，主要还是看group.id的设置以确定它们是否在同一个group。最后如果是你想区分客户端，那么可以设置不同的client.id</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-17 12:04:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/bb/c9/37924ad4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>天天向上</span>
  </div>
  <div class="_2_QraFYR_0">元数据不包含协调者信息吗？为啥还要再请求一次协调者信息 什么设计思路？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不包括，因为你请求元数据的broker可能不是Coordinator，没有Coordinator的信息</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-19 09:26:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/e5/39/951f89c8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>信信</span>
  </div>
  <div class="_2_QraFYR_0">一共建过四次连接。若connection.max.idle.ms 不为-1，最终会断开第一次连的ID为-1的连接。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-24 00:47:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/5c/d7/e4673fde.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>October</span>
  </div>
  <div class="_2_QraFYR_0">总共创建4个连接，最终保持3个连接：<br>        确定消费者所属的消费组对应的GroupCoordinator和获取集群的metadata时创建一个TCP连接，由于此时的node id = -1，所以该连接无法重用。<br>        连接GroupCoordinator时，创建第二个TCP连接，node id值为Integer.MAX_VALUE-id<br>        消费者会与每个分区的leader创建一个TCP连接来消费数据，node id为broker.id，由于kafka只是用id这一维度来表征Socket连接信息，因此如果多个分区的leader在同一个broker上时，会共用一个TCP连接，由于分区数大于broker的数量，所以会创建两个TCP连接消费数据。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-21 10:57:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83eqxnw92EOgZbyHDGMZ1d1OFDjjJKnBdmpiac8J7kBEN5h3AvvzU85mo5Chj8pkIHQc390dL1mu0neQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>真锅</span>
  </div>
  <div class="_2_QraFYR_0">意思是即便知道了协调者在node 2上，还是会依然用2147483645这个id的TCP连接去跟协调者通信吗。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个数字不是固定的，而是用MAX - broker ID算出来的。对Coordinator的连接来说，是的！ 它的ID就是这个算法</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-25 22:43:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/x86UN2kFbJGGwiaw7yeVtyaf05y5eZmdOciaAF09WEBRVicbPGsej1b62UAH3icjeJqvibVc6aqB0EuFwDicicKKcF47w/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Eco</span>
  </div>
  <div class="_2_QraFYR_0">应该是3个tcp连接，第一个id=-1的没什么争议，然后是连接协调者的，但是broker，5个分区的leader肯定会分布到这两台broker上，那么第三类tcp就是2个tcp连接，但是这2个中完全可以有一个是直接使用连接协调者的那个tcp连接吧，但老师好像说过连接协调者的连接会和传输数据的分开，id的计算都不相同，好吧，那就4个tcp连接吧。可这里真的不能复用吗？我觉得可以。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 目前与Coordinator和普通数据交互的TCP连接的确是分开的，你要说是否能复用，我觉得当然可以复用，只不过现在没有这么设计：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-19 23:35:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/1b/7a/390a8530.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小木匠</span>
  </div>
  <div class="_2_QraFYR_0">“负载是如何评估的呢？其实很简单，就是看消费者连接的所有 Broker 中，谁的待发送请求最少。”  <br>老师这个没太明白，这时候消费者不是还没连接么？那这部分信息是从哪获取到的呢？消费者本地吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 刚开始的时候当然就类似于随机选broker了，但后面慢慢积累了一些数据之后这个小优化还是会起一些作用的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-22 07:43:51</div>
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
  <div class="_2_QraFYR_0">老师我有个疑问，consumer在FindCoordinator的时候会选择负载最小的broker进行连接，文章说看消费者连接的所有 Broker 中，谁的待发送请求最少。请问consumer如何得知这个消息？如果它想知道这个消息，不就得先和“某个东西”建立连接了？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 它首先倾向于使用已建立连接的节点。如果已建立连接的节点都在使用中，可能会创建新的TCP连接</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-20 18:16:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83eqKurYDna034zK0ibDIWicybibQhiaM6afgla2zVFqBrAemLgCh3WoibjACnic5ibiaYMd29tMB26cjaGaPoA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Treagzhao</span>
  </div>
  <div class="_2_QraFYR_0">老师，“消费者程序会向集群中当前负载最小的那台 Broker 发送请求”，消费者怎么单方面知道服务器待发送的消息数量呢？而且应该只有leader才会实际发送消息吧，follower待发送的都是0,消费者怎么在建立连接之前就知道服务器的角色呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是这样判断的，就是看消费者与broker的TCP连接上的待处理请求的个数</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-19 16:15:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/64/34/03335c4a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>臧萌</span>
  </div>
  <div class="_2_QraFYR_0">我们要设计一个消息系统。有两个选择，更好的一种是每种不同schema的消息发一个topic。但是有一种担心是consumer会为每个topic建立一个连接，造成连接数太多。请问胡老师，kafka client的consumer是每个集群固定数目的tcp连接，还是和topic数目相关？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 和它要订阅的topic分区数以及这些分区在broker上的散列情况有关。比如你订阅了100个分区，但这个100个分区的leader副本都在一个broker上，那么长期来看consumer也就只和这1个broker建立连接；相反如果这100分区散列在100个broker上，那么长期来看consumer会和100个broker维持长连接</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-29 18:17:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/57/3c/081b89ec.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>rm -rf 😊ི</span>
  </div>
  <div class="_2_QraFYR_0">我认为也是3个连接，第一个是查找Coordinator的，这个会在后面断开。然后5个partition会分布在2个broker上，那么客户端最多也就连接2次就能消费所有partition了，因此是连接3个，最后保持2个。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-21 17:08:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/b5/3c/967d7291.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>艺超(鲁鸣)</span>
  </div>
  <div class="_2_QraFYR_0">老师好，请教一个问题，现在对于producer和consumer都介绍了维持tcp连接的情况，那么对于kafka集群 broker来说，这么多的tcp连接，是如何管理的呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 其实也没什么管理。如果一定要说管理，通过KafkaChannel对象来管理。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-29 10:23:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/d3/43/f22b39c3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>举个荔枝</span>
  </div>
  <div class="_2_QraFYR_0">老师，想问下这里是不是笔误。<br>还记得消费者端有个组件叫Coordinator吗？协调者应该是位于Broker端的吧？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯嗯，其实这里的消费者端指的是广义的消费者，我是想说在Kafka消费者的概念中有Coordinator。当然如你所说Coordinator是Broker端的组件没错。这里的确有不严谨的地方，多谢指出：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-02 16:28:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/4f/60/049a20e9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>吴宇晨</span>
  </div>
  <div class="_2_QraFYR_0">一个获取元数据的连接（之后会断开）+两个连接分区leader的连接+一个连接协调者的连接</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-22 10:23:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/4d/49/28e73b9c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>明翼</span>
  </div>
  <div class="_2_QraFYR_0">连接有三个阶段：首先获取协调者连接同时也获取元数据信息，这个连接后面会关闭；连接协调者执行，等待分配分区，组协调等，这需要一个连接；后面真正消费五个分区两个broker最多就两个连接，分区大于broker所以一定是两个，因为第一类连接没有id，所以无法重用，会在第三类开启连接后关闭，所以开始四个连接最终保持三个连接</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-20 09:29:04</div>
  </div>
</div>
</div>
</li>
</ul>