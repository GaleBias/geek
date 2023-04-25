<audio title="开篇词 _ 别再让Linux性能问题成为你的绊脚石" src="https://static001.geekbang.org/resource/audio/06/18/06cedcb5e74c88cec921ac260069e918.mp3" controls="controls"></audio> 
<p>你好，我是倪朋飞，一个云计算老兵，Kubernetes项目维护者，主要负责开源容器编排系统Kubernetes在Azure的落地实践。</p><p>一直以来，我都在云计算领域工作。对于服务器性能的关注，可以追溯到我刚参加工作那会儿。为什么那么早就开始探索性能问题呢？其实是源于一次我永远都忘不了的“事故”。</p><p>那会儿我在盛大云工作，忙活了大半夜把产品发布上线后，刚刚躺下打算休息，却突然收到大量的告警。匆忙爬起来登录到服务器之后，我发现有一些系统进程的CPU使用率高达 100%。</p><p>当时我完全是两眼一抹黑，可以说是只能看到症状，却完全不知道该从哪儿下手去排查和解决它。直到最后，我也没能想到好办法，这次发布也成了我心中之痛。</p><p>从那之后，我开始到处查看各种相关书籍，从操作系统原理、到Linux内核，再到硬件驱动程序等等。可是，学了那么多知识之后，我还是不能很快解决类似的性能问题。</p><p>于是，我又通过网络搜索，或者请教公司的技术大拿，学习了大量性能优化的思路和方法，这期间尝试了大量的Linux性能工具。在不断的实践和总结后，我终于知道，怎么<strong>把观察到的性能问题跟系统原理关联起来，特别是把系统从应用程序、库函数、系统调用、再到内核和硬件等不同的层级贯穿起来。</strong></p><!-- [[[read_end]]] --><p>这段学习可以算得上是我的“黑暗”经历了。我想，不仅是我一个人，很多人应该都有过这样的挫折。比如说：</p><ul>
<li>
<p>流量高峰期，服务器CPU使用率过高报警，你登录Linux上去top完之后，却不知道怎么进一步定位，到底是系统CPU资源太少，还是程序并发部分写的有问题？</p>
</li>
<li>
<p>系统并没有跑什么吃内存的程序，但是敲完free命令之后，却发现系统已经没有什么内存了，那到底是哪里占用了内存？为什么？</p>
</li>
<li>
<p>一大早就收到Zabbix告警，你发现某台存放监控数据的数据库主机的iowait较高，这个时候该怎么办？</p>
</li>
</ul><p>这些问题或者场景，你肯定或多或少都遇到过。</p><p>实际上，<strong>性能优化一直都是大多数软件工程师头上的“紧箍咒”</strong>，甚至许多工作多年的资深工程师，也无法准确地分析出线上的很多性能问题。</p><p>性能问题为什么这么难呢？我觉得主要是因为性能优化是个系统工程，总是牵一发而动全身。它涉及了从程序设计、算法分析、编程语言，再到系统、存储、网络等各种底层基础设施的方方面面。每一个组件都有可能出问题，而且很有可能多个组件同时出问题。</p><p>毫无疑问，性能优化是软件系统中最有挑战的工作之一，但是换个角度看，<strong>它也是最考验体现你综合能力的工作之一</strong>。如果说你能把性能优化的各个关键点吃透，那我可以肯定地说，你已经是一个非常优秀的软件工程师了。</p><p>那怎样才能掌握这个技能呢？你可以像我前面说的那样，花大量的时间和精力去钻研，从内功到实战一一苦练。当然，那样可行，但也会走很多弯路，而且可能你啃了很多大块头的书，终于拿下了最难的底层体系，却因为缺乏实战经验，在实际开发工作中仍然没有头绪。</p><p>其实，对于我们大多数人来说，<strong>最好的学习方式一定是带着问题学习</strong>，而不是先去啃那几本厚厚的原理书籍，这样很容易把自己的信心压垮。</p><p>我认为，<strong>学习要会抓重点</strong>。其实只要你了解少数几个系统组件的基本原理和协作方式，掌握基本的性能指标和工具，学会实际工作中性能优化的常用技巧，你就已经可以准确分析和优化大多数的性能问题了。在这个认知的基础上，再反过来去阅读那些经典的操作系统或者其它图书，你才能事半功倍。</p><p>所以，在这个专栏里，我会以<strong>案例驱动</strong>的思路，给你讲解Linux性能的基本指标、工具，以及相应的观测、分析和调优方法。</p><p>具体来看，我会分为5个模块。前4个模块我会从资源使用的视角出发，带你分析各种Linux资源可能会碰到的性能问题，包括 <strong>CPU 性能</strong>、<strong>磁盘 I/O 性能</strong>、<strong>内存性能</strong>以及<strong>网络性能</strong>。每个模块还由浅入深划分为四个不同的篇章。</p><ul>
<li>
<p><strong>基础篇</strong>，介绍Linux必备的基本原理以及对应的性能指标和性能工具。比如怎么理解平均负载，怎么理解上下文切换，Linux内存的工作原理等等。</p>
</li>
<li>
<p><strong>案例篇</strong>，这里我会通过模拟案例，帮你分析高手在遇到资源瓶颈时，是如何观测、定位、分析并优化这些性能问题的。</p>
</li>
<li>
<p><strong>套路篇</strong>，在理解了基础，亲身体验了模拟案例之后，我会帮你梳理出排查问题的整体思路，也就是检查性能问题的一般步骤，这样，以后你遇到问题，就可以按照这样的路子来。</p>
</li>
<li>
<p><strong>答疑篇</strong>，我相信在学习完每一个模块之后，你都会有很多的问题，在答疑篇里，我会拿出提问频次较高的问题给你系统解答。</p>
</li>
</ul><p>第 5 个综合实战模块，我将为你还原真实的工作场景，手把手带你在“<strong>高级战场</strong>”中演练，这样你能把前面学到的所有知识融会贯通，并且看完专栏，马上就能用在工作中。</p><p>整个专栏，我会把内容尽量写得通俗易懂，并帮你划出重点、理出知识脉络，再通过案例分析和套路总结，让你学得更透、用得更熟。</p><p>明天就要正式开课了，开始之前，我要把何炅说过的那句我特别认同的鸡汤送给你，“<strong>想要得到你就要学会付出，要付出还要坚持；如果你真的觉得很难，那你就放弃，如果你放弃了就不要抱怨。人生就是这样，世界是平衡的，每个人都是通过自己的努力，去决定自己生活的样子。</strong>”</p><p>不为别的，就希望你能和我坚持下去，一直到最后一篇文章。这中间，有想不明白的地方，你要先自己多琢磨几次；还是不懂的，你可以在留言区找我问；有需要总结提炼的知识点，你也要自己多下笔。你还可以写下自己的经历，记录你的分析步骤和思路，我都会及时回复你。</p><p>最后，你可以在留言区给自己立个Flag，<strong>哪怕只是在留言区打卡你的学习天数，我相信都是会有效果的</strong>。3个月后，我们一起再来验收。</p><p>总之，让我们一起携手，为你交付“Linux性能优化”这个大技能！</p><p><a href="time://mall?url=https%3A%2F%2Fshop18793264.youzan.com%2Fv2%2Fgoods%2F1y7qqgp3ghd2g%3Fdc_ps%3D2347114008676525065.200001"><img src="https://static001.geekbang.org/resource/image/19/bc/19bc90ffcf4b1fba4938727e5bc0ecbc.jpg?wh=1029*315" alt="unpreview"></a></p><p>Linux知识地图2.0典藏版，现货发售2000份，把5米长的图谱装进背包，1分钟定位80%的高频问题。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/e1/14/006e8b9b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>臧天霸</span>
  </div>
  <div class="_2_QraFYR_0">打卡！40了，学点年轻时候没搞明白的知识，坚持下去，改变自己，再老也不迟</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-20 08:41:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/44/19/17fadc62.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>郭蕾</span>
  </div>
  <div class="_2_QraFYR_0">5年前，我还是一名程序员的时候，就经常受到Linux性能问题的困扰。因为生产环境中，一遇到流量高峰，或者不知道其他什么原因，总是会有些问题，比如CPU使用率高，或者内容吃紧或者IO性能上不去等等。<br><br>那这个时候怎么办呢？只能上去看看到底是哪里的问题，首先，大部分问题，都会先排除机器或者操作系统层面的问题，因为这些在上线之初基本都已经验证没问题了，并且这种问题，到开发这边，基本就是说是程序的问题了。<br><br>比如CPU的，一般就会定位到死锁或者并发代码的问题，那这块怎么分析？我当时其实是不太懂的，只是见厉害的人随便敲一些命令，然后也不知道嘴里念叨着什么，然后过一会，他就说大概是哪里的问题，然后我一看，果然是，然后献出了膝盖。<br><br>现在，我开始做教育，就和朋飞（朋飞是非常资深的Linux玩家）一起策划了这个专栏，这个专栏一个是献给和我当年一样懵懂的朋友，也做给现在的自己。<br><br>review稿件的时候，我觉得这里我能懂了，也很自豪，因为老师写的浅显易懂，写的通透啊。<br><br>ps：这个专栏里，有很多案例，这些案例都非常棒，作者也花了很多精力来做这事。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢郭总支持！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-19 17:09:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/42/f7/9dce9a7b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>十三</span>
  </div>
  <div class="_2_QraFYR_0">D1打卡<br>希望能坚持学完这四个月，然后，我想涨工资😉</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 加油！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-19 16:06:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/0f/4f/5040f415.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>祥伟</span>
  </div>
  <div class="_2_QraFYR_0">第一次尝试，竟然忘记使用新人卷。。。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-19 16:22:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/c5/79/57ac0f23.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>动感超人</span>
  </div>
  <div class="_2_QraFYR_0">要做高手，从性能入手</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-19 19:26:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/14/e5/cf4477eb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>shirly</span>
  </div>
  <div class="_2_QraFYR_0">打卡，盛大云是我第一份工作，曾经还是懵懵懂懂的小姑娘，到今天一边挤奶边充电，满满的苍伤感啊。还是要保持积极学习的状态来影响下一代，可以看出作者花了不少心思在这个专栏下，一定不辜负老同事的付出，好好学习天天天天向上，自勉</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 😊</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-21 01:12:31</div>
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
  <div class="_2_QraFYR_0">希望大神讲得深入些，我们都有运维基础和工作经验……linux性能优化方面的书太少了，红帽到有，但讲得一般！期待中……</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-19 21:32:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/74/68/3725546b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Carlos</span>
  </div>
  <div class="_2_QraFYR_0">看到这个专栏，我毫不犹豫的就订上了，希望可以解决我长久以来的疑惑。没有经历过，永远不知道这种问题发生时有多痛苦呀</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 学完这个专栏就轻松了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-19 17:25:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/27/08/e7cc0ceb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>赵赵</span>
  </div>
  <div class="_2_QraFYR_0">我就会点性能理论，上次突然领导让我做性能，然后就是网上恶补，在测试的过程中我最苦恼的是，我不知道CPU，内存，io等使用率达到多少就算有问题，单个接口在高并发下响应时间，吞吐量等这些值的标准，另外还有一个问题就是测试过程中一天每次压响应时间，吞吐量差别还挺大，一次不如一次的值，还有个问题就是产出报告的时候怎么才能给别人答复说，没问题能支持多少并发，这些疑问一直盘旋，我头大死了，老师您能指个方向吗？ps：我上次草草了事了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-22 14:22:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/fb/7f/746a6f5e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Q</span>
  </div>
  <div class="_2_QraFYR_0">看完作者的介绍，感觉很靠谱！我以前也是去啃大部头，但总觉得看得不透，很难和实际工作做结合，而且市面上介绍性能优化这块的书寥寥无几，大多只是谈工具，不谈方法和思路，更不要说结合实际案例去介绍性能优化。我是一名系统工程师，每天都和Linux 系统打交道，给开发提供支持。希望能学好这门综合性质的课程！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对的，讲工具的书比较多，但还需要串起来才能解决实际的问题</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-19 19:20:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/e9/0b/1171ac71.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>WL</span>
  </div>
  <div class="_2_QraFYR_0">我要加油, 认认真真把这个专栏读懂, 我要涨工资</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-23 08:28:57</div>
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
  <div class="_2_QraFYR_0">打卡！！！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-21 19:55:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/ff/8c/a6a2b26b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Cordova</span>
  </div>
  <div class="_2_QraFYR_0">刚来北京，为了稳住脚立马进了一家团队不大的软件公司，发现来了之后不仅要开发，还要做运维，虽然公司项目日活还不错，但是由于项目有三年以上了，却还是简单依靠nginx做负载，当时就查了cpu，发现mysql占cpu200%-300% 我的天… 立马把mysql执行优化一下算是稳下来了… 后面还做了项目日志的切割，想想一个日志1-3个G的文件流不释放、也挺可怕的… 很幸运碰到倪老师这门课程，虽然已经更新了三十多讲，但是只要能碰到，就已经很幸运了！希望后面能跟随倪老师思路，好好学习，好好进步，好好解决项目的各种问题！也希望能与各位好好交流、共同进步！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-02 21:35:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/0f/f3/7fbdcd48.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>DUO</span>
  </div>
  <div class="_2_QraFYR_0">运维工程师一枚，日常维护服务器，希望能有所收获</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-19 20:30:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/15/26/a7ffa2a2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>籽籽</span>
  </div>
  <div class="_2_QraFYR_0">微软也开发linux吗？  怎么觉得在毁三观呢？ linux服务器出问题给微软打电话的话，有人管吗？  给钱找哪个部门合适？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-20 07:47:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/10/7a/b15e6572.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>:(){ :|: };:</span>
  </div>
  <div class="_2_QraFYR_0">应届毕业生，刚工作4个月，虽然当时应聘是Linux相关、没想到负责维护产品是Windows服务器.<br>不过还是放不下一颗爱玩Linux的心～相信原理都是融会贯通的.</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-19 20:44:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/11/b1/8be55aa2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>。。。</span>
  </div>
  <div class="_2_QraFYR_0">最近正好需要  希望三个月下来能 游刃有余</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-19 20:29:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/f0/98/5df126cf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>吕栋</span>
  </div>
  <div class="_2_QraFYR_0">如何理清思路？发现问题后该知道下一步该做什么！才是最重要的，希望能得到答案！！！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这就是专栏要教的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-19 21:42:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/54/9a/76c0af70.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>每天晒白牙</span>
  </div>
  <div class="_2_QraFYR_0">立flag，跟着学</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-19 20:05:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/97/5f/287ae292.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>子非魚</span>
  </div>
  <div class="_2_QraFYR_0">性能问题已经成了我提升技术能力的瓶颈，看了课程目录就毫不犹豫的订阅了。希望跟着大家一起学习，突破瓶颈！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-19 19:01:03</div>
  </div>
</div>
</div>
</li>
</ul>