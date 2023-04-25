<audio title="29｜接口：为什么nil接口不等于nil？" src="https://static001.geekbang.org/resource/audio/f3/f9/f3b3a83eb77e786e3b9d048f05e42bf9.mp3" controls="controls"></audio> 
<p>你好，我是Tony Bai。</p><p>上一讲我们学习了Go接口的基础知识与设计惯例，知道Go接口是构建Go应用骨架的重要元素。从语言设计角度来看，Go语言的接口（interface）和并发（concurrency）原语是我最喜欢的两类Go语言语法元素。Go语言核心团队的技术负责人Russ Cox也曾说过这样一句话：“<strong>如果要从Go语言中挑选出一个特性放入其他语言，我会选择接口</strong>”，这句话足以说明接口这一语法特性在这位Go语言大神心目中的地位。</p><p>为什么接口在Go中有这么高的地位呢？这是因为<strong>接口是Go这门静态语言中唯一“动静兼备”的语法特性</strong>。而且，接口“动静兼备”的特性给Go带来了强大的表达能力，但同时也给Go语言初学者带来了不少困惑。要想真正解决这些困惑，我们必须深入到Go运行时层面，看看Go语言在运行时是如何表示接口类型的。在这一讲中，我就带着你一起深入到接口类型的运行时表示层面看看。</p><p>好，在解惑之前，我们先来看看接口的静态与动态特性，看看“动静皆备”到底是什么意思。</p><h2>接口的静态特性与动态特性</h2><p>接口的<strong>静态特性</strong>体现在<strong>接口类型变量具有静态类型</strong>，比如<code>var err error</code>中变量err的静态类型为error。拥有静态类型，那就意味着编译器会在编译阶段对所有接口类型变量的赋值操作进行类型检查，编译器会检查右值的类型是否实现了该接口方法集合中的所有方法。如果不满足，就会报错：</p><!-- [[[read_end]]] --><pre><code class="language-plain">var err error = 1 // cannot use 1 (type int) as type error in assignment: int does not implement error (missing Error method)
</code></pre><p>而接口的<strong>动态特性</strong>，就体现在接口类型变量在运行时还存储了右值的真实类型信息，这个右值的真实类型被称为接口类型变量的<strong>动态类型</strong>。你看一下下面示例代码：</p><pre><code class="language-plain">var err error
err = errors.New("error1")
fmt.Printf("%T\n", err)  // *errors.errorString
</code></pre><p>我们可以看到，这个示例通过errros.New构造了一个错误值，赋值给了error接口类型变量err，并通过fmt.Printf函数输出接口类型变量err的动态类型为*errors.errorString。</p><p>那接口的这种“动静皆备”的特性，又带来了什么好处呢？</p><p>首先，接口类型变量在程序运行时可以被赋值为不同的动态类型变量，每次赋值后，接口类型变量中存储的动态类型信息都会发生变化，这让Go语言可以像动态语言（比如Python）那样拥有使用<a href="https://en.wikipedia.org/wiki/Duck_typing">Duck Typing（鸭子类型）</a>的灵活性。所谓鸭子类型，就是指某类型所表现出的特性（比如是否可以作为某接口类型的右值），不是由其基因（比如C++中的父类）决定的，而是由类型所表现出来的行为（比如类型拥有的方法）决定的。</p><p>比如下面的例子：</p><pre><code class="language-plain">type QuackableAnimal interface {
    Quack()
}

type Duck struct{}

func (Duck) Quack() {
    println("duck quack!")
}

type Dog struct{}

func (Dog) Quack() {
    println("dog quack!")
}

type Bird struct{}

func (Bird) Quack() {
    println("bird quack!")
}                         
                          
func AnimalQuackInForest(a QuackableAnimal) {
    a.Quack()             
}                         
                          
func main() {             
    animals := []QuackableAnimal{new(Duck), new(Dog), new(Bird)}
    for _, animal := range animals {
        AnimalQuackInForest(animal)
    }  
}
</code></pre><p>这个例子中，我们用接口类型QuackableAnimal来代表具有“会叫”这一特征的动物，而Duck、Bird和Dog类型各自都具有这样的特征，于是我们可以将这三个类型的变量赋值给QuackableAnimal接口类型变量a。每次赋值，变量a中存储的动态类型信息都不同，Quack方法的执行结果将根据变量a中存储的动态类型信息而定。</p><p>这里的Duck、Bird、Dog都是“鸭子类型”，但它们之间并没有什么联系，之所以能作为右值赋值给QuackableAnimal类型变量，只是因为他们表现出了QuackableAnimal所要求的特征罢了。</p><p>不过，与动态语言不同的是，Go接口还可以保证“动态特性”使用时的安全性。比如，编译器在编译期就可以捕捉到将int类型变量传给QuackableAnimal接口类型变量这样的明显错误，决不会让这样的错误遗漏到运行时才被发现。</p><p>接口类型的动静特性让我们看到了接口类型的强大，但在日常使用过程中，很多人都会产生各种困惑，其中最经典的一个困惑莫过于“nil的error值不等于nil”了。下面我们来详细看一下。</p><h2>nil error值 != nil</h2><p>这里我们直接来看一段改编自<a href="https://go.dev/doc/faq#nil_error">GO FAQ中的例子</a>的代码：</p><pre><code class="language-plain">type MyError struct {
    error
}

var ErrBad = MyError{
    error: errors.New("bad things happened"),
}

func bad() bool {
    return false
}

func returnsError() error {
    var p *MyError = nil
    if bad() {
        p = &amp;ErrBad
    }
    return p
}

