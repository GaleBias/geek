<audio title="22｜实战演练：玩转Kubernetes（2）" src="https://static001.geekbang.org/resource/audio/84/9c/84e6eb32c3be589241cdaf32e9dab99c.mp3" controls="controls"></audio> 
<p>你好，我是Chrono。</p><p>我们的“中级篇”到今天马上就要结束了，感谢你这段时间坚持不懈的学习。</p><p>作为“中级篇”的收尾课程，我照例还是会对前面学过的内容做一个全面的回顾和总结，把知识点都串联起来，加深你对它们的印象。</p><p>下面我先梳理一下“中级篇”里讲过的Kubernetes知识要点，然后是实战演示，搭建WordPress网站。当然这次比前两次又有进步，不用Docker，也不用裸Pod，而是用我们新学习的Deployment、Service、Ingress等对象。</p><h2>Kubernetes技术要点回顾</h2><p>Kubernetes是云原生时代的操作系统，它能够管理大量节点构成的集群，让计算资源“池化”，从而能够自动地调度运维各种形式的应用。</p><p>搭建多节点的Kubernetes集群是一件颇具挑战性的工作，好在社区里及时出现了kubeadm这样的工具，可以“一键操作”，使用 <code>kubeadm init</code>、<code>kubeadm join</code> 等命令从无到有地搭建出生产级别的集群（<a href="https://time.geekbang.org/column/article/534762">17讲</a>）。</p><p>kubeadm使用容器技术封装了Kubernetes组件，所以只要节点上安装了容器运行时（Docker、containerd等），它就可以自动从网上拉取镜像，然后以容器的方式运行组件，非常简单方便。</p><!-- [[[read_end]]] --><p>在这个更接近实际生产环境的Kubernetes集群里，我们学习了<strong>Deployment、DaemonSet、Service、Ingress、Ingress Controller</strong>等API对象。</p><p>（<a href="https://time.geekbang.org/column/article/535209">18讲</a>）Deployment是用来管理Pod的一种对象，它代表了运维工作中最常见的一类在线业务，在集群中部署应用的多个实例，而且可以很容易地增加或者减少实例数量，从容应对流量压力。</p><p>Deployment的定义里有两个关键字段：一个是 <code>replicas</code>，它指定了实例的数量；另一个是 <code>selector</code>，它的作用是使用标签“筛选”出被Deployment管理的Pod，这是一种非常灵活的关联机制，实现了API对象之间的松耦合。</p><p>（<a href="https://time.geekbang.org/column/article/536803">19讲</a>）DaemonSet是另一种部署在线业务的方式，它很类似Deployment，但会在集群里的每一个节点上运行一个Pod实例，类似Linux系统里的“守护进程”，适合日志、监控等类型的应用。</p><p>DaemonSet能够任意部署Pod的关键概念是“污点”（taint）和“容忍度”（toleration）。Node会有各种“污点”，而Pod可以使用“容忍度”来忽略“污点”，合理使用这两个概念就可以调整Pod在集群里的部署策略。</p><p>（<a href="https://time.geekbang.org/column/article/536829">20讲</a>）由Deployment和DaemonSet部署的Pod，在集群中处于“动态平衡”的状态，总数量保持恒定，但也有临时销毁重建的可能，所以IP地址是变化的，这就为微服务等应用架构带来了麻烦。</p><p>Service是对Pod IP地址的抽象，它拥有一个固定的IP地址，再使用iptables规则把流量负载均衡到后面的Pod，节点上的kube-proxy组件会实时维护被代理的Pod状态，保证Service只会转发给健康的Pod。</p><p>Service还基于DNS插件支持域名，所以客户端就不再需要关心Pod的具体情况，只要通过Service这个稳定的中间层，就能够访问到Pod提供的服务。</p><p>（<a href="https://time.geekbang.org/column/article/538760">21讲</a>）Service是四层的负载均衡，但现在的绝大多数应用都是HTTP/HTTPS协议，要实现七层的负载均衡就要使用Ingress对象。</p><p>Ingress定义了基于HTTP协议的路由规则，但要让规则生效，还需要<strong>Ingress Controller</strong>和<strong>Ingress Class</strong>来配合工作。</p><ul>
<li>Ingress Controller是真正的集群入口，应用Ingress规则调度、分发流量，此外还能够扮演反向代理的角色，提供安全防护、TLS卸载等更多功能。</li>
<li>Ingress Class是用来管理Ingress和Ingress Controller的概念，方便我们分组路由规则，降低维护成本。</li>
</ul><p>不过Ingress Controller本身也是一个Pod，想要把服务暴露到集群外部还是要依靠Service。Service支持NodePort、LoadBalancer等方式，但NodePort的端口范围有限，LoadBalancer又依赖于云服务厂商，都不是很灵活。</p><p>折中的办法是用少量NodePort暴露Ingress Controller，用Ingress路由到内部服务，外部再用反向代理或者LoadBalancer把流量引进来。</p><h2>WordPress网站基本架构</h2><p>简略回顾了Kubernetes里这些API对象，下面我们就来使用它们再搭建出WordPress网站，实践加深理解。</p><p>既然我们已经掌握了Deployment、Service、Ingress这些Pod之上的概念，网站自然会有新变化，架构图我放在了这里：</p><p><img src="https://static001.geekbang.org/resource/image/96/07/9634b8850c3abf62047689b885d7ef07.jpg?wh=1920x1138" alt="图片"></p><p>这次的部署形式比起Docker、minikube又有了一些细微的差别，<strong>重点是我们已经完全舍弃了Docker，把所有的应用都放在Kubernetes集群里运行，部署方式也不再是裸Pod，而是使用Deployment，稳定性大幅度提升</strong>。</p><p>原来的Nginx的作用是反向代理，那么在Kubernetes里它就升级成了具有相同功能的Ingress Controller。WordPress原来只有一个实例，现在变成了两个实例（你也可以任意横向扩容），可用性也就因此提高了不少。而MariaDB数据库因为要保证数据的一致性，暂时还是一个实例。</p><p>还有，因为Kubernetes内置了服务发现机制Service，我们再也不需要去手动查看Pod的IP地址了，只要为它们定义Service对象，然后使用域名就可以访问MariaDB、WordPress这些服务。</p><p>网站对外提供服务我选择了两种方式。</p><p>一种是让WordPress的Service对象以NodePort的方式直接对外暴露端口30088，方便测试；另一种是给Nginx Ingress Controller添加“hostNetwork”属性，直接使用节点上的端口号，类似Docker的host网络模式，好处是可以避开NodePort的端口范围限制。</p><p>下面我们就按照这个基本架构来逐步搭建出新版本的WordPress网站，编写YAML声明。</p><p>这里有个小技巧，在实际操作的时候你一定要记得善用 <code>kubectl create</code>、<code>kubectl expose</code> 创建样板文件，节约时间的同时，也能避免低级的格式错误。</p><h2>1. WordPress网站部署MariaDB</h2><p>首先我们还是要部署MariaDB，这个步骤和在<a href="https://time.geekbang.org/column/article/534644">第15讲</a>里做的也差不多。</p><p>先要用ConfigMap定义数据库的环境变量，有 <code>DATABASE</code>、<code>USER</code>、<code>PASSWORD</code>、<code>ROOT_PASSWORD</code>：</p><pre><code class="language-yaml">apiVersion: v1
kind: ConfigMap
metadata:
&nbsp; name: maria-cm

