<audio title="41 _ Kafka Streams DSL开发实例" src="https://static001.geekbang.org/resource/audio/5a/d3/5a1bc6f3e3055466d64dc0dc565db9d3.mp3" controls="controls"></audio> 
<p>你好，我是胡夕。今天我要和你分享的主题是：Kafka Streams DSL开发实例。</p><p>DSL，也就是Domain Specific Language，意思是领域特定语言。它提供了一组便捷的API帮助我们实现流式数据处理逻辑。今天，我就来分享一些Kafka Streams中的DSL开发方法以及具体实例。</p><h2>Kafka Streams背景介绍</h2><p>在上一讲中，我们提到流处理平台是专门处理无限数据集的引擎。就Kafka Streams而言，它仅仅是一个客户端库。所谓的Kafka Streams应用，就是调用了Streams API的普通Java应用程序。只不过在Kafka Streams中，流处理逻辑是用<strong>拓扑</strong>来表征的。</p><p>一个拓扑结构本质上是一个有向无环图（DAG），它由多个处理节点（Node）和连接节点的多条边组成，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/da/22/da3db610f959e0952a76d6b06d249c22.jpg?wh=1334*1674" alt=""></p><p>图中的节点也称为处理单元或Processor，它封装了具体的事件处理逻辑。Processor在其他流处理平台也被称为操作算子。常见的操作算子包括转换（map）、过滤（filter）、连接（join）和聚合（aggregation）等。后面我会详细介绍几种常见的操作算子。</p><p>大体上，Kafka Streams开放了两大类API供你定义Processor逻辑。</p><!-- [[[read_end]]] --><p>第1类就是我刚刚提到的DSL，它是声明式的函数式API，使用起来感觉和SQL类似，你不用操心它的底层是怎么实现的，你只需要调用特定的API告诉Kafka Streams你要做什么即可。</p><p>举个简单的例子，你可以看看下面这段代码，尝试理解下它是做什么的。</p><pre><code>movies.filter((title, movie) -&gt; movie.getGenre().equals(&quot;动作片&quot;)).xxx()...
</code></pre><p>这段代码虽然用了Java 8的Lambda表达式，但从整体上来看，它要做的事情应该还是很清晰的：它要从所有Movie事件中过滤出影片类型是“动作片”的事件。这就是DSL声明式API的实现方式。</p><p>第2类则是命令式的低阶API，称为Processor API。比起DSL，这组API提供的实现方式更加灵活。你可以编写自定义的算子来实现一些DSL天然没有提供的处理逻辑。事实上，DSL底层也是用Processor API实现的。</p><p>目前，Kafka Streams DSL提供的API已经很丰富了，基本上能够满足我们大部分的处理逻辑需求，我今天重点介绍一下DSL的使用方法。</p><p><strong>不论是用哪组API实现，所有流处理应用本质上都可以分为两类：有状态的（Stateful）应用和无状态的（Stateless）应用</strong>。</p><p>有状态的应用指的是应用中使用了类似于连接、聚合或时间窗口（Window）的API。一旦调用了这些API，你的应用就变为有状态的了，也就是说你需要让Kafka Streams帮你保存应用的状态。</p><p>无状态的应用是指在这类应用中，某条消息的处理结果不会影响或依赖其他消息的处理。常见的无状态操作包括事件转换以及刚刚那个例子中的过滤等。</p><h2>关键概念</h2><p>了解了这些背景之后，你还需要掌握一些流处理领域内的关键概念，即流、表以及流表二元性，还有时间和时间窗口。</p><h3>流表二元性</h3><p>首先，我来介绍一下流处理中流和表的概念，以及它们之间的关系。</p><p>流就是一个永不停止（至少理论上是这样的）的事件序列，而表和关系型数据库中的概念类似，是一组行记录。在流处理领域，两者是有机统一的：<strong>流在时间维度上聚合之后形成表，表在时间维度上不断更新形成流，这就是所谓的流表二元性（Duality of Streams and Tables）</strong>。流表二元性在流处理领域内的应用是Kafka框架赖以成功的重要原因之一。</p><p>下面这张图展示了表转换成流，流再转换成表的全过程。</p><p><img src="https://static001.geekbang.org/resource/image/3f/ee/3f056c4ba024d7169df804e161a37bee.jpg?wh=3544*2250" alt=""></p><p>刚开始时，表中只有一条记录“张三：1”。将该条记录转成流，变成了一条事件。接着，表增加了新记录“李四：1”。针对这个变更，流中也增加了对应的新事件。之后，表中张三的对应值，从1更新为2，流也增加了相应的更新事件。最后，表中添加了新数据“王五：1”，流也增加了新记录。至此，表转换成流的工作就完成了。</p><p>从这个过程中我们可以看出，流可以看作是表的变更事件日志（Changelog）。与之相反的是，流转换成表的过程，可以说是这个过程的逆过程：我们为流中的每条事件打一个快照（Snapshot），就形成了表。</p><p>流和表的概念在流处理领域非常关键。在Kafka Streams DSL中，流用<strong>KStream</strong>表示，而表用<strong>KTable</strong>表示。</p><p>Kafka Streams还定义了<strong>GlobalKTable</strong>。本质上它和KTable都表征了一个表，里面封装了事件变更流，但是它和KTable的最大不同在于，当Streams应用程序读取Kafka主题数据到GlobalKTable时，它会读取主题<strong>所有分区的数据</strong>，而对KTable而言，Streams程序实例只会读取<strong>部分分区的数据</strong>，这主要取决于Streams实例的数量。</p><h3>时间</h3><p>在流处理领域内，精确定义事件时间是非常关键的：一方面，它是决定流处理应用能否实现正确性的前提；另一方面，流处理中时间窗口等操作依赖于时间概念才能正常工作。</p><p>常见的时间概念有两类：事件发生时间（Event Time）和事件处理时间（Processing Time）。理想情况下，我们希望这两个时间相等，即事件一旦发生就马上被处理，但在实际场景中，这是不可能的，<strong>Processing Time永远滞后于Event Time</strong>，而且滞后程度又是一个高度变化，无法预知，就像“Streaming Systems”一书中的这张图片所展示的那样：</p><p><img src="https://static001.geekbang.org/resource/image/ab/92/ab8e5206729b484b0a16f2bb9de3a792.png?wh=1179*1274" alt=""></p><p>该图中的45°虚线刻画的是理想状态，即Event Time等于Processing Time，而粉色的曲线表征的是真实情况，即Processing Time落后于Event Time，而且落后的程度（Lag）不断变化，毫无规律。</p><p>如果流处理应用要实现结果的正确性，就必须要使用基于Event Time的时间窗口，而不能使用基于Processing Time的时间窗口。</p><h3>时间窗口</h3><p>所谓的时间窗口机制，就是将流数据沿着时间线切分的过程。常见的时间窗口包括：固定时间窗口（Fixed Windows）、滑动时间窗口（Sliding Windows）和会话窗口（Session Windows）。Kafka Streams同时支持这三类时间窗口。在后面的例子中，我会详细介绍如何使用Kafka Streams API实现时间窗口功能。</p><h2>运行WordCount实例</h2><p>好了，关于Kafka Streams及其DSL的基本概念我都阐述完了，下面我给出大数据处理领域的Hello World实例：WordCount程序。</p><p>每个大数据处理框架第一个要实现的程序基本上都是单词计数。我们来看下Kafka Streams DSL如何实现WordCount。我先给出完整代码，稍后我会详细介绍关键部分代码的含义以及运行它的方法。</p><pre><code>package kafkalearn.demo.wordcount;


