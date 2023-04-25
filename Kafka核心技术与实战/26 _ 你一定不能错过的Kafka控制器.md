<audio title="26 _ 你一定不能错过的Kafka控制器" src="https://static001.geekbang.org/resource/audio/2b/82/2bf32541ec259d9ae17d9999c1f93082.mp3" controls="controls"></audio> 
<p>你好，我是胡夕。今天我要和你分享的主题是：Kafka中的控制器组件。</p><p><strong>控制器组件（Controller），是Apache Kafka的核心组件。它的主要作用是在Apache ZooKeeper的帮助下管理和协调整个Kafka集群</strong>。集群中任意一台Broker都能充当控制器的角色，但是，在运行过程中，只能有一个Broker成为控制器，行使其管理和协调的职责。换句话说，每个正常运转的Kafka集群，在任意时刻都有且只有一个控制器。官网上有个名为activeController的JMX指标，可以帮助我们实时监控控制器的存活状态。这个JMX指标非常关键，你在实际运维操作过程中，一定要实时查看这个指标的值。下面，我们就来详细说说控制器的原理和内部运行机制。</p><p>在开始之前，我先简单介绍一下Apache ZooKeeper框架。要知道，<strong>控制器是重度依赖ZooKeeper的</strong>，因此，我们有必要花一些时间学习下ZooKeeper是做什么的。</p><p><strong>Apache ZooKeeper是一个提供高可靠性的分布式协调服务框架</strong>。它使用的数据模型类似于文件系统的树形结构，根目录也是以“/”开始。该结构上的每个节点被称为znode，用来保存一些元数据协调信息。</p><!-- [[[read_end]]] --><p>如果以znode持久性来划分，<strong>znode可分为持久性znode和临时znode</strong>。持久性znode不会因为ZooKeeper集群重启而消失，而临时znode则与创建该znode的ZooKeeper会话绑定，一旦会话结束，该节点会被自动删除。</p><p>ZooKeeper赋予客户端监控znode变更的能力，即所谓的Watch通知功能。一旦znode节点被创建、删除，子节点数量发生变化，抑或是znode所存的数据本身变更，ZooKeeper会通过节点变更监听器(ChangeHandler)的方式显式通知客户端。</p><p>依托于这些功能，ZooKeeper常被用来实现<strong>集群成员管理、分布式锁、领导者选举</strong>等功能。Kafka控制器大量使用Watch功能实现对集群的协调管理。我们一起来看一张图片，它展示的是Kafka在ZooKeeper中创建的znode分布。你不用了解每个znode的作用，但你可以大致体会下Kafka对ZooKeeper的依赖。</p><p><img src="https://static001.geekbang.org/resource/image/4a/fb/4a2ec3372ff5e4639e5e9c780ec7fcfb.jpg?wh=2250*2896" alt=""></p><p>掌握了ZooKeeper的这些基本知识，现在我们就可以开启对Kafka控制器的讨论了。</p><h2>控制器是如何被选出来的？</h2><p>你一定很想知道，控制器是如何被选出来的呢？我们刚刚在前面说过，每台Broker都能充当控制器，那么，当集群启动后，Kafka怎么确认控制器位于哪台Broker呢？</p><p>实际上，Broker在启动时，会尝试去ZooKeeper中创建/controller节点。Kafka当前选举控制器的规则是：<strong>第一个成功创建/controller节点的Broker会被指定为控制器</strong>。</p><h2>控制器是做什么的？</h2><p>我们经常说，控制器是起协调作用的组件，那么，这里的协调作用到底是指什么呢？我想了一下，控制器的职责大致可以分为5种，我们一起来看看。</p><p>1.<strong>主题管理（创建、删除、增加分区）</strong></p><p>这里的主题管理，就是指控制器帮助我们完成对Kafka主题的创建、删除以及分区增加的操作。换句话说，当我们执行<strong>kafka-topics脚本</strong>时，大部分的后台工作都是控制器来完成的。关于kafka-topics脚本，我会在专栏后面的内容中，详细介绍它的使用方法。</p><p>2.<strong>分区重分配</strong></p><p>分区重分配主要是指，<strong>kafka-reassign-partitions脚本</strong>（关于这个脚本，后面我也会介绍）提供的对已有主题分区进行细粒度的分配功能。这部分功能也是控制器实现的。</p><p>3.<strong>Preferred领导者选举</strong></p><p>Preferred领导者选举主要是Kafka为了避免部分Broker负载过重而提供的一种换Leader的方案。在专栏后面说到工具的时候，我们再详谈Preferred领导者选举，这里你只需要了解这也是控制器的职责范围就可以了。</p><p>4.<strong>集群成员管理（新增Broker、Broker主动关闭、Broker宕机）</strong></p><p>这是控制器提供的第4类功能，包括自动检测新增Broker、Broker主动关闭及被动宕机。这种自动检测是依赖于前面提到的Watch功能和ZooKeeper临时节点组合实现的。</p><p>比如，控制器组件会利用<strong>Watch机制</strong>检查ZooKeeper的/brokers/ids节点下的子节点数量变更。目前，当有新Broker启动后，它会在/brokers下创建专属的znode节点。一旦创建完毕，ZooKeeper会通过Watch机制将消息通知推送给控制器，这样，控制器就能自动地感知到这个变化，进而开启后续的新增Broker作业。</p><p>侦测Broker存活性则是依赖于刚刚提到的另一个机制：<strong>临时节点</strong>。每个Broker启动后，会在/brokers/ids下创建一个临时znode。当Broker宕机或主动关闭后，该Broker与ZooKeeper的会话结束，这个znode会被自动删除。同理，ZooKeeper的Watch机制将这一变更推送给控制器，这样控制器就能知道有Broker关闭或宕机了，从而进行“善后”。</p><p>5.<strong>数据服务</strong></p><p>控制器的最后一大类工作，就是向其他Broker提供数据服务。控制器上保存了最全的集群元数据信息，其他所有Broker会定期接收控制器发来的元数据更新请求，从而更新其内存中的缓存数据。</p><h2>控制器保存了什么数据？</h2><p>接下来，我们就详细看看，控制器中到底保存了哪些数据。我用一张图来说明一下。</p><p><img src="https://static001.geekbang.org/resource/image/21/d4/2174fb81fa7db42122915fee856790d4.jpg?wh=2250*3303" alt=""></p><p>怎么样，图中展示的数据量是不是很多？几乎把我们能想到的所有Kafka集群的数据都囊括进来了。这里面比较重要的数据有：</p><ul>
<li>所有主题信息。包括具体的分区信息，比如领导者副本是谁，ISR集合中有哪些副本等。</li>
<li>所有Broker信息。包括当前都有哪些运行中的Broker，哪些正在关闭中的Broker等。</li>
<li>所有涉及运维任务的分区。包括当前正在进行Preferred领导者选举以及分区重分配的分区列表。</li>
</ul><p>值得注意的是，这些数据其实在ZooKeeper中也保存了一份。每当控制器初始化时，它都会从ZooKeeper上读取对应的元数据并填充到自己的缓存中。有了这些数据，控制器就能对外提供数据服务了。这里的对外主要是指对其他Broker而言，控制器通过向这些Broker发送请求的方式将这些数据同步到其他Broker上。</p><h2>控制器故障转移（Failover）</h2><p>我们在前面强调过，在Kafka集群运行过程中，只能有一台Broker充当控制器的角色，那么这就存在<strong>单点失效</strong>（Single Point of Failure）的风险，Kafka是如何应对单点失效的呢？答案就是，为控制器提供故障转移功能，也就是说所谓的Failover。</p><p><strong>故障转移指的是，当运行中的控制器突然宕机或意外终止时，Kafka能够快速地感知到，并立即启用备用控制器来代替之前失败的控制器</strong>。这个过程就被称为Failover，该过程是自动完成的，无需你手动干预。</p><p>接下来，我们一起来看一张图，它简单地展示了控制器故障转移的过程。</p><p><img src="https://static001.geekbang.org/resource/image/fb/7d/fb9c538a27253fe069ff7ea2f02fa17d.jpg?wh=3507*1989" alt=""></p><p>最开始时，Broker 0是控制器。当Broker 0宕机后，ZooKeeper通过Watch机制感知到并删除了/controller临时节点。之后，所有存活的Broker开始竞选新的控制器身份。Broker 3最终赢得了选举，成功地在ZooKeeper上重建了/controller节点。之后，Broker 3会从ZooKeeper中读取集群元数据信息，并初始化到自己的缓存中。至此，控制器的Failover完成，可以行使正常的工作职责了。</p><h2>控制器内部设计原理</h2><p>在Kafka 0.11版本之前，控制器的设计是相当繁琐的，代码更是有些混乱，这就导致社区中很多控制器方面的Bug都无法修复。控制器是多线程的设计，会在内部创建很多个线程。比如，控制器需要为每个Broker都创建一个对应的Socket连接，然后再创建一个专属的线程，用于向这些Broker发送特定请求。如果集群中的Broker数量很多，那么控制器端需要创建的线程就会很多。另外，控制器连接ZooKeeper的会话，也会创建单独的线程来处理Watch机制的通知回调。除了以上这些线程，控制器还会为主题删除创建额外的I/O线程。</p><p>比起多线程的设计，更糟糕的是，这些线程还会访问共享的控制器缓存数据。我们都知道，多线程访问共享可变数据是维持线程安全最大的难题。为了保护数据安全性，控制器不得不在代码中大量使用<strong>ReentrantLock同步机制</strong>，这就进一步拖慢了整个控制器的处理速度。</p><p>鉴于这些原因，社区于0.11版本重构了控制器的底层设计，最大的改进就是，<strong>把多线程的方案改成了单线程加事件队列的方案</strong>。我直接使用社区的一张图来说明。</p><p><img src="https://static001.geekbang.org/resource/image/90/a3/90be543d426a6a450f360ab40e2734a3.jpg?wh=4160*2155" alt=""></p><p>从这张图中，我们可以看到，社区引入了一个<strong>事件处理线程</strong>，统一处理各种控制器事件，然后控制器将原来执行的操作全部建模成一个个独立的事件，发送到专属的事件队列中，供此线程消费。这就是所谓的单线程+队列的实现方式。</p><p>值得注意的是，这里的单线程不代表之前提到的所有线程都被“干掉”了，控制器只是把缓存状态变更方面的工作委托给了这个线程而已。</p><p>这个方案的最大好处在于，控制器缓存中保存的状态只被一个线程处理，因此不再需要重量级的线程同步机制来维护线程安全，Kafka不用再担心多线程并发访问的问题，非常利于社区定位和诊断控制器的各种问题。事实上，自0.11版本重构控制器代码后，社区关于控制器方面的Bug明显少多了，这也说明了这种方案是有效的。</p><p>针对控制器的第二个改进就是，<strong>将之前同步操作ZooKeeper全部改为异步操作</strong>。ZooKeeper本身的API提供了同步写和异步写两种方式。之前控制器操作ZooKeeper使用的是同步的API，性能很差，集中表现为，<strong>当有大量主题分区发生变更时，ZooKeeper容易成为系统的瓶颈</strong>。新版本Kafka修改了这部分设计，完全摒弃了之前的同步API调用，转而采用异步API写入ZooKeeper，性能有了很大的提升。根据社区的测试，改成异步之后，ZooKeeper写入提升了10倍！</p><p>除了以上这些，社区最近又发布了一个重大的改进！之前Broker对接收的所有请求都是一视同仁的，不会区别对待。这种设计对于控制器发送的请求非常不公平，因为这类请求应该有更高的优先级。</p><p>举个简单的例子，假设我们删除了某个主题，那么控制器就会给该主题所有副本所在的Broker发送一个名为<strong>StopReplica</strong>的请求。如果此时Broker上存有大量积压的Produce请求，那么这个StopReplica请求只能排队等。如果这些Produce请求就是要向该主题发送消息的话，这就显得很讽刺了：主题都要被删除了，处理这些Produce请求还有意义吗？此时最合理的处理顺序应该是，<strong>赋予StopReplica请求更高的优先级，使它能够得到抢占式的处理。</strong></p><p>这在2.2版本之前是做不到的。不过自2.2开始，Kafka正式支持这种不同优先级请求的处理。简单来说，Kafka将控制器发送的请求与普通数据类请求分开，实现了控制器请求单独处理的逻辑。鉴于这个改进还是很新的功能，具体的效果我们就拭目以待吧。</p><h2>小结</h2><p>好了，有关Kafka控制器的内容，我已经讲完了。最后，我再跟你分享一个小窍门。当你觉得控制器组件出现问题时，比如主题无法删除了，或者重分区hang住了，你不用重启Kafka Broker或控制器。有一个简单快速的方式是，去ZooKeeper中手动删除/controller节点。<strong>具体命令是rmr /controller</strong>。这样做的好处是，既可以引发控制器的重选举，又可以避免重启Broker导致的消息处理中断。</p><p><img src="https://static001.geekbang.org/resource/image/a7/07/a77479402c0fddbf7541d26d72a97707.jpg?wh=2069*2580" alt=""></p><h2>开放讨论</h2><p>目前，控制器依然是重度依赖于ZooKeeper的。未来如果要减少对ZooKeeper的依赖，你觉得可能的方向是什么？</p><p>欢迎写下你的思考和答案，我们一起讨论。如果你觉得有所收获，也欢迎把文章分享给你的朋友。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/25/7f/473d5a77.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>曾轼麟</span>
  </div>
  <div class="_2_QraFYR_0">老师控制器选举是不是漏了一个环节，重新选出的controller会增加epoch的值，避免旧的controller复活导致出现两个控制器</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯嗯，是的。fencing机制很重要的，应该要提一下的<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-06 08:52:05</div>
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
  <div class="_2_QraFYR_0">1作用：<br>控制器组件（Controller），是Apache Kafka的核心组件。它的主要作用是Apache Zookeeper的帮助下管理和协调整个Kafka集群。<br>集群中任意一台Broker都能充当控制器的角色，但在运行过程中，只能有一个Broker成为控制器。<br><br>2 特点：控制器是重度依赖Zookeeper。<br>3 产生：<br>	控制器是被选出来的，Broker在启动时，会尝试去Zookeeper中创建&#47;controller节点。Kafka当前选举控制器的规则是：第一个成功创建&#47;controller节点的Broker会被指定为控制器。<br>	<br>4 功能：<br>	A ：主题管理（创建，删除，增加分区）<br>	当执行kafka-topics脚本时，大部分的后台工作都是控制器来完成的。<br>	B ：分区重分配<br>		Kafka-reassign-partitions脚本提供的对已有主题分区进行细粒度的分配功能。<br>	C ：Preferred领导者选举<br>		Preferred领导者选举主要是Kafka为了避免部分Broker负载过重而提供的一种换Leade的方案。<br>		D ：集群成员管理（新增Broker，Broker主动关闭，Broker宕机）<br>		控制器组件会利用watch机制检查Zookeeper的&#47;brokers&#47;ids节点下的子节点数量变更。当有新Broker启动后，它会在&#47;brokers下创建专属的znode节点。一旦创建完毕，Zookeeper会通过Watch机制将消息通知推送给控制器，这样，控制器就能自动地感知到这个变化。进而开启后续新增Broker作业。<br>		侦测Broker存活性则是依赖于刚刚提到的另一个机制：临时节点。每个Broker启动后，会在&#47;brokers&#47;ids下创建一个临时的znode。当Broker宕机或主机关闭后，该Broker与Zookeeper的会话结束，这个znode会被自动删除。同理，Zookeeper的Watch机制将这一变更推送给控制器，这样控制器就能知道有Broker关闭或宕机了，从而进行善后。<br>		<br>E ：数据服务<br>控制器上保存了最全的集群元数据信息，其他所有Broker会定期接收控制器发来的元数据更新请求，从而更新其内存中的缓存数据。<br>	<br>	5 控制器保存的数据<br>	 <br>		<br>	控制器中保存的这些数据在Zookeeper中也保存了一份。每当控制器初始化时，它都会从Zookeeper上读取对应的元数据并填充到自己的缓存中。<br><br>	6 控制器故障转移（Failover）<br>	故障转移是指：当运行中的控制器突然宕机或意外终止时，Kafka能够快速地感知到，并立即启用备用控制器来替代之前失败的控制器。<br><br>	7 内部设计原理<br>		A ：控制器的内部设计相当复杂<br>		控制器是多线程的设计，会在内部创建很多线程。如：<br>（1）为每个Broker创建一个对应的Socket连接，然后在创建一个专属的线程，用于向这些Broker发送特定的请求。<br>		（2）控制连接zookeeper,也会创建单独的线程来处理Watch机制通知回调。<br>		（3）控制器还会为主题删除创建额外的I&#47;O线程。<br>	这些线程还会访问共享的控制器缓存数据，为了维护数据安全性，控制在代码中大量使用ReetrantLock同步机制，进一步拖慢了整个控制器的处理速度。<br><br>		B ：在0.11版对控制器的低沉设计进了重构。<br><br>（1）最大的改进是：把多线程的方案改成了单线程加事件对列的方案。<br>	 	<br>		a. 单线程+队列的实现方式：社区引入了一个事件处理线程，统一处理各种控制器事件，然后控制器将原来执行的操作全部建模成一个个独立的事件，发送到专属的事件队列中，供此线程消费。<br>		b. 单线程不代表之前提到的所有线程都被干掉了，控制器只是把缓存状态变更方面的工作委托给了这个线程而已。<br>（2）第二个改进：将之前同步操作Zookeeper全部改为异步操作。<br>			a.  Zookeeper本身的API提供了同步写和异步写两种方式。同步操作zk，在有大量主题分区发生变更时，Zookeeper容易成为系统的瓶颈。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-10 17:37:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/1e/3a/5b21c01c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>nightmare</span>
  </div>
  <div class="_2_QraFYR_0">类似rocket mq写一个name server的注册模块出来，代替zookeeper   ，从而实现 控制器选举 ，元数据共享，还有broker信息注册等功能</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-01 00:57:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/8d/56/2a04dd88.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>icejoywoo</span>
  </div>
  <div class="_2_QraFYR_0">基于raft搞一套来替代zookeeper？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，做一组controller，基于Raft算法组成quorum</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-09 19:07:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/diaGeaLuw3oTAcyFMnkiaVur33RXdUZL8z1LtfHibIyh4r629YSexJQz5JXYjBH7v9rwHH7ham5CzqDZF75QnGIwg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>野性力量</span>
  </div>
  <div class="_2_QraFYR_0">epoch这个词我经常看到，查了是纪元的意思，不过用在这里应该怎么理解呢。（在图里）</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 暂时可以理解成版本</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-01 09:59:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/31/9d/daad92d2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Stony.修行僧</span>
  </div>
  <div class="_2_QraFYR_0">KIP-500: Replace ZooKeeper with a Self-Managed Metadata Quorum<br>了解一下</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，看到这个KIP了，最近很火，有人还翻译出来了。事实上这个KIP只是在讨论阶段，目前还没有被accept</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-02 18:57:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/9c/35/9dc79371.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>你好旅行者</span>
  </div>
  <div class="_2_QraFYR_0">老师好，关于线程的优化，我能否这样理解:之前是为每一个事件分配一个线程，线程本身的切换以及锁会带来繁重的开销。在后续的版本中，讲请求封装成了一个个的事件，采用异步串行化的方式，放入到队列中，由统一的一个线程来轮询这个队列，从而避免了锁的开销。不知道这样的理解是否准确？<br><br>此外，老师说的多个线程之间共享Broker缓内存区域，可否举个例子，在什么情况下他们需要共享内存区域呢？<br><br>谢谢老师！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Controller有个context，里面缓存了很多数据。以前的设计是多个线程会同时访问这些数据，比如topic删除线程、controller线程等。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-01 13:22:57</div>
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
  <div class="_2_QraFYR_0">老师，请问：<br>1. 当控制器发生故障时，其他broker是如何感知到的？是ZK watch controller没有节点，然后广播通知其他的broker, 来争抢新建一个控制器节点吗？ 还是其他broker没定时收到控制器发送的元数据同步请求？<br>2. ZK不是可以确保节点的唯一性，为什么还会出现控制器大于1的情况？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. Controller所在broker发生故障，ZooKeeper上的&#47;controller节点会自动消失。其他broker监控这个节点的存在，因此会第一时间感知到<br>2. 极少数情况下，不排除出现脑裂的情形，比如出现network partitioning，Zookeeper ensemble被分割成两个，的确有可能出现两个controller</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-02 16:50:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/0d/ac/09678490.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>谢特</span>
  </div>
  <div class="_2_QraFYR_0">多个节点之间内存一般怎么共享</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 一般不共享内存，甚至什么都不共享， 这就是所谓的Shard-Nothing架构</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-10 23:13:28</div>
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
  <div class="_2_QraFYR_0">我也想知道rocketmq的name server和用zk的区别和优劣势？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-01 22:08:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/75/a8/dfe4cade.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>电光火石</span>
  </div>
  <div class="_2_QraFYR_0">老师好，想问一下：<br>1.如何看出重分区被hang住了，是长时间没有响应就被hang住，还是有一些jmx的参数可以观察？<br>2.当我们删除&#47;controller的时候，是否会有数据丢失的可能，比如重分区的请求，是否会先存储在zk，然后controler从中读取请求进行处理，这个时候发生重failover，是否新的controller会重新读到这个请求？<br>谢谢了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. 执行reassign命令总提示分区在reassign中，或者ZooKeeper中的&#47;admin&#47;reassign_partitions下相应节点未被删除<br>2. 不会丢失数据，如果真丢失了，果断开jira，因为这是一个严重的bug：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-07 11:25:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/63/c6/0167415c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>林肯</span>
  </div>
  <div class="_2_QraFYR_0">各个broker之间怎样保证元数据的一致性？controller挂了后重新选举的机制是怎样的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 异步发送元数据来保持一致性。最权威的数据保存在Zk上。当controller挂掉之后，Zk上的临时节点&#47;controller消失，所有存活broker都会感知到这一变化，于是抢注&#47;controller，谁抢上谁就是新的controller</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-11 19:49:51</div>
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
  <div class="_2_QraFYR_0">老师，若给kafka-server配置文件中的zookeeper.connect参数写了3个zk的节点，他们是如何交换？kafka-server先尝试和第一个zk节点开始建立tcp socket连接，若成功，那么后面两个就不用建立连接, 待异常情况下，做备用吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 严格来说，Kafka是把这个事情交由ZooKeeper来完成的。如果你指定了多台，那么ZooKeeper会把Kafka的建立连接请求发送给ZooKeeper的leader节点，其他standby<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-09 17:52:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/6b/1b/4b397b80.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>云师兄</span>
  </div>
  <div class="_2_QraFYR_0">社区改造单线程加队列的方式，有没有数据或者图标对比下改造前后性能的差异？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 据我所知没有性能方面的报告出来。改进最大的收益来自于可理解性和可维护性方面的提升</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-24 08:39:40</div>
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
  <div class="_2_QraFYR_0">修改主题分区的broker_host是随便指定一个吗？最后由zk通知到控制器？<br>作者: 其实如果指定的是broker host，后面是不走ZooKeeper的<br><br>追问：<br>前面文章提到增加分区是控制器完成的。  <br>那如果随机在一个broker上执行增加分区的命令，再由这个broker通知控制器去做吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 客户端会先去找controller所在节点，然后直接给它发送请求</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-08 20:25:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/9b/a8/6a391c66.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Leon📷</span>
  </div>
  <div class="_2_QraFYR_0">脑裂问题希望详细说说</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-01 09:21:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/86/fa/4bcd7365.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>玉剑冰锋</span>
  </div>
  <div class="_2_QraFYR_0">如何区分临时znode和永久znode？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: znode的ephemeralOwner不为0的就是临时节点</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-01 08:12:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_03</span>
  </div>
  <div class="_2_QraFYR_0">老师，consumer频繁打印Marking the coorinator dead for group日志是什么原因呢？网上很多答案都是说要配置host，但是我们这边consumer端的服务器上已经配置了host，而且感觉这个信息的打印跟consumer的拉取速度有很大关系</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 负载大的时候才抛这个错误还是一开始就抛。如果是后者，那么连不上Coordinator的可能性很大，如果是前者，可能和Rebalance的设置有关，比如max.poll.interval.ms，max.poll.records是否合理</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-08 14:09:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/d2/c7/14d235bb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>猕猴桃 盛哥</span>
  </div>
  <div class="_2_QraFYR_0">为何在2.2.1版本的zk的controller和consumer节点下没有任何数据和子节点呢？(单机环境)</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: &#47;consumer已经被弃用了。&#47;controller没有说明你的Kafka集群没起来。一个正常启动的Kafka集群必然有个Broker充当Controller，去抢注&#47;controller节点</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-25 03:00:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/06/ac/604de2e8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张庆</span>
  </div>
  <div class="_2_QraFYR_0">胡夕大拿，您好，zookeeper中保存了一份kafka的元数据信息，控制器中也会保存一份。当控制器初始化的时候会从zookeeper中拉一份数据，那么之后zookeeper和controller中的数据怎么保持一致啊？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 每次变更都会更新zookeeper</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-30 10:29:42</div>
  </div>
</div>
</div>
</li>
</ul>