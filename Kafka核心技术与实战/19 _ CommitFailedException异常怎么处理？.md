<audio title="19 _ CommitFailedException异常怎么处理？" src="https://static001.geekbang.org/resource/audio/bc/73/bccc4d34ccff271123c704a33236d273.mp3" controls="controls"></audio> 
<p>你好，我是胡夕。今天我来跟你聊聊CommitFailedException异常的处理。</p><p>说起这个异常，我相信用过Kafka Java Consumer客户端API的你一定不会感到陌生。<strong>所谓CommitFailedException，顾名思义就是Consumer客户端在提交位移时出现了错误或异常，而且还是那种不可恢复的严重异常</strong>。如果异常是可恢复的瞬时错误，提交位移的API自己就能规避它们了，因为很多提交位移的API方法是支持自动错误重试的，比如我们在上一期中提到的<strong>commitSync方法</strong>。</p><p>每次和CommitFailedException一起出现的，还有一段非常著名的注释。为什么说它很“著名”呢？第一，我想不出在近50万行的Kafka源代码中，还有哪个异常类能有这种待遇，可以享有这么大段的注释，来阐述其异常的含义；第二，纵然有这么长的文字解释，却依然有很多人对该异常想表达的含义感到困惑。</p><p>现在，我们一起领略下这段文字的风采，看看社区对这个异常的最新解释：</p><blockquote>
<p>Commit cannot be completed since the group has already rebalanced and assigned the partitions to another member. This means that the time between subsequent calls to poll() was longer than the configured max.poll.interval.ms, which typically implies that the poll loop is spending too much time message processing. <span class="orange">You can address this either by increasing max.poll.interval.ms or by reducing the maximum size of batches returned in poll() with max.poll.records.</span></p>
</blockquote><!-- [[[read_end]]] --><p>这段话前半部分的意思是，本次提交位移失败了，原因是消费者组已经开启了Rebalance过程，并且将要提交位移的分区分配给了另一个消费者实例。出现这个情况的原因是，你的消费者实例连续两次调用poll方法的时间间隔超过了期望的max.poll.interval.ms参数值。这通常表明，你的消费者实例花费了太长的时间进行消息处理，耽误了调用poll方法。</p><p>在后半部分，社区给出了两个相应的解决办法（即橙色字部分）：</p><ol>
<li>增加期望的时间间隔max.poll.interval.ms参数值。</li>
<li>减少poll方法一次性返回的消息数量，即减少max.poll.records参数值。</li>
</ol><p>在详细讨论这段文字之前，我还想提一句，实际上这段文字总共有3个版本，除了上面的这个最新版本，还有2个版本，它们分别是：</p><blockquote>
<p>Commit cannot be completed since the group has already rebalanced and assigned the partitions to another member.<span class="orange"> This means that the time between subsequent calls to poll() was longer than the configured session.timeout.ms</span>, which typically implies that the poll loop is spending too much time message processing. You can address this either by <span class="orange">increasing the session timeout</span> or by reducing the maximum size of batches returned in poll() with max.poll.records.</p>
</blockquote><blockquote>
<p>Commit cannot be completed since the group has already rebalanced and assigned the partitions to another member. <span class="orange">This means that the time between subsequent calls to poll() was longer than the configured max.poll.interval.ms</span>, which typically implies that the poll loop is spending too much time message processing. You can address this either by <span class="orange">increasing the session timeout</span> or by reducing the maximum size of batches returned in poll() with max.poll.records.</p>
</blockquote><p>这两个较早的版本和最新版相差不大，我就不详细解释了，具体的差异我用橙色标注了。我之所以列出这些版本，就是想让你在日后看到它们时能做到心中有数，知道它们说的是一个事情。</p><p>其实不论是哪段文字，它们都表征位移提交出现了异常。下面我们就来讨论下该异常是什么时候被抛出的。从源代码方面来说，CommitFailedException异常通常发生在手动提交位移时，即用户显式调用KafkaConsumer.commitSync()方法时。从使用场景来说，有两种典型的场景可能遭遇该异常。</p><p><strong>场景一</strong></p><p>我们先说说最常见的场景。当消息处理的总时间超过预设的max.poll.interval.ms参数值时，Kafka Consumer端会抛出CommitFailedException异常。这是该异常最“正宗”的登场方式。你只需要写一个Consumer程序，使用KafkaConsumer.subscribe方法随意订阅一个主题，之后设置Consumer端参数max.poll.interval.ms=5秒，最后在循环调用KafkaConsumer.poll方法之间，插入Thread.sleep(6000)和手动提交位移，就可以成功复现这个异常了。在这里，我展示一下主要的代码逻辑。</p><pre><code>…
Properties props = new Properties();
…
props.put(&quot;max.poll.interval.ms&quot;, 5000);
consumer.subscribe(Arrays.asList(&quot;test-topic&quot;));
 
