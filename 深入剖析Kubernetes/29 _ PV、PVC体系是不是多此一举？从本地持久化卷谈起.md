<audio title="29 _ PV、PVC体系是不是多此一举？从本地持久化卷谈起" src="https://static001.geekbang.org/resource/audio/b2/02/b2ecc008949240aaeffe3bafe427d102.mp3" controls="controls"></audio> 
<p>你好，我是张磊。今天我和你分享的主题是：PV、PVC体系是不是多此一举？从本地持久化卷谈起。</p><p>在上一篇文章中，我为你详细讲解了PV、PVC持久化存储体系在Kubernetes项目中的设计和实现原理。而在文章最后的思考题中，我为你留下了这样一个讨论话题：像PV、PVC这样的用法，是不是有“过度设计”的嫌疑？</p><p>比如，我们公司的运维人员可以像往常一样维护一套NFS或者Ceph服务器，根本不必学习Kubernetes。而开发人员，则完全可以靠“复制粘贴”的方式，在Pod的YAML文件里填上Volumes字段，而不需要去使用PV和PVC。</p><p>实际上，如果只是为了职责划分，PV、PVC体系确实不见得比直接在Pod里声明Volumes字段有什么优势。</p><p>不过，你有没有想过这样一个问题，如果<a href="https://kubernetes.io/docs/concepts/storage/persistent-volumes/#types-of-persistent-volumes">Kubernetes内置的20种持久化数据卷实现</a>，都没办法满足你的容器存储需求时，该怎么办？</p><p>这个情况乍一听起来有点不可思议。但实际上，凡是鼓捣过开源项目的读者应该都有所体会，“不能用”“不好用”“需要定制开发”，这才是落地开源基础设施项目的三大常态。</p><p>而在持久化存储领域，用户呼声最高的定制化需求，莫过于支持“本地”持久化存储了。</p><!-- [[[read_end]]] --><p>也就是说，用户希望Kubernetes能够直接使用宿主机上的本地磁盘目录，而不依赖于远程存储服务，来提供“持久化”的容器Volume。</p><p>这样做的好处很明显，由于这个Volume直接使用的是本地磁盘，尤其是SSD盘，它的读写性能相比于大多数远程存储来说，要好得多。这个需求对本地物理服务器部署的私有Kubernetes集群来说，非常常见。</p><p>所以，Kubernetes在v1.10之后，就逐渐依靠PV、PVC体系实现了这个特性。这个特性的名字叫作：Local Persistent Volume。</p><p>不过，首先需要明确的是，<strong>Local Persistent Volume并不适用于所有应用</strong>。事实上，它的适用范围非常固定，比如：高优先级的系统应用，需要在多个不同节点上存储数据，并且对I/O较为敏感。典型的应用包括：分布式数据存储比如MongoDB、Cassandra等，分布式文件系统比如GlusterFS、Ceph等，以及需要在本地磁盘上进行大量数据缓存的分布式应用。</p><p>其次，相比于正常的PV，一旦这些节点宕机且不能恢复时，Local Persistent Volume的数据就可能丢失。这就要求<strong>使用Local Persistent Volume的应用必须具备数据备份和恢复的能力</strong>，允许你把这些数据定时备份在其他位置。</p><p>接下来，我就为你深入讲解一下这个特性。</p><p>不难想象，<span class="orange">Local Persistent Volume的设计，主要面临两个难点。</span></p><p><strong>第一个难点在于</strong>：如何把本地磁盘抽象成PV。</p><p>可能你会说，Local Persistent Volume，不就等同于hostPath加NodeAffinity吗？</p><p>比如，一个Pod可以声明使用类型为Local的PV，而这个PV其实就是一个hostPath类型的Volume。如果这个hostPath对应的目录，已经在节点A上被事先创建好了。那么，我只需要再给这个Pod加上一个nodeAffinity=nodeA，不就可以使用这个Volume了吗？</p><p>事实上，<strong>你绝不应该把一个宿主机上的目录当作PV使用</strong>。这是因为，这种本地目录的存储行为完全不可控，它所在的磁盘随时都可能被应用写满，甚至造成整个宿主机宕机。而且，不同的本地目录之间也缺乏哪怕最基础的I/O隔离机制。</p><p>所以，一个Local Persistent Volume对应的存储介质，一定是一块额外挂载在宿主机的磁盘或者块设备（“额外”的意思是，它不应该是宿主机根目录所使用的主硬盘）。这个原则，我们可以称为“<strong>一个PV一块盘</strong>”。</p><p><strong>第二个难点在于</strong>：调度器如何保证Pod始终能被正确地调度到它所请求的Local Persistent Volume所在的节点上呢？</p><p>造成这个问题的原因在于，对于常规的PV来说，Kubernetes都是先调度Pod到某个节点上，然后，再通过“两阶段处理”来“持久化”这台机器上的Volume目录，进而完成Volume目录与容器的绑定挂载。</p><p>可是，对于Local PV来说，节点上可供使用的磁盘（或者块设备），必须是运维人员提前准备好的。它们在不同节点上的挂载情况可以完全不同，甚至有的节点可以没这种磁盘。</p><p>所以，这时候，调度器就必须能够知道所有节点与Local Persistent Volume对应的磁盘的关联关系，然后根据这个信息来调度Pod。</p><p>这个原则，我们可以称为“<strong>在调度的时候考虑Volume分布</strong>”。在Kubernetes的调度器里，有一个叫作VolumeBindingChecker的过滤条件专门负责这个事情。在Kubernetes v1.11中，这个过滤条件已经默认开启了。</p><p>基于上述讲述，<span class="orange">在开始使用Local Persistent Volume之前，你首先需要在集群里配置好磁盘或者块设备。</span>在公有云上，这个操作等同于给虚拟机额外挂载一个磁盘，比如GCE的Local SSD类型的磁盘就是一个典型例子。</p><p>而在我们部署的私有环境中，你有两种办法来完成这个步骤。</p><ul>
<li>第一种，当然就是给你的宿主机挂载并格式化一个可用的本地磁盘，这也是最常规的操作；</li>
<li>第二种，对于实验环境，你其实可以在宿主机上挂载几个RAM Disk（内存盘）来模拟本地磁盘。</li>
</ul><p>接下来，我会使用第二种方法，在我们之前部署的Kubernetes集群上进行实践。</p><p><strong>首先</strong>，在名叫node-1的宿主机上创建一个挂载点，比如/mnt/disks；<strong>然后</strong>，用几个RAM Disk来模拟本地磁盘，如下所示：</p><pre><code># 在node-1上执行
$ mkdir /mnt/disks
$ for vol in vol1 vol2 vol3; do
    mkdir /mnt/disks/$vol
    mount -t tmpfs $vol /mnt/disks/$vol
