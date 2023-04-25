<audio title="26｜StatefulSet：怎么管理有状态的应用？" src="https://static001.geekbang.org/resource/audio/a1/cb/a1d41d97c379f2c690b9b1a2b56252cb.mp3" controls="controls"></audio> 
<p>你好，我是Chrono。</p><p>在中级篇里，我们学习了Deployment和DaemonSet两种API对象，它们是在Kubernetes集群里部署应用的重要工具，不过它们也有一个缺点，只能管理“无状态应用”（Stateless Application），不能管理“有状态应用”（Stateful Application）。</p><p>“有状态应用”的处理比较复杂，要考虑的事情很多，但是这些问题我们其实可以通过组合之前学过的Deployment、Service、PersistentVolume等对象来解决。</p><p>今天我们就来研究一下什么是“有状态应用”，然后看看Kubernetes为什么会设计一个新对象——StatefulSet来专门管理“有状态应用”。</p><h2>什么是有状态的应用</h2><p>我们先从PersistentVolume谈起，它为Kubernetes带来了持久化存储的功能，能够让应用把数据存放在本地或者远程的磁盘上。</p><p>那么你有没有想过，持久化存储，对应用来说，究竟意味着什么呢？</p><p>有了持久化存储，应用就可以把一些运行时的关键数据落盘，相当于有了一份“保险”，如果Pod发生意外崩溃，也只不过像是按下了暂停键，等重启后挂载Volume，再加载原数据就能够满血复活，恢复之前的“状态”继续运行。</p><!-- [[[read_end]]] --><p>注意到了吗？这里有一个关键词——“<strong>状态</strong>”，应用保存的数据，实际上就是它某个时刻的“运行状态”。</p><p>所以从这个角度来说，理论上任何应用都是有状态的。</p><p>只是有的应用的状态信息不是很重要，即使不恢复状态也能够正常运行，这就是我们常说的“<strong>无状态应用</strong>”。“无状态应用”典型的例子就是Nginx这样的Web服务器，它只是处理HTTP请求，本身不生产数据（日志除外），不需要特意保存状态，无论以什么状态重启都能很好地对外提供服务。</p><p>还有一些应用，运行状态信息就很重要了，如果因为重启而丢失了状态是绝对无法接受的，这样的应用就是“<strong>有状态应用</strong>”。</p><p>“有状态应用”的例子也有很多，比如Redis、MySQL这样的数据库，它们的“状态”就是在内存或者磁盘上产生的数据，是应用的核心价值所在，如果不能够把这些数据及时保存再恢复，那绝对会是灾难性的后果。</p><p>理解了这一点，我们结合目前学到的知识思考一下：<strong>Deployment加上PersistentVolume，在Kubernetes里是不是可以轻松管理有状态的应用了呢？</strong></p><p>的确，用Deployment来保证高可用，用PersistentVolume来存储数据，确实可以部分达到管理“有状态应用”的目的（你可以自己试着编写这样的YAML）。</p><p>但是Kubernetes的眼光则更加全面和长远，它认为“状态”不仅仅是数据持久化，在集群化、分布式的场景里，还有多实例的依赖关系、启动顺序和网络标识等问题需要解决，而这些问题恰恰是Deployment力所不及的。</p><p>因为只使用Deployment，多个实例之间是无关的，启动的顺序不固定，Pod的名字、IP地址、域名也都是完全随机的，这正是“无状态应用”的特点。</p><p>但对于“有状态应用”，多个实例之间可能存在依赖关系，比如master/slave、active/passive，需要依次启动才能保证应用正常运行，外界的客户端也可能要使用固定的网络标识来访问实例，而且这些信息还必须要保证在Pod重启后不变。</p><p>所以，Kubernetes就在Deployment的基础之上定义了一个新的API对象，名字也很好理解，就叫StatefulSet，专门用来管理有状态的应用。</p><h2>如何使用YAML描述StatefulSet</h2><p>首先我们还是用命令 <code>kubectl api-resources</code> 来查看StatefulSet的基本信息，可以知道它的简称是 <code>sts</code>，YAML文件头信息是：</p><pre><code class="language-yaml">apiVersion: apps/v1
kind: StatefulSet
metadata:
&nbsp; name: xxx-sts
</code></pre><p>和DaemonSet类似，StatefulSet也可以看做是Deployment的一个特例，它也不能直接用 <code>kubectl create</code> 创建样板文件，但它的对象描述和Deployment差不多，你同样可以把Deployment适当修改一下，就变成了StatefulSet对象。</p><p>这里我给出了一个使用Redis的StatefulSet，你来看看它与Deployment有什么差异：</p><pre><code class="language-yaml">apiVersion: apps/v1
kind: StatefulSet
metadata:
&nbsp; name: redis-sts

