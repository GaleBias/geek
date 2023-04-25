<audio title="02｜拒绝“Hello and Bye”：Go语言的设计哲学是怎么一回事？" src="https://static001.geekbang.org/resource/audio/9a/f5/9abcbb68fc91093ab3cfd18293aa49f5.mp3" controls="controls"></audio> 
<p>你好，我是Tony Bai。</p><p>上一讲，我们探讨了<strong>“Go从哪里来，并可能要往哪里去”</strong>的问题。根据“绝大多数主流编程语言将在其15至20年间大步前进”这个依据，我们给出了一个结论：<strong>Go语言即将进入自己的黄金5~10年</strong>。</p><p>那么此时此刻，想必你已经跃跃欲试，想要尽快开启Go编程之旅。但在正式学习Go语法之前，我还是要再来给你<strong>泼泼冷水</strong>，因为这将决定你后续的学习结果，是“从入门到继续”还是“从入门到放弃”。</p><p>很多编程语言的初学者在学习初期，可能都会遇到这样的问题：最初兴致勃勃地开始学习一门编程语言，学着学着就发现了很多“别扭”的地方，比如想要的语言特性缺失、语法风格冷僻与主流语言差异较大、语言的不同版本间无法兼容、语言的语法特性过多导致学习曲线陡峭、语言的工具链支持较差，等等。</p><p>其实以上的这些问题，本质上都与语言设计者的设计哲学有关。所谓编程语言的设计哲学，就是指决定这门语言演化进程的高级原则和依据。</p><p><strong>设计哲学之于编程语言，就好比一个人的价值观之于这个人的行为。</strong></p><p>因为如果你不认同一个人的价值观，那你其实很难与之持续交往下去，即所谓道不同不相为谋。类似的，如果你不认同一门编程语言的设计哲学，那么大概率你在后续的语言学习中，就会遇到上面提到的这些问题，而且可能会让你失去继续学习的精神动力。</p><!-- [[[read_end]]] --><p>因此，在真正开始学习Go语法和编码之前，我们还需要先来了解一下Go语言的设计哲学，等学完这一讲之后，你就能更深刻地认识到自己学习Go语言的原因了。</p><p>我将Go语言的设计哲学总结为五点：简单、显式、组合、并发和面向工程。下面，我们就先从Go语言的第一设计哲学“<strong>简单</strong>”开始了解吧。</p><h3>简单</h3><p>知名Go开发者戴维·切尼（Dave Cheney）曾说过：“大多数编程语言创建伊始都致力于成为一门简单的语言，但最终都只是满足于做一个强大的编程语言”。</p><p><strong>而Go语言是一个例外。Go语言的设计者们在语言设计之初，就拒绝了走语言特性融合的道路，选择了“做减法”并致力于打造一门简单的编程语言。</strong></p><p>选择了“简单”，就意味着Go不会像C++、Java那样将其他编程语言的新特性兼蓄并收，所以你在Go语言中看不到传统的面向对象的类、构造函数与继承，看不到结构化的异常处理，也看不到本属于函数编程范式的语法元素。</p><p>其实，Go语言也没它看起来那么简单，自身实现起来并不容易，但这些复杂性被Go语言的设计者们“隐藏”了，所以Go语法层面上呈现了这样的状态：</p><ul>
<li>仅有25个关键字，主流编程语言最少；</li>
<li>内置垃圾收集，降低开发人员内存管理的心智负担；</li>
<li>首字母大小写决定可见性，无需通过额外关键字修饰；</li>
<li>变量初始为类型零值，避免以随机值作为初值的问题；</li>
<li>内置数组边界检查，极大减少越界访问带来的安全隐患；</li>
<li>内置并发支持，简化并发程序设计；</li>
<li>内置接口类型，为组合的设计哲学奠定基础；</li>
<li>原生提供完善的工具链，开箱即用；</li>
<li>… …</li>
</ul><p>看，我说的没错吧，确实挺简单的。当然了，任何的设计都存在着权衡与折中。我们看到Go设计者选择的“简单”，其实是站在巨人肩膀上，去除或优化了以往语言中，已经被开发者证明为体验不好或难以驾驭的语法元素和语言机制，并提出了自己的一些创新性的设计。比如，首字母大小写决定可见性、变量初始为类型零值、内置以go关键字实现的并发支持等。</p><p>Go这种有些“逆潮流”的“简单哲学”并不是一开始就能得到程序员的理解的，但在真正使用Go之后，我们才能真正体会到这种简单所带来的收益：简单意味着可以使用更少的代码实现相同的功能；简单意味着代码具有更好的可读性，而可读性好的代码通常意味着更好的可维护性以及可靠性。</p><p>总之，在软件工程化的今天，这些都意味着对生产效率提升的极大促进，我们可以认为<strong>简单的设计哲学是Go生产力的源泉</strong>。</p><h3>显式</h3><p>好，接下来我们继续来了解学习下Go语言的第二大设计哲学：<strong>显式</strong>。</p><p>首先，我想先带你来看一段C程序，我们一起来看看“隐式”代码的行为特征。</p><p>在C语言中，下面这段代码可以正常编译并输出正确结果：</p><pre><code class="language-plain">#include &lt;stdio.h&gt;

