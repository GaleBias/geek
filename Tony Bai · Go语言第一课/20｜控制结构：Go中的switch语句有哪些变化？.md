<audio title="20｜控制结构：Go中的switch语句有哪些变化？" src="https://static001.geekbang.org/resource/audio/ee/66/ee1685e5f7b3yy4183ff384b77171166.mp3" controls="controls"></audio> 
<p>你好，我是Tony Bai。</p><p>经过前两节课的学习，我们已经掌握了控制结构中的分支结构以及循环结构。前面我们也提到过，在计算机世界中，再复杂的算法都可以通过顺序、分支和循环这三种基本的控制结构构造出来。所以，理论上讲，我们现在已经具备了实现任何算法的能力了。</p><p>不过理论归理论，我们还是要回到现实中来，继续学习Go语言中的控制结构，现在我们还差一种分支控制结构没讲。除了if语句之外，Go语言还提供了一种更适合多路分支执行的分支控制结构，也就是<strong>switch语句</strong>。</p><p>今天这一节课，我们就来系统学习一下switch语句。Go语言中的switch语句继承自它的先祖C语言，所以我们这一讲的重点是Go switch语句相较于C语言的switch，有哪些重要的改进与创新。</p><p>在讲解改进与创新之前，我们先来认识一下switch语句。</p><h2>认识switch语句</h2><p>我们先通过一个例子来直观地感受一下switch语句的优点。在一些执行分支较多的场景下，使用switch分支控制语句可以让代码更简洁，可读性更好。</p><p>比如下面例子中的readByExt函数会根据传入的文件扩展名输出不同的日志，它使用了if语句进行分支控制：</p><pre><code class="language-plain">func readByExt(ext string) {
    if ext == "json" {
        println("read json file")
    } else if ext == "jpg" || ext == "jpeg" || ext == "png" || ext == "gif" {
        println("read image file")
    } else if ext == "txt" || ext == "md" {
        println("read text file")
    } else if ext == "yml" || ext == "yaml" {
        println("read yaml file")
    } else if ext == "ini" {
        println("read ini file")
    } else {
        println("unsupported file extension:", ext)
    }
}
</code></pre><!-- [[[read_end]]] --><p>如果用switch改写上述例子代码，我们可以这样来写：</p><pre><code class="language-plain">func readByExtBySwitch(ext string) {
    switch ext {
    case "json":
        println("read json file")
    case "jpg", "jpeg", "png", "gif":
        println("read image file")
    case "txt", "md":
        println("read text file")
    case "yml", "yaml":
        println("read yaml file")
    case "ini":
        println("read ini file")
    default:
        println("unsupported file extension:", ext)
    }
}
</code></pre><p>从代码呈现的角度来看，针对这个例子，使用switch语句的实现要比if语句的实现更加简洁紧凑。并且，即便你这个时候还没有系统学过switch语句，相信你也能大致读懂上面readByExtBySwitch的执行逻辑。</p><p>简单来说，readByExtBySwitch函数就是将输入参数ext与每个case语句后面的表达式做比较，如果相等，就执行这个case语句后面的分支，然后函数返回。这里具体的执行逻辑，我们在后面再分析，现在你有个大概认识就好了。</p><p>接下来，我们就来进入正题，来看看Go语言中switch语句的一般形式：</p><pre><code class="language-plain">switch initStmt; expr {
    case expr1:
        // 执行分支1
    case expr2:
        // 执行分支2
    case expr3_1, expr3_2, expr3_3:
        // 执行分支3
    case expr4:
        // 执行分支4
    ... ...
    case exprN:
        // 执行分支N
    default: 
        // 执行默认分支
}
</code></pre><p>我们按语句顺序来分析一下。首先看这个switch语句一般形式中的第一行，这一行由switch关键字开始，它的后面通常接着一个表达式（expr），这句中的initStmt是一个可选的组成部分。和if、for语句一样，我们可以在initStmt中通过短变量声明定义一些在switch语句中使用的临时变量。</p><p>接下来，switch后面的大括号内是一个个代码执行分支，每个分支以case关键字开始，每个case后面是一个表达式或是一个逗号分隔的表达式列表。这里还有一个以default关键字开始的特殊分支，被称为<strong>默认分支</strong>。</p><p>最后，我们再来看switch语句的执行流程。其实也很简单，switch语句会用expr的求值结果与各个case中的表达式结果进行比较，如果发现匹配的case，也就是case后面的表达式，或者表达式列表中任意一个表达式的求值结果与expr的求值结果相同，那么就会执行该case对应的代码分支，分支执行后，switch语句也就结束了。如果所有case表达式都无法与expr匹配，那么程序就会执行default默认分支，并且结束switch语句。</p><p>那么问题就来了！在有多个case执行分支的switch语句中，<strong>Go是按什么次序对各个case表达式进行求值，并且与switch表达式（expr）进行比较的</strong>？</p><p>我们通过一段示例代码来回答这个问题。这是一个一般形式的switch语句，为了能呈现switch语句的执行次序，我以多个输出特定日志的函数作为switch表达式以及各个case表达式：</p><pre><code class="language-plain">func case1() int {
    println("eval case1 expr")
    return 1
}