func main() {
    err := returnsError()
    if err != nil {
        fmt.Printf("error occur: %+v\n", err)
        return
    }
    fmt.Println("ok")
}
</code></pre><p>在这个例子中，我们的关注点集中在returnsError这个函数上面。这个函数定义了一个<code>*MyError</code>类型的变量p，初值为nil。如果函数bad返回false，returnsError函数就会直接将p（此时p = nil）作为返回值返回给调用者，之后调用者会将returnsError函数的返回值（error接口类型）与nil进行比较，并根据比较结果做出最终处理。</p><p>如果你是一个初学者，我猜你的的思路大概是这样的：p为nil，returnsError返回p，那么main函数中的err就等于nil，于是程序输出<strong>ok</strong>后退出。</p><p>但真实的运行结果是什么样的呢？我们来看一下：</p><pre><code class="language-plain">error occur: &lt;nil&gt;
</code></pre><p>我们看到，示例程序并未如我们前面预期的那样输出ok。程序显然是进入了错误处理分支，输出了err的值。那这里就有一个问题了：明明returnsError函数返回的p值为nil，为什么却满足了<code>if err != nil</code>的条件进入错误处理分支呢？</p><p>要想弄清楚这个问题，我们需要进一步了解接口类型变量的内部表示。</p><h2>接口类型变量的内部表示</h2><p>接口类型“动静兼备”的特性也决定了它的变量的内部表示绝不像一个静态类型变量（如int、float64）那样简单，我们可以在<code>$GOROOT/src/runtime/runtime2.go</code>中找到接口类型变量在运行时的表示：</p><pre><code class="language-plain">// $GOROOT/src/runtime/runtime2.go
type iface struct {
    tab  *itab
    data unsafe.Pointer
}

type eface struct {
    _type *_type
    data  unsafe.Pointer
}
</code></pre><p>我们看到，在运行时层面，接口类型变量有两种内部表示：<code>iface</code>和<code>eface</code>，这两种表示分别用于不同的接口类型变量：</p><ul>
<li>eface用于表示没有方法的空接口（<strong>e</strong>mpty inter<strong>face</strong>）类型变量，也就是interface{}类型的变量；</li>
<li>iface用于表示其余拥有方法的接口<strong>i</strong>nter<strong>face</strong>类型变量。</li>
</ul><p>这两个结构的共同点是它们都有两个指针字段，并且第二个指针字段的功能相同，都是指向当前赋值给该接口类型变量的动态类型变量的值。</p><p>那它们的不同点在哪呢？就在于eface表示的空接口类型并没有方法列表，因此它的第一个指针字段指向一个<code>_type</code>类型结构，这个结构为该接口类型变量的动态类型的信息，它的定义是这样的：</p><pre><code class="language-plain">// $GOROOT/src/runtime/type.go

type _type struct {
    size       uintptr
    ptrdata    uintptr // size of memory prefix holding all pointers
    hash       uint32
    tflag      tflag
    align      uint8
    fieldAlign uint8
    kind       uint8
    // function for comparing objects of this type
    // (ptr to object A, ptr to object B) -&gt; ==?
    equal func(unsafe.Pointer, unsafe.Pointer) bool
    // gcdata stores the GC type data for the garbage collector.
    // If the KindGCProg bit is set in kind, gcdata is a GC program.
    // Otherwise it is a ptrmask bitmap. See mbitmap.go for details.
    gcdata    *byte
    str       nameOff
    ptrToThis typeOff
}
</code></pre><p>而iface除了要存储动态类型信息之外，还要存储接口本身的信息（接口的类型信息、方法列表信息等）以及动态类型所实现的方法的信息，因此iface的第一个字段指向一个<code>itab</code>类型结构。itab结构的定义如下：</p><pre><code class="language-plain">// $GOROOT/src/runtime/runtime2.go
type itab struct {
    inter *interfacetype
    _type *_type
    hash  uint32 // copy of _type.hash. Used for type switches.
    _     [4]byte
    fun   [1]uintptr // variable sized. fun[0]==0 means _type does not implement inter.
}
</code></pre><p>这里我们也可以看到，itab结构中的第一个字段<code>inter</code>指向的interfacetype结构，存储着这个接口类型自身的信息。你看一下下面这段代码表示的interfacetype类型定义， 这个interfacetype结构由类型信息（typ）、包路径名（pkgpath）和接口方法集合切片（mhdr）组成。</p><pre><code class="language-plain">// $GOROOT/src/runtime/type.go
type interfacetype struct {
    typ     _type
    pkgpath name
    mhdr    []imethod
}
</code></pre><p>itab结构中的字段<code>_type</code>则存储着这个接口类型变量的动态类型的信息，字段<code>fun</code>则是动态类型已实现的接口方法的调用地址数组。</p><p>下面我们再结合例子用图片来直观展现eface和iface的结构。首先我们看一个用eface表示的空接口类型变量的例子：</p><pre><code class="language-plain">type T struct {
    n int
    s string
}

func main() {
    var t = T {
        n: 17,
        s: "hello, interface",
    }
    
    var ei interface{} = t // Go运行时使用eface结构表示ei
}
</code></pre><p>这个例子中的空接口类型变量ei在Go运行时的表示是这样的：</p><p><img src="https://static001.geekbang.org/resource/image/2d/ae/2d8f103e2973d2e31c9f4237e6799eae.jpg?wh=1920x1047" alt="图片"></p><p>我们看到空接口类型的表示较为简单，图中上半部分_type字段指向它的动态类型T的类型信息，下半部分的data则是指向一个T类型的实例值。</p><p>我们再来看一个更复杂的用iface表示非空接口类型变量的例子：</p><pre><code class="language-plain">type T struct {
    n int
    s string
}

func (T) M1() {}
func (T) M2() {}

type NonEmptyInterface interface {
    M1()
    M2()
}

