<audio title="加餐（二） _ 书单推荐：网络原理和 Linux 内核实现" src="https://static001.geekbang.org/resource/audio/e2/46/e24af09796ec670798000ebce759a546.mp3" controls="controls"></audio> 
<p>你好，我是倪朋飞。欢迎来到 Linux 性能优化专栏的加餐时间。</p><p>上一期的专栏加餐，我给你推荐了一些 Linux 入门、体系结构、内核原理再到性能优化的书籍。这里再简单强调一下，主要包括下面这几本。</p><ul>
<li>
<p>Linux基础入门书籍：《鸟哥的Linux私房菜》</p>
</li>
<li>
<p>计算机体系结构书籍：《深入理解计算机系统》</p>
</li>
<li>
<p>Linux编程书籍：《Linux程序设计》和《UNIX环境高级编程》</p>
</li>
<li>
<p>Linux内核书籍：《深入Linux内核架构》</p>
</li>
<li>
<p>性能优化书籍：《性能之巅：洞悉系统、企业与云计算》</p>
</li>
</ul><p>你可以通过学习这些书，进一步深入到系统内部，掌握系统的内部原理。这样，再结合我们专栏中的性能优化方法，你就可以更清楚地理解性能瓶颈的根源，以及性能优化的思路。</p><p>根据前面几个模块的学习，你应该也感觉到了，网络知识，要比 CPU、内存和磁盘等更为复杂；想解决相应的性能问题，也需要更多的基础知识来支撑。</p><p>而且，任何一个高性能系统，都是多台计算机通过网络组成的集群系统。网络性能，在大多数情况下，自然也就成了影响整个集群性能的核心因素。</p><p>今天，我就来给你推荐一些，关于网络的原理，以及 Linux 内核实现的书籍。</p><h2>计算机网络经典教材《计算机网络（第5版）》</h2><p><img src="https://static001.geekbang.org/resource/image/ce/36/cef3bf15fa095140d499ba56fe4f2e36.png?wh=586*800" alt=""></p><p>既然想优化网络的性能，那么，第一步当然还是要熟悉网络本身。所以，今天我推荐的第一本书，就是一本国内外广泛使用的经典教材——《计算机网络（第5版）》。</p><!-- [[[read_end]]] --><p>这本书按照网络协议模型，自下而上地介绍了计算机网络的基本原理。其中，涵盖范围广是其最大的特点，内容包括了物理层、数据链路层、访问控制层、网络层、传输层和应用层等，是理解计算机网络工作原理的重要参考书。</p><h2>网络协议必读书籍《TCP/IP详解 卷1：协议》</h2><p><img src="https://static001.geekbang.org/resource/image/07/56/07732b5c083e68874e0796a6ba708f56.png?wh=475*676" alt=""></p><p>掌握了计算机网络的基本原理后，接下来就要深入了解，TCP/IP 协议族中各个协议的原理。在这一点上，《TCP/IP详解 卷1：协议》，是当之无愧的圣经级书籍。</p><p>这本书按照 TCP/IP 协议族，也是自下而上介绍了各种协议的原理，并且还穿插了大量的实例，帮你更透彻地理解相关知识。我们分析网络性能时经常碰见的那些协议，这本书都有讲解，比如 ARP、ICMP、路由、TCP、UDP、NAT、DNS 等等。</p><p>无论是想学习掌握，各种网络协议的工作原理；还是更直接落实在工作上，分析优化复杂环境中的网络性能问题，这本书都是你必不可少的宝典。</p><h2>Wireshark 书籍《Wireshark网络分析就这么简单》和《Wireshark网络分析的艺术》</h2><p><img src="https://static001.geekbang.org/resource/image/fe/79/feaf5c9f1b5dd8c4a1546344c67e3979.png?wh=798*1000" alt=""><img src="https://static001.geekbang.org/resource/image/27/f6/278f19c944ae955de49575bca3fde0f6.png?wh=700*880" alt=""></p><p>在学习网络协议时，最大的难点，就是这些协议初学比较抽象，要理解它们的原理也比较困难。这时，如果可以借助 Wireshark 提供的图形界面，你就可以更直观形象地认识这些协议。</p><p>《Wireshark网络分析就这么简单》和《Wireshark网络分析的艺术》，就是两本不错的讲解 Wireshark 使用方法的书籍。这两本书通过诙谐风趣的案例，由浅入深地带你使用 Wireshark，来分析常见的网络问题。</p><p>正如我所说，通俗易懂是其最大特点，相对前面两本大部头来说，你读起来会轻松很多。这两本书在内容上有些重合，内容范围也并不算丰富，但作为入门书籍，却实实在在可以带你，更轻松地理解常见网络问题的分析方法。</p><h2>网络编程书籍《UNIX网络编程》</h2><p><img src="https://static001.geekbang.org/resource/image/74/7e/74f218f137c7a61d7cb40c117831137e.png?wh=600*841" alt=""></p><p>熟悉了协议后，那么接下来自然就是要看，怎么使用这些网络协议，来开发各式各样的应用程序，也就是网络编程。在 Linux 中，我们需要通过套接字接口，跟网络协议栈交互。所以，这里我推荐的是一本介绍套接字接口的书籍——《UNIX网络编程》。</p><p>这本书为你详细介绍了，各种套接字 API 的使用方法，还包含了大量可以直接运行的实例。如果你是一个想实现高性能网络的开发者，这本书是很不错的参考。</p><p>《UNIX网络编程》主要介绍了套接字接口的使用方法，但并不包括 Linux 内核网络协议栈的实现方法。不过没关系，网络协议栈相关内容，我们上一期加餐推荐过的《深入Linux内核架构》中，就已经包括了，所以你不需要再借助其他内核书籍。</p><p><img src="https://static001.geekbang.org/resource/image/e1/5e/e1ed53283b51ed81a96b9c9d2e72d65e.png?wh=398*500" alt=""></p><p>最后，我还是想补充一句，读书不在多，而在于精。哪怕只是啃下我推荐的这几本，你能获得的，一定是质的飞跃。</p><p>今天推荐的这些书里，你可能会觉得有些书很难，还觉得有些知识过时了。但你要知道，核心的网络原理基本没有太大变化，总是不过时的。并且网络本身，也是现代互联网和各种高可用、可扩展架构的基石。多花点儿时间坚持学和练，效果一定显而易见。</p><p>同时，在进入最后的实战进阶篇前的这几天，我也希望你能抽出时间，来复习或者补全专栏前面的知识。虽然总有人自我调侃，说技术类的东西，学了不一定会用，但是反过来说，不去学，一定不会用。坚持下去，相信在专栏结课时，我们一起，一定能看到一个更好的你。</p><p>行动起来吧！</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/57/6e/b6795c44.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>夜空中最亮的星</span>
  </div>
  <div class="_2_QraFYR_0">学了老师的课，是时候肯一下大部头的书了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-08 06:56:35</div>
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
  <div class="_2_QraFYR_0">打卡day48<br>网络的书具有神奇的催眠作用😂</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 😄 一开始可能会觉得抽象，使用 Wireshark 这些图形界面的工具辅助学习可能会效果好些</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-08 08:09:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/2f/f4/2dede51a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小老鼠</span>
  </div>
  <div class="_2_QraFYR_0">计算机网络经典教材《计算机网络（第 5 版）》是不是讲IOS七层协议？我1996年大学学的，现在出入大吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 基础原理不会变化很大，保质期很长😊</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-28 17:49:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKzqiaZnBw2myRWY802u48Rw3W2zDtKoFQ6vN63m4FdyjibM21FfaOYe8MbMpemUdxXJeQH6fRdVbZA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>kissingers</span>
  </div>
  <div class="_2_QraFYR_0">网络还是值得投入时间的知识。长期有效 变化更新相对慢 适用面广。加油！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-08 16:11:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/64/05/6989dce6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我来也</span>
  </div>
  <div class="_2_QraFYR_0">[D48打卡]<br>还是先把手上已有的书看完了再买吧.<br>要不然买了也是在那躺着.😂</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-08 10:17:14</div>
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
  <div class="_2_QraFYR_0">读书有点像游戏里的加技能点，技能点是有限的，要根据计划、设计、自身角色的特点，有针对性的对书籍进行“过滤”，构建自己的技术体系，有自己的特色。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-03 11:27:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/b2/71/76481005.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Kingdom</span>
  </div>
  <div class="_2_QraFYR_0">好多都是大学的原课本啊，大学的时候总是学不下去，工作了才知道大学基础课程的重要性</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，基础原理是必须要掌握的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-24 10:18:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/22/c1/d402bfbf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>一生一世</span>
  </div>
  <div class="_2_QraFYR_0">老师出书了？我像拿到这门课的书</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-30 07:57:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/75/ba/ac35ac17.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>chich chung kai</span>
  </div>
  <div class="_2_QraFYR_0">web页面终于优化了，看起来舒服多了，更加便利了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-09 11:07:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/47/00/3202bdf0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>piboye</span>
  </div>
  <div class="_2_QraFYR_0">bgp有什么好书</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-25 01:09:43</div>
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
  <div class="_2_QraFYR_0">谢谢老师带我起飞</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-13 18:58:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/3f/0d/1e8dbb2c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>怀揣梦想的学渣</span>
  </div>
  <div class="_2_QraFYR_0">TCP&#47;IP那个，要么自己买原版，要么在图书馆看，千万别白嫖网上那些电子版，目前网上有些电子版，里面有残缺的，还有一些印错的。看的时候还是慎重。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-08 22:19:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1c/00/3b/2543b3ca.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>苏煌</span>
  </div>
  <div class="_2_QraFYR_0">打卡，看不完的书</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-27 11:03:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/c3/48/3a739da6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>天草二十六</span>
  </div>
  <div class="_2_QraFYR_0">wireshark功能强大，入门使用较难。工具类的出书，也足以说明～</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-20 13:48:09</div>
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
  <div class="_2_QraFYR_0">加餐打卡，最近自己也在做性能优化方面的一些事情，虽然还没有看全，但是已经深受帮助，这是我认为最划算的一门课了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-23 14:26:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>如果</span>
  </div>
  <div class="_2_QraFYR_0">打卡</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-11 10:52:45</div>
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
  <div class="_2_QraFYR_0">突然发现书籍中中高级的3本都买了：初中级的4本反而没看过，下手-补基础了；怪不得内核我啃到崩溃，原来是漏了底层、、、</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-25 09:52:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/f3/2d/4b7f12b6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>死后的天空</span>
  </div>
  <div class="_2_QraFYR_0">TCP&#47;IP协议是当初考NP的时候买的，第一本看完了，第二本看了一点，UNIX网络编程买来，信誓旦旦的说每天坚持看10页，但是看了200也就坚持不住了 T _ T。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 😊加油</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-13 13:52:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/4d/49/28e73b9c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>明翼</span>
  </div>
  <div class="_2_QraFYR_0">这样书有的自己买了躺起，看了一部分就不想看了<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-10 22:15:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/5c/cb/3ebdcc49.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>怀特</span>
  </div>
  <div class="_2_QraFYR_0">行动起来，复习一遍！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-08 18:27:12</div>
  </div>
</div>
</div>
</li>
</ul>