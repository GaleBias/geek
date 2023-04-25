<audio title="32 _ KafkaAdminClient：Kafka的运维利器" src="https://static001.geekbang.org/resource/audio/bc/8c/bcd8560eaf99793a712e26c9e100118c.mp3" controls="controls"></audio> 
<p>你好，我是胡夕。今天我要和你分享的主题是：Kafka的运维利器KafkaAdminClient。</p><h2>引入原因</h2><p>在上一讲中，我向你介绍了Kafka自带的各种命令行脚本，这些脚本使用起来虽然方便，却有一些弊端。</p><p>首先，不论是Windows平台，还是Linux平台，命令行的脚本都只能运行在控制台上。如果你想要在应用程序、运维框架或是监控平台中集成它们，会非常得困难。</p><p>其次，这些命令行脚本很多都是通过连接ZooKeeper来提供服务的。目前，社区已经越来越不推荐任何工具直连ZooKeeper了，因为这会带来一些潜在的问题，比如这可能会绕过Kafka的安全设置。在专栏前面，我说过kafka-topics脚本连接ZooKeeper时，不会考虑Kafka设置的用户认证机制。也就是说，任何使用该脚本的用户，不论是否具有创建主题的权限，都能成功“跳过”权限检查，强行创建主题。这显然和Kafka运维人员配置权限的初衷背道而驰。</p><p>最后，运行这些脚本需要使用Kafka内部的类实现，也就是Kafka<strong>服务器端</strong>的代码。实际上，社区还是希望用户只使用Kafka<strong>客户端</strong>代码，通过现有的请求机制来运维管理集群。这样的话，所有运维操作都能纳入到统一的处理机制下，方便后面的功能演进。</p><!-- [[[read_end]]] --><p>基于这些原因，社区于0.11版本正式推出了Java客户端版的AdminClient，并不断地在后续的版本中对它进行完善。我粗略地计算了一下，有关AdminClient的优化和更新的各种提案，社区中有十几个之多，而且贯穿各个大的版本，足见社区对AdminClient的重视。</p><p>值得注意的是，<strong>服务器端也有一个AdminClient</strong>，包路径是kafka.admin。这是之前的老运维工具类，提供的功能也比较有限，社区已经不再推荐使用它了。所以，我们最好统一使用客户端的AdminClient。</p><h2>如何使用？</h2><p>下面，我们来看一下如何在应用程序中使用AdminClient。我们在前面说过，它是Java客户端提供的工具。想要使用它的话，你需要在你的工程中显式地增加依赖。我以最新的2.3版本为例来进行一下展示。</p><p>如果你使用的是Maven，需要增加以下依赖项：</p><pre><code>&lt;dependency&gt;
    &lt;groupId&gt;org.apache.kafka&lt;/groupId&gt;
    &lt;artifactId&gt;kafka-clients&lt;/artifactId&gt;
    &lt;version&gt;2.3.0&lt;/version&gt;
