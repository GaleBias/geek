<audio title="14 _ 可观测（下）： 如何构建多维度视角下的Serverless监测体系？" src="https://static001.geekbang.org/resource/audio/f7/5e/f70893d9419e221e133d96e18bac975e.mp3" controls="controls"></audio> 
<p>你好，我是静远。</p><p>上一节课，我们一起梳理了Serverless下可观测的重要性和构建可观测监测体系的要点，也结合案例，学习了指标的收集方法，了解了FaaS形态下指标上报的架构设计和注意事项。</p><p>今天这节课，我们继续来看可观测的另外两个数据支柱：日志和链路追踪。</p><h2>日志</h2><p>我们知道，通常在运维一个系统的时候，从监控大盘了解问题的大致轮廓后，经常会根据日志去查看具体的错误细节。​</p><p>日志的作用是记录离散事件，并通过分析这些记录了解程序的整体行为，比如出现过哪些关键数据，调用过哪些方法。也就是说，它能够帮我们定位问题的根源。</p><p>在函数计算场景下，我们需要考虑到用户日志与系统日志两种类型。其中，用户日志记录的主要是用户函数代码中业务流程发生的过程。这部分日志信息是在函数维度上独立收集的，并且用户可以通过前端控制台查看相关日志信息。而系统日志，则是整个平台侧发生事件的信息记录，最终汇总在一起，供平台侧的运维或开发人员排查问题。</p><h3>日志数据源</h3><p>那么日志应该什么时候打印又应该怎么打印呢？</p><p></p><p>首先，<strong>我们<strong><strong>需要</strong></strong>明确日志的级别</strong>。以系统日志为例，常见的包括Error、Info和Warn，分别表示错误日志、信息日志以及警告日志，在开发调试过程中可能还会用到Debug类型。我们需要根据实际的执行逻辑来设定不同的等级。</p><!-- [[[read_end]]] --><p>其次，在添加日志时，我们要<strong>尽可能<strong><strong>地收敛</strong></strong>错误信息</strong>，尽量避免重复信息的打印，因为频繁的IO不仅会加大日志采集的工作量，更会影响服务性能。</p><p>再次，<strong>日志应该尽量打在每个服务模块的入口和出口</strong>，服务内方法之间的调用和错误信息，也应该尽量通过传递的方式上报，而不是每个方法内都打印一条日志。</p><p>比如函数计算中请求的调度过程。调度模块中可能涉及到获取元信息、鉴权、并发度限制、获取函数实例信息等等一系列串行过程，每一个过程都可能包含多个方法之间的调用，其中任意一步出现问题都会导致调度失败。那么，我们在开发时就要尽可能给每个方法都添加一个error类型的返回值，在出现错误后，只需按照递归调用栈逐级返回，最终在入口处打印一条即可。这样可以有效减少重复信息的打印次数。</p><p>另外，对于函数开发者用到的用户日志，我也有两条使用上的建议。</p><p>一方面，<strong>减少print的使用，控制整体的信息大小</strong><strong>。</strong>为了方便在函数这个黑盒中快速定位问题，一些开发者习惯性地将print当成Debug工具来使用，但因为平台侧都会对单次的函数执行日志有一定限制，所以在函数开发过程中应该尽量减少无用信息的打印。</p><p>另一方面，<strong>关注Event。</strong>Event作为函数入口的基本传参，携带了请求源的关键信息，关注Event，也可以方便后续的溯源。</p><h3>日志的采集与清洗</h3><p>有了日志数据后，我们就可以进行收集了。</p><p>系统日志的数据都会写入到固定路径的文件中，而用户日志，通常都是采用DaemonSet日志组件进行收集。所以，一般函数实例内的日志文件都会存放在节点的挂载路径下，或者是集群内的持久卷中。</p><p>前面我们也提到用户日志是以函数为粒度的，而为了缓解不断增长的日志数据造成的节点磁盘的压力，通常会一次请求对应一个日志文件，请求结束则上报并删除文件内容。而系统日志则可以通过设置“定时删除任务”来处理。</p><p>在开源日志收集器的选型上，常用的有Logstash，Fluentd，Fluent-Bit以及Vector等比较不错的采集工具，他们之间各有不同的优势。</p><p>具体的对比，你可以参考Stela Udovicic 2021年12月在<a href="https://era.co/blog/choose-open-source-log-collector">ERA Software’s blog</a>的文章，她指出，我们很难找到一个完美的日志收集器，选择正确的日志收集器主要取决于你自己的特定需求。</p><p>比如，如果你需要资源占用较少的日志收集器，那么使用Vector或者Fluent-Bit就是一个不错的选择，而不是占用资源较高的Logstash。如果你需要寻找不具供应商色彩的收集器，那么Fluentd和Fluent-Bit是不错的选择。</p><p>在函数计算平台的构建中，通常我们会结合这几种工具的能力共同部署，你也可以根据具体的业务情况自己选择。这里，为了便于你理解他们各自的优势，我画了一个示意图，来看一下数据采集的具体流程。</p><p><img src="https://static001.geekbang.org/resource/image/67/4a/6761643139e5d8f9f3c5be425268454a.jpg?wh=1920x808" alt="图片"></p><p>在采集数据时，由于Fluent-Bit在Kubernetes集群等容器化环境中的运行比较出色，所以通常我们会使用轻量的Fluent-Bit对日志数据进行整体的上报，如果是集群的日志信息，则会以DaemonSet的形式部署。</p><p>因为Logstash过滤功能强大，但资源耗费多，所以并不能像Fluent-Bit那样以DaemonSet的形式部署在整个集群，只需部署少量虚机实例，并利用Logstash进行整体的数据清洗即可。</p><p>如果考虑到峰值问题，比如前面提到的某一时刻存在请求高峰导致日志量显著增大，也可以利用kafka缓冲一轮。最后，再由Logstash将过滤后的数据交由相应的存储服务。</p><p></p><h3>日志的存储与检索</h3><p>最后，我们再说日志的存储和检索。</p><p><img src="https://static001.geekbang.org/resource/image/67/4a/6761643139e5d8f9f3c5be425268454a.jpg?wh=1920x808" alt="图片"></p><p>Logstash支持丰富的插件，除了可以支持像kafka、本地文件等多种数据输入源，输出的部分同样也支持与Elasticsearch、对象存储等多种数据存储服务的对接。</p><p>其中，Elasticsearch是一个分布式、高扩展、高实时的搜索与数据分析引擎，配合Kibana这种数据可视化工具，能快速搜索和分析我们上传的日志。</p><p></p><p>为了方便利用Kibana进行快速筛查，我们在日志打印阶段就应该以键值对的形式标记出关键信息，这样就可以在Kibana根据key进行筛查，比如：</p><p></p><pre><code class="language-yaml">{
&nbsp;&nbsp;&nbsp;&nbsp;"level":"info",&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; //日志等级
&nbsp;&nbsp;&nbsp;&nbsp;"ts":1657957846.7413378,&nbsp;&nbsp; // 时间戳
&nbsp;&nbsp;&nbsp;&nbsp;"caller":"apiserver/handler.go:154", // 调用的代码行数
&nbsp;&nbsp;&nbsp;&nbsp;"msg":"service start", // 关键信息
&nbsp;&nbsp;&nbsp;&nbsp;"request_id":"41bae18f-a083-493f-af43-7c3aed7ec53c",
&nbsp;&nbsp;&nbsp;&nbsp;"service":"apiserver"&nbsp; // 服务名称
}
&nbsp;
</code></pre><p>另外，如果需要对日志文件进行溯源，或者需要考虑为函数计算平台拓展一些长期的数据报表功能，也可以让Logstash对接一个对象存储服务。</p><h2>链路</h2><p>了解了指标和日志两个可观测数据之后，我们再去看第三个数据，链路。</p><p>除了代码出错，在一些延迟敏感的场景下，性能分析也是必不可少的，尤其是函数计算这种架构复杂，模块交互较多的服务。这个时候，链路追踪功能就派上了用场。</p><p>在函数计算场景下，它不仅可以提高函数计算系统的可观察性，帮助系统管理员检测、诊断系统的性能问题以保证预期的服务水平，还可以帮助开发者追踪函数的执行过程，快速分析、诊断函数计算架构下的调用关系及性能瓶颈，提高开发和诊断效率。</p><h3>链路信息</h3><p>我们先说链路信息的获取。对用户而言，更关心的是端到端的整体耗时，而除了代码本身的执行，其余耗时主要发生在<a href="https://time.geekbang.org/column/article/563691">冷启动</a>的准备阶段。因此，平台可以默认提供给用户函数总耗时以及冷启动过程的耗时，其中也可以包括准备代码、运行时初始化等步骤的耗时。</p><p>而在复杂的业务场景中，往往会涉及到函数与函数或者函数与其他云服务之间的调用。这时，我们可以为开发者提供自定义的链路支持，将链路信息记录在相应的结构体中（如Header），就可以完整地串联起整个外部的调用链路。而内部的调用链路，也可以通过上下文的形式用SDK去处理。</p><p>通过这种与内置链路结合的方式，平台可以有效地帮助用户定位出超时、性能瓶颈以及涉及多个云服务关联等类似的故障问题。</p><p>在平台层面，则可以根据模块之间的关系以及系统架构的实际情况来构建链路，整体思路和用户侧的链路构建差不多。不过需要注意的是，并不是链路信息越详细越好，因为链路追踪本身也需要耗费一定资源，所以最好根据实际的运维需求来构建。</p><h3>链路拓扑的可视化</h3><p>那么，我们如何追踪链路信息并友好地展示出来呢？常用的解决方案，一般是基于标准的OpenTelemetry协议，利用其提供的SDK和Otel Agent完成对链路Span的生成、传播和上报，最终通过分布式追踪系统（如Jaeger）进行收集，形成链路拓扑的可视化。</p><p>结合函数计算的特点，这里我也给出了一个基本的链路追踪功能架构图供你参考。从图中可以看出，OpenTelemetry可以通过三种方式来上报链路信息，包括直接使用SDK、Agent Sidecar和Agent DaemonSet，你可以根据自己的业务情况选择一种，或者多种组合方式。</p><p><img src="https://static001.geekbang.org/resource/image/d9/c1/d9d596b6d1ebd73129bfb1ff77a466c1.jpg?wh=1920x1130" alt="图片"></p><p>以Jaeger为例，节点服务可以使用Opentelemetry 的SDK或者Agent上报链路信息，通过Jaeger Collector统一收集，并由ElasticSearch进行存储，最终由Jaeger-Query负责展示数据查找。</p><p>到这里，我们来解答一下上一节课的课后思考题，Metrics指标数据是否也可以用此方式来收集呢？完全可以，基于Otel Agent，我们可以将数据发往Kafka、Promethues等数据存储后端的Backends。</p><h2>小结</h2><p>最后，我来小结一下这两节课的内容。这两节课，我们一直在讨论函数计算平台下可观测体系的解决方案。我们可以按照可观测中指标、日志和链路这三要素的架构去构建解决方案。</p><p>首先，监控指标是指对系统中某一类信息的统计聚合。在函数计算平台上的监控，不仅需要考虑常见资源指标，比如CPU、Memory利用率等，还需要考虑用户实际关心的业务指标，比如函数调用次数、错误次数、执行时间等。</p><p>再说日志的构建。日志起到的作用，更像是在“保留现场”。通过日志，我们可以分析出程序的行为，比如曾经调用过什么方法，操作过哪些数据等。在打印日志时，我们也需要关注函数在关键节点上的输出。</p><p>最后，链路的追踪是通过对请求打标、透传、串联还原完整的请求过程。追踪主要是为了排障和优化，比如分析调用链的哪一部分、哪个方法出现了错误或阻塞，输入输出是否符合预期等。服务之间的链路信息耶可以通过Header进行传递，而整个数据过程其实和日志采集类似。但从形态上来看，<strong>日志更像是离散的事件，而链路追踪更像是连续的事件</strong>。</p><h2>思考题</h2><p>好了，这节课到这里也就结束了，最后我给你留了一个思考题。</p><p>随着OpenTelemetry的盛行，原则上使用一套Library或SDK，就可以自动地收集三种数据，然后由统一的Collector处理，但实际应用中，还会受限于新老系统、Otel的成熟度，你是怎么处理的？</p><p>欢迎在留言区写下你的思考和答案，我们一起交流讨论。感谢你的阅读，也欢迎你把这节课分享给更多的朋友一起交流学习。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/a0/c5/9259d5ca.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>daydaygo</span>
  </div>
  <div class="_2_QraFYR_0">目前已经从 trace1.0（opentracing jaeger）升降到 trace2.0（otel），提升非常明显：采样率100%</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-18 09:25:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/60/a1/8f003697.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>静心</span>
  </div>
  <div class="_2_QraFYR_0">可以使用Filebeat吗？Fluent-Bit 与 Filebeat哪个更好？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以使用，从使用的扩展性和上下游来看，更推荐Fluentbit </p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-02 16:35:37</div>
  </div>
</div>
</div>
</li>
</ul>