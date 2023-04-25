<audio title="33 _ 临时对象池sync.Pool" src="https://static001.geekbang.org/resource/audio/a7/db/a7cf48a570b80a3e4026b572e4971fdb.mp3" controls="controls"></audio> 
<p>到目前为止，我们已经一起学习了Go语言标准库中最重要的那几个同步工具，这包括非常经典的互斥锁、读写锁、条件变量和原子操作，以及Go语言特有的几个同步工具：</p><ol>
<li><code>sync/atomic.Value</code>；</li>
<li><code>sync.Once</code>；</li>
<li><code>sync.WaitGroup</code></li>
<li><code>context.Context</code>。</li>
</ol><p>今天，我们来讲Go语言标准库中的另一个同步工具：<code>sync.Pool</code>。</p><p><code>sync.Pool</code>类型可以被称为临时对象池，它的值可以被用来存储临时的对象。与Go语言的很多同步工具一样，<code>sync.Pool</code>类型也属于结构体类型，它的值在被真正使用之后，就不应该再被复制了。</p><p>这里的“临时对象”的意思是：不需要持久使用的某一类值。这类值对于程序来说可有可无，但如果有的话会明显更好。它们的创建和销毁可以在任何时候发生，并且完全不会影响到程序的功能。</p><p>同时，它们也应该是无需被区分的，其中的任何一个值都可以代替另一个。如果你的某类值完全满足上述条件，那么你就可以把它们存储到临时对象池中。</p><p>你可能已经想到了，我们可以把临时对象池当作针对某种数据的缓存来用。实际上，在我看来，临时对象池最主要的用途就在于此。</p><p><code>sync.Pool</code>类型只有两个方法——<code>Put</code>和<code>Get</code>。Put用于在当前的池中存放临时对象，它接受一个<code>interface{}</code>类型的参数；而Get则被用于从当前的池中获取临时对象，它会返回一个<code>interface{}</code>类型的值。</p><!-- [[[read_end]]] --><p>更具体地说，这个类型的<code>Get</code>方法可能会从当前的池中删除掉任何一个值，然后把这个值作为结果返回。如果此时当前的池中没有任何值，那么这个方法就会使用当前池的<code>New</code>字段创建一个新值，并直接将其返回。</p><p><code>sync.Pool</code>类型的<code>New</code>字段代表着创建临时对象的函数。它的类型是没有参数但有唯一结果的函数类型，即：<code>func() interface{}</code>。</p><p>这个函数是<code>Get</code>方法最后的临时对象获取手段。<code>Get</code>方法如果到了最后，仍然无法获取到一个值，那么就会调用该函数。该函数的结果值并不会被存入当前的临时对象池中，而是直接返回给<code>Get</code>方法的调用方。</p><p>这里的<code>New</code>字段的实际值需要我们在初始化临时对象池的时候就给定。否则，在我们调用它的<code>Get</code>方法的时候就有可能会得到<code>nil</code>。所以，<code>sync.Pool</code>类型并不是开箱即用的。不过，这个类型也就只有这么一个公开的字段，因此初始化起来也并不麻烦。</p><p>举个例子。标准库代码包<code>fmt</code>就使用到了<code>sync.Pool</code>类型。这个包会创建一个用于缓存某类临时对象的<code>sync.Pool</code>类型值，并将这个值赋给一个名为<code>ppFree</code>的变量。这类临时对象可以识别、格式化和暂存需要打印的内容。</p><pre><code>var ppFree = sync.Pool{
 New: func() interface{} { return new(pp) },
}
</code></pre><p>临时对象池<code>ppFree</code>的<code>New</code>字段在被调用的时候，总是会返回一个全新的<code>pp</code>类型值的指针（即临时对象）。这就保证了<code>ppFree</code>的<code>Get</code>方法总能返回一个可以包含需要打印内容的值。</p><p><code>pp</code>类型是<code>fmt</code>包中的私有类型，它有很多实现了不同功能的方法。不过，这里的重点是，它的每一个值都是独立的、平等的和可重用的。</p><blockquote>
<p><span class="reference">更具体地说，这些对象既互不干扰，又不会受到外部状态的影响。它们几乎只针对某个需要打印内容的缓冲区而已。由于<code>fmt</code>包中的代码在真正使用这些临时对象之前，总是会先对其进行重置，所以它们并不在意取到的是哪一个临时对象。这就是临时对象的平等性的具体体现。</span></p>
</blockquote><p>另外，这些代码在使用完临时对象之后，都会先抹掉其中已缓冲的内容，然后再把它存放到<code>ppFree</code>中。这样就为重用这类临时对象做好了准备。</p><p>众所周知的<code>fmt.Println</code>、<code>fmt.Printf</code>等打印函数都是如此使用<code>ppFree</code>，以及其中的临时对象的。因此，在程序同时执行很多的打印函数调用的时候，<code>ppFree</code>可以及时地把它缓存的临时对象提供给它们，以加快执行的速度。</p><p>而当程序在一段时间内不再执行打印函数调用时，<code>ppFree</code>中的临时对象又能够被及时地清理掉，以节省内存空间。</p><p>显然，在这个维度上，临时对象池可以帮助程序实现可伸缩性。这就是它的最大价值。</p><p>我想，到了这里你已经清楚了临时对象池的基本功能、使用方式、适用场景和存在意义。我们下面来讨论一下它的一些内部机制，这样，我们就可以更好地利用它做更多的事。</p><p>首先，我来问你一个问题。这个问题很可能也是你想问的。今天的问题是：为什么说临时对象池中的值会被及时地清理掉？</p><p>这里的典型回答是：因为，Go语言运行时系统中的垃圾回收器，所以在每次开始执行之前，都会对所有已创建的临时对象池中的值进行全面地清除。</p><h2>问题解析</h2><p>我在前面已经向你讲述了临时对象会在什么时候被创建，下面我再来详细说说它会在什么时候被销毁。</p><p><code>sync</code>包在被初始化的时候，会向Go语言运行时系统注册一个函数，这个函数的功能就是清除所有已创建的临时对象池中的值。我们可以把它称为池清理函数。</p><p>一旦池清理函数被注册到了Go语言运行时系统，后者在每次即将执行垃圾回收时就都会执行前者。</p><p>另外，在<code>sync</code>包中还有一个包级私有的全局变量。这个变量代表了当前的程序中使用的所有临时对象池的汇总，它是元素类型为<code>*sync.Pool</code>的切片。我们可以称之为池汇总列表。</p><p>通常，在一个临时对象池的<code>Put</code>方法或<code>Get</code>方法第一次被调用的时候，这个池就会被添加到池汇总列表中。正因为如此，池清理函数总是能访问到所有正在被真正使用的临时对象池。</p><p>更具体地说，池清理函数会遍历池汇总列表。对于其中的每一个临时对象池，它都会先将池中所有的私有临时对象和共享临时对象列表都置为<code>nil</code>，然后再把这个池中的所有本地池列表都销毁掉。</p><p>最后，池清理函数会把池汇总列表重置为空的切片。如此一来，这些池中存储的临时对象就全部被清除干净了。</p><p>如果临时对象池以外的代码再无对它们的引用，那么在稍后的垃圾回收过程中，这些临时对象就会被当作垃圾销毁掉，它们占用的内存空间也会被回收以备他用。</p><p>以上，就是我对临时对象清理的进一步说明。首先需要记住的是，池清理函数和池汇总列表的含义，以及它们起到的关键作用。一旦理解了这些，那么在有人问到你这个问题的时候，你应该就可以从容地应对了。</p><p>不过，我们在这里还碰到了几个新的词，比如：私有临时对象、共享临时对象列表和本地池。这些都代表着什么呢？这就涉及了下面的问题。</p><h2>知识扩展</h2><h3>问题1：临时对象池存储值所用的数据结构是怎样的？</h3><p>在临时对象池中，有一个多层的数据结构。正因为有了它的存在，临时对象池才能够非常高效地存储大量的值。</p><p>这个数据结构的顶层，我们可以称之为本地池列表，不过更确切地说，它是一个数组。这个列表的长度，总是与Go语言调度器中的P的数量相同。</p><p>还记得吗？Go语言调度器中的P是processor的缩写，它指的是一种可以承载若干个G、且能够使这些G适时地与M进行对接，并得到真正运行的中介。</p><p>这里的G正是goroutine的缩写，而M则是machine的缩写，后者指代的是系统级的线程。正因为有了P的存在，G和M才能够进行灵活、高效的配对，从而实现强大的并发编程模型。</p><p>P存在的一个很重要的原因是为了分散并发程序的执行压力，而让临时对象池中的本地池列表的长度与P的数量相同的主要原因也是分散压力。这里所说的压力包括了存储和性能两个方面。在说明它们之前，我们先来探索一下临时对象池中的那个数据结构。</p><p>在本地池列表中的每个本地池都包含了三个字段（或者说组件），它们是：存储私有临时对象的字段<code>private</code>、代表了共享临时对象列表的字段<code>shared</code>，以及一个<code>sync.Mutex</code>类型的嵌入字段。</p><p><img src="https://static001.geekbang.org/resource/image/82/22/825cae64e0a879faba34c0a157b7ca22.png?wh=1168*729" alt=""><br>
<strong>sync.Pool中的本地池与各个G的对应关系</strong></p><p>实际上，每个本地池都对应着一个P。我们都知道，一个goroutine要想真正运行就必须先与某个P产生关联。也就是说，一个正在运行的goroutine必然会关联着某个P。</p><p>在程序调用临时对象池的<code>Put</code>方法或<code>Get</code>方法的时候，总会先试图从该临时对象池的本地池列表中，获取与之对应的本地池，依据的就是与当前的goroutine关联的那个P的ID。</p><p>换句话说，一个临时对象池的<code>Put</code>方法或<code>Get</code>方法会获取到哪一个本地池，完全取决于调用它的代码所在的goroutine关联的那个P。</p><p>既然说到了这里，那么紧接着就会有下面这个问题。</p><h3>问题 2：临时对象池是怎样利用内部数据结构来存取值的？</h3><p>临时对象池的<code>Put</code>方法总会先试图把新的临时对象，存储到对应的本地池的<code>private</code>字段中，以便在后面获取临时对象的时候，可以快速地拿到一个可用的值。</p><p>只有当这个<code>private</code>字段已经存有某个值时，该方法才会去访问本地池的<code>shared</code>字段。</p><p>相应的，临时对象池的<code>Get</code>方法，总会先试图从对应的本地池的<code>private</code>字段处获取一个临时对象。只有当这个<code>private</code>字段的值为<code>nil</code>时，它才会去访问本地池的<code>shared</code>字段。</p><p>一个本地池的<code>shared</code>字段原则上可以被任何goroutine中的代码访问到，不论这个goroutine关联的是哪一个P。这也是我把它叫做共享临时对象列表的原因。</p><p>相比之下，一个本地池的<code>private</code>字段，只可能被与之对应的那个P所关联的goroutine中的代码访问到，所以可以说，它是P级私有的。</p><p>以临时对象池的<code>Put</code>方法为例，它一旦发现对应的本地池的<code>private</code>字段已存有值，就会去访问这个本地池的<code>shared</code>字段。当然，由于<code>shared</code>字段是共享的，所以此时必须受到互斥锁的保护。</p><p>还记得本地池嵌入的那个<code>sync.Mutex</code>类型的字段吗？它就是这里用到的互斥锁，也就是说，本地池本身就拥有互斥锁的功能。<code>Put</code>方法会在互斥锁的保护下，把新的临时对象追加到共享临时对象列表的末尾。</p><p>相应的，临时对象池的<code>Get</code>方法在发现对应本地池的<code>private</code>字段未存有值时，也会去访问后者的<code>shared</code>字段。它会在互斥锁的保护下，试图把该共享临时对象列表中的最后一个元素值取出并作为结果。</p><p>不过，这里的共享临时对象列表也可能是空的，这可能是由于这个本地池中的所有临时对象都已经被取走了，也可能是当前的临时对象池刚被清理过。</p><p>无论原因是什么，<code>Get</code>方法都会去访问当前的临时对象池中的所有本地池，它会去逐个搜索它们的共享临时对象列表。</p><p>只要发现某个共享临时对象列表中包含元素值，它就会把该列表的最后一个元素值取出并作为结果返回。</p><p><img src="https://static001.geekbang.org/resource/image/df/21/df956fe29f35b41a14f941a9efd80d21.png?wh=1183*675" alt=""><br>
<strong>从sync.Pool中获取临时对象的步骤</strong></p><p>当然了，即使这样也可能无法拿到一个可用的临时对象，比如，在所有的临时对象池都刚被大清洗的情况下就会是如此。</p><p>这时，<code>Get</code>方法就会使出最后的手段——调用可创建临时对象的那个函数。还记得吗？这个函数是由临时对象池的<code>New</code>字段代表的，并且需要我们在初始化临时对象池的时候给定。如果这个字段的值是<code>nil</code>，那么<code>Get</code>方法此时也只能返回<code>nil</code>了。</p><p>以上，就是我对这个问题的较完整回答。</p><h2>总结</h2><p>今天，我们一起讨论了另一个比较有用的同步工具——<code>sync.Pool</code>类型，它的值被我称为临时对象池。</p><p>临时对象池有一个<code>New</code>字段，我们在初始化这个池的时候最好给定它。临时对象池还拥有两个方法，即：<code>Put</code>和<code>Get</code>，它们分别被用于向池中存放临时对象，和从池中获取临时对象。</p><p>临时对象池中存储的每一个值都应该是独立的、平等的和可重用的。我们应该既不用关心从池中拿到的是哪一个值，也不用在意这个值是否已经被使用过。</p><p>要完全做到这两点，可能会需要我们额外地写一些代码。不过，这个代码量应该是微乎其微的，就像<code>fmt</code>包对临时对象池的用法那样。所以，在选用临时对象池的时候，我们必须要把它将要存储的值的特性考虑在内。</p><p>在临时对象池的内部，有一个多层的数据结构支撑着对临时对象的存储。它的顶层是本地池列表，其中包含了与某个P对应的那些本地池，并且其长度与P的数量总是相同的。</p><p>在每个本地池中，都包含一个私有的临时对象和一个共享的临时对象列表。前者只能被其对应的P所关联的那个goroutine中的代码访问到，而后者却没有这个约束。从另一个角度讲，前者用于临时对象的快速存取，而后者则用于临时对象的池内共享。</p><p>正因为有了这样的数据结构，临时对象池才能够有效地分散存储压力和性能压力。同时，又因为临时对象池的<code>Get</code>方法对这个数据结构的妙用，才使得其中的临时对象能够被高效地利用。比如，该方法有时候会从其他的本地池的共享临时对象列表中，“偷取”一个临时对象。</p><p>这样的内部结构和存取方式，让临时对象池成为了一个特点鲜明的同步工具。它存储的临时对象都应该是拥有较长生命周期的值，并且，这些值不应该被某个goroutine中的代码长期的持有和使用。</p><p>因此，临时对象池非常适合用作针对某种数据的缓存。从某种角度讲，临时对象池可以帮助程序实现可伸缩性，这也正是它的最大价值。</p><h2>思考题</h2><p>今天的思考题是：怎样保证一个临时对象池中总有比较充足的临时对象？</p><p>请从临时对象池的初始化和方法调用两个方面作答。必要时可以参考<code>fmt</code>包以及demo70.go文件中使用临时对象池的方式。</p><p>感谢你的收听，我们下次再见。</p><p><a href="https://github.com/hyper0x/Golang_Puzzlers">戳此查看Go语言专栏文章配套详细代码。</a></p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/20/27/a6932fbe.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>虢國技醬</span>
  </div>
  <div class="_2_QraFYR_0">go1.13对本地池的shared共享列表做了存储结构变更,改为双向链表（在shared的头部存，尾部取），取消锁以提高性能</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-09 10:49:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/bc/15/23ce17f9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>数字记忆</span>
  </div>
  <div class="_2_QraFYR_0">这个代码很形象：<br><br>package main<br><br>import (<br>	&quot;fmt&quot;<br>	&quot;sync&quot;<br>	&quot;time&quot;<br>)<br><br>&#47;&#47; 一个[]byte的对象池，每个对象为一个[]byte<br>var bytePool = sync.Pool{<br>	New: func() interface{} {<br>		b := make([]byte, 1024)<br>		return &amp;b<br>	},<br>}<br><br>func main() {<br>	a := time.Now().Unix()<br>	fmt.Println(a)<br>	&#47;&#47; 不使用对象池<br>	for i := 0; i &lt; 1000000000; i++{<br>		obj := make([]byte,1024)<br>		_ = obj<br>	}<br>	b := time.Now().Unix()<br>	fmt.Println(b)<br>	&#47;&#47; 使用对象池<br>	for i := 0; i &lt; 1000000000; i++{<br>		obj := bytePool.Get().(*[]byte)<br>		_ = obj<br>		bytePool.Put(obj)<br>	}<br>	c := time.Now().Unix()<br>	fmt.Println(c)<br>	fmt.Println(&quot;without pool &quot;, b - a, &quot;s&quot;)<br>	fmt.Println(&quot;with    pool &quot;, c - b, &quot;s&quot;)<br>}<br><br>&#47;&#47;  run时禁用掉编译器优化，才会体现出有pool的优势<br>&#47;&#47;  go run -gcflags=&quot;-l -N&quot; testSyncPool1.go</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-20 11:43:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/33/7a/ac307bfc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>到不了的塔</span>
  </div>
  <div class="_2_QraFYR_0">临时对象池初始化时指定new字段对应的函数返回一个新建临时对象；<br>临时对象使用完毕时调用临时对象池的put方法，把该临时对象put回临时对象池中。<br>这样就能保证一个临时对象池中总有比较充足的临时对象。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-17 15:13:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/c4/c1/1118e24e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>赵赟</span>
  </div>
  <div class="_2_QraFYR_0">看了一下 1.14 的源码，那个锁现在是全局的了，即一个临时对象池中本地池列表中的所有本地池都共享一个锁，而不是每个本地池都有自己的锁。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这么说也不准确。看这行源码：<br><br>shared  poolChain<br><br>poolChain 这个类型的方法会动用原子操作。<br><br>再看这行源码：<br><br>var allPoolsMu Mutex<br><br>allPoolsMu 会保护单一程序中的所有 sync.Pool，而不是某一个 Pool 的本地池。<br><br>然而，sync.Pool 只会在获取 P 的 ID 以及查找对应的 本地池的时候才会动用 allPoolsMu，而在操作本地池的时候没有用。<br><br>所以，综上来看，本地池的操作已经通过更好的设计去掉了互斥锁，改为原子操作，同时仅在必要时（也就是定位本地池时）短暂动用互斥锁。<br><br>我没去看新 Pool 的性能测试，但是相信一定又有了不小的性能提升。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-09 15:29:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/04/fb/40f298bb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小罗希冀</span>
  </div>
  <div class="_2_QraFYR_0">请问一下老师, 如果syn.Pool广泛的应用场景是缓存, 那为什么不直接使用map缓存呢?这样岂不是更方便, 更快捷?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你这句话的前后逻辑不通啊，sync.Pool 和 map 是两个东西啊，它们的适用场景完全不一样啊。<br><br>sync.Pool 用于缓存“可交换”、“可遗失”的对象。可交换的意思是就是，我用对象 A 也可以，用对象 B 也可以，无所谓。可遗失的意思是，存在里边的对象没了就没了，无所谓，我再创建一个就是了。<br><br>map 如果用作缓存的话，其中的元素值是“不可交换”的，通常也是“不可遗失”的（或者说对遗失敏感的）。你思考一下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-26 23:02:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/74/57/7b828263.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张sir</span>
  </div>
  <div class="_2_QraFYR_0">还有一个问题，如果多goruntine同时申请临时对象池内资源，所有goruntine都可以同时获取到吗，还是只能有一个goruntine获取到，其它的goruntine都阻塞，直到这个goruntine释放完后才能使用</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我大概明白你的意思。这篇文章你可能还没有仔细看。<br><br>你需要先搞清楚（以下内容在文章里都有）：<br><br>在涉及到本地池的 shared 字段的时候会有锁，但是这种锁是分段锁，也就是说，每个本地池都会有自己的锁。<br><br>因此，在对应某个 P 的本地池的锁处于锁定状态的时候，所有正试图访问（不论是 Get 还是 Put）这个本地池的 goroutine 都会被阻塞。<br><br>一个临时对象池拥有的本地池的数量与 P 的数量相同。所以，即使有 goroutine 因此被阻塞，往往也只是少数。又因为分段锁的缘故，它们被锁住的时间一般也是很短暂的。<br><br>当你知道了这些，你就会明白，临时对象池在并发访问方面是很高效的。<br><br>再结合我在专栏里揭示的访问步骤和细节，你应该就可以搞懂你问的问题了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-21 11:17:28</div>
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
  <div class="_2_QraFYR_0">二刷</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-29 14:35:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/0a/0b/985d3800.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>郭星</span>
  </div>
  <div class="_2_QraFYR_0">&quot;在每个本地池中，都包含一个私有的临时对象和一个共享的临时对象列表。前者只能被其对应的 P 所关联的那个 goroutine 中的代码访问到，而后者却没有这个约束&quot;<br>对于private只能被当前协程才能访问,其他协程不能访问到private,这个应该怎么测试呢?<br>import (<br>	&quot;runtime&quot;<br>	&quot;sync&quot;<br>	&quot;testing&quot;<br>)<br><br>type cache struct {<br>	value int<br>}<br>func TestShareAndPrivate(t *testing.T) {<br>	p := sync.Pool{}<br>	&#47;&#47; 在主协程写入10<br>	p.Put(cache{value: 10})<br>	var wg sync.WaitGroup<br>	wg.Add(1)<br>	go func() {<br>		for i := 0; i &lt; 10; i++ {<br>			p.Put(cache{value: i})<br>		}<br>		wg.Done()<br>	}()<br>	wg.Wait()<br>	wg.Add(1)<br>	go func() {<br>		for true {<br>			v := p.Get()<br>			if v == nil {<br>				break<br>			}<br>			t.Log(v)<br>		}<br>		wg.Done()<br>	}()<br>	wg.Wait()<br>}<br>这段代码没有体现出来私有和共享的区别</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个测试比较困难，私有临时对象主要是为了加速对象的存取，但是临时对象池**并不保证**返回给我的对象是按照固定顺序的，你可以认为是随机的。<br><br>我们也没必要测试，知道有这样一个加速优化就好了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-03 17:34:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/a4/27/15e75982.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小袁</span>
  </div>
  <div class="_2_QraFYR_0">为啥本地池列表长度不是跟M一致，而是跟P一致？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 因为P是调度的核心啊，起到了衔接M和G的作用。P实际上也是“并发线”的根数，所以：若少于P数量则未充分利用并发机制，若多于P数量则加重了调度器的负担。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-13 11:26:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/e0/4b/bbb48b22.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>越努力丨越幸运</span>
  </div>
  <div class="_2_QraFYR_0">老师，当一个goroutine在访问某个临时对象池中一个本地池的shared字段时被锁住，此时另外一个goroutine访问临时对象池时，是会跳过这个本地池，去访问其他的本地池，还是说会被阻塞住？<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不会跳过，但是它用的不是锁，而是原子操作，因为存的都是指针。所以速度会非常快。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-19 19:06:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/12/70/10faf04b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Lywane</span>
  </div>
  <div class="_2_QraFYR_0">直到看到最近两三章，我才体会到，老师就是在讲源码啊！对着源码学习课程，对着课程学习源码。事半功倍！！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-31 18:12:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/75/aa/21275b9d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>闫飞</span>
  </div>
  <div class="_2_QraFYR_0">这里存放的临时对象是否是无状态，无唯一标识符的纯值对象? 对象的类型是否都是一样，还是说必须要用户自己做好具体类型的判定?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你放在一个池子里的实例最好是一个类型的，要不后面用的时候会很麻烦。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-17 08:20:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/44/84/25a17f05.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>苏安</span>
  </div>
  <div class="_2_QraFYR_0">老师，不知道还有几讲，最初的课程大纲有相关的拾遗章节，不知道后续的安排还有没？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我会讲完的，放心，预计45讲左右。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-26 09:00:08</div>
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
  <div class="_2_QraFYR_0">意外之喜，隔壁专栏鸟窝的《Go 并发编程实战课》对 sync.Pool 有了新的补充，这一讲有困惑的同学可以过去康康。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-21 11:33:35</div>
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
  <div class="_2_QraFYR_0">之前学习 go routine的时候 初次了解到这个p以为就是用来调度goroutine的  但是今天又讨论到这个p 这个P还关联到了临时对象池，这个临时对象池也涉及到被运行时系统所清理 所以我产生了以为 这个p时候就是运行时系统呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你要想了解Go语言的调度器，可与你参考我写的那本《Go并发编程实战》。（因为一句两句说不清楚）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-16 20:16:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c6/73/abb7bfe3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>疯琴</span>
  </div>
  <div class="_2_QraFYR_0">请问老师，demo70 的 37 行 return 后面没跟东西，是相当于 return nil 么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不是，是返回结果变量err的值。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-02 16:28:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/53/e3/39dcfb11.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>来碗绿豆汤</span>
  </div>
  <div class="_2_QraFYR_0">是不是临时对象池里面最多有2p个临时对象</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-28 20:27:12</div>
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
  <div class="_2_QraFYR_0">郝老师您好，你在article70.go 的示例中使用sync.Pool 的作用是啥呢，看不出来。你看：<br>func main() {<br>	&#47;&#47; buf := GetBuffer()<br>	&#47;&#47; defer buf.Free()<br>	buf := &amp;myBuffer{delimiter: delimiter}<br><br>在main函数中，我用`buf := &amp;myBuffer{delimiter: delimiter}`这行代码代替上面两行代码后，执行的效果是一样的。 article70.go 的示例，为啥要使用 sync.Pool 呢，麻烦老师进一步讲解下</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你每次从这个 sync.Pool 当中，用 Get 方法获取 Buffer，都会得到一个存在于该池中的、已就绪的 Buffer 实例，而 Put 一个 Buffer 实例，则会把它归还给这个 sync.Pool。这是这里演示的 sync.Pool 的基本用法。<br><br>这里为了简单和直观，我就没有划分成两个代码包，你可以想象一下，bufPool 变量、Buffer 接口、myBuffer 结构体、GetBuffer 函数同在某一个代码包里面，而 main 函数则存在于另一个代码包里。如此一来，直接写成 &amp;myBuffer{delimiter: delimiter} ，就等于把内部实现暴露给了外界，这样会在后期造成维护成本的增加，况且 myBuffer 还是包级私有的类型。<br><br>这里只是一个简单的演示，演示怎样利用 sync.Pool 存储一些可相互替换的同类对象，以供其他程序来高效的获取（Get）和归还（Put）对象。这个 sync.Pool 就是所谓的对象池，其内部实现得非常高效，用法也很简单，不是吗？我们就不用自己实现对象池了，用现成的就好了。<br><br>在这个示例中，还包含了一些“便捷函数”（如 GetBuffer 函数）和“便捷方法”（如 myBuffer 的 Free 方法），以方便外界更容易的使用这类 Buffer。<br><br>更具体地说，它隐藏了 bufPool 变量（假设该变量不在 main 函数所在的代码包里），并提供了简单调用一下就能获取一个 Buffer 实例的 GetBuffer 函数，以及用完某个 Buffer 就可以直接在它之上调用的 Free 方法（用于将该 Buffer 实例归还给 bufPool）。<br><br>总之，这是一种比较好的实现方式，提供给你们参考用的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-19 16:28:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/a9/b1/0d550474.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Haij!</span>
  </div>
  <div class="_2_QraFYR_0">fmt包为了识别、格式化和暂存需要打印的内容，定义了一个名为pp的结构体。调用不同的打印方法时，都需要一个pp的结构体介入逻辑进行处理；如果未使用sync.Pool，则每次均会通过new函数初始化pp类型的变量，这时会频繁申请内存。所以为避免每次需要时都调用pp的new方法申请内存，故基于sync.Pool创建一个临时对象池。当打印操作很活跃时，可以直接从池中获取pp结构体并使用；使用后抹取过程中的信息再存入。一方面可以利用“缓存”特性进行性能提升，避免频繁内存申请分配；另一方面可以借由sync.Pool初始时在运行时系统注册的cleanPool方法，及时清理空间，释放内存。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-25 01:06:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/24/27/7c/97b0c1dd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>林嘉裕</span>
  </div>
  <div class="_2_QraFYR_0">数组可以通过put(arr[:0])清空，如果是map呢？只能通过遍历？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你要是真是想清空，重新 Put 一个空的 map 就好了啊，切片的话也可以这样操作。只要这个 map 或者 slice 再没有别的代码引用它了，GC 就会进行回收了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-21 23:48:49</div>
  </div>
</div>
</div>
</li>
</ul>