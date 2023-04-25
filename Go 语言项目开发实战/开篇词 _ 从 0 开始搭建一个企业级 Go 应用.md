<audio title="开篇词 _ 从 0 开始搭建一个企业级 Go 应用" src="https://static001.geekbang.org/resource/audio/94/fd/94c369b4b59eef3ccafc09e124a7c3fd.mp3" controls="controls"></audio> 
<p>你好，我是孔令飞，很高兴能在这里和你聊聊如何用 Go 构建企业级应用。</p><p>在过去的 5 年里，我一直在腾讯使用 Go 做大型企业级项目。比如说，腾讯云云函数 SCF、腾讯游戏容器平台 TenC、腾讯游戏微服务中台等。目前，我在腾讯云负责容器服务 TKE 的相关研发工作，专注于云原生混合云领域的基础架构开发。</p><h2>“云”是大势所趋，而Go是云时代的语言</h2><p>最近几年，我发现腾讯很多团队的开发语言都在转 Go。其实，不光腾讯，像阿里、华为和百度这类国内一线大厂也都在积极转 Go。甚至不少团队，所有项目都是用 Go 构建的。伴随而来的，就是各个公司对Go研发工程师的需求越来越旺盛。那么， Go 为什么会变得这么火热呢？我认为，原因主要有两个方面。</p><p>一方面，Go 是一门非常优秀的语言，它具有很多核心优势，例如：语言简单、语言层面支持并发编程、跨平台编译和自带垃圾回收机制等，这些优势是这些团队选择 Go 最根本的原因。</p><p>另一方面，也因为 Go 是云时代的语言。为什么这么说呢？下面，我来详细说说。</p><p>随着云计算平台的逐渐成熟，应用上云已经成为一个不可逆转的趋势了，很多公司都选择将基础架构/业务架构云化，例如阿里、腾讯都在将公司内部业务全面云化。可以说，全面云化已经是公司层面的核心 KPI了，我们甚至可以理解为以后所有的技术都会围绕着云来构建。</p><!-- [[[read_end]]] --><p><strong>而云目前是朝着云原生架构的方向演进的，云原生架构中具有统治力（影响力）的项目绝大部分又是用 Go 构建的。</strong>我们从下面这张云原生技术栈语言组成图中可以看到，有  63% 的具有统治力的云原生项目都是用 Go 来构建的。</p><p><img src="https://static001.geekbang.org/resource/image/c6/51/c608623543c26b8f088bea958856f551.png?wh=5703*4942" alt="" title="云原生技术栈"></p><blockquote>
<p><span class="reference">完整的云原生技术栈可参考<a href="https://landscape.cncf.io/images/landscape.png">云原生技术图谱</a></span></p>
</blockquote><p>因此，想要把基础架构/业务架构云化，离不开对这些云原生开源项目的学习、改造。而一个团队为了节省成本，技术栈最好统一。既然我们一定要会 Go，而且 Go 这么优秀，那最好的方式就是将整个团队的语言技术栈 all in Go，这也是 Go 为什么重要的另一个原因了。</p><p>那么，我们用 Go 做什么呢，当然是项目开发。但很多开发者在用 Go 进行项目开发时会面临一系列问题。</p><h2>学习 Go 项目开发面临哪些问题？</h2><p>我带过不少刚接触 Go 语言的开发者，他们为了学习 Go 项目开发，会上网搜很多 Go 相关的技术文章，也确实花了很多时间去学习。但是，当我做 Code Review 时，发现他们开发的代码仍然存在很多问题。</p><p>比如说，有个开发者写的代码依赖数据库连接，没法写单元测试。细问之后，我发现他参考的文章没有将数据库层跟业务层通过接口解耦。</p><p>再比如说，还有一些开发者开发的项目很难维护，项目中出现了大量的 common、util、const 这类 Go 包。只看包名，我完全不知道包所实现的功能，问了之后才发现他是参考了一个带有 dao、model、controller、service 目录的、不符合 Go 设计哲学的项目。</p><p>而这些问题其实只是冰山一角，总的来说，我们在学习 Go 项目开发时会面临以下4大类问题。</p><ol>
<li><strong>知识盲区</strong>：Go 项目开发会涉及很多知识点，但自己对这些知识点却一无所知。想要学习，却发现网上很多文章结构混乱、讲解不透彻。想要搜索一遍优秀的文章，又要花费很多时间，劳神劳力。</li>
<li><strong>学不到最佳实践，能力提升有限</strong>：网上有很多文章会介绍 Go 项目的构建方法，但很多都不是最佳实践，学完之后不能在能力和认知上带来最佳提升，还要自己花时间整理学习，事倍功半。</li>
<li><strong>不知道如何完整地开发一个 Go 项目</strong>：学了很多 Go 开发相关的知识点、构建方法，但都不体系、不全面、不深入。学完之后，自己并不能把它们有机结合成一个 Go 项目研发体系，真正开发的时候还是一团乱，效率也很低。</li>
<li><strong>缺乏一线项目练手，很难检验学习效果</strong>：为了避免闭门造车，我们肯定想学习一线大厂的大型项目构建和研发经验，来检验自己的学习成果，但自己平时又很难接触到，没有这样的学习途径。</li>
</ol><p>为了解决这些问题，我设计了《Go 语言项目开发实战》这个专栏，希望帮助你成为一名优秀的Go开发者，在职场中建立自己的核心竞争力。</p><h2>这个专栏是如何设计的？</h2><p>《Go 语言项目开发实战》这个专栏又是如何解决上述问题的呢？在这个专栏里，我会围绕一个可部署、可运行的企业应用源码，为你详细讲解实际开发流程中会涉及的技能点，让你彻底学会如何构建企业级 Go 应用，并解决 Go 项目开发所面临的各类问题。</p><p>一方面，你能够从比较高的视野俯瞰整个 Go 企业应用开发流程，不仅知道一个优秀的企业应用涉及的技能点和开发工作，还能知道如何高效地完成每个阶段的开发工作。另一方面，你能够深入到每个技能点，掌握它们的具体构建方法、业界的最佳实践和一线开发经验。</p><p>最后我还想强调一点，除了以上内容，专栏最终还会交付给你一套优秀、可运行的企业应用代码。这套代码能够满足绝大部分的企业应用开发场景，你可以基于它做二次开发，快速构建起你的企业应用。</p><p>说了这么多，我们到底能学到哪些技能点呢？我按照开发顺序把它们总结在下面这张图中，图中包含了Go项目开发中大部分技能点。</p><p><img src="https://static001.geekbang.org/resource/image/c4/8c/c4a4bdfc103f193d292b54e44510f28c.jpg?wh=6375*2250" alt=""></p><p>除此之外，<strong>专栏中的每个技能点我都会尽可能朝着“最佳实践”的方向去设计</strong>。例如，我使用的 Go 包都是业界采纳度最高的包，而且设计时，我也会尽可能遵循 Go 设计模式、Go 开发规范、Go 最佳实践、go clean architecture 等。同时，我也会尽量把我自己<strong>做一线Go项目研发的经验，融合到讲解的过程中</strong>，给你最靠谱的建议，这些经验和建议可以让你在构建应用的过程中，少走很多弯路。</p><p>为了让你更好地学习这门课程，我把整个专栏划分为了6个模块。其中，第1个模块是实战环境准备，第2到第6个模块我会带着你按照研发的流程来实际构建一个应用。</p><p><strong>实战准备</strong>：我会先手把手带你准备一个实验环境，再带你部署我们的实战项目。加深你对实战项目的理解的同时，给你讲解一些部署的技能点，包括如何准备开发环境、制作CA证书，安装和配置用到的数据库、应用，以及Shell脚本编写技巧等。</p><p><strong>实战第 1 站：规范设计</strong>：我会详细介绍开发中常见的10大规范，例如目录规范、日志规范、错误码规范、Commit规范等。通过本模块，你能够学会如何设计常见的规范，为高效开发一个高质量、易阅读、易维护的 Go 应用打好基础。</p><p><strong>实战第 2 站：基础功能设计或开发</strong>：我会教你设计或开发一些Go应用开发中的基础功能，这些功能会影响整个应用的构建方式，例如日志包、错误包、错误码等。</p><p><strong>实战第 3 站：服务开发</strong>：我会带你一起解析一个企业级的Go项目代码，让你学会如何开发Go应用。在解析的过程中，我也会详细讲解Go开发阶段的各个技能点，例如怎么设计和开发API服务、Go SDK、客户端工具等。</p><p><strong>实战第 4 站：服务测试</strong>：我会围绕实战项目来讲解进行单元测试、功能测试、性能分析和性能调优的方法，最终让你交付一个性能和稳定性都经过充分测试的、生产级可用的服务。</p><p><strong>实战第 5 站：服务部署</strong>：本模块通过实战项目的部署，来告诉你如何部署一个高可用、安全、具备容灾能力，又可以轻松水平扩展的企业应用。这里，我会重点介绍 2 种部署方式：传统部署方式和容器化部署方式，每种方式在部署方法、复杂度和能力上都有所不同。</p><p>最后，关于怎么学习这个专栏，我还想给你一些建议。</p><p>第一，我建议你先学习这个专栏的图文内容，再详细去读源码。学习过程中如果产生一些想法可以通过修改代码，并查看运行结果的方式来加以验证。这个专栏的代码，我都放在GitHub上，你可以点击<a href="https://github.com/marmotedu/iam">这个链接</a>查看。</p><p>第二，在专栏中，我不会详细去介绍每行代码，只会挑选一些核心代码来讲。一些没有讲到的地方，如果有疑问，你一定要在评论区留言，因为这个专栏我就是要带你攻克开发过程中的所有难题，千万不要让小问题积攒成大难题，那真的得不偿失。我可以承诺的是，留言回复可能会迟到，但绝不会缺席。</p><p>好啦，从现在开始，让我们一起开启这场充满挑战的Go项目实战旅途，为真正开发出一个优秀的企业级Go应用，成为一个Go资深开发者，一起努力吧！</p>
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
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLCrJQ4AZe8VrDkR6IO03V4Tda9WexVT4zZiahBjLSYOnZb1Y49JvD2f70uQwYSMibUMQvib9NmGxEiag/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Dowen Liu</span>
  </div>
  <div class="_2_QraFYR_0">毛大那里学完理论，这里来实践</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-26 22:42:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/7a/d4/aa028773.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张诚</span>
  </div>
  <div class="_2_QraFYR_0">的确目前go语言的网上资料存在水平参差不齐的情况。在学习任何一门语言，在架构设计、编程规范等很多方面非常讲究”最佳实践“。看了下本文学习内容图，不仅仅在开发阶段内容非常详实外，课程非常注重设计和后续部署阶段，特别是从部署阶段可以看出作者践行了作为一个云原生应用开发课程的目的。会保持学习下去。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-26 20:30:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/3c/bd/ea0b59f6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span></span>
  </div>
  <div class="_2_QraFYR_0">如果从练手来看，是否先用GO来写一些业务比较好，如CURD RESTFul之类的服务？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 练手还是找个优秀的开源项目来学习，最高效。比如iam。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-31 23:45:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/e1/0d/ecf81935.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Empty</span>
  </div>
  <div class="_2_QraFYR_0">写了一段时间的Go了，但是总是感觉理解不了Go的设计思想，写不出符合Go风格的代码，期望跟着老师能深入了解Go的设计思想和代码风格，为自己立个Flag吧</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 老哥加油!</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-31 10:07:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/26/97/a5/e52d10bb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>。。。不知道起啥名字</span>
  </div>
  <div class="_2_QraFYR_0">请问一下 这个适合什么程度的程序员看呢 我还在大二，好多看不懂，麻烦了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 有一定经验的。<br><br>不过大学生，只要愿意啃，没有啃不下来的，啃的过程也是学习的过程，而且这个过程学习效率，记忆也是最好的。<br><br>鼓励你学习，中间遇到问题，可以留言区，或者加老师微信支持~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-27 20:18:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/d4/f1/c06aa702.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>惟新</span>
  </div>
  <div class="_2_QraFYR_0">问的问题，可以整理到出来，放到github中。效果会更好</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞👍🏻等github repo star量上来了，可以考虑</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-31 12:51:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/28/8d/c6/9afdffaa.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Andylin</span>
  </div>
  <div class="_2_QraFYR_0">老师，我学了一些go但就是不知道怎么深入，总是感觉还是不会写</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以找一些工作可以使用的项目，拿来尝试做二次开发，一方面工作有输出，另一方面也学到了东西</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-12 15:04:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/2b/80/2535381c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>莫鸣之名</span>
  </div>
  <div class="_2_QraFYR_0">已经30了，一直做数仓开发，看了GO的前景感觉很不错，不知道还能能来得 及转GO。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不晚的，加油</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-04 13:50:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/82/54/b9cd3674.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小可爱(`へ´*)ノ</span>
  </div>
  <div class="_2_QraFYR_0">前年跟着大佬搞了掘金的APIServer，今年保持跟进。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 老相识了，祝你Go技能再上一层</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-11 14:33:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/30/67/a1e9aaba.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Roway</span>
  </div>
  <div class="_2_QraFYR_0">老师，学习这个课程之前，需要具备哪些能力？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 肯学就成了，哪里不会google哪里。<br>如果想学的轻松，可以有一些基本的linux知识，一些go基本语法知识</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-07 16:12:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/fb/7f/746a6f5e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Q</span>
  </div>
  <div class="_2_QraFYR_0">高质量的文章都是作者用时间和汗水浇筑而成的，跟着大佬一起学习！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-27 08:56:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Bright</span>
  </div>
  <div class="_2_QraFYR_0">老师有wx，或者群方便联系</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 看下github iam项目的README</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-08 19:59:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jeffery</span>
  </div>
  <div class="_2_QraFYR_0">果断下单 ！云原生go 必备！跟着大佬go</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Keepgoing!</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-28 20:25:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_83cf81</span>
  </div>
  <div class="_2_QraFYR_0">大佬，其他语言的直接啃这个，ok吗</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-08 11:20:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/53/7e/b6829040.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>SevenMonths</span>
  </div>
  <div class="_2_QraFYR_0">终于找到你，刚转Go那会，跟着掘金的ApiServer，学完搞了个管理系统直接上线使用了。当时就想这作者啥时候在出个更完善点的课呀。哈哈。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 666666</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-15 14:20:58</div>
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
  <div class="_2_QraFYR_0">一起来卷 Go</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-09 21:18:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/1a/f3/41d5ba7d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>iLeGeND</span>
  </div>
  <div class="_2_QraFYR_0">从java程序员的角度看，go语言的语法太丑了，各种特殊符号，写着费劲</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-09 20:15:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/53/a8/73618c92.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>kai楷</span>
  </div>
  <div class="_2_QraFYR_0">突然想转Go了，发现Java现在真的往死里卷了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 转吧，老哥，你不会后悔的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-28 20:13:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/24/db/d6/8d49bb77.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>天狼</span>
  </div>
  <div class="_2_QraFYR_0">打卡第一天！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-25 22:13:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4b/39/d376dfb7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>clong</span>
  </div>
  <div class="_2_QraFYR_0">java也开始搞go</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 大佬，搞起来，有什么问题欢迎留言</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-27 21:16:43</div>
  </div>
</div>
</div>
</li>
</ul>