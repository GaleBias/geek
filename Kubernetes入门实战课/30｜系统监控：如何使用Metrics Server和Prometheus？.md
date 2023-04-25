<audio title="30｜系统监控：如何使用Metrics Server和Prometheus？" src="https://static001.geekbang.org/resource/audio/3a/20/3a1bee9e607cd6cf0945392214602b20.mp3" controls="controls"></audio> 
<p>你好，我是Chrono。</p><p>在前面的两节课里，我们学习了对Pod和对集群的一些管理方法，其中的要点就是设置资源配额，让Kubernetes用户能公平合理地利用系统资源。</p><p>虽然有了这些方法，但距离我们把Pod和集群管好用好还缺少一个很重要的方面——集群的可观测性。也就是说，我们希望给集群也安装上“检查探针”，观察到集群的资源利用率和其他指标，让集群的整体运行状况对我们“透明可见”，这样才能更准确更方便地做好集群的运维工作。</p><p>但是观测集群是不能用“探针”这种简单的方式的，所以今天我就带你一起来看看Kubernetes为集群提供的两种系统级别的监控项目：Metrics Server和Prometheus，以及基于它们的水平自动伸缩对象HorizontalPodAutoscaler。</p><h2>Metrics Server</h2><p>如果你对Linux系统有所了解的话，也许知道有一个命令 <code>top</code> 能够实时显示当前系统的CPU和内存利用率，它是性能分析和调优的基本工具，非常有用。<strong>Kubernetes也提供了类似的命令，就是 <code>kubectl top</code>，不过默认情况下这个命令不会生效，必须要安装一个插件Metrics Server才可以。</strong></p><p>Metrics Server是一个专门用来收集Kubernetes核心资源指标（metrics）的工具，它定时从所有节点的kubelet里采集信息，但是对集群的整体性能影响极小，每个节点只大约会占用1m的CPU和2MB的内存，所以性价比非常高。</p><!-- [[[read_end]]] --><p>下面的<a href="https://kubernetes.io/zh-cn/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/#metrics-server">这张图</a>来自Kubernetes官网，你可以对Metrics Server的工作方式有个大概了解：它调用kubelet的API拿到节点和Pod的指标，再把这些信息交给apiserver，这样kubectl、HPA就可以利用apiserver来读取指标了：</p><p><img src="https://static001.geekbang.org/resource/image/8f/9e/8f4a22788c03b06377cabe791c67989e.png?wh=1562x572" alt="图片"></p><p>在Metrics Server的项目网址（<a href="https://github.com/kubernetes-sigs/metrics-server">https://github.com/kubernetes-sigs/metrics-server</a>）可以看到它的说明文档和安装步骤，不过如果你已经按照<a href="https://time.geekbang.org/column/article/534762">第17讲</a>用kubeadm搭建了Kubernetes集群，就已经具备了全部前提条件，接下来只需要几个简单的操作就可以完成安装。</p><p>Metrics Server的所有依赖都放在了一个YAML描述文件里，你可以使用wget或者curl下载：</p><pre><code class="language-plain">wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
</code></pre><p>但是在 <code>kubectl apply</code> 创建对象之前，我们还有两个准备工作要做。</p><p><strong>第一个工作，是修改YAML文件</strong>。你需要在Metrics Server的Deployment对象里，加上一个额外的运行参数 <code>--kubelet-insecure-tls</code>，也就是这样：</p><pre><code class="language-yaml">apiVersion: apps/v1
kind: Deployment
metadata:
&nbsp; name: metrics-server
&nbsp; namespace: kube-system
spec:
  ... ... 
&nbsp; template:
&nbsp; &nbsp; spec:
&nbsp; &nbsp; &nbsp; containers:
&nbsp; &nbsp; &nbsp; - args:
&nbsp; &nbsp; &nbsp; &nbsp; - --kubelet-insecure-tls
        ... ... 
</code></pre><p>这是因为Metrics Server默认使用TLS协议，要验证证书才能与kubelet实现安全通信，而我们的实验环境里没有这个必要，加上这个参数可以让我们的部署工作简单很多（生产环境里就要慎用）。</p><p><strong>第二个工作，是预先下载Metrics Server的镜像。</strong>看这个YAML文件，你会发现Metrics Server的镜像仓库用的是gcr.io，下载很困难。好在它也有国内的镜像网站，你可以用<a href="https://time.geekbang.org/column/article/534762">第17讲</a>里的办法，下载后再改名，然后把镜像加载到集群里的节点上。</p><p>这里我给出一段Shell脚本代码，供你参考：</p><pre><code class="language-plain">repo=registry.aliyuncs.com/google_containers

name=k8s.gcr.io/metrics-server/metrics-server:v0.6.1
src_name=metrics-server:v0.6.1

docker pull $repo/$src_name

