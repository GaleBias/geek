<audio title="32｜并发：聊聊Goroutine调度器的原理" src="https://static001.geekbang.org/resource/audio/eb/65/eb6fc98f283176142c92a32aceeb0065.mp3" controls="controls"></audio> 
<p>你好，我是Tony Bai。</p><p>上一讲我们学习了并发的基本概念和Go的并发方案，也就是Goroutine的一些基本使用和注意事项。对于大多数Gopher来说，这些内容作为Go并发入门已经是足够了。</p><p>但毕竟Go没有采用基于线程的并发模型，可能很多Gopher都好奇Go运行时究竟是如何将一个个Goroutine调度到CPU上执行的。当然，Goroutine的调度本来是Go语言核心开发团队才应该关注的事情，大多数Gopher们无需关心。但就我个人的学习和实践经验而言，我觉得了解Goroutine的调度模型和原理，能够帮助我们编写出更高质量的Go代码。</p><p>因此，在这一讲中，我想和你一起简单探究一下Goroutine调度器的原理和演化历史。</p><h2>Goroutine调度器</h2><p>提到“调度”，我们首先想到的就是操作系统对进程、线程的调度。操作系统调度器会将系统中的多个线程按照一定算法调度到物理CPU上去运行。</p><p>前面我们也提到，传统的编程语言，比如C、C++等的并发实现，多是基于线程模型的，也就是应用程序负责创建线程（一般通过libpthread等库函数调用实现），操作系统负责调度线程。当然，我们也说过，这种传统支持并发的方式有很多不足。为了解决这些问题，Go语言中的并发实现，使用了Goroutine，代替了操作系统的线程，也不再依靠操作系统调度。</p><!-- [[[read_end]]] --><p>Goroutine占用的资源非常小，上节课我们也说过，每个Goroutine栈的大小默认是2KB。而且，Goroutine调度的切换也不用陷入（trap）操作系统内核层完成，代价很低。因此，一个Go程序中可以创建成千上万个并发的Goroutine。而将这些Goroutine按照一定算法放到“CPU”上执行的程序，就被称为<strong>Goroutine调度器（Goroutine Scheduler）</strong>，注意，这里说的“CPU”打了引号。</p><p>不过，一个Go程序对于操作系统来说只是一个<strong>用户层程序</strong>，操作系统眼中只有线程，它甚至不知道有一种叫<strong>Goroutine</strong>的事物存在。所以，Goroutine的调度全要靠Go自己完成。那么，实现Go程序内Goroutine之间“公平”竞争“CPU”资源的任务，就落到了Go运行时（runtime）头上了。要知道在一个Go程序中，除了用户层代码，剩下的就是Go运行时了。</p><p>于是，Goroutine的调度问题就演变为，Go运行时如何将程序内的众多Goroutine，按照一定算法调度到“CPU”资源上运行的问题了。</p><p><strong>可是，在操作系统层面，线程竞争的“CPU”资源是真实的物理CPU，但在Go程序层面，各个Goroutine要竞争的“CPU”资源又是什么呢？</strong></p><p>Go程序是用户层程序，它本身就是整体运行在一个或多个操作系统线程上的。所以这个答案就出来了：Goroutine们要竞争的“CPU”资源就是操作系统线程。这样，Goroutine调度器的任务也就明确了：<strong>将Goroutine按照一定算法放到不同的操作系统线程中去执行</strong>。</p><p>那么，Goroutine调度器究竟是以怎样的算法模型，将Goroutine调度到不同的操作系统线程上去的呢？我们继续向下看。</p><h2>Goroutine调度器模型与演化过程</h2><p>Goroutine调度器的实现不是一蹴而就的，它的调度模型与算法也是几经演化，从最初的G-M模型、到G-P-M模型，从不支持抢占，到支持协作式抢占，再到支持基于信号的异步抢占，Goroutine调度器经历了不断地优化与打磨。</p><p>首先我们来看最初的<strong>G-M模型</strong>。</p><p>2012年3月28日，Go 1.0正式发布。在这个版本中，Go开发团队实现了一个简单的Goroutine调度器。在这个调度器中，每个Goroutine对应于运行时中的一个抽象结构：<code>G(Goroutine)</code> ，</p><p>而被视作“物理CPU”的操作系统线程，则被抽象为另外一个结构：<code>M(machine)</code>。</p><p>调度器的工作就是将G调度到M上去运行。为了更好地控制程序中活跃的M的数量，调度器引入了GOMAXPROCS变量来表示Go调度器可见的“处理器”的最大数量。</p><p>这个模型实现起来比较简单，也能正常工作，但是却存在着诸多问题。前英特尔黑带级工程师、现谷歌工程师<a href="https://github.com/dvyukov">德米特里·维尤科夫（Dmitry Vyukov</a>）在其《<a href="https://docs.google.com/document/d/1TTj4T2JO42uD5ID9e89oa0sLKhJYD0Y_kqxDv3I3XMw/edit#!">Scalable Go Scheduler Design</a>》一文中指出了<strong>G-M模型</strong>的一个重要不足：限制了Go并发程序的伸缩性，尤其是对那些有高吞吐或并行计算需求的服务程序。</p><p>这个问题主要体现在这几个方面：</p><ul>
<li>单一全局互斥锁<code>(Sched.Lock)</code> 和集中状态存储的存在，导致所有Goroutine相关操作，比如创建、重新调度等，都要上锁；</li>
<li>Goroutine传递问题：M经常在M之间传递“可运行”的Goroutine，这导致调度延迟增大，也增加了额外的性能损耗；</li>
<li>每个M都做内存缓存，导致内存占用过高，数据局部性较差；</li>
<li>由于系统调用（syscall）而形成的频繁的工作线程阻塞和解除阻塞，导致额外的性能损耗。</li>
</ul><p>为了解决这些问题，德米特里·维尤科夫又亲自操刀改进了Go调度器，在Go 1.1版本中实现了<strong>G-P-M调度模型</strong>和<a href="http://supertech.csail.mit.edu/papers/steal.pdf">work stealing算法</a>，这个模型一直沿用至今。模型如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/43/a8/43ffdbc6b2203d9400ac98423192caa8.png?wh=1224x1142" alt="图片"></p><p>有人说过：<strong>“计算机科学领域的任何问题都可以通过增加一个间接的中间层来解决”</strong>，德米特里·维尤科夫的<strong>G-P-M</strong>模型恰是这一理论的践行者。你可以看到，德米特里·维尤科夫通过向G-M模型中增加了一个P，让Go调度器具有很好的伸缩性。</p><p>P是一个“逻辑Proccessor”，每个G（Goroutine）要想真正运行起来，首先需要被分配一个P，也就是进入到P的本地运行队列（local runq）中。对于G来说，P就是运行它的“CPU”，可以说：<strong>在G的眼里只有P</strong>。但从Go调度器的视角来看，真正的“CPU”是M，只有将P和M绑定，才能让P的runq中的G真正运行起来。</p><p>G-P-M模型的实现算是<code>Go调度器</code>的一大进步，但调度器仍然有一个令人头疼的问题，那就是<strong>不支持抢占式调度</strong>，这导致一旦某个G中出现死循环的代码逻辑，那么G将永久占用分配给它的P和M，而位于同一个P中的其他G将得不到调度，出现“<strong>饿死</strong>”的情况。</p><p>更为严重的是，当只有一个P（GOMAXPROCS=1）时，整个Go程序中的其他G都将“饿死”。于是德米特里·维尤科夫又提出了《<a href="https://docs.google.com/document/d/1ETuA2IOmnaQ4j81AtTGT40Y4_Jr6_IDASEKg0t0dBR8/edit#!">Go Preemptive Scheduler Design</a>》并在<strong>Go 1.2中实现了基于协作的“抢占式”调度</strong>。</p><p>这个抢占式调度的原理就是，Go编译器在每个函数或方法的入口处加上了一段额外的代码(runtime.morestack_noctxt)，让运行时有机会在这段代码中检查是否需要执行抢占调度。</p><p>这种解决方案只能说局部解决了“饿死”问题，只在有函数调用的地方才能插入“抢占”代码（埋点），对于没有函数调用而是纯算法循环计算的G，Go调度器依然无法抢占。</p><p>比如，死循环等并没有给编译器插入抢占代码的机会，这就会导致GC在等待所有Goroutine停止时的等待时间过长，从而<a href="https://github.com/golang/go/issues/10958">导致GC延迟</a>，内存占用瞬间冲高；甚至在一些特殊情况下，导致在STW（stop the world）时死锁。</p><p>为了解决这些问题，Go在1.14版本中接受了奥斯汀·克莱门茨（Austin Clements）的<a href="https://go.googlesource.com/proposal/+/master/design/24543-non-cooperative-preemption.md">提案</a>，增加了<strong>对非协作的抢占式调度的支持</strong>，这种抢占式调度是基于系统信号的，也就是通过向线程发送信号的方式来抢占正在运行的Goroutine。</p><p>除了这些大的迭代外，Goroutine的调度器还有一些小的优化改动，比如<strong>通过文件I/O poller减少M的阻塞等</strong>。</p><p>Go运行时已经实现了netpoller，这使得即便G发起网络I/O操作，也不会导致M被阻塞（仅阻塞G），也就不会导致大量线程（M）被创建出来。</p><p>但是对于文件I/O操作来说，一旦阻塞，那么线程（M）将进入挂起状态，等待I/O返回后被唤醒。这种情况下P将与挂起的M分离，再选择一个处于空闲状态（idle）的M。如果此时没有空闲的M，就会新创建一个M（线程），所以，这种情况下，大量I/O操作仍然会导致大量线程被创建。</p><p>为了解决这个问题，Go开发团队的伊恩·兰斯·泰勒（Ian Lance Taylor）在Go 1.9中增加了一个<a href="https://groups.google.com/forum/#!topic/golang-dev/tT8SoKfHty0">针对文件I/O的Poller</a>的功能，这个功能可以像netpoller那样，在G操作那些支持监听（pollable）的文件描述符时，仅会阻塞G，而不会阻塞M。不过这个功能依然不能对常规文件有效，常规文件是不支持监听的（pollable）。但对于Go调度器而言，这也算是一个不小的进步了。</p><p>从Go 1.2以后，Go调度器就一直稳定在G-P-M调度模型上，尽管有各种优化和改进，但也都是基于这个模型之上的。那未来的Go调度器会往哪方面发展呢？德米特里·维尤科夫在2014年9月提出了一个新的设计草案文档：《<a href="https://docs.google.com/document/u/0/d/1d3iI2QWURgDIsSR6G2275vMeQ_X7w-qxM2Vp7iGwwuM/pub">NUMA‐aware scheduler for Go</a>》，作为对未来Goroutine调度器演进方向的一个提议，不过至今似乎这个提议也没有列入开发计划。</p><p>通过前面对Goroutine调度器演化的分析，你可以看到，目前G-M模型已经废弃，NUMA调度模型尚未实现，那么现在我们要理解如今的Goroutine调度，只需要学习G-P-M模型就可以了，接下来我们就来看看G-P-M模型下Goroutine的调度原理。</p><h2>深入G-P-M模型</h2><p>Go语言中Goroutine的调度、GC、内存管理等是Go语言原理最复杂、最难懂的地方，随便拿出一个都可以讲上几节课，并且这三方面的内容随着Go版本的演进也在不断更新。因为我们是入门课，所以这里我就只基于Go 1.12.7版本（支持基于协作的抢占式调度）给你粗略介绍一下基于G-P-M模型的调度原理，如果你还对这方面感兴趣，可以基于这些介绍深入到相关的Go源码中去，深入挖掘细节。</p><h3>G、P和M</h3><p>关于G、P、M的定义，我们可以参见<code>$GOROOT/src/runtime/runtime2.go</code>这个源文件。你可以看到，G、P、M这三个结构体定义都是大块头，每个结构体定义都包含十几个甚至二三十个字段。更不用说，像调度器这样的核心代码向来很复杂，考虑的因素也非常多，代码“耦合”成一坨。不过从复杂的代码中，我们依然可以看出来G、P、M的各自的大致用途，我们这里简要说明一下：</p><ul>
<li>G:  代表Goroutine，存储了Goroutine的执行栈信息、Goroutine状态以及Goroutine的任务函数等，而且G对象是可以重用的；</li>
<li>P:  代表逻辑processor，P的数量决定了系统内最大可并行的G的数量，P的最大作用还是其拥有的各种G对象队列、链表、一些缓存和状态；</li>
<li>M:  M代表着真正的执行计算资源。在绑定有效的P后，进入一个调度循环，而调度循环的机制大致是<strong>从P的本地运行队列以及全局队列中获取G，切换到G的执行栈上并执行G的函数，调用goexit做清理工作并回到M，如此反复</strong>。M并不保留G状态，这是G可以跨M调度的基础。</li>
</ul><p>我这里也给出了G、P、M定义的代码片段（注意：我们这里使用的是Go 1.12.7版本，随着Go演化，结构体中的字段定义可能会有不同），你也可以看一看：</p><pre><code class="language-plain">//src/runtime/runtime2.go
type g struct {
    stack      stack   // offset known to runtime/cgo
    sched      gobuf
    goid       int64
    gopc       uintptr // pc of go statement that created this goroutine
    startpc    uintptr // pc of goroutine function
    ... ...
}

