<audio title="35 _ 跨集群备份解决方案MirrorMaker" src="https://static001.geekbang.org/resource/audio/d3/9b/d3ba7bf9197dcf37d13646b0749a009b.mp3" controls="controls"></audio> 
<p>你好，我是胡夕。今天我要和你分享的主题是：Kafka的跨集群数据镜像工具MirrorMaker。</p><p>一般情况下，我们会使用一套Kafka集群来完成业务，但有些场景确实会需要多套Kafka集群同时工作，比如为了便于实现灾难恢复，你可以在两个机房分别部署单独的Kafka集群。如果其中一个机房出现故障，你就能很容易地把流量打到另一个正常运转的机房下。再比如，你想为地理相近的客户提供低延时的消息服务，而你的主机房又离客户很远，这时你就可以在靠近客户的地方部署一套Kafka集群，让这套集群服务你的客户，从而提供低延时的服务。</p><p>如果要实现这些需求，除了部署多套Kafka集群之外，你还需要某种工具或框架，来帮助你实现数据在集群间的拷贝或镜像。</p><p>值得注意的是，<strong>通常我们把数据在单个集群下不同节点之间的拷贝称为备份，而把数据在集群间的拷贝称为镜像</strong>（Mirroring）。</p><p>今天，我来重点介绍一下Apache Kafka社区提供的MirrorMaker工具，它可以帮我们实现消息或数据从一个集群到另一个集群的拷贝。</p><h2>什么是MirrorMaker？</h2><p>从本质上说，MirrorMaker就是一个消费者+生产者的程序。消费者负责从源集群（Source Cluster）消费数据，生产者负责向目标集群（Target Cluster）发送消息。整个镜像流程如下图所示：</p><!-- [[[read_end]]] --><p><img src="https://static001.geekbang.org/resource/image/a7/e2/a771601d702eb35187a0a8894307eee2.jpg?wh=3681*787" alt=""></p><p>MirrorMaker连接的源集群和目标集群，会实时同步消息。当然，你不要认为你只能使用一套MirrorMaker来连接上下游集群。事实上，很多用户会部署多套集群，用于实现不同的目的。</p><p>我们来看看下面这张图。图中部署了三套集群：左边的源集群负责主要的业务处理；右上角的目标集群可以用于执行数据分析；而右下角的目标集群则充当源集群的热备份。</p><p><img src="https://static001.geekbang.org/resource/image/03/70/036955f42db6fe759849fb24a0d16070.jpg?wh=3535*1835" alt=""></p><h2>运行MirrorMaker</h2><p>Kafka默认提供了MirrorMaker命令行工具kafka-mirror-maker脚本，它的常见用法是指定生产者配置文件、消费者配置文件、线程数以及要执行数据镜像的主题正则表达式。比如下面的这个命令，就是一个典型的MirrorMaker执行命令。</p><pre><code>$ bin/kafka-mirror-maker.sh --consumer.config ./config/consumer.properties --producer.config ./config/producer.properties --num.streams 8 --whitelist &quot;.*&quot;
</code></pre><p>现在我来解释一下这条命令中各个参数的含义。</p><ul>
<li>
<p>consumer.config参数。它指定了MirrorMaker中消费者的配置文件地址，最主要的配置项是<strong>bootstrap.servers</strong>，也就是该MirrorMaker从哪个Kafka集群读取消息。因为MirrorMaker有可能在内部创建多个消费者实例并使用消费者组机制，因此你还需要设置group.id参数。另外，我建议你额外配置auto.offset.reset=earliest，否则的话，MirrorMaker只会拷贝那些在它启动之后到达源集群的消息。</p>
</li>
<li>
<p>producer.config参数。它指定了MirrorMaker内部生产者组件的配置文件地址。通常来说，Kafka Java Producer很友好，你不需要配置太多参数。唯一的例外依然是<strong>bootstrap.servers</strong>，你必须显式地指定这个参数，配置拷贝的消息要发送到的目标集群。</p>
</li>
<li>
<p>num.streams参数。我个人觉得，这个参数的名字很容易给人造成误解。第一次看到这个参数名的时候，我一度以为MirrorMaker是用Kafka Streams组件实现的呢。其实并不是。这个参数就是告诉MirrorMaker要创建多少个KafkaConsumer实例。当然，它使用的是多线程的方案，即在后台创建并启动多个线程，每个线程维护专属的消费者实例。在实际使用时，你可以根据你的机器性能酌情设置多个线程。</p>
</li>
<li>
<p>whitelist参数。如命令所示，这个参数接收一个正则表达式。所有匹配该正则表达式的主题都会被自动地执行镜像。在这个命令中，我指定了“.*”，这表明我要同步源集群上的所有主题。</p>
</li>
</ul><h2>MirrorMaker配置实例</h2><p>现在，我就在测试环境中为你演示一下MirrorMaker的使用方法。</p><p>演示的流程大致是这样的：首先，我们会启动两套Kafka集群，它们是单节点的伪集群，监听端口分别是9092和9093；之后，我们会启动MirrorMaker工具，实时地将9092集群上的消息同步镜像到9093集群上；最后，我们启动额外的消费者来验证消息是否拷贝成功。</p><h3>第1步：启动两套Kafka集群</h3><p>启动日志如下所示：</p><blockquote>
<p>[2019-07-23 17:01:40,544] INFO Kafka version: 2.3.0 (org.apache.kafka.common.utils.AppInfoParser)<br>
[2019-07-23 17:01:40,544] INFO Kafka commitId: fc1aaa116b661c8a (org.apache.kafka.common.utils.AppInfoParser)<br>
[2019-07-23 17:01:40,544] INFO Kafka startTimeMs: 1563872500540 (org.apache.kafka.common.utils.AppInfoParser)<br>
[2019-07-23 17:01:40,545] INFO [KafkaServer id=0] started (kafka.server.KafkaServer)</p>
</blockquote><blockquote>
<p>[2019-07-23 16:59:59,462] INFO Kafka version: 2.3.0 (org.apache.kafka.common.utils.AppInfoParser)<br>
[2019-07-23 16:59:59,462] INFO Kafka commitId: fc1aaa116b661c8a (org.apache.kafka.common.utils.AppInfoParser)<br>
[2019-07-23 16:59:59,462] INFO Kafka startTimeMs: 1563872399459 (org.apache.kafka.common.utils.AppInfoParser)<br>
[2019-07-23 16:59:59,463] INFO [KafkaServer id=1] started (kafka.server.KafkaServer)</p>
</blockquote><h3>第2步：启动MirrorMaker工具</h3><p>在启动MirrorMaker工具之前，我们必须准备好刚刚提过的Consumer配置文件和Producer配置文件。它们的内容分别如下：</p><pre><code>consumer.properties：
bootstrap.servers=localhost:9092
group.id=mirrormaker
auto.offset.reset=earliest
</code></pre><pre><code>producer.properties:
bootstrap.servers=localhost:9093
</code></pre><p>现在，我们来运行命令启动MirrorMaker工具。</p><pre><code>$ bin/kafka-mirror-maker.sh --producer.config ../producer.config --consumer.config ../consumer.config --num.streams 4 --whitelist &quot;.*&quot;
WARNING: The default partition assignment strategy of the mirror maker will change from 'range' to 'roundrobin' in an upcoming release (so that better load balancing can be achieved). If you prefer to make this switch in advance of that release add the following to the corresponding config: 'partition.assignment.strategy=org.apache.kafka.clients.consumer.RoundRobinAssignor'
</code></pre><p>请你一定要仔细阅读这个命令输出中的警告信息。这个警告的意思是，在未来版本中，MirrorMaker内部消费者会使用轮询策略（Round-robin）来为消费者实例分配分区，现阶段使用的默认策略依然是基于范围的分区策略（Range）。Range策略的思想很朴素，它是将所有分区根据一定的顺序排列在一起，每个消费者依次顺序拿走各个分区。</p><p>Round-robin策略的推出时间要比Range策略晚。通常情况下，我们可以认为，社区推出的比较晚的分区分配策略会比之前的策略好。这里的好指的是能实现更均匀的分配效果。该警告信息的最后一部分内容提示我们，<strong>如果我们想提前“享用”轮询策略，需要手动地在consumer.properties文件中增加partition.assignment.strategy的设置</strong>。</p><h3>第3步：验证消息是否拷贝成功</h3><p>好了，启动MirrorMaker之后，我们可以向源集群发送并消费一些消息，然后验证是否所有的主题都能正确地同步到目标集群上。</p><p>假设我们在源集群上创建了一个4分区的主题test，随后使用kafka-producer-perf-test脚本模拟发送了500万条消息。现在，我们使用下面这两条命令来查询一下，目标Kafka集群上是否存在名为test的主题，并且成功地镜像了这些消息。</p><pre><code>$ bin/kafka-run-class.sh kafka.tools.GetOffsetShell --broker-list localhost:9093 --topic test --time -2
test:0:0
</code></pre><pre><code>$ bin/kafka-run-class.sh kafka.tools.GetOffsetShell --broker-list localhost:9093 --topic test --time -1
test:0:5000000
</code></pre><p>-1和-2分别表示获取某分区最新的位移和最早的位移，这两个位移值的差值就是这个分区当前的消息数，在这个例子中，差值是500万条。这说明主题test当前共写入了500万条消息。换句话说，MirrorMaker已经成功地把这500万条消息同步到了目标集群上。</p><p>讲到这里，你一定会觉得很奇怪吧，我们明明在源集群创建了一个4分区的主题，为什么到了目标集群，就变成单分区了呢？</p><p>原因很简单。<strong>MirrorMaker在执行消息镜像的过程中，如果发现要同步的主题在目标集群上不存在的话，它就会根据Broker端参数num.partitions和default.replication.factor的默认值，自动将主题创建出来</strong>。在这个例子中，我们在目标集群上没有创建过任何主题，因此，在镜像开始时，MirrorMaker自动创建了一个名为test的单分区单副本的主题。</p><p><strong>在实际使用场景中，我推荐你提前把要同步的所有主题按照源集群上的规格在目标集群上等价地创建出来</strong>。否则，极有可能出现刚刚的这种情况，这会导致一些很严重的问题。比如原本在某个分区的消息同步到了目标集群以后，却位于其他的分区中。如果你的消息处理逻辑依赖于这样的分区映射，就必然会出现问题。</p><p>除了常规的Kafka主题之外，MirrorMaker默认还会同步内部主题，比如在专栏前面我们频繁提到的位移主题。MirrorMaker在镜像位移主题时，如果发现目标集群尚未创建该主题，它就会根据Broker端参数offsets.topic.num.partitions和offsets.topic.replication.factor的值来制定该主题的规格。默认配置是50个分区，每个分区3个副本。</p><p>在0.11.0.0版本之前，Kafka不会严格依照offsets.topic.replication.factor参数的值。这也就是说，如果你设置了该参数值为3，而当前存活的Broker数量少于3，位移主题依然能被成功创建，只是副本数取该参数值和存活Broker数之间的较小值。</p><p>这个缺陷在0.11.0.0版本被修复了，这就意味着，Kafka会严格遵守你设定的参数值，如果发现存活Broker数量小于参数值，就会直接抛出异常，告诉你主题创建失败。因此，在使用MirrorMaker时，你一定要确保这些配置都是合理的。</p><h2>其他跨集群镜像方案</h2><p>讲到这里，MirrorMaker的主要功能我就介绍完了。你大致可以感觉到执行MirrorMaker的命令是很简单的，而且它提供的功能很有限。实际上，它的运维成本也比较高，比如主题的管理就非常不便捷，同时也很难将其管道化。</p><p>基于这些原因，业界很多公司选择自己开发跨集群镜像工具。我来简单介绍几个。</p><p>1.<strong>Uber的uReplicator工具</strong></p><p>Uber公司之前也是使用MirrorMaker的，但是在使用过程中，他们发现了一些明显的缺陷，比如MirrorMaker中的消费者使用的是消费者组的机制，这不可避免地会碰到很多Rebalance的问题。</p><p>为此，Uber自己研发了uReplicator。它使用Apache Helix作为集中式的主题分区管理组件，并且重写了消费者程序，来替换之前MirrorMaker下的消费者，使用Helix来管理分区的分配，从而避免了Rebalance的各种问题。</p><p>讲到这里，我个人有个小小的感慨：社区最近正在花大力气去优化消费者组机制，力求改善因Rebalance导致的各种场景，但其实，其他框架开发者反而是不用Group机制的。他们宁愿自己开发一套机制来维护分区分配的映射。这些都说明Kafka中的消费者组还是有很大的提升空间的。</p><p>另外，Uber专门写了一篇<a href="https://eng.uber.com/ureplicator/">博客</a>，详细说明了uReplicator的设计原理，并罗列了社区的MirrorMaker工具的一些缺陷以及uReplicator的应对方法。我建议你一定要读一读这篇博客。</p><p>2.<strong>LinkedIn开发的Brooklin Mirror Maker工具</strong></p><p>针对现有MirrorMaker工具不易实现管道化的缺陷，这个工具进行了有针对性的改进，同时也对性能做了一些优化。目前，在LinkedIn公司，Brooklin Mirror Maker已经完全替代了社区版的MirrorMaker。如果你想深入了解它是如何做到的，我给你推荐一篇<a href="https://www.slideshare.net/jyouping/brooklin-mirror-maker-how-and-why-we-moved-away-from-kafka-mirror-maker">博客</a>，你可以详细阅读一下。</p><p>3.<strong>Confluent公司研发的Replicator工具</strong></p><p>这个工具提供的是企业级的跨集群镜像方案，是市面上已知的功能最为强大的工具，可以便捷地为你提供Kafka主题在不同集群间的迁移。除此之外，Replicator工具还能自动在目标集群上创建与源集群上配置一模一样的主题，极大地方便了运维管理。不过凡事有利就有弊，Replicator是要收费的。如果你所在的公司预算充足，而且你们关心数据在多个集群甚至是多个数据中心间的迁移质量，不妨关注一下Confluent公司的<a href="https://www.confluent.io/confluent-replicator/">Replicator工具</a>。</p><h2>小结</h2><p>好了，我们总结一下今天所讲的MirrorMaker。它是Apache Kafka社区提供的跨集群镜像解决方案，主要实现将Kafka消息实时从一个集群同步复制或镜像到另一个集群。你可以把MirrorMaker应用到很多实际的场景中，比如数据备份、主备集群等。MirrorMaker本身功能简单，应用灵活，但也有运维成本高、性能差等劣势，因此业界有厂商研发了自己的镜像工具。你可以根据自身的业务需求，选择合适的工具来帮助你完成跨集群的数据备份。</p><p><img src="https://static001.geekbang.org/resource/image/c0/5e/c0b75891bf37c1086dea98358c481d5e.jpg?wh=2069*2560" alt=""></p><h2>开放讨论</h2><p>今天我只演示了MirrorMaker最基本的使用方法，即把消息原样搬移。如果我们想在消息被镜像前做一些处理，比如修改消息体内容，那么，我们应该如何实现呢？（提示：指定kafka-mirror-maker脚本的 --message.handler参数。）</p><p>欢迎写下你的思考和答案，我们一起讨论。如果你觉得有所收获，也欢迎把文章分享给你的朋友。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/cd/2a/bdbed6ed.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>无菇朋友</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，问一下怎么在本地搭建两个kafka集群</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 设置zookeeper.connect时使用不同的chroot，比如一个是zk:2181&#47;kafka1，另一个是zk:2181&#47;kafka2</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-02 15:34:49</div>
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
  <div class="_2_QraFYR_0">老师，你好，你提的这几款工具，MirrorMaker是免费的，其它的也是免费可以用的么</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: LinkedIn的Brooklin MM好像没有开源，Confluent的Replicator要收费，Uber的uReplicator应该开源了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-22 08:36:06</div>
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
  <div class="_2_QraFYR_0">胡老师，想问一下，MirrorMaker 工具能保证同步前后分区序号，以及分区位移是一样的吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 很难。我指的是实际环境中，至少代价极大。就拿位移来说你就很难保证严格一致。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-15 09:09:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>wgcris</span>
  </div>
  <div class="_2_QraFYR_0">老师，您好，请教个问题，如果使用mirrormaker做集群数据同步，是不是内部topic数据也一起进行同步？另外在主备场景下，如果主集群挂了，如何保证备集群能正常提供服务，客户端消费数据是正确的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 默认是不同步内部topic，如果要同步的话，需要两个条件：在consumer.config中设置exclude.internal.topics=false以及在producer.config中设置client.id=__admin_client。具体参见这个jira：<br>https:&#47;&#47;issues.apache.org&#47;jira&#47;browse&#47;KAFKA-6524<br><br>如果主挂了，client需要自行处理切换的问题，目前Kafka端尚未自动切换的解决方案</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-24 00:13:33</div>
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
  <div class="_2_QraFYR_0">您好，我们最近也在调研brooklin和uReplicator，能否比较下二者的优缺点啊？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-22 08:39:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/67/f4/9a1feb59.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>钱</span>
  </div>
  <div class="_2_QraFYR_0">kafka集群间数据同步工具，MirrorMaker——本质上就是一个消费者+一个生产者程序，利用这一取一放将数据在两个kafka集群中到一道手。不过这个工具性能差一些，也存在其他一些问题，所以，牛一点的公司都在她的基础上开发了自己的集群间数据同步工具。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-24 07:04:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4d/fd/0aa0e39f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>许童童</span>
  </div>
  <div class="_2_QraFYR_0">跟着老师一起精进。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-22 15:21:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/1d/71/4d3d2a62.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>猪哥灰</span>
  </div>
  <div class="_2_QraFYR_0">老师好，想请教一下mirrormaker是否支持两个分别支持不同认证方式的kafka集群，或者两个不同kerberos认证的集群</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: mirrormaker 1是不支持的，可以考虑使用MM2</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-26 20:36:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/3a/6e/e39e90ca.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>大坏狐狸</span>
  </div>
  <div class="_2_QraFYR_0">完了 我咋感觉 用不到这东西。我又怕以后用到了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: MM后面应用的场景还是很多的，加油<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-20 15:32:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>wgcris</span>
  </div>
  <div class="_2_QraFYR_0">老师，您好，再请教个问题最近我们生产环境发现，broker进程意外挂掉之后，重启broker时间会很长，当数据量很大的时候，浏览了一下社区，KIP-263提了一个解决方案，而且已经合入到2.3版本，但测试发现并没有什么效果，不知道您有了解这方面的优化吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个KIP还在讨论中啊，没有进2.3版本。基本上都是因为加载日志段时间过长导致的。目前社区针对这个问题的终极解决方案是KIP-500</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-26 22:11:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/20/d0/ad12060d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>蓝色海洋</span>
  </div>
  <div class="_2_QraFYR_0">老师，我想问问，低版本的集群消息可以同步到高版本集群吗，目前版本是0.8想同步到2.0集群</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 个人觉得使用MirrorMaker似乎也是可以的，只是版本跨度有点大~ 0.8和2.0的消息格式都是不一样的，</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-30 07:59:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/a7/5c/977d714c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>RJZ</span>
  </div>
  <div class="_2_QraFYR_0">用kafka connector实现可以不</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Kafka Connect多是实现Kafka与其他外部系统之间的互连，不是用于Kafka多集群间的数据迁移，而且也没有对应的kafka connector....</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-27 09:58:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/8c/5c/3f164f66.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>亚林</span>
  </div>
  <div class="_2_QraFYR_0">我怎么感觉有点kafka劝退专栏的味道了😭</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-14 09:20:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/16/b9/a8a7a6af.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_ea21df</span>
  </div>
  <div class="_2_QraFYR_0">如果集群使用了认证，MirrorMaker怎么指定用户名和密码？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-05 13:29:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/22/26/23/e8356436.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>夏末?秋初</span>
  </div>
  <div class="_2_QraFYR_0">老师，如果用mirrormaker作为集群间数据迁移工具，迁移后能否保证消费位移一致，前提是源端目的端所有配置、主题、分区数这些都一致，mirrormaker能否达到这种场景下的迁移啊?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以保证，因为mirrormaker也可以迁移__consumer_offsets，只是我严重怀疑在实践中做到这点还是有点技术含量：）<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-04 15:13:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/51/bc/c7514a30.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Dev.lu</span>
  </div>
  <div class="_2_QraFYR_0">如果是用confluent replicator 可以自己build SMT 插件, cstomize transformation</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-01 18:54:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/51/0d/fc1652fe.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>James</span>
  </div>
  <div class="_2_QraFYR_0">--message.handler 执行生产者雷修改消息体吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 就是在consumer消费前对消息进行处理</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-05 14:50:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/a7/5c/977d714c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>RJZ</span>
  </div>
  <div class="_2_QraFYR_0">接上条，只是感觉connect就是来做数据对接场景的😂，另外confluent的复制工具出现在confluent hub里，一度怀疑他们是开发了个connector</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-27 21:58:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/4d/5e/c5c62933.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lmtoo</span>
  </div>
  <div class="_2_QraFYR_0">是不是通过MirrorMaker拷贝，只要是目标集群的主题分区数量和源主题分区数量一致，拷贝的主题分区数据就是一一对应的？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-22 09:54:01</div>
  </div>
</div>
</div>
</li>
</ul>