<audio title="16｜复合数据类型：原生map类型的实现机制是怎样的？" src="https://static001.geekbang.org/resource/audio/c9/3c/c93b64db020b1e0e10203yy9e36c813c.mp3" controls="controls"></audio> 
<p>你好，我是Tony Bai。</p><p>上一节课，我们学习了Go语言中最常用的两个复合类型：数组与切片。它们代表<strong>一组连续存储的同构类型元素集合。</strong>不同的是，数组的长度是确定的，而切片，我们可以理解为一种“动态数组”，它的长度在运行时是可变的。</p><p>这一节课，我们会继续前面的脉络，学习另外一种日常Go编码中比较常用的复合类型，这种类型可以让你将一个值（Value）唯一关联到一个特定的键（Key）上，可以用于实现特定键值的快速查找与更新，这个复合数据类型就是<strong>map</strong>。很多中文Go编程语言类技术书籍都会将它翻译为映射、哈希表或字典，但在我的课程中，<strong>为了保持原汁原味，我就直接使用它的英文名，map</strong>。</p><p>map是我们继切片之后，学到的第二个由Go编译器与运行时联合实现的复合数据类型，它有着复杂的内部实现，但却提供了十分简单友好的开发者使用接口。这一节课，我将从map类型的定义，到它的使用，再到map内部实现机制，由浅到深地让你吃透map类型。</p><h2>什么是map类型？</h2><p>map是Go语言提供的一种抽象数据类型，它表示一组无序的键值对。在后面的讲解中，我们会直接使用key和value分别代表map的键和值。而且，map集合中每个key都是唯一的：</p><!-- [[[read_end]]] --><p><img src="https://static001.geekbang.org/resource/image/f6/28/f6ac7392831455d3cf85f189a9dc9528.jpg?wh=1920x1047" alt="图片"></p><p>和切片类似，作为复合类型的map，它在Go中的类型表示也是由key类型与value类型组成的，就像下面代码：</p><pre><code class="language-plain">map[key_type]value_type
</code></pre><p>key与value的类型可以相同，也可以不同：</p><pre><code class="language-plain">map[string]string // key与value元素的类型相同
map[int]string    // key与value元素的类型不同
</code></pre><p>如果两个map类型的key元素类型相同，value元素类型也相同，那么我们可以说它们是同一个map类型，否则就是不同的map类型。</p><p>这里，我们要注意，map类型对value的类型没有限制，但是对key的类型却有严格要求，因为map类型要保证key的唯一性。Go语言中要求，<strong>key的类型必须支持“==”和“!=”两种比较操作符</strong>。</p><p>但是，在Go语言中，函数类型、map类型自身，以及切片只支持与nil的比较，而不支持同类型两个变量的比较。如果像下面代码这样，进行这些类型的比较，Go编译器将会报错：</p><pre><code class="language-plain">s1 := make([]int, 1)
s2 := make([]int, 2)
f1 := func() {}
f2 := func() {}
m1 := make(map[int]string)
m2 := make(map[int]string)
println(s1 == s2) // 错误：invalid operation: s1 == s2 (slice can only be compared to nil)
println(f1 == f2) // 错误：invalid operation: f1 == f2 (func can only be compared to nil)
println(m1 == m2) // 错误：invalid operation: m1 == m2 (map can only be compared to nil)
</code></pre><p>因此在这里，你一定要注意：<strong>函数类型、map类型自身，以及切片类型是不能作为map的key类型的</strong>。</p><p>知道如何表示一个map类型后，接下来，我们来看看如何声明和初始化一个map类型的变量。</p><h2>map变量的声明和初始化</h2><p>我们可以这样声明一个map变量：</p><pre><code class="language-plain">var m map[string]int // 一个map[string]int类型的变量
</code></pre><p>和切片类型变量一样，如果我们没有显式地赋予map变量初值，map类型变量的默认值为nil。</p><p>不过切片变量和map变量在这里也有些不同。初值为零值nil的切片类型变量，可以借助内置的append的函数进行操作，这种在Go语言中被称为“<strong>零值可用</strong>”。定义“零值可用”的类型，可以提升我们开发者的使用体验，我们不用再担心变量的初始状态是否有效。</p><p><strong>但map类型，因为它内部实现的复杂性，无法“零值可用”</strong>。所以，如果我们对处于零值状态的map变量直接进行操作，就会导致运行时异常（panic），从而导致程序进程异常退出：</p><pre><code class="language-plain">var m map[string]int // m = nil
m["key"] = 1         // 发生运行时异常：panic: assignment to entry in nil map
</code></pre><p>所以，我们必须对map类型变量进行显式初始化后才能使用。那我们怎样对map类型变量进行初始化呢？</p><p>和切片一样，为map类型变量显式赋值有两种方式：一种是使用复合字面值；另外一种是使用make这个预声明的内置函数。</p><p><strong>方法一：使用复合字面值初始化map类型变量。</strong></p><p>我们先来看这句代码：</p><pre><code class="language-plain">m := map[int]string{}
</code></pre><p>这里，我们显式初始化了map类型变量m。不过，你要注意，虽然此时map类型变量m中没有任何键值对，但变量m也不等同于初值为nil的map变量。这个时候，我们对m进行键值对的插入操作，不会引发运行时异常。</p><p>这里我们再看看怎么通过稍微复杂一些的复合字面值，对map类型变量进行初始化：</p><pre><code class="language-plain">m1 := map[int][]string{
    1: []string{"val1_1", "val1_2"},
    3: []string{"val3_1", "val3_2", "val3_3"},
    7: []string{"val7_1"},
}

