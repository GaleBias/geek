<audio title="38 _ bytes包与字节串操作（上）" src="https://static001.geekbang.org/resource/audio/ba/ba/ba50388910f70a3e9f48adde5e12dbba.mp3" controls="controls"></audio> 
<p>我相信，经过上一次的学习，你已经对<code>strings.Builder</code>和<code>strings.Reader</code>这两个类型足够熟悉了。</p><p>我上次还建议你去自行查阅<code>strings</code>代码包中的其他程序实体。如果你认真去看了，那么肯定会对我们今天要讨论的<code>bytes</code>代码包，有种似曾相识的感觉。</p><h2>前导内容： <code>bytes.Buffer</code>基础知识</h2><p><code>strings</code>包和<code>bytes</code>包可以说是一对孪生兄弟，它们在API方面非常的相似。单从它们提供的函数的数量和功能上讲，差别可以说是微乎其微。</p><p><strong>只不过，<code>strings</code>包主要面向的是Unicode字符和经过UTF-8编码的字符串，而<code>bytes</code>包面对的则主要是字节和字节切片。</strong></p><p>我今天会主要讲<code>bytes</code>包中最有特色的类型<code>Buffer</code>。顾名思义，<code>bytes.Buffer</code>类型的用途主要是作为字节序列的缓冲区。</p><p>与<code>strings.Builder</code>类型一样，<code>bytes.Buffer</code>也是开箱即用的。</p><p>但不同的是，<code>strings.Builder</code>只能拼接和导出字符串，而<code>bytes.Buffer</code>不但可以拼接、截断其中的字节序列，以各种形式导出其中的内容，还可以顺序地读取其中的子序列。</p><p>可以说，<code>bytes.Buffer</code>是集读、写功能于一身的数据类型。当然了，这些也基本上都是作为一个缓冲区应该拥有的功能。</p><!-- [[[read_end]]] --><p>在内部，<code>bytes.Buffer</code>类型同样是使用字节切片作为内容容器的。并且，与<code>strings.Reader</code>类型类似，<code>bytes.Buffer</code>有一个<code>int</code>类型的字段，用于代表已读字节的计数，可以简称为已读计数。</p><p>不过，这里的已读计数就无法通过<code>bytes.Buffer</code>提供的方法计算出来了。</p><p>我们先来看下面的代码：</p><pre><code>var buffer1 bytes.Buffer
contents := &quot;Simple byte buffer for marshaling data.&quot;
fmt.Printf(&quot;Writing contents %q ...\n&quot;, contents)
buffer1.WriteString(contents)
fmt.Printf(&quot;The length of buffer: %d\n&quot;, buffer1.Len())
fmt.Printf(&quot;The capacity of buffer: %d\n&quot;, buffer1.Cap())
</code></pre><p>我先声明了一个<code>bytes.Buffer</code>类型的变量<code>buffer1</code>，并写入了一个字符串。然后，我想打印出这个<code>bytes.Buffer</code>类型的值（以下简称<code>Buffer</code>值）的长度和容量。在运行这段代码之后，我们将会看到如下的输出：</p><pre><code>Writing contents &quot;Simple byte buffer for marshaling data.&quot; ...
The length of buffer: 39
The capacity of buffer: 64
</code></pre><p>乍一看这没什么问题。长度<code>39</code>和容量<code>64</code>的含义看起来与我们已知的概念是一致的。我向缓冲区中写入了一个长度为<code>39</code>的字符串，所以<code>buffer1</code>的长度就是<code>39</code>。</p><p>根据切片的自动扩容策略，<code>64</code>这个数字也是合理的。另外，可以想象，这时的已读计数的值应该是<code>0</code>，这是因为我还没有调用任何用于读取其中内容的方法。</p><p>可实际上，与<code>strings.Reader</code>类型的<code>Len</code>方法一样，<code>buffer1</code>的<code>Len</code>方法返回的也是内容容器中未被读取部分的长度，而不是其中已存内容的总长度（以下简称内容长度）。示例如下：</p><pre><code>p1 := make([]byte, 7)
n, _ := buffer1.Read(p1)
fmt.Printf(&quot;%d bytes were read. (call Read)\n&quot;, n)
fmt.Printf(&quot;The length of buffer: %d\n&quot;, buffer1.Len())
fmt.Printf(&quot;The capacity of buffer: %d\n&quot;, buffer1.Cap())
</code></pre><p>当我从<code>buffer1</code>中读取一部分内容，并用它们填满长度为<code>7</code>的字节切片<code>p1</code>之后，<code>buffer1</code>的<code>Len</code>方法返回的结果值也会随即发生变化。如果运行这段代码，我们会发现，这个缓冲区的长度已经变为了<code>32</code>。</p><p>另外，因为我们并没有再向该缓冲区中写入任何内容，所以它的容量会保持不变，仍是<code>64</code>。</p><p><strong>总之，在这里，你需要记住的是，<code>Buffer</code>值的长度是未读内容的长度，而不是已存内容的总长度。</strong> 它与在当前值之上的读操作和写操作都有关系，并会随着这两种操作的进行而改变，它可能会变得更小，也可能会变得更大。</p><p>而<code>Buffer</code>值的容量指的是它的内容容器（也就是那个字节切片）的容量，它只与在当前值之上的写操作有关，并会随着内容的写入而不断增长。</p><p>再说已读计数。由于<code>strings.Reader</code>还有一个<code>Size</code>方法可以给出内容长度的值，所以我们用内容长度减去未读部分的长度，就可以很方便地得到它的已读计数。</p><p>然而，<code>bytes.Buffer</code>类型却没有这样一个方法，它只有<code>Cap</code>方法。可是<code>Cap</code>方法提供的是内容容器的容量，也不是内容长度。</p><p>并且，这里的内容容器容量在很多时候都与内容长度不相同。因此，没有了现成的计算公式，只要遇到稍微复杂些的情况，我们就很难估算出<code>Buffer</code>值的已读计数。</p><p>一旦理解了已读计数这个概念，并且能够在读写的过程中，实时地获得已读计数和内容长度的值，我们就可以很直观地了解到当前<code>Buffer</code>值各种方法的行为了。不过，很可惜，这两个数字我们都无法直接拿到。</p><p>虽然，我们无法直接得到一个<code>Buffer</code>值的已读计数，并且有时候也很难估算它，但是我们绝对不能就此作罢，而应该通过研读<code>bytes.Buffer</code>和文档和源码，去探究已读计数在其中起到的关键作用。</p><p>否则，我们想用好<code>bytes.Buffer</code>的意愿，恐怕就不会那么容易实现了。</p><p>下面的这个问题，如果你认真地阅读了<code>bytes.Buffer</code>的源码之后，就可以很好地回答出来。</p><p><strong>我们今天的问题是：<code>bytes.Buffer</code>类型的值记录的已读计数，在其中起到了怎样的作用？</strong></p><p>这道题的典型回答是这样的。</p><p><code>bytes.Buffer</code>中的已读计数的大致功用如下所示。</p><ol>
<li>读取内容时，相应方法会依据已读计数找到未读部分，并在读取后更新计数。</li>
<li>写入内容时，如需扩容，相应方法会根据已读计数实现扩容策略。</li>
<li>截断内容时，相应方法截掉的是已读计数代表索引之后的未读部分。</li>
<li>读回退时，相应方法需要用已读计数记录回退点。</li>
<li>重置内容时，相应方法会把已读计数置为<code>0</code>。</li>
<li>导出内容时，相应方法只会导出已读计数代表的索引之后的未读部分。</li>
<li>获取长度时，相应方法会依据已读计数和内容容器的长度，计算未读部分的长度并返回。</li>
</ol><h2>问题解析</h2><p>通过上面的典型回答，我们已经能够体会到已读计数在<code>bytes.Buffer</code>类型，及其方法中的重要性了。没错，<code>bytes.Buffer</code>的绝大多数方法都用到了已读计数，而且都是非用不可。</p><p><strong>在读取内容的时候</strong>，相应方法会先根据已读计数，判断一下内容容器中是否还有未读的内容。如果有，那么它就会从已读计数代表的索引处开始读取。</p><p><strong>在读取完成后</strong>，它还会及时地更新已读计数。也就是说，它会记录一下又有多少个字节被读取了。<strong>这里所说的相应方法包括了所有名称以<code>Read</code>开头的方法，以及<code>Next</code>方法和<code>WriteTo</code>方法。</strong></p><p><strong>在写入内容的时候</strong>，绝大多数的相应方法都会先检查当前的内容容器，是否有足够的容量容纳新的内容。如果没有，那么它们就会对内容容器进行扩容。</p><p><strong>在扩容的时候</strong>，方法会在必要时，依据已读计数找到未读部分，并把其中的内容拷贝到扩容后内容容器的头部位置。</p><p>然后，方法将会把已读计数的值置为<code>0</code>，以表示下一次读取需要从内容容器的第一个字节开始。<strong>用于写入内容的相应方法，包括了所有名称以<code>Write</code>开头的方法，以及<code>ReadFrom</code>方法。</strong></p><p><strong>用于截断内容的方法<code>Truncate</code>，会让很多对<code>bytes.Buffer</code>不太了解的程序开发者迷惑。</strong> 它会接受一个<code>int</code>类型的参数，这个参数的值代表了：在截断时需要保留头部的多少个字节。</p><p>不过，需要注意的是，这里说的头部指的并不是内容容器的头部，而是其中的未读部分的头部。头部的起始索引正是由已读计数的值表示的。因此，在这种情况下，已读计数的值再加上参数值后得到的和，就是内容容器新的总长度。</p><p><strong>在<code>bytes.Buffer</code>中，用于读回退的方法有<code>UnreadByte</code>和<code>UnreadRune</code>。</strong> 这两个方法分别用于回退一个字节和回退一个Unicode字符。调用它们一般都是为了退回在上一次被读取内容末尾的那个分隔符，或者为重新读取前一个字节或字符做准备。</p><p>不过，退回的前提是，在调用它们之前的那一个操作必须是“读取”，并且是成功的读取，否则这些方法就只能忽略后续操作并返回一个非<code>nil</code>的错误值。</p><p><code>UnreadByte</code>方法的做法比较简单，把已读计数的值减<code>1</code>就好了。而<code>UnreadRune</code>方法需要从已读计数中减去的，是上一次被读取的Unicode字符所占用的字节数。</p><p>这个字节数由<code>bytes.Buffer</code>的另一个字段负责存储，它在这里的有效取值范围是[1, 4]。只有<code>ReadRune</code>方法才会把这个字段的值设定在此范围之内。</p><p>由此可见，只有紧接在调用<code>ReadRune</code>方法之后，对<code>UnreadRune</code>方法的调用才能够成功完成。该方法明显比<code>UnreadByte</code>方法的适用面更窄。</p><p>我在前面说过，<code>bytes.Buffer</code>的<code>Len</code>方法返回的是内容容器中未读部分的长度，而不是其中已存内容的总长度（即：内容长度）。</p><p>而该类型的<code>Bytes</code>方法和<code>String</code>方法的行为，与<code>Len</code>方法是保持一致的。前两个方法只会去访问未读部分中的内容，并返回相应的结果值。</p><p>在我们剖析了所有的相关方法之后，可以这样来总结：在已读计数代表的索引之前的那些内容，永远都是已经被读过的，它们几乎没有机会再次被读取。</p><p>不过，这些已读内容所在的内存空间可能会被存入新的内容。这一般都是由于重置或者扩充内容容器导致的。这时，已读计数一定会被置为<code>0</code>，从而再次指向内容容器中的第一个字节。这有时候也是为了避免内存分配和重用内存空间。</p><h2>总结</h2><p>总结一下，<code>bytes.Buffer</code>是一个集读、写功能于一身的数据类型。它非常适合作为字节序列的缓冲区。我们会在下一篇文章中继续对bytes.Buffer的知识进行延展。如果你对于这部分内容有什么样问题，欢迎给我留言，我们一起讨论。</p><p>感谢你的收听，我们下次再见。</p><p><a href="https://github.com/hyper0x/Golang_Puzzlers">戳此查看Go语言专栏文章配套详细代码。</a></p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/71/4c/2cefec07.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>静水流深</span>
  </div>
  <div class="_2_QraFYR_0">一图胜千言</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-07 07:41:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/a5/67/bf286335.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>AllenGFLiu</span>
  </div>
  <div class="_2_QraFYR_0">赫老師多次闡明源碼的重要性，但源碼內容的量確實不小。<br>不知道老師能不能給個閱讀源碼的路徑？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果想了解标准库中某个包的功能，可以从应用程序调用的函数看起，一步步深入。标准库是一张网，所以你在看的时候也需要织一张网，也就是说需要记好笔记，并且在必要的时候把那些点联结起来。这样才看得明白。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-24 11:49:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/KFgDEHIEpnSjjGClCeqmKYJsSOQo40BMHRTtNYrWyQP9WypAjTToplVND944one2pkEyH5Oib4m4wUOJ9xBEIZQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>sket</span>
  </div>
  <div class="_2_QraFYR_0">原来read把从缓冲区读的字节放到p1里面了，让我这个小白纠结了好半天</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-10 10:42:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/30/3c/0668d6ae.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>盘胧</span>
  </div>
  <div class="_2_QraFYR_0">花了2小时，一遍啃源码，一遍对照文章,一边自己写测试用例，一边搞官方包自己实现的用例。。。绝了，啃津津有味，不知不觉都天黑了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-23 18:26:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKOM6tVLSiciaQeQst0g3iboWO74ibicicVAia9qno0X6cf65pEKLgdKkUdcpCWpjAB5e6semrFrruiaGQWhg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>NoTryNoSuccess</span>
  </div>
  <div class="_2_QraFYR_0">坚持到这里了，给自己点赞！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-31 08:53:28</div>
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
  <div class="_2_QraFYR_0">郝林老师，以下问题，麻烦回答一下：<br><br>1.  「bytes.Buffer」 为什么不提供一个 Buffer值的已读计数值的获取方法呢？<br><br>2.  文中说： “在我们剖析了所有的相关方法之后，可以这样来总结：在已读计数代表的索引之前的那些内容，永远都是已经被读过的，它们几乎没有机会再次被读取”。我理解的是： 唯一有机会获取到这些内容的机会是 「UnreadByte」和 「UnreadRune」这两个方法吗？<br><br>3. 文中说：“这些已读内容所在的内存空间可能会被存入新的内容。这一般都是由于重置或者扩充内容容器导致的”。 已读的内容不是已经 复制给另外的变量返回给调用方了吗？为什么还会被被存入新的内容呢？没理解这句话的想表达的意思。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. 已读计数值属于内部状态，外界不需要知道啊。你想拿这个东西做什么用呢？该封装的就得封装起来，充分解耦。<br><br>2. 对的。<br><br>3. Buffer 在扩充自己时有可能会对数据进行压缩，也就是把未读的数据迁移到已读的那些元素槽位上。在这种情况下，如果经压缩后那些之前的“已读空间”中还有空位，那么新数据就可以存到那里。<br><br>另外，会导致“已读计数”变化的那些“读方法”读出去的是一个个数据元素（通过 copy 函数做的），而不是基于Buffer的切片。所以压缩数据并不会对已经读出去的那些数据造成影响。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-24 23:52:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/8c/e4/ad3e7c39.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>L.</span>
  </div>
  <div class="_2_QraFYR_0">老师同学们好，请问一下大家手头有能够辅助阅读源码的书籍或者资料吗，有的包下边的源码实在是理解不了，有的是不理解整体逻辑，有的是不理解某一行的写法等等(小白求助</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以进入我的知识星球“GoHackers VIP”然后向我提问（知识星球是一款App），当然也可以在这里向我提问。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-12 10:37:29</div>
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
  <div class="_2_QraFYR_0">我去阅读了源码 发现重要的不止是 off 我暂且叫他偏移量 还有一个重要的字段lastRead 标识上一次是否是成功读取 和读了多少个字节 所以UnreadRune方法应该还是可以正常使用吧(不包含ReadRune的情况) 毕竟底层是b.off -= int(b.lastRead) </div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-29 14:46:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/78/c7/083a3a0b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>新世界</span>
  </div>
  <div class="_2_QraFYR_0">每天坚持刷，讲的很不错</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-21 09:06:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/1d/7d/368df396.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>somenzz</span>
  </div>
  <div class="_2_QraFYR_0">无法直接得到一个Buffer值的已读计数？？<br>	n, _ := buffer1.Read(p1)<br>这个 n 不就是已读取的字节数吗？<br><br>老师还说 buffer1的Len方法返回的也是内容容器中未被读取部分的长度，我认为是错的，buffer1的Len方法返回就是 buffer1 内的长度，只要读取了，就不会存在 buffer1 里面了，难道不是吗？ 难道 buffer1 中还保留有已读取的字节吗？如果我错了，请老师给段代码证明下，buffer1 已读取的字节仍然可再次读取。 </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不要自己认为，要去看Go的源码。源码面前无秘密！<br><br>而且咱们专栏的代码例子中也有相应的说明：<br><br>https:&#47;&#47;github.com&#47;hyper0x&#47;Golang_Puzzlers&#47;tree&#47;master&#47;src&#47;puzzlers&#47;article31<br><br>再多说一点吧，Read() 方法返回的 n 只是当次读取的字节数。另外 Len 方法的源码是这样的：<br><br>func (b *Buffer) Len() int { return len(b.buf) - b.off }<br><br>这里的 b.off 代表的是读偏移量（相当于“读指针”），而 len(b.buf) 代表的是写偏移量（相当于“写指针”）。<br><br>另外，已读的那些字节并不会被马上删除掉，那样的话操作会太频繁（读时成本过高），它们是在写时发现需要增长缓存容量的时候才被删除掉的。<br><br>** 编程工作第一大忌讳：只自己想象，不看源码。切记切记！！**<br><br>最后，对所有本专栏的读者说一句：读这个专栏，越往后越需要随看随查源码。这样才能够有深刻的理解和认识。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-01 08:53:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/30/3c/0668d6ae.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>盘胧</span>
  </div>
  <div class="_2_QraFYR_0">学习源码以及使用的过程，按照作者写这个标准库的时候的思路：库函数 &gt; 结构定义 &gt; 结构函数<br>举例学习strings包：<br>1. 先熟悉外部库函数：go doc strings|grep &quot;^func&quot;<br>2. 熟悉对应的结构定义：go doc strings|grep &quot;^type&quot;|grep &quot;struct&quot;<br>3. 接下来就应该学习对应的内部结构的函数了 go doc strings.Builder|grep &quot;^func&quot;<br>可以使用思维导图，顺着函数的调用去看，把过程记录下来</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 很好：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-23 18:33:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/20/f2/ba/aad606a6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>南雁</span>
  </div>
  <div class="_2_QraFYR_0">我个人觉得老师讲的内容很不错，融入了他的经验和精华，但你觉得这种叙事方式我个人觉得还是不太适合新手的。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-28 06:06:06</div>
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
  <div class="_2_QraFYR_0">二刷<br>注意点：<br>String和Bytes方法不会更新内部计数器off</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-10 12:20:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/0b/80/a0533acb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>勇.Max</span>
  </div>
  <div class="_2_QraFYR_0">老师 请教下👇<br>golang如何实验solidity中的require和assert函数功能啊 试着写了下 没啥思路 求老师指点 谢谢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-08 16:16:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/0b/80/a0533acb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>勇.Max</span>
  </div>
  <div class="_2_QraFYR_0">老师，请教下👇<br>golang中如何实现类似solidity中的require(用来判断输入的参数是否符合某些条件)和assert(不只是对类型的断言)呢？<br>试着写了下，没思路，求老师指教。非常感谢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-08 16:14:46</div>
  </div>
</div>
</div>
</li>
</ul>