<audio title="19 _ 案例篇：为什么系统的Swap变高了（上）" src="https://static001.geekbang.org/resource/audio/3d/f6/3d900530991b73ec231d411d877e14f6.mp3" controls="controls"></audio> 
<p>你好，我是倪朋飞。</p><p>上一节，我通过一个斐波那契数列的案例，带你学习了内存泄漏的分析。如果在程序中直接或间接地分配了动态内存，你一定要记得释放掉它们，否则就会导致内存泄漏，严重时甚至会耗尽系统内存。</p><p>不过，反过来讲，当发生了内存泄漏时，或者运行了大内存的应用程序，导致系统的内存资源紧张时，系统又会如何应对呢？</p><p>在内存基础篇我们已经学过，这其实会导致两种可能结果，内存回收和 OOM 杀死进程。</p><p>我们先来看后一个可能结果，内存资源紧张导致的 OOM（Out Of Memory），相对容易理解，指的是系统杀死占用大量内存的进程，释放这些内存，再分配给其他更需要的进程。</p><p>这一点我们前面详细讲过，这里就不再重复了。</p><p>接下来再看第一个可能的结果，内存回收，也就是系统释放掉可以回收的内存，比如我前面讲过的缓存和缓冲区，就属于可回收内存。它们在内存管理中，通常被叫做<strong>文件页（</strong>File-backed Page）。</p><p>大部分文件页，都可以直接回收，以后有需要时，再从磁盘重新读取就可以了。而那些被应用程序修改过，并且暂时还没写入磁盘的数据（也就是脏页），就得先写入磁盘，然后才能进行内存释放。</p><p>这些脏页，一般可以通过两种方式写入磁盘。</p><ul>
<li>
<p>可以在应用程序中，通过系统调用 fsync  ，把脏页同步到磁盘中；</p>
</li>
<li>
<p>也可以交给系统，由内核线程 pdflush 负责这些脏页的刷新。</p>
</li>
</ul><!-- [[[read_end]]] --><p>除了缓存和缓冲区，通过内存映射获取的文件映射页，也是一种常见的文件页。它也可以被释放掉，下次再访问的时候，从文件重新读取。</p><p>除了文件页外，还有没有其他的内存可以回收呢？比如，应用程序动态分配的堆内存，也就是我们在内存管理中说到的<strong>匿名页</strong>（Anonymous Page），是不是也可以回收呢？</p><p>我想，你肯定会说，它们很可能还要再次被访问啊，当然不能直接回收了。非常正确，这些内存自然不能直接释放。</p><p>但是，如果这些内存在分配后很少被访问，似乎也是一种资源浪费。是不是可以把它们暂时先存在磁盘里，释放内存给其他更需要的进程？</p><p>其实，这正是Linux的Swap机制。Swap把这些不常访问的内存先写到磁盘中，然后释放这些内存，给其他更需要的进程使用。再次访问这些内存时，重新从磁盘读入内存就可以了。</p><p>在前几节的案例中，我们已经分别学过缓存和OOM的原理和分析。那Swap 又是怎么工作的呢？因为内容比较多，接下来，我将用两节课的内容，带你探索Swap的工作原理，以及Swap升高后的分析方法。</p><p>今天我们先来看看，Swap究竟是怎么工作的。</p><h2>Swap原理</h2><p>前面提到，Swap说白了就是把一块磁盘空间或者一个本地文件（以下讲解以磁盘为例），当成内存来使用。它包括换出和换入两个过程。</p><ul>
<li>
<p>所谓换出，就是把进程暂时不用的内存数据存储到磁盘中，并释放这些数据占用的内存。</p>
</li>
<li>
<p>而换入，则是在进程再次访问这些内存的时候，把它们从磁盘读到内存中来。</p>
</li>
</ul><p>所以你看，Swap其实是把系统的可用内存变大了。这样，即使服务器的内存不足，也可以运行大内存的应用程序。</p><p>还记得我最早学习Linux操作系统时，内存实在太贵了，一个普通学生根本就用不起大的内存，那会儿我就是开启了Swap来运行Linux桌面。当然，现在的内存便宜多了，服务器一般也会配置很大的内存，那是不是说Swap就没有用武之地了呢？</p><p>当然不是。事实上，内存再大，对应用程序来说，也有不够用的时候。</p><p>一个很典型的场景就是，即使内存不足时，有些应用程序也并不想被OOM杀死，而是希望能缓一段时间，等待人工介入，或者等系统自动释放其他进程的内存，再分配给它。</p><p>除此之外，我们常见的笔记本电脑的休眠和快速开机的功能，也基于Swap 。休眠时，把系统的内存存入磁盘，这样等到再次开机时，只要从磁盘中加载内存就可以。这样就省去了很多应用程序的初始化过程，加快了开机速度。</p><p>话说回来，既然Swap是为了回收内存，那么Linux到底在什么时候需要回收内存呢？前面一直在说内存资源紧张，又该怎么来衡量内存是不是紧张呢？</p><p>一个最容易想到的场景就是，有新的大块内存分配请求，但是剩余内存不足。这个时候系统就需要回收一部分内存（比如前面提到的缓存），进而尽可能地满足新内存请求。这个过程通常被称为<strong>直接内存回收</strong>。</p><p>除了直接内存回收，还有一个专门的内核线程用来定期回收内存，也就是<strong>kswapd0</strong>。为了衡量内存的使用情况，kswapd0定义了三个内存阈值（watermark，也称为水位），分别是</p><p>页最小阈值（pages_min）、页低阈值（pages_low）和页高阈值（pages_high）。剩余内存，则使用 pages_free 表示。</p><p>这里，我画了一张图表示它们的关系。</p><p><img src="https://static001.geekbang.org/resource/image/c1/20/c1054f1e71037795c6f290e670b29120.png?wh=350*233" alt=""></p><p>kswapd0定期扫描内存的使用情况，并根据剩余内存落在这三个阈值的空间位置，进行内存的回收操作。</p><ul>
<li>
<p>剩余内存小于<strong>页最小阈值</strong>，说明进程可用内存都耗尽了，只有内核才可以分配内存。</p>
</li>
<li>
<p>剩余内存落在<strong>页最小阈值</strong>和<strong>页低阈值</strong>中间，说明内存压力比较大，剩余内存不多了。这时kswapd0会执行内存回收，直到剩余内存大于高阈值为止。</p>
</li>
<li>
<p>剩余内存落在<strong>页低阈值</strong>和<strong>页高阈值</strong>中间，说明内存有一定压力，但还可以满足新内存请求。</p>
</li>
<li>
<p>剩余内存大于<strong>页高阈值</strong>，说明剩余内存比较多，没有内存压力。</p>
</li>
</ul><p>我们可以看到，一旦剩余内存小于页低阈值，就会触发内存的回收。这个页低阈值，其实可以通过内核选项 /proc/sys/vm/min_free_kbytes 来间接设置。min_free_kbytes 设置了页最小阈值，而其他两个阈值，都是根据页最小阈值计算生成的，计算方法如下 ：</p><pre><code>pages_low = pages_min*5/4
pages_high = pages_min*3/2
</code></pre><h2>NUMA与Swap</h2><p>很多情况下，你明明发现了 Swap 升高，可是在分析系统的内存使用时，却很可能发现，系统剩余内存还多着呢。为什么剩余内存很多的情况下，也会发生 Swap 呢？</p><p>看到上面的标题，你应该已经想到了，这正是处理器的 NUMA （Non-Uniform Memory Access）架构导致的。</p><p>关于 NUMA，我在 CPU 模块中曾简单提到过。在 NUMA 架构下，多个处理器被划分到不同 Node 上，且每个 Node 都拥有自己的本地内存空间。</p><p>而同一个 Node 内部的内存空间，实际上又可以进一步分为不同的内存域（Zone），比如直接内存访问区（DMA）、普通内存区（NORMAL）、伪内存区（MOVABLE）等，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/be/d9/be6cabdecc2ec98893f67ebd5b9aead9.png?wh=388*279" alt=""></p><p>先不用特别关注这些内存域的具体含义，我们只要会查看阈值的配置，以及缓存、匿名页的实际使用情况就够了。</p><p>既然 NUMA 架构下的每个 Node 都有自己的本地内存空间，那么，在分析内存的使用时，我们也应该针对每个 Node 单独分析。</p><p>你可以通过 numactl 命令，来查看处理器在 Node 的分布情况，以及每个 Node 的内存使用情况。比如，下面就是一个 numactl 输出的示例：</p><pre><code>$ numactl --hardware
available: 1 nodes (0)
node 0 cpus: 0 1
node 0 size: 7977 MB
node 0 free: 4416 MB
...
</code></pre><p>这个界面显示，我的系统中只有一个 Node，也就是Node 0 ，而且编号为 0 和 1 的两个 CPU， 都位于 Node 0 上。另外，Node 0 的内存大小为 7977 MB，剩余内存为 4416 MB。</p><p>了解了 NUNA 的架构和 NUMA 内存的查看方法后，你可能就要问了这跟 Swap 有什么关系呢？</p><p>实际上，前面提到的三个内存阈值（页最小阈值、页低阈值和页高阈值），都可以通过内存域在 proc 文件系统中的接口 /proc/zoneinfo 来查看。</p><p>比如，下面就是一个 /proc/zoneinfo 文件的内容示例：</p><pre><code>$ cat /proc/zoneinfo
...
Node 0, zone   Normal
 pages free     227894
       min      14896
       low      18620
       high     22344
