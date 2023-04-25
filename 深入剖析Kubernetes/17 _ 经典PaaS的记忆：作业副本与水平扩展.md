<audio title="17 _ 经典PaaS的记忆：作业副本与水平扩展" src="https://static001.geekbang.org/resource/audio/e4/bd/e443846dd934662f1084a0e8c4984abd.mp3" controls="controls"></audio> 
<p>你好，我是张磊。今天我和你分享的主题是：经典PaaS的记忆之作业副本与水平扩展。</p><p>在上一篇文章中，我为你详细介绍了Kubernetes项目中第一个重要的设计思想：控制器模式。</p><p>而在今天这篇文章中，我就来为你详细讲解一下，Kubernetes里第一个控制器模式的完整实现：Deployment。</p><p>Deployment看似简单，但实际上，它实现了Kubernetes项目中一个非常重要的功能：<span class="orange">Pod的“水平扩展/收缩”（horizontal scaling out/in）</span>。这个功能，是从PaaS时代开始，一个平台级项目就必须具备的编排能力。</p><p>举个例子，如果你更新了Deployment的Pod模板（比如，修改了容器的镜像），那么Deployment就需要遵循一种叫作“滚动更新”（rolling update）的方式，来升级现有的容器。</p><p>而这个能力的实现，依赖的是Kubernetes项目中的一个非常重要的概念（API对象）：ReplicaSet。</p><p>ReplicaSet的结构非常简单，我们可以通过这个YAML文件查看一下：</p><pre><code>apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-set
  labels:
    app: nginx
spec:
  replicas: 3
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
        image: nginx:1.7.9
</code></pre><p>从这个YAML文件中，我们可以看到，<strong>一个ReplicaSet对象，其实就是由副本数目的定义和一个Pod模板组成的</strong>。不难发现，它的定义其实是Deployment的一个子集。</p><!-- [[[read_end]]] --><p><strong>更重要的是，Deployment控制器实际操纵的，正是这样的ReplicaSet对象，而不是Pod对象。</strong></p><p>还记不记得我在上一篇文章<a href="https://time.geekbang.org/column/article/40583">《编排其实很简单：谈谈“控制器”模型》</a>中曾经提出过这样一个问题：对于一个Deployment所管理的Pod，它的ownerReference是谁？</p><p>所以，这个问题的答案就是：ReplicaSet。</p><p>明白了这个原理，我再来和你一起分析一个如下所示的Deployment：</p><pre><code>apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
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
        image: nginx:1.7.9
        ports:
        - containerPort: 80
