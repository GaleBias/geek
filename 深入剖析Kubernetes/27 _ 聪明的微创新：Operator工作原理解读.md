<audio title="27 _ 聪明的微创新：Operator工作原理解读" src="https://static001.geekbang.org/resource/audio/4d/a1/4d5b3efdb7687190bd4fefad136eb2a1.mp3" controls="controls"></audio> 
<p>你好，我是张磊。今天我和你分享的主题是：聪明的微创新之Operator工作原理解读。</p>
<p>在前面的几篇文章中，我已经和你分享了Kubernetes项目中的大部分编排对象（比如Deployment、StatefulSet、DaemonSet，以及Job），也介绍了“有状态应用”的管理方法，还阐述了为Kubernetes添加自定义API对象和编写自定义控制器的原理和流程。</p>
<p>可能你已经感觉到，在Kubernetes中，管理“有状态应用”是一个比较复杂的过程，尤其是编写Pod模板的时候，总有一种“在YAML文件里编程序”的感觉，让人很不舒服。</p>
<p>而在Kubernetes生态中，还有一个相对更加灵活和编程友好的管理“有状态应用”的解决方案，它就是：Operator。</p>
<p>接下来，我就以Etcd Operator为例，来为你讲解一下Operator的工作原理和编写方法。</p>
<p><span class="orange">Etcd Operator的使用方法非常简单，只需要两步即可完成：</span></p>
<p><strong>第一步，将这个Operator的代码Clone到本地：</strong></p>
<pre><code>$ git clone https://github.com/coreos/etcd-operator
</code></pre>
<p><strong>第二步，将这个Etcd Operator部署在Kubernetes集群里。</strong></p>
<p>不过，在部署Etcd Operator的Pod之前，你需要先执行这样一个脚本：</p><!-- [[[read_end]]] -->
<pre><code>$ example/rbac/create_role.sh
</code></pre>
<p>不用我多说你也能够明白：这个脚本的作用，就是为Etcd Operator创建RBAC规则。这是因为，Etcd Operator需要访问Kubernetes的APIServer来创建对象。</p>
<p>更具体地说，上述脚本为Etcd Operator定义了如下所示的权限：</p>
<ol>
<li>
<p>对Pod、Service、PVC、Deployment、Secret等API对象，有所有权限；</p>
</li>
<li>
<p>对CRD对象，有所有权限；</p>
</li>
<li>
<p>对属于etcd.database.coreos.com这个API Group的CR（Custom Resource）对象，有所有权限。</p>
</li>
</ol>
<p>而Etcd Operator本身，其实就是一个Deployment，它的YAML文件如下所示：</p>
<pre><code>apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: etcd-operator
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: etcd-operator
    spec:
      containers:
      - name: etcd-operator
        image: quay.io/coreos/etcd-operator:v0.9.2
        command:
        - etcd-operator
        env:
        - name: MY_POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: MY_POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
...
</code></pre>
<p>所以，我们就可以使用上述的YAML文件来创建Etcd Operator，如下所示：</p>
<pre><code>$ kubectl create -f example/deployment.yaml
</code></pre>
<p>而一旦Etcd Operator的Pod进入了Running状态，你就会发现，有一个CRD被自动创建了出来，如下所示：</p>
<pre><code>$ kubectl get pods
NAME                              READY     STATUS      RESTARTS   AGE
etcd-operator-649dbdb5cb-bzfzp    1/1       Running     0          20s

$ kubectl get crd
NAME                                    CREATED AT
etcdclusters.etcd.database.coreos.com   2018-09-18T11:42:55Z
</code></pre>
<p>这个CRD名叫<code>etcdclusters.etcd.database.coreos.com</code> 。你可以通过kubectl describe命令看到它的细节，如下所示：</p>
<pre><code>$ kubectl describe crd  etcdclusters.etcd.database.coreos.com
...
Group:   etcd.database.coreos.com
  Names:
    Kind:       EtcdCluster
    List Kind:  EtcdClusterList
    Plural:     etcdclusters
    Short Names:
      etcd
    Singular:  etcdcluster
  Scope:       Namespaced
  Version:     v1beta2
  