import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.common.serialization.Serdes;
import org.apache.kafka.streams.KafkaStreams;
import org.apache.kafka.streams.StreamsBuilder;
import org.apache.kafka.streams.StreamsConfig;
import org.apache.kafka.streams.kstream.KStream;
import org.apache.kafka.streams.kstream.KTable;
import org.apache.kafka.streams.kstream.Produced;


import java.util.Arrays;
import java.util.Locale;
import java.util.Properties;
import java.util.concurrent.CountDownLatch;


public final class WordCountDemo {
    public static void main(final String[] args) {
        final Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, &quot;wordcount-stream-demo&quot;);
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, &quot;localhost:9092&quot;);
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass().getName());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass().getName());
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, &quot;earliest&quot;);


        final StreamsBuilder builder = new StreamsBuilder();


        final KStream&lt;String, String&gt; source = builder.stream(&quot;wordcount-input-topic&quot;);


        final KTable&lt;String, Long&gt; counts = source
            .flatMapValues(value -&gt; Arrays.asList(value.toLowerCase(Locale.getDefault()).split(&quot; &quot;)))
            .groupBy((key, value) -&gt; value)
            .count();


        counts.toStream().to(&quot;wordcount-output-topic&quot;, Produced.with(Serdes.String(), Serdes.Long()));


        final KafkaStreams streams = new KafkaStreams(builder.build(), props);
        final CountDownLatch latch = new CountDownLatch(1);


        Runtime.getRuntime().addShutdownHook(new Thread(&quot;wordcount-stream-demo-jvm-hook&quot;) {
            @Override
            public void run() {
                streams.close();
                latch.countDown();
            }
        });


        try {
            streams.start();
            latch.await();
        } catch (final Throwable e) {
            System.exit(1);
        }
        System.exit(0)