spec:
&nbsp; serviceName: redis-svc
&nbsp; replicas: 2
&nbsp; selector:
&nbsp; &nbsp; matchLabels:
&nbsp; &nbsp; &nbsp; app: redis-sts

&nbsp; template:
&nbsp; &nbsp; metadata:
&nbsp; &nbsp; &nbsp; labels:
&nbsp; &nbsp; &nbsp; &nbsp; app: redis-sts
&nbsp; &nbsp; spec:
&nbsp; &nbsp; &nbsp; containers:
&nbsp; &nbsp; &nbsp; - image: redis:5-alpine
&nbsp; &nbsp; &nbsp; &nbsp; name: redis
&nbsp; &nbsp; &nbsp; &nbsp; ports:
&nbsp; &nbsp; &nbsp; &nbsp; - containerPort: 6379
</code></pre><p>我们会发现，YAML文件里除了 <code>kind</code> 必须是“<strong>StatefulSet</strong>”，在 <code>spec</code> 里还多出了一个“<strong>serviceName</strong>”字段，其余的部分和Deployment是一模一样的，比如 <code>replicas</code>、<code>selector</code>、<code>template</code> 等等。</p><p>这两个不同之处其实就是StatefulSet与Deployment的关键区别。想要真正理解这一点，我们得结合StatefulSet在Kubernetes里的使用方法来分析。</p><h2>如何在Kubernetes里使用StatefulSet</h2><p>让我们用 <code>kubectl apply</code> 创建StatefulSet对象，用 <code>kubectl get</code> 先看看它是什么样的：</p><pre><code class="language-plain">kubectl apply -f redis-sts.yml
kubectl get sts
kubectl get pod
</code></pre><p><img src="https://static001.geekbang.org/resource/image/71/88/71b485401dca6946fe4788fa97e3fd88.png?wh=1268x414" alt="图片"></p><p>从截图里，你应该能够看到，StatefulSet所管理的Pod不再是随机的名字了，而是有了顺序编号，从0开始分别被命名为 <code>redis-sts-0</code>、<code>redis-sts-1</code>，Kubernetes也会按照这个顺序依次创建（0号比1号的AGE要长一点），这就解决了<strong>“有状态应用”的第一个问题：启动顺序</strong>。</p><p>有了启动的先后顺序，应用该怎么知道自己的身份，进而确定互相之间的依赖关系呢？</p><p>Kubernetes给出的方法是<strong>使用hostname</strong>，也就是每个Pod里的主机名，让我们再用 <code>kubectl exec</code> 登录Pod内部看看：</p><pre><code class="language-plain">kubectl exec -it redis-sts-0 -- sh
</code></pre><p><img src="https://static001.geekbang.org/resource/image/be/39/be44f94eaf07f3591c7a2a8b9cdd1739.png?wh=1308x468" alt="图片"></p><p>在Pod里查看环境变量 <code>$HOSTNAME</code> 或者是执行命令 <code>hostname</code>，都可以得到这个Pod的名字 <code>redis-sts-0</code>。</p><p>有了这个唯一的名字，应用就可以自行决定依赖关系了，比如在这个Redis例子里，就可以让先启动的0号Pod是主实例，后启动的1号Pod是从实例。</p><p>解决了启动顺序和依赖关系，还剩下<strong>第三个问题：网络标识，这就需要用到Service对象</strong>。</p><p>不过这里又有一点奇怪的地方，我们不能用命令 <code>kubectl expose</code> 直接为StatefulSet生成Service，只能手动编写YAML。但是这肯定难不倒你，经过了这么多练习，现在你应该能很轻松地写出一个Service对象。</p><p>因为不能自动生成，你在写Service对象的时候要小心一些，<code>metadata.name</code> 必须和StatefulSet里的 <code>serviceName</code> 相同，<code>selector</code> 里的标签也必须和StatefulSet里的一致：</p><pre><code class="language-yaml">apiVersion: v1
kind: Service
metadata:
&nbsp; name: redis-svc

