<audio title="39 _ bytes包与字节串操作（下）" src="https://static001.geekbang.org/resource/audio/1f/c6/1f81df9ff385ae6e343d551f09628fc6.mp3" controls="controls"></audio> 
<p>你好，我是郝林，今天我们继续分享bytes包与字节串操作的相关内容。</p><p>在上一篇文章中，我们分享了<code>bytes.Buffer</code>中已读计数的大致功用，并围绕着这个问题做了解析，下面我们来进行相关的知识扩展。</p><h2>知识扩展</h2><h3>问题 1：<code>bytes.Buffer</code>的扩容策略是怎样的？</h3><p><code>Buffer</code>值既可以被手动扩容，也可以进行自动扩容。并且，这两种扩容方式的策略是基本一致的。所以，除非我们完全确定后续内容所需的字节数，否则让<code>Buffer</code>值自动去扩容就好了。</p><p>在扩容的时候，<code>Buffer</code>值中相应的代码（以下简称扩容代码）会<strong>先判断内容容器的剩余容量</strong>，是否可以满足调用方的要求，或者是否足够容纳新的内容。</p><p><strong>如果可以，那么扩容代码会在当前的内容容器之上，进行长度扩充。</strong></p><p>更具体地说，如果内容容器的容量与其长度的差，大于或等于另需的字节数，那么扩容代码就会通过切片操作对原有的内容容器的长度进行扩充，就像下面这样：</p><pre><code>b.buf = b.buf[:length+need]
</code></pre><p><strong>反之，如果内容容器的剩余容量不够了，那么扩容代码可能就会用新的内容容器去替代原有的内容容器，从而实现扩容。</strong></p><p>不过，这里还有一步优化。</p><p><strong>如果当前内容容器的容量的一半，仍然大于或等于其现有长度（即未读字节数）再加上另需的字节数的和</strong>，即：</p><!-- [[[read_end]]] --><pre><code>cap(b.buf)/2 &gt;= b.Len() + need
</code></pre><p>那么，扩容代码就会复用现有的内容容器，并把容器中的未读内容拷贝到它的头部位置。</p><p>这也意味着其中的已读内容，将会全部被未读内容和之后的新内容覆盖掉。</p><p>这样的复用预计可以至少节省掉一次后续的扩容所带来的内存分配，以及若干字节的拷贝。</p><p><strong>若这一步优化未能达成</strong>，也就是说，当前内容容器的容量小于新长度的二倍。</p><p>那么，扩容代码就只能再创建一个新的内容容器，并把原有容器中的未读内容拷贝进去，最后再用新的容器替换掉原有的容器。这个新容器的容量将会等于原有容量的二倍再加上另需字节数的和。</p><blockquote>
<p>新容器的容量=2*原有容量+所需字节数</p>
</blockquote><p>通过上面这些步骤，对内容容器的扩充基本上就完成了。不过，为了内部数据的一致性，以及避免原有的已读内容可能造成的数据混乱，扩容代码还会把已读计数置为<code>0</code>，并再对内容容器做一下切片操作，以掩盖掉原有的已读内容。</p><p>顺便说一下，对于处在零值状态的<code>Buffer</code>值来说，如果第一次扩容时的另需字节数不大于<code>64</code>，那么该值就会基于一个预先定义好的、长度为<code>64</code>的字节数组来创建内容容器。</p><p>在这种情况下，这个内容容器的容量就是<code>64</code>。这样做的目的是为了让<code>Buffer</code>值在刚被真正使用的时候就可以快速地做好准备。</p><h3>问题2：<code>bytes.Buffer</code>中的哪些方法可能会造成内容的泄露？</h3><p>首先明确一点，什么叫内容泄露？这里所说的内容泄露是指，使用<code>Buffer</code>值的一方通过某种非标准的（或者说不正式的）方式，得到了本不该得到的内容。</p><p>比如说，我通过调用<code>Buffer</code>值的某个用于读取内容的方法，得到了一部分未读内容。我应该，也只应该通过这个方法的结果值，拿到在那一时刻<code>Buffer</code>值中的未读内容。</p><p>但是，在这个<code>Buffer</code>值又有了一些新内容之后，我却可以通过当时得到的结果值，直接获得新的内容，而不需要再次调用相应的方法。</p><p>这就是典型的非标准读取方式。这种读取方式是不应该存在的，即使存在，我们也不应该使用。因为它是在无意中（或者说一不小心）暴露出来的，其行为很可能是不稳定的。</p><p>在<code>bytes.Buffer</code>中，<code>Bytes</code>方法和<code>Next</code>方法都可能会造成内容的泄露。原因在于，它们都把基于内容容器的切片直接返回给了方法的调用方。</p><p>我们都知道，通过切片，我们可以直接访问和操纵它的底层数组。不论这个切片是基于某个数组得来的，还是通过对另一个切片做切片操作获得的，都是如此。</p><p>在这里，<code>Bytes</code>方法和<code>Next</code>方法返回的字节切片，都是通过对内容容器做切片操作得到的。也就是说，它们与内容容器共用了同一个底层数组，起码在一段时期之内是这样的。</p><p>以<code>Bytes</code>方法为例。它会返回在调用那一刻其所属值中的所有未读内容。示例代码如下：</p><pre><code>contents := &quot;ab&quot;
buffer1 := bytes.NewBufferString(contents)
fmt.Printf(&quot;The capacity of new buffer with contents %q: %d\n&quot;,
 contents, buffer1.Cap()) // 内容容器的容量为：8。
