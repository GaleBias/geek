<audio title="20｜Service：微服务架构的应对之道" src="https://static001.geekbang.org/resource/audio/17/b2/178f82fca6e1778f90d8e31a25e9c2b2.mp3" controls="controls"></audio> 
<p>你好，我是Chrono。</p><p>在前面的课里我们学习了Deployment和DaemonSet这两个API对象，它们都是在线业务，只是以不同的策略部署应用，Deployment创建任意多个实例，Daemon为每个节点创建一个实例。</p><p>这两个API对象可以部署多种形式的应用，而在云原生时代，微服务无疑是应用的主流形态。为了更好地支持微服务以及服务网格这样的应用架构，Kubernetes又专门定义了一个新的对象：Service，它是集群内部的负载均衡机制，用来解决服务发现的关键问题。</p><p>今天我们就来看看什么是Service、如何使用YAML来定义Service，以及如何在Kubernetes里用好Service。</p><h2>为什么要有Service</h2><p>有了Deployment和DaemonSet，我们在集群里发布应用程序的工作轻松了很多。借助Kubernetes强大的自动化运维能力，我们可以把应用的更新上线频率由以前的月、周级别提升到天、小时级别，让服务质量更上一层楼。</p><p>不过，在应用程序快速版本迭代的同时，另一个问题也逐渐显现出来了，就是“<strong>服务发现</strong>”。</p><p>在Kubernetes集群里Pod的生命周期是比较“短暂”的，虽然Deployment和DaemonSet可以维持Pod总体数量的稳定，但在运行过程中，难免会有Pod销毁又重建，这就会导致Pod集合处于动态的变化之中。</p><!-- [[[read_end]]] --><p>这种“动态稳定”对于现在流行的微服务架构来说是非常致命的，试想一下，后台Pod的IP地址老是变来变去，客户端该怎么访问呢？如果不处理好这个问题，Deployment和DaemonSet把Pod管理得再完善也是没有价值的。</p><p>其实，这个问题也并不是什么难事，业内早就有解决方案来针对这样“不稳定”的后端服务，那就是“<strong>负载均衡</strong>”，典型的应用有LVS、Nginx等等。它们在前端与后端之间加入了一个“中间层”，屏蔽后端的变化，为前端提供一个稳定的服务。</p><p>但LVS、Nginx毕竟不是云原生技术，所以Kubernetes就按照这个思路，定义了新的API对象：<strong>Service</strong>。</p><p>所以估计你也能想到，Service的工作原理和LVS、Nginx差不多，Kubernetes会给它分配一个静态IP地址，然后它再去自动管理、维护后面动态变化的Pod集合，当客户端访问Service，它就根据某种策略，把流量转发给后面的某个Pod。</p><p>下面的这张图来自Kubernetes<a href="https://kubernetes.io/zh/docs/concepts/services-networking/service/">官网文档</a>，比较清楚地展示了Service的工作原理：</p><p><img src="https://static001.geekbang.org/resource/image/03/74/0347a0b3bae55fb9ef6c07469e964b74.png?wh=1622x1214" alt="图片"></p><p>你可以看到，这里Service使用了iptables技术，每个节点上的kube-proxy组件自动维护iptables规则，客户不再关心Pod的具体地址，只要访问Service的固定IP地址，Service就会根据iptables规则转发请求给它管理的多个Pod，是典型的负载均衡架构。</p><p>不过Service并不是只能使用iptables来实现负载均衡，它还有另外两种实现技术：性能更差的userspace和性能更好的ipvs，但这些都属于底层细节，我们不需要刻意关注。</p><h2>如何使用YAML描述Service</h2><p>知道了Service的基本工作原理，我们来看看怎么为Service编写YAML描述文件。</p><p>照例我们还是可以用命令 <code>kubectl api-resources</code> 查看它的基本信息，可以知道它的简称是<code>svc</code>，apiVersion是 <code>v1</code>。<strong>注意，这说明它与Pod一样，属于Kubernetes的核心对象，不关联业务应用，与Job、Deployment是不同的。</strong></p><p>现在，相信你很容易写出Service的YAML文件头了吧：</p><pre><code class="language-yaml">apiVersion: v1
kind: Service
metadata:
&nbsp; name: xxx-svc
</code></pre><p>同样的，能否让Kubernetes为我们自动创建Service的YAML样板呢？还是使用命令 <code>kubectl create</code> 吗？</p><p>这里Kubernetes又表现出了行为上的不一致。<strong>虽然它可以自动创建YAML样板，但不是用命令</strong> <code>kubectl create</code><strong>，而是另外一个命令</strong> <code>kubectl expose</code>，也许Kubernetes认为“expose”能够更好地表达Service“暴露”服务地址的意思吧。</p><p>因为在Kubernetes里提供服务的是Pod，而Pod又可以用Deployment/DaemonSet对象来部署，所以 <code>kubectl expose</code>  支持从多种对象创建服务，Pod、Deployment、DaemonSet都可以。</p><p>使用 <code>kubectl expose</code> 指令时还需要用参数 <code>--port</code> 和 <code>--target-port</code> 分别指定映射端口和容器端口，而Service自己的IP地址和后端Pod的IP地址可以自动生成，用法上和Docker的命令行参数 <code>-p</code> 很类似，只是略微麻烦一点。</p><p>比如，如果我们要为<a href="https://time.geekbang.org/column/article/535209">第18讲</a>里的ngx-dep对象生成Service，命令就要这么写：</p><pre><code class="language-plain">export out="--dry-run=client -o yaml"
kubectl expose deploy ngx-dep --port=80 --target-port=80 $out
</code></pre><p>生成的Service YAML大概是这样的：</p><pre><code class="language-yaml">apiVersion: v1
kind: Service
metadata:
&nbsp; name: ngx-svc
&nbsp;&nbsp;
spec:
&nbsp; selector:
&nbsp; &nbsp; app: ngx-dep
&nbsp; &nbsp;&nbsp;
&nbsp; ports:
&nbsp; - port: 80
&nbsp; &nbsp; targetPort: 80
&nbsp; &nbsp; protocol: TCP
</code></pre><p>你会发现，Service的定义非常简单，在“spec”里只有两个关键字段，<code>selector</code> 和 <code>ports</code>。</p><p><code>selector</code> 和Deployment/DaemonSet里的作用是一样的，用来过滤出要代理的那些Pod。因为我们指定要代理Deployment，所以Kubernetes就为我们自动填上了ngx-dep的标签，会选择这个Deployment对象部署的所有Pod。</p><p>从这里你也可以看到，Kubernetes的这个标签机制虽然很简单，却非常强大有效，很轻松就关联上了Deployment的Pod。</p><p><code>ports</code> 就很好理解了，里面的三个字段分别表示外部端口、内部端口和使用的协议，在这里就是内外部都使用80端口，协议是TCP。</p><p>当然，你在这里也可以把 <code>ports</code> 改成“8080”等其他的端口，这样外部服务看到的就是Service给出的端口，而不会知道Pod的真正服务端口。</p><p>为了让你看清楚Service与它引用的Pod的关系，我把这两个YAML对象画在了下面的这张图里，需要重点关注的是 <code>selector</code>、<code>targetPort</code> 与Pod的关联：</p><p><img src="https://static001.geekbang.org/resource/image/0f/64/0f74ae3a71a6a661376698e481903d64.jpg?wh=1920x1322" alt="图片"></p><h2>如何在Kubernetes里使用Service</h2><p>在使用YAML创建Service对象之前，让我们先对第18讲里的Deployment做一点改造，方便观察Service的效果。</p><p>首先，我们创建一个ConfigMap，定义一个Nginx的配置片段，它会输出服务器的地址、主机名、请求的URI等基本信息：</p><pre><code class="language-yaml">apiVersion: v1
kind: ConfigMap
metadata:
&nbsp; name: ngx-conf

