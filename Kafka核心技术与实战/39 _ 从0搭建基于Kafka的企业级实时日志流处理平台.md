<audio title="39 _ 从0搭建基于Kafka的企业级实时日志流处理平台" src="https://static001.geekbang.org/resource/audio/61/dc/6107b53eb86da2856c87be5fff9d11dc.mp3" controls="controls"></audio> 
<p>你好，我是胡夕。今天我要和你分享的主题是：从0搭建基于Kafka的企业级实时日志流处理平台。</p><p>简单来说，我们要实现一些大数据组件的组合，就如同玩乐高玩具一样，把它们“插”在一起，“拼”成一个更大一点的玩具。</p><p>在任何一个企业中，服务器每天都会产生很多的日志数据。这些数据内容非常丰富，包含了我们的<strong>线上业务数据</strong>、<strong>用户行为数据</strong>以及<strong>后端系统数据</strong>。实时分析这些数据，能够帮助我们更快地洞察潜在的趋势，从而有针对性地做出决策。今天，我们就使用Kafka搭建一个这样的平台。</p><h2>流处理架构</h2><p>如果在网上搜索实时日志流处理，你应该能够搜到很多教你搭建实时流处理平台做日志分析的教程。这些教程使用的技术栈大多是Flume+Kafka+Storm、Spark Streaming或Flink。特别是Flume+Kafka+Flink的组合，逐渐成为了实时日志流处理的标配。不过，要搭建这样的处理平台，你需要用到3个框架才能实现，这既增加了系统复杂度，也提高了运维成本。</p><p>今天，我来演示一下如何使用Apache Kafka这一个框架，实现一套实时日志流处理系统。换句话说，我使用的技术栈是Kafka Connect+Kafka Core+Kafka Streams的组合。</p><!-- [[[read_end]]] --><p>下面这张图展示了基于Kafka的实时日志流处理平台的流程。</p><p><img src="https://static001.geekbang.org/resource/image/09/51/09e4dc28ea338ee69b5662596c0b8751.jpg?wh=2625*1083" alt=""></p><p>从图中我们可以看到，日志先从Web服务器被不断地生产出来，随后被实时送入到Kafka Connect组件，Kafka Connect组件对日志进行处理后，将其灌入Kafka的某个主题上，接着发送到Kafka Streams组件，进行实时分析。最后，Kafka Streams将分析结果发送到Kafka的另一个主题上。</p><p>我在专栏前面简单介绍过Kafka Connect和Kafka Streams组件，前者可以实现外部系统与Kafka之间的数据交互，而后者可以实时处理Kafka主题中的消息。</p><p>现在，我们就使用这两个组件，结合前面学习的所有Kafka知识，一起构建一个实时日志分析平台。</p><h2>Kafka Connect组件</h2><p>我们先利用Kafka Connect组件<strong>收集数据</strong>。如前所述，Kafka Connect组件负责连通Kafka与外部数据系统。连接外部数据源的组件叫连接器（Connector）。<strong>常见的外部数据源包括数据库、KV存储、搜索系统或文件系统等。</strong></p><p>今天我们使用文件连接器（File Connector）实时读取Nginx的access日志。假设access日志的格式如下：</p><pre><code>10.10.13.41 - - [13/Aug/2019:03:46:54 +0800] &quot;GET /v1/open/product_list?user_key=****&amp;user_phone=****&amp;screen_height=1125&amp;screen_width=2436&amp;from_page=1&amp;user_type=2&amp;os_type=ios HTTP/1.0&quot; 200 1321
</code></pre><p>在这段日志里，请求参数中的os_type字段目前有两个值：ios和android。我们的目标是实时计算当天所有请求中ios端和android端的请求数。</p><h3>启动Kafka Connect</h3><p>当前，Kafka Connect支持单机版（Standalone）和集群版（Cluster），我们用集群的方式来启动Connect组件。</p><p>首先，我们要启动Kafka集群，假设Broker的连接地址是localhost:9092。</p><p>启动好Kafka集群后，我们启动Connect组件。在Kafka安装目录下有个config/connect-distributed.properties文件，你需要修改下列项：</p><pre><code>bootstrap.servers=localhost:9092
rest.host.name=localhost
rest.port=8083
</code></pre><p>第1项是指定<strong>要连接的Kafka集群</strong>，第2项和第3项分别指定Connect组件开放的REST服务的<strong>主机名和端口</strong>。保存这些变更之后，我们运行下面的命令启动Connect。</p><pre><code>cd kafka_2.12-2.3.0
bin/connect-distributed.sh config/connect-distributed.properties
</code></pre><p>如果一切正常，此时Connect应该就成功启动了。现在我们在浏览器访问localhost:8083的Connect REST服务，应该能看到下面的返回内容：</p><pre><code>{&quot;version&quot;:&quot;2.3.0&quot;,&quot;commit&quot;:&quot;fc1aaa116b661c8a&quot;,&quot;kafka_cluster_id&quot;:&quot;XbADW3mnTUuQZtJKn9P-hA&quot;}
</code></pre><h3>添加File Connector</h3><p>看到该JSON串，就表明Connect已经成功启动了。此时，我们打开一个终端，运行下面这条命令来查看一下当前都有哪些Connector。</p><pre><code>$ curl http://localhost:8083/connectors
[]
</code></pre><p>结果显示，目前我们没有创建任何Connector。</p><p>现在，我们来创建对应的File Connector。该Connector读取指定的文件，并为每一行文本创建一条消息，并发送到特定的Kafka主题上。创建命令如下：</p><pre><code>$ curl -H &quot;Content-Type:application/json&quot; -H &quot;Accept:application/json&quot; http://localhost:8083/connectors -X POST --data '{&quot;name&quot;:&quot;file-connector&quot;,&quot;config&quot;:{&quot;connector.class&quot;:&quot;org.apache.kafka.connect.file.FileStreamSourceConnector&quot;,&quot;file&quot;:&quot;/var/log/access.log&quot;,&quot;tasks.max&quot;:&quot;1&quot;,&quot;topic&quot;:&quot;access_log&quot;}}'
{&quot;name&quot;:&quot;file-connector&quot;,&quot;config&quot;:{&quot;connector.class&quot;:&quot;org.apache.kafka.connect.file.FileStreamSourceConnector&quot;,&quot;file&quot;:&quot;/var/log/access.log&quot;,&quot;tasks.max&quot;:&quot;1&quot;,&quot;topic&quot;:&quot;access_log&quot;,&quot;name&quot;:&quot;file-connector&quot;},&quot;tasks&quot;:[],&quot;type&quot;:&quot;source&quot;}
</code></pre><p>这条命令本质上是向Connect REST服务发送了一个POST请求，去创建对应的Connector。在这个例子中，我们的Connector类是Kafka默认提供的<strong>FileStreamSourceConnector</strong>。我们要读取的日志文件在/var/log目录下，要发送到Kafka的主题名称为access_log。</p><p>现在，我们再次运行curl http: // localhost:8083/connectors， 验证一下刚才的Connector是否创建成功了。</p><pre><code>$ curl http://localhost:8083/connectors
[&quot;file-connector&quot;]
</code></pre><p>显然，名为file-connector的新Connector已经创建成功了。如果我们现在使用Console Consumer程序去读取access_log主题的话，应该会发现access.log中的日志行数据已经源源不断地向该主题发送了。</p><p>如果你的生产环境中有多台机器，操作也很简单，在每台机器上都创建这样一个Connector，只要保证它们被送入到相同的Kafka主题以供消费就行了。</p><h2>Kafka Streams组件</h2><p>数据到达Kafka还不够，我们还需要对其进行实时处理。下面我演示一下如何编写Kafka Streams程序来实时分析Kafka主题数据。</p><p>我们知道，Kafka Streams是Kafka提供的用于实时流处理的组件。</p><p>与其他流处理框架不同的是，它仅仅是一个类库，用它编写的应用被编译打包之后就是一个普通的Java应用程序。你可以使用任何部署框架来运行Kafka Streams应用程序。</p><p>同时，你只需要简单地启动多个应用程序实例，就能自动地获得负载均衡和故障转移，因此，和Spark Streaming或Flink这样的框架相比，Kafka Streams自然有它的优势。</p><p>下面这张来自Kafka官网的图片，形象地展示了多个Kafka Streams应用程序组合在一起，共同实现流处理的场景。图中清晰地展示了3个Kafka Streams应用程序实例。一方面，它们形成一个组，共同参与并执行流处理逻辑的计算；另一方面，它们又都是独立的实体，彼此之间毫无关联，完全依靠Kafka Streams帮助它们发现彼此并进行协作。</p><p><img src="https://static001.geekbang.org/resource/image/6f/67/6fc7c835c993b48b1ef1558e02f24f67.png?wh=1950*1291" alt=""></p><p>关于Kafka Streams的原理，我会在专栏后面进行详细介绍。今天，我们只要能够学会利用它提供的API编写流处理应用，帮我们找到刚刚提到的请求日志中ios端和android端发送请求数量的占比数据就行了。</p><h3>编写流处理应用</h3><p>要使用Kafka Streams，你需要在你的Java项目中显式地添加kafka-streams依赖。我以最新的2.3版本为例，分别演示下Maven和Gradle的配置方法。</p><pre><code>Maven:
&lt;dependency&gt;
    &lt;groupId&gt;org.apache.kafka&lt;/groupId&gt;
    &lt;artifactId&gt;kafka-streams&lt;/artifactId&gt;
    &lt;version&gt;2.3.0&lt;/version&gt;
