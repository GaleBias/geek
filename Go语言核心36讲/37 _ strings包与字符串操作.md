<audio title="37 _ strings包与字符串操作" src="https://static001.geekbang.org/resource/audio/d8/46/d86e6fe25606592dd8afd5a5da601d46.mp3" controls="controls"></audio> 
<p>在上一篇文章中，我介绍了Go语言与Unicode编码规范、UTF-8编码格式的渊源及运用。</p><p>Go语言不但拥有可以独立代表Unicode字符的类型<code>rune</code>，而且还有可以对字符串值进行Unicode字符拆分的<code>for</code>语句。</p><p>除此之外，标准库中的<code>unicode</code>包及其子包还提供了很多的函数和数据类型，可以帮助我们解析各种内容中的Unicode字符。</p><p>这些程序实体都很好用，也都很简单明了，而且有效地隐藏了Unicode编码规范中的一些复杂的细节。我就不在这里对它们进行专门的讲解了。</p><p>我们今天主要来说一说标准库中的<code>strings</code>代码包。这个代码包也用到了不少<code>unicode</code>包和<code>unicode/utf8</code>包中的程序实体。</p><ul>
<li>
<p>比如，<code>strings.Builder</code>类型的<code>WriteRune</code>方法。</p>
</li>
<li>
<p>又比如，<code>strings.Reader</code>类型的<code>ReadRune</code>方法，等等。</p>
</li>
</ul><p>下面这个问题就是针对<code>strings.Builder</code>类型的。<strong>我们今天的问题是：与<code>string</code>值相比，<code>strings.Builder</code>类型的值有哪些优势？</strong></p><p>这里的<strong>典型回答</strong>是这样的。</p><p><code>strings.Builder</code>类型的值（以下简称<code>Builder</code>值）的优势有下面的三种：</p><ul>
<li>已存在的内容不可变，但可以拼接更多的内容；</li>
<li>减少了内存分配和内容拷贝的次数；</li>
<li>可将内容重置，可重用值。</li>
</ul><!-- [[[read_end]]] --><h2>问题解析</h2><p><strong>先来说说<code>string</code>类型。</strong> 我们都知道，在Go语言中，<code>string</code>类型的值是不可变的。 如果我们想获得一个不一样的字符串，那么就只能基于原字符串进行裁剪、拼接等操作，从而生成一个新的字符串。</p><ul>
<li>裁剪操作可以使用切片表达式；</li>
<li>拼接操作可以用操作符<code>+</code>实现。</li>
</ul><p>在底层，一个<code>string</code>值的内容会被存储到一块连续的内存空间中。同时，这块内存容纳的字节数量也会被记录下来，并用于表示该<code>string</code>值的长度。</p><p>你可以把这块内存的内容看成一个字节数组，而相应的<code>string</code>值则包含了指向字节数组头部的指针值。如此一来，我们在一个<code>string</code>值上应用切片表达式，就相当于在对其底层的字节数组做切片。</p><p>另外，我们在进行字符串拼接的时候，Go语言会把所有被拼接的字符串依次拷贝到一个崭新且足够大的连续内存空间中，并把持有相应指针值的<code>string</code>值作为结果返回。</p><p>显然，当程序中存在过多的字符串拼接操作的时候，会对内存的分配产生非常大的压力。</p><p>注意，虽然<code>string</code>值在内部持有一个指针值，但其类型仍然属于值类型。不过，由于<code>string</code>值的不可变，其中的指针值也为内存空间的节省做出了贡献。</p><p>更具体地说，一个<code>string</code>值会在底层与它的所有副本共用同一个字节数组。由于这里的字节数组永远不会被改变，所以这样做是绝对安全的。</p><p><strong>与<code>string</code>值相比，<code>Builder</code>值的优势其实主要体现在字符串拼接方面。</strong></p><p><code>Builder</code>值中有一个用于承载内容的容器（以下简称内容容器）。它是一个以<code>byte</code>为元素类型的切片（以下简称字节切片）。</p><p>由于这样的字节切片的底层数组就是一个字节数组，所以我们可以说它与<code>string</code>值存储内容的方式是一样的。</p><p>实际上，它们都是通过一个<code>unsafe.Pointer</code>类型的字段来持有那个指向了底层字节数组的指针值的。</p><p>正是因为这样的内部构造，<code>Builder</code>值同样拥有高效利用内存的前提条件。虽然，对于字节切片本身来说，它包含的任何元素值都可以被修改，但是<code>Builder</code>值并不允许这样做，其中的内容只能够被拼接或者完全重置。</p><p>这就意味着，已存在于<code>Builder</code>值中的内容是不可变的。因此，我们可以利用<code>Builder</code>值提供的方法拼接更多的内容，而丝毫不用担心这些方法会影响到已存在的内容。</p><blockquote>
<p><span class="reference">这里所说的方法指的是，<code>Builder</code>值拥有的一系列指针方法，包括：<code>Write</code>、<code>WriteByte</code>、<code>WriteRune</code>和<code>WriteString</code>。我们可以把它们统称为拼接方法。</span></p>
</blockquote><p>我们可以通过调用上述方法把新的内容拼接到已存在的内容的尾部（也就是右边）。这时，如有必要，<code>Builder</code>值会自动地对自身的内容容器进行扩容。这里的自动扩容策略与切片的扩容策略一致。</p><p>换句话说，我们在向<code>Builder</code>值拼接内容的时候并不一定会引起扩容。只要内容容器的容量够用，扩容就不会进行，针对于此的内存分配也不会发生。同时，只要没有扩容，<code>Builder</code>值中已存在的内容就不会再被拷贝。</p><p>除了<code>Builder</code>值的自动扩容，我们还可以选择手动扩容，这通过调用<code>Builder</code>值的<code>Grow</code>方法就可以做到。<code>Grow</code>方法也可以被称为扩容方法，它接受一个<code>int</code>类型的参数<code>n</code>，该参数用于代表将要扩充的字节数量。</p><p>如有必要，<code>Grow</code>方法会把其所属值中内容容器的容量增加<code>n</code>个字节。更具体地讲，它会生成一个字节切片作为新的内容容器，该切片的容量会是原容器容量的二倍再加上<code>n</code>。之后，它会把原容器中的所有字节全部拷贝到新容器中。</p><pre><code>var builder1 strings.Builder
// 省略若干代码。
fmt.Println(&quot;Grow the builder ...&quot;)
builder1.Grow(10)
fmt.Printf(&quot;The length of contents in the builder is %d.\n&quot;, builder1.Len())
</code></pre><p>当然，<code>Grow</code>方法还可能什么都不做。这种情况的前提条件是：当前的内容容器中的未用容量已经够用了，即：未用容量大于或等于<code>n</code>。这里的前提条件与前面提到的自动扩容策略中的前提条件是类似的。</p><pre><code>fmt.Println(&quot;Reset the builder ...&quot;)
builder1.Reset()
fmt.Printf(&quot;The third output(%d):\n%q\n&quot;, builder1.Len(), builder1.String())
</code></pre><p>最后，<code>Builder</code>值是可以被重用的。通过调用它的<code>Reset</code>方法，我们可以让<code>Builder</code>值重新回到零值状态，就像它从未被使用过那样。</p><p>一旦被重用，<code>Builder</code>值中原有的内容容器会被直接丢弃。之后，它和其中的所有内容，将会被Go语言的垃圾回收器标记并回收掉。</p><h2>知识扩展</h2><h3>问题1：<code>strings.Builder</code>类型在使用上有约束吗？</h3><p>答案是：有约束，概括如下：</p><ul>
<li>在已被真正使用后就不可再被复制；</li>
<li>由于其内容不是完全不可变的，所以需要使用方自行解决操作冲突和并发安全问题。</li>
</ul><p>我们只要调用了<code>Builder</code>值的拼接方法或扩容方法，就意味着开始真正使用它了。显而易见，这些方法都会改变其所属值中的内容容器的状态。</p><p>一旦调用了它们，我们就不能再以任何的方式对其所属值进行复制了。否则，只要在任何副本上调用上述方法就都会引发panic。</p><p>这种panic会告诉我们，这样的使用方式是并不合法的，因为这里的<code>Builder</code>值是副本而不是原值。顺便说一句，这里所说的复制方式，包括但不限于在函数间传递值、通过通道传递值、把值赋予变量等等。</p><pre><code>var builder1 strings.Builder
builder1.Grow(1)
builder3 := builder1
//builder3.Grow(1) // 这里会引发panic。
_ = builder3
</code></pre><p>虽然这个约束非常严格，但是如果我们仔细思考一下的话，就会发现它还是有好处的。</p><p>正是由于已使用的<code>Builder</code>值不能再被复制，所以肯定不会出现多个<code>Builder</code>值中的内容容器（也就是那个字节切片）共用一个底层字节数组的情况。这样也就避免了多个同源的<code>Builder</code>值在拼接内容时可能产生的冲突问题。</p><p>不过，虽然已使用的<code>Builder</code>值不能再被复制，但是它的指针值却可以。无论什么时候，我们都可以通过任何方式复制这样的指针值。注意，这样的指针值指向的都会是同一个<code>Builder</code>值。</p><pre><code>f2 := func(bp *strings.Builder) {
 (*bp).Grow(1) // 这里虽然不会引发panic，但不是并发安全的。
 builder4 := *bp
 //builder4.Grow(1) // 这里会引发panic。
 _ = builder4
}
f2(&amp;builder1)
</code></pre><p>正因为如此，这里就产生了一个问题，即：如果<code>Builder</code>值被多方同时操作，那么其中的内容就很可能会产生混乱。这就是我们所说的操作冲突和并发安全问题。</p><p><code>Builder</code>值自己是无法解决这些问题的。所以，我们在通过传递其指针值共享<code>Builder</code>值的时候，一定要确保各方对它的使用是正确、有序的，并且是并发安全的；而最彻底的解决方案是，绝不共享<code>Builder</code>值以及它的指针值。</p><p>我们可以在各处分别声明一个<code>Builder</code>值来使用，也可以先声明一个<code>Builder</code>值，然后在真正使用它之前，便将它的副本传到各处。另外，我们还可以先使用再传递，只要在传递之前调用它的<code>Reset</code>方法即可。</p><pre><code>builder1.Reset()
builder5 := builder1
builder5.Grow(1) // 这里不会引发panic。
</code></pre><p>总之，关于复制<code>Builder</code>值的约束是有意义的，也是很有必要的。虽然我们仍然可以通过某些方式共享<code>Builder</code>值，但最好还是不要以身犯险，“各自为政”是最好的解决方案。不过，对于处在零值状态的<code>Builder</code>值，复制不会有任何问题。</p><h3>问题2：为什么说<code>strings.Reader</code>类型的值可以高效地读取字符串？</h3><p>与<code>strings.Builder</code>类型恰恰相反，<code>strings.Reader</code>类型是为了高效读取字符串而存在的。后者的高效主要体现在它对字符串的读取机制上，它封装了很多用于在<code>string</code>值上读取内容的最佳实践。</p><p><code>strings.Reader</code>类型的值（以下简称<code>Reader</code>值）可以让我们很方便地读取一个字符串中的内容。在读取的过程中，<code>Reader</code>值会保存已读取的字节的计数（以下简称已读计数）。</p><p>已读计数也代表着下一次读取的起始索引位置。<code>Reader</code>值正是依靠这样一个计数，以及针对字符串值的切片表达式，从而实现快速读取。</p><p>此外，这个已读计数也是读取回退和位置设定时的重要依据。虽然它属于<code>Reader</code>值的内部结构，但我们还是可以通过该值的<code>Len</code>方法和<code>Size</code>把它计算出来的。代码如下：</p><pre><code>var reader1 strings.Reader
// 省略若干代码。
readingIndex := reader1.Size() - int64(reader1.Len()) // 计算出的已读计数。
</code></pre><p><code>Reader</code>值拥有的大部分用于读取的方法都会及时地更新已读计数。比如，<code>ReadByte</code>方法会在读取成功后将这个计数的值加<code>1</code>。</p><p>又比如，<code>ReadRune</code>方法在读取成功之后，会把被读取的字符所占用的字节数作为计数的增量。</p><p>不过，<code>ReadAt</code>方法算是一个例外。它既不会依据已读计数进行读取，也不会在读取后更新它。正因为如此，这个方法可以自由地读取其所属的<code>Reader</code>值中的任何内容。</p><p>除此之外，<code>Reader</code>值的<code>Seek</code>方法也会更新该值的已读计数。实际上，这个<code>Seek</code>方法的主要作用正是设定下一次读取的起始索引位置。</p><p>另外，如果我们把常量<code>io.SeekCurrent</code>的值作为第二个参数值传给该方法，那么它还会依据当前的已读计数，以及第一个参数<code>offset</code>的值来计算新的计数值。</p><p>由于<code>Seek</code>方法会返回新的计数值，所以我们可以很容易地验证这一点。比如像下面这样：</p><pre><code>offset2 := int64(17)
expectedIndex := reader1.Size() - int64(reader1.Len()) + offset2
fmt.Printf(&quot;Seek with offset %d and whence %d ...\n&quot;, offset2, io.SeekCurrent)
readingIndex, _ := reader1.Seek(offset2, io.SeekCurrent)
fmt.Printf(&quot;The reading index in reader: %d (returned by Seek)\n&quot;, readingIndex)
fmt.Printf(&quot;The reading index in reader: %d (computed by me)\n&quot;, expectedIndex)
</code></pre><p>综上所述，<code>Reader</code>值实现高效读取的关键就在于它内部的已读计数。计数的值就代表着下一次读取的起始索引位置。它可以很容易地被计算出来。<code>Reader</code>值的<code>Seek</code>方法可以直接设定该值中的已读计数值。</p><h2>总结</h2><p>今天，我们主要讨论了<code>strings</code>代码包中的两个重要类型，即：<code>Builder</code>和<code>Reader</code>。前者用于构建字符串，而后者则用于读取字符串。</p><p>与<code>string</code>值相比，<code>Builder</code>值的优势主要体现在字符串拼接方面。它可以在保证已存在的内容不变的前提下，拼接更多的内容，并且会在拼接的过程中，尽量减少内存分配和内容拷贝的次数。</p><p>不过，这类值在使用上也是有约束的。它在被真正使用之后就不能再被复制了，否则就会引发panic。虽然这个约束很严格，但是也可以带来一定的好处。它可以有效地避免一些操作冲突。虽然我们可以通过一些手段（比如传递它的指针值）绕过这个约束，但这是弊大于利的。最好的解决方案就是分别声明、分开使用、互不干涉。</p><p><code>Reader</code>值可以让我们很方便地读取一个字符串中的内容。它的高效主要体现在它对字符串的读取机制上。在读取的过程中，<code>Reader</code>值会保存已读取的字节的计数，也称已读计数。</p><p>这个计数代表着下一次读取的起始索引位置，同时也是高效读取的关键所在。我们可以利用这类值的<code>Len</code>方法和<code>Size</code>方法，计算出其中的已读计数的值。有了它，我们就可以更加灵活地进行字符串读取了。</p><p>我只在本文介绍了上述两个数据类型，但并不意味着<code>strings</code>包中有用的程序实体只有这两个。实际上，<code>strings</code>包还提供了大量的函数。比如：</p><pre><code>`Count`、`IndexRune`、`Map`、`Replace`、`SplitN`、`Trim`，等等。
</code></pre><p>它们都是非常易用和高效的。你可以去看看它们的源码，也许会因此有所感悟。</p><h2>思考题</h2><p>今天的思考题是：<code>*strings.Builder</code>和<code>*strings.Reader</code>都分别实现了哪些接口？这样做有什么好处吗？</p><p><a href="https://github.com/hyper0x/Golang_Puzzlers">戳此查看Go语言专栏文章配套详细代码。</a></p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/7f/d3/b5896293.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Realm</span>
  </div>
  <div class="_2_QraFYR_0">1 string拼接的结果是生成新的string，需要把原字符串拷贝到新的string中；Builder底层有个[]byte,按需扩容，不必每次拼接都需要拷贝；<br><br>2 Reader的优势是维护一个已读计数器，知道下一次读的位置，读得更快.</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，是的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-05 22:55:44</div>
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
  <div class="_2_QraFYR_0">二刷了一遍，又看了一遍源码；我觉得对于Builder和Reader理解应该注意：<br>1，结构：<br>    1.1 Builder结构体内部内容容器是一个切片buf还有一个addr（复制检测用的指针）<br>    1.2 Reader结构体内部内容容器是一个string的s和一个内部计数器i<br>2. Builder<br>    2.1 想法方法内部先调用copyCheck方法进行值复制的检测（即老师说的使用后在复制引发panic就是这个方法）<br>    2.2 内容容器是切片，相关拼接方法内部应用的是append函数，这些方法使用时间可以结合slice和append的原理<br>    2.3 公开方法Grow进行是否扩容判断逻辑，然后调用内部方法grow执行切片扩容，扩容策略：原内容容器切片容量 * 2 + Grow参数n；用这个容量make申请新的内存空间，然后copy原内容容器切片底层数组值<br><br>3. Reader<br>   3.1 读取方法底层是对内容容器s字符串的切片操作，这里要注意在多字节字符读取时，字符串的切片操作可能会导致拿到的字符串有乱码的风险，<br>   3.2 对于Read、ReadAt这些将字符串读取到传入的切片参数时，底层应用的是copy函数，so最终读出的字符串字节切片长度是copy函数两个参数中较小的一个参数的长度。同时Read、ReadAt这些方法的off参数不恰当时，会因为多字节字符串切片导致两头可能出现乱码</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-09 12:02:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/4b/f3/8b9df836.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jimmy</span>
  </div>
  <div class="_2_QraFYR_0">strings.Builder里边的String方法是<br>&#47;&#47; String returns the accumulated string.<br>func (b *Builder) String() string {<br>	return *(*string)(unsafe.Pointer(&amp;b.buf))<br>}<br>这样实现的， 请问老师为什么不是<br>&#47;&#47; String returns the accumulated string.<br>func (b *Builder) String() string {<br>	return string(b.buf)<br>}<br>有什么特殊的点吗？ 谢谢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 省去了类型转换的开销，效率会高很多。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-17 19:49:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/72/f1/24bc01e3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>南方有嘉木</span>
  </div>
  <div class="_2_QraFYR_0">请问容量增加n个字节，为什么是原来的2倍再加上n呢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-27 12:46:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/b8/77/0e1f2b4b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Garry</span>
  </div>
  <div class="_2_QraFYR_0">老师，我在看strings 源码的时候发现了<br>func noescape(p unsafe.Pointer) unsafe.Pointer {<br>	x := uintptr(p)<br>	return unsafe.Pointer(x ^ 0)<br>}<br>这个函数 最后用了个x ^ 0,但是这么操作的最后结果不还是x么，为何还要这样操作呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 为了产生一个新值啊，要跟这个函数的参数值划清界限。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-02 13:47:00</div>
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
  <div class="_2_QraFYR_0">看到源码有处不理解如下：<br>func noescape(p unsafe.Pointer) unsafe.Pointer {<br>x := uintptr(p)<br>return unsafe.Pointer(x ^ 0)<br>}<br>这个方法的意思避免逃逸分析，不太理解请指教？<br>第一：为什么经过这么转换会避免逃逸？<br>第二：避免逃逸有什么好处，既然会逃逸肯定会到heap上，如果避免逃逸那这个变量怎么使用呢，或者说是这样再stack上又分配了一个新的变量么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 第一个问题：<br><br>因为这个函数的结果值是一个在“内部”生成的新值，不再与那个参数值有任何关系。关键在于 uintptr 的中转。<br><br>首先你得理解，逃逸分析是什么。Go语言的编译器如果发现一个goroutine中的程序持有存放在其他goroutine的堆栈中的指针，那么就会把这个指针指向的值分配到堆上。这么做主要是为了方便goroutine堆栈的自动伸缩。<br><br>注意，把值分配到堆上会给GC带来更大压力。因为堆的区域是公共的，不能像goroutine堆栈那样（在goroutine运行完成后）被自动清除。<br><br>第二个问题：<br><br>避免逃逸分析这种做法只应该在Go语言内部使用。因为这是一把双刃剑。它可以避免增加GC的压力，但是如果使用不当，在运行时系统伸缩goroutine堆栈时就会出问题。<br><br>逃逸分析之所以称为分析，是因为它有一个分析的过程。这不是在程序运行时做的，而是在编译时做的（决定一个值是分配在当前goroutine的堆栈上还是分配在公共的堆上），所以不存在拷贝来拷贝去的问题。<br><br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-23 11:14:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/97/e1/0f4d90ff.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>乖，摸摸头</span>
  </div>
  <div class="_2_QraFYR_0">strings.Reader这里我一直有个疑问， 它读写  很对地方都是<br>if r.i &gt;= int64(len(r.s)) {<br>		return 0, nil<br>}<br>为什么在  strings.NewReader 的时候 不直接求出 len(r.s)的长度，而是每次去算长度，这样不会有性能浪费吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你是在说“为什么每次都len(r.s)，而不把长度信息存储在某个地方”么？<br><br>这样其实根本不会浪费性能。因为字符串值中本身就存着字符串的长度，详见Go语言源码文件src&#47;runtime&#47;string.go中的类型定义stringStruct。<br><br>况且，把这种信息记录在reader内会造成额外的维护成本。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-10 15:47:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/3e/77/22843055.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>kingkang</span>
  </div>
  <div class="_2_QraFYR_0">请问byte数组转string出现乱码怎么处理？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果字节数组的内容不是UTF-8编码的Unicode字符，这样直接转就会出现乱码。先要搞清楚两个问题：1. 这个字节数组的内容会是可打印的字符吗？2. 如果是可打印的字符，那它使用什么编码的？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-04 19:36:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/80/b5/f59d92f1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Cloud</span>
  </div>
  <div class="_2_QraFYR_0">很实用！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-05 10:03:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/GBU53SA3W8GNRAwZIicc3gTEc0nSvfPJw7iboAMicjicmP6egDcibib28DkUfTYOjMd31DIznmofdRZrpIXvmXvjV1PQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>博博</span>
  </div>
  <div class="_2_QraFYR_0">Builder类型中的addr *Builder 字段的意义是什么呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个 addr 字段的意义是，保存其所属值所在的内存地址。如此一来，一旦这个值被拷贝了，使用内存地址比较的方式就可以检测出来。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-22 17:42:21</div>
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
  <div class="_2_QraFYR_0">郝林老师，能分析以下这段代码的执行步骤吗？没怎么看懂：<br><br>*(*string)(unsafe.Pointer(&amp;b.buf))  </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: unsafe.Pointer(&amp;b.buf) <br>-&gt; 得到一个 Pointer 值<br>-&gt; 将这个 Pointer 值的类型转换为 *string（即 string 的指针类型）<br>-&gt; 在这个 *string 类型的指针值之上求值（也就是求它指向的那个值）<br>-&gt; 得到一个 string 类型的值 （即一个字符串值） </p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-24 16:15:46</div>
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
  <div class="_2_QraFYR_0">郝林老师，麻烦看看以下问题：<br><br>1. &quot;不过，由于string值的不可变，其中的指针值也为内存空间的节省做出了贡献&quot;。 这句话该怎么理解呢？<br><br>2. 文中的这段代码：<br><br>f2 := func(bp *strings.Builder) { <br><br>(*bp).Grow(1)  &#47;&#47; 这里虽然不会引发panic，但不是并发安全的。 <br>builder4 := *bp<br> &#47;&#47;builder4.Grow(1) &#47;&#47; 这里会引发panic。<br> _ = builder4<br> <br>}<br><br>f2(&amp;builder1)<br><br>不是说：“虽然已使用的Builder值不能再被复制，但是它的指针值却可以。”<br><br>那这段代码：builder4.Grow(1) 。 为何还会引发panic呢？ </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: builder4 := *bp 会把 bp 指向的原值复制一份并赋给变量 builder4。当我们调用 (*bp).Grow(1) 时调用的还是原值的方法，但是调用 builder4.Grow(1) 就是在调用原值的副本的方法了。非空的 strings.Builder 值是禁止被复制的，所以副本在发现自己是副本之后抛出了 panic 。 </p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-24 11:47:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/8f/df/6261264f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>微微一怒很倾城</span>
  </div>
  <div class="_2_QraFYR_0">对于builder的主要性能优势，我的理解是原始的string拼接，因为每次都要生成新的string，所以每次都要重新分配内存和拷贝，但是builder就不需要，只需要第一次分配，然后后面就不停地拼接和拷贝</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-16 20:23:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/96/94/abb7bfe3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Ke</span>
  </div>
  <div class="_2_QraFYR_0">strings.Builder使用之后，可以从strings.Builder复制出来新变量，但是这个变量无法在使用WriteString或者Grow之类的操作，只能读取，并且就算原始的strings.Builder变量做了reset，这个新变量的读取也不受影响</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-16 14:51:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83erNhKGpqicibpQO3tYvl9vwiatvBzn27ut9y5lZ8hPgofPCFC24HX3ko7LW5mNWJficgJncBCGKpGL2jw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_1ed70f</span>
  </div>
  <div class="_2_QraFYR_0">读源代码讲得好深....</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-14 11:49:45</div>
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
  <div class="_2_QraFYR_0">打卡</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-04 01:33:15</div>
  </div>
</div>
</div>
</li>
</ul>