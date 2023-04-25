<audio title="35 _ OpenResty：更灵活的Web服务器" src="https://static001.geekbang.org/resource/audio/7d/68/7d395f67094c2bfa140ee2d100996168.mp3" controls="controls"></audio> 
<p>在上一讲里，我们看到了高性能的Web服务器Nginx，它资源占用少，处理能力高，是搭建网站的首选。</p><p>虽然Nginx成为了Web服务器领域无可争议的“王者”，但它也并不是没有缺点的，毕竟它已经15岁了。</p><p>“一个人很难超越时代，而时代却可以轻易超越所有人”，Nginx当初设计时针对的应用场景已经发生了变化，它的一些缺点也就暴露出来了。</p><p>Nginx的服务管理思路延续了当时的流行做法，使用磁盘上的静态配置文件，所以每次修改后必须重启才能生效。</p><p>这在业务频繁变动的时候是非常致命的（例如流行的微服务架构），特别是对于拥有成千上万台服务器的网站来说，仅仅增加或者删除一行配置就要分发、重启所有的机器，对运维是一个非常大的挑战，要耗费很多的时间和精力，成本很高，很不灵活，难以“随需应变”。</p><p>那么，有没有这样的一个Web服务器，它有Nginx的优点却没有Nginx的缺点，既轻量级、高性能，又灵活、可动态配置呢？</p><p>这就是我今天要说的OpenResty，它是一个“更好更灵活的Nginx”。</p><h2>OpenResty是什么？</h2><p>其实你对OpenResty并不陌生，这个专栏的实验环境就是用OpenResty搭建的，这么多节课程下来，你应该或多或少对它有了一些印象吧。</p><!-- [[[read_end]]] --><p>OpenResty诞生于2009年，到现在刚好满10周岁。它的创造者是当时就职于某宝的“神级”程序员<strong>章亦春</strong>，网名叫“agentzh”。</p><p>OpenResty并不是一个全新的Web服务器，而是基于Nginx，它利用了Nginx模块化、可扩展的特性，开发了一系列的增强模块，并把它们打包整合，形成了一个<strong>“一站式”的Web开发平台</strong>。</p><p>虽然OpenResty的核心是Nginx，但它又超越了Nginx，关键就在于其中的ngx_lua模块，把小巧灵活的Lua语言嵌入了Nginx，可以用脚本的方式操作Nginx内部的进程、多路复用、阶段式处理等各种构件。</p><p>脚本语言的好处你一定知道，它不需要编译，随写随执行，这就免去了C语言编写模块漫长的开发周期。而且OpenResty还把Lua自身的协程与Nginx的事件机制完美结合在一起，优雅地实现了许多其他语言所没有的“<strong>同步非阻塞</strong>”编程范式，能够轻松开发出高性能的Web应用。</p><p>目前OpenResty有两个分支，分别是开源、免费的“OpenResty”和闭源、商业产品的“OpenResty+”，运作方式有社区支持、OpenResty基金会、OpenResty.Inc公司，还有其他的一些外界赞助（例如Kong、CloudFlare），正在蓬勃发展。</p><p><img src="https://static001.geekbang.org/resource/image/9f/01/9f7b79c43c476890f03c2c716a20f301.png?wh=1142*481" alt="unpreview"></p><p>顺便说一下OpenResty的官方logo，是一只展翅飞翔的海鸥，选择海鸥是因为“鸥”与OpenResty的发音相同。另外，这个logo的形状也像是左手比出的一个“OK”姿势，正好也是一个“O”。</p><h2>动态的Lua</h2><p>刚才说了，OpenResty里的一个关键模块是ngx_lua，它为Nginx引入了脚本语言Lua。</p><p>Lua是一个比较“小众”的语言，虽然历史比较悠久，但名气却没有PHP、Python、JavaScript大，这主要与它的自身定位有关。</p><p><img src="https://static001.geekbang.org/resource/image/4f/d5/4f24aa3f53969b71baaf7d9c7cf68fd5.png?wh=1142*641" alt="unpreview"></p><p>Lua的设计目标是嵌入到其他应用程序里运行，为其他编程语言带来“脚本化”能力，所以它的“个头”比较小，功能集有限，不追求“大而全”，而是“小而美”，大多数时间都“隐匿”在其他应用程序的后面，是“无名英雄”。</p><p>你或许玩过或者听说过《魔兽世界》《愤怒的小鸟》吧，它们就在内部嵌入了Lua，使用Lua来调用底层接口，充当“胶水语言”（glue language），编写游戏逻辑脚本，提高开发效率。</p><p>OpenResty选择Lua作为“工作语言”也是基于同样的考虑。因为Nginx C开发实在是太麻烦了，限制了Nginx的真正实力。而Lua作为“最快的脚本语言”恰好可以成为Nginx的完美搭档，既可以简化开发，性能上又不会有太多的损耗。</p><p>作为脚本语言，Lua还有一个重要的“<strong>代码热加载</strong>”特性，不需要重启进程，就能够从磁盘、Redis或者任何其他地方加载数据，随时替换内存里的代码片段。这就带来了“<strong>动态配置</strong>”，让OpenResty能够永不停机，在微秒、毫秒级别实现配置和业务逻辑的实时更新，比起Nginx秒级的重启是一个极大的进步。</p><p>你可以看一下实验环境的“www/lua”目录，里面存放了我写的一些测试HTTP特性的Lua脚本，代码都非常简单易懂，就像是普通的英语“阅读理解”，这也是Lua的另一个优势：易学习、易上手。</p><h2>高效率的Lua</h2><p>OpenResty能够高效运行的一大“秘技”是它的“<strong>同步非阻塞</strong>”编程范式，如果你要开发OpenResty应用就必须时刻铭记于心。</p><p>“同步非阻塞”本质上还是一种“<strong>多路复用</strong>”，我拿上一讲的Nginx epoll来对比解释一下。</p><p>epoll是操作系统级别的“多路复用”，运行在内核空间。而OpenResty的“同步非阻塞”则是基于Lua内建的“<strong>协程</strong>”，是应用程序级别的“多路复用”，运行在用户空间，所以它的资源消耗要更少。</p><p>OpenResty里每一段Lua程序都由协程来调度运行。和Linux的epoll一样，每当可能发生阻塞的时候“协程”就会立刻切换出去，执行其他的程序。这样单个处理流程是“阻塞”的，但整个OpenResty却是“非阻塞的”，多个程序都“复用”在一个Lua虚拟机里运行。</p><p><img src="https://static001.geekbang.org/resource/image/9f/c6/9fc3df52df7d6c11aa02b8013f8e9bc6.png?wh=2159*909" alt=""></p><p>下面的代码是一个简单的例子，读取POST发送的body数据，然后再发回客户端：</p><pre><code>ngx.req.read_body()                  -- 同步非阻塞(1)