type p struct {
    lock mutex

    id          int32
    status      uint32 // one of pidle/prunning/...
  
    mcache      *mcache
    racectx     uintptr

    // Queue of runnable goroutines. Accessed without lock.
    runqhead uint32
    runqtail uint32
    runq     [256]guintptr

    runnext guintptr

    // Available G's (status == Gdead)
    gfree    *g
    gfreecnt int32

    ... ...
}



type m struct {
    g0            *g     // goroutine with scheduling stack
    mstartfn      func()
    curg          *g     // current running goroutine
    ... ...
}
</code></pre><p>而Goroutine调度器的目标，就是公平合理地将各个G调度到P上“运行”，下面我们重点看看G是如何被调度的。</p><h3>G被抢占调度</h3><p>我们先来说常规情况，也就是如果某个G没有进行系统调用（syscall）、没有进行I/O操作、没有阻塞在一个channel操作上，<strong>调度器是如何让G停下来并调度下一个可运行的G的呢</strong>？</p><p>答案就是：<strong>G是被抢占调度的</strong>。</p><p>前面说过，除非极端的无限循环，否则只要G调用函数，Go运行时就有了抢占G的机会。Go程序启动时，运行时会去启动一个名为sysmon的M（一般称为监控线程），这个M的特殊之处在于它不需要绑定P就可以运行（以g0这个G的形式），这个M在整个Go程序的运行过程中至关重要，你可以看下我对sysmon被创建的部分代码以及sysmon的执行逻辑摘录：</p><pre><code class="language-plain">//$GOROOT/src/runtime/proc.go

