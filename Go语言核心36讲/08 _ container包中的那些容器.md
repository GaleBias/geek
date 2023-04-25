<audio title="08 _ container包中的那些容器" src="https://static001.geekbang.org/resource/audio/7c/9f/7c4b9c8aaf83f93cb8f9c632807c449f.mp3" controls="controls"></audio> 
<p>我们在上次讨论了数组和切片，当我们提到数组的时候，往往会想起链表。那么Go语言的链表是什么样的呢？</p><p>Go语言的链表实现在标准库的<code>container/list</code>代码包中。这个代码包中有两个公开的程序实体——<code>List</code>和<code>Element</code>，List实现了一个双向链表（以下简称链表），而Element则代表了链表中元素的结构。</p><p><strong>那么，我今天的问题是：可以把自己生成的<code>Element</code>类型值传给链表吗？</strong></p><p>我们在这里用到了<code>List</code>的四种方法。</p><p><code>MoveBefore</code>方法和<code>MoveAfter</code>方法，它们分别用于把给定的元素移动到另一个元素的前面和后面。</p><p><code>MoveToFront</code>方法和<code>MoveToBack</code>方法，分别用于把给定的元素移动到链表的最前端和最后端。</p><p>在这些方法中，“给定的元素”都是<code>*Element</code>类型的，<code>*Element</code>类型是<code>Element</code>类型的指针类型，<code>*Element</code>的值就是元素的指针。</p><pre><code>func (l *List) MoveBefore(e, mark *Element)
func (l *List) MoveAfter(e, mark *Element)

func (l *List) MoveToFront(e *Element)
func (l *List) MoveToBack(e *Element)
</code></pre><p>具体问题是，如果我们自己生成这样的值，然后把它作为“给定的元素”传给链表的方法，那么会发生什么？链表会接受它吗？</p><p>这里，给出一个<strong>典型回答</strong>：不会接受，这些方法将不会对链表做出任何改动。因为我们自己生成的<code>Element</code>值并不在链表中，所以也就谈不上“在链表中移动元素”。更何况链表不允许我们把自己生成的<code>Element</code>值插入其中。</p><!-- [[[read_end]]] --><h2>问题解析</h2><p>在<code>List</code>包含的方法中，用于插入新元素的那些方法都只接受<code>interface{}</code>类型的值。这些方法在内部会使用<code>Element</code>值，包装接收到的新元素。</p><p>这样做正是为了避免直接使用我们自己生成的元素，主要原因是避免链表的内部关联，遭到外界破坏，这对于链表本身以及我们这些使用者来说都是有益的。</p><p><code>List</code>的方法还有下面这几种：</p><p><code>Front</code>和<code>Back</code>方法分别用于获取链表中最前端和最后端的元素，<br>
<code>InsertBefore</code>和<code>InsertAfter</code>方法分别用于在指定的元素之前和之后插入新元素，<code>PushFront</code>和<code>PushBack</code>方法则分别用于在链表的最前端和最后端插入新元素。</p><pre><code>func (l *List) Front() *Element
func (l *List) Back() *Element

func (l *List) InsertBefore(v interface{}, mark *Element) *Element
func (l *List) InsertAfter(v interface{}, mark *Element) *Element

