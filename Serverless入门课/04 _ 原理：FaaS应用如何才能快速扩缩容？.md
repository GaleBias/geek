<audio title="04 _ 原理：FaaS应用如何才能快速扩缩容？" src="https://static001.geekbang.org/resource/audio/da/6c/da76ed035c1ff81951b72652dbce4a6c.mp3" controls="controls"></audio> 
<p>你好，我是秦粤。上一讲我们介绍了FaaS的两种进程模型：用完即毁型和常驻进程型，这两种进程模型最大的区别就是在函数执行阶段，函数执行完之后函数实例是否直接结束。同时，我还给你演示了用完即毁型的应用场景，数据编排和服务编排。</p><p>这里我估计你可能会有点疑虑，这两个场景用常驻进程型，应该也可以实现吧？当然可以，但你还记得不，我多次强调用完即毁型是FaaS最纯正的用法。那既然介绍了两种进程模型，为什么我要说用完即毁型FaaS模型比常驻进程型纯正？它背后的逻辑是什么？你可以停下来自己想想。</p><p>要真正理解这个问题，我们需要引入进来复杂互联网应用架构演进的一个重要知识点：扩缩容，这也是我们这节课的重点。</p><p>为了授课需要，我还是会搬出我们之前提到的创业项目“待办任务”Web网站。这一次，需要你动动手，在自己本地的机器上运行下这个项目。项目的代码我已经写好了，放到GitHub上了，你需要把它下载到本地，然后阅读README.md安装和启动我们的应用。</p><p>GitHub地址：<a href="https://github.com/pusongyang/todolist-backend">https://github.com/pusongyang/todolist-backend</a></p><p>我给你简单介绍下我们目前这个项目的功能。这是一个后端项目，前端代码不是我们的重点，当然如果你有兴趣，我的REAME.md里面也有前端代码地址，你可以在待办任务列表里面创建、删除、完成任务。</p><!-- [[[read_end]]] --><p>技术实现上，待办任务数据就存储在了数组里。宏观上看，它是个典型的Node.js传统MVC应用，Control函数就是app.get和app.post；Model我们放在内存里，就是Todos对象；View是纯静态的单页应用代码，在public目录。</p><p>你先想一下，假如我们让200个用户<strong>同时并发访问</strong>你本地开发环境的“待办任务”Web网站首页index.html，你本地的Web网站实例，会出现什么样的场景？如果方便的话，你可以用Apache<span class="orange">[1]</span> 提供的ab工具，压测一下我们的项目。</p><pre><code># 模拟1000个请求，由200个用户并发访问我们启动的本地3001端口
ab -n 1000 -c 200 http://localhost:3001/
</code></pre><p>我来试着描述下你PC此时的状态，首先客户端与PC建立了200个TCP/IP的连接，这时PC还可以勉强承受得住。然后200个客户端同时发起HTTP请求"/ GET"，我们Web服务的主进程，会创建“CPU核数-1”个子进程并发，来处理这些请求。注意，这里CPU核数之所以要减一，是因为有一个要留给主进程。</p><p>例如4核CPU就会创建3条子进程，并发处理3个客户端请求，剩下的客户端请求排队等待；子进程开始处理"/ GET"，命中路由规则，进入对应的Control函数，返回index.html给客户端；子进程发送完index.html文件后，被主进程回收，主进程又创建一个新的子进程去处理下一个客户端请求，直到所有的客户端请求都处理完。具体如下图所示。</p><p><img src="https://static001.geekbang.org/resource/image/bb/b1/bb998d5ac7b84cf0468932afad2448b1.png?wh=1502*662" alt="" title="PC状态示意图"></p><p>理解了这一点，接下来的问题就很简单了。如果我问你，为了提升我们客户端队列的处理速度，我们应该怎么做？我想答案你应该已经脱口而出了。</p><h2>纵向扩缩容与横向扩缩容</h2><p>是的，我们很容易想到最直接的方式就是增加CPU的核数。要增加CPU的核数，我们可以通过升级单台机器配置，例如从4核变成8核，那并发的子进程就有7个了。</p><p>除了直接增加CPU的核数，我们还可以增加机器数（还是增加一个4核的），我们用2台机器，让500个客户端访问一台，剩下500个客户端访问另外一台，这样我们并发的子进程也能增加到6个。</p><p><img src="https://static001.geekbang.org/resource/image/65/51/652b85301f140dc4d3b2b0e35fafa151.png?wh=1184*776" alt="" title="纵向扩缩容 &amp; 横向扩缩容"></p><p>我画了张图，你可以看看。增加或减少单机性能就是纵向扩缩容，纵向扩缩容随着性能提升成本曲线会陡增，通常我们采用时要慎重考虑。而增加或减少机器数量就是横向扩缩容，横向扩缩容成本更加可控，也是我们最常用的默认扩缩容方式。这里我估计很多人知道，为了照顾初学者，所以再啰嗦下。</p><p>你理解了这一点，我们就要增加难度了。因为index.html只是单个文件，如果是数据呢？无论是纵向还是横向扩缩容，我们都需要重启机器。现在待办列表的数据保存在内存中，它每次重启都会被还原到最开始的时候，那我们要如何在扩缩容的时候保存我们的数据呢？</p><p>在讲解这个问题前，我们还是需要简化一下模型。我们先从宏观架构角度去看“待办任务”Web服务端数据流的网络拓扑图，数据请求从左往右，经过我们架构设计的各个节点后，最终获取到它要的数据，然后聚合数据并返回。那在这个过程中，哪些节点可以扩缩容，哪些节点不容易扩缩容呢？</p><p><img src="https://static001.geekbang.org/resource/image/d3/54/d39538136a0fafcc3fa4c7e0d5c8f554.png?wh=1302*570" alt="" title="“待办任务”Web服务端数据流网络拓扑图"></p><h2>Stateful VS Stateless</h2><p>网络拓扑中的节点，我们可以根据是否保存状态分为Stateful和Stateless。Stateful就是有状态的节点，Stateful节点用来保存状态，也就是存储数据，因此Stateful节点我们需要额外关注，需要保障稳定性，不能轻易改动。例如通常数据库都会采用主-从结构，当主节点出问题时，我们立即切换到从节点，让Stateful节点整体继续提供服务。</p><p>Stateless就是无状态的节点，Stateless不存储任何状态，或者只能短暂存储不可靠的部分数据。Stateless节点没有任何状态，因此在并发量高的时候，我们可以对Stateless节点横向扩容，而没有流量时我们可以缩容到0（是不是有些熟悉了？）。Stateful节点则不行，如果面对流量峰值峰谷的流量差比较大时，我们要按峰值去设计Stateful节点来抗住高流量，没有流量时我们也要维持开销。</p><p><img src="https://static001.geekbang.org/resource/image/e2/40/e2a28f3a4dbd473fe2db11c7116b9140.png?wh=1414*658" alt="" title="“待办任务”Web服务端数据流网络拓扑图"></p><p>在我们“待办任务”的项目中，数据库就是典型Stateful节点，因为它要持久化保存用户的待办任务。另外负载均衡也是Stateful节点，就跟我们思维试验中保存客户端队列的主进程一样，它要保存客户端的链接，才能将我们Web应用的处理结果返回给客户端。</p><p>回到我们的进程模型，<strong>用完即毁型是天然的Stateless</strong>，因为它执行完就销毁，你无法单纯用它持久化存储任何值；<strong>常驻进程型则是天然的Stateful</strong>，因为它的主进程不退出，主进程可以存储部分值。</p><p><img src="https://static001.geekbang.org/resource/image/c5/51/c5ee951df1c8c5f9e88d5d44ffdb2551.png?wh=1414*1156" alt="" title="进程模型实例"></p><p>如上图所示，我们将待办任务列表的数据存储在了主进程的内存中，而在FaaS中，即使我们在常驻进程型的主进程中保存了值，它也可能会被云服务商回收。即便我们购买了预留实例，但扩容出来的节点与节点之间，它们各自内存中的数据是无法共享的，这个我们上节课有讲过。</p><p>所以我们要让常驻进程型也变成Stateless，我们就要避免在主进程中保存值，或者只保存临时变量，而将持久化保存的值，移出去交给Stateful的节点，例如数据库。</p><p><img src="https://static001.geekbang.org/resource/image/f6/27/f637868794ce276e85fb209b845da527.png?wh=1738*1162" alt="" title="进程模型实例"></p><p>我们将主进程节点中的数据独立出来，主进程不保存数据，这时我们的应用就变成Stateless。数据我们放入独立出来的数据库Stateful节点，网络拓扑图就是上面这张图。这个例子也就变成了我们上节课讲常驻进程型FaaS的例子，我们在主进程启动时连接数据库，通过子进程访问数据库数据，但这样做的弊端其实也很明显，它会直接增加冷启动时间。那有没有更好的解决方案呢？</p><p>换一种数据持久化的思路，我们为什么非要自己连接数据库呢？我们对数据的增删改查，无非就是子进程复用主进程建立好的TCP链接，发送数据库语句，获取数据。咱们大胆想象下，如果向数据库发送指令，变成HTTP访问数据接口POST、DELETE、PUT、GET，那是不是就可以利用上一课的数据编排和服务编排了？</p><p>是的，铺垫了这么多，就是为了引出我们今天的主角：BaaS化。数据接口的POST、DELETE、PUT、GET其实就是语义化的RESTful API<span class="orange">[2]</span> 的HTTP方法。用MySQL举例，那POST对应CREATE指令，DELETE对应DELETE指令，PUT对应UPDATE指令，GET对应SELECT指令，语义上是一一对应的，因此我们可以天然地将MySQL的操作转为RESTful API操作。</p><p>为了防止有同学误解，我觉得我还是需要补充一下。传统数据库方式，因为TCP链路复用和通信字段冗余低，同样的操作会比HTTP快。FaaS可以直连数据库，但传统数据通过IP形式连接往往很难生效，因为云上环境都是用VPC切割的。所以FaaS直连数据库，我们通常要采用云服务商提供的数据库BaaS服务，但目前很多BaaS服务还不成熟。</p><p>再进一步考虑，既然FaaS不适合用作Stateful的节点，那我们是不是可以将Stateful的操作全部变成数据接口，外移？这样我们的FaaS就可以用我们上一课讲的数据编排，自由扩缩容了。</p><h2>后端应用BaaS化</h2><p>BaaS这个词我们前面已经讲过了，在我看来，BaaS化的核心思想就是将后端应用转换成<strong>NoOps的数据接口</strong>，这样FaaS在SFF层就可以放开手脚，而不用再考虑冷启动时间了。其实我们上一课在讲SFF的时候，后端应用就是一定程度的BaaS化。后端应用接口化只是BaaS化的一小部分，BaaS化最重要的部分是后端数据接口应用的开发人员也可以不再关心服务端运维的事情。</p><p><img src="https://static001.geekbang.org/resource/image/f4/c2/f47b84ac700130339e3422908a9931c2.png?wh=1782*856" alt="" title="SFF示意图"></p><p>回顾一下，<a href="https://time.geekbang.org/column/article/224559">[第 1 课]</a> 中我们说的<strong>Serverless应用 = FaaS+BaaS</strong>，相信此刻你一定会有不一样的感悟了吧。</p><p>BaaS化的概念容易理解，但实际上要实践，将我们的网站后端改造BaaS化，就比较困难，这其中主要的难点在于后端的运维体系如何Serverless化，改造后端BaaS化的内容相比FaaS的SFF要复杂得多。在本专栏后续的课程中，我将通过我们的创业项目“待办任务”Web服务逐步演进，带你一起学习后端BaaS化，不过你也不必有压力，因为我们在学习FaaS的过程中已经掌握的知识点，也是适用于后端BaaS化的。</p><p>另外值得一提的是，云服务商也在大力发展BaaS，例如AWS提供的DynamoDB服务或Aurora服务。数据库就是BaaS化的，我们无需关心服务端运维，也无需关心IP，我们只要通过域名和密钥访问我们的DB，就像使用数据编排一样。而且BaaS的阵营还在不停壮大，不要忘了我们手中还有服务编排这一利器。</p><h2>总结</h2><p>用完即毁型之所以比常驻进程型更加纯正，就是因为常驻进程型往往容易误导我们，让我们以为它像PaaS一样受控，可以用作Stateful节点，永久存储数据。实际上，在FaaS中即使我们采用常驻进程型，我们的函数实例还是会被云服务商回收。</p><p>就像我们的“待办任务”Web网站的例子，将数据Todos放在内存中，我们每次重启都会重置一样。我们用数据编排的思路，将后端对数据库的操作转为数据接口，那我们就可以将FaaS中的数据存储移出到后端应用上，采用上一节课讲的数据编排跟我们的后端进行交互。但后端应用我们不光要做成数据接口，还要BaaS化，让后端工程师在开发过程中，也能不用关心服务端运维。</p><p>现在我们再来回顾一下这节课的知识点：</p><ol>
<li>扩缩容我们可以选择纵向扩缩容和横向扩缩容，纵向扩缩容就是提升单机性能，价格上升曲线陡峭，我们通常要慎重选择；横向扩缩容就是提升机器数量，价格上升平稳，也是我们常用的默认扩缩容方式。</li>
<li>在网络拓扑图中，Stateful是存数据的节点；Stateless是处理数据的节点，不负责保存数据。只有Stateless节点才能任意扩缩容，Stateful节点因为是保存我们的重要数据，所以我们要谨慎对待。如果我们的网络拓扑节点想自由扩缩容，则需要将这个节点的数据操作外移到专门的Stateful节点。</li>
<li>我们的FaaS访问Stateful节点，那我们就希望Stateful节点对FaaS提供数据接口，而不是单纯的数据库指令，因为数据库连接会增加FaaS的额外开支。另外为了方便后端工程师开发，我们需要将Stateful节点BaaS化，BaaS化的内容，我们将在后续的课程中展开。</li>
</ol><h2>作业</h2><p>本节课我们创业项目“待办任务”中的数据处理并没有按照RESTFul API的HTTP语义化来开发，不太规范。作业中的GitHub仓库，这个版本我已经将请求方式转为语义化的RESTFul API了，你可以对比一下master分支中的代码，看看语义化带来的好处。另外我引入一个本地数据库lowdb<span class="orange">[3]</span>，在你第一次启动后，创建本地数据库文件db.json，我们的增删改查不会因为重启项目而丢失了，但是在FaaS上我们却无法使用db.json文件，原因是FaaS的实例文件系统是只读的。因此FaaS版本，我们用了内存来替换文件系统。</p><p>作业初始化项目地址：<a href="https://github.com/pusongyang/todolist-backend/tree/lesson04-homework">https://github.com/pusongyang/todolist-backend/tree/lesson04-homework</a></p><p>给你的作业是，你要将这个项目部署到云上函数服务，注意FaaS的版本是index-faas.js。如果你条件允许的话，最好用自己的域名关联。我们<a href="https://time.geekbang.org/column/article/224559">[第 1 课]</a> 已经讲过FaaS官方提供的域名受限，只能下载，这个链接就是我用FaaS部署的“待办任务”：<a href="http://todo.jike-serverless.online/list">http://todo.jike-serverless.online/list</a></p><p>期待你的作业。如果今天的内容让你有所收获，也欢迎你转发给你的朋友，邀请他一起学习。</p><h2>参考资料</h2><p>[1] <a href="http://httpd.apache.org/">http://httpd.apache.org/</a><br>
[2] <a href="https://restfulapi.net/http-methods/">https://restfulapi.net/http-methods/</a><br>
[3] <a href="https://github.com/typicode/lowdb">https://github.com/typicode/lowdb</a></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/af/56/89952257.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>蒲松洋</span>
  </div>
  <div class="_2_QraFYR_0">看来很多同学都没有了解过egg.js框架，这里我需要补充一下，我Node.js的主进程和子进程的例子，其实是用了egg.js的进程模型，想跟大家解释这点知识内容。但我代码中的Express.js框架，要使用子进程要额外使用node.js的cluster模块。Node.js是单线程的，但实际它是用event loop让内核的线程去处理事件，响应时再回调handle，其实协程。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-24 11:51:37</div>
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
  <div class="_2_QraFYR_0">我比较好奇老师在实践Serverless的过程中踩了多少坑.<br><br># 我来说说我昨天在阿里云上体验Faas时的采坑记录<br><br>## 1. `fun build` 命令无法做到幂等性<br>相同的输入,经过相同的处理,得到的结果却具有不确定性.<br>症状是:<br>借助git命令,保证目录下的文件完全一致,但是执行`fun build`命令的结果却会具有不确定性.<br><br>说实话,这个功能对我没任何影响,我只是来体验FC的功能的.他们的基础服务有问题,修不修复对我来说不重要.<br>但我觉得吧,像这种大厂的用户会比较多,以后难保其他用户不会遇到这种问题.<br><br>本着方便后来人的角度,我还是花时间研究了下,如何成功在我本地大概率的复现该问题.<br>这个问题给他们反馈了,也提供了视频和日志,不确定他们是否会重视这个问题.<br><br>## 2. 命令行工具`fun`与VSCode插件行为不太一致<br>在命令行中使用fun deploy部署时,实际是优先使用的` .fun&#47;build&#47;artifacts&#47;template.yml`文件<br>而VSCode插件中默认使用的是项目根目录下的`.&#47;template.yml`<br>前者是使用`fun build`命令自动生成的.<br><br>我之前其实在命令行中已经照着官方文档把服务部署好了,但是我为了体验VSCode的插件,我又搭建了一次.<br>这次就遇到了VSCode上部署的服务无法正常工作,提示缺少依赖项.<br><br>后来在钉钉群里咨询,才知道现在需要用`fun install`安装依赖.<br>使用`fun install`安装依赖的方式,再在VSCode上一键部署的服务就是可用的了.<br><br># 最近几天体验函数计算的感悟<br>## 看上去确实很方便<br>只要你把服务调通了,几乎不用你操心剩下的运维等工作,现在还有免费额度用,个人肯定是用不完的.<br><br>## 排查问题会比较困难<br>特别是跟服务提供方相关的,对于开发者来说,完全是黑盒操作.<br>自身代码的调试,还可以借助本地调试功能来排查.<br>但是一旦提交到了函数计算平台,想查问题就非常难了.<br><br>例如:<br>1. 服务在本地可以正常运行,为什么在远程会提示缺少依赖项?<br>   其实在执行`fun deploy`的过程中,它会帮你把本地目录打包成zip文件,上次到云上,使用zip包内的文件部署函数服务.<br>   而`fun deploy`打包了哪些文件,其实我们是很难知道的.<br><br>   我上面提到的第二个坑就跟这个有关系.<br>   fun build把依赖包安装到了一个隐藏的目录,与fun install安装的目录并不相同<br>   而fun deploy会根据使用的template.yml文件来确定待压缩文件的路径.<br>   恰巧`fun deploy`又比较`智能`,会优先使用`.fun&#47;build&#47;artifacts&#47;template.yml`,若该文件不存在,才会使用`.&#47;template.yml`.<br>   这样,我用fun build安装的依赖文件, 在VSCode上通过一键部署时, 并未打包到zip文件中.<br><br>   这个也是我自己琢磨出来的.<br><br>2. 函数计算的冷启动时间如何优化?<br>   现在连具体的冷启动时间是多长,都无法确定,更无法谈如何优化了.<br>   不清楚 node.js python 的冷启动为什么只有600-800ms, 而用`Custom Runtime`打包的golang服务却要2.5s.<br><br>3. 函数的具体单次执行时间如何确定?<br>   目前只能借助云平台的日志,查看函数的平均执行时间.<br>   自己最多只能在函数执行的入口和出口加入时间统计.<br>   我就发现,我用`Custom Runtime`打包的golang服务<br>   显示的函数平均执行时间始终是100ms,而node.js和python的服务耗时时间就是动态的,只有20ms+,远没有到100ms.<br><br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你的回答都很精彩，我都放到部落里了。欢迎你加入微信群一起讨论。<br>我踩的坑肯定不少，可以在群里讨论一下。掌握了背后Serverless的原理，有助于你定位解决问题。的确目前FC的调试比较困难，所以很多人会去用fun工具。<br>不过我可以先预告一下，后面的课程，让大家自己用docker容器搭建serverless，可控性更高。就可以本地调试，build&#47;ship&#47;run。最后一课会介绍给大家一个平台，解决调试困难的问题。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-24 13:27:38</div>
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
  <div class="_2_QraFYR_0"># 对于 用完即毁型和常驻进程型 的体会<br>以前我做游戏时,很多状态都是维护在内存中.<br>这种服务如果迁移到FaaS就很困难.需要做改造,把需要持久化的数据存储到其他地方.<br><br>现在做的服务,是基于接口对外提供的服务.<br>本身不存储状态,都是根据数据库中的数据库反馈结果.<br><br>现在的服务,理论上其实可以很方便的迁移到FaaS.<br>虽然是常驻进程型的,但本身并不存储状态.<br>就是在初始化阶段,需要连各种数据库,会耽误点时间.<br><br># 对于并发访问的思考<br>昨天晚上还在想,明天要抽时间用ab压测一下函数计算提供的接口.<br>看看所谓的`并发度`到底是什么.<br>之前在文档上看过,一个函数实例可以配置允许的同时并发度,如果不够了,就会冷启动新的实例.<br><br>我之前测试的服务都是常驻进程型的,我就想搞个定时任务,每分钟请求一次,保证服务进程不会被回收.<br>其实每分钟请求一次的消耗也不大,但如果能消除冷启动,绝对是划算的.<br><br>但后来一想,这个只针对单实例有效,如果想保证同时有N个实例,如何才能保证定时请求可以将N个实例都访问到呢?<br>这个需要用再实践一下.<br><br># 课后作业<br>由于前几天已经配置了阿里云的fun本地环境,自己也有备案了域名,所以实践老师的作业只需要简单的几部.<br>[我在老师的专栏开始之前完全未接触过node.js 现在也只是跟着老师部署了几个简单的node.js服务]<br><br>1. 克隆代码<br>	git clone https:&#47;&#47;github.com&#47;pusongyang&#47;todolist-backend<br>2. 拷贝文件<br>	cp index-faas.js index.js<br>3. 安装依赖<br>	npm install<br>4. 创建template.yml文件<br>```<br>ROSTemplateFormatVersion: &#39;2015-09-01&#39;<br>Transform: &#39;Aliyun::Serverless-2018-04-03&#39;<br>Resources:<br>  todolist-backend:<br>    Type: &#39;Aliyun::Serverless::Service&#39;<br>    Properties:<br>      Description: &#39;helloworld&#39;<br>    todolist-backend:<br>      Type: &#39;Aliyun::Serverless::Function&#39;<br>      Properties:<br>        Handler: index.handler<br>        Runtime: nodejs10<br>        CodeUri: &#39;.&#47;&#39;<br>      Events:<br>        httpTrigger:<br>          Type: HTTP<br>          Properties:<br>            AuthType: ANONYMOUS<br>            Methods:<br>              - GET<br>              - POST<br>```<br>5. 部署服务<br>	fun deploy -y<br>6. 绑定自定义域名<br>	需要把路径&#47;*都绑定到该服务上<br>7. 验证效果<br>	可以重现老师的服务: http:&#47;&#47;todo.jike-serverless.online&#47;list<br><br># 扩展思考<br>如果只是测试,可以配合NAS服务来持久化部分数据.<br>我之前写demo时,用这个功能,简单的记录我函数计算服务的访问日志和时间.<br><br>仅仅只是测试,并未考虑到多实例并发访问的问题.<br><br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 欢迎将你的做法也提MR到github上面来。<br>https:&#47;&#47;github.com&#47;pusongyang&#47;todolist-backend&#47;</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-24 12:18:37</div>
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
  <div class="_2_QraFYR_0">用 fun 部署完成只会，访问 报错啊<br>The CA process either cannot be started or exited:ContainerStartDuration:25516062312， 又没用用 https 咋还有CA证书了？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 用fun先部署master分支体验一下吧，这节课的分支lesson04-homework，我稍后有时间返回调试一下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-24 16:47:39</div>
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
  <div class="_2_QraFYR_0">当 Nodejs 处理并发请求的并不会自动创建子进程，利多核CPU的的特性。 Nodejs一直都是单线程的<br>A single instance of Node.js runs in a single thread. To take advantage of multi-core systems, the user will sometimes want to launch a cluster of Node.js processes to handle the load<br>如果想利用多核 就要使用 cluster 模块<br><br>还有就是 并发 是在一个 CPU 核心上交替执行， 在多个 CPU 核心上执行这叫做并行</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这里涉及到Node.js底层的原理：Node.js是单线程的，但实际它是用event loop让内核的线程去处理事件，响应时再回调handle。<br>为了简化模型，我将它描述为子线程，这里其实协程。<br>如果自己部署可以VM用PM2启动cluster。或者egg.js框架。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-24 08:00:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/70/67/0c1359c2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>qinsi</span>
  </div>
  <div class="_2_QraFYR_0">看了下代码似乎并没有开node多进程...如果是挂在nginx上面的话nginx确实会创建worker进程，但也不是每次请求来都会创建新进程...</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这里涉及到Node.js底层的原理：Node.js是单线程的，但实际它是用event loop让内核的线程去处理事件，响应时再回调handle。<br>为了简化模型，我将它描述为子线程，这里其实协程。<br>如果自己部署可以VM用PM2启动cluster。或者egg.js框架。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-24 07:35:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/24/96/3f/7c8a8676.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>周科</span>
  </div>
  <div class="_2_QraFYR_0">Node.js处理请求，并不是来一个请求就创建一个子进程去处理啊，应该说是来一个请求就创建一个context。4核机器跑egg.js应用，默认也是4个子进程吧，而且Node.js进程一直常驻内存的，不会根据请求动态创建或回收。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个简化模型只是为了方便理解，worker进程不活跃时对系统来说除了占些内存没有其它影响，在请求request过来时才会占用cup和更多内存资源。监听函数处理完后GC再释放这些资源。本课并不是讲node.js或egg.js因此不想讲的太深入这部分。egg.js也是用的node.js的cluster，worker分配其实也不是通过master。BTW：我也是egg.js的官方维护者。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-04 17:35:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/75/9b/611e74ab.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>技术修行者</span>
  </div>
  <div class="_2_QraFYR_0">问一个问题，后端存储变为BaaS后，传统的数据库事务怎么处理呀？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 微服务里面有全局事物，例如阿里云的GTS，不过目前好像只支持java。可以关注后期发展。<br>如果是云服务商提供的BaaS服务和传统数据库用法是一样的，只是链接方式有所不同。<br>自己构建的RESTful API，则可以自己API实现中处理事物逻辑。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-16 13:13:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/36/93/7dc76b32.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>PP-CIPDS-GRC</span>
  </div>
  <div class="_2_QraFYR_0">所以其实是每个调用阶段的管控颗粒度更细了，stateful 服务调用采用 BaaS 服务和数据持久化，费用单独算。前端的数据编排接口采用 FaaS 按次数调用，可以这么理解？静态的还是静态，动态的保持动态。整个调用 Pipeline 的各个动静部分解耦。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以这样理解。<br>pipeline现在通常是说发布管道。你这里应该说的是调用链接？<br>后面课程讲到service mesh就是负责东西流量通信的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-06 17:07:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/fUDCLLob6DFS8kZcMUfxOc4qQHeQfW4rIMK5Ty2u2AqLemcdhVRw7byx85HrVicSvy5AiabE0YGMj5gVt8ibgrusA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>NO.9</span>
  </div>
  <div class="_2_QraFYR_0">今天终于有空做作业。我是aws Lambda+api Gateway 。说实话 我没看懂应该部署哪个文件。我先尝试用aws的示例代码 invoke了一次 成功了？但是把index.js的内容贴进去(是的，我只粘贴了这一个文件内容) 显示失败。。。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我的例子是基于阿里云fc部署的，因为有runtime的限制。在aws中部署的话，要根据aws lambda的runtime提供的能力去实现</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-19 22:01:54</div>
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
  <div class="_2_QraFYR_0">BaaS 的数据库， 可以像FaaS 一样按使用量收费吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 最近腾讯云推出了BaaS DB，postgreSQL，不过收费模式还是传统的。db很难做到用时拉起，只能看未来技术发展了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-08 21:36:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/57/27/805786be.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>笨笨</span>
  </div>
  <div class="_2_QraFYR_0">个人感悟：GraphQL就是BaaS的一种实践。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: GraphQL更像是一种API描述协议。GraphQL我们通常对应的是RESTFul API。<br>BaaS关注的是后端能力微服务化（免运维，提供前端与鉴权。。。）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-29 13:00:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q3auHgzwzM6hsCtODfwaIPW9T9qzxNAhhkdn4ImGHeZicA1UyhCOXDf8MtJXw4QnTFQgUia4BPTZdD2zpgV1qTfQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_f7f72f</span>
  </div>
  <div class="_2_QraFYR_0">对于传统数据库那块，还有一点疑问。文中提到传统数据库同样的操作会比 HTTP 快，同时又提到FaaS直连数据库增加了冷启动时间。那么改造后的BaaS又是如何避免这个问题的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: BaaS可以自己维护DB连接池，而且可以限流降级处理。<br>现在很多云服务商提供的BaaS，会利用FaaS runtime去建立连接。FaaS未来支持自定义runtime，那么利用传统数据库还是比较简单的。不过，并发量上升，数据库很快又会成为你改造的瓶颈。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-14 09:19:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/fa/63/bfbee4ac.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>深黑色</span>
  </div>
  <div class="_2_QraFYR_0">老师，请教一下在阿里云上貌似没有看见用完即毁型和常驻型的区分选项呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这2种实际是进程运行模型。FC本身没有限制我们如何使用它，但是不理解FC的能力和边界，很容易出bug。<br>FC的用法参照进程运行模型，你可以让函数执行完就销毁，一个FaaS部署一个简单函数。也可以让一个FaaS函数部署多个函数，自己不释放，等待被云服务商调度回收。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-03 22:59:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/6c/69/d5a28079.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Bora.Don</span>
  </div>
  <div class="_2_QraFYR_0">按照这种思路，faas也不需要了，直接前端代码里的js调用baas，一把梭哈，全部搞定。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: js直接调用BaaS不太安全，不然几年前的mBaaS就应该火了。FaaS可以做BFF层，处理数据编排。还有部分事件触发的场景。毕竟FaaS还是便宜。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-24 23:22:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/75/a8/dfe4cade.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>电光火石</span>
  </div>
  <div class="_2_QraFYR_0">在传统的服务话架构中，BFF后面的服务是有蛮重的业务逻辑在，BFF本身会做的很薄。在servless中，BASS是否只是对数据层的接口包装？还是也会包含逻辑？谢谢了！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: BaaS是后端即服务，将数据库或者一些业务逻辑包裹成API给FaaS编排使用。<br>但BFF只是FaaS的一个使用场景，有部分BaaS也可以用FaaS去实现的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-24 11:07:21</div>
  </div>
</div>
</div>
</li>
</ul>