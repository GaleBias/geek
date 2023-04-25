<audio title="01｜SRE迷思：无所不能的角色？还是运维的升级？" src="https://static001.geekbang.org/resource/audio/6c/b4/6c369871a455c580414561aca4f5c9b4.mp3" controls="controls"></audio> 
<p>你好，我是赵成。</p><p>作为这个课程的第一讲，我先从实践的角度，和你聊聊应该怎么理解SRE。</p><p>为什么要强调是实践的角度呢？</p><p>开篇词里我们就提到过，有人认为SRE就是一个岗位，而且是一个具备全栈能力的岗位，只要有这么一个人，他就能解决所有稳定性问题。这还只是一种理解，而且这个理解多是站在管理者的角度。</p><p>也有人站在运维人员的角度，认为做好SRE主要是做好监控，做到快速发现问题、快速找到问题根因；还有人站在平台的角度，认为做好SRE要加强容量规划，学习Google做到完全自动化的弹性伸缩；甚至还有人认为，SRE就是传统运维的升级版，把运维自动化做好就行了。</p><p>你看，其实不同的人站在不同的角度，对SRE的理解就会天差地别，但是好像又都有各自的道理。</p><p>所以，我特别强调实践的角度，我们不站队，就看真实的落地情况。我总结了一下从实践角度看SRE的关键点，就一个词：<strong>体系化</strong>。<strong>SRE是一套体系化的方法，我们也只有用全局视角才能更透彻地理解它。</strong></p><p>好了，下面我们就一起来看怎么理解SRE这个体系化工程。</p><h2>SRE，我们应该怎么来理解它？</h2><p>我先给你分享一张图，这是结合我自己团队的日常工作，做出来的SRE稳定性保障规划图。<br>
<img src="https://static001.geekbang.org/resource/image/31/f6/31144fb00cf21005a8d0ae3dc02378f6.jpg?wh=3037*1874" alt=""><br>
我们最初画这张图是为了提高故障处理效率，将每个阶段可以做的事情填了进去，并在实践中不断补充完善，最终形成了我们探索SRE的框架图。你应该也发现了，这里面很多事情都很常见，比如容量评估、故障演练、服务降级、服务限流、异常熔断、监控告警等等。</p><!-- [[[read_end]]] --><p>这就是为什么我一直给你打气，说SRE并不神秘。学习SRE，我们可以有非常熟悉的抓手。但是，我们不能停留在向Google或其他大厂学习具体的技术经验上，而是更应该学习如何将这些技术有机地结合起来，形成一套稳定性体系，<strong>让体系发挥出力量</strong>，告别只发挥某项技术的能力。</p><p>同时，结合这些具体的事情，你应该明白，这些工作并不是某个人、某个角色，甚至是某个团队就可以单枪匹马完成的。比如这里的很多事情要依赖运维自动化，像容量扩缩容，必然会与运维团队有交集；如果做得再弹性一些，还需要与监控结合，就要与监控团队有合作；同时还可能依赖DevOps提供持续交付、配置变更，以及灰度发布这些基础能力，就要与开发和效能团队有交集。</p><p>这些能力之间的相互依赖，就决定了<strong>从职能分工上，SRE体系的建设绝不是单个岗位或单个部门就能独立完成的，必然要求有高效的跨团队组织协作才可以</strong>。</p><p>所以，不要想着设定一个SRE岗位，就能把稳定性的事情全部解决掉，这明显不现实。你应该从体系的角度出发，设置不同的职能岗位，同时还要有让不同角色有效协作的机制。</p><p>从另一个角度来讲，如果当前我们的稳定性建设还仅仅聚焦在某个或某些具体的技术点上，每个角色和团队之间的工作相对还是独立、各自为战的，那就说明SRE体系和组织的真正威力还没有发挥出来。</p><p>所以，如果你已经是这些领域的实践者，比如你是一个运维工程师，或者是一位监控开发工程师，又或者是一个DevOps开发工程师，我建议你除了负责当前的事情外，应该更多地关注一下SRE全局视图。这样会帮助你深入了解SRE岗位所需要的技术能力，进而提升你的平台架构能力。</p><p>要知道，如果严格遵循Google的要求，SRE岗位对技术要求是非常高的。只有具备了全面的技术能力，拥有了全局架构的思维，你才会在SRE的道路上走得更远。</p><p>总结一下，从实践角度来看，SRE的力量不能通过一个岗位、一个或几个技术就发挥出来。SRE是一个体系化工程，它需要协同多个部门、多项技术。</p><h2>做好SRE的根本目的是什么？</h2><p>认识到SRE是个体系化的事儿还不够，我们还可以“以终为始”，看看SRE最后要达成什么样的目标，以此来加深对它的理解。</p><p>SRE做了这么多的事情，最后的目的是什么呢？</p><p>这个答案很明显嘛，就是提升稳定性。但是怎样才算提升了稳定性呢？要回答这个问题，我们有必要来讨论下稳定性的衡量标准。</p><p>从业界稳定性通用的衡量标准看，有两个非常关键的指标：</p><ul>
<li><strong>MTBF</strong>，Mean Time Between Failure，平均故障时间间隔。</li>
<li><strong>MTTR</strong>，Mean Time To Repair， 故障平均修复时间。</li>
</ul><p>还来看前面的SRE稳定性保障规划图，你会发现我们把整个软件运行周期按照这两个指标分成了两段。通俗地说，MTBF指示了系统正常运行的阶段，而MTTR则意味着系统故障状态的阶段。<br>
<img src="https://static001.geekbang.org/resource/image/b3/93/b3c54d78efe3a7616a5e0fc8c124df93.jpg?wh=3034*1874" alt=""><br>
到了这里，我们也就明白了，如果想提升稳定性，就会有两个方向：<strong>提升MTBF</strong>，也就是减少故障发生次数，提升故障发生间隔时长；<strong>降低MTTR</strong>，故障不可避免，那就提升故障处理效率，减少故障影响时长。</p><p>你想，如果我们把故障发生时间的间隔变长，并将故障影响的时间减少，系统稳定性是不是自然就提升了呢？答案是显然的。</p><p>从SRE稳定性保障规划图中，可以看出MTTR可以细分为4个指标：MTTI、MTTK、MTTF和MTTV。我把它们的具体解释做了一张图，你可以保存下来，有时间就琢磨一下。这4个指标，你要烂熟于心。<br>
<img src="https://static001.geekbang.org/resource/image/95/29/9542c5c234c8332f22a9f18b0707cf29.jpg?wh=3107*1874" alt=""><br>
现在，我们再来看SRE稳定性保障规划这张图，你就会理解，为什么要把所做的事情分组分块呈现。目的就是区分清楚，<strong>我们做的任何一件事情、开发的任何一套系统、引入的任何一个理念和方法论，有且只有一个目标，那就是“提升MTBF，降低MTTR”</strong>，也就是把故障发生时间的间隔变长，将故障影响的时间减少。</p><p>比如，在Pre-MTBF阶段（无故障阶段），我们要做好架构设计，提供限流、降级、熔断这些Design-for-Failure的服务治理手段，以具备故障快速隔离的条件；还可以考虑引入混沌工程这样的故障模拟机制，在线上模拟故障，提前发现问题。</p><p>在Post-MTBF阶段，也就是上一故障刚结束，开启新的MTBF阶段，我们应该要做故障复盘，总结经验，找到不足，落地改进措施等。</p><p>在MTTI阶段，我们就需要依赖监控系统帮我们及时发现问题，对于复杂度较高和体量非常大的系统，要依赖AIOps的能力，提升告警准确率，做出精准的响应。</p><p>同时AIOps能力在大规模分布式系统中，在MTTK阶段也非常关键，因为我们在这个阶段需要确认根因，至少是根因的范围。</p><p>你看，我们做了很多事情， 新的理念和方法出来后也特别愿意尝试，就是因为有明确的目标，所有工作的开展都是围绕这个核心目标而去的。</p><p>好了，按照以终为始的思路，SRE要实现的目标就是“提升MTBF、降低MTTR”，从这个角度出发，我们再次认识到一定要<strong>把SRE作为一个体系化的工程来建设</strong>，因为单纯在某些技术点上发力是没有多大意义的。</p><h2>总结</h2><p>今天我要分享的内容就到这里了。总结一下，你需要记住下面这两点。</p><p>第一，我们需要从全局的角度去理解SRE。SRE一定是靠整个技术和组织体系发挥作用的，单纯从某个技术点或环节出发，都无法呈现出效果，也就是说<strong>SRE一定要从全局考虑，体系一定要“配套”。</strong></p><p>第二，SRE的目的，本质上是减少故障时间，增加系统正常运行时间，也就是 <strong>“减少故障，提升MTBF；同时，提升故障处理效率，降低MTTR”</strong>。SRE要做的所有事儿，都是为这两个目标服务的。</p><p>我想无论你是团队负责人、架构师，还是一线的技术专家，有了这样的全局视角后，就会知道接下来应该从何入手，以及如何与其他团队协作来构建这样的体系。</p><h2>思考题</h2><p>开篇词中我们提到过大家对SRE的困惑，比如SRE和DevOps都是很好的方法论，我应该怎么选？到底哪个更适合我？今天我们讲了SRE应该怎么理解，我想对这个问题你应该有答案了。那就请你来说一说，SRE和DevOps到底哪个更适合你？它们有什么区别和相同点？</p><p>请你在留言区说出自己的思考，也欢迎你把今天的内容分享给身边的朋友，和他一起讨论。</p><p>我们下节课见。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/03/fe/d2c856c5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>无聊的上帝</span>
  </div>
  <div class="_2_QraFYR_0">说下我的感受。<br>DevOps主要是以驱动价值交付为主，搭建企业内部的功效平台。<br>SRE主要需要协调多团队合作来提高稳定性。<br>例如：<br>- 与开发和业务团队落实降级<br>- 在开发和测试团队内推动混沌工程落地<br>- 与开发团队定制可用性衡量标准<br>- 与开发，测试，devops，产品团队，共同解决代码质量和需求之间的平衡问题。<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 分析地恰到好处，你找到了看待这个问题的正确角度。<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-18 17:50:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/df/43/1aa8708a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>子杨</span>
  </div>
  <div class="_2_QraFYR_0">听成哥这么一说，顿时有种豁然开朗的感觉。降低 MTTR，提升 MTBF。而 MTTR 里面，MTTI 一定要快。不过这里有一点不太理解，平常都是要以业务恢复为第一优先级的，这时候可能回滚变更操作就非常重要了，然后再去定位根因，先定位根因再去恢复业务，这个是有适用场景的吧。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 非常棒的问题，而且“业务恢复”永远都是第一优先级，你的理解非常正确。<br><br>这里提到是正常流程，但是实践中我们一定会根据实际情况来活学活用，所以，是不是要定位出根因，还是定位出根因范围就要看取舍了，有时我们大致定位出根因范围，然后就要开始执行响应的恢复和隔离措施了。<br><br>有点类似当前我们正在经历的疫情，虽然我们知道是新型病毒，但是根源来自哪里并不清楚，而且也没有预防和有效的治愈良药，这时最有效的办法就是提前发现，然后隔离，这种就不可能等着根因定位清楚，再采取措施。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-19 22:55:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/6b/3a/1ed3634f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>云峰</span>
  </div>
  <div class="_2_QraFYR_0">赵老师，好。SRE是否从项目开始就需要参与系统架构设计？如果只是在项目上线运行后才接触，遇到架构不合理的地方如何处理？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 肯定是越早参与越好，并不一定参与设计本身，但是要知道再哪里提出稳定性要求。比如一个促销场景，要知道可能流量是怎么样的，限流措施要设定在哪些接口上，限流量多大，通过什么方式验证等等，通过这种提要求的方式，倒逼业务开发和架构师思考设计和编码。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-20 14:18:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKagLyKbgMsyKTrLplWu3iauiaGh97dhwFbfQ7RSoCx1SYiaL3icV3UsA5njaUVIYV01a1d2gBzf4CCbQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>海格</span>
  </div>
  <div class="_2_QraFYR_0">SRE解决运维领域的故障目标，DevOps更偏向于为价值导向的效率目标，但是这个又是你中有我，我中有你，互相成就的一个过程，在实践SRE体系过程中，不可避免的要使用到一些DevOps中的一些技术，方法论，组织文化等，通过这些，达成一致目标。~~~</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 理解地非常正确！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-19 09:35:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/64/5b/2591309e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>海峰</span>
  </div>
  <div class="_2_QraFYR_0">看赵老师之前的书讲：SRE是 用软件工程的方法重新设计和定义运维。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: SRE的一个理念，非常关键，一定要通过技术手段解决运维问题，而不是人肉投入。<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-18 21:22:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/6c/67/07bcc58f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>虹炎</span>
  </div>
  <div class="_2_QraFYR_0">感觉要做好sre ,  带头人至少是优秀架构师水准。普通开发只能做好分内事，慢慢学习经验。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你说地对，优秀的SRE就是优秀架构师的水准，甚至比架构师要求更高，而且SRE的成长过程也会比一般开发和架构师更痛苦，要经过更多的磨炼才可以。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-01 20:55:28</div>
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
  <div class="_2_QraFYR_0">SRE  要求对公司业务架构要有一个宏观的了解</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你讲到一个非常重要的点，SRE要想做好，必须要对公司业务有全局的了解，甚至是非常深入的了解<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-18 17:44:26</div>
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
  <div class="_2_QraFYR_0">class SRE implements DevOps <br>可以简单理解为DevOps 是一种接口，但是没说怎么实现，SRE 提供了一种视角，这么做在Google 成功过，可以结合自己企业的特点去实现DevOps 这个接口，做有自己特色的‘SRE’ 即可。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: SRE是DevOps的一项最佳实践。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-18 21:57:14</div>
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
  <div class="_2_QraFYR_0">个人认为：DevOps整体的表现倾向于价值交付，与敏捷的价值观贴合；而SRE的侧重点在于保障系统的稳定运行，通过系统稳定性实现价值最大化！两者有不同，也有交叉，他们不是非此即彼的选项。至于哪种更好，主要看团队的实际情况，产品本身所处的阶段，是另一个重要的考量依据！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 理解地很深入。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-26 00:31:43</div>
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
  <div class="_2_QraFYR_0">赵老师你好，听完本篇我又重新听了开篇。您接下来所讲的内容，应该都是基于团队已经在SRE实践的道路上。1.那么我们该如何判断某条业务线是否值的推行SRE体系呢？(业务的背景大致可以理解为:既追求稳定性，又不过分追求,且DevOps成熟度基本满足业务需求)<br><br><br>下面是自己对DevOps，SRE概念理解的方式，欢迎指正：<br>a. 人们对SRE理解存在偏差，是因为局限个人经验与当下所处维度(IT环境)造成的；通常当我们对一个概念理解存在模糊状态时，通过追溯到它的历史起源，对于理解它会更加深刻，也更加能够看清它真正的意图。<br>b. 一个职位的兴起，绝不是凭空在当前维度出现的，兴许是上游出现了某种压力&#47;变化，于是下游便出现某个职位来应对这种压力&#47;变化。<br><br>文章结尾补充一个问题：DevOps与SRE矛盾点：<br>c. devops解决全栈交付，全栈交付是非稳定性因素之一，而SRE关注稳定性</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 某条业务线是否推行SRE，有个判断依据就是，是不是需要运维或SRE这样的团队介入，如果需要运维或SRE保障，那就要推行后面我们要讲的SRE体系，如果是开发自己维护，没有另外的团队参与，那就自行判断。<br><br>关于你的理解，我觉得很棒，特别是B，任何一个岗位或方法的出现，都是因为有问题解决不了被倒逼出来的，从我的角度，DevOps是如此，SRE也是如此。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-21 12:39:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/ab/f1/cd9ab023.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>潜行</span>
  </div>
  <div class="_2_QraFYR_0">我们是只有三个人的运维小团队，而且职能也没有分的那么清楚细，基本上是每个人所有的事情都做，而SRE又涉及到这么多方面。请问老师，对于我们这种小团队，要实践SRE应该先从哪方面入手？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 团队规模不大，我不建议一开始就去推行SRE。按照经验，这个阶段很有可能你的运维标准，自动化这些工作还没有完全做到位，如果要入手，建议从这里开始，再往后就是做软件的发布，再接下来可以考虑按照我们后面要讲的SLO的内容，去做稳定性的运营。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-21 11:18:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/8f/d3/333f912a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>乐迪_张建</span>
  </div>
  <div class="_2_QraFYR_0">个人觉得DEVOPS目的是更快的创新和更好的客户体验。SER的目的是最快的速度恢复故障！不管是AWS的DEVOPS还是GG的SRE，适合的才是最好的！都是在慢慢通往这个道路上。最后隐隐约约:SRE和DevOps它有，它无:它无，它有！不知道用语言怎么表达</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感觉很准奥，可以看看其他评论内容，这个问题基本就有答案了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-20 12:19:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLke9yyicW4DA3fvmjEXJqnqBiciaUM0R421fvoqVZfvPvDwoTfFo0wjVd9VrMHibk6QgN4PjB00YPXYw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_642328</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，<br>我现在有一个困境，运维关注生产环境的稳定性，线上升级变更操作，我们希望次数尽可能可控，但目前公司产品迭代快，版本更新快，线上升级频繁。在这个过程中，生产环境的稳定性我要如何去把控</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 跟业务和开发达成一致，变更带来的稳定性影响，各方要接受。<br><br>再就是，如果故障较多，可以分析清楚是变更本身，比如代码bug带来的，还是变更执行，人为原因造成的，不同的原因，不同的改进措施。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-13 09:36:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/01/d3/5cbaeb95.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>HUNTER</span>
  </div>
  <div class="_2_QraFYR_0">除了故障平均时间间隔和故障修复时间，衡量标准里是否还应该有故障影响半径呢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-23 18:57:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/66/00/b7b58d74.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我？好吧好吧！</span>
  </div>
  <div class="_2_QraFYR_0">有个小疑问~当稳定性提高到一定程度时，技术手段对稳定性的推动就会减速或者陷入瓶颈，这个时候往往需要增加成本稳定性才能获得进一步的提高（资源较多冗余）。老师如何看待这种用成本换取稳定性的模式？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 成本换稳定的一个前提是架构上要支持，有时候单纯的成本投入也是没有意义的，比如要见双活，本质上要架构支持双活，比如数据作了分片、同步效率足够高、架构做了单元化等等。<br><br>如果单纯从成本考虑，那就看roi，或者看企业对于成本的承受情况。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-14 18:05:45</div>
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
  <div class="_2_QraFYR_0">虽然现在是在做devops，我们公司没有sre的岗位，听完这节课之后还是对这两个岗位有了一定的认识。<br>devops目标是提升开发效率和提升交付效率。<br>sre是保证服务稳定。<br>devops针对的是交付产品和开发者，sre针对的是服务。<br>一个追求交付在产品的质量上的快，一个是追求产品部署之后部署的稳。<br><br>在请教老师个问题，是不是sre岗位在to c的公司居多呢，国内sre现状是咋样的呢。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不仅toC需要，toB也需要，很多的云计算公司，都有专门的SRE部门和岗位。<br><br>再就是也不一定非要有SRE的岗位，就像我在本篇中讲到的，很多的SRE工作我们都在做，只不过是分散在了不同的部门和岗位上。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-20 23:40:22</div>
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
  <div class="_2_QraFYR_0">sre体系，更多能直接的反映出企业运维架构和能力水准<br>devops体系，更多能直接反应企业开发快速交付和快速上线部署的能力</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-25 16:09:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/0b/ef/de0374ac.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>祝洪娇</span>
  </div>
  <div class="_2_QraFYR_0">SRE更适合很少有需求变更的项目，主要是进行系统运维，侧重维护保持系统的稳定性。<br>DevOps更适合需求变更比较频繁的项目，侧重快速稳定交付。<br>但是大部分项目都需要二者相结合，在持续交付的同时保持系统的稳定性。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-02 21:27:54</div>
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
  <div class="_2_QraFYR_0">SRE是DevOPs的一种实现方式，这两者有紧密的联系</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-09 15:37:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/df/52/92389851.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>hy</span>
  </div>
  <div class="_2_QraFYR_0">DevOps三原则:流动、反馈、持续学习与实验都是为了快速实现价值流交付，和开发团队的目标是一致的。SRE更多的是从稳定性出发，和运维团队的目标是一致的。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-15 23:51:52</div>
  </div>
</div>
</div>
</li>
</ul>