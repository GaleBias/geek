<audio title="18 _ 深入理解StatefulSet（一）：拓扑状态" src="https://static001.geekbang.org/resource/audio/89/12/896c974f179010713e58ada2f49d7c12.mp3" controls="controls"></audio> 
<p>你好，我是张磊。今天我和你分享的主题是：深入理解StatefulSet之拓扑状态。</p><p>在上一篇文章中，我在结尾处讨论到了Deployment实际上并不足以覆盖所有的应用编排问题。</p><p>造成这个问题的根本原因，在于Deployment对应用做了一个简单化假设。</p><p>它认为，一个应用的所有Pod，是完全一样的。所以，它们互相之间没有顺序，也无所谓运行在哪台宿主机上。需要的时候，Deployment就可以通过Pod模板创建新的Pod；不需要的时候，Deployment就可以“杀掉”任意一个Pod。</p><p>但是，在实际的场景中，并不是所有的应用都可以满足这样的要求。</p><p>尤其是分布式应用，它的多个实例之间，往往有依赖关系，比如：主从关系、主备关系。</p><p>还有就是数据存储类应用，它的多个实例，往往都会在本地磁盘上保存一份数据。而这些实例一旦被杀掉，即便重建出来，实例与数据之间的对应关系也已经丢失，从而导致应用失败。</p><p>所以，这种实例之间有不对等关系，以及实例对外部数据有依赖关系的应用，就被称为“有状态应用”（Stateful Application）。</p><p>容器技术诞生后，大家很快发现，它用来封装“无状态应用”（Stateless Application），尤其是Web服务，非常好用。但是，一旦你想要用容器运行“有状态应用”，其困难程度就会直线上升。而且，这个问题解决起来，单纯依靠容器技术本身已经无能为力，这也就导致了很长一段时间内，“有状态应用”几乎成了容器技术圈子的“忌讳”，大家一听到这个词，就纷纷摇头。</p><!-- [[[read_end]]] --><p>不过，Kubernetes项目还是成为了“第一个吃螃蟹的人”。</p><p>得益于“控制器模式”的设计思想，Kubernetes项目很早就在Deployment的基础上，扩展出了对“有状态应用”的初步支持。这个编排功能，就是：StatefulSet。</p><p>StatefulSet的设计其实非常容易理解。它把真实世界里的应用状态，抽象为了两种情况：</p><ol>
<li>
<p><strong>拓扑状态</strong>。这种情况意味着，应用的多个实例之间不是完全对等的关系。这些应用实例，必须按照某些顺序启动，比如应用的主节点A要先于从节点B启动。而如果你把A和B两个Pod删除掉，它们再次被创建出来时也必须严格按照这个顺序才行。并且，新创建出来的Pod，必须和原来Pod的网络标识一样，这样原先的访问者才能使用同样的方法，访问到这个新Pod。</p>
</li>
<li>
<p><strong>存储状态</strong>。这种情况意味着，应用的多个实例分别绑定了不同的存储数据。对于这些应用实例来说，Pod A第一次读取到的数据，和隔了十分钟之后再次读取到的数据，应该是同一份，哪怕在此期间Pod A被重新创建过。这种情况最典型的例子，就是一个数据库应用的多个存储实例。</p>
</li>
</ol><p>所以，<strong>StatefulSet的核心功能，就是通过某种方式记录这些状态，然后在Pod被重新创建时，能够为新Pod恢复这些状态。</strong></p><p>在开始讲述StatefulSet的工作原理之前，我就必须<span class="orange">先为你讲解一个Kubernetes项目中非常实用的概念：Headless Service。</span></p><p>我在和你一起讨论Kubernetes架构的时候就曾介绍过，Service是Kubernetes项目中用来将一组Pod暴露给外界访问的一种机制。比如，一个Deployment有3个Pod，那么我就可以定义一个Service。然后，用户只要能访问到这个Service，它就能访问到某个具体的Pod。</p><p>那么，这个Service又是如何被访问的呢？</p><p><strong>第一种方式，是以Service的VIP（Virtual IP，即：虚拟IP）方式</strong>。比如：当我访问10.0.23.1这个Service的IP地址时，10.0.23.1其实就是一个VIP，它会把请求转发到该Service所代理的某一个Pod上。这里的具体原理，我会在后续的Service章节中进行详细介绍。</p><p><strong>第二种方式，就是以Service的DNS方式</strong>。比如：这时候，只要我访问“my-svc.my-namespace.svc.cluster.local”这条DNS记录，就可以访问到名叫my-svc的Service所代理的某一个Pod。</p><p>而在第二种Service DNS的方式下，具体还可以分为两种处理方法：</p><p>第一种处理方法，是Normal Service。这种情况下，你访问“my-svc.my-namespace.svc.cluster.local”解析到的，正是my-svc这个Service的VIP，后面的流程就跟VIP方式一致了。</p><p>而第二种处理方法，正是Headless Service。这种情况下，你访问“my-svc.my-namespace.svc.cluster.local”解析到的，直接就是my-svc代理的某一个Pod的IP地址。<strong>可以看到，这里的区别在于，Headless Service不需要分配一个VIP，而是可以直接以DNS记录的方式解析出被代理Pod的IP地址。</strong></p><p>那么，这样的设计又有什么作用呢？</p><p>想要回答这个问题，我们需要从Headless Service的定义方式看起。</p><p>下面是一个标准的Headless Service对应的YAML文件：</p><pre><code>apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
</code></pre><p>可以看到，所谓的Headless Service，其实仍是一个标准Service的YAML文件。只不过，它的clusterIP字段的值是：None，即：这个Service，没有一个VIP作为“头”。这也就是Headless的含义。所以，这个Service被创建后并不会被分配一个VIP，而是会以DNS记录的方式暴露出它所代理的Pod。</p><p>而它所代理的Pod，依然是采用我在前面第12篇文章<a href="https://time.geekbang.org/column/article/40008">《牛刀小试：我的第一个容器化应用》</a>中提到的Label Selector机制选择出来的，即：所有携带了app=nginx标签的Pod，都会被这个Service代理起来。</p><p>然后关键来了。</p><p>当你按照这样的方式创建了一个Headless Service之后，它所代理的所有Pod的IP地址，都会被绑定一个这样格式的DNS记录，如下所示：</p><pre><code>&lt;pod-name&gt;.&lt;svc-name&gt;.&lt;namespace&gt;.svc.cluster.local
</code></pre><p>这个DNS记录，正是Kubernetes项目为Pod分配的唯一的“可解析身份”（Resolvable Identity）。</p><p>有了这个“可解析身份”，只要你知道了一个Pod的名字，以及它对应的Service的名字，你就可以非常确定地通过这条DNS记录访问到Pod的IP地址。</p><p>那么，<span class="orange">StatefulSet又是如何使用这个DNS记录来维持Pod的拓扑状态的呢？</span></p><p>为了回答这个问题，现在我们就来编写一个StatefulSet的YAML文件，如下所示：</p><pre><code>apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: &quot;nginx&quot;
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.9.1
        ports:
        - containerPort: 80
          name: web
