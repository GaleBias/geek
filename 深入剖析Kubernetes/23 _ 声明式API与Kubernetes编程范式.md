<audio title="23 _ 声明式API与Kubernetes编程范式" src="https://static001.geekbang.org/resource/audio/0b/d9/0b697c5bde2f28826ddfc0ebf386b2d9.mp3" controls="controls"></audio> 
<p>你好，我是张磊。今天我和你分享的主题是：声明式API与Kubernetes编程范式。</p><p>在前面的几篇文章中，我和你分享了很多Kubernetes的API对象。这些API对象，有的是用来描述应用，有的则是为应用提供各种各样的服务。但是，无一例外地，为了使用这些API对象提供的能力，你都需要编写一个对应的YAML文件交给Kubernetes。</p><p>这个YAML文件，正是Kubernetes声明式API所必须具备的一个要素。不过，是不是只要用YAML文件代替了命令行操作，就是声明式API了呢？</p><p>举个例子。我们知道，Docker Swarm的编排操作都是基于命令行的，比如：</p><pre><code>$ docker service create --name nginx --replicas 2  nginx
$ docker service update --image nginx:1.7.9 nginx
</code></pre><p>像这样的两条命令，就是用Docker Swarm启动了两个Nginx容器实例。其中，第一条create命令创建了这两个容器，而第二条update命令则把它们“滚动更新”成了一个新的镜像。</p><p>对于这种使用方式，我们称为<strong>命令式命令行操作</strong>。</p><p>那么，像上面这样的创建和更新两个Nginx容器的操作，在Kubernetes里又该怎么做呢？</p><p>这个流程，相信你已经非常熟悉了：我们需要在本地编写一个Deployment的YAML文件：</p><pre><code>apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
</code></pre><!-- [[[read_end]]] --><p>然后，我们还需要使用kubectl create命令在Kubernetes里创建这个Deployment对象：</p><pre><code>$ kubectl create -f nginx.yaml
</code></pre><p>这样，两个Nginx的Pod就会运行起来了。</p><p>而如果要更新这两个Pod使用的Nginx镜像，该怎么办呢？</p><p>我们前面曾经使用过kubectl set image和kubectl edit命令，来直接修改Kubernetes里的API对象。不过，相信很多人都有这样的想法，我能不能通过修改本地YAML文件来完成这个操作呢？这样我的改动就会体现在这个本地YAML文件里了。</p><p>当然可以。</p><p>比如，我们可以修改这个YAML文件里的Pod模板部分，把Nginx容器的镜像改成1.7.9，如下所示：</p><pre><code>...
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
</code></pre><p>而接下来，我们就可以执行一句kubectl replace操作，来完成这个Deployment的更新：</p><pre><code>$ kubectl replace -f nginx.yaml
</code></pre><p>可是，上面这种基于YAML文件的操作方式，是“声明式API”吗？</p><p>并不是。</p><p>对于上面这种先kubectl create，再replace的操作，我们称为<strong>命令式配置文件操作。</strong></p><p>也就是说，它的处理方式，其实跟前面Docker Swarm的两句命令，没什么本质上的区别。只不过，它是把Docker命令行里的参数，写在了配置文件里而已。</p><p><strong>那么，到底什么才是“声明式API”呢？</strong></p><p>答案是，kubectl apply命令。</p><p>在前面的文章中，我曾经提到过这个kubectl apply命令，并推荐你使用它来代替kubectl create命令（你也可以借此机会再回顾一下第12篇文章<a href="https://time.geekbang.org/column/article/40008">《牛刀小试：我的第一个容器化应用》</a>中的相关内容）。</p><p>现在，我就使用kubectl apply命令来创建这个Deployment：</p><pre><code>$ kubectl apply -f nginx.yaml
</code></pre><p>这样，Nginx的Deployment就被创建了出来，这看起来跟kubectl create的效果一样。</p><p>然后，我再修改一下nginx.yaml里定义的镜像：</p><pre><code>...
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
</code></pre><p>这时候，关键来了。</p><p>在修改完这个YAML文件之后，我不再使用kubectl replace命令进行更新，而是继续执行一条kubectl apply命令，即：</p><pre><code>$ kubectl apply -f nginx.yaml
</code></pre><p>这时，Kubernetes就会立即触发这个Deployment的“滚动更新”。</p><p>可是，它跟kubectl replace命令有什么本质区别吗？</p><p>实际上，你可以简单地理解为，kubectl replace的执行过程，是使用新的YAML文件中的API对象，<strong>替换原有的API对象</strong>；而kubectl apply，则是执行了一个<strong>对原有API对象的PATCH操作</strong>。</p><blockquote>
<p>类似地，kubectl set image和kubectl edit也是对已有API对象的修改。</p>
</blockquote><p>更进一步地，这意味着kube-apiserver在响应命令式请求（比如，kubectl replace）的时候，一次只能处理一个写请求，否则会有产生冲突的可能。而对于声明式请求（比如，kubectl apply），<strong>一次能处理多个写操作，并且具备Merge能力</strong>。</p><p>这种区别，可能乍一听起来没那么重要。而且，正是由于要照顾到这样的API设计，做同样一件事情，Kubernetes需要的步骤往往要比其他项目多不少。</p><p>但是，如果你仔细思考一下Kubernetes项目的工作流程，就不难体会到这种声明式API的独到之处。</p><p>接下来，我就以Istio项目为例，来为你讲解一下声明式API在实际使用时的重要意义。</p><p>在2017年5月，Google、IBM和Lyft公司，共同宣布了Istio开源项目的诞生。很快，这个项目就在技术圈儿里，掀起了一阵名叫“微服务”的热潮，把Service Mesh这个新的编排概念推到了风口浪尖。</p><p>而Istio项目，实际上就是一个基于Kubernetes项目的微服务治理框架。它的架构非常清晰，如下所示：</p><p><img src="https://static001.geekbang.org/resource/image/d3/1b/d38daed2fedc90e20e9d2f27afbaec1b.jpg?wh=1920*1080" alt=""><br>
在上面这个架构图中，我们不难看到Istio项目架构的核心所在。<strong>Istio最根本的组件，是运行在每一个应用Pod里的Envoy容器</strong>。</p><p>这个Envoy项目是Lyft公司推出的一个高性能C++网络代理，也是Lyft公司对Istio项目的唯一贡献。</p><p>而Istio项目，则把这个代理服务以sidecar容器的方式，运行在了每一个被治理的应用Pod中。我们知道，Pod里的所有容器都共享同一个Network Namespace。所以，Envoy容器就能够通过配置Pod里的iptables规则，把整个Pod的进出流量接管下来。</p><p>这时候，Istio的控制层（Control Plane）里的Pilot组件，就能够通过调用每个Envoy容器的API，对这个Envoy代理进行配置，从而实现微服务治理。</p><p>我们一起来看一个例子。</p><p>假设这个Istio架构图左边的Pod是已经在运行的应用，而右边的Pod则是我们刚刚上线的应用的新版本。这时候，Pilot通过调节这两Pod里的Envoy容器的配置，从而将90%的流量分配给旧版本的应用，将10%的流量分配给新版本应用，并且，还可以在后续的过程中随时调整。这样，一个典型的“灰度发布”的场景就完成了。比如，Istio可以调节这个流量从90%-10%，改到80%-20%，再到50%-50%，最后到0%-100%，就完成了这个灰度发布的过程。</p><p>更重要的是，在整个微服务治理的过程中，无论是对Envoy容器的部署，还是像上面这样对Envoy代理的配置，用户和应用都是完全“无感”的。</p><p>这时候，你可能会有所疑惑：Istio项目明明需要在每个Pod里安装一个Envoy容器，又怎么能做到“无感”的呢？</p><p>实际上，<strong>Istio项目使用的，是Kubernetes中的一个非常重要的功能，叫作Dynamic Admission Control。</strong></p><p>在Kubernetes项目中，当一个Pod或者任何一个API对象被提交给APIServer之后，总有一些“初始化”性质的工作需要在它们被Kubernetes项目正式处理之前进行。比如，自动为所有Pod加上某些标签（Labels）。</p><p>而这个“初始化”操作的实现，借助的是一个叫作Admission的功能。它其实是Kubernetes项目里一组被称为Admission Controller的代码，可以选择性地被编译进APIServer中，在API对象创建之后会被立刻调用到。</p><p>但这就意味着，如果你现在想要添加一些自己的规则到Admission Controller，就会比较困难。因为，这要求重新编译并重启APIServer。显然，这种使用方法对Istio来说，影响太大了。</p><p>所以，Kubernetes项目为我们额外提供了一种“热插拔”式的Admission机制，它就是Dynamic Admission Control，也叫作：Initializer。</p><p>现在，我给你举个例子。比如，我有如下所示的一个应用Pod：</p><pre><code>apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox
    command: ['sh', '-c', 'echo Hello Kubernetes! &amp;&amp; sleep 3600']