...
</code></pre>
<p>可以看到，这个CRD相当于告诉了Kubernetes：接下来，如果有API组（Group）是<code>etcd.database.coreos.com</code>、API资源类型（Kind）是“EtcdCluster”的YAML文件被提交上来，你可一定要认识啊。</p>
<p>所以说，通过上述两步操作，你实际上是在Kubernetes里添加了一个名叫EtcdCluster的自定义资源类型。而Etcd Operator本身，就是这个自定义资源类型对应的自定义控制器。</p>
<p>而当Etcd Operator部署好之后，接下来在这个Kubernetes里创建一个Etcd集群的工作就非常简单了。你只需要编写一个EtcdCluster的YAML文件，然后把它提交给Kubernetes即可，如下所示：</p>
<pre><code>$ kubectl apply -f example/example-etcd-cluster.yaml
</code></pre>
<p>这个example-etcd-cluster.yaml文件里描述的，是一个3个节点的Etcd集群。我们可以看到它被提交给Kubernetes之后，就会有三个Etcd的Pod运行起来，如下所示：</p>
<pre><code>$ kubectl get pods
NAME                            READY     STATUS    RESTARTS   AGE
example-etcd-cluster-dp8nqtjznc   1/1       Running     0          1m
example-etcd-cluster-mbzlg6sd56   1/1       Running     0          2m
example-etcd-cluster-v6v6s6stxd   1/1       Running     0          2m
</code></pre>
<p>那么，<span class="orange">究竟发生了什么，让创建一个Etcd集群的工作如此简单呢？</span></p>
<p>我们当然还是得从这个example-etcd-cluster.yaml文件开始说起。</p>
<p>不难想到，这个文件里定义的，正是EtcdCluster这个CRD的一个具体实例，也就是一个Custom Resource（CR）。而它的内容非常简单，如下所示：</p>
<pre><code>apiVersion: &quot;etcd.database.coreos.com/v1beta2&quot;
kind: &quot;EtcdCluster&quot;
metadata:
  name: &quot;example-etcd-cluster&quot;
spec:
  size: 3
  version: &quot;3.2.13&quot;
</code></pre>
<p>可以看到，EtcdCluster的spec字段非常简单。其中，size=3指定了它所描述的Etcd集群的节点个数。而version=“3.2.13”，则指定了Etcd的版本，仅此而已。</p>
<p>而真正把这样一个Etcd集群创建出来的逻辑，就是Etcd Operator要实现的主要工作了。</p>
<p>看到这里，相信你应该已经对Operator有了一个初步的认知：</p>
<p><strong>Operator的工作原理，实际上是利用了Kubernetes的自定义API资源（CRD），来描述我们想要部署的“有状态应用”；然后在自定义控制器里，根据自定义API对象的变化，来完成具体的部署和运维工作。</strong></p>
<p>所以，编写一个Etcd Operator，与我们前面编写一个自定义控制器的过程，没什么不同。</p>
<p>不过，考虑到你可能还不太清楚<span class="orange">Etcd集群的组建方式</span>，我在这里先简单介绍一下这部分知识。</p>
<p><strong>Etcd Operator部署Etcd集群，采用的是静态集群（Static）的方式</strong>。</p>
<p>静态集群的好处是，它不必依赖于一个额外的服务发现机制来组建集群，非常适合本地容器化部署。而它的难点，则在于你必须在部署的时候，就规划好这个集群的拓扑结构，并且能够知道这些节点固定的IP地址。比如下面这个例子：</p>
<pre><code>$ etcd --name infra0 --initial-advertise-peer-urls http://10.0.1.10:2380 \
  --listen-peer-urls http://10.0.1.10:2380 \
...
  --initial-cluster-token etcd-cluster-1 \
  --initial-cluster infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380 \
  --initial-cluster-state new
  
$ etcd --name infra1 --initial-advertise-peer-urls http://10.0.1.11:2380 \
  --listen-peer-urls http://10.0.1.11:2380 \
...
  --initial-cluster-token etcd-cluster-1 \
  --initial-cluster infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380 \
  --initial-cluster-state new
  
$ etcd --name infra2 --initial-advertise-peer-urls http://10.0.1.12:2380 \
  --listen-peer-urls http://10.0.1.12:2380 \
...
  --initial-cluster-token etcd-cluster-1 \
  --initial-cluster infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380 \
  --initial-cluster-state new
</code></pre>
<p>在这个例子中，我启动了三个Etcd进程，组成了一个三节点的Etcd集群。</p>
<p>其中，这些节点启动参数里的–initial-cluster参数，非常值得你关注。它的含义，正是<strong>当前节点启动时集群的拓扑结构。<strong>说得更详细一点，就是</strong>当前这个节点启动时，需要跟哪些节点通信来组成集群</strong>。</p>
<p>举个例子，我们可以看一下上述infra2节点的–initial-cluster的值，如下所示：</p>
<pre><code>...
--initial-cluster infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380 \
</code></pre>
<p>可以看到，–initial-cluster参数是由“&lt;节点名字&gt;=&lt;节点地址&gt;”格式组成的一个数组。而上面这个配置的意思就是，当infra2节点启动之后，这个Etcd集群里就会有infra0、infra1和infra2三个节点。</p>
<p>同时，这些Etcd节点，需要通过2380端口进行通信以便组成集群，这也正是上述配置中–listen-peer-urls字段的含义。</p>
<p>此外，一个Etcd集群还需要用–initial-cluster-token字段，来声明一个该集群独一无二的Token名字。</p>
<p>像上述这样为每一个Ectd节点配置好它对应的启动参数之后把它们启动起来，一个Etcd集群就可以自动组建起来了。</p>
<p>而我们要编写的Etcd Operator，就是要把上述过程自动化。这其实等同于：用代码来生成每个Etcd节点Pod的启动命令，然后把它们启动起来。</p>
<p>接下来，我们一起来实践一下这个流程。</p>
<p>当然，在编写自定义控制器之前，我们首先需要完成EtcdCluster这个CRD的定义，它对应的types.go文件的主要内容，如下所示：</p>
<pre><code>// +genclient
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

