<audio title="34｜GitOps 开发循环慢，时间都去哪了？" src="https://static001.geekbang.org/resource/audio/76/84/76ebd5ffa376aca63bf3d099c3eb0d84.mp3" controls="controls"></audio> 
<p>你好，我是王炜。</p><p>从这节课开始，我们正式进入到云原生开发领域的学习。</p><p>相比较传统的单体应用，云原生应用有非常多的不同之处。比如，云原生应用往往由多个微服务组成，使用了容器技术解决业务的打包和运行问题，此外，它还使用了很多云上的托管服务，例如 MQ，数据库等。</p><p>随着业务应用对云原生的依赖越来越大，我们会逐渐发现云原生应用的开发越来越复杂，也变得越来越慢。你可以把这理解为把应用迁移到云原生架构下的<strong>“副作用”</strong>，在今天，这些问题仍然没有最优的解决方案。</p><p>这节课，我会带你从几个角度了解造成云原生开发变慢的原因，从开发循环反馈的概念讲到架构演进。此外，我还会比较单体应用和云原生应用在开发上差异，带你深入理解为什么会产生这些“副作用”。最后介绍两种提升开发效率的技术方案。</p><h2>开发循环反馈</h2><p>我们先来了解一下什么是开发循环反馈，这要从软件研发过程开始说起。</p><p>在软件研发的过程中，开发者在接到需求后会进行构思、编码。根据编码难度的不同，在编写完一段代码后，它们需要对程序进行编译并运行，以便查看结果。如果结果不符合预期，则会尝试输出一些变量值进行调试，如果这样不能满足调试需求，就需要进行断点调试。</p><p>调试完成并找到问题后，再修复代码，并重新进行上述的过程。</p><!-- [[[read_end]]] --><p>这个完整的过程，也就是我们说的开发循环反馈，如下图所示。</p><p><img src="https://static001.geekbang.org/resource/image/52/b5/52615b3a0419322fe34863c383ee97b5.jpg?wh=1893x1184" alt="图片"></p><p>通过上面的描述，我们很容易得出下面几个结论。</p><ol>
<li>大部分情况下，一次开发循环反馈验证的代码量在几行和几十行之间，取决于编码难度。</li>
<li>在完成一个需求时，我们可能需要经历数十个甚至数百个开发循环反馈。</li>
<li>开发循环反馈的效率很大程度上决定开发效率。</li>
</ol><p>在上面的结论中，第一点和第二点都很好理解。关于第三点我举一个例子，在开发一些大型的 Java 项目的时候，修改代码后要查看编码效果，编译和启动过程可能长达 10 分钟。这意味着，每次编码循环反馈至少有 10 分钟是花在无效的等待上的。也就是说，满打满算一小时只能进行 6 次开发循环反馈，<strong>这是导致开发效率变低的原因之一。</strong>不过显然，这并不是将应用迁移到云原生架构导致的。</p><h2>开发循环反馈在架构演进下的差异</h2><p>现在，让我们来简单回顾下在不同的架构下开发循环反馈的差异。</p><p>我将按照架构演进顺序，依次分析<strong>前云时代架构、云时代架构和云原生时代架构</strong>。</p><h3>前云时代架构</h3><p>在前云时代架构下，我们通常使用瀑布的开发方式，最经典的例子是操作系统。通常，我们需要数年的时间进行研发，也不经常发布更新。</p><p>在这种架构体系下，<strong>开发循环反馈的效率取决于项目的大小，但通常我们可以很容易地在本地进行开发。</strong></p><h3>云时代架构</h3><p>在云时代架构下，敏捷和 DevOps 的开发方法代替了瀑布的开发方法，我们可以更快地开发和交付应用。</p><p>相比较前云时代的架构，云时代的架构更多使用了云端的技术，例如远程和云上开发开始出现，我们可以通过连接云端的机器进行开发和编译，这在一定程度解决了本地资源不足导致的编译和启动过慢的问题。</p><p>在这种架构体系下，我们可以使用<strong>大型的云上开发机</strong>来提升开发循环反馈效率，这种开发方式逐渐变得流行起来。</p><h3>云原生时代架构</h3><p>在云原生时代架构下，业务应用产生了巨大的变化：<strong>微服务</strong>。</p><p>越来越多的单体应用被拆分成了微服务，微服务各司其职，通常一个完整的业务流程需要调用多个微服务才能完成，如下图所示。</p><p><img src="https://static001.geekbang.org/resource/image/16/98/16a530bd28ec99f5fb9544cc74923c98.jpg?wh=1920x1111" alt="图片"></p><p>在微服务架构下，如果本地资源充足，只需要将所有依赖的微服务在本地运行起来，小型的应用仍然可以在本地进行开发。待开发的微服务仍然可以像单体应用一样进行编码、编译和调试。在这种场景下，<strong>开发循环反馈没有出现变化</strong>。</p><p>但是，随着容器化改造和迁移到 Kubernetes，尤其是对于那些无法在本地开发的大型业务应用来说，原有的开发循环反馈流程发生了<strong>巨大的变化</strong>。</p><p><img src="https://static001.geekbang.org/resource/image/19/f3/19b5f8bbf46ae7b1200e72e8a8ddeff3.jpg?wh=1920x1366" alt="图片"></p><p>从上面这张图我们可以看出，在非容器化架构下，开发循环反馈流程只有编码、编译和调试过程。但在将应用迁移到容器和 Kubernetes 架构时，要查看编码效果，我们必须要先<strong>构建镜像、推送到镜像仓库并等待容器重启</strong>才能看到结果，开发效率也随之大幅降低。</p><p>我们已经知道，微服务和容器化是造成开发循环反馈产生变化的最根本原因。那么，要怎么理解它们呢？</p><p>首先，业务应用在进行微服务改造之后，由于不同应用的环境差异，依赖差异以及本地的资源问题，在开发过程，我们很难在本地将所有微服务都启动起来。在这种情况下，使用<strong>云端的 Kubernetes 集群</strong>作为开发环境是唯一的解决办法。</p><p>此外，当业务进行容器化改造以及迁移到 Kubernetes 之后，当要查看某个微服务编码效果时，已经无法像单体应用一样只需要简单的编译和启动了，这是因为这个微服务可能需要依赖其他的微服务、数据库和中间件才能正常工作。这就导致我们只能将应用构建成镜像，并将它部署到<strong>具备业务应用完整依赖</strong>的远端 Kubernetes 集群，然后才能看到编码效果。</p><p>如果从编程的角度来理解，单体应用的调用关系是代码中的函数和类，而微服务应用的调用关系则是通过 HTTP 或 RPC 协议调用其他的微服务。</p><h2>GitOps 架构下的开发循环反馈</h2><p>通过上面的分析，我们已经大体上了解了为什么在 GitOps 架构下开发循环反馈的效率会变低。</p><p>总结来说，在 GitOps 架构下，如果我们要查看编码效果，我们会在下面这些阶段花费大量无效的等待时间：</p><ol>
<li>构建镜像</li>
<li>推送到镜像仓库</li>
<li>修改工作负载镜像版本并等待 Kubernetes 拉取镜像</li>
<li>等待新镜像启动完成</li>
</ol><p>以下面这张示意图为例，试想一下如果所有微服务都部署在云端的 Kubernetes 集群内，要将 B 服务修改为新镜像版本，是不是一定会经历上面的这些步骤呢？</p><p><img src="https://static001.geekbang.org/resource/image/9c/a4/9ce30f8228ea2d8ae5aeb2780ed19aa4.jpg?wh=1920x1232" alt="图片"></p><p>到这里，我相信有一些同学会产生疑问，上面的这些阶段难道不是 GitOps 自动化的优势吗？为什么会变成拖累开发效率的原因？</p><p>从发布角度来说，这些步骤是必不可少的，这也是 GitOps 的优势。但从开发角度来说，每次编码循环反馈都需要经历这些步骤是<strong>非常浪费时间</strong>的。</p><p>在开发过程中，既然最大的耗时是在构建镜像的过程里，那么，怎样才能降低镜像构建对开发效率的影响呢？</p><h2>提高开发循环反馈效率</h2><p>在 GitOps 架构下，要提高开发循环反馈效率，我为你提供下面三个思路。</p><ol>
<li>提升镜像构建和拉取速度。</li>
<li>本地+远程的混合开发方式。</li>
<li>远程开发。</li>
</ol><h3>提升镜像构建和拉取速度</h3><p>要提高开发循环的反馈效率，最容易想到的就是提升镜像构建和拉取速度。例如，你可以按照<a href="https://time.geekbang.org/column/article/621351">第 14 讲</a>的内容减小镜像体积大小来加快构建速度，也可以优化 Dockerfile 让构建过程使用缓存来加快速度，还可以部署私有镜像仓库并将它和 Kubernetes 集群配置在同一个 VPC 网络下提升拉取速度。</p><p><strong>这种方式虽然能在一定程度提高开发循环反馈效率，但由于它仍然需要构建镜像，所以并不能从根本上解决问题。</strong></p><h3>本地+远程的混合开发方式</h3><p>我们已经知道，之所以需要将镜像部署到远端 Kubernetes 集群，是因为远端集群具备待开发微服务的所有依赖，例如数据库、中间件和其他微服务等等。</p><p>那换一个思路，<strong>如果我们可以将待开发的微服务在本地直接运行，是不是就不需要构建镜像了呢？</strong>如下图所示。</p><p><img src="https://static001.geekbang.org/resource/image/b3/df/b38c864b962da9ayy96afa3e0cdeefdf.jpg?wh=1920x867" alt="图片"></p><p>以开发 B 服务为例，要实现这种混合的开发方式，我们需要将集群的 B 服务变成 Proxy 代理，它负责将集群内对 B 服务的访问流量转发到本地，并在本地同时启动 Proxy 和 B 服务，这样，就可以将本地的 B 服务对集群的依赖服务（例如数据库、中间件和其他微服务）的请求代理到集群内了。</p><p>通过这种<strong>“双向代理”</strong>能力，我们就可以在本地以源码的方式开发 B 服务了。像开发单体应用一样，我们在编码后不再需要构建镜像等一系列操作，而是可以直接编译源码并启动，开发循环反馈回到了效率最高的时代。</p><p>不过，在真正要实现这种开发方式的时候，我们通常会遇到两个比较大的限制。</p><p>首先，在一些严格的内网环境下，Proxy 代理可能改变网络拓扑结构，原有网络可能失效，此外，Proxy 在一些场景下也可能无法全流量代理。</p><p>其次，要在本地启动 B 服务并不容易。在 Kubernetes 环境下，B 服务的配置可能是由 ConfigMap 或 Secret 提供的。但在本地我们并不能模拟这两种类型的配置文件，在配置非常复杂的情况下，手动编写配置也比较困难。</p><p>所以，这种方式比较适合业务应用比较小，并且某个微服务可以很轻易地在本地运行的情况。</p><h3>远程开发</h3><p>最后，我再来介绍一种远程开发方式，它可以很好地解决我们在上面提到的一系列的限制。</p><p>远程开发核心的思想仍然是：<strong>复用远端 Kubernetes 集群的环境和依赖</strong>，如下图所示。</p><p><img src="https://static001.geekbang.org/resource/image/2c/54/2cbb32002ccdf50c3213yy92514fe854.jpg?wh=1920x883" alt="图片"></p><p>以开发 B 服务为例，和本地+远程的混合开发方式不同的是，远端开发的核心原理是将 B 服务修改为“开发模式”，让其以源码的方式启动，并将本地源码的实时修改同步到远端集群 B 服务的容器内，通过重启业务进程的方式，实现本地编码实时生效的效果。</p><p>由于这种方式不需要在本地启动微服务，所以本地只需要有 IDE 编辑器即可，不需要任何编程环境。此外，由于业务进程仍然是在原来的容器内启动的，所以配置文件、ConfigMap 和 Secret 的读取方式没有产生任何变化，这也是业务进程能够在容器里直接启动的根本原因，也是实现远程开发的基础。</p><p>在进行编码时，远程开发方式不需要重新构建镜像，开发体验就和在本地开发单体应用一样，只需要经过编译和启动即可。</p><p>不过，值得注意的是，<strong>由于编译和启动过程都是在容器内进行的，所以通常我们需要为容器设置较大的资源配额，以便获得更好的开发体验。</strong></p><p><strong>从通用性和体验角度来看，在 GitOps 架构下，远程开发是我最推荐的开发方式。</strong></p><h2>总结</h2><p>这节课，我们介绍了开发循环反馈的概念以及它在不同架构演进下的差异，尤其是在云原生时代，随着微服务和容器技术的采用，开发循环反馈发生了巨大的变化。</p><p>在简单的单体应用场景下，开发循环反馈只需要经历编码、编译和运行过程。但在云原生时代，这种极简的编程体验已经不复存在了。</p><p>产生这种巨大变化的根本原因是：受到资源限制，大型云原生微服务应用已经很难在本地进行开发，我们不得不借助云端 Kubernetes 集群作为开发环境，这就导致要查看编码效果，就必须经历构建镜像、推送到镜像仓库、修改工作负载的镜像版本并等待新镜像启动的过程，而这些过程正是开发效率变低的重要因素，开发循环反馈从单体应用的秒级变成了分钟级。</p><p>对于云原生开发来说，目前社区并没有标准的解决方案。对此，我介绍了三种解决方案，它们分别是提升镜像构建和拉取速度、本地+远程混合的开发方式以及远端开发的方式。</p><p>其中，提升镜像构建和拉取速度并不能从根本上解决问题，本地+远程混合开发方式虽然改变了开发循环反馈的方式，但它受限于使用场景且不够通用，而远端开发方式则是一种通用性更强的开发方式，也是我将在下一节课重点介绍的内容。</p><p>下一节课，我将带你深入远程开发，并带你学习如何使用 CNCF Sandbox 项目 Nocalhost 来开发 Kubernetes 应用。</p><h2>思考题</h2><p>最后，给你留一道思考题吧。</p><p>你能和我们分享你现在的应用开发和调试方式吗？例如本地开发、混合开发或者远程开发。</p><p>欢迎你给我留言交流讨论，你也可以把这节课分享给更多的朋友一起阅读。我们下节课见。</p>
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
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>bingo</span>
  </div>
  <div class="_2_QraFYR_0">老师请问一下，使用Nocalhost远程debug Maven应用怎么在启动参数里设置自定义的setting.xml文件</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 在 nocalhost 的配置中 debug 的命令里写上业务启动命令，启动命令带上读取特定的配置文件。<br>实际上进入开发模式之后配置文件都有的，一般只要直接启动业务进程就行。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-24 13:57:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/f3/8d/402e0e0f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>林龍</span>
  </div>
  <div class="_2_QraFYR_0">对于开发循环反馈这个流程，在微服务下是一定会遇到的问题， GitOps自动化是减少了整个的流程所花费的时间。在这三个虽然是解决方案，但是并不是最优秀的做法。其实对于该问题我觉得更应该从开发的角度去分析，<br><br>1.每个函数都有单元测试，确保每个函数的入参后得到的出参能符合预期<br>2.微服务的拆分，通过ddd的方式划分服务，尽可能把业务内的代码不要跨服务去处理。<br>3.在微服务中加入链路服务，可通过链路可视化工具把整个流程进行展示，然后对比各个微服务之间的入参出参是否与预期相同，如不同，则可对应到具体的微服务，具体的函数<br><br>当多个函数里面里面各个入参跟出参符合预期后，最终再把代码上传，构建，推送镜像，k8s启动。这样可以减少这个流程的次数。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 很好的经验分享👍🏻<br>在开发过程最主要还是要解决开发环境和编码查看代码效果的问题。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-24 09:58:05</div>
  </div>
</div>
</div>
</li>
</ul>