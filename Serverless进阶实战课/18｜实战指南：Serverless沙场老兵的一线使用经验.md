<audio title="18｜实战指南：Serverless沙场老兵的一线使用经验" src="https://static001.geekbang.org/resource/audio/2f/b1/2f5a4332a7a8ab4b74e69b1f95780bb1.mp3" controls="controls"></audio> 
<p>你好，我是静远。</p><p>恭喜你完成了Serverless核心技术和扩展能力的学习，从这节课开始，我们就要运用之前所学的知识点，开始实战演练了。</p><p>前面我们讲到的触发器、冷启动、扩缩容、运行时、可观测等知识点，相信你都已经掌握得差不多了，随便拿出一个相关的问题，你也可以快速应对。但是，这些零碎的“武功招式”怎么才能在实战中运用起来呢？Serverless架构模式的开发和传统架构应用的开发如此不同，我们需要格外注意哪些“坑”呢？</p><p>今天，我就把平常“取经”得到的经验以及自己总结出来的“客户常见难题”，分成方案设计、资源评估、调用速度、开发技巧、线上运维等几个方面，跟你聊聊<strong>如何用好“Serverless”</strong>。</p><p>希望这节课，让你能够在选择合适的方案和开发手段这些方面得到启发，真正享受Serverless带来的红利。</p><h2>方案选型</h2><p>方案选型的时候，选错场景和技术大方向是比较忌讳的，不仅可能导致服务不稳定，还会导致修补工作量上升。那么，什么情况下适合用Serverless技术呢？</p><p>比较适合Serverless技术的场景，通常由<strong>数据处理计算、应用后端服务、事件触发处理、有状态服务和传统服务升级</strong> 5个维度组成。</p><p><img src="https://static001.geekbang.org/resource/image/23/66/23f35b9eba72ac0e2yy8d79c52037a66.png?wh=1920x1060" alt="图片"></p><p>接下来，我们可以对每一个业务的特性做一个抽象总结，看看5个维度不同的侧重点，让你在日新月异的Serverless场景迭代下，也能够自己动手，丰富这张技术选型图谱。</p><!-- [[[read_end]]] --><ol>
<li>数据处理计算的场景</li>
</ol><p>如果是大流量数据，且没有峰谷显著特性的话，你完全可以使用Spark、Flink这样的技术来处理，只有在峰谷显著的时候，Serverless的“极致弹性”才有发挥的余地。也就是，<strong>数据场景，流量变化较大、需要算力动态调配的时候，选用Serverless</strong>。</p><ol start="2">
<li>事件触发类场景</li>
</ol><p>这个场景是从FaaS诞生就常用的领域了。它的特性在于“流量的不稳定”，有就触发，没有就缩容至0。如果你的<strong>业务场景比较轻量，且流量不稳定，那么“精益成本”（按需使用，按量付费）的Serverless就是你的最佳选择</strong>。</p><ol start="3">
<li>应用后端服务</li>
</ol><p>区别于传统的微服务应用，应用后端服务更多的是<strong>对于微服务的一种补充，多用于相对独立、架构简单的业务应用</strong>。比如小程序、H5活动页、BFF的场景，它印证了Serverless“快速交付”这一特点。</p><p>这三个场景，我用蓝色做了区分，不管是通过函数编写还是自定义镜像，FaaS形态的技术都足以支撑了。而黄色的部分代表Serverless内涵延伸后，尤其是近几年技术底座和产品发展后，能够支撑更多的场景。我们继续往下说。</p><ol start="4">
<li>有状态的场景</li>
</ol><p>有状态的场景<strong>打破了之前狭义上Serverless无状态的特性</strong>，让长进程、多任务、有状态需求的交互也能在Serverless下完成。</p><ol start="5">
<li>传统服务升级</li>
</ol><p>当我们遇到<strong>历史包袱比较重</strong>，比如服务架构集成了SpringBoot、SpringCloud、Dubbo等框架的情况，就可以考虑通过Serverless托管服务的模式转型了。另外如果<strong>希望省去K8<strong><strong>s</strong></strong>的运维和容量的规划成本</strong>，考虑Serverless的容器转型，也可以划分在这类场景中。</p><p>除去这张图上的5个维度，反过来想一想，哪些情况不太建议使用Serverless技术呢？</p><p>我们之前选入的场景，都具有“运维门槛较高需要免运费和全托管”“流量变化较大需要弹性伸缩”“轻量应用需要快速交付”“流量不确定需要能缩容至0，按需使用按量付费”的特性，那么反过来，比如延时非常敏感的、SLA级别极高的场景，比如类似网页搜索、银行交易，就不太建议使用Serverless技术了。</p><p>在这里，我也给你分享一个技巧：<strong>你在初次接触Serverless的时候，可以多参考云厂商官网的案例，提工单和他们的售前工程师交流，可以帮助你更好的来选型和用好他们的产品</strong>。虽然我前面的课程中，已经给出了很多一线案例和经验，但<strong>Serverless的发展是日新月异的，紧贴官网是最好的选择之一</strong>。</p><h2>资源评估</h2><p>方案确定之后，我们就要做资源评估了。虽然Serverless具备按需使用、按量付费的特点，但这并不能说明Serverless的架构所产生的费用就比传统服务低。</p><p>你可以尝试按照这两个步骤评估：</p><p>第一，你的服务一天跑下来，是否存在明显的空闲期或者具备明显的“峰谷”现象？</p><p>第二，具备第一点后，是否选择了合适的资源规格去计算、对比各个方案费用？</p><p><strong>这两点适用于所有选用Serverless技术的产品</strong>。我们拿函数计算来说，它的收费是按照<strong>调用次数、资源使用耗时和出网流量</strong>来计算的。从各个云厂商的计价表可以看出，主要的费用来自于这三点中的资源使用耗时，而“资源耗时=内存大小*运行时间”。</p><p>从公式可以看出，资源的耗时成本是和内存大小、运行时间成正比关系的。在资源评估阶段，我们先说<strong>内存大小</strong>的问题。</p><p>我之前遇到一个客户，他的调用次数每天大概是50W次，每次执行在3秒左右，但费用还不低。</p><p>通过平台监控，我们发现他为了处理少部分数据量计算比较大的文件，准备的是1G的内存，这个数量只有不到1000次的调用量。</p><p>听到这里，你有啥感想呢？不急，我们先来看一下这两种内存下的收费情况，我以阿里云函数计算<a href="https://help.aliyun.com/document_detail/54301.html?spm=5176.137990.J_5253785160.6.32cc1608XmNBWP">FC活跃实例资源价格</a>计算如下：</p><p><img src="https://static001.geekbang.org/resource/image/13/43/1368d9d09ee36a37653a8e2a66884243.jpg?wh=1246x345" alt="图片"></p><p>我们可以发现，在不同的内存资源下，费用相差4倍。所以，当时我们给客户的建议，是在第一个函数 Func1 中判断一下文件大小，如果超过256MB承受的范围，就通过异步策略发往下一个函数 Func2 处理，这个函数设置为1GB，就可以在比较低的成本下完成他的业务处理。</p><p>我们来看一下费用对比：</p><p><img src="https://static001.geekbang.org/resource/image/d3/35/d3c5501b81c237a3a1de509840cc9335.jpg?wh=1701x362" alt="图片"></p><p>所以，在资源评估时，<strong>依据情况合理地进行函数拆分</strong>，也是至关重要的。</p><h2>调用速度</h2><p>刚才公式里面的第二个关键因素是“运行时间”，这就和调用速度相关了。那影响调用速度的因素有哪些？或者说，调用速度卡在哪里了呢？</p><p>从我跟客户打交道的经验来看，常见的有这么几个点，也是代码编程中容易忽视Serverless这种开发模式的地方。</p><ul>
<li><strong>代码包太大，引入了不必要的依赖包</strong>：这种情况尤其在Java的开发者中特别常见，要善于用exclude去掉一些不相关的依赖。</li>
<li><strong>运行时的影响</strong>：之前我的一个客户，由于使用了Java，导致出现偶发的超时错误，发现就是冷启动的时候耗时较长，后来干脆替换成了Python就没有这种问题了。尽管在阿里云等厂商对Java的运行时做了优化，但还记得我之前在运行时小节中跟你对比的么，Java的耗时除了在冷启动，在Invocation阶段也比Golang要长。</li>
<li><strong>调用下游服务返回慢</strong>：要知道，函数是按照使用次数和时长来计费的，抛开这个不说，函数的超时时间设定的限制，如果太慢的下游服务返回，会出现超时错误。不仅浪费成本，还使得服务体验不好。</li>
<li><strong>代码习惯问题</strong>：比如引入不必要的框架，它确实可以加快开发的速度，但也会影响服务的速度；另外，就是不必要的循环和Sleep关键字在代码中的使用，这些都是在设计函数代码架构和逻辑中格外需要注意的。</li>
</ul><p>总结下来，攻克这些问题需要做的就是两件事，<strong>一个是函数特性的优化，另一个是函数之间的协作优化</strong>。那么，作为一名Serverless平台的用户，具体可以做些什么呢？</p><p>在函数特性优化方面，你可以做5件事：</p><ol>
<li>尽可能<strong>减少不必要的代码依赖</strong>，<strong>优化代码体积</strong>，提升代码的下载和加载速度；</li>
<li><strong>选择性能较高的运行时</strong>，提升函数的启动速度，比如Golang语言；</li>
<li><strong>合理利用本地缓存</strong>，对于比较大的数据，可以适当缓存在挂载路径中；</li>
<li><strong>预留实例</strong>。我们知道Serverless的实例在没有流量进来的时候，是会缩容掉的，那么下次再有流量进来，就会进入冷启动的流程，你可以在成本允许的范围内对部分延时敏感性较高的函数设置一定的实例进行预留和分策略加载；</li>
<li><strong>请求预热</strong>。你可以通过配置一个定时触发器的方式，来主动调用函数来请求预热，设置的时间间隔你可以根据云厂商的实例回收时间来灵活配置。</li>
<li><strong>开发模式</strong>。尽量让自己的代码在一层的处理逻辑中，以“平铺”的方式解决，避免层层嵌套、循环、停顿这种在微服务或者脚本上经常使用的习惯发生。</li>
</ol><p>针对函数之间的协作，你同样可以做如下优化，来提升调用的速度：</p><ol>
<li><strong>尽可能使用异步调用</strong>，尤其是级联的函数调用避免同步调用下游服务产生的耗时，因此我们在选型的时候得要注意评估；</li>
<li><strong>一个函数只做一件事情</strong>，也就是说要根据业务进行合理的函数粒度拆分。在成本、复用性、性能上都存在其必要性。</li>
</ol><p>这样的优化能带来什么好处呢？相信学过冷启动的你能立马回答出来：提升服务的体验。如果你是一个HTTP网站，服务响应越慢，用户的流失也就越严重，间接上你的页面访问和流量也会降低，最后影响的是营收。</p><h2>开发技巧</h2><p>接下来，我们就到了开发阶段的讨论了。我会从开发工具选择以及编码技巧两个方面切入，这也是我们工作中最重要的两个抓手。</p><h3>工具的选择</h3><p>先说工具的选择。一般云厂商都会提供各种方便的工具来支撑项目的开发、调试、打包、发布和运维，实现项目全生命周期的管理能力。比如阿里云的Serverless Devs，腾讯云的Serverless Framework CLI，百度智能云的BSAM等等。</p><p><img src="https://static001.geekbang.org/resource/image/6c/f7/6c035cd8abffabc270d57437c821ddf7.png?wh=1364x560" alt="图片"></p><p>开发工具一般分为三种，本地开发工具CLI、WebIDE和IDE插件集成。其中，WebIDE的能力更偏向支持轻量化的在线开发管理，IDE插件的能力更偏向照顾那些使用成熟IDE的技术人员，如VSCode的开发人员。</p><p>那面对这些工具我们该怎么选呢？</p><ul>
<li>如果你的<strong>业务比较简单，采用的又是解释型语言，那么首选WebID</strong>E，关于它的优势和好处，可以在回到<a href="https://time.geekbang.org/column/article/573015">第11节课</a>回顾一下加深一下记忆；</li>
<li>如果你的业务比较复杂或者是编译型语言，那么我推荐你使用本地CLI工具或者插件集成。</li>
</ul><p>这几类工具的区别在于，如果你希望交互体验好一点，并且对类似VSCode的IDE工具开发的比较顺手了，那么将插件绑定在编辑器上是一个不错的选择。而类似Serverless Devs的这种工具，如果你是富有极客精神的命令行狂热爱好者，你可以用起来，丰富的命令足可以支撑你完成项目的管理工作了。</p><h3>编码技巧</h3><p>有了开发工具，那编码有啥需要注意的呢？一方面，是资源的初始化复用思想；另一方面，是函数业务的调用和设置细节。我们通过一个例子来看一下。</p><p>我在运行时小节中跟你提到过，基于标准运行时构建的函数其实是基于一个代码框架运行的。平台提供给你的是一个handler的入口函数，你可以直接在这个函数里编写你的业务代码逻辑。但如果函数需要和资源打交道，那么就需要在执行存储操作之前先建立连接了。</p><p>如果我们在handler函数中写了如下一段连接数据库的代码，想一想，这会有啥问题？</p><p>是不是每次请求过来都要重新连接做一遍这个操作？如果这是一个偶尔触发一次的函数，可能看不出问题。但如果一旦并发量比较高的服务场景，就会极大的影响性能，还可能导致数据库的连接句柄超限。</p><pre><code class="language-python">import pymysql
def handler(event, context):
  db =pymysql.connect(
  &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; host=os.Getenv("MYSQL_HOST"),
  &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; port=os.Getenv("MYSQL_PORT"),
  &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; user=os.Getenv("MYSQL_USER"),
  &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; passwd=os.Getenv("MYSQL_PASSWORD"),
  &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; charset='utf8',
  &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; db=os.Getenv("MYSQL_DBNAME")
  )
  cursor = db.cursor()
