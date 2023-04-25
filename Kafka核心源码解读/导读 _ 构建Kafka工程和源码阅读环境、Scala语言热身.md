<audio title="导读 _ 构建Kafka工程和源码阅读环境、Scala语言热身" src="https://static001.geekbang.org/resource/audio/f5/8e/f5dd9821615c4c3dd4e62391b7a94a8e.mp3" controls="controls"></audio> 
<p>你好，我是胡夕。</p><p>从今天开始，我们就要正式走入Kafka源码的世界了。既然咱们这个课程是教你阅读Kafka源码的，那么，你首先就得掌握如何在自己的电脑上搭建Kafka的源码环境，甚至是知道怎么对它们进行调试。在这节课，我展示了很多实操步骤，建议你都跟着操作一遍，否则很难会有特别深刻的认识。</p><p>话不多说，现在，我们就先来搭建源码环境吧。</p><h2>环境准备</h2><p>在阅读Kafka源码之前，我们要先做一些必要的准备工作。这涉及到一些工具软件的安装，比如Java、Gradle、Scala、IDE、Git，等等。</p><p>如果你是在Linux或Mac系统下搭建环境，你需要安装Java、IDE和Git；如果你使用的是Windows，那么你需要全部安装它们。</p><p>咱们这个课程统一使用下面的版本进行源码讲解。</p><ul>
<li>Oracle Java 8：我们使用的是Oracle的JDK及Hotspot JVM。如果你青睐于其他厂商或开源的Java版本（比如OpenJDK），你可以选择安装不同厂商的JVM版本。</li>
<li>Gradle 6.3：我在这门课里带你阅读的Kafka源码是社区的Trunk分支。Trunk分支目前演进到了2.5版本，已经支持Gradle 6.x版本。你最好安装Gradle 6.3或更高版本。</li>
<li>Scala 2.13：社区Trunk分支编译当前支持两个Scala版本，分别是2.12和2.13。默认使用2.13进行编译，因此我推荐你安装Scala 2.13版本。</li>
<li>IDEA + Scala插件：这门课使用IDEA作为IDE来阅读和配置源码。我对Eclipse充满敬意，只是我个人比较习惯使用IDEA。另外，你需要为IDEA安装Scala插件，这样可以方便你阅读Scala源码。</li>
<li>Git：安装Git主要是为了管理Kafka源码版本。如果你要成为一名社区代码贡献者，Git管理工具是必不可少的。</li>
</ul><!-- [[[read_end]]] --><h2>构建Kafka工程</h2><p>等你准备好以上这些之后，我们就可以来构建Kafka工程了。</p><p>首先，我们下载Kafka源代码。方法很简单，找一个干净的源码路径，然后执行下列命令去下载社区的Trunk代码即可：</p><pre><code>$ git clone https://github.com/apache/kafka.git
</code></pre><p>在漫长的等待之后，你的路径上会新增一个名为kafka的子目录，它就是Kafka项目的根目录。如果在你的环境中，上面这条命令无法执行的话，你可以在浏览器中输入<a href="https://codeload.github.com/apache/kafka/zip/trunk">https://codeload.github.com/apache/kafka/zip/trunk</a>下载源码ZIP包并解压，只是这样你就失去了Git管理，你要手动链接到远程仓库，具体方法可以参考这篇<a href="https://help.github.com/articles/fork-a-repo/">Git文档</a>。</p><p>下载完成后，你要进入工程所在的路径下，也就是进入到名为kafka的路径下，然后执行相应的命令来构建Kafka工程。</p><p>如果你是在Mac或Linux平台上搭建环境，那么直接运行下列命令构建即可：</p><pre><code>$ ./gradlew build
</code></pre><p>该命令首先下载Gradle Wrapper所需的jar文件，然后对Kafka工程进行构建。需要注意的是，在执行这条命令时，你很可能会遇到下面的这个异常：</p><pre><code>Failed to connect to raw.githubusercontent.com port 443: Connection refused
</code></pre><p>如果碰到了这个异常，你也不用惊慌，你可以去这个<a href="https://raw.githubusercontent.com/gradle/gradle/v6.3.0/gradle/wrapper/gradle-wrapper.jar">官网链接</a>或者是我提供的<a href="https://pan.baidu.com/s/1tuVHunoTwHfbtoqMvoTNoQ">链接</a>（提取码：ntvd）直接下载Wrapper所需的Jar包，手动把这个Jar文件拷贝到kafka路径下的gradle/wrapper子目录下，然后重新执行gradlew build命令去构建工程。</p><p>我想提醒你的是，官网链接包含的版本号是v6.3.0，但是该版本后续可能会变化，因此，你最好先打开gradlew文件，去看一下社区使用的是哪个版本的Gradle。<strong>一旦你发现版本不再是v6.3.0了，那就不要再使用我提供的链接了。这个时候，你需要直接去官网下载对应版本的Jar包</strong>。</p><p>举个例子，假设gradlew文件中使用的Gradle版本变更为v6.4.0，那么你需要把官网链接URL中的版本号修改为v6.4.0，然后去下载这个版本的Wrapper Jar包。</p><p>如果你是在Windows平台上构建，那你就不能使用Gradle Wrapper了，因为Kafka没有提供Windows平台上可运行的Wrapper Bat文件。这个时候，你只能使用你自己的环境中自行安装的Gradle。具体命令是：</p><pre><code>kafka&gt; gradle.bat build
</code></pre><p>无论是gradle.bat build命令，还是gradlew build命令，首次运行时都要花费相当长的时间去下载必要的Jar包，你要耐心地等待。</p><p>下面，我用一张图给你展示下Kafka工程的各个目录以及文件：</p><p><img src="https://static001.geekbang.org/resource/image/a2/f7/a2ef664cd8d5494f55919643df1305f7.png?wh=940*770" alt=""></p><p>这里我再简单介绍一些主要的组件路径。</p><ul>
<li><strong>bin目录</strong>：保存Kafka工具行脚本，我们熟知的kafka-server-start和kafka-console-producer等脚本都存放在这里。</li>
<li><strong>clients目录</strong>：保存Kafka客户端代码，比如生产者和消费者的代码都在该目录下。</li>
<li><strong>config目录</strong>：保存Kafka的配置文件，其中比较重要的配置文件是server.properties。</li>
<li><strong>connect目录</strong>：保存Connect组件的源代码。我在开篇词里提到过，Kafka Connect组件是用来实现Kafka与外部系统之间的实时数据传输的。</li>
<li><strong>core目录</strong>：保存Broker端代码。Kafka服务器端代码全部保存在该目录下。</li>
<li><strong>streams目录</strong>：保存Streams组件的源代码。Kafka Streams是实现Kafka实时流处理的组件。</li>
</ul><p>其他的目录要么不太重要，要么和配置相关，这里我就不展开讲了。</p><p>除了上面的gradlew build命令之外，我再介绍一些常用的构建命令，帮助你调试Kafka工程。</p><p>我们先看一下测试相关的命令。Kafka源代码分为4大部分：Broker端代码、Clients端代码、Connect端代码和Streams端代码。如果你想要测试这4个部分的代码，可以分别运行以下4条命令：</p><pre><code>$ ./gradlew core:test
$ ./gradlew clients:test
$ ./gradlew connect:[submodule]:test
$ ./gradlew streams:test
</code></pre><p>你可能注意到了，在这4条命令中，Connect组件的测试方法不太一样。这是因为Connect工程下细分了多个子模块，比如api、runtime等，所以，你需要显式地指定要测试的子模块名，才能进行测试。</p><p>如果你要单独对某一个具体的测试用例进行测试，比如单独测试Broker端core包的LogTest类，可以用下面的命令：</p><pre><code>$ ./gradlew core:test --tests kafka.log.LogTest
</code></pre><p>另外，如果你要构建整个Kafka工程并打包出一个可运行的二进制环境，就需要运行下面的命令：</p><pre><code>$ ./gradlew clean releaseTarGz
</code></pre><p>成功运行后，core、clients和streams目录下就会分别生成对应的二进制发布包，它们分别是：</p><ul>
<li><strong>kafka-2.12-2.5.0-SNAPSHOT.tgz</strong>。它是Kafka的Broker端发布包，把该文件解压之后就是标准的Kafka运行环境。该文件位于core路径的/build/distributions目录。</li>
<li><strong>kafka-clients-2.5.0-SNAPSHOT.jar</strong>。该Jar包是Clients端代码编译打包之后的二进制发布包。该文件位于clients目录下的/build/libs目录。</li>
<li><strong>kafka-streams-2.5.0-SNAPSHOT.jar</strong>。该Jar包是Streams端代码编译打包之后的二进制发布包。该文件位于streams目录下的/build/libs目录。</li>
</ul><h2>搭建源码阅读环境</h2><p>刚刚我介绍了如何使用Gradle工具来构建Kafka项目工程，现在我来带你看一下如何利用IDEA搭建Kafka源码阅读环境。实际上，整个过程非常简单。我们打开IDEA，点击“文件”，随后点击“打开”，选择上一步中的Kafka文件路径即可。</p><p>项目工程被导入之后，IDEA会对项目进行自动构建，等构建完成之后，你可以找到core目录源码下的Kafka.scala文件。打开它，然后右键点击Kafka，你应该就能看到这样的输出结果了：</p><p><img src="https://static001.geekbang.org/resource/image/ce/d2/ce0a63e7627c641da471b48a62860ad2.png?wh=933*372" alt=""></p><p>这就是无参执行Kafka主文件的运行结果。通过这段输出，我们能够学会启动Broker所必需的参数，即指定server.properties文件的地址。这也是启动Kafka Broker的标准命令。</p><p>在开篇词中我也说了，这个课程会聚焦于讲解Kafka Broker端源代码。因此，在正式学习这部分源码之前，我先给你简单介绍一下Broker端源码的组织架构。下图展示了Kafka core包的代码架构：</p><p><img src="https://static001.geekbang.org/resource/image/df/b2/dfdd73cc95ecc5390ebeb73c324437b2.png?wh=940*844" alt=""></p><p>我来给你解释几个比较关键的代码包。</p><ul>
<li>controller包：保存了Kafka控制器（Controller）代码，而控制器组件是Kafka的核心组件，后面我们会针对这个包的代码进行详细分析。</li>
<li>coordinator包：保存了<strong>消费者端的GroupCoordinator代码</strong>和<strong>用于事务的TransactionCoordinator代码</strong>。对coordinator包进行分析，特别是对消费者端的GroupCoordinator代码进行分析，是我们弄明白Broker端协调者组件设计原理的关键。</li>
<li>log包：保存了Kafka最核心的日志结构代码，包括日志、日志段、索引文件等，后面会有详细介绍。另外，该包下还封装了Log Compaction的实现机制，是非常重要的源码包。</li>
<li>network包：封装了Kafka服务器端网络层的代码，特别是SocketServer.scala这个文件，是Kafka实现Reactor模式的具体操作类，非常值得一读。</li>
<li>server包：顾名思义，它是Kafka的服务器端主代码，里面的类非常多，很多关键的Kafka组件都存放在这里，比如后面要讲到的状态机、Purgatory延时机制等。</li>
</ul><p>在后续的课程中，我会挑选Kafka最主要的代码类进行详细分析，帮助你深入了解Kafka Broker端重要组件的实现原理。</p><p>另外，虽然这门课不会涵盖测试用例的代码分析，但在我看来，<strong>弄懂测试用例是帮助你快速了解Kafka组件的最有效的捷径之一</strong>。如果时间允许的话，我建议你多读一读Kafka各个组件下的测试用例，它们通常都位于代码包的src/test目录下。拿Kafka日志源码Log来说，它对应的LogTest.scala测试文件就写得非常完备，里面多达几十个测试用例，涵盖了Log的方方面面，你一定要读一下。</p><h2>Scala 语言热身</h2><p>因为Broker端的源码完全是基于Scala的，所以在开始阅读这部分源码之前，我还想花一点时间快速介绍一下 Scala 语言的语法特点。我先拿几个真实的 Kafka 源码片段来帮你热热身。</p><p>先来看第一个：</p><pre><code>def sizeInBytes(segments: Iterable[LogSegment]): Long =
    segments.map(_.size.toLong).sum