// The main goroutine.
func main() {
     ... ...
    systemstack(func() {
        newm(sysmon, nil)
    })
    .... ...
}

// Always runs without a P, so write barriers are not allowed.
//
//go:nowritebarrierrec
func sysmon() {
    // If a heap span goes unused for 5 minutes after a garbage collection,
    // we hand it back to the operating system.
    scavengelimit := int64(5 * 60 * 1e9)
    ... ...

    if  .... {
        ... ...
        // retake P's blocked in syscalls
        // and preempt long running G's
        if retake(now) != 0 {
            idle = 0
        } else {
            idle++
        }
       ... ...
    }
}
</code></pre><p>我们看到，sysmon每20us~10ms启动一次，sysmon主要完成了这些工作：</p><ul>
<li>释放闲置超过5分钟的span内存；</li>
<li>如果超过2分钟没有垃圾回收，强制执行；</li>
<li>将长时间未处理的netpoll结果添加到任务队列；</li>
<li>向长时间运行的G任务发出抢占调度；</li>
<li>收回因syscall长时间阻塞的P；</li>
</ul><p>我们看到sysmon将“向长时间运行的G任务发出抢占调度”，这个事情由函数<code>retake</code>实施：</p><pre><code class="language-plain">// $GOROOT/src/runtime/proc.go