func main() {
    var t = T{
        n: 18,
        s: "hello, interface",
    }
    var i NonEmptyInterface = t
}
</code></pre><p>和eface比起来，iface的表示稍微复杂些。我也画了一幅表示上面NonEmptyInterface接口类型变量在Go运行时表示的示意图：</p><p><img src="https://static001.geekbang.org/resource/image/36/44/369810ba10b9b8792d8edfd8e931b344.jpg?wh=1980x1080" alt=""></p><p>由上面的这两幅图，我们可以看出，每个接口类型变量在运行时的表示都是由两部分组成的，针对不同接口类型我们可以简化记作：<code>eface(_type, data)</code>和<code>iface(tab, data)</code>。</p><p>而且，虽然eface和iface的第一个字段有所差别，但tab和_type可以统一看作是动态类型的类型信息。Go语言中每种类型都会有唯一的_type信息，无论是内置原生类型，还是自定义类型都有。Go运行时会为程序内的全部类型建立只读的共享_type信息表，因此拥有相同动态类型的同类接口类型变量的_type/tab信息是相同的。</p><p>而接口类型变量的data部分则是指向一个动态分配的内存空间，这个内存空间存储的是赋值给接口类型变量的动态类型变量的值。未显式初始化的接口类型变量的值为<code>nil</code>，也就是这个变量的_type/tab和data都为nil。</p><p>也就是说，我们判断两个接口类型变量是否相等，只需判断_type/tab以及data是否都相等即可。两个接口变量的_type/tab不同时，即两个接口变量的动态类型不相同时，两个接口类型变量一定不等。</p><p>当两个接口变量的_type/tab相同时，对data的相等判断要有区分。当接口变量的动态类型为指针类型时(*T)，Go不会再额外分配内存存储指针值，而会将动态类型的指针值直接存入data字段中，这样data值的相等性决定了两个接口类型变量是否相等；当接口变量的动态类型为非指针类型(T)时，我们判断的将不是data指针的值是否相等，而是判断data指针指向的内存空间所存储的数据值是否相等，若相等，则两个接口类型变量相等。</p><p>不过，通过肉眼去辨别接口类型变量是否相等总是困难一些，我们可以引入一些<strong>helper函数</strong>。借助这些函数，我们可以清晰地输出接口类型变量的内部表示，这样就可以一目了然地看出两个变量是否相等了。</p><p>由于eface和iface是runtime包中的非导出结构体定义，我们不能直接在包外使用，所以也就无法直接访问到两个结构体中的数据。不过，Go语言提供了println预定义函数，可以用来输出eface或iface的两个指针字段的值。</p><p>在编译阶段，编译器会根据要输出的参数的类型将println替换为特定的函数，这些函数都定义在<code>$GOROOT/src/runtime/print.go</code>文件中，而针对eface和iface类型的打印函数实现如下：</p><pre><code class="language-plain">// $GOROOT/src/runtime/print.go
func printeface(e eface) {
    print("(", e._type, ",", e.data, ")")
}

func printiface(i iface) {
    print("(", i.tab, ",", i.data, ")")
}
</code></pre><p>我们看到，printeface和printiface会输出各自的两个指针字段的值。下面我们就来使用println函数输出各类接口类型变量的内部表示信息，并结合输出结果，解析接口类型变量的等值比较操作。</p><h3>第一种：nil接口变量</h3><p>我们前面提过，未赋初值的接口类型变量的值为nil，这类变量也就是nil接口变量，我们来看这类变量的内部表示输出的例子：</p><pre><code class="language-plain">func printNilInterface() {
	// nil接口变量
	var i interface{} // 空接口类型
	var err error     // 非空接口类型
	println(i)
	println(err)
	println("i = nil:", i == nil)
	println("err = nil:", err == nil)
	println("i = err:", i == err)
}
</code></pre><p>运行这个函数，输出结果是这样的：</p><pre><code class="language-plain">(0x0,0x0)
(0x0,0x0)
i = nil: true
err = nil: true
i = err: true
</code></pre><p>我们看到，无论是空接口类型还是非空接口类型变量，一旦变量值为nil，那么它们内部表示均为<code>(0x0,0x0)</code>，也就是类型信息、数据值信息均为空。因此上面的变量i和err等值判断为true。</p><h3>第二种：空接口类型变量</h3><p>下面是空接口类型变量的内部表示输出的例子：</p><pre><code class="language-plain">  func printEmptyInterface() {
      var eif1 interface{} // 空接口类型
      var eif2 interface{} // 空接口类型
      var n, m int = 17, 18
  
      eif1 = n
      eif2 = m

      println("eif1:", eif1)
      println("eif2:", eif2)
      println("eif1 = eif2:", eif1 == eif2) // false
  
      eif2 = 17
      println("eif1:", eif1)
      println("eif2:", eif2)
      println("eif1 = eif2:", eif1 == eif2) // true
 
      eif2 = int64(17)
      println("eif1:", eif1)
      println("eif2:", eif2)
      println("eif1 = eif2:", eif1 == eif2) // false
 }
</code></pre><p>这个例子的运行输出结果是这样的：</p><pre><code class="language-plain">eif1: (0x10ac580,0xc00007ef48)
eif2: (0x10ac580,0xc00007ef40)
eif1 = eif2: false
eif1: (0x10ac580,0xc00007ef48)
eif2: (0x10ac580,0x10eb3d0)
eif1 = eif2: true
eif1: (0x10ac580,0xc00007ef48)
eif2: (0x10ac640,0x10eb3d8)
eif1 = eif2: false
</code></pre><p>我们按顺序分析一下这个输出结果。</p><p>首先，代码执行到第11行时，eif1与eif2已经分别被赋值整型值17与18，这样eif1和eif2的动态类型的类型信息是相同的（都是0x10ac580），但data指针指向的内存块中存储的值不同，一个是17，一个是18，于是eif1不等于eif2。</p><p>接着，代码执行到第16行的时候，eif2已经被重新赋值为17，这样eif1和eif2不仅存储的动态类型的类型信息是相同的（都是0x10ac580），data指针指向的内存块中存储值也相同了，都是17，于是eif1等于eif2。</p><p>然后，代码执行到第21行时，eif2已经被重新赋值了int64类型的数值17。这样，eif1和eif2存储的动态类型的类型信息就变成不同的了，一个是int，一个是int64，即便data指针指向的内存块中存储值是相同的，最终eif1与eif2也是不相等的。</p><p>从输出结果中我们可以总结一下：<strong>对于空接口类型变量，只有_type和data所指数据内容一致的情况下，两个空接口类型变量之间才能划等号</strong>。另外，Go在创建eface时一般会为data重新分配新内存空间，将动态类型变量的值复制到这块内存空间，并将data指针指向这块内存空间。因此我们多数情况下看到的data指针值都是不同的。</p><h3>第三种：非空接口类型变量</h3><p>这里，我们也直接来看一个非空接口类型变量的内部表示输出的例子：</p><pre><code class="language-plain">type T int

func (t T) Error() string { 
    return "bad error"
}