docker tag $repo/$src_name $name
docker rmi $repo/$src_name
</code></pre><p>两个准备工作都完成之后，我们就可以使用YAML部署Metrics Server了：</p><pre><code class="language-plain">kubectl apply -f components.yaml
</code></pre><p>Metrics Server属于名字空间“kube-system”，可以用 <code>kubectl get pod</code> 加上 <code>-n</code> 参数查看它是否正常运行：</p><pre><code class="language-plain">kubectl get pod -n kube-system
</code></pre><p><img src="https://static001.geekbang.org/resource/image/b9/93/b93124cbc1b7d98b7c4f055f0723bf93.png?wh=1506x822" alt="图片"></p><p>现在有了Metrics Server插件，我们就可以使用命令 <code>kubectl top</code> 来查看Kubernetes集群当前的资源状态了。它有<strong>两个子命令，<code>node</code> 查看节点的资源使用率，<code>pod</code> 查看Pod的资源使用率</strong>。</p><p>由于Metrics Server收集信息需要时间，我们必须等一小会儿才能执行命令，查看集群里节点和Pod状态：</p><pre><code class="language-plain">kubectl top node
kubectl top pod -n kube-system
</code></pre><p><img src="https://static001.geekbang.org/resource/image/d4/61/d450b7e01f5f47ac56335f6c69707e61.png?wh=1800x1052" alt="图片"></p><p>从这个截图里你可以看到：</p><ul>
<li>集群里两个节点CPU使用率都不高，分别是8%和4%，但内存用的很多，master节点用了差不多一半（48%），而worker节点几乎用满了（89%）。</li>
<li>名字空间“kube-system”里有很多Pod，其中apiserver最消耗资源，使用了75m的CPU和363MB的内存。</li>
</ul><h2>HorizontalPodAutoscaler</h2><p>有了Metrics Server，我们就可以轻松地查看集群的资源使用状况了，不过它另外一个更重要的功能是辅助实现应用的“<strong>水平自动伸缩</strong>”。</p><p>在<a href="https://time.geekbang.org/column/article/535209">第18讲</a>里我们提到有一个命令 <code>kubectl scale</code>，可以任意增减Deployment部署的Pod数量，也就是水平方向的“扩容”和“缩容”。但是手动调整应用实例数量还是比较麻烦的，需要人工参与，也很难准确把握时机，难以及时应对生产环境中突发的大流量，所以最好能把这个“扩容”“缩容”也变成自动化的操作。</p><p>Kubernetes为此就定义了一个新的API对象，叫做“<strong>HorizontalPodAutoscaler</strong>”，简称是“<strong>hpa</strong>”。顾名思义，它是专门用来自动伸缩Pod数量的对象，适用于Deployment和StatefulSet，但不能用于DaemonSet（原因很明显吧）。</p><p>HorizontalPodAutoscaler的能力完全基于Metrics Server，它从Metrics Server获取当前应用的运行指标，主要是CPU使用率，再依据预定的策略增加或者减少Pod的数量。</p><p>下面我们就来看看该怎么使用HorizontalPodAutoscaler，首先要定义Deployment和Service，创建一个Nginx应用，作为自动伸缩的目标对象：</p><pre><code class="language-yaml">apiVersion: apps/v1
kind: Deployment
metadata:
&nbsp; name: ngx-hpa-dep

spec:
&nbsp; replicas: 1
&nbsp; selector:
&nbsp; &nbsp; matchLabels:
&nbsp; &nbsp; &nbsp; app: ngx-hpa-dep

&nbsp; template:
&nbsp; &nbsp; metadata:
&nbsp; &nbsp; &nbsp; labels:
&nbsp; &nbsp; &nbsp; &nbsp; app: ngx-hpa-dep
&nbsp; &nbsp; spec:
&nbsp; &nbsp; &nbsp; containers:
&nbsp; &nbsp; &nbsp; - image: nginx:alpine
&nbsp; &nbsp; &nbsp; &nbsp; name: nginx
&nbsp; &nbsp; &nbsp; &nbsp; ports:
&nbsp; &nbsp; &nbsp; &nbsp; - containerPort: 80

&nbsp; &nbsp; &nbsp; &nbsp; resources:
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; requests:
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; cpu: 50m
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; memory: 10Mi
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; limits:
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; cpu: 100m
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; memory: 20Mi
---

apiVersion: v1
kind: Service
metadata:
&nbsp; name: ngx-hpa-svc
spec:
&nbsp; ports:
&nbsp; - port: 80
&nbsp; &nbsp; protocol: TCP
&nbsp; &nbsp; targetPort: 80
&nbsp; selector:
&nbsp; &nbsp; app: ngx-hpa-dep
</code></pre><p>在这个YAML里我只部署了一个Nginx实例，名字是 <code>ngx-hpa-dep</code>。<strong>注意在它的</strong> <code>spec</code> <strong>里一定要用 <code>resources</code> 字段写清楚资源配额</strong>，否则HorizontalPodAutoscaler会无法获取Pod的指标，也就无法实现自动化扩缩容。</p><p>接下来我们要用命令 <code>kubectl autoscale</code> 创建一个HorizontalPodAutoscaler的样板YAML文件，它有三个参数：</p><ul>
<li>min，Pod数量的最小值，也就是缩容的下限。</li>
<li>max，Pod数量的最大值，也就是扩容的上限。</li>
<li>cpu-percent，CPU使用率指标，当大于这个值时扩容，小于这个值时缩容。</li>
</ul><p>好，现在我们就来为刚才的Nginx应用创建HorizontalPodAutoscaler，指定Pod数量最少2个，最多10个，CPU使用率指标设置的小一点，5%，方便我们观察扩容现象：</p><pre><code class="language-plain">export out="--dry-run=client -o yaml"              # 定义Shell变量
kubectl autoscale deploy ngx-hpa-dep --min=2 --max=10 --cpu-percent=5 $out
</code></pre><p>得到的YAML描述文件就是这样：</p><pre><code class="language-yaml">apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
&nbsp; name: ngx-hpa

