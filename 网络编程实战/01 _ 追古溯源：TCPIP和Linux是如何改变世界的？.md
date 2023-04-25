<audio title="01 _ 追古溯源：TCPIP和Linux是如何改变世界的？" src="https://static001.geekbang.org/resource/audio/ac/c8/acd295377e01afc19716ab214423ebc8.mp3" controls="controls"></audio> 
<p>你好，我是盛延敏。今天是网络编程课程的第一课，我想你一定满怀热情，期望快速进入到技术细节里，了解那些你不熟知的编程技能。而今天我却想和你讲讲历史，虽然这些事情看着不是“干货”，但它可以帮助你理解网络编程中各种技术的来龙去脉。</p><p>你我都是程序员，说句实在话，我们正处于一个属于我们的时代里，我们也正在第一线享受着这个时代的红利。在我看来，人类历史上还从来没有一项技术可以像互联网一样深刻地影响人们生活的方方面面。</p><p>而具体到互联网技术里，有两件事最为重要，一个是TCP/IP协议，它是万物互联的事实标准；另一个是Linux操作系统，它是推动互联网技术走向繁荣的基石。</p><p>今天，我就带你穿越时间的走廊，看一看TCP/IP事实标准和Linux操作系统是如何一步一步发展到今天的。</p><h2>TCP发展历史</h2><p>一般来说，我们认为互联网起源于阿帕网（ARPANET）。</p><p>最早的阿帕网还是非常简陋的，网络控制协议（Network Control Protocol，缩写NCP）是阿帕网中连接不同计算机的通信协议。</p><p>在构建阿帕网（ARPANET）之后，其他数据传输技术的研究又被摆上案头。NCP诞生两年后，NCP的开发者温特·瑟夫（Vinton Cerf）和罗伯特·卡恩（Robert E. Kahn）一起开发了一个阿帕网的下一代协议，并在1974年发表了以分组、序列化、流量控制、超时和容错等为核心的一种新型的网络互联协议，一举奠定了TCP/IP协议的基础。</p><!-- [[[read_end]]] --><h3>OSI &amp; TCP/IP</h3><p>在这之后，TCP/IP逐渐发展。咱们话分两头说，一头是一个叫ISO的组织发现计算机设备的互联互通是一个值得研究的新领域，于是，这个组织出面和众多厂商游说，“我们一起定义，出一个网络互联互通的标准吧，这样大家都遵守这个标准，一起把这件事情做大，大家就有钱赚了”。众多厂商觉得可以啊，于是ISO组织就召集了一帮人，认真研究起了网络互联这件事情，还真的搞出来一个非常强悍的标准，这就是OSI参考模型。这里我不详细介绍OSI参考模型了，你可以阅读<a href="https://time.geekbang.org/column/article/99286">罗剑锋老师的专栏</a>，他讲得很好。</p><p>这个标准发布的时候已经是1984年了，有点尴尬的是，OSI搞得是很好，大家也都很满意，不过，等它发布的时候，ISO组织却惊讶地发现满世界都在用一个叫做TCP/IP协议栈的东西，而且，跟OSI标准半毛钱关系都没有。</p><p>这就涉及到了另一头——TCP/IP的发展。</p><p>事实上，我在前面提到的那两位牛人卡恩和瑟夫，一直都在不遗余力地推广TCP/IP。当然，TCP/IP的成功也不是偶然的，而是综合了几个因素后的结果：</p><ol>
<li>TCP/IP是免费或者是少量收费的，这样就扩大了使用人群；</li>
<li>TCP/IP搭上了UNIX这辆时代快车，很快推出了基于套接字（socket）的实际编程接口；</li>
<li>这是最重要的一点，TCP/IP来源于实际需求，大家都在翘首盼望出一个统一标准，可是在此之前实际的问题总要解决啊，TCP/IP解决了实际问题，并且在实际中不断完善。</li>
</ol><p>回过来看，OSI的七层模型定得过于复杂，并且没有参考实现，在一定程度上阻碍了普及。</p><p>不过，OSI教科书般的层次模型，对后世的影响很深远，一般我们说的4层、7层，也是遵从了OSI模型的定义，分别指代传输层和应用层。</p><p>我们说TCP/IP的应用层对应了OSI的应用层、表示层和会话层；TCP/IP的网络接口层对应了OSI的数据链路层和物理层。</p><p><img src="https://static001.geekbang.org/resource/image/cb/b4/cb34e0e3b7769498ea703fe6231201b4.png?wh=958*738" alt=""></p><h2>UNIX操作系统发展历史</h2><p>前面我们提到了TCP/IP协议的成功，离不开UNIX操作系统的发展。接下来我们就看下UNIX操作系统是如何诞生和演变的。</p><p>下面这张图摘自<a href="https://en.wikipedia.org/wiki/File:Unix_timeline.en.svg">维基百科</a>，它将UNIX操作系统几十年的发展历史表述得非常清楚。</p><p><img src="https://static001.geekbang.org/resource/image/a6/0f/a68c3b9b267574ea2f309ed6a4e0de0f.png?wh=1579*1198" alt=""><br>
UNIX的各种版本和变体都起源于在PDP-11系统上运行的UNIX分时系统第6版（1976年）和第7版（1979年），它们通常分别被称为V6和V7。这两个版本是在贝尔实验室以外首先得到广泛应用的UNIX系统。</p><p>这张图画得比较概括，我们主要从这张图上看3个分支：</p><ul>
<li>图上标示的Research橘黄色部分，是由AT&amp;T贝尔实验室不断开发的UNIX研究版本，从此引出UNIX分时系统第8版、第9版，终止于1990年的第10版（10.5）。这个版本可以说是操作系统界的少林派。天下武功皆出少林，世上UNIX皆出自贝尔实验室。</li>
<li>图中最上面所标识的操作系统版本，是加州大学伯克利分校（BSD）研究出的分支，从此引出4.xBSD实现，以及后面的各种BSD版本。这个可以看做是学院派。在历史上，学院派有力地推动了UNIX的发展，包括我们后面会谈到的socket套接字都是出自此派。</li>
<li>图中最下面的那一个部分，是从AT&amp;T分支的商业派，致力于从UNIX系统中谋取商业利润。从此引出了System III和System V（被称为UNIX的商用版本），还有各大公司的UNIX商业版。</li>
</ul><p>下面这张图也是源自维基百科，将UNIX的历史表达得更为详细。</p><p><img src="https://static001.geekbang.org/resource/image/df/bb/df2b6d77a0a46e3d9b068f6d517a15bb.png?wh=2560*1774" alt=""><br>
一个基本事实是，网络编程套接字接口，最早是在BSD 4.2引入的，这个时间大概是1983年，几经演变后，成为了事实标准，包括System III/V分支也吸收了这部分能力，在上面这张大图上也可以看出来。</p><p>其实这张图也说明了一个很有意思的现象，BSD分支、System III/System V分支、正统的UNIX分时系统分支都是互相借鉴的，也可以说是互相“抄袭”吧。但如果这样发展下去，互相不买对方的账，导致上层的应用程序在不同的UNIX版本间不能很好地兼容，这该怎么办？这里先留一个疑问，你也可以先想一想，稍后我会给你解答。</p><p>下面我再介绍几个你耳熟能详的重要UNIX玩家。</p><h3>SVR 4</h3><p>SVR4（UNIX System V Release 4）是AT&amp;T的UNIX系统实验室的一个商业产品。它基本上是一个操作系统的大杂烩，这个操作系统之所以重要，是因为它是System III/V分支各家商业化UNIX操作系统的“先祖”，包括IBM的AIX、HP的HP-UX、SGI的IRIX、Sun（后被Oracle收购）的Solaris等等。</p><h3>Solaris</h3><p>Solaris是由Sun Microsystems（现为Oracle）开发的UNIX系统版本，它基于SVR4，并且在商业上获得了不俗的成绩。2005年，Sun Microsystems开源了Solaris操作系统的大部分源代码，作为OpenSolaris开放源代码操作系统的一部分。相对于Linux，这个开源操作系统的进展比较一般。</p><h3>BSD</h3><p>BSD（Berkeley Software Distribution），我们上面已经介绍过了，是由加州大学伯克利分校的计算机系统研究组（CSRG）研究开发和分发的。4.2BSD于1983年问世，其中就包括了网络编程套接口相关的设计和实现，4.3BSD则于1986年发布，正是由于TCP/IP和BSD操作系统的完美拍档，才有了TCP/IP逐渐成为事实标准的这一历史进程。</p><h3>macOS X</h3><p>用mac笔记本的同学都有这样的感觉：macOS提供的环境和Linux环境非常像，很多代码可以在macOS上以接近线上Linux真实环境的方式运行。</p><p>有心的同学应该想过，这背后有一定的原因。</p><p>答案其实很简单，macOS和Linux的血缘是相近的，它们都是UNIX基础上发展起来的，或者说，它们各自就是一个类UNIX的系统。</p><p>macOS系统又被称为Darwin，它已被验证过就是一个UNIX操作系统。如果打开Mac系统的socket.h头文件定义，你会明显看到macOS系统和BSD千丝万缕的联系，说明这就是从BSD系统中移植到macOS系统来的。</p><h2>Linux</h2><p>我们把Linux操作系统单独拿出来讲，是因为它实在太重要了，全世界绝大部分数据中心操作系统都是跑在Linux上的，就连手机操作系统Android，也是一个被“裁剪”过的Linux操作系统。</p><p>Linux操作系统的发展有几个非常重要的因素，这几个因素叠加在一起，造就了如今Linux非凡的成就。我们一一来看。</p><h3>UNIX的出现和发展</h3><p>第一个就是UNIX操作系统，要知道，Linux操作系统刚出世的时候， 4.2/4.3 BSD都已经出现快10年了，这样就为Linux系统的发展提供了一个方向，而且Linux的开发语言是C语言，C语言也是在UNIX开发过程中发明的一种语言。</p><h3>POSIX标准</h3><p>UNIX操作系统虽然好，但是它的源代码是不开源的。那么如何向UNIX学习呢？这就要讲一下POSIX标准了，POSIX（Portable Operating System Interface for Computing Systems）这个标准基于现有的UNIX实践和经验，描述了操作系统的调用服务接口。有了这么一个标准，Linux完全可以去实现并兼容它，这从最早的Linux内核头文件的注释可见一斑。</p><p>这个头文件里定义了一堆POSIX宏，并有一句注释：“嗯，也许只是一个玩笑，不过我正在完成它。”</p><pre><code># ifndef _UNISTD_H

