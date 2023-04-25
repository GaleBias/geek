<audio title="09｜案例：互联网典型的SRE组织架构是怎样的？" src="https://static001.geekbang.org/resource/audio/65/36/65433a6881c8eba5a01cc52ad4f87936.mp3" controls="controls"></audio> 
<p>你好，我是赵成，欢迎回来。</p><p>前面三讲，我们从故障这个关键事件入手，讲解了“优先恢复业务是最高优先级”这个原则，基于这个原则，在故障发生后，我们要做好快速响应和应急，并从故障中学习和改进。在这个学习过程中，你应该也能体会到，高效的故障应对和管理工作，其实是需要整个技术团队共同参与和投入的。这就引出了大家落地SRE都会遇到的一个难点：组织架构调整。</p><p>那落地SRE必须调整组织架构吗？典型的SRE组织架构是怎样的？接下来，我会用两讲内容和你探讨这些问题，分享我在蘑菇街实践的一些经验。</p><h2>落地SRE必须调整组织架构吗？</h2><p>好，那我们就开始吧，先给你看一张技术架构图。<br>
<img src="https://static001.geekbang.org/resource/image/69/ac/69a12388ac0795a84bcdc8489bb196ac.jpg?wh=3107*1874" alt=""><br>
这是蘑菇街基于微服务和分布式技术的High-Level的架构图，也是非常典型的互联网技术架构图，自下而上共四层，分别是基础设施层、业务&amp;技术中台层、业务前台层以及接入层，在右侧还有一个技术保障体系。如果你平时经常看一些架构方面的图书和文章，或者听过一些技术大会演讲的话，对这样的图应该不陌生。</p><p>你也许会问，咦，我们不是讲组织架构吗？咋一上来就说到技术架构上了？别急，我这么讲是有原因的，在讲SRE的组织架构之前，我们需要先明确两点内容。</p><p><strong>第一，组织架构要与技术架构相匹配</strong>。</p><!-- [[[read_end]]] --><p>技术架构实现组织目标，组织架构服务并促成技术架构的实现，所以，我们不会单纯去讲组织架构，一定会结合着技术架构的现状和演进过程来分析。</p><p><strong>第二，SRE是微服务和分布式架构的产物</strong>。</p><p>SRE这个岗位，或者说这个通过最佳实践提炼出来的方法论，就是在Google这样一个全球最大的应用分布式技术的公司产生出来的。</p><p>正是因为分布式技术的复杂性，特别是在运维层面的超高复杂性，才产生了对传统运维模式和理念的冲击和变革。</p><p>如果我们去梳理一下整个软件架构发展的历程，就可以得到下面这张图。我们会发现不仅仅是SRE和DevOps，就连容器相关的技术，持续交付相关的方法和理念，也是在微服务架构不断流行的趋势下所产生的，它们的产生就是为了解决在这种架构下运维复杂度过高的问题。</p><p>这样一套架构方法体系，也构成了现在非常流行和火热的概念：云原生架构。<br>
<img src="https://static001.geekbang.org/resource/image/c0/b8/c00aeaab24ac3cf7cdc518701a33d5b8.jpg?wh=3005*1829" alt=""><br>
所以，不得不承认，这里的现实情况就是，基本所有的SRE经验都是基于微服务和分布式架构的，也都是在这样一个基础下产生的。大到BAT、头条和美团等，中等规模如蘑菇街，甚至是在传统行业中落地比较突出的，如部分运营商和银行。</p><p>那么，<strong>想要引入SRE体系，并做对应的组织架构调整，首先要看我们的技术架构是不是在朝着服务化和分布式的方向演进</strong>。如果架构还没这么复杂，其实也没有必要引入SRE这么复杂的运维体系，这本身就不匹配；再就是如果没有对应的架构支持，SRE技术层面的建设就没有切入点，想做也没法做。</p><p>总的来说，如果我们的技术架构朝着微服务和分布式的方向演进，那我们可以考虑落地SRE，这时候，我们的组织架构就要匹配我们的技术架构，也就是说，<strong>要想在组织内引入并落地SRE这套体系，原有技术团队的组织架构，或者至少是协作模式必须要做出一些变革才可以</strong>。</p><p>那下面我就以蘑菇街的技术架构为例，带你一起来看，在这样一个技术架构体系下，SRE的角色、职责分工以及协作模式应该是怎么样的。</p><h2>蘑菇街的SRE组织架构实践</h2><p>我们回到在开头给出的技术架构图，可以看到上下左右总共分了五块区域，我们就分块来看。</p><p>先看最下面的<strong>基础设施层</strong>，我们现在也定义为IaaS层，主要是以资源为主，包括IDC、服务器、虚拟机、存储以及网络等部分。</p><p>这里基础设施层和所需的运维能力，其实就是我们常说的<strong>传统运维这个角色所要具备的能力</strong>。但是这一层现在对于绝大多数的公司，无论在资源层面还是在能力层面，都有很大的可替代性。如果能够依托云的能力，不管是公有云还是私有云，这一部分传统运维的能力在绝大部分公司，基本就不需要了。</p><p>接下来往上，我们来看<strong>中台</strong>这一层。这里包括两部分，<strong>技术中台</strong>和<strong>业务中台</strong>。</p><p>技术中台主要包括我们使用到的各种分布式部件，如缓存、消息、数据库、对象存储以及大数据等产品，这一层最大的特点就是“有状态” ，也就是要存储数据的产品。</p><p>业务中台层，就是将具有业务共性的产品能力提炼出来，比如用户、商品、交易、支付、风控以及优惠等等，上面支撑的是业务前台应用。</p><p>什么是<strong>业务前台</strong>呢？如果以阿里的产品体系来举例，可以类比为淘宝、天猫、盒马、聚划算这样的业务产品。</p><p>无论这些业务前台的形态是什么样，但是都会利用到中台这些共享能力。这两层就是微服务化形态的业务应用了，这些应用最大的特点就是“无状态”，有状态的数据都会沉淀到刚才提到的技术中台的产品中。</p><p>最上面是<strong>接入层</strong>，分为四层负载均衡和七层负载均衡。前者因为跟业务逻辑不相关，所以通常会跟基础设施层放在一起，由传统运维负责。但是七层负载需要做很多业务层面的规则配置，所以通常会跟中台和前台一起运维，那这部分职责应该属于哪个角色呢？我们接下来就会讲到。</p><p>中台及前台的运维职责是怎么分工的呢？</p><p>技术中台的很多部件相对比较标准和通用，而且在公有云上也相对比较成熟了，比如数据库和缓存，对于绝大部分公司是可以直接选择对应的公有云产品的，比如蘑菇街，基本都已经将这些能力迁移到了云上。</p><p>在没有上云之前，我们的中间件团队会自研对应的技术产品，这部分产品的运维也会由中间件团队自运维。很多大型的公司会有专门的平台运维团队，负责整个中间件产品的运维。</p><p>业务中台和前台这两层的运维职责，通常就是我们常说的<strong>应用运维、PE</strong>（Production Engineer）或者叫<strong>技术运营</strong>这样的角色来承担。在我自己的团队是统一用PE来代表的。</p><p>其实这里PE的职责跟我们前面讲的SRE的职责已经非常接近了。在国内，PE这个角色与Google定义的SRE所具备的能力，最大差别就在于国内PE的软件工程能力有所缺失或相对较弱。这就导致很多基于技术中台的自动化工具、服务治理以及稳定性保障类的平台没办法自己研发，需要由另外一个团队来支撑和弥补，也就是图中技术中台的衍生部分，技术保障体系。</p><p>PE这个角色，是我们未来引入SRE实践的非常关键一环，PE要跟业务开发团队一起对业务系统的稳定性负责。</p><p>那我们接着看技术保障体系。</p><p><strong>技术保障平台</strong>，这部分的能力一定基于技术中台的能力之上生长出来的，是技术中台的一部分，如果脱离了这个架构体系，这个技术保障体系是没有任何意义的。</p><p>所以这里我们又要强调一个理念：“<strong>运维能力一定是整个技术架构能力的体现，而不是单纯的运维的运维能力体现</strong>”。微服务和分布式架构下的运维能力，一定是跟整个架构体系不分家的。</p><p>它们的具体依赖关系，我在图中已经标示出来，你可以结合我给出的图示再体会和深入理解一下。</p><p>回到技术保障体系的建设上，我们看下架构图的右侧，它又分为效率和稳定两块。</p><p><strong>工具平台团队</strong>，负责效能工具的研发，比如实现CMDB、运维自动化、持续交付流水线以及部分技术运营报表的实现，为基础运维和应用运维提供效率平台支持。</p><p>这个要更多地介入到研发流程中，因为流程复杂度比较高，而且还要对接很多研发平台，如Git、Maven、代码扫描、自动化测试以及安全等平台，所以对业务理解及系统集成能力要比较强，但是技术能力要求相对就没那么高。</p><p><strong>稳定性平台团队</strong>，负责稳定性保障相关的标准和平台，比如监控、服务治理相关的限流降级、全链路跟踪、容量压测和规划。我们会看到这个团队主要是为稳定性保障提供支撑，平台提供出来的能力是可以直接支撑业务开发团队的，反倒是PE这样的角色并不会直接使用。</p><p>这个团队和人员的技术能力要求会比较高，因为这里面很多的技术点是要深入到代码底层的，比如Java的字节码或Socket网络；有时还要面对海量数据，以及低时延实时计算的处理，比如全链路跟踪和监控；甚至是门槛更高的AIOps，还要懂专业算法。所以这个团队的建设复杂度会比较高，会需要很多不同领域的专业人员。</p><p>好了，到这里，我们从技术架构入手，层层剖析，分析了对应的人员安排和能力，你会发现，按照分层进行职责分工就是：基础设施和接入层的4层负载部分属于传统运维；技术中台如果自研，也就是我们常说的中间件团队，开发同学自己负责对应的工作，如果上云，则由PE负责；业务中台、业务前台以及接入层的7层负责均衡，同样是由PE来负责。</p><p>如果我们用一张组织架构图来展示的话，基本形态就是下图：<br>
<img src="https://static001.geekbang.org/resource/image/84/85/842f14ae8168a7ee1703b5ba26a55f85.jpg?wh=3107*1874" alt=""></p><h2>总结</h2><p>给出这张组织架构图，我们今天的内容也就讲完了。总结一下，从组织架构的角度来讲，对于稳定性或者说SRE我们可以推导出一个结论：SRE并不是一个单纯的岗位定义，它是由多个不同角色组合而成的一个团队。如果从分工来看就是：</p><p><strong>SRE = PE + 工具平台开发 + 稳定性平台开发</strong></p><p>同时，这里的SRE，跟我们前面讲的运维和架构不分家一样，在组织架构上，是与承担技术中台或分布式架构建设的中间件团队在同一个体系中的。</p><p>根据我平时的交流情况看，很多的传统行业也基本是按照这样一个模式组建SRE体系，一方面会有向互联网借鉴的原因，另一方面，我觉得也是本质的原因，就是前面我们提到的，组织架构往往是与技术架构相匹配的，技术上如果朝着分布式和微服务架构的方向演进，那必然会产生出类似的组织模式。</p><p>讲到这里，一个典型的互联网式的SRE组织架构就介绍完了，但是仅有组织架构是不够的，还需要继续深入下去，看看这个组织下的每个角色是如何与外部协作，如何发挥稳定性保障的具体职能的。所以，我们下一讲就会结合具体的稳定性保障场景，来分享SRE与其他组织的协作模式是怎么样的。</p><h2>思考题</h2><p>最后，给你留一个思考题。</p><p>学习完本节课，你觉得在一个组织进行SRE体系建设和变革过程中，哪个角色最为关键呢？同时，哪个角色是跟你最相关的呢？</p><p>欢迎你在留言区分享，或者提出你对本节课程的任何疑问，也欢迎你把本篇文章分享给你身边的朋友。</p><p>我是赵成，我们下节课见。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/0c/30/bb4bfe9d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lyonger</span>
  </div>
  <div class="_2_QraFYR_0">从一个公司的组织架构，往往能看出公司的技术架构。另外很期待老师分享一些云原生领域在SRE下的实践。😄</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这是个很好的议题，我们也正在实践，我会先记下，后面有一定的积累了，我分享出来。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-07 11:09:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/c3/d1/bdf895bf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>penng</span>
  </div>
  <div class="_2_QraFYR_0">我觉得PE这个角色很重要，需要整理业务需求和反馈，沉淀到平台工具开发团队和稳定性开发团队。又需要和业务团队沟通交流，来适配技术中台。虽然他不是一手需求设计者，也不是具体技术中台功能开发者。但是确是关键的核心枢纽。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你提到的枢纽这个定义，这个定位很关键，理解透彻。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-07 10:39:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/17/27/ec30d30a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Jxin</span>
  </div>
  <div class="_2_QraFYR_0">1.我认为是效能和稳定性工具平台的开发。<br>2.在所有角色中没有重要的轻重，因为我们要的是保证系统稳定运行这一最终目标，所以，哪怕是一根稻草的重量，那也是同等的重要.<br>3.但是，搭建整套完善的sre是一个长期的工程，安排好优先级可以让团队和公司在在搭建的过程中获得更高的效益，毕竟效益这个东西不是快照，而是一个时间上的积累。<br>4.所以，我认为优先做效能和稳定性平台的开发会比较好，因为他能比较快的拿出东西，也没有那么多因地制宜的限制，在sre推进的前期，既可以降低公司的顾虑，也可以提高团队的士气。是个开刀的好口子。至于iaas和paas这两块，能用云先用云。而业务这块，这是一件任重道远的事，不适合做先头。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 非常棒的分享和理解！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-07 00:27:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/34/df/64e3d533.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>leslie</span>
  </div>
  <div class="_2_QraFYR_0">个人觉得技术运营或者说去年GOPS大会提及的运维运营：这个其实是个中间层，熟悉产品、懂得且能处理技术相关的问题、能针对现状提出问题；个人可能对此还有更深层次的整体设计的事情。<br>站在中间且能看到两头大致梳理好两头其实这是任何事情发展到中期必然要面对的问题：中台的概念个人觉得其实同样基于此。就像我们说起系统运维，这个其实同样有两头硬件设备和网络，系统只是中间而已，如果硬件方面和网络方面没有处理好系统、、、<br>谢谢老师的分享尤其是云原生架构图引发的反思，期待后续的课程。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-06 09:59:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/9e/d3/abb7bfe3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>wholly</span>
  </div>
  <div class="_2_QraFYR_0">SRE角色不仅仅是定位某个问题，还应该具备更全的技术栈及系统思维，当业务发展到一定规模的时候，我觉得这个角色还是很有必要的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 相当有必要，不可或缺。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-07 08:45:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/7a/e1/326f83b9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>石头</span>
  </div>
  <div class="_2_QraFYR_0">国内传统行业的sre组织架构是什么样了的，有必要设置sre岗位吗？<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 传统行业，做的比较优秀的，比如部分运营商的SRE组织架构其实跟我分享的是差不多的。有很多还没做到这个程度，需要时间和经验的积累。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-06 15:16:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/7b/2b/97e4d599.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Podman</span>
  </div>
  <div class="_2_QraFYR_0">“运维能力一定是整个技术架构能力的体现，而不是单纯的运维的运维能力体现。微服务和分布式架构下的运维能力，一定是跟整个架构体系不分家的。”<br>也是运维崛起的契机<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-12 00:01:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/3d/f5/5c7d550f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小蓝罐</span>
  </div>
  <div class="_2_QraFYR_0">老师，反之我更多的认为SRE其实合适任何大小项目架构的。只要你需要可靠性这样的高要求那就能行。大的架构有大的做法，小的也同理。SRE一样适配</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-11 11:05:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/0f/d5/73ebd489.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>于加硕</span>
  </div>
  <div class="_2_QraFYR_0">工具平台团队，可以理解成DevOps吗？或部分能力</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-29 20:19:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/a2/4f/0d983a3e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Quinn</span>
  </div>
  <div class="_2_QraFYR_0">云原生架构的图中，从左到右是什么轴？为什么敏捷要和虚拟机一起？我的理解是敏捷和devops是相辅相成。devops只是在工具上支持敏捷而已。希望细致讲解一下这张图的用意。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这张图的重点在我们标注的第三列，我们主要是想说明，微服务、分布式、devops、sre、容器等等，这些概念并不是独立存在的，它们之间是有关联关系的。<br><br>其它几列，只是类比同一个阶段内，这些技术是同时出现的，代表了一个时代的技术特点，但是关系可能没有像第三列这么紧密。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-05 02:10:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83epy83Wich4cvDO6dvYbC9DVONqluiaTKn5ATuTmY3HQuXWlAcx2k52FE8TUcjq7Q08f6NwYwdlNnj0A/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_80674c</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，<br>请问对于在大厂私有云里做基础架构层的运维，也就是你文中指的传统运维，有什么职业发展的建议吗？文中对这个职位介绍比较少，大厂里基础架构部人数也不小，所以希望可以指导职业发展。<br>是向PE转型比较好？<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 建议可以往上走，也就是做PE或业务运维，再往上，可以尝试运维开发等等。<br><br>这里我的理解，个人的转型，一方面要靠自己，另一方面，所在的组织也有责任，要提供这样的轮转通道才可以，不然个人努力，但是组织不给机会，就会变成一厢情愿或者死循环。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-27 23:08:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/0f/d5/73ebd489.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>于加硕</span>
  </div>
  <div class="_2_QraFYR_0">个人认为业务运维较重要吧，在我们的环境中，业务运维承载了业务方需求，与应用的维护，运维产品的推广，这种持续性的工作是可以为SRE产出MVP提供输入的。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-21 13:37:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/bf/aa/32a6449c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>蒋悦</span>
  </div>
  <div class="_2_QraFYR_0">看了这个组织架构，我都不好意思说我是哪家公司的，传统行业，差距太大</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 适合的才是最好的，互联网公司可以这样做是因为没有太多的历史包袱，传统行业更重要的是确定好方向，循序渐进。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-08 22:02:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/bf/aa/32a6449c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>蒋悦</span>
  </div>
  <div class="_2_QraFYR_0">请问，什么规模的公司可以选择上云，什么规模的公司可以考虑自建云呢？<br>我们现在的公司其实规模不小，是自建的云，是否有可能上云呢？是否有可能分批上云呢？比如先让前台上云，成功了，在把中台上云？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 理论上，无论规模大小都可以考虑上云。比如阿里和腾讯内部的系统也都全部上云了，只不过上的是自家的云。<br><br>自建的话，至少得像美团、苏宁这样的体量吧，我觉得市面上95%以上的公司都没有必要自建。<br><br>分批上云的思路是ok的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-08 22:00:51</div>
  </div>
</div>
</div>
</li>
</ul>