func case2_1() int {
    println("eval case2_1 expr")
    return 0 
}
func case2_2() int {
    println("eval case2_2 expr")
    return 2 
}

func case3() int {
    println("eval case3 expr")
    return 3
}

func switchexpr() int {
    println("eval switch expr")
    return 2
}

func main() {
    switch switchexpr() {
    case case1():
        println("exec case1")
    case case2_1(), case2_2():
        println("exec case2")
    case case3():
        println("exec case3")
    default:
        println("exec default")
    }
}
</code></pre><p>执行一下这个示例程序，我们得到如下结果：</p><pre><code class="language-plain">eval switch expr
eval case1 expr
eval case2_1 expr
eval case2_2 expr
exec case2
</code></pre><p>从输出结果中我们看到，Go先对switch expr表达式进行求值，然后再按case语句的出现顺序，从上到下进行逐一求值。在带有表达式列表的case语句中，Go会从左到右，对列表中的表达式进行求值，比如示例中的case2_1函数就执行于case2_2函数之前。</p><p>如果switch表达式匹配到了某个case表达式，那么程序就会执行这个case对应的代码分支，比如示例中的“exec case2”。这个分支后面的case表达式将不会再得到求值机会，比如示例不会执行case3函数。这里要注意一点，即便后面的case表达式求值后也能与switch表达式匹配上，Go也不会继续去对这些表达式进行求值了。</p><p>除了这一点外，你还要注意default分支。<strong>无论default分支出现在什么位置，它都只会在所有case都没有匹配上的情况下才会被执行的。</strong></p><p>不知道你有没有发现，这里其实有一个优化小技巧，考虑到switch语句是按照case出现的先后顺序对case表达式进行求值的，那么如果我们将匹配成功概率高的case表达式排在前面，就会有助于提升switch语句执行效率。这点对于case后面是表达式列表的语句同样有效，我们可以将匹配概率最高的表达式放在表达式列表的最左侧。</p><p>到这里，我们已经了解了switch语句的一般形式以及执行次序。有了这个基础后，接下来我们就来看看这节课重点：Go语言的switch语句和它的“先祖”C语言中的Switch语句相比，都有哪些优化与创新？</p><h2>switch语句的灵活性</h2><p>为方便对比，我们先来简单了解一下C语言中的switch语句。C语言中的switch语句对表达式类型有限制，每个case语句只可以有一个表达式。而且，除非你显式使用break跳出，程序默认总是执行下一个case语句。这些特性开发人员带来了使用上的心智负担。</p><p>相较于C语言中switch语句的“死板”，Go的switch语句表现出极大的灵活性，主要表现在如下几方面：</p><p><strong>首先，switch语句各表达式的求值结果可以为各种类型值，只要它的类型支持比较操作就可以了。</strong></p><p>C语言中，switch语句中使用的所有表达式的求值结果只能是int或枚举类型，其他类型都会被C编译器拒绝。</p><p>Go语言就宽容得多了，只要类型支持比较操作，都可以作为switch语句中的表达式类型。比如整型、布尔类型、字符串类型、复数类型、元素类型都是可比较类型的数组类型，甚至字段类型都是可比较类型的结构体类型，也可以。下面就是一个使用自定义结构体类型作为switch表达式类型的例子：</p><pre><code class="language-plain">type person struct {
    name string
    age  int
}