</code></pre><p>可以看到，这个Pod里面只有一个用户容器，叫作：myapp-container。</p><p>接下来，Istio项目要做的，就是在这个Pod YAML被提交给Kubernetes之后，在它对应的API对象里自动加上Envoy容器的配置，使这个对象变成如下所示的样子：</p><pre><code>apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox
    command: ['sh', '-c', 'echo Hello Kubernetes! &amp;&amp; sleep 3600']
  - name: envoy
    image: lyft/envoy:845747b88f102c0fd262ab234308e9e22f693a1
    command: [&quot;/usr/local/bin/envoy&quot;]
    ...
</code></pre><p>可以看到，被Istio处理后的这个Pod里，除了用户自己定义的myapp-container容器之外，多出了一个叫作envoy的容器，它就是Istio要使用的Envoy代理。</p><p>那么，Istio又是如何在用户完全不知情的前提下完成这个操作的呢？</p><p>Istio要做的，就是编写一个用来为Pod“自动注入”Envoy容器的Initializer。</p><p><strong>首先，Istio会将这个Envoy容器本身的定义，以ConfigMap的方式保存在Kubernetes当中</strong>。这个ConfigMap（名叫：envoy-initializer）的定义如下所示：</p><pre><code>apiVersion: v1
kind: ConfigMap
metadata:
  name: envoy-initializer