</code></pre><p>这个YAML文件，和我们在前面文章中用到的nginx-deployment的唯一区别，就是多了一个serviceName=nginx字段。</p><p>这个字段的作用，就是告诉StatefulSet控制器，在执行控制循环（Control Loop）的时候，请使用nginx这个Headless Service来保证Pod的“可解析身份”。</p><p>所以，当你通过kubectl create创建了上面这个Service和StatefulSet之后，就会看到如下两个对象：</p><pre><code>$ kubectl create -f svc.yaml
$ kubectl get service nginx
NAME      TYPE         CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
nginx     ClusterIP    None         &lt;none&gt;        80/TCP    10s

$ kubectl create -f statefulset.yaml
$ kubectl get statefulset web
NAME      DESIRED   CURRENT   AGE
web       2         1         19s
</code></pre><p>这时候，如果你手比较快的话，还可以通过kubectl的-w参数，即：Watch功能，实时查看StatefulSet创建两个有状态实例的过程：</p><blockquote>
<p>备注：如果手不够快的话，Pod很快就创建完了。不过，你依然可以通过这个StatefulSet的Events看到这些信息。</p>
</blockquote><pre><code>$ kubectl get pods -w -l app=nginx
NAME      READY     STATUS    RESTARTS   AGE
web-0     0/1       Pending   0          0s
web-0     0/1       Pending   0         0s
web-0     0/1       ContainerCreating   0         0s
web-0     1/1       Running   0         19s
web-1     0/1       Pending   0         0s
web-1     0/1       Pending   0         0s
web-1     0/1       ContainerCreating   0         0s
web-1     1/1       Running   0         20s
</code></pre><p>通过上面这个Pod的创建过程，我们不难看到，StatefulSet给它所管理的所有Pod的名字，进行了编号，编号规则是：<code>&lt;statefulset name&gt;-&lt;ordinal index&gt;</code>。</p><p>而且这些编号都是从0开始累加，与StatefulSet的每个Pod实例一一对应，绝不重复。</p><p>更重要的是，这些Pod的创建，也是严格按照编号顺序进行的。比如，在web-0进入到Running状态、并且细分状态（Conditions）成为Ready之前，web-1会一直处于Pending状态。</p><blockquote>
<p>备注：Ready状态再一次提醒了我们，为Pod设置livenessProbe和readinessProbe的重要性。</p>
</blockquote><p>当这两个Pod都进入了Running状态之后，你就可以查看到它们各自唯一的“网络身份”了。</p><p>我们使用kubectl exec命令进入到容器中查看它们的hostname：</p><pre><code>$ kubectl exec web-0 -- sh -c 'hostname'
web-0
$ kubectl exec web-1 -- sh -c 'hostname'
web-1
</code></pre><p>可以看到，这两个Pod的hostname与Pod名字是一致的，都被分配了对应的编号。接下来，我们再试着以DNS的方式，访问一下这个Headless Service：</p><pre><code>$ kubectl run -i --tty --image busybox:1.28.4 dns-test --restart=Never --rm /bin/sh 
</code></pre><p>通过这条命令，我们启动了一个一次性的Pod，因为--rm意味着Pod退出后就会被删除掉。然后，在这个Pod的容器里面，我们尝试用nslookup命令，解析一下Pod对应的Headless Service：</p><pre><code>$ kubectl run -i --tty --image busybox:1.28.4 dns-test --restart=Never --rm /bin/sh
$ nslookup web-0.nginx
Server:    10.0.0.10
Address 1: 10.0.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-0.nginx
Address 1: 10.244.1.7