done
</code></pre><p>需要注意的是，如果你希望其他节点也能支持Local Persistent Volume的话，那就需要为它们也执行上述操作，并且确保这些磁盘的名字（vol1、vol2等）都不重复。</p><p><span class="orange">接下来，我们就可以为这些本地磁盘定义对应的PV了</span>，如下所示：</p><pre><code>apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-pv
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /mnt/disks/vol1
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - node-1
</code></pre><p>可以看到，这个PV的定义里：local字段，指定了它是一个Local Persistent Volume；而path字段，指定的正是这个PV对应的本地磁盘的路径，即：/mnt/disks/vol1。</p><p>当然了，这也就意味着如果Pod要想使用这个PV，那它就必须运行在node-1上。所以，在这个PV的定义里，需要有一个nodeAffinity字段指定node-1这个节点的名字。这样，调度器在调度Pod的时候，就能够知道一个PV与节点的对应关系，从而做出正确的选择。<strong>这正是Kubernetes实现“在调度的时候就考虑Volume分布”的主要方法。</strong></p><p><strong>接下来</strong>，我们就可以使用kubect create来创建这个PV，如下所示：</p><pre><code>$ kubectl create -f local-pv.yaml 
persistentvolume/example-pv created

$ kubectl get pv
NAME         CAPACITY   ACCESS MODES   RECLAIM POLICY  STATUS      CLAIM             STORAGECLASS    REASON    AGE
example-pv   5Gi        RWO            Delete           Available                     local-storage             16s
</code></pre><p>可以看到，这个PV创建后，进入了Available（可用）状态。</p><p>而正如我在上一篇文章里所建议的那样，使用PV和PVC的最佳实践，是你要创建一个StorageClass来描述这个PV，如下所示：</p><pre><code>kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
</code></pre><p>这个StorageClass的名字，叫作local-storage。需要注意的是，在它的provisioner字段，我们指定的是no-provisioner。这是因为Local Persistent Volume目前尚不支持Dynamic Provisioning，所以它没办法在用户创建PVC的时候，就自动创建出对应的PV。也就是说，我们前面创建PV的操作，是不可以省略的。</p><p>与此同时，这个StorageClass还定义了一个volumeBindingMode=WaitForFirstConsumer的属性。它是Local Persistent Volume里一个非常重要的特性，即：<strong>延迟绑定</strong>。</p><p>我们知道，当你提交了PV和PVC的YAML文件之后，Kubernetes就会根据它们俩的属性，以及它们指定的StorageClass来进行绑定。只有绑定成功后，Pod才能通过声明这个PVC来使用对应的PV。</p><p>可是，如果你使用的是Local Persistent Volume的话，就会发现，这个流程根本行不通。</p><p>比如，现在你有一个Pod，它声明使用的PVC叫作pvc-1。并且，我们规定，这个Pod只能运行在node-2上。</p><p>而在Kubernetes集群中，有两个属性（比如：大小、读写权限）相同的Local类型的PV。</p><p>其中，第一个PV的名字叫作pv-1，它对应的磁盘所在的节点是node-1。而第二个PV的名字叫作pv-2，它对应的磁盘所在的节点是node-2。</p><p>假设现在，Kubernetes的Volume控制循环里，首先检查到了pvc-1和pv-1的属性是匹配的，于是就将它们俩绑定在一起。</p><p>然后，你用kubectl create创建了这个Pod。</p><p>这时候，问题就出现了。</p><p>调度器看到，这个Pod所声明的pvc-1已经绑定了pv-1，而pv-1所在的节点是node-1，根据“调度器必须在调度的时候考虑Volume分布”的原则，这个Pod自然会被调度到node-1上。</p><p>可是，我们前面已经规定过，这个Pod根本不允许运行在node-1上。所以。最后的结果就是，这个Pod的调度必然会失败。</p><p><strong>这就是为什么，在使用Local Persistent Volume的时候，我们必须想办法推迟这个“绑定”操作。</strong></p><p>那么，具体推迟到什么时候呢？</p><p><strong>答案是：推迟到调度的时候。</strong></p><p>所以说，StorageClass里的volumeBindingMode=WaitForFirstConsumer的含义，就是告诉Kubernetes里的Volume控制循环（“红娘”）：虽然你已经发现这个StorageClass关联的PVC与PV可以绑定在一起，但请不要现在就执行绑定操作（即：设置PVC的VolumeName字段）。</p><p>而要等到第一个声明使用该PVC的Pod出现在调度器之后，调度器再综合考虑所有的调度规则，当然也包括每个PV所在的节点位置，来统一决定，这个Pod声明的PVC，到底应该跟哪个PV进行绑定。</p><p>这样，在上面的例子里，由于这个Pod不允许运行在pv-1所在的节点node-1，所以它的PVC最后会跟pv-2绑定，并且Pod也会被调度到node-2上。</p><p>所以，通过这个延迟绑定机制，原本实时发生的PVC和PV的绑定过程，就被延迟到了Pod第一次调度的时候在调度器中进行，从而保证了这个<strong>绑定结果不会影响Pod的正常调度</strong>。</p><p>当然，在具体实现中，调度器实际上维护了一个与Volume Controller类似的控制循环，专门负责为那些声明了“延迟绑定”的PV和PVC进行绑定工作。</p><p>通过这样的设计，这个额外的绑定操作，并不会拖慢调度器的性能。而当一个Pod的PVC尚未完成绑定时，调度器也不会等待，而是会直接把这个Pod重新放回到待调度队列，等到下一个调度周期再做处理。</p><p>在明白了这个机制之后，我们就可以创建StorageClass了，如下所示：</p><pre><code>$ kubectl create -f local-sc.yaml 
storageclass.storage.k8s.io/local-storage created
</code></pre><p><span class="orange">接下来，我们只需要定义一个非常普通的PVC，就可以让Pod使用到上面定义好的Local Persistent Volume了</span>，如下所示：</p><pre><code>kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: example-local-claim
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: local-storage
</code></pre><p>可以看到，这个PVC没有任何特别的地方。唯一需要注意的是，它声明的storageClassName是local-storage。所以，将来Kubernetes的Volume Controller看到这个PVC的时候，不会为它进行绑定操作。</p><p>现在，我们来创建这个PVC：</p><pre><code>$ kubectl create -f local-pvc.yaml 
persistentvolumeclaim/example-local-claim created

