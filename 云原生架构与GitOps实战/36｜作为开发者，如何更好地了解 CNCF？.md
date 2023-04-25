<audio title="36｜作为开发者，如何更好地了解 CNCF？" src="https://static001.geekbang.org/resource/audio/f7/95/f79c0131b3c03a52160ca46a8c632495.mp3" controls="controls"></audio> 
<p>你好，我是王炜。</p><p>从这节课开始，我们进入到一个相对轻松的模块，也就是特别放送模块。</p><p>提到云原生领域，我相信你多少都听说过 CNCF (Cloud Native Computing Foundation)&nbsp;云原生计算基金会，它是 Linux 基金会组织和管理的一个非营利性的技术基金会，致力于推动云原生计算的发展。CNCF 主要关注云原生软件的标准化、普及，以及云原生计算的教育和培训。</p><p>对于开发者来说，由于 CNCF 是云原生最上游的组织，所以对它保持关注有助于我们获取一手信息，还有助于我们了解行业发展情况，提升技能水平。</p><p>所以，这节课，我将会带你从零开始认识 CNCF，包括它的历史、社区组织形式、项目托管以及职业认证等，让你了解 CNCF 的运作机制，更好地从 CNCF 获取信息。</p><h2>CNCF 和云计算历史</h2><p>CNCF 成立于 2015 年 12 月，它是 Linux 基金会的一部分。在成立之初，CNCF 得到了 Google 和 SoundCloud 的支持，这两家公司分别捐赠了著名的 Kubernetes 以及 Prometheus，在当时，一并作为会员加入 CNCF 的企业还有：Cisco、CoreOS、Docker、Google、华为、IBM、Intel 和 Redhat 等。</p><!-- [[[read_end]]] --><p>如果我们回顾云计算历史，会发现 CNCF 的诞生是非常顺应时代的。</p><p>2000 年以前，当时流行的技术是以 Sun 公司为代表的非虚拟化技术，在需要运行应用时，首先要购买物理服务器，然后在服务器上运行它。</p><p>1 年后，也就是 2001 年，VMWARE 的虚拟化技术得到普及，我们能够在一台物理机上运行多个虚拟机了，虚拟机成为了程序运行的载体。</p><p>2006 年，Iaas（基础设施即服务） 诞生了，AWS 创建了以 EC2 服务器为代表的云计算和弹性计费的方式，用户在使用的时候可以按小时付费，AMI（Amazon Machine Image）镜像成为了程序打包和运行的普遍方式。</p><p>3 年后，也就是 2009 年，PaaS（平台即服务）诞生了，以 Heroku 为代表的 PaaS 平台变得非常流行，这时候，基础设施层面产生了巨大的变化，以 Buildpack 为代表的技术已经开始有了容器的概念，尽管这个过程并不透明。在当时，交付应用只需要执行一条命令简直是一项魔法技术。</p><p>2010 年，IaaS 层的开源方案 OpenStack 诞生了，它由 AWS 和 VMWARE 完成，至今 OpenStack 在私有云的市场仍然是非常流行的解决方案。2011 年，Cloud Foundry 发布了开源的 PaaS 解决方案，它是 Heroku 的开源替代方案。</p><p>2013 年，最著名的 Docker 技术诞生了，Docker 整合了 LXC、联合文件系统和 cgroups 技术并创建了一个容器化标准，它是有史以来普及率最快的开发者技术，现在仍然被全世界的开发者使用。Docker 技术实现了隔离、可重用和不可更改性，它彻底改变了应用的构建、分享和交付方式。</p><p>随着容器技术的蓬勃发展，2 年后，也就是 2015 年，CNCF 成立了，CNCF 开始传播微服务和容器化的技术。直到今天，微服务和容器化仍然是让企业趋之若鹜的热点技术。</p><p>从历史发展中看，我们会发现应用的运行环境产生了巨大的变化。从最初的物理机，到虚拟机，再到 Buildpacks，最后到容器，应用的交付产物越来越内聚。</p><p>此外，运行环境的隔离性也产生了一系列的变化，从最初的硬件隔离，到虚拟化隔离，最后到容器技术的 cgroups 隔离，它们的隔离方式越来越轻量。</p><p>最后，从供应商的角度来看，软件从最初的封闭和单一供应商逐渐演进为开源和跨供应商。</p><h2>组织形式</h2><p>CNCF 是一个中立组织，它主要通过推动开源项目的发展来实现自身的目标，所以它的社区组织形式是为了更好地推动开源项目发展而设计的。</p><h3>员工、会员和大使</h3><p>首先，CNCF 作为非营利性组织有它自己的全职员工。就像公司的组织架构一样，它也有 CTO、总监、项目管理和开发者关系等职能岗位。此外，由于 CNCF 的工作大多数是围绕着开源项目的社区会议进行的，所以它还有诸如会议和事件管理岗负责统筹和协调。</p><p>其次，CNCF还会向全球企业招募会员，例如国内的腾讯云、蚂蚁金服和华为等都是其会员。这些云厂商每年需要向 CNCF 支付一定的费用来维持它在基金会的席位，这其实也是CNCF的重要收入来源。会员是 CNCF 组织形式中非常重要的组成部分，这些厂商和 CNCF 一样也押注在云原生领域，并投入研发的人力来参与到社区的项目中，以获得更广的影响力。</p><p>此外，大使也是社区非常重要的组成部分。大使是 CNCF 非官方的布道师，它们通常是社区的意见领袖，CNCF 借助大使的影响力来传播云原生技术。</p><p>可见，员工、会员以及大使三个角色是 CNCF 最核心的职位。</p><h3>TOC 和 SIG</h3><p>除了上面提到的三类角色以外，由于 CNCF 也非常注重开源社区的贡献，所以，CNCF 还设置了 TOC（Technical Oversight Committee）技术监督委员会小组，TOC 小组的成员主要来自两部分，分别是会员（云厂商）固定席位和社区投票。TOC 是 CNCF 的领导层，负责决策和管理 CNCF 的项目和社区。</p><p>TOC 主要关注 CNCF 的总体战略和管理，对于 CNCF 托管的项目细节，TOC 是很难在代码层面提供指导的。为此，CNCF 还设置了 SIG（Special Interest Group），也就是特别兴趣小组。SIG 是 CNCF 的技术管理机构，它负责制定规范以及监督所有的 CNCF 项目。目前，活跃的 SIG 小组有以下几个。</p><ol>
<li><a href="https://github.com/cncf/tag-security">安全小组</a>：负责云原生访问策略和控制。</li>
<li><a href="https://github.com/cncf/tag-storage">存储小组</a>：负责云原生存储项目标准制定。</li>
<li><a href="https://github.com/cncf/tag-app-delivery">应用交付小组</a>：负责云原生应用交付，包括构建、部署和管理。</li>
<li><a href="https://github.com/cncf/tag-network">网络小组</a>：负责云原生网络例如 API 网关和负载均衡等。</li>
<li><a href="https://github.com/cncf/tag-runtime">运行时小组</a>：负责制定云原生运行时标准。</li>
<li><a href="https://github.com/cncf/tag-contributor-strategy">贡献者策略小组</a>：负责贡献者体验、在可持续性、治理和开放性提供指导。</li>
<li><a href="https://github.com/cncf/tag-observability">可观测性小组</a>：负责云原生可观测性和最佳实践。</li>
<li><a href="https://github.com/cncf/tag-env-sustainability">环境可持续性小组</a>：负责云原生环境可持续性，例如碳排放。</li>
</ol><p>总的来说，CNCF 的组织形式可以用下面这张图来归纳。</p><p><img src="https://static001.geekbang.org/resource/image/d4/3c/d4b1d5cfc73fyyf363fa1396952d2c3c.jpg?wh=1920x1116" alt="图片"></p><h2>项目托管</h2><p>开源项目是 CNCF 的核心资产，比如著名的 Kubernetes、etcd 和 Helm 等项目都是 CNCF 的托管项目。托管项目来自厂商的捐赠，捐赠内容包括源码、商标和网站等和项目相关的内容。</p><p>为了区分项目的成熟度，CNCF 把项目分成了三个阶段，分别是：</p><ol>
<li>Sandbox（沙箱阶段）</li>
<li>Incubating（孵化阶段）</li>
<li>Graduated（毕业阶段）</li>
</ol><p>当一个项目被捐赠时，会首先进入到沙箱阶段。进入沙箱阶段后，CNCF 会给予项目一些宣发资源以及参加云原生大会的机会。经过一段时间的发展后，如果项目的使用人数、贡献者和成熟度符合一定的要求，经过 TOC 的评审，项目会进入到下一个孵化阶段，最后再到毕业阶段。</p><p>由此可见，CNCF 的毕业项目是从所有捐赠项目中层层筛选出来的，它们通常已经非常成熟并且被广泛使用了，它们一般代表了云原生某个领域的事实标准。</p><p>那么，厂商为什么会把自己重金投入的项目免费捐赠给 CNCF 呢？我认为主要的原因有三个。首先，CNCF 基金会作为云原生的风向标，项目被接受意味着 CNCF 对项目的认可。从事开源的团队一般都是大公司团队，这对团队的考核有非常大的帮助，也是团队实施捐赠的动力来源。</p><p>其次，在项目捐赠后，可以通过 CNCF 的影响力吸引更多的用户以及贡献者，在进一步完善开源项目的同时，也增加了核心维护厂商的品牌影响力，促进他们换取更高的商业价值。</p><p>最后，所有捐赠给 CNCF 的项目都有机会成为云原生某个领域的标准，一旦自己所维护的项目成为了标准，其商业价值是不可估量的。</p><p>今天，捐赠已经不再是一种纯粹的开源行为，更多地代表了背后厂商的利益，通过捐赠能够加速项目的发展，最终为项目带来更多商业化的可能性。</p><h2>职业认证</h2><p>职业认证是 CNCF 最重要的板块之一，在为开发者提供认证的同时，CNCF 也能从中获得收入。</p><p>目前，CNCF 的职业认证有以下几个。</p><ol>
<li><a href="https://www.cncf.io/certification/cka/">CKA</a>：Kubernetes 管理员认证</li>
<li><a href="https://www.cncf.io/certification/ckad/">CKAD</a>：Kubernetes 开发者认证</li>
<li><a href="https://www.cncf.io/certification/cks/">CKS</a>：Kubernetes 安全认证</li>
<li><a href="https://www.cncf.io/certification/kcna/">KCNA</a>：Kubernetes 管理员助理认证</li>
<li><a href="https://www.cncf.io/certification/pca/">PCA</a>：Prometheus 管理员认证</li>
<li><a href="https://www.cncf.io/certification/kcsa/">KCSA</a>：Kubernetes 安全助理认证</li>
</ol><p>对于开发者来说，我推荐你参加 CKA 和 CKAD 认证。这两个认证推出的时间长，市场认可度高，在很多 DevOps、SRE 和运维开发工程师的招聘描述上，你都能看到这两个认证的要求。</p><p>职业认证一般是通过远程的方式来进行的，考官会在远程监督考试过程。</p><h2>总结</h2><p>总结一下，这节课，我带你认识了云计算简单的历史发展以及 CNCF 的运作机制。其中，我重点介绍了 CNCF 的组织形式、项目托管和职业认证。</p><p>对于开发者而言，SIG 和 TOC 是我们获取一手信息的渠道。在介绍 SIG 的时候，我已经把每一个 SIG 的主页放在了文稿中，每一个 SIG 的议题都是公开透明的，在例会上，你可以参加讨论相关项目、行业发展和标准相关的内容。通过公开的链接，任何人都可以参加 SIG 例会。</p><p>TOC 的运作过程则相对保密，对于一些技术决策，TOC 一般会公开，但涉及 CNCF 自身运营的决策则不会对外公开。</p><p>如果你想参与到云原生标准的建立中来，项目捐赠是一个非常好的开始，你可以查看<a href="https://github.com/cncf/toc/blob/main/process/project_proposals.md">这份文档</a>来了解如何将项目捐赠给 CNCF。CNCF 在接受到提交后，会定期进行评审，评审通过后，将进入到商标和网站的交割流程，最后完成整个捐赠过程。</p><p>最后，作为从业者，我强烈建议你参加 CNCF 的职业认证。一方面它是对我们技术能力的认可，更重要的是在考试过程中我们可以进一步查缺补漏，深入学习云原生的核心技术。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/62/b8/0e1b655e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>fireshort</span>
  </div>
  <div class="_2_QraFYR_0">“对于开发者来说，我推荐你参加 CKA 和 CKAD 认证。这两个认证推出的时间长，市场认可度高，在很多 DevOps、SRE 和运维开发工程师的招聘描述上，你都能看到这两个认证的要求。”<br><br>好奇去搜了一下，没看到职位描述有写需要这两个认证的……</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这两个认证一般是加分项～</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-13 10:31:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/43/05/3fbf26cf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Noel ZHANG</span>
  </div>
  <div class="_2_QraFYR_0">国内有没有什么关于SRE&#47;devops以及相关领域的线下论坛或者技术研讨会可以参加的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不定期举办，可以多关注 CNCF 的公众号。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-24 21:07:30</div>
  </div>
</div>
</div>
</li>
</ul>