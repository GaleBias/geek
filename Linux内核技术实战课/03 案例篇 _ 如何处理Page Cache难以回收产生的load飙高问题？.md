<audio title="03 案例篇 _ 如何处理Page Cache难以回收产生的load飙高问题？" src="https://static001.geekbang.org/resource/audio/06/2b/06fef6388306bdcc63edcf79ca73612b.mp3" controls="controls"></audio> 
<p>你好，我是邵亚方。今天这节课，我想跟你聊一聊怎么处理在生产环境中，因为Page Cache管理不当引起的系统load飙高的问题。</p><p>相信你在平时的工作中，应该会或多或少遇到过这些情形：系统很卡顿，敲命令响应非常慢；应用程序的RT变得很高，或者抖动得很厉害。在发生这些问题时，很有可能也伴随着系统load飙得很高。</p><p>那这是什么原因导致的呢？据我观察，大多是有三种情况：</p><ul>
<li>直接内存回收引起的load飙高；</li>
<li>系统中脏页积压过多引起的load飙高；</li>
<li>系统NUMA策略配置不当引起的load飙高。</li>
</ul><p>这是应用开发者和运维人员向我咨询最多的几种情况。问题看似很简单，但如果对问题产生的原因理解得不深，解决起来就会很棘手，甚至配置得不好，还会带来负面的影响。</p><p>所以这节课，我们一起来分析下这三种情况，可以说，搞清楚了这几种情况，你差不多就能解决掉绝大部分Page Cache引起的load飙高问题了。如果你对问题的原因排查感兴趣，也不要着急，在第5讲，我会带你学习load飙高问题的分析方法。</p><p>接下来，我们就来逐一分析下这几类情况。</p><h2>直接内存回收引起load飙高或者业务时延抖动</h2><p>直接内存回收是指在进程上下文同步进行内存回收，那么它具体是怎么引起load飙高的呢？</p><!-- [[[read_end]]] --><p>因为直接内存回收是在进程申请内存的过程中同步进行的回收，而这个回收过程可能会消耗很多时间，进而导致进程的后续行为都被迫等待，这样就会造成很长时间的延迟，以及系统的CPU利用率会升高，最终引起load飙高。</p><p>我们详细地描述一下这个过程，为了尽量不涉及太多技术细节，我会用一张图来表示，这样你理解起来会更容易。</p><p><img src="https://static001.geekbang.org/resource/image/fe/00/fe84eb2bd4956bbbdd5b0259df8c9400.jpg?wh=3000*2912" alt="" title="内存回收过程"></p><p>从图里你可以看到，在开始内存回收后，首先进行后台异步回收（上图中蓝色标记的地方），这不会引起进程的延迟；如果后台异步回收跟不上进程内存申请的速度，就会开始同步阻塞回收，导致延迟（上图中红色和粉色标记的地方，这就是引起load高的地址）。</p><p>那么，针对直接内存回收引起load飙高或者业务RT抖动的问题，一个解决方案就是<strong>及早地触发后台回收来避免应用程序进行直接内存回收，那具体要怎么做呢？</strong></p><p>我们先来了解一下后台回收的原理，如图：</p><p><img src="https://static001.geekbang.org/resource/image/44/72/44d471fdae7376eb13e6e6bfc70b3172.jpg?wh=2440*1300" alt=""></p><p>它的意思是：当内存水位低于watermark low时，就会唤醒kswapd进行后台回收，然后kswapd会一直回收到watermark high。</p><p>那么，我们可以增大min_free_kbytes这个配置选项来及早地触发后台回收，该选项最终控制的是内存回收水位，不过，内存回收水位是内核里面非常细节性的知识点，我们可以先不去讨论。</p><blockquote>
<p>vm.min_free_kbytes = 4194304</p>
</blockquote><p>对于大于等于128G的系统而言，将min_free_kbytes设置为4G比较合理，这是我们在处理很多这种问题时总结出来的一个经验值，既不造成较多的内存浪费，又能避免掉绝大多数的直接内存回收。</p><p>该值的设置和总的物理内存并没有一个严格对应的关系，我们在前面也说过，如果配置不当会引起一些副作用，所以在调整该值之前，我的建议是：你可以渐进式地增大该值，比如先调整为1G，观察sar -B中pgscand是否还有不为0的情况；如果存在不为0的情况，继续增加到2G，再次观察是否还有不为0的情况来决定是否增大，以此类推。</p><p>在这里你需要注意的是，即使将该值增加得很大，还是可能存在pgscand不为0的情况（这个略复杂，涉及到内存碎片和连续内存申请，我们在此先不展开，你知道有这么回事儿就可以了）。那么这个时候你要考虑的是，业务是否可以容忍，如果可以容忍那就没有必要继续增加了，也就是说，增大该值并不是完全避免直接内存回收，而是尽量将直接内存回收行为控制在业务可以容忍的范围内。</p><p>这个方法可以用在3.10.0以后的内核上（对应的操作系统为CentOS-7以及之后更新的操作系统）。</p><p>当然了，这样做也有一些缺陷：提高了内存水位后，应用程序可以直接使用的内存量就会减少，这在一定程度上浪费了内存。所以在调整这一项之前，你需要先思考一下，<strong>应用程序更加关注什么，如果关注延迟那就适当地增大该值，如果关注内存的使用量那就适当地调小该值。</strong></p><p>除此之外，对CentOS-6(对应于2.6.32内核版本)而言，还有另外一种解决方案：</p><blockquote>
<p>vm.extra_free_kbytes = 4194304</p>
</blockquote><p>那就是将extra_free_kbytes 配置为4G。extra_free_kbytes在3.10以及以后的内核上都被废弃掉了，不过由于在生产环境中还存在大量的机器运行着较老版本内核，你使用到的也可能会是较老版本的内核，所以在这里还是有必要提一下。它的大致原理如下所示：</p><p><img src="https://static001.geekbang.org/resource/image/7d/d2/7d9e537e23489cd4f5f34fedcd6f89d2.jpg?wh=2760*1340" alt=""></p><p>extra_free_kbytes的目的是为了解决min_free_kbyte造成的内存浪费，但是这种做法并没有被内核主线接收，因为这种行为很难维护会带来一些麻烦，感兴趣的可以看一下这个讨论：<a href="https://lkml.org/lkml/2013/2/17/210">add extra free kbytes tunable</a></p><p>总的来说，通过调整内存水位，在一定程度上保障了应用的内存申请，但是同时也带来了一定的内存浪费，因为系统始终要保障有这么多的free内存，这就压缩了Page Cache的空间。调整的效果你可以通过/proc/zoneinfo来观察：</p><pre><code>$ egrep &quot;min|low|high&quot; /proc/zoneinfo 
...
        min      7019
        low      8773
        high     10527
