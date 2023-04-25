<audio title="17 基础篇 _ CPU是如何执行任务的？" src="https://static001.geekbang.org/resource/audio/8c/fc/8cb9696ba6fde7d76b15e3eb0466d7fc.mp3" controls="controls"></audio> 
<p>你好，我是邵亚方。</p><p>如果你做过性能优化的话，你应该有过这些思考，比如说：</p><ul>
<li>如何让CPU读取数据更快一些？</li>
<li>同样的任务，为什么有时候执行得快，有时候执行得慢？</li>
<li>我的任务有些比较重要，CPU如果有争抢时，我希望可以先执行这些任务，这该怎么办呢？</li>
<li>多线程并行读写数据是如何保障同步的？</li>
<li>…</li>
</ul><p>要想明白这些问题，你就需要去了解CPU是如何执行任务的，只有明白了CPU的执行逻辑，你才能更好地控制你的任务执行，从而获得更好的性能。</p><h2>CPU是如何读写数据的 ？</h2><p>我先带你来看下CPU的架构，因为你只有理解了CPU的架构，你才能更好地理解CPU是如何执行指令的。CPU的架构图如下所示：</p><p><img src="https://static001.geekbang.org/resource/image/a4/7f/a418fbfc23d96aeb4813f1db4cbyy17f.jpg?wh=2598*1617" alt="" title="CPU架构"></p><p>你可以直观地看到，对于现代处理器而言，一个实体CPU通常会有两个逻辑线程，也就是上图中的Core 0和Core  1。每个Core都有自己的L1 Cache，L1 Cache又分为dCache和iCache，对应到上图就是L1d和L1i。L1 Cache只有Core本身可以看到，其他的Core是看不到的。同一个实体CPU中的这两个Core会共享L2 Cache，其他的实体CPU是看不到这个L2 Cache的。所有的实体CPU会共享L3 Cache。这就是典型的CPU架构。</p><!-- [[[read_end]]] --><p>相信你也看到，在CPU外还会有内存（DRAM）、磁盘等，这些存储介质共同构成了体系结构里的金字塔存储层次。如下所示：</p><p><img src="https://static001.geekbang.org/resource/image/6e/3d/6eace3466bc42185887a351c6c3e693d.jpg?wh=2535*1705" alt="" title="金字塔存储层次"></p><p>在这个“金字塔”中，越往下，存储容量就越大，它的速度也会变得越慢。Jeff Dean曾经研究过CPU对各个存储介质的访问延迟，具体你可以看下<a href="https://gist.github.com/jboner/2841832">latency</a>里的数据，里面详细记录了每个存储层次的访问延迟，这也是我们在性能优化时必须要知道的一些延迟数据。你可不要小瞧它，在某些场景下，这些不同存储层次的访问延迟差异可能会导致非常大的性能差异。</p><p>我们就以Cache访问延迟（L1 0.5ns，L2 10ns）和内存访问延迟（100ns）为例，我给你举一个实际的案例来说明访问延迟的差异对性能的影响。</p><p>之前我在做网络追踪系统时，为了更方便地追踪TCP连接，我给Linux Kernel提交了一个PATCH来记录每个连接的编号，具体你可以参考这个commit：<a href="https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?h=v5.9-rc4&amp;id=c6849a3ac17e336811f1d5bba991d2a9bdc47af1">net: init sk_cookie for inet socket</a>。该PATCH的大致作用是，在每次创建一个新的TCP连接时（关于TCP这部分知识，你可以去温习上一个模块的内容），它都会使用net namespace（网络命名空间）中的cookie_gen生成一个cookie给这个新建的连接赋值。</p><p>可是呢，在这个PATCH被合入后，Google工程师Eric Dumazet发现在他的SYN Flood测试中网络吞吐量会下降约24%。后来经过分析发现，这是因为net namespace中所有TCP连接都在共享cookie_gen。在高并发情况下，瞬间会有非常多的新建TCP连接，这时候cookie_gen就成了一个非常热的数据，从而被缓存在Cache中。如果cookie_gen的内容被修改的话，Cache里的数据就会失效，那么当有其他新建连接需要读取这个数据时，就不得不再次从内存中去读取。而你知道，内存的延迟相比Cache的延迟是大很多的，这就导致了严重的性能下降。这个问题就是典型的False Sharing，也就是Cache伪共享问题。</p><p>正因为这个PATCH给高并发建连这种场景带来了如此严重的性能损耗，所以它就被我们给回退（Revert）了，你具体可以看<a href="https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?h=v5.9-rc4&amp;id=a06ac0d67d9fda7c255476c6391032319030045d">Revert "net: init sk_cookie for inet socket"</a>这个commit。不过，cookie_gen对于网络追踪还是很有用的，比如说在使用ebpf来追踪cgroup的TCP连接时，所以后来Facebook的一个工程师把它从net namespace这个结构体里移了出来，<a href="https://github.com/torvalds/linux/commit/cd48bdda4fb82c2fe569d97af4217c530168c99c">改为了一个全局变量</a>。</p><p>由于net namespace被很多TCP连接共享，因此这个结构体非常容易产生这类Cache伪共享问题，Eric Dumazet也在这里引入过一个Cache伪共享问题：<a href="https://github.com/torvalds/linux/commit/2a06b8982f8f2f40d03a3daf634676386bd84dbc">net: reorder ‘struct net’ fields to avoid false sharing</a>。</p><p>接下来，我们就来看一下Cache伪共享问题究竟是怎么回事。</p><p><img src="https://static001.geekbang.org/resource/image/ed/9f/ed552cedfb95d0a3af920eca78c3069f.jpg?wh=2780*1583" alt="" title="Cache Line False Sharing"></p><p>如上图所示，两个CPU上并行运行着两个不同线程，它们同时从内存中读取两个不同的数据，这两个数据的地址在物理内存上是连续的，它们位于同一个Cache Line中。CPU从内存中读数据到Cache是以Cache Line为单位的，所以该Cache Line里的数据被同时读入到了这两个CPU的各自Cache中。紧接着这两个线程分别改写不同的数据，每次改写Cache中的数据都会将整个Cache Line置为无效。因此，虽然这两个线程改写的数据不同，但是由于它们位于同一个Cache Line中，所以一个CPU中的线程在写数据时会导致另外一个CPU中的Cache Line失效，而另外一个CPU中的线程在读写数据时就会发生cache miss，然后去内存读数据，这就大大降低了性能。</p><p>Cache伪共享问题可以说是性能杀手，我们在写代码时一定要留意那些频繁改写的共享数据，必要的时候可以将它跟其他的热数据放在不同的Cache Line中避免伪共享问题，就像我们在内核代码里经常看到的____cacheline_aligned所做的那样。</p><p>那怎么来观测Cache伪共享问题呢？你可以使用<a href="https://man7.org/linux/man-pages/man1/perf-c2c.1.html">perf c2c</a>这个命令，但是这需要较新版本内核支持才可以。不过，perf同样可以观察cache miss的现象，它对很多性能问题的分析还是很有帮助的。</p><p>CPU在写完Cache后将Cache置为无效（invalidate）, 这本质上是为了保障多核并行计算时的数据一致性，一致性问题是Cache这个存储层次很典型的问题。</p><p>我们再来看内存这个存储层次中的典型问题：并行计算时的竞争，即两个CPU同时去操作同一个物理内存地址时的竞争。关于这类问题，我举一些简单的例子给你说明一下。</p><p>以C语言为例：</p><pre><code>struct foo {
    int a;
    int b;
};
</code></pre><p>在这段示例代码里，我们定义了一个结构体，该结构体里的两个成员a和b在地址上是连续的。如果CPU 0去写a，同时CPU 1去读b的话，此时不会有竞争，因为a和b是不同的地址。不过，a和b由于在地址上是连续的，它们可能会位于同一个Cache Line中，所以为了防止前面提到的Cache伪共享问题，我们可以强制将b的地址设置为Cache Line对齐地址，如下:</p><pre><code>struct foo {
    int a;
    int b ____cacheline_aligned;
};
</code></pre><p>接下来，我们看下另外一种情况：</p><pre><code>struct foo {
    int a:1;
    int b:1;
};
</code></pre><p>这个示例程序定义了两个位域（bit field），a和b的地址是一样的，只是属于该地址的不同bit。在这种情况下，CPU 0去写a （a = 1），同时CPU 1去写b （b = 1），就会产生竞争。在总线仲裁后，先写的数据就会被后写的数据给覆盖掉。这就是执行RMW操作时典型的竞争问题。在这种场景下，就需要同步原语了，比如使用atomic操作。</p><p>关于位操作，我们来看一个实际的案例。这是我前段时间贡献给Linux内核的一个PATCH：<a href="https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?h=v5.9-rc4&amp;id=1066d1b6974e095d5a6c472ad9180a957b496cd6">psi: Move PF_MEMSTALL out of task-&gt;flags</a>，它在struct task_struct这个结构体里增加了一个in_memstall的位域，在该PATCH里无需考虑多线程并行操作该位域时的竞争问题，你知道这是为什么吗？我将它作为一个课后思考题留给你，欢迎你在留言区与我讨论交流。为了让这个问题简单些，我给你一个提示：如果你留意过task_struct这个结构体里的位域被改写的情况，你会发现只有current（当前在运行的线程）可以写，而其他线程只能去读。但是PF_*这些全局flag可以被其他线程写，而不仅仅是current来写。</p><p>Linux内核里的task_struct结构体就是用来表示一个线程的，每个线程都有唯一对应的task_struct结构体，它也是内核进行调度的基本单位。我们继续来看下CPU是如何选择线程来执行的。</p><h2>CPU是如何选择线程执行的 ？</h2><p>你知道，一个系统中可能会运行着非常多的线程，这些线程数可能远超系统中的CPU核数，这时候这些任务就需要排队，每个CPU都会维护着自己运行队列（runqueue）里的线程。这个运行队列的结构大致如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/66/62/6649d7e5984a3b9cd003fcbc97bfde62.jpg?wh=3245*1930" alt="" title="CPU运行队列"></p><p>每个CPU都有自己的运行队列（runqueue），需要运行的线程会被加入到这个队列中。因为有些线程的优先级高，Linux内核为了保障这些高优先级任务的执行，设置了不同的调度类（Scheduling Class），如下所示：</p><p><img src="https://static001.geekbang.org/resource/image/15/b1/1507d0ef23d5d1cd33769dd1953cffb1.jpg?wh=3017*1218" alt=""></p><p>这几个调度类的优先级如下：Deadline &gt; Realtime &gt; Fair。Linux内核在选择下一个任务执行时，会按照该顺序来进行选择，也就是先从dl_rq里选择任务，然后从rt_rq里选择任务，最后从cfs_rq里选择任务。所以实时任务总是会比普通任务先得到执行。</p><p>如果你的某些任务对延迟容忍度很低，比如说在嵌入式系统中就有很多这类任务，那就可以考虑将你的任务设置为实时任务，比如将它设置为SCHED_FIFO的任务：</p><blockquote>
<p>$ chrt -f -p 1 1327</p>
</blockquote><p>如果你不做任何设置的话，用户线程在默认情况下都是普通线程，也就是属于Fair调度类，由CFS调度器来进行管理。CFS调度器的目的是为了实现线程运行的公平性，举个例子，假设一个CPU上有两个线程需要执行，那么每个线程都将分配50%的CPU时间，以保障公平性。其实，各个线程之间执行时间的比例，也是可以人为干预的，比如在Linux上可以调整进程的nice值来干预，从而让优先级高一些的线程执行更多时间。这就是CFS调度器的大致思想。</p><p>好了，我们这堂课就先讲到这里。</p><h2>课堂总结</h2><p>我来总结一下这节课的知识点：</p><ul>
<li>要想明白CPU是如何执行任务的，你首先需要去了解CPU的架构；</li>
<li>CPU的存储层次对大型软件系统的性能影响会很明显，也是你在性能调优时需要着重考虑的；</li>
<li>高并发场景下的Cache Line伪共享问题是一个普遍存在的问题，你需要留意一下它；</li>
<li>系统中需要运行的线程数可能大于CPU核数，这样就会导致线程排队等待CPU，这可能会导致一些延迟。如果你的任务对延迟容忍度低，你可以通过一些手段来人为干预Linux默认的调度策略。</li>
</ul><h2>课后作业</h2><p>这节课的作业就是我们前面提到的思考题：在<a href="https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?h=v5.9-rc4&amp;id=1066d1b6974e095d5a6c472ad9180a957b496cd6">psi: Move PF_MEMSTALL out of task-&gt;flags</a>这个PATCH中，为什么没有考虑多线程并行操作新增加的位域（in_memstall）时的竞争问题？欢迎你在留言区与我讨论。</p><p>感谢你的阅读，如果你认为这节课的内容有收获，也欢迎把它分享给你的朋友，我们下一讲见。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/3b/d7/9d942870.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>邵亚方</span>
  </div>
  <div class="_2_QraFYR_0">课后作业答案：<br>- 这节课的作业就是我们前面提到的思考题：在psi: Move PF_MEMSTALL out of task-&gt;flags这个 PATCH 中，为什么没有考虑多线程并行操作新增加的位域（in_memstall）时的竞争问题？<br>评论区很多同学回到的都很好。<br>因为新增的位域（in_memstall）只有current会写，而其他线程是只读，所以不存在并行写的问题，也就是没有竞争。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-11 14:06:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/48/a9/3177e3ac.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>sufish</span>
  </div>
  <div class="_2_QraFYR_0">1.不用考虑多线程并发访问：是因为调度机制已经保证了同一时间同个线程只会有一个cpu处理，相当于同一时间只会有一个cpu访问task_struct ，所以就避免了并发访问的问题？<br>2.关于Cache 伪共享问题的问题，是否可以考虑在全局变量的前面进行空填充，让这个变量在一个cache line里除了他之外，其他的变量都是占位用的。我猜 ____cacheline_aligned; 实际上也是干这个事</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. 回答的准确。<br>2. 是的，____cacheline_aligned;就是干这个事的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-28 14:36:46</div>
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
  <div class="_2_QraFYR_0">课后思考题,如果不是老师的提示,估计我们是没法理解的了.<br>由于PF_*可能被其他线程写,而task_struct只能被当前运行的线程写,所以这样更改后最终的效果都一样,但是避免了写时冲突的可能.<br><br>我有两个疑问:<br>1. 这样调整后,实际的性能有多少提升呢?<br>   写时冲突是减少了,但读时,理论上还是可能引发类似伪共享的问题吧.<br>   (由于我对内核不太了解,也不清楚`current-&gt;flags`与`current-&gt;in_memstall`在其他地方被使用的频率.还请老师见谅)<br><br>2. 如何获取patch的更多上下文信息呢?<br>   老师对内核提交了很多patch,我很是佩服.<br>   每次commit的message也写的比较详细了.<br>   但我还是想了解更多相关的信息,比如别人的一些讨论,中间做过的一些尝试等等.<br>   目前老师附带的链接中只能看到`Link`(https:&#47;&#47;lkml.kernel.org&#47;r&#47;1584408485-1921-1-git-send-email-laoar.shao@gmail.com)<br>   由这个Link可以查看到`Subject`,`Message-ID`.<br>   但尝试了很久也没找类似issue的讨论信息.<br><br>   由于linux内核维护有自己的一套流程,好像走的邮件列表,我们外行对这些不太了解.<br>   如果老师觉得这个问题太简单的话,能否给一个学习资料,便于我们去学习和入门如何获取更多的patch相关信息?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你的回答很正确。<br><br>关于你的疑问：<br>1. 这样调整的目的不是为了性能提升，而是为了解决PF_*不足的问题。理论上是会存在cache伪共享的问题，不过改写的地方都是在慢速路径上，所以不会成为性能瓶颈。<br><br>2. 方式有很多：<br>1）. 去lore上查找对应子系统的信息https:&#47;&#47;lore.kernel.org&#47;linux-mm&#47;<br>然后搜索相应的subject，或者author。<br>2）lkml上可能也会有（抄送给linux-kernel的邮件）<br>https:&#47;&#47;lkml.org&#47;<br>3）订阅邮件列表<br>http:&#47;&#47;vger.kernel.org&#47;vger-lists.html<br>4）通过git blame来查看文件历史修改记录<br>看看这些patch被commit之前的一些讨论还是很有帮助的，希望对你能有帮助。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-27 16:22:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/8f/cf/890f82d6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>那时刻</span>
  </div>
  <div class="_2_QraFYR_0">老师的思考题，我的想法是这个in_memstall 的位域，是每个线程独享（排它）一个位域么？这样每个线程可以独立操作其所属位域，不存在竞争了。不知理解是否正确？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个位域其其他线程也能看到，通过task-&gt;in_memstall.，但是其他线程是只读，而只有current才会写，也就是同时只会有一个线程写，所以不会存在竞争。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-27 10:02:34</div>
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
  <div class="_2_QraFYR_0">请教一下邵老师：经常看到CPU高，而IPC不高，说是性能问题，这种是不是跟cache miss有关呢?另外使用perf之类的需要对PMU这些有所了解，请问有这块的资料推荐吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-17 21:48:41</div>
  </div>
</div>
</div>
</li>
</ul>