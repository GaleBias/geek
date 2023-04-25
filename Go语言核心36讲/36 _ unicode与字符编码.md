<audio title="36 _ unicode与字符编码" src="https://static001.geekbang.org/resource/audio/b1/6b/b14d7b224de2a5924a4cb5aa253e246b.mp3" controls="controls"></audio> 
<p>到目前为止，我们已经一起陆陆续续地学完了Go语言中那些最重要也最有特色的概念、语法和编程方式。我对于它们非常喜爱，简直可以用如数家珍来形容了。</p><p>在开始今天的内容之前，我先来做一个简单的总结。</p><h2>Go语言经典知识总结</h2><p>基于混合线程的并发编程模型自然不必多说。</p><p>在<strong>数据类型</strong>方面有：</p><ul>
<li>基于底层数组的切片；</li>
<li>用来传递数据的通道；</li>
<li>作为一等类型的函数；</li>
<li>可实现面向对象的结构体；</li>
<li>能无侵入实现的接口等。</li>
</ul><p>在<strong>语法</strong>方面有：</p><ul>
<li>异步编程神器<code>go</code>语句；</li>
<li>函数的最后关卡<code>defer</code>语句；</li>
<li>可做类型判断的<code>switch</code>语句；</li>
<li>多通道操作利器<code>select</code>语句；</li>
<li>非常有特色的异常处理函数<code>panic</code>和<code>recover</code>。</li>
</ul><p>除了这些，我们还一起讨论了<strong>测试Go程序</strong>的主要方式。这涉及了Go语言自带的程序测试套件，相关的概念和工具包括：</p><ul>
<li>独立的测试源码文件；</li>
<li>三种功用不同的测试函数；</li>
<li>专用的<code>testing</code>代码包；</li>
<li>功能强大的<code>go test</code>命令。</li>
</ul><p>另外，就在前不久，我还为你深入讲解了Go语言提供的那些<strong>同步工具</strong>。它们也是Go语言并发编程工具箱中不可或缺的一部分。这包括了：</p><ul>
<li>经典的互斥锁；</li>
<li>读写锁；</li>
<li>条件变量；</li>
<li>原子操作。</li>
</ul><p>以及<strong>Go语言特有的一些数据类型</strong>，即：</p><ul>
<li>单次执行小助手<code>sync.Once</code>；</li>
<li>临时对象池<code>sync.Pool</code>；</li>
<li>帮助我们实现多goroutine协作流程的<code>sync.WaitGroup</code>、<code>context.Context</code>；</li>
<li>一种高效的并发安全字典<code>sync.Map</code>。</li>
</ul><!-- [[[read_end]]] --><p>毫不夸张地说，如果你真正地掌握了上述这些知识，那么就已经获得了Go语言编程的精髓。</p><p>在这之后，你再去研读Go语言标准库和那些优秀第三方库中的代码的时候，就一定会事半功倍。同时，在使用Go语言编写软件的时候，你肯定也会如鱼得水、游刃有余的。</p><p>我用了大量的篇幅讲解了Go语言中最核心的知识点，真心希望你已经搞懂了这些内容。</p><p><strong>在后面的日子里，我会与你一起去探究Go语言标准库中最常用的那些代码包，弄清它们的用法、了解它们的机理。当然了，我还会顺便讲一讲那些必备的周边知识。</strong></p><h2>前导内容1：Go语言字符编码基础</h2><p>首先，让我们来关注字符编码方面的问题。这应该是在计算机软件领域中非常基础的一个问题了。</p><p>我在前面说过，Go语言中的标识符可以包含“任何Unicode编码可以表示的字母字符”。我还说过，虽然我们可以直接把一个整数值转换为一个<code>string</code>类型的值。</p><p>但是，被转换的整数值应该可以代表一个有效的Unicode代码点，否则转换的结果就将会是<code>"�"</code>，即：一个仅由高亮的问号组成的字符串值。</p><p>另外，当一个<code>string</code>类型的值被转换为<code>[]rune</code>类型值的时候，其中的字符串会被拆分成一个一个的Unicode字符。</p><p>显然，Go语言采用的字符编码方案从属于Unicode编码规范。更确切地说，Go语言的代码正是由Unicode字符组成的。Go语言的所有源代码，都必须按照Unicode编码规范中的UTF-8编码格式进行编码。</p><p>换句话说，Go语言的源码文件必须使用UTF-8编码格式进行存储。如果源码文件中出现了非UTF-8编码的字符，那么在构建、安装以及运行的时候，go命令就会报告错误“illegal UTF-8 encoding”。</p><p>在这里，我们首先要对Unicode编码规范有所了解。不过，在讲述它之前，我先来简要地介绍一下ASCII编码。</p><h3>前导内容 2： ASCII编码</h3><p>ASCII是英文“American Standard Code for Information Interchange”的缩写，中文译为美国信息交换标准代码。它是由美国国家标准学会（ANSI）制定的单字节字符编码方案，可用于基于文本的数据交换。</p><p>它最初是美国的国家标准，后又被国际标准化组织（ISO）定为国际标准，称为ISO 646标准，并适用于所有的拉丁文字字母。</p><p>ASCII编码方案使用单个字节（byte）的二进制数来编码一个字符。标准的ASCII编码用一个字节的最高比特（bit）位作为奇偶校验位，而扩展的ASCII编码则将此位也用于表示字符。ASCII编码支持的可打印字符和控制字符的集合也被叫做ASCII编码集。</p><p>我们所说的Unicode编码规范，实际上是另一个更加通用的、针对书面字符和文本的字符编码标准。它为世界上现存的所有自然语言中的每一个字符，都设定了一个唯一的二进制编码。</p><p>它定义了不同自然语言的文本数据在国际间交换的统一方式，并为全球化软件创建了一个重要的基础。</p><p>Unicode编码规范以ASCII编码集为出发点，并突破了ASCII只能对拉丁字母进行编码的限制。它不但提供了可以对世界上超过百万的字符进行编码的能力，还支持所有已知的转义序列和控制代码。</p><p>我们都知道，在计算机系统的内部，抽象的字符会被编码为整数。这些整数的范围被称为代码空间。在代码空间之内，每一个特定的整数都被称为一个代码点。</p><p>一个受支持的抽象字符会被映射并分配给某个特定的代码点，反过来讲，一个代码点总是可以被看成一个被编码的字符。</p><p>Unicode编码规范通常使用十六进制表示法来表示Unicode代码点的整数值，并使用“U+”作为前缀。比如，英文字母字符“a”的Unicode代码点是U+0061。在Unicode编码规范中，一个字符能且只能由与它对应的那个代码点表示。</p><p>Unicode编码规范现在的最新版本是11.0，并会于2019年3月发布12.0版本。而Go语言从1.10版本开始，已经对Unicode的10.0版本提供了全面的支持。对于绝大多数的应用场景来说，这已经完全够用了。</p><p>Unicode编码规范提供了三种不同的编码格式，即：UTF-8、UTF-16和UTF-32。其中的UTF是UCS Transformation Format的缩写。而UCS又是Universal Character Set的缩写，但也可以代表Unicode Character Set。所以，UTF也可以被翻译为Unicode转换格式。它代表的是字符与字节序列之间的转换方式。</p><p>在这几种编码格式的名称中，“-”右边的整数的含义是，以多少个比特位作为一个编码单元。以UTF-8为例，它会以8个比特，也就是一个字节，作为一个编码单元。并且，它与标准的ASCII编码是完全兼容的。也就是说，在[0x00, 0x7F]的范围内，这两种编码表示的字符都是相同的。这也是UTF-8编码格式的一个巨大优势。</p><p>UTF-8是一种可变宽的编码方案。换句话说，它会用一个或多个字节的二进制数来表示某个字符，最多使用四个字节。比如，对于一个英文字符，它仅用一个字节的二进制数就可以表示，而对于一个中文字符，它需要使用三个字节才能够表示。不论怎样，一个受支持的字符总是可以由UTF-8编码为一个字节序列。以下会简称后者为UTF-8编码值。</p><p>现在，在你初步地了解了这些知识之后，请认真地思考并回答下面的问题。别担心，我会在后面进一步阐述Unicode、UTF-8以及Go语言对它们的运用。</p><p><strong>问题：一个<code>string</code>类型的值在底层是怎样被表达的？</strong></p><p><strong>典型回答</strong> 是在底层，一个<code>string</code>类型的值是由一系列相对应的Unicode代码点的UTF-8编码值来表达的。</p><h2>问题解析</h2><p>在Go语言中，一个<code>string</code>类型的值既可以被拆分为一个包含多个字符的序列，也可以被拆分为一个包含多个字节的序列。</p><p>前者可以由一个以<code>rune</code>为元素类型的切片来表示，而后者则可以由一个以<code>byte</code>为元素类型的切片代表。</p><p><code>rune</code>是Go语言特有的一个基本数据类型，它的一个值就代表一个字符，即：一个Unicode字符。</p><p>比如，<code>'G'</code>、<code>'o'</code>、<code>'爱'</code>、<code>'好'</code>、<code>'者'</code>代表的就都是一个Unicode字符。</p><p>我们已经知道，UTF-8编码方案会把一个Unicode字符编码为一个长度在[1, 4]范围内的字节序列。所以，一个<code>rune</code>类型的值也可以由一个或多个字节来代表。</p><pre><code>type rune = int32
</code></pre><p>根据<code>rune</code>类型的声明可知，它实际上就是<code>int32</code>类型的一个别名类型。也就是说，一个<code>rune</code>类型的值会由四个字节宽度的空间来存储。它的存储空间总是能够存下一个UTF-8编码值。</p><p>一个<code>rune</code>类型的值在底层其实就是一个UTF-8编码值。前者是（便于我们人类理解的）外部展现，后者是（便于计算机系统理解的）内在表达。</p><p>请看下面的代码：</p><pre><code>str := &quot;Go爱好者&quot;
fmt.Printf(&quot;The string: %q\n&quot;, str)
fmt.Printf(&quot;  =&gt; runes(char): %q\n&quot;, []rune(str))
fmt.Printf(&quot;  =&gt; runes(hex): %x\n&quot;, []rune(str))
fmt.Printf(&quot;  =&gt; bytes(hex): [% x]\n&quot;, []byte(str))
</code></pre><p>字符串值<code>"Go爱好者"</code>如果被转换为<code>[]rune</code>类型的值的话，其中的每一个字符（不论是英文字符还是中文字符）就都会独立成为一个<code>rune</code>类型的元素值。因此，这段代码打印出的第二行内容就会如下所示：</p><pre><code>  =&gt; runes(char): ['G' 'o' '爱' '好' '者']
</code></pre><p>又由于，每个<code>rune</code>类型的值在底层都是由一个UTF-8编码值来表达的，所以我们可以换一种方式来展现这个字符序列：</p><pre><code>  =&gt; runes(hex): [47 6f 7231 597d 8005]
</code></pre><p>可以看到，五个十六进制数与五个字符相对应。很明显，前两个十六进制数<code>47</code>和<code>6f</code>代表的整数都比较小，它们分别表示字符<code>'G'</code>和<code>'o'</code>。</p><p>因为它们都是英文字符，所以对应的UTF-8编码值用一个字节表达就足够了。一个字节的编码值被转换为整数之后，不会大到哪里去。</p><p>而后三个十六进制数<code>7231</code>、<code>597d</code>和<code>8005</code>都相对较大，它们分别表示中文字符<code>'爱'</code>、<code>'好'</code>和<code>'者'</code>。</p><p>这些中文字符对应的UTF-8编码值，都需要使用三个字节来表达。所以，这三个数就是把对应的三个字节的编码值，转换为整数后得到的结果。</p><p>我们还可以进一步地拆分，把每个字符的UTF-8编码值都拆成相应的字节序列。上述代码中的第五行就是这么做的。它会得到如下的输出：</p><pre><code>  =&gt; bytes(hex): [47 6f e7 88 b1 e5 a5 bd e8 80 85]
</code></pre><p>这里得到的字节切片比前面的字符切片明显长了很多。这正是因为一个中文字符的UTF-8编码值需要用三个字节来表达。</p><p>这个字节切片的前两个元素值与字符切片的前两个元素值是一致的，而在这之后，前者的每三个元素值才对应字符切片中的一个元素值。</p><p>注意，对于一个多字节的UTF-8编码值来说，我们可以把它当做一个整体转换为单一的整数，也可以先把它拆成字节序列，再把每个字节分别转换为一个整数，从而得到多个整数。</p><p>这两种表示法展现出来的内容往往会很不一样。比如，对于中文字符<code>'爱'</code>来说，它的UTF-8编码值可以展现为单一的整数<code>7231</code>，也可以展现为三个整数，即：<code>e7</code>、<code>88</code>和<code>b1</code>。</p><p><img src="https://static001.geekbang.org/resource/image/0d/85/0d8dac40ccb2972dbceef33d03741085.png?wh=1129*604" alt=""><br>
（字符串值的底层表示）</p><p>总之，一个<code>string</code>类型的值会由若干个Unicode字符组成，每个Unicode字符都可以由一个<code>rune</code>类型的值来承载。</p><p>这些字符在底层都会被转换为UTF-8编码值，而这些UTF-8编码值又会以字节序列的形式表达和存储。因此，一个<code>string</code>类型的值在底层就是一个能够表达若干个UTF-8编码值的字节序列。</p><h2>知识扩展</h2><p><strong>问题 1：使用带有<code>range</code>子句的<code>for</code>语句遍历字符串值的时候应该注意什么？</strong></p><p>带有<code>range</code>子句的<code>for</code>语句会先把被遍历的字符串值拆成一个字节序列，然后再试图找出这个字节序列中包含的每一个UTF-8编码值，或者说每一个Unicode字符。</p><p>这样的<code>for</code>语句可以为两个迭代变量赋值。如果存在两个迭代变量，那么赋给第一个变量的值，就将会是当前字节序列中的某个UTF-8编码值的第一个字节所对应的那个索引值。</p><p>而赋给第二个变量的值，则是这个UTF-8编码值代表的那个Unicode字符，其类型会是<code>rune</code>。</p><p>例如，有这么几行代码：</p><pre><code>str := &quot;Go爱好者&quot;
for i, c := range str {
 fmt.Printf(&quot;%d: %q [% x]\n&quot;, i, c, []byte(string(c)))
}
</code></pre><p>这里被遍历的字符串值是<code>"Go爱好者"</code>。在每次迭代的时候，这段代码都会打印出两个迭代变量的值，以及第二个值的字节序列形式。完整的打印内容如下：</p><pre><code>0: 'G' [47]
1: 'o' [6f]
2: '爱' [e7 88 b1]
5: '好' [e5 a5 bd]
8: '者' [e8 80 85]
</code></pre><p>第一行内容中的关键信息有<code>0</code>、<code>'G'</code>和<code>[47]</code>。这是由于这个字符串值中的第一个Unicode字符是<code>'G'</code>。该字符是一个单字节字符，并且由相应的字节序列中的第一个字节表达。这个字节的十六进制表示为<code>47</code>。</p><p>第二行展示的内容与之类似，即：第二个Unicode字符是<code>'o'</code>，由字节序列中的第二个字节表达，其十六进制表示为<code>6f</code>。</p><p>再往下看，第三行展示的是<code>'爱'</code>，也是第三个Unicode字符。因为它是一个中文字符，所以由字节序列中的第三、四、五个字节共同表达，其十六进制表示也不再是单一的整数，而是<code>e7</code>、<code>88</code>和<code>b1</code>组成的序列。</p><p>下面要注意了，正是因为<code>'爱'</code>是由三个字节共同表达的，所以第四个Unicode字符<code>'好'</code>对应的索引值并不是<code>3</code>，而是<code>2</code>加<code>3</code>后得到的<code>5</code>。</p><p>这里的<code>2</code>代表的是<code>'爱'</code>对应的索引值，而<code>3</code>代表的则是<code>'爱'</code>对应的UTF-8编码值的宽度。对于这个字符串值中的最后一个字符<code>'者'</code>来说也是类似的，因此，它对应的索引值是<code>8</code>。</p><p>由此可以看出，这样的<code>for</code>语句可以逐一地迭代出字符串值里的每个Unicode字符。但是，相邻的Unicode字符的索引值并不一定是连续的。这取决于前一个Unicode字符是否为单字节字符。</p><p>正因为如此，如果我们想得到其中某个Unicode字符对应的UTF-8编码值的宽度，就可以用下一个字符的索引值减去当前字符的索引值。</p><p>初学者可能会对<code>for</code>语句的这种行为感到困惑，因为它给予两个迭代变量的值看起来并不总是对应的。不过，一旦我们了解了它的内在机制就会拨云见日、豁然开朗。</p><h2>总结</h2><p>我们今天把目光聚焦在了Unicode编码规范、UTF-8编码格式，以及Go语言对字符串和字符的相关处理方式上。</p><p>Go语言的代码是由Unicode字符组成的，它们都必须由Unicode编码规范中的UTF-8编码格式进行编码并存储，否则就会导致go命令的报错。</p><p>Unicode编码规范中的编码格式定义的是：字符与字节序列之间的转换方式。其中的UTF-8是一种可变宽的编码方案。</p><p>它会用一个或多个字节的二进制数来表示某个字符，最多使用四个字节。一个受支持的字符，总是可以由UTF-8编码为一个字节序列，后者也可以被称为UTF-8编码值。</p><p>Go语言中的一个<code>string</code>类型值会由若干个Unicode字符组成，每个Unicode字符都可以由一个<code>rune</code>类型的值来承载。</p><p>这些字符在底层都会被转换为UTF-8编码值，而这些UTF-8编码值又会以字节序列的形式表达和存储。因此，一个<code>string</code>类型的值在底层就是一个能够表达若干个UTF-8编码值的字节序列。</p><p>初学者可能会对带有<code>range</code>子句的<code>for</code>语句遍历字符串值的行为感到困惑，因为它给予两个迭代变量的值看起来并不总是对应的。但事实并非如此。</p><p>这样的<code>for</code>语句会先把被遍历的字符串值拆成一个字节序列，然后再试图找出这个字节序列中包含的每一个UTF-8编码值，或者说每一个Unicode字符。</p><p>相邻的Unicode字符的索引值并不一定是连续的。这取决于前一个Unicode字符是否为单字节字符。一旦我们清楚了这些内在机制就不会再困惑了。</p><p>对于Go语言来说，Unicode编码规范和UTF-8编码格式算是基础之一了。我们应该了解到它们对Go语言的重要性。这对于正确理解Go语言中的相关数据类型以及日后的相关程序编写都会很有好处。</p><h2>思考题</h2><p>今天的思考题是：判断一个Unicode字符是否为单字节字符通常有几种方式？</p><p><a href="https://github.com/hyper0x/Golang_Puzzlers">戳此查看Go语言专栏文章配套详细代码。</a></p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/ce/2d/ef3b42df.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>wade</span>
  </div>
  <div class="_2_QraFYR_0">而后三个十六进制数7231、597d和8005都相对较大，它们分别表示中文字符&#39;爱&#39;、&#39;好&#39;和&#39;者&#39;。这些中文字符对应的 UTF-8 编码值，都需要使用三个字节来表达。所以，这三个数就是把对应的三个字节来表达。所以，这三个数就是把对应的三个字节的编码值，转换为整数后得到的结果。<br><br>&quot;爱好者&quot;对应的7231、597d、8005，不是UTF-8编码值，是unicode码点。unicode码点和最终计算器存储用的utf-8编码值不是一样的。转换成rune的时候rune切片每个元素存储一个unicode码点，也就是string里的一个字符转成rune切片的一个元素。string是以utf-8编码存储，byte切片也就是存储用的string用utf-8编码存储后的字节序。<br><br>unicode和utf-8的关系，可以看这个文章<br>http:&#47;&#47;www.ruanyifeng.com&#47;blog&#47;2007&#47;10&#47;ascii_unicode_and_utf-8.html</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-07 11:17:14</div>
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
  <div class="_2_QraFYR_0">unicode只是给全世界的字符做了一个编码，假如世界上有100亿个字符，那么从0到100亿，有的数只占用一个字节，有的数二个有的数三个…，找到对应的编号就找到了对应的字符。unicode只给字符分配了编号，没有说怎么去存储这些字符，用1字节存不下，100字节太浪费，于是utf-8就想到了一个存储的点子，因为生活中用到的字符用4个字节应该能存下了，于是就用1到4个字节来存储这些字符，占1个字节的就用1字节存储，2字节的就用2字节存储，以此类推……，utf8要做的就是怎么去判断一个字符到底用了几个字节存储，就好比买绳子一样，有的人要两米有的人要一米，不按照尺寸剪肯定不行，具体怎么减咱们先不关心这个留给utf8去处理。存储这些编码在计算机里就是二进制。这些二进制utf8能读懂，但是计算机看来就是01没啥了不起的，8二进制放到4字节存储没毛病吧，装不满的高位大不了填充0，就是这么有钱豆浆喝一杯倒一杯，这个就是[]rune, 但是世界总有吃不饱饭的人看着闹心，那还是一个字节一个字节存吧，等需要查看字符的时候大不了再转换为rune切片，这就是[]byte</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-15 22:23:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/99/c9/a7c77746.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>冰激凌的眼泪</span>
  </div>
  <div class="_2_QraFYR_0">src文件编码是utf8<br>string是utf8编码的mb，len(string)是字节的长度<br>string可以转化为[]rune，unicode码，32bit的数字，当字符看，len([]rune)为字符长度<br>string可以转化为[]byte，utf8编码字节串，len([]byte)和len(string)是一样的<br>for range的时候，迭代出首字节下标和rune，首字符下标可能跳跃(视上一个字符编码长度定)</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-05 08:44:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/99/c9/a7c77746.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>冰激凌的眼泪</span>
  </div>
  <div class="_2_QraFYR_0">看rune大小<br>转成byte看长度<br>加个小尾巴,range看间隔<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-02 22:02:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/57/d8/425e1b0a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小虾米</span>
  </div>
  <div class="_2_QraFYR_0">这篇文章把unicode和utf8区分的不是很清楚，正如上面有个网友说的rune切变16进制输出是字符的unicode的码点，而byte切片输出的才是utf8的编码</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这么说没错，不过rune在底层也是字节串。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-16 13:27:08</div>
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
  <div class="_2_QraFYR_0">郝林老师，请问一下：“基于混合线程的并发编程模型”。这句话该怎么理解呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这里所谓的混合线程就是：OS内核外的线程&#47;协程 + OS内核里的系统线程。<br><br>它牵扯到了两个程序级别，一个是用户级，一个是内核级，所以也叫“混合”或“两级”的线程模型。从数量对应的角度讲，它也叫M:N的线程模型（M指核外线程数量，N指核内线程数量），即两者之间是动态匹配的。你想想看，goroutine和系统线程是不是动态匹配的。Go语言的并发模型中不止有混合线程，但底层是基于此的。<br><br>相对应的，Java用的是1:1的纯内核级线程模型，也就是说JVM中的一个线程就等同于内核中的一个系统线程，一一对应。<br><br>还有就是，Python的绿色线程，是M:1的纯用户级线程模型。同进程内的多个绿色线程实际上共用一个线程，它们并不是真的并发执行（共用一个线程不可能出现并发执行的情况），只是通过调度看起来像并发执行而已。<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-23 16:36:55</div>
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
  <div class="_2_QraFYR_0">+ isrunesingle.go<br><br>```go<br>package show_rune_length<br><br>func isSingleCharA(c rune) bool {<br>	return int32(c) &lt; 128<br>}<br><br>func isSingleCharB(c rune) bool {<br>	data := []byte(string(c))<br>	return len(data) == 1<br>}<br><br>func isSingleCharC(c rune) bool {<br>	data := string(c) + &quot; &quot;<br><br>	for i, _ := range data {<br>		if i == 0 {<br>			continue<br>		}<br><br>		if i == 1 {<br>			return true<br>		} else {<br>			return false<br>		}<br>	}<br><br>	return false<br>}<br>```<br><br>+ isrunesingle_test.go<br><br>```go<br>package show_rune_length<br><br>import (<br>	&quot;testing&quot;<br><br>	&quot;github.com&#47;stretchr&#47;testify&#47;assert&quot;<br>)<br><br>type CharJudger func(c rune) bool<br><br>func TestIsSingleChar(t *testing.T) {<br><br>	for _, judger := range []CharJudger{<br>		isSingleCharA,<br>		isSingleCharB,<br>		isSingleCharC,<br>	} {<br>		assert.True(t, judger(&#39;A&#39;))<br>		assert.True(t, judger(rune(&#39; &#39;)))<br>		assert.False(t, judger(&#39;😔&#39;))<br>		assert.False(t, judger(&#39;爱&#39;))<br>	}<br>}<br>```</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以用 unicode&#47;utf8 代码包中的 RuneCount 函数。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-22 23:59:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/a5/a3/f8bcd9dd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>韡WEI</span>
  </div>
  <div class="_2_QraFYR_0">rune怎么翻译？有道查的：神秘的记号。为什么这么命名这个类型？有没有什么故事？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-15 18:18:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/48/21/bd739446.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Gryllus</span>
  </div>
  <div class="_2_QraFYR_0">终于追上了进度</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 🐂</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-02 15:23:06</div>
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
  <div class="_2_QraFYR_0">一个unicode字符在内存中存的是码点的值还是utf8对应的编码值？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 内存里存的是 UTF-8 的编码值，会由 1~4 个字节组成。Unicode 代码点是 Unicode 标准中用来唯一标识字符的东西。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-18 23:55:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>qiushye</span>
  </div>
  <div class="_2_QraFYR_0">str := &quot;Go 爱好者 &quot;fmt.Printf(&quot;Th...<br><br>您在文章里举的这个例子在Go后面多加了空格，会让人误解成遍历字符串可以跳过空格，github代码里没问题。<br><br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 收到，谢谢。已经通知编辑修正了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-05 09:43:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/f7/83/7fa4bd45.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>趣学车</span>
  </div>
  <div class="_2_QraFYR_0">len(s) == 1 则为单字节</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-18 17:25:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/80/b2/2e9f442d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>文武木子</span>
  </div>
  <div class="_2_QraFYR_0">十六进制四个数字不是一共占用32位吗，一个字节不是8位嘛，这样不是占用4个字节吗？求大神解答</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你想问什么？十六进制的一位相当于二进制的四位。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-10 09:11:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/5b/66/ad35bc68.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>党</span>
  </div>
  <div class="_2_QraFYR_0">一个unicode字符点都是由两个字节存储，为什么在go语言中 type rune = int32 四个字节 而不是 type rune=16 两个字节啊</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Unicode 代码点是一个抽象的概念，并且没有规定占用的字节数，只是说以“U+XXXX”的形式来表示。“X”还可能有 5 个或 6 个。<br><br>Unicode 代码点是 Unicode 编码标准的一部分。但你要分清楚编码标准和编码格式（像 UTF-8、UTF-16 等等）的区别。前者是概念和表示法，后者是真正的落地格式。编码格式才是真正与存储占用（比如一个字符占用多少个字节）有关的东西。<br><br>rune 类型的宽度是 4 个字节，那是因为 Go 语言使用 UTF-8 编码格式。这种格式编出来的码最多占用 4 个字节。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-30 01:47:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/09/22/22c0c4fa.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>benying</span>
  </div>
  <div class="_2_QraFYR_0">打卡,讲的挺清楚的,谢谢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-13 22:25:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/9b/9d/d487c368.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>花见笑</span>
  </div>
  <div class="_2_QraFYR_0">UTF-8的编码规则:<br>1、对于单字节的字符，字节的第一位设为0，后面七位为这个字符的Unicode码。<br>因此对于英文字符，UTF-8编码和ASCII码是相同的。<br><br>2、对于n字节的字符(n&gt;1)，第一个字节的前n位都设为1，第n+1位设为0，后面字节的<br>前两位一律设为10。剩下的没有提及的二进制位，全部为这个字符的Unicode编码。<br><br>UTF-8每次传送8位数据,并且是一种可变长的编码格式<br>具体来说，是怎么的可变长呢.<br><br>分为四个区间：<br>0x0000 0000 至 0x0000 007F:0xxxxxxx<br>0x0000 0080 至 0x0000 07FF:110xxxxx 10xxxxxx<br>0x0000 0800 至 0x0000 FFFF:1110xxxx 10xxxxxx 10xxxxxx<br>0x0001 0000 至 0x0010 FFFF:11110xxx 10xxxxxx 10xxxxxx 10xxxxxx<br><br>UTF-8解码过程:<br><br>对于采用UTF-8编码的任意字符B<br><br>如果B的第一位为0，则B为ASCII码，并且B独立的表示一个字符;<br><br>如果B的前两位为1，第三位为0，则B为一个非ASCII字符，该字符由多个字节表示，<br>并且该字符由两个字节表示;<br><br>如果B的前三位为1，第四位为0，则B为一个非ASCII字符，该字符由多个字节表示，<br>并且该字符由三个字节表示;<br><br>比如汉字 “王”,它的二进制形式为: 0x0000 738B,属于第三区间,<br>0x0000 738B - 00000000 00000000 01110011 10001011,<br>第三区间的编码是 1110xxxx 10xxxxxx 10xxxxxx<br>把x都给替换,则最终&quot;王&quot;字对应的Unicode的编码是<br>11100111 10001110 10001011<br>转换成16进制 0xe7 0x8e 0x8b<br><br>如果B的前四位为1，第五位为0，则B为一个非ASCII字符，该字符由多个字节表示，<br>并且该字符由四个字节表示;</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-19 17:51:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/ZonGorryP0YHQic0YGutFumhCRvQPiamqUV0uYzY1pDVqjaGPdWKfseyJn7iaHxNZYs8rwibKlHp8k7IDYGxcyAQKQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_479239</span>
  </div>
  <div class="_2_QraFYR_0">谢谢老师，这篇很有收获</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不客气，加油！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-20 14:30:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/4b/d7/f46c6dfd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>William Ning</span>
  </div>
  <div class="_2_QraFYR_0">打卡，一直不明白字符集unicode与字符编码utf-8以及其他的编码实现方式的区别，现在有些理解，但是还是需要理一理。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你还可以看看这篇文章：https:&#47;&#47;mp.weixin.qq.com&#47;s&#47;syyKS7lztRGTu1t00YhoFA ，我的另一本书的免费章节。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-18 14:14:15</div>
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
  <div class="_2_QraFYR_0">看它的长度或者转换类型判断</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-15 15:08:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/56/8d/9b8a6837.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Louhwz</span>
  </div>
  <div class="_2_QraFYR_0">这章讲的很清楚，谢谢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-06 09:45:54</div>
  </div>
</div>
</div>
</li>
</ul>