$ nslookup web-1.nginx
Server:    10.0.0.10
Address 1: 10.0.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-1.nginx
Address 1: 10.244.2.7
</code></pre><p>从nslookup命令的输出结果中，我们可以看到，在访问web-0.nginx的时候，最后解析到的，正是web-0这个Pod的IP地址；而当访问web-1.nginx的时候，解析到的则是web-1的IP地址。</p><p>这时候，如果你在另外一个Terminal里把这两个“有状态应用”的Pod删掉：</p><pre><code>$ kubectl delete pod -l app=nginx
pod &quot;web-0&quot; deleted
pod &quot;web-1&quot; deleted
</code></pre><p>然后，再在当前Terminal里Watch一下这两个Pod的状态变化，就会发现一个有趣的现象：</p><pre><code>$ kubectl get pod -w -l app=nginx
NAME      READY     STATUS              RESTARTS   AGE
web-0     0/1       ContainerCreating   0          0s
NAME      READY     STATUS    RESTARTS   AGE
web-0     1/1       Running   0          2s
web-1     0/1       Pending   0         0s
web-1     0/1       ContainerCreating   0         0s
web-1     1/1       Running   0         32s
</code></pre><p>可以看到，当我们把这两个Pod删除之后，Kubernetes会按照原先编号的顺序，创建出了两个新的Pod。并且，Kubernetes依然为它们分配了与原来相同的“网络身份”：web-0.nginx和web-1.nginx。</p><p>通过这种严格的对应规则，<strong>StatefulSet就保证了Pod网络标识的稳定性</strong>。</p><p>比如，如果web-0是一个需要先启动的主节点，web-1是一个后启动的从节点，那么只要这个StatefulSet不被删除，你访问web-0.nginx时始终都会落在主节点上，访问web-1.nginx时，则始终都会落在从节点上，这个关系绝对不会发生任何变化。</p><p>所以，如果我们再用nslookup命令，查看一下这个新Pod对应的Headless Service的话：</p><pre><code>$ kubectl run -i --tty --image busybox dns-test --restart=Never --rm /bin/sh 
$ nslookup web-0.nginx
Server:    10.0.0.10
Address 1: 10.0.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-0.nginx
Address 1: 10.244.1.8

$ nslookup web-1.nginx
Server:    10.0.0.10
Address 1: 10.0.0.10 kube-dns.kube-system.svc.cluster.local