</code></pre><p>可以看到，这就是一个我们常用的nginx-deployment，它定义的Pod副本个数是3（spec.replicas=3）。</p><p>那么，在具体的实现上，这个Deployment，与ReplicaSet，以及Pod的关系是怎样的呢？</p><p>我们可以用一张图把它描述出来：</p><p><img src="https://static001.geekbang.org/resource/image/71/58/711c07208358208e91fa7803ebc73058.jpg?wh=2248*1346" alt=""></p><p>通过这张图，我们就很清楚地看到，一个定义了replicas=3的Deployment，与它的ReplicaSet，以及Pod的关系，实际上是一种“层层控制”的关系。</p><p>其中，ReplicaSet负责通过“控制器模式”，保证系统中Pod的个数永远等于指定的个数（比如，3个）。这也正是Deployment只允许容器的restartPolicy=Always的主要原因：只有在容器能保证自己始终是Running状态的前提下，ReplicaSet调整Pod的个数才有意义。</p><p>而在此基础上，Deployment同样通过“控制器模式”，来操作ReplicaSet的个数和属性，进而实现“水平扩展/收缩”和“滚动更新”这两个编排动作。</p><p>其中，“水平扩展/收缩”非常容易实现，Deployment Controller只需要修改它所控制的ReplicaSet的Pod副本个数就可以了。</p><p>比如，把这个值从3改成4，那么Deployment所对应的ReplicaSet，就会根据修改后的值自动创建一个新的Pod。这就是“水平扩展”了；“水平收缩”则反之。</p><p>而用户想要执行这个操作的指令也非常简单，就是kubectl scale，比如：</p><pre><code>$ kubectl scale deployment nginx-deployment --replicas=4
deployment.apps/nginx-deployment scaled
</code></pre><p>那么，<span class="orange">“滚动更新”又是什么意思，是如何实现的呢？</span></p><p>接下来，我还以这个Deployment为例，来为你讲解“滚动更新”的过程。</p><p>首先，我们来创建这个nginx-deployment：</p><pre><code>$ kubectl create -f nginx-deployment.yaml --record
</code></pre><p>注意，在这里，我额外加了一个–record参数。它的作用，是记录下你每次操作所执行的命令，以方便后面查看。</p><p>然后，我们来检查一下nginx-deployment创建后的状态信息：</p><pre><code>$ kubectl get deployments
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3         0         0            0           1s
</code></pre><p>在返回结果中，我们可以看到四个状态字段，它们的含义如下所示。</p><ol>
<li>
<p>DESIRED：用户期望的Pod副本个数（spec.replicas的值）；</p>
</li>
<li>
<p>CURRENT：当前处于Running状态的Pod的个数；</p>
</li>
<li>
<p>UP-TO-DATE：当前处于最新版本的Pod的个数，所谓最新版本指的是Pod的Spec部分与Deployment里Pod模板里定义的完全一致；</p>
</li>
<li>
<p>AVAILABLE：当前已经可用的Pod的个数，即：既是Running状态，又是最新版本，并且已经处于Ready（健康检查正确）状态的Pod的个数。</p>
</li>
</ol><p>可以看到，只有这个AVAILABLE字段，描述的才是用户所期望的最终状态。</p><p>而Kubernetes项目还为我们提供了一条指令，让我们可以实时查看Deployment对象的状态变化。这个指令就是kubectl rollout status：</p><pre><code>$ kubectl rollout status deployment/nginx-deployment
Waiting for rollout to finish: 2 out of 3 new replicas have been updated...
deployment.apps/nginx-deployment successfully rolled out
</code></pre><p>在这个返回结果中，“2 out of 3 new replicas have been updated”意味着已经有2个Pod进入了UP-TO-DATE状态。</p><p>继续等待一会儿，我们就能看到这个Deployment的3个Pod，就进入到了AVAILABLE状态：</p><pre><code>NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3         3         3            3           20s
</code></pre><p>此时，你可以尝试查看一下这个Deployment所控制的ReplicaSet：</p><pre><code>$ kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-3167673210   3         3         3       20s
</code></pre><p>如上所示，在用户提交了一个Deployment对象后，Deployment Controller就会立即创建一个Pod副本个数为3的ReplicaSet。这个ReplicaSet的名字，则是由Deployment的名字和一个随机字符串共同组成。</p><p>这个随机字符串叫作pod-template-hash，在我们这个例子里就是：3167673210。ReplicaSet会把这个随机字符串加在它所控制的所有Pod的标签里，从而保证这些Pod不会与集群里的其他Pod混淆。</p><p>而ReplicaSet的DESIRED、CURRENT和READY字段的含义，和Deployment中是一致的。所以，<strong>相比之下，Deployment只是在ReplicaSet的基础上，添加了UP-TO-DATE这个跟版本有关的状态字段。</strong></p><p>这个时候，如果我们修改了Deployment的Pod模板，“滚动更新”就会被自动触发。</p><p>修改Deployment有很多方法。比如，我可以直接使用kubectl edit指令编辑Etcd里的API对象。</p><pre><code>$ kubectl edit deployment/nginx-deployment
... 
    spec:
      containers:
      - name: nginx
        image: nginx:1.9.1 # 1.7.9 -&gt; 1.9.1
        ports:
        - containerPort: 80
