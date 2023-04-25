<audio title="42 _ Kafka Streams在金融领域的应用" src="https://static001.geekbang.org/resource/audio/78/ca/78cfd5d0d28acda9d69fb6fe154ae0ca.mp3" controls="controls"></audio> 
<p>你好，我是胡夕。今天我要和你分享的主题是：Kafka Streams在金融领域的应用。</p><h2>背景</h2><p>金融领域囊括的内容有很多，我今天分享的主要是，如何利用大数据技术，特别是Kafka Streams实时计算框架，来帮助我们更好地做企业用户洞察。</p><p>众所周知，金融领域内的获客成本是相当高的，一线城市高净值白领的获客成本通常可达上千元。面对如此巨大的成本压力，金融企业一方面要降低广告投放的获客成本，另一方面要做好精细化运营，实现客户生命周期内价值（Custom Lifecycle Value, CLV）的最大化。</p><p><strong>实现价值最大化的一个重要途径就是做好用户洞察，而用户洞察要求你要更深度地了解你的客户</strong>，即所谓的Know Your Customer（KYC），真正做到以客户为中心，不断地满足客户需求。</p><p>为了实现KYC，传统的做法是花费大量的时间与客户见面，做面对面的沟通以了解客户的情况。但是，用这种方式得到的数据往往是不真实的，毕竟客户内心是有潜在的自我保护意识的，短时间内的面对面交流很难真正洞察到客户的真实诉求。</p><p>相反地，渗透到每个人日常生活方方面面的大数据信息则代表了客户的实际需求。比如客户经常浏览哪些网站、都买过什么东西、最喜欢的视频类型是什么。这些数据看似很随意，但都表征了客户最真实的想法。将这些数据汇总在一起，我们就能完整地构造出客户的画像，这就是所谓的用户画像（User Profile）技术。</p><!-- [[[read_end]]] --><h2>用户画像</h2><p>用户画像听起来很玄妙，但实际上你应该是很熟悉的。你的很多基本信息，比如性别、年龄、所属行业、工资收入和爱好等，都是用户画像的一部分。举个例子，我们可以这样描述一个人：某某某，男性，28岁，未婚，工资水平大致在15000到20000元之间，是一名大数据开发工程师，居住在北京天通苑小区，平时加班很多，喜欢动漫或游戏。</p><p>其实，这一连串的描述就是典型的用户画像。通俗点来说，构建用户画像的核心工作就是给客户或用户打标签（Tagging）。刚刚那一连串的描述就是用户系统中的典型标签。用户画像系统通过打标签的形式，把客户提供给业务人员，从而实现精准营销。</p><h2>ID映射（ID Mapping）</h2><p>用户画像的好处不言而喻，而且标签打得越多越丰富，就越能精确地表征一个人的方方面面。不过，在打一个个具体的标签之前，弄清楚“你是谁”是所有用户画像系统首要考虑的问题，这个问题也被称为ID识别问题。</p><p>所谓的ID即Identification，表示用户身份。在网络上，能够标识用户身份信息的常见ID有5种。</p><ul>
<li>身份证号：这是最能表征身份的ID信息，每个身份证号只会对应一个人。</li>
<li>手机号：手机号通常能较好地表征身份。虽然会出现同一个人有多个手机号或一个手机号在不同时期被多个人使用的情形，但大部分互联网应用使用手机号表征用户身份的做法是很流行的。</li>
<li>设备ID：在移动互联网时代，这主要是指手机的设备ID或Mac、iPad等移动终端设备的设备ID。特别是手机的设备ID，在很多场景下具备定位和识别用户的功能。常见的设备ID有iOS端的IDFA和Android端的IMEI。</li>
<li>应用注册账号：这属于比较弱的一类ID。每个人在不同的应用上可能会注册不同的账号，但依然有很多人使用通用的注册账号名称，因此具有一定的关联性和识别性。</li>
<li>Cookie：在PC时代，浏览器端的Cookie信息是很重要的数据，它是网络上表征用户信息的重要手段之一。只不过随着移动互联网时代的来临，Cookie早已江河日下，如今作为ID数据的价值也越来越小了。我个人甚至认为，在构建基于移动互联网的新一代用户画像时，Cookie可能要被抛弃了。</li>
</ul><p>在构建用户画像系统时，我们会从多个数据源上源源不断地收集各种个人用户数据。通常情况下，这些数据不会全部携带以上这些ID信息。比如在读取浏览器的浏览历史时，你获取的是Cookie数据，而读取用户在某个App上的访问行为数据时，你拿到的是用户的设备ID和注册账号信息。</p><p>倘若这些数据表征的都是一个用户的信息，我们的用户画像系统如何识别出来呢？换句话说，你需要一种手段或技术帮你做各个ID的打通或映射。这就是用户画像领域的ID映射问题。</p><h2>实时ID Mapping</h2><p>我举个简单的例子。假设有一个金融理财用户张三，他首先在苹果手机上访问了某理财产品，然后在安卓手机上注册了该理财产品的账号，最后在电脑上登录该账号，并购买了该理财产品。ID Mapping 就是要将这些不同端或设备上的用户信息聚合起来，然后找出并打通用户所关联的所有ID信息。</p><p>实时ID Mapping的要求就更高了，它要求我们能够实时地分析从各个设备收集来的数据，并在很短的时间内完成ID Mapping。打通用户ID身份的时间越短，我们就能越快地为其打上更多的标签，从而让用户画像发挥更大的价值。</p><p>从实时计算或流处理的角度来看，实时ID Mapping能够转换成一个<strong>流-表连接问题</strong>（Stream-Table Join），即我们实时地将一个流和一个表进行连接。</p><p>消息流中的每个事件或每条消息包含的是一个未知用户的某种信息，它可以是用户在页面的访问记录数据，也可以是用户的购买行为数据。这些消息中可能会包含我们刚才提到的若干种ID信息，比如页面访问信息中可能包含设备ID，也可能包含注册账号，而购买行为信息中可能包含身份证信息和手机号等。</p><p>连接的另一方表保存的是<strong>用户所有的ID信息</strong>，随着连接的不断深入，表中保存的ID品类会越来越丰富，也就是说，流中的数据会被不断地补充进表中，最终实现对用户所有ID的打通。</p><h2>Kafka Streams实现</h2><p>好了，现在我们就来看看如何使用Kafka Streams来实现一个特定场景下的实时ID Mapping。为了方便理解，我们假设ID Mapping只关心身份证号、手机号以及设备ID。下面是用Avro写成的Schema格式：</p><pre><code>{
  &quot;namespace&quot;: &quot;kafkalearn.userprofile.idmapping&quot;,
  &quot;type&quot;: &quot;record&quot;,
  &quot;name&quot;: &quot;IDMapping&quot;,
  &quot;fields&quot;: [
    {&quot;name&quot;: &quot;deviceId&quot;, &quot;type&quot;: &quot;string&quot;},
    {&quot;name&quot;: &quot;idCard&quot;, &quot;type&quot;: &quot;string&quot;},
    {&quot;name&quot;: &quot;phone&quot;, &quot;type&quot;: &quot;string&quot;}
  ]
}
</code></pre><p>顺便说一下，<strong>Avro是Java或大数据生态圈常用的序列化编码机制</strong>，比如直接使用JSON或XML保存对象。Avro能极大地节省磁盘占用空间或网络I/O传输量，因此普遍应用于大数据量下的数据传输。</p><p>在这个场景下，我们需要两个Kafka主题，一个用于构造表，另一个用于构建流。这两个主题的消息格式都是上面的IDMapping对象。</p><p>新用户在填写手机号注册App时，会向第一个主题发送一条消息，该用户后续在App上的所有访问记录，也都会以消息的形式发送到第二个主题。值得注意的是，发送到第二个主题上的消息有可能携带其他的ID信息，比如手机号或设备ID等。就像我刚刚所说的，这是一个典型的流-表实时连接场景，连接之后，我们就能够将用户的所有数据补齐，实现ID Mapping的打通。</p><p>基于这个设计思路，我先给出完整的Kafka Streams代码，稍后我会对重点部分进行详细解释：</p><pre><code>package kafkalearn.userprofile.idmapping;

