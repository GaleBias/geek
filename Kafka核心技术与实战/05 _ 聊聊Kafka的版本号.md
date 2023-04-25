<audio title="05 _ 聊聊Kafka的版本号" src="https://static001.geekbang.org/resource/audio/0a/d9/0a2078eee0647452a6fe68e5861dead9.mp3" controls="controls"></audio> 
<p>你好，我是胡夕。今天我想和你聊聊如何选择Kafka版本号这个话题。今天要讨论的内容实在是太重要了，我觉得它甚至是你日后能否用好Kafka的关键。</p><p>上一期我介绍了目前流行的几种Kafka发行版，其实不论是哪种Kafka，本质上都内嵌了最核心的Apache Kafka，也就是社区版Kafka，那今天我们就来说说Apache Kafka版本号的问题。在开始之前，我想强调一下后面出现的所有“版本”这个词均表示Kafka具体的版本号，而非上一篇中的Kafka种类，这一点切记切记！</p><p>那么现在你可能会有这样的疑问：我为什么需要关心版本号的问题呢？直接使用最新版本不就好了吗？当然了，这的确是一种有效的选择版本的策略，但我想强调的是这种策略并非在任何场景下都适用。如果你不了解各个版本之间的差异和功能变化，你怎么能够准确地评判某Kafka版本是不是满足你的业务需求呢？因此在深入学习Kafka之前，花些时间搞明白版本演进，实际上是非常划算的一件事。</p><h2>Kafka版本命名</h2><p>当前Apache Kafka已经迭代到2.2版本，社区正在为2.3.0发版日期进行投票，相信2.3.0也会马上发布。但是稍微有些令人吃惊的是，很多人对于Kafka的版本命名理解存在歧义。比如我们在官网上下载Kafka时，会看到这样的版本：</p><!-- [[[read_end]]] --><p><img src="https://static001.geekbang.org/resource/image/c1/23/c10df9e6f72126e9c721fba38e27ac23.png?wh=784*208" alt=""></p><p>于是有些同学就会纳闷，难道Kafka版本号不是2.11或2.12吗？其实不然，前面的版本号是编译Kafka源代码的Scala编译器版本。Kafka服务器端的代码完全由Scala语言编写，Scala同时支持面向对象编程和函数式编程，用Scala写成的源代码编译之后也是普通的“.class”文件，因此我们说Scala是JVM系的语言，它的很多设计思想都是为人称道的。</p><p>事实上目前Java新推出的很多功能都是在不断向Scala语言靠近罢了，比如Lambda表达式、函数式接口、val变量等。一个有意思的事情是，Kafka新版客户端代码完全由Java语言编写，于是有些人展开了“Java VS Scala”的大讨论，并从语言特性的角度尝试分析Kafka社区为什么放弃Scala转而使用Java重写客户端代码。其实事情远没有那么复杂，仅仅是因为社区来了一批Java程序员而已，而以前老的Scala程序员隐退罢了。可能有点跑题了，但不管怎样我依然建议你有空去学学Scala语言。</p><p>回到刚才的版本号讨论。现在你应该知道了对于kafka-2.11-2.1.1的提法，真正的Kafka版本号实际上是2.1.1。那么这个2.1.1又表示什么呢？前面的2表示大版本号，即Major Version；中间的1表示小版本号或次版本号，即Minor Version；最后的1表示修订版本号，也就是Patch号。Kafka社区在发布1.0.0版本后特意写过一篇文章，宣布Kafka版本命名规则正式从4位演进到3位，比如0.11.0.0版本就是4位版本号。</p><p>坦率说，这里我和社区的意见是有点不同的。在我看来像0.11.0.0这样的版本虽然有4位版本号，但其实它的大版本是0.11，而不是0，所以如果这样来看的话Kafka版本号从来都是由3个部分构成，即“大版本号 - 小版本号 - Patch号”。这种视角可以统一所有的Kafka版本命名，也方便我们日后的讨论。我们来复习一下，假设碰到的Kafka版本是0.10.2.2，你现在就知道了它的大版本是0.10，小版本是2，总共打了两个大的补丁，Patch号是2。</p><h2>Kafka版本演进</h2><p>Kafka目前总共演进了7个大版本，分别是0.7、0.8、0.9、0.10、0.11、1.0和2.0，其中的小版本和Patch版本很多。哪些版本引入了哪些重大的功能改进？关于这个问题，我建议你最好能做到如数家珍，因为这样不仅令你在和别人交谈Kafka时显得很酷，而且如果你要向架构师转型或者已然是架构师，那么这些都是能够帮助你进行技术选型、架构评估的重要依据。</p><p>我们先从0.7版本说起，实际上也没什么可说的，这是最早开源时的“上古”版本了，以至于我也从来都没有接触过。这个版本只提供了最基础的消息队列功能，甚至连副本机制都没有，我实在想不出有什么理由你要使用这个版本，因此一旦有人向你推荐这个版本，果断走开就好了。</p><p>Kafka从0.7时代演进到0.8之后正式引入了<strong>副本机制</strong>，至此Kafka成为了一个真正意义上完备的分布式高可靠消息队列解决方案。有了副本备份机制，Kafka就能够比较好地做到消息无丢失。那时候生产和消费消息使用的还是老版本的客户端API，所谓的老版本是指当你用它们的API开发生产者和消费者应用时，你需要指定ZooKeeper的地址而非Broker的地址。</p><p>如果你现在尚不能理解这两者的区别也没关系，我会在专栏的后续文章中详细介绍它们。老版本客户端有很多的问题，特别是生产者API，它默认使用同步方式发送消息，可以想见其吞吐量一定不会太高。虽然它也支持异步的方式，但实际场景中可能会造成消息的丢失，因此0.8.2.0版本社区引入了<strong>新版本Producer API</strong>，即需要指定Broker地址的Producer。</p><p>据我所知，国内依然有少部分用户在使用0.8.1.1、0.8.2版本。<strong>我的建议是尽量使用比较新的版本。如果你不能升级大版本，我也建议你至少要升级到0.8.2.2这个版本，因为该版本中老版本消费者API是比较稳定的。另外即使你升到了0.8.2.2，也不要使用新版本Producer API，此时它的Bug还非常多。</strong></p><p>时间来到了2015年11月，社区正式发布了0.9.0.0版本。在我看来这是一个重量级的大版本更迭，0.9大版本增加了基础的安全认证/权限功能，同时使用Java重写了新版本消费者API，另外还引入了Kafka Connect组件用于实现高性能的数据抽取。如果这么多眼花缭乱的功能你一时无暇顾及，那么我希望你记住这个版本的另一个好处，那就是<strong>新版本Producer API在这个版本中算比较稳定了</strong>。如果你使用0.9作为线上环境不妨切换到新版本Producer，这是此版本一个不太为人所知的优势。但和0.8.2引入新API问题类似，不要使用新版本Consumer API，因为Bug超多的，绝对用到你崩溃。即使你反馈问题到社区，社区也不会管的，它会无脑地推荐你升级到新版本再试试，因此千万别用0.9的新版本Consumer API。对于国内一些使用比较老的CDH的创业公司，鉴于其内嵌的就是0.9版本，所以要格外注意这些问题。</p><p>0.10.0.0是里程碑式的大版本，因为该版本<strong>引入了Kafka Streams</strong>。从这个版本起，Kafka正式升级成分布式流处理平台，虽然此时的Kafka Streams还基本不能线上部署使用。0.10大版本包含两个小版本：0.10.1和0.10.2，它们的主要功能变更都是在Kafka Streams组件上。如果你把Kafka用作消息引擎，实际上该版本并没有太多的功能提升。不过在我的印象中自0.10.2.2版本起，新版本Consumer API算是比较稳定了。<strong>如果你依然在使用0.10大版本，我强烈建议你至少升级到0.10.2.2然后使用新版本Consumer API。还有个事情不得不提，0.10.2.2修复了一个可能导致Producer性能降低的Bug。基于性能的缘故你也应该升级到0.10.2.2。</strong></p><p>在2017年6月，社区发布了0.11.0.0版本，引入了两个重量级的功能变更：一个是提供幂等性Producer API以及事务（Transaction） API；另一个是对Kafka消息格式做了重构。</p><p>前一个好像更加吸引眼球一些，毕竟Producer实现幂等性以及支持事务都是Kafka实现流处理结果正确性的基石。没有它们，Kafka Streams在做流处理时无法向批处理那样保证结果的正确性。当然同样是由于刚推出，此时的事务API有一些Bug，不算十分稳定。另外事务API主要是为Kafka Streams应用服务的，实际使用场景中用户利用事务API自行编写程序的成功案例并不多见。</p><p>第二个重磅改进是消息格式的变化。虽然它对用户是透明的，但是它带来的深远影响将一直持续。因为格式变更引起消息格式转换而导致的性能问题在生产环境中屡见不鲜，所以你一定要谨慎对待0.11版本的这个变化。不得不说的是，这个版本中各个大功能组件都变得非常稳定了，国内该版本的用户也很多，应该算是目前最主流的版本之一了。也正是因为这个缘故，社区为0.11大版本特意推出了3个Patch版本，足见它的受欢迎程度。我的建议是，如果你对1.0版本是否适用于线上环境依然感到困惑，那么至少将你的环境升级到0.11.0.3，因为这个版本的消息引擎功能已经非常完善了。</p><p>最后我合并说下1.0和2.0版本吧，因为在我看来这两个大版本主要还是Kafka Streams的各种改进，在消息引擎方面并未引入太多的重大功能特性。Kafka Streams的确在这两个版本有着非常大的变化，也必须承认Kafka Streams目前依然还在积极地发展着。如果你是Kafka Streams的用户，至少选择2.0.0版本吧。</p><p>去年8月国外出了一本书叫Kafka Streams in Action（中文版：《Kafka Streams实战》），它是基于Kafka Streams 1.0版本撰写的。最近我用2.0版本去运行书中的例子，居然很多都已经无法编译了，足见两个版本变化之大。不过如果你在意的依然是消息引擎，那么这两个大版本都是适合于生产环境的。</p><p>最后还有个建议，不论你用的是哪个版本，都请尽量保持服务器端版本和客户端版本一致，否则你将损失很多Kafka为你提供的性能优化收益。</p><h2>小结</h2><p>我希望现在你对如何选择合适的Kafka版本能做到心中有数了。每个Kafka版本都有它恰当的使用场景和独特的优缺点，切记不要一味追求最新版本。事实上我周围的很多工程师都秉承这样的观念：不要成为最新版本的“小白鼠”。了解了各个版本的差异之后，我相信你一定能够根据自己的实际情况做出最正确的选择。</p><p><img src="https://static001.geekbang.org/resource/image/9b/d1/9be3f8c5c2930f6482fb43d8bca507d1.jpg?wh=2069*2560" alt=""></p><h2>开放讨论</h2><p>如何评估Kafka版本升级这件事呢？你和你所在的团队有什么独特的见解？</p><p>欢迎你写下自己的思考或疑问，我们一起讨论 。如果你觉得有所收获，也欢迎把文章分享给你的朋友。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/4d/87/57236a2d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>木卫六</span>
  </div>
  <div class="_2_QraFYR_0">版本号：<br>大 + 小 + patch<br><br>0.7版本:<br>只有基础消息队列功能，无副本；打死也不使用<br><br>0.8版本:<br>增加了副本机制，新的producer API；建议使用0.8.2.2版本；不建议使用0.8.2.0之后的producer API<br><br>0.9版本:<br>增加权限和认证，新的consumer API，Kafka Connect功能；不建议使用consumer API；<br><br>0.10版本:<br>引入Kafka Streams功能，bug修复；建议版本0.10.2.2；建议使用新版consumer API<br><br>0.11版本:<br>producer API幂等，事物API，消息格式重构；建议版本0.11.0.3；谨慎对待消息格式变化<br><br>1.0和2.0版本:<br>Kafka Streams改进；建议版本2.0；<br><br>江湖经验：不要成为最新版本的小白鼠<br><br>谢谢老师的讲解，收工。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-14 20:08:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/b6/48/1275e0ce.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小头针</span>
  </div>
  <div class="_2_QraFYR_0">胡老师讲的这个版本，一下戳到我的痛处。讲一下我在生产环境中遇到的Kafka版本带来的坑。我参与到项目中一年，运行的版本是0.10.0.1，前半年还算稳定，偶尔出现进程假死问题。但是慢慢的生产环境数据量增加，假死频发，导致客户数据丢失，问题很严重。但是一直又没有证据证明这个版本确实存在问题，虽然官网上有提到，但是我们老大的意思还是要找到根本原因。妥所以开始对生产环境的进程进程监控，确实监控出此版本存在线程死锁问题。（妥从一个开发转变为现场运维人员）然后研究官网bug列表，选出0.11.0.3这个版本，至今稳定运行。<br><br>再顺便讲一下选择这门课的原因，虽然项目已经高一段落，但是在整个项目过程中，一直处于哪里不会点哪里的状态，感觉一直还是没有真正的掀开Kafka的面纱。所以想系统的学习一下，听了几节课，之前有些知其然而不知其所以然的内容，似乎开始有点茅塞顿开了。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-13 22:26:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/7b/57/a9b04544.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>QQ怪</span>
  </div>
  <div class="_2_QraFYR_0">的确在工作中遇到了kafka版本不同导致消息格式不兼容问题，后来服务端和客户端统一版本号才解决，👍👍👍</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-13 08:21:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/55/99/4bdadfd3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Chloe</span>
  </div>
  <div class="_2_QraFYR_0">只想简单点赞👍，看了标题觉得版本号没有什么好聊了，看到才知道Kaka一路走来充满荆棘啊。感谢感谢！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-14 09:25:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/8f/35/f1839bb2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>风中花</span>
  </div>
  <div class="_2_QraFYR_0">每次看完就想打卡，以表示我还在坚持学习，如此看来，这个版本真不能忽视</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯嗯，之前在做咨询的时候发现过两个问题：1. 很多人碰到的问题实际上已经是新版本解决的bug；2. 客户端&#47;服务器端版本不一致导致的性能问题<br><br>因此觉得写一篇版本的文章还是有点必要的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-17 17:10:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/52/eb/eec719f3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>开水</span>
  </div>
  <div class="_2_QraFYR_0">目前用的是hdp2.4.2内嵌版本。应该是apache版本的0.8.2.0。遇到很多问题都很难找到解决方法。比如前几天遇到了replicaFetcherThread oom的问题，网上根本找不到什么正经的解释。但又不能一味的调高jvm参数。老大说现在生产稳定就行了，暂时不要升级了，改代码耗时切面对的问题未知。期待老大能看到这篇文章，升级到0.11.0.3。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯嗯，可以查一下ZooKeeper中是否存在大量session超时的情况。不过还是建议升级吧，听着很像是一个已知的bug。如果暂时不能升级，可以尝试调低replica.fetch.max.bytes的值试试。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-13 08:35:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/ab/82/b6af9f94.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>King Yao</span>
  </div>
  <div class="_2_QraFYR_0">我是一名运维，在维护工作环境维护几十台Kafka。最近打算扩容。我有三个问题请教：<br>1.kafka如何做压力测试，它的参考主要指标是什么，比如QPS,最大连接数，延迟等等。<br>2.扩容如何做到平滑扩容，不影响原业务<br>3.kafka有什么好的监控软件。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. Kafka提供了命令行脚本可以执行producer和consumer的性能测试，主要指标还是TPS，延时<br>2. 增加broker很简单，也不会对现有业务有影响。关键是做好迁移计划——比如避开业务高峰时刻，如果迁移对业务影响最小</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-13 23:35:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/8d/6f/c21eff20.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>平叔叔</span>
  </div>
  <div class="_2_QraFYR_0">不论你用的是哪个版本，都请尽量保持服务器端版本和客户端版本一致  另外即使你升到了 0.8.2.2，也不要使用新版本 Producer api 不是互相矛盾吗？<br><br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不矛盾。保持版本一致是任何时候都要尽力保持的。如果你依然使用0.8.x版本，那么最好使用Scala producer，而不是java producer</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-22 10:49:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/88/26/b8c53cee.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>南辕北辙</span>
  </div>
  <div class="_2_QraFYR_0">记得在刚开始学kafka写demo时，找到了kafka.producer.Producer,以及apache.kafka...KafkaProducer，还以为只是2种不同的实现方式，后来在老师的书上才得知这完全是2个版本。针对今天讨论的版本差异，书上也做了很好了总结。<br>现在看来比较难理解的就是客户端的版本与服务端版本的兼容问题，与之前的各种技术还是有点差异的，并不是一味的较新客户端就完事，kafka中还有一种请求版本号的存在。<br>图片为书中版本对比<br>https:&#47;&#47;raw.githubusercontent.com&#47;DarkerPWQ&#47;picgo&#47;master&#47;img&#47;Kafka%E7%89%88%E6%9C%AC%E5%8F%98%E8%BF%81.png</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯嗯，请求版本号偏底层的设计了，一般用户用不到。其实客户端和服务器端版本的差异很大一部分也是请求版本号的差异</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-13 09:52:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/11/7f/80d56c1c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>莫问流年</span>
  </div>
  <div class="_2_QraFYR_0">目前公司kafka都是由运维团队统一维护的，开发人员需要时向运维人员申请kafka服务器端实例。加上Spring Boot项目对Kafka客户端的自动配置机制，导致大部分开发人员对Kfka的版本缺乏足够的了解。这种对开发人员完全透明的方式，虽然一定程度上减轻了开发人员的负担，但是也为开发人员深入理解Kafka设置了一定障碍。作为开发团的一员，我希望结合老师的课程和Kafka在公司项目的实践，更深入的理解Kafka的核心原理和设计思想。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-13 08:50:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/0d/b7/86d4f621.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>少林寺三毛</span>
  </div>
  <div class="_2_QraFYR_0">👍</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-13 07:19:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c6/73/abb7bfe3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>疯琴</span>
  </div>
  <div class="_2_QraFYR_0">美音最正的老师没有之一。有两个问题：1 文章前面说某些旧版本的服务端不适用新版本的客户端，文末又说二者应该保持一致，感觉是矛盾的，应该是我没理解清楚，还请说明 2 服务端版本靠编号就可以识别，而客户端是怎么定义新旧版本的呢？谢谢。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. 其实我也没太明白您的意思：） “旧版本的服务端不适用新版本的客户端 所以建议保持一致”，不是挺自然的结论吗。。。。 <br><br>2. 客户端也有版本号，和broker端是一样的。比如Java客户端，如果我们使用Gradle的话，就类似于这样：<br><br>compile group: &#39;org.apache.kafka&#39;, name: &#39;kafka-clients&#39;, version: &#39;2.2.1&#39;<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-13 08:22:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/b2/c5/6ae0be56.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>木偶人King</span>
  </div>
  <div class="_2_QraFYR_0">找到0.10.2.2 和0.11.0.3<br>http:&#47;&#47;mirrors.hust.edu.cn&#47;apache&#47;kafka&#47;</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-03 14:06:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/05/b2/2ff81f75.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>bee</span>
  </div>
  <div class="_2_QraFYR_0">请问胡老师， RESTful的幂等性是无论调用多少次都不会有不同结果的HTTP 方法。也就是说不会去改变资源。这里的幂等性虽然是确保了单个partition的数据不会重复，但是更改了资源。这样说会有冲突吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这里的幂等性也没有你所谓的更改资源。当producer发送了一条重复消息时Broker端会拒绝接收，而不是在后面自己做去重</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-13 18:49:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/PiajxSqBRaEIaTvOKvUt4WnuSjkBp0tjd6O6vvVyw5fcib3UgZibE8tz2ICbTfkwbzs8MHNMJjV6W2mLjywLsvBibg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>火力全开</span>
  </div>
  <div class="_2_QraFYR_0">“最后还有个建议，不论你用的是哪个版本，都请尽量保持服务器端版本和客户端版本一致，否则你将损失很多 Kafka 为你提供的性能优化收益。”  ——  KIP-35 - Retrieving protocol version 这个特性不是可以让支持的client适配多个不同版本的broker吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 兼容性是一方面。因版本差异导致消息格式转换会丧失Zero Copy</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-14 16:21:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/3d/5d/ac666969.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>miwucc</span>
  </div>
  <div class="_2_QraFYR_0">版本号坑多的很啊，比如心跳是否采用业务线程，是否支持发送者ack机制啊等等，坑了我很多当时。。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-14 12:55:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/b7/f9/a8f26b10.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jacke</span>
  </div>
  <div class="_2_QraFYR_0">胡老师，问下：<br>&quot;如果你不能升级大版本，我也建议你至少要升级到0.8.2.2+这个版本，因为该版本中老版本消费者API是比较稳定的。另外即使你升到了0.8.2.2，也不要使用新版本+Producer+API，此时它的Bug还非常多&quot; == 不是很明白<br>这段话的意思升级到0.8.2.2，使用0.8.2.2的consumer api, 但是不要0.8.2.2的 producer api? 使用0.8大版本号下的producer api 是这个意思吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果使用0.8.2.2，那么使用老版本的consumer和producer，不要使用新版本的客户端。它们此时还有很多bug。我是这个意思。。。。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-22 14:15:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/67/c5/63b09189.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>刘朋</span>
  </div>
  <div class="_2_QraFYR_0">升级Kafka版本,<br>首先，看业务在使用当前Kafka版本是否有问题,是否有性能问题,<br>其次，当前版本特性是否满足业务需求,是否需要新的Kafka特性<br>然后，查看该当前版本是否还在迭代更新,以及迭代周期<br>最后，升级Kafka版开发人员所付出的人工成本和时间成本<br><br>在升级版本时,不是一味的追求最新版本,而是在满足业务需求为前提条件下的还在社区维护更新的稳定版本.</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-13 08:56:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/cf/ad/c639270f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>云＆龙</span>
  </div>
  <div class="_2_QraFYR_0">既然新版本的老功能更加完善，而老版本又没有新版本的新功能，那为什么不无脑用最新的版本呢？？？反正用老版本也没有新功能。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果是新环境，也许可以采用最新版本。但如果是要升级的话就要考虑这些问题了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-13 07:51:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/4c/6c/11fb0f1d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>布兰特</span>
  </div>
  <div class="_2_QraFYR_0">胡老师，如果使用kafka 0.11, Flink 1.10版本的前提下， kafka -&gt; Flink -&gt; kafka stream -&gt; HBase 这样的一个数据流向，是不是可以实现 exactly once ??</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Flink -&gt; Kafka Streams? 一般没有这样的通路吧，另外Kafka Streams也没法到HBase</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-01 11:25:53</div>
  </div>
</div>
</div>
</li>
</ul>