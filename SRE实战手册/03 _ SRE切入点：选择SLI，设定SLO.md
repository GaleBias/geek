<audio title="03 _ SRE切入点：选择SLI，设定SLO" src="https://static001.geekbang.org/resource/audio/2e/b1/2e8280888c9d2978da98c71cc4956eb1.mp3" controls="controls"></audio> 
<p>你好，我是赵成，欢迎回来。</p><p>还是先来复习下上节课讲的“系统可用性”的两种计算方式，一种是从故障角度出发，以时长维度对系统进行稳定性评估；另一种是从成功请求占比角度出发，以请求维度对系统进行稳定性评估。同时，我们还讲到，在SRE实践中，通常会选择第二种，也就是根据成功请求的比例来衡量稳定性：</p><p><strong>Availability = Successful request / Total request</strong></p><p>SRE强调的稳定性，一般不是看单次请求的成功与否，而是看整体情况，所以我们会把成功请求的占比设定为一个可以接受的目标，也就是我们常说的“3个9”或“4个9”这样的可量化的数字。</p><p>那么，这个“确定成功请求条件，设定达成占比目标”的过程，在SRE中就是<strong>设定稳定性衡量标准的SLI和SLO的过程</strong>。</p><p>具体来看下这两个概念。SLI，Service Level Indicator，服务等级指标，其实就是我们选择哪些指标来衡量我们的稳定性。而SLO，Service Level Objective，服务等级目标，指的就是我们设定的稳定性目标，比如“几个9”这样的目标。</p><p>SLI和SLO这两个概念你一定要牢牢记住，接下来我们会反复讲到它们，因为落地SRE的第一步其实就是“<strong>选择合适的SLI，设定对应的SLO</strong>”。</p><!-- [[[read_end]]] --><p>好，那我们就正式开始今天的内容。我会带你彻底理解SLI和SLO这两个概念，并掌握识别SLI、设定SLO的具体方法。</p><h3>SLI和SLO到底是啥？</h3><p>SLI和SLO这两个概念比较有意思，看字面意思好像就已经很明白了，但是呢，仔细一想，你会发现它们很抽象。SLI和SLO指的到底是啥呢？</p><p>接下来我给你讲一个具体的例子，讲完后，你肯定就能理解了。</p><p>我们以电商交易系统中的一个核心应用“购物车”为例，给它取名叫做trade_cart。trade_cart是以请求维度来衡量稳定性的，也就是说单次请求如果返回的是非5xx的状态码，我们认为该次请求是成功的；如果返回的是5xx状态码，如我们常见的502或503，我们就判断这次请求是失败的。</p><p>但是，这个状态码只能标识单次请求的场景。我们之前讲过，单次的异常与否并不能代表这个应用是否稳定，所以，我们就要看在一个周期内，所有调用次数的成功率是多少，以此来确定它是否稳定。比如我们给这个“状态码返回为非5xx的比例”设定一个目标，如果大于等于99.95%，我们就认为这个应用是稳定的。</p><p>在SRE实践中，我们用SLI和SLO来描述。“状态码为非5xx的比例”就是SLI，“大于等于99.95%”就是SLO。说得更直接一点，SLO是SLI要达成的目标。</p><p>通过这个例子，你现在是不是已经理解了这两个概念呢。SLI就是我们要监控的指标，SLO就是这个指标对应的目标。</p><p>好，那接下来我们要解决的问题就很具体了。我们应该选择哪些指标来监控系统的稳定性？指标选好后，对应地怎么定它的目标呢？下面咱们就一一来探索。</p><h4>系统运行状态指标那么多，哪些适合SLI？</h4><p>我们先来讨论怎么选择SLI。要回答怎么选择这个问题，我们得先来看看都有哪些可供选择的指标。</p><p>在下面这张图中，我列举了系统中常见的监控指标。<br>
<img src="https://static001.geekbang.org/resource/image/e8/9c/e8a49d7fbfb8df42db84efe44f48f59c.jpg?wh=3107*1874" alt=""><br>
这些指标是不是都很熟悉？那该怎么选呢？好像每一个指标都很重要啊！</p><p>确实，这些指标都很重要。我们可以通过问自己两个问题来选择。</p><p>第一个问题：<strong>我要衡量谁的稳定性？</strong> 也就是先找到稳定性的主体。主体确定后，我们继续问第二问题：<strong>这个指标能够标识这个实例是否稳定吗？</strong> 一般来说，这两个问题解决了，SLI指标也就确认了。</p><p>从我的经验来看，给指标分层非常关键。就像上图那样分层后，再看稳定性主体是属于哪一层的，就可以在这一层里选择适合的指标。但是，你要注意，即便都是应用层的，针对具体的主体，这一层的指标也不是每一个都适合。</p><p>根据这几年的实践经验，我总结了选择SLI指标的两大原则。</p><p><strong>原则一：选择能够标识一个主体是否稳定的指标，如果不是这个主体本身的指标，或者不能标识主体稳定性的，就要排除在外。</strong></p><p><strong>原则二：针对电商这类有用户界面的业务系统，优先选择与用户体验强相关或用户可以明显感知的指标。</strong></p><p>还拿我们上面trade_cart的例子来说，主体确定了，就是trade_cart，应用层面的，请求返回状态码和时延就是很好的指标，再来检查下它们能否标识的trade_cart稳定性，毫无疑问，这两个指标都可以，那么请求返回状态码和时延就可以作为trade_cart稳定性的SLI指标。</p><p>我们换一个指标，CPU的使用率这个指标适合吗？根据我们刚才说的原则，既然我们关注的是trade_cart的运行状况，而CPU是系统层的指标，所以，在选择应用层SLI的指标时，自然会把CPU排除掉。</p><p>你可能会说，这样是不是太武断了呀？</p><p>我们简单来分析下。假设CPU使用率达到了95%，但是只要CPU处理能力足够，状态码成功率可能还是保持在4个9，时延还是在80ms以内，用户体验没有受到影响。另外一种情况，假设CPU使用率只有10%，但是可能因为网络超时或中断，导致大量的请求失败，甚至是时延飙升，购物车这个应用的运行状态也不一定是正常的。所以结论就是，CPU使用率不管是10%还是95%，都不能直接反映trade_cart运行是正常还是异常，不适合作为trade_cart这样的应用运行稳定性的SLI指标。</p><p>讲到这里，你可能会问，哎呀，你说的这两个原则我理解了，分层也大概能做到，但是我还是需要做很多详细的分析才能选择出SLI指标，有没有什么更便捷、更快速的方法来帮助我选择啊？</p><p>嗯，不要着急，还真有这样一套方法。怎么选SLI，我们可以直接借鉴Google的方法：VALET。</p><h2>快速识别SLI指标的方法：VALET</h2><p>VALET是5个单词的首字母，分别是Volume、Availability、Latency、Error和Ticket。这5个单词就是我们选择SLI指标的5个维度。我们还是结合trade_cart这个例子，一起看一下每个维度具体是什么。</p><p><strong>Volume-容量</strong></p><p>Volume（容量）是指服务承诺的最大容量是多少。比如，一个应用集群的QPS、TPS、会话数以及连接数等等，如果我们对日常设定一个目标，就是日常的容量SLO，对双11这样的大促设定一个目标，就是大促SLO。对于数据平台，我们要看它的吞吐能力，比如每小时能处理的记录数或任务数。</p><p><strong>Availablity-可用性</strong></p><p>Availablity（可用性）代表服务是否正常。比如，我们前面介绍到的请求调用的非5xx状态码成功率，就可以归于可用性。对于数据平台，我们就看任务的执行成功情况，这个也可以根据不同的任务执行状态码来归类。</p><p><strong>Latency-时延</strong></p><p>Latency（时延）是说响应是否足够快。这是一个会直接影响用户访问体验的指标。对于任务类的作业，我们会看每个任务是否在规定时间内完成了。</p><p>讲到这里，我要延伸下，因为通常对于时延这个指标，我们不会直接做所有请求时延的平均，因为整个时延的分布也符合正态分布，所以通常会以类似“90%请求的时延 &lt;= 80ms，或者95%请求的时延 &lt;=120ms ”这样的方式来设定时延SLO，熟悉数理统计的同学应该知道，这个90%或95%我们称之为置信区间。</p><p>因为不排除很多请求从业务逻辑层面是不成功的，这时业务逻辑的处理时长就会非常短（可能10ms），或者出现404这样的状态码（可能就1ms）。从可用性来讲，这些请求也算成功，但是这样的请求会拉低整个均值。</p><p>同时，也会出现另一种极端情况，就是某几次请求因为各种原因，导致时延高了，到了500ms，但是因为次数所占比例较低，数据被平均掉了，单纯从平均值来看是没有异常的。但是从实际情况看，有少部分用户的体验其实已经非常糟糕了。所以，为了识别出这种情况，我们就要设定不同的置信区间来找出这样的用户占比，有针对性地解决。</p><p><strong>Errors-错误率</strong></p><p>错误率有多少？这里除了5xx之外，我们还可以把4xx列进来，因为前面我们的服务可用性不错，但是从业务和体验角度，4xx太多，用户也是不能接受的。</p><p>或者可以增加一些自定义的状态码，看哪些状态是对业务有损的，比如某些热门商品总是缺货，用户登录验证码总是输入错误，这些虽不是系统错误，但从业务角度来看，对用户的体验影响还是比较大的。</p><p><strong>Tickets-人工介入</strong></p><p>是否需要人工介入？如果一项工作或任务需要人工介入，那说明一定是低效或有问题的。举一个我们常见的场景，数据任务跑失败了，但是无法自动恢复，这时就要人工介入恢复；或者超时了，也需要人工介入，来中断任务、重启拉起来跑等等。</p><p>Tickets的SLO可以想象成它的中文含义：门票。一个周期内，门票数量是固定的，比如每月20张，每次人工介入，就消耗一张，如果消耗完了，还需要人工介入，那就是不达标了。</p><p>这里我给出一个Google提供的，针对类似于我们trade_cart的一个应用服务，基于VALET设计出来的SLO的Dashboard样例，结合上面我们介绍的部分，就一目了然了。<br>
<img src="https://static001.geekbang.org/resource/image/fa/f8/fafcdd765f8a58a42fbc75bfca4fd7f8.png?wh=1440*1034" alt="" title="VALET示例图"><br>
好，VALET我们就讲完了，怎么选SLI指标，你是不是一下子就清楚了。可以说，这是一个我们可以直接复用的工具。上面Google的这张SLO样例图，建议你多看几遍，看到时候，对比思考下自己系统的情况。</p><h2>如何通过SLO计算可用性？</h2><p>到这里，我们已经能够根据自己想要保障稳定性的主体，来选择合适的SLI指标了，也知道SLO就是对应SLI要实现的目标，比如“几个9”。</p><p>但是，我们前面讲到了系统可用性：</p><p><strong>Availability = Successful request / Total request</strong></p><p>然后又深入到了提炼具体的SLI，以及设定对应的SLO，那这两者之间是什么关系呢？也就是通过SLO应该怎么去计算我们的系统可用性的呢？这就涉及到系统整体可用性的两种计算方式。</p><p><strong>第一种，直接根据成功的定义来计算</strong>。<br>
也就是我们前面定义一个请求的返回状态码必须是非5xx才算成功，同时时延还要低于80ms，同时满足这两个条件，我们才算是成功的，也就是纳入上述公式中Successful request的统计中，用公式来表示：</p><p><strong>Successful = （状态码非5xx） &amp; （时延 &lt;= 80ms）</strong></p><p>如果还有其它条件，直接在后面增加做综合判定。</p><p>但是，这种计算方式存在的问题就是，对单次请求的成功与否的判定太过死板，容易错杀误判。比如我们前面讲对于时延，我们一般会设定置信区间，比如90%时延小于等于200ms这样的场景，用这种方式就很难体现出来。而且，对于状态码成功率和时延成功率的容忍度，通常也是也不一样的，通过这种方式就不够准确。所以，我们就会采取第二种方式。</p><p><strong>第二种方式，SLO方式计算</strong>。</p><p>我们前面讲时延时讲过以下几个SLO，这时我们设定稳定性的时候，就需要把公式计算方式灵活调整定义一下了。</p><ul>
<li>SLO1：99.95% 状态码成功率</li>
<li>SLO2：90% Latency &lt;= 80ms</li>
<li>SLO3：99% Latency &lt;= 200ms</li>
</ul><p>直接用公式表示：</p><p><strong>Availability = SLO1 &amp; SLO2 &amp; SLO3</strong></p><p>只有当这个三个SLO同时达标时，整个系统的稳定性才算达标，有一个不达标就不算达标，这样就可以很好地将SLO设定的合理性与最终可用性结合了起来。所以，通常在SRE实践中，我们通常会采用这种设定方式。</p><p>如果是这样，第一种方式是不是就没有用途了呢？当然不是。第一种计算方式也会有它特有的应用场景，它通常会被利用在第三方提供的服务承诺中，也就是SLA（ Service Level Agreement）。因为对于第三方提供商来说，比如云厂商，它们要面对的客户群体是非常大的，所以很难跟每一家客户都去制定像SLO这么细粒度稳定性目标，而且每家客户对稳定性的要求和感知也不同，就没法统一。</p><p>这种情况下，反而是第一种计算方式是相对简单直接的，但是这样也决定了SLA的承诺相比SLO肯定也相对比较宽松，因为SLA是商业服务承诺，如果达不成是要进行赔偿的。关于SLA，最直接的参考资料，就是各个公有云厂商在官网公开的信息资料，你可以找到这些资料，作为自己的一个补充学习。</p><h2>总结</h2><p>讲到这里，怎么选择SLI指标、如何制定SLO目标，我们就介绍完了。你需要掌握下面三个重点。</p><ol>
<li>对系统相关指标要分层，识别出我们要保障稳定性的主体（系统、业务或应用）是什么，然后基于这个主体来选择合适的SLI指标。</li>
<li>不是所有的指标都是适合做SLI指标，它一定要能够直接体现和反映主体的稳定性状态。可以优先选择用户或使用者能感受到的体验类指标，比如时延、可用性、错误率等。</li>
<li>掌握VALET方法，快速选择SLI指标。</li>
</ol><h2>思考题</h2><p>最后，给你留一个思考题。</p><p>下面我给出一个Google的SLI和SLO设定标准示例，内容很直观，需要你认真研究一下这个文档，结合今天我们所讲的内容，请你尝试按照Google提供的规范格式，制定一个自己所负责系统的SLO。</p><p>Google的SLI和SLO设定模板链接：<a href="https://landing.google.com/sre/workbook/chapters/slo-document">https://landing.google.com/sre/workbook/chapters/slo-document</a></p><p>另外，对今天的内容如果你还有什么疑惑，都可以在留言区提问，也欢迎你把今天的内容分享给身边的朋友，和他一起学习讨论。</p><p>我是赵成，我们下节课见。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/05/62/0a4e5831.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>soong</span>
  </div>
  <div class="_2_QraFYR_0">关于SLI和SLO，分析得很清楚！理念上帮助理解，同时还有实践的指引，极具深度！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 非常感谢！也希望与你一起有更深度地交流和探讨。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-28 21:12:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/97/d7/02bbaba5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>飞鸿无痕</span>
  </div>
  <div class="_2_QraFYR_0">关于SLI和SLO的选择和制定写得很好，人工点赞。请教一个践行SRE过程中关于SLI选择的疑惑。对于现在的微服务架构，会有非常多的服务，而且其中包含有很多关键服务模块，比如订单、购物车、商品等等，我们在选择SLI的时候，各个服务都会有对应的错误率和访问延迟，但是反应系统稳定性又是一个综合的体现。请教赵成老师，在微服务架构下，应该如何合理的地选择SLI来反应系统的整体稳定性？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这是一个很好的问题，其实我们接下来的一篇内容就要讲到这个具体的问题，可以耐心等待一下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-25 08:40:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/50/33/43833f7c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Nick</span>
  </div>
  <div class="_2_QraFYR_0">通过SLI和SLO, 是不是只能判断在一个时间范围内是否都达标. 但没法做到像前面的那个时间纬度那样的3条9和4条9这样的表. SaaS的服务承诺好像是用类似时间维度那样的3条9, 4条9这样的故障时间评判表. </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: SLO同样可以做到的3个9或4个9这样的定义的，具体可以看04的Error Budget这部分内容。<br><br>其实，即使是针对时间维度的定义，这里还会有一个问题，就是怎么算可用时长，怎么不可算，最终归根结底还是要落到SLI上面。<br><br>这部分内容比较多，可以花点时间好好消化下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-23 22:01:55</div>
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
  <div class="_2_QraFYR_0">老师分享的标准要翻墙、、、我就基于课程内容谈谈吧，觉得课程中的东西结合之前运维体系课程可以理解缘由。<br>“SLO 是 SLI 要达成的目标。”一个是等级一个是分数；就如同“Availability = SLO1 &amp; SLO2 &amp; SLO3”公式一样；这就如同行业等级测评一样，到达某个等级必须多项条件都只要达到一个分数。<br>浅谈一下DatabaseSystem这块的吧：这块云服务中其实蛮多的，简单谈一下，这个其实是一段时间爱你观察的一个积累吧；<br>失败请求的次数、CPU使用率、内存使用率、IO使用率、数据的增长率，对应的标准其实各个行业有各自的特性，云服务一概而论的评级评分个人一直觉得有不少错，当然这也就是人为监控的价值。<br>如果没有特性，估计去年大会所谓的10%的专业运维都不用了，2-5%-特性创造价值。<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 分享的很好。系统打分是基于系统运行过程中的多个指标综合评判得出的。<br><br>云厂商定的标准我们一般称之为SLA，不一定是错误，但肯定是相对简单和通用一些，因为它是要面向更大范围的客户的，定的太过精细反而不易于执行。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-23 09:38:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_91c9b9</span>
  </div>
  <div class="_2_QraFYR_0">关于Tickets- 人工介入，对于电商网站，“故障恢复”(指的是真实发生了故障)绝不大部分都是需要人工介入和排查的，故障很难自愈，现在的服务也很少能够在故障时自愈，这个方式对于制定指标是不是作用不大？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-06 16:36:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/d5/12/ba7214ce.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lzkfun</span>
  </div>
  <div class="_2_QraFYR_0">系统整体可用性的两种计算方式其实就是: 前者一个粒度教粗, 后者可以将每个粒度细分<br><br>不过&quot;系统整体可用性&quot;的说法好难理解啊, 感觉再说整体稳定目标是否达成, 是在做二元判断, 没法量化.<br><br>感觉能给个可用性的百分值会更好点, 是不是可以对slo可用性加权平均或者相乘下.<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-10 12:57:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/46/5b/07858c33.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Pixar</span>
  </div>
  <div class="_2_QraFYR_0">关于SLO计算系统可用性有个疑问，希望老师可以帮助解答一下。 我们的可用性目标最终设定在几个9上,比如每天的可用性是99.9 %, 通过第一种计算方式(根据成功的定义)可以很方便的计算: (1天内的成功请求数)&#47;(一天内的请求总数).  但是通过SLO1 &amp;SLO2 &amp; SLO3 这种计算方式如何计算出该系统在一天内的可用性呢？ </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 按天算可能留给我们的budget会比较少，建议还是以4周为单位去算。<br><br>多个SLO，如果有一个不达标，其实就是稳定性不达标了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-06 12:22:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/23/bb/a1a61f7c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>GAC·DU</span>
  </div>
  <div class="_2_QraFYR_0">老师人工介入是2票，错误率是2.4%，那么可用率99.3%是怎么算的呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个问题可以描述的详细些吗？我没有理解的很清楚。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-27 08:31:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>oci</span>
  </div>
  <div class="_2_QraFYR_0">分析得清楚sli和slo</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢，能让大家更清晰的理解这两个概念，就是本篇内容的目的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-26 19:42:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/2a/ff/a9d72102.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>BertGeek</span>
  </div>
  <div class="_2_QraFYR_0">可以优先选择用户或使用者能感受到的体验类指标，比如时延、可用性、错误率等。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-04 18:05:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/d0/4d/2116c1a4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Bravery168</span>
  </div>
  <div class="_2_QraFYR_0">指标分层，上下层指标之间往往有联系，如何结合起来分析呢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-09 11:02:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>大都督陈</span>
  </div>
  <div class="_2_QraFYR_0">老师，文章中的链接打不开，请问怎么操作了，谢谢~</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-20 15:47:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/2b/02/7ef138a0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>润豪</span>
  </div>
  <div class="_2_QraFYR_0">Availability = SLO1 &amp; SLO2 &amp; SLO3, 每个 SLO 的监控最后汇总成一个监控吗? 一般用什么系统进行监控呢?</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-31 20:03:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/11/4b/fa64f061.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>xfan</span>
  </div>
  <div class="_2_QraFYR_0">老师，请问如果我在一段时间里，去压测，去故意破坏，比如我重启一台服务器，那这个时候，我怎么来制定SLI呢，是不是可以定义为 在对外侧没有影响服务就行吗，因为CPU，LOAD，内存使用率在这个时候都没有什么意义</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-15 16:40:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/de/15/a12e71a4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>MrKuma</span>
  </div>
  <div class="_2_QraFYR_0">第二种计算方法SLO1&amp;SLO2&amp;SLO3 这是按位与计算还是什么意思？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-16 10:49:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/b0/8a/3ecf6853.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Browser</span>
  </div>
  <div class="_2_QraFYR_0">量化的很好，有收获</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不量化，就无法表达。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-19 22:37:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/97/d7/02bbaba5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>飞鸿无痕</span>
  </div>
  <div class="_2_QraFYR_0">在指定SLO的时候，可用性比较好评估，还有一个指标是服务的RT，一个服务一般会提供多个接口，对于其中某一个或者某几个接口RT比较长的常见(比如：一个服务80%的接口返回都在30ms，但是有几个接口返回是300ms左右，针对RT的SLO要怎么设置才比较合理?)，这种应该如何来确定SLO呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 建议分接口评估，而且要关注关键核心接口。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-09 14:11:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/a6/c0/ee6ea3a9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>James-东方</span>
  </div>
  <div class="_2_QraFYR_0">真的太棒了！内容我感觉就是SRE的BIBLE</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 哇，这评价太高了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-14 12:48:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>思维决定未来</span>
  </div>
  <div class="_2_QraFYR_0">有个问题，在计算SLO的时候，时间怎么选择？是每天计算一次，还是每小时，或者每秒？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-16 16:00:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/f1/43/bd24ec72.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小子</span>
  </div>
  <div class="_2_QraFYR_0">可用性这块，有点没理解，还请老师指点，Availablity（可用性）代表服务是否正常，这句话，我只能理解，服务负载不高，能正常接收请求，即使4xx也归结为正常，但是在延伸点，400能代表这个服务正常么？最近就碰到开发上线导致400了，就是有点纠结这个定义，应当怎样正确理解呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-07 14:50:23</div>
  </div>
</div>
</div>
</li>
</ul>