Name:      web-1.nginx
Address 1: 10.244.2.8
</code></pre><p>我们可以看到，在这个StatefulSet中，这两个新Pod的“网络标识”（比如：web-0.nginx和web-1.nginx），再次解析到了正确的IP地址（比如：web-0 Pod的IP地址10.244.1.8）。</p><p>通过这种方法，<strong>Kubernetes就成功地将Pod的拓扑状态（比如：哪个节点先启动，哪个节点后启动），按照Pod的“名字+编号”的方式固定了下来</strong>。此外，Kubernetes还为每一个Pod提供了一个固定并且唯一的访问入口，即：这个Pod对应的DNS记录。</p><p>这些状态，在StatefulSet的整个生命周期里都会保持不变，绝不会因为对应Pod的删除或者重新创建而失效。</p><p>不过，相信你也已经注意到了，尽管web-0.nginx这条记录本身不会变，但它解析到的Pod的IP地址，并不是固定的。这就意味着，对于“有状态应用”实例的访问，你必须使用DNS记录或者hostname的方式，而绝不应该直接访问这些Pod的IP地址。</p><h2>总结</h2><p>在今天这篇文章中，我首先和你分享了StatefulSet的基本概念，解释了什么是应用的“状态”。</p><p>紧接着 ，我为你分析了StatefulSet如何保证应用实例之间“拓扑状态”的稳定性。</p><p>如果用一句话来总结的话，你可以这么理解这个过程：</p><blockquote>
<p>StatefulSet这个控制器的主要作用之一，就是使用Pod模板创建Pod的时候，对它们进行编号，并且按照编号顺序逐一完成创建工作。而当StatefulSet的“控制循环”发现Pod的“实际状态”与“期望状态”不一致，需要新建或者删除Pod进行“调谐”的时候，它会严格按照这些Pod编号的顺序，逐一完成这些操作。</p>
</blockquote><p>所以，StatefulSet其实可以认为是对Deployment的改良。</p><p>与此同时，通过Headless Service的方式，StatefulSet为每个Pod创建了一个固定并且稳定的DNS记录，来作为它的访问入口。</p><p>实际上，在部署“有状态应用”的时候，应用的每个实例拥有唯一并且稳定的“网络标识”，是一个非常重要的假设。</p><p>在下一篇文章中，我将会继续为你剖析StatefulSet如何处理存储状态。</p><h2>思考题</h2><p>你曾经运维过哪些有拓扑状态的应用呢（比如：主从、主主、主备、一主多从等结构）？你觉得这些应用实例之间的拓扑关系，能否借助这种为Pod实例编号的方式表达出来呢？如果不能，你觉得Kubernetes还应该为你提供哪些支持来管理这个拓扑状态呢？</p><p>感谢你的收听，欢迎你给我留言，也欢迎分享给更多的朋友一起阅读。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/64/05/6989dce6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我来也</span>
  </div>
  <div class="_2_QraFYR_0">今天按文章中的内容来, 确实也遇到了nslookup 反馈失败的状况.<br>** server can&#39;t find web-0.nginx: NXDOMAIN<br>*** Can&#39;t find web-0.nginx: No answer<br>但是直接ping  web-0.nginx 是可以获取真实ip地址的.<br>确实是如楼下的同学所说, 需要用 busybox:1.28.4 的镜像. 这个是最新版busybox的问题.<br>kubectl run -i --tty --image busybox:1.28.4 dns-test --restart=Never --rm &#47;bin&#47;sh<br>这样就可获得期待的结果了.<br>我也是google到了 https:&#47;&#47;github.com&#47;kubernetes&#47;kubernetes&#47;issues&#47;66924 才知道的.<br>再看楼下的评论,才发现有其他同学也遇到了,且在几天前已经给出了解决方案. 👍<br>新技术确实变化太快了,作者在写文章时用的没问题,也许隔一天因为某个默认镜像修改了,就会出现新的状况.</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 看来这个镜像确实有问题</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-13 14:47:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/ec/3d/a8b38ce3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>along2018</span>
  </div>
  <div class="_2_QraFYR_0">创建statefulset必须要先创建一个headless的service?分两步操作？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-08 19:42:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/9c/67/c0128e6c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Dillion</span>
  </div>
  <div class="_2_QraFYR_0">在上面的例子中，web-0、web-1启动后，此时如果web-0挂了，那在创建web-0的过程中，web-1也会被重新创建一次么？？？也就是如果一个StatefulSet中只有某个Pod挂了，在重启它的时候，如何确保文中说的Pod启动顺序呢？？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 任何一次pod的更新都会触发statefulset 滚动更新，更新一定会按编号顺序。但如果只是删除那就直接重建当前pod就够了，这并不破坏拓扑状态。当然，必要的时候，你的pod启动命令要能够区分第一次启动和重启，见下一节。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-03 01:06:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/27/4b/170654bb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>千寻</span>
  </div>
  <div class="_2_QraFYR_0">说dns访问不到那个童鞋，不要用latest标签的busybox，用busybox:1.28.4这个tag的就可以了，我也是这样。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这咋还跟镜像有关系呢？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-09 11:08:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/a9/2e/01b2839e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>巩夫建</span>
  </div>
  <div class="_2_QraFYR_0">Headless Service中不通过vip，外部访问的时候，是轮询还是随机还是热备的方式访问到web-0和web-1，dns解析好像没有轮询随机概念吧。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 只能访问到固定的一个pod。所以说headless service不能替代normal service </p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-06 14:11:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/PUiby8MibibKcMd88OtDq1c0myEILZjap46fyiaOlML0UlNWzj9NTIEXOhXCCR1tcUibG0I6UoGp59Zj8H5EYwzkY9g/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>fldhmily63319</span>
  </div>
  <div class="_2_QraFYR_0">我也想问， &quot;Normal Service&quot;和&quot;Headless Service“的应用场景是什么？<br><br>根据文章所说的，”Headless Service不需要分配一个 VIP，而是可以直接以 DNS 记录的方式解析出被代理 Pod 的 IP 地址“，同时由于其编号的严格规定，会按照编号顺序完成创建工作。<br><br>这样是不是说，如果在部署StatefulSet的时候，大部分是推荐使用&quot;Headless Service&quot; ，而不是&quot;Normal Service”呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是，必须用headless service</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-03 10:20:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/5a/56/115c6433.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jssfy</span>
  </div>
  <div class="_2_QraFYR_0">通过这种方法，Kubernetes 就成功地将 Pod 的拓扑状态（比如：哪个节点先启动，哪个节点后启动），按照 Pod 的“名字 + 编号”的方式固定了下来。<br>如上所述，<br>1. 是否可以这样理解：sts在这里只是保留了“名字 + 编号”这种网络身份，而不同网络身份对应的pod其实本质上是一样的，都是同一个模板replicate出来的？<br>2. 这里sts的主要意义是什么呢：仅仅是保证不同网络身份的启动顺序？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: pod镜像是一样，但启动pod的命令和初始化流程可以完全不一样啊。可以参考后面的完整案例</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-04 07:19:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/93/c7/1640226d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>刘欣洲</span>
  </div>
  <div class="_2_QraFYR_0">访问不到啊<br>&#47; # nslookup web-0.nginx<br>Server:		10.96.0.10<br>Address:	10.96.0.10:53<br><br>** server can&#39;t find web-0.nginx: NXDOMAIN<br><br>*** Can&#39;t find web-0.nginx: No answer<br><br>是不是需要DNS插件啊， 该如何启动呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: dns是默认插件。你按我前面的部署流程、也就是官方的部署流程，是必然有dns的。看看pod列表debug一下吧。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-04 06:13:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/af/d3/abb7bfe3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Lukri</span>
  </div>
  <div class="_2_QraFYR_0">加上namespace就可以了。<br>&#47; # ping web-0.nginx.default.svc.cluster.local<br>PING web-0.nginx.default.svc.cluster.local (10.44.0.2): 56 data bytes<br>64 bytes from 10.44.0.2: seq=0 ttl=64 time=0.138 ms<br>64 bytes from 10.44.0.2: seq=1 ttl=64 time=0.089 ms<br>64 bytes from 10.44.0.2: seq=2 ttl=64 time=0.091 ms<br>64 bytes from 10.44.0.2: seq=3 ttl=64 time=0.090 ms<br>64 bytes from 10.44.0.2: seq=4 ttl=64 time=0.093 ms<br>64 bytes from 10.44.0.2: seq=5 ttl=64 time=0.128 ms<br>64 bytes from 10.44.0.2: seq=6 ttl=64 time=0.093 ms<br>64 bytes from 10.44.0.2: seq=7 ttl=64 time=0.091 ms<br>^C<br>--- web-0.nginx.default.svc.cluster.local ping statistics ---<br>8 packets transmitted, 8 packets received, 0% packet loss<br>round-trip min&#47;avg&#47;max = 0.089&#47;0.101&#47;0.138 ms<br>&#47; # nslookup web-0.nginx.default.svc.cluster.local<br>Server:    10.96.0.10<br>Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local<br><br>Name:      web-0.nginx.default.svc.cluster.local<br>Address 1: 10.44.0.2 web-0.nginx.default.svc.cluster.local</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-29 16:19:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/88/23/a0966b4d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Tim Zhang</span>
  </div>
  <div class="_2_QraFYR_0">既然默认有安装dns 为啥还要开启dns插件呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 因为有太多人是自己拿二进制文件DIY部署的，不按kubeadm的流程来</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-04 08:48:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/68/c3/8e1a8dbf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Plantegg</span>
  </div>
  <div class="_2_QraFYR_0">首先busybox镜像的&#47;bin&#47; 下几百个可执行命令的md5签名都是一样的。不能按照正常的ping、nslookup逻辑来理解了。 也就是md5sum &#47;bin&#47;ping 和 md5sum &#47;bin&#47;nslookup 签名一样，我猜测这个可执行命令都是空架子，会被拦截下来。<br>另外就是1.28.4和1.29.3（latest) 的 nslookup 签名也不一样了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: busybox就是这么做出来的，正常</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-18 19:01:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/bc/08/c43f85d9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>IOVE.-Minn</span>
  </div>
  <div class="_2_QraFYR_0">请教张大佬，当我跑jenkins in k8s的时候是用的statefulset部署的jenkins的master，我关联的service但是却不是无头服务啊，  spec.ports 是用的nodeport  也没有用clusterIP：none这样。但是整个服务也是正常的。这不是和你讲的statefulset必须是关联headless service违背了么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这说明你不需要通过带编号的dns名字访问这个节点呗，没啥奇怪的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-09 10:39:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/98/37/7f575aec.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>vx:jiancheng_goon</span>
  </div>
  <div class="_2_QraFYR_0">service里的dns信息是存在etcd里的嘛？有些应用的pod的域名是自己定义的。而不是k8s创建出来的带有local的域名。我可以更改service里的dns信息嘛？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: kubernetes 当然允许你自定义域名了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-04 11:30:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/09/da/2fbd0760.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>兽医</span>
  </div>
  <div class="_2_QraFYR_0">请问当需要对StatefulSet进行缩容后，再扩容到原有规模。在涉及变动的Pod中，状态也会被保持吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 当然</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-03 15:18:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/94/47/75875257.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>虎虎❤️</span>
  </div>
  <div class="_2_QraFYR_0">您好。滚动升级的时候，如果新的web-0和老的web-2同时ready，但新老版本不兼容怎么办？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 滚动更新的结果一定是版本一样，你说的这个状态只可能是中间状态</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-03 11:36:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/b9/f5/ffc2bb23.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Rodinian</span>
  </div>
  <div class="_2_QraFYR_0">一直很好奇，normal service和headless service都可以指定DNS name。那两者的区别是什么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: service部分会讲解</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-16 21:29:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/32/0b/81ae214b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>凌</span>
  </div>
  <div class="_2_QraFYR_0">我们的服务都是通过固定ip+tcp的方式访问的，如果按照kubernetes的理念应该每次先用服务命取得一次ip在建连访问？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: ip改成用service vip或者dns名字即可</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-08 09:08:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/3e/f5/ba756775.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_6bab38</span>
  </div>
  <div class="_2_QraFYR_0">教程中两个nginx是一块启动：<br>如果是有顺序的，比如redis先启动，然后启动在启动nginx这样子，需要怎样设置呢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-18 17:51:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTIacvl2hoQU10YHABicE8cTu2IKiagWw9wXPiaRcvJfF1tL2ticJVzHGqd7LmTaDuIScvPDhhMOucjUtg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>zoroyouxi</span>
  </div>
  <div class="_2_QraFYR_0">实践中有一点疑问，既然sts是pod的实际所有者并不经过rs，那为什么sts声明下的pod的restartpolicy也只能设置成always呢?希望张老师解答谢谢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 因为它也依赖于容器自动重启来恢复实例状态。sts其实就是个改造后的deployment。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-21 11:41:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/d9/93/43c876e8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>rnn</span>
  </div>
  <div class="_2_QraFYR_0">eureka 需要用到stateful set</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-13 06:56:49</div>
  </div>
</div>
</div>
</li>
</ul>