type EtcdCluster struct {
  metav1.TypeMeta   `json:&quot;,inline&quot;`
  metav1.ObjectMeta `json:&quot;metadata,omitempty&quot;`
  Spec              ClusterSpec   `json:&quot;spec&quot;`
  Status            ClusterStatus `json:&quot;status&quot;`
}

type ClusterSpec struct {
 // Size is the expected size of the etcd cluster.
 // The etcd-operator will eventually make the size of the running
 // cluster equal to the expected size.
 // The vaild range of the size is from 1 to 7.
 Size int `json:&quot;size&quot;`
 ... 
}
</code></pre>
<p>可以看到，EtcdCluster是一个有Status字段的CRD。在这里，我们可以不必关心ClusterSpec里的其他字段，只关注Size（即：Etcd集群的大小）字段即可。</p>
<p>Size字段的存在，就意味着将来如果我们想要调整集群大小的话，应该直接修改YAML文件里size的值，并执行kubectl apply -f。</p>
<p>这样，Operator就会帮我们完成Etcd节点的增删操作。这种“scale”能力，也是Etcd Operator自动化运维Etcd集群需要实现的主要功能。</p>
<p>而为了能够支持这个功能，我们就不再像前面那样在–initial-cluster参数里把拓扑结构固定死。</p>
<p>所以，Etcd Operator的实现，虽然选择的也是静态集群，但这个集群具体的组建过程，是逐个节点动态添加的方式，即：</p>
<p><strong>首先，Etcd Operator会创建一个“种子节点”；</strong><br />
<strong>然后，Etcd Operator会不断创建新的Etcd节点，然后将它们逐一加入到这个集群当中，直到集群的节点数等于size。</strong></p>
<p>这就意味着，在生成不同角色的Etcd Pod时，Operator需要能够区分种子节点与普通节点。</p>
<p>而这两种节点的不同之处，就在于一个名叫–initial-cluster-state的启动参数：</p>
<ul>
<li>当这个参数值设为new时，就代表了该节点是种子节点。而我们前面提到过，种子节点还必须通过–initial-cluster-token声明一个独一无二的Token。</li>
<li>而如果这个参数值设为existing，那就是说明这个节点是一个普通节点，Etcd Operator需要把它加入到已有集群里。</li>
</ul>
<p>那么接下来的问题就是，每个Etcd节点的–initial-cluster字段的值又是怎么生成的呢？</p>
<p>由于这个方案要求种子节点先启动，所以对于种子节点infra0来说，它启动后的集群只有它自己，即：–initial-cluster=infra0=http://10.0.1.10:2380。</p>
<p>而对于接下来要加入的节点，比如infra1来说，它启动后的集群就有两个节点了，所以它的–initial-cluster参数的值应该是：infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380。</p>
<p>其他节点，都以此类推。</p>
<p>现在，你就应该能在脑海中构思出上述三节点Etcd集群的部署过程了。</p>
<p>首先，只要用户提交YAML文件时声明创建一个EtcdCluster对象（一个Etcd集群），那么Etcd Operator都应该先创建一个单节点的种子集群（Seed Member），并启动这个种子节点。</p>
<p>以infra0节点为例，它的IP地址是10.0.1.10，那么Etcd Operator生成的种子节点的启动命令，如下所示：</p>
<pre><code>$ etcd
  --data-dir=/var/etcd/data
  --name=infra0
  --initial-advertise-peer-urls=http://10.0.1.10:2380
  --listen-peer-urls=http://0.0.0.0:2380
  --listen-client-urls=http://0.0.0.0:2379
  --advertise-client-urls=http://10.0.1.10:2379
  --initial-cluster=infra0=http://10.0.1.10:2380
  --initial-cluster-state=new
  --initial-cluster-token=4b5215fa-5401-4a95-a8c6-892317c9bef8
</code></pre>
<p>可以看到，这个种子节点的initial-cluster-state是new，并且指定了唯一的initial-cluster-token参数。</p>
<p>我们可以把这个创建种子节点（集群）的阶段称为：<strong>Bootstrap</strong>。</p>
<p>接下来，<strong>对于其他每一个节点，Operator只需要执行如下两个操作即可</strong>，以infra1为例。</p>
<p>第一步：通过Etcd命令行添加一个新成员：</p>
<pre><code>$ etcdctl member add infra1 http://10.0.1.11:2380
</code></pre>
<p>第二步：为这个成员节点生成对应的启动参数，并启动它：</p>
<pre><code>$ etcd
    --data-dir=/var/etcd/data
    --name=infra1
    --initial-advertise-peer-urls=http://10.0.1.11:2380
    --listen-peer-urls=http://0.0.0.0:2380
    --listen-client-urls=http://0.0.0.0:2379
    --advertise-client-urls=http://10.0.1.11:2379
    --initial-cluster=infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380
    --initial-cluster-state=existing