$ kubectl get pvc
NAME                  STATUS    VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS    AGE
example-local-claim   Pending                                       local-storage   7s
</code></pre><p>可以看到，尽管这个时候，Kubernetes里已经存在了一个可以与PVC匹配的PV，但这个PVC依然处于Pending状态，也就是等待绑定的状态。</p><p><span class="orange">然后，我们编写一个Pod来声明使用这个PVC</span>，如下所示：</p><pre><code>kind: Pod
apiVersion: v1
metadata:
  name: example-pv-pod
spec:
  volumes:
    - name: example-pv-storage
      persistentVolumeClaim:
       claimName: example-local-claim
  containers:
    - name: example-pv-container
      image: nginx
      ports:
        - containerPort: 80
          name: &quot;http-server&quot;
      volumeMounts:
        - mountPath: &quot;/usr/share/nginx/html&quot;
          name: example-pv-storage
</code></pre><p>这个Pod没有任何特别的地方，你只需要注意，它的volumes字段声明要使用前面定义的、名叫example-local-claim的PVC即可。</p><p>而我们一旦使用kubectl create创建这个Pod，就会发现，我们前面定义的PVC，会立刻变成Bound状态，与前面定义的PV绑定在了一起，如下所示：</p><pre><code>$ kubectl create -f local-pod.yaml 
pod/example-pv-pod created