data:
&nbsp; DATABASE: 'db'
&nbsp; USER: 'wp'
&nbsp; PASSWORD: '123'
&nbsp; ROOT_PASSWORD: '123'
</code></pre><p>然后我们需要<strong>把MariaDB由Pod改成Deployment的方式</strong>，replicas设置成1个，template里面的Pod部分没有任何变化，还是要用 <code>envFrom</code>把配置信息以环境变量的形式注入Pod，相当于把Pod套了一个Deployment的“外壳”：</p><pre><code class="language-yaml">apiVersion: apps/v1
kind: Deployment
metadata:
&nbsp; labels:
&nbsp; &nbsp; app: maria-dep
&nbsp; name: maria-dep

spec:
&nbsp; replicas: 1
&nbsp; selector:
&nbsp; &nbsp; matchLabels:
&nbsp; &nbsp; &nbsp; app: maria-dep

&nbsp; template:
&nbsp; &nbsp; metadata:
&nbsp; &nbsp; &nbsp; labels:
&nbsp; &nbsp; &nbsp; &nbsp; app: maria-dep
&nbsp; &nbsp; spec:
&nbsp; &nbsp; &nbsp; containers:
&nbsp; &nbsp; &nbsp; - image: mariadb:10
&nbsp; &nbsp; &nbsp; &nbsp; name: mariadb
&nbsp; &nbsp; &nbsp; &nbsp; ports:
&nbsp; &nbsp; &nbsp; &nbsp; - containerPort: 3306

