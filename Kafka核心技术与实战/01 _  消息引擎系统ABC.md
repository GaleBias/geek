<audio title="01 _  消息引擎系统ABC" src="https://static001.geekbang.org/resource/audio/0b/4d/0b4336a9167249cf5e269de27944974d.mp3" controls="controls"></audio> 
<p>你好，我是胡夕。欢迎你来到“Kafka核心技术与实战”专栏。如果你对Kafka及其背后的消息引擎、流处理感兴趣，很高兴我们可以在此相聚，并在未来的一段日子里一同学习有关Kafka的方方面面。</p><p>毫无疑问，你现在对Apache Kafka一定充满了各种好奇，那么今天就允许我先来尝试回答下Kafka是什么这个问题。对了，先卖个关子，在下一期我还将继续回答这个问题，而且答案是不同的。那么，Kafka是什么呢？用一句话概括一下：<strong>Apache Kafka是一款开源的消息引擎系统</strong>。</p><p>倘若“消息引擎系统”这个词对你来说有点陌生的话，那么“消息队列”“消息中间件”的提法想必你一定是有所耳闻的。不过说实话我更愿意使用消息引擎系统这个称谓，因为消息队列给出了一个很不明确的暗示，仿佛Kafka是利用队列的方式构建的；而消息中间件的提法有过度夸张“中间件”之嫌，让人搞不清楚这个中间件到底是做什么的。</p><p>像Kafka这一类的系统国外有专属的名字叫Messaging System，国内很多文献将其简单翻译成消息系统。我个人认为并不是很恰当，因为它片面强调了消息主体的作用，而忽视了这类系统引以为豪的消息传递属性，就像引擎一样，具备某种能量转换传输的能力，所以我觉得翻译成消息引擎反倒更加贴切。</p><!-- [[[read_end]]] --><p>讲到这里，说点题外话。我觉得目前国内在翻译国外专有技术词汇方面做得不够标准化，各种名字和提法可谓五花八门。我举个例子，比如大名鼎鼎的Raft算法和Paxos算法。了解它的人都知道它们的作用是在分布式系统中让多个节点就某个决定达成共识，都属于Consensus Algorithm一族。如果你在搜索引擎中查找Raft算法，国内多是称呼它们为一致性算法。实际上我倒觉得翻译成共识算法是最准确的。我们使用“一致性”这个字眼太频繁了，国外的Consistency被称为一致性、Consensus也唤作一致性，甚至是Coherence都翻译成一致性。</p><p>还是拉回来继续聊消息引擎系统，那这类系统是做什么用的呢？我先来个官方严肃版本的答案。</p><p>根据维基百科的定义，消息引擎系统是一组规范。企业利用这组规范在不同系统之间传递语义准确的消息，实现松耦合的异步式数据传递。</p><p>果然是官方定义，有板有眼。如果觉得难于理解，那么可以试试我下面这个民间版：</p><p>系统A发送消息给消息引擎系统，系统B从消息引擎系统中读取A发送的消息。</p><p>最基础的消息引擎就是做这点事的！不论是上面哪个版本，它们都提到了两个重要的事实：</p><ul>
<li>消息引擎传输的对象是消息；</li>
<li>如何传输消息属于消息引擎设计机制的一部分。</li>
</ul><p>既然消息引擎是用于在不同系统之间传输消息的，那么如何设计待传输消息的格式从来都是一等一的大事。试问一条消息如何做到信息表达业务语义而无歧义，同时它还要能最大限度地提供可重用性以及通用性？稍微停顿几秒去思考一下，如果是你，你要如何设计你的消息编码格式。</p><p>一个比较容易想到的是使用已有的一些成熟解决方案，比如使用CSV、XML亦或是JSON；又或者你可能熟知国外大厂开源的一些序列化框架，比如Google的Protocol Buffer或Facebook的Thrift。这些都是很酷的办法。那么现在我告诉你Kafka的选择：它使用的是纯二进制的字节序列。当然消息还是结构化的，只是在使用之前都要将其转换成二进制的字节序列。</p><p>消息设计出来之后还不够，消息引擎系统还要设定具体的传输协议，即我用什么方法把消息传输出去。常见的有两种方法：</p><ul>
<li><strong>点对点模型</strong>：也叫消息队列模型。如果拿上面那个“民间版”的定义来说，那么系统A发送的消息只能被系统B接收，其他任何系统都不能读取A发送的消息。日常生活的例子比如电话客服就属于这种模型：同一个客户呼入电话只能被一位客服人员处理，第二个客服人员不能为该客户服务。</li>
<li><strong>发布/订阅模型</strong>：与上面不同的是，它有一个主题（Topic）的概念，你可以理解成逻辑语义相近的消息容器。该模型也有发送方和接收方，只不过提法不同。发送方也称为发布者（Publisher），接收方称为订阅者（Subscriber）。和点对点模型不同的是，这个模型可能存在多个发布者向相同的主题发送消息，而订阅者也可能存在多个，它们都能接收到相同主题的消息。生活中的报纸订阅就是一种典型的发布/订阅模型。</li>
</ul><p>比较酷的是Kafka同时支持这两种消息引擎模型，专栏后面我会分享Kafka是如何做到这一点的。</p><p>提到消息引擎系统，你可能会问JMS和它是什么关系。JMS是Java Message Service，它也是支持上面这两种消息引擎模型的。严格来说它并非传输协议而仅仅是一组API罢了。不过可能是JMS太有名气以至于很多主流消息引擎系统都支持JMS规范，比如ActiveMQ、RabbitMQ、IBM的WebSphere MQ和Apache Kafka。当然Kafka并未完全遵照JMS规范，相反，它另辟蹊径，探索出了一条特有的道路。</p><p>好了，目前我们仅仅是了解了消息引擎系统是做什么的以及怎么做的，但还有个重要的问题是为什么要使用它。</p><p>依旧拿上面“民间版”举例，我们不禁要问，为什么系统A不能直接发送消息给系统B，中间还要隔一个消息引擎呢？</p><p>答案就是“<strong>削峰填谷</strong>”。这四个字简直比消息引擎本身还要有名气。</p><p>我翻了很多文献，最常见的就是这四个字。所谓的“削峰填谷”就是指缓冲上下游瞬时突发流量，使其更平滑。特别是对于那种发送能力很强的上游系统，如果没有消息引擎的保护，“脆弱”的下游系统可能会直接被压垮导致全链路服务“雪崩”。但是，一旦有了消息引擎，它能够有效地对抗上游的流量冲击，真正做到将上游的“峰”填满到“谷”中，避免了流量的震荡。消息引擎系统的另一大好处在于发送方和接收方的松耦合，这也在一定程度上简化了应用的开发，减少了系统间不必要的交互。</p><p>说了这么多，可能你对“削峰填谷”并没有太多直观的感受。我还是举个例子来说明一下Kafka在这中间是怎么去“抗”峰值流量的吧。回想一下你在极客时间是如何购买这个课程的。如果我没记错的话极客时间每门课程都有一个专门的订阅按钮，点击之后进入到付费页面。这个简单的流程中就可能包含多个子服务，比如点击订阅按钮会调用订单系统生成对应的订单，而处理该订单会依次调用下游的多个子系统服务 ，比如调用支付宝和微信支付的接口、查询你的登录信息、验证课程信息等。显然上游的订单操作比较简单，它的TPS要远高于处理订单的下游服务，因此如果上下游系统直接对接，势必会出现下游服务无法及时处理上游订单从而造成订单堆积的情形。特别是当出现类似于秒杀这样的业务时，上游订单流量会瞬时增加，可能出现的结果就是直接压跨下游子系统服务。</p><p>解决此问题的一个常见做法是我们对上游系统进行限速，但这种做法对上游系统而言显然是不合理的，毕竟问题并不出现在它那里。所以更常见的办法是引入像Kafka这样的消息引擎系统来对抗这种上下游系统TPS的错配以及瞬时峰值流量。</p><p>还是这个例子，当引入了Kafka之后。上游订单服务不再直接与下游子服务进行交互。当新订单生成后它仅仅是向Kafka Broker发送一条订单消息即可。类似地，下游的各个子服务订阅Kafka中的对应主题，并实时从该主题的各自分区（Partition）中获取到订单消息进行处理，从而实现了上游订单服务与下游订单处理服务的解耦。这样当出现秒杀业务时，Kafka能够将瞬时增加的订单流量全部以消息形式保存在对应的主题中，既不影响上游服务的TPS，同时也给下游子服务留出了充足的时间去消费它们。这就是Kafka这类消息引擎系统的最大意义所在。</p><p>如果你对Kafka Broker、主题和分区等术语还不甚了解的话也不必担心，我会在专栏后面专门花时间介绍一下Kafka的常见概念和术语。</p><p>在今天结束之前，我还想和你分享一个自己的小故事。在2015年那会儿，我花了将近1年的时间阅读Kafka源代码，期间多次想要放弃。你要知道阅读将近50万行源码是多么痛的领悟。我还记得当初为了手写源代码注释，自己写满了一个厚厚的笔记本。不过幸运的是我坚持了下来，之前的所有努力也没有白费，以至于后面写书、写极客时间专栏就变成了一件件水到渠成的事情。</p><p>最后我想送给你一句话：<strong>聪明人也要下死功夫</strong>。我不记得这是曾国藩说的还是季羡林说的，但这句话对我有很大影响，当我感到浮躁的时候它能帮我静下心来踏踏实实做事情。希望这句话对你也有所启发。切记：聪明人要下死功夫！</p><p><img src="https://static001.geekbang.org/resource/image/8b/26/8bc58bf5bb98db09fd6ef343e0f28826.jpg?wh=2069*2560" alt=""></p><h2>开放讨论</h2><p>请谈谈你对消息引擎系统的理解，或者分享一下你的公司或组织是怎么使用消息引擎来处理实际问题的。</p><p>欢迎写下你的思考和答案，我们一起讨论。如果你觉得有所收获，也欢迎把文章分享给你的朋友。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/12/73/2183839d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>huaweichen</span>
  </div>
  <div class="_2_QraFYR_0">曾国藩：真正聪明人都在下笨功夫！<br><br>https:&#47;&#47;zhuanlan.zhihu.com&#47;p&#47;25100394</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-04 14:37:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/3b/ad/31193b83.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>孙志强</span>
  </div>
  <div class="_2_QraFYR_0">讲讲怎么把50完行源代码读下来的? 嘿嘿</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 一行一行啃下来的。如果你也有兴趣，我建议可以先从kafka.log包开始读起，会很有收获的~~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-04 16:00:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/08/3f/820ae54e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>开发无止境，BUG随身行</span>
  </div>
  <div class="_2_QraFYR_0">有个问题请教下老师:<br>之前也用过kafka，怎么解决实时结果响应问题呢？比如秒杀商品，生产者产生订单，消费者处理订单结果，那这结果如何实时返回给用户呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个场景使用Kafka Streams比较适合，它就是为read-process-write场景服务的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-03 21:27:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/cb/05/64d3b05a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Lei Yang</span>
  </div>
  <div class="_2_QraFYR_0">老师可以讲一讲Kafka和别的mq的区别和最佳选择方法么？例如什么时候选择RabbitMQ什么时候选择Kafka等等</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: RabbitMQ属于比较传统的消息队列系统，支持标准的消息队列协议（AMQP, STOMP，MQTT等），如果你的应用程序需要支持这些协议，那么还是使用RabbitMQ。另外RabbitMQ支持比较复杂的consumer Routing，这点也是Kafka不提供的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-03 22:44:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/20/08/bc06bc69.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Dovelol</span>
  </div>
  <div class="_2_QraFYR_0">老师好，想问下有些业务用mq来做异步处理，为了削峰填谷，是不是上游发送消息成功就认为业务成功了，可能下游过很久去消费，那实时性要求很高的业务怎么办呢，比如生成了订单但是一直不处理也不好吧。另外想请教下老师的角度来讲下mq和rpc调用的区别是什么呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: mq和rpc的区别往大了说属于数据流模式（dataflow mode）的问题。我们常见的数据流有三种：1. 通过数据库；2. 通过服务调用（REST&#47;RPC）; 3. 通过异步消息传递（消息引擎，如Kafka）<br>RPC和MQ是有相似之处的，毕竟我们远程调用一个服务也可以看做是一个事件，但不同之处在于：<br>1. MQ有自己的buffer，能够对抗过载（overloaded）和不可用场景<br>2. MQ支持重试<br>3. 允许发布&#47;订阅模式<br>当然它们还有其他区别。应该这样说RPC是介于通过数据库和通过MQ之间的数据流模式。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-03 20:13:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/b4/50/c5dad2dc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Shane</span>
  </div>
  <div class="_2_QraFYR_0">老师，今天才学习到这篇文章，还是老师能够在百忙之中抽出时间来解答我的困惑。<br>这篇文章提到了消息的协议，老师这里介绍了两种模式一种是点对点，一种是订阅，发布模式。但是，为什么我一开始想到消息的协议是http之类的传输协议？这两个有什么区别和联系？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: http不属于消息传输协议，它是网络通信协议的一种，严格来说这是两个范畴或者说是两个层次上的协议。<br><br>通常来说，两个进程进行数据流交互的方式一般有三种：<br>1. 通过数据库：进程1写入数据库；进程2读取数据库<br>2. 通过服务调用：比如REST或RPC，而HTTP协议通常就作为REST方式的底层通讯协议<br>3. 通过消息传递的方式：进程1发送消息给名为broker的中间件，然后进程2从该broker中读取消息。消息传输协议属于这种模式<br><br>因此我说虽然我们都称它们为协议，但它们不是一个层次上的协议。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-14 23:40:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/50/2b/2344cdaa.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>第一装甲集群司令克莱斯特</span>
  </div>
  <div class="_2_QraFYR_0">我们公司用kafaka通过埋点，日志分析，做链路监控，某个业务接口出现问题，预警系统发送消息给处理人。很及时有效，不用等运维那么慢的反馈了。合作方对比处理效率也很满意。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-07 22:54:20</div>
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
  <div class="_2_QraFYR_0">pulsar高吞吐低延迟和kafka谁会主宰未来？夕哥、能不能拓展下flink+kafka的耦合！谢谢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 和Pulsar的斯杰、翟佳都相识，不敢妄下结论。Flink + Kafka最近的确有标准套餐的趋势：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-03 19:35:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/66/bd/bd5d503e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>杨鹏程baci</span>
  </div>
  <div class="_2_QraFYR_0">胡夕老师好，我是第一次在这提问，这门课程我应该是0基础了，有一些疑问希望老师帮忙解答一下，用消息引擎的这种数据流数据方式，上游是不是就无法得知处理结果了，甚至是无法将返回值传回上游了？谢谢！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯嗯，确实不太容易。因为这种通信方式一般是异步且是单向的，如果你需要这种回馈机制，最好使用服务调用 的方式</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-20 09:02:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/f9/bf/bcf0fdf9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>gnayuh</span>
  </div>
  <div class="_2_QraFYR_0">老师讲的很好，我是之前那种听过kafaka等消息引擎大名的初学者，听完第一节课，联想到这个发布订阅模型跟之前学过java设计模式之观察者，总感觉它们之间有那么点类似，想知道它们之间的某种关系。还有就是学操作系统时候的生产者消费者也很像</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 它们的确很类似。特别是发布&#47;订阅与观察者模式。在《Head first Design Pattern》一书中更是有这样的话： Publishers + Subscribers = Observer Pattern<br><br>不过细究起来还是有些许不同，pub与sub之间通常都隔了一层，比如broker或message channel，但是Observer模式中Observer通常都直接对接被观测者，因此Pub&#47;Sub模式中组件的耦合度更低；另外Pub&#47;Sub经常是以异步的方式实现，而Observer模式通常都是同步的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-03 22:57:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/d3/db/5ce5bb26.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>燕子上</span>
  </div>
  <div class="_2_QraFYR_0">我司直接把kafka当mq来使用，高吞吐、低延迟、松耦合。对！我司看上了松耦合，哪哪都要用kafka解耦，真正的面向kafka编程🤣</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-04 00:26:22</div>
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
  <div class="_2_QraFYR_0">1. consesus algorithm，在区块链中多翻译为共识算法，而在其它领域多被翻译为一致性算法，个人觉得共识算法表意更清楚。<br><br>2. 削峰填谷，实际上就是流量整形的形象表达，主要还是为了应对上游瞬时大流量的冲击，避免出现流量毛刺现象，保护下游应用和数据库不被大流量打垮。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-03 19:04:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/cf/76/9e413c61.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Bin滨</span>
  </div>
  <div class="_2_QraFYR_0">谢谢知识分享。<br>在Martin Kleppmann 的书中把kafka 定义成 log-based message brocker， 这个基本上是对kafka最简单的定义了。append-only， partition， totally ordered 是比较需要理解的概念。 </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯嗯，DDIA是一本神书：）<br><br>其实这个定义最早还是Kafka作者Jay Kreps提出的，有兴趣可以看看Kafka的论文：http:&#47;&#47;notes.stephenholiday.com&#47;Kafka.pdf<br>以及Jay Kreps的 《I ❤️ Logs》</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-19 05:02:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/ca/bd/a51ae4b2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>吃饭饭</span>
  </div>
  <div class="_2_QraFYR_0">胡老师好，我想请教一个学习方法，我在做Kafka测试的时候遇到一个问题，我记得以前老版本的时候使用命令行进行Demo测试时，消费消息到控制台使用：bin&#47;kafka-console-consumer.sh --zookeeper localhost:2181&#47;kafka --topic test 就可以，但是今天我换了高版本发现不对了，以前的 --zookeeper 新版本不支持了。知道这点后我希望能够从官网找到具体是哪个版本开始删除这个指令的以及删除的原因，但是我这种为题我不会查询官网，只能从百度等搜索引擎上看其他人的一些总结，希望老师能给示例一番，感谢。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 指定--zookeeper是老版本的消费者，新版本需要指定--bootstrap-server。新版本消费者API是0.9版本引入的，主要是为了移除消费者API对ZooKeeper的依赖。专栏后面有文章谈到这一点。<br>新版本使用方法：<br>bin&#47;kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-03 23:51:17</div>
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
  <div class="_2_QraFYR_0">Kafka官网的描述是“Apache Kafka® is a distributed streaming platform.”，我觉得这里的重点在于分布式和流式处理，而且我认为消息引擎也可以看做是流式处理的一种，不知道老师怎么看？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Kafka是以消息引擎起家的，后面转型成流处理平台。没有冒犯的意思，我不认为消息引擎是流处理的一种。事实上，流处理在意的是如何处理无限数据集的问题。它们是不同的领域：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-03 17:28:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/fb/58/6c422e70.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>清晨吼于林</span>
  </div>
  <div class="_2_QraFYR_0">1、A系统为什么不能直接把消息发送给B系统？ 这可以出一个面试题，😆<br>2、作者的学习经历确实让人很振奋，可不可以花一个章节，专门讲讲，你当时是怎么读kafka的源码的？🙏</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-11 16:37:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/ac/46/7e24bad6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>杨俊</span>
  </div>
  <div class="_2_QraFYR_0">希望后面能说下要是kafka突然宕机或者临时停止服务进行更新，上游服务的消息该怎么正确更好处理呢？怎么保证消息的能够在kafka恢复工作的时候正确传递，谢谢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果是升级Kafka这种主动停机，应该采用rolling upgrade来做，不至于服务中断。如果是大面积突然宕机，快速处理反而是最重要的。如果在乎上游系统的消息delivery语义，增加retries的同时试试幂等producer吧</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-04 08:00:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/52/91/c0414437.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>草头</span>
  </div>
  <div class="_2_QraFYR_0">50万行，我的天……大牛都是这样炼成的！向大佬看齐，做不到喊喊口号也好！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-03 22:09:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/f6/96/cc4d553e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>安不安生</span>
  </div>
  <div class="_2_QraFYR_0">我们公司用来传输视频切片，然后使用集群进行视频分析，之前曾经用过kafka ，因为没有人熟悉，不会维护，导致放弃，现在使用aws kinesis 服务，怎么才能说服领导引进kafka 呢？<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: hmmm... 使用Kafka自己把控度会高一些。另外很多公司对数据出公网是有顾虑的，使用云上的服务必然涉及到将 公司数据传给云服务器的问题。如果是敏感数据这也是要考虑的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-03 19:23:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/c8/10/36f36b42.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>tracy</span>
  </div>
  <div class="_2_QraFYR_0">现在消息中间件很多，想要了解kafka和其他消息中间件的优劣点，系统选型时需要考虑什么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果是以实现高吞吐量为主要目标，Kafka是不错的首选；如果是以实现业务系统为主要目标，特别是金融类业务，可以考虑应用Kafka的流处理组件Kafka Streams。不过坦率说目前将Kafka应用于纯业务系统的并不多，但是前景依然可期：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-03 18:13:45</div>
  </div>
</div>
</div>
</li>
</ul>