func printNonEmptyInterface() { 
    var err1 error // 非空接口类型
    var err2 error // 非空接口类型
    err1 = (*T)(nil)
    println("err1:", err1)
    println("err1 = nil:", err1 == nil)

    err1 = T(5)
    err2 = T(6)
    println("err1:", err1)
    println("err2:", err2)
    println("err1 = err2:", err1 == err2)

    err2 = fmt.Errorf("%d\n", 5)
    println("err1:", err1)
    println("err2:", err2)
    println("err1 = err2:", err1 == err2)
}   
</code></pre><p>这个例子的运行输出结果如下：</p><pre><code class="language-plain">err1: (0x10ed120,0x0)
err1 = nil: false
err1: (0x10ed1a0,0x10eb310)
err2: (0x10ed1a0,0x10eb318)
err1 = err2: false
err1: (0x10ed1a0,0x10eb310)
err2: (0x10ed0c0,0xc000010050)
err1 = err2: false
</code></pre><p>我们看到上面示例中每一轮通过println输出的err1和err2的tab和data值，要么data值不同，要么tab与data值都不同。</p><p>和空接口类型变量一样，只有tab和data指的数据内容一致的情况下，两个非空接口类型变量之间才能划等号。这里我们要注意err1下面的赋值情况：</p><pre><code class="language-plain">err1 = (*T)(nil)
</code></pre><p>针对这种赋值，println输出的err1是（0x10ed120, 0x0），也就是非空接口类型变量的类型信息并不为空，数据指针为空，因此它与nil（0x0,0x0）之间不能划等号。</p><p>现在我们再回到我们开头的那个问题，你是不是已经豁然开朗了呢？开头的问题中，从returnsError返回的error接口类型变量err的数据指针虽然为空，但它的类型信息（iface.tab）并不为空，而是*MyError对应的类型信息，这样err与nil（0x0,0x0）相比自然不相等，这就是我们开头那个问题的答案解析，现在你明白了吗？</p><h3>第四种：空接口类型变量与非空接口类型变量的等值比较</h3><p>下面是非空接口类型变量和空接口类型变量之间进行比较的例子：</p><pre><code class="language-plain">func printEmptyInterfaceAndNonEmptyInterface() {
	var eif interface{} = T(5)
	var err error = T(5)
	println("eif:", eif)
	println("err:", err)
	println("eif = err:", eif == err)

	err = T(6)
	println("eif:", eif)
	println("err:", err)
	println("eif = err:", eif == err)
}
</code></pre><p>这个示例的输出结果如下：</p><pre><code class="language-plain">eif: (0x10b3b00,0x10eb4d0)
err: (0x10ed380,0x10eb4d8)
eif = err: true
eif: (0x10b3b00,0x10eb4d0)
err: (0x10ed380,0x10eb4e0)
eif = err: false
</code></pre><p>你可以看到，空接口类型变量和非空接口类型变量内部表示的结构有所不同（第一个字段：_type vs. tab)，两者似乎一定不能相等。但Go在进行等值比较时，类型比较使用的是eface的_type和iface的tab._type，因此就像我们在这个例子中看到的那样，当eif和err都被赋值为<code>T(5)</code>时，两者之间是划等号的。</p><p>好了，到这里，我们已经学完了各类接口类型变量在运行时层的表示。我们可以通过println可以查看这个表示信息，从中我们也知道了接口变量只有在类型信息与值信息都一致的情况下才能划等号。</p><h2>输出接口类型变量内部表示的详细信息</h2><p>不过，println输出的接口类型变量的内部表示信息，在一般情况下都是足够的，但有些时候又显得过于简略，比如在上面最后一个例子中，如果仅凭<code>eif: (0x10b3b00,0x10eb4d0)</code>和<code>err: (0x10ed380,0x10eb4d8)</code>的输出，我们是无法想到两个变量是相等的。</p><p>那这时如果我们能输出接口类型变量内部表示的详细信息（比如：tab._type），那势必可以取得事半功倍的效果。接下来我们就看看这要怎么做。</p><p>前面提到过，eface和iface以及组成它们的itab和_type都是runtime包下的非导出结构体，我们无法在外部直接引用它们。但我们发现，组成eface、iface的类型都是基本数据类型，我们完全可以通过<strong>“复制代码”</strong>的方式将它们拿到runtime包外面来。</p><p>不过，这里要注意，由于runtime中的eface、iface，或者它们的组成可能会随着Go版本的变化发生变化，因此这个方法不具备跨版本兼容性。也就是说，基于Go 1.17版本复制的代码，可能仅适用于使用Go 1.17版本编译。这里我们就以Go 1.17版本为例看看：</p><pre><code class="language-plain">// dumpinterface.go 
type eface struct {
    _type *_type
    data  unsafe.Pointer
}

type tflag uint8
type nameOff int32
type typeOff int32

type _type struct {
    size       uintptr
    ptrdata    uintptr // size of memory prefix holding all pointers
    hash       uint32
    tflag      tflag
    align      uint8
    fieldAlign uint8
    kind       uint8
    // function for comparing objects of this type
    // (ptr to object A, ptr to object B) -&gt; ==?
    equal func(unsafe.Pointer, unsafe.Pointer) bool
    // gcdata stores the GC type data for the garbage collector.
    // If the KindGCProg bit is set in kind, gcdata is a GC program.
    // Otherwise it is a ptrmask bitmap. See mbitmap.go for details.
    gcdata    *byte
    str       nameOff
    ptrToThis typeOff
}

type iface struct {
    tab  *itab
    data unsafe.Pointer
}

type itab struct {
    inter *interfacetype
    _type *_type
    hash  uint32 // copy of _type.hash. Used for type switches.
    _     [4]byte
    fun   [1]uintptr // variable sized. fun[0]==0 means _type does not implement inter.
}

... ...

const ptrSize = unsafe.Sizeof(uintptr(0))

func dumpEface(i interface{}) {
    ptrToEface := (*eface)(unsafe.Pointer(&amp;i))
    fmt.Printf("eface: %+v\n", *ptrToEface)

    if ptrToEface._type != nil {
        // dump _type info
        fmt.Printf("\t _type: %+v\n", *(ptrToEface._type))
    }

    if ptrToEface.data != nil {
        // dump data
        switch i.(type) {
        case int:
            dumpInt(ptrToEface.data)
        case float64:
            dumpFloat64(ptrToEface.data)
        case T:
            dumpT(ptrToEface.data)

        // other cases ... ...
        default:
            fmt.Printf("\t unsupported data type\n")
        }
    }
    fmt.Printf("\n")
}

