<audio title="28 _ PV、PVC、StorageClass，这些到底在说啥？" src="https://static001.geekbang.org/resource/audio/32/5c/327d67f77f2389f233559ebcf54a715c.mp3" controls="controls"></audio> 
<p>你好，我是张磊。今天我和你分享的主题是：PV、PVC、StorageClass，这些到底在说啥？</p>
<p>在前面的文章中，我重点为你分析了Kubernetes的各种编排能力。</p>
<p>在这些讲解中，你应该已经发现，容器化一个应用比较麻烦的地方，莫过于对其“状态”的管理。而最常见的“状态”，又莫过于存储状态了。</p>
<p>所以，从今天这篇文章开始，我会<strong>通过4篇文章为你剖析Kubernetes项目处理容器持久化存储的核心原理</strong>，从而帮助你更好地理解和使用这部分内容。</p>
<p>首先，我们来回忆一下我在第19篇文章<a href="https://time.geekbang.org/column/article/41154">《深入理解StatefulSet（二）：存储状态》</a>中，和你分享StatefulSet如何管理存储状态的时候，介绍过的<span class="orange">Persistent Volume（PV）和Persistent Volume Claim（PVC）</span>这套持久化存储体系。</p>
<p>其中，<strong>PV描述的，是持久化存储数据卷</strong>。这个API对象主要定义的是一个持久化存储在宿主机上的目录，比如一个NFS的挂载目录。</p>
<p>通常情况下，PV对象是由运维人员事先创建在Kubernetes集群里待用的。比如，运维人员可以定义这样一个NFS类型的PV，如下所示：</p>
<pre><code>apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  nfs:
    server: 10.244.1.4
    path: &quot;/&quot;
</code></pre>
<p>而<strong>PVC描述的，则是Pod所希望使用的持久化存储的属性</strong>。比如，Volume存储的大小、可读写权限等等。</p><!-- [[[read_end]]] -->
<p>PVC对象通常由开发人员创建；或者以PVC模板的方式成为StatefulSet的一部分，然后由StatefulSet控制器负责创建带编号的PVC。</p>
<p>比如，开发人员可以声明一个1 GiB大小的PVC，如下所示：</p>
<pre><code>apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: manual
  resources:
    requests:
      storage: 1Gi
</code></pre>
<p>而用户创建的PVC要真正被容器使用起来，就必须先和某个符合条件的PV进行绑定。这里要检查的条件，包括两部分：</p>
<ul>
<li>第一个条件，当然是PV和PVC的spec字段。比如，PV的存储（storage）大小，就必须满足PVC的要求。</li>
<li>而第二个条件，则是PV和PVC的storageClassName字段必须一样。这个机制我会在本篇文章的最后一部分专门介绍。</li>
</ul>
<p>在成功地将PVC和PV进行绑定之后，Pod就能够像使用hostPath等常规类型的Volume一样，在自己的YAML文件里声明使用这个PVC了，如下所示：</p>
<pre><code>apiVersion: v1
kind: Pod
metadata:
  labels:
    role: web-frontend
spec:
  containers:
  - name: web
    image: nginx
    ports:
      - name: web
        containerPort: 80
    volumeMounts:
        - name: nfs
          mountPath: &quot;/usr/share/nginx/html&quot;
  volumes:
  - name: nfs
    persistentVolumeClaim:
      claimName: nfs