&lt;/dependency&gt;
</code></pre><p>如果你使用的是Gradle，那么添加方法如下：</p><pre><code>compile group: 'org.apache.kafka', name: 'kafka-clients', version: '2.3.0'
</code></pre><h2>功能</h2><p>鉴于社区还在不断地完善AdminClient的功能，所以你需要时刻关注不同版本的发布说明（Release Notes），看看是否有新的运维操作被加入进来。在最新的2.3版本中，AdminClient提供的功能有9大类。</p><ol>
<li>主题管理：包括主题的创建、删除和查询。</li>
<li>权限管理：包括具体权限的配置与删除。</li>
<li>配置参数管理：包括Kafka各种资源的参数设置、详情查询。所谓的Kafka资源，主要有Broker、主题、用户、Client-id等。</li>
<li>副本日志管理：包括副本底层日志路径的变更和详情查询。</li>
<li>分区管理：即创建额外的主题分区。</li>
<li>消息删除：即删除指定位移之前的分区消息。</li>
<li>Delegation Token管理：包括Delegation Token的创建、更新、过期和详情查询。</li>
<li>消费者组管理：包括消费者组的查询、位移查询和删除。</li>
<li>Preferred领导者选举：推选指定主题分区的Preferred Broker为领导者。</li>
</ol><h2>工作原理</h2><p>在详细介绍AdminClient的主要功能之前，我们先简单了解一下AdminClient的工作原理。<strong>从设计上来看，AdminClient是一个双线程的设计：前端主线程和后端I/O线程</strong>。前端线程负责将用户要执行的操作转换成对应的请求，然后再将请求发送到后端I/O线程的队列中；而后端I/O线程从队列中读取相应的请求，然后发送到对应的Broker节点上，之后把执行结果保存起来，以便等待前端线程的获取。</p><p>值得一提的是，AdminClient在内部大量使用生产者-消费者模式将请求生成与处理解耦。我在下面这张图中大致描述了它的工作原理。</p><p><img src="https://static001.geekbang.org/resource/image/bd/88/bd820c9c2a5fb554561a9f78d6543c88.jpg?wh=2968*1836" alt=""></p><p>如图所示，前端主线程会创建名为Call的请求对象实例。该实例有两个主要的任务。</p><ol>
<li><strong>构建对应的请求对象</strong>。比如，如果要创建主题，那么就创建CreateTopicsRequest；如果是查询消费者组位移，就创建OffsetFetchRequest。</li>
<li><strong>指定响应的回调逻辑</strong>。比如从Broker端接收到CreateTopicsResponse之后要执行的动作。一旦创建好Call实例，前端主线程会将其放入到新请求队列（New Call Queue）中，此时，前端主线程的任务就算完成了。它只需要等待结果返回即可。</li>
</ol><p>剩下的所有事情就都是后端I/O线程的工作了。就像图中所展示的那样，该线程使用了3个队列来承载不同时期的请求对象，它们分别是新请求队列、待发送请求队列和处理中请求队列。为什么要使用3个呢？原因是目前新请求队列的线程安全是由Java的monitor锁来保证的。<strong>为了确保前端主线程不会因为monitor锁被阻塞，后端I/O线程会定期地将新请求队列中的所有Call实例全部搬移到待发送请求队列中进行处理</strong>。图中的待发送请求队列和处理中请求队列只由后端I/O线程处理，因此无需任何锁机制来保证线程安全。</p><p>当I/O线程在处理某个请求时，它会显式地将该请求保存在处理中请求队列。一旦处理完成，I/O线程会自动地调用Call对象中的回调逻辑完成最后的处理。把这些都做完之后，I/O线程会通知前端主线程说结果已经准备完毕，这样前端主线程能够及时获取到执行操作的结果。AdminClient是使用Java Object对象的wait和notify实现的这种通知机制。</p><p>严格来说，AdminClient并没有使用Java已有的队列去实现上面的请求队列，它是使用ArrayList和HashMap这样的简单容器类，再配以monitor锁来保证线程安全的。不过，鉴于它们充当的角色就是请求队列这样的主体，我还是坚持使用队列来指代它们了。</p><p>了解AdminClient工作原理的一个好处在于，<strong>它能够帮助我们有针对性地对调用AdminClient的程序进行调试</strong>。</p><p>我们刚刚提到的后端I/O线程其实是有名字的，名字的前缀是kafka-admin-client-thread。有时候我们会发现，AdminClient程序貌似在正常工作，但执行的操作没有返回结果，或者hang住了，现在你应该知道这可能是因为I/O线程出现问题导致的。如果你碰到了类似的问题，不妨使用<strong>jstack命令</strong>去查看一下你的AdminClient程序，确认下I/O线程是否在正常工作。</p><p>这可不是我杜撰出来的好处，实际上，这是实实在在的社区bug。出现这个问题的根本原因，就是I/O线程未捕获某些异常导致意外“挂”掉。由于AdminClient是双线程的设计，前端主线程不受任何影响，依然可以正常接收用户发送的命令请求，但此时程序已经不能正常工作了。</p><h2>构造和销毁AdminClient实例</h2><p>如果你正确地引入了kafka-clients依赖，那么你应该可以在编写Java程序时看到AdminClient对象。<strong>切记它的完整类路径是org.apache.kafka.clients.admin.AdminClient，而不是kafka.admin.AdminClient</strong>。后者就是我们刚才说的服务器端的AdminClient，它已经不被推荐使用了。</p><p>创建AdminClient实例和创建KafkaProducer或KafkaConsumer实例的方法是类似的，你需要手动构造一个Properties对象或Map对象，然后传给对应的方法。社区专门为AdminClient提供了几十个专属参数，最常见而且必须要指定的参数，是我们熟知的<strong>bootstrap.servers参数</strong>。如果你想了解完整的参数列表，可以去<a href="https://kafka.apache.org/documentation/#adminclientconfigs">官网</a>查询一下。如果要销毁AdminClient实例，需要显式调用AdminClient的close方法。</p><p>你可以简单使用下面的代码同时实现AdminClient实例的创建与销毁。</p><pre><code>Properties props = new Properties();
props.put(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, &quot;kafka-host:port&quot;);
props.put(&quot;request.timeout.ms&quot;, 600000);