type Position struct { 
    x float64 
    y float64
}

m2 := map[Position]string{
    Position{29.935523, 52.568915}: "school",
    Position{25.352594, 113.304361}: "shopping-mall",
    Position{73.224455, 111.804306}: "hospital",
}
</code></pre><p>我们看到，上面代码虽然完成了对两个map类型变量m1和m2的显式初始化，但不知道你有没有发现一个问题，作为初值的字面值似乎有些“臃肿”。你看，作为初值的字面值采用了复合类型的元素类型，而且在编写字面值时还带上了各自的元素类型，比如作为map[int] []string值类型的[]string，以及作为map[Position]string的key类型的Position。</p><p>别急！针对这种情况，Go提供了“语法糖”。这种情况下，<strong>Go允许省略字面值中的元素类型</strong>。因为map类型表示中包含了key和value的元素类型，Go编译器已经有足够的信息，来推导出字面值中各个值的类型了。我们以m2为例，这里的显式初始化代码和上面变量m2的初始化代码是等价的：</p><pre><code class="language-plain">m2 := map[Position]string{
    {29.935523, 52.568915}: "school",
    {25.352594, 113.304361}: "shopping-mall",
    {73.224455, 111.804306}: "hospital",
}
</code></pre><p>以后在无特殊说明的情况下，我们都将使用这种简化后的字面值初始化方式。</p><p><strong>方法二：使用make为map类型变量进行显式初始化。</strong></p><p>和切片通过make进行初始化一样，通过make的初始化方式，我们可以为map类型变量指定键值对的初始容量，但无法进行具体的键值对赋值，就像下面代码这样：</p><pre><code class="language-plain">m1 := make(map[int]string) // 未指定初始容量
m2 := make(map[int]string, 8) // 指定初始容量为8
</code></pre><p>不过，map类型的容量不会受限于它的初始容量值，当其中的键值对数量超过初始容量后，Go运行时会自动增加map类型的容量，保证后续键值对的正常插入。</p><p>了解完map类型变量的声明与初始化后，我们就来看看，在日常开发中，map类型都有哪些基本操作和注意事项。</p><h2>map的基本操作</h2><p>针对一个map类型变量，我们可以进行诸如插入新键值对、获取当前键值对数量、查找特定键和读取对应值、删除键值对，以及遍历键值等操作。我们一个个来学习。</p><p><strong>操作一：插入新键值对。</strong></p><p>面对一个非nil的map类型变量，我们可以在其中插入符合map类型定义的任意新键值对。插入新键值对的方式很简单，我们只需要把value赋值给map中对应的key就可以了：</p><pre><code class="language-plain">m := make(map[int]string)
m[1] = "value1"
m[2] = "value2"
m[3] = "value3"
</code></pre><p>而且，我们不需要自己判断数据有没有插入成功，因为Go会保证插入总是成功的。这里，Go运行时会负责map变量内部的内存管理，因此除非是系统内存耗尽，我们可以不用担心向map中插入新数据的数量和执行结果。</p><p>不过，如果我们插入新键值对的时候，某个key已经存在于map中了，那我们的插入操作就会用新值覆盖旧值：</p><pre><code class="language-plain">m := map[string]int {
	"key1" : 1,
	"key2" : 2,
}

m["key1"] = 11 // 11会覆盖掉"key1"对应的旧值1
m["key3"] = 3  // 此时m为map[key1:11 key2:2 key3:3]
</code></pre><p>从这段代码中你可以看到，map类型变量m在声明的同时就做了初始化，它的内部建立了两个键值对，其中就包含键key1。所以后面我们再给键key1进行赋值时，Go不会重新创建key1键，而是会用新值(11)把key1键对应的旧值(1)替换掉。</p><p><strong>操作二：获取键值对数量。</strong></p><p>如果我们在编码中，想知道当前map类型变量中已经建立了多少个键值对，那我们可以怎么做呢？和切片一样，map类型也可以通过内置函数<strong>len</strong>，获取当前变量已经存储的键值对数量：</p><pre><code class="language-plain">m := map[string]int {
	"key1" : 1,
	"key2" : 2,
}

fmt.Println(len(m)) // 2
m["key3"] = 3  
fmt.Println(len(m)) // 3
</code></pre><p>不过，这里要注意的是<strong>我们不能对map类型变量调用cap，来获取当前容量</strong>，这是map类型与切片类型的一个不同点。</p><p><strong>操作三：查找和数据读取</strong></p><p>和写入相比，map类型更多用在查找和数据读取场合。所谓查找，就是判断某个key是否存在于某个map中。有了前面向map插入键值对的基础，我们可能自然而然地想到，可以用下面代码去查找一个键并获得该键对应的值：</p><pre><code class="language-plain">m := make(map[string]int)
v := m["key1"]
</code></pre><p>乍一看，第二行代码在语法上好像并没有什么不当之处，但其实通过这行语句，我们还是无法确定键key1是否真实存在于map中。这是因为，当我们尝试去获取一个键对应的值的时候，如果这个键在map中并不存在，我们也会得到一个值，这个值是value元素类型的<strong>零值</strong>。</p><p>我们以上面这个代码为例，如果键key1在map中并不存在，那么v的值就会被赋予value元素类型int的零值，也就是0。所以我们无法通过v值判断出，究竟是因为key1不存在返回的零值，还是因为key1本身对应的value就是0。</p><p>那么在map中查找key的正确姿势是什么呢？Go语言的map类型支持通过用一种名为“<strong>comma ok</strong>”的惯用法，进行对某个key的查询。接下来我们就用“comma ok”惯用法改造一下上面的代码：</p><pre><code class="language-plain">m := make(map[string]int)
v, ok := m["key1"]
if !ok {
    // "key1"不在map中
}