func main() {
    p := person{"tom", 13}
    switch p {
    case person{"tony", 33}:
        println("match tony")
    case person{"tom", 13}:
        println("match tom")
    case person{"lucy", 23}:
        println("match lucy")
    default:
        println("no match")
    }
}
</code></pre><p>不过，实际开发过程中，以结构体类型为switch表达式类型的情况并不常见，这里举这个例子仅是为了说明Go switch语句对各种类型支持的广泛性。</p><p>而且，当switch表达式的类型为布尔类型时，如果求值结果始终为true，那么我们甚至可以省略switch后面的表达式，比如下面例子：</p><pre><code class="language-plain">// 带有initStmt语句的switch语句
switch initStmt; {
    case bool_expr1:
    case bool_expr2:
    ... ...
}

// 没有initStmt语句的switch语句
switch {
    case bool_expr1:
    case bool_expr2:
    ... ...
}
</code></pre><p>不过，这里要注意，在带有initStmt的情况下，如果我们省略switch表达式，那么initStmt后面的分号不能省略，因为initStmt是一个语句。</p><p><strong>第二点：switch语句支持声明临时变量。</strong></p><p>在前面介绍switch语句的一般形式中，我们看到，和if、for等控制结构语句一样，switch语句的initStmt可用来声明只在这个switch隐式代码块中使用的变量，这种就近声明的变量最大程度地缩小了变量的作用域。</p><p><strong>第三点：case语句支持表达式列表。</strong></p><p>在C语言中，如果要让多个case分支的执行相同的代码逻辑，我们只能通过下面的方式实现：</p><pre><code class="language-plain">void check_work_day(int a) {
    switch(a) {
        case 1:
        case 2:
        case 3:
        case 4:
        case 5:
            printf("it is a work day\n");
            break;
        case 6:
        case 7:
            printf("it is a weekend day\n");
            break;
        default:
            printf("do you live on earth?\n");
    }
}
</code></pre><p>在上面这段C代码中，case 1~case 5匹配成功后，执行的都是case 5中的代码逻辑，case 6~case 7匹配成功后，执行的都是case 7中的代码逻辑。</p><p>之所以可以实现这样的逻辑，是因为当C语言中的switch语句匹配到某个case后，如果这个case对应的代码逻辑中没有break语句，那么代码将继续执行下一个case。比如当a = 3时，case 3后面的代码为空逻辑，并且没有break语句，那么C会继续向下执行case4、case5，直到在case 5中调用了break，代码执行流才离开switch语句。</p><p>这样看，虽然C也能实现多case语句执行同一逻辑的功能，但在case分支较多的情况下，代码会显得十分冗长。</p><p>Go语言中的处理要好得多。Go语言中，switch语句在case中支持表达式列表。我们可以用表达式列表实现与上面的示例相同的处理逻辑：</p><pre><code class="language-plain">func checkWorkday(a int) {
    switch a {
    case 1, 2, 3, 4, 5:
        println("it is a work day")
    case 6, 7:
        println("it is a weekend day")
    default:
        println("are you live on earth")
    }
}
</code></pre><p>根据前面我们讲过的switch语句的执行次序，理解上面这个例子应该不难。和C语言实现相比，使用case表达式列表的Go实现简单、清晰、易懂。</p><p><strong>第四点：取消了默认执行下一个case代码逻辑的语义。</strong></p><p>在前面的描述和check_work_day这个C代码示例中，你都能感受到，在C语言中，如果匹配到的case对应的代码分支中没有显式调用break语句，那么代码将继续执行下一个case的代码分支，这种“隐式语义”并不符合日常算法的常规逻辑，这也经常被诟病为C语言的一个缺陷。要修复这个缺陷，我们只能在每个case执行语句中都显式调用break。</p><p>Go语言中的Swith语句就修复了C语言的这个缺陷，取消了默认执行下一个case代码逻辑的“非常规”语义，每个case对应的分支代码执行完后就结束switch语句。</p><p>如果在少数场景下，你需要执行下一个case的代码逻辑，你可以显式使用Go提供的关键字fallthrough来实现，这也是Go“显式”设计哲学的一个体现。下面就是一个使用fallthrough的switch语句的例子，我们简单来看一下：</p><pre><code class="language-plain">func case1() int {
    println("eval case1 expr")
    return 1
}

