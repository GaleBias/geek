<audio title="29 _ Kafka动态配置了解下？" src="https://static001.geekbang.org/resource/audio/73/51/73ef41517a156952de4049d0ff088d51.mp3" controls="controls"></audio> 
<p>你好，我是胡夕。今天我要和你讨论的主题是：Kafka的动态Broker参数配置。</p><h2>什么是动态Broker参数配置？</h2><p>在开始今天的分享之前，我们先来复习一下设置Kafka参数，特别是Broker端参数的方法。</p><p>在Kafka安装目录的config路径下，有个server.properties文件。通常情况下，我们会指定这个文件的路径来启动Broker。如果要设置Broker端的任何参数，我们必须在这个文件中显式地增加一行对应的配置，之后启动Broker进程，令参数生效。我们常见的做法是，一次性设置好所有参数之后，再启动Broker。当后面需要变更任何参数时，我们必须重启Broker。但生产环境中的服务器，怎么能随意重启呢？所以，目前修改Broker端参数是非常痛苦的过程。</p><p>基于这个痛点，社区于1.1.0版本中正式引入了动态Broker参数（Dynamic Broker Configs）。所谓动态，就是指修改参数值后，无需重启Broker就能立即生效，而之前在server.properties中配置的参数则称为静态参数（Static Configs）。显然，动态调整参数值而无需重启服务，是非常实用的功能。如果你想体验动态Broker参数的话，那就赶快升级到1.1版本吧。</p><!-- [[[read_end]]] --><p>当然了，当前最新的2.3版本中的Broker端参数有200多个，社区并没有将每个参数都升级成动态参数，它仅仅是把一部分参数变成了可动态调整。那么，我们应该如何分辨哪些参数是动态参数呢？</p><p>如果你打开1.1版本之后（含1.1）的Kafka官网，你会发现<a href="https://kafka.apache.org/documentation/#brokerconfigs">Broker Configs</a>表中增加了Dynamic Update Mode列。该列有3类值，分别是read-only、per-broker和cluster-wide。我来解释一下它们的含义。</p><ul>
<li>read-only。被标记为read-only的参数和原来的参数行为一样，只有重启Broker，才能令修改生效。</li>
<li>per-broker。被标记为per-broker的参数属于动态参数，修改它之后，只会在对应的Broker上生效。</li>
<li>cluster-wide。被标记为cluster-wide的参数也属于动态参数，修改它之后，会在整个集群范围内生效，也就是说，对所有Broker都生效。你也可以为具体的Broker修改cluster-wide参数。</li>
</ul><p>我来举个例子说明一下per-broker和cluster-wide的区别。Broker端参数listeners想必你应该不陌生吧。它是一个per-broker参数，这表示你只能为单个Broker动态调整listeners，而不能直接调整一批Broker的listeners。log.retention.ms参数是cluster-wide级别的，Kafka允许为集群内所有Broker统一设置一个日志留存时间值。当然了，你也可以为单个Broker修改此值。</p><h2>使用场景</h2><p>你可能会问，动态Broker参数的使用场景都有哪些呢？实际上，因为不必重启Broker，动态Broker参数的使用场景非常广泛，通常包括但不限于以下几种：</p><ul>
<li>动态调整Broker端各种线程池大小，实时应对突发流量。</li>
<li>动态调整Broker端连接信息或安全配置信息。</li>
<li>动态更新SSL Keystore有效期。</li>
<li>动态调整Broker端Compact操作性能。</li>
<li>实时变更JMX指标收集器(JMX Metrics Reporter)。</li>
</ul><p>在这些使用场景中，动态调整线程池大小应该算是最实用的功能了。很多时候，当Kafka Broker入站流量（inbound data）激增时，会造成Broker端请求积压（Backlog）。有了动态参数，我们就能够动态增加网络线程数和I/O线程数，快速消耗一些积压。当突发流量过去后，我们也能将线程数调整回来，减少对资源的浪费。整个过程都不需要重启Broker。你甚至可以将这套调整线程数的动作，封装进定时任务中，以实现自动扩缩容。</p><h2>如何保存？</h2><p>由于动态配置的特殊性，它必然有和普通只读参数不同的保存机制。下面我来介绍一下Kafka是如何保存动态配置的。</p><p>首先，Kafka将动态Broker参数保存在ZooKeeper中，具体的znode路径如下图所示。</p><p><img src="https://static001.geekbang.org/resource/image/67/59/67125a22a27028e18bab9f03a6664859.png" alt=""></p><p>我来解释一下图中的内容。changes是用来实时监测动态参数变更的，不会保存参数值；topics是用来保存Kafka主题级别参数的。虽然它们不属于动态Broker端参数，但其实它们也是能够动态变更的。</p><p>users和clients则是用于动态调整客户端配额（Quota）的znode节点。所谓配额，是指Kafka运维人员限制连入集群的客户端的吞吐量或者是限定它们使用的CPU资源。</p><p>分析到这里，我们就会发现，/config/brokers znode才是真正保存动态Broker参数的地方。该znode下有两大类子节点。第一类子节点就只有一个，它有个固定的名字叫&lt; default &gt;，保存的是前面说过的cluster-wide范围的动态参数；另一类则以broker.id为名，保存的是特定Broker的per-broker范围参数。由于是per-broker范围，因此这类子节点可能存在多个。</p><p>我们一起来看一张图片，它展示的是我的一个Kafka集群环境上的动态Broker端参数。</p><p><img src="https://static001.geekbang.org/resource/image/c4/7c/c4154989775023afb619f2fdb07f6f7c.png" alt=""></p><p>在这张图中，我首先查看了/config/brokers下的子节点，我们可以看到，这里面有&lt; default &gt;节点和名为0、1的子节点。&lt; default &gt;节点中保存了我设置的cluster-wide范围参数；0和1节点中分别保存了我为Broker 0和Broker  1设置的per-broker参数。</p><p>接下来，我分别展示了cluster-wide范围和per-broker范围的参数设置。拿num.io.threads参数为例，其cluster-wide值被动态调整为12，而在Broker 0上被设置成16，在Broker 1上被设置成8。我为Broker 0和Broker 1单独设置的值，会覆盖掉cluster-wide值，但在其他Broker上，该参数默认值还是按12计算。</p><p>如果我们再把静态参数加进来一起讨论的话，cluster-wide、per-broker和static参数的优先级是这样的：per-broker参数 &gt; cluster-wide参数 &gt; static参数 &gt; Kafka默认值。</p><p>另外，如果你仔细查看上图中的<strong>ephemeralOwner字段</strong>，你会发现它们的值都是0x0。这表示这些znode都是持久化节点，它们将一直存在。即使ZooKeeper集群重启，这些数据也不会丢失，这样就能保证这些动态参数的值会一直生效。</p><h2>如何配置？</h2><p>讲完了保存原理，我们来说说如何配置动态Broker参数。目前，设置动态参数的工具行命令只有一个，那就是Kafka自带的kafka-configs脚本。接下来，我来以unclean.leader.election.enable参数为例，演示一下如何动态调整。</p><p>下面这条命令展示了如何在集群层面设置全局值，即设置cluster-wide范围值。</p><pre><code>$ bin/kafka-configs.sh --bootstrap-server kafka-host:port --entity-type brokers --entity-default --alter --add-config unclean.leader.election.enable=true
Completed updating default config for brokers in the cluster,
</code></pre><p>总体来说命令很简单，但有一点需要注意。<strong>如果要设置cluster-wide范围的动态参数，需要显式指定entity-default</strong>。现在，我们使用下面的命令来查看一下刚才的配置是否成功。</p><pre><code>$ bin/kafka-configs.sh --bootstrap-server kafka-host:port --entity-type brokers --entity-default --describe
Default config for brokers in the cluster are:
  unclean.leader.election.enable=true sensitive=false synonyms={DYNAMIC_DEFAULT_BROKER_CONFIG:unclean.leader.election.enable=true}