&lt;/dependency&gt;
</code></pre><pre><code>Gradle:
compile group: 'org.apache.kafka', name: 'kafka-streams', version: '2.3.0'
</code></pre><p>现在，我先给出完整的代码，然后我会详细解释一下代码中关键部分的含义。</p><pre><code>package com.geekbang.kafkalearn;

import com.google.gson.Gson;
import org.apache.kafka.common.serialization.Serdes;
import org.apache.kafka.streams.KafkaStreams;
import org.apache.kafka.streams.StreamsBuilder;
import org.apache.kafka.streams.StreamsConfig;
import org.apache.kafka.streams.Topology;
import org.apache.kafka.streams.kstream.KStream;
import org.apache.kafka.streams.kstream.Produced;
import org.apache.kafka.streams.kstream.TimeWindows;
import org.apache.kafka.streams.kstream.WindowedSerdes;

import java.time.Duration;
import java.util.Properties;
import java.util.concurrent.CountDownLatch;

public class OSCheckStreaming {

    public static void main(String[] args) {


        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, &quot;os-check-streams&quot;);
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, &quot;localhost:9092&quot;);
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_WINDOWED_KEY_SERDE_INNER_CLASS, Serdes.StringSerde.class.getName());

        final Gson gson = new Gson();
        final StreamsBuilder builder = new StreamsBuilder();

        KStream&lt;String, String&gt; source = builder.stream(&quot;access_log&quot;);
        source.mapValues(value -&gt; gson.fromJson(value, LogLine.class)).mapValues(LogLine::getPayload)
                .groupBy((key, value) -&gt; value.contains(&quot;ios&quot;) ? &quot;ios&quot; : &quot;android&quot;)
                .windowedBy(TimeWindows.of(Duration.ofSeconds(2L)))
                .count()
                .toStream()
                .to(&quot;os-check&quot;, Produced.with(WindowedSerdes.timeWindowedSerdeFrom(String.class), Serdes.Long()));

        final Topology topology = builder.build();

        final KafkaStreams streams = new KafkaStreams(topology, props);
        final CountDownLatch latch = new CountDownLatch(1);

        Runtime.getRuntime().addShutdownHook(new Thread(&quot;streams-shutdown-hook&quot;) {
            @Override
            public void run() {
                streams.close();
                latch.countDown();
            }
        });

        try {
            streams.start();
            latch.await();
        } catch (Exception e) {
            System.exit(1);
        }
        System.exit(0);
    }
}


