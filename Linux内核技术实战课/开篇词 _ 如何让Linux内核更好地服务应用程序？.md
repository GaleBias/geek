<audio title="开篇词 _ 如何让Linux内核更好地服务应用程序？" src="https://static001.geekbang.org/resource/audio/89/71/89ac84fb7b9d68c2a8e927dfdda02071.mp3" controls="controls"></audio> 
<p>你好，我是邵亚方，欢迎加入我的课程，和我一起学习Linux内核知识。</p><p>从2010年接触Linux内核到现在，因为工作的关系，我参与解决了大量直接与生产环境相关的性能问题。前些年，我还在蘑菇街的时候，那会蘑菇街的业务增长速度非常快。你知道，业务增长了，随之而来的肯定就是服务的稳定性挑战了，比如TCP重传该怎么分析、怎么在运⾏时不打断任务的情况下排查内存泄漏问题、CPU sys利⽤率⾼怎么快速解决，这都是实实在在的问题，你会就会，不会就是不会。</p><p>以我们常见的TCP重传为例，如果你熟悉的话，服务器上一般都会有TCP重传率的监控，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/ba/f6/baee41941867cd5d0ee36c8f67d389f6.jpg?wh=896*488" alt=""></p><p>就像图片显示的那样，这么高的TCP重传率，必然会导致系统QPS减小，所以你不敢含糊，得赶紧找问题出在哪里。但真正排查的时候，你会发现不知道从哪里开始，因为发生重传时的现场信息并没有记录下来。之所以没有记录下来这些现场信息，是因为记录的成本太昂贵了。如果你排查过网络问题，你应该知道，网络数据量是非常大的，即使只记录TCP头部信息也是非常大的存储开销，而且信息记录的过程也会带来较多的性能开销。</p><p>为了解决这类问题，我们团队就针对TCP重传的监控做了很多的改进，这些改进可以在不对业务性能造成明显影响的情况下，提升团队定位和分析问题的效率。我们记录的TCP重传现场信息如下所示：</p><!-- [[[read_end]]] --><pre><code>18:21:58 10.17.130.19:20612 124.74.250.144:44  SYN_SENT   
18:22:00 10.17.130.19:20612 124.74.250.144:443 SYN_SENT   
18:23:21 10.17.130.19:20716 124.74.250.144:443 SYN_SENT   
18:23:23 10.17.130.19:20716 124.74.250.144:443 SYN_SENT   
18:24:39 10.17.130.19:20796 124.74.250.144:443 SYN_SENT   
18:24:41 10.17.130.19:20796 124.74.250.144:443 SYN_SENT   
18:25:43 10.17.130.19:20861 124.74.250.144:443 SYN_SENT   
18:25:45 10.17.130.19:20861 124.74.250.144:443 SYN_SENT   
18:27:23 10.17.130.19:20973 124.74.250.144:443 SYN_SENT   
18:27:25 10.17.130.19:20973 124.74.250.144:443 SYN_SENT
</code></pre><p>通过上面这些信息，你能很简单地看到，TCP重传是发生在哪些服务器（IP地址）之间，哪些业务上（服务端口），以及为什么会重传（SYN_SENT）。你看，这样效率不就高了吗？具体我们是怎么做的，后面咱们课程里我会详细和你说。</p><p>其实在我看来，对于类似TCP重传这样复杂稳定性问题的定位，除了从开发⼈员的视⻆来分析外，更是需要能够从系统、从内核的视⻆来分析，只有这样，你才能够追本溯源、一劳永逸地解决问题。</p><p>而大家之所以觉得这些问题难，本质上还是对Linux内核理解不到位。比如说，我接触过的业务开发者们，基本上都被业务的性能毛刺困扰过，但是很多人在分析这些性能毛刺时，只能分析到哪些系统调用引起的毛刺，而一些业务专家，却可以再往底层看是什么系统资源引起的业务毛刺。再比如，当发生TCP重传时，有人可以从tcpdump里面的信息看到是哪个TCP连接进行重传，然而高手们却可以通过这些信息看到为什么会发生重传。</p><p>能够深入到Linux内核层分析问题的这些人，<strong>他们看问题能直击本质，定位、分析问题的能力强，往往能解决别人解决不了的问题</strong>，所以他们基本上也是各自领域的翘楚。</p><p>然而，大部分做应用的开发者往往会忽略对Linux内核的学习，这并不难理解，毕竟本职工作更多是在业务代码的优化和调配上，在互联网公司普遍“996”工作的大环境下，很难有精力和时间去深入到内核层面学习。而且，Linux内核本身就是很复杂的，这种复杂度不仅让应用开发者望而却步，也让很多内核初学者知难而退。</p><p>就拿我自己来说吧，我为了学习好Linux内核，就啃过非常多中英文的技术书籍，比如：</p><ul>
<li>为了理解应用程序是如何使用Linux内核的，我把<a href="https://book.douban.com/subject/25900403/">《Unix环境高级编程》</a>这本书读了很多遍；</li>
<li>为了掌握系统体系结构，我仔细读完了<a href="https://book.douban.com/subject/26912767/">《深入理解计算机系统》</a>和<a href="https://book.douban.com/subject/20271450/">《支撑处理器的技术》</a>；</li>
<li>为了理解应用程序在Linux上是如何编译的，我深入阅读了<a href="https://book.douban.com/subject/1787854/">《An Introduction to GCC》</a>以及<a href="https://book.douban.com/subject/1436811/">《Linkers and Loaders》</a>；</li>
<li>为了掌握Linux设备驱动的运行原理，我阅读了<a href="https://book.douban.com/subject/1723151/">《Linux设备驱动程序》</a>这本书，然后，我才开始深入阅读<a href="https://book.douban.com/subject/2287506/">《深入理解Linux内核》</a>这本Linux内核开发者的入门书籍；</li>
<li>在学习过程中，为了更好的掌握内存子系统，我阅读了<a href="https://book.douban.com/subject/1865724/">《深入理解Linux虚拟内存管理》</a>；</li>
<li>为了掌握网络子系统，我阅读了<a href="https://book.douban.com/subject/1088054/">《TCP/IP Illustrated, Volume 1: The Protocols》</a>以及<a href="https://book.douban.com/subject/3397220/">《TCP/IP Architecture, Design and Implementation in Linux》</a>；</li>
<li>为了熟悉其他操作系统的设计原理，我阅读了<a href="https://book.douban.com/subject/1456525/">《The Design and Implementation of the FreeBSD Operating System》</a>来学习FreeBSD；</li>
<li>为了更好的分析Linux内核问题，我读了<a href="https://book.douban.com/subject/6799412/">《Debug Hacks : 深入调试的技术和工具》</a>和<a href="https://book.douban.com/subject/24840375/">《Systems Performance : Enterprise and the Cloud》</a>。</li>
<li>为了更好的和国外开发者做交流，我又阅读了大量的英文原版书籍来提升自己的英语水平，上面我列的这些书名为英文的就是其中部分原版技术书籍。</li>
</ul><p>不可否认，如果想系统学习Linux内核，成本是非常高的。不过话说回来，如果你不是内核开发者，也确实没有必要去搞懂它的每个细节，去掌握它的每一个机制，去理解它所有的优秀设计思想。</p><p>在我看来，一个优秀的软件，或者一个优秀的代码，<strong>存在的本质是为了解决我们遇到的实实在在的问题，更好地满足我们实际的需求。</strong></p><p>也就是说，你能通过掌握的Linux内核知识，解决实际应用层的问题就够了，这也是我开设这门课的初衷。我希望能把自己多年的Linux内核学习和实践经验，通过<strong>“解决问题，满足需求”</strong>的方式传递给你，让Linux内核更好地服务你的应用程序。</p><p>为了更好地达到这个目的，我会从一些生产环境中比较常见的问题入手，带你去了解：你的应用程序是怎么跟系统资源打交道的？你的业务类型应该要选择什么样的配置才会更好？出了棘手的问题该如何一步步地去排查？</p><p>那么，从系统资源的维度，我们需要关注的问题可以分为四类，分别是磁盘I/O、内存、网络I/O、CPU。那在这系列课程中，我会带你从这四大类中的典型问题入手，深入学习其中的：<strong>Page Cache管理问题、内存泄漏问题、TCP重传问题、内核态CPU利用率飙高问题</strong>。这也对应着我们课程的四个模块。</p><p>掌握了这四类典型问题以及其分析思路，你会对磁盘I/O、内存、网络I/O和CPU这四类服务器上最基础的资源有更加深入的理解，在遇到其他问题时也能够触类旁通，从此再也不用回避一些棘手的系统问题。</p><p>为了方便你循序渐进地学习，我们的每个模块都会按照基础篇、案例篇和分析篇的方式来呈现。</p><p>在<strong>Page Cache管理</strong>这个模块中，我会重点分析如何更好地利用Page Cache来减少无谓的I/O开销，Page Cache管理不当会引起的一些问题，以及如何去分析和解决这类问题。</p><p>在<strong>内存泄漏</strong>这个模块中，我会重点分析应用程序都是如何从系统中申请内存以及如何释放的。通过内存泄露这类案例来带你了解应用程序使用内存的细节，以及如果内存使用不当会引发的一些问题。当然，我也会带你去观察、分析和解决这类问题。</p><p>在<strong>TCP重传</strong>这个模块中，我会重点分析TCP连接的建立、传输以及断开的过程。这个过程究竟会受哪些配置项的影响？如果配置不当会引起什么网络问题？然后我会从TCP重传这类具体案例出发，来带你认识你必须要去掌握的一些网络细节知识，以及遇到网络相关的问题时，你该如何去分析和解决它。</p><p>在<strong>内核态CPU利用率飙高</strong>这个模块中，我会分析应用程序该如何高效地使用CPU，以及哪些情况下会导致CPU的使用很低效：比如内核态CPU利用率过高就是一个很低效的表现。那么，针对内核态CPU利用率高的这个案例，我会侧重讲解哪些Linux内核的特性或者系统配置项会引起这种问题，以及如何分析和解决具体的问题。</p><p>在每个模块的最后，我都会总结一下这些常见问题的一般分析思路，让你在面对类型问题时能够有一个大致的分析方向。</p><p>当然，这个课不仅仅是针对应用开发者和运维，对于内核开发者，特别是不那么资深的内核开发者也会很有帮助，它可以帮助你更好地理解业务。Linux内核本质上是给业务服务的，理解好了业务，你才能更好地实现内核特性。就像我给Linux内核提交一些patch时，maintainer们总是会喜欢问这个问题：“What is your real life usecase ?” 结合业务，不盲目设计，也不要过度设计，这是每一个Linux内核开发都需要谨记的。</p><p>这个课程，<strong>除了教你如何更好地学习Linux内核之外，也会带你批判性地来看Linux内核。</strong></p><p>Linux内核从Linus在1991年发布第一版开始，迄今已发展了近30年，代码行数也从最开始的1万行代码发展到了现在的几千万行代码，这么复杂甚至有点臃肿的工程，肯定是存在Bug的，而且也存在很多糟糕的设计。所以，你在平时工作中遇到的很多费解的现象， 也有可能是Linux内核缺陷，我们在这个课程里也会带你认识这些随处可见的缺陷。</p><p>在你遇到有些疑惑的地方，或者你觉得Linux内核哪里不够好时，你可以大胆地去质疑它，去向精通Linux内核的人求助，或者向Linux社区求助，正是因为你的这些需求，我们才能把Linux内核建设的更好更强大。虽然Linus本人是个稍微有点脾气的人，但是整体上Linux社区是很友好的，不论你遇到什么样子的难题，总有人能够解答你的疑问，甚至引导你该如何更好地改进Linux内核。</p><p>最后，我来大概介绍一下我在Linux内核领域的工作经历吧。我从2010年开始，在华为正式从事Linux内核的开发工作，然后又经历了外企（Juniper Networks），也经历了互联网企业（蘑菇街）以及现在所在的某知名互联网企业。</p><p>不同的企业风格，不同的业务场景，但是它们对Linux内核都有着相同的诉求：<strong>更好更稳定。</strong></p><p>这些不同的业务场景，也丰富了我对Linux内核的认识，我也逐渐感受到Linux内核在处理更多以及更新的业务场景下的不足之处，所以我也慢慢地参与到Linux内核社区中来改进Linux内核。</p><p>我目前主要是活跃在Linux内核的内存管理子系统( linux-mm@kvack.org )，如果你有关注这个邮件列表的话，会经常看到我的名字(Yafang Shao &lt; laoar.shao@gmail.com &gt;)。如果你在工作中遇到Linux内核相关的问题，也可以直接发送邮件给我，不过因为平时工作要“996”，我大概率只能在周末才会有精力来回复你。当然，也欢迎你在这个课程下留言提问，我看到都会第一时间处理。</p><p>在最后，我再次温馨地提醒你一下，Linux底层知识的学习并不是一蹴而就的，想要很好地掌握它是会花很多时间的。但是，如果有经验丰富的人带你来学习，你的学习时间会大大缩短，你的学习成本也会降低很多。我相信，通过这个系列课程的学习，你不仅可以很好地掌握必备的Linux基础知识，也能够从中学习到很多解决实际问题的技巧，以及避免去踩很多已经被别人踩过的坑。</p><p>好了，最后欢迎你与我讨论，你现在的工作中有遇到哪些困惑吗？你对Linux内核理解到什么程度了呢？你可以把自己当下的起点或者疑惑记录下来，等全部学完这个课程再来回顾，相信你会有不一样的体会。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/99/4a/bdf26d5c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>石头汤</span>
  </div>
  <div class="_2_QraFYR_0">当我看到邵老师也要996的时候，手里的砖忽然变轻了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-18 09:28:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/5e/96/a03175bc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>莫名</span>
  </div>
  <div class="_2_QraFYR_0">TCP重传推荐使用bcc tcpretrans工具。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-17 23:46:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/9a/83/0802f4e7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_lucky_brian</span>
  </div>
  <div class="_2_QraFYR_0">前几天排查一个MySQL环境的问题，问题现象是运行久了之后会出现：有那么几秒钟内，CPU sys值冲高，很短时间内从10变成了80甚至90，随之还会出现tcp断链，以及磁盘性能下降。<br>最后的发现，出现这个问题的时候，都是在free内存只剩总共的1％时，而且sys变高的几秒内，伴随着page cache的清理。<br>所以，推测是free内存不足引起page cache清理，然后清理的优先级比较高，导致其他io包括网络和磁盘的服务质量下降，引发了断链。<br><br>老师看看这个结论对吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，在回收pagecache的过程中是比较容易导致这些现象的。sys高就是它的一个外在表现。占用了较高cpu，不仅抢占了其他业务的cpu使用，而且如果业务本身在进行回收，也会产生阻塞。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-28 22:22:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/90/8f/9c691a5f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>奔跑的码仔</span>
  </div>
  <div class="_2_QraFYR_0">本人是一名嵌入式linux程序猿，平时主要接触的是内核构建,裁剪以及一些设备驱动，再就是应用软件的设计和实现，不过也是嵌入式环境的。感觉这门课中设计的大多是linux在互联网方面的应用，不知道对于嵌入式linux 方向，咱们这门课适合吗？谢谢🙏</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-17 22:08:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/34/df/64e3d533.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>leslie</span>
  </div>
  <div class="_2_QraFYR_0">系统的东西接触不少：追的最猛的应当是成哥的东西-专栏书籍被我狗皮膏药的成手册用了。<br>太多的时候碰到人说我程序或者应用慢，可是当我提及系统时客户会觉得简单且不可能却又不会去否认这确实是一方面事实存在的原因。系统慢这条坑其实越挖越深，工具、原理、网络，现在只能挖内核去整体思考和理解，从而不要人家来句你系统慢就、、、</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 系统性的分析问题是必要的，系统性的分析问题要建立在你对具体模块的理解深度，如果各个模块都理解不深，其实是很难系统性的来分析问题。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-31 22:31:22</div>
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
  <div class="_2_QraFYR_0">“能够深入到 Linux 内核层分析问题的这些人，他们看问题能直击本质，定位、分析问题的能力强，往往能解决别人解决不了的问题，所以他们基本上也是各自领域的翘楚。”<br>-----------------------------------------------<br>成为行业翘楚的人，解决别人解决不了的问题。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-21 15:30:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/83/c9/0b25d9eb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>NullPointerException</span>
  </div>
  <div class="_2_QraFYR_0">老师能分享下内核学习路线吗，不知道从哪里入手，学什么？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-10 13:07:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/25/2b/ca/71ff1fd7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>谁家内存泄露了</span>
  </div>
  <div class="_2_QraFYR_0">道理我都懂，为什么你的发量是这样？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 其实有很多白头发了:(</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-03 05:28:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/3e/3c/fc3ad983.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>佳伦</span>
  </div>
  <div class="_2_QraFYR_0">其他的内核课程都是基于内存管理，文件系统，进程调度这些知识框架分类的。而本课程别具一格，从实际场景出发，从实践到理论，这样更容易理解，学习动力和效率也更高，👍</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-25 16:54:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/df/6c/5af32271.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Dylan</span>
  </div>
  <div class="_2_QraFYR_0">真是惭愧，《深入理解 Linux 内核》这本书买了快10年了，到现在还是没看完，停留在内存管理和进程调度那部分~~</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 想要了解内核 这本书还是需要沉下心来好好看看</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-02 21:13:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/41/27/3ff1a1d6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>hua168</span>
  </div>
  <div class="_2_QraFYR_0">老大，像我们运维的，没有学过内核，会搭建、shell&#47;python编程，这类的。学习这门课还需要学习什么为基础吗？<br>要学C语言、数据结构和算法之类不？你上面提到的那些要不要学？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-19 15:44:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>福利社</span>
  </div>
  <div class="_2_QraFYR_0">请教老师，我发现clock_gettime(MONOTONIC)一秒内时间经常比系统时间快1秒或6，7秒，这个时间是什么原理？可能是什么原因造成的？开发的虚拟机和容器环境，物理机没试过。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-22 20:14:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2b/8a/09/adf4a438.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>孙国庆</span>
  </div>
  <div class="_2_QraFYR_0">很多内存管理的知识看了很多遍，还是觉得没有完全理解</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-14 10:21:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/ac/96/ea51844f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Frère Jac</span>
  </div>
  <div class="_2_QraFYR_0">我想了解什么情况下系统内核可以平滑升级，什么情况下不能。现在系统的内核版本太多了，有时候真的很难决策。比如现在系统使用的是redhat 6.7 的版本，但是因为应用版本升级原因，需要升级到7.x以上才行，之前记得有过挂载7.x镜像，然后建立本地yum源，可以升级过去，这次却不行，依赖项太多。退一步讲内核升级可以只升级必要的内核和关键的依赖</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 通常而言，内核更新的首选项是backport需要的特性，如果backport成本太高，才需要考虑升级内核。如果升级内核就可以解决，那就不要升级操作系统，只有相应的userspace需要配套更新时才会考虑升级操作系统。<br>升级操作系统后，兼容性测试验证很重要。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-11 08:57:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/ee/28/c04a0c83.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小炭</span>
  </div>
  <div class="_2_QraFYR_0">学好系统底层知识，可上可下。是一项值得长期可投入的事情。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对的 底层知识还是要尽量去搞明白</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-20 12:39:23</div>
  </div>
</div>
</div>
</li>
</ul>