// "key1"在map中，v将被赋予"key1"键对应的value
</code></pre><p>我们看到，这里我们通过了一个布尔类型变量ok，来判断键“key1”是否存在于map中。如果存在，变量v就会被正确地赋值为键“key1”对应的value。</p><p>不过，如果我们并不关心某个键对应的value，而只关心某个键是否在于map中，我们可以使用空标识符替代变量v，忽略可能返回的value：</p><pre><code class="language-plain">m := make(map[string]int)
_, ok := m["key1"]
... ...
</code></pre><p>因此，你一定要记住：<strong>在Go语言中，请使用“comma ok”惯用法对map进行键查找和键值读取操作。</strong></p><p><strong>操作四：删除数据。</strong></p><p>接下来，我们再看看看如何从map中删除某个键值对。在Go中，我们需要借助<strong>内置函数delete</strong>来从map中删除数据。使用delete函数的情况下，传入的第一个参数是我们的map类型变量，第二个参数就是我们想要删除的键。我们可以看看这个代码示例：</p><pre><code class="language-plain">m := map[string]int {
	"key1" : 1,
	"key2" : 2,
}

fmt.Println(m) // map[key1:1 key2:2]
delete(m, "key2") // 删除"key2"
fmt.Println(m) // map[key1:1]
</code></pre><p>这里要注意的是，<strong>delete函数是从map中删除键的唯一方法</strong>。即便传给delete的键在map中并不存在，delete函数的执行也不会失败，更不会抛出运行时的异常。</p><p><strong>操作五：遍历map中的键值数据</strong></p><p>最后，我们来说一下如何遍历map中的键值数据。这一点虽然不像查询和读取操作那么常见，但日常开发中我们还是有这个需求的。在Go中，遍历map的键值对只有一种方法，那就是<strong>像对待切片那样通过for range语句对map数据进行遍历</strong>。我们看一个例子：</p><pre><code class="language-plain">package main
  
import "fmt"

func main() {
    m := map[int]int{
        1: 11,
        2: 12,
        3: 13,
    }

    fmt.Printf("{ ")
    for k, v := range m {
        fmt.Printf("[%d, %d] ", k, v)
    }
    fmt.Printf("}\n")
}
</code></pre><p>你看，通过for range遍历map变量m，每次迭代都会返回一个键值对，其中键存在于变量k中，它对应的值存储在变量v中。我们可以运行一下这段代码，可以得到符合我们预期的结果：</p><pre><code class="language-plain">{ [1, 11] [2, 12] [3, 13] }
</code></pre><p>如果我们只关心每次迭代的键，我们可以使用下面的方式对map进行遍历：</p><pre><code class="language-plain">for k, _ := range m { 
	// 使用k
}
</code></pre><p>当然更地道的方式是这样的：</p><pre><code class="language-plain">for k := range m {
	// 使用k
}
</code></pre><p>如果我们只关心每次迭代返回的键所对应的value，我们同样可以通过空标识符替代变量k，就像下面这样：</p><pre><code class="language-plain">for _, v := range m {
	// 使用v
}
</code></pre><p>不过，前面map遍历的输出结果都非常理想，给我们的表象好像是迭代器按照map中元素的插入次序逐一遍历。那事实是不是这样呢？我们再来试试，多遍历几次这个map看看。</p><p>我们先来改造一下代码：</p><pre><code class="language-plain">package main
  
import "fmt"

func doIteration(m map[int]int) {
    fmt.Printf("{ ")
    for k, v := range m {
        fmt.Printf("[%d, %d] ", k, v)
    }
    fmt.Printf("}\n")
}

func main() {
    m := map[int]int{
        1: 11,
        2: 12,
        3: 13,
    }

    for i := 0; i &lt; 3; i++ {
        doIteration(m)
    }
}
</code></pre><p>运行一下上述代码，我们可以得到这样结果：</p><pre><code class="language-plain">{ [3, 13] [1, 11] [2, 12] }
{ [1, 11] [2, 12] [3, 13] }
{ [3, 13] [1, 11] [2, 12] }
</code></pre><p>我们可以看到，<strong>对同一map做多次遍历的时候，每次遍历元素的次序都不相同</strong>。这是Go语言map类型的一个重要特点，也是很容易让Go初学者掉入坑中的一个地方。所以这里你一定要记住：<strong>程序逻辑千万不要依赖遍历map所得到的的元素次序</strong>。</p><p>从我们前面的讲解，你应该也感受到了，map类型非常好用，那么，我们在各个函数方法间传递map变量会不会有很大开销呢？</p><h2>map变量的传递开销</h2><p>其实你不用担心开销的问题。</p><p>和切片类型一样，map也是引用类型。这就意味着map类型变量作为参数被传递给函数或方法的时候，实质上传递的只是一个<strong>“描述符”</strong>（后面我们再讲这个描述符究竟是什么)，而不是整个map的数据拷贝，所以这个传递的开销是固定的，而且也很小。</p><p>并且，当map变量被传递到函数或方法内部后，我们在函数内部对map类型参数的修改在函数外部也是可见的。比如你从这个示例中就可以看到，函数foo中对map类型变量m进行了修改，而这些修改在foo函数外也可见。</p><pre><code class="language-plain">package main
  
import "fmt"

func foo(m map[string]int) {
    m["key1"] = 11
    m["key2"] = 12
}

