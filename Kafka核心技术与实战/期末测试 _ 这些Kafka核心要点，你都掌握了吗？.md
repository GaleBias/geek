<p>你好，我是胡夕。</p><p>《Kafka核心技术与实战》已经结课一段时间了，你掌握得怎么样了呢？我给你准备了一个结课小测试，来帮助你检验自己的学习效果。</p><p>这套测试题有选择题和简答题两种形式。选择题共有 20 道题目，考题范围覆盖专栏的 42 讲正文，题目类型为单选题和多选题，满分 100 分，系统自动评分。简答题共有5道，建议你拿出纸笔，写下你的思考和答案，然后再和文末的答案进行对照。</p><p>还等什么，点击下面按钮开始测试吧！</p><p><a href="http://time.geekbang.org/quiz/intro?act_id=94&exam_id=190"><img src="https://static001.geekbang.org/resource/image/28/a4/28d1be62669b4f3cc01c36466bf811a4.png?wh=1142*201" alt=""></a></p><h2>简答题</h2><ol>
<li>如果副本长时间不在ISR中，这说明什么？</li>
<li>谈一谈Kafka Producer的acks参数的作用。</li>
<li>Kafka中有哪些重要组件?</li>
<li>简单描述一下消费者组（Consumer Group）。</li>
<li>Kafka为什么不像Redis和MySQL那样支持读写分离？</li>
</ol><h2>答案与解析</h2><p><span class="orange">1.如果副本长时间不在ISR中，这说明什么？</span></p><p><strong>答案与解析：</strong></p><p>如果副本长时间不在ISR中，这表示Follower副本无法及时跟上Leader副本的进度。通常情况下，你需要查看Follower副本所在的Broker与Leader副本的连接情况以及Follower副本所在Broker上的负载情况。</p><p><span class="orange">2.请你谈一谈Kafka Producer的acks参数的作用。</span></p><p><strong>答案与解析：</strong></p><!-- [[[read_end]]] --><p>目前，acks参数有三个取值：0、1和-1（也可以表示成all）。</p><p>0表示Producer不会等待Broker端对消息写入的应答。这个取值对应的Producer延迟最低，但是存在极大的丢数据的可能性。</p><p>1表示Producer等待Leader副本所在Broker对消息写入的应答。在这种情况下，只要Leader副本数据不丢失，消息就不会丢失，否则依然有丢失数据的可能。</p><p>-1表示Producer会等待ISR中所有副本所在Broker对消息写入的应答。这是最强的消息持久化保障。</p><p><span class="orange">3.Kafka中有哪些重要组件?</span></p><p><strong>答案与解析：</strong></p><ul>
<li>Broker——Kafka服务器，负责各类RPC请求的处理以及消息的持久化。</li>
<li>生产者——负责向Kafka集群生产消息。</li>
<li>消费者——负责从Kafka集群消费消息。</li>
<li>主题——保存消息的逻辑容器，生产者发送的每条消息都会被发送到某个主题上。</li>
</ul><p><span class="orange">4.简单描述一下消费者组（Consumer Group）。</span></p><p><strong>答案与解析：</strong></p><p>消费者组是Kafka提供的可扩展且具有容错性的消费者机制。同一个组内包含若干个消费者或消费者实例（Consumer Instance），它们共享一个公共的ID，即Group ID。组内的所有消费者协调在一起来消费订阅主题的所有分区。每个分区只能由同一个消费组内的一个消费者实例来消费。</p><p><span class="orange">5.Kafka为什么不像Redis和MySQL那样支持读写分离？</span></p><p><strong>答案与解析：</strong></p><p>第一，这和它们的使用场景有关。对于那种读操作很多而写操作相对不频繁的负载类型而言，采用读写分离是非常不错的方案——我们可以添加很多Follower横向扩展，提升读操作性能。反观Kafka，它的主要场景还是在消息引擎，而不是以数据存储的方式对外提供读服务，通常涉及频繁地生产消息和消费消息，这不属于典型的读多写少场景。因此，读写分离方案在这个场景下并不太适合。</p><p>第二，Kafka副本机制使用的是异步消息拉取，因此存在Leader和Follower之间的不一致性。如果要采用读写分离，必然要处理副本滞后引入的一致性问题，比如如何实现Read-your-writes、如何保证单调读（Monotonic Reads）以及处理消息因果顺序颠倒的问题。相反，如果不采用读写分离，所有客户端读写请求都只在Leader上处理，也就没有这些问题了。当然，最后的全局消息顺序颠倒的问题在Kafka中依然存在，常见的解决办法是使用单分区，其他的方案还有Version Vector，但是目前Kafka没有提供。</p>
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
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/qsmAdOC3R3twep9xwiboiaNF6u3fk5jNZGibKrBuILKgyNMH0DAQMg3liaWQ7ntVAFGEBCg5uB9y9KdKrhD65TyGgQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>镜子中间</span>
  </div>
  <div class="_2_QraFYR_0">终于看完了Kafka课程，这是我在极客时间学完的第一门课，也是坚持的最久的，期末测试得了80分，仍有进步的空间，感谢胡夕老师，也感谢坚持下来的自己，Mark一下，准备开始二刷了！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯嗯，谢谢你的鼓励。其实，专栏期间我自己也对Kafka有了更进一步的了解，算是额外的收获了：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-20 15:55:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/57/1b/37a93b76.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>剑锋所指</span>
  </div>
  <div class="_2_QraFYR_0">学完打卡 90分</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-03 10:12:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/93/ce/092acd6a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>孙同学</span>
  </div>
  <div class="_2_QraFYR_0">做完题 85 哈哈 以为自己都忘了。。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-10 00:05:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1c/4e/88/791d0f5e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lei</span>
  </div>
  <div class="_2_QraFYR_0">胡老师，学完源码课然后来的实战课，收获很大。有没有好的讲解分布式的资料呢，能够综合一些学习</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 个人推荐阅读下《Designing Data-Intensive Applications》，然后根据自己的兴趣决定深入学习分布式系统哪个部分</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-23 09:05:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLRKgow8PPLLgCqt6ZWiaylrFG1ButvHRSqOZ8hvFZFHoUGaoYCLlasbRiaaM0pTWKeeLbW4xBM4vjg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>执芳之手</span>
  </div>
  <div class="_2_QraFYR_0">老师，你好。我的服务运行了一段时间，发送消息报错：Cannot perform operation after producer has been closed。网上说是，KafkaProducer 已经close了。但是，我不知道为什么被关闭的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以贴一贴代码吗？我看下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-14 11:45:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/3c/09/b7f0eac6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>谁都会变</span>
  </div>
  <div class="_2_QraFYR_0">kafka不同读写分离，互相主副本野是一个原因吧。它的读写压力本身就是分散的。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-24 11:11:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>杨栋</span>
  </div>
  <div class="_2_QraFYR_0">为什么我学完了进度还是显示97% </div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-11 15:45:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>杨栋</span>
  </div>
  <div class="_2_QraFYR_0">很有深度 </div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-11 15:42:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/26/eb/d7/90391376.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ifelse</span>
  </div>
  <div class="_2_QraFYR_0">选择题90分</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-14 18:36:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/8f/cd/09874b2f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Godning</span>
  </div>
  <div class="_2_QraFYR_0">老师，我们在对kafka性能测试时遇到了too many files open问题，单节点服务器是256&#47;48t存储 磁盘做了raid0 集群是三台服务器构成 ，我们对单节点进行测试 启用五个topic 都是单分区 ，写入时带宽几乎占满 。当写入数据量到达3.6t时 系统文件句柄数就超过200w了 导致kafka崩溃 这种情况是kafka本身问题还是系统配置问题还是我们使用的问题呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 增加下ulimit -n</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-28 11:08:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>墙角儿的花</span>
  </div>
  <div class="_2_QraFYR_0">老师，服务部署在阿里云上，生产者发送消息经常在非上班期间发生超时，如晚8点到第二天8点间出现发送超时(org.apache.kafka.common.errors.TimeoutException)，而且根本就没什么qps，一分钟内都是个位数，白天时有一定的qps，但却不会出现超时，阿里云说网络没问题，请问该如何排查呢，谢谢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 通常都是连不上broker导致的。如果不是网络问题，查看下bootstrap.servers的配置吧</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-27 09:36:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/6c/e9/377a3b09.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>H.L.</span>
  </div>
  <div class="_2_QraFYR_0">kafka集群扩容， reassign主题分区迁移，这个不讲了吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 之前的课程应该有涉及这些</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-14 11:03:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/74/85/9443a0a9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>旭旭</span>
  </div>
  <div class="_2_QraFYR_0">完整的看了你的这门课 实在不错 有种相见恨晚的感觉 希望老师多推出一些大数据相关的优秀课程！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢鼓励，对于Kafka我还有点信心，其他的信心不足，哈哈哈<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-03 14:29:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/31/9d/daad92d2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Stony.修行僧</span>
  </div>
  <div class="_2_QraFYR_0">补充一下<br>如果你的Partition只有一个副本，也就是一个Leader，任何Follower都没有，你认为acks=all有用吗？<br>当然没用了，因为ISR里就一个Leader，他接收完消息后宕机，也会导致数据丢失。<br>所以说，这个acks=all，必须跟ISR列表里至少有2个以上的副本配合使用，起码是有一个Leader和一个Follower才可以。<br>这样才能保证说写一条数据过去，一定是2个以上的副本都收到了才算是成功，此时任何一个副本宕机，不会导致数据丢失。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-25 06:12:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/6a/06/66831563.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>阡陌</span>
  </div>
  <div class="_2_QraFYR_0">嗯，很多收货。但是也有很多没理解透彻的，准备从头再来一遍，加深理解和记忆。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 加油，有问题随时留言交流：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-07 11:21:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/11/ba/2175bc50.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Mr.Brooks</span>
  </div>
  <div class="_2_QraFYR_0">虽然工作中从没用过kafka，依然学到很多。准备去追老师下一个源码课了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯嗯，我们第二季见~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-12 11:27:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/bf/22/26530e66.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>趁早</span>
  </div>
  <div class="_2_QraFYR_0">终于看完了，收货颇多，感谢胡总的辛苦付出</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢鼓励，一起加油！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-20 15:28:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/07/36/d677e741.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>黑山老妖</span>
  </div>
  <div class="_2_QraFYR_0">谢谢老师！<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 加油！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-17 22:54:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/f7/51/1bacf04f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Tc</span>
  </div>
  <div class="_2_QraFYR_0">老师的解读源码什么时候上</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 准备中。。。。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-11 07:34:22</div>
  </div>
</div>
</div>
</li>
</ul>