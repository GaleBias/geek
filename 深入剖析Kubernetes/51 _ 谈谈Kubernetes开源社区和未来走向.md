<audio title="51 _ 谈谈Kubernetes开源社区和未来走向" src="https://static001.geekbang.org/resource/audio/67/19/679a76276faae0c00e479787d23db319.mp3" controls="controls"></audio> 
<p>你好，我是张磊。今天我和你分享的主题是：谈谈Kubernetes开源社区和未来走向。</p><p>在前面的文章中，我已经为你详细讲解了容器与 Kubernetes项目的所有核心技术点。在今天这最后一篇文章里，我就跟你谈一谈 Kubernetes 开源社区以及 CNCF 相关的一些话题。</p><p>我们知道 Kubernetes 这个项目是托管在 CNCF 基金会下面的。但是，我在专栏最前面讲解容器与 Kubernetes 的发展历史的时候就已经提到过，CNCF 跟 Kubernetes 的关系，并不是传统意义上的基金会与托管项目的关系，CNCF 实际上扮演的，是 Kubernetes 项目的 Marketing 的角色。</p><p>这就好比，本来 Kubernetes 项目应该是由 Google 公司一家维护、运营和推广的。但是为了表示中立，并且吸引更多的贡献者加入，Kubernetes 项目从一开始就选择了由基金会托管的模式。而这里的关键在于，这个基金会本身，就是 Kubernetes 背后的“大佬们”一手创建出来的，然后以中立的方式，对 Kubernetes 项目进行运营和 Marketing。</p><p>通过这种方式，Kubernetes 项目既避免了因为 Google 公司在开源社区里的“不良作风”和非中立角色被竞争对手口诛笔伐，又可以站在开源基金会的制高点上团结社区里所有跟容器相关的力量。而随后 CNCF 基金会的迅速发展和壮大，也印证了这个思路其实是非常正确和有先见之明的。</p><!-- [[[read_end]]] --><p>不过，在 Kubernetes 和 Prometheus 这两个 CNCF 的一号和二号项目相继毕业之后，现在 CNCF 社区的更多职能，就是扮演一个传统的开源基金会的角色，吸纳会员，帮助项目孵化和运转。</p><p>只不过，由于 Kubernetes 项目的巨大成功，CNCF 在云计算领域已经取得了极高的声誉和认可度，也填补了以往 Linux 基金会在这一领域的空白。所以说，你可以认为现在的 CNCF，就是云计算领域里的 Apache ，而它的作用跟当年大数据领域里 Apache 基金会的作用是一样的。</p><p>不过，需要指出的是，<strong>对于开源项目和开源社区的运作来说，第三方基金会从来就不是一个必要条件。</strong>事实上，这个世界上绝大多数成功的开源项目和社区，都来自于一个聪明的想法或者一帮杰出的黑客。在这些项目的发展过程中，一个独立的、第三方基金会的作用，更多是在该项目发展到一定程度后主动进行商业运作的一部分。开源项目与基金会间的这一层关系，希望你不要本末倒置了。</p><p>另外，需要指出的是，CNCF 基金会仅仅负责成员项目的Marketing， 而绝不会、也没有能力直接影响具体项目的发展历程。无论是任何一家成员公司或者是 CNCF 的 TOC（Technical Oversight Committee，技术监督委员会），都没有对 Kubernetes 项目“指手画脚”的权利，除非这位 TOC 本人就是 Kubernetes 项目里的关键人物。</p><p>所以说，真正能够影响 Kubernetes 项目发展的，当然还是 Kubernetes 社区本身。可能你会好奇，<strong>Kubernetes 社区本身的运作方式，又是怎样的呢？</strong></p><p>通常情况下，一个基金会下面托管的项目，都需要遵循基金会本身的管理机制，比如统一的 CI 系统、Code Review流程、管理方式等等。</p><p>但是，在我们这个社区的实际情况，是先有的 Kubernetes，然后才有的 CNCF，并且 CNCF 基金会还是 Kubernetes “一手带大”的。所以，在项目治理这个事情上，Kubernetes 项目早就自成体系，并且发展得非常完善了。而基金会里的其他项目一般各自为阵，CNCF不会对项目本身的治理方法提出过多的要求。</p><p>而说到 <strong>Kubernetes 项目的治理方式，其实还是比较贴近 Google 风格的，即：重视代码，重视社区的民主性。</strong></p><p>首先，Kubernetes 项目是一个没有“Maintainer”的项目。这一点非常有意思，Kubernetes 项目里曾经短时间内存在过 Maintainer 这个角色，但是很快就被废弃了。取而代之的，则是 approver+reviewer 机制。这里具体的原理，是在 Kubernetes 的每一个目录下，你都可以添加一个 OWNERS 文件，然后在文件里写入这样的字段：</p><pre><code>approvers:
- caesarxuchao
reviewers:
- lavalamp
labels:
- sig/api-machinery
- area/apiserver
</code></pre><p>比如，上面这个例子里，approver 的 GitHub ID 就是caesarxuchao （Xu Chao），reviewer 就是 lavalamp。这就意味着，任何人提交的Pull Request（PR，代码修改请求），只要修改了这个目录下的文件，那么就必须要经过 lavalamp 的 Code Review，然后再经过caesarxuchao的 Approve 才可以被合并。当然，在这个文件里，caesarxuchao 的权力是最大的，它可以既做 Code Review，也做最后的 Approve。但， lavalamp 是不能进行 Approve 的。</p><p>当然，无论是 Code Review 通过，还是 Approve，这些维护者只需要在 PR下面Comment /lgtm 和 /approve，Kubernetes 项目的机器人（k8s-ci-robot）就会自动给该 PR 加上 lgtm 和 approve标签，然后进入 Kubernetes 项目 CI 系统的合并队列，最后被合并。此外，如果你要对这个项目加标签，或者把它 Assign 给其他人，也都可以通过 Comment 的方式来进行。</p><p>可以看到，在上述整个过程中，代码维护者不需要对Kubernetes 项目拥有写权限，就可以完成代码审核、合并等所有流程。这当然得益于 Kubernetes 社区完善的机器人机制，这也是 GitHub 最吸引人的特性之一。</p><p>顺便说一句，很多人问我，<strong>GitHub 比 GitLab 或者其他代码托管平台强在哪里？</strong>实际上， GitHub 庞大的API 和插件生态，才是这个产品最具吸引力的地方。</p><p>当然，当你想要将你的想法以代码的形式提交给 Kubernetes项目时，除非你的改动是 bugfix 或者很简单的改动，否则，你直接提交一个 PR 上去，是大概率不会被 Approve 的。<strong>这里的流程，</strong>一定要按照我下面的讲解来进行：</p><ol>
<li>
<p>在 Kubernetes 主库里创建 Issue，详细地描述你希望解决的问题、方案，以及开发计划。而如果社区里已经有相关的Issue存在，那你就必须要在这里把它们引用过来。而如果社区里已经存在相同的 Issue 了，你就需要确认一下，是不是应该直接转到原有 issue 上进行讨论。</p>
</li>
<li>
<p>给 Issue 加上与它相关的 SIG 的标签。比如，你可以直接 Comment /sig node，那么这个 Issue 就会被加上 sig-node 的标签，这样 SIG-Node的成员就会特别留意这个 Issue。</p>
</li>
<li>
<p>收集社区对这个 Issue 的信息，回复 Comment，与 SIG 成员达成一致。必要的时候，你还需要参加 SIG 的周会，更好地阐述你的想法和计划。</p>
</li>
<li>
<p>在与 SIG 的大多数成员达成一致后，你就可以开始进行详细的设计了。</p>
</li>
<li>
<p>如果设计比较复杂的话，你还需要在 Kubernetes 的<a href="https://github.com/kubernetes/community/tree/master/contributors/design-proposals">设计提议目录</a>（在Kubernetes Community 库里）下提交一个 PR，把你的设计文档加进去。这时候，所有关心这个设计的社区成员，都会来对你的设计进行讨论。不过最后，在整个 Kubernetes 社区只有很少一部分成员才有权限来 Review 和 Approve 你的设计文档。他们当然也被定义在了这个目录下面的 OWNERS 文件里，如下所示：</p>
</li>
</ol><pre><code>reviewers:
  - brendandburns
  - dchen1107
  - jbeda
  - lavalamp
  - smarterclayton
  - thockin
  - wojtek-t
  - bgrant0607
