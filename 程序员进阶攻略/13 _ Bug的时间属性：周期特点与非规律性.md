<audio title="13 _ Bug的时间属性：周期特点与非规律性" src="https://static001.geekbang.org/resource/audio/56/05/56c6cad43b9358f1796ecfbe7a220e05.mp3" controls="controls"></audio> 
<p>在上一篇文章中，我说明了“技术性 Bug 可以从很多维度分类，而我则习惯于从 Bug 出现的 ‘时空’ 特征角度来分类”。并且我也已讲解了Bug 的<strong>空间维度</strong>特征：程序对运行环境的依赖、反应及应对。</p>
<p>接下来我再继续分解 Bug 的<strong>时间维度</strong>特征。</p>
<p>Bug 有了时间属性，Bug 的出现就是一个概率性问题了，它体现出如下特征。</p>
<h2>周期特点</h2>
<p>周期特点，是一定频率出现的 Bug 的特征。</p>
<p>这类 Bug 因为会周期性地复现，相对还是容易捕捉和解决。比较典型的呈现此类特征的 Bug 一般是资源泄漏问题。比如，Java 程序员都不陌生的 <code>OutOfMemory</code> 错误，就属于内存泄漏问题，而且一定会周期性地出现。</p>
<p>好多年前，我才刚参加工作不久，就碰到这么一个周期性出现的 Bug。但它的特殊之处在于，出现 Bug 的程序已经稳定运行了十多年了，突然某天开始就崩溃（进程 Crash）了。而程序的原作者，早已不知去向，十多年下来想必也已换了好几代程序员来维护了。</p>
<p>一开始项目组内经验老到的高工认为也许这只是一个意外事件，毕竟这个程序已经稳定运行了十来年了，而且检查了一遍程序编译后的二进制文件，更新时间都还停留在那遥远的十多年前。所以，我们先把程序重启起来让业务恢复，重启后的程序又恢复了平稳运行，但只是安稳了这么一天，第二天上班没多久，进程又莫名地崩溃了，我们再次重启，但没多久后就又崩溃了。这下没人再怀疑这是意外了，肯定有 Bug。</p><!-- [[[read_end]]] -->
<p>当时想想能找出一个隐藏了这么多年的 Bug，还挺让人兴奋的，就好像发现了埋藏在地下久远的宝藏。</p>
<p>寻找这个 Bug 的过程有点像《盗墓笔记》中描述的盗墓过程：项目经理（三叔）带着两个高级工程师（小哥和胖子）连续奋战了好几天，而我则是个新手，主要负责 “看门”，在他们潜入跟踪分析探索的过程中，我就盯着那个随时有可能崩溃的进程，一崩掉就重启。他们“埋伏”在那里，系统崩溃后抓住现场，定位到对应的源代码处，最后终于找到了原因并顺利修复。</p>
<p>依稀记得，最后定位到的原因与网络连接数有关，也是属于资源泄漏的一种，只是因为过去十来年交易量一直不大且稳定，所以没有显现出来。但在我参加工作那年（2006年），中国股市悄然引来一场有史以来最大的牛市，这个处理银行和证券公司之间资金进出的程序的“工作量”突然出现了爆发性增长，从而引发了该 Bug。</p>
<p>我可以理解上世纪九十年代初那个编写该服务进程的程序员，他可能也难以预料到当初写的用者寥寥的程序，最终在十多年后的一天会服务于成百上千万的用户。</p>
<p>周期性的 Bug，虽然乍一看很难解决的样子，但它总会重复出现，就像可以重新倒带的 “案发现场”，找到真凶也就简单了。案例中这个 Bug 隐藏的时间很长，但它所暴露出的周期特点很明显，解决起来也就没那么困难。</p>
<p>其实主要麻烦的是那种这次出现了，但不知道下次会在什么时候出现的 Bug。</p>
<h2>非规律性</h2>
<p>没有规律性的 Bug，才是让人抓狂的。</p>
<p>曾经我接手过一个系统，是一个典型的生产者、消费者模型系统。系统接过来就发现一个比较明显的性能瓶颈问题，生产者的数据源来自数据库，生产者按规则提取数据，经过系统产生一系列的转换渲染后发送到多个外部系统。这里的瓶颈就在数据库上，生产能力不足，从而导致消费者饥饿。</p>
<p>问题比较明显，我们先优化 SQL，但效果不佳，遂改造设计实现，在数据库和系统之间增加一个内存缓冲区从而缓解了数据库的负载压力。缓冲区的效果，类似大河之上的堤坝，旱时积水，涝时泄洪。引入缓冲区后，生产者的生产能力得到了有效保障，生产能力高效且稳定。</p>
<p>本以为至此解决了该系统的瓶颈问题，但在生产环境运行了一段时间后，系统表现为速度时快时慢，这时真正的 Bug 才显形了。</p>
<p>这个系统有个特点，就是 I/O 密集型。消费者要与多达 30 个外部系统并发通信，所以猜测极有可能导致系统性能不稳定的 Bug 就在此，于是我把目光锁定在了消费者与外部系统的 I/O 通信上。既然锁定了怀疑区域，接下来就该用证据来证明，并给出合理的解释原因了。一开始假设在某些情况下触碰到了阈值极限，当达到临界点时程序性能则急剧下降，不过这还停留在怀疑假设阶段，接下来必须量化验证这个推测。</p>
<p>那时的生产环境不太方便直接验证测试，我便在测试环境模拟。用一台主机模拟外部系统，一台主机模拟消费者。模拟主机上的线程池配置等参数完全保持和生产环境一致，以模仿一致的并发数。通过不断改变通信数据包的大小，发现在数据包接近 100k 大小时，两台主机之间直连的千兆网络 I/O 达到满负载。</p>
<p>于是，再回头去观察生产环境的运行状况，当一出现性能突然急剧下降的情况时，立刻分析了生产者的数据来源。其中果然有不少大报文数据，有些甚至高达 200k，至此基本确定了与外部系统的 I/O 通信瓶颈。解决办法是增加了数据压缩功能，以牺牲 CPU 换取 I/O。</p>
<p>增加了压缩功能重新上线后，问题却依然存在，系统性能仍然时不时地急剧降低，而且这个时不时很没有时间规律，但关联上了一个 “嫌疑犯”：它的出现和大报文数据有关，这样复现起来就容易多了。I/O 瓶颈的怀疑被证伪后，只好对程序执行路径增加了大量跟踪调试诊断代码，包含了每个步骤的时间度量。</p>
<p>在完整的程序执行路径中，每个步骤的代码块的执行时间独立求和结果仅有几十毫秒，最高也就在一百毫秒左右，但多线程执行该路径的汇总平均时间达到了 4.5 秒，这比我预期值整整高了两个量级。通过这两个时间度量的巨大差异，我意识到线程执行该代码路径的时间其实并不长，但花在等待 CPU 调度的时间似乎很长。</p>
<p>那么是 CPU 达到了瓶颈么？通过观察服务器的 CPU 消耗，平均负载却不高。只好再次分析代码实现机制，终于在数据转换渲染子程序中找到了一段可疑的代码实现。为了验证疑点，再次做了一下实验测试：用 150k 的线上数据报文作为该程序输入，单线程运行了下，发现耗时居然接近 50 毫秒，我意识到这可能是整个代码路径中最耗时的一个代码片段。</p>
<p>由于这个子程序来自上上代程序员的遗留代码，包含一些稀奇古怪且复杂的渲染逻辑判断和业务规则，很久没人动过了。仔细分析了其中实现，基本就是大量的文本匹配和替换，还包含一些加密、Hash 操作，这明显是一个 CPU 密集型的函数啊。那么在多线程环境下，运行这个函数大概平均每个线程需要多少时间呢？</p>
<p>先从理论上来分析下，我们的服务器是 4 核，设置了 64 个线程，那么理想情况下同一时间可以运行 4 个线程，而每个线程执行该函数约为 50 毫秒。这里我们假设 CPU 50 毫秒才进行线程上下文切换，那么这个调度模型就被简化了。第一组 4 个线程会立刻执行，第二组 4 个线程会等待 50 毫秒，第三组会等待 100 毫秒，依此类推，第 16 组线程执行时会等待 750 毫秒。平均下来，每组线程执行前的平均等待时间应该是在 300 到 350 毫秒之间。这只是一个理论值，实际运行测试结果，平均每个线程花费了 2.6 秒左右。</p>
<p>实际值比理论值慢一个量级，这是为什么呢？因为上面理论的调度模型简化了 CPU 的调度机制，在线程执行过程的 50 毫秒中，CPU 将发生非常多次的线程上下文切换。50 毫秒对于 CPU 的时间分片来说，实在是太长了，因为线程上下文的多次切换和 CPU 争夺带来了额外的开销，导致在生产环境上，实际的监测值达到了 4.5 秒，因为整个代码路径中除了这个非常耗时的子程序函数，还有额外的线程同步、通知和 I/O 等操作。</p>
<p>分析清楚后，通过简单优化该子程序的渲染算法，从近 50 毫秒降低到 3、4 毫秒后，整个代码路径的线程平均执行时间下降到 100 毫秒左右。收益是明显的，该子程序函数性能得到了 10 倍的提高，而整体执行时间从 4.5 秒降低为 100 毫秒，性能提高了 45 倍。</p>
<p>至此，这个非规律性的 Bug 得到了解决。</p>
<p>虽然案例中最终解决了 Bug，但用的方法却非正道，更多依靠的是一些经验性的怀疑与猜测，再去反过来求证。这样的方法局限性非常明显，完全依赖程序员的经验，然后就是运气了。如今再来反思，一方面由于是刚接手的项目，所以我对整体代码库掌握还不够熟悉；另一方面也说明当时对程序性能的分析工具了解有限。</p>
<p>而更好的办法就应该是采用工具，直接引入代码 Profiler 等性能剖析工具，就可以准确地找到有性能问题的代码段，从而避免了看似有理却无效的猜测。</p>
<p>面对非规律性的 Bug，最困难的是不知道它的出现时机，但一旦找到它重现的条件，解决起来也没那么困难了。</p>
<h2>神出鬼没</h2>
<p>能称得上神出鬼没的 Bug 只有一种：<strong>海森堡 Bug（Heisenbug）</strong>。</p>
<p>这个 Bug 的名字来自量子物理学的 “海森堡不确定性原理”，其认为观测者观测粒子的行为会最终影响观测结果。所以，我们借用这个效应来指代那些无法进行观测的 Bug，也就是在生产环境下不经意出现，费尽心力却无法重现的 Bug。</p>
<p>海森堡 Bug 的出现场景通常都是和分布式的并发编程有关。我曾经在写一个网络服务端程序时就碰到过一次海森堡 Bug。这个程序在稳定性负载测试时，连续跑了十多个小时才出现了一次异常，然后在之后的数天内就再也不出现了。</p>
<p>第一次出现时捕捉到的现场信息太少，然后增加了更多诊断日志后，怎么测都不出现了。最后是怎么定位到的？还好那个程序的代码量不大，就天天反复盯着那些代码，好几天过去还真就灵光一现发现了一个逻辑漏洞，而且从逻辑推导，这个漏洞如果出现的话，其场景和当时测试发现的情况是吻合的。</p>
<p>究其根源，该 Bug 复现的场景与网络协议包的线程执行时序有关。所以，一方面比较难复现，另一方面通过常用的调试和诊断手段，诸如插入日志语句或是挂接调试器，往往会修改程序代码，或是更改变量的内存地址，或是改变其执行时序。这都影响了程序的行为，如果正好影响到了 Bug，就可能诞生了一个海森堡 Bug。</p>
<p>关于海森堡 Bug，一方面很少有机会碰到，另一方面随着你编程经验的增加，掌握了很多编码的优化实践方法，也会大大降低撞上海森堡 Bug 的几率。</p>
<p>综上所述，每一个 Bug 都是具体的，每一个具体的 Bug 都有具体的解法。但所有 Bug 的解决之道只有两类：事后和事前。</p>
<p>事后，就是指 Bug 出现后容易捕捉现场并定位解决的，比如第一类周期特点的 Bug。但对于没有明显重现规律，甚至神出鬼没的海森堡 Bug，靠抓现场重现的事后方法就比较困难了。针对这类 Bug，更通用和有效的方法就是在事前预防与埋伏。</p>
<p>之前在讲编程时说过一类代码：运维代码，它们提供的一种能力就像人体血液中的白细胞，可以帮助发现、诊断、甚至抵御 Bug 的 “入侵”。</p>
<p>而为了得到一个更健康、更健壮的程序，运维类代码需要写到何种程度，这又是编程的 “智慧” 领域了，充满了权衡选择。</p>
<p>程序员不断地和 Bug 对抗，正如医生不断和病菌对抗。不过Bug 的存在意味着这是一段活着的、有价值的代码，而死掉的代码也就无所谓 Bug 了。</p>
<p>在你的程序员职业生涯中，有碰到过哪些有意思的 Bug呢？欢迎你给我留言分享讨论。</p>
<p></p>

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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/66/2a/3bac3cec.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>sunny</span>
  </div>
  <div class="_2_QraFYR_0">看得我惊心动魄，以前老是害怕bug出现，现在有点小期待；看看热闹，长长见识，毕竟还在初级，</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 哈哈，心情平复没</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-31 08:21:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/67/f4/9a1feb59.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>钱</span>
  </div>
  <div class="_2_QraFYR_0">目前在优化的一些缓存刷新的定时任务，就属于过几年可能会出Bug的代码（因为这代码就是几年前写的，现在出问题了）原因如下<br>1:缓存刷新的方式是先删后插<br><br>2:我厂的统一规定不允许数据物理删除<br><br>3:经久年月，无效数据越来越多，原来缓存刷新没问题，后来就有了空窗期，在空窗期内访问缓存就会出现问题了<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-01 21:52:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/48/9e/99cb0a7a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>心在飞</span>
  </div>
  <div class="_2_QraFYR_0">我现在就遇到个海森堡bug, 客户现场出现过一次，在自己的服务器环境里一切正常，只能通过code review的方式做一些防御性编程，结果发现算法是老美算法专家92年写的！乱飞的point、各种业务处理算法，瞬间我就不想看了！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 太古老的代码，理解起来没了上下文，全靠想象力了，多半要重写了吧</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-31 08:53:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/38/57/a9f9705a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>无聊夫斯基</span>
  </div>
  <div class="_2_QraFYR_0">需要这么多的逻辑判断的50ms的程序你是如何优化成3ms的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 因为代码写得实在太糟糕了，可优化空间太大😄</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-31 08:40:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/56/ec/1c38b82c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>香槟</span>
  </div>
  <div class="_2_QraFYR_0">之前有遇到一个bug，关于redis序列化和反序列化的。线上有6台服务器，升级了其中一台服务器，内容是增加了调用链监控的程序。升级完先上线看效果。由于机子的aspnet版本需更新，同时更新了内部redis封装的库。前半天运行正常，然后出现部分列表数据不一致的情形。赶紧写了段程序输出值看下。发现值均变为0。看代码是反序列化出值部分。看正常的列表数据，反序列化有值。看库里，发现两者的区别是一种序列化进去有引号，另一种序列化进去没引号。没引号的能在其他5台机子解析，而有引号的只能在升级库中解析。所以定位到了原因。为了顺利上线监控程序，又能平滑升级，选择了调整封装库的序列策略，改为序列成无引号的方式。这才解决不一致的问题。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 升级这种基础库，还是要多回归测试，兼容性问题不难，但处理起来很费时间和精力</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-23 12:03:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/20/39/3168c4ca.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>二木🐶</span>
  </div>
  <div class="_2_QraFYR_0">第一类bug的查找案例实在是不敢恭维，重要的正式环境这样去不断重启应用，加日志等方式简直就是不可能的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 以前写代码都是直接在正式环境编译运行，相当野😂</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-06 17:06:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/27/fc/b8d83d56.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>liangjf</span>
  </div>
  <div class="_2_QraFYR_0">少壮不遇bug，老大徒伤悲</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 哈哈哈😄</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-07 21:30:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/a4/5a/e708e423.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>third</span>
  </div>
  <div class="_2_QraFYR_0">迟到了。总算解决好学校的事情了。<br><br>心得如下<br><br>1，bug的时间属性：周期特点和非规律性<br><br><br>2，周期性出现，比如OutOfMemory，内存泄露。<br><br><br>3，非规律性，解决麻烦，采用工具，直接引入代码 Profiler 等性能剖析工具，就可以准确地找到有性能问题的代码段<br><br><br>4，神出鬼没，海森堡 Bug（Heisenbug）<br><br><br>5，bug的解决之道有两种，事前的，事后的。<br><br><br>6，事后，Bug 出现后，捕捉现场并定位解决的<br><br><br>7，事前进行预防和埋伏</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不迟😊</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-01 17:21:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/64/04/18875529.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>艾尔欧唯伊</span>
  </div>
  <div class="_2_QraFYR_0">最近刚入职组里遇到的问题。。四个应用加一层设备，然后还有硬件资源紧张，合在一起出现的bug表象，基本就是各种展示数据不对。。。<br>有些提了缺陷，但是问题环境都没了。。复现都很难。。。真不容易。。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，现实多是带着一身 Bug 勇敢的上线了😊</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-31 12:43:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/22/46/df595e4a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>CatTalk</span>
  </div>
  <div class="_2_QraFYR_0">生动形象的讲解</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 😊</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-31 00:20:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/7a/d2/4ba67c0c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Sch0ng</span>
  </div>
  <div class="_2_QraFYR_0">1. 应对时间变迁带来的变化，定时检查<br>2. debug很考验基本功<br>3. 遇到灵异事件，掂量一下自己能不能兜住，兜不住的话及早上报，免得挨板子</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-25 20:33:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/65/26/2e9bc97f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>今之人兮</span>
  </div>
  <div class="_2_QraFYR_0">周期性bug 非规律性bug和海森堡bug，周期性的bug因为复现很容易会比较好解决。非规律性的虽然没有特殊规律。但是bug一直存在总会找到如何解决。海森堡bug难以浮现没有规律。可能随着代码量的增加以及代码深度理解可以有效避免这类bug</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-04 18:52:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/6e/93/6fef7aaa.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>石头</span>
  </div>
  <div class="_2_QraFYR_0">时间类Bug种类：周期、非规律、海森堡。<br>解决：事后与事前。事后：根据逻辑、性能工具等进行分析与定位案发现场[预防与埋伏]，然后解决之；事前：运维代码证，帮助发现、诊断、甚至抵御bug。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-05 21:23:28</div>
  </div>
</div>
</div>
</li>
</ul>