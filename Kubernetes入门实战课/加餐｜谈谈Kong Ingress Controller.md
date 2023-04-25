<audio title="加餐｜谈谈Kong Ingress Controller" src="https://static001.geekbang.org/resource/audio/b4/95/b4ef7ffe35e3eaa31b74b317ba291195.mp3" controls="controls"></audio> 
<p>你好，我是Chrono。</p><p>课程已经完结三个多月了，还记得结课时我说的那句话吗：“是终点，更是起点”，课程的完结绝不意味着我们终止对Kubernetes的钻研，相反，不论你我，都会在这个学习的道路上持续地走下去。</p><p>当初开课时，我计划了很多内容，不过Kubernetes的领域实在太广，加上我日常工作比较忙，时间和精力有限，所以一些原定的知识点没有来得及展现，比较可惜，我一直想找机会做个补偿。</p><p>这几天开发任务略微空闲了些，我就又回到了专栏，准备使用另一个流行的工具：Kong Ingress Controller，再来讲讲对Kubernetes集群管理非常重要的Ingress。</p><h2>认识Kong Ingress Controller</h2><p>我们先快速回顾一下Ingress的知识（<a href="https://time.geekbang.org/column/article/538760">第21讲</a>）。</p><p>Ingress类似Service，基于HTTP/HTTPS协议，是七层负载均衡规则的集合，但它自身没有管理能力，必须要借助Ingress Controller才能控制Kubernetes集群的进出口流量。</p><p>所以，基于Ingress的定义，就出现了各式各样的Ingress Controller实现。</p><p>我们已经见过了Nginx官方开发的Nginx Ingress Controller，但它局限于Nginx自身的能力，Ingress、Service等对象更新时必须要修改静态的配置文件，再重启进程（reload），在变动频繁的微服务系统里就会引发一些问题。</p><!-- [[[read_end]]] --><p>而今天要说的<strong>Kong Ingress Controller</strong>，则是站在了Nginx这个巨人的肩膀之上，基于OpenResty和内嵌的LuaJIT环境，实现了完全动态的路由变更，消除了reload的成本，运行更加平稳，而且还有很多额外的增强功能，非常适合那些对Kubernetes集群流量有更高、更细致管理需求的用户（<a href="https://konghq.com/solutions/build-on-kubernetes">图片来源Kong官网</a>）。</p><p><img src="https://static001.geekbang.org/resource/image/8e/43/8ed5086bb73d01ccdcd83deec1b8d643.png?wh=400x300" alt="图片"></p><h2>安装Kong Ingress Controller</h2><p>接下来我们就来看看如何在Kubernetes集群里引入Kong Ingress Controller。</p><p>简单起见，这次我选择了minikube环境，版本还是1.25.2，对应的Kubernetes也是之前的1.23.3：</p><p><img src="https://static001.geekbang.org/resource/image/25/53/252d80c9a08b7ddfccceb09a63b2da53.png?wh=1266x360" alt="图片"></p><p>目前Kong Ingress Controller的最新版本是2.7.0，你可以从GitHub上(<a href="https://github.com/Kong/kubernetes-ingress-controller">https://github.com/Kong/kubernetes-ingress-controller</a>)直接获取它的源码包：</p><pre><code class="language-plain">wget https://github.com/Kong/kubernetes-ingress-controller/archive/refs/tags/v2.7.0.tar.gz
</code></pre><p>Kong Ingress Controller安装所需的YAML文件，都存放在解压缩后的“deploy”目录里，提供“有数据库”和“无数据库”两种部署方式，这里我选择了最简单的“无数据库”方式，只需要一个 <code>all-in-one-dbless.yaml</code> 就可以完成部署工作，也就是执行这条 <code>kubectl apply</code> 命令：</p><pre><code class="language-plain">kubectl apply -f all-in-one-dbless.yaml
</code></pre><p><img src="https://static001.geekbang.org/resource/image/ee/a6/ee592a45c3b45c8335a2d1ae08bd73a6.png?wh=1920x981" alt="图片"></p><p>我们可以再对比一下两种 Ingress Controller的安装方式。Nginx Ingress Controller是由多个分散的YAML文件组成的，需要顺序执行多次 <code>kubectl apply</code> 命令，有点麻烦；<strong>而Kong Ingress Controller则把Namespace、RBAC、Secret、CRD等对象都合并在了一个文件里，安装很方便，同时也不会发生遗忘创建资源的错误。</strong></p><p>安装之后，Kong Ingress Controller会创建一个新的名字空间“kong”，里面有一个默认的Ingress Controller，还有对应的Service，你可以用 <code>kubectl get</code> 来查看：</p><p><img src="https://static001.geekbang.org/resource/image/96/9e/967cbd227fc4aa8202b2531b59341f9e.png?wh=1920x326" alt="图片"></p><p>看这里的截图，你可能会注意到，在 <code>kubectl get pod</code> 输出的“READY”列里显示的是“2/2”，意思是这个Pod里有两个容器。</p><p>这也是Kong Ingress Controller与Nginx Ingress Controller在实现架构方面的一个明显不同点。</p><p>Kong Ingress Controller，在Pod里使用两个容器，分别运行管理进程Controller和代理进程Proxy，两个容器之间使用环回地址（Loopback）通信；而Nginx Ingress Controller则是因为要修改静态的Nginx配置文件，所以管理进程和代理进程必须在一个容器里（<a href="https://docs.konghq.com/kubernetes-ingress-controller/2.7.x/concepts/design/">图片</a>表示Kong架构设计）。</p><p><img src="https://static001.geekbang.org/resource/image/e7/d3/e78c0c518c6257bc5018d0df8bd536d3.png?wh=1262x644" alt="图片"></p><p>两种方式并没有优劣之分，但<strong>Kong Ingress Controller分离的好处是两个容器彼此独立，可以各自升级维护，对运维更友好一点</strong>。</p><p>Kong Ingress Controller还创建了两个Service对象，其中的“kong-proxy”是转发流量的服务，注意它被定义成了“LoadBalancer”类型，显然是为了在生产环境里对外暴露服务，不过在我们的实验环境（无论是minikube还是kubeadm）中只能使用NodePort的形式，这里可以看到80端口被映射到了节点的32201。</p><p>现在让我们尝试访问一下Kong Ingress Controller，IP就用worker节点的地址，如果你和我一样用的是minikube，则可以用 <code>$(minikube ip)</code> 的形式简单获取：</p><pre><code class="language-plain">curl $(minikube ip):32201 -i
</code></pre><p><img src="https://static001.geekbang.org/resource/image/a0/f5/a00aea97b464f071e9bea603f5678ef5.png?wh=1256x606" alt="图片"></p><p>从curl获取的响应结果可以看到， Kong Ingress Controller 2.7内部使用的Kong版本是3.0.1，因为现在我们没有为它配置任何Ingress资源，所以返回了状态码404，这是正常的。</p><p>我们还可以用 <code>kubectl exec</code> 命令进入Pod，查看它的内部信息：</p><p><img src="https://static001.geekbang.org/resource/image/60/32/60a4f01dd3bbb0afab4d142b1d232a32.png?wh=1920x1814" alt="图片"></p><p>虽然Kong Ingress Controller里有两个容器，但我们不需要特意用 <code>-c</code> 选项指定容器，它会自动进入默认的Proxy容器（另一个Controller容器里因为不包含Shell，也是无法进入查看的）。</p><h2>使用Kong Ingress Controller</h2><p>安装好了，我们看如何使用。和第21讲里的一样，我们仍然不使用默认的Ingress Controller，而是利用Ingress Class自己创建一个新的实例，这样能够更好地理解掌握Kong Ingress Controller的用法。</p><p>首先，定义后端应用，还是用Nginx来模拟，做法和<a href="https://time.geekbang.org/column/article/536829">第20讲</a>里的差不多，用ConfigMap定义配置文件再加载进Nginx Pod里，然后部署Deployment和Service，比较简单，你也比较熟悉，就不列出YAML 代码了，只看一下运行命令后的截图：</p><p><img src="https://static001.geekbang.org/resource/image/a2/05/a2d859927200ca449d9e50f6122e5705.png?wh=1654x484" alt="图片"></p><p>显示我创建了两个Nginx Pod，Service对象的名字是ngx-svc。</p><p>接下来是定义Ingress Class，名字是“<strong>kong-ink</strong>”， “spec.controller”字段的值是Kong Ingress Controller的名字“<strong>ingress-controllers.konghq.com/kong</strong>”，YAML的格式可以参考<a href="https://time.geekbang.org/column/article/538760">第21讲</a>：</p><pre><code class="language-yaml">apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
&nbsp; name: kong-ink

spec:
&nbsp; controller: ingress-controllers.konghq.com/kong
</code></pre><p>然后就是定义Ingress对象了，我们还是可以用 <code>kubectl create</code> 来生成YAML 样板文件，用 <code>--rule</code> 指定路由规则，用 <code>--class</code> 指定Ingress Class：</p><pre><code class="language-plain">kubectl create ing kong-ing --rule="kong.test/=ngx-svc:80" --class=kong-ink $out
</code></pre><p>生成的Ingress对象大概就是下面这样，域名是“kong.test”，流量会转发到后端的ngx-svc服务：</p><pre><code class="language-yaml">apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
&nbsp; name: kong-ing

spec:
&nbsp; ingressClassName: kong-ink

&nbsp; rules:
&nbsp; - host: kong.test
&nbsp; &nbsp; http:
&nbsp; &nbsp; &nbsp; paths:
&nbsp; &nbsp; &nbsp; - path: /
&nbsp; &nbsp; &nbsp; &nbsp; pathType: Prefix
&nbsp; &nbsp; &nbsp; &nbsp; backend:
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; service:
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; name: ngx-svc
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; port:
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; number: 80
</code></pre><p>最后，我们要从 <code>all-in-one-dbless.yaml</code> 这个文件中分离出Ingress Controller的定义。其实也很简单，只要搜索“Deployment”就可以了，然后把它以及相关的Service代码复制一份，另存成“kic.yml”。</p><p>当然了，刚复制的代码和默认的Kong Ingress Controller是完全相同的，所以我们必须要参考帮助文档做一些修改，要点我列在了这里：</p><ul>
<li>Deployment、Service里metadata的 name 都要重命名，比如改成 ingress-kong-dep、ingress-kong-svc。</li>
<li>spec.selector 和 template.metadata.labels 也要修改成自己的名字，一般来说和Deployment的名字一样，也就是ingress-kong-dep。</li>
<li>第一个容器是流量代理Proxy，它里面的镜像可以根据需要，改成任意支持的版本，比如Kong:2.7、Kong:2.8或者Kong:3.1。</li>
<li>第二个容器是规则管理Controller，要用环境变量“CONTROLLER_INGRESS_CLASS”指定新的Ingress Class名字 <code>kong-ink</code>，同时用“CONTROLLER_PUBLISH_SERVICE”指定Service的名字 <code>ingress-kong-svc</code>。</li>
<li>Service对象可以把类型改成NodePort，方便后续的测试。</li>
</ul><p>改了这些之后，一个新的Kong Ingress Controller就完成了，大概是这样，修改点我也加注释了你可以对照着看：</p><pre><code class="language-yaml">apiVersion: apps/v1
kind: Deployment
metadata:
&nbsp; name: ingress-kong-dep            # 重命名
&nbsp; namespace: kong
spec:
&nbsp; replicas: 1
&nbsp; selector:
&nbsp; &nbsp; matchLabels:
&nbsp; &nbsp; &nbsp; app: ingress-kong-dep        # 重命名
&nbsp; template:
&nbsp; &nbsp; metadata:
&nbsp; &nbsp; &nbsp; labels:
&nbsp; &nbsp; &nbsp; &nbsp; app: ingress-kong-dep        # 重命名
&nbsp; &nbsp; spec:
&nbsp; &nbsp; &nbsp; containers:
&nbsp; &nbsp; &nbsp; - env:                       # 第一个容器， Proxy
        ...
&nbsp; &nbsp; &nbsp; &nbsp; image: kong:3.1            # 改镜像

&nbsp; &nbsp; &nbsp; - env:                       # 第二个容器，Controller
&nbsp; &nbsp; &nbsp; &nbsp; - name: CONTROLLER_INGRESS_CLASS
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; value: kong-ink                  # 改Ingress Class
&nbsp; &nbsp; &nbsp; &nbsp; - name: CONTROLLER_PUBLISH_SERVICE
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; value: kong/ingress-kong-svc     # 改Service
        ...
---

apiVersion: v1
kind: Service
metadata:
&nbsp; name: ingress-kong-svc          # 重命名
&nbsp; namespace: kong
spec:
  ...
&nbsp; selector:
&nbsp; &nbsp; app: ingress-kong-dep         # 重命名
&nbsp; type: NodePort                  # 改类型 
</code></pre><p>在我们专栏的配套GitHub项目里，你也可以直接找到改好的YAML 文件。<br>
把这些都准备好，我们就可以来测试验证Kong Ingress Controller了：</p><pre><code class="language-plain">kubectl apply -f ngx-deploy.yml
kubectl apply -f ingress.yml
kubectl apply -f kic.yml
</code></pre><p><img src="https://static001.geekbang.org/resource/image/7a/cc/7a00571f1c6ffc3391bda762206e74cc.png?wh=1920x442" alt="图片"></p><p>这个截图显示了这些对象的创建结果，其中，新Service对象的NodePort端口是32521。</p><p>下面我们就来用curl发送HTTP请求，注意，<strong>应该用“--resolve”或者“-H”参数指定Ingress里定义的域名“kong.test”</strong>，否则Kong Ingress Controller会找不到路由：</p><pre><code class="language-plain">curl $(minikube ip):32521 -H 'host: kong.test' -v
</code></pre><p><img src="https://static001.geekbang.org/resource/image/16/3d/16b663f31e55015a526791c22f6bcc3d.png?wh=1764x1320" alt="图片"></p><p>你可以看到，Kong Ingress Controller正确应用了Ingress路由规则，返回了后端Nginx应用的响应数据，而且从响应头“Via”里还可以发现，它现在用的是Kong 3.1。</p><h2>扩展Kong Ingress Controller</h2><p>到这里，Kong Ingress Controller的基本用法你就掌握了。</p><p>不过，只使用Kubernetes标准的Ingress资源来管理流量，是无法发挥出Kong Ingress Controller的真正实力的，它还有很多非常好用、非常实用的增强功能。</p><p>我们在<a href="https://time.geekbang.org/column/article/547301">第27讲</a>里曾经说过annotation，是Kubernetes为资源对象提供的一个方便扩展功能的手段，所以，<strong>使用annotation就可以在不修改Ingress自身定义的前提下，让Kong Ingress Controller更好地利用内部的Kong来管理流量。</strong></p><p>目前Kong Ingress Controller支持在Ingress和Service这两个对象上添加annotation，相关的详细文档可以参考官网（<a href="https://docs.konghq.com/kubernetes-ingress-controller/2.7.x/references/annotations/">https://docs.konghq.com/kubernetes-ingress-controller/2.7.x/references/annotations/</a>），这里我只介绍两个annotation。</p><p>第一个是“<strong>konghq.com/host-aliases</strong>”，它可以为Ingress规则添加额外的域名。</p><p>你应该知道吧，Ingress的域名里可以使用通配符 <code>*</code>，比如 <code>*.abc.com</code>，但问题在于 <code>*</code> 只能是前缀，不能是后缀，也就是说我们无法写出 <code>abc.*</code> 这样的域名，这在管理多个域名的时候就显得有点麻烦。</p><p>有了“konghq.com/host-aliases”，我们就可以用它来“绕过”这个限制，让Ingress轻松匹配有不同后缀的域名。</p><p>比如，我修改一下Ingress定义，在“metadata”里添加一个annotation，让它除了“kong.test”，还能够支持“kong.dev”“kong.ops”等域名，就是这样：</p><pre><code class="language-yaml">apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
&nbsp; name: kong-ing
&nbsp; annotations:
&nbsp; &nbsp; konghq.com/host-aliases: "kong.dev, kong.ops"  #注意这里
spec:
  ...
</code></pre><p>使用 <code>kubectl apply</code> 更新Ingress，再用curl来测试一下：</p><p><img src="https://static001.geekbang.org/resource/image/34/yy/341506cde73d8b7a75eb6ee75ea77eyy.png?wh=1646x478" alt="图片"></p><p>你就会发现Ingress已经支持了这几个新域名。</p><p>第二个是“<strong>konghq.com/plugins</strong>”，它可以启用Kong Ingress Controller内置的各种插件（Plugins）。</p><p>插件，是Kong Ingress Controller的一个特色功能，你可以理解成是“预制工件”，能够附加在流量转发的过程中，实现各种数据处理，并且这个插件机制是开放的，我们既可以使用官方插件，也可以使用第三方插件，还可以使用Lua、Go等语言编写符合自己特定需求的插件。</p><p>Kong公司维护了一个经过认证的插件中心（<a href="https://docs.konghq.com/hub/">https://docs.konghq.com/hub/</a>），你可以在这里找到涉及认证、安全、流控、分析、日志等多个领域大约100多个插件，今天我们看两个常用的 Response Transformer、Rate Limiting。</p><p><img src="https://static001.geekbang.org/resource/image/ed/8d/edc8b1307a7194935d2ccc96fb4bbe8d.png?wh=1920x1164" alt="图片"></p><p>Response Transformer插件实现了对响应数据的修改，能够添加、替换、删除响应头或者响应体；Rate Limiting插件就是限速，能够以时分秒等单位任意限制客户端访问的次数。</p><p>定义插件需要使用CRD资源，名字是“<strong>KongPlugin</strong>”，你也可以用<code>kubectl api-resources</code>、<code>kubectl explain</code> 等命令来查看它的apiVersion、kind等信息：</p><p><img src="https://static001.geekbang.org/resource/image/36/43/36a25a182b0b1b671c1336a7b0610443.png?wh=1920x791" alt="图片"></p><p>下面我就给出这两个插件对象的示例定义：</p><pre><code class="language-yaml">apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
&nbsp; name: kong-add-resp-header-plugin

plugin: response-transformer
config:
&nbsp; add:
&nbsp; &nbsp; headers:
&nbsp; &nbsp; - Resp-New-Header:kong-kic

---

apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
&nbsp; name: kong-rate-limiting-plugin

plugin: rate-limiting
config:
&nbsp; minute: 2
</code></pre><p>KongPlugin对象，因为是自定义资源，所以和标准Kubernetes对象不一样，不使用“spec”字段，而是用“<strong>plugin</strong>”来指定插件名，用“<strong>config</strong>”来指定插件的配置参数。</p><p>比如在这里，我就让Response Transformer插件添加一个新的响应头字段，让Rate Limiting插件限制客户端每分钟只能发两个请求。</p><p>定义好这两个插件之后，我们就可以在Ingress对象里用annotations来启用插件功能了：</p><pre><code class="language-yaml">apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
&nbsp; name: kong-ing
&nbsp; annotations:
&nbsp; &nbsp; konghq.com/plugins: |
&nbsp; &nbsp; &nbsp; &nbsp; kong-add-resp-header-plugin,
&nbsp; &nbsp; &nbsp; &nbsp; kong-rate-limiting-plugin
</code></pre><p>现在让我们应用这些插件对象，并且更新Ingress：</p><pre><code class="language-plain">kubectl apply -f crd.yml
</code></pre><p>再发送curl请求：</p><pre><code class="language-plain">curl $(minikube ip):32521 -H 'host: kong.test' -i
</code></pre><p><img src="https://static001.geekbang.org/resource/image/a6/15/a6cc78271975de082c140a83e581b615.png?wh=1756x1200" alt="图片"></p><p>你就会发现响应头里多出了几个字段，其中的 <code>RateLimit-*</code> 是限速信息，而 <code>Resp-New-Header</code> 就是新加的响应头字段。</p><p>把curl连续执行几次，就可以看到限速插件生效了：</p><p><img src="https://static001.geekbang.org/resource/image/b2/82/b2b321c8f7597c9ed151c2783af76282.png?wh=1756x1144" alt="图片"></p><p>Kong Ingress Controller会返回429错误，告诉你访问受限，而且会用“Retry-After”等字段来告诉你多少秒之后才能重新发请求。</p><h2>小结</h2><p>好了，今天我们学习了另一种在Kubernetes管理集群进出流量的工具：Kong Ingress Controller，小结一下要点内容：</p><ol>
<li>Kong Ingress Controller的底层内核仍然是Nginx，但基于OpenResty和LuaJIT，实现了对路由的完全动态管理，不需要reload。</li>
<li>使用“无数据库”的方式可以非常简单地安装Kong Ingress Controller，它是一个由两个容器组成的Pod。</li>
<li>Kong Ingress Controller支持标准的Ingress资源，但还使用了annotation和CRD提供更多的扩展增强功能，特别是插件，可以灵活地加载或者拆卸，实现复杂的流量管理策略。</li>
</ol><p>作为一个CNCF云原生项目，Kong Ingress Controller已经得到了广泛的应用和认可，而且在近年的发展过程中，它也开始支持新的Gateway API，等下次有机会我们再细聊吧。</p><h2>课下作业</h2><p>最后是课下作业时间，给你留两个思考题：</p><ol>
<li>你能否对比一下Kong Ingress Controller和Nginx Ingress Controller这两个产品，你看重的是它哪方面的表现呢？</li>
<li>你觉得插件这种机制有什么好处，能否列举一些其他领域里的类似项目？</li>
</ol><p>好久不见了，期待看到你的想法，我们一起讨论，留言区见。</p><p><img src="https://static001.geekbang.org/resource/image/e9/c8/e9c9050ebcd1d0fc001c53e1d75a37c8.jpg?wh=1920x2401" alt="图片"></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/a6/d4/8d50d502.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>简简单单</span>
  </div>
  <div class="_2_QraFYR_0">赞</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: thanks.</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-15 21:54:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/95/af/b7f8dc43.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>拓山</span>
  </div>
  <div class="_2_QraFYR_0">谢谢老师，两周的高强度学习，完成了k8s的入门。<br>后面就要开始用k8s完成一系列云原生应用开发部署</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: awesome!</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-20 17:50:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/7c/25/19cbcd56.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>StackOverflow</span>
  </div>
  <div class="_2_QraFYR_0">和 apisix 功能很类似</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 当然，不过两者的先后顺序要搞清楚。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-02 16:06:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/5c/65/27fabb5f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>茗</span>
  </div>
  <div class="_2_QraFYR_0">老师，我初始化集群的时候制定了service的网段，为啥创建的service的ip地址不是我指定的网段呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个可能要查文档了，不是一下子就能解答的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-17 10:20:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/19/35/be8372be.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>quietwater</span>
  </div>
  <div class="_2_QraFYR_0">老师的课程很对我的胃口，按部就班学习不会有什么障碍，必须点赞！！！APISIX也是一种Ingress，老师是否可以介绍一下。查了一些资料，感觉APISIX比Kong更强大。求解惑！！！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: sorry，这个就不方便谈了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-23 11:38:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2f/85/6f/1654f4b9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>nc_ops</span>
  </div>
  <div class="_2_QraFYR_0">除了执行curl $(minikube ip):32521 -H &#39;host: kong.test&#39; -i，如果想在网页上查看实验结果，该怎么操作呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 因为minikube是运行在Linux里的，如果是本机可以直接用Firefox来访问，不然就只能外面再来一个Nginx做代理了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-17 15:37:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/cb/1f/d12f34de.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Sheldon</span>
  </div>
  <div class="_2_QraFYR_0">谢谢老师的加餐，这个课程真的很超值。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: my pleasure.</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-01 11:53:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2f/85/6f/1654f4b9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>nc_ops</span>
  </div>
  <div class="_2_QraFYR_0">老师，进入pod时为什么会默认进入proxy这个容器？这是在yaml文件的哪个位置确定的呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 应该是pod里定义的第一个容器。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-03 10:59:31</div>
  </div>
</div>
</div>
</li>
</ul>