func main() {
    m := map[string]int{
        "key1": 1,
        "key2": 2,
    }

    fmt.Println(m) // map[key1:1 key2:2]  
    foo(m)
    fmt.Println(m) // map[key1:11 key2:12] 
}
</code></pre><h2>map的内部实现</h2><p>和切片相比，map类型的内部实现要更加复杂。Go运行时使用一张哈希表来实现抽象的map类型。运行时实现了map类型操作的所有功能，包括查找、插入、删除等。在编译阶段，Go编译器会将Go语法层面的map操作，重写成运行时对应的函数调用。大致的对应关系是这样的：</p><pre><code class="language-plain">// 创建map类型变量实例
m := make(map[keyType]valType, capacityhint) → m := runtime.makemap(maptype, capacityhint, m)

// 插入新键值对或给键重新赋值
m["key"] = "value" → v := runtime.mapassign(maptype, m, "key") v是用于后续存储value的空间的地址

// 获取某键的值 
v := m["key"]      → v := runtime.mapaccess1(maptype, m, "key")
v, ok := m["key"]  → v, ok := runtime.mapaccess2(maptype, m, "key")

// 删除某键
delete(m, "key")   → runtime.mapdelete(maptype, m, “key”)
</code></pre><p>这是map类型在Go运行时层实现的示意图：</p><p><img src="https://static001.geekbang.org/resource/image/c9/1f/c9b2d05ffcb3yyd3b7da06763ee46a1f.jpg?wh=1980x1080" alt=""></p><p>我们可以看到，和切片的运行时表示图相比，map的实现示意图显然要复杂得多。接下来，我们结合这张图来简要描述一下map在运行时层的实现原理。我们重点讲解一下一个map变量在初始状态、进行键值对操作后，以及在并发场景下的Go运行时层的实现原理。</p><h3>初始状态</h3><p>从图中我们可以看到，与语法层面 map 类型变量（m）一一对应的是*runtime.hmap 的实例，即runtime.hmap类型的指针，也就是我们前面在讲解 map 类型变量传递开销时提到的 <strong>map 类型的描述符</strong>。hmap 类型是 map 类型的头部结构（header），它存储了后续 map 类型操作所需的所有信息，包括：</p><p><img src="https://static001.geekbang.org/resource/image/2f/04/2f5ff72fbdb17cf0cb0b8da102c3e604.jpg?wh=1920x1047" alt="图片"></p><p>真正用来存储键值对数据的是桶，也就是bucket，每个bucket中存储的是Hash值低bit位数值相同的元素，默认的元素个数为 BUCKETSIZE（值为 8，Go 1.17版本中在$GOROOT/src/cmd/compile/internal/reflectdata/reflect.go中定义，与 runtime/map.go 中常量 bucketCnt 保持一致）。</p><p>当某个bucket（比如buckets[0])的8个空槽slot）都填满了，且map尚未达到扩容的条件的情况下，运行时会建立overflow bucket，并将这个overflow bucket挂在上面bucket（如buckets[0]）末尾的overflow指针上，这样两个buckets形成了一个链表结构，直到下一次map扩容之前，这个结构都会一直存在。</p><p>从图中我们可以看到，每个bucket由三部分组成，从上到下分别是tophash区域、key存储区域和value存储区域。</p><ul>
<li><strong>tophash区域</strong></li>
</ul><p>当我们向map插入一条数据，或者是从map按key查询数据的时候，运行时都会使用哈希函数对key做哈希运算，并获得一个哈希值（hashcode）。这个hashcode非常关键，运行时会把hashcode“一分为二”来看待，其中低位区的值用于选定bucket，高位区的值用于在某个bucket中确定key的位置。我把这一过程整理成了下面这张示意图，你理解起来可以更直观：</p><p><img src="https://static001.geekbang.org/resource/image/ef/08/ef729c06cd8fa19f29f89df212c7ea08.jpg?wh=1920x1047" alt="图片"></p><p>因此，每个bucket的tophash区域其实是用来快速定位key位置的，这样就避免了逐个key进行比较这种代价较大的操作。尤其是当key是size较大的字符串类型时，好处就更突出了。这是一种以空间换时间的思路。</p><ul>
<li><strong>key存储区域</strong></li>
</ul><p>接着，我们看tophash区域下面是一块连续的内存区域，存储的是这个bucket承载的所有key数据。运行时在分配bucket的时候需要知道key的Size。那么运行时是如何知道key的size的呢？</p><p>当我们声明一个map类型变量，比如var m map[string]int时，Go运行时就会为这个变量对应的特定map类型，生成一个runtime.maptype实例。如果这个实例已经存在，就会直接复用。maptype实例的结构是这样的：</p><pre><code class="language-plain">type maptype struct {
    typ        _type
    key        *_type
    elem       *_type
    bucket     *_type // internal type representing a hash bucket
    keysize    uint8  // size of key slot
    elemsize   uint8  // size of elem slot
    bucketsize uint16 // size of bucket
    flags      uint32
} 
</code></pre><p>我们可以看到，这个实例包含了我们需要的map类型中的所有"元信息"。我们前面提到过，编译器会把语法层面的map操作重写成运行时对应的函数调用，这些运行时函数都有一个共同的特点，那就是第一个参数都是maptype指针类型的参数。</p><p><strong>Go运行时就是利用maptype参数中的信息确定key的类型和大小的。</strong>map所用的hash函数也存放在maptype.key.alg.hash(key, hmap.hash0)中。同时maptype的存在也让Go中所有map类型都共享一套运行时map操作函数，而不是像C++那样为每种map类型创建一套map操作函数，这样就节省了对最终二进制文件空间的占用。</p><ul>
<li><strong>value存储区域</strong></li>
</ul><p>我们再接着看key存储区域下方的另外一块连续的内存区域，这个区域存储的是key对应的value。和key一样，这个区域的创建也是得到了maptype中信息的帮助。Go运行时采用了把key和value分开存储的方式，而不是采用一个kv接着一个kv的kv紧邻方式存储，这带来的其实是算法上的复杂性，但却减少了因内存对齐带来的内存浪费。</p><p>我们以map[int8]int64为例，看看下面的存储空间利用率对比图：</p><p><img src="https://static001.geekbang.org/resource/image/5b/5a/5bce9aaebc78bdea7d2999606891325a.jpg?wh=1920x1047" alt="图片"></p><p>你会看到，当前Go运行时使用的方案内存利用效率很高，而kv紧邻存储的方案在map[int8]int64这样的例子中内存浪费十分严重，它的内存利用率是72/128=56.25%，有近一半的空间都浪费掉了。</p><p>另外，还有一点我要跟你强调一下，如果key或value的数据长度大于一定数值，那么运行时不会在bucket中直接存储数据，而是会存储key或value数据的指针。目前Go运行时定义的最大key和value的长度是这样的：</p><pre><code class="language-plain">// $GOROOT/src/runtime/map.go
const (
    maxKeySize  = 128
    maxElemSize = 128
)
</code></pre><h3>map扩容</h3><p>我们前面提到过，map会对底层使用的内存进行自动管理。因此，在使用过程中，当插入元素个数超出一定数值后，map一定会存在自动扩容的问题，也就是怎么扩充bucket的数量，并重新在bucket间均衡分配数据的问题。</p><p>那么map在什么情况下会进行扩容呢？Go运行时的map实现中引入了一个LoadFactor（负载因子），当<strong>count &gt; LoadFactor * 2^B</strong>或overflow bucket过多时，运行时会自动对map进行扩容。目前Go最新1.17版本LoadFactor设置为6.5（loadFactorNum/loadFactorDen）。这里是Go中与map扩容相关的部分源码：</p><pre><code class="language-plain">// $GOROOT/src/runtime/map.go
const (
	... ...

	loadFactorNum = 13
	loadFactorDen = 2
	... ...
)

