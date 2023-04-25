<audio title="37 _ 主流的Kafka监控框架" src="https://static001.geekbang.org/resource/audio/12/32/1242204af096f82f9c4f14731d199732.mp3" controls="controls"></audio> 
<p>你好，我是胡夕。今天我要和你分享的主题是：那些主流的Kafka监控框架。</p><p>在上一讲中，我们重点讨论了如何监控Kafka集群，主要是侧重于讨论监控原理和监控方法。今天，我们来聊聊具体的监控工具或监控框架。</p><p>令人有些遗憾的是，Kafka社区似乎一直没有在监控框架方面投入太多的精力。目前，Kafka的新功能提议已超过500个，但没有一个提议是有关监控框架的。当然，Kafka的确提供了超多的JMX指标，只是，单独查看这些JMX指标往往不是很方便，我们还是要依赖于框架统一地提供性能监控。</p><p>也许，正是由于社区的这种“不作为”，很多公司和个人都自行着手开发Kafka监控框架，其中并不乏佼佼者。今天我们就来全面地梳理一下主流的监控框架。</p><h2>JMXTool工具</h2><p>首先，我向你推荐JMXTool工具。严格来说，它并不是一个框架，只是社区自带的一个工具罢了。JMXTool工具能够实时查看Kafka JMX指标。倘若你一时找不到合适的框架来做监控，JMXTool可以帮你“临时救急”一下。</p><p>Kafka官网没有JMXTool的任何介绍，你需要运行下面的命令，来获取它的使用方法的完整介绍。</p><pre><code>bin/kafka-run-class.sh kafka.tools.JmxTool
</code></pre><p>JMXTool工具提供了很多参数，但你不必完全了解所有的参数。我把主要的参数说明列在了下面的表格里，你至少要了解一下这些参数的含义。</p><!-- [[[read_end]]] --><p><img src="https://static001.geekbang.org/resource/image/79/d3/795399a24e665c1bf744085b5344f5d3.jpg?wh=1763*1103" alt=""></p><p>现在，我举一个实际的例子来说明一下如何运行这个命令。</p><p>假设你要查询Broker端每秒入站的流量，即所谓的JMX指标BytesInPerSec，这个JMX指标能帮助你查看Broker端的入站流量负载，如果你发现这个值已经接近了你的网络带宽，这就说明该Broker的入站负载过大。你需要降低该Broker的负载，或者将一部分负载转移到其他Broker上。</p><p>下面这条命令，表示每5秒查询一次过去1分钟的BytesInPerSec均值。</p><pre><code>bin/kafka-run-class.sh kafka.tools.JmxTool --object-name kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec --jmx-url service:jmx:rmi:///jndi/rmi://:9997/jmxrmi --date-format &quot;YYYY-MM-dd HH:mm:ss&quot; --attributes OneMinuteRate --reporting-interval 1000
</code></pre><p>在这条命令中，有几点需要你注意一下。</p><ul>
<li>设置 --jmx-url参数的值时，需要指定JMX端口。在这个例子中，端口是9997，在实际操作中，你需要指定你的环境中的端口。</li>
<li>由于我是直接在Broker端运行的命令，因此就把主机名忽略掉了。如果你是在其他机器上运行这条命令，你要记得带上要连接的主机名。</li>
<li>关于 --object-name参数值的完整写法，我们可以直接在Kafka官网上查询。我们在前面说过，Kafka提供了超多的JMX指标，你需要去官网学习一下它们的用法。我以ActiveController JMX指标为例，介绍一下学习的方法。你可以在官网上搜索关键词ActiveController，找到它对应的 --object-name，即kafka.controller:type=KafkaController,name=ActiveControllerCount，这样，你就可以执行下面的脚本，来查看当前激活的Controller数量。</li>
</ul><pre><code>$ bin/kafka-run-class.sh kafka.tools.JmxTool --object-name kafka.controller:type=KafkaController,name=ActiveControllerCount --jmx-url service:jmx:rmi:///jndi/rmi://:9997/jmxrmi --date-format &quot;YYYY-MM-dd HH:mm:ss&quot; --reporting-interval 1000
Trying to connect to JMX url: service:jmx:rmi:///jndi/rmi://:9997/jmxrmi.
&quot;time&quot;,&quot;kafka.controller:type=KafkaController,name=ActiveControllerCount:Value&quot;
2019-08-05 15:08:30,1
2019-08-05 15:08:31,1
</code></pre><p>总体来说，JMXTool是社区自带的一个小工具，对于一般简单的监控场景，它还能应付，但是它毕竟功能有限，复杂的监控整体解决方案，还是要依靠监控框架。</p><h2>Kafka Manager</h2><p>说起Kafka监控框架，最有名气的当属Kafka Manager了。Kafka Manager是雅虎公司于2015年开源的一个Kafka监控框架。这个框架用Scala语言开发而成，主要用于管理和监控Kafka集群。</p><p>应该说Kafka Manager是目前众多Kafka监控工具中最好的一个，无论是界面展示内容的丰富程度，还是监控功能的齐全性，它都是首屈一指的。不过，目前该框架已经有4个月没有更新了，而且它的活跃的代码维护者只有三四个人，因此，很多Bug或问题都不能及时得到修复，更重要的是，它无法追上Apache Kafka版本的更迭速度。</p><p>当前，Kafka Manager最新版是2.0.0.2。在其Github官网上下载tar.gz包之后，我们执行解压缩，可以得到kafka-manager-2.0.0.2目录。</p><p>之后，我们需要运行sbt工具来编译Kafka Manager。sbt是专门用于构建Scala项目的编译构建工具，类似于我们熟知的Maven和Gradle。Kafka Manager自带了sbt命令，我们直接运行它构建项目就可以了：</p><pre><code>./sbt clean dist
</code></pre><p>经过漫长的等待之后，你应该可以看到项目已经被成功构建了。你可以在Kafka Manager的target/universal目录下找到生成的zip文件，把它解压，然后修改里面的conf/application.conf文件中的kafka-manager.zkhosts项，让它指向你环境中的ZooKeeper地址，比如：</p><pre><code>kafka-manager.zkhosts=&quot;localhost:2181&quot;
</code></pre><p>之后，运行以下命令启动Kafka Manager：</p><pre><code>bin/kafka-manager -Dconfig.file=conf/application.conf -Dhttp.port=8080
</code></pre><p>该命令指定了要读取的配置文件以及要启动的监听端口。现在，我们打开浏览器，输入对应的IP:8080，就可以访问Kafka Manager了。下面这张图展示了我在Kafka Manager中添加集群的主界面。</p><p><img src="https://static001.geekbang.org/resource/image/b4/66/b492ae08e527e295d29da65d07ad9566.png?wh=1950*1785" alt=""></p><p>注意，要勾选上Enable JMX Polling，这样你才能监控Kafka的各种JMX指标。下图就是Kafka Manager框架的主界面。</p><p><img src="https://static001.geekbang.org/resource/image/99/14/990944c78f22adc6f6c836d489eade14.png?wh=1950*1098" alt=""></p><p>从这张图中，我们可以发现，Kafka Manager清晰地列出了当前监控的Kafka集群的主题数量、Broker数量等信息。你可以点击顶部菜单栏的各个条目去探索其他功能。</p><p>除了丰富的监控功能之外，Kafka Manager还提供了很多运维管理操作，比如执行主题的创建、Preferred Leader选举等。在生产环境中，这可能是一把双刃剑，毕竟这意味着每个访问Kafka Manager的人都能执行这些运维操作。这显然是不能被允许的。因此，很多Kafka Manager用户都有这样一个诉求：把Kafka Manager变成一个纯监控框架，关闭非必要的管理功能。</p><p>庆幸的是，Kafka Manager提供了这样的功能。<strong>你可以修改config下的application.conf文件，删除application.features中的值</strong>。比如，如果我想禁掉Preferred Leader选举功能，那么我就可以删除对应KMPreferredReplicaElectionFeature项。删除完之后，我们重启Kafka Manager，再次进入到主界面，我们就可以发现之前的Preferred Leader Election菜单项已经没有了。</p><p><img src="https://static001.geekbang.org/resource/image/16/01/16b5ac5eeb4f32f872265ec91d130401.png?wh=1950*1343" alt=""></p><p>总之，作为一款非常强大的Kafka开源监控框架，Kafka Manager提供了丰富的实时监控指标以及适当的管理功能，非常适合一般的Kafka集群监控，值得你一试。</p><h2>Burrow</h2><p>我要介绍的第二个Kafka开源监控框架是Burrow。<strong>Burrow是LinkedIn开源的一个专门监控消费者进度的框架</strong>。事实上，当初其开源时，我对它还是挺期待的。毕竟是LinkedIn公司开源的一个框架，而LinkedIn公司又是Kafka创建并发展壮大的地方。Burrow应该是有机会成长为很好的Kafka监控框架的。</p><p>然而令人遗憾的是，它后劲不足，发展非常缓慢，目前已经有几个月没有更新了。而且这个框架是用Go写的，安装时要求必须有Go运行环境，所以，Burrow在普及率上不如其他框架。另外，Burrow没有UI界面，只是开放了一些HTTP Endpoint，这对于“想偷懒”的运维来说，更是一个减分项。</p><p>如果你要安装Burrow，必须要先安装Golang语言环境，然后依次运行下列命令去安装Burrow：</p><pre><code>$ go get github.com/linkedin/Burrow
$ cd $GOPATH/src/github.com/linkedin/Burrow
$ dep ensure
$ go install
</code></pre><p>等一切准备就绪，执行Burrow启动命令就可以了。</p><pre><code>$GOPATH/bin/Burrow --config-dir /path/containing/config
</code></pre><p>总体来说，Burrow目前提供的功能还十分有限，普及率和知名度都是比较低的。不过，它的好处是，该项目的主要贡献者是LinkedIn团队维护Kafka集群的主要负责人，所以质量是很有保证的。如果你恰好非常熟悉Go语言生态，那么不妨试用一下Burrow。</p><h2>JMXTrans + InfluxDB + Grafana</h2><p>除了刚刚说到的专属开源Kafka监控框架之外，其实现在更流行的做法是，<strong>在一套通用的监控框架中监控Kafka</strong>，比如使用<strong>JMXTrans + InfluxDB + Grafana的组合</strong>。由于Grafana支持对<strong>JMX指标</strong>的监控，因此很容易将Kafka各种JMX指标集成进来。</p><p>我们来看一张生产环境中的监控截图。图中集中了很多监控指标，比如CPU使用率、GC收集数据、内存使用情况等。除此之外，这个仪表盘面板还囊括了很多关键的Kafka JMX指标，比如BytesIn、BytesOut和每秒消息数等。将这么多数据统一集成进一个面板上直观地呈现出来，是这套框架非常鲜明的特点。</p><p><img src="https://static001.geekbang.org/resource/image/22/5a/22f68c477d80919f4170da11c4fc8d5a.jpeg?wh=1950*925" alt=""></p><p>与Kafka Manager相比，这套监控框架的优势在于，你可以在一套监控框架中同时监控企业的多个关键技术组件。特别是<strong>对于那些已经搭建了该监控组合的企业来说，直接复用这套框架可以极大地节省运维成本，不失为一个好的选择</strong>。</p><h2>Confluent Control Center</h2><p>最后，我们来说说Confluent公司发布的Control Center。这是目前已知的最强大的Kafka监控框架了。</p><p><strong>Control Center不但能够实时地监控Kafka集群，而且还能够帮助你操作和搭建基于Kafka的实时流处理应用。更棒的是，Control Center提供了统一式的主题管理功能。你可以在这里享受到Kafka主题和Schema的一站式管理服务。</strong></p><p>下面这张图展示了Control Center的主题管理主界面。从这张图中，我们可以直观地观测到整个Kafka集群的主题数量、ISR副本数量、各个主题对应的TPS等数据。当然，Control Center提供的功能远不止这些，你能想到的所有Kafka运维管理和监控功能，Control Center几乎都能提供。</p><p><img src="https://static001.geekbang.org/resource/image/f0/e2/f067a91f29d24c181ddc19ace163f3e2.png?wh=1193*629" alt=""></p><p>不过，如果你要使用Control Center，就必须使用Confluent Kafka Platform企业版。换句话说，Control Center不是免费的，你需要付费才能使用。如果你需要一套很强大的监控框架，你可以登录Confluent公司官网，去订购这套真正意义上的企业级Kafka监控框架。</p><h2>小结</h2><p>其实，除了今天我介绍的Kafka Manager、Burrow、Grafana和Control Center之外，市面上还散落着很多开源的Kafka监控框架，比如Kafka Monitor、Kafka Offset Monitor等。不过，这些框架基本上已经停止更新了，有的框架甚至好几年都没有人维护了，因此我就不详细展开了。如果你是一名开源爱好者，可以试着到开源社区中贡献代码，帮助它们重新焕发活力。</p><p>值得一提的是，国内最近有个Kafka Eagle框架非常不错。它是国人维护的，而且目前还在积极地演进着。根据Kafka Eagle官网的描述，它支持最新的Kafka  2.x版本，除了提供常规的监控功能之外，还开放了告警功能（Alert），非常值得一试。</p><p>总之，每个框架都有自己的特点和价值。Kafka Manager框架适用于基本的Kafka监控，Grafana+InfluxDB+JMXTrans的组合适用于已经具有较成熟框架的企业。对于其他的几个监控框架，你可以把它们作为这两个方案的补充，加入到你的监控解决方案中。</p><p><img src="https://static001.geekbang.org/resource/image/51/7a/514225120dca48f08b20b09400217b7a.jpg?wh=2069*2560" alt=""></p><h2>开放讨论</h2><p>如果想知道某台Broker上是否存在请求积压，我们应该监控哪个JMX指标？</p><p>欢迎写下你的思考和答案，我们一起讨论。如果你觉得有所收获，也欢迎把文章分享给你的朋友。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/62/30/a8df1a4e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张亮</span>
  </div>
  <div class="_2_QraFYR_0">https:&#47;&#47;github.com&#47;didi&#47;Logi-KafkaManager   一站式Apache Kafka集群指标监控与运维管控平台，1600+Star，650+ 用户的选择，绝对的好用的Kafka监控利器！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-15 11:11:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4d/fd/0aa0e39f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>许童童</span>
  </div>
  <div class="_2_QraFYR_0">感觉Grafana+InfluxDB这一套，可以用于任何语言，还可以自定义接口出来加入监控。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-27 15:06:18</div>
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
  <div class="_2_QraFYR_0">kafka集群监控工具，免费的功能少，功能强大的收费。看自己的情况选择了，作为技术关注点还在于这些工具的实现原理。<br>不过任何监控工具，估计都类似，以下是猜测的：<br>1：获取监控数据，通常是日志信息，加埋点或者利用OS的功能获取<br>2：存储监控数据，未经清洗的数据<br>3：清洗数据，格式化数据，聚合数据，汇总数据<br>4：展示监控信息<br>5：功能需求没问题后就是各种优化了，比如：UI展示优化&#47;获取数据不丢消息的优化&#47;展示数据的性能优化&#47;功能优化，可以加各种报警设置，给出问题产生的主要场景和解决思路。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-24 08:35:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/e9/f0/7c8ea410.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>李沁</span>
  </div>
  <div class="_2_QraFYR_0">搜了一下kafka manager，已经找不到了，最后发现已经改名叫CMAK(Cluster Manager for Apache Kafka)</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯嗯，各类框架发展得太快了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-30 17:21:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/ee/d6/a9b34bd3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Joypan</span>
  </div>
  <div class="_2_QraFYR_0">Grafana监控，kafka manager运维管理</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-03 00:56:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/s3l1cvljHFoibnZEHaLl4IJ1ryRFI0zYePAVdYfhfcoyr20uEaf1ibZhMHb5aze8Fuib8FnoYbsC0mnggIeYIjVzg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>午月十三</span>
  </div>
  <div class="_2_QraFYR_0">新发现的一款管理工具kafka-console-ui，感觉对初学者使用特别友好，用着挺简单的：https:&#47;&#47;github.com&#47;xxd763795151&#47;kafka-console-ui</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-19 14:06:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张瑞松</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，我们最近在做kafka数据层面的监控，就是消息情况的监控，包括状态，生产消费时差，消息趋势等，这一方面的工具，老师这边有好的建议吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 目前监控基本上是找主流的监控框架，支持JMX监控的就行，比如Prometheus</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-06 23:37:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/8f/76/1d8be696.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>kennyji</span>
  </div>
  <div class="_2_QraFYR_0">有个地方不太准确 BytesInPerSec是leader副本的入流量 并不等于网卡流量 要关注带宽指标还是需要具体看网卡的流量指标</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: hmmmm.... 好像我没有说BytesInPerSec=网卡流量，BytesInPerSec是broker端的入站流量。如果接近带宽，需要调整broker上的负载。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-17 09:25:22</div>
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
  <div class="_2_QraFYR_0">请求积压，监控两个idle就好吧？但是具体哪些请求积压和哪些ip的请求，这块还不清楚，求指教。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 目前只能监控是否存在请求积压，无法确认到底是那些请求积压的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-28 08:51:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4d/fd/0aa0e39f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>许童童</span>
  </div>
  <div class="_2_QraFYR_0">监控records-lag-max 和 records-lead-min，它们分别表示此消费者在测试窗口时间内曾经达到的最大的 Lag 值和最小的 Lead 值。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-27 15:11:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/4e/29/1be3dd40.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ykkk88</span>
  </div>
  <div class="_2_QraFYR_0">老师 生产环境建议用confluent免费版本的kafka么 比如5.3版本基于apache kafka 2.3的？我们想自己搭kafka 在confluent和apache里面选一个，都是免费的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: confluent免费版不错的，可以用：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-27 08:46:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/18/6b/a1448af1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>贝影</span>
  </div>
  <div class="_2_QraFYR_0">Burrow主要就是用于动态监控lag的，个人感觉其实很不错，也学了其监控算法迁移到自己监控平台上</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-10 10:03:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/6b/0f/293b999c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>旋风</span>
  </div>
  <div class="_2_QraFYR_0">老师，JMXTrans + InfluxDB + Grafana 的监控方案的grafana的模板可以提供一下吗</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-25 02:03:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/e8/a0/c2daafdb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>A_阿海</span>
  </div>
  <div class="_2_QraFYR_0">滴滴新开源了一个，可以参考(https:&#47;&#47;github.com&#47;didi&#47;Logi-KafkaManager)</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-22 22:28:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/8b/4b/15ab499a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>风轻扬</span>
  </div>
  <div class="_2_QraFYR_0">老师，您说的去kafka官网搜索--object-names，这个参数在官网怎么搜索啊？官网左侧的tab都点开搜了，没有看到。☹️</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 主要是这里：http:&#47;&#47;kafka.apache.org&#47;documentation&#47;#monitoring</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-26 07:49:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>一眼万年</span>
  </div>
  <div class="_2_QraFYR_0">Kafka Eagle会导致zookeeper连接占满不释放</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-27 09:33:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/fb/49/e14b1c54.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>稳健的少年</span>
  </div>
  <div class="_2_QraFYR_0">老师，Kafka Manager貌似不支持Kafka 2.x版本吧</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以支持</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-27 08:26:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek6238</span>
  </div>
  <div class="_2_QraFYR_0"><br>bin&#47;kafka-run-class.sh kafka.tools.JmxTool --object-name kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec --jmx-url service:jmx:rmi:&#47;&#47;&#47;jndi&#47;rmi:&#47;&#47;:9997&#47;jmxrmi --date-format &quot;YYYY-MM-dd HH:mm:ss&quot; --attributes OneMinuteRate --reporting-interval 1000</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-14 19:35:42</div>
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
  <div class="_2_QraFYR_0">Grafana可以监控K8S ,中间件集群redis   kafka  zk, mysql , es, etcd, minio, maogodb的 CPU  内存  磁盘 等数据指标  挺好的</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-24 16:58:33</div>
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
  <div class="_2_QraFYR_0">尝试了一把kafka eagle</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-03 15:23:50</div>
  </div>
</div>
</div>
</li>
</ul>