</code></pre>
<p>可以看到，对于这个infra1成员节点来说，它的initial-cluster-state是existing，也就是要加入已有集群。而它的initial-cluster的值，则变成了infra0和infra1两个节点的IP地址。</p>
<p>所以，以此类推，不断地将infra2等后续成员添加到集群中，直到整个集群的节点数目等于用户指定的size之后，部署就完成了。</p>
<p>在熟悉了这个部署思路之后，我再为你讲解<span class="orange">Etcd Operator的工作原理</span>，就非常简单了。</p>
<p>跟所有的自定义控制器一样，Etcd Operator的启动流程也是围绕着Informer展开的，如下所示：</p>
<pre><code>func (c *Controller) Start() error {
 for {
  err := c.initResource()
  ...
  time.Sleep(initRetryWaitTime)
 }
 c.run()
}

func (c *Controller) run() {
 ...
 
 _, informer := cache.NewIndexerInformer(source, &amp;api.EtcdCluster{}, 0, cache.ResourceEventHandlerFuncs{
  AddFunc:    c.onAddEtcdClus,
  UpdateFunc: c.onUpdateEtcdClus,
  DeleteFunc: c.onDeleteEtcdClus,
 }, cache.Indexers{})
 
 ctx := context.TODO()
 // TODO: use workqueue to avoid blocking
 informer.Run(ctx.Done())
}
</code></pre>
<p>可以看到，<strong>Etcd Operator启动要做的第一件事</strong>（ c.initResource），是创建EtcdCluster对象所需要的CRD，即：前面提到的<code>etcdclusters.etcd.database.coreos.com</code>。这样Kubernetes就能够“认识”EtcdCluster这个自定义API资源了。</p>
<p>而<strong>接下来，Etcd Operator会定义一个EtcdCluster对象的Informer</strong>。</p>
<p>不过，需要注意的是，由于Etcd Operator的完成时间相对较早，所以它里面有些代码的编写方式会跟我们之前讲解的最新的编写方式不太一样。在具体实践的时候，你还是应该以我讲解的模板为主。</p>
<p>比如，在上面的代码最后，你会看到有这样一句注释：</p>
<pre><code>// TODO: use workqueue to avoid blocking
...
</code></pre>
<p>也就是说，Etcd Operator并没有用工作队列来协调Informer和控制循环。这其实正是我在第25篇文章<a href="https://time.geekbang.org/column/article/42076">《深入解析声明式API（二）：编写自定义控制器》</a>中，给你留的关于工作队列的思考题的答案。</p>
<p>具体来讲，我们在控制循环里执行的业务逻辑，往往是比较耗时间的。比如，创建一个真实的Etcd集群。而Informer的WATCH机制对API对象变化的响应，则非常迅速。所以，控制器里的业务逻辑就很可能会拖慢Informer的执行周期，甚至可能Block它。而要协调这样两个快、慢任务的一个典型解决方法，就是引入一个工作队列。</p>
<blockquote>
<p>备注：如果你感兴趣的话，可以给Etcd Operator提一个patch来修复这个问题。提PR修TODO，是给一个开源项目做有意义的贡献的一个重要方式。</p>
</blockquote>
<p>由于Etcd Operator里没有工作队列，那么在它的EventHandler部分，就不会有什么入队操作，而直接就是每种事件对应的具体的业务逻辑了。</p>
<p>不过，Etcd Operator在业务逻辑的实现方式上，与常规的自定义控制器略有不同。我把在这一部分的工作原理，提炼成了一个详细的流程图，如下所示：</p>
<p><img src="https://static001.geekbang.org/resource/image/e7/36/e7f2905ae46e0ccd24db47c915382536.jpg?wh=1920*1080" alt="" /></p>
<p>可以看到，Etcd Operator的特殊之处在于，它为每一个EtcdCluster对象，都启动了一个控制循环，“并发”地响应这些对象的变化。显然，这种做法不仅可以简化Etcd Operator的代码实现，还有助于提高它的响应速度。</p>
<p>以文章一开始的example-etcd-cluster的YAML文件为例。</p>
<p>当这个YAML文件第一次被提交到Kubernetes之后，Etcd Operator的Informer，就会立刻“感知”到一个新的EtcdCluster对象被创建了出来。所以，EventHandler里的“添加”事件会被触发。</p>
<p>而这个Handler要做的操作也很简单，即：在Etcd Operator内部创建一个对应的Cluster对象（cluster.New），比如流程图里的Cluster1。</p>
<p>这个Cluster对象，就是一个Etcd集群在Operator内部的描述，所以它与真实的Etcd集群的生命周期是一致的。</p>
<p>而一个Cluster对象需要具体负责的，其实有两个工作。</p>
<p><strong>其中，第一个工作只在该Cluster对象第一次被创建的时候才会执行。这个工作，就是我们前面提到过的Bootstrap，即：创建一个单节点的种子集群。</strong></p>
<p>由于种子集群只有一个节点，所以这一步直接就会生成一个Etcd的Pod对象。这个Pod里有一个InitContainer，负责检查Pod的DNS记录是否正常。如果检查通过，用户容器也就是Etcd容器就会启动起来。</p>
<p>而这个Etcd容器最重要的部分，当然就是它的启动命令了。</p>
<p>以我们在文章一开始部署的集群为例，它的种子节点的容器启动命令如下所示：</p>
<pre><code>/usr/local/bin/etcd
  --data-dir=/var/etcd/data
  --name=example-etcd-cluster-mbzlg6sd56
  --initial-advertise-peer-urls=http://example-etcd-cluster-mbzlg6sd56.example-etcd-cluster.default.svc:2380
  --listen-peer-urls=http://0.0.0.0:2380
  --listen-client-urls=http://0.0.0.0:2379
  --advertise-client-urls=http://example-etcd-cluster-mbzlg6sd56.example-etcd-cluster.default.svc:2379
  --initial-cluster=example-etcd-cluster-mbzlg6sd56=http://example-etcd-cluster-mbzlg6sd56.example-etcd-cluster.default.svc:2380
  --initial-cluster-state=new
  --initial-cluster-token=4b5215fa-5401-4a95-a8c6-892317c9bef8