class LogLine {
    private String payload;
    private Object schema;

    public String getPayload() {
        return payload;
    }
}
</code></pre><p>这段代码会实时读取access_log主题，每2秒计算一次ios端和android端请求的总数，并把这些数据写入到os-check主题中。</p><p>首先，我们构造一个Properties对象。这个对象负责初始化Streams应用程序所需要的关键参数设置。比如，在上面的例子中，我们设置了bootstrap.servers参数、application.id参数以及默认的序列化器（Serializer）和解序列化器（Deserializer）。</p><p>bootstrap.servers参数你应该已经很熟悉了，我就不多讲了。这里的application.id是Streams程序中非常关键的参数，你必须要指定一个集群范围内唯一的字符串来标识你的Kafka Streams程序。序列化器和解序列化器设置了默认情况下Streams程序执行序列化和反序列化时用到的类。在这个例子中，我们设置的是String类型，这表示，序列化时会将String转换成字节数组，反序列化时会将字节数组转换成String。</p><p>构建好Properties实例之后，下一步是创建StreamsBuilder对象。稍后我们会用这个Builder去实现具体的流处理逻辑。</p><p>在这个例子中，我们实现了这样的流计算逻辑：每2秒去计算一下ios端和android端各自发送的总请求数。还记得我们的原始数据长什么样子吗？它是一行Nginx日志，只不过Connect组件在读取它后，会把它包装成JSON格式发送到Kafka，因此，我们需要借助Gson来帮助我们把JSON串还原为Java对象，这就是我在代码中创建LogLine类的原因。</p><p>代码中的mapValues调用将接收到的JSON串转换成LogLine对象，之后再次调用mapValues方法，提取出LogLine对象中的payload字段，这个字段保存了真正的日志数据。这样，经过两次mapValues方法调用之后，我们成功地将原始数据转换成了实际的Nginx日志行数据。</p><p>值得注意的是，代码使用的是Kafka Streams提供的mapValues方法。顾名思义，<strong>这个方法就是只对消息体（Value）进行转换，而不变更消息的键（Key）</strong>。</p><p>其实，Kafka Streams也提供了map方法，允许你同时修改消息Key。通常来说，我们认为<strong>mapValues要比map方法更高效</strong>，因为Key的变更可能导致下游处理算子（Operator）的重分区，降低性能。如果可能的话最好尽量使用mapValues方法。</p><p>拿到真实日志行数据之后，我们调用groupBy方法进行统计计数。由于我们要统计双端（ios端和android端）的请求数，因此，我们groupBy的Key是ios或android。在上面的那段代码中，我仅仅依靠日志行中是否包含特定关键字的方式来确定是哪一端。更正宗的做法应该是，<strong>分析Nginx日志格式，提取对应的参数值，也就是os_type的值</strong>。</p><p>做完groupBy之后，我们还需要限定要统计的时间窗口范围，即我们统计的双端请求数是在哪个时间窗口内计算的。在这个例子中，我调用了<strong>windowedBy方法</strong>，要求Kafka Streams每2秒统计一次双端的请求数。设定好了时间窗口之后，下面就是调用<strong>count方法</strong>进行统计计数了。</p><p>这一切都做完了之后，我们需要调用<strong>toStream方法</strong>将刚才统计出来的表（Table）转换成事件流，这样我们就能实时观测它里面的内容。我会在专栏的最后几讲中解释下流处理领域内的流和表的概念以及它们的区别。这里你只需要知道toStream是将一个Table变成一个Stream即可。</p><p>最后，我们调用<strong>to方法</strong>将这些时间窗口统计数据不断地写入到名为os-check的Kafka主题中，从而最终实现我们对Nginx日志进行实时分析处理的需求。</p><h3>启动流处理应用</h3><p>由于Kafka Streams应用程序就是普通的Java应用，你可以用你熟悉的方式对它进行编译、打包和部署。本例中的OSCheckStreaming.java就是一个可执行的Java类，因此直接运行它即可。如果一切正常，它会将统计数据源源不断地写入到os-check主题。</p><h3>查看统计结果</h3><p>如果我们想要查看统计的结果，一个简单的方法是使用Kafka自带的kafka-console-consumer脚本。命令如下：</p><pre><code>$ bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic os-check --from-beginning --property value.deserializer=org.apache.kafka.common.serialization.LongDeserializer --property print.key=true --property key.deserializer=org.apache.kafka.streams.kstream.TimeWindowedDeserializer --property key.deserializer.default.windowed.key.serde.inner=org.apache.kafka.common.serialization.Serdes\$StringSerde
[android@1565743788000/9223372036854775807] 1522
[ios@1565743788000/9223372036854775807] 478
[ios@1565743790000/9223372036854775807] 1912
[android@1565743790000/9223372036854775807] 5313
[ios@1565743792000/9223372036854775807] 780
[android@1565743792000/9223372036854775807] 1949
[android@1565743794000/9223372036854775807] 37
……
</code></pre><p>由于我们统计的结果是某个时间窗口范围内的，因此承载这个统计结果的消息的Key封装了该时间窗口信息，具体格式是：<strong>[ios或android@开始时间/结束时间]</strong>，而消息的Value就是一个简单的数字，表示这个时间窗口内的总请求数。</p><p>如果把上面ios相邻输出行中的开始时间相减，我们就会发现，它们的确是每2秒输出一次，每次输出会同时计算出ios端和android端的总请求数。接下来，你可以订阅这个Kafka主题，将结果实时导出到你期望的其他数据存储上。</p><h2>小结</h2><p>至此，基于Apache Kafka的实时日志流处理平台就简单搭建完成了。在搭建的过程中，我们只使用Kafka这一个大数据框架就完成了所有组件的安装、配置和代码开发。比起Flume+Kafka+Flink这样的技术栈，纯Kafka的方案在运维和管理成本上有着极大的优势。如果你打算从0构建一个实时流处理平台，不妨试一下Kafka Connect+Kafka Core+Kafka Streams的组合。</p><p>其实，Kafka Streams提供的功能远不止做计数这么简单。今天，我只是为你展示了Kafka Streams的冰山一角。在专栏的后几讲中，我会重点向你介绍Kafka Streams组件的使用和管理，敬请期待。</p><p><img src="https://static001.geekbang.org/resource/image/fd/9f/fdb79f35b43cab31edb945b977cc609f.jpg?wh=2069*2560" alt=""></p><h2>开放讨论</h2><p>请比较一下Flume+Kafka+Flink方案和纯Kafka方案，思考一下它们各自的优劣之处。在实际场景中，我们该如何选择呢？</p><p>欢迎写下你的思考和答案，我们一起讨论。如果你觉得有所收获，也欢迎把文章分享给你的朋友。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/f1/55/8ac4f169.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>陈国林</span>
  </div>
  <div class="_2_QraFYR_0">老师好，说下我这边的一些实践。19年一直在做容器日志平台，我们目前的方案是 Fluentd + Kafka + ELK。使用Fluentd做为容器平台的采集器实时采集数据，采集完之后数据写入Kafka通过Kafka进行解耦，使用Logstash消费后写入ES。这套方案目前在容器环境下应该可以说是标配</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 棒棒：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-03 22:03:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/a5/56/d6732e61.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小刀</span>
  </div>
  <div class="_2_QraFYR_0">老师，上述Kafka Connect+Kafka Core+Kafka Streams例子中，生产者和消费者分别是什么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 此时，生产者和消费者化身成这个大平台的小组件了。Connect中只有producer，将读取的日志行数据写入到Kafka源主题中。Streams中既有producer也有consumer：producer负责将计算结果实时写入到目标Kafka主题；consumer负责从源主题中读取消息供下游实时计算之用。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-03 19:24:49</div>
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
  <div class="_2_QraFYR_0">胡老师，对于Stream的处理和之前的topic-message，我感觉没什么大的区别，感觉流程是类似的。只不过是在以前的consumer中额外添加了producer的逻辑，把处理结果发送到另一个topic中。感觉不用这里说的stream也能实现一样的效果。我不是个明白本质的区别是什么，麻烦能解释一下吗？谢谢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Kafka Streams提供了consume-process-produce的原子性操作，也就是端到端的EOS。如果你自己实现代价很高</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-05 11:04:10</div>
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
  <div class="_2_QraFYR_0">老师，示例中开启Connect后启动读取的是本机的nginx日志，但如果nginx日志是在其他机器上面，那Connect是不是支持远程读的还是怎么样可以读取到其他机器的日志？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 在Nginx日志机器上开启，因为目前File Connector只支持从本地文件读取</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-14 23:34:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/dd/09/feca820a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>helloworld</span>
  </div>
  <div class="_2_QraFYR_0">写的不太明白啊, 难道每一个nginx服务器上都要部署kafka吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-10 16:50:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/b2/8e/0911bf35.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>w</span>
  </div>
  <div class="_2_QraFYR_0">老师我想问一下。Kafka Connect 是一个单独的组件么？类似agent一样，可以在目标采集机器（非kafka集群）上部署？那如果要用，岂不是每个业务机器都要装个kafka？<br>单机模式跟集群模式有啥区别呢？没太懂。比如kafka集群上启动单机模式的connect ？不行么？<br>最终操作，都往同一个topic里扔就好了?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是单独的组件。需要单独部署。<br>kafka集群上启动单机模式的connect  --- 这是可以的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-25 17:24:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/37/2b/b32f1d66.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Ball</span>
  </div>
  <div class="_2_QraFYR_0">老师我有问题要请教下，添加 Connector 步骤里面是用 http REST 接口新建的，那新建的 Connector 是跑在 Broker 里面还是说又启动了一个新的 Java 进程执行 Connector？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 新的java进程</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-14 16:10:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/ed/4c/8674b6ad.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>timmy21</span>
  </div>
  <div class="_2_QraFYR_0">老师如果有多个分区，并且消息写入是随机的。那么多个kafka streams实例在对os_type进行group_by统计时，需要相互之间传输数据进行shuffle操作吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 需要的。Kafka Streams使用特殊的repartition topics来保存shuffle后的数据</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-03 18:30:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/a3/e2/439731b2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>霍格沃兹小学徒</span>
  </div>
  <div class="_2_QraFYR_0">老师 想咨询个问题 我这边如果日志不是来自于文件 而是来自于telenet的输出 需要怎么做日志实时分析。 而且需要有上万个telenet实例一起在输出 我需要分别分析每个实例按照某种统一规则</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以为每个telnet的数据流编制一个key来管理</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-25 00:11:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_6e00ab</span>
  </div>
  <div class="_2_QraFYR_0">为什么消费者取出来的值是乱码的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 因为是二进制字节序列，需要反序列化</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-28 16:38:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/1d/64/52a5863b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>大土豆</span>
  </div>
  <div class="_2_QraFYR_0">问个大数据相关的问题，一定要读取和写入都在hdfs上才是离线计算吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不一定。离线计算通常是指延时比较高的batch computing，和用什么存储没有强绑定关系</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-23 10:25:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/64/9b/d1ab239e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>J.Smile</span>
  </div>
  <div class="_2_QraFYR_0">“另一方面，它们又都是独立的实体，彼此之间毫无关联，完全依靠 Kafka Streams 帮助它们发现彼此并进行协作。”<br>————————————————————<br>完全依靠“kafka cluster”这个吧！？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 依靠Kafka Streams找到Kafka Cluster上的彼此。这么说是不是好点：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-21 20:40:26</div>
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
  <div class="_2_QraFYR_0">connect与filebeat比单文件收集性能比较来说哪个更高？我们用filebeat单文件收集性能到瓶颈了，我们现在的做法是减小单文件的，大小，来提高并行度，不知道connect性能如何</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: hmmm..... 这个只能靠测试才能知道了：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-03 01:11:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/dc/08/64f5ab52.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>陈斌</span>
  </div>
  <div class="_2_QraFYR_0">Kafka Stream 能不能将消费者Topic的一个消息分解转换成生产者的多条消息，而且其中还涉及可以把消息数据存入Redis中？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-09 11:25:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/m7fLWyJrnwEPoIefiaxusQRh6D1Nq7PCXA8RiaxkmzdNEmFARr5q8L4qouKNaziceXia92an8hzYa5MLic6N6cNMEoQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>alex_lai</span>
  </div>
  <div class="_2_QraFYR_0">Kafka stream 的输出必须要返回到topic么？ 我可以直接用stream 的api输出到我的目标db么？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-22 03:25:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/16/20/ce768739.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>L.</span>
  </div>
  <div class="_2_QraFYR_0">老师，如果要将处理后的结果写到另一个 kafka 集群的 topic，应该怎么做呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 使用KStream的to方法</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-15 09:45:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>InfoQ_ad0a52af1586</span>
  </div>
  <div class="_2_QraFYR_0">这个案例运行了下没有两秒钟就输出，老师能怎么排查下是什么问题吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: .windowedBy(TimeWindows.of(Duration.ofSeconds(2L)))<br><br>改长点</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-22 21:58:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/cf/81/96f656ef.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>杨逸林</span>
  </div>
  <div class="_2_QraFYR_0">我看了有点多的 SpringCloud + Kafka Stream 整合的，按照 Spring 官方给的教学视频学，结果到头来还是只会改下字符串（从一个 Topic 监听数据，送到另一个 Topic 做数据清洗）。Kafka 官网都是像你写的 demo 一样的，其实还是有点懵的。。。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 其实我一直有个想法：在学习任何大数据流式处理框架前，我们至少要对Java的Stream和Lambda表达式有一定的了解，这样我们才能更好地理解如何把操作算子组合在一起。比如要理解基本的有状态操作算子和无状态操作算子都有哪些等</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-11 21:16:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/0c/52/f25c3636.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>长脖子树</span>
  </div>
  <div class="_2_QraFYR_0">用过kafka connector，直接在集群上启动单机的connector连接mqtt，但感觉这种方案的扩展性不高</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Connect是可以集群搭建的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-11 10:06:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/5b/08/b0b0db05.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>丁丁历险记</span>
  </div>
  <div class="_2_QraFYR_0">说好的从0 开始的，kafka 集群怎么装。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 哈哈哈，有关搭建集群的内容在前面的课程中：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-24 22:44:55</div>
  </div>
</div>
</div>
</li>
</ul>