...
     nr_free_pages 227894
     nr_zone_inactive_anon 11082
     nr_zone_active_anon 14024
     nr_zone_inactive_file 539024
     nr_zone_active_file 923986
...
</code></pre><p>这个输出中有大量指标，我来解释一下比较重要的几个。</p><ul>
<li>
<p>pages处的min、low、high，就是上面提到的三个内存阈值，而free是剩余内存页数，它跟后面的 nr_free_pages 相同。</p>
</li>
<li>
<p>nr_zone_active_anon和nr_zone_inactive_anon，分别是活跃和非活跃的匿名页数。</p>
</li>
<li>
<p>nr_zone_active_file和nr_zone_inactive_file，分别是活跃和非活跃的文件页数。</p>
</li>
</ul><p>从这个输出结果可以发现，剩余内存远大于页高阈值，所以此时的 kswapd0 不会回收内存。</p><p>当然，某个 Node 内存不足时，系统可以从其他 Node 寻找空闲内存，也可以从本地内存中回收内存。具体选哪种模式，你可以通过 /proc/sys/vm/zone_reclaim_mode 来调整。它支持以下几个选项：</p><ul>
<li>
<p>默认的 0 ，也就是刚刚提到的模式，表示既可以从其他 Node 寻找空闲内存，也可以从本地回收内存。</p>
</li>
<li>
<p>1、2、4 都表示只回收本地内存，2 表示可以回写脏数据回收内存，4 表示可以用 Swap 方式回收内存。</p>
</li>
</ul><h2>swappiness</h2><p>到这里，我们就可以理解内存回收的机制了。这些回收的内存既包括了文件页，又包括了匿名页。</p><ul>
<li>
<p>对文件页的回收，当然就是直接回收缓存，或者把脏页写回磁盘后再回收。</p>
</li>
<li>
<p>而对匿名页的回收，其实就是通过 Swap 机制，把它们写入磁盘后再释放内存。</p>
</li>
</ul><p>不过，你可能还有一个问题。既然有两种不同的内存回收机制，那么在实际回收内存时，到底该先回收哪一种呢？</p><p>其实，Linux提供了一个  /proc/sys/vm/swappiness 选项，用来调整使用Swap的积极程度。</p><p>swappiness的范围是0-100，数值越大，越积极使用Swap，也就是更倾向于回收匿名页；数值越小，越消极使用Swap，也就是更倾向于回收文件页。</p><p>虽然 swappiness 的范围是 0-100，不过要注意，这并不是内存的百分比，而是调整 Swap 积极程度的权重，即使你把它设置成0，当<a href="https://www.kernel.org/doc/Documentation/sysctl/vm.txt">剩余内存+文件页小于页高阈值</a>时，还是会发生Swap。</p><p>清楚了 Swap 原理后，当遇到 Swap 使用变高时，又该怎么定位、分析呢？别急，下一节，我们将用一个案例来探索实践。</p><h2>小结</h2><p>在内存资源紧张时，Linux通过直接内存回收和定期扫描的方式，来释放文件页和匿名页，以便把内存分配给更需要的进程使用。</p><ul>
<li>
<p>文件页的回收比较容易理解，直接清空，或者把脏数据写回磁盘后再释放。</p>
</li>
<li>
<p>而对匿名页的回收，需要通过Swap换出到磁盘中，下次访问时，再从磁盘换入到内存中。</p>
</li>
</ul><p>你可以设置/proc/sys/vm/min_free_kbytes，来调整系统定期回收内存的阈值（也就是页低阈值），还可以设置/proc/sys/vm/swappiness，来调整文件页和匿名页的回收倾向。</p><p>在 NUMA 架构下，每个 Node 都有自己的本地内存空间，而当本地内存不足时，默认既可以从其他 Node 寻找空闲内存，也可以从本地内存回收。</p><p>你可以设置 /proc/sys/vm/zone_reclaim_mode ，来调整NUMA本地内存的回收策略。</p><h2>思考</h2><p>最后，我想请你一起来聊聊你理解的 SWAP。我估计你以前已经碰到过 Swap 导致的性能问题，你是怎么分析这些问题的呢？你可以结合今天讲的 Swap 原理，记录自己的操作步骤，总结自己的解决思路。</p><p>欢迎在留言区和我讨论，也欢迎把这篇文章分享给你的同事、朋友。我们一起在实战中演练，在交流中进步。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/38/d7/d549bf3e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Ray</span>
  </div>
  <div class="_2_QraFYR_0">关于上面有同学表示 hadoop 集群建议关 swap 提升性能。事实上不仅 hadoop，包括 ES 在内绝大部分 Java 的应用都建议关 swap，这个和 JVM 的 gc 有关，它在 gc 的时候会遍历所有用到的堆的内存，如果这部分内存是被 swap 出去了，遍历的时候就会有磁盘IO<br><br>可以参考这两篇文章：<br>https:&#47;&#47;www.elastic.co&#47;guide&#47;en&#47;elasticsearch&#47;reference&#47;current&#47;setup-configuration-memory.html<br><br>https:&#47;&#47;dzone.com&#47;articles&#47;just-say-no-swapping</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯嗯，大部分应用都不需要swap</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-22 16:52:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/91/b4/d5d9e4fb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>爱学习的小学生</span>
  </div>
  <div class="_2_QraFYR_0">请问老师、为什么kubernetes要关闭swap呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 一个是性能问题，开启swap会严重影响性能（包括内存和I&#47;O）；另一个是管理问题，开启swap后通过cgroups设置的内存上限就会失效。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-21 11:03:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/f8/70/f3a33a14.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>某、人</span>
  </div>
  <div class="_2_QraFYR_0">swap应该是针对以前内存小的一种优化吧,不过现在内存没那么昂贵之后,所以就没那么大的必要开启了<br>numa感觉是对系统资源做的隔离分区,不过目前虚拟化和docker这么流行。而且node与node之间访问更耗时,针对大程序不一定启到了优化作用,针对小程序,也没有太大必要。所以numa也没必要开启。<br>不知道我的理解对否,老师</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-02 20:56:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/d4RrUl4AJ3tdhsK76Kdcc9jQ2cvCzLsBfLYIiaRop7Ufj3byHrnvPS0O7sO935GG0fH8kicB67PklFdQJENZXNDQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>bob</span>
  </div>
  <div class="_2_QraFYR_0">swappiness=0<br>Kernel version 3.5 and newer: disables swapiness.<br>Kernel version older than 3.5: avoids swapping processes out of physical memory for as long as possible.<br>如果linux内核是3.5及以后的，最好是设置swappiness=10，不要设置swappiness=0<br><br>以前整理的说明，供大家参考。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢分享</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-03 09:18:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/J2FTWGBAoMDpKicgiaJ4b5AourOIQvlicJRu0iaDAJhlJkyRaialzuW9UUahN0CUQRuYLYciaN8Lu6u6ibYZrgj0XoXuQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>日行一善520</span>
  </div>
  <div class="_2_QraFYR_0">看到评论有人问<br>hadoop集群服务器一般是建议关闭swap交换空间，这样可提高性能。在什么情况下开swap、什么情况下关swap？<br><br>为了性能关闭swap，这样就不会交换也不会慢了。内核里有个vm.xx的值可以调节swap和内存的比例，在使用内存90%时才交换到swap，可以设置这个来保持性能。在内存比较少的时候，还可以交换，就好了。<br><br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢分享</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-16 14:02:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/7a/e1/326f83b9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>石头</span>
  </div>
  <div class="_2_QraFYR_0">hadoop集群服务器一般是建议关闭swap交换空间，这样可提高性能。在什么情况下开swap、什么情况下关swap？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-02 08:28:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/f2/e4/825ab8d9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>刘政伟</span>
  </div>
  <div class="_2_QraFYR_0">老师，在工作中经常会遇到这种情况，系统中的剩余内存较小、缓存内存较大的，也就是整体可用内存较高的情况下，就开始使用swap了，而查看swappiness的配置为10，理论上不应该使用swap的；具体看下面的free命令，麻烦老师看下是什么原因？<br>[root@shvsolman ~]# free -m<br>             total       used       free     shared    buffers     cached<br>Mem:         32107      31356        750          0         15      12514<br>-&#47;+ buffers&#47;cache:      18825      13281<br>Swap:         3071       1581       1490<br>[root@shvsolman ~]# sysctl -a | grep swappiness<br>vm.swappiness = 10</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这是容易误解的地方，其实，即使把swappiness设置成0也不会禁止swap。想要禁止，就不要开启swap。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-11 09:46:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/1b/ca/9afb89a2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Days</span>
  </div>
  <div class="_2_QraFYR_0">我们公司处理嵌入式系统都是关闭swap分区，具体不知道什么原因？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 一般是为了减少写的次数，延长Flash存储的寿命</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-02 12:33:46</div>
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
  <div class="_2_QraFYR_0">[D19打卡]<br>很遗憾,还未遇到过swap导致的性能问题.<br>刚买电脑时,512M内存要四百多,可是当时也不玩linux.<br>等工作了,用linux了,内存相对来说已经比较便宜了.<br>现在就更不用说了,基本小钱能解决的问题都不是问题了.<br>--------------------<br>以前的程序喜欢在启动时预分配很多内存,可是现在的几乎都是动态分配了.<br>以前一个程序动辄实际使用内存2-3G.<br>现在即使重构后,只需要100M内存,老板都不愿意换了.[稳定第一]</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍 </p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-02 11:13:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/16/31/ae8adf82.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>路过</span>
  </div>
  <div class="_2_QraFYR_0">老师，前面你写只有当剩余内存落在页最小阈值和页低阈值中间，才开始回收内存。后面讲即使把 swappiness 设置为0，当剩余内存 + 文件页小于页高阈值时，还是会发生 Swap。我理解，这里是不是应该是：当剩余内存 + 文件页小于页低阈值时，还是会发生 Swap。谢谢！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 阈值的比较只代表回收内存的时机，具体回收哪些内存才是swappiness的目的，但是swappiness只是个倾向，而非绝对值</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-03 15:22:37</div>
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
  <div class="_2_QraFYR_0">老师，可否认为匿名页就是堆内存和共享内存？两者都是应用程序控制</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-15 07:49:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLx1Jz78aibuoJEWdLTsDhucnVDTvkkeRX2w6ZJWXp0h7Zfe7GM6vKAx3jNhFhJJaElDCicyHpf1e9Q/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>13001236383</span>
  </div>
  <div class="_2_QraFYR_0">除了缓存和缓冲区，通过内存映射获取的文件映射页，也是一种常见的文件页。这个和缓存和缓冲中的文件页有啥区别了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-17 09:54:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/24/ca/932741cb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>爆爱渣科_无良🌾 🐖</span>
  </div>
  <div class="_2_QraFYR_0">感觉后面越写越变成讲述linux工具的文章。。。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 案例篇都会介绍一些常用的性能工具，用好工具事半功倍。当然，如果你有更好的方法，也欢迎分享</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-02 22:00:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKsz8j0bAayjSne9iakvjzUmvUdxWEbsM9iasQ74spGFayIgbSE232sH2LOWmaKtx1WqAFDiaYgVPwIQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>2xshu</span>
  </div>
  <div class="_2_QraFYR_0">非常感谢老师的课程。让我受益匪浅。<br>老师，我还有两个问题，想请教。<br>1：zone_reclaim_mode设置成1，也即是开启zone reclaim。此时当内存即是低于water 水位的low值，是不是也得需要满足min_unmapped_ratio所给的百分比才会让kswapd0&#47;1开始内存回收？<br>(实际场景，zone内存小于low，kswapd0&#47;1确实没有调用。 不知道是不是和min_unmapped_ratio有关系.)<br>2：min_unmapped_ratio官网解释如下：<br>This is available only on NUMA kernels.<br><br>This is a percentage of the total pages in each zone. Zone reclaim will<br>only occur if more than this percentage of pages are in a state that<br>zone_reclaim_mode allows to be reclaimed.<br><br>If zone_reclaim_mode has the value 4 OR&#39;d, then the percentage is compared<br>against all file-backed unmapped pages including swapcache pages and tmpfs<br>files. Otherwise, only unmapped pages backed by normal files but not tmpfs<br>files and similar are considered.<br><br>The default is 1 percent.<br><br>我自己理解是&#47;proc&#47;pagetypeinfo中Reclaimable所占的页数需要达到该zone总页数的百分比，才会真正回收内存？<br>希望得到老师的解答。谢谢。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-02 14:55:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/cb/0a/6a9e6602.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>React</span>
  </div>
  <div class="_2_QraFYR_0">老师好，文中说电脑的休眠是基于swap.如果系统没有分配swap分区，还会将内存数据写入磁盘吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 除了分区之外，也可以用文件</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-02 09:33:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/2d/24/28acca15.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>DJH</span>
  </div>
  <div class="_2_QraFYR_0">倪老师，请教一下，Linux下怎么关闭SWAP功能？直接不分配SWAP卷（或者分区、文件），还是通过某个关闭SWAP功能的系统选项？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: swapoff命令可以动态关闭，持久化还要从fstab里面删除</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-02 08:44:39</div>
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
  <div class="_2_QraFYR_0">打卡day20<br>我们机器上，都不启用swap😂</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我的也是😊</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-02 08:22:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/e4/23/ac13d916.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>圆哥哥呐丶</span>
  </div>
  <div class="_2_QraFYR_0">pages 处的 min、low、high：是指 page 的数量，乘上 4k 才能换算为内存大小。<br>命令：free，cat &#47;proc&#47;zoneinfo，两者的结果是一致的，可以试着去算一下，记录下留言区老哥的这段话</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-25 17:31:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/7b/2b/97e4d599.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Podman</span>
  </div>
  <div class="_2_QraFYR_0">有一个小问题求大佬们解答：<br>buffer&#47;cache都属于内存的概念，那我理解为二者是属于物理内存的概念。<br>那么二者对应到虚拟地址空间，应该对应哪部分呢？文件映射部分么？<br><br>看网上很多人说二者和虚拟地址空间没关系。但是也说不清所以然。而且我理解，如果一个程序读写文件，那么文件必然会缓存到cache中。<br>这样，程序对文件读写操作，就要对这部分cache的物理内存做操作。程序和物理内存对应的操作需要借助虚拟地址空间，也就是说，这部分cache物理内存应该是有对应的虚拟地址空间，这样程序才能借助这部分虚拟地址来读写缓存文件。那么，这部分地址属于虚拟地址空间的什么位置呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-14 14:53:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/39/fa/a7edbc72.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>安排</span>
  </div>
  <div class="_2_QraFYR_0">用户空间没有文件背景的内存页都叫匿名页，例如，堆，栈等</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-24 21:34:11</div>
  </div>
</div>
</div>
</li>
</ul>