</code></pre><p>从输出来看，我们成功地在全局层面上设置该参数值为true。注意sensitive=false的字眼，它表明我们要调整的参数不是敏感数据。如果我们调整的是类似于密码这样的参数时，该字段就会为true，表示这属于敏感数据。</p><p>好了，调整完cluster-wide范围的参数，我来演示下如何设置per-broker范围参数。我们还是以unclean.leader.election.enable参数为例，我现在为ID为1的Broker设置一个不同的值。命令如下：</p><pre><code>$ bin/kafka-configs.sh --bootstrap-server kafka-host:port --entity-type brokers --entity-name 1 --alter --add-config unclean.leader.election.enable=false
Completed updating config for broker: 1.
</code></pre><p>同样，我们使用下列命令，来查看一下刚刚的设置是否生效了。</p><pre><code>$ bin/kafka-configs.sh --bootstrap-server kafka-host:port --entity-type brokers --entity-name 1 --describe
Configs for broker 1 are:
  unclean.leader.election.enable=false sensitive=false synonyms={DYNAMIC_BROKER_CONFIG:unclean.leader.election.enable=false, DYNAMIC_DEFAULT_BROKER_CONFIG:unclean.leader.election.enable=true, DEFAULT_CONFIG:unclean.leader.election.enable=false}