...
deployment.extensions/nginx-deployment edited
</code></pre><p>这个kubectl edit指令，会帮你直接打开nginx-deployment的API对象。然后，你就可以修改这里的Pod模板部分了。比如，在这里，我将nginx镜像的版本升级到了1.9.1。</p><blockquote>
<p>备注：kubectl edit并不神秘，它不过是把API对象的内容下载到了本地文件，让你修改完成后再提交上去。</p>
</blockquote><p>kubectl edit指令编辑完成后，保存退出，Kubernetes就会立刻触发“滚动更新”的过程。你还可以通过kubectl rollout status指令查看nginx-deployment的状态变化：</p><pre><code>$ kubectl rollout status deployment/nginx-deployment
Waiting for rollout to finish: 2 out of 3 new replicas have been updated...
deployment.extensions/nginx-deployment successfully rolled out
</code></pre><p>这时，你可以通过查看Deployment的Events，看到这个“滚动更新”的流程：</p><pre><code>$ kubectl describe deployment nginx-deployment
...
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
...
  Normal  ScalingReplicaSet  24s   deployment-controller  Scaled up replica set nginx-deployment-1764197365 to 1
  Normal  ScalingReplicaSet  22s   deployment-controller  Scaled down replica set nginx-deployment-3167673210 to 2
  Normal  ScalingReplicaSet  22s   deployment-controller  Scaled up replica set nginx-deployment-1764197365 to 2
  Normal  ScalingReplicaSet  19s   deployment-controller  Scaled down replica set nginx-deployment-3167673210 to 1
  Normal  ScalingReplicaSet  19s   deployment-controller  Scaled up replica set nginx-deployment-1764197365 to 3
  Normal  ScalingReplicaSet  14s   deployment-controller  Scaled down replica set nginx-deployment-3167673210 to 0