approvers:
  - brendandburns
  - dchen1107
  - jbeda
  - lavalamp
  - smarterclayton
  - thockin
  - wojtek-t
  - bgrant0607
labels:
  - kind/design
</code></pre><p>这几位成员，就可以称为社区里的“大佬”了。不过我在这里要提醒你的是，“大佬”并不一定代表水平高，所以你还是要擦亮眼睛。此外，Kubernetes 项目的几位创始成员，被称作 Elders（元老），分别是jbeda、bgrant0607、brendandburns、dchen1107和thockin。你可以查看一下这个列表与上述“大佬”名单有什么不同。</p><ol>
<li>
<p>上述 Design Proposal被合并后，你就可以开始按照设计文档的内容编写代码了。这个流程，才是正常大家所熟知的编写代码、提交 PR、通过 CI 测试、进行Code Review，然后等待合并的流程。</p>
</li>
<li>
<p>如果你的 feature 是需要要在 Kubernetes 的正式 Release 里发布上线的，那么你还需要在<a href="https://github.com/kubernetes/enhancements/blob/master/keps">Kubernetes Enhancements</a>这个库里面提交一个 KEP（即Kubernetes Enhancement Proposal）。这个 KEP 的主要内容，是详细地描述你的编码计划、测试计划、发布计划，以及向后兼容计划等软件工程相关的信息，供全社区进行监督和指导。</p>
</li>
</ol><p>以上内容，就是 Kubernetes 社区运作的主要方式了。</p><h1>总结</h1><p>在本篇文章里，我为你详细讲述了 CNCF 和 Kubernetes 社区的关系，以及 Kubernetes 社区的运作方式，希望能够帮助你更好地理解这个社区的特点和它的先进之处。</p><p>除此之外，你可能还听说过 Kubernetes 社区里有一个叫作Kubernetes Steering Committee的组织。这个组织，其实也是属于<a href="https://github.com/kubernetes/community">Kubernetes Community 库</a>的一部分。这个组织成员的主要职能，是对 Kubernetes 项目治理的流程进行约束和规范，但通常并不会直接干涉 Kubernetes 具体的设计和代码实现。</p><p>其实，到目前为止，Kubernetes 社区最大的一个优点，就是把“搞政治”的人和“搞技术”的人分得比较清楚。相信你也不难理解，这两种角色在一个活跃的开源社区里其实都是需要的，但是，如果这两部分人发生了大量的重合，那对于一个开源社区来说，恐怕就是个灾难了。</p><h1>思考题</h1><p>你能说出 Kubernetes 社区同 OpenStack 社区相比的不同点吗？你觉得这两个社区各有哪些优缺点呢？</p><p>感谢你的收听，欢迎你给我留言，也欢迎分享给更多的朋友一起阅读。</p><p><img src="https://static001.geekbang.org/resource/image/96/25/96ef8576a26f5e6266c422c0d6519725.jpg?wh=1110*659" alt=""></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/57/a5/8adda747.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>神灯</span>
  </div>
  <div class="_2_QraFYR_0">终于看完一遍了，接下来还要看第二遍，第三遍！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢支持</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-25 21:53:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/27/4b/170654bb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>千寻</span>
  </div>
  <div class="_2_QraFYR_0">完结撒花。谢谢张老师这段时间的付出。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-20 23:40:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/bc/2a/00a3d488.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>gl328518397</span>
  </div>
  <div class="_2_QraFYR_0">打卡： 2020年5月6日。感谢张磊老师，学习本专栏受益匪浅。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-06 23:06:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/99/bb/90d97247.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>shaobo</span>
  </div>
  <div class="_2_QraFYR_0">请教下作者，对开发者来说，kubernetes除了ops有帮助外，对个人的发展还有哪些？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Kubernetes 是开发者工具，重要的事情只说一遍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-19 08:55:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/9a/0f/da7ed75a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>芒果少侠</span>
  </div>
  <div class="_2_QraFYR_0">没参加过社区，以后有机会一定要！提升自我的好地方</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-11 02:38:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/6a/d7/1b80a01b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我的猫那</span>
  </div>
  <div class="_2_QraFYR_0">看完一遍，留个纪念吧，还有好多不懂，随时复习！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-03 15:29:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/a0/a7/db7a7c50.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>送普选</span>
  </div>
  <div class="_2_QraFYR_0">全部读完打卡，感谢张磊老师这段的付出，也感谢评论区的各位同学的评论。这是自己在极客时间订阅的第一门课，自己能全部读完，对自己也很有帮助，物超所值，谢谢。现在DevOps和云原生时代，互联网公司的开发角色也要了解上下游，架构师角色要了解网络，系统，开发，运维等多方面的知识，不能只从设计和编码的角度来看待Kubernetes和容器云，谢谢。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-24 20:15:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLK4NPyaFDl4rzrd4A9z42tQDibFLdCicbrAdyUa5P2B5u8UCUBenpHX7VgUCibvHvhpza4icMBKVnhmA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>学无止境</span>
  </div>
  <div class="_2_QraFYR_0">终于耐心的看完一遍了，中途有一段真的看不下去了，东西太多，需要好好的消化整理，并动手实践才能真正理解这个好东西，感谢张老师的耐心讲解和付出，收获颇多，估计还要二刷，三刷！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-30 09:46:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/d2/2f/04882ff8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>龙坤</span>
  </div>
  <div class="_2_QraFYR_0">&quot;KEP 的主要内容，是详细地描述你的编码计划、测试计划、发布计划，以及向后兼容计划等软件工程相关的信息，供全社区进行监督和指导&quot; 读到这句话，真的觉得贡献开源社区力量的人确实不容易，这些任务平时工作都不能很好的完成，在开源社区上还需要详细的做出来，并且和大家一起研讨，自主能动性真重要</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-26 14:41:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/51/0d/fc1652fe.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>James</span>
  </div>
  <div class="_2_QraFYR_0">第一次看，感觉好复杂。再看一次= =</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-03 23:10:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/dd/09/feca820a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>helloworld</span>
  </div>
  <div class="_2_QraFYR_0">精读了前半部分, 听完了后半部分, 整体过一遍了, 文章写的是真的好!</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-20 11:36:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/2c/cc/34859bbe.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>核子飞弹</span>
  </div>
  <div class="_2_QraFYR_0">看完打卡，接下来结合实践再看一遍</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-18 14:44:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/bb/f6/af833125.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Vinsec</span>
  </div>
  <div class="_2_QraFYR_0">还没跟上来，完结打个卡。订阅的诸多专栏中，质量或许是最高的一个。能把技术深入浅出式地输出对于技术人来讲很可贵。希望将来有能力也可以给这种规模的开源项目提PR:)</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-20 22:19:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/e5/02/ea609428.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>wzm1990</span>
  </div>
  <div class="_2_QraFYR_0">完结撒花🎉</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-04 09:45:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/3a/f8/c1a939e7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>君哥聊技术</span>
  </div>
  <div class="_2_QraFYR_0">擦亮眼睛，哈哈，👍</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-20 08:32:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/8e/76/6d55e26f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张晓辉</span>
  </div>
  <div class="_2_QraFYR_0">打卡，学习完了。收益匪浅，后面还要复习。谢谢张老师的分享。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-21 07:37:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/bd/43/c4ccb0ba.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Jinghua</span>
  </div>
  <div class="_2_QraFYR_0">谢谢老师的分享，十分受教。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-27 20:45:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/96/fb/46f1a117.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>战栗的生灵</span>
  </div>
  <div class="_2_QraFYR_0">感谢张磊老师的分享。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-20 17:55:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/c1/60/fc3689d0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小谢同学</span>
  </div>
  <div class="_2_QraFYR_0">请问张老师，能否这样理解，若使用k8s+service mesh 来构建应用，那原来的spring or dubbo框架则只需要负责服务间的高效通信即可(例如rpc调用，序列化）这些工作？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，可惜dubbo也要搞mesh，摊手</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-15 12:39:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83eo6TmyGF3wMIRLx3lPWOlBWusQCxyianFvZvWeW6hYCABLqEow3p7tGc6XgnqUPVvf6Cbj2KUYQIiag/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>孙健波</span>
  </div>
  <div class="_2_QraFYR_0">精彩的课程！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-22 13:13:38</div>
  </div>
</div>
</div>
</li>
</ul>