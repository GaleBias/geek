<audio title="21｜Ingress：集群进出流量的总管" src="https://static001.geekbang.org/resource/audio/42/55/424c8a661bcc9yya39858a37c2838255.mp3" controls="controls"></audio> 
<p>你好，我是Chrono。</p><p>上次课里我们学习了Service对象，它是Kubernetes内置的负载均衡机制，使用静态IP地址代理动态变化的Pod，支持域名访问和服务发现，是微服务架构必需的基础设施。</p><p>Service很有用，但也只能说是“基础设施”，它对网络流量的管理方案还是太简单，离复杂的现代应用架构需求还有很大的差距，所以Kubernetes就在Service之上又提出了一个新的概念：Ingress。</p><p>比起Service，Ingress更接近实际业务，对它的开发、应用和讨论也是社区里最火爆的，今天我们就来看看Ingress，还有与它关联的Ingress Controller、Ingress Class等对象。</p><h2>为什么要有Ingress</h2><p>通过上次课程的讲解，我们知道了Service的功能和运行机制，它本质上就是一个由kube-proxy控制的四层负载均衡，在TCP/IP协议栈上转发流量（<a href="https://kubernetes.io/zh/docs/concepts/services-networking/service/">Service工作原理示意图</a>）：</p><p><img src="https://static001.geekbang.org/resource/image/03/74/0347a0b3bae55fb9ef6c07469e964b74.png?wh=1622x1214" alt="图片"></p><p>但在四层上的负载均衡功能还是太有限了，只能够依据IP地址和端口号做一些简单的判断和组合，而我们现在的绝大多数应用都是跑在七层的HTTP/HTTPS协议上的，有更多的高级路由条件，比如主机名、URI、请求头、证书等等，而这些在TCP/IP网络栈里是根本看不见的。</p><!-- [[[read_end]]] --><p>Service还有一个缺点，它比较适合代理集群内部的服务。如果想要把服务暴露到集群外部，就只能使用NodePort或者LoadBalancer这两种方式，而它们都缺乏足够的灵活性，难以管控，这就导致了一种很无奈的局面：我们的服务空有一身本领，却没有合适的机会走出去大展拳脚。</p><p>该怎么解决这个问题呢？</p><p>Kubernetes还是沿用了Service的思路，既然Service是四层的负载均衡，那么我再引入一个新的API对象，在七层上做负载均衡是不是就可以了呢？</p><p><strong>不过除了七层负载均衡，这个对象还应该承担更多的职责，也就是作为流量的总入口，统管集群的进出口数据</strong>，“扇入”“扇出”流量（也就是我们常说的“南北向”），让外部用户能够安全、顺畅、便捷地访问内部服务（<a href="https://bishoylabib.com/exposing-your-application-to-the-public-ingress/">图片来源</a>）：</p><p><img src="https://static001.geekbang.org/resource/image/e6/55/e6ce31b027ba2a8d94cdc553a2c97255.png?wh=1288x834" alt="图片"></p><p>所以，这个API对象就顺理成章地被命名为 <code>Ingress</code>，意思就是集群内外边界上的入口。</p><h2>为什么要有Ingress Controller</h2><p>再对比一下Service我们就能更透彻地理解Ingress。</p><p>Ingress可以说是在七层上另一种形式的Service，它同样会代理一些后端的Pod，也有一些路由规则来定义流量应该如何分配、转发，只不过这些规则都使用的是HTTP/HTTPS协议。</p><p>你应该知道，Service本身是没有服务能力的，它只是一些iptables规则，<strong>真正配置、应用这些规则的实际上是节点里的kube-proxy组件</strong>。如果没有kube-proxy，Service定义得再完善也没有用。</p><p>同样的，Ingress也只是一些HTTP路由规则的集合，相当于一份静态的描述文件，真正要把这些规则在集群里实施运行，还需要有另外一个东西，这就是 <code>Ingress Controller</code>，它的作用就相当于Service的kube-proxy，能够读取、应用Ingress规则，处理、调度流量。</p><p>按理来说，Kubernetes应该把Ingress Controller内置实现，作为基础设施的一部分，就像kube-proxy一样。</p><p><strong>不过Ingress Controller要做的事情太多，与上层业务联系太密切，所以Kubernetes把Ingress Controller的实现交给了社区</strong>，任何人都可以开发Ingress Controller，只要遵守Ingress规则就好。</p><p>这就造成了Ingress Controller“百花齐放”的盛况。</p><p>由于Ingress Controller把守了集群流量的关键入口，掌握了它就拥有了控制集群应用的“话语权”，所以众多公司纷纷入场，精心打造自己的Ingress Controller，意图在Kubernetes流量进出管理这个领域占有一席之地。</p><p>这些实现中最著名的，就是老牌的反向代理和负载均衡软件Nginx了。从Ingress Controller的描述上我们也可以看到，HTTP层面的流量管理、安全控制等功能其实就是经典的反向代理，而Nginx则是其中稳定性最好、性能最高的产品，所以它也理所当然成为了Kubernetes里应用得最广泛的Ingress Controller。</p><p>不过，因为Nginx是开源的，谁都可以基于源码做二次开发，所以它又有很多的变种，比如社区的Kubernetes Ingress Controller（<a href="https://github.com/kubernetes/ingress-nginx">https://github.com/kubernetes/ingress-nginx</a>）、Nginx公司自己的Nginx Ingress Controller（<a href="https://github.com/nginxinc/kubernetes-ingress">https://github.com/nginxinc/kubernetes-ingress</a>）、还有基于OpenResty的Kong Ingress Controller（<a href="https://github.com/Kong/kubernetes-ingress-controller">https://github.com/Kong/kubernetes-ingress-controller</a>）等等。</p><p>根据Docker Hub上的统计，<strong>Nginx公司的开发实现是下载量最多的Ingress Controller</strong>，所以我将以它为例，讲解Ingress和Ingress Controller的用法。</p><p>下面的<a href="https://www.nginx.com/products/nginx-ingress-controller/">这张图</a>就来自Nginx官网，比较清楚地展示了Ingress Controller在Kubernetes集群里的地位：</p><p><img src="https://static001.geekbang.org/resource/image/eb/f8/ebebd12312fa5e6eb1ea90c930bd5ef8.png?wh=1920x706" alt="图片"></p><h2>为什么要有IngressClass</h2><p>那么到现在，有了Ingress和Ingress Controller，我们是不是就可以完美地管理集群的进出流量了呢？</p><p>最初Kubernetes也是这么想的，一个集群里有一个Ingress Controller，再给它配上许多不同的Ingress规则，应该就可以解决请求的路由和分发问题了。</p><p>但随着Ingress在实践中的大量应用，很多用户发现这种用法会带来一些问题，比如：</p><ul>
<li>由于某些原因，项目组需要引入不同的Ingress Controller，但Kubernetes不允许这样做；</li>
<li>Ingress规则太多，都交给一个Ingress Controller处理会让它不堪重负；</li>
<li>多个Ingress对象没有很好的逻辑分组方式，管理和维护成本很高；</li>
<li>集群里有不同的租户，他们对Ingress的需求差异很大甚至有冲突，无法部署在同一个Ingress Controller上。</li>
</ul><p>所以，Kubernetes就又提出了一个 <code>Ingress Class</code> 的概念，让它插在Ingress和Ingress Controller中间，作为流量规则和控制器的协调人，解除了Ingress和Ingress Controller的强绑定关系。</p><p>现在，<strong>Kubernetes用户可以转向管理Ingress Class，用它来定义不同的业务逻辑分组，简化Ingress规则的复杂度</strong>。比如说，我们可以用Class A处理博客流量、Class B处理短视频流量、Class C处理购物流量。</p><p><img src="https://static001.geekbang.org/resource/image/88/0e/8843704c6314706c9b6f4f2399ca940e.jpg?wh=1920x1306" alt="图片"></p><p>这些Ingress和Ingress Controller彼此独立，不会发生冲突，所以上面的那些问题也就随着Ingress Class的引入迎刃而解了。</p><h2>如何使用YAML描述Ingress/Ingress Class</h2><p>我们花了比较多的篇幅学习Ingress、 Ingress Controller、Ingress Class这三个对象，全是理论，你可能觉得学得有点累。但这也是没办法的事情，毕竟现实的业务就是这么复杂，而且这个设计架构也是社区经过长期讨论后达成的一致结论，是我们目前能获得的最佳解决方案。</p><p>好，了解了这三个概念之后，我们就可以来看看如何为它们编写YAML描述文件了。</p><p>和之前学习Deployment、Service对象一样，首先应当用命令 <code>kubectl api-resources</code> 查看它们的基本信息，输出列在这里了：</p><pre><code class="language-plain">kubectl api-resources