</code></pre><p>在程序开头，我构造了一个Properties对象实例，对Kafka Streams程序的关键参数进行了赋值，比如application id、bootstrap servers和默认的KV序列化器（Serializer）和反序列化器（Deserializer）。其中，application id是Kafka Streams应用的唯一标识，必须要显式地指定。默认的KV序列化器、反序列化器是为消息的Key和Value进行序列化和反序列化操作的。</p><p>接着，我构造了一个StreamsBuilder对象，并使用该对象实例创建了一个KStream，这个KStream从名为wordcount-input-topic的Kafka主题读取消息。该主题消息由一组单词组成，单词间用空格分割，比如zhangsan lisi wangwu。</p><p>由于我们要进行单词计数，所以就需要将消息中的单词提取出来。有了前面的概念介绍，你应该可以猜到，KTable是很合适的存储结构，因此，下一步就是将刚才的这个KStream转换成KTable。</p><p>我们先对单词进行分割，这里我用到了flatMapValues方法，代码中的Lambda表达式实现了从消息中提取单词的逻辑。由于String.split()方法会返回多个单词，因此我们使用flatMapValues而不是mapValues。原因是，前者能够将多个元素“打散”成一组单词，而如果使用后者，我们得到的就不是一组单词，而是多组单词了。</p><p>这些都做完之后，程序调用groupBy方法对单词进行分组。由于是计数，相同的单词必须被分到一起，然后就是调用count方法对每个出现的单词进行统计计数，并保存在名为counts的KTable对象中。</p><p>最后，我们将统计结果写回到Kafka中。由于KTable是表，是静态的数据，因此这里要先将其转换成KStream，然后再调用to方法写入到名为wordcount-output-topic的主题中。此时，counts中事件的Key是单词，而Value是统计个数，因此我们在调用to方法时，同时指定了Key和Value的序列化器，分别是字符串序列化器和长整型序列化器。</p><p>至此，Kafka Streams的流计算逻辑就编写完了，接下来就是构造KafkaStreams实例并启动它了。通常来说，这部分的代码都是类似的，即调用start方法启动整个流处理应用，以及配置一个JVM关闭钩子（Shutdown Hook）实现流处理应用的关闭等。</p><p>总体来说，Kafka Streams DSL实现WordCount的方式还是很简单的，仅仅调用几个操作算子就轻松地实现了分布式的单词计数实时处理功能。事实上，现在主流的实时流处理框架越来越倾向于这样的设计思路，即通过提供丰富而便捷的开箱即用操作算子，简化用户的开发成本，采用类似于搭积木的方式快捷地构建实时计算应用。</p><p>待启动该Java程序之后，你需要创建出对应的输入和输出主题，并向输入主题不断地写入符合刚才所说的格式的单词行，之后，你需要运行下面的命令去查看输出主题中是否正确地统计了你刚才输入的单词个数：</p><pre><code>bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 \
    --topic wordcount-output-topic \
    --from-beginning \
    --formatter kafka.tools.DefaultMessageFormatter \
    --property print.key=true \
    --property print.value=true \
    --property key.deserializer=org.apache.kafka.common.serialization.StringDeserializer \
    --property value.deserializer=org.apache.kafka.common.serialization.LongDeserializer
