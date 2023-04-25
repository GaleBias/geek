<audio title="11 _ 经验：Serverless开发最佳实践" src="https://static001.geekbang.org/resource/audio/b5/02/b5748378ca4f1036894f1df5c7bf0702.mp3" controls="controls"></audio> 
<p>你好，我是秦粤。上节课，我们了解了利用K8s集群的迁移和扩展能力，可以解决云服务商锁定的问题。我们还横向对比了各大云服务商的特点和优势，纵向梳理了云服务商提供的各种服务能力。最后我们可以看到，利用Knative提供的Container Serverless能力，我们可以任意迁移部署我们的应用架构，选择适合我们的云服务商。</p><p>但同时我们也发现，FaaS相对于Knative，反而是我们的瓶颈，我们无法平滑地迁移FaaS的函数。云服务商大力发展FaaS，其实部分原因也是看中了FaaS新建立起来的Vendor-lock，因此目前各大运营商都在拼FaaS的体验和生态建设。</p><p>那这节课，我们就来看看FaaS是如何解除云服务商锁定的吧。但在正式开始之前呢，我们先得了解一些FaaS的使用场景，以免一些同学误会，要是你的实践是为了Serverless而去Serverless，那就不好了，我们还是应该从技术应用的角度出发。</p><h2>FaaS场景</h2><p>我从<a href="https://time.geekbang.org/column/article/229905">[第5课]</a> 开始就在讲FaaS和BaaS的底层实现原理Container Serverless，是希望通过底层实现原理，帮助你更好地掌握Serverless的思想。但在日常工作中，使用Serverless，只需要借助云服务商提供的Serverless化的FaaS和BaaS服务就可以了。</p><!-- [[[read_end]]] --><p>就像我上节课中所讲的，FaaS因为可以帮助云服务商利用碎片化的物理机算力，所以它是最便宜的。在日常工作中，我十分鼓励你去使用FaaS。</p><p>那么说到使用FaaS，以我的经验来说，我们可以将FaaS的最佳使用场景分为2种：事件响应和Serverless应用。</p><h3>FaaS事件响应</h3><p>FaaS最擅长的还是处理事件响应，所以我们应该先了解清楚，你选择的云服务商的FaaS函数具体支持哪些触发器，例如阿里云的触发器列表<span class="orange">[1]</span>、腾讯云的触发器列表<span class="orange">[2]</span>。因为这些触发器通常都是云服务商根据自身Serverless的成功案例所制定的下沉方案，也就是说这些触发器都具备完整的使用场景。基于FaaS支持的触发器，设计事件响应函数，我们就可以最大限度地享受FaaS的红利，而且不用担心用法不正确。</p><h3>Serverless应用</h3><p>那如果FaaS是作为Serverless应用的使用场景，我们则需要关注大型互联网公司的成功案例所沉淀出来的成熟方案。这样既可以轻松简单地搭建出Serverless应用，还能帮我们节省不少费用。</p><p>因为我们尝试用FaaS去解构自己的整个应用时，如果超出了目前FaaS的适应边界，那么技术挑战还是比较大的。所以我并不推荐你将整个现存的应用完全Serverless化，而是应该去找出适用FaaS事件响应的场景去使用FaaS。</p><p>如果你关注Serverless就不难发现，至少目前它还处在发展阶段，很多解决方案还在摸索之中。大厂具备技术实力，也乐于在拓展Serverless边界的同时发现痛点、解决痛点并占领技术高地。那如果你是一般用户，最好的选择还是持续关注Serverless动向，及时了解哪些方案可以为自己所用。当然，这是Serverless的舒适圈，但技术挑战也意味着机遇，我确实也看到有不少同学因为拓展Serverless的边界而被吸收进入大厂，去参与Serverless共建。</p><h2>FaaS痛点</h2><p>以上就是我对FaaS应用场景的一些看法和建议。虽然FaaS最擅长是事件响应，但是如果可以改造Serverless应用，无疑FaaS的附加值会更高。接下来我们就看看FaaS在Serverless应用过程中切实存在的痛点，相信随着课程一路跟下来的同学一定深有体会。</p><ul>
<li>首先，调试困难，我们每次在本地修改和线上运行的函数都无法保持一致，而且线上的报错也很奇怪，常见的错误都是超时、内存不足等等，让我们根本无法用这些信息去定位问题。</li>
<li>其次，部署麻烦，每次改动都要压缩上传代码包，而且因为每个云服务商的函数Runtime不一致，导致我们也很难兼容多云服务商。</li>
<li>最后，很多问题客服无法跟进，需要自己到处打听。</li>
</ul><p>但如果你有仔细阅读文档的话，你就会发现各大云服务商都会提供命令行调试工具，例如阿里云提供的"fun"工具<span class="orange">[3]</span>、腾讯云合作的Serverless Framework等等。不过云服务商提供的工具，都是云服务商锁定的。为了破解FaaS的新Vendor-lock，我从目前热门的Serverless社区中，选择了3个Serverless应用框架和1个平台，介绍给你帮助你破解Vendor-lock，同时也能更好地解决痛点。</p><h2>Serverless应用框架</h2><h3>Serverless Framework <span class="orange">[4]</span></h3><p>Serverless框架作为第一款支持跨云服务商的Serverless应用解决方案，一旦进入首页，我们就可以看到：Serverless X Tencent Cloud。其实最早的Serverless框架官方例子是AWS的，不过从去年开始被腾讯云强势入驻了。正如我上节课中所讲，腾讯云将Serverless作为切入云服务商市场的突破口，正在大力推进Serverless腾讯云的生态建设。</p><p>Serverless框架具备先发优势，而且已经有很多成熟的应用方案，我们可以直接拿来使用。我们可以通过Serverless Component用yaml文件编排整个Serverless应用架构。</p><p>另外Serverless Framework也支持将应用部署到其他Provider：AWS、阿里云、Azure，还有Knative等等，正如以下图示。不过这里有个小问题：就是我们只有在<a href="https://www.serverless.com/framework/docs/providers/">英文版Serverless框架的官网</a>上才能看到其他云服务商的Provider和文档介绍（<a href="https://www.serverless.com/cn">中文版Serverless框架官网</a>一并给出）。</p><p><img src="https://static001.geekbang.org/resource/image/8b/19/8b389343945134f7a0782ae1eb2d6919.jpg?wh=5869*1551" alt=""></p><p>Serverless Framework官方本身主要解决跨云服务商Serverless应用的部署运维问题。它利用类似K8s组件Component的概念，让开发者可以自己开发并且共享组件。通过组件来支持多语言的模板，而且Serverless组件的扩展能力还让Serverless框架具备生态优势。</p><p>目前，Serverless的确是前端参与的比较多，因此在Serverless应用框架中，除了Node.js外，其他语言的选择并不多。下面我就再介绍2款Node.js新的Serverless框架。</p><h3>Malagu <span class="orange">[5]</span></h3><p>Malagu是基于 TypeScript 的 Serverless First、可扩展和组件化的应用框架。为了提升Serverless的开发体验，只专注Node.js Serverless应用。</p><p>首先，Malagu的理念是支持前后端一体化，基于JSON RPC，前端像调用本地方法一样调用后端方法。前后端支持RPC和MVC两种通信形式，MVC可以满足传统纯后端Rest风格接口的开发。</p><p>其次，Malagu框架本身也是基于组件化实现的，将复杂的大型项目拆解成一个个Malagu组件，可以提高代码的复用能力、降低代码的维护难度。</p><p>最后，命令行工具插件化，默认提供初始化、运行、构建、部署能力，通过插件可以扩展命令行的能力。</p><p>我有幸与Malagu的开发者共事过，Malagu汲取了众家所长，我觉得这是一个值得关注的Serverless框架。而且目前Malagu已经有很多合作伙伴，他们正在积极使用和推广Malagu的解决方案。</p><h3>Midway FaaS <span class="orange">[6]</span></h3><p>接着是我今天想给你介绍的重点，就是：Midway Faas和它的工具链"f"。Midway FaaS 是用于构建 Node.js 云函数的 Serverless 框架，可以帮助你在云原生时代大幅降低维护成本，更专注于产品研发。</p><p>Midway FaaS诞生于Node.js老牌框架Midway，我也有幸与Midway的开发者一起共事过。Midway FaaS 是阿里巴巴集团发起的开源项目，由一个专业的 Node.js 架构团队进行维护。目前已大规模应用于阿里集团各 BU 线上业务，稳定承载了数千万的流量。</p><p>Midway FaaS有这样一些特点，也可以说是特色。</p><ul>
<li><strong>跨云厂商：</strong>一份代码可在多个云平台间快速部署，不用担心你的产品会被云厂商所绑定。</li>
<li><strong>代码复用：</strong>通过框架的依赖注入能力，让每一部分逻辑单元都天然可复用，可以快速方便地组合以生成复杂的应用。</li>
<li><strong>传统迁移：</strong>通过框架的运行时扩展能力，让Egg.js 、Koa、Express.js等传统应用无缝迁移至各云厂商的云函数。</li>
</ul><p>我们的“待办任务”Web服务，也将采用这个方案进行重构。</p><p>首先，我们通过命令行全局安装工具链"f"。</p><pre><code>npm install -g @midwayjs/faas-cli
</code></pre><p>然后，拉取我们最新的<a href="https://github.com/pusongyang/todolist-backend/tree/lesson11">lesson11代码分支</a>，注意这个分支跟之前的分支相比改动比较大。因此，我们需要重新安装NPM依赖包。</p><pre><code>npm install
</code></pre><p>前端代码还是在public目录，后端代码在src目录，项目里面的代码逻辑也比较简单，你可以自己看一下。我们需要讲解的是比较核心的配置文件f.yml，这里有我放的f.yml的文件结构。</p><pre><code>service: fc-qinyue-test

