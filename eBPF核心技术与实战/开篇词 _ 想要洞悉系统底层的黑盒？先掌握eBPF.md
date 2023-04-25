<audio title="开篇词 _ 想要洞悉系统底层的黑盒？先掌握eBPF" src="https://static001.geekbang.org/resource/audio/89/da/890edcdce068yye89395d96e56b018da.mp3" controls="controls"></audio> 
<p>你好，我是倪朋飞，一名云计算从业者。</p><p>自从十年前参加工作以来，我一直都在云计算领域工作，特别专注于包括虚拟化、软件定义网络、容器等在内的云计算基础设施领域。</p><p>我其实是极客时间的老朋友了，曾在几年前开过<a href="https://time.geekbang.org/column/intro/100020901?tab=catalog">《Linux 性能优化实战》</a>专栏。这门课聚焦于 Linux 性能优化技术，从系统底层原理、性能指标再到实际工作的优化技巧，带你准确分析和优化 Linux 性能问题。</p><p>这门课上线后，同学们的交流热情完全出乎我的意料，也激励着我把专栏的篇幅延长了10多篇。同时，我也注意到，很多同学在学习涉及系统底层知识较多的模块时掉了队，特别是面对系统内核的原理时有些畏惧的心理。</p><p>所以，今天我又给你带来了一门全新的课程。这门课的主要目标就是带你利用 eBPF 去洞悉内核的运行状态，并去解决性能优化、网络观测、安全控制等实际生产环境中的问题。在 eBPF 的助力下，你并不需要成为内核开发者，也可以掌控内核的运行状态。</p><h2>为什么要学习 eBPF？</h2><p>其实，我与 eBPF 的“初次接触”，还要从最流行的网络抓包和分析工具 tcpdump 开始说起。</p><p>我相信你和我一样，在一开始学习 TCP/IP 网络原理时，大量借助了 tcpdump 来了解网络协议的工作原理。而在实际的工作中，大多数网络问题的排查也是借助了 tcpdump 才能了解到网络层面上到底发生了什么事情。</p><!-- [[[read_end]]] --><p>后来，在排查断断续续的网络丢包问题时，我才发现只有 tcpdump 是不够的。tcpdump 只能告诉你网络上传输了哪些包，至于为什么这么传输，它就无能为力了。所以，当时我也搜索了大量的网络资料，偶然发现了 <a href="https://github.com/iovisor/bcc">BCC</a> 这个工具，并借助它顺利解决了很多类似的网络问题。自此以后，网络问题就不再是我的心魔。</p><p><strong>这里我要说的是，tcpdump 和 BCC 之所以这么高效强大，都是得益于 BPF/eBPF 技术。</strong></p><p>eBPF 是什么呢？ 从它的全称“扩展的伯克利数据包过滤器 (Extended Berkeley Packet Filter)” 来看，它是一种数据包过滤技术，是从 BPF (Berkeley Packet Filter) 技术扩展而来的。</p><p>BPF 提供了一种在内核事件和用户程序事件发生时安全注入代码的机制，这就让非内核开发人员也可以对内核进行控制。随着内核的发展，BPF 逐步从最初的数据包过滤扩展到了网络、内核、安全、跟踪等，而且它的功能特性还在快速发展中，这种扩展后的 BPF 被简称为 eBPF（相应的，早期的 BPF 被称为经典 BPF，简称 cBPF）。实际上，现代内核所运行的都是 eBPF，如果没有特殊说明，内核和开源社区中提到的 BPF 等同于 eBPF（在我们的专栏里，它们的含义也完全相同）。</p><p>我想你已经知道，在 eBPF 之前，内核模块是注入内核的最主要机制。由于缺乏对内核模块的安全控制，内核的基本功能很容易被一个有缺陷的内核模块破坏。<strong>而 eBPF 则借助即时编译器（JIT），在内核中运行了一个虚拟机，保证只有被验证安全的 eBPF 指令才会被内核执行。</strong>同时，因为 eBPF 指令依然运行在内核中，无需向用户态复制数据，这就大大提高了事件处理的效率。</p><p>正是由于这些突出的特性，eBPF 现如今已经在故障诊断、网络优化、安全控制、性能监控等领域获得大量应用。比如，Facebook 开源的高性能网络负载均衡器 <a href="https://github.com/facebookincubator/katran">Katran</a>、Isovalent 开源的容器网络方案 <a href="https://cilium.io">Cilium</a> ，以及著名的内核跟踪排错工具 <a href="https://github.com/iovisor/bcc">BCC</a> 和 <a href="https://github.com/iovisor/bpftrace">bpftrace</a> 等，都是基于 eBPF 技术实现的。</p><p>下图（来自 <a href="https://ebpf.io">ebpf.io</a>）是对eBPF 技术及其应用的一个概览：</p><p><img src="https://static001.geekbang.org/resource/image/7d/53/7de332b0fd6dc10b757a660305a90153.png?wh=1500x769" alt="图片" title="eBPF 技术概览"></p><p>可以说，如果你想洞悉内核的运行状态，优化内核网络性能，控制诸如容器等应用程序的安全，那么eBPF 就是一个你必须要掌握的技能。</p><h2>要掌握 eBPF 是不是得先成为内核开发者？</h2><p>到这里，我已经带你初步了解了 eBPF 强大的功能特性，我想你应该对进一步深入学习 eBPF 迫不及待了吧！不过，我猜你也有可能像 5 年前的我一样，在看到 eBPF 需要运行在内核中，并且还涉及到一些内核的编程开发时，心里有点打退堂鼓。那么，我们是不是需要先成为内核开发者，才可以掌握 eBPF 呢？</p><p>实际上，前面我提到的 BCC、bpftrace 等一系列的开源项目已经提供了大量工具，可以帮你解决像故障诊断、性能监控、安全控制等绝大部分场景中的问题。而在你需要开发新的 eBPF 程序时，内核社区提供的 <a href="https://github.com/libbpf/libbpf">libbpf</a> 库不仅帮你避免了直接调用内核函数，而且还提供了跨内核版本的兼容性（即一次编译到处执行，简称 CO-RE）。</p><p>所以，掌握 eBPF 并不需要掌握内核开发。我认为，学习最快的方法就是理解原理的同时配合大量的实践，eBPF 也不例外。下面这三点是学习 eBPF 的重中之重：</p><p><img src="https://static001.geekbang.org/resource/image/dc/2e/dcd84984b168a5534d69a445b08c692e.jpg?wh=1920x1443" alt="图片"></p><p>只要理解了 eBPF 的基本原理，掌握了 eBPF 的运行机制和核心的编程接口，再结合大量的实践技巧，你也可以掌握 eBPF，并把它应用到真实的工作场景中。而这门课会针对不同场景，把这三个方面给你讲清楚。下面我们来具体看看这门课的设计思路。</p><h2>这门课是怎么设计的？</h2><p>为了帮你吃透 eBPF，在理解 eBPF 的同时更好地把它用起来，我会以案例驱动的思路，给你讲解 eBPF 的基本原理、使用方法以及相应的实践案例。</p><p>因为 eBPF 是一个实践性很强的技术，为了更好地掌握它，我希望你在学习这门课之前熟悉 Linux 的基本使用方法，并看得懂简单的 C、Python 等编程语言的基础语法。这样，你就能在后续课程中更好地掌握 eBPF 的实践方法。如果你觉得这些基础知识还没掌握好，也不用太担心，我会在 02 讲中为你详细介绍具体的学习路径、方法技巧，帮你快速查漏补缺，补足基础。</p><p>我会尽量把这门课的内容写得通俗易懂，并帮你划出重点、理出知识脉络，再通过案例分析和套路总结，让你学得更透、用得更熟。总之，我会带你从基础到实践，再结合实际案例，逐层深入 eBPF 相关的系统知识。</p><p><img src="https://static001.geekbang.org/resource/image/43/09/4314b5f14ed6cf38199289ab914b8309.jpg?wh=1920x1141" alt="图片"></p><p>由于 eBPF 还是一个快速发展的技术，也是 Linux 内核社区最活跃和变更最频繁的模块之一，这门课将尝试用全新的方式向你交付。具体来看，<strong>这门课的内容并不会一次性发布完毕，而是按时间分成两大阶段：常规更新阶段 + 动态更新阶段。</strong></p><p>第一个阶段，是像常规的专栏一样，每周更新三篇。这个阶段的内容分成学习准备篇、基础入门篇、实战进阶篇三个模块。</p><p>我会讲解 eBPF 的基本原理、使用方法、案例分析，以及常用工具、学习资料和学习经验总结。这些基本的知识，并不会随着时间的发展过时，它们是你理解 eBPF 机制、把握 eBPF 进化方向的抓手。</p><ul>
<li><strong>学习准备篇</strong>，介绍 eBPF 的发展历程、工作原理以及主要的应用场景。同时，我也会带你梳理 eBPF 的技术脉络和学习路线，并分享我在学习 eBPF 时总结的技巧。</li>
<li><strong>基础入门篇</strong>，介绍 eBPF 的基本原理、编程接口，以及进行详细的原理讲解，这包括：
<ul>
<li>如何搭建 eBPF 的开发环境；</li>
<li>如何用好 BCC 并在它的基础上扩展自己的 eBPF 程序；</li>
<li>如何从零开发一个 eBPF 程序；</li>
<li>如何根据实际需要选择具体的 eBPF 程序类型。</li>
</ul>
</li>
<li><strong>实战进阶篇</strong>，在了解了 eBPF 的基本使用方法后，我会通过一些案例，带你实践 eBPF 的主要应用场景，这包括：
<ul>
<li>如何使用 eBPF 跟踪内核状态；</li>
<li>如何使用 eBPF 跟踪进程状态；</li>
<li>如何使用 eBPF 排查网络问题；</li>
<li>如何使用 eBPF 增强容器安全；</li>
<li>如何开发一个 eBPF 负载均衡程序。</li>
</ul>
</li>
</ul><p>我相信学完第一个阶段的内容后，你就能掌握 eBPF 的运行原理，并且可以编写自己的 eBPF 程序，观测内核的运行状态，并对 eBPF 常见的应用场景了然于胸。</p><p><strong>第二个阶段则是一个动态的过程，我们准备把它叫作“技术雷达篇”。</strong>在第一阶段结束后的 4 年里，每个季度我都会更新一篇文章，带你持续跟踪内核和开源社区的最新进展和应用案例。</p><p><img src="https://static001.geekbang.org/resource/image/a7/5b/a7dfe133bd7c98a98f9427a21b7c705b.jpg?wh=1920x787" alt="图片"></p><p>也就是说，这门课会全方位地解决你在学习、应用 eBPF 时候的一切重点问题。我会先在第一阶段把基础知识交付给你，然后在第二阶段定期向你交付 eBPF 技术的最新进展、发展趋势。 eBPF 技术时时刻刻在发展变化，但是只要你紧跟这颗“雷达”，就能在第一时间获得我为你梳理的最新信息。这样，你就不用再漫无目的地看资讯、查资料、找重点，可以把更多时间花在用好 eBPF 上。</p><p>由于这是一个全新的动态交付过程，我希望你可以和我，还有学习这门课的其他同学保持交流，分享你的学习心得和实践经验，并为课程未来的内容提供建议。</p><p>我邀请你在接下来的时间里，跟我一起走入 eBPF 的奇妙世界。我希望这门课可以帮你掌握 eBPF 的基础原理，并且学会如何把它们真正落地到你的产品之中，解决真实生产环境中的各种问题。同时，我们还会一起见证未来几年中 eBPF 技术的快速更新，共同探索技术发展的更多可能。接下来，就让我们正式开始这段旅程吧！</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/5e/96/a03175bc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>莫名</span>
  </div>
  <div class="_2_QraFYR_0">首先给『技术雷达篇』的更新模式点个赞，对于 BPF 这种变更频繁的技术尤其适用。<br><br>然后抛出两个问题：<br>1、『内核中运行了一个虚拟机』，这个说法觉得有一些误导性。虚拟机通常会有一个中间层，BPF 程序经过 JIT 编译后可以直接在 CPU 上执行，称为 BPF 执行引擎可能更恰当一些。<br>2、BPF 虽然不要求具备内核开发经验，但也有相对较高的门槛。探针虽好用，但仍需对症下针。BPF 技术学习起来挺简单，想用好却有一定难度的，需要对操作系统原理、内核主要模块的源码有一定程度的了解。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. 虚拟机这个词实际上是沿用了开源社区的称呼，的确是执行引擎更精确一些。<br>2. 是的，越熟悉内核就越容易把BPF用的更好。我们专栏的目标是让从没做过内核开发的同学也学会eBPF的用法，进而借助eBPF了解内核，再去根据实际需要深入内核。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-17 21:36:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/50/4d/db91e218.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>sam700000</span>
  </div>
  <div class="_2_QraFYR_0">支持支持，之前的linux课程真的受益匪浅</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 欢迎加入新一季专栏！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-17 18:44:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/8c/2f/68045a83.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>麟</span>
  </div>
  <div class="_2_QraFYR_0">我是上一个专栏的读者。<br>而本专栏直接命中本人需求。<br>已经购买支持。ヾ(✿ﾟ▽ﾟ)ノ</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 欢迎进入新一季专栏。这个专栏可以说是Linux性能优化实战的进阶版，在性能优化的很多案例中其实也使用了ebpf。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-17 18:29:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/9f/a1/d75219ee.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>po</span>
  </div>
  <div class="_2_QraFYR_0">老师才工作十年？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 哈哈，不知不觉已经十年了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-20 01:10:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/47/a8/ec330e70.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Liu</span>
  </div>
  <div class="_2_QraFYR_0">真赞，跟着倪大，学好ebpf</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 加油👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-18 08:10:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/88/58/3e19586a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>晓双</span>
  </div>
  <div class="_2_QraFYR_0">有java相关的实践嘛？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 有的，后面的课程会介绍各种不同编程语言应用程序的跟踪方法。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-27 14:01:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/3f/0d/1e8dbb2c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>怀揣梦想的学渣</span>
  </div>
  <div class="_2_QraFYR_0">我是Linux性能优化的买家，新用户白嫖课程时，我无意发现了作者的Linux性能优化课，看完第一遍，我就果断开了全费会员，再看了一遍作者的课，并且推荐给了同行， 后来还看到华为内部有买这门课给内部员工看，我对ebpf这课程同样充满期待，希望可以收获不亚于Linux性能优化课的知识。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-16 23:01:11</div>
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
  <div class="_2_QraFYR_0">怎么看待 ebpf 应用到 k8s，istio 等？ebpf 能够上生产环境？有很大的性能提升？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-22 15:02:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/24/99/89/d3f9cf62.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Br</span>
  </div>
  <div class="_2_QraFYR_0">作者你好，我最近有个需求，需要自动化梳理系统的api资产，获取到api的主机，地址，请求参数和响应数据，我想用ebpf开发一个探针实现这个需求，但是对ebpf不了解，ebpf可以实现这个需求吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 理论上是可以的，但实现起来要可能并不容易，特别是你有多种不同的编程语言程序和多种不同的接口协议时。如果只是API的话，看起来通过API网关之类的架构更容易管理。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-26 19:16:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/cd/7b/7e3e97dc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>燕鑫</span>
  </div>
  <div class="_2_QraFYR_0">原本是摸不着头脑的自己摸索，这下好了！感恩！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 加油！也欢迎随时在评论区跟我分享你的思考和学习经验。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-17 18:56:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/22/9a/52/93416b65.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>不明真相的群众</span>
  </div>
  <div class="_2_QraFYR_0">朋友推荐过来的，希望跟着博主，day day up~</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我们一起加油！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-24 15:00:16</div>
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
  <div class="_2_QraFYR_0">我又订阅啦，老师</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢支持专栏，我们一起加油！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-20 19:14:17</div>
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
  <div class="_2_QraFYR_0">上一篇LINUX性能优化专栏读后感是相见恨晚，学一专栏抵我工作三年的知识库积累，本片更是如此，ebpf非常能解决我作为软硬结合优化工程师的性能痛点～ 加油！一起学习！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 加油！欢迎在留言区分享学习和实践经验。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-18 21:55:27</div>
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
  <div class="_2_QraFYR_0">哈哈哈～ 等这门课等了好久！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 久等了，我们一起加油！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-18 10:39:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/f6/24/547439f1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ghostwritten</span>
  </div>
  <div class="_2_QraFYR_0">正看看你的2023年4月8日开源云原生开发日eBPF在云原生中的应用。😄</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-08 14:30:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/23/3b/1d/cbfbd93e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小印_zoe</span>
  </div>
  <div class="_2_QraFYR_0">初学eBPF，请问一下这个有eBPF的交流群么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我们有一个交流群，可以关注“漫谈云原生”公众号，回复“联系”（不带引号）加入。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-06 13:37:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83erLKlSIdiadmBR0awVgQcTGbsnd1dp1uaDcdfgyFNmREXNEANjMVSDKV3yYD2AKQEicibvKY35RVpmmg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>novoer</span>
  </div>
  <div class="_2_QraFYR_0">“eBPF 需要运行在内核中，并且还涉及到一些内核的编程开发时，心里有点打退堂鼓”  说到心坎里去了，另一个“技术雷达篇”的更新模式 真的很用心开这个专栏</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢支持👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-11 17:34:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/4VCgcBbU51SiasW8tpjYwdldVmAa4c3I44wS1ISJA0Q4A1cU50q3jcOGa379fI2LhDibE6ECWBMuYV13vuEnIjwA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_da2490</span>
  </div>
  <div class="_2_QraFYR_0">ebpf入门佳作</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-21 21:43:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/9e/27/abb7bfe3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>贝壳</span>
  </div>
  <div class="_2_QraFYR_0">由于手头有未学习完的课程，一直没有发现这门课程，意外之喜，今天入坑</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-14 07:55:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/23/73/de/2d9bc244.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>彬</span>
  </div>
  <div class="_2_QraFYR_0">老师，如果开发一个ebpf过滤网络包的应用适合用那个语言来写用户态程序</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-12 18:57:06</div>
  </div>
</div>
</div>
</li>
</ul>