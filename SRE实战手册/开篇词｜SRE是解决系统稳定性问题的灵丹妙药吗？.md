<audio title="开篇词｜SRE是解决系统稳定性问题的灵丹妙药吗？" src="https://static001.geekbang.org/resource/audio/8d/c1/8d9c16d7a0c2c7bdad59e93197f0c7c1.mp3" controls="controls"></audio> 
<p>你好，我是赵成，欢迎加入我的课程，和我一起学习SRE。</p><p>为了加强彼此的了解，我先做个简单的自我介绍吧。我在基础架构和运维领域工作有10多年了，现在负责蘑菇街平台技术部，主导中间件、稳定性、工具平台、运维和安全等工作。</p><p>2017年底，我在极客时间开了一门课程，叫<a href="https://time.geekbang.org/column/intro/100003401">《赵成的运维体系管理课》</a>，系统整理并分享了我在运维和DevOps方面的经验，带给你不一样的运维思考。</p><p>这两年，近距离地接触了很多不同类型、不同规模的企业IT团队，我发现他们为了<strong>提升用户价值的交付效率</strong>，都在积极采用微服务、容器，以及其他的分布式技术和产品，而且也在积极引入像DevOps这样的先进理念。</p><p>这些公司选择了正确的架构演进方向和交付理念，效率自然是提升了一大截。这样的情况，是不是也发生在你的公司、发生在你自己身上？这时候你会发现，效率提升了，但挑战紧跟着也来了：在引入了这么多先进的技术和理念之后，<strong>这种复杂架构的系统稳定性很难得到保障，怎么办？</strong></p><p>这个问题其实不难回答，答案就是SRE。</p><p>这几年业界对SRE的关注越来越多，大家也几乎达成了共识，Google SRE就是目前稳定性领域的最佳实践。也可以说，SRE已经成为稳定性的代名词。</p><p>SRE这么厉害，是有什么神奇之处吗？其实，SRE要做的事情并不神秘，我们每天做的监控告警、运维自动化、故障处理和复盘等工作，就是SRE的一部分。你也会发现，Google在介绍SRE的时候，很多篇幅介绍的就是这些我们熟悉的内容。</p><!-- [[[read_end]]] --><p>最近两年，我和团队也花了很多精力来做稳定性保障方面的事情，不断地探索在SRE方面的实践。比如，在日常的稳定性规范制定，监控、压测、服务治理、大促稳定性保障，故障应急和管理，以及组织架构建设等方面，我们都做了尝试，也积累了很多经验。</p><p>2019年6月，在SRE领域最具国际影响力的SRECon上，我分享了蘑菇街在容量压测方面的实践经验，和来自全球各大公司的同行们做了一次深度交流，让他们也见识了国内电商大促稳定性保障的技术实力。</p><p>从这些经验和交流探讨中，我收获了一条宝贵的经验：<strong>要想系统地做好稳定性这件事儿，SRE就是必修科目</strong>。</p><p>同时，我也深刻体会到落地SRE会遇到各种问题，深知大家对SRE的困惑。所以，我系统梳理了自己的经验和调研，打磨出这个课程，目的就是帮你<strong>构建起体系化建设SRE的思路。</strong></p><h2>标杆立在那里，落地SRE有哪些问题？</h2><p>那SRE在落地时具体会有哪些问题呢？接下来，我先说个我经常遇到的场景吧。</p><p>我在外部参加会议演讲或参与交流的时候，经常会有一些朋友向我求助，这里不乏一些公司的CTO、技术VP、总监或架构师，让我帮忙推荐运维或SRE专家。</p><p>每次遇到这样的情况，我都会问，你们现在遇到了什么问题，对这样的专家有什么要求。他们就会告诉我当前他们团队遇到的一些状况，比如系统三天两头出问题，有时候遇到一些问题，一排查就要老半天，特别是引入了微服务架构后，问题好像更多了。为了解决这些问题，开发和运维都要投入很多精力，结果却不尽如人意：系统不稳定会被业务团队投诉，好，那就赶快处理问题，但是这时候需求来了，响应不及时，业务团队又会不满意。事后，还要为了谁承担责任推来推去，对团队氛围影响很大。</p><p>这种人困马乏却谁都不满意的情况多了，我们就特别希望能找到一招制胜的办法。</p><p>这时候，SRE就被当作了灵丹妙药。因此，他们希望我能介绍一些这样的专家，来了就能把这样的问题干脆利落地统统解决掉。</p><p>说实话，每次遇到这样的问题，我都非常犯难。因为我发现我身边这样的大牛或专家也非常稀缺，还真不好推荐。另外，也是更重要的一点，从根本上来说，这绝不是招一两个或几个专家就能解决的问题。</p><p>那，为什么大家还总是向我提出推荐SRE专家这样的求助呢？很明显，这是大家对SRE的理解出现了偏差。很多人想当然地认为，SRE就是一个岗位，是一个角色，而且是无所不能的角色。</p><p>当然，这只是其中的问题之一。在实际落地SRE时，我们要么是不知道该从何入手，要么就是开始了却总会掉进这样那样的坑里，遇到各种问题和疑惑。</p><p>我将大家遇到的问题和疑惑，汇总到了下面这个清单里：</p><p><img src="https://static001.geekbang.org/resource/image/fc/82/fc3095dad7c58acea4ce856f34643182.jpg?wh=3107*1874" alt=""></p><p>清单很长，你看完什么感受？这些问题不是我凭空臆想出来的，而是在跟众多企业IT团队交流和调研的过程中，我被问及最多、最频繁的问题。</p><p>问题虽然多，但总结起来其实就是两大类：</p><ol>
<li>理念：SRE到底是什么？我们应该怎么来理解它？有哪些关键点？</li>
<li>实践：到底应该从哪里入手建设SRE？组织架构应该怎么匹配？</li>
</ol><p>这些问题确实令人头痛，不过也不用害怕，我先给你吃一颗定心丸，这些问题我们都可以解决。</p><p>比如，你想要找到建设SRE体系的切入点，最好的办法就是建立稳定性的标准化。有时你会和周边团队就稳定性问题产生一些争执，说到底就是因为你们没有达成共识的、统一的衡量标准。Google SRE已经给我们提供了很好的标准化手段，也就是SLO。你看，这个问题不就得到解决了吗？</p><p>再比如，组织架构如何建设的问题，虽然Google没有给出放之四海而皆准的答案，但经过多年的实践，很多互联网公司甚至是传统企业，都已经积累了很丰富的经验。借鉴这些经验，建设组织架构的问题也能解决。</p><p>接下来，这个课程就会带你一一攻克这些问题。</p><h2>这门课程是如何设计的？</h2><p>具体来说，整个课程分为两个部分。</p><p><strong>第一部分，夯实基础，带你建立SRE稳定性标准。</strong></p><p>在这一部分，我会先讲清楚SRE是什么，以及业界衡量稳定性的标准是什么。我会把SLO作为引入SRE的切入点，因为它就相当于我们稳定性标准化的基础。同时，SLO也是稳定性保障的共识机制，有了这个共识，我们才能更好地管理稳定性，消除掉来自周边团队的很多不理解和不认可。</p><p>同时，在这一部分我还会引入一个电商的案例，跟你一起看一下，在实际的场景中设定SLO应该考虑哪些因素。</p><p><strong>第二部分，SRE最佳实践</strong>。</p><p>这一部分，我会从“故障”和“组织架构”这两个关键词入手来讲。</p><p>第一个是“故障”。我会围绕故障这个影响稳定性的核心事件，结合实践案例，分享应该从哪些方面减少故障发生次数，缩短故障影响时间，进而提升系统可用性及稳定性。</p><p>第二个是“组织架构”。这是做SRE绕不开的关键问题，要想做好SRE的落地，必须得有与之匹配的组织架构和协作机制。我会结合我的实践经验，以及我了解到的行业经验，让你看到真实的组织架构设置和跨团队协作模式。</p><p><img src="https://static001.geekbang.org/resource/image/7f/f1/7f112814ffb26d07b96c20a094b430f1.jpg?wh=1043*1562" alt=""></p><p>通过这两个维度的学习，从理念到实践，我相信可以系统地解答你心中很多关于SRE的具体疑惑。<strong>如果你想从0到1建设SRE体系，有效地管理好你的系统稳定性，希望有一个合理的组织架构有效应对各种稳定性问题</strong>，那就和我一起学习吧。</p><p>另外，我想和你说说答疑相关的事情。SRE是个非常体系化的内容，我们的课程不会面面俱到。但是没关系，在学习过程中，你可以在留言区大胆提出你的任何疑惑，分享你的思考，我会在留言区答复，和你交流探讨；同时，我也会挑选有代表性的问题，单独成文，有针对性地做补充分享，作为答疑加餐发布出来。</p><p>最后，我还想再多啰嗦几句。答案很重要，但往往并不是最重要的东西，在探寻答案的过程中，我们获得的思路和方法更有意义，因为它们可以帮助我们举一反三，在未来更多的场景中发挥价值。希望接下来我们一起探索SRE的这个过程，也能有这样的价值。</p><p>好了，那咱就正式开始SRE的学习之旅吧！对了，你是怎么看SRE的？目前都有哪些困惑呢？记得留言和我说说你的情况。</p><p>我是赵成，再次欢迎你来到我的课程，我们下一讲见！</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/0f/d5/73ebd489.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>于加硕</span>
  </div>
  <div class="_2_QraFYR_0">一直在看找老师的书，这次就当听书了，看了下精选留言，分享下目前我的认知：DevOps核心是做全栈交付，SRE的核心是稳定性保障，关注业务所有活动，两者的共性是：都使用软件工程解决问题;<br>DevOps的诞生是由于互联网商业市场竞争加剧，企业为减少试错成本，往往仅推出最小可行产品，产品需要不断且高频的迭代来满足市场需求，抢占市场(产品的迭代是关乎一整条交付链的事)，高频的迭代则会促使研发团队使用敏捷模式，敏捷模式下对运维的全栈交付能力要求更严格，则运维必须开启DevOps来实现全栈交付；因为不断的迭代交付(也就是俗称的变更)是触发故障，非稳定性根源，而互联网产品&#47;服务稳定性缺失会造成用户流失，甚至流到竞争对手那里， 因此关注业务稳定性也变得十分重要，SRE由此诞生。希望看完赵老师的课程后对理论能有所提升。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 非常棒，目前为止，看到的最深入，最精彩地解答。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-19 23:15:36</div>
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
  <div class="_2_QraFYR_0">学习过程有点不一样：先学了老师《赵成的运维体系管理课》，然后又把里面相关推荐的书基本都看了数遍，本想找个平台去整合一切，又误打误撞在做数据系统架构和管理；下半年把全栈工程师的课学了一遍，雪峰老师的DevOps学完了且参加了相关的大会去交流学习。<br>自己近十年一直是在数据系统和系统运维之间作为主业：其实之前国内运维大会以及运维圈子的交流中有感受到SRE和DevOps其实相辅相成。极客时间DevOps都出了，SRE不出似乎不合适；课程终于出来了算是等待了数月的课程吧。<br>希望能够在课程学习中把之前老师课程中提及的运维体系和SRE的东西融入其中去更好的理解，真正理解好SRE且用好，觉得没有那么容易。<br>期待数月的课程终于出来了：希望完课时能站在不一样的视角去理解课程以及更好的理解老师之前的《赵成的运维体系管理课》。谢谢老师的分享。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢你的认可。既然已经学习了这么多内容，你可以试着看一下，提出一个你现在遇到的具体问题，因为DevOps也好，运维也好，还是SRE也罢，只是一种方法和思路，但是只有能解决你的问题才会有用。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-18 19:48:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/fe/ff/daf96217.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>null</span>
  </div>
  <div class="_2_QraFYR_0">来字节跳动做SRE 呀， 感兴趣的留言 ;-)</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">编辑回复: 你来给大家做个分享吧要不</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-18 18:24:22</div>
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
  <div class="_2_QraFYR_0">老师，现在的困惑是sre和devops有啥关系呢。感觉有的devops团队会把sre的事情搞了呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不出所料，这个问题大家果然有疑惑，不过期望大家看完接下来的01篇之后，再来思考一下。<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-18 17:45:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/1e/4a/ff855511.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>沈子砚Stella</span>
  </div>
  <div class="_2_QraFYR_0">这是成哥本人的声音吧</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 确认就是本人^_^</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-18 22:10:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/87/62/f99b5b05.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>曙光</span>
  </div>
  <div class="_2_QraFYR_0">做为JAVA工程师，学习SRE会获得什么启发呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 更全面和系统的了解软件架构。<br><br>另外，懂开发的SRE现在可以市场上紧缺的人才奥。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-20 21:45:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTK7WWEtUNCkPUjzXg3cKLGewSmy6NqsnlhEc0UMqST4icydJ0OvhjqpAL8Io46drdXZf3vMFjWlzaw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>初级SRE</span>
  </div>
  <div class="_2_QraFYR_0">SRE由运维而来，但是运维不是全部。<br>作为一个由传统作为转到SRE的人，我希望能够尽快理解差异，不足短板</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 第一句话分享的很有感触。同时，也恭喜你找对了不断和提升的方向，就是SRE。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-18 19:45:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/92/be/8de4e1fe.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>kaizen</span>
  </div>
  <div class="_2_QraFYR_0">目前在一家外企做了半年多SRE了，希望能在这门课上有新的启发</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 期望这么课程对你有帮助，如果有任何具体的问题，欢迎在留言区给我提问<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-18 17:15:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/db/1a/30201f1a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_kevin</span>
  </div>
  <div class="_2_QraFYR_0">非常想了解一下，线上系统变更，在SRE中是如何做的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 理想状态，SRE是可以不用介入其中的，整个发布流程可以由开发自动化的完成，SRE只需要关注系统的SLO是否受到影响即可，关于这部分的内容，我后面会讲到。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-22 23:35:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/55/fd/abb7bfe3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>epic2005</span>
  </div>
  <div class="_2_QraFYR_0">成哥，SRE 如果在云上 有哪些Ops工作要做呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这是个很好的问题，业务上云一般只能解决基础层面服务问题，但是业务和应用还是自己的，这就需要把运维的视角提升到这个层面，要关注业务和应用的稳定性。<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-19 00:12:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/b1/da/88197585.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>甘陵笑笑生</span>
  </div>
  <div class="_2_QraFYR_0">不错，这门课更实战</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 希望能给你带来帮助<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-18 22:33:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/9d/84/171b2221.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jeffery</span>
  </div>
  <div class="_2_QraFYR_0">sre 和aiops 又啥不同、怎么在企业推广sre</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以看接下来的01篇，看看是不是可以解答你的疑问<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-18 19:48:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/57/6e/b6795c44.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>夜空中最亮的星</span>
  </div>
  <div class="_2_QraFYR_0">今年就考老师的这个课了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 期望对你能有所帮助。<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-18 18:42:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/b8/24/039f84a2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>咱是吓大的</span>
  </div>
  <div class="_2_QraFYR_0">重要的不是答案，而是思考的过程</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-20 21:31:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/c3/48/3a739da6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>天草二十六</span>
  </div>
  <div class="_2_QraFYR_0">配合《SRE Google运维解密》这本书来看，开发和运维结合起来，想推它，万事开头难啊</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 事情总是因为难能，而可贵，才会更有价值，当然也蕴藏着大量的机会。所以，一起加油，干起来再说。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-05 16:58:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/b1/33/8993eae0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>黄展志</span>
  </div>
  <div class="_2_QraFYR_0">成哥，你的&lt;&lt;赵成的运维体系管理课&gt;&gt; 这个课程也很nice，正在读第二遍，写得真好，非常感谢，谢谢，订阅成哥的课程</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢你的认可，这门课程有任何的问题，可以给我留言，我们一起交流<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-19 13:38:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/48/4c/29b0d5ae.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Alex_Shen</span>
  </div>
  <div class="_2_QraFYR_0">老师SLA,SLI,SLO这三者有什么区别</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以先接着看下面的课程，我会讲到这三者的内容，如果有疑问可以继续跟我留言提问<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-19 09:10:39</div>
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
  <div class="_2_QraFYR_0">开发人员路过，不懂SRE是什么，希望能学习到了解到</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 先了解学习起来，有问题记得给我留言，我一定会回复，跟大家一起进步<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-18 22:30:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/c0/bd/912f07a5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>馬正偉</span>
  </div>
  <div class="_2_QraFYR_0">传统运维与SRE的本质区别是引入了业务维度的考量，SRE围绕着稳定性，制定一系列规范标准。故障应急响应标准(可通过SLO)，以故障会核心，务必会涉及跨团队协作，组织架构调整。这时候落地适合sre体系的组织架构和跨团队机制特别重要。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-28 21:42:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/21/20/1299e137.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>秋天</span>
  </div>
  <div class="_2_QraFYR_0">目前 在新公司  负责系统稳定性建设  对于这种体系化的东西  思路或者从哪入手 有点困难  希望可以从本专栏中解惑这些问题  </div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-28 16:35:44</div>
  </div>
</div>
</div>
</li>
</ul>