spec:
&nbsp; selector:
&nbsp; &nbsp; app: redis-sts

&nbsp; ports:
&nbsp; - port: 6379
&nbsp; &nbsp; protocol: TCP
&nbsp; &nbsp; targetPort: 6379
</code></pre><p>写好Service之后，还是用 <code>kubectl apply</code> 创建这个对象：</p><p><img src="https://static001.geekbang.org/resource/image/5f/c8/5f8e4dbedaa563801bb6bbe09c441dc8.png?wh=1584x1056" alt="图片"></p><p>可以看到这个Service并没有什么特殊的地方，也是用标签选择器找到StatefulSet管理的两个Pod，然后找到它们的IP地址。</p><p>不过，StatefulSet的奥秘就在它的域名上。</p><p>还记得在<a href="https://time.geekbang.org/column/article/536829">第20讲</a>里我们说过的Service的域名用法吗？Service自己会有一个域名，格式是“<strong>对象名.名字空间</strong>”，每个Pod也会有一个域名，形式是“<strong>IP地址.名字空间</strong>”。但因为IP地址不稳定，所以Pod的域名并不实用，一般我们会使用稳定的Service域名。</p><p>当我们把Service对象应用于StatefulSet的时候，情况就不一样了。</p><p>Service发现这些Pod不是一般的应用，而是有状态应用，需要有稳定的网络标识，所以就会为Pod再多创建出一个新的域名，格式是“<strong>Pod名.服务名.名字空间.svc.cluster.local</strong>”。当然，这个域名也可以简写成“<strong>Pod名.服务名</strong>”。</p><p>我们还是用 <code>kubectl exec</code> 进入Pod内部，用ping命令来验证一下：</p><pre><code class="language-plain">kubectl exec -it redis-sts-0 -- sh
</code></pre><p><img src="https://static001.geekbang.org/resource/image/f1/39/f1b058b5fb3e5218c638ca0534b92439.png?wh=1524x1338" alt="图片"></p><p>显然，在StatefulSet里的这两个Pod都有了各自的域名，也就是稳定的网络标识。那么接下来，外部的客户端只要知道了StatefulSet对象，就可以用固定的编号去访问某个具体的实例了，虽然Pod的IP地址可能会变，但这个有编号的域名由Service对象维护，是稳定不变的。</p><p>到这里，通过StatefulSet和Service的联合使用，Kubernetes就解决了“有状态应用”的依赖关系、启动顺序和网络标识这三个问题，剩下的多实例之间内部沟通协调等事情就需要应用自己去想办法处理了。</p><p>关于Service，有一点值得再多提一下。</p><p>Service原本的目的是负载均衡，应该由它在Pod前面来转发流量，但是对StatefulSet来说，这项功能反而是不必要的，因为Pod已经有了稳定的域名，外界访问服务就不应该再通过Service这一层了。所以，从安全和节约系统资源的角度考虑，<strong>我们可以在Service里添加一个字段 <code>clusterIP: None</code> ，告诉Kubernetes不必再为这个对象分配IP地址</strong>。</p><p>我画了一张图展示StatefulSet与Service对象的关系，你可以参考一下它们字段之间的互相引用：</p><p><img src="https://static001.geekbang.org/resource/image/49/22/490d814cf0f25db56537a20f3af57e22.jpg?wh=1920x1094" alt="图片"></p><h2>如何实现StatefulSet的数据持久化</h2><p>现在StatefulSet已经有了固定的名字、启动顺序和网络标识，只要再给它加上数据持久化功能，我们就可以实现对“有状态应用”的管理了。</p><p>这里就能用到上一节课里学的PersistentVolume和NFS的知识，我们可以很容易地定义StorageClass，然后编写PVC，再给Pod挂载Volume。</p><p>不过，为了强调持久化存储与StatefulSet的一对一绑定关系，Kubernetes为StatefulSet专门定义了一个字段“<strong>volumeClaimTemplates</strong>”，直接把PVC定义嵌入StatefulSet的YAML文件里。这样能保证创建StatefulSet的同时，就会为每个Pod自动创建PVC，让StatefulSet的可用性更高。</p><p>“<strong>volumeClaimTemplates</strong>”这个字段好像有点难以理解，你可以把它和Pod的 <code>template</code>、Job的 <code>jobTemplate</code> 对比起来学习，它其实也是一个“套娃”的对象组合结构，里面就是应用了StorageClass的普通PVC而已。</p><p>让我们把刚才的Redis StatefulSet对象稍微改造一下，加上持久化存储功能：</p><pre><code class="language-yaml">apiVersion: apps/v1
kind: StatefulSet
metadata:
&nbsp; name: redis-pv-sts