func dumpItabOfIface(ptrToIface unsafe.Pointer) {
    p := (*iface)(ptrToIface)
    fmt.Printf("iface: %+v\n", *p)

    if p.tab != nil {
        // dump itab
        fmt.Printf("\t itab: %+v\n", *(p.tab))
        // dump inter in itab
        fmt.Printf("\t\t inter: %+v\n", *(p.tab.inter))

        // dump _type in itab
        fmt.Printf("\t\t _type: %+v\n", *(p.tab._type))

        // dump fun in tab
        funPtr := unsafe.Pointer(&amp;(p.tab.fun))
        fmt.Printf("\t\t fun: [")
        for i := 0; i &lt; len((*(p.tab.inter)).mhdr); i++ {
            tp := (*uintptr)(unsafe.Pointer(uintptr(funPtr) + uintptr(i)*ptrSize))
            fmt.Printf("0x%x(%d),", *tp, *tp)
        }
        fmt.Printf("]\n")
    }
}

func dumpDataOfIface(i interface{}) {
    // this is a trick as the data part of eface and iface are same
    ptrToEface := (*eface)(unsafe.Pointer(&amp;i))

    if ptrToEface.data != nil {
        // dump data
        switch i.(type) {
        case int:
            dumpInt(ptrToEface.data)
        case float64:
            dumpFloat64(ptrToEface.data)
        case T:
            dumpT(ptrToEface.data)

        // other cases ... ...

        default:
            fmt.Printf("\t unsupported data type\n")
        }
    }
    fmt.Printf("\n")
}

func dumpT(dataOfIface unsafe.Pointer) {
    var p *T = (*T)(dataOfIface)
    fmt.Printf("\t data: %+v\n", *p)
}
... ...

</code></pre><p>这里我挑选了关键部分，省略了部分代码。上面这个dumpinterface.go中提供了三个主要函数:</p><ul>
<li>dumpEface: 用于输出空接口类型变量的内部表示信息；</li>
<li>dumpItabOfIface: 用于输出非空接口类型变量的tab字段信息；</li>
<li>dumpDataOfIface: 用于输出非空接口类型变量的data字段信息；</li>
</ul><p>我们利用这三个函数来输出一下前面printEmptyInterfaceAndNonEmptyInterface函数中的接口类型变量的信息：</p><pre><code class="language-plain">package main

import "unsafe"

type T int

func (t T) Error() string {
    return "bad error"
}

func main() {
    var eif interface{} = T(5)
    var err error = T(5)
    println("eif:", eif)
    println("err:", err)
    println("eif = err:", eif == err)
    
    dumpEface(eif)
    dumpItabOfIface(unsafe.Pointer(&amp;err))
    dumpDataOfIface(err)
}
</code></pre><p>运行这个示例代码，我们得到了这个输出结果：</p><pre><code class="language-plain">eif: (0x10b38c0,0x10e9b30)
err: (0x10eb690,0x10e9b30)
eif = err: true
eface: {_type:0x10b38c0 data:0x10e9b30}
	 _type: {size:8 ptrdata:0 hash:1156555957 tflag:15 align:8 fieldAlign:8 kind:2 equal:0x10032e0 gcdata:0x10e9a60 str:4946 ptrToThis:58496}
	 data: bad error

iface: {tab:0x10eb690 data:0x10e9b30}
	 itab: {inter:0x10b5e20 _type:0x10b38c0 hash:1156555957 _:[0 0 0 0] fun:[17454976]}
		 inter: {typ:{size:16 ptrdata:16 hash:235953867 tflag:7 align:8 fieldAlign:8 kind:20 equal:0x10034c0 gcdata:0x10d2418 str:3666 ptrToThis:26848} pkgpath:{bytes:&lt;nil&gt;} mhdr:[{name:2592 ityp:43520}]}
		 _type: {size:8 ptrdata:0 hash:1156555957 tflag:15 align:8 fieldAlign:8 kind:2 equal:0x10032e0 gcdata:0x10e9a60 str:4946 ptrToThis:58496}
		 fun: [0x10a5780(17454976),]
	 data: bad error
</code></pre><p>从输出结果中，我们看到eif的_type（0x10b38c0）与err的tab._type（0x10b38c0）是一致的，data指针所指内容（“bad error”）也是一致的，因此<code>eif == err</code>表达式的结果为true。</p><p>再次强调一遍，上面这个实现可能仅在Go 1.17版本上测试通过，并且在输出iface或eface的data部分内容时只列出了int、float64和T类型的数据读取实现，没有列出全部类型的实现，你可以根据自己的需要实现其余数据类型。dumpinterface.go的完整代码你可以在<a href="https://github.com/bigwhite/publication/tree/master/column/timegeek/go-first-course/29">这里</a>找到。</p><p>我们现在已经知道了，接口类型有着复杂的内部结构，所以我们将一个类型变量值赋值给一个接口类型变量值的过程肯定不会像<code>var i int = 5</code>那么简单，那么接口类型变量赋值的过程是怎样的呢？其实接口类型变量赋值是一个“装箱”的过程。</p><h2>接口类型的装箱（boxing）原理</h2><p><strong>装箱（boxing）</strong>是编程语言领域的一个基础概念，一般是指把一个值类型转换成引用类型，比如在支持装箱概念的Java语言中，将一个int变量转换成Integer对象就是一个装箱操作。</p><p>在Go语言中，将任意类型赋值给一个接口类型变量也是<strong>装箱</strong>操作。有了前面对接口类型变量内部表示的学习，我们知道<strong>接口类型的装箱实际就是创建一个eface或iface的过程</strong>。接下来我们就来简要描述一下这个过程，也就是接口类型的装箱原理。</p><p>我们基于下面这个例子中的接口装箱操作来说明：</p><pre><code class="language-plain">// interface_internal.go

  type T struct {
      n int
      s string
  }
  
  func (T) M1() {}
  func (T) M2() {}
  
  type NonEmptyInterface interface {
      M1()
      M2()
  }
  
  func main() {
      var t = T{
          n: 17,
          s: "hello, interface",
      }
      var ei interface{}
      ei = t
 
      var i NonEmptyInterface
      i = t
      fmt.Println(ei)
      fmt.Println(i)
  }