</code></pre><p>这条命令的输出信息很多。我们关注两点即可。</p><ol>
<li>在Broker 1层面上，该参数被设置成了false，这表明命令运行成功了。</li>
<li>从倒数第二行可以看出，在全局层面上，该参数值依然是true。这表明，我们之前设置的cluster-wide范围参数值依然有效。</li>
</ol><p>如果我们要删除cluster-wide范围参数或per-broker范围参数，也非常简单，分别执行下面的命令就可以了。</p><pre><code># 删除cluster-wide范围参数
$ bin/kafka-configs.sh --bootstrap-server kafka-host:port --entity-type brokers --entity-default --alter --delete-config unclean.leader.election.enable
Completed updating default config for brokers in the cluster,
</code></pre><pre><code># 删除per-broker范围参数
$ bin/kafka-configs.sh --bootstrap-server kafka-host:port --entity-type brokers --entity-name 1 --alter --delete-config unclean.leader.election.enable
Completed updating config for broker: 1.
</code></pre><p><strong>删除动态参数要指定delete-config</strong>。当我们删除完动态参数配置后，再次运行查看命令，结果如下：</p><pre><code># 查看cluster-wide范围参数
$ bin/kafka-configs.sh --bootstrap-server kafka-host:port  --entity-type brokers --entity-default --describe
Default config for brokers in the cluster are:
</code></pre><pre><code># 查看Broker 1上的动态参数配置
$ bin/kafka-configs.sh --bootstrap-server kafka-host:port  --entity-type brokers --entity-name 1 --describe
Configs for broker 1 are:
</code></pre><p>此时，刚才配置的所有动态参数都已经被成功移除了。</p><p>刚刚我只是举了一个参数的例子，如果你想要知道动态Broker参数都有哪些，一种方式是在Kafka官网中查看Broker端参数列表，另一种方式是直接运行无参数的kafka-configs脚本，该脚本的说明文档会告诉你当前动态Broker参数都有哪些。我们可以先来看看下面这两张图。</p><p><img src="https://static001.geekbang.org/resource/image/40/f1/400532989c3214f00a2f764fe7983cf1.png" alt=""></p><p><img src="https://static001.geekbang.org/resource/image/eb/67/eb99853bca3d91ad3a53db5dbe3e3b67.png" alt=""></p><p>看到有这么多动态Broker参数，你可能会问：这些我都需要调整吗？你能告诉我最常用的几个吗？根据我的实际使用经验，我来跟你分享一些有较大几率被动态调整值的参数。</p><p><strong>1.log.retention.ms。</strong></p><p>修改日志留存时间应该算是一个比较高频的操作，毕竟，我们不可能完美地预估所有业务的消息留存时长。虽然该参数有对应的主题级别参数可以设置，但拥有在全局层面上动态变更的能力，依然是一个很好的功能亮点。</p><p><strong>2.num.io.threads和num.network.threads。</strong></p><p>这是我们在前面提到的两组线程池。就我个人而言，我觉得这是动态Broker参数最实用的场景了。毕竟，在实际生产环境中，Broker端请求处理能力经常要按需扩容。如果没有动态Broker参数，我们是无法做到这一点的。</p><p><strong>3.与SSL相关的参数。</strong></p><p>主要是4个参数（ssl.keystore.type、ssl.keystore.location、ssl.keystore.password和ssl.key.password）。允许动态实时调整它们之后，我们就能创建那些过期时间很短的SSL证书。每当我们调整时，Kafka底层会重新配置Socket连接通道并更新Keystore。新的连接会使用新的Keystore，阶段性地调整这组参数，有利于增加安全性。</p><p><strong>4.num.replica.fetchers。</strong></p><p>这也是我认为的最实用的动态Broker参数之一。Follower副本拉取速度慢，在线上Kafka环境中一直是一个老大难的问题。针对这个问题，常见的做法是增加该参数值，确保有充足的线程可以执行Follower副本向Leader副本的拉取。现在有了动态参数，你不需要再重启Broker，就能立即在Follower端生效，因此我说这是很实用的应用场景。</p><h2>小结</h2><p>好了，我们来小结一下。今天，我们重点讨论了Kafka 1.1.0版本引入的动态Broker参数。这类参数最大的好处在于，无需重启Broker，就可以令变更生效，因此能够极大地降低运维成本。除此之外，我还给出了动态参数的保存机制和设置方法。在专栏的后面，我还会给出动态参数设置的另一种方法，敬请期待。</p><p><img src="https://static001.geekbang.org/resource/image/22/74/22ef35547f9bddb8d56bdb350fd52f74.jpg" alt=""></p><h2>开放讨论</h2><p>目前，社区只是将一部分Broker参数升级为动态参数，在实际使用过程中，你觉得还有哪些参数也应该变为可动态修改？</p><p>欢迎写下你的思考和答案，我们一起讨论。如果你觉得有所收获，也欢迎把文章分享给你的朋友。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/fd/fd/326be9bb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>注定非凡</span>
  </div>
  <div class="_2_QraFYR_0">1 诞生背景：<br>	普通做法是，一次性在server.properties文件中配置好所有参数后，启动Broker。但是在需要变更任何参数后，就必须要重启Broker，这很不方便<br>	在1.1.0版本中正是引入了动态Broker参数（Dynamic Broker Configs）。<br>2 概念：<br>	所谓动态，就是指修改参数值后，无需重启Broker就能立即生效，在server.properties中配置的参数称之为静态参数（Static Configs）。<br><br>3 分类：<br>	read-only:被标记为read-only的参数和原来的参数行为一样，只有重启Broker，才能令修改生效。<br>	per-broker:被标记为per-broker的参数属于动态参数，修改它之后，只会在对应的Broker上生效。<br>	cluster-wide:被标记为cluster-wide的参数也属于动态参数，修改它之后，会在整个集群范围内生效。<br><br>4 使用场景：<br>	A ：动态调整Broker端各种线程池大小，实时应对突发流量<br>	B ：动态调整Broker端连接信息或安全配置信息<br>	C ：动态更新SSL Keystore有效期<br>	D ：动态调整Broker端Compact操作性能<br>	E ：实时变更JMX指示收集器（JMX Metrics Reporter）<br>5 保存：<br> <br>	A ：首先Kafka将动态Broker参数保存在Zookeeper中<br>		（1）changes是用来实时监控动态参数变更的，不会保存参数值；<br>		（2）topic是用来保存Kafka主题级别参数的。<br>		（3）user和clients是用于动态调整客户端配额（Quota）的znode节点。所谓配额是指Kafka运维人员限制连入集群的客户端的吞吐量或是限定他们的使用CPU资源。<br><br>	B ：&#47;config&#47;brokers znode才是真正保存动态Broker参数的地方。该znode下有两大类子节点。<br>		（1）第一类子节点只有一个，固定叫&lt;default&gt;，保存cluster-wide范围的动态参数<br>		（2）第二类以Broker.id为名，保存特定Broker的per-broker范围参数。<br><br>	C ：参数的优先级别：per-broker参数 &gt; cluster-wide参数 &gt; static参数 &gt; Kafka默认值。<br><br>6 如何配置：<br>	A ：使用Kafka自带的Kafka自带的Kafka-configs脚本。<br>	如果要设置cluster-wide范围的动态参数，需要显式指定entity-default。<br>	<br>	B ：较大几率被动态调整的参值<br>	（1）：log.retention.ms：修改日志留存时间<br>	（2）：num.io.threads 和 num.network.threads <br>	（3）：与SS相关的参数<br>		ssl.keystore.type，ssl.keystore.location，ssl.kestore.password和ssl.key.password。允许动态实时调整他们，就能创建过期时间很短的SSL证书，使用新的Keystore，阶段性的调整这组参数，提升安全性。<br>	（4）：num.replica.fetchers：这个可以增加该参数值，确保有充足的线程可以执行Follower副本向Leader副本的拉取。<br>	<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-11 13:52:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/67/f4/9a1feb59.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>钱</span>
  </div>
  <div class="_2_QraFYR_0">请问老师，kafka动态调整配置的实现原理是啥？它怎么做到不需重启broker就能使配置生效的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 把配置保存在ZooKeeper中，然后动态监听这个变更，一旦监听到立即处理</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-18 22:54:19</div>
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
  <div class="_2_QraFYR_0">follower为什么会拉取副本慢？它不负责读写只专心同步副本数据，增加线程就有用吗？会增加leader负担吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 因为是异步的网络请求。一个Broker上可能有很多follower副本，增加线程可以均匀分散这些follower副本来去任务</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-09 09:07:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/ab/9b/b12c223d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>第一片心意</span>
  </div>
  <div class="_2_QraFYR_0">动态更改之后的参数，如果某个broker重启的话，那些动态修改过的参数会失效吧，毕竟broker启动时，会读取 config&#47;server.propertyes 文件的内容。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不会失效。动态参数配置在ZooKeeper中</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-16 16:47:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/cd/2a/bdbed6ed.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>无菇朋友</span>
  </div>
  <div class="_2_QraFYR_0">老师 请问下 怎么查看每个broker的请求积压情况？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以看看broker端的请求队列的JMX指标</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-20 20:09:36</div>
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
  <div class="_2_QraFYR_0">怎样知道什么时候该调整这两组线程池大小？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以监控请求队列的大小</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-09 09:05:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/49/3d/4ac37cc2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>外星人</span>
  </div>
  <div class="_2_QraFYR_0">您好，我们生产上num.io.threads已经提高到了24，但是对应idle还是在0.3左右，是不是kafkaapi哪个接口处理慢了？这种情况一般如何定位啊？谢谢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 大概率是PRODUCE和FETCH请求。不过你最好结合kafka.network:type=RequestMetrics,name=RequestQueueTimeMs,request={Produce|FetchConsumer|FetchFollower}确认下是否需要调整num.network.threads</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-22 08:42:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/49/3d/4ac37cc2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>外星人</span>
  </div>
  <div class="_2_QraFYR_0">你好，我们用的版本是1.1，请问下，动态参数有没有bug啊？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 目前没有太严重的bug<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-08 08:43:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/86/fa/4bcd7365.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>玉剑冰锋</span>
  </div>
  <div class="_2_QraFYR_0">这么看的话，是不是不管是动态参数还是静态参数，集群中的broker配置是可以不一样的对吧？我用0.11（测试环境），环境中三台kafka磁盘容量不一样，导致的问题就是另外两台磁盘已经到了警戒值，剩下那一台磁盘始终用不上，如果可以的话我直接调整一台的保留时间就好了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 每个Broker可以是不同的配置。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-08 08:01:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJkKNKezgVwHxn13elAia7xueLnHAguTrbXcnMmDWWpUsnVWj4iapXWSs4tJGZvL6VotbudUaGSLo6w/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>summer</span>
  </div>
  <div class="_2_QraFYR_0">老师好，最近在做一个kafka限流功能，网上大部分都是通过命令或者直接修改zk节点的方式来管理配额，看jar包里有cilentQuotaCallback这个类，请问这个类的具体使用的方式是什么了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-02 00:08:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/55/a9/5282a560.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>yic</span>
  </div>
  <div class="_2_QraFYR_0">胡老师，有个问题想请教一下，我们项目想通过广播消息给下游服务。用到了kafka，消费消息的服务每个实例都设置自己的消费组groupId，如：server_prefix_IP_PORT。这样的话，每次服务部署，消费组的groupId都会发生变化。那之前的消费者组信息却还保留在系统中。这样就导致kafka监控时，还能看到老的消费者组相关指标。请问有什么好的方式处理这个问题吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 消费者组的名字不太适合总是动态修改，因为每次都相当于创建了一个新的消费者组。当然，如果一定要这么用，你可以手动把消费者组的信息删除掉（高版本的Kafka提供了这个功能）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-18 17:31:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/b3/53/25a4ae4b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>DK</span>
  </div>
  <div class="_2_QraFYR_0">胡老师好，我们通过canal将db的数据发送到kafka（kafka_2.11-1.1.1），设置的log.retention.hours=72，保留3天，但是偶尔出现某些topic的partion会删除当天的数据文件导致数据丢失。查看当时的kafka日志<br>[2021-03-15 11:06:51,824] INFO [Log partition=um.am_gongzhonghao_scene_user-0, dir=&#47;apps&#47;srv&#47;kafka&#47;data] Deleting segments LogSegment(baseOffset=73347921, size=470345801, lastModifiedTime=1615777551000, largestTime=1615518241494) (kafka.log.Log)<br>发现lastModifiedTime是更新了的，但largestTime没有更新，1615518241494对应的时间为“2021-03-12 11:04:01”，刚好是3天前。不知道这种情况胡老师是否遇到过。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 查查是不是canal客户端的问题，比如发送的kafka消息没有时间戳？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-11 20:04:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/e8/c2/77a413a7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Action</span>
  </div>
  <div class="_2_QraFYR_0">老师我进入到zookeepr安装目录，并没有找到您提到的路径，是我安装的版本不对吗？我的安装版本是 zookeeper-3.4.8]<br>以下是安装目录：<br>bin          conf     dist-maven       ivy.xml      NOTICE.txt            recipes              zookeeper-3.4.8.jar.asc<br>build.xml    contrib  docs             lib          README_packaging.txt  src                  zookeeper-3.4.8.jar.md5<br>CHANGES.txt  data     ivysettings.xml  LICENSE.txt  README.txt </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 您下载的是二进制安装包还是源码包？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-03 23:33:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/99/f0/d9343049.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>星亦辰</span>
  </div>
  <div class="_2_QraFYR_0">请教下老师， confluent_kafka 的类库怎么样获取 consumer 指定group 的offset 值？ 或者有没有其他的办法监控到一个数据队列对于一个group 而言，堆积了多少未处理的数据？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以使用AdminClient.listConsumerGroupOffsets方法获取之</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-19 16:26:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/3a/70/a874d69c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Mick</span>
  </div>
  <div class="_2_QraFYR_0">老师，请问下在文件系统中是不是看不到 znode路径结构的?我找了半天没找到图上的目录结构。我要怎么才能看到？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你要访问ZooKeeper才能看到ZooKeep下的路径结构。如果你使用的是Kafka自带的zookeeper，可以运行bin&#47;zookeeper-shell.sh zkhost:zkport</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-20 09:28:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/3a/70/a874d69c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Mick</span>
  </div>
  <div class="_2_QraFYR_0">INFO Got user-level KeeperException when processing sessionid:0x100000400f10000 type:create cxid:0x1 zxid:0x726 txntype:-1 reqpath:n&#47;a Error Path:&#47;consumers Error:KeeperErrorCode = NodeExists for &#47;consumers (org.apache.zookeeper.server.PrepRequestProcessor)<br>INFO Got user-level KeeperException when processing sessionid:0x100000400f10000 type:create cxid:0x2 zxid:0x727 txntype:-1 reqpath:n&#47;a Error Path:&#47;brokers&#47;ids Error:KeeperErrorCode = NodeExists for &#47;brokers&#47;ids (org.apache.zookeeper.server.PrepRequestProcessor)<br><br>INFO Got user-level KeeperException when processing sessionid:0x100000400f10000 type:create cxid:0x3 zxid:0x728 txntype:-1 reqpath:n&#47;a Error Path:&#47;brokers&#47;topics Error:KeeperErrorCode = NodeExists for &#47;brokers&#47;topics (org.apache.zookeeper.server.PrepRequestProcessor)<br><br>INFO Got user-level KeeperException when processing sessionid:0x100000400f10000 type:create cxid:0x4 zxid:0x729 txntype:-1 reqpath:n&#47;a Error Path:&#47;config&#47;changes Error:KeeperErrorCode = NodeExists for &#47;config&#47;changes (org.apache.zookeeper.server.PrepRequestProcessor)<br><br>INFO Got user-level KeeperException when processing sessionid:0x100000400f10000 type:create cxid:0xb zxid:0x730 txntype:-1 reqpath:n&#47;a Error Path:&#47;config&#47;clients Error:KeeperErrorCode = NodeExists for &#47;config&#47;clients (org.apache.zookeeper.server.PrepRequestProcessor)<br><br>INFO Got user-level KeeperException when processing sessionid:0x100000400f10000 type:create cxid:0xc zxid:0x731 txntype:-1 reqpath:n&#47;a Error Path:&#47;config&#47;users Error:KeeperErrorCode = NodeExists for &#47;config&#47;users (org.apache.zookeeper.server.PrepRequestProcessor)<br><br>INFO Got user-level KeeperException when processing sessionid:0x100000400f10000 type:create cxid:0xd zxid:0x732 txntype:-1 reqpath:n&#47;a Error Path:&#47;config&#47;brokers Error:KeeperErrorCode = NodeExists for &#47;config&#47;brokers (org.apache.zookeeper.server.PrepRequestProcessor)<br>老师，请问下启动zk 然后启动kafka后 zk报错 这是什么导致的?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这些是INFO级别的日志，Zk上某些节点不存在，不一定就是问题，有其他的ERROR日志吗？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-19 23:50:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/0f/67/1cfb574e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Sunney</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，我想咨询一下NATS和Kafka的区别和各自的适用场景是什么？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-09 11:19:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/9b/27/f5e29b94.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>边城</span>
  </div>
  <div class="_2_QraFYR_0">您好，动态参数会一直生效吗？是不是需要在 server.properties 里同步维护，防止下次重启时参数失效。<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不会失效，即使重启</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-08 09:23:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/86/fa/4bcd7365.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>玉剑冰锋</span>
  </div>
  <div class="_2_QraFYR_0">如果可以每台设置不同的日志保存时间，这样的话会导致分区留存时间不一致，消费者消费的问题怎么办？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 统一的机制。反正消费不到数据的话就发生位移越界的情况</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-08 08:10:57</div>
  </div>
</div>
</div>
</li>
</ul>