<audio title="31 _ sync.WaitGroup和sync.Once" src="https://static001.geekbang.org/resource/audio/6b/7e/6b88fa520c495b8cbb58b1f263fc6f7e.mp3" controls="controls"></audio> 
<p>我们在前几次讲的互斥锁、条件变量和原子操作都是最基本重要的同步工具。在Go语言中，除了通道之外，它们也算是最为常用的并发安全工具了。</p><p>说到通道，不知道你想过没有，之前在一些场合下里，我们使用通道的方式看起来都似乎有些蹩脚。</p><p>比如：<strong>声明一个通道，使它的容量与我们手动启用的goroutine的数量相同，之后再利用这个通道，让主goroutine等待其他goroutine的运行结束。</strong></p><p>这一步更具体地说就是：让其他的goroutine在运行结束之前，都向这个通道发送一个元素值，并且，让主goroutine在最后从这个通道中接收元素值，接收的次数需要与其他的goroutine的数量相同。</p><p>这就是下面的<code>coordinateWithChan</code>函数展示的多goroutine协作流程。</p><pre><code>func coordinateWithChan() {
 sign := make(chan struct{}, 2)
 num := int32(0)
 fmt.Printf(&quot;The number: %d [with chan struct{}]\n&quot;, num)
 max := int32(10)
 go addNum(&amp;num, 1, max, func() {
  sign &lt;- struct{}{}
 })
 go addNum(&amp;num, 2, max, func() {
  sign &lt;- struct{}{}
 })
 &lt;-sign
 &lt;-sign
}
</code></pre><p>其中的<code>addNum</code>函数的声明在demo65.go文件中。<code>addNum</code>函数会把它接受的最后一个参数值作为其中的<code>defer</code>函数。</p><p>我手动启用的两个goroutine都会调用<code>addNum</code>函数，而它们传给该函数的最后一个参数值（也就是那个既无参数声明，也无结果声明的函数）都只会做一件事情，那就是向通道<code>sign</code>发送一个元素值。</p><!-- [[[read_end]]] --><p>看到<code>coordinateWithChan</code>函数中最后的那两行代码了吗？重复的两个接收表达式<code>&lt;-sign</code>，是不是看起来很丑陋？</p><h2>前导内容：<code>sync</code>包的<code>WaitGroup</code>类型</h2><p>其实，在这种应用场景下，我们可以选用另外一个同步工具，即：<code>sync</code>包的<code>WaitGroup</code>类型。它比通道更加适合实现这种一对多的goroutine协作流程。</p><p><code>sync.WaitGroup</code>类型（以下简称<code>WaitGroup</code>类型）是开箱即用的，也是并发安全的。同时，与我们前面讨论的几个同步工具一样，它一旦被真正使用就不能被复制了。</p><p><code>WaitGroup</code>类型拥有三个指针方法：<code>Add</code>、<code>Done</code>和<code>Wait</code>。你可以想象该类型中有一个计数器，它的默认值是<code>0</code>。我们可以通过调用该类型值的<code>Add</code>方法来增加，或者减少这个计数器的值。</p><p>一般情况下，我会用这个方法来记录需要等待的goroutine的数量。相对应的，这个类型的<code>Done</code>方法，用于对其所属值中计数器的值进行减一操作。我们可以在需要等待的goroutine中，通过<code>defer</code>语句调用它。</p><p>而此类型的<code>Wait</code>方法的功能是，阻塞当前的goroutine，直到其所属值中的计数器归零。如果在该方法被调用的时候，那个计数器的值就是<code>0</code>，那么它将不会做任何事情。</p><p>你可能已经看出来了，<code>WaitGroup</code>类型的值（以下简称<code>WaitGroup</code>值）完全可以被用来替换<code>coordinateWithChan</code>函数中的通道<code>sign</code>。下面的<code>coordinateWithWaitGroup</code>函数就是它的改造版本。</p><pre><code>func coordinateWithWaitGroup() {
 var wg sync.WaitGroup
 wg.Add(2)
 num := int32(0)
 fmt.Printf(&quot;The number: %d [with sync.WaitGroup]\n&quot;, num)
 max := int32(10)
 go addNum(&amp;num, 3, max, wg.Done)
 go addNum(&amp;num, 4, max, wg.Done)
 wg.Wait()
}
</code></pre><p>很明显，整体代码少了好几行，而且看起来也更加简洁了。这里我先声明了一个<code>WaitGroup</code>类型的变量<code>wg</code>。然后，我调用了它的<code>Add</code>方法并传入了<code>2</code>，因为我会在后面启用两个需要等待的goroutine。</p><p>由于<code>wg</code>变量的<code>Done</code>方法本身就是一个既无参数声明，也无结果声明的函数，所以我在<code>go</code>语句中调用<code>addNum</code>函数的时候，可以直接把该方法作为最后一个参数值传进去。</p><p>在<code>coordinateWithWaitGroup</code>函数的最后，我调用了<code>wg</code>的<code>Wait</code>方法。如此一来，该函数就可以等到那两个goroutine都运行结束之后，再结束执行了。</p><p>以上就是<code>WaitGroup</code>类型最典型的应用场景了。不过不能止步于此，对于这个类型，我们还是有必要再深入了解一下的。我们一起看下面的问题。</p><p><strong>问题：<code>sync.WaitGroup</code>类型值中计数器的值可以小于<code>0</code>吗？</strong></p><p>这里的典型回答是：不可以。</p><h2>问题解析</h2><p>为什么不可以呢，我们解析一下。<strong>之所以说<code>WaitGroup</code>值中计数器的值不能小于<code>0</code>，是因为这样会引发一个panic。</strong> 不适当地调用这类值的<code>Done</code>方法和<code>Add</code>方法都会如此。别忘了，我们在调用<code>Add</code>方法的时候是可以传入一个负数的。</p><p>实际上，导致<code>WaitGroup</code>值的方法抛出panic的原因不只这一种。</p><p>你需要知道，在我们声明了这样一个变量之后，应该首先根据需要等待的goroutine，或者其他事件的数量，调用它的<code>Add</code>方法，以使计数器的值大于<code>0</code>。这是确保我们能在后面正常地使用这类值的前提。</p><p>如果我们对它的<code>Add</code>方法的首次调用，与对它的<code>Wait</code>方法的调用是同时发起的，比如，在同时启用的两个goroutine中，分别调用这两个方法，<strong>那么就有可能会让这里的<code>Add</code>方法抛出一个panic。</strong></p><p>这种情况不太容易复现，也正因为如此，我们更应该予以重视。所以，虽然<code>WaitGroup</code>值本身并不需要初始化，但是尽早地增加其计数器的值，还是非常有必要的。</p><p>另外，你可能已经知道，<code>WaitGroup</code>值是可以被复用的，但需要保证其计数周期的完整性。这里的计数周期指的是这样一个过程：该值中的计数器值由<code>0</code>变为了某个正整数，而后又经过一系列的变化，最终由某个正整数又变回了<code>0</code>。</p><p>也就是说，只要计数器的值始于<code>0</code>又归为<code>0</code>，就可以被视为一个计数周期。在一个此类值的生命周期中，它可以经历任意多个计数周期。但是，只有在它走完当前的计数周期之后，才能够开始下一个计数周期。</p><p><img src="https://static001.geekbang.org/resource/image/fa/8d/fac7dfa184053d2a95e121aa17141d8d.png?wh=1350*683" alt=""><br>
（sync.WaitGroup的计数周期）</p><p>因此，也可以说，如果一个此类值的<code>Wait</code>方法在它的某个计数周期中被调用，那么就会立即阻塞当前的goroutine，直至这个计数周期完成。在这种情况下，该值的下一个计数周期，必须要等到这个<code>Wait</code>方法执行结束之后，才能够开始。</p><p>如果在一个此类值的<code>Wait</code>方法被执行期间，跨越了两个计数周期，<strong>那么就会引发一个panic。</strong></p><p>例如，在当前的goroutine因调用此类值的<code>Wait</code>方法，而被阻塞的时候，另一个goroutine调用了该值的<code>Done</code>方法，并使其计数器的值变为了<code>0</code>。</p><p>这会唤醒当前的goroutine，并使它试图继续执行<code>Wait</code>方法中其余的代码。但在这时，又有一个goroutine调用了它的<code>Add</code>方法，并让其计数器的值又从<code>0</code>变为了某个正整数。<strong>此时，这里的<code>Wait</code>方法就会立即抛出一个panic。</strong></p><p>纵观上述会引发panic的后两种情况，我们可以总结出这样一条关于<code>WaitGroup</code>值的使用禁忌，即：<strong>不要把增加其计数器值的操作和调用其<code>Wait</code>方法的代码，放在不同的goroutine中执行。换句话说，要杜绝对同一个<code>WaitGroup</code>值的两种操作的并发执行。</strong></p><p>除了第一种情况外，我们通常需要反复地实验，才能够让<code>WaitGroup</code>值的方法抛出panic。再次强调，虽然这不是每次都发生，但是在长期运行的程序中，这种情况发生的概率还是不小的，我们必须要重视它们。</p><p>如果你对复现这些异常情况感兴趣，那么可以参看<code>sync</code>代码包中的waitgroup_test.go文件。其中的名称以<code>TestWaitGroupMisuse</code>为前缀的测试函数，很好地展示了这些异常情况的发生条件。你可以模仿这些测试函数自己写一些测试代码，执行一下试试看。</p><h2>知识扩展</h2><h3>问题：<code>sync.Once</code>类型值的<code>Do</code>方法是怎么保证只执行参数函数一次的？</h3><p>与<code>sync.WaitGroup</code>类型一样，<code>sync.Once</code>类型（以下简称<code>Once</code>类型）也属于结构体类型，同样也是开箱即用和并发安全的。由于这个类型中包含了一个<code>sync.Mutex</code>类型的字段，所以，复制该类型的值也会导致功能的失效。</p><p><code>Once</code>类型的<code>Do</code>方法只接受一个参数，这个参数的类型必须是<code>func()</code>，即：无参数声明和结果声明的函数。</p><p>该方法的功能并不是对每一种参数函数都只执行一次，而是只执行“首次被调用时传入的”那个函数，并且之后不会再执行任何参数函数。</p><p>所以，如果你有多个只需要执行一次的函数，那么就应该为它们中的每一个都分配一个<code>sync.Once</code>类型的值（以下简称<code>Once</code>值）。</p><p><code>Once</code>类型中还有一个名叫<code>done</code>的<code>uint32</code>类型的字段。它的作用是记录其所属值的<code>Do</code>方法被调用的次数。不过，该字段的值只可能是<code>0</code>或者<code>1</code>。一旦<code>Do</code>方法的首次调用完成，它的值就会从<code>0</code>变为<code>1</code>。</p><p>你可能会问，既然<code>done</code>字段的值不是<code>0</code>就是<code>1</code>，那为什么还要使用需要四个字节的<code>uint32</code>类型呢？</p><p>原因很简单，因为对它的操作必须是“原子”的。<code>Do</code>方法在一开始就会通过调用<code>atomic.LoadUint32</code>函数来获取该字段的值，并且一旦发现该值为<code>1</code>，就会直接返回。这也初步保证了“<code>Do</code>方法，只会执行首次被调用时传入的函数”。</p><p>不过，单凭这样一个判断的保证是不够的。因为，如果有两个goroutine都调用了同一个新的<code>Once</code>值的<code>Do</code>方法，并且几乎同时执行到了其中的这个条件判断代码，那么它们就都会因判断结果为<code>false</code>，而继续执行<code>Do</code>方法中剩余的代码。</p><p>在这个条件判断之后，<code>Do</code>方法会立即锁定其所属值中的那个<code>sync.Mutex</code>类型的字段<code>m</code>。然后，它会在临界区中再次检查<code>done</code>字段的值，并且仅在条件满足时，才会去调用参数函数，以及用原子操作把<code>done</code>的值变为<code>1</code>。</p><p>如果你熟悉GoF设计模式中的单例模式的话，那么肯定能看出来，这个<code>Do</code>方法的实现方式，与那个单例模式有很多相似之处。它们都会先在临界区之外，判断一次关键条件，若条件不满足则立即返回。这通常被称为<strong>“快路径”，或者叫做“快速失败路径”。</strong></p><p>如果条件满足，那么到了临界区中还要再对关键条件进行一次判断，这主要是为了更加严谨。这两次条件判断常被统称为（跨临界区的）“双重检查”。</p><p>由于进入临界区之前，肯定要锁定保护它的互斥锁<code>m</code>，显然会降低代码的执行速度，所以其中的第二次条件判断，以及后续的操作就被称为“慢路径”或者“常规路径”。</p><p>别看<code>Do</code>方法中的代码不多，但它却应用了一个很经典的编程范式。我们在Go语言及其标准库中，还能看到不少这个经典范式及它衍生版本的应用案例。</p><p><strong>下面我再来说说这个<code>Do</code>方法在功能方面的两个特点。</strong></p><p><strong>第一个特点</strong>，由于<code>Do</code>方法只会在参数函数执行结束之后把<code>done</code>字段的值变为<code>1</code>，因此，如果参数函数的执行需要很长时间或者根本就不会结束（比如执行一些守护任务），那么就有可能会导致相关goroutine的同时阻塞。</p><p>例如，有多个goroutine并发地调用了同一个<code>Once</code>值的<code>Do</code>方法，并且传入的函数都会一直执行而不结束。那么，这些goroutine就都会因调用了这个<code>Do</code>方法而阻塞。因为，除了那个抢先执行了参数函数的goroutine之外，其他的goroutine都会被阻塞在锁定该<code>Once</code>值的互斥锁<code>m</code>的那行代码上。</p><p><strong>第二个特点</strong>，<code>Do</code>方法在参数函数执行结束后，对<code>done</code>字段的赋值用的是原子操作，并且，这一操作是被挂在<code>defer</code>语句中的。因此，不论参数函数的执行会以怎样的方式结束，<code>done</code>字段的值都会变为<code>1</code>。</p><p>也就是说，即使这个参数函数没有执行成功（比如引发了一个panic），我们也无法使用同一个<code>Once</code>值重新执行它了。所以，如果你需要为参数函数的执行设定重试机制，那么就要考虑<code>Once</code>值的适时替换问题。</p><p>在很多时候，我们需要依据<code>Do</code>方法的这两个特点来设计与之相关的流程，以避免不必要的程序阻塞和功能缺失。</p><h2>总结</h2><p><code>sync</code>代码包的<code>WaitGroup</code>类型和<code>Once</code>类型都是非常易用的同步工具。它们都是开箱即用和并发安全的。</p><p>利用<code>WaitGroup</code>值，我们可以很方便地实现一对多的goroutine协作流程，即：一个分发子任务的goroutine，和多个执行子任务的goroutine，共同来完成一个较大的任务。</p><p>在使用<code>WaitGroup</code>值的时候，我们一定要注意，千万不要让其中的计数器的值小于<code>0</code>，否则就会引发panic。</p><p>另外，<strong>我们最好用“先统一<code>Add</code>，再并发<code>Done</code>，最后<code>Wait</code>”这种标准方式，来使用<code>WaitGroup</code>值。</strong> 尤其不要在调用<code>Wait</code>方法的同时，并发地通过调用<code>Add</code>方法去增加其计数器的值，因为这也有可能引发panic。</p><p><code>Once</code>值的使用方式比<code>WaitGroup</code>值更加简单，它只有一个<code>Do</code>方法。同一个<code>Once</code>值的<code>Do</code>方法，永远只会执行第一次被调用时传入的参数函数，不论这个函数的执行会以怎样的方式结束。</p><p>只要传入某个<code>Do</code>方法的参数函数没有结束执行，任何之后调用该方法的goroutine就都会被阻塞。只有在这个参数函数执行结束以后，那些goroutine才会逐一被唤醒。</p><p><code>Once</code>类型使用互斥锁和原子操作实现了功能，而<code>WaitGroup</code>类型中只用到了原子操作。	所以可以说，它们都是更高层次的同步工具。它们都基于基本的通用工具，实现了某一种特定的功能。<code>sync</code>包中的其他高级同步工具，其实也都是这样的。</p><h2>思考题</h2><p>今天的思考题是：在使用<code>WaitGroup</code>值实现一对多的goroutine协作流程时，怎样才能让分发子任务的goroutine获得各个子任务的具体执行结果？</p><p><a href="https://github.com/hyper0x/Golang_Puzzlers">戳此查看Go语言专栏文章配套详细代码。</a></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/27/fc/b8d83d56.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>liangjf</span>
  </div>
  <div class="_2_QraFYR_0">“双重检查” 貌似也并不是完全安全的吧，像c++11那样加入内存屏障才是真正线性安全的。go有这类接口吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Go语言底层内置了内存屏障。它的好处就是不用像C++那样什么都需要自己搞。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-26 20:49:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/05/3c/6f2a4724.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>唐大少在路上。。。</span>
  </div>
  <div class="_2_QraFYR_0">个人感觉Once里面的逻辑设计得不够简洁，既然目的就是只要能够拿到once的锁的gorountine就会消费掉这个once，那其实直接在Do方法的最开始用if atomic.CompareAndSwapUint32(&amp;o.done, 0,1)不就行了，连锁都不用。<br>还请老师指正，哈哈</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这样不行啊，它还得执行你给它的函数啊。怎么能在没执行函数之前就把 done 变成 1 呢，对吧。但如果是在执行之后 swap，那又太晚了，有可能出现重复执行函数的情况。<br><br>所以 Once 中才有两个执行路径，一个是仅包含原子操作的快路径，另一个是真正准备执行函数的慢路径。这样才可以兼顾多种情况，让总体性能更优。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-24 16:42:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/02/3b/b4a47f63.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>only</span>
  </div>
  <div class="_2_QraFYR_0">可不可以把 sync.once 理解为单例模式，比如连接数据库只需要连接一次，把连接数据库的代码实在once.do()里面</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 它跟单例模式还不太一样。单例模式指的是某类结构的唯一实例，而 once 指的是对某段代码的唯一一次执行。它们的维度不一样。<br><br>连接数据库的代码其实不太适合放到 Do 里面执行，或者说不太恰当。初始化数据库链接的代码可以放到里面。而，断链重连的机制也应该在其中。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-10 08:42:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/20/27/a6932fbe.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>虢國技醬</span>
  </div>
  <div class="_2_QraFYR_0">二刷走起</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-26 15:09:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTIGMzsqGicBHicmA0xosKibBSmFjrwG8LuRwky3QlZibJt1treDMLPuaKviaC9JrJdQpdJE199ztVMJOGQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>超大叮当当</span>
  </div>
  <div class="_2_QraFYR_0">sync.Once 不用 Mutex ，直接用 atomic.CompareAndSwapUint32 函数也可以安全吧？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 原子操作是CPU级别的互斥，而且防中断。但是支持的数据类型很少，而且并不灵活。所以如果是对代码块进行保护，还需要用锁。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-13 17:34:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/4a/53/063f9d17.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>moonfox</span>
  </div>
  <div class="_2_QraFYR_0">请问一下，在 sync.Once的源码里， doSlow()方法中，已经用了o.m.Lock()，什么写入o.done=1的时候，还要用原子写入呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这是两码事啊，原子操作还有一个作用是保证被操作值的完整性。比如，done字段的值要么是0要么是1。别忘了，done字段的值是由32个比特位组成的。如果在修改值的过程中（还没改完），其他的代码在读取它，那岂不是会读到一个非0非1的值吗？（这是一个小概率问题，但是万一出了错，复现都没法复现，排查起来就太困难了，所以恰恰需要极力避免）<br><br>就像源码中的<br><br>if atomic.LoadUint32(&amp;o.done) == 0 {<br><br>这里。<br><br>这行代码可没有锁的加持啊，它可不管另一个goroutine执行到doSlow函数中的哪一步了。另外，对一个值的原子操作必须全面覆盖（如果用，就都要用原子操作）。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-09 15:30:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/6c/b2/362353ba.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>蔺晨</span>
  </div>
  <div class="_2_QraFYR_0">思考题 :<br>func getAllGoroutineResult(){<br>		wg := sync.WaitGroup{}<br>		wg.Add(3)<br><br>		once := sync.Once{}<br>		var aAndb int<br>		var aStrAndb string<br>		var gflag int32<br><br>		addNum := func(a,b int, ret *int) {<br>			defer wg.Done()<br>			time.Sleep(time.Millisecond * 2000)<br>			*ret = a+b<br>			atomic.AddInt32(&amp;gflag,1)<br>		}<br><br>		addStr := func(a,b string, ret *string) {<br>			defer wg.Done()<br>			time.Sleep(time.Millisecond * 1000)<br>			*ret = a+b<br>			atomic.AddInt32(&amp;gflag,1)<br>		}<br><br>		&#47;&#47; waitRet需要等待 addNum和addStr执行完成后的结果<br>		waitRet := func(ret *int, strRet *string) {<br>			defer wg.Done()<br>			once.Do(func() {<br>				for atomic.LoadInt32(&amp;gflag) != 2 {<br>					fmt.Println(&quot;Wait: addNum &amp; addStr&quot;)<br>					time.Sleep(time.Millisecond * 200)<br>				}<br>			})<br>			fmt.Println(fmt.Sprintf(&quot;AddNum&#39;s Ret is: %d\n&quot;, *ret))<br>			fmt.Println(fmt.Sprintf(&quot;AddStr&#39;s Ret is: %s\n&quot;, *strRet))<br>		}<br><br>		&#47;&#47; waitRet goroutine等待AddNum和AddStr结束<br>		go waitRet(&amp;aAndb, &amp;aStrAndb)<br>		go addNum(10, 20, &amp;aAndb)<br>		go addStr(&quot;测试结果&quot;, &quot;满意不?&quot;, &amp;aStrAndb)<br><br>		wg.Wait()<br>}</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-21 16:30:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>窗外</span>
  </div>
  <div class="_2_QraFYR_0">go func ()  {<br>		 wg.Done()<br>		fmt.Println(&quot;send complete&quot;)<br>	 }()<br>老师，为什么在Done()后的代码就不会被执行呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你在后面 wg.Wait() 了吗？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-03 10:28:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/bb/10/f01eafe4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ricktian</span>
  </div>
  <div class="_2_QraFYR_0">执行结果如果不用channel实现，还有什么方法？请老师指点～</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-30 11:29:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4a/96/99466a06.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Laughing</span>
  </div>
  <div class="_2_QraFYR_0">子任务的结果应该用通道来传递吧。另外once的应用场景还是没有理解。郝大能简单说一下么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以通过通道，但这就不是wg的作用范围了。once一般是执行只应该执行一次的任务，比如初始化连接池等等。你可以在go源码里搜一下，用的地方还是不少的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-30 09:28:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/4f/78/c3d8ecb0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>undifined</span>
  </div>
  <div class="_2_QraFYR_0">执行结果用 Callback，放在通道中，在主 goroutine 中接收返回结果</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-22 14:55:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/97/c5/84491beb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>罗峰</span>
  </div>
  <div class="_2_QraFYR_0">老师，你好，waitgroup的计数周期这个概念是自创的吗？使用上感觉 只要 add操作在wait语句之前执行就可以，使用个例子：<br> for {<br>      select {<br>      case &lt;- cancel:<br>         break;<br>      case &lt;- taskqueue:<br>        go func {<br>             wg.add(1)<br>              ....<br>             defer wg.done()<br>            }<br>     }<br>}<br>wg.wait()</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: WaitGroup与操作系统的信号灯异曲同工。<br><br>必须要有 wg.Done() 啊，否则计数就无法归零，wg.Wait() 也就无法消除阻塞。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-22 11:33:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/c3/22/8520be75.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>1287</span>
  </div>
  <div class="_2_QraFYR_0">没理解使用once和自己只调用一次有什么区别，类似初始化的操作，我在程序执行前写个init也是只执行一次吧，求教</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 同一个 sync.Once 实例的 Do 方法只会被有效调用一次。init 函数是在当前程序中只会被调用一次，而且它的作用域是代码包。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-25 11:50:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>手指饼干</span>
  </div>
  <div class="_2_QraFYR_0">请问老师，如下deferFunc为什么要用func包装起来，直接使用defer deferFunc()不可以吗？<br>func addNum(numP *int32, id, max int32, deferFunc func()) {<br>	defer func() {<br>		deferFunc()<br>	}()<br>	&#47;&#47;...<br>}</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你那么写也可以，我弄一坨只是想引起你们的注意。在不影响程序功能和运行效率的前提下，我会在程序里尽量多展示几种写法。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-10 10:25:27</div>
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
  <div class="_2_QraFYR_0">今日总结<br>这里主要讲了sync&#47;waitgroup和sync&#47;once这两种同步的方式<br>waitgroup一般用于一对多的同步 例如 主goroutine等待其他goroutine的执行完成<br>唯一要注意的是 add方法和wait方法不要并行执行 也即 wait方法调用过后 不要立即再调用add方法<br>once值 主要还是保证函数只被执行一次 且只有第一次调用Do方法的参数函数会被执行一次 once值底层依赖的是互斥锁 所以操作和互斥锁很类似<br>并且 如果某个Do的参数函数一直不结束 那么其他调用Do方法的goroutine都会被阻塞在获取互斥锁这一步，只有当首次Do方法的参数函数执行结束过后 这些goroutine才会被逐一唤醒<br>所以要注意 Once 在使用死的死锁 </div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-10 10:44:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/97/86/fb564a19.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>bluuus</span>
  </div>
  <div class="_2_QraFYR_0">func coordinateWithWaitGroup() {<br>	var wg sync.WaitGroup<br>	wg.Add(2)<br>	num := int32(0)<br>	fmt.Printf(&quot;The number: %d [with sync.WaitGroup]\n&quot;, num)<br>	max := int32(10)<br>	go addNum(&amp;num, 3, max, wg.Done)<br>	&#47;&#47;go addNum(&amp;num, 4, max, wg.Done)<br>	wg.Wait()<br>}<br><br>run result:<br>The number: 0 [with sync.WaitGroup]<br>The number: 2 [3-0]<br>The number: 4 [3-1]<br>The number: 6 [3-2]<br>The number: 8 [3-3]<br>The number: 10 [3-4]<br>fatal error: all goroutines are asleep - deadlock!<br><br>执行这段代码会死锁，我以为最多在wai()方法那儿阻塞，谁能解释一下？<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你对 addNum 函数有改动吗？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-06 16:18:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/e8/55/92f82281.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>MClink</span>
  </div>
  <div class="_2_QraFYR_0">easy case<br>func TestWaitGroup() {<br>	var wg sync.WaitGroup<br>	wg.Add(3)<br>	for i := 0; i &lt; 3; i++ {<br>		go func() {<br>			defer wg.Done()<br>			time.Sleep(time.Second * 3)<br>			fmt.Println(&quot;done!&quot;)<br>		}()<br>	}<br><br>	wg.Wait()<br>	fmt.Println(&quot;over&quot;)<br>}<br><br>func TestOnce() {<br>	var o sync.Once<br>	var wg sync.WaitGroup<br>	wg.Add(10)<br>	for i := 0; i &lt; 10; i++ {<br>		go func() {<br>			defer wg.Done()<br>			time.Sleep(time.Second * 1)<br>			o.Do(func() {<br>				fmt.Println(&quot;mclink&quot;)<br>			})<br>		}()<br>	}<br>	wg.Wait()<br>	fmt.Println(&quot;over&quot;)<br>}</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-10 15:05:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>NeoMa</span>
  </div>
  <div class="_2_QraFYR_0">您好，关于once.go中 doSlow方法有两个疑问：<br>1. 在拿到m.Lock()锁之后的o.done == 0 的判断能否省略？或者说这个判断针对的是某些可能的未知场景吗？我的理解是，理论上此时这个值一定是0，但为了确保预防未知场景又做了一次判断。<br>2. defer atomic.StoreUint32(&amp;o.done, 1) 这个操作是否可以不用原子操作，只是使用例如 o.done = 1类似的赋值操作？<br>希望您多多指正。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. 当然不能省略了，如果在当前 goroutine 的 atomic.LoadUint32(&amp;o.done) == 0 和 o.doSlow(f) 的执行间隙有其他 goroutine 执行到了 atomic.StoreUint32(&amp;o.done, 1) ，那么再等到当前 goroutine 继续执行的时候 done 可就不是 0 了。<br><br>注意！锁只保证：它保护的某一个临界区内的代码在同一时刻只被某一个 goroutine 执行（也可以说接触到），而不会保证其中代码执行的原子性。所以，在其中代码执行期间也有可能被中断，中断的粒度通常是语句级别的。<br><br>2. 针对同一个共享变量的原子操作必须是完全的，否则就跟没做原子操作没啥区别。就像刚才说的，锁只负责保证串行访问和执行，它没法实现原子性操作。在这里，锁和原子操作是各司其职的。（除非你完全用锁把对 done 的一切操作都保护起来，否则，即然用了原子操作来保证 done 的原子性就要用完全）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-17 17:15:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/3b/ba/3b30dcde.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>窝窝头</span>
  </div>
  <div class="_2_QraFYR_0">执行结果可以通过共享内存、channel之类的，如果只是想让其他协程等待所有协程都完成后统一退出的话是不是每个协程里面wg.Done,然后wg.Wait就行了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-06 10:26:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83erpYZalYvFGcBs7zZvYwaQAZwTLiaw0mycJ4PdYpP3VxAYkAtyIRHhjSOrOK0yESaPpgEbVQUwf6LA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Harlan</span>
  </div>
  <div class="_2_QraFYR_0">go 里面使用waitGroup后，对多个goroutine 的结果还是得用 ch来处理,比较难受</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-15 17:44:41</div>
  </div>
</div>
</div>
</li>
</ul>