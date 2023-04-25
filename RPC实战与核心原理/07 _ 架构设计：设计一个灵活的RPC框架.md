<audio title="07 _ 架构设计：设计一个灵活的RPC框架" src="https://static001.geekbang.org/resource/audio/39/1e/39db1403746007e0bece57ace8123f1e.mp3" controls="controls"></audio> 
<p>你好，我是何小锋。到今天为止，基础篇的知识我们就全部学习完了，接下来我们进入进阶篇。</p><p>在基础篇里面，我们讲了RPC的通信原理以及RPC里各个功能组件的作用，不妨用一段话再次回顾下：“其实RPC就是把拦截到的方法参数，转成可以在网络中传输的二进制，并保证在服务提供方能正确地还原出语义，最终实现像调用本地一样地调用远程的目的。”<strong>你记住了吗？</strong></p><p>那学到这儿，距离实现一个灵活的RPC框架其实还是有距离的。知道了各个功能组件只是迈出了第一步，接下来你必须要清楚各个组件之间是怎么完成数据交互的，这也是今天这讲的重点，我们一起搞清楚RPC的架构设计。</p><h2>RPC架构</h2><p>说起架构设计，我相信你一定不陌生。我理解的架构设计呢，就是从顶层角度出发，厘清各模块组件之间数据交互的流程，让我们对系统有一个整体的宏观认识。我们先看看RPC里面都有哪些功能模块。</p><p>我们讲过，RPC本质上就是一个远程调用，那肯定就需要通过网络来传输数据。虽然传输协议可以有多种选择，但考虑到可靠性的话，我们一般默认采用TCP协议。为了屏蔽网络传输的复杂性，我们需要封装一个单独的数据传输模块用来收发二进制数据，这个单独模块我们可以叫做传输模块。</p><p>用户请求的时候是基于方法调用，方法出入参数都是对象数据，对象是肯定没法直接在网络中传输的，我们需要提前把它转成可传输的二进制，这就是我们说的序列化过程。但只是把方法调用参数的二进制数据传输到服务提供方是不够的，我们需要在方法调用参数的二进制数据后面增加“断句”符号来分隔出不同的请求，在两个“断句”符号中间放的内容就是我们请求的二进制数据，这个过程我们叫做协议封装。</p><!-- [[[read_end]]] --><p><strong>虽然这是两个不同的过程，但其目的都是一样的，都是为了保证数据在网络中可以正确传输。</strong>这里我说的正确，可不仅指数据能够传输，还需要保证传输后能正确还原出传输前的语义。所以我们可以把这两个处理过程放在架构中的同一个模块，统称为协议模块。</p><p>除此之外，我们还可以在协议模块中加入压缩功能，这是因为压缩过程也是对传输的二进制数据进行操作。在实际的网络传输过程中，我们的请求数据包在数据链路层可能会因为太大而被拆分成多个数据包进行传输，为了减少被拆分的次数，从而导致整个传输过程时间太长的问题，我们可以在RPC调用的时候这样操作：在方法调用参数或者返回值的二进制数据大于某个阈值的情况下，我们可以通过压缩框架进行无损压缩，然后在另外一端也用同样的压缩算法进行解压，保证数据可还原。</p><p>传输和协议这两个模块是RPC里面最基础的功能，它们使对象可以正确地传输到服务提供方。但距离RPC的目标——实现像调用本地一样地调用远程，还缺少点东西。因为这两个模块所提供的都是一些基础能力，要让这两个模块同时工作的话，我们需要手写一些黏合的代码，但这些代码对我们使用RPC的研发人员来说是没有意义的，而且属于一个重复的工作，会导致使用过程的体验非常不友好。</p><p>这就需要我们在RPC里面把这些细节对研发人员进行屏蔽，让他们感觉不到本地调用和远程调用的区别。假设有用到Spring的话，我们希望RPC能让我们把一个RPC接口定义成一个Spring Bean，并且这个Bean也会统一被Spring Bean Factory管理，可以在项目中通过Spring依赖注入到方式引用。这是RPC调用的入口，我们一般叫做Bootstrap模块。</p><p><strong>学到这儿，一个点对点（Point to Point）版本的RPC框架就完成了。</strong>我一般称这种模式的RPC框架为单机版本，因为它没有集群能力。所谓集群能力，就是针对同一个接口有着多个服务提供者，但这多个服务提供者对于我们的调用方来说是透明的，所以在RPC里面我们还需要给调用方找到所有的服务提供方，并需要在RPC里面维护好接口跟服务提供者地址的关系，这样调用方在发起请求的时候才能快速地找到对应的接收地址，这就是我们常说的“服务发现”。</p><p>但服务发现只是解决了接口和服务提供方地址映射关系的查找问题，这更多是一种“静态数据”。说它是静态数据是因为，对于我们的RPC来说，我们每次发送请求的时候都是需要用TCP连接的，相对服务提供方IP地址，TCP连接状态是瞬息万变的，所以我们的RPC框架里面要有连接管理器去维护TCP连接的状态。</p><p>有了集群之后，提供方可能就需要管理好这些服务了，那我们的RPC就需要内置一些服务治理的功能，比如服务提供方权重的设置、调用授权等一些常规治理手段。而服务调用方需要额外做哪些事情呢？每次调用前，我们都需要根据服务提供方设置的规则，从集群中选择可用的连接用于发送请求。</p><p>那到这儿，一个比较完善的RPC框架基本就完成了，功能也差不多就是这些了。按照分层设计的原则，我将这些功能模块分为了四层，具体内容见图示：</p><p><img src="https://static001.geekbang.org/resource/image/30/fb/30f52b433aa5f103114a8420c6f829fb.jpg?wh=2951*2181" alt="" title="架构图"></p><h2>可扩展的架构</h2><p>那RPC架构设计出来就完事了吗？当然不，技术迭代谁都躲不过。</p><p>不知道你有没有这样的经历，你设计的一个系统它看上去很完善，也能很好地运行，然后你成功地把它交付给了业务方。有一天业务方有了新的需求，要加入很多新的功能，这时候你就会发现当前架构面临的可就是大挑战了，要修改很多地方才能实现。</p><p>举个例子，假如你设计了一个商品发布系统，早些年我们只能在网上购买电脑、衣服等实物商品，但现在发展成可以在网上购买电话充值卡、游戏点卡等虚拟商品，实物商品的发布流程是需要选择购买区域的，但虚拟商品并没有这一限制。如果你想要在一套发布系统里面同时完成实物和虚拟商品发布的话，你就只能在代码里面加入很多的if else判断逻辑，这样是能行，可整个代码就臃肿、杂乱了，后期也极难维护。</p><p>其实，我们设计RPC框架也是一样的，我们不可能在开始时就面面俱到。那有没有更好的方式来解决这些问题呢？这就是我们接下来要讲的插件化架构。</p><p>在RPC框架里面，我们是怎么支持插件化架构的呢？我们可以将每个功能点抽象成一个接口，将这个接口作为插件的契约，然后把这个功能的接口与功能的实现分离，并提供接口的默认实现。在Java里面，JDK有自带的SPI（Service Provider Interface）服务发现机制，它可以动态地为某个接口寻找服务实现。使用SPI机制需要在Classpath下的META-INF/services目录里创建一个以服务接口命名的文件，这个文件里的内容就是这个接口的具体实现类。</p><p>但在实际项目中，我们其实很少使用到JDK自带的SPI机制，首先它不能按需加载，ServiceLoader加载某个接口实现类的时候，会遍历全部获取，也就是接口的实现类得全部载入并实例化一遍，会造成不必要的浪费。另外就是扩展如果依赖其它的扩展，那就做不到自动注入和装配，这就很难和其他框架集成，比如扩展里面依赖了一个Spring Bean，原生的Java SPI就不支持。</p><p>加上了插件功能之后，我们的RPC框架就包含了两大核心体系——核心功能体系与插件体系，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/a3/a6/a3688580dccd3053fac8c0178cef4ba6.jpg?wh=3084*2183" alt="" title="插件化RPC"></p><p><strong>这时，整个架构就变成了一个微内核架构</strong>，我们将每个功能点抽象成一个接口，将这个接口作为插件的契约，然后把这个功能的接口与功能的实现分离并提供接口的默认实现。这样的架构相比之前的架构，有很多优势。首先它的可扩展性很好，实现了开闭原则，用户可以非常方便地通过插件扩展实现自己的功能，而且不需要修改核心功能的本身；其次就是保持了核心包的精简，依赖外部包少，这样可以有效减少开发人员引入RPC导致的包版本冲突问题。</p><h2>总结</h2><p>我们都知道软件开发的过程很复杂，不仅是因为业务需求经常变化，更难的是在开发过程中要保证团队成员的目标统一。我们需要用一种可沟通的话语、可“触摸”的愿景达成目标，我认为这就是软件架构设计的意义。</p><p>但仅从功能角度设计出的软件架构并不够健壮，系统不仅要能正确地运行，还要以最低的成本进行可持续的维护，因此我们十分有必要关注系统的可扩展性。只有这样，才能满足业务变化的需求，让系统的生命力不断延伸。</p><h2>课后思考</h2><p>你能分享一下，在日常工作中，你都有哪些地方是用到了插件思想来解决扩展性问题的吗？</p><p>欢迎留言和我分享你的思考，也欢迎你把文章分享给你的朋友，邀请他加入学习。我们下节课再见！</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/3a/27/5d218272.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>八台上</span>
  </div>
  <div class="_2_QraFYR_0">这个文章没有做到插件化～  与接口的实现（Java）耦合严重， 其他语言版本的无机可乘～</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-16 10:02:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/48/f3/f1034ffd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>cricket1981</span>
  </div>
  <div class="_2_QraFYR_0">如何实现按需加载的SPI? </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以参考springboot的Condition实现</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-04 08:39:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/7b/98/8f1aecf4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>楼下小黑哥</span>
  </div>
  <div class="_2_QraFYR_0">我们系统中有个路由模块，可以分发交易到指定交易渠道。新接入的模块，只需要接入规定的接口，就能被路由模块识别到。<br>我觉得这也是类似于插件化开发，每次新接入交易渠道，路由模块都是无需重启。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 很棒！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-06 09:18:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/26/38/ef063dc2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Darren</span>
  </div>
  <div class="_2_QraFYR_0">插件功能都可以通过类似链式调用的方式实现，可以全量加载，但按需激活，不管是condition还是SPI，然后把所有激活的保存在List中，然后执行的时候，通过链式的形式调用</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-17 17:51:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/22/0a/7d/ac715471.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>独孤九剑</span>
  </div>
  <div class="_2_QraFYR_0">架构模式三板斧：分层+分割+分布式，节点间通过RPC框架和分布式锁等机制进行协作。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-16 16:08:19</div>
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
  <div class="_2_QraFYR_0">这篇感觉很棒，高屋建瓴，一个完整的RPC架构设计图就出来了。每个部分具体怎么实现就因人因公司而异了，不过也是见证功力的地方，目前在研究thrift，感觉很有帮助。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 希望能帮助到你</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-14 08:28:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/f3/a0/a693e561.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>魔曦</span>
  </div>
  <div class="_2_QraFYR_0">这篇文章的设计思路很独特，按需加载，请问有按需加载的demo，配合demo跟有意义</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-26 17:32:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/4b/d7/f46c6dfd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>William Ning</span>
  </div>
  <div class="_2_QraFYR_0">完全避开的非java同学的问题</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-20 23:05:25</div>
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
  <div class="_2_QraFYR_0">想问下老师idea里面的插件是什么思想，但是现在的版本需要重启idea才能生效，但新的版本听说是不用重启就可以生效，想知道新技术是什么？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-22 10:55:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83epKJlW7sqts2ZbPuhMbseTAdvHWnrc4ficAeSZyKibkvn6qyxflPrkKKU3mH6XCNmYvDg11tB6y0pxg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>pc</span>
  </div>
  <div class="_2_QraFYR_0">完全没理解到底怎么实现插件的，或者有什么方式去实现插件的<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-21 10:21:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/7b/ae/66ae403d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>熊猫</span>
  </div>
  <div class="_2_QraFYR_0">老师，你好，有没有开源插件开发做的好的项目？看看怎么实现的</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-21 13:11:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/e5/a1/74259971.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>李逍遥</span>
  </div>
  <div class="_2_QraFYR_0">这算是模板方法设计模式的提现么？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-24 20:54:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/6c/ea/ce9854a5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>坤</span>
  </div>
  <div class="_2_QraFYR_0">WebHook机制算是插件机制的使用吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-15 18:39:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/e9/86/d34800a4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>heyman</span>
  </div>
  <div class="_2_QraFYR_0">设计好了编码格式：带有头部，头部指定payload长度。那是不是就不需要“断句”符号了呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-12 17:28:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/e5/10/0a94311f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>joker</span>
  </div>
  <div class="_2_QraFYR_0">老师，您好。关于您说的那个基于插件的微内核设计架构，我有几点不理解的地方：<br>1.这里的插件定义似乎和我理解的不一样：拿idea来举例，它有很丰富的插件系统，可以通过实现它设计的插件接口就可以丰富idea的功能，我理解它的插件接口应该是统一的吧。所以我理解插件应该是使用同一套接口，然后有多种不同的实现，插件之间是相互独立的，它们是可以共存的。不知我的理解是否正确<br>2.但是在我们这边的微内核架构中，同一个插件实际上只能有一个存在，也就是说同一个模块同一时间只能有一个实现在运行。这个也能称之为插件吗？感觉是定义了一套接口，大家可以提供不同实现，然后根据业务需求加载不同的实现<br>3.文中说到的按需加载，我不是很理解，这里如果要实现按需加载，是我们通过配置文件的形式指定当前要加载哪一个实现吗？也就是说需要重启服务才能激活新加载的实现，对吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-19 08:24:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/e1/19/c756aaed.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>鸠摩智</span>
  </div>
  <div class="_2_QraFYR_0">阿里开源的datax也用到了插件的设计</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 现在很多开源框架都用到了插件</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-09 08:53:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/dd/50/21eb5d76.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>三叶泷</span>
  </div>
  <div class="_2_QraFYR_0">在web 前后端分离的项目中，我们自定义了一套UI插件，通过后端下发规则控制显示，后端开发只需定义好显示规则即可，应该也是一种插件化开发吧。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-30 17:44:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/10/90/5cb92311.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>麦兜布熊</span>
  </div>
  <div class="_2_QraFYR_0">我是个小测试。使用jmeter进行压力测试。jmeter官网中支持进行定制sampler取样器，写好的jar包放在lib\ext下，再启动jmter时就能看到了。不知道这算不算插件化开发</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 插件化是一个概念，有很多种实现方式，这种也可以算。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-22 23:02:25</div>
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
  <div class="_2_QraFYR_0">还有spring的spring.factories这一套也是利用的面向接口编程的思想吧，感觉这个比jdk自带的spi也好很多，既然有些问题，那为啥jdk的spi不优化一下呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: jdk我理解更多是标准</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-11 23:48:29</div>
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
  <div class="_2_QraFYR_0">老师好，想问下用jdk自带spi一般会有一个接口加载很多实现类的情况吗，因为只能用迭代器遍历，导致只能用类型判断才能找到自己想要的类，这样感觉不够优雅吧，所以我感觉应该都是一个接口配置一个实现类这样就是使用者想要的情况了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 那样插件的意义就不存在了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-11 23:38:08</div>
  </div>
</div>
</div>
</li>
</ul>