<audio title="18｜案例：如何排查RocketMQ消息发送超时故障？" src="https://static001.geekbang.org/resource/audio/64/a4/649ea55d0a58a5b77aafab478c8bd6a4.mp3" controls="controls"></audio> 
<p>你好，我是丁威。</p><p>不知道你在使用RocketMQ的时候有没有遇到过让人有些头疼的问题。我在用RocketMQ时遇到的最常见，也最让我头疼的问题就是<strong>消息发送超时。</strong>而且这种超时不是大面积的，而是偶尔会发生，占比在万分之一到万分之五之间。</p><h2>现象与关键日志</h2><p>消息发送超时的情况下，客户端的日志通常是下面这样：</p><p><img src="https://static001.geekbang.org/resource/image/97/b3/9742c5e7e8ec8d1aebdc5f76b2a8c2b3.png?wh=1079x207" alt="图片"></p><p>我们这节课就从这些日志入手，看看怎样排查RocketMQ的消息发送超时故障。</p><p>首先，我们要查看RocketMQ相关的日志，在应用服务器上，RocketMQ的日志默认路径为${USER_HOME}/logs/rocketmqlogs/ rocketmq_client.log。</p><p>在上面这张图中，有两条非常关键的日志。</p><ul>
<li>
<p>invokeSync：wait response timeout exception.<br>
它表示等待响应结果超时。</p>
</li>
<li>
<p>recive response, but not matched any request.<br>
这条日志非常关键，它表示，尽管客户端在获取服务端返回结果时超时了，但客户端最终还是能收到服务端的响应结果，只是此时客户端已经在等待足够时间之后放弃处理了。</p>
</li>
</ul><h2>单一长连接如何实现多请求并发发送？</h2><p>为什么第二条日志超时后还能收到服务端的响应结果，又为什么匹配不到对应的请求了呢？</p><!-- [[[read_end]]] --><p>我们可以详细探究一下这背后的原理。原来，这是使用单一长连接进行网络请求的编程范式。举个例子，一条长连接向服务端先后发送了两个请求，客户端在收到服务端响应结果时，需要判断这个响应结果对应的是哪个请求。</p><p><img src="https://static001.geekbang.org/resource/image/43/0f/435035c392ea7907b9c46e59b169420f.jpg?wh=1920x581" alt="图片"></p><p>正如上图所示，客户端多个线程通过一条连接依次发送了req1，req2两个请求，服务端解码请求后，会将请求转发到线程池中异步执行。如果请求2处理得比较快，比请求1更早将结果返回给客户端，那客户端怎么识别服务端返回的数据对应的是哪个请求呢？</p><p>解决办法是，客户端在发送请求之前，会为这个请求生成一个本机器唯一的请求ID（requestId），它还会采用Future模式，将requestId和Future对象放到一个Map中，然后将reqestId放入请求体。服务端在返回响应结果时，会将请求ID原封不动地放入响应结果中。客户端收到响应时，会先解码出requestId，然后从缓存中找到对应的Future对象，唤醒业务线程，将返回结果通知给调用方，完成整个通信。</p><p>结合日志我发现，如果客户端在指定时间内没有收到服务端的请求，最终会抛出超时异常。但是，网络层面上客户端还是能收到服务端的响应结果。这就把矛头直接指向了Broker端，是不是Broker有瓶颈，处理慢导致的呢？</p><h2>如何诊断Broker端内存写入性能？</h2><p>我们知道消息发送时，一个非常重要的过程就是服务端写入。如果服务端出现写入瓶颈，通常会返回各种各样的Broker Busy。我们可以简单来看一下消息发送的写入流程：</p><p><img src="https://static001.geekbang.org/resource/image/9y/6c/9yy4cd8794f51ddab23cc7847943476c.jpg?wh=1920x1093" alt="图片"></p><p>我们首先要判断的是，是不是消息写入PageCache或者磁盘写入慢导致的问题。我们这个集群采用的是异步刷盘机制，所以写磁盘这一环可以忽略。</p><p>然后，我们可以通过跟踪Broker端写入PageCache的数据指标来判断Broker有没有遇到瓶颈。具体做法是查看RocketMQ中的store.log文件，具体使用命令如下：</p><pre><code class="language-plain">cd /home/codingw/logs/rocketmqlogs/store.log //其中codingw为当前rocketmq broker进程的归属用户
grep "PAGECACHERT" store.log
</code></pre><p>执行命令后，可以得到这样的结果：</p><p><img src="https://static001.geekbang.org/resource/image/28/41/28e8dcd57b18b1db9cd562796e923341.png?wh=1077x226" alt="图片"></p><p>这段日志记录了消息写入到PageCache的耗时分布。通过分析我们可以知道，写入PageCache的耗时都小于100ms，所以PageCache的写入并没有产生瓶颈。不过，<strong>客户端可是真真切切地在3秒后才收到响应结果，难道是网络问题？</strong></p><h2>网络层排查通用方法</h2><p>接下来我们就分析一下网络。</p><p>通常，我们可以用netstat命令来分析网络通信，需要重点关注网络通信中的Recv-Q与Send-Q这两个指标。</p><p>netstat命令的执行效果如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/b3/75/b395b317544c3961c064985027026a75.png?wh=1077x244" alt="图片"></p><p>解释一下，这里的Recv-Q是TCP通道的接受缓存区；Send-Q是TCP通道的发送缓存区。</p><p>在TCP中，Recv-Q和Send-Q的工作机制如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/a0/1b/a0f374d6e43cf7aaeb950fc6ec634f1b.jpg?wh=1920x736" alt="图片"></p><p>正如上图描述的那样，网络通信有下面几个关键步骤。</p><ul>
<li>客户端调用网络通道时（例如NIO的Channel写入数据），数据首先是写入到TCP的发送缓存区，如果发送缓存区已满，客户端无法继续向该通道发送请求，从NIO层面调用Channel底层的write方法的时候会返回0。这个时候在应用层面需要注册写事件，待发送缓存区有空闲时，再通知上层应用程序继续写入上次未写入的数据。</li>
<li>数据进入到发送缓存区后，会随着网络到达目标端。数据首先进入的是目标端的接收缓存区，如果服务端采用事件选择机制的话，通道的读事件会就绪。应用从接收缓存区成功读取到字节后，会发送ACK给发送方。</li>
<li>发送方在收到ACK后，会删除发送缓冲区的数据。如果接收方一直不读取数据，那发送方也无法发送数据。</li>
</ul><p>运维同事分别在客户端和MQ服务器上，在服务器上写一个脚本，每500ms采集一次netstat 。最终汇总到的采集结果如下：</p><p><img src="https://static001.geekbang.org/resource/image/d2/bc/d2663e95c03913273d319eece0baaabc.png?wh=1075x234" alt="图片"></p><p>从客户端来看，客户端的Recv-Q中出现大量积压，它对应的是MQ的Send-Q中的大量积压。</p><p>结合Recv-Q、Send-Q的工作机制，再次怀疑可能是客户端从网络中读取字节太慢导致的。为了验证这个观点，我修改了和RocketMQ Client相关的包，加入了Netty性能采集方面的代码：</p><p><img src="https://static001.geekbang.org/resource/image/7c/4d/7c7ac37cae929419c65f4327cfbdb14d.png?wh=856x530" alt="图片"></p><p>我的核心思路是，针对每一次被触发的读事件，判断客户端会对一个通道进行多少次读取操作。如果一次读事件需要触发很多次的读取，说明这个通道确实积压了很多数据，网络读存在瓶颈。</p><p>部分采集数据如下：</p><p><img src="https://static001.geekbang.org/resource/image/53/95/5374ce040ff2a31e2b7ce6e565762f95.png?wh=1058x102" alt="图片"></p><p>我们可以通过awk命令对这个数据进行分析。从结果可以看出，一次读事件触发，大部分通道只要读两次就可以成功抽取读缓存区中的数据。读数据方面并不存在瓶颈。</p><p>统计分析结果如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/38/ff/38c60d07ce3ff424c69e81de77dc8eff.png?wh=673x173" alt="图片"></p><p>如此看来，瓶颈应该不在客户端，还是需要将目光转移到服务端。</p><p>从刚才的分析中我们已经看到，Broker服务端写入PageCache很快。但是刚刚我们唯独没有监控“响应结果写入网络”这个环节。那是不是写入响应结果不及时，导致消息大量积压在Netty的写缓存区，不能及时写入到TCP的发送缓冲区，最终造成消息发送超时呢？</p><h2>解决方案</h2><p>为了验证这个设想，我最初的打算是改造代码，从Netty层面监控服务端的写性能。但这样做的风险比较大，所以我暂时搁置了这个计划，又认真读了一遍RocketMQ封装Netty的代码。在这之前，我一直以为 RocketMQ的 网络层基本不需要参数优化，因为公司的服务器都是64核心的，而Netty的IO线程默认都是CPU的核数。</p><p>但这次阅读源码后我发现，RocketMQ中和IO相关的线程参数有两个，分别是serverSelectorThreads（默认值为3）和serverWorkerThreads（默认值为8）。</p><p>在Netty中，serverSelectorThreads就是WorkGroup，即所谓的IO线程池。每一个线程池会持有一个NIO中的Selector对象用来进行事件选择，所有的通道会轮流注册在这3个线程中，绑定在一个线程中的所有Channel会串行进行网络读写操作。</p><p>我们的MQ服务器的配置，CPU的核数都在48C及以上，用3个线程来做这件事显然太“小家子气”，这个参数可以调优。</p><p>RocketMQ的网络通信层使用的是Netty框架，默认情况下事件的传播（编码、解码）都在IO线程中，也就是上面提到的Selector对象所在的线程。</p><p>在RocketMQ中IO线程就只负责网络读、写，然后将读取到的二进制数据转发到一个线程池处理。这个线程池会负责数据的编码、解码等操作，线程池线程数量由serverWorkerThreads指定。</p><p>看到这里，我开始心潮澎湃了，我感觉自己离真相越来越近了。参考Netty将IO线程设置为CPU核数的两倍，我的第一波优化是让serverSelectorThreads=16，serverWorkerThreads=32，然后在生产环境中进行一波验证。</p><p>经过一个多月的验证，在集群数量逐步减少，业务量逐步上升的背景下，我们生产环境的消息发送超时比例达到了十万分之一，基本可以忽略不计。</p><p>网络超时问题的排查到这里就彻底完成了。但生产环境复杂无比，我们基本无法做到100%不出现超时。</p><p>比方说，虽然调整了Broker服务端网络的相关参数，超时问题得到了极大的缓解，但有时候还是会因为一些未知的问题导致网络超时。如果在一定时间内出现大量网络超时，会导致线程资源耗尽，继而影响其他业务的正常执行。</p><p>所以在这节课的最后我们再从代码层面介绍如何应对消息发送超时。</p><h2>发送超时兜底策略</h2><p>我们在应用中使用消息中间件就是看中了消息中间件的低延迟。但是如果消息发送超时，这就和我们的初衷相违背了。为了尽可能避免这样的问题出现，消息中间件领域解决超时的另一个思路是：<strong>增加快速失败的最大等待时长，并减少消息发送的超时时间，增加重试次数。</strong></p><p>我们来看下具体做法。</p><ol>
<li>增加 Broker 端快速失败的等待时长。这里建议为 1000。在 Broker 的配置文件中增加如下配置：</li>
</ol><pre><code class="language-plain">maxWaitTimeMillsInQueue=1000
</code></pre><ol start="2">
<li>减少超时时间，增加重试次数。</li>
</ol><p>你可能会问，现在已经发生超时了，你还要减少超时时间，那发生超时的概率岂不是更大了？</p><p>这样做背后的动机是希望客户端尽快超时并快速重试。因为局域网内的网络抖动是瞬时的，下次重试时就能恢复。并且 RocketMQ 有故障规避机制，重试的时候会尽量选择不同的 Broker。</p><p>执行这个操作的代码和版本有关，如果 RocketMQ 的客户端版本低于4.3.0，代码如下：</p><pre><code class="language-plain">  DefaultMQProducer producer = new DefaultMQProducer("dw_test_producer_group");
 &nbsp;producer.setNamesrvAddr("127.0.0.1:9876");
 &nbsp;producer.setRetryTimesWhenSendFailed(5);//　同步发送模式：重试次数
 &nbsp;producer.setRetryTimesWhenSendAsyncFailed(5);// 异步发送模式：重试次数
 &nbsp;producer.start();
 &nbsp;producer.send(msg,500);//消息发送超时时间
