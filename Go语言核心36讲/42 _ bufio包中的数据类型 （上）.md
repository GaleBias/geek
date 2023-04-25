<audio title="42 _ bufio包中的数据类型 （上）" src="https://static001.geekbang.org/resource/audio/b3/9c/b399e847c0dd8b41b8ebd71f7788419c.mp3" controls="controls"></audio> 
<p>今天，我们来讲另一个与I/O操作强相关的代码包<code>bufio</code>。<code>bufio</code>是“buffered I/O”的缩写。顾名思义，这个代码包中的程序实体实现的I/O操作都内置了缓冲区。</p><p><code>bufio</code>包中的数据类型主要有：</p><ol>
<li><code>Reader</code>；</li>
<li><code>Scanner</code>；</li>
<li><code>Writer</code>和<code>ReadWriter</code>。</li>
</ol><p>与<code>io</code>包中的数据类型类似，这些类型的值也都需要在初始化的时候，包装一个或多个简单I/O接口类型的值。（这里的简单I/O接口类型指的就是<code>io</code>包中的那些简单接口。）</p><p>下面，我们将通过一系列问题对<code>bufio.Reader</code>类型和<code>bufio.Writer</code>类型进行讨论（以前者为主）。<strong>今天我的问题是：<code>bufio.Reader</code>类型值中的缓冲区起着怎样的作用？</strong></p><p><strong>这道题的典型回答是这样的。</strong></p><p><code>bufio.Reader</code>类型的值（以下简称<code>Reader</code>值）内的缓冲区，其实就是一个数据存储中介，它介于底层读取器与读取方法及其调用方之间。所谓的底层读取器，就是在初始化此类值的时候传入的<code>io.Reader</code>类型的参数值。</p><p><code>Reader</code>值的读取方法一般都会先从其所属值的缓冲区中读取数据。同时，在必要的时候，它们还会预先从底层读取器那里读出一部分数据，并暂存于缓冲区之中以备后用。</p><!-- [[[read_end]]] --><p>有这样一个缓冲区的好处是，可以在大多数的时候降低读取方法的执行时间。虽然，读取方法有时还要负责填充缓冲区，但从总体来看，读取方法的平均执行时间一般都会因此有大幅度的缩短。</p><h2>问题解析</h2><p><code>bufio.Reader</code>类型并不是开箱即用的，因为它包含了一些需要显式初始化的字段。为了让你能在后面更好地理解它的读取方法的内部流程，我先在这里简要地解释一下这些字段，如下所示。</p><ol>
<li><code>buf</code>：<code>[]byte</code>类型的字段，即字节切片，代表缓冲区。虽然它是切片类型的，但是其长度却会在初始化的时候指定，并在之后保持不变。</li>
<li><code>rd</code>：<code>io.Reader</code>类型的字段，代表底层读取器。缓冲区中的数据就是从这里拷贝来的。</li>
<li><code>r</code>：<code>int</code>类型的字段，代表对缓冲区进行下一次读取时的开始索引。我们可以称它为已读计数。</li>
<li><code>w</code>：<code>int</code>类型的字段，代表对缓冲区进行下一次写入时的开始索引。我们可以称之为已写计数。</li>
<li><code>err</code>：<code>error</code>类型的字段。它的值用于表示在从底层读取器获得数据时发生的错误。这里的值在被读取或忽略之后，该字段会被置为<code>nil</code>。</li>
<li><code>lastByte</code>：<code>int</code>类型的字段，用于记录缓冲区中最后一个被读取的字节。读回退时会用到它的值。</li>
<li><code>lastRuneSize</code>：<code>int</code>类型的字段，用于记录缓冲区中最后一个被读取的Unicode字符所占用的字节数。读回退的时候会用到它的值。这个字段只会在其所属值的<code>ReadRune</code>方法中才会被赋予有意义的值。在其他情况下，它都会被置为<code>-1</code>。</li>
</ol><p><code>bufio</code>包为我们提供了两个用于初始化<code>Reader</code>值的函数，分别叫：</p><ul>
<li>
<p><code>NewReader</code>；</p>
</li>
<li>
<p><code>NewReaderSize</code>；</p>
</li>
</ul><p>它们都会返回一个<code>*bufio.Reader</code>类型的值。</p><p><code>NewReader</code>函数初始化的<code>Reader</code>值会拥有一个默认尺寸的缓冲区。这个默认尺寸是4096个字节，即：4 KB。而<code>NewReaderSize</code>函数则将缓冲区尺寸的决定权抛给了使用方。</p><p>由于这里的缓冲区在一个<code>Reader</code>值的生命周期内其尺寸不可变，所以在有些时候是需要做一些权衡的。<code>NewReaderSize</code>函数就提供了这样一个途径。</p><p>在<code>bufio.Reader</code>类型拥有的读取方法中，<code>Peek</code>方法和<code>ReadSlice</code>方法都会调用该类型一个名为<code>fill</code>的包级私有方法。<code>fill</code>方法的作用是填充内部缓冲区。我们在这里就先重点说说它。</p><p><code>fill</code>方法会先检查其所属值的已读计数。如果这个计数不大于<code>0</code>，那么有两种可能。</p><p>一种可能是其缓冲区中的字节都是全新的，也就是说它们都没有被读取过，另一种可能是缓冲区刚被压缩过。</p><p>对缓冲区的压缩包括两个步骤。<strong>第一步，把缓冲区中在<code>[已读计数, 已写计数)</code>范围之内的所有元素值（或者说字节）都依次拷贝到缓冲区的头部。</strong></p><p>比如，把缓冲区中与已读计数代表的索引对应字节拷贝到索引<code>0</code>的位置，并把紧挨在它后边的字节拷贝到索引<code>1</code>的位置，以此类推。</p><p>这一步之所以不会有任何副作用，是因为它基于两个事实。</p><p><strong>第一事实，</strong>已读计数之前的字节都已经被读取过，并且肯定不会再被读取了，因此把它们覆盖掉是安全的。</p><p><strong>第二个事实，</strong>在压缩缓冲区之后，已写计数之后的字节只可能是已被读取过的字节，或者是已被拷贝到缓冲区头部的未读字节，又或者是代表未曾被填入数据的零值<code>0x00</code>。所以，后续的新字节是可以被写到这些位置上的。</p><p><strong>在压缩缓冲区的第二步中，<code>fill</code>方法会把已写计数的新值设定为原已写计数与原已读计数的差。这个差所代表的索引，就是压缩后第一次写入字节时的开始索引。</strong></p><p>另外，该方法还会把已读计数的值置为<code>0</code>。显而易见，在压缩之后，再读取字节就肯定要从缓冲区的头部开始读了。</p><p><img src="https://static001.geekbang.org/resource/image/68/84/687b56d4137ea4d01e0b20d259f91284.png?wh=1920*934" alt=""></p><p>（bufio.Reader中的缓冲区压缩）</p><p>实际上，<code>fill</code>方法只要在开始时发现其所属值的已读计数大于<code>0</code>，就会对缓冲区进行一次压缩。之后，如果缓冲区中还有可写的位置，那么该方法就会对其进行填充。</p><p>在填充缓冲区的时候，<code>fill</code>方法会试图从底层读取器那里，读取足够多的字节，并尽量把从已写计数代表的索引位置到缓冲区末尾之间的空间都填满。</p><p>在这个过程中，<code>fill</code>方法会及时地更新已写计数，以保证填充的正确性和顺序性。另外，它还会判断从底层读取器读取数据的时候，是否有错误发生。如果有，那么它就会把错误值赋给其所属值的<code>err</code>字段，并终止填充流程。</p><p>好了，到这里，我们暂告一个段落。在本题中，我对<code>bufio.Reader</code>类型的基本结构，以及相关的一些函数和方法进行了概括介绍，并且重点阐述了该类型的<code>fill</code>方法。</p><p>后者是我们在后面要说明的一些读取流程的重要组成部分。你起码要记住的是：这个<code>fill</code>方法大致都做了些什么。</p><h2>知识扩展</h2><p>问题1：<code>bufio.Writer</code>类型值中缓冲的数据什么时候会被写到它的底层写入器？</p><p>我们先来看一下<code>bufio.Writer</code>类型都有哪些字段：</p><ol>
<li><code>err</code>：<code>error</code>类型的字段。它的值用于表示在向底层写入器写数据时发生的错误。</li>
<li><code>buf</code>：<code>[]byte</code>类型的字段，代表缓冲区。在初始化之后，它的长度会保持不变。</li>
<li><code>n</code>：<code>int</code>类型的字段，代表对缓冲区进行下一次写入时的开始索引。我们可以称之为已写计数。</li>
<li><code>wr</code>：<code>io.Writer</code>类型的字段，代表底层写入器。</li>
</ol><p><code>bufio.Writer</code>类型有一个名为<code>Flush</code>的方法，它的主要功能是把相应缓冲区中暂存的所有数据，都写到底层写入器中。数据一旦被写进底层写入器，该方法就会把它们从缓冲区中删除掉。</p><p>不过，这里的删除有时候只是逻辑上的删除而已。不论是否成功地写入了所有的暂存数据，<code>Flush</code>方法都会妥当处置，并保证不会出现重写和漏写的情况。该类型的字段<code>n</code>在此会起到很重要的作用。</p><p><code>bufio.Writer</code>类型值（以下简称<code>Writer</code>值）拥有的所有数据写入方法都会在必要的时候调用它的<code>Flush</code>方法。</p><p>比如，<code>Write</code>方法有时候会在把数据写进缓冲区之后，调用<code>Flush</code>方法，以便为后续的新数据腾出空间。<code>WriteString</code>方法的行为与之类似。</p><p>又比如，<code>WriteByte</code>方法和<code>WriteRune</code>方法，都会在发现缓冲区中的可写空间不足以容纳新的字节，或Unicode字符的时候，调用<code>Flush</code>方法。</p><p>此外，如果<code>Write</code>方法发现需要写入的字节太多，同时缓冲区已空，那么它就会跨过缓冲区，并直接把这些数据写到底层写入器中。</p><p>而<code>ReadFrom</code>方法，则会在发现底层写入器的类型是<code>io.ReaderFrom</code>接口的实现之后，直接调用其<code>ReadFrom</code>方法把参数值持有的数据写进去。</p><p>总之，在通常情况下，只要缓冲区中的可写空间无法容纳需要写入的新数据，<code>Flush</code>方法就一定会被调用。并且，<code>bufio.Writer</code>类型的一些方法有时候还会试图走捷径，跨过缓冲区而直接对接数据供需的双方。</p><p>你可以在理解了这些内部机制之后，有的放矢地编写你的代码。不过，在你把所有的数据都写入<code>Writer</code>值之后，再调用一下它的<code>Flush</code>方法，显然是最稳妥的。</p><h2>总结</h2><p>今天我们从“<code>bufio.Reader</code>类型值中的缓冲区起着怎样的作用”这道问题入手，介绍了一部分bufio包中的数据类型，在下一次的分享中，我会沿着这个问题继续展开。</p><p>你对今天的内容有什么样的思考，可以给我留言，我们一起讨论。感谢你的收听，我们下期再见。</p><p><a href="https://github.com/hyper0x/Golang_Puzzlers">戳此查看Go语言专栏文章配套详细代码。</a></p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/60/47/fa06038b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>人生入戏须尽欢</span>
  </div>
  <div class="_2_QraFYR_0">bufio和bytes.buffer有什么区别吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: bufio 包中的 Reader 和 Writer 的主要作用是在已有的 IO 读取器或者写入器之上再包一层缓冲区，是面向 IO 操作。而 bytes.Buffer 是基于字节数组的缓冲区，是面向纯内存的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-05 17:19:19</div>
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
  <div class="_2_QraFYR_0">bufio的应用场景应该是为了加快io速度，尤其是对比较零碎的数据（小数据）的io加速更明显。<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-19 21:24:36</div>
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
  <div class="_2_QraFYR_0">老师，什么场景适合使用bufio，能否举几个栗子呀</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-19 08:43:02</div>
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
  <div class="_2_QraFYR_0">对照源码看专栏，好理解好记忆。👍</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-13 10:47:25</div>
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
  <div class="_2_QraFYR_0">郝林老师你好。<br><br>请问一下，demo84.go中：示例4和示例5中，为什么调用Peek方法最后返回的字节切片的长度都为213呀。我想的是示例5中应该返回300才对的呀。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这是因为底层的 reader 中总共只有 213 个字符啊，这与 buffer 的大小没有关系，要看有多少个字符是真正存在的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-28 11:56:49</div>
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
  <div class="_2_QraFYR_0">赫老师，您好。这段话，您写得是不是有问题，我读了好几遍也不能理解。能否详细讲解下：<br>“第二个事实，在压缩缓冲区之后，已写计数之后的字节只可能是已被读取过的字节，或者是已被拷贝到缓冲区头部的未读字节，又或者是代表未曾被填入数据的零值0x00。所以，后续的新字节是可以被写到这些位置上的。”</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这里说的是，以“已写计数”这个指针为界线，在缓冲区经过压缩之后，它后面会是什么样的数据。<br>1. 在压缩之后，在该指针之后有可能是已经被读取过的旧数据。正因为这些数据在该指针指向的位置之后，所以压缩时并不会清除它们，等到写入新数据时自然也就被覆盖掉了；<br>2. 在压缩之后，在该指针之后的数据也可能是那些（由于压缩的进行）已经被拷贝到缓冲区头部的（还未被读取的）数据，这些数据同样可以被后来的新数据覆盖掉，因为缓冲区头部已经存在这些数据了；<br>3. 在压缩之后，在该指针之后的数据还可能是“初始状态”的，因为那些数据槽位还没有被用到过，所以它们才是“初始状态”，如果某个数据槽位被用过的话，就会是前两种情况中的任意一种了。<br><br>首先，你要理解前面说的缓冲区数据压缩是做什么的，会有哪些具体操作，然后再来理解这里的含义。简单来说，压缩其实就是，删掉那些在缓冲区头部的、已经被读取过的数据，然后把后边的未读数据拷贝到缓冲区的头部。你可以再好好理解一下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-07 09:14:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/a4/d2/c483b836.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Wiwen</span>
  </div>
  <div class="_2_QraFYR_0">如果Write方法发现需要写入的字节太多，同时缓冲区已空，直接写到底层写入器。<br>这个写入字节应该是新要写入的字节大小超过了缓存区的大小(默认值是4096)时，才直接写到底层写入器。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-17 21:48:37</div>
  </div>
</div>
</div>
</li>
</ul>