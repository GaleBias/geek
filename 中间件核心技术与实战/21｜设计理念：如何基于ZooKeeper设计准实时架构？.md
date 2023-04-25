<audio title="21｜设计理念：如何基于ZooKeeper设计准实时架构？" src="https://static001.geekbang.org/resource/audio/e1/8e/e1453cyycd552e9128d9f04b7c75248e.mp3" controls="controls"></audio> 
<p>你好，我是丁威。</p><p>先跟你分享一段我的经历吧。记得我在尝试学习分布式调度框架时，因为我们公司采用的分布式调度框架是ElasticJob，所以我决定以ElasticJob为突破口，通过研读ElasticJob的源码，深入探究定时调度框架的实现原理。</p><p>在阅读ElasticJob源码的过程中，它灵活使用ZooKeeper来实现多进程协作的机制让我印象深刻，这里蕴藏着互联网一种通用的架构设计理念，那就是：基于ZooKeeper实现元信息配置管理与实时感知。</p><p>上节课中我们也重点提到过，ElasticJob可以实现分布式部署、并且支持数据分片，它同时还支持故障转移机制，其实这一切都是依托ZooKeeper来实现的。</p><h2>基于ZooKeeper的事件通知机制</h2><p>ElasticJob的架构采取的是去中心化设计，也就是说，ElasticJob在集群部署时，各个节点之间没有主从之分，它们的地位都是平等的。并且，ElasticJob的调度侧重对数据进行分布式处理（也就是数据分片机制），在调度每一个任务之前需要先计算分片信息，然后才能下发给集群内的其他节点来执行。实际部署效果图如下：</p><p><img src="https://static001.geekbang.org/resource/image/7c/25/7cdb21b91a10d1501f49ed7fdee2d925.jpg?wh=1920x646" alt="图片"></p><p>在这张图中，order-service-job应用中创建了两个定时任务job-1和job-2，而且order-service-job这个应用部署在两台机器上，也就是说，我们拥有两个调度执行器。那么问题来了，job-1和job-2的分片信息由哪个节点来计算呢？</p><!-- [[[read_end]]] --><p>在ElasticJob的实现中，并不是将分片的计算任务固定分配给某一个节点，而是以任务为维度允许各个调度器参与竞选，竞选成功的调度器成为该任务的Leader节点，竞选失败的节点成为备选节点。备选节点只能在Leader节点宕机时重新竞争，选举出新的Leader并接管前任Leader的工作，从而实现高可用。</p><p>那具体如何实现呢？原来，ElasticJob利用了ZooKeeper的强一致性与事件监听机制。</p><p>当一个任务需要被调度时，调度器会首先将任务注册到ZooKeeper中，具体操作为在ZooKeeper中创建对应的节点。ElasticJob中的任务在ZooKeeper中的目录布局如下：</p><p><img src="https://static001.geekbang.org/resource/image/b2/65/b29ee902d616533bb69b23e7cdf96865.jpg?wh=1920x1329" alt="图片"></p><p>简单说明一下各个节点的用途。</p><ul>
<li>config：存放任务的配置信息。</li>
<li>servers：存放任务调度器服务器IP地址。</li>
<li>instances：存放任务调度器实例（IP+进程）。</li>
<li>sharding：存放任务的分片信息。</li>
<li>leader/election/instance：存放任务的Leader信息。</li>
</ul><p>创建好对应的节点之后，就要根据不同的业务处理注册事件监听了。在ElasticJob中，根据不同的任务会创建如下事件监听管理器，从而完成核心功能：</p><p><img src="https://static001.geekbang.org/resource/image/80/ae/80f7a4191c11248c01f44ff30a5f25ae.png?wh=804x206" alt="图片"></p><p>我们这节课重点关注的是ElectionListenerManager的实现细节，掌握基于ZooKeeper事件通知的编程技巧。</p><p>ElectionListenerManager会在内部进行事件注册：</p><p><img src="https://static001.geekbang.org/resource/image/4c/1c/4c021dca25f495a67d4412558c3d561c.png?wh=1920x556" alt="图片"></p><p>事件注册的底层使用的是ZooKeeper的watch，每一个监听器在一个特定的节点处监听，一旦节点信息发生变化，ZooKeeper就会通知执行注册的事件监听器，执行对应的业务处理。</p><p><strong>一个节点信息的变化包括：节点创建、节点值内容变更、节点删除、子节点新增、子节点删除、子节点内容变更等。</strong></p><p>调度器监听了ZooKeeper中的任务节点之后，一旦任务节点下任何一个子节点发生变化，调度器Leader选举监听器就会得到通知，进而执行LeaderElectionJobListener的onChange方法，触发选举。</p><p>接下来我们结合核心代码实现，来学习一下如何使用Zookeeper来实现主节点选举。</p><p>ElasticJob直接使用了Apache Curator开源框架（ZooKeeper客户端API类库）提供的实现类（org.apache.curator.framework.recipes.leader.LeaderLatch），具体代码如下：</p><pre><code class="language-plain">public void executeInLeader(final String key, final LeaderExecutionCallback callback) {
 &nbsp; &nbsp; &nbsp; &nbsp;try (LeaderLatch latch = new LeaderLatch(client, key)) { // @1
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;latch.start(); // @2
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;latch.await();
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;callback.execute();
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;//CHECKSTYLE:OFF
 &nbsp; &nbsp; &nbsp;  } catch (final Exception ex) {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;//CHECKSTYLE:ON
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;handleException(ex);
 &nbsp; &nbsp; &nbsp;  }
 &nbsp;  }
