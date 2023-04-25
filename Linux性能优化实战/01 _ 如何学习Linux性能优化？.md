<audio title="01 _ 如何学习Linux性能优化？" src="https://static001.geekbang.org/resource/audio/c9/79/c9c7941e6ef5d2400dcd5880642f3479.mp3" controls="controls"></audio> 
<p>你好，我是倪朋飞。</p><p>你是否也曾跟我一样，看了很多书、学了很多Linux性能工具，但在面对Linux性能问题时，还是束手无策？实际上，性能分析和优化始终是大多数软件工程师的一个痛点。但是，面对难题，我们真的就无解了吗？</p><p>固然，性能问题的复杂性增加了学习难度，但这并不能成为我们进阶路上的“拦路虎”。在我看来，大多数人对性能问题“投降”，原因可能只有两个。</p><p>一个是你没找到有效的方法学原理，一听到“系统”、“底层”这些词就发怵，觉得东西太难，自己一定学不会，自然也就无法深入学下去，从而不能建立起性能的全局观。</p><p>再一个就是，你看到性能问题的根源太复杂，既不懂怎么去分析，也不能抽丝剥茧找到瓶颈。</p><p>你可能会想，反正程序出了问题，上网查就是了，用别人的方法，囫囵吞枣地多试几次，有可能就解决了。于是，你懒得深究这些方法为啥有效，更不知道为什么，很多方法在别人的环境有效，到你这儿就不行了。</p><p>所以，相同的错误重复在犯，相同的状况也是重复出现。</p><p>其实，性能问题并没有你想像得那么难，<strong>只要你理解了应用程序和系统的少数几个基本原理，再进行大量的实战练习，建立起整体性能的全局观</strong>，大多数性能问题的优化就会水到渠成。</p><p>我见过很多工程师，在分析应用程序所使用的第三方组件的性能时，并不熟悉这些组件所用的编程语言，却依然可以分析出线上问题的根源，并能通过一些方法进行优化，比如修改应用程序对它们的调用逻辑，或者调整组件的配置选项等。</p><!-- [[[read_end]]] --><p>还是那句话，<strong><span class="orange">你不需要了解每个组件的所有实现细节</span></strong>，只要能理解它们最基本的工作原理和协作方式，你也可以做到。</p><h2>性能指标是什么？</h2><p>学习性能优化的第一步，一定是了解“性能指标”这个概念。</p><p>当看到性能指标时，你会首先想到什么呢？我相信“<strong>高并发</strong>”和“<strong>响应快</strong>”一定是最先出现在你脑海里的两个词，而它们也正对应着性能优化的两个核心指标——“<span class="orange">吞吐</span>”和“<span class="orange">延时</span>”。这两个指标是<strong>从应用负载的视角</strong>来考察性能，直接影响了产品终端的用户体验。跟它们对应的，是<strong>从系统资源的视角</strong>出发的指标，比如资源使用率、饱和度等。</p><p><img src="https://static001.geekbang.org/resource/image/92/1d/920601da775da08844d231bc2b4c301d.png?wh=1051*595" alt=""></p><p>我们知道，随着应用负载的增加，系统资源的使用也会升高，甚至达到极限。而<strong>性能问题的本质</strong>，就是系统资源已经达到瓶颈，但请求的处理却还不够快，无法支撑更多的请求。</p><p>性能分析，其实就是<strong>找出应用或系统的瓶颈，并设法去避免或者缓解它们</strong>，从而更高效地利用系统资源处理更多的请求。这包含了一系列的步骤，比如下面这六个步骤。</p><ul>
<li>
<p>选择指标评估应用程序和系统的性能；</p>
</li>
<li>
<p>为应用程序和系统设置性能目标；</p>
</li>
<li>
<p>进行性能基准测试；</p>
</li>
<li>
<p>性能分析定位瓶颈；</p>
</li>
<li>
<p>优化系统和应用程序；</p>
</li>
<li>
<p>性能监控和告警。</p>
</li>
</ul><p>了解了这些性能相关的基本指标和核心步骤后，该怎么学呢？接下来，我来说说要学好Linux 性能优化的几个重要问题。</p><h2>学这个专栏需要什么基础</h2><p>首先你要明白，我们这个专栏的核心是<span class="orange">性能的分析和优化</span>，而不是最基本的Linux操作系统的使用方法。</p><p>因而，我希望你最好用过Ubuntu或其他Linux操作系统，然后要具备一些<strong>编程基础</strong>，比如：</p><ul>
<li>
<p>了解Linux常用命令的使用方法；</p>
</li>
<li>
<p>知道怎么安装和管理软件包；</p>
</li>
<li>
<p>知道怎么通过编程语言开发应用程序等。</p>
</li>
</ul><p>这样，在我讲性能时，你就更容易理解性能背后的原理，特别是在结合专栏里的案例实践后，对性能分析能有更直观的体会。</p><p>这个专栏不会像教科书那样，详细教你操作系统、算法原理、网络协议乃至各种编程语言的全部细节，但一些重要的系统原理还是必不可少的。我还会用实际案例一步步教你，贯穿从应用程序到操作系统的各个组件。</p><h2>学习的重点是什么？</h2><p>想要学习好性能分析和优化，<strong>建立整体系统性能的全局观</strong>是最核心的话题。因而，</p><ul>
<li>
<p>理解最基本的几个系统知识原理；</p>
</li>
<li>
<p>掌握必要的性能工具；</p>
</li>
<li>
<p>通过实际的场景演练，贯穿不同的组件。</p>
</li>
</ul><p>这三点，就是我们学习的重中之重。我会在专栏的每篇文章中，针对不同场景，把这三个方面给你讲清楚，你也一定要花时间和心思来消化它们。</p><p>其实说到性能工具，就不得不提性能领域的大师布伦丹·格雷格（Brendan Gregg）。他不仅是动态追踪工具DTrace的作者，还开发了许许多多的性能工具。我相信你一定见过他所描绘的Linux性能工具图谱：</p><p><img src="https://static001.geekbang.org/resource/image/9e/7a/9ee6c1c5d88b0468af1a3280865a6b7a.png?wh=3000*2100" alt=""></p><p>（图片来自<a href="http://www.brendangregg.com/Perf/linux_perf_tools_full.png">brendangregg.com</a>）</p><p>这个图是Linux性能分析最重要的参考资料之一，它告诉你，在Linux不同子系统出现性能问题后，应该用什么样的工具来观测和分析。</p><p>比如，当遇到I/O性能问题时，可以参考图片最下方的I/O子系统，使用iostat、iotop、blktrace等工具分析磁盘I/O的瓶颈。你可以把这个图保存下来，在需要的时候参考查询。</p><p>另外，我还要特别强调一点，就是<strong>性能工具的选用</strong>。有句话是这么说的，一个正确的选择胜过千百次的努力。虽然夸张了些，但是选用合适的性能工具，确实可以大大简化整个性能优化过程。在什么场景选用什么样的工具、以及怎么学会选择合适工具，都是我想教给你的东西。</p><p>但是切记，<strong><span class="orange">千万不要把性能工具当成学习的全部</span></strong>。工具只是解决问题的手段，关键在于你的用法。只有真正理解了它们背后的原理，并且结合具体场景，融会贯通系统的不同组件，你才能真正掌握它们。</p><p>最后，为了让你对性能有个全面的认识，我画了一张思维导图，里面涵盖了大部分性能分析和优化都会包含的知识，专栏中也基本都会讲到。你可以<span class="orange">保存或者打印</span>下来，每学会一部分就标记出来，记录并把握自己的学习进度。</p><p><img src="https://static001.geekbang.org/resource/image/0f/ba/0faf56cd9521e665f739b03dd04470ba.png?wh=2586*8464" alt=""></p><h2>怎么学更高效？</h2><p>前面我给你讲了Linux性能优化的学习重点，接下来我再跟你分享一下，我的几个学习技巧。掌握这些技巧，可以让你学得更轻松。</p><p><strong>技巧一：虽然系统的原理很重要，但在刚开始一定不要试图抓住所有的实现细节。</strong></p><p>深陷到系统实现的内部，可能会让你丢掉学习的重点，而且繁杂的实现逻辑，很可能会打退你学习的积极性。所以，我个人观点是一定要适度。</p><p>你可以先学会我给你讲的这些系统工作原理，但不要去深究Linux内核是如何做到的，而是要把你的重点放到如何观察和运用这些原理上，比如：</p><ul>
<li>
<p>有哪些指标可以衡量性能？</p>
</li>
<li>
<p>使用什么样的性能工具来观察指标？</p>
</li>
<li>
<p>导致这些指标变化的因素等。</p>
</li>
</ul><p><strong>技巧二：边学边实践，通过大量的案例演习掌握Linux性能的分析和优化。</strong></p><p>只有通过在机器上练习，把我讲的知识和案例自己过一遍，这些东西才能转化成你的。我精心设计这些案例，正是为了让你有更好的学习理解和操作体验。</p><p>所以我强烈推荐你去实际运行、分析这些案例，或者用学到的知识去分析你自己的系统，这样你会有更直观的感受，获得更好的学习效果。</p><p><strong>技巧三：勤思考，多反思，善总结，多问为什么。</strong></p><p>想真正学懂一门知识，最好的方法就是<span class="orange">问问题</span>。当你能提出好的问题时，就说明你已经深入了解了它。</p><p>你可以随时在留言区给我留言，写下自己的疑问、思考和总结，和我还有其他的学习者一起讨论切磋。你也可以写下自己经历过的性能问题，记录你的分析步骤和优化思路，我们一起互动探讨。</p><h2>学习之前，你的准备</h2><p>作为一个包含大量案例实践的课程，我会在每篇文章中，使用一到两台Ubuntu 18.04虚拟机，作为案例运行和分析的环境。如果你只是单纯听音频的讲解，却从不动手实践，学习的效果一定会大打折扣。</p><p>所以，你是不是可以准备好一台Linux机器，用于课程案例的实践呢？<span class="orange">任意的虚拟机或物理机都可以，并不局限于Ubuntu系统</span>。</p><h2>思考</h2><p>今天的内容是我们后续学习的热身准备。从下篇文章开始，我们就要正式进入Linux性能分析和优化了。所以，我想请你来聊一聊，你之前在解决Linux性能问题时，有遇到过什么样的困难或者疑惑吗？或者是之前自己学习Linux性能优化时，有哪些问题吗？参考我今天所讲的内容，你又打算怎么来学这个专栏？</p><p>欢迎在留言区和我分享。</p><p><img src="https://static001.geekbang.org/resource/image/56/52/565d66d658ad23b2f4997551db153852.jpg?wh=1110*549" alt=""></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/60/05/3797d774.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>forever</span>
  </div>
  <div class="_2_QraFYR_0">我遇到性能瓶颈的排查思路<br><br>有监控的情况下，首先去看看监控大盘，看看有没有异常报警，如果初期还没有监控的情况我会按照下面步骤去看看系统层面有没有异常<br>1、我首先会去看看系统的平均负载，使用top或者htop命令查看,平均负载体现的是系统的一个整体情况，他应该是cpu、内存、磁盘性能的一个综合，一般是平均负载的值大于机器cpu的核数，这时候说明机器资源已经紧张了<br>2、平均负载高了以后，接下来就要看看具体是什么资源导致，我首先会在top中看cpu每个核的使用情况，如果占比很高，那瓶颈应该是cpu,接下来就要看看是什么进程导致的<br>3、如果cpu没有问题，那接下来我会去看内存，首先是用free去查看内存的是用情况，但不直接看他剩余了多少，还要结合看看cache和buffer，然后再看看具体是什么进程占用了过高的内存，我也是是用top去排序<br>4、内存没有问题的话就要去看磁盘了，磁盘我用iostat去查看，我遇到的磁盘问题比较少<br>5、还有就是带宽问题，一般会用iftop去查看流量情况，看看流量是否超过的机器给定的带宽<br>6、涉及到具体应用的话，就要根据具体应用的设定参数来查看，比如连接数是否查过设定值等<br>7、如果系统层各个指标查下来都没有发现异常，那么就要考虑外部系统了，比如数据库、缓存、存储等<br><br>基本上就上面这些步骤，有些不完整，希望跟着老师学习一些更系统的排查思路！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 总结的很好👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-21 09:45:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/42/f7/9dce9a7b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>十三</span>
  </div>
  <div class="_2_QraFYR_0">D2打卡<br><br>1. 笔记<br>技巧一：虽然系统的原理很重要，但在刚开始一定不要试图抓住所有的实现细节。”<br><br>深陷到系统实现的内部，可能会让你丢掉学习的重点，而且繁杂的实现逻辑，很可能会打退你学习的积极性。所以，我个人观点是一定要适度。<br><br>2. 心得<br>作为一个完美主义者，一学起原理类的东西，真的不要太容易跑偏😂经常是看着某个重要原理，就想着找找看相关内容，然后就各种跳转搜索，以前最开始学数据结构的定义，都能跑到编译原理上，最后开始计算二进制了。<br><br>有时候大半天了，一个原理都没看完，就各种死抠和联想。这么做确实印象深刻，但是真的很低效，心累。<br><br>老师这里说的适度，真的很重要，而且这个度，确实应该是过来人才知道啊。<br><br>我一向喜欢系统化的学习，能有个“升级简化版”的系统知识图谱，不要太开心。可惜不能上传图片，不然可以把每次标记和补充也都打个卡了。<br><br>开始学了，加油！冲着我的四个月后涨工资的目标去了！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 总结的好👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-21 00:16:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/14/00/94ee0daf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Dragon</span>
  </div>
  <div class="_2_QraFYR_0">能否在翻页的左侧或右侧加一个暂停键, 你想如果边听边看文字时候暂停还要返回头顶,多麻烦</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-23 22:28:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/36/d2/c7357723.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>发条橙子 。</span>
  </div>
  <div class="_2_QraFYR_0">以前看服务器的资源使用只会简单的使用 top命令 看cpu使用的百分比。，但是却不清楚到底多高才算高危 ，面对持续增长我该怎么预防或处理 ， load指标具体的含义 和 cpu有什么关联 ..这些都没有一个整体的概念 </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 下一篇就是平均负载😊</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-21 00:18:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/dd/b9/ce84d577.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Charles.Gast</span>
  </div>
  <div class="_2_QraFYR_0">晚上下班，坐着地铁，看着教程。完美的过站。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-04 21:28:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/1d/1f/6bc10297.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Allen</span>
  </div>
  <div class="_2_QraFYR_0">『day1』<br>这周工作中遇到了一个紧急的问题（用的是arm系列的单板），单板的空间几乎快满了。 使用了top和free命令查看，单板内存的使用情况，仅仅凭借这两个命令，是不可能分析出来原因的。<br><br>查看&#47;proc&#47;&lt;pid&gt;&#47;下的的meminfo、status等文件可以具体才看到虚拟内存和实际物理内存的使用情况。  之前根本不了解&#47;proc&#47;&lt;pid&gt;里面的文件都是干嘛的。<br><br>希望跟着老师的专栏，可以了解下linux系统的基本知识，以后遇到相关问题时，可以有一些思路。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 内存模块会详细介绍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-21 00:41:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJTqaqppkyn9avZHqCnONVJ2Cwp9fmr7yQqUB8icRMFgCbYEHPAyogHqfYjgKPQBteKxnEKQHJzDmQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_60821e</span>
  </div>
  <div class="_2_QraFYR_0">希望之后课程能深入的用工具分析原因，不要讲工具怎么用，怎么用工具上网free的实例很多，找出瓶颈的原因是我来听课目的，wish nice</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-21 13:23:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4c/0c/32ec9763.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Luna</span>
  </div>
  <div class="_2_QraFYR_0">性能指标概念：高并发 =&gt; 吞吐   响应快 =&gt; 延时<br>该概念是从应用负载的角度出发：Application ▹Libraries▹System Call▹Linux Kernel ▹Drive<br><br> 与之对应的是系统资源视角出发 ：Drive▹Linux Kernel ▹System Call ▹Libraries ▹Application <br><br>性能指标的评判有以上二种常用的角度<br><br>接着六步<br><br>1.选择性能指标评估应用和系统的性能<br><br>2.为应用和系统设定性能目标<br><br>3.进行性能基准测试<br><br>4.性能分析定位瓶颈<br><br>5.优化系统和应用程序<br><br>6.性能监控和告警<br><br>六步总结，从正确的角度出发，设定目标（性能优化不是漫无目的的），基准测试（了解现有系统应用的运行时情况），根据情况分析瓶颈，优化它，设置监控和告警（其实可以再扩展比如达到一定的负载，采取降级等操作）</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 总结的很好</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-21 10:18:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/64/9b/d1ab239e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>J.Smile</span>
  </div>
  <div class="_2_QraFYR_0">“你可以先学会我给你讲的这些系统工作原理，但不要去深究 Linux 内核是如何做到的，而是要把你的重点放到如何观察和运用这些原理上”<br>--------------------------<br>感觉说的很对，前几天订阅了刘超老师的《趣谈LinuxOS》，这个专栏罗列了不少Linux内核的代码片段，可以说是基本都看不懂，作为一名7年java尴尬的一批，主要就是总是陷入到Linux内核的代码中，导致一篇专栏反复看了两遍也没懂得更多，所以果断阶段性放弃了，所以聚焦比较重要，我也是拿这个作为自己的座右铭了，希望可以把这个专栏坚持下去！加油！💪！☆！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 加油👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-20 15:40:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/b7/c6/839984bc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>周</span>
  </div>
  <div class="_2_QraFYR_0">性能领域的大师布伦丹·格雷格（Brendan Gregg），个人博客地址，http:&#47;&#47;www.brendangregg.com，国内可以访问。<br>本地虚拟机已安装好<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-22 17:27:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/8c/41/5a66afc8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>上善若水</span>
  </div>
  <div class="_2_QraFYR_0">之前调试网络性能问题，组网比较复杂，测试仪千万级用户拨号，越到后面越是上线速率急速下降，每个点都进行抓包监控丢包率，top也观测进程cpu利用率，就是找不出原因，进程cpu利用率也不高，丢包也不严重，对比抓包，发现上送包处理有很大延迟，后来pstack应用进程发现一处一点，原来应用进程有一处调用函数是阻塞的，光有生产者没有消费者，导致大用户拨号后阻塞超时，而且那个api还是第三方的没有源码，找到维护的组才确认问题，前后加班熬了两个星期才确认问题，所以痛下心来学习下性能定位课程</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-21 13:03:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/5c/b1/9e70e7aa.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>石壹笑</span>
  </div>
  <div class="_2_QraFYR_0">切记，千万不要把性能工具当成学习的全部。<br>应消除干扰，专注于愿问题本身。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-21 00:35:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/57/ed/d50de13c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>mj4ever</span>
  </div>
  <div class="_2_QraFYR_0">之所以选择学习这个专栏，就是希望能解决实际工作中的一些技术问题，当公司研发的产品在现场运行，出现性能问题的时，不会束手无策，毫无思路，误打误撞的去解决问题。<br>因此，希望通过3个月的学习，自己可以掌握以下几个方面：<br>1、建立整体性能的全局观<br>2、理解最基本的几个系统知识原理<br>3、掌握Linux 性能工具图谱的熟练应用<br>感谢大家，也感谢能不断坚持的自己。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-21 09:15:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/7d/d8/d7c77764.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>HunterYuan</span>
  </div>
  <div class="_2_QraFYR_0">我做过性能优化，主要是网络方面的，比如根据网络设备的吞吐、时延、新建和并发指标进行优化。涉及到的优化网络层级有底层，中间层和应用层。底层主要是网络驱动中断、轮询、虚拟化（dpdk和netmap）,同时根据性能指标，设置网卡驱动的中断频率，将网卡多队列绑定到不同CPU，是整个网络同种数据流在整个内核协议栈上到同一个CPU，减少CPU切换和CPU缓冲频繁读写，例如NAT前后的数据去往同一个CPU。中间层，主要是将应用socket和内核协议栈采用零拷贝方式，减少内存拷贝消耗的性能，例如Netlink mmap相关内容。应用层，主要是开启多线程，将不同线程根据策略绑定到不同的CPU，是否开启独占CPU等。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-28 17:58:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/51/cd/d6fe851f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Gopher</span>
  </div>
  <div class="_2_QraFYR_0">小哥哥声音好听，语音很清晰</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-22 16:33:53</div>
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
  <div class="_2_QraFYR_0">已经在朋友圈转发2次老师的课了。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-21 11:27:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/b9/76/bc5f8468.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>品牌运营|陆晓明</span>
  </div>
  <div class="_2_QraFYR_0">学习这块，是想应用到手机系统上。安卓的性能一直是个诟病，想通过一些工具进行复测，能够找到一些思路进行优化，然后使用一套测试打分，来评估到底是改善还是没有。<br><br>在了解学习linux的常用性能分析工具，想从本课程琢磨出一套方式，来进行系统的优化，找到一条优化安卓系统的有效方法。<br><br>感谢！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-21 00:19:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/1f/67/d8c69191.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>谢艺华</span>
  </div>
  <div class="_2_QraFYR_0">我之前做过CPU性能优化的工作，任务维持了几个星期，从中学到的很多东西，因为我也比较喜欢进行总结和分享，有兴趣的朋友可以看一下我的博客https:&#47;&#47;blog.csdn.net&#47;xieyihua1994&#47;article&#47;details&#47;88854881（肯定没有老师讲的好，但是我一点一点自己总结的，希望能够帮助一些朋友）。也希望在这个专栏中，能够更进一步</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-20 16:45:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLy5AYicpsylYzmLnxdsruoic57vOnMuenIYUB5tlJ9qnH6TbjnSXicEX0HFiap0iasSDlTyqZ4FzNvEJA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>番茄炒鸡蛋</span>
  </div>
  <div class="_2_QraFYR_0">前段时间遇到服务请求过多，导致内存增高，但是过段时间请求下降，但内存还是一直比较高，如果后续再次遇到请求过多，就会导致没有内存，程序崩溃，请问这类问题如何排查？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-21 12:42:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/19/ac/d39011f0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>King_stone</span>
  </div>
  <div class="_2_QraFYR_0">遇到过的问题：Linux上tomcat性能下降，资源消耗较大，最终判断是JAVA程序一个方法的问题。解决和监控的手段不够智能，写过一个监控但是JAVA本身的监控不稳定，消耗也较大，最终撤掉，希望课程能对这方面优化监控有介绍</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-21 09:54:09</div>
  </div>
</div>
</div>
</li>
</ul>