provider:
  name: aliyun
  runtime: nodejs10

functions:
  index:
    handler: index.handler
    events:
      - http:
          path: /*

  getUser:
    handler: user.get
    events:
      - http:
          path: /user/:id
          method: get

package:
  include:
    - public/*

aggregation:
  agg-demo-all:
    deployOrigin: false
    functions:
      - index
      - getUser

custom:
  customDomain:
    domainName: lesson11-ali.jike-serverless.online

</code></pre><p>这里我们主要关注的是Provider，目前Midway FaaS支持阿里云和腾讯云，后续还会陆续增加其他云服务商。我们切换云服务商只需要修改Provider就可以了。其次，你也可以看到Functions部分，我们定义了所有的入口函数和路径path的映射关系。最后我们在部署环节，自己指定了域名。我们在配置域名解析时，根据云服务商提供的方式，通常都是修改域名的CNAME，指向云服务提供的FaaS地址就可以了。</p><p>Midway FaaS的强大之处就在于我们可以借助它提供的工具链"f"，根据我们的f.yml文件配置内容，从而直接在本地模拟云上事件触发，调试我们的应用。</p><p>这里你只需要执行：</p><pre><code>f invoke -p
</code></pre><p>我们就可以通过浏览器访问：<a href="http://127.0.0.1:3000">http://127.0.0.1:3000</a> 调试我们的FaaS函数代码了。并且因为执行就在我们本地，所以调试时，我们可以借助现代IDE提供的Node.js debug能力逐行调试。而我们部署时也只需要一行代码：</p><pre><code>f deploy
</code></pre><p>然后Midway FaaS就会根据我们的f.yml文件配置内容，帮我们部署到云上。</p><p>下面这2个URL地址，就是我用Midway FaaS分别部署在阿里云和腾讯云上的“待办任务”Web服务。</p><p>阿里云“待办任务”Web服务：<a href="http://lesson11-ali.jike-serverless.online/">http://lesson11-ali.jike-serverless.online/</a></p><p>腾讯云“待办任务”Web服务：<a href="http://lesson11-tx.jike-serverless.online/">http://lesson11-tx.jike-serverless.online/</a></p><p>这里我需要提醒你一下，这2个云服务都是以阿里云的OTS服务作为数据库存储。这也是FaaS服务编排的强大之处，现在有些云服务商提供的BaaS服务是支持其他云服务商通过ak/sk访问的。另外如下图所示，如果我们采用ak/sk编排云服务商的HTTP API服务，那么利用FaaS就可以实现混合云编排服务了。</p><p><img src="https://static001.geekbang.org/resource/image/62/37/623069c983b6359bd5418ee1f35cc337.jpg?wh=1702*1248" alt=""></p><p>如果你留意过GitHub，你会发现Midway FaaS和Malagu都是非常新的GitHub仓库。Serverless作为一门新兴技术，还有非常大的想象空间。</p><p>另外阿里云也推出了自己的云开发平台，而且将Serverless应用的生态集成了进去。目前Midway FaaS和Malagu也都在其中作为应用创建模板供你使用。</p><h3>阿里云开发平台 <span class="orange">[7]</span></h3><p>阿里云开发平台就是由咱们专栏特别放送中的被采访者，杜欢（笔名风驰）负责的。</p><p>他的核心理念呢，就是将大厂成熟的Serverless应用场景抽象成解决方案，并且透出到云开发平台上来，为各个中小企业和个人开发者赋能。</p><p>正如下图所示，云开发平台是将大厂的行业应用场景沉淀成了解决方案，解析生成Serverless架构，这样我们就可以把云开发平台上实例化的“解决方案”快速应用到我们的Serverless应用中来。基于成熟Serverless的应用，“反序列化”快速沉淀为解决方案，这也是云服务平台会收录Midway FaaS和Malagu的原因。</p><p>而且我相信以后还会有更多的Serverless应用解决方案被收录到云开发平台中的。另外阿里云开发平台并不限制开发语言，但截止到咱们这节课的发布时间，阿里云开发平台的模板仅支持Node.js和Java的Serverless应用，其它语言还需要等成功案例沉淀。</p><p><img src="https://static001.geekbang.org/resource/image/bd/06/bd7cfa2c4711ad7c0533d5a198323306.jpg?wh=1746*1044" alt=""></p><h2>总结</h2><p>这节课我向你介绍了如何打破FaaS的Vendor-lock，并且介绍了我经验中FaaS的最佳使用场景：事件响应和Serverless应用。事件响应无疑就是FaaS的最佳使用场景；而Serverless应用则需要我们在云服务商提供的FaaS服务上，利用我们专栏前面学习到的知识和原理，挑战将完整的应用完全Serverless化。</p><p>为了解决FaaS的痛点和云服务商锁定，我向你介绍了3个框架和一个平台。</p><ul>
<li>目前由腾讯云主导的Serverless Framework，它支持多语言，具备组件扩展性。目前仅在英文版官网可以看到其支持的云服务商Provider文档。</li>
<li>由阿里云FC团队主导的Malagu解决方案，目前仅支持Node.js，同样具备扩展性。目前正在积极的和很多开发团队合作共建Serverless应用生态。</li>
<li>由阿里巴巴Node.js框架Midway团队主导的Midway FaaS解决方案，它仅支持Node.js，扩展性由Midway团队官方保障。Midway团队的社区响应迅速，很多GitHub上的issue会在第一时间响应。</li>
<li>由阿里云主导的阿里云开发平台，它支持多语言，具备模板扩展性，扩展性由官方提供的模板保障。目前收录了Serverless生态比较成熟的方案，可以快速部署到阿里云FC上。虽然开发平台有一定的云服务商锁定嫌疑，但其采用的模板却可以支持解除云服务商锁定，例如Midway FaaS。</li>
</ul><p>作为本专栏的最后一课，我还想再啰嗦一句。Serverless确实是一门新兴技术，其未来无限可能，接下来该由你探索了！</p><blockquote>
<p>想要我的财宝吗？想要的话就给你，去找出来吧，这世上所有的一切都放在那里。<br>
世界吗？没错！去追求自由。<br>
去超越吧！在信念的旗帜的带领下。</p>
<p align="right">—— 海贼王 罗杰</p>
</blockquote><h2>作业</h2><p>Serverless技术实践还是很重要的，最后一节课的作业我们就继续实操。请你将这节课“待办任务”Web服务的Midway FaaS版本，在本地调试运行起来，并且通过 <code>f deploy</code> 把它部署到阿里云或者腾讯云上。</p><p>期待你的成果，有问题欢迎随时留言与我交流。如果今天的内容让你有所收获，也欢迎你把文章分享给更多的朋友。</p><h2>参考资料</h2><p>[1] <a href="https://help.aliyun.com/document_detail/74707.html?">https://help.aliyun.com/document_detail/74707.html</a></p><p>[2] <a href="https://cloud.tencent.com/document/product/583/31927">https://cloud.tencent.com/document/product/583/31927</a></p><p>[3] <a href="https://help.aliyun.com/document_detail/140283.html?">https://help.aliyun.com/document_detail/140283.html</a></p><p>[4] <a href="https://www.serverless.com/cn/framework/docs/getting-started/">https://www.serverless.com/cn/framework/docs/getting-started/</a></p><p>[5] <a href="https://github.com/alibaba/malagu">malagu框架项目地址</a>；<a href="https://www.yuque.com/cellbang/malagu">malagu框架详细文档</a></p><p>[6] <a href="https://github.com/midwayjs/midway-faas">https://github.com/midwayjs/midway-faas</a></p><p>[7] <a href="https://workbench.aliyun.com/">https://workbench.aliyun.com/</a></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/f4/17/0bb45a21.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>peter</span>
  </div>
  <div class="_2_QraFYR_0">挺好的，不错！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-18 19:35:16</div>
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
  <div class="_2_QraFYR_0">今天的例子部署的比较顺利.<br>只修改了f.yml和src&#47;config&#47;config.default.ts.<br><br>使用npm install安装依赖后<br>`f invoke -p`就可以本地调试了<br>`f deploy`就可以部署上阿里云了<br><br>这次部署后,只有一个函数,但是代码中的函数却都可以成功调用.<br>应该是`aggregation`的功劳吧.<br><br>-------<br>之前尝试&lt;阿里云开发平台&gt;时,自动生成过一个项目,也是跟老师这次的代码结构类似,也是有f.yml文件.<br>好像就是用的`Midway FaaS`框架<br><br>不过这次想用它部署老师的项目就遇到了问题.<br>编辑器会提示:VS Code 的 tsserver 已被其他应用程序(例如运行异常的病毒检测工具)删除。请重新安装 VS Code。<br>我也不会修,暂时在钉钉群中询问了也没有答复.<br><br>------<br>老师这一会node.js,一会TypeScript的,作为后端开发的我,比较懵.<br>只会简单的部署,代码不会改也不会调.<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: TS现在我们阿里已经全面使用了。TS避免了JS的弱类型，类对象支持等等问题，而且还支持IoC。<br>Midway FaaS正式依赖IoC来实现简化业务逻辑的。<br>Node.js也属于后端语言了，我以前写过2年Java，7年PHP。现在全职做Node.js。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-11 20:01:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/PiajxSqBRaEJnj0bL0YdiawRe9rauOCiczLlPuM9JdmJhFzGNMq4ILic4FGOzGfibZHLcfiblYMPA8AevjVpETYlTTHg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_a7fcb9</span>
  </div>
  <div class="_2_QraFYR_0">老师，看完了您的文章。有2个问题想请教一下。<br>问题一：<br>【k8s主要是对容器的编排，knative是一种基于k8s的serverless平台的实现，相比较于k8s提供了灰度发布、自动扩缩容之类的功能。】请问这样理解对吗？<br>-------------------------------------------------------------------------<br>问题二：<br>【Serverless 有多种实现方法，比如：Serverless Framework，Malagu，Midway FaaS。上边三者类似knative可以帮助我们搭建serverless平台。而“阿里云开发平台”是已经搭建好的serverless平台。】请问这样理解对吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 问题一，理解的差不多，knative在k8s插件基础上提供了serverless的能力，例如有部分网络能力依赖istio插件。灰度发布还需要其他的K8s插件提供。<br>问题二，这3者都不是帮我们搭建serverless的。他们是开发框架，帮助我们快速开发serverless应用。他们提供主流云服务商的部署支持。部署应用的时候，则需要通过框架的配置，应用会直接上传并部署在云服务商上。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-30 17:54:42</div>
  </div>
</div>
</div>
</li>
</ul>