<audio title="32 _ context.Context类型" src="https://static001.geekbang.org/resource/audio/aa/ff/aa51f1f404677c148c385f61c8c6e9ff.mp3" controls="controls"></audio> 
<p>我们在上篇文章中讲到了<code>sync.WaitGroup</code>类型：一个可以帮我们实现一对多goroutine协作流程的同步工具。</p><p><strong>在使用<code>WaitGroup</code>值的时候，我们最好用“先统一<code>Add</code>，再并发<code>Done</code>，最后<code>Wait</code>”的标准模式来构建协作流程。</strong></p><p>如果在调用该值的<code>Wait</code>方法的同时，为了增大其计数器的值，而并发地调用该值的<code>Add</code>方法，那么就很可能会引发panic。</p><p>这就带来了一个问题，如果我们不能在一开始就确定执行子任务的goroutine的数量，那么使用<code>WaitGroup</code>值来协调它们和分发子任务的goroutine，就是有一定风险的。一个解决方案是：分批地启用执行子任务的goroutine。</p><h2>前导内容：WaitGroup值补充知识</h2><p>我们都知道，<code>WaitGroup</code>值是可以被复用的，但需要保证其计数周期的完整性。尤其是涉及对其<code>Wait</code>方法调用的时候，它的下一个计数周期必须要等到，与当前计数周期对应的那个<code>Wait</code>方法调用完成之后，才能够开始。</p><p>我在前面提到的可能会引发panic的情况，就是由于没有遵循这条规则而导致的。</p><p>只要我们在严格遵循上述规则的前提下，分批地启用执行子任务的goroutine，就肯定不会有问题。具体的实现方式有不少，其中最简单的方式就是使用<code>for</code>循环来作为辅助。这里的代码如下：</p><!-- [[[read_end]]] --><pre><code>func coordinateWithWaitGroup() {
 total := 12
 stride := 3
 var num int32
 fmt.Printf(&quot;The number: %d [with sync.WaitGroup]\n&quot;, num)
 var wg sync.WaitGroup
 for i := 1; i &lt;= total; i = i + stride {
  wg.Add(stride)
  for j := 0; j &lt; stride; j++ {
   go addNum(&amp;num, i+j, wg.Done)
  }
  wg.Wait()
 }
 fmt.Println(&quot;End.&quot;)
}
</code></pre><p>这里展示的<code>coordinateWithWaitGroup</code>函数，就是上一篇文章中同名函数的改造版本。而其中调用的<code>addNum</code>函数，则是上一篇文章中同名函数的简化版本。这两个函数都已被放置在了demo67.go文件中。</p><p>我们可以看到，经过改造后的<code>coordinateWithWaitGroup</code>函数，循环地使用了由变量<code>wg</code>代表的<code>WaitGroup</code>值。它运用的依然是“先统一<code>Add</code>，再并发<code>Done</code>，最后<code>Wait</code>”的这种模式，只不过它利用<code>for</code>语句，对此进行了复用。</p><p>好了，至此你应该已经对<code>WaitGroup</code>值的运用有所了解了。不过，我现在想让你使用另一种工具来实现上面的协作流程。</p><p><strong>我们今天的问题就是：怎样使用<code>context</code>包中的程序实体，实现一对多的goroutine协作流程？</strong></p><p>更具体地说，我需要你编写一个名为<code>coordinateWithContext</code>的函数。这个函数应该具有上面<code>coordinateWithWaitGroup</code>函数相同的功能。</p><p>显然，你不能再使用<code>sync.WaitGroup</code>了，而要用<code>context</code>包中的函数和<code>Context</code>类型作为实现工具。这里注意一点，是否分批启用执行子任务的goroutine其实并不重要。</p><p>我在这里给你一个参考答案。</p><pre><code>func coordinateWithContext() {
 total := 12
 var num int32
 fmt.Printf(&quot;The number: %d [with context.Context]\n&quot;, num)
 cxt, cancelFunc := context.WithCancel(context.Background())
 for i := 1; i &lt;= total; i++ {
  go addNum(&amp;num, i, func() {
   if atomic.LoadInt32(&amp;num) == int32(total) {
    cancelFunc()
   }
  })
 }
 &lt;-cxt.Done()
 fmt.Println(&quot;End.&quot;)
}
</code></pre><p>在这个函数体中，我先后调用了<code>context.Background</code>函数和<code>context.WithCancel</code>函数，并得到了一个可撤销的<code>context.Context</code>类型的值（由变量<code>cxt</code>代表），以及一个<code>context.CancelFunc</code>类型的撤销函数（由变量<code>cancelFunc</code>代表）。</p><p>在后面那条唯一的<code>for</code>语句中，我在每次迭代中都通过一条<code>go</code>语句，异步地调用<code>addNum</code>函数，调用的总次数只依据了<code>total</code>变量的值。</p><p>请注意我给予<code>addNum</code>函数的最后一个参数值。它是一个匿名函数，其中只包含了一条<code>if</code>语句。这条<code>if</code>语句会“原子地”加载<code>num</code>变量的值，并判断它是否等于<code>total</code>变量的值。</p><p>如果两个值相等，那么就调用<code>cancelFunc</code>函数。其含义是，如果所有的<code>addNum</code>函数都执行完毕，那么就立即通知分发子任务的goroutine。</p><p>这里分发子任务的goroutine，即为执行<code>coordinateWithContext</code>函数的goroutine。它在执行完<code>for</code>语句后，会立即调用<code>cxt</code>变量的<code>Done</code>函数，并试图针对该函数返回的通道，进行接收操作。</p><p>由于一旦<code>cancelFunc</code>函数被调用，针对该通道的接收操作就会马上结束，所以，这样做就可以实现“等待所有的<code>addNum</code>函数都执行完毕”的功能。</p><h2>问题解析</h2><p><code>context.Context</code>类型（以下简称<code>Context</code>类型）是在Go 1.7发布时才被加入到标准库的。而后，标准库中的很多其他代码包都为了支持它而进行了扩展，包括：<code>os/exec</code>包、<code>net</code>包、<code>database/sql</code>包，以及<code>runtime/pprof</code>包和<code>runtime/trace</code>包，等等。</p><p><code>Context</code>类型之所以受到了标准库中众多代码包的积极支持，主要是因为它是一种非常通用的同步工具。它的值不但可以被任意地扩散，而且还可以被用来传递额外的信息和信号。</p><p>更具体地说，<code>Context</code>类型可以提供一类代表上下文的值。此类值是并发安全的，也就是说它可以被传播给多个goroutine。</p><p>由于<code>Context</code>类型实际上是一个接口类型，而<code>context</code>包中实现该接口的所有私有类型，都是基于某个数据类型的指针类型，所以，如此传播并不会影响该类型值的功能和安全。</p><p><code>Context</code>类型的值（以下简称<code>Context</code>值）是可以繁衍的，这意味着我们可以通过一个<code>Context</code>值产生出任意个子值。这些子值可以携带其父值的属性和数据，也可以响应我们通过其父值传达的信号。</p><p>正因为如此，所有的<code>Context</code>值共同构成了一颗代表了上下文全貌的树形结构。这棵树的树根（或者称上下文根节点）是一个已经在<code>context</code>包中预定义好的<code>Context</code>值，它是全局唯一的。通过调用<code>context.Background</code>函数，我们就可以获取到它（我在<code>coordinateWithContext</code>函数中就是这么做的）。</p><p>这里注意一下，这个上下文根节点仅仅是一个最基本的支点，它不提供任何额外的功能。也就是说，它既不可以被撤销（cancel），也不能携带任何数据。</p><p>除此之外，<code>context</code>包中还包含了四个用于繁衍<code>Context</code>值的函数，即：<code>WithCancel</code>、<code>WithDeadline</code>、<code>WithTimeout</code>和<code>WithValue</code>。</p><p>这些函数的第一个参数的类型都是<code>context.Context</code>，而名称都为<code>parent</code>。顾名思义，这个位置上的参数对应的都是它们将会产生的<code>Context</code>值的父值。</p><p><code>WithCancel</code>函数用于产生一个可撤销的<code>parent</code>的子值。在<code>coordinateWithContext</code>函数中，我通过调用该函数，获得了一个衍生自上下文根节点的<code>Context</code>值，和一个用于触发撤销信号的函数。</p><p>而<code>WithDeadline</code>函数和<code>WithTimeout</code>函数则都可以被用来产生一个会定时撤销的<code>parent</code>的子值。至于<code>WithValue</code>函数，我们可以通过调用它，产生一个会携带额外数据的<code>parent</code>的子值。</p><p>到这里，我们已经对<code>context</code>包中的函数和<code>Context</code>类型有了一个基本的认识了。不过这还不够，我们再来扩展一下。</p><h2>知识扩展</h2><h3>问题1：“可撤销的”在<code>context</code>包中代表着什么？“撤销”一个<code>Context</code>值又意味着什么？</h3><p>我相信很多初识<code>context</code>包的Go程序开发者，都会有这样的疑问。确实，“可撤销的”（cancelable）这个词在这里是比较抽象的，很容易让人迷惑。我这里再来解释一下。</p><p>这需要从<code>Context</code>类型的声明讲起。这个接口中有两个方法与“撤销”息息相关。<code>Done</code>方法会返回一个元素类型为<code>struct{}</code>的接收通道。不过，这个接收通道的用途并不是传递元素值，而是让调用方去感知“撤销”当前<code>Context</code>值的那个信号。</p><p>一旦当前的<code>Context</code>值被撤销，这里的接收通道就会被立即关闭。我们都知道，对于一个未包含任何元素值的通道来说，它的关闭会使任何针对它的接收操作立即结束。</p><p>正因为如此，在<code>coordinateWithContext</code>函数中，基于调用表达式<code>cxt.Done()</code>的接收操作，才能够起到感知撤销信号的作用。</p><p>除了让<code>Context</code>值的使用方感知到撤销信号，让它们得到“撤销”的具体原因，有时也是很有必要的。后者即是<code>Context</code>类型的<code>Err</code>方法的作用。该方法的结果是<code>error</code>类型的，并且其值只可能等于<code>context.Canceled</code>变量的值，或者<code>context.DeadlineExceeded</code>变量的值。</p><p>前者用于表示手动撤销，而后者则代表：由于我们给定的过期时间已到，而导致的撤销。</p><p>你可能已经感觉到了，对于<code>Context</code>值来说，“撤销”这个词如果当名词讲，指的其实就是被用来表达“撤销”状态的信号；如果当动词讲，指的就是对撤销信号的传达；而“可撤销的”指的则是具有传达这种撤销信号的能力。</p><p>我在前面讲过，当我们通过调用<code>context.WithCancel</code>函数产生一个可撤销的<code>Context</code>值时，还会获得一个用于触发撤销信号的函数。</p><p>通过调用这个函数，我们就可以触发针对这个<code>Context</code>值的撤销信号。一旦触发，撤销信号就会立即被传达给这个<code>Context</code>值，并由它的<code>Done</code>方法的结果值（一个接收通道）表达出来。</p><p>撤销函数只负责触发信号，而对应的可撤销的<code>Context</code>值也只负责传达信号，它们都不会去管后边具体的“撤销”操作。实际上，我们的代码可以在感知到撤销信号之后，进行任意的操作，<code>Context</code>值对此并没有任何的约束。</p><p>最后，若再深究的话，这里的“撤销”最原始的含义其实就是，终止程序针对某种请求（比如HTTP请求）的响应，或者取消对某种指令（比如SQL指令）的处理。这也是Go语言团队在创建<code>context</code>代码包，和<code>Context</code>类型时的初衷。</p><p>如果我们去查看<code>net</code>包和<code>database/sql</code>包的API和源码的话，就可以了解它们在这方面的典型应用。</p><h3>问题2：撤销信号是如何在上下文树中传播的？</h3><p>我在前面讲了，<code>context</code>包中包含了四个用于繁衍<code>Context</code>值的函数。其中的<code>WithCancel</code>、<code>WithDeadline</code>和<code>WithTimeout</code>都是被用来基于给定的<code>Context</code>值产生可撤销的子值的。</p><p><code>context</code>包的<code>WithCancel</code>函数在被调用后会产生两个结果值。第一个结果值就是那个可撤销的<code>Context</code>值，而第二个结果值则是用于触发撤销信号的函数。</p><p>在撤销函数被调用之后，对应的<code>Context</code>值会先关闭它内部的接收通道，也就是它的<code>Done</code>方法会返回的那个通道。</p><p>然后，它会向它的所有子值（或者说子节点）传达撤销信号。这些子值会如法炮制，把撤销信号继续传播下去。最后，这个<code>Context</code>值会断开它与其父值之间的关联。</p><p><img src="https://static001.geekbang.org/resource/image/a8/9e/a801f8f2b5e89017ec2857bc1815fc9e.png?wh=1112*647" alt=""></p><p>（在上下文树中传播撤销信号）</p><p>我们通过调用<code>context</code>包的<code>WithDeadline</code>函数或者<code>WithTimeout</code>函数生成的<code>Context</code>值也是可撤销的。它们不但可以被手动撤销，还会依据在生成时被给定的过期时间，自动地进行定时撤销。这里定时撤销的功能是借助它们内部的计时器来实现的。</p><p>当过期时间到达时，这两种<code>Context</code>值的行为与<code>Context</code>值被手动撤销时的行为是几乎一致的，只不过前者会在最后停止并释放掉其内部的计时器。</p><p>最后要注意，通过调用<code>context.WithValue</code>函数得到的<code>Context</code>值是不可撤销的。撤销信号在被传播时，若遇到它们则会直接跨过，并试图将信号直接传给它们的子值。</p><h3>问题 3：怎样通过<code>Context</code>值携带数据？怎样从中获取数据？</h3><p>既然谈到了<code>context</code>包的<code>WithValue</code>函数，我们就来说说<code>Context</code>值携带数据的方式。</p><p><code>WithValue</code>函数在产生新的<code>Context</code>值（以下简称含数据的<code>Context</code>值）的时候需要三个参数，即：父值、键和值。与“字典对于键的约束”类似，这里键的类型必须是可判等的。</p><p>原因很简单，当我们从中获取数据的时候，它需要根据给定的键来查找对应的值。不过，这种<code>Context</code>值并不是用字典来存储键和值的，后两者只是被简单地存储在前者的相应字段中而已。</p><p><code>Context</code>类型的<code>Value</code>方法就是被用来获取数据的。在我们调用含数据的<code>Context</code>值的<code>Value</code>方法时，它会先判断给定的键，是否与当前值中存储的键相等，如果相等就把该值中存储的值直接返回，否则就到其父值中继续查找。</p><p>如果其父值中仍然未存储相等的键，那么该方法就会沿着上下文根节点的方向一路查找下去。</p><p>注意，除了含数据的<code>Context</code>值以外，其他几种<code>Context</code>值都是无法携带数据的。因此，<code>Context</code>值的<code>Value</code>方法在沿路查找的时候，会直接跨过那几种值。</p><p>如果我们调用的<code>Value</code>方法的所属值本身就是不含数据的，那么实际调用的就将会是其父辈或祖辈的<code>Value</code>方法。这是由于这几种<code>Context</code>值的实际类型，都属于结构体类型，并且它们都是通过“将其父值嵌入到自身”，来表达父子关系的。</p><p>最后，提醒一下，<code>Context</code>接口并没有提供改变数据的方法。因此，在通常情况下，我们只能通过在上下文树中添加含数据的<code>Context</code>值来存储新的数据，或者通过撤销此种值的父值丢弃掉相应的数据。如果你存储在这里的数据可以从外部改变，那么必须自行保证安全。</p><h2>总结</h2><p>我们今天主要讨论的是<code>context</code>包中的函数和<code>Context</code>类型。该包中的函数都是用于产生新的<code>Context</code>类型值的。<code>Context</code>类型是一个可以帮助我们实现多goroutine协作流程的同步工具。不但如此，我们还可以通过此类型的值传达撤销信号或传递数据。</p><p><code>Context</code>类型的实际值大体上分为三种，即：根<code>Context</code>值、可撤销的<code>Context</code>值和含数据的<code>Context</code>值。所有的<code>Context</code>值共同构成了一颗上下文树。这棵树的作用域是全局的，而根<code>Context</code>值就是这棵树的根。它是全局唯一的，并且不提供任何额外的功能。</p><p>可撤销的<code>Context</code>值又分为：只可手动撤销的<code>Context</code>值，和可以定时撤销的<code>Context</code>值。</p><p>我们可以通过生成它们时得到的撤销函数来对其进行手动的撤销。对于后者，定时撤销的时间必须在生成时就完全确定，并且不能更改。不过，我们可以在过期时间达到之前，对其进行手动的撤销。</p><p>一旦撤销函数被调用，撤销信号就会立即被传达给对应的<code>Context</code>值，并由该值的<code>Done</code>方法返回的接收通道表达出来。</p><p>“撤销”这个操作是<code>Context</code>值能够协调多个goroutine的关键所在。撤销信号总是会沿着上下文树叶子节点的方向传播开来。</p><p>含数据的<code>Context</code>值可以携带数据。每个值都可以存储一对键和值。在我们调用它的<code>Value</code>方法的时候，它会沿着上下文树的根节点的方向逐个值的进行查找。如果发现相等的键，它就会立即返回对应的值，否则将在最后返回<code>nil</code>。</p><p>含数据的<code>Context</code>值不能被撤销，而可撤销的<code>Context</code>值又无法携带数据。但是，由于它们共同组成了一个有机的整体（即上下文树），所以在功能上要比<code>sync.WaitGroup</code>强大得多。</p><h2>思考题</h2><p>今天的思考题是：<code>Context</code>值在传达撤销信号的时候是广度优先的，还是深度优先的？其优势和劣势都是什么？</p><p><a href="https://github.com/hyper0x/Golang_Puzzlers">戳此查看Go语言专栏文章配套详细代码。</a></p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/80/b5/f59d92f1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Cloud</span>
  </div>
  <div class="_2_QraFYR_0">还没用过context包的我看得一愣一愣的</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-24 10:13:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/51/cb/c8b42257.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Spike</span>
  </div>
  <div class="_2_QraFYR_0">https:&#47;&#47;blog.golang.org&#47;pipelines<br>https:&#47;&#47;blog.golang.org&#47;context<br>要了解context的来源和用法，建议先阅读官网的这两篇blog<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-05 17:34:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/15/51/223b6e04.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>勉才</span>
  </div>
  <div class="_2_QraFYR_0">context 树不难理解，context.Background 是根节点，但它是个空的根节点，然后通过它我们可以创建出自己的 context 节点，在这个节点之下又可以创建出新的 context 节点。看了 context 的实现，其实它就是通过一个 channel 来实现，cancel() 就是关闭该管道，context.Done() 在 channel 关闭后，会返回。替我们造的轮子主要实现两个功能：1. 加锁，实现线程安全；2. cancel() 会将本节点，及子节点的 channel 都关闭。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-01 22:48:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/3d/6e/60680aa4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Li Yao</span>
  </div>
  <div class="_2_QraFYR_0">如果能举一个实际的应用场景就更好了，这篇看不太懂用途</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-16 20:59:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/b4/30/d929bf57.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>丁香与茉莉</span>
  </div>
  <div class="_2_QraFYR_0">http:&#47;&#47;www.flysnow.org&#47;2017&#47;05&#47;12&#47;go-in-action-go-context.html</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-30 15:25:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/ac/95/9b3e3859.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Timo</span>
  </div>
  <div class="_2_QraFYR_0">Context 使用原则<br>不要把Context放在结构体中，要以参数的方式传递<br>以Context作为参数的函数方法，应该把Context作为第一个参数，放在第一位。<br>给一个函数方法传递Context的时候，不要传递nil，如果不知道传递什么，就使用context.TODO<br>Context的Value相关方法应该传递必须的数据，不要什么数据都使用这个传递<br>Context是线程安全的，可以放心的在多个goroutine中传递</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-28 15:13:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/7e/05/431d380f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>拂尘</span>
  </div>
  <div class="_2_QraFYR_0">@郝老师 有几点疑问烦劳回答下，谢谢！<br>1、在coordinateWithContext的例子中，总共有12个子goroutine被创建，第12个即最后一个子goroutine在运行结束时，会通过计算defer表达式从而触发cancelFunc的调用，从而通知主goroutine结束在ctx.Done上获取通道的接收等待。我的问题是，在第12个子goroutine计算defer表达式的时候，会不会存在if条件不满足，未执行到cancelFunc的情况？或者说，在此时，第1到第11的子goroutine中，会存在自旋cas未执行完的情况吗？如果这种情况有，是否会导致主goroutine永远阻塞的情况？<br>2、在撤销函数被调用的时候，在当前context上，通过contex.Done获取的通道会马上感知到吗？还是会同步等待，使撤销信号在当前context的所有subtree上的所有context传播完成后，再感知到？还是有其他情况？<br>3、WithDeadline和WithTimeout的区别是什么？具体说，deadline是针对某个具体时间，而timeout是针对当前时间的延时来定义自动撤销时间吗？<br>感谢回复！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 首先要明确：<br>1. coordinateWithContext函数里的for语句是为了启用12个goroutine，但是这些go函数谁先执行谁后执行与这些goroutine的启用顺序而关。<br>2. addNum函数里的defer函数会在它后面的for语句执行完毕之后才开始执行。<br><br>所以，第一个问题的答案是：不会。因为总会有一个addNum函数把num的值变成12，然后执行deferFunc函数，并由于num==12而执行cancelFunc函数。<br><br>另外，这些go函数在调用addNum函数时会碰到自旋的情况（程序会打印出来），但是绝不会造成死锁，因为这些AddNum函数中的CAS操作早晚会执行成功。原子的Load+CAS外加for语句，相当于乐观锁，而且它们的操作都很“规矩”（都只是+1而已）。你也可以理解为把这12次“累加”串行化了，只不过大家是并发的，都在寻找自己“累加”成功的机会。<br><br>第二个问题：当前context会马上感知到，但前提是它是可撤销的。通过WithValue函数构造出来的context只会传递不会感知（通过匿名字段实现的）。<br><br>第三个问题：你已经把答案说出来了，我就不复述了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-05 05:51:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/aa/53/768aec0a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>郝林</span>
  </div>
  <div class="_2_QraFYR_0">我看不少读者都说写一篇难理解。可能的确如此，因为我假设你们已对context包有基本的了解。<br><br>不过没关系，你们有此方面的任何问题都可以通过以下三个途径与我讨论：<br><br>1. 直接在专栏文章下留言；<br>2. 在 GoHackers 微信群或者 BearyChat 中的 GoHackers 团队空间里艾特我；<br>3. 在知识星球中的 GoHackers VIP 社群里向我提问。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-17 12:44:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/18/07/9f5f5dd3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>憶海拾貝</span>
  </div>
  <div class="_2_QraFYR_0">服务间调用通常要传递上下文数据，用带值Context在每个函数间传递是一种方式，但从Python过来的我觉得这对代码侵入性蛮大的。请问go中还有什么更好的办法传递上下文数据呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-15 18:12:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/4a/d4/a3231668.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Shawn</span>
  </div>
  <div class="_2_QraFYR_0">看代码是深度优先，但是我自己写了demo，顺序是乱的，求老师讲解</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 打印出来的顺序不定是正常的，因为goroutine会被实时调度啊，打印出来的顺序不一定就是真实顺序。每填语句执行完都可能被调度。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-25 02:13:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q3auHgzwzM6iagw7ct4ca3niaSEFNicu2wy2KuCibO6eiaRzoRGJb50WTrbkKQib9mTArnTr8jJUazO9O2ibLZNfjjl35cfCHkBPs7N/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_f39659</span>
  </div>
  <div class="_2_QraFYR_0">根据这句话：“A great mental model of using Context is that it should flow through your program. Imagine a river or running water. This generally means that you don’t want to store it somewhere like in a struct. Nor do you want to keep it around any more than strictly needed. Context should be an interface that is passed from function to function down your call stack, augmented as needed.”<br><br>感觉上Context设计上更像是一个运行时的动态概念，它更像是代表传统call stack 层层镶套外加分叉关系的那颗树。代表着运行时态的调用树。所以它最好就是只存在于函数体的闭包之中，大家口口相传，“传男不传女”。“因为我调用你，所以我才把这个传给你，你可以传给你的子孙，但不要告诉别人！”。所以最好不要把它保存起来让旁人有机会看得到或用到。<br><br>楼上有人提到这种风格的入侵性，我能理解你的感觉。但以我以前玩Node.js中的cls (continuation local storage)踩过的那些坑来看，我宁愿两害相权取其轻。这种入侵性至少是可控的，显式的。同步编程的世界我们只需要TLS(Thread local storage)就好了，但对应的异步编程的世界里玩cls很难搞的。在我来看，Context显然是踩过那些坑的老鸟搞出来的。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-08 14:57:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/1a/3c/60b12211.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>mclee</span>
  </div>
  <div class="_2_QraFYR_0"><br>实测了下，context.WithValue 得到的新的 ctx 当其 parent context cancle 时也能收到 done 信号啊，并不是文中说的那样会跳过！<br><br>package main<br><br>import (<br>	&quot;context&quot;<br>	&quot;fmt&quot;<br>	&quot;time&quot;<br>)<br><br>func main() {<br>	ctx1, cancelFun := context.WithCancel(context.Background())<br>	ctx2 := context.WithValue(ctx1, &quot;&quot;, &quot;&quot;)<br>	ctx3, _ := context.WithCancel(ctx1)<br><br>	go watch(ctx1, &quot;ctx1&quot;)<br>	go watch(ctx2, &quot;ctx2&quot;)<br>	go watch(ctx3, &quot;ctx3&quot;)<br><br>	time.Sleep(2 * time.Second)<br>	fmt.Println(&quot;可以了，通知监控停止&quot;)<br>	cancelFun()<br><br>	&#47;&#47;为了检测监控过是否停止，如果没有监控输出，就表示停止了<br>	time.Sleep(5 * time.Second)<br><br>}<br><br>func watch(ctx context.Context, name string) {<br>	for {<br>		select {<br>		case &lt;-ctx.Done():<br>			fmt.Println(name,&quot;监控退出，停止了...&quot;)<br>			return<br>		default:<br>			fmt.Println(name,&quot;goroutine监控中...&quot;)<br>			time.Sleep(2 * time.Second)<br>		}<br>	}<br>}<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我说的跳过是源码级别的跳过，是跳过value节点直接传到它下级的节点，因为value节点本身是没有timeout机制，无需让cancel信号在那里发挥什么作用。<br><br>在value节点上的Done()在源码级别实际上调用的并不是value节点自己的方法，而是它上级节点（甚至上上级）的方法。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-11 20:10:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/6d/89/14031273.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Direction</span>
  </div>
  <div class="_2_QraFYR_0">在 Go 服务中，每个传入的请求都在它自己的 goroutine 中处理。请求处理程序通常启动额外的 goroutine 来访问后端，如数据库和 RPC 服务。<br>  处理请求的 goroutine 集通常需要访问特定于请求的值，例如最终用户的身份、授权令牌和请求的截止日期。当一个请求被取消或超时时（cancelFunc() or WithTimeout()），处理该请求的所<br>  有 goroutines 应该快速退出(ctx.Done（）)，以便系统可以回收（reclaim）它们正在使用的任何资源。<br><br>感觉这个举例挺好的（参考至：https:&#47;&#47;blog.golang.org&#47;context）</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-22 21:03:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/15/0f/954be2db.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>茴香根</span>
  </div>
  <div class="_2_QraFYR_0">留言区很多人说Context 是深度优先，但是我在想每个goroutine 被调用的顺序都是不确定的，因此在编写goroutine 代码时，实际的撤销响应不能假定其父或子context 所在的goroutine一定先或者后结束。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，这涉及到两个方面，需要综合起来看。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-25 08:32:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/bd/68/3fd6428d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Cutler</span>
  </div>
  <div class="_2_QraFYR_0">cotext.backround()和cotext.todo()有什么区别</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 很明显，context.Background()返回的是全局的上下文根（我在文章中多次提到），context.TODO()返回的是空的上下文（表明应用的不确定性）。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-09 08:06:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/59/c0/6f02c096.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>属雨</span>
  </div>
  <div class="_2_QraFYR_0">深度优先，看func (c *cancelCtx) cancel(removeFromParent bool, err error)方法的源代码。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-25 12:57:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/87/8c/468aa27b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>HOPE</span>
  </div>
  <div class="_2_QraFYR_0">老师，请教个问题。我运行了多个goroutine，每个goroutine有不同的配置。我现在想修改运行时的goroutine的配置，使用context可以实现吗？或者可以用什么办法实现？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-12 14:57:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/65/14/2e4edf4e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>罗志洪</span>
  </div>
  <div class="_2_QraFYR_0">context树有点难理解。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-24 10:44:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/b1/20/8718252f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>鲲鹏飞九万里</span>
  </div>
  <div class="_2_QraFYR_0">老师，您还能看到我的留言吗，现在已经是2023年了。您看我下面的代码，比您的代码少了一句time.Sleep(time.Millisecond * 200)， 之后，打印的结果就是错的，只打印了12个数，您能给解释一下吗。（我运行环境是：go version go1.18.3 darwin&#47;amd64， 2.3 GHz 四核Intel Core i5）<br>func main() {<br>	&#47;&#47; coordinateWithWaitGroup()<br>	coordinateWithContext()<br>}<br><br>func coordinateWithContext() {<br>	total := 12<br>	var num int32<br>	fmt.Printf(&quot;The number: %d [with context.Context]\n&quot;, num)<br>	cxt, cancelFunc := context.WithCancel(context.Background())<br>	for i := 1; i &lt;= total; i++ {<br>		go addNum(&amp;num, i, func() {<br>			&#47;&#47; 如果所有的addNum函数都执行完毕，那么就立即分发子任务的goroutine<br>			&#47;&#47; 这里分发子任务的goroutine，就是执行 coordinateWithContext 函数的goroutine.<br>			if atomic.LoadInt32(&amp;num) == int32(total) {<br>				&#47;&#47; &lt;-cxt.Done() 针对该函数返回的通道进行接收操作。<br>				&#47;&#47; cancelFunc() 函数被调用，针对该通道的接收会马上结束。<br>				&#47;&#47; 所以，这样做就可以实现“等待所有的addNum函数都执行完毕”的功能<br>				cancelFunc()<br>			}<br>		})<br>	}<br>	&lt;-cxt.Done()<br>	fmt.Println(&quot;end.&quot;)<br>}<br><br>func addNum(numP *int32, id int, deferFunc func()) {<br>	defer func() {<br>		deferFunc()<br>	}()<br>	for i := 0; ; i++ {<br>		currNum := atomic.LoadInt32(numP)<br>		newNum := currNum + 1<br>		&#47;&#47; time.Sleep(time.Millisecond * 200)<br>		if atomic.CompareAndSwapInt32(numP, currNum, newNum) {<br>			fmt.Printf(&quot;The number: %d [%d-%d]\n&quot;, newNum, id, i)<br>			break<br>		} else {<br>			fmt.Printf(&quot;The CAS option failed. [%d-%d]\n&quot;, id, i)<br>		}<br>	}<br>}<br><br>运行的结果为：<br>$ go run demo01.go<br>The number: 0 [with context.Context]<br>The number: 1 [12-0]<br>The number: 2 [1-0]<br>The number: 3 [2-0]<br>The number: 4 [3-0]<br>The number: 5 [4-0]<br>The number: 6 [9-0]<br>The number: 7 [10-0]<br>The number: 8 [11-0]<br>The number: 10 [6-0]<br>The number: 11 [5-0]<br>The number: 9 [8-0]<br>end.<br><br><br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 从 addNum 成功 +1，到它向标准输出打印内容，这中间是也是有延迟的，而且在那两行代码之间，Go语言的 runtime 也可能会进行调度。所以，部分数字的打印可能会出现乱序，或者直到程序运行结束也没来得及打印出来。加入 time.Sleep(time.Millisecond * 200) 就是为了能让 addNum 执行得慢一点，并且让 CAS 操作多重试几次，这样就更容易让数字的打印呈现出自然序。<br><br>要是没有 time.Sleep 的话，就有可能发生这种情况：某个 addNum 已经把 num 加到 12 了，并且已经执行了 cancelFunc，但是其他的 addNum 还没有来得及打印内容。<br><br>不过，即使在这种情况下，其他的数字都可能来不及打印，但是 12 肯定是会打印出来的。因为把数字添加到 12 的那个 addNum 一定就是执行 cancelFunc 函数的那个函数，并且打印语句肯定会在 cancelFunc 之前执行。12 可能会出现在“end.”之后，但是肯定是会打印出来的。<br><br>你这个打印结果是不是没有贴全？<br><br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-09 23:00:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/52/bb/225e70a6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>hunterlodge</span>
  </div>
  <div class="_2_QraFYR_0">“由于Context类型实际上是一个接口类型，而context包中实现该接口的所有私有类型，都是基于某个数据类型的指针类型，所以，如此传播并不会影响该类型值的功能和安全。”<br>请问老师，这句话中的「所以」二字怎么理解呢？指针不是会导致数据共享和竞争吗？为什么反而是安全的呢？谢谢！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你引用的这段话其实有两层含义：<br><br>1. 某个值的指针值的传递并不会导致源值（即指针指向的那个值）的拷贝。<br>2. 接口值里面其实会存储实际值的指针值，而不是实际值。所以在拷贝之后，其中的实际值依然是源值。<br><br>你看过sync包的文档吗？里面的同步工具大都不允许“使用之后的再传递”。<br><br>其实这种约束的原因就是，传递值会导致值的拷贝。如此一来，原值（即拷贝前的值）和拷贝值就是两个（几乎）不相干的值了。在两边分别对它们进行操作，也就起不到同步工具原有的作用了。<br><br>但如果是传递指针值的话，两边操作的仍然会是同一个源值，对吧？这就避免了同步工具的“操作失灵”的问题。<br><br>你说的“指针会导致数据共享和竞争”是另一个视角的问题。但是这两个问题的底层知识是一个，即：传递指针值的时候并不会拷贝源值，导致分别操作两个指针值相当于在操作同一个源值。<br><br>同步工具的内部自有避免竞争的手段，所以拷贝其指针值在大多数情况下是可以的。但是，最好还是不要拷贝它们的指针值（尤其是sync包中的那些工具），因为这样很可能会迷惑住读代码和后续写代码的人，导致理解错误或操作错误的概率很大。<br><br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-05 23:45:23</div>
  </div>
</div>
</div>
</li>
</ul>