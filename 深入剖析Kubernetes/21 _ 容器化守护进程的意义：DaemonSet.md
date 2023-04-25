<audio title="21 _ 容器化守护进程的意义：DaemonSet" src="https://static001.geekbang.org/resource/audio/59/18/593cf97eb297671d472238e41d05f218.mp3" controls="controls"></audio> 
<p>你好，我是张磊。今天我和你分享的主题是：容器化守护进程的意义之DaemonSet。</p><p>在上一篇文章中，我和你详细分享了使用StatefulSet编排“有状态应用”的过程。从中不难看出，StatefulSet其实就是对现有典型运维业务的容器化抽象。也就是说，你一定有方法在不使用Kubernetes、甚至不使用容器的情况下，自己DIY一个类似的方案出来。但是，一旦涉及到升级、版本管理等更工程化的能力，Kubernetes的好处，才会更加凸现。</p><p>比如，如何对StatefulSet进行“滚动更新”（rolling update）？</p><p>很简单。你只要修改StatefulSet的Pod模板，就会自动触发“滚动更新”:</p><pre><code>$ kubectl patch statefulset mysql --type='json' -p='[{&quot;op&quot;: &quot;replace&quot;, &quot;path&quot;: &quot;/spec/template/spec/containers/0/image&quot;, &quot;value&quot;:&quot;mysql:5.7.23&quot;}]'
statefulset.apps/mysql patched
</code></pre><p>在这里，我使用了kubectl patch命令。它的意思是，以“补丁”的方式（JSON格式的）修改一个API对象的指定字段，也就是我在后面指定的“spec/template/spec/containers/0/image”。</p><p>这样，StatefulSet Controller就会按照与Pod编号相反的顺序，从最后一个Pod开始，逐一更新这个StatefulSet管理的每个Pod。而如果更新发生了错误，这次“滚动更新”就会停止。此外，StatefulSet的“滚动更新”还允许我们进行更精细的控制，比如金丝雀发布（Canary Deploy）或者灰度发布，<strong>这意味着应用的多个实例中被指定的一部分不会被更新到最新的版本。</strong></p><!-- [[[read_end]]] --><p>这个字段，正是StatefulSet的spec.updateStrategy.rollingUpdate的partition字段。</p><p>比如，现在我将前面这个StatefulSet的partition字段设置为2：</p><pre><code>$ kubectl patch statefulset mysql -p '{&quot;spec&quot;:{&quot;updateStrategy&quot;:{&quot;type&quot;:&quot;RollingUpdate&quot;,&quot;rollingUpdate&quot;:{&quot;partition&quot;:2}}}}'
statefulset.apps/mysql patched
</code></pre><p>其中，kubectl patch命令后面的参数（JSON格式的），就是partition字段在API对象里的路径。所以，上述操作等同于直接使用 kubectl edit命令，打开这个对象，把partition字段修改为2。</p><p>这样，我就指定了当Pod模板发生变化的时候，比如MySQL镜像更新到5.7.23，那么只有序号大于或者等于2的Pod会被更新到这个版本。并且，如果你删除或者重启了序号小于2的Pod，等它再次启动后，也会保持原先的5.7.2版本，绝不会被升级到5.7.23版本。</p><p>StatefulSet可以说是Kubernetes项目中最为复杂的编排对象，希望你课后能认真消化，动手实践一下这个例子。</p><p>而在今天这篇文章中，我会为你重点讲解一个相对轻松的知识点：DaemonSet。</p><p>顾名思义，DaemonSet的主要作用，是让你在Kubernetes集群里，运行一个Daemon Pod。 所以，这个Pod有如下三个特征：</p><ol>
<li>
<p>这个Pod运行在Kubernetes集群里的每一个节点（Node）上；</p>
</li>
<li>
<p>每个节点上只有一个这样的Pod实例；</p>
</li>
<li>
<p>当有新的节点加入Kubernetes集群后，该Pod会自动地在新节点上被创建出来；而当旧节点被删除后，它上面的Pod也相应地会被回收掉。</p>
</li>
</ol><p>这个机制听起来很简单，但Daemon Pod的意义确实是非常重要的。我随便给你列举几个例子：</p><ol>
<li>
<p>各种网络插件的Agent组件，都必须运行在每一个节点上，用来处理这个节点上的容器网络；</p>
</li>
<li>
<p>各种存储插件的Agent组件，也必须运行在每一个节点上，用来在这个节点上挂载远程存储目录，操作容器的Volume目录；</p>
</li>
<li>
<p>各种监控组件和日志组件，也必须运行在每一个节点上，负责这个节点上的监控信息和日志搜集。</p>
</li>
</ol><p>更重要的是，跟其他编排对象不一样，DaemonSet开始运行的时机，很多时候比整个Kubernetes集群出现的时机都要早。</p><p>这个乍一听起来可能有点儿奇怪。但其实你来想一下：如果这个DaemonSet正是一个网络插件的Agent组件呢？</p><p>这个时候，整个Kubernetes集群里还没有可用的容器网络，所有Worker节点的状态都是NotReady（NetworkReady=false）。这种情况下，普通的Pod肯定不能运行在这个集群上。所以，这也就意味着DaemonSet的设计，必须要有某种“过人之处”才行。</p><p>为了弄清楚DaemonSet的工作原理，我们还是按照老规矩，先从它的API对象的定义说起。</p><pre><code>apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: k8s.gcr.io/fluentd-elasticsearch:1.20
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
</code></pre><p>这个DaemonSet，管理的是一个fluentd-elasticsearch镜像的Pod。这个镜像的功能非常实用：通过fluentd将Docker容器里的日志转发到ElasticSearch中。</p><p>可以看到，DaemonSet跟Deployment其实非常相似，只不过是没有replicas字段；它也使用selector选择管理所有携带了name=fluentd-elasticsearch标签的Pod。</p><p>而这些Pod的模板，也是用template字段定义的。在这个字段中，我们定义了一个使用 fluentd-elasticsearch:1.20镜像的容器，而且这个容器挂载了两个hostPath类型的Volume，分别对应宿主机的/var/log目录和/var/lib/docker/containers目录。</p><p>显然，fluentd启动之后，它会从这两个目录里搜集日志信息，并转发给ElasticSearch保存。这样，我们通过ElasticSearch就可以很方便地检索这些日志了。</p><p>需要注意的是，Docker容器里应用的日志，默认会保存在宿主机的/var/lib/docker/containers/{{.容器ID}}/{{.容器ID}}-json.log文件里，所以这个目录正是fluentd的搜集目标。</p><p>那么，<strong>DaemonSet又是如何保证每个Node上有且只有一个被管理的Pod呢？</strong></p><p>显然，这是一个典型的“控制器模型”能够处理的问题。</p><p>DaemonSet Controller，首先从Etcd里获取所有的Node列表，然后遍历所有的Node。这时，它就可以很容易地去检查，当前这个Node上是不是有一个携带了name=fluentd-elasticsearch标签的Pod在运行。</p><p>而检查的结果，可能有这么三种情况：</p><ol>
<li>
<p>没有这种Pod，那么就意味着要在这个Node上创建这样一个Pod；</p>
</li>
<li>
<p>有这种Pod，但是数量大于1，那就说明要把多余的Pod从这个Node上删除掉；</p>
</li>
<li>
<p>正好只有一个这种Pod，那说明这个节点是正常的。</p>
</li>
</ol><p>其中，删除节点（Node）上多余的Pod非常简单，直接调用Kubernetes API就可以了。</p><p>但是，<strong>如何在指定的Node上创建新Pod呢？</strong></p><p>如果你已经熟悉了Pod API对象的话，那一定可以立刻说出答案：用nodeSelector，选择Node的名字即可。</p><pre><code>nodeSelector:
    name: &lt;Node名字&gt;
