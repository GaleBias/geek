<audio title="43 _ bufio包中的数据类型（下）" src="https://static001.geekbang.org/resource/audio/14/0a/14b60f9568f1135273b78051726a240a.mp3" controls="controls"></audio> 
<p>你好，我是郝林，我今天继续分享bufio包中的数据类型。</p><p>在上一篇文章中，我提到了<code>bufio</code>包中的数据类型主要有<code>Reader</code>、<code>Scanner</code>、<code>Writer</code>和<code>ReadWriter</code>。并着重讲到了<code>bufio.Reader</code>类型与<code>bufio.Writer</code>类型，今天，我们继续专注<code>bufio.Reader</code>的内容来进行学习。</p><h2>知识扩展</h2><h3>问题 ：<code>bufio.Reader</code>类型读取方法有哪些不同？</h3><p><code>bufio.Reader</code>类型拥有很多用于读取数据的指针方法，<strong>这里面有4个方法可以作为不同读取流程的代表，它们是：<code>Peek</code>、<code>Read</code>、<code>ReadSlice</code>和<code>ReadBytes</code>。</strong></p><p><strong><code>Reader</code>值的<code>Peek</code>方法</strong>的功能是：读取并返回其缓冲区中的<code>n</code>个未读字节，并且它会从已读计数代表的索引位置开始读。</p><p>在缓冲区未被填满，并且其中的未读字节的数量小于<code>n</code>的时候，该方法就会调用<code>fill</code>方法，以启动缓冲区填充流程。但是，如果它发现上次填充缓冲区的时候有错误，那就不会再次填充。</p><p>如果调用方给定的<code>n</code>比缓冲区的长度还要大，或者缓冲区中未读字节的数量小于<code>n</code>，那么<code>Peek</code>方法就会把“所有未读字节组成的序列”作为第一个结果值返回。</p><p>同时，它通常还把“<code>bufio.ErrBufferFull</code>变量的值（以下简称缓冲区已满的错误）”<br>
作为第二个结果值返回，用来表示：虽然缓冲区被压缩和填满了，但是仍然满足不了要求。</p><!-- [[[read_end]]] --><p>只有在上述的情况都没有出现时，<code>Peek</code>方法才能返回：“以已读计数为起始的<code>n</code>个字节”和“表示未发生任何错误的<code>nil</code>”。</p><p><strong><code>bufio.Reader</code>类型的Peek方法有一个鲜明的特点，那就是：即使它读取了缓冲区中的数据，也不会更改已读计数的值。</strong></p><p>这个类型的其他读取方法并不是这样。就拿<strong>该类型的<code>Read</code>方法来说</strong>，它有时会把缓冲区中的未读字节，依次拷贝到其参数<code>p</code>代表的字节切片中，并立即根据实际拷贝的字节数增加已读计数的值。</p><ul>
<li>
<p>在缓冲区中还有未读字节的情况下，该方法的做法就是如此。不过，在另一些时候，其所属值的已读计数会等于已写计数，这表明：此时的缓冲区中已经没有任何未读的字节了。</p>
</li>
<li>
<p>当缓冲区中已无未读字节时，<code>Read</code>方法会先检查参数<code>p</code>的长度是否大于或等于缓冲区的长度。如果是，那么<code>Read</code>方法会索性放弃向缓冲区中填充数据，转而直接从其底层读取器中读出数据并拷贝到<code>p</code>中。这意味着它完全跨过了缓冲区，并直连了数据供需的双方。</p>
</li>
</ul><p>需要注意的是，<code>Peek</code>方法在遇到类似情况时的做法与这里的区别（这两种做法孰优孰劣还要看具体的使用场景）。</p><p><code>Peek</code>方法会在条件满足时填充缓冲区，并在发现参数<code>n</code>的值比缓冲区的长度更大时，直接返回缓冲区中的所有未读字节。</p><p>如果我们当初设定的缓冲区长度很大，那么在这种情况下的方法执行耗时，就有可能会比较长。最主要的原因是填充缓冲区需要花费较长的时间。</p><p>由<code>fill</code>方法执行的流程可知，它会尽量填满缓冲区中的可写空间。然而，<code>Read</code>方法在大多数的情况下，是不会向缓冲区中写入数据的，尤其是在前面描述的那种情况下，即：缓冲区中已无未读字节，且参数<code>p</code>的长度大于或等于缓冲区的长度。</p><p>此时，该方法会直接从底层读取器那里读出数据，所以数据的读出速度就成为了这种情况下方法执行耗时的决定性因素。</p><p>当然了，我在这里说的只是耗时操作在某些情况下更可能出现在哪里，一切的结论还是要以性能测试的客观结果为准。</p><p>说回<code>Read</code>方法的内部流程。如果缓冲区中已无未读字节，但其长度比参数<code>p</code>的长度更大，那么该方法会先把已读计数和已写计数的值都重置为<code>0</code>，然后再尝试着使用从底层读取器那里获取的数据，对缓冲区进行一次从头至尾的填充。</p><p>不过要注意，这里的尝试只会进行一次。无论在这一时刻是否能够获取到数据，也无论获取时是否有错误发生，都会是如此。而<code>fill</code>方法的做法与此不同，只要没有发生错误，它就会进行多次尝试，因此它真正获取到一些数据的可能性更大。</p><p>不过，这两个方法有一点是相同，那就是：只要它们把获取到的数据写入缓冲区，就会及时地更新已写计数的值。</p><p><strong>再来说<code>ReadSlice</code>方法和<code>ReadBytes</code>方法。</strong> 这两个方法的功能总体上来说，都是持续地读取数据，直至遇到调用方给定的分隔符为止。</p><p><strong><code>ReadSlice</code>方法</strong>会先在其缓冲区的未读部分中寻找分隔符。如果未能找到，并且缓冲区未满，那么该方法会先通过调用<code>fill</code>方法对缓冲区进行填充，然后再次寻找，如此往复。</p><p>如果在填充的过程中发生了错误，那么它会把缓冲区中的未读部分作为结果返回，同时返回相应的错误值。</p><p>注意，在这个过程中有可能会出现虽然缓冲区已被填满，但仍然没能找到分隔符的情况。</p><p>这时，<code>ReadSlice</code>方法会把整个缓冲区（也就是<code>buf</code>字段代表的字节切片）作为第一个结果值，并把缓冲区已满的错误（即<code>bufio.ErrBufferFull</code>变量的值）作为第二个结果值。</p><p>经过<code>fill</code>方法填满的缓冲区肯定从头至尾都只包含了未读的字节，所以这样做是合理的。</p><p>当然了，一旦<code>ReadSlice</code>方法找到了分隔符，它就会在缓冲区上切出相应的、包含分隔符的字节切片，并把该切片作为结果值返回。无论分隔符找到与否，该方法都会正确地设置已读计数的值。</p><p>比如，在返回缓冲区中的所有未读字节，或者代表全部缓冲区的字节切片之前，它会把已写计数的值赋给已读计数，以表明缓冲区中已无未读字节。</p><p>如果说<code>ReadSlice</code>是一个容易半途而废的方法的话，那么可以说<code>ReadBytes</code>方法算得上是相当的执着。</p><p><strong><code>ReadBytes</code>方法</strong>会通过调用<code>ReadSlice</code>方法一次又一次地从缓冲区中读取数据，直至找到分隔符为止。</p><p>在这个过程中，<code>ReadSlice</code>方法可能会因缓冲区已满而返回所有已读到的字节和相应的错误值，但<code>ReadBytes</code>方法总是会忽略掉这样的错误，并再次调用<code>ReadSlice</code>方法，这使得后者会继续填充缓冲区并在其中寻找分隔符。</p><p>除非<code>ReadSlice</code>方法返回的错误值并不代表缓冲区已满的错误，或者它找到了分隔符，否则这一过程永远不会结束。</p><p>如果寻找的过程结束了，不管是不是因为找到了分隔符，<code>ReadBytes</code>方法都会把在这个过程中读到的所有字节，按照读取的先后顺序组装成一个字节切片，并把它作为第一个结果值。如果过程结束是因为出现错误，那么它还会把拿到的错误值作为第二个结果值。</p><p>在<code>bufio.Reader</code>类型的众多读取方法中，依赖<code>ReadSlice</code>方法的除了<code>ReadBytes</code>方法，还有<code>ReadLine</code>方法。不过后者在读取流程上并没有什么特别之处，我就不在这里赘述了。</p><p>另外，该类型的<code>ReadString</code>方法完全依赖于<code>ReadBytes</code>方法，前者只是在后者返回的结果值之上做了一个简单的类型转换而已。</p><p><strong>最后，我还要提醒你一下，有个安全性方面的问题需要你注意。<code>bufio.Reader</code>类型的<code>Peek</code>方法、<code>ReadSlice</code>方法和<code>ReadLine</code>方法都有可能会造成内容泄露。</strong></p><p>这主要是因为它们在正常的情况下都会返回直接基于缓冲区的字节切片。我在讲<code>bytes.Buffer</code>类型的时候解释过什么叫内容泄露。你可以返回查看。</p><p>调用方可以通过这些方法返回的结果值访问到缓冲区的其他部分，甚至修改缓冲区中的内容。这通常都是很危险的。</p><h2>总结</h2><p>我们用比较长的篇幅介绍了<code>bufio</code>包中的数据类型，其中的重点是<code>bufio.Reader</code>类型。</p><p><code>bufio.Reader</code>类型代表的是携带缓冲区的读取器。它的值在被初始化的时候需要接受一个底层的读取器，后者的类型必须是<code>io.Reader</code>接口的实现。</p><p><code>Reader</code>值中的缓冲区其实就是一个数据存储中介，它介于底层读取器与读取方法及其调用方之间。此类值的读取方法一般都会先从该值的缓冲区中读取数据，同时在必要的时候预先从其底层读取器那里读出一部分数据，并填充到缓冲区中以备后用。填充缓冲区的操作通常会由该值的<code>fill</code>方法执行。在填充的过程中，<code>fill</code>方法有时还会对缓冲区进行压缩。</p><p>在<code>Reader</code>值拥有的众多读取方法中，有4个方法可以作为不同读取流程的代表，它们是：<code>Peek</code>、<code>Read</code>、<code>ReadSlice</code>和<code>ReadBytes</code>。</p><p><code>Peek</code>方法的特点是即使读取了缓冲区中的数据，也不会更改已读计数的值。而<code>Read</code>方法会在参数值的长度过大，且缓冲区中已无未读字节时，跨过缓冲区并直接向底层读取器索要数据。</p><p><code>ReadSlice</code>方法会在缓冲区的未读部分中寻找给定的分隔符，并在必要时对缓冲区进行填充。</p><p>如果在填满缓冲区之后仍然未能找到分隔符，那么该方法就会把整个缓冲区作为第一个结果值返回，同时返回缓冲区已满的错误。</p><p><code>ReadBytes</code>方法会通过调用<code>ReadSlice</code>方法，一次又一次地填充缓冲区，并在其中寻找分隔符。除非发生了未预料到的错误或者找到了分隔符，否则这一过程将会一直进行下去。</p><p><code>Reader</code>值的<code>ReadLine</code>方法会依赖于它的<code>ReadSlice</code>方法，而其<code>ReadString</code>方法则完全依赖于<code>ReadBytes</code>方法。</p><p>另外，值得我们特别注意的是，<code>Reader</code>值的<code>Peek</code>方法、<code>ReadSlice</code>方法和<code>ReadLine</code>方法都可能会造成其缓冲区中的内容的泄露。</p><p>最后再说一下<code>bufio.Writer</code>类型。把该类值的缓冲区中暂存的数据写进其底层写入器的功能，主要是由它的<code>Flush</code>方法实现的。</p><p>此类值的所有数据写入方法都会在必要的时候调用它的<code>Flush</code>方法。一般情况下，这些写入方法都会先把数据写进其所属值的缓冲区，然后再增加该值中的已写计数。但是，在有些时候，<code>Write</code>方法和<code>ReadFrom</code>方法也会跨过缓冲区，并直接把数据写进其底层写入器。</p><p>请记住，虽然这些写入方法都会不时地调用<code>Flush</code>方法，但是在写入所有的数据之后再显式地调用一下这个方法总是最稳妥的。</p><h2>思考题</h2><p>今天的思考题是：<code>bufio.Scanner</code>类型的主要功用是什么？它有哪些特点？</p><p>感谢你的收听，我们下期再见。</p><p><a href="https://github.com/hyper0x/Golang_Puzzlers">戳此查看Go语言专栏文章配套详细代码。</a></p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/80/fb/ffa57759.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>xyz</span>
  </div>
  <div class="_2_QraFYR_0">慢慢追上了（不过有些内容学的比较粗略……），感觉郝林老师的这个系列，特别是后面的内容更适合作为一个了解常用库的索引存在，有在实际工作中碰到了问题再来参考会更好理解。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-20 22:20:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/43/e2/a1ff289c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>wang jl</span>
  </div>
  <div class="_2_QraFYR_0">我就想看bufio.Scanner才买的这个教程😓结果是思考题</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-11 20:23:20</div>
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
  <div class="_2_QraFYR_0">可以把 io.Reader 或 strings.NewReader 想像成一个实际的存在于硬盘之上的文件IO对象，你要对这个文件进行读写操作，感觉这样比较生动具体，可以更好的理解bufio包的功能用途，bufio包可以有效的降低系统IO的读写次数，从而提高了程序的性能。 XD</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-08 23:00:34</div>
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
  <div class="_2_QraFYR_0">bufio.Scanner 通过一个分隔函数，循环读取底层读取器中的内容，每次读取一段，使用Scanner.Scan() 读取， 通过Scanner.Text() 或 Scanner.Bytes() 获取读到的内容</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-20 13:11:09</div>
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
  <div class="_2_QraFYR_0">好吧，看了源码， writer2.ReadFrom 只有缓冲区为0才会跨过缓冲区</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-11 19:08:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/5e/4f/56ad818d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张剑</span>
  </div>
  <div class="_2_QraFYR_0">先学一遍 有个整体的分类和粗略了解 以后有用到还可回来找到</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-19 19:57:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/37/92/961ba560.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>授人以🐟，不如授人以渔</span>
  </div>
  <div class="_2_QraFYR_0">郝老师，我通过阅读这部分的源代码感受到源代码作者的用心良苦，以及对 I&#47;O 读写控制的精细。我的疑问时：对于方法返回值的用途，以及在设计 API 时，特别是方法的返回值都有哪些需要注意的地方。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 简单说，如果是单独的API，那么起码需要明确体现返回值的含义，以及在出现错误或异常情况下的具体行为。<br><br>如果是成体系的API，那就要遵循完整统一的规范了。这个规范可以根据项目的实际情况、团队的既有习惯和风格自己定义，比如：返回值文档的写法、返回值的命名方法、多返回值的次序、多返回值的联合赋值方式（如“若err不为nil则ret必为0”之类的）等等。<br><br>制定这类规范的时候可以多参考Go标准库中的代码和文档，努力做到自有规范与官方规范的基本一致性。这样做对跨团队协作会更加友好。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-17 16:07:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/37/92/961ba560.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>授人以🐟，不如授人以渔</span>
  </div>
  <div class="_2_QraFYR_0">「bufio.Reader类型的 Peek 方法有一个鲜明的特点，那就是：即使它读取了缓冲区中的数据，也不会更改已读计数的值。」这里需要说明的是 Peek 方法可能会修改 b.r 和 b.w 的值，但是对于 Peek 方法来说，其含义是并非是一种“真实”的读取，意味着接下来再用 ReadByte 还能读取到相同的数据。比如用下面的示例程序可以验证：<br><br>func main() {<br>	buf := bytes.NewBufferString(&quot;abcd&quot;)<br>	reader := bufio.NewReader(buf)<br><br>	slice, err := reader.Peek(4)<br>	if err != nil {<br>		log.Fatal(err)<br>	}<br>	fmt.Printf(&quot;%v.\n&quot;, slice)<br><br>	value, err := reader.ReadByte()<br>	fmt.Printf(&quot;%v.\n&quot;, value)<br>}</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-17 15:48:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/17/96/a10524f5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>计算机漫游</span>
  </div>
  <div class="_2_QraFYR_0">“ ReadSlice方法会先在其缓冲区的未读部分中寻找分隔符。如果未能找到，并且缓冲区未满，那么该方法会先通过调用fill方法对缓冲区进行填充，然后再次寻找，如此往复。”  虽然通过后续内容了解了ReadSlice方法只填满一次缓冲区，但是这里上下文中的 “如此反复” 一词容易让人产生和readbyte功能一样的误解。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: “填充”和“填满”是有区别的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-21 09:56:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKELX1Rd1vmLRWibHib8P95NA87F4zcj8GrHKYQL2RcLDVnxNy1ia2geTWgW6L2pWn2kazrPNZMRVrIg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jxs1211</span>
  </div>
  <div class="_2_QraFYR_0">请教一个读代码是遇到的疑问，collectFragments方法中fullBuffers并没有提前声明，这样也可添加buff吗吗，另外注释中说这种方式可以减少调用时的内存开销和数据拷贝，是指这里分片拷贝后一次填充到一个切片中返回，就不用外部调用者自己去组装的意思吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 第一个问题，fullBuffers 在 collectFragments 方法中代表了一个有名的结果啊。<br><br>它的声明已经包含在该方法的签名中了：<br><br>  collectFragments(delim byte) (fullBuffers [][]byte, finalFragment []byte, totalLen int, err error)<br><br>函数或方法签名中的有名结果会在该函数或方法被调用时自动地创建并初始化，其初始化值会是其类型的零值。<br><br>第二个问题，你说的对。<br><br>collectFragments 方法其实是在做二次缓存，不论读取成功还是出错，都先用“（动态）结构化”的方式把扫描过的数据缓存下来，但把“成败”的判定权留给调用它的代码。更确切地说，是留给 collectFragments 方法的调用方的调用方。<br><br>这相当于：“我”在内部把比较繁杂以及容易造成（空间或时间）浪费的那部分操作先做了，正所谓“内置了最佳实践”，至于操作结果的解读“你们”自己做吧。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-09 23:09:02</div>
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
  <div class="_2_QraFYR_0">大家一定要看看老师写的示例代码，写的非常用心了。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-29 18:25:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/4d/79/803537db.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>慢动作</span>
  </div>
  <div class="_2_QraFYR_0">这几节感觉直接看api或源码就好，没有什么印象深刻的地方</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 那祝贺你，说明你已经熟知这些知识点了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-14 15:02:03</div>
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
  <div class="_2_QraFYR_0">basicWriter3 := &amp;strings.Builder{}<br>reader2 := strings.NewReader(&quot;1234456789&quot;)<br>writer2 := bufio.NewWriterSize(basicWriter3,3)<br>writer2.WriteString(&quot;abcd&quot;)<br>&#47;&#47;此时缓冲区中还有数据 d, basicWriter3的数据为 ”abc“<br>writer2.ReadFrom(reader2) &#47;&#47;这一步不经过缓冲区<br>&#47;&#47;basicWriter3.String()  数据理论上就应该是 abc123456789<br>&#47;&#47;实际我打印出来的数据为 abcd1234567<br>writer2.Flush()<br>&#47;&#47;应该是  abc123456789d 才对，但是实际打印出来并不是这样的<br><br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-11 19:06:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/89/aa/eb4c37db.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>John</span>
  </div>
  <div class="_2_QraFYR_0">打卡</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-15 11:21:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/90/02/ba6c8c73.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jimmy</span>
  </div>
  <div class="_2_QraFYR_0">坚持，加油</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-05 21:58:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/44/fc/1ca9c88c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>曾春云</span>
  </div>
  <div class="_2_QraFYR_0">每天坚持一课</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-26 23:46:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/06/4f/de1e5a54.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Zero</span>
  </div>
  <div class="_2_QraFYR_0">坚持</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-08 09:16:52</div>
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
    <div class="_3Hkula0k_0">2019-03-13 01:24:56</div>
  </div>
</div>
</div>
</li>
</ul>