func mapassign(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	... ...
	if !h.growing() &amp;&amp; (overLoadFactor(h.count+1, h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
		hashGrow(t, h)
		goto again // Growing the table invalidates everything, so try again
	}
	... ...
}
</code></pre><p>这两方面原因导致的扩容，在运行时的操作其实是不一样的。如果是因为overflow bucket过多导致的“扩容”，实际上运行时会新建一个和现有规模一样的bucket数组，然后在assign和delete时做排空和迁移。</p><p>如果是因为当前数据数量超出LoadFactor指定水位而进行的扩容，那么运行时会建立一个<strong>两倍于现有规模的bucket数组</strong>，但真正的排空和迁移工作也是在assign和delete时逐步进行的。原bucket数组会挂在hmap的oldbuckets指针下面，直到原buckets数组中所有数据都迁移到新数组后，原buckets数组才会被释放。你可以结合下面的map扩容示意图来理解这个过程，这会让你理解得更深刻一些：</p><p><img src="https://static001.geekbang.org/resource/image/6e/29/6e94c7ee51a01fcf2267a7e2145d6929.jpg?wh=1920x1047" alt="图片"></p><h3>map与并发</h3><p>接着我们来看一下map和并发。从上面的实现原理来看，充当map描述符角色的hmap实例自身是有状态的（hmap.flags），而且对状态的读写是没有并发保护的。所以说map实例不是并发写安全的，也不支持并发读写。如果我们对map实例进行并发读写，程序运行时就会抛出异常。你可以看看下面这个并发读写map的例子：</p><pre><code class="language-plain">package main

import (
    "fmt"
    "time"
)

func doIteration(m map[int]int) {
    for k, v := range m {
        _ = fmt.Sprintf("[%d, %d] ", k, v)
    }
}

func doWrite(m map[int]int) {
    for k, v := range m {
        m[k] = v + 1
    }
}

func main() {
    m := map[int]int{
        1: 11,
        2: 12,
        3: 13,
    }

    go func() {
        for i := 0; i &lt; 1000; i++ {
            doIteration(m)
        }
    }()

    go func() {
        for i := 0; i &lt; 1000; i++ {
            doWrite(m)
        }
    }()

    time.Sleep(5 * time.Second)
}
</code></pre><p>运行这个示例程序，我们会得到下面的执行错误结果：</p><pre><code class="language-plain">fatal error: concurrent map iteration and map write
</code></pre><p>不过，如果我们仅仅是进行并发读，map是没有问题的。而且，Go 1.9版本中引入了支持并发写安全的sync.Map类型，可以在并发读写的场景下替换掉map。如果你有这方面的需求，可以查看一下<a href="https://pkg.go.dev/sync#Map">sync.Map的手册</a>。</p><p>另外，你要注意，考虑到map可以自动扩容，map中数据元素的value位置可能在这一过程中发生变化，所以<strong>Go不允许获取map中value的地址，这个约束是在编译期间就生效的</strong>。下面这段代码就展示了Go编译器识别出获取map中value地址的语句后，给出的编译错误：</p><pre><code class="language-plain">p := &amp;m[key]  // cannot take the address of m[key]
fmt.Println(p)
</code></pre><h2>小结</h2><p>好了，今天的课讲到这里就结束了。这一节课，我们讲解了Go语言的另一类十分常用的复合数据类型：map。</p><p>在Go语言中，map类型是一个无序的键值对的集合。它有两种类型元素，一类是键（key），另一类是值（value）。在一个map中，键是唯一的，在集合中不能有两个相同的键。Go也是通过这两种元素类型来表示一个map类型，你要记得这个通用的map类型表示：“map[key_type]value_type”。</p><p>map类型对key元素的类型是有约束的，它要求key元素的类型必须支持"==“和”!="两个比较操作符。value元素的类型可以是任意的。</p><p>不过，map类型变量声明后必须对它进行初始化后才能操作。map类型支持插入新键值对、查找和数据读取、删除键值对、遍历map中的键值数据等操作，Go为开发者提供了十分简单的操作接口。这里要你重点记住的是，我们在查找和数据读取时一定要使用“comma ok”惯用法。此外，map变量在函数与方法间传递的开销很小，并且在函数内部通过map描述符对map的修改会对函数外部可见。</p><p>另外，map的内部实现要比切片复杂得多，它是由Go编译器与运行时联合实现的。Go编译器在编译阶段会将语法层面的map操作，重写为运行时对应的函数调用。Go运行时则采用了高效的算法实现了map类型的各类操作，这里我建议你要结合Go项目源码来理解map的具体实现。</p><p>和切片一样，map是Go语言提供的重要数据类型，也是Gopher日常Go编码是最常使用的类型之一。我们在日常使用map的场合要把握住下面几个要点，不要走弯路：</p><ul>
<li>不要依赖map的元素遍历顺序；</li>
<li>map不是线程安全的，不支持并发读写；</li>
<li>不要尝试获取map中元素（value）的地址。</li>
</ul><h2>思考题</h2><p>通过上面的学习，我们知道对map类型进行遍历所得到的键的次序是随机的，那么我想请你思考并实现一个方法，让我们能对map的进行稳定次序遍历？期待在留言区看到你的想法。</p><p>欢迎你把这节课分享给更多对Go语言map类型感兴趣的朋友。我是Tony Bai，我们下节课见。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/26/27/eba94899.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>罗杰</span>
  </div>
  <div class="_2_QraFYR_0">还想深入研究 map 的小伙伴，可以去研究这些博客：<br>https:&#47;&#47;www.qcrao.com&#47;2019&#47;05&#47;22&#47;dive-into-go-map&#47;   <br>https:&#47;&#47;qcrao.com&#47;2020&#47;05&#47;06&#47;dive-into-go-sync-map&#47;<br>https:&#47;&#47;draveness.me&#47;golang&#47;docs&#47;part2-foundation&#47;ch03-datastructure&#47;golang-hashmap&#47;</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-17 13:20:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/lfMbV8RibrhFxjILg4550cZiaay64mTh5Zibon64TiaicC8jDMEK7VaXOkllHSpS582Jl1SUHm6Jib2AticVlHibiaBvUOA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>用0和1改变自己</span>
  </div>
  <div class="_2_QraFYR_0">可以用一个有序结构存储key,如slice,然后for这个slice,用key获取值。资料来源至：https:&#47;&#47;go.dev&#47;blog&#47;maps</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: ✅</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-17 11:13:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLBFkSq1oiaEMRjtyyv4ZpCI0OuaSsqs04ODm0OkZF6QhsAh3SvqhxibS2n7PLAVZE3QRSn5Hic0DyXg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ddh</span>
  </div>
  <div class="_2_QraFYR_0">请问一下老师， map 元素不能寻址， 是因为动态扩容， 那么切片不也有动态扩容吗。 为什么切片元素可以寻址呢？  难道切片动态扩容之后， 它的元素地址不会变吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 好问题！<br><br>关于这个问题，官方没有明确说明。<br><br>我觉得之所以map element是不可寻址的，还是复杂性和安全性的考虑。<br><br>Go slice和map都是复合数据类型，但两者的内部实现复杂度却远远不同。map要远复杂于slice。<br><br>slice的元素的地址要么在要么在原底层数组中，要么在扩容后的新数组中。并且slice扩容是一次完成的。<br>但map扩容是“蚂蚁搬家”，element位置复杂一些，可能在原bucket中、可能在overflow bucket中，也可能在新扩容的bucket中。<br><br>此外，一旦暴露map element的地址，比如我们用一个栈上的指针变量引用了该地址，当发生element移动时，go就要考虑连带变更这些栈上指针变量的指向，这些变更操作在每个map操作中都会导致运行时消耗的增加。<br>           <br>至于安全性，map有着一个复杂内部结构，这个结构环环相扣。一旦map element地址被暴露，通过该地址不仅可以修改value值，还可能会对map内部结构造成破坏，这种破坏将导致整个map工作不正常。而slice的element即便被hack，改掉的也只是元素值。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-25 16:25:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/83/c9/5d03981a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>thomas</span>
  </div>
  <div class="_2_QraFYR_0">go团队为什么要故意把map的遍历设置为随机？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这一话题要追溯到Go 1.0版本发布的时候，从Go 1.0版本的发布文档- https:&#47;&#47;go.dev&#47;doc&#47;go1中，我们能找到如下内容：<br><br><br>The old language specification did not define the order of iteration for maps, and in practice it differed across hardware platforms. This caused tests that iterated over maps to be fragile and non-portable, with the unpleasant property that a test might always pass on one machine but break on another.<br>                         <br>In Go 1, the order in which elements are visited when iterating over a map using a for range statement is defined to be unpredictable, even if the same loop is run multiple times with the same map. Code should not assume that the elements are visited in any particular order.<br>                         <br>This change means that code that depends on iteration order is very likely to break early and be fixed long before it becomes a problem. Just as important, it allows the map implementation to ensure better map balancing even when programs are using range loops to select an element from a map.                                <br>                                            <br>                                                                             <br>翻译过来，大致意思就是：1.0版本之前的语言规范中没有规定map的迭代次序，从而导致在实践中的一些糟糕的开发运行体验。于是Go团队选<br>择故意在map中加入随机迭代功能，这样一旦出现开发人员依赖key迭代顺序的错误行为，这一行为导致的问题在开发和测试早期就能被及时发现，而不会出现在生产运行环境中导致更大的危害。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-12 09:37:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/26/38/ef063dc2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Darren</span>
  </div>
  <div class="_2_QraFYR_0">可以参考java的LinkedHashMap,能实现插入有序或者访问有序，就是使用额外的链表来保存顺序。<br>go 中可以基于container&#47;list来实现。github上现成的功能。<br>https:&#47;&#47;github.com&#47;elliotchance&#47;orderedmap</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-17 23:09:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_244c46</span>
  </div>
  <div class="_2_QraFYR_0">map的key和value的值，是否可以为null</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 用代码回答你的问题：）<br><br>&#47;&#47; testnilasmapkey.go<br>package main<br><br>func main() {<br>	var m = make(map[*int]*int)<br>	m[nil] = nil<br><br>	println(m)<br>	println(len(m))<br><br>	var b = 5<br>	m[nil] = &amp;b<br>	p := m[nil]<br>	println(*p)<br><br>}<br><br>0xc0000466b0<br>1<br>5<br><br>nil作为value不新鲜。但nil可作为key可是很多人都不知道的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-26 19:46:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>flexiver</span>
  </div>
  <div class="_2_QraFYR_0">老师，想请问一下，hmap这个结构中的extra字段， 在key和value都不是指针的情况下，会存储所有的overflow bucket的指针，里面有提到一个内联，这个内联是什么意思？以及为什么当key和value都不是指针的情况下，会将bucket中的overflow指针全部放到extra字段存储</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 在Go项目源码(Go 1.17版本)的 src&#47;cmd&#47;compile&#47;internal&#47;reflectdata&#47;reflect.go中，func MapBucketType(t *types.Type) *types.Type&gt;这个函数实现中，我们可以勾勒出一个bucket桶的结构：<br><br>tophash<br>keys<br>values<br>overflow pointer<br><br>不过这个overflow pointer有两种情况：<br><br>      &#47;&#47; If keys and elems have no pointers, the map implementation<br>      &#47;&#47; can keep a list of overflow pointers on the side so that<br>      &#47;&#47; buckets can be marked as having no pointers.<br>      &#47;&#47; Arrange for the bucket to have no pointers by changing<br>      &#47;&#47; the type of the overflow field to uintptr in this case.<br>      &#47;&#47; See comment on hmap.overflow in runtime&#47;map.go.<br>      otyp := types.Types[types.TUNSAFEPTR]<br>      if !elemtype.HasPointers() &amp;&amp; !keytype.HasPointers() {<br>          otyp = types.Types[types.TUINTPTR]<br>      }<br><br>当key和value中都没有指针时，比如map[int]int。此时考虑到gc优化，编译器将overflow的类型设置为uintptr。<br><br>uintptr是一个整型，无法被GC识别，这样一来uintptr指向的overflow bucket就没有指向它的指针，这样gc就会将overflow bucket视为unrea<br>chable的mem块而将其释放掉。为了避免这种情况，hmap中的extra此时就会指向上面这类bucket的overflow bucket，保证key和value中都不含指针时，overflow bucket依旧可以不被gc。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-21 20:06:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/ce/d7/5315f6ce.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>不负青春不负己🤘</span>
  </div>
  <div class="_2_QraFYR_0">可以把key 赋值到变量，使用sort 排序，在遍历</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: ✅</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-19 23:06:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLBFkSq1oiaEMRjtyyv4ZpCI0OuaSsqs04ODm0OkZF6QhsAh3SvqhxibS2n7PLAVZE3QRSn5Hic0DyXg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ddh</span>
  </div>
  <div class="_2_QraFYR_0">把key存到有序切片中，用切片遍历</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是一个可行方法。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-17 09:55:25</div>
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
  <div class="_2_QraFYR_0">我看了《Go语言实践》，《Go专家编程》，《Go语言底层原理剖析》这几本书对map的描述，对比才发现白老师讲得最清晰，在封装和细节方面拿捏得最好，大大地赞👍 剩下不清楚的地方就只能自己看源码了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 过奖。其实原理这东西最终还是要落到代码上。而且不同版本实现也有差异。要想真正把原理吃透，还得自己多过几遍代码</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-05 14:58:05</div>
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
  <div class="_2_QraFYR_0">突然想到之前碰到的一个问题，就是golang 结构体作为map的元素时，不能够直接赋值给结构体的某个字段。这个也有“Go 不允许获取 map 中 value 的地址”的原因在嘛？假如这样的话，那么为什么读结构体中的某个字段是可以的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 读操作得到的是值的一个copy</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-22 11:13:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2b/26/bc/18314557.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>高雪斌</span>
  </div>
  <div class="_2_QraFYR_0">func doIteration(m map[int]int) {<br>	mu.RLock()<br>	defer mu.RUnlock()<br><br>	keys := []int{}<br><br>	for k := range m {<br>		keys = append(keys, k)<br>	}<br><br>	sort.SliceStable(keys, func(x, y int) bool {<br>		return x &lt; y<br>	})<br><br>	for _, k := range keys {<br>		fmt.Printf(&quot;[%d, %d] &quot;, k, m[k])<br>	}<br><br>	fmt.Println()<br>}<br><br>func doWrite(m map[int]int) {<br>	mu.Lock()<br>	defer mu.Unlock()<br><br>	for k, v := range m {<br>		m[k] = v + 1<br>	}<br>}<br><br>==&gt; 对并发示例代码的稳定排序输出 (原例1000次输出太多，输出前10个作为说明)：<br>[1, 11] [2, 12] [3, 13] <br>[1, 12] [2, 13] [3, 14] <br>[1, 13] [2, 14] [3, 15] <br>[1, 14] [2, 15] [3, 16] <br>[1, 15] [2, 16] [3, 17] <br>[1, 16] [2, 17] [3, 18] <br>[1, 17] [2, 18] [3, 19] <br>[1, 18] [2, 19] [3, 20] <br>[1, 19] [2, 20] [3, 21] <br>[1, 20] [2, 21] [3, 22] <br>。。。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不错的思路。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-17 17:07:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/f2/f5/b82f410d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Unknown element</span>
  </div>
  <div class="_2_QraFYR_0">老师我想问下这里：<br>“如果是因为 overflow bucket 过多导致的“扩容”，实际上运行时会新建一个和现有规模一样的 bucket 数组，然后在 assign 和 delete 时做排空和迁移。”<br>如果bucket规模不变的话，那所有key在hash之后还是分到原来的桶中，好像该overflow还是会overflow？主要是因为这里既没有通过真正增加桶的数量实现扩容，也没有通过更换hash函数改变key在桶中的分布，那么这个overflow的问题到底是怎么解决的呢？谢谢老师</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 好问题。这个等量扩容来自这个issue：https:&#47;&#47;github.com&#47;golang&#47;go&#47;issues&#47;16070  这是一种比较特殊的情况，即当持续向哈希中插入数据，然后又将它们全部删除时，这个过程如果触发增容扩容，就会导致overflow bucket过多，但此时原bucket中已经没有数据了。所以go引入了利用现有扩容机制的等量扩容 https:&#47;&#47;github.com&#47;golang&#47;go&#47;commit&#47;9980b70cb460f27907a003674ab1b9bea24a847c 将原bucket的overflow bucket中的数据重新挪到新bucket中。算法较复杂，专栏定位入门，所以没进行深入。感兴趣可以读一下这个cl commit的源码。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-03 15:08:29</div>
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
  <div class="_2_QraFYR_0">老师，请问一下如果是因为overflow bucket过多导致的&quot;扩容&quot;, 是否可以理解为这个hash函数的算法不太合理，导致大部分的key都分配到了一个bucket中，是否可以通过修改hash算法来重新hash一遍呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: go map底层的hash函数要考虑通用性。谁也不能预测用户会使用什么样的key数据。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-02 15:55:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/6e/6f/44da923f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>邹志鹏.Joey ⁷⁷⁷</span>
  </div>
  <div class="_2_QraFYR_0">既切片之后, 应该是 &quot;继切片之后&quot;?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍。稍后让编辑老师修改一下这个typo。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-18 15:51:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_d86547</span>
  </div>
  <div class="_2_QraFYR_0">老师 map遍历的随机性可以再详细说明一些嘛？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你看看https:&#47;&#47;tonybai.com&#47;go-course-faq中的补充是否满足你的需求。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-16 17:47:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/8c/60/58b6c39e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>zzy</span>
  </div>
  <div class="_2_QraFYR_0">想问下golang为何不像java那样，在底层使用红黑树，性能应该是更好的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 关于你提到的go内置类型map的实现的选型问题，我还真不清楚。我在公开资料里搜了一下，也没查到。不过我觉得，go team当初应该不单单是从性能这一个维度考虑的。毕竟是runtime层的实现，而不是一个单独的标准库包。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-13 15:18:41</div>
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
  <div class="_2_QraFYR_0">彩！只要key是有序的并且访问顺序是固定的，value也就可以确定顺序了，通过切片、数组可以实现</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: ✅</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-13 11:45:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/9d/cd/c21a01dd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>W-T</span>
  </div>
  <div class="_2_QraFYR_0">老师，自定义数据类型能否支持for range语句，支持的话需要实现什么接口</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不需要实现什么接口。只要自定义数据类型的底层类型(underlying type)是支持for range即可，比如切片、channel、数组、map等。<br><br>比如：<br><br>type Rangable []int<br><br>func main() {<br>	var r = Rangable{1, 2, 3, 4, 5, 6}<br>	for _, n := range r {<br>		println(n)<br>	}<br>}</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-22 09:16:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>大帅哥</span>
  </div>
  <div class="_2_QraFYR_0">想问一下，当map扩容后，有新的数据插入是，如果此时数据还没有迁移到新的bucket中，那么runtime是怎么知道这个可以是在olderbuckets和buckets的具体位置的，是不是计算规则不一样，但是这个规则具体是怎么样的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果在扩容过程中，那么新插入数据会触发迁移过程。如果key原先没有，则会直接在新buckets中，如果有，则会重新计算它在新buckets中的位置，然后写入新buckets中。原bucket标记为已迁移。<br><br> 迁移过程的比较繁琐，要想真正理解，还得自己read一下 runtime.mapassign的代码。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-06 21:17:18</div>
  </div>
</div>
</div>
</li>
</ul>