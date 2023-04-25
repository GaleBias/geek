<audio title="结束语 _ 从学习Redis到向Redis学习" src="https://static001.geekbang.org/resource/audio/d0/15/d01e3e64a02118809121f46916b31715.mp3" controls="controls"></audio> 
<p>你好，我是蒋德钧。</p><p>这么快就到课程的尾声了，到了和你说再见的时候了。</p><p>在过去的4个多月时间里，我们掌握了Redis的各种关键技术和核心知识。在课程的最后，我想带你切换一个视角：<strong>如果说我们之前一直在学习Redis本身，那么今天，我们来看看能向Redis学到什么。</strong></p><p>在聊这个“视角”之前，我想先问你一个问题：你有没有想过，学习技术究竟意味着什么呢？</p><p>大多数人人都会觉得，就是掌握具体的原理，进行实战，并且学习别人的经验，解决自己在实际工作中的问题。比如说，学习Redis时，我们会把它用在缓存、分布式锁、数据集群等业务场景中，这就需要我们掌握关键实践技巧、常见问题和应对方法，这也是我们课程的聚焦点。</p><p>但是，我认为，这只是学习技术的第一个层面。当我们对技术的认识和积累达到一定程度后，我们就应该“<strong>向技术致敬</strong>”。所谓的致敬，就是向技术学习，来解决我们在生活中遇到的问题。这是第二个层面。</p><p>这背后的道理其实非常朴素：每一项优秀技术都是一些精华思想的沉淀成果，<strong>向技术学习，其实就是向优秀的思想学习</strong>。</p><p><strong>我一直很崇尚一个理念：一个优秀的计算机系统设计本身就包含了不少人生哲学</strong>。所以，接下来，我们就再往前迈一步，从Redis设计中总结一些做事方法。</p><!-- [[[read_end]]] --><h2>向Redis单线程模式学习，专心致志做重要的事</h2><p>Redis的最大特点是快，这是Redis在设计之初就设立的目标。而能成为某项技术的高手、某个技术方向的大牛，通常是我们给自己设立的目标。Redis实现“快”这个目标的关键机制就是单线程架构。单线程架构就给我们提供了一个很好的做事方式：<strong>专心致志做一件事，把事情做到极致，是达到目标的核心要素。</strong></p><p>在Redis的设计中，主线程专门负责处理请求，而且会以最快的速度完成。对于其他会阻碍这个目标的事情（例如生成快照、删除、AOF重写等），就想办法用异步的方式，或者是用后台线程来完成。在给你介绍6.0版本时，我还提到，Redis特意把请求网络包读写和解析也从主线程中剥离出来了，这样主线程就可以更加“专注”地做请求处理了。</p><p>我认为，“单线程”思想是非常值得我们品味的。在确定目标以后，我们也可以采用“单线程模式”，把精力集中在核心目标上，竭尽全力做好这件事，同时合理安排自己的时间，主动避开干扰因素。</p><p>当我们沉浸在一件事上，并且做到极致时，距离成为大牛，也就不远了。</p><p>当然，我们说在一件事上做到极致，并不是说只盯着某一个知识点或某一项技术，而是指在一个技术方向上做到极致。</p><p>比如说，Redis属于键值数据库，我们就可以给自己定个目标：精通主要的键值数据库。因此，我们不仅要扎实地掌握现有技术，还要持续关注最新的技术发展。这就要提到我们可以向Redis学习的第二点了：具备可扩展能力。</p><h2>向Redis集群学习可扩展能力</h2><p>在应用Redis时，我们会遇到数据量增长、负载压力增大的情况，但Redis都能轻松应对，这就是得益于它的可扩展集群机制：当数据容量增加时，Redis会增加实例实现扩容；当读压力增加时，Redis会增加从库，来分担压力。</p><p>Redis的新特性在持续推出，新的存储硬件也在快速地发展，这些最新技术的发展，很可能就会改变Redis的关键机制和使用方法。<strong>所以，想要应对复杂的场景变化，我们也要像Redis集群一样，具备可扩展能力。</strong>毕竟，技术的迭代速度如此之快，各种需求也越来越复杂。如果只是专注于学习现有的技术知识，或者是基于目前的场景去苦心钻研，很可能会被时代快速地抛弃。</p><p>只有紧跟技术发展的步伐，具备解决各种突发问题的能力，才能成为真正的技术大牛。</p><p>怎么培养可扩展能力呢？很简单，随时随地记录新鲜的东西。这里的“新鲜”未必是指最新的内容，而是指你不了解的内容。当你的认知范围越来越大，你的可扩展能力自然就会越来越强。</p><p>说到这儿，我想跟你分享一个我的小习惯。我有一个小笔记本，会随身携带着，在看文章、参加技术会议，或是和别人聊天时，只要学到了新东西，我就会赶紧记下来，之后再专门找时间去搜索相关的资料，时不时地拿出来回顾一下。这个习惯，让我能够及时地掌握最新的技术，轻松地应对各种变化。</p><p>我们做技术的同学，通常习惯于脚踏实地地把事情做好，但是，也千万别忘了，脚踏实地的同时，也是需要“仰望星空”的。要把学习变成一种习惯，从为了应对问题的被动学习，到为了增强自己的可扩展性而主动学习，这个转变绝对可以让你的技术能力远超过其他人。</p><p>当然，Redis的优秀设计思想还有很多，你还可以自己提炼总结下。我还想再跟你探讨的话题是，我们该怎么把向Redis学到的思想真正落地到实践中呢？</p><p>其实，道理也很简单：<strong>从做成一件事开始</strong>。在竭尽全力做成事情的过程当中，磨炼自己的专注力，锻炼自己的可扩展能力。</p><h2>从做成一件事开始</h2><p>我们常说“不积跬步，无以至千里”，这句话中的“跬步”，我把它解释为做成一件事。我们总是会做很多事，但是，很多时候，能够让我们真正得到提升的是把事做成。</p><p>对我来说，创作这门课完全是一次全新的尝试。在写作时，无论是思考内容的结构，确认具体的细节，还是连夜赶稿以保证按时更新，我都感受到了不少压力。但是，现在我回过头来看过去的半年，感到很欣慰，因为这事儿我做成了，而且有很多额外的收获。</p><p>其实，做成一件事的目标不分大小。它可以很小，比如学完两节课，也可以很大，比如花3个月时间把Redis源码读完。</p><p>最重要的是，一旦定好目标，我们就要尽全力把这件事做成。我们不可避免地会遇到各种困难，比如临时有其他的工作安排，抽不出时间，或者是遇到了不理解的内容，很难再学进去。但是，这就像爬山，爬到半山腰的时候，往往也是我们最累的时候。</p><p>我再跟你分享一下我自己的小故事。</p><p>在看Redis数据结构的源码时，我觉得非常困难。Redis的数据类型非常多，每种数据类型还有不同的底层结构实现，而有的数据结构本身就设计得很复杂。</p><p>当时我差一点就决定放弃了，但是，我后来憋着一口气，说我一定要把事情做成。冷静下来之后，我进一步细分目标，每周搞定一个结构，先从原理上理解结构的设计，自己在白纸上推演一遍。然后，把每个结构的代码看一遍，同时自己也把关键部分编写一遍。毕竟，我们在看代码的时候，很容易想当然地跳过一些地方，只有自己一行行地去编写时，才会思考得更细致，理解得也更透彻。</p><p>攻克了“数据结构”这个难关之后，我发现，后面的就简单多了。甚至在遇到其他困难时，我也不再害怕了。</p><p>因为每一次把一件事做成，都会增强我们的自信心，提升我们的能力。随着我们做成的事越来越多，我们也就越来越接近山顶了，这时，你会真正地体会到“会当凌绝顶，一览众山小”的感觉。</p><p>好了，到这里，真的要和你说再见了。“此地一为别，孤蓬万里征”，这是李白送别友人时说的，比较忧伤。古代的通讯和交通没有那么便利，分别之后，好友只能是自己独自奋斗了。</p><p>但咱们不是。虽然课程结束了，但是这些内容会持续存在，你可以时不时地复习一下。如果你遇见了什么问题，也欢迎继续给我留言。</p><p>最后，我给你准备了一份结课问卷，希望你花1分钟时间填写一下，聊一聊你对这门课的看法和反馈，就有机会获得“Redis 快捷口令超大鼠标垫”和价值99元的极客时间课程阅码。期待你的畅所欲言。</p><p><a href="https://jinshuju.net/f/deBEiK"><img src="https://static001.geekbang.org/resource/image/7f/de/7f21e7e0fabb48347d59c1e0e1dddcde.jpg?wh=720*505" alt=""></a></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/90/8a/288f9f94.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Kaito</span>
  </div>
  <div class="_2_QraFYR_0">历时 4 个月，这个专栏全程跟了下来，除了答疑篇之外的所有文章，每篇都在第一时间认真写留言、解答课后问题，坚持做了 4 个月，也算是把这件“小事”做到了极致，有始有终。<br><br>在写这最后一篇留言时，我也梳理了一下在这个专栏下留下的文字，所有留言 + 2篇加餐文章，输出共计近 6 万字！<br><br>写下这些文字的时间和地点各有不同，有早晚高峰的地铁上，有 996 回家的出租车上，有凌晨两三点的台灯下，有在医院守夜的病床旁...<br><br>也感谢极客时间这个平台，在这个专栏，除了学到了很多之前没接触到的内容之外，还被邀请写了加餐文章，同时还做了专栏内容的审稿和勘误工作...一路下来，学到很多，成长很多。<br><br>虽然专栏要结束了，但对于我来说是一个新的开始，在专栏更新期间，我建了一个 Redis 交流群，方便大家在学习过程中，遇到问题时可以帮大家答疑。现在这个群已经 200 多人了，每天都在讨论技术问题，讨论的内容质量也很高，没有沦落为水群，也算是小有成效。<br><br>这个群目前计划会一直维护下去，方便有需求的人继续学习专栏内容。想入群的同学可以加我微信（black-rye），也欢迎对技术有热情的同学，和我探讨技术问题~<br><br>最后，也算是在这里立一个 Flag 吧，我打算把这个专栏下的每篇留言，再梳理一遍，整理成文章发出来，把 Redis 常见的问题和不易理解的细节点，用简单易懂的方式讲出来，方便后面遇到类似问题的人学习，也欢迎大家监督我 :)</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-04 01:15:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJgkajjl84U3gcXxwdlcgNZdJoNVibFhDRMtBJibFCzto1eBSPHz83jsxMw5XOicLia2KykbehE7J8p4Q/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>o</span>
  </div>
  <div class="_2_QraFYR_0">https:&#47;&#47;github.com&#47;yanlongLv&#47;redisNotes<br>我在这里把redis知识做了系统的总结，望大家多提提意见</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 知识图谱很不错，赞一个！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-09 18:11:18</div>
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
  <div class="_2_QraFYR_0">整个课程看了两遍，收获颇丰，感谢老师这么高质量的专栏，感谢各位同学无私的交流分享。道阻且长，行则将至。大家江湖再见~</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 加油，江湖再见 :)</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-24 11:29:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/20/5c/ad/3934e3cc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>新年快乐啊</span>
  </div>
  <div class="_2_QraFYR_0">谢谢蒋老师，谢谢课代表Kaito。虽然没能全程跟上节奏，中间一段时间还断开了，但是真的学到了很多很多。从学习Redis，再到Redis学习，真正最重要的是掌握一门技术的学习方法，从被动到主动，点线面网的循序渐进。我还会反复看这们课程，每次看都感觉能有更深一点的了解与掌握。谢谢老师的这碗鸡汤啦，最后大家一起加油加油加油！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢，让我们一起加油！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-14 19:27:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/5d/b5/99f650c6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>皮皮洛</span>
  </div>
  <div class="_2_QraFYR_0">谢谢老师，专栏完全跟上了，一路很顺畅，对工作非常有帮助，也很感谢评论区的大牛解答，江湖再见~</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-04 00:21:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/d1/a5/2bbedc3b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>over</span>
  </div>
  <div class="_2_QraFYR_0">谢谢蒋老师，谢谢课代表Kaito。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-11 13:55:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTI2icbib62icXtibTkThtyRksbuJLoTLMts7zook2S30MiaBtbz0f5JskwYicwqXkhpYfvCpuYkcvPTibEaQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>xuanyuan</span>
  </div>
  <div class="_2_QraFYR_0">注定成为redis的经典课程</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-10 23:49:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c6/39/0b837f63.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>不過勝負</span>
  </div>
  <div class="_2_QraFYR_0">首先，感谢老师，也感谢极客时间为我们请到蒋老师。<br>其次，在整个课程中个人收获很大，系统掌握了redis技术；另外自己也有所感悟，知识的传递到方法的传授，技术思想升华到人生哲学，比较有高度。<br>最后，像这一节课性质的内容，我认为是更重要的，它远远凌驾技术本身的思想以及思维模式以及认知提升。希望极客时间和蒋老师将来针对这类型课程专门开一档连载专栏。<br>谢谢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢，向技术本身学习，这是我们学习技术的一个目标 :)</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-09 09:07:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/05/fc/bceb3f2b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>开心哥</span>
  </div>
  <div class="_2_QraFYR_0">囫囵吞枣刷一遍，期待二刷。感谢老师的精彩讲解，还有认真带路的课代表。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-05 21:09:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/50/d7/f82ed283.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>辣么大</span>
  </div>
  <div class="_2_QraFYR_0">谢谢老师！入职两个多月，之前没有用过Redis，周围的小伙伴也没有会Redis的。我学了您的专栏，现在已经用在实际项目中了，再次感谢！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-07 16:47:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/97/1a/389eab84.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>而立斋</span>
  </div>
  <div class="_2_QraFYR_0">大概20天左右，看了一遍。<br>先来自己回答一下这个问题吧，学习技术究竟意味着什么？<br>现实一点讲，就是让自己会的更多一点吧。提升自身技术水平，获得更多的回报。可能有点狭隘了，你看我就是这么急于否定自己。<br>我理解的技术绝不是一个点，而是一张网。而我要做的就是得自己去亲手织出这张网来。<br>大话谁都会说，更实际一点吧，学就是为用，在相同的场景下经典的套用，在类似的场景下靠用，不同的场景下灵活用，在不同的领域中借用。在思想上，先是形，再是神，最后形神兼备</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-21 11:08:46</div>
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
  <div class="_2_QraFYR_0">从学习技术过渡到向技术学习</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-28 21:55:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/28/7b/43/20cf4dd0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>aimin</span>
  </div>
  <div class="_2_QraFYR_0">完成了学习笔记，一份思维导图和常见面试题分享给大家。<br>https:&#47;&#47;www.processon.com&#47;view&#47;6250c3f9e401fd072efe60cc?fromnew=1</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-22 17:38:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/3f/30/23f6b413.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>五十九秒</span>
  </div>
  <div class="_2_QraFYR_0">一个优秀的计算机系统设计本身就包含了不少人生哲学，赞</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-13 16:49:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/92/6d/becd841a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>escray</span>
  </div>
  <div class="_2_QraFYR_0">我之前能想到的，从学习 Redis 中可以复习计算机底层的体系结构的知识，但是蒋老师已经升华到了做人做事人生哲学的高度。<br><br>单线程，专心做重要的事<br>集群，可扩展<br><br>从做成一件事开始，从学完这门课开始<br><br>虽然已经不能抽奖了，但还是去填了问卷。<br><br>其实我是没有打算学完全部专栏的，一开始完全是因为阿里云的一个活动，而且蒋老师恰好也是第一课的主讲老师。<br><br>结果，专栏看完了，但是阿里云的那个活动半途而废……<br><br>虽然在工作中并没有大规模的使用 Redis，或者说没有使用过大规模的 Redis（集群），但是还是能够从专栏中学到很多。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-05 22:12:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/ec/13/49e98289.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>neohope</span>
  </div>
  <div class="_2_QraFYR_0">老师的课程很棒，收获很多，从Kaito这里也学到了不少经验。工作原因，中断了两次，今天终于看完了。<br><br>托这个专栏的福，我还编译并阅读了部分源码，虽然囫囵吞枣，但redis整体代码的组成现在还算比较有数了，以后遇到问题也有能力去翻翻对应源码了，希望后面有时间去细细看下源码。<br><br>希望老师以后还能有更深入的课程啊，再次感谢！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-25 23:03:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/cb/38/4c9cfdf4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>谢小路</span>
  </div>
  <div class="_2_QraFYR_0">课程非常好，得益于作者高校背景，比一般的作者写的文章更严谨，更合理，推荐。系统的跟了这个课，整体上有更深刻的把握，4个月周期比较长，还需要巩固下。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-04 00:26:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/8b/4b/15ab499a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>风轻扬</span>
  </div>
  <div class="_2_QraFYR_0">通过这个课程，我接触到了redis核心功能的工作原理，收益良多，也决定从此深入的研究redis，以此为基点拓宽自己在计算机系统、文件系统的知识。感谢老师的辛勤付出，谢谢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-05 22:10:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/28/a1/d8/42252c48.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>123</span>
  </div>
  <div class="_2_QraFYR_0">感谢老师</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-30 15:20:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83eqQ98OibIJxJ5XoRo4rJVLhhO8GZoeEjON4FbMpjfM1I93QeQVg4icHXLJjsNrbYXGQptoXcEGFOkRA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lpf</span>
  </div>
  <div class="_2_QraFYR_0">非常感谢老师的精心讲解，通过这个专题，开阔了自己的眼界。看了一遍，对Redis有了一个整体的认识，同时一起学习伙伴的精心留言，感觉自己应该向大家看齐。争取再看一遍，也做好相关的梳理和整理，然后和大家共享。 </div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-20 20:39:24</div>
  </div>
</div>
</div>
</li>
</ul>