</code></pre><p>没错。</p><p>不过，在Kubernetes项目里，nodeSelector其实已经是一个将要被废弃的字段了。因为，现在有了一个新的、功能更完善的字段可以代替它，即：nodeAffinity。我来举个例子：</p><pre><code>apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: metadata.name
            operator: In
            values:
            - node-geektime
</code></pre><p>在这个Pod里，我声明了一个spec.affinity字段，然后定义了一个nodeAffinity。其中，spec.affinity字段，是Pod里跟调度相关的一个字段。关于它的完整内容，我会在讲解调度策略的时候再详细阐述。</p><p>而在这里，我定义的nodeAffinity的含义是：</p><ol>
<li>
<p>requiredDuringSchedulingIgnoredDuringExecution：它的意思是说，这个nodeAffinity必须在每次调度的时候予以考虑。同时，这也意味着你可以设置在某些情况下不考虑这个nodeAffinity；</p>
</li>
<li>
<p>这个Pod，将来只允许运行在“<code>metadata.name</code>”是“node-geektime”的节点上。</p>
</li>
</ol><p>在这里，你应该注意到nodeAffinity的定义，可以支持更加丰富的语法，比如operator: In（即：部分匹配；如果你定义operator: Equal，就是完全匹配），这也正是nodeAffinity会取代nodeSelector的原因之一。</p><blockquote>
<p>备注：其实在大多数时候，这些Operator语义没啥用处。所以说，在学习开源项目的时候，一定要学会抓住“主线”。不要顾此失彼。</p>
</blockquote><p>所以，<strong>我们的DaemonSet Controller会在创建Pod的时候，自动在这个Pod的API对象里，加上这样一个nodeAffinity定义</strong>。其中，需要绑定的节点名字，正是当前正在遍历的这个Node。</p><p>当然，DaemonSet并不需要修改用户提交的YAML文件里的Pod模板，而是在向Kubernetes发起请求之前，直接修改根据模板生成的Pod对象。这个思路，也正是我在前面讲解Pod对象时介绍过的。</p><p>此外，DaemonSet还会给这个Pod自动加上另外一个与调度相关的字段，叫作tolerations。这个字段意味着这个Pod，会“容忍”（Toleration）某些Node的“污点”（Taint）。</p><p>而DaemonSet自动加上的tolerations字段，格式如下所示：</p><pre><code>apiVersion: v1
kind: Pod
metadata:
  name: with-toleration