</code></pre><h2>开发API</h2><p>介绍了具体的例子之后，我们来看下Kafka Streams还提供了哪些功能强大的API。我们可以重点关注两个方面，一个是常见的操作算子，另一个是时间窗口API。</p><h3>常见操作算子</h3><p>操作算子的丰富程度和易用性是衡量流处理框架受欢迎程度的重要依据之一。Kafka Streams DSL提供了很多开箱即用的操作算子，大体上分为两大类：<strong>无状态算子和有状态算子</strong>。下面我就向你分别介绍几个经常使用的算子。</p><p><strong>在无状态算子中，filter的出场率是极高的</strong>。它执行的就是过滤的逻辑。依然拿WordCount为例，假设我们只想统计那些以字母s开头的单词的个数，我们可以在执行完flatMapValues后增加一行代码，代码如下：</p><pre><code>.filter(((key, value) -&gt; value.startsWith(&quot;s&quot;)))
</code></pre><p><strong>另一个常见的无状态算子当属map一族了</strong>。Streams DSL提供了很多变体，比如map、mapValues、flatMap和flatMapValues。我们已经见识了flatMapValues的威力，其他三个的功能也是类似的，只是所有带Values的变体都只对消息体执行转换，不触及消息的Key，而不带Values的变体则能修改消息的Key。</p><p>举个例子，假设当前消息没有Key，而Value是单词本身，现在我们想要将消息变更成这样的KV对，即Key是单词小写，而Value是单词长度，那么我们可以调用map方法，代码如下：</p><pre><code>KStream&lt;String, Integer&gt; transformed = stream.map(
    (key, value) -&gt; KeyValue.pair(value.toLowerCase(), value.length()));
</code></pre><p>最后，我再介绍一组调试用的无状态算子：<strong>print和peek</strong>。Streams DSL支持你使用这两个方法查看你的消息流中的事件。这两者的区别在于，print是终止操作，一旦你调用了print方法，后面就不能再调用任何其他方法了，而peek则允许你在查看消息流的同时，依然能够继续对其进行处理，比如下面这两段代码所示：</p><pre><code>stream.print(Printed.toFile(&quot;streams.out&quot;).withLabel(&quot;debug&quot;));
</code></pre><pre><code>stream.peek((key, value) -&gt; System.out.println(&quot;key=&quot; + key + &quot;, value=&quot; + value)).map(...);
</code></pre><p>常见的有状态操作算子主要涉及聚合（Aggregation）方面的操作，比如计数、求和、求平均值、求最大最小值等。Streams DSL目前只提供了count方法用于计数，其他的聚合操作需要你自行使用API实现。</p><p>假设我们有个消息流，每条事件就是一个单独的整数，现在我们想要对其中的偶数进行求和，那么Streams DSL中的实现方法如下：</p><pre><code>final KTable&lt;Integer, Integer&gt; sumOfEvenNumbers = input
         .filter((k, v) -&gt; v % 2 == 0)
         .selectKey((k, v) -&gt; 1)
         .groupByKey()
         .reduce((v1, v2) -&gt; v1 + v2);