int main() {
		short int a = 5;

		int b = 8;
		long c = 0;
		
		c = a + b;
		printf("%ld\n", c);
}
</code></pre><p>我们看到在上面这段代码中，变量a、b和c的类型均不相同，C语言编译器在编译<code>c = a + b</code>这一行时，会自动将短整型变量a和整型变量b，先转换为long类型然后相加，并将所得结果存储在long类型变量c中。那如果换成Go来实现这个计算会怎么样呢？我们先把上面的C程序转化成等价的Go代码：</p><pre><code class="language-plain">package main

import "fmt"

func main() {
    var a int16 = 5
    var b int = 8
    var c int64

    c = a + b
    fmt.Printf("%d\n", c)
}
</code></pre><p>如果我们编译这段程序，将得到类似这样的编译器错误：“invalid operation: a + b (mismatched types int16 and int)”。我们能看到Go与C语言的隐式自动类型转换不同，Go不允许不同类型的整型变量进行混合计算，它同样也不会对其进行隐式的自动转换。</p><p>因此，如果要使这段代码通过编译，我们就需要对变量a和b进行<strong>显式转型</strong>，就像下面代码段中这样：</p><pre><code class="language-plain">c = int64(a) + int64(b)
fmt.Printf("%d\n", c)
</code></pre><p>而这其实就是Go语言<strong>显式设计哲学</strong>的一个体现。</p><p>在Go语言中，不同类型变量是不能在一起进行混合计算的，这是因为<strong>Go希望开发人员明确知道自己在做什么</strong>，这与C语言的“信任程序员”原则完全不同，因此你需要以显式的方式通过转型统一参与计算各个变量的类型。</p><p>除此之外，Go设计者所崇尚的显式哲学还直接决定了Go语言错误处理的形态：Go语言采用了<strong>显式的基于值比较的错误处理方案</strong>，函数/方法中的错误都会通过return语句显式地返回，并且通常调用者不能忽略对返回的错误的处理。</p><p>这种有悖于“主流语言潮流”的错误处理机制还一度让开发者诟病，社区也提出了多个新错误处理方案，但或多或少都包含隐式的成分，都被Go开发团队一一否决了，这也与显式的设计哲学不无关系。</p><h3>组合</h3><p>接着，我们来看第三个设计哲学：<strong>组合</strong>。</p><p>这个设计哲学和我们各个程序之间的耦合有关，Go语言不像C++、Java等主流面向对象语言，我们在Go中是找不到经典的面向对象语法元素、类型体系和继承机制的，Go推崇的是组合的设计哲学。</p><p>在诠释组合之前，我们需要先来了解一下Go在语法元素设计时，是如何为“组合”哲学的应用奠定基础的。</p><p>在Go语言设计层面，Go设计者为开发者们提供了正交的语法元素，以供后续组合使用，包括：</p><ul>
<li>Go语言无类型层次体系，各类型之间是相互独立的，没有子类型的概念；</li>
<li>每个类型都可以有自己的方法集合，类型定义与方法实现是正交独立的；</li>
<li>实现某个接口时，无需像Java那样采用特定关键字修饰；</li>
<li>包之间是相对独立的，没有子包的概念。</li>
</ul><p>我们可以看到，无论是包、接口还是一个个具体的类型定义，Go语言其实是为我们呈现了这样的一幅图景：一座座没有关联的“孤岛”，但每个岛内又都很精彩。那么现在摆在面前的工作，就是在这些孤岛之间以最适当的方式建立关联，并形成一个整体。而<strong>Go选择采用的组合方式，也是最主要的方式</strong>。</p><p>Go语言为支撑组合的设计提供了<strong>类型嵌入</strong>（Type Embedding）。通过类型嵌入，我们可以将已经实现的功能嵌入到新类型中，以快速满足新类型的功能需求，这种方式有些类似经典面向对象语言中的“继承”机制，但在原理上却与面向对象中的继承完全不同，这是一种Go设计者们精心设计的“语法糖”。</p><p>被嵌入的类型和新类型两者之间没有任何关系，甚至相互完全不知道对方的存在，更没有经典面向对象语言中的那种父类、子类的关系，以及向上、向下转型（Type Casting）。通过新类型实例调用方法时，方法的匹配主要取决于方法名字，而不是类型。这种组合方式，我称之为<strong>垂直组合</strong>，即通过类型嵌入，快速让一个新类型“复用”其他类型已经实现的能力，实现功能的垂直扩展。</p><p>你可以看看下面这个Go标准库中的一段使用类型嵌入的组合方式的代码段：</p><pre><code class="language-plain">// $GOROOT/src/sync/pool.go
type poolLocal struct {
    private interface{}   
    shared  []interface{}
    Mutex               
    pad     [128]byte  
}
</code></pre><p>在代码段中，我们在poolLocal这个结构体类型中嵌入了类型Mutex，这就使得poolLocal这个类型具有了互斥同步的能力，我们可以通过poolLocal类型的变量，直接调用Mutex类型的方法Lock或Unlock。<br>
另外，我们在标准库中还会经常看到类似如下定义接口类型的代码段：</p><pre><code class="language-plain">// $GOROOT/src/io/io.go
type ReadWriter interface {
    Reader
    Writer
}
</code></pre><p>这里，标准库通过嵌入接口类型的方式来实现接口行为的聚合，组成大接口，这种方式在标准库中尤为常用，并且已经成为了Go语言的一种惯用法。<br>
垂直组合本质上是一种“能力继承”，采用嵌入方式定义的新类型继承了嵌入类型的能力。Go还有一种常见的组合方式，叫<strong>水平组合</strong>。和垂直组合的能力继承不同，水平组合是一种能力委托（Delegate），我们通常使用接口类型来实现水平组合。</p><p>Go语言中的接口是一个创新设计，它只是方法集合，并且它与实现者之间的关系无需通过显式关键字修饰，它让程序内部各部分之间的耦合降至最低，同时它也是连接程序各个部分之间“纽带”。</p><p>水平组合的模式有很多，比如一种常见方法就是，通过接受接口类型参数的普通函数进行组合，如以下代码段所示：</p><pre><code class="language-plain">// $GOROOT/src/io/ioutil/ioutil.go
func ReadAll(r io.Reader)([]byte, error)

