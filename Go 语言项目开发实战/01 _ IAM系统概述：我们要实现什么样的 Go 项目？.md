<audio title="01 _ IAM系统概述：我们要实现什么样的 Go 项目？" src="https://static001.geekbang.org/resource/audio/79/da/796yye80b0c10b8b2ddca36bd9bb77da.mp3" controls="controls"></audio> 
<p>你好，我是孔令飞。从今天开始我们进入课前准备阶段，我会用3讲的时间给你讲清楚，我们要实现的实战项目 IAM 应用长啥样、它能干什么，以及怎么把它部署到 Linux 服务器上。先和我一起扫除基础的障碍，你就能够更轻松地学习后面的课程了。</p><p>今天这一讲，我先来说说为什么选择 IAM 应用，它能实现什么功能，以及它的架构和使用流程。</p><h2>项目背景：为什么选择 IAM 系统作为实战项目？</h2><p>我们在做 Go 项目开发时，绕不开的一个话题是安全，如何保证 Go 应用的安全，是每个开发者都要解决的问题。虽然 Go 应用的安全包含很多方面，但大体可分为如下 2 类：</p><ul>
<li><strong>服务自身的安全</strong>：为了保证服务的安全，需要禁止非法用户访问服务。这可以通过服务器层面和软件层面来解决。服务器层面可以通过物理隔离、网络隔离、防火墙等技术从底层保证服务的安全性，软件层面可以通过 HTTPS、用户认证等手段来加强服务的安全性。服务器层面一般由运维团队来保障，软件层面则需要开发者来保障。</li>
<li><strong>服务资源的安全</strong>：服务内有很多资源，为了避免非法访问，开发者要避免 UserA 访问到 UserB 的资源，也即需要对资源进行授权。通常，我们可以通过资源授权系统来对资源进行授权。</li>
</ul><p><strong>总的来说，为了保障Go应用的安全，我们需要对访问进行认证，对资源进行授权</strong>。那么，我们要如何实现访问认证和资源授权呢？</p><!-- [[[read_end]]] --><p>认证功能不复杂，我们可以通过 JWT （JSON Web Token）认证来实现。授权功能比较复杂，授权功能的复杂性使得它可以囊括很多 Go 开发技能点。因此，在这个专栏中，我将认证和授权的功能实现升级为 IAM 系统，通过讲解它的构建过程，给你讲清楚 Go 项目开发的全部流程。</p><h2>IAM 系统是什么？</h2><p>IAM（Identity and Access Management，身份识别与访问管理）系统是用 Go 语言编写的一个 Web 服务，用于给第三方用户提供访问控制服务。</p><p>IAM 系统可以帮用户解决的问题是：<strong>在特定的条件下，谁能够/不能够对哪些资源做哪些操作</strong>（<strong>Who</strong> is <strong>able</strong> to do <strong>what</strong> on <strong>something</strong> given some <strong>context</strong>），也即完成资源授权功能。</p><blockquote>
<p><span class="reference">提示：以后我们在提到 IAM 系统或者 IAM 时都是指代 IAM 应用。</span></p>
</blockquote><p>那么，IAM 系统是如何进行资源授权的呢？下面，我们通过 IAM 系统的资源授权的流程，来看下它是如何工作的，整个过程可以分为 4 步。</p><p><img src="https://static001.geekbang.org/resource/image/ee/50/eed75fcd91d6e726ca74315d65193150.jpg?wh=2513*1134" alt="" title="IAM 系统的功能示意图"></p><ol>
<li>用户需要提供昵称、密码、邮箱、电话等信息注册并登录到 IAM 系统，这里是以用户名和密码作为唯一的身份标识来访问 IAM 系统，并且完成认证。</li>
<li>因为访问 IAM 的资源授权接口是通过密钥（secretID/secretKey）的方式进行认证的，所以用户需要在 IAM 中创建属于自己的密钥资源。</li>
<li>因为 IAM 通过授权策略完成授权，所以用户需要在 IAM 中创建授权策略。</li>
<li>请求 IAM 提供的授权接口，IAM 会根据用户的请求内容和授权策略来决定一个授权请求是否被允许。</li>
</ol><p>我们可以看到，在上面的流程中，IAM 使用到了 3 种系统资源：用户（User）、密钥（Secret）和策略（Policy），它们映射到程序设计中就是 3 种 RESTful 资源：</p><ul>
<li>用户（User）：实现对用户的增、删、改、查、修改密码、批量修改等操作。</li>
<li>密钥（Secret）：实现对密钥的增、删、改、查操作。</li>
<li>策略（Policy）：实现对策略的增、删、改、查、批量删除操作。</li>
</ul><h2>IAM 系统的架构长啥样？</h2><p>知道了 IAM 的功能之后，我们再来详细说说 IAM 系统的架构，架构图如下：</p><p><img src="https://static001.geekbang.org/resource/image/0a/42/0a5f6fd67af1eda1c690c8216dc5e042.jpg?wh=3197*2063" alt="" title="IAM 系统的完整架构"></p><p>总的来说，IAM 架构中包括 9 大组件和 3 大数据库。我将这些组件和功能都总结在下面的表格中。这里面，我们主要记住5个核心组件，包括iam-apiserver、iam-authz-server、iam-pump、marmotedu-sdk-go和iamctl的功能，还有3个数据库Redis、MySQL和MongoDB的功能。</p><p><img src="https://static001.geekbang.org/resource/image/6c/71/6cdbde36255c7fb2d4f2e718c9077a71.jpeg?wh=1920*1043" alt="" title="前 5 个组件是我们需要实现的核心组件。[br]后 4 个组件是一些旁路组件，不影响项目的使用。[br]如果感兴趣，你可以自行实现"></p><p>此外，IAM 系统为存储数据使用到的 3 种数据库的说明如下所示。<br>
<img src="https://static001.geekbang.org/resource/image/e6/f2/e68c21e1991c74becc4b8a6a8bf5a8f2.jpeg?wh=1818*496" alt=""></p><h3>通过使用流程理解架构</h3><p>只看到这样的系统架构图和核心功能讲解，你可能还不清楚整个系统是如何协作，来最终完成资源授权的。所以接下来，我们通过详细讲解 IAM 系统的使用流程及其实现细节，来进一步加深你对 IAM 架构的理解。总的来说，我们可以通过 4 步去使用 IAM 系统的核心功能。</p><p><strong>第1步，创建平台资源。</strong></p><p>用户通过 iam-webconsole（RESTful API）或 iamctl（sdk marmotedu-sdk-go）客户端请求 iam-apiserver 提供的 RESTful API 接口完成用户、密钥、授权策略的增删改查，iam-apiserver 会将这些资源数据持久化存储在 MySQL 数据库中。而且，为了确保通信安全，客户端访问服务端都是通过 HTTPS 协议来访问的。</p><p><strong>第2步，请求 API 完成资源授权。</strong></p><p>用户可以通过请求 iam-authz-server 提供的 /v1/authz 接口进行资源授权，请求 /v1/authz 接口需要通过密钥认证，认证通过后 /v1/authz 接口会查询授权策略，从而决定资源请求是否被允许。</p><p>为了提高 /v1/authz 接口的性能，iam-authz-server 将密钥和策略信息缓存在内存中，以便实现快速查询。那密钥和策略信息是如何实现缓存的呢？</p><p>首先，iam-authz-server 通过调用 iam-apiserver 提供的 gRPC 接口，将密钥和授权策略信息缓存到内存中。同时，为了使内存中的缓存信息和 iam-apiserver 中的信息保持一致，当 iam-apiserver 中有密钥或策略被更新时，iam-apiserver 会往特定的 Redis Channel（iam-authz-server 也会订阅该 Channel）中发送 PolicyChanged 和 SecretChanged 消息。这样一来，当 iam-authz-server 监听到有新消息时就会获取并解析消息，根据消息内容判断是否需要重新调用 gRPC 接来获取密钥和授权策略信息，再更新到内存中。</p><p><strong>第3步，授权日志数据分析。</strong></p><p>iam-authz-server 会将授权日志上报到 Redis 高速缓存中，然后 iam-pump 组件会异步消费这些授权日志，再把清理后的数据保存在 MongoDB 中，供运营系统 iam-operating-system 查询。</p><p>这里还有一点你要注意：iam-authz-server 将授权日志保存在 Redis 高性能 key-value 数据库中，可以最大化减少写入延时。不保存在内存中是因为授权日志量我们没法预测，当授权日志量很大时，很可能会将内存耗尽，造成服务中断。</p><p><strong>第4步，运营平台授权数据展示。</strong></p><p>iam-operating-system 是 IAM 的运营系统，它可以通过查询 MongoDB 获取并展示运营数据，比如某个用户的授权/失败次数、授权失败时的授权信息等。此外，我们也可以通过 iam-operating-system 调用 iam-apiserver 服务来做些运营管理工作。比如，以上帝视角查看某个用户的授权策略供排障使用，或者调整用户可创建密钥的最大个数，再或者通过白名单的方式，让某个用户不受密钥个数限制的影响等等。</p><h3>IAM 软件架构模式</h3><p>在设计软件时，我们首先要做的就是选择一种软件架构模式，它对软件后续的开发方式、软件维护成本都有比较大的影响。因此，这里我也会和你简单聊聊 2 种最常用的软件架构模式，分别是前后端分离架构和 MVC 架构。</p><h4>前后端分离架构</h4><p>因为IAM 系统采用的就是前后端分离的架构，所以我们就以 IAM 的运营系统 iam-operating-system 为例来详细说说这个架构。一般来说，运营系统的功能可多可少，对于一些具有复杂功能的运营系统，我们可以采用前后端分离的架构。其中，<strong>前端负责页面的展示以及数据的加载和渲染，后端只负责返回前端需要的数据</strong>。</p><p>iam-operating-system 前后端分离架构如下图所示。</p><p><img src="https://static001.geekbang.org/resource/image/a2/76/a2e1f1cc135debd86611yya1f221c476.jpg?wh=2519*1447" alt=""></p><p>采用了前后端分离架构之后，当你通过浏览器请求前端 ops-webconsole 时，ops-webconsole 会先请求静态文件服务器加载静态文件，比如 HTML、CSS 和 JavaScript，然后它会执行 JavaScript，通过负载均衡请求后端数据，最后把后端返回的数据渲染到前端页面中。</p><p>采用前后端分离的架构，让前后端通过 RESTful API 通信，会带来以下5点好处：</p><ul>
<li>可以让前、后端人员各自专注在自己业务的功能开发上，让专业的人做专业的事，来提高代码质量和开发效率</li>
<li>前后端可并行开发和发布，这也能提高开发和发布效率，加快产品迭代速度</li>
<li>前后端组件、代码分开，职责分明，可以增加代码的维护性和可读性，减少代码改动引起的 Bug 概率，同时也能快速定位 Bug</li>
<li>前端 JavaScript 可以处理后台的数据，减少对后台服务器的压力</li>
<li>可根据需要选择性水平扩容前端或者后端来节约成本</li>
</ul><h4>MVC 架构</h4><p>但是，如果运营系统功能比较少，采用前后端分离框架的弊反而大于利，比如前后端分离要同时维护 2 个组件会导致部署更复杂，并且前后端分离将人员也分开了，这会增加一定程度的沟通成本。同时，因为代码中也需要实现前后端交互的逻辑，所以会引入一定的开发量。</p><p>这个时候，我们可以尝试直接采用 MVC 软件架构，MVC 架构如下图所示。</p><p><img src="https://static001.geekbang.org/resource/image/a2/eb/a23b8ba92705710c694fd7cb99812feb.jpg?wh=1753*869" alt=""></p><p>MVC 的全名是 Model View Controller，它是一种架构模式，分为 Model、View、Controller 三层，每一层的功能如下：</p><ul>
<li>View（视图）：提供给用户的操作界面，用来处理数据的显示。</li>
<li>Controller（控制器）：根据用户从 View 层输入的指令，选取 Model 层中的数据，然后对其进行相应的操作，产生最终结果。</li>
<li>Model（模型）：应用程序中用于处理数据逻辑的部分。</li>
</ul><p><strong>MVC 架构的好处是通过控制器层将视图层和模型层分离之后，当更改视图层代码后时，我们就不需要重新编译控制器层和模型层的代码了。</strong>同样，如果业务流程发生改变也只需要变更模型层的代码就可以。在实际开发中为了更好的 UI 效果，视图层需要经常变更，但是通过 MVC 架构，在变更视图层时，我们根本不需要对业务逻辑层的代码做任何变化，这不仅减少了风险还能提高代码变更和发布的效率。</p><p>除此之外，还有一种跟 MVC 比较相似的软件开发架构叫三层架构，它包括UI 层、BLL 层和DAL 层。其中，UI 层表示用户界面，BLL 层表示业务逻辑，DAL 层表示数据访问。在实际开发中很多人将 MVC 当成三层架构在用，比如说，很多人喜欢把软件的业务逻辑放在 Controller 层里，将数据库访问操作的代码放在 Model 层里，软件最终的代码放在 View 层里，就这样硬生生将 MVC 架构变成了伪三层架构。</p><p>这种代码不仅不伦不类，同时也失去了三层架构和 MVC 架构的核心优势，也就是：<strong>通过 Controller层将 Model层和 View层解耦，从而使代码更容易维护和扩展</strong>。因此在实际开发中，我们也要注意遵循 MVC 架构的开发规范，发挥 MVC 的核心价值。</p><h2>总结</h2><p>一个好的 Go 应用必须要保证应用的安全性，这可以通过认证和授权来保障。也因此认证和授权是开发一个 Go 项目必须要实现的功能。为了帮助你实现这 2 个功能，并借此机会学习 Go 项目开发，我将这 2 个功能升级为一个 IAM系统。通过讲解如何开发 IAM 系统，来教你如何开发 Go 项目。</p><p>因为后面的学习都是围绕 IAM 系统展开的，所以这一讲我们要重点掌握 IAM 的功能、架构和使用流程，我们可以通过 4 步使用流程来了解。</p><p>首先，用户通过调用 iam-apiserver 提供的 RESTful API 接口完成注册和登录系统，再调用接口创建密钥和授权策略。</p><p>创建完密钥对和授权策略之后，IAM 可以通过调用 iam-authz-server 的授权接口完成资源的授权。具体来说，iam-authz-server 通过 gRPC 接口获取 iam-apiserver 中存储的密钥和授权策略信息，通过 JWT 完成认证之后，再通过 <a href="https://github.com/ory/ladon">ory/ladon</a> 包完成资源的授权。</p><p>接着，iam-pump 组件异步消费 Redis 中的数据，并持久化存储在 MongoDB 中，供 iam-operating-system 运营平台展示。</p><p>最后，IAM 相关的产品、研发人员可以通过 IAM 的运营系统 iam-operating-system 来查看 IAM 系统的使用情况，进行运营分析。例如某个用户的授权/失败次数、授权失败时的授权信息等。</p><p>另外，为了提高开发和访问效率，IAM 分别提供了 marmotedu-sdk-go SDK 和 iamctl 命令行工具，二者通过 HTTPS 协议访问 IAM 提供的 RESTful 接口。</p><h2>课后练习</h2><ol>
<li>你在做 Go 项目开发时经常用到哪些技能点？有些技能点是 IAM 没有包含的？</li>
<li>在你所接触的项目中，哪些是前后端分离架构，哪些是 MVC 架构呢？你觉得项目采用的架构是否合理呢？</li>
</ol><p>期待在留言区看到你的思考和分享，我们下一讲见！</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/f2/56/131b4173.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>素衣绾绾</span>
  </div>
  <div class="_2_QraFYR_0">就是说，项目的源码已经完成，然后作者会在专栏中带着我们一起搭建一个web项目框架的雏形，也就是专栏中说的一些目录规范，api设计规范，错误包设计等，具体的功能代码，就是照着已经写好的代码挑选其中核心的功能进行讲解，是吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 专栏会介绍如何开发Go项目，demo项目只是专栏介绍的开发技术的一个落地项目。<br><br>二者是包含与被包含的关系，专栏包含了实战项目，Go开发方法和思路，一些知识点的讲解，以及我的一些研发经验和建议。<br><br>只看源码，你看到的仅仅是一个源码，但是很可能不了解源码背后的构建思路以及一些其它需要注意的地方，比如说只看源码，你仍然不知道开发规范，仍然不知道用了哪些设计模式和技巧。<br><br>所以建议以专栏学习为主，代码阅读为辅。通过代码去验证专栏的设计思路。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-27 09:19:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/16/6b/af7c7745.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>tiny🌾</span>
  </div>
  <div class="_2_QraFYR_0">沙发，跟作者也算半个同事</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-26 19:24:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/36/7d/0cfc0752.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小屎丸</span>
  </div>
  <div class="_2_QraFYR_0">为啥选择 MongoDB 作为日志数据分析展示库，怎么考虑的？谢谢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. mongodb带有数据聚合功能，在某些场景下可以实现复杂的数据统计<br>2. 而且字段增减随意，查询方便<br>3. IAM系统授权日志量不大，场景也不复杂，再加上，这里是想展示mongodb的教学，所以就采用了mongo</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-11 01:13:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/e4/1b/e8f5f5e4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>大菠萝</span>
  </div>
  <div class="_2_QraFYR_0">authz，这个名字有啥含义么？主要是这个z</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 认证：authentication，缩写：authn<br>授权：authorization，缩写：authz</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-06 12:31:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/ff/21/77949293.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Paualf</span>
  </div>
  <div class="_2_QraFYR_0">在做项目的时候，项目间通信经常使用的是kafka这类消息队列，后台会经常用搜索功能，这部分数据都会放在了ES中如搜索，日志或者后台系统的一些搜索。IAM系统中使用了Redis做消息队列，使用了MongoDB去进行存储和查询，这些是我目前想到不一样的技能点。<br><br>现在接触的大部分项目都是前后端分离的架构，接触过有一些后台管理系统使用的MVC架构。我觉得使用前后端分离的架构更加的合理，这样关注点进行分离，让专业的人干专业的事情，这样确实像老师文章中说的，增加了人力和沟通成本。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-02 09:51:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/89/71/03a99e56.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>、荒唐_戏_</span>
  </div>
  <div class="_2_QraFYR_0">完蛋了，上来一个iam系统就没理解😓</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: iam系统主要完成认证和授权功能。<br>认证：用来判断是否是平台的合法用户，例如用户名和密码就是认证的一种方式<br>授权：用来判断，是否可以访问平台的某类资源。<br><br>认证和授权 2个功能可以抽象成一个系统，这个系统名我起名为iam。类似于aws的iam，腾讯云的cam和阿里云的ram。<br><br>建议你参考下腾讯云的cam以协助你理解：https:&#47;&#47;cloud.tencent.com&#47;document&#47;product&#47;599&#47;40011。<br><br>不难，花几分钟相信你就会理解了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-03 10:11:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/dd/09/feca820a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>helloworld</span>
  </div>
  <div class="_2_QraFYR_0">是不是可以理解为每一个使用iam系统作为认证和资源授权的业务应用，该业务应用的用户每次访问该业务应用的接口都要先经过iam系统，认证和资源鉴权通过后，才能继续访问该业务应用接口呢，也就是iam是业务应用的网关代理？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 理解的没毛病，可以理解为一个授权webhook。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-24 01:15:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/b7/0e/20ed751b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>老妖纣王</span>
  </div>
  <div class="_2_QraFYR_0">老师，您好，正好讲到iam系统有几个问题想请教你一下，在项目中使用认证授权，遇到几个疑惑的问题一直困扰着我？<br>假如APP是内部应用的场景，流程是用户通过登录API直接获取token，然后每次携带token进行请求，网关拦截进行token的认证。<br>（1）这里面token是保存在客户端的，是否安全？<br>（2）token是有有效期的，网上一般说失败了通过刷新token去刷新，（客户端模式）客户端定时请求去触发，如果客户端多的话，频繁地刷新token的请求，显然带来很大的网络开销？为了解决这个问题，我们通过请求然后判断token时间是否到了阈值，然后续签（服务端实现）。但是这个也有问题，如果长时间没有请求会导致token过期，就不能续签了。如果是像淘宝这种，感觉永远都不会退出的，是不是就是延长token的有效时间+服务端刷新。那么永久有效，需要刷新token吗？<br>（3）oauth2中有客户端概念，我认为客户端就是app，那么appId和秘钥是不是保存到了app应用端，这样会有安全问题吗？如果密码都保存在了app端，服务端还需要对秘钥验证吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. 使用客户端用用户名登录后，证明客户端短时间是安全的，token默认有2小时的过期时间。<br><br>2. 这个刷新成本服务端还是能抗住的。如果token过期就只能重新登录了，至于多久过期，每个业务会根据需要自己选择，但很多开源软件一般默认2小时过期。<br><br>3. 密钥保存在客户端，需要客户端使用者去保证密钥不泄露，服务端也会保存一份密钥，用来对请求进行再加密，并和客户端传来的token对比，如果一致则请求通过。<br><br><br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-02 20:22:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/dd/09/feca820a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>helloworld</span>
  </div>
  <div class="_2_QraFYR_0">iam这个认证与资源授权系统和网关的认证与鉴权功能有何本质区别呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 跟网关的认证功能是一样的，iam的认证功能也是参考了tyk网关的实现。<br><br>跟网关的鉴权还是有些区别，iam是对资源鉴权，网关是对api鉴权，就是谁能或不能调用哪个api接口。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-09 00:39:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/1f/17/f2a69e62.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Fis.</span>
  </div>
  <div class="_2_QraFYR_0">我理解authz是面向用户的借口，apiserver是核心处理模块，sdk不直接对接apiserver吗？authz和api server分离设计的优点是什么？另外authz从redis获取数据，那不是有可能读到过期数据？（比如当时写入请求还在写入mysql过程中，还没同步到redis）</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: sdk可以对接iam-apiserver和iam-authz-server。<br>authz和api server分离的优点，其实是：数据流和控制流的分离。<br><br>这样做的好处如下：<br><br>1. 控制流主要是用来做资源的CURD，如果控制流服务出故障，会影响用户的使用，但不会影响用户的业务。<br>2. 数据流出故障会影响用户的业务。<br><br>对于一个系统来说，影响用户业务是很严重的事情，而且我们日常变更最多的是控制流服务，控制流和数据流分离，使得我们发布变更控制流服务时，不会影响到数据流服务，也即不会影响用户的业务。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-02 08:25:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/dd/09/feca820a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>helloworld</span>
  </div>
  <div class="_2_QraFYR_0">IAM能和网关结合使用吗，还是必须二选一呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 网关做的是接口鉴权，也就是哪个用户&#47;密钥可以访问哪个接口。而IAM做的是资源鉴权。例如淘宝中的iphone资源等</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-24 01:09:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/7a/d2/4ba67c0c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Sch0ng</span>
  </div>
  <div class="_2_QraFYR_0">目标是go工程，IAM只是一个支点。对IAM暂且理解个大概，能跟得上后面的学习即可。学完整个专栏，go工程能力更完善即可，顺带理解IAM更多细节，岂不是美滋滋。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 老哥，理解的很到位!</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-09 21:56:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/57/2a/a00a3ce7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Nio</span>
  </div>
  <div class="_2_QraFYR_0">IAM只是我们学习这个专栏的小工具，主要还是借助这个项目学习go语言项目的设计及思想以及具体实现，其中这些理论能力会体现在项目的各个实现细节中，是这个意思吧</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Nio理解的很正确!</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-27 17:48:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/4b/11/d7e08b5b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>dll</span>
  </div>
  <div class="_2_QraFYR_0">不太明白，怎么同意这个用户创建并给密钥对，并给到对应的权限策略，这个步骤不需要人来同意吗？如果谁都能注册自己选择权限，那不是很危险？这个aim和jwt有什么关系</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 管理员注册iam系统，并登录系统，创建密钥对和策略。密钥对分发给其他人，其他人通过密钥对进行资源授权，授权时填写自己的用户名就可以了。<br><br>Jwt是iam认证的方式</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-04 11:21:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/76/93/64ed7385.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>arch</span>
  </div>
  <div class="_2_QraFYR_0">令飞老师，腾讯云的CAM和IAM系统有什么区别吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 授权模型不一样。<br>CAM是ABAC。<br>IAM授权模型类似于RBAC和ACL</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-27 23:58:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLDUJyeq54fiaXAgF62tNeocO3lHsKT4mygEcNoZLnibg6ONKicMgCgUHSfgW8hrMUXlwpNSzR8MHZwg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>types</span>
  </div>
  <div class="_2_QraFYR_0">关于秘钥对有几个问题不太明白<br>按照文中所讲，密钥对是在控制平面中创建<br>1. 根据后面的文章，数据流中（authz）是基于JWT认证的<br>请问秘钥对如何进行用于JWT，jwt token是谁负责创建的APP还是authz server<br>2. 秘钥对包含secretId，secretKey<br>请问APP包含密钥对的什么信息，是如何提供给APP的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. 在apiserver中创建密钥对，authz服务会自动将新创建的秘钥信息从apiserver中拉去，并缓存在authz的内存中。<br><br>jwt token可以使用iamctl jwt sign &lt;secretID&gt; &lt;secretKey&gt;来创建。<br>也可以通过网上的在线工具来创建<br>还可以自己编写jwt token生成代码。<br>jwt token的创建是有一定规则的，所以只要满足jwt token规则的工具都可以创建。<br><br>直接回答你的问题：jwt token由APP侧来创建<br><br>2. APP需要知道secretId和secretKey。<br>通过secretId和secretKey，再加上jwt token固定的生成规则，就可以创建jwt token用于请求。 </p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-05 00:50:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/25/29/39/59f9d41c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>乳酸菌</span>
  </div>
  <div class="_2_QraFYR_0">始终没有理解MVC 和三层架构的区别，老师有例子能加深理解吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 重要区别：<br>MVC总的controller层不做具体的业务逻辑处理，主要用来做请求逻辑分发，可以把业务逻辑具体实现放在类似service这样的包中。在controller引用这个包。<br>三层架构中的BLL层，是用来做业务处理的。<br><br>还有其它区别，具体可以google下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-28 20:50:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/0d/00/d07ea760.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>qiaocc</span>
  </div>
  <div class="_2_QraFYR_0">异步消费没有用过。 python里面有用过celery， go里面有类似的第三方包吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: machinery  和 gocelery 了解下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-09 14:19:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/0d/00/d07ea760.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>qiaocc</span>
  </div>
  <div class="_2_QraFYR_0">1. 授权策略 2. 对资源的授权， 这两个有什么区别呀，能举个例子吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 授权策略是一个策略，里面说明了，谁能&#47;不能对哪些资源做哪些操作。<br><br>对资源的授权，是一个动作，例如，小明删除密钥。<br><br>对资源的授权是通过验证授权策略来实现的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-09 14:18:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/63/84/f45c4af9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Vackine</span>
  </div>
  <div class="_2_QraFYR_0">iam 存储的密钥是创建了之后就是静态的一直存在mysql的么？会不会有密钥截获的风险？还是每个密钥有超时时间？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 一直保存在mysql数据库中，登录用户名密码登录系统获取临时密钥保存在客户端，默认2小时过期。后面的通信全都走https，通过认证，https，过期时间来保证安全性</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-28 23:14:42</div>
  </div>
</div>
</div>
</li>
</ul>