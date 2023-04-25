<audio title="01｜定义：到底什么是Serverless？" src="https://static001.geekbang.org/resource/audio/72/56/72049d40d625f17c5bffdab43dc24356.mp3" controls="controls"></audio> 
<p>你好，我是秦粤。Serverless目前是大热的话题，相信你肯定听过，但如果你去百度、Google或者维基百科上查的话，你会发现它连个准确的定义都没有。</p><p>作为本专栏的第一讲，今天我就想带你深入地了解下Serverless，看看它都能解决哪些问题，以及为什么难定义。</p><h2>Serverless能解决什么问题？</h2><p>理清Serverless要解决的问题其实很简单，我们可以从字面上把它拆开来看。</p><p>Server这里指服务端，它是Serverless解决问题的边界；而less我们可以理解为较少关心，它是Serverless解决问题的目的。组合在一起就是“较少关心服务端”。怎么理解这句话呢？我们依然是拆开来分析。</p><h3>什么是服务端？</h3><p>我们先看Server，这里我用Web应用经典的MVC架构来举例。</p><p>现代研发体系主要分为前端和后端，前端负责客户终端的体验，也就是View层；后端负责商业的业务逻辑和数据处理，也就是Control层和Model层。如果你有过一些开发经验，应该会了解自己的代码在本地开发和调试时的数据流。</p><p><img src="https://static001.geekbang.org/resource/image/3d/b3/3db954564207537706321daa50b870b3.png?wh=2134*534" alt="" title="MVC架构的Web应用"></p><p>通常我们会在自己电脑上启动一个端口号，例如127.0.0.1:3001。浏览器访问这个地址，就可以调用和调试自己的代码，但如果我们要将这个Web应用部署到互联网上，提供给互联网用户访问，就需要服务端的运维知识了。</p><!-- [[[read_end]]] --><p><img src="https://static001.geekbang.org/resource/image/41/71/4174f5f6485cb1e942782f2ad003d971.png?wh=2174*1042" alt="" title="MVC架构的Web应用"></p><p>通常我部署运维一个应用时，由于要考虑容灾和容错，我都会保障异地多活，因此我会部署多个Web应用实例。每个Web应用实例跟我们在本地开发时是一样的，只是IP改为了私有网络IP。</p><p>随着云服务商的兴起，现在已经很少有互联网企业还在自己维护物理机了。在云服务的运维体系中，各个环节都已经有了对应的成熟的云服务产品或解决方案。</p><p>为了使多个Web应用实例在容灾和容错的场景下稳定地切换流量，我就需要负载均衡服务和反向代理服务。负载均衡服务，正如其名是负责将流量均衡地分配到各个应用机器上；反向代理，常见的就是Nginx，它的任务是从请求中解析出域名信息，并将请求转发到上游upstream的监听地址。</p><p>服务端的边界，就是上图中的整个蓝色部分，它是负责应用或代码的线上运维。Serverless解决问题的边界，就是服务端的边界，即服务端运维。</p><h3>服务端运维发展史，从full到less</h3><p>了解完Server，我们再来看less。</p><p>我们可以先看Serverfull的概念，对比Serverfull和Serverless之间的差别，相信这样可以加深你的理解。Serverfull就是服务端运维全由我们自己负责，Serverless则是服务端运维较少由我们自己负责，大多数的运维工作交给自动化工具负责。</p><p>可能这么说比较抽象，我举个例子来给你讲吧。这个例子有点长，但可以带你很好地了解下服务端运维的发展史，我们后面也会再次用到。</p><p>假设我有一家互联网公司，我的产品是“待办任务（ToDoList）”Web应用——记录管理每个用户的待办任务列表。针对这个网站的研发，我们简化为两个独立角色：研发工程师小程和运维工程师小服。</p><p>做研发的小程，他是个精通前后端的全栈工程师，但他只关心应用的业务逻辑。具体来说就是，整个MVC架构Web应用的开发都归小程负责，也就是从服务端界面View层，到业务逻辑Control层，再到数据存储Model层，整个Web应用的版本管理和线上bug修复都归小程。</p><p>负责运维的小服，则只关心应用的服务端运维事务。他负责部署上线小程的Web应用，绑定域名以及日志监控。在用户访问量大的时候，他要给这个应用扩容；在用户访问量小的时候，他要给这个应用缩容；在服务器挂了的时候，他还要重启或者换一台服务器。</p><p><strong>史前时代，Serverfull</strong></p><p>最开始运维工程师小服承诺将运维的事情全包了，小程不用关心任何部署运维相关的事情。小程每次发布新的应用，都会打电话给小服，让小服部署上线最新的代码。小服要管理好迭代版本的发布，分支合并，将应用上线，遇到问题回滚。如果线上出了故障，还要抓取线上的日志发给小程解决。</p><p>小程和小服通过工作职责任务上的安排，将研发和运维完全隔离开来了。好处很明显：分工明确小程可以专心做好自己的业务；缺陷也很明显：小服成了工具人，被困在了大量的运维工作中，处理各种发布相关琐碎的杂事。</p><p>这个时代研发和运维隔离，服务端运维都交给小服一个人，纯人力处理，也就是Serverfull。</p><p>我们可以停下来想想，像发布版本和处理线上故障这种工作这些是小程的职责，都要小服协助，是不是应该小程自己处理？</p><p><strong>农耕时代，DevOps</strong></p><p>后来，小服渐渐发现日常其实有很多事情都是重复性的工作，尤其是发布新版本的时候，与其每次都等小程电话，线上出故障了还要自己抓日志发过去，效率很低，不如干脆自己做一套运维控制台OpsConsole，将部署上线和日志抓取的工作让小程自己处理。</p><p>OpsConsole上线后，小服稍微轻松了一些，但是优化架构节省资源和扩缩容资源方案，还是需要小服定期审查。而小程除了开发的任务，每次发布新版本或解决线上故障，都要自己到OpsConsole平台上去处理。</p><p>这个时代就是研发兼运维DevOps，小程兼任了小服的部分工作，小服将部分服务端运维的工作工具化了，自己可以去做更加专业的事情。相对史前时代，看起来是不是小程负责的事情多（More）了？但实际这些事情本身就应该是小程负责的，版本控制、线上故障都是小程自己应该处理的，而小服已经将这部分人力的工作工具化了，更加高效，其实已经有变少（less）的趋势了。</p><p>我们再想想能否进一步提升效率，让小程连OpsConsole平台都可以不用？</p><p><strong>工业时代</strong></p><p>这时，小服发现资源优化和扩缩容方案也可以利用性能监控+流量估算解决。小服又基于小程的开发流程，OpsConsole系统再进一步，帮小程做了一套代码自动化发布的流水线：代码扫描-测试-灰度验证-上线。现在的小程连OpsConsole都不用登陆操作，只要将最新的代码合并到Git仓库指定的develop分支，剩下的就都由流水线自动化处理发布上线了。</p><p>这个时代研发不需要运维了，免运维NoOps。小服的服务端运维工作全部自动化了，小程也变回到最初，只需要关心自己的应用业务就可以了。我们不难看出，在服务端运维的发展历史中，对于小程来说，小服的角色存在感越来越弱，需要小服参与的事情越来越少，都由自动化工具替代了。这就是“Serverless”。</p><p>到这里你一定会想，既然服务端都做到免运维了，小服是不是就自己革了自己的命，失业了？</p><p><strong>未来</strong></p><p>实现了免运维NoOps，并不意味着小服要失业了，而是小服要转型，转型去做更底层的服务，做基础架构的建设，提供更加智能、更加节省资源、更加周到的服务。小程则可以完全不被运维的事情困扰，放心大胆地依赖Serverless服务，专注做好自己的业务，提升用户体验，思考业务价值。</p><p>免运维NoOps并不是说服务端运维就不存在了，而是通过全知全能的服务，覆盖研发部署需要的所有需求，让研发同学小程对它的感知越来越少。另外，NoOps是理想状态，因为我们只能无限逼近NoOps，所以这个单词是less，不可能是Server<strong>Least</strong>或者Server<strong>Zero</strong>。</p><p>另外你需要知道的是，目前大多数互联网公司，包括一线互联网公司，都还在DevOps时代。但Serverless的概念已经提出，NoOps的时代正在到来。</p><p>Server限定了Serverless解决问题的边界，即服务端运维；less说明了Serverless解决问题的目的，即免运维NoOps。所以我们重新组合一下这个词的话，Serverless就应该叫做服务端免运维。这也就是Serverless要解决的问题。</p><h2>Serverless为什么难准确定义？</h2><p>换句话说，服务端免运维，要解决的就是将小服的工作彻底透明化；研发同学只关心业务逻辑，不用关心部署运维和线上的各种问题。要实现这个终态，就意味着需要对整个互联网服务端的运维工作进行极端抽象。</p><p>那越抽象的东西，其实越难定义，因为蕴含的信息量太大了，这就是Serverless很难准确定义的根本原因。Serverless对Web服务开发的革命之一，就是极度简化了服务端运维模型，使一个零经验的新手，也能快速搭建一套低成本、高性能的服务端应用（下一讲，我就会带你体验一下从零开始的Serverless应用）。</p><p>我们现在虽然知道了Serverless的终态是NoOps，但它作为一门新兴的技术，我们还是要尝试给它定义吧？</p><h2>到底什么是Serverless?</h2><p>我在日常和其他同事沟通的时候，我发现大家对Serverless概念的认知很模糊，往往要根据沟通时的上下内容去看到底此时的Serverless在指代什么。主要是因为Serverless这个词包含的信息量太大，而且适用性很广，但总结来说Serverless的含义有这样两种。</p><p>第一种：狭义Serverless（最常见）= Serverless computing架构 = FaaS架构 = Trigger（事件驱动）+ FaaS（函数即服务）+ BaaS（后端即服务，持久化或第三方服务）= FaaS + BaaS</p><p>第二种：广义Serverless = 服务端免运维 = 具备Serverless特性的云服务</p><p><img src="https://static001.geekbang.org/resource/image/ff/28/ff0f26ec67f6455dfc142c2943da0028.png?wh=1840*1012" alt="" title="Serverless定义"></p><p>我用图片来阐明一下这两种概念。其实你不难看出，广义Serverless包含的东西更多，适用范围更广，但我们经常在工作中提到的Serverless一般都是指狭义的Serverless。其实这是历史原因，2014 年11月份，亚马逊推出真正意义上的第一款Serverless FaaS服务：Lambda。Serverless的概念才进入了大多数人的视野，也因此Serverless曾经一度就等于FaaS。</p><p>我们再聚焦到图上“狭义Serverless”这里。你注意看的话，图中有几个陌生的名词，我先来给你解释下。FaaS(Function as a Service)就是函数即服务，BaaS(Backend as a Service)就是后端即服务。XaaS(X as a Service)就是X即服务，这是云服务商喜欢使用的一种命名方式，比如我们熟悉的SaaS、PaaS、IaaS都是这样。</p><p>先说FaaS，函数即服务，它还有个名字叫作Serverless Computing，它可以让我们随时随地创建、使用、销毁一个函数。</p><p>你可以想一下通常函数的使用过程：它需要先从代码加载到内存，也就是实例化，然后被其它函数调用时执行。在FaaS中也是一样的，函数需要实例化，然后被触发器Trigger或者被其他的函数调用。二者最大的区别就是在Runtime，也就是函数的上下文，函数执行时的语境。</p><p>FaaS的Runtime是预先设置好的，Runtime里面加载的函数和资源都是云服务商提供的，我们可以使用却无法控制。你可以理解为FaaS的Runtime是临时的，函数调用完后，这个临时Runtime和函数一起销毁。</p><p>FaaS的函数调用完后，云服务商会销毁实例，回收资源，所以FaaS推荐无状态的函数。如果你是一位前端工程师的话，可能很好理解，就是函数不可改变Immutable。简单解释一下，就是说一个函数只要参数固定，返回的结果也必须是固定的。</p><p>用我们上面的MVC架构的Web应用举例，View层是客户端展现的内容，通常并不需要函数算力；Control层，就是函数的典型使用场景。MVC架构里面，一个HTTP的数据请求，就会对应一个Control函数，我们完全可以用FaaS函数来代替Control函数。在HTTP的数据请求量大的时候，FaaS函数会自动扩容多实例同时运行；在HTTP的数据请求量小时，又会自动缩容；当没有HTTP数据请求时，还会缩容到0实例，节省开支。</p><p><img src="https://static001.geekbang.org/resource/image/41/a6/415a8dc0dc026fe2db71e2406e568ba6.png?wh=2154*994" alt="" title="MVC架构的Web应用"></p><p>此刻或许你会有点疑惑，Runtime不可控，FaaS函数无状态，函数的实例又不停地扩容缩容，那我需要持久化存储一些数据怎么办，MVC里面的Model层怎么解决？</p><p>此时我就要介绍另一位嘉宾，BaaS了。</p><p>BaaS其实是一个集合，是指具备高可用性和弹性，而且免运维的后端服务。说简单点，就是专门支撑FaaS的服务。FaaS就像高铁的车头，如果我们的后端服务还是老旧的绿皮火车车厢，那肯定是要散架的，而BaaS就是专门为FaaS准备的高铁车厢。</p><p>MVC架构中的Model层，就需要我们用BaaS来解决。Model层我们以MySQL为例，后端服务最好是将FaaS操作的数据库的命令，封装成HTTP的OpenAPI，提供给FaaS调用，自己控制这个API的请求频率以及限流降级。这个后端服务本身则可以通过连接池、MySQL集群等方式去优化。各大云服务商自身也在改造自己的后端服务，BaaS这个集合也在日渐壮大。</p><p><img src="https://static001.geekbang.org/resource/image/f6/66/f6fa8a879b34885d8037265bedbd2366.png?wh=2162*1018" alt="" title="MVC架构的Model层"></p><p>基于Serverless架构，我们完全可以把传统的MVC架构转换为BaaS+View+FaaS的组合，重构或实现。</p><p>这样看下来的话，狭义Serverless的含义也就不难理解了。</p><blockquote>
<p>第一种：狭义Serverless（最常见）= Serverless computing架构 = FaaS架构 = Trigger（事件驱动）+ FaaS（函数即服务）+ BaaS（后端即服务，持久化或第三方服务）= FaaS + BaaS</p>
</blockquote><p>Serverless毋庸置疑正是因为FaaS架构才流行起来，进入大家认知的。所以我们最常见的Serverless都是指Serverless Computing架构，也就是由Trigger、FaaS和BaaS架构组成的应用。这也是我给出的狭义Serverless的定义。</p><p><strong>那什么是广义Serverless呢？</strong></p><p>将狭义的Serverless推升至广义，具备以下特性的，就是Serverless服务。你可以回忆一下小服的工作，要达成NoOps，都应该具备什么条件？</p><p>小服的工作职责：</p><ol>
<li>无需用户关心服务端的事情（容错、容灾、安全验证、自动扩缩容、日志调试等等)。</li>
<li>按使用量（调用次数、时长等）付费，低费用和高性能并行，大多数场景下节省开支。</li>
<li>快速迭代&amp;试错能力（多版本控制，灰度，CI&amp;CD等等）。</li>
</ol><p>广义Serverless，其实就是指服务端免运维，也是未来的主要趋势。</p><p>总结来说的话就是，我们日常谈Serverless的时候，基本都是指狭义的Serverless，但当我们提到某个服务Serverless化的时候，往往都是指广义的Serverless。我们后面的课程中也是如此。</p><h2>总结</h2><p>今天我们一起学习了Serverless的边界、目标以及定义。我还介绍了即将贯穿我们整个专栏的Web应用，即任务列表。</p><p>现在我们可以回过头来看看最初的两个问题了，你是不是已经有答案了呢？</p><ol>
<li>Serverless能解决什么问题？Serverless可以使应用在服务端免运维。</li>
<li>Serverless为什么难定义？Serverless将服务端运维高度抽象成了一种解决方案，包含的信息量太大了。</li>
</ol><p>另外，Serverless可分为狭义和广义。狭义Serverless是指用FaaS+BaaS这种Serverless架构开发的应用；广义Serverless则是具备Serverless特性的云服务。现在的云服务商，正在积极地将自己提供的各种云服务Serverless化。</p><h2>作业</h2><p>最后，留给你一个作业：请你注册一个云服务商的云账号，并根据云服务商的文档和教程，自己部署一个Serverless应用。下节课，我将用云上的FaaS部署一个Serverless应用，给你详细讲解Serverless引擎盖内的知识。如果今天这节课让你有所收获，也欢迎你把它分享给更多的朋友。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/52/40/e57a736e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>pedro</span>
  </div>
  <div class="_2_QraFYR_0">这里不得不提微信小程序的云开发其实就是一种意义上的 serverless，让前端工程师不仅可以开发页面还可以通过云函数(Faas)来写业务，而且还提供了基础存储(Baas)。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，微信小程序云开发，也是一种serverless的应用场景。Serverless发展的一个方向，也在追求这种一体化的开发体验。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-15 18:38:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/29/1a/87f11f3d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>JackPn</span>
  </div>
  <div class="_2_QraFYR_0">这个有点牛皮，感觉这样一来程序员的技术活只剩下写逻辑了，其他的都是管理</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Serverless还在发展阶段，体验也还在完善中，但肯定是未来值得关注的内容。未来技术门槛只会越来越低。程序员和其他的工作一样，应该是去拼想象力，并不是除了技术就是管理。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-15 19:08:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/c9/68/1cf4dcf7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>大表哥</span>
  </div>
  <div class="_2_QraFYR_0">听了这节课感觉这钱没白花，妥妥的实力派。然后一开始吸引我的是省钱哈哈哈（我是说中长尾服务模块）。作为一个12年后台开发、架构师也来拥抱一下serverless。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Serverless对于中长尾应用的场景，的确比较适合。在没有流量的时候缩容为0，节省流量。可以节省不少开支。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-28 16:32:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/84/37/9c4dd979.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>24601</span>
  </div>
  <div class="_2_QraFYR_0">我在力拓上写函数，感觉也算是 severless🤔</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你说的是leetcode吗？leetcode上面刷题，执行一个函数并不算是serverless哦，它只是在沙箱里执行一个函数返回结果而已。serverless解决的是服务端运维的事情。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-18 08:18:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/65/c1/afcd981b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>程序员二师兄</span>
  </div>
  <div class="_2_QraFYR_0">我的作业：http:&#47;&#47;my-bucket-1253451803.cos-website.ap-guangzhou.myqcloud.com&#47;</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不错呀~，学习Serverless最好的方式就是实践~ 。手动点赞！我的下节课会讲FaaS的原理，欢迎学习。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-16 18:00:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/d0/ee/f5c5e191.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>LYy</span>
  </div>
  <div class="_2_QraFYR_0">serverless实力解决的是运维问题<br>运维的职责:<br>1 高可用&amp;安全&amp;故障追踪: 容错 容灾 安全 扩缩容 日志调试<br>2 低成本: 按量购买 及时调整<br>3 高质量自动化上线: CI&amp;CD 灰度发布 多版本控制<br>serverless旨在将服务端除业务之外的工作解耦出去 由对应的云服务承载</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-13 14:38:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/67/59/017b5726.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>猫切切切切切</span>
  </div>
  <div class="_2_QraFYR_0">读后感：<br><br>1. 是什么：severless是一套运维框架，它封装了基础通用的服务端架构所需能力，通常由云服务商提供，旨在解决传统运维工作繁琐重复又不够自动化的问题，及其导致的人力和机器成本上升和不利于公司与个人发展的问题，并最终无限接近服务端免运维的目标。（理想状态）<br><br>2. 怎么用：对于新项目，需要基于severless框架开发，对于已有项目，需要对其进行severless化改造。<br><br>3. 优缺点：<br>优点是可降低服务运维成本<br>缺点是对云服务商形成依赖</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 总结的很棒👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-13 22:01:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/b9/8e/a6eb7923.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>黄耗子皮</span>
  </div>
  <div class="_2_QraFYR_0">请问图是用什么工具画的。。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Balsqmiq Mockups,我之前用免费的,可惜现在要收费了.</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-12 11:33:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/7b/f1/da02bebb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>闹够了</span>
  </div>
  <div class="_2_QraFYR_0">因为之前已经看过不少这种了。对serverless也比较感兴趣。但是之前写的demo都是类似于hello world。不知道怎么连接数据库。哈哈哈。现在大概知道了。然后的话他是通过算法还是什么达到的扩容和收缩？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 要做到扩缩容，FaaS比较简单通过函数里面配置单例并发就可以了。但具体扩缩容的原理，后面课程会讲到。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-24 08:39:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/f5/c5/9b06bdb5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>神执念の浅言多行</span>
  </div>
  <div class="_2_QraFYR_0">老师，请问完成这个课程的作业，是需要拥有一个域名才可以的吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个课程不用，不过如果有计划，最好注册一个域名。后续的课程也会需要的。没有域名，FaaS部署的HTTP服务只能下载，不能用浏览器访问。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-21 09:55:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/65/c1/afcd981b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>程序员二师兄</span>
  </div>
  <div class="_2_QraFYR_0">Serverless 让服务端免运维，这一点很赞，发布应用不需要运维了。<br>老师留的作业准备今天中午实践一下，实践完再来留言。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，Serverless需要大家自己多体验一下，才能有切身感受。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-16 08:08:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/54/9a/76c0af70.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>每天晒白牙</span>
  </div>
  <div class="_2_QraFYR_0">这样搞下去，程序员门槛越来越低，如果不提高自己的核心竞争力，真的要失业了😂</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不用担心，程序员跟其他职业一样只不过会往更专业，更精细化的分工去走。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-16 06:47:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/00/4e/be2b206b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>吴小智</span>
  </div>
  <div class="_2_QraFYR_0">老师的思路还是很赞，对 Serverless  有了比较清晰饿认识。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 多谢你的支持！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-15 23:23:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/fc/c7/c0daa0cf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>David.</span>
  </div>
  <div class="_2_QraFYR_0">FaaS 在aws上可以通过API Gateway+lambda实现，请问老师BaaS具体可以通过什么方式实现呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 同学你能这样问说明你在认真学习和思考，赞一个！我专栏后续的课程会将，怎么将后端的应用BaaS化。当然现在云服务商也提供了很多BaaS的能力。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-15 23:02:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_ef1d01</span>
  </div>
  <div class="_2_QraFYR_0">老师好，我是serverless小白，我在阿里云上搜索serverless，都是小程序serverless；请问serverless的安装教程在哪里呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 从提问来看，你对serverless的理解还比较粗浅。建议先多学习一下serverless的相关课程。serverless是后端的服务，不存在安装教程。小程序serverless，是指为小程序服务的serverless服务。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-20 15:46:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/b7/4e/d71e8d2f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Adoy</span>
  </div>
  <div class="_2_QraFYR_0">老师，我想说一下一个小细节，-less这个后缀其实就是“没有，无法，without”的意思。这个可能会误导同学们理解这个词的原意。说得有点晚了，但是我觉得这个挺重要的，所以说出来供大家参考一下。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 英文less后缀并不等于完全没有。但是中文翻译可以用无，例如fearless，thoughtless，你可以意会一下。很多人将serverless直接翻译成“无服务器”，这里主要是为了区分。因为这些词是国外造的，serverless的反义词是serverfull。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-03 22:18:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/16/5b/83a35681.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Monday</span>
  </div>
  <div class="_2_QraFYR_0">FaaS典型的应用场景，无状态化，少依赖，独立的。<br>比如：图片处理（如水印），视频处理（如转码），日志处理等</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-24 23:08:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/16/5b/83a35681.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Monday</span>
  </div>
  <div class="_2_QraFYR_0">老师的例子虽长，但是很接地气。确认是现在大多数国内公司都是处于DevOps阶段。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-24 22:41:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/d7/a9/3f882fbf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lcssptz</span>
  </div>
  <div class="_2_QraFYR_0">谢谢老师，讲得很清晰很透彻。之前一直有听说过serverless，但一直没去了解。我现在先去用用Faas和Baas体验一下。如果不错，就把一些应用转移到serverless去。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 学习serverless最佳体验也是多多动手体验。目前BaaS体系不太完善，可能需要先看看后面章节的知识，建立自己的BaaS服务。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-26 20:59:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/e0/3b/889b6480.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>daqiuqu</span>
  </div>
  <div class="_2_QraFYR_0">请教下老师，这节课提到 FaaS 适合无状态，那涉及有状态和无状态混用的适合吗，比如应用中有 http api 也有 socket 的即时通信</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 后续的课程会讲到，状态适合放在BaaS中处理。目前的FaaS适合做无状态的，快速伸缩的应用。例如数据编排，数据接口层。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-05 00:14:26</div>
  </div>
</div>
</div>
</li>
</ul>