// $GOROOT/src/io/io.go
func Copy(dst Writer, src Reader)(written int64, err error)
</code></pre><p>也就是说，函数ReadAll通过io.Reader这个接口，将io.Reader的实现与ReadAll所在的包低耦合地水平组合在一起了，从而达到从任意实现io.Reader的数据源读取所有数据的目的。类似的水平组合“模式”还有点缀器、中间件等，这里我就不展开了，在后面讲到接口类型时再详细叙述。</p><p>此外，我们还可以将Go语言内置的并发能力进行灵活组合以实现，比如，通过goroutine+channel的组合，可以实现类似Unix Pipe的能力。</p><p>总之，组合原则的应用实质上是塑造了Go程序的骨架结构。类型嵌入为类型提供了垂直扩展能力，而接口是水平组合的关键，它好比程序肌体上的“关节”，给予连接“关节”的两个部分各自“自由活动”的能力，而整体上又实现了某种功能。并且，组合也让遵循“简单”原则的Go语言，在表现力上丝毫不逊色于其他复杂的主流编程语言。</p><h3>并发</h3><p>好，前面我们已经看过3个设计哲学了，紧接着我带你看的是第4个：<strong>并发</strong>。</p><p>“并发”这个设计哲学的出现有它的背景，你也知道CPU都是靠提高主频来改进性能的，但是现在这个做法已经遇到了瓶颈。主频提高导致CPU的功耗和发热量剧增，反过来制约了CPU性能的进一步提高。2007年开始，处理器厂商的竞争焦点从主频转向了多核。</p><p>在这种大背景下，Go的设计者在决定去创建一门新语言的时候，果断将面向多核、<strong>原生支持并发</strong>作为了新语言的设计原则之一。并且，Go放弃了传统的基于操作系统线程的并发模型，而采用了<strong>用户层轻量级线程</strong>，Go将之称为<strong>goroutine</strong>。</p><p>goroutine占用的资源非常小，Go运行时默认为每个goroutine分配的栈空间仅2KB。goroutine调度的切换也不用陷入（trap）操作系统内核层完成，代价很低。因此，一个Go程序中可以创建成千上万个并发的goroutine。而且，所有的Go代码都在goroutine中执行，哪怕是go运行时的代码也不例外。</p><p>在提供了开销较低的goroutine的同时，Go还在语言层面内置了辅助并发设计的原语：channel和select。开发者可以通过语言内置的channel传递消息或实现同步，并通过select实现多路channel的并发控制。相较于传统复杂的线程并发模型，Go对并发的原生支持将大大降低开发人员在开发并发程序时的心智负担。</p><p>此外，并发的设计哲学不仅仅让Go在语法层面提供了并发原语支持，其对Go应用程序设计的影响更为重要。并发是一种程序结构设计的方法，它使得并行成为可能。</p><p>采用并发方案设计的程序在单核处理器上也是可以正常运行的，也许在单核上的处理性能可能不如非并发方案。但随着处理器核数的增多，并发方案可以自然地提高处理性能。</p><p>而且，并发与组合的哲学是一脉相承的，并发是一个更大的组合的概念，它在程序设计的全局层面对程序进行拆解组合，再映射到程序执行层面上：goroutines各自执行特定的工作，通过channel+select将goroutines组合连接起来。并发的存在鼓励程序员在程序设计时进行独立计算的分解，而对并发的原生支持让Go语言也更适应现代计算环境。</p><h3>面向工程</h3><p>最后，我们来看一下Go的最后一条设计哲学：面向工程。</p><p>Go语言设计的初衷，就是<strong>面向解决真实世界中Google内部大规模软件开发存在的各种问题，为这些问题提供答案</strong>，这些问题包括：程序构建慢、依赖管理失控、代码难于理解、跨语言构建难等。</p><p>很多编程语言设计者和他们的粉丝们认为这些问题并不是一门编程语言应该去解决的，但Go语言的设计者并不这么看，他们在Go语言最初设计阶段就<strong>将解决工程问题作为Go的设计原则之一</strong>去考虑Go语法、工具链与标准库的设计，这也是Go与其他偏学院派、偏研究型的编程语言在设计思路上的一个重大差异。</p><p>语法是编程语言的用户接口，它直接影响开发人员对于这门语言的使用体验。在面向工程设计哲学的驱使下，Go在语法设计细节上做了精心的打磨。比如：</p><ul>
<li>重新设计编译单元和目标文件格式，实现Go源码快速构建，让大工程的构建时间缩短到类似动态语言的交互式解释的编译速度；</li>
<li>如果源文件导入它不使用的包，则程序将无法编译。这可以充分保证任何Go程序的依赖树是精确的。这也可以保证在构建程序时不会编译额外的代码，从而最大限度地缩短编译时间；</li>
<li>去除包的循环依赖，循环依赖会在大规模的代码中引发问题，因为它们要求编译器同时处理更大的源文件集，这会减慢增量构建；</li>
<li>包路径是唯一的，而包名不必唯一的。导入路径必须唯一标识要导入的包，而名称只是包的使用者如何引用其内容的约定。“包名称不必是唯一的”这个约定，大大降低了开发人员给包起唯一名字的心智负担；</li>
<li>故意不支持默认函数参数。因为在规模工程中，很多开发者利用默认函数参数机制，向函数添加过多的参数以弥补函数API的设计缺陷，这会导致函数拥有太多的参数，降低清晰度和可读性；</li>
<li>增加类型别名（type alias），支持大规模代码库的重构。</li>
</ul><p>在标准库方面，Go被称为“自带电池”的编程语言。如果说一门编程语言是“自带电池”，则说明这门语言标准库功能丰富，多数功能不需要依赖外部的第三方包或库，Go语言恰恰就是这类编程语言。</p><p>由于诞生年代较晚，而且目标比较明确，Go在标准库中提供了各类高质量且性能优良的功能包，其中的<code>net/http</code>、<code>crypto</code>、<code>encoding</code>等包充分迎合了云原生时代的关于API/RPC Web服务的构建需求，Go开发者可以直接基于标准库提供的这些包实现一个满足生产要求的API服务，从而减少对外部第三方包或库的依赖，降低工程代码依赖管理的复杂性，也降低了开发人员学习第三方库的心理负担。</p><p>而且，开发人员在工程过程中肯定是需要使用工具的，Go语言就提供了足以让所有其它主流语言开发人员羡慕的工具链，工具链涵盖了编译构建、代码格式化、包依赖管理、静态代码检查、测试、文档生成与查看、性能剖析、语言服务器、运行时程序跟踪等方方面面。</p><p>这里值得重点介绍的是<strong>gofmt</strong>，它统一了Go语言的代码风格，在其他语言开发者还在为代码风格争论不休的时候，Go开发者可以更加专注于领域业务中。同时，相同的代码风格让以往困扰开发者的代码阅读、理解和评审工作变得容易了很多，至少Go开发者再也不会有那种因代码风格的不同而产生的陌生感。Go的这种统一代码风格思路也在开始影响着后续新编程语言的设计，并且一些现有的主流编程语言也在借鉴Go的一些设计。</p><p>在提供丰富的工具链的同时，Go在标准库中提供了官方的词法分析器、语法解析器和类型检查器相关包，开发者可以基于这些包快速构建并扩展Go工具链。</p><h3>小结</h3><p>好了，今天的课讲到这里就结束了，现在我们一起来回顾一下吧。</p><p>在这一讲中，我和你一起了解了Go语言的设计哲学：<strong>简单</strong>、<strong>显式</strong>、<strong>组合</strong>、<strong>并发和面向工程</strong>。</p><ul>
<li>简单是指Go语言特性始终保持在少且足够的水平，不走语言特性融合的道路，但又不乏生产力。简单是Go生产力的源泉，也是Go对开发者的最大吸引力；</li>
<li>显式是指任何代码行为都需开发者明确知晓，不存在因“暗箱操作”而导致可维护性降低和不安全的结果；</li>
<li>组合是构建Go程序骨架的主要方式，它可以大幅降低程序元素间的耦合，提高程序的可扩展性和灵活性；</li>
<li>并发是Go敏锐地把握了CPU向多核方向发展这一趋势的结果，可以让开发人员在多核时代更容易写出充分利用系统资源、支持性能随CPU核数增加而自然提升的应用程序；</li>
<li>面向工程是Go语言在语言设计上的一个重大创新，它将语言要解决的问题域扩展到那些原本并不是由编程语言去解决的领域，从而覆盖了更多开发者在开发过程遇到的“痛点”，为开发者提供了更好的使用体验。</li>
</ul><p>这些设计哲学直接影响了Go语言自身的设计。理解这些设计哲学，也能帮助我们理解Go语言语法、标准库以及工具链的演化决策过程。</p><p>好了，学完这节课之后，你认同Go的设计哲学吗？认同的话就继续跟着我学下去吧。</p><h3>思考题</h3><p>今天，我还想问下你，你还能举出哪些符合Go语言设计哲学的例子吗？欢迎在留言区多多和我分享讨论。</p><p>感谢你和我一起学习，也欢迎你把这节课分享给更多对Go语言的设计哲学感兴趣的朋友。我是Tony Bai，我们下节课见。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/27/52/40/db9b0eb2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>自由</span>
  </div>
  <div class="_2_QraFYR_0">Tony Bai 老师，你好，例举两个我认为复合 Go 语言设计哲学的例子，我的技术能力捉襟见肘，说的不对地方还希望老师斧正。<br><br>一、异常处理<br><br>Go 语言核心开发者 Dave 曾说过 “You only need to check the error value if you care about the result”，在我们不处理错误的时候，我们不应该对它的返回值抱有任何幻想。<br><br>Go 的异常处理逻辑，没有引入 exception，而是使用了多参数返回，在返回中带上错误，由调用者来判定这个错误。<br><br>- 简单<br>- 没有隐藏的控制流<br>- 完全交给你控制 error<br>- 考虑失败，而不是成功<br><br>二、类型别名<br><br>在特定情况下，帮助代码逐步修复。<br><br>类型别名的存在，是 渐进式代码修复(Gradual code repair) 的关键，什么是渐进式代码修复？举一个🌰 重构。重构代码，我们当然希望重构后的好处，能够适用于所有代码，但是，重构的好处与代价是成正比的，往往一次重构会伴随着大量的修改，随着代码量越来越大，一次完成所有修改变得不可行。修复需要逐步完成。在代码量少时，我们可以一次性完成所有的修复，这样的修复被称为原子代码修复(atomic code repair)，它的概念很简单，就是在一次提交中，更新所有的因为重构带来的问题修复，但是概念的简单会被实际的复杂性抵消，一次提交可能非常大，大的提交很难去一次性修复，出现问题也很难去溯源，最重要的是，可能会与其他同学的工作产生冲突，例如某个同学，在工作时，使用了旧的 API，合并代码时，并不会产生冲突，而我的提交错过了它的引用。<br><br>因此，我们需要一个过渡期，这个过渡期就是为了逐步替换，也就是渐进式代码修复，将旧的引用，逐步替换，同时将旧的换为新的，这就是渐进式代码修复，它的缺点是比原子代码修复的工作量更大，但是它更容易提交、审查，并且保证了，没有人引用后再删除旧的类型别名。<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍。说的很好。很认同你提到的基于“类型别名”的渐进式代码修复(Gradual code repair) 思路。这也是类型别名最初被引入go的初衷（https:&#47;&#47;github.com&#47;golang&#47;proposal&#47;blob&#47;master&#47;design&#47;18130-type-alias.md）。我觉得它也是go面向工程设计哲学的体现。另外type alias在基于现有实现进行扩展并做出新的封装方面也有“奇效”。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-15 01:47:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/79/69/5960a2af.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>王智</span>
  </div>
  <div class="_2_QraFYR_0">表示身为一名Java工程师，在看到组合的时候有一点疑惑，我的想法是这里的组合就是将另一个类里面的东西平移过来，类似于java中的继承，我想问的是如果存在两个类包含相同名字的方法或者属性，这个go怎么处理？还是直接就不允许呢？go语言从来没接触过，不懂就问</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 好问题！<br><br>go的组合有多种形式。按你提到的“继承”型组合中，如果组合的两个类型中有相同名字的字段，那怎么解决呢？看下面中的例子：<br><br>package main<br><br>type T1 struct {<br>	a int<br>}<br>type T2 struct {<br>	a int<br>}<br>type T struct {<br>	T1<br>	T2<br>}<br><br>func main() {<br>	var t T<br>	&#47;&#47;	t.a = 5 &#47;&#47; 编译报错：ambiguous selector t.a<br>	t.T1.a = 5<br>	t.T2.a = 6<br>	println(t.T1.a) &#47;&#47; 5<br>	println(t.T2.a) &#47;&#47; 6<br>}<br><br>如果T组合的两个类型T1和T2都包含字段a，那么我们不能直接使用t.a，而是通过t.T1.a和t.T2.a分别指代各自类型中的字段a。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-26 17:11:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/26/27/eba94899.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>罗杰</span>
  </div>
  <div class="_2_QraFYR_0">21 世纪的 C 语言，的确实至名归。依然有几个小问题：1. Go 有 GC，我们使用 Go 来开发后端的所有服务，有个 PVP 的服务，需要逐帧计算客户端上报的结果是否正确，此时对于内存的分配就要特别小心，开发起来很不顺畅。是否这种服务的性质不太合适使用 Go 来开发；2. 有人吐槽 Go 核心人员不想做的东西，就是 Less is more，自己想做就是各种哲学，这个问题，老师怎么看？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 罗杰出品的问题，都是精品问题^_^。<br><br>先来看第一个问题，Go最初设计目标是通用的系统编程语言，但Go选择支持了GC。Go的GC虽然在go团队的努力下，开销越来越小，但开销小，低延迟不代表没有，这就决定了Go在一些对性能极其敏感的领域可能并不是最好的选择。你的问题中也提到了pvp服务，想必你们也是采用了面向服务的架构，这种架构本身就是可以天然适合技术异构的。如果觉得不妥，也别强求，果断换非GC语言，比如c、c++或是rust。如果说非要坚持用go来完成，那么说明你是go的骨灰粉，在解决问题的过程中，你也会完成一次go技能的升华。<br><br><br>第二个问题，Go语言的简单或者说功能特性少，的确来自与less is more的理念。保持一门小语言，让语言更容易学习与理解。同时每个特性都是经过精心打磨与实现，不能再少了。上周我看了rob pike最新一期的talk，他还在说 “Go语言中变量声明的方式有些多了”，这也是我在实际编码过程中的体会。如果重新来过，我想rob pike会更彻底的执行less is more，将变量声明方式再减少一种。所以说，特性少不是不想做，而是经过深思熟虑，那个特性的确没必要加入到语言中。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-15 12:32:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/20/1e/18/9d1f1439.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>liaomars</span>
  </div>
  <div class="_2_QraFYR_0">go的异常处理，使用起来简单，但是不方便，请问老师这是在践行go的简单设计哲学吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 从go设计者的初衷来看(https:&#47;&#47;golang.google.cn&#47;doc&#47;faq#exceptions)，go没有采用像java那样的结构化异常处理的确是出于对“简单”原则的考虑。<br><br>在java中错误处理与真正的“异常”是混杂在Try-catch机制中的，并没有明显的界限，无论是错误还是异常，一旦throw，方法的调用者就得负责处理它。<br><br>但在go中，错误处理与真正的异常处理是严格分开的，也就是说不要将panic掺和到错误处理中。<br><br>错误处理是常态，go中只有错误是返回给上层的。一旦出现panic，这意味着整个程序处于即将崩溃的状态，返回给上层几乎也是“无济于事”，所以在go中，一个常见的api设计思路是 不要向外部抛出panic（don&#39;t panic!）。如果api中存在panic的可能性，那么api自己要负责处理panic，并通过error将状态返回给上层。如果api无法处理panic，那程序就很大可能是要崩溃了，这种panic多是因为程序bug导致的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-15 11:48:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2a/e1/09/efa69f7a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>学昊</span>
  </div>
  <div class="_2_QraFYR_0">本人老java码农了。进阶的代码设计是设计模式，让代码能更优雅的实现。终极的代码设计是哲学，是代码中表达出的价值观。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-15 14:20:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_399042</span>
  </div>
  <div class="_2_QraFYR_0">自动加入分号是不是也是简单的设计哲学呢，能让编译器做的事不需要交给开发者。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 手动点赞</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-15 14:59:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/0a/a4/828a431f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张申傲</span>
  </div>
  <div class="_2_QraFYR_0">尤其认同 Go 语言的“面向工程”这一设计哲学。作为 Java 的资深用户，每天都深受编译速度慢、依赖树失控、代码风格不统一等问题的困扰。Go 语言的设计哲学恰恰迎合了现代大规模业务系统的开发和维护。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-01 18:04:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/38/65/edf48816.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>悟二空</span>
  </div>
  <div class="_2_QraFYR_0">只有 TonyBai 老师的这门专栏课，我会把每条留言评论都认真看完，因为老师都有很认真的在回复，一点儿也不含糊。能学到非常多的知识，非常感谢老师，我一定能学好Go语言并进入自己想去的公司的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 💪</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-27 00:11:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/16/5c/c0322969.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Howe</span>
  </div>
  <div class="_2_QraFYR_0">老师，请教两个问题：<br>1、文中提到的正交独立是什么意思？不是很理解。<br>2、Go不支持面向对象，那意味着复用性不好，这种后面老师会讲工程实践吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. 正交(orthogonality)是从几何学中引入的术语，如果两条线以直角相交，如图形上的轴线，就是正交的。如果说两个事物是正交的，那么我们说<br>这两个事物是独立且解耦的，一个事物的变化不影响另外一个事物。<br><br>我们经常用“正交”来评价一个系统的设计，比如在一个设计良好的系统中，数据库代码将与用户界面正交：你可以在不影响数据库的情况下，独&gt;立进行界面的演进。<br><br>编程语言的语法元素间也存在着正交的情况，比如文中提到的类型定义与方法是正交的。这意味着一个类型可以有方法，也可以没有方法。而方&gt;法本质上接收类型作为其第一个参数的函数而已(具体参考第24讲)。<br><br>在Go语言中，正交的语法还有一些，比如接口就与Go语言其他部分是正交的。<br><br>但正交的两个语法特性组合起来可以实现其它特性，这也是我们在一个系统中经常做的事情。<br><br>2. 难道不用面向对象就不能复用了么？:) Go有自己的组合的设计哲学，组合也可以实现复用。课程后面会有讲解。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-11 22:04:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/cd/ed/825d84ee.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>费城的二鹏</span>
  </div>
  <div class="_2_QraFYR_0">老师讲的很透彻，如数家珍。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-15 08:48:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/ea/27/a3737d61.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Shanks-王冲</span>
  </div>
  <div class="_2_QraFYR_0">放弃之前，看这篇，找找最初的心动，哈哈：）</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-19 20:52:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/f3/95/77628ed0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>van</span>
  </div>
  <div class="_2_QraFYR_0">您好，不好意思，之前提的问题可能不对，其实我真正想问的是其他主流比如java有没有利用多核并行？比如java构建了很多线程，这些线程的调度在多核cpu上已经天然是多核并行的，这个并不是java的语言特性，而是操作系统的特性。<br>因为go用的是用户态协程，所以go做了一个大换血，自己手动支持多核并行。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-09 22:11:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/7b/bd/ccb37425.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>进化菌</span>
  </div>
  <div class="_2_QraFYR_0">Go 语言的设计哲学总结为五点：简单、显式、组合、并发和面向工程。可能对go还不熟悉，很多地方还是没有特别的感觉，学完之后回头看，应该会是不一样的感受吧~</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-21 21:58:27</div>
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
  <div class="_2_QraFYR_0">感谢 Tony Bai 这一篇的分享，很精彩。有以下几点困惑，麻烦有时间回答一下：<br><br>1. go1.16.4 版本中的 poolLocal 结构体的实现和本文中的不太一样呢？<br><br>2. 水平组合“模式”还有点缀器、中间件等方式后面的文章中会有例子吗？<br><br>3. 很多课程中，都有“并发原语”一词，百度查了一下，有一些理解。老师这里会有比较通俗易懂的理解吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>你真非常细致和喜欢思考，提出的问题都很棒！👍<br><br>第一个问题，专栏中的poolLocal实现的确是早期的版本，不过也无伤大雅，文中的目的就是为了说明：类型嵌入的组合方式。感谢<br>你指出。不过这块我就不改了。<br><br>第二个问题，在讲解接口类型时，应该会有具体例子。<br><br>第三个问题，在后面讲解并发的章节中，我希望我的讲解能让你觉得更通俗易懂^_^。<br><br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-15 22:38:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/a2/5e/3871ff79.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>迷途书童</span>
  </div>
  <div class="_2_QraFYR_0">Go语言的设计哲学有什么权威出处吗？还是老师自己总结的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 自己总结的。但也依据了go核心团队的一些talk中的内容以及个人对go的理解。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-24 16:17:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/23/e6/12b3d2bf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Holy</span>
  </div>
  <div class="_2_QraFYR_0">go底层代码深入发现了很多巧妙的地方， <br>defer+panic简单使用，里面的实现不一般<br>内存管理多级缓存快速减少锁，为GC埋下众多伏笔，<br>Mutex兼顾公平<br>GMP模式实现等等，<br>每个版本迭代，持续进化（GOPHER坐享其成）</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-18 17:11:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/fb/6a/03aabb63.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Alexhuihui</span>
  </div>
  <div class="_2_QraFYR_0">“水平组合是一种能力委托（Delegate），我们通常使用接口类型来实现水平组合。“，原文这段话没理解，老师能再解释一下水平组合吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个我在后面讲解接口时会详细展开。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-23 23:09:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/48/11/420b6a25.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Hua.R</span>
  </div>
  <div class="_2_QraFYR_0">搭车问一下，去年我曾在大佬博客咨询过慕课网的专栏出版纸质书的问题，大佬回答当年底出版。大半年过去后没有动静我又问了一次你说已经提交出版社。那大概多久可以看到实物呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 编辑给我的最新消息是在年末左右。图书的出版周期都长一些。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-16 18:33:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1c/96/dd/1620a744.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>simple_孙</span>
  </div>
  <div class="_2_QraFYR_0">通读一遍课程之后，回头看这篇文章更通透了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-20 10:04:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/a3/ea/bd83bd4f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>麦芽糖</span>
  </div>
  <div class="_2_QraFYR_0">设计哲学很重要，就比如一个人的价值观，或者背景。 <br>在不了解别人价值观的时候去评价是不合理的，具备同样价值观的人才能走的更好。<br>比如国家之前的相互不理解，是因为国家的文化不同。<br><br>同样，需要理解 Go 的设计哲学显得非常重要。<br><br>设计哲学有<br>● 简单<br>● 显示<br>● 组合<br>● 并发<br>● 面向工程<br><br>简单。<br>比如关键字就很少。入门快。<br><br>显示。<br>如不同类型不能做运算，避免意料之外的事情发生。<br><br>组合。<br>暂时还不是很理解。<br><br>并发。<br>天生就为多核 CPU 开发，同时有自己的 goroutine 用户层轻量级线程，性能更好。<br><br>面向工程。<br>格式化、调试、工具链、默认参数等。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: “具备同样价值观的人才能走的更好” 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-15 08:00:56</div>
  </div>
</div>
</div>
</li>
</ul>