&nbsp; &nbsp; &nbsp; &nbsp; envFrom:
&nbsp; &nbsp; &nbsp; &nbsp; - prefix: 'MARIADB_'
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; configMapRef:
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; name: maria-cm
</code></pre><p>我们还需要再<strong>为MariaDB定义一个Service对象</strong>，映射端口3306，让其他应用不再关心IP地址，直接用Service对象的名字来访问数据库服务：</p><pre><code class="language-yaml">apiVersion: v1
kind: Service
metadata:
&nbsp; labels:
&nbsp; &nbsp; app: maria-dep
&nbsp; name: maria-svc

spec:
&nbsp; ports:
&nbsp; - port: 3306
&nbsp; &nbsp; protocol: TCP
&nbsp; &nbsp; targetPort: 3306
&nbsp; selector:
&nbsp; &nbsp; app: maria-dep
</code></pre><p>因为这三个对象都是数据库相关的，所以可以在一个YAML文件里书写，对象之间用 <code>---</code> 分开，这样用 <code>kubectl apply</code> 就可以一次性创建好：</p><pre><code class="language-plain">kubectl apply&nbsp;-f wp-maria.yml
</code></pre><p>执行命令后，你应该用 <code>kubectl get</code> 查看对象是否创建成功，是否正常运行：</p><p><img src="https://static001.geekbang.org/resource/image/2f/a4/2fa0050d5bf61224dc17dc67f6da16a4.png?wh=1664x852" alt="图片"></p><h2>2. WordPress网站部署WordPress</h2><p>第二步是部署WordPress应用。</p><p>因为刚才创建了MariaDB的Service，所以在写ConfigMap配置的时候“HOST”就不应该是IP地址了，而<strong>应该是DNS域名，也就是Service的名字</strong><code>maria-svc</code><strong>，这点需要特别注意</strong>：</p><pre><code class="language-yaml">apiVersion: v1
kind: ConfigMap
metadata:
&nbsp; name: wp-cm

data:
&nbsp; HOST: 'maria-svc'
&nbsp; USER: 'wp'
&nbsp; PASSWORD: '123'
&nbsp; NAME: 'db'
</code></pre><p>WordPress的Deployment写法和MariaDB也是一样的，给Pod套一个Deployment的“外壳”，replicas设置成2个，用字段“<strong>envFrom</strong>”配置环境变量：</p><pre><code class="language-yaml">apiVersion: apps/v1
kind: Deployment
metadata:
&nbsp; labels:
&nbsp; &nbsp; app: wp-dep
&nbsp; name: wp-dep

spec:
&nbsp; replicas: 2
&nbsp; selector:
&nbsp; &nbsp; matchLabels:
&nbsp; &nbsp; &nbsp; app: wp-dep

&nbsp; template:
&nbsp; &nbsp; metadata:
&nbsp; &nbsp; &nbsp; labels:
&nbsp; &nbsp; &nbsp; &nbsp; app: wp-dep
&nbsp; &nbsp; spec:
&nbsp; &nbsp; &nbsp; containers:
&nbsp; &nbsp; &nbsp; - image: wordpress:5
&nbsp; &nbsp; &nbsp; &nbsp; name: wordpress
&nbsp; &nbsp; &nbsp; &nbsp; ports:
&nbsp; &nbsp; &nbsp; &nbsp; - containerPort: 80

&nbsp; &nbsp; &nbsp; &nbsp; envFrom:
&nbsp; &nbsp; &nbsp; &nbsp; - prefix: 'WORDPRESS_DB_'
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; configMapRef:
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; name: wp-cm
</code></pre><p>然后我们仍然要为WordPress创建Service对象，这里我使用了“<strong>NodePort</strong>”类型，并且手工指定了端口号“30088”（必须在30000~32767之间）：</p><pre><code class="language-yaml">apiVersion: v1
kind: Service
metadata:
&nbsp; labels:
&nbsp; &nbsp; app: wp-dep
&nbsp; name: wp-svc

spec:
&nbsp; ports:
&nbsp; - name: http80
&nbsp; &nbsp; port: 80
&nbsp; &nbsp; protocol: TCP
&nbsp; &nbsp; targetPort: 80
&nbsp; &nbsp; nodePort: 30088

