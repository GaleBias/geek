<audio title="01 基础篇 _ 如何用数据观测Page Cache？" src="https://static001.geekbang.org/resource/audio/f9/84/f98yy95aff1459863b134b975bced684.mp3" controls="controls"></audio> 
<p>你好，我是邵亚方。今天我想和你聊一聊Page Cache的话题。</p><p>Page Cache你应该不陌生了，如果你是一名应用开发者或者Linux运维人员，那么在工作中，你可能遇见过与Page Cache有关的场景，比如：</p><ul>
<li>服务器的load飙高；</li>
<li>服务器的I/O吞吐飙高；</li>
<li>业务响应时延出现大的毛刺；</li>
<li>业务平均访问时延明显增加。</li>
</ul><p>这些问题，很可能是由于Page Cache管理不到位引起的，因为Page Cache管理不当除了会增加系统I/O吞吐外，还会引起业务性能抖动，我在生产环境上处理过很多这类问题。</p><p>据我观察，这类问题出现后，业务开发人员以及运维人员往往会束手无策，究其原因在于他们对Page Cache的理解仅仅停留在概念上，并不清楚Page Cache如何和应用、系统关联起来，对它引发的问题自然会束手无策了。所以，要想不再踩Page Cache的坑，你必须对它有个清晰的认识。</p><p>那么在我看来，认识Page Cache最简单的方式，就是用数据说话，通过具体的数据你会更加深入地理解Page Cache的本质。为了帮你消化和理解，我会用两节课的时间，用数据剖析什么是Page Cache，为什么需要Page Cache，Page Cache的产生和回收是什么样的。这样一来，你会从本质到表象，透彻理解它，深切感受它和你的应用程序之间的关系，从而能更好地理解上面提到的四个问题。</p><!-- [[[read_end]]] --><p>不过，在这里我想给你提个醒，要学习今天的内容，你最好具备一些Linux编程的基础，比如，如何打开一个文件；如何读写一个文件；如何关闭一个文件等等。这样，你理解今天的内容会更加容易，当然了，不具备也没有关系，如果遇到你实在看不懂的地方，你可以查阅《UNIX环境高级编程》这本书，它是每一位Linux开发者以及运维人员必看的入门书籍。</p><p>好了，话不多说，我们进入今天的学习。</p><h2>什么是Page Cache？</h2><p>我记得很多应用开发者或者运维在向我寻求帮助，解决Page Cache引起的问题时，总是喜欢问我Page Cache到底是属于内核还是属于用户？针对这样的问题，我一般会让他们先看下面这张图：</p><p><img src="https://static001.geekbang.org/resource/image/f3/1b/f344917f3cacd5bc06ae7c743a217f1b.png?wh=2860*2440" alt="" title="应用程序产生Page Cache的逻辑示意图 "></p><p>通过这张图片你可以清楚地看到，红色的地方就是Page Cache，<strong>很明显，Page Cache是内核管理的内存，也就是说，它属于内核不属于用户。</strong></p><p>那咱们怎么来观察Page Cache呢？其实，在Linux上直接查看Page Cache的方式有很多，包括/proc/meminfo、free 、/proc/vmstat命令等，它们的内容其实是一致的。</p><p>我们拿/proc/meminfo命令举例看一下（如果你想了解/proc/meminfo中每一项具体含义的话，可以去看<a href="https://www.kernel.org/doc/Documentation/filesystems/proc.rst">Kernel Documentation</a>的meminfo这一节，它详细解释了每一项的具体含义，Kernel Documentation是应用开发者想要了解内核最简单、直接的方式）。</p><pre><code>$ cat /proc/meminfo
...
Buffers:            1224 kB
Cached:           111472 kB
SwapCached:        36364 kB
Active:          6224232 kB
Inactive:         979432 kB
Active(anon):    6173036 kB
Inactive(anon):   927932 kB
Active(file):      51196 kB
Inactive(file):    51500 kB
...
Shmem:             10000 kB
...
SReclaimable:      43532 kB
...
</code></pre><p>根据上面的数据，你可以简单得出这样的公式（等式两边之和都是112696 KB）：</p><blockquote>
<p>Buffers + Cached + SwapCached = Active(file) + Inactive(file) + Shmem + SwapCached</p>
</blockquote><p><strong>那么等式两边的内容就是我们平时说的Page Cache。</strong>请注意你没有看错，两边都有SwapCached，之所以要把它放在等式里，就是说它也是Page Cache的一部分。</p><p>接下来，我带你分析一下这些项的具体含义。等式右边这些项把Buffers和Cached做了一下细分，分为了Active(file)，Inactive(file) 和Shmem，因为Buffers更加依赖于内核实现，在不同内核版本中它的含义可能有些不一致，而等式右边和应用程序的关系更加直接，所以我们从等式右边来分析。</p><p>在Page Cache中，Active(file)+Inactive(file)是File-backed page（与文件对应的内存页），是你最需要关注的部分。因为你平时用的mmap()内存映射方式和buffered I/O来消耗的内存就属于这部分，<strong>最重要的是，这部分在真实的生产环境上也最容易产生问题，</strong>我们在接下来的课程案例篇会重点分析它。</p><p>而SwapCached是在打开了Swap分区后，把Inactive(anon)+Active(anon)这两项里的匿名页给交换到磁盘（swap out），然后再读入到内存（swap in）后分配的内存。<strong>由于读入到内存后原来的Swap File还在，所以SwapCached也可以认为是File-backed page，即属于Page Cache。</strong>这样做的目的也是为了减少I/O。你是不是觉得这个过程有些复杂？我们用一张图直观地看一下：</p><p><img src="https://static001.geekbang.org/resource/image/a9/7b/a9010acc1d7b55d5c24562d2c32b437b.jpg?wh=4336*1491" alt=""></p><p>我希望你能通过这个简单的示意图明白SwapCached是怎么产生的。在这个过程中你要注意，SwapCached只在Swap分区打开的情况下才会有，而我建议你在生产环境中关闭Swap分区，因为Swap过程产生的I/O会很容易引起性能抖动。</p><p>除了SwapCached，Page Cache中的Shmem是指匿名共享映射这种方式分配的内存（free命令中shared这一项），比如tmpfs（临时文件系统），这部分在真实的生产环境中产生的问题比较少，不是我们今天的重点内容，我们这节课不对它做过多关注，你知道有这回事就可以了。</p><p>当然了，很多同学也喜欢用free命令来查看系统中有多少Page Cache，会根据buff/cache来判断存在多少Page Cache。如果你对free命令有所了解的话，肯定知道free命令也是通过解析/proc/meminfo得出这些统计数据的，这些都可以通过free工具的源码来找到。free命令的源码是开源，你可以去看下<a href="https://gitlab.com/procps-ng/procps">procfs</a>里的free.c文件，源码是最直接的理解方式，它会加深你对free命令的理解。</p><p>不过你是否好奇过，free命令中的buff/cache究竟是指什么呢？我们在这里先简单地看一下：</p><pre><code>$ free -k
              total        used        free      shared  buff/cache   available
