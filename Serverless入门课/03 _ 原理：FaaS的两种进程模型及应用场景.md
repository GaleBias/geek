<audio title="03 _ 原理：FaaS的两种进程模型及应用场景" src="https://static001.geekbang.org/resource/audio/a9/2e/a91024a738381ce20c413897d499f62e.mp3" controls="controls"></audio> 
<p>你好，我是秦粤。上一讲我们通过一个Node.js纯FaaS的Serverless应用，给你介绍了Serverless引擎盖下的运作机制，总结来说，FaaS依赖分层调度和极速冷启动的特性，在无事件时它居然可以缩容到0，就像我们的声控灯一样，有人的时候它可以亮起来，没人的时候，又可以自动关了。</p><p>听完了原理，我估计你肯定会问，FaaS这么好，但是它的应用场景是什么呢？今天我们就来一起看下。不过，想要理解FaaS的应用场景，我们就需要先理解FaaS的进程模型，这也是除了冷启动之后的另外一个重要概念。</p><h2>FaaS进程模型</h2><p>咱先回想一下上节课的FaaS的冷启动过程，我们知道容器和Runtime准备阶段都是由云服务商负责的，我们只需要关注具体的函数执行就可以了。而函数执行在FaaS里是由“函数服务”负责的，当函数触发器通知“事件”到来时，函数服务就会根据情况创建函数实例，然后执行函数。当函数执行完之后，函数实例也随之结束自己的使命，FaaS应用缩容到0，然后开始进入节能模式。</p><p>上面这套逻辑是我们上节课讲的，课后有同学就问，函数执行完之后实例能否不结束，让它继续等待下一次函数被调用呢？这样省去了每次都要冷启动的时间，响应时间不就可以更快了吗？</p><!-- [[[read_end]]] --><p>是的，本身FaaS也考虑到了这种情况，所以从运行函数实例的进程角度来看，就有两种模型。我也画了张图，方便你理解。</p><ul>
<li>用完即毁型：函数实例准备好后，执行完函数就直接结束。这是FaaS最纯正的用法。</li>
<li>常驻进程型：函数实例准备好后，执行完函数不结束，而是返回继续等待下一次函数被调用。<strong>这里需要注意，即使FaaS是常驻进程型，如果一段时间没有事件触发，函数实例还是会被云服务商销毁。</strong></li>
</ul><p><img src="https://static001.geekbang.org/resource/image/84/20/84a81773202e2599474f9c9272a65d20.png?wh=1628*832" alt="" title="模型示意图"></p><p>这两个模型其实也对应两种不同的应用场景。我举个例子，比如你要把我们第一讲中的“待办任务”应用部署上线，还记得小程同学吧，他完成了第一个版本，他用Express.js<span class="orange">[1]</span> 框架开发的MVC架构，View层他采用流行的React<span class="orange">[2]</span>，并且使用了Ant Design Pro<span class="orange">[3]</span> React组件库，Model数据库采用MongoDB。小程的第一个版本，就是一个典型的传统Web服务。</p><p>从可控性和改造成本角度来看Web服务，服务端部署方案最适合的还是托管平台PaaS或者自己搭服务跑在IaaS上。正如我上一讲所说，使用FaaS就必须在FaaS的条件限制内使用，最佳的做法应该是一开始就选用FaaS开发。</p><p>但是小程的运气比较好，我们查了一下文档，发现FaaS的Node.js的Runtime是支持Express的，所以我们只需少量修改，<strong>小程的第一个版本就可以使用FaaS的常驻进程方案部署。</strong></p><p>这里我要做个对比。在之前，假设没有FaaS，我们要将应用部署到托管平台PaaS上；启动Web服务时，主进程初始化连接MongoDB，初始化完成后，持续监听服务器的80端口，直到监听端口的句柄关闭或主进程接收到终止信号；当80端口和客户端建立完TCP链接，有HTTP请求过来，服务器就会将请求转发给Web服务的主进程，这时主进程会创建一个子进程来处理这个请求。</p><p>而在FaaS常驻进程型模式下，首先我们要改造一下代码，Node.js的Server对象采用FaaS Runtime提供的Server对象；然后我们把监听端口改为监听HTTP事件；启动Web服务时，主进程初始化连接MongoDB，初始化完成后，持续监听HTTP事件，直到被云服务商控制的父进程关闭回收。</p><p>当HTTP事件发生时，我们的Web服务主进程跟之前一样，创建一个子进程来处理这个请求事件。主进程就如我们上图中绘制的那个蓝色的圆点，当HTTP事件发生时，它创建的子进程就是蓝色弧形箭头，当子进程处理完后就会被主进程回收。</p><p>在我看来，常驻进程型就是为了传统MVC架构部署上FaaS专门设计的。数据库也可以使用原来的DB连接方式，不过这样做会增加冷启动的时间（我特意在图中用曲线代表时间增加），从而导致第一次请求长延迟甚至失败。比较适合的做法是我们<a href="https://time.geekbang.org/column/article/224559">[第 1 课]</a> 中，讲Serverless架构时说的，数据持久化采用BaaS服务。</p><p>那么我们能否用用完即毁型来部署小程的这个MVC架构的Web服务呢？可以，但是我不推荐你这样做，因为用完即毁型对传统MVC改造的成本太大。</p><p><img src="https://static001.geekbang.org/resource/image/2f/33/2f1c4643057d8fbcbec6f7514dd9cd33.png?wh=2182*880" alt="" title="模型示意图"></p><p>说到这里，我们再将上面对比两个模型的示意图镜头再拉远一点，加上HTTP触发器看看。其实从另外一个角度看，触发器就是一个常驻进程型模型一直在等待，只不过这个触发器是由云服务商处理罢了。</p><p>这里我再啰嗦强调下，还是我们上一讲说的，FaaS只是做了极端抽象，云服务商通过技术手段帮助开发者屏蔽了细节，让他们尽量只关注代码本身。</p><p>所以，在用完即毁型中，我们只要将MVC的Control层部署到函数执行就可以了。这也意味着我们要将我们的MVC架构的Control函数一个个拆解出来部署，一个HTTP请求对应一个Control函数；Control函数实例启动时连接MongoDB，一个请求处理完后直接结束。你如果要提升Control函数的冷启动时间，Model层同样要考虑BaaS化改造。这里你听着可能有点陌生，没关系，后面我会通过代码给你演示，你到时候再理解也不迟。</p><p>现在，理解了两种类型，我们再来看看FaaS是怎么收费的，以及常驻型进程这种模式是不是官方会多收费。云服务商FaaS函数服务的收费标准各不相同，但他们都会提供一定的免费额度。我给你归纳下FaaS的收费标准，主要有两个维度：调用函数次数和函数耗时。</p><ul>
<li>调用函数次数，函数每次被事件触发，计数器加一。例如我们Hello World例子的index.js文件的handler函数，它每调用一次，计数就加一。这种模式因为不占资源，所以资源利用率高、收费低。</li>
<li>函数耗时，说的是函数执行的运行时长，它的计算单位是CU-S，也就是CPU运行了多少秒。</li>
</ul><p>例如我们上面“待办任务”改造的常驻进程型和用完即毁型，多数情况下其实他们两个的函数耗时是一样的。这里可能有些绕，需要给你解释一下。</p><p>常驻进程型改造后主要占用的是内存，而FaaS收费的是CPU计算时间，也就是说常驻进程的模式并不会持续收费。但常驻型应用的冷启动时间会增加，所以我们要尽量避免冷启动，避免冷启动通常又需要做一些额外的工作，比如定时触发一下实例或者购买预留实例，这地方就会增加额外的费用了。这样听起来，是不是觉得常驻进程型改造MVC应用用起来很别扭？是的，我们前面也说了，常驻进程模式就是为了传统MVC架构部署上FaaS专门设计的，算是一种权宜之计吧。</p><p>用完即毁型改造后，同样冷启动时间会增加，但是冷启动时间是云服务商负责的。我们Control函数的执行时间，和MVC部署在FaaS中Control的执行时间是一样的。每个请求都增加了冷启动时间，响应时间会更长一些，但我们不用考虑额外的成本。那学到这儿，相信你也可以感觉到了，用完即毁型也不太适合传统MVC架构的改造，也是一种权宜之计，但这是FaaS最纯正的用法，肯定还是有它的用武之地的。</p><p>接下来，我们就继续把焦点放到用完即毁型上，来具体看看它可以用在哪些更加自然的场景里。</p><h2>数据编排</h2><p>我们做开发的多多少少都知道，目前最成功最广泛的设计模式就是MVC模式。但随着前端MVVM框架越来越火，前端View层逐渐前置，发展成SPA单页应用；后端Control和Model层逐渐下沉，发展成面向服务编程的后端应用。</p><p>这种情况下，前后端更加彻底地解耦了，前端开发可以依赖Mock数据接口完全脱离后端限制，而后端的同学则可以面向数据接口开发，但这也产生了高网络I/O的数据网关层。</p><p>Node.js的异步非阻塞和JavaScript天然亲近前端工程师的特性，自然地接过数据网关层。因此也诞生了Node.js的BFF层(Backend For Frontend)，将后端数据和后端接口编排，适配成前端需要的数据结构，提供给前端使用。</p><p>我们的程序员好朋友小程也跟进了这个潮流，将“待办任务”Web服务重构成了第二个版本。他将原先的应用拆解成了2个项目：前端项目采用React+AntDesignPro+Umi.js<span class="orange">[4]</span> 的单页应用，后端项目还是采用Express。我们本专栏的示例也采用这个技术架构一步一步教你在云上部署SPA+FaaS混合框架演进。</p><p><img src="https://static001.geekbang.org/resource/image/dd/09/dd608f746a18d6172b7057f083ad2c09.png?wh=1822*852" alt="" title="BFF示意图"></p><p>如上图所示，BFF层充当了中间胶水层的角色，粘合前后端。未经加工的数据，我们称为元数据Raw Data，对于普通用户来说元数据几乎不可读。所以我们需要将有用的数据组合起来，并且加工数据，让数据具备价值。对于数据的组合和加工，我们称之为<strong>数据编排</strong>。</p><p>BFF层通常是由善于处理高网络I/O的Node.js应用负责。传统的服务端运维Node.js应用还是比较重的，需要我们购买虚拟机，或者使用应用托管PaaS平台。</p><p>因为BFF层只是做无状态的数据编排，所以我们完全可以用FaaS用完即毁型模型替换掉BFF层的Node.js应用，也就是最近圈子里老说的那个新名词SFF（Serverless For Frontend）。</p><p>好，到这儿，我们已经理解了BFF到SFF的演进过程，现在我们再串下新的请求链路逻辑。前端的一个数据请求过来，函数触发器触发我们的函数服务；我们的函数启动后，调用后端提供的元数据接口，并将返回的元数据加工成前端需要的数据格式；我们的FaaS函数完全就可以休息了。具体如下图所示。</p><p><img src="https://static001.geekbang.org/resource/image/43/4b/43d5ae274d0169bbc0cc4aece791054b.png?wh=1844*864" alt="" title="SFF示意图"></p><p>另外，除了我们自己的后端应用数据接口，互联网上还有大量的数据供我们使用。比如疫情期间，你要爬取下各个地区的疫情数据、天气数据，这些工作，也都可以放到FaaS上轻松搞定，并且基本还能免费，因为目前各大云服务商都提供了免费的额度，这个我刚给你讲过了。</p><p>编排后端接口，编排互联网上的数据，这俩场景我想你也很容易想到。不过，我觉得，编排云服务商的各种服务才能让你真正体会到那种触电的感觉。我第一次体验之后，就对我同事说：“变天了，真的变天了，喊了这么多年的云计算时代真的来了。”</p><h2>服务编排</h2><p><strong>服务编排和数据编排很像，主要区别是对云服务商提供的各种服务进行组合和加工。</strong>在FaaS出现之前，就有服务编排的概念，但服务编排受限于服务支持的SDK语言版本，常见的情况是我们用yaml文件或命令行来编排服务。我们要使用这些服务或API，都要通过自己熟悉的编程语言去找对应的SDK，在自己的代码中加载SDK，使用秘钥调用SDK方法进行编排。就和数据编排一样，服务端运维部署成本非常高，而且如果没有SDK，则需要自己根据平台提供的接口或协议实现SDK。</p><p>现在有了FaaS，FaaS拓展了我们可以使用SDK边界，这是什么意思呢？比如小程的“待办任务”Web服务需要发送验证码邮件，我们可以用一个用完即毁型FaaS函数，调用云服务商的SDK发送邮件；再用一个常驻进程型FaaS函数生成随机字符串验证码，生成后记录这个验证码，并且调用发送邮件的FaaS将验证码发给用户邮箱；用户验证时，我们再调用常驻进程型FaaS的方法校验验证码是否正确。</p><p>我还是用阿里云来举例，我们查阅阿里云的邮件服务文档，发现它只支持Java、PHP和Python的SDK。我们一直都是在讲Node.js，这里没有Node.js的SDK，怎么办？如果我们根据阿里云邮箱服务的文档，自己开发Node.js的SDK，那肯定是饶了弯路，费了没用的力气。</p><p>因为我们发送邮件的用完即毁型FaaS函数功能很单一，所以我们完全可以参考邮件服务的PHP文档，就用PHP的SDK创建一个FaaS服务来发送邮件的。你会发现使用PHP邮件服务的成本居然如此之低。</p><p><video poster="https://media001.geekbang.org/af58590691bc48c9b99ded23575e854c/snapshots/94bc417d5b8a472080439dd788527ede-00005.jpg" preload="none" controls=""><source src="https://media001.geekbang.org/customerTrans/7264ef1853aeb04326c1b89724de902a/4926d6b2-1719cc01371-0000-0000-01d-dbacd.mp4" type="video/mp4"></video></p><p>你会看到在这个例子中，我用了我并不太熟悉PHP语言编排了邮件发送服务。不知道你意识到没有，这个也是FaaS一个亮点：语言无关性。它意味着你的团队不再局限于单一的开发语言了，你们可以利用Java、PHP、Python、Node.js各自的语言优势，混合开发出复杂的应用。</p><p>FaaS服务编排被云服务商特别关注正是因为它具备的这种开放性。使用FaaS可以创造出各种各样复杂的服务编排场景，而且还与语言无关，这大大增加了云服务商各种服务的使用场景。当然，这对开发者也提出了要求，它要求开发者去更多地了解云服务商提供的各种服务。</p><p>甚至我还知道，西雅图就有创业团队利用FaaS服务编排能力做了一套开源框架：Pulumi<span class="orange">[5]</span>，并且还拿到了融资。感兴趣的话，你可以去他们的官网看看。</p><h2>总结</h2><p>好，到这里，我们这节课的内容就讲完了。我再来总结一下这节课的关键点。</p><ol>
<li>FaaS的进程模型有两种：常驻进程型和用完即毁型。常驻进程型是为了适应传统MVC架构设计的，它看起来并不自然；如果你从现在开始玩FaaS的话，我当然首选推荐用完即毁型，它可以最大限度发挥FaaS的优势。</li>
<li>追溯历史，我给你梳理了前后端分离发展出的BFF，然后BFF又可以被SFF替代。不管是内部的接口编排，还是外部一些数据的编排，FaaS都可以发挥出极大优势，你看看我视频演示的例子就懂了。</li>
<li>从数据编排再进一步，我们可以利用FaaS和云服务商云服务的能力，做到服务编排，编排出更加强大的组合服务场景，提升我们的研发效能。并且通过我这么长时间的体验，我还想感叹说，依赖云服务商的各种能力，再通过FaaS编排开发一个项目时，往往可以做到事半功倍。</li>
</ol><h2>作业</h2><p>今天的作业和上一讲类似，我视频中给你做了个简单的Demo，你可以随便找个云平台去run一下试试，百闻不如一见，体验完之后，你可以在留言区谈谈你的感想。另外，如果今天这节课让你有所收获，也欢迎你把它分享给更多的朋友。</p><p>我Demo中的代码地址：<a href="https://github.com/pusongyang/sls-send-email">https://github.com/pusongyang/sls-send-email</a></p><h2>参考资料</h2><p>[1] Express是Node.js著名的Web服务框架&lt;<a href="https://expressjs.com/">https://expressjs.com/</a>&gt;。</p><p>[2] React 是Facebook开源的MVVM框架&lt;<a href="https://zh-hans.reactjs.org/">https://zh-hans.reactjs.org/</a>&gt;。</p><p>[3] AntDesignPro是蚂蚁开源的React组件库&lt;<a href="https://pro.ant.design/">https://pro.ant.design/</a>&gt;。</p><p>[4] Umi.js是蚂蚁开源的React企业级解决方案脚手架&lt;<a href="https://umijs.org/">https://umijs.org/</a>&gt;。</p><p>[5] Pulumi &lt;<a href="https://pulumi.io/">https://pulumi.io/</a>&gt;</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/64/05/6989dce6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我来也</span>
  </div>
  <div class="_2_QraFYR_0"># 最近几天实战了下阿里云的函数计算服务.<br>使用golang按着官方文档,实现了几个常驻进程模型的服务.<br>也照着老师的操作,建了node.js和python的函数.<br><br>相比之下,python和node.js的冷启动时长确实比较短,我这边看冷启动时接口的响应耗时在600-800ms.<br>但是之后热启动时的耗时就只有30-50ms.<br>而golang的冷启动时长不知道为什么要2.5s.而热启动后的接口耗时也才60ms.<br><br>我还发现,阿里云上,5分钟以后,常驻进程模型的函数就被干掉了.症状就是初次接口耗时又要2s+.<br>可以肯定的是,不到10分钟,无响应的函数就被系统回收了.肯定是没到15分钟.<br><br>我还发现,介于冷启动和热启动之间,还有一个状态,接口的响应耗时也是介于两者之间.<br><br># 对服务编排的个人感悟<br>感觉函数服务配合服务编排,就像是在linux上使用shell组合各种命令,实现复杂的功能.<br>虽然每个命令都很简单,但是组合后的功能就很强大了.<br>现在的云服务都会有很多现成的sdk,确实如老师所说,需要用到某个云服务时临时把官方的文档拿出来,几乎只需要做很少的变动,就可以马上投入使用.<br><br>我目前使用函数服务,配合nas和日志服务,就可以很容易的把东西存在nas上,在阿里云的日志服务中搜索相关日志.<br>不足之处就是调试没有之前方便了.<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 必须给你手动点赞，写的非常详细。云服务商承诺的回收时间，目前其实是不确定的。能够承诺时间的只有AWS（所以我的文章中没有给出具体回收时间）。通常阿里云1分钟到10分钟，没有请求就会被回收，不过1分钟是可以承诺的。因为冷启动的时间很快，所以通常影响不大。如果对冷启动增加的几百毫秒比较敏感的场景，还是建议使用CaaS服务。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-22 12:12:56</div>
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
  <div class="_2_QraFYR_0">阿里云 有个 serverless 工作流 是不是就是 一个阿里云提供的云服务的编排工具？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 手动给你点个赞，悟性很高。我后面也会介绍到，除了代码编排外，还可用事件流编排。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-22 10:11:57</div>
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
  <div class="_2_QraFYR_0">常驻型应用的冷启动时间会增加, 这里为什么会增加呢？ 不应该是减少吗？ 只需要第一次启动后，长驻内存不就行了吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 常驻进程启动后，就没有冷启动过程了。冷启动指的是函数实例的启动过程。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-22 08:13:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/49/10/af49fa20.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>左耳朵狮子</span>
  </div>
  <div class="_2_QraFYR_0">老师 有没有什么好的Serverless 论文推荐推荐 CMU 啥的都可以哈。<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 伯克利大学的这篇比较出名： https:&#47;&#47;www2.eecs.berkeley.edu&#47;Pubs&#47;TechRpts&#47;2019&#47;EECS-2019-3.pdf</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-02 08:23:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKGeekKkZzialiaB8zNQ4gVp2fNfaAfic8iaiaHBibqZDAMjaROR6sPQIjsSyGmFzLZVFibETh9ZoUhWwHfQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>InfoQ_f14f3f64ad3e</span>
  </div>
  <div class="_2_QraFYR_0">老师你好！函数服务一般是用在无状态场景下，但是看到发邮件的那个例子，存储验证码的常驻服务看起来是个有状态的，是常驻进程类型的可以处理有状态服务吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以的，常驻进程类型，之前课程讨论区，我们说的，常驻进程型，如果没有流量1分钟会被销毁。所以如果你的验证码，有限期是1分钟以内，就可以用。当然也可以将数据用BaaS存储起来的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-27 01:45:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/65/41/820e14af.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Marooned。</span>
  </div>
  <div class="_2_QraFYR_0">因为用完即毁型对传统 MVC 改造的成本太大 为什么会大呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-22 14:51:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_d972f2</span>
  </div>
  <div class="_2_QraFYR_0">&quot;我们的 Web 服务主进程跟之前一样，创建一个子进程来处理这个请求事件&quot;这里没明白，Web服务主进程是指Node进程吗？如果是，Node不是采用事件循环机制来处理请求吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: javascript引擎的实现是一个进程拥有一个自己的事件循环。Node.js中可以通过child proceses创建子进程。这部分可以看看Node.js文档的cluster章节</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-07 00:50:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/10/79/390568f3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>kkkkkkkk</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，我node基础薄弱的前端开发，这一节看得有点吃力，需要补充哪些知识可以更能理解老师的优质内容</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 建议可以补习一下，朴灵老师的《Nodejs深入浅出》</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-24 23:11:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/b0/78/b4a8d4d1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>贫困山区杨先生</span>
  </div>
  <div class="_2_QraFYR_0">请问一下老师：按照上面的做，也设置了SDK的access key和secret key<br>Response<br>Module &#39;&#47;code&#47;index.php&#39; is missing.</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 需要跟多的信息，你可以加一下课程微信群，在群里聊。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-27 14:27:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/47/00/3202bdf0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>piboye</span>
  </div>
  <div class="_2_QraFYR_0">php 最初的模型就是一个请求创建一个进程。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，后面php也支持fastcgi，各有利弊。具体还是看应用场景。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-08 21:20:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/f5/05/09aaa06c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>yiwu</span>
  </div>
  <div class="_2_QraFYR_0">老师，我在想问FaaS适合做VPN吗？好像也没有人操作过这个，应该是有什么难题需要解决。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不适合，VPN需要客户端管道。FaaS的触发器目前不支持。<br>你可以考虑用CaaS，docker容器来做。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-14 10:19:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q3auHgzwzM6hsCtODfwaIPW9T9qzxNAhhkdn4ImGHeZicA1UyhCOXDf8MtJXw4QnTFQgUia4BPTZdD2zpgV1qTfQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_f7f72f</span>
  </div>
  <div class="_2_QraFYR_0">似乎传统的OOP和设计模式在FaaS领域应用不大?  感觉缺少class，适用场景确实有限制</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: OOP和设计模式，对于FaaS来说是一样的。<br>第十一课，TS就是用class的方式写的。 </p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-14 08:00:36</div>
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
  <div class="_2_QraFYR_0">Serverless要想发展的好，一定要有一套标准的应用迁移方案，这样才能保证现有的大量应用可以平滑的过渡。<br>目前来看，我个人的理解，对于现有应用，可能常驻进程方式合适一些，对于新应用，用后即销毁的方式更合适。这里搞了一刀切，可能不太对哈。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你可以换个角度思考一下，虽然2种模型收费是一样的。但是对于云服务商来说，用完即毁型，显然对碎片化的机器资源利用率更高。<br>常驻进程型，云服务商如果自己资源比较紧张时，可能会采用比较激进的销毁实例做法。<br>我的第11课会讲回这部分，目前的FaaS做Serverless应用，这2种做法都有。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-10 07:59:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>suke</span>
  </div>
  <div class="_2_QraFYR_0">用完即毁型的服务里如果有mysql的连接查询，建立连接岂不是很慢？而且如果查询不频繁，那岂不是每次都查询要等好久？老师有这方面的实践么</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个是后面课的内容，将mysql变成RESTFul HTTP。不过更好的做法是，云服务商提供mysql BaaS服务。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-29 19:11:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/81/d5/00efc5b7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Charlie</span>
  </div>
  <div class="_2_QraFYR_0">你好老师，我的没收到邮件，显示如下，请问是什么问题？？怎么解决？<br>Response<br><br>email send!<br><br>Function Logs<br><br>FC Invoke Start RequestId: 2607ad8c-02c9-41b0-b062-16f2ac5507c5<br><br>InvalidAccessKeyId.NotFoundSpecified access key is not found.FC Invoke End RequestId: 2607ad8c-02c9-41b0-b062-16f2ac5507c5<br><br><br>Duration: 116.70 ms, Billed Duration: 200 ms, Memory Size: 512 MB, Max Memory Used: 9.97 MB<br><br>Request ID<br>2607ad8c-02c9-41b0-b062-16f2ac5507c5</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 同学你的access key没有找到。这个需要开通一下access key和secret key。然后将这2个值填写到代码里面。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-26 17:40:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/71/ed/45ab9f03.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>八哥</span>
  </div>
  <div class="_2_QraFYR_0">服务编排应该后面会讲到serverless framework。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-24 17:54:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/6c/69/d5a28079.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Bora.Don</span>
  </div>
  <div class="_2_QraFYR_0">对老师关于数据编排的看法，存在一点疑问：数据不在本地，会不会存在稳定性和速度的问题？<br>FaaS感觉看上去很美，实际操作空间还有待考验，个人观点。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: FaaS目前的确有使用场景限制，不是银弹。<br>数据不在本地确实会有稳定性和速度问题，所以也要准备容灾和缓存策略，避免依赖的数据不可用的场景。<br>SOA的一个重要思想就是：面向失败编程。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-24 09:21:20</div>
  </div>
</div>
</div>
</li>
</ul>