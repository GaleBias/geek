<audio title="结束语 _ Kubernetes：赢开发者赢天下" src="https://static001.geekbang.org/resource/audio/33/ef/335c67ede29d9d7306e581cfff81e2ef.mp3" controls="controls"></audio> 
<p>你好，我是张磊。</p><p>在本专栏一开始，我用了大量的笔墨和篇幅和你探讨了这样一个话题：Kubernetes 为什么会赢？</p><p>而在当时的讨论中，我为你下了这样一个结论：Kubernetes 项目之所以能赢，最重要的原因在于它争取到了云计算生态里的绝大多数开发者。不过，相信在那个时候，你可能会对这个结论有所疑惑：大家不都说 Kubernetes 是一个运维工具么？怎么就和开发者搭上了关系呢？</p><p>事实上，Kubernetes 项目发展到今天，已经成为了云计算领域中平台层当仁不让的事实标准。但这样的生态地位，并不是一个运维工具或者 Devops 项目所能达成的。这里的原因也很容易理解：Kubernetes 项目的成功，是成千上万云计算平台上的开发者用脚投票的结果。而在学习完本专栏之后，相信你也应该能够明白，云计算平台上的开发者们所关心的，并不是调度，也不是资源管理，更不是网络或者存储，他们关心的只有一件事，那就是 Kubernetes 的 API。</p><p>这也是为什么，在 Kubernetes 这个项目里，只要是跟 API 相关的事情，那就都是大事儿；只要是想要在这个社区构建影响力的人或者组织，就一定会在 API 层面展开角逐。这一层 “API 为王”的思路，早已经深入到了 Kubernetes 里每一个 API 对象的每一个字段的设计过程当中。</p><!-- [[[read_end]]] --><p>所以说，Kubernetes 项目的本质其实只有一个，那就是“控制器模式”。这个思想，不仅仅是 Kubernetes 项目里每一个组件的“设计模板”，也是Kubernetes 项目能够将开发者们紧紧团结到自己身边的重要原因。作为一个云计算平台的用户，能够用一个 YAML 文件表达我开发的应用的最终运行状态，并且自动地对我的应用进行运维和管理。这种信赖关系，就是连接 Kubernetes 项目和开发者们最重要的纽带。更重要的是，当这个 API 趋向于足够稳定和完善的时候，越来越多的开发者会自动汇集到这个 API 上来，依托它所提供的能力构建出一个全新的生态。</p><p>事实上，在云计算发展的历史上，像这样一个围绕一个 API 创建出一个“新世界”的例子，已经出现过了一次，这正是 AWS 和它庞大的开发者生态的故事。而这一次 Kubernetes 项目的巨大成功，其实就是 AWS 故事的另一个版本而已。只不过，相比于 AWS 作为基础设施层提供运维和资源抽象标准的故事，Kubernetes 生态终于把触角触碰到了应用开发者的边界，使得应用的开发者可以有能力去关心自己开发的应用的运行状态和运维方法，实现了经典 PaaS 项目很多年前就已经提出、但却始终没能达成的美好愿景。</p><p>这也是为什么我在本专栏里一再强调，Kubernetes 项目里最重要的，是它的“容器设计模式”，是它的 API 对象，是它的 API 编程范式。这些，都是未来云计算时代的每一个开发者需要融会贯通、融化到自己开发基因里的关键所在。也只有这样，作为一个开发者，你才能够开发和构建出符合未来云计算形态的应用。而更重要的是，也只有这样，你才能够借助云计算的力量，让自己的应用真正产生价值。</p><p>而通过本专栏的讲解，我希望你能够真正理解 Kubernetes API 背后的设计思想，能够领悟 Kubernetes 项目为了赢得开发者信赖的“煞费苦心”。更重要的是，当你带着这种“觉悟”再去理解和学习 Kubernetes 调度、网络、存储、资源管理、容器运行时的设计和实现方法时，才会真正触碰到这些机制隐藏在文档和代码背后的灵魂所在。</p><p>所以说，<strong>当你不太理解为什么要学习 Kubernetes 项目的时候，或者，你在学习 Kubernetes 项目感到困难的时候，不妨想象一下 Kubernetes 就是未来的 Linux 操作系统。</strong>在这个云计算以前所未有的速度迅速普及的世界里，Kubernetes 项目很快就会像操作系统一样，成为每一个技术从业者必备的基础知识。而现在，你不仅牢牢把握住了这个项目的精髓，也就是声明式 API 和控制器模式；掌握了这个 API 独有的编程范式，即 Controller 和 Operator；还以此为基础详细地了解了这个项目每一个核心模块和功能的设计与实现方法。那么，对于这个未来云计算时代的操作系统，你还有什么好担心的呢？</p><p>所以说，《深入剖析 Kubernetes 》专栏的结束，其实是你技术生涯全新的开始。我相信你一定能够带着这个“赢开发者赢天下”的启发，在云计算的海洋里继续乘风破浪、一往无前！</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/8d/d9/e968747f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>海蓝</span>
  </div>
  <div class="_2_QraFYR_0">感谢张老师，从使用和原理的角度全面剖析了kubernetes这个未来之星。<br>讲解入木三分，专栏非常超值，建议出书。<br>给你一个大写的赞。<br>给你一个感谢的拥抱。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-24 14:01:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/e4/8b/8a0a6c86.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>haha</span>
  </div>
  <div class="_2_QraFYR_0">特意回来留言：<br>理事会（GB）任命的新成员为：<br><br>张磊，阿里巴巴（@resouer）- 张磊是 Kubernetes 社区的共同维护者，CNCF App Delivery SIG 的联合主席。张磊在阿里巴巴共同领导工程工作，包括 Kubernetes 和大规模集群管理系统。在加入阿里巴巴之前，张磊曾就职于 Hyper_和微软研究院（MSR）。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-03 12:10:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/4b/b2/7f600f9e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>silence</span>
  </div>
  <div class="_2_QraFYR_0">感谢张磊老师。有计划新的课程吗，啥时候能讲讲 k8s 源码就好啦，哈哈哈。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-26 08:54:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/96/50/bde525b1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>北卡</span>
  </div>
  <div class="_2_QraFYR_0">希望张老师能出新的课程。<br>想看kubernetes源码解析哈哈哈哈哈哈</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-02 14:46:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/a5/08/10b18682.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>混沌渺无极</span>
  </div>
  <div class="_2_QraFYR_0">这是从华山之巅往下看到的风景，甚感妙哉！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-24 00:11:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/55/73/0b6351b8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>细雨</span>
  </div>
  <div class="_2_QraFYR_0">老师：<br><br>请问 k8s 本地的开发环境，你是怎么搭建的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: minikube</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-25 10:39:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/e0/99/5d603697.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>MJ</span>
  </div>
  <div class="_2_QraFYR_0">感谢老师👨‍🏫<br><br>老师可否推荐几本入门参考书？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-10 07:18:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/bb/33/5545b42e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Bobo</span>
  </div>
  <div class="_2_QraFYR_0">磊神，五一期间我又来刷了。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-03 07:57:59</div>
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
  <div class="_2_QraFYR_0">最后完结打卡，云时代k8s是操作系统，得开发者得天下! 感谢张老师!</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-24 22:50:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/80/18/87f99236.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小王</span>
  </div>
  <div class="_2_QraFYR_0">磊哥专栏的水平，超过了市面上我所能看到的所有kubernetes的相关书籍，求磊哥出书！！！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-08 22:46:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/20/6e/46/a612177a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Jupiter</span>
  </div>
  <div class="_2_QraFYR_0">很幸运在几天前搜索kubernetes的时候 搜到您的课。课程深入 但是通俗易懂，虽然我还有很多知识点没有理解，还会继续二刷三刷。作为一个CRUD的业务逻辑程序员，最近因为遇到的一些事情 突然顿悟了 想转型DevOps, 而且也突然觉悟了之前在大学学的基础课的重要性。就跟打仗一样，一开始大家都武器装备优良，但是时间久了 最后就赤膊上阵，考验的就是体力了。 所以职业发展也是，最后拼的都是对于基础的掌握 理解和应用。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-28 11:23:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/05/fc/bceb3f2b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>开心哥</span>
  </div>
  <div class="_2_QraFYR_0">赢！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-23 18:18:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/1b/89/b7fae170.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>那迦树</span>
  </div>
  <div class="_2_QraFYR_0">最近听完了k8s的课程，还是感觉学到很多东西，我们项目最近也在整合k8s，遇到很多问题，希望可以多交流</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-26 20:11:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLyfjHfjulibFGPTewSZZHm2M8yfI7BZmO9vLUFoagveCw3DWYDss7y1CecKia7lT5yb9KoAmsya2zg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Goteswille</span>
  </div>
  <div class="_2_QraFYR_0">打卡 结课 谢谢作者 </div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-31 22:13:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/7f/1a/eb8021c3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>追寻云的痕迹</span>
  </div>
  <div class="_2_QraFYR_0">感谢！每一篇文章都是对K8S运作机制非常凝炼的阐述。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-29 08:32:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/6a/c6/9f2fbc17.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>wswcfan</span>
  </div>
  <div class="_2_QraFYR_0">收货满满，求磊神再出本k8s原理相关的书~</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-25 00:43:09</div>
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
  <div class="_2_QraFYR_0">磊哥最后这一篇收官之作画龙点睛，👍🏻</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-24 20:56:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/64/5d/de0536e8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>木木</span>
  </div>
  <div class="_2_QraFYR_0">k8s，一个超大型分布式操作系统，对系统的各种资源做最大化利用.</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-22 15:10:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/87/64/3882d90d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>yandongxiao</span>
  </div>
  <div class="_2_QraFYR_0">嗨，请问，在落地时，公司一般会直接把Kubernetes暴露给开发人员吗？<br>我发现一些公司搞了很多封装，最后Kubernetes就是个容器调度器。跟开发几乎没啥关系了，也只能写写无状态的应用。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-27 16:12:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/ea/91/d2e09361.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>zero</span>
  </div>
  <div class="_2_QraFYR_0">打卡，结课。一个月左右的上下班通勤时间听完了，收货满满！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 太赞了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-03 08:17:30</div>
  </div>
</div>
</div>
</li>
</ul>