data:
  config: |
    containers:
      - name: envoy
        image: lyft/envoy:845747db88f102c0fd262ab234308e9e22f693a1
        command: [&quot;/usr/local/bin/envoy&quot;]
        args:
          - &quot;--concurrency 4&quot;
          - &quot;--config-path /etc/envoy/envoy.json&quot;
          - &quot;--mode serve&quot;
        ports:
          - containerPort: 80
            protocol: TCP
        resources:
          limits:
            cpu: &quot;1000m&quot;
            memory: &quot;512Mi&quot;
          requests:
            cpu: &quot;100m&quot;
            memory: &quot;64Mi&quot;
        volumeMounts:
          - name: envoy-conf
            mountPath: /etc/envoy
    volumes:
      - name: envoy-conf
        configMap:
          name: envoy
</code></pre><p>相信你已经注意到了，这个ConfigMap的data部分，正是一个Pod对象的一部分定义。其中，我们可以看到Envoy容器对应的containers字段，以及一个用来声明Envoy配置文件的volumes字段。</p><p>不难想到，Initializer要做的工作，就是把这部分Envoy相关的字段，自动添加到用户提交的Pod的API对象里。可是，用户提交的Pod里本来就有containers字段和volumes字段，所以Kubernetes在处理这样的更新请求时，就必须使用类似于git merge这样的操作，才能将这两部分内容合并在一起。</p><p>所以说，在Initializer更新用户的Pod对象的时候，必须使用PATCH API来完成。而这种PATCH API，正是声明式API最主要的能力。</p><p><strong>接下来，Istio将一个编写好的Initializer，作为一个Pod部署在Kubernetes中</strong>。这个Pod的定义非常简单，如下所示：</p><pre><code>apiVersion: v1
kind: Pod
metadata:
  labels:
    app: envoy-initializer
  name: envoy-initializer
spec:
  containers:
    - name: envoy-initializer
      image: envoy-initializer:0.0.1
      imagePullPolicy: Always