</code></pre><p>如果客户端版本是 4.3.0 及以上版本，因为设置的消息发送超时时间是所有重试的总的超时时间，所以不能直接设置 RocketMQ 的发送 API 的超时时间，而是需要对RocketMQ API 进行包装，例如示例代码如下：</p><pre><code class="language-plain">&nbsp;public static SendResult send(DefaultMQProducer producer, Message msg, int 
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;retryCount) {
 &nbsp; &nbsp; &nbsp;Throwable e = null;
 &nbsp; &nbsp; &nbsp;for(int i =0; i &lt; retryCount; i ++ ) {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;try {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;return producer.send(msg,500); //设置超时时间，为 500ms，内部有重试机制
 &nbsp; &nbsp; &nbsp; &nbsp;  } catch (Throwable e2) {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;e = e2;
 &nbsp; &nbsp; &nbsp; &nbsp;  }
 &nbsp; &nbsp;  }
 &nbsp; &nbsp; &nbsp;throw new RuntimeException("消息发送异常",e);
 &nbsp; }
</code></pre><h2><strong>总结</strong></h2><p>好了，我们这节课就介绍到这里了。</p><p>这节课，我首先抛出一个生产环境中，消息发送环节最容易遇到的问题：消息发送超时问题。我们对日志现象进行了解读，并引出了单一长连接支持多线程网络请求的原理。</p><p>整个排查过程，我首先判断了一下Broker写入PageCache是否有瓶颈，然后通过netstat命令，以Recv-Q、Send-Q两个指标为依据进行了网络方面的排查，最终定位到瓶颈可能在于服务端网络读写模型。通过研读RocketMQ的网络模型，我发现了两个至关重要的参数，serverSelectorThreads和serverWorkerThreads。其中：</p><ul>
<li>serverSelectorThreads是RocketMQ服务端IO线程的个数，默认为3，建议设置为CPU核数；</li>
<li>serverWorkerThreads是RocketMQ事件处理线程数，主要承担编码、解码等责任，默认为8，建议设置为CPU核数的两倍。</li>
</ul><p>通过调整这两个参数，我们极大地降低了网络超时发生的概率。</p><p>不过，发生网络超时的原因是多种多样的，所以我们还介绍了第二种方法，<strong>那就是降低超时时间，增加重试的次数，从而降低网络超时对运行时线程的影响，降低系统响应时间。</strong></p><h2><strong>课后题</strong></h2><p>学完今天的内容，我也给你留一道课后题吧。</p><p>网络通讯在中间件领域非常重要，掌握网络排查相关的知识对线上故障分析有很大的帮助。建议你系统地学习一下netstat命令和网络抓包相关技能，分享一下你的经验和困惑。我们下节课见！</p>
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
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJ1rIbtzMltwtkdOgyk7nxzQOZtocVBwuAsZbUgY2gZHfnds4Onj6Zcxcba7fPI1qyHcb9jzJibZqA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>koutann</span>
  </div>
  <div class="_2_QraFYR_0">运维同事分别在客户端和 MQ 服务器上，在服务器上写一个脚本，每 500ms 采集一次 netstat 。从客户端的日志信息发现Recv-Q 中出现大量积压<br>======<br>这里MQ服务端采集到的netstat日志Send-Q有积压现象吗？<br>如果没有积压的话，因为服务器IO线程数量不足导致的问题，为啥会导致客户端的Recv-Q出现积压呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个问题问的非常棒，讲真，其实这里的逻辑，受限于我网络方面的知识，目前还不能给出一个底层的逻辑，后续解决这个问题，还是想着从网络层面再想想版本，就仔细阅读了相关的源码，发现的那两个参数，调整后确实效果好了，问题得以解决，关于网络方面，我估计在今年4季度会针对性的学习，学习成功将发布在我的公众「中间件兴趣圈」，欢迎关注，我们后续多多交流。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-07 14:33:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/20/69/ed/2ea74ecd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>越过山丘</span>
  </div>
  <div class="_2_QraFYR_0">return producer.send(msg,500); &#47;&#47;设置超时时间，为 500ms，内部有重试机制<br><br>这里的500ms低于 Broker maxWaitTimeMillsInQueue=1000<br>会不会导致 Broker繁忙时候或者网络抖动时间，响应时间超过了500ms，导致客户端所有消息都重试多次，重试消息和之前的消息都积压在Broker的内存中，一方面对Broker造成压力，影响Broker的正常处理能力，一方面造成消息的重复率变高<br><br>目的是让客户端快速</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对的，为你👍，消息发送阶段的重复率会变高，这里需要做权衡，主要是因为消息发送超时，会占用线程资源，如果并发比较高，消息发送超时发生概率大，就有阻塞线程，导致线程容易耗尽，造成用户使用响应极慢。当然在实践过程中，可以让设置的超时时间大于maxWaitTimeMillsInQueue</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-11 16:59:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/ce/79/673f4268.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小杰</span>
  </div>
  <div class="_2_QraFYR_0">请教老师：<br>1、netstat里面的队列是tcp的发送和接受队列吗？<br>2、netty的发送缓存和tcp的发送缓存怎么个区别呢？怎么看这两个呢？<br>3、老师给的那个awk -F &#39;[, ,:]&#39; &#39;$12&gt;2&#39; r.log |wc -l 为啥是12列，咋看不懂呢？<br><br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-04 17:04:12</div>
  </div>
</div>
</div>
</li>
</ul>