while (true) {
    ConsumerRecords&lt;String, String&gt; records = 
		consumer.poll(Duration.ofSeconds(1));
    // 使用Thread.sleep模拟真实的消息处理逻辑
    Thread.sleep(6000L);
    consumer.commitSync();
}
</code></pre><p>如果要防止这种场景下抛出异常，你需要简化你的消息处理逻辑。具体来说有4种方法。</p><ol>
<li>
<p><strong>缩短单条消息处理的时间</strong>。比如，之前下游系统消费一条消息的时间是100毫秒，优化之后成功地下降到50毫秒，那么此时Consumer端的TPS就提升了一倍。</p>
</li>
<li>
<p><strong>增加Consumer端允许下游系统消费一批消息的最大时长</strong>。这取决于Consumer端参数max.poll.interval.ms的值。在最新版的Kafka中，该参数的默认值是5分钟。如果你的消费逻辑不能简化，那么提高该参数值是一个不错的办法。值得一提的是，Kafka 0.10.1.0之前的版本是没有这个参数的，因此如果你依然在使用0.10.1.0之前的客户端API，那么你需要增加session.timeout.ms参数的值。不幸的是，session.timeout.ms参数还有其他的含义，因此增加该参数的值可能会有其他方面的“不良影响”，这也是社区在0.10.1.0版本引入max.poll.interval.ms参数，将这部分含义从session.timeout.ms中剥离出来的原因之一。</p>
</li>
<li>
<p><strong>减少下游系统一次性消费的消息总数</strong>。这取决于Consumer端参数max.poll.records的值。当前该参数的默认值是500条，表明调用一次KafkaConsumer.poll方法，最多返回500条消息。可以说，该参数规定了单次poll方法能够返回的消息总数的上限。如果前两种方法对你都不适用的话，降低此参数值是避免CommitFailedException异常最简单的手段。</p>
</li>
<li>
<p><strong>下游系统使用多线程来加速消费</strong>。这应该算是“最高级”同时也是最难实现的解决办法了。具体的思路就是，让下游系统手动创建多个消费线程处理poll方法返回的一批消息。之前你使用Kafka  Consumer消费数据更多是单线程的，所以当消费速度无法匹及Kafka Consumer消息返回的速度时，它就会抛出CommitFailedException异常。如果是多线程，你就可以灵活地控制线程数量，随时调整消费承载能力，再配以目前多核的硬件条件，该方法可谓是防止CommitFailedException最高档的解决之道。事实上，很多主流的大数据流处理框架使用的都是这个方法，比如Apache Flink在集成Kafka时，就是创建了多个KafkaConsumerThread线程，自行处理多线程间的数据消费。不过，凡事有利就有弊，这个方法实现起来并不容易，特别是在多个线程间如何处理位移提交这个问题上，更是极容易出错。在专栏后面的内容中，我将着重和你讨论一下多线程消费的实现方案。</p>
</li>
</ol><p>综合以上这4个处理方法，我个人推荐你首先尝试采用方法1来预防此异常的发生。优化下游系统的消费逻辑是百利而无一害的法子，不像方法2、3那样涉及到Kafka Consumer端TPS与消费延时（Latency）的权衡。如果方法1实现起来有难度，那么你可以按照下面的法则来实践方法2、3。</p><p>首先，你需要弄清楚你的下游系统消费每条消息的平均延时是多少。比如你的消费逻辑是从Kafka获取到消息后写入到下游的MongoDB中，假设访问MongoDB的平均延时不超过2秒，那么你可以认为消息处理需要花费2秒的时间。如果按照max.poll.records等于500来计算，一批消息的总消费时长大约是1000秒，因此你的Consumer端的max.poll.interval.ms参数值就不能低于1000秒。如果你使用默认配置，那默认值5分钟显然是不够的，你将有很大概率遭遇CommitFailedException异常。将max.poll.interval.ms增加到1000秒以上的做法就属于上面的第2种方法。</p><p>除了调整max.poll.interval.ms之外，你还可以选择调整max.poll.records值，减少每次poll方法返回的消息数。还拿刚才的例子来说，你可以设置max.poll.records值为150，甚至更少，这样每批消息的总消费时长不会超过300秒（150*2=300），即max.poll.interval.ms的默认值5分钟。这种减少max.poll.records值的做法就属于上面提到的方法3。</p><p><strong>场景二</strong></p><p>Okay，现在我们已经说完了关于CommitFailedException异常的经典发生场景以及应对办法。从理论上讲，关于该异常你了解到这个程度，已经足以帮助你应对应用开发过程中由该异常带来的“坑”了 。但其实，该异常还有一个不太为人所知的出现场景。了解这个冷门场景，可以帮助你拓宽Kafka Consumer的知识面，也能提前预防一些古怪的问题。下面我们就来说说这个场景。</p><p>之前我们花了很多时间学习Kafka的消费者，不过大都集中在消费者组上，即所谓的Consumer Group。其实，Kafka Java Consumer端还提供了一个名为Standalone Consumer的独立消费者。它没有消费者组的概念，每个消费者实例都是独立工作的，彼此之间毫无联系。不过，你需要注意的是，独立消费者的位移提交机制和消费者组是一样的，因此独立消费者的位移提交也必须遵守之前说的那些规定，比如独立消费者也要指定group.id参数才能提交位移。你可能会觉得奇怪，既然是独立消费者，为什么还要指定group.id呢？没办法，谁让社区就是这么设计的呢？总之，消费者组和独立消费者在使用之前都要指定group.id。</p><p>现在问题来了，如果你的应用中同时出现了设置相同group.id值的消费者组程序和独立消费者程序，那么当独立消费者程序手动提交位移时，Kafka就会立即抛出CommitFailedException异常，因为Kafka无法识别这个具有相同group.id的消费者实例，于是就向它返回一个错误，表明它不是消费者组内合法的成员。</p><p>虽然说这个场景很冷门，但也并非完全不会遇到。在一个大型公司中，特别是那些将Kafka作为全公司级消息引擎系统的公司中，每个部门或团队都可能有自己的消费者应用，谁能保证各自的Consumer程序配置的group.id没有重复呢？一旦出现不凑巧的重复，发生了上面提到的这种场景，你使用之前提到的哪种方法都不能规避该异常。令人沮丧的是，无论是刚才哪个版本的异常说明，都完全没有提及这个场景，因此，如果是这个原因引发的CommitFailedException异常，前面的4种方法全部都是无效的。</p><p>更为尴尬的是，无论是社区官网，还是网上的文章，都没有提到过这种使用场景。我个人认为，这应该算是Kafka的一个bug。比起返回CommitFailedException异常只是表明提交位移失败，更好的做法应该是，在Consumer端应用程序的某个地方，能够以日志或其他方式友善地提示你错误的原因，这样你才能正确处理甚至是预防该异常。</p><h2>小结</h2><p>总结一下，今天我们详细讨论了Kafka Consumer端经常碰到的CommitFailedException异常。我们从它的含义说起，再到它出现的时机和场景，以及每种场景下的应对之道。当然，我也留了个悬念，在专栏后面的内容中，我会详细说说多线程消费的实现方式。希望通过今天的分享，你能清晰地掌握CommitFailedException异常发生的方方面面，从而能在今后更有效地应对此异常。</p><p><img src="https://static001.geekbang.org/resource/image/df/88/df3691cee68c7878efd21e79719bec88.jpg?wh=2069*2569" alt=""></p><h2>开放讨论</h2><p>请比较一下今天我们提到的预防该异常的4种方法，并说说你对它们的理解。</p><p>欢迎写下你的思考和答案，我们一起讨论。如果你觉得有所收获，也欢迎把文章分享给你的朋友。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/4f/85/c92bcc6b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小生向北</span>
  </div>
  <div class="_2_QraFYR_0">max.poll.interval.ms是指两次poll()的最大间隔时间，kafka消费者以轮询的方式来拉取消息，并且一次拉取批量的消息（默认500条），而批量的大小是通过max.poll.records来控制的。两次poll()的实际时间取决于 单条消息的处理时间*一次拉取的消息量（500），当超过max.poll.interval.ms配置的时间Kafka server认为kafka consumer掉线了，于是就执行分区再均衡将这个consumer踢出消费者组。但是consumer又不知道服务端把自己给踢出了，下次在执行poll()拉取消息的时候（在poll()拉取消息之前有个自动提交offset的操作），就会触发该问题。 可见第2,3种方案是通过调整Kafka consumer的配置参数来缩短业务总的处理时间或者增加服务端判断时长，比较容易实现；第1种就跟业务有关了，比较难搞，有些业务可能就是要这么长的时间，很难再缩短；第4种方案就更复杂了，要把同步消息转换成异步，交给其它线程来处理，这时需要把auto.commit.enable=false，手动提交offset，并且consumer是线程不安全的，异步线程何时处理完，何时该提交，在哪提交，也是应用需要考虑的问题！希望胡老师针对第4种方案重点探讨一下！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-17 16:23:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/c7/dc/9408c8c2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ban</span>
  </div>
  <div class="_2_QraFYR_0">老师，1、请问Standalone Consumer 的独立消费者一般什么情况会用到<br>2、Standalone Consumer 的独立消费者 使用跟普通消费者组有什么区别的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. 很多流处理框架的Kafka connector都没有使用consumer group，而是直接使用standalone consumer，因为group机制不好把控<br>2. standalone consumer没有rebalance，也没有group提供的负载均衡，你需要自己实现。其他方面（比如位移提交）和group没有太大的不同</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-16 23:01:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/47/1b/64262861.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>胡小禾</span>
  </div>
  <div class="_2_QraFYR_0">“当消息处理的总时间超过预设的 max.poll.interval.ms 参数值时，Kafka Consumer 端会抛出 CommitFailedException 异常”。<br><br><br>其实逻辑是这样：消息处理的总时间超过预设的 max.poll.interval.ms 参数值  导致了 Rebalance‘；<br>rebalance导致了 partition assgined 的consumer member变了；<br>导致原来的consumer 想要commit都没法commit 。（因为元信息,比如连的broker都变了）.<br><br>请老师指正下</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，差不多是这个道理：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-11 00:38:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/47/1b/64262861.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>胡小禾</span>
  </div>
  <div class="_2_QraFYR_0">为啥自动commit 不会抛 CommitFailedException？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 自动commit失败由Kafka内部消化处理</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-11 23:56:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/fd/59/4416b40f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>德惠先生</span>
  </div>
  <div class="_2_QraFYR_0">希望老师可以更加具体的说说，rebalance的细节，比如某个consumer发生full gc的场景，它的partition是怎么被分配走的，重连之后提交会发生什么</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 假设full gc导致所有线程STW，从而心跳中断，导致被踢出group，Coordinator向其他存活consumer发送心跳response，通知它们开启新一轮rebalance。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-16 08:43:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/c7/dc/9408c8c2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ban</span>
  </div>
  <div class="_2_QraFYR_0">老师，我想问下max.poll.interval.ms两者session.timeout.ms有什么联系，可以说0.10.1.0 之前的客户端 API，相当于session.timeout.ms代替了max.poll.interval.ms吗？<br>比如说session.timeout.ms是5秒，如果消息处理超过5秒，也算是超时吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，我更愿意说是max.poll.interval.ms承担了session.timeout.ms的部分功能。在没有max.poll.interval.ms和单独的心跳线程之前，如果session.timeout.ms = 5s，消息处理超过了5s，那么consumer就算是超时</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-16 22:37:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/1a/ad/faf1bf19.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>windcaller</span>
  </div>
  <div class="_2_QraFYR_0">To use this mode, instead of subscribing to the topic using subscribe, you just call assign(Collection) with the full list of partitions that you want to consume.<br><br>     String topic = &quot;foo&quot;;<br>     TopicPartition partition0 = new TopicPartition(topic, 0);<br>     TopicPartition partition1 = new TopicPartition(topic, 1);<br>     consumer.assign(Arrays.asList(partition0, partition1));<br> <br>Once assigned, you can call poll in a loop, just as in the preceding examples to consume records. The group that the consumer specifies is still used for committing offsets, but now the set of partitions will only change with another call to assign. Manual partition assignment does not use group coordination, so consumer failures will not cause assigned partitions to be rebalanced. Each consumer acts independently even if it shares a groupId with another consumer. To avoid offset commit conflicts, you should usually ensure that the groupId is unique for each consumer instance.<br><br>老师 standalone mode 是上面这段内容吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的。使用assign的consumer就是standalone consumer</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-31 06:08:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/a8/e2/f8e51df2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Li Shunduo</span>
  </div>
  <div class="_2_QraFYR_0">假如broker集群整个挂掉了，过段时间集群恢复后，consumer group会自动恢复消费吗？还是需要手动重启consumer机器？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: consumer有重连机制</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-16 17:39:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/fd/fd/326be9bb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>注定非凡</span>
  </div>
  <div class="_2_QraFYR_0">A ：定义：所谓CommitFailedException，是指Consumer客户端在提交位移时出现了错误或异常，并且并不可恢复的严重异常。<br><br>B ：导致原因：<br>	（1）消费者端处理的总时间超过预设的max.poll.interval.ms参数值<br>	（2）出现一个Standalone Consumerd的独立消费者，配置的group.id重名冲突。<br><br>C ：解决方案：<br>	（1）减少单条消息处理的时间<br>	（2）增加Consumer端允许下游系统消费一批消息的最大时长<br>	（3）减少下游系统一次性消费的消息总数。<br>	（4）下游使用多线程加速消费<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-05 16:28:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/05/db/1bd78419.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>有时也，命也，运也，如之奈何？</span>
  </div>
  <div class="_2_QraFYR_0">老师kafka死信该怎么去实现的？<br>2.0之后增加了如下配置：<br>errors.tolerance = all<br>errors.deadletterqueue.topic.name = &quot;&quot;？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 还不够，你需要使用Kafka Connect组件才能实现。见：https:&#47;&#47;www.confluent.io&#47;blog&#47;kafka-connect-deep-dive-error-handling-dead-letter-queues</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-30 11:36:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/04/0d/3dc5683a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>柯察金</span>
  </div>
  <div class="_2_QraFYR_0">老师，没有设置 group.id 话，会怎么样，系统会自动生成唯一的一个值吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: group.id是必须要设置的，否则会抛InvalidGroupIdException异常</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-08 08:56:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/1a/ad/faf1bf19.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>windcaller</span>
  </div>
  <div class="_2_QraFYR_0">我没在kafka官网、stackoverflow 、google 找到任何关于 standalone kafka consumer的 例子，还望老师给个链接学习学习</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Standalone consumer的提法并未出现在官方文档中，你可以在javadoc中看到一些：https:&#47;&#47;kafka.apache.org&#47;23&#47;javadoc&#47;org&#47;apache&#47;kafka&#47;clients&#47;consumer&#47;KafkaConsumer.html#manualassignment</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-30 03:51:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/48/f3/f1034ffd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>cricket1981</span>
  </div>
  <div class="_2_QraFYR_0">&quot;不幸的是，session.timeout.ms 参数还有其他的含义，因此增加该参数的值可能会有其他方面的“不良影响”，这也是社区在 0.10.1.0 版本引入 max.poll.interval.ms 参数，将这部分含义从 session.timeout.ms 中剥离出来的原因之一。&quot;---&gt;能细述一下不良影响吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 之前版本中session.timeout.ms有多重含义，session过期时间、消息处理逻辑最大时间等</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-16 09:03:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/15/d5/6d66288b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>极极</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，这边遇到个很奇怪的问题<br><br>开启第一个消费者的时候，正常消费<br><br>但是开启另一个后，触发了 rebalanced<br><br>这时候第一个消费者会报出如下错误：<br>The provided member is not known in the current generation<br><br>是因为第一个消费者被踢出了 generation，但是它不知道，还在继续消费提交位移，或者做着其他事情？这个其他事情可能是什么？<br><br>还有就是 rebalance发生的时候，消费者是立即暂停，还是会消费完整个poll？这时候coordinator会等他吗？还是直接踢出去？<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 两个consumer都是使用subscribe方法订阅的topic吗？<br><br>另外，rebalance发生时Coordinator会通过心跳response通知消费者。如果消费者此时正在处理消息，肯定是不会响应的，毕竟还没有强行中断的机制</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-22 16:50:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>crud~boy</span>
  </div>
  <div class="_2_QraFYR_0">老师poll一批消息，多线程处理并且是手动处理，会不会每个线程速度不一致，会导致提交位移时，offset小得后提交，会有什么影响吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果是多线程提交位移，的确有可能出现这种情况。影响就是可能出现重复消费</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-20 08:07:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/d5/7b/c512da6a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>石栖</span>
  </div>
  <div class="_2_QraFYR_0">关于这个错误，我这边很奇怪，consumer用的是自动提交的配置，但是也出现了这个错误。看错误应该是broker-2挂掉了，然后rediscovery。但是后面日志又说The coordinator is not aware of this member.再后面就是Commit cannot be completed 的错误了。我这里是有多个broker，然后采用的域名的方式，ip可能会变。老师通过这些日志，能给点建议吗？<br>错误日志：<br>2020-05-04 18:49:56.592  INFO 6 --- [ntainer#0-0-C-1] o.a.k.c.c.internals.AbstractCoordinator  : [Consumer clientId=consumer-2, groupId=test-group] Discovered group coordinator broker-2-lhfm0slmx1v4nyfz.kafka.svc01.local:9093 (id: 2147483645 rack: null)<br>2020-05-04 18:49:56.592  INFO 6 --- [ntainer#0-0-C-1] o.a.k.c.c.internals.AbstractCoordinator  : [Consumer clientId=consumer-2, groupId=test-group] Group coordinator broker-2-lhfm0slmx1v4nyfz.kafka.svc01.local:9093 (id: 2147483645 rack: null) is unavailable or invalid, will attempt rediscovery<br>2020-05-04 18:49:56.694  INFO 6 --- [ntainer#0-0-C-1] o.a.k.c.c.internals.AbstractCoordinator  : [Consumer clientId=consumer-2, groupId=test-group] Discovered group coordinator broker-2-lhfm0slmx1v4nyfz.kafka.svc01.local:9093 (id: 2147483645 rack: null)<br>2020-05-04 18:49:56.735 ERROR 6 --- [ntainer#0-0-C-1] o.a.k.c.c.internals.ConsumerCoordinator  : [Consumer clientId=consumer-2, groupId=test-group] Offset commit failed on partition test.topic-0 at offset 0: The coordinator is not aware of this member.<br>2020-05-04 18:49:56.735  WARN 6 --- [ntainer#0-0-C-1] o.a.k.c.c.internals.ConsumerCoordinator  : [Consumer clientId=consumer-2, groupId=test-group] Asynchronous auto-commit of offsets {test.topic-0=OffsetAndMetadata{offset=0, metadata=&#39;&#39;}} failed: Commit cannot be completed since the group has already rebalanced and assigned the partitions to another member. This means that the time between subsequent calls to poll() was longer than the configured max.poll.interval.ms, which typically implies that the poll loop is spending too much time message processing. You can address this either by increasing the session timeout or by reducing the maximum size of batches returned in poll() with max.poll.records.</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 能告知一下Kafka版本吗？目前社区对这部分代码改动很多，需要确认下是哪个版本碰到的问题</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-05 17:56:47</div>
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
  <div class="_2_QraFYR_0">好像只说了如何避免出现异常，但是异常出现了怎么处理呢，有类似回滚什么的吗？会重复消费吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不会回滚。消费出了任何问题都表现为consumer无法继续处理，然后人工需要介入，然后可能出现重复消费</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-01 13:45:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>松鼠鱼</span>
  </div>
  <div class="_2_QraFYR_0">老师，我的程序使用的是自动提交offset，其他interval等相关参数（max poll, session timeout等）都是默认的。消息消费的速度也算快，500条应该就是一秒的事儿。可是这样也还是偶尔会发生Offset commit failed异常，通常是在程序跑了一阵之后，请问您有没有遇到过类似的情况？这是不是和单一一个consumer同时消费多个partition有关？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这可能是因为broker端的问题而引起的，比如Coordinator变更了，或开启了新的Rebalance。如果是偶发的，你不用太过担心~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-27 16:44:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/0c/c2/bad34a50.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张洋</span>
  </div>
  <div class="_2_QraFYR_0">老师有两个疑问：<br><br>1.相同的GroupId的Consumer 不应该就是同一个Consumer Group 组下的吗，或者有其他的区分条件，比如订阅的Topic不同？<br><br>2.如果这个standalone Consumer 再给他添加一个同组的standalone Conusmer，会发生什么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. 设置相同group.id的consumer就是属于同一个group，你说的是对的：）<br>2. 会出现位移提交失败的问题。严格来说这其实是一个问题，但是如果反馈到社区，社区会认为这不是标准用法</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-21 09:55:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/52/d8/123a4981.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>绿箭侠</span>
  </div>
  <div class="_2_QraFYR_0">场景二，为什么只强调standalone 和 group之间可能会发生groupid相同，那group之间相同怎么样呢？ 是不是会在coordinator注册group时就能发现组之间重复了groupid ?!!!</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: group之间相同？那么就会认为是同一个group</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-30 23:51:35</div>
  </div>
</div>
</div>
</li>
</ul>