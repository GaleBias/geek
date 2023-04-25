<audio title="01｜技术概览：eBPF的发展历程及工作原理" src="https://static001.geekbang.org/resource/audio/e0/ca/e08331f01b1319319e023e8891f007ca.mp3" controls="controls"></audio> 
<p>你好，我是倪朋飞。</p><p>在正式介绍 eBPF 的使用方法和具体应用之前，我会用两讲的内容，带你了解eBPF的技术脉络和学习路线，为你后面的学习做好准备。</p><p>在开篇词里，我带你一起了解了这门课的设计思路和主要内容，也简单介绍了 eBPF 的主要应用场景，包括故障诊断、网络优化、安全控制、性能监控等。那你可能就要问了：为什么 eBPF 可以应用到这么广泛的领域呢？</p><p>eBPF 广泛的应用场景和强大的功能，跟它的发展历程、基本原理密切相关。那么，eBPF 的发展历程是什么样的？它又是如何在确保安全的前提下，允许非内核开发者去扩展内核的功能的呢？今天，我就带你一起来看看这些问题。</p><h2>eBPF 的发展历程是什么样的?</h2><p>在开篇词中，我曾经提到，eBPF 是从 BPF (Berkeley Packet Filter) 技术扩展而来的。而说起 BPF，它的历史就更悠长了。</p><p>早在 1992 年的 USENIX 会议上，Steven McCanne 和 Van Jacobson 发布的论文“<a href="https://www.tcpdump.org/papers/bpf-usenix93.pdf">The BSD Packet Filter: A New Architecture for User-level Packet Capture</a>” 就为 BSD 操作系统带来了革命性的包过滤机制 BSD Packet Filter（简称为 BPF），这比当时最先进的数据包过滤技术还快 20 倍。为什么性能这么好呢？这主要得益于 BPF 的两大设计：</p><!-- [[[read_end]]] --><ul>
<li>第一，内核态引入一个新的虚拟机，所有指令都在内核虚拟机中运行。</li>
<li>第二，用户态使用 BPF 字节码来定义过滤表达式，然后传递给内核，由内核虚拟机解释执行。</li>
</ul><p>这就使得包过滤可以直接在内核中执行，避免了向用户态复制每个数据包，从而极大提升了包过滤的性能，进而被各大操作系统广泛接受。BPF 最初的名字 BSD Packet Filter ，也被作者的工作单位名所替代，变成了 Berkeley Packet Filter（很巧的是，还是简称 BPF）。</p><p>在 BPF 诞生五年后，Linux 2.1.75 首次引入了 BPF 技术，随后&nbsp;BPF 开始了不温不火的发展历程。其中，Linux 3.0 中增加的 BPF 即时编译器可以算是一个最重大的更新了。它替换掉了原本性能更差的解释器，进一步优化了 BPF 指令运行的效率。但直到此时，BPF 的应用还是仅限于网络包过滤这个传统的领域中。</p><p>时间到了 2014 年。为了研究新的软件定义网络方案，Alexei Starovoitov 为 BPF 带来了第一次革命性的更新，将 BPF 扩展为一个通用的虚拟机，也就是 eBPF。eBPF 不仅扩展了寄存器的数量，引入了全新的 BPF 映射存储，还在 4.x 内核中将原本单一的数据包过滤事件逐步扩展到了内核态函数、用户态函数、跟踪点、性能事件（perf_events）以及安全控制等。</p><p><strong>eBPF 的诞生是 BPF 技术的一个转折点，使得 BPF 不再仅限于网络栈，而是成为内核的一个顶级子系统。</strong></p><p>在内核发展的同时，eBPF 繁荣的生态也进一步促进了 eBPF 的蓬勃发展。这其中，最典型的就是 iovisor 带来的 BCC、bpftrace 等工具，成为 eBPF 在跟踪和排错领域的最佳实践。由于 eBPF 无需修改内核源码和重新编译内核就可以扩展内核的功能，Cilium、Katran、Falco 等一系列基于 eBPF 优化网络和安全的开源项目也逐步诞生。并且，越来越多的开源和商业解决方案开始借助 eBPF，优化其网络、安全以及观测的性能。比如，最流行的网络解决方案之一 Calico，就在最近的版本中引入了 <a href="https://www.tigera.io/blog/introducing-the-calico-ebpf-dataplane/">eBPF 数据面网络</a>，大大提升了网络的性能。</p><p>为了帮你更好地理解 eBPF 的发展历程，我把 eBPF 诞生以来的发展过程整理成了一张图片：</p><p><img src="https://static001.geekbang.org/resource/image/b4/ff/b44562381748de369b50403219c0d1ff.jpg?wh=2284x7454" alt="" title="eBPF 发展历程"></p><p>直到今天，eBPF 依然是内核社区最活跃的子模块之一，还处在一个快速发展的过程中。可以说，<strong>eBPF 开启的创新才刚刚开始</strong>，在未来我们会看到更多的创新案例。正是为了确保每个 eBPF 学习者不掉队，我们把这门课设计成了动态发布的形式，带你随时跟踪这些最新的发展和案例。</p><p>了解了 eBPF 的诞生过程后，还有一点需要你留意：在内核社区的开发讨论中，通常还是使用 BPF 这个缩略词，而在很多应用的文档中可能会倾向使用 eBPF。其实它们的含义是一样的，都是指扩展版的 BPF。</p><h2>eBPF 是怎么工作的?</h2><p>eBPF 程序并不像常规的线程那样，启动后就一直运行在那里，它需要事件触发后才会执行。这些事件包括系统调用、内核跟踪点、内核函数和用户态函数的调用退出、网络事件，等等。借助于强大的内核态插桩（kprobe）和用户态插桩（uprobe），eBPF 程序几乎可以在内核和应用的任意位置进行插桩。</p><p>看到这个令人惊叹的能力，你一定有疑问：这会不会像内核模块一样，一个异常的 eBPF 程序就会损坏整个内核的稳定性呢？其实，<strong>确保安全和稳定一直都是 eBPF 的首要任务</strong>，不安全的 eBPF 程序根本就不会提交到内核虚拟机中执行。</p><p>Linux 内核是如何实现 eBPF 程序的安全和稳定的呢？其实很简单，我带你看个 eBPF 程序的执行过程，你就明白了。</p><p>如下图（图片来自<a href="https://www.brendangregg.com/ebpf.html">brendangregg.com</a>）所示，通常我们借助 <a href="https://llvm.org/">LLVM</a> 把编写的 eBPF 程序转换为 BPF 字节码，然后再通过 bpf 系统调用提交给内核执行。内核在接受 BPF 字节码之前，会首先通过验证器对字节码进行校验，只有校验通过的 BPF 字节码才会提交到即时编译器执行。</p><p><img src="https://static001.geekbang.org/resource/image/a7/6a/a7165eea1fd9fc24090a3a1e8987986a.png?wh=1500x550" alt="图片" title="eBPF 程序执行过程"></p><p>如果 BPF 字节码中包含了不安全的操作，验证器会直接拒绝 BPF 程序的执行。比如，下面就是一些典型的验证过程：</p><ul>
<li>只有特权进程才可以执行 bpf 系统调用；</li>
<li>BPF 程序不能包含无限循环；</li>
<li>BPF 程序不能导致内核崩溃；</li>
<li>BPF 程序必须在有限时间内完成。</li>
</ul><p>BPF 程序可以利用 BPF 映射（map）进行存储，而用户程序通常也需要通过 BPF 映射同运行在内核中的 BPF 程序进行交互。如下图（图片来自<a href="https://ebpf.io/what-is-ebpf">ebpf.io</a>）所示，在性能观测中，BPF 程序收集内核运行状态存储在映射中，用户程序再从映射中读出这些状态。</p><p><img src="https://static001.geekbang.org/resource/image/53/dd/53af7f7db99c3ca57f981f00303949dd.png?wh=1401x733" alt="图片" title="BPF 映射"></p><p>可以看到，eBPF 程序的运行需要历经编译、加载、验证和内核态执行等过程，而用户态程序则需要借助 BPF 映射来获取内核态 eBPF 程序的运行状态。</p><h2>eBPF 是万能的吗?</h2><p>看到这里，你是不是因为 eBPF 在扩展内核功能上的强大能力而兴奋不已？我猜你已经迫不及待想要体验一下了。不过，在你体验之前，我还要提醒你一点：eBPF 并不是万能的，它也有很多的局限性。下面是一些最常见的&nbsp;eBPF 限制：</p><ul>
<li>eBPF 程序必须被验证器校验通过后才能执行，且不能包含无法到达的指令；</li>
<li>eBPF 程序不能随意调用内核函数，只能调用在 API 中定义的辅助函数；</li>
<li>eBPF 程序栈空间最多只有 512 字节，想要更大的存储，就必须要借助映射存储；</li>
<li>在内核 5.2 之前，eBPF 字节码最多只支持 4096 条指令，而 5.2 内核把这个限制提高到了 100 万条；</li>
<li>由于内核的快速变化，在不同版本内核中运行时，需要访问内核数据结构的 eBPF 程序很可能需要调整源码，并重新编译。</li>
</ul><p>此外，虽然 Linux 内核很早就已经支持了 eBPF，但很多新特性都是在 4.x 版本中逐步增加的，具体你可以看下<a href="https://github.com/iovisor/bcc/blob/master/docs/kernel-versions.md#main-features">这个链接</a>。所以，想要稳定运行 eBPF 程序，<strong>内核版本至少需要 4.9 或者更新</strong>。而在开发和学习 eBPF 时，<strong>为了体验最新的 eBPF 特性，我推荐使用更新的 5.x 内核</strong>。在这门课后面的内容中，我还会给你详细讲解开发环境的搭建步骤，以及推荐的 Linux&nbsp;发行版。</p><h2>小结</h2><p>今天，我带你一起探索了 eBPF 技术的发展历程，并梳理了 eBPF 的工作原理。</p><p>eBPF 是从 BPF 技术扩展而来的。BPF 出现后，一直都是网络数据包过滤的核心，但直到 eBPF 诞生前，BPF 都仅用于包过滤这个场景中。eBPF 的诞生是 BPF 技术的一个转折点，使它的应用范围逐步从包过滤扩展到内核函数、用户函数、跟踪点、性能事件、安全控制等全新的领域中。而这也进一步催生了Cilium、Katran、Falco 等一大批基于 eBPF 构建的网络和安全解决方案，形成了繁荣的 eBPF 生态。</p><p>eBPF 程序以内核事件触发的方式运行，并且其运行过程包括编译、加载、验证和内核态执行等。为了保护内核的安全和稳定，如果编译后 BPF 字节码中包含了不安全的操作，验证阶段会直接拒绝 BPF 程序的执行。</p><p>不过，需要提醒你的是：为了确保安全和稳定，eBPF 程序也有很多的限制，这是你在后续的学习过程中需要特别留心的。</p><h2>思考题</h2><p>在这一讲的最后，想和你交流的问题是：在之前的学习和工作中，你有没有使用过 eBPF 呢？如果使用过，你又用 eBPF 解决过哪些实际的问题呢？期待你在留言区分享，并和我交流讨论。</p><p>今天的内容到这里就结束了，欢迎把它分享给你的同事和朋友，我们下一讲见。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/71/c3/09e22c1d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>大卫李</span>
  </div>
  <div class="_2_QraFYR_0">非常高兴看到eBPF技术能在极客时间开学习专栏了，算是中文圈的一个里程碑！本人也在一直学习bpf技术（也简单写了发展史：https:&#47;&#47;davidlovezoe.club&#47;bpf ），希望跟倪老师及大家共勉。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 欢迎在留言区跟同学们分享学习中的问题和经验。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-17 20:00:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/24/f2/79/b2012f53.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>余生</span>
  </div>
  <div class="_2_QraFYR_0">极客上特别喜欢的几位大咖老师，倪朋飞、陶辉、刘超、彭东！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢对专栏的支持，我们一起加油！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-18 13:31:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/24/59/e9867100.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Unknown</span>
  </div>
  <div class="_2_QraFYR_0">bcc tools 工具都很好用，比如Diffie-Hellman密钥交换算法使得就算有key都无法解密，不过如果通过bcc tools中的sslsniff能直接查看了，对调试来说挺方便的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯嗯，是的，谢谢分享。欢迎在学习过程中分享更多的实践经验！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-17 23:24:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/f2/f5/b82f410d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Unknown element</span>
  </div>
  <div class="_2_QraFYR_0">老师我看您在评论区说ebpf程序是在加载到内核之前验证，那文中的执行顺序是不是不太准确？应该是 编译 验证 加载 内核执行？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这儿可能有一点说的不是特别清楚，“加载”可能对应两个操作：<br>1）第一个是用户态程序通过BPF系统调用加载字节码；<br>2）第二个是内核态收到这个系统调用后，再执行验证+加载的过程。<br><br>所以，对内核来说，“验证+加载”是在用户态程序调用“加载”操作之后的执行步骤。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-09 08:36:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/9f/fc/0232f005.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我要收购腾讯</span>
  </div>
  <div class="_2_QraFYR_0">用过 bcc profile + crd 的方式给公司内部的k8s控制台开发过一键生成火焰图的功能, （全套的可观测性部署不太适合开发测试用集群）</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢分享！欢迎在学习过程中分享更多的学习和实践经验！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-21 13:15:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/e6/ee/8fdbd5db.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Damoncui</span>
  </div>
  <div class="_2_QraFYR_0">好激动！目前正在看翻译版的《bpf之巅》，配合食用简直快乐到不行～英文版吃起来太痛苦……<br><br>选对课程、教程基本节约了一半时间！<br><br>希望和各类牛人一起加油进步！<br><br>ebpf一定会越来越火🔥～</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯嗯，这本书也不错，我们一起加油！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-18 22:01:27</div>
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
  <div class="_2_QraFYR_0">刚来到极客时间买的早期课程就有倪老师的Linux优化实战，看过多遍，大神必出精品！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢对专栏的支持，我们一起加油！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-19 10:36:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/60/36/1848c2b7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>dovefi</span>
  </div>
  <div class="_2_QraFYR_0">使用过bcc-tools 中的filetop cachetop等工具分析过缓存占用的问题，第一次接触到ebpf工具，同事的大佬也有使用systemtap解决过很多问题</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢分享实践经验👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-28 16:34:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/d0/d0/a6c6069d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>坚</span>
  </div>
  <div class="_2_QraFYR_0">倪老师好，我是做嵌入式开发，需要自己配置内核，在搭建bpf的使用环境的时候可以也提一下需要开启内核哪些配置吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯嗯，我们第03讲会详细介绍开发环境的搭建的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-18 08:42:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_07f1e3</span>
  </div>
  <div class="_2_QraFYR_0">曾利用bcc解决过容器DNS请求监控的问题场景，但是正如倪老师文末所提及的eBPF与内核版本之间的限制关系，导致方案最后没被通过，但也算是一次不错的实践体验。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢分享，欢迎在后续的学习过程中分享更多的经验！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-28 16:23:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/92/7c/12c571b6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Slience-0°C</span>
  </div>
  <div class="_2_QraFYR_0">老师，eBPF可以做控制操作的功能？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-15 14:27:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/5f/43/3799a0f3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>magina</span>
  </div>
  <div class="_2_QraFYR_0">看完没懂“内核态引入一个新的虚拟机，所有指令都在内核虚拟机中运行”，怎么体现在文中的图上？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-23 14:56:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/17/18/e4382a8e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>有识之士</span>
  </div>
  <div class="_2_QraFYR_0">开发 eBPF 程序，如何进行调试？类似 go、java 这种高级语言可以方便 debug 一样的？ </div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-22 15:21:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/23/13/bd/3e86e791.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>十七</span>
  </div>
  <div class="_2_QraFYR_0">嵌入式系统ebpf是否适合使用?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 要看系统的，如果是Linux并且内核版本比较新，那应该是没问题的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-04 09:23:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/d0/d0/a6c6069d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>坚</span>
  </div>
  <div class="_2_QraFYR_0">老师，我想问一下eBPF和perf有关系吗？还是说这是两个东西</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 他们都可以用来跟踪和排查内核问题，但ebpf功能更完善，还可以自己写程序自定义处理逻辑。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-17 22:37:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/2e/0d/72a65e1c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>helloworld</span>
  </div>
  <div class="_2_QraFYR_0">非常高兴看到这样的课程。我这里工作实践中用个疑问，我们一般遇到的操作系统低版本的比较多，比如centos内核3.1x的，怎么能做到在低版本适配呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 很多ebpf特性在老版本的内核都是不支持的，所以基于新特性开发的程序有可能是没法直接适配的（一般重新开发一个或者换成systemtap等工具可能更容易）。具体内核版本中支持的特性列表可以参考https:&#47;&#47;github.com&#47;iovisor&#47;bcc&#47;blob&#47;master&#47;docs&#47;kernel-versions.md</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-15 07:57:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/53/db/858337e3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Ethan Liu</span>
  </div>
  <div class="_2_QraFYR_0">eBPF程序执行需要进行验证，不影响性能吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 只是在加载到内核之前进行验证，加载之后就不需要了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-20 20:35:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/82/de/7f6cae1f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>娟</span>
  </div>
  <div class="_2_QraFYR_0">bpf很早就有一些了解，觉得那些语法挺好玩，跟着大牛一起进步，系统化了解了解ebpf。希望结束的时候能很有收获。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 加油！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-20 20:19:01</div>
  </div>
</div>
</div>
</li>
</ul>