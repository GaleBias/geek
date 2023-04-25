<audio title="23 _ RocketMQ客户端如何在集群中找到正确的节点？" src="https://static001.geekbang.org/resource/audio/10/c4/101d5b238d6e34039294281d2b91ddc4.mp3" controls="controls"></audio> 
<p>你好，我是李玥。</p><p>我们在《<a href="https://time.geekbang.org/column/article/135120">21 | RocketMQ Producer源码分析：消息生产的实现过程</a>》这节课中，讲解RocketMQ的生产者启动流程时提到过，生产者只要配置一个接入地址，就可以访问整个集群，并不需要客户端配置每个Broker的地址。RocketMQ会自动根据要访问的主题名称和队列序号，找到对应的Broker地址。如果Broker发生宕机，客户端还会自动切换到新的Broker节点上，这些对于用户代码来说都是透明的。</p><p>这些功能都是由NameServer协调Broker和客户端共同实现的，其中NameServer的作用是最关键的。</p><p>展开来讲，不仅仅是RocketMQ，任何一个弹性分布式集群，都需要一个类似于NameServer服务，来帮助访问集群的客户端寻找集群中的节点，这个服务一般称为NamingService。比如，像Dubbo这种RPC框架，它的注册中心就承担了NamingService的职责。在Flink中，则是JobManager承担了NamingService的职责。</p><p>也就是说，这种使用NamingService服务来协调集群的设计，在分布式集群的架构设计中，是一种非常通用的方法。你在学习这节课之后，不仅要掌握RocketMQ的NameServer是如何实现的，还要能总结出通用的NamingService的设计思想，并能应用于其他分布式系统的设计中。</p><!-- [[[read_end]]] --><p>这节课，我们一起来分析一下NameServer的源代码，看一下NameServer是如何协调集群中众多的Broker和客户端的。</p><h2>NameServer是如何提供服务的？</h2><p>在RocketMQ中，NameServer是一个独立的进程，为Broker、生产者和消费者提供服务。NameServer最主要的功能就是，为客户端提供寻址服务，协助客户端找到主题对应的Broker地址。此外，NameServer还负责监控每个Broker的存活状态。</p><p>NameServer支持只部署一个节点，也支持部署多个节点组成一个集群，这样可以避免单点故障。在集群模式下，NameServer各节点之间是不需要任何通信的，也不会通过任何方式互相感知，每个节点都可以独立提供全部服务。</p><p>我们一起通过这个图来看一下，在RocketMQ集群中，NameServer是如何配合Broker、生产者和消费者一起工作的。这个图来自<a href="https://github.com/apache/rocketmq/tree/master/docs">RocketMQ的官方文档</a>。</p><p><img src="https://static001.geekbang.org/resource/image/53/5e/53baeb70d388de042f7347d137b9d35e.jpeg?wh=2208*916" alt=""></p><p>每个Broker都需要和所有的NameServer节点进行通信。当Broker保存的Topic信息发生变化的时候，它会主动通知所有的NameServer更新路由信息，为了保证数据一致性，Broker还会定时给所有的NameServer节点上报路由信息。这个上报路由信息的RPC请求，也同时起到Broker与NameServer之间的心跳作用，NameServer依靠这个心跳来确定Broker的健康状态。</p><p>因为每个NameServer节点都可以独立提供完整的服务，所以，对于客户端来说，包括生产者和消费者，只需要选择任意一个NameServer节点来查询路由信息就可以了。客户端在生产或消费某个主题的消息之前，会先从NameServer上查询这个主题的路由信息，然后根据路由信息获取到当前主题和队列对应的Broker物理地址，再连接到Broker节点上进行生产或消费。</p><p>如果NameServer检测到与Broker的连接中断了，NameServer会认为这个Broker不再能提供服务。NameServer会立即把这个Broker从路由信息中移除掉，避免客户端连接到一个不可用的Broker上去。而客户端在与Broker通信失败之后，会重新去NameServer上拉取路由信息，然后连接到其他Broker上继续生产或消费消息，这样就实现了自动切换失效Broker的功能。</p><p>此外，NameServer还提供一个类似Redis的KV读写服务，这个不是主要的流程，我们不展开讲。</p><p>接下来我带你一起分析NameServer的源代码，看一下这些服务都是如何实现的。</p><h2>NameServer的总体结构</h2><p>由于NameServer的结构非常简单，排除KV读写相关的类之后，一共只有6个类，这里面直接给出这6个类的说明：</p><ul>
<li><strong>NamesrvStartup</strong>：程序入口。</li>
<li><strong>NamesrvController</strong>：NameServer的总控制器，负责所有服务的生命周期管理。</li>
<li><strong>RouteInfoManager</strong>：NameServer最核心的实现类，负责保存和管理集群路由信息。</li>
<li><strong>BrokerHousekeepingService</strong>：监控Broker连接状态的代理类。</li>
<li><strong>DefaultRequestProcessor</strong>：负责处理客户端和Broker发送过来的RPC请求的处理器。</li>
<li><strong>ClusterTestRequestProcessor</strong>：用于测试的请求处理器。</li>
</ul><p>RouteInfoManager这个类中保存了所有的路由信息，这些路由信息都是保存在内存中，并且没有持久化的。在代码中，这些路由信息保存在RouteInfoManager的几个成员变量中：</p><pre><code>public class BrokerData implements Comparable&lt;BrokerData&gt; {
  // ...
  private final HashMap&lt;String/* topic */, List&lt;QueueData&gt;&gt; topicQueueTable;
  private final HashMap&lt;String/* brokerName */, BrokerData&gt; brokerAddrTable;
  private final HashMap&lt;String/* clusterName */, Set&lt;String/* brokerName */&gt;&gt; clusterAddrTable;
  private final HashMap&lt;String/* brokerAddr */, BrokerLiveInfo&gt; brokerLiveTable;
  private final HashMap&lt;String/* brokerAddr */, List&lt;String&gt;/* Filter Server */&gt; filterServerTable;
  // ...
}
</code></pre><p>以上代码中的这5个Map对象，保存了集群所有的Broker和主题的路由信息。</p><p>topicQueueTable保存的是主题和队列信息，其中每个队列信息对应的类QueueData中，还保存了brokerName。需要注意的是，这个brokerName并不真正是某个Broker的物理地址，它对应的一组Broker节点，包括一个主节点和若干个从节点。</p><p>brokerAddrTable中保存了集群中每个brokerName对应Broker信息，每个Broker信息用一个BrokerData对象表示：</p><pre><code>public class BrokerData implements Comparable&lt;BrokerData&gt; {
    private String cluster;
    private String brokerName;
    private HashMap&lt;Long/* brokerId */, String/* broker address */&gt; brokerAddrs;
    // ...
}
</code></pre><p>BrokerData中保存了集群名称cluster，brokerName和一个保存Broker物理地址的Map：brokerAddrs，它的Key是BrokerID，Value就是这个BrokerID对应的Broker的物理地址。</p><p>下面这三个map相对没那么重要，简单说明如下：</p><ul>
<li>brokerLiveTable中，保存了每个Broker当前的动态信息，包括心跳更新时间，路由数据版本等等。</li>
<li>clusterAddrTable中，保存的是集群名称与BrokerName的对应关系。</li>
<li>filterServerTable中，保存了每个Broker对应的消息过滤服务的地址，用于服务端消息过滤。</li>
</ul><p>可以看到，在NameServer的RouteInfoManager中，主要的路由信息就是由topicQueueTable和brokerAddrTable这两个Map来保存的。</p><p>在了解了总体结构和数据结构之后，我们再来看一下实现的流程。</p><h2>NameServer如何处理Broker注册的路由信息？</h2><p>首先来看一下，NameServer是如何处理Broker注册的路由信息的。</p><p>NameServer处理Broker和客户端所有RPC请求的入口方法是：“DefaultRequestProcessor#processRequest”，其中处理Broker注册请求的代码如下：</p><pre><code>public class DefaultRequestProcessor implements NettyRequestProcessor {
    // ...
    @Override
    public RemotingCommand processRequest(ChannelHandlerContext ctx,
        RemotingCommand request) throws RemotingCommandException {
        // ...
        switch (request.getCode()) {
            // ...
            case RequestCode.REGISTER_BROKER:
                Version brokerVersion = MQVersion.value2Version(request.getVersion());
                if (brokerVersion.ordinal() &gt;= MQVersion.Version.V3_0_11.ordinal()) {
                    return this.registerBrokerWithFilterServer(ctx, request);
                } else {
                    return this.registerBroker(ctx, request);
                }
            // ...
            default:
                break;
        }
        return null;
    }
    // ...
}
</code></pre><p>这是一个非常典型的处理Request的路由分发器，根据request.getCode()来分发请求到对应的处理器中。Broker发给NameServer注册请求的Code为REGISTER_BROKER，在代码中根据Broker的版本号不同，分别有两个不同的处理实现方法：“registerBrokerWithFilterServer”和"registerBroker"。这两个方法实现的流程是差不多的，实际上都是调用了"RouteInfoManager#registerBroker"方法，我们直接看这个方法的代码：</p><pre><code>public RegisterBrokerResult registerBroker(
    final String clusterName,
    final String brokerAddr,
    final String brokerName,
    final long brokerId,
    final String haServerAddr,
    final TopicConfigSerializeWrapper topicConfigWrapper,
    final List&lt;String&gt; filterServerList,
    final Channel channel) {
    RegisterBrokerResult result = new RegisterBrokerResult();
    try {
        try {
            // 加写锁，防止并发修改数据
            this.lock.writeLock().lockInterruptibly();

            // 更新clusterAddrTable
            Set&lt;String&gt; brokerNames = this.clusterAddrTable.get(clusterName);
            if (null == brokerNames) {
                brokerNames = new HashSet&lt;String&gt;();
                this.clusterAddrTable.put(clusterName, brokerNames);
            }
            brokerNames.add(brokerName);

            // 更新brokerAddrTable
            boolean registerFirst = false;

            BrokerData brokerData = this.brokerAddrTable.get(brokerName);
            if (null == brokerData) {
                registerFirst = true; // 标识需要先注册
                brokerData = new BrokerData(clusterName, brokerName, new HashMap&lt;Long, String&gt;());
                this.brokerAddrTable.put(brokerName, brokerData);
            }
            Map&lt;Long, String&gt; brokerAddrsMap = brokerData.getBrokerAddrs();
            // 更新brokerAddrTable中的brokerData
            Iterator&lt;Entry&lt;Long, String&gt;&gt; it = brokerAddrsMap.entrySet().iterator();
            while (it.hasNext()) {
                Entry&lt;Long, String&gt; item = it.next();
                if (null != brokerAddr &amp;&amp; brokerAddr.equals(item.getValue()) &amp;&amp; brokerId != item.getKey()) {
                    it.remove();
                }
            }

            // 如果是新注册的Master Broker，或者Broker中的路由信息变了，需要更新topicQueueTable
            String oldAddr = brokerData.getBrokerAddrs().put(brokerId, brokerAddr);
            registerFirst = registerFirst || (null == oldAddr);

            if (null != topicConfigWrapper
                &amp;&amp; MixAll.MASTER_ID == brokerId) {
                if (this.isBrokerTopicConfigChanged(brokerAddr, topicConfigWrapper.getDataVersion())
                    || registerFirst) {
                    ConcurrentMap&lt;String, TopicConfig&gt; tcTable =
                        topicConfigWrapper.getTopicConfigTable();
                    if (tcTable != null) {
                        for (Map.Entry&lt;String, TopicConfig&gt; entry : tcTable.entrySet()) {
                            this.createAndUpdateQueueData(brokerName, entry.getValue());
                        }
                    }
                }
            }

            // 更新brokerLiveTable
            BrokerLiveInfo prevBrokerLiveInfo = this.brokerLiveTable.put(brokerAddr,
                new BrokerLiveInfo(
                    System.currentTimeMillis(),
                    topicConfigWrapper.getDataVersion(),
                    channel,
                    haServerAddr));
            if (null == prevBrokerLiveInfo) {
                log.info(&quot;new broker registered, {} HAServer: {}&quot;, brokerAddr, haServerAddr);
            }

            // 更新filterServerTable
            if (filterServerList != null) {
                if (filterServerList.isEmpty()) {
                    this.filterServerTable.remove(brokerAddr);
                } else {
                    this.filterServerTable.put(brokerAddr, filterServerList);
                }
            }

            // 如果是Slave Broker，需要在返回的信息中带上master的相关信息
            if (MixAll.MASTER_ID != brokerId) {
                String masterAddr = brokerData.getBrokerAddrs().get(MixAll.MASTER_ID);
                if (masterAddr != null) {
                    BrokerLiveInfo brokerLiveInfo = this.brokerLiveTable.get(masterAddr);
                    if (brokerLiveInfo != null) {
                        result.setHaServerAddr(brokerLiveInfo.getHaServerAddr());
                        result.setMasterAddr(masterAddr);
                    }
                }
            }
        } finally {
            // 释放写锁
            this.lock.writeLock().unlock();
        }
    } catch (Exception e) {
        log.error(&quot;registerBroker Exception&quot;, e);
    }

    return result;
}
</code></pre><p>上面这段代码比较长，但总体结构很简单，就是根据Broker请求过来的路由信息，依次对比并更新clusterAddrTable、brokerAddrTable、topicQueueTable、brokerLiveTable和filterServerTable这5个保存集群信息和路由信息的Map对象中的数据。</p><p>另外，在RouteInfoManager中，这5个Map作为一个整体资源，使用了一个读写锁来做并发控制，避免并发更新和更新过程中读到不一致的数据问题。这个读写锁的使用方法，和我们在之前的课程《<a href="https://time.geekbang.org/column/article/129333">17 | 如何正确使用锁保护共享数据，协调异步线程？</a>》中讲到的方法是一样的。</p><h2>客户端如何寻找Broker？</h2><p>下面我们来看一下，NameServer如何帮助客户端来找到对应的Broker。对于客户端来说，无论是生产者还是消费者，通过主题来寻找Broker的流程是一样的，使用的也是同一份实现。客户端在启动后，会启动一个定时器，定期从NameServer上拉取相关主题的路由信息，然后缓存在本地内存中，在需要的时候使用。每个主题的路由信息用一个TopicRouteData对象来表示：</p><pre><code>public class TopicRouteData extends RemotingSerializable {
    // ...
    private List&lt;QueueData&gt; queueDatas;
    private List&lt;BrokerData&gt; brokerDatas;
    // ...
}
</code></pre><p>其中，queueDatas保存了主题中的所有队列信息，brokerDatas中保存了主题相关的所有Broker信息。客户端选定了队列后，可以在对应的QueueData中找到对应的BrokerName，然后用这个BrokerName找到对应的BrokerData对象，最终找到对应的Master Broker的物理地址。这部分代码在org.apache.rocketmq.client.impl.factory.MQClientInstance这个类中，你可以自行查看。</p><p>下面我们看一下在NameServer中，是如何实现根据主题来查询TopicRouteData的。</p><p>NameServer处理客户端请求和处理Broker请求的流程是一样的，都是通过路由分发器将请求分发的对应的处理方法中，我们直接看具体的实现方法RouteInfoManager#pickupTopicRouteData：</p><pre><code>public TopicRouteData pickupTopicRouteData(final String topic) {

    // 初始化返回数据topicRouteData
    TopicRouteData topicRouteData = new TopicRouteData();
    boolean foundQueueData = false;
    boolean foundBrokerData = false;
    Set&lt;String&gt; brokerNameSet = new HashSet&lt;String&gt;();
    List&lt;BrokerData&gt; brokerDataList = new LinkedList&lt;BrokerData&gt;();
    topicRouteData.setBrokerDatas(brokerDataList);

    HashMap&lt;String, List&lt;String&gt;&gt; filterServerMap = new HashMap&lt;String, List&lt;String&gt;&gt;();
    topicRouteData.setFilterServerTable(filterServerMap);

    try {
        try {

            // 加读锁
            this.lock.readLock().lockInterruptibly();

            //先获取主题对应的队列信息
            List&lt;QueueData&gt; queueDataList = this.topicQueueTable.get(topic);
            if (queueDataList != null) {

                // 把队列信息返回值中
                topicRouteData.setQueueDatas(queueDataList);
                foundQueueData = true;

                // 遍历队列，找出相关的所有BrokerName
                Iterator&lt;QueueData&gt; it = queueDataList.iterator();
                while (it.hasNext()) {
                    QueueData qd = it.next();
                    brokerNameSet.add(qd.getBrokerName());
                }

                // 遍历这些BrokerName，找到对应的BrokerData，并写入返回结果中
                for (String brokerName : brokerNameSet) {
                    BrokerData brokerData = this.brokerAddrTable.get(brokerName);
                    if (null != brokerData) {
                        BrokerData brokerDataClone = new BrokerData(brokerData.getCluster(), brokerData.getBrokerName(), (HashMap&lt;Long, String&gt;) brokerData
                            .getBrokerAddrs().clone());
                        brokerDataList.add(brokerDataClone);
                        foundBrokerData = true;
                        for (final String brokerAddr : brokerDataClone.getBrokerAddrs().values()) {
                            List&lt;String&gt; filterServerList = this.filterServerTable.get(brokerAddr);
                            filterServerMap.put(brokerAddr, filterServerList);
                        }
                    }
                }
            }
        } finally {
            // 释放读锁
            this.lock.readLock().unlock();
        }
    } catch (Exception e) {
        log.error(&quot;pickupTopicRouteData Exception&quot;, e);
    }

    log.debug(&quot;pickupTopicRouteData {} {}&quot;, topic, topicRouteData);

    if (foundBrokerData &amp;&amp; foundQueueData) {
        return topicRouteData;
    }

    return null;
}
</code></pre><p>这个方法的实现流程是这样的：</p><ol>
<li>初始化返回的topicRouteData后，获取读锁。</li>
<li>在topicQueueTable中获取主题对应的队列信息，并写入返回结果中。</li>
<li>遍历队列，找出相关的所有BrokerName。</li>
<li>遍历这些BrokerName，从brokerAddrTable中找到对应的BrokerData，并写入返回结果中。</li>
<li>释放读锁并返回结果。</li>
</ol><h2>小结</h2><p>这节课我们一起分析了RocketMQ NameServer的源代码，NameServer在集群中起到的一个核心作用就是，为客户端提供路由信息，帮助客户端找到对应的Broker。</p><p>每个NameServer节点上都保存了集群所有Broker的路由信息，可以独立提供服务。Broker会与所有NameServer节点建立长连接，定期上报Broker的路由信息。客户端会选择连接某一个NameServer节点，定期获取订阅主题的路由信息，用于Broker寻址。</p><p>NameServer的所有核心功能都是在RouteInfoManager这个类中实现的，这类中使用了几个Map来在内存中保存集群中所有Broker的路由信息。</p><p>我们还一起分析了RouteInfoManager中的两个比较关键的方法：注册Broker路由信息的方法registerBroker，以及查询Broker路由信息的方法pickupTopicRouteData。</p><p>建议你仔细读一下这两个方法的代码，结合保存路由信息的几个Map的数据结构，体会一下RocketMQ NameServer这种简洁的设计。</p><p>把以上的这些NameServer的设计和实现方法抽象一下，我们就可以总结出通用的NamingService的设计思想。</p><p>NamingService负责保存集群内所有节点的路由信息，NamingService本身也是一个小集群，由多个NamingService节点组成。这里我们所说的“路由信息”也是一种通用的抽象，含义是：“客户端需要访问的某个特定服务在哪个节点上”。</p><p>集群中的节点主动连接NamingService服务，注册自身的路由信息。给客户端提供路由寻址服务的方式可以有两种，一种是客户端直接连接NamingService服务查询路由信息，另一种是，客户端连接集群内任意节点查询路由信息，节点再从自身的缓存或者从NamingService上进行查询。</p><p>掌握了以上这些NamingService的设计方法，将会非常有助于你理解其他分布式系统的架构，当然，你也可以把这些方法应用到分布式系统的设计中去。</p><h2>思考题</h2><p>今天的思考题是这样的，在RocketMQ的NameServer集群中，各节点之间不需要互相通信，每个节点都可以独立的提供服务。课后请你想一想，这种独特的集群架构有什么优势，又有什么不足？欢迎在评论区留言写下你的想法。</p><p>感谢阅读，如果你觉得这篇文章对你有一些启发，也欢迎把它分享给你的朋友。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4b/4d/0239bc19.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>益军</span>
  </div>
  <div class="_2_QraFYR_0">优点:  nameserver本身设计为无状态，实现简单，<br>缺点: broker客户端通信成本复杂，适合在客户端环境完全可控的情况下设计。namesrv 一致性无法保证，需要定时幂等性心跳保持最终一致性。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-20 15:26:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/30/8a/b5ca7286.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>业余草</span>
  </div>
  <div class="_2_QraFYR_0">线上环境突发消息延迟2个小时，该如何尽快解决？以及后期如何避免这类问题？说说你的思路和经验！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个我在之前的课程中讲到过，首先需要先看一下是消费慢还是生产慢，如果是生产慢，一般需要扩容Producer的节点数量，如果是消费慢，需要扩容队列数和Consumer数量。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-17 08:34:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/99/af/d29273e2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>饭粒</span>
  </div>
  <div class="_2_QraFYR_0">优点：实现简单，集群节点平等，比较容易的水平扩展节点数量提供高可用性。路由数据读写都是内存，QPS比较高。<br>缺点：每个 broker 需要与所有 nameserver 节点心跳通信，通信成本较大，无法保证强一致性。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-17 00:31:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/9c/9b/eec0d41f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>康师傅</span>
  </div>
  <div class="_2_QraFYR_0">对于rocketmq和kafka而言，都有自己的“注册中心”，但对于rabbitmq而言，它的集群允许你连接到集群中的任何一台进行生产消费，即便队列master所在节点并不是你连接的这台，rabbitmq内部会帮你进行中转，但这会有一个很大的弊端，就是节点间会有较大的流量并且不可控，并且整体的性能会受影响<br><br>想请问下，这种时候，是否有较好的优化方式？<br>想到的一个办法是，自行做一个NameServer，按队列进行注册分流，生产者消费者先与NameServer交互得到真实的节点ip，然后直接连接到队列master所在节点进行生产消费</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 理论上是可以的，但权衡工作量来说，最好还是换一个MQ吧。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-19 23:56:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/d0/42/6fd01fb9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我已经设置了昵称</span>
  </div>
  <div class="_2_QraFYR_0">老师，我们生产环境。同一个group，多台机器消费同一个topic消息，遇到了某几个实例消费的分区延迟特别高，导致消息堆积的情况。但另外几个实例消费的分区并没有堆积。这通过扩容consumer也没法解决。而且很难排查，有什么好的方式吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 一是查一下每个分区入队速度是不是均匀，另外在消费者加一个消费监控（记录每次消费的时延）看一下消费速度是不是有问题。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-25 11:17:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/99/af/d29273e2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>饭粒</span>
  </div>
  <div class="_2_QraFYR_0">请问下老师，如果 nameserver 有节点重启了或是新加了一个节点，恢复内存中的路由数据过程是通过 broker 的心跳上报路由信息重新注册一遍吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，RocketMQ的NameServer数据是以Broker上的为准，并且是从Broker上同步过来的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-17 00:20:54</div>
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
  <div class="_2_QraFYR_0">NamingService集群有点像去中心化的结构设计，每个节点保存所有数据，很好的保证了节点的可用性，但每个节点之间不互相通信，很难确保节点间的数据一致性。想问下老师主题(和其中队列)在broker节点的分布情况是怎样的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 主题和队列是分散到所有Broker节点上的，每个Broker只保存自己负责的那部分主题和队列信息。并且实际上在集群中，元数据最终是以Broker上保存的信息为准的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-19 00:17:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/34/df/64e3d533.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>leslie</span>
  </div>
  <div class="_2_QraFYR_0">    老师最近的课都是在啃代码且非常整体性：学习上是越来越辛苦了，总要翻阅和梳理知识才能明白题目可能的问题。<br>    节点之间不互相通信其实减少了网络开销以及相互的等待确认的过程从而节约了时间,不会互相影响互相继承：换个角度来思考这个问题其实就像是我们用虚拟化一样，RocketMQ的这种NameServer就像是docker&#47;Kubernetes，各自出了问题不会影响其它的docker&#47;K8。<br>   优势就是减少了节点之间的通信以及等待的代价，不足就是出了问题如何发现以及通知系统&#47;服务端。等待老师对于这个问题的答案的公布：等待明天老师继续的分享，谢谢。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-18 21:13:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/cf/b5/d1ec6a7d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Stalary</span>
  </div>
  <div class="_2_QraFYR_0">老师，我想问一下，如果起了很多个NameServer，都保持长连接的话是不是开销会较大呢，为什么没有采用订阅发布的模式去更新broker呢，是因为即时性吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这又是一个设计选择而已，NameServer只是负责存储一下元数据，数据量不大，处理请求的TPS也不高，所以没必要启动很多个NameServer，所以并不会存在你说的很多个NameServer的情况。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-17 08:24:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/10/7d/a9b5d5f0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>糖醋🏀</span>
  </div>
  <div class="_2_QraFYR_0">nnameserver各个节点独立不通信，是ap的思路。<br>各个节点总是可用，但是节点之间不通信，有可能由于网络原因，某个节点的路由信息可能会不一致。<br>客户端拉去所有节点的路由信息，可以弥补某个节点路由信息不一致的情况。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-17 08:46:11</div>
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
  <div class="_2_QraFYR_0">NameServer是单独存在的服务吗，像Kafka和Zookpeer？RocketMQ好像没有Zookpeer，它是怎么做的？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-02 13:51:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2c/73/7f/9088256b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>莫名其妙的人</span>
  </div>
  <div class="_2_QraFYR_0">文章没有提到客户端与哪个NameServer节点通信，看了一些网上说法，应该是客户端只需要链接NameServer集群地址，然后产生一个随机数取模（失败则轮训即可）。这种随机选择策略不会产生一些不均衡吗？为什么不适用负载均衡，动态路由等方式呢？<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-09 23:29:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/d9/ff/b23018a6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Heaven</span>
  </div>
  <div class="_2_QraFYR_0">1.越简单的设计,健壮性越好,越单一的功能,性能越好,这样设计必然提供了更高的性能<br>2,但是容易出现无法保证一致性的问题,但是我想,可能RMQ也不在乎</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-18 11:36:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/e0/6b/f61d7466.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>prader26</span>
  </div>
  <div class="_2_QraFYR_0"> rocketmq 如何在集群中找到正确的节点。<br> <br> 核心是利用的NameServer 和broker 联合实现的。<br> 1 每个broker 都和全部的NameServer 进行通信。<br> 2 当broker 上Topic信息发生改变的时候，会通知所有的NameServer 更新路由信息，同时broker 也会定时把信息<br>   上报到所有的NameServer节点。这个就起到了NameServer对broker 进行健康检测的作用。<br>3  每台NameServer 都能单独提供服务。当NameServer 和broker 之间的通信断掉，消费者会重新去NameServer上拉<br>  取别的broker 信息，这样就起到了自动切换失效Broker的作用。   </div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-08 11:24:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>信大捷安</span>
  </div>
  <div class="_2_QraFYR_0">看了留言中，nameServer可以做一个集群，topic是散列在nameServer集群中的一个服务中，那么是如何保证topic正确的散列的其中的一个nameServer中的，很好奇？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-18 17:45:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/8c/5c/3f164f66.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>亚林</span>
  </div>
  <div class="_2_QraFYR_0">牺牲一致性，提高可用性</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-14 08:38:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/a0/08/065b3cf5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Yippee</span>
  </div>
  <div class="_2_QraFYR_0">想问下老师文中讲的代码是基于 RocketMQ 的哪个版本啊，我在 4.5.1 和 4.7.0 中找不到 RouteInfoManager 等类</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我们使用当前最新的 release 版本 release-4.5.1 进行分析，使用 Git 在 GitHub 上直接下载源码到本地：<br><br>git clone git@github.com:apache&#47;rocketmq.git<br>cd rocketmq<br>git checkout release-4.5.1</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-11 09:25:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/6d/46/e16291f8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>丁小明</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，有个疑问就是如果客户端链接的那个nameserver不可用了怎么办呢，如果broker也恰好有变动，那这些客户端是不是也都不可用了。且无法自动恢复</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 所以一般生产环境都使用多个NameServer节点来避免单点故障。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-06 20:50:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/2d/ca/02b0e397.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>fomy</span>
  </div>
  <div class="_2_QraFYR_0">假如其中一台NameServer挂了，客户端会自动切换到其他的吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-17 15:00:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/57/f6/2c7ac1ad.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Peter</span>
  </div>
  <div class="_2_QraFYR_0">优点是性能高，Broker只需关心单个节点，而不需要关注集群状态，客户端只需要和一个NameServer打交道，不用关心集群状态。<br>缺点是不能保证一致性</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-01 18:48:19</div>
  </div>
</div>
</div>
</li>
</ul>