data:
&nbsp; default.conf: |
&nbsp; &nbsp; server {
&nbsp; &nbsp; &nbsp; listen 80;
&nbsp; &nbsp; &nbsp; location / {
&nbsp; &nbsp; &nbsp; &nbsp; default_type text/plain;
&nbsp; &nbsp; &nbsp; &nbsp; return 200
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; 'srv : $server_addr:$server_port\nhost: $hostname\nuri : $request_method $host $request_uri\ndate: $time_iso8601\n';
&nbsp; &nbsp; &nbsp; }
&nbsp; &nbsp; }
</code></pre><p>然后我们在Deployment的“<strong>template.volumes</strong>”里定义存储卷，再用“<strong>volumeMounts</strong>”把配置文件加载进Nginx容器里：</p><pre><code class="language-yaml">apiVersion: apps/v1
kind: Deployment
metadata:
&nbsp; name: ngx-dep

spec:
&nbsp; replicas: 2
&nbsp; selector:
&nbsp; &nbsp; matchLabels:
&nbsp; &nbsp; &nbsp; app: ngx-dep

&nbsp; template:
&nbsp; &nbsp; metadata:
&nbsp; &nbsp; &nbsp; labels:
&nbsp; &nbsp; &nbsp; &nbsp; app: ngx-dep
&nbsp; &nbsp; spec:
&nbsp; &nbsp; &nbsp; volumes:
&nbsp; &nbsp; &nbsp; - name: ngx-conf-vol
&nbsp; &nbsp; &nbsp; &nbsp; configMap:
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; name: ngx-conf

