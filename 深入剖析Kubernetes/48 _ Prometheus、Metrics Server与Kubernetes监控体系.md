<audio title="48 _ Prometheus、Metrics Server与Kubernetes监控体系" src="https://static001.geekbang.org/resource/audio/75/79/75a1f40a2a9f78e948c578d1d954a879.mp3" controls="controls"></audio> 
<p>你好，我是张磊。今天我和你分享的主题是：Prometheus、Metrics Server与Kubernetes监控体系。</p><p>通过前面的文章，我已经和你分享过了Kubernetes 的核心架构，编排概念，以及具体的设计与实现。接下来，我会用3篇文章，为你介绍 Kubernetes 监控相关的一些核心技术。</p><p>首先需要明确指出的是，Kubernetes 项目的监控体系曾经非常繁杂，在社区中也有很多方案。但这套体系发展到今天，已经完全演变成了以Prometheus 项目为核心的一套统一的方案。</p><p>在这里，可能有一些同学对 Prometheus 项目还太不熟悉。所以，我先来简单为你介绍一下这个项目。</p><p>实际上，Prometheus 项目是当年 CNCF 基金会起家时的“第二把交椅”。而这个项目发展到今天，已经全面接管了 Kubernetes 项目的整套监控体系。</p><p>比较有意思的是，Prometheus项目与 Kubernetes 项目一样，也来自于 Google 的 Borg 体系，它的原型系统，叫作BorgMon，是一个几乎与 Borg 同时诞生的内部监控系统。而 Prometheus 项目的发起原因也跟 Kubernetes 很类似，都是希望通过对用户更友好的方式，将 Google 内部系统的设计理念，传递给用户和开发者。</p><!-- [[[read_end]]] --><p>作为一个监控系统，Prometheus 项目的作用和工作方式，其实可以用如下所示的一张官方示意图来解释。</p><p><img src="https://static001.geekbang.org/resource/image/2a/d3/2ada1ece66fcc81d704c2ba46f9dd7d3.png?wh=1351*811" alt=""><br>
可以看到，Prometheus 项目工作的核心，是使用 Pull （抓取）的方式去搜集被监控对象的 Metrics 数据（监控指标数据），然后，再把这些数据保存在一个 TSDB （时间序列数据库，比如 OpenTSDB、InfluxDB 等）当中，以便后续可以按照时间进行检索。</p><p>有了这套核心监控机制， Prometheus 剩下的组件就是用来配合这套机制的运行。比如 Pushgateway，可以允许被监控对象以Push 的方式向 Prometheus 推送 Metrics 数据。而Alertmanager，则可以根据 Metrics 信息灵活地设置报警。当然， Prometheus 最受用户欢迎的功能，还是通过 Grafana 对外暴露出的、可以灵活配置的监控数据可视化界面。</p><p>有了 Prometheus 之后，我们就可以按照Metrics 数据的来源，来对 Kubernetes 的监控体系做一个汇总了。</p><p><strong>第一种 Metrics，是宿主机的监控数据。</strong>这部分数据的提供，需要借助一个由 Prometheus 维护的<a href="https://github.com/prometheus/node_exporter">Node Exporter</a> 工具。一般来说，Node Exporter 会以 DaemonSet 的方式运行在宿主机上。其实，所谓的 Exporter，就是代替被监控对象来对 Prometheus 暴露出可以被“抓取”的 Metrics 信息的一个辅助进程。</p><p>而 Node Exporter 可以暴露给 Prometheus 采集的Metrics 数据， 也不单单是节点的负载（Load）、CPU 、内存、磁盘以及网络这样的常规信息，它的 Metrics 指标可以说是“包罗万象”，你可以查看<a href="https://github.com/prometheus/node_exporter#enabled-by-default">这个列表</a>来感受一下。</p><p><strong>第二种 Metrics，是来自于 Kubernetes 的 API Server、kubelet 等组件的/metrics API</strong>。除了常规的 CPU、内存的信息外，这部分信息还主要包括了各个组件的核心监控指标。比如，对于 API Server 来说，它就会在/metrics API 里，暴露出各个 Controller 的工作队列（Work Queue）的长度、请求的 QPS 和延迟数据等等。这些信息，是检查 Kubernetes 本身工作情况的主要依据。</p><p><strong>第三种 Metrics，是 Kubernetes 相关的监控数据。</strong>这部分数据，一般叫作 Kubernetes 核心监控数据（core metrics）。这其中包括了 Pod、Node、容器、Service 等主要 Kubernetes 核心概念的 Metrics。</p><p>其中，容器相关的 Metrics 主要来自于 kubelet 内置的 cAdvisor 服务。在 kubelet 启动后，cAdvisor 服务也随之启动，而它能够提供的信息，可以细化到每一个容器的CPU 、文件系统、内存、网络等资源的使用情况。</p><p>需要注意的是，这里提到的 Kubernetes 核心监控数据，其实使用的是 Kubernetes 的一个非常重要的扩展能力，叫作 Metrics Server。</p><p>Metrics Server 在 Kubernetes 社区的定位，其实是用来取代 Heapster 这个项目的。在 Kubernetes 项目发展的初期，Heapster 是用户获取 Kubernetes 监控数据（比如 Pod 和 Node的资源使用情况） 的主要渠道。而后面提出来的 Metrics Server，则把这些信息，通过标准的 Kubernetes API 暴露了出来。这样，Metrics 信息就跟 Heapster 完成了解耦，允许 Heapster 项目慢慢退出舞台。</p><p>而有了 Metrics Server 之后，用户就可以通过标准的 Kubernetes API 来访问到这些监控数据了。比如，下面这个 URL：</p><pre><code>http://127.0.0.1:8001/apis/metrics.k8s.io/v1beta1/namespaces/&lt;namespace-name&gt;/pods/&lt;pod-name&gt;
</code></pre><p>当你访问这个 Metrics API时，它就会为你返回一个 Pod 的监控数据，而这些数据，其实是从 kubelet 的 Summary API （即&lt;kubelet_ip&gt;:&lt;kubelet_port&gt;/stats/summary）采集而来的。Summary API 返回的信息，既包括了 cAdVisor的监控数据，也包括了 kubelet 本身汇总的信息。</p><p>需要指出的是， Metrics Server 并不是 kube-apiserver 的一部分，而是通过 Aggregator 这种插件机制，在独立部署的情况下同 kube-apiserver 一起统一对外服务的。</p><p>这里，Aggregator APIServer 的工作原理，可以用如下所示的一幅示意图来表示清楚：</p><p><img src="https://static001.geekbang.org/resource/image/0b/09/0b767b5224ad1906ddc4cce075618809.png?wh=736*367" alt=""></p><blockquote>
<p>备注：图片出处<a href="https://blog.jetstack.io/blog/resource-and-custom-metrics-hpa-v2/">https://blog.jetstack.io/blog/resource-and-custom-metrics-hpa-v2/</a></p>
</blockquote><p>可以看到，当Kubernetes 的 API Server 开启了 Aggregator 模式之后，你再访问apis/metrics.k8s.io/v1beta1的时候，实际上访问到的是一个叫作kube-aggregator 的代理。而kube-apiserver，正是这个代理的一个后端；而 Metrics Server，则是另一个后端。</p><p>而且，在这个机制下，你还可以添加更多的后端给这个 kube-aggregator。所以<strong>kube-aggregator其实就是一个根据 URL 选择具体的 API 后端的代理服务器。</strong>通过这种方式，我们就可以很方便地扩展 Kubernetes 的 API 了。</p><p>而 Aggregator 模式的开启也非常简单：</p><ul>
<li>如果你是使用 kubeadm 或者<a href="https://github.com/kubernetes/kubernetes/blob/master/cluster/kube-up.sh">官方的kube-up.sh 脚本</a>部署Kubernetes集群的话，Aggregator 模式就是默认开启的；</li>
<li>如果是手动 DIY 搭建的话，你就需要在 kube-apiserver 的启动参数里加上如下所示的配置：</li>
</ul><pre><code>--requestheader-client-ca-file=&lt;path to aggregator CA cert&gt;
--requestheader-allowed-names=front-proxy-client
--requestheader-extra-headers-prefix=X-Remote-Extra-
--requestheader-group-headers=X-Remote-Group
--requestheader-username-headers=X-Remote-User
--proxy-client-cert-file=&lt;path to aggregator proxy cert&gt;
--proxy-client-key-file=&lt;path to aggregator proxy key&gt;
</code></pre><p>而这些配置的作用，主要就是为 Aggregator 这一层设置对应的 Key 和 Cert 文件。而这些文件的生成，就需要你自己手动完成了，具体流程请参考这篇<a href="https://github.com/kubernetes-incubator/apiserver-builder/blob/master/docs/concepts/auth.md">官方文档</a>。</p><p>Aggregator 功能开启之后，你只需要将 Metrics Server 的 YAML 文件部署起来，如下所示：</p><pre><code>$ git clone https://github.com/kubernetes-incubator/metrics-server
$ cd metrics-server
$ kubectl create -f deploy/1.8+/
</code></pre><p>接下来，你就会看到metrics.k8s.io这个API 出现在了你的 Kubernetes API 列表当中。</p><p>在理解了Prometheus 关心的三种监控数据源，以及 Kubernetes 的核心 Metrics 之后，作为用户，你其实要做的就是将 Prometheus Operator 在 Kubernetes 集群里部署起来。然后，按照本篇文章一开始介绍的架构，把上述Metrics 源配置起来，让 Prometheus 自己去进行采集即可。</p><p>在后续的文章中，我会为你进一步剖析 Kubernetes 监控体系以及自定义Metrics （自定义监控指标）的具体技术点。</p><h2>总结</h2><p>在本篇文章中，我主要为你介绍了 Kubernetes 当前监控体系的设计，介绍了 Prometheus 项目在这套体系中的地位，讲解了以 Prometheus 为核心的监控系统的架构设计。</p><p>然后，我为你详细地解读了 Kubernetes 核心监控数据的来源，即：Metrics Server的具体工作原理，以及 Aggregator APIServer 的设计思路。</p><p>通过以上讲述，我希望你能够对 Kubernetes 的监控体系形成一个整体的认知，体会到 Kubernetes 社区在监控这个事情上，全面以 Prometheus 项目为核心进行建设的大方向。</p><p>最后，在具体的监控指标规划上，我建议你<strong>遵循业界通用的 USE 原则和 RED 原则。</strong></p><p>其中，USE 原则指的是，按照如下三个维度来规划资源监控指标：</p><ol>
<li>
<p>利用率（Utilization），资源被有效利用起来提供服务的平均时间占比；</p>
</li>
<li>
<p>饱和度（Saturation），资源拥挤的程度，比如工作队列的长度；</p>
</li>
<li>
<p>错误率（Errors），错误的数量。</p>
</li>
</ol><p>而 RED 原则指的是，按照如下三个维度来规划服务监控指标：</p><ol>
<li>
<p>每秒请求数量（Rate）；</p>
</li>
<li>
<p>每秒错误数量（Errors）；</p>
</li>
<li>
<p>服务响应时间（Duration）。</p>
</li>
</ol><p>不难发现， USE 原则主要关注的是“资源”，比如节点和容器的资源使用情况，而 RED 原则主要关注的是“服务”，比如 kube-apiserver 或者某个应用的工作情况。这两种指标，在我今天为你讲解的 Kubernetes + Prometheus 组成的监控体系中，都是可以完全覆盖到的。</p><h2>思考题</h2><p>在监控体系中，对于数据的采集，其实既有 Prometheus 这种 Pull 模式，也有 Push 模式。请问，你如何看待这两种模式的异同和优缺点呢？</p><p>感谢你的收听，欢迎你给我留言，也欢迎分享给更多的朋友一起阅读。</p>
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
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83eo6TmyGF3wMIRLx3lPWOlBWusQCxyianFvZvWeW6hYCABLqEow3p7tGc6XgnqUPVvf6Cbj2KUYQIiag/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>孙健波</span>
  </div>
  <div class="_2_QraFYR_0">Pull和Push两种模式的区别非常有意思，Prometheus非常大胆的采用了pull的模式，但是仔细思考后就会觉得非常适合监控的场景。<br><br>Pull模式的特点<br>1. 被监控方提供一个server，并负责维护<br>2. 监控方控制采集频率<br><br>第一点其实对用户来说要求过高了，但是好处很多，比如pull 不到数据本身就说明了节点存在故障；又比如监控指标自然而言由用户自己维护，使得标准化很简单。<br>第二点其实更为重要，那就是监控系统对metric采集的统一和稳定有了可靠的保证，对于数据量大的情况下很重要。<br><br><br>缺点也很明显，用户不知道你什么时候来pull一下，数据维护多久更新也不好控制，容易造成一些信息的丢失和不准确。<br><br>当把这些优缺点权衡过后就会发现，纯监控的场景确实是适合pull的<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-22 12:59:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/84/c1/dfcad82a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Acter</span>
  </div>
  <div class="_2_QraFYR_0">1.老师能讲下prometheus的部署方式吗？比如helm部署，operator部署，过程中具体发生了什么？<br>2.能否解析下prometheus server配置文件中，jobs的写法，alert rule的写法？还有alertmanager的配置。感谢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 篇幅所限，prometheus 项目具体的玩法不进行详细展开了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-12 08:38:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/ab/c9/b1b48878.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Goswing</span>
  </div>
  <div class="_2_QraFYR_0">Push模式的优点是在正常情况下数据的延迟可以做到更低，也就是能更快的获取metrics并发现问题，而pull模式一般不会设定很短的轮询时间，所以延迟更高一些。<br><br>Push模式的缺点我能想到以下几个: <br>1 增加了服务的实现复杂度(比如推送错误处理等); <br>2 不利于平行扩展; <br>3 不支持自定义metrics采集策略，比如高峰期减少采集频率。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-13 08:12:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/69/4d/81c44f45.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>拉欧</span>
  </div>
  <div class="_2_QraFYR_0">pull是拉动作，监听者主动调用被监听者的接口<br>push是推动作，被监听者主动上报，监听者被动采集<br>拉动作有助于监听者自己控制频率和采样量，缺点是需要掌握所有被监听者的地址和端口，也就是要有注册中心；<br>推动作有利于被监听者自己控制上报数量和频率，但有可能对监听者构成额外的压力，同时有信息丢失的风险</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-30 16:51:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/74/4e/83d6ddce.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>CalvinXiao</span>
  </div>
  <div class="_2_QraFYR_0">push 方式针对没有 http 接口应用，例如 worker，可以设定每 5 秒上报处理了多少个 job，平均每个 job 耗时多少，有多少个错误等数据，需要配合 push gateway 使用。<br><br>当然，也可以自己写一个 exporter 通过查询日志来得到这些数据，然后用 pull 方式来获取。push 方式可以联想到 APM。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-13 02:22:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/9f/fc/0232f005.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我要收购腾讯</span>
  </div>
  <div class="_2_QraFYR_0">prometheus 的pull模式搭配自己的kubernetes SD, 加上prometheus-operator的service monitor的抽象，可以很大程度的简化配置的复杂程度</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-12 17:15:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/67/df/8b85d0d9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>--</span>
  </div>
  <div class="_2_QraFYR_0">核心监控数据是Prometheus通过Metric Server拉取的还是直接从kubelet拉取？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: metric server</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-24 15:17:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/1e/f9/bfb54326.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>狮锅艺</span>
  </div>
  <div class="_2_QraFYR_0">文中缺少的链接：https:&#47;&#47;github.com&#47;kubernetes&#47;kubernetes&#47;blob&#47;master&#47;cluster&#47;kube-up.sh</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-17 20:31:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/dd/79/2fde8492.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>李伟达</span>
  </div>
  <div class="_2_QraFYR_0">有一个问题，就是metric server是怎么注册给metrics.k8s.io&#47;v1beta1这个api的，或者说当client访问metrics.k8s.io&#47;v1beta1这个api时，aggregator如何知道转发给哪个后端？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-24 17:23:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/92/32/af5db71f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>kindule</span>
  </div>
  <div class="_2_QraFYR_0">你好，为什么node_exporter要单独分为一类而不是算作core metrics</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-12 10:26:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/21/41/37/b89f3d67.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我在睡觉</span>
  </div>
  <div class="_2_QraFYR_0">我认为拉取的最大好处是解耦，metric server对外提供标准接口，第三方实现自己的监控逻辑。还有部署的时候更方便，如果使用push的方式，那么部署一个监控组件之后需要用某种方式配置metric server，让他知道一个监控服务的存在以便通知上报数据，这样对于监控数据描述的配置文件都要配置到metric server（metric server也需要实现逻辑来处理这些配置），这显然增加了使用的复杂度，也不不够灵活。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-20 16:50:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLyfjHfjulibFGPTewSZZHm2M8yfI7BZmO9vLUFoagveCw3DWYDss7y1CecKia7lT5yb9KoAmsya2zg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Goteswille</span>
  </div>
  <div class="_2_QraFYR_0">坚持、打卡</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-17 11:31:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/37/3b/495e2ce6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>陈斯佳</span>
  </div>
  <div class="_2_QraFYR_0">第四十六课:Prometheus、Metrics<br><br>Prometheus 项目工作的核心，是使用 Pull （抓取）的方式去搜集被监控对象的 Metrics 数据（监控指标数据），然后，再把这些数据保存在一个 TSDB （时间序列数据库，比如 OpenTSDB、InfluxDB 等）当中，以便后续可以按照时间进行检索。<br><br>Metrics数据来源种类有:<br>第一种是宿主机的监控数据。最主要是借助Node Exporter，以DaemonSet的方式运行在宿主机上，收集包括节点负载（Load）,CPU,内存，磁盘以及网络这些常规信息。<br>第二种是来自于K8s的API Server,kubelet等组建的&#47;metrics API 。除了常规的 CPU、内存的信息外，这部分信息还主要包括了各个组件的核心监控指标。比如，对于 API Server 来说，它就会在 &#47;metrics API 里，暴露出各个 Controller 的工作队列（Work Queue）的长度、请求的 QPS 和延迟数据等等。这些信息，是检查 Kubernetes 本身工作情况的主要依据。<br>第三种 Metrics，是 Kubernetes 相关的监控数据。这部分数据，一般叫作 Kubernetes 核心监控数据（core metrics）。这其中包括了 Pod、Node、容器、Service 等主要 Kubernetes 核心概念的 Metrics。其中，容器相关的 Metrics 主要来自于 kubelet 内置的 cAdvisor 服务。在 kubelet 启动后，cAdvisor 服务也随之启动，而它能够提供的信息，可以细化到每一个容器的 CPU 、文件系统、内存、网络等资源的使用情况。<br><br>Metrics Server 在 Kubernetes 社区的定位，其实是用来取代 Heapster 这个项目的。有了 Metrics Server 之后，用户就可以通过标准的 Kubernetes API 来访问到这些监控数据了。Metrics Server 并不是 kube-apiserver 的一部分，而是通过 Aggregator 这种插件机制，在独立部署的情况下同 kube-apiserver 一起统一对外服务的。kube-aggregator 其实就是一个根据 URL 选择具体的 API 后端的代理服务器。<br><br>USE 原则指的是，按照如下三个维度来规划资源监控指标：<br>1. 利用率（Utilization），资源被有效利用起来提供服务的平均时间占比；<br>2. 饱和度（Saturation），资源拥挤的程度，比如工作队列的长度；<br>3. 错误率（Errors），错误的数量。<br><br>RED 原则指的是，按照如下三个维度来规划服务监控指标：<br>1. 每秒请求数量（Rate）；<br>2. 每秒错误数量（Errors）；<br>3. 服务响应时间（Duration）。<br><br>USE 原则主要关注的是“资源”，比如节点和容器的资源使用情况，而 RED 原则主要关注的是“服务”，比如 kube-apiserver 或者某个应用的工作情况。这两种指标，在我今天为你讲解的 Kubernetes + Prometheus 组成的监控体系中，都是可以完全覆盖到的。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-07 06:45:10</div>
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
  <div class="_2_QraFYR_0">老师好，请问在拉模式下，prometheus是如何「发现」采集点的呢？谢谢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-03 19:22:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/ef/a2/6ea5bb9e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>LEON</span>
  </div>
  <div class="_2_QraFYR_0">老师我一直对Prometheus和SNMP server有什么区别搞不明白。这俩是一个东西吗？他们有什么具体的区别？是不是可以理解为SNMP server是Prometheus的一个子集？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-12 09:23:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/ab/4e/82e9657c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jeff</span>
  </div>
  <div class="_2_QraFYR_0">太草草了事了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-26 16:40:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/24/33/bcf37f50.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>阿甘</span>
  </div>
  <div class="_2_QraFYR_0">Prometheus的时序数据存储依赖于第三方TSDB吗？目前这块没有像关系型数据库MySQL这样的王者啊。OpenTSDB性能非常一般，运维也比较复杂，InfluxDB社区版本只有单机版本。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-26 15:56:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_089889</span>
  </div>
  <div class="_2_QraFYR_0">Metric和链路trace 使用什么方案关联起来？为后续功能做分析支撑，</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-23 15:03:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/24/6d/9e/6a82a5ea.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>good boby</span>
  </div>
  <div class="_2_QraFYR_0">damonset对象可以方便提供prometheus的各个节点监控</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-16 22:16:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/RxZc4ZGUwpUp5grZqJEtjgWXTgMicjmQsQByACCAmuUibJ6NZsI90IYiaOwEejy3TiaxEb2BbByve1dicNnGRgicFtFw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_6a11b8</span>
  </div>
  <div class="_2_QraFYR_0">想问一下怎么向controller manager的metrics接口添加metric，使调用接口后，可以返回自定义metric</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-25 16:02:45</div>
  </div>
</div>
</div>
</li>
</ul>