...
</code></pre><p>其中min、low、high分别对应上图中的三个内存水位。你可以观察一下调整前后min、low、high的变化。需要提醒你的是，内存水位是针对每个内存zone进行设置的，所以/proc/zoneinfo里面会有很多zone以及它们的内存水位，你可以不用去关注这些细节。</p><h2>系统中脏页过多引起load飙高</h2><p>接下来，我们分析下由于系统脏页过多引起load飙高的情况。在前一个案例中我们也提到，直接回收过程中，如果存在较多脏页就可能涉及在回收过程中进行回写，这可能会造成非常大的延迟，而且因为这个过程本身是阻塞式的，所以又可能进一步导致系统中处于D状态的进程数增多，最终的表现就是系统的load值很高。</p><p>我们来看一下这张图，这是一个典型的脏页引起系统load值飙高的问题场景：</p><p><img src="https://static001.geekbang.org/resource/image/90/75/90c693c95d67cfaf89b86edbd1228d75.jpg?wh=3200*2500" alt=""></p><p>如图所示，如果系统中既有快速I/O设备，又有慢速I/O设备（比如图中的ceph RBD设备，或者其他慢速存储设备比如HDD），直接内存回收过程中遇到了正在往慢速I/O设备回写的page，就可能导致非常大的延迟。</p><p>这里我多说一点。这类问题其实是不太好去追踪的，为了更好追踪这种慢速I/O设备引起的抖动问题，我也给Linux Kernel提交了一个patch来进行更好的追踪：<a href="https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=19343b5bdd16ad4ae6b845ef829f68b683c4dfb5">mm/page-writeback: introduce tracepoint for wait_on_page_writeback()</a>，这种做法是在原来的基础上增加了回写的设备，这样子用户就能更好地将回写和具体设备关联起来，从而判断问题是否是由慢速I/O设备导致的（具体的分析方法我会在后面第5讲分析篇里重点来讲）。</p><p>那如何解决这类问题呢？一个比较省事的解决方案是控制好系统中积压的脏页数据。很多人知道需要控制脏页，但是往往并不清楚如何来控制好这个度，脏页控制的少了可能会影响系统整体的效率，脏页控制的多了还是会触发问题，所以我们接下来看下如何来衡量好这个“度”。</p><p>首先你可以通过sar -r来观察系统中的脏页个数：</p><pre><code>$ sar -r 1
07:30:01 PM kbmemfree kbmemused  %memused kbbuffers  kbcached  kbcommit   %commit  kbactive   kbinact   kbdirty
09:20:01 PM   5681588   2137312     27.34         0   1807432    193016      2.47    534416   1310876         4
09:30:01 PM   5677564   2141336     27.39         0   1807500    204084      2.61    539192   1310884        20
09:40:01 PM   5679516   2139384     27.36         0   1807508    196696      2.52    536528   1310888        20
09:50:01 PM   5679548   2139352     27.36         0   1807516    196624      2.51    536152   1310892        24
</code></pre><p>kbdirty就是系统中的脏页大小，它同样也是对/proc/vmstat中nr_dirty的解析。你可以通过调小如下设置来将系统脏页个数控制在一个合理范围:</p><blockquote>
<p>vm.dirty_background_bytes = 0<br>
vm.dirty_background_ratio = 10<br>
vm.dirty_bytes = 0<br>
vm.dirty_expire_centisecs = 3000<br>
vm.dirty_ratio = 20</p>
</blockquote><p>调整这些配置项有利有弊，调大这些值会导致脏页的积压，但是同时也可能减少了I/O的次数，从而提升单次刷盘的效率；调小这些值可以减少脏页的积压，但是同时也增加了I/O的次数，降低了I/O的效率。</p><p><strong>至于这些值调整大多少比较合适，也是因系统和业务的不同而异，我的建议也是一边调整一边观察，将这些值调整到业务可以容忍的程度就可以了，即在调整后需要观察业务的服务质量(SLA)，要确保SLA在可接受范围内</strong>。调整的效果你可以通过/proc/vmstat来查看：</p><pre><code>$ grep &quot;nr_dirty_&quot; /proc/vmstat
nr_dirty_threshold 366998
nr_dirty_background_threshold 183275
</code></pre><p>你可以观察一下调整前后这两项的变化。<strong>这里我要给你一个避免踩坑的提示</strong>，解决该方案中的设置项如果设置不妥会触发一个内核Bug，这是我在2017年进行性能调优时发现的一个内核Bug，我给社区提交了一个patch将它fix掉了，具体的commit见<a href="https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=94af584692091347baea4d810b9fc6e0f5483d42">writeback: schedule periodic writeback with sysctl</a> <a href="https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=94af584692091347baea4d810b9fc6e0f5483d42"></a> , commit log清晰地描述了该问题，我建议你有时间看一看。</p><h2>系统NUMA策略配置不当引起的load飙高</h2><p>除了我前面提到的这两种引起系统load飙高或者业务延迟抖动的场景之外，还有另外一种场景也会引起load飙高，那就是系统NUMA策略配置不当引起的load飙高。</p><p>比如说，我们在生产环境上就曾经遇到这样的问题：系统中还有一半左右的free内存，但还是频频触发direct reclaim，导致业务抖动得比较厉害。后来经过排查发现是由于设置了zone_reclaim_mode，这是NUMA策略的一种。</p><p>设置zone_reclaim_mode的目的是为了增加业务的NUMA亲和性，但是在实际生产环境中很少会有对NUMA特别敏感的业务，这也是为什么内核将该配置从默认配置1修改为了默认配置0: <a href="https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=4f9b16a64753d0bb607454347036dc997fd03b82">mm: disable zone_reclaim_mode by default</a>  ，配置为0之后，就避免了在其他node有空闲内存时，不去使用这些空闲内存而是去回收当前node的Page Cache，也就是说，通过减少内存回收发生的可能性从而避免它引发的业务延迟。</p><p>那么如何来有效地衡量业务延迟问题是否由zone reclaim引起的呢？它引起的延迟究竟有多大呢？这个衡量和观察方法也是我贡献给Linux Kernel的：<a href="https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=132bb8cfc9e081238e7e2fd0c37c8c75ad0d2963">mm/vmscan: add tracepoints for node reclaim</a>  ，大致的思路就是利用linux的tracepoint来做这种量化分析，这是性能开销相对较小的一个方案。</p><p>我们可以通过numactl来查看服务器的NUMA信息，如下是两个node的服务器：</p><pre><code>$ numactl --hardware
available: 2 nodes (0-1)
node 0 cpus: 0 1 2 3 4 5 6 7 8 9 10 11 24 25 26 27 28 29 30 31 32 33 34 35
node 0 size: 130950 MB
node 0 free: 108256 MB
node 1 cpus: 12 13 14 15 16 17 18 19 20 21 22 23 36 37 38 39 40 41 42 43 44 45 46 47
node 1 size: 131072 MB
node 1 free: 122995 MB
node distances:
node   0   1 
  0:  10  21 
  1:  21  10 