</code></pre><p>这个例子中，对ei和i两个接口类型变量的赋值都会触发装箱操作，要想知道Go在背后做了些什么，我们需要“下沉”一层，也就是要输出上面Go代码对应的汇编代码：</p><pre><code class="language-plain">$go tool compile -S interface_internal.go &gt; interface_internal.s
</code></pre><p>对应<code>ei = t</code>一行的汇编如下：</p><pre><code class="language-plain">    0x0026 00038 (interface_internal.go:24) MOVQ    $17, ""..autotmp_15+104(SP)
    0x002f 00047 (interface_internal.go:24) LEAQ    go.string."hello, interface"(SB), CX
    0x0036 00054 (interface_internal.go:24) MOVQ    CX, ""..autotmp_15+112(SP)
    0x003b 00059 (interface_internal.go:24) MOVQ    $16, ""..autotmp_15+120(SP)
    0x0044 00068 (interface_internal.go:24) LEAQ    type."".T(SB), AX
    0x004b 00075 (interface_internal.go:24) LEAQ    ""..autotmp_15+104(SP), BX
    0x0050 00080 (interface_internal.go:24) PCDATA  $1, $0
    0x0050 00080 (interface_internal.go:24) CALL    runtime.convT2E(SB)
</code></pre><p>对应i = t一行的汇编如下：</p><pre><code class="language-plain">    0x005f 00095 (interface_internal.go:27) MOVQ    $17, ""..autotmp_15+104(SP)
    0x0068 00104 (interface_internal.go:27) LEAQ    go.string."hello, interface"(SB), CX
    0x006f 00111 (interface_internal.go:27) MOVQ    CX, ""..autotmp_15+112(SP)
    0x0074 00116 (interface_internal.go:27) MOVQ    $16, ""..autotmp_15+120(SP)
    0x007d 00125 (interface_internal.go:27) LEAQ    go.itab."".T,"".NonEmptyInterface(SB), AX
    0x0084 00132 (interface_internal.go:27) LEAQ    ""..autotmp_15+104(SP), BX
    0x0089 00137 (interface_internal.go:27) PCDATA  $1, $1
    0x0089 00137 (interface_internal.go:27) CALL    runtime.convT2I(SB)
</code></pre><p>在将动态类型变量赋值给接口类型变量语句对应的汇编代码中，我们看到了<code>convT2E</code>和<code>convT2I</code>两个runtime包的函数。这两个函数的实现位于<code>$GOROOT/src/runtime/iface.go</code>中：</p><pre><code class="language-plain">// $GOROOT/src/runtime/iface.go

func convT2E(t *_type, elem unsafe.Pointer) (e eface) {
    if raceenabled {
        raceReadObjectPC(t, elem, getcallerpc(), funcPC(convT2E))
    }
    if msanenabled {
        msanread(elem, t.size)
    }
    x := mallocgc(t.size, t, true)
    typedmemmove(t, x, elem)
    e._type = t
    e.data = x
    return
}