NAME&nbsp; &nbsp; &nbsp; 		SHORTNAMES&nbsp; &nbsp;APIVERSION&nbsp; &nbsp; &nbsp; &nbsp;		NAMESPACED&nbsp; &nbsp;KIND
ingresses&nbsp; &nbsp; &nbsp; &nbsp;ing&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; networking.k8s.io/v1&nbsp; &nbsp;true&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;Ingress
ingressclasses&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;networking.k8s.io/v1&nbsp; &nbsp;false&nbsp; &nbsp; &nbsp; &nbsp; IngressClass
</code></pre><p>你可以看到，Ingress和Ingress Class的apiVersion都是“<strong>networking.k8s.io/v1</strong>”，而且Ingress有一个简写“<strong>ing</strong>”，但Ingress Controller怎么找不到呢？</p><p>这是因为Ingress Controller和其他两个对象不太一样，它不只是描述文件，是一个要实际干活、处理流量的应用程序，而应用程序在Kubernetes里早就有对象来管理了，那就是Deployment和DaemonSet，所以我们只需要再学习Ingress和Ingress Class的的用法就可以了。</p><p>先看Ingress。</p><p>Ingress也是可以使用 <code>kubectl create</code> 来创建样板文件的，和Service类似，它也需要用两个附加参数：</p><ul>
<li><code>--class</code>，指定Ingress从属的Ingress Class对象。</li>
<li><code>--rule</code>，指定路由规则，基本形式是“URI=Service”，也就是说是访问HTTP路径就转发到对应的Service对象，再由Service对象转发给后端的Pod。</li>
</ul><p>好，现在我们就执行命令，看看Ingress到底长什么样：</p><pre><code class="language-plain">export out="--dry-run=client -o yaml"
kubectl create ing ngx-ing --rule="ngx.test/=ngx-svc:80" --class=ngx-ink $out
</code></pre><pre><code class="language-yaml">apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
&nbsp; name: ngx-ing
&nbsp;&nbsp;
spec:

&nbsp; ingressClassName: ngx-ink
&nbsp;&nbsp;
&nbsp; rules:
&nbsp; - host: ngx.test
&nbsp; &nbsp; http:
&nbsp; &nbsp; &nbsp; paths:
      - path: /
        pathType: Exact
&nbsp; &nbsp; &nbsp;   backend:
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; service:
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; name: ngx-svc
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; port:
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; number: 80
</code></pre><p>在这份Ingress的YAML里，有两个关键字段：“<strong>ingressClassName</strong>”和“<strong>rules</strong>”，分别对应了命令行参数，含义还是比较好理解的。</p><p>只是“rules”的格式比较复杂，嵌套层次很深。不过仔细点看就会发现它是把路由规则拆散了，有host和http path，在path里又指定了路径的匹配方式，可以是精确匹配（Exact）或者是前缀匹配（Prefix），再用backend来指定转发的目标Service对象。</p><p>不过我个人觉得，Ingress YAML里的描述还不如 <code>kubectl create</code> 命令行里的 <code>--rule</code> 参数来得直观易懂，而且YAML里的字段太多也很容易弄错，建议你还是让kubectl来自动生成规则，然后再略作修改比较好。</p><p>有了Ingress对象，那么与它关联的Ingress Class是什么样的呢？</p><p>其实Ingress Class本身并没有什么实际的功能，只是起到联系Ingress和Ingress Controller的作用，所以它的定义非常简单，在“<strong>spec</strong>”里只有一个必需的字段“<strong>controller</strong>”，表示要使用哪个Ingress Controller，具体的名字就要看实现文档了。</p><p>比如，如果我要用Nginx开发的Ingress Controller，那么就要用名字“<strong>nginx.org/ingress-controller</strong>”：</p><pre><code class="language-yaml">apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
&nbsp; name: ngx-ink

spec:
&nbsp; controller: nginx.org/ingress-controller
</code></pre><p>Ingress和Service、Ingress Class的关系我也画成了一张图，方便你参考：</p><p><img src="https://static001.geekbang.org/resource/image/6b/af/6bd934a9c8c81a9f194d2d90ede172af.jpg?wh=1920x1005" alt="图片"></p><h2>如何在Kubernetes里使用Ingress/Ingress Class</h2><p>因为Ingress Class很小，所以我把它与Ingress合成了一个YAML文件，让我们用 <code>kubectl apply</code> 创建这两个对象：</p><pre><code class="language-plain">kubectl apply -f ingress.yml
</code></pre><p>然后我们用 <code>kubectl get</code> 来查看对象的状态：</p><pre><code class="language-plain">kubectl get ingressclass
kubectl get ing
</code></pre><p><img src="https://static001.geekbang.org/resource/image/f9/b9/f9396112f84076528d9072e358d1ebb9.png?wh=1510x366" alt="图片"></p><p>命令 <code>kubectl describe</code> 可以看到更详细的Ingress信息：</p><pre><code class="language-plain">kubectl describe ing ngx-ing
</code></pre><p><img src="https://static001.geekbang.org/resource/image/b7/13/b708b7d41ef44844af7bf02cbb334313.png?wh=1576x664" alt="图片"></p><p>可以看到，Ingress对象的路由规则Host/Path就是在YAML里设置的域名“ngx.test/”，而且已经关联了第20讲里创建的Service对象，还有Service后面的两个Pod。</p><p>另外，不要对Ingress里“Default backend”的错误提示感到惊讶，在找不到路由的时候，它被设计用来提供一个默认的后端服务，但不设置也不会有什么问题，所以大多数时候我们都忽略它。</p><h2>如何在Kubernetes里使用Ingress Controller</h2><p>准备好了Ingress和Ingress Class，接下来我们就需要部署真正处理路由规则的Ingress Controller。</p><p>你可以在GitHub上找到Nginx Ingress Controller的项目（<a href="https://github.com/nginxinc/kubernetes-ingress">https://github.com/nginxinc/kubernetes-ingress</a>），因为它以Pod的形式运行在Kubernetes里，所以同时支持Deployment和DaemonSet两种部署方式。这里我选择的是Deployment，相关的YAML也都在我们课程的项目（<a href="https://github.com/chronolaw/k8s_study/tree/master/ingress">https://github.com/chronolaw/k8s_study/tree/master/ingress</a>）里复制了一份。</p><p>Nginx Ingress Controller的安装略微麻烦一些，有很多个YAML需要执行，但如果只是做简单的试验，就只需要用到4个YAML：</p><pre><code class="language-plain">kubectl apply -f common/ns-and-sa.yaml
kubectl apply -f rbac/rbac.yaml
kubectl apply -f common/nginx-config.yaml
kubectl apply -f common/default-server-secret.yaml
</code></pre><p>前两条命令为Ingress Controller创建了一个独立的名字空间“nginx-ingress”，还有相应的账号和权限，这是为了访问apiserver获取Service、Endpoint信息用的；后两条则是创建了一个ConfigMap和Secret，用来配置HTTP/HTTPS服务。</p><p>部署Ingress Controller不需要我们自己从头编写Deployment，Nginx已经为我们提供了示例YAML，但创建之前为了适配我们自己的应用还必须要做几处小改动：</p><ul>
<li>metadata里的name要改成自己的名字，比如 <code>ngx-kic-dep</code>。</li>
<li>spec.selector和template.metadata.labels也要修改成自己的名字，比如还是用 <code>ngx-kic-dep</code>。</li>
<li>containers.image可以改用apline版本，加快下载速度，比如 <code>nginx/nginx-ingress:2.2-alpine</code>。</li>
<li>最下面的args要加上 <code>-ingress-class=ngx-ink</code>，也就是前面创建的Ingress Class的名字，这是让Ingress Controller管理Ingress的关键。</li>
</ul><p>修改完之后，Ingress Controller的YAML大概是这个样子：</p><pre><code class="language-yaml">apiVersion: apps/v1
kind: Deployment
metadata:
&nbsp; name: ngx-kic-dep
&nbsp; namespace: nginx-ingress

