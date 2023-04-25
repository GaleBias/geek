<audio title="加餐（一） _ 书单推荐：性能优化和Linux 系统原理" src="https://static001.geekbang.org/resource/audio/a3/61/a33403c6d755bdb000bd47af0d221361.mp3" controls="controls"></audio> 
<p>你好，我是倪朋飞。欢迎来到 Linux 性能优化专栏的加餐时间。</p><p>之前，很多同学留言让我推荐一些性能优化以及 Linux 系统原理方面的书，今天我就和你分享一些我认为不错的书。</p><p>Linux 系统原理和性能优化涉及的面很广，相关的书籍自然也很多。学习咱们专栏，你先要了解Linux 系统的工作原理，基于此，再去分析、理解各类性能瓶颈，最终找出方法、优化性能。围绕这几个方面，我来推荐一些相应书籍。</p><h2>Linux基础入门书籍：《鸟哥的Linux私房菜》</h2><p><img src="https://static001.geekbang.org/resource/image/8e/b4/8e3b114e11f6f5195e176290e4aa6eb4.png?wh=800*800" alt=""></p><p>咱们专栏的目标是优化 Linux 系统以及在Linux上运行的软件性能。那么，第一步当然是要熟悉 Linux 本身。所以，我推荐的第一本书，正是小有名气的 Linux 系统入门书——《鸟哥的 Linux 私房菜》。</p><p>这本书以 CentOS 7 为例，介绍了 Linux 系统的基本使用和管理方法，主要内容包括系统安装、文件和目录操作、磁盘和文件系统管理、编辑器、Bash 以及 Linux 系统的管理维护等。这些内容都是 Linux 初学者需要掌握的基础知识，非常适合刚入门 Linux 系统的新手。</p><p>当然，掌握这些基础知识，其实也是学习咱们专栏的基本门槛。比如，我在很多案例里提到的软件包的安装、Bash 命令的运行、grep 和 awk 等基本命令的使用、文档的查询方法等，这本书都有涉及。</p><!-- [[[read_end]]] --><p>另外，这本书的大部分内容，还可以在其繁体中文<a href="http://linux.vbird.org/linux_basic/">官方网站</a>上在线学习。</p><h2>计算机原理书籍：《深入理解计算机系统》</h2><p><img src="https://static001.geekbang.org/resource/image/6b/07/6b0cadb6858c3e00885e829d0910b207.png?wh=348*499" alt=""></p><p>掌握 Linux 基础后，接下来就该进一步理解计算机系统的工作原理。所以，我推荐的第二本书，正是计算机系统原理的经典黑皮书——《深入理解计算机系统》。</p><p>这也是一本经典的计算机学科入门教材，它的英文版名称“Computer Systems: A Programmer’s Perspective”，其实更能体现本书的核心，即从开发者的角度来理解计算机系统。</p><p>这本书介绍了计算机系统最基本的工作原理，内容比较广泛。它主要包括信息的计算机表示，程序的编译、链接及运行，处理器体系结构，虚拟内存，存储系统 I/O，网络以及并发等内容。</p><p>书本身比较厚，内容也比较多，但作为一本优秀的入门书籍，这本书介绍的各个知识点虽然有点偏向于编程和系统底层，但并不会过于深入这些，对初学者来说非常合适。</p><p>此外，这本书的<a href="http://csapp.cs.cmu.edu/">官方网站</a>上还提供了丰富的资源，可以帮你进一步理解、深入书里的内容，还提供了多个实验操作，助你加深掌握。</p><h2>Linux编程书籍：《Linux程序设计》和《UNIX环境高级编程》</h2><p><img src="https://static001.geekbang.org/resource/image/1f/e7/1fe3cc0a1d0772282be0047dbfd67fe7.png?wh=412*500" alt=""><img src="https://static001.geekbang.org/resource/image/86/90/86ac9cfbba6a255c3592de13950be190.png?wh=665*1000" alt=""></p><p>介绍完计算机系统工作原理的书籍，接下来，我要推荐的是编程相关的两本书，分别是《Linux 程序设计》和《UNIX 环境高级编程》。</p><p>之所以要推荐编程书籍，是因为优化性能的过程中，理解应用程序的执行逻辑至关重要。而要做到这一点，编程基础就是刚需。</p><p>我推荐的这两本书中，《Linux 程序设计》主要针对 Linux 系统中的应用程序开发，是一本入门书籍，内容包括 SHELL、标准库、数据库、多进程、进程间通信、套接字以及图像界面等。</p><p>《UNIX 环境高级编程》则被誉为 UNIX 编程圣经，是深入 UNIX 环境（包括Linux）编程的必读书籍。主要内容包括标准库、文件 I/O、进程控制、多进程和进程间通信、多线程以及高级 I/O 等，这些内容都是开发高性能、高可靠应用程序的必备基础。</p><p>这两本书籍，可以让你更清楚 Linux 系统以及应用程序的执行过程，甚至在必要时帮你更好地理解应用程序乃至内核的源代码。</p><h2>Linux内核书籍：《深入Linux内核架构》</h2><p><img src="https://static001.geekbang.org/resource/image/e1/5e/e1ed53283b51ed81a96b9c9d2e72d65e.png?wh=398*500" alt=""></p><p>为了方便你学习和运用，我们专栏内容都是从 Linux 系统的原理出发，借助系统内置或外部安装的各类工具，找出瓶颈所在。所以，理解 Linux 系统原理也是我们的重点，同时，了解内核架构，也可以帮助你分析清楚瓶颈为什么发生。</p><p>所以，我推荐的第五本，就是关于 Linux 内核原理的一本书籍——《深入Linux内核架构》。这是一本大块头，涉及了 Linux 内核中的进程管理、内存管理、文件系统、磁盘、网络、设备驱动、时钟等大量知识。书中还引用了大量 Linux 内核的源码（内核版本为 2.6.24，虽然有些老，但不影响你理解原理），帮你透彻掌握相关知识点。</p><p>如果你是第一次读这本书，不要因为厚厚的页码或者部分内容看不懂就放弃。换个时间重新来看，你会有不同的发现。</p><h2>性能优化书籍：《性能之巅：洞悉系统、企业与云计算》</h2><p><img src="https://static001.geekbang.org/resource/image/5b/55/5b8392e187c770b796c445ded4819655.png?wh=1000*1070" alt=""></p><p>最后一本，是我曾在 <a href="https://time.geekbang.org/column/article/73738">Linux 性能优化答疑（二）</a>中提到过的《性能之巅：洞悉系统、企业与云计算》。</p><p>这本书，堪称 Linux 性能优化最权威的一本书，而作者 Brendan Gregg ，也是很多我们熟悉的性能优化工具和方法的开创者。</p><p>书里主要提供了 Linux 性能分析和调优的基本思路，并具体讲解，如何借助动态追踪等性能工具，分析并优化各种性能问题。同时，这本书也介绍了很多性能工具的使用方法，可以当作你性能优化过程的工具参考书。</p><p>最后，我还想再说一句，读书不在多，而在精。</p><p>今天我推荐的这些书，你可能或多或少都看过一部分，但这远远不够。要真正掌握它们的核心内容，不仅需要你理解书中讲解的内容，更需要你用大量实践来融汇贯通。</p><p>有些书，你可能会觉得很难啃下来，还不如现在层出不穷的新技术时髦。但要注意，这些内容都是基本不会过时的硬知识，多花点儿时间坚持啃下来，相信你一定会有巨大的收获。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/4e/0f/c43745e7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>hola</span>
  </div>
  <div class="_2_QraFYR_0">花了一周，每天下班晚上追，晚上终于追到这里来了。<br>关于深入理解计算机系统这本书，我建议不要完全按照目录顺序阅读，这样会容易从入门到放弃。<br>可以从后面几章开始阅读，前面几张是偏操作系统硬件层面，后面是偏操作系统软件层面，更贴近我们平时的了解。比如系统级IO，网络编程等等</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯呢</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-23 22:08:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/c4/a5/716951be.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>dahey</span>
  </div>
  <div class="_2_QraFYR_0">谢谢老师推荐的书籍，虽然这几本我差不多都买了纸质书，但是深入理解计算机系统是真的难啃😢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以先把跟手头工作相关的部分啃下来，其他再慢慢来，这样效果更好</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-04 23:27:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/e0/99/5d603697.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>MJ</span>
  </div>
  <div class="_2_QraFYR_0">正在读深入理解计算机系统，每天10页</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍 加油</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-04 08:53:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/ff/f6/a6708f8d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>crane</span>
  </div>
  <div class="_2_QraFYR_0">这些书DBA有必须都看吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 建议了解下</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-04 09:47:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/3a/6e/e39e90ca.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>大坏狐狸</span>
  </div>
  <div class="_2_QraFYR_0">老师，我这个人比较笨，比如linux 鸟哥私房菜 服务器架设篇--<br>那么厚的书，我是要读完这一本再读下一本，还是交叉的学习呢。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个看你的安排和学习习惯吧。我一般是看情况，有时间的化就按照书籍系统学习，但如果只对某个主题感兴趣，那就找几本相关书交叉学习。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-14 17:09:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/4c/c8/bed1e08a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>辣椒</span>
  </div>
  <div class="_2_QraFYR_0">老师辛苦，大年三十还推专栏。列举的书都是经典，我虽然是做开发的，也想买来读一下</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍 这些都是开发必读的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-04 07:53:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKsz8j0bAayjSne9iakvjzUmvUdxWEbsM9iasQ74spGFayIgbSE232sH2LOWmaKtx1WqAFDiaYgVPwIQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>2xshu</span>
  </div>
  <div class="_2_QraFYR_0">哎呀呀，准备起来看网络呢。结果没更新</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 现在已经更新两篇了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-04 07:38:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://wx.qlogo.cn/mmopen/vi_32/PiajxSqBRaEKQMM4m7NHuicr55aRiblTSEWIYe0QqbpyHweaoAbG7j2v7UUElqqeP3Ihrm3UfDPDRb1Hv8LvPwXqA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ninuxer</span>
  </div>
  <div class="_2_QraFYR_0">打卡day34<br>开工，赶紧补补落下的课程，深入理解Linux内核架构和性能之巅已经开始啃了，需要很大的毅力啊～</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-13 19:36:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/97/68/bb8377d1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>KimiZhou</span>
  </div>
  <div class="_2_QraFYR_0">谢谢老师的推荐。最近在解决一个多媒体服务进程的性能瓶颈问题，参考了老师推荐的一些方法，的确给了我很多指导和启发。谢谢~</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 😊</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-22 11:49:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/c4/7e/35a129da.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>void *</span>
  </div>
  <div class="_2_QraFYR_0">买了本性能之巅，摩拜下大神的巨作</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-10 09:26:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/69/88/528442b0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Dale</span>
  </div>
  <div class="_2_QraFYR_0">性能之巅不错，作为工具书使用</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-12 09:06:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/00/83/9329a697.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>马殿军</span>
  </div>
  <div class="_2_QraFYR_0">很棒！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-04 08:45:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/4a/fb/70f14340.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>maoxiajun</span>
  </div>
  <div class="_2_QraFYR_0">打卡</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-12 19:55:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/91/d0/35bc62b1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>无咎</span>
  </div>
  <div class="_2_QraFYR_0">建议搞的豆瓣的图书豆列。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-15 20:13:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/ajNVdqHZLLAhj2fB8NI2TPI1SNicgiciczuMUHyAb9HHBkkKJHrgtR162fsicaTqdAneHfuVX7icDXaVibDHstM9L47g/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_0c1732</span>
  </div>
  <div class="_2_QraFYR_0">学这些要会C吗</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-16 05:20:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/51/76/bebe6133.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>TomShine</span>
  </div>
  <div class="_2_QraFYR_0">推荐的书都不错</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-14 00:01:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/1d/de/62bfa83f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>aoe</span>
  </div>
  <div class="_2_QraFYR_0">感谢老师推荐，找到对的书也是力气活。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-13 18:53:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/82/3d/356fc3d6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>忆水寒</span>
  </div>
  <div class="_2_QraFYR_0">晚来了一年了，该课程有很大收获。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-24 21:15:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/53/f4/e277325d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>bin.chen</span>
  </div>
  <div class="_2_QraFYR_0">这么多书，边学边实践才能有所提高，书太多，怕自己坚持不下来</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-25 00:32:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/f0/8f/ca9fded0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>~尘曦~</span>
  </div>
  <div class="_2_QraFYR_0">深入理解Linux内核架构 里面有些C代码还是比较晕的<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-18 16:32:30</div>
  </div>
</div>
</div>
</li>
</ul>