Mem:        7926580     7277960      492392       10000      156228      430680
Swap:       8224764      380748     7844016
</code></pre><p>通过procfs源码里面的<a href="https://gitlab.com/procps-ng/procps/-/blob/master/proc/sysinfo.c">proc/sysinfo.c</a>这个文件，你可以发现buff/cache包括下面这几项：</p><blockquote>
<p>buff/cache = Buffers + Cached + SReclaimable</p>
</blockquote><p>通过前面的数据我们也可以验证这个公式: 1224 + 111472 + 43532的和是156228。</p><p>另外，这里你要注意，你在做比较的过程中，一定要考虑到这些数据是动态变化的，而且执行命令本身也会带来内存开销，所以这个等式未必会严格相等，不过你不必怀疑它的正确性。</p><p>从这个公式中，你能看到free命令中的buff/cache是由Buffers、Cached和SReclaimable这三项组成的，它强调的是内存的可回收性，也就是说，可以被回收的内存会统计在这一项。</p><p>其中SReclaimable是指可以被回收的内核内存，包括dentry和inode等。而这部分内容是内核非常细节性的东西，对于应用开发者和运维人员理解起来相对有些难度，所以我们在这里不多说。</p><p>掌握了Page Cache具体由哪些部分构成之后，在它引发一些问题时，你就能够知道需要去观察什么。比如说，应用本身消耗内存（RSS）不多的情况下，整个系统的内存使用率还是很高，那不妨去排查下是不是Shmem(共享内存)消耗了太多内存导致的。</p><p>讲到这儿，我想你应该对Page Cache有了一些直观的认识了吧？当然了，有的人可能会说，内核的Page Cache这么复杂，我不要不可以么？</p><p>我相信有这样想法的人不在少数，如果不用内核管理的Page Cache，那有两种思路来进行处理：</p><ul>
<li>第一种，应用程序维护自己的Cache做更加细粒度的控制，比如MySQL就是这样做的，你可以参考<a href="https://dev.mysql.com/doc/refman/8.0/en/innodb-buffer-pool.html">MySQL Buffer Pool</a> ，它的实现复杂度还是很高的。对于大多数应用而言，实现自己的Cache成本还是挺高的，不如内核的Page Cache来得简单高效。</li>
<li>第二种，直接使用Direct I/O来绕过Page Cache，不使用Cache了，省的去管它了。这种方法可行么？那我们继续用数据说话，看看这种做法的问题在哪儿？</li>
</ul><h2>为什么需要Page Cache？</h2><p>通过第一张图你其实已经可以直观地看到，标准I/O和内存映射会先把数据写入到Page Cache，这样做会通过减少I/O次数来提升读写效率。我们看一个具体的例子。首先，我们来生成一个1G大小的新文件，然后把Page Cache清空，确保文件内容不在内存中，以此来比较第一次读文件和第二次读文件耗时的差异。具体的流程如下。</p><p>先生成一个1G的文件：</p><blockquote>
<p>$ dd if=/dev/zero of=/home/yafang/test/dd.out bs=4096 count=$((1024*256))</p>
</blockquote><p>其次，清空Page Cache，需要先执行一下sync来将脏页（第二节课，我会解释一下什么是脏页）同步到磁盘再去drop cache。</p><blockquote>
<p>$ sync &amp;&amp; echo 3 &gt; /proc/sys/vm/drop_caches</p>
</blockquote><p>第一次读取文件的耗时如下：</p><pre><code>$ time cat /home/yafang/test/dd.out &amp;&gt; /dev/null
real	0m5.733s
user	0m0.003s
sys	0m0.213s
</code></pre><p>再次读取文件的耗时如下：</p><pre><code>$ time cat /home/yafang/test/dd.out &amp;&gt; /dev/null 
real	0m0.132s
user	0m0.001s
sys	0m0.130s
</code></pre><p>通过这样详细的过程你可以看到，第二次读取文件的耗时远小于第一次的耗时，这是因为第一次是从磁盘来读取的内容，磁盘I/O是比较耗时的，而第二次读取的时候由于文件内容已经在第一次读取时被读到内存了，所以是直接从内存读取的数据，内存相比磁盘速度是快很多的。<strong>这就是Page Cache存在的意义：减少I/O，提升应用的I/O速度。</strong></p><p>所以，如果你不想为了很细致地管理内存而增加应用程序的复杂度，那你还是乖乖使用内核管理的Page Cache吧，它是ROI(投入产出比)相对较高的一个方案。</p><p>你要知道，我们在做方案抉择时找到一个各方面都很完美的方案还是比较难的，大多数情况下都是经过权衡后来选择一个合适的方案。因为，我一直坚信，合适的就是最好的。</p><p>而我之所以说Page Cache是合适的，而不是说它是最好的，那是因为Page Cache的不足之处也是有的，这个不足之处主要体现在，它对应用程序太过于透明，以至于应用程序很难有好方法来控制它。</p><p>为什么这么说呢？要想知道这个答案，你就需要了解Page Cache的产生过程，这里卖个关子，我在下一讲会跟你讨论。</p><h2>课堂总结</h2><p>我们这节课主要是讲述了如何很好地理解Page Cache，在我看来，要想很好的理解它，直观的方式就是从数据入手，所以，我从如何观测Page Cache出发来带你认识什么是Page Cache；然后再从它为什么容易产生问题出发，带你回顾了它存在的意义，我希望通过这样的方式，帮你明确这样几个要点：</p><ol>
<li>Page Cache是属于内核的，不属于用户。</li>
<li>Page Cache对应用提升I/O效率而言是一个投入产出比较高的方案，所以它的存在还是有必要的。</li>
</ol><p>在我看来，如何管理好Page Cache，最主要的是你要知道如何来观测它以及观测关于它的一些行为，有了这些数据做支撑，你才能够把它和你的业务更好地结合起来。而且，在我看来，当你对某一个概念很模糊、搞不清楚它到底是什么时，最好的认知方式就是先搞明白如何来观测它，然后动手去观测看看它究竟是如何变化的，正所谓纸上得来终觉浅，绝知此事要躬行！</p><p>这节课就讲到这里，下一节我们使用数据来观察Page Cache的产生和释放，这样一来，你就能了解Page Cache的整个生命周期，从而对于它引发的一些问题能有一个大概的判断。</p><h2>课后作业</h2><p>最后我给你留一道思考题，请你写一个程序来构造出来Page Cache，然后观察/proc/meminfo和/proc/vmstat里面的数据是如何变化的， 欢迎在留言区分享你的看法。</p><p>感谢你的阅读，如果你认为这节课的内容有收获，也欢迎把它分享给你的朋友，我们下一讲见。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/34/df/64e3d533.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>leslie</span>
  </div>
  <div class="_2_QraFYR_0">Coding能力有限：不过学过倪鹏飞老师的课，侧重点应当不同吧；PageCache大小的合理设置不知道为何老师不曾提及？单独的去谈代码似乎忘了底层硬件吧。我们说性能是基于硬件去提问，老师单独的去问而不提及硬件基础是否、、、这就像我们去说怎么样去设置软件更合理，不去提及我们硬件的主频和大小以及、、、</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: pagecache和硬件关系不大，如果说pagecache一定要跟硬件关联起来的话，会有这几方面：一个page有多大，tlb有多大，cache有多大，物理内存多大，磁盘是ssd，hdd还是nvme。page的大小和物理内存的大小决定了pagecache的量，但是你会发现pagecache的回收机制其实是独立于这个量的，也就是，不论它有多少，active和inactive的替换逻辑是不变的。tlb大小和cache大小则影响了pagecache命中与否的性能，但是你会发现相比pagecache miss而言，tlb&#47;cache miss几乎可以忽略不计，这是不同数量级的差异。磁盘介质又决定了pagecache的重要性，如果你的磁盘是hdd，那pagecache对应用的提升会非常明显；如果是nvme，那么pagecache miss后业务的性能下降也不会太明显；但是你会发现，不论磁盘介质是什么，pagecache如果命中的话，它的性能都是数量级的差异。<br>也就是说，pagecache自身的机制是独立与硬件的，不论你是什么样的硬件，都要尽量保障热数据在内存中，而冷数据则尽量不要占用内存，这就是pagecache机制的核心。<br>换一个角度而言，不论是x86还是arm，linux都要能工作，而能工作就是linux的本质，这个本质是我们想要讨论的。而不是讨论linux在arm上的工作和x86的工作会有什么差异。<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-31 23:38:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/20/08/bc06bc69.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Dovelol</span>
  </div>
  <div class="_2_QraFYR_0">老师好，能详细讲一下buffer cache和page cache的区别吗？这两个到底作用在哪的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: buffer可以理解为是一类特殊文件的cache，这类特殊文件就是设备文件，比如&#47;dev&#47;sda1, 这类设备文件的内容被读到内存后就是buffer。而cached则是普通文件的内容被读到了内存。你可以做一个试验，运行下面这个程序，然后观察&#47;proc&#47;meminfo里Buffers和Cached这两项的变化，你会发现增加的是前者。<br>#include &lt;sys&#47;types.h&gt;<br>#include &lt;stdio.h&gt;<br>#include &lt;stdlib.h&gt;<br>#include &lt;sys&#47;stat.h&gt;<br>#include &lt;fcntl.h&gt;<br>#include &lt;unistd.h&gt;<br><br>#define DEV &quot;&#47;dev&#47;sda1&quot;<br>#define SIZE (1024 * 1024 * 1024)<br><br>int main()<br>{<br>        int fd = open(DEV, O_RDONLY);<br>        char *buf;<br>        int size;<br><br>        buf = malloc(SIZE);<br>        if (!buf)<br>                return -1;<br><br>        size = read(fd, buf, SIZE);<br>        printf(&quot;%d\n&quot;, size);<br><br>        while (1) {<br>        }<br><br>        return 0;<br>}</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-21 00:39:35</div>
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
  <div class="_2_QraFYR_0">感觉学习到了很多。请教老师一个问题，如果不做sync而是直接去drop cache，drop期间是不是也会把脏页刷盘呢？如果是的话，那么为什么还要单独做一次sync呢？直接drop cache不就好了吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: dropcache不会清理脏页，只会清理干净的页，所以如果你想清理所有的页的话，是需要在drop前先sync的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-28 22:56:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/0f/70/cdef7a3d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Joe Black</span>
  </div>
  <div class="_2_QraFYR_0">我们的数据处理算法实际上对超大输入文件是从头到尾读一遍，而且整个程序仅读一遍，这样其实用direct io就行了吧？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的。如果只读一遍后续再也不会用到，那确实没有必要还用pagecache，使用direct IO就可以了。而且超大文件读入到内存也会影响其他进程对内存的使用。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-23 08:00:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/fb/7f/746a6f5e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Q</span>
  </div>
  <div class="_2_QraFYR_0">”从这个公式中，你能看到 free 命令中的 buff&#47;cache 是由 Buffers、Cached 和 SReclaimable 这三项组成的，它强调的是内存的可回收性，也就是说，可以被回收的内存会统计在这一项“ -- 读到这句话，感觉豁然开朗，我们知道Linux是尽可能把空闲内存作为IO的缓存来使用，提升文件读写的效率，但是这些内存是可以按实际需求会进行回收的。果然，跟大佬学习能减少自己理解的难度。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-19 16:10:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/57/4f/6fb51ff1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>一步</span>
  </div>
  <div class="_2_QraFYR_0">所以 Page Cache 就是 磁盘的一个缓存区？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以理解为是磁盘的缓存，但是它属于内存，不属于磁盘。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-23 17:49:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/f9/1a/d289c2ac.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>大飞哥</span>
  </div>
  <div class="_2_QraFYR_0">很棒，让我们比较系统的了解了Page Cache，要观测PageCache就直接读写文件就能感受得到了，特别是大文件，可以与O_DIRECT一起对比。如果调试过程发现IO这一块确实容易出问题的话，可以使用 mount -o commit 降低文件系统同步时间可以更快复现及查找问题。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-19 11:00:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/96/47/93838ff7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>青鸟飞鱼</span>
  </div>
  <div class="_2_QraFYR_0">老师，你好，在有些地方看到经过PageCache的Io叫缓存IO，有些地方说是直接IO。不知道缓存IO，直接IO有什么区别吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 缓存io是指应用在读磁盘文件的时候会先经过缓存（内存），而且直接io则不经过缓存而直接与磁盘交互。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-12 22:43:44</div>
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
  <div class="_2_QraFYR_0">老师讲的不错，深入浅出！<br>总结：<br>PageCache的计算：<br>Buffers + Cached + SwapCached = Active(file) + Inactive(file) + Shmem + SwapCached<br>①SwapCached 是在打开了 Swap 分区后，把 Inactive(anon)+Active(anon) 这两项里的匿名页给交换到磁盘（swap out），然后再读入到内存（swap in）后分配的内存。由于读入到内存后原来的 Swap File 还在，所以 SwapCached 也可以认为是 File-backed page，即属于 Page Cache。<br>②SwapCached 只在 Swap 分区打开的情况下才会有，而我建议你在生产环境中关闭 Swap 分区，因为 Swap 过程产生的 I&#47;O 会很容易引起性能抖动，所以一般在禁止swap的情况下忽略SwapCached。那free命令中的buffer&#47;cache就是pageCache。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-21 17:43:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/4Lprf2mIWpJOPibgibbFCicMtp5bpIibyLFOnyOhnBGbusrLZC0frG0FGWqdcdCkcKunKxqiaOHvXbCFE7zKJ8TmvIA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_c2089d</span>
  </div>
  <div class="_2_QraFYR_0">Swap 过程产生的 I&#47;O 会很容易引起性能抖动不太明白是什么意思？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 因为swap得数据量往往较大，而且发生swap时往往是内存很紧张时，各种因素叠加会导致swap过程非常容易出问题。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-15 16:08:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/e3/cc/0947ff0b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>nestle</span>
  </div>
  <div class="_2_QraFYR_0">老是，第一张图有个地方没太明白，Page Cache是由VFS模块来管理的吗？回写、预读这些操作都是由VFS控制的吗？还是说VFS只负责普通文件I&#47;O呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: pagecache不是vfs模块来管理的，vfs只是提供了pagecache的一个使用接口，也就是标准io这种接口。pagecache是mm这个模块来管理的，回写和预读主要是涉及到mm和io子系统。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-30 12:02:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/20/08/bc06bc69.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Dovelol</span>
  </div>
  <div class="_2_QraFYR_0">老师好，想问下你知道rocketmq消息中间件，作者用了一个堆外内存，手动commit一页4k数据来优化写pagecache，为什么是优化呢，也就是少量的写pagecache和每次写一页pagecache有啥区别呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-02 15:59:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/9c/84/4c72f91a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Rony</span>
  </div>
  <div class="_2_QraFYR_0">请问下flush和fsync操作的区别，以及那个操作是针对page cache的呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-18 22:03:21</div>
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
  <div class="_2_QraFYR_0">mmap 产生的pageCache 也是内核态的吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 内核态和用户态是指进程的运行状态 并不能用来表示内存。<br>应用程序通过mmap系统调用返回的地址空间属于“用户空间”。<br>mmap的用户空间的地址对应的物理内存则是page cache，这是不同的概念。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-21 21:38:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/09/fc/6f53d426.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>nuo-promise</span>
  </div>
  <div class="_2_QraFYR_0">老师,  mac 上可以推荐 读linux  kernel 的 ide 和 推荐 kernel 源码的版本么？跟着你的教程,然后 课后 源码文件看起来 效果还是不错的<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我在Mac上都是使用Vim+cscope来阅读内核源码，也是很方便的，最主要是这种方式在什么平台上都能用，通用性较好。<br>至于内核源码，可以紧跟upstream的源码来看，在你遇到一些疑惑时，你可以git blame来查看修改记录，修改记录里一般都会详细说明这么实现的原因。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-01 19:48:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/aa/9a/92d2df36.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>tianfeiyu</span>
  </div>
  <div class="_2_QraFYR_0">老师，想问一下，如果没有开启 Swap 分区，Inactive(anon)+Active(anon) 既然不会在 page cache，那就只会存在内存中吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果没有开启swap，匿名页就只能在内存中，知道它们被释放掉。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-08 10:08:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/K2DPibcnicQFUOxEGrnHysfvAqK8uyXCR3vqsas3xNwqvIMVuGoWRIV37Kiaia3vUlPkMRD5mDWh1OPaOBTEs86zbA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>轩轩</span>
  </div>
  <div class="_2_QraFYR_0">你好，“你可以发现 buff&#47;cache 包括下面这几项：<br>buff&#47;cache = Buffers + Cached + SReclaimable”<br>请问buffers统计里面的确是不包括SReclaimable这个的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: sreclaimable是slab的一种 主要是内核的一些数据结构对应的数据；buffers则是直接读取的设备数据。二者的含义不一样。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-24 00:34:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/6e/78/e7045b49.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>BlingBling</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，请教一下，我这边在项目的嵌入式设备上，发现<br>Buffers + Cached + SwapCached = Active(file) + Inactive(file) + Shmem + SwapCached<br>两者并不相等， Active(file) + Inactive(file) 的和远大于Buffers + Cached，请问下这是什么原因呢？或者什么原因可能导致这个现象？<br><br>另外，这个设备上执行free命令，看到的也只能看到buffers，看不到cached，这又是为什么呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Active(file)和inactive(file)会被包含buffers+cached中，你说的这个现象很诡异，你是什么内核版本？能否把meminfo里的信息都贴出来看下？<br><br>free命令这个问题，可能跟你的free版本有关，有些版本把buffers和cached给统一了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-12 20:12:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/6e/78/e7045b49.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>BlingBling</span>
  </div>
  <div class="_2_QraFYR_0">请教一下，sync &amp;&amp; echo 3 &gt; &#47;proc&#47;sys&#47;vm&#47;drop_caches命令是否并不能保证一定能将cache清除？<br>我在一台嵌入式设备上，<br>1. 多次执行cat命令读取dd创建的文件，耗时无差，并且&#47;proc&#47;memeinfo里面cached的大小也基本不变，<br>2. 执行sync &amp;&amp; echo 3 &gt; &#47;proc&#47;sys&#47;vm&#47;drop_caches命令后，立即查看&#47;proc&#47;memeinfo里面cached的大小，也和命令执行前基本一致？<br>请教下老是这是什么原因？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，并不能保障pagecache被清除，原因可能有几个：<br>1. 有些pagecache被lock了，被lock的page是无法清掉的，你可以通过meminfo来看有多lock&#47;unevictable的。<br>2. 有些pagecache还在percpu的pagevector里，还没有被放到lru上所以无法清除。<br>另外，你的内核是什么版本？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-12 19:45:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/1a/9c/082cf625.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>权</span>
  </div>
  <div class="_2_QraFYR_0">第一张图很赞，一图胜千言</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-18 08:53:45</div>
  </div>
</div>
</div>
</li>
</ul>