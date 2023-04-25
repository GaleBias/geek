<audio title="结束语｜和你一起迎接Go的黄金十年" src="https://static001.geekbang.org/resource/audio/52/40/525yy505ebdbd96faebd257655252d40.mp3" controls="controls"></audio> 
<p>你好，我是Tony Bai。</p><p>在虎年春节营造的欢乐祥和的气氛中，我们迎来了这个专栏的最后一节课。</p><p>这个专栏的撰写开始于2021年5月中旬，我使用GitHub仓库管理专栏文稿，这是我的第一次提交：</p><p><img src="https://static001.geekbang.org/resource/image/de/62/de2a005cb6bd57c9b9cb004d2fe20362.png?wh=1068x252" alt="图片"></p><p>从那时开始，我便进入了专栏写作的节奏。从2021年5月到2022年2月，9个月的时间，我洋洋洒洒地写下了20多万字（估计值），写作过程的艰辛依然历历在目。</p><p>你见过凌晨4点你所居住的城市吗？我每周至少要见三次。你每天能睡多长时间呢？4～5个小时是我的常态。在日常工作繁忙，家里带俩娃的背景下，这算是挑战我自己的极限了！但当我看到有那么多订阅学习专栏、认真完成课后思考题以及在留言区留言的同学，我又顿感自己的付出没有白费，我的自我挑战是成功的。</p><p><strong>“老师，你跑题了！”</strong></p><p>不好意思，小小感慨了一下。我们还是回到结束语上来。我想了很久，在这最后一讲里，还能给你说点什么呢？人生大道理？职涯规划建议？轻松的、搞笑的段子？可惜这些我都不擅长。最后，我决定借着这篇结束语，和你聊聊下面这几件事，请耐心地听我道来。</p><h2>专栏回顾与“与时俱进”</h2><p>在下笔写这篇文章之前，我认真回顾了一下这门课的内容，对照着Go语言规范细数了一下，这门课覆盖了绝大多数Go语言的语法点，这为你建立Go语言的整体知识脉络，后续继续深入学习奠定了基础。这也是我在设计这门课的大纲时的一个基本目标，现在这个目标算是实现了。</p><!-- [[[read_end]]] --><p>而且，从同学们的留言反馈情况来看，彻底抛弃GOPATH，把Go Module构建模式、Go项目布局的讲解前置到入门篇中，是无比正确的决定。相信你在学完这些知识点后，即便遇到规模再大的Go项目，也能“庖丁解牛”，快速掌握项目的结构，并知道从何处开始阅读和理解代码，这为你“尽早地动手实践”提供了方便。</p><p>另外，这个专栏对一些语法概念，比如切片、字符串、map、接口类型等进行了超出入门范畴的原理性讲解，也得到了来自同学们的肯定，这也算是这个入门课的吸睛之处。</p><p>不过课程依然存在遗憾，其中最令我不安的就是对“指针”这个概念的讲解的缺失。在规划课程之时，我没有意识到，很多来自动态语言的同学完全没有对“指针”这个概念的认知，我的这个疏忽给一些同学的后续学习带来了困惑。为了弥补这个遗憾，<strong>我会在后面以加餐的形式补充对Go指针的基础讲解</strong>。</p><p>按理说，写完这一讲后，我们的这个专栏就正式结束了。但我们都知道，即将发布的Go 1.18版本将加入泛型语法特性，对于定位为“Go语言第一课”的本专栏来说，对泛型语法的系统讲解肯定是不能缺少的，并且，Go泛型很可能会是Go语法特性的最后一次较大更新了。</p><p>虽然我们已经通过加餐聊过泛型了，但那些还是比较粗线条的，所以在2022年Go 1.18泛型正式发布后，我会补充泛型篇，通过大约3节课给你系统、全面地介绍Go泛型语法的细节。我们的专栏也要做到“与时俱进”！</p><h2>接下来，我该学点啥？怎么学？</h2><p>就像这一讲的头图所写的那样，这节课的结束不是你Go语言学习的终点，而是你深入和实践Go的起点。专栏的留言区也有同学在问：Go应该如何进阶呢？进阶的话我该学点啥呢？</p><p>这里我借用“T字形”发展模式，按<strong>语言深度</strong>与<strong>工程宽度</strong>两个方向，在一幅图中列出Go进阶需要了解的知识与技能点：</p><p><img src="https://static001.geekbang.org/resource/image/e4/d6/e42620d46eb73692d9c5c1479d6244d6.jpg?wh=1920x1047" alt="图片"></p><p>沿着“语言深度”这条线我们看到，在纯语言层面的进阶，我们要学习和理解的知识点还有很多，包括这个专栏没有包含的反射（reflect）、cgo（与C语言交互的手段）、unsafe编程等高级语法点，还有迈向Go高级程序员必要的Go编译器原理、Go汇编、Goroutine调度、Go内存分配以及GC等的实现细节。</p><p>当你掌握这些之后，你就会有一种打通“任督二脉”的感觉，再难的Go语言问题在你面前也会变得简单透明。更重要的是，这会让你拥有一种判断力，可以<strong>判断在什么场合不应该使用Go语言</strong>。《Kubernetes Up＆Running》一书的作者、Google开发人员凯尔西·海托（Kelsey Hightower）曾说过：“如果你不知道什么时候不应该使用一种工具，那你就还没有掌握这种工具”。拥有这种判断力，也代表你真正掌握了Go语言。</p><p>当然，Go语言的进阶同样也离不开工程层面的知识与技能的学习。在上面图中，我将工程宽度分成两大块，一块是Go标准库与Go工具链，另外一块是语言之外的工程技能。这些知识与技能都是你在Go进阶以及Go实践之路上不可或缺的。</p><p>那么知道了学啥后，又该如何学呢？</p><p>其实，这个专栏中我一直强调的“手勤+脑勤”同样适合Go进阶的学习，多实践多思考是学习编程语言的不二法门。</p><p>此外，在进阶学习的过程中，我还要向你推荐一种学习方法，同时这<strong>也是我本人使用的方法</strong>，那就是<strong>“输出”</strong>。如果你对“输出”这个词还不太理解，那么你应该或多或少听说过“费曼学习法”吧？</p><p>费曼学习法是由诺贝尔物理学奖得主理查德·菲利普斯·费曼贡献给全世界的学习技巧。这个学习法中的一个环节就是<strong>以教促学</strong>，也就是学完一个知识点后，用你自己的理解将这个知识点讲给其它人，在这个过程中，你既可以检验自己对这个知识点的掌握程度，而且也可通过他人的反馈确认自己对这个知识点的理解是否正确。而这个学习技巧的本质就是“输出”。</p><p>在如今移动互联网的时代，“输出”拥有了更多样的形式，比如：</p><ul>
<li>学习笔记/博客/公众号/问答/视频直播/音频播客/社群；</li>
<li>开源/内源项目；</li>
<li>内部培训/外部技术大会；</li>
<li>译书/著书。</li>
</ul><p>所有的这些形式都要遵循一个共同点：<strong>公开</strong>，也就是将你的“输出”公之于众，接受所有人的检验与评判。这个过程一旦正常运转起来，可以快速修正你理解上的错误，加深你的理解，加快你的学习，并会敦促你主动优化你后续的输出。形成了良性循环之后，再高深的知识点对你来说也就不是什么问题了。</p><p>不过古人云：“知易行难”，学会“输出”也需要一个循序渐进的过程。尤其是一开始“输出”时，不要怕错，不要怕没人看，更不要怕别人笑话你。</p><h2>Go语言的未来</h2><p>最后我们再来谈谈大家都关心的话题：<strong>Go语言的未来</strong>。</p><p>在我写这一讲的时候，刚刚好著名编程语言排名指数<a href="https://www.tiobe.com/tiobe-index/">TIOBE</a>发布了2022年2月编程语言排名情况，如下图：</p><p><img src="https://static001.geekbang.org/resource/image/71/0b/715a7e555baa58dbf9279b51d95acb0b.png?wh=1916x1486" alt="图片"></p><p>在这期排名中，Go上升到第11位，相较于2021年年底各大编程语言的最终排名，以及2021年2月份的排名都上升了2位。Go语言位次的提升在我的预料之中。TIOBE在1月份发布的2021年年终编程语言排行榜的配文中也认为，除了Swift和Go之外，尚不会有新的编程语言能迅速进入前3名甚至前5名，这也在一定程度上证明了TIOBE对Go发展趋势的看好。</p><p>再老生常谈一下，纵观近十年来的新兴后端编程语言，Go集齐了成为下一代佼佼者需要的所有要素：名家设计（三巨头）、出身豪门（谷歌）、杀手应用（Kubernetes）、精英团队（Google专职开发团队）、百万拥趸、生产力与性能的最佳结合，以及云原生基础设施的头部语言。</p><p>在2021年，为了加强Go社区建设与Go官网改进，Go团队雇佣了专人负责。Go核心开发团队专职人员的数量逐年增多，根据<a href="https://changelog.com/gotime/210">Go核心团队工程总监萨梅尔-阿马尼(SAMEER AJMANI)在之前Go Time的AMA环节中透露的信息</a>，当前Go核心团队的规模已经达到了50人：</p><p><img src="https://static001.geekbang.org/resource/image/81/98/81d5425114f6eed69c78124d3a59ea98.jpg?wh=800x111" alt="图片"></p><p>而且，Go语言在国内的发展也是越来越好。大厂方面，腾讯公司近几年在Go语言方面投入很大，不仅让Go语言成为其公司内部增速最快的语言，腾讯还在2021年发布和开源了多款基于Go开发的重量级产品。</p><p>字节跳动更是国内大厂中拥抱Go语言最积极的公司之一，它的技术体系就是以Go语言为主，公司里有超过55％的服务都是采用Go语言开发的。长期的Go实践让字节跳动内部积累了丰富的Go产品和经验，2021年字节也开启了对外开源之路，并且一次性放出了若干个基于Go的微服务框架与中间件产品，包括kitex、netpoll、thriftgo等。这些开源项目统一放在<a href="https://github.com/cloudwego">https://github.com/cloudwego</a>下面了。</p><p>除了大厂积极拥抱Go之外，小公司与初创公司也在积极探索Go的落地。根据我从圈子里、周边朋友、面试时了解的情况，用Go的小公司/初创公司越来越多了。究其原因还是那句话：<strong>Go语言是生产力与战斗力的最佳结合</strong>。这对小公司/初创公司而言，就是真（省）金（人）白（省）银（机器）啊。 甚至，Go已经渗透到新冠防疫领域，我前不久得知，河北移动支撑的新冠疫情流调系统的后端服务也是用Go实现的。</p><p>2022年，Go语言的最大事件就是3月份<strong>Go 1.18的发布以及Go泛型的落地</strong>。泛型的加入势必会给Go社区带来巨大影响。随之而来的将是位于各个层次的Go包的重写或重构：底层库、中间件、数据结构/算法库，乃至业务层面。这一轮之后，Go社区将诞生有关于Go泛型编码的最佳实践，这些实践也会反过来为Go核心团队提供Go泛型演化与在标准库中应用的素材。</p><p>在我们专栏的第一讲<a href="https://time.geekbang.org/column/article/426282">“前世今生：你不得不了解的Go的历史和现状”</a>中，我曾提到过：<strong>绝大多数主流编程语言将在其诞生后的第15至第20年间大步前进</strong>。按照这个编程语言的一般规律，已经迈过开源第12个年头的Go，很可能将进入自己的黄金5-10年。而2022年就很大可能会成为Go语言黄金5-10年的起点，并且这个标志只能是Go泛型语法的落地。</p><p>按照Go语言的调性，在加入泛型后，Go在语法层面上很难再有大的改变了，错误处理将是最后一个硬骨头，也许在泛型引入后，Go核心团队能有新的解决思路。剩下的就是对Go编译器、运行时层、标准库以及工具链的不断打磨与优化了。到时候，我们就坐收这些优化所带来的红利就可以了。</p><p>经过这些对Go语言当前状态和未来可能演化路线的分析，你是不是对Go的未来更加有信心了呢？</p><p>学习Go语言十余年的我，很庆幸，也很骄傲当初做出了正确的选择。最后，<strong>在Go即将迎来黄金十年的历史时刻，希望你能在Go语言之路上走的更远，并实现你的个人价值</strong>。</p><p><a href="https://jinshuju.net/f/AclglJ"><img src="https://static001.geekbang.org/resource/image/0a/25/0a8b8d870401fa225c607450a916f625.jpg?wh=1142x801" alt=""></a></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/5c/da/0a8bc27b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>文经</span>
  </div>
  <div class="_2_QraFYR_0">感谢白老师的辛苦付出，我昨天开始带着部门的同事二刷这门课程。<br>也入手了白老师的新书《Go语言精进之路》1，2册，进一步学习。<br>十年后，我也会同样庆幸自己在2021年选择转战Go，并且遇到了这个专栏。<br>再次感谢白老师！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢对我的各种支持:)。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-16 11:30:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/0a/a4/828a431f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张申傲</span>
  </div>
  <div class="_2_QraFYR_0">这是我学习过的质量最高、老师最认真负责的专栏之一了，感谢老师的辛苦付出，我自己受益匪浅。期待老师后面的加餐和新的课程~</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谬赞了。感谢支持。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-23 18:35:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/53/a8/abc96f70.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>return</span>
  </div>
  <div class="_2_QraFYR_0">老师真是业界良心， 无私奉献啊。<br>感谢老师的辛勤产出， 受益匪浅。<br>期待老师有精力的时候 在出个进阶的。<br>买了好几个GO专栏， 白老师的 讲的最透彻 关键点讲的最好 非常喜欢。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谬赞了。感谢支持！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-18 09:30:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/24/c4/51/5bca1604.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>aLong</span>
  </div>
  <div class="_2_QraFYR_0">通过此课程还是填补了很多从其他平台、书籍、视频中未提及的的知识点。看到这个课程介绍我最开始不是很想买。因为入门的内容在网上还是很多的，其中视频课程是最多的。我目前已学习36课，已经感觉到很多知识在其他的课程中没有的内容。希望以后老师在进阶或典型实践方向推出系列可能。最后，感谢老师和老师的课程。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢支持！继续加油！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-16 13:15:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/27/af/fd/a1708649.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ゝ骑着小车去兜风。</span>
  </div>
  <div class="_2_QraFYR_0">老师后续会不会考虑再出一个go语言的进阶版课程，我特别期待。真是太喜欢这门课了，其中方法和接口这部分内容解决了我之前太多的难题，对go有了更深一点的认识！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢支持！后续是否有其他课程，目前尚不确定:)</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-01 13:52:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/61/58/7b078879.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Julien</span>
  </div>
  <div class="_2_QraFYR_0">有幸在学习本课程的同时，趁春节期间深入学习了Goroutine 调度原理和细节，今天下午还在公司内部做了技术分享，有一点成就感。这门课程很好，老师讲解很明了，感谢~</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍。感谢支持。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-18 18:19:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/ba/ce/fd45714f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>bearlu</span>
  </div>
  <div class="_2_QraFYR_0">老师学完本专栏，接下来是需要做些什么项目？<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以将你以前用其他语言编写的一些工具移植到go，或是之前想写但没有写的小项目或小工具用go实现一遍。当然最好是在工作中能把go用起来:)。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-17 11:24:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/22/7d/a6/15798bf2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>温雅小公子</span>
  </div>
  <div class="_2_QraFYR_0">完结撒花🎉</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢支持。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-16 15:41:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2b/0a/23/c26f4e50.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Sunrise</span>
  </div>
  <div class="_2_QraFYR_0">首先感谢作者，极客上的关于 Go 的专栏我几乎都买了，这篇 Go 语言第一课是我觉得质量最高买的最值的，老师的两本书也入手了，既然有第一课肯定有第二课吧，希望老师继续后续的课程。。。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-21 15:56:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/cc/97/ae4d1400.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>林枫</span>
  </div>
  <div class="_2_QraFYR_0">作为一个比较挑剔的读者，我认为这个课质量蛮高的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-24 08:44:25</div>
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
  <div class="_2_QraFYR_0">谢谢老师。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-01 00:13:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/9a/63/bfada10f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Asher</span>
  </div>
  <div class="_2_QraFYR_0">点赞，质量很高。老师</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢支持。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-24 14:17:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2c/7b/01/4ab9fba7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>天空</span>
  </div>
  <div class="_2_QraFYR_0">打个卡</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 收到！😁</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-23 08:41:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/90/23/5c74e9b7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>$侯</span>
  </div>
  <div class="_2_QraFYR_0">三刷了，老师能讲讲context吗~</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 最近太忙，后续看看能否以加餐形式补充一下吧，但时间上暂不能保证😜。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-10 13:34:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/7a/ec/19104a37.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>李强</span>
  </div>
  <div class="_2_QraFYR_0">老师讲的很系统，学完有收获，手动点赞！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-14 13:03:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2a/eb/a4/b247e9b7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>独钓寒江</span>
  </div>
  <div class="_2_QraFYR_0">请问为什么多数偏入门的教程都没有讲数据库连接的呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: go与数据库互操作算是go的一个应用了。而且多数语言与数据库操作都比较简单。没啥特殊的，也就没啥可讲的了。难点其实是在数据库侧，多数并不在语言侧。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-11 11:32:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/ec/31/07c1dc6a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>晓鹤</span>
  </div>
  <div class="_2_QraFYR_0">感谢老师，想要继续深入的同学可以买《Go语言精进之路》1，2 册，我看里面是有涉及到一些课程里面没有提到的内容的，比如 reflect, unsafe 包的这部分内容</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-08 07:24:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/1f/14/57cb7926.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>FECoding</span>
  </div>
  <div class="_2_QraFYR_0">go 1.18 已经发布啦，泛型是啥时候更新呀</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 写稿中:) go 1.18正式版发布前，担心泛型规范有变化，一直未下笔。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-24 17:21:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2a/db/74/6ccc9189.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>寿限无</span>
  </div>
  <div class="_2_QraFYR_0">老师的go语言精进之路讲得也很棒</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢支持。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-24 10:51:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/4b/d7/f46c6dfd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>William Ning</span>
  </div>
  <div class="_2_QraFYR_0">刚学完20讲，先来看看有多少人到了最后一课～～ <br>2022-02-16 结课<br>抓紧时间，还不算太晚～～<br>提前谢谢老师～</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 💪</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-17 15:16:38</div>
  </div>
</div>
</div>
</li>
</ul>