# define _UNISTD_H


/* ok, this may be a joke, but I'm working on it */

# define _POSIX_VERSION  198808L


# define _POSIX_CHOWN_RESTRICTED /* only root can do a chown (I think..) */

/* #define _POSIX_NO_TRUNC*/ /* pathname truncation (but see in kernel) */

# define _POSIX_VDISABLE '\0' /* character to disable things like ^C */

/*#define _POSIX_SAVED_IDS */ /* we'll get to this yet */

/*#define _POSIX_JOB_CONTROL */ /* we aren't there quite yet. Soon hopefully */
</code></pre><p>POSIX相当于给大厦画好了图纸，给Linux的发展提供了非常好的指引。这也是为什么我们的程序在macOS和Linux可以兼容运行的原因，因为大家用的都是一张图纸，只不过制造商不同，程序当然可以兼容运行了。</p><h3>Minix操作系统</h3><p>刚才提到了UNIX操作系统不开源的问题，那么有没有一开始就开源的UNIX操作系统呢？这里就要提到Linux发展的第三个机遇，Minix操作系统，它在早期是Linux发展的重要指引。这个操作系统是由一个叫做安迪·塔能鲍姆（Andy Tanenbaum）的教授开发的，他的本意是用来做UNIX教学的，甚至有人说，如果Minix操作系统也完全走社区开放的道路，那么未必有现在的Linux操作系统。当然，这些话咱们就权当作是马后炮了。Linux早期从Minix中借鉴了一些思路，包括最早的文件系统等。</p><h3>GNU</h3><p>Linux操作系统得以发展还有一个非常重要的因素，那就是GNU（GNU’s NOT UNIX），它的创始人就是鼎鼎大名的理查·斯托曼（Richard Stallman）。斯托曼的想法是设计一个完全自由的软件系统，用户可以自由使用，自由修改这些软件系统。</p><p>GNU为什么对Linux的发展如此重要呢？事实上，GNU之于Linux是要早很久的，GNU在1984年就正式诞生了。最开始，斯托曼是想开发一个类似UNIX的操作系统的。</p><blockquote>
<p>From CSvax:pur-ee:inuxc!ixn5c!ihnp4!houxm!mhuxi!eagle!mit-vax!mit-eddie!RMS@ MIT-OZ<br>
From: RMS% MIT-OZ@ mit-eddie<br>
Newsgroups: net.unix-wizards,net.usoft<br>
Subj ect: new UNIX implementation<br>
Date: Tue, 27-Sep-83 12:35:59 EST<br>
Organization: MIT AI Lab, Cambridge, MA<br>
Free Unix!<br>
Starting this Thanksgiving I am going to write a complete Unix-compatible software system called GNU (for Gnu’s Not Unix), and give it away free to everyone who can use it. Contributions of time,money, programs and equipment are greatly needed.<br>
To begin with, GNU will be a kernel plus all the utilities needed to write and run C programs: editor, shell, C compiler, linker, assembler, and a few other things. After this we will add a text formatter, a YACC, an Empire game, a spreadsheet, and hundreds of other things. We hope to supply, eventually, everything useful that normally comes with a Unix system, and anything else useful, including on-line and hardcopy documentation.<br>
…</p>
</blockquote><p>在这个设想宏大的GNU计划里，包括了操作系统内核、编辑器、shell、编译器、链接器和汇编器等等，每一个都是极其难啃的硬骨头。</p><p>不过斯托曼可是个牛人，单枪匹马地开发出世界上最牛的编辑器Emacs，继而组织和成立了自由软件基金会（the Free Software Foundation - FSF）。</p><p>GNU在自由软件基金会统一组织下，相继推出了编译器GCC、调试器GDB、Bash Shell等运行于用户空间的程序。正是这些软件为Linux 操作系统的开发创造了一个合适的环境，比如编译器GCC、Bash Shell就是Linux能够诞生的基础之一。</p><p>你有没有发现，GNU独缺操作系统核心？</p><p>实际上，1990年，自由软件基金会开始正式发展自己的操作系统Hurd，作为GNU项目中的操作系统。不过这个项目再三耽搁，1991年，Linux出现，1993年，FreeBSD发布，这样GNU的开发者开始转向于Linux或FreeBSD，其中，Linux成为更常见的GNU软件运行平台。</p><p>斯托曼主张，Linux操作系统使用了许多GNU软件，正式名应为GNU/Linux，但没有得到Linux社群的一致认同，形成著名的GNU/Linux命名争议。</p><p>GNU是这么解释为什么应该叫GNU/Linux的：“大多数基于Linux内核发布的操作系统，基本上都是GNU操作系统的修改版。我们从1984 年就开始编写GNU 软件，要比Linus开始编写它的内核早许多年，而且我们开发了系统的大部分软件，要比其它项目多得多，我们应该得到公平对待。”</p><p>从这段话里，我们可以知道GNU和GNU/Linux互相造就了对方，没有GNU当然没有Linux，不过没有Linux，GNU也不可能大发光彩。</p><p>在开源的世界里，也会发生这种争名夺利的事情，我们也不用觉得惊奇。</p><h2>操作系统对TCP/IP的支持</h2><p>讲了这么多操作系统的内容，我们再来看下面这张图。图中展示了TCP/IP在各大操作系统的演变历史。可以看到，即使是大名鼎鼎的Linux以及90年代大发光彩的Windows操作系统，在TCP/IP网络这块，也只能算是一个后来者。</p><p><img src="https://static001.geekbang.org/resource/image/0f/e1/0f783e74927d70794421cf5983f22ae1.png?wh=1450*1354" alt=""></p><h2>总结</h2><p>这是我们专栏的第一讲，我没有直接开始讲网络编程，而是对今天互联网技术的基石，TCP和Linux进行了简单的回顾。通过这样的回顾，熟悉历史，可以指导我们今后学习的方向，在后面的章节中，我们都将围绕Linux下的TCP/IP程序设计展开。</p><p>最后你不妨思考一下，Linux TCP/IP网络协议栈最初的实现“借鉴”了多少BSD的实现呢？Linux到底是不是应该被称为GNU/Linux呢？</p><p>欢迎你在评论区写下你的思考，我会和你一起讨论这些问题。如果这篇文章帮你厘清了TCP/IP和Linux的发展脉络，欢迎把它分享给你的朋友或者同事。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/a0/a3/8da99bb0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>业余爱好者</span>
  </div>
  <div class="_2_QraFYR_0">网络协议，epoll,系统调用...脑子里对这些基础知识一直一知半解，不成体系，希望通过追一个贴近实战的网络专栏建立知识之间的关联。flag~</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 竭力帮你达成所愿</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-02 19:21:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/b5/37/ea4d7271.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>陈俊豪200</span>
  </div>
  <div class="_2_QraFYR_0">讲的很好啊，很多书上提及的混乱的历史都被梳理了一遍，很有兴趣，谢谢老师！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这就是我想要的目标</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-02 17:08:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83erC0Eto5QhW3TgBvkg9weaWcrMl4bcADqHbiaxbcHq9RXqIibr1xyCibhMBgXG2oYlTdNOQmLMgJqQIw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>java小白</span>
  </div>
  <div class="_2_QraFYR_0">能把这么复杂的问题讲清楚，真的是一种能力，真心的叫你一声“老师”</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢，这让我很有成就感</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-06 17:08:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/44/a8/0ce75c8c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Skrpy</span>
  </div>
  <div class="_2_QraFYR_0">我认为没必要把世人熟知的 Linux 改成 GNU&#47;Linux，我觉得如果一个操作系统名字这么写，那着实不好看……(我不是说 TCP&#47;IP )。C 语言的编译器就是 GNU 家的，它还有许多为人称道的自由软件，GNU 会名垂千古，没必要把自己贴到同样名垂千古的 Linux 脸上</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 各有各的想法吧</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-02 13:45:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/43/5c/b9eafca8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>高超</span>
  </div>
  <div class="_2_QraFYR_0">支持将 linux 叫做 GNU&#47;Linux，这样也算是对 GNU 致敬了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你是挺GUN的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-02 11:36:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/9b/a8/6a391c66.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Leon📷</span>
  </div>
  <div class="_2_QraFYR_0">之前学习linux网络编程的时候被人给忽悠了，说mac系统不需要装linux，完全兼容，然后学习过程中遇到了一堆不明觉厉的问题，最坑的是macos没有epoll，然后我和一个同学一脸懵逼的把epoll改成了poll才把程序编过去，学习linux编程最好还是装个linux虚拟机，哈哈哈，当前往事不堪回首</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 用vagarant+virtualbox，完美搞定Linux虚拟机环境</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-02 16:59:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/7f/ca/ea85bfdd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>helloworld</span>
  </div>
  <div class="_2_QraFYR_0">老师文章中那两张UNIX发展历史图非常好，让人一下子明白了UNIX、Linux和我所用的Mac系统的关系。UNIX操作系统最早诞生于贝尔实验室，这是所欲*NIX系统的鼻祖。在这个基础上出现了各种各样的发行版，包括开源，闭源的和商业的。名宿Linux属于开源的，Mac属于闭源的（并且属于BSD分支中的一个演变过来的），而SUN（已被Oracle收购）的Solaris等均属于商业性质的</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-13 11:19:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/bf/55/198c6104.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小伟</span>
  </div>
  <div class="_2_QraFYR_0">了解操作系统的由来给我最大的启发是，没有什么是理所当然和凭空出现的，有需求才有产品。理解了需求，也许可以设计出更优秀的产品，而不是只能学习和使用别人的</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-11 09:09:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/a1/a0/7687413f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Zopen</span>
  </div>
  <div class="_2_QraFYR_0">问个问题：<br><br>摄像头推送视频流到流媒体服务器（基于TCP传输），理论传输速度是3M&#47;S，实际测试发现是2M&#47;S，请分析下可能出现的原因？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 网卡，交换机，应用程序，都可能是瓶颈</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-02 21:00:53</div>
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
  <div class="_2_QraFYR_0">有些看隋唐演义的感觉，过瘾！<br>有生于无，由简入繁，认知还是需要一个循序渐进的过程的，这样就不会怀疑自己是否有大脑了，会觉得一切都是自然而然的就发生了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 了解历史，才能知道我们从哪里来，到哪里去。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-18 08:55:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/b3/4e/8b765c56.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Mr. Ren</span>
  </div>
  <div class="_2_QraFYR_0">连续听了两节课，第一节中所说的学习网络过程中的那种心里很符合我现在的情况，到底是学基础还是学框架有点迷茫，第二课中说了发展历史，对这些年了解的历史更加细一些，图文加语音，足见老师对课程的用心，站在巨人的肩上，可以看的得更远，期待老师下一课的课程🤝</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我在用心写，你在用心听，一定会有收获</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-04 19:48:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/pZ5ibu3jOPTfWVtzTeNTiaL2PiabGT2Y2yKd2TNDcZMkIY34T5fhGcSnBjgpkd54Q3S6b3gRW3yYTxZk0QHYB0qnw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>啊树</span>
  </div>
  <div class="_2_QraFYR_0">老师 OSI模型更多的是运用到硬件设备分层，TCPIP协议更多作为互联网协议么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: OSI是一个参考模型，实际上我们用的都是TCPIP协议，TCPIP里也有和硬件设备打交道的分层</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-04 18:04:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/b6/17/76f29bfb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_66666</span>
  </div>
  <div class="_2_QraFYR_0">所以平时教科书上osi七层在实际中并没有采用而是采用tcp&#47;ip标准？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-02 17:56:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/46/12/965a6cc9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>菠萝power</span>
  </div>
  <div class="_2_QraFYR_0">为什么在那么早期、网络还没有开始普及时候，就想出这些个复杂、全面的协议，并一直持续发展下去，以前的人怎么那么强啊</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 看看我们的四大发明，长城，你就不会觉得奇怪了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-20 23:18:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/6c/a8/1922a0f5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>郑祖煌</span>
  </div>
  <div class="_2_QraFYR_0">毕竟是GNU先出来的，所以Linux为了自身的营销和发展，叫自己为GNU&#47;Linux 估计也是为了借这个GNU这个势能吧。就像JavaScript明明是跟Java无关的一种语言，为了自身的发展，可能需要搭在老大哥的名字上。这算是借助别人文化势能的力量来发展自己，等自己发展壮大了，自己本身也成了一种文化势能。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Linux是不想叫GNU&#47;Linux，GNU组织觉得大家应该一起玩。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-12 16:51:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/3c/8a/900ca88a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>test</span>
  </div>
  <div class="_2_QraFYR_0">老师，linux的源码和unix到底一样吗还是实现区别大？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 实现是完全不一样的，你可以这么理解，Linux照着Unix对外的需求(API)自己从零开始写了一个。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-21 22:19:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/39/b8/e4b6a677.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>YUANWOW</span>
  </div>
  <div class="_2_QraFYR_0">我觉得应该是整体借鉴<br>是Linux还是GNU&#47;Linux <br>Linux只是内核，如果发行版的内核不是通过GCC编译出来的可以叫Linux<br>如果是GCC编译出来的就应该叫做GNU&#47;Linux  </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 仁者见仁智者见智吧，我看Ubuntu就是叫GUN&#47;Linux<br><br>85-Ubuntu SMP Tue Jul 23 09:17:01 UTC 2019 x86_64 x86_64 x86_64 GNU&#47;Linux</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-08 20:15:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/8f/3b/234ce259.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>萌妻的路飞</span>
  </div>
  <div class="_2_QraFYR_0">打卡，对知识的来龙去脉有了解，更有助于对知识的学习</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，很多时候我们在课堂上学习不甚了了，是因为我们不知道来龙去脉</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-02 11:39:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/b7/37/171c307e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>葛学强</span>
  </div>
  <div class="_2_QraFYR_0">打卡</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-02 09:20:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/af/32/74465c5e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Karl</span>
  </div>
  <div class="_2_QraFYR_0"> 哈哈，不知道 GNU 读成 &#47;nju:&#47; 是哪个英语国家的习惯。在澳洲 GNU 普遍被读成 &#47;g(ɜ)nu:&#47;。还有一个有趣的单词 “data” 在澳洲被读成 &#47;da:ta:&#47; </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 汗，这个读法确实很折腾人</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-02 07:57:03</div>
  </div>
</div>
</div>
</li>
</ul>