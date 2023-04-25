<audio title="10 _ Swap：容器可以使用Swap空间吗？" src="https://static001.geekbang.org/resource/audio/17/92/17878f4c07c7b04c1f30473d2a8ba492.mp3" controls="controls"></audio> 
<p>你好，我是程远。这一讲，我们来看看容器中是否可以使用Swap空间。</p><p>用过Linux的同学应该都很熟悉Swap空间了，简单来说它就是就是一块磁盘空间。</p><p>当内存写满的时候，就可以把内存中不常用的数据暂时写到这个Swap空间上。这样一来，内存空间就可以释放出来，用来满足新的内存申请的需求。</p><p>它的好处是可以<strong>应对一些瞬时突发的内存增大需求</strong>，不至于因为内存一时不够而触发OOM Killer，导致进程被杀死。</p><p>那么对于一个容器，特别是容器被设置了Memory Cgroup之后，它还可以使用Swap空间吗？会不会出现什么问题呢？</p><h2>问题再现</h2><p>接下来，我们就结合一个小例子，一起来看看吧。</p><p>首先，我们在一个有Swap空间的节点上启动一个容器，设置好它的Memory Cgroup的限制，一起来看看接下来会发生什么。</p><p>如果你的节点上没有Swap分区，也没有关系，你可以用下面的<a href="https://github.com/chengyli/training/blob/main/memory/swap/create_swap.sh">这组命令</a>来新建一个。</p><p>这个例子里，Swap空间的大小是20G，你可以根据自己磁盘空闲空间来决定这个Swap的大小。执行完这组命令之后，我们来运行free命令，就可以看到Swap空间有20G。</p><p>输出的结果你可以参考下面的截图。</p><p><img src="https://static001.geekbang.org/resource/image/33/5b/337a5efa84fc64f5a7ab2b12295e8b5b.png?wh=1622*496" alt=""></p><p>然后我们再启动一个容器，和OOM那一讲里的<a href="https://github.com/chengyli/training/blob/main/memory/oom/start_container.sh">例子</a>差不多，容器的Memory Cgroup限制为512MB，容器中的mem_alloc程序去申请2GB内存。</p><!-- [[[read_end]]] --><p>你会发现，这次和上次OOM那一讲里的情况不一样了，并没有发生OOM导致容器退出的情况，容器运行得好好的。</p><p>从下面的图中，我们可以看到，mem_alloc进程的RSS内存一直在512MB（RES: 515596）左右。</p><p><img src="https://static001.geekbang.org/resource/image/f3/dd/f3be95c49af5bed1965654dd79db7bdd.png?wh=1584*196" alt=""><br>
那我们再看一下Swap空间，使用了1.5GB (used 1542144KB)。输出的结果如下图，简单计算一下，1.5GB + 512MB，结果正好是mem_alloc这个程序申请的2GB内存。</p><p><img src="https://static001.geekbang.org/resource/image/e9/3f/e922df98666ab06e80f816d81e11883f.png?wh=1658*198" alt=""></p><p>通过刚刚的例子，你也许会这么想，因为有了Swap空间，本来会被OOM Kill的容器，可以好好地运行了。初看这样似乎也挺好的，不过你仔细想想，这样一来，Memory Cgroup对内存的限制不就失去了作用么？</p><p>我们再进一步分析，如果一个容器中的程序发生了内存泄漏（Memory leak），那么本来Memory Cgroup可以及时杀死这个进程，让它不影响整个节点中的其他应用程序。结果现在这个内存泄漏的进程没被杀死，还会不断地读写Swap磁盘，反而影响了整个节点的性能。</p><p>你看，这样一分析，对于运行容器的节点，你是不是又觉得应该禁止使用Swap了呢?</p><p>我想提醒你，不能一刀切地下结论，我们总是说，具体情况要具体分析，我们落地到具体的场景里，就会发现情况又没有原先我们想得那么简单。</p><p>比如说，某一类程序就是需要Swap空间，才能防止因为偶尔的内存突然增加而被OOM Killer杀死。因为这类程序重新启动的初始化时间会很长，这样程序重启的代价就很大了，也就是说，打开Swap对这类程序是有意义的。</p><p>这一类程序一旦放到容器中运行，就意味着它会和“别的容器”在同一个宿主机上共同运行，那如果这个“别的容器” 如果不需要Swap，而是希望Memory Cgroup的严格内存限制。</p><p>这样一来，在这一个宿主机上的两个容器就会有冲突了，我们应该怎么解决这个问题呢？要解决这个问题，我们先来看看Linux里的Swappiness这个概念，后面它可以帮到我们。</p><h2>如何正确理解swappiness参数？</h2><p>在普通Linux系统上，如果你使用过Swap空间，那么你可能配置过proc文件系统下的swappiness 这个参数 (/proc/sys/vm/swappiness)。swappiness的定义在<a href="https://www.kernel.org/doc/Documentation/sysctl/vm.txt">Linux 内核文档</a>中可以找到，就是下面这段话。</p><blockquote>
<p>swappiness</p>
</blockquote><blockquote>
<p>This control is used to define how aggressive the kernel will swap memory pages.  Higher values will increase aggressiveness, lower values decrease the amount of swap.  A value of 0 instructs the kernel not to initiate swap until the amount of free and file-backed pages is less than the high water mark in a zone.</p>
</blockquote><blockquote>
<p>The default value is 60.</p>
</blockquote><p>前面两句话大致翻译过来，意思就是 <strong>swappiness可以决定系统将会有多频繁地使用交换分区。</strong></p><p>一个较高的值会使得内核更频繁地使用交换分区，而一个较低的取值，则代表着内核会尽量避免使用交换分区。swappiness的取值范围是0–100，缺省值60。</p><p>我第一次读到这个定义，再知道了这个取值范围后，我觉得这是一个百分比值，也就是定义了使用Swap空间的频率。</p><p>当这个值是100的时候，哪怕还有空闲内存，也会去做内存交换，尽量把内存数据写入到Swap空间里；值是0的时候，基本上就不做内存交换了，也就不写Swap空间了。</p><p>后来再回顾的时候，我发现这个想法不能说是完全错的，但是想得简单了些。那这段swappiness的定义，应该怎么正确地理解呢？</p><p>你还记得，我们在上一讲里说过的两种内存类型Page Cache 和RSS么?</p><p>在有磁盘文件访问的时候，Linux会尽量把系统的空闲内存用作Page Cache来提高文件的读写性能。在没有打开Swap空间的情况下，一旦内存不够，这种情况下就只能把Page Cache释放了，而RSS内存是不能释放的。</p><p>在RSS里的内存，大部分都是没有对应磁盘文件的内存，比如用malloc()申请得到的内存，这种内存也被称为<strong>匿名内存（Anonymous memory）</strong>。那么当Swap空间打开后，可以写入Swap空间的，就是这些匿名内存。</p><p>所以在Swap空间打开的时候，问题也就来了，在内存紧张的时候，Linux系统怎么决定是先释放Page Cache，还是先把匿名内存释放并写入到Swap空间里呢？</p><p>我们一起来分析分析，都可能发生怎样的情况。最可能发生的是下面两种情况：</p><p>第一种情况是，如果系统先把Page Cache都释放了，那么一旦节点里有频繁的文件读写操作，系统的性能就会下降。</p><p>还有另一种情况，如果Linux系统先把匿名内存都释放并写入到Swap，那么一旦这些被释放的匿名内存马上需要使用，又需要从Swap空间读回到内存中，这样又会让Swap（其实也是磁盘）的读写频繁，导致系统性能下降。</p><p>显然，我们在释放内存的时候，需要平衡Page Cache的释放和匿名内存的释放，而swappiness，就是用来定义这个平衡的参数。</p><p>那么swappiness具体是怎么来控制这个平衡的？我们看一下在Linux内核代码里是怎么用这个swappiness参数。</p><p>我们前面说了swappiness的这个值的范围是0到100，但是请你一定要注意，<strong>它不是一个百分比，更像是一个权重</strong>。它是用来定义Page Cache内存和匿名内存的释放的一个比例。</p><p>我结合下面的这段代码具体给你讲一讲。</p><p>我们可以看到，这个比例是anon_prio: file_prio，这里anon_prio的值就等于swappiness。下面我们分三个情况做讨论：</p><p>第一种情况，当swappiness的值是100的时候，匿名内存和Page Cache内存的释放比例就是100: 100，也就是等比例释放了。</p><p>第二种情况，就是swappiness缺省值是60的时候，匿名内存和Page Cache内存的释放比例就是60 : 140，Page Cache内存的释放要优先于匿名内存。</p><pre><code class="language-shell">        /*
         * With swappiness at 100, anonymous and file have the same priority.
         * This scanning priority is essentially the inverse of IO cost.
         */

        anon_prio = swappiness;
        file_prio = 200 - anon_prio;
