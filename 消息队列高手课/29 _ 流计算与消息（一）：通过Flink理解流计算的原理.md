<audio title="29 _ 流计算与消息（一）：通过Flink理解流计算的原理" src="https://static001.geekbang.org/resource/audio/c4/41/c48b2a303aebccc614870b56c026ce41.mp3" controls="controls"></audio> 
<p>你好，我是李玥。</p><p>在上节课中，我简单地介绍了消息队列和流计算的相关性。在生产中，消息队列和流计算往往是相互配合，一起来使用的。而流计算也是后端程序员技术栈中非常重要的一项技术。在接下来的两节课中，我们一起通过两个例子来实际演练一下，如何使用消息队列配合流计算框架实现一些常用的流计算任务。</p><p>这节课，我们一起来基于Flink实现一个流计算任务，通过这个例子来感受一下流计算的好处，同时我还会给你讲解流计算框架的实现原理。下一节课中，我们会把本节课中的例子升级改造，使用Kafka配合Flink来实现Exactly Once语义，确保数据在计算过程中不重不丢。</p><p>无论你之前是否接触过像Storm、Flink或是Spark这些流计算框架都没有关系，因为我们已经学习了消息队列的实现原理，以及实现消息队列必备的像异步网络传输、序列化这些知识。在掌握了这些知识和底层的原理之后，再来学习和理解流计算框架的实现原理，你会发现，事情就变得非常简单了。</p><p>为什么这么说，一个原因是，对于很多中间件或者说基础框架这类软件来说，它们用到很多底层的技术都是一样；另外一个原因是，流计算和消息队列处理的都实时的、流动的数据，很多处理流数据的方法也是一样的。</p><!-- [[[read_end]]] --><h2>哪些问题适合用流计算解决？</h2><p>首先，我们来说一下，哪些问题适合用流计算来解决？或者说，流计算它的应用场景是什么样的呢？</p><p>在这里，我用一句话来回答这个问题：<strong>对实时产生的数据进行实时统计分析，这类场景都适合使用流计算来实现。 </strong></p><p>你在理解这句话的时候，需要特别注意的是，这里面有两个“实时”，一个是说，数据是“实时”产生的，另一个是说，统计分析这个过程是“实时”进行的，统计结果也是第一时间就计算出来了。对于这样的场景，你都可以考虑使用流计算框架。</p><p>因为流计算框架可以自动地帮我们实现实时的并行计算，性能非常好，并且内置了很多常用的统计分析的算子，比如TimeWindow、GroupBy、Sum和Count，所以非常适合用来做实时的统计和分析。举几个例子：</p><ul>
<li>每分钟按照IP统计Web请求次数；</li>
<li>电商在大促时，实时统计当前下单量；</li>
<li>实时统计App中的埋点数据，分析营销推广效果。</li>
</ul><p>以上这些场景，以及和这些场景类似的场景，都是比较适合用流计算框架来实现的。特别是基于时间维度的统计分析，使用流计算框架来实现是非常方便的。</p><h2>用代码定义Job并在Flink中执行</h2><p>接下来，我们用Flink来实现一个实时统计任务：接收NGINX的access.log，每5秒钟按照IP地址统计Web请求的次数。这个统计任务它一个非常典型的，按照Key来进行分类汇总的统计任务，并且汇总是按照一定周期来实时进行的，我们日常工作中遇到的很多统计分析类的需求，都可以套用这个例子的模式来实现，所以我们就以它为例来做一个实现。</p><p>假设我们已经有一个实时发送access.log的日志服务，它运行在本地的9999端口上，只要有客户端连接上来，他就会通过Socket给客户端发送实时的访问日志，日志的内容只包含访问时间和IP地址，每条数据的结尾用一个换行符(\n)作为分隔符。这个日志服务就是我们流计算任务的数据源。</p><p>我们用NetCat连接到这个服务上，看一下数据格式：</p><pre><code>$nc localhost 9999
14:37:11 192.168.1.3
14:37:11 192.168.1.2
14:37:12 192.168.1.4
14:37:14 192.168.1.2
14:37:14 192.168.1.4
14:37:14 192.168.1.3
...
</code></pre><p>接下来我们用Scala语言和Flink来实现这个流计算任务。你可以先不用关心如何部署启动Flink，如何设置开发环境这些问题，一起来跟我看一下定义这个流计算任务的代码：</p><pre><code>object SocketWindowIpCount {

  def main(args: Array[String]) : Unit = {

    // 获取运行时环境
    val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment
    // 按照EventTime来统计
    env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)
    // 设置并行度
    env.setParallelism(4)

    // 定义输入：从Socket端口中获取数据输入
    val hostname: String = &quot;localhost&quot;
    val port: Int = 9999
    // Task 1
    val input: DataStream[String] = env.socketTextStream(hostname, port, '\n')

    // 数据转换：将非结构化的以空格分隔的文本转成结构化数据IpAndCount
    // Task 2
    input
      .map { line =&gt; line.split(&quot;\\s&quot;) }
      .map { wordArray =&gt; IpAndCount(new SimpleDateFormat(&quot;HH:mm:ss&quot;).parse(wordArray(0)), wordArray(1), 1) }

    // 计算：每5秒钟按照ip对count求和

      .assignAscendingTimestamps(_.date.getTime) // 告诉Flink时间从哪个字段中获取


      .keyBy(&quot;ip&quot;) // 按照ip地址统计
      // Task 3
      .window(TumblingEventTimeWindows.of(Time.seconds(5))) // 每5秒钟统计一次
      .sum(&quot;count&quot;) // 对count字段求和

    // 输出：转换格式，打印到控制台上

      .map { aggData =&gt; new SimpleDateFormat(&quot;HH:mm:ss&quot;).format(aggData.date) + &quot; &quot; + aggData.ip + &quot; &quot; + aggData.count }
      .print()

    env.execute(&quot;Socket Window IpCount&quot;)
  }

  /** 中间数据结构 */

  case class IpAndCount(date: Date, ip: String, count: Long)
}
</code></pre><p>我来给你解读一下这段代码。</p><p>首先需要获取流计算的运行时环境，也就是这个env对象，对env做一些初始化的设置。然后，我们再定义输入的数据源，这里面就是我刚刚讲的，运行在9999端口上的日志服务。</p><p>在代码中，env.socketTextStream(hostname, port, ‘\n’)这个语句中的三个参数分别是主机名、端口号和分隔符，返回值的数据类型是DataStream[String]，代表一个数据流，其中的每条数据都是String类型的。它告诉Flink，我们的数据源是一个Socket服务。这样，Flink在执行这个计算任务的时候，就会去连接日志服务来接收数据。</p><p>定义完数据源之后，需要做一些数据转换，把字符串转成结构化的数据IpAndCount，便于后续做计算。在定义计算的部分，依次告诉Flink：时间从date字段中获取，按照IP地址进行汇总，每5秒钟汇总一次，汇总方式就是对count字段求和。</p><p>之后定义计算结果如何输出，在这个例子中，我们直接把结果打印到控制台上就好了。</p><p>这样就完成了一个流计算任务的定义。可以看到，定义一个计算任务的代码还是非常简单的，如果我们要自己写一个分布式的统计程序来实现一样的功能，代码量和复杂度肯定要远远超过上面这段代码。</p><p>总结下来，无论是使用Flink、Spark还是其他的流计算框架，定义一个流计算的任务基本上都可以分为：定义输入、定义计算逻辑和定义输出三部分，通俗地说，也就是：<strong>数据从哪儿来，怎么计算，结果写到哪儿去</strong>，这三件事儿。</p><p>我把这个例子的代码上传到了GitHub上，你可以在<a href="https://github.com/liyue2008/IpCount">这里</a>下载，关于如何设置环境、编译并运行这个例子，我在代码中的README中都给出了说明，你可以下载查看。</p><p>执行计算任务打印出的计算结果是这样的：</p><pre><code>1&gt; 18:40:10 192.168.1.2 23
4&gt; 18:40:10 192.168.1.4 16
4&gt; 18:40:15 192.168.1.4 27
3&gt; 18:40:15 192.168.1.3 23
1&gt; 18:40:15 192.168.1.2 25
4&gt; 18:40:15 192.168.1.1 21
1&gt; 18:40:20 192.168.1.2 21
3&gt; 18:40:20 192.168.1.3 31
4&gt; 18:40:20 192.168.1.1 25
4&gt; 18:40:20 192.168.1.4 26
</code></pre><p>对于流计算的初学者，特别不好理解的一点是，我们上面编写的这段代码，<strong>它只是“用来定义计算任务的代码”，而不是“真正处理数据的代码”。</strong>对于普通的应用程序，源代码编译之后，计算机就直接执行了，这个比较好理解。而在Flink中，当这个计算任务在Flink集群的计算节点中运行的时候，真正处理数据的代码并不是我们上面写的那段代码，而是Flink在解析了计算任务之后，动态生成的代码。</p><p>这个有点儿类似于我们在查询MySQL的时候执行的SQL，我们提交一个SQL查询后，MySQL在执行查询遍历数据库中每条数据时，并不是对每条数据执行一遍SQL，真正执行的其实是MySQL自己的代码。SQL只是告诉MySQL我们要如何来查询数据，同样，我们编写的这段定义计算任务的代码，只是告诉Flink我们要如何来处理数据而已。</p><h2>Job是如何在Flink集群中执行的？</h2><p>那我们的计算任务是如何在Flink中执行的呢？在讲解这个问题之前，我们先简单看一下Flink集群在运行时的架构。</p><p>下面这张图来自于<a href="https://github.com/liyue2008/IpCount">Flink的官方文档</a>。</p><p><img src="https://static001.geekbang.org/resource/image/91/78/91fd3d493f1b7e3e18224d9a8ba33678.png?wh=1824*1306" alt=""></p><p>这张图稍微有点儿复杂，我们先忽略细节看整体。Flink的集群和其他分布式系统都是类似的，集群的大部分节点都是TaskManager节点，每个节点就是一个Java进程，负责执行计算任务。另外一种节点是JobManager节点，它负责管理和协调所有的计算节点和计算任务，同时，客户端和Web控制台也是通过JobManager来提交和管理每个计算任务的。</p><p>我们编写好计算任务的代码后，打包成JAR文件，然后通过Flink的客户端提交到JobManager上。计算任务被Flink解析后，会生成一个Dataflow Graph，也叫JobGraph，简称DAG，这是一个有向无环图（DAG），比如我们的这个例子，它生成的DAG是这样的：</p><p><img src="https://static001.geekbang.org/resource/image/4f/f6/4f8358c8f2ca8b2ed57dfa8bc7aa4cf6.png?wh=2276*690" alt=""></p><p>图中的每个节点是一个Task，每个Task就是一个执行单元，运行在某一个TaskManager的进程内。你可以想象一下，就像电流流过电路图一样，数据从Source Task流入，进入这个DAG，每流过一个Task，就被这个Task做一些计算和变换，然后数据继续流入下一个Task，直到最后一个Sink Task流出DAG，就自然完成了计算。</p><p>对于图中的3个Task，每个Task对应执行了什么计算，完全可以和我们上面定义计算任务的源代码对应上，我也在源代码的注释中，用"//Task n"的形式给出了标注。第一个Task执行的计算很简单，就是连接日志服务接收日志数据，然后将日志数据发往下一个Task。第二个Task执行了两个map变换，把文本数据转换成了结构化的数据，并添加Watermark（水印）。Watermark这个概念可以先不用管，主要是用于触发按时间汇总的操作。第三个Task执行了剩余的计算任务，按时间汇总日志，并输出打印到控制台上。</p><p>这个DAG仍然是一个逻辑图，它到底是怎么在Flink集群中执行的呢？你注意到图中每个Task都标注了一个Parallelism（并行度）的数字吗？这个并行度的意思就是，这个Task可以被多少个线程并行执行。比如图中的第二个任务，它的并行度是4，就代表Task在Flink集群中运行的时候，会有4个线程都在执行这个Task，每个线程就是一个SubTask（子任务）。注意，如果Flink集群的节点数够多，这4个SubTask可能会运行在不同的TaskManager节点上。</p><p>建立了SubTask的概念之后，我们再重新回过头来看一下这个图中的两个箭头。第一个箭头连接前两个Task，这个箭头标注了REBALANCE（重新分配），因为第一个Task并行度是1，而第二个Task并行度是4，意味着从第一个Task流出的数据将被重新分配给第二个Task的4个线程，也就是4个SubTask（子任务）中，这样就实现了并行处理。这和消息队列中每个主题分成多个分区进行并行收发的设计思想是一样的。</p><p>再来看连接第二、第三这两个Task的箭头，这个箭头上标注的是HASH，为什么呢？可以看到，第二个Task中最后一步业务逻辑是：keyBy(“ip”)，也就是按照IP这个字段做一个HASH分流。你可以想一下，第三个Task，它的并行度是4，也就是有4个线程在并行执行汇总。如果要统计每个IP的日志条数，那必须得把相同IP的数据发送到同一个SubTask（子任务）中去，这样在每个SubTask（子任务）中，对于每一条数据，只要在对应IP汇总记录上进行累加就可以了。</p><p>反之，要是相同IP的数据被分到多个SubTask（子任务）上，这些SubTask又可能分布在多个物理节点上，那就没办法统计了。所以，第二个Task会把数据按照IP地址做一个HASH分流，保证IP相同的数据都发送到第三个Task中相同的SubTask（子任务）中。这个HASH分流的设计是不是感觉很眼熟？我们之前课程中讲到的，严格顺序消息的实现方法：通过HASH算法，让key相同的数据总是发送到相同的分区上来保证严格顺序，和Flink这里的设计就是一样的。</p><p>最后在第三个Task中，4个SubTask并行进行数据汇总，每个SubTask负责汇总一部分IP地址的数据。最终打印到控制台上的时候，也是4个线程并行打印。你可以回过头去看一下输出的计算结果，每一行数据前面的数字，就是第三个Task中SubTask的编号。</p><p>到这里，我们不仅实现并运行了一个流计算任务，也理清了任务在Flink集群中运行的过程。</p><h2>小结</h2><p>流计算框架适合对实时产生的数据进行实时统计分析。我们通过一个“按照IP地址统计Web请求的次数”的例子，学习了Flink实现流计算任务的原理。首先，我们用一段代码定义了计算任务，把计算任务代码编译成JAR包后，通过Flink客户端提交到JobManager上。</p><p>这里需要注意的是，我们编写的代码只是用来定义计算任务，和在Flink节点上执行的真正做实时计算的代码是不一样的。真正执行计算的代码是Flink在解析计算任务后，动态生成的。</p><p>Flink分析计算任务之后生成JobGraph，JobGraph是一个有向无环图，数据流过这个图中的节点，在每个节点进行计算和变换，最终流出有向无环图就完成了计算。JobGraph中的每个节点是一个Task，Task是可以并行执行的，每个线程就是一个SubTask。SubTask被JobManager分配给某个TaskManager，在TaskManager进程中的一个线程中执行。</p><p>通过分析Flink的实现原理，我们可以看到，流计算框架本身并没有什么神奇的技术，之所以能够做到非常好的性能，主要有两个原因。一个是，它能自动拆分计算任务来实现并行计算，这个和Hadoop中Map Reduce的原理是一样的。另外一个原因是，流计算框架中，都内置了很多常用的计算和统计分析的算子，这些算子的实现都是经过很多大神级程序员反复优化过的，不仅能方便我们开发，性能上也比大多数程序员自行实现要快很多。</p><h2>思考题</h2><p>我们在启动Flink集群之前，修改了Flink的一个配置：槽数taskmanager.numberOfTaskSlots。请你课后看一下Flink的文档，搞清楚这个槽数的含义。然后再想一下，这个槽数和我们在计算任务中定义的并行度又是什么关系呢？</p><p>欢迎在留言区写下你的思考，如果有任何问题，也欢迎与我交流。</p><p>感谢阅读，如果你觉得这篇文章对你有一些启发，也欢迎把它分享给你的朋友。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/8b/ec/dc03f5ad.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张天屹</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，请教个概念，虽然处理统计的程序叫做“流”计算，但是比如每5秒或者1分钟这样的时间间隔统计一次，那么对于有向无环图的每个节点，其实还是按照“批处理的”吗，即必须等这一分钟的结果汇集为一批处理完了，才传输到协议个Task节点。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复:  不是这样的，每条数据是实时流过的，并不会等待。<br>只有做按时间汇聚的那个节点，它会记录汇总的中间结果（注意，它只记录汇总的中间结果，并不知把所有数据都攒起来），在每个时间窗口结束后会在流中产生一条新的数据（也就是统计结果），流往下游。<br><br>比如，每分钟统计数据条数，汇聚的这个节点每流过一条数据，统计值就+1，它不会保存流过的数据，等到时间窗口结束，这个统计就做完了，它直接生成一条统计结果数据，发往下游。<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-04 22:16:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/63/85/1dc41622.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>姑射仙人</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，采用时序数据库比如prometheus，然后用grafana去做展示监控指标信息。感觉从表现形式上和流计算有些类似，时序数据库也支持统计功能。这两种形式的架构和使用场景，老师有什么看法呢？（除了并行计算这个点）</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 场景不太一样，这种场景更在乎的是海量吞吐，快速查询，对于数据可靠性和一致性的要求没那么高，毕竟一个图标上少一两个点也没什么关系。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-09 18:23:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/a4/2b/3ba9f64b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Devin</span>
  </div>
  <div class="_2_QraFYR_0">思考题：槽数和并行度的关系<br>“槽数 Task Slot”代表的是每个 Task Manager 可以同时执行的子任务 Sub Task 数<br>“并行度 Parallelism”代表的是执行该任务的线程数，每个线程对应的就是一个 Sub Task，这些Sub Task可能在同一个 Task Manager 中，也可能分布在多个 Task Manager中</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-02 10:36:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/d1/29/1b1234ed.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>DFighting</span>
  </div>
  <div class="_2_QraFYR_0">上一篇留言无意提交了，文章具体内容可以谷歌搜索Streaming 101,Streaming 102</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Streaming 101 这篇博文就是Flink的理论基础。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-28 14:26:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/87/6b/0b6cd39a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>朱月俊</span>
  </div>
  <div class="_2_QraFYR_0">flink框架是如何根据逻辑代码进行子task拆解的哈?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个过程还是比较复杂的，很难一两句话讲清楚。<br>你可以去看一下相关的文章或者Flink的源码，重点看一下：StreamGraph - JobGraph - ExecutionGraph这个过程。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-25 13:12:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c7/e6/11f21cb4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>川杰</span>
  </div>
  <div class="_2_QraFYR_0">老师，请问网上说的ACK机制，在消息队列中到底什么场景下要使用呢？<br>我理解，异步线程发送消息后，虽然主线程没法捕获异常，但子线程也可以判断出是否发送成功。那么为什么还要等待接收方返回一个数据处理完的结果呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个原因很简单，你想一下，客户端发送成功并不等于服务端处理成功，如果数据在网络传输过程中丢了呢？如果服务端在处理过程是失败了呢？所以，需要客户端收到服务端明确的告知：”数据我收到并且处理成功了“，才能保证数据不会丢失。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-01 14:10:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/87/6b/0b6cd39a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>朱月俊</span>
  </div>
  <div class="_2_QraFYR_0">slot是槽位的意思，每个槽位可以运行一个sub task。因此，TaskManager中的槽位有两个含义：（1）最多容纳sub task的个数；（2）最高并发数；</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-01 21:36:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/21/03/c5/600fd645.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>tianbingJ</span>
  </div>
  <div class="_2_QraFYR_0">感觉还是没有说Exactly Once是怎么实现的</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-01 13:15:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/aa/24/01162b6c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>UncleNo2</span>
  </div>
  <div class="_2_QraFYR_0">那，计算代码是jobmanager生成的，还是由执行它的taskmanager生成的？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-15 12:01:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/bf/22/26530e66.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>趁早</span>
  </div>
  <div class="_2_QraFYR_0">Flink</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-17 09:58:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/80/61/ae3bb67c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>毛毛虫大帝</span>
  </div>
  <div class="_2_QraFYR_0">ck是可以通过配置定时触发的，主要是由jobmanager负责管理与存储，每隔这个配置时间就会产生一个checkpoint barrier放到数据流中，并且在jobmanager里生成该pending状态的ck，那么等到所有的task在完成这个barrier前所有的数据处理后，便会提交该ck为completed。那么从现在开始，可以用这个ck来故障恢复了。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-12 18:43:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/d1/29/1b1234ed.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>DFighting</span>
  </div>
  <div class="_2_QraFYR_0">taskmanager.numberOfTaskSlots表示TaskManager可以支配的CPU内核数，和并行度并不是一个层次的概念，一个基础配置，一个是运行时可以从集群中分配到多少资源。<br>ps:关于流计算的相关知识有两篇文章可以学习下，我觉得他们是最好的Flink入门papper了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-28 14:23:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c7/e6/11f21cb4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>川杰</span>
  </div>
  <div class="_2_QraFYR_0">如果服务端在处理过程是失败了呢？所以，需要客户端收到服务端明确的告知：”数据我收到并且处理成功了“，才能保证数据不会丢失。<br>老师您好，对于上一问题，我还有疑问。<br>对于服务端处理过程中的失败，假设场景: 业务A处理完毕后，数据需要落地，结果数据保存时出现异常，无法正常落地。<br>对于这种场景，应该是业务处理完毕后就发送确认给消息产生者吧？<br>我想表达的意思是，对于服务端这种业务场景，是否使用ACK，应该还是要具体问题具体分析的吧？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 一般应该是数据持久化完成后在发送消费成功确认。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-02 17:20:57</div>
  </div>
</div>
</div>
</li>
</ul>