spec:
&nbsp; serviceName: redis-pv-svc

&nbsp; volumeClaimTemplates:
&nbsp; - metadata:
&nbsp; &nbsp; &nbsp; name: redis-100m-pvc
&nbsp; &nbsp; spec:
&nbsp; &nbsp; &nbsp; storageClassName: nfs-client
&nbsp; &nbsp; &nbsp; accessModes:
&nbsp; &nbsp; &nbsp; &nbsp; - ReadWriteMany
&nbsp; &nbsp; &nbsp; resources:
&nbsp; &nbsp; &nbsp; &nbsp; requests:
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; storage: 100Mi

&nbsp; replicas: 2
&nbsp; selector:
&nbsp; &nbsp; matchLabels:
&nbsp; &nbsp; &nbsp; app: redis-pv-sts

&nbsp; template:
&nbsp; &nbsp; metadata:
&nbsp; &nbsp; &nbsp; labels:
&nbsp; &nbsp; &nbsp; &nbsp; app: redis-pv-sts
&nbsp; &nbsp; spec:
&nbsp; &nbsp; &nbsp; containers:
&nbsp; &nbsp; &nbsp; - image: redis:5-alpine
&nbsp; &nbsp; &nbsp; &nbsp; name: redis
&nbsp; &nbsp; &nbsp; &nbsp; ports:
&nbsp; &nbsp; &nbsp; &nbsp; - containerPort: 6379

