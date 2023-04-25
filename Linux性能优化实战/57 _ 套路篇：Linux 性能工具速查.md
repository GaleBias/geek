<audio title="57 _ 套路篇：Linux 性能工具速查" src="https://static001.geekbang.org/resource/audio/5e/c2/5ed68a32cd1313aaba8506babfc9cbc2.mp3" controls="controls"></audio> 
<p>你好，我是倪朋飞。</p><p>上一节，我带你一起梳理了常见的性能优化思路，先简单回顾一下。</p><p>我们可以从系统和应用程序两个角度，来进行性能优化。</p><ul>
<li>
<p>从系统的角度来说，主要是对 CPU、内存、网络、磁盘 I/O 以及内核软件资源等进行优化。</p>
</li>
<li>
<p>而从应用程序的角度来说，主要是简化代码、降低 CPU 使用、减少网络请求和磁盘 I/O，并借助缓存、异步处理、多进程和多线程等，提高应用程序的吞吐能力。</p>
</li>
</ul><p>性能优化最好逐步完善，动态进行。不要追求一步到位，而要首先保证能满足当前的性能要求。性能优化通常意味着复杂度的提升，也意味着可维护性的降低。</p><p>如果你发现单机的性能调优带来过高复杂度，一定不要沉迷于单机的极限性能，而要从软件架构的角度，以水平扩展的方法来提升性能。</p><p>工欲善其事，必先利其器。我们知道，在性能分析和优化时，借助合适的性能工具，可以让整个过程事半功倍。你还记得有哪些常用的性能工具吗？今天，我就带你一起梳理一下常用的性能工具，以便你在需要时，可以迅速找到自己想要的。</p><h2>性能工具速查</h2><p>在梳理性能工具之前，首先给你提一个问题，那就是，在什么情况下，我们才需要去查找、挑选性能工具呢？你可以先自己想一下，再继续下面的内容。</p><p>其实在我看来，只有当你想了解某个性能指标，却不知道该怎么办的时候，才会想到，“要是有一个性能工具速查表就好了”这个问题。如果已知一个性能工具可用，我们更多会去查看这个工具的手册，找出它的功能、用法以及注意事项。</p><!-- [[[read_end]]] --><p>关于工具手册的查看，man 应该是我们最熟悉的方法，我在专栏中多次介绍过。实际上，除了 man 之外，还有另外一个查询命令手册的方法，也就是 info。</p><p>info 可以理解为 man 的详细版本，提供了诸如节点跳转等更强大的功能。相对来说，man 的输出比较简洁，而 info 的输出更详细。所以，我们通常使用 man 来查询工具的使用方法，只有在man 的输出不太好理解时，才会再去参考 info 文档。</p><p>当然，我说过了，要查询手册，前提一定是已知哪个工具可用。如果你还不知道要用哪个工具，就要根据想了解的指标，去查找有哪些工具可用。这其中：</p><ul>
<li>
<p>有些工具不需要额外安装，就可以直接使用，比如内核的 /proc 文件系统；</p>
</li>
<li>
<p>而有些工具，则需要安装额外的软件包，比如 sar、pidstat、iostat 等。</p>
</li>
</ul><p><strong>所以，在选择性能工具时，除了要考虑性能指标这个目的外，还要结合待分析的环境来综合考虑</strong>。比如，实际环境是否允许安装软件包，是否需要新的内核版本等。</p><p>明白了工具选择的基本原则后，我们来看 Linux 的性能工具。首先还是要推荐下面这张图，也就是Brendan Gregg 整理的性能工具谱图。我在专栏中多次提到过，你肯定也已经参考过。<br>
<img src="https://static001.geekbang.org/resource/image/b0/01/b07ca95ef8a3d2c89b0996a042d33901.png?wh=3000*2100" alt=""><br>
（图片来自 <a href="http://www.brendangregg.com/linuxperf.html">brendangregg.com</a>）</p><p>这张图从 Linux 内核的各个子系统出发，汇总了对各个子系统进行性能分析时，你可以选择的工具。不过，虽然这个图是性能分析最好的参考资料之一，它其实还不够具体。</p><p>比如，当你需要查看某个性能指标时，这张图里对应的子系统部分，可能有多个性能工具可供选择。但实际上，并非所有这些工具都适用，具体要用哪个，还需要你去查找每个工具的手册，对比分析做出选择。</p><p>那么，有没有更好的方法来理解这些工具呢？<strong>我的建议，还是从性能指标出发，根据性能指标的不同，将性能工具划分为不同类型</strong>。比如，最常见的就是可以根据 CPU、内存、磁盘 I/O 以及网络的各类性能指标，将这些工具进行分类。</p><p>接下来，我就从 CPU、内存、磁盘 I/O 以及网络等几个角度，梳理这些常见的 Linux 性能工具，特别是从性能指标的角度出发，理清楚到底有哪些工具，可以用来监测特定的性能指标。这些工具，实际上贯穿在我们专栏各模块的各个案例中。为了方便你查看，我将它们都整理成了表格，并增加了每个工具的使用场景。</p><h2>CPU性能工具</h2><p>首先，从 CPU 的角度来说，主要的性能指标就是 CPU 的使用率、上下文切换以及 CPU Cache 的命中率等。下面这张图就列出了常见的 CPU 性能指标。<br>
<img src="https://static001.geekbang.org/resource/image/9a/69/9a211905538faffb5b3221ee01776a69.png?wh=1241*1212" alt=""><br>
从这些指标出发，再把 CPU 使用率，划分为系统和进程两个维度，我们就可以得到，下面这个 CPU 性能工具速查表。注意，因为每种性能指标都可能对应多种工具，我在每个指标的说明中，都帮你总结了这些工具的特点和注意事项。这些也是你需要特别关注的地方。<br>
<img src="https://static001.geekbang.org/resource/image/28/b0/28cb85011289f83804c51c1fb275dab0.png?wh=1707*2563" alt=""></p><h2>内存性能工具</h2><p>接着我们来看内存方面。从内存的角度来说，主要的性能指标，就是系统内存的分配和使用、进程内存的分配和使用以及 SWAP 的用量。下面这张图列出了常见的内存性能指标。<br>
<img src="https://static001.geekbang.org/resource/image/ee/c0/ee36f73b9213063b3bcdaed2245944c0.png?wh=1581*1760" alt=""><br>
从这些指标出发，我们就可以得到如下表所示的内存性能工具速查表。同 CPU 性能工具一样，这儿我也帮你梳理了，常见工具的特点和注意事项。<br>
<img src="https://static001.geekbang.org/resource/image/79/f8/79ad5caf0a2c105b7e9ce77877d493f8.png?wh=1653*2198" alt=""><br>
注：最后一行pcstat的源码链接为 <a href="https://github.com/tobert/pcstat">https://github.com/tobert/pcstat</a></p><h2>磁盘I/O性能工具</h2><p>接下来，从文件系统和磁盘 I/O 的角度来说，主要性能指标，就是文件系统的使用、缓存和缓冲区的使用，以及磁盘 I/O 的使用率、吞吐量和延迟等。下面这张图列出了常见的 I/O 性能指标。<br>
<img src="https://static001.geekbang.org/resource/image/72/3b/723431a944034b51a9ef13a8a1d4d03b.png?wh=2631*808" alt=""><br>
从这些指标出发，我们就可以得到，下面这个文件系统和磁盘 I/O 性能工具速查表。同 CPU和内存性能工具一样，我也梳理出了这些工具的特点和注意事项。<br>
<img src="https://static001.geekbang.org/resource/image/c2/a3/c232dcb4185f7b7ba95c126889cf6fa3.png?wh=1714*2424" alt=""></p><h2>网络性能工具</h2><p>最后，从网络的角度来说，主要性能指标就是吞吐量、响应时间、连接数、丢包数等。根据 TCP/IP 网络协议栈的原理，我们可以把这些性能指标，进一步细化为每层协议的具体指标。这里我同样用一张图，分别从链路层、网络层、传输层和应用层，列出了各层的主要指标。<br>
<img src="https://static001.geekbang.org/resource/image/37/a4/37d04c213acfa650bd7467e3000356a4.png?wh=1983*1104" alt=""><br>
从这些指标出发，我们就可以得到下面的网络性能工具速查表。同样的，我也帮你梳理了各种工具的特点和注意事项。<br>
<img src="https://static001.geekbang.org/resource/image/5d/5d/5dde213baffd7811ab73c82883b2a75d.png?wh=1709*2462" alt=""></p><h2>基准测试工具</h2><p>除了性能分析外，很多时候，我们还需要对系统性能进行基准测试。比如，</p><ul>
<li>
<p>在文件系统和磁盘 I/O 模块中，我们使用 fio 工具，测试了磁盘 I/O 的性能。</p>
</li>
<li>
<p>在网络模块中，我们使用 iperf、pktgen 等，测试了网络的性能。</p>
</li>
<li>
<p>而在很多基于 Nginx 的案例中，我们则使用 ab、wrk 等，测试 Nginx 应用的性能。</p>
</li>
</ul><p>除了专栏里介绍过的这些工具外，对于 Linux 的各个子系统来说，还有很多其他的基准测试工具可能会用到。下面这张图，是 Brendan Gregg 整理的 Linux 基准测试工具图谱，你可以保存下来，在需要时参考。<br>
<img src="https://static001.geekbang.org/resource/image/f0/e9/f094f489049602e1058e02edc708e6e9.png?wh=1500*1050" alt=""><br>
（图片来自 <a href="http://www.brendangregg.com/linuxperf.html">brendangregg.com</a>）</p><h2>小结</h2><p>今天，我们一起梳理了常见的性能工具，并从 CPU、内存、文件系统和磁盘 I/O、网络以及基准测试等不同的角度，汇总了各类性能指标所对应的性能工具速查表。</p><p>当分析性能问题时，大的来说，主要有这么两个步骤：</p><ul>
<li>
<p>第一步，从性能瓶颈出发，根据系统和应用程序的运行原理，确认待分析的性能指标。</p>
</li>
<li>
<p>第二步，根据这些图表，选出最合适的性能工具，然后了解并使用工具，从而更快观测到需要的性能数据。</p>
</li>
</ul><p>虽然 Linux 的性能指标和性能工具都比较多，但熟悉了各指标含义后，你自然就会发现这些工具同性能指标间的关联。顺着这个思路往下走，掌握这些工具的选用其实并不难。</p><p>当然，正如咱们专栏一直强调的，不要把性能工具当成性能分析和优化的全部。</p><ul>
<li>
<p>一方面，性能分析和优化的核心，是对系统和应用程序运行原理的掌握，而性能工具只是辅助你更快完成这个过程的帮手。</p>
</li>
<li>
<p>另一方面，完善的监控系统，可以提供绝大部分性能分析所需的基准数据。从这些数据中，你很可能就能大致定位出性能瓶颈，也就不用再去手动执行各类工具了。</p>
</li>
</ul><h2>思考</h2><p>最后，我想邀请你一起来聊聊，你都使用过哪些性能工具。你通常是怎么选择性能工具的？又是如何想到要用这些性能工具，来排查和分析性能问题的？你可以结合我的讲述，总结自己的思路。</p><p>欢迎在留言区和我讨论，也欢迎你把这篇文章分享给你的同事、朋友。我们一起在实战中演练，在交流中进步。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/64/9b/d1ab239e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>J.Smile</span>
  </div>
  <div class="_2_QraFYR_0">思考：<br>⭐问题，办法，工具的关系——问题是人一生中唯一的事。解决问题的办法有很多，办法使用到的工具������️也有很多。<br><br>⭐转化——核心技术，是一种重要的工具，行之有效的工具，少了这个核心工具就做不成事情，此时的工具就转化为办法。<br><br>⭐技术，是一种工具，不要误以为是办法。只有当除此技术就无法解决问题的时候，技术工具才是办法本身，比如ai,大数据只是解决问题的办法，tensorflow,spark等是解决问题的工具。<br><br>⭐工具，似乎不可能唯一。当解决问题的办法出现后，工具也趋向多元化。所以，似乎办法比工具重要的多。<br><br>⭐办法，也就是方法，解决问题的思路，步骤。需要理解问题的结构、细节，进而采取措施。兵来将挡，水来土掩。所以，关键就是知道兵将关系，水土关系，理解问题，剖析问题是根本。<br><br>⭐问题，是程序bug？运营问题？管理问题？一个问题的出现可能由多方因素。<br><br>那么作为技术人员，能拿技术解决什么问题？解决问题，才能收货利益。比如朝九晚五写bug，夜以继日搞私活。解决的问题都不是痛点，或者并不是太痛。<br><br>⭐痛点，有多痛，就有多大价值，就可以带来多大收益。要体现自我价值，就得解决痛点。谁解决了痛点，谁就是有价值的人。但痛点本身与技术无关。<br>一句话：抓住痛点(方向)，剖析原理(方法)，用好工具(利剑)。<br>-----拿着’利剑‘用正确的’方法‘挥向正确的’方向‘--------------------</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-21 11:48:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/9b/ba/333b59e5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Linuxer</span>
  </div>
  <div class="_2_QraFYR_0">有一个问题请教，很多时候工具有了，但是对于指标是否在合理范围好像没有明确的标准，都是经验式的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯 这确实是经验式的，没有绝对的指标，不同应用不同场景都不太一样。不过USE法的这些指标都比较直观，可以优先查看</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-08 08:43:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/3a/6e/e39e90ca.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>大坏狐狸</span>
  </div>
  <div class="_2_QraFYR_0">我看到nethogs的时候去尝试理解一下，结果v0.8.0 会有如下错误：creating socket failed while establishing local IP - are you root?<br>即使是root用户去执行。看网上说v0.8.1 解决了这个问题。所以就下载了8.5版本。<br><br>wget -c https:&#47;&#47;github.com&#47;raboof&#47;nethogs&#47;archive&#47;v0.8.5.tar.gz<br>tar xf v0.8.5.tar.gz <br>cd .&#47;nethogs-0.8.5&#47;<br><br>sudo apt-get install libncurses5-dev libpcap-dev<br>make &amp;&amp; sudo make install </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍 谢谢分享</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-15 13:54:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/4e/0f/c43745e7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>hola</span>
  </div>
  <div class="_2_QraFYR_0">可以做一期dtrace的讲解吗？感觉这个工具很厉害，但是挺复杂的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: dtrace在linux上无法使用，其实我也不熟。官方网站上有个很详细的文档，可以去看看</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-08 10:17:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/e0/c5/c324a7de.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jorin@zou</span>
  </div>
  <div class="_2_QraFYR_0">是我学的太快了吗，怎么越学到后面，评论越少了，得放慢脚步</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-09 22:06:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/75/e3/ef489d57.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Roy</span>
  </div>
  <div class="_2_QraFYR_0">Linux什么时候有跑分软件?象安兔兔</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个不了解😓 不过如果针对具体性能指标，都有很多相关的性能评测工具，我们专栏里面也提到不少</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-16 08:12:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://wx.qlogo.cn/mmopen/vi_32/PiajxSqBRaEKQMM4m7NHuicr55aRiblTSEWIYe0QqbpyHweaoAbG7j2v7UUElqqeP3Ihrm3UfDPDRb1Hv8LvPwXqA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ninuxer</span>
  </div>
  <div class="_2_QraFYR_0">打卡day61<br>这工具，还是要经常用，感觉用各个模块的套路篇，来的更快点～</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-08 08:03:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/6c/40/f5ea25b1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Jiyong</span>
  </div>
  <div class="_2_QraFYR_0">请教老师，有哪些限制cpu 内存等资源使用率的工具。谢谢啊</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-26 09:00:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1c/d3/0e/ef771277.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Akon</span>
  </div>
  <div class="_2_QraFYR_0">大佬，有没有块设备稳定性测试的例子（比如iozone、fio这样的工具，以及制作或眼图的方法）</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-20 10:27:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLRwYJzyB6pdic3GeQvb5FSmMAOXepWERNT1nJsIY7M0adpbrACsdia56DCDxWhWicXHahSiaaIUpgZxg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>youbuxiaoer</span>
  </div>
  <div class="_2_QraFYR_0">老师，工作中遇到内存占用较高，通过meminfo确认是匿名页占用较多内存，请问匿名页内存是如何产生的？如何释放？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 请参考我们专栏第19篇</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-20 08:51:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/9d/2c/6248ffbd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>柏森森</span>
  </div>
  <div class="_2_QraFYR_0">请教一个问题,我在系统中使用find  &#47;query&#47;tmp  -type f  -min +1  -exec rm -rf {} \;<br>用来删除文件发现这个脚本占用的资源特别多，CPU可以占用到20-30%,但系统不断地有交易上来，导致了性能的下降。系统&#47;query&#47;tmp中主要是小文件，请教下有没有好办法可以进行更高效，更快速地删除？另外，find工作的原理是什么</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-18 09:04:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/c0/ce/fc41ad5e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>陳先森</span>
  </div>
  <div class="_2_QraFYR_0">原理大致清楚，但是工具不是太熟悉···还是得多练，概念性的东西当时记住了后面又容易忘，知识还是要温故而知新···还是要多看几遍，多敲几遍才记得住。慢慢的干货</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-07 11:05:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/64/05/6989dce6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我来也</span>
  </div>
  <div class="_2_QraFYR_0">[D57打卡]<br>对性能分析的原理,比之前了解的多多了.<br>工具还是要经常用才行,不用就容易忘记. 有时用man还可以想起一点点来.<br><br>今天又把这些指标和工具又温习了一遍,以后也要常拿出来温习才行.<br>毕竟目前的主要工作还不是性能分析这一块.</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-08 14:36:30</div>
  </div>
</div>
</div>
</li>
</ul>