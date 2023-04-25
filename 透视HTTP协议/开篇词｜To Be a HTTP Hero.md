<audio title="开篇词｜To Be a HTTP Hero" src="https://static001.geekbang.org/resource/audio/24/7f/243a3f2cac4406a379a8030f9e2e897f.mp3" controls="controls"></audio> 
<p>你好，我是罗剑锋（Chrono），一名埋头于前线，辛勤“耕耘”了十余载的资深“码农”。</p><p>工作的这十多年来，我开发过智能IC卡，也倒腾过商用密码机；做过政务项目，也做过商务搜索；写过网游核心引擎，也写过CDN存储系统；在Windows上用C/C++做客户端，在AIX、Linux上用Java、PHP写后台服务……现在则是专注于“魔改”Nginx，深度定制实现网络协议的分析与检测。</p><p>当极客时间的编辑联系我，要我写HTTP专栏的时候，我的第一反应是：“HTTP协议好简单的，有这个必要吗？”</p><p>你可能也会有同样的想法：“HTTP不就是请求/响应、GET/POST、Header/Body吗？网络上的资料一抓一大把，有什么问题搜一下就是了。”</p><p>不瞒你说，我当时就是这么想的，在之前的工作中也是一直这么做的，而且一直“感觉良好”，觉得HTTP就是这个样子，没有什么特别的地方，没有什么值得讲的。</p><p>但在编辑的一再坚持下，我“勉为其难”接下了这个任务。然后做了一个小范围的“调查”，问一些周围的同事，各个领域的都有，比如产品、开发、运维、测试、前端、后端、手机端……想看看他们有什么意见。</p><p>出乎我的意料，他们无一例外都对这个“HTTP专栏”有很强烈的需求，想好好“补补课”，系统地学习了解HTTP，这其中甚至还包括有七、八年（甚至更多）工作经验的老手。</p><!-- [[[read_end]]] --><p>这不禁让我陷入了思考，<span class="orange">为什么如此“简单”的协议却还有这么多的人想要学呢？</span></p><p>我想，一个原因可能是HTTP协议“<strong>太常见</strong>”了。就像现实中的水和空气一样，如此重要却又如此普遍，普遍到我们几乎忽视了它的存在。真的很像那句俗语所说：“鱼总是最后看见水的”，但水对鱼的生存却又是至关重要。</p><p>我认真回忆了一下这些年的工作经历，这才发现HTTP只是表面上显得简单，而底层的运行机制、工作原理绝不简单，可以说是非常地复杂。只是我们平常总是“KPI优先”，网上抓到一个解决方法用过就完事了，没有去深究里面的要点和细节。</p><p>下面的几个场景，都是我周围同事的实际感受，你是否也在工作中遇到过这样的困惑呢？你能把它们都解释清楚吗？</p><ul>
<li>用Nginx搭建Web服务器，照着网上的文章配好了，但里面那么多的指令，什么keepalive、rewrite、proxy_pass都是怎么回事？为什么要这么配置？</li>
<li>用Python写爬虫，URI、URL“傻傻分不清”，有时里面还会加一些奇怪的字符，怎么处理才好？</li>
<li>都说HTTP缓存很有用，可以大幅度提升系统性能，可它是怎么做到的？又应该用在何时何地？</li>
<li>HTTP和HTTPS是什么关系？还经常听说有SSL/TLS/SNI/OCSP/ALPN……这么多稀奇古怪的缩写，头都大了，实在是搞不懂。</li>
</ul><p>其实这些问题也并不是什么新问题，把关键字粘贴进搜索栏，再点一下按钮，搜索引擎马上就能找出几十万个相关的页面。但看完第一页的前几个链接后，通常还是有种“懵懵懂懂”“似懂非懂”的感觉，觉得说的对，又不全对，和自己的思路总是不够“Match”。</p><p>不过大多数情况下你可能都没有时间细想，优先目标是把手头的工作“对付过去”。长此以来，你对HTTP的认识也可能仅限于这样的“知其然，而不知其所以然”，实际情况就是HTTP天天用，时时用，但想认真、系统地学习一下，梳理出自己的知识体系，经常会发现无从下手。</p><p>我把这种HTTP学习的现状归纳为三点：<strong>正式资料“少”、网上资料“杂”、权威资料“难”</strong>。</p><p>第一个，<strong>正式资料“少”</strong>。</p><p>上购书网站，搜个Python、Java，搜个MySQL、Node.js，能出一大堆。但搜HTTP，实在是少得可怜，那么几本，一只手的手指头就可以数得过来，和语言类、数据库类、框架类图书真是形成了鲜明的对比。</p><p>现有的HTTP相关图书我都看过，怎么说呢，它们都有一个特点，“广撒网，捕小鱼”，都是知识点，可未免太“照本宣科”了，理论有余实践不足，看完了还是不知道怎么去用。</p><p>而且这些书的“岁数”都很大，依据的都是20年前的RFC2616，很多内容都不合时宜，而新标准7230已经更新了很多关键的细节。</p><p>第二个，<strong>网上资料“杂”</strong>。</p><p>正式的图书少，而且过时，那就求助于网络社区吧。现在的博客、论坛、搜索引擎非常发达，网上有很多HTTP协议相关的文章，也都是网友的实践经验分享，“干货”很多，很能解决实际问题。</p><p>但网上文章的特点是细小、零碎，通常只“钉”在一个很小的知识点上，而且由于帖子长度的限制，无法深入展开论述，很多都是“浅尝辄止”，通常都止步在“How”层次，很少能说到“Why”，能说透的更是寥寥无几。</p><p>网文还有一个难以避免的“毛病”，就是“良莠不齐”。同一个主题可能会有好几种不同的说法，有的还会互相矛盾、以讹传讹。这种情况是最麻烦的，你必须花大力气去鉴别真假，不小心就会被“带到沟里”。</p><p>可想而知，这种“东一榔头西一棒子”的学习方式，用“碎片”拼凑出来的HTTP知识体系是非常不完善的，会有各种漏洞，遇到问题时基本派不上用场，还得再去找其他的“碎片”。</p><p>第三个，<strong>权威资料“难”</strong>。</p><p>图书少，网文杂，我们还有一个终极的学习资料，那就是RFC文档。</p><p>RFC是互联网工程组（IETF）发布的官方文件，是对HTTP最权威的定义和解释。但它也是最难懂的，全英文看着费劲，理解起来更是难上加难，文档之间还会互相关联引用，“劝退率”极高。</p><p>这三个问题就像是“三座大山”，阻碍了像你这样的很多有心人去学习、了解HTTP协议。</p><p>那么，怎么才能更好地学习HTTP呢？</p><p>我为这个专栏定了一个基调：<span class="orange">“要有广度，但更要有深度”</span>。目标是成为含金量最高的HTTP学习资料，新手可以由浅入深、系统学习，老手可以温故知新、查缺补漏，让你花最少的时间，用最少的精力，掌握最多、最全面、最系统的知识。</p><p>由于HTTP应用得非常广泛，几乎涉及到所有的领域，所以我会在广度上从HTTP尽量向外扩展，不只讲协议本身，与它相关的TCP/IP、DNS、SSL/TLS、Web Server等都会讲到，而且会把它们打通串联在一起，形成知识链，让你知道它们之间是怎么联系、怎么运行的。</p><p>专栏文章的深度上我也是下足了功夫，全部基于最新的RFC标准文档，再结合我自己多年的实践体会，力求讲清讲透，能让你看了以后有豁然开朗的感觉。</p><p>比如分析HTTPS，我会用Wireshark从建立TCP连接时就开始抓包，从二进制最底层来分析里面的Record、Cipher Suite、Extension，讲ECDHE、AES、SHA384，再画出详细的流程图，做到“一览无余”。</p><p>陆游有诗：“<strong>纸上得来终觉浅，绝知此事要躬行</strong>”。学习网络协议最重要的就是实践，在专栏里我还会教你用Nginx搭建一个“麻雀虽小，五脏俱全”的实验环境，让你与HTTP零距离接触。</p><p>它有一个最大的优点：自身就是一个完整的网络环境，即使不联网也能够在里面收发HTTP消息。</p><p>我还精心设计了配套的测试用例，最小化应用场景，排除干扰因素，你可以在里面任意测试HTTP的各种特性，再配合Wireshark抓包，就能够理论结合实践，更好地掌握HTTP的知识。</p><p>每一讲的末尾，我也会留几个思考题，你可以把它当作是求职时的面试官问题，尽量认真思考后再回答，这样能够<span class="orange">把专栏的学习由“被动地听”，转变为“主动地学”，实现“学以致用”</span>。</p><p>当然了，你和我的“兴趣点”不可能完全一样，我在讲课时也难免“顾此失彼”“挂一漏万”，希望你积极留言，我会视情况做些调整，或者用答疑的形式补充没讲到的内容。</p><p>今年是万维网和HTTP诞生30周年，也是HTTP/1.1诞生20周年，套用莎翁《哈姆雷特》里的名句，让我们在接下来的三个月里一起努力。</p><p>“To Be a HTTP Hero！”</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/78/ac/e5e6e7f3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>古夜</span>
  </div>
  <div class="_2_QraFYR_0">冲你这个发型我订了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 服就一个字。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-29 19:08:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/fd/1c/8245a8d8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>imxintian</span>
  </div>
  <div class="_2_QraFYR_0">我也是看这个发型，直接就来了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 羡慕你的秀发，笑。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-30 15:56:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/IPdZZXuHVMibwfZWmm7NiawzeEFGsaRoWjhuN99iaoj5amcRkiaOePo6rH1KJ3jictmNlic4OibkF4I20vOGfwDqcBxfA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>鱼向北游</span>
  </div>
  <div class="_2_QraFYR_0">看发型我就知道老师是高手 买买买</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-04 09:30:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/ec/cb/6ee732c9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span> Sapph</span>
  </div>
  <div class="_2_QraFYR_0">HTTP权威指南买了好长时间，我就没看完过，太难搞了，希望能跟着Chrono老师学好HTTP💪 </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 一起努力，这本书的内容比较陈旧，专栏里会紧跟最新的标准文档。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-29 17:32:41</div>
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
  <div class="_2_QraFYR_0">罗老师胖了，头发更少了~~~</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 目前正在跑步减肥，头发是没办法了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-01 22:39:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>极客酱</span>
  </div>
  <div class="_2_QraFYR_0">老师是从硬搞到软的啊，软硬结合的人最有技术含量，就冲这一点，我买了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 做的比较杂，见笑了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-30 08:37:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/71/ca/87688ff9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>iGeneral</span>
  </div>
  <div class="_2_QraFYR_0">发型来分析是整个极客第二厉害的最强王者，订阅。（http上课打瞌睡，然后自己就看了一本http图解，膜拜大佬，抓住痛点）</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不用着急，很快就成琦玉了，笑。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-30 10:56:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/94/db/4e658ce8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>继业(Adrian)</span>
  </div>
  <div class="_2_QraFYR_0">光看介绍，就让人不禁的搓搓手</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-29 18:55:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/75/89/74ee014c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_Corey</span>
  </div>
  <div class="_2_QraFYR_0">想好好学习一下 曾经被面试官打击的要死 现在准备学好再打击回去</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 进击的程序员！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-30 08:50:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/52/eb/eec719f3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>开水</span>
  </div>
  <div class="_2_QraFYR_0">冲着这发型，必须得喊一句，老师！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-04 10:49:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/57/4f/6fb51ff1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>一步</span>
  </div>
  <div class="_2_QraFYR_0">RFC文档的根本看不下去啊，都是文字，还那么长，老师可以讲讲有关RFC文档的东西，怎么阅读，怎么去查找某个标准对应的哪一个RFC文档，还有RFC的文档名怎么定义的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 破冰篇最后一讲会有GitHub项目，列出http相关的rfc，方便你查阅。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-29 21:03:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/8b/7d/bfa7b4ad.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>黄</span>
  </div>
  <div class="_2_QraFYR_0">是时候还罗老师送我的书钱了！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 好熟悉的头像，哈哈。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-29 23:26:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/e8/c9/59bcd490.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>听水的湖</span>
  </div>
  <div class="_2_QraFYR_0">之前看趣谈网络协议的时候，里面关于HTTP就两三篇，而这个专栏看来是把HTTP讲细讲透了，上线就更了一篇正文，读起来挺有意思的，不枯燥，种草了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 期待与你共同学习进步。<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-29 19:36:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/55/86/ca7c94ce.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>羁绊12221</span>
  </div>
  <div class="_2_QraFYR_0">深知读者痛点，赞一个！很期待后续的课程</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: thanks。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-31 14:19:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/ajNVdqHZLLAxTB7VjiboJLKujMGtj9EeTSX8yPStoqsjzqjeuBQkWd1IMQvicOMQhZEPZemBFBeoQupGz4UsSic7g/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>名</span>
  </div>
  <div class="_2_QraFYR_0">一、专栏开篇<br>需求：HTTP协议太常见，各职业角色都有系统想真正搞懂的需要<br>现状：权威、正式资料少且晦涩难懂且个别信息过时；网上资料太碎没有体系的东西<br>学习：理论与实践结合实操起来并形象的类比理解学习</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 欢迎新同学。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-12 15:57:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/75/93/8135f895.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>bywuu</span>
  </div>
  <div class="_2_QraFYR_0">我就是在极客时间上评论“看发型我就知道，这个是高手！”的人。。。不订对不起自己的判断</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢支持。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-01 17:44:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKObibUvAjbt2hG3Sb9uFAnLurM6aQDvppQOia7f7QCPk50W8KCc24PaXEm9YVxEOND1PDpp24NUloA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Kauh</span>
  </div>
  <div class="_2_QraFYR_0">掉头发是对技术最大的尊重</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个我不能同意，最好是不掉头发也能尊重技术，笑。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-22 08:13:46</div>
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
  <div class="_2_QraFYR_0">老师我正在学rtsp这个视频流应用层协议，和http类似，用ffmpeg接受，有什么方法可以拦截两个软件之间的流量呢，可以把接收方软件的地址搞成类似实验环境里的127.0.0.1吗？<br>有时vs开发一些软件，也会发http给远程设备，这种情况怎么简化为127.0.0.1这种实验环境呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.用wireshark可以捕获任何网络中的数据包。<br><br>2.可以在本机上搭一个类似的环境，然后客户端配置成127.0.0.1，但需要结合你的实际情况。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-06 10:28:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/28/71/0a6f52c4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>不会凉的黄花菜</span>
  </div>
  <div class="_2_QraFYR_0">已经学完了该专栏，很喜欢老师的讲课风格，有趣易懂不枯燥。老师把知识用最通俗最容易理解的方式讲了出来，引人入胜，受益匪浅。期待老师，出下一个新专栏。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 承蒙夸奖，不胜感激。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-06 14:40:31</div>
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
  <div class="_2_QraFYR_0">楼主和1楼都优秀，老司机开车，稳！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-30 10:59:49</div>
  </div>
</div>
</div>
</li>
</ul>