func case2() int {
    println("eval case2 expr")
    return 2
}

func switchexpr() int {
    println("eval switch expr")
    return 1
}

func main() {
    switch switchexpr() {
    case case1():
        println("exec case1")
        fallthrough
    case case2():
        println("exec case2")
        fallthrough
    default:
        println("exec default")
    }
}
</code></pre><p>执行一下这个示例程序，我们得到这样的结果：</p><pre><code class="language-plain">eval switch expr
eval case1 expr
exec case1
exec case2
exec default
</code></pre><p>我们看到，switch expr的求值结果与case1匹配成功，Go执行了case1对应的代码分支。而且，由于case1代码分支中显式使用了fallthrough，执行完case1后，代码执行流并没有离开switch语句，而是继续执行下一个case，也就是case2的代码分支。</p><p>这里有一个注意点，由于fallthrough的存在，Go不会对case2的表达式做求值操作，而会直接执行case2对应的代码分支。而且，在这里case2中的代码分支也显式使用了fallthrough，于是最后一个代码分支，也就是default分支对应的代码也被执行了。</p><p>另外，还有一点要注意的是，如果某个case语句已经是switch语句中的最后一个case了，并且它的后面也没有default分支了，那么这个case中就不能再使用fallthrough，否则编译器就会报错。</p><p>到这里，我们看到Go的switch语句不仅修复了C语言switch的缺陷，还为Go开发人员提供了更大的灵活性，我们可以使用更多类型表达式作为switch表达式类型，也可以使用case表达式列表简化实现逻辑，还可以自行根据需要，确定是否使用fallthrough关键字继续向下执行下一个case的代码分支。</p><p>除了这些之外，Go语言的switch语句还支持求值结果为类型信息的表达式，也就是type switch语句，接下来我们就详细分析一下。</p><h2>type switch</h2><p>“type switch”这是一种特殊的switch语句用法，我们通过一个例子来看一下它具体的使用形式：</p><pre><code class="language-plain">func main() {
    var x interface{} = 13
    switch x.(type) {
    case nil:
        println("x is nil")
    case int:
        println("the type of x is int")
    case string:
        println("the type of x is string")
    case bool:
        println("the type of x is string")
    default:
        println("don't support the type")
    }
}
</code></pre><p>我们看到，这个例子中switch语句的形式与前面是一致的，不同的是switch与case两个关键字后面跟着的表达式。</p><p>switch关键字后面跟着的表达式为<code>x.(type)</code>，这种表达式形式是switch语句专有的，而且也只能在switch语句中使用。这个表达式中的<strong>x必须是一个接口类型变量</strong>，表达式的求值结果是这个接口类型变量对应的动态类型。</p><p>什么是一个接口类型的动态类型呢？我们简单解释一下。以上面的代码<code>var x interface{} = 13</code>为例，x是一个接口类型变量，它的静态类型为<code>interface{}</code>，如果我们将整型值13赋值给x，x这个接口变量的动态类型就为int。关于接口类型变量的动态类型，我们后面还会详细讲，这里先简单了解一下就可以了。</p><p>接着，case关键字后面接的就不是普通意义上的表达式了，而是一个个具体的类型。这样，Go就能使用变量x的动态类型与各个case中的类型进行匹配，之后的逻辑就都是一样的了。</p><p>现在我们运行上面示例程序，输出了x的动态变量类型：</p><pre><code class="language-plain">the type of x is int
</code></pre><p>不过，通过<code>x.(type)</code>，我们除了可以获得变量x的动态类型信息之外，也能获得其动态类型对应的值信息，现在我们把上面的例子改造一下：</p><pre><code class="language-plain">func main() {
    var x interface{} = 13
    switch v := x.(type) {
    case nil:
        println("v is nil")
    case int:
        println("the type of v is int, v =", v)
    case string:
        println("the type of v is string, v =", v)
    case bool:
        println("the type of v is bool, v =", v)
    default:
        println("don't support the type")
    }
}
</code></pre><p>这里我们将switch后面的表达式由<code>x.(type)</code>换成了<code>v := x.(type)</code>。对于后者，你千万不要认为变量v存储的是类型信息，其实<strong>v存储的是变量x的动态类型对应的值信息</strong>，这样我们在接下来的case执行路径中就可以使用变量v中的值信息了。</p><p>然后，我们运行上面示例，可以得到v的动态类型和值：</p><pre><code class="language-plain">the type of v is int, v = 13
</code></pre><p>另外，你可以发现，在前面的type switch演示示例中，我们一直使用interface{}这种接口类型的变量，Go中所有类型都实现了interface{}类型，所以case后面可以是任意类型信息。</p><p>但如果在switch后面使用了某个特定的接口类型I，那么case后面就只能使用实现了接口类型I的类型了，否则Go编译器会报错。你可以看看这个例子：</p><pre><code class="language-plain">  type I interface {
      M()
  }
  
  type T struct {
  }
  
 func (T) M() {
 }
 
 func main() {
     var t T
     var i I = t
     switch i.(type) {
     case T:
         println("it is type T")
     case int:
         println("it is type int")
     case string:
         println("it is type string")
     }
 }