// forcePreemptNS is the time slice given to a G before it is
// preempted.
const forcePreemptNS = 10 * 1000 * 1000 // 10ms

func retake(now int64) uint32 {
          ... ...
           // Preempt G if it's running for too long.
            t := int64(_p_.schedtick)
            if int64(pd.schedtick) != t {
                pd.schedtick = uint32(t)
                pd.schedwhen = now
                continue
            }
            if pd.schedwhen+forcePreemptNS &gt; now {
                continue
            }
            preemptone(_p_)
         ... ...
}

func preemptone(_p_ *p) bool {
    mp := _p_.m.ptr()
    if mp == nil || mp == getg().m {
        return false
    }
    gp := mp.curg
    if gp == nil || gp == mp.g0 {
        return false
    }

    gp.preempt = true //设置被抢占标志

    // Every call in a go routine checks for stack overflow by
    // comparing the current stack pointer to gp-&gt;stackguard0.
    // Setting gp-&gt;stackguard0 to StackPreempt folds
    // preemption into the normal stack overflow check.
    gp.stackguard0 = stackPreempt
    return true
}
</code></pre><p>从上面的代码中，我们可以看出，<strong>如果一个G任务运行10ms，sysmon就会认为它的运行时间太久而发出抢占式调度的请求</strong>。一旦G的抢占标志位被设为true，那么等到这个G下一次调用函数或方法时，运行时就可以将G抢占并移出运行状态，放入队列中，等待下一次被调度。</p><p>不过，除了这个常规调度之外，还有两个特殊情况下G的调度方法。</p><p><strong>第一种：channel阻塞或网络I/O情况下的调度。</strong></p><p>如果G被阻塞在某个channel操作或网络I/O操作上时，G会被放置到某个等待（wait）队列中，而M会尝试运行P的下一个可运行的G。如果这个时候P没有可运行的G供M运行，那么M将解绑P，并进入挂起状态。当I/O操作完成或channel操作完成，在等待队列中的G会被唤醒，标记为可运行（runnable），并被放入到某P的队列中，绑定一个M后继续执行。</p><p><strong>第二种：系统调用阻塞情况下的调度。</strong></p><p>如果G被阻塞在某个系统调用（system call）上，那么不光G会阻塞，执行这个G的M也会解绑P，与G一起进入挂起状态。如果此时有空闲的M，那么P就会和它绑定，并继续执行其他G；如果没有空闲的M，但仍然有其他G要去执行，那么Go运行时就会创建一个新M（线程）。</p><p>当系统调用返回后，阻塞在这个系统调用上的G会尝试获取一个可用的P，如果没有可用的P，那么G会被标记为runnable，之前的那个挂起的M将再次进入挂起状态。</p><h2>小结</h2><p>好了，今天的课讲到这里就结束了，现在我们一起来回顾一下吧。</p><p>基于Goroutine的并发设计离不开一个高效的生产级调度器。Goroutine调度器演进了十余年，先后经历了G-M模型、G-P-M模型和work stealing算法、协作式的抢占调度以及基于信号的异步抢占等改进与优化，目前Goroutine调度器相对稳定和成熟，可以适合绝大部分生产场合。</p><p>现在的G-P-M模型和最初的G-M模型相比，通过向G-M模型中增加了一个代表逻辑处理器的P，使得Goroutine调度器具有了更好的伸缩性。</p><p>M是Go代码运行的真实载体，包括Goroutine调度器自身的逻辑也是在M中运行的。</p><p>P在G-P-M模型中占据核心地位，它拥有待调度的G的队列，同时M要想运行G必须绑定一个P。一个G被调度执行的时间不能过长，超过特定长的时间后，G会被设置为<strong>可抢占</strong>，并在下一次执行函数或方法时被Go运行时移出运行状态。</p><p>如果G被阻塞在某个channel操作或网络I/O操作上时，M可以不被阻塞，这避免了大量创建M导致的开销。但如果G因慢系统调用而阻塞，那么M也会一起阻塞，但在阻塞前会与P解绑，P会尝试与其他M绑定继续运行其他G。但若没有现成的M，Go运行时会建立新的M，这也是系统调用可能导致系统线程数量增加的原因，你一定要注意这一点。</p><h2>思考题</h2><p>为了让你更好理解Goroutine调度原理，我这里留个思考题。请看下面代码：</p><pre><code class="language-plain">func deadloop() {
    for {
    } 
}

