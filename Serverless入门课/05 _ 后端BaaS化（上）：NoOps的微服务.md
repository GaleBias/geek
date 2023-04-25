<audio title="05 _ 后端BaaS化（上）：NoOps的微服务" src="https://static001.geekbang.org/resource/audio/d6/11/d63a54ce8d097aca2b5ff55a41dc1d11.mp3" controls="controls"></audio> 
<p>你好，我是秦粤。现在我们知道了在网络拓扑图中，只有Stateless节点才能自由扩缩容，而Stateful节点因为保存了重要数据，我们要谨慎对待，因此很难做扩缩容。</p><p>FaaS连接并访问传统数据库会增加额外的开销，我们可以采用数据编排的思想，将数据库操作转为RESTful API。顺着这个思路，我引出了后端应用的BaaS化，一句话总结，后端应用BaaS化就是将后端应用转换成NoOps的数据接口。那怎么理解这句话呢？后端应用BaaS化，究竟应该怎么做？接下来的几节课，我们会展开来讲。</p><p>我们先回忆一下上节课的“待办任务”Web应用，这个项目前端是单页应用，中间用了FaaS做SFF数据网关，后端数据接口还要BaaS化。这个案例会贯穿我们的课程，所以你一定要动手run一下。为了让你对我们的项目有个宏观上的认识，我还是先交付你一张大图。</p><p><img src="https://static001.geekbang.org/resource/image/ba/c9/bab7e22b588d69cbe0197d36696411c9.jpg?wh=1670*820" alt="" title="“待办任务”Web应用架构图"></p><p>这个架构的优势是什么呢？我们将这个图变个形，你就更容易理解了。</p><p><img src="https://static001.geekbang.org/resource/image/66/8f/66aeb01686c94478b2847be5bb2a398f.jpg?wh=2462*828" alt="" title="架构变形示意图"></p><p>咱从左往右看上面这张图。用户从浏览器打开我们网站时，前端应用响应返回index.html；然后浏览器去CDN下载我们的静态资源，完成页面静态资源的加载；与此同时，浏览器也向前端应用发起数据请求；前端应用经过安全签名后，再将数据请求发送给SFF；SFF再根据数据请求，调用后端接口或服务，最终将结果编排后返回。</p><!-- [[[read_end]]] --><p>从图里你可以看到，除了数据库是Stateful的，其它节点都已经变成了Stateless。如果你公司业务量不大的话，这个架构其实已经足够了。就像传统的MVC架构一样，单点数据库，承载基本的并发量不是问题，而且数据也可以通过主从结构和客户端读写分离的方式来优化。</p><p>但MVC架构最大的问题就是累积，当一个MVC架构的应用，在经历长期迭代和运营后，数据库一定会变得臃肿，极大降低数据库的读写性能。而且在高并发达到一定量级，Stateful的数据库还是会成为瓶颈。那我们可以将自己的数据库也变成BaaS吗？</p><p>要解决数据库的问题，也可以选择我上节课和你说的云服务商提供的BaaS服务，比如DynamoDB。但云服务商BaaS服务究竟是怎么做到的？如果BaaS服务能力不全，不够满足我们的需要时怎么办？今天我就先带你看看传统的MVC应用中的数据库怎么改造成BaaS。</p><p>当然，BaaS化的过程有些复杂，这也正是我们后面需要用几节课才能跟你解释清楚的核心知识点。正如我们本节课的标题：后端应用BaaS化，就是NoOps的微服务。在我看来后端应用BaaS化，跟微服务高度重合，微服务几乎涵盖了我们BaaS化要做的所有内容。</p><p>所以我们先来学习一下微服务是什么。</p><h2>微服务的概念</h2><p>微服务的概念对很多做后端同学来说并不陌生，尤其是做Java的同学，因为早些年Java就提出SOA面向服务架构。微服务算是SOA的一个子集，2014年由ThoughtWorks的Martin Fowler提出。微服务设计之初是为了拆解巨石应用，巨石应用就是指那些生命周期较长的，累计了大量业务高度耦合和冗余代码的企业应用。</p><p>跟Serverless的概念还在发展中不同，微服务的概念在这么多年的发展中已经有了明确的定义了。下面是AWS官方的解释：</p><blockquote>
<p>微服务是一种开发软件的架构和组织方法，其中软件由通过明确定义的 API 进行通信的小型独立服务组成，这些服务由各个小型独立团队负责。微服务架构使应用程序更易于扩展和更快地开发，从而加速创新并缩短新功能的上市时间。</p>
</blockquote><p>那在我看来，微服务就是先拆后合，它将一个复杂的大型应用拆解成职责单一的小功能模块，各模块之间的数据模型相互独立，模块采用API接口暴露自己对外提供服务，然后再通过这些接口组合出原先的大型应用。拆解的好处是，小模块便于维护，可以快速迭代，跨应用复用。</p><p>我们的Serverless专栏并不打算给你详细地讲解微服务，但是希望你能一定程度上了解微服务。FaaS和微服务架构的诞生几乎是在同一时期，它俩的很多理念都是来自12要素（Twelve-Factor App）<span class="orange">[1]</span>，所以微服务概念和FaaS概念高度相似，也有不少公司用FaaS实现微服务架构；不同的是，微服务的领军公司ThoughtWorks和NetFlix到处宣扬他们的微服务架构带来的好处，而且他们提出了一整套方法论，包括微服务架构如何设计，如何拆解微服务，尤其是数据库如何设计等等。</p><p>我们后端应用BaaS化，首先要将复杂的业务逻辑拆开，拆成职责单一的微服务。<strong>因为职责单一，所以服务端运维的成本会更低。</strong>而且拆分就像治理洪水时的分流一样，它能减轻每个微服务上承受的压力。</p><p>我在Serverless的专栏里向你介绍微服务，主要就是想引入微服务拆解业务的分流思想和微服务对数据库的解耦方式。</p><h2>微服务10要素</h2><p>谈起微服务，很多人都要说12要素<span class="orange">[2]</span>，认为微服务也应该满足12要素。12要素当初是为了SaaS设计的，用来提升Web应用的适用性和可移植性。我在做微服务初期也学习过12要素，但我发现这个只能提供理论认识。</p><p>所以在2018年GIAC的大会上，我将我们团队在微服务架构落地中的经验总结为<strong>微服务的10要素：API、服务调用、服务发现；日志、链路追踪；容灾性、监控、扩缩容；发布管道；鉴权。</strong></p><p>我们回想一下<a href="https://time.geekbang.org/column/article/224559">[第1课]</a> 中小服的工作职责：</p><ol>
<li>无需用户关心服务端的事情（容错、容灾、安全验证、自动扩缩容、日志调试等等)。</li>
<li>按使用量（调用次数、时长等）付费，低费用和高性能并行，大多数场景下节省开支。</li>
<li>快速迭代&amp;试错能力（多版本控制，灰度，CI&amp;CD等等）。</li>
</ol><p>你有没有发现跟微服务的要素有大量的重合。API就是RESTful的HTTP数据接口；服务调用你可以理解为就是HTTP请求；服务发现你可以理解为我们只能用域名调用我们的HTTP请求，不能用IP；日志、容灾、监控都不难理解；链路追踪，是微服务重要的一环，因为相对传统MVC架构，我们一个请求在后端的调用链增长了，为了快速定位问题，我们都需要打印整个调用链路的异常栈；发布管道和鉴权，和我们的FaaS也有很重要的关联，我将放在下一课讲解。</p><p><img src="https://static001.geekbang.org/resource/image/45/f1/4558c4088c38623cf366e7d686654ff1.png?wh=952*628" alt="" title="木桶效应示意图"></p><p>接下来，我不准备按照微服务的10要素一一操作，但我希望可以通过Serverless帮你建立知识体系的索引，所以关联的技术栈我都尽量点出来，你可以自己进一步了解学习。</p><p>我们再次拿出我们的创业项目“待办任务”Web网站，这次我将带你一起看看微服务如何让数据库解耦。</p><p>我们先分析一下“待办任务”Web服务。上一讲末尾我的作业其实已经为你准备好了，我们一起看看index.js文件，这是一个典型的MVC架构的应用。View层就是index.html和静态资源文件；Model层，我们引入lowdb用一个db.json文件代替数据库；Control层，就是app.METHOD，处理/api/*的数据逻辑。微服务是解决后端应用的，所以我们只需要关注Control层。</p><p><img src="https://static001.geekbang.org/resource/image/25/a4/25981be7fb564da00a736bfe93f101a4.jpg?wh=2302*686" alt="" title="index.js文件"></p><p>API的数据处理，主要有两个，一个是/api/rule处理待办任务的，一个是/api/user负责用户登录态，而且我们“待办任务”主要的数据模型也就是待办任务和用户这两个。那我们可以将后端拆分出两个微服务：待办任务微服务和用户微服务。这里我要强调下，我只是为了向你演示微服务才这样做，在实际业务中，这么简单的功能，没必要过早地拆分。</p><p><img src="https://static001.geekbang.org/resource/image/3a/ab/3abe89510aa6897b0b1f50e239c91dab.jpg?wh=2330*558" alt="" title="微服务拆解演示"></p><p>初步拆解，我们将index.js中的数据库移走了，而且拆分后各微服务的数据库比原来混杂在一起简单且容易维护多了。user.js负责用户相关业务逻辑，只维护用户信息的数据库，并且暴露RESTful的HTTP方法；rule.js负责待办任务的增删改查，只维护待办任务的数据库，并且暴露RESTful的HTTP方法；index.js只需要负责将请求HTTP返回给数据编排。</p><p>HTTP协议要满足我们的服务调用与发现，发布的user.js和rule.js必须使用域名，例如api.jike-serverless.online/user和api.jike-serverless.online/rule。这样新扩容的机器IP，只需要注册到这个域名下就可以被服务发现IP并调用了。</p><p>我们现在拆分后只是将数据库从index.js移出，分给到了user.js和rule.js，但两个节点到目前为止数据库仍然是Stateful的。我们如何再让这两个数据库节点变成Stateless呢？你可以按一下暂停键思考一下。</p><p>我们先想想，对于rule.js这个微服务来说，我要再扩容一个新的实例，而且让两边的实例保持一致，那么我们是否只需要让新的数据库和旧的数据库保持同步更新就可以了呢？</p><h2>解耦数据库</h2><p>没错，其实要解决这个问题的核心就是让数据启动时，可以更新到最新的数据库。在其中一个数据库更新时，通知另外一个数据库更新。</p><p>这里就要引出一个关键的Stateful对象：消息队列<span class="orange">[3]</span>。它是一个稳定的绝对值得信赖的Stateful节点，而且对你的应用来说消息队列是全局唯一的。解耦数据库的思路就是，通过消息队列解决数据库和副本之间的同步问题，有了消息队列，剩下的就简单了。</p><p>我们每个微服务中的数据库，例如MySQL，写的操作都会产生binary log，通过额外的进程将binary log变更同步到消息队列，并监听消息队列将binary log更新在本地执行，修改MySQL。比较著名的解决方案就是领英开源的Kafka，现在云服务商基本也都会提供Kafka服务。当然数据库和副本之间会有短暂的同步延迟问题，但问题其实也不大，因为我们通常对数据库进行写操作时也会锁表。</p><p>如果你要细究这个问题，那就会碰到分布式架构无法同时满足的CAP<span class="orange">[4]</span> 课题。我们在此也不深入展开CAP话题了，目前异步同步的时间差在很多场景下我觉得是可以接受的。</p><p><img src="https://static001.geekbang.org/resource/image/96/36/966b47dfea82a3d841e45cd260270636.jpg?wh=2194*904" alt="" title="微服务拆解演示"></p><p>拿我们自己的例子来说也很简单，我们用消息队列RocketMQ，写操作我们都写入RocketMQ，rule.js进程中我们监听RocketMQ，一旦有rule写入的消息，我们就更新我们的lowdb数据。围绕消息队列建立的同步机制，让每个微服务的数据库和它的扩容副本之间自动同步了。对于微服务来说，它本身的数据库还是Stateful的，但在微服务外部看来，这个微服务是Stateless的。如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/7a/3a/7ab3bebbffabd18282d54e17093d0c3a.jpg?wh=1148*627" alt="" title="同步机制"></p><p>跟传统的主从数据库方式不同的是，我们每个微服务的单点实例都是独享一个数据库的，rule这个微服务单点可以对外提供所有待办任务的RESTful API接口。</p><p>你要知道消息队列，例如RocketMQ的可靠性是99.99999999%，而我们常用的虚拟机ECS的可靠性也只不过99.95%。在高并发架构的设计中，我们通常会要求Stateful节点一定要稳定，而且越少越好，所以消息队列，对于微服务和FaaS来说，都是可以作为支撑业务的核心。我们依赖消息队列，会让整个架构更稳定。</p><h2>总结</h2><p>我们为了避免在FaaS中直接操作数据库，而将数据库操作变成BaaS服务。为了理解BaaS化具体的工作，我们引入了微服务的概念。微服务先将业务拆解成单一职责和功能的小模块便于独立运维，再将小模块组装成复杂的大型应用程序。这一点上跟我们要做的后端应用BaaS化相似度非常高。所以参考微服务，我们先将MVC架构的后端解成了两个微服务，再用微服务解耦数据库的操作，先将数据库拆解成职责单一的小数据库，再用消息队列同步数据库和副本，从而将数据变成了Stateless。</p><p>现在我们再来回顾一下这节课的重要知识点。</p><ol>
<li>微服务的概念：它就是将一个复杂的大型应用拆解成职责单一的小功能模块，各模块之间数据模型相互独立，模块采用API接口暴露自己对外提供的服务，然后再通过这些API接口组合成大型应用。这个小功能模块，就是微服务。</li>
<li>微服务10要素：API、服务调用、服务发现；日志、链路追踪；容灾性、监控、扩缩容；发布管道；鉴权。这跟我们要做的BaaS化高度重合，我们可以借助微服务来实现我们的BaaS化。</li>
<li>解耦数据库：通过额外进程让数据库与副本直接通过消息队列进行同步，所以对于微服务应用来说，只需要关注自身独享的数据库就可以了。微服务通过数据库解耦，将后端应用变成Stateless的了，但对后端应用本身而言，数据库还是Stateful的。</li>
</ol><h2>作业</h2><p>由于Kafka云服务比较贵，所以这节课的作业我们采用表格存储<span class="orange">[5]</span>来模拟Kafka，以实现DB的同步功能。<strong>你可以尝试自己部署一下rule-faas到一个新的域名。</strong>下面是具体的代码地址：</p><ul>
<li>仓库地址：<a href="https://github.com/pusongyang/todolist-backend/tree/lesson05">https://github.com/pusongyang/todolist-backend/tree/lesson05</a></li>
<li>前端地址：<a href="https://github.com/pusongyang/todolist-frontend/tree/lesson05">https://github.com/pusongyang/todolist-frontend/tree/lesson05</a></li>
<li>实现后效果：<a href="http://lesson5.jike-serverless.online/list">http://lesson5.jike-serverless.online/list</a></li>
</ul><p>你可以根据自己的域名，替换一下前端文件中的这个地址：<a href="http://faas.jike-serverless.online/api/rule">http://faas.jike-serverless.online/api/rule</a>，或者用我的前端代码<a href="https://github.com/pusongyang/todolist-frontend/tree/lesson05">lesson5</a>分支修改/src/pages/ListTableList/service.ts中的地址，自己 <code>npm run build</code> 一下，替换掉项目public目录，压缩上传到函数服务目录。</p><p>这次的部署会有些复杂，我们要部署两个FaaS服务，一个是rule-faas.js，用来处理/api/rule；另一个是index-faas.js，用来处理我们的静态资源请求。原理图如下所示：</p><p><img src="https://static001.geekbang.org/resource/image/98/b0/987db6d15137ce820b56fcf97e91e8b0.jpg?wh=2132*914" alt="" title="部署示意图"></p><p>另外.aliyunConfig文件格式如下，你需要填入自己的配置情况，切记这个文件不可以上传到Git仓库：</p><pre><code>//.aliyunConfig文件，保存秘钥，切记不可以上传Git
const endpoint = &quot;https://rule.cn-shanghai.ots.aliyuncs.com&quot;;
// AccessKey 阿里云身份验证，在阿里云服务器管理控制台创建
const accessKeyId = &quot;AccessKey&quot;;
// SecretKey 阿里云身份验证，在阿里云服务器管理控制台创建
const accessKeySecret = &quot;SecretKey&quot;;
// 在数据链表中查看
const instancename = &quot;rule&quot;;
const tableName = 'todos';
const primaryKey = [{ 'key': 'list' } ];


module.exports = {endpoint, accessKeyId, accessKeySecret, instancename, tableName, primaryKey};

</code></pre><p>期待你的作业和思考，如果今天的内容让你有所收获，也欢迎你把文章分享给身边的朋友，邀请他加入学习。</p><h2>参考资料</h2><p>[1] <a href="https://en.wikipedia.org/wiki/Twelve-Factor_App_methodology">https://en.wikipedia.org/wiki/Twelve-Factor_App_methodology</a></p><p>[2] <a href="https://12factor.net/zh_cn/">https://12factor.net/zh_cn/</a></p><p>[3] <a href="https://help.aliyun.com/document_detail/29532.html?spm=5176.11065259.1996646101.searchclickresult.6936139bcS8BlU">https://help.aliyun.com/document_detail/29532.html?spm=5176.11065259.1996646101.searchclickresult.6936139bcS8BlU</a></p><p>[4] <a href="https://www.ruanyifeng.com/blog/2018/07/cap.html">https://www.ruanyifeng.com/blog/2018/07/cap.html</a></p><p>[5] <a href="https://ots.console.aliyun.com/index#/list/cn-shanghai">https://ots.console.aliyun.com/</a></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/70/67/0c1359c2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>qinsi</span>
  </div>
  <div class="_2_QraFYR_0">请问数据库解耦时副本数据库允许数据写入吗？如果允许写入的话会不会因为写入的数据冲突导致数据不一致呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 数据副本只是为了区分的叫法，当然允许写入了。Binary log有一定的容错性，要考虑事务逻辑。但如果是银行或者支付的场景，要解决强一致性。可以直接写消息队列，再去消费数据。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-28 18:13:01</div>
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
  <div class="_2_QraFYR_0">老师好，在后台服务baas化的过程中，关于数据库跟以前微服务的方式有些不一样，微服务的各个节点是共享数据库的，但是baas化中，每个节点有自己的数据库，而且数据库的内容是一样的。这样是否带来2个问题：<br>1. 在数据量很大的情况下，每个节点都有一样的数据库，是否会使得数据存储量会翻几倍，成本会很高<br>2. 在扩容的情况下，如果数据量很大，新节点需要复制的数据也很多，启动时间会非常的长<br>3. 有多少个应用，就有多少个数据库实例。一般情况下，我们应用的数量会远高于数据库机器的数量。高并发场景下，会不会导致网络占用很高，性能整体上升<br>还是说这种方案本身就不适合大并发或者数据量很高的场景，谢谢了！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 微服务标准的做法是一个微服务实例独享一个数据库实例的。当然你业务量不大，可以多个微服务实例共享一个数据库实例。<br>微服务架构在docker流行后才火，就是因为docker可以让计算资源进一步碎片化，所以成本并不会很高。<br>数据实例的启动时，通常是先从现有数据库副本创建，再根据消息队列同步后续数据。启动实例过程是秒级的。<br>微服务架构，本身就是通过拆解将复杂的业务问题变成一个个小的职责单一的小服务。运维的复杂度会上升，但是整体性能并不会下降多少。<br>你可以想想，现在的京东，淘宝这样的网站，背后都是微服务架构的。如果你用单体应用开发这么庞大的一个应用，这么多团队和人员参与，怎么去运维和开发。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-27 12:30:29</div>
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
  <div class="_2_QraFYR_0">哈哈，原来老师也觉得云厂商提供的kafka云服务贵。<br>我对这个深有体会！<br><br>我看领导买的一个阿里云kafka云服务，要4K+&#47;月，配置也就一般吧，可能监控做的比较好吧。<br>再想想我买的竞价付费实例，约64元&#47;月，4cpu8g配置，还不是突发性能的。当开发环境的k8s节点，也不怕丢数据，哈哈。<br><br>想一想，还是有钱真好。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 用CaaS服务，自己搭建一套Kafka还便宜一些。现在程序员可以调度的机器资源还是很充沛的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-27 08:10:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/47/ae/0a5f7a56.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>此方彼方Francis</span>
  </div>
  <div class="_2_QraFYR_0">微服务标准的做法是一个微服务实例独享一个数据库实例的。<br><br>这个说法有些夸张了吧？老师有生产上的实践案例吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我这里提的微服务实例独享一个数据库实例是标准做法，这样做可以让微服务实例+数据库形成的整体，变成stateless。这个叫做：数据库的可弃性。<br>实际场景比较灵活，你可以一个微服务多个实例，使用一个数据库。不过你需要额外关注：这个数据库不会成为你的瓶颈。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-03 16:40:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/09/4f/84ef0db0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>liubin</span>
  </div>
  <div class="_2_QraFYR_0">确实，有点扯了，每个应用副本一个db，谁来负责业务数据同步，不是kafka能解决的吧。而且事物也没法搞。这工作量比单db要大100倍</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 首先kafka有专门的各种db同步解决方案，你可以了解一下，这门课不展开了。其次单个db对于并发量高的应用来说通常的解决方案就是分库分表，看你如何去组织而已。如果你的场景db不是瓶颈，你100个应用用一个db实例都可以。跟事务也没关系事务是数据库本身就应该保证的，同步数据库用的是数据库日志。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-16 16:45:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c4/e4/81ee2d8f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Wisdom</span>
  </div>
  <div class="_2_QraFYR_0">一个微服务，n个实例，一个实例对应一个数据库，这种方案，有点太浪费，而且数据的一致性方面还可能会有问题，不赞同这样，京乐、淘宝也没有这样搞，感觉这里有点误导</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 一个微服务N个实例，只用一个数据库也可以，甚至几个微服务共享一个数据库也可以。我这里举例子是在数据库是瓶颈的时候，将数据库解耦。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-14 15:35:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/f9/ee/5294ebf4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>youngwang1228</span>
  </div>
  <div class="_2_QraFYR_0">我记得是微服务说的是： 每个服务有自己单独的数据库。<br>假如一个服务有5个实例，每个实例独享一个数据库的话，会不会有点奢侈。<br>例如，一个数据库的数据量是10G，那5个库就要50G，存储成本直线上升。<br>而且，新加实例的话，复制一个10G数据量的库实例，时间不短吧。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 微服务，推荐一个实例对应一个DB。是因为通常DB是瓶颈，如果微服务实例是瓶颈也可以1:n，不过有些协调成本。<br>数据库通常并不是那么弹性的，需要随时从0创建。通常微服务的数据库还也是需要定期精心维护的。<br>微服务的拆解是为了解耦数据库，当数据库积累到一定量后也要看看能否进一步拆解。<br>如果你的核心业务可以做到这么庞大，公司体量也会很庞大，那就需要更加高级的架构解决的问题了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-14 11:24:38</div>
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
  <div class="_2_QraFYR_0">BaaS 化只能基于 HTTP（RESTful）进行吗？可不可以基于 TCP，比如走 RPC 的方式？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: BaaS可以基于RPC的。<br>不过http目前各种service mesh支持度比较广泛，可以做无侵入式网络接管。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-06 19:51:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/b1/81/13f23d1e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>方勇(gopher)</span>
  </div>
  <div class="_2_QraFYR_0">请问Rocketmq  10个9是怎么得出来的，是mttf  还是mttr  是一年周期么</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 阿里云官方提供的数据：服务可用性 99.95%，Region 化、多可用区、分布式集群化部署，确保服务高可用，即便整个机房不可用仍可正常提供消息服务；数据可靠性 99.99999999%，同步双写、超三副本数据冗余与快速切换技术确保数据可靠；<br>数据可靠性，是数据不会丢失。实际使用的服务可用性是99.95%，尤其因为网络等等问题无法获取数据。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-22 10:51:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/ZkcKFzQ3qaIWpHjfOib9yU1LicthmiawfmkNhiakDdTUbTtCcworSZsxpiaib4sygOoc32duoUdQcI3Y1EmgcbicTzAyA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_a5c054</span>
  </div>
  <div class="_2_QraFYR_0">TypeError: Cannot read property &#39;0&#39; of undefined<br>    at Response.&lt;anonymous&gt; (&#47;home&#47;zyq&#47;Downloads&#47;todolist-backend-lesson08&#47;index.js:182:44)<br>这个是为什么呀</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 读取数据库失败了，data.row.attributes[0].columnValue。请看一下08课的答复里面内容。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-06 17:12:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/48/15/8db238ac.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>神仙朱</span>
  </div>
  <div class="_2_QraFYR_0">疑问：通过额外进程让数据库与副本直接通过消息队列进行同步<br><br>为什么要让数据库产生一个副本，然后进行同步。这样不是两份相同的数据了吗，这样跟数据库瓶颈有什么关系，因为这样好像没有分担数据库的压力？只是copy 了一份？原数据库的数据多了，有了瓶颈，另一个副本因为数据一样，不也会产生瓶颈的吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这里不是数据库副本，而是2个数据库实例。<br>一个应用实例独享一个数据库实例，相当于将复杂应用垂直切分成了数个小型应用。保持数据单一性，简化模型。这是微服务的思想。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-05 13:53:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/f9/e3/2529c7dd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>吴科🍀</span>
  </div>
  <div class="_2_QraFYR_0">老师，关于用mq实现数据库同步的问题，如果要求数据强一直都业务场景很难实现啊，比如秒杀，通过mq同步不及时，用户会查到不同的库存，造成多卖的情况呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你说的这种数据强一致性的场景，要直接用消息队列去维护一致性问题，生产&#47;消费模型，还要用到事务完成和回滚。<br>其实有些超纲，架构设计要考虑自己业务发展，如果你们发展到这个程度，架构配套肯定也要跟进。我举例的业务场景下，没有那么强的一致性。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-28 09:03:22</div>
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
  <div class="_2_QraFYR_0">感觉现在的学习门槛越来越高了.😁<br><br>0.要有自己的域名,最好还是备案过的.<br>1.需要会排查依赖包缺失的问题.<br>2.需要会开通简单的云服务.<br>3.需要会配置访问秘钥及授权.<br>4.还要会配置函数计算的日志,还需要会看.<br><br>-----------------<br>有个疑问咨询下老师:<br>如何用一套代码中的不同入口文件部署多个函数计算服务?<br><br>我现在是把代码仓库整个拷贝了一份,<br>一个是`cp index-faas.js index.js`,<br>另一个`cp rule-faas.js index.js`.<br>然后再微调下`template.yml`.<br><br>由于默认的运行时环境指定了入口文件`index.js`,我只能用这个笨办法.<br>不知道目前有没有什么简洁的方式.<br>[我目前只会使用fun命令行部署服务]<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 目前阿里云有个云开发平台，正在内测。最后一课。我会放出来给大家介绍。<br>门槛的确会越来越高，带大家走进到serverless的深水区，层层深入。然后，再回头看serverless，你会有不一样的视野。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-27 14:43:48</div>
  </div>
</div>
</div>
</li>
</ul>