spec:
&nbsp; replicas: 1
&nbsp; selector:
&nbsp; &nbsp; matchLabels:
&nbsp; &nbsp; &nbsp; app: ngx-kic-dep

&nbsp; template:
&nbsp; &nbsp; metadata:
&nbsp; &nbsp; &nbsp; labels:
&nbsp; &nbsp; &nbsp; &nbsp; app: ngx-kic-dep
    ...
&nbsp; &nbsp; spec:
&nbsp; &nbsp; &nbsp; containers:
&nbsp; &nbsp; &nbsp; - image: nginx/nginx-ingress:2.2-alpine
        ...
&nbsp; &nbsp; &nbsp; &nbsp; args:
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; - -ingress-class=ngx-ink
</code></pre><p>有了Ingress Controller，这些API对象的关联就更复杂了，你可以用下面的这张图来看出它们是如何使用对象名字联系起来的：</p><p><img src="https://static001.geekbang.org/resource/image/bb/14/bb7a911e10c103fb839e01438e184914.jpg?wh=1920x736" alt="图片"></p><p>确认Ingress Controller 的YAML修改完毕之后，就可以用 <code>kubectl apply</code> 创建对象：</p><pre><code class="language-plain">kubectl apply -f kic.yml
</code></pre><p>注意Ingress Controller位于名字空间“<strong>nginx-ingress</strong>”，所以查看状态需要用“<strong>-n</strong>”参数显式指定，否则我们只能看到“default”名字空间里的Pod：</p><pre><code class="language-plain">kubectl get deploy -n nginx-ingress
kubectl get pod -n nginx-ingress
</code></pre><p><img src="https://static001.geekbang.org/resource/image/63/a6/6389033863c8f809b4c0048be44903a6.png?wh=1476x364" alt="图片"></p><p>现在Ingress Controller就算是运行起来了。</p><p>不过还有最后一道工序，因为Ingress Controller本身也是一个Pod，想要向外提供服务还是要依赖于Service对象。所以你至少还要再为它定义一个Service，使用NodePort或者LoadBalancer暴露端口，才能真正把集群的内外流量打通。这个工作就交给你课下自己去完成了。</p><p>这里，我就用<a href="https://time.geekbang.org/column/article/534644">第15讲</a>里提到的<strong>命令</strong><code>kubectl port-forward</code><strong>，它可以直接把本地的端口映射到Kubernetes集群的某个Pod里</strong>，在测试验证的时候非常方便。</p><p>下面这条命令就把本地的8080端口映射到了Ingress Controller Pod的80端口：</p><pre><code class="language-plain">kubectl port-forward -n nginx-ingress ngx-kic-dep-8859b7b86-cplgp 8080:80 &amp;
</code></pre><p><img src="https://static001.geekbang.org/resource/image/1f/67/1f9cyy6e78d19e23db9594a272fa4267.png?wh=1920x349" alt="图片"></p><p>我们在curl发测试请求的时候需要注意，因为Ingress的路由规则是HTTP协议，所以就不能用IP地址的方式访问，必须要用域名、URI。</p><p>你可以修改 <code>/etc/hosts</code> 来手工添加域名解析，也可以使用 <code>--resolve</code> 参数，指定域名的解析规则，比如在这里我就把“ngx.test”强制解析到“127.0.0.1”，也就是被 <code>kubectl port-forward</code> 转发的本地地址：</p><pre><code class="language-plain">curl --resolve ngx.test:8080:127.0.0.1 http://ngx.test:8080
</code></pre><p><img src="https://static001.geekbang.org/resource/image/24/ec/2410bb40faa73be25e8d9b3c46c6deec.png?wh=1920x767" alt="图片"></p><p>把这个访问结果和上一节课里的Service对比一下，你会发现最终效果是一样的，都是把请求转发到了集群内部的Pod，但Ingress的路由规则不再是IP地址，而是HTTP协议里的域名、URI等要素。</p><h2>小结</h2><p>好了，今天就讲到这里，我们学习了Kubernetes里七层的反向代理和负载均衡对象，包括Ingress、Ingress Controller、Ingress Class，它们联合起来管理了集群的进出流量，是集群入口的总管。</p><p>小结一下今天的主要内容：</p><ol>
<li>Service是四层负载均衡，能力有限，所以就出现了Ingress，它基于HTTP/HTTPS协议定义路由规则。</li>
<li>Ingress只是规则的集合，自身不具备流量管理能力，需要Ingress Controller应用Ingress规则才能真正发挥作用。</li>
<li>Ingress Class解耦了Ingress和Ingress Controller，我们应当使用Ingress Class来管理Ingress资源。</li>
<li>最流行的Ingress Controller是Nginx Ingress Controller，它基于经典反向代理软件Nginx。</li>
</ol><p>再补充一点，目前的Kubernetes流量管理功能主要集中在Ingress Controller上，已经远不止于管理“入口流量”了，它还能管理“出口流量”，也就是 <code>egress</code>，甚至还可以管理集群内部服务之间的“东西向流量”。</p><p>此外，Ingress Controller通常还有很多的其他功能，比如TLS终止、网络应用防火墙、限流限速、流量拆分、身份认证、访问控制等等，完全可以认为它是一个全功能的反向代理或者网关，感兴趣的话你可以找找这方面的资料。</p><h2>课下作业</h2><p>最后是课下作业时间，给你留两个思考题：</p><ol>
<li>四层负载均衡（Service）与七层负载均衡（Ingress）有哪些异同点？</li>
<li>你认为Ingress Controller作为集群的流量入口还应该做哪些事情？</li>
</ol><p>欢迎留言写下你的想法，思考题闭环是你巩固所学的第一步，进步从完成开始。</p><p>下节课是我们这个章节的实战演练课，我们下节课再见。</p><p><img src="https://static001.geekbang.org/resource/image/6a/08/6a373b5b5e8c0869f6b77bc8d5b35708.jpg?wh=1920x2856" alt=""></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/14/b9/47377590.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Jasper</span>
  </div>
  <div class="_2_QraFYR_0">四层架构简单，无需解析消息内容，在网络吞吐量及处理性能上高于七层。<br>而七层负载优势在于功能多，控制灵活强大。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-08 10:41:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/bb/69/a8bb7a4a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Xu.</span>
  </div>
  <div class="_2_QraFYR_0">老师，我在安装文档里找到了大多数同学遇到的问题的解决方法：<br>https:&#47;&#47;docs.nginx.com&#47;nginx-ingress-controller&#47;installation&#47;installation-with-manifests&#47;<br>Create Custom Resources 这一节<br><br>注意：默认情况下，需要为 VirtualServer, VirtualServerRoute, TransportServer and Policy 创建自定义资源的定义。否则，Ingress Controller Pod 将不会变为 Ready 状态。如果要禁用该要求，请将 -enable-custom-resources 命令行参数配置为 Readyfalse 并跳过此部分。<br><br>也就是说可以 kubectl apply -f 下面几个文件：<br><br>$ kubectl apply -f common&#47;crds&#47;k8s.nginx.org_virtualservers.yaml<br>$ kubectl apply -f common&#47;crds&#47;k8s.nginx.org_virtualserverroutes.yaml<br>$ kubectl apply -f common&#47;crds&#47;k8s.nginx.org_transportservers.yaml<br>$ kubectl apply -f common&#47;crds&#47;k8s.nginx.org_policies.yaml<br><br>然后就启动成功了。<br><br>也可以将 -enable-custom-resources 命令行参数配置为 Readyfalse </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good solution！ </p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-22 10:39:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/b3/d9/cf061262.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>新时代农民工</span>
  </div>
  <div class="_2_QraFYR_0">文末的kic.yml是来自 https:&#47;&#47;github.com&#47;nginxinc&#47;kubernetes-ingress&#47;blob&#47;main&#47;deployments&#47;deployment&#47;nginx-ingress.yaml</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-09 00:01:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/oib0a89lqtOhJL1UvfUp4uTsRLrDbhoGk9jLiciazxMu0COibJsFCZDypK1ZFcHEJc9d9qgbjvgR41ImL6FNPoVlWA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>stefen</span>
  </div>
  <div class="_2_QraFYR_0">最后ingress-controller运行起来的pod 可以看作是一个pod的nginx反向代理的VIP， 由于pod网络隔离的原因，需要还套娃一个service, 对外提供统一的管理入口，是否可以换种思路， 在启动这种ingress-controller运行起来的pod的，设置pod的网络为host，就是公用宿主机网卡，这样就不用套娃service了.</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以的，后面的实战就是这么做的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-28 23:40:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/63/9a/6872c932.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Grey</span>
  </div>
  <div class="_2_QraFYR_0">Nginx Ingress Controller 只用那4个不行，看了23节，跟着老师用bash脚本全部弄进去了才把kic起了起来</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 脚本里用的YAML 比较多，还创建了crd资源。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-14 21:55:31</div>
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
  <div class="_2_QraFYR_0">service方式如下：<br><br>apiVersion: v1<br>kind: Service<br>metadata:<br>  name: ingress-svc<br>  namespace: nginx-ingress<br>spec:<br>  selector:<br>    app: ngx-kic-dep<br>  ports:<br>  - port: 80<br>    targetPort: 80<br>  type: NodePort<br><br><br>请求 后面的端口要根据kubectl get svc -n nginx-ingress 查看<br><br>curl --resolve ngx.test:31967:127.0.0.1 http:&#47;&#47;ngx.test:31967</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-12 18:40:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/44/d8/708a0932.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>李一</span>
  </div>
  <div class="_2_QraFYR_0">老师，请问 Ingress 工作在7层协议中，指针对http(s)应用层协议进行控制，那如果 我的应用是需要长链接的 不如IM通讯相关，那是不是Ingress就无法满足了，只能通过service 定义吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 四层协议可以由Ingress Controller的crd资源来定义，比如Nginx Ingress Controller的Transport Server，可以参考它们的相关文档，比如：https:&#47;&#47;docs.nginx.com&#47;nginx-ingress-controller&#47;configuration&#47;transportserver-resource&#47;</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-08 11:29:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/28/1b/f9/018197f1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小江爱学术</span>
  </div>
  <div class="_2_QraFYR_0">一个小问题老师，service基于四层转发，会暴露ip。基于这些缺点我们引入了ingress，基于七层网络协议转发，但是为了外部服务访问，需要在ingress前再暴露一个nodeport类型的service，那我们这么做的意义在哪里捏，最外层的入口处不还是service吗。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Ingress基于http协议，有更丰富的路由规则，能够更精细地管理流量。而且有了Ingress，只需要对外暴露一个Service对象就可以了，也更加安全。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-06 10:07:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2c/c7/89/16437396.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>极客酱酱</span>
  </div>
  <div class="_2_QraFYR_0">为ingres-controller设置Service:<br>➜  ingress kubectl expose deploy nginx-kic-dep -n nginx-ingress --port=80 --target-port=80 $=out&gt;ingress-svc.yml<br><br>➜  ingress cat ingress-svc.yml     <br><br>apiVersion: v1<br>kind: Service<br>metadata:<br>  name: nginx-kic-svc<br>  namespace: nginx-ingress<br>spec:<br>  ports:<br>  - port: 80<br>    protocol: TCP<br>    targetPort: 80<br>  selector:<br>    app: nginx-kic-dep<br>  type: NodePort<br><br>➜  ingress kubectl get svc -n nginx-ingress<br>NAME            TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE<br>nginx-kic-svc   NodePort   10.105.174.176   &lt;none&gt;        80:32519&#47;TCP   3m41s<br><br>➜  ingress kubectl get node -o wide        <br>NAME     STATUS   ROLES                  AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE           KERNEL-VERSION      CONTAINER-RUNTIME<br>ubuntu   Ready    control-plane,master   4d    v1.23.3   10.211.55.5   &lt;none&gt;        Ubuntu 22.04 LTS   5.15.0-58-generic   docker:&#47;&#47;20.10.12<br>worker   Ready    &lt;none&gt;                 4d    v1.23.3   10.211.55.6   &lt;none&gt;        Ubuntu 22.04 LTS   5.15.0-58-generic   docker:&#47;&#47;20.10.12<br><br>➜  ingress curl --resolve ngx.test:32519:10.211.55.5 http:&#47;&#47;ngx.test:32519<br>srv : 10.10.1.10:80<br>host: ngx-dep-6796688696-867dm<br>uri : GET ngx.test &#47;<br>date: 2023-02-09T15:10:48+00:00<br><br>➜  ingress curl --resolve ngx.test:32519:10.211.55.5 http:&#47;&#47;ngx.test:32519<br>srv : 10.10.1.11:80<br>host: ngx-dep-6796688696-psp5v<br>uri : GET ngx.test &#47;<br>date: 2023-02-09T15:10:50+00:00</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-09 23:13:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/ibZVAmmdAibBeVpUjzwId8ibgRzNk7fkuR5pgVicB5mFSjjmt2eNadlykVLKCyGA0GxGffbhqLsHnhDRgyzxcKUhjg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>pyhhou</span>
  </div>
  <div class="_2_QraFYR_0">老师，setup Ingress Controller 那里有个问题不知道如何解决，看到评论区里也有很多同学有一样的问题<br><br>$ kubectl logs ngx-kic-dep-75f4f5c7c-v9lt8 -n nginx-ingress<br>I0907 22:10:01.222921       1 main.go:213] Starting NGINX Ingress Controller Version=2.2.2 GitCommit=a88b7fe6dbde5df79593ac161749afc1e9a009c6 Date=2022-05-24T00:33:34Z Arch=linux&#47;arm64 PlusFlag=false<br>I0907 22:10:01.233010       1 main.go:344] Kubernetes version: 1.23.3<br>F0907 22:10:01.233818       1 main.go:357] Error when getting IngressClass ngx-ink: ingressclasses.networking.k8s.io &quot;ngx-ink&quot; is forbidden: User &quot;system:serviceaccount:nginx-ingress:default&quot; cannot get resource &quot;ingressclasses&quot; in API group &quot;networking.k8s.io&quot; at the cluster scope<br><br>pod 的状态是 CrashLoopBackOff<br><br>我是直接执行你 GitHub 上面的 setup.sh 脚本的，然后再执行 kic.yaml 的，所以 rbac.yaml 也是执行了的。或者说 rbac.yaml 文件中的参数需要更改？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 应该先创建Ingress Class对象吧，可以参考一下后面的视频课。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-08 06:27:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2a/92/4b/1262f052.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>许飞</span>
  </div>
  <div class="_2_QraFYR_0">F0818 12:36:23.718350       1 main.go:357] Error when getting IngressClass ngx-ink: ingressclasses.networking.k8s.io &quot;ngx-ink&quot; is forbidden: User &quot;system:serviceaccount:nginx-ingress:default&quot; cannot get resource &quot;ingressclasses&quot; in API group &quot;networking.k8s.io&quot; at the cluster scope<br>老师这报错是为啥</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 好像是没有安装Nginx Ingress Controller的rbac，再把它相关的安装YAML 都执行一下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-18 20:37:45</div>
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
  <div class="_2_QraFYR_0">请教老师几个问题：<br>Q1：taint是在daemonset部分讲的，节点的taint会影响pod是否会部署到节点上。<br>但是，对于deployment，taint会影响吗？<br><br>Q2：第21讲最后一部分是搭建ingress controller。 这部分提到了五个yml文件。 是先执行前面四个，然后再执行最后一个来创建ingress controller吗？ 还是只执行“kubectl apply -f kic.yml”？ （如果先执行前面四个文件，这四个文件有顺序吗？）<br><br>Q3：github上的四个文件，看不到下载按钮。是因为没有登录吗？ 如果不是因为登录，是否能提供下载按钮？（现在的方法是拷贝文件内容，比较麻烦）<br><br>Q4：运行controller后，查看deploy和POD的状态都不正常。<br>先执行github上的四个文件，顺序是文中列出的顺序，创建都成功了。然后启动controller。<br>查看结果如下：<br>kubectl get pod -n nginx-ingress，状态是：CrashLoopBackOff<br>kubectl get deploy -n nginx-ingress： Ready是 0&#47;1<br>可能是什么原因？ （这个问题有点笼统，不太具体，老师给出一点建议即可。）</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1. taint是对pod生效的，所以也会影响Deployment。<br><br>2.前面4个是安装Nginx Ingress Controller，是必须要做的，有顺序要求。<br><br>3.用git clone，把这个项目下载下来。<br><br>4.用kubectl describe&#47;logs看原因。<br><br>遇到错误是好事，排查错误的过程就能够涨经验。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-09 20:51:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/9c/31/e4677275.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>潜光隐耀</span>
  </div>
  <div class="_2_QraFYR_0">请问下，会不会介绍下gateway api？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 现在Gateway 已经是beta版了，可能会以加餐的形式，</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-08 09:11:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/fd/30/83badcc2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Albert</span>
  </div>
  <div class="_2_QraFYR_0">service文件<br>apiVersion: v1<br>kind: Service<br>metadata:<br>  name: ngx-kic-svc<br>  namespace: nginx-ingress<br><br>spec:<br>  selector:<br>    app: ngx-kic-dep<br>  type: NodePort<br>  ports:<br>  - port: 8082<br>    targetPort: 80<br>    protocol: TCP<br>~                   <br><br>查看svc的ip后访问：<br>curl -H &quot;Host: ngx.test&quot; &quot;http:&#47;&#47;10.103.79.195:8082&quot;<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-23 19:06:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/fd/30/83badcc2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Albert</span>
  </div>
  <div class="_2_QraFYR_0">可能版本更新，建议按照官方教程来：https:&#47;&#47;docs.nginx.com&#47;nginx-ingress-controller&#47;installation&#47;installation-with-manifests&#47;<br><br>需要8个文件<br><br>kubectl apply -f common&#47;ns-and-sa.yaml<br>kubectl apply -f rbac&#47;rbac.yaml<br>kubectl apply -f common&#47;nginx-config.yaml<br>kubectl apply -f common&#47;default-server-secret.yaml<br><br>kubectl apply -f common&#47;crds&#47;k8s.nginx.org_virtualservers.yaml<br>kubectl apply -f common&#47;crds&#47;k8s.nginx.org_virtualserverroutes.yaml<br>kubectl apply -f common&#47;crds&#47;k8s.nginx.org_transportservers.yaml<br>kubectl apply -f common&#47;crds&#47;k8s.nginx.org_policies.yaml<br><br>其中rbac&#47;rbac.yaml建议使用最新版https:&#47;&#47;github.com&#47;nginxinc&#47;kubernetes-ingress&#47;blob&#47;main&#47;deployments&#47;rbac&#47;rbac.yaml，否则pod无法启动，报错：endpointslices.discovery.k8s.io is forbidden<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-23 17:39:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_e46db2</span>
  </div>
  <div class="_2_QraFYR_0">如果大家遇到apply kic.yml无法ready，pod日志显示： manager.go:335] Failed to get nginx version: fork&#47;exec &#47;usr&#47;sbin&#47;nginx: operation not permitted，时候。可以看下老师后面实操视频，需要修改容器capabilities配置，添加NET_BIND_SERVICE属性。<br>capabilities:<br>            drop:<br>            - ALL<br>            add:<br>            - NET_BIND_SERVICE</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-26 23:57:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_e46db2</span>
  </div>
  <div class="_2_QraFYR_0">kubectl logs ngx-kic-dep-79797d5ccf-crw7k -n nginx-ingress<br>NGINX Ingress Controller Version=3.0.2 Commit=40e3a2bc24a0b158938c1e1c5e5f8db5feec624e Date=2023-02-14T11:27:05Z DirtyState=false Arch=linux&#47;amd64 Go=go1.19.7<br>I0326 15:28:47.288487       1 flags.go:294] Starting with flags: [&quot;-ingress-class=ngx-ink&quot; &quot;-enable-custom-resources=false&quot; &quot;-nginx-configmaps=nginx-ingress&#47;nginx-config&quot;]<br>I0326 15:28:47.302018       1 main.go:227] Kubernetes version: 1.23.3<br>F0326 15:28:47.303848       1 manager.go:339] Failed to get nginx version: fork&#47;exec &#47;usr&#47;sbin&#47;nginx: operation not permitted<br><br>---<br>这个问题是为啥呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 好像自己解决了，good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-26 23:29:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/7e/25/3932dafd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>GeekNEO</span>
  </div>
  <div class="_2_QraFYR_0">之前的Service的ngx-svc修改为ClusterIP，然后把ingress-svc的type修改为NodePort，ingress-svc如下：<br>apiVersion: v1<br>kind: Service<br>metadata:<br>  name: ingress-svc<br>  namespace: nginx-ingress<br>spec:<br>  type: NodePort<br>  selector:<br>    app: ngx-kic-dep<br>  ports:<br>  - port: 80<br>    protocol: TCP<br>    targetPort: 80<br><br>查看生成的Service命令：kubectl get svc -n nginx-ingress  可以看到映射端口31404 外部端口是31404。<br>curl模拟访问：<br>curl --resolve ngx.test:31404:127.0.0.1 http:&#47;&#47;ngx.test:31404<br>正确返回结果：之前定义的cm对象写入的nginx location定义的信息<br>srv : 10.10.1.8:80<br>host: ngx-dep-67c5b96797-r8kjm<br>uri : GET ngx.test &#47;<br>date: 2023-03-26T05:56:28+00:00<br>或者：<br>srv : 10.10.2.8:80<br>host: ngx-dep-67c5b96797-vspwr<br>uri : GET ngx.test &#47;<br>date: 2023-03-26T05:56:28+00:00<br><br>返回的pod信息是之前创建的2台pod，负载均衡应该是还是轮询，虽然前面加了nginx-ingress，但是后端对应的Service还是一个，所以最终还是RR，老师不知道我说的可对？<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Nginx默认使用round robin算法，可以在YAML 里改成其他的算法。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-26 14:03:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/7e/25/3932dafd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>GeekNEO</span>
  </div>
  <div class="_2_QraFYR_0">我发现了一个问题，我是用了最新版的yml，从github拉的，没有直接用老师的，但是kic.yml文件就是使用老师贴出来的，然后运行起来，pod一直都是未READY和AVAILABLE，查看日志logs，报错：<br>Failed to watch *v1.Endpoints: failed to list *v1.Endpoints: endpoints is forbidden: User &quot;system:serviceaccount:nginx-ingress:nginx-ingress&quot; cannot list resource &quot;endpoints&quot; in API group &quot;&quot; at the cluster scope<br><br>我排查了很久，最后发现rabc文件规则异同，导致版本不兼容，不知道我猜测的对不对？<br>把老师的镜像文件 nginx&#47;nginx-ingress:2.2-alpine更换为nginx&#47;nginx-ingress:3.0.2<br>再次apply后，就一切正常OK了，没问题了。<br>总结：因没有拉老师提供的yml清单生成对应的对象，而是自己去github拉最新版本的yml生成对象，而镜像文件使用了老师提供，导致pod起不来。目测是版本对rabc规则的不兼容导致，不知道老师可以解答下吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，镜像和YAML 必须对应，否则会版本不匹配。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-26 02:20:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/98/b1/f89a84d0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Tokamak</span>
  </div>
  <div class="_2_QraFYR_0">罗老师，我现在遇到一个问题，使用 kubectl get pod 是可以看到pod的, 假设该pod的名称是poda, 使用 kubectl describe pod poda 也是可以看到正常信息的，使用docker exec -ti &lt;container-id&gt; sh 是可以进入该容器的， 但是使用 kubectl exec poda -- sh 提示 error: unable to upgrade connection: pod does not exist<br><br>我的环境是: Virtualbox 运行的两台 ubuntu 22.04, 使用的是 kubeadm, k8s=v1.23.3；<br>希望老师能给个思路，如何解决呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 在stackoverflow上搜到了一个类似的问题，可以参考。<br>https:&#47;&#47;stackoverflow.com&#47;questions&#47;51154911&#47;kubectl-exec-results-in-error-unable-to-upgrade-connection-pod-does-not-exi</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-25 18:34:57</div>
  </div>
</div>
</div>
</li>
</ul>