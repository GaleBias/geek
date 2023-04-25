<audio title="23 _ Kafka副本机制详解" src="https://static001.geekbang.org/resource/audio/53/39/53086ba9da1792a3fc84bd24bfe34b39.mp3" controls="controls"></audio> 
<p>你好，我是胡夕。今天我要和你分享的主题是：Apache Kafka的副本机制。</p><p>所谓的副本机制（Replication），也可以称之为备份机制，通常是指分布式系统在多台网络互联的机器上保存有相同的数据拷贝。副本机制有什么好处呢？</p><ol>
<li><strong>提供数据冗余</strong>。即使系统部分组件失效，系统依然能够继续运转，因而增加了整体可用性以及数据持久性。</li>
<li><strong>提供高伸缩性</strong>。支持横向扩展，能够通过增加机器的方式来提升读性能，进而提高读操作吞吐量。</li>
<li><strong>改善数据局部性</strong>。允许将数据放入与用户地理位置相近的地方，从而降低系统延时。</li>
</ol><p>这些优点都是在分布式系统教科书中最常被提及的，但是有些遗憾的是，对于Apache Kafka而言，目前只能享受到副本机制带来的第1个好处，也就是提供数据冗余实现高可用性和高持久性。我会在这一讲后面的内容中，详细解释Kafka没能提供第2点和第3点好处的原因。</p><p>不过即便如此，副本机制依然是Kafka设计架构的核心所在，它也是Kafka确保系统高可用和消息高持久性的重要基石。</p><h2>副本定义</h2><p>在讨论具体的副本机制之前，我们先花一点时间明确一下副本的含义。</p><p>我们之前谈到过，Kafka是有主题概念的，而每个主题又进一步划分成若干个分区。副本的概念实际上是在分区层级下定义的，每个分区配置有若干个副本。</p><!-- [[[read_end]]] --><p><strong>所谓副本（Replica），本质就是一个只能追加写消息的提交日志</strong>。根据Kafka副本机制的定义，同一个分区下的所有副本保存有相同的消息序列，这些副本分散保存在不同的Broker上，从而能够对抗部分Broker宕机带来的数据不可用。</p><p>在实际生产环境中，每台Broker都可能保存有各个主题下不同分区的不同副本，因此，单个Broker上存有成百上千个副本的现象是非常正常的。</p><p>接下来我们来看一张图，它展示的是一个有3台Broker的Kafka集群上的副本分布情况。从这张图中，我们可以看到，主题1分区0的3个副本分散在3台Broker上，其他主题分区的副本也都散落在不同的Broker上，从而实现数据冗余。</p><p><img src="https://static001.geekbang.org/resource/image/3b/77/3b5f28c6d19b2c6fe592b2b78d3ebc77.jpg?wh=3109*1485" alt=""></p><h2>副本角色</h2><p>既然分区下能够配置多个副本，而且这些副本的内容还要一致，那么很自然的一个问题就是：我们该如何确保副本中所有的数据都是一致的呢？特别是对Kafka而言，当生产者发送消息到某个主题后，消息是如何同步到对应的所有副本中的呢？针对这个问题，最常见的解决方案就是采用<strong>基于领导者（Leader-based）的副本机制</strong>。Apache Kafka就是这样的设计。</p><p>基于领导者的副本机制的工作原理如下图所示，我来简单解释一下这张图里面的内容。</p><p><img src="https://static001.geekbang.org/resource/image/38/c2/381eda4b56991d52727934be7c7e6ec2.jpg?wh=3591*1508" alt=""></p><p>第一，在Kafka中，副本分成两类：领导者副本（Leader Replica）和追随者副本（Follower Replica）。每个分区在创建时都要选举一个副本，称为领导者副本，其余的副本自动称为追随者副本。</p><p>第二，Kafka的副本机制比其他分布式系统要更严格一些。在Kafka中，追随者副本是不对外提供服务的。这就是说，任何一个追随者副本都不能响应消费者和生产者的读写请求。所有的请求都必须由领导者副本来处理，或者说，所有的读写请求都必须发往领导者副本所在的Broker，由该Broker负责处理。追随者副本不处理客户端请求，它唯一的任务就是从领导者副本<strong>异步拉取</strong>消息，并写入到自己的提交日志中，从而实现与领导者副本的同步。</p><p>第三，当领导者副本挂掉了，或者说领导者副本所在的Broker宕机时，Kafka依托于ZooKeeper提供的监控功能能够实时感知到，并立即开启新一轮的领导者选举，从追随者副本中选一个作为新的领导者。老Leader副本重启回来后，只能作为追随者副本加入到集群中。</p><p>你一定要特别注意上面的第二点，即<strong>追随者副本是不对外提供服务的</strong>。还记得刚刚我们谈到副本机制的好处时，说过Kafka没能提供读操作横向扩展以及改善局部性吗？具体的原因就在于此。</p><p>对于客户端用户而言，Kafka的追随者副本没有任何作用，它既不能像MySQL那样帮助领导者副本“扛读”，也不能实现将某些副本放到离客户端近的地方来改善数据局部性。</p><p>既然如此，Kafka为什么要这样设计呢？其实这种副本机制有两个方面的好处。</p><p>1.<strong>方便实现“Read-your-writes”</strong>。</p><p>所谓Read-your-writes，顾名思义就是，当你使用生产者API向Kafka成功写入消息后，马上使用消费者API去读取刚才生产的消息。</p><p>举个例子，比如你平时发微博时，你发完一条微博，肯定是希望能立即看到的，这就是典型的Read-your-writes场景。如果允许追随者副本对外提供服务，由于副本同步是异步的，因此有可能出现追随者副本还没有从领导者副本那里拉取到最新的消息，从而使得客户端看不到最新写入的消息。</p><p>2.<strong>方便实现单调读（Monotonic Reads）</strong>。</p><p>什么是单调读呢？就是对于一个消费者用户而言，在多次消费消息时，它不会看到某条消息一会儿存在一会儿不存在。</p><p>如果允许追随者副本提供读服务，那么假设当前有2个追随者副本F1和F2，它们异步地拉取领导者副本数据。倘若F1拉取了Leader的最新消息而F2还未及时拉取，那么，此时如果有一个消费者先从F1读取消息之后又从F2拉取消息，它可能会看到这样的现象：第一次消费时看到的最新消息在第二次消费时不见了，这就不是单调读一致性。但是，如果所有的读请求都是由Leader来处理，那么Kafka就很容易实现单调读一致性。</p><h2>In-sync Replicas（ISR）</h2><p>我们刚刚反复说过，追随者副本不提供服务，只是定期地异步拉取领导者副本中的数据而已。既然是异步的，就存在着不可能与Leader实时同步的风险。在探讨如何正确应对这种风险之前，我们必须要精确地知道同步的含义是什么。或者说，Kafka要明确地告诉我们，追随者副本到底在什么条件下才算与Leader同步。</p><p>基于这个想法，Kafka引入了In-sync Replicas，也就是所谓的ISR副本集合。ISR中的副本都是与Leader同步的副本，相反，不在ISR中的追随者副本就被认为是与Leader不同步的。那么，到底什么副本能够进入到ISR中呢？</p><p>我们首先要明确的是，Leader副本天然就在ISR中。也就是说，<strong>ISR不只是追随者副本集合，它必然包括Leader副本。甚至在某些情况下，ISR只有Leader这一个副本</strong>。</p><p>另外，能够进入到ISR的追随者副本要满足一定的条件。至于是什么条件，我先卖个关子，我们先来一起看看下面这张图。</p><p><img src="https://static001.geekbang.org/resource/image/52/5f/521ff90472a5fd2e6cfac0e6176aa75f.jpg?wh=3506*1040" alt=""></p><p>图中有3个副本：1个领导者副本和2个追随者副本。Leader副本当前写入了10条消息，Follower1副本同步了其中的6条消息，而Follower2副本只同步了其中的3条消息。现在，请你思考一下，对于这2个追随者副本，你觉得哪个追随者副本与Leader不同步？</p><p>答案是，要根据具体情况来定。换成英文，就是那句著名的“It depends”。看上去好像Follower2的消息数比Leader少了很多，它是最有可能与Leader不同步的。的确是这样的，但仅仅是可能。</p><p>事实上，这张图中的2个Follower副本都有可能与Leader不同步，但也都有可能与Leader同步。也就是说，Kafka判断Follower是否与Leader同步的标准，不是看相差的消息数，而是另有“玄机”。</p><p><strong>这个标准就是Broker端参数replica.lag.time.max.ms参数值</strong>。这个参数的含义是Follower副本能够落后Leader副本的最长时间间隔，当前默认值是10秒。这就是说，只要一个Follower副本落后Leader副本的时间不连续超过10秒，那么Kafka就认为该Follower副本与Leader是同步的，即使此时Follower副本中保存的消息明显少于Leader副本中的消息。</p><p>我们在前面说过，Follower副本唯一的工作就是不断地从Leader副本拉取消息，然后写入到自己的提交日志中。如果这个同步过程的速度持续慢于Leader副本的消息写入速度，那么在replica.lag.time.max.ms时间后，此Follower副本就会被认为是与Leader副本不同步的，因此不能再放入ISR中。此时，Kafka会自动收缩ISR集合，将该副本“踢出”ISR。</p><p>值得注意的是，倘若该副本后面慢慢地追上了Leader的进度，那么它是能够重新被加回ISR的。这也表明，ISR是一个动态调整的集合，而非静态不变的。</p><h2>Unclean领导者选举（Unclean Leader Election）</h2><p>既然ISR是可以动态调整的，那么自然就可以出现这样的情形：ISR为空。因为Leader副本天然就在ISR中，如果ISR为空了，就说明Leader副本也“挂掉”了，Kafka需要重新选举一个新的Leader。可是ISR是空，此时该怎么选举新Leader呢？</p><p><strong>Kafka把所有不在ISR中的存活副本都称为非同步副本</strong>。通常来说，非同步副本落后Leader太多，因此，如果选择这些副本作为新Leader，就可能出现数据的丢失。毕竟，这些副本中保存的消息远远落后于老Leader中的消息。在Kafka中，选举这种副本的过程称为Unclean领导者选举。<strong>Broker端参数unclean.leader.election.enable控制是否允许Unclean领导者选举</strong>。</p><p>开启Unclean领导者选举可能会造成数据丢失，但好处是，它使得分区Leader副本一直存在，不至于停止对外提供服务，因此提升了高可用性。反之，禁止Unclean领导者选举的好处在于维护了数据的一致性，避免了消息丢失，但牺牲了高可用性。</p><p>如果你听说过CAP理论的话，你一定知道，一个分布式系统通常只能同时满足一致性（Consistency）、可用性（Availability）、分区容错性（Partition tolerance）中的两个。显然，在这个问题上，Kafka赋予你选择C或A的权利。</p><p>你可以根据你的实际业务场景决定是否开启Unclean领导者选举。不过，我强烈建议你<strong>不要</strong>开启它，毕竟我们还可以通过其他的方式来提升高可用性。如果为了这点儿高可用性的改善，牺牲了数据一致性，那就非常不值当了。</p><h2>小结</h2><p>今天，我主要跟你分享了Apache Kafka的副本机制以及它们实现的原理。坦率地说，我觉得有些地方可能讲浅了，如果要百分之百地了解Replication，你还是要熟读一下Kafka相应的源代码。不过你也不用担心，在专栏后面的内容中，我会专门从源码角度分析副本机制，特别是Follower副本从Leader副本拉取消息的全过程。从技术深度上来说，那一讲应该算是本专栏中最贴近技术内幕的分析了，你一定不要错过。</p><p><img src="https://static001.geekbang.org/resource/image/d7/72/d75c01661ca5367cfd23ad92cc10e372.jpg?wh=2069*2535" alt=""></p><h2>开放讨论</h2><p>到目前为止，我反复强调了Follower副本不对外提供服务这件事情。有意思的是，社区最近正在考虑是否要打破这个限制，即允许Follower副本处理客户端消费者发来的请求。社区主要的考量是，这能够用于改善云上数据的局部性，更好地服务地理位置相近的客户。如果允许Follower副本对外提供读服务，你觉得应该如何避免或缓解因Follower副本与Leader副本不同步而导致的数据不一致的情形？</p><p>欢迎写下你的思考和答案，我们一起讨论。如果你觉得有所收获，也欢迎把文章分享给你的朋友。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/9c/35/9dc79371.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>你好旅行者</span>
  </div>
  <div class="_2_QraFYR_0">老师讲的很好，我做一些补充吧。<br><br>Kafka在启动的时候会开启两个任务，一个任务用来定期地检查是否需要缩减或者扩大ISR集合，这个周期是replica.lag.time.max.ms的一半，默认5000ms。当检测到ISR集合中有失效副本时，就会收缩ISR集合，当检查到有Follower的HighWatermark追赶上Leader时，就会扩充ISR。<br><br>除此之外，当ISR集合发生变更的时候还会将变更后的记录缓存到isrChangeSet中，另外一个任务会周期性地检查这个Set,如果发现这个Set中有ISR集合的变更记录，那么它会在zk中持久化一个节点。然后因为Controllr在这个节点的路径上注册了一个Watcher，所以它就能够感知到ISR的变化，并向它所管理的broker发送更新元数据的请求。最后删除该路径下已经处理过的节点。<br><br>此外，在0.9X版本之前，Kafka中还有另外一个参数replica.lag.max.messages，它也是用来判定失效副本的，当一个副本滞后leader副本的消息数超过这个参数的大小时，则判定它处于同步失效的状态。它与replica.lag.time.max.ms参数判定出的失效副本取并集组成一个失效副本集合。<br><br>不过这个参数本身很难给出一个合适的值。以默认的值4000为例，对于消息流入速度很低的主题（比如TPS为10），这个参数就没什么用；对于消息流入速度很高的主题（比如TPS为2000），这个参数的取值又会引入ISR的频繁变动。所以从0.9x版本开始，Kafka就彻底移除了这一个参数。<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-27 15:23:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/a0/cb/aab3b3e7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张三丰</span>
  </div>
  <div class="_2_QraFYR_0">replica.lag.time.max.ms，感觉老师对这个参数的解释有歧义。<br><br>应该是如果leader发现flower超过这个参数所设置的时间没有向它发起fech请求（也就是复制请求），那么leader考虑将这个flower从ISR移除。<br>而不是连续落后这么长时间</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，是的。你的解释更精准:)</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-16 15:52:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/3a/70/a874d69c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Mick</span>
  </div>
  <div class="_2_QraFYR_0">老师，LEO和HW这两个概念不理解，能不能详细说下，谢谢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 一个分区有3个副本，一个leader，2个follower。producer向leader写了10条消息，follower1从leader处拷贝了5条消息，follower2从leader处拷贝了3条消息，那么leader副本的LEO就是10，HW=3；follower1副本的LEO是5。这样说清楚些吗</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-02 00:08:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/ed/c9/57d571a3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>凯</span>
  </div>
  <div class="_2_QraFYR_0">请问一下，producer生产消息ack=all的时候，消息是怎么保证到follower的，因为看到follower是异步拉取数据的，难道是看leader和follower上面的offset吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 通过HW机制。leader处的HW要等所有follower LEO都越过了才会前移</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-01 15:00:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c4/76/abb7bfe3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ideal sail</span>
  </div>
  <div class="_2_QraFYR_0">老师，假设一个分区有5个副本，Broker的min.insync.replicas设置为2，生产者设置acks=all，这时是有2个副本同步了就可以，还是必须是5个副本都同步，他们是什么关系。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Producer端认为消息已经成功提交的条件是：ISR中所有副本都已经保存了该消息，但producer并没有指定ISR中需要几个副本。这就是min.insync.replicas参数的作用。<br><br>正常情况下，如果5个副本都在ISR中，那么它们必须都同步才行，但如果4个副本不在ISR中了，不满足min.insync.replicas了，此时broker会抛出异常给producer，告诉producer这条消息无法正确保存</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-16 11:03:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLl9nj9b6RydKADq82ZwOad0fQcvXWyQKk5U5RFC2kzHGI4GjIQsIZvHsEm7mFELgMiaGx3lGq9vag/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>咸淡一首诗</span>
  </div>
  <div class="_2_QraFYR_0">老师，“这个标准就是 Broker 端参数 replica.lag.time.max.ms 参数值。这个参数的含义是 Follower 副本能够落后 Leader 副本的最长时间间隔，当前默认值是 10 秒” 这句话中的最长时间间隔是怎么计算的，以什么时间为基准？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: follower从leader拿到消息后会更新一个名为_lastCaughtUpTimeMs的字段。每当要检查follower是否out of ISR时就会用当前时间减去这个字段值去和replica.lag.time.max.ms 比较</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-07 16:40:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/5f/b2/c4780c10.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>曹伟雄</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，有个问题请教一下，麻烦抽空看看，谢谢。<br>生产环境，因磁盘满了，所有broker宕机了，重启集群后，主题中的部分分区中，有1个副本被踢出ISR集合，只剩下leader副本了。<br><br>试了以下几种方法都没有自动加入进来：<br>1、等了3天后还是没有加入到ISR；<br>2、然后重启kafka集群；<br>3、用kafka-reassign-partitions.sh命令重新分配分区；<br><br>针对此情况，请问一下有什么办法让它自动加入进来？  或者手工处理加入进来也可以。<br>有什么命令可以查看follower落后多少吗？  麻烦老师给点建议或解决思路，谢谢。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 试试到ZooKeeper中手动删除&#47;controller节点。这通常都是因为Controller与ZooKeeper状态不同步导致的。<br><br>试试这个命令吧： rmr &#47;controller<br><br>确保在业务低峰时刻执行这个命令</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-16 12:47:37</div>
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
  <div class="_2_QraFYR_0">1 副本机制的定义：所谓副本机制（Replication），也可以称之为备份机制，通常是指分布式在多台网络互连的机器上保存有相同的数据拷贝。	<br>2 副本机制的价值：A ：提供数据冗余 B ：提供高伸缩性 C ：改善数据局部性<br>但 Kafka的副本机制，只实现了提供数据冗余的价值。<br>	<br>3 副本定义：<br>A ：Kafka有主题的概念，每个主题又分为若干个分区。副本的概念是在分区层级下定义的，每个分区配置有若干个副本。<br>B ：所谓副本（Replica），本质是一个只能追加写消息的提交日志。<br>   根据Kafka副本机制的定义，同一个分区下的所有副本保存有相同的消息序列，这些副本分散保存在不同的Broker上，从而能够对抗部分Broker宕机带来的数据不可用。<br>	<br>4 副本角色：<br>A ：为解决分区下多个副本的内容一致性问题，常用方案就是采用基于领导者的副本机制。<br>B ：在kafka中，副本分两类：领导者副本和追随者副本。每个分区在创建时都选举一个副本，称为领导者副本，其余的副本自动成为追随者副本。<br>C ：Kafka的副本机制比其他分布式系统严格。Kafka的追随者副本不对外提供服务。所有的请求都要由领导者副本处理。追随者副本唯一的任务就是从领导者副本异步拉取消息，并写入到自己的提交日志中，从而实现与领导者副本的同步。<br>D ：当领导者副本所在Broker宕机了，Kafka依托于Zookeeper提供的监控功能能够实时感知到，并立即开启新一轮的领导者选举，从追随者副本中选一个新的领导者。当老的Leader副本重启回来后，只能作为追随者副本加入到集群中。<br><br>4 Kafka副本机制的优点：<br>A ：方便实现“Read-your-writes”<br>（1）含义：当使用生产者API向Kafka成功写入消息后，马上使用消息者API去读取刚才生产的消息。<br>（2）如果允许追随者副本对外提供服务，由于副本同步是异步的，就可能因为数据同步时间差，从而使客户端看不到最新写入的消息。	<br>B ：方便实现单调读（Monotonic Reads）<br>（1）单调读：对于一个消费者用户而言，在多处消息消息时，他不会看到某条消息一会存在，一会不存在。<br>（2）如果允许追随者副本提供读服务，由于消息是异步的，则多个追随者副本的状态可能不一致。若客户端每次命中的副本不同，就可能出现一条消息一会看到，一会看不到。<br>	<br>5 In-sync Replicas（ISR）同步副本<br>A ：追随者副本定期的异步拉取领导者副本中的数据，这存在不能和Leader实时同步的风险。<br>B ：Kafka引入了In-sync Replicas。ISR中的副本都是于Leader同步的副本，相反，不在ISR中的追随者副本就是被认为是与Leader不同步的。<br>C ：Leader 副本天然就在ISR中，即ISR不只是追随者副本集合，他必然包括Leader副本。甚至某些情况下，ISR只有Leade这一个副本。<br>D ：follower副本是否与leader同步的判断标准取决于Broker端参数 replica.lag.time.max.ms参数值。默认为10秒，只要一个Follower副本落后Leader副本的时间不连续超过10秒，那么Kafka就认为该Follower副本与leader是同步的，即使此时Follower副本中保存的消息明显小于Leader副本中的消息。<br>E ：如果同步过程持续慢于Leader副本消息的写入速度，那么replica.lag.time.max.ms时间后，此Follower副本就会被认为是与Leader副本不同步的，因此不能再放入ISR中。此时，kafka会自动收缩ISR的进度，将该副本“踢出”ISR。ISR是一个动态调整的集合，而非静态不变的。<br><br>6 Unclean 领导者选举（Unclean Leader Election）<br>	A ：ISR是可以动态调整的，所以会出现ISR为空的情况，由于Leader副本天然就在ISR中，如果ISR为空了，这说明Leader副本也挂掉了，Kafka需要重新选举一个新的Leader。<br>	B ：Kafka把所有不在ISR中的存活副本都会称为非同步副本。通常，非同步副本落后Leader太多，如果让这些副本做为新的Leader，就可能出现数据的丢失。在kafka中，选举这种副本的过程称为Unclean领导者选举。<br>	C ：Broker端参数unclean.leader.election.enable 控制是否允许Unclean领导者选举。开启Unclean领导者选举可能会造成数据丢失，但它使得分区Leader副本一直存在，不至于停止对外提供服务，因此提升了高可用性。禁止Unclean领导者选举的好处是在于维护了数据的一致性，避免了消息丢失，但牺牲了高可用性。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-07 20:01:49</div>
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
  <div class="_2_QraFYR_0">ack=all时候，生产者向leader发送完数据，而副本是异步拉取的，那生产者写入线程要一直阻塞等待吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不会阻塞，你可以认为是不断轮询状态</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-11 09:17:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Cv</span>
  </div>
  <div class="_2_QraFYR_0">生产者acks=all使用异步提交, 如果ISR副本迟迟不能完成从leader的同步, 那么10s过后, 生产者会收到提交失败的回调吗? 还是一直不会有回调</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 会的，回调中会包含对应的错误码</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-13 15:54:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/e2/6e/0a300829.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>李先生</span>
  </div>
  <div class="_2_QraFYR_0">胡哥：分区选举leader，是通过抢占模式来选举的。如果不开启unclean.leader.election.enable，是只能isr集合中的broker才能竞争吗？这个竞争的过程能具体说下是如何实现的吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 目前选举leader的算法很简单，一般是选择AR中第一个处在ISR集合的副本为leader。比如AR的副本顺序是[1,2,3]，ISR是[2,3]，那么副本2就是leader</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-13 11:03:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/4c/6d/c20f2d5a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>LJK</span>
  </div>
  <div class="_2_QraFYR_0">老师好，请问ISR中的副本一定可以保证和leader副本的一致性吗？如果有一种情况是某个ISR中副本与leader副本的lag在ISR判断的边界值，这时如果leader副本挂了的话，还是会有数据丢失是吗？谢谢老师</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: ISR中的follower副本非常有可能与leader不一致的。如果leader挂了，其他follower又都没有保存该消息，那么该消息是可能丢失的。如果你要避免这种情况，设置producer端的acks=all吧</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-25 09:01:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/64/77/ce921ed4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>球球</span>
  </div>
  <div class="_2_QraFYR_0">胡夕老师好，replica.lag.time.max.ms配置follower是否处于isr中，那么是否存在，在这个时间段内数据写完leader，follower还没有完全同步leader数据，leader宕机，isr中follower提升为新leader，那这一部分数据是否就丢失呢？该如何避免呢？谢谢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不会丢失。还是那句话：Kafka只对已提交消息做持久化保证。如果你设置了最高等级的持久化需求（比如acks=all），那么follower副本没有同步完成前这条消息就不算已提交，也就不算丢失了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-25 09:45:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/f1/55/8ac4f169.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>陈国林</span>
  </div>
  <div class="_2_QraFYR_0">老师好，我觉得是否可以这样分场景。对于读新的数据可以从 leader replica 读取，对于老一些的数据从follower replica 读取，这样不懂是否可行</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 初衷是好的。难点在于我们如何区分什么数据是新的什么是老的：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-18 15:02:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/f6/9c/b457a937.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>不能扮演天使</span>
  </div>
  <div class="_2_QraFYR_0">老师，ack=all,是保证ISR中的follower同步还是所有的follower同步，还有消费者是只能消费到ISR中的HW处的offset么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: acks=all保证ISR中的所有副本都要同步</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-10 23:05:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/49/3d/4ac37cc2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>外星人</span>
  </div>
  <div class="_2_QraFYR_0">请问，关闭unclean后，有哪些方法可以保证available啊？谢谢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 增加副本数：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-28 11:43:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/12/7b/456e46f2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>刘彬</span>
  </div>
  <div class="_2_QraFYR_0">老师好，想请教您两个问题，如下：<br>如果某个follower副本同步持续慢于leader副本写入速度，repkica.lag.time.max.ms 是对于二者的同步时间做的判断，我理解就是如果一直检查10s follower都赶不上leader副本的进度！<br>但是，这个同步进度是用哪一块进行判别的呢？是通过index值吗？<br>另外，如果某个follower不在ISR中了，kafka如果维持副本数均衡呢？比如设置了副本数为3，其中一个副本不在ISR集合中了，那么就一直少了一个副本吗？前提是这个副本一直没有跟上leader的同步进度！<br>谢谢！<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. 通过比较follower和leader的最新消息位移或末端消息位移（Log End Offset, LEO）<br>2. 嗯，就一直少一个副本了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-25 07:54:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/84/19/7ed2ffa6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>风</span>
  </div>
  <div class="_2_QraFYR_0">ack=all时候，生产者向leader发送完数据，而副本是异步拉取的，那生产者写入线程要一直阻塞等待吗<br>老师这个问题您是说不会阻塞,可以认为是生产会不断地轮询状态<br>那是否可能存在发送了两条消息,可能导致后发送的先写入分区(设置max.in.flight.requests.per.connection),如果不可能的话,此时生产者是否处于阻塞模式,无法再次发送消息</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是可能出现消息乱序的情形</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-24 19:35:45</div>
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
  <div class="_2_QraFYR_0">老师，请问以replica.lag.time.max.ms=10s为副本机制的判断依据，遇到以下场景的话，是如何解决？<br><br>比如 Leader-A LEO=1000  Follower-B LEO=800,  他们的差值是200， 那如何判断这200条消息是否会在10s内同步完？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: follower b leo追到1000时会更新一个时间戳。然后通过当前时间与这个时间戳的差值和10s进行判断<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-30 10:02:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Cv</span>
  </div>
  <div class="_2_QraFYR_0">ISR中的数据也会落后leader, 那么leader挂了之后的重新选举, 一样会造成数据丢失, 为了避免这种请问, 我们是否需要把replica.lag.time.max.ms设置为0呢?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 通常不需要。如果你在意这种情况，使用acks=all的producer吧</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-13 15:08:25</div>
  </div>
</div>
</div>
</li>
</ul>