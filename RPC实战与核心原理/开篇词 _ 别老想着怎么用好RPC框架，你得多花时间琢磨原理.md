<audio title="开篇词 _ 别老想着怎么用好RPC框架，你得多花时间琢磨原理" src="https://static001.geekbang.org/resource/audio/b5/15/b567a1a4a27577d97243acc07a50dd15.mp3" controls="controls"></audio> 
<p>你好，我是何小锋。欢迎你和我一起学习RPC。</p><p>在专栏开始之前，我先简单介绍下自己。我是1998年从北航毕业的，毕业以后我就一直在一线编程写代码。2011年，我正式加入京东，刚好赶上了京东的快速发展期，一路做到了现在的技术架构部首席架构师。盘点下在京东的这9年时间，我参加过17次大促备战，和我的技术团队一起见证了京东的技术演进过程。我也曾带领团队攻克过很多技术领域难题，包括自主研发微服务框架、高性能消息中间件、智能监控以及容器平台等等。</p><p>近几年，我主攻分布式系统架构与设计，这也是我的专长所在。而在搭建分布式系统的过程中，我发现 RPC  总能充当较为关键的角色，它对整个分布式系统性能的提升起到了非常重要的作用。</p><p>我期待通过这个专栏，能把我这些年积攒的一些有关RPC的实战经验分享给你。</p><h2>为什么要学习RPC？</h2><p>做任何事情都应该  Start with Why，那我们就先来说说为什么要学习RPC。要回答这个问题，我们就得先考虑下RPC的实际应用场景。</p><p>说到RPC，可能你的第一反应就是“微服务”。RPC最大的特点就是可以让我们像调用本地一样发起远程调用，这一特点常常会让人感觉RPC就是为“微服务”或SOA而生的。现在的大多数应用系统发展到一定规模之后，都会向“微服务化”演进，演进后的大型应用系统也的确是由一个个“微服务”组成的。</p><!-- [[[read_end]]] --><p>我们可以说 RPC 是“微服务”的基础，这一点是毋庸置疑的。现在我们就可以反过来想这样一个问题——RPC是不是只应用在“微服务”中呢？</p><p><strong>当然不是，只要涉及到网络通信，我们就可能用到RPC。</strong>一起看这样两个例子。</p><p>例1：大型分布式应用系统可能会依赖消息队列、分布式缓存、分布式数据库以及统一配置中心等，应用程序与依赖的这些中间件之间都可以通过RPC进行通信。比如 etcd，它作为一个统一的配置服务，客户端就是通过gRPC框架与服务端进行通信的。</p><p>例2：我们经常会谈到的容器编排引擎 Kubernetes，它本身就是分布式的，Kubernetes的 kube-apiserver 与整个分布式集群中的每个组件间的通讯，都是通过gRPC框架进行的。</p><p>所以说，RPC的应用场景还是非常广泛的。既然应用如此广泛，那它的核心价值又在哪里呢？</p><p><strong>在我看来，RPC是解决分布式系统通信问题的一大利器。</strong></p><p>分布式系统中的网络通信一般都会采用四层的TCP协议或七层的HTTP协议，在我的了解中，前者占大多数，这主要得益于TCP协议的稳定性和高效性。网络通信说起来简单，但实际上是一个非常复杂的过程，这个过程主要包括：对端节点的查找、网络连接的建立、传输数据的编码解码以及网络连接的管理等等，每一项都很复杂。</p><p>你可以想象一下，在搭建一个复杂的分布式系统过程中，如果开发人员在编码时要对每个涉及到网络通信的逻辑都进行一系列的复杂编码，这将是件多么恐怖的事儿。所以说，网络通信是搭建分布式系统的一个大难题，是一点不为过的，我们必须给予足够的重视。</p><p>而RPC对网络通信的整个过程做了完整包装，在搭建分布式系统时，它会使网络通信逻辑的开发变得更加简单，同时也会让网络通信变得更加安全可靠。</p><p>现在你是不是感觉到学好RPC是很有必要的？</p><h2>如何学习RPC？</h2><p>那我们应该怎么去学习RPC呢？</p><p>其实，深刻了解了为什么之后，怎么学这个问题并不难找到答案。就我自己的经验来看，我觉得可以用“<strong>逐步深入</strong>”这四个字来概括我的学习方式。</p><p>说起来也特别简单。当我们认识到，使用RPC就可以像调用本地一样发起远程调用，用它可以解决通信问题，这时候我们肯定要去学序列化、编解码以及网络传输这些内容。</p><p>把这些内容掌握后，你就会发现，原来这些只是RPC的基础，RPC还有更吸引人的点，它真正强大的地方是它的治理功能，比如连接管理、健康检测、负载均衡、优雅启停机、异常重试、业务分组以及熔断限流等等。突然间，你会感觉自己走进了一个新世界，这些内容会成为你今后学习RPC的重点和难点。</p><p>这个逐步深入的过程，一定离不开真实的实践场景。学习知识，解决问题，遇到新问题，继续学习，不断解决问题，最后你会发现自己的学习曲线大概是这样的。</p><p><img src="https://static001.geekbang.org/resource/image/74/8c/74539ca9da65ee0461ddb9299c277f8c.jpeg?wh=1066*990" alt=""></p><p>总结一下，学习RPC时，我们先要了解其基本原理以及关键的网络通信部分，不要一味依赖现成的框架；之后我们再学习RPC的重点和难点，了解RPC框架中的治理功能以及集群管理功能等；这个时候你已经很厉害了，但这还不是终点，我们要对RPC活学活用，学会提升RPC的性能以及它在分布式环境下如何定位问题等等。</p><h2>整个专栏能让你学到什么？</h2><p>上面提到的这些内容，就是我想通过这个专栏和你分享的。下面我来讲下本专栏的设计思路。</p><p>我把整个专栏的内容分为了三大部分，分别是基础篇、进阶篇和高级篇。</p><p><strong>基础篇：</strong>重点讲解RPC的基础知识，包括RPC的基本原理以及它的基本功能模块，夯实基础之后，我们会以一场实战，通过剖析一款RPC框架来将知识点串联起来。</p><p><strong>进阶篇：</strong>重点讲解RPC框架的架构设计，以及RPC框架集群、治理相关的知识。这部分我会列举很多我在运营RPC框架中遇到的实际问题，以及这些问题的解决方案。</p><p><strong>高级篇：</strong>通过对上述两部分的学习，你已经对RPC有了较高层次的理解了。在这部分，我主要会从性能优化、线上问题排查以及一些比较有特色的功能设计上讲解RPC的应用。</p><p><img src="https://static001.geekbang.org/resource/image/d1/bf/d15af80828fc3a9da2fea7a1aa232dbf.jpg?wh=750*2285" alt=""></p><p>整个专栏跟下来，虽然主要讲解的都是RPC相关的知识，但你会接触到很多的案例和解决方案，它们首先会使你对RPC的理解到达一个较高的层次；其次就是这些知识和解决方案会有相通性，只要你能举一反三，对你今后的工作就会有很大的帮助。</p><p>最后，我也很想听听你的想法。我们可以在留言区认识一下，期待你和我讲讲你的工作经历，你对RPC的认识，以及学习它的痛点、难点，我也好有针对性地为你讲解。现在，就让我们共同开启这段学习之旅吧！</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/30/8a/b5ca7286.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>业余草</span>
  </div>
  <div class="_2_QraFYR_0">通俗解释，如果 HTTP 是普通话， 那么 RPC 就是方言。既然是方言，你就会看到各家都有自己的方言， 比如 google 的 gRPC, 百度的 bRPC, facebook 的 thrift，阿里的 dubbo... 而 HTTP 只有官方的一套标准</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你好，业余草。rpc目前还没有行业标准，百家争鸣，有竞争才有动力。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-17 17:25:05</div>
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
  <div class="_2_QraFYR_0">真正的大佬，写的代码也见识过，可惜JSF源码不开源，反编译看过一点。<br>互联网三剑客：RPC、MQ、REDIS，不过底座是网络通信。<br>RPC的核心在于使进程间通信像进程内通信一样简单，更直白一点调用其他应用的方法好似调用本地方法一样简单方便。进程内之所以方便是因为写操作系统的人把复杂的事情给处理了，让业务研发专注于业务逻辑。进程间的调用简单是因为写RPC框架的人把复杂的工作给做了，使业务研发继续专注于业务逻辑。<br>进程间能通信是第一步，后面随着机器的增多事情就变得复杂了起来，所以，有了服务注册中心、有了服务治理平台、有了配置管理中心、有了容器平台、有了各个各有的监控平台、有了UMP&#47;UCC&#47;LOGBOOK&#47;MDC&#47;OPS&#47;QONE&#47;JONE&#47;CAP等等一系列的东西。<br>不过问题的根源还是单机的性能及容量有上限导致的，否则也就没这么多的事情了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-10 22:28:46</div>
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
  <div class="_2_QraFYR_0">原来RPC最强大的是治理功能，之前只是用作网络通信</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你好，一步。为了高可用，RPC需要治理</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-17 17:23:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/e8/0a/16a3609e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>逝光无痕</span>
  </div>
  <div class="_2_QraFYR_0">RPC是通过四层TCP协议还是七层HTTP协议去完成呢，以前看有些文章说是只能是TCP协议去完成调用？对这有些疑惑，希望从这里得到答案！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 大部分采用的tcp协议。grpc采用的http2</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-18 11:07:18</div>
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
  <div class="_2_QraFYR_0">rpc的基础是序列化、编解码和网络传输；高级应用是健康检测、负载均衡、服务治理等。<br>学问太多了。老师掌握好rpc的应用和原理，算是什么级别的程序员啊？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 公司里面的牛人</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-22 16:29:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/b2/08/92f42622.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>尔冬橙</span>
  </div>
  <div class="_2_QraFYR_0">老师，netty和rpc框架间是什么关系呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: netty可以作为rpc的网络传输层</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-26 01:26:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/fb/c2/09a2712e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Walter_Mitty</span>
  </div>
  <div class="_2_QraFYR_0">生涯进入瓶颈期，过来充充电</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 期望收获满满</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-19 11:42:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/5d/d4/e5ea1c25.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>sun留白</span>
  </div>
  <div class="_2_QraFYR_0">你好，何老师。可以说说rpc和restful的区别吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 两个处理流程形似，rpc一般更喜欢tcp，用二进制协议，性能更好，常用的rpc框架功能比restful客户端更多。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-18 23:57:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/SM4fwn9uFicXU8cQ1rNF2LQdKNbZI1FX1jmdwaE2MTrBawbugj4TQKjMKWG0sGbmqQickyARXZFS8NZtobvoWTHA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>td901105</span>
  </div>
  <div class="_2_QraFYR_0">老师，问一下，一般rpc的文章中都会有一张rpc调用的原理图，里面的client stub和server stub是实际存在的吗？还是一个虚拟概念？如果是实际存在的话能举一个具体的例子吗？比如像grpc中具体是怎么实现client stub和server stub的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: stub是实际存在的，有些可能运行中生成一个代理类，grpc的stub是静态代理</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-09 15:54:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Gswh7ibY4tubXhp0BXOmV2pXZ3XsXic1d942ZMAEgWrRSF99bDskOTsG1g172ibORXxSCWTn9HWUX5vSSUVWU5I4A/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>奔奔奔跑</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，有幸购买您的专栏，我有个疑问想请教一下，你说的在学习rpc的过程中解决实际问题，但是我好像没遇见过啥实际问题呀，框架一用，一跑，完事了。所以我的学习收益太低了，老师能给我讲讲吗？谢谢了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 很多问题在框架里面解决了，了解背后的原理</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-27 00:15:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/98/73/2b208bc9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Hanson</span>
  </div>
  <div class="_2_QraFYR_0">您好！RPC可用于TCP或者HTTP，gRPC是基于HTTP&#47;2通讯的，那往后的HTTP&#47;3会有什么样的突破呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: http3减少了传输交互，性能会更快</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-18 09:57:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/64/15/9c9ca35c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>sipom</span>
  </div>
  <div class="_2_QraFYR_0">rpc屏蔽系统间调用复杂性，分布式系统的基础。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞同。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-22 11:28:03</div>
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
  <div class="_2_QraFYR_0">您好，何老师很幸运能跟随您学习更好的了解和使用rpc相关计数，本身我是从事游戏服务器开发，由于现在大家对社交或者说交互的需求越来越多，导致游戏业务中涉及到越来越多的跨服功能，促使我接触并使用rpc，但仅仅停留在能熟练使用，知其然不知其所以然，希望能在您的课程中加深对rpc的学习和了解，并能够在以后的工作中开发出一款针对游戏业务的高可用性的rpc框架！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 加油</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-27 15:34:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/89/e5/a346ba59.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>沈伟敏</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，想问一下，问什么京东自研rpc需要配置token呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 安全需要</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-27 09:29:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/53/a8/abc96f70.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>return</span>
  </div>
  <div class="_2_QraFYR_0">老师 请教一下， rpc和restful的关系，区别。两者的应用场景。<br>我感觉我们好像都用的restful。 对rpc感觉有误解 老感觉没那个必要。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: rpc可以基于tcp的私有协议。在高并发，高性能场景有优势</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-19 18:09:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/55/5c/e604f38a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>program man</span>
  </div>
  <div class="_2_QraFYR_0">老师，RPC和服务治理不是一个内容吧？服务治理应该是对RPC应用的扩展？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 实际应用场景是需要两个结合起来用。很多rpc框架都包含服务治理的内容</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-21 17:59:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/09/1e/fc5144ff.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>王轲</span>
  </div>
  <div class="_2_QraFYR_0">kubernetes是gRPC? 是http吧</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你好，王柯。grpc基于http2</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-20 00:41:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1c/a9/54/8b0d4b0f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Why</span>
  </div>
  <div class="_2_QraFYR_0">我记得dubbo就是一个号称RPC的框架，使用的时候在xml中可以配置通过什么样的协议调用远程的服务，比如配置tcp，rmi，http等，所以我的理解是RPC是实现远程调用的一种思想，而具体怎么去实现这种思想需要通过底层的协议</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 有多种实现。了解原理就会举一反三</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-19 15:26:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/b8/49/99ca2069.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>哼歌儿李</span>
  </div>
  <div class="_2_QraFYR_0">rpc是分布式系统的关键</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-29 15:02:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/a5/36/d93c851b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jinny</span>
  </div>
  <div class="_2_QraFYR_0">老师，我是一位老新人，老是因为工作时间也10几年了，但之前的工作比较杂，各种类型的开发都做（主要基于C和C++）， 最近几年接触java，还没扎实的java编程基础，现在学习这些架构理论不知道能否学好。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 加油</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-02 11:21:00</div>
  </div>
</div>
</div>
</li>
</ul>