</code></pre><p>其中CPU0～11，24～35的local node为node 0；而CPU12～23，36～47的local node为node 1。如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/80/e2/80e7c19a8f310d5bf30d368cef86bbe2.jpg?wh=2260*1115" alt=""></p><p>推荐将zone_reclaim_mode配置为0。</p><blockquote>
<p>vm.zone_reclaim_mode = 0</p>
</blockquote><p>因为相比内存回收的危害而言，NUMA带来的性能提升几乎可以忽略，所以配置为0，利远大于弊。</p><p>好了，对于Page Cache管理不当引起的系统load飙高和业务时延抖动问题，我们就分析到这里，希望通过这篇的学习，在下次你遇到直接内存回收引起的load飙高问题时不再束手无策。</p><p>总的来说，这些问题都是Page Cache难以释放而产生的问题，那你是否想过，是不是Page Cache很容易释放就不会产生问题了？这个答案可能会让你有些意料不到：Page Cache容易释放也有容易释放的问题。这到底是怎么回事呢，我们下节课来分析下这方面的案例。</p><h2>课堂总结</h2><p>这节课我们讲的这几个案例都是内存回收过程中引起的load飙高问题。关于内存回收这事，我们可以做一个形象的类比。我们知道，内存是操作系统中很重要的一个资源，它就像我们在生活过程中很重要的一个资源——钱一样，如果你的钱（内存）足够多，那想买什么就可以买什么，而不用担心钱花完（内存用完）后要吃土（引起load飙高）。</p><p>但是现实情况是我们每个人用来双十一购物的钱（内存）总是有限的，在买东西（运行程序）的时候总需要精打细算，一旦预算快超了（内存快不够了），就得把一些不重要的东西（把一些不活跃的内容）从购物车里删除掉（回收掉），好腾出资金（空闲的内存）来买更想买的东西（运行需要运行的程序）。</p><p>我们讲的这几个案例都可以通过调整系统参数/配置来解决，调整系统参数/配置也是应用开发者和运维人员在发生了内核问题时所能做的改动。比如说，直接内存回收引起load飙高时，就去调整内存水位设置；脏页积压引起load飙高时，就需要去调整脏页的水位；NUMA策略配置不当引起load飙高时，就去检查是否需要关闭该策略。同时我们在做这些调整的时候，一定要边调整边观察业务的服务质量，确保SLA是可以接受的。</p><p>如果你想要你的系统更加稳定，你的业务性能更好，你不妨去研究一下系统中的可配置项，看看哪些配置可以帮助你的业务。</p><h2>课后作业</h2><p>这节课我给你布置的作业是针对直接内存回收的，现在你已经知道直接内存回收容易产生问题，是我们需要尽量避免的，那么我的问题是：请你执行一些模拟程序来构造出直接内存回收的场景（小提示： 你可以通过sar -B中的pgscand来判断是否有了直接内存回收）。欢迎在留言区分享你的看法。</p><p>感谢你的阅读，如果你认为这节课的内容有收获，也欢迎把它分享给你的朋友，我们下一讲见。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/43/79/18073134.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>test</span>
  </div>
  <div class="_2_QraFYR_0">系统load飙高：<br>1.直接内存回收：<br>观察：sar -B中pgscank&#47;s、pgscand&#47;s表示扫描的页面数量，前者表示kswapd扫描结果，后者表示直接扫描。需要让直接扫描越小越好。<br>解决：设置vm.min_free_kbytes，尽早开始后台回收。<br>2.脏页积压：<br>观察：sar -r中kbdirty即是脏页大小。<br>解决：设置vm&#47;dirtyxxx控制脏页个数在合理范围内。<br>3.NUMA设置不合理：<br>观察：numactl --hardware查看是否还有一半内存空闲，但是还是频频发生direct reclaim。<br>解决：vm.zone_reclaim_mode = 0</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-30 12:42:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/87/75/03aa0ab6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lookzy</span>
  </div>
  <div class="_2_QraFYR_0">老师您好！我们都说剩余内存达到low水位时，就会进行内存回收。但每个zone都有水位，剩余内存是指某个zone里的剩余内存，还是node，还是整个系统的剩余内存？还有内存回收只针对当前zone，还是node，还是整个系统？谢谢！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这是三个不同的维度：系统 -&gt; node -&gt; zone。我们调整的水位最终是提现在zone上，回收也是针对zone的。进程在申请内存时其实是会指定zone的，只是应用程序并不感知这个细节，对应用程序而言，它申请的内存一般都是normal zone，那么当normal zone中的剩余内存不足时就会触发回收；这里面又会涉及到node概念，假设有2个node，node 0的normal zone里剩余内存不足了就会唤醒kswapd0来回收node 0里normal zone的内存，但是此时应用是可以去继续申请node 1的normal zone的内存的（如果没有配置numa策略），如果node 1的normal zone剩余内存也不足了，那会唤醒kswapd1来回收node 1的normal zone。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-27 18:27:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/83/14/099742ae.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>xzy</span>
  </div>
  <div class="_2_QraFYR_0">你好请教个问题：我们线上出现 kswapd0 占用 cpu 较高，free -h 看了available 的内存不到 1g，应该是可用内存太少触发了 kswapd0 回收内存。我们关闭了 swap，vmstat  -s 没发现 page 换入换出，但是用 <br> iotop 却发现大量进程 io 很高（这些业务进程并没有 io 的逻辑），请问这种情况如何定位，是否是 kswapd0 导致的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: kswapd占用cpu高就意味着回收很困难，也就是系统中存在大量不可回收的内存，或者pagecache的回收较困难，比如脏页太多。kswapd本身回进行脏页的回写，但是iotop可以看到是kswapd在进行io；如果是业务进程io高，那有可能是因为它的page cache被回收掉，导致它不得不再次从磁盘来读数据，这个时候应该有很多pagein才会。你也可以观察下workingset refault，看看这个指标是否也高。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-27 20:39:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/47/00/3202bdf0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>piboye</span>
  </div>
  <div class="_2_QraFYR_0">老师有个dirty page的问题想请教一下。 1创建文件；2写入日志数据，3 移除文件；4 关闭文件。我想知道这种情况情况下dirty page还会去刷盘不？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这是个好问题。<br><br>首先我们假设这个文件只有这一个进程打开。<br><br>如果是先关闭文件，再移除文件，那么移除文件时就会把这些dirty page释放掉，无需刷盘。<br>如果是先移除文件，那么在移除文件时由于该文件还有引用（还没有close），那么这些dirty pages还是会保留在内存中，接下来如果系统内存紧张了，就会触发回收内存，回收的过程中就可能会回写这些dirty pages，也就是说删除文件只是path不在了但是inode还在；如果没有回收行为发生，那么就不会去回写这些dirty pages，等close的时候就会把这些dirty pages释放掉。<br>另外 ，你说的这个使用文件的方式是进程创建临时文件的方法，也就是进程意外退出后不留下文件痕迹的方法。<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-23 09:56:26</div>
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
  <div class="_2_QraFYR_0">老师可以再解释下为什么调大min_free_kbytes会减少回收频率而降低延迟吗?而调小该值就会减少内存浪费呢？我的理解是增加min_free_kbytes之后，free剩余内存更大的时候就触发了回收，这不是增加回收频率了吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 调大min_free_kbytes其实是为了更早唤醒kswapd，从而避免直接回收，它降低的是直接回收的频率，会增加了kswapd后台回收的频率。对应用而言，直接回收会导致它的延迟，这个危害较大；如果后台回收太频繁，那后台回收线程kswapd可能会跟应用抢cpu，这也会间接影响应用，只是没那么明显，但也尽量要避免kswapd太消耗cpu，也就是不能把min_free_kbytes设置太大，合适就好。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-27 16:51:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/a6/3f/5cd75d31.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>蛋卷儿</span>
  </div>
  <div class="_2_QraFYR_0">请教老师,kswapd和direct reclaim如果都是把内存回收到zone里high的水位,那么如果某一次内存申请数量大于zone的high值,是否就会导致os kill？我们线上机器的zone normal high是46M,那某次申请内存超过46Mb是不是就挂了？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: memory watermark主要是用来内存回收用的。在申请内存时，主要是看空闲内存（free list）是否足够，如果足够就可分配，如果不够的话，会先触发回收&#47;规整，然后如果还不能分配到足够内存的话，就可能会产生oom。<br>所以zone normal high和触发os kill并没有直接关系。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-24 20:16:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/20/bd/2a/e86058d3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Cooper</span>
  </div>
  <div class="_2_QraFYR_0">您好，请教一下，我以前单位部署多个小场景服务器， node写的，每个服务有最大服务（内存）上限，当时服务部署是将进程绑定指定cpu，1个内核2进程，请问这个设计是否是为了muna 策略，把zone_reclaim_mode设置1，如果是，这样对性能提升有帮助吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 进程绑定主要是减少调度开销以及cpu争抢，另外也可以提升cache命中率，将进程绑定后也可以更好的使用numa。理论上设置为1是会提升性能的，还要看node的kswapd是否能够及时回收内存，如果不能，那么设置为1反而会降低性能。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-06 20:11:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/ed/a2/905e32fc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>咸鱼</span>
  </div>
  <div class="_2_QraFYR_0">老师,请教下改变内存水位为什么会影响内存可使用量</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 比如说min水位，这部分内存只有在部分场景下才能申请，是为了应对紧急内存申请，比如说atomic这种申请方式。那么min水位之下的内存，对于应用程序而言，就是无法申请的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-26 11:45:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/a5/5b/e1ffd5e4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>纸君</span>
  </div>
  <div class="_2_QraFYR_0">请问一下 发生 直接内存回收时，是占用内核态还是用户态的cpu?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 直接内存回收是在内核代码里执行的 它是属于内核态</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-01 15:06:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/18/ac/4d68ba46.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>金时</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，文中说的第一种情况“直接内存回收引起 load 飙高或者业务时延抖动“， 和第二种情况“系统中脏页过多引起 load 飙高“， 不太理解这两种情况的区别。  第一种情况回收的“内存“，这里的“内存”不是指第二种情况的“脏页”吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 包括脏页，可以理解为第二种情况是第一种情况的子集。<br>在直接内存回收时，可能系统中脏页很少，但是其他情况也会引起延迟，比如说内存规整（即系统内存碎片很严重时）。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-18 12:21:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/0c/bf/863d440a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jaxzhai</span>
  </div>
  <div class="_2_QraFYR_0">kernel: INFO: task kswapd0:224 blocked for more than 120 seconds.  这段时间大数据系统，进程出现这种情况，之后kswapd0进程变成僵尸进程了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这是什么内核版本？这个现象可以复现吗？如果可以的话。你可以使用sysrq把线程栈打印出来看看，根据线程栈就可以大概做出判断。<br>另外内核线程是不会变成僵尸进程的，你指的应该是kswapd变成了D状态？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-02 20:16:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/9a/83/0802f4e7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_lucky_brian</span>
  </div>
  <div class="_2_QraFYR_0">老师好，老师文中的几个问题和MySQL数据库的性能和稳定性都有很大联系，让我收益颇多。<br>（1）之前定位到当内存不足时，产生pgscand，数据库的io会产生问题，包括服务质量会下降，且会导致与客户端断链。之前是通过按照固定间隔去drop_cache解决，现在已经引入了vm.min_free_bytes的方式。<br>（2）关于numa，我们发现在x86上并不是很敏感，但是在arm平台上，一旦CPU为高核（150以上），就发现MySQL数据库性能压测上不去，这时候通过numa绑核就可以解决掉性能问题。这种场景，老师是否建议开启zone_reclaim_mode？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对于（2），可能是跟arm的cache小有关系；不清楚你们具体是如何来绑核的，如果是因为numa亲核性导致的性能问题，那可以将zone_recliam_mode打开，这样numa的亲和性会更好些 不过我建议你最好在线下先压测观察下，因为这个的机制存在一些缺陷 打开它可能会引起一些问题。也欢迎与我交流你们打开这个之后的现象与问题。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-13 17:42:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/5a/56/115c6433.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jssfy</span>
  </div>
  <div class="_2_QraFYR_0">请问在虚机或者容器场景，vm.min_free_kbytes这一部分的空间建议预留还是也作为可分配空间呢?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个要看具体的业务类型，其实和虚拟机或者容器关系不大，如果虚拟机或者容器中的业务对延迟比较敏感，那你需要适当的调大这个值。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-29 16:11:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/5c/0a/1bd98d4b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>vearne</span>
  </div>
  <div class="_2_QraFYR_0">stress --vm 10  --vm-hang 30 --vm-bytes 1024000000</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-22 17:44:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/tKvmZ3Vs4t6RZ3X7cAliaW4OobyW6v7GwmIyg77VVKYGAY7iaf0VibF5TkibTSPY03PX9qyYUjOoXXxdTdVSNmW9Yw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_Q</span>
  </div>
  <div class="_2_QraFYR_0">老师好，请教一下，最近生产服务器遇到page cache吃满内存后rtt大的问题。应用程序是多线程，线程A写所有收到的数据到ssd磁盘,线程B网络收发，线程C是业务，所有内存都是启动就开好，运行时百毫秒用json库申请少量堆内存用于心跳，应用程序约占内存200GB。page cache把内存占满后，有时线程A卡死三秒pgscand 10000多，看sar当时有500MB pageout；有时线程B卡死一秒，pgsand为0，线程C也卡但不影响RTT。服务器32核，1TB内存，四个node，不绑核。目前知道跟内存满有关，但为什么这么好性能机器，线程卡住3秒不动，还是解释不了，请求帮忙分析下。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-05 20:54:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/3e/3c/fc3ad983.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>佳伦</span>
  </div>
  <div class="_2_QraFYR_0">脏页积压和水位线不是一回事吗？不是因为水位线太高导致了积压吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-20 17:50:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKSGEOl6BlJOLpBSrct7ibvic616p7SXbwCDl9IzftYdMsibKPYUZpkKJbP0Lu7JcFEb2ajEqu7M2TMw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>amos</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，文中说直接内存回收过程中遇到了正在往慢速 I&#47;O 设备回写的 page，就可能导致非常大的延迟，这句话不太理解，这个应该是异步写的，为什么还会有较大延迟？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-27 20:03:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKSGEOl6BlJOLpBSrct7ibvic616p7SXbwCDl9IzftYdMsibKPYUZpkKJbP0Lu7JcFEb2ajEqu7M2TMw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>amos</span>
  </div>
  <div class="_2_QraFYR_0">老师好，慢速 I&#47;O 设备回写的 page,可能会有很大延迟 这个怎么排查，我看05里面没有呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-26 19:20:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/c4/35/2cc10d43.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Wade_阿伟</span>
  </div>
  <div class="_2_QraFYR_0">系统中还有一半左右的 free 内存，但还是频频触发 direct reclaim，导致业务抖动得比较厉害。后来经过排查发现是由于设置了 zone_reclaim_mode，这是 NUMA 策略的一种。请问老师能分享下这个问题的排查思路吗 ？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 最简单的方式是kprobe内核里direct reclaim的函数，看看它是通过什么路径来触发的，有了执行路径后，具体的原因就很容易搞清楚了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-11 16:33:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJpJXWFP3dNle88WnTkRTsEQkPJmOhepibiaTfhEtMRrbdg5EAWm4EzurA61oKxvCK2ZjMmy1QvmChw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>唐江</span>
  </div>
  <div class="_2_QraFYR_0">文件被程序读取到内存，pagecache缓存的是部分吧，如果这时候把文件删除了，程序还可以读到部分缓存的数据吧？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对的 。把文件删除后 它的内容还在内存中 还是可以读到的 直到这部分内存被释放。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-20 19:14:34</div>
  </div>
</div>
</div>
</li>
</ul>