spec:
  tolerations:
  - key: node.kubernetes.io/unschedulable
    operator: Exists
    effect: NoSchedule
</code></pre><p>这个Toleration的含义是：“容忍”所有被标记为unschedulable“污点”的Node；“容忍”的效果是允许调度。</p><blockquote>
<p>备注：关于如何给一个Node标记上“污点”，以及这里具体的语法定义，我会在后面介绍调度器的时候做详细介绍。这里，你可以简单地把“污点”理解为一种特殊的Label。</p>
</blockquote><p>而在正常情况下，被标记了unschedulable“污点”的Node，是不会有任何Pod被调度上去的（effect: NoSchedule）。可是，DaemonSet自动地给被管理的Pod加上了这个特殊的Toleration，就使得这些Pod可以忽略这个限制，继而保证每个节点上都会被调度一个Pod。当然，如果这个节点有故障的话，这个Pod可能会启动失败，而DaemonSet则会始终尝试下去，直到Pod启动成功。</p><p>这时，你应该可以猜到，我在前面介绍到的<strong>DaemonSet的“过人之处”，其实就是依靠Toleration实现的。</strong></p><p>假如当前DaemonSet管理的，是一个网络插件的Agent Pod，那么你就必须在这个DaemonSet的YAML文件里，给它的Pod模板加上一个能够“容忍”<code>node.kubernetes.io/network-unavailable</code>“污点”的Toleration。正如下面这个例子所示：</p><pre><code>...
template:
    metadata:
      labels:
        name: network-plugin-agent
    spec:
      tolerations:
      - key: node.kubernetes.io/network-unavailable
        operator: Exists
        effect: NoSchedule
</code></pre><p>在Kubernetes项目中，当一个节点的网络插件尚未安装时，这个节点就会被自动加上名为<code>node.kubernetes.io/network-unavailable</code>的“污点”。</p><p><strong>而通过这样一个Toleration，调度器在调度这个Pod的时候，就会忽略当前节点上的“污点”，从而成功地将网络插件的Agent组件调度到这台机器上启动起来。</strong></p><p>这种机制，正是我们在部署Kubernetes集群的时候，能够先部署Kubernetes本身、再部署网络插件的根本原因：因为当时我们所创建的Weave的YAML，实际上就是一个DaemonSet。</p><blockquote>
<p>这里，你也可以再回顾一下第11篇文章<a href="https://time.geekbang.org/column/article/39724">《从0到1：搭建一个完整的Kubernetes集群》</a>中的相关内容。</p>
</blockquote><p>至此，通过上面这些内容，你应该能够明白，<strong>DaemonSet其实是一个非常简单的控制器</strong>。在它的控制循环中，只需要遍历所有节点，然后根据节点上是否有被管理Pod的情况，来决定是否要创建或者删除一个Pod。</p><p>只不过，在创建每个Pod的时候，DaemonSet会自动给这个Pod加上一个nodeAffinity，从而保证这个Pod只会在指定节点上启动。同时，它还会自动给这个Pod加上一个Toleration，从而忽略节点的unschedulable“污点”。</p><p>当然，<strong>你也可以在Pod模板里加上更多种类的Toleration，从而利用DaemonSet达到自己的目的</strong>。比如，在这个fluentd-elasticsearch DaemonSet里，我就给它加上了这样的Toleration：</p><pre><code>tolerations:
- key: node-role.kubernetes.io/master
  effect: NoSchedule
