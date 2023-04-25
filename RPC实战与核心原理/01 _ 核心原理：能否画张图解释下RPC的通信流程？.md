<audio title="01 _ 核心原理：能否画张图解释下RPC的通信流程？" src="https://static001.geekbang.org/resource/audio/0b/48/0bf4282806173a0339aea119b9822f48.mp3" controls="controls"></audio> 
<p>你好，我是何小锋。只要你做过几年开发，那我相信RPC这个词你肯定是不陌生了。写专栏之前，我还特意查了下RPC的百度指数，发现这些年RPC的搜索趋势都是稳步上升的，这也侧面说明了这项技术正在逐步渗透到我们的日常开发中。作为专栏的第一讲，我想只围绕“RPC”这个词，和你聊聊它的定义，它要解决的问题，以及工作原理。</p><p>在前些年，我面试工程师的时候，最喜欢问候选人一个问题，“你能否给我解释下RPC的通信流程”。这问题其实并不难，不过因为很多工程师平时都在用各种框架，他们可能并未停下来思考过框架的原理，所以，问完这问题，有的人就犹豫了，吱唔了半天也没说出所以然来。</p><p>紧接着，我会引导他说，“你想想，如果没有RPC框架，那你要怎么调用另外一台服务器上的接口呢”。你看，这问题可深可浅，也特别考验候选人的基本功。如果你是候选人，你会怎么回答呢？今天我就来试着回答你这个问题。</p><h2>什么是RPC？</h2><p>我知道你肯定不喜欢听概念，我也是这样，看书的时候一看到概念就直接略过。不过，到后来，我才发现，“定义”是一件多么伟大的事情。当我们能够用一句话把一个东西给定义出来的时候，侧面也说明你已经彻底理解这事了，不仅知道它要解决什么问题，还要知道它的边界。所以，你可以先停下来想想，什么是RPC。</p><!-- [[[read_end]]] --><p>RPC的全称是Remote Procedure Call，即远程过程调用。简单解读字面上的意思，远程肯定是指要跨机器而非本机，所以需要用到网络编程才能实现，但是不是只要通过网络通信访问到另一台机器的应用程序，就可以称之为RPC调用了？显然并不够。</p><p>我理解的RPC是帮助我们屏蔽网络编程细节，实现调用远程方法就跟调用本地（同一个项目中的方法）一样的体验，我们不需要因为这个方法是远程调用就需要编写很多与业务无关的代码。</p><p>这就好比建在小河上的桥一样连接着河的两岸，如果没有小桥，我们需要通过划船、绕道等其他方式才能到达对面，但是有了小桥之后，我们就能像在路面上一样行走到达对面，并且跟在路面上行走的体验没有区别。所以<strong>我认为，RPC的作用就是体现在这样两个方面：</strong></p><ul>
<li>屏蔽远程调用跟本地调用的区别，让我们感觉就是调用项目内的方法；</li>
<li>隐藏底层网络通信的复杂性，让我们更专注于业务逻辑。</li>
</ul><h2>RPC通信流程</h2><p>理解了什么是RPC，接下来我们讲下RPC框架的通信流程，方便我们进一步理解RPC。</p><p>如前面所讲，RPC能帮助我们的应用透明地完成远程调用，发起调用请求的那一方叫做调用方，被调用的一方叫做服务提供方。为了实现这个目标，我们就需要在RPC框架里面对整个通信细节进行封装，<strong>那一个完整的RPC会涉及到哪些步骤呢？</strong></p><p>我们已经知道RPC是一个远程调用，那肯定就需要通过网络来传输数据，并且RPC常用于业务系统之间的数据交互，需要保证其可靠性，所以RPC一般默认采用TCP来传输。我们常用的HTTP协议也是建立在TCP之上的。</p><p>网络传输的数据必须是二进制数据，但调用方请求的出入参数都是对象。对象是肯定没法直接在网络中传输的，需要提前把它转成可传输的二进制，并且要求转换算法是可逆的，这个过程我们一般叫做“序列化”。</p><p>调用方持续地把请求参数序列化成二进制后，经过TCP传输给了服务提供方。服务提供方从TCP通道里面收到二进制数据，那如何知道一个请求的数据到哪里结束，是一个什么类型的请求呢？</p><p>在这里我们可以想想高速公路，它上面有很多出口，为了让司机清楚地知道从哪里出去，管理部门会在路上建立很多指示牌，并在指示牌上标明下一个出口是哪里、还有多远。那回到数据包识别这个场景，我们是不是也可以建立一些“指示牌”，并在上面标明数据包的类型和长度，这样就可以正确的解析数据了。确实可以，并且我们把数据格式的约定内容叫做“协议”。大多数的协议会分成两部分，分别是数据头和消息体。数据头一般用于身份识别，包括协议标识、数据大小、请求类型、序列化类型等信息；消息体主要是请求的业务参数信息和扩展属性等。</p><p>根据协议格式，服务提供方就可以正确地从二进制数据中分割出不同的请求来，同时根据请求类型和序列化类型，把二进制的消息体逆向还原成请求对象。这个过程叫作“反序列化”。</p><p>服务提供方再根据反序列化出来的请求对象找到对应的实现类，完成真正的方法调用，然后把执行结果序列化后，回写到对应的TCP通道里面。调用方获取到应答的数据包后，再反序列化成应答对象，这样调用方就完成了一次RPC调用。</p><p><strong>那上述几个流程就组成了一个完整的RPC吗？</strong></p><p>在我看来，还缺点东西。因为对于研发人员来说，这样做要掌握太多的RPC底层细节，需要手动写代码去构造请求、调用序列化，并进行网络调用，整个API非常不友好。</p><p>那我们有什么办法来简化API，屏蔽掉RPC细节，让使用方只需要关注业务接口，像调用本地一样来调用远程呢？</p><p>如果你了解Spring，一定对其AOP技术很佩服，其核心是采用动态代理的技术，通过字节码增强对方法进行拦截增强，以便于增加需要的额外处理逻辑。其实这个技术也可以应用到RPC场景来解决我们刚才面临的问题。</p><p>由服务提供者给出业务接口声明，在调用方的程序里面，RPC框架根据调用的服务接口提前生成动态代理实现类，并通过依赖注入等技术注入到声明了该接口的相关业务逻辑里面。该代理实现类会拦截所有的方法调用，在提供的方法处理逻辑里面完成一整套的远程调用，并把远程调用结果返回给调用方，这样调用方在调用远程方法的时候就获得了像调用本地接口一样的体验。</p><p>到这里，一个简单版本的RPC框架就实现了。我把整个流程都画出来了，供你参考：</p><p><img src="https://static001.geekbang.org/resource/image/ac/fa/acf53138659f4982bbef02acdd30f1fa.jpg?wh=3846*1377" alt=""></p><h2>RPC在架构中的位置</h2><p>围绕RPC我们讲了这么多，那RPC在架构中究竟处于什么位置呢？</p><p>如刚才所讲，RPC是解决应用间通信的一种方式，而无论是在一个大型的分布式应用系统还是中小型系统中，应用架构最终都会从“单体”演进成“微服务化”，整个应用系统会被拆分为多个不同功能的应用，并将它们部署在不同的服务器中，而应用之间会通过RPC进行通信，可以说RPC对应的是整个分布式应用系统，就像是“经络”一样的存在。</p><p>那么如果没有RPC，我们现实中的开发过程是怎样的一个体验呢？</p><p>所有的功能代码都会被我们堆砌在一个大项目中，开发过程中你可能要改一行代码，但改完后编译会花掉你2分钟，编译完想运行起来验证下结果可能要5分钟，是不是很酸爽？更难受的是在人数比较多的团队里面，多人协同开发的时候，如果团队其他人把接口定义改了，你连编译通过的机会都没有，系统直接报错，从而导致整个团队的开发效率都会非常低下。而且当我们准备要上线发版本的时候，QA也很难评估这次的测试范围，为了保险起见我们只能把所有的功能进行回归测试，这样会导致我们上线新功能的整体周期都特别长。</p><p>无论你是研发还是架构师，我相信这种系统架构我们肯定都不能接受，那怎么才能解决这个问题呢？</p><p>我们首先都会想到可以采用“分而治之”的思想来进行拆分，但是拆分完的系统怎么保持跟未拆分前的调用方式一样呢？我们总不能因为架构升级，就把所有的代码都推倒重写一遍吧。</p><p><strong>RPC框架能够帮助我们解决系统拆分后的通信问题，并且能让我们像调用本地一样去调用远程方法。</strong>利用RPC我们不仅可以很方便地将应用架构从“单体”演进成“微服务化”，而且还能解决实际开发过程中的效率低下、系统耦合等问题，这样可以使得我们的系统架构整体清晰、健壮，应用可运维度增强。</p><p>当然RPC不仅可以用来解决通信问题，它还被用在了很多其他场景，比如：发MQ、分布式缓存、数据库等。下图是我之前开发的一个应用架构图：</p><p><img src="https://static001.geekbang.org/resource/image/50/be/506e902e06e91663334672c29bfbc2be.jpg?wh=3205*1778" alt=""></p><p>在这个应用中，我使用了MQ来处理异步流程、Redis缓存热点数据、MySQL持久化数据，还有就是在系统中调用另外一个业务系统的接口，对我的应用来说这些都是属于RPC调用，而MQ、MySQL持久化的数据也会存在于一个分布式文件系统中，他们之间的调用也是需要用RPC来完成数据交互的。</p><p>由此可见，RPC确实是我们日常开发中经常接触的东西，只是被包装成了各种框架，导致我们很少意识到这就是RPC，让RPC变成了我们最“熟悉的陌生人”。现在，回过头想想，我说RPC是整个应用系统的“经络”，这不为过吧？我们真的很有必要学好RPC，不仅因为RPC是构建复杂系统的基石，还是提升自身认知的利器。</p><h2>总结</h2><p>本讲我主要讲了下RPC的原理，RPC就是提供一种透明调用机制，让使用者不必显式地区分本地调用和远程调用。RPC虽然可以帮助开发者屏蔽远程调用跟本地调用的区别，但毕竟涉及到远程网络通信，所以这里还是有很多使用上的区别，比如：</p><ul>
<li>调用过程中超时了怎么处理业务？</li>
<li>什么场景下最适合使用RPC？</li>
<li>什么时候才需要考虑开启压缩？</li>
</ul><p>无论你是一个初级开发者还是高级开发者，RPC都应该是你日常开发过程中绕不开的一个话题，所以作为软件开发者的我们，真的很有必要详细地了解RPC实现细节。只有这样，才能帮助我们更好地在日常工作中使用RPC。</p><h2>课后思考</h2><ol>
<li>你应用中有哪些地方用到了RPC？</li>
<li>你认为，RPC使用过程中需要注意哪些问题？</li>
</ol><p>欢迎留言和我分享你的思考和疑惑，也欢迎你把文章分享给你的朋友，邀请他加入学习。我们下节课再见！</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/7b/98/8f1aecf4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>楼下小黑哥</span>
  </div>
  <div class="_2_QraFYR_0">我们目前服务内部调用都是使用 rpc，对外接口采用 restful 接口。<br>采用rpc 开发最终要我觉得是设置合理超时时间以及重试次数。因为 rpc毕竟需要走网络调用，存在网络耗时。超时间太短，可能导致服务提供端实际执行成功，消费端却因为超时报错结束。这就有可能导致数据状态不一致。<br><br>另外，整个链路的超时需要合理设置，如A-》B-〉C，A的超时时间要大于B。<br><br>重试次数也需要关注，默认情况下，如 dubbo 重试次数为2，调用失败的情况下，框架会重新调用。而有些服务不能重复调用。<br>服务提供者应该是最熟悉自己服务的，所以服务提供者可以设置默认超时时间以及重试次数，消费者不设置，就会采用服务提供者参数设置。<br>😅想了下，开发过程中其实还有好多细节要注意，细节决定成败，后面章节可以再聊聊，让我们跟老师一起学习。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 思路很好。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-18 09:17:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJTBnGDMyaGia98uoKVFwpVFC4CiafrWySk2DsTA3pDSrm4wEfeFSnsnWc9qzcVWnDZEsYtV1DcEkYQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_c8b5a1</span>
  </div>
  <div class="_2_QraFYR_0">1、你应用中有哪些地方用到了 RPC？<br>在公司内部不同服务之间的调用都是走的RPC<br>2、你认为，RPC 使用过程中需要注意哪些问题？<br>1）下游服务的服务能力，避免因为你的调用把别人给调挂了，要事前协商好qps等，做好限流<br>2）调用服务异常时，要考虑降级、重试等措施<br>3）核心的服务不能强依赖非核心的服务，避免核心服务因为非核心服务异常而不可用</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 很好</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-25 13:37:33</div>
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
  <div class="_2_QraFYR_0">1：你应用中有哪些地方用到了 RPC？<br>       我认为还是分开说比较好一点，侠义的RPC就是为了实现进程间方法调用像进程内方法调用一样简单。广义的RPC可以认为只要跨进程通信就是RPC，即使是在一台机器上两个进程间进行通信了，也是RPC，，不过目前来看RPC更强调侠义的含义。广义的含义和网络通信是一个意思，话说两台电脑之间想通信不靠网络靠什么呢？人肉操作嘛？<br>OK，RPC核心就是为了应用解藕而存在，只有系统间是进程通信必然会用到RPC。<br><br>2：你认为，RPC 使用过程中需要注意哪些问题？<br>第一服务注册服务发现，服务注册中心<br>第二服务治理，有多少服务？都是那些服务？谁调用谁？怎么下线服务？怎么修改服务分组？怎么修改服务别名？服务限流怎么控制？服务降级怎么控制？服务上下游信息？服务调用链信息？<br>第三服务监控，方法调用链监控？每个方法的监控，比如：TPS&#47;调用量&#47;可用率&#47;以及各种汇总聚合信息，最小&#47;最大&#47;平均&#47;各种TP分位统计，报警配置信息等等，这些东西一下就知道服务是否可用？在一个完整的调用链上那个服务比较慢？也可以统计服务的调用次数？对于分析排查问题，尤其是性能问题帮助非常大<br>第四日志查询平台，实时日志、现场日志、历史日志都能根据关键字界面化傻瓜式的查询出来，也能统计出日志里的报错信息关键字等。非常利于业务问题的排查，及时发现系统中的业务问题。<br>第五配置中心，可以调整日志级别、各种业务开关、服务分组别名信息，对于服务控制会非常灵活。<br>话说JD这些做的还挺不错，不过全链路跟踪系统好像做的还不太好，如果这个改造好了，那链路上那个系统慢就一目了然了，性能问题的排查会更简单方便。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-10 23:19:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/c9/23/76511858.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>洛奇</span>
  </div>
  <div class="_2_QraFYR_0">第一幅图中，编解码是一种码吗？<br>为什么序列化后生成编解码后还要再编码，才能放到网络上呢?<br>为什么不能直接一步序列化就放到网络上？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 序列化是对方法调用的请求信息进行处理，编解码是对网络传输消息进行处理。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-17 23:56:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/ed/91/5dece756.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>陛下</span>
  </div>
  <div class="_2_QraFYR_0">现在最严重的问题就是事物吧，分布式事物，感觉一直没有好办法解决</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 分布式事务是痛点，目前有开源方案了。我们为了性能常用补偿事务，最终一致性。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-18 11:08:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/0f/bf/ee93c4cf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>雨霖铃声声慢</span>
  </div>
  <div class="_2_QraFYR_0">1. 你应用中有哪些地方用到了 RPC？<br>我们的应用是微服务架构的，RPC就是连接这些微服务之间的纽带。<br>2. 你认为，RPC 使用过程中需要注意哪些问题？<br>因为RPC也是网络调用，性能方面肯定不如本地调用，所有RPC的API设计要仔细考虑，比如一次性能完成的调用就不要走多次调用。另外我认为最重要的是要有监控系统能监控所有的调用链，方便问题排查和性能调优。 </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍👍👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-18 23:10:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/bf/55/198c6104.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小伟</span>
  </div>
  <div class="_2_QraFYR_0">我觉得广义来说，只要涉及调用的网络通信都属于RPC的范畴，包括Rest API，因为本质上都是走网络通信的非本地调用。<br>关于服务内部用RPC，服务外部用Rest API，主要考量的还是安全性。服务内部网络一般认为是相对安全的，因为已经有了很多手段来避免数据包外泄，故不需要强认证。而服务外部是对公网开放的，或至少部分是对公网开放的，数据在公网传输被认为是相对不安全的，所以要强认证。认证强弱的差别导致了RPC分成两派：针对服务内的高效的狭义RPC，和针对服务外的相对低效的RPC(Rest API)。<br>早在EJB的时代，有个叫RMI的东西，流程和RPC惊人的一致，只不过RMI还需要调用方维护大量底层细节，感觉RPC是从RMI发展来的，是好用版RMI。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 外部用restful，主要是http协议标准，浏览器支持简单。而不是安全。都可以采用ssl加密</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-23 22:16:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/82/3d/356fc3d6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>忆水寒</span>
  </div>
  <div class="_2_QraFYR_0">1、你应用中有哪些地方用到了 RPC？<br>答：我们目前系统进行拆分（C++开发的），也是分布式部署的，我们的RPC在系统间交互（或同步）数据时使用RPC接口进行调用。其次，我们RPC还是一个信息管家，可以通过事件进行提醒应用层主备机信息等。<br>2、你认为，RPC 使用过程中需要注意哪些问题？<br>答：这个问题让我想起了一次面试中面试官问我“你觉得一个设计RPC框架中最重要的是哪一点？”我当时首先说了RPC框架首先是通信、自定义协议（protobuf）、序列化、注册中心。我们的RPC由于C++开发的，只提供消息传输的功能，序列化和协议在应用层做的（主要是考虑不同项目的业务也有区别）。我觉得其中最重要的就是注册中心（数据中心）实现了，这个决定了RPC所能提供扩展功能。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这些都是rpc的核心功能。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-18 09:19:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/4a/59/899e3b06.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Leon</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，最近在从零开始手写个RPC框架，深有感触。<br>实现了多种序列化机制，集成了protobuff、protostuff、json和hessian等。<br>目前在编码服务发现，基于zk，思路是有，不是太清晰，编码总是断断续续。<br>希望多点实战性的指导</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你好，leon。互相交流。都是我们实战的经验</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-18 15:12:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/67/0e/c77ad9b1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>eason2017</span>
  </div>
  <div class="_2_QraFYR_0">调用过程中超时了怎么处理业务？<br>重试机制，降级处理。<br>什么场景下最适合使用 RPC？<br>网络安全稳定的环境。<br>什么时候才需要考虑开启压缩？<br>压缩后，数据量有明显的降低，压缩会使用CPU等资源，还是要看性价比。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-18 10:27:06</div>
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
  <div class="_2_QraFYR_0">服务调用的RPC框架<br>RPC在使用中需要接口的版本，比如服务提供方升级了接口，比如增加字段了。请求方没有修改接口的版本。这样调用就会出问题了。这个问题的根本属于协议内容，如果设计好的协议支持兼容扩展，一般是向下兼容，就能实现低版本的调用方照样调用高版本的服务方</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 接口是契约，需要接口设计者做向下兼容</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-17 21:28:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/9b/88/34c171f1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>herongwei</span>
  </div>
  <div class="_2_QraFYR_0">我是刚开始学习 RPC 的同学，团队内部要考虑使用 RPC 了，老大让我来学习下这方面的相关的资料，第一时间就想到了极客时间，搜了下，发现了老师的这个专栏，果断下手，跟着学起来，希望能有所收获~</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-19 10:30:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/b0/1c/2e30eeb8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>旺旺</span>
  </div>
  <div class="_2_QraFYR_0">RPC 是整个应用系统的“经络”，精辟</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-26 17:43:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/c2/fe/038a076e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>阿卧</span>
  </div>
  <div class="_2_QraFYR_0">1. 服务内部系统间交互时会经常用到rpc，例如创建订单的流程，订单中心调用业务系统的创单，并返回结果。<br>2. 要注意，rpc接口调用超时，接口访问量过高导致服务被拖垮等问题。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-23 14:30:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/42/e5/61cfe267.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Eclipse</span>
  </div>
  <div class="_2_QraFYR_0">1 请求远程api接口，RESTful?<br>2 通讯的话，netty更适合做底层的事，rpc设计了部分业务治理？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你好，eclipse。restful是一种常用的请求方式，在高性能，大并发的情况下，私有的rpc协议也很常用。如grpc，dubbo等。好的rpc总算要伴随治理才能完善。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-17 19:34:31</div>
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
  <div class="_2_QraFYR_0">这个专栏反复看了好多遍了，受益匪浅。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-01 16:58:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://wx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKYLPAlGUWic4yAqsGtEYBSRR7gDjyg9yiaJicNhMwiaNw4rMKQ5DHTfp7gmic0gpqEwCZaou8G6CdHKCg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ant</span>
  </div>
  <div class="_2_QraFYR_0">近期在做一个世界boss功能，其中涉及到一个跨服的世界boss伤害排行榜在服务器重启后需要通过rpc去玩家所在服务器获取消息，最开始开发时第一版的实现是遍历排行榜逐个rpc去请求消息，在代码review中发现了这个问题并优化。优化点：将相同区服玩家打包通过一次rpc拿到该服上榜玩家的信息。 原因：rpc即便让我门在使用过程中像是本地操作，但他毕竟是跨服务器的远程调用，中间各种网络请求，在耗费资源上还是远高于本地请求的所以，请求次数能少则少。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-27 16:45:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/2f/ed/a87bb8fa.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>此鱼不得水</span>
  </div>
  <div class="_2_QraFYR_0">希望老师可以详细讲解一下服务注册和发现的流程，目前网上很多的资料对这个部分都介绍的比较马虎。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 后面有分享，需要解答的可以提问。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-18 11:40:52</div>
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
  <div class="_2_QraFYR_0">老师文中提到RPC主要体现再两个方面: <br>屏蔽远程调用跟本地调用的区别，让我们感觉就是调用项目内的方法；<br>隐藏底层网络通信的复杂性，让我们更专注于业务逻辑。<br><br>这样话是不是RPC的实现和具体的传输协议没有关系，不管是TCP还是UDP哪怕是http&#47;2只要能满足上面就可以？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: rpc可以在不同网络协议上实现。具体的一种rpc实现一般采用一种网络协议。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-18 09:39:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/c9/23/76511858.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>洛奇</span>
  </div>
  <div class="_2_QraFYR_0">老师，RPC和RMI  (远程方法调用) 有什么关系？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 目前没有关系，目标是一样，进行远程调用</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-18 00:10:52</div>
  </div>
</div>
</div>
</li>
</ul>