// omit imports……

public class IDMappingStreams {


    public static void main(String[] args) throws Exception {

        if (args.length &lt; 1) {
            throw new IllegalArgumentException(&quot;Must specify the path for a configuration file.&quot;);
        }

        IDMappingStreams instance = new IDMappingStreams();
        Properties envProps = instance.loadProperties(args[0]);
        Properties streamProps = instance.buildStreamsProperties(envProps);
        Topology topology = instance.buildTopology(envProps);

        instance.createTopics(envProps);

        final KafkaStreams streams = new KafkaStreams(topology, streamProps);
        final CountDownLatch latch = new CountDownLatch(1);

        // Attach shutdown handler to catch Control-C.
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
        } catch (Throwable e) {
            System.exit(1);
        }
        System.exit(0);
    }

    private Properties loadProperties(String propertyFilePath) throws IOException {
        Properties envProps = new Properties();
        try (FileInputStream input = new FileInputStream(propertyFilePath)) {
            envProps.load(input);
            return envProps;
        }
    }

    private Properties buildStreamsProperties(Properties envProps) {
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, envProps.getProperty(&quot;application.id&quot;));
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, envProps.getProperty(&quot;bootstrap.servers&quot;));
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        return props;
    }

    private void createTopics(Properties envProps) {
        Map&lt;String, Object&gt; config = new HashMap&lt;&gt;();
        config.put(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, envProps.getProperty(&quot;bootstrap.servers&quot;));
        try (AdminClient client = AdminClient.create(config)) {
            List&lt;NewTopic&gt; topics = new ArrayList&lt;&gt;();
            topics.add(new NewTopic(
                    envProps.getProperty(&quot;stream.topic.name&quot;),
                    Integer.parseInt(envProps.getProperty(&quot;stream.topic.partitions&quot;)),
                    Short.parseShort(envProps.getProperty(&quot;stream.topic.replication.factor&quot;))));

            topics.add(new NewTopic(
                    envProps.getProperty(&quot;table.topic.name&quot;),
                    Integer.parseInt(envProps.getProperty(&quot;table.topic.partitions&quot;)),
                    Short.parseShort(envProps.getProperty(&quot;table.topic.replication.factor&quot;))));

            client.createTopics(topics);
        }
    }

    private Topology buildTopology(Properties envProps) {
        final StreamsBuilder builder = new StreamsBuilder();
        final String streamTopic = envProps.getProperty(&quot;stream.topic.name&quot;);
        final String rekeyedTopic = envProps.getProperty(&quot;rekeyed.topic.name&quot;);
        final String tableTopic = envProps.getProperty(&quot;table.topic.name&quot;);
        final String outputTopic = envProps.getProperty(&quot;output.topic.name&quot;);
        final Gson gson = new Gson();

        // 1. 构造表
        KStream&lt;String, IDMapping&gt; rekeyed = builder.&lt;String, String&gt;stream(tableTopic)
                .mapValues(json -&gt; gson.fromJson(json, IDMapping.class))
                .filter((noKey, idMapping) -&gt; !Objects.isNull(idMapping.getPhone()))
                .map((noKey, idMapping) -&gt; new KeyValue&lt;&gt;(idMapping.getPhone(), idMapping));
        rekeyed.to(rekeyedTopic);
        KTable&lt;String, IDMapping&gt; table = builder.table(rekeyedTopic);

        // 2. 流-表连接
        KStream&lt;String, String&gt; joinedStream = builder.&lt;String, String&gt;stream(streamTopic)
                .mapValues(json -&gt; gson.fromJson(json, IDMapping.class))
                .map((noKey, idMapping) -&gt; new KeyValue&lt;&gt;(idMapping.getPhone(), idMapping))
                .leftJoin(table, (value1, value2) -&gt; IDMapping.newBuilder()
                        .setPhone(value2.getPhone() == null ? value1.getPhone() : value2.getPhone())
                        .setDeviceId(value2.getDeviceId() == null ? value1.getDeviceId() : value2.getDeviceId())
                        .setIdCard(value2.getIdCard() == null ? value1.getIdCard() : value2.getIdCard())
                        .build())
                .mapValues(v -&gt; gson.toJson(v));

        joinedStream.to(outputTopic);

        return builder.build();
    }
}

