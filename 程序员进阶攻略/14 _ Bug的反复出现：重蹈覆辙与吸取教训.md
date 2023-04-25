<audio title="14 _ Bug的反复出现：重蹈覆辙与吸取教训" src="https://static001.geekbang.org/resource/audio/d0/98/d0b58e1b9e1c158f3cb05f3a9504bf98.mp3" controls="controls"></audio> 
<p>Bug 除了时间和空间两种属性，还有一个特点是和程序员直接相关的。在编程的路上，想必你也曾犯过一些形态各异、但本质重复的错误，导致一些 Bug 总是以不同的形态反复出现。在你捶胸顿足懊恼之时，不妨试着反思一下：为什么你总会写出有 Bug 的程序，而且有些同类型的 Bug 还会反复出现？</p>
<h2>1. 重蹈覆辙</h2>
<p>重蹈覆辙的错误，老实说曾经我经历过不止一次。</p>
<p>也许每次具体的形态可能有些差异，但仔细究其本质却是类似的。想要写出没有 Bug 的程序是不可能的，因为所有的程序员都受到自身能力水平的局限。而我所经历的重蹈覆辙型错误，总结下来大概都可以归为以下三类原因。</p>
<h3>1.1 粗心大意</h3>
<p>人人都会犯粗心大意的错误，因为这就是 “人” 这个系统的普遍固有缺陷（Bug）之一。所以，作为人的程序员一定会犯一些非常低级的、因为粗心大意而导致的 Bug。</p>
<p>这就好比写文章、写书都会有错别字，即使经历过三审三校后正式出版的书籍，都无法完全避免错别字的存在。</p>
<p>而程序中也有这类 “错别字” 类型的低级错误，比如：条件<code>if</code> 后面没有大括号导致的语义变化，<code>==</code>、<code>=</code> 和 <code>===</code> 的数量差别，<code>++</code> 或<code>--</code> 的位置，甚至 <code>;</code>的有无在某些编程语言中带来的语义差别。即使通过反复检查也可能有遗漏，而自己检查自己的代码会更难发现这些缺陷，这和自己不容易发现自己的错别字是一个道理。</p><!-- [[[read_end]]] -->
<p>心理学家汤姆·斯塔福德（Tom Stafford）曾在英国谢菲尔德大学研究拼写错误，他说：“当你在书写的时候，你试图传达想法，这是非常高级的任务。而在做高级任务时，大脑将简单、零碎的部分（拼词和造句）概化，这样就可以更专注于更复杂的任务，比如将句子变成复杂的观点。”</p>
<p>而在阅读时，他解释说：“我们不会抓住每个细节，相反，我们吸收感官信息，将感觉和期望融合，并且从中提炼意思。”这样，如果我们读的是他人的作品，就能帮助我们用更少的脑力更快地理解含义。</p>
<p>但当我们验证自己的文章时，我们知道想表达的东西是什么。因为我们预期这些含义都存在，所以很容易忽略掉某些感官（视觉）表达上的缺失。我们眼睛看到的，在与我们脑子里的印象交战。这，便是我们对自己的错误视而不见的原因。</p>
<p>写程序时，我们是在进行一项高级的复杂任务：将复杂的需求或产品逻辑翻译为程序逻辑，并且还要补充上程序固有的非业务类控制逻辑。因而，一旦我们完成了程序，再来复审写好的代码，这时我们预期的逻辑含义都预先存在于脑中，同样也就容易忽略掉某些视觉感官表达上的问题。</p>
<p>从进化角度看，粗心写错别字，还看不出来，不是因为我们太笨，而恰恰还是进化上的权衡优化选择。</p>
<h3>1.2 认知偏差</h3>
<p>认知偏差，是重蹈覆辙类错误的最大来源。</p>
<p>曾经，我就对 Java 类库中的线程 API 产生过认知偏差，导致反复出现问题。Java 自带线程池有三个重要参数：核心线程数（core）、最大线程数（max）和队列长度（queues）。我曾想当然地以为当核心线程数（core）不够了，就会继续创建线程达到最大线程数（max），此时如果还有任务需要处理但已经没有线程了就会放进队列等待。</p>
<p>但实际却不是这样工作的，类库的实现是核心线程（core）满了就会进队列（queues）等待，直到队列也满了再创建新线程直至达到最大线程数（max）的限制。这类认知偏差曾带来线上系统的偶然性异常故障，然后还怎么都找不到原因。因为这进入了我的认知盲区，我以为的和真正的现象之间的差异一度让我困惑不解。</p>
<p>还有一个来自生活中的小例子，虽然不是关于程序的，但本质是一个性质。</p>
<p>有时互联网上，朋友圈中小道消息满天飞，与此类现象有关的一个成语叫 “空穴来风”，现在很多媒体文章有好多是像下面这样用这个成语的：</p>
<blockquote>
<p>他俩要离婚了？看来空穴来风，事出有因啊！<br />
物价上涨的传闻恐怕不是空穴来风。</p>
</blockquote>
<p>第一句是用的成语原意：指有根据、有来由，“空”发三声读 kǒng，意同 “孔”。第二句是表达：没有根据和由来，“空”发一声读kōnɡ。第二种的新意很多名作者和普通大众沿用已久，约定俗成，所以又有辞书与时俱进增加了这个新的义项，允许这两种完全相反的解释并存，自然发展，这在语义学史上也不多见。</p>
<p>而关于程序上有些 API 的定义和实现也犯过 “空穴来风” 的问题，一个 API 可以表达两种完全相反的含义和行为。不过这样的 API 就很容易引发认知偏差导致的 Bug，所以在设计和实现 API 时我们就要避免这种情况的出现，而是要提供单一原子化的设计。</p>
<h3>1.3 熵增问题</h3>
<p>熵增，是借用了物理热力学的比喻，表达更复杂混乱的现象；程序规模变大，复杂度变高之后，再去修改程序或添加功能就更容易引发未知的 Bug。</p>
<p>腾讯曾经分享过 QQ 的架构演进变化，到了 3.5 版本 QQ 的用户在线规模进入亿时代，此时在原有架构下去新增一些功能，比如：</p>
<blockquote>
<p>“昵称” 长度增加一半，需要两个月；</p>
<p>增加 “故乡” 字段，需要两个月；</p>
<p>最大好友数从 500 变成 1000，需要三个月。</p>
</blockquote>
<p>后端系统的高度复杂性和耦合作用导致即使增加一些小功能特性，也可能带来巨大的牵连影响，所以一个小改动才需要数月时间。</p>
<p>我们不断进行架构升级的本质，就在于随着业务和场景功能的增加，去控制住程序系统整体 “熵” 的增加。而复杂且耦合度高（熵很高）的系统，正是容易滋生 Bug 的温床。</p>
<h2>2. 吸取教训</h2>
<p>为了避免重蹈覆辙，我们有什么办法来吸取曾经犯错的教训么？</p>
<h3>2.1 优化方法</h3>
<p>粗心大意，可以通过开发规范、代码风格、流程约束，代码评审和工具检查等工程手段来加以避免。甚至相对写错别字，代码更进一步，通过补充单元测试在运行时做一个正确性后验，反过来去发现这类我们视而不见的低级错误。</p>
<p>认知偏差，一般没什么太好的自我发现机制，但可以依赖团队和技术手段来纠偏。每次掉坑里爬出来后的经验教训总结和团队内部分享，另外就是像一些静态代码扫描工具也提供了内置的优化实践，通过它们的提示来发现与你的认知产生碰撞纠偏。</p>
<p>熵增问题，业界不断迭代更新流行的架构模式就是在解决这个问题。比如，微服务架构相对曾经的单体应用架构模式，就是通过增加开发协作，部署测试和运维上的复杂度来换取系统开发的敏捷性。在协作方式、部署运维等方面付出的代价都可以通过提升自动化水平来降低成本，但只有编程活动是没法自动化的，依赖程序员来完成，而每个程序员对复杂度的驾驭能力是有不同上限的。</p>
<p>所以，微服务本质上就是将一个大系统的熵增问题，局部化在一个又一个的小服务中。而每个微服务都有一个熵增的极限值，而这个极限值一般是要低于该服务负责人的驾驭能力上限的。对于一个熵增接近极限附近的微服务，服务负责人就需要及时重构优化，降低熵的水平。而高水平和低水平程序员负责的服务本质差别在于熵的大小。</p>
<p>而熵增问题若不及时重构优化，最后可能会付出巨大的代价。</p>
<p>丰田曾陷入的 “刹车门” 事件，就是因为其汽车动力控制系统软件存在缺陷。而为追查其原因，在十八个月中，有 12 位嵌入式系统专家受原告诉讼团所托，被关在马里兰州一间高度保安的房间内对丰田动力控制系统软件（主要是 2005 年的凯美瑞）源代码进行深度审查。最后得到的结论把丰田的软件缺陷分为三类：</p>
<ul>
<li>非常业余的结构设计</li>
<li>不符合软件开发规范</li>
<li>对关键变量缺乏保护</li>
</ul>
<p>第一类属于熵增问题，导致系统规模不断变大、变复杂，结果驾驭不了而失控；第二类属于开发过程的认知与管理问题；第三类才是程序员实现上的水平与粗心大意问题。</p>
<h3>2.2 塑造环境</h3>
<p>为了修正真正的错误，而不是头痛医头、脚痛医脚，我们需要更深刻地认识问题的本质，再来开出 “处方单”。</p>
<p>在亚马逊（Amazon），严重的故障需要写一个 COE（Correction of Errors）的文档，这是一种帮助去总结经验教训，加深印象避免再犯的形式。其目的也是为了帮助认识问题的本质，修正真正的错误。</p>
<p>但一旦这个东西和 KPI 之类的挂上钩，引起的负面作用是 COE 的数量会变少，但真正的问题并没有减少，只是被隐藏了。而其正面的效应像总结经验、吸取教训、找出真正问题等，就会被大大削弱。</p>
<p>关于如何构造一个鼓励修正错误的环境，我们可以看看来自《异类》一书讲述的大韩航空的例子，大韩航空曾一度困扰于它的飞机损失率：</p>
<blockquote>
<p>美国联合航空 1988 年到 1998 年的飞机损失率为百万分之 0.27，也就是说联合航空每飞行 400 万次，会在一次事故中损失一架飞机；而大韩航空同期的飞机损失率为百万分之 4.79，是前者的 17 倍之多。</p>
</blockquote>
<p>事实上大韩航空的飞机也是买自美国，和联合航空并无多大差别。它的飞行员们的飞行时长，经验和训练水平从统计数据看也差别不大，那为什么飞机损失率会如此地高于其他航空公司的平均水平呢？在《异类》这本书中，作者以此为案例做了详细分析，我这里直接引用结论。</p>
<blockquote>
<p>现代商业客机，就目前发展水平而言，跟家用烤面包机一样可靠。空难很多时候是一系列人为的小失误、机械的小故障累加的结果，一个典型空难通常包括 7 个人为的错误。</p>
</blockquote>
<p>一个飞机上有正副两个机长，副机长的作用是帮助发现、提醒和纠正机长在飞行过程中可能发生的一些人为小错误。大韩航空的问题正在于副机长是否敢于以及如何提醒纠正机长的错误。其背后的理论依据源自荷兰心理学家吉尔特·霍夫斯泰德（Geert Hofstede）对不同族裔之间文化差异的研究，就是今天被社会广泛接受的跨文化心理学经典理论框架：霍夫斯泰德文化纬度（Hofstede’s Dimensions）。</p>
<blockquote>
<p>在霍夫斯泰德的几个文化维度中，最引人注目的大概就是 “权力距离指数（Power Distance Index）”。权力距离是指人们对待比自己更高等级阶层的态度，特别是指对权威的重视和尊重程度。</p>
</blockquote>
<blockquote>
<p>而霍夫斯泰德的研究也提出了一个航空界专家从未想到过的问题：让副机长在机长面前维护自己的意见，必须帮助他们克服所处文化的权力距离。</p>
</blockquote>
<p>想想我们看过的韩国电影或电视剧中，职场上后辈对前辈、下级对上级的态度，就能感知到韩国文化相比美国所崇尚的自由精神所表现出来的权力距离是特别远的。因而造成了大韩航空未被纠正的人为小错误比例更高，最终的影响是空难率也更高，而空难就是航空界的终极系统故障，而且结果不可挽回。</p>
<p>吸取大韩航空的教训应用到软件系统开发和维护上，就是：需要<strong>建立和维护有利于程序员及时暴露并修正错误，挑战权威和主动改善系统的低权力距离文化氛围，这其实就是推崇扁平化管理和 “工程师文化” 的关键所在</strong>。</p>
<p>一旦系统出了故障非技术背景的管理者通常喜欢用流程、制度甚至价值观来应对问题，而技术背景的管理者则喜欢从技术本身的角度去解决当下的问题。我觉着两者需要结合，站在更高的维度去考虑问题：<strong>规则、流程或评价体系的制定所造成的文化氛围，对于错误是否以及何时被暴露，如何被修正有着决定性的影响</strong>。</p>
<p>我们常与错误相伴，查理·芒格说：</p>
<blockquote>
<p>世界上不存在不犯错误的学习或行事方式，只是我们可以通过学习，比其他人少犯一些错误，也能够在犯了错误之后，更快地纠正错误。但既要过上富足的生活又不犯很多错误是不可能的。实际上，生活之所以如此，是为了让你们能够处理错误。</p>
</blockquote>
<p>人固有缺陷，程序固有 Bug；吸取教训避免重蹈覆辙，除了不断提升方法，也要创造环境。你觉得呢？欢迎你留言和我分享。</p>
<hr />
<p></p>

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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/a4/5a/e708e423.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>third</span>
  </div>
  <div class="_2_QraFYR_0">心得如下：<br><br>1.bug本质上是无法避免的，最好的办法不是不犯错，而是少犯错误，犯错之后尽快改正。最快的成长方法是不退步<br><br>2.bug的反复出现，有三个重要原因，粗心大意，认知偏差，熵增问题<br><br>3.认知偏差是错误的最大来源，思想错误，行动又怎么会正确呢？<br><br>4.熵增问题，是随着复杂增高，而必然有更高概率发生bug<br><br>5.应对这些问题，我们要从各方面吸取教训，其中最重要的是优化方法和塑造环境。<br><br>6.粗心大意，可以通过流程和工具来减少，建议补充代码单元测试<br><br>7.认知偏差，可以通过团队和技术手段来纠偏<br><br>8.熵增问题，通过业界不断迭代更新流行的框架模式来解决。<br><br>9.塑造环境，塑造一个正视错误，改正错误的环境</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍，总结的比我详细😊</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-03 11:51:03</div>
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
  <div class="_2_QraFYR_0">世界是不完美的，人是这个世界的极小部分，人也是不完美的，如果想使人做的事情趋近于完美，那就需要集体的智慧了。之于编程一套好的流程能扼杀许多的Bug，自测、测试（包括：测试环境联调、预发布环境联通等）、代码review，上线前准备、上线中自验、上线后产品及业务验证等如果都做到位了，至少能保证主流程没问题，对于性能有要求的可以加上性能测试。现在我们基本是这么玩的，另外，就是各种监控和日志的加持啦！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 流程本身就是环境的产物，对于软件开发而言，每个公司估计流程都不会完全一样</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-03 08:32:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/05/97/b83ebf66.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>softtwilight</span>
  </div>
  <div class="_2_QraFYR_0">胡大看问题的角度真是独特，而且深刻</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: ^_^，这也是可以培养和训练的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-03 09:14:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/1e/81/c3e541c0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>李正阳Lee</span>
  </div>
  <div class="_2_QraFYR_0">看到塑造犯错环境这节，想起了桥水基金创始人达利欧对待犯错的态度。<br><br>他允许员工犯错，强调每个人应反思错误，不容许一错再错。并在公司内部建立了一个“错误日志”，用来记录每人犯过的错误和造成的不良后果，这样可以追根溯源，系统化的解决问题。<br><br>我们技术人也可以建立自己或团队的“错误日志”，在错误中成长。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 恩，不错的主意</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-04 10:10:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/f4/ee/8abb67ec.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Frank.w</span>
  </div>
  <div class="_2_QraFYR_0">总是粗心大意导致出现bug，甚至一些低级的条件语句发生错误。<br>自我总结：<br>1. 做好单元测试。<br>2.做好代码走查，在写完代码后，在今进行从新构思审查。<br>3.对于代码改动影响的所有代码进行测试。<br>4.提交测试，同时提交改动范围，改动代码所影响的所有功能。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可复用的单元测试会拯救未来的你^_^</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-28 10:04:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/5b/6f/113e24e6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>阿信</span>
  </div>
  <div class="_2_QraFYR_0">程序之术这部分 笔记 (期待支持markdown)<br><br># 总览<br>主要是讲解程序员技能的锤炼。<br>主线：设计、开发、bug处理 (按正常的功能开发流程介绍的)<br>* 设计<br>    * 认知设计。架构与实现，他们的连接与分界；框架和模式的区别<br>    * 如何来设计。从哪些维度来思考(多维度视图)<br>* 编码<br>    * 工业级编码包含哪些内容 (功能、控制、运维)<br>    * 编码的两种方式：粗放和精益<br>    * 编码的态度：炫技和克制<br>    * 编码的三阶段：调试、编码、运行<br>* bug处理<br>    * bug按特性进行的分类以及各分类介绍。空间特性和时间特性<br>    * bug出现的原因分析，以及降低bug的方法<br><br><br><br># 05 架构与实现：它们的连接与分界<br>架构：做的是顶层设计，技术、框架选型，边界划分。熵尽量小。设计和实现做取舍<br>实现：交付代码，尽量简。围绕架构做的程序时间。<br>连接：<br>架构师：关注系统边界、关键点、战略性细节点<br><br># 07 多维与视图：系统设计的思考维度与展现视图<br>理解或表现系统的视角。<br>组成视图：描述组成系统的子系统、服务、组件。便于总体了解系统服务组成，以及其职能。服务切分原则：高内聚，职能单一；正交化，同一个功能只有一个服 务提供<br>交互视图：系统组成部分中各部分的依赖、被依赖关系。整体的流转。<br>部署视图：系统如何来部署的。关于服务、中间件、使用者之间的网络传输，确认IO瓶颈，从高维度看系统设计的合理性<br>流程视图：功能逻辑流<br>状态视图：状态的变化流<br>我的理解：组成、交互、部署视图，适合描述顶层设计。流程、状态视图，适合描述具体的功能<br><br># 08 代码与分类：工业级编程代码的分类与特征<br>总结：<br>对于最终需要交付运行的程序代码，<br>从代码的用途(or使用场景)上将作者代码分为三种：功能代码、控制代码、运维代码。<br>功能代码，满足于用户需求，为实现特定的业务功能而开发。<br>控制代码，对代码执行流的控制，如并行、异步、限流、熔断、超时控制等。RPC、中间件、代理服务器等，应该是对代码执行流的控制。控制代码的需求，是从众多(已有的或预见的)功能开发中提炼而来，基于一些共性的特征抽取。源于功能而高于功能。<br>运维代码，用于解决运行过程中出现的问题，或者为解决问题提供必要的信息。<br><br>结合工作，我们做的防重组件，是控制代码。基于pinpoint实现链路跟踪，是对业界已有的运维代码成果的使用<br><br># 09 精益与粗放，编程的两种思路与方式<br>介绍编程的两种思路，完美型和现实型。<br>看这篇文章，脑海中想到了两件事情，一是卖油翁的故事；二是态度(吴军老师写的)一书中提到的一些观点：最好是更好的敌人(或者说进步一点比什么都不做好)、做事情时境界要高。<br>陶器制作第一组的同学，可以说是熟能生巧；陶器制作第二组的同学以及课程设计的同学，境界是比较高的，但在其事情落地上执行方式有点问题，因为没有找到完美的方案而踌躇不前，最终没有输出满意的成果。<br>套用到我们自身的工作，以及绝大多数程序员身上，大量的开发(+用心理解)可以提升我们的水平，我们要求输出的成果有deadline，设计时我们格局可以大一些，考虑后来的扩展，实现时可以分步来执行，多次的改进让我们朝着目标前进，甚至超过原有心中的完美方案。<br><br># 10 炫技与克制：代码的两种味道与态度<br>炫技：能简单处理的采用了复杂的方式<br>克制：不随性。第一层意思是采用了简单的方案处理合适的问题；第二层意思是看到不好的代码、设计，克制立刻修改的冲动，了解整个逻辑后，在合适的时机进行重构。<br><br># 11 三阶段进化：调试，编写与运行代码<br>介绍的是两种写代码的方式：一是走一步算一步，边写边调；二是先整理清楚整个思路，然后再动手。<br><br><br># 12、13、14 bug分析<br>技术性bug，根据特征分为两类：<br>空间：环境过敏<br>时间：周期规律<br>环境过敏，像人到一个新环境过敏一样，程序运行在不同的环境，因为环境的差异而导致程序运行遇到问题。如磁盘故障，网络带宽低等，原本(在次品正常、网络带宽高机器上)运行正常的程序，运行出现了问题。<br><br>bug出现的原因分类：<br>粗心大意<br>认知偏差，和理解需求出现了偏差<br>熵增问题 (系统复杂度变高)<br><br>降低bug出现次数的方法：<br>粗心大意，可以通过流程和工具减少；<br>认知偏差，可以通过团队和技术手段来纠偏<br>熵增问题，重构，如使用业界更好的模式、框架等来降低复杂度<br>塑造环境。塑造一个正视错误、改正错误的环境。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍，学习心得感悟可以快速记录在这里，总结整理还是汇总到一个自己专用的笔记软件方便些</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-13 23:16:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/e1/5c/86606d9c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>湮汐</span>
  </div>
  <div class="_2_QraFYR_0">我和胡老师一样，也被线程池池误导过。我也以为先创建线程到max，然后再进入队列，事实上和我们想的正好相反。而我总结了一下，我们之所以被误导，也是有原因的：数据库连接池就是先创建连接到max，如果没有可用连接时，就等待maxWait时间直到有空闲连接或抛出异常。<br>在学习的时候虽然有“触类旁通”，但是有时候确实会有一些细节不一样，没有去踩过这个坑，自然就会想当然了。<br>出错一直都是正常的，一定要敢于面对自己的错误，并且纠正自己的错误。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，惯性思维也容易导致犯错</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-27 14:11:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/64/04/18875529.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>艾尔欧唯伊</span>
  </div>
  <div class="_2_QraFYR_0">我最近面试遇到这个问题，我就说错了。后来继续解释的时候发现说不通了，回来一看，发现之前理解的都是错的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 所以要多交流才能发现自己的认知偏差，面试也算一种特殊的交流</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-03 18:28:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/64/04/18875529.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>艾尔欧唯伊</span>
  </div>
  <div class="_2_QraFYR_0">线程池那个认知偏差真是一样一样的啊。。。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 听说不少人都掉过这个坑……</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-03 08:23:16</div>
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
  <div class="_2_QraFYR_0">才看了几篇，就能感觉到胡大是个喜欢看书的人</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，第一爱好是读书^_^</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-14 19:14:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/20/39/3168c4ca.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>二木🐶</span>
  </div>
  <div class="_2_QraFYR_0">这篇文章写的真好</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: ^_^</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-27 21:13:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/4e/8b/964b3b80.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>热勇</span>
  </div>
  <div class="_2_QraFYR_0">非常值得反思的文章！！！为自己加油！谢谢老师</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 今天刚碰到一个线上事故，反思中……😢</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-20 08:23:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/8c/5c/3f164f66.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>亚林</span>
  </div>
  <div class="_2_QraFYR_0">嗯 站在更高的角度解决问题</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-11 12:04:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/6e/93/6fef7aaa.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>石头</span>
  </div>
  <div class="_2_QraFYR_0">首先要有一套流程避免大问题，然后对于个人来说有一个收获就是建立COF，不断总结优化提高。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-05 21:01:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/ae/e1/78701ecf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>莫小鹏</span>
  </div>
  <div class="_2_QraFYR_0">今天的文章不错</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: ^_^</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-04 00:26:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/de/ac/f279ac31.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>吴秀華</span>
  </div>
  <div class="_2_QraFYR_0">我承认我是个经常出BUG的人。每次都是粗心大意或未完全了解需求造成的。我正在努力的完善自己的思维和工作态度。在工作中得不到认可和改进意见。在这里，非常感谢峰哥的分享，让我在迷茫中，看到曙光。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: ^_^，加油💪</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-03 16:09:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>！巴甫洛夫的狗</span>
  </div>
  <div class="_2_QraFYR_0">既然最后引用了芒格语录，那么，在规避错误上其实也可以借鉴芒格的思想。作为一个保守派的思想家、投资人。芒格总结了23种心理偏差，上百种跨学科的思维方式，目的都是为了更为理性。像飞行员般列清单;如果我知道我在哪里死去，我就再也不去那个地方;祖母的规则;反过来想，都是避免错误的一些方式。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-26 09:01:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/bc/d5/036bc464.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>封志强</span>
  </div>
  <div class="_2_QraFYR_0">老师，优秀哦</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-24 17:07:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/7a/d2/4ba67c0c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Sch0ng</span>
  </div>
  <div class="_2_QraFYR_0">程序反复出现的问题一般不是程序的问题。<br>如果是程序的问题，还反复出现，那就是技术能力的问题。<br>既然反复出现了，说明问题不在技术能力，更可能在人身上。<br>比如强压预算，导致实现变形；比如强加需求，导致系统走形。<br>所以，有时候走不出bug，不一定是程序的原因，也有可能是所处的环境的原因。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-25 20:50:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/2b/39/19041d78.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>😳</span>
  </div>
  <div class="_2_QraFYR_0">我出现的bug一般都是粗心大意，但是每次出现后，都会记住这个问题，以便后续少发生类似的事情，偶尔也会出现一些认知偏差的问题。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-10 15:44:29</div>
  </div>
</div>
</div>
</li>
</ul>