</code></pre><p>可以看到，首先，当你修改了Deployment里的Pod定义之后，Deployment Controller会使用这个修改后的Pod模板，创建一个新的ReplicaSet（hash=1764197365），这个新的ReplicaSet的初始Pod副本数是：0。</p><p>然后，在Age=24 s的位置，Deployment Controller开始将这个新的ReplicaSet所控制的Pod副本数从0个变成1个，即：“水平扩展”出一个副本。</p><p>紧接着，在Age=22 s的位置，Deployment Controller又将旧的ReplicaSet（hash=3167673210）所控制的旧Pod副本数减少一个，即：“水平收缩”成两个副本。</p><p>如此交替进行，新ReplicaSet管理的Pod副本数，从0个变成1个，再变成2个，最后变成3个。而旧的ReplicaSet管理的Pod副本数则从3个变成2个，再变成1个，最后变成0个。这样，就完成了这一组Pod的版本升级过程。</p><p>像这样，<strong>将一个集群中正在运行的多个Pod版本，交替地逐一升级的过程，就是“滚动更新”。</strong></p><p>在这个“滚动更新”过程完成之后，你可以查看一下新、旧两个ReplicaSet的最终状态：</p><pre><code>$ kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-1764197365   3         3         3       6s
nginx-deployment-3167673210   0         0         0       30s
</code></pre><p>其中，旧ReplicaSet（hash=3167673210）已经被“水平收缩”成了0个副本。</p><p><strong>这种“滚动更新”的好处是显而易见的。</strong></p><p>比如，在升级刚开始的时候，集群里只有1个新版本的Pod。如果这时，新版本Pod有问题启动不起来，那么“滚动更新”就会停止，从而允许开发和运维人员介入。而在这个过程中，由于应用本身还有两个旧版本的Pod在线，所以服务并不会受到太大的影响。</p><p>当然，这也就要求你一定要使用Pod的Health Check机制检查应用的运行状态，而不是简单地依赖于容器的Running状态。要不然的话，虽然容器已经变成Running了，但服务很有可能尚未启动，“滚动更新”的效果也就达不到了。</p><p>而为了进一步保证服务的连续性，Deployment Controller还会确保，在任何时间窗口内，只有指定比例的Pod处于离线状态。同时，它也会确保，在任何时间窗口内，只有指定比例的新Pod被创建出来。这两个比例的值都是可以配置的，默认都是DESIRED值的25%。</p><p>所以，在上面这个Deployment的例子中，它有3个Pod副本，那么控制器在“滚动更新”的过程中永远都会确保至少有2个Pod处于可用状态，至多只有4个Pod同时存在于集群中。这个策略，是Deployment对象的一个字段，名叫RollingUpdateStrategy，如下所示：</p><pre><code>apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
...
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
</code></pre><p>在上面这个RollingUpdateStrategy的配置中，maxSurge指定的是除了DESIRED数量之外，在一次“滚动”中，Deployment控制器还可以创建多少个新Pod；而maxUnavailable指的是，在一次“滚动”中，Deployment控制器可以删除多少个旧Pod。</p><p>同时，这两个配置还可以用前面我们介绍的百分比形式来表示，比如：maxUnavailable=50%，指的是我们最多可以一次删除“50%*DESIRED数量”个Pod。</p><p>结合以上讲述，现在我们可以扩展一下Deployment、ReplicaSet和Pod的关系图了。</p><p><img src="https://static001.geekbang.org/resource/image/bb/5d/bbc4560a053dee904e45ad66aac7145d.jpg?wh=2248*1346" alt=""></p><p>如上所示，Deployment的控制器，实际上控制的是ReplicaSet的数目，以及每个ReplicaSet的属性。</p><p>而一个应用的版本，对应的正是一个ReplicaSet；这个版本应用的Pod数量，则由ReplicaSet通过它自己的控制器（ReplicaSet Controller）来保证。</p><p>通过这样的多个ReplicaSet对象，Kubernetes项目就实现了对多个“应用版本”的描述。</p><p>而明白了“应用版本和ReplicaSet一一对应”的设计思想之后，我就可以为你讲解一下<span class="orange">Deployment对应用进行版本控制的具体原理</span>了。</p><p>这一次，我会使用一个叫<strong>kubectl set image</strong>的指令，直接修改nginx-deployment所使用的镜像。这个命令的好处就是，你可以不用像kubectl edit那样需要打开编辑器。</p><p>不过这一次，我把这个镜像名字修改成为了一个错误的名字，比如：nginx:1.91。这样，这个Deployment就会出现一个升级失败的版本。</p><p>我们一起来实践一下：</p><pre><code>$ kubectl set image deployment/nginx-deployment nginx=nginx:1.91
deployment.extensions/nginx-deployment image updated
</code></pre><p>由于这个nginx:1.91镜像在Docker Hub中并不存在，所以这个Deployment的“滚动更新”被触发后，会立刻报错并停止。</p><p>这时，我们来检查一下ReplicaSet的状态，如下所示：</p><pre><code>$ kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-1764197365   2         2         2       24s
nginx-deployment-3167673210   0         0         0       35s
nginx-deployment-2156724341   2         2         0       7s
</code></pre><p>通过这个返回结果，我们可以看到，新版本的ReplicaSet（hash=2156724341）的“水平扩展”已经停止。而且此时，它已经创建了两个Pod，但是它们都没有进入READY状态。这当然是因为这两个Pod都拉取不到有效的镜像。</p><p>与此同时，旧版本的ReplicaSet（hash=1764197365）的“水平收缩”，也自动停止了。此时，已经有一个旧Pod被删除，还剩下两个旧Pod。</p><p>那么问题来了， 我们如何让这个Deployment的3个Pod，都回滚到以前的旧版本呢？</p><p>我们只需要执行一条kubectl rollout undo命令，就能把整个Deployment回滚到上一个版本：</p><pre><code>$ kubectl rollout undo deployment/nginx-deployment
deployment.extensions/nginx-deployment
</code></pre><p>很容易想到，在具体操作上，Deployment的控制器，其实就是让这个旧ReplicaSet（hash=1764197365）再次“扩展”成3个Pod，而让新的ReplicaSet（hash=2156724341）重新“收缩”到0个Pod。</p><p>更进一步地，如果我想回滚到更早之前的版本，要怎么办呢？</p><p><strong>首先，我需要使用kubectl rollout history命令，查看每次Deployment变更对应的版本</strong>。而由于我们在创建这个Deployment的时候，指定了–record参数，所以我们创建这些版本时执行的kubectl命令，都会被记录下来。这个操作的输出如下所示：</p><pre><code>$ kubectl rollout history deployment/nginx-deployment
deployments &quot;nginx-deployment&quot;
REVISION    CHANGE-CAUSE
1           kubectl create -f nginx-deployment.yaml --record
2           kubectl edit deployment/nginx-deployment
3           kubectl set image deployment/nginx-deployment nginx=nginx:1.91
</code></pre><p>可以看到，我们前面执行的创建和更新操作，分别对应了版本1和版本2，而那次失败的更新操作，则对应的是版本3。</p><p>当然，你还可以通过这个kubectl rollout history指令，看到每个版本对应的Deployment的API对象的细节，具体命令如下所示：</p><pre><code>$ kubectl rollout history deployment/nginx-deployment --revision=2
</code></pre><p><strong>然后，我们就可以在kubectl rollout undo命令行最后，加上要回滚到的指定版本的版本号，就可以回滚到指定版本了</strong>。这个指令的用法如下：</p><pre><code>$ kubectl rollout undo deployment/nginx-deployment --to-revision=2
deployment.extensions/nginx-deployment
</code></pre><p>这样，Deployment Controller还会按照“滚动更新”的方式，完成对Deployment的降级操作。</p><p>不过，你可能已经想到了一个问题：我们对Deployment进行的每一次更新操作，都会生成一个新的ReplicaSet对象，是不是有些多余，甚至浪费资源呢？</p><p>没错。</p><p>所以，Kubernetes项目还提供了一个指令，使得我们对Deployment的多次更新操作，最后 只生成一个ReplicaSet。</p><p>具体的做法是，在更新Deployment前，你要先执行一条kubectl rollout pause指令。它的用法如下所示：</p><pre><code>$ kubectl rollout pause deployment/nginx-deployment
deployment.extensions/nginx-deployment paused
</code></pre><p>这个kubectl rollout pause的作用，是让这个Deployment进入了一个“暂停”状态。</p><p>所以接下来，你就可以随意使用kubectl edit或者kubectl set image指令，修改这个Deployment的内容了。</p><p>由于此时Deployment正处于“暂停”状态，所以我们对Deployment的所有修改，都不会触发新的“滚动更新”，也不会创建新的ReplicaSet。</p><p>而等到我们对Deployment修改操作都完成之后，只需要再执行一条kubectl rollout resume指令，就可以把这个Deployment“恢复”回来，如下所示：</p><pre><code>$ kubectl rollout resume deployment/nginx-deployment
deployment.extensions/nginx-deployment resumed
</code></pre><p>而在这个kubectl rollout resume指令执行之前，在kubectl rollout pause指令之后的这段时间里，我们对Deployment进行的所有修改，最后只会触发一次“滚动更新”。</p><p>当然，我们可以通过检查ReplicaSet状态的变化，来验证一下kubectl rollout pause和kubectl rollout resume指令的执行效果，如下所示：</p><pre><code>$ kubectl get rs
NAME               DESIRED   CURRENT   READY     AGE
nginx-1764197365   0         0         0         2m
nginx-3196763511   3         3         3         28s
</code></pre><p>通过返回结果，我们可以看到，只有一个hash=3196763511的ReplicaSet被创建了出来。</p><p>不过，即使你像上面这样小心翼翼地控制了ReplicaSet的生成数量，随着应用版本的不断增加，Kubernetes中还是会为同一个Deployment保存很多很多不同的ReplicaSet。</p><p>那么，我们又该如何控制这些“历史”ReplicaSet的数量呢？</p><p>很简单，Deployment对象有一个字段，叫作spec.revisionHistoryLimit，就是Kubernetes为Deployment保留的“历史版本”个数。所以，如果把它设置为0，你就再也不能做回滚操作了。</p><h2>总结</h2><p>在今天这篇文章中，我为你详细讲解了Deployment这个Kubernetes项目中最基本的编排控制器的实现原理和使用方法。</p><p>通过这些讲解，你应该了解到：Deployment实际上是一个<strong>两层控制器</strong>。首先，它通过<strong>ReplicaSet的个数</strong>来描述应用的版本；然后，它再通过<strong>ReplicaSet的属性</strong>（比如replicas的值），来保证Pod的副本数量。</p><blockquote>
<p>备注：Deployment控制ReplicaSet（版本），ReplicaSet控制Pod（副本数）。这个两层控制关系一定要牢记。</p>
</blockquote><p>不过，相信你也能够感受到，Kubernetes项目对Deployment的设计，实际上是代替我们完成了对“应用”的抽象，使得我们可以使用这个Deployment对象来描述应用，使用kubectl rollout命令控制应用的版本。</p><p>可是，在实际使用场景中，应用发布的流程往往千差万别，也可能有很多的定制化需求。比如，我的应用可能有会话黏连（session sticky），这就意味着“滚动更新”的时候，哪个Pod能下线，是不能随便选择的。</p><p>这种场景，光靠Deployment自己就很难应对了。对于这种需求，我在专栏后续文章中重点介绍的“自定义控制器”，就可以帮我们实现一个功能更加强大的Deployment Controller。</p><p>当然，Kubernetes项目本身，也提供了另外一种抽象方式，帮我们应对其他一些用Deployment无法处理的应用编排场景。这个设计，就是对有状态应用的管理，也是我在下一篇文章中要重点讲解的内容。</p><h2>思考题</h2><p>你听说过金丝雀发布（Canary Deployment）和蓝绿发布（Blue-Green Deployment）吗？你能说出它们是什么意思吗？</p><p>实际上，有了Deployment的能力之后，你可以非常轻松地用它来实现金丝雀发布、蓝绿发布，以及A/B测试等很多应用发布模式。这些问题的答案都在<a href="https://github.com/ContainerSolutions/k8s-deployment-strategies/tree/master/canary">这个GitHub库</a>，建议你在课后实践一下。</p><p>感谢你的收听，欢迎你给我留言，也欢迎分享给更多的朋友一起阅读。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/90/aa/83c31ba3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小龙</span>
  </div>
  <div class="_2_QraFYR_0">老师真的是厉害，我在极客时间买了17门课了，这个是含金量最高的一门</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 17门，你也厉害！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-01 11:12:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/57/37/8877f206.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>胖宝王</span>
  </div>
  <div class="_2_QraFYR_0">金丝雀部署：优先发布一台或少量机器升级，等验证无误后再更新其他机器。优点是用户影响范围小，不足之处是要额外控制如何做自动更新。<br>蓝绿部署：2组机器，蓝代表当前的V1版本，绿代表已经升级完成的V2版本。通过LB将流量全部导入V2完成升级部署。优点是切换快速，缺点是影响全部用户。<br>本文学习的滚动更新，我觉得就是一个自动化更新的金丝雀发布。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 说的没错，可以看看文末的例子实践一下</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-01 09:59:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/8b/52/6659dc1b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>黑米</span>
  </div>
  <div class="_2_QraFYR_0">老师真的是厉害，我在极客时间买了24门课了，这个是含金量第二高的一门</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不是第一高不开心</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-22 15:27:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/46/5b/07858c33.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Pixar</span>
  </div>
  <div class="_2_QraFYR_0">关于Pod的状态一直有一些疑问, 学完这节课就更混了, 囧.   还望老师能解答下.<br><br>`kubectl get deployments` 得到的 available 字段表示的是处于Running状态且健康检查通过的Pod, 这里有一个疑问: 健康检查不是针对Pod里面的Container吗? 如果某一个Pod里面包含多个Container, 但是这些Container健康检查有些并没有通过, 那么此时该Pod会出现在 available里面吗? <br><br>Pod通过健康检查是指里面所有的Container都通过吗?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 都通过</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-07 15:35:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/e1/21/5f0bf9a0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>阿文</span>
  </div>
  <div class="_2_QraFYR_0">注意，在这里，我额外加了一个–record 参数。它的作用，...<br><br>这个应该要解释下--record 是只记录当前命令，老师，你下面的命令没有加。history 里面只看到<br>   kubectl create --filename=nginx-deployment.yaml --record=true</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-16 16:12:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/20/10/5786e0d8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>bus801</span>
  </div>
  <div class="_2_QraFYR_0">如果我直接edit rs，将image修改成新的版本，是不是也能实现pod中容器镜像的更新？我试了一下，什么反应也没有。既然rs控制pod，为什么这样改不能生效呢？还请指教一下</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 因为rs controller 不处理rollout逻辑</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-19 18:25:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/7d/69/21b4b5cb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>王由华</span>
  </div>
  <div class="_2_QraFYR_0">有个问题，scale down时，k8s是对pod里的容器发送kill 信号吗？所以应用需要处理好这个信号？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 先term 再kill。需要处理。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-17 22:40:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/b0/eb/112e7d16.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我是一根葱</span>
  </div>
  <div class="_2_QraFYR_0">请教个问题，如果水平收缩的过程中，某个pod中的容器有正在运行的业务，而业务如果中断的话可能会导致数据库数据出错，该怎么办？如何保证把pod的业务执行完再收缩？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 业务需要优雅处理sig term</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-04 10:12:11</div>
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
  <div class="_2_QraFYR_0">在滚动更新的过程中，Service的流量转发会有怎样的变化呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: service只会代理readiness检查返回正确的pod</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-01 09:17:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/57/37/8877f206.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>胖宝王</span>
  </div>
  <div class="_2_QraFYR_0">金丝雀发布：先发布一台机器或少量机器，做流量验证。如果新版没问题在把剩余机器全部更新。优点是影响范围小，不足的是要自己想办法如何控制自动更新。<br>蓝绿部署：事先准备好一组机器(绿组)全部更新，然后调整LB将流量全部引到绿组。优点是切换快捷回滚方便。不足的是有问题则影响全部用户。<br>对于本文学习的 rolling update，我理解的像是自动化更新的金丝雀发布。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-01 09:48:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/8b/52/6659dc1b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>黑米</span>
  </div>
  <div class="_2_QraFYR_0">不用不高兴，你第二没人敢认第一👍</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-23 23:34:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/a4/95/474d5eaf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Tigerfive</span>
  </div>
  <div class="_2_QraFYR_0">半夜从火车上醒来，就来看看有没有更新，果然没有让我失望！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 国庆可以充电啦</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-01 02:28:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/f6/48/b7054856.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>原来只是梦</span>
  </div>
  <div class="_2_QraFYR_0">老师你好！之前有同学提到这个rollout像自动化的金丝雀发布，对于这一点我不太理解。发布的时候会是一个轮换的过程，也就是说用不了多少时间，就会都运行在新的rs，或者都回滚到老的rs(出错)。我的理解要金丝雀的话，需要保持同时存在两个rs一定时间，以确保新版本没问题。那么这个在k8s里是怎么实现呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-16 15:17:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q3auHgzwzM49ONuR097wB6LqR8nn5kWiaQiaPic1y8UznibDOScQergTj5qeL6zQ4bIicYEkqlMiash3CUCAYmSt9tQA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>哈希碰撞</span>
  </div>
  <div class="_2_QraFYR_0">一个 ReplicaSet 对象,不难发现是 Deployment 的一个子集？<br>请问怎么不难发现？ 我觉得很难发现。。。从示例的YAML文件内容上看，看不出任何关连。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-08 14:05:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Nokiak8</span>
  </div>
  <div class="_2_QraFYR_0">Cronjob 类型也有spec.revisionHistoryLimit么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: successfulJobsHistoryLimit和failedJobsHistoryLimit</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-12 07:18:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/df/e2/823a04b4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小小笑儿</span>
  </div>
  <div class="_2_QraFYR_0">有几个问题想请问一下:<br>1是在 deployment rollout undo 的时候，是也会创建一个新的rs对象吗？如果是的话那么这个rs的template hash不就重复了？如果不是得话又是如何处理的呢？<br>2是deployment 关注的应该是自身的api对象和rs的api对象，但是我看deployment controller 的源码中也关注了pod的变更，这是为了处理哪种情况？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 回滚又不是创建新版本，版本与rs一一对应，怎么会出现新的rs呢？滚动升级反向操作即可。<br><br>它只关心pod被全删除的情况，因为有一种滚动更新策略是这时候重新创建新的deployment </p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-01 19:54:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>思维决定未来</span>
  </div>
  <div class="_2_QraFYR_0">金丝雀和蓝绿发布实际是在流量入口(Ingress)来控制的，并不是通过其他Controller来控制Deployment，在更新时，直接部署一个新的Deployment，然后通过调整流量比例到不同的Deployment从而实现对应更新，而蓝绿和金丝雀的区别就是新版本的Deployment是否一次性将副本数开到跟原Deployment一样多。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-04 12:26:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>与君共勉</span>
  </div>
  <div class="_2_QraFYR_0">我看完了“kubenetes权威指南”这本书，感觉这本书很像是使用指导，深度不够，还是张老师的课程有些深度。“kubenetes权威指南”是这样讲Deployment和ReplicateSet的区别的：Deployment继承了ReplicateSet的所有功能，让我以为他们是继承关系，认为Deployment是增强版的ReplicateSet。看了老师的文章才知道，Deployment控制的是ReplicateSet，ReplicateSet控制的是pods数量，Deployment是通过ReplicateSet间接控制了pod的数目。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-07 21:27:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/6e/6b/9c3f3abb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>阿鹏</span>
  </div>
  <div class="_2_QraFYR_0">老师，我们公司准备试水k8s，我看网上很多文章都在说跨主机容器间通信的解决方案，如果我们的服务分批容器化，需要解决宿主机网络和容器网络的互通，我用flannel或者calico目前都只能做到宿主机能访问容器网络或者容器能访问宿主机网络，不能做到双向通讯，老师能指点一下吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 为什么是 或者？宿主机和容器网络互通是基本假设。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-01 15:08:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ch_ort</span>
  </div>
  <div class="_2_QraFYR_0">控制器模式： 获取对象的期望状态和当前状态，对比这两个状态，如果不一致就做相应的动作来让当前状态和期望状态一致。<br><br>Deployment通过控制器模式控制ReplicaSet来实现“水平扩展&#47;收缩”、“滚动更新”，“版本控制”。ReplicaSet通过控制器模式来维持Pod的副本数量。<br><br>滚动更新： 你的游戏角色装备了一套一级铭文，现在有一套二级铭文可以替换。一个个替换，一次替换一个铭文，这就是滚动更新。<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-26 23:12:06</div>
  </div>
</div>
</div>
</li>
</ul>