</code></pre><p>我们可以看到，这个envoy-initializer使用的envoy-initializer:0.0.1镜像，就是一个事先编写好的“自定义控制器”（Custom Controller），我将会在下一篇文章中讲解它的编写方法。而在这里，我要先为你解释一下这个控制器的主要功能。</p><p>我曾在第16篇文章<a href="https://time.geekbang.org/column/article/40583">《编排其实很简单：谈谈“控制器”模型》</a>中和你分享过，一个Kubernetes的控制器，实际上就是一个“死循环”：它不断地获取“实际状态”，然后与“期望状态”作对比，并以此为依据决定下一步的操作。</p><p>而Initializer的控制器，不断获取到的“实际状态”，就是用户新创建的Pod。而它的“期望状态”，则是：这个Pod里被添加了Envoy容器的定义。</p><p>我还是用一段Go语言风格的伪代码，来为你描述这个控制逻辑，如下所示：</p><pre><code>for {
  // 获取新创建的Pod
  pod := client.GetLatestPod()
  // Diff一下，检查是否已经初始化过
  if !isInitialized(pod) {
    // 没有？那就来初始化一下
    doSomething(pod)
  }
}
</code></pre><ul>
<li>如果这个Pod里面已经添加过Envoy容器，那么就“放过”这个Pod，进入下一个检查周期。</li>
<li>而如果还没有添加过Envoy容器的话，它就要进行Initialize操作了，即：修改该Pod的API对象（doSomething函数）。</li>
</ul><p>这时候，你应该立刻能想到，Istio要往这个Pod里合并的字段，正是我们之前保存在envoy-initializer这个ConfigMap里的数据（即：它的data字段的值）。</p><p>所以，在Initializer控制器的工作逻辑里，它首先会从APIServer中拿到这个ConfigMap：</p><pre><code>func doSomething(pod) {
  cm := client.Get(ConfigMap, &quot;envoy-initializer&quot;)
}
</code></pre><p>然后，把这个ConfigMap里存储的containers和volumes字段，直接添加进一个空的Pod对象里：</p><pre><code>func doSomething(pod) {
  cm := client.Get(ConfigMap, &quot;envoy-initializer&quot;)
  
  newPod := Pod{}
  newPod.Spec.Containers = cm.Containers
  newPod.Spec.Volumes = cm.Volumes
}
</code></pre><p>现在，关键来了。</p><p>Kubernetes的API库，为我们提供了一个方法，使得我们可以直接使用新旧两个Pod对象，生成一个TwoWayMergePatch：</p><pre><code>func doSomething(pod) {
  cm := client.Get(ConfigMap, &quot;envoy-initializer&quot;)

  newPod := Pod{}
  newPod.Spec.Containers = cm.Containers
  newPod.Spec.Volumes = cm.Volumes

  // 生成patch数据
  patchBytes := strategicpatch.CreateTwoWayMergePatch(pod, newPod)

  // 发起PATCH请求，修改这个pod对象
  client.Patch(pod.Name, patchBytes)
}
</code></pre><p><strong>有了这个TwoWayMergePatch之后，Initializer的代码就可以使用这个patch的数据，调用Kubernetes的Client，发起一个PATCH请求</strong>。</p><p>这样，一个用户提交的Pod对象里，就会被自动加上Envoy容器相关的字段。</p><p>当然，Kubernetes还允许你通过配置，来指定要对什么样的资源进行这个Initialize操作，比如下面这个例子：</p><pre><code>apiVersion: admissionregistration.k8s.io/v1alpha1
kind: InitializerConfiguration
metadata:
  name: envoy-config
initializers:
  // 这个名字必须至少包括两个 &quot;.&quot;
  - name: envoy.initializer.kubernetes.io
    rules:
      - apiGroups:
          - &quot;&quot; // 前面说过， &quot;&quot;就是core API Group的意思
        apiVersions:
          - v1
        resources:
          - pods
</code></pre><p>这个配置，就意味着Kubernetes要对所有的Pod进行这个Initialize操作，并且，我们指定了负责这个操作的Initializer，名叫：envoy-initializer。</p><p>而一旦这个InitializerConfiguration被创建，Kubernetes就会把这个Initializer的名字，加在所有新创建的Pod的Metadata上，格式如下所示：</p><pre><code>apiVersion: v1
kind: Pod
metadata:
  initializers:
    pending:
      - name: envoy.initializer.kubernetes.io
  name: myapp-pod
  labels:
    app: myapp
...
</code></pre><p>可以看到，每一个新创建的Pod，都会自动携带了metadata.initializers.pending的Metadata信息。</p><p>这个Metadata，正是接下来Initializer的控制器判断这个Pod有没有执行过自己所负责的初始化操作的重要依据（也就是前面伪代码中isInitialized()方法的含义）。</p><p><strong>这也就意味着，当你在Initializer里完成了要做的操作后，一定要记得将这个metadata.initializers.pending标志清除掉。这一点，你在编写Initializer代码的时候一定要非常注意。</strong></p><p>此外，除了上面的配置方法，你还可以在具体的Pod的Annotation里添加一个如下所示的字段，从而声明要使用某个Initializer：</p><pre><code>apiVersion: v1
kind: Pod
metadata
  annotations:
    &quot;initializer.kubernetes.io/envoy&quot;: &quot;true&quot;
    ...
