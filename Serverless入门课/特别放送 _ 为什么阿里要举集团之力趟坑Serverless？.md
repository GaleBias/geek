<audio title="特别放送 _ 为什么阿里要举集团之力趟坑Serverless？" src="https://static001.geekbang.org/resource/audio/d2/ee/d2ecde527136a457ee99cdb9b53c6aee.mp3" controls="controls"></audio> 
<p>你好，我是专栏编辑冬青。在课程正式开始之前，我要先向你推荐一篇InfoQ记者蔡芳芳的文章。为了探究Serverless的价值、生态位以及局限之处，蔡芳芳采访了阿里高级前端技术专家杜欢，并表达了自己的看法。以下为具体内容。</p><p>作为阿里巴巴经济体前端 Serverless 研发升级项目的负责人，杜欢过去两年花了大量时间推进集团内部的 Serverless 研发模式升级工作。这是一项牵涉整个阿里集团层面的技术升级工作。阿里 2018 年正式启动内部 Serverless 资源底座的准备工作，2019 年基于搭建好的底座建设上层前端框架，到 2019 年双十一，阿里已经在部分电商导购业务上开始实践这套新的研发模式。</p><p>Serverless 兴起于 2017 年，在最近两年伴随着云原生概念的推广愈发火爆。在这波 Serverless 浪潮里，阿里是国内走得最前面、说得也最多的一个。但为什么阿里会上升到整个集团的高度来推进 Serverless 研发模式升级，我们仍然感到好奇。对此，杜欢表示：“<span class="orange"> 这件事本身好像是一件技术的事情，但其实它背后就是钱的事情，都是跟钱相关的。</span>”</p><p>杜欢告诉 InfoQ 记者，为了保障业务的稳定性和可用性，阿里对每一个应用上线都有相应的规范和规则。哪怕是一个很小的内部应用，一天可能只有一两个访问量，上线也需要遵守既有的规范，这势必会消耗一些固定资源。单个应用消耗的资源可能很有限，但所有应用消耗的资源累积起来也是一个不小的数字。阿里内部自己做了分析发现，除了主要的核心应用之外，已经上线的应用中超过 80% 都是非核心的中长尾应用。目前阿里经济体的体量和业务量已经达到非常大的量级，在现有的研发模式下，这些中长尾应用会带来比较大的资源浪费。</p><!-- [[[read_end]]] --><p>其次，现有的研发形态并不能最大化地发挥部分工作岗位的价值，这同样是一种浪费。以导购类型的业务为例，开发这样一个业务通常需要前端开发工程师和后端开发工程师一起配合，但这两个开发岗位在该业务形态下并不能很好地发挥自己的全部价值。对于后端工程师来说，他在这个业务里要做的事情更多只是把现有的一些服务能力、数据组合在一起，变成一个新的数据提供给前端；而前端开发工程师负责把这个数据在页面上展示出来。这样的工作比较机械化，也没有太大的挑战，并不利于后端工程师的个人成长和岗位价值发挥；但在现有的研发模式下，由于缺乏前后端的连接点，前端工程师又不能去做这些比较简单的后端工作，业务上线也不可能给到前端工程师时间和机会去学习再实践。</p><p>而 Serverless 既可以满足资源最大化利用的需求，也能够调优行业内的开发岗位分层结构，让每个开发岗位都能够在最适合自己的地方发挥最大的价值。杜欢表示，阿里正是因为看到了经济体内存在的上述问题，在尝试寻找相应解决方案的过程中发现 Serverless 可能是一个好的解决方案，才开始研究 Serverless，研究后认为确实可行，才启动了“阿里巴巴经济体前端 Serverless 研发升级项目”这样一个项目，拉上大家一起来共建。</p><p>那么 Serverless 是只对阿里这样的公司才适用吗？什么样的公司、应用或场景应该选用 Serverless 的架构模式？</p><p>在杜欢看来，这是一个“伪问题”。他直言，并不存在什么样的场景和模式适合 Serverless，Serverless 应该被广泛地运用在不同的场景和实际开发需求中。从云厂商的角度来看，云计算未来一定会成为整个社会和商业的基础设施，届时使用云计算就应该像现在我们使用水电煤一样简单，不需要了解水从哪里来、怎么过滤、怎么铺设管道等一系列问题，只需要打开水龙头接一杯水而已。</p><p>而 <span class="orange">Serverless 的概念正好可以帮助云计算朝这个方向往前走一步</span>，它提倡的是人们不需要关心应用逻辑以外的服务相关的事情，包括管理、配置、运维等，用多少就付多少。从这个角度来看，Serverless 是真正让云计算变成社会商业基础设施的一个实现路径，也更接近现在业内提倡的云原生的方式，因此人们在使用云计算的过程中自然就应该按照 Serverless 的方式来使用。</p><p>杜欢认为，大家今天对是不是该用 Serverless 还有疑问，主要是因为还没有看到足够多 Serverless 成功应用的案例。这也是阿里巴巴先从自己内部实践 Serverless 研发模式升级的另外一个原因，希望通过这件事达成两个目的，一是向大家普及 Serverless 的概念，二是从自己的实践过程中总结出一套好的实践方式并共享出来，帮助大家更好地了解应该怎么落地 Serverless。</p><h3>前端开发者看好、后端开发者观望，Serverless 为何如此？</h3><p>过去这一年 Serverless 在前端领域被频繁提及，很多开发者非常看好，并认为它一定是未来前端大趋势之一，相比之下非前端领域还是观望态度居多。杜欢告诉记者，之所以会出现热度差，是因为 Serverless 的概念天然弥补了前端开发工程师的不足，但却跟现在很多后端开发工程师的能力有一定程度的重叠。</p><p>在杜欢看来，绝大多数后端开发工程师的成长路径是先做后端业务逻辑，慢慢了解应用的每个环节，再逐步成长为后端架构师，能 Hold 住整个应用架构。但云原生 Serverless 的出现，一定程度上会使这种后端架构能力变得普及，并且是以平台或服务的方式，而不是以人的方式。这乍看起来对于整个后端岗位可能是一种冲击，但杜欢认为 Serverless 并不会完全取代后端。当前整个开发生态依然缺少优秀的架构服务，尤其是低成本、可持续发展、针对特定行业优化的架构服务。未来后端工程师还是会成长为架构师，但是可能不是通用架构师，而是偏业务解决方案的行业架构师。此外，云厂商也需要专门构建 Serverless 方案的架构师，这是后端工程师的另一个新机会。</p><p>杜欢告诉 InfoQ 记者，Serverless 对前端和后端带来的影响总体都是正向的。对于前端来说，Serverless 不仅补足了前端工程师现有的能力，还可能使整个前端行业的定位发生变化。原来经常有人会认为前端的工作很简单，面向 UI 做好开发就行，剩下的工作可以交给后端。但是云端的应用开发模型出来之后，也就是前端和 Serverless 结合之后，大家对前端的诉求就不仅仅是开发一个页面了，而是要能交付整个应用的开发。</p><p>前端工程师除了要保持在 UI、交互逻辑方面的优势，还要理解整个业务和业务背后的意图，这意味着未来前端行业的思考模式会变成面向业务的思考模式。与此同时，前端的协同和开发模式、上下游流程也会发生变化，原来前端可能很少跟产品经理、设计打交道，未来前端要对整个应用负责，就需要天天跟产品经理、设计打交道。后端则要在最底层提供更深的能力付出，比如如何按照一亿流量的支出支撑十亿流量，这是更大的挑战。</p><h3>目前 Serverless 最佳实践模式尚未出现</h3><p>当前谈 Serverless，很多人会提函数计算、FaaS，但杜欢认为当前 Serverless 尚未出现一个最佳实践模式。在他看来，现在已经有一些 Serverless 的框架开始涌现，云厂商对 Serverless 的支持也越来越完善，确实是时候去尝试实践 Serverless 了。但要说最佳实践，就必须有足够多的人去实践过并表示认同，而现在实践 Serverless 的人本来就很少、实现的业务也很少，因此还没有哪一种方式谈得上是最佳实践。</p><p>不过杜欢提到 Serverless 实践过程中有一点需要重点关注——学习曲线是否足够平缓。要成为最佳实践模式，至少要做到能让开发者以一种方式专注于业务代码的开发，无需关注运行平台的差异性，一处编写可以处处运行，开发者只要掌握一种方式就可以在不同业务之间没有学习成本地切换。阿里巴巴近期开源的函数运行时框架 Midway FaaS Run Time 就是朝着这样的目标设计出来的，但还需要更多人尝试并认可，才有可能在未来变成最佳实践。</p><p>杜欢表示，目前 Serverless 在国内的发展和采用依然处于初期阶段，经过这两年的概念普及，大部分人都已经注意到并接受了 Serverless，但业务实践偏少，仍在不断探索之中。相比之下，国外整体要领先 1-2 年，国外几个大的云厂商前期对整个研发生态的教育和布道做得比较多，应用也比较早。现在国外已经出现不少 Serverless 框架，比较知名包括 Serverless.com 和 Zeit.com。</p><p>但对于 Serverless 未来的发展，杜欢信心十足。在他看来，未来云计算的普惠，一定是通过 Serverless 的方式去放大和落地实现的，而基于云端的应用模式也一定会是未来创新创业的选择。问题在于如何做好从传统开发模式到云开发模式的迁移。其中非技术层面的挑战主要来自于开发者，前后端工程师对 Serverless 的态度可能不一致，有的后端工程师会觉得 Serverless 抢了自己的工作很难接受，他没有看到更深层次对自己有收益的地方；有的前端工程师认为 Serverless 只是增加了自己要做的事情，而不能看到这个东西对自己提出了更高的要求，那他未来也未必能够胜任这项工作。</p><p>而技术层面的挑战主要包括两块：首先，云厂商自身要提供更多的 Serverless 能力，或者说现有云计算的能力要有更多被转换成能以 Serverless 的方式提供服务，未来云厂商要越来越多地提供这方面的支持；其次是研发模式，未来对于云时代的原住民来说，所有东西都在云上，开发方式必然会发生变化，如何去解决这些问题，让一个新人加入一项新业务之后可以更快地写下第一行代码，这是另一项挑战。</p><p>展望 2020 年 Serverless 的发展趋势，杜欢说道：“2020 年 Serverless 会进入初步实践阶段，还不能称之为大规模实践，可能到 2021 年才会进入大规模实践阶段。在这个过程当中，云厂商会进一步补充更多的 Serverless 服务，包括一些后端的 BaaS 服务，把基础打得更牢一点。”</p><p>今天的分享就到这里，下节课我们正式开始学习！</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/c9/68/1cf4dcf7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>大表哥</span>
  </div>
  <div class="_2_QraFYR_0">好一个’钱’字，不仅仅打动了我，也打动了老板的心儿啊。哈哈哈。你们要小心了后台架构来学serverless了！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-28 11:17:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/c2/e3/df7447ff.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>008</span>
  </div>
  <div class="_2_QraFYR_0">如果有vendor-lock的前提的话，“至少要做到能让开发者以一种方式专注于业务代码的开发，无需关注运行平台的差异性，一处编写可以处处运行，开发者只要掌握一种方式就可以在不同业务之间没有学习成本地切换”，这句话还是片面了，“处处运行”仍然是局限的。<br><br>不过所有上云的事情，总会面临不少厂商差异，就看企业如何评估这个差异在未来给自己带来的麻烦。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-20 19:09:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/63/85/1dc41622.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>姑射仙人</span>
  </div>
  <div class="_2_QraFYR_0">对云厂商最有优势</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Serverless有一定的vendor-lock属性，不过对开发者也利好，所有云服务商都提供免费的FaaS额度给大家使用。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-17 08:50:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/7b/f1/da02bebb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>闹够了</span>
  </div>
  <div class="_2_QraFYR_0">我要提前上车了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 来不及了，快上车</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-22 08:24:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/4c/b5/fcede1a9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>IT小村</span>
  </div>
  <div class="_2_QraFYR_0">serverless目前还是后端在用</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-05 17:16:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/ee/e2/23e44221.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>余熙</span>
  </div>
  <div class="_2_QraFYR_0">今天看到，阿里函数计算(包年包月) - 已于2021年08月03日0时下线，我去试用腾讯云函数，给大伙探探路鸭</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-20 11:11:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/cf/32/9aede769.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>麦乐</span>
  </div>
  <div class="_2_QraFYR_0">我也要上车了~</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 欢迎欢迎~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-16 13:10:09</div>
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
  <div class="_2_QraFYR_0">水电煤是好东西</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-29 21:06:44</div>
  </div>
</div>
</div>
</li>
</ul>