</code></pre><p>我简单解释一下selectKey调用。由于我们要对所有事件中的偶数进行求和，因此需要把这些消息的Key都调整成相同的值，因此这里我使用selectKey指定了一个Dummy Key值，即上面这段代码中的数值1。它没有任何含义，仅仅是让所有消息都赋值上这个Key而已。真正核心的代码在于reduce调用，它是执行求和的关键逻辑。</p><h3>时间窗口实例</h3><p>前面说过，Streams DSL支持3类时间窗口。前两类窗口通过<strong>TimeWindows.of方法</strong>来实现，会话窗口通过<strong>SessionWindows.with</strong>来实现。</p><p>假设在刚才的WordCount实例中，我们想每一分钟统计一次单词计数，那么需要在调用count之前增加下面这行代码：</p><pre><code>.windowedBy(TimeWindows.of(Duration.ofMinutes(1)))
</code></pre><p>同时，你还需要修改counts的类型，此时它不再是KTable&lt;String, Long&gt;了，而变成了KTable&lt;Windowed<string>, Long&gt;，因为引入了时间窗口，所以，事件的Key也必须要携带时间窗口的信息。除了这两点变化，WordCount其他部分代码都不需要修改。</string></p><p>可见，Streams DSL在API封装性方面还是做得很好的，通常你只需要增加或删减几行代码，就能实现处理逻辑的修改了。</p><h2>小结</h2><p>好了，我们来小结一下。今天我跟你分享了Kafka Streams以及DSL的背景与概念，然后我根据实例展示了WordCount单词计数程序以及运行方法，最后，针对常见的操作算子和时间窗口，我给出了示例代码，这些内容应该可以帮你应对大部分的流处理开发。另外，我建议你经常性地查询一下<a href="https://kafka.apache.org/23/documentation/streams/developer-guide/dsl-api.html#aggregating">官网文档</a>，去学习一些更深入更高级的用法，比如固定时间窗口的用法。在很多场景中，我们都想知道过去一段时间内企业某个关键指标的值是多少，如果要实现这个需求，时间窗口是必然要涉及到的。</p><p><img src="https://static001.geekbang.org/resource/image/dc/4d/dcc10168b7e3fdc43a47effdba45304d.jpg?wh=2069*2560" alt=""></p><h2>开放讨论</h2><p>今天给出的WordCount例子没有调用时间窗口API，我们统计的是每个单词的总数。如果现在我们想统计每5分钟内单词出现的次数，应该加一行什么代码呢？</p><p>欢迎写下你的思考和答案，我们一起讨论。如果你觉得有所收获，也欢迎把文章分享给你的朋友。</p>
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
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>leige</span>
  </div>
  <div class="_2_QraFYR_0">stream和table都会有对应的topic吧，老师？他们的本质区别是什么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 在topic层面，它们的区别反映在cleanup.policy上：stream是non-compact的topic；table是compact的topic</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-09 23:09:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/4d/49/28e73b9c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>明翼</span>
  </div>
  <div class="_2_QraFYR_0">单词计数里面的groupBy算子里面的key和value不太明白，不太明确那，为什么最后还要count计数</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: groupBy之后才能进行各种aggregation操作，比如count。groupBy要指定根据什么进行group，这就是key</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-09 08:02:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/23/ed/a4a774a8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>What for</span>
  </div>
  <div class="_2_QraFYR_0">有几个疑惑详情老师赐教：<br>1、demo 是一个可以单机跑的 Java 进程，运行时会有几个线程工作？<br>2、如果输入 topic 有 3 个分区，在计算过程中 consumer 会在 3 个不同的线程里分别起 1 个消费 1 个分区还是说有其他的配置项可以调整？<br>3、遇到 shuffle 时下游的计算计算分区还是就统一汇总计算？如果 shuffle 下游有分区怎么确定分区策略以及写入输出 topic 的时候会有几个 producer？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. 不知道你说的是哪个Java进程。比如对Broker进程而言，线程数可能会有很多，大约十几个。如果想了解都有哪些线程，不妨用JConsole去查看下<br>2. Java Consumer使用单线程获取所有分区数据，至于拿到消息之后是否分多个线程由用户决定。对于Kafka Streams而言，由参数num.stream.threads而定<br>3. 这个因你使用什么operator而定。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-04 22:40:44</div>
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
  <div class="_2_QraFYR_0">老师，我们领导提出了这样一些需求：1. 可以根据消息中的字段查询消息内容，这样可以用来诊断消息确实写入了kafka；2. 可以对某些消息重放；3. 可以对某些消息打标记从而控制消息的消费。我调研了一圈，第一点貌似可以用confluent的ksql做到，但是需要引入ksql server等复杂性，第二点也可以基于ksql来复制消息到重放队列（这样每一个topic都会存在一个重放topic）。第三点暂时还没有很好的思路。求助老师更好的方案，谢谢！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 第三点使用Interceptor试试呢？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-27 20:57:16</div>
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
  <div class="_2_QraFYR_0">老师，我有几个疑问：<br>1. 如果客户端应用重启了，KTable及写入的KStream在重启前的状态就都清楚了对吗？如果是的话，重启后，单词计数要重新对队列中的所有数据从头到尾再次计算，对吗？<br>2. 在没有指定时间窗口的情况下，应用读取队列消息的周期是什么呢？Stream API也是通过poll方式读取队列数据吗？<br>3. “所以，事件的 Key 也必须要携带时间窗口的信息。”，这里携带时间窗口信息是指什么呢？能举个例子吗？<br><br>谢谢！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. 不需要，它会记录消费位移的<br>2. 也是通过poll的，因为本质上就是consume-process-produce的流程<br>3. “事件的 Key 也必须要携带时间窗口的信息” 这句话是指因为引入了时间窗口，所以key中带了时间窗口的信息，比如窗口开始时间，结束时间等</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-12 11:46:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/25/7f/473d5a77.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>曾轼麟</span>
  </div>
  <div class="_2_QraFYR_0">老师有个疑问，如果按照这个事例，我使用kafka普通client的batch方式消费，搭载JAVA8的lambda不是实现更快捷吗？而且我中间还能自己通过代码写入各种数据库或者其它持久化方式？lambda本身也支持map—reduce的方式计算，而且consumer group本身也是一种负载均衡的思路</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果你自己写java代码肯定是可以的，不过这样你就要自行处理负载均衡、故障转移等问题了啊</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-10 09:21:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/4d/5e/c5c62933.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lmtoo</span>
  </div>
  <div class="_2_QraFYR_0">第一个例子没有时间窗口的情况下，统计的是什么？最终单词的计数，还是某个时间段的计数</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 截止到程序运行时</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-07 18:58:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/8d/56/2a04dd88.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>icejoywoo</span>
  </div>
  <div class="_2_QraFYR_0">count之前加上.windowedBy(TimeWindows.of(Duration.ofMinutes(5)))，应该就可以了吧</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-07 11:43:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/c0/8b/0371baee.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张丽娜</span>
  </div>
  <div class="_2_QraFYR_0">这个章节，老师讲的东西，我竟然听懂了，感谢老师耐心的讲解啊。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢您的鼓励：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-05 15:38:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/58/28/c86340ca.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>达文西</span>
  </div>
  <div class="_2_QraFYR_0">感觉打开了新世界的大门,虽然暂时在业务上用不上</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-27 10:18:15</div>
  </div>
</div>
</div>
</li>
</ul>