</code></pre>
<p>上述启动命令里的各个参数的含义，我已经在前面介绍过。</p>
<p>可以看到，在这些启动参数（比如：initial-cluster）里，Etcd Operator只会使用Pod的DNS记录，而不是它的IP地址。</p>
<p>这当然是因为，在Operator生成上述启动命令的时候，Etcd的Pod还没有被创建出来，它的IP地址自然也无从谈起。</p>
<p>这也就意味着，每个Cluster对象，都会事先创建一个与该EtcdCluster同名的Headless Service。这样，Etcd Operator在接下来的所有创建Pod的步骤里，就都可以使用Pod的DNS记录来代替它的IP地址了。</p>
<blockquote>
<p>备注：Headless Service的DNS记录格式是：<pod-name>.<svc-name>.<namespace>.svc.cluster.local。如果你记不太清楚了，可以借此再回顾一下第18篇文章<a href="https://time.geekbang.org/column/article/41017">《深入理解StatefulSet（一）：拓扑状态》</a>中的相关内容。</p>
</blockquote>
<p><strong>Cluster对象的第二个工作，则是启动该集群所对应的控制循环。</strong></p>
<p>这个控制循环每隔一定时间，就会执行一次下面的Diff流程。</p>
<p>首先，控制循环要获取到所有正在运行的、属于这个Cluster的Pod数量，也就是该Etcd集群的“实际状态”。</p>
<p>而这个Etcd集群的“期望状态”，正是用户在EtcdCluster对象里定义的size。</p>
<p>所以接下来，控制循环会对比这两个状态的差异。</p>
<p>如果实际的Pod数量不够，那么控制循环就会执行一个添加成员节点的操作（即：上述流程图中的addOneMember方法）；反之，就执行删除成员节点的操作（即：上述流程图中的removeOneMember方法）。</p>
<p>以addOneMember方法为例，它执行的流程如下所示：</p>
<ol>
<li>
<p>生成一个新节点的Pod的名字，比如：example-etcd-cluster-v6v6s6stxd；</p>
</li>
<li>
<p>调用Etcd Client，执行前面提到过的etcdctl member add example-etcd-cluster-v6v6s6stxd命令；</p>
</li>
<li>
<p>使用这个Pod名字，和已经存在的所有节点列表，组合成一个新的initial-cluster字段的值；</p>
</li>
<li>
<p>使用这个initial-cluster的值，生成这个Pod里Etcd容器的启动命令。如下所示：</p>
</li>
</ol>
<pre><code>/usr/local/bin/etcd
  --data-dir=/var/etcd/data
  --name=example-etcd-cluster-v6v6s6stxd
  --initial-advertise-peer-urls=http://example-etcd-cluster-v6v6s6stxd.example-etcd-cluster.default.svc:2380
  --listen-peer-urls=http://0.0.0.0:2380
  --listen-client-urls=http://0.0.0.0:2379
  --advertise-client-urls=http://example-etcd-cluster-v6v6s6stxd.example-etcd-cluster.default.svc:2379
  --initial-cluster=example-etcd-cluster-mbzlg6sd56=http://example-etcd-cluster-mbzlg6sd56.example-etcd-cluster.default.svc:2380,example-etcd-cluster-v6v6s6stxd=http://example-etcd-cluster-v6v6s6stxd.example-etcd-cluster.default.svc:2380
  --initial-cluster-state=existing