</code></pre><p>这个Java类代码中最重要的方法是<strong>buildTopology函数</strong>，它构造了我们打通ID Mapping的所有逻辑。</p><p>在该方法中，我们首先构造了StreamsBuilder对象实例，这是构造任何Kafka Streams应用的第一步。之后我们读取配置文件，获取了要读写的所有Kafka主题名。在这个例子中，我们需要用到4个主题，它们的作用如下：</p><ul>
<li>streamTopic：保存用户登录App后发生的各种行为数据，格式是IDMapping对象的JSON串。你可能会问，前面不是都创建Avro Schema文件了吗，怎么这里又用回JSON了呢？原因是这样的：社区版的Kafka没有提供Avro的序列化/反序列化类支持，如果我要使用Avro，必须改用Confluent公司提供的Kafka，但这会偏离我们专栏想要介绍Apache Kafka的初衷。所以，我还是使用JSON进行说明。这里我只是用了Avro Code Generator帮我们提供IDMapping对象各个字段的set/get方法，你使用Lombok也是可以的。</li>
<li>rekeyedTopic：这个主题是一个中间主题，它将streamTopic中的手机号提取出来作为消息的Key，同时维持消息体不变。</li>
<li>tableTopic：保存用户注册App时填写的手机号。我们要使用这个主题构造连接时要用到的表数据。</li>
<li>outputTopic：保存连接后的输出信息，即打通了用户所有ID数据的IDMapping对象，将其转换成JSON后输出。</li>
</ul><p>buildTopology的第一步是构造表，即KTable对象。我们修改初始的消息流，以用户注册的手机号作为Key，构造了一个中间流，之后将这个流写入到rekeyedTopic，最后直接使用builder.table方法构造出KTable。这样每当有新用户注册时，该KTable都会新增一条数据。</p><p>有了表之后，我们继续构造消息流来封装用户登录App之后的行为数据，我们同样提取出手机号作为要连接的Key，之后使用KStream的<strong>leftJoin方法</strong>将其与上一步的KTable对象进行关联。</p><p>在关联的过程中，我们同时提取两边的信息，尽可能地补充到最后生成的IDMapping对象中，然后将这个生成的IDMapping实例返回到新生成的流中。最后，我们将它写入到outputTopic中保存。</p><p>至此，我们使用了不到200行的Java代码，就简单实现了一个真实场景下的实时ID Mapping任务。理论上，你可以将这个例子继续扩充，扩展到任意多个ID Mapping，甚至是含有其他标签的数据，连接原理是相通的。在我自己的项目中，我借助于Kafka Streams帮助我实现了用户画像系统的部分功能，而ID Mapping就是其中的一个。</p><h2>小结</h2><p>好了，我们小结一下。今天，我展示了Kafka Streams在金融领域的一个应用案例，重点演示了如何利用连接函数来实时关联流和表。其实，Kafka Streams提供的功能远不止这些，我推荐你阅读一下<a href="https://kafka.apache.org/23/documentation/streams/developer-guide/">官网</a>的教程，然后把自己的一些轻量级的实时计算线上任务改为使用Kafka Streams来实现。</p><p><img src="https://static001.geekbang.org/resource/image/75/e7/75df06c2b75c3886ca3496a774730de7.jpg?wh=2069*2560" alt=""></p><h2>开放讨论</h2><p>最后，我们来讨论一个问题。在刚刚的这个例子中，你觉得我为什么使用leftJoin方法而不是join方法呢？（小提示：可以对比一下SQL中的left join和inner join。）</p><p>欢迎写下你的思考和答案，我们一起讨论。如果你觉得有所收获，也欢迎把文章分享给你的朋友。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/bd/18/2af6bf4b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>兔2🐰🍃</span>
  </div>
  <div class="_2_QraFYR_0">App上发现该栏目更新了最后一节，目前才学到22节，完成了前三章的了解。我从上个月20日开始的，有20天时间了，也算是从0开始的，对Kafka有了很多了解，每听完一节，遍跟着写笔记，结构化文章内容，感觉有点慢，实践还没开始。明天起换个方式试试效果，搭个环境，实践下前22节内容，接着后面的章节先每章节内容听一遍后梳理整个章节的内容，再到实践。<br>感谢胡老师的教课，节日快乐~</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-10 21:14:25</div>
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
  <div class="_2_QraFYR_0">left join 可以关联上不存在的数据行，而join其实是inner join 需要两张表数据都匹配上结果才会有，left join的好处就是，如果其中有一张表没有匹配数据，也不会导致结果集没有这条记录</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-11 08:57:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/6a/fe/446c0282.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Minze</span>
  </div>
  <div class="_2_QraFYR_0">胡老师您好，最近我在学习使用Kafka streams，有一点让我困惑的是在使用join的时候，生成的topology中会有一些名字包含KSTREAM–JOINTHIS和KSTREAM–OUTEROTHER的processor和store创建出来，这些本地的store应该是用来存储join要用到的数据，这我能理解，但是这些processor的作用和JOINTHIS、OUTEROTHER的命名含义却百思不得其解，老师可否给些建议或者提示？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 没有什么特别的含义，源码就是这么命名的，如下：<br>static final String OUTEROTHER_NAME = &quot;KSTREAM-OUTEROTHER-&quot;;</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-29 19:42:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/ef/0c/e2169f41.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>King</span>
  </div>
  <div class="_2_QraFYR_0">最后写入outputTopic是以append还是update？可以保证用户在outputTopic中只有一条记录吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: append的方式。不能保证，因为最后是将所有状态转换成stream了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-21 15:47:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/d2/6a/a9039139.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>IT小僧</span>
  </div>
  <div class="_2_QraFYR_0">老师好，这个构造IDMapping中的KTable对象肯定会越来越大吧，这个怎么处理呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 也没关系啊，保存在底层的topic中</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-06 12:52:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2a/b9/40/e350862c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Kaine</span>
  </div>
  <div class="_2_QraFYR_0">老师在&lt;VT, VR&gt; KStream&lt;K, VR&gt; leftJoin(final KTable&lt;K, VT&gt; table,<br>                                     final ValueJoiner&lt;? super V, ? super VT, ? extends VR&gt; joiner);<br>有这么一段注释，上面说输入流的分区数必须与Ktable对应的topic的分区数一致这是为什么呢<br>Both input streams (or to be more precise, their underlying source topics) need to have the same number of partitions. If this is not the case, you would need to call through(String) for this KStream before doing the join, using a pre-created topic with the same number of partitions as the given KTable. Furthermore, both input streams need to be co-partitioned on the join key (i.e., use the same partitioner); cf. join(GlobalKTable, KeyValueMapper, ValueJoiner). If this requirement is not met, Kafka Streams will automatically repartition the data, i.e., it will create an internal repartitioning topic in Kafka and write and re-read the data via this topic before the actual join. The repartitioning topic will be named &quot;${applicationId}-&lt;name&gt;-repartition&quot;, where &quot;applicationId&quot; is user-specified in StreamsConfig via parameter APPLICATION_ID_CONFIG, &quot;&lt;name&gt;&quot; is an internally generated name, and &quot;-repartition&quot; is a fixed suffix.</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-23 17:47:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83eq6pWvKsV4rzQ62z5MDEjaEU5MbDfmzbA62kUgoqia2tgKIIxw4ibkDhF7W48iat5dT8UB9Adky2NuzQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小仙</span>
  </div>
  <div class="_2_QraFYR_0">这里用到的 4 个 topic 就是普通的 topic 吗？也是默认 7 天消息过期等等设定，并非永久保存</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-18 14:27:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/19/df/5486f00d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>朝花西落</span>
  </div>
  <div class="_2_QraFYR_0">代码的99行是不是没有必要呢，我感觉既然是按手机号key来做join的，那不管value2是否为null，最终都应该是value1的值。因为如果不为null，value1和value2也是相等的吧。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-12 09:53:10</div>
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
  <div class="_2_QraFYR_0">KTable是全部保存在内存吗？具体实现是怎么样的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 保存在底层的Topic中，这类topic一定是启用了compact了的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-03 21:44:33</div>
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
  <div class="_2_QraFYR_0">老师，我看到有使用stream加迭代器的方式去消费消息的，貌似建立连接时传入了一个topicCount的map，也就是说这个stream其实可以消费到多个topic的数据，总觉得有点奇怪，因为跟老师之前课程里那种poll的方式差别很大。我不太能明白这种形式消费数据，到底消费者对应是某一个stream？还是topicCount的每个topic？是哪个决定了consumer的数量？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果是Kafka Streams，consumer的数量由num.stream.threads直接决定，当然也受订阅总分区数的制约</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-21 09:13:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/bb/c2/cbd652ab.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>布吉岛上的不科学君</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，我有一个疑惑：<br>首先，假设用户注册的时候只有【手机号】信息，那么KTable里是没有【DeviceId】和【IdCard】的。<br>然后用户发生的第一次操作，携带了【DeviceId】，第二次操作携带了【IdCard】，这些数据从代码上看是不会更新【KTable】的，那么【outputTopic】两次输出的内容都是不全面，也就是说【outputTopic】没法同时输出携带了【DeviceId和IdCard】信息的吧？希望胡老师能够抽空解答一下，谢谢。<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-09 23:53:15</div>
  </div>
</div>
</div>
</li>
</ul>