</code></pre><p>我们解读一下关键代码。LeaderLatch需要传入两个参数：CuratorFramework client和latchPath。</p><p>CuratorFramework client是Curator的框架客户端。latchPath则是锁节点路径，ElasticJob的锁节点路径为：/{namespace}/{Jobname}/leader/election/latch。</p><p>启动LeaderLatch的start方法之后，ZooKeeper客户端会尝试去latchPath路径下创建一个<strong>临时顺序节点。</strong>如果创建的节点序号最小，LeaderLatch的await方法会返回后执行LeaderExecutionCallback的execute方法，如果存放具体实例的节点({namespace}/{jobname}/leader/election/instance)不存在，那就要创建这个临时节点，节点存储的内容为IP地址@-@进程ID，也就是说创建一个临时节点，记录当前任务的Leader信息，从而完成选举。</p><p>当Leader所在进程宕机后，在锁节点路径（/leader/election/latch）中创建的临时顺序节点会被删除，并且删除事件能够被其他节点感知，继而能够及时重新选举Leader，实现Leader的高可用。</p><p><img src="https://static001.geekbang.org/resource/image/f8/54/f8026e8e96763516aff25cff93cc6c54.jpg?wh=1920x1214" alt="图片"></p><p>经过上面两个步骤，我们就基于ZooKeeper轻松实现了分布式环境下集群的选举功能。我们再来总结一下基于ZooKeeper事件监听机制的编程要点。</p><ol>
<li>在Zookeeper中创建对应的节点。</li>
</ol><p>节点的类型通常分为临时节点与持久节点。如果是存放静态信息（例如配置信息），我们通常使用持久节点；如果是存储运行时信息，则要创建临时节点。当会话失效后，临时节点会自动删除。</p><ol start="2">
<li>在对应节点通过watch机制注册事件回调机制。</li>
</ol><p>如果你对这一机制感兴趣，建议你看看ElasticJob在这方面的源码，我的<a href="https://mp.weixin.qq.com/mp/homepage?__biz=MzIzNzgyMjYxOQ==&hid=3&sn=e09cc30d43246842fde82da2c7671553&devicetype=android-29&version=28001996&lang=zh_CN&nettype=WIFI&ascene=15&session_us=gh_1f5a9b23a34b&wx_header=3&scene=1">源码分析专栏 </a>应该也可以给你提供一些帮助。</p><h2>应用案例</h2><p>深入学习一款中间件，不仅能让我们了解中间件的底层实现细节，还能学到一些设计理念，那ElasticJob这种基于ZooKeeper实现元数据动态感知的设计模式会有哪些应用实战呢？</p><p>我想分享两个我在工作中遇到的实际场景。</p><h3>案例一</h3><p>2019年，我刚来到中通，我在负责的全链路压测项目需要在压测任务开启后自动启动影子消费组，然后等压测结束后，在不重启应用程序的情况下关闭影子消费组。我们在释放线程资源时，就用到了ZooKeeper的事件通知机制。</p><p>首先我们来图解一下当时的需求：</p><p><img src="https://static001.geekbang.org/resource/image/31/68/31b065dfc2ac0db83910b2a32eb0fd68.jpg?wh=1920x625" alt="图片"></p><p>我们解读一下具体的实现思路。</p><p>第一步，在压测任务配置界面中，提供对应的配置项，将本次压测任务需要关联的消费组存储到数据库中，同时持久到ZooKeeper的一个指定目录中，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/5e/38/5e53392c4fae74755e6f8964c250c738.png?wh=1794x722" alt="图片"></p><p>ZooKeeper中的目录设计结构如下。</p><ul>
<li>/zpt：全链路压测的根目录。</li>
<li>/zpt/order_service_consumer：应用Aphid。</li>
<li>/zpt/order_service_consumer/zpt_task_1：压测任务。</li>
<li>/zpt/order_service_consumer/zpt_task_1/order_bil_group：具体的消费组。</li>
</ul><p>在这里，每一个消费组节点存储的值为JSON格式，其中，从enable字段可以看出该消费组的影子消费组是否需要启动，默认为0表示不启动。</p><p>第二步，启动应用程序时，应用程序会根据应用自身的AppID去ZooKeeper中查找所有的消费组，提取出各个消费组的enable属性，如果enable属性如果为1，则立即启动影子消费组。</p><p>同时，我们还要监听/zpt/order_service_consumer节点，一旦该节点下任意一个子节点发生变化，zpt-sdk就能收到一个事件通知。</p><p>在需要进行全链路压测时，用户如果在全链路压测页面启动压测任务，就将该任务下消费组的enable属性设置为1，同时更新ZooKeeper中的值。一旦节点的值发生变化，zpt-sdk将收到一个节点变更事件，并启动对应消费组的影子消费组。</p><p>当停止全链路压测时，压测控制台将对应消费组在ZooKeeper中的值修改为0，这样zpt同样会收到一个事件通知，从而动态停止消费组。</p><p>这样，我们在不重启应用程序的情况下就实现了影子消费组的启动与停止。</p><p>注册事件的关键代码如下：</p><pre><code class="language-plain">private CuratorFramework client; // carator客户端
​
public static void addDataListener(String path, TreeCacheListener listener) { //注册事件监听
 &nbsp; &nbsp;TreeCache cache = instance.caches.get(path);
 &nbsp; &nbsp;if(cache == null ) {
 &nbsp; &nbsp; &nbsp; &nbsp; cache = addCacheData(path);
 &nbsp;  }
 &nbsp; &nbsp;cache.getListenable().addListener(listener);
}
</code></pre><p>事件监听器中的关键代码如下：</p><pre><code class="language-plain">class MqConsumerGroupDataNodeListener extends TreeCacheListener {
 &nbsp; &nbsp; &nbsp; &nbsp;protected void dataChanged(String path, TreeCacheEvent.Type eventType, String data) {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;//首先触发事件的节点，判断路径是否为消费组的路径，如果不是，忽略本次处理
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;if(StringUtils.isBlank(path) || !ZptNodePath.isMQConsumerGroupDataNode(path)) {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;logger.warn(String.format("path:%s is empty or is not a consumerGroup path.", path));
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;return;
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;  }
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;try {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;String consumerGroup = getLastKey(path);
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;if(logger.isDebugEnabled()) {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;logger.debug(String.format("节点path:%s,节点值:%s", path, data));
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;  }
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;if(!Zpt.isConsumerGroup(consumerGroup)) {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;logger.info(String.format("消费组:%s,不属于当前应用提供的，故无需订阅", consumerGroup));
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;return;
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;  }
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;    // 如果节点的变更类型为删除，则直接停止消费组
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;if(StringUtils.isBlank(data) || TreeCacheEvent.Type.NODE_REMOVED.equals(eventType)) {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;invokeListener(consumerGroup, false);
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;  }
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;// 取得节点的值
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;MqConsumerVo mqVo = JsonUtil.parse(data, MqConsumerVo.class);
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;// 如果为空，或则为0，则停止消费组
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;if(mqVo == null || StringUtils.isBlank(mqVo.getEnable()) || "0".equals(mqVo.getEnable())) {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;invokeListener(consumerGroup, false);
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;return;
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;  } else { // 否则启动消费组。
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;invokeListener(consumerGroup, true);
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;return;
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;  }
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;  } catch (Throwable e) {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;logger.error("zk mq consumerGroup manager dataChange error", e);
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;  }
​
 &nbsp; &nbsp; &nbsp;  }
 &nbsp;  }