try (AdminClient client = AdminClient.create(props)) {
         // 执行你要做的操作……
}
</code></pre><p>这段代码使用Java 7的try-with-resource语法特性创建了AdminClient实例，并在使用之后自动关闭。你可以在try代码块中加入你想要执行的操作逻辑。</p><h2>常见的AdminClient应用实例</h2><p>讲完了AdminClient的工作原理和构造方法，接下来，我举几个实际的代码程序来说明一下如何应用它。这几个例子，都是我们最常见的。</p><h3>创建主题</h3><p>首先，我们来看看如何创建主题，代码如下：</p><pre><code>String newTopicName = &quot;test-topic&quot;;
try (AdminClient client = AdminClient.create(props)) {
         NewTopic newTopic = new NewTopic(newTopicName, 10, (short) 3);
         CreateTopicsResult result = client.createTopics(Arrays.asList(newTopic));
         result.all().get(10, TimeUnit.SECONDS);
}
</code></pre><p>这段代码调用AdminClient的createTopics方法创建对应的主题。构造主题的类是NewTopic类，它接收主题名称、分区数和副本数三个字段。</p><p>注意这段代码倒数第二行获取结果的方法。目前，AdminClient各个方法的返回类型都是名为***Result的对象。这类对象会将结果以Java Future的形式封装起来。如果要获取运行结果，你需要调用相应的方法来获取对应的Future对象，然后再调用相应的get方法来取得执行结果。</p><p>当然，对于创建主题而言，一旦主题被成功创建，任务也就完成了，它返回的结果也就不重要了，只要没有抛出异常就行。</p><h3>查询消费者组位移</h3><p>接下来，我来演示一下如何查询指定消费者组的位移信息，代码如下：</p><pre><code>String groupID = &quot;test-group&quot;;
try (AdminClient client = AdminClient.create(props)) {
         ListConsumerGroupOffsetsResult result = client.listConsumerGroupOffsets(groupID);
         Map&lt;TopicPartition, OffsetAndMetadata&gt; offsets = 
                  result.partitionsToOffsetAndMetadata().get(10, TimeUnit.SECONDS);
         System.out.println(offsets);
}
</code></pre><p>和创建主题的风格一样，<strong>我们调用AdminClient的listConsumerGroupOffsets方法去获取指定消费者组的位移数据</strong>。</p><p>不过，对于这次返回的结果，我们不能再丢弃不管了，<strong>因为它返回的Map对象中保存着按照分区分组的位移数据</strong>。你可以调用OffsetAndMetadata对象的offset()方法拿到实际的位移数据。</p><h3>获取Broker磁盘占用</h3><p>现在，我们来使用AdminClient实现一个稍微高级一点的功能：获取某台Broker上Kafka主题占用的磁盘空间量。有些遗憾的是，目前Kafka的JMX监控指标没有提供这样的功能，而磁盘占用这件事，是很多Kafka运维人员要实时监控并且极为重视的。</p><p>幸运的是，我们可以使用AdminClient来实现这一功能。代码如下：</p><pre><code>try (AdminClient client = AdminClient.create(props)) {
         DescribeLogDirsResult ret = client.describeLogDirs(Collections.singletonList(targetBrokerId)); // 指定Broker id
         long size = 0L;
         for (Map&lt;String, DescribeLogDirsResponse.LogDirInfo&gt; logDirInfoMap : ret.all().get().values()) {
                  size += logDirInfoMap.values().stream().map(logDirInfo -&gt; logDirInfo.replicaInfos).flatMap(
                           topicPartitionReplicaInfoMap -&gt;
                           topicPartitionReplicaInfoMap.values().stream().map(replicaInfo -&gt; replicaInfo.size))
                           .mapToLong(Long::longValue).sum();
         }
         System.out.println(size);
}
</code></pre><p>这段代码的主要思想是，使用AdminClient的<strong>describeLogDirs方法</strong>获取指定Broker上所有分区主题的日志路径信息，然后把它们累积在一起，得出总的磁盘占用量。</p><h2>小结</h2><p>好了，我们来小结一下。社区于0.11版本正式推出了Java客户端版的AdminClient工具，该工具提供了几十种运维操作，而且它还在不断地演进着。如果可以的话，你最好统一使用AdminClient来执行各种Kafka集群管理操作，摒弃掉连接ZooKeeper的那些工具。另外，我建议你时刻关注该工具的功能完善情况，毕竟，目前社区对AdminClient的变更频率很高。</p><p><img src="https://static001.geekbang.org/resource/image/06/f0/068a8cb7ea769799e60ad7f5a80a9bf0.jpg?wh=2069*2560" alt=""></p><h2>开放讨论</h2><p>请思考一下，如果我们要使用AdminClient去增加某个主题的分区，代码应该怎么写？请给出主体代码。</p><p>欢迎写下你的思考和答案，我们一起讨论。如果你觉得有所收获，也欢迎把文章分享给你的朋友。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/26/f9/2a7d80a3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>itzzy</span>
  </div>
  <div class="_2_QraFYR_0">可视化kafka管理工具，老师能推荐下吗？能支持2.0+版本 感谢！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: kafka manager</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-17 00:12:51</div>
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
  <div class="_2_QraFYR_0">1 引入原因：<br>	A ：kafka自带的各种命令行脚本都只能运行在控制台上，不便于集成进应用程序或运维框架<br>	B ：这些命令行脚本很多都是通过连接Zookeeper来提供服务，这存在一些潜在问题，如这可能绕开Kafka的安全设置。<br>	C ：这些脚本需要使用Kafka内部的类实现，即Kafka服务端的代码。社区希望用户只使用Kafka客户端代码，通过现有的请求机制来运维管理集群。<br><br>2 如何使用：<br>	A ：要想使用，需要在工程中显示的地增加依赖。<br><br>3 功能：<br>	A ：有九大类功能：<br>	（1）主题管理：包括主题的创建，查询和删除<br>	（2）权限管理：包括具体权限的配置与删除<br>	（3）配置参数管理：包括Kafka各种资源的参数设置，详情查询。所谓的kafka资源主要有Broker，主题，用户，Client-id等<br>	（4）副本日志管理：包括副本底层日志路径的变更和详情查询<br>	（5）分区管理：即创建额外的主题分区<br>	（6）消息删除：删除指定位移之前的分区消息<br>	（7）Delegation Token管理：包括Delegation Token的创建，更新，过期和详情查询<br>	（8）消费者组管理：包括消费者组的查询，位移查询和删除<br>	（9）Preferred领导者选举：推选指定主题分区的Preferred Broker为领导者。<br>4 工作原理<br>	A ：从设计上来看，AdminClient是一个双线程的设计：前端主线程和后端I&#47;O线程。<br>	（1）前端线程负责将用户要执行的操作转换成对应的请求，然后将请求发送到后端I&#47;O线程的队列中；<br>	（2）后端I&#47;O线程从队列中读取相应的请求，然后发送到对应的Broker节点上，之后把执行结果保存起来，以便等待前端线程的获取。<br>	B ：AdminClient在内部大量使用生产者—消费者模型将请求生产和处理解耦<br>	C ：前端主线程会创建一个名为Call的请求对象实例。该实例的有两个主要任务<br>	（1）构建对应的请求对象：如要创建主题，就创建CreateTopicRequest；要查询消费者位移，就创建OffsetFetchRequest<br>	（2）指定响应的回调逻辑：如Broker端接收到CreateTopicResponse之后要执行的动作。<br>	（*）一旦创建好Call实例，前端主线程会将其放入到新请求队列（New Call Queue）中，此时，前端主线程的任务就算完成了。他只需要等待结果返回即可。剩下的所有事情都是后端I&#47;O线程的工作了。<br>	<br>	D ：后端I&#47;O线程，该线程使用了3个队列来承载不同时期的请求对象，他们分别是新请求队列，待发送请求队列和处理中请求队列。<br>	（1）使用3个队列的原因：新请求队列的线程安全是有Java的monitor锁来保证的。为了确保前端主线程不会因为monitor锁被阻塞，后端I&#47;O线程会定期地将新请求队列中的所有Call实例全部搬移到待发送请求队列中进行处理。<br>	（2）待发送请求队列和处理中请求队列只由后端I&#47;O线程处理，因此无需任何锁机制来保证线程安全。<br>	（3）当I&#47;O线程在处理某个请求时，他会显式的将该请求保存在处理中请求队列。一旦处理完成，I&#47;O线程会自动地调用Call 对象中的回调完成最后的处理。<br>	（4）最后，I&#47;O线程会通知前端主线程处理完毕，这样前端主线程就能够及时的获取到执行操作的结果。<br><br>5 构造和销毁AdminClient实例<br>	A ：切记它的的完整路径是org.apche.kafka.clients.admin.AdminClient。<br>	B ：创建AdminClient实例和创建KafkaProducer或KafkaConsumer实例的方法是类似的，你需要手动构造一个Properties对象或Map对象，然后传给对应的方法。<br><br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-12 12:47:13</div>
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
  <div class="_2_QraFYR_0">想问下老师，在kafka某个topic下不小心创建了多个不用的消费组，怎么删除掉不用的消费组呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 弃之不用，Kafka会自动删除它们的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-15 20:39:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/1b/4a/f9df2d06.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>蒙开强</span>
  </div>
  <div class="_2_QraFYR_0">老师，你好，这个只是提供了API是吧，那要是想可视化工具，还得基于它写代码是么</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯嗯，是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-15 08:30:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/a0/cb/aab3b3e7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张三丰</span>
  </div>
  <div class="_2_QraFYR_0">这句话没懂，就算引入了其它两个队列，也无法避免锁阻塞啊，放进新请求队列的时候是一定会存在锁争用的。我完全可以开启一个后台IO线程直接消费新请求的队列，因为新请求队列一定是有序且线程安全的。<br><br>为了确保前端主线程不会因为 monitor 锁被阻塞，后端 I&#47;O 线程会定期地将新请求队列中的所有 Call 实例全部搬移到待发送请求队列中进行处理。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我的意思是，至少不用Kafka自己实现线程同步了，交由Java类自行处理就好</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-04 18:57:39</div>
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
  <div class="_2_QraFYR_0">是不是存在有 高版本的 AdminClient  不能兼容低版本的 broker的问题？<br><br>我记得调用 2.x 的 AdminClient API去触发低版本（0.8.x.x）broker的reassign，会报错提示 “这个操作不被支持”</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对的，不兼容。我记得只有在0.10.2.1之后才实现双向兼容</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-26 23:40:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/18/99/f47bcf7b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Unity</span>
  </div>
  <div class="_2_QraFYR_0">老师 请问 org.apache.kafka 的kafka-clients 和 kafka_{scala版本号}这两个jar包的区别是啥 ? </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 前者是Clients端代码，后者是Server端代码</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-26 16:21:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/be/b9/f2481c2c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>诗泽</span>
  </div>
  <div class="_2_QraFYR_0">老师可以简单对比一下pulsar 与kafka吗？感觉pulsar 的好多设计都是借鉴kafka的，最大的一个区别是将broker 与数据存储分离，使得broker 可以更加容易扩展。另外，consumer 数量的扩展也不受partition 数量的限制。pulsar 大有取代kafka之势，老师怎么看？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 哈哈，和Pulsar的郭总和翟总相识，不敢妄言。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-15 09:59:06</div>
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
  <div class="_2_QraFYR_0">添加JMX指标以获取 Broker 磁盘占用这块感觉可以提个KIP</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-15 09:33:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/fe/b4/295338e7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Allan</span>
  </div>
  <div class="_2_QraFYR_0">2.7 看了源码的 增加分区。但是没有看明白逻辑，对于kafka设计的原理不理解导致嘛？就是说为啥里面是这么分配。里面的集合的asList(1, 2),asList(2, 3), asList(3, 1) 表示的是什么意思？第一位是broker嘛？第二位是分区数量？看了半天没整明白？<br>Increase the partition count for a topic to the given totalCount assigning the new partitions according to the given newAssignments. The length of the given newAssignments should equal totalCount - oldCount, since the assignment of existing partitions are not changed. Each inner list of newAssignments should have a length equal to the topic&#39;s replication factor. The first broker id in each inner list is the &quot;preferred replica&quot;.<br>For example, suppose a topic currently has a replication factor of 2, and has 3 partitions. The number of partitions can be increased to 6 using a NewPartition constructed like this:<br>       NewPartitions.increaseTo(6, asList(asList(1, 2),<br>                                          asList(2, 3),<br>                                          asList(3, 1)))<br>       <br>In this example partition 3&#39;s preferred leader will be broker 1, partition 4&#39;s preferred leader will be broker 2 and partition 5&#39;s preferred leader will be broker 3.<br>Params:<br>totalCount – The total number of partitions after the operation succeeds.<br>newAssignments – The replica assignments for the new partitions.<br><br>public static NewPartitions increaseTo(int totalCount, List&lt;List&lt;Integer&gt;&gt; newAssignments) {<br>        return new NewPartitions(totalCount, newAssignments);<br>    }</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 没懂您的问题具体是什么。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-21 18:12:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/24/55/09/73f24874.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>建华</span>
  </div>
  <div class="_2_QraFYR_0">老师好，用adminclient中的creatacl实现授权好像没有效果，用kafka-acl.sh查不到记录，老师在java代码中怎么动态授权呀</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 有具体代码吗？这些信息量无法进一步判断问题：（</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-05 19:13:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/32/ba/16a12b9e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>王晓辉</span>
  </div>
  <div class="_2_QraFYR_0">增加分区数<br>Map&lt;String, NewPartitions&gt; newPartitionsMap = new HashMap&lt;&gt;();<br>        newPartitionsMap.put(&quot;test-topic&quot;, NewPartitions.increaseTo(13));   &#47;&#47; 增加到x分区，x要比原有分区数大<br>        CreatePartitionsResult result = adminClient.createPartitions(newPartitionsMap);<br>        result.all().get();</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: ������</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-17 13:44:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/51/0d/fc1652fe.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>James</span>
  </div>
  <div class="_2_QraFYR_0">之前代码统计分区数好像不太对(代码未测试),修改为以上代码<br>for (KafkaFuture&lt;TopicDescription&gt; kafkaFuture : kafkaFutures) {<br>                List&lt;TopicPartitionInfo&gt; topicPartitionInfos = kafkaFuture.get().partitions();<br>                count += topicPartitionInfos.size();<br>}</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-04 16:45:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/51/0d/fc1652fe.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>James</span>
  </div>
  <div class="_2_QraFYR_0"> 使用 AdminClient 去增加某个主题的分区,暂时还没有测试<br>   private void test() throws InterruptedException, ExecutionException, TimeoutException {<br>        Properties props = new Properties();props.put(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, &quot;kafka-host:port&quot;);props.put(&quot;request.timeout.ms&quot;, 600000);<br>        &#47;&#47;使用 AdminClient 去增加某个主题的分区<br>        String newTopicName = &quot;test-topic&quot;;<br>        try (AdminClient client = AdminClient.create(props)) {<br>            int count = 0;<br>            DescribeTopicsResult result = client.describeTopics(Arrays.asList(newTopicName));<br>            Map&lt;String, KafkaFuture&lt;TopicDescription&gt;&gt; kafkaFutureMap = result.values();<br>            Collection&lt;KafkaFuture&lt;TopicDescription&gt;&gt; kafkaFutures = kafkaFutureMap.values();<br>            for (KafkaFuture&lt;TopicDescription&gt; kafkaFuture : kafkaFutures) {<br>                List&lt;TopicPartitionInfo&gt; topicPartitionInfos = kafkaFuture.get().partitions();<br>                for (TopicPartitionInfo topicPartitionInfo : topicPartitionInfos) {<br>                    count += topicPartitionInfo.partition();<br>                }<br>            }<br>            &#47;&#47;新增一个分区<br>            ++count;<br>            Map&lt;String, NewPartitions&gt; newPartitionsMap = new HashMap&lt;&gt;();<br>            NewPartitions newPartition = NewPartitions.increaseTo(count);<br>            newPartitionsMap.put(newTopicName, newPartition);<br>            client.createPartitions(newPartitionsMap);<br>        }<br>    }</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-04 16:39:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1c/af/7c/6d90b40a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>毛怪</span>
  </div>
  <div class="_2_QraFYR_0">老师10版本的kafka怎么可以通过JMX获取指定的监控对象值吗，所有api可以调用吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 有个JmxTool工具，你可以运行bin&#47;kafka-run-class.sh kafka.tools.JmxTool去学习下它的用法</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-24 10:10:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/07/8a/4bef6202.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>大叮当</span>
  </div>
  <div class="_2_QraFYR_0">请问下，想写个java程序，该程序的功能是传入一个topic，能列出该topic下当前各个parition最小的offset各是多少，请问用哪个类啊，谢谢您。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 用KafkaConsumer就行，里面有endOffsets方法</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-06 12:25:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/52/bb/225e70a6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>hunterlodge</span>
  </div>
  <div class="_2_QraFYR_0">老师，请问怎么采集consumer group的性能指标呢？比如消息堆积数，需要了解到消费应用程序的JMX端口才能采集吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，典型的JMX指标包括lag, max-lag等</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-27 20:27:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/0b/e6/99183c8d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Subfire</span>
  </div>
  <div class="_2_QraFYR_0">👍</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-23 10:54:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/76/f8/66e25be4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>maslke</span>
  </div>
  <div class="_2_QraFYR_0">手里的开发环境是这样：一台widnows 10的机器，在linux子系统中安装了kafka，在windows中进行AdminClient调用，刚开始连接不上kakfa。后来通过在windows下，调用bat脚本才发现是PCNAME.localdomain这个hostname识别不了。后来通过在hosts进行了一下配置才ok。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯嗯，最好别在Windows上测试Kafka：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-07 11:47:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/76/f8/66e25be4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>maslke</span>
  </div>
  <div class="_2_QraFYR_0">        Map&lt;String, NewPartitions&gt; newPartitionsMap = new HashMap&lt;&gt;();<br>        newPartitionsMap.put(topicName, NewPartitions.increaseTo(partitions));<br>        CreatePartitionsResult createPartitionsResult = client.createPartitions(newPartitionsMap);<br>        KafkaFuture&lt;Void&gt; future1 = createPartitionsResult.values().get(topicName);<br>        future1.get();<br>        System.out.println(&quot;ok&quot;);</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-07 11:45:25</div>
  </div>
</div>
</div>
</li>
</ul>