unreadBytes := buffer1.Bytes()
fmt.Printf(&quot;The unread bytes of the buffer: %v\n&quot;, unreadBytes) // 未读内容为：[97 98]。
</code></pre><p>我用字符串值<code>"ab"</code>初始化了一个<code>Buffer</code>值，由变量<code>buffer1</code>代表，并打印了当时该值的一些状态。</p><p>你可能会有疑惑，我只在这个<code>Buffer</code>值中放入了一个长度为<code>2</code>的字符串值，但为什么该值的容量却变为了<code>8</code>。</p><p>虽然这与我们当前的主题无关，但是我可以提示你一下：你可以去阅读<code>runtime</code>包中一个名叫<code>stringtoslicebyte</code>的函数，答案就在其中。</p><p>接着说<code>buffer1</code>。我又向该值写入了字符串值<code>"cdefg"</code>，此时，其容量仍然是<code>8</code>。我在前面通过调用<code>buffer1</code>的<code>Bytes</code>方法得到的结果值<code>unreadBytes</code>，包含了在那时其中的所有未读内容。</p><p>但是，由于这个结果值与<code>buffer1</code>的内容容器在此时还共用着同一个底层数组，所以，我只需通过简单的再切片操作，就可以利用这个结果值拿到<code>buffer1</code>在此时的所有未读内容。如此一来，<code>buffer1</code>的新内容就被泄露出来了。</p><pre><code>buffer1.WriteString(&quot;cdefg&quot;)
fmt.Printf(&quot;The capacity of buffer: %d\n&quot;, buffer1.Cap()) // 内容容器的容量仍为：8。
unreadBytes = unreadBytes[:cap(unreadBytes)]
fmt.Printf(&quot;The unread bytes of the buffer: %v\n&quot;, unreadBytes) // 基于前面获取到的结果值可得，未读内容为：[97 98 99 100 101 102 103 0]。
</code></pre><p>如果我当时把<code>unreadBytes</code>的值传到了外界，那么外界就可以通过该值操纵<code>buffer1</code>的内容了，就像下面这样：</p><pre><code>unreadBytes[len(unreadBytes)-2] = byte('X') // 'X'的ASCII编码为88。
fmt.Printf(&quot;The unread bytes of the buffer: %v\n&quot;, buffer1.Bytes()) // 未读内容变为了：[97 98 99 100 101 102 88]。
</code></pre><p>现在，你应该能够体会到，这里的内容泄露可能造成的严重后果了吧？对于<code>Buffer</code>值的<code>Next</code>方法，也存在相同的问题。</p><p>不过，如果经过扩容，<code>Buffer</code>值的内容容器或者它的底层数组被重新设定了，那么之前的内容泄露问题就无法再进一步发展了。我在demo80.go文件中写了一个比较完整的示例，你可以去看一看，并揣摩一下。</p><h2>总结</h2><p>我们结合两篇内容总结一下。与<code>strings.Builder</code>类型不同，<code>bytes.Buffer</code>不但可以拼接、截断其中的字节序列，以各种形式导出其中的内容，还可以顺序地读取其中的子序列。</p><p><code>bytes.Buffer</code>类型使用字节切片作为其内容容器，并且会用一个字段实时地记录已读字节的计数。</p><p>虽然我们无法直接计算出这个已读计数，但是由于它在<code>Buffer</code>值中起到的作用非常关键，所以我们很有必要去理解它。</p><p>无论是读取、写入、截断、导出还是重置，已读计数都是功能实现中的重要一环。</p><p>与<code>strings.Builder</code>类型的值一样，<code>Buffer</code>值既可以被手动扩容，也可以进行自动的扩容。除非我们完全确定后续内容所需的字节数，否则让<code>Buffer</code>值自动去扩容就好了。</p><p><code>Buffer</code>值的扩容方法并不一定会为了获得更大的容量，替换掉现有的内容容器，而是先会本着尽量减少内存分配和内容拷贝的原则，对当前的内容容器进行重用。并且，只有在容量实在无法满足要求的时候，它才会去创建新的内容容器。</p><p>此外，你可能并没有想到，<code>Buffer</code>值的某些方法可能会造成内容的泄露。这主要是由于这些方法返回的结果值，在一段时期内会与其所属值的内容容器共用同一个底层数组。</p><p><strong>如果我们有意或无意地把这些结果值传到了外界，那么外界就有可能通过它们操纵相关联<code>Buffer</code>值的内容。</strong></p><p>这属于很严重的数据安全问题。我们一定要避免这种情况的发生。最彻底的做法是，在传出切片这类值之前要做好隔离。比如，先对它们进行深度拷贝，然后再把副本传出去。</p><h2>思考题</h2><p>今天的思考题是：对比<code>strings.Builder</code>和<code>bytes.Buffer</code>的<code>String</code>方法，并判断哪一个更高效？原因是什么？</p><p><a href="https://github.com/hyper0x/Golang_Puzzlers">戳此查看Go语言专栏文章配套详细代码。</a></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/4c/b2/74519a7c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>失了智的沫雨</span>
  </div>
  <div class="_2_QraFYR_0">如果只看strings.Builder 和bytes.Buffer的String方法的话，strings.Builder 更高效一些。<br>我们可以直接查看两个String方法的源代码，其中strings.Builder String方法中<br>*(*string)(unsafe.Pointer(&amp;b.buf))  是直接取得buf的地址然后转换成string返回。<br>而bytes.Buffer的String方法是   string(b.buf[b.off:])<br> 对buf 进行切片操作,我认为这比直接取址要花费更多的时间。<br>测试函数:<br>func BenchmarkStrings(b *testing.B) {<br>	str := strings.Builder{}&#47;bytes.Buffer{}<br>	str.WriteString(&quot;test&quot;)<br>	for i := 0; i &lt; b.N; i++ {<br>		str.String()<br>	}<br>}<br>结果为<br>BenchmarkStrings-8   	2000000000	         0.66 ns&#47;op<br>BenchmarkBuffer-8    	300000000	         5.64 ns&#47;op<br>所以strings.Builder的String方法更高效</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-11 18:14:25</div>
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
  <div class="_2_QraFYR_0">https:&#47;&#47;github.com&#47;golang&#47;go&#47;blob&#47;master&#47;src&#47;strings&#47;builder_test.go#L319-L366<br><br>发现最后的问题，Go 的标准库中，已经给出了相关的测试代码了。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-26 22:17:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/e9/ed/daa94663.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>1thinc0</span>
  </div>
  <div class="_2_QraFYR_0">bytes.Buffer 值的 String() 方法在转换时采用了指针 *(*string)(unsafe.Pointer(&amp;b.buf))，更节省时间和内存</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-16 10:17:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_51aa7f</span>
  </div>
  <div class="_2_QraFYR_0">在Byte()或者Next()结果返回的字节切片处理后才可以返回给外部函数<br>unreadBytes = unreadBytes[:len(unreadBytes):len(unreadBytes)]</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-07 03:17:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/4a/42/b2c7dd30.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>骏Jero</span>
  </div>
  <div class="_2_QraFYR_0">读了老师的两篇文章，strings.Builder更多是拼接数据和以及拼接完成后的读取使用上应该更适合。而buffer更为动态接受和读取数据时，更为高效。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-09 10:18:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/18/c4/e55fdc1c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>cygnus</span>
  </div>
  <div class="_2_QraFYR_0">```<br>func (b *Buffer) grow(n int) int {<br>        ......<br>	&#47;&#47; Restore b.off and len(b.buf).<br>	b.off = 0<br>	b.buf = b.buf[:m+n]<br>	return m<br><br>func (b *Buffer) Grow(n int) {<br>	if n &lt; 0 {<br>		panic(&quot;bytes.Buffer.Grow: negative count&quot;)<br>	}<br>	m := b.grow(n)<br>	b.buf = b.buf[:m]<br>}<br>```<br>请问下老师，bytes.Buffer里grow函数返回前做过一次切片b.buf = b.buf[:m+n]，返回后在Grow函数又做了一次切片b.buf = b.buf[:m]，这样做的目的是什么呢？感觉有点冗余</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-13 09:12:23</div>
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
  <div class="_2_QraFYR_0">文章中说：“如果当前内容容器的容量的一半，仍然大于或等于其现有长度再加上另需的字节数的和，cap(b.buf)&#47;2 &gt;= len(b.buf)+need”。<br><br>如果是这样的话，那说明还有很多的未使用的空间，起码有一半，那就不用扩容，直接追加就可以了。所以这里的代码是错的，源码(1.16.2)中是 n &lt;= c&#47;2-m ，即cap(b.buf) &gt;= b.Len() + need，也就是未读字节长度 + 另需的字节数 小于等于容量的一半时，才会把未读字节copy到buf的头部，只有在未读字节较少的时候，才发生copy，如果有太多的未读字节，就不copy到头部了(费时)</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 源码： n &lt;= c&#47;2-m<br>相当于： n &lt;= cap(b.buf)&#47;2 - b.Len()<br>相当于： n + b.Len() &lt;= cap(b.buf)&#47;2<br>得出： cap(b.buf)&#47;2 &gt;= len(b.buf) + need （为了在省略掉多余上下文的同时保持清晰，我在这里使用了更容易理解的代码 cap(b.buf) 和 len(b.buf) ，以及英文单词 need）<br><br>所以我这里没写错啊。你后面描述的不是跟我说的一个意思是么？？<br><br>不过，如果这里让你产生了误解，说明我的文字有所欠缺，我改进一下吧。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-02 19:59:03</div>
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
  <div class="_2_QraFYR_0">老师，Buffer的Write，WriteString方法返回一个int，一个err，int是参数长度，err永远是nil，感觉这个返回没啥用啊。这么设计是为了实现某些接口么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对，主要是实现接口，而且也保留返回错误值的权利和义务。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-08 18:47:20</div>
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
  <div class="_2_QraFYR_0">请问老师，用 bytes.Buffer 而不用字节切片是因为它有计数器和一些方法使得操作更方便，还有高效的扩容策略么。这么说对么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 因为有缓冲区啊，比较适合分步构建字符串值。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-07 20:49:49</div>
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
  <div class="_2_QraFYR_0">老师你好，为什么我本地的src&#47;runtime包下的stringtoslicebyte方法里面tmpBuf的默认长度是32。<br>所以文中例子，输出的容量是32</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你可以把前因后果都摆上来。这篇文章里有 tmpBuf 吗？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-11 21:44:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/f5/96/0cf9f3c7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Aeins</span>
  </div>
  <div class="_2_QraFYR_0">“你可能会有疑惑，我只在这个Buffer值中放入了一个长度为2的字符串值，但为什么该值的容量却变为了8”<br><br>目前，字符串长度小于 32 的直接分配在缓冲区上，Cap 是 32，大于 32 的，再重新分配内存</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-10 16:26:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/c5/5c/04be6fb7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>costaLong</span>
  </div>
  <div class="_2_QraFYR_0">	contents := &quot;ab&quot;<br>	buffer1 := bytes.NewBufferString(contents)<br>	fmt.Printf(&quot;The capacity of new buffer with content %q: %d\n&quot;, contents, buffer1.Cap()) &#47;&#47; 内容容器的容量：8<br>单独执行这段代码输出的结果是：The capacity of new buffer with content &quot;ab&quot;: 32<br> 请问是原因呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 没看明白，“单独执行这段代码”是什么意思？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-17 10:01:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/27/ff/e4/927547a9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>无名无姓</span>
  </div>
  <div class="_2_QraFYR_0">老师讲的很透彻，需要结合源码</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-30 15:47:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/57/41/22e85c6c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>漠映残夕</span>
  </div>
  <div class="_2_QraFYR_0">更具体地说，如果内容容器的容量与其长度的差，大于或等于另需的字节数，那么扩容代码就会通过切片操作对原有的内容容器的长度进行扩充。<br>这句话中与“其长度的差”，表述不明确，应该是已存内容长度的差。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-02 00:40:11</div>
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
  <div class="_2_QraFYR_0">思考题: strings.Builder的实现是 return *(*string)(unsafe.Pointer(&amp;b.buf))， bytes.Buffe的实现是string(b.buf[b.off:])，显然strings.Builder的String方法更高效，它没有发生拷贝，而是直接使用(共享)了buf的内存数据(数组，长度)</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-02 16:37:47</div>
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
  <div class="_2_QraFYR_0">如果当前内容容器的容量的一半，仍然大于或等于其现有长度再加上另需的字节数的和，即：cap(b.buf)&#47;2 &gt;= len(b.buf)+need，那么，扩容代码就会复用现有的内容容器，并把容器中的未读内容拷贝到它的头部位置。这也意味着其中的已读内容，将会全部被未读内容和之后的新内容覆盖掉。<br><br>其实这一步并不会覆盖所有已读内容（copy(b.buf, b.buf[b.off:])），因为根据条件可以推出未读+剩余&lt;已读部分。所以在Grow方法的最后又截取了一下</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-08 18:38:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/23/1c/d5c164df.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>rename</span>
  </div>
  <div class="_2_QraFYR_0">如果当前内容容器的容量的一半，仍然大于或等于其现有长度再加上所需的字节数的和，即：cap(b.buf)&#47;2 &gt;= len(b.buf)+need<br>这边len(b.buf)用b.Len()似乎更准确？才是获取未读部分的实际长度</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 当然是要看 buf 本身的长度了。这是 bytes.Buffer 内部的算法，你可以看一看源码。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-14 16:45:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/b1/0f/e81a93ed.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>嘎嘎</span>
  </div>
  <div class="_2_QraFYR_0">源码里给了推荐的构建方法<br>&#47;&#47; To build strings more efficiently, see the strings.Builder type.<br>func (b *Buffer) String() string {<br>	if b == nil {<br>		&#47;&#47; Special case, useful in debugging.<br>		return &quot;&lt;nil&gt;&quot;<br>	}<br>	return string(b.buf[b.off:])<br>}</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个Buffer类型比strings.Builder类型出现要早。我觉得后者质量更高一些。你可以参看一下后者的String方法。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-15 13:50:03</div>
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
    <div class="_3Hkula0k_0">2019-03-06 08:42:53</div>
  </div>
</div>
</div>
</li>
</ul>