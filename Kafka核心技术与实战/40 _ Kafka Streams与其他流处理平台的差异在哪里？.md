<audio title="40 _ Kafka Streams与其他流处理平台的差异在哪里？" src="https://static001.geekbang.org/resource/audio/2f/53/2f381f477db320b11f23164dfc904f53.mp3" controls="controls"></audio> 
<p>你好，我是胡夕。今天我要和你分享的主题是：Kafka Streams与其他流处理平台的差异。</p><p>近些年来，开源流处理领域涌现出了很多优秀框架。光是在Apache基金会孵化的项目，关于流处理的大数据框架就有十几个之多，比如早期的Apache Samza、Apache Storm，以及这两年火爆的Spark以及Flink等。</p><p>应该说，每个框架都有自己独特的地方，也都有自己的缺陷。面对这众多的流处理框架，我们应该如何选择呢？今天，我就来梳理几个主流的流处理平台，并重点分析一下Kafka Streams与其他流处理平台的差异。</p><h2>什么是流处理平台？</h2><p>首先，我们有必要了解一下流处理平台的概念。<a href="https://www.oreilly.com/library/view/streaming-systems/9781491983867/ch01.html">“Streaming Systems”</a>一书是这么定义“流处理平台”的：<strong>流处理平台（Streaming System）是处理无限数据集（Unbounded Dataset）的数据处理引擎，而流处理是与批处理（Batch Processing）相对应的。</strong></p><p>所谓的无限数据，是指数据永远没有尽头。流处理平台是专门处理这种数据集的系统或框架。当然，这并不是说批处理系统不能处理这种无限数据集，只是通常情况下，它更擅长处理有限数据集（Bounded Dataset）。</p><!-- [[[read_end]]] --><p>那流处理和批处理究竟该如何区分呢？下面这张图应该能帮助你快速且直观地理解它们的区别。</p><p><img src="https://static001.geekbang.org/resource/image/2f/b2/2f8e72ce532cf1d05306cb8b78510bb2.png?wh=1837*1340" alt=""></p><p>好了，现在我来详细解释一下流处理和批处理的区别。</p><p>长期以来，流处理给人的印象通常是低延时，但是结果不准确。每来一条消息，它就能计算一次结果，但由于它处理的大多是无界数据，可能永远也不会结束，因此在流处理中，我们很难精确描述结果何时是精确的。理论上，流处理的计算结果会不断地逼近精确结果。</p><p>但是，它的竞争对手批处理则正好相反。批处理能提供准确的计算结果，但往往延时很高。</p><p>因此，业界的大神们扬长避短，将两者结合在一起使用。一方面，利用流处理快速地给出不那么精确的结果；另一方面，依托于批处理，最终实现数据一致性。这就是所谓的<strong>Lambda架构</strong>。</p><p>延时低是个很好的特性，但如果计算结果不准确，流处理是无法完全替代批处理的。所谓计算结果准确，在教科书或文献中有个专属的名字，叫正确性（Correctness）。可以这么说，<strong>目前难以实现正确性是流处理取代批处理的最大障碍</strong>，而实现正确性的基石是精确一次处理语义（Exactly Once Semantics，EOS）。</p><p>这里的精确一次是流处理平台能提供的一类一致性保障。常见的一致性保障有三类：</p><ul>
<li>至多一次（At most once）语义：消息或事件对应用状态的影响最多只有一次。</li>
<li>至少一次（At least once）语义：消息或事件对应用状态的影响最少一次。</li>
<li>精确一次（Exactly once）语义：消息或事件对应用状态的影响有且只有一次。</li>
</ul><p>注意，我这里说的都是<strong>对应用状态的影响</strong>。对于很多有副作用（Side Effect）的操作而言，实现精确一次语义几乎是不可能的。举个例子，假设流处理中的某个步骤是发送邮件操作，当邮件发送出去后，倘若后面出现问题要回滚整个流处理流程，已发送的邮件是没法追回的，这就是所谓的副作用。当你的流处理逻辑中存在包含副作用的操作算子时，该操作算子的执行是无法保证精确一次处理的。因此，我们通常只是保证这类操作对应用状态的影响精确一次罢了。后面我们会重点讨论Kafka Streams是如何实现EOS的。</p><p>我们今天讨论的流处理既包含真正的实时流处理，也包含微批化（Microbatch）的流处理。<strong>所谓的微批化，其实就是重复地执行批处理引擎来实现对无限数据集的处理</strong>。典型的微批化实现平台就是<strong>Spark Streaming</strong>。</p><h2>Kafka Streams的特色</h2><p>相比于其他流处理平台，<strong>Kafka Streams最大的特色就是它不是一个平台</strong>，至少它不是一个具备完整功能（Full-Fledged）的平台，比如其他框架中自带的调度器和资源管理器，就是Kafka Streams不提供的。</p><p>Kafka官网明确定义Kafka Streams是一个<strong>Java客户端库</strong>（Client Library）。<strong>你可以使用这个库来构建高伸缩性、高弹性、高容错性的分布式应用以及微服务</strong>。</p><p>使用Kafka Streams API构建的应用就是一个普通的Java应用程序。你可以选择任何熟悉的技术或框架对其进行编译、打包、部署和上线。</p><p>在我看来，这是Kafka Streams与Storm、Spark Streaming或Flink最大的区别。</p><p>Java客户端库的定位既可以说是特色，也可以说是一个缺陷。目前Kafka Streams在国内推广缓慢的一个重要原因也在于此。毕竟，很多公司希望它是一个功能完备的平台，既能提供流处理应用API，也能提供集群资源管理与调度方面的能力。所以，这个定位到底是特色还是缺陷，仁者见仁、智者见智吧。</p><h2>Kafka Streams与其他框架的差异</h2><p>接下来，我从应用部署、上下游数据源、协调方式和消息语义保障（Semantic Guarantees）4个方面，详细分析一下Kafka Streams与其他框架的差异。</p><h3>应用部署</h3><p>首先，我们从流处理应用部署方式上对Kafka Streams及其他框架进行区分。</p><p>我们刚刚提到过，Kafka Streams应用需要开发人员自行打包和部署，你甚至可以将Kafka Streams应用嵌入到其他Java应用中。因此，作为开发者的你，除了要开发代码之外，还要自行管理Kafka Streams应用的生命周期，要么将其打包成独立的jar包单独运行，要么将流处理逻辑嵌入到微服务中，开放给其他服务调用。</p><p>但不论是哪种部署方式，你需要自己处理，不要指望Kafka Streams帮你做这些事情。</p><p>相反地，其他流处理平台则提供了完整的部署方案。我以Apache Flink为例来解释一下。在Flink中，流处理应用会被建模成单个的流处理计算逻辑，并封装进Flink的作业中。类似地，Spark中也有作业的概念，而在Storm中则叫拓扑（Topology）。作业的生命周期由框架来管理，特别是在Flink中，Flink框架自行负责管理作业，包括作业的部署和更新等。这些都无需应用开发人员干预。</p><p>另外，Flink这类框架都存在<strong>资源管理器</strong>（Resource Manager）的角色。一个作业所需的资源完全由框架层的资源管理器来支持。常见的资源管理器，如YARN、Kubernetes、Mesos等，比较新的流处理框架（如Spark、Flink等）都是支持的。像Spark和Flink这样的框架，也支持Standalone集群的方式，即不借助于任何已有的资源管理器，完全由集群自己来管理资源。这些都是Kafka Streams无法提供的。</p><p>因此，从应用部署方面来看，Kafka Streams更倾向于将部署交给开发人员来做，而不是依赖于框架自己实现。</p><h3>上下游数据源</h3><p>谈完了部署方式的差异，我们来说说连接上下游数据源方面的差异。简单来说，<strong>Kafka Streams目前只支持从Kafka读数据以及向Kafka写数据</strong>。在没有Kafka Connect组件的支持下，Kafka Streams只能读取Kafka集群上的主题数据，在完成流处理逻辑后也只能将结果写回到Kafka主题上。</p><p>反观Spark Streaming和Flink这类框架，它们都集成了丰富的上下游数据源连接器（Connector），比如常见的连接器MySQL、ElasticSearch、HBase、HDFS、Kafka等。如果使用这些框架，你可以很方便地集成这些外部框架，无需二次开发。</p><p>当然，由于开发Connector通常需要同时掌握流处理框架和外部框架，因此在实际使用过程中，Connector的质量参差不齐，在具体使用的时候，你可以多查查对应的<strong>jira官网</strong>，看看有没有明显的“坑”，然后再决定是否使用。</p><p>在这个方面，我是有前车之鉴的。曾经，我使用过一个Connector，我发现它在读取Kafka消息向其他系统写入的时候似乎总是重复消费。费了很多周折之后，我才发现这是一个已知的Bug，而且早就被记录在jira官网上了。因此，我推荐你多逛下jira，也许能提前避开一些“坑”。</p><p><strong>总之，目前Kafka Streams只支持与Kafka集群进行交互，它没有提供开箱即用的外部数据源连接器。</strong></p><h3>协调方式</h3><p>在分布式协调方面，Kafka Streams应用依赖于Kafka集群提供的协调功能，来提供<strong>高容错性和高伸缩性</strong>。</p><p><strong>Kafka Streams应用底层使用了消费者组机制来实现任意的流处理扩缩容</strong>。应用的每个实例或节点，本质上都是相同消费者组下的独立消费者，彼此互不影响。它们之间的协调工作，由Kafka集群Broker上对应的协调者组件来完成。当有实例增加或退出时，协调者自动感知并重新分配负载。</p><p>我画了一张图来展示每个Kafka Streams实例内部的构造，从这张图中，我们可以看出，每个实例都由一个消费者实例、特定的流处理逻辑，以及一个生产者实例组成，而这些实例中的消费者实例，共同构成了一个消费者组。</p><p><img src="https://static001.geekbang.org/resource/image/4d/4d/4de6620323d74aa537127c5405bac54d.jpg?wh=2416*2037" alt=""></p><p>通过这个机制，Kafka Streams应用同时实现了<strong>高伸缩性和高容错性</strong>，而这一切都是自动提供的，不需要你手动实现。</p><p>而像Flink这样的框架，它的容错性和扩展性是通过专属的主节点（Master Node）全局来协调控制的。</p><p>Flink支持通过ZooKeeper实现主节点的高可用性，避免单点失效：<strong>某个节点出现故障会自动触发恢复操作</strong>。<strong>这种全局性协调模型对于流处理中的作业而言非常实用，但不太适配单独的流处理应用程序</strong>。原因就在于它不像Kafka Streams那样轻量级，应用程序必须要实现特定的API来开启检查点机制（checkpointing），同时还需要亲身参与到错误恢复的过程中。</p><p>应该这样说，在不同的场景下，Kafka Streams和Flink这种重量级的协调模型各有优劣。</p><h3>消息语义保障</h3><p>我们刚刚提到过EOS，目前很多流处理框架都宣称它们实现了EOS，也包括Kafka Streams本身。关于精确一次处理语义，有一些地方需要澄清一下。</p><p>实际上，当把Spark、Flink与Kafka结合使用时，如果不使用Kafka在0.11.0.0版本引入的幂等性Producer和事务型Producer，这些框架是无法实现端到端的EOS的。</p><p>因为这些框架与Kafka是相互独立的，彼此之间没有任何语义保障机制。但如果使用了事务机制，情况就不同了。这些外部系统利用Kafka的事务机制，保障了消息从Kafka读取到计算再到写入Kafka的全流程EOS。这就是所谓的端到端精确一次处理语义。</p><p>之前Spark和Flink宣称的EOS都是在各自的框架内实现的，无法实现端到端的EOS。只有使用了Kafka的事务机制，它们对应的Connector才有可能支持端到端精确一次处理语义。</p><p>Spark官网上明确指出了<strong>用户若要实现与Kafka的EOS，必须自己确保幂等输出和位移保存在同一个事务中。如果你不能自己实现这套机制，那么就要依赖于Kafka提供的事务机制来保证</strong>。</p><p>而Flink在Kafka 0.11之前也宣称提供EOS，不过是有前提条件的，即每条消息对<strong>Flink应用状态</strong>的影响有且只有一次。</p><p>举个例子，如果你使用Flink从Kafka读取消息，然后不加任何处理直接写入到MySQL，那么这个操作就是无状态的，此时Flink无法保证端到端的EOS。</p><p>换句话说，Flink最后写入到MySQL的Kafka消息可能有重复的。当然，Flink社区自1.4版本起正式实现了端到端的EOS，其基本设计思想正是基于Kafka 0.11幂等性Producer的两阶段提交机制。</p><p>两阶段提交（2-Phase Commit，2PC）机制是一种分布式事务机制，用于实现分布式系统上跨多个节点事务的原子性提交。下面这张图来自于神书“Designing Data-Intensive Applications”中关于2PC讲解的章节。它清晰地描述了一次成功2PC的过程。在这张图中，两个数据库参与到分布式事务的提交过程中，它们各自做了一些变更，现在需要使用2PC来保证两个数据库的变更被原子性地提交。如图所示，2PC被分为两个阶段：Prepare阶段和Commit阶段。只有完整地执行了这两个阶段，这个分布式事务才算是提交成功。</p><p><img src="https://static001.geekbang.org/resource/image/eb/c4/eb979df60ca9a2d2bb811febbe4545c4.png?wh=2880*962" alt=""></p><p>分布式系统中的2PC常见于数据库内部实现或以XA事务的方式供各种异质系统使用。Kafka也借鉴了2PC的思想，在Kafka内部实现了基于2PC的事务机制。</p><p>但是，对于Kafka Streams而言，情况就不同了。它天然支持端到端的EOS，因为它本来就是和Kafka紧密相连的。</p><p>下图展示了一个典型的Kafka Streams应用的执行逻辑。</p><p><img src="https://static001.geekbang.org/resource/image/3a/44/3a0cd5a5b04ea5d7c7c082ecd9d63144.jpg?wh=1857*1811" alt=""></p><p>通常情况下，一个Kafka Streams需要执行5个步骤：</p><ol>
<li>读取最新处理的消息位移；</li>
<li>读取消息数据；</li>
<li>执行处理逻辑；</li>
<li>将处理结果写回到Kafka；</li>
<li>保存位置信息。</li>
</ol><p>这五步的执行必须是<strong>原子性</strong>的，否则无法实现精确一次处理语义。</p><p>在设计上，Kafka Streams在底层大量使用Kafka事务机制和幂等性Producer来实现多分区的原子性写入，又因为它只能读写Kafka，因此Kafka Streams很容易地就实现了端到端的EOS。</p><p>总之，虽然Flink自1.4版本也提供与Kafka的EOS，但从适配性来考量的话，应该说Kafka Streams与Kafka的适配性是最好的。</p><h2>小结</h2><p>好了，我们来小结一下。今天，我重点分享了Kafka Streams与其他流处理框架或平台的差异。总的来说，Kafka Streams是一个轻量级的客户端库，而其他流处理平台都是功能完备的流处理解决方案。这是Kafka Streams的特色所在，但同时可能也是缺陷。不过，我认为很多情况下我们并不需要重量级的流处理解决方案，采用轻量级的库API帮助我们实现实时计算是很方便的情形，我想，这或许是Kafka Streams未来的破局之路吧。</p><p>在专栏后面的内容中，我会详细介绍如何使用Kafka Streams API实现实时计算，并跟你分享一个实际的案例，希望这些能激发你对Kafka Streams的兴趣，并为你以后的探索奠定基础。</p><p><img src="https://static001.geekbang.org/resource/image/d2/0e/d2d0b922414c3776d34523946feeae0e.jpg?wh=2069*2560" alt=""></p><h2>开放讨论</h2><p>知乎上有个关于Kafka Streams的“灵魂拷问”：<a href="https://www.zhihu.com/question/337923430/answer/787298849">为什么Kafka Streams没什么人用？</a>我推荐你去看一下，并谈谈你对这个问题的理解和答案。</p><p>欢迎写下你的思考和答案，我们一起讨论。如果你觉得有所收获，也欢迎把文章分享给你的朋友。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/9f/5e/479f33a6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>沧海一粟</span>
  </div>
  <div class="_2_QraFYR_0">分不太清streams和直接启动consumer消费有什么区别？都是实时的呀。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Kafka Streams是实时流处理组件，默认提供了很多算子，组合在一起可以实现较为复杂的流处理逻辑。Consumer只是单纯的消费者组件，没有这些算子。另外Consumer也不能保证EOS和operator状态管理等常见的流处理框架提供的功能</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-19 16:00:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/52/bb/225e70a6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>hunterlodge</span>
  </div>
  <div class="_2_QraFYR_0">老师，我一直没理解流处理的正确性是什么，既然是处理无限的数据，那又怎么可以和批处理来比较呢？好比我们无法比较一个无限整数集合的sum以及一个有限整数集合的sum呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 无限数据集也可以按照时间线进行窗口化切分，那么我们就关心每个窗口的实时计算结果是否能够和离线计算这段时间内的结果匹配上</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-07 11:51:24</div>
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
  <div class="_2_QraFYR_0">留言好少，我加一个场景，我在用stream关联应用日志和istio的open tracing调用链日志</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-19 12:25:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/64/15/9c9ca35c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>sipom</span>
  </div>
  <div class="_2_QraFYR_0">batch、streaming区别的图非常形象👍</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Thanks, man:)</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-22 11:51:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c8/69/ce605c29.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>轶</span>
  </div>
  <div class="_2_QraFYR_0">单纯从功能角度来看，Flink 等标准的大数据处理框架肯定是胜过 kafka streams 的。但按照 Confluent CTO 的说法，二者之间的最大区别是：“Flink 和 Kafka Streams 程序之间的根本区别在于它们的部署和管理方式。”<br>怎么理解这句话呢？一般来说 Flink 等大数据处理平台都是由公司的大数据团队部署和管理的，如果某个应用项目中的部分处理逻辑需要依赖于这些大数据处理平台的话，那必然涉及团队间的协作问题，很多时候这比技术问题本身还难以解决😅。但是如果相关处理不是很复杂的话，其实完全可以通过 Kafka streams 来解决，因此应用研发团队可以完全掌控相关的代码和部署，从而能够快速响应需求变化。 </div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-23 14:47:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/DLcTlSKOQlrhRq1hzBNvnWfENsyFrxNnhJ5UPibPMLazy9c2nBlSd1sxHqzHaOTTaZIYkEDAby3HpdianMxt6Dsw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Joey</span>
  </div>
  <div class="_2_QraFYR_0">所以相比flink，kafka stream一点也不香</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-01 15:55:19</div>
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
  <div class="_2_QraFYR_0">想问老师对新出的ksql有什么看法</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 个人感觉市场定位不是很清晰。大数据工程师本身不会用，而对于纯数据分析人员门槛又有点高。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-05 08:40:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/tVtdATOIDmDt85Xuv5dAHNpk93NVTPiaU2MtPA4DLXQia6VXwycpreNzkQs91bLuMCrYUhdGShOtcR1GKW3cp5xA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jellyabd</span>
  </div>
  <div class="_2_QraFYR_0"> xAcs</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-05 07:42:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1c/9f/0c/8484f8b1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>夏落de烦恼</span>
  </div>
  <div class="_2_QraFYR_0">flink和kafka stream的算子是一样的么？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-05 08:55:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/PiajxSqBRaEIoRAqV3Yic1wa2gKDq74h1SB5azIpZAOE2uY43CZevju1vd4wxibXq3Y6LJvxJ4tlsJEEmkI64ZJvw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>余晓杰</span>
  </div>
  <div class="_2_QraFYR_0">Java8 stream和kafka stream 有可比性吗</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-11 22:47:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/26/eb/d7/90391376.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ifelse</span>
  </div>
  <div class="_2_QraFYR_0">中国程序员喜欢开箱即用，都要自己开发很多人可能能力就不足以胜任。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-11 21:06:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/12/4e/ff0702fc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>火锅小王子</span>
  </div>
  <div class="_2_QraFYR_0">老师好，请教下，kafka stream模型底层有应该也是采用拉模型的方式吧，这样的话还是一个轮训的过程，也就是实际上不是属于那种真正的流数据吧 ？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: push还pull不影响真正的流处理</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-27 11:41:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/f1/12/7dac30d6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>你为啥那么牛</span>
  </div>
  <div class="_2_QraFYR_0">我说一下对于流处理计算不准确的问题，似乎大家对这块不太理解。什么是不准确？没有绝对的准确。即使是批处理也不敢保证准确。<br><br>流处理与批处理最大的区别就是实时性，批处理在一个处理间隔周期内，从业务角度来说，我们可以认为它所依靠的基础数据是可信可靠的，可以保证的。基于此，我们认为批处理上准确的。<br><br>但是流处理就不一样了，它是实时性的，基础数据在最终确认可信前，数据状态是一直在变化的，就比如网上买东西，购物车一直是变化的，只有你下单后，才能确定具体买了哪些。<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-13 12:57:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/52/c7/f9ad5669.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>时彬斌</span>
  </div>
  <div class="_2_QraFYR_0">老师好，请问有好用的golang版的处理streams的库推荐吗，目前用的kasper</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对golang的客户端确实不太了解。。。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-13 06:19:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/41/59/78042964.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Cryhard</span>
  </div>
  <div class="_2_QraFYR_0">突然觉得，这节课的文稿封面图片和内容还是很搭的。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-05 17:44:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/0c/52/f25c3636.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>长脖子树</span>
  </div>
  <div class="_2_QraFYR_0">配图 好评 哈哈哈</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 哈哈哈，谢谢~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-17 16:57:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/59/67/f4ba1da4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Hello world</span>
  </div>
  <div class="_2_QraFYR_0">老师，你使用有bug的connector是官方的还是自己写的呢？kafka stream如果要写入其他数据源，是不是就得开发自己的connector呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不是官方的，是个人写的。目前Confluent公司在给各个connector做认证。我使用的时候还是比较久远的年代。。。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-11 19:38:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/1b/4a/f9df2d06.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>蒙开强</span>
  </div>
  <div class="_2_QraFYR_0">老师，你好，kafka流处理sink端的自带支持少，但可以自己用第三方包把结果写入mysql，hbase等的</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-05 08:32:12</div>
  </div>
</div>
</div>
</li>
</ul>