<audio title="开篇词 _ 跟着学，你也能成为Go语言高手" src="https://static001.geekbang.org/resource/audio/8d/3a/8d3fd57db02767dfb8b33aae4c53003a.mp3" controls="controls"></audio> 
<p>你好，我是郝林。今天想跟你聊聊我和Go语言的故事。</p><p>Go语言是由Google出品的一门通用型计算机编程语言。作为在近年来快速崛起的编程语言，Go已经成功跻身主流编程语言的行列。</p><p>它的种种亮点都受到了广大编程爱好者的追捧。特别是一些对团队协作有较高要求的公司和技术团队，已经在有意识地大量使用Go语言编程，并且，使用的人群还在持续迅猛增长。</p><p>我个人很喜欢Go语言。我是从2012年底开始关注Go语言的，虽然这个日期与Go语言诞生的2009年11月10日相比并不算早，但我也算得上国内比较早期的使用者了。</p><p>Go程序可以在装有Windows、Linux、FreeBSD等操作系统的服务器上运行，并用于提供基础软件支撑、API服务、Web服务、网页服务等等。</p><p>Go语言也在移动端进行了积极的探索，现在在Android和iOS上都可以运行其程序。另外，Go语言也已经与WebAssembly强强联合，加入了WASM平台。这意味着过不了多久，互联网浏览器也可以运行Go编写的程序了。</p><p>从业务维度看，在云计算、微服务、大数据、区块链、物联网等领域，Go语言早已蓬勃发展。有的使用率已经非常之高，有的已有一席之地。即使是在Python为王的数据科学和人工智能领域，Go语言也在缓慢渗透，并初露头角。</p><!-- [[[read_end]]] --><p>从公司角度看，许多大厂都已经拥抱Go语言，包括以Java打天下的阿里巴巴，更别提深爱着Go语言的滴滴、今日头条、小米、奇虎360、京东等明星公司。同时，创业公司也很喜欢Go语言，主要因为其入门快、程序库多、运行迅速，很适合快速构建互联网软件产品，比如轻松筹、快手、知乎、探探、美图、猎豹移动等等。</p><p>我从2013年开始准备撰写《Go并发编程实战》这本书，在经历了一些艰辛和坎坷之后，本书终于在2014年底由人民邮电出版社的图灵公司正式出版。</p><p>时至今日，《Go并发编程实战》的第2版已经出版一年多了，也受到了广大Go语言爱好者的欢迎。同时，我也发起和维护着一个Go语言爱好者组织GoHackers，至今已有近4000人的规模。我们每年都会举办一些活动，交流技术、互通有无。当然，我们平常都会在一些线上的群组里交流。欢迎你的加入。</p><p>2015年初，我开始帮助公司和团队招聘Go程序员。我面试过的Go程序员应该已经有几百个了。虽然一场面试的交流内容远不止技术能力这种硬技能，更别提只限于一门编程语言。</p><p>但是就事论事，我在这里只说Go语言。在所有的应聘者当中，真正掌握Go语言基础知识的比例恐怕超不过50%，而真正熟悉Go语言高阶技术的比例也不超过30%。当然了，情况是明显一年比一年好的，尤其是今年。</p><p>我写此专栏的初衷是，让希望迅速掌握Go语言的爱好者们，通过一种比较熟悉和友好的路径去学习。我并不想事无巨细地去阐述Go语言规范的每个细节以及其标准库中的每个API，更不想写那种填鸭式的教学文章，我更想去做的是详细论述这门语言的重点和主线。</p><p>我会努力探究我们对新技能，尤其是编程语言的学习方式，并以这种方式一步步带领和引导你去记忆和实践。我几乎总会以一道简单的题目为引子，并以一连串相关且重要的概念和知识为主线，而后再进行扩充，以助你进行发散性的思考。</p><p><span class="orange">我希望用这种先点、后线、再面的方式，帮你占领一个个重要的阵地。别的不敢说，如果你认真地跟我一起走完这个专栏，那么基本掌握Go语言是肯定的。</span></p><p>为什么说基本掌握？因为软件技术，尤其是编程技术，必须经过很多的实践甚至历练才能完全掌握，这需要时间而不能速成。不过，本专栏一定会成为你学习Go语言最重要的敲门砖和垫脚石。</p><p>下面，我们一起浏览一下本专栏的主要模块，一共分成3大模块，5个章节。</p><ul>
<li>
<p>基础概念：我会讲述Go语言基础中的基础，包括一些基本概念和运作机制。它们都应该是你初识Go语言时必须知道的，同时也有助于你理解后面的知识。</p>
</li>
<li>
<p>数据类型和语句：Go语言中的数据类型大都是很有特色的，你只有理解了它们才能真正玩转Go语言。我将和你一起与探索它们的奥妙。另外，我也会一一揭示怎样使用各种语法和语句操纵它们。</p>
</li>
<li>
<p>Go程序的测试：很多程序员总以为测试是另一个团队的事情，其实不然。单元测试甚至接口测试其实都应该是程序员去做的，并且应该受到重视。在Go语言中怎样做好测试这件事？我会跟你说清楚、讲明白。</p>
</li>
<li>
<p>标准库的用法：虽然Go语言提供了自己的高效并发编程方式，但是同步方法依然不容忽视。这些方法集中在<code>sync</code>代码包及其子包中。这部分还涉及了字节和字符问题、OS操控方法和Web服务写法等，这些都是我们在日常工作中很可能会用到的。</p>
</li>
<li>
<p>Go语言拾遗：这部分将会讲述一些我们使用Go语言做软件项目的过程中很可能会遇到的问题，至少会包含两篇文章，是附赠给广大Go语言爱好者的。虽然我已经有一个计划了，但是具体会讲哪些内容我还是选择暂时保密。请你和我一起小期待一下吧。</p>
</li>
</ul><p>我希望本专栏能帮助或推动你去做更多的实践和思考。同时我也希望，你能通过学习本专栏感受到学习的快乐，并能够在应聘Go语言相关岗位的时候更加游刃有余。</p><p>所以，如果学，请深学。我不敢自称布道师，但很愿意去做推广优秀技术的事情。如果我的输出能为你的宝塔添砖加瓦，那将会是我的快乐之源。我也相信这几十篇文章可以做到这一点。</p><p><img src="https://static001.geekbang.org/resource/image/35/48/358e4e8578a706598e18a7dfed3ed648.jpg?wh=1110*549" alt=""></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/ac/11/a7d6bb99.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_49eb3e</span>
  </div>
  <div class="_2_QraFYR_0">虽然我是Java控，但也要支持一下</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我之前也做了8年的Java开发，有空还想学学Kotlin呢。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-09 16:41:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/0f/57/f65f059f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Diviner.</span>
  </div>
  <div class="_2_QraFYR_0">祝早日康复</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-09 12:41:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/7b/df/4ded0f2c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>刘宝峰_DEV</span>
  </div>
  <div class="_2_QraFYR_0">愿早日康复，愿专栏越来越好，也祝愿自己能跟您多学知识😊！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-06 18:32:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/2d/18/918eaecf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>后端进阶</span>
  </div>
  <div class="_2_QraFYR_0">用go语言照着微信支付官方sdk写了一个项目之后，我深深地被go语言的大道至简吸引了，我已经爱上go不能自拔了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-06 23:34:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/47/ce/70d2720e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>老刘2</span>
  </div>
  <div class="_2_QraFYR_0">2015年3月就买了郝林的《Go并发编程实战》，一直都没怎么看，随着区块链中的兴起，golang也将随风口再上一个台阶</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-06 21:44:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/7a/79/a74b95f3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>邋邋遢遢</span>
  </div>
  <div class="_2_QraFYR_0">已经读完《go并发编程》，收益良多。一直期待极客能有专门讲go的大神，当读完小黄书2时还在幻想，如果郝林专门开一栏go语言的课程该多好。没想到还整把这事做了让我很是期待。另外最重要的是，祝郝林早日康复，一起go go go！！！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-06 21:35:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/47/a0/f083a071.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>于恩水</span>
  </div>
  <div class="_2_QraFYR_0">本来没有计划学习golang，看到你的精神，让我不得不参与。学不学不重要，先支持一下吧</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-07 03:27:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/ad/2a/5d0fbab1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>天痕</span>
  </div>
  <div class="_2_QraFYR_0">从14年开始写Go，到现在经历了Go版本不稳定到稳定的过程，期间将大量之前的PHP，java，node的项目都转成了Go的。相信Go以后会越来越好。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-08 09:07:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>bestkf</span>
  </div>
  <div class="_2_QraFYR_0">祝君早日康复！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-07 07:58:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/22/e9/32f5fa34.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>QuincySx</span>
  </div>
  <div class="_2_QraFYR_0">祝早日康复</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-06 21:08:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/ac/a1/43d83698.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>云学</span>
  </div>
  <div class="_2_QraFYR_0">学一门语言最重要的就是学习它的特性和主线，idiomatic effective用法，期待。。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-06 20:57:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/5b/79/d55044ac.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>coder</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，有个问题困扰我一段时间了，看Go编译器的源码会发现一大堆的go代码，那么问题来了，go是怎么做到自举的，在没有go编译器的时候，它是怎么做到自己编译自己的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 所谓自举，指的是自己编译自己。这肯定不是说“自己生成自己”，而是说“自己生成新版的自己”。对于Go语言，从 Go 1.5 版本开始，你可以用当前版本的Go编译器去编译下一个版本的Go语言源码。比如，你可以用 Go 1.12 的编译器去编译将来的 Go 1.13 的源码。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-05 20:12:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/b2/54/2d897b27.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我是一条鱼</span>
  </div>
  <div class="_2_QraFYR_0">兄弟加油</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-06 23:27:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/f9/e4/1bdea4b6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>一梦三四年</span>
  </div>
  <div class="_2_QraFYR_0">PHP程序猿 0基础接触go语言，打卡开始。。。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-29 15:12:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83erkMmuQOedHRiaI8lpzFPiaSjM8CG77I2PUbRKukEic7vic69Ribs7BE7hHU5ccDvPAs2tVJ1ZxXCPXCUw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>BridgetLai</span>
  </div>
  <div class="_2_QraFYR_0">真的有必要学习多门编程语言吗?<br>Java做了几年了.但是现在新兴的Python,Go,Kotlin  他们都有自己擅长的领域.那再学习的必要性有多大呢?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 学习多门编程语言绝对没有坏处，不仅可以开拓思路，还可以扩展知识面。每个编程语言都有自己的编程哲学和设计理念，了解这些并且深入探索一些自己感兴趣的点是很有好处的。学无止境，尤其是计算机软件这个领域。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-30 22:20:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/a2/fd/7bed7a0f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jacky</span>
  </div>
  <div class="_2_QraFYR_0">支持一下、早日康复。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-10 09:55:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/91/b1/fb117c21.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>先听</span>
  </div>
  <div class="_2_QraFYR_0">有时想定义一些数组或者map类型的常量。但是go语言不支持。请问这种情况下有没有什么好的实践方法呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 常量是不可点的量，字典是引用类型，所以是两个阵营的东西。你可以做到包外不可变，也就是说字典谁为私有，然后公开几个只读的函数。或者基于字典再封装一个结构体，先保证这个类型的方法都是只读方法，然后把此类值的原语指针赋给常量。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-30 20:04:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/91/83/7576f967.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>先生</span>
  </div>
  <div class="_2_QraFYR_0">写php的支持一下</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-26 11:48:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/b5/f6/1b34b8f5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>向信-Itegel</span>
  </div>
  <div class="_2_QraFYR_0">普通胰腺炎都是非常遭罪的，我看到了男人的血气阳刚。祝兄弟早日康复，越来越强大！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-06 19:17:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/56/cf/401e8363.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>傻乐</span>
  </div>
  <div class="_2_QraFYR_0">dba,也来学习下</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-10 17:08:09</div>
  </div>
</div>
</div>
</li>
</ul>