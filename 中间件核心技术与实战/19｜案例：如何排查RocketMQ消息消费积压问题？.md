<audio title="19｜案例：如何排查RocketMQ消息消费积压问题？" src="https://static001.geekbang.org/resource/audio/53/cb/53e17da4c1c759801046311274d74bcb.mp3" controls="controls"></audio> 
<p>你好，我是丁威。</p><p>我想，几乎每一位使用过消息中间件的小伙伴，都会在消息消费时遇到消费积压的问题。在处理这类问题时，大部分同学都会选择横向扩容。但不幸的是，这种解决办法治标不治本，到最后问题还是得不到解决。</p><p>说到底，消费端出现消息消费积压是一个结果，但引起这个结果的原因是什么呢？<strong>在没有弄清楚原因之前谈优化和解决方案都显得很苍白。</strong></p><p>这节课，我们就进一步认识一下消费积压和RocketMQ的消息消费模型，看看怎么从根本上排查消费积压的问题。</p><h2>RocketMQ的消息消费模型</h2><p>在RocketMQ消费领域中，判断消费端遇到的瓶颈通常会用到两个重要的指标：Delay和LastConsumeTime。</p><p>在开源版本的控制台rocketmq-console界面中，我们可以查阅消费端的这两个指标：</p><p><img src="https://static001.geekbang.org/resource/image/yy/07/yy0cf8266e7c6c1cb8cfa7caf7562207.png?wh=1039x692" alt="图片"></p><ul>
<li>Delay指的是消息积压数量，它是由BrokerOffset（服务端当前最大的逻辑偏移量）减去ConsumerOffset（消费者消费的当前位点）计算出来的。<strong>如果Delay值很大，说明消费端遇到了瓶颈</strong>。</li>
<li>LastConsumeTime表示上一次成功消费消息的存储时间。<strong>这个值如果很大，同样能说明消费端遇到了瓶颈。</strong>如果这个值线上为1970年，表示消费者当前消费位点对应的消息在服务端已经过期，被删除了。</li>
</ul><!-- [[[read_end]]] --><p>那为什么消费会积压呢？要理解这个问题，我们首先要了解RocketMQ消费者的消费处理模型。核心流程如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/69/72/694cf4yybd4f3ef68c14377d4e755672.jpg?wh=1920x889" alt="图片"></p><p>说明一下具体的工作流程。</p><ol>
<li>PullMessageService线程从拉取任务队列中获取一个待拉取任务PullRquest。</li>
<li>PullMessageService线程根据PullRequest中的主题名称、队列编号、拉取位点向Broker服务器拉取一批消息。拉取到消息后，服务端会更新PullRequest中下一次拉取任务的偏移量，将其放到队列的尾部。</li>
<li>PullMessageService线程将拉取到的消息存入到处理队列（ProcessQueue），每一个MessageQueue（Broker名称+主题名称+队列编号）对应一个处理队列。</li>
<li>PullMessageService线程将拉取到的消息提交到线程池。</li>
<li>PullMessageService线程将消息提交到线程池后，不会等这批消息处理完成，而是立即返回。然后PullMessageService线程重复步骤一到步骤五。</li>
<li>当消息提交到消费线程池后，进行异步消费。消息消费成功后，会将消息从处理队列（ProcessQueue）中移除，然后获取处理队列中的最小偏移量，提交消费位点。</li>
</ol><p>从这个过程中可以看出，在RocketMQ的消费处理模型中，PullMessageService线程“马不停歇”地从拉取队列中获取任务，拉完一批消息后继续再将PullRequest（待拉取任务）放入到队列末尾，确保PullMessageService可以不间断地拉取消息，从而实现Push模式的效果。</p><p>从理论设计的角度，我们不难看出产生消费积压的原因可能有两个。</p><ul>
<li>第一，Pull线程不拉取消息，那就无法消费消息，没有消费消息，消费位点自然不会提交。</li>
<li>第二，消费线程池中的线程因为某种原因阻塞，导致不消费消息，进而同样使得消费位点不提交。</li>
</ul><p>针对第一点，Pull线程的run方法采用的是while(true)+try catch的模式，只要不主动关闭消费者，这个线程是不会停止的。具体的代码实现如下：</p><p><img src="https://static001.geekbang.org/resource/image/c2/ac/c2dc20f9952417b2fdd742c09c26dcac.png?wh=1920x978" alt="图片"></p><p>这么看来，消费积压基本都是消费线程池由于某种原因阻塞导致的。</p><p>在探究阻塞会发生在何处之前，你不妨思考一下，如果消费线程不干活，但拉取线程还一直在从服务端拉取消息，再将消息提交到消费线程池和ProcessQueue，这时会出现什么问题？</p><p>没错，内存溢出。所以，为了保护消费者进程，这个时候我们必须引入限流机制限制拉取线程的行为。</p><p>在RocketMQ中，我们主要通过三点来判断是否需要进行限流：</p><ul>
<li>消息消费端队列中积压的消息超过 1000 条；</li>
<li>消息处理队列中积压的消息尽管没有超过 1000 条，但最大偏移量和最小偏移量的差值超过2000；</li>
<li>消息处理队列中积压的消息总大小超过100M。</li>
</ul><p>RocketMQ一旦触发限流，往往会在${user_home}/logs/rocketmqlogs/rocketmq_client.log文件中打印对应的日志，如果日志中包含了关键字“so do flow control”，表明消费端存在性能瓶颈，这就是我们的突破方向。</p><h2>如何排查RocketMQ消息消费积压问题？</h2><p>那如何定位消费端慢在哪，又是卡在了哪行代码呢？</p><p>我们常用的排查方法是跟踪线程栈，利用jstack命令查看线程运行情况，以此探究线程的运行情况。通常可以使用下面的命令：</p><pre><code class="language-plain">ps -ef | grep java
jstack pid &gt; j1.log
</code></pre><p>为了方便对比，我一般会连续打印五个文件，这样可以在五个文件中查看同一个消费者线程的状态，看它是否发生了变化。如果始终没有变化，说明该消费线程长时间阻塞，这就需要我们重点关注了。</p><p>在RocketMQ中，消费端线程以ConsumeMessageThread_打头，通过对线程的判断，可以发现下面这段代码：</p><p><img src="https://static001.geekbang.org/resource/image/9c/d0/9c079bb6ff965945595be90c8c3378d0.png?wh=1073x472" alt="图片"></p><p>这些线程的状态为RUNNABLE，并且在jstack日志中状态一直没有发生变化，说明这些线程是有问题的。通过线程栈，我们可以清楚地定位到具体的代码行。</p><p>在这个示例中，通过对线程栈的分析，我们发现是调用HTTP请求时没有设置超时时间，这就导致线程一直阻塞，对应的消息始终没有处理完成。消息一直在处理队列（ProcessQueue）中，而RocketMQ采取的又是最小位点提交机制，消费位点无法继续向前推进，这才出现了消费积压。</p><p>至此，消费积压问题的根本原因就定位出来了。</p><p>最后，我还想跟你分享几个小经验。</p><p>结合我的生产实践，通常情况下，RocketMQ消息发送问题很可能与服务端有直接关系，而RocketMQ消费端遇到的一些性能问题通常与消费进程自身有关系。</p><p>另外，消费积压的时候，可以简单关注一下这个集群其他消费者的情况。如果其他消费者没有积压，只有你负责的消费组有积压，那就一定是消费端代码的问题了。</p><p><strong>在这里最后再强调一遍，查看线程栈并不只是去查看线程状态为BLOCKED、TIME_WRATING的线程，RUNNABLE的线程状态同样需要查看。因为在一些网络操作中（例如，HTTP请求等待返回结果时、MySQL写入/查询等待获取执行结果时），线程的状态也是RUNNABLE。</strong></p><h2>总结</h2><p>好了，今天就讲到这里。我们这节课主要是聚焦在RocketMQ消息消费积压这个核心问题上，这是消费端最常见的问题。</p><p>刚才，我简单地介绍了消费积压、和LastConsumeTime的计算规则，然后详细地介绍了RocketMQ消息消费的核心流程，探究了消费者的限流策略，最后介绍了精准定位消费积压的方法。</p><h2>思考题</h2><p>在课程的最后，我也给你留一道思考题。</p><p>我们这节课提到，RocketMQ在消费端主要通过三种方式来判断是否需要限流。其中，限制积压的消息条数和消息总大小这个很容易理解，因为这样可以避免内存溢出。可是，为什么还需要限制消息处理队列中最大与最小偏移量之间的间隔呢？</p><p>欢迎你在留言区与我交流讨论，我们下节课见！</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/ce/79/673f4268.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小杰</span>
  </div>
  <div class="_2_QraFYR_0">因为消费位点是consumer定时发给broker的，而每次发的时候只发最小确认offset，所以有可能消费者挂了又起来，大量消息都得重新消费，老师是这个原因吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 非常正确，👏👏，虽然rocketmq无法避免消息重复投送，但还是有义务尽量少发生重复推送。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-04 18:38:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/fa/64/457325e6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Sam Fu</span>
  </div>
  <div class="_2_QraFYR_0">老师可以讲下消费成功或者失败 本地以及broker offset的更新机制吗？<br>是每条消息消费完都会更新本地offset还是拉取的一批全部处理完才会更新.<br><br>如果这一批有一条消费失败的就全部扔回吗 还是只扔消费失败的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 好，你这个我分两个问题回答。<br>1、关于消息的位点提交机制，为了提高降低服务端持久化位点的压力，RocketMQ的位点提交机制是批量+异步方式。具体是消费端在处理完一条消息后，会提交位点，但这个位点提交只是将位点存储在本地内存缓存中，然后定时(默认每隔5s)将本地缓存一次性的提交到Broker，Broker收到位点提交机制后，更新内存中的位点缓存，然后每隔5s持久化到位点存储文件。<br><br>2、如果一批失败了是怎么提交，一般是指并发消费，我们可以留意一下MessageListenerConcurrently中consumeMessage这个方法声明：ConsumeConcurrentlyStatus consumeMessage(final List&lt;MessageExt&gt; msgs, final ConsumeConcurrentlyContext context); 一批消息对应一个处理状态，所以如果其中一条消息失败了，这个时候这一批消息都会重试。<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-04 01:09:55</div>
  </div>
</div>
</div>
</li>
</ul>