func convT2I(tab *itab, elem unsafe.Pointer) (i iface) {
    t := tab._type
    if raceenabled {
        raceReadObjectPC(t, elem, getcallerpc(), funcPC(convT2I))
    }
    if msanenabled {
        msanread(elem, t.size)
    }
    x := mallocgc(t.size, t, true)
    typedmemmove(t, x, elem)
    i.tab = tab
    i.data = x
    return
}
</code></pre><p>convT2E用于将任意类型转换为一个eface，convT2I用于将任意类型转换为一个iface。两个函数的实现逻辑相似，主要思路就是根据传入的类型信息（convT2E的_type和convT2I的tab._type）分配一块内存空间，并将elem指向的数据拷贝到这块内存空间中，最后传入的类型信息作为返回值结构中的类型信息，返回值结构中的数据指针（data）指向新分配的那块内存空间。</p><p>由此我们也可以看出，经过装箱后，箱内的数据，也就是存放在新分配的内存空间中的数据与原变量便无瓜葛了，比如下面这个例子：</p><pre><code class="language-plain">func main() {
	var n int = 61
	var ei interface{} = n
	n = 62  // n的值已经改变
	fmt.Println("data in box:", ei) // 输出仍是61
}
</code></pre><p>那么convT2E和convT2I函数的类型信息是从何而来的呢？</p><p>其实这些都依赖Go编译器的工作。编译器知道每个要转换为接口类型变量（toType）和动态类型变量的类型（fromType），它会根据这一对类型选择适当的convT2X函数，并在生成代码时使用选出的convT2X函数参与装箱操作。</p><p>不过，装箱是一个有性能损耗的操作，因此Go也在不断对装箱操作进行优化，包括对常见类型如整型、字符串、切片等提供系列快速转换函数：</p><pre><code class="language-plain">// $GOROOT/src/runtime/iface.go
func convT16(val any) unsafe.Pointer     // val must be uint16-like
func convT32(val any) unsafe.Pointer     // val must be uint32-like
func convT64(val any) unsafe.Pointer     // val must be uint64-like
func convTstring(val any) unsafe.Pointer // val must be a string
func convTslice(val any) unsafe.Pointer  // val must be a slice
</code></pre><p>这些函数去除了typedmemmove操作，增加了零值快速返回等特性。</p><p>同时Go建立了staticuint64s区域，对255以内的小整数值进行装箱操作时<a href="https://github.com/golang/go/issues/17725">不再分配新内存</a>，而是利用staticuint64s区域的内存空间，下面是staticuint64s的定义：</p><pre><code class="language-plain">// $GOROOT/src/runtime/iface.go
// staticuint64s is used to avoid allocating in convTx for small integer values.
var staticuint64s = [...]uint64{
    0x00, 0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07,
    0x08, 0x09, 0x0a, 0x0b, 0x0c, 0x0d, 0x0e, 0x0f,
	... ...
}
</code></pre><h2>小结</h2><p>好了，今天的课讲到这里就结束了，现在我们一起来回顾一下吧。</p><p>接口类型作为参与构建Go应用骨架的重要参与者，在Go语言中有着很高的地位。它这个地位的取得离不开它拥有的“动静兼备”的语法特性。Go接口的动态特性让Go拥有与动态语言相近的灵活性，而静态特性又在编译阶段保证了这种灵活性的安全。</p><p>要更好地理解Go接口的这两种特性，我们需要深入到Go接口在运行时的表示层面上去。接口类型变量在运行时表示为eface和iface，eface用于表示空接口类型变量，iface用于表示非空接口类型变量。只有两个接口类型变量的类型信息（eface._type/iface.tab._type）相同，且数据指针（eface.data/iface.data）所指数据相同时，两个接口类型变量才是相等的。</p><p>我们可以通过println输出接口类型变量的两部分指针变量的值。而且，通过拷贝runtime包eface和iface相关类型源码，我们还可以自定义输出eface/iface详尽信息的函数，不过要注意的是，由于runtime层代码的演进，这个函数可能不具备在Go版本间的移植性。</p><p>最后，接口类型变量的赋值本质上是一种装箱操作，装箱操作是由Go编译器和运行时共同完成的，有一定的性能开销，对于性能敏感的系统来说，我们应该尽量避免或减少这类装箱操作。</p><h2>思考题</h2><p>像nil error值 != nil那个例子中的“坑”你在日常编码时有遇到过吗？可以和我们分享一下吗？另外，我们这节课中的这个例子如何修改，才能让它按我们最初的预期结果输出呢？</p><p>欢迎在留言区分享你的经验和想法。也欢迎你把这节课分享给更多对Go接口感兴趣的朋友。我是Tony Bai，我们下节课见。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/53/a8/abc96f70.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>return</span>
  </div>
  <div class="_2_QraFYR_0">老师讲的太好， 这一篇 知识密度相当大啊， <br>就这一篇就值专栏的价格了。<br>感谢老师如此用心的输出。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 受宠若惊😁</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-29 12:15:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/75/bc/e24e181e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Calvin</span>
  </div>
  <div class="_2_QraFYR_0">思考题有2 种方法：<br>1）returnsError() 函数不返回 error 非空接口类型，而是直接返回结构体指针 *MyError（明确的类型，阻止自动装箱）；<br>2）不要直接 err != nil 这样判断，而是使用类型断言来判断：<br>if e, ok := err.(*MyError); ok &amp;&amp; e != nil {<br>    fmt.Printf(&quot;error occur: %+v\n&quot;, e)<br>    return<br>}<br><br>PS：Go 的“接口”在编程中需要特别注意，必须搞清楚接口类型变量在运行时的表示，以避免踩坑！！！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-05 11:43:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/11/66/ac631a36.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geralt</span>
  </div>
  <div class="_2_QraFYR_0">修改方法:<br>1. 把returnsError()里面p的类型改为error<br>2. 删除p，直接return &amp;ErrBad或者nil</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: ✅</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-29 03:50:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/c9/d9/00870178.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Slowdive</span>
  </div>
  <div class="_2_QraFYR_0">老师， 请问这里发生装箱了吗？ 返回类型是error， 是一个接口， p是*MyError， p的方法列表覆盖了error这个接口， 所以是可以赋值给error类型的变量。 <br>这个过程发生了隐式转换，赋值给接口类型，做装箱创建iface， <br>p != nil就成了 (&amp;tab, 0x0) != (0x0, 0x0)<br><br>func returnsError() error {    <br>    var p *MyError = nil    <br>    if bad() {<br>        p = &amp;ErrBad<br>    }<br>    return p<br>}<br><br>这样理解对吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 正确。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-20 13:40:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/1d/de/62bfa83f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>aoe</span>
  </div>
  <div class="_2_QraFYR_0">原来装箱是这样：将任意类型赋值给一个接口类型变量就是装箱操作。<br>接口类型的装箱实际就是创建一个 eface 或 iface 的过程</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-03 15:20:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/b0/6e/921cb700.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>在下宝龙、</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，在   eif2 = 17 这个操作后，输出后的data  ,0xc00007ef48 和0x10eb3d0 不相等呀，为甚么说他们是一样的<br>eif1: (0x10ac580,0xc00007ef48)<br>eif2: (0x10ac580,0x10eb3d0)</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 判相等不要看data指针的值，要看data指针指向的内存块中存储的值是否相同。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-29 13:00:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/GJXKh8OG00U5ial64plAIibbIuwkzhPc8uYic9Hibl8SbqvhnS2JImHgCD4JGvTktiaVnCjHQWbA5wicaxRUN5aTEWnQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_a6104e</span>
  </div>
  <div class="_2_QraFYR_0">eif: (0x10b38c0,0x10e9b30)<br>err: (0x10eb690,0x10e9b30)<br>eif = err: true<br>eface: {_type:0x10b38c0 data:0x10e9b30}<br>   _type: {size:8 ptrdata:0 hash:1156555957 tflag:15 align:8 fieldAlign:8 kind:2 equal:0x10032e0 gcdata:0x10e9a60 str:4946 ptrToThis:58496}<br>   data: bad error<br><br>iface: {tab:0x10eb690 data:0x10e9b30}<br>   itab: {inter:0x10b5e20 _type:0x10b38c0 hash:1156555957 _:[0 0 0 0] fun:[17454976]}<br>     inter: {typ:{size:16 ptrdata:16 hash:235953867 tflag:7 align:8 fieldAlign:8 kind:20 equal:0x10034c0 gcdata:0x10d2418 str:3666 ptrToThis:26848} pkgpath:{bytes:&lt;nil&gt;} mhdr:[{name:2592 ityp:43520}]}<br>     _type: {size:8 ptrdata:0 hash:1156555957 tflag:15 align:8 fieldAlign:8 kind:2 equal:0x10032e0 gcdata:0x10e9a60 str:4946 ptrToThis:58496}<br>     fun: [0x10a5780(17454976),]<br>   data: bad error 请问为什么data会是bad error不应该是5吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 好问题。<br><br>为什么输出bad error而不是5，是因为我们的dumpT函数的实现：<br><br>func dumpT(dataOfIface unsafe.Pointer) {<br>    var p *T = (*T)(dataOfIface)<br>    fmt.Printf(&quot;\t data: %+v\n&quot;, *p)<br>}<br><br>这里的Printf使用了%+v。<br><br>在标准库fmt包的manual（https:&#47;&#47;pkg.go.dev&#47;fmt）中有，当verb为%v时，如果操作数实现了error接口，那么Printf将会调用这个操作数的Error方法将其转换为字符串。<br><br>原文：If an operand implements the error interface, the Error method will be invoked to convert the object to a string<br><br>所以这里输出的是bad error。<br><br>可以再举一个简单的例子：<br><br>package main<br>  <br>import &quot;fmt&quot;<br><br>type T int<br><br>func (t T) Error() string {<br>    return &quot;bad error&quot;<br>}<br><br>func main() {<br>    var t = T(5)<br><br>    fmt.Printf(&quot;%d\n&quot;, t) &#47;&#47; 5<br>    fmt.Printf(&quot;%v\n&quot;, t) &#47;&#47; bad error<br>}<br><br><br><br><br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-04 09:09:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/75/bc/e24e181e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Calvin</span>
  </div>
  <div class="_2_QraFYR_0">Go 指针这块，感觉可以单独抽出一讲来讲下，并且结合unsafe 讲解，不知道大白老师能否满足大家的愿望呢？😂</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 好多人提出来了，后续定弄个加餐说说指针。不过需要把所有正文都更完后，编辑老师催的紧，你了解的:)</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-05 23:44:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/16/48/01567df1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>郑泽洲</span>
  </div>
  <div class="_2_QraFYR_0">请教老师，接口类型装箱过程为什么普遍要把原来的值复制一份到data？（除了staticuint64s等特例）直接用原来的值不行吗，还能提升点性能</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 好问题！<br><br>假设按照你说的，interface中直接用原先的值，那么interface类型在runtime中的表示一定是(type, ptr)的二元组。而ptr指向原值的地址。这样的情况下，看个例子：<br><br>func foo(i interface{}) {<br>   i.(int) = 8<br>}<br><br>var a int = 6<br>var i interface{} = a<br>i.(int) = 7<br>println(a) &#47;&#47; a = 7 这似乎还说得过去。<br><br>但是如果将i传递给函数foo：<br>foo(i) <br><br>foo对i的修改将都反映到a上：<br><br>println(a) &#47;&#47; a = 8<br><br>这与值拷贝语义似乎有悖。<br><br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-26 11:29:06</div>
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
  <div class="_2_QraFYR_0">大白老师的这一节干货很多，读的意犹未尽。有几个疑惑点，麻烦老师解忧。<br><br>1. 文中类似：“_type” 这种命名，前面加下划线，这种有什么含义呢？<br><br>2. 文中关于打印两类接口内部详细信息的代码中，运用了大量的 * 还有 &amp; 再加上  unsafe.Pointer 的使用，看起来会非常困惑，希望老师后面能讲一讲Go的指针吧。刚从动态语言转过来，确实应该好好理解一下。不然后面写出来的代码一定会有很多潜在的风险。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. 没有啥特殊含义。我们自己写代码，不要用以下划线为前缀的命名方式。<br>2. 指针加餐后续应该会加上。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-31 23:19:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/59/28/62ee741f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>埼玉</span>
  </div>
  <div class="_2_QraFYR_0">返回 *MyError 而不是 error</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: ✅</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-29 22:54:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/31/2a/4e/a3f53cae.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>撕影</span>
  </div>
  <div class="_2_QraFYR_0"> 装箱 inerface=struct 和前面说的 i.(T) 好像一对反操作</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-12 10:27:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/98/b1/f89a84d0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Tokamak</span>
  </div>
  <div class="_2_QraFYR_0">白老师我将 returnsErro()改为如下的方式,<br>func returnsError() error {<br>	var p MyError<br>	return p<br>}<br>然后在main()中使用<br>err := returnsError()<br>if err != nil {<br>	fmt.Printf(&quot;error :%+v\n&quot;, err)  &#47;&#47; 输出: error :%!v(PANIC=Error method: runtime error: invalid memory address or nil pointer dereferenc<br>}<br><br>如果在MyError 显式实现 error的Error()函数, 就不会报错了, 即:<br>func (MyError) Error() string {<br>	return &quot;bad things happend&quot;<br>}<br><br>我用 dumpItabOfIface(unsafe.Pointer(&amp;err)) 查看一下输出, 发现不管是否显式实现 MyError 中的 Error(),<br>tab.fun 字段都是有值的，因此就很疑惑为什么显式实现了 Error()就不会报错呢？ 麻烦白老师帮我解惑一下，谢谢~~</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 好问题！<br><br>问题在于fmt.Printf函数！它在打印error接口类型的实例时，会调用该实例的Error方法以获得表示该error类型的字符串。<br><br>在MyError未实现Error方法前，你说tab.fun也是有值的，这个值和你实现Error方法后的值一样么，我怀疑这个值是一个不可访问的地址。<br><br>如果这个地址是不可访问的地址，那么Printf调用导致panic就合情合理了。<br><br>但是当你手工实现了Error方法，那么这个tab.fun字段的值就应该是Error方法的合法地址，这时你Printf调用Error方法就不会报错。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-12 08:22:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/87/5f/6bf8b74a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Kepler</span>
  </div>
  <div class="_2_QraFYR_0">这篇有点高强度对抗啊</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 💪</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-27 11:01:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/5c/da/0a8bc27b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>文经</span>
  </div>
  <div class="_2_QraFYR_0">虽然是Go语言第一课，但这一部分讲得很深入，而且很厉害的一点是，把难以理解的技术细节隐藏的刚刚好，这一篇要再看几遍。白老师真是讲课的高手啊👍👍</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-14 19:26:15</div>
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
  <div class="_2_QraFYR_0">这次课很干。需要再学一遍</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-29 10:49:58</div>
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
  <div class="_2_QraFYR_0">var err error<br>err = errors.New(&quot;error1&quot;)<br>fmt.Printf(&quot;%T\n&quot;, err)  &#47;&#47; *errors.errorString<br><br>为什么说这里的 errors.New(&quot;error1&quot;) 赋值给err 是体现动态性了呢？<br>errors.New(&quot;error1&quot;) 能不能赋值给 err 不是也是在编译阶段就能知道了吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: “接口的动态特性，就体现在接口类型变量在运行时还存储了右值的真实类型信息，这个右值的真实类型被称为接口类型变量的动态类型”<br><br>例子中输出了err的动态类型：*error.errorString！<br><br>编译期能不能给err赋值是接口类型变量静态特性的体现。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-06 23:49:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/24/5f/f1/c66c8c51.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>吃两块云</span>
  </div>
  <div class="_2_QraFYR_0">实在是太干了，喝了两瓶水才咽下去。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">编辑回复: 哈哈哈哈，你太可爱了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-21 18:28:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/60/6d/3c2a5143.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>二进制傻瓜</span>
  </div>
  <div class="_2_QraFYR_0">木有看懂，还得多看几遍。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 加油💪</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-08 22:24:27</div>
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
  <div class="_2_QraFYR_0">nil error ！= nil的价值是啥，data为空，itab有类型信息的接口变量这种东西有什么具体使用场景吗？ 如果只是因为实现的原因，我觉得就是go在挖坑，典型的违背了直觉啊</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-30 18:03:00</div>
  </div>
</div>
</div>
</li>
</ul>