$ kubectl get pvc
NAME                  STATUS    VOLUME       CAPACITY   ACCESS MODES   STORAGECLASS    AGE
example-local-claim   Bound     example-pv   5Gi        RWO            local-storage   6h
</code></pre><p>也就是说，在我们创建的Pod进入调度器之后，“绑定”操作才开始进行。</p><p>这时候，我们可以尝试在这个Pod的Volume目录里，创建一个测试文件，比如：</p><pre><code>$ kubectl exec -it example-pv-pod -- /bin/sh
# cd /usr/share/nginx/html
# touch test.txt
</code></pre><p>然后，登录到node-1这台机器上，查看一下它的 /mnt/disks/vol1目录下的内容，你就可以看到刚刚创建的这个文件：</p><pre><code># 在node-1上
$ ls /mnt/disks/vol1
test.txt
</code></pre><p>而如果你重新创建这个Pod的话，就会发现，我们之前创建的测试文件，依然被保存在这个持久化Volume当中：</p><pre><code>$ kubectl delete -f local-pod.yaml 

$ kubectl create -f local-pod.yaml 

$ kubectl exec -it example-pv-pod -- /bin/sh
# ls /usr/share/nginx/html
# touch test.txt
</code></pre><p>这就说明，像Kubernetes这样构建出来的、基于本地存储的Volume，完全可以提供容器持久化存储的功能。所以，像StatefulSet这样的有状态编排工具，也完全可以通过声明Local类型的PV和PVC，来管理应用的存储状态。</p><p><strong>需要注意的是，我们上面手动创建PV的方式，即Static的PV管理方式，在删除PV时需要按如下流程执行操作：</strong></p><ol>
<li>
<p>删除使用这个PV的Pod；</p>
</li>
<li>
<p>从宿主机移除本地磁盘（比如，umount它）；</p>
</li>
<li>
<p>删除PVC；</p>
</li>
<li>
<p>删除PV。</p>
</li>
</ol><p>如果不按照这个流程的话，这个PV的删除就会失败。</p><p>当然，由于上面这些创建PV和删除PV的操作比较繁琐，Kubernetes其实提供了一个Static Provisioner来帮助你管理这些PV。</p><p>比如，我们现在的所有磁盘，都挂载在宿主机的/mnt/disks目录下。</p><p>那么，当Static Provisioner启动后，它就会通过DaemonSet，自动检查每个宿主机的/mnt/disks目录。然后，调用Kubernetes API，为这些目录下面的每一个挂载，创建一个对应的PV对象出来。这些自动创建的PV，如下所示：</p><pre><code>$ kubectl get pv
NAME                CAPACITY    ACCESSMODES   RECLAIMPOLICY   STATUS      CLAIM     STORAGECLASS    REASON    AGE
local-pv-ce05be60   1024220Ki   RWO           Delete          Available             local-storage             26s

