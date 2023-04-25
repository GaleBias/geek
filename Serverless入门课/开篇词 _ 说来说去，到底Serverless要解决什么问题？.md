<audio title="开篇词 _ 说来说去，到底Serverless要解决什么问题？" src="https://static001.geekbang.org/resource/audio/f5/6e/f52633b211fe2715f15034ee0646106e.mp3" controls="controls"></audio> 
<p>你好，我是秦粤，欢迎你和我一起学习Serverless。说起Serverless这个词啊，我估计你应该不陌生，就算你没详细了解过，但我确定你肯定在过去几年时间里听别人说过。去年我在参加<a href="https://gmtc.infoq.cn/2019/shenzhen/track/675">GMTC全球大前端技术大会</a>的时候，也惊讶地发现国内几个大公司都已经有成熟的应用案例了，所以我当时就感叹说：“Serverless终于要飞入寻常百姓家了。”</p><h3>三个问题</h3><p>作为一名Serverless的拥趸者，过去几年时间里，我总是喜欢向朋友和同事推荐Serverless技术。不过，在“推销”的过程中，他们经常会问我一些问题，那在今天的开篇词里，我就来统一回答下这些共性的问题吧，我估计你也会问到。</p><p><strong>问题一：说来说去，到底Serverless要解决什么问题？</strong></p><p>我不知道你有没有算过你们公司每年在服务器上的开销，反正在创业之前我是没算过，总觉得这钱算了也没必要，还是应该多花心思在怎么挣钱上，但后面当我真正自己开公司之后，才知道柴米贵。</p><p>咱们就拿自己部署一套博客来说吧，常见的Node.js MVC架构，需要购买云服务商的Linux虚拟机、RDS关系型数据库，做得好的话还要购买Redis缓存、负载均衡、CDN等等，再专业一点，可能还会考虑容灾和备份，这么算下来一年最小开销都在1万元左右。但如果你用Serverless的话，这个成本可以直接降到1000元以下。</p><!-- [[[read_end]]] --><p>Serverless是对运维体系的极端抽象，就像iPhone当年颠覆诺基亚一样，它给应用开发和部署提供了一个极简模型。这种高度抽象的模型，可以让一个零运维经验的人，几分钟就部署一个Web应用上线，并对外提供服务。也就是说，你根本不需要再学习怎么在Linux上装Web服务器，怎么配置负载均衡等等这些繁琐的偏运维方向的工作。</p><p>所以，你要问我Serverless解决了什么问题，<span class="orange">一句话总结就是它可以帮你省钱、省力气。</span></p><p><strong>问题二：为什么阿里巴巴、腾讯这样的公司都在关注Serverless？</strong></p><p>首先，Serverless可以有效降低企业中中长尾应用的运营成本。中长尾应用就是那些每天大部分时间都没有流量或者有很少流量的应用，你可以想想你们公司是不是也有很多。这一点我特别有感触，尤其是企业在落地微服务架构后，一些边缘的微服务被调用的概率其实很低。而这个时候，我们往往又很难通过人工来控制中长尾应用，因为这里面不少应用还是被强依赖的，不可以直接下线处理。Serverless之前，这些中长尾应用至少要独占1台虚拟机；现在有了Serverless的极速冷启动特性，企业就可以节省这部分开销。</p><p>其次，Serverless可以提高研发效能。我们专栏会讲到Serverless应用架构的设计，其中，SFF（Serverless For Frontend）可以让前端同学自行负责数据接口的编排，微服务BaaS化则让我们的后端同学更加关注领域设计。可以说，这是一个颠覆性的变化，它能够进一步放大前端工程师的价值。</p><p>最后我想说Serverless作为一门新兴技术，未来的想象空间很大。我看到有创业公司用FaaS来做基础设施编排和云服务编排；也有外包公司利用Serverless应用架构的快速迭代能力，提升开发效率，降低出错率，还可以给自己沉淀领域的解决方案；还有包括风险投资方也在逐渐开始关注Serverless领域，毕竟这也是一个新的风口。我讲大企业的使用方式只是希望给你一些灵感，不想过多限制你的想象，Serverless的疆域边界还在等你去扩展。</p><p>这里是GMTC会议上几个大公司的分享资料，你感兴趣的话，可以先看看。</p><ul>
<li><a href="https://static001.geekbang.org/con/55/pdf/1710853715/file/%E7%BC%AA%E4%BC%A0%E6%9D%B0.pdf">阿里跨境供应链前端架构演进与 Serverless 实践</a></li>
<li><a href="https://static001.geekbang.org/con/55/pdf/3151321591/file/%E7%8E%8B%E4%BF%8A%E6%9D%B0%20%20Serverless%20%E5%89%8D%E7%AB%AF%E5%B7%A5%E7%A8%8B%E5%8C%96%E8%90%BD%E5%9C%B0%E4%B8%8E%E5%AE%9E%E8%B7%B5.pdf">Serverless 前端工程化落地与实践</a></li>
<li><a href="https://static001.geekbang.org/con/55/pdf/1359804153/file/%E6%9D%9C%E6%AC%A2%20%20%E4%BB%8E%E5%89%8D%E7%AB%AF%E5%92%8C%E4%BA%91%E5%8E%82%E5%95%86%E7%9A%84%E8%A7%86%E8%A7%92%E7%9C%8B%20Serverless%20%E4%B8%8E%E6%9C%AA%E6%9D%A5%E7%9A%84%E5%BC%80%E5%8F%91%E7%94%9F%E6%80%81.pdf">从前端和云厂商的视角看 Serverless 与未来的开发生态</a></li>
</ul><p><strong>问题三：Serverless对前端工程师来说会有什么机遇？为什么我们要学习Serverless？</strong></p><p>相对其他工种而言，Serverless给前端工程师带来的机遇更大，它能让前端工程师也享受到云服务的红利。如果说Node.js语言的出现解放了一波前端工程师的生产力，那Node.js+Serverless又可以进一步激发前端工程师的创造力。口说无凭，到时候你看完咱们专栏里的例子就能体会到这句话的意思了。</p><p>另外，我觉得学习Serverless是成为云开发者的最适合的切入点。无论你是零基础，还是资深服务端运维，都可以从Serverless上学习到现代服务端运维体系的重要思想（我在第一节课就会讲）。</p><p>回答完这几个问题，我想你对Serverless已经有了初步了解。接下来我也不想绕弯子，我就再和你聊聊我准备怎么给你讲这门课吧。</p><h3>课程设计</h3><p>基础篇，我会继续带你理解Serverless要解决什么问题，以及Serverless的边界和定义。搞清楚了来龙去脉，我们会进入动手环节，我会通过一个例子来给你讲解Serverless引擎盖下的工作原理，以及FaaS的一些应用场景。</p><p>进阶篇，我们将一起学习FaaS的后端解决方案BaaS，以及我们自己现有的后端应用如何BaaS化。为了更好地展现Serverless的发展历程和背后的思考，我也为你准备了一个基于Node.js的待办任务的Web应用，你要做好准备，这里我会给你布置很多动手作业。</p><p>GitHub地址：<a href="https://github.com/pusongyang/todolist-backend">https://github.com/pusongyang/todolist-backend</a></p><p>实战篇，我会通过Google开源的Kubernetes向你演示本地化Serverless环境如何搭建，并根据我的经验，和你聊聊Serverless架构应该如何选型，以及目前Serverless开发的最佳实践。</p><p><img src="https://static001.geekbang.org/resource/image/8b/83/8bde1e4a6ae3adb4f5ab5f410a9b1e83.jpg?wh=2308*882" alt="" title="学习路径图"></p><p>最后，我再来介绍下我自己吧。我叫蒲松洋，秦粤是我的花名。2006年从电子科技大学毕业后，我就进入了UT斯达康（现在这公司已经谢幕，它是小灵通的主要生产厂商）做通讯相关的工作，当时的职位是PHP前端工程师。2013年，我跳槽加入百度，从PHP前端工程师转为了Node.js前端工程师。2015年开始又和朋友折腾创业，用Node.js做智能家居IoT。2016年底，创业没成，我又回到了国内某一线互联网公司，负责Node.js应用治理和Node.js微服务架构设计。</p><p>我在用Node.js做微服务时发现，微服务本身提出了很多理念，但微服务在服务端运维却缺少给力的支撑平台，后来我们就试着用容器集群搭建了自己的Container Serverless。结果证明它不但可以支撑Node.js微服务运维，还可以提高Node.js中长尾应用的资源利用率。再后面，大家都意识到了Serverless带来的便利性，于是我们团队也就整体参与到了公司Serverless整体建设中了。</p><p>在研究并落地Serverless技术的过程中，我发现国外的Serverless开源社区其实比国内更加活跃。自从2014年AWS推出第一个FaaS服务Lambda后，国外的很多公司都在积极推进Serverless的生态发展，并且开始占领高地，制定了很多的规范。而放眼国内，目前还只有为数不多的大型互联网公司在重点跟进，其他人基本上只是在观望或者看热闹。</p><p>因此，我也特别希望通过这个专栏能够带你真正理解Serverless，并让你尽快享受到技术的红利。Serverless 肯定是未来云计算发展的重点方向，作为工程师，特别是前端工程师，我们应该思考的是如何抓住这波机遇，如何利用FaaS+各种创意，组合碰撞出各种化学反应，去为公司、为自己创造更大的价值。</p><p>以上就是今天的全部内容。有关Serverless，不知道你的看法是怎样的？如果你有什么疑惑，或者在实践中遇到了哪些困难？欢迎在留言区中提出，我们共同探讨。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/7b/c1/22fbcab2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>阿杜S考特</span>
  </div>
  <div class="_2_QraFYR_0">serverless字面的意思就是无服务的架构，我个人的理解就是当没有request访问或者触发的时候，他不启动任何服务和资源，一旦触发了就会启动服务去处理任务，好处是不用关心服务是否挂了，它适合处理耗时不长的快速事务处理，当流量大的时候，它也能自动扩容去响应客户端。但是如果大量的并发一下冲过来的时候或者一下子没有流量的时候，它的自动扩容和缩容机制是否会导致更多的开销？aws的lambda现在有了version和concorrency capacity的概念，也就是最小的启动服务数，这样感觉又违背了serverless的初衷，请问老师，是否有比较标准或者最佳实践来处理这种情况呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 目前Serverless并不是万金油，它有它的局限性和使用场景。我们学习Serverless，应该将它作为一种工具，在适合的场景中使用。课程的后面部分会讲到，所以我的最佳实践，讲的是结合Serverless和CaaS的混合应用架构。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-15 20:25:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/ee/d2/7024431c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>探索无止境</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，serverless在实际应用上虽然说跟语言无关，但实际公司选择是node.js多，还是其他语言多？感觉更多是提到前端工程师的机会，后端作为Java开发者影响多大？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Node.js在Serverless Computing，也就是FaaS，中因为JIT的特性，冷启动速度确实是优势。我们的实践经验来说，Java或其它语言比较适合下沉，去做后端服务化，也就是BaaS。目前比较好的是FaaS+BaaS架构。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-15 20:53:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/41/87/46d7e1c2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Better me</span>
  </div>
  <div class="_2_QraFYR_0">Serverless解决了什么问题？<br><br>1、省钱省力<br>可减少运维资源成本以及学习成本<br>2、提高研发效能<br>前端可自己对数据接口编排，外包可快速迭代，降低容错，沉淀领域解决方案<br>3、解放生产力，激发创造力<br>前端可自由通过FaaS组合完成业务需求，大大激发前端创造力<br>4、后端应用BaaS化<br>通过FaaS的后端解决方案将后端服务BaaS化，让后端工程师更加专注领域设计</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 总结的好~！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-27 23:37:29</div>
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
  <div class="_2_QraFYR_0">serverless 就是 让用户 不关注 (less) server , 把 server 下沉，抽象话</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-15 22:01:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/47/31/f35367c8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小晏子</span>
  </div>
  <div class="_2_QraFYR_0">很早之前体验过微信的小程序云开发，感觉这个就是通过severless实现的，直接写函数然后上传到小程序云端，然后就能用了，非常方便。不过也遇到一些困难，比如脱离了服务器，不知道该如何给后端服务安装证书，也没找到文档说明，后来还是在内部论坛上才问到该如何解决，所以我感觉开发简单了，但如果涉及到一些运维方面的工作，蛮不好弄的，也许自己搭建的serverless环境会比较方便。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，我们专栏后面几节课会教大家如何用K8s搭建私有Serverless环境</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-15 18:00:28</div>
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
  <div class="_2_QraFYR_0">才发现,老师文章提到的&quot;GMTC 会议上几个大公司的分享资料&quot;,<br>在极客时间的每日一课中,都有完整的视频.<br>我还都已经看完了.😁</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-02 14:15:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/05/b1/f43a473b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Tao</span>
  </div>
  <div class="_2_QraFYR_0">一个完整的app后端，包括用户系统：用户信息，关注，内容系统：帖子，评论，点赞等，还有通知私信等。这样的场景适合用serverless来实现嘛，还是用传统后端开发来做</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果是新业务，可以体验一下Serverless技术栈。像你说的用户信息，通知私信，这些都有很多现成的云服务提供了，用FaaS很容易搭建：云服务编排。但如果是已有业务上云，那么建议还是逐步迁移吧。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-15 21:12:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/EcYNib1bnDf5dz6JcrE8AoyZYMdqic2VNmbBtCcVZTO9EoDZZxqlQDEqQKo6klCCmklOtN9m0dTd2AOXqSneJYLw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>博弈</span>
  </div>
  <div class="_2_QraFYR_0">师兄的课程，必须支持</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: OMG，买它~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-21 22:52:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/ec/2e/49d13bd2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>SMTCode</span>
  </div>
  <div class="_2_QraFYR_0">云原生的Noops确实是未来发展方向，但并不是说有了noops，程序员就轻松了，只有了解整体的项目流程和之中涉及的技术过程，才能更好的使用好noops。而且现在函数计算等，适用面还窄，存在很大局限性。云厂商实际想把云技术做成像水电一般，信息技术中的社会基础资源，所有人都不得不用，这样可以躺着赚钱了，也掌握了定价权。有利有弊，但方向难阻挡，就算团队实力强悍，自己能坑哧瘪肚实现所有用到的基础服务，需要的开发和维护时间比用云技术高几百倍，再考虑到各种中间件、基础服务的复杂性，简直没办法和云厂商竞争，如果真有这样的技术团队，必然会被云厂商想尽办法招致麾下了。只要价格足够有诱惑力，公有云的优势，云厂商完全可以在私有云中一一实现。缺点也是像丰巢一样的对云厂商存在依赖。优势与不足很明显，但势不可挡，只能顺势而为，在未来的不断碰撞中，一切都会达到一个各方都适应的平衡。云平台，从技术、成本、人才等各种角度看，只有真正的大厂才玩得起，我们跟着分羹一杯也不错，避免重复性非必要劳动，省出时间才是最值钱的。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-16 05:31:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/ac/5d/7cfded43.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>铁生</span>
  </div>
  <div class="_2_QraFYR_0">感觉更适合前端工程师，后端适合吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 后端同学我觉得也适合，Serverless你可以随时随地调用一个云端函数。对前端同学来说，降低了服务端运维的门槛；对后端同学来说，则是掌握一种新的云计算服务。后面的课程会讲到微服务。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-16 09:33:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/9b/c7/0f3b699f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>朋克是夏天的冰镇雪碧</span>
  </div>
  <div class="_2_QraFYR_0">我最近在找工作，写了两个应用当面试作品。本来有台服务器把这俩应用放上去了，结果总被黑客攻击……我也是个运维小白，网上能查到的方法都用了，还是总被删库😂然后就想了解一下 serverless，正好极客时间推出了这门课程，真是太及时了哈哈，期待老师的更新，也希望自己求职成功</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 学习Serverless可以帮助你了解到后端服务器运维的知识，理清数据请求全链路的思路。祝你早日求职成功。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-15 17:55:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/12/13/e103a6e3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>扩散性百万咸面包</span>
  </div>
  <div class="_2_QraFYR_0">微服务用Node.JS做的多吗？感觉一般都是用后端语言做的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 据我所知的目前不多。不过微服务和Serverless一样，都是语言无关的，面向接口编程。你用Node.js开发也可以和后端其他语言开发的微服务通信的。主要看你熟悉的语言。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-15 17:39:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/45/28/16360d2e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>赵孔磊</span>
  </div>
  <div class="_2_QraFYR_0">让前端做后端的业务工作，但业务的工作也不能涉及太难，不然前端也无法实现。比如只能通过 serverless完成，数据查场景工作，以及不重要的业务。那这样前端只不过是，承担了一些后端工程师不愿意的工作，那对前端有什么成长吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这取决于怎么定义前端工程师，例如为什么你觉得前端工程师无法实现复杂的后端业务呢？或者有没有具体的业务场景？<br>我认识很多前端团队的做全栈，他们能力并不比后端弱，同样也可以实现复杂业务并部署线上。例如teambition团队，淘宝前端团队等等。<br>另外对于纯前端团队，现在越来越多研发团队采用微服务架构，前端团队可以用serverless做服务编排，降低联调和沟通成本。<br>或者你反过来想想，serverless如果只是想你描述的这样解决简单的问题，各大云厂商就不会如此重视它了。<br>serverless只不过目前因为node.js的JIT特性，前端工程师比较多关注而已，它实际上应用场景非常广泛。各种云服务商提供的事件触发，可以实现各种异步任务。例如用户上传一个视频后，faas自动转码，转码成功还可以写入db。还有定时自动触发ML算法，跑tensorflow的模型。等等。<br>我并不想讲太多场景，因为那只会限制你的想象力。具体各种serverless用法，需要切身体验。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-17 17:11:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/0c/30/bb4bfe9d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lyonger</span>
  </div>
  <div class="_2_QraFYR_0">老师，请问下severless对运维工程师有什么机遇和挑战吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对运维工程师，多掌握一种事件响应的解决方案。<br>而且FC是目前云服务里面性价比最高的服务，结合运维工程师现有的各种工具，也可以组合变化出各种高级工具。利用Serverless的思想，去设计运维架构，还可以减轻自己运维的压力。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-03 21:37:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/3b/03/9332db5a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>NSDGB</span>
  </div>
  <div class="_2_QraFYR_0">serverless给后端带来什么好处呢？ 能详细说一下嘛 </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 最直接的就是后端免运维，节省人员开销和服务器开销。让后端开发和运维的门槛降低。如果要详细解释，需要知道你具体的问题是什么？你所说的后端指的具体哪些事情？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-15 18:33:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/99/6b/77bb8501.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>丫头</span>
  </div>
  <div class="_2_QraFYR_0">开始前先介绍下Serverless是什么会不会好点？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: serverless的概念：无需关心服务器的云服务，但现如今已经被塞入了太多东西了，所以需要限定边界一步步讲。不知道听众的背景，不太好展开，因为毕竟不是面向零基础的知识。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-16 18:17:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/6f/8f/0e1fd90f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Josh</span>
  </div>
  <div class="_2_QraFYR_0">serverless感觉是让前端往原本后端和运维的领域各插了一脚，深深浅浅地让前端更加全栈，不过这更让我感兴趣的是其对现有开发模式的颠覆。希望通过本专栏的学习，能让我对serverless的实践有进一步的锻炼，和其能力边界有一个更深入的了解。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 最好的实践还是自己上手实操，配合课后的习题练习应该会让你对serverless演进有更深体会。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-16 10:59:53</div>
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
  <div class="_2_QraFYR_0">“Serverless 之前，这些中长尾应用至少要独占 1 台虚拟机；现在有了 Serverless 的极速冷启动特性，企业就可以节省这部分开销。”<br><br>这段话不是很理解，微服务流行以后，所有的服务以容器化的方式运行，每个虚拟机中可以同时跑多个服务，不应该还存在中长尾应用独占虚拟机的情况吧。<br><br>另外，要支持Serverless极速冷启动，需要提前设置好启动依赖的资源限制吗？例如CPU，内存等。<br><br>目前还没有对Serverless有一个整体的认知，希望后面的学习过程中能厘清这些疑问。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Serverless一大特点就是缩容到0，平时没有流量时是不占用任何资源，除了硬盘。<br>所以即使Serverless底层的容器方案，也可能是docker。但是却更加节省资源。<br>希望这节课的内容可以帮助你进一步理解Serverless。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-08 06:44:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/85/b0/496ae224.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Boogie 捷</span>
  </div>
  <div class="_2_QraFYR_0">哈哈哈, 电子科大2014毕业的学弟来报道</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 欢迎，欢迎~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-03 09:37:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/39/ac/d185d0e5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Quincy</span>
  </div>
  <div class="_2_QraFYR_0">这serverless 咋感觉像是后端用nodejs来写，前端全栈一样，类似之前的前后端不分离，java、python换成了nodejs</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Serverless架构中，后端同学应该去写BaaS微服务。前端同学可以自己写FaaS自由编排这些BaaS微服务。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-29 16:52:23</div>
  </div>
</div>
</div>
</li>
</ul>