</code></pre><p>这是一个典型的 Scala 方法，方法名是 sizeInBytes。它接收一组 LogSegment 对象，返回一个长整型。LogSegment 对象就是我们后面要谈到的日志段。你在 Kafka 分区目录下看到的每一个.log 文件本质上就是一个 LogSegment。从名字上来看，这个方法计算的是这组 LogSegment 的总字节数。具体方法是遍历每个输入 LogSegment，调用其 size 方法并将其累加求和之后返回。</p><p>再来看一个：</p><pre><code>val firstOffset: Option[Long] = ......

def numMessages: Long = {
    firstOffset match {
      case Some(firstOffsetVal) if (firstOffsetVal &gt;= 0 &amp;&amp; lastOffset &gt;= 0) =&gt; (lastOffset - firstOffsetVal + 1)
      case _ =&gt; 0
    }
  }
</code></pre><p>该方法是 LogAppendInfo 对象的一个方法，统计的是 Broker 端一次性批量写入的消息数。这里你需要重点关注 <strong>match</strong> 和 <strong>case</strong>  这两个关键字，你可以近似地认为它们等同于 Java 中的 switch，但它们的功能要强大得多。该方法统计写入消息数的逻辑是：如果 firstOffsetVal 和 lastOffset 值都大于 0，则写入消息数等于两者的差值+1；如果不存在 firstOffsetVal，则无法统计写入消息数，简单返回 0 即可。</p><p>倘若对你而言，弄懂上面这两段代码有些吃力，我建议你去快速地学习一下Scala语言。重点学什么呢？我建议你重点学习下Scala中对于<strong>集合的遍历语法</strong>，以及<strong>基于match的模式匹配用法</strong>。</p><p>另外，由于Scala具有的函数式编程风格，你至少<strong>要理解Java中Lambda表达式的含义</strong>，这会在很大程度上帮你扫清阅读障碍。</p><p>相反地，如果上面的代码对你来讲很容易理解，那么，读懂Broker端80%的源码应该没有什么问题。你可能还会关心，剩下的那晦涩难懂的20%源码怎么办呢？其实没关系，你可以等慢慢精通了Scala语言之后再进行阅读，它们不会对你熟练掌握核心源码造成影响的。另外，后面涉及到比较难的Scala语法特性时，我还会再具体给你解释的，所以，还是那句话，你完全不用担心语言的问题！</p><h2>总结</h2><p>今天是我们开启Kafka源码分析的“热身课”，我给出了构建Kafka工程以及搭建Kafka源码阅读环境的具体方法。我建议你对照上面的内容完整地走一遍流程，亲身体会一下Kafka工程的构建与源码工程的导入。毕竟，这些都是后面阅读具体Kafka代码的前提条件。</p><p>最后我想再强调一下，阅读任何一个大型项目的源码都不是一件容易的事情，我希望你在任何时候都不要轻言放弃。很多时候，碰到读不懂的代码你就多读几遍，也许稍后就会有醍醐灌顶的感觉。</p><h2>课后讨论</h2><p>熟悉Kafka的话，你一定听说过kafka-console-producer.sh脚本。我前面提到过，该脚本位于工程的bin目录下，你能找到它对应的Java类是哪个文件吗？这个搜索过程能够给你一些寻找Kafka所需类文件的好思路，你不妨去试试看。</p><p>欢迎你在留言区畅所欲言，跟我交流讨论，也欢迎你把文章分享给你的朋友。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/a7/9a/495cb99a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>胡夕</span>
  </div>
  <div class="_2_QraFYR_0">你好，我是胡夕。<br><br>我最近看到了一些在搭建环境时经常遇到的问题，根据这些问题，我丰富了一下这节课的部分内容，并且针对可能会遇到的一些异常，补充了解决方案，希望可以解决你的问题。如果还有其他问题，欢迎留言，我会尽快给你解答～</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-26 10:20:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/49/a5/e4c1c2d4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小文同学</span>
  </div>
  <div class="_2_QraFYR_0">我自己在拉取 github kafka 项目的时候遇到超时的问题。通过 码云 做中介完成了快速拉取，步骤写在了博客里了。<br>https:&#47;&#47;juejin.im&#47;post&#47;5e954374e51d4546f03d9a27</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 厉害！很棒的总结~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-15 11:25:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/00/5c/9cb1792d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>高舒扬</span>
  </div>
  <div class="_2_QraFYR_0">老师，您好。之前也看过很多项目的源码，一直比较困惑的怎么构建自己的代码阅读笔记系统。总感觉读懂了，但不能很好的把读代码的结果记录下来。不知道老师在这方面有什么好的建议吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我的一个建议是：不要认为能把每行代码加上注释就表示自己已经懂了。明白源码的2个标志是：1. 能够自行调试源码；2. 能够在源码上独立编写高阶功能。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-16 20:45:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/67/49/864dba17.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>东风第一枝</span>
  </div>
  <div class="_2_QraFYR_0">该脚本有一行代码：<br>exec $(dirname $0)&#47;kafka-run-class.sh kafka.tools.ConsoleProducer &quot;$@&quot;<br>可以看到，调用了kafka-run-class.sh，看脚本的名字，应该就是用来执行kafka某个类的，并且传入了kafka.tools.ConsoleProducer这个参数<br>所以，对应的Java类是kafka.tools.ConsoleProducer</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Bingo！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-13 22:23:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/ff/bc/368b9f80.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>灰机</span>
  </div>
  <div class="_2_QraFYR_0">胡老师您好， 我看您是mac运行的kafka，方便配置上您的gradle 版本 jdk版本 scala版本么？ 我在用mac执行kafka 的时候 发现Execution failed for task &#39;:core:Kafka.main()&#39;.<br>&gt; Process &#39;command &#39;&#47;Library&#47;Java&#47;JavaVirtualMachines&#47;jdk1.8.0_221.jdk&#47;Contents&#47;Home&#47;bin&#47;java&#39;&#39; finished with non-zero exit value 1  换了很多个版本都可以  windows已经成功执行，但是mac就是死活无法执行 。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我的环境上，jdk版本是build 1.8.0_181-b13，Scala版本是2.12.11，Kafka gradle wrapper用的Gradle版本是5.6.2</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-17 23:40:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_58afe9</span>
  </div>
  <div class="_2_QraFYR_0">是看master分支吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯嗯，我们这次选用trunk分支。Kafka社区里面主干分支叫trunk</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-14 07:54:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>懂码哥(GerryWen)</span>
  </div>
  <div class="_2_QraFYR_0">2020-06-28 拉取trunk代码，gradle版本号6.5.0 Scala版本号2.13.3 <br>版本必须匹配一致，不然 .&#47;gradlew build 会报错。<br>最新推荐 .&#47;gradlew jar</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 强</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-28 22:55:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/hvb2KFTy9wdlpTA5dPVPyow44pDtAFH3abXphxktutTlpe2yveiaTqicz68icsUXkZGTjQnQ5zjX6ZQSRn5C6OicaA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_904551</span>
  </div>
  <div class="_2_QraFYR_0">很困惑 为什么不拉取某一个固定版本的代码，而是去拉trunk，trunk随时更新，后面的人看到的代码和你讲解的代码到时候出入又会很大。看了一下本文也没有标识到底用的哪个kafka版本。还望作者能给出到底是基于哪个kafka版本进行的分析！我觉得这点很重要</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯嗯，很好的建议。使用trunk的初衷是在短短出课程的这段时间，trunk也不太可能有太大的修改。让大家使用trunk就会有每天更新的感觉，有真实参与到社区的感觉。这点比弄懂源码本身要重要得多</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-10 14:33:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/91/80/bc38f890.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>珍妮•玛仕多</span>
  </div>
  <div class="_2_QraFYR_0">胡老师你好，我build中只有9个test用例build失败，在启动kafka.main时报错如下<br><br>2020-11-09T11:42:21.964+0800 [ERROR] [system.err]                        server.properties file                            <br>2020-11-09T11:42:21.964+0800 [ERROR] [system.err] --version            Print version information and exit.                 <br>2020-11-09T11:42:21.998+0800 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter] <br>2020-11-09T11:42:21.998+0800 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter] FAILURE: Build failed with an exception.<br>2020-11-09T11:42:21.999+0800 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter] <br>2020-11-09T11:42:21.999+0800 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter] * What went wrong:<br>2020-11-09T11:42:21.999+0800 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter] Execution failed for task &#39;:core:Kafka.main()&#39;.<br>2020-11-09T11:42:21.999+0800 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter] &gt; Process &#39;command &#39;D:&#47;software&#47;java&#47;jdk&#47;bin&#47;java.exe&#39;&#39; finished with non-zero exit value 1<br>2020-11-09T11:42:21.999+0800 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter] <br>2020-11-09T11:42:21.999+0800 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter] * Try:<br>2020-11-09T11:42:21.999+0800 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter] Run with --stacktrace option to get the stack trace.  Run with --scan to get full insights.<br>2020-11-09T11:42:21.999+0800 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter] <br>2020-11-09T11:42:21.999+0800 [ERROR] [org.gradle.internal.buildevents.BuildExceptionReporter] * Get more help at https:&#47;&#47;help.gradle.org<br>2020-11-09T11:42:21.999+0800 [ERROR] [org.gradle.internal.buildevents.BuildResultLogger] <br>2020-11-09T11:42:21.999+0800 [ERROR] [org.gradle.internal.buildevents.BuildResultLogger] BUILD FAILED in 2s<br>11:42:22: Task execution finished &#39;Kafka.main() --debug&#39;.<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这是因为不加参数启动Kafka，返回值不是0，IDE视为有问题。其实已经打印出了Kafka的帮助提示信息：<br><br>“Print version information and exit.”<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-09 11:48:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>懂码哥(GerryWen)</span>
  </div>
  <div class="_2_QraFYR_0">编译.&#47;gradlew jar报错<br>单独编译.&#47;gradlew srcJar 可以通过<br>编译 .&#47;gradlew docsJar  文档报错。<br><br><br>gerrydeMBP:kafka gerry$ .&#47;gradlew jar<br>Starting a Gradle Daemon (subsequent builds will be faster)<br>&lt;-------------&gt; 0% CONFIGURING [4s]<br><br>&gt; Configure project :<br>Building project &#39;core&#39; with Scala version 2.13.3<br>Building project &#39;streams-scala&#39; with Scala version 2.13.3<br>&lt;-------------&gt; 4% EXECUTING [12s]<br><br>&gt; Task :core:compileJava FAILED<br><br>FAILURE: Build failed with an exception.<br><br>* What went wrong:<br>Execution failed for task &#39;:core:compileJava&#39;.<br>&gt; Could not resolve all files for configuration &#39;:core:compileClasspath&#39;.<br>   &gt; Could not find scala-library-2.13.3.jar (org.scala-lang:scala-library:2.13.3).<br>     Searched in the following locations:<br>         http:&#47;&#47;maven.aliyun.com&#47;nexus&#47;content&#47;repositories&#47;jcenter&#47;org&#47;scala-lang&#47;scala-library&#47;2.13.3&#47;scala-library-2.13.3.jar<br><br>* Try:<br>Run with --stacktrace option to get the stack trace. Run with --info or --debug option to get more log output. Run with --scan to get full insights.<br><br>* Get more help at https:&#47;&#47;help.gradle.org<br><br>Deprecated Gradle features were used in this build, making it incompatible with Gradle 7.0.<br>Use &#39;--warning-mode all&#39; to show the individual deprecation warnings.<br>See https:&#47;&#47;docs.gradle.org&#47;6.5&#47;userguide&#47;command_line_interface.html#sec:command_line_warnings<br><br>BUILD FAILED in 17s<br>9 actionable tasks: 6 executed, 3 up-to-date<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 看样子是修改了maven仓库地址。不过似乎依然是无法获取Scala 2.13<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-29 08:57:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/2b/fd/7013289d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>温故而知新可以为师矣</span>
  </div>
  <div class="_2_QraFYR_0">老师，环境如下：<br>Mac<br>gradle6.5<br>scala2.13<br>jdk8<br>idea2020.1<br>运行Kafka.scala报如下图错误（红框内）：<br>https:&#47;&#47;share.weiyun.com&#47;8jiPccMP<br><br>感谢老师！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个看着是正常现象，你看看是否有打印Kafka的usage信息。这是因为你运行Kafka main时没有指定函数，所以程序指定了exit是非0，那么就会抛出这样的错误。但其实usage已经打印出来了，说明你已经正确的运行了Kafka的main函数</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-26 18:19:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/2d/33/2489480f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>U2</span>
  </div>
  <div class="_2_QraFYR_0"><br>* What went wrong:<br>A problem occurred evaluating root project &#39;kafka&#39;.<br>&gt; Failed to apply plugin [id &#39;org.gradle.scala&#39;]<br>   &gt; Could not find method scala() for arguments [build_8fovasgxikn1u4oofqzi4l8yq$_run_closure5$_closure73$_closure105@2ed041de] on object of type org.gradle.api.plugins.scala.ScalaPlugin.<br><br>* Try:<br>老师，编译报这个错 怎么解决啊？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你使用的是gradle命令还是wrapper的gradlew？感觉像是使用了自己装的gradle版本导致的。无法apply某个plugin可能有很多原因，运行gradle命令时后面加个--stacktrace看下是什么原因吧？也可以把完整的stacktrace贴出来让我看下~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-20 21:45:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/ydVBBkofXDqCyP7pdwkicHZ9xtyEEuZvzrrkeWcnQjZ1ibEgG60eLotQTsKJFpWibuf6e7G9r0I1xaribUAQibPMl7g/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Shine</span>
  </div>
  <div class="_2_QraFYR_0">启动的时候看不到日志，debug不到为什么 Task :core:Kafka.main() FAILED</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 能把完整的log贴出来吗？或者说你在IDE里面直接运行Kafka.scala试试？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-19 13:41:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/01/38/04aae3fc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>济沧海</span>
  </div>
  <div class="_2_QraFYR_0">我这执行gradle和.&#47;gradlew jar没问题，但是右键运行kafka的时候报错了。<br>&gt; Task :core:Kafka.main() FAILED<br>SLF4J: Failed to load class &quot;org.slf4j.impl.StaticLoggerBinder&quot;.<br>SLF4J: Defaulting to no-operation (NOP) logger implementation<br>SLF4J: See http:&#47;&#47;www.slf4j.org&#47;codes.html#StaticLoggerBinder for further details.<br>USAGE: java [options] KafkaServer server.properties [--override property=value]*<br>Option               Description                                         <br>------               -----------                                         <br>--override &lt;String&gt;  Optional property that should override values set in<br>                       server.properties file                            <br>--version            Print version information and exit.                 <br><br>FAILURE: Build failed with an exception.<br><br>* What went wrong:<br>Execution failed for task &#39;:core:Kafka.main()&#39;.<br>&gt; Process &#39;command &#39;&#47;Library&#47;Java&#47;JavaVirtualMachines&#47;jdk1.8.0_221.jdk&#47;Contents&#47;Home&#47;bin&#47;java&#39;&#39; finished with non-zero exit value 1<br><br>jdk --1.8.0_221<br>scala--2.12.11<br>gradle-- 5.6.2<br>os--Mac OS X 10.15.3 x86_64<br>换了好几次Scala和gradle版本还是这个问题。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不用管slf4j那些warning，你已经运行成功了啊。不是已经看到&quot;Usage&quot;了吗。另外建议在IDE中调试这个命令</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-17 10:31:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/1a/05/f154d134.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>刘楠</span>
  </div>
  <div class="_2_QraFYR_0">kafka-console-producer.sh <br>文件中有写exec $(dirname $0)&#47;kafka-run-class.sh kafka.tools.ConsoleProducer &quot;$@&quot; <br>core包中 scala是 <br>kafka.tools.ConsoleProducer.scala<br><br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯嗯，有兴趣读下ConsoleProducer源码。你会发现console-producer原来有这么多用途，哈哈哈</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-16 10:30:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/25/06/9f8f2b8e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Glenn.G</span>
  </div>
  <div class="_2_QraFYR_0">板凳</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-13 17:59:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/48/ee/872ad07e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>西西弗与卡夫卡</span>
  </div>
  <div class="_2_QraFYR_0">沙发</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-13 17:27:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83eplWzPJuvFyWRNzOAVYjuV6mOypHCaibuZxWXLStwk0kXC19OYgPlhXnEk0DWLVhibUB0Sfp93wkjTQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>alvinzhang</span>
  </div>
  <div class="_2_QraFYR_0">你可以找到 core 目录源码下的 Kafka.scala 文件 , 为什么我的目录下没有这个呢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-09 10:49:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83eplWzPJuvFyWRNzOAVYjuV6mOypHCaibuZxWXLStwk0kXC19OYgPlhXnEk0DWLVhibUB0Sfp93wkjTQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>alvinzhang</span>
  </div>
  <div class="_2_QraFYR_0">最新匹配的版本有谁会有么？<br>我build的时候总是出错。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-08 12:20:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c6/6f/ac3003fa.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>xiong</span>
  </div>
  <div class="_2_QraFYR_0">该类在core目录中，包名是kafka.tools</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-24 19:02:51</div>
  </div>
</div>
</div>
</li>
</ul>