local data = ngx.req.get_body_data()
if data then
    ngx.print(&quot;body: &quot;, data)        -- 同步非阻塞(2)
end
</code></pre><p>代码中的“ngx.req.read_body”和“ngx.print”分别是数据的收发动作，只有收到数据才能发送数据，所以是“同步”的。</p><p>但即使因为网络原因没收到或者发不出去，OpenResty也不会在这里阻塞“干等着”，而是做个“记号”，把等待的这段CPU时间用来处理其他的请求，等网络可读或者可写时再“回来”接着运行。</p><p>假设收发数据的等待时间是10毫秒，而真正CPU处理的时间是0.1毫秒，那么OpenResty就可以在这10毫秒内同时处理100个请求，而不是把这100个请求阻塞排队，用1000毫秒来处理。</p><p>除了“同步非阻塞”，OpenResty还选用了<strong>LuaJIT</strong>作为Lua语言的“运行时（Runtime）”，进一步“挖潜增效”。</p><p>LuaJIT是一个高效的Lua虚拟机，支持JIT（Just In Time）技术，可以把Lua代码即时编译成“本地机器码”，这样就消除了脚本语言解释运行的劣势，让Lua脚本跑得和原生C代码一样快。</p><p>另外，LuaJIT还为Lua语言添加了一些特别的增强，比如二进制位运算库bit，内存优化库table，还有FFI（Foreign Function Interface），让Lua直接调用底层C函数，比原生的压栈调用快很多。</p><h2>阶段式处理</h2><p>和Nginx一样，OpenResty也使用“流水线”来处理HTTP请求，底层的运行基础是Nginx的“阶段式处理”，但它又有自己的特色。</p><p>Nginx的“流水线”是由一个个C模块组成的，只能在静态文件里配置，开发困难，配置麻烦（相对而言）。而OpenResty的“流水线”则是由一个个的Lua脚本组成的，不仅可以从磁盘上加载，也可以从Redis、MySQL里加载，而且编写、调试的过程非常方便快捷。</p><p>下面我画了一张图，列出了OpenResty的阶段，比起Nginx，OpenResty的阶段更注重对HTTP请求响应报文的加工和处理。</p><p><img src="https://static001.geekbang.org/resource/image/36/df/3689312a970bae0e949b017ad45438df.png?wh=1211*1437" alt=""></p><p>OpenResty里有几个阶段与Nginx是相同的，比如rewrite、access、content、filter，这些都是标准的HTTP处理。</p><p>在这几个阶段里可以用“xxx_by_lua”指令嵌入Lua代码，执行重定向跳转、访问控制、产生响应、负载均衡、过滤报文等功能。因为Lua的脚本语言特性，不用考虑内存分配、资源回收释放等底层的细节问题，可以专注于编写非常复杂的业务逻辑，比C模块的开发效率高很多，即易于扩展又易于维护。</p><p>OpenResty里还有两个不同于Nginx的特殊阶段。</p><p>一个是“<strong>init阶段</strong>”，它又分成“master init”和“worker init”，在master进程和worker进程启动的时候运行。这个阶段还没有开始提供服务，所以慢一点也没关系，可以调用一些阻塞的接口初始化服务器，比如读取磁盘、MySQL，加载黑白名单或者数据模型，然后放进共享内存里供运行时使用。</p><p>另一个是“<strong>ssl阶段</strong>”，这算得上是OpenResty的一大创举，可以在TLS握手时动态加载证书，或者发送“OCSP Stapling”。</p><p>还记得<a href="https://time.geekbang.org/column/article/111940">第29讲</a>里说的“SNI扩展”吗？Nginx可以依据“服务器名称指示”来选择证书实现HTTPS虚拟主机，但静态配置很不灵活，要编写很多雷同的配置块。虽然后来Nginx增加了变量支持，但它每次握手都要读磁盘，效率很低。</p><p>而在OpenResty里就可以使用指令“ssl_certificate_by_lua”，编写Lua脚本，读取SNI名字后，直接从共享内存或者Redis里获取证书。不仅没有读盘阻塞，而且证书也是完全动态可配置的，无需修改配置文件就能够轻松支持大量的HTTPS虚拟主机。</p><h2>小结</h2><ol>
<li><span class="orange">Nginx依赖于磁盘上的静态配置文件，修改后必须重启才能生效，缺乏灵活性；</span></li>
<li><span class="orange">OpenResty基于Nginx，打包了很多有用的模块和库，是一个高性能的Web开发平台；</span></li>
<li><span class="orange">OpenResty的工作语言是Lua，它小巧灵活，执行效率高，支持“代码热加载”；</span></li>
<li><span class="orange">OpenResty的核心编程范式是“同步非阻塞”，使用协程，不需要异步回调函数；</span></li>
<li><span class="orange">OpenResty也使用“阶段式处理”的工作模式，但因为在阶段里执行的都是Lua代码，所以非常灵活，配合Redis等外部数据库能够实现各种动态配置。</span></li>
</ol><h2>课下作业</h2><ol>
<li>谈一下这些天你对实验环境里OpenResty的感想和认识。</li>
<li>你觉得Nginx和OpenResty的“阶段式处理”有什么好处？对你的实际工作有没有启发？</li>
</ol><p>欢迎你把自己的学习体会写在留言区，与我和其他同学一起讨论。如果你觉得有所收获，也欢迎把文章分享给你的朋友。</p><p><img src="https://static001.geekbang.org/resource/image/c5/9f/c5b7ac40c585c800af0fe3ab98f3449f.png?wh=1769*4723" alt="unpreview"></p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/67/8a/babd74dc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>锦</span>
  </div>
  <div class="_2_QraFYR_0">老师好，多路复用理解起来有点困难，主语是什么呢？ 多路  复用分别怎么理解呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 英文是I&#47;O multiplexing，就是指多个I&#47;O请求“复用”到一个进程或线程里处理，而不是开多个进程、线程处理。<br><br>可以看示意图再体会一下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-16 20:09:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/75/aa/21275b9d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>闫飞</span>
  </div>
  <div class="_2_QraFYR_0">看起来OpenResty的核心武器是协程模型和Lua语言嵌入融合，合理照顾到了开发效率和程序执行效率之间的平衡。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-19 08:14:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/35/51/c616f95a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>阿锋</span>
  </div>
  <div class="_2_QraFYR_0">域名一般都是带www，也可以不带www，这两者有什么区别？www的作用是什么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: www是“主机名”，也就是表示这台主机用于提供万维网服务。<br><br>但现在大部分互联网上的主机都是http服务器，所以www现在只是一个“历史习惯”了，比如极客时间的网站“time.geekbang.org”，虽然不是www，但也是http服务。<br><br>不带www是域名的一种简化，通常会使用重定向跳转到其他的域名。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-17 16:39:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/79/4b/740f91ca.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>-W.LI-</span>
  </div>
  <div class="_2_QraFYR_0">老师好!<br>同步阻塞:代码同步顺序执行，等待阻塞操作完成继续往下走。<br>同步非阻塞:代码顺序执行，遇见阻塞操作时，CPU执行世间会让出去，得到结果时通过callBack继续回到之前阻塞的地方。<br>大概是这样么?<br>然后就是同步阻塞的话，在阻塞的时候会占用CPU执行时间么?<br>同步非阻塞的话，遇到阻塞操作，主线程直接让出CPU执行时间，上下文会切换么?上下文切换开销会很大吧，如果只是让出怎么实现阻塞数据没就绪时不被分配cpu，如果一直没回调这个线程会死锁么?<br>代码中请求，redis，数据库这些操作是同步阻塞，还是同步非阻塞?<br><br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.基本正确，但同步阻塞的时候是这个线程被阻塞了，操作系统会把这个线程切换出去干别的，不会耗cpu，相当于这个线程没有充分利用cpu给它的资源。<br><br>2.同步非阻塞，是线程自己主动切换cpu给其他任务，但并没有让出cpu给其他线程或进程，因为在用户态，所以成本低，底层是epoll和Nginx的事件机制。<br><br>3.有超时机制，超时就会任务执行失败，不会死锁。<br><br>4.OpenResty的代码都是同步非阻塞的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-16 10:33:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/30/8a/b5ca7286.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>业余草</span>
  </div>
  <div class="_2_QraFYR_0">老师不写OpenResty专栏亏才了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以看实体书《OpenResty完全开发指南》。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-23 08:55:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/a5/98/a65ff31a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>djfhchdh</span>
  </div>
  <div class="_2_QraFYR_0">2、“阶段式处理”，我的理解这个与“流水线”很像，许多的业务流程模型其实都可以抽象为流水线，通过配置化的方法，可以定制化地把各个模块组成业务流水线</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-16 18:19:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Aemon</span>
  </div>
  <div class="_2_QraFYR_0">nginx reload不影响应用吧？秒级是认真的吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我曾经见过一个Nginx实例重启，要加载几百几千个配置文件，那速度……</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-10 19:08:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/67/f4/9a1feb59.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>钱</span>
  </div>
  <div class="_2_QraFYR_0">同步非阻塞，是线程自己主动切换cpu给其他任务，但并没有让出cpu给其他线程或进程，因为在用户态，所以成本低，底层是epoll和Nginx的事件机制。<br><br>老师，这块没太明白，同步和非阻塞原本是矛盾的，一个大动作由多个小动作组成，如果其中一个小动作是一个慢动作，而且是同步模式，下面的动作必然会被阻塞住吧？<br>你上面解释说“线程自己主动切换CPU给其他任务”，<br>1：那线程什么时候主动切换CPU给其他任务？<br>2：这里的其他任务指什么？<br>3：线程主动切换CPU给其他任务后处于什么状态？为什么？<br>5：还有我的假设中慢动作后面的动作不是被阻塞了吗？<br>6：还是说维度与层次不同，同步非阻塞的主体是线程，而不是线程中的一系列动作？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.协程只是同步非阻塞的一种实现方式，两者不完全等同。<br><br>2.OpenResty里的线程其实是协程，内部有调度器，当有阻塞的时候就切换到其他的协程（任务）去执行。<br><br>3.协程主动yield后就出于暂停状态，可以随时切换回来继续执行，所以它自己是阻塞的，但整个程序不会因为一个协程而被阻塞。<br><br>4.可以参考其他语言里的协程概念来理解。记住协程是用户态的线程，而在操作系统来看实际上只要一个线程，所以如果有大量的磁盘io那么必然会阻塞。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-05 11:00:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4d/fd/0aa0e39f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>许童童</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，可以说一下OpenResty 和 nginx njs 有什么区别吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: OpenResty现在已经是一个成熟的Web开发生态体系了，已经有很多商业公司基于OpenResty开发各种业务应用，底层的LuaJIT性能很高，保证了它的运行效率。<br><br>njs现在还是处于起步阶段，功能比较弱，Nginx对它的定位是“可编程配置语言”，关注点还是在辅助Nginx，而不是用来开发复杂的业务逻辑。<br><br>还有很重要的一点是OpenResty里的LuaJIT支持FFI，可以直接调用C接口，扩展性极高，而njs这方面的能力为零，只能限制在vm里。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-16 11:52:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTIVR2wY9icec2CGzZ4VKPdwK2icytM5k1tHm08qSEysFOgl1y7lk2ccDqSCvzibHufo2Cb9c2hjr0LIg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>dahai</span>
  </div>
  <div class="_2_QraFYR_0">Nginx 的服务管理思路延续了当时的流行做法，使用磁盘上的静态配置文件，所以每次修改后必须重启才能生效。<br>Nginx 有reload 命令，只是不是自动reload。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: reload要读取磁盘文件，而配置文件复杂的话这就有一定的成本，而且重启进程的消耗也是不容忽视的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-06 16:11:56</div>
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
  <div class="_2_QraFYR_0">老师，既然OpenResty这么厉害，为什么现在大部分公司还是用的Nginx啊？我公司都有Lua程序员，但是Web服务器还是用的Nginx。是不是学习和运维成本都挺高的啊？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: OpenResty出来的比较晚，而且在最近几年才开始商业化推广，自然要比Nginx的市场份额要少一些，相关的资料也少，不过最近OpenResty的市场占有率已经开始快速增长了。<br><br>OpenResty其实就是Nginx加上一些非常有用的开发组件，本质上还是Nginx，但开发起来更方便。但因为它多了Lua等其他东西，功能已经不再是单纯的web server，所以用起来要复杂一点。如果是单纯搭建网站，没有二次开发的需求，自然很多人会出于简单的目的选择Nginx，但我觉得选择OpenResty会更好，更有潜力。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-27 16:57:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/79/4b/740f91ca.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>-W.LI-</span>
  </div>
  <div class="_2_QraFYR_0">老师好!看完回复好像明白了一点<br>同步非阻塞:nginx，是单线程模型，主线程类似一个多路复用器(和NIO的IO模型类似?)，所有的请求是以任务形式被受理，任务是交给协程程处理。任务结束，主线程检测到事件进行对应操作。主线程和协程一直都在处理任务，所以不会涉及到线程的上下文切换。传统的web服务器，Tomcat这些都是线程池形式的。一个请求交给一个线程，请求阻塞了这个线程就会被切换出去开销很大。nginx协程开销已经小了，又通过事件+异步非阻塞模型减少了上下文切换所以吞吐量就能很大。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 基本正确。不过Nginx里没有使用协程，它使用的是epoll的事件机制，向epoll注册socket的读写事件，当socket可读可写时调用响应的处理函数。<br><br>你说的协程是应用在OpenResty的Lua虚拟机里。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-16 17:59:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4d/fd/0aa0e39f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>许童童</span>
  </div>
  <div class="_2_QraFYR_0">谈一下这些天你对实验环境里 OpenResty 的感想和认识。<br>我感觉有些时候，写代码比写配置文件更加灵活，OpenResty 通过Lua脚本就可以达到这个效果。<br><br>你觉得 Nginx 和 OpenResty 的“阶段式处理”有什么好处？对你的实际工作有没有启发？<br>阶段式处理，有点类似一个类的生命周期，又有点类似责任链模式。实际工作中编写前端组件，也可以采取类似的方式，把组件渲染分阶段，生命周期细分，使组件更专注更内聚。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 说的挺好。<br><br>其实Nginx最初的模块设计就是想把配置文件弄成语言的形式，通过模块实现指令来增加语言里的词汇，但Nginx的配置文件修改后必须重启，而且C模块开发太麻烦。<br><br>OpenResty引入Lua后C模块开发的就越来越少了，因为脚本语言比简单的指令更灵活，开发的成本也更低。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-16 11:51:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/26/eb/d7/90391376.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ifelse</span>
  </div>
  <div class="_2_QraFYR_0">都是硬货</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: nice</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-06 12:44:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/d6/16/107f0d04.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>山青</span>
  </div>
  <div class="_2_QraFYR_0">这个跟Nginx+lua 感觉跟 Golang的思想有点像啊、</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 现代语言都是互相学习借鉴。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-25 17:48:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/96/47/93838ff7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>青鸟飞鱼</span>
  </div>
  <div class="_2_QraFYR_0">老师，请教一个openresty的问题，实现一个主worker向rabbitmq发送消息，在其他worker将消息发送给主worker，由主worker保持tcp连接。在init_worker_by_lua_block阶段用的定时器ngx.timer.at(0, handler)处理tcp连接，同时在这里注册回调函数，回调函数里发送数据，但是回调函数里面的连接却失效了？怎么回事呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 没在openresty里用过rabbitmq，感觉问题比较复杂，不是在这里回复能够解决的，抱歉了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-01 19:09:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/f6/c4/e14686d4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>shk1230</span>
  </div>
  <div class="_2_QraFYR_0">epoll 和协程有什么关系？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 没有直接的关系。<br><br>epoll是操作系统的多路复用，而协程的编程语言里的范式，而openresty把这两者结合了起来。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-01 18:00:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/29/b0/d3/200e82ff.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>功夫熊猫</span>
  </div>
  <div class="_2_QraFYR_0">io口多路复用和多线程是我拿c写网络编程用的最多的方法。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 现在这个是Linux里的通用做法了，不过在20年前可不是那么容易的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-30 00:14:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/64/9b/d1ab239e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>J.Smile</span>
  </div>
  <div class="_2_QraFYR_0">也可能是nginx已经解决了大部分的问题，openresty对很多公司并没有体现出相比nginx的优势，导致用不起来，不是很流行，大家都知道nginx，但是未必都知道openresty。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: openresty在CDN等领域用的还是很多的，很多大公司也在用，比如阿里京东。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-20 19:02:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/cd/77/b2ab5d44.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>👻 小二</span>
  </div>
  <div class="_2_QraFYR_0">多路复用， 这种是需要操作系统底层提供支持吗？感觉自己的代码再怎么写， 也是多开一个线程在那边等， </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 目前是这样，需要操作系统底层有epoll、kqueue等系统调用，然后基于这些系统调用实现reactor、proactor等模式，也就是多路复用。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-23 17:10:42</div>
  </div>
</div>
</div>
</li>
</ul>