</code></pre>
<p>这样，当这个容器启动之后，一个新的Etcd成员节点就被加入到了集群当中。控制循环会重复这个过程，直到正在运行的Pod数量与EtcdCluster指定的size一致。</p>
<p>在有了这样一个与EtcdCluster对象一一对应的控制循环之后，你后续对这个EtcdCluster的任何修改，比如：修改size或者Etcd的version，它们对应的更新事件都会由这个Cluster对象的控制循环进行处理。</p>
<p>以上，就是一个Etcd Operator的工作原理了。</p>
<p>如果对比一下Etcd Operator与我在第20篇文章<a href="https://time.geekbang.org/column/article/41217">《深入理解StatefulSet（三）：有状态应用实践》</a>中讲解过的MySQL StatefulSet的话，你可能会有两个问题。</p>
<p><strong>第一个问题是</strong>，在StatefulSet里，它为Pod创建的名字是带编号的，这样就把整个集群的拓扑状态固定了下来（比如：一个三节点的集群一定是由名叫web-0、web-1和web-2的三个Pod组成）。可是，<strong>在Etcd Operator里，为什么我们使用随机名字就可以了呢？</strong></p>
<p>这是因为，Etcd Operator在每次添加Etcd节点的时候，都会先执行etcdctl member add &lt;Pod名字&gt;；每次删除节点的时候，则会执行etcdctl member remove &lt;Pod名字&gt;。这些操作，其实就会更新Etcd内部维护的拓扑信息，所以Etcd Operator无需在集群外部通过编号来固定这个拓扑关系。</p>
<p><strong>第二个问题是，为什么我没有在EtcdCluster对象里声明Persistent Volume？</strong></p>
<p>难道，我们不担心节点宕机之后Etcd的数据会丢失吗？</p>
<p>我们知道，Etcd是一个基于Raft协议实现的高可用Key-Value存储。根据Raft协议的设计原则，当Etcd集群里只有半数以下（在我们的例子里，小于等于一个）的节点失效时，当前集群依然可以正常工作。此时，Etcd Operator只需要通过控制循环创建出新的Pod，然后将它们加入到现有集群里，就完成了“期望状态”与“实际状态”的调谐工作。这个集群，是一直可用的 。</p>
<blockquote>
<p>备注：关于Etcd的工作原理和Raft协议的设计思想，你可以阅读<a href="http://www.infoq.com/cn/articles/etcd-interpretation-application-scenario-implement-principle">这篇文章</a>来进行学习。</p>
</blockquote>
<p>但是，当这个Etcd集群里有半数以上（在我们的例子里，大于等于两个）的节点失效的时候，这个集群就会丧失数据写入的能力，从而进入“不可用”状态。此时，即使Etcd Operator创建出新的Pod出来，Etcd集群本身也无法自动恢复起来。</p>
<p>这个时候，我们就必须使用Etcd本身的备份数据来对集群进行恢复操作。</p>
<p>在有了Operator机制之后，上述Etcd的备份操作，是由一个单独的Etcd Backup Operator负责完成的。</p>
<p>创建和使用这个Operator的流程，如下所示：</p>
<pre><code># 首先，创建etcd-backup-operator
$ kubectl create -f example/etcd-backup-operator/deployment.yaml

# 确认etcd-backup-operator已经在正常运行
$ kubectl get pod
NAME                                    READY     STATUS    RESTARTS   AGE
etcd-backup-operator-1102130733-hhgt7   1/1       Running   0          3s

# 可以看到，Backup Operator会创建一个叫etcdbackups的CRD
$ kubectl get crd
NAME                                    KIND
etcdbackups.etcd.database.coreos.com    CustomResourceDefinition.v1beta1.apiextensions.k8s.io

# 我们这里要使用AWS S3来存储备份，需要将S3的授权信息配置在文件里
$ cat $AWS_DIR/credentials
[default]
aws_access_key_id = XXX
aws_secret_access_key = XXX

$ cat $AWS_DIR/config
[default]
region = &lt;region&gt;

# 然后，将上述授权信息制作成一个Secret
$ kubectl create secret generic aws --from-file=$AWS_DIR/credentials --from-file=$AWS_DIR/config

# 使用上述S3的访问信息，创建一个EtcdBackup对象
$ sed -e 's|&lt;full-s3-path&gt;|mybucket/etcd.backup|g' \
    -e 's|&lt;aws-secret&gt;|aws|g' \
    -e 's|&lt;etcd-cluster-endpoints&gt;|&quot;http://example-etcd-cluster-client:2379&quot;|g' \
    example/etcd-backup-operator/backup_cr.yaml \
    | kubectl create -f -
</code></pre>
<p>需要注意的是，每当你创建一个EtcdBackup对象（<a href="https://github.com/coreos/etcd-operator/blob/master/example/etcd-backup-operator/backup_cr.yaml">backup_cr.yaml</a>），就相当于为它所指定的Etcd集群做了一次备份。EtcdBackup对象的etcdEndpoints字段，会指定它要备份的Etcd集群的访问地址。</p>
<p>所以，在实际的环境里，我建议你把最后这个备份操作，编写成一个Kubernetes的CronJob以便定时运行。</p>
<p>而当Etcd集群发生了故障之后，你就可以通过创建一个EtcdRestore对象来完成恢复操作。当然，这就意味着你也需要事先启动Etcd Restore Operator。</p>
<p>这个流程的完整过程，如下所示：</p>
<pre><code># 创建etcd-restore-operator
$ kubectl create -f example/etcd-restore-operator/deployment.yaml