spec:
&nbsp; maxReplicas: 10
&nbsp; minReplicas: 2
&nbsp; scaleTargetRef:
&nbsp; &nbsp; apiVersion: apps/v1
&nbsp; &nbsp; kind: Deployment
&nbsp; &nbsp; name: ngx-hpa-dep
&nbsp; targetCPUUtilizationPercentage: 5
</code></pre><p>我们再使用命令 <code>kubectl apply</code> 创建这个HorizontalPodAutoscaler后，它会发现Deployment里的实例只有1个，不符合min定义的下限的要求，就先扩容到2个：</p><p><img src="https://static001.geekbang.org/resource/image/3e/6c/3ec01a9746274ac28b10d612f1512a6c.png?wh=1630x704" alt="图片"></p><p>从这张截图里你可以看到，HorizontalPodAutoscaler会根据YAML里的描述，找到要管理的Deployment，把Pod数量调整成2个，再通过Metrics Server不断地监测Pod的CPU使用率。</p><p>下面我们来给Nginx加上压力流量，运行一个测试Pod，使用的镜像是“<strong>httpd:alpine</strong>”，它里面有HTTP性能测试工具ab（Apache Bench）：</p><pre><code class="language-plain">kubectl run test -it --image=httpd:alpine -- sh
</code></pre><p><img src="https://static001.geekbang.org/resource/image/d0/bd/d058182500cb83ac3e3c9cc01a42c9bd.png?wh=1896x354" alt="图片"></p><p>然后我们向Nginx发送一百万个请求，持续1分钟，再用 <code>kubectl get hpa</code> 来观察HorizontalPodAutoscaler的运行状况：</p><pre><code class="language-plain">ab -c 10 -t 60 -n 1000000 'http://ngx-hpa-svc/'
</code></pre><p><img src="https://static001.geekbang.org/resource/image/65/b4/6538ecd78118fabeb8d7c8f4fbabdbb4.png?wh=1920x794" alt="图片"></p><p>因为Metrics Server大约每15秒采集一次数据，所以HorizontalPodAutoscaler的自动化扩容和缩容也是按照这个时间点来逐步处理的。</p><p>当它发现目标的CPU使用率超过了预定的5%后，就会以2的倍数开始扩容，一直到数量上限，然后持续监控一段时间，如果CPU使用率回落，就会再缩容到最小值。</p><h2>Prometheus</h2><p>显然，有了Metrics Server和HorizontalPodAutoscaler的帮助，我们的应用管理工作又轻松了一些。不过，Metrics Server能够获取的指标还是太少了，只有CPU和内存，想要监控到更多更全面的应用运行状况，还得请出这方面的权威项目“<strong>Prometheus</strong>”。</p><p>其实，Prometheus的历史比Kubernetes还要早一些，它最初是由Google的离职员工在2012年创建的开源项目，灵感来源于Borg配套的BorgMon监控系统。后来在2016年，Prometheus作为第二个项目加入了CNCF，并在2018年继Kubernetes之后顺利毕业，成为了CNCF的不折不扣的“二当家”，也是云原生监控领域的“事实标准”。</p><p><img src="https://static001.geekbang.org/resource/image/69/58/69f4b76ca7323433cyy28574f1ee9358.png?wh=1200x600" alt="图片"></p><p>和Kubernetes一样，Prometheus也是一个庞大的系统，我们这里就只做一个简略的介绍。</p><p>下面的<a href="https://prometheus.io/docs/introduction/overview/">这张图</a>是Prometheus官方的架构图，几乎所有文章在讲Prometheus的时候必然要拿出来，所以我也没办法“免俗”：</p><p><img src="https://static001.geekbang.org/resource/image/e6/64/e62cebb3acc995246f203d698dfdc964.png?wh=1351x811" alt="图片"></p><p>Prometheus系统的核心是它的Server，里面有一个时序数据库TSDB，用来存储监控数据，另一个组件Retrieval使用拉取（Pull）的方式从各个目标收集数据，再通过HTTP Server把这些数据交给外界使用。</p><p>在Prometheus Server之外还有三个重要的组件：</p><ul>
<li>Push Gateway，用来适配一些特殊的监控目标，把默认的Pull模式转变为Push模式。</li>
<li>Alert Manager，告警中心，预先设定规则，发现问题时就通过邮件等方式告警。</li>
<li>Grafana是图形化界面，可以定制大量直观的监控仪表盘。</li>
</ul><p>由于同属于CNCF，所以Prometheus自然就是“云原生”，在Kubernetes里运行是顺理成章的事情。不过它包含的组件实在是太多，部署起来有点麻烦，这里我选用了“<strong>kube-prometheus</strong>”项目（<a href="https://github.com/prometheus-operator/kube-prometheus/">https://github.com/prometheus-operator/kube-prometheus/</a>），感觉操作起来比较容易些。</p><p>下面就跟着我来在Kubernetes实验环境里体验一下Prometheus吧。</p><p>我们先要下载kube-prometheus的源码包，当前的最新版本是0.11：</p><pre><code class="language-plain">wget https://github.com/prometheus-operator/kube-prometheus/archive/refs/tags/v0.11.0.tar.gz
</code></pre><p>解压缩后，Prometheus部署相关的YAML文件都在 <code>manifests</code> 目录里，有近100个，你可以先大概看一下。</p><p>和Metrics Server一样，我们也必须要做一些准备工作，才能够安装Prometheus。</p><p>第一步，是修改 <code>prometheus-service.yaml</code>、<code>grafana-service.yaml</code>。</p><p>这两个文件定义了Prometheus和Grafana服务对象，我们可以给它们添加 <code>type: NodePort</code>（参考<a href="https://time.geekbang.org/column/article/536829">第20讲</a>），这样就可以直接通过节点的IP地址访问（当然你也可以配置成Ingress）。</p><p><strong>第二步，是修改 <code>kubeStateMetrics-deployment.yaml</code>、<code>prometheusAdapter-deployment.yaml</code>，因为它们里面有两个存放在gcr.io的镜像，必须解决下载镜像的问题。</strong></p><p>但很遗憾，我没有在国内网站上找到它们的下载方式，为了能够顺利安装，只能把它们下载后再上传到Docker Hub上。所以你需要修改镜像名字，把前缀都改成 <code>chronolaw</code>：</p><pre><code class="language-plain">image: k8s.gcr.io/kube-state-metrics/kube-state-metrics:v2.5.0
image: k8s.gcr.io/prometheus-adapter/prometheus-adapter:v0.9.1