</code></pre>
<p>可以看到，Pod需要做的，就是在volumes字段里声明自己要使用的PVC名字。接下来，等这个Pod创建之后，kubelet就会把这个PVC所对应的PV，也就是一个NFS类型的Volume，挂载在这个Pod容器内的目录上。</p>
<p>不难看出，<strong>PVC和PV的设计，其实跟“面向对象”的思想完全一致。</strong></p>
<p>PVC可以理解为持久化存储的“接口”，它提供了对某种持久化存储的描述，但不提供具体的实现；而这个持久化存储的实现部分则由PV负责完成。</p>
<p>这样做的好处是，作为应用开发者，我们只需要跟PVC这个“接口”打交道，而不必关心具体的实现是NFS还是Ceph。毕竟这些存储相关的知识太专业了，应该交给专业的人去做。</p>
<p>而在上面的讲述中，其实还<span class="orange">有一个比较棘手的情况。</span></p>
<p>比如，你在创建Pod的时候，系统里并没有合适的PV跟它定义的PVC绑定，也就是说此时容器想要使用的Volume不存在。这时候，Pod的启动就会报错。</p>
<p>但是，过了一会儿，运维人员也发现了这个情况，所以他赶紧创建了一个对应的PV。这时候，我们当然希望Kubernetes能够再次完成PVC和PV的绑定操作，从而启动Pod。</p>
<p>所以在Kubernetes中，实际上存在着一个专门处理持久化存储的控制器，叫作Volume Controller。这个Volume Controller维护着多个控制循环，其中有一个循环，扮演的就是撮合PV和PVC的“红娘”的角色。它的名字叫作PersistentVolumeController。</p>
<p>PersistentVolumeController会不断地查看当前每一个PVC，是不是已经处于Bound（已绑定）状态。如果不是，那它就会遍历所有的、可用的PV，并尝试将其与这个“单身”的PVC进行绑定。这样，Kubernetes就可以保证用户提交的每一个PVC，只要有合适的PV出现，它就能够很快进入绑定状态，从而结束“单身”之旅。</p>
<p>而所谓将一个PV与PVC进行“绑定”，其实就是将这个PV对象的名字，填在了PVC对象的spec.volumeName字段上。所以，接下来Kubernetes只要获取到这个PVC对象，就一定能够找到它所绑定的PV。</p>
<p>那么，<span class="orange">这个PV对象，又是如何变成容器里的一个持久化存储的呢？</span></p>
<p>我在前面讲解容器基础的时候，已经为你详细剖析了容器Volume的挂载机制。用一句话总结，<strong>所谓容器的Volume，其实就是将一个宿主机上的目录，跟一个容器里的目录绑定挂载在了一起。</strong>（你可以借此机会，再回顾一下专栏的第8篇文章<a href="https://time.geekbang.org/column/article/18119">《白话容器基础（四）：重新认识Docker容器》</a>中的相关内容）</p>
<p><strong>而所谓的“持久化Volume”，指的就是这个宿主机上的目录，具备“持久性”</strong>。即：这个目录里面的内容，既不会因为容器的删除而被清理掉，也不会跟当前的宿主机绑定。这样，当容器被重启或者在其他节点上重建出来之后，它仍然能够通过挂载这个Volume，访问到这些内容。</p>
<p>显然，我们前面使用的hostPath和emptyDir类型的Volume并不具备这个特征：它们既有可能被kubelet清理掉，也不能被“迁移”到其他节点上。</p>
<p>所以，大多数情况下，持久化Volume的实现，往往依赖于一个远程存储服务，比如：远程文件存储（比如，NFS、GlusterFS）、远程块存储（比如，公有云提供的远程磁盘）等等。</p>
<p>而Kubernetes需要做的工作，就是使用这些存储服务，来为容器准备一个持久化的宿主机目录，以供将来进行绑定挂载时使用。而所谓“持久化”，指的是容器在这个目录里写入的文件，都会保存在远程存储中，从而使得这个目录具备了“持久性”。</p>
<p><strong>这个准备“持久化”宿主机目录的过程，我们可以形象地称为“两阶段处理”。</strong></p>
<p>接下来，我通过一个具体的例子为你说明。</p>
<p>当一个Pod调度到一个节点上之后，kubelet就要负责为这个Pod创建它的Volume目录。默认情况下，kubelet为Volume创建的目录是如下所示的一个宿主机上的路径：</p>
<pre><code>/var/lib/kubelet/pods/&lt;Pod的ID&gt;/volumes/kubernetes.io~&lt;Volume类型&gt;/&lt;Volume名字&gt;
</code></pre>
<p>接下来，kubelet要做的操作就取决于你的Volume类型了。</p>
<p>如果你的Volume类型是远程块存储，比如Google Cloud的Persistent Disk（GCE提供的远程磁盘服务），那么kubelet就需要先调用Goolge Cloud的API，将它所提供的Persistent Disk挂载到Pod所在的宿主机上。</p>
<blockquote>
<p>备注：你如果不太了解块存储的话，可以直接把它理解为：一块<strong>磁盘</strong>。</p>
</blockquote>
<p>这相当于执行：</p>
<pre><code>$ gcloud compute instances attach-disk &lt;虚拟机名字&gt; --disk &lt;远程磁盘名字&gt;
</code></pre>
<p>这一步<strong>为虚拟机挂载远程磁盘的操作，对应的正是“两阶段处理”的第一阶段。在Kubernetes中，我们把这个阶段称为Attach。</strong></p>
<p>Attach阶段完成后，为了能够使用这个远程磁盘，kubelet还要进行第二个操作，即：格式化这个磁盘设备，然后将它挂载到宿主机指定的挂载点上。不难理解，这个挂载点，正是我在前面反复提到的Volume的宿主机目录。所以，这一步相当于执行：</p>
<pre><code># 通过lsblk命令获取磁盘设备ID
$ sudo lsblk
# 格式化成ext4格式
$ sudo mkfs.ext4 -m 0 -F -E lazy_itable_init=0,lazy_journal_init=0,discard /dev/&lt;磁盘设备ID&gt;
# 挂载到挂载点
$ sudo mkdir -p /var/lib/kubelet/pods/&lt;Pod的ID&gt;/volumes/kubernetes.io~&lt;Volume类型&gt;/&lt;Volume名字&gt;
</code></pre>
<p>这个<strong>将磁盘设备格式化并挂载到Volume宿主机目录的操作，对应的正是“两阶段处理”的第二个阶段，我们一般称为：Mount。</strong></p>
<p>Mount阶段完成后，这个Volume的宿主机目录就是一个“持久化”的目录了，容器在它里面写入的内容，会保存在Google Cloud的远程磁盘中。</p>
<p>而如果你的Volume类型是远程文件存储（比如NFS）的话，kubelet的处理过程就会更简单一些。</p>
<p>因为在这种情况下，kubelet可以跳过“第一阶段”（Attach）的操作，这是因为一般来说，远程文件存储并没有一个“存储设备”需要挂载在宿主机上。</p>
<p>所以，kubelet会直接从“第二阶段”（Mount）开始准备宿主机上的Volume目录。</p>
<p>在这一步，kubelet需要作为client，将远端NFS服务器的目录（比如：“/”目录），挂载到Volume的宿主机目录上，即相当于执行如下所示的命令：</p>
<pre><code>$ mount -t nfs &lt;NFS服务器地址&gt;:/ /var/lib/kubelet/pods/&lt;Pod的ID&gt;/volumes/kubernetes.io~&lt;Volume类型&gt;/&lt;Volume名字&gt; 
</code></pre>
<p>通过这个挂载操作，Volume的宿主机目录就成为了一个远程NFS目录的挂载点，后面你在这个目录里写入的所有文件，都会被保存在远程NFS服务器上。所以，我们也就完成了对这个Volume宿主机目录的“持久化”。</p>
<p><strong>到这里，你可能会有疑问，Kubernetes又是如何定义和区分这两个阶段的呢？</strong></p>
<p>其实很简单，在具体的Volume插件的实现接口上，Kubernetes分别给这两个阶段提供了两种不同的参数列表：</p>
<ul>
<li>对于“第一阶段”（Attach），Kubernetes提供的可用参数是nodeName，即宿主机的名字。</li>
<li>而对于“第二阶段”（Mount），Kubernetes提供的可用参数是dir，即Volume的宿主机目录。</li>
</ul>
<p>所以，作为一个存储插件，你只需要根据自己的需求进行选择和实现即可。在后面关于编写存储插件的文章中，我会对这个过程做深入讲解。</p>
<p>而经过了“两阶段处理”，我们就得到了一个“持久化”的Volume宿主机目录。所以，接下来，kubelet只要把这个Volume目录通过CRI里的Mounts参数，传递给Docker，然后就可以为Pod里的容器挂载这个“持久化”的Volume了。其实，这一步相当于执行了如下所示的命令：</p>
<pre><code>$ docker run -v /var/lib/kubelet/pods/&lt;Pod的ID&gt;/volumes/kubernetes.io~&lt;Volume类型&gt;/&lt;Volume名字&gt;:/&lt;容器内的目标目录&gt; 我的镜像 ...
</code></pre>
<p>以上，就是Kubernetes处理PV的具体原理了。</p>
<blockquote>
<p>备注：对应地，在删除一个PV的时候，Kubernetes也需要Unmount和Dettach两个阶段来处理。这个过程我就不再详细介绍了，执行“反向操作”即可。</p>
</blockquote>
<p>实际上，你可能已经发现，这个PV的处理流程似乎跟Pod以及容器的启动流程没有太多的耦合，只要kubelet在向Docker发起CRI请求之前，确保“持久化”的宿主机目录已经处理完毕即可。</p>
<p>所以，在Kubernetes中，上述<strong>关于PV的“两阶段处理”流程，是靠独立于kubelet主控制循环（Kubelet Sync Loop）之外的两个控制循环来实现的。</strong></p>
<p>其中，“第一阶段”的Attach（以及Dettach）操作，是由Volume Controller负责维护的，这个控制循环的名字叫作：<strong>AttachDetachController</strong>。而它的作用，就是不断地检查每一个Pod对应的PV，和这个Pod所在宿主机之间挂载情况。从而决定，是否需要对这个PV进行Attach（或者Dettach）操作。</p>
<p>需要注意，作为一个Kubernetes内置的控制器，Volume Controller自然是kube-controller-manager的一部分。所以，AttachDetachController也一定是运行在Master节点上的。当然，Attach操作只需要调用公有云或者具体存储项目的API，并不需要在具体的宿主机上执行操作，所以这个设计没有任何问题。</p>
<p>而“第二阶段”的Mount（以及Unmount）操作，必须发生在Pod对应的宿主机上，所以它必须是kubelet组件的一部分。这个控制循环的名字，叫作：<strong>VolumeManagerReconciler</strong>，它运行起来之后，是一个独立于kubelet主循环的Goroutine。</p>
<p>通过这样将Volume的处理同kubelet的主循环解耦，Kubernetes就避免了这些耗时的远程挂载操作拖慢kubelet的主控制循环，进而导致Pod的创建效率大幅下降的问题。实际上，<strong>kubelet的一个主要设计原则，就是它的主控制循环绝对不可以被block</strong>。这个思想，我在后续的讲述容器运行时的时候还会提到。</p>
<p><span class="orange">在了解了Kubernetes的Volume处理机制之后，我再来为你介绍这个体系里最后一个重要概念：StorageClass。</span></p>
<p>我在前面介绍PV和PVC的时候，曾经提到过，PV这个对象的创建，是由运维人员完成的。但是，在大规模的生产环境里，这其实是一个非常麻烦的工作。</p>
<p>这是因为，一个大规模的Kubernetes集群里很可能有成千上万个PVC，这就意味着运维人员必须得事先创建出成千上万个PV。更麻烦的是，随着新的PVC不断被提交，运维人员就不得不继续添加新的、能满足条件的PV，否则新的Pod就会因为PVC绑定不到PV而失败。在实际操作中，这几乎没办法靠人工做到。</p>
<p>所以，Kubernetes为我们提供了一套可以自动创建PV的机制，即：Dynamic Provisioning。</p>
<p>相比之下，前面人工管理PV的方式就叫作Static Provisioning。</p>
<p>Dynamic Provisioning机制工作的核心，在于一个名叫StorageClass的API对象。</p>
<p><strong>而StorageClass对象的作用，其实就是创建PV的模板。</strong></p>
<p>具体地说，StorageClass对象会定义如下两个部分内容：</p>
<ul>
<li>第一，PV的属性。比如，存储类型、Volume的大小等等。</li>
<li>第二，创建这种PV需要用到的存储插件。比如，Ceph等等。</li>
</ul>
<p>有了这样两个信息之后，Kubernetes就能够根据用户提交的PVC，找到一个对应的StorageClass了。然后，Kubernetes就会调用该StorageClass声明的存储插件，创建出需要的PV。</p>
<p>举个例子，假如我们的Volume的类型是GCE的Persistent Disk的话，运维人员就需要定义一个如下所示的StorageClass：</p>
<pre><code>apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: block-service
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
</code></pre>
<p>在这个YAML文件里，我们定义了一个名叫block-service的StorageClass。</p>
<p>这个StorageClass的provisioner字段的值是：<code>kubernetes.io/gce-pd</code>，这正是Kubernetes内置的GCE PD存储插件的名字。</p>
<p>而这个StorageClass的parameters字段，就是PV的参数。比如：上面例子里的type=pd-ssd，指的是这个PV的类型是“SSD格式的GCE远程磁盘”。</p>
<p>需要注意的是，由于需要使用GCE Persistent Disk，上面这个例子只有在GCE提供的Kubernetes服务里才能实践。如果你想使用我们之前部署在本地的Kubernetes集群以及Rook存储服务的话，你的StorageClass需要使用如下所示的YAML文件来定义：</p>
<pre><code>apiVersion: ceph.rook.io/v1beta1
kind: Pool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  replicated:
    size: 3
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: block-service
provisioner: ceph.rook.io/block
parameters:
  pool: replicapool
  #The value of &quot;clusterNamespace&quot; MUST be the same as the one in which your rook cluster exist
  clusterNamespace: rook-ceph