&nbsp; selector:
&nbsp; &nbsp; app: wp-dep
&nbsp; type: NodePort
</code></pre><p>现在让我们用 <code>kubectl apply</code> 部署WordPress：</p><pre><code class="language-plain">kubectl apply&nbsp; -f wp-dep.yml
</code></pre><p>这些对象的状态可以从下面的截图看出来：</p><p><img src="https://static001.geekbang.org/resource/image/da/4f/daeb742118aca29577e1af91c89aff4f.png?wh=1920x1022" alt="图片"></p><p>因为WordPress的Service对象是NodePort类型的，我们可以在集群的每个节点上访问WordPress服务。</p><p>比如一个节点的IP地址是“192.168.10.210”，那么你就在浏览器的地址栏里输入“<a href="http://192.168.10.210:30088">http://192.168.10.210:30088</a>”，其中的“30088”就是在Service里指定的节点端口号，然后就能够看到WordPress的安装界面了：</p><p><img src="https://static001.geekbang.org/resource/image/1f/38/1f5fa840441d52e94d3e7609a3bb9438.png?wh=874x1162" alt="图片"></p><h2>3. WordPress网站部署Nginx Ingress Controller</h2><p>现在MariaDB，WordPress都已经部署成功了，第三步就是部署Nginx Ingress Controller。</p><p>首先我们需要定义Ingress Class，名字就叫“wp-ink”，非常简单：</p><pre><code class="language-yaml">apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
&nbsp; name: wp-ink

spec:
&nbsp; controller: nginx.org/ingress-controller
</code></pre><p>然后用 <code>kubectl create</code> 命令生成Ingress的样板文件，指定域名是“wp.test”，后端Service是“wp-svc:80”，Ingress Class就是刚定义的“wp-ink”：</p><pre><code class="language-plain">kubectl create ing wp-ing --rule="wp.test/=wp-svc:80" --class=wp-ink $out
</code></pre><p>得到的Ingress YAML就是这样，注意路径类型我还是用的前缀匹配“<strong>Prefix</strong>”：</p><pre><code class="language-yaml">apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
&nbsp; name: wp-ing

spec:
&nbsp; ingressClassName: wp-ink

&nbsp; rules:
&nbsp; - host: wp.test
&nbsp; &nbsp; http:
&nbsp; &nbsp; &nbsp; paths:
&nbsp; &nbsp; &nbsp; - path: /
&nbsp; &nbsp; &nbsp; &nbsp; pathType: Prefix
&nbsp; &nbsp; &nbsp; &nbsp; backend:
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; service:
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; name: wp-svc
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; port:
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; number: 80
</code></pre><p>接下来就是<strong>最关键的Ingress Controller对象了，它仍然需要从Nginx项目的示例YAML修改而来，要改动名字、标签，还有参数里的Ingress Class</strong>。</p><p>在之前讲基本架构的时候我说过了，这个Ingress Controller不使用Service，而是给它的Pod加上一个特殊字段 <code>hostNetwork</code>，让Pod能够使用宿主机的网络，相当于另一种形式的NodePort：</p><pre><code class="language-yaml">apiVersion: apps/v1
kind: Deployment
metadata:
&nbsp; name: wp-kic-dep
&nbsp; namespace: nginx-ingress

spec:
&nbsp; replicas: 1
&nbsp; selector:
&nbsp; &nbsp; matchLabels:
&nbsp; &nbsp; &nbsp; app: wp-kic-dep

&nbsp; template:
&nbsp; &nbsp; metadata:
&nbsp; &nbsp; &nbsp; labels:
&nbsp; &nbsp; &nbsp; &nbsp; app: wp-kic-dep

&nbsp; &nbsp; spec:
&nbsp; &nbsp; &nbsp; serviceAccountName: nginx-ingress

&nbsp; &nbsp; &nbsp; # use host network
&nbsp; &nbsp; &nbsp; hostNetwork: true

&nbsp; &nbsp; &nbsp; containers:
      ...