func (l *List) PushFront(v interface{}) *Element
func (l *List) PushBack(v interface{}) *Element
</code></pre><p>这些方法都会把一个<code>Element</code>值的指针作为结果返回，它们就是链表留给我们的安全“接口”。拿到这些内部元素的指针，我们就可以去调用前面提到的用于移动元素的方法了。</p><p><strong>知识扩展</strong></p><p><strong>1. 问题：为什么链表可以做到开箱即用？</strong></p><p><code>List</code>和<code>Element</code>都是结构体类型。结构体类型有一个特点，那就是它们的零值都会是拥有特定结构，但是没有任何定制化内容的值，相当于一个空壳。值中的字段也都会被分别赋予各自类型的零值。</p><blockquote>
<p><span class="reference">广义来讲，所谓的零值就是只做了声明，但还未做初始化的变量被给予的缺省值。每个类型的零值都会依据该类型的特性而被设定。</span></p>
<p><span class="reference">比如，经过语句<code>var a [2]int</code>声明的变量<code>a</code>的值，将会是一个包含了两个<code>0</code>的整数数组。又比如，经过语句<code>var s []int</code>声明的变量<code>s</code>的值将会是一个<code>[]int</code>类型的、值为<code>nil</code>的切片。</span></p>
</blockquote><p>那么经过语句<code>var l list.List</code>声明的变量<code>l</code>的值将会是什么呢？[1] 这个零值将会是一个长度为<code>0</code>的链表。这个链表持有的根元素也将会是一个空壳，其中只会包含缺省的内容。那这样的链表我们可以直接拿来使用吗？</p><p>答案是，可以的。这被称为“开箱即用”。Go语言标准库中很多结构体类型的程序实体都做到了开箱即用。这也是在编写可供别人使用的代码包（或者说程序库）时，我们推荐遵循的最佳实践之一。那么，语句<code>var l list.List</code>声明的链表<code>l</code>可以直接使用，这是怎么做到的呢？</p><p>关键在于它的“延迟初始化”机制。</p><p>所谓的<strong>延迟初始化</strong>，你可以理解为把初始化操作延后，仅在实际需要的时候才进行。延迟初始化的优点在于“延后”，它可以分散初始化操作带来的计算量和存储空间消耗。</p><p>例如，如果我们需要集中声明非常多的大容量切片的话，那么那时的CPU和内存空间的使用量肯定都会一个激增，并且只有设法让其中的切片及其底层数组被回收，内存使用量才会有所降低。</p><p>如果数组是可以被延迟初始化的，那么计算量和存储空间的压力就可以被分散到实际使用它们的时候。这些数组被实际使用的时间越分散，延迟初始化带来的优势就会越明显。</p><blockquote>
<p><span class="reference">实际上，Go语言的切片就起到了延迟初始化其底层数组的作用，你可以想一想为什么会这么说的理由。</span></p>
<p><span class="reference">延迟初始化的缺点恰恰也在于“延后”。你可以想象一下，如果我在调用链表的每个方法的时候，它们都需要先去判断链表是否已经被初始化，那这也会是一个计算量上的浪费。在这些方法被非常频繁地调用的情况下，这种浪费的影响就开始显现了，程序的性能将会降低。</span></p>
</blockquote><p>在这里的链表实现中，一些方法是无需对是否初始化做判断的。比如<code>Front</code>方法和<code>Back</code>方法，一旦发现链表的长度为<code>0</code>,直接返回<code>nil</code>就好了。</p><p>又比如，在用于删除元素、移动元素，以及一些用于插入元素的方法中，只要判断一下传入的元素中指向所属链表的指针，是否与当前链表的指针相等就可以了。</p><p>如果不相等，就一定说明传入的元素不是这个链表中的，后续的操作就不用做了。反之，就一定说明这个链表已经被初始化了。</p><p>原因在于，链表的<code>PushFront</code>方法、<code>PushBack</code>方法、<code>PushBackList</code>方法以及<code>PushFrontList</code>方法总会先判断链表的状态，并在必要时进行初始化，这就是延迟初始化。</p><p>而且，我们在向一个空的链表中添加新元素的时候，肯定会调用这四个方法中的一个，这时新元素中指向所属链表的指针，一定会被设定为当前链表的指针。所以，指针相等是链表已经初始化的充分必要条件。</p><p>明白了吗？<code>List</code>利用了自身以及<code>Element</code>在结构上的特点，巧妙地平衡了延迟初始化的优缺点，使得链表可以开箱即用，并且在性能上可以达到最优。</p><p><strong>问题 2：<code>Ring</code>与<code>List</code>的区别在哪儿？</strong></p><p><code>container/ring</code>包中的<code>Ring</code>类型实现的是一个循环链表，也就是我们俗称的环。其实<code>List</code>在内部就是一个循环链表。它的根元素永远不会持有任何实际的元素值，而该元素的存在就是为了连接这个循环链表的首尾两端。</p><p>所以也可以说，<code>List</code>的零值是一个只包含了根元素，但不包含任何实际元素值的空链表。那么，既然<code>Ring</code>和<code>List</code>在本质上都是循环链表，那它们到底有什么不同呢？</p><p>最主要的不同有下面几种。</p><ol>
<li><code>Ring</code>类型的数据结构仅由它自身即可代表，而<code>List</code>类型则需要由它以及<code>Element</code>类型联合表示。这是表示方式上的不同，也是结构复杂度上的不同。</li>
<li>一个<code>Ring</code>类型的值严格来讲，只代表了其所属的循环链表中的一个元素，而一个<code>List</code>类型的值则代表了一个完整的链表。这是表示维度上的不同。</li>
<li>在创建并初始化一个<code>Ring</code>值的时候，我们可以指定它包含的元素的数量，但是对于一个<code>List</code>值来说却不能这样做（也没有必要这样做）。循环链表一旦被创建，其长度是不可变的。这是两个代码包中的<code>New</code>函数在功能上的不同，也是两个类型在初始化值方面的第一个不同。</li>
<li>仅通过<code>var r ring.Ring</code>语句声明的<code>r</code>将会是一个长度为<code>1</code>的循环链表，而<code>List</code>类型的零值则是一个长度为<code>0</code>的链表。别忘了<code>List</code>中的根元素不会持有实际元素值，因此计算长度时不会包含它。这是两个类型在初始化值方面的第二个不同。</li>
<li><code>Ring</code>值的<code>Len</code>方法的算法复杂度是O(N)的，而<code>List</code>值的<code>Len</code>方法的算法复杂度则是O(1)的。这是两者在性能方面最显而易见的差别。</li>
</ol><p>其他的不同基本上都是方法方面的了。比如，循环链表也有用于插入、移动或删除元素的方法，不过用起来都显得更抽象一些，等等。</p><p><strong>总结</strong></p><p>我们今天主要讨论了<code>container/list</code>包中的链表实现。我们详细讲解了链表的一些主要的使用技巧和实现特点。由于此链表实现在内部就是一个循环链表，所以我们还把它与<code>container/ring</code>包中的循环链表实现做了一番比较，包括结构、初始化以及性能方面。</p><p><strong>思考题</strong></p><ol>
<li><code>container/ring</code>包中的循环链表的适用场景都有哪些？</li>
<li>你使用过<code>container/heap</code>包中的堆吗？它的适用场景又有哪些呢？</li>
</ol><p>在这里，我们先不求对它们的实现了如指掌，能用对、用好才是我们进阶之前的第一步。好了，感谢你的收听，我们下次再见。</p><hr></hr><p>[1]：<code>List</code>这个结构体类型有两个字段，一个是<code>Element</code>类型的字段<code>root</code>，另一个是<code>int</code>类型的字段<code>len</code>。顾名思义，前者代表的就是那个根元素，而后者用于存储链表的长度。注意，它们都是包级私有的，也就是说使用者无法查看和修改它们。</p><p>像前面那样声明的<code>l</code>，其字段<code>root</code>和<code>len</code>都会被赋予相应的零值。<code>len</code>的零值是<code>0</code>，正好可以表明该链表还未包含任何元素。由于<code>root</code>是<code>Element</code>类型的，所以它的零值就是该类型的空壳，用字面量表示的话就是<code>Element{}</code>。</p><p><code>Element</code>类型包含了几个包级私有的字段，分别用于存储前一个元素、后一个元素以及所属链表的指针值。另外还有一个名叫<code>Value</code>的公开的字段，该字段的作用就是持有元素的实际值，它是<code>interface{}</code>类型的。在<code>Element</code>类型的零值中，这些字段的值都会是<code>nil</code>。</p><h2>参考阅读</h2><h3>切片与数组的比较</h3><p>切片本身有着占用内存少和创建便捷等特点，但它的本质上还是数组。切片的一大好处是可以让我们通过窗口快速地定位并获取，或者修改底层数组中的元素。</p><p>不过，当我们想删除切片中的元素的时候就没那么简单了。元素复制一般是免不了的，就算只删除一个元素，有时也会造成大量元素的移动。这时还要注意空出的元素槽位的“清空”，否则很可能会造成内存泄漏。</p><p>另一方面，在切片被频繁“扩容”的情况下，新的底层数组会不断产生，这时内存分配的量以及元素复制的次数可能就很可观了，这肯定会对程序的性能产生负面的影响。</p><p>尤其是当我们没有一个合理、有效的”缩容“策略的时候，旧的底层数组无法被回收，新的底层数组中也会有大量无用的元素槽位。过度的内存浪费不但会降低程序的性能，还可能会使内存溢出并导致程序崩溃。</p><p>由此可见，正确地使用切片是多么的重要。不过，一个更重要的事实是，任何数据结构都不是银弹。不是吗？数组的自身特点和适用场景都非常鲜明，切片也是一样。它们都是Go语言原生的数据结构，使用起来也都很方便.不过，你的集合类工具箱中不应该只有它们。这就是我们使用链表的原因。</p><p>不过，对比来看，一个链表所占用的内存空间，往往要比包含相同元素的数组所占内存大得多。这是由于链表的元素并不是连续存储的，所以相邻的元素之间需要互相保存对方的指针。不但如此，每个元素还要存有它所属链表的指针。</p><p>有了这些关联，链表的结构反倒更简单了。它只持有头部元素（或称为根元素）基本上就可以了。当然了，为了防止不必要的遍历和计算，链表的长度记录在内也是必须的。</p><p><a href="https://github.com/hyper0x/Golang_Puzzlers">戳此查看Go语言专栏文章配套详细代码。</a></p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/50/99/44378317.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>李皮皮皮皮皮</span>
  </div>
  <div class="_2_QraFYR_0">1.list可以作为queue和<br>stack的基础数据结构<br>2.ring可以用来保存固定数量的元素，例如保存最近100条日志，用户最近10次操作<br>3.heap可以用来排序。游戏编程中是一种高效的定时器实现方案</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-29 19:55:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/6b/23/73f18275.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>陌上人 .</span>
  </div>
  <div class="_2_QraFYR_0">老师,之后的课可不可以多加一些图形解释,原理性的知识只用文字确实有些晦涩难懂</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-28 10:23:01</div>
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
  <div class="_2_QraFYR_0">关于 container包中的链表list 和环ring的知识总结<br>按我的思考<br>1. 为什么go语言会出现list这个包<br>    首先一个语言或者新技术的出现肯定是为了解决一些疑难杂症 在go语言中数组的特点非常鲜明 固定不可变 访问方便,但是如果不适合动态增长，所以出现了slice切片 切片是对数组的一层封装为了解决数组动态扩容问题， 但是实际上底层依赖的还是数组，但是问题来了 如果slice切片 在添加或者删除元素的时候如果没一个好的策略 扩容或者缩容过后旧的切片没有释放 则会造成内存泄漏 就是c语言的malloc 久而久之 内存越来越少 程序就会崩溃, 而List的出现就是为了解决动态扩容或者缩容的后遗症 因为依赖指针这个东西 所以删除和增加都非常方便<br>2. 延迟初始化机制<br>   延迟初始化机制 主要是为了解决像数组这种 声明的时候就分配了内存空间的问题，有的时候我们只需要声明，但还不需要使用它，这个时候没有必要分配内存空间，再如文中提到的 在同一时刻声明大量的内存空间的话 那么cpu的使用和内存空间的使用将会激增<br>   所以我们需要延迟初始化机制(设计模式中的单例模式也提到了延迟初始化问题,避免声明出来没人使用的尴尬局面)<br>3. list关于延迟初始化机制的一些处理<br>   延迟初始化机制的缺点就在于在使用的时候需要判断是否已经初始化了,如果使用比较频繁的话，就会造成大量的计算浪费(cpu浪费)<br>   所以list当中关于延迟初始化机制的处理方案如下<br>   3.1 在插入新元素时 需要检查是否已经初始化好了<br>   3.2 在移动 或者将已有元素再修改位置插入时 需要判断元素身上的链表是否和要操作的链表的指针相同 如果不相同说明元素不存在该链表中 如果相同则说明这个链表肯定已经初始化好了<br>4. ring包和list包的区别<br>    首先从源码来看<br>   type Ring struct {<br>       next, prev *Ring<br>       Value<br>   }<br><br>  type Element struct {<br>       next, prev *Element<br>       list *List &#47;&#47;它所属的链表的指针<br>       Value interface{} <br>  }<br><br>   type List struct {<br>       root element<br>       len int<br>   }<br>   从源码的定义分析得出 ring 的元素就是ring结构体(只代表了一个元素)  而list的元素 是list+element(代表了一个链表)<br>  list在创建的时候不需要指定元素也没有必要,因为它会不停的增长 而ring的创建需要指定元素个数 且不再增长<br>  并且list的还存在一个没有实际意义的根元素  该根元素还可以用来连接首位两端 使其成为一个循环链表<br>关于文中 两个结论的思考<br>结论1 go语言中切片实现了数组的延迟初始化机制<br>       我的思考是 因为切片延迟初始化了， 所以他的底层数组在切片声明时也没有被初始化出来<br>结论2 ring 使用len方法是o(n) 而list使用len方法是o(1)<br>       还没看讲解我就去翻看了源码(我比较喜欢的一句话是源码之下无秘密),从上面两个结构体的声明和list的insert方法可以看出 因为list的根元素这个根元素也代表了链表(突然想明白ring和list的第二点区别) 在这个根元素上存放了一个len数据表示链表的长度 insert时这个长度会执行+1,所以执行len方法时只需要取出这个长度即可从而达到了o(1)的时间复杂度 而ring结构体中却不存在这样的len所以需要遍历完整个环,所以时间复杂度为o(n)<br>关于思考题<br>1. ring包从实现来分析得出 适合用来执行长度固定的循环事件<br>2. heap包 则适合用来做堆排序 求第k大 第k小等问题 还有就是前面某些同学提到的优先调度问题<br>关于优先调度问题我觉得思路大概如下<br>首先维护一个堆 然后针对每个要调度的事件 分配一个优先级 然后从下到上执行堆化过程 让优先级(最低或者最高的放到堆的顶部) 当处理完成之后 再把堆尾部的事件放到堆顶部 然后执行从上往下进行堆化维护好堆的顺序 再执行逻辑</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-04 16:07:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/46/50/511205c3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>louis</span>
  </div>
  <div class="_2_QraFYR_0">郝老师，这里不太理解什么叫“自己生成的Element类型值”？把自己生成的Element类型值传给链表——这个能不能再通俗点描述？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 比如你用 list.New 函数创建了一个 List 类型的双向链表，然后通过它的一些方法往里面塞了一些元素。可以往里面塞的元素的方法有 PushFront、PushBack、InsertAfter、InsertBefore 等。<br><br>但是你发现没有，这些方法接受的新元素的类型都是 interface{} 的。也就是说，这个 List 类型的链表只接受 interface{} 类型的新元素值。<br><br>而当新元素值进入链表之后，链表会把它们再包装成 list.Element 类型的值。你看，那些往里塞元素值的方法返回的都是被包装后的 *list.Element 类型的元素值。<br><br>当你像我这样浏览了 container&#47;list.List 类型的相关 API 之后，就应该可以明白我问这个问题的背景了。<br><br>这个 List 类型只会接受 interface{} 类型的新元素值，并且只会吐出 *list.Element 类型的已有元素值。显然，任何移动已有元素值或者删除已有元素值的方法，都只会接受该链表自己吐出来的“Element 值”。因此，对于我们自己生成的“Element 值”，这个链表的任何方法都是不会接受的。<br><br>当然了，如果你之前完全没用过 List 类型，可能会觉得这个问题有些突兀。但是当你看完下面的详细解释之后，我相信你就会有所了解了。<br><br>我们这个专栏的一个风格就是：“先抛出问题，然后再解释前因后果”。目的就是，逼迫大家在碰到问题之后自己先去了解背景并试着找找答案，然后再回来看我的答案。这样的话，你对这些知识点的记忆会更牢固，不容易忘。<br><br>我非常希望这个专栏能成为大家的“枕边书”，而不是听听音频就放在一边的那种。所以才有了这样的结构设计。如果你们能在今后碰到问题时想起这个专栏，到这里翻一翻并能找到答案的线索，那我就太高兴了。<br><br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-23 17:37:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/77/a4/e57f2014.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Err</span>
  </div>
  <div class="_2_QraFYR_0">我觉得写一个实际的例子能帮助更好理解</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-23 08:41:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/9f/1d/ec173090.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>melon</span>
  </div>
  <div class="_2_QraFYR_0">list的一个典型应用场景是构造FIFO队列；ring的一个典型应用场景是构造定长环回队列，比如网页上的轮播；heap的一个典型应用场景是构造优先级队列。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-29 16:06:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/b7/b5/07fc5f58.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>fliter</span>
  </div>
  <div class="_2_QraFYR_0">为什么不把list像slice，map一样作为一种不需要import其他包就能使用的数据类型？是因为使用场景较后两者比较少吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这需要一个过程，之前list也不是标准库中的一员。况且也没必要把太多的东西多做到语言里，这样反倒不利于后面的扩展。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-29 01:06:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/ac/a1/43d83698.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>云学</span>
  </div>
  <div class="_2_QraFYR_0">在内存上，和ring的区别是list多了一个特殊表头节点，充当哨兵</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-31 09:20:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/39/04/a8817ecf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>会网络的老鼠</span>
  </div>
  <div class="_2_QraFYR_0">现在大家写golang程序，一般用什么IDE？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-30 07:21:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/2d/18/918eaecf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>后端进阶</span>
  </div>
  <div class="_2_QraFYR_0">前面的网友，goland了解一下，超赞的ide</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-06 09:11:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKdiaUiaCYQe9tibemaNU5ya7RrU3MYcSGEIG7zF27u0ZDnZs5lYxPb7KPrAsj3bibM79QIOnPXAatfIw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_a8be59</span>
  </div>
  <div class="_2_QraFYR_0">您好 能否出一个list 链表生成的一个图解，现在我看源码用图去模拟生成 一直搞混掉，特别是在初始化的时候prev和next都指向自身的root 这个很迷糊<br>比如:<br>	c.PushBack(&quot;123&quot;)<br>	c.PushFront(&quot;456&quot;)<br>	c.PushFront(&quot;789&quot;)<br>根据个人图解应该是789-》456-》nil，为什么能遍历出来很不清楚。能否有一个从初始化到最后生成的样例看一下 万分感谢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 按照你这三行代码，应该是 789 -&gt; 456 -&gt; 123 啊，你那个“789-》456-》ni”是怎么出来的？<br><br>这个链表里所谓的 root 就是用来表示链表两端的尽头的。所以，这个链表的末端实际上并不是 nil ，而是 root。只是在 Element 的 Next 方法中，如果发现它的 next 字段的值等于 root，就会返回 nil 而已。<br><br>明白了吗？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-26 10:57:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/5f/09/80484e2e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>李斌</span>
  </div>
  <div class="_2_QraFYR_0">用 vscode 就蛮好的，我之前是八年 vim 党，写 golang 时硬生生地被掰成 vscode</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，也算是与时俱进吧，未来有脑机接口了，也就用不着这些了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-30 22:42:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/e9/71/40b04914.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Zer0</span>
  </div>
  <div class="_2_QraFYR_0">不能把自己生成的Element传给List主要是因为Element的list成员是不开放的，我们不能操作，而在List上操作Element的时候是会判断Element的list成员是否是自己。是这样吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: ✅</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-06 09:26:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/9d/a4/5d2b5aed.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>雷朝建</span>
  </div>
  <div class="_2_QraFYR_0">老师， 我看了一下list.go的源码，发现一个疑问是：延迟初始化的含义就是调用lazyInit，它的一个判断条件是：l.root.next==nil； 但是我们在使用list时候，不是先调用New函数吗？那么不应该会出现l.root.next为nil的情况的。<br>什么时候回出现l.root.next==nil, 从而导致源码中每次的PushFront等操作调用lazyInit呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: List 是一个 struct 啊，有了 lazyInit 你就可以直接 var myList list.List 了啊。这不是就做到开箱即用了吗？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-02 07:18:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/5a/37/8775d714.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jackstraw</span>
  </div>
  <div class="_2_QraFYR_0">我尝试打印了 “var l = list.New()” 与 “var l list.List”两种方式的l类型，发现是不一样的，但是下面的操作却都是可以的<br>func main() {<br>    &#47;&#47;l := list.New()<br>    var l list.List<br>    e4 := l.PushBack(4)<br>    e1 := l.PushFront(1)<br>    l.InsertBefore(3, e4)<br>    l.InsertAfter(2, e1)<br>    &#47;&#47;travel(l)<br>    travel(&amp;l)<br>}</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 当然不一样，list.New 返回的是指针值。另外你可以再看看讲结构体和方法那篇文章。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-09 12:05:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/16/2a/68913d36.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>杨震</span>
  </div>
  <div class="_2_QraFYR_0">以后再有课的话  希望老师多加点图   虽然费点事  但应该更多为学员着想吧。文字阐述一点也不直观。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: OK。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-25 11:21:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/3f/c9/1ccefb9a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Sky</span>
  </div>
  <div class="_2_QraFYR_0">这一讲没有实例代码</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 建议大家在阅读这一篇文章时对照着 container 包的文档看。对这几个类型的 API 有一定了解之后，使用就是水到渠成的事情了。<br><br>自己先试一试，如果有具体的问题，可以来这里问。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-12 17:45:26</div>
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
  <div class="_2_QraFYR_0">对于这一讲的内容，确实要用IDE（例如Goland）打开源码文件一边阅读一边对着课中的文字，一边推敲、实验。<br><br>几个源码文件都是一两百来行，读起来不吃力。（前提是已经有了Go的基础）</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-27 11:31:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/37/3f/a9127a73.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>KK</span>
  </div>
  <div class="_2_QraFYR_0">关于list的结构，画个图会更加直观</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-19 16:30:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c9/a8/98507423.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lixiaofeng</span>
  </div>
  <div class="_2_QraFYR_0">func flist(){<br>    link := list.New()<br>    for i:=1; i&lt; 11; i++  {<br>        link.PushBack(i)<br>    }<br>    for p:= link.Front(); p!=link.Back(); p=p.Next(){<br>        fmt.Println(&quot;Number&quot;, p.Value)<br>    }<br>}<br>#链表的常用方法<br>func (e *Element) Next() *Element<br>func (e *Element) Prev() *Element<br>func (l *List) Init() *List<br> New() *List { return new(List).Init() }<br> func (l *List) Len() int { return l.len }<br> func (l *List) Front() *Element<br> func (l *List) Back() *Element<br> func (l *List) Remove(e *Element) interface{}<br> func (l *List) PushFront(v interface{}) *Element<br> func (l *List) PushBack(v interface{}) *Element<br> func (l *List) InsertBefore(v interface{}, mark *Element) *Element <br> func (l *List) InsertAfter(v interface{}, mark *Element) *Element<br> func (l *List) MoveToFront(e *Element)<br> func (l *List) MoveToBack(e *Element) <br> func (l *List) MoveBefore(e, mark *Element)<br> func (l *List) MoveAfter(e, mark *Element)<br> func (l *List) PushBackList(other *List)<br> func (l *List) PushFrontList(other *List)</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-20 23:06:38</div>
  </div>
</div>
</div>
</li>
</ul>