</code></pre><p>正确的做法应该把它单独拎出来，放到一个函数中，然后单独调用一次，这样在函数实例被回收之前，如果有新的请求过来，就可以直接复用了。</p><p>这个解决方案不仅利用了FaaS平台的<strong>实例复用</strong>技术，也在实例复用的基础上<strong>复用了资源对象</strong>。如果你还想进一步优化，也可以在给数据库对象db赋值和获取的时候判断是否为Null值，使得代码更鲁棒。类似地，像kafkaClient、redisClient等中间件资源的初始化都可以这样进行。</p><p>资源的初始化这个关键问题非常关键，一旦处理不好，可能还会导致资源服务的连接超限等问题；除此以外，根据客户常见的业务逻辑问题，我综合选取了所有技巧中常用、高效方面的TOP5，希望你能灵活用起来。</p><p><strong>首先，print日志的时候尽可能少而精</strong><strong>。</strong>如果你使用的是云平台，直接print“event”这种整个结构体很可能会撑爆日志的限制。所以，尽量打印出你需要的业务字段。</p><p><strong>其次，尽量使用异步策略代替微服务形态下的异步框架</strong><strong>。</strong>假如你在原生的微服务中使用Python的Tornado框架，你会发现FaaS形态的Serverless是很难发挥其异步的价值的。这个时候，你就可以直接选用我在第3讲<a href="https://time.geekbang.org/column/article/561957">高级应用</a>中跟你提到的FaaS平台的异步处理框架。</p><p><strong>再次，调用下游服务的接口，设置的超时时间一定要小于函数的超时时间</strong><strong>。</strong>否则会因为短板在下游，而造成一次请求超时。浪费成本不说，也会让服务不稳定。</p><p><strong>另外，要将公共代码放到层上。</strong>这也是我在高级属性这一节中跟你提到的必备高级技能之一，它类似于我们平时的公共类库，可以让具体的函数代码包更轻量。</p><p><strong>最后，提前设置并发数的上限</strong><strong>。</strong>一般云厂商都会设置默认的实例并发数，比如阿里云FC限定300 * InstanceConcurrency，百度智能云CFC限定单实例100等，这对于一个并发量稍微高一点的生产应用来说是不够的，你需要提前设置，申请扩大并发度。</p><h2>线上运维</h2><p>开发完代码，发布到线上后，有没有想当然的认为Serverless就是“免运维的”？虽然我们多次强调过Serverless免运维的特性，但上线后的服务稳定性、访问指标以及可观测等问题，也是需要关注的。我们着重说一下稳定性和可观测。</p><p>稳定性方面，云厂商设置的异步调用策略，可以确保失败后的请求重试，预留实例可以确保流量突增的响应及时性。</p><p>除此之外，我之前遇到的一位客户简单的方法也很值得借鉴。这位客户是一个在线的网站，对重要接口采用轮训模拟请求的方式进行探活检测，当出现连续N次出现问题的时候，就会报警和切换到备用集群、备用区域。</p><p>你会发现，在Serverless云厂商还不是很完善的情况下，作为一个开发者，可以基于“免运费”之上做一层“运维保险”。这个方式不一定最优，但确实很实用，我们交流之后也发现，对保障服务的稳定和客户的体验都有明显的效果。</p><p><strong>可观测</strong><strong>方面，</strong>除了我在<a href="https://time.geekbang.org/column/article/574555">可观测</a>小节中跟你细聊了通过指标、日志、链路三大支柱实现一个监测系统的机制，从使用的经验上，我也建议你“活”用云厂商关联的支撑服务。他们的监控仪表盘足以支撑你常用的指标，你还可以通过关联的报警服务、链路追踪服务来更全面地“问诊”服务。</p><h2>小结</h2><p>最后我们来小结一下。今天我给你介绍了Serverless实战中的一些使用经验，包括方案选型、资源评估、调用速度、开发技巧、线上运维这一个完整生命周期里的心得。</p><p>你会发现，这节课我多次提到了在第1、2模块中的知识点。比如在调用速度里，你可以关注<a href="https://time.geekbang.org/column/article/563691">冷启动</a>、<a href="https://time.geekbang.org/column/article/573740">函数之间调用</a>这两节课；在开发技巧方面，你可以着重关注<a href="https://time.geekbang.org/column/article/573015">WebIDE</a>、<a href="https://time.geekbang.org/column/article/561957">高级应用</a>这两节课，在线上运维方面，可以更多地关注可观测的内容。</p><p>这节课不一定能够解决你在实战过程中遇到的所有的问题，但一定能够作为你的知识地图，帮助你定位每一个困惑背后需要不断精进的知识点。希望能够让你在开始接下来的实战进阶课之前，串联起讲解过的技术细节，形成自己的知识网络。</p><p>随着Serverless的不断演进，未来的众多产品也一定会朝着Serverless的形态转变。你在后续的工作中，还会遇到更多的业务诉求转型、难点挑战，希望你也能一个一个找到更优的解决方案，并沉淀成方法分享给更多的Serverless前行者。</p><p>后续的实战课中，我们将一起针对性地练习Serverless技术中的5大核心场景：连接云生态、跨平台开发、传统服务的迁移、私有化部署和引擎底座选取以及基于引擎构建平台。准备好了吗？</p><h2>思考题</h2><p>好了，这节课到这里也就结束了，最后我给你留了一个思考题。</p><p>你在使用Serverless的过程中还有哪些“踩坑”经验，或者你在技术选型的时候还有哪些疑惑的地方？</p><p>欢迎在留言区写下你的思考和答案，我们一起交流讨论。</p><p>感谢你的阅读，也欢迎你把这节课分享给更多的朋友一起阅读。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c4/e4/81ee2d8f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Wisdom</span>
  </div>
  <div class="_2_QraFYR_0">能不能有老师的联系方式？邮箱或是什么的，做一些交流呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以的，极客教研助理很快会联系您哈</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-08 19:50:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c4/e4/81ee2d8f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Wisdom</span>
  </div>
  <div class="_2_QraFYR_0">我是想构建自己的serverless平台，不直接使用云厂商的serverless平台.</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以的，现在有的企业也在部署私有化，有的是自研，有的是基于开源引擎框架做二次开发，比如openFaas ，Knative 等，在22节开始可以继续关注这一方面的内容，欢迎留言讨论哦</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-08 19:48:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/57/4f/6fb51ff1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>一步</span>
  </div>
  <div class="_2_QraFYR_0">Serverless 平台是否支持 设置日记级别的？ 在开发的打印 debug ，生产的时候 不打印， 还是只能开发自己在代码中进行控制</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 平台本身目前没有可配置的地方，但自己可以设置，类似java 这种，自己在yaml等属性里设置，还是按照原本微服务的打包方式就行</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-07 09:26:32</div>
  </div>
</div>
</div>
</li>
</ul>