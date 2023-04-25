<audio title="26 _ sync.Mutex与sync.RWMutex" src="https://static001.geekbang.org/resource/audio/08/e5/0835ae30d16b2f864e5781e029b0b7e5.mp3" controls="controls"></audio> 
<p>我在前面用20多篇文章，为你详细地剖析了Go语言本身的一些东西，这包括了基础概念、重要语法、高级数据类型、特色语句、测试方案等等。</p><p>这些都是Go语言为我们提供的最核心的技术。我想，这已经足够让你对Go语言有一个比较深刻的理解了。</p><p>从本篇文章开始，我们将一起探讨Go语言自带标准库中一些比较核心的代码包。这会涉及这些代码包的标准用法、使用禁忌、背后原理以及周边的知识。</p><hr></hr><p>既然Go语言是以独特的并发编程模型傲视群雄的语言，那么我们就先来学习与并发编程关系最紧密的代码包。</p><h2>前导内容： 竞态条件、临界区与同步工具</h2><p>我们首先要看的就是<code>sync</code>包。这里的“sync”的中文意思是“同步”。我们下面就从同步讲起。</p><p>相比于Go语言宣扬的“用通讯的方式共享数据”，通过共享数据的方式来传递信息和协调线程运行的做法其实更加主流，毕竟大多数的现代编程语言，都是用后一种方式作为并发编程的解决方案的（这种方案的历史非常悠久，恐怕可以追溯到上个世纪多进程编程时代伊始了）。</p><p>一旦数据被多个线程共享，那么就很可能会产生争用和冲突的情况。这种情况也被称为<strong>竞态条件（race condition）</strong>，这往往会破坏共享数据的一致性。</p><p>共享数据的一致性代表着某种约定，即：多个线程对共享数据的操作总是可以达到它们各自预期的效果。</p><!-- [[[read_end]]] --><p>如果这个一致性得不到保证，那么将会影响到一些线程中代码和流程的正确执行，甚至会造成某种不可预知的错误。这种错误一般都很难发现和定位，排查起来的成本也是非常高的，所以一定要尽量避免。</p><p>举个例子，同时有多个线程连续向同一个缓冲区写入数据块，如果没有一个机制去协调这些线程的写入操作的话，那么被写入的数据块就很可能会出现错乱。比如，在线程A还没有写完一个数据块的时候，线程B就开始写入另外一个数据块了。</p><p>显然，这两个数据块中的数据会被混在一起，并且已经很难分清了。因此，在这种情况下，我们就需要采取一些措施来协调它们对缓冲区的修改。这通常就会涉及同步。</p><p>概括来讲，<strong>同步的用途有两个，一个是避免多个线程在同一时刻操作同一个数据块，另一个是协调多个线程，以避免它们在同一时刻执行同一个代码块。</strong></p><p>由于这样的数据块和代码块的背后都隐含着一种或多种资源（比如存储资源、计算资源、I/O资源、网络资源等等），所以我们可以把它们看做是共享资源，或者说共享资源的代表。我们所说的同步其实就是在控制多个线程对共享资源的访问。</p><p>一个线程在想要访问某一个共享资源的时候，需要先申请对该资源的访问权限，并且只有在申请成功之后，访问才能真正开始。</p><p>而当线程对共享资源的访问结束时，它还必须归还对该资源的访问权限，若要再次访问仍需申请。</p><p>你可以把这里所说的访问权限想象成一块令牌，线程一旦拿到了令牌，就可以进入指定的区域，从而访问到资源，而一旦线程要离开这个区域了，就需要把令牌还回去，绝不能把令牌带走。</p><p>如果针对某个共享资源的访问令牌只有一块，那么在同一时刻，就最多只能有一个线程进入到那个区域，并访问到该资源。</p><p>这时，我们可以说，多个并发运行的线程对这个共享资源的访问是完全串行的。只要一个代码片段需要实现对共享资源的串行化访问，就可以被视为一个临界区（critical section），也就是我刚刚说的，由于要访问到资源而必须进入的那个区域。</p><p>比如，在我前面举的那个例子中，实现了数据块写入操作的代码就共同组成了一个临界区。如果针对同一个共享资源，这样的代码片段有多个，那么它们就可以被称为相关临界区。</p><p>它们可以是一个内含了共享数据的结构体及其方法，也可以是操作同一块共享数据的多个函数。临界区总是需要受到保护的，否则就会产生竞态条件。<strong>施加保护的重要手段之一，就是使用实现了某种同步机制的工具，也称为同步工具。</strong></p><p><img src="https://static001.geekbang.org/resource/image/73/6c/73d3313640e62bb95855d40c988c2e6c.png?wh=1145*618" alt=""></p><p>（竞态条件、临界区与同步工具）</p><p><strong>在Go语言中，可供我们选择的同步工具并不少。其中，最重要且最常用的同步工具当属互斥量（mutual exclusion，简称mutex）。</strong><code>sync</code>包中的<code>Mutex</code>就是与其对应的类型，该类型的值可以被称为互斥量或者互斥锁。</p><p>一个互斥锁可以被用来保护一个临界区或者一组相关临界区。我们可以通过它来保证，在同一时刻只有一个goroutine处于该临界区之内。</p><p>为了兑现这个保证，每当有goroutine想进入临界区时，都需要先对它进行锁定，并且，每个goroutine离开临界区时，都要及时地对它进行解锁。</p><p>锁定操作可以通过调用互斥锁的<code>Lock</code>方法实现，而解锁操作可以调用互斥锁的<code>Unlock</code>方法。以下是demo58.go文件中重点代码经过简化之后的片段：</p><pre><code>mu.Lock()
_, err := writer.Write([]byte(data))
if err != nil {
 log.Printf(&quot;error: %s [%d]&quot;, err, id)
}
mu.Unlock()
</code></pre><p>你可能已经看出来了，这里的互斥锁就相当于我们前面说的那块访问令牌。那么，我们怎样才能用好这块访问令牌呢？请看下面的问题。</p><p><strong>我们今天的问题是：我们使用互斥锁时有哪些注意事项？</strong></p><p>这里有一个典型回答。</p><p>使用互斥锁的注意事项如下：</p><ol>
<li>不要重复锁定互斥锁；</li>
<li>不要忘记解锁互斥锁，必要时使用<code>defer</code>语句；</li>
<li>不要对尚未锁定或者已解锁的互斥锁解锁；</li>
<li>不要在多个函数之间直接传递互斥锁。</li>
</ol><h2>问题解析</h2><p>首先，你还是要把互斥锁看作是针对某一个临界区或某一组相关临界区的唯一访问令牌。</p><p>虽然没有任何强制规定来限制，你用同一个互斥锁保护多个无关的临界区，但是这样做，一定会让你的程序变得很复杂，并且也会明显地增加你的心智负担。</p><p>你要知道，对一个已经被锁定的互斥锁进行锁定，是会立即阻塞当前的goroutine的。这个goroutine所执行的流程，会一直停滞在调用该互斥锁的<code>Lock</code>方法的那行代码上。</p><p>直到该互斥锁的<code>Unlock</code>方法被调用，并且这里的锁定操作成功完成，后续的代码（也就是临界区中的代码）才会开始执行。这也正是互斥锁能够保护临界区的原因所在。</p><p>一旦，你把一个互斥锁同时用在了多个地方，就必然会有更多的goroutine争用这把锁。这不但会让你的程序变慢，还会大大增加死锁（deadlock）的可能性。</p><p>所谓的死锁，指的就是当前程序中的主goroutine，以及我们启用的那些goroutine都已经被阻塞。这些goroutine可以被统称为用户级的goroutine。这就相当于整个程序都已经停滞不前了。</p><p>Go语言运行时系统是不允许这种情况出现的，只要它发现所有的用户级goroutine都处于等待状态，就会自行抛出一个带有如下信息的panic：</p><pre><code>fatal error: all goroutines are asleep - deadlock!
</code></pre><p><strong>注意，这种由Go语言运行时系统自行抛出的panic都属于致命错误，都是无法被恢复的，调用<code>recover</code>函数对它们起不到任何作用。也就是说，一旦产生死锁，程序必然崩溃。</strong></p><p>因此，我们一定要尽量避免这种情况的发生。而最简单、有效的方式就是让每一个互斥锁都只保护一个临界区或一组相关临界区。</p><p>在这个前提之下，我们还需要注意，对于同一个goroutine而言，既不要重复锁定一个互斥锁，也不要忘记对它进行解锁。</p><p>一个goroutine对某一个互斥锁的重复锁定，就意味着它自己锁死了自己。先不说这种做法本身就是错误的，在这种情况下，想让其他的goroutine来帮它解锁是非常难以保证其正确性的。</p><p>我以前就在团队代码库中见到过这样的代码。那个作者的本意是先让一个goroutine自己锁死自己，然后再让一个负责调度的goroutine定时地解锁那个互斥锁，从而让前一个goroutine周期性地去做一些事情，比如每分钟检查一次服务器状态，或者每天清理一次日志。</p><p>这个想法本身是没有什么问题的，但却选错了实现的工具。对于互斥锁这种需要精细化控制的同步工具而言，这样的任务并不适合它。</p><p>在这种情况下，即使选用通道或者<code>time.Ticker</code>类型，然后自行实现功能都是可以的，程序的复杂度和我们的心智负担也会小很多，更何况还有不少已经很完备的解决方案可供选择。</p><p>话说回来，其实我们说“不要忘记解锁互斥锁”的一个很重要的原因就是：<strong>避免重复锁定。</strong></p><p>因为在一个goroutine执行的流程中，可能会出现诸如“锁定、解锁、再锁定、再解锁”的操作，所以如果我们忘记了中间的解锁操作，那就一定会造成重复锁定。</p><p>除此之外，忘记解锁还会使其他的goroutine无法进入到该互斥锁保护的临界区，这轻则会导致一些程序功能的失效，重则会造成死锁和程序崩溃。</p><p>在很多时候，一个函数执行的流程并不是单一的，流程中间可能会有分叉，也可能会被中断。</p><p>如果一个流程在锁定了某个互斥锁之后分叉了，或者有被中断的可能，那么就应该使用<code>defer</code>语句来对它进行解锁，而且这样的<code>defer</code>语句应该紧跟在锁定操作之后。这是最保险的一种做法。</p><p>忘记解锁导致的问题有时候是比较隐秘的，并不会那么快就暴露出来。这也是我们需要特别关注它的原因。相比之下，解锁未锁定的互斥锁会立即引发panic。</p><p>并且，与死锁导致的panic一样，它们是无法被恢复的。<strong>因此，我们总是应该保证，对于每一个锁定操作，都要有且只有一个对应的解锁操作。</strong></p><p>换句话说，我们应该让它们成对出现。这也算是互斥锁的一个很重要的使用原则了。在很多时候，利用<code>defer</code>语句进行解锁可以更容易做到这一点。</p><p><img src="https://static001.geekbang.org/resource/image/4f/0d/4f86467d09ffca6e0c02602a9cb7480d.png?wh=1610*1052" alt=""></p><p>（互斥锁的重复锁定和重复解锁）</p><p>最后，可能你已经知道，Go语言中的互斥锁是开箱即用的。换句话说，一旦我们声明了一个<code>sync.Mutex</code>类型的变量，就可以直接使用它了。</p><p>不过要注意，该类型是一个结构体类型，属于值类型中的一种。把它传给一个函数、将它从函数中返回、把它赋给其他变量、让它进入某个通道都会导致它的副本的产生。</p><p>并且，原值和它的副本，以及多个副本之间都是完全独立的，它们都是不同的互斥锁。</p><p>如果你把一个互斥锁作为参数值传给了一个函数，那么在这个函数中对传入的锁的所有操作，都不会对存在于该函数之外的那个原锁产生任何的影响。</p><p>所以，你在这样做之前，一定要考虑清楚，这种结果是你想要的吗？我想，在大多数情况下应该都不是。即使你真的希望，在这个函数中使用另外一个互斥锁也不要这样做，这主要是为了避免歧义。</p><p>以上这些，就是我想要告诉你的关于互斥锁的锁定、解锁，以及传递方面的知识。这其中还包括了我的一些理解。希望能够对你有用。相关的例子我已经写在demo59.go文件中了，你可以去阅读一番，并运行起来看看。</p><h2>知识扩展</h2><p>问题1：读写锁与互斥锁有哪些异同？</p><p>读写锁是读/写互斥锁的简称。在Go语言中，读写锁由<code>sync.RWMutex</code>类型的值代表。与<code>sync.Mutex</code>类型一样，这个类型也是开箱即用的。</p><p>顾名思义，读写锁是把对共享资源的“读操作”和“写操作”区别对待了。它可以对这两种操作施加不同程度的保护。换句话说，相比于互斥锁，读写锁可以实现更加细腻的访问控制。</p><p>一个读写锁中实际上包含了两个锁，即：读锁和写锁。<code>sync.RWMutex</code>类型中的<code>Lock</code>方法和<code>Unlock</code>方法分别用于对写锁进行锁定和解锁，而它的<code>RLock</code>方法和<code>RUnlock</code>方法则分别用于对读锁进行锁定和解锁。</p><p>另外，对于同一个读写锁来说有如下规则。</p><ol>
<li>在写锁已被锁定的情况下再试图锁定写锁，会阻塞当前的goroutine。</li>
<li>在写锁已被锁定的情况下试图锁定读锁，也会阻塞当前的goroutine。</li>
<li>在读锁已被锁定的情况下试图锁定写锁，同样会阻塞当前的goroutine。</li>
<li>在读锁已被锁定的情况下再试图锁定读锁，并不会阻塞当前的goroutine。</li>
</ol><p>换一个角度来说，对于某个受到读写锁保护的共享资源，多个写操作不能同时进行，写操作和读操作也不能同时进行，但多个读操作却可以同时进行。</p><p>当然了，只有在我们正确使用读写锁的情况下，才能达到这种效果。还是那句话，我们需要让每一个锁都只保护一个临界区，或者一组相关临界区，并以此尽量减少误用的可能性。顺便说一句，我们通常把这种不能同时进行的操作称为互斥操作。</p><p>再来看另一个方面。对写锁进行解锁，会唤醒“所有因试图锁定读锁，而被阻塞的goroutine”，并且，这通常会使它们都成功完成对读锁的锁定。</p><p>然而，对读锁进行解锁，只会在没有其他读锁锁定的前提下，唤醒“因试图锁定写锁，而被阻塞的goroutine”；并且，最终只会有一个被唤醒的goroutine能够成功完成对写锁的锁定，其他的goroutine还要在原处继续等待。至于是哪一个goroutine，那就要看谁的等待时间最长了。</p><p>除此之外，读写锁对写操作之间的互斥，其实是通过它内含的一个互斥锁实现的。因此，也可以说，Go语言的读写锁是互斥锁的一种扩展。</p><p>最后，需要强调的是，与互斥锁类似，解锁“读写锁中未被锁定的写锁”，会立即引发panic，对于其中的读锁也是如此，并且同样是不可恢复的。</p><p>总之，读写锁与互斥锁的不同，都源于它把对共享资源的写操作和读操作区别对待了。这也使得它实现的互斥规则要更复杂一些。</p><p>不过，正因为如此，我们可以使用它对共享资源的操作，实行更加细腻的控制。另外，由于这里的读写锁是互斥锁的一种扩展，所以在有些方面它还是沿用了互斥锁的行为模式。比如，在解锁未锁定的写锁或读锁时的表现，又比如，对写操作之间互斥的实现方式。</p><h2>总结</h2><p>我们今天讨论了很多与多线程、共享资源以及同步有关的知识。其中涉及了不少重要的并发编程概念，比如，竞态条件、临界区、互斥量、死锁等。</p><p>虽然Go语言是以“用通讯的方式共享数据”为亮点的，但是它依然提供了一些易用的同步工具。其中，互斥锁是我们最常用到的一个。</p><p>互斥锁常常被用来：保证多个goroutine并发地访问同一个共享资源时的完全串行，这是通过保护针对此共享资源的一个临界区，或一组相关临界区实现的。因此，我们可以把它看做是goroutine进入相关临界区时，必须拿到的访问令牌。</p><p>为了用对并且用好互斥锁，我们需要了解它实现的互斥规则，更要理解一些关于它的注意事项。</p><p>比如，不要重复锁定或忘记解锁，因为这会造成goroutine不必要的阻塞，甚至导致程序的死锁。</p><p>又比如，不要传递互斥锁，因为这会产生它的副本，从而引起歧义并可能导致互斥操作的失效。</p><p>再次强调，我们总是应该让每一个互斥锁都只保护一个临界区，或一组相关临界区。</p><p>至于读写锁，它是互斥锁的一种扩展。我们需要知道它与互斥锁的异同，尤其是互斥规则和行为模式方面的异同。一个读写锁中同时包含了读锁和写锁，由此也可以看出它对于针对共享资源的读操作和写操作是区别对待的。我们可以基于这件事，对共享资源实施更加细致的访问控制。</p><p>最后，需要特别注意的是，无论是互斥锁还是读写锁，我们都不要试图去解锁未锁定的锁，因为这样会引发不可恢复的panic。</p><h2>思考题</h2><ol>
<li>你知道互斥锁和读写锁的指针类型都实现了哪一个接口吗？</li>
<li>怎样获取读写锁中的读锁？</li>
</ol><p><a href="https://github.com/hyper0x/Golang_Puzzlers">戳此查看Go语言专栏文章配套详细代码。</a></p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/59/c0/6f02c096.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>属雨</span>
  </div>
  <div class="_2_QraFYR_0">第一个问题：<br>Lock接口。<br>第二个问题：<br>变量.Rlock()</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-10 12:07:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/f9/88/cdda9e6f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>阳仔</span>
  </div>
  <div class="_2_QraFYR_0">学习将近一半的课程了发现:<br>1. 内容不够简洁，很多知识点其实画图出来更容易让读者理解<br>2. 感觉不大适合初学者，反而对已经入门有一定经验的学习者会帮助更大一些<br>3. 可以看出来作者是非常精通go的</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-03 23:02:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_cd5dcf</span>
  </div>
  <div class="_2_QraFYR_0">讲的通俗易懂，还是挺好理解的，想问下mutex如果加锁后<br>mutex.lock（）<br>defer mutex.unlock（）<br>在所有场景下都不会出错吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对，不会，这是defer的机制保证的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-11 08:17:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/ea/80/8759e4c1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>🐻</span>
  </div>
  <div class="_2_QraFYR_0">1. Locker 接口<br>2. func (rw *RWMutex) RLocker() Locker</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: √</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-16 09:19:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/82/fb/9d232a7a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Pana</span>
  </div>
  <div class="_2_QraFYR_0">如果cpu只有一个核心，是不是就不会产生并发的情况？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果CPU只有一个核心，那么就不会有真正的并行计算了。但是，并发还是会有的。这是因为仍然可以同时有多个goroutine存在（它们可以同时处于可运行状态），只不过Go语言的运行时系统无法让它们在同一时刻都运行罢了。<br><br>并发和并行这两个词的含义是不同的，需要我们分清楚。简单来说，并发是指在同一个时间段内提交多个任务给系统，并行是指在同一时刻系统能执行多个任务。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-08 10:27:31</div>
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
  <div class="_2_QraFYR_0">goroutine和协程有什么本质区别啊，搜了网上也没看出来啥本质区别，有这方面的资料吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 传统的协程只是线程内的流程控制工具。它没法做到一个线程内有两个及以上控制流同时进行，只能是这一个挂起那一个运行然后那一个挂起这一个再运行。同时它也不属于多线程编程，没法统一调度多个线程内的控制流。<br><br>goroutine 我就不用多说了，它属于用户级线程，与系统级线程搭配使用，很强大也很灵活。深入的东西可以看我的那本《Go 并发编程实战》。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-22 09:52:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/d7/39/6698b6a9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Hector</span>
  </div>
  <div class="_2_QraFYR_0">读写锁总结，1、可以随便读，即使加锁多个goroutine同时读  2、加锁写的时候，啥也不能干，不能写。即使没加读锁也不能读。老师写的有点绕蒙了。<br>引用原文：对写锁进行解锁，会唤醒“所有因试图锁定读锁，而被阻塞的 goroutine”，并且，这通常会使它们都成功完成对读锁的锁定。意思就是在对资源的写锁进行解锁时，原来你在对该资源上写锁的时候，所有的读锁会锁定来配合写操作，直到写锁解除锁定时，这些读锁才会解锁。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-16 15:25:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/37/92/961ba560.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>授人以🐟，不如授人以渔</span>
  </div>
  <div class="_2_QraFYR_0">老师，麻烦分析一下这样的场景：main goroutine 拿到读锁，此时 goroutine 1 试图拿到写锁但被阻塞，紧接着 goroutine 2 试图拿到读锁。我想知道 goroutine 2 为什么也会被阻塞，另外 main goroutine 读锁被释放后，哪个 goroutine 会继续运行？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. 我又看了下源码。这是为了避免“迭代读锁定”的问题。这个问题最终会导致当前读写锁永不可用。你想想，如果一个 goroutine 一直在不断地读锁定同一个读写锁，那么想要写锁定这个读写锁的 goroutine 就会永远阻塞在那里。<br><br>2. main goroutine 释放读锁之后，goroutine 1 会首先得到写锁定的机会。这同样是为了避免“迭代读锁定”。因为如果先给 goroutine 2 机会，那 goroutine 1 的写锁定不是还得等吗？要真是这样的话，假如 main goroutine 和 goroutine 1 都在不断地试图读锁定，那么 goroutine 2 就会一直阻塞下去。<br><br>所以说，如果这个事情让你的程序停滞了，那么你就要检查一下程序中是不是有“迭代读锁定”的情况。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-07 12:40:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKOM6tVLSiciaQeQst0g3iboWO74ibicicVAia9qno0X6cf65pEKLgdKkUdcpCWpjAB5e6semrFrruiaGQWhg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>NoTryNoSuccess</span>
  </div>
  <div class="_2_QraFYR_0">请问老师，多核心条件下如果两个goroutine底层同时运行在两个线程上，那么此时这两个goroutine实际上是完全并行的。此时它们如果同时进行互斥锁的锁定操作（随后可能同时对同一资源进行写操作）岂不是不能达到对临界区的保护目的了吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 互斥锁、条件变量和原子操作都是由操作系统和CPU指令集支撑的，所以Go语言的这些同步工具是可以在多线程以及多核CPU甚至多CPU上正确执行的。无需担心。相反，这些同步工具恰恰针对的就是并发和并行的应用场景。这正是它们的用武之地啊。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-04 15:40:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/83/4b/0e96fcae.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>sky</span>
  </div>
  <div class="_2_QraFYR_0">郝大 关于demo59这个案例能大概描述下具体的功能流程吗 代码看起来没有方向感啊 </div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-22 16:17:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/df/1e/cea897e8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>传说中的成大大</span>
  </div>
  <div class="_2_QraFYR_0">今日总结<br>今天主要是讲了 关于并发同步问题<br>两种方式<br>1. 互斥锁 sync&#47;Mutex<br>   保证他们对同一个临界区的访问是串行的<br>2. 读写锁 <br>   分为读锁和写锁<br>   当读写锁定时 任何尝试的写的锁都会被阻塞 而其他的读却不会 原因在于写的时候会改变内存区域的数据<br> 当写锁锁定时 无论是读还是写加锁都会阻塞  原因同上<br>当写锁进行解锁时 会唤醒所有读锁阻塞的goroutine <br>而读锁解锁时 会唤醒某一个写锁 具体是哪一个写锁 就看那个写锁等待的事件最长<br>尤其要注意 <br>一个锁应该只保护一个临界区域<br>不要尝试重复锁定(有可能会导致死锁) 引发系统运行时自动的panic 并且这个panic无法被恢复<br>不要对未加锁的锁进行 会引发panic 并且无法恢复<br>不要传递锁 因为锁时结构体类型 传递时 是值传递 会产生很多副本 导致 锁不是一个锁了<br>一定要释放锁 最好的办法还是 defer语句解锁 加锁和解锁语句成对出现<br>关于思考题<br>Lock接口<br>2func (rw *RWMutex) RLocker() Locker</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-31 11:41:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/8e/b0/ef201991.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>CcczzZ</span>
  </div>
  <div class="_2_QraFYR_0">老师，有个疑问，文中说的这句：「对读锁进行解锁，只会在没有其他读锁锁定的前提下，唤醒“因试图锁定写锁，而被阻塞的 goroutine”」。<br>我的理解是，对读锁进行解锁时，此刻若存在其他读锁等待的话，是会优先唤醒读锁的，如果不存在其他等待的读锁，才会唤醒写锁。不知道这样理解是否正确？<br><br>而基于上面的理解，我写了段代码测试了一下，发现结果并不是这样，实际情况是：「当读锁进行解锁时，若此刻存在其他的读锁和写锁，会根据他们实际阻塞等待的时间长短，优先唤醒并执行」<br>就像下面，写锁在前面执行，等待的时间也比读锁场，所以当读锁解锁时，优先唤醒的是等待时间较长的写锁。<br><br>func main() {<br><br>    var rwMu sync.RWMutex<br><br>    &#47;&#47; 模拟多个写&#47;读锁进行阻塞，当释放读锁的时候看谁先获取到锁（会在没有其他读锁的时候，唤醒写锁）<br><br>    rwMu.RLock()<br>    fmt.Println(&quot;start RLock&quot;)<br><br>    &#47;&#47; 写<br>    go func() {<br>        defer func() {<br>            rwMu.Unlock()<br>            fmt.Println(&quot;get UnLock&quot;)<br>        }()<br>        rwMu.Lock()<br>        fmt.Println(&quot;get Lock&quot;)<br>    }()<br>    time.Sleep(time.Millisecond * 200)<br><br>    &#47;&#47; 读<br>    go func() {<br>        defer func() {<br>            rwMu.RUnlock()<br>            fmt.Println(&quot;get RUnLock&quot;)<br>        }()<br>        rwMu.RLock()<br>        fmt.Println(&quot;get RLock&quot;)<br>    }()<br>    time.Sleep(time.Millisecond * 200)<br><br>    rwMu.RUnlock()<br>    fmt.Println(&quot;start RUnLock&quot;)<br>    time.Sleep(time.Second * 1)<br>}<br><br>运行结果（等待时间较长的写操作先执行了）：<br>start RLock<br>start RUnLock<br>get Lock<br>get UnLock<br>get RLock<br>get RUnLock</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 简单一句话：读写锁中的读锁锁定操作之间是不互斥的。另外，对于读写锁，读锁锁定操作会与写锁锁定操作互斥，写锁锁定操作之间也会互斥。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-16 18:48:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/9b/a8/6a391c66.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Leon📷</span>
  </div>
  <div class="_2_QraFYR_0">老师，我想问下多协程并发情况怎么调试日志，google官方似乎也不推荐我们在日志中把协程号打印出来，只能通过添加唯一序列号识别吗</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-13 11:12:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTK66wjYZxVSnSJ33hbBCzSgHXsyQia8JrvJvibeH6Fcficu1lx83K0gzc3lpcxYllqH9ficibqanqKWGrA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_b5876e</span>
  </div>
  <div class="_2_QraFYR_0">通过strace观察了一下锁的系统调用，发现了 futex ，go 语言锁的实现也是依赖于操作系统原语和信号量的</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-19 18:22:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c6/f9/caf27bd3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>大王叫我来巡山</span>
  </div>
  <div class="_2_QraFYR_0">需要请教老师的是，主协程收到信号就被唤醒了，认为可以读了，但是被阻塞的写协程收到锁释放的消息会不会比主协程要早，然后继续获得写的机会，主协程会不会被阻塞？我认为是不会的，此处的锁只是保证了不同写协程互斥的写入，也就是写操作是原子的，但是并不保证读操作一定在写完后就读吧</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对于非缓冲通道，写的 goroutine 必然会先完成操作。锁本身只保证互斥。被阻塞的 goroutine 也会有先有后，但会根据被阻塞那一刻的先后，而不是什么读写的先后。<br><br>另外互斥锁跟原子操作有本质上的区别，不要搞混。<br><br>再另外，goroutine 与协程也有本质上的区别，不要搞混。<br><br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-13 08:24:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/58/d4/c52f9f6d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>芝士老爹</span>
  </div>
  <div class="_2_QraFYR_0">如果一直有新的读锁请求，会不会导致写锁锁不了？<br>还是说如果有了一个wlock锁请求了，现在因为有rlock未释放锁，wlock的协程被阻塞，后面再有新的rlock锁请求也会先被阻塞，等待wlock锁协程先恢复？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 那要看谁先等待了，这里的等待队列是先进先出的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-04 00:17:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/45/2a/c4413de4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>soooldier</span>
  </div>
  <div class="_2_QraFYR_0">配套代码里puzzlers&#47;article26下并没有demo58.go，也没有demo59.go，懵圈中。。。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 看这里吧：https:&#47;&#47;github.com&#47;hyper0x&#47;Golang_Puzzlers&#47;tree&#47;master&#47;src&#47;puzzlers&#47;article22<br><br>你可以把这里的 article 理解成 topic。一些比较长的 topic 可能会被编辑拆分为多篇文章，所以就出现了这种情况。太长的文章对读者们不太友好，不容易集中精力读下去。<br><br>你可以对照着专栏的目录，按照主题，找一下对应的 articleXX 目录。<br><br>Update:<br>我刚刚添加了一个序号映射表：https:&#47;&#47;github.com&#47;hyper0x&#47;Golang_Puzzlers&#47;blob&#47;master&#47;mapping_table.md 。你用这个就可以方便地对照了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-31 13:47:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/ed/4c/8674b6ad.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>timmy21</span>
  </div>
  <div class="_2_QraFYR_0">郝老师，之前问了一个是否需要上锁的问题。有一个细节我忘说了，一个写者，一个读者，并且“读者”读取变量时不需要保证返回最新值。这种场景下是否可以不上锁？或者不上锁会有什么问题吗？会panic吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-15 14:16:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/fe/2d/2c9177ca.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>给力</span>
  </div>
  <div class="_2_QraFYR_0">对于使用锁有个疑问：<br>type Mutex struct {<br>	state int32<br>	sema  uint32<br>}<br>state表示锁的一个状态<br><br>sema这个变量没太理清是做什么的？什么场景下使用</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: sema 实际上代表着基于内存地址的锁机制。相关的代码在 sync 包的 runtime.go 文件和 runtime 包的 seme.go 文件中。你有兴趣的话可以看一看。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-26 10:54:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/b7/f9/a8f26b10.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jacke</span>
  </div>
  <div class="_2_QraFYR_0">问下老师,读写锁解锁部分有点不明白：写锁解锁的时候如果同时有写锁和读锁在等待，是优先唤醒读锁是把？<br>这个规定对读锁解锁也适用是把?</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-07 22:20:14</div>
  </div>
</div>
</div>
</li>
</ul>