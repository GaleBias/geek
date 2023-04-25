<audio title="01 _ 工作区和GOPATH" src="https://static001.geekbang.org/resource/audio/28/07/28558270d89996af78b58fb5ce15f807.mp3" controls="controls"></audio> 
<blockquote>
<p>这门课中Go语言的代码比较多，建议你配合文章收听音频。</p>
</blockquote><p>你好，我是郝林。从今天开始，我将和你一起梳理Go语言的整个知识体系。</p><p>在过去的几年里，我与广大爱好者一起见证了Go语言的崛起。</p><p>从Go 1.5版本的自举（即用Go语言编写程序来实现Go语言自身），到Go 1.7版本的极速GC（也称垃圾回收器），再到2018年2月发布的Go 1.10版本对其自带工具的全面升级，以及可预见的后续版本关键特性（比如用来做程序依赖管理的<code>go mod</code>命令），这一切都令我们欢欣鼓舞。Go语言在一步步走向辉煌的同时，显然已经成为软件工程师们最喜爱的编程语言之一。</p><p>我开办这个专栏的主要目的，是要与你一起探索Go语言的奥秘，并帮助你在学习和实践的过程中获取更多。</p><p>我假设本专栏的读者已经具备了一定的计算机基础，比如，你要知道操作系统是什么、环境变量怎么设置、怎样正确使用命令行，等等。</p><p>当然了，如果你已经有了编程经验，尤其是一点点Go语言编程经验，那就更好了，毕竟我想教给你的，都是Go语言中非常核心的技术。</p><p>如果你对Go语言中最基本的概念和语法还不够了解，那么可能需要在学习本专栏的过程中去查阅<a href="https://golang.google.cn/ref/spec">Go语言规范文档</a>，也可以把预习篇的基础知识图拿出来好好研究一下。</p><!-- [[[read_end]]] --><p>最后，我来说一下专栏的讲述模式。我总会以一道Go语言的面试题开始，针对它进行解答，我会告诉你为什么我要关注这道题，这道题的背后隐藏着哪些知识，并且，我会对这部分的内容，进行相关的知识扩展。</p><p>好了，准备就绪，我们一起开始。</p><hr><p>我们学习Go语言时，要做的第一件事，都是根据自己电脑的计算架构（比如，是32位的计算机还是64位的计算机）以及操作系统（比如，是Windows还是Linux），从<a href="https://golang.google.cn">Go语言官网</a>下载对应的二进制包，也就是可以拿来即用的安装包。</p><p>随后，我们会解压缩安装包、放置到某个目录、配置环境变量，并通过在命令行中输入<code>go version</code>来验证是否安装成功。</p><p>在这个过程中，我们还需要配置3个环境变量，也就是GOROOT、GOPATH和GOBIN。这里我可以简单介绍一下。</p><ul>
<li>GOROOT：Go语言安装根目录的路径，也就是GO语言的安装路径。</li>
<li>GOPATH：若干工作区目录的路径。是我们自己定义的工作空间。</li>
<li>GOBIN：GO程序生成的可执行文件（executable file）的路径。</li>
</ul><p>其中，GOPATH背后的概念是最多的，也是最重要的。那么，<strong>今天我们的面试问题是：你知道设置GOPATH有什么意义吗？</strong></p><p>关于这个问题，它的<strong>典型回答</strong>是这样的：</p><p>你可以把GOPATH简单理解成Go语言的工作目录，它的值是一个目录的路径，也可以是多个目录路径，每个目录都代表Go语言的一个工作区（workspace）。</p><p>我们需要利用这些工作区，去放置Go语言的源码文件（source file），以及安装（install）后的归档文件（archive file，也就是以“.a”为扩展名的文件）和可执行文件（executable file）。</p><p>事实上，由于Go语言项目在其生命周期内的所有操作（编码、依赖管理、构建、测试、安装等）基本上都是围绕着GOPATH和工作区进行的。所以，它的背后至少有3个知识点，分别是：</p><p><strong>1. Go语言源码的组织方式是怎样的；</strong></p><p><strong>2.你是否了解源码安装后的结果（只有在安装后，Go语言源码才能被我们或其他代码使用）；</strong></p><p><strong>3.你是否理解构建和安装Go程序的过程（这在开发程序以及查找程序问题的时候都很有用，否则你很可能会走弯路）。</strong></p><p>下面我就重点来聊一聊这些内容。</p><h2>知识扩展</h2><h2>1. Go语言源码的组织方式</h2><p>与许多编程语言一样，Go语言的源码也是以代码包为基本组织单位的。在文件系统中，这些代码包其实是与目录一一对应的。由于目录可以有子目录，所以代码包也可以有子包。</p><p>一个代码包中可以包含任意个以.go为扩展名的源码文件，这些源码文件都需要被声明属于同一个代码包。</p><p>代码包的名称一般会与源码文件所在的目录同名。如果不同名，那么在构建、安装的过程中会以代码包名称为准。</p><p>每个代码包都会有导入路径。代码包的导入路径是其他代码在使用该包中的程序实体时，需要引入的路径。在实际使用程序实体之前，我们必须先导入其所在的代码包。具体的方式就是<code>import</code>该代码包的导入路径。就像这样：</p><pre><code>import &quot;github.com/labstack/echo&quot;
</code></pre><p>在工作区中，一个代码包的导入路径实际上就是从src子目录，到该包的实际存储位置的相对路径。</p><p>所以说，Go语言源码的组织方式就是以环境变量GOPATH、工作区、src目录和代码包为主线的。一般情况下，Go语言的源码文件都需要被存放在环境变量GOPATH包含的某个工作区（目录）中的src目录下的某个代码包（目录）中。</p><h2>2. 了解源码安装后的结果</h2><p>了解了Go语言源码的组织方式后，我们很有必要知道Go语言源码在安装后会产生怎样的结果。</p><p>源码文件以及安装后的结果文件都会放到哪里呢？我们都知道，源码文件通常会被放在某个工作区的src子目录下。</p><p>那么在安装后如果产生了归档文件（以“.a”为扩展名的文件），就会放进该工作区的pkg子目录；如果产生了可执行文件，就可能会放进该工作区的bin子目录。</p><p>我再讲一下归档文件存放的具体位置和规则。</p><p>源码文件会以代码包的形式组织起来，一个代码包其实就对应一个目录。安装某个代码包而产生的归档文件是与这个代码包同名的。</p><p>放置它的相对目录就是该代码包的导入路径的直接父级。比如，一个已存在的代码包的导入路径是</p><pre><code>github.com/labstack/echo
</code></pre><p>那么执行命令</p><pre><code>go install github.com/labstack/echo
</code></pre><p>生成的归档文件的相对目录就是 github.com/labstack ，文件名为echo.a 。</p><p>顺便说一下，上面这个代码包导入路径还有另外一层含义，那就是：该代码包的源码文件存在于GitHub网站的labstack组的代码仓库echo中。</p><p>再说回来，归档文件的相对目录与pkg目录之间还有一级目录，叫做平台相关目录。平台相关目录的名称是由build（也称“构建”）的目标操作系统、下划线和目标计算架构的代号组成的。</p><p>比如，构建某个代码包时的目标操作系统是Linux，目标计算架构是64位的，那么对应的平台相关目录就是linux_amd64。</p><p>因此，上述代码包的归档文件就会被放置在当前工作区的子目录pkg/linux_amd64/github.com/labstack中。</p><p><img src="https://static001.geekbang.org/resource/image/2f/3c/2fdfb5620e072d864907870e61ae5f3c.png?wh=1472*797" alt=""><br>
（GOPATH与工作区）</p><p>总之，你需要记住的是，某个工作区的src子目录下的源码文件在安装后一般会被放置到当前工作区的pkg子目录下对应的目录中，或者被直接放置到该工作区的bin子目录中。</p><h2>3. 理解构建和安装Go程序的过程</h2><p>我们再来说说构建和安装Go程序的过程都是怎样的，以及它们的异同点。</p><p>构建使用命令<code>go build</code>，安装使用命令<code>go install</code>。构建和安装代码包的时候都会执行编译、打包等操作，并且，这些操作生成的任何文件都会先被保存到某个临时的目录中。</p><p>如果构建的是库源码文件，那么操作后产生的结果文件只会存在于临时目录中。这里的构建的主要意义在于检查和验证。</p><p>如果构建的是命令源码文件，那么操作的结果文件会被搬运到源码文件所在的目录中。（这里讲到的两种源码文件我在<a href="https://time.geekbang.org/column/article/13540?utm_source=weibo&utm_medium=xuxiaoping&utm_campaign=promotion&utm_content=columns">“预习篇”的基础知识图</a>中提到过，在后面的文章中我也会带你详细了解。）</p><p>安装操作会先执行构建，然后还会进行链接操作，并且把结果文件搬运到指定目录。</p><p>进一步说，如果安装的是库源码文件，那么结果文件会被搬运到它所在工作区的pkg目录下的某个子目录中。</p><p>如果安装的是命令源码文件，那么结果文件会被搬运到它所在工作区的bin目录中，或者环境变量<code>GOBIN</code>指向的目录中。</p><p>这里你需要记住的是，构建和安装的不同之处，以及执行相应命令后得到的结果文件都会出现在哪里。</p><h2>总结</h2><p>工作区和GOPATH的概念和含义是每个Go工程师都需要了解的。虽然它们都比较简单，但是说它们是Go程序开发的核心知识并不为过。</p><p>然而，我在招聘面试的过程中仍然发现有人忽略掉了它们。Go语言提供的很多工具都是在GOPATH和工作区的基础上运行的，比如上面提到的<code>go build</code>、<code>go install</code>和<code>go get</code>，这三个命令也是我们最常用到的。</p><h2>思考题</h2><p>说到Go程序中的依赖管理，其实还有很多问题值得我们探索。我在这里留下两个问题供你进一步思考。</p><ol>
<li>Go语言在多个工作区中查找依赖包的时候是以怎样的顺序进行的？</li>
<li>如果在多个工作区中都存在导入路径相同的代码包会产生冲突吗？</li>
</ol><p>这两个问题之间其实是有一些关联的。答案并不复杂，你做几个试验几乎就可以找到它了。你也可以看一下Go语言标准库中<code>go build</code>包及其子包的源码。那里面的宝藏也很多，可以助你深刻理解Go程序的构建过程。</p><hr><h2>补充阅读</h2><h2>go build命令一些可选项的用途和用法</h2><p>在运行<code>go build</code>命令的时候，默认不会编译目标代码包所依赖的那些代码包。当然，如果被依赖的代码包的归档文件不存在，或者源码文件有了变化，那它还是会被编译。</p><p>如果要强制编译它们，可以在执行命令的时候加入标记<code>-a</code>。此时，不但目标代码包总是会被编译，它依赖的代码包也总会被编译，即使依赖的是标准库中的代码包也是如此。</p><p>另外，如果不但要编译依赖的代码包，还要安装它们的归档文件，那么可以加入标记<code>-i</code>。</p><p>那么我们怎么确定哪些代码包被编译了呢？有两种方法。</p><ol>
<li>运行<code>go build</code>命令时加入标记<code>-x</code>，这样可以看到<code>go build</code>命令具体都执行了哪些操作。另外也可以加入标记<code>-n</code>，这样可以只查看具体操作而不执行它们。</li>
<li>运行<code>go build</code>命令时加入标记<code>-v</code>，这样可以看到<code>go build</code>命令编译的代码包的名称。它在与<code>-a</code>标记搭配使用时很有用。</li>
</ol><p>下面再说一说与Go源码的安装联系很紧密的一个命令：<code>go get</code>。</p><p>命令<code>go get</code>会自动从一些主流公用代码仓库（比如GitHub）下载目标代码包，并把它们安装到环境变量<code>GOPATH</code>包含的第1工作区的相应目录中。如果存在环境变量<code>GOBIN</code>，那么仅包含命令源码文件的代码包会被安装到<code>GOBIN</code>指向的那个目录。</p><p>最常用的几个标记有下面几种。</p><ul>
<li><code>-u</code>：下载并安装代码包，不论工作区中是否已存在它们。</li>
<li><code>-d</code>：只下载代码包，不安装代码包。</li>
<li><code>-fix</code>：在下载代码包后先运行一个用于根据当前Go语言版本修正代码的工具，然后再安装代码包。</li>
<li><code>-t</code>：同时下载测试所需的代码包。</li>
<li><code>-insecure</code>：允许通过非安全的网络协议下载和安装代码包。HTTP就是这样的协议。</li>
</ul><p>Go语言官方提供的<code>go get</code>命令是比较基础的，其中并没有提供依赖管理的功能。目前GitHub上有很多提供这类功能的第三方工具，比如<code>glide</code>、<code>gb</code>以及官方出品的<code>dep</code>、<code>vgo</code>等等，它们在内部大都会直接使用<code>go get</code>。</p><p>有时候，我们可能会出于某种目的变更存储源码的代码仓库或者代码包的相对路径。这时，为了让代码包的远程导入路径不受此类变更的影响，我们会使用自定义的代码包导入路径。</p><p>对代码包的远程导入路径进行自定义的方法是：在该代码包中的库源码文件的包声明语句的右边加入导入注释，像这样：</p><pre><code>package semaphore // import &quot;golang.org/x/sync/semaphore&quot;
</code></pre><p>这个代码包原本的完整导入路径是<code>github.com/golang/sync/semaphore</code>。这与实际存储它的网络地址对应的。该代码包的源码实际存在GitHub网站的golang组的sync代码仓库的semaphore目录下。而加入导入注释之后，用以下命令即可下载并安装该代码包了：</p><pre><code>go get golang.org/x/sync/semaphore
</code></pre><p>而Go语言官网golang.org下的路径/x/sync/semaphore并不是存放<code>semaphore</code>包的真实地址。我们称之为代码包的自定义导入路径。</p><p>不过，这还需要在golang.org这个域名背后的服务端程序上，添加一些支持才能使这条命令成功。</p><p>关于自定义代码包导入路径的完整说明可以参看<a href="https://github.com/hyper0x/go_command_tutorial/blob/master/0.3.md">这里</a>。</p><p>好了，对于<code>go build</code>命令和<code>go get</code>命令的简短介绍就到这里。如果你想查阅更详细的文档，那么可以访问Go语言官方的<a href="https://golang.google.cn/cmd/go">命令文档页面</a>，或者在命令行下输入诸如<code>go help build</code>这类的命令。</p><p><a href="https://github.com/hyper0x/Golang_Puzzlers">戳此查看Go语言专栏文章配套详细代码。</a></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/aa/53/768aec0a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>郝林</span>
  </div>
  <div class="_2_QraFYR_0">有很多读写问归档文件是什么意思。归档文件在Linux下就是扩展名是.a的文件，也就是archive文件。写过C程序的朋友都知道，这是程序编译后生成的静态库文件。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-10 11:42:31</div>
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
  <div class="_2_QraFYR_0">下午上班时间随便读了一下,感觉有点讲的太散,只吸收了20%,晚上专门花了时间精读几遍 吸收了100%后真的干货满满,以前不懂得原理都能知道了 这是网上博客不会有的工匠级解说</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-27 00:07:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/5f/bb/cd23de6c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>全干工程师</span>
  </div>
  <div class="_2_QraFYR_0">什么是归档文件，归档文件里都是什么？有什么作用。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 请看我置顶的留言。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-10 00:53:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/d3/9f/36ea3be4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>千年孤独</span>
  </div>
  <div class="_2_QraFYR_0">如果在面试中让老师来回答“设置&#39;GOPATH有什么意义？”这个问题，除去典型回答<br><br>老师会如何简要明了回答这个问题？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以说，为了集中组织代码，以及代码互相引用。当然了，这么说后面试官可能还会让你具体解答。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-10 21:55:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4e/3b/c5ae689a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>许明</span>
  </div>
  <div class="_2_QraFYR_0">ide 我觉得vscode就很好用了，我现在是vscode + glide</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，我也用这种组合。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-10 22:57:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/53/2d/8bc7a3a6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>第五季节</span>
  </div>
  <div class="_2_QraFYR_0">工作区是指： 放go的源码目录。<br>gopath是指：指向工作区的路径。<br><br>归档文件： 相当于java的jar包。下载到本地私服<br><br><br>不知道对不对。纯属个人观点。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 前两个对，归档文件在Linux下就是.a文件，也就是archive文件，是程序编译成功后生成的静态库文件。这跟Java的jar包还不太一样，jar包相当于动态链接库了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-10 11:37:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/21/b8/aca814dd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>江山如画</span>
  </div>
  <div class="_2_QraFYR_0">对于初学者第一次看确实有些难懂，但是多看几遍后就会发现其实干货满满，我读了好几遍，接触golang也快一年了，但是很多知识点是第一次接触到，感谢郝林老师！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对你有帮助就好。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-14 20:38:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/04/81/fe2797cf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>xyang</span>
  </div>
  <div class="_2_QraFYR_0">go语言适合做什么业务，能概述性的罗列讲述下吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你好，我会在后边另写文章介绍。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-11 09:10:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/56/ea/32608c44.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>giteebravo</span>
  </div>
  <div class="_2_QraFYR_0">一脸懵逼，并不知道归档文件是啥😄</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 请看我置顶的留言。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-10 07:30:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://wx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTK1iadgQFxhYdu7wIUf7n5XYZlchNicdGBsafY9GPX3hNq0313DfE7ia6CeRm7VZAmwGPsLI8icTJUqXg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jians</span>
  </div>
  <div class="_2_QraFYR_0">看完再结合测试后的疑问：<br>在不同项目src中有同名包时，go build, install只会执行gopath中最早发现包的工作区，哪如何编译后面其他工作区中的同名包呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这就需要自己去放置了，或者临时把前面的工作区从gopath中去掉。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-10 17:49:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/8e/5c/f34849ac.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>白宇</span>
  </div>
  <div class="_2_QraFYR_0">请教一下，如何解决下载第三方包失败情况</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你好，这属于是网络问题。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-11 17:58:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/53/2d/8bc7a3a6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>第五季节</span>
  </div>
  <div class="_2_QraFYR_0">gopath 的设计类似java  。<br>具体的用途是执行命令要用 例如：go run、go install、go get等。<br>允许设置多个路径。<br>在idea里面的多个project或工具组建都并列放在gopath的src下面。<br>例如：go install myProject1<br>            go install myProject2 <br>所以老师说的这个归档文件。可以理解成工作空间吗？<br>至于老师说的两个问题。<br>1:按照代码的执行顺序 从上往下 每个引入的初始化。<br>2:引入一下试试就知道了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 归档文件的解释请看置顶的留言。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-10 10:43:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/42/7c/8ef14715.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>🄽🄸🅇🅄🅂</span>
  </div>
  <div class="_2_QraFYR_0">归档文件，可以理解为程序的动态链接库文件吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 相当于静态链接文件。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-12 22:39:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/62/54/cd487e91.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>有风的林子</span>
  </div>
  <div class="_2_QraFYR_0">目前还没用到GOPATH包含多个工作区，不知多个目录间的分隔符是什么？空格、冒号、还是分号？如果作者顺便说一下就好了，至少增加一个知识点。😁</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你好，这根据具体的操作系统而定。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-12 15:14:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/10/e8/172b5915.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张珂</span>
  </div>
  <div class="_2_QraFYR_0">突然出现“库源码文件”和“命令源码文件”，只看名字不知道是什么。看了某个留言之后才知道“命令源码文件”是包含main入口函数的源码文件。虽说该专栏定位不是基础教学，但还是要照顾一下大家基础不一，阅读门槛太高也不好。<br><br>总体还感觉文字组织让人读起来很费劲，文字逻辑需要优化。<br><br>图好像也很少。文不如表，表不如图，图不如动画。有些东西一图胜千言。其他专栏都有很多图的，虽说音频很不方便，但我觉得音频对于方法论的讲述比较合适（比如架构的思想），不适合该专栏。大部分用户应该是用“阅读”来学习的。<br><br>个人观点</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-19 19:25:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/fb/44/8b2600fd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>咖啡色的羊驼</span>
  </div>
  <div class="_2_QraFYR_0">对于作者的两个问题。三个纬度延伸总结回答下:<br><br>1.总执行顺序的角度<br>引入的包 -&gt; 当前包的变量常量 -&gt; init()[多个同一包则按照顺序执行] -&gt; main函数<br><br>2.依赖包执行顺序<br>被依赖的总是优先执行初始化，一个包只会被初始化一次。<br>a引入b，b引入c，则执行顺序c -&gt; b -&gt; a<br><br>3.单个包执行顺序的角度<br>总的前提:按照包中源文件名的字典顺序来排序执行。<br>当前包排序后的变量常量 -&gt; 排序后的init()</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-10 01:31:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/47/bd/8b3c7bcb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Bob.Chen</span>
  </div>
  <div class="_2_QraFYR_0">请问对于之前不太了解go的同学，是否有推荐的入门书可以和专栏配合着看？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-10 09:23:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/9f/1d/ec173090.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>melon</span>
  </div>
  <div class="_2_QraFYR_0">感觉一脸懵逼的，建议先看一下官网这篇深入浅出的文章:https:&#47;&#47;golang.org&#47;doc&#47;code.html，看完再回过头来看就了然了。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-29 17:12:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/58/c4/b28bafc8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>乌慢</span>
  </div>
  <div class="_2_QraFYR_0">对于新手来说看起来不是那么的通俗易懂，如果加上图解就更好了，希望以后更加好</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-11 17:11:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/68/30/2de62b89.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>少年</span>
  </div>
  <div class="_2_QraFYR_0">希望老师少用写陌生的概念，比如：命令源码文件。如果用，请加点解释和类比。用过javascript，php，python，感觉这篇看的还是一脸萌比。命令源码文件这词，还是在留言区看明白的。可以参照廖雪峰老师的文笔，傻比都能看明白。只是提供些反馈，希望老师的专栏越做越好。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-19 09:05:28</div>
  </div>
</div>
</div>
</li>
</ul>