<audio title="06 _ Kafka线上集群部署方案怎么做？" src="https://static001.geekbang.org/resource/audio/1e/30/1e9f93f27a7f4dad4dd34fdcb1ae7830.mp3" controls="controls"></audio> 
<p>专栏前面几期内容，我分别从Kafka的定位、版本的变迁以及功能的演进等几个方面循序渐进地梳理了Apache Kafka的发展脉络。通过这些内容，我希望你能清晰地了解Kafka是用来做什么的，以及在实际生产环境中该如何选择Kafka版本，更快地帮助你入门Kafka。</p><p>现在我们就来看看在生产环境中的Kafka集群方案该怎么做。既然是集群，那必然就要有多个Kafka节点机器，因为只有单台机器构成的Kafka伪集群只能用于日常测试之用，根本无法满足实际的线上生产需求。而真正的线上环境需要仔细地考量各种因素，结合自身的业务需求而制定。下面我就分别从操作系统、磁盘、磁盘容量和带宽等方面来讨论一下。</p><h2>操作系统</h2><p>首先我们先看看要把Kafka安装到什么操作系统上。说起操作系统，可能你会问Kafka不是JVM系的大数据框架吗？Java又是跨平台的语言，把Kafka安装到不同的操作系统上会有什么区别吗？其实区别相当大！</p><p>的确，如你所知，Kafka由Scala语言和Java语言编写而成，编译之后的源代码就是普通的“.class”文件。本来部署到哪个操作系统应该都是一样的，但是不同操作系统的差异还是给Kafka集群带来了相当大的影响。目前常见的操作系统有3种：Linux、Windows和macOS。应该说部署在Linux上的生产环境是最多的，也有一些Kafka集群部署在Windows服务器上。Mac虽然也有macOS Server，但是我怀疑是否有人（特别是国内用户）真的把生产环境部署在Mac服务器上。</p><!-- [[[read_end]]] --><p>如果考虑操作系统与Kafka的适配性，Linux系统显然要比其他两个特别是Windows系统更加适合部署Kafka。虽然这个结论可能你不感到意外，但其中具体的原因你也一定要了解。主要是在下面这三个方面上，Linux的表现更胜一筹。</p><ul>
<li>I/O模型的使用</li>
<li>数据网络传输效率</li>
<li>社区支持度</li>
</ul><p>我分别来解释一下，首先来看I/O模型。什么是I/O模型呢？你可以近似地认为I/O模型就是操作系统执行I/O指令的方法。</p><p>主流的I/O模型通常有5种类型：阻塞式I/O、非阻塞式I/O、I/O多路复用、信号驱动I/O和异步I/O。每种I/O模型都有各自典型的使用场景，比如Java中Socket对象的阻塞模式和非阻塞模式就对应于前两种模型；而Linux中的系统调用select函数就属于I/O多路复用模型；大名鼎鼎的epoll系统调用则介于第三种和第四种模型之间；至于第五种模型，其实很少有Linux系统支持，反而是Windows系统提供了一个叫IOCP线程模型属于这一种。</p><p>你不必详细了解每一种模型的实现细节，通常情况下我们认为后一种模型会比前一种模型要高级，比如epoll就比select要好，了解到这一程度应该足以应付我们下面的内容了。</p><p>说了这么多，I/O模型与Kafka的关系又是什么呢？实际上Kafka客户端底层使用了Java的selector，selector在Linux上的实现机制是epoll，而在Windows平台上的实现机制是select。<strong>因此在这一点上将Kafka部署在Linux上是有优势的，因为能够获得更高效的I/O性能。</strong></p><p>其次是网络传输效率的差别。你知道的，Kafka生产和消费的消息都是通过网络传输的，而消息保存在哪里呢？肯定是磁盘。故Kafka需要在磁盘和网络间进行大量数据传输。如果你熟悉Linux，你肯定听过零拷贝（Zero Copy）技术，就是当数据在磁盘和网络进行传输时避免昂贵的内核态数据拷贝从而实现快速的数据传输。Linux平台实现了这样的零拷贝机制，但有些令人遗憾的是在Windows平台上必须要等到Java 8的60更新版本才能“享受”到这个福利。<strong>一句话总结一下，在Linux部署Kafka能够享受到零拷贝技术所带来的快速数据传输特性。</strong></p><p>最后是社区的支持度。这一点虽然不是什么明显的差别，但如果不了解的话可能比前两个因素对你的影响更大。简单来说就是，社区目前对Windows平台上发现的Kafka Bug不做任何承诺。虽然口头上依然保证尽力去解决，但根据我的经验，Windows上的Bug一般是不会修复的。<strong>因此，Windows平台上部署Kafka只适合于个人测试或用于功能验证，千万不要应用于生产环境。</strong></p><h2>磁盘</h2><p>如果问哪种资源对Kafka性能最重要，磁盘无疑是要排名靠前的。在对Kafka集群进行磁盘规划时经常面对的问题是，我应该选择普通的机械磁盘还是固态硬盘？前者成本低且容量大，但易损坏；后者性能优势大，不过单价高。我给出的建议是使用普通机械硬盘即可。</p><p>Kafka大量使用磁盘不假，可它使用的方式多是顺序读写操作，一定程度上规避了机械磁盘最大的劣势，即随机读写操作慢。从这一点上来说，使用SSD似乎并没有太大的性能优势，毕竟从性价比上来说，机械磁盘物美价廉，而它因易损坏而造成的可靠性差等缺陷，又由Kafka在软件层面提供机制来保证，故使用普通机械磁盘是很划算的。</p><p>关于磁盘选择另一个经常讨论的话题就是到底是否应该使用磁盘阵列（RAID）。使用RAID的两个主要优势在于：</p><ul>
<li>提供冗余的磁盘存储空间</li>
<li>提供负载均衡</li>
</ul><p>以上两个优势对于任何一个分布式系统都很有吸引力。不过就Kafka而言，一方面Kafka自己实现了冗余机制来提供高可靠性；另一方面通过分区的概念，Kafka也能在软件层面自行实现负载均衡。如此说来RAID的优势就没有那么明显了。当然，我并不是说RAID不好，实际上依然有很多大厂确实是把Kafka底层的存储交由RAID的，只是目前Kafka在存储这方面提供了越来越便捷的高可靠性方案，因此在线上环境使用RAID似乎变得不是那么重要了。综合以上的考量，我给出的建议是：</p><ul>
<li>追求性价比的公司可以不搭建RAID，使用普通磁盘组成存储空间即可。</li>
<li>使用机械磁盘完全能够胜任Kafka线上环境。</li>
</ul><h2>磁盘容量</h2><p>Kafka集群到底需要多大的存储空间？这是一个非常经典的规划问题。Kafka需要将消息保存在底层的磁盘上，这些消息默认会被保存一段时间然后自动被删除。虽然这段时间是可以配置的，但你应该如何结合自身业务场景和存储需求来规划Kafka集群的存储容量呢？</p><p>我举一个简单的例子来说明该如何思考这个问题。假设你所在公司有个业务每天需要向Kafka集群发送1亿条消息，每条消息保存两份以防止数据丢失，另外消息默认保存两周时间。现在假设消息的平均大小是1KB，那么你能说出你的Kafka集群需要为这个业务预留多少磁盘空间吗？</p><p>我们来计算一下：每天1亿条1KB大小的消息，保存两份且留存两周的时间，那么总的空间大小就等于1亿 * 1KB * 2 / 1000 / 1000 = 200GB。一般情况下Kafka集群除了消息数据还有其他类型的数据，比如索引数据等，故我们再为这些数据预留出10%的磁盘空间，因此总的存储容量就是220GB。既然要保存两周，那么整体容量即为220GB * 14，大约3TB左右。Kafka支持数据的压缩，假设压缩比是0.75，那么最后你需要规划的存储空间就是0.75 * 3 = 2.25TB。</p><p>总之在规划磁盘容量时你需要考虑下面这几个元素：</p><ul>
<li>新增消息数</li>
<li>消息留存时间</li>
<li>平均消息大小</li>
<li>备份数</li>
<li>是否启用压缩</li>
</ul><h2>带宽</h2><p>对于Kafka这种通过网络大量进行数据传输的框架而言，带宽特别容易成为瓶颈。事实上，在我接触的真实案例当中，带宽资源不足导致Kafka出现性能问题的比例至少占60%以上。如果你的环境中还涉及跨机房传输，那么情况可能就更糟了。</p><p>如果你不是超级土豪的话，我会认为你和我平时使用的都是普通的以太网络，带宽也主要有两种：1Gbps的千兆网络和10Gbps的万兆网络，特别是千兆网络应该是一般公司网络的标准配置了。下面我就以千兆网络举一个实际的例子，来说明一下如何进行带宽资源的规划。</p><p>与其说是带宽资源的规划，其实真正要规划的是所需的Kafka服务器的数量。假设你公司的机房环境是千兆网络，即1Gbps，现在你有个业务，其业务目标或SLA是在1小时内处理1TB的业务数据。那么问题来了，你到底需要多少台Kafka服务器来完成这个业务呢？</p><p>让我们来计算一下，由于带宽是1Gbps，即每秒处理1Gb的数据，假设每台Kafka服务器都是安装在专属的机器上，也就是说每台Kafka机器上没有混部其他服务，毕竟真实环境中不建议这么做。通常情况下你只能假设Kafka会用到70%的带宽资源，因为总要为其他应用或进程留一些资源。</p><p>根据实际使用经验，超过70%的阈值就有网络丢包的可能性了，故70%的设定是一个比较合理的值，也就是说单台Kafka服务器最多也就能使用大约700Mb的带宽资源。</p><p>稍等，这只是它能使用的最大带宽资源，你不能让Kafka服务器常规性使用这么多资源，故通常要再额外预留出2/3的资源，即单台服务器使用带宽700Mb / 3  ≈  240Mbps。需要提示的是，这里的2/3其实是相当保守的，你可以结合你自己机器的使用情况酌情减少此值。</p><p>好了，有了240Mbps，我们就可以计算1小时内处理1TB数据所需的服务器数量了。根据这个目标，我们每秒需要处理2336Mb的数据，除以240，约等于10台服务器。如果消息还需要额外复制两份，那么总的服务器台数还要乘以3，即30台。</p><p>怎么样，还是很简单的吧。用这种方法评估线上环境的服务器台数是比较合理的，而且这个方法能够随着你业务需求的变化而动态调整。</p><h2>小结</h2><p>所谓“兵马未动，粮草先行”。与其盲目上马一套Kafka环境然后事后费力调整，不如在一开始就思考好实际场景下业务所需的集群环境。在考量部署方案时需要通盘考虑，不能仅从单个维度上进行评估。相信今天我们聊完之后，你对如何规划Kafka生产环境一定有了一个清晰的认识。现在我来总结一下今天的重点：</p><p><img src="https://static001.geekbang.org/resource/image/ba/04/bacf5700e4b145328f4d977575f28904.jpg?wh=3769*1533" alt=""></p><h2>开放讨论</h2><p>对于今天我所讲的这套评估方法，你有什么问题吗？你还能想出什么改进的方法吗？</p><p>欢迎你写下自己的思考或疑问，我们一起讨论 。如果你觉得有所收获，也欢迎把文章分享给你的朋友。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/55/55/19ec7b0e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>mickle</span>
  </div>
  <div class="_2_QraFYR_0">1000*1000&#47;(60*60)=277,这个2336MB是怎么换算来的，还有什么要考虑的吗?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 277MB，乘以8，大致等于2300+Mb（小b）。带宽资源一般用Mbps而不是MBps衡量</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-15 10:03:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c9/90/c7fbf39e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>A_NATE_👻</span>
  </div>
  <div class="_2_QraFYR_0">我们曾经也认为用普通硬盘就行，换成普通硬盘导致生产者堵塞写入负载偏高，换成SSD就没事了，我们每天消息数大概50亿。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯嗯，专栏里面只是给出一个评估的方法。具体还要结合自己的实际情况来调整。通常我们认为SSD的顺序写TPS大约是HDD的4倍。除了纵向扩展使用SSD之外，也可以尝试一下横向扩展，增加更多的broker或HDD分散负载：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-15 11:42:40</div>
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
  <div class="_2_QraFYR_0">老师，你好，你讲的这几个纬度很好，之前我们搭建一套kafka集群就不知道怎么去衡量，我再问一个相关问题，我个人觉得kafka会出现丢数据情况，比如某个分区的leader挂了，在切换选举到另外副本为leader时，这个副本还没同步之前的leader数据，这样数据就丢了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯嗯，对于producer而言，如果在乎数据持久性，那么应该设置acks=all，这样当出现你说的这个情况时，producer会被显式通知消息发送失败，从而可以重试。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-17 08:34:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/e9/0b/1171ac71.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>WL</span>
  </div>
  <div class="_2_QraFYR_0">有三个问题请教一下老师:<br>1. 上文提到对于千兆网卡kafka服务器最多使用700M的带宽资源, 这700M的资源是单机使用的还是集群共用的, 为什么不能作为常规使用呢?<br>2. 文章举例是1小时1T的数据处理目标, 那一秒中是不是1024&#47;3600 = 0.284G = 285M, 请问下文章中的2336M是咋算出来的.<br>3. 文章中的例子kafka单机要达到240M的读写能力, CPU应该配几核的?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. 这个700Mb只是经验值罢了。另外预留buffer的意思是即使你最好不要让broker常规占用700Mb的资源。一旦碰到峰值流量，很容易将带宽打满。故做了一些资源预留<br>2. 285M是大B，即字节啊，乘以8之后就是2336Mb。带宽资源一般用Mbps而非MBps衡量<br>3. 我没有谈及CPU，是因为通常情况下Kafka不太占用CPU，因此没有这方面的最佳实践出来。但有些情况下Kafka broker是很耗CPU的：1. server和client使用了不同的压缩算法；2. server和client版本不一致造成消息格式转换；3. broker端解压缩校验<br><br>其中前两个都能规避，第三个目前无法规避。不过相比带宽资源，CPU通常都不是瓶颈</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-15 11:15:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/90/d0/48037ba6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>李跃爱学习</span>
  </div>
  <div class="_2_QraFYR_0">老师希望解答一下，之前也说明了Kafka 机器上没有混布其他服务，为什么常规需要预留2&#47;3，只能跑240Mbps，</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 为follower拉取留一些带宽</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-16 11:07:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_Sue</span>
  </div>
  <div class="_2_QraFYR_0">胡老师，您好，我想请问下，我们公司的环境是基于Docker这种微服务架构，那么kafka部署在Docker容器中部署方案是否会有一些不同呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 目前社区对Docker方案支持的并不是太好，主要都是一些第三方公司还有Confluent公司在提供解决方案。在Docker上部署我个人觉得没有太大的不同，只是注意带宽资源吧，因为常见的做法都是买一台性能超强的服务器然后在上面启动多个Docker容器，虽然CPU、RAM、磁盘都能承受，但单机还是受限于带宽的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-17 14:57:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>墙角儿的花</span>
  </div>
  <div class="_2_QraFYR_0">弱弱的问一句老师，“根据这个目标，我们每秒需要处理 2336Mb 的数据，除以 240，约等于 10 台服务器”，机房入口带宽1Gbps,怎么能做到1秒处理2336Mb的数据的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这里是指单机带宽，机房总带宽不可能这么小的。。。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-15 09:00:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c6/73/abb7bfe3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>疯琴</span>
  </div>
  <div class="_2_QraFYR_0">老师，partitons的数量和硬盘的数量有匹配关系么？一块盘一个partiton比一块盘多个partiton要快么？是线性的关系么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 没有具体的关系。<br><br>“一块盘一个partiton比一块盘多个partiton要快么？” 没有实验数据支撑，单纯从分析角度来看我是认同的。当某块磁盘上有太多的分区时引入了很多随机IO拖慢了性能。事实上，阿里的RocketMQ一直宣称当单磁盘超过256分区时Kafka性能不如RocketMQ，原因也在于此。<br>数据来源：http:&#47;&#47;jm.taobao.org&#47;2016&#47;04&#47;07&#47;kafka-vs-rocketmq-topic-amout&#47;</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-17 08:28:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/85/3b/cfdc8bf2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Royal</span>
  </div>
  <div class="_2_QraFYR_0">您好，我想请教下kafka metric相关的知识，比如kafka produce速率获取等</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: kafka producer速率可以监控这个JMX指标：<br><br>kafka.producer:type=[consumer|producer|connect]-node-metrics,client-id=([-.\w]+),node-id=([0-9]+)</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-15 00:31:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/88/26/b8c53cee.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>南辕北辙</span>
  </div>
  <div class="_2_QraFYR_0">这个假设是：follower与leader处于不同的broker而实际环境中不推荐单机多broker的架构  摘自老师回复其他同学。<br>老师这个的意思是不是生产上的架构通常一台服务器上只会有leader或者follow的分区，而不会二者存在一台服务器上，所以根据带宽计算服务器数量时，根据备份数为2，所以就直接✖️3了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Leader副本和Follower副本必然在不同的Broker上，而生产环境一般也不推荐将多台Broker混布到同一台服务器上。当然服务器性能强劲的话也未尝不可：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-17 16:39:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/ae/ef/cbb8d881.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>黄智荣</span>
  </div>
  <div class="_2_QraFYR_0">老师您好,我们现在用的网络是10Gb,的万兆网.这样主要性能瓶性可会在硬盘IO上,我们配置了24坏的sas 机械盘。<br>刚才老师主要从容量跟网络性能,评估集群的规模。<br>我想问一下,怎么从硬盘的IO性能,去评估集群的规模?<br>还有做6个盘组成一个raid5,比直接用裸盘,性能会有多大的损失?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: RAID-5具体有多少性能损失不好说，但肯定会有的，最好还是以测试结果为准。另外咱们是否可以更多地利用Kafka在软件层面提供的高可靠性来保证数据完整性呢？不用单纯依赖于硬件。<br><br>至于评估方法，磁盘要相对复杂一些。毕竟当topic数量很多的时候磁盘不一定都是顺序写。不过你姑且可以做这样的测试：做一个单partition的topic，测试一下该topic对应的TPS，然后用你磁盘的TPS去核算单块磁盘上大概能放多少个partition。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-18 10:28:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/07/2e/f6a2dc27.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Eagles</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，Leader Follower 之间的数据冗余复制会不会占带宽？如果占带宽且有两个以上的follows，岂不是把预留的2&#47;3的带宽全部用掉了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 会占带宽的，这也是预留的部分之一，就是给非业务活动留的带宽资源，包括操作系统其他进程消耗的带宽啊。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-17 01:29:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/02/9b/b1a3c60d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>CDz</span>
  </div>
  <div class="_2_QraFYR_0"><br>系统选择：Linux为主<br>原因：<br>* 社区支持<br>* IO模型<br>    * kafka底层是使用Java的NIO，在Linux在Java默认是用的epoll实现相比select更高效<br>* 数据传输<br>    * Linux环境下JavaNIO使用零拷贝技术（zero copy）<br>    * Win环境下只有在Java8 后面版本才支持<br><br>磁盘：<br>使用机械硬盘还是SSD？<br><br>因为kafka更多的是顺序读写，很大程度上避免了机械硬盘的随机读写慢问题。所以在数据量不多的情况下，使用机械硬盘物美价廉<br><br>机械硬盘是否需要RAID?<br><br>综合考虑，数据量并不是特别大的情况下，并不需要，因为kafka本身有集群备份的功能。<br><br>磁盘容量需要多大？<br><br>一般默认消息保存两周，根据自己的实际每天消息数量，进行换算。换算时留出10%的磁盘空间即可。<br><br>考虑点：<br>* 每日消息量、消息大小<br>* 保存时间<br>* 备份数量<br>* 是否开启压缩（消耗CPU）<br><br>宽带：<br><br>网络宽带单位与磁盘存储单位是什么关系？<br><br>存储单位一般使用的字节(Byte)为基本单位，数据传输大多使用的是“位”(bit)<br><br>1Byte = 8bit<br><br>所以换算下来，平时说的1Gbps的网络实际速度为：<br><br>1Gbps = 1024Mb(位)&#47;8=128MB(字节)&#47;s;<br><br>Over;<br><br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-16 09:04:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/ee/27/5b15fa63.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Nero</span>
  </div>
  <div class="_2_QraFYR_0">好了，有了 240Mbps，我们就可以计算 1 小时内处理 1TB 数据所需的服务器数量了。根据这个目标，我们每秒需要处理 2336Mb 的数据，         老师这一段中2334Mb是怎么算出来的，没看懂，能否解释一下，谢谢。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1024*1024&#47;3600*8</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-15 09:26:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c6/73/abb7bfe3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>疯琴</span>
  </div>
  <div class="_2_QraFYR_0">老师，如果单机起多个broker是否有可能造成同一个partition的多个副本在一台机器上，损失了容灾能力？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，有这个可能。现在的分布式集群都倾向于使用普通性能的机器搭建，因此单台单Broker性价比很高的。<br><br>当然我也知道很多公司采购了超强的服务器，然后在上面跑多个实例。这么做的原因是因为机器价格其实很便宜，贵的是IDC机架位（至少在北京是这样），因此两个方案都有自己合理的地方吧。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-17 18:32:05</div>
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
  <div class="_2_QraFYR_0">老师尽然把我模糊不清的GB和Gb的区别给复习了一遍😂</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-15 22:22:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/9c/7c/c92f8a53.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>坤蛰</span>
  </div>
  <div class="_2_QraFYR_0">您好！我在使用Kafka中，出现消息阻塞不消费的问题，换了消费组之后过段时间又不消费了，不知什么原因，望老师解惑。谢谢！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 两个可能的原因，①就是没有新消息可供消费了，②某天消息格式有问题导致解析不了了，不过这种情况很罕见，一般是因为网络传输出问题导致</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-15 09:07:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/0f/70/c8680841.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Joe Black</span>
  </div>
  <div class="_2_QraFYR_0">发现问的最多的竟然是Byte和Bit的问题...我们用的40Gb的网络，服务器内存290多G，24块万转SAS盘，按老师说的，几十台这样的服务器可以处理很大规模的数据量了吧，都不用怎么规划…</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 很多公司的机器没有这么强劲的性能......</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-01 17:44:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/mQddXC7nRiaKHTwdficicTB3bH0q5ic5UoSab51Omic7eyLBz0SNcvbLpQnNib7zP1yJFm7xxx4ia81iahfibRVnbTwHmhw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>浮石沉木</span>
  </div>
  <div class="_2_QraFYR_0">带宽、磁盘单位问题看这个：<br>https:&#47;&#47;www.cnblogs.com&#47;linckle&#47;archive&#47;2007&#47;12&#47;03&#47;981396.html</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-13 23:08:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://wx.qlogo.cn/mmopen/vi_32/mvxl1xzYcAfJERgGHBCswnbeibZJmS1IlP1z9P7KsSh2EPuM78DqPKPicAmHXPYUib6RPcjGcf5vrXkQXAiaLorB2w/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>漩涡鸣人</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，我遇到一个场景 kafka数据 网络丢包会不会 导致json 字段丢失。比如 {姓名，年龄，生日，{其他信息}  }  嵌套的其他信息数据丢失变成null </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 说实话感觉不像是Kafka的问题，有可能是序列化&#47;反序列化的问题。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-18 00:30:45</div>
  </div>
</div>
</div>
</li>
</ul>