<audio title="38 _ 调优Kafka，你做到了吗？" src="https://static001.geekbang.org/resource/audio/33/10/33db1bd2ffd88dbe7b57194ecf0c7410.mp3" controls="controls"></audio> 
<p>你好，我是胡夕。今天我要和你分享的主题是：如何调优Kafka。</p><h2>调优目标</h2><p>在做调优之前，我们必须明确优化Kafka的目标是什么。通常来说，调优是为了满足系统常见的非功能性需求。在众多的非功能性需求中，性能绝对是我们最关心的那一个。不同的系统对性能有不同的诉求，比如对于数据库用户而言，性能意味着请求的响应时间，用户总是希望查询或更新请求能够被更快地处理完并返回。</p><p>对Kafka而言，性能一般是指吞吐量和延时。</p><p>吞吐量，也就是TPS，是指Broker端进程或Client端应用程序每秒能处理的字节数或消息数，这个值自然是越大越好。</p><p>延时和我们刚才说的响应时间类似，它表示从Producer端发送消息到Broker端持久化完成之间的时间间隔。这个指标也可以代表端到端的延时（End-to-End，E2E），也就是从Producer发送消息到Consumer成功消费该消息的总时长。和TPS相反，我们通常希望延时越短越好。</p><p>总之，高吞吐量、低延时是我们调优Kafka集群的主要目标，一会儿我们会详细讨论如何达成这些目标。在此之前，我想先谈一谈优化漏斗的问题。</p><h2>优化漏斗</h2><p>优化漏斗是一个调优过程中的分层漏斗，我们可以在每一层上执行相应的优化调整。总体来说，层级越靠上，其调优的效果越明显，整体优化效果是自上而下衰减的，如下图所示：</p><!-- [[[read_end]]] --><p><img src="https://static001.geekbang.org/resource/image/94/59/94486dc0eb55b68855478ef7e5709359.png?wh=1950*1248" alt=""></p><p><strong>第1层：应用程序层</strong>。它是指优化Kafka客户端应用程序代码。比如，使用合理的数据结构、缓存计算开销大的运算结果，抑或是复用构造成本高的对象实例等。这一层的优化效果最为明显，通常也是比较简单的。</p><p><strong>第2层：框架层</strong>。它指的是合理设置Kafka集群的各种参数。毕竟，直接修改Kafka源码进行调优并不容易，但根据实际场景恰当地配置关键参数的值，还是很容易实现的。</p><p><strong>第3层：JVM层</strong>。Kafka Broker进程是普通的JVM进程，各种对JVM的优化在这里也是适用的。优化这一层的效果虽然比不上前两层，但有时也能带来巨大的改善效果。</p><p><strong>第4层：操作系统层</strong>。对操作系统层的优化很重要，但效果往往不如想象得那么好。与应用程序层的优化效果相比，它是有很大差距的。</p><h2>基础性调优</h2><p>接下来，我就来分别介绍一下优化漏斗的4个分层的调优。</p><h3>操作系统调优</h3><p>我先来说说操作系统层的调优。在操作系统层面，你最好在挂载（Mount）文件系统时禁掉atime更新。atime的全称是access time，记录的是文件最后被访问的时间。记录atime需要操作系统访问inode资源，而禁掉atime可以避免inode访问时间的写入操作，减少文件系统的写操作数。你可以执行<strong>mount -o noatime命令</strong>进行设置。</p><p>至于文件系统，我建议你至少选择ext4或XFS。尤其是XFS文件系统，它具有高性能、高伸缩性等特点，特别适用于生产服务器。值得一提的是，在去年10月份的Kafka旧金山峰会上，有人分享了ZFS搭配Kafka的案例，我们在专栏<a href="https://time.geekbang.org/column/article/101763">第8讲</a>提到过与之相关的<a href="https://www.confluent.io/kafka-summit-sf18/kafka-on-zfs">数据报告</a>。该报告宣称ZFS多级缓存的机制能够帮助Kafka改善I/O性能，据说取得了不错的效果。如果你的环境中安装了ZFS文件系统，你可以尝试将Kafka搭建在ZFS文件系统上。</p><p>另外就是swap空间的设置。我个人建议将swappiness设置成一个很小的值，比如1～10之间，以防止Linux的OOM Killer开启随意杀掉进程。你可以执行sudo sysctl vm.swappiness=N来临时设置该值，如果要永久生效，可以修改/etc/sysctl.conf文件，增加vm.swappiness=N，然后重启机器即可。</p><p>操作系统层面还有两个参数也很重要，它们分别是<strong>ulimit -n和vm.max_map_count</strong>。前者如果设置得太小，你会碰到Too Many File Open这类的错误，而后者的值如果太小，在一个主题数超多的Broker机器上，你会碰到<strong>OutOfMemoryError：Map failed</strong>的严重错误，因此，我建议在生产环境中适当调大此值，比如将其设置为655360。具体设置方法是修改/etc/sysctl.conf文件，增加vm.max_map_count=655360，保存之后，执行sysctl -p命令使它生效。</p><p>最后，不得不提的就是操作系统页缓存大小了，这对Kafka而言至关重要。在某种程度上，我们可以这样说：给Kafka预留的页缓存越大越好，最小值至少要容纳一个日志段的大小，也就是Broker端参数log.segment.bytes的值。该参数的默认值是1GB。预留出一个日志段大小，至少能保证Kafka可以将整个日志段全部放入页缓存，这样，消费者程序在消费时能直接命中页缓存，从而避免昂贵的物理磁盘I/O操作。</p><h3>JVM层调优</h3><p>说完了操作系统层面的调优，我们来讨论下JVM层的调优，其实，JVM层的调优，我们还是要重点关注堆设置以及GC方面的性能。</p><p>1.设置堆大小。</p><p>如何为Broker设置堆大小，这是很多人都感到困惑的问题。我来给出一个朴素的答案：<strong>将你的JVM堆大小设置成6～8GB</strong>。</p><p>在很多公司的实际环境中，这个大小已经被证明是非常合适的，你可以安心使用。如果你想精确调整的话，我建议你可以查看GC log，特别是关注Full GC之后堆上存活对象的总大小，然后把堆大小设置为该值的1.5～2倍。如果你发现Full GC没有被执行过，手动运行jmap -histo:live &lt; pid &gt;就能人为触发Full GC。</p><p>2.GC收集器的选择。</p><p><strong>我强烈建议你使用G1收集器，主要原因是方便省事，至少比CMS收集器的优化难度小得多</strong>。另外，你一定要尽力避免Full GC的出现。其实，不论使用哪种收集器，都要竭力避免Full GC。在G1中，Full GC是单线程运行的，它真的非常慢。如果你的Kafka环境中经常出现Full GC，你可以配置JVM参数-XX:+PrintAdaptiveSizePolicy，来探查一下到底是谁导致的Full GC。</p><p>使用G1还很容易碰到的一个问题，就是<strong>大对象（Large Object）</strong>，反映在GC上的错误，就是“too many humongous allocations”。所谓的大对象，一般是指至少占用半个区域（Region）大小的对象。举个例子，如果你的区域尺寸是2MB，那么超过1MB大小的对象就被视为是大对象。要解决这个问题，除了增加堆大小之外，你还可以适当地增加区域大小，设置方法是增加JVM启动参数-XX:+G1HeapRegionSize=N。默认情况下，如果一个对象超过了N/2，就会被视为大对象，从而直接被分配在大对象区。如果你的Kafka环境中的消息体都特别大，就很容易出现这种大对象分配的问题。</p><h3>Broker端调优</h3><p>我们继续沿着漏斗往上走，来看看Broker端的调优。</p><p>Broker端调优很重要的一个方面，就是合理地设置Broker端参数值，以匹配你的生产环境。不过，后面我们在讨论具体的调优目标时再详细说这部分内容。这里我想先讨论另一个优化手段，<strong>即尽力保持客户端版本和Broker端版本一致</strong>。不要小看版本间的不一致问题，它会令Kafka丧失很多性能收益，比如Zero Copy。下面我用一张图来说明一下。</p><p><img src="https://static001.geekbang.org/resource/image/53/6e/5310d7d29235b080c872e0a9eb396e6e.png?wh=816*388" alt=""></p><p>图中蓝色的Producer、Consumer和Broker的版本是相同的，它们之间的通信可以享受Zero Copy的快速通道；相反，一个低版本的Consumer程序想要与Producer、Broker交互的话，就只能依靠JVM堆中转一下，丢掉了快捷通道，就只能走慢速通道了。因此，在优化Broker这一层时，你只要保持服务器端和客户端版本的一致，就能获得很多性能收益了。</p><h3>应用层调优</h3><p>现在，我们终于来到了漏斗的最顶层。其实，这一层的优化方法各异，毕竟每个应用程序都是不一样的。不过，有一些公共的法则依然是值得我们遵守的。</p><ul>
<li><strong>不要频繁地创建Producer和Consumer对象实例</strong>。构造这些对象的开销很大，尽量复用它们。</li>
<li><strong>用完及时关闭</strong>。这些对象底层会创建很多物理资源，如Socket连接、ByteBuffer缓冲区等。不及时关闭的话，势必造成资源泄露。</li>
<li><strong>合理利用多线程来改善性能</strong>。Kafka的Java Producer是线程安全的，你可以放心地在多个线程中共享同一个实例；而Java Consumer虽不是线程安全的，但我们在专栏<a href="https://time.geekbang.org/column/article/108512">第20讲</a>讨论过多线程的方案，你可以回去复习一下。</li>
</ul><h2>性能指标调优</h2><p>接下来，我会给出调优各个目标的参数配置以及具体的配置原因，希望它们能够帮助你更有针对性地调整你的Kafka集群。</p><h3>调优吞吐量</h3><p>首先是调优吞吐量。很多人对吞吐量和延时之间的关系似乎有些误解。比如有这样一种提法还挺流行的：假设Kafka每发送一条消息需要花费2ms，那么延时就是2ms。显然，吞吐量就应该是500条/秒，因为1秒可以发送1 / 0.002 = 500条消息。因此，吞吐量和延时的关系可以用公式来表示：TPS = 1000 / Latency(ms)。但实际上，吞吐量和延时的关系远不是这么简单。</p><p>我们以Kafka Producer为例。假设它以2ms的延时来发送消息，如果每次只是发送一条消息，那么TPS自然就是500条/秒。但如果Producer不是每次发送一条消息，而是在发送前等待一段时间，然后统一发送<strong>一批</strong>消息，比如Producer每次发送前先等待8ms，8ms之后，Producer共缓存了1000条消息，此时总延时就累加到10ms（即 2ms + 8ms）了，而TPS等于1000 / 0.01 = 100,000条/秒。由此可见，虽然延时增加了4倍，但TPS却增加了将近200倍。这其实也是批次化（batching）或微批次化（micro-batching）目前会很流行的原因。</p><p>在实际环境中，用户似乎总是愿意用较小的延时增加的代价，去换取TPS的显著提升。毕竟，从2ms到10ms的延时增加通常是可以忍受的。事实上，Kafka Producer就是采取了这样的设计思想。</p><p>当然，你可能会问：发送一条消息需要2ms，那么等待8ms就能累积1000条消息吗？答案是可以的！Producer累积消息时，一般仅仅是将消息发送到内存中的缓冲区，而发送消息却需要涉及网络I/O传输。内存操作和I/O操作的时间量级是不同的，前者通常是几百纳秒级别，而后者则是从毫秒到秒级别不等，因此，Producer等待8ms积攒出的消息数，可能远远多于同等时间内Producer能够发送的消息数。</p><p>好了，说了这么多，我们该怎么调优TPS呢？我来跟你分享一个参数列表。</p><p><img src="https://static001.geekbang.org/resource/image/7a/cb/7aec00207dc149bd804d20df6e3b9ccb.jpg?wh=1713*1983" alt=""></p><p>我稍微解释一下表格中的内容。</p><p>Broker端参数num.replica.fetchers表示的是Follower副本用多少个线程来拉取消息，默认使用1个线程。如果你的Broker端CPU资源很充足，不妨适当调大该参数值，加快Follower副本的同步速度。因为在实际生产环境中，<strong>配置了acks=all的Producer程序吞吐量被拖累的首要因素，就是副本同步性能</strong>。增加这个值后，你通常可以看到Producer端程序的吞吐量增加。</p><p>另外需要注意的，就是避免经常性的Full GC。目前不论是CMS收集器还是G1收集器，其Full GC采用的是Stop The World的单线程收集策略，非常慢，因此一定要避免。</p><p><strong>在Producer端，如果要改善吞吐量，通常的标配是增加消息批次的大小以及批次缓存时间，即batch.size和linger.ms</strong>。目前它们的默认值都偏小，特别是默认的16KB的消息批次大小一般都不适用于生产环境。假设你的消息体大小是1KB，默认一个消息批次也就大约16条消息，显然太小了。我们还是希望Producer能一次性发送更多的消息。</p><p>除了这两个，你最好把压缩算法也配置上，以减少网络I/O传输量，从而间接提升吞吐量。当前，和Kafka适配最好的两个压缩算法是<strong>LZ4和zstd</strong>，不妨一试。</p><p>同时，由于我们的优化目标是吞吐量，最好不要设置acks=all以及开启重试。前者引入的副本同步时间通常都是吞吐量的瓶颈，而后者在执行过程中也会拉低Producer应用的吞吐量。</p><p>最后，如果你在多个线程中共享一个Producer实例，就可能会碰到缓冲区不够用的情形。倘若频繁地遭遇TimeoutException：Failed to allocate memory within the configured max blocking time这样的异常，那么你就必须显式地增加<strong>buffer.memory</strong>参数值，确保缓冲区总是有空间可以申请的。</p><p>说完了Producer端，我们来说说Consumer端。Consumer端提升吞吐量的手段是有限的，你可以利用多线程方案增加整体吞吐量，也可以增加fetch.min.bytes参数值。默认是1字节，表示只要Kafka Broker端积攒了1字节的数据，就可以返回给Consumer端，这实在是太小了。我们还是让Broker端一次性多返回点数据吧。</p><h3>调优延时</h3><p>讲完了调优吞吐量，我们来说说如何优化延时，下面是调优延时的参数列表。</p><p><img src="https://static001.geekbang.org/resource/image/26/3a/2688329a0614601fed497f3858c98e3a.jpg?wh=1803*1083" alt=""></p><p>在Broker端，我们依然要增加num.replica.fetchers值以加快Follower副本的拉取速度，减少整个消息处理的延时。</p><p>在Producer端，我们希望消息尽快地被发送出去，因此不要有过多停留，所以必须设置linger.ms=0，同时不要启用压缩。因为压缩操作本身要消耗CPU时间，会增加消息发送的延时。另外，最好不要设置acks=all。我们刚刚在前面说过，Follower副本同步往往是降低Producer端吞吐量和增加延时的首要原因。</p><p>在Consumer端，我们保持fetch.min.bytes=1即可，也就是说，只要Broker端有能返回的数据，立即令其返回给Consumer，缩短Consumer消费延时。</p><h2>小结</h2><p>好了，我们来小结一下。今天，我跟你分享了Kafka调优方面的内容。我们先从调优目标开始说起，然后我给出了调优层次漏斗，接着我分享了一些基础性调优，包括操作系统层调优、JVM层调优以及应用程序调优等。最后，针对Kafka关心的两个性能指标吞吐量和延时，我分别从Broker、Producer和Consumer三个维度给出了一些参数值设置的最佳实践。</p><p>最后，我来分享一个性能调优的真实小案例。</p><p>曾经，我碰到过一个线上环境的问题：该集群上Consumer程序一直表现良好，但是某一天，它的性能突然下降，表现为吞吐量显著降低。我在查看磁盘读I/O使用率时，发现其明显上升，但之前该Consumer Lag很低，消息读取应该都能直接命中页缓存。此时磁盘读突然飙升，我就怀疑有其他程序写入了页缓存。后来经过排查，我发现果然有一个测试Console Consumer程序启动，“污染”了部分页缓存，导致主业务Consumer读取消息不得不走物理磁盘，因此吞吐量下降。找到了真实原因，解决起来就简单多了。</p><p>其实，我给出这个案例的真实目的是想说，对于性能调优，我们最好按照今天给出的步骤一步一步地窄化和定位问题。一旦定位了原因，后面的优化就水到渠成了。</p><p><img src="https://static001.geekbang.org/resource/image/d4/38/d40b07ca7bcc0fc43c5266b0b2c81c38.jpg?wh=2069*2560" alt=""></p><h2>开放讨论</h2><p>请分享一个你调优Kafka的真实案例，详细说说你是怎么碰到性能问题的，又是怎么解决的。</p><p>欢迎写下你的思考和答案，我们一起讨论。如果你觉得有所收获，也欢迎把文章分享给你的朋友。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/be/b9/f2481c2c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>诗泽</span>
  </div>
  <div class="_2_QraFYR_0">请问最后这个例子中的测试 Console Consume是怎样污染缓存页的？是因为它读取了比较老的数据，使得新数据被写入磁盘导致的吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-29 09:56:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/be/b9/f2481c2c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>诗泽</span>
  </div>
  <div class="_2_QraFYR_0">如果将kafka 部署到k8s 中，因为k8s 的节点都是禁用swap 的，所以文中提到的swappiness 设置也就失效了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-29 20:12:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>diyun</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，我们生产遇到一个问题，一个kafka broker集群升级到1.0.0后，consumer 客户端版本还是老的0.10.0.1，这样没法使用zero copy特性了导致有大量数据写入后kafka broker OOM了。请问我如果把consumer 客户端版本升级到1.0.0后，他连接的另一个老的broker集群（还是0.10.0.1版本）是否还能使用zero copy？kafka版本是向下兼容的吗，还是consumer和broker版本必须一致才能使用zero copy这样的特性。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 还是consumer和broker版本必须一致才能使用zero copy这样的特性 --- 是的，否则就要在broker端做消息转换，这样消息对象会在jvm堆上重建，丧失了zero copy特性。<br><br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-12 08:05:55</div>
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
  <div class="_2_QraFYR_0">你好，请教您一个问题，关于leader均衡，如果集群规模比较大，一次leader均衡会有上千个partition要进行leader切换，这会导致客户端很长时间不可用，目前针对这个场景有没有一些比较成熟的解决方案？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 关闭自动leader均衡，手动调整leader迁移是目前比较好的做法</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-07 00:13:23</div>
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
  <div class="_2_QraFYR_0">你好，一个broker建议最多存放多少个topic partition啊？这个个数和broker 的性能有啥关系吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 没有一定之规。不过据官网文章，单broker最多能承受2000个分区，这个和性能还是有很大关系的。毕竟分区数越多，物理IO性能就可能越差</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-01 14:19:38</div>
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
  <div class="_2_QraFYR_0">怎么查询linux是否开启了atime</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: mount -l，默认是开启的，如果发现noatime则是关闭的。Linux 2.6.30引入了relatime。有了relatime，atime的更新时机被缩小了，如果atime=mtime就不会被更新了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-29 10:09:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/df/79/2ef99993.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小鱼</span>
  </div>
  <div class="_2_QraFYR_0">关于zero copy那块，是不是仅针对consumer端而言呢？还是说，producer版本与broker不一致时，也会降低性能？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: &quot;producer版本与broker不一致时，也会降低性能&quot; --- 不会的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-29 15:49:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/c7/dc/9408c8c2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ban</span>
  </div>
  <div class="_2_QraFYR_0">老师，为什么测试 Console Consume读取比较老的数据，新的数据为什么会写入磁盘？这里不懂，Consume只是读取怎么会影响到写入</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 操作系统会被最近读取的page缓存起来，所以会“污染”页缓存</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-14 16:48:07</div>
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
  <div class="_2_QraFYR_0">没有kafka性能调优的经验，不过性能调优的思路是一致的。优化漏斗很形象，大部分调优主要在应用层，再深一点会到框架层，此时就需要对框架有很好的掌握啦！再深一点就到JVM了，这里主要是看内存空间分配是否合理，垃圾收集器是否正确选择。系统层调优，貌似没做过，这一层就必须对操作系统非常了解了。<br>万变不离其宗，提高性能的思路就那么几种：<br>1：使用更快的硬件，比如：内存<br>2：使用合适的数据结构<br>3：异步化<br>4：并行化<br>5：异步化和并行化，其实是在出现速度差的情况下，充分利用更快的组件的思路。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-24 09:15:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/62/59/a01a5ddd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ProgramGeek</span>
  </div>
  <div class="_2_QraFYR_0">老师，最近遇到一个问题，在kafka加入SASL  ACL中，生产的时候出现需要给事务ID赋权，那有个问题在有多生产者的情况下，同一主题下的事务ID能一样吗？如果ID不能一样，那我在加入kafka的时候每次都需要赋权怎么办</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果你并没有共享KafkaProducer实例，那么每个生产者最好设置成不同的transactional.id。2.0版本开始支持ACL前缀，可以用kafka-acls.sh --resource-pattern-type prefixed 一试</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-30 16:37:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/40/c3/e545ba80.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张振宇</span>
  </div>
  <div class="_2_QraFYR_0">老师，发现kafka的log文件比较大，执行rm删除log后df- h还未释放空间。<br>然后lsof -n &#47;data |grep deleted  发现查出来的进程是kafka的进程号，也不敢直接kill<br>想咨询下老师这种情况如何处理，有没更优雅的方式来优化。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: hmmm... 直接删除log文件的做法非常不优雅。。。<br>如果你要删除至少也要关闭Kafka集群之后再这么做。<br>最优雅的方式还是通过删除topic的命令来执行</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-14 18:57:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/d5/79/3d711fed.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>SAM(陈嘉奇)</span>
  </div>
  <div class="_2_QraFYR_0">老师你好我查到的swappiness的解释是这样的:<br>This control is used to define how aggressive (sic) the kernel will swap memory pages. Higher values will increase aggressiveness, lower values decrease the amount of swap. A value of 0 instructs the kernel not to initiate swap until the amount of free and file-backed pages is less than the high water mark in a zone. <br>按这个意思swap = 0的意思应该不是不交换。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯嗯，我看到那篇文章了。似乎swappiness作用的真实性有待确认。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-05 21:58:22</div>
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
  <div class="_2_QraFYR_0">胡哥：<br>消费者避免重复消费消息，比如kafka中的一条消息包含10个子消息，是针对kafka中的一条消息做幂等呢，还是针对kafka的每个子消息做幂等呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 针对外层消息。你下面的子消息自行维护</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-15 17:57:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/j24oyxHcpB5AMR9pMO6fITqnOFVOncnk2T1vdu1rYLfq1cN6Sj7xVrBVbCvHXUad2MpfyBcE4neBguxmjIxyiaQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>vcjmhg</span>
  </div>
  <div class="_2_QraFYR_0">这个优化思路感觉超级好！！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-20 20:33:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/89/aa/eb4c37db.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>John</span>
  </div>
  <div class="_2_QraFYR_0">请问老师，swappiness越小越不积极swap内存，不是更容易oom吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: oom也没事啊，关键是要弄明白原因。其实我们并不是怕oom</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-08 11:38:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/6a/fe/446c0282.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Minze</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，我最近发现有时候consumergroup的current offset经常会几秒钟不动，而这是log end offset是不断增加的。我该从哪些方向着手检查呢？已经检查过了consumer端，消费的方法并没有阻塞的情况，执行时间都是毫秒级的，手动commit offset</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 哪个版本的Kafka呢？如果是java consumer，查一下有没有提交位移失败的log？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-28 13:21:22</div>
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
  <div class="_2_QraFYR_0">优化tps和优化延迟 参数都是相反的啊，这个要根据需求来优化么。<br>那么tps如果增加了 延迟不是高了么，总的时间消耗 跟优化延迟哪个更可取呢? </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不一定在任何场景下都是相反的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-21 13:57:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/91/b4/d5d9e4fb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>爱学习的小学生</span>
  </div>
  <div class="_2_QraFYR_0">processing fetch with max size 30720000 from consumer on partition  PartitionData(fetchOffset=12297049, logStartOffset=-1, maxBytes=30720000, currentLeaderEpoch=Optional.empty, lastFetchedEpoch=Optional.empty) (kafka.server.ReplicaManager)<br>org.apache.kafka.common.errors.CorruptRecordException: Found record size -1554710528 smaller than minimum record overhead (14) in file 老师我的kafka有这个报错，并且这个分区的isr只有一个，这是什么问题呢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-15 23:58:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/35/9c/4ed5ae0a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小铁จุ๊บMia</span>
  </div>
  <div class="_2_QraFYR_0">既然讲到调优，怎么对kafka进行性能压测呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-13 16:19:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/2d/4c/983ce1b9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>海水</span>
  </div>
  <div class="_2_QraFYR_0">单一broker cpu 和带宽过高 感觉像是__consumer_offsets 相关的原因 老师 需要怎么排查问题呢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-21 16:08:17</div>
  </div>
</div>
</div>
</li>
</ul>