func main() {
    go deadloop()
    for {
        time.Sleep(time.Second * 1)
        fmt.Println("I got scheduled!")
    }
}
</code></pre><p>我的问题是：</p><ol>
<li>
<p>在一个拥有多核处理器的主机上，使用Go 1.13.x版本运行这个示例代码，你在命令行终端上是否能看到“I got scheduled!”输出呢？也就是main goroutine在创建deadloop goroutine之后是否能继续得到调度呢？</p>
</li>
<li>
<p>我们通过什么方法可以让上面示例中的main goroutine，在创建deadloop goroutine之后无法继续得到调度？</p>
</li>
</ol><p>欢迎你把这节课分享给更多对Gouroutine调度原理感兴趣的朋友。我是Tony Bai，我们下节课见。</p>
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
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_cca544</span>
  </div>
  <div class="_2_QraFYR_0">go1.13的话加上runtime.GOMAXPROCS(1) main goroutine在创建 deadloop goroutine 之后就无法继续得到调度<br>但如果是go1.14之后的话即使加上runtime.GOMAXPROCS(1) main goroutine在创建 deadloop goroutine 之后还是可以得到调度，应该是因为增加了对非协作的抢占式调度的支持</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: ✅</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-11 00:19:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/9d/a4/e481ae48.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lesserror</span>
  </div>
  <div class="_2_QraFYR_0">大白老师这篇算是让我重新对Go的G、P、M模型有了一个新的认识。感谢。不过还是有几处疑惑的地方：<br><br>1. 怎么理解文中的：“集中状态存储”和“数据局部性较差”，能再进一步解释一下么？<br><br>2. 编译器在每个函数或方法的入口处加上了一段额外的代码 (runtime.morestack_noctxt)，括号中的：runtime.morestack_noctxt 这是一个文件么？<br><br>3. 怎么理解“协作式”、“非协作式”呢？看了文章还是没太明白。<br><br>4. 关于挂起，百度的说法大概是：“暂时被淘汰出内存的进程。”，这里该怎么理解呢？<br><br>5. 为什么 M有时必须要与 G 一起挂起？M 不是可以不保存 G的状态的吗？M不能直接去绑定别的p吗？为什么要频繁的挂起呢？<br><br>PS：老师的文档链接好评，最原始的出处标准的很明确。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1.按照Dmitry Vyukov原文的意思： 集中状态(centralized state)，我理解就是一把全局锁要保护的数据太多。这样无论访问哪个数据，都要锁这把全局锁。数据局部性差是因为每个m都会缓存它执行的代码或数据，但是如果在m之间频繁传递goroutine，那么这种局部缓存的意义就没有了。无法实现局部缓存带来的性能提升。<br><br>2. runtime.morestack_noctxt 是一个函数。<br>3. 协作式：大家都按事先定义好的规则来，比如：一个goroutine执行完后，退出，让出p，然后下一个goroutine被调度到p上运行。这样做的缺点就在于 是否让出p的决定权在groutine自身。一旦某个g不主动让出p或执行时间较长，那么后面的goroutine只能等着，没有方法让前者让出p，导致延迟甚至饿死。而非协作: 就是由runtime来决定一个goroutine运行多长时间，如果你不主动让出，对不起，我有手段可以抢占你，把你踢出去，让后面的goroutine进来运行。<br><br>4. 挂起这里你可以理解为暂停运行，等待下一次调度。<br>5. 当进行一些慢系统调用的时候，比如常规文件io，执行系统调用的m就要和g一起挂起，这是os的要求，不是go runtime的要求。毕竟真正执行代码的还是m。<br> </p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-13 21:57:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJ8ic8eLTo5rnqIMJicUfpkVBrOUJAW4fANicKIbHdC54O9SOdwSoeK6o8icibaUbh7ZUXAkGF9zwHqo0Q/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ivhong</span>
  </div>
  <div class="_2_QraFYR_0">谢谢大白老师，能把这么晦涩的原理讲的这么清楚。我反复看了很多遍，做了如下总结，不知道是不是合理，希望老师闲暇之余给予指正，🙏。<br>在文中提及的GPM，以及GPM之前的相互配合，完成了go的并发功能。<br><br>G：可以看作关键字go 后面跟着的可执行代码块（即goroutine），当然包含这个代码块的一些本身的信息（比如栈信息、状态等等一些必要的属性）。另外存在一个G的全局队列，只要是需要执行的 goroutine 都会被放倒这个全局队列里。<br><br>P：可以看作逻辑上的“任务处理器”，go有多个这个处理器，具体的数量由runtime.GOMAXPROCS(1)指定，它有下面的指责：<br>    1. 它有自己G队列。当他发现自己的队列为空时，可以去全局G队列里获取等待执行的goroutine，甚至可以去其他的P的队列里抢用goroutine。他把拿过来的goroutine放到自己的队列里<br>    2. 他可以找到一个空闲的M与自己绑定，用来运行自己队列中的goroutine，如果没有空闲的M，则创建一个M来绑定<br>    3. 被P绑定的M，可以自己主动的与P解绑，当P发现自己的M被解绑，就执行2<br>    4. 如果自己队列中没有goroutine，也无法从“外面”获取goroutine，则与M解绑（解绑M时，是按什么逻辑选择挂起M或者释放M呢？）<br><br>M：物理的处理器，具体执行goroutine的系统线程，所有goroutine都是在M中执行的，它被P创建，与P绑定后可执行P队列中的goroutine，在执行goroutine会处理3中情况保证并发是顺利的（不会发生“饿死”，资源分配不平等的情况）<br>    1. 当G长时间运行时，可以被Go暂停执行而被移出M（是不是放到全局G队列呢？），等待下次运行（即抢占式调度）。<br>    2. 如果G发生了channel 等待或者 网络I&#47;O请求，则把G放到某个等待队列中，M继续执行下一goroutine，当G等待的结果返回时，会唤醒“G”，并把它放入到全局G的队列中，等待P的获取（这里不知道理解的对不对？）。<br>    3. 如果G产生了系统调用，则M与P解绑，然后M和它正执行的G被操作系统“挂起”等待操作系统的中断返回（对于操作系统而言，M和G是一回事；而对于GO来说，G只有能在M中运行，只有运行才触发发系统调用）。<br><br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-15 11:28:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/27/d8/39/acb7e16e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>麦芒极客</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，G遇到网络IO阻塞时，真正的线程即M不应该也阻塞吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 好问题！不过当网络I&#47;O阻塞时，M真不会阻塞。<br><br>因为runtime层实现了netpoller，netpoller是基于os提供的I&#47;O多路复用模型实现的(比如:linux上的epoll)，这允许一个线程处理多个socket。这就使得当某个goroutine的socket发送i&#47;o阻塞时，仅会让goroutine变为阻塞状态，runtime会将对应的socket与goroutine加入到netpoller的监听中，然后M继续执行其他goroutine的任务。等netpoller监视到之前的socket可读&#47;可写时，再把对应的goroutine唤醒继续执行网络i&#47;o。<br><br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-22 18:40:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/ba/ce/fd45714f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>bearlu</span>
  </div>
  <div class="_2_QraFYR_0">老师，想学习goroutine调度器，演进的关键版本，依次是go的什么发行版？还有什么相关资料书籍？谢谢老师</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 欧长坤老师维护的这个go history中有关调度器的演化可以参考：https:&#47;&#47;golang.design&#47;history&#47;#scheduler</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-10 09:08:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/53/a8/abc96f70.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>return</span>
  </div>
  <div class="_2_QraFYR_0">老师讲的太好了，干货慢慢，老师已经把流程和原理讲清楚了，<br>原理下的 更底层细节 得自己再研究学习。<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-11 10:45:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/7hOJBKZVrJD8ayZoYnq0ECFsoWf9yx6eyq3JU8p1449A6cKEp5rSpicJRg5vH3HZgq5YwbpGicBvyARFutcPLGgw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Mr_D</span>
  </div>
  <div class="_2_QraFYR_0">请问G的可重用具体指什么呢？是针对已经运行结束的G留下的内存空间，进行相关数据的重填么</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对，goroutine退出后，runtime层的G对象会被放入一个Free G对象的池子中，后续有新Goroutine创建时，优先从池子里get一个。可以看Go源码：runtime&#47;proc.go中的代码。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-11 17:37:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/54/9a/76c0af70.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>每天晒白牙</span>
  </div>
  <div class="_2_QraFYR_0">思考题<br>1.在一个拥有多核处理器的主机上，使用 Go 1.13.x 版本运行这个示例代码，你在命令行终端上是否能看到“I got scheduled!”输出呢？也就是 main goroutine 在创建 deadloop goroutine 之后是否能继续得到调度呢？<br>答：不会得到调度，首先 Go 是在 1.2 版本增加的抢占式调度，1.13.x版本还不支持，因为deadloop goroutine 是死循环，所以这个G会永远占用分配的P和M<br><br>2.我们通过什么方法可以让上面示例中的 main goroutine，在创建 deadloop goroutine 之后无法继续得到调度？<br>两个必要条件<br>①go的版本要在1.2之前，因为1.2之前不支持抢占式调度，可以确保G可以占用分配给它的P和M<br>②设置GOMAXPROCS=1<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: “Go 是在 1.2 版本增加的抢占式调度，1.13.x版本还不支持” ， 1.2版本？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-19 17:50:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/64/8a/bc8cb43c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>路边的猪</span>
  </div>
  <div class="_2_QraFYR_0">第一种，因为网络io导致阻塞的处理方式这里。我想问，网络io势必会引起系统调用，比如最基础的建立tcp连接这些，那这块儿是咋区分系统调用和网络io的呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 由于go运行时netpoller的存在，我们很难精确区分。这种“网络io”会被runtime转换为goroutine的调度等，不一定真的实施网络io。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-05 23:58:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/0c/46/dfe32cf4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>多选参数</span>
  </div>
  <div class="_2_QraFYR_0">有几个问题想问老师： <br>1. 文章中提到的 Go 运行时这块更多是指哪些内容？Go 调度器应该也是 Go 运行时中的部分内容吧？就是 Go 开源的源代码吗？<br>2. 有关 Go 运行时比较好的资料推荐吗？比如 Go 运行时的架构、Go 运行时是怎么和用户编写的代码合在一起的，因为我理解的 Go 运行时不像 JVM 那样，Go 是编译性的，也就是说会将用户编写的代码和 Go 运行时代码一起编译成二进制，然后运行是从 Go 运行时开始的，然后在从用户逻辑的代码开始。<br><br>老师有空的时候麻烦解答下，谢谢老师～</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 《Go语言第一课FAQ》中有这方面的解答，你可以先看看。 https:&#47;&#47;tonybai.com&#47;go-course-faq </p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-29 20:55:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/f1/de/3ebcbb3f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>DullBird</span>
  </div>
  <div class="_2_QraFYR_0">小结上面一句：之前的那个挂起的 M 将再次进入挂起状态。   最后是不是运行状态吧？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 文里有说，是在系统调用返回后，找不到idle的P的情况下，m会进入到idle m池中挂起。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-11 09:50:46</div>
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
  <div class="_2_QraFYR_0">思考题：<br>是不是不管怎么处理，main goroutine  都会被调度？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 试试将P的数量置为1.</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-10 17:57:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2e/8c/f9/e1dab0ca.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>怎么睡才能做这种梦</span>
  </div>
  <div class="_2_QraFYR_0">老师，我想问一下，Go语言有了GPM模型，还有线程池存在的必要吗？<br>还是说可能存在特殊情况，比如说当有系统调用发生阻塞的情况下，线程池才有作用，因为只有系统调用发生阻塞才可能创建线程。或者说自定义的线程池补充了GPM模型的哪些缺点呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你说的线程池是goroutine池么？如果是，第35讲 对goroutine池的必要性做了一些说明，可以先看看。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-16 11:30:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/64/8a/bc8cb43c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>路边的猪</span>
  </div>
  <div class="_2_QraFYR_0">抢占式调度这里，比如 go func(){ for {}}<br>这种情况 协程内确实没有其他的函数调用 是一个死循环，但是这个func() 本身不就是个函数吗，为啥不能在这个函数开头插入检测代码呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 即便如你所说的那样，但当进入到for {}后，还哪有机会走到插入的检测的代码呢？<br><br>下面这样才可以进入到func1的检测代码：(Go 1.14版本之前)<br><br>go func() {<br>    for {<br>       func1()<br>    }<br>}</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-05 17:15:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/79/96/fb0d8a65.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>李亮</span>
  </div>
  <div class="_2_QraFYR_0">抢占是什么意思？G被抢占是指本来停止运行G，换成另外一个G吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 原先正在运行的G被强行终止，换另外一个G运行。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-21 10:46:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83eoGLuQGiaBftJglfdCicBnCWw8tZKkbEJoU3lpgI10mAzaYgS0QLicG02GMH9zSzFD2KFQ0SHc6PuMUA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_1d2661</span>
  </div>
  <div class="_2_QraFYR_0">1.用1.13是实现了抢占式调度 但是那个是通过函数埋点的  这里的不是函数 所以不会运行吧<br>2.使用函数调度应该可以吧</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. 可从多核角度考虑一下<br>2. 还是从可利用的调度资源角度考虑一下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-18 18:17:58</div>
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
  <div class="_2_QraFYR_0">gpm模型还是有性能问题吧？ m应该要尽量绑核，阻塞调用，应该用其它线程去处理，尽量不要占用当前m。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-02 17:57:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2d/b7/30/c1e4f5b5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>微微超级丹💫</span>
  </div>
  <div class="_2_QraFYR_0">请问常规文件是不支持监听的（pollable）是什么意思呀？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 就是说对常规文件的读写操作（因为是系统调用）依然会阻塞M。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-20 10:19:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/24/eb/2e/90fea784.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>柒</span>
  </div>
  <div class="_2_QraFYR_0">Goroutine 传递问题：M 经常在 M 之间传递“可运行”的 Goroutine？为啥会有传递呀？ 就算是上下文切换，只会挂起一个G，M切换到执行另一个G，它不会从一个M传递到另一个M嘛。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 首先这是P-M模型中的问题；<br>其次，可运行是指runnable，即该G在nextG或M的本地队列中(P-M模型)，并不是正在M上运行的G；<br>最后，当在M上运行的G出现慢系统调用后，M和G挂起，其nextG和本地队列中的G可能就要被传递给其他M(已有的idle m或新建M)处理。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-15 23:09:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/2e/ca/469f7266.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>菠萝吹雪—Code</span>
  </div>
  <div class="_2_QraFYR_0">打卡</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-08 10:34:06</div>
  </div>
</div>
</div>
</li>
</ul>