&nbsp; &nbsp; &nbsp; &nbsp; volumeMounts:
&nbsp; &nbsp; &nbsp; &nbsp; - name: redis-100m-pvc
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; mountPath: /data
</code></pre><p>这个YAML文件比较长，内容比较多，不过你只要有点耐心，分功能模块逐个去看也能很快看明白。</p><p>首先StatefulSet对象的名字是 <code>redis-pv-sts</code>，表示它使用了PV存储。然后“volumeClaimTemplates”里定义了一个PVC，名字是 <code>redis-100m-pvc</code>，申请了100MB的NFS存储。在Pod模板里用 <code>volumeMounts</code> 引用了这个PVC，把网盘挂载到了 <code>/data</code> 目录，也就是Redis的数据目录。</p><p>下面的这张图就是这个StatefulSet对象完整的关系图：<br>
<img src="https://static001.geekbang.org/resource/image/1a/0f/1a06987c87f3db948b591883a81bac0f.jpg?wh=4000x2946" alt=""></p><p>最后使用 <code>kubectl apply</code> 创建这些对象，一个带持久化功能的“有状态应用”就算是运行起来了：</p><pre><code class="language-plain">kubectl apply -f redis-pv-sts.yml
</code></pre><p>你可以使用命令 <code>kubectl get pvc</code> 来查看StatefulSet关联的存储卷状态：</p><p><img src="https://static001.geekbang.org/resource/image/33/f5/33eee3c5a5033e4bf73f5003669c4ff5.png?wh=1920x189" alt="图片"></p><p>看这两个PVC的命名，不是随机的，是有规律的，用的是PVC名字加上StatefulSet的名字组合而成，所以即使Pod被销毁，因为它的名字不变，还能够找到这个PVC，再次绑定使用之前存储的数据。</p><p>那我们就来实地验证一下吧，用 <code>kubectl exec</code> 运行Redis的客户端，在里面添加一些KV数据：</p><pre><code class="language-plain">kubectl exec -it redis-pv-sts-0 -- redis-cli
</code></pre><p><img src="https://static001.geekbang.org/resource/image/94/b7/94a96b1b8a000dcd852d2ea11yy8ddb7.png?wh=1562x530" alt="图片"></p><p>这里我设置了两个值，分别是 <code>a=111</code> 和 <code>b=222</code>。</p><p>现在我们模拟意外事故，删除这个Pod：</p><pre><code class="language-plain">kubectl delete pod redis-pv-sts-0
</code></pre><p>由于StatefulSet和Deployment一样会监控Pod的实例，发现Pod数量少了就会很快创建出新的Pod，并且名字、网络标识也都会和之前的Pod一模一样：</p><p><img src="https://static001.geekbang.org/resource/image/52/23/52e2f02a1d80d8bba2a42c8258cda923.png?wh=1300x236" alt="图片"></p><p>那Redis里存储的数据怎么样了呢？是不是真的用到了持久化存储，也完全恢复了呢？</p><p>你可以再用Redis客户端登录去检查一下：</p><pre><code class="language-plain">kubectl exec -it redis-pv-sts-0 -- redis-cli
</code></pre><p><img src="https://static001.geekbang.org/resource/image/c7/08/c78ca845ee20459dd2d8bayy3db71808.png?wh=1544x530" alt="图片"></p><p>因为我们把NFS网络存储挂载到了Pod的 <code>/data</code> 目录，Redis就会定期把数据落盘保存，所以新创建的Pod再次挂载目录的时候会从备份文件里恢复数据，内存里的数据就恢复原状了。</p><h2>小结</h2><p>好了，今天我们学习了专门部署“有状态应用”的API对象StatefulSet，它与Deployment非常相似，区别是由它管理的Pod会有固定的名字、启动顺序和网络标识，这些特性对于在集群里实施有主从、主备等关系的应用非常重要。</p><p>我再简单小结一下今天的内容：</p><ol>
<li>StatefulSet的YAML描述和Deployment几乎完全相同，只是多了一个关键字段 <code>serviceName</code>。</li>
<li>要为StatefulSet里的Pod生成稳定的域名，需要定义Service对象，它的名字必须和StatefulSet里的 <code>serviceName</code> 一致。</li>
<li>访问StatefulSet应该使用每个Pod的单独域名，形式是“Pod名.服务名”，不应该使用Service的负载均衡功能。</li>
<li>在StatefulSet里可以用字段“volumeClaimTemplates”直接定义PVC，让Pod实现数据持久化存储。</li>
</ol><h2>课下作业</h2><p>最后是课下作业时间，给你留两个思考题：</p><ol>
<li>有了StatefulSet提供的固定名字和启动顺序，应用还需要怎么做才能实现主从等依赖关系呢？</li>
<li>是否可以不使用“volumeClaimTemplates”内嵌定义PVC呢？会有什么样的后果呢？</li>
</ol><p>欢迎在留言区参与讨论，分享你的想法。我们下节课再见。</p><p><img src="https://static001.geekbang.org/resource/image/88/e5/884a5c91b82cb515c856ce2ece6a91e5.jpg?wh=1920x1544" alt="图片"></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/2c/7e/f1efd18b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>摊牌</span>
  </div>
  <div class="_2_QraFYR_0">老师，既然statefulSet对象管理的pod可以直接通过域名指定来访问，那可不可以 不给statefulSet对象创建service</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不行，域名必须通过Service对象才能实现，可以自己试试没有Service对象会怎么样。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-01 22:06:29</div>
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
  <div class="_2_QraFYR_0">好奇redis的主从，哨兵，cluster都是怎么在sts上实现的，打算抽个时间深入的学习一下。<br><br>btw，越学习越能理解到 老师讲的“云原生”的概念了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Redis的主从部署直接用StatefulSet来实现还是很麻烦的，这个可能用operator来做会比较简单。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-22 20:07:12</div>
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
  <div class="_2_QraFYR_0">有了 StatefulSet 提供的固定名字和启动顺序，应用还需要怎么做才能实现主从等依赖关系呢？<br><br>答：我理解是采用StatefulSet对象管理多个（2n+1）有状态pod的情形下，应该在有状态应用中基于pod的固定名字进行实例通信交互，比如redis集群中节点之间通过Gossip协议进行广播自身的状态信息，从而完成实例之间依赖关系，保证集群的可用性</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-01 22:25:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/cd/db/7467ad23.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Bachue Zhou</span>
  </div>
  <div class="_2_QraFYR_0">我感觉 statefulset 起到的作用相比于普通的 systemd 差不多，特别是对于数据库这种真正有状态的服务而言，实例运行的节点通常是固定的，因为对硬件的要求要比普通的节点高很多，且在生产环境不可能用任何基于网络的文件系统来存储数据库文件。由于节点固定，所以 ip 也就固定，没必要非用域名来访问，而且现在有些服务本来也实现了服务发现，客户端连接集群的任意实例都可以获取完整集群节点的 ip 就可以直连，改用域名反而不太直接，statefulset 也不能让主从配置或是sharding配置变得更方便。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: StatefulSet的功能还是比较弱的，直接用还不是很方便，所以后来才出了operator等等，但StatefulSet无疑是基础。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-18 19:34:49</div>
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
  <div class="_2_QraFYR_0">作业：<br>1. 这个应该具体应用具体设置吧。比如 Redis ，需要给 主、从 实例加载不同的 conf 。以我目前的 kube 知识我不知道如何给不同的副本使用不同的配置文件。我只能使用临时命令实现主从 kubectl exec -it redis-pv-sts-1 -- redis-cli replicaof redis-pv-sts-0.redis-svc 6379 。<br>2. 若不使用“volumeClaimTemplates”内嵌定义 PVC，那么可能的后果就是，多个副本挂载同一个网络存储设备，这可能会导致数据丢失。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1. 用StatefulSet实现主从还是比较麻烦的，可以根据hostname，在镜像里编写脚本来启动不同的逻辑。<br><br>2.正确。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-16 18:47:46</div>
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
  <div class="_2_QraFYR_0">老师 能讲解一下什么是operator吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: operator比较复杂，可能在我们这个课里不会细讲了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-14 09:24:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/65/da/29fe3dde.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小宝</span>
  </div>
  <div class="_2_QraFYR_0">“访问 StatefulSet 应该使用每个 Pod 的单独域名，形式是“Pod 名. 服务名”，不应该使用 Service 的负载均衡功能。”<br>请教老师，通常会在StatefulSet上创建一个Headless Service吧，作为pod的负载均衡。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-23 18:30:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2b/9e/e5/9e732ec1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>泽</span>
  </div>
  <div class="_2_QraFYR_0">老师 求您个事 ，讲讲helm吧，迫切想学</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我对helm了解不是太深，简单讲讲倒是可以，但可能只是大概的介绍，有时间会写一篇。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-09 14:53:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/45/c3/775fe460.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>rubys_</span>
  </div>
  <div class="_2_QraFYR_0">在我的虚拟机上 ping redis-sts-1.redis-svc 失败，一种解决方案是，kubectl get pod -o wide -n kube-system 找到 coredns 的 pod，然后删除那两个 pod，比如 kubectl delete pod coredns-65c54cc984-qlkt9 -n kube-system。等待 k8s 重新创建 coredns 的 pod 就可以 ping 了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 有的时候coredns会有错误，可以删除重启，或者用rollout restart。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-25 10:57:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2c/45/44/8df79d3c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>事已至此开始撤退</span>
  </div>
  <div class="_2_QraFYR_0">实践看这个，底层原理看张坤</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-23 14:23:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/ec/29/895dbe3f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Leon</span>
  </div>
  <div class="_2_QraFYR_0">老师，我发现StatefulSet里面的volumeClaimTemplates配置的PVC在用于多个POD时，容器中挂载的NFS文件系统是分别两个不同的pvc，这样两个redis中的数据还是分别隔离的，这样是正常的吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，模板会创建各自独立的pv，因为存储是和Pod绑定的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-05 15:57:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>庄颖</span>
  </div>
  <div class="_2_QraFYR_0">老师，请问下文章最后的pvc挂载，直接使用deployment添加pvc挂载，删除对应的pod，或者删除deployment重建，只要对应的pvc名称不变，数据也不会变化啊。<br>这是不是意味着deployment和statefulset在使用pvc持久化数据时，是没什么去别的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: kubernetes里存储和应用是分离的，所以这两者没有必然的关系，只是StatefulSet在创建PVC时有特别的规则。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-22 14:19:50</div>
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
  <div class="_2_QraFYR_0">经过测试创建svc时候有个小坑，service的name必须和sts的serviceName一致，不然域名解析不到</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个是当然的，不过确实容易忽视。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-18 16:09:59</div>
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
  <div class="_2_QraFYR_0">将svc转换为headless service时，修改redis-svc.yml，添加clusterIP: None后，<br>执行kubectl apply -f redis-svc.yml更新svc时会报spec.clusterIPs[0]: Invalid value，<br>需要先将svc删除后重新apply才行，直接更新不起来。<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-17 11:31:46</div>
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
  <div class="_2_QraFYR_0">Pod负责服务，Job负责调度，<br>Daemon&#47;Deployment负责无状态部署，StatefulSet负责状态部署，<br>Service负责四层访问（负载均衡、IP分配、域名访问），Ingress负责应用层（7层）访问（路由规则），<br>PVC&#47;PV负责可靠性存储。<br><br>K8s提供的解决方案基本就是代表了微服务部署的最佳实践了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great.</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-01 00:06:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/23/b9/6f/b40d1acf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>mkcaptain</span>
  </div>
  <div class="_2_QraFYR_0">老师，有个疑问，今天讲解的中，如果固定域名绑定了pod，那结尾数字1的那个有什么作用呢？感觉一直都只会用0结尾的那个？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Kubernetes只是提供了域名解析，如何利用每个pod的域名是业务层面的事情了，需要我们自己根据业务去考虑，课程里只是示例，比较简单。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-25 09:57:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/30/5b/4f4b0a40.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>悟远</span>
  </div>
  <div class="_2_QraFYR_0">老师好，我有个需求，想部署分布式爬虫环境，有什么方案可以让集群中所有副本同时启动执行任务吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是不是用Job就行了，应该不用StatefulSet吧。看看Kubernetes里的这些对象哪个最符合你的实际需求，看人下菜。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-03 19:09:19</div>
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
  <div class="_2_QraFYR_0">问题1： 想办法给这两个配置上配置文件，但感觉比较麻烦。 问题2： 不知道，我看答案说的是可能会多个挂载一个nfs，但如果是statefulset里面设置就不会发生这样的事情吗？<br><br>下面是一些总结和思考：<br>1. 有状态的状态，指的是启动顺序，机器标识和网络标识，网络标识是通过svc实现的。通过sts和svc的绑定，可以让pod拥有自己的域名，可以被直接访问到。<br>2. 有状态的状态对比起无状态服务，本质上来说，就是有更多的确定性，当一个deployment启动的时候，pod名称很多时候是随机的，但sts设置好以后，名称是固定的，hostname也是固定的，网络域名也是固定的，这些状态被固定住。<br>3. 有一个问题是，在docker-compose中，其实着一些东西也是固定的，无状态的设计是新加的，而有状态的服务是后面来的，依赖关系也在docker-compose中设计好，那么问题来了，为什么会有无状态服务，为什么不把所有服务都设计为有状态服务，我想的是不管有没有必要，都规范不就好了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这些问题可以参考其他同学的回答，重要的是思考的过程。<br><br>有状态服务不好维护不方便扩容缩容，无状态服务管理简单。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-18 23:25:30</div>
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
  <div class="_2_QraFYR_0">执行了redis-pv-sts.yml，生产pvc一直是pending，然后报错如下<br><br>persistentvolume-controller  waiting for a volume to be created, either by external provisioner &quot;k8s-sigs.io&#47;nfs-subdir-external-provisioner&quot; or manually created by system administrator</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 看看nfs是否工作正常，然后再看看Provisioner的状态。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-02 17:35:59</div>
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
  <div class="_2_QraFYR_0">请教老师几个问题：<br>Q1：怎么查询一个POD是通过什么创建的？<br>用“delete -f . ”来删除，发现只有最近的通过YAML创建的被删除，比如今天上午通过YAML创建了两个POD，用“delete -f . ”只删除了刚创建的这个sts，但以前通过deploy YAML创建的POD无法删除。为了删除其它POD,需要知道这个POD是通过什么创建的，请问：用哪个命令可以查看一个POD是通过什么创建的？<br>Q2：应用怎么控制启动顺序？<br>执行“kubectl apply -f redis-sts.yml”以后，两个POD被K8S启动了，已经启动了，应用还能控制启动顺序吗？<br>Q3：什么命令可以列出所有创建的对象？<br>目的是想查看以前创建的对象，各种类型的对象。是否有一个命令可以列出所有创建过得对象？<br>Q4：怎么判断console上安装的NFS server是否在运行，或者说怎么判断其状态？ Nfs version吗？同样，在client上，开机后怎么判断nfs client状态？（也许开机后自动运行？）</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1. 这个还真不知道，也许只能加强运维管理了吧。<br><br>2.应用不能控制，需要自己编写脚本，判断hostname得知自己是第几个启动的。<br><br>3. kubectl get all -A<br><br>4.用showmount检查，还有service等系统管理命令。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-23 14:58:17</div>
  </div>
</div>
</div>
</li>
</ul>