</code></pre><p>在这个例子中，我们在type switch中使用了自定义的接口类型I。那么，理论上所有case后面的类型都只能是实现了接口I的类型。但在这段代码中，只有类型T实现了接口类型I，Go原生类型int与string都没有实现接口I，于是在编译上述代码时，编译器会报出如下错误信息：</p><pre><code class="language-plain">19:2: impossible type switch case: i (type I) cannot have dynamic type int (missing M method)
21:2: impossible type switch case: i (type I) cannot have dynamic type string (missing M method)
</code></pre><p>好了，到这里，关于switch语句语法层面的知识就都学习完了。Go对switch语句的优化与增强使得我们在日常使用switch时很少遇到坑，但这也并不意味着没有，最后我们就来看在Go编码过程中，我们可能遇到的一个与switch使用有关的问题，跳不出循环的break。</p><h2>跳不出循环的break</h2><p>在上一节课讲解break语句的时候，我们曾举了一个找出整型切片中第一个偶数的例子，当时我们是把for与if语句结合起来实现的。现在，我们把那个例子中if分支结构换成这节课学习的switch分支结构试试看。我们这里直接看改造后的代码：</p><pre><code class="language-plain">func main() {
    var sl = []int{5, 19, 6, 3, 8, 12}
    var firstEven int = -1

    // find first even number of the interger slice
    for i := 0; i &lt; len(sl); i++ {
        switch sl[i] % 2 {
        case 0:
            firstEven = sl[i]
            break
        case 1:
            // do nothing
        }        
    }         
    println(firstEven) 
}
</code></pre><p>我们运行一下这个修改后的程序，得到结果为12。</p><p>奇怪，这个输出的值与我们的预期的好像不太一样。这段代码中，切片中的第一个偶数是6，而输出的结果却成了切片的最后一个偶数12。为什么会出现这种结果呢？</p><p>这就是Go中 break语句与switch分支结合使用会出现一个“小坑”。和我们习惯的C家族语言中的break不同，Go语言规范中明确规定，<strong>不带label的break语句中断执行并跳出的，是同一函数内break语句所在的最内层的for、switch或select</strong>。所以，上面这个例子的break语句实际上只跳出了switch语句，并没有跳出外层的for循环，这也就是程序未按我们预期执行的原因。</p><p>要修正这一问题，我们可以利用上节课学到的带label的break语句试试。这里我们也直接看看改进后的代码:</p><pre><code class="language-plain">func main() {
    var sl = []int{5, 19, 6, 3, 8, 12}
    var firstEven int = -1

    // find first even number of the interger slice
loop:
    for i := 0; i &lt; len(sl); i++ {
        switch sl[i] % 2 {
        case 0:
            firstEven = sl[i]
            break loop
        case 1:
            // do nothing
        }
    }
    println(firstEven) // 6
}
</code></pre><p>在改进后的例子中，我们定义了一个label：loop，这个label附在for循环的外面，指代for循环的执行。当代码执行到“break loop”时，程序将停止label loop所指代的for循环的执行。关于带有label的break语句，你可以再回顾一下第19讲，这里就不多说了。</p><p>和switch语句一样能阻拦break跳出的还有一个语句，那就是select，我们后面讲解并发程序设计的时候再来详细分析。</p><h2>小结</h2><p>好了，今天的课讲到这里就结束了，现在我们一起来回顾一下吧。</p><p>在这一讲中，我们讲解了Go语言提供的另一种分支控制结构：switch语句。和if分支语句相比，在一些执行分支较多的场景下，使用switch分支控制语句可以让代码更简洁、可读性更好。</p><p>Go语言的switch语句继承自C语言，但“青出于蓝而胜于蓝”，Go不但修正了C语言中switch语句默认执行下一个case的“坑”，还对switch语句进行了改进与创新，包括支持更多类型、支持表达式列表等，让switch的表达力得到进一步提升。</p><p>除了使用常规表达式作为switch表达式和case表达式之外，Go switch语句又创新性地支持type switch，也就是用类型信息作为分支条件判断的操作数。在Go中，这种使用方式也是switch所独有的。这里，我们要注意的是只有接口类型变量才能使用type switch，并且所有case语句中的类型必须实现switch关键字后面变量的接口类型。</p><p>最后还需要你记住的是switch会阻拦break语句跳出for循环，就像我们这节课最后那个例子中那样，对于初学者来说，这是一个很容易掉下去的坑，你一定不要走弯路。</p><h2>思考题</h2><p>为了验证在多分支下基于switch语句实现的分支控制更为简洁，你可以尝试将这节课中的那些稍复杂一点的例子，改写为基于if条件分支的实现，然后再对比一下两种实现的复杂性，直观体会一下switch语句的优点。</p><p>欢迎你把这节课分享给更多对Go语言中的switch语句感兴趣的朋友。我是Tony Bai，我们下节课见。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/70/67/0c1359c2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>qinsi</span>
  </div>
  <div class="_2_QraFYR_0">type switch里的<br><br>switch v := x.(type) {<br><br>看上去像是一个初始化语句，但其实是一个type guard，所以后面没有分号。如果有初始化语句的话就是这样的：<br><br>switch a:= f(); v := x.(type) {<br><br>另外type switch里是不能fallthrough的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-04 00:09:44</div>
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
  <div class="_2_QraFYR_0">今天《极客时间》两个专栏同时更新，主题都是 switch<br><br>- 《深入剖析 Java 新特性》06 | Switch表达式：怎么简化多情景操作？<br>- 《Tony Bai · Go 语言第一课》20｜控制结构：Go中的switch语句有哪些变化？<br><br>对比结果<br><br>- Java17 居然可以比 Go 简洁！<br>- 但然综合能力 Go 的更灵活<br><br><br>Java17 switch<br><br>```java<br>String checkWorkday(int day) {<br>	return switch (day) {<br>		case 1, 2, 3, 4, 5 -&gt; &quot;it is a work day&quot;;<br>		case 6, 7 -&gt; &quot;it is a weekend day&quot;;<br>		default -&gt; &quot;are you live on earth&quot;;<br>	};<br>}<br>```<br><br>Go switch<br><br>```go<br>func checkWorkday(day int) string {<br>	switch day {<br>	case 1, 2, 3, 4, 5:<br>		return &quot;it is a work day&quot;<br>	case 6, 7:<br>		return &quot;it is a weekend day&quot;<br>	default:<br>		return &quot;are you live on earth&quot;<br>	}<br>}<br>```</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-26 21:17:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83eo5vic8QksE4b8ricXxKrEWJyOX9pwiadhk3kvHYoLXoKRTWvbFCxibFTbExNQWDG4nvNfpic9t1umibKww/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>江楠大盗</span>
  </div>
  <div class="_2_QraFYR_0">文中说x.(type)这种表达式形式是 switch 语句专有的，但是类型断言也可以这么写，所以不应该是专有的吧？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 只有switch表达式可以用x.(type)，类型断言的格式是类似像x.(*int)的形式，类型断言后面括号里必须是某个具体的类型。而switch表达式的x.(type)就是x.(type)。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-12 13:51:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/24/58/45/9ba77dc3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lyy</span>
  </div>
  <div class="_2_QraFYR_0">个人感觉fallthrough，执行完 case1 后，继续case2里面的代码，而不用判断case2的条件是否成立，这一点设计的并不好，估计很多人会理解为继续判断case2条件</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 但是如果按你的设计，fallthrough后，先要判断下一个case的条件，那么fallthrough的意义就不存在了，因为下一个case的条件求值后基本不会是true。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-06 09:18:42</div>
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
  <div class="_2_QraFYR_0">所以，switch 不需要 break 是出于大多数情况 switch 只需要走一条分支的缘故吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-26 21:19:05</div>
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
  <div class="_2_QraFYR_0">Tony Bai 老师的文章讲解的非常细致，鼓掌。<br><br>想问一下老师，文中的内容基本都能理解，但是过一段时间就遗忘比较多了，尤其是后面的内容涉及到前面的知识时。希望老师在这门课中搞个小型的实战项目，能把前面的知识串在一起就好了。<br><br>这样，不会觉得纸上得来终觉浅......</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 后续会有实战项目，尽量串联吧。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-26 22:19:39</div>
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
  <div class="_2_QraFYR_0">白老师，我这样理解对不对：<br>x.(type) 如果没有 := 符号的话，这个表达式是获取x的具体类型<br>v := x.(type), 这个则是把x从具体的接口类型中获取它实际类型的值。<br>x.(SomeType)， 则是判段x是否遵守SomeType接口，并转化为具体类型的值。<br>所有的这些行为都是编译器把它转化为相应的代码。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 差不多。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-24 16:19:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/0a/da/dcf8f2b1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>qiutian</span>
  </div>
  <div class="_2_QraFYR_0">哪来的case2呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 在这个case中：<br>case case2_1(), case2_2():        <br>      println(&quot;exec case2&quot;)</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-01 22:20:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/70/6d/11ea66f0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>peison</span>
  </div>
  <div class="_2_QraFYR_0">我想请教一下，文中type switch中的  v:=x.(type)后面，为什么switch中的case分支，不是和x.(type)的返回值v做比较？那实际上case分支是和什么值做比较</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: type switch是go的一种比较特殊的语法。能运用该语法的x.(type)中的x必须是interface类型。<br><br>case分支是用某类型与x的动态类型作比较。而v := x.(type)中的v存储的是转换后对应的动态类型的值。<br><br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-09 10:59:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/05/e4/3e676c4d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ps Sensking</span>
  </div>
  <div class="_2_QraFYR_0">我们要注意的是只有接口类型变量才能使用 type switch，并且所有 case 语句中的类型必须实现 switch 关键字后面变量的接口类型。 <br>您好这个例子用interface 里面只要实现一个 int  或者 string 就可以正常启动吗？ 如果是type 自定义的类型 T 就需要制定一个int 或者 string吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 问题没太看懂。文中例子switch x.(type) 中的x是一个interface{}类型接口，Go所有类型都实现了该接口，包括自定义类型T。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-22 04:16:25</div>
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
  <div class="_2_QraFYR_0">白老师，switch v := x.(type)，有点不太好理解。<br>这个语句编译器是不是转化为类型这样的代码：<br>v := x.(type)<br>switch x.(type)<br><br>我直观上会理解成：<br>v := x.(type)<br>switch v<br><br>这算不算编译器提供的一种语法糖？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: switch的不是v。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-24 16:04:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/29/44/b5/7eba5a0e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>木木</span>
  </div>
  <div class="_2_QraFYR_0">C语言的switch是为了模拟跳转表，所以如果目的是根据值执行一小段的话，需要每个条件的执行语句最后都加break，go的switch已经不再是为了模拟跳转表了，就是按着人们常用的方法设计的，所以不用加break，但是break的作用依旧留着</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-14 21:08:32</div>
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
  <div class="_2_QraFYR_0">讲的非常详细，值得好好学习</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-26 11:54:30</div>
  </div>
</div>
</div>
</li>
</ul>