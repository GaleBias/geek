<audio title="03 _ 基础篇：经常说的 CPU 上下文切换是什么意思？（上）" src="https://static001.geekbang.org/resource/audio/be/3a/bea8e1df9d4cbddc0b66fc753e8cee3a.mp3" controls="controls"></audio> 
<p>你好，我是倪朋飞。</p><p>上一节，我给你讲了要怎么理解平均负载（ Load Average），并用三个案例展示了不同场景下平均负载升高的分析方法。这其中，多个进程竞争 CPU 就是一个经常被我们忽视的问题。</p><p>我想你一定很好奇，进程在竞争 CPU 的时候并没有真正运行，为什么还会导致系统的负载升高呢？看到今天的主题，你应该已经猜到了，CPU 上下文切换就是罪魁祸首。</p><p>我们都知道，Linux 是一个多任务操作系统，它支持远大于 CPU 数量的任务同时运行。当然，这些任务实际上并不是真的在同时运行，而是因为系统在很短的时间内，将 CPU 轮流分配给它们，造成多任务同时运行的错觉。</p><p>而在每个任务运行前，CPU 都需要知道任务从哪里加载、又从哪里开始运行，也就是说，需要系统事先帮它设置好 <strong>CPU 寄存器和程序计数器</strong>（Program Counter，PC）。</p><p>CPU 寄存器，是 CPU 内置的容量小、但速度极快的内存。而程序计数器，则是用来存储 CPU 正在执行的指令位置、或者即将执行的下一条指令位置。它们都是 CPU 在运行任何任务前，必须的依赖环境，因此也被叫做 <strong>CPU 上下文</strong>。</p><p><img src="https://static001.geekbang.org/resource/image/98/5f/98ac9df2593a193d6a7f1767cd68eb5f.png?wh=438*345" alt=""></p><p>知道了什么是 CPU 上下文，我想你也很容易理解 <strong>CPU 上下文切换</strong>。CPU 上下文切换，就是先把前一个任务的 CPU 上下文（也就是 CPU 寄存器和程序计数器）保存起来，然后加载新任务的上下文到这些寄存器和程序计数器，最后再跳转到程序计数器所指的新位置，运行新任务。</p><!-- [[[read_end]]] --><p>而这些保存下来的上下文，会存储在系统内核中，并在任务重新调度执行时再次加载进来。这样就能保证任务原来的状态不受影响，让任务看起来还是连续运行。</p><p>我猜肯定会有人说，CPU 上下文切换无非就是更新了 CPU 寄存器的值嘛，但这些寄存器，本身就是为了快速运行任务而设计的，为什么会影响系统的 CPU 性能呢？</p><p>在回答这个问题前，不知道你有没有想过，操作系统管理的这些“任务”到底是什么呢？</p><p>也许你会说，任务就是进程，或者说任务就是线程。是的，进程和线程正是最常见的任务。但是除此之外，还有没有其他的任务呢？</p><p>不要忘了，硬件通过触发信号，会导致中断处理程序的调用，也是一种常见的任务。</p><p>所以，<span class="orange">根据任务的不同</span>，CPU 的上下文切换就可以分为几个不同的场景，也就是<strong>进程上下文切换</strong>、<strong>线程上下文切换</strong>以及<strong>中断上下文切换</strong>。</p><p>这节课我就带你来看看，怎么理解这几个不同的上下文切换，以及它们为什么会引发 CPU 性能相关问题。</p><h2>进程上下文切换</h2><p>Linux 按照特权等级，把进程的运行空间分为内核空间和用户空间，分别对应着下图中， CPU 特权等级的 Ring 0 和 Ring 3。</p><ul>
<li>
<p>内核空间（Ring 0）具有最高权限，可以直接访问所有资源；</p>
</li>
<li>
<p>用户空间（Ring 3）只能访问受限资源，不能直接访问内存等硬件设备，必须通过系统调用陷入到内核中，才能访问这些特权资源。</p>
</li>
</ul><p><img src="https://static001.geekbang.org/resource/image/4d/a7/4d3f622f272c49132ecb9760310ce1a7.png?wh=321*312" alt=""></p><p>换个角度看，也就是说，进程既可以在用户空间运行，又可以在内核空间中运行。进程在用户空间运行时，被称为进程的用户态，而陷入内核空间的时候，被称为进程的内核态。</p><p>从用户态到内核态的转变，需要通过<strong>系统调用</strong>来完成。比如，当我们查看文件内容时，就需要多次系统调用来完成：首先调用 open() 打开文件，然后调用 read() 读取文件内容，并调用 write() 将内容写到标准输出，最后再调用 close() 关闭文件。</p><p>那么，系统调用的过程有没有发生 CPU 上下文的切换呢？答案自然是肯定的。</p><p>CPU 寄存器里原来用户态的指令位置，需要先保存起来。接着，为了执行内核态代码，CPU 寄存器需要更新为内核态指令的新位置。最后才是跳转到内核态运行内核任务。</p><p>而系统调用结束后，CPU寄存器需要<strong>恢复</strong>原来保存的用户态，然后再切换到用户空间，继续运行进程。<span class="orange">所以，一次系统调用的过程，其实是发生了两次 CPU 上下文切换。</span></p><p>不过，需要注意的是，系统调用过程中，并不会涉及到虚拟内存等进程用户态的资源，也不会切换进程。这跟我们通常所说的进程上下文切换是不一样的：</p><ul>
<li>
<p>进程上下文切换，是指从一个进程切换到另一个进程运行。</p>
</li>
<li>
<p>而系统调用过程中一直是同一个进程在运行。</p>
</li>
</ul><p>所以，<strong>系统调用过程通常称为特权模式切换，而不是上下文切换</strong>。但实际上，系统调用过程中，CPU 的上下文切换还是无法避免的。</p><p>那么，进程上下文切换跟系统调用又有什么区别呢？</p><p>首先，你需要知道，进程是由内核来管理和调度的，进程的切换只能发生在内核态。所以，进程的上下文不仅包括了虚拟内存、栈、全局变量等用户空间的资源，还包括了内核堆栈、寄存器等内核空间的状态。</p><p>因此，进程的上下文切换就比系统调用时多了一步：在保存当前进程的内核状态和CPU寄存器之前，需要先把该进程的虚拟内存、栈等保存下来；而加载了下一进程的内核态后，还需要刷新进程的虚拟内存和用户栈。</p><p>如下图所示，保存上下文和恢复上下文的过程并不是“免费”的，需要内核在 CPU 上运行才能完成。</p><p><img src="https://static001.geekbang.org/resource/image/39/6b/395666667d77e718da63261be478a96b.png?wh=966*186" alt=""></p><p>根据<a href="https://blog.tsunanet.net/2010/11/how-long-does-it-take-to-make-context.html"> Tsuna </a>的测试报告，每次上下文切换都需要几十纳秒到数微秒的 CPU 时间。这个时间还是相当可观的，特别是在进程上下文切换次数较多的情况下，很容易导致 CPU 将大量时间耗费在寄存器、内核栈以及虚拟内存等资源的保存和恢复上，进而大大缩短了真正运行进程的时间。这也正是上一节中我们所讲的，导致平均负载升高的一个重要因素。</p><p>另外，我们知道， Linux 通过 TLB（Translation Lookaside Buffer）来管理虚拟内存到物理内存的映射关系。当虚拟内存更新后，TLB 也需要刷新，内存的访问也会随之变慢。特别是在多处理器系统上，缓存是被多个处理器共享的，刷新缓存不仅会影响当前处理器的进程，还会影响共享缓存的其他处理器的进程。</p><p>知道了进程上下文切换潜在的性能问题后，我们再来看，究竟什么时候会切换进程上下文。</p><p>显然，进程切换时才需要切换上下文，换句话说，只有在进程调度的时候，才需要切换上下文。Linux 为每个 CPU 都维护了一个就绪队列，将活跃进程（即正在运行和正在等待CPU的进程）按照优先级和等待 CPU 的时间排序，然后选择最需要 CPU 的进程，也就是优先级最高和等待CPU时间最长的进程来运行。</p><p>那么，进程在什么时候才会被调度到 CPU 上运行呢？</p><p>最容易想到的一个时机，就是进程执行完终止了，它之前使用的CPU会释放出来，这个时候再从就绪队列里，拿一个新的进程过来运行。<span class="orange">其实还有很多其他场景，也会触发进程调度，在这里我给你逐个梳理下</span>。</p><p>其一，为了保证所有进程可以得到公平调度，CPU时间被划分为一段段的时间片，这些时间片再被轮流分配给各个进程。这样，当某个进程的时间片耗尽了，就会被系统挂起，切换到其它正在等待 CPU 的进程运行。</p><p>其二，进程在系统资源不足（比如内存不足）时，要等到资源满足后才可以运行，这个时候进程也会被挂起，并由系统调度其他进程运行。</p><p>其三，当进程通过睡眠函数  sleep 这样的方法将自己主动挂起时，自然也会重新调度。</p><p>其四，当有优先级更高的进程运行时，为了保证高优先级进程的运行，当前进程会被挂起，由高优先级进程来运行。</p><p>最后一个，发生硬件中断时，CPU上的进程会被中断挂起，转而执行内核中的中断服务程序。</p><p>了解这几个场景是非常有必要的，因为一旦出现上下文切换的性能问题，它们就是幕后凶手。</p><h2>线程上下文切换</h2><p>说完了进程的上下文切换，我们再来看看线程相关的问题。</p><p>线程与进程最大的区别在于，<strong>线程是调度的基本单位，而进程则是资源拥有的基本单位</strong>。说白了，所谓内核中的任务调度，实际上的调度对象是线程；而进程只是给线程提供了虚拟内存、全局变量等资源。所以，对于线程和进程，我们可以这么理解：</p><ul>
<li>
<p>当进程只有一个线程时，可以认为进程就等于线程。</p>
</li>
<li>
<p>当进程拥有多个线程时，这些线程会共享相同的虚拟内存和全局变量等资源。这些资源在上下文切换时是不需要修改的。</p>
</li>
<li>
<p>另外，线程也有自己的私有数据，比如栈和寄存器等，这些在上下文切换时也是需要保存的。</p>
</li>
</ul><p>这么一来，线程的上下文切换其实就可以分为两种情况：</p><p>第一种， 前后两个线程属于不同进程。此时，因为资源不共享，所以切换过程就跟进程上下文切换是一样。</p><p>第二种，前后两个线程属于同一个进程。此时，因为虚拟内存是共享的，所以在切换时，虚拟内存这些资源就保持不动，只需要切换线程的私有数据、寄存器等不共享的数据。</p><p>到这里你应该也发现了，虽然同为上下文切换，但同进程内的线程切换，要比多进程间的切换消耗更少的资源，而这，也正是多线程代替多进程的一个优势。</p><h2>中断上下文切换</h2><p>除了前面两种上下文切换，还有一个场景也会切换 CPU 上下文，那就是中断。</p><p>为了快速响应硬件的事件，<strong>中断处理会打断进程的正常调度和执行</strong>，转而调用中断处理程序，响应设备事件。而在打断其他进程时，就需要将进程当前的状态保存下来，这样在中断结束后，进程仍然可以从原来的状态恢复运行。</p><p>跟进程上下文不同，中断上下文切换并不涉及到进程的用户态。所以，即便中断过程打断了一个正处在用户态的进程，也不需要保存和恢复这个进程的虚拟内存、全局变量等用户态资源。中断上下文，其实只包括内核态中断服务程序执行所必需的状态，包括CPU 寄存器、内核堆栈、硬件中断参数等。</p><p><strong>对同一个 CPU 来说，中断处理比进程拥有更高的优先级</strong>，所以中断上下文切换并不会与进程上下文切换同时发生。同样道理，由于中断会打断正常进程的调度和执行，所以大部分中断处理程序都短小精悍，以便尽可能快的执行结束。</p><p>另外，跟进程上下文切换一样，中断上下文切换也需要消耗CPU，切换次数过多也会耗费大量的 CPU，甚至严重降低系统的整体性能。所以，当你发现中断次数过多时，就需要注意去排查它是否会给你的系统带来严重的性能问题。</p><h2>小结</h2><p>总结一下，不管是哪种场景导致的上下文切换，你都应该知道：</p><ol>
<li>
<p>CPU 上下文切换，是保证 Linux 系统正常工作的核心功能之一，一般情况下不需要我们特别关注。</p>
</li>
<li>
<p>但过多的上下文切换，会把CPU时间消耗在寄存器、内核栈以及虚拟内存等数据的保存和恢复上，从而缩短进程真正运行的时间，导致系统的整体性能大幅下降。</p>
</li>
</ol><p>今天主要为你介绍这几种上下文切换的工作原理，下一节，我将继续案例实战，说说上下文切换问题的分析方法。</p><h2>思考</h2><p>最后，我想邀请你一起来聊聊，你所理解的 CPU 上下文切换。你可以结合今天的内容，总结自己的思路和看法，写下你的学习心得。</p><p>欢迎在留言区和我讨论。</p><p><img src="https://static001.geekbang.org/resource/image/56/52/565d66d658ad23b2f4997551db153852.jpg?wh=1110*549" alt=""></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/e7/47/d0715205.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>D白菜</span>
  </div>
  <div class="_2_QraFYR_0">今天的课抽象的东西很多，还需要时间理解消化。对学习做了个简单的总结，初学者有何很多地方理解不对，请老师和童鞋们多多指正。<br>1、多任务竞争CPU，cpu变换任务的时候进行CPU上下文切换(context switch)。CPU执行任务有4种方式：进程、线程、或者硬件通过触发信号导致中断的调用。<br>2、当切换任务的时候，需要记录任务当前的状态和获取下一任务的信息和地址(指针)，这就是上下文的内容。因此，上下文是指某一时间点CPU寄存器(CPU register)和程序计数器(PC)的内容, 广义上还包括内存中进程的虚拟地址映射信息.<br>3、上下文切换的过程：<br>      (1)记录当前任务的上下文(即寄存器和计算器等所有的状态)；<br>      (2)找到新任务的上下文并加载；<br>      (3)切换到新任务的程序计算器位置，恢复其任务。<br>4、根据任务的执行形式，相应的下上文切换，有进程上下文切换、线程上下文切换、以及中断上下文切换三类。<br>5、进程和线程的区别：<br>进程是资源分配和执行的基本单位；线程是任务调度和运行的基本单位。线程没有资源，进程给指针提供虚拟内存、栈、变量等共享资源，而线程可以共享进程的资源。<br>6、进程上下文切换：是指从一个进程切换到另一个进程。<br>(1)进程运行态为内核运行态和进程运行态。内核空间态资源包括内核的堆栈、寄存器等；用户空间态资源包括虚拟内存、栈、变量、正文、数据等<br>(2)系统调用(软中断)在内核态完成的，需要进行2次CPU上下文切换(用户空间--&gt;内核空间--&gt;用户空间)，不涉及用户态资源，也不会切换进程。<br>(3)进程是由内核来管理和调度的，进程的切换只能发生在内核态。所以，进程的上下文不仅包括了用户空间的资源，也包括内核空间资源。<br>(4)进程的上下文切换过程：<br>   (a)接收到切换信号，挂起进程，记录当前进程的虚拟内存、栈等资源存储;<br>   (b)将这个进程在 CPU 中的上下文状态存储于起来;<br>   (c)然后在内存中检索下一个进程的上下文;<br>   (d)并将其加载到 CPU的寄存器中恢复;<br>   (3)还需要刷新进程的虚拟内存和用户栈;<br>   (f)最后跳转到程序计数器所指向的位置（即跳转到进程被中断时的代码行），以恢复该进程。<br>(5)、下列将会触发进程上下文切换的场景：<br>  (a)、根据调度策略，将CPU时间划片为对应的时间片，当时间片耗尽，当前进程必须挂起。<br>  (b)、资源不足的，在获取到足够资源之前进程挂起。<br>  (c)、进程sleep挂起进程。<br>  (d)、高优先级进程导致当前进度挂起<br>  (e)、硬件中断，导致当前进程挂起<br>7、线程上下文切换：<br>(1)、不通进程之间的线程上下文切换，其过程和进程上下文切换大致相同。<br>(2)、进程内部的线程进上下文切换。不需要切换进程的用户资源，只需要切换线程私有的数据和寄存器等。这会比进程上下文进程切换消耗的资源少，所以多线程相比多进程的优势。<br>8、中断上下文切换<br>快速响应硬件的事件，中断处理会打断进程的正常调度和执行。同一CPU内，硬件中断优先级高于进程。切换过程类似于系统调用的时候，不涉及到用户运行态资源。但大量的中断上下文切换同样可能引发性能问题。<br><br>有些疑惑，想请教老师，<br>(1)从用户态和内核态之间的转变过程通过系统调用，那么系统调用属于一种上下文切换吗？如果是上下文切换，属于哪一类上下文切换呢，和进程上下文切换的关系是什么？<br>(2)指令寄存器和程序计算器，这两个东东比较模糊。<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍课代表来了<br><br>（1）系统调用属于同进程内的CPU上下文切换；<br>（2）分别是CPU用来存储指令和下一条要执行指令的寄存器</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-26 17:09:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/umxqic2CHGySYhT47Rz03ePCloIZ7X21dCLZMVo82m2gjhJdJVqYt3AOdZUXg1roTNSRibkj2g5eia76cbFdxjliag/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小花</span>
  </div>
  <div class="_2_QraFYR_0">进程切换我想到了很多年前在银行柜台办理业务的情形。<br>1：银行分配各个窗口给来办理业务的人<br>2：如果只有1个窗口开放（系统资源不足），大部分都得等<br>3：如果正在办理业务的突然说自己不办了（sleep）,那他就去旁边再想想（等）<br>4：如果突然来了个VIP客户，可以强行插队<br>5：如果突然断电了（中断），都得等。。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞，太形象了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-25 18:16:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJB6kaR6vXGP95RvKy6GnmT33EEAkoPJrUCENE0ankWfKicm6QMbfw6yiadNJlIPZEIlPYgIm8mYEwg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>龙猫</span>
  </div>
  <div class="_2_QraFYR_0">怎么知道cpu是耗费在上下文切换呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-26 17:45:32</div>
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
  <div class="_2_QraFYR_0">学习总结:<br><br>* 上下文切换是什么？<br>上下文切换是对任务当前运行状态的暂存和恢复<br>* CPU为什么要进行上下文切换？<br>当多个进程竞争CPU的时候，CPU为了保证每个进程能公平被调度运行，采取了处理任务时间分片的机制，轮流处理多个进程，由于CPU处理速度非常快，在人类的感官上认为是并行处理，实际是&quot;伪&quot;并行，同一时间只有一个任务在运行处理。<br>* 上下文切换主要消耗什么资源，为什么说上下文切换次数过多不可取？<br>根据 Tsuna 的测试报告，每次上下文切换都需要几十纳秒到到微秒的CPU时间，这些时间对CPU来说，就好比人类对1分钟或10分钟的感觉概念。在分秒必争的计算机处理环境下，浪费太多时间在切换上，只能会降低真正处理任务的时间，表象上导致延时、排队、卡顿现象发生。<br>* 上下文切换分几种？<br>进程上下文切换、线程上下文切换、中断上下文切换<br>* 什么情况下会触发上下文切换？<br>系统调用、进程状态转换(运行、就绪、阻塞)、时间片耗尽、系统资源不足、sleep、优先级调度、硬件中断等<br>* 线程上下文切换和进程上下文切换的最大区别？<br>线程是调度的基本单位，进程是资源拥有的基本单位，同属一个进程的线程，发生上下文切换，只切换线程的私有数据，共享数据不变，因此速度非常快。<br>* 中断上下文切换，如何理解？<br>为了快速响应硬件的事件(如USB接入)，中断处理会打断进程的正常调度和执行，转而调用中断处理程序，响应设备事件。而打断其它进程执行时，需要进行上下文切换。中断事件过多，会无谓的消耗CPU资源，导致进程处理时间延长。<br>* 有哪些减少上下文切换的技术用例？<br>数据库连接池（复用连接）、合理设置应用的最大进程，线程数、直接内存访问DMA、零拷贝技术</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-25 09:14:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/d9/d3/63742921.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>炀</span>
  </div>
  <div class="_2_QraFYR_0">cpu上下文切换就好比一个人有好多朋友要拜访，有的朋友房子大（进程），进进出出里三层外三层，有的朋友住帐篷（线程），就拉开帐篷聊聊天，有的朋友就隔着窗户说两句话打个照面路过（中断）</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 😊很棒的比喻</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-27 14:38:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/4a/2c/f8451d77.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>石维康</span>
  </div>
  <div class="_2_QraFYR_0">课程内容中说&quot;用户空间（Ring 3）只能访问受限资源，不能直接访问内存等..&quot;，其实在用户态，还是可以直接访问属于用户态的内存的吧</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-26 09:34:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/12/b3/9e8f4b4f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>MoFanDon</span>
  </div>
  <div class="_2_QraFYR_0">以前的确是没怎么注意 CPU上下文切换，进程上下文切换 是不同的。学习了。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-26 01:00:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/54/49/ede4ab2f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>地心引力</span>
  </div>
  <div class="_2_QraFYR_0">进程调用系统调用，从用户态切换到内核态需要保存进程的上下文，而中断发生时，中断处理程序也是在内核态运行，但此时为什么就不需要保存进程的上下文呢，这种差别的本质原因是什么，望老师指教</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不是不保存进程上下文，而是不需要刷新虚拟内存这些用户空间的数据，但具体到内核中的状态肯定还是要刷新的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-29 20:46:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/06/81/28418795.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>衣申人</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，有三个问题不是很理解：<br><br>1.系统调用的过程是上下文切换的话，那么函数调用是不是也可以叫上下文切换？<br>2.进程上下文切换，还需要刷新虚拟内存，是什么意思呢？上下文切换应该是把进程在cpu中的状态信息保存到内存，然后读取另一个进程内存中的状态信息到cpu，为什么涉及虚拟内存的刷新？还会影响到TLB？<br>3.CPU时间片是固定大小的吗？如果是，那么如果每个进程都正常获得时间片和到期释放，那么不管进程数量多少，是不是单位时间内的上下文切换次数是一定的？<br><br>盼望老师解答，谢谢。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-26 19:56:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/86/fa/4bcd7365.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>玉剑冰锋</span>
  </div>
  <div class="_2_QraFYR_0">Ring1和Ring2代表啥意思？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-30 08:12:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/93/43/0e84492d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Maxwell</span>
  </div>
  <div class="_2_QraFYR_0">上下文切换多高算高呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-26 09:51:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/a4/39/6a5cd1d8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>sotey</span>
  </div>
  <div class="_2_QraFYR_0">交作业: 老师，在我的理解，CPU上下文是指程序执行的指令和环境变量和内存位置，上下文切换是CPU把当前正在执行的程序上下文从CPU内容(寄存器和计数器)保存到内存，内存的分层结构这里暂不关注(因为我也一知半解)，再把需要执行的程序从内存里保存的上下文加载到CPU内(寄存器)交给CPU执行。进程和线程以及中断其实大致过程是一样的，由于离内核的距离不同，需要切换的内容量不同，因此切换效率是不同的。以上为今天的作业，请老师审阅。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-26 09:06:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/f3/3a/63cf1d0e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>设置昵称</span>
  </div>
  <div class="_2_QraFYR_0">本文的核心观点<br><br>任务的调度导致的上下文切换并不是免费的，其需要保存和恢复操作，而这些保存和恢复操作都需要内核在CPU上运行才能够完成，因此我们就需要分析当系统负载高时，上下文切换的贡献有多大？<br><br>下面将对上一句话进行解析<br>1. 任务<br>任务广泛的说是执行某些功能的程序，分为进程，线程，中断程序<br>2.任务的调度（上下文切换）<br>任务的调度主要是执行保存和恢复操作，针对任务的不同，其保存和恢复的对象又略有不同。<br>a) 任务是系统调用<br>当任务是系统调用时，其本质是同一个进程从用户态转换到内核态的过程，因此，该上下文切换仅设计到内核空间的状态，如核堆栈，寄存器。<br>b) 当任务是进程<br>当任务是进程时，其时从一个进程到另一个进程的切换。因此其不仅要保存内核空间状态，还需要把偶才能用户空间的状态，如全局变量，虚拟内存等<br>c) 当任务是线程<br>只需要弄清线程和进程的区别即可<br>线程是调度的基本单位，进程是资源拥有的基本单位。说白了，所谓内核中的任务调度，实际上的调度对象是线程；而进程只是给线程提供了虚拟内存、全局变量等资源。<br>d) 当任务是中断<br>见原文<br><br>----------------------------------------------------------------------------------------------------------<br>上下文的切换是由内核来决定的，主要分为主动切换和被动切换<br>主动切换：进程主动释放CPU资源，主要包含以下几种情况。<br>a) 程序本身，如sleep函数，<br>b) 进程需要等待一些其他资源如io，才能继续进行下去<br>c) ...<br>被动切换：由内核强制切换进程，主要包括以下几种情况。<br>a) 时间片调度时，时间片的时间到了<br>b) 出现了一个中断<br>c) 出来高优先级进程<br>d）...</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-12 08:55:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/e7/e0/33521e13.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>DigDeeply</span>
  </div>
  <div class="_2_QraFYR_0">请老师帮忙解答一个疑问，在 Kubernetes 编排的这种虚拟化技术中，当给某个容器限定只有一个 CPU 时，容器内的镜像是拥有 24 CPU 的，那么在这种情况下，业务内是当成只存在一个 CPU 使用，还是 拥有 24 个，但每个 CPU 的能力只有一个正常CPU的 1&#47;24 ？不同的使用方式，应该会影响到 CPU 的上下文切换。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-26 12:53:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/fc/fc/1e235814.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>耿长学</span>
  </div>
  <div class="_2_QraFYR_0">老师，推荐基本性能相关的书籍吧</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-26 07:21:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/5b/d2/34a8c79c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Juinn</span>
  </div>
  <div class="_2_QraFYR_0">ring1和ring2是干嘛的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 在Linux中没用到</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-23 10:25:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/49/5d/fd0c908c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>王晓冉</span>
  </div>
  <div class="_2_QraFYR_0">内核堆栈能详细介绍下吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-27 15:31:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/85/3a/bb885be4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>glinuxer</span>
  </div>
  <div class="_2_QraFYR_0">“其一，为了保证所有进程可以得到公平调度，CPU 时间被划分为一段段的时间片，这些时间片再被轮流分配给各个进程。”。<br>CFS 调度器没有时间片的概念了，叫做虚拟运行时间。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍感谢指出</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-27 05:56:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/3c/93/88febcb3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Bugsy</span>
  </div>
  <div class="_2_QraFYR_0">因为内核态是cpu的一种特权模式，在这种模式下，内核可以访问系统资源（包括内存、cpu和其他IO设备），所以cpu上下文切换（包括进程上下文切换、线程上下文切换和中断切换）均是发生在内核态。<br>对于应用程序，在最开始都是运行在用户态，当程序中需要调用或者使用系统资源时（包括内存、cpu和其他io设备），由于受到安全性限制，他们只能能通过系统调用来运行部分内核的代码实现对系统资源的调用使用，这个过程就是从用户态陷入内核态的过程，在系统调用过程中一直是在同一个进程中实现。简短总结下，老师有没有其他资料可以推荐学习下。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍 后续推荐一些书籍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-26 19:37:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/ff/fb/2f99df3e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>蔡泳</span>
  </div>
  <div class="_2_QraFYR_0">hello，对您这句话有些许不理解<br>&#39;因此，进程的上下文切换就比系统调用时多了一步：在保存当前进程的内核状态和 CPU 寄存器之前，需要先把该进程的虚拟内存、栈等保存下来；而加载了下一进程的内核态后，还需要刷新进程的虚拟内存和用户栈。&#39;<br>不理解的地方有：<br>1. 进程上下文切换前，需要先保存虚拟内存，具体而言这个&#39;虚拟内存&#39;在内存中的如何表示的？<br>2. 虚拟内存是如何被保存，恢复的？<br><br>我有个人的理解，但不知是否正确，如下：<br>1. 内核维护了一个进程id到虚拟内存的表，通过进程id可以查询当前进程对应的虚拟内存，通过虚拟内存又可以对应到物理内存地址上<br>2. 进程运行过程中会对上面虚拟内存进行修改，比如内存映射新增了某段虚拟内存到物理内存的映射，所以进程切换需要对这些修改进行保存？具体而言怎么保存的不清楚了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-26 15:22:45</div>
  </div>
</div>
</div>
</li>
</ul>