# 确认它已经正常运行
$ kubectl get pods
NAME                                     READY     STATUS    RESTARTS   AGE
etcd-restore-operator-4203122180-npn3g   1/1       Running   0          7s

# 创建一个EtcdRestore对象，来帮助Etcd Operator恢复数据，记得替换模板里的S3的访问信息
$ sed -e 's|&lt;full-s3-path&gt;|mybucket/etcd.backup|g' \
    -e 's|&lt;aws-secret&gt;|aws|g' \
    example/etcd-restore-operator/restore_cr.yaml \
    | kubectl create -f -
</code></pre>
<p>上面例子里的EtcdRestore对象（<a href="https://github.com/coreos/etcd-operator/blob/master/example/etcd-restore-operator/restore_cr.yaml">restore_cr.yaml</a>），会指定它要恢复的Etcd集群的名字和备份数据所在的S3存储的访问信息。</p>
<p>而当一个EtcdRestore对象成功创建后，Etcd Restore Operator就会通过上述信息，恢复出一个全新的Etcd集群。然后，Etcd Operator会把这个新集群直接接管过来，从而重新进入可用的状态。</p>
<p>EtcdBackup和EtcdRestore这两个Operator的工作原理，与Etcd Operator的实现方式非常类似。所以，这一部分就交给你课后去探索了。</p>
<h2>总结</h2>
<p>在今天这篇文章中，我以Etcd Operator为例，详细介绍了一个Operator的工作原理和编写过程。</p>
<p>可以看到，Etcd集群本身就拥有良好的分布式设计和一定的高可用能力。在这种情况下，StatefulSet“为Pod编号”和“将Pod同PV绑定”这两个主要的特性，就不太有用武之地了。</p>
<p>而相比之下，Etcd Operator把一个Etcd集群，抽象成了一个具有一定“自治能力”的整体。而当这个“自治能力”本身不足以解决问题的时候，我们可以通过两个专门负责备份和恢复的Operator进行修正。这种实现方式，不仅更加贴近Etcd的设计思想，也更加编程友好。</p>
<p>不过，如果我现在要部署的应用，既需要用StatefulSet的方式维持拓扑状态和存储状态，又有大量的编程工作要做，那我到底该如何选择呢？</p>
<p>其实，Operator和StatefulSet并不是竞争关系。你完全可以编写一个Operator，然后在Operator的控制循环里创建和控制StatefulSet而不是Pod。比如，业界知名的<a href="https://github.com/coreos/prometheus-operator">Prometheus项目的Operator</a>，正是这么实现的。</p>
<p>此外，CoreOS公司在被RedHat公司收购之后，已经把Operator的编写过程封装成了一个叫作<a href="https://github.com/operator-framework/operator-sdk">Operator SDK</a>的工具（整个项目叫作Operator Framework），它可以帮助你生成Operator的框架代码。感兴趣的话，你可以试用一下。</p>
<h2>思考题</h2>
<p>在Operator的实现过程中，我们再一次用到了CRD。可是，你一定要明白，CRD并不是万能的，它有很多场景不适用，还有性能瓶颈。你能列举出一些不适用CRD的场景么？你知道造成CRD性能瓶颈的原因主要在哪里么？</p>
<p>感谢你的收听，欢迎你给我留言，也欢迎分享给更多的朋友一起阅读。</p>
<p></p>

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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/96/ec/5c53f9e5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小熹</span>
  </div>
  <div class="_2_QraFYR_0">CRD在ETCD里面是以JSON格式来存储的，而K8S的API对象是以protobuf格式存储，在资源对象数量多的时候JSON的序列化和反序列化性能会成为瓶颈。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-18 16:15:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>程序修行</span>
  </div>
  <div class="_2_QraFYR_0"> 已经到了看不懂的时候了～。丧心</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-20 20:06:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/96/89/9312b3a2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Vincen</span>
  </div>
  <div class="_2_QraFYR_0">是否可以将operator和自定义控制器划等号</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 原理上是一个东西</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-24 09:44:04</div>
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
  <div class="_2_QraFYR_0">感觉operator就是一个自定义的controller，需要有一定的开发能力来实现。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-24 09:05:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/c8/8f/759b7761.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ethfoo</span>
  </div>
  <div class="_2_QraFYR_0">etcd operator现在使用empty dir存储etcd数据，万一真的出现大部分节点挂掉，但是数据备份又不是实时的，会存在部分数据丢失的情况吧？那为什么etcd operator不使用pv呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 要是大部分数据丢失备份又不实时的时候，用pv把数据找回来就能完美work，那还要raft干啥。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-24 08:14:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/fd/4d/41875340.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Simon Wu</span>
  </div>
  <div class="_2_QraFYR_0">Etcd operator已经支持pvc了，依靠备份机制还是有一段时间的数据丢失，就看你的业务允不允许。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-22 17:01:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/18/3c/b5a00c1a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>johnson.skiii</span>
  </div>
  <div class="_2_QraFYR_0">张大，请教个问题。<br><br>如果ETCD每个pod不绑定pv的话，那么当其中一个挂掉的话，etcd operator会再启动一个pod，但是启动到那一台worker未知；那么，由于raft机制，leader会同步数据给这个新的pod，这个过程可能会很耗时；<br><br>与其这样，能否限定运行etcd的pod使用特定的pv，这样当出现pod挂掉的情况，leader同步数据也会更快。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 恢复数据这个操作还是要的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-09 12:25:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/04/0d/3dc5683a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>柯察金</span>
  </div>
  <div class="_2_QraFYR_0">我问一个问题，平时开发人员需要用到的 CRD 的地方多吗，如果要自定义的话还是挺麻烦的呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: kubebuilder operatorsdk都是帮你简化操作的工具</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-08 19:45:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/83/af/1cb42cd3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>马以</span>
  </div>
  <div class="_2_QraFYR_0">该文章中的例子其实是创建不成功的，因为我的k8s版本是1.23.0，最新的api已经从原来的extensions&#47;v1beta1变为了apps&#47;v1,所以这里在创建example&#47;deployment.yaml时虽然能够成功，但是创建CRD的时候是失败的，我们通过log也可以看出来：<br>Could not construct reference to: &#39;&amp;v1.Endpoints{TypeMeta:v1.TypeMeta{Kind:&quot;&quot;, APIVersion:&quot;&quot;}, ObjectMeta:v1.ObjectMeta{Name:&quot;etcd-operator&quot;, GenerateName:&quot;&quot;, Namespace:&quot;default.......<br><br>time=&quot;2022-03-12T06:46:45Z&quot; level=error msg=&quot;initialization failed: fail to init CRD: failed to create CRD: the server could not find the requested resource&quot; pkg=controller<br><br>我想解决办法只能是修改源码吧，但是我不会go，很难受😣</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-12 15:33:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/f9/d3/2cb7516e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>？</span>
  </div>
  <div class="_2_QraFYR_0">我觉得这一章应该算扩展，需要有一定基础，并且编写过crd的朋友来学习的，普通的使用还用不上crd，毕竟很多开源项目都自定义了crd，我们只需要查看他们的文档配置即可。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-16 10:52:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/9b/5d/f99d0ce6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>化石</span>
  </div>
  <div class="_2_QraFYR_0">最前面的例子 etcd 创建 infra0-infra2 3 个节点的时候，--initial-cluster-state 都是 new，是否和后面的内容不一致?</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-07 22:57:44</div>
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
  <div class="_2_QraFYR_0">请教老师，EtcdBackup和EtcdRestore资源创建（并运行）后会自动删除吗？是否需要手工删除？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 需要自己清理</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-24 19:26:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/96/89/9312b3a2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Vincen</span>
  </div>
  <div class="_2_QraFYR_0">厉害了，全是干货！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-24 09:11:18</div>
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
  <div class="_2_QraFYR_0">有点难了，得多看几遍，留个爪</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-26 21:29:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/b4/94/2796de72.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>追风筝的人</span>
  </div>
  <div class="_2_QraFYR_0">已经到了看不懂的时候了   这是个悲伤的故事</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-06 15:06:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/20/41/7b/cda2e622.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>致良知</span>
  </div>
  <div class="_2_QraFYR_0">学到了 真心不错</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-05 07:38:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/19/ac/32da70c9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Spring.Fred</span>
  </div>
  <div class="_2_QraFYR_0">之前用StatefulSet实现的mysql主从读写分离的例子， 能给个Operator的示例代码吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-23 21:27:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/a5/98/a65ff31a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>djfhchdh</span>
  </div>
  <div class="_2_QraFYR_0">应该是client端与apiserver交互存在性能瓶颈</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-09 15:03:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/9f/a1/d75219ee.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>po</span>
  </div>
  <div class="_2_QraFYR_0">请教一个问题，etcd operator 使用empty dir存储etcd数据，那么在集群突然断电或者关机的情况下，再启动集群，那么所有etcd的pod数据都是为空，那raft算法在这个时候也没法工作了吧？这个问题要怎么解决呢？我看现在openshift集群的etcd就是用operator部署的，请老师解答下，谢谢。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-13 00:46:01</div>
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
  <div class="_2_QraFYR_0">老师，在测试官方的prometheus-operator。因为我的集群是二进制部署的，我想监控到我的组件，kube-contorller-manager和kube-scheduler组件，我做了一些修改（把这些组件的bind address设定为了0.0.0.0，自己单独创建了他们的ep）。当时是能在prometheus的targets看到，并且也能监控到。但是过了一段时间后，这些svc和ep会自动失效。这个问题怎么来查看呢？<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: kube proxy不知道你的改动</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-26 15:04:19</div>
  </div>
</div>
</div>
</li>
</ul>