</code></pre><p>还有一种情况， 当swappiness的值是0的时候，会发生什么呢？这种情况下，Linux系统是不允许匿名内存写入Swap空间了吗？</p><p>我们可以回到前面，再看一下那段swappiness的英文定义，里面特别强调了swappiness为0的情况。</p><p>当空闲内存少于内存一个zone的"high water mark"中的值的时候，Linux还是会做内存交换，也就是把匿名内存写入到Swap空间后释放内存。</p><p>在这里zone是Linux划分物理内存的一个区域，里面有3个水位线（water mark），水位线可以用来警示空闲内存的紧张程度。</p><p>这里我们可以再做个试验来验证一下，先运行 <code>echo 0 &gt; /proc/sys/vm/swappiness</code> 命令把swappiness设置为0， 然后用我们之前例子里的mem_alloc程序来申请内存。</p><p>比如我们的这个节点上内存有12GB，同时有2GB的Swap，用mem_alloc申请12GB的内存，我们可以看到Swap空间在mem_alloc调用之前，used=0，输出结果如下图所示。</p><p><img src="https://static001.geekbang.org/resource/image/35/50/35b806bd26e089506d909d31f1e87550.png?wh=1496*340" alt=""></p><p>接下来，调用mem_alloc之后，Swap空间就被使用了。</p><p><img src="https://static001.geekbang.org/resource/image/e2/ec/e245d137131b1yyc1e169a24fd10b5ec.png?wh=1618*200" alt=""></p><p>因为mem_alloc申请12GB内存已经和节点最大内存差不多了，我们如果查看 <code>cat /proc/zoneinfo</code> ，也可以看到normal zone里high （water mark）的值和free的值差不多，这样在free&lt;high的时候，系统就会回收匿名内存页面并写入Swap空间。</p><p><img src="https://static001.geekbang.org/resource/image/21/1d/212f4dde0982f656610cd8a3b293051d.png?wh=630*240" alt=""><br>
好了，在这里我们介绍了Linux系统里swappiness的概念，它是用来决定在内存紧张时候，回收匿名内存和Page Cache内存的比例。</p><p><strong>swappiness的取值范围在0到100，值为100的时候系统平等回收匿名内存和Page Cache内存；一般缺省值为60，就是优先回收Page Cache；即使swappiness为0，也不能完全禁止Swap分区的使用，就是说在内存紧张的时候，也会使用Swap来回收匿名内存。</strong></p><h2>解决问题</h2><p>那么运行了容器，使用了Memory Cgroup之后，swappiness怎么工作呢？</p><p>如果你查看一下Memory Cgroup控制组下面的参数，你会看到有一个memory.swappiness参数。这个参数是干啥的呢？</p><p>memory.swappiness可以控制这个Memroy Cgroup控制组下面匿名内存和page cache的回收，取值的范围和工作方式和全局的swappiness差不多。这里有一个优先顺序，在Memory Cgorup的控制组里，如果你设置了memory.swappiness参数，它就会覆盖全局的swappiness，让全局的swappiness在这个控制组里不起作用。</p><p>不过，这里有一点不同，需要你留意：<strong>当memory.swappiness = 0的时候，对匿名页的回收是始终禁止的，也就是始终都不会使用Swap空间。</strong></p><p>这时Linux系统不会再去比较free内存和zone里的high water mark的值，再决定一个Memory Cgroup中的匿名内存要不要回收了。</p><p>请你注意，当我们设置了"memory.swappiness=0时，在Memory Cgroup中的进程，就不会再使用Swap空间，知道这一点很重要。</p><p>我们可以跑个容器试一试，还是在一个有Swap空间的节点上运行，运行和这一讲开始一样的容器，唯一不同的是把容器对应Memory Cgroup里的memory.swappiness设置为0。</p><p><img src="https://static001.geekbang.org/resource/image/46/a7/46a1a06abfa2f817570c8cyy5faa62a7.png?wh=1920*495" alt=""></p><p>这次我们在容器中申请内存之后，Swap空间就没有被使用了，而当容器申请的内存超过memory.limit_in_bytes之后，就发生了OOM Kill。</p><p>好了，有了"memory.swappiness = 0"的配置和功能，就可以解决我们在这一讲里最开始提出的问题了。</p><p>在同一个宿主机上，假设同时存在容器A和其他容器，容器A上运行着需要使用Swap空间的应用，而别的容器不需要使用Swap空间。</p><p>那么，我们还是可以在宿主机节点上打开Swap空间，同时在其他容器对应的Memory Cgroups控制组里，把memory.swappiness这个参数设置为0。这样一来，我们不但满足了容器A的需求，而且别的容器也不会受到影响，仍然可以严格按照Memory Cgroups里的memory.limit_in_bytes来限制内存的使用。</p><p>总之，memory.swappiness这个参数很有用，通过它可以让需要使用Swap空间的容器和不需要Swap的容器，同时运行在同一个宿主机上。</p><h2>重点总结</h2><p>这一讲，我们主要讨论的问题是在容器中是否可以使用Swap？</p><p>这个问题没有看起来那么简单。当然了，只要在宿主机节点上打开Swap空间，在容器中就是可以用到Swap的。但出现的问题是在同一个宿主机上，对于不需要使用swap的容器， 它的Memory Cgroups的限制也失去了作用。</p><p>针对这个问题，我们学习了Linux中的swappiness这个参数。swappiness参数值的作用是，在系统里有Swap空间之后，当系统需要回收内存的时候，是优先释放Page Cache中的内存，还是优先释放匿名内存（也就是写入Swap）。</p><p>swappiness的取值范围在0到100之间，我们可以记住下面三个值：</p><ul>
<li>值为100的时候， 释放Page Cache和匿名内存是同等优先级的。</li>
<li>值为60，这是大多数Linux系统的缺省值，这时候Page Cache的释放优先级高于匿名内存的释放。</li>
<li>值为0的时候，当系统中空闲内存低于一个临界值的时候，仍然会释放匿名内存并把页面写入Swap空间。</li>
</ul><p><img src="https://static001.geekbang.org/resource/image/6a/11/6aa89d2b88493ddeb7d37ab9db275811.jpeg?wh=3200*1800" alt=""></p><p>swappiness参数除了在proc文件系统下有个全局的值外，在每个Memory Cgroup控制组里也有一个memory.swappiness，那它们有什么不同呢？</p><p>不同就是每个Memory Cgroup控制组里的swappiness参数值为0的时候，就可以让控制组里的内存停止写入Swap。这样一来，有了memory.swappiness这个参数后，需要使用Swap和不需要Swap的容器就可以在同一个宿主机上同时运行了，这样对于硬件资源的利用率也就更高了。</p><h2>思考题</h2><p>在一个有Swap分区的节点上用Docker启动一个容器，对它的Memory Cgroup控制组设置一个内存上限N，并且将memory.swappiness设置为0。这时，如果在容器中启动一个不断读写文件的程序，同时这个程序再申请1/2N的内存，请你判断一下，Swap分区中会有数据写入吗？</p><p>欢迎在留言区分享你的收获和疑问。如果这篇文章让你有所收获，也欢迎分享给你的朋友，一起交流和学习。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/ff/46/7e4039ea.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>伟平</span>
  </div>
  <div class="_2_QraFYR_0">k8s下容器貌似不能用swap</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: kubelet缺省不能在打开swap的节点上运行。<br>配置“failSwapOn: false”参数，kubelet可以在swap enabled的节点上运行。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-07 20:38:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/7e/99/c4302030.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Khirye</span>
  </div>
  <div class="_2_QraFYR_0"> 想问下老师，对于最近k8s宣布放弃docker有什么看法？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对于k8s来说，的确没有必要再用docker了。<br>这是我们组之前做的从docker切换到containerd的技术分享:<br>https:&#47;&#47;www.infoq.cn&#47;article&#47;odslclsjvo8bnx*mbrbk</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-11 10:27:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/5e/96/a03175bc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>莫名</span>
  </div>
  <div class="_2_QraFYR_0">Memory Cgroup 参数 memory.swappiness 起到局部控制的作用，前提是节点开启了 swap 功能，不然无论如何设置都无济于事。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: @莫名, 没错</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-07 15:08:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1c/f4/c7/037235c9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>kimoti</span>
  </div>
  <div class="_2_QraFYR_0">既然memory.swappiness设置为0了,Swap分区是不会有数据写入的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: @kimoti <br>是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-07 10:34:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/f5/97/9a7ee7b3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek4329</span>
  </div>
  <div class="_2_QraFYR_0">感谢老师的分享，学习的越多，越感到自己的自信😼<br><br>学完这个课程，并完全吸收的话，是不是可以说自己接近精通容器技术了😁</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: @Geek4329<br>很高兴我的分享对你有帮助！<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-07 17:29:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/27/8e/3e/bdfb6dc3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>刘宏</span>
  </div>
  <div class="_2_QraFYR_0">请问一下在容器里如何加快page cache的释放速度，原因是发现scp的时候，page cache会很快将内存占满，速度会骤然下降，drop cache以后，速度能显著提升；配置了vm.vfs_cache_pressure为1w，依旧在容器里没改善</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-03 19:10:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/b0/c8/cae61286.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>chong chong</span>
  </div>
  <div class="_2_QraFYR_0">老师，k8s pod默认的memory.swappiness 值是60，如何设置才能使得默认是0呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以修改kube代码</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-11 00:21:21</div>
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
  <div class="_2_QraFYR_0">老师，文章中你说的这两句话好像是矛盾的，swappiness设置为0的时候，到底会不会回收匿名内存呢？<br>1. 不过，这里有一点不同，需要你留意：当 memory.swappiness = 0 的时候，对匿名页的回收是始终禁止的，也就是始终都不会使用 Swap 空间。这时 Linux 系统不会再去比较 free 内存和 zone 里的 high water mark 的值，再决定一个 Memory Cgroup 中的匿名内存要不要回收了。请你注意，当我们设置了&quot;memory.swappiness=0 时，在 Memory Cgroup 中的进程，就不会再使用 Swap 空间，知道这一点很重要.<br><br>2. 值为 0 的时候，当系统中空闲内存低于一个临界值的时候，仍然会释放匿名内存并把页面写入 Swap 空间。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. memory.swappiness 是指cgroup中的参数<br>2. 这里说的是&#47;proc&#47;sys&#47;vm&#47;swappiness</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-15 14:31:52</div>
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
  <div class="_2_QraFYR_0">所以对于开启了swap，且swap空间够大的前提下，goup的mem limit没用咯?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的， 如果只是设置 memory.limit_in_bytes</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-15 04:03:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/23/81/60/71ed6ac7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>谢哈哈</span>
  </div>
  <div class="_2_QraFYR_0">已经设置memory.swappiness参数，全局参数swappiness参数失效，那么容器里就不能使用swap了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: @谢哈哈<br>是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-07 19:41:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/56/a4/9d4089e8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Yann彦</span>
  </div>
  <div class="_2_QraFYR_0">k8s集群中是否建议打开swap呢？打开过关闭有什么优缺点呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-21 17:21:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/8d/6d/8c0a487b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>册书一幕</span>
  </div>
  <div class="_2_QraFYR_0">“不过，这里有一点不同，需要你留意：当 memory.swappiness = 0 的时候，对匿名页的回收是始终禁止的，也就是始终都不会使用 Swap 空间。”<br>这句话是指的仅在cgroup情况下吗？在主机下，memory.swappiness = 0是不会分配swap但是会回收</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-04 18:32:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/8d/6d/8c0a487b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>册书一幕</span>
  </div>
  <div class="_2_QraFYR_0">不过，这里有一点不同，需要你留意：当 memory.swappiness = 0 的时候，对匿名页的回收是始终禁止的，也就是始终都不会使用 Swap 空间。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-04 18:31:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/a6/69/cf2eb6be.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>景b</span>
  </div>
  <div class="_2_QraFYR_0">如果有分布式存储的能力且能做到按卷级别做限速，每个容器都是单独挂一个卷进去，是不是开swap就利大于弊。可以这么理解不</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-07 14:21:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/2a/ff/a9d72102.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>BertGeek</span>
  </div>
  <div class="_2_QraFYR_0">1、swappiness（&#47;proc&#47;sys&#47;vm&#47;swappiness） 的取值范围在 0 到 100 之间，我们可以记住下面三个值：<br>1.1 值为 100 的时候，释放 Page Cache 和匿名内存是同等优先级的。<br>1.2 值为 60，这是大多数 Linux 系统的缺省值，这时候 Page Cache 的释放优先级高于匿名内存的释放。<br>1.3 值为0，也不能完全禁止 Swap 分区的使用，就是当系统中空闲内存低于一个临界值的时候，仍然会通过swap回收匿名内存（释放匿名内存并把页面写入 Swap 空间）。<br><br>2、Memroy Cgroup 控制组中的memory.swappiness参数<br>    如果设置memory.swappiness参数，此容器中的全局swappiness参数失效，那么此容器里就不能使用swap了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-05 16:36:16</div>
  </div>
</div>
</div>
</li>
</ul>