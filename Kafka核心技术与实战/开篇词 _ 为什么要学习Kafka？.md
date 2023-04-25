<audio title="开篇词 _ 为什么要学习Kafka？" src="https://static001.geekbang.org/resource/audio/f8/14/f8827506375a9a34508485121f775314.mp3" controls="controls"></audio> 
<p>你好，我是胡夕，Apache Kafka Committer，老虎证券用户增长团队负责人，也是《Apache Kafka实战》这本书的作者。</p><p>在过去5年中，我经历了Kafka从最初的0.8版本逐步演进到现在的2.3版本的完整过程，踩了很多坑也交了很多学费，慢慢地我梳理出了一个相对系统、完整的Kafka应用实战指南，最终以“Kafka核心技术与实战”专栏的形式呈现给你，希望分享我对Apache Kafka的理解和实战方面的经验，帮你透彻理解Kafka、更好地应用Kafka。</p><p>你可能会有这样的疑问，<strong>我为什么要学习Kafka呢</strong>？要回答这个问题，我们不妨从更大的视角来审视它，先聊聊我对这几年互联网技术发展的理解吧。</p><p>互联网蓬勃发展的这些年涌现出了很多令人眼花缭乱的新技术。以我个人的浅见，截止到2019年，当下互联网行业最火的技术当属ABC了，即所谓的AI人工智能、BigData大数据和Cloud云计算云平台。我个人对区块链技术发展前景存疑，毕竟目前没有看到特别好的落地应用场景，也许在未来几年它会更令人刮目相看吧。</p><p>在这ABC当中，坦率说A和C是有点曲高和寡的，不是所有玩家都能入场。反观B要显得平民得多，几乎所有公司都能参与进来。我曾经到过一个理发厅，那里的人都宣称他们采用了大数据系统帮助客户设计造型，足见BigData是很“下里巴人”的。</p><!-- [[[read_end]]] --><p>作为工程师或架构师，你在实际工作过程中一定参与到了很多大数据业务系统的构建。由于这些系统都是为公司业务服务的，所以通常来说它们仅仅是执行一些常规的业务逻辑，因此它们不能算是计算密集型应用，相反更应该是数据密集型的。</p><p>对于数据密集型应用来说，如何应对数据量激增、数据复杂度增加以及数据变化速率变快，是彰显大数据工程师、架构师功力的最有效表征。我们欣喜地发现Kafka在帮助你应对这些问题方面能起到非常好的效果。就拿数据量激增来说，Kafka能够有效隔离上下游业务，将上游突增的流量缓存起来，以平滑的方式传导到下游子系统中，避免了流量的不规则冲击。由此可见，如果你是一名大数据从业人员，熟练掌握Kafka是非常必要的一项技能。</p><p>刚刚所举的例子仅仅是Kafka助力业务的一个场景罢了。事实上，Kafka有着非常广阔的应用场景。不谦虚地说，目前Apache Kafka被认为是整个消息引擎领域的执牛耳者，仅凭这一点就值得我们好好学习一下它。另外，从学习技术的角度而言，Kafka也是很有亮点的。我们仅需要学习一套框架就能在实际业务系统中实现消息引擎应用、应用程序集成、分布式存储构建，甚至是流处理应用的开发与部署，听起来还是很超值的吧。</p><p>不仅如此，再给你看一个数据。援引美国2019年Dice技术薪资报告中的数据，在10大薪资最高的技术技能中，掌握Kafka以平均每年12.8万美元排名第二！排名第一位的是13.2万美元/年的Go语言。好吧，希望你看到这个之后不会立即关闭我的专栏然后转头直奔隔壁的Go语言专栏。虽然这是美国人才市场的数据，但是我们有理由相信在国内Kafka的行情也是水涨船高。2019年两会上再一次提到了要深化<strong>大数据</strong>、人工智能等研发应用，而Kafka无论是作为消息引擎还是实时流处理平台，都能在大数据工程领域发挥重要的作用。</p><p>总之Kafka是个利器，值得一试！既然知道了为什么要学Kafka，那我们就要行动起来，把它学透，而学透Kafka有什么路径吗？</p><p>如果你是一名软件开发工程师的话，掌握Kafka的第一步就是要根据你掌握的编程语言去寻找对应的Kafka客户端。当前Kafka最重要的两大客户端是Java客户端和libkafka客户端，它们更新和维护的速度很快，非常适合你持续花时间投入。</p><p>一旦确定了要使用的客户端，马上去官网上学习一下代码示例，如果能够正确编译和运行这些样例，你就能轻松地驾驭客户端了。</p><p>下一步你可以尝试修改样例代码尝试去理解并使用其他的API，之后观测你修改的结果。如果这些都没有难倒你，你可以自己编写一个小型项目来验证下学习成果，然后就是改善和提升客户端的可靠性和性能了。到了这一步，你可以熟读一遍Kafka官网文档，确保你理解了那些可能影响可靠性和性能的参数。</p><p>最后是学习Kafka的高级功能，比如流处理应用开发。流处理API不仅能够生产和消费消息，还能执行高级的流式处理操作，比如时间窗口聚合、流处理连接等。</p><p>如果你是系统管理员或运维工程师，那么相应的学习目标应该是学习搭建及管理Kafka线上环境。如何根据实际业务需求评估、搭建生产线上环境将是你主要的学习目标。另外对生产环境的监控也是重中之重的工作，Kafka提供了超多的JMX监控指标，你可以选择任意你熟知的框架进行监控。有了监控数据，作为系统运维管理员的你，势必要观测真实业务负载下的Kafka集群表现。之后如何利用已有的监控指标来找出系统瓶颈，然后提升整个系统的吞吐量，这也是最能体现你工作价值的地方。</p><p>在明确了自己要学什么以及怎么学之后，你现在会不会有一种感慨：原来我要学习这么多东西呀！不用担心，刚刚我提到的所有内容都会在专栏中被覆盖到。</p><p>下面是我特意为专栏画的一张思维导图，可以帮你迅速了解这个专栏的知识结构体系是什么样的。专栏大致从六个方面展开，包括Kafka入门、Kafka的基本使用、客户端详解、Kafka原理介绍、Kafka运维与监控以及高级Kafka应用。</p><p><img src="https://static001.geekbang.org/resource/image/8b/95/8b28137150c70d66200f649e26ff2395.jpg?wh=1804*1132" alt=""></p><ul>
<li>专栏的第一部分我会介绍消息引擎这类系统大致的原理和用途，以及作为优秀消息引擎代表的Kafka在这方面的表现。</li>
<li>第二部分则重点探讨Kafka如何用于生产环境，特别是线上环境方案的制定。</li>
<li>在第三部分中我会陪你一起学习Kafka客户端的方方面面，既有生产者的实操讲解也有消费者的原理剖析，你一定不要错过。</li>
<li>第四部分会着重介绍Kafka最核心的设计原理，包括Controller的设计机制、请求处理全流程解析等。</li>
<li>第五部分则涵盖Kafka运维与监控的内容，想获得高效运维Kafka集群以及有效监控Kafka的实战经验？我必当倾囊相助！</li>
<li>最后一个部分我会简单介绍一下Kafka流处理组件Kafka Streams的实战应用，希望能让你认识一个不太一样的Kafka。</li>
</ul><p>这里不得不提的是，有熟悉我的读者可能知道我出版过的图书《Apache Kafka实战》。你可能有这样的疑问：既然有书了，那么这个专栏与书的区别又是什么呢？《Apache Kafka实战》这本书是基于Kafka 1.0版本撰写的，但目前Kafka已经演进到2.3版本了，我必须要承认书中的部分内容已经过时甚至是不准确了，而专栏的写作是基于Kafka的最新版。并且专栏作为一次全新的交付，我希望能用更轻松更容易理解的语言和形式，帮你获取到最新的Kafka实战经验。</p><p>我希望通过学习这个专栏，你不仅能够将Kafka熟练运用到实际工作当中去，而且还能培养出对于Kafka或是其他技术框架的浓厚学习兴趣。</p><p>最后我希望用一句话收尾与你共勉：Stay focused and work hard！</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/ee/98/299059e8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>阿武</span>
  </div>
  <div class="_2_QraFYR_0">老师 发量 保持的还不错！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-03 23:51:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/02/17/2930d112.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>大番茄</span>
  </div>
  <div class="_2_QraFYR_0">1.希望老师 讲下rabbitmq和kafka区别，传统企业如何选择?<br><br>2. 课程更新周期是怎么样的？<br><br>3. 学习这个课，老师能推荐些辅助资料么？<br><br>🙏感谢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-04 10:11:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/a0/0f/e3587e06.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>默无闻</span>
  </div>
  <div class="_2_QraFYR_0">老师，我恰好会go语言，再加上kafka是不是碉堡了！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 莫老板别闹：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-12 23:48:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/04/60/64d166b6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Fan</span>
  </div>
  <div class="_2_QraFYR_0">Stay focused and work hard！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 特别喜欢这句话：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-03 18:12:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/92/58/b4f6365d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小北</span>
  </div>
  <div class="_2_QraFYR_0">希望后续会有php以及go客户端的讲解</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-04 22:21:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/1c/cd/8d552516.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Gojustforfun</span>
  </div>
  <div class="_2_QraFYR_0">期待，同时期望老师在专栏中穿插一些高频的面试题 ——作为消息中间件使用时，消息的可靠传输、顺序，重复消费等问题在kafka中是如何解决的。这不仅仅对面试有帮助，也对进入公司后快速上手熟悉系统、排查线上问题等有帮助，谢谢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯嗯，专栏中有些文章的标题就是常见的面试题。比如消息无丢失配置，再比如消费者组rebalance过程等，可以关注下~~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-04 14:40:01</div>
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
  <div class="_2_QraFYR_0">Stay focused and work hard！<br>又要开始一段新的旅程，希望看到一路美轮美奂丰富多彩的风景。<br>kafka 久仰大名未曾谋面，这次希望对她多一些了解。<br>目前对消息队列中间的理解<br>1：作用——系统解藕，流量削峰填谷<br>2：本质——生产者消费者模式的经典实现<br>3：目前听说的有rocketMQ&#47;rabbitMQ，相信大厂都有自研的<br>4：感觉使用比较简单<br>我希望通过本专栏的学习，能了解到，如下内容<br>1：kafka的使用以及监控和问题分析与排查？<br>2：听说kafka比较牛逼，它到底优秀在哪里？<br>3：kafka为什么这么优秀？<br>4：kafka和其他消息中间件相比有什么特点？<br>5：kafka的最佳实践？<br>6：老师是怎么研究kafka的？怎么像老师一样优秀？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-10 23:02:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/42/fc/c89243d8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>侯代烨</span>
  </div>
  <div class="_2_QraFYR_0">我们的kafka集群，每个节点只能存储35G的数据量，超过这个量之后，kafka进程就会挂掉，启动会报内存溢出的错误，困扰很久了，不知道学习了这个课程，能不能解决这个问题。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 后面会有性能调优的部分。分析性能问题还是要结合具体 的错误来看，比如内存溢出要明确是哪部分内存溢出，然后才有可能有针对性的调整</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-05 20:27:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/98/7f/9ce24253.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>paradox</span>
  </div>
  <div class="_2_QraFYR_0">相比人工智能和区块链，大数据和云计算成熟很多。云计算的成熟，又促成了大数据技术的大规模普及。不过真正有“大”数据的公司并不多。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-03 18:50:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/67/49/864dba17.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>东风第一枝</span>
  </div>
  <div class="_2_QraFYR_0">这个专栏等了一段时间了，前段时间正好在学习Kafka，后期会用到项目实践中，期待能有一个深入的理解。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢支持与鼓励：）期待你的反馈我们一起学习~~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-03 17:20:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/21/70/06fcbdab.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>陈盼</span>
  </div>
  <div class="_2_QraFYR_0">您好，请教下要清除kafka所有的缓存信息，要删哪些目录？默认情况下。我现在重装时删除了log.dirs指定的目录再重新发布时会自动创建以前的topic，而且没有__consumer_offset</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. 关闭集群和ZooKeeper<br>2. 删除log.dirs配置的目录下的内容<br>3. 删除ZooKeeper路径下的内容<br>4. 重启ZooKeeper和集群</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-19 19:06:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/9d/84/171b2221.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jeffery</span>
  </div>
  <div class="_2_QraFYR_0">书+专栏更完美、期待蜕变</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢支持与鼓励：） 我们一起学习</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-03 18:51:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/12/08/abb7bfe3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>roger</span>
  </div>
  <div class="_2_QraFYR_0">有新出比kafka更灵活的消息队列吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不确定百分之百理解了“灵活”的含义，不过可以关注下Apache Pulsar</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-05 07:41:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/PiajxSqBRaELTibicWYgMaJW7pW1V7bgBdHWibJPM49iaQB8EpGzWx8ZJPN2H2vfZSw0yT8WiaLSIdia2rh3REibibLDKkQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_e5967d</span>
  </div>
  <div class="_2_QraFYR_0">赞一个，最近开始学习kafka运维，我们使用的还是用的0.8.1版本😿</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-03 21:39:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/75/9b/611e74ab.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>技术修行者</span>
  </div>
  <div class="_2_QraFYR_0">请教一下老师，在Kafka中如果要实现多租户，有什么需要考虑的，以及基本设计思路是什么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 目前开源版的Kafka要实现多租户只能自己实现，有几个基本的事情要做：<br>1. 构建完备的用户认证和权限体系<br>2. 构建配额体系<br>3. 构建完善的监控体系<br>4. 开发方便的UI界面实现以上3点：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-07 07:20:01</div>
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
  <div class="_2_QraFYR_0">期待早点讲到监控这一章，作为运维，对监控的需求很迫切。现在只知道监控broker的jmx端口参数，另外用burrow监控过消费延时，但光这些监控还是感觉太少。最近有监控rabalance发生情况的需求，还没有思路。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 坦率说目前Kafka的免费监控方案没有特别好的，到时候我们一起聊聊。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-16 12:04:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/52/eb/eec719f3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>开水</span>
  </div>
  <div class="_2_QraFYR_0">老师，后续课程中会对比一下其他消息框架么。比如rocket，rabbit</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 坦率说不会有太多涉及，毕竟还是以介绍Apache Kafka为主</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-05 07:41:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/dd/e0/034ce26f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Fouy_飞虎</span>
  </div>
  <div class="_2_QraFYR_0">赞一个，正准备深入学习一下。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢支持与鼓励：） </p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-03 18:40:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/59/a0/66486058.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>robbin🐳</span>
  </div>
  <div class="_2_QraFYR_0">老师，我之前有老大和我说他们之前用kafka在量大的时候会丢数据，但我在网上和自己使用中并未遇到，想问老师是真的会有丢数据的情况吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我们这边量也很大，但是没有碰到过丢消息。可能还是配置的问题。当然Kafka重复消息倒是常见</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-23 11:15:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/f2/01/b78add60.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>流浪的阳光</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，请问kafca适合做两个系统之间的转账处理吗？<br><br>另外，请问kafca的使用案例中，最多支持过什么数量级的消费者和生产者。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 坦率说不合适，还是让数据库做吧~<br><br>你指的数量级是什么呢？如果是消息数，每天数十亿的系统我就接触过。国内大厂怕是更多了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-03 22:08:40</div>
  </div>
</div>
</div>
</li>
</ul>