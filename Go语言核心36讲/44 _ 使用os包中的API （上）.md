<audio title="44 _ 使用os包中的API （上）" src="https://static001.geekbang.org/resource/audio/a1/0b/a1215371bd81a30f92e1a0711db47e0b.mp3" controls="controls"></audio> 
<p>我们今天要讲的是<code>os</code>代码包中的API。这个代码包可以让我们拥有操控计算机操作系统的能力。</p><h2>前导内容：os包中的API</h2><p>这个代码包提供的都是平台不相关的API。那么说，什么叫平台不相关的API呢？</p><p>它的意思是：这些API基于（或者说抽象自）操作系统，为我们使用操作系统的功能提供高层次的支持，但是，它们并不依赖于具体的操作系统。</p><p>不论是Linux、macOS、Windows，还是FreeBSD、OpenBSD、Plan9，<code>os</code>代码包都可以为之提供统一的使用接口。这使得我们可以用同样的方式，来操纵不同的操作系统，并得到相似的结果。</p><p><code>os</code>包中的API主要可以帮助我们使用操作系统中的文件系统、权限系统、环境变量、系统进程以及系统信号。</p><p>其中，操纵文件系统的API最为丰富。我们不但可以利用这些API创建和删除文件以及目录，还可以获取到它们的各种信息、修改它们的内容、改变它们的访问权限，等等。</p><p>说到这里，就不得不提及一个非常常用的数据类型：<code>os.File</code>。</p><p>从字面上来看，<code>os.File</code>类型代表了操作系统中的文件。但实际上，它可以代表的远不止于此。或许你已经知道，对于类Unix的操作系统（包括Linux、macOS、FreeBSD等），其中的一切都可以被看做是文件。</p><!-- [[[read_end]]] --><p>除了文本文件、二进制文件、压缩文件、目录这些常见的形式之外，还有符号链接、各种物理设备（包括内置或外接的面向块或者字符的设备）、命名管道，以及套接字（也就是socket），等等。</p><p>因此，可以说，我们能够利用<code>os.File</code>类型操纵的东西太多了。不过，为了聚焦于<code>os.File</code>本身，同时也为了让本文讲述的内容更加通用，我们在这里主要把<code>os.File</code>类型应用于常规的文件。</p><p>下面这个问题，就是以<code>os.File</code>类型代表的最基本内容入手。<strong>我们今天的问题是：<code>os.File</code>类型都实现了哪些<code>io</code>包中的接口？</strong></p><p>这道题的<strong>典型回答</strong>是这样的。</p><p><code>os.File</code>类型拥有的都是指针方法，所以除了空接口之外，它本身没有实现任何接口。而它的指针类型则实现了很多<code>io</code>代码包中的接口。</p><p>首先，对于<code>io</code>包中最核心的3个简单接口<code>io.Reader</code>、<code>io.Writer</code>和<code>io.Closer</code>，<code>*os.File</code>类型都实现了它们。</p><p>其次，该类型还实现了另外的3个简单接口，即：<code>io.ReaderAt</code>、<code>io.Seeker</code>和<code>io.WriterAt</code>。</p><p>正是因为<code>*os.File</code>类型实现了这些简单接口，所以它也顺便实现了<code>io</code>包的9个扩展接口中的7个。</p><p>然而，由于它并没有实现简单接口<code>io.ByteReader</code>和<code>io.RuneReader</code>，所以它没有实现分别作为这两者的扩展接口的<code>io.ByteScanner</code>和<code>io.RuneScanner</code>。</p><p>总之，<code>os.File</code>类型及其指针类型的值，不但可以通过各种方式读取和写入某个文件中的内容，还可以寻找并设定下一次读取或写入时的起始索引位置，另外还可以随时对文件进行关闭。</p><p>但是，它们并不能专门地读取文件中的下一个字节，或者下一个Unicode字符，也不能进行任何的读回退操作。</p><p>不过，单独读取下一个字节或字符的功能也可以通过其他方式来实现，比如，调用它的<code>Read</code>方法并传入适当的参数值就可以做到这一点。</p><h2>问题解析</h2><p><span class="orange">这个问题其实在间接地问“<code>os.File</code>类型能够以何种方式操作文件？”我在前面的典型回答中也给出了简要的答案。</span></p><p>在我进一步地说明一些细节之前，我们先来看看，怎样才能获得一个<code>os.File</code>类型的指针值（以下简称<code>File</code>值）。</p><p>在<code>os</code>包中，有这样几个函数，即：<code>Create</code>、<code>NewFile</code>、<code>Open</code>和<code>OpenFile</code>。</p><p><strong><code>os.Create</code>函数用于根据给定的路径创建一个新的文件。</strong> 它会返回一个<code>File</code>值和一个错误值。我们可以在该函数返回的<code>File</code>值之上，对相应的文件进行读操作和写操作。</p><p>不但如此，我们使用这个函数创建的文件，对于操作系统中的所有用户来说，都是可以读和写的。</p><p>换句话说，一旦这样的文件被创建出来，任何能够登录其所属的操作系统的用户，都可以在任意时刻读取该文件中的内容，或者向该文件写入内容。</p><p>注意，如果在我们给予<code>os.Create</code>函数的路径之上，已经存在了一个文件，那么该函数会先清空现有文件中的全部内容，然后再把它作为第一个结果值返回。</p><p>另外，<code>os.Create</code>函数是有可能返回非<code>nil</code>的错误值的。</p><p>比如，如果我们给定的路径上的某一级父目录并不存在，那么该函数就会返回一个<code>*os.PathError</code>类型的错误值，以表示“不存在的文件或目录”。</p><p><strong>再来看<code>os.NewFile</code>函数。</strong> 该函数在被调用的时候，需要接受一个代表文件描述符的、<code>uintptr</code>类型的值，以及一个用于表示文件名的字符串值。</p><p>如果我们给定的文件描述符并不是有效的，那么这个函数将会返回<code>nil</code>，否则，它将会返回一个代表了相应文件的<code>File</code>值。</p><p>注意，不要被这个函数的名称误导了，它的功能并不是创建一个新的文件，而是依据一个已经存在的文件的描述符，来新建一个包装了该文件的<code>File</code>值。</p><p>例如，我们可以像这样拿到一个包装了标准错误输出的<code>File</code>值：</p><pre><code>file3 := os.NewFile(uintptr(syscall.Stderr), &quot;/dev/stderr&quot;)
</code></pre><p>然后，通过这个<code>File</code>值向标准错误输出上写入一些内容：</p><pre><code>if file3 != nil {
 defer file3.Close()
 file3.WriteString(
  &quot;The Go language program writes the contents into stderr.\n&quot;)
}
</code></pre><p><strong><code>os.Open</code>函数会打开一个文件并返回包装了该文件的<code>File</code>值。</strong> 然而，该函数只能以只读模式打开文件。换句话说，我们只能从该函数返回的<code>File</code>值中读取内容，而不能向它写入任何内容。</p><p>如果我们调用了这个<code>File</code>值的任何一个写入方法，那么都将会得到一个表示了“坏的文件描述符”的错误值。实际上，我们刚刚说的只读模式，正是应用在<code>File</code>值所持有的文件描述符之上的。</p><p>所谓的文件描述符，是由通常很小的非负整数代表的。它一般会由I/O相关的系统调用返回，并作为某个文件的一个标识存在。</p><p>从操作系统的层面看，针对任何文件的I/O操作都需要用到这个文件描述符。只不过，Go语言中的一些数据类型，为我们隐匿掉了这个描述符，如此一来我们就无需时刻关注和辨别它了（就像<code>os.File</code>类型这样）。</p><p>实际上，我们在调用前文所述的<code>os.Create</code>函数、<code>os.Open</code>函数以及将会提到的<code>os.OpenFile</code>函数的时候，它们都会执行同一个系统调用，并且在成功之后得到这样一个文件描述符。这个文件描述符将会被储存在它们返回的<code>File</code>值中。</p><p><code>os.File</code>类型有一个指针方法，名叫<code>Fd</code>。它在被调用之后将会返回一个<code>uintptr</code>类型的值。这个值就代表了当前的<code>File</code>值所持有的那个文件描述符。</p><p>不过，在<code>os</code>包中，除了<code>NewFile</code>函数需要用到它，它也没有什么别的用武之地了。所以，如果你操作的只是常规的文件或者目录，那么就无需特别地在意它了。</p><p><strong>最后，再说一下<code>os.OpenFile</code>函数。</strong> 这个函数其实是<code>os.Create</code>函数和<code>os.Open</code>函数的底层支持，它最为灵活。</p><p>这个函数有3个参数，分别名为<code>name</code>、<code>flag</code>和<code>perm</code>。其中的<code>name</code>指代的就是文件的路径。而<code>flag</code>参数指的则是需要施加在文件描述符之上的模式，我在前面提到的只读模式就是这里的一个可选项。</p><p>在Go语言中，这个只读模式由常量<code>os.O_RDONLY</code>代表，它是<code>int</code>类型的。当然了，这里除了只读模式之外，还有几个别的模式可选，我们稍后再细说。</p><p><code>os.OpenFile</code>函数的参数<code>perm</code>代表的也是模式，它的类型是<code>os.FileMode</code>，此类型是一个基于<code>uint32</code>类型的再定义类型。</p><p>为了加以区别，我们把参数<code>flag</code>指代的模式叫做操作模式，而把参数<code>perm</code>指代的模式叫做权限模式。可以这么说，操作模式限定了操作文件的方式，而权限模式则可以控制文件的访问权限。关于权限模式的更多细节我们将在后面讨论。</p><p><img src="https://static001.geekbang.org/resource/image/d3/93/d3414376a3343926a2b33cdeeb094893.png?wh=1920*831" alt=""><br>
（获得os.File类型的指针值的几种方式）</p><p>到这里，你需要记住的是，通过<code>os.File</code>类型的值，我们不但可以对文件进行读取、写入、关闭等操作，还可以设定下一次读取或写入时的起始索引位置。</p><p>此外，<code>os</code>包中还有用于创建全新文件的<code>Create</code>函数，用于包装现存文件的<code>NewFile</code>函数，以及可被用来打开已存在的文件的<code>Open</code>函数和<code>OpenFile</code>函数。</p><h2>总结</h2><p>我们今天讲的是<code>os</code>代码包以及其中的程序实体。我们首先讨论了<code>os</code>包存在的意义，和它的主要用途。代码包中所包含的API，都是对操作系统的某方面功能的高层次抽象，这使得我们可以通过它以统一的方式，操纵不同的操作系统，并得到相似的结果。</p><p>在这个代码包中，操纵文件系统的API最为丰富，最有代表性的就是数据类型<code>os.File</code>。<code>os.File</code>类型不但可以代表操作系统中的文件，还可以代表很多其他的东西。尤其是在类Unix的操作系统中，它几乎可以代表一切可以操纵的软件和硬件。</p><p>在下一期的文章中，我会继续讲解os包中的API的内容。如果你对这部分的知识有什么问题，可以给我留言，感谢你的收听，我们下期再见。</p><p><a href="https://github.com/hyper0x/Golang_Puzzlers">戳此查看Go语言专栏文章配套详细代码。</a></p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/48/97/520f3511.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Walking In The Air</span>
  </div>
  <div class="_2_QraFYR_0">最希望老师把net包内极相关的包讲解一下，这部分用的最频繁，但是总有一种似懂非懂的感觉，只是知道是这样用，不知道为什么，对底层知识不清晰，没有一个轮廓</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-22 11:06:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/2d/36/d3c8d272.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>HF</span>
  </div>
  <div class="_2_QraFYR_0">老师，高级语言的标准库实现方式有哪些？用到的系统服务是封装系统调用还是用系统库函数</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 通常都是两者兼而有之的。Go语言也是这样，但是在它实现自举之后，它已经尽可能地进行直接的系统调用了。<br><br>至于其他高级语言怎样做，就要看它们的理念以及是否有能力足够“贴近”操作系统了。<br><br>绕过C语言库，直接连接最底层，是需要足够的实力的，包括对操作系统本身足够熟悉等等。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-26 17:57:04</div>
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
  <div class="_2_QraFYR_0">郝林老师能简单说一下demo87.go 中的 ：reflect.TypeOf((*io.ReadWriteSeeker)(nil)).Elem() 运作流程吗？ 感觉这种写法还挺特别的。<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. (*io.ReadWriteSeeker)(nil) 是为了得到一个 *io.ReadWriteSeeker 类型的值，但又不想实例化任何东西，所以右边括号里的是 nil ；<br>2. 通过反射得到这个 nil 值的类型，一个指针类型（如这里的 *io.ReadWriteSeeker）；<br>3. 调用 Type 类型的 Elem 方法，取出这个指针类型指向的那个类型，即那个接口类型（如这里的 io.ReadWriteSeeker）。<br><br>这是一个小技巧，可以获取任意接口的 Type 值。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-30 11:45:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/7c/e5/3dca2495.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>上山的o牛</span>
  </div>
  <div class="_2_QraFYR_0">同求net包讲解</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-12 10:10:33</div>
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
  <div class="_2_QraFYR_0">郝林老师，demo87.go 样例中好像少了这一段 关闭文件的代码：<br><br>defer file3.Close()</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这只是一个示例程序，一瞬间就运行结束了，所以没必要添加那么多defer，容易妨碍正题：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-30 16:58:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/ac/95/9b3e3859.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Timo</span>
  </div>
  <div class="_2_QraFYR_0">打卡</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-13 14:36:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/ce/b2/1f914527.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>海盗船长</span>
  </div>
  <div class="_2_QraFYR_0">打卡</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-17 22:47:50</div>
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
    <div class="_3Hkula0k_0">2019-03-13 22:20:04</div>
  </div>
</div>
</div>
</li>
</ul>