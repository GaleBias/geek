<audio title="36 _ 你应该怎么监控Kafka？" src="https://static001.geekbang.org/resource/audio/51/08/51aa021a7b8a6ab04426ed50c9e7ae08.mp3" controls="controls"></audio> 
<p>你好，我是胡夕。今天我要和你分享的主题是：如何监控Kafka。</p><p>监控Kafka，历来都是个老大难的问题。无论是在我维护的微信公众号，还是Kafka QQ群里面，大家问得最多的问题，一定是Kafka的监控。大家提问的内容看似五花八门，但真正想了解的，其实都是监控这点事，也就是我应该监控什么，怎么监控。那么今天，我们就来详细聊聊这件事。</p><p>我个人认为，和头疼医头、脚疼医脚的问题类似，在监控Kafka时，如果我们只监控Broker的话，就难免以偏概全。单个Broker启动的进程虽然属于Kafka应用，但它也是一个普通的Java进程，更是一个操作系统进程。因此，我觉得有必要从Kafka主机、JVM和Kafka集群本身这三个维度进行监控。</p><h2>主机监控</h2><p>主机级别的监控，往往是揭示线上问题的第一步。<strong>所谓主机监控，指的是监控Kafka集群Broker所在的节点机器的性能</strong>。通常来说，一台主机上运行着各种各样的应用进程，这些进程共同使用主机上的所有硬件资源，比如CPU、内存或磁盘等。</p><p>常见的主机监控指标包括但不限于以下几种：</p><ul>
<li>机器负载（Load）</li>
<li>CPU使用率</li>
<li>内存使用率，包括空闲内存（Free Memory）和已使用内存（Used Memory）</li>
<li>磁盘I/O使用率，包括读使用率和写使用率</li>
<li>网络I/O使用率</li>
<li>TCP连接数</li>
<li>打开文件数</li>
<li>inode使用情况</li>
</ul><!-- [[[read_end]]] --><p>考虑到我们并不是要系统地学习调优与监控主机性能，因此我并不打算对上面的每一个指标都进行详细解释，我重点分享一下机器负载和CPU使用率的监控方法。我会以Linux平台为例来进行说明，其他平台应该也是类似的。</p><p>首先，我们来看一张图片。我在Kafka集群的某台Broker所在的主机上运行top命令，输出的内容如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/00/e0/00f0ead463b17e667d09b6cea4e42de0.png?wh=1950*1546" alt=""></p><p>在图片的右上角，我们可以看到load average的3个值：4.85，2.76和1.26，它们分别代表过去1分钟、过去5分钟和过去15分钟的Load平均值。在这个例子中，我的主机总共有4个CPU核，但Load值却达到了4.85，这就说明，一定有进程暂时“抢不到”任何CPU资源。同时，Load值一直在增加，也说明这台主机上的负载越来越大。</p><p>举这个例子，其实我真正想说的是CPU使用率。很多人把top命令中“%CPU”列的输出值当作CPU使用率。比如，在上面这张图中，PID为2637的Java进程是Broker进程，它对应的“%CPU”的值是102.3。你不要认为这是CPU的真实使用率，这列值的真实含义是进程使用的所有CPU的平均使用率，只是top命令在显示的时候转换成了单个CPU。因此，如果是在多核的主机上，这个值就可能会超过100。在这个例子中，我的主机有4个CPU核，总CPU使用率是102.3，那么，平均每个CPU的使用率大致是25%。</p><h2>JVM监控</h2><p>除了主机监控之外，另一个重要的监控维度就是JVM监控。Kafka Broker进程是一个普通的Java进程，所有关于JVM的监控手段在这里都是适用的。</p><p>监控JVM进程主要是为了让你全面地了解你的应用程序（Know Your Application）。具体到Kafka而言，就是全面了解Broker进程。比如，Broker进程的堆大小（HeapSize）是多少、各自的新生代和老年代是多大？用的是什么GC回收器？这些监控指标和配置参数林林总总，通常你都不必全部重点关注，但你至少要搞清楚Broker端JVM进程的Minor GC和Full GC的发生频率和时长、活跃对象的总大小和JVM上应用线程的大致总数，因为这些数据都是你日后调优Kafka Broker的重要依据。</p><p>我举个简单的例子。假设一台主机上运行的Broker进程在经历了一次Full GC之后，堆上存活的活跃对象大小是700MB，那么在实际场景中，你几乎可以安全地将老年代堆大小设置成该数值的1.5倍或2倍，即大约1.4GB。不要小看700MB这个数字，它是我们设定Broker堆大小的重要依据！</p><p>很多人会有这样的疑问：我应该怎么设置Broker端的堆大小呢？其实，这就是最合理的评估方法。试想一下，如果你的Broker在Full GC之后存活了700MB的数据，而你设置了堆大小为16GB，这样合理吗？对一个16GB大的堆执行一次GC要花多长时间啊？！</p><p>因此，我们来总结一下。要做到JVM进程监控，有3个指标需要你时刻关注：</p><ol>
<li>Full GC发生频率和时长。这个指标帮助你评估Full GC对Broker进程的影响。长时间的停顿会令Broker端抛出各种超时异常。</li>
<li>活跃对象大小。这个指标是你设定堆大小的重要依据，同时它还能帮助你细粒度地调优JVM各个代的堆大小。</li>
<li>应用线程总数。这个指标帮助你了解Broker进程对CPU的使用情况。</li>
</ol><p>总之，你对Broker进程了解得越透彻，你所做的JVM调优就越有效果。</p><p>谈到具体的监控，前两个都可以通过GC日志来查看。比如，下面的这段GC日志就说明了GC后堆上的存活对象大小。</p><blockquote>
<p>2019-07-30T09:13:03.809+0800: 552.982: [GC cleanup 827M-&gt;645M(1024M), 0.0019078 secs]</p>
</blockquote><p>这个Broker JVM进程默认使用了G1的GC算法，当cleanup步骤结束后，堆上活跃对象大小从827MB缩减成645MB。另外，你可以根据前面的时间戳来计算每次GC的间隔和频率。</p><p>自0.9.0.0版本起，社区将默认的GC收集器设置为G1，而G1中的Full GC是由单线程执行的，速度非常慢。因此，<strong>你一定要监控你的Broker GC日志，即以kafkaServer-gc.log开头的文件</strong>。注意不要出现Full GC的字样。一旦你发现Broker进程频繁Full GC，可以开启G1的-XX:+PrintAdaptiveSizePolicy开关，让JVM告诉你到底是谁引发了Full GC。</p><h2>集群监控</h2><p>说完了主机和JVM监控，现在我来给出监控Kafka集群的几个方法。</p><p><strong>1.查看Broker进程是否启动，端口是否建立。</strong></p><p>千万不要小看这一点。在很多容器化的Kafka环境中，比如使用Docker启动Kafka Broker时，容器虽然成功启动了，但是里面的网络设置如果配置有误，就可能会出现进程已经启动但端口未成功建立监听的情形。因此，你一定要同时检查这两点，确保服务正常运行。</p><p><strong>2.查看Broker端关键日志。</strong></p><p>这里的关键日志，主要涉及Broker端服务器日志server.log，控制器日志controller.log以及主题分区状态变更日志state-change.log。其中，server.log是最重要的，你最好时刻对它保持关注。很多Broker端的严重错误都会在这个文件中被展示出来。因此，如果你的Kafka集群出现了故障，你要第一时间去查看对应的server.log，寻找和定位故障原因。</p><p><strong>3.查看Broker端关键线程的运行状态。</strong></p><p>这些关键线程的意外挂掉，往往无声无息，但是却影响巨大。比方说，Broker后台有个专属的线程执行Log Compaction操作，由于源代码的Bug，这个线程有时会无缘无故地“死掉”，社区中很多Jira都曾报出过这个问题。当这个线程挂掉之后，作为用户的你不会得到任何通知，Kafka集群依然会正常运转，只是所有的Compaction操作都不能继续了，这会导致Kafka内部的位移主题所占用的磁盘空间越来越大。因此，我们有必要对这些关键线程的状态进行监控。</p><p>可是，一个Kafka Broker进程会启动十几个甚至是几十个线程，我们不可能对每个线程都做到实时监控。所以，我跟你分享一下我认为最重要的两类线程。在实际生产环境中，监控这两类线程的运行情况是非常有必要的。</p><ul>
<li>Log Compaction线程，这类线程是以kafka-log-cleaner-thread开头的。就像前面提到的，此线程是做日志Compaction的。一旦它挂掉了，所有Compaction操作都会中断，但用户对此通常是无感知的。</li>
<li>副本拉取消息的线程，通常以ReplicaFetcherThread开头。这类线程执行Follower副本向Leader副本拉取消息的逻辑。如果它们挂掉了，系统会表现为对应的Follower副本不再从Leader副本拉取消息，因而Follower副本的Lag会越来越大。</li>
</ul><p>不论你是使用jstack命令，还是其他的监控框架，我建议你时刻关注Broker进程中这两类线程的运行状态。一旦发现它们状态有变，就立即查看对应的Kafka日志，定位原因，因为这通常都预示会发生较为严重的错误。</p><p><strong>4.查看Broker端的关键JMX指标。</strong></p><p>Kafka提供了超多的JMX指标供用户实时监测，我来介绍几个比较重要的Broker端JMX指标：</p><ul>
<li>BytesIn/BytesOut：即Broker端每秒入站和出站字节数。你要确保这组值不要接近你的网络带宽，否则这通常都表示网卡已被“打满”，很容易出现网络丢包的情形。</li>
<li>NetworkProcessorAvgIdlePercent：即网络线程池线程平均的空闲比例。通常来说，你应该确保这个JMX值长期大于30%。如果小于这个值，就表明你的网络线程池非常繁忙，你需要通过增加网络线程数或将负载转移给其他服务器的方式，来给该Broker减负。</li>
<li>RequestHandlerAvgIdlePercent：即I/O线程池线程平均的空闲比例。同样地，如果该值长期小于30%，你需要调整I/O线程池的数量，或者减少Broker端的负载。</li>
<li>UnderReplicatedPartitions：即未充分备份的分区数。所谓未充分备份，是指并非所有的Follower副本都和Leader副本保持同步。一旦出现了这种情况，通常都表明该分区有可能会出现数据丢失。因此，这是一个非常重要的JMX指标。</li>
<li>ISRShrink/ISRExpand：即ISR收缩和扩容的频次指标。如果你的环境中出现ISR中副本频繁进出的情形，那么这组值一定是很高的。这时，你要诊断下副本频繁进出ISR的原因，并采取适当的措施。</li>
<li>ActiveControllerCount：即当前处于激活状态的控制器的数量。正常情况下，Controller所在Broker上的这个JMX指标值应该是1，其他Broker上的这个值是0。如果你发现存在多台Broker上该值都是1的情况，一定要赶快处理，处理方式主要是查看网络连通性。这种情况通常表明集群出现了脑裂。脑裂问题是非常严重的分布式故障，Kafka目前依托ZooKeeper来防止脑裂。但一旦出现脑裂，Kafka是无法保证正常工作的。</li>
</ul><p>其实，Broker端还有很多很多JMX指标，除了上面这些重要指标，你还可以根据自己业务的需要，去官网查看其他JMX指标，把它们集成进你的监控框架。</p><p><strong>5.监控Kafka客户端。</strong></p><p>客户端程序的性能同样需要我们密切关注。不管是生产者还是消费者，我们首先要关心的是客户端所在的机器与Kafka Broker机器之间的<strong>网络往返时延</strong>（Round-Trip Time，RTT）。通俗点说，就是你要在客户端机器上ping一下Broker主机IP，看看RTT是多少。</p><p>我曾经服务过一个客户，他的Kafka生产者TPS特别低。我登到机器上一看，发现RTT是1秒。在这种情况下，无论你怎么调优Kafka参数，效果都不会太明显，降低网络时延反而是最直接有效的办法。</p><p>除了RTT，客户端程序也有非常关键的线程需要你时刻关注。对于生产者而言，有一个以kafka-producer-network-thread开头的线程是你要实时监控的。它是负责实际消息发送的线程。一旦它挂掉了，Producer将无法正常工作，但你的Producer进程不会自动挂掉，因此你有可能感知不到。对于消费者而言，心跳线程事关Rebalance，也是必须要监控的一个线程。它的名字以kafka-coordinator-heartbeat-thread开头。</p><p>除此之外，客户端有一些很重要的JMX指标，可以实时告诉你它们的运行情况。</p><p>从Producer角度，你需要关注的JMX指标是request-latency，即消息生产请求的延时。这个JMX最直接地表征了Producer程序的TPS；而从Consumer角度来说，records-lag和records-lead是两个重要的JMX指标。我们在专栏<a href="https://time.geekbang.org/column/article/109238">第22讲</a>解释过这两个指标的含义，这里我就不再赘述了。总之，它们直接反映了Consumer的消费进度。如果你使用了Consumer Group，那么有两个额外的JMX指标需要你关注下，一个是join rate，另一个是sync rate。它们说明了Rebalance的频繁程度。如果在你的环境中，它们的值很高，那么你就需要思考下Rebalance频繁发生的原因了。</p><h2>小结</h2><p>好了，我们来小结一下。今天，我介绍了监控Kafka的方方面面。除了监控Kafka集群，我还推荐你从主机和JVM的维度进行监控。对主机的监控，往往是我们定位和发现问题的第一步。JVM监控同样重要。要知道，很多Java进程碰到的性能问题是无法通过调整Kafka参数是解决的。最后，我罗列了一些比较重要的Kafka JMX指标。在下一讲中，我会专门介绍一下如何使用各种工具来查看这些JMX指标。</p><p><img src="https://static001.geekbang.org/resource/image/28/93/28e6d8c2459b5d123f443173ac122c93.jpg?wh=2069*2560" alt=""></p><h2>开放讨论</h2><p>请分享一下你在监控Kafka方面的心得，以及你的运维技巧。</p><p>欢迎写下你的思考和答案，我们一起讨论。如果你觉得有所收获，也欢迎把文章分享给你的朋友。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/d0/42/6fd01fb9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我已经设置了昵称</span>
  </div>
  <div class="_2_QraFYR_0">要怎么看到JMX指标呢，能否讲下</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 无论是Broker端还是Clients端启动前要先设置JMX_PORT，然后使用任何能够连接JMX MBean Server的工具或框架连接（如JConsole）就能看到了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-04 09:03:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/7c/b0/0ee17e1b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>r</span>
  </div>
  <div class="_2_QraFYR_0">老师总结的真好。我有个疑问，没找到相关资料做支撑。就是一套kafka集群，最多能容纳多少个topic-partition，这个是集群规模有关吗，</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 根据社区的报告，Kafka 1.1.0之后可以支持单集群20万个分区。和集群规模不能说没有关系，但其实和集群总的物理硬件资源有很大关系。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-24 17:55:39</div>
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
  <div class="_2_QraFYR_0">感觉离开平台自己真的什么都不是，公司内部的监控挺全的，单机的CPU&#47;硬盘&#47;内存&#47;网络&#47;jvm等都有，也有针对方法级别的性能&#47;可用率&#47;调用次数，针对MQ有流入&#47;流出&#47;积压等，这里的每个监控工具都有专门的团队来负责，分工比较细，现在想一想业务开发，如果对业务不精通真是没有什么存在感和价值的。<br>感觉监控最大的痛点是怎么获取到对应的监控信息，只要能获取监控信息，剩下的就是怎么聚合和汇总展示的问题了。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-24 07:54:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/4e/29/1be3dd40.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ykkk88</span>
  </div>
  <div class="_2_QraFYR_0">有什么好的开源的监控工具么 </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我觉得Kafka Manager就挺不错的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-25 21:18:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/df/e5/65e37812.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>快跑</span>
  </div>
  <div class="_2_QraFYR_0">请教老师一下<br>从监控上能看到读取kafka数据是从页缓存还是磁盘么，对应的指标有哪些？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 无法看出。不过你可以监控一下broker的磁盘IO，对于那些同步的consumer而言，磁盘IO读应该很少才对</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-14 23:08:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/W1qXe7yEB8C9fsossNLH59OrNBrEhxnibaMNfKro6YtKyL3thNN3AMyGyme2el0IgzwGpiaycFwwSvKLINITjhzA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>frenco</span>
  </div>
  <div class="_2_QraFYR_0">老师好， 请教个问题：    按您之前有个推荐的配置kafka内存的说法，一般堆内存配置6G就好了。 那新生代和老年代默认2：1  分配。      如果只需要6G的内存，  我们生产的机器一般都是64G以上内存， 那机器是不是有很大浪费呢。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 那就单台多broker吧，不过网卡最好万兆</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-08 09:31:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/c5/a7/e32dcfe7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>谦寻</span>
  </div>
  <div class="_2_QraFYR_0">请教下老师，我们最近遇到一个监控问题，监控各个topic的消息堆积，发现如果业务方由于服务下线，不使用某个consume group了，结果这个group的消息堆积会一直增加，运维就会收到监控告警，但是运维并不好判断哪个group已经不使用了，这个能有什么自动化的手段吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果group不使用了，它的状态就是nonactive了，一段时间之后Kafka会自动删除的它数据。如果判断状态的话，新一点版本的Kafka可以使用kafka-consumer-groups --describe --group *** 来查看group状态。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-29 19:40:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/99/1a/53ed3004.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>wxr</span>
  </div>
  <div class="_2_QraFYR_0">怎样比较好的监控消费延时呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个取决于你对消费延时的定义。从Kafka的角度，当poll方法返回后，消息已经算是被消费了，但通常我们获取到消息后还要对消息进行处理，如果你认为处理完成后才算是消费就要加上这部分的时间，但处理逻辑、工具、方法都不尽相同，因此你需要自己来监控消息处理的总时间。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-24 10:35:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/8f/35/f1839bb2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>风中花</span>
  </div>
  <div class="_2_QraFYR_0">老师你的公众号怎么找到呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 大数据Kafka技术分享</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-30 13:37:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTIFrA5ztRGqQTFYIMoBVFgvlhH8GZOCj0K6QLhddcACsugr3BABZdWdSrNobhAWcuEb1W1vS2yicDg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_72a3d3</span>
  </div>
  <div class="_2_QraFYR_0">“同时，Load 值一直在增加，也说明这台主机上的负载越来越大。”<br>老师，您好，Load值好像是越来越小。？？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 3个值的排序是过去1分钟，5分钟和15分钟，因此表明load越来越大</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-17 09:36:13</div>
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
  <div class="_2_QraFYR_0">你好，单个topic可以支撑的最多partition个数多少啊？我们生产上有个topic超级大，占了整个集群的一半以上的流量，这种情况是需要拆分吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果性能okay而仅仅是你觉得不太好，那么我认为先不用拆分。单个topic最多能有多少partition没有定数，主要还是看底层物理资源。当然分区数过多，使得broker上平均分区数增加的确会降低Kafka的TPS。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-24 09:21:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/62/30/a8df1a4e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张亮</span>
  </div>
  <div class="_2_QraFYR_0">Kafka监控是一个非常专业和体系化的事情，Elasticearch基本将系统指标、JVM指标作为Metric上报出来自闭环非常方便实用，在开源Logi-KafkaManager的时候，我一直计划将这些指标通过JMX直接暴露出来，你怎么看？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我觉得可行：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-15 10:58:18</div>
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
  <div class="_2_QraFYR_0">这应该是最有水平的一篇文章了，经验值超高</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 过奖了~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-07 23:26:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/86/d7/33d628b1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>夏日</span>
  </div>
  <div class="_2_QraFYR_0">ttl一般多少以内比较正常，比如在考虑在双活中心搭建一套kafka集群的时候，怎么判断不会由于节点之间的传输延时导致kafka性能不高？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 通常ttl超过500ms就要关注下了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-23 15:10:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/47/1b/64262861.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>胡小禾</span>
  </div>
  <div class="_2_QraFYR_0">“如果group不使用了，它的状态就是nonactive了”<br><br>这个nonactive 在ZK上是不是有节点？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 目前Kafka的consumer group完全不使用ZooKeeper来保存元数据了，因此无论任何状态的group在ZK上都没有节点了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-28 23:41:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/bf/69/dbfd10f7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>追光者</span>
  </div>
  <div class="_2_QraFYR_0">老师，您好，想请教一个关于  Metricbeat 采集 kafka 数据的问题：<br>配置好 modules.d&#47;kafka.yml 启动 metricbeat 采集不到数据，提示信息：<br>2019-08-29T16:13:33.827+0800    INFO    kafka&#47;log.go:53 kafka message: Successful SASL handshake<br>2019-08-29T16:13:33.828+0800    INFO    kafka&#47;log.go:53 SASL authentication successful with broker 10.162.7.2:9092:4 - [0 0 0 0]<br>2019-08-29T16:13:33.828+0800    INFO    kafka&#47;log.go:53 Connected to broker at 10.162.7.2:9092 (unregistered)<br>2019-08-29T16:13:33.832+0800    INFO    kafka&#47;log.go:53 Closed connection to broker 10.162.7.2:9092<br>system 的可以采集到，请问这是什么原因呀<br>配置文件：<br>- module: kafka<br>metricsets:<br>- partition<br>- consumergroup<br>period: 10s<br>hosts: [&quot;10.162.3.90:9092&quot;]<br>client_id: xl<br>retries: 3<br>backoff: 250ms<br>topics: []<br>username: &quot;admin&quot;<br>password: &quot;admin&quot;</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这里都是IINFO日志看不出有什么问题，有其他日志吗？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-30 09:34:22</div>
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
  <div class="_2_QraFYR_0">这些指标经验很有用</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-14 11:01:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTI34ZlT6HSOtJBeTvTvfNLfYECDdJXnHCMj2BHdrRaqRLnZiafnxmKQ2aXoQkW1RLQOyt0tlyzEWIA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ahu0605</span>
  </div>
  <div class="_2_QraFYR_0">胡老师，您对kafka部署k8s中有什么建议吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-21 13:12:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/5b/aa/777d7f88.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>谁谁</span>
  </div>
  <div class="_2_QraFYR_0">老师，tps不是应该包括ttl？从客户端发送请求到服务端处理完成返回，文中为什么说tps小而ttl大呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这两个概念没有直接的关联吧</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-13 10:41:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/d2/68/2149f518.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Rosy</span>
  </div>
  <div class="_2_QraFYR_0">kafka会频繁地删掉broker，导致频繁地切换leader，这是什么情况呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 能详细解释下”删掉broker”的含义吗？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-23 11:03:04</div>
  </div>
</div>
</div>
</li>
</ul>