<audio title="06 _ 故障发现：如何建设On-Call机制？" src="https://static001.geekbang.org/resource/audio/63/8f/634329a384810be3071d1a3b3fe2c38f.mp3" controls="controls"></audio> 
<p>你好，我是赵成，从今天开始，我们进入课程实践篇的内容。</p><p>在上一部分，我们学习了SRE的基础，需要掌握的重点是SLI和SLO以及Error Budget（错误预算）策略。SLI是我们选择的衡量系统稳定性的指标，SLO是每个指标对应的目标，而我们又经常把SLO转化为错误预算，因为错误预算的形式更加直观。转化后，我们要做的稳定性提升和保障工作，其实就是想办法不要把错误预算消耗完，或者不能把错误预算快速大量地消耗掉。</p><p>这么说，主要是针对两种情况：一种是我们制定的错误预算在周期还没有结束之前就消耗完了，这肯定就意味着稳定性目标达不成了；另一种情况是错误预算在单次问题中被消耗过多，这时候我们也要把这样的问题定性为故障。</p><p>今天，我们就来聊一聊，为了最大程度地避免错误预算被消耗，当我们定义一个问题为故障时，我们应该采取什么措施。</p><h2>聚焦MTTR，故障处理的关键环节</h2><p>好了，我们先回顾下在第1讲的时候，提到故障处理的环节就是MTTR，它又细分为4个指标：MTTI、MTTK、MTTF和MTTV，之所以分组分块，也是为了更加有目的性地做到系统稳定性。</p><p><img src="https://static001.geekbang.org/resource/image/3d/dd/3dd910e354da003e234b0340b68e76dd.jpg?wh=1940*1166" alt=""></p><p>那么，这四个环节，在我们故障处理MTTR又各自占多长时间呢？下面这个MTTR的时长分布图，是IBM在做了大量的统计分析之后给出的。<br>
<img src="https://static001.geekbang.org/resource/image/e5/fd/e5b28b0bed414afe8feda4f67b21a6fd.jpg?wh=1884*1837" alt=""><br>
我们可以看到，MTTK部分，也就是故障定位部分的时长占比最大。这一点，我们应该都会有一些共鸣，就是绝大部分的故障，往往只要能定位出问题出在哪儿了，一般都可以快速地解决故障，慢就慢在不知道问题出在哪儿，所以说我们大部分时间都花在寻找问题上了。</p><!-- [[[read_end]]] --><p>不过，从我实际分析的情况看，很多时候MTTR的时间分布跟这个图会有一些差别，为什么呢？</p><p>因为上面这张图是IBM针对网络设施的故障统计出来的，鉴于网络设备部署模式相对简单，以及日志和报错信息比较固定和明确，所以出现问题时，MTTI的判定阶段不会花太多时间。而且真的出现问题，基本上采取的策略一般也是定位到根因，再采取措施处理故障，除了主备切换等冗余措施，基本是没有太多其它手段。</p><p>但是，在一个分布式的软件系统下，我们判定一个问题发生在哪儿，影响范围到底是怎么样的，要召集哪些人共同参与定位排查等等，这个反而会消耗更多时间，所以我们看到MTTI阶段占比会更重。</p><p>另外，当一个分布式系统发生故障时，我们的策略一定不是找到根因，而是优先恢复业务。所以，我们往往是通过故障隔离的方式，先快速恢复业务，也就是我们经常听到的Design for Failure这样的策略，再具体一点，就是服务限流、降级、熔断甚至是主备切换这样的手段，这样的话，MTTK的占比会有所下降。</p><p>因此，从分布式系统的实际情况来看，整个MTTR的时间占比分布大致是这样的：<br>
<img src="https://static001.geekbang.org/resource/image/8e/ac/8e55074c8ec8e2c741ebeac7f7ae80ac.jpg?wh=1887*1815" alt=""><br>
其实，不管是不是分布式系统，我们针对处理故障的目的都是比较明确的，就是要提升每个环节的处理效率，缩短它们的处理时间，这样才会缩短整个MTTR，减少对不可用时长的消耗。</p><p>今天，我们就先来看MTTR的第一个环节，MTTI。看看在MTTI这个环节我们可以采取哪些措施来提高处理故障效率。关于MTTR中的另外三个环节，也就是MTTK、MTTF和MTTV我们会在后面的课程中继续讨论。</p><h2>MTTI：从发现故障到响应故障</h2><p>MTTI，就是故障的平均确认时间，也就是从故障实际发生时间点，到触发采取实际行动去定位前的这段时间。</p><p>这个环节其实主要有两件事情要做：<strong>第一件事，判断出现的问题是不是故障；第二件事，确定由谁来响应和召集</strong>。我们一一来看。</p><p>先看第一件事。我们平时遇到的问题很多，但是这些问题的影响不见得都很严重，或者都需要我们立即做出响应去处理。即便是故障，我们也会分不同的响应级别去处理。那我们怎么判断出现的问题是不是故障呢？如果是故障，我们应该以哪种级别去响应呢？</p><p>解决方案很明确，就是利用我们在<a href="https://time.geekbang.org/column/article/215649">《04 | 错误预算：达成稳定性目标的共识机制》</a>中讲过的，根据错误预算制定故障等级的方式来判定响应方式。我们把trade_cart购物车的案例再拿出来，你可以再看下，作为参考。<br>
<img src="https://static001.geekbang.org/resource/image/cc/c6/cc0987ffd5d5d9391bc58eb505c6ecc6.jpg?wh=3108*1875" alt=""><br>
在没有SLO和错误预算这个体系时，我们通常会根据用户投诉，或客服对同一问题的重复反馈来判断。比如10分钟内，有超过50个用户投诉无法购买商品、支付不成功或者优惠券未生效等问题，我们就会启动应急响应。不过，一旦等用户和客服反馈，就说明故障影响已经非常恶劣了，用户也已经处于无法忍受的状态了。</p><p>显然这种方式并不高效。既然我们已经搭建了SRE体系，设定了SLO，能对稳定性进行量化管理，那我们就能比用户更快发现问题并响应问题了。至于用户投诉和客服反馈的渠道，只能是作为我们的辅助手段。</p><p>所以，我们可以根据设定的SLO和错误预算，准确判断发生的问题是否是故障，并依据故障等级决定我们采取什么样的措施去处理这些问题，大大提高反应效率。</p><p>接着来看第二件事，谁来响应和召集。这件事很容易理解，如果我们判定这个问题就是一个故障，首先要有人能判定，接下来需要哪些人来介入，并能够把相关的人员召集起来，一起来处理故障。</p><p>这两件事，其实就是SRE里面提到的<strong>On-Call机制</strong>。</p><p>我们可以看到，第一件事其实主要依赖于我们的监控和告警体系。这里我想再强调一下，在On-Call阶段，并不是所有的告警我们都要响应，如果不影响稳定性，一般的告警是可以降低响应级别的，我们只需要响应基于SLO的告警。</p><p>我和很多公司交流过，发现大家都非常重视监控体系的建设，监控指标梳理得也非常全面，而且各种监控技术和产品都会使用。比如，从最早的Zabbix，以及日志系统栈ELK，再到近两年随着Kubernetes而火爆起来的Prometheus等等，从技术上来说，这一点不是太大的问题。但是，第二件事往往容易被我们忽视，也就是On-Call的流程机制建设。</p><h2>关于On-Call的两个案例</h2><p>在正式讲流程机制之前，我先分享两个关于On-Call的案例。</p><p>第一个案例是发生在我自己团队里的，当时遇到一次大数据产品HBase故障。那是一个周六的下午，负责数据平台的运维同学收到了故障告警，出现了部分节点不可用，同时依赖该产品的搜索和推荐等业务也开始反馈异常，进而影响到广告曝光，导致资金损失。</p><p>问题出现后，我们很快明确了故障的严重程度，开始联系对应的负责人上线一起处理。因为是周末，所以需要一个个打电话联系，同时我们的HBase已经上云，所以处理过程又需要云产品的同事介入，这就又涉及到跨组织的协调。云产品内部又有分工，一线服务、二线产品研发支持，所以这个协调过程又是一个比较长的链条。再加上其他的权限问题，所以等相关人员就位开始真正排查问题的时候，距离发生故障已经有很长一段时间了。</p><p>当时我大致统计了下，仅仅协调人员到位并上线开始处理问题，这个过程就花去了45分钟，而真正解决问题的环节只用了15分钟。</p><p>第二个例子是国内第一大企业IM产品。因为产品本身的特点，每天早上八点半到九点，会有一波使用高峰，产品经常在这个时间段出故障。同时呢，这个时间段正好是互联网公司通勤时间，大部分技术员工都在上班的路上，根本没办法处理问题，导致故障时间过长。</p><p>所以为了保证及时响应，企业决定分批上下班，一批人早上在家值守，甚至是守在电脑旁随时应急，其他员工准时上班。等过了产品使用高峰期，值守的员工再陆续从家里出发上班。同时，下班也错开时间，保证有人可以及时响应。</p><p>这种错时保障机制，很多电商企业在大促保障或者重大活动期间也经常用，就是<strong>确保关键角色必须在线，随时应急响应</strong>。</p><p>通过以上两个案例，我们发现从整体上讲，如果要缩短MTTI时长，技术相关的监控告警很重要，但更关键的是一整套协作机制。这也是为什么像Google这样技术体系已经如此完善的公司，还要强调On-Call机制的重要性。</p><p>那么，我们怎样才能建设好On-Call 的流程机制呢？</p><h2>On-Call的流程机制建设</h2><p>接下来，我就跟你说说，在蘑菇街我们是怎么做的，我把这个流程总结为“<strong>On-Call关键5步法</strong>”。</p><p>1.<strong>确保关键角色在线。</strong></p><p>这里的关键角色不是指一两个值班的运维或SRE人员，而是整个产品体系中的所有关键角色。比如电商就需要安排核心业务应用（如用户、商品、交易、优惠及支付等）的Owner，或Backup中至少有一人参与On-Call轮班。不过，接收故障告警或第一时间响应故障的，一般是运维、PE或SRE这样的角色，业务开发同事要确保及时响应SRE发起的故障应急。</p><p>2.<strong>组织War Room应急组织。</strong></p><p>我们有专门处理故障的“消防群”（暗含着灭火的意思），会将这些关键角色纳入其中，当有故障发生时会第一时间在消防群通报，这时对应的On-Call同事就要第一时间最高优先级响应呼叫（Page）。如果是工作日，对于严重故障，会立刻组织成立War Room，责任人会马上聚到一起；如果是非工作时间，则会通过电话会议的方式拉起虚拟War Room。</p><p>3.<strong>建立合理的呼叫方式。</strong></p><p>在On-Call的落地过程中，我们经常会遇到的一种情况，就是谁最熟悉某个系统，谁就最容易被7*24小时打扰。比如系统或应用的Owner或者是架构师，出现问题的时候一定是找这些同事处理的效率最高，所以就会造成这些同事被默认为On-Call。</p><p>但是这样做会极大地影响这些同事在正常业务开发上的精力投入。他们要么总是被打断，要么就是经常通宵处理问题，导致第二天无法正常工作，甚至在非工作日也得不到正常的休息。</p><p>这种情况下，我们建议使用固定的On-Call手机，建立手机与所有On-Call系统的对应关系，比如这个手机号码对应交易核心应用，另一个号码对应IDC机房运维等等。这样我们在On-Call时就不再找具体的哪个人，而是手机在谁手中，谁就承担On-Call职责。除非问题迟迟得不到解决，或者遇到了疑难杂症，这种时候再呼叫其他同事参与进来。</p><p>无论是SRE、架构师，还是一线开发，<strong>熟悉某个系统的最快最好的方式就是参与On-Call，而不是看架构图和代码。</strong>所以，有效的On-Call机制，可以让团队更深刻地认识到目前系统的存在哪些问题，对自己的痛苦状态也会有更深刻的感受，进而对后面的改进措施才能更有针对性和落地性。同时，On-Call也可以培养和锻炼新人和Backup角色，这也是让新人尽快融入团队和承担责任的最好的方式。</p><p>4.<strong>确保资源投入的升级机制。</strong></p><p>这个跟前面几条有相关性，有很多团队认为On-Call就是设置几个人值班，所有的事情都交给这几个人做；最极端的还会认为所有的事情，都应该是冲在最前线的运维或SRE来完成。但是，在分布式架构这种复杂场景下，这种思路是明显不可行的。</p><p>这里最核心的一点就是要给到运维和SRE授权，当他发现问题不是自己或现有On-Call人员能解决的时候，他有权调动其他必要资源投入。如果故障等级偏高，一下无法明确具体找哪些人员投入的时候，SRE或运维可以直接上升到自己的主管或相关主管那里，要求上级主管协调资源投入。必要时，还可以上升到更高级主管，甚至CTO或技术VP来关注。所以，授权非常关键，这一点的具体操作层面，我们会在后面的故障处理过程中再详细介绍。</p><p>5.<strong>与云厂商联合的On-Call。</strong></p><p>现在企业上云是大势所趋，绝大多数情况下，我们对问题和故障的处理，是离不开与云产品工程师一起高效联动的。所以，我们应该把云产品和云厂商作为我们系统的一部分，而不是单纯的第三方。而对于云厂商来说，也要考虑把客户业务作为自身业务的一部分，只有这样双方才能紧密合作。我们应该提前做好与云厂商之间的协作磨合，联合故障演练，必要的授权等等。</p><h2>总结</h2><p>最后，我们做一下简单的总结。</p><p>我们按照MTTR的细化分段可以看到，处理故障的第一个环节就是MTTI，也就是故障发现阶段。这个阶段，我们要靠基于SLO的告警，更加精准地告知我们当前系统的稳定性出现的问题，并根据对SLO的影响程度，也就是错误预算的消耗情况作出判断，是否定性成故障。如果是故障，我们就要启动应急响应，而高效快速的应急响应，是要靠On-Call的流程机制来保证的。</p><p>关于建设On-Call的流程机制，我给你分享了我自己团队的“On-Call关键5步法”，咱们再一起复习一下：</p><ol>
<li>确保关键角色在线；</li>
<li>组织War Room应急组织；</li>
<li>建立合理的呼叫方式；</li>
<li>确保资源投入的升级机制；</li>
<li>与云的联合On-Call。</li>
</ol><p>当我们能够很好地做到以上两点的时候，我们就能大幅降低MTTI，也为接下来更高效的故障定位和恢复打下了坚实的基础。</p><h2>思考题</h2><p>最后，给你留一个思考题。</p><p>学完本节课的内容后，你觉得在故障处理中，是优先建设监控体系更重要，还是建设高效的On-Call机制更重要？它们两者应该怎么来结合会更有效？欢迎你在留言区分享你的思考。</p><p>如果你在On-Call方面也积累了一些经验，请分享在留言区，也欢迎你把今天的内容分享给身边的朋友。</p><p>我是赵成，我们下节课见。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/86/06/72b01bb7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>美美</span>
  </div>
  <div class="_2_QraFYR_0">谷歌SRE应急时间处理策略有一条是：由万（全）能工程师解决生产问题 向 手持“运维手册”的经过演练的on-call工程师 演进，核心思路是建设故障处理SOP，保证SRE可以处理大多数故障。这个思路是否和确保关键角色在线冲突！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: “由万（全）能工程师解决生产问题 向 手持“运维手册”的经过演练的on-call工程师 演进，核心思路是建设故障处理SOP，保证SRE可以处理大多数故障”<br><br>感谢你的分享，直接道出了我们oncall应该追求的目标，对于我们内容是非常好的补充。<br><br>内容里我们提到的关键角色在线，就是指on-call工程师必须在值守过程中准实时在线，目的是为了及时响应，这个是机制上的保障，然后在处理问题时，需要有故障处理SOP，并且能按照SOP执行，这个是能力上的要求。<br><br>所以，两者不矛盾。再次感谢你的补充，非常棒。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-31 01:08:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/05/62/0a4e5831.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>soong</span>
  </div>
  <div class="_2_QraFYR_0">监控体系解决的问题是给我们提供一个视角，更快发现或感知到问题发生。On-Call机制想要解决的在于真正去处理、解决问题的部分，关注点是有效性与机制建设本身。<br>没有监控系统的支撑，感知到故障发生所需的时间就要很久；没有高效的On-Call机制，处理并解决故障问题的时间也会被拉长，老师文中也有举例！从重要性的层面来看，两者是相互促进、相互支撑的作用。<br>从公司的发展过程来看，我个人认为，先建设一个能有效运行的监控系统，随后跟进On-Call机制的建设，是一个可行的路线！先建设On-Call机制，如果缺乏有效的监控系统，从提升系统的稳定性来看，针对性似显不足。希望看到老师的观点！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 前面两段讲清楚了，监控和on-call机制的关系，很赞。<br><br>最后的问题，其实我的建议是，先要有on-call机制，因为问题反馈的渠道不一定只要监控，可能还有用户投诉、同事反馈等等，当遇到这些问题时，也需要有高效的响应机制。而且在早期，有可能监控并不准确，反而是其它渠道的反馈占比更高。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-04 21:22:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/22/75/79693c63.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>EQLT</span>
  </div>
  <div class="_2_QraFYR_0">ON-call机制重要，毕竟监控体系是持续优化进步的过程，而业务的故障是随时发生，优先恢复业务是最高优先级。良好的on-call机制既能保障业务，也能将处理过的故障快速反馈到监控体系，进一步优化监控体系，达到双赢。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 思考的很全面奥</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-30 09:25:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/fb/b2/57cf8247.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>moqi</span>
  </div>
  <div class="_2_QraFYR_0">在某某公司的运维小伙和我提起过，他们那的系统出了问题，SRE的第一件事是写故障报告，第二件事是解决问题，坑爹的是解决问题的时候还得不停的回复boss们的询问，更坑爹的是其他的SRE没有任何的协作机制，看他一人在那忙死，没有丝毫的互备机制，老大们似乎也不觉着这是个巨大的问题。<br><br>有再强大的监控体系，但没有协同作战的意识，这样团队里的成员哪来的团队荣誉感<br><br>这套On-call响应机制很棒，应该是团队不停的磨合和共创出来的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 很好的关于感受的分享。<br><br>特别是第一段，应该是我们要极力避免的状态和情况。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-30 23:57:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/03/8d/38a98dc6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>牧野静风</span>
  </div>
  <div class="_2_QraFYR_0">对于中小型企业的我们来说，运维能做的就是做好各种监控体系，尽可能在用户反馈问题之前监控到故障，进行恢复。我们的做法是，每个项目有个Leader，出问题运维，开发，DBA协同处理，小团队这样配合解决问题还是挺快的。但是以前遇到过凌晨出现问题，各种人员Call不上，所以我们上线尽量放在周一到周四</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 变更是“万恶之源”，所以人员不容易聚集的情况下，尽量避免变更是合理的策略。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-07 14:58:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/60/90/521a6bcf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>daniel＿yc</span>
  </div>
  <div class="_2_QraFYR_0">从事一线运维工作6年了，最大的感受就是能力强的能被累死，我们行业比较特殊，基本上每个大的客户现场都有人驻守，加上客户自己的保障人员。有完整的on-call流程，奇葩的是，出现问题，第一反应是找能力强的人来解决，而当时当班的人只是用来传话的。。这就导致能力强的人被累死，基本上没周末没假期。。。。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我遇到过很多这样的情况，其实这不是个人的问题了，这个是公司或机制的问题，应该要能轮转起来，并且一定周期内得对故障或问题的发生有容忍度才可以。另外，对于发生的问题或故障，要有分级才可以，不然啥问题过来都要立即马上处理，这种对oncall的同事来说，压力过大。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-04 19:44:00</div>
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
  <div class="_2_QraFYR_0">熟悉某个系统的最快最好的方式就是参与 On-Call，而不是看架构图和代码。<br><br>这块感觉应该是精通系统细节</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 在炮火中磨炼，会成长的更快，对于细节的了解也会更深入。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-03 06:57:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/d0/51/f1c9ae2d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Christopher</span>
  </div>
  <div class="_2_QraFYR_0">第一次听赵成老师的声音，还挺有磁性的，哈哈</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢！<br><br>其实内容也很有料，是吗？^_^</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-30 09:57:59</div>
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
  <div class="_2_QraFYR_0">看完老师的课程，很想从开发转岗到SRE，感觉很刺激😂</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不经历磨难，怎么见光明。不一定要转岗，但是开发过程中要多跟SRE交流学习，时间长了，你就是具备SRE意识和能力的开发。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-30 08:52:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKJrOl63enWXCRxN0SoucliclBme0qrRb19ATrWIOIvibKIz8UAuVgicBMibIVUznerHnjotI4dm6ibODA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Helios</span>
  </div>
  <div class="_2_QraFYR_0">on—call和监控，我感觉是鸡生蛋蛋生鸡的问题。首先oncall不能没有监控作为基础，要不然只能靠人工反馈了，单纯有监控没有oncall不能及时解决问题。<br><br>监控对事故复盘有很好的作用，能完善oncall的指标，oncall反过来也能促进监控的发展，又是相辅相成的关系。<br><br>但是如果这两个都没有的情况下，先建设哪一个。<br>这个时候业务量不大，可以先建设监控，然后根据客服用户反馈解决事故，通过事故复盘增加oncall机制。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 非常好的分享，最终我们要学会两条腿走路，这才是最关键的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-04 08:04:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/61/50/f5cc9f43.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>huadongjin</span>
  </div>
  <div class="_2_QraFYR_0">MTTR的流程说的真好，在日常的稳定性工作中对这块有概念，但一直没有抽象出来，听赵老师你一讲，顿时豁然开朗。也给的工作规划起到了指导作用，那就是如何去缩短MTTI和MTTK的时长，提高故障修复效率，进而减少故障持续时长。我准备从告警和全网变更入手，这两项是故障的狼烟和故障的推手，在有疑似故障或故障定位中，拉取近一段时间的变更事件，包括安全、系统、网络、发布、数据库、中间件等变更类型，去协助定位故障。相信对他们的掌握有利于站在更高的角度看问题。请问这样的思路可行嘛，赵老师。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 变更日志的记录和检索，这个思路是没有问题的，但是因为每个系统和体系的日志不一定规范或者能够汇总全面，甚至是有些操作是业务层面的，所以，现实中不一定可行。<br><br>更可行的方式是，针对一些关键部件的变更进行收集，比如软件发布、网络变更、数据库变更等等。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-03 08:20:52</div>
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
  <div class="_2_QraFYR_0">关于On Call流程机制的建设说到了痛点，往往就是系统出问题了，不能迅速的联系对应的责任人</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个是通病，也是非常容易被忽视掉的部分，而忽视的原因，是因为跟技术没有太大关系。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-30 21:47:17</div>
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
  <div class="_2_QraFYR_0">老师提及了错峰上班，可能我所经历的某些企业就是直接的翻班，这是国内IDC机房普遍使用的策略。必要时候这种翻班是从部门经理层一直到一线员工，集体翻班或者延长工作时间去保障其稳定性。<br>监控的体系再怎么做其实都是后知后觉，就像SRE中说及&quot;没有故障是特殊现象“，有问题有故障才是常态，故而On-call的机制是第一时间处理问题；二者又是相互的。只有在解决完处理完成后才能进一步去改善监控体系。<br>谢谢老师今天的分享：期待后续的课程。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 很好的分享。<br><br>因为IDC机房是基础设施，太关键了，很多银行和金融企业也是24小时现场值班，因为经不起半点疏忽。<br><br>监控和on-call在一段时间内是要互补的，监控跟不上，on-call机制来补，随着业务体量的增大，监控的重要性会越来越大，也要同步建设起来。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-30 20:05:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/36/b3/6f/a57c81f4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>hanson</span>
  </div>
  <div class="_2_QraFYR_0">Google 工作手册中提到监控、告警是SRE必要的两大核心原则，可见建设监控体系更重要，同时提倡系统自动化机制。在On-Call 方面投入的也是有考虑时间和成本付出的，所以Google的 On-Call 文化注重个人的健康。<br>在SRE方面不是付出健全的情况情况下，既重视建设监控系统，同时On-Call 的投入成本也是非常高的。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-19 06:58:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/0c/86/8e52afb8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>花花大脸猫</span>
  </div>
  <div class="_2_QraFYR_0">建设监控体系更加重要！但是对于监控体系中出现的 问题，如何快速的去定位解决，需要on-call机制及时的介入</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-24 11:01:43</div>
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
  <div class="_2_QraFYR_0">又来回顾了一下，赵老师咨询个问题，MTTR的各项时间占比给的这个图是有代表性的吗？能否告知从哪里统计的结果 ，另外蘑菇街在MTTR各阶段的时间大概是多少，能否告知下呢？我们做个参考。谢谢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-24 15:52:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/60/05/3797d774.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>forever</span>
  </div>
  <div class="_2_QraFYR_0">应该优先建立on-call机制，因为应用已经上线了，出了故障就需要找到一个关键工程师进行响应，监控体系可以根据on-call的一些历史数据进行针对性建设，这样未来监控体系也会更加实用</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-23 17:41:59</div>
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
  <div class="_2_QraFYR_0">我们之前还有专门的NOC团队，专门负责五步法中的故障监控、消息分发、问题上升及拉群和后续的组织复盘工作。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-13 21:05:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/f4/89/86a8220a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>牧野</span>
  </div>
  <div class="_2_QraFYR_0">hello 赵老师，最近领了公司的回滚方案任务，一头雾水，今天看到老师的讲解有了一点想法，不知道是否正确：<br>MTTR：MTTI =&gt; MTTK =&gt; MTTF =&gt; MTTV<br>在可回滚的前提下：回滚将执行流程优化为：MTTI =&gt; MTTV，消除MTTK，MTTF时间消耗(预计有50%)<br>这个可以理解为做回滚的出发点吗?</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-01 17:26:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/38/2d/9c971119.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>若丶相依</span>
  </div>
  <div class="_2_QraFYR_0">On-Call 更优先高一些，监控体系可以后期补。<br>On-Call 表示有人正在处理问题，监控只是更快的定位问题。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 讲的非常好，现实中，技术体系的建设需要一个周期，所以流程机制上要更先建设起来</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-12 21:35:32</div>
  </div>
</div>
</div>
</li>
</ul>