&nbsp; &nbsp; &nbsp; containers:
&nbsp; &nbsp; &nbsp; - image: nginx:alpine
&nbsp; &nbsp; &nbsp; &nbsp; name: nginx
&nbsp; &nbsp; &nbsp; &nbsp; ports:
&nbsp; &nbsp; &nbsp; &nbsp; - containerPort: 80

&nbsp; &nbsp; &nbsp; &nbsp; volumeMounts:
&nbsp; &nbsp; &nbsp; &nbsp; - mountPath: /etc/nginx/conf.d
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; name: ngx-conf-vol
</code></pre><p>这两处修改用到了<a href="https://time.geekbang.org/column/article/533395">第14讲</a>里的知识，如果你还没有熟练掌握，可以回去复习一下。</p><p>部署这个Deployment之后，我们就可以创建Service对象了，用的还是 <code>kubectl apply</code>：</p><pre><code class="language-plain">kubectl apply -f svc.yml
</code></pre><p>创建之后，用命令 <code>kubectl get</code> 就可以看到它的状态：</p><p><img src="https://static001.geekbang.org/resource/image/c3/e1/c3502c6c00d870eyy506351e2ba828e1.png?wh=1844x362" alt="图片"></p><p>你可以看到，Kubernetes为Service对象自动分配了一个IP地址“10.96.240.115”，这个地址段是独立于Pod地址段的（比如第17讲里的10.10.xx.xx）。而且Service对象的IP地址还有一个特点，它是一个“<strong>虚地址</strong>”，不存在实体，只能用来转发流量。</p><p>想要看Service代理了哪些后端的Pod，你可以用 <code>kubectl describe</code> 命令：</p><pre><code class="language-plain">kubectl describe svc ngx-svc
</code></pre><p><img src="https://static001.geekbang.org/resource/image/80/16/80b6e738bc13e1f1d56fa99080f65716.png?wh=1244x846" alt="图片"></p><p>截图里显示Service对象管理了两个endpoint，分别是“10.10.0.232:80”和“10.10.1.86:80”，初步判断与Service、Deployment的定义相符，那么这两个IP地址是不是Nginx Pod的实际地址呢？</p><p>我们还是用 <code>kubectl get pod</code> 来看一下，加上参数 <code>-o wide</code>：</p><pre><code class="language-plain">kubectl get pod -o wide
</code></pre><p><img src="https://static001.geekbang.org/resource/image/35/34/355129b4eb2290b3df50f7c184c06634.png?wh=1920x241" alt="图片"></p><p>把Pod的地址与Service的信息做个对比，我们就能够验证Service确实用一个静态IP地址代理了两个Pod的动态IP地址。</p><p><strong>那怎么测试Service的负载均衡效果呢？</strong></p><p>因为Service、 Pod的IP地址都是Kubernetes集群的内部网段，所以我们需要用 <code>kubectl exec</code> 进入到Pod内部（或者ssh登录集群节点），再用curl等工具来访问Service：</p><pre><code class="language-bash">kubectl exec -it ngx-dep-6796688696-r2j6t -- sh
</code></pre><p><img src="https://static001.geekbang.org/resource/image/72/28/72eab1f20e7d91ddfe07b5e521712b28.png?wh=1638x838" alt="图片"></p><p>在Pod里，用curl访问Service的IP地址，就会看到它把数据转发给后端的Pod，输出信息会显示具体是哪个Pod响应了请求，就表明Service确实完成了对Pod的负载均衡任务。</p><p>我们再试着删除一个Pod，看看Service是否会更新后端Pod的信息，实现自动化的服务发现：</p><pre><code class="language-bash">kubectl delete pod ngx-dep-6796688696-r2j6t
</code></pre><p><img src="https://static001.geekbang.org/resource/image/68/65/688362b0d462ba94fed6f9c2fcbed565.png?wh=1920x1200" alt="图片"></p><p>由于Pod被Deployment对象管理，删除后会自动重建，而Service又会通过controller-manager实时监控Pod的变化情况，所以就会立即更新它代理的IP地址。通过截图你就可以看到有一个IP地址“10.10.1.86”消失了，换成了新的“10.10.1.87”，它就是新创建的Pod。</p><p>你也可以再尝试一下使用“ping”来测试Service的IP地址：</p><p><img src="https://static001.geekbang.org/resource/image/71/1d/7182131d675c5d03ab9c91be4869a51d.png?wh=1638x428" alt="图片"></p><p>会发现根本ping不通，因为Service的IP地址是“虚”的，只用于转发流量，所以ping无法得到回应数据包，也就失败了。</p><h2>如何以域名的方式使用Service</h2><p>到这里Service的基本用法就讲得差不多了，不过它还有一些高级特性值得了解。</p><p>我们先来看看DNS域名。</p><p>Service对象的IP地址是静态的，保持稳定，这在微服务里确实很重要，不过数字形式的IP地址用起来还是不太方便。这个时候Kubernetes的DNS插件就派上了用处，它可以为Service创建易写易记的域名，让Service更容易使用。</p><p>使用DNS域名之前，我们要先了解一个新的概念：<strong>名字空间</strong>（namespace）。</p><p>注意它与我们在<a href="https://time.geekbang.org/column/article/528640">第2讲</a>里说的用于资源隔离的Linux namespace技术完全不同，千万不要弄混了。Kubernetes只是借用了这个术语，但目标是类似的，用来在集群里实现对API对象的隔离和分组。</p><p>namespace的简写是“<strong>ns</strong>”，你可以使用命令 <code>kubectl get ns</code> 来查看当前集群里都有哪些名字空间，也就是说API对象有哪些分组：</p><pre><code class="language-plain">kubectl get ns
</code></pre><p><img src="https://static001.geekbang.org/resource/image/16/09/169398a24700368f1550950f0e34b409.png?wh=854x368" alt="图片"></p><p>Kubernetes有一个默认的名字空间，叫“<strong>default</strong>”，如果不显式指定，API对象都会在这个“default”名字空间里。而其他的名字空间都有各自的用途，比如“kube-system”就包含了apiserver、etcd等核心组件的Pod。</p><p>因为DNS是一种层次结构，为了避免太多的域名导致冲突，Kubernetes就把名字空间作为域名的一部分，减少了重名的可能性。</p><p>Service对象的域名完全形式是“<strong>对象.名字空间.svc.cluster.local</strong>”，但很多时候也可以省略后面的部分，直接写“<strong>对象.名字空间</strong>”甚至“<strong>对象名</strong>”就足够了，默认会使用对象所在的名字空间（比如这里就是default）。</p><p>现在我们来试验一下DNS域名的用法，还是先 <code>kubectl exec</code>  进入Pod，然后用curl访问 <code>ngx-svc</code>、<code>ngx-svc.default</code> 等域名：</p><p><img src="https://static001.geekbang.org/resource/image/9b/8b/9b8f58e19f7551f9e3a152d79d9d1e8b.png?wh=1638x1204" alt="图片"></p><p>可以看到，现在我们就不再关心Service对象的IP地址，只需要知道它的名字，就可以用DNS的方式去访问后端服务。</p><p>比起Docker，这无疑是一个巨大的进步，而且对比其他微服务框架（如Dubbo、Spring Cloud），由于服务发现机制被集成在了基础设施里，也会让应用的开发更加便捷。</p><p>（顺便说一下，Kubernetes也为每个Pod分配了域名，形式是“<strong>IP地址.名字空间.pod.cluster.local</strong>”，但需要把IP地址里的 <code>.</code> 改成 <code>-</code> 。比如地址 <code>10.10.1.87</code>，它对应的域名就是 <code>10-10-1-87.default.pod</code>。）</p><h2>如何让Service对外暴露服务</h2><p>由于Service是一种负载均衡技术，所以它不仅能够管理Kubernetes集群内部的服务，还能够担当向集群外部暴露服务的重任。</p><p>Service对象有一个关键字段“<strong>type</strong>”，表示Service是哪种类型的负载均衡。前面我们看到的用法都是对集群内部Pod的负载均衡，所以这个字段的值就是默认的“<strong>ClusterIP</strong>”，Service的静态IP地址只能在集群内访问。</p><p>除了“ClusterIP”，Service还支持其他三种类型，分别是“<strong>ExternalName</strong>”“<strong>LoadBalancer</strong>”“<strong>NodePort</strong>”。不过前两种类型一般由云服务商提供，我们的实验环境用不到，所以接下来就重点看“NodePort”这个类型。</p><p>如果我们在使用命令 <code>kubectl expose</code> 的时候加上参数 <code>--type=NodePort</code>，或者在YAML里添加字段 <code>type:NodePort</code>，那么Service除了会对后端的Pod做负载均衡之外，还会在集群里的每个节点上创建一个独立的端口，用这个端口对外提供服务，这也正是“NodePort”这个名字的由来。</p><p>让我们修改一下Service的YAML文件，加上字段“type”：</p><pre><code class="language-yaml">apiVersion: v1
...
spec:
&nbsp; ...
&nbsp; type: NodePort
</code></pre><p>然后创建对象，再查看它的状态：</p><pre><code class="language-plain">kubectl get svc
</code></pre><p><img src="https://static001.geekbang.org/resource/image/64/f9/643cf4690a42f723732f9f150021fff9.png?wh=1756x248" alt="图片"></p><p>就会看到“TYPE”变成了“NodePort”，而在“PORT”列里的端口信息也不一样，除了集群内部使用的“80”端口，还多出了一个“30651”端口，这就是Kubernetes在节点上为Service创建的专用映射端口。</p><p>因为这个端口号属于节点，外部能够直接访问，所以现在我们就可以不用登录集群节点或者进入Pod内部，直接在集群外使用任意一个节点的IP地址，就能够访问Service和它代理的后端服务了。</p><p>比如我现在所在的服务器是“192.168.10.208”，在这台主机上用curl访问Kubernetes集群的两个节点“192.168.10.210”“192.168.10.220”，就可以得到Nginx Pod的响应数据：</p><p><img src="https://static001.geekbang.org/resource/image/eb/75/eb917ecdf52cc3f266e6555bd7a1b075.png?wh=1076x666" alt="图片"></p><p>我把NodePort与Service、Deployment的对应关系画成了图，你看了应该就能更好地明白它的工作原理：</p><p><img src="https://static001.geekbang.org/resource/image/fy/4a/fyyebea67e4471aa53cb3a0e8ebe624a.jpg?wh=1920x940" alt="图片"></p><p>学到这里，你是不是觉得NodePort类型的Service很方便呢。</p><p>不过它也有一些缺点。</p><p>第一个缺点是它的端口数量很有限。Kubernetes为了避免端口冲突，默认只在“30000~32767”这个范围内随机分配，只有2000多个，而且都不是标准端口号，这对于具有大量业务应用的系统来说根本不够用。</p><p>第二个缺点是它会在每个节点上都开端口，然后使用kube-proxy路由到真正的后端Service，这对于有很多计算节点的大集群来说就带来了一些网络通信成本，不是特别经济。</p><p>第三个缺点，它要求向外界暴露节点的IP地址，这在很多时候是不可行的，为了安全还需要在集群外再搭一个反向代理，增加了方案的复杂度。</p><p>虽然有这些缺点，但NodePort仍然是Kubernetes对外提供服务的一种简单易行的方式，在其他更好的方式出现之前，我们也只能使用它。</p><h2>小结</h2><p>好了，今天我们学习了Service对象，它实现了负载均衡和服务发现技术，是Kubernetes应对微服务、服务网格等现代流行应用架构的解决方案。</p><p>我再小结一下今天的要点：</p><ol>
<li>Pod的生命周期很短暂，会不停地创建销毁，所以就需要用Service来实现负载均衡，它由Kubernetes分配固定的IP地址，能够屏蔽后端的Pod变化。</li>
<li>Service对象使用与Deployment、DaemonSet相同的“selector”字段，选择要代理的后端Pod，是松耦合关系。</li>
<li>基于DNS插件，我们能够以域名的方式访问Service，比静态IP地址更方便。</li>
<li>名字空间是Kubernetes用来隔离对象的一种方式，实现了逻辑上的对象分组，Service的域名里就包含了名字空间限定。</li>
<li>Service的默认类型是“ClusterIP”，只能在集群内部访问，如果改成“NodePort”，就会在节点上开启一个随机端口号，让外界也能够访问内部的服务。</li>
</ol><h2>课下作业</h2><p>最后是课下作业时间，给你留两个思考题：</p><ol>
<li>为什么Service的IP地址是静态且虚拟的？出于什么目的，有什么好处？</li>
<li>你了解负载均衡技术吗？它都有哪些算法，Service会用哪种呢？</li>
</ol><p>欢迎在留言区分享你的思考，以输出带动自己输入。到今天，你已经完成2/3的专栏学习了，回看一起学过的内容，不知你收获如何呢。</p><p>如果觉得有帮助，不妨分享给自己身边的朋友一起学习，我们下节课再见。</p><p><img src="https://static001.geekbang.org/resource/image/73/68/7370727f61e82f96acf0316456329968.jpg?wh=1920x2465" alt=""></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/06/7e/735968e2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>西门吹牛</span>
  </div>
  <div class="_2_QraFYR_0">Service 的 IP 是 vip，其实就是保证外部请求的ip不变，不会因为节点变动，而 ip 跟着动，和 lvs + nginx 部署高可用集群一样，主要保证高可用。方便。<br>负载均衡算法，Service 会用哪种呢？<br>轮询，加权轮询，随机，针对有状态的一致性hash，还有针对最少活跃调用数的。<br>最好的负载均衡是自适应负载均衡，可以动态的监控收集服务的状态，各种指标进行加权计算，从中选出个最合适的。<br>感觉，service 应该是自适应，都有能力自动编排了，对于一些服务状态的数据指标，收集应该问题不大<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 说的很好。<br><br>Service的负载均衡能力比较弱，只支持最简单的round-robin。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-05 13:43:21</div>
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
  <div class="_2_QraFYR_0">前几节课多节点集群看起来一切正常，但本节的 dns 测试一直无法通过。周末搞了两天，最终发现是 virtualbox 两张网卡的坑。首先是集群节点的 kubelet 配置需要分别指定 `KUBELET_EXTRA_ARGS=&quot;--node-ip=192.168.56.x&quot;`；然后时安装 flannel 需要指定 `--iface=enp0s3`，这张网卡应该时前面的 node-ip 对应的网卡。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great，两张网卡确实会遇到一些问题，感谢经验分享。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-14 19:21:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/32/a8/d5bf5445.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>郑海成</span>
  </div>
  <div class="_2_QraFYR_0">Q1: svc是基于内核的netfilter技术实现的，在用户态通过iptable和ipvs应用hook链和表，所以它的IP注定是虚拟的，至于静态考虑则是为暴露的服务提供相对稳定性能的dns解析<br><br>Q2: 负载均衡技术分为DNS、四层和七层，dns也是一种简单的负载均衡只是算法很简单随机；四层负载均衡主要有nat、IP隧道等，主要是做一些IP和端口的变换，算法也不多比如rr，七层负载均衡则是基于http header来做负载算法很多，可以做一些流量控制；其次还有在客户端做负载均衡比如nacos、istio等</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-08 20:07:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/cd/ba/3a348f2d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>YueShi</span>
  </div>
  <div class="_2_QraFYR_0">各位大佬们，如果有用虚拟机的，使用挂起再恢复的，建议每次遇到问题先重启虚拟机，踩了很多次这种挂起再恢复导致的网络问题的坑了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，Kubernetes感觉比较娇贵，不那么皮实，重启可以解决很多问题。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-05 19:26:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKotsBr2icbYNYlRSlicGUD1H7lulSTQUAiclsEz9gnG5kCW9qeDwdYtlRMXic3V6sj9UrfKLPJnQojag/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ppd0705</span>
  </div>
  <div class="_2_QraFYR_0">不知道有没有同样遇到 dns 无法解析域名的同学，重启coredns试试：<br>kubectl -n kube-system rollout restart deployment coredns</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 有的时候这些pod会有问题，用describe或者logs看看，有问题就重启。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-06 10:40:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/23/81/3865297c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>龙之大者</span>
  </div>
  <div class="_2_QraFYR_0">进入pod后，如果curl ngx-svc出现Could not resolve hostnames报错，可以重启coredns deployment<br><br>kubectl -n kube-system rollout restart deployment coredns<br><br>https:&#47;&#47;stackoverflow.com&#47;questions&#47;45805483&#47;kubernetes-pods-cant-resolve-hostnames</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，Kubernetes里的组件也不会总是运行正常的，发现有问题就可以简单地重启解决。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-24 21:39:40</div>
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
  <div class="_2_QraFYR_0">NodePort 设置后，得注意可以通过curl集群的节点来访问服务，但这里面不包括master节点</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 一般master节点不会调度运行Pod，如果去掉污点就可以了，可以参考一下DaemonSet。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-26 16:30:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/cd/e0/c85bb948.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>朱雯</span>
  </div>
  <div class="_2_QraFYR_0">nginx负载均衡是可以设置策略的，权重之类的，svc和nodeport是怎么设置这方面的信息呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 因为Kubernetes是基础设施，高级的负载均衡策略属于应用层次，Kubernetes来做这个就属于越位了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-18 17:28:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/b2/91/714c0f07.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>zero</span>
  </div>
  <div class="_2_QraFYR_0">不知道有没有同样遇到 dns 无法解析域名的同学，可能你不能访问的原因是你网络插件Flannel的问题，我自己的虚拟机双网卡，需要在kube-flannel.yml上面指定你的网卡<br>        args:<br>        - --ip-masq<br>        - --kube-subnet-mgr<br>        - --iface=enp0s3  &#47;&#47; 这里替换成自己的网卡</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great!<br><br>我的电脑是m1的，对virtualbox用的不多，可能有的问题没有遇到，感谢经验分享。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-08 13:15:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>叶峥瑶</span>
  </div>
  <div class="_2_QraFYR_0">ping不通可能是配置问题。 可以参考这篇配置成ipvs https:&#47;&#47;blog.csdn.net&#47;Urms_handsomeyu&#47;article&#47;details&#47;106294085</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-07 15:01:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/05/93/3c3f2a6d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>安石</span>
  </div>
  <div class="_2_QraFYR_0">curl ngx-svc我也运行不通，直接在pod里面访问ip可以，域名就不行。<br>也设置了网卡之类（虚拟机）还是不行，按照k8s官方文档操作可以了<br>https:&#47;&#47;kubernetes.io&#47;zh-cn&#47;docs&#47;tasks&#47;administer-cluster&#47;dns-debugging-resolution&#47;</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 应该还是coredns的问题吧。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-31 16:37:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/ef/f1/8b06801a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>哇哈哈</span>
  </div>
  <div class="_2_QraFYR_0">看到这里有点懵，各种 ip 地址不懂怎么来的也不懂为什么要这么搞</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: ip地址是Kubernetes的cni插件分配的，后面有一节课简单讲了Kubernetes的网络。<br><br>其实也没必要搞清楚底层细节，就知道每个pod会自动有个IP地址就行了，然后Service对象就是一个四层的负载均衡功能。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-21 16:47:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/90/23/5c74e9b7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>$侯</span>
  </div>
  <div class="_2_QraFYR_0">虚拟机 curl ngx-svc 产生 Could not resolve 问题解决办法：<br>1. kube-flannel.yml 文件中加入  --iface=enp0s3，位置如下<br>  containers:<br>  - name: kube-flannel<br>    image: xxx<br>    command:<br>    - &#47;opt&#47;bin&#47;flanneld<br>    args:<br>    - --ip-masq<br>    - --kube-subnet-mgr<br>    - --iface=enp0s3 # 这里新增这条<br>2. 修改kubelet配置<br>sudo mkdir &#47;etc&#47;sysconfig<br>sudo vim &#47;etc&#47;sysconfig&#47;kubelet<br>加入：<br>KUBELET_EXTRA_ARGS=&quot;--node-ip=192.168.56.208&quot;<br>保存<br>重启kubelet：<br>systemctl restart kubelet</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个一般都是kubeadm的时候集群没有配置好，最好reset后重新搭建，或者在minikube里做实验也可以。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-02 10:12:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/2c/7e/f1efd18b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>摊牌</span>
  </div>
  <div class="_2_QraFYR_0">service声明为nodePort类型时，可以指定端口吗，老师？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以的，用字段“nodePort”，可以用kubectl explain看文档。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-18 22:48:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTIuj7Wx21ecNlPHCfBsQIchmFxVSlPepwUiaKh0RMGgDB0aibTM50ibQN06dDmbqjuQZUIdH4qiaRJkgQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_adb513</span>
  </div>
  <div class="_2_QraFYR_0">花了钱，又没跟上课程。。。现在接着看，感觉前面内容又丢了😭😭</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 多动手，学习是一个长期的过程，来得及。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-02 22:50:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/3c/4d/3dec4bfe.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>蔡晓慧</span>
  </div>
  <div class="_2_QraFYR_0">老师，请教两个问题 <br>1.Service 和 Deployment 是不是可以写在同一个yml文件里？我看有这么干的。<br>2.如果可以写在同一个文件里，启动命令还是kubectl apply -f *.yml吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 当然可以了，注意YAML 里用“---” 分隔开多个对象，还是用kubectl apply来创建。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-08 15:57:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1c/f0/e9/1ff0a3d5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>...</span>
  </div>
  <div class="_2_QraFYR_0">为什么curl svc.default的时候，老要重启dns deployment才可以。如何彻底解决好呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个就比较麻烦了，如果核心组件dns出了问题，就得找对Kubernetes有足够运维经验的人来解决了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-11 11:06:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/94/fe/5fbf1bdc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Layne</span>
  </div>
  <div class="_2_QraFYR_0">done<br>在实操遇到的问题：<br>kubectl exec -it ngx-dep-7598659bf4-f644t -- sh<br>error: unable to upgrade connection: pod does not exist<br>原因：集群搭建的时候网络配置没弄好，一直到这节需要进入容器才发现<br>所以在搭建集群环境的时候，启动一个pod后，最好进入pod应用验证下。<br><br>---<br>感觉每篇只要理解老师画的关系图，就可以说明白了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-11 10:03:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/a2/3e/7d9812f2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>LRG-</span>
  </div>
  <div class="_2_QraFYR_0">老师，这一段 “因为这个端口号属于节点，外部能够直接访问，所以现在我们就可以不用登录集群节点或者进入 Pod 内部，直接在集群外使用任意一个节点的 IP 地址，就能够访问 Service 和它代理的后端服务了。”， 我尝试，只有用worker的ip可以，用master的ip还是不可以的。是那样的吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: master节点有污点吧，用kubectl看看。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-24 08:58:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/a2/3e/7d9812f2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>LRG-</span>
  </div>
  <div class="_2_QraFYR_0">我先用名称来访问，总是说找不到：<br>&#47; # curl ngx-svc<br>curl: (6) Could not resolve host: ngx-svc<br>&#47; # curl ngx-svc.default<br>curl: (6) Could not resolve host: ngx-svc.default<br>&#47; # curl ngx-svc.default.svc.cluster.local<br>curl: (6) Could not resolve host: ngx-svc.default.svc.cluster.local<br>老师能指导一下我是不是漏了设定什么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 看看coredns是否工作正常，尝试重启它的pod看看。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-23 07:45:22</div>
  </div>
</div>
</div>
</li>
</ul>