</code></pre><p>这是因为在默认情况下，Kubernetes集群不允许用户在Master节点部署Pod。因为，Master节点默认携带了一个叫作<code>node-role.kubernetes.io/master</code>的“污点”。所以，为了能在Master节点上部署DaemonSet的Pod，我就必须让这个Pod“容忍”这个“污点”。</p><p>在理解了DaemonSet的工作原理之后，接下来我就通过一个具体的实践来帮你更深入地掌握DaemonSet的使用方法。</p><blockquote>
<p>备注：需要注意的是，在Kubernetes v1.11之前，由于调度器尚不完善，DaemonSet是由DaemonSet Controller自行调度的，即它会直接设置Pod的spec.nodename字段，这样就可以跳过调度器了。但是，这样的做法很快就会被废除，所以在这里我也不推荐你再花时间学习这个流程了。</p>
</blockquote><p><strong>首先，创建这个DaemonSet对象：</strong></p><pre><code>$ kubectl create -f fluentd-elasticsearch.yaml
</code></pre><p>需要注意的是，在DaemonSet上，我们一般都应该加上resources字段，来限制它的CPU和内存使用，防止它占用过多的宿主机资源。</p><p>而创建成功后，你就能看到，如果有N个节点，就会有N个fluentd-elasticsearch Pod在运行。比如在我们的例子里，会有两个Pod，如下所示：</p><pre><code>$ kubectl get pod -n kube-system -l name=fluentd-elasticsearch
NAME                          READY     STATUS    RESTARTS   AGE
fluentd-elasticsearch-dqfv9   1/1       Running   0          53m
fluentd-elasticsearch-pf9z5   1/1       Running   0          53m
</code></pre><p>而如果你此时通过kubectl get查看一下Kubernetes集群里的DaemonSet对象：</p><pre><code>$ kubectl get ds -n kube-system fluentd-elasticsearch
NAME                    DESIRED   CURRENT   READY     UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
fluentd-elasticsearch   2         2         2         2            2           &lt;none&gt;          1h
</code></pre><blockquote>
<p>备注：Kubernetes里比较长的API对象都有短名字，比如DaemonSet对应的是ds，Deployment对应的是deploy。</p>
</blockquote><p>就会发现DaemonSet和Deployment一样，也有DESIRED、CURRENT等多个状态字段。这也就意味着，DaemonSet可以像Deployment那样，进行版本管理。这个版本，可以使用kubectl rollout history看到：</p><pre><code>$ kubectl rollout history daemonset fluentd-elasticsearch -n kube-system
daemonsets &quot;fluentd-elasticsearch&quot;
REVISION  CHANGE-CAUSE
1         &lt;none&gt;
</code></pre><p><strong>接下来，我们来把这个DaemonSet的容器镜像版本到v2.2.0：</strong></p><pre><code>$ kubectl set image ds/fluentd-elasticsearch fluentd-elasticsearch=k8s.gcr.io/fluentd-elasticsearch:v2.2.0 --record -n=kube-system
</code></pre><p>这个kubectl set image命令里，第一个fluentd-elasticsearch是DaemonSet的名字，第二个fluentd-elasticsearch是容器的名字。</p><p>这时候，我们可以使用kubectl rollout status命令看到这个“滚动更新”的过程，如下所示：</p><pre><code>$ kubectl rollout status ds/fluentd-elasticsearch -n kube-system
Waiting for daemon set &quot;fluentd-elasticsearch&quot; rollout to finish: 0 out of 2 new pods have been updated...
Waiting for daemon set &quot;fluentd-elasticsearch&quot; rollout to finish: 0 out of 2 new pods have been updated...
Waiting for daemon set &quot;fluentd-elasticsearch&quot; rollout to finish: 1 of 2 updated pods are available...
daemon set &quot;fluentd-elasticsearch&quot; successfully rolled out
</code></pre><p>注意，由于这一次我在升级命令后面加上了–record参数，所以这次升级使用到的指令就会自动出现在DaemonSet的rollout history里面，如下所示：</p><pre><code>$ kubectl rollout history daemonset fluentd-elasticsearch -n kube-system
daemonsets &quot;fluentd-elasticsearch&quot;
REVISION  CHANGE-CAUSE
1         &lt;none&gt;
2         kubectl set image ds/fluentd-elasticsearch fluentd-elasticsearch=k8s.gcr.io/fluentd-elasticsearch:v2.2.0 --namespace=kube-system --record=true
</code></pre><p>有了版本号，你也就可以像Deployment一样，将DaemonSet回滚到某个指定的历史版本了。</p><p>而我在前面的文章中讲解Deployment对象的时候，曾经提到过，Deployment管理这些版本，靠的是“一个版本对应一个ReplicaSet对象”。可是，DaemonSet控制器操作的直接就是Pod，不可能有ReplicaSet这样的对象参与其中。<strong>那么，它的这些版本又是如何维护的呢？</strong></p><p>所谓，一切皆对象！</p><p>在Kubernetes项目中，任何你觉得需要记录下来的状态，都可以被用API对象的方式实现。当然，“版本”也不例外。</p><p>Kubernetes v1.7之后添加了一个API对象，名叫<strong>ControllerRevision</strong>，专门用来记录某种Controller对象的版本。比如，你可以通过如下命令查看fluentd-elasticsearch对应的ControllerRevision：</p><pre><code>$ kubectl get controllerrevision -n kube-system -l name=fluentd-elasticsearch
NAME                               CONTROLLER                             REVISION   AGE
fluentd-elasticsearch-64dc6799c9   daemonset.apps/fluentd-elasticsearch   2          1h
</code></pre><p>而如果你使用kubectl describe查看这个ControllerRevision对象：</p><pre><code>$ kubectl describe controllerrevision fluentd-elasticsearch-64dc6799c9 -n kube-system
Name:         fluentd-elasticsearch-64dc6799c9
Namespace:    kube-system
Labels:       controller-revision-hash=2087235575
              name=fluentd-elasticsearch