</code></pre><p>在这个Pod里，我们添加了一个Annotation，写明： <code>initializer.kubernetes.io/envoy=true</code>。这样，就会使用到我们前面所定义的envoy-initializer了。</p><p>以上，就是关于Initializer最基本的工作原理和使用方法了。相信你此时已经明白，<strong>Istio项目的核心，就是由无数个运行在应用Pod中的Envoy容器组成的服务代理网格</strong>。这也正是Service Mesh的含义。</p><blockquote>
<p>备注：如果你对这个Demo感兴趣，可以在<a href="https://github.com/resouer/kubernetes-initializer-tutorial">这个GitHub链接</a>里找到它的所有源码和文档。这个Demo，是我fork自Kelsey Hightower的一个同名的Demo。</p>
</blockquote><p>而这个机制得以实现的原理，正是借助了Kubernetes能够对API对象进行在线更新的能力，这也正是<strong>Kubernetes“声明式API”的独特之处：</strong></p><ul>
<li>首先，所谓“声明式”，指的就是我只需要提交一个定义好的API对象来“声明”，我所期望的状态是什么样子。</li>
<li>其次，“声明式API”允许有多个API写端，以PATCH的方式对API对象进行修改，而无需关心本地原始YAML文件的内容。</li>
<li>最后，也是最重要的，有了上述两个能力，Kubernetes项目才可以基于对API对象的增、删、改、查，在完全无需外界干预的情况下，完成对“实际状态”和“期望状态”的调谐（Reconcile）过程。</li>
</ul><p>所以说，<strong>声明式API，才是Kubernetes项目编排能力“赖以生存”的核心所在</strong>，希望你能够认真理解。</p><p>此外，不难看到，无论是对sidecar容器的巧妙设计，还是对Initializer的合理利用，Istio项目的设计与实现，其实都依托于Kubernetes的声明式API和它所提供的各种编排能力。可以说，Istio是在Kubernetes项目使用上的一位“集大成者”。</p><blockquote>
<p>要知道，一个Istio项目部署完成后，会在Kubernetes里创建大约43个API对象。</p>
</blockquote><p>所以，Kubernetes社区也看得很明白：Istio项目有多火热，就说明Kubernetes这套“声明式API”有多成功。这，既是Google Cloud喜闻乐见的事情，也是Istio项目一推出就被Google公司和整个技术圈儿热捧的重要原因。</p><p>而在使用Initializer的流程中，最核心的步骤，莫过于Initializer“自定义控制器”的编写过程。它遵循的，正是标准的“Kubernetes编程范式”，即：</p><blockquote>
<p><strong>如何使用控制器模式，同Kubernetes里API对象的“增、删、改、查”进行协作，进而完成用户业务逻辑的编写过程。</strong></p>
</blockquote><p>这，也正是我要在后面文章中为你详细讲解的内容。</p><h2>总结</h2><p>在今天这篇文章中，我为你重点讲解了Kubernetes声明式API的含义。并且，通过对Istio项目的剖析，我为你说明了它使用Kubernetes的Initializer特性，完成Envoy容器“自动注入”的原理。</p><p>事实上，从“使用Kubernetes部署代码”，到“使用Kubernetes编写代码”的蜕变过程，正是你从一个Kubernetes用户，到Kubernetes玩家的晋级之路。</p><p>而，如何理解“Kubernetes编程范式”，如何为Kubernetes添加自定义API对象，编写自定义控制器，正是这个晋级过程中的关键点，也是我要在后面几篇文章中分享的核心内容。</p><p>此外，基于今天这篇文章所讲述的Istio的工作原理，尽管Istio项目一直宣称它可以运行在非Kubernetes环境中，但我并不建议你花太多时间去做这个尝试。</p><p>毕竟，无论是从技术实现还是在社区运作上，Istio与Kubernetes项目之间都是紧密的、唇齿相依的关系。如果脱离了Kubernetes项目这个基础，那么这条原本就不算平坦的“微服务”之路，恐怕会更加困难重重。</p><h2>思考题</h2><p>你是否对Envoy项目做过了解？你觉得为什么它能够击败Nginx以及HAProxy等竞品，成为Service Mesh体系的核心？</p><p>感谢你的收听，欢迎你给我留言，也欢迎分享给更多的朋友一起阅读。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/54/21/8c13a2b4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>周龙亭</span>
  </div>
  <div class="_2_QraFYR_0">是因为envoy提供了api形式的配置入口，更方便做流量治理</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-26 20:45:27</div>
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
  <div class="_2_QraFYR_0">老师，用声明式api的好处没有体会太深刻。<br>如果在dosomething中merge出新的yaml，然后用replace会有什么缺点？<br>好像在这篇文章中仅仅提到声明式的可以多个客户端同时写。除此之外，还有其他优点吗？<br>也就是说修改对象比替换对象的优势在哪？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: istio不就是例子？系统里完全可以有好几个initializer在改同一个pod，你直接replace了别人还玩不玩了？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-15 18:46:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/40/ba/2c8af305.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_zz</span>
  </div>
  <div class="_2_QraFYR_0">居然看一遍就记住了这节课的原理</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-15 07:15:58</div>
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
  <div class="_2_QraFYR_0">通俗地说：如果要把一个东西存入冰箱。命令式API需要做的是：打开冰箱 -&gt; 放入东西 -&gt; 关掉冰箱。而声明式就是： 这个东西在冰箱里。<br><br><br><br>命令式请求与声明式请求：<br>（1）服务对于命令式请求，一次只能处理一个写请求，否则可能会导致冲突<br>（2）服务对于声明式请求，一次能处理多个请求，并且具备Merge的能力<br><br>kubectl replace命令与kubectl apply命令的本质区别在于，kubectl replace的执行过程，是使用新的YAML文件中的API对象，替换原有的API对象；而kubectl apply，则是执行了一个对原有API对象的PATCH（部分更新）操作<br><br>声明式API特点：<br>（1）我们只需要提交一个定义好的API对象来“声明”，我所期望的状态是什么样子<br>（2）声明式API允许由多个API写端，以PATCH的方式对API对象进行修改，而无需关心本地原始YAML文件的内容<br>（3）有了上述两个能力，Kubernetes项目才可以给予对API对象的增、删、改、查，在完全无需外界干预的情况下，完成对“实际状态”和“期望状态”的调谐（Reconcile）过程<br><br><br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-04 17:22:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4c/49/d21c134f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>swordholder</span>
  </div>
  <div class="_2_QraFYR_0">有个疑问，在envoy-initializer的“控制循环”中获取新创建的Pod，这个Pod是否已经在正常运行了？<br>Initializer 提交patch修改Pod对象，Kubernetes发现Pod更新，然后以“滚动升级”的方式更新运行中的Pod？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不会啊。注意看apiserver的流程图，initializer发生在admission阶段，这个阶段完成后pod才会创建出来。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-15 18:54:53</div>
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
  <div class="_2_QraFYR_0">kubectl apply 是通过mvcc 实现的并发写吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是啊</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-16 19:07:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/a5/08/10b18682.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>混沌渺无极</span>
  </div>
  <div class="_2_QraFYR_0">dynamic admission control有点像防火墙的DNAT，数据包即将进入路由表的瞬间被修改了目的地址，这样路由表就对数据包的修改[无感]。<br>patch就像多人使用git来进行文件的&quot;合并型&quot;修改。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 就是这么回事儿</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-16 01:48:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/72/3e/534db55d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>huan</span>
  </div>
  <div class="_2_QraFYR_0">又查了下envoy的设计，感觉它支持热更新和热重启，应该很适合声明式规则的开发范式，这可以看做一种优势，相比而言，nginx的reload需要把worker进程退出，比较面向命令</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这确实是一个因素</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-15 23:22:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/2d/24/28acca15.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>DJH</span>
  </div>
  <div class="_2_QraFYR_0">请教老师，Initializer和Preset都能注入POD配置，那么这两种方法的适用场景有何不同？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: preset相当于initializer的子集，比较适合在发布流程里处理比较简单的情况。initializer是要写代码的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-15 07:44:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1c/cd/56/dca89081.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lucasun</span>
  </div>
  <div class="_2_QraFYR_0">Initializer不是一直bata然后废弃了嘛，istio用的是MutatingAdmissionWebhook吧</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Istio 现在是admission hook。这部分功能变化太多了，最好是自己写个sidecar operator 来管理。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-03 20:27:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/98/9e/b9069b65.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Lis</span>
  </div>
  <div class="_2_QraFYR_0">老师好，课后作业的方式非常棒，可否在下一节课的开始先总结一下课后作业呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-09 10:21:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/03/5e/818a8b1b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Alex</span>
  </div>
  <div class="_2_QraFYR_0">Initializer与新的pod 在git merge冲突了该怎么解决？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 就不会注入成功了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-14 16:18:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/3e/d2/5f9d3fa7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>羽翼1982</span>
  </div>
  <div class="_2_QraFYR_0">所以这个问题的答案是什么呢？<br>我的理解是Envy性能更高，占用系统资源更少</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 编程友好的api，方便容器化，配置方便</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-27 17:54:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/75/aa/21275b9d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>闫飞</span>
  </div>
  <div class="_2_QraFYR_0">服务网格最初是由linkerd项目提出概念的，lstio是另外一个后起之秀，使大家都关注到了边车代理模式和服务治理的新方法的巨大威力。文中应该笔误写错为微服务了。<br><br>不过瑕不掩瑜，本节写的极其精彩和深入浅出。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: service mesh is just a fancy way of microservice</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-16 08:14:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/22/42/79604ce6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>公众号：云原生Serverless</span>
  </div>
  <div class="_2_QraFYR_0">磊哥竟然穿插了istio的讲解，后续有没有计划讲讲knative呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: knative没啥特别的，暂时就不给篇幅了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-15 12:50:59</div>
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
  <div class="_2_QraFYR_0">老师，为什么修改对象可以多个客户端同时写，而替换不行？感觉还差一层窗户纸，老师帮我捅破:)<br>或者有什么资料可以让我更深入理解下吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你的错误之处在于，patch数据只可能被PATCH API认识。你总想着让replace也能用patch数据，那replace不就成了patch api了？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-15 18:47:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/98/02/14e24394.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Zhikun Lao</span>
  </div>
  <div class="_2_QraFYR_0">老师你好！<br>”如果你对这个 Demo 感兴趣，可以在这个 GitHub 链接里找到它的所有源码和文档。这个 Demo，是我 fork 自 Kelsey Hightower 的一个同名的 Demo“<br><br>我看这个initializer的plugin都已经没了，现在是不是都要写Admission hook了？ https:&#47;&#47;kubernetes.io&#47;docs&#47;reference&#47;access-authn-authz&#47;extensible-admission-controllers&#47;#<br><br>例如这个 https:&#47;&#47;github.com&#47;kubernetes&#47;kubernetes&#47;blob&#47;v1.13.0&#47;test&#47;images&#47;webhook&#47;main.go</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-12 10:28:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/80/be/8350f94d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>gotojeff</span>
  </div>
  <div class="_2_QraFYR_0">Hi 老师<br>‘’‘有个疑问： name 为envoy的configmap是在哪里定义的呢？<br>2018-10-16<br> 作者回复<br>文中不是贴出来了？<br>’‘’<br>configmap模板中的 metadata - name是envoy-initializer，但是在下面的container volumes中的configmap name是envoy，我的疑惑是这是2个不同的configmap吧？后者是在其他地方定义的？<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 哦哦，这个忘记贴了。没错，这个configmap需要使用envoy的配置文件创建出来，配置文件网上可以下到。kubectl create configmap envoy —from-file envoy.json</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-17 04:17:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/18/e7/d58e287c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>参悟</span>
  </div>
  <div class="_2_QraFYR_0">我想envoy的成功，是因为它真正理解了k8s的技术精髓，并成功的应用到了当前最火的微服务领域，将微服务体系与K8S捆绑在一起，service mesh成为微服务新一代技术的代言，这无论从技术上还是战略上都赢得了google的芳心，成功也就水到渠成。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 但其实envoy发布的不算晚，称不上搭便车。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-15 22:14:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/1b/b4/a6db1c1e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>silver</span>
  </div>
  <div class="_2_QraFYR_0">kubectl apply怎么做concurrency control呢？假设client A和B都有version 1的spec。然后他们在各自修改了spec之后call apply。假设client A的patch操作先成功，如果kubectl简单的与etcd里有的spec做一个diff，会不会出现一个client B把client A的更新个revert的情况？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: API对象都有revision，所以apiserver处理merge的流程跟git server是一样的。这跟kubectl这关系，它只是个客户端。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-16 13:36:04</div>
  </div>
</div>
</div>
</li>
</ul>