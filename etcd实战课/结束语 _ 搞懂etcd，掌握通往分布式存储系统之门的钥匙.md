<audio title="结束语 _ 搞懂etcd，掌握通往分布式存储系统之门的钥匙" src="https://static001.geekbang.org/resource/audio/38/65/3816f8492c18d8d7e3e9371d74235565.mp3" controls="controls"></audio> 
<p>你好，我是唐聪。</p><p>时间过得真快，这就到了我们的定期更新的最后一节课了。从筹备、上线到今天专栏完结，过去了将近7个多月的时间。</p><p>说句实在话，刚开始筹备专栏的时候，我没想过战线会拉得如此之长。当时就是简单地觉得，我的经验也比较丰富了，输出应该很简单。但是其实做专栏耗费的心力远超我的预期：每一节课的构思写作都会花费我大量的时间，而且写完后还得考虑文章逻辑是否有优化的空间，怎样加配图、加一个什么样的配图可以更加形象，甚至部分文章写完自己不满意我还会重写一遍。</p><p>细心的你应该能发现，其实这个专栏每一节课的内容都是比较多的。一开始的筹划是每篇文章3500字左右，但最后为了讲清楚、讲明白，每一节课大部分都是到了6000字到7000字的内容（有的文章字数是破万了）。在此特别感谢我的“好基友”王超凡非常用心地和我一块深度review每一篇文章，因为平时工作也很忙，还经常得封闭式开发，所以录音只能放在凌晨。</p><p>在这里和你分享一件有意思的小事，专栏上线的前一天凌晨，我们和编辑正霖都激动得睡不着，在群里预览文章，聊上线后会是怎么样的一番景象。我们甚至想，会不会上线后被各位疯狂吐槽，以至于不得不录一个“负荆请罪”视频。</p><p><img src="https://static001.geekbang.org/resource/image/45/c8/452c5eeed7d79d3cba7a145ae67f57c8.jpg?wh=1080*1642" alt=""></p><p>现在回想起来，真的是做好了被大家吐槽的准备。但你们给我的是超出预期的热情。不少同学从上线到结束，都在时刻关注、学习每一节课，并留下优质的提问以及鼓励、认可。</p><!-- [[[read_end]]] --><ul>
<li>有的同学是比较资深的etcd使用者，会独立分析源码，撰写高质量的技术博客，并给出精彩的回答；</li>
<li>有的同学是刨根问底的etcd兴趣用户，会细致思考每一个异常场景，给出精彩的提问；</li>
<li>有的同学刚刚入门etcd用户，正因为你们的提问，让我意识到需要在基础篇中多去增加一些特性初体验的案例；</li>
<li>还有的同学着急说面试要用，所以春节期间我们没有筹划春节特别活动，而是正常更新课程正文；</li>
<li>……</li>
</ul><p>当然在这过程中，我也收获满满。为了解答你们的疑问，我必须得更加深入地阅读etcd源码，也是倒逼着我去进一步成长。</p><p>编辑正霖半开玩笑地和我说，我们是以百米冲刺的速度去跑马拉松。这段经历真的很难忘，你们的评论和收藏证明了我们的付出是值得的。</p><p>在这最后一节课里，我想最后和你再分享下我个人的etcd学习经验，以及这整个专栏设计和写作思路。</p><p><strong>如果要用一个核心词来总结这个专栏，那我希望是问题及任务式驱动。</strong></p><p>从我的个人经验上来看，我每次进一步学习etcd的动力，其实都是源于某个棘手的问题。数据不一致、死锁等一系列棘手问题，它们会倒逼我走出舒适区，实现更进一步成长。</p><p>从专栏目录中你也可以看到，每讲都是围绕着一个<strong>问题</strong>深入展开。在具体写作思路上，我会先从整体上给你介绍整体架构、解决方案，让你有个全局的认识。随后围绕每个点，按照由浅入深的思路给你分析各种解决方案。</p><p>另外，<strong>任务式驱动</strong>也是激励你不断学习的一个非常好的手段，通过任务实践你可以获得满满的成就感，建立正向反馈。你在学习etcd专栏的过程中，可结合自己的实际情况，为自己设立几个进阶任务，下面我给你列举了部分：</p><ul>
<li>从0到1搭建一个etcd集群（可以先从单节点再到多节点，并进行节点的增删操作）；</li>
<li>业务应用中使用etcd的核心API；</li>
<li>自己动手实现一个分布式锁的包；</li>
<li>阅读etcd的源码，写篇源码分析博客 （可从早期的etcd v2开始）；</li>
<li>基于raftexample实现一个支持多存储引擎的KV服务；</li>
<li>基于Kubernetes的Operator机制，实现一个etcd operator，创建一个CRD资源就可新建一个集群；</li>
<li>……</li>
</ul><p>我希望带给你的不仅仅是etcd原理与实践案例，更希望你收获的是一系列分布式核心问题解决方案，它们不仅能帮助你搞懂etcd背后的设计思想与实现，更像是一把通往分布式存储系统之门的钥匙，让你更轻松地学习、理解其他存储系统。</p><p>那你可能会问了，为什么搞懂etcd就能更深入理解分布式存储系统呢？</p><p>因为etcd相比其他分布式系统如HBase等，它足够简洁、轻量级，又涵盖了分布式系统常见的问题和核心概念，如API、数据模型、共识算法、存储引擎、事务、快照、WAL等，非常适合新人去学习。</p><p><img src="https://static001.geekbang.org/resource/image/7b/28/7b54b6ca9134c130bf3940c7db497928.png?wh=1920*1177" alt=""></p><p>上图我为你总结了etcd以及其他分布式系统的核心技术点，下面我再和你简要分析一下几个分布式核心问题及解决方案，并以Redis Cluster集群模式作为对比案例，希望能够帮助你触类旁通。</p><p>首先是服务可用性问题。分布式存储系统的解决方案是共识算法、复制模型。etcd使用的是Raft共识算法，一个写请求至少要一半以上节点确认才能成功，可容忍少数节点故障，具备高可用、强一致的目标。<a href="https://redis.io/topics/cluster-spec">Redis Cluster</a>则使用的是主备异步复制和Gossip协议，基于主备异步复制协议，可将数据同步到多个节点，实现高可用。同时，通过Gossip协议发现集群中的其他节点、传递集群分片等元数据信息等操作，不依赖于元数据存储组件，实现了去中心化，降低了集群运维的复杂度。</p><p>然后是数据如何存取的问题。分布式存储系统的解决方案是存储引擎。除了etcd使用的boltdb，常见的存储引擎还有我们实践篇<a href="https://time.geekbang.org/column/article/347136">18</a>中所介绍bitcask、leveldb、rocksdb（leveldb优化版）等。不同的分布式存储系统在面对不同业务痛点时（读写频率、是否支持事务等），所选择的解决方案不一样。etcd v2使用的是内存tree，etcd v3则使用的是boltdb，而Redis Cluster则使用的是基于内存实现的各类数据结构。</p><p>最后是如何存储大量数据的问题。分布式存储系统的解决方案是一系列的分片算法。etcd定位是个小型的分布式协调服务、元数据存储服务，因此etcd v2和etcd v3都不支持分片，每个节点含有全量的key-value数据。而Redis Cluster定位是个分布式、大容量的分布式缓存解决方案，因此它必须要使用分片机制，将数据打散在各个节点上。目前Redis Cluster使用的分片算法是哈希槽，它根据你请求的key，基于crc16哈希算法计算slot值，每个slot分配给对应的node节点处理。</p><pre><code>HASH_SLOT = CRC16(key) mod 16384
</code></pre><p>etcd 作为最热门的云原生存储之一，在腾讯、阿里、Google、AWS、美团、字节跳动、拼多多、Shopee、明源云等公司都有大量的应用，覆盖的业务不仅仅是 Kubernetes 相关的各类容器产品，更有视频、推荐、安全、游戏、存储、集群调度等核心业务。</p><p>更快、更稳是etcd未来继续追求的方向，etcd社区将紧密围绕Kubernetes社区做一系列的优化工作，提供集群降级、自动将Non-Voting的Learner节点提升为Voting Member等特性，彻底解决饱受开发者诟病的版本管理等问题。</p><p>希望这个专栏一方面能帮助你遵循最佳实践，高效解决核心业务中各类痛点问题，另一方面能轻松帮你搞定面试过程中常见etcd问题，拿到满意的offer。</p><p>当然，我发现很多同学只是默默地收藏，一直在“潜水”。我希望在这最后一课里，大家一块来“灌灌水”，分享一下你自己的etcd学习方法以及你对这门课的感受。我为你准备了一份<a href="https://jinshuju.net/f/sz6QOc">问卷</a>，希望你花两分钟填一下，说不定你就是我们这门课的“小锦鲤”~</p><p>最后，再次感谢，我们留言区和加餐见！</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/9d/84/171b2221.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jeffery</span>
  </div>
  <div class="_2_QraFYR_0">真不舍！太清爽了！希望加餐！😀要求是不是太过分了😀多谢唐老师三个月份的付出分享！确实牛</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢jeffery同学认可，短暂休息下，近期先推出答疑文章，随后再来加餐，查漏补缺，兼顾各个层次的同学，帮助大家更好学习专栏内容</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-22 08:04:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/a2/94/ae0a60d8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>江山未</span>
  </div>
  <div class="_2_QraFYR_0">感谢老师的精彩讲解，在地铁上刷完了课程。师傅领进门，修行在个人。对我来说还有很多有待深挖的点和不清楚的地方，下一步开始慢慢看源码。希望和大家共同进步！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢江山未同学的支持与认可！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-28 09:06:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_daf51a</span>
  </div>
  <div class="_2_QraFYR_0">辛苦老师，从专栏中学到了很多，有etcd历史、有原理、有重点问题案例、有定位延时、内存等问题方法以及最佳实践、还有有kubernetes、配置服务发现、分布式锁等剖析，过程中大量配图总结，覆盖了etcd方方面面，能感受到老师的用心与认真，值了，期待后面的加餐</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢支持！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-22 07:59:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/ac/50/71c25e34.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>赵先生</span>
  </div>
  <div class="_2_QraFYR_0">唐老师，etcd 3.5来了，希望加餐！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢支持，昨天我第一时间在公众号上详细解读了etcd 3.5, https:&#47;&#47;mp.weixin.qq.com&#47;s&#47;mhHPyCAmdbT5wXF0vBiGzQ，后面将继续优化下加餐到专栏中。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-17 11:21:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/be/1e/94886200.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小鱼儿吐泡泡</span>
  </div>
  <div class="_2_QraFYR_0">专栏质量确实很高，从整体到细节 读起来很舒服；非常符合学习一个新知识得过程； 结合实践案例，收货颇深； 同时，想问下，唐老师，可以把课后问题得答案补充下吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 好的，感谢支持，最近项目和家庭事情比较多，课后答案我将尽快补充（预计9月份）。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-27 11:03:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/f0/27/6679da14.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>程序员历小冰</span>
  </div>
  <div class="_2_QraFYR_0">老师学习和讲解etcd的思路很赞，可以用在大多数中间件的学习上.</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢你的认可与支持</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-26 21:11:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJiaeTzf4V8ib4xKcYjWEIflBSqkjbpkscoaedppgnBAD9ZAibjYSz0DNSJQw8icz7xljEgbNQ5hrzPAA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>liudu_ec</span>
  </div>
  <div class="_2_QraFYR_0">终于看完了，意犹未尽</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞，能坚持这么快看完，了不起</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-24 08:58:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/61/e6/fedd20dc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>mmm</span>
  </div>
  <div class="_2_QraFYR_0">感谢老师 期待加餐</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-22 23:41:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c3/09/dc368335.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>杰sir</span>
  </div>
  <div class="_2_QraFYR_0">两周看看这个系列的文章，收获很大。包括etcd读写流程的梳理，各个模块的分析，尤其是boltd的剖析，我再其他etcd相关的文章里几乎没有看到有这块的解读，还有运维层面的总结，使用过程碰到的问题和相关的优化，以及对“一致性”这个中文词语在计算里领域里不同场景的解读（这个也是我到目前为止让我觉得解释的最清楚的）都让我读的大呼过瘾，后面自己有问题看看源码基本都可以搞定了。<br><br>然后感觉老师写技术文章的能力也可以多学学，其中的思路，框架也可以在自己向别人输出自己所学的时候用到。<br><br>最后催更一下思考题的答疑，啥时候更新啊。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢支持，最近在忙arch summit&#47;kubecon&#47;kstone开源各种事情耽误了答疑加餐篇更新，不过这些事情也都是对外的，相信能给大家工作带来帮助，尤其是我们的开源项目kstone, 它是业界首个etcd治理项目，后面我也会在加餐篇中详细介绍，这里先给你一些资料。<br>kstone开源项目，一站式etcd治理平台, 开源项目地址如下<br>https:&#47;&#47;github.com&#47;tkestack&#47;kstone<br>kubecon分享，如何管理数以万计的etcd集群, 分享pdf如下<br>https:&#47;&#47;static.sched.com&#47;hosted_files&#47;kccncosschn21&#47;9e&#47;KubeCon-China-2021-How-to-Efficiently-Manage-Tens-of-Thousands-of-etcd-Clusters.pdf<br>arch summit架构师大会腾讯大规模云原生平台稳定性实践, pdf如下<br>https:&#47;&#47;static001.geekbang.org&#47;con&#47;85&#47;pdf&#47;1890348645&#47;file&#47;%E8%85%BE%E8%AE%AF%E5%A4%A7%E8%A7%84%E6%A8%A1%E4%BA%91%E5%8E%9F%E7%94%9F%E5%B9%B3%E5%8F%B0%E7%A8%B3%E5%AE%9A%E6%80%A7%E5%AE%9E%E8%B7%B5-%E5%94%90%E8%81%AA.pdf</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-15 19:51:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/b4/94/2796de72.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>追风筝的人</span>
  </div>
  <div class="_2_QraFYR_0">老师 什么工具或命令 可以 查看etcd server   RPC调用速率  和磁盘平均延时？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: etcd metrics自带了这些指标，推荐体验下我们开源的一站式etcd治理平台kstone,https:&#47;&#47;github.com&#47;tkestack&#47;kstone, 它含有丰富的功能</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-15 09:57:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/3a/27/5d218272.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>八台上</span>
  </div>
  <div class="_2_QraFYR_0">坐等思考题答案</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 抱歉，最近忙各种事情耽误了，其中最大的一个事情就是推出了首个开源的etcd治理项目kstone<br>https:&#47;&#47;github.com&#47;tkestack&#47;kstone</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-29 10:12:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/be/1f/57c88dd1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小丢👣</span>
  </div>
  <div class="_2_QraFYR_0">对etcd的应用实践，我几乎为零，看了一遍专栏过后，我觉得我了解的差不多了。在准备给部门小伙伴做一次系列分享时，我发现我懂得太少了，然后我把几篇基础篇至少来回看了5遍，才有勇气分享给小伙伴。当然也仅仅是入门，只是想说一下这就是任务式驱动吧，没有分享的任务，可能就不会去思考各种细节 ，也不会在短时间内对etcd认知有一个较大的提升。感谢老师的精彩讲解，这个专栏我还是会反复看的，太值了！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢小丢同学的支持与认可! 一起加油!</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-06 16:55:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_7154d8</span>
  </div>
  <div class="_2_QraFYR_0">老师讲的太精彩了，比如分布式锁那一章，令我对分布式锁的技术要点顿时茅塞大开。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢支持!</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-14 11:37:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/30/85/14c2f16c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>石小</span>
  </div>
  <div class="_2_QraFYR_0">我问的不太明白哈😄，是这样的：mysql可以控制刷日志（通过innodb_flush_log_at_trx_commit），通过这个日志能控制刷磁盘，也会影响到事务提交成败。etcd有类似这样控制日志刷新的控制吗？还是说etcd每次提交必须刷磁盘（写wal日志），保证了事务一定会提交成功（即使boltdb写入失败）,如果是这样的，问什么etcd要这样设计？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，etcd WAL一定要刷盘的，因为etcd定位是高可靠的元数据存储、关键协调服务，它需要保证数据一致性、可靠性</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-12 10:19:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/22/3b/46/3701e908.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Coder</span>
  </div>
  <div class="_2_QraFYR_0">辛苦唐老师，收获满满！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-22 10:28:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>川川</span>
  </div>
  <div class="_2_QraFYR_0">老师，你微信多少</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-20 01:33:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/L19sKxTMAPrPpHibMribicRB2o9LVknHc4GI6icyialvickFsuakY1dlF9SzLnUW4qu4sT6KlVWtZJUYGYYE6fqaBsyA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>幽幽夜</span>
  </div>
  <div class="_2_QraFYR_0">期待加餐，催更思考题...</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-01 14:08:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2a/03/ef/d2fd9d36.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>愤叔</span>
  </div>
  <div class="_2_QraFYR_0">这些课 ，有没有pdf 数据或者文档可以下载下来 </div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-09 21:45:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/26/ba/92f91744.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>吃完了饭的人</span>
  </div>
  <div class="_2_QraFYR_0">潜水完，冒个泡，反反复复看了两三遍</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞，若有时间，最好结合结束语中给的一些任务，多实践，理解更深</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-31 14:26:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/22/3a/de/e5c30589.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>云原生工程师</span>
  </div>
  <div class="_2_QraFYR_0">感谢老师，期待加餐</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-23 08:00:57</div>
  </div>
</div>
</div>
</li>
</ul>