Annotations:  deprecated.daemonset.template.generation=2
              kubernetes.io/change-cause=kubectl set image ds/fluentd-elasticsearch fluentd-elasticsearch=k8s.gcr.io/fluentd-elasticsearch:v2.2.0 --record=true --namespace=kube-system
API Version:  apps/v1
Data:
  Spec:
    Template:
      $ Patch:  replace
      Metadata:
        Creation Timestamp:  &lt;nil&gt;
        Labels:
          Name:  fluentd-elasticsearch
      Spec:
        Containers:
          Image:              k8s.gcr.io/fluentd-elasticsearch:v2.2.0
          Image Pull Policy:  IfNotPresent
          Name:               fluentd-elasticsearch
...
Revision:                  2
Events:                    &lt;none&gt;
</code></pre><p>就会看到，这个ControllerRevision对象，实际上是在Data字段保存了该版本对应的完整的DaemonSet的API对象。并且，在Annotation字段保存了创建这个对象所使用的kubectl命令。</p><p>接下来，我们可以尝试将这个DaemonSet回滚到Revision=1时的状态：</p><pre><code>$ kubectl rollout undo daemonset fluentd-elasticsearch --to-revision=1 -n kube-system
daemonset.extensions/fluentd-elasticsearch rolled back
</code></pre><p>这个kubectl rollout undo操作，实际上相当于读取到了Revision=1的ControllerRevision对象保存的Data字段。而这个Data字段里保存的信息，就是Revision=1时这个DaemonSet的完整API对象。</p><p>所以，现在DaemonSet Controller就可以使用这个历史API对象，对现有的DaemonSet做一次PATCH操作（等价于执行一次kubectl apply -f “旧的DaemonSet对象”），从而把这个DaemonSet“更新”到一个旧版本。</p><p>这也是为什么，在执行完这次回滚完成后，你会发现，DaemonSet的Revision并不会从Revision=2退回到1，而是会增加成Revision=3。这是因为，一个新的ControllerRevision被创建了出来。</p><h2>总结</h2><p>在今天这篇文章中，我首先简单介绍了StatefulSet的“滚动更新”，然后重点讲解了本专栏的第三个重要编排对象：DaemonSet。</p><p>相比于Deployment，DaemonSet只管理Pod对象，然后通过nodeAffinity和Toleration这两个调度器的小功能，保证了每个节点上有且只有一个Pod。这个控制器的实现原理简单易懂，希望你能够快速掌握。</p><p>与此同时，DaemonSet使用ControllerRevision，来保存和管理自己对应的“版本”。这种“面向API对象”的设计思路，大大简化了控制器本身的逻辑，也正是Kubernetes项目“声明式API”的优势所在。</p><p>而且，相信聪明的你此时已经想到了，StatefulSet也是直接控制Pod对象的，那么它是不是也在使用ControllerRevision进行版本管理呢？</p><p>没错。在Kubernetes项目里，ControllerRevision其实是一个通用的版本管理对象。这样，Kubernetes项目就巧妙地避免了每种控制器都要维护一套冗余的代码和逻辑的问题。</p><h2>思考题</h2><p>我在文中提到，在Kubernetes v1.11之前，DaemonSet所管理的Pod的调度过程，实际上都是由DaemonSet Controller自己而不是由调度器完成的。你能说出这其中有哪些原因吗？</p><p>感谢你的收听，欢迎你给我留言，也欢迎分享给更多的朋友一起阅读。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/94/47/75875257.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>虎虎❤️</span>
  </div>
  <div class="_2_QraFYR_0">思考题，<br><br>我觉得应该是效率的问题。<br><br>查了一下v1.11 的 release notes。scheduler关于affinity谓词的性能大大提高了。<br><br>查阅了Ds用默认调度器代替controller的设计文档<br>之前的做法是：<br>controller判断调度谓词，符合的话直接在controller中直接设置spec.hostName去调度。<br>目前的做法是：<br>controller不再判断调度条件，给每个pode设置NodeAffinity。控制器根据NodeAffinity去检查每个node上是否启动了相应的Pod。并且可以利用调度优先级去优先调度关键的ds pods。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，关键就在于，调度优先级这个特性出现了。所以现在的设计其实没啥特别的地方。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-10 10:58:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/96/50/bde525b1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>北卡</span>
  </div>
  <div class="_2_QraFYR_0">我跟上面的朋友有同样的疑问，关于Partition更新的。<br><br>我设置了Partition，用部分pod来做灰度发布，然后发现没问题，我要全部更新，就只需要去掉Partition字段吗？<br>然后我下一次更新的时候，就要再先加上Partition，然后再更新。全部更新时再去掉。<br><br>我看了老师的回复，表达的是这个意思吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的。准确的说，是一步一步的减小partition ，一直减成0。就发布完了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-10 22:40:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/64/af/6dbbb482.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>紫夜</span>
  </div>
  <div class="_2_QraFYR_0">张老师，DaemonSet的滚动更新，是先delete旧的pod，再启动新的pod，还是和Deployment一样，先创建新的pod，再删除旧的pod?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 到目前为止，DS只有一种策略就是先删除再创建 OnDelete</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-15 18:44:43</div>
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
  <div class="_2_QraFYR_0">张老师，请教几个基础问题：<br>1. 在上一讲中，有一点我还是没想通，为何MySQL的数据复制操作必须要用sidecar容器来处理，而不用Mysql主容器来一并解决，你当时提到说是因为容器是单进程模型。如果取消sidecar容器，把数据复制操作和启动MySQL服务这两个操作一并写到MySQL主容器的sh -c命令中，这样算不算一个进程呢？<br><br>2. StatefulSet的容器启动有先后顺序，那么当序号较小的容器由于某种原因需要重启时，会不会先把序号较大的容器KILL掉，再按照它们本来的顺序重新启动一次？<br><br>3. 在这一讲中，你提到了滚动升级时StatefulSet控制新旧副本数的spec.updateStrategy.rollingUpdate.Partition字段。假设我现在已经用这个功能已经完成了灰度发布，需要把所有POD都更新到最新版本，那么是不是Edit或者Patch这个StatefulSet，把spec.updateStrategy.rollingUpdate.Partition字段修改成总的POD数即可？<br><br>4. 在这一讲中提到ControllerRevision这个API对象，K8S会对StatefulSet或DaemonSet的每一次滚动升级都会产生一个新的ControllerRevision对象，还是每个StatefulSet或DaemonSet对象只会有一个关联的ControllerRevision对象，不同的revision记录到同一个ControllerRevision对象中？<br><br>5. Deployment里可以控制保留历史ReplicaSet的数量，那么ControllerRevision这个API对象能不能做到保留指定数量的版本记录？<br><br>问题比较多，谢谢！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 已经解释。不会，重启不会破坏拓扑规则，而且文章里说了，你要确保你的脚本不受重启影响。去掉partition。多个对象。支持，字段也一样。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-10 08:02:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/a0/f3/871e4a54.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Kanner</span>
  </div>
  <div class="_2_QraFYR_0">那为什么Deployment不用ControllerRevison管理版本呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 第一，它比较老。第二，它要用replicaset。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-10 17:55:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/13/9d/0ff43179.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Andy</span>
  </div>
  <div class="_2_QraFYR_0">k8s.gcr.io&#47;fluentd-elasticsearch镜像下不到，百度搜到有人用 mirrorgooglecontainers&#47;前头的镜像替代“科学上网”，<br>于是访问 https:&#47;&#47;hub.docker.com&#47;r&#47;镜像名&#47;tags 来查看替代镜像的所有版本。<br><br>https:&#47;&#47;hub.docker.com&#47;r&#47;mirrorgooglecontainers&#47;fluentd-elasticsearch&#47;tags<br><br>找到了以下两个版本来做实验。<br>mirrorgooglecontainers&#47;fluentd-elasticsearch:v2.0.0<br><br>mirrorgooglecontainers&#47;fluentd-elasticsearch:v2.4.0</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-26 13:12:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/7e/8b/3cc461b3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>宋晓明</span>
  </div>
  <div class="_2_QraFYR_0">老师：有没有公司这样用k8s的：apiserver管理很多pod 每个pod的ip地址全部暴露出来 nginx的upstream配置全是pod的ip地址，访问流程也就是client—&gt;nginx—&gt;pod:port  还有一个程序会监控pod地址变化，一旦变化，自动更新nginx配置。这是新公司使用k8s的流程，感觉好多k8s特性都没用到，比如service，ingress等 大材小用了。。。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 小规模用用可以，上生产环境还是要用标准的功能。你会发现越用越爽。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-10 18:13:42</div>
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
  <div class="_2_QraFYR_0">一直不理解notation和label的区别，他们的设计思想是什么呢？加污点是前者还是后者？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 系统用的和用户用的。taint是spec的一个字段</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-10 09:25:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/c1/60/fc3689d0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小谢同学</span>
  </div>
  <div class="_2_QraFYR_0">Stateful set 管理的replica 不是通过RS实现的么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不是。直接管pod</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-12 08:46:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/8e/5a/2ed0cdec.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>donson</span>
  </div>
  <div class="_2_QraFYR_0">“需要注意的是，在 Kubernetes v1.11 之前，由于调度器尚不完善，DaemonSet 是由 DaemonSet Controller 自行调度的，即它会直接设置 Pod 的 spec.nodename 字段，这样就可以跳过调度器了。”，后来随着调度器的完善，调度器就把DaemonSet的调度逻辑收回，由调度器统一调度。划清边界，领域内聚</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 调度器只是原因之一。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-10 08:51:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/91/d4/f530914a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>陈水金</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，有两个问题请教您:1.既然有ControllerVersion这样通用的版本管理对象，为什么Deployment还需要通过ReplicaSet来进行版本控制呢？2.Deployment的ownerReference又是谁呢？期待老师的解惑</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-08 12:00:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/e9/91/4219d305.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>初学者</span>
  </div>
  <div class="_2_QraFYR_0">还是没有明白damonset的实现与&quot;污点&quot;的关系，理论上为了实现每个node上有且只有pod, daemonset controller 和nodeaffinity就可以了，为啥需要&quot;污点&quot;机制？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 所以你就得在daemonset controller里写调度了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-29 23:51:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/61/b9/dbf629c0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Tao</span>
  </div>
  <div class="_2_QraFYR_0">DaemonSet如何保证每个node上只运行一个pod。<br>	我理解Nodeaffinity保证了daemonSet指定运行在哪些node上，Toleration保证了指定的Node上都可以运行pod；<br>	但是没有看到那个地方限制了DaemonSet保证在node上只允许一个pod。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-30 21:06:05</div>
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
  <div class="_2_QraFYR_0">sts 在update的过程中如果失败，并且没有办法restore，比如pulling image fail。这种情况应该怎么恢复？<br><br>我尝试再patch一个正确的image 路径，但是没有反应。 然后我delete掉了出错的pod。正常的做法应该是什么？roll out 到上一个&#47;下一个正确版本？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 目前没有自动化的手段，需要你删除错误pod，rollout正确的版本。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-10 15:23:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/b9/c8/be96e383.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>放开那坨便便</span>
  </div>
  <div class="_2_QraFYR_0">“这样，mysql 这个 StatefulSet 就会严格按照 Pod 的序号，逐一更新 MySQL 容器的镜像。而如果更新有错误，它会自动回滚到原先的版本。”<br><br>我测试的时候没有自动回滚，请问自动回滚是如何实现的？需要修改什么配置么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这里写的不准确了。不是自动回滚，而是不断使用当前版本重新创建。对于已经更新过的，当前版本就是新版本。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-10 11:40:35</div>
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
  <div class="_2_QraFYR_0">第二十一课:容器化守护进程的意义:DaemonSet<br>DaemonSet的主要作用是让你在Kubernete集群里，运行有且只有一个DaemonSet Pod，并且会随着新的Node添加或老的Node删除而添加&#47;删除Node上的Pod。它的应用场景有网络插件、存储插件和监控以及日志收集。它很多时候比Kubernetes集群出现的时机都要早。<br><br>这种添加删除的过程是，首先从Etcd里获取所有的Node列表，然后遍历所有的Node，看看它是否有运行着带有某种label的Pod。如果没有，就创建一个；如果多了就删除多余的。在创建的时候，通过nodeSelector或者nodeAffiinity字段，还有tolerations字段来和需要运行的Node进行一对一绑定。其中tolerations字段使得DaemonSet能够比其他控制器更早出现在Kubernetes集群启动时候。对了，这个也顺带解决了我一个疑惑，就是我现在K8s环境里的Grafana为啥没有监控到master服务器，因为少了以下这个字段<br><br>tolerations:<br>- key: node-role.kubernetes.io&#47;master<br>  effect: NoSchedule<br><br>K8s中还有一个专门的API用于版本控制，它是ControllerRevision，它在自己的Data字段里保存了对应版本DaemonSet的API对象，还有在Annotation里保存了kubectl的命令</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-10 18:02:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/ac/fd/f70a6e3e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>大上海</span>
  </div>
  <div class="_2_QraFYR_0">GFW问题拉不下来，可以自己先拉下来，再创建<br><br>docker pull mirrorgooglecontainers&#47;fluentd-elasticsearch:v2.4.0<br><br>docker tag docker.io&#47;mirrorgooglecontainers&#47;fluentd-elasticsearch:v2.4.0 k8s.gcr.io&#47;fluentd-elasticsearch:v2.4.0</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-30 19:37:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/QKbgfE8mY91fjkLuyDKHUGlpfxKyhiaib5v3ic3YT6qrLibFWxoiaKCxzLeuJROiaWquCb0cNI0lCjiaDY92hSAKHsHUg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>罗罗</span>
  </div>
  <div class="_2_QraFYR_0">在这里，你应该注意到 nodeAffinity 的定义，可以支持更加丰富的语法，比如 operator: In（即：部分匹配；如果你定义 operator: Equal，就是完全匹配）  </div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-02 22:07:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/46/3f/7825378a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>无名氏</span>
  </div>
  <div class="_2_QraFYR_0">假设有这么一种场景，业务Pod需要在每个Node上有且只有1个，在灰度升级时，按10%，30%，50%，100%批量升级，后续某个版本再次升级时，这种升级策略可能会变更，比如10%，70%，100%，请问怎么处理，谢谢。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-01 16:48:12</div>
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
  <div class="_2_QraFYR_0">DJH 问的问题很好，有时候我也有过这样的疑问，稍纵即逝，没有记录下来。我试着回答一下。<br><br>1. 如果在一个容器里，命令总有先后顺序。那么你load数据的sql在server启动前如何执行。<br>2. 应该不会kill吧，不知道有没有参数控制。这样是不是要有一个downtime去做升级？因为可能出现版本不一致中间状态<br>3 应该把partation参数去掉吧，否则以后增加节点会不会出现意想不到的问题？<br>4应该是一个版本一个对象，每个版本对象版本号不一样<br>5 不了解，但是设计应该是一致的。可能有</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-10 09:33:28</div>
  </div>
</div>
</div>
</li>
</ul>