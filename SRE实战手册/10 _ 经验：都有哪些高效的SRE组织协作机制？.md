<audio title="10 _ 经验：都有哪些高效的SRE组织协作机制？" src="https://static001.geekbang.org/resource/audio/52/9a/5299653fa0e072f8a90e393c32adcc9a.mp3" controls="controls"></audio> 
<p>你好，我是赵成，欢迎回来。</p><p>上节课，我们讲了在互联网企业典型的SRE团队中，一般包含几个关键角色，PE、工具开发和稳定性开发，他们分别承担的职责就是业务运维、建设运维自动化平台和建设稳定性平台。在建设这些平台的过程中，这些角色对内要与中间件团队合作，基于微服务和分布式的架构来开发各类平台；对外，还要与业务开发配合，将提升效率和稳定性的能力提供出去。</p><p>但是仅仅有组织架构，有了队形还是不够的，各个团队和角色之间必须要配合协作起来才能发挥出SRE的作用，特别是对外与业务开发的合作，这样才能是一个有机的整体。</p><p>那怎样才能将这些角色有效地组合到一起呢？今天我就和你分享下我的经验，总结起来就是四个字：<strong>以赛带练</strong>。</p><h2>什么是“以赛带练”？</h2><p>“以赛代练”这个术语最早也是在一些体育赛事中提出来的，完整解释是通过比赛<strong>带动</strong>训练。比如足球、篮球或田径等比赛，目的就是让运动员在真实的高强度竞争态势和压力下，充分暴露出个人和团队的不足及薄弱点，然后在后续的训练中有针对性地改进，这样对于比赛成绩的提升有很大帮助，而不是循规蹈矩地做机械性训练。你也注意到，我用的是“带”，而不是“代”，所以整个过程不是用比赛代替训练，它更主要的作用是带动。</p><!-- [[[read_end]]] --><p>同样，这样的策略放在我们的系统稳定性建设中也非常适用。你也可以选择类似的真实且具备高压效果的场景，来充分暴露我们的稳定性存在哪些问题和薄弱点，然后有针对性地进行改进。</p><p>类似的场景，比如说极端的海量用户访问，像电商类产品的双十一大促、社交类产品在春晚的抢红包，以及媒体类产品的突发热点新闻等等，用户瞬时访问的流量是极大的，对于系统的稳定性冲击和挑战也就非常大。再或者，是极端的故障场景，比如机房断电、存储故障或核心部件失效等等。</p><p>所以，我们讲稳定性建设，一定也是针对极端场景下的稳定性建设。</p><p>不知道你有没有发现，其实我们有很多的稳定性保障技术和理念，比如容量规划和压测、全链路跟踪、服务治理、同城双活甚至是异地多活等，基本都是在这些场景下催生出来的，甚至是被倒逼出来的。</p><p>如果针对这样的极端场景，我们都可以从容应对，那你可以想象一下，日常一般的问题或小故障处理起来自然也就不在话下了。而且，目前各大电商网站的大促已经基本日常化，每月都会有，甚至每周每天都会有不同的促销活动，所以在这种密集程度非常高的大促节奏下，“以赛带练”的效果就会更加突出。</p><p>可以看到，“以赛带练”的目的就是要检验我们的系统稳定性状况到底如何，我们的系统稳定性还有哪些薄弱点。</p><p>那么，如果你在自己的团队来落地“以赛带练”的话，<strong>一定要先考虑自己的业务系统应对的极端场景到底是什么，然后基于这些场景去设计和规划。</strong></p><h2>SRE的各个角色如何协作？</h2><p>赛场有了，要打好这场仗，我们得保证上赛场的选手们能够通力协作。对于我们想要达成的稳定性目标来说，就是要有一套高效的SRE协作机制。接下来我还是以电商大促这个场景为例，跟你分享我们SRE团队中各个角色的分工，以及可以通过哪些组织形式把各个角色组织起来。</p><p>想要把电商大促这场赛事做漂亮，保障准备工作至关重要。这个准备工作前后共有四个关键步骤，我们就一步一步来看。</p><p><strong>第一步，大促项目开工会</strong>。</p><p>在这个会议上，我们会明确业务指标，指定大促项目的技术保障负责人，通常由经验丰富的业务技术成员或平台技术成员承担，同时会明确技术团队的分工以及各个团队的接口人，然后根据大促日期，倒排全链路压测计划。分工和计划敲定了，接下来就是一步步执行了。</p><p><strong>第二步，业务指标分解及用户模型分析评审会</strong>。</p><p>业务指标分解和用户模型分析阶段，需要业务开发和PE团队共同配合，主要是共同分析出核心链路，同时PE要分析链路上的应用日常运行情况，特别是QPS水位，这里就要利用到我们前面03讲中示例的SLO的Dashboard，结合这两点，大致判断出要扩容的资源需求。</p><p>这里你可能会有疑问，为什么业务开发和PE要一同分析？因为每个PE是要负责一整条核心链路上的应用的，所以PE可以识别出更全局的链路以及整条链路上的容量水位情况。而每位业务开发往往只专注在某几个应用上，所以他可以把业务目标拆解得更细致，并转化成QPS和TPS容量要求，然后分配到一个个具体应用上，并且每个应用的Owner在后面要确保这些应用能够满足各自的QPS和TPS容量要求。</p><p>从这个过程中，你会发现，<strong>PE会更多地从全局角度关注线上真实的运行状态</strong>，而业务开发则根据这些信息做更细致的分析，甚至是深入到代码层面的分析。</p><p>同时，业务开发和PE在分析的时候，就要使用到稳定性开发团队提供的全链路跟踪和监控平台了，而不是靠手工提取日志来做统计分析。</p><p>接下来，根据容量评估的结果，由PE准备扩容资源，然后业务开发的应用Owner，通过运维自动化平台，做完全自动化的应用部署和服务上线即可，整个过程无需PE介入。</p><p><strong>第三步，应急预案评审会</strong>。</p><p>在预案准备阶段，仍然是PE与业务开发配合，针对核心链路和核心应用做有针对性的预案讨论。这时就要细化到接口和方法上，看是否准备好限流、降级和熔断策略，策略有了还要讨论具体的限流值是多少、降级和熔断的具体条件是怎样的，最后这些配置值和配置策略都要落到对应的稳定性配置中心管理起来。</p><p>这里<strong>PE更多<strong><strong>地</strong></strong>是负责平台级的策略</strong>，比如，如果出现流量激增，首先要做的就是在接入层Nginx做限流，并发QPS值要设定为多少合适；再比如缓存失效是否要降级到数据库，如果降级到数据库，必然要降低访问QPS，降低多少合适，等等。</p><p>而<strong>业务开发则要更多<strong><strong>地</strong></strong>考虑具体业务逻辑上的策略</strong>，比如商品评论应用故障，是否可以直接降级不显示；首页或详情页这样的核心页面，是否在故障时可以完全实现静态化等等。</p><p><strong>第四步，容量压测及复盘会</strong>。</p><p>在容量压测这个阶段，就需要PE、业务开发和稳定性平台的同学来配合了。业务开发在容量规划平台上构造压测数据和压测模型，稳定性平台的同学在过程中要给予配合支持，如果有临时性的平台或工具需求，还要尽快开发实现。</p><p>压测过程会分为单机、单链路和全链路几个环节，PE和业务开发，在过程中结合峰值数据做相应的扩容。如果是性能有问题，业务开发还要做代码和架构上的优化，同时，双方还要验证前面提到的服务治理手段是否生效。</p><p>压测过程中和每次压测结束后，都要不断地总结和复盘，然后再压测验证、扩容和调优，直至容量和预案全部验证通过。这个过程一般要持续2～3轮，时间周期上要3～4周左右。整个过程就会要求三个角色必须要非常紧密地配合才可以。</p><p>到这里，整个保障准备过程就结束了。你可以先思考下，这个过程中每个角色发挥的作用有哪些。接下来，我跟你一起提炼总结下。</p><p>第一，PE更加关注线上整体的运行状态，所以视角会更全局一些，业务开发则更关注自己负责的具体应用，以及深入到代码层面的分析工作。</p><p>第二，PE会主要负责平台级的公共部件，如缓存、消息和文件等；DBA负责数据库，所起到的作用与PE相同，他们关注这些平台部件的容量评估、服务治理策略，以及应急预案等；而业务开发则会把更多的注意力放在业务层面的容量评估和各类策略上。同时，PE和业务开发关注的内容是相互依赖的，他们的经验也有非常大的互补性和依赖性，所以这个过程，双方必须要紧密配合。</p><p>第三，这个过程中，你可能会发现，工具开发和稳定性开发的同学在其中参与并不多，这是因为他们的价值更多体现在平时。我们依赖的各类平台和工具，比如扩缩容、链路分析、测试数据制造、压测流量的模拟等能力，都是通过这两个团队在平时开发出来的。对于这两个团队来说，这个时候出现得越少，他们的价值才是越大的。</p><p>讲到这里，对于SRE体系中，每个角色要起到的作用，以及他们之间的协作机制就讲完了，各方职责不同，但是彼此互补和依赖，只有密切合作才能发挥出SRE体系的力量。</p><h2>赛场上的哪些工作可以例行化？</h2><p>谈完了“以赛带练”这种大促场景下的协作机制，那没有大促的时候，SRE组织中的各个角色平时应该要做些什么呢？</p><p>其实，我们前面提到的“以赛带练”的事情，会有一部分转化为例行工作，同时还会增加一些周期性的工作。总结起来就是以下两项主要工作。</p><p><strong>第一项，核心应用变更及新上线业务的稳定性评审工作</strong>。这里就包括前面讲到的容量评估和压测、预案策略是否完备等工作。PE会跟业务开发一起评估变动的影响，比如变动的业务逻辑会不会导致性能影响，进而影响容量；对于新增加的接口或逻辑，是否要做限流、降级和熔断等服务治理策略，如果评估出来是必需的，那上线前一定要把这些策略完善好；同时在测试环境上还要做验证，上线后要关注SLO是否发生变化等。</p><p><strong>第二项，周期性技术运营工作</strong>。这些就包括了我们要例行关注错误预算的消耗情况，每周或每月输出系统整体运行报表，并召集业务开发一起开评审会，对变化较大或有风险的SLO重点评估，进而确认是否要采取改进措施或暂停变更，以此来驱动业务开发关注和提升稳定性。</p><p>这里的技术运营工作，是PE职责非常大的转变，因为随着各类平台的完善，很多原来依赖运维完成的事情，业务开发完全可以依赖平台自主完成，无需运维介入。比如代码发布、配置变更，甚至是资源申请。</p><p>所以，这时我们会更强调PE从全局角度关注系统，除了稳定性，还可以关注资源消耗的成本数据，在稳定和效率都有保证的前提下，也能够做到成本的不断优化。这样也会使得PE从原有繁琐的事务中抽离出来，能够去做更有价值的事情。关于这一点，也是给运维同学一个转型和提升的建议。</p><h2>总结</h2><p>好了，我们做下本节课的总结。</p><p>我们借助大促这样的场景，通过“以赛带练”的思路来驱动稳定性体系的建设和提升。这个过程中，PE更加关注全局和系统层面的稳定性，并提供各类生产环境的运行数据；而业务开发则会更关注业务代码和逻辑层面的稳定性，与PE互补且相互依赖；而稳定性和工具开发提供平台和工具支撑，保障每一个环节更加高效地完成。最后，我们还会将这些工作例行化，转化成日常的稳定性保障事项。</p><h2>思考题</h2><p>最后，给你留一个思考题。</p><p>对于“以赛带练”的思路，除了今天我们介绍到的，在你实际的工作中还有哪些场景可以尝试？</p><p>欢迎你在留言区分享，或者提出你对本节课程的任何疑问，也欢迎你把本篇文章分享给你身边的朋友。</p><p>我是赵成，我们下节课见。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/9e/d3/abb7bfe3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>wholly</span>
  </div>
  <div class="_2_QraFYR_0">平时以赛带练的场景有很多，除了系统集成测试方的压力测试、可靠性测试、性能专项测试外，还经常做一些局点演示及故障模拟训练，这些都是一些快速暴露问题和提取改进点的有效方式，持续提升系统稳定性。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 局部的演练测试也是一种有效策略。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-08 09:17:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/eb/ef/fc2d102c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>老张</span>
  </div>
  <div class="_2_QraFYR_0">我目前就职的公司是电商类企业，以赛带练最符合的场景就是每年的大促了。我自己本身是性能测试，但通过几次生产全链路压测，发现我的工作内容和老师本节课程的中所提到的PE角色很类似。无论是前期准备阶段的链路梳理强弱依赖，数据预埋模型分析，还是整个压测节奏的把控，全局进度跟进和风险分析，压测复盘，以及大促当天的值班指挥，甚至是某部分预案的决策，都会参与其中。通过很多的压测和复盘，自己也模糊看到了性能测试工程师的职场发展方向，是向稳定性保障的角色靠拢，通过学习老师的课程，也买了《Google sre运维解密》书来看，发现PE或者说类似SRE的角色，是很适合的前进方向。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-29 23:20:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_f35d52</span>
  </div>
  <div class="_2_QraFYR_0">日常运维中发现有一个问题，特别是微服务化后，每次变更发布，开发如果有调整了接口的调用，而又没有及时更新相关资料的话，会直接影响运维的效率。在现在的敏捷开发中，代码迭代很快的，这样的话，实际中是怎么处理的呢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-01 14:56:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/a1/e6/50da1b2d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>旭东(Frank)</span>
  </div>
  <div class="_2_QraFYR_0">PE和业务开发分工并没这么明确</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-20 05:23:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/0c/30/bb4bfe9d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lyonger</span>
  </div>
  <div class="_2_QraFYR_0">我认为SRE的协作机制主要有2种，一种是对外，另一种是对内。对外的协作机制更多的是依托业务场景制订的、双方达成一致的协作规范规范，比如发布规范，配置管理规范，服务变更规范，容量规划规范等。对内的话，业务运维，工具开发，稳定性开发需有一套可正常流转的SOP，确保团队成员的协作规范和需求跟踪全流程，内外兼修，方可游刃有余。😄</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 非常棒！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-09 09:36:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/ee/28/c04a0c83.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小炭</span>
  </div>
  <div class="_2_QraFYR_0">学习经常搞的模拟考试就是“以赛带练”的思路吧</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 也是一种，但是要相对正式的那种才行</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-08 14:47:29</div>
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
  <div class="_2_QraFYR_0">赵老师，你好<br>	技术中台 到 业务中台 之间的运维岗位被称为 应用运维，业务运维，技术运营；为会会有这三种不同称谓的岗位呢？他们应该是有不同侧重点的吧？能否详细介绍下差异</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-31 10:49:46</div>
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
  <div class="_2_QraFYR_0">&quot;以赛代练&quot;个人觉得这种方式应当是源自体育：尤其是一些主要的比赛，我们会发现有各种各样的比赛；真正的比赛可能就是联赛，不同情况下还会有各种舍取。<br>电商平台中不少典型的可能会在双11之类的大型促销出现的策略，我们已经能够在平时的小型促销中去看到；合适的策略就是在生产环境的某些场景中可以利用平时业务量不大降低资源投入，非真正核心交易环节，平时降低资源投入测试一下其实风险并不大，只要随时能跟上就行。例如：快递有时我们会无法知道中间到哪儿了，不过发货方或收件方对于途中到哪儿其实并不关心，确定发出以及到了收获城市会有提醒就好。<br>      记得之前听DevOps课程中就有提及过，&quot;用重要而非绝对核心的业务去测试，循序渐进。&quot;谢谢老师今天的分享，期待后续课程的分享。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-08 06:51:15</div>
  </div>
</div>
</div>
</li>
</ul>