image: chronolaw/kube-state-metrics:v2.5.0
image: chronolaw/prometheus-adapter:v0.9.1
</code></pre><p>这两个准备工作完成之后，我们要执行两个 <code>kubectl create</code> 命令来部署Prometheus，先是 <code>manifests/setup</code> 目录，创建名字空间等基本对象，然后才是 <code>manifests</code> 目录：</p><pre><code class="language-plain">kubectl create -f manifests/setup
kubectl create -f manifests
</code></pre><p>Prometheus的对象都在名字空间“<strong>monitoring</strong>”里，创建之后可以用 <code>kubectl get</code> 来查看状态：</p><p><img src="https://static001.geekbang.org/resource/image/1b/09/1b4a1a1313ede9058b348c13a1020c09.png?wh=1894x878" alt="图片"></p><p>确定这些Pod都运行正常，我们再来看看它对外的服务端口：</p><pre><code class="language-plain">kubectl get svc -n monitoring
</code></pre><p><img src="https://static001.geekbang.org/resource/image/4c/59/4c423a203a688271d9d08b15a6782d59.png?wh=1920x531" alt="图片"></p><p>前面修改了Grafana和Prometheus的Service对象，所以这两个服务就在节点上开了端口，Grafana是“30358”，Prometheus有两个端口，其中“9090”对应的“30827”是Web端口。</p><p>在浏览器里输入节点的IP地址（我这里是“<a href="http://192.168.10.210">http://192.168.10.210</a>”），再加上端口号“30827”，我们就能看到Prometheus自带的Web界面，：</p><p><img src="https://static001.geekbang.org/resource/image/1b/dc/1b73040e258dfa8776c2a0a657a885dc.png?wh=1906x1934" alt="图片"></p><p>Web界面上有一个查询框，可以使用PromQL来查询指标，生成可视化图表，比如在这个截图里我就选择了“node_memory_Active_bytes”这个指标，意思是当前正在使用的内存容量。</p><p>Prometheus的Web界面比较简单，通常只用来调试、测试，不适合实际监控。我们再来看Grafana，访问节点的端口“30358”（我这里是“<a href="http://192.168.10.210:30358">http://192.168.10.210:30358</a>”），它会要求你先登录，默认的用户名和密码都是“<strong>admin</strong>”：</p><p><img src="https://static001.geekbang.org/resource/image/a2/31/a2614b09347b3436c317644374c36e31.png?wh=1906x1934" alt="图片"></p><p>Grafana内部已经预置了很多强大易用的仪表盘，你可以在左侧菜单栏的“Dashboards - Browse”里任意挑选一个：</p><p><img src="https://static001.geekbang.org/resource/image/23/5a/23ddb3db05e36c2da4a8f8067366f55a.png?wh=1906x1934" alt="图片"></p><p>比如我选择了“Kubernetes / Compute Resources / Namespace (Pods)”这个仪表盘，就会出来一个非常漂亮图表，比Metrics Server的 <code>kubectl top</code> 命令要好看得多，各种数据一目了然：</p><p><img src="https://static001.geekbang.org/resource/image/1f/bd/1f6ccc0b6d358c29419276fbf74e38bd.png?wh=1920x1696" alt="图片"></p><p>关于Prometheus就暂时介绍到这里，再往下讲可能就要偏离我们的Kubernetes主题了，如果你对它感兴趣的话，可以课后再去它的<a href="https://prometheus.io/">官网</a>上看文档，或者参考其他的学习资料。</p><h2>小结</h2><p>在云原生时代，系统的透明性和可观测性是非常重要的。今天我们一起学习了Kubernetes里的两个系统监控项目：命令行方式的Metrics Server、图形化界面的Prometheus，利用好它们就可以让我们随时掌握Kubernetes集群的运行状态，做到“明察秋毫”。</p><p>再简单小结一下今天的内容：</p><ol>
<li>Metrics Server是一个Kubernetes插件，能够收集系统的核心资源指标，相关的命令是 <code>kubectl top</code>。</li>
<li>Prometheus是云原生监控领域的“事实标准”，用PromQL语言来查询数据，配合Grafana可以展示直观的图形界面，方便监控。</li>
<li>HorizontalPodAutoscaler实现了应用的自动水平伸缩功能，它从Metrics Server获取应用的运行指标，再实时调整Pod数量，可以很好地应对突发流量。</li>
</ol><h2>课下作业</h2><p>最后是课下作业时间，给你留两个思考题：</p><ol>
<li>部署了HorizontalPodAutoscaler之后，如果再执行 <code>kubectl scale</code> 手动扩容会发生什么呢？</li>
<li>你有过应用监控的经验吗？应该关注哪些重要的指标呢？</li>
</ol><p>非常期待在留言区看到你的发言，同我同其他同学一起讨论。我们下节课再见。</p><p><img src="https://static001.geekbang.org/resource/image/ff/01/ff8b9d4fdcd5d227a58391f215761601.jpg?wh=1920x2985" alt="图片"></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/83/af/1cb42cd3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>马以</span>
  </div>
  <div class="_2_QraFYR_0">操作中遇到的问题以及解决办法<br>1: metris-server如果出现执行命令 kubectl top 不生效可以加如下配置<br>	apiVersion: apps&#47;v1<br>	kind: Deployment<br>	metadata:<br>	 ...<br>	  <br>	  template:<br>	    ....<br>	    spec:<br>	      nodeName: k8s-master #你自己的节点名称<br><br>2: prometheus 镜像问题<br>   这里我偷懒，不用骑驴找驴了，直接用老师的<br><br>   这里我建议您push到docker hub (因为集群有多个节点，push到docker hub上，这样pod调度到任意一个节点都可以方便下载)<br><br>   docker pull chronolaw&#47;kube-state-metrics:v2.5.0<br>   docker tag chronolaw&#47;kube-state-metrics:v2.5.0 k8s.gcr.io&#47;kube-state-metrics&#47;kube-state-metrics:v2.5.0<br>   docker rmi chronolaw&#47;kube-state-metrics:v2.5.0<br>   docker push k8s.gcr.io&#47;kube-state-metrics&#47;kube-state-metrics:v2.5.0<br><br><br>   prometheus-adapter 老师的版本运行不起来，我在docker hub上 找了一个可以用的<br><br>   docker pull pengyc2019&#47;prometheus-adapter:v0.9.1<br>   docker tag pengyc2019&#47;prometheus-adapter:v0.9.1 k8s.gcr.io&#47;prometheus-adapter&#47;prometheus-adapter:v0.9.1<br>   docker rmi pengyc2019&#47;prometheus-adapter:v0.9.1<br>   docker push k8s.gcr.io&#47;prometheus-adapter&#47;prometheus-adapter:v0.9.1<br><br>然后执行：<br>	kubectl create -f manifests&#47;setup<br>	kubectl create -f manifests<br><br> 到此，运行成功<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢分享。<br><br>不知道为什么prometheus-adapter:v0.9.1有问题，可能我上传的是arm64版本的，有空再检查一下。<br><br>下载看了看，sha256是一样的，不知道哪里有问题。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-06 01:14:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/25/87/f3a69d1b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>peter</span>
  </div>
  <div class="_2_QraFYR_0">请教老师几个问题：<br>Q1：Grafana 不是Prometheus的组件吧<br>架构图中标红色的是属于Prometheus，在UI方面，架构图中的”prometheus web ui”应该是Prometheus的组件吧。Grafana好像是一个公共组件，不是Prometheus独有的。<br><br>Q2: Prometheus部署后有四个POD的状态不是RUNNING<br>blackbox-exporter-746c64fd88-ts299：状态是ImagePullBackOff<br>prometheus-adapter-8547d6666f-6npn6：状态是CrashLoopBackOff，<br>错误信息：<br>sc = Get &quot;https:&#47;&#47;registry-1.docker.io&#47;v2&#47;jimmidyson&#47;configmap-reload&#47;manifests&#47;sha256:91467ba755a0c41199a63fe80a2c321c06edc4d3affb4f0ab6b3d20a49ed88d1&quot;: net&#47;http: TLS handshake timeout<br>好像和TLS有关，需要和Metrics Server一样加“kubelet-insecure-tls”吗？在哪个YAML文件中修改？<br><br>Q3：Prometheus部署的逆向操作是什么？<br>用这两个命令来部署：<br>kubectl create -f manifests&#47;setup<br>kubectl create -f manifests<br><br>部署后，如果想重新部署，需要清理环境，那么，怎么清理掉以前部署的东西？<br>和kubectl create -f manifests&#47;setup相反的操作是“kubectl delete -f manifests&#47;setup”吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1. 是的。Grafana是独立的项目，图省事就和Prometheus在一起说了，可能造成了误解，抱歉。<br><br>2.镜像拉取失败和tls没关系，可以自己用docker pull命令再尝试一下。<br><br>3.create 换成delete，先是manifests，然后是setup，你的理解正确。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-31 17:17:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>邵涵</span>
  </div>
  <div class="_2_QraFYR_0">在使用hpa做自动扩容&#47;缩容时，遇到了只扩容不缩容的问题，具体情况如下：<br>1. 按文中的步骤，使用ab加压，可以看到pod增加到了10个<br>[shaohan@k8s4 ~]$ kubectl get hpa ngx-hpa -w<br>NAME      REFERENCE                TARGETS   MINPODS   MAXPODS   REPLICAS   AGE<br>ngx-hpa   Deployment&#47;ngx-hpa-dep   150%&#47;5%   2         10        2          90s<br>ngx-hpa   Deployment&#47;ngx-hpa-dep   129%&#47;5%   2         10        4          105s<br>ngx-hpa   Deployment&#47;ngx-hpa-dep   93%&#47;5%    2         10        8          2m<br>ngx-hpa   Deployment&#47;ngx-hpa-dep   57%&#47;5%    2         10        10         2m15s<br>2. 在ab停止运行之后过了几分钟，kubectl get hpa ngx-hpa -w的输出中才出现了一行新数据，如下<br>ngx-hpa   Deployment&#47;ngx-hpa-dep   0%&#47;5%     2         10        10         7m31s<br>仍然是10个pod，并没有自动缩容<br>3. 查看pod<br>[shaohan@k8s4 ~]$ kubectl get pod<br>NAME                           READY   STATUS    RESTARTS     AGE<br>ngx-hpa-dep-86f66c75f5-d82rh   0&#47;1     Pending   0            63s<br>ngx-hpa-dep-86f66c75f5-dtc88   0&#47;1     Pending   0            63s<br>（其他ngx-hpa-dep-xxx省略，因为评论字数限制……）<br>test                           1&#47;1     Running   1 (6s ago)   2m5s<br>可以看到，有两个pod是pending状态的，kubectl describe这两个pending pod中的一个，可以看到<br>Warning  FailedScheduling  18s (x2 over 85s)  default-scheduler  0&#47;2 nodes are available: 1 Insufficient cpu, 1 node(s) had taint {node-role.kubernetes.io&#47;master: }, that the pod didn&#39;t tolerate.<br>是因为worker节点资源不足，master节点没有开放给pod调度，造成扩容时有两个pod虽然创建了，但一直在等待资源而没有进入running状态<br>4. 手动删除了用于执行ab的apache pod，释放资源，然后立刻查看pod，就只有两个了<br>kubectl get hpa ngx-hpa -w的输出中也出现了一行新数据<br>ngx-hpa   Deployment&#47;ngx-hpa-dep   0%&#47;5%     2         10        2          7m46s<br>自动缩容成功执行了<br><br>所以，看起来，hpa是“严格按顺序执行的”，它按照规则设定的条件，要扩容到10个pod，在10个pod全都running之前，即使已经符合缩容的条件了，它也不执行缩容，而是要等到之前扩容的操作彻底完成，也就是10个pod全都running了，才会执行缩容</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-01 16:15:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/0b/9a/8ff51c91.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ningfei</span>
  </div>
  <div class="_2_QraFYR_0">prometheus-adapter里使用这个willdockerhub&#47;prometheus-adapter:v0.9.1镜像,可以启动成功</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-02 01:40:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/9a/a6/29ac6f6a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>XXG</span>
  </div>
  <div class="_2_QraFYR_0">metrics-server-85bc76798b-hr56n           0&#47;1     ImagePullBackOff 原因：<br><br>&lt;1&gt; 注意metrics-server版本，我拉下来的yml文件版本变成了v0.6.2，所以要根据最新的components.yaml文件中的metrics-server版本对应改一下老师的脚本；<br>&lt;2&gt; 注意要在Worker节点执行脚本，我就在master节点上执行了好几遍。。。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good，多实践，集群管理和本机管理的差距较大，很多时候会弄混。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-30 22:48:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/99/87/98ebb20e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>dao</span>
  </div>
  <div class="_2_QraFYR_0">简单的回答一下思考题，<br><br>1，会根据设置进行扩容（scale out），但是如果不满足 HPA 的指标条件，接着会立即进行缩容（scale in），下面是我的操作观察到的日志<br>---<br>Normal  SuccessfulCreate  3s    replicaset-controller  Created pod: ngx-hpa-dep-86f66c75f5-z2gjk<br>Normal  SuccessfulCreate  3s    replicaset-controller  Created pod: ngx-hpa-dep-86f66c75f5-p46lr<br>Normal  SuccessfulDelete  3s    replicaset-controller  Deleted pod: ngx-hpa-dep-86f66c75f5-z2gjk<br>Normal  SuccessfulDelete  3s    replicaset-controller  Deleted pod: ngx-hpa-dep-86f66c75f5-p46lr<br>---<br><br>2，我现有的经验很有限，主要集中在单机&#47;集群机器指标的监控，以及对于应用及产品的监控（通过监控日志实现的），同时结合一些告警系统（alert），以便快速发现故障及解决。<br>关注的指标有 CPU、内存、网卡、磁盘读写，应用的性能监控和错误率（可用性）监控。性能监控指标主要是响应时间（RT）、各种吞吐量（比如QPS）等。错误率指标主要是指应用错误及异常、HTTP错误响应代码等。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: awesome!</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-02 12:03:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/56/2f/4518f8e1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>放不下荣华富贵</span>
  </div>
  <div class="_2_QraFYR_0">自动扩容的话，通常生产环境常用的指标是什么呢？<br><br>文中采用cpu占用率，比如md5sum是非常偏向于cpu运算的工作，但扩容可能并不能达到预期的效果（工作占满了一个核但是再多一个核却不能分摊这个工作）。<br>所以看起来系统负载而非占用率应该是更好的扩容指标，不知道我这么理解是否正确？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 还可以基于其他的一些自定义指标，Kubernetes在这里提供了一个很好的可扩展的框架。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-01 22:41:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/ba/02/78e0d4ac.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>运维赵宏业</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，请教下，多k8s集群的场景下，每个集群内都要部署一套kube-prometheus吗？感谢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个不是太确定，可能要对运维比较熟悉的同学来解答了，我理解应该是这样。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-16 16:00:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/45/a9/3d48d6a2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Lorry</span>
  </div>
  <div class="_2_QraFYR_0">老师，有没有办法能够让k8s访问宿主机网络？物理机安装prometus+grafana比较麻烦，如果能够直接通过docker&#47;k8s装好就方便很多；但是现在生产&#47;测试环境都是物理机环境，所以想要反向通过k8s的应用监控宿主机所在网络的服务。<br><br>应该怎么配置呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以的，有个hostNetwork属性，查一下资料，我们的课程里也用过一次。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-05 08:36:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/99/87/98ebb20e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>dao</span>
  </div>
  <div class="_2_QraFYR_0">分享一下我的本课实践经验。当所有部署结束后，检查发现有 Pod 无法正常运行（也就是几个 Web UI 无法正常打开的）<br><br>```bash<br># 查看 Pod 状态，发现 prometheus-operator 无法正常启动<br>kubectl get pod -n monitoring<br>NAME                                  READY   STATUS    RESTARTS   AGE<br>blackbox-exporter-746c64fd88-wmnd5    3&#47;3     Running   0          68s<br>grafana-5fc7f9f55d-qqt77              1&#47;1     Running   0          68s<br>kube-state-metrics-6c8846558c-w66x9   3&#47;3     Running   0          67s<br>node-exporter-68l4p                   2&#47;2     Running   0          67s<br>node-exporter-vs55z                   2&#47;2     Running   0          67s<br>prometheus-adapter-6455646bdc-4lz5l   1&#47;1     Running   0          66s<br>prometheus-adapter-6455646bdc-gxbrl   1&#47;1     Running   0          66s<br>prometheus-operator-f59c8b954-btxtw   0&#47;2     Pending   0          66s<br><br># 调查 Pod prometheus-operator<br>kubectl describe pod prometheus-operator-f59c8b954-btxtw -n monitoring<br>Events:<br>Type     Reason            Age   From               Message<br>----     ------            ----  ----               -------<br>Warning  FailedScheduling  82s   default-scheduler  0&#47;2 nodes are available: 1 Insufficient memory, 1 node(s) had taint {node-role.kubernetes.io&#47;master: }, that the pod didn&#39;t tolerate.<br>Warning  FailedScheduling  20s   default-scheduler  0&#47;2 nodes are available: 1 Insufficient memory, 1 node(s) had taint {node-role.kubernetes.io&#47;master: }, that the pod didn&#39;t tolerate.<br><br># 去除 node 的污点 node-role.kubernetes.io&#47;master<br>kubectl taint nodes --all node-role.kubernetes.io&#47;master-<br>```</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great!</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-02 12:09:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/b6/88/e8deccbc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>romance</span>
  </div>
  <div class="_2_QraFYR_0">metrics-server成功部署了，kubectl get pod -n kube-system 查看正常运行了，但用kubectl top node 时显示  Error from server (ServiceUnavailable): the server is currently unable to handle the request (get nodes.metrics.k8s.io)</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以再用logs、describe看具体的状态，有问题就重启pod。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-31 15:41:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/d0/51/f1c9ae2d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Christopher</span>
  </div>
  <div class="_2_QraFYR_0">prometheus-adapter的pod一直是CrashLoopBackOff，日志显示exec &#47;adapter: exec format error是啥情况？grafana和Prometheus的web端倒是可以正常打开正常显示的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 用describe命令看一下它的状态，看样子像是镜像不对，是不是amd64&#47;arm64的问题，可以用docker inspect看看。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-31 09:54:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/54/ad/6ee2b7cb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Jacob.C</span>
  </div>
  <div class="_2_QraFYR_0">请问装普罗米修斯之前必须装 mertrics 吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这两个没有直接关系，完全可以独立安装。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-15 11:50:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/45/a9/3d48d6a2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Lorry</span>
  </div>
  <div class="_2_QraFYR_0">分享一点：拉去metric-server镜像的时候，一定要确保k8s集群所有的节点都通过docker进行拉取（执行文中提供的拉取脚本）。<br>我碰到的问题是只是在master拉去来的，但是任务打到work节点后，总是出现拉去失败的异常。<br><br>kubectl decribe在定位问题的时候，真香</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: nice</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-06 07:48:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/24/c4/51/5bca1604.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>aLong</span>
  </div>
  <div class="_2_QraFYR_0">alertmanager-main-2,prometheus-k8s-0&amp;1, 这三个都会提示污点问题导致pending。去除掉污点会正常启动。 这是否是常规操作？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 正常，应该是实验集群太小，没有足够的worker node来运行它们。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-19 17:44:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2c/9a/7b/aa937172.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>LiL</span>
  </div>
  <div class="_2_QraFYR_0">按照老师的操作，正常安装promethus和grafana,但是grafana界面打不开，提示如下，还请老师解答下呢，网上搜了一圈也没发现可行的解决方案：<br>If you&#39;re seeing this Grafana has failed to load its application files<br><br>1. This could be caused by your reverse proxy settings.<br><br>2. If you host grafana under subpath make sure your grafana.ini root_url setting includes subpath. If not using a reverse proxy make sure to set serve_from_sub_path to true.<br><br>3. If you have a local dev build make sure you build frontend using: yarn start, yarn start:hot, or yarn build<br><br>4. Sometimes restarting grafana-server can help<br><br>5. Check if you are using a non-supported browser. For more information, refer to the list of supported browsers.</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 没遇到这样的问题，先看看Prometheus的版本是否与课程里的一致，再参考一下后面的视频，是否有哪步错了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-01 21:29:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/28/fb/8a/3e8f42f1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ybc</span>
  </div>
  <div class="_2_QraFYR_0">遇到个问题想请教下老师和同学们： 在prometheus、grafana部署成功后，我采用了ingress的方式来进行访问，但发现prometheus的web可以正常访问，但grafana的界面访问不到，查看日志，访问成功到了ingress controller的pod，但没能转发到grafana的pod，查证ingress的转发规则没问题，试着在ingress controller的pod中ping grafana节点的ip，发现ping不通，发现同一node的某些pod可以ping通，有些则不行。 不知道大家有没有知道原因或排查思路的？<br><br>贴下我的ingress规则：<br>  rules:<br>  - host: prome.test<br>    http:<br>      paths:<br>      - path: &#47;<br>        pathType: Prefix<br>        backend:<br>          service:<br>            name: prometheus-k8s<br>            port:<br>              number: 9090<br>  - host: grafana.test<br>    http:<br>      paths:<br>      - path: &#47;<br>        pathType: Prefix<br>        backend:<br>          service:<br>            name: grafana<br>            port:<br>              number: 3000<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 看Ingress应该是没问题，可以再看看Nginx的日志，有没有什么信息。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-09 03:21:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>邵涵</span>
  </div>
  <div class="_2_QraFYR_0">安装metrics server之后，kubectl top执行失败，如下<br>[shaohan@k8s4 ~]$ kubectl top node<br>Error from server (ServiceUnavailable): the server is currently unable to handle the request (get nodes.metrics.k8s.io)<br><br>执行kubectl get apiservice v1beta1.metrics.k8s.io -o yaml看到<br>status:<br>  conditions:<br>  - lastTransitionTime: &quot;2022-10-29T09:57:08Z&quot;<br>    message: &#39;failing or missing response from https:&#47;&#47;10.98.115.140:443&#47;apis&#47;metrics.k8s.io&#47;v1beta1:<br>      Get &quot;https:&#47;&#47;10.98.115.140:443&#47;apis&#47;metrics.k8s.io&#47;v1beta1&quot;: dial tcp 10.98.115.140:443:<br>      i&#47;o timeout&#39;<br>    reason: FailedDiscoveryCheck<br>    status: &quot;False&quot;<br>    type: Available<br>在components.yaml中给deployment对象添加hostNetwork: true（在spec-&gt;template-&gt;spec下）就可以解决这个问题</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感觉这个不是太好的办法，尽量不要用hostNetwork: true。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-01 17:00:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>邵涵</span>
  </div>
  <div class="_2_QraFYR_0">遇到了一个grafana的问题：仪表盘，比如文中展示的“Kubernetes &#47; Compute Resources &#47; Namespace (Pods)”，页面中图形报表没有数据展示，数字部分，比如资源使用率百分比，展示的是NO DATA<br><br>这个看起来是grafana调用prometheus获取数据失败了<br>安装的grafana中的默认数据源中配置的prometheus url是http:&#47;&#47;prometheus-k8s.monitoring.svc:9090，service对象prometheus-k8s的域名“prometheus-k8s.monitoring.svc”此时在集群中访问失败（通过kubectl exec进入某测试用的pod去访问该域名，得到的是Could not resolve host: prometheus-k8s.monitoring.svc），此时，在default namespace下创建新的deployment+service，新的service对象的域名也是一样访问失败的<br>重启coredns也不起作用，是在重启了k8s集群的虚拟机几次之后，这个问题才自己好了，service的域名都可以访问了，grafana中的仪表盘页面也有数据了<br><br>另，在发生上述问题时，prometheus的alertmanager pod也是启动失败的<br>[shaohan@k8s4 ~]$ kubectl get pod -n monitoring<br>NAME                                  READY   STATUS    RESTARTS     AGE<br>alertmanager-main-0                   1&#47;2     Running   2 (9s ago)   3m29s<br>alertmanager-main-1                   1&#47;2     Running   2 (9s ago)   3m29s<br>alertmanager-main-2                   1&#47;2     Running   2 (9s ago)   3m29s<br>通过kubectl describe可以看到<br>  Warning  Unhealthy  4m34s (x6 over 5m24s)  kubelet            Liveness probe failed: Get &quot;http:&#47;&#47;10.10.1.12:9093&#47;-&#47;healthy&quot;: dial tcp 10.10.1.12:9093: connect: connection refused<br>  Warning  Unhealthy  30s (x68 over 5m27s)   kubelet            Readiness probe failed: Get &quot;http:&#47;&#47;10.10.1.12:9093&#47;-&#47;ready&quot;: dial tcp 10.10.1.12:9093: connect: connection refused<br>也是网络问题，当上边提到的service域名问题经过虚拟机重启解决了之后，这个alertmanager pod探针失败的问题，也同时消失了，pod成功运行</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 看样子像是coredns的问题，看看它的状态。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-01 16:37:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/50/87/dde718fa.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>alexgreenbar</span>
  </div>
  <div class="_2_QraFYR_0">kubectl create -f manifests&#47;setup<br><br>用create没有问题，但用apply会报错，之前一直用apply，这里是有什么区别吗？<br><br>**********<br>kubectl apply -f manifests&#47;setup<br>customresourcedefinition.apiextensions.k8s.io&#47;alertmanagerconfigs.monitoring.coreos.com created<br>customresourcedefinition.apiextensions.k8s.io&#47;alertmanagers.monitoring.coreos.com created<br>customresourcedefinition.apiextensions.k8s.io&#47;podmonitors.monitoring.coreos.com created<br>customresourcedefinition.apiextensions.k8s.io&#47;probes.monitoring.coreos.com created<br>customresourcedefinition.apiextensions.k8s.io&#47;prometheusrules.monitoring.coreos.com created<br>customresourcedefinition.apiextensions.k8s.io&#47;servicemonitors.monitoring.coreos.com created<br>customresourcedefinition.apiextensions.k8s.io&#47;thanosrulers.monitoring.coreos.com created<br>namespace&#47;monitoring created<br>The CustomResourceDefinition &quot;prometheuses.monitoring.coreos.com&quot; is invalid: metadata.annotations: Too long: must have at most 262144 bytes<br>**********</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，这里必须用create，不能用apply。<br><br>一般来说create和apply可以互换，但Prometheus有点特殊，具体的原因还没有深究。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-01 16:33:05</div>
  </div>
</div>
</div>
</li>
</ul>