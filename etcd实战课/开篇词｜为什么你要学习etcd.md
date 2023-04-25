<audio title="开篇词｜为什么你要学习etcd" src="https://static001.geekbang.org/resource/audio/51/0d/5140e6337fe49d48e5d3d3414a64420d.mp3" controls="controls"></audio> 
<p>你好，我是唐聪，etcd活跃贡献者，腾讯资深工程师，欢迎你和我一起学习etcd。</p><p>开门见山，今天我想和你聊聊为什么要学习etcd。随着Kubernetes成为容器编排领域霸主，etcd也越来越火热，越来越多的软件工程师使用etcd去解决各类业务场景中遇到的痛点。你知道吗？etcd的GitHub star数已超过34.2K，它的应用场景相当广泛，从服务发现到分布式锁，从配置存储到分布式协调等等。可以说，etcd已经成为了云原生和分布式系统的存储基石。</p><p>另外，etcd作为最热门的云原生存储之一，在腾讯、阿里、Google、AWS、美团、字节跳动、拼多多、Shopee、明源云等公司都有大量的应用，覆盖的业务可不仅仅是Kubernetes相关的各类容器产品，更有视频、推荐、安全、游戏、存储、集群调度等核心业务。</p><h2>我想为你解决哪些问题？</h2><p>在工作和参与etcd社区贡献的过程中，我经常会收到各类问题咨询，同时自己也经历了各种问题。我相信你在使用Kubernetes、etcd的过程中，很可能也会遇到下面这些典型问题：</p><ul>
<li>etcd Watch机制能保证事件不丢吗？（原理类）</li>
<li>哪些因素会导致你的集群Leader发生切换? （稳定性类）</li>
<li>为什么基于Raft实现的etcd还可能会出现数据不一致？（一致性类）</li>
<li>为什么你删除了大量数据，db大小不减少？为何etcd社区建议db大小不要超过8G？（db大小类）</li>
<li>为什么集群各节点磁盘I/O延时很低，写请求也会超时？（延时类）</li>
<li>为什么你只存储了1个几百KB的key/value， etcd进程却可能耗费数G内存? （内存类）</li>
<li>当你在一个namespace下创建了数万个Pod/CRD资源时，同时频繁通过标签去查询指定Pod/CRD资源时，APIServer和etcd为什么扛不住?（最佳实践类）</li>
</ul><!-- [[[read_end]]] --><p>当然，你在学习和使用etcd、Kubernetes过程中遇到的问题肯定远远不止这些，下面我用思维导图给你总结了更多类似问题，你可以对照自身的经历去看一下。</p><p><img src="https://static001.geekbang.org/resource/image/7e/54/7e05c744ba292cf26c39d69101200554.jpg?wh=5378*2217" alt=""></p><p>这门课就是为了帮助你解决这些问题而生。不过你可能会想，你能把这些东西都讲明白么？我先和你聊聊我的个人etcd经历，你就知道我为什么有自信能带你学好etcd了。</p><h2>我和etcd的那些事</h2><p>本科毕业后，我通过校招加入了腾讯。不到一年的时间，我就主导完成了一个亿级用户的业务核心存储平滑迁移任务。</p><p>在2015到2017的这两年时间里，为了满足业务大量的Redis诉求，我基于Redis/Codis构建了大规模的排行榜和Redis集群平台服务，支撑了公司的多个重要业务。在这期间，我积累了大量的NoSQL数据库知识与经验，为后面工作转岗到To B，负责Kubernetes的元数据存储etcd奠定了良好的基础。</p><p>2017年后，我就开始接触Docker和Kubernetes，并通过Kubernetes来解决大规模Redis集群的治理问题，提升服务的可用性、降低运维成本。</p><p>2018年，我转岗到了腾讯云，负责Kubernetes集群存储etcd治理工作。我主导构建的云原生etcd平台，支持自动化的集群管理、调度、迁移、监控、巡检、备份，成功解决了集群大规模增长过程中的各类etcd稳定性问题，支撑了万级的Kubernetes和etcd集群。</p><p>etcd平台从解决Kubernetes etcd稳定性问题，到为各类云原生产品提供etcd基础服务，再到保障开箱即用的腾讯云etcd产品化服务，它发挥着重要作用。在这个过程中，我也见证了越来越多的软件工程师加入etcd的阵营，越来越多的产品使用etcd。目前，etcd作为腾讯众多产品的基础设施，服务用户已达数亿。</p><p>同时，我也遇到了很多问题，从内存泄露到数据不一致，从节点crash到性能慢，再到死锁、OOM等稳定性问题等等。最令我记忆犹新的是，我和小伙伴王超凡通过混沌工程发现并修复了多个数据不一致Bug，其中一个Bug已经存在近3年之久，而且很严重，重启就可能会触发数据不一致。</p><p>从解决类似上面的棘手Bug到提交稳定性、性能优化PR，从提交QoS特性设计方案、POC到给新的contributor review PR，通过一点点的积累，大量周末、凌晨时间的付出，我成为了2020年etcd社区的全球Top3活跃贡献者，与Google、AWS、阿里巴巴的小伙伴们，一起推动etcd项目越来越好，服务于全球开发者。</p><p>总结来说的话，过去几年我一直在与Redis、etcd打交道，一线的经历、解决的问题都让我收获良多，所以我也非常有自信能把这些经验都交付给你。</p><p>在业务实践方面，我成功解决过众多大规模业务增长过程中，遇到的存储稳定性、可扩展性等痛点，积累了丰富的理论知识、大规模集群的实战、治理经验，能直接帮助到你今后的工作。</p><p>另外，在etcd开源项目方面，我深度参与etcd开源项目的贡献经历，让我可以从开发者的视角，为你分析问题、梳理最佳实践、解读特性设计方案、阐述社区未来演进方向等等，帮助你深度理解etcd以及分布式服务。</p><h2>你应该怎么学etcd？</h2><p>在我看来，etcd学习其实可以分为大中小三个目标。最大的目标我当然是希望你能够用最低的学习成本，掌握etcd核心原理与最佳实践，让etcd为你所用，帮助你解决业务过程中的各类痛点，在工作中少踩坑、少交学费，多升职、多涨薪。</p><p>但是这个大的目标怎么实现呢？</p><p>我的答案是<strong>使用拆解法</strong>。下面我给你提出了学习这个专栏的一些中等大小目标，希望你能带着这些目标进行学习，每过一段时间，回过头来看看，这些目标实现了多少？</p><p>首先，你能知道什么是etcd，了解它的基本读写原理、核心特性和能解决什么问题。</p><p>然后，在使用etcd解决各类业务场景需求时，能独立判断etcd是否适合你的业务场景，并能设计出良好的存储结构，避免expensive request。</p><p>其次，在使用Kubernetes的过程中，你能清晰地知道你的每个操作背后的etcd是如何工作的，并遵循Kubernetes/etcd最佳实践，让你的Kubernetes集群跑得更快更稳。</p><p>接着，在运维etcd集群的时候，你能知道etcd集群核心监控指标，了解常见的坑，制定良好的巡检、监控策略，及时发现、规避问题，避免事故的产生。</p><p>最后，当你遇到etcd问题时，能自己分析为什么会出现这样的错误，并知道如何解决，甚至给社区提PR优化，做到知其然知其所以然。</p><p>做到以上五个目标其实也并不容易，别着急，我们接着往下拆分。为了让你实现以上五个目标，我把专栏分为了基础和实践两大主线。每个主线里都有一个一个的小目标，我们逐个攻破就容易多了。</p><p>基础篇主线是为了帮助你建立起对etcd的整体认知，搞懂读写请求、各个核心特性背后的原理，为我们后面的实践篇打下基础。</p><p>基础篇的学习也是一个中小型分布式存储系统从0到1的实现案例解读，学习它你收获的不仅仅是etcd，更是如何构建分布式存储系统的理论知识。</p><p>我把基础篇分为了以下的学习小目标：</p><ul>
<li>etcd基础架构。通过为你梳理etcd前世今生、分析etcd读写流程，帮助你建立起对etcd的整体认知，了解一个分布式存储系统的基本模型、设计思想。</li>
<li>Raft算法。通过为你介绍Raft算法在etcd中是如何工作的，帮助你了解etcd高可用、高可靠背后的核心原理。</li>
<li>鉴权模块。通过介绍etcd的鉴权、授权体系，带你了解etcd是如何保护你的数据安全，以及各个鉴权机制的优缺点。</li>
<li>租约模块。介绍etcd租约特性的实现，帮助你搞懂如何检测一个进程的存活性，为什么它可以用于Leader选举中。</li>
<li>MVCC/Watch模块。通过这两个模块帮助你搞懂Kubernetes控制器编程模型背后的原理。</li>
</ul><p>在介绍etcd原理的过程中，我也会从更上层的角度，为你解读分布式系统存储系统的核心技术难点是什么，常见的解决方案有哪些，以及为什么etcd要这样设计、实现。让你对整个分布式系统有更深层次的理解，明白不同存储系统只是在面对各自的业务场景的时候，选择了合适的技术方案，让你从本质上去理解分布式存储系统要解决的核心问题基本是一致的。</p><p>当然基础篇讲的远不止这些，关于基础篇的更多内容，你可以参考下面的etcd基础篇思维导图：</p><p><img src="https://static001.geekbang.org/resource/image/ed/ea/edf53f37c0725c9757e4ecb89982a7ea.jpg?wh=3920*2495" alt=""></p><p>通过基础篇掌握好etcd核心模块原理后，实践篇我将为你解读实际使用etcd时，可能会遇到的各种问题，帮助你提前避坑、遇到类似问题时能独立分析、解决。</p><p>我把实践篇分为以下的学习小目标：</p><ul>
<li>问题篇。为你分析etcd使用过程中的各类典型问题，和你细聊各种异常现象背后的原理、最佳实践。</li>
<li>性能优化篇。通过读写链路的分析，为你梳理可能影响etcd性能的每一个瓶颈。</li>
<li>实战篇。带你从0到1亲手参与构建一个简易的分布式KV数据库，进一步提升你对分布式存储系统的认知。</li>
<li>Kubernetes实践篇。为你分析etcd在Kubernetes中的应用，让你对Kubernetes原理有更深层次的理解。</li>
<li>etcd应用篇。介绍etcd在分布式锁、配置系统、服务发现场景中的应用。</li>
</ul><p>更多实践篇内容你可以参考下面的思维导图：</p><p><img src="https://static001.geekbang.org/resource/image/42/81/42eaa94b0f4f7a18895780e6f61ce381.jpg?wh=3912*2990" alt=""></p><p>这样一来，我们的学习目标就比较明确了。最终目标是让etcd为你所用，少踩坑、多升职加薪；而为了实现这个目标，我们需要从多方面提升自己对etcd的掌控能力，也就是实现中等目标；但进阶的难度还是比较大的，所以我们需要把一个个小目标当作基石（也就是每一节课的知识点学习），来达成个人能力的提升。</p><p>现在我们不妨就带着这些目标，共同开启etcd的学习之旅吧！</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/67/ae/37b492db.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>唐聪</span>
  </div>
  <div class="_2_QraFYR_0">非常感谢关注etcd专栏的同学们，大家在试读过程中，可能已经发现每讲比较长，大部分5000多字左右，就为了尽量给大家透彻的分享清楚，一两个问题。<br><br>希望通过好几个月周末与凌晨时间，总结出的15万多字文章，1百多张图，能帮助大家通过短短短短几个月的学习，快速获得我多年知识与丰富实战经验总结，提升大家对etcd的了解，少踩坑，一定会物有所值。<br><br>欢迎大家和我一起学习etcd。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-27 13:25:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/02/db/5d227be5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>xiaoyu</span>
  </div>
  <div class="_2_QraFYR_0">正在学习kubernetes，之前不是很了解，所以etcd是什么，云原生存储？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: kubernetes的元数据存储，保存了你通过kubectl命令所看到的node&#47;pod&#47;deployment等各种资源、配置信息等</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-30 17:47:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/13/45/16c60da2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>@%初%@</span>
  </div>
  <div class="_2_QraFYR_0">等来了etcd,一起加油呦</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢你的支持，一起加油!</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-20 21:56:09</div>
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
  <div class="_2_QraFYR_0">终于来了！期待已久！按照老师的拆解法去执行！一定收获满满……</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，后续有什么不清楚的，欢迎随时给我留言! 一起加油!</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-20 18:43:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/46/42/eaa0cc92.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>无风</span>
  </div>
  <div class="_2_QraFYR_0">厉害了，看思维导图很清晰。<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-21 16:10:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1c/ac/de/68f35320.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小来子</span>
  </div>
  <div class="_2_QraFYR_0">看到了学长的名字. 哈哈哈哈哈哈~</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-21 16:17:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/61/e6/fedd20dc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>mmm</span>
  </div>
  <div class="_2_QraFYR_0">期待已久，赶紧交学费！leader频繁切换，几百kv却消耗几G内存都是工作中遇到过问题，一脸茫然，希望能在这里找到答案</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，可以先看看前面的基础篇文章，04 raft就可能会导致内存和leader频繁切换问题</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-30 10:51:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/2a/b9/2bf8cc89.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>无名氏</span>
  </div>
  <div class="_2_QraFYR_0">工作中正好用到，跟随大佬学习etcd</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 一起学习，加油，一定能让你收获满满，少踩坑</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-28 12:32:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/34/c3/ed5881c6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>手机失联户</span>
  </div>
  <div class="_2_QraFYR_0">etcd 实现的 Raft 算法模块是比较复杂的，很期待老师会怎么讲解这一部分</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-21 10:05:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/5c/88/9575e5d3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>linxs</span>
  </div>
  <div class="_2_QraFYR_0">从12月等到现在终于来了hhhh</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢支持，相信你一定能有所收获!</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-20 16:54:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/78/df/424bdc4a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>于途</span>
  </div>
  <div class="_2_QraFYR_0">从去年11月盼到了现在。O(∩_∩)O哈哈~</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-21 14:10:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/75/0b/553d4ec4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>逍遥</span>
  </div>
  <div class="_2_QraFYR_0">之前两个项目简单用过，算大致了解怎么用，但是一些具体的细节或者深入的原理和问题背后的原因，就模糊了。这个课程很及时啊！紧跟大佬的步伐，冲呀</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，用在什么场景，具体是什么问题，我看看内容是否已经覆盖<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-20 17:45:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/75/9b/611e74ab.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>技术修行者</span>
  </div>
  <div class="_2_QraFYR_0">平时工作中，直接使用etcd的场合不多，但是它是分布式系统中很重要的一部分，之前的使用，更多像是黑盒模式。<br>看完导读，感觉内容很有吸引力，希望学完后，我能对etcd有一个全面深入的理解。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-14 18:20:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/d9/6f/62e027f4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>风有点儿大</span>
  </div>
  <div class="_2_QraFYR_0">你好，有ETCD key 命名规范的文章吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-26 14:57:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/29/80/e3a4962d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>雪原与星空之间</span>
  </div>
  <div class="_2_QraFYR_0">开始学习</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-14 13:59:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/93/ec/985675c8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小高</span>
  </div>
  <div class="_2_QraFYR_0">掌握etcd，就掌握了分布式系统的基础，加油，四刷了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 加油，建议再结合源码一起看看</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-14 10:16:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/20/c3/94/e89ebc50.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>神毓逍遥</span>
  </div>
  <div class="_2_QraFYR_0">第一边快扫而过，这是第二边，认真以待，老师的知识结构划分是很不错的，混沌工程如何发现数据不一致性的，这个可以提下吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢支持与认可，主要是基于https:&#47;&#47;github.com&#47;chaos-mesh&#47;chaos-mesh和etcd functional test(https:&#47;&#47;github.com&#47;etcd-io&#47;etcd&#47;tree&#47;main&#47;tests&#47;functional)、还有我们自己基于开发的一些chaos monkey，去随机restart&#47;kill etcd等</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-06 23:10:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/1pMbwZrAl5g8ZRC5SOlz9VQOVRGN5V1rFNwsmEehicGzxMRicGj330jNfwTE1MRLaZTdSvX8R2YJVEh8DrYnMVAQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_edd932</span>
  </div>
  <div class="_2_QraFYR_0">老师，请问一下，etcd集群从V3.3.18能零停机、滚动升级到V3.5.0吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 一般场景下，建议先测试环境升级到v3.4最新版本，再升级到3.5版本，3.5才出没多久，建议再等等，先测试环境用起来。<br>升级的时候务必看看upgrade checklist, 有breaking changes api和其他已知bug,比如之前我们遇到的数据不一致bug.https:&#47;&#47;github.com&#47;etcd-io&#47;website&#47;blob&#47;main&#47;content&#47;en&#47;docs&#47;v3.5&#47;upgrades&#47;upgrade_3_3.md，测试环境搞完再搞生产，数据做好备份</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-30 22:23:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/2c/e2/afa3867c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>凡俗</span>
  </div>
  <div class="_2_QraFYR_0">玩Kubernetes的话，如果不能理解它的数据存储层面，那么还是不够深度</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-30 12:54:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/29/f1/78/3ef66168.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>StarOS</span>
  </div>
  <div class="_2_QraFYR_0">团队上k8s的优雅姿势，不需要懂k8s，即可享受k8s带来的便利，体验产品 http:&#47;&#47;staros.cloud?hmsr=C16&amp;hmpl=A5<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-27 17:18:22</div>
  </div>
</div>
</div>
</li>
</ul>