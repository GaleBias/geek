<audio title="06 _ 后端BaaS化（中）：业务逻辑的拆与合" src="https://static001.geekbang.org/resource/audio/33/90/33ccbfc021b2b37321afe8e1cb955390.mp3" controls="controls"></audio> 
<p>你好，我是秦粤。上一课中，我们学习了后端BaaS化的重要模块：微服务。现在我们知道微服务的核心理念就是先拆后合，拆解功能是为了提升我们功能的利用率。同步我们也了解了实现微服务的10要素，这10要素要真讲起来够单独开一门课的。如果你不熟悉，我向你推荐杨波老师的<a href="https://time.geekbang.org/course/intro/100003901">《微服务架构核心20讲》</a>课程。</p><p>BaaS化的核心其实就是把我们的后端应用封装成RESTful API，然后对外提供服务，而为了后端应用更容易维护，我们需要将后端应用拆解成免运维的微服务。这个逻辑你要理解，这也是为什么我要花这么多篇幅给你谈微服务的关键原因。</p><p>上节课我们将“待办任务”Web服务的后端，拆解为用户微服务和待办任务微服务。但为什么要这样拆？是凭感觉，还是有具体的方法论？这里你可以停下来想想。</p><p>微服务的拆解和合并，都有一个度需要把握，因为我们在一拆一合之间，都是有成本产生的。如果我们拆解得太细，就必然会导致我们的调用链路增长。调用链路变长，首先影响的就是网络延迟，这个好理解，毕竟你路远了，可能“堵车”的地方也会变多；其次是运维成本的增加，调用链路越长，整个链条就越脆弱，因为其中一环出现问题，都会导致整个调用链条访问失败，而且我们排查问题也变得更加困难。</p><!-- [[[read_end]]] --><p>反过来看，如果我们拆解得太粗，调用链路倒是短了，但是这个微服务的复用性就差了，更别提因为高耦合带来的复杂且冗余的数据库表结构，让我们后续难以维护。我画了个图，你感受下。</p><p><img src="https://static001.geekbang.org/resource/image/e3/81/e3809702efd712b18d28465b2512e681.png?wh=1870*880" alt="" title="拆解程度示意图"></p><h2>拆之，领域驱动设计</h2><p>那我们要合理地拆解微服务，应该怎么拆解呢？上节课其实我有提到，目前主流的解决方案就是领域驱动设计，也叫DDD<span class="orange">[1]</span>。DDD是Eric Evans在其2004年的同名书中提出来的一个思想，但一直仅仅局限在Java的圈子里，直到2014年，微服务兴起后大家才发现它可以指导微服务的拆分，这才走进了大多数人的视野。用一句话简单总结，DDD就是一套方法论：<strong>通过对业务分层抽象，分析定义出领域模型，用领域模型驱动我们设计系统，最终将复杂的业务模型拆解为独立运维的领域模型。</strong></p><p>实际我自己在使用微服务开发的过程发现，微服务整体应该是一个动态网络结构<span class="orange">[2]</span>，随着业务的发展，这个网络结构也会发生变化。DDD能帮助我们前期分析出一个较好的网络结构，但实际上，我们更应该思考的是如何整体优化动态网络：<strong>减少核心节点，保护核心节点，降低网络深度等等。</strong></p><p>怎么理解动态网络优化呢？我们可以做个思维实验：假设我们将所有的功能都拆解成微服务，任意的微服务节点之间都可以相互调用，调用越频繁它们之间的距离就越近。那么我们考虑一下，当我们网站的访问请求流量稳定后，我们整个微服务节点组成的网络状态是怎么样的？</p><p>首先网络节点的相互制约总会让那些相互之间强依赖的、高耦合的节点，越走越近，最后聚集成一团节点。其次那些跟业务逻辑无关的节点，逐渐被边缘化，甚至消失。我们看这些聚集成团的节点，如果团里的点聚合太近了，其实是不适合拆分的，它们整体应该作成一个微服务。等这些节点太近的团合并成一个微服务节点后，我们再看那些聚集在一起、又不太近的节点就是一个个微服务了。</p><p>所以，我们在启动项目时，不用太过纠结应该如何去拆解微服务。而应该持续关注，并思考每个微服务节点的合理性。就像看待动态网络一样，持续地调整优化，去除核心节点。最终它会伴随你业务的发展阶段，达到各个阶段的稳定动态网络结构。</p><p>就像我们上节课“待办任务”Web服务一样，我们可以先简单地将我们的项目后端分为：用户微服务和待办任务微服务。当然这里我们目前的业务太简单了，用DDD去分析，也是大材小用。随着我这个项目的业务发展，我们添加的功能会越来越多。让微服务根据业务一起成长演变就可以了。这并不是说我们就放任微服务不管了，而是从整体网络的角度思考，去看我们的微服务如何演进。</p><p><img src="https://static001.geekbang.org/resource/image/58/cf/589469933bdc3b0335f9754ff2f555cf.png?wh=1388*626" alt="" title="微服务演进过程"></p><h2>合之，Streaming</h2><p>看完拆解，我们再看合并。合并呢，换个高大上的词其实就是前面课程中提到的编排。目前为止，我们整个“待办任务”Web应用架构的设计基本完成了，而且所有节点都是Stateless的了。变成Stateless节点后，其实对于前端的同学来说，一点都不陌生，比如React的单向数据流中的State也要求我们Immutable，Immutable其实就是Stateless。</p><p>我们上面已经看到了，拆解后的架构是个动态网络，那我们应该怎么合并或者编排呢？当然你像SFF那样通过传统的函数，将每个HTTP数据的请求结果通过数组或对象加工处理，再将这些结果返回也是可以的。但我在这里想向你介绍另外一种编排思路，工作流。</p><p><img src="https://static001.geekbang.org/resource/image/d2/57/d267851c983200b430dfb5d53fd4d557.gif?wh=512*288" alt="" title="数据流演示图"></p><p>我们可以将用户的请求想象成我们的呼吸系统，我们的肺就是SFF，而微服务和FaaS节点就是需要氧气的各个器官。我们吸一口气，氧气进入肺部，血液循环将氧气按顺序流经我们每个器官，这就是请求链路。每个器官一接收到新鲜血液，就会吸取氧气返回二氧化碳，最终血液循环将二氧化碳带到肺部呼出，这个就是数据返回链路。我们的各个器官，就被请求链路通过新鲜血液到来的这个事件串联起来了，这个就是事件流，也就是用一个个事件去串联FaaS或微服务。</p><p>现在我们用<a href="https://time.geekbang.org/column/article/227454">[第3课]</a> 讲的，PHP发邮件改造一下，举个例子。当用户注册时，我们完全可以将用户的信息和注册验证码存入数据库；PHP发邮件的FaaS触发器改为数据库插入新记录触发事件；用户从邮箱验证获取验证码，把验证码写到输入框后，点击验证，则是另一个HTTP触发器，触发FaaS函数校验验证码通过，修改数据库注册成功，并且返回302跳转到登录成功页面。具体流程可参考下图：</p><p><img src="https://static001.geekbang.org/resource/image/71/38/712127c53b9ba90fe69cf00179862338.png?wh=2022*996" alt="" title="案例流程图"></p><p>当然现在这个解决方案也有成熟对应的云BaaS服务：Serverless工作流<span class="orange">[3]</span>。</p><h2>安全门神</h2><p>理解了拆合的思想，我们就可以将目前“待办项目”的架构再演进一下：静态文件我们用CDN托管，前端项目只负责域名支撑和index.html，剩下的请求直接访问FaaS微服务。这时候，我估计你会问，咱们数据的安全性如何保障呢？是的，到目前为止，我们的FaaS都一直在用匿名模式访问，完全没有任何安全防护可言，也就是说目前我们FaaS服务的接口一直都在互联网上“裸奔”。</p><h3>鉴权</h3><p>其实，FaaS提供的安全防护通常是放在触发器上的。触发器的授权类型或认证方式我们可以设置为：匿名anonymous或函数function。匿名方式就是不需要签名认证，匿名的用户也能访问；而函数方式，则是需要签名认证<span class="orange">[4]</span>，这个签名认证的算法，参数需要用到我们账户的访问秘钥ak/sk<span class="orange">[5]</span>，ak/sk相当于我们云账户的银行卡密码，这么重要的账户信息，我们只能限定在服务端使用，前端代码里绝对不可以出现。</p><p>也就是说，我们只能在服务端使用函数安全认证方式。如果是这种方案，我们的“待办任务”架构就演进成下图这样了。</p><p><img src="https://static001.geekbang.org/resource/image/1f/b3/1f30750203dddab945fb1ff471e40ab3.png?wh=2388*884" alt="" title="“待办任务”架构图"></p><p>那有没有针对匿名认证方式的安全策略呢？当然有，这里我们同样需要借鉴一下微服务的鉴权设计：JSON Web Token，简称JWT<span class="orange">[6]</span>。JWT简单来说，就是将用户身份信息和签名信息，一起传到客户端去了。用户在请求数据时，带上这个JWT，服务端通过校验签名确定用户身份。JWT存在于客户端，JWT验证只需要通过服务端的sk和算法验证签名。同样，我画了张图，以帮助你理解。</p><p><img src="https://static001.geekbang.org/resource/image/f1/f1/f187db1d21b27d58c55000272f4825f1.png?wh=1760*1292" alt="" title="JWT示意图"></p><p>要解决后端互调的安全性，我们用VPC或IP白名单，都很容易解决。比较难处理的是前后端的信任问题，JWT正好就提供了一种信任链的解决思路。当然，关于鉴权也有一些云服务商推出了一些更加安全易用的BaaS服务，例如AWS的IAM和Cognito<span class="orange">[7]</span>。</p><p>安全性是我们考虑架构设计时重要的一环，因为安全架构设计的失败，会直接导致我们资产的损失。鉴权是识别用户身份，防止用户信息泄漏和恶意攻击使用的。但根据我统计的数据，我们在日常99%的问题，都发生在新版本上线的环节。</p><p>那我们该怎么稳定持续地快速迭代，发布新版本上线呢？我们可以回想一下<a href="https://time.geekbang.org/column/article/224559">[第 1 课]</a>，小程和小服的例子，小程最后实现NoOps后，小服则只要将代码合并到指定分支就可以发布上线了。那现实中，这点该怎么实现呢？</p><p>当我们的项目Serverless化以后，代码的质量变得尤为重要。你可以想想，Serverless化之前，你不小心上线了一个bug，影响的范围最大也就只有一个应用。但是Serverless化之后，如果是核心节点发布了严重的bug上线，那么影响的范围就是所有依赖它的线上应用了。</p><p>不过，你也不用太担心，微服务和FaaS都具备快速独立迭代的能力。以前我们一个应用的迭代周期通常要一周到两周。但对于Serverless化后的应用来说，每个节点借助独立运维的特性，可以随时随地的发布上线。</p><p>综上，我们知道了，微服务和FaaS都是快速迭代的，修复问题很快，但我们也不能每次都等问题出现，再去依赖这个能力呀。有没有什么办法可以提前发现问题，保证我们既快又稳？目前软件工程的最佳做法就是代码流水线的发布管道。</p><h3>发布管道</h3><p>发布管道的流水线主要有3个部分：</p><ol>
<li>代码发布前的验证，代码测试覆盖率CI/CD；</li>
<li>模拟流量回归测试通过，发布到灰度环境；</li>
<li>代码正式上线，灰度环境替换正式环境。流水线的每个节点产生的结果，都会作为下一个节点必要的启始参数。</li>
</ol><p><img src="https://static001.geekbang.org/resource/image/8e/4b/8e3a2a76cfcf51d2cc2eafaaf0a2db4b.png?wh=1872*928" alt="" title="发布管道流水线"></p><p>我们先看看上图，我来解释下这个流程。</p><ul>
<li>我们的代码合并到指定分支后，通常我会用Develop分支。</li>
<li>Git的钩子就会触发后续的流水线，开始进入构建打包、测试流程。</li>
<li>测试节点做的事情就是跑所有测试Case，并且统计覆盖率。</li>
<li>覆盖率验证通过，代码实例用录制流量模拟验证。</li>
<li>模拟验证通过，发布代码实例到灰度环境。</li>
<li>线上根据灰度策略，将小部分流量导入灰度环境验证灰度版本。</li>
<li>在灰度窗口期，比如两个小时，灰度验证没有异常则用灰度版本替换正式版本；反之则立即丢弃这个灰度版本，止损。</li>
</ul><p>这套流程，目前规模大一些的互联网公司发布流程基本都在这样跑，如果你不是很了解，可以自己尝试用我们介绍的Serverless工作流或者云服务商提供的工作流工具<span class="orange">[8]</span>动手搭建下。</p><p>在这套流程的基础上，很多企业为了追求更高的稳定性，还会设定环境隔离的流水线和安全卡口。比如隔离测试环境和线上环境，测试环境用来复现故障。每次代码进入发布管道，都必须先在测试环境跑通，跑通后安全卡口放行，才能进入线上环境的流水线。</p><h2>总结</h2><p>这节课，我们继续讲后端的BaaS化。我们再梳理一下这节课的重要知识点吧。</p><ol>
<li>如何拆解BaaS应用，我们学习了微服务的重要拆解思想DDD：<strong>通过对业务分层抽象，分析定义出领域模型，用领域模型驱动我们设计系统，最终将复杂的业务模型拆解为独立运维的领域模型。</strong>另外我也介绍了另一种更适合初创企业的拆分思路：动态网络演进。</li>
<li>拆解完之后，我们就要考虑合并。这里我们介绍了代码编排以外的另一种编排方式：事件流编排，它就是通过一个个事件顺序将我们的微服务或FaaS串联起来。</li>
<li>为了解决拆解后，微服务之间的信任问题。我们先了解了FaaS触发器的安全方案：数字签名。还借鉴了微服务的鉴权做法JWT，将用户鉴权加密信息放在客户端，让鉴权服务变成Stateless。最后，为了让微服务又快又稳地发布版本，我们借鉴了微服务的发布管道：打造自动灰度流水线。</li>
</ol><h2>作业</h2><p>这节课的作业就是我们JWT鉴权的“待办任务”Web应用，你来部署上线。</p><p>后端代码GitHub地址：<a href="https://github.com/pusongyang/todolist-backend/tree/lesson06">https://github.com/pusongyang/todolist-backend/tree/lesson06</a></p><p>前端代码GitHub地址：<a href="https://github.com/pusongyang/todolist-frontend">https://github.com/pusongyang/todolist-frontend</a></p><p>演示预览地址：<a href="http://lesson6.jike-serverless.online/list">http://lesson6.jike-serverless.online/list</a></p><p>期待你的作业，如果今天的内容让你有所收获，也欢迎你把文章分享给身边的朋友，邀请他加入学习。</p><h2>参考资料</h2><p>[1] <a href="https://en.wikipedia.org/wiki/Domain-driven_design">https://en.wikipedia.org/wiki/Domain-driven_design</a></p><p>[2] <a href="https://en.wikipedia.org/wiki/Dynamic_network_analysis">https://en.wikipedia.org/wiki/Dynamic_network_analysis</a></p><p>[3] <a href="https://www.aliyun.com/product/fnf">https://www.aliyun.com/product/fnf</a></p><p>[4] <a href="https://github.com/aliyun/fc-nodejs-sdk/blob/master/lib/client.js?spm=a2c4g.11186623.2.15.16e016d7lo8NBQ#L840">https://github.com/aliyun/fc-nodejs-sdk/blob/master/lib/client.js?spm=a2c4g.11186623.2.15.16e016d7lo8NBQ#L840</a></p><p>[5] <a href="https://help.aliyun.com/document_detail/154851.html?spm=5176.2020520153.0.0.371a415dLXyltZ">https://help.aliyun.com/document_detail/154851.html?spm=5176.2020520153.0.0.371a415dLXyltZ</a></p><p>[6] <a href="https://jwt.io/">https://jwt.io/</a></p><p>[7] <a href="https://docs.aws.amazon.com/zh_cn/cognito/latest/developerguide/what-is-amazon-cognito.html">https://docs.aws.amazon.com/zh_cn/cognito/latest/developerguide/what-is-amazon-cognito.html</a></p><p>[8] <a href="https://www.aliyun.com/product/yunxiao/devops?spm=5176.10695662.1173276.1.6c724a38akCjgo">https://www.aliyun.com/product/yunxiao/devops?spm=5176.10695662.1173276.1.6c724a38akCjgo</a></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/0c/30/bb4bfe9d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lyonger</span>
  </div>
  <div class="_2_QraFYR_0">老师，请问下BaaS化就是把后端服务设计成微服务，提供标准的RestFul Api给客户端，且客户端无需关注服务端的ops工作是吧， 感觉像是后端服务直接&quot;云化&quot;了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你的感觉没有错。<br>后端BaaS化，现在就是让大家的应用后端，变成类似云服务商提供的BaaS服务。<br>采用HTTP，而非RPC方案，这样的好处是可以享受我们后面讲的Service Mesh接管流量，和服务发现，服务注册。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-09 09:39:28</div>
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
  <div class="_2_QraFYR_0">今天在实践的过程中,走了些弯路,在此提醒一下同学.<br><br>实际上的架构图是&#39;JWT示意图&#39;.<br>index-faas和rule-faas两个是分开部署的.<br>两个函数的触发器都是ANONYMOUS方式!!!<br><br>index-faas中的&#47;api&#47;currentUser接口负责生成jwtToken.<br>rule-faas中接口&#47;api&#47;rule的post、delete、put方法才验证jwtToken.<br>作为实验,可以在`新建`代办前,将浏览器中的cookie移除,再请求时会收到403的错误码.<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 陈独秀同学~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-29 14:15:49</div>
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
  <div class="_2_QraFYR_0">老师的课程实践性很强。实验的例子，如果没有云服务器，可以在本地环境模似吗。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以的，我后面的实践课就是教大家在本地搭建环境。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-29 09:29:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTIro8BKyich3jMOTRFibsbYeX9oWfNUa6dAcNDia5EH7VVHbibiaZavnDX1VlZ8NbQGrtJuYz0oKkfgSNA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Programmer</span>
  </div>
  <div class="_2_QraFYR_0">老师想咨询一下，为什么“待办任务”架构图到这一节没有SFF层了呢，这里的faas函数充当什么角色呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: SFF层是可选项，通常是后端比较完善或者已经比较成熟的团队，用FaaS做服务编排，将后端的服务通过这一层聚合数据给前端使用。<br>这里是FaaS的另外一种用法，直接将node.js应用整个运行在FaaS函数里面，index-faas.js是入口文件。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-28 22:20:28</div>
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
  <div class="_2_QraFYR_0">老师好，现在我们做的作业就是baas吗，怎么感觉还是在faas中做，这样还是不能连数据库哇</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个是给大家展示用FaaS去做微服务。不是不能连接数据库，FaaS本身并没有做限制。具体的应用场景，可以都尝试一下，体验一下优缺点。Serverless未来也会针对各种场景优化，只要官方不做限制，都可以多尝试一下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-06 09:02:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/49/10/af49fa20.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>左耳朵狮子</span>
  </div>
  <div class="_2_QraFYR_0">&quot;线上根据灰度策略，将小部分流量导入灰度环境验证灰度版本。&quot; 老师这块能否说的再细一点。<br>假设在Cloud-native 开发中，小部分流量倒入这个docker container(s) 来验证，如果灰度发布不成功。是否多个containers 全部销毁，还是编排到其他containers。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 灰度发布不成功，只需要撤回灰度版本就可以了。撤回后需要重新修改验证，再次发布，直到灰度版本成功。<br>这个类似于多版本在线的概念，后面Knative会讲到如何控制多版本流量分配权重。<br>线上版本用K8s或者Knative的方案都支持版本回滚。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-19 07:03:47</div>
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
  <div class="_2_QraFYR_0">我昨天实践了云溪社区的一篇文章,是关于Serverless进行CI&#47;CD的.<br>觉得值得一看,推荐给大家.<br>[Serverless 实战 —— Funcraft + OSS + ROS 进行 CI&#47;CD](https:&#47;&#47;yq.aliyun.com&#47;articles&#47;741414)<br>该文章,演示了发布前的验证环节.<br><br>后面的回归验证和灰度流量验证,就需要其他技术了.<br>比如借助k8s方便的部署多套环境,进行回归验证.<br>使用k8s&#47;istio实现金丝雀&#47;灰度发布.<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 文中提的前后端分离方案，其实中间SFF可以抽象出接口层。后端同学做微服务，针对文档化接口编程，很容易测试。前端同学自己写FaaS编排后端BaaS，也很容器Mock测试。前后端，各个微服务分开迭代测试发布。<br>掌握了这些工具，其实研发流程和架构设计，随手拈来。所以现在也有云开发者的概念，未来大部分开发者应该都掌握云服务的各种能力。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-29 12:04:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTIrg3ZKwyfUSoWDdB4mdmEOCeicfWO5WJXvNwDJsy6QV18gwQ5rlUg9MmYGIjCWU6QqQIZnXXGonIw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>miser</span>
  </div>
  <div class="_2_QraFYR_0">或许我有基本的后端部署认知，我感觉serverless在架构设计上意义不大，但是在人员安排上意义重大，对于前端来说可以不太关心运维的工作，代码推上线就完了，它帮助前端或者不太懂运维的开发者解决了大量运用工作，降低了上线成本和学习成本。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，未来组织结构和架构演变上，前后端分离会更加彻底。前端同学可以自己写FaaS编排服务，后端同学专注写BaaS。FaaS还具备价格优势。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-29 11:45:25</div>
  </div>
</div>
</div>
</li>
</ul>