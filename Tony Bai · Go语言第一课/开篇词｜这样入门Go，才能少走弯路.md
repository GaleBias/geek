<audio title="开篇词｜这样入门Go，才能少走弯路" src="https://static001.geekbang.org/resource/audio/a1/00/a115d707e48834b69c7eba62d1c20200.mp3" controls="controls"></audio> 
<p>你好，我是白明（英文名：Tony Bai），欢迎你和我一起学习Go语言。</p><p>我现在在一家初创企业东软睿驰工作，是一名车联网平台的架构师，同时我也是技术博客tonybai.com的博主、GopherChina大会讲师。</p><p>从2011年开始我便关注了Go语言，是Go语言在国内的早期接纳者。那个时候，离Go开源还不过两年，没有人想到它会成长到今天这样，成为后端开发的主流语言之一。</p><p>在对Go长达十年的跟随和研究中，我沉淀了很多个人的经验和思考。我也希望通过这门课，跟你分享我学习和使用Go语言的一些心法。</p><h2>我与Go的这十年</h2><p>2011年，一次偶然的机会，我非常幸运地看到了Go语言之父Rob Pike的Go语言课程<a href="https://www.cs.cmu.edu/afs/cs.cmu.edu/academic/class/15440-f11/go/doc/GoCourseDay1.pdf">幻灯片</a>。当时我正经受着C语言内存管理、线程调度和跨平台运行等问题的折磨，看到Go语言的语法清新简洁，还支持内存垃圾回收、原生支持并发，便一见钟情。</p><p>我是个对编程语言非常“挑剔”的人，这跟我从事的方向有关。十多年来，我一直在电信领域从事高并发、高性能、大容量的网关类平台服务端的开发，这两年也进入了智能网联汽车行业。由于长期从事后端服务开发，我涉猎过很多后端编程语言。</p><p>我曾深入研究过C++，短暂研究过Java、Ruby、Erlang、Haskell与Common Lisp，但都因为复杂度、耗资源、性能不够、不适用于大规模生产等种种原因放弃了。</p><!-- [[[read_end]]] --><p>而且，如果你对我所在的行业有所了解，你可能会知道，我参与开发的系统对性能十分敏感。我也曾长期使用C语言作为生产语言，同时使用Python开发各种辅助工具。但是，C语言的生产效率不高，各种陷阱也很多，而Python开发效率确实很高，但性能又不好。</p><p><strong>难道就没有一门相对“完美”、符合我使用需求的编程语言吗？这个时候，Go来了。</strong></p><p>但在我开始接触和学习Go的2011年，Go语言还未发布Go 1.0稳定版本，还处于“周发布（weekly release）”的状态。我还记得我接触的第一个Go版本还是<a href="https://github.com/golang/go/tree/release-branch.r60">release.r60</a>，也就比Rob Pike的Go课程里使用的版本稍新一些。</p><p>但这个时候的Go版本，存在着很多不尽如人意的地方，尤其是GC延迟比较大，成为了Go语言上生产环境的最大障碍。</p><p>虽然Go早期版本无法上生产环境，但我却一直紧跟Go语言的演化进程。</p><p>从<a href="https://tonybai.com/2014/11/04/some-changes-in-go-1-4/">Go 1.4版本</a>开始，每当Go发布一个大版本，我都撰文分析这个大版本中Go语言的主要变化。这一系列文章至今仍在继续，未来也将持续进行下去。</p><p>特别是在 Go 1.5版本实现自举、Go 1.11版本解决Go包依赖问题后，Go语言逐渐成熟，我也逐渐尝试在生产中使用Go。从开始替代Python编写一些辅助工具，到编写一些网关所需的网络协议，我发现Go都可以完美适用。</p><p>一直到近些年，Go替代了C、Python，成为了我的第一生产语言。我开始直接使用Go编写生产系统，诸如短信网关、5G消息网关、MQTT网关，还有API网关等等。事实证明，Go语言不仅生产效率高，其应用的执行效率也完全能满足要求。</p><p>从2019年开始，我将自己每天阅读到的Go社区的优秀技术资料整理成公开的<a href="https://github.com/bigwhite/gopherdaily">Gopher日报</a>供大家参阅。在国内Go社区中，Gopher日报得到了圈里许多同学的欢迎。</p><p>紧跟Go演进十年的我，已经将Go语言的点点滴滴深深地烙印在大脑中。</p><h2>推荐你入坑Go的三大理由</h2><p>如果说十年前的我是因为“一见钟情”瞬间入坑Go，那么在十年后的今天，我们应该做的是系统地、认真地思考一下为什么要选择学习Go。</p><p>我想了想，我会从这三个角度建议你现在开始学习Go语言。</p><p><strong>第一个理由：对初学者足够友善，能够快速上手。</strong></p><p>十多年来，业界都公认：<strong>Go是一种非常简单的语言</strong>。到底有多简单呢？在2011年，我从一个C语言程序员的身份开始学习Go，使用Rob Pike的Go教程，我在一天之内就学完了Go的全部语法，一周内就可以编写一些简单、实用，而且质量不低的小程序了。</p><p>而且，跟现在很多逐渐添加各种特性的语言相比，Go不仅一开始简单，直到现在也都保持“简单”。Go的设计者们在发布Go 1.0版本和兼容性规范后，似乎就把主要精力放在精心打磨Go的实现、改进语言周边工具链，还有提升Go开发者体验上了。演化了十多年，Go新增的语言特性也同样是“屈指可数”。</p><p>正因为如此，<strong>作为静态编程语言的Go已经将入门门槛，降低到和动态语言一个水平线上了。</strong></p><p><strong>第二个理由：生产力与性能的最佳结合。</strong></p><p>Go的简单和对初学者的友善可以让更多的开发者走进Go语言大门，但要让更多开发者留在Go语言世界，Go还需体现出自己的核心竞争力。这个核心竞争力就是，<strong>Go语言是生产力与性能的最佳结合</strong>。</p><p>如果你熟悉的是静态语言，那你刚好就是Go最初的目标用户。Go创建的最初目的，就是构建流行的、高性能的服务器端编程语言，用以替代当时在这个领域使用较多的Java和C++。而且，Go也实现了它的这个目标。</p><p>Go语言的性能在带有GC和运行时的语言中名列前茅，与不带GC的静态编程语言（比如C/C++）之间也没有数量级的差距。在各大基准测试网站上，在相同的资源消耗水平的前提下，Go的性能虽然低于C++，但高出Java不少。</p><p>如果你熟悉的是动态语言，那也完全不用担心。Go的大部分早期采用者，就来自动态语言程序员群体，包括Python、JavaScript、Ruby和PHP等语言的使用群体。因为和动态语言相比，Go能够在保持生产力的同时，大幅度提高性能。比如，全球知名的非营利教育组织可汗学院从2019年末开始，就将其在线教育平台的实现<a href="https://blog.khanacademy.org/half-a-million-lines-of-go/">从Python迁移到了Go</a>。虽然Go代码行数要多于Python，但他们收获了近10倍的性能提升。</p><p>如果你立志或者已经上手云开发，那你就更应该马上开始学习Go语言。现在，Go已经成为了云基础架构语言，它在云原生基础设施、中间件与云服务领域大放异彩。同时，GO在DevOps/SRE、区块链、命令行交互程序（CLI）、Web服务，还有数据处理等方面也有大量拥趸，我们甚至可以看到Go在微控制器、机器人、游戏领域也有广泛应用。</p><p><strong>第三个理由：快乐又有“钱景”。</strong></p><p>Go最初的目标，就是重新为开发人员带来快乐。这个快乐来自哪里呢？相比C/C++，甚至是Java来说，Go在开发体验上的确有很大提升。笼统来说，这个提升来自于简单的语法、得心应手的工具链、丰富和健壮的标准库，还有生产力与性能的完美结合、免除内存管理的心智负担，对并发设计的原生支持等等。而这些良好的体验，恰恰是你写Go代码时的快乐源泉。</p><p>当然了，我相信你学习和使用Go肯定不是为了自嗨。运用Go体现自身价值，赢得理想职位才是最终目标。在十年后的今天，无论是在国内还是国外，无论是在大厂还是初创小公司，Go都有着广泛的应用。Go语言人才越来越抢手，对他们的争夺也日益激烈。</p><p>有报告表明，在腾讯、字节跳动、Uber等许多公司，Go都是极其受欢迎，在字节跳动、Uber内部甚至已经成长为主力语言。</p><p>更何况，相对于C、C++以及Java等主流语言，Go语言人才目前仍处于蓝海阶段。根据<a href="https://insights.stackoverflow.com/survey/2021">stackoverflow 2021调查报告</a>的结果，仅考虑主流语言的话，Go语言平均薪水位于头部位置。</p><p><img src="https://static001.geekbang.org/resource/image/64/da/649823d58098499226bdc8d0058103da.png?wh=1694x1300" alt="图片"></p><p>这还仅仅是以欧美开发人员调查数据为主的计算结果。在Go更加火爆的国内，Go的平均薪水水平位次可能还要更高。</p><h2>怎样学才能少走弯路？</h2><p>看到Go语言对初学者如此友好，又是生产力与性能结合得最好的语言，写起来还能体会到快乐。关键是当前国内外互联网大厂、初创小厂也都广泛接纳并应用Go，就业“钱景”极佳！很多人都纷纷投身于Go语言的学习中，但是盲目的“一头热”只会让你多走许多弯路。</p><p>我总结了一下，最常见的无非就是这几个：</p><ul>
<li>“入错行”，从开始到放弃。如果你在最开始缺乏对这门语言的认真评估，盲目投入进去，后期沉没的时间和精力成本都会巨大；</li>
<li>不动手。语言学习不是“纸上谈兵”，要动手去用，并且动手越早效果越好；</li>
<li>用其他编程语言思维编写Go代码。我认为，每门编程语言都有着自己独特的编程思维方式，如果你用其他编程语言的思维方式去写Go代码，那只能“形神皆丧”，无法掌握语言真谛；</li>
<li>没有建立起“设计意识”。编程语言学习的最终目的是写出具有现实实用意义的程序，所以你要培养自己选择适当的语言元素构造程序骨架的能力，也就是“设计意识”。尤其是要弄清楚不同语言元素所在的层次，不然你只能停留在“Hello, World”的世界里。</li>
</ul><p>那么，我们到底要怎么学才能学好Go呢？</p><p>学好Go的前提是能坚持学下去。而要保证持续学下去，做好Go入门学习就至关重要。入门学习就好比一座在建大厦的地基，只有地基坚实、稳固，大厦才可能迎来建成，并耸立云霄的那天。</p><p>那么如何做好入门学习呢？这里告诉你<strong>三个诀窍与五个阶段</strong>。所谓三个诀窍是“<strong>心定、手勤、脑勤”</strong>。什么意思呢？我将这个入门课的学习分为下面五个阶段，我会结合这五个阶段来和你说明这三个诀窍。</p><p><strong>第一个阶段：前置篇，“心定”建立认同感。</strong></p><p>这一部分，我会带你了解Go的前世今生和设计哲学。如果你是有其他语言编程经验的开发人员，你就更应该完成前置篇的学习了。</p><p>这一部分存在的意义就是让你“心定”。所谓“心定”就是为了建立对Go语言的认同感。这种认同感是全方位的，包括对Go语言的设计目标、设计哲学、演化思路，还有社区行为规范等等。只有“心定”，才能避免出现“Hello-and-Bye”的情况，这是学好Go的前提。</p><p><strong>第二个阶段：入门篇，“手勤”多动手实践。</strong></p><p>在这一部分中，我将告诉你在不同平台上安装各种Go版本的方法，带你了解一个Go程序应该长什么模样，看看一些实用Go程序都有哪些语法元素和结构。关于Go的版本，如果我在课程中没有特地说明，那便默认使用的是Go最新的稳定版本，这里请你注意下。</p><p>编程不是“纸上谈兵”。我们最终都是要将编写完的源码提交给计算机编译运行的，因此，我希望你在这部分多动手、多实践。我在入门篇中将会让你拥有“照猫画虎”的能力。只有拥有这种能力，你才能“随心所欲”地动手实践。</p><p><strong>第三个阶段：基础篇，“脑勤”多理解，夯实基础。</strong></p><p>这一部分，我们会围绕着“程序=数据+算法”的逻辑，从变量、常量等基本概念，到数据类型，再到广义的算法，让你可以用Go建立对现实世界的抽象认知，也能明白Go程序运行的基本逻辑。</p><p>在基础篇的结尾，我们会结合已学习的基础语法做一个小练习项目。实践与理论的结合才能达到更好的学习效果。</p><p><strong>第四个阶段：核心篇，“脑勤+”建立自己的Go应用设计意识。</strong></p><p>在这部分，我会跟你介绍Go语言独有或经过较大创新的接口类型与goroutine等并发原语类型，这些语法元素是Go语言的核心。</p><p>Go接口与Goroutine等并发原语类型有一个共同的特点，那就是它们都是可以影响到Go应用程序的结构设计的语法元素。Goroutine等并发原语是Go应用运行时并发设计的基础，而接口则是Go推崇的面向组合设计的抓手，这一动一静共同构成了Go应用程序的骨架。通过这一部分，你就能建立自己的Go应用“设计意识”。</p><p><strong>第五个阶段：实战篇，攻克Go开发的“最后一公里”。</strong></p><p>编程就是要做到学以致用。在掌握了Go语言的基础语法、核心语法并建立起自己的“设计意识”后，我们就是时候应用这些Go语言的特性来解决实际问题了。</p><p>在这部分中，我们将通过一个实战的例子，展示如何做好学习与使用之间的衔接，帮助你走完“使用Go进行生产级开发”这“最后一公里”。</p><p><img src="https://static001.geekbang.org/resource/image/fc/9d/fcf857acac0ec2512de6f9dd77b1a69d.jpg?wh=1920x1080" alt="图片"></p><p>更具体的目录，我也放在了这里，你可以看一下：</p><p><img src="https://static001.geekbang.org/resource/image/61/2e/611962c4a98776cf9c79010ab189552e.jpg?wh=1563x8650" alt=""></p><p>在开篇词的最后，我想说，Go是一门非常优秀的后端编程语言，它简单而不失表达力，兼具高生产力与战斗力（高性能）。它既能给你带去编码的快乐，也能因市场的广泛接受与热捧而提升你的个人价值。</p><p>所以，我衷心地希望你能完成这门课程的学习。希望我的这门课能帮助你将Go语言之路走得更顺畅，早日成长为一名优秀的Go语言开发工程师。</p><p>不要再犹豫了，来和我一起开启Go语言的学习之旅吧。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/27/52/40/db9b0eb2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>自由</span>
  </div>
  <div class="_2_QraFYR_0">Tony Bai 老师，你好，我接触 go 目前已经 3 个月了。在接触 go 一个月后，我就选择跳槽去了一家 go 的公司，我对 go 的发展是坚定不移的肯定，相信它会越来越好。<br><br>我的问题是，阅读完该专栏，我是否可以得到 go 风格的代码编写风格、优雅的 go 编程姿势？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢你对专栏的支持。<br><br>虽然这门课的定位是入门课，而并非进阶课，但我在课程讲解以及go示例代码中都会尽力以native的go代码去呈现。并且课程讲解穿插着一些关于go编码的最佳实践建议，希望你在阅读后能有收获。<br><br>btw，要写出native 的go代码，一定要多读高质量go代码，Go标准库是一个最好的选择。俗话说：&quot;熟读唐诗三百首，不会作诗也会吟&quot;，多读高质量代码，与此有异曲同工之妙。<br><br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-13 17:28:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/5c/da/0a8bc27b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>文经</span>
  </div>
  <div class="_2_QraFYR_0">白老师，Go 实现自举具体是什么意思，是不是用Go语言开发的工具链来编译和执行Go源代码？这个具体是一个什么样的过程，Go语言为什么到了1.5版本才实现自举，是因为这个过程很难吗，为什么自举对Go来说是一个重要的节点？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 和很多主流语言一样，Go语言编译器最初都是由C语言和汇编语言实现的。C语言和汇编实现的Go编译器(记作A)用来编译Go源文件。那么问题来了？是否可以用Go语言自身实现一个Go编译器B，用编译器A来编译Go编译器B工程的源码并链接成最终的Go编译器B呢？这就是Go核心团队在Go 1.5版本时做的事情。                                <br>                                                                          <br>他们将绝大多数原来用C和汇编编写的Go编译器以及运行时实现改为使用Go语言编写，并用Go 1.4.x编译器(C与汇编实现的，相当于A)编译出Go 1.5编译器。这样自Go 1.5版本开始，Go编译器就是用Go语言实现的了，这就是所谓的自举。即用要编译的目标编程<br>语言(Go语言)编写其（Go）编译器。                                             <br>                                                                             <br>这之后，Go核心团队基本就告别C代码了，可以专心写Go代码了。这可以让Go核心团队积累更为丰富的Go语言编码经验，也算是一种“吃狗粮”。同时Go语言自身就是站在C语言等的肩膀上，修正了C语言等的缺陷并加入创新机制而形成的，用Go编码效率高，还可<br>避面C语言的很多坑。                                                            <br>                                                                               <br>在这个基础上，使用Go语言实现编译器和runtime还利于Go编译器以及运行时的优化，Go 1.5及后续版本GC延迟大幅降低以及性能的大幅提升都说明了这一点。这就是自举的重要之处。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-26 17:15:32</div>
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
  <div class="_2_QraFYR_0">学习Go是因为VS Code的Copilot插件支持Go但不支持Java，AI插件写代码不是一般的强</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Copilot插件我还没体验过，如果真的如果所言，那也是Go语言的一大幸事👍。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-14 11:57:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/1b/ba/16f2017d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>老韩</span>
  </div>
  <div class="_2_QraFYR_0">老师，什么样的人适合学 Go 语言，我目前是Java工程师，在一家小公司里。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Go设计之初，其目标是成为一门通用的系统编程语言。这一目标基本上就将go划分到后端编程语言行列。虽然go社区在前端、移动端编程的支持上面都做了很多尝试，比如：gopherjs项目以及go支持编译为webassembly来应对前端开发，再比如gomobile项目（https:&#47;&#47;pkg.go.dev&#47;golang.org&#47;x&#47;mobile）让go也可以在移动端编程占有一席之地，但这么多年下来，go的主力战场还是云原生基础设施、中间件、web&#47;api服务、微服务、命令行应用等等。因此如果你的目标与这些领域重合，那么go是一个很有竞争力的选择。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-14 15:15:17</div>
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
  <div class="_2_QraFYR_0">很早就关注老师的微博了，看到老师几乎每天都在分享关于Go的知识。<br><br>关于Go的入门课程看过很多了，目前认为普遍存在的问题有以下几个：要么过于侧重理论，脱离了实践；要么泛泛而谈，重点内容也没有说清楚；要么就是基于以前的gopath项目管理去讲解的；要么没有结合动态语言的特性来对比Go语言的不同，不能让动态语言的开发者很好的转变过来思维......<br><br>另外，文中这样说：“Goroutine 等并发原语是 Go 应用运行时并发设计的基础，而接口则是 Go 推崇的面向组合设计的抓手，这一动一静共同构成了 Go 应用程序的骨架。”<br><br>怎么理解这里的一动一静呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 没有一门课是完美的，从你的问题来看，我只能说我的专栏能尽可能多的满足你的需求。<br><br>关于“一动一静”，“动”主要指程序的并发设计层面，如何设计去管理和控制goroutine。当程序运行起来后，真正“动”的是一个一个goroutine。而“静”，则是go源码中的实体以及它们之间的耦合关系。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-13 18:13:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/d3/ef/928f6340.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jimmyd</span>
  </div>
  <div class="_2_QraFYR_0">老师 请问go在机器学习算法包括工程这一块 前景如何</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个要实话实说。在机器学习领域，python是当之无愧的老大。但python也有自己的瓶颈，主要是性能相较于静态语言有数量级差距。各个编程语言也都试图争抢python在机器学习领域的份额，包括julia、c++、rust，Go也不例外。但与在云原生领域的投入相比，Go社区在机器学习算法库方向上的投入还不够，但也有一些成果，比较知名的项目包括gonum、gorgonia等。在帮助构建机器学习&#47;深度学习平台层面上，go倒是发挥了更大的作用，比如kubeflow。<br><br>机器学习算法上，python已经形成一家独大之势，其他语言，包括Go都会在自己擅长的领域一起助力机器学习的发展了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-13 21:28:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/65/da/29fe3dde.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小宝</span>
  </div>
  <div class="_2_QraFYR_0">收获了三个诀窍与五个阶段。所谓三个诀窍是“心定、手勤、脑勤”，大道至简，适用于任何语言。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-13 17:37:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/3a/2c/43e31bad.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>逍遥布顽童</span>
  </div>
  <div class="_2_QraFYR_0">终于等到课程上线，卡点刷新，第一时间购买学习。不知道自己是第一个，还是第二个购买的？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-13 17:12:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/27/0f/00/31114441.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Kong</span>
  </div>
  <div class="_2_QraFYR_0">请问中小公司中的Go语言技术栈的岗位多吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: go是生产力与执行效率两方面都有突出表现的语言。这两方面都能给中小公司省下不少money。一线城市接纳新语言的开发者较多，招聘也不再是问题了。因此我觉得一线城市应该不少，这方面具体数据还得看招聘网站。二三线城市这些年go也在拓展地盘。在我地处的东北地区，越来越多小公司选用了go，趋势是好的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-15 09:34:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/76/93/64ed7385.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>arch</span>
  </div>
  <div class="_2_QraFYR_0">tony老师，文中提及go GC很多次，期待补充些golang GC相关的文章哈</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对于任何一门带gc的语言，gc都是高级话题。对于入门第一课，gc不是重点。我这里后续会根据大家需求，考虑是否以加餐形式对gc做系统说明。大家有有关gc问题也可以随时在留言区提问。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-14 16:42:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/40/1e/eb6ff5a1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>夏周一</span>
  </div>
  <div class="_2_QraFYR_0">第一时间买下</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-13 17:59:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/2d/ca/02b0e397.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>fomy</span>
  </div>
  <div class="_2_QraFYR_0">我是Java开发者，没有系统学习过Go语言，希望老师能说一下Java和Go的区别。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这是很大的话题，也是一个极容易“引战”的话题。<br><br>看待这个问题有多种维度，比如从语言语法、生产力、性能、社区活跃度，生态成熟度、发展前景等等。<br><br>语言语法见仁见智，java是不折不扣的面向对象编程语言，就像“java编程思想”一书中说：“一切都是对象”。而go是传统的命令式编程语言，按照go语言family图谱，它的先祖来自C、Pascal、Newsqueak等。语法简单，但谈不上“领先”，就像很多人说的在最近10年出品的编程语言中，go的语法显得有些“土气”，我更喜欢称之为朴实无华。很多人就像我，就是喜欢这种朴实。虽然朴实，但go的表达力并不差哦。<br><br>在生产力方面，目前来看go是要高于java的。<br><br>性能方面，同资源消耗下，go也是要高于java的。另外一点就是即便是新手写go，性能也不会很差。<br><br>社区活跃度方面，两者都是主流语言，java诞生年头多，且是目前企业应用领域的第一语言，其社区自然更好一些。生态成熟度也是如此，现在很难找到一个领域没有java的开源实现。实话说，go在这方面规模还不及java，但是增长速度要更快。<br><br>至于，发展前景，两者都是自己擅长领域的佼佼者，都有不错的前景。go由于处于成长期，蓝海属性更强一些。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-15 07:54:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/50/2b/2344cdaa.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>第一装甲集群司令克莱斯特</span>
  </div>
  <div class="_2_QraFYR_0">Java开发者，最近学习了Golang和Python，了解了这2种语言的设计思想，语法结构，融会贯通！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-04 11:19:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/fb/6a/03aabb63.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Alexhuihui</span>
  </div>
  <div class="_2_QraFYR_0">记得大二的时候看过Go 的语法，现在已经作为Java工程师工作2年了，重学Go ing</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 加油！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-23 11:51:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/cb/df/e72646dd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>多喝热水</span>
  </div>
  <div class="_2_QraFYR_0">老师辽宁人吧</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 哈哈，这都让你听出来了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-16 15:22:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/22/0b/09/202a0483.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>JS</span>
  </div>
  <div class="_2_QraFYR_0">之前自己的基础比较薄弱，有时候遇到一些问题，不太知道如何处理，这次就权当复习了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我个人空闲时也喜欢翻看go spec，也是一种复习。每次看后或多或少都会有些收获。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-14 15:22:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/e5/d6/37a1be71.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>凡</span>
  </div>
  <div class="_2_QraFYR_0">目前是2023年1月，想从客户端iOS转Go，不知道是否可行？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 为什么不行呢？列一列你认为不行的原因。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-01 13:59:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/2e/ca/469f7266.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>菠萝吹雪—Code</span>
  </div>
  <div class="_2_QraFYR_0">二刷开始，记笔记，敲代码，打卡</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-15 20:23:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/64/80/61107e24.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>快乐就好</span>
  </div>
  <div class="_2_QraFYR_0">运维转go开发，有机会入门吗  </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以啊。go很简单。特别适合运维编写一些运维管理工具。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-11 16:52:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/9b/a7/440aff07.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>六维</span>
  </div>
  <div class="_2_QraFYR_0">老师对比说说go和rust</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 想听哪方面：）学习难易？前&#47;“钱”途？😁</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-25 20:03:07</div>
  </div>
</div>
</div>
</li>
</ul>