</code></pre><p>准备好Ingress资源后，我们创建这些对象：</p><pre><code class="language-plain">kubectl apply -f wp-ing.yml -f wp-kic.yml
</code></pre><p><img src="https://static001.geekbang.org/resource/image/f5/6a/f57224b4c64ecc6b04651d1986406c6a.png?wh=1614x666" alt="图片"></p><p>现在所有的应用都已经部署完毕，可以在集群外面访问网站来验证结果了。</p><p>不过你要注意，Ingress使用的是HTTP路由规则，用IP地址访问是无效的，所以在集群外的主机上必须能够识别我们的“wp.test”域名，也就是说要把域名“wp.test”解析到Ingress Controller所在的节点上。</p><p>如果你用的是Mac，那就修改 <code>/etc/hosts</code>；如果你用的是Windows，就修改 <code>C:\Windows\System32\Drivers\etc\hosts</code>，添加一条解析规则就行：</p><pre><code class="language-plain">cat /etc/hosts
192.168.10.210&nbsp; wp.test
</code></pre><p>有了域名解析，在浏览器里你就不必使用IP地址，直接用域名“wp.test”走Ingress Controller就能访问我们的WordPress网站了：</p><p><img src="https://static001.geekbang.org/resource/image/e1/ea/e10a1f53d9d163e74a53fb18d81cf5ea.png?wh=1550x1098" alt="图片"></p><p>到这里，我们在Kubernetes上部署WordPress网站的工作就全部完成了。</p><h2>小结</h2><p>这节课我们回顾了“中级篇”里的一些知识要点，我把它们总结成了思维导图，你课后可以对照着它查缺补漏，巩固学习成果。</p><p><img src="https://static001.geekbang.org/resource/image/6c/7c/6c051e3c12db763851b1yya34a90c67c.jpg?wh=1920x1543" alt="图片"></p><p>今天我们还在Kubernetes集群里再次搭建了WordPress网站，应用了新对象Deployment、Service、Ingress，为网站增加了横向扩容、服务发现和七层负载均衡这三个非常重要的功能，提升了网站的稳定性和可用性，基本上解决了在“初级篇”所遇到的问题。</p><p>虽然这个网站离真正实用还差得比较远，但框架已经很完善了，你可以在这个基础上添加其他功能，比如创建证书Secret、让Ingress支持HTTPS等等。</p><p>另外，我们保证了网站各项服务的高可用，但对于数据库MariaDB来说，虽然Deployment在发生故障时能够及时重启Pod，新Pod却不会从旧Pod继承数据，之前网站的数据会彻底消失，这个后果是完全不可接受的。</p><p>所以在后续的“高级篇”里，我们会继续学习持久化存储对象PersistentVolume，以及有状态的StatefulSet等对象，进一步完善我们的网站。</p><h2>课下作业</h2><p>最后是课下作业时间，还是两个动手操作题：</p><ol>
<li>你能否把WordPress和Ingress Controller改成DaemonSet的部署方式？</li>
<li>你能否为Ingress Controller创建Service对象，让它以NodePort的方式对外提供服务？</li>
</ol><p>欢迎留言分享你的实操体验，如果觉得这篇文章对你有帮助，也欢迎你分享给身边的朋友一起学习。下节课是视频课，我们下节课再见。</p><p><img src="https://static001.geekbang.org/resource/image/60/b2/607a15a372b6dd4fd59d2060d1e811b2.jpg?wh=1920x1377" alt="图片"></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/99/87/98ebb20e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>dao</span>
  </div>
  <div class="_2_QraFYR_0">因为前面几节都是按照文稿来演练，坑都踩完了，本节很轻松完成。<br>作业：<br>1. 直接可以由 deployment 改编一下：修改 Kind，移除 replicas ； 根据需要添加 tolerations 设置。<br>2. 为 Ingress Controller 创建 Service 对象，用YAML样板来实现吧<br>```bash<br>kubectl expose deploy wp-kic-dep --port=80,443 --type=NodePort --name=wp-kic-svc -n nginx-ingress $out<br>```<br>大概的 YAML <br>```yaml<br>  - name: http<br>    port: 80<br>    protocol: TCP<br>    targetPort: 80<br>    nodePort: 30080<br>  - name: https<br>    port: 443<br>    protocol: TCP<br>    targetPort: 443<br>    nodePort: 30443<br>```<br>记得移除 ingress controller deployment 里的 `hostNetwork: true`。<br>现在就可以愉快地访问 https:&#47;&#47;wp.test:30443&#47; 。<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-17 15:29:42</div>
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
  <div class="_2_QraFYR_0">使用http:&#47;&#47;wp.test&#47;访问,需要在host改wp-kic-dep部署的那个节点的ip才可以</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-12 23:57:26</div>
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
  <div class="_2_QraFYR_0">改成DaemonSet只需要把kind的Deployment变成DeamonSet、把replicas注释掉就可以</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-14 18:46:49</div>
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
  <div class="_2_QraFYR_0">老师，22讲最后一步，就是创建nginx controller失败，POD状态是“CrashLoopBackOff”。用logs命令查看，错误信息是：<br>Error when getting IngressClass wp-ink: ingressclasses.networking.k8s.io &quot;wp-ink&quot; is forbidden: User &quot;system:serviceaccount:nginx-ingress:nginx-ingress&quot; cannot get resource &quot;ingressclasses&quot; in API group &quot;networking.k8s.io&quot; at the cluster scope<br><br>不知道是什么意思？不知道怎么修改？ （能看懂英文，但不知道说的是什么） （注：kic文件是拷贝老师的，https:&#47;&#47;github.com&#47;chronolaw&#47;k8s_study&#47;blob&#47;master&#47;ch3&#47;wp-kic.yml   完全拷贝，成功创建）</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 应该是前面没有安装好Nginx Ingress Controller吧，需要运行Ingress目录里的setup.sh，apply相关的YAML 。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-11 07:23:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_2ce074</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，Ingress Controller 对象，在哪个 Nginx 项目的示例 YAML 里有呀，根本找不到呀</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 参考这个：https:&#47;&#47;docs.nginx.com&#47;nginx-ingress-controller&#47;installation&#47;installation-with-manifests&#47;</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-10 17:54:16</div>
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
  <div class="_2_QraFYR_0">老师，数据库配置没问题，svc也正常，但访问网站时提示 Error establishing a database connection，不知道什么原因</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 应该是WordPress配置的不对吧，看看环境变量的设置。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-10 10:57:10</div>
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
  <div class="_2_QraFYR_0">关于hostNetwork方式，分享一点，就是hostNetwork这种形式虽然可以让外界访问到，但是你一定要让外界机器通过域名，这里是“wp.test”能够路由到这台机器，当然这个很简单，只要在&#47;etc&#47;hosts里面添加一条记录就可以，但是wp.test要映射到哪一台机器？开始我在这个地方卡了好久，以为配置的是master机器，但是其实不是，应该是ingress controller的pod部署的那台机器，通过命令（kubectl get pod -n nginx-ingress -o wide）可以查看到pod是部署的机器，然后决定映射的ip；<br><br>这一点文章中有提到，但是容易忽略。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-26 17:41:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1c/9f/d2/b6d8df48.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>菲茨杰拉德</span>
  </div>
  <div class="_2_QraFYR_0">配置hosts后，解析出来的还是ingress的ip，在七层负载不还是相当于用的ip来访问的吗？这个解析过程不是k8s做的吧。<br>我原来理解的是七层负载走http，通过域名来访问。输入域名后，k8s解析然后找到四层的ip和端口然后请求。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 域名解析是由运行在Kubernetes里的coredns实现的，不是Kubernetes自身的功能，可以认为是插件。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-23 11:26:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/a0/ab/5f43eee8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>笨晓孩</span>
  </div>
  <div class="_2_QraFYR_0">老师，我部署完wp-svc之后，通过nodeIP:30088的方式没办法访问Wordpress页面，采用的是nodeport的方式</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 先看一下Service是否正常，里面的endpoint是否是WordPress的pod，在集群节点或者pod里ping WordPress，看pod是否正常。<br><br>我感觉应该还是Service没有设置好。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-27 23:17:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/g4os8I4iaB6jn06PsvyqI1BooV5XbOC0vI3niaJ4I3SlAhkbBKG2eewlPHHJ4ROcDia18bbPFSZPDXXmgHXtrBlLg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_f2f06e</span>
  </div>
  <div class="_2_QraFYR_0">Service 对象是 NodePort 类型的，为什么还是只能在worker节点上访问 WordPress 服务。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 看看service是否哪里设置的不对。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-08 17:52:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/ca/07/22dd76bf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>kobe</span>
  </div>
  <div class="_2_QraFYR_0">我加上ingress controller后启动，报了这个错，能帮看下是什么原因么：<br>NGINX Ingress Controller Version=3.1.0 Commit=057c6d7e4f2361f5d2ddd897e9995bcb48ed7e32 Date=2023-03-27T10:15:43Z DirtyState=false Arch=linux&#47;amd64 Go=go1.20.2<br>I0404 10:41:32.746481       1 flags.go:294] Starting with flags: [&quot;-nginx-configmaps=nginx-ingress&#47;nginx-config&quot; &quot;-ingress-class=wp-ink&quot;]<br>I0404 10:41:32.761875       1 main.go:234] Kubernetes version: 1.23.3<br>I0404 10:41:32.771440       1 main.go:380] Using nginx version: nginx&#47;1.23.3<br>E0404 10:41:32.773053       1 main.go:753] Error getting pod: pods &quot;wp-kic-dep-7d46fd4f8d-mnxpk&quot; is forbidden: User &quot;system:serviceaccount:nginx-ingress:nginx-ingress&quot; cannot get resource &quot;pods&quot; in API group &quot;&quot; in the namespace &quot;nginx-ingress&quot;<br>2023&#47;04&#47;04 10:41:32 [emerg] 12#12: bind() to 0.0.0.0:80 failed (13: Permission denied)</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 应该是Ingress controller的资源没有完全创建好吧，用GitHub上的setup脚本，把所有相关的yaml都安装一遍。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-04 19:17:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/ca/07/22dd76bf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>kobe</span>
  </div>
  <div class="_2_QraFYR_0">我这个ingress controller启动后没成功，一直在不断的创建pod，如下：<br>wp-kic-dep-545dffd6d7-2n9cq    0&#47;1     SysctlForbidden   0          48s<br>wp-kic-dep-545dffd6d7-2zkh2    0&#47;1     SysctlForbidden   0          65s<br>wp-kic-dep-545dffd6d7-4f42q    0&#47;1     SysctlForbidden   0          20s<br>wp-kic-dep-545dffd6d7-4vxlw    0&#47;1     SysctlForbidden   0          53s<br>wp-kic-dep-545dffd6d7-6vshx    0&#47;1     SysctlForbidden   0          8s<br>wp-kic-dep-545dffd6d7-6zgx7    0&#47;1     SysctlForbidden   0          60s<br>wp-kic-dep-545dffd6d7-85mjh    0&#47;1     SysctlForbidden   0          46s<br>wp-kic-dep-545dffd6d7-88jcn    0&#47;1     SysctlForbidden   0          55s<br>wp-kic-dep-545dffd6d7-8mfjp    0&#47;1     SysctlForbidden   0          35s<br>wp-kic-dep-545dffd6d7-8z5ps    0&#47;1     SysctlForbidden   0          72s<br>wp-kic-dep-545dffd6d7-9bzgp    0&#47;1     SysctlForbidden   0          41s<br>wp-kic-dep-545dffd6d7-9v86q    0&#47;1     SysctlForbidden   0          15s<br>wp-kic-dep-545dffd6d7-c42rl    0&#47;1     SysctlForbidden   0          38s<br>wp-kic-dep-545dffd6d7-c8zx8    0&#47;1     SysctlForbidden   0          57s<br>wp-kic-dep-545dffd6d7-ccjfm    0&#47;1     SysctlForbidden   0          7s<br>wp-kic-dep-545dffd6d7-chwt9    0&#47;1     SysctlForbidden   0          16s<br>wp-kic-dep-545dffd6d7-crf4s    0&#47;1     SysctlForbidden   0          17s<br>wp-kic-dep-545dffd6d7-cx8bx    0&#47;1     SysctlForbidden   0          20s<br>wp-kic-dep-545dffd6d7-dc7w2    0&#47;1     SysctlForbidden   0          11s<br>wp-kic-dep-545dffd6d7-dzvvx    0&#47;1     SysctlForbidden   0          73s<br>wp-kic-dep-545dffd6d7-f72lj    0&#47;1     SysctlForbidden   0          66s<br>wp-kic-dep-545dffd6d7-gjdkz    0&#47;1     SysctlForbidden   0          27s</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 用describe看看，有没有错误信息。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-04 15:01:43</div>
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
  <div class="_2_QraFYR_0">apiVersion: v1<br>kind: Service<br>metadata:<br>  creationTimestamp: null<br>  name: wp-kic-svc<br>  namespace: nginx-ingress<br>spec:<br>  ports:<br>  - name: http<br>    port: 80<br>    protocol: TCP<br>    targetPort: 80<br>    nodePort: 30089<br>  selector:<br>    app: wp-kic-dep<br>  type: NodePort<br>status:<br>  loadBalancer: {}<br><br>老师，这是我写的Ingress Controller 的 Service 对象，之前用http:&#47;&#47;wp.test:30089 访问 <br>提示<br>该网页无法正常运作<br>wp.test 目前无法处理此请求。<br>HTTP ERROR 502<br><br>是因为我电脑开了vpn代理，然后关了就可以访问了，这个大家看看和我是否有同样的问题。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good，自己研究并解决问题，会对以后的工作更有帮助。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-17 17:52:35</div>
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
  <div class="_2_QraFYR_0">apiVersion: v1<br>kind: Service<br>metadata:<br>  creationTimestamp: null<br>  name: wp-kic-svc<br>  namespace: nginx-ingress<br>spec:<br>  ports:<br>  - name: http<br>    port: 80<br>    protocol: TCP<br>    targetPort: 80<br>    nodePort: 30089<br>  selector:<br>    app: wp-kic-dep<br>  type: NodePort<br>status:<br>  loadBalancer: {}<br><br>老师，这是我写的Ingress Controller 的 Service 对象，用http:&#47;&#47;wp.test:30089 访问 提示拒绝连接   有什么排查思路吗？ hostNetwork: true 在wp-kic.yml我也注掉了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: service只能保证在kubernetes集群里使用域名，在外界使用域名要修改hosts文件。<br><br>service是否正常可以用kubectl来检查，还有看coredns的运行状态。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-14 18:28:04</div>
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
  <div class="_2_QraFYR_0">E0315 07:51:23.286819       1 reflector.go:138] pkg&#47;mod&#47;k8s.io&#47;client-go@v0.23.6&#47;tools&#47;cache&#47;reflector.go:167: Failed to watch *v1.VirtualServerRoute: failed to list *v1.VirtualServerRoute: the server could not find the requested resource (get virtualserverroutes.k8s.nginx.org)<br><br>老师，ingress-controller pod里的日志报上面这个错误，是ingress-controller安装的有问题吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 应该是没有安装Nginx Ingress controller的crd资源吧，看一下GitHub上的脚本，把所有的yaml都安装上。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-14 15:53:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/d2/00/9a247b1e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jason</span>
  </div>
  <div class="_2_QraFYR_0">老师，wp.test这个域名需要指定ingress controller那个pod所在的节点ip，而ingress controller是通过deployment来管理的，pod重建时可能会被部署到别的节点，这样不是又要改host配置了吗？想问一下生产环境是怎样解决这个问题的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 生产环境用的一般是把Ingress Controller的service配置成LoadBalancer，云厂商会自动分配IP地址，我们这个是实验环境，所以比较简陋。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-01 08:21:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/32/50/8d/ded8482f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>一久</span>
  </div>
  <div class="_2_QraFYR_0">Error establishing a database connection<br>apiVersion: v1<br>kind: Service<br>metadata:<br>  labels:<br>    app: maria-dep<br>  name: maria-svc<br>spec:<br>  ports:<br>  - port: 3306<br>    protocol: TCP<br>    targetPort: 3306<br>  selector:<br>    app: maria-dep<br><br><br>apiVersion: v1<br>kind: Service<br>metadata:<br>  labels:<br>    app: wp-dep<br>  name: wp-svc<br><br>spec:<br>  ports:<br>  - name: http80<br>    port: 80<br>    protocol: TCP<br>    targetPort: 80<br><br>  selector:<br>    app: wp-dep<br>老师这个报错咋排查<br>这是两个service yaml文件</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 连不上数据库，看看dns是否正常，用命令行连接mariaDB试试看问题在哪里。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-22 09:04:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/28/ca/47333d8b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lpqoang</span>
  </div>
  <div class="_2_QraFYR_0">Nodeport方式可以访问， 使用inggress配置域名解析的方式会404，可能是什么问题呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 看看coreDNS是否工作正常。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-14 08:33:56</div>
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
  <div class="_2_QraFYR_0">pod数量设置为1个，就可以了。wp.test这个域名是不是随机指向多个后端pod，每个pod域名也不一样，而wordpress他使用cookie，感觉存在跨域问题</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不是很了解WordPress的工作机制，感谢分析。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-13 10:37:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1c/9f/d2/b6d8df48.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>菲茨杰拉德</span>
  </div>
  <div class="_2_QraFYR_0">在部署controller之前命名空间都是默认的。后面deploy换成了nginx-ingress,deploy和pod都是在这个空间下，有问题 找不到配置。应该空间一样<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Ingress Controller可以在专用的nginx-ingress里运行，没有问题的，可以再看看GitHub上的YAML 。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-23 12:02:05</div>
  </div>
</div>
</div>
</li>
</ul>