</code></pre><h3>案例二</h3><p>在这节课的最后，我们再看一下另外一个案例：消息中间件SDK的核心设计理念。</p><p>我们公司对消息中间件的消息发送与消息消费做了统一的封装，对用户弱化了集群的概念，用户发送、消费消息时，不需要知道主题所在的集群地址，相关的API如下所示：</p><pre><code class="language-plain">public static SendResult send(String topic, SimpleMessage simpleMessage) 
public static void subscribe(String consumerGroup, MessageListener listener)
</code></pre><p>那问题来了，我们在调用消息发送API时，如何正确路由到真实的消息集群呢？</p><p>其实，我们公司对主题、消费组进行了强管控，项目组在使用主题、消费组之前，需要通过消息运维平台进行申请，审批通过后会将主题分配到对应的物理集群上，并会将topic的元数据分别存储到数据库和ZooKeeper中。因为这属于配置类信息，所以这一类节点会创建为持久化节点。</p><p>这样，消息发送 API 在初次发送主题时，会根据主题的名称在ZooKeeper中查找主题的元信息，包括主题的类型（RocketMQ/Kafka）、所在的集群地址（NameServer地址或Kafka Broker地址）等，然后构建对应的消息发送客户端进行消息发送。</p><p>那我们为什么要将主题、消费组的信息存储到ZooKeeper中呢？</p><p>这是因为，为了便于高效运维，我们对主题、消费组的使用方屏蔽了集群相关的信息，你可以看看下面这个场景：</p><p><img src="https://static001.geekbang.org/resource/image/9d/74/9d02103fa5faa197a4yyb63a1f4f3674.jpg?wh=1920x747" alt="图片"></p><p>你能在不重启应用的情况下将order_topic从A集群迁移到B集群吗？</p><p>没错，在我们这种架构下，将主题从一个集群迁移到另外一个集群将变得非常简单。</p><p>我们只需要在ZooKeeper中修改一下order_topic的元信息，将维护的集群的信息由集群A变更为集群B，然后zms-sdk在监听order_topic对应的主题节点时，就能收到主题元信息变更事件了。然后zms-sdk会基于新的元信息重新构建一个MQ Producer对象，再关闭老的生产者，这样就实现了主题流量的无缝迁移，快速进行故障恢复，极大程度保证了系统的高可用性。</p><p>我们公司已经把这个项目开源了，具体的实现代码你可以打开链接查看（<a href="https://github.com/ZTO-Express/zms/tree/master/zms-client/src/main/java/com/zto/zms/client/producer">ZMS开源项目</a>）。</p><p><img src="https://static001.geekbang.org/resource/image/7e/0c/7ebac62759557035b675137815dd660c.png?wh=997x576" alt="图片"></p><h2>总结</h2><p>好了，这节课我们就介绍到这里了。</p><p>这节课我们通过ElasticJob分布式环境中的集群部署，引出了ZooKeeper来实现多进程协作机制。并着重介绍了基于ZooKeeper实现Leader选举的方法。我们还总结出了一套互联网中常用的设计模式：基于ZooKeeper的事件通知机制。</p><p>我还结合我工作中两个真实的技术需求，将ZooKeeper作为配置中心，结合事件监听机制实现了不重启项目，在不重启应用程序的情况下，完成了影子消费组和消息发送者的启动与停止。</p><h2>课后题</h2><p>最后我也给你留一道题。</p><p>请你尝试编写一个功能，使用curator开源类库，去监听ZooKeeper中的一个节点，打印节点的名称，并能动态感知节点内容的变化、子节点列表的变化。程序编写后，可以通过ZooKeeper运维命令去操作节点，从而验证程序的输出值是否正确。</p><p>欢迎你在留言区与我交流讨论，我们下节课再见。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/bd/f6/558bb119.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ᯤ⁵ᴳ</span>
  </div>
  <div class="_2_QraFYR_0">快结课了，希望老师也讲讲分库分表的中间件</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 分库分表，我在我的「中间件兴趣圈」上发表了一个mycat专栏，你可以先看看，兴许能为你打开一扇窗。<br><br>目前我还没有深入去研究shardingjdbc，我今年下半年的计划是深入研究kafka与系统监控指标、网络等知识，这些都会在我的个人公众号上连载，希望你保持对我的关注，我们一起努力，多多交流。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-14 10:54:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/55/bf/d442b55e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>mikewt</span>
  </div>
  <div class="_2_QraFYR_0">老师 watch机制其实可以用客户端主动轮询来解决，那么这两者有啥区别吗，使用场景是什么</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，万变不离其宗，目前客户端、服务端通常有两种交互模式：推与拉。<br>推：服务端主动通知客户端。<br>拉：客户端定时主动查询服务端。<br>两者一个最大的区别：推模式，实时性更强，一旦服务端数据有变化，通常会及时推送给客户端，即客户端能及时感知。而拉模式服务端数据更新后，客户端并不会及时知道，需要定时去查询。当然在拉模式中，还可以引入一种机制，叫长轮循，我建议你去百度上查一下长轮询。<br><br>另外就是一个实现难度的问题，通常推模式实现起来复杂，拉模式简单。<br><br>至于场景的话，Dubbo使用Zookeeper当注册中心，使用了推模式，而RocketMQ的nameserver使用了拉模式。<br><br>一个宗旨是如果拉模式可以满足需求，或者不会带来太大的问题，或者这些问题可以简单的规避，通常可以使用拉模式，关于推拉模式，我建议你再看看我这篇文章：https:&#47;&#47;mp.weixin.qq.com&#47;s&#47;76DibRKtywdgYTQDgKuoQQ<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-10 08:11:00</div>
  </div>
</div>
</div>
</li>
</ul>