$ kubectl describe pv local-pv-ce05be60 
Name:  local-pv-ce05be60
...
StorageClass: local-storage
Status:  Available
Claim:  
Reclaim Policy: Delete
Access Modes: RWO
Capacity: 1024220Ki
NodeAffinity:
  Required Terms:
      Term 0:  kubernetes.io/hostname in [node-1]
Message: 
Source:
    Type: LocalVolume (a persistent volume backed by local storage on a node)
    Path: /mnt/disks/vol1
</code></pre><p>这个PV里的各种定义，比如StorageClass的名字、本地磁盘挂载点的位置，都可以通过provisioner的<a href="https://github.com/kubernetes-incubator/external-storage/tree/master/local-volume/helm">配置文件指定</a>。当然，provisioner也会负责前面提到的PV的删除工作。</p><p>而这个provisioner本身，其实也是一个我们前面提到过的<a href="https://github.com/kubernetes-incubator/external-storage/tree/master/local-volume">External Provisioner</a>，它的部署方法，在<a href="https://github.com/kubernetes-incubator/external-storage/tree/master/local-volume#option-1-using-the-local-volume-static-provisioner">对应的文档里</a>有详细描述。这部分内容，就留给你课后自行探索了。</p><h2>总结</h2><p>在今天这篇文章中，我为你详细介绍了Kubernetes里Local Persistent Volume的实现方式。</p><p>可以看到，正是通过PV和PVC，以及StorageClass这套存储体系，这个后来新添加的持久化存储方案，对Kubernetes已有用户的影响，几乎可以忽略不计。作为用户，你的Pod的YAML和PVC的YAML，并没有任何特殊的改变，这个特性所有的实现只会影响到PV的处理，也就是由运维人员负责的那部分工作。</p><p>而这，正是这套存储体系带来的“解耦”的好处。</p><p>其实，Kubernetes很多看起来比较“繁琐”的设计（比如“声明式API”，以及我今天讲解的“PV、PVC体系”）的主要目的，都是希望为开发者提供更多的“可扩展性”，给使用者带来更多的“稳定性”和“安全感”。这两个能力的高低，是衡量开源基础设施项目水平的重要标准。</p><h2>思考题</h2><p>正是由于需要使用“延迟绑定”这个特性，Local Persistent Volume目前还不能支持Dynamic Provisioning。你是否能说出，为什么“延迟绑定”会跟Dynamic Provisioning有冲突呢？</p><p>感谢你的收听，欢迎你给我留言，也欢迎分享给更多的朋友一起阅读。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/4d/dd/3b594e8d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>realc</span>
  </div>
  <div class="_2_QraFYR_0">知其然，知其所以然。很多教程教材，就跟上学时学校直接灌给我们一样，要让我们去硬啃。不如老师这课程，一个知识点，一个功能的来龙去脉、前世今生都给讲的清清楚楚的。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-29 11:13:30</div>
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
  <div class="_2_QraFYR_0">思考题，我的理解：<br>因为当一个pvc创建之后，kubernetes因为dynamic provisioning机制会调用pvc指定的storageclass里的provisioner自动使用local disk的信息去创建pv。而且pv一旦创建，nodeaffinity参数就指定了固定的node。而此时，provisioner并没有pod调度的相关信息。<br>延迟绑定发生的时机是pod已经进入调度器。此时对应的pv已经创建，绑定了node。并可能与pod的调度信息发生冲突。<br>如果dynamic provisioning机制能够推迟到pod 调度的阶段，同时考虑pod调度条件和node硬件信息，这样才能实现dynamic provisioning。实现上可以参考延迟绑定，来一个延迟 provision。另外实现一个controller在pod调度阶段创建pv。<br><br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-29 14:19:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/11/4b/fa64f061.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>xfan</span>
  </div>
  <div class="_2_QraFYR_0">思考题：<br>因为dynamic provision机制不知道pod需要在哪个node下运行，而提前就创建好了，，</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-23 11:00:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/52/3c/d6fcb93a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张三</span>
  </div>
  <div class="_2_QraFYR_0">Dynamic Provisioning 提供的是自动创建PV的机制，会根据PVC来创建PV并绑定，而我们的延迟绑定是需要在调度的时候综合考虑所有的调度条件来进行PVC和PV的绑定并调度Pod到PV所在的节点，二者有冲突</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-12 17:54:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/69/4d/81c44f45.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>拉欧</span>
  </div>
  <div class="_2_QraFYR_0">Dynamic Provisioning 是通过pvc 创建指定规格的pv, 而Local Persistent Volume 是先创建pv, 在创建pvc, 然后在pod创建的时候绑定pv和pvc；从语义上讲，Dynamic Provisioning 就不太可能支持延迟这种效果</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-16 17:33:12</div>
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
  <div class="_2_QraFYR_0">因为Dynamic Provisioning会自动创建PV，也就是说，在PVC创建后就根据StorageClass去自动创建PV并绑定了，而“延迟绑定”发生在调度Pod时，此时PVC已经创建了。因此二者是矛盾的~~</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-13 11:39:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/83/ae/c082bb25.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>大星星</span>
  </div>
  <div class="_2_QraFYR_0">手动删除pv的步骤中，1234步骤，为什么不是1342呢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-14 13:26:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/a9/45/fc03d0cf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>海。</span>
  </div>
  <div class="_2_QraFYR_0">如果不是Local Persistent Volume， 而使用云块存储 aws ebs ， “延迟绑定”和 Dynamic Provisioning 可以不冲突吧？ 我看https:&#47;&#47;github.com&#47;kubernetes-sigs&#47;aws-ebs-csi-driver&#47;blob&#47;master&#47;examples&#47;kubernetes&#47;dynamic-provisioning&#47;specs&#47;storageclass.yaml 的volumeBindingMode也是 WaitForFirstConsumer</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-10 10:43:13</div>
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
  <div class="_2_QraFYR_0">liuchjlu<br>请教一个问题，当使用Local Persistent Volume的时候，pv中声明的local path如果所在节点没有这个目录会不会自动创建？<br><br><br>我的理解是：pv无论是使用NAS或者其他的存储，还是local persistent volume，都是infra的童鞋帮忙先规划好的存储区域。那么你在写这个pv的时候，要确保这块存储可用，比如目录存在。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-09 17:29:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/6c/9a/8e16409b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>liuchjlu</span>
  </div>
  <div class="_2_QraFYR_0">请教一个问题，当使用Local Persistent Volume的时候，pv中声明的local path如果所在节点没有这个目录会不会自动创建？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-03 18:40:45</div>
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
  <div class="_2_QraFYR_0">&#39;一个 PV 一块盘&#39;能再解释下么，如果直接写宿主机上的本地磁盘目录只要把每个container所消耗的硬盘空间都加个cap就能防止磁盘被写满吧？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-30 13:21:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/O6qftWBakkjQHrAhF5tia21GKkQxibJaPy2nWUKc9eiaouaqb67Hj60RRKgjgHhzPmaxaHkLszcNYrDSkj21lPylQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_a852c8</span>
  </div>
  <div class="_2_QraFYR_0">[root@VM-20-5-centos test1]# kubectl get pv <br>NAME         CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                         STORAGECLASS    REASON   AGE<br>example-pv   1Gi        RWO            Delete           Failed   default&#47;example-local-claim   local-storage            81s<br><br>绑定了pvc后的pv。如果删除了pvc后。状态变成failed.那怎么才能让他变回Available呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-27 20:07:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/42/99/7d9b1fd5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>阿喵</span>
  </div>
  <div class="_2_QraFYR_0">思考题：<br>假设现在有node1和node2两个节点，且pod因为某些原因只能运行在node2节点上，而dynamic provisioning在pvc创建之后就自动创建pv与其绑定，此时并不会考虑pod的调度情况，假设这个pv被创建在node1节点上，那么pod必然调度失败</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-12 17:33:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/be/18/ecac08dc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>阿伦</span>
  </div>
  <div class="_2_QraFYR_0">有一点疑惑，重启的pod，是如何保证自己一定启动在之前关联的PV机器上的，上面的例子说pod只能运行在2节点上，那如果没有这个限制，被部署在了1机器上，然后写了数据之后，pod重启，运行到2机器上了，那1机器上的PV有数据持久化又有什么意义呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-13 21:03:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/80/a5/abb7bfe3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>要没时间了</span>
  </div>
  <div class="_2_QraFYR_0">Dynamic Provisioning 这个机制，关键是在于，如果它最终的结果是仅有一个合法的PV以及底层的持久化目录， 这就反向影响了Pod Schduling的决定，相当于可被调度的候选节点只有一个了。<br><br>要不然，就在多个，甚至集群全部节点都创建好相应的local PV。这样可以一定程度，或者彻底消除调度失败的可能。<br><br>否则，就只能在volume创建的时候想办法，在scheduler里面实现相应的通过调用CSI 的API创建volume的逻辑。<br><br>后者所谓的“动态创建”的方式，虽然可能，但是是不适合放在 scheduler里面的。scheduler按照责任划分来说，不应该再对资源进行修改了，以为她本身就是一个根据现有资源情况做调度决定的组件。<br><br>通过上面的考虑，如果资源充足，那就在集群出划分出一个专门可以使用local volume的节点池，在上面每一个节点都创建相应的PV。否则，感觉是没办法解决这个冲突的</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-25 22:30:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/T7sFX0O4Tdwic8RUolZVe4hNPDiaiaxsfGD4qCBsmac8Iqcibe23Y3jEOQyTic7hsYn46ETeC56jhJ4nFOdOsEZxchw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>loser</span>
  </div>
  <div class="_2_QraFYR_0">只有几个G那种怎么挂载一个单盘呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-11 08:08:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/94/7c/70bf584f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>勤奋的辣牛肉</span>
  </div>
  <div class="_2_QraFYR_0">Static Provisioner  和 local-volume-provisioner  是一个东西么?</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-09 21:43:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/25/2f/03/0be37ebd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Lccc</span>
  </div>
  <div class="_2_QraFYR_0">如果PVC支持扩展nodeAffinity字段, 是不是就可以解决延迟绑定了, StorageClass在读取到新增PVC后, 直接创建指定node的PV并进行Bound</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-04 01:25:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/6b/e2/f02e45df.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>恰同学少年。</span>
  </div>
  <div class="_2_QraFYR_0">&quot;需要注意的是，我们上面手动创建 PV 的方式，即 Static 的 PV 管理方式，在删除 PV 时需要按如下流程执行操作：删除使用这个 PV 的 Pod；从宿主机移除本地磁盘（比如，umount 它）；删除 PVC；删除 PV。&quot;<br><br><br>请问这里为什么不按这个顺序就会失败？我在k8s环境实践了以下两种方式也可以删除成功。<br>方案一：只不过这里可能的问题是在删除pv pvc的窗口期pod读写会有异常？<br>- 删除PV<br>- 删除PVC<br>- 删除Pod<br><br>方案二：<br>- 删除PVC<br>- 删除Pod<br>- 删除PV</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-19 16:17:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/0e/6b/9ee9f422.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>wx</span>
  </div>
  <div class="_2_QraFYR_0">思考题: <br>1. Static Provisioning 的pv 是提前创建好的，调度器在调度pod时，一方面可以考虑nodeAffinity等调度需求，另一方面也能通过已有pv信息知道 pv的空间够不够。所以pv绑定pvc 可以放在Pod调度后进行。<br>2. Dynamic Provisioning，最大的问题: 无法提前知道节点 空间够不够。<br>情况一: 如果让Pod先调度到节点上之后，再通过 storageClass 创建pv绑定pvc，很有可能 节点本地磁盘空间 都不够了。<br>情况二: 如果先让 storageClass 创建pv绑定pvc，再调度Pod，pvc所在的节点信息 和 Pod 需要的节点信息 产生冲突。<br><br>求老师给 正确解答</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-21 20:52:59</div>
  </div>
</div>
</div>
</li>
</ul>