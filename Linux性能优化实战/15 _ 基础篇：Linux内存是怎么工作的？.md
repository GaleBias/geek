<audio title="15 _ 基础篇：Linux内存是怎么工作的？" src="https://static001.geekbang.org/resource/audio/9e/29/9ed4cb575d9e5315f4e2160c5e1f6129.mp3" controls="controls"></audio> 
<p>你好，我是倪朋飞。</p><p>前几节我们一起学习了 CPU 的性能原理和优化方法，接下来，我们将进入另一个板块——内存。</p><p>同 CPU 管理一样，内存管理也是操作系统最核心的功能之一。内存主要用来存储系统和应用程序的指令、数据、缓存等。</p><p>那么，Linux 到底是怎么管理内存的呢？今天，我就来带你一起来看看这个问题。</p><h2>内存映射</h2><p>说到内存，你能说出你现在用的这台计算机内存有多大吗？我估计你记得很清楚，因为这是我们购买时，首先考虑的一个重要参数，比方说，我的笔记本电脑内存就是 8GB 的 。</p><p>我们通常所说的内存容量，就像我刚刚提到的8GB，其实指的是物理内存。物理内存也称为主存，大多数计算机用的主存都是动态随机访问内存（DRAM）。只有内核才可以直接访问物理内存。那么，进程要访问内存时，该怎么办呢？</p><p>Linux 内核给每个进程都提供了一个独立的虚拟地址空间，并且这个地址空间是连续的。这样，进程就可以很方便地访问内存，更确切地说是访问虚拟内存。</p><p>虚拟地址空间的内部又被分为<strong>内核空间和用户空间</strong>两部分，不同字长（也就是单个CPU指令可以处理数据的最大长度）的处理器，地址空间的范围也不同。比如最常见的 32 位和 64 位系统，我画了两张图来分别表示它们的虚拟地址空间，如下所示：</p><!-- [[[read_end]]] --><p><img src="https://static001.geekbang.org/resource/image/ed/7b/ed8824c7a2e4020e2fdd2a104c70ab7b.png?wh=1004*456" alt=""></p><p>通过这里可以看出，32位系统的内核空间占用 1G，位于最高处，剩下的3G是用户空间。而 64 位系统的内核空间和用户空间都是 128T，分别占据整个内存空间的最高和最低处，剩下的中间部分是未定义的。</p><p>还记得进程的用户态和内核态吗？进程在用户态时，只能访问用户空间内存；只有进入内核态后，才可以访问内核空间内存。虽然每个进程的地址空间都包含了内核空间，但这些内核空间，其实关联的都是相同的物理内存。这样，进程切换到内核态后，就可以很方便地访问内核空间内存。</p><p>既然每个进程都有一个这么大的地址空间，那么所有进程的虚拟内存加起来，自然要比实际的物理内存大得多。所以，并不是所有的虚拟内存都会分配物理内存，只有那些实际使用的虚拟内存才分配物理内存，并且分配后的物理内存，是通过<strong>内存映射</strong>来管理的。</p><p>内存映射，其实就是将<strong>虚拟内存地址</strong>映射到<strong>物理内存地址</strong>。为了完成内存映射，内核为每个进程都维护了一张页表，记录虚拟地址与物理地址的映射关系，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/fc/b6/fcfbe2f8eb7c6090d82bf93ecdc1f0b6.png?wh=572*370" alt=""></p><p>页表实际上存储在 CPU 的内存管理单元 MMU中，这样，正常情况下，处理器就可以直接通过硬件，找出要访问的内存。</p><p>而当进程访问的虚拟地址在页表中查不到时，系统会产生一个<strong>缺页异常</strong>，进入内核空间分配物理内存、更新进程页表，最后再返回用户空间，恢复进程的运行。</p><p>另外，我在 <a href="https://time.geekbang.org/column/article/69859">CPU 上下文切换的文章中</a>曾经提到， TLB（Translation Lookaside Buffer，转译后备缓冲器）会影响 CPU 的内存访问性能，在这里其实就可以得到解释。</p><p>TLB 其实就是 MMU 中页表的高速缓存。由于进程的虚拟地址空间是独立的，而 TLB 的访问速度又比 MMU 快得多，所以，通过减少进程的上下文切换，减少TLB的刷新次数，就可以提高TLB 缓存的使用率，进而提高CPU的内存访问性能。</p><p>不过要注意，MMU 并不以字节为单位来管理内存，而是规定了一个内存映射的最小单位，也就是页，通常是 4 KB大小。这样，每一次内存映射，都需要关联 4 KB 或者 4KB 整数倍的内存空间。</p><p>页的大小只有4 KB ，导致的另一个问题就是，整个页表会变得非常大。比方说，仅 32 位系统就需要 100 多万个页表项（4GB/4KB），才可以实现整个地址空间的映射。为了解决页表项过多的问题，Linux 提供了两种机制，也就是多级页表和大页（HugePage）。</p><p>多级页表就是把内存分成区块来管理，将原来的映射关系改成区块索引和区块内的偏移。由于虚拟内存空间通常只用了很少一部分，那么，多级页表就只保存这些使用中的区块，这样就可以大大地减少页表的项数。</p><p>Linux 用的正是四级页表来管理内存页，如下图所示，虚拟地址被分为5个部分，前4个表项用于选择页，而最后一个索引表示页内偏移。</p><p><img src="https://static001.geekbang.org/resource/image/b5/25/b5c9179ac64eb5c7ca26448065728325.png?wh=1390*460" alt=""></p><p>再看大页，顾名思义，就是比普通页更大的内存块，常见的大小有 2MB 和 1GB。大页通常用在使用大量内存的进程上，比如 Oracle、DPDK 等。</p><p>通过这些机制，在页表的映射下，进程就可以通过虚拟地址来访问物理内存了。那么具体到一个 Linux 进程中，这些内存又是怎么使用的呢？</p><h2>虚拟内存空间分布</h2><p>首先，我们需要进一步了解虚拟内存空间的分布情况。最上方的内核空间不用多讲，下方的用户空间内存，其实又被分成了多个不同的段。以 32 位系统为例，我画了一张图来表示它们的关系。</p><p><img src="https://static001.geekbang.org/resource/image/71/5d/71a754523386cc75f4456a5eabc93c5d.png?wh=266*354" alt=""></p><p>通过这张图你可以看到，用户空间内存，从低到高分别是五种不同的内存段。</p><ol>
<li>
<p>只读段，包括代码和常量等。</p>
</li>
<li>
<p>数据段，包括全局变量等。</p>
</li>
<li>
<p>堆，包括动态分配的内存，从低地址开始向上增长。</p>
</li>
<li>
<p>文件映射段，包括动态库、共享内存等，从高地址开始向下增长。</p>
</li>
<li>
<p>栈，包括局部变量和函数调用的上下文等。栈的大小是固定的，一般是 8 MB。</p>
</li>
</ol><p>在这五个内存段中，堆和文件映射段的内存是动态分配的。比如说，使用 C 标准库的 malloc() 或者 mmap() ，就可以分别在堆和文件映射段动态分配内存。</p><p>其实64位系统的内存分布也类似，只不过内存空间要大得多。那么，更重要的问题来了，内存究竟是怎么分配的呢？</p><h2>内存分配与回收</h2><p>malloc() 是 C 标准库提供的内存分配函数，对应到系统调用上，有两种实现方式，即 brk() 和 mmap()。</p><p>对小块内存（小于128K），C 标准库使用 brk() 来分配，也就是通过移动堆顶的位置来分配内存。这些内存释放后并不会立刻归还系统，而是被缓存起来，这样就可以重复使用。</p><p>而大块内存（大于 128K），则直接使用内存映射 mmap() 来分配，也就是在文件映射段找一块空闲内存分配出去。</p><p>这两种方式，自然各有优缺点。</p><p>brk() 方式的缓存，可以减少缺页异常的发生，提高内存访问效率。不过，由于这些内存没有归还系统，在内存工作繁忙时，频繁的内存分配和释放会造成内存碎片。</p><p>而 mmap() 方式分配的内存，会在释放时直接归还系统，所以每次 mmap 都会发生缺页异常。在内存工作繁忙时，频繁的内存分配会导致大量的缺页异常，使内核的管理负担增大。这也是malloc 只对大块内存使用 mmap  的原因。</p><p>了解这两种调用方式后，我们还需要清楚一点，那就是，当这两种调用发生后，其实并没有真正分配内存。这些内存，都只在首次访问时才分配，也就是通过缺页异常进入内核中，再由内核来分配内存。</p><p>整体来说，Linux 使用伙伴系统来管理内存分配。前面我们提到过，这些内存在MMU中以页为单位进行管理，伙伴系统也一样，以页为单位来管理内存，并且会通过相邻页的合并，减少内存碎片化（比如brk方式造成的内存碎片）。</p><p>你可能会想到一个问题，如果遇到比页更小的对象，比如不到1K的时候，该怎么分配内存呢？</p><p>实际系统运行中，确实有大量比页还小的对象，如果为它们也分配单独的页，那就太浪费内存了。</p><p>所以，在用户空间，malloc 通过 brk() 分配的内存，在释放时并不立即归还系统，而是缓存起来重复利用。在内核空间，Linux 则通过 slab 分配器来管理小内存。你可以把slab 看成构建在伙伴系统上的一个缓存，主要作用就是分配并释放内核中的小对象。</p><p>对内存来说，如果只分配而不释放，就会造成内存泄漏，甚至会耗尽系统内存。所以，在应用程序用完内存后，还需要调用 free() 或 unmap() ，来释放这些不用的内存。</p><p>当然，系统也不会任由某个进程用完所有内存。在发现内存紧张时，系统就会通过一系列机制来回收内存，比如下面这三种方式：</p><ul>
<li>
<p>回收缓存，比如使用 LRU（Least Recently Used）算法，回收最近使用最少的内存页面；</p>
</li>
<li>
<p>回收不常访问的内存，把不常用的内存通过交换分区直接写到磁盘中；</p>
</li>
<li>
<p>杀死进程，内存紧张时系统还会通过 OOM（Out of Memory），直接杀掉占用大量内存的进程。</p>
</li>
</ul><p>其中，第二种方式回收不常访问的内存时，会用到交换分区（以下简称 Swap）。Swap 其实就是把一块磁盘空间当成内存来用。它可以把进程暂时不用的数据存储到磁盘中（这个过程称为换出），当进程访问这些内存时，再从磁盘读取这些数据到内存中（这个过程称为换入）。</p><p>所以，你可以发现，Swap 把系统的可用内存变大了。不过要注意，通常只在内存不足时，才会发生 Swap 交换。并且由于磁盘读写的速度远比内存慢，Swap 会导致严重的内存性能问题。</p><p>第三种方式提到的  OOM（Out of Memory），其实是内核的一种保护机制。它监控进程的内存使用情况，并且使用 oom_score 为每个进程的内存使用情况进行评分：</p><ul>
<li>
<p>一个进程消耗的内存越大，oom_score 就越大；</p>
</li>
<li>
<p>一个进程运行占用的 CPU 越多，oom_score 就越小。</p>
</li>
</ul><p>这样，进程的 oom_score 越大，代表消耗的内存越多，也就越容易被 OOM 杀死，从而可以更好保护系统。</p><p>当然，为了实际工作的需要，管理员可以通过 /proc 文件系统，手动设置进程的 oom_adj ，从而调整进程的 oom_score。</p><p>oom_adj 的范围是 [-17, 15]，数值越大，表示进程越容易被 OOM 杀死；数值越小，表示进程越不容易被 OOM 杀死，其中 -17 表示禁止 OOM。</p><p>比如用下面的命令，你就可以把 sshd 进程的 oom_adj 调小为 -16，这样， sshd 进程就不容易被 OOM 杀死。</p><pre><code>echo -16 &gt; /proc/$(pidof sshd)/oom_adj
</code></pre><h2>如何查看内存使用情况</h2><p>通过了解内存空间的分布，以及内存的分配和回收，我想你对内存的工作原理应该有了大概的认识。当然，系统的实际工作原理更加复杂，也会涉及其他一些机制，这里我只讲了最主要的原理。掌握了这些，你可以对内存的运作有一条主线认识，不至于脑海里只有术语名词的堆砌。</p><p>那么在了解内存的工作原理之后，我们又该怎么查看系统内存使用情况呢？</p><p>其实前面CPU内容的学习中，我们也提到过一些相关工具。在这里，你第一个想到的应该是 free  工具吧。下面是一个 free 的输出示例：</p><pre><code># 注意不同版本的free输出可能会有所不同
$ free
              total        used        free      shared  buff/cache   available