</code></pre>
<p>在这个YAML文件中，我们定义的还是一个名叫block-service的StorageClass，只不过它声明使的存储插件是由Rook项目。</p>
<p>有了StorageClass的YAML文件之后，运维人员就可以在Kubernetes里创建这个StorageClass了：</p>
<pre><code>$ kubectl create -f sc.yaml
</code></pre>
<p>这时候，作为应用开发者，我们只需要在PVC里指定要使用的StorageClass名字即可，如下所示：</p>
<pre><code>apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: claim1
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: block-service
  resources:
    requests:
      storage: 30Gi
</code></pre>
<p>可以看到，我们在这个PVC里添加了一个叫作storageClassName的字段，用于指定该PVC所要使用的StorageClass的名字是：block-service。</p>
<p>以Google Cloud为例。</p>
<p>当我们通过kubectl create创建上述PVC对象之后，Kubernetes就会调用Google Cloud的API，创建出一块SSD格式的Persistent Disk。然后，再使用这个Persistent Disk的信息，自动创建出一个对应的PV对象。</p>
<p>我们可以一起来实践一下这个过程（如果使用Rook的话下面的流程也是一样的，只不过Rook创建出的是Ceph类型的PV）：</p>
<pre><code>$ kubectl create -f pvc.yaml
</code></pre>
<p>可以看到，我们创建的PVC会绑定一个Kubernetes自动创建的PV，如下所示：</p>
<pre><code>$ kubectl describe pvc claim1
Name:           claim1
Namespace:      default
StorageClass:   block-service
Status:         Bound
Volume:         pvc-e5578707-c626-11e6-baf6-08002729a32b
Labels:         &lt;none&gt;
Capacity:       30Gi
Access Modes:   RWO
No Events.
</code></pre>
<p>而且，通过查看这个自动创建的PV的属性，你就可以看到它跟我们在PVC里声明的存储的属性是一致的，如下所示：</p>
<pre><code>$ kubectl describe pv pvc-e5578707-c626-11e6-baf6-08002729a32b
Name:            pvc-e5578707-c626-11e6-baf6-08002729a32b
Labels:          &lt;none&gt;
StorageClass:    block-service
Status:          Bound
Claim:           default/claim1
Reclaim Policy:  Delete
Access Modes:    RWO
Capacity:        30Gi
...
No events.
</code></pre>
<p>此外，你还可以看到，这个自动创建出来的PV的StorageClass字段的值，也是block-service。<strong>这是因为，Kubernetes只会将StorageClass相同的PVC和PV绑定起来。</strong></p>
<p>有了Dynamic Provisioning机制，运维人员只需要在Kubernetes集群里创建出数量有限的StorageClass对象就可以了。这就好比，运维人员在Kubernetes集群里创建出了各种各样的PV模板。这时候，当开发人员提交了包含StorageClass字段的PVC之后，Kubernetes就会根据这个StorageClass创建出对应的PV。</p>
<blockquote>
<p><a href="https://kubernetes.io/docs/concepts/storage/storage-classes/#provisioner">Kubernetes的官方文档</a>里已经列出了默认支持Dynamic Provisioning的内置存储插件。而对于不在文档里的插件，比如NFS，或者其他非内置存储插件，你其实可以通过<a href="https://github.com/kubernetes-incubator/external-storage">kubernetes-incubator/external-storage</a>这个库来自己编写一个外部插件完成这个工作。像我们之前部署的Rook，已经内置了external-storage的实现，所以Rook是完全支持Dynamic Provisioning特性的。</p>
</blockquote>
<p>需要注意的是，<strong>StorageClass并不是专门为了Dynamic Provisioning而设计的。</strong></p>
<p>比如，在本篇一开始的例子里，我在PV和PVC里都声明了storageClassName=manual。而我的集群里，实际上并没有一个名叫manual的StorageClass对象。这完全没有问题，这个时候Kubernetes进行的是Static Provisioning，但在做绑定决策的时候，它依然会考虑PV和PVC的StorageClass定义。</p>
<p>而这么做的好处也很明显：这个PVC和PV的绑定关系，就完全在我自己的掌控之中。</p>
<p>这里，你可能会有疑问，我在之前讲解StatefulSet存储状态的例子时，好像并没有声明StorageClass啊？</p>
<p>实际上，如果你的集群已经开启了名叫DefaultStorageClass的Admission Plugin，它就会为PVC和PV自动添加一个默认的StorageClass；<strong>否则，PVC的storageClassName的值就是“”，这也意味着它只能够跟storageClassName也是“”的PV进行绑定。</strong></p>
<h2>总结</h2>
<p>在今天的分享中，我为你详细解释了PVC和PV的设计与实现原理，并为你阐述了StorageClass到底是干什么用的。这些概念之间的关系，可以用如下所示的一幅示意图描述：</p>
<p><img src="https://static001.geekbang.org/resource/image/e8/d9/e8b2586e4e14eb54adf8ff95c5c18cd9.png?wh=968*679" alt="" /><br />
从图中我们可以看到，在这个体系中：</p>
<ul>
<li>
<p>PVC描述的，是Pod想要使用的持久化存储的属性，比如存储的大小、读写权限等。</p>
</li>
<li>
<p>PV描述的，则是一个具体的Volume的属性，比如Volume的类型、挂载目录、远程存储服务器地址等。</p>
</li>
<li>
<p>而StorageClass的作用，则是充当PV的模板。并且，只有同属于一个StorageClass的PV和PVC，才可以绑定在一起。</p>
</li>
</ul>
<p>当然，StorageClass的另一个重要作用，是指定PV的Provisioner（存储插件）。这时候，如果你的存储插件支持Dynamic Provisioning的话，Kubernetes就可以自动为你创建PV了。</p>
<p>基于上述讲述，为了统一概念和方便叙述，在本专栏中，我以后凡是提到“Volume”，指的就是一个远程存储服务挂载在宿主机上的持久化目录；而“PV”，指的是这个Volume在Kubernetes里的API对象。</p>
<p>需要注意的是，这套容器持久化存储体系，完全是Kubernetes项目自己负责管理的，并不依赖于docker volume命令和Docker的存储插件。当然，这套体系本身就比docker volume命令的诞生时间还要早得多。</p>
<h2>思考题</h2>
<p>在了解了PV、PVC的设计和实现原理之后，你是否依然觉得它有“过度设计”的嫌疑？或者，你是否有更加简单、足以解决你90%需求的Volume的用法？</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4c/49/d21c134f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>swordholder</span>
  </div>
  <div class="_2_QraFYR_0">容器持久化存储涉及的概念比较多，试着总结一下整体流程。<br><br>用户提交请求创建pod，Kubernetes发现这个pod声明使用了PVC，那就靠PersistentVolumeController帮它找一个PV配对。<br><br>没有现成的PV，就去找对应的StorageClass，帮它新创建一个PV，然后和PVC完成绑定。<br><br>新创建的PV，还只是一个API 对象，需要经过“两阶段处理”变成宿主机上的“持久化 Volume”才真正有用：<br>第一阶段由运行在master上的AttachDetachController负责，为这个PV完成 Attach 操作，为宿主机挂载远程磁盘；<br>第二阶段是运行在每个节点上kubelet组件的内部，把第一步attach的远程磁盘 mount 到宿主机目录。这个控制循环叫VolumeManagerReconciler，运行在独立的Goroutine，不会阻塞kubelet主循环。<br><br>完成这两步，PV对应的“持久化 Volume”就准备好了，POD可以正常启动，将“持久化 Volume”挂载在容器内指定的路径。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 总结的好！后面还有更多概念！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-26 17:59:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLbswPFgMWmE28MQvjBHpAKg2Ny426dqRCgzp0ibyh0rr3nySEF621bWicySpAjATVEVyoibqloPqeLw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_e2f5e1</span>
  </div>
  <div class="_2_QraFYR_0">老师，如果我原先存储上就有数据需要挂载进去，那格式化操作岂不是不能满足我的需求？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 格式化前肯定要检查啊，只有raw格式才需要格式化</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-09 22:52:09</div>
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
  <div class="_2_QraFYR_0">张老师，问一个比较空泛的问题。您之前是做paas平台的，今后的pass平台的发展方向是什么呢？当前做paas平台，最大的阻碍是什么？最大的价值又是什么呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: paas最后就是各家基于kubernetes DIY。这样多好。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-27 15:32:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/f0/9e/cf6570f7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>耀</span>
  </div>
  <div class="_2_QraFYR_0">没有过度设计，如果没有PVC，那么用户声明就会有涉及到具体的存储类型；存储类型一旦变化了所有的微服务都要跟着变化，所以PVC和PV要分离。如果没有storageclass,那么PVC和PV的绑定就需要完全有人工去指定，这将会成为整个集群最重复而低效的事情之一，所以这种设计是刚好的设计。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-29 23:08:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>tuxknight</span>
  </div>
  <div class="_2_QraFYR_0">在公有云上使用 PV&#47;PVC 有个很重要的限制：块存储设备必须和实例在同一个可用区。在 Pod 没被创建的时候是不确定会被调度到哪个可用区，从而无法动态的创建出PV。这种问题要怎么处理？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: scheduler里的volumezonechecker规则了解一下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-28 22:46:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/84/c1/dfcad82a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Acter</span>
  </div>
  <div class="_2_QraFYR_0">“所谓将一个 PV 与 PVC 进行“绑定”，其实就是将这个 PV 对象的名字，填在了 PVC 对象的 spec.volumeName 字段上。” <br>请问老师为什么在pvc的yaml文件中看不到这个字段呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 看它的api types定义</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-29 13:22:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/6c/ea/ce9854a5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>坤</span>
  </div>
  <div class="_2_QraFYR_0">如果采用ceph  rbd StorageClass，Pod所在的node宕机后，在调度到另外一台Node上，这个过程中，k8s是会新node上重新创建PV吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个取决于你的存储插件是什么类型的。支持 auto provisioning的就可以自动创建。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-04 12:10:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/7d/75/c812597b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>yuliz</span>
  </div>
  <div class="_2_QraFYR_0">你好，我想请教下实际中的疑问点，如果我使用NFS作为共享存储，两个集群中的PV绑定NFS的同一目录，且这两个PV被pvc绑定，最终pod绑定pvc,当第二个pod绑定时会格式化nfs的目录，导致之前的pod数据丢失么？两个集群的pv能共用一个nfs目录和同一rbd么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: volumeMounts有个subpath字段了解一下。可以，但不建议。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-30 08:56:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/99/66/e283b8b2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>GR</span>
  </div>
  <div class="_2_QraFYR_0">一个pv可以对应几个pvc，一对一吗？<br>可以创建一个大的pv然后每个应用有自己的pvc，存储路径通过subpath来区分，是否可行呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-26 07:50:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/f5/a5/f8210f04.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>夕月</span>
  </div>
  <div class="_2_QraFYR_0">所谓将一个 PV 与 PVC 进行“绑定”，其实就是将这个PV 对象的名字，填在了 PVC 对象的 spec.volumeName 字段上，这个好像在yaml文件里没有提现啊，只有storageClassName是一样的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你把api对象和yaml文件搞混了？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-25 11:11:28</div>
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
  <div class="_2_QraFYR_0">请问<br>1. 同一集群的多个pod可否同时挂载同一个pv的同一个subpath<br>2. 如果pv写满了如何扩容</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 要看存储插件是否支持。pv只是逻辑概念，你的数据是在远程存储里的，所以resize是存储插件的功能。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-25 23:08:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/eb/77/ffd16123.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>重洋</span>
  </div>
  <div class="_2_QraFYR_0">持久化存储的控制器 Volume Controller，kube-controller-manager 的一部分，运行在 master节点。其中包括控制循环 PersistentVolumeController、AttachDetachController。<br><br>PV、PVC匹配控制器 PersistentVolumeController ：执行控制循环，为每一个 PVC 遍历挑选可用的的 PV 进行绑定。<br>绑定：将 PV 对象的名字，填在 PVC 对象的 spec.volumeName 字段上。<br>匹配条件：<br>1.spec字段符合要求。<br>2.storageClassName一致。<br><br>Volume 类型是远程存储时：<br>一、远程存储挂载至宿主机(PV的准备流程)，分为两阶段：<br>第一阶段（Attach）（挂载磁盘）：AttachDetachController 不断地检查每一个 Pod 对应的 PV 和这个 Pod 所在宿主机之间挂载情况。从而决定，是否需要对这个 PV 进行 Attach（或者 Dettach）操作，为虚拟机挂载远程磁盘。Kubernetes 提供的可用参数是 nodeName，即宿主机的名字。<br>第二阶段（Mount）（mount至宿主机）：VolumeManagerReconciler，执行mount操作（必须发生再Pod宿主机上，所以是 kubelet 组件的一部分），将磁盘设备格式化并挂载到 Volume 宿主机目录。Kubernetes 提供的可用参数是 dir，即 Volume 的宿主机目录。<br>$ mount -t nfs &lt;NFS服务器地址&gt;:&#47; &#47;var&#47;lib&#47;kubelet&#47;pods&#47;&lt;Pod的ID&gt;&#47;volumes&#47;kubernetes.io~&lt;Volume类型&gt;&#47;&lt;Volume名字&gt; <br><br>二、Pod 与持久化Volume宿主机目录挂载：<br>kubelet 为 Volume 创建的默认宿主机目录：&#47;var&#47;lib&#47;kubelet&#47;pods&#47;&lt;Pod的ID&gt;&#47;volumes&#47;kubernetes.io~&lt;Volume类型&gt;&#47;&lt;Volume名字&gt;<br>kubelet 把 Volume 目录通过 CRI 里的 Mounts 参数，传递给 Docker，为 Pod 里的容器挂载这个“持久化”的 Volume 。<br>$ docker run -v &#47;var&#47;lib&#47;kubelet&#47;pods&#47;&lt;Pod的ID&gt;&#47;volumes&#47;kubernetes.io~&lt;Volume类型&gt;&#47;&lt;Volume名字&gt;:&#47;&lt;容器内的目标目录&gt; 我的镜像 ...<br><br><br>Static Provisioning：手动创建 PV 与 PVC，StorageClassName 仅作为匹配字段。<br>Dynamic Provisioning：通过 StorageClass 自动创建 PV、自动绑定 PV 与 PVC。<br>StorageClass 定义插件、PV 属性，k8s根据 PVC 寻找 StorageClass，StorageClass 创建 PV。<br>PVC——StorageClass——PV。<br><br><br>Local Persistent Volume：Volume直接使用本地磁盘。<br>优点：本地磁盘读写性能更好。<br>缺点：宕机数据丢失，需要恢复机制与定期备份。<br><br>先准备好本地磁盘，然后延迟绑定。<br>延迟绑定：StorageClass volumeBindingMode: WaitForFirstConsumer，将 PV、PVC 的绑定推迟到 Pod 调度的时候，从而保证绑定结果不会影响 Pod 的正常调度。<br>1.PersistentVolumeController 遍历发现通过 StorageClass 关联的可以与 PVC 绑定的 PV；<br>2.第一个声明使用该 PVC 的 Pod 出现在调度器之后，调度器综合考虑这个 PVC 跟哪个 PV 绑定。<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-02 15:15:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/b4/0c/93fd5c51.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jkmzg</span>
  </div>
  <div class="_2_QraFYR_0">请问下从同一个pod spec 创建出来的不同pod中，pvc相同，会不会冲突？k8s的机制是什么呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 带id，不冲突</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-13 21:05:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/a5/ff/6201122c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_89bbab</span>
  </div>
  <div class="_2_QraFYR_0"> kubectl describe persistentvolumeclaim&#47;claim1<br>Name:          claim1<br>Namespace:     default<br>StorageClass:  block-service<br>Status:        Pending<br>Volume:        <br>Labels:        &lt;none&gt;<br>Annotations:   .......<br>Events:<br>  Type     Reason              Age                 From                                                                                         Message<br>  ----     ------              ----                ----                                                                                         -------<br>  Warning  ProvisioningFailed  10m (x13 over 23m)  ceph.rook.io&#47;block rook-ceph-operator-5bfbf654db-ncgdf 97142e78-de86-11e8-a7d1-e6678be2ea25  Failed to provision volume with StorageClass &quot;block-service&quot;: Failed to create rook block image replicapool&#47;pvc-5b68d13b-e501-11e8-8b01-00163e0cf240: failed to create image pvc-5b68d13b-e501-11e8-8b01-00163e0cf240 in pool replicapool of size 2147483648: Failed to complete &#39;&#39;: exit status 2. rbd: error opening pool &#39;replicapool&#39;: (2) No such file or directory<br>. output:<br>  Normal  Provisioning          5m (x15 over 23m)   ceph.rook.io&#47;block rook-ceph-operator-5bfbf654db-ncgdf 97142e78-de86-11e8-a7d1-e6678be2ea25  External provisioner is provisioning volume for claim &quot;default&#47;claim1&quot;<br>  Normal  ExternalProvisioning  2m (x331 over 23m)  persistentvolume-controller                                                                  waiting for a volume to be created, either by external provisioner &quot;ceph.rook.io&#47;block&quot; or manually created by system administrator<br><br>老师，这个是什么原因导致的？ 没有这个文件或者目录?</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-11 00:32:10</div>
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
  <div class="_2_QraFYR_0">老师的问题的思考，90%都是动态申请存储的，所以我觉得pv和pvc都去掉，只有storage class和必要的参数（空间大小和读写属性 ）放在pod中即可</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-28 20:44:34</div>
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
  <div class="_2_QraFYR_0">请问，现在的NFS也是有storageclass也是可以动态配置pv的啊，但是在官方，体现的是没有的啊？这个是第三方开发的么？provisioner: fuseim.pri&#47;ifs</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是啊，都是第三方的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-26 17:01:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q3auHgzwzM6GcSnUMzR0H9haiaAxssjibGLQMLAsPKonh50g9W2Iz38LcZNGH39HPaANLtovXTp1YvsINIZoH6F0iaSGuxJMXZS/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_10d981</span>
  </div>
  <div class="_2_QraFYR_0">其实老师最后的提问仅仅是让大家深入思考的，没有固定的解答。我尝试理解：这种分层的概念其实在高端存储用的很多，比如IBM的SVC,HDS的LUSE，EMC的TIRE，很多存储厂家的THIN PROVISION都是分层的概念，中端的NETAPP也是类似的技术（NETAPP横着切 一刀，竖着再切一刀，IBM DS7000，,8000系列也是这样，HP 3PAR也是彻底分层），当然，分层的时候要考虑SAN的结构和MULTIPATH。分层的好处很多，不仅仅只是解耦，也是对磁盘的利用率，动态调节，性能诊断更灵活。还有，HDS,EMC等厂家可以详细的性能诊断，因为高端存储都有自己的高速缓存（32G-128G，还有CROSS BAR技术），很多存储的高级功能必须是要分层才能实现的。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-29 00:00:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/96/ec/5c53f9e5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小熹</span>
  </div>
  <div class="_2_QraFYR_0">用CSI标准实现第三方存储，把存储卷的管理全部托付给第三方，就不用自己纠结pv pvc的概念了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-20 09:35:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/a5/ff/6201122c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_89bbab</span>
  </div>
  <div class="_2_QraFYR_0">有人执行pvc有遇到这样的错误吗？<br>Failed to provision volume with StorageClass &quot;rook-ceph-block&quot;: Failed to create rook block image replicapool&#47;pvc-0574eb19-e58c-11e8-8b01-00163e0cf240: failed to create image pvc-0574eb19-e58c-11e8-8b01-00163e0cf240 in pool replicapool of size 2147483648: Failed to complete &#39;&#39;: exit status 2. rbd: error opening pool &#39;replicapool&#39;: (2) No such file or directory<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-11 17:17:41</div>
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
  <div class="_2_QraFYR_0">@GR 是一一对应的关系，可以创建一个大的pvc共用，用子目录区别开。前提是在一个namespace下。<br>也可以开发插件，支持动态创建pv</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-26 09:14:12</div>
  </div>
</div>
</div>
</li>
</ul>