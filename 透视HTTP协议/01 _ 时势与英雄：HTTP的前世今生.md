<audio title="01 _ 时势与英雄：HTTP的前世今生" src="https://static001.geekbang.org/resource/audio/94/33/945bf44de1d08d72f9ce94d290401733.mp3" controls="controls"></audio> 
<p>HTTP协议在我们的生活中随处可见，打开手机或者电脑，只要你上网，不论是用iPhone、Android、Windows还是Mac，不论是用浏览器还是App，不论是看新闻、短视频还是听音乐、玩游戏，后面总会有HTTP在默默为你服务。</p><p>据NetCraft公司统计，目前全球至少有16亿个网站、2亿多个独立域名，而这个庞大网络世界的底层运转机制就是HTTP。</p><p>那么，在享受如此便捷舒适的网络生活时，你有没有想过，<span class="orange">HTTP协议是怎么来的？它最开始是什么样子的？又是如何一步一步发展到今天，几乎“统治”了整个互联网世界的呢？</span></p><p>常言道：“<strong>时势造英雄，英雄亦造时势</strong>”。</p><p>今天我就和你来聊一聊HTTP的发展历程，看看它的成长轨迹，看看历史上有哪些事件推动了它的前进，它又促进了哪些技术的产生，一起来见证“英雄之旅”。</p><p>在这个过程中，你也能够顺便了解一下HTTP的“历史局限性”，明白HTTP为什么会设计成现在这个样子。</p><h2>史前时期</h2><p>20世纪60年代，美国国防部高等研究计划署（ARPA）建立了ARPA网，它有四个分布在各地的节点，被认为是如今互联网的“始祖”。</p><p>然后在70年代，基于对ARPA网的实践和思考，研究人员发明出了著名的TCP/IP协议。由于具有良好的分层结构和稳定的性能，TCP/IP协议迅速战胜其他竞争对手流行起来，并在80年代中期进入了UNIX系统内核，促使更多的计算机接入了互联网。</p><!-- [[[read_end]]] --><h2>创世纪</h2><p><img src="https://static001.geekbang.org/resource/image/a2/9a/a2960d0e44ef6a8fedd4e9bb836e049a.jpg?wh=425*239" alt="unpreview"></p><center><span class="reference">蒂姆·伯纳斯-李</span></center><p>1989年，任职于欧洲核子研究中心（CERN）的蒂姆·伯纳斯-李（Tim Berners-Lee）发表了一篇论文，提出了在互联网上构建超链接文档系统的构想。这篇论文中他确立了三项关键技术。</p><ol>
<li>URI：即统一资源标识符，作为互联网上资源的唯一身份；</li>
<li>HTML：即超文本标记语言，描述超文本文档；</li>
<li>HTTP：即超文本传输协议，用来传输超文本。</li>
</ol><p>这三项技术在如今的我们看来已经是稀松平常，但在当时却是了不得的大发明。基于它们，就可以把超文本系统完美地运行在互联网上，让各地的人们能够自由地共享信息，蒂姆把这个系统称为“万维网”（World Wide Web），也就是我们现在所熟知的Web。</p><p>所以在这一年，我们的英雄“HTTP”诞生了，从此开始了它伟大的征途。</p><h2>HTTP/0.9</h2><p>20世纪90年代初期的互联网世界非常简陋，计算机处理能力低，存储容量小，网速很慢，还是一片“信息荒漠”。网络上绝大多数的资源都是纯文本，很多通信协议也都使用纯文本，所以HTTP的设计也不可避免地受到了时代的限制。</p><p>这一时期的HTTP被定义为0.9版，结构比较简单，为了便于服务器和客户端处理，它也采用了纯文本格式。蒂姆·伯纳斯-李最初设想的系统里的文档都是只读的，所以只允许用“GET”动作从服务器上获取HTML文档，并且在响应请求之后立即关闭连接，功能非常有限。</p><p>HTTP/0.9虽然很简单，但它作为一个“原型”，充分验证了Web服务的可行性，而“简单”也正是它的优点，蕴含了进化和扩展的可能性，因为：</p><p><span class="orange">“把简单的系统变复杂”，要比“把复杂的系统变简单”容易得多。</span></p><h2>HTTP/1.0</h2><p>1993年，NCSA（美国国家超级计算应用中心）开发出了Mosaic，是第一个可以图文混排的浏览器，随后又在1995年开发出了服务器软件Apache，简化了HTTP服务器的搭建工作。</p><p>同一时期，计算机多媒体技术也有了新的发展：1992年发明了JPEG图像格式，1995年发明了MP3音乐格式。</p><p>这些新软件新技术一经推出立刻就吸引了广大网民的热情，更的多的人开始使用互联网，研究HTTP并提出改进意见，甚至实验性地往协议里添加各种特性，从用户需求的角度促进了HTTP的发展。</p><p>于是在这些已有实践的基础上，经过一系列的草案，HTTP/1.0版本在1996年正式发布。它在多方面增强了0.9版，形式上已经和我们现在的HTTP差别不大了，例如：</p><ol>
<li>增加了HEAD、POST等新方法；</li>
<li>增加了响应状态码，标记可能的错误原因；</li>
<li>引入了协议版本号概念；</li>
<li>引入了HTTP Header（头部）的概念，让HTTP处理请求和响应更加灵活；</li>
<li>传输的数据不再仅限于文本。</li>
</ol><p>但HTTP/1.0并不是一个“标准”，只是记录已有实践和模式的一份参考文档，不具有实际的约束力，相当于一个“备忘录”。</p><p>所以HTTP/1.0的发布对于当时正在蓬勃发展的互联网来说并没有太大的实际意义，各方势力仍然按照自己的意图继续在市场上奋力拼杀。</p><h2>HTTP/1.1</h2><p>1995年，网景的Netscape Navigator和微软的Internet Explorer开始了著名的“浏览器大战”，都希望在互联网上占据主导地位。</p><p><img src="https://static001.geekbang.org/resource/image/94/42/9470d41cab80f36438ebb06a71672242.png?wh=1142*640" alt="unpreview"></p><p>这场战争的结果你一定早就知道了，最终微软的IE取得了决定性的胜利，而网景则“败走麦城”（但后来却凭借Mozilla Firefox又扳回一局）。</p><p>“浏览器大战”的是非成败我们放在一边暂且不管，不可否认的是，它再一次极大地推动了Web的发展，HTTP/1.0也在这个过程中经受了实践检验。于是在“浏览器大战”结束之后的1999年，HTTP/1.1发布了RFC文档，编号为2616，正式确立了延续十余年的传奇。</p><p>从版本号我们就可以看到，HTTP/1.1是对HTTP/1.0的小幅度修正。但一个重要的区别是：它是一个“正式的标准”，而不是一份可有可无的“参考文档”。这意味着今后互联网上所有的浏览器、服务器、网关、代理等等，只要用到HTTP协议，就必须严格遵守这个标准，相当于是互联网世界的一个“立法”。</p><p>不过，说HTTP/1.1是“小幅度修正”也不太确切，它还是有很多实质性进步的。毕竟经过了多年的实战检验，比起0.9/1.0少了“学术气”，更加“接地气”，同时表述也更加严谨。HTTP/1.1主要的变更点有：</p><ol>
<li>增加了PUT、DELETE等新的方法；</li>
<li>增加了缓存管理和控制；</li>
<li>明确了连接管理，允许持久连接；</li>
<li>允许响应数据分块（chunked），利于传输大文件；</li>
<li>强制要求Host头，让互联网主机托管成为可能。</li>
</ol><p>HTTP/1.1的推出可谓是“众望所归”，互联网在它的“保驾护航”下迈开了大步，由此走上了“康庄大道”，开启了后续的“Web 1.0”“Web 2.0”时代。现在许多的知名网站都是在这个时间点左右创立的，例如Google、新浪、搜狐、网易、腾讯等。</p><p>不过由于HTTP/1.1太过庞大和复杂，所以在2014年又做了一次修订，原来的一个大文档被拆分成了六份较小的文档，编号为7230-7235，优化了一些细节，但此外没有任何实质性的改动。</p><h2>HTTP/2</h2><p>HTTP/1.1发布之后，整个互联网世界呈现出了爆发式的增长，度过了十多年的“快乐时光”，更涌现出了Facebook、Twitter、淘宝、京东等互联网新贵。</p><p>这期间也出现了一些对HTTP不满的意见，主要就是连接慢，无法跟上迅猛发展的互联网，但HTTP/1.1标准一直“岿然不动”，无奈之下人们只好发明各式各样的“小花招”来缓解这些问题，比如以前常见的切图、JS合并等网页优化手段。</p><p>终于有一天，搜索巨头Google忍不住了，决定“揭竿而起”，就像马云说的“如果银行不改变，我们就改变银行”。那么，它是怎么“造反”的呢？</p><p>Google首先开发了自己的浏览器Chrome，然后推出了新的SPDY协议，并在Chrome里应用于自家的服务器，如同十多年前的网景与微软一样，从实际的用户方来“倒逼”HTTP协议的变革，这也开启了第二次的“浏览器大战”。</p><p>历史再次重演，不过这次的胜利者是Google，Chrome目前的全球的占有率超过了60%。“挟用户以号令天下”，Google借此顺势把SPDY推上了标准的宝座，互联网标准化组织以SPDY为基础开始制定新版本的HTTP协议，最终在2015年发布了HTTP/2，RFC编号7540。</p><p>HTTP/2的制定充分考虑了现今互联网的现状：宽带、移动、不安全，在高度兼容HTTP/1.1的同时在性能改善方面做了很大努力，主要的特点有：</p><ol>
<li>二进制协议，不再是纯文本；</li>
<li>可发起多个请求，废弃了1.1里的管道；</li>
<li>使用专用算法压缩头部，减少数据传输量；</li>
<li>允许服务器主动向客户端推送数据；</li>
<li>增强了安全性，“事实上”要求加密通信。</li>
</ol><p>虽然HTTP/2到今天已经四岁，也衍生出了gRPC等新协议，但由于HTTP/1.1实在是太过经典和强势，目前它的普及率还比较低，大多数网站使用的仍然还是20年前的HTTP/1.1。</p><h2>HTTP/3</h2><p>看到这里，你可能会问了：“HTTP/2这么好，是不是就已经完美了呢？”</p><p>答案是否定的，这一次还是Google，而且它要“革自己的命”。</p><p>在HTTP/2还处于草案之时，Google又发明了一个新的协议，叫做QUIC，而且还是相同的“套路”，继续在Chrome和自家服务器里试验着“玩”，依托它的庞大用户量和数据量，持续地推动QUIC协议成为互联网上的“既成事实”。</p><p>“功夫不负有心人”，当然也是因为QUIC确实自身素质过硬。</p><p>在去年，也就是2018年，互联网标准化组织IETF提议将“HTTP over QUIC”更名为“HTTP/3”并获得批准，HTTP/3正式进入了标准化制订阶段，也许两三年后就会正式发布，到时候我们很可能会跳过HTTP/2直接进入HTTP/3。</p><h2>小结</h2><p>今天我和你一起跨越了三十年的历史长河，回顾了HTTP协议的整个发展过程，在这里简单小结一下今天的内容：</p><ol>
<li><span class="orange">HTTP协议始于三十年前蒂姆·伯纳斯-李的一篇论文；</span></li>
<li><span class="orange">HTTP/0.9是个简单的文本协议，只能获取文本资源；</span></li>
<li><span class="orange">HTTP/1.0确立了大部分现在使用的技术，但它不是正式标准；</span></li>
<li><span class="orange">HTTP/1.1是目前互联网上使用最广泛的协议，功能也非常完善；</span></li>
<li><span class="orange">HTTP/2基于Google的SPDY协议，注重性能改善，但还未普及；</span></li>
<li><span class="orange">HTTP/3基于Google的QUIC协议，是将来的发展方向。</span></li>
</ol><p>希望通过今天的介绍，你能够对HTTP有一个初步但清晰的印象，知道了“来龙”才能更好地知道“去脉”。</p><h2>课下作业</h2><ol>
<li>你认为推动HTTP发展的原动力是什么？</li>
<li>你是怎么理解HTTP（超文本传输协议）的？</li>
</ol><p>欢迎你把自己的答案写在留言区，与我和其他同学一起讨论。暂时回答不出来也不要紧，你可以带着这些问题在后续的课程里寻找答案。</p><p>如果你觉得有所收获，欢迎你把文章分享给你的朋友。</p><p><img src="https://static001.geekbang.org/resource/image/20/00/2016d4e0ab9698c3d2cd560f0fa89a00.png?wh=750*824" alt="unpreview"></p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/b7/cb/18f12eae.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>不靠谱～</span>
  </div>
  <div class="_2_QraFYR_0">用户需求推动技术发展</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Great。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-29 19:21:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/54/f3/2216f04f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我叫不知道</span>
  </div>
  <div class="_2_QraFYR_0">1.协议标准不同于原理，原理是相对稳定的，而标准则需要与时俱进，随着业务和技术发展中出现的新问题一起变化。在实际商业应用、竞争和实践中反复打磨，让协议标准适应不断发展变化的实际业务问题，而不是让日渐庞大复杂的业务去适应受限于特定时空因素的标准。<br>标准的诞生和发展一方面是基于具体业务需要和技术发展，另一方面是为了统一游戏规则，让各厂商的软硬件产品可以方便地“互联”，降低“沟通”和“翻译”的成本，提高网络互联的开放性。<br>2.http对厂商和技术人员来说，某种意义上，是一种技术语言，便于通过软硬件相互沟通；对用户来说……编不下去了<br>个人的一点拙见，还请大佬点评指正～</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 写的很好，go on。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-29 19:13:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/78/51/4790e13e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Smallfly</span>
  </div>
  <div class="_2_QraFYR_0">老师文中说，HTTP2.0 的新特点：“二进制协议，不再是纯文本”。<br><br>那像 HTTP&#47;1.1 中的 application&#47;octet-stream 和 multipart&#47;form-data 也属于本文格式吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，只要是HTTP&#47;1.1，就都是文本格式，虽然里面的数据可能是二进制，但分隔符还是文本，这些都会在“进阶篇”里讲。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-29 19:53:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/bc/6d/f6f0a442.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>汤小高</span>
  </div>
  <div class="_2_QraFYR_0">超文本和文本有什么区别吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 超文本有超链接，是网状结构，文本是线性结构。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-29 21:30:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/2f/fd/dead7549.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>codewu</span>
  </div>
  <div class="_2_QraFYR_0">老师提的问题很好，我之前都没考虑过～<br><br>比如，<br>ftp、telnet使用前必须输入用户名和密码，更偏向于一对一的使用，对用户来说不够开放。<br><br>而http设计之初就是对所有用户开放，而且还统一了访问方式，使用门槛很低，就会有很多人用。至于后续各种优化和功能的添加，那都是顺其自然的事了。<br><br>所以总的来说，是http对用户的开放性，使得用户推动其蓬勃发展。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 真理越辩越明，欢迎多讨论发言。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-30 12:54:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/a5/f0/8648c464.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Joker</span>
  </div>
  <div class="_2_QraFYR_0">从历史的进程来看，就是互联网的用户推动协议的发展的。刚刚开始只有文本，都只是文字；后来有了超文本，不仅仅是文字；后来嫌弃速度慢，有了持久连接，缓存机制；后来为了安全，有了加密通信。一切都是以用户的需求为导向的，用户的需要越来越高，协议就越来越高级，越来越完善。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 说的很好，互联网上的一切都是这么发展的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-29 21:30:04</div>
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
  <div class="_2_QraFYR_0">1：你认为推动 HTTP 发展的原动力是什么？<br>       我觉得推动HTTP协议发展的原动力是人类的好奇心＋逐利，那为啥其他协议没有一统互联网江湖呢？HTTP简单、开放、顺应潮流，初心满足了人类天生好奇的需求，顺势满足了大家都能从中获利的需求，由于这两点支持的力量就变得强大无比，进一步反而增加了她一统互联网天下的能力。<br><br>2：你是怎么理解 HTTP（超文本传输协议）的？<br>超文本传输协议＝超文本＋传输＋协议，协议即约定，HTTP就是约定超文本怎么传输的。初心就是分享信息，所以，简单、开放、有求有应，只针对文本，后来出现了音频、视频、动画、图片、超链接这些玩意，比纯文本复杂了一些，不过初心不改，所以，原则未变，只是需要调整一下适应这些正当其时的需求而已。<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-22 22:42:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/cd/c1/19531313.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>innovationmech</span>
  </div>
  <div class="_2_QraFYR_0">希望破冰篇和基础篇能更新快点</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 慢慢来吧，还是要照顾很多对http不太了解的同学，你可以“养肥了再看”。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-29 23:04:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/24/b9/b9/9e4d7aa4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>乘风破浪</span>
  </div>
  <div class="_2_QraFYR_0">关于host头和主机托管的关系，尝试自己理解了一下，请老师指正<br>一个主机&#47;IP地址可以运行多个网站，即虚拟主机<br>		www.a.com<br>		www.b.com<br>		…<br>它们在浏览器地址栏无论输入www.a.com或www.b.com都将解析到同一个IP地址<br>但不同网站的浏览器发起的访问请求，host填的URI不一样<br>如www.a.com请求host里填的是www.a.com, www.b.com填的www.b.com<br>这样就把同一个IP的不同网站（虚拟主机）区分开了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 理解的非常好，go on！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-25 15:19:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/f8/e5/7c06cd59.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>二楞子</span>
  </div>
  <div class="_2_QraFYR_0">1.用户需求<br>2.我理解的http 类似河里的船 传输东西用的工具</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 比喻有点像，tcp是河，http是船。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-30 08:06:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/07/8c/0d886dcc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>蚂蚁内推+v</span>
  </div>
  <div class="_2_QraFYR_0">1. HTTP 发展的原动力我认为还是人们对信息获取的需求升级，从单一的文本到静态图片，再到动态视频、音乐，更到未来的 AR&#47;VR，以及与日俱增的风险，因此对于安全性、隐私的保护，为了满足更高层级的需要，HTTP 协议本身也要与时俱进；<br>2. HTTP 的本质是 P（Protocol），即一个协议，定义了服务端与客户端数据交互的标准。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: ✅</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-02 17:51:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/27/35/ba972e11.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>因缺思厅</span>
  </div>
  <div class="_2_QraFYR_0">看完了，觉得很赞。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-29 19:51:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/27/3b/fd/2e2feec3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>💍</span>
  </div>
  <div class="_2_QraFYR_0">课后总结：<br>  http 0.9 ： 功能较为简单， 且传输格式只能是纯文本格式 ， 只支持get请求 ， 请求完毕立即关闭请求<br>  http 1.0： 1. 增加了head， post 等请求<br>                  2. 增加返回状态码<br>                  3. 引入版本号概念<br>                  4. 增加了http头部的概念， 提高灵活度<br>                  5. 传输的文本不再限制纯文本格式， 增加了音视频等格式<br>    http1.1： 1. 增加了put， delete请求 <br>                   2. 增加缓存管理<br>                   3. 明确链接管理， 推动持久链接 <br>                   4. 允许响应数据分块<br>                   5. 强制要求host头<br>   http 2.0 ： 1. 二进制协议<br>                    2. 支持同时发送多个请求<br>                    3. 压缩头部， 减少数据传输量<br>                    4.  允许服务端主动推送数据<br>                    5. 增加安全性， 增加加密通信<br><br>http不同版本特点：<br>    http 0.9：  版本功能单一简单， 因前期设计简单， 后期版本更新就会比较容易<br>    http 1.0：  功能相对0.9 更加丰富， 但并不是统一标准 只是一份文档， 不具约束力<br>    http 1.1 ： 相对1.0 添加了小规模更新， 但是它算是一份http统一的标准， 所有的http请求都需严格按照这个标准<br>   http 2.0 ： 相对1.1 提升了http请求的性能和安全性 <br><br>               </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: awesome！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-22 16:08:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/7d/3c/fdab3611.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>灰</span>
  </div>
  <div class="_2_QraFYR_0">HTTP 1.1 的 强制要求 Host 头，让互联网主机托管成为可能。<br><br>难道不是总是要经过DNS解析吗，如果都要经过DNS解析的话，Host的设计和主机托管有什么关系。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 说起来比较复杂，在同一个IP地址上可能会托管有多个主机服务，在域名解析后到达服务器的时候，就需要用域名来选择。如果你用过Nginx，可能就会比较好理解。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-30 17:48:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/36/d2/c7357723.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>发条橙子 。</span>
  </div>
  <div class="_2_QraFYR_0">突然想到一个点 ，是不是因为2.0之前数据都是以文本形式传输 ，所以才命名为 超文本传输协议 。 那后来2.0可以支持二进制形式传输了 ， 实际上HTTP这个命名也不太准确了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个就是“历史遗留问题”了，不过也不用太在意，比如我们现在说的汽车、火车、轮船，习惯了就好。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-30 09:14:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/5f/73/bb3dc468.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>拒绝</span>
  </div>
  <div class="_2_QraFYR_0">开发至今，只使用到了http的get、post的请求方式，至于put、delete的方式，它们的存在肯定是有原因，至于是什么原因，应用在怎样的场景下，请老师解答下</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: put、delete这些可以用在restful应用里，表示各种对资源的操作。因为HTTP很灵活，也有一些历史遗留问题，不必要强求什么特性都用上。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-29 19:19:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Dang</span>
  </div>
  <div class="_2_QraFYR_0">HTTP 1.1应该已经支持了多连接，但是每个连接都需要遵循request-response然后再req-resp的模式。<br>HTTP 2支持不需要等待resp就可以继续发送多个req。<br><br>这是文章中提到的HTTP 2的特点：2、可发起多个请求，废弃了 1.1 里的管道；的意思么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我说的可能不是太准确，抱歉。<br><br>http&#47;1和http&#47;2都使用的是请求-应答的工作模式，但http&#47;1必须多个请求-应答串行，而http&#47;2有流特性，可以多个请求应答并行，也就是可同时发起多个请求。<br><br>管道特性是http&#47;1里为了解决请求串行的低效率问题而提出的，但比较复杂，很多浏览器和服务器都没有实现，实用价值低，所以到了http&#47;2，因为有了更好的流，所以这个管道机制就废弃了。<br><br>可以结合后面的http&#47;2再理解。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-11 18:13:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/5f/30/4ae82e16.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>wordMan</span>
  </div>
  <div class="_2_QraFYR_0">请问老师，HTTP&#47;2： “二进制协议，不再是纯文本” 这个怎么理解的，如果是指传输的话应该都是二进制啊</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>http&#47;1里的协议头是ASCII，肉眼可读，所以叫纯文本<br><br>而http&#47;2的协议头用二进制位，不能直接看。<br><br>所谓的二进制协议是相当于是否可以人类直接阅读而言的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-13 21:14:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/8d/0e/5e97bbef.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>半橙汁</span>
  </div>
  <div class="_2_QraFYR_0">1，原动力:<br>     时代思想环境的日渐成熟;<br>     与之匹配的相关技术的达标;<br>     求知欲，好奇心等因素催生出的相关技术工种人才;<br>     各个行业的用户对信息流通速率提升的高度关注与需求;(占比较大的因素)<br>2，http<br>    超文本传输协议，可以拆解着来理解(个人拙见):<br>    超(over):  更多，更快，更加便捷等等;<br>    文本(information): 信息，资源等等;<br>   传输(translate): 交互，响应;<br>   协议(role): 规则;<br><br>最后，附上个人的一点疑惑(希望得到各位的响应):<br>http0.9---http1.0---http1.1---http2.0---http3.0<br>对应的相关RFC编号，从无到已知的具体编号，之间有各种联系？<br>RFC7230对应1.1，随意定义的还是通过审核收集意见制定的？<br>     </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 回答的挺好。<br><br>注意，自http1.1之后，版本没有小数点后的次版本号，名字是http&#47;2和http&#47;3，可参考飞翔篇。<br><br>RFC并不是专门为http准备的，而是一系列的国际标准文档，依照发布的顺序编号。<br><br>7230是对2616的一次整理、简化，为http&#47;2的7540做准备。<br><br>作为国际标准，任何rfc都不是随意制定的，而是要经过多次讨论，发布多个草稿，最终表决通过。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-09 12:56:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/5c/de/16695891.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小太阳</span>
  </div>
  <div class="_2_QraFYR_0">大家都说源动力来自用户需求，但是没说清楚是什么需求。<br>我觉得源动力是知识共享和工作协同的需要，也是因为这个原因在研究所里有了万维网，在万维网的基础上又想要利用这个网络来分享知识、同步信息。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 再往深了说就是源自人类自身的群体性，还有沟通、交流的欲望。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-21 22:38:33</div>
  </div>
</div>
</div>
</li>
</ul>