Mem:        8169348      263524     6875352         668     1030472     7611064
Swap:             0           0           0
</code></pre><p>你可以看到，free 输出的是一个表格，其中的数值都默认以字节为单位。表格总共有两行六列，这两行分别是物理内存 Mem 和交换分区 Swap 的使用情况，而六列中，每列数据的含义分别为：</p><ul>
<li>
<p>第一列，total 是总内存大小；</p>
</li>
<li>
<p>第二列，used 是已使用内存的大小，包含了共享内存；</p>
</li>
<li>
<p>第三列，free 是未使用内存的大小；</p>
</li>
<li>
<p>第四列，shared 是共享内存的大小；</p>
</li>
<li>
<p>第五列，buff/cache 是缓存和缓冲区的大小；</p>
</li>
<li>
<p>最后一列，available 是新进程可用内存的大小。</p>
</li>
</ul><p>这里尤其注意一下，最后一列的可用内存available 。available不仅包含未使用内存，还包括了可回收的缓存，所以一般会比未使用内存更大。不过，并不是所有缓存都可以回收，因为有些缓存可能正在使用中。</p><p>不过，我们知道，free 显示的是整个系统的内存使用情况。如果你想查看进程的内存使用情况，可以用 top 或者 ps 等工具。比如，下面是 top 的输出示例：</p><pre><code># 按下M切换到内存排序
$ top
...
KiB Mem :  8169348 total,  6871440 free,   267096 used,  1030812 buff/cache
KiB Swap:        0 total,        0 free,        0 used.  7607492 avail Mem


  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
  430 root      19  -1  122360  35588  23748 S   0.0  0.4   0:32.17 systemd-journal
 1075 root      20   0  771860  22744  11368 S   0.0  0.3   0:38.89 snapd
 1048 root      20   0  170904  17292   9488 S   0.0  0.2   0:00.24 networkd-dispat
    1 root      20   0   78020   9156   6644 S   0.0  0.1   0:22.92 systemd
12376 azure     20   0   76632   7456   6420 S   0.0  0.1   0:00.01 systemd
12374 root      20   0  107984   7312   6304 S   0.0  0.1   0:00.00 sshd
...
</code></pre><p>top 输出界面的顶端，也显示了系统整体的内存使用情况，这些数据跟 free 类似，我就不再重复解释。我们接着看下面的内容，跟内存相关的几列数据，比如 VIRT、RES、SHR 以及 %MEM 等。</p><p>这些数据，包含了进程最重要的几个内存使用情况，我们挨个来看。</p><ul>
<li>
<p>VIRT 是进程虚拟内存的大小，只要是进程申请过的内存，即便还没有真正分配物理内存，也会计算在内。</p>
</li>
<li>
<p>RES 是常驻内存的大小，也就是进程实际使用的物理内存大小，但不包括 Swap 和共享内存。</p>
</li>
<li>
<p>SHR 是共享内存的大小，比如与其他进程共同使用的共享内存、加载的动态链接库以及程序的代码段等。</p>
</li>
<li>
<p>%MEM 是进程使用物理内存占系统总内存的百分比。</p>
</li>
</ul><p>除了要认识这些基本信息，在查看 top 输出时，你还要注意两点。</p><p>第一，虚拟内存通常并不会全部分配物理内存。从上面的输出，你可以发现每个进程的虚拟内存都比常驻内存大得多。</p><p>第二，共享内存 SHR 并不一定是共享的，比方说，程序的代码段、非共享的动态链接库，也都算在 SHR 里。当然，SHR 也包括了进程间真正共享的内存。所以在计算多个进程的内存使用时，不要把所有进程的 SHR 直接相加得出结果。</p><h2>小结</h2><p>今天，我们梳理了 Linux 内存的工作原理。对普通进程来说，它能看到的其实是内核提供的虚拟内存，这些虚拟内存还需要通过页表，由系统映射为物理内存。</p><p>当进程通过 malloc() 申请内存后，内存并不会立即分配，而是在首次访问时，才通过缺页异常陷入内核中分配内存。</p><p>由于进程的虚拟地址空间比物理内存大很多，Linux 还提供了一系列的机制，应对内存不足的问题，比如缓存的回收、交换分区 Swap 以及 OOM 等。</p><p>当你需要了解系统或者进程的内存使用情况时，可以用 free 和 top 、ps 等性能工具。它们都是分析性能问题时最常用的性能工具，希望你能熟练使用它们，并真正理解各个指标的含义。</p><h2>思考</h2><p>最后，我想请你来聊聊你所理解的Linux内存。你碰到过哪些内存相关的性能瓶颈？你又是怎么样来分析它们的呢？你可以结合今天学到的内存知识和工作原理，提出自己的观点。</p><p>欢迎在留言区和我讨论，也欢迎你把这篇文章分享给你的同事、朋友。我们一起在实战中演练，在交流中进步。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLdWHFCr66TzHS2CpCkiaRaDIk3tU5sKPry16Q7ic0mZZdy8LOCYc38wOmyv5RZico7icBVeaPX8X2jcw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>JohnT3e</span>
  </div>
  <div class="_2_QraFYR_0">目前我配合使用《Operating System Concepts》这本书学习此专栏，也是一个不错的选择。此书中的第四部分“Memory Management”对今天这部分内容有较为详细的描述，感兴趣的同学可以去看一下。另外对于TOP命令中输出的内存情况解释，可以认真看一下man手册的内容。如果你动手能力比较强，可以看https:&#47;&#47;blog.holbertonschool.com&#47;hack-the-virtual-memory-malloc-the-heap-the-program-break&#47;这篇博客，手把手教你使用程序（C语言）来实际地去搞清楚虚拟内存分布。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍 谢谢分享，这篇博客很详细</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-24 14:02:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/2a/6d/0e436c1c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>梦回汉唐</span>
  </div>
  <div class="_2_QraFYR_0">1、调用c标准库的都是用户空间的调用，用户空间的内存分配都是基于buddy算法（伙伴算法），并不涉及slab<br>2、brk()方式之所以会产生内存碎片，是由于brk分配的内存是推_edata指针，从堆的低地址向高地址推进。这种情况下，如果高地址的内存不释放，低地址的内存是得不到释放的<br>3、mmap()方式分配的内存，是在堆与栈之间的空闲区域分配虚拟内存，直接拿到的是内存地址，可以直接操作内存的释放<br><br>上述的都是在用户空间发生的行为，只有在内核空间，内核调用kmalloc去分配内存的时候，才会涉及到slab</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-27 10:18:14</div>
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
  <div class="_2_QraFYR_0">[D15打卡]<br>linux的内存跟windows的很不一样。类linux的系统会尽量使用内存缓存东西，提供运行效率。所以linux&#47;mac显示的free剩余内存通常很小，但实际上被缓存的cache可能很大，并不代表系统内存紧张！<br>曾经就闹过笑话，看见系统free值很低，怕程序因为oom被系统杀掉，还特意写个c程序去挤内存。程序不停的申请1MB内存然后memset，随机挑几个位置写，保证申请的都被加载到物理内存中。（跟文中描述的一致，只申请不使用不会加载到物理内存）然后挤的差不多了就把测试程序关掉。看上去free变大了很多很开心。现在想想，就是掩耳盗铃罢了。<br>以前物理机上还有swap交换分区，现在都是云服务器，基本没有了该分区。也不会遇到因为频繁使用交换分区导致性能下降的问题了。<br>我内存方面的问题遇到的都比较简单，基本上就是top&#47;free看看系统和各程序的，找到有问题的程序，看看是否有内存泄露。平常不泄漏都是够用的。<br>redis对内存比较敏感，曾经就因为配置项是默认值，在内存用完后，所有的set操作都直接返回错误，导致线上系统故障。（redis在备份时会新开一个进程，实际使用内存量会翻番。）后来会定期检查redis 的info memory 看内存使用情况。<br>——————————<br>期待的内存篇开始了，好开心！又可以跟着老师学新知识啦！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍 谢谢分享你的经历。缓存和swap后面都还会细讲</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-24 01:19:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/4d/fe/882eaf0f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>威</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，请问如果发生了OOM，系统会记录下是哪个（些）进程被杀掉吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 会的，比如 dmesg |grep -E ‘kill|oom|out of memory’</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-24 09:44:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/2o1Izf2YyJSnnI0ErZ51pYRlnrmibqUTaia3tCU1PjMxuwyXSKOLUYiac2TQ5pd5gNGvS81fVqKWGvDsZLTM8zhWg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>划时代</span>
  </div>
  <div class="_2_QraFYR_0">我的分析步骤：使用top和ps查询系统中大量占用内存的进程，使用cat &#47;proc&#47;[pid]&#47;status和pmap -x pid查看某个进程使用内存的情况和动态变化。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍 </p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-24 09:34:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/b3/46/ec914238.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>espzest</span>
  </div>
  <div class="_2_QraFYR_0">用户空间malloc不会使用slab机制吧，slab是内核空间的，对不？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯嗯，是的，谢谢指出。slab是内核空间的，只用来管理内核中的小块内存</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-24 08:33:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/63/3b/c5cd68ab.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>生活不如诗</span>
  </div>
  <div class="_2_QraFYR_0">一、内存工作原理：<br>	1.linux内核给每个进程提供了一个独立的虚拟地址空间，并且这个空间是连续的。<br>	2.虚拟地址空间分为内核空间和用户空间。进程在用户态时只能访问用户空间内存，只有进入内核态后才能访问内核空间内存；虽然每个进程都有内核空间，但这些内核空间关联的是相同的物理内存。<br>	3.并不是所有的虚拟内存都会分配物理内存，只有实际使用的虚拟内存才分配物理内存；<br>二、内存回收的三种方式：<br>	1.回收缓存，比如通过LRU算法回收最近很少使用的内存页面；<br>	2.回收不常访问的内存，把不常用的内存通过交换分区直接写到磁盘中；<br>		注意：此方法会用到swap分区。把进程暂时不用的数据放到磁盘（swap）上，不过会严重影响性能；<br>	3.通过oom杀死进程；<br>		1.一个进程消耗的内存越大，oom_score越大；<br>		2.一个进程运行占用的cpu越多，oom_score越小；<br>		3.oom_score越大的进程，越容易被OOM杀死；<br>		4.可以通过调整&#47;proc&#47;${pidof sshd)&#47;oom_adj来调整oom_score，值范围是[-17,15]，-17表示禁止被OOM；<br>三、内存查看方式：<br>	1. free<br>	2. top<br>		1. VIRT是进程申请的虚拟内存，比实际占用内存要大得多；<br>		2. RES是常驻内存，是进程实际占用内存；<br>		3. SHR是共享内存，比如与其他进程共同使用的共享内存、加载的动态链接库以及程序的代码段等。不过SHR也会有程序代码段，非共享动态链接库等，所以不能把多个进程的SHR相加得结果；<br>	注意：<br>		1. 虚拟内存通常不会全部分配到物理内存；</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-01 15:47:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/61/bc/88a905a5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>亮点</span>
  </div>
  <div class="_2_QraFYR_0">倪老师，“Operating Systems Design and Implementation”里面说，大部分页式存储管理，页表是存储到内存里面的，为降低频繁访问内存计算物理地址，引入TLB用于缓存常用映射以提升性能。以现在流行的linux为例，本节课说页表存储在MMU是否还正确呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: MMU全称就是内存管理单元，管理地址映射关系（也就是页表）。但MMU的性能跟CPU比还是不够快，所以又有了TLB。TLB实际上是MMU的一部分，把页表缓存起来以提升性能。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-24 19:46:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/94/47/75875257.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>虎虎❤️</span>
  </div>
  <div class="_2_QraFYR_0">请教老师几个问题<br><br>1. 当内存紧张时，系统通过三种机制回收内存。第二种换页比较好理解， 但是第一种LRU回收内存页怎么理解？回收后的页去哪了?如果直接删除会导致程序出问题吗？<br><br>2. OOM的分数是参照进程的实际消耗内存还是虚拟内存大小？<br><br>3. 进程启动时，是不是需要分配一个最小的内存？都包括什么呢？如何确定最小的内存是多大呢？比如我在k8s里设置container的request值，我希望是能容乃下这个container的最小内存，有没有办法计算呢？<br><br>有的问题比较小白，望老师包含。如果无法简短的回答，能否推荐些资料呢？谢谢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-24 10:29:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/09/5c/b5d79d20.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>李亮亮</span>
  </div>
  <div class="_2_QraFYR_0">我的理解就是，Linux里面所有的概念皆结构（C语言的结构体），页表也是一种结构体，也就是数据，所以保存还是在内存里面。mmu其实就是一个硬件，它的作用就是找到某个虚拟地址到物理地址的映射，也就是找到正确的页表项。然后再交给CPU，当然也可以把找到的结果缓存在TBL中，这样CPU下次使用同一个虚拟地址就省了转换这步了，老师，不知道我理解的有错吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对的，Linux也是一个程序，本质上还是数据结构+算法😊</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-28 21:06:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/61/bc/88a905a5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>亮点</span>
  </div>
  <div class="_2_QraFYR_0">倪老师，如果进程间的虚拟内存空间独立，每个进程又有自己的页表，那么进程怎么获知其他进程占用的物理内存情况，怎么防止覆盖其他进程的物理内存块呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. 进程不需要获取其他进程的内存情况<br>2. 所有进程的内存都是由内核来管理的，内核保证内存的访问安全。比如，访问非法地址时，进程会panic</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-24 19:20:39</div>
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
  <div class="_2_QraFYR_0">打卡day16<br>工作中，发生oom基本都是程序跑的，都甩给研发了～😂</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 😂</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-24 08:24:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/db/26/54f2c164.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>靠人品去赢</span>
  </div>
  <div class="_2_QraFYR_0">看的很爽这篇文章，之前知道一点操作系统关于内存的分配使用，内存映射管理物理内存和虚拟内存的对应关系，也知道页。但是还没见过将二者串起来，好多疑问解决。<br>我们经常说的多少位长，到底有什么区别？就是虚拟内存空间大小，以及空间的分配区别。<br>虚拟内存和物理内存的分配，怎么让物理内存能及时跟进程需要的虚拟内存交互对应起来？内核态和用户态的切换，知道进程有个虚拟地址，一个是用户的，一个是内核的，当调用进程的某些内存发现维护的内存映射没有对应的，及切换到内核态，内核把对应的虚拟内存放进去物理内存，再切回来用户态，进程继续，妙。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-11 15:49:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>zhaolm</span>
  </div>
  <div class="_2_QraFYR_0">https:&#47;&#47;blog.csdn.net&#47;tennysonsky&#47;article&#47;details&#47;45092229<br>这边文章关于的内存使用  比较容易懂；</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-18 17:09:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/dd/a6/adb3b1d9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>圣域烈焰</span>
  </div>
  <div class="_2_QraFYR_0">请教一个问题，使用free命令的时候，看到的buffer、cache和shared有什么区别？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 下一期讲 buffer 和 cache</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-24 17:36:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/ac/db/51399da1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>大甜菜</span>
  </div>
  <div class="_2_QraFYR_0">malloc本身不会使用slab<br>只有内核中使用kmalloc才回通过slab分配内存</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯嗯，是的，谢谢指出</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-24 11:05:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/0b/0a/fa152399.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>wahaha</span>
  </div>
  <div class="_2_QraFYR_0">请问，当手动让一个进程暂停后，如何手动让内核立即swap out该进程占用的内存？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-24 09:18:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/rxz5aKicRkvqWmt6c6c7eayHvh577uibBTVQzcJKwSTqI9FaxZSRlx7NRVw4atWpqER8ncA5jErQb3wb4cPzZxlA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_065895</span>
  </div>
  <div class="_2_QraFYR_0">15.<br>关键字：<br>内存映射 <br>虚拟地址空间<br>MMU<br>TLB<br>多级页表<br>大页<br>虚拟内存空间分布<br>malloc和mmap比较<br>内存回收的三种方式<br>交换区<br>OOM机制<br>free<br>top<br>ps<br>知识点：<br>内存映射：将虚拟地址转变为物理地址<br>虚拟地址空间：<br>MMU：Memory Management Unit，负责进行内存映射，存储页表<br>TLB：页表缓存<br>多级页表：多级映射<br>大页：每个页大小变大，比如1M等，适用于数据库等<br>虚拟内存空间分布：从上往下，栈区、文件映射、堆区、数据段、只读段<br>malloc和mmap比较：malloc通过控制brk（）进行内存分配，释放后会先缓存起来；mmap进行文件映射，释放后直接清理内存。频繁mmap会触发大量缺页中断。两种方法都是第一次访问时才分配内存<br>内存回收三种方式：回收缓存，比如用LRU算法；回收不常用的内存（通过交换分区直接写到磁盘中）；OOM，直接杀掉占用大量内存的进程<br>交换区：<br>OOM机制：内核用oom_score为每一个进程的内存使用情况进行评分：消耗内存越大，占用CPU越少，oom_score越大，越容易被OOM杀死<br><br>命令行指令：<br>free<br>top:VIRT（虚拟内存大小），RES(常驻内存大小)，SHR（共享内存大小），<br>ps<br><br><br>问题<br>1. 为啥多级页表能减少页表项数呢<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-24 09:42:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/69/44/1588e8b6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>刘文坛</span>
  </div>
  <div class="_2_QraFYR_0">页表是放在内存当中的，不是在mmu中吧？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-06 18:43:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/a9/36/d054c979.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>G.S.K</span>
  </div>
  <div class="_2_QraFYR_0">请教老师，如果一个程序的可执行文件大小为2M，那么运行这个程序后，进程的虚拟内存空间中的只读段就是大约2M的大小吗？如果一个程序的可执行文件大小为2G，那么运行这个程序后，进程的虚拟内存空间中的只读段就是大约2G的大小吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-14 21:08:44</div>
  </div>
</div>
</div>
</li>
</ul>