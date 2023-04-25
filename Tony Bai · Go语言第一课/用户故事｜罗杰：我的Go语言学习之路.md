<audio title="用户故事｜罗杰：我的Go语言学习之路" src="https://static001.geekbang.org/resource/audio/51/9c/51e418fecdf14375ee053674byybf89c.mp3" controls="controls"></audio> 
<p>你好，我是罗杰，目前在一家游戏公司担任后端开发主程。今天，我想跟你分享一下我学习Go的一些经历，如果你还是一个Go新人，希望我的这些经历，能给你带来一些启发和帮助。</p><p>说起来，我接触Go语言已经很久了，但前面好多次都没真正学起来。</p><p>我第一次接触 Go 语言是在 2010 年，当时我还在读大二，一个学长建议我了解一下 Go 语言，毕竟是谷歌出的一门语言，可能未来比较有发展前景。所以我当时下载并安装了Go的开发环境，还写了个 “hello world”，但是由于没有中文的教程，也没有人一起学习，学习 Go 语言这件事情很快就被我抛在脑后了。</p><p>我第二次接触Go是在 2015 年。当时我跟在豆瓣工作的发小聊天，我说最近想学 Python，他却坚定地告诉我学 Go。因为他们团队无法忍受 Python 总是半夜异常导致全站挂掉，正在往 Go 迁移。但我当时并没有给予足够的重视，没有学半个月又去玩了。</p><p>第三次接触Go语言是在 2017 年。我当时的技术栈只有 C++，对于 Web 服务这块，几乎没有任何经验，而且我们组的其他成员也没有相关经验。我就在想，总不能 Web 服务这一块总是向其他项目组“借”个 PHP 的同学过来协助吧？</p><p>于是，我开始再次尝试学习使用 Go 语言，并且因为一篇《<a href="https://tonybai.com/2015/11/17/tcp-programming-in-golang/">Go语言TCP Socket编程</a>》的博客认识了 Tony Bai 老师。当时我考虑把原本用 C++ 写的游戏服务改成用 Go 来实现，但当时能用中文搜索到的与 TCP 网络编程的文档非常有限，而Tony Bai 老师的文章写得非常详细，使我对 Go 是如何做这一部分有了基本的了解。</p><!-- [[[read_end]]] --><p>这一年，我开始尝试使用 Go 语言写一些小的 demo，例如操作数据库，以及Redis 和 Protobuf 相关等在 C++ 中要必须使用到的组件。</p><p>2018 年，在 Go 的实践上，我迈出了更大一步。我开始在新的项目上尝试使用 Go 语言完成了一些简单的功能，例如游戏版本控制，后台管理服务。但因为我从来没有深入学习过 Go 语言，在完成这些功能时，踩过好多坑。 比如，我当时开发管理后台，在谷歌上搜索要不要使用框架，就被某个知乎的言论坑得很惨。大意是 Go 已经非常简单了，没有必要使用框架。结果，我用 http 库辛苦搞了一阵后，才发现几乎没有人这么干，都是基于 gin 开发。这个时候，我意识到我必须要认真学习下Go语言了。</p><p>好在，当时的相关学习资料已经比较丰富了，我在慕课网学习了一遍 ccmouse 老师的<a href="https://coding.imooc.com/class/180.html#Anchor">视频课</a>，算是正式入门了 Go 语言。期间，我因为 Web 后台需要，在多看阅读上购买了《<a href="https://www.duokan.com/pc/detail/18375d92b9b74ee48a840ba3665024df">Go Web编程</a>》，不过到现在我都没有认真把这本书读完，基本上是把它当工具书，遇到问题，查查看里面有什么解决方案。</p><p>2019 年，我的项目上线了，前面我用 Go 写的两个服务的重要性就体现出来了，版本控制负责了游戏登录前的工作以及所有的平台方充值，管理后台则是运营人员最主要的使用工具。要知道 Web 这一块，我之前想都不敢想，但是现在我竟然做出来了，上线之后，稳定性也超出预期。接下来基本上一整年的时间，我都在不停地重构与维护这两个服务，期间还由于涉及前端页面的东西，在 B 站学了不少<a href="https://www.bilibili.com/video/BV1BE411y7yr?spm_id_from=333.999.0.0"> HTML/CSS</a>，<a href="https://www.bilibili.com/video/BV1TE411B7KU?from=search&amp;seid=4991647789799854809&amp;spm_id_from=333.337.0.0">JavaScript</a> 的课程来配合业务方完成相应的功能。</p><p>但通过一年时间的修修补补，我意识到，我基本上还不能算入门 Go 语言，因为稍微高级一点的功能我都不会，也几乎没有深入到源码中去研究这些功能是如何实现的。</p><p>2020 年以疫情开始，在被 Go 语言折磨了近一年之后，我终于下定决心要深入 Go 语言了。当时我因为《<a href="https://qcrao.com/2019/04/02/dive-into-go-slice/">深度解密Go语言之slice</a>》这一篇文章认识了饶大，也关注了一些“Go 夜读”的成员，以及 <a href="https://draveness.me/golang/">draveness</a>，这些大佬们对我触动非常大。</p><p>看了这么多优秀博主的博客之后，我之前的恐惧都没有了，因为他们把底层的源码都翻了出来，努力解除我们的困惑，也让我更有信心在工作中使用 Go 语言。</p><p>不过，虽然有优秀的博客，但是我们学习一定不能只依靠零散的博客，而要成体系地学习。后来，我在慕课网上学习了 Tony Bai 老师的 《<a href="https://www.imooc.com/read/87">改善Go语言编程质量的50个有效实践</a>》 。年底的时候，无意中又阅读了COLLSHELL 《<a href="https://coolshell.cn/articles/21128.html">Go 编程模式</a>》的系列博客，感觉对 Go 的理解又上了一个层次，作为回报，我在极客时间订阅了作者的专栏《<a href="https://time.geekbang.org/column/intro/100002201?tab=catalog">左耳听风</a>》。</p><p>通过近一年的 Go 的学习与积累，我应该可以算是熟练使用 Go 语言了，我开始尝试阅读一些源码，并且使用了 go-micro 作为框架层完成新游戏的开发。而且，使用 Go 开发的效率显然比 C++ 要高出不少，因为我们后端就只有两个人。</p><p>2021 年，学习的脚步依旧不能停止。这一年里，我最大的收获就是<strong>如何正确地学习</strong>。如今信息真是满天飞，各种各样的 APP 都想把各种内容塞给我们。而我们的大脑逐渐进入到了“看一遍就当记住了，收藏了就是掌握了”的状态当中。</p><p>我经常看了一个小时的手机，回头一想，刚才看了个什么，好像啥也没记住。这就是因为学习的方法错了，我发现学东西真的不能看一遍就了事。《论语》第一句就是：“<strong>学而时习之，不亦说乎</strong>”，为政篇更是提到“<strong>温故而知新，可以为师矣</strong>”。</p><p>除了勤奋之外，我们还应该明白学习要抓住核心、抓住本质。什么叫“书读千遍，其义自见”？就是说学习一定不是学得越多越好，而应该抓住本质。我最近很喜欢一部电影，叫《银河补习班》，电影里的爸爸把除课本以外的书籍都扔掉，发现所有课本才十一厘米。</p><p><img src="https://static001.geekbang.org/resource/image/ca/5e/ca3dc0c0c8e9d80838d177eca11aeb5e.jpg?wh=658x493" alt="图片"></p><p>关于更具体的学习方法，我推荐陈天老师的《<a href="https://www.bilibili.com/video/BV1n54y1z7KM?from=search&amp;seid=773335947933248559&amp;spm_id_from=333.337.0.0">如何学习一门技术？</a>》，你可以学习一下，里面分享了非常多的干货，而且生动有趣。毕竟，在了解了大神是如何学习之后，我们才有可能成为一个大神。陈天老师在极客时间也有专栏《<a href="https://time.geekbang.org/column/intro/100085301?tab=catalog">陈天 · Rust 编程第一课</a>》，如果你有兴趣也可以了解一下。</p><p>从今年八月份起，我每周在极客时间App 上学习的时间都超过了十个小时，算是非常活跃的用户了。</p><p>当我在推广页看到Tony Bai 老师的课的那一瞬间，我就购买了。因为从老师的博客和之前的专栏上，我确实学到很多在其它地方学习不到的内容。我相信你在学这个专栏的时候应该也有体会，老师会不停地强调 Go 语言的设计哲学，更会直达Go语言的本质。</p><p>虽然我用 Go 语言做开发的时间已经超过三年了，但依然从这个专栏上学到许多非常实用的技巧，弥补了之前遗漏的很多知识点。我在其中一讲留言说：<strong>老师在我心目中就是 “Go 语言百科全书”</strong>，这句话真的是我对老师发自内心的敬佩 。</p><p>洋洋洒洒写了这么多，我觉得我在学习、提高的时候走了很多弯路，从一开始的学习方法就是错误的：</p><ul>
<li><strong>首先是没有主动性。</strong>如果 2010 年我就能坚持去阅读英文的文档，深入去学习 Go 语言，可能现在我也能跟这些大佬一样，写出优秀的博客来帮助其他人；</li>
<li><strong>其次是懒得学习。</strong>2013 年，我是在 COLLSHELL 中学习Vim，但是我没有再关注过博主的其它文章，其实当时博主很多关注 C++ 的文章，写得也非常优秀。如果我能更主动一些，就能发现更多的“宝藏”；</li>
<li><strong>最后是方法错误。</strong>除了 Go 之外，我还学过 Python 三次，而且每次都完整地看完了一本书，或者学习了完整的视频，但是至今我也无法很快写出 Python 代码，因为我从来都没有实践过。</li>
</ul><p>所以，我想跟你说的是，如果你想让自己的学习更有收获、少走弯路，我建议你多注意一下学习方法。你可以了解一下两个重要理论。首先是<a href="https://en.wikipedia.org/wiki/Learning_pyramid">学习金字塔和</a><a href="https://www.bilibili.com/video/BV1UE411y7mw?from=search&amp;seid=4884264404421278473&amp;spm_id_from=333.337.0.0">费曼学习法</a>，从这里我们可以知道，通过听讲与阅读知识的留存率最多只有 10%。第二个是<a href="https://zh.wikipedia.org/wiki/%E9%81%97%E5%BF%98%E6%9B%B2%E7%BA%BF">艾宾浩斯遗忘曲线</a>，从这里我们还可以知道如果不复习，第二天我们学到的知识只剩下了 5%。久而久之，我们就会对学习失去了兴趣。</p><p><img src="https://static001.geekbang.org/resource/image/55/9f/555114ab8c1f818b7a016a6c3375269f.jpg?wh=550x356" alt="" title="学习金字塔"></p><p><img src="https://static001.geekbang.org/resource/image/5e/38/5ed3eb28e9230ca78656de2f2c3beb38.jpg?wh=1920x1643" alt="" title="艾宾浩斯遗忘曲线"></p><p>那我们可以怎么结合这两个理论，提升自己的学习效率/能力呢？</p><p>我跟你分享下我现在学习专栏课的方法吧：</p><ul>
<li>第一天仔细阅读一遍（如果有不理解的，我可能会多读几遍）；</li>
<li>第二天复习一遍，如果本周内时间充足，一定要做笔记；</li>
<li>如果有实战的内容，抽出时间写代码（如果没有时间，加到待办任务中，等有时间一定要做一遍）；</li>
<li>如果时间允许，最好有一个月内再复习一次，这样才能有效地抵御遗忘曲线。</li>
</ul><p>除此之外，我还会将学到的知识按照我的理解给同事讲解，一定要用心讲解，因为只有教会别人，才可能是自己真正掌握的时候。同时，我也会在工作中践行学习到的内容，比如最近我也在学习<a href="https://time.geekbang.org/column/intro/100039001?tab=catalog">《设计模式之美》</a>专栏，发现只有把学到的思想应用在平时代码中，你的学习才会有明显的效果。要是你没有能讲解的对象，也无法立刻在工作中使用，我想写博客应该也是个不错的选择，总之有输入必须要有输出。</p><p>最后我想说的是，学习真的要认真对待，我建议你养成做笔记的好习惯。不管是看专栏、读书，还是阅读微信公众号，看 B站的视频，我们都可以将学到的东西记录下来，<strong>时常回顾。<strong>让学到的知识不轻易流失，让要学习的内容越来越少，我们才会觉得越来越轻松</strong>。</strong></p><p>希望所有人都能学会正确的学习方法，坚持终身学习的理念，让自己变得越来越好。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/ea/7a/d857723d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Vfeelit</span>
  </div>
  <div class="_2_QraFYR_0">这个同学就讲得很好 我也学习过很多门语言 go是我最喜爱的 没有之一</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-14 22:47:34</div>
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
  <div class="_2_QraFYR_0">原来是 C++ 大佬！<br>感谢分享 gin 不然我也会天真的认为要什么框架，直接写 web 不行吗？<br>谢谢分享了很多优秀资源<br>这个专栏是我第三次学 Go 了，感觉要学会了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 第三次学go，韧性十足。👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-01 19:44:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/22/b1/54/6d663b95.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>瓜牛</span>
  </div>
  <div class="_2_QraFYR_0">说下我为什么购买此课程吧，之前搜grpc问题搜到过作者的博客，阅读下来感觉很好，能感受到作者功力深厚，写文章抽丝剥茧，娓娓道来，于是就购买课程支持一下。之前也买过极客时间另一个go课程，作者的语言表达能力真是不敢恭维，总是自创一些词汇，看得十分吃力，能把人当场说晕过去。（当时也吐槽了一下，没想到主编还给我退钱了😢）最后说下学习方法，学习这么多年了，每个人应该都有自己的学习方法，适合自己的才是最好的。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-10 19:58:54</div>
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
  <div class="_2_QraFYR_0">老师可以催更吗，无聊开始二刷了哈哈</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我会努力更的。:)</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-27 10:22:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/34/01/30ca98e6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>arronK</span>
  </div>
  <div class="_2_QraFYR_0">分享学习方法<br>1. 拆分大的知识内容为小的知识模块<br>2. 专注刻意练习每个模块<br>3. 构建知识体系（建立知识之间的联系）<br>4. 用起来或教别人<br>5. 间隔性的检索（不是看着书复习，而是什么都不看，能不能像给人上课一样把整个知识体系全盘托出，如果不能请回到 2 重复）</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-26 23:01:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/8a/89/8940ea1e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小戈</span>
  </div>
  <div class="_2_QraFYR_0">游戏服务器开发怎么学</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 直接加罗杰为好友，问他吧😄</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-06 08:32:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/d7/bf/9d8984b7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>码狐</span>
  </div>
  <div class="_2_QraFYR_0">杰哥🐮，本文提到的方法这段时间学习专栏感受很深，确实要复习，阶段总结，不然就成了走马观花，很容易忘个光。<br>所以这段时间课程中的代码我都尽可能都敲一遍，发现很多看似简单的代码实际动手了还是没有思路。不过一定要切记不能抄代码，可以先列出注释，然后照着注释动手，有些过去的知识点也可以在写代码过程中有意识的融合进去，加深自己的理解。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-29 08:40:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/b8/d8/f81b5604.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>hcyycb</span>
  </div>
  <div class="_2_QraFYR_0">罗杰同学优秀啊，学习</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-27 11:18:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/dd/7d/5d3ab033.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>不求闻达</span>
  </div>
  <div class="_2_QraFYR_0">谢谢分享学习心得，谢谢分享很多优秀资源。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-07 07:35:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/33/9a/f295dea5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>李正g</span>
  </div>
  <div class="_2_QraFYR_0">第二次学，希望能有所成就</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 💪</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-15 17:50:23</div>
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
  <div class="_2_QraFYR_0">和优秀的人在一起！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-20 16:29:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/ef/4d/83a56dad.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Z.</span>
  </div>
  <div class="_2_QraFYR_0">初次接触go，感觉真的很难，希望能坚持下去<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 加油💪。Go已经算是编程语言中较为简单的一种了:)。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-21 16:30:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/30/67/a1e9aaba.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Roway</span>
  </div>
  <div class="_2_QraFYR_0">怎样加罗杰？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-08 10:51:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c9/10/65fe5b06.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>J²</span>
  </div>
  <div class="_2_QraFYR_0">我叫李杰，希望有一天也能像罗杰一样优秀。😄</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 💪</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-09 11:17:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/7b/bd/ccb37425.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>进化菌</span>
  </div>
  <div class="_2_QraFYR_0">不刻意去练习，真的太容易忘记以及放弃了。系统去学习go，用go~</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-28 00:13:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2b/46/af/c20a8244.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>南极熊</span>
  </div>
  <div class="_2_QraFYR_0">罗杰同学优秀，向罗杰学习</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-27 20:35:27</div>
  </div>
</div>
</div>
</li>
</ul>