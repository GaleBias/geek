<audio title="45 _ 使用os包中的API （下）" src="https://static001.geekbang.org/resource/audio/41/25/413105a082fb333eb4f2f3ce080d2c25.mp3" controls="controls"></audio> 
<p>你好，我是郝林，今天我们继续分享使用os包中的API。</p><p>我们在上一篇文章中。从“<code>os.File</code>类型都实现了哪些<code>io</code>包中的接口”这一问题出发，介绍了一系列的相关内容。今天我们继续围绕这一知识点进行扩展。</p><h2>知识扩展</h2><h3>问题1：可应用于<code>File</code>值的操作模式都有哪些？</h3><p>针对<code>File</code>值的操作模式主要有只读模式、只写模式和读写模式。</p><p>这些模式分别由常量<code>os.O_RDONLY</code>、<code>os.O_WRONLY</code>和<code>os.O_RDWR</code>代表。在我们新建或打开一个文件的时候，必须把这三个模式中的一个设定为此文件的操作模式。</p><p>除此之外，我们还可以为这里的文件设置额外的操作模式，可选项如下所示。</p><ul>
<li><code>os.O_APPEND</code>：当向文件中写入内容时，把新内容追加到现有内容的后边。</li>
<li><code>os.O_CREATE</code>：当给定路径上的文件不存在时，创建一个新文件。</li>
<li><code>os.O_EXCL</code>：需要与<code>os.O_CREATE</code>一同使用，表示在给定的路径上不能有已存在的文件。</li>
<li><code>os.O_SYNC</code>：在打开的文件之上实施同步I/O。它会保证读写的内容总会与硬盘上的数据保持同步。</li>
<li><code>os.O_TRUNC</code>：如果文件已存在，并且是常规的文件，那么就先清空其中已经存在的任何内容。</li>
</ul><p>对于以上操作模式的使用，<code>os.Create</code>函数和<code>os.Open</code>函数都是现成的例子。</p><!-- [[[read_end]]] --><pre><code>func Create(name string) (*File, error) {
 return OpenFile(name, O_RDWR|O_CREATE|O_TRUNC, 0666)
}
</code></pre><p><code>os.Create</code>函数在调用<code>os.OpenFile</code>函数的时候，给予的操作模式是<code>os.O_RDWR</code>、<code>os.O_CREATE</code>和<code>os.O_TRUNC</code>的组合。</p><p>这就基本上决定了前者的行为，即：如果参数<code>name</code>代表路径之上的文件不存在，那么就新建一个，否则，先清空现存文件中的全部内容。</p><p>并且，它返回的<code>File</code>值的读取方法和写入方法都是可用的。这里需要注意，多个操作模式是通过按位或操作符<code>|</code>组合起来的。</p><p>func Open(name string) (*File, error) {<br>
return OpenFile(name, O_RDONLY, 0)<br>
}</p><p>我在前面说过，<code>os.Open</code>函数的功能是：以只读模式打开已经存在的文件。其根源就是它在调用<code>os.OpenFile</code>函数的时候，只提供了一个单一的操作模式<code>os.O_RDONLY</code>。</p><p>以上，就是我对可应用于<code>File</code>值的操作模式的简单解释。在demo88.go文件中还有少许示例，可供你参考。</p><h3>问题2：怎样设定常规文件的访问权限？</h3><p>我们已经知道，<code>os.OpenFile</code>函数的第三个参数<code>perm</code>代表的是权限模式，其类型是<code>os.FileMode</code>。但实际上，<code>os.FileMode</code>类型能够代表的，可远不只权限模式，它还可以代表文件模式（也可以称之为文件种类）。</p><p>由于<code>os.FileMode</code>是基于<code>uint32</code>类型的再定义类型，所以它的每个值都包含了32个比特位。在这32个比特位当中，每个比特位都有其特定的含义。</p><p>比如，如果在其最高比特位上的二进制数是<code>1</code>，那么该值表示的文件模式就等同于<code>os.ModeDir</code>，也就是说，相应的文件代表的是一个目录。</p><p>又比如，如果其中的第26个比特位上的是<code>1</code>，那么相应的值表示的文件模式就等同于<code>os.ModeNamedPipe</code>，也就是说，那个文件代表的是一个命名管道。</p><p>实际上，在一个<code>os.FileMode</code>类型的值（以下简称<code>FileMode</code>值）中，只有最低的9个比特位才用于表示文件的权限。当我们拿到一个此类型的值时，可以把它和<code>os.ModePerm</code>常量的值做按位与操作。</p><p>这个常量的值是<code>0777</code>，是一个八进制的无符号整数，其最低的9个比特位上都是<code>1</code>，而更高的23个比特位上都是<code>0</code>。</p><p>所以，经过这样的按位与操作之后，我们即可得到这个<code>FileMode</code>值中所有用于表示文件权限的比特位，也就是该值所表示的权限模式。这将会与我们调用<code>FileMode</code>值的<code>Perm</code>方法所得到的结果值是一致。</p><p>在这9个用于表示文件权限的比特位中，每3个比特位为一组，共可分为3组。</p><p><strong>从高到低，这3组分别表示的是文件所有者（也就是创建这个文件的那个用户）、文件所有者所属的用户组，以及其他用户对该文件的访问权限。而对于每个组，其中的3个比特位从高到低分别表示读权限、写权限和执行权限。</strong></p><p>如果在其中的某个比特位上的是<code>1</code>，那么就意味着相应的权限开启，否则，就表示相应的权限关闭。</p><p>因此，八进制整数<code>0777</code>就表示：操作系统中的所有用户都对当前的文件有读、写和执行的权限，而八进制整数<code>0666</code>则表示：所有用户都对当前文件有读和写的权限，但都没有执行的权限。</p><p>我们在调用<code>os.OpenFile</code>函数的时候，可以根据以上说明设置它的第三个参数。但要注意，只有在新建文件的时候，这里的第三个参数值才是有效的。在其他情况下，即使我们设置了此参数，也不会对目标文件产生任何的影响。</p><h2>总结</h2><p>为了聚焦于<code>os.File</code>类型本身，我在这两篇文章中主要讲述了怎样把os.File类型应用于常规的文件。该类型的指针类型实现了很多<code>io</code>包中的接口，因此它的具体功用也就可以不言自明了。</p><p>通过该类型的值，我们不但可以对文件进行各种读取、写入、关闭等操作，还可以设定下一次读取或写入时的起始索引位置。</p><p>在使用这个类型的值之前，我们必须先要创建它。所以，我为你重点介绍了几个可以创建，并获得此类型值的函数。</p><p>包括：<code>os.Create</code>、<code>os.NewFile</code>、<code>os.Open</code>和<code>os.OpenFile</code>。我们用什么样的方式创建<code>File</code>值，就决定了我们可以使用它来做什么。</p><p>利用<code>os.Create</code>函数，我们可以在操作系统中创建一个全新的文件，或者清空一个现存文件中的全部内容并重用它。</p><p>在相应的<code>File</code>值之上，我们可以对该文件进行任何的读写操作。虽然<code>os.NewFile</code>函数并不是被用来创建新文件的，但是它能够基于一个有效的文件描述符包装出一个可用的<code>File</code>值。</p><p><code>os.Open</code>函数的功能是打开一个已经存在的文件。但是，我们只能通过它返回的<code>File</code>值对相应的文件进行读操作。</p><p><code>os.OpenFile</code>是这些函数中最为灵活的一个，通过它，我们可以设定被打开文件的操作模式和权限模式。实际上，<code>os.Create</code>函数和<code>os.Open</code>函数都只是对它的简单封装而已。</p><p>在使用<code>os.OpenFile</code>函数的时候，我们必须要搞清楚操作模式和权限模式所代表的真正含义，以及设定它们的正确方式。</p><p>我在本文的扩展问题中分别对它们进行了较为详细的解释。同时，我在对应的示例文件中也编写了一些代码。</p><p>你需要认真地阅读和理解这些代码，并在运行它们的过程当中悟出这两种模式的真谛。</p><p>我在本文中讲述的东西对于<code>os</code>包来说，只是海面上的那部分冰山而已。这个代码包囊括的知识众多，而且延展性都很强。</p><p>如果你想完全理解它们，可能还需要去参看操作系统等方面的文档和教程。由于篇幅原因，我在这里只是做了一个引导，帮助你初识该包中的一些重要的程序实体，并给予你一个可以深入下去的切入点，希望你已经在路上了。</p><p><strong>思考题</strong></p><p>今天的思考题是：怎样通过<code>os</code>包中的API创建和操纵一个系统进程？</p><p><a href="https://github.com/hyper0x/Golang_Puzzlers">戳此查看Go语言专栏文章配套详细代码。</a></p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/80/b5/f59d92f1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Cloud</span>
  </div>
  <div class="_2_QraFYR_0">func Syscall</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-27 19:06:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/86/fb/4add1a52.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>兵戈</span>
  </div>
  <div class="_2_QraFYR_0">思考题：怎样通过os包中的 API 创建和操纵一个系统进程？<br>个人思路如下：<br>1. os 包及其子包 os&#47;exec 提供了创建进程的方法<br>2. os&#47;proc.go 提供了不少获取进程属性的方法</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-10 10:26:27</div>
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
  <div class="_2_QraFYR_0">操作模式，限定了可以通过*File执行的操作<br>权限模式，对应操作系统上的文件权限</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-23 08:33:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/ba/89/fc2aad72.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>心静梵音</span>
  </div>
  <div class="_2_QraFYR_0">郝大大，咱们os&#47;exec和os&#47;signal包还会讲嘛？我看咱们的课程介绍上列了，是不是在其他讲讲过了？<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这次不讲了，已经超出太多了，而且我觉得从重要性来讲这两个包稍逊，而且也不复杂，我书里也有讲，没必要再搞一套相似的讲解。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-30 08:30:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/5c/f6/56d77359.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>SamuraiDeng</span>
  </div>
  <div class="_2_QraFYR_0">权限，看的不是很懂，但是，我感觉跟Linux给文件加权限应该是一个出处</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这里的权限其实就是基于操作系统来做的，各种表示方法也基本一致，只不过通过API的方式暴露了出来。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-18 21:09:10</div>
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
  <div class="_2_QraFYR_0">怎样创建系统进程？通过cmd的api可以运行系统命令，其底层是系统调用fork和execv家族函数</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-23 07:48:52</div>
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
    <div class="_3Hkula0k_0">2019-03-15 00:21:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/b7/f9/75bae002.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>manky</span>
  </div>
  <div class="_2_QraFYR_0">跟linux文件访问规则差不多</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-23 16:31:17</div>
  </div>
</div>
</div>
</li>
</ul>