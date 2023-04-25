<audio title="11 _ 从0到1：搭建一个完整的Kubernetes集群" src="https://static001.geekbang.org/resource/audio/88/5d/88f0e653ae6279883eada707758f2a5d.mp3" controls="controls"></audio> 
<p>你好，我是张磊。今天我和你分享的主题是：从0到1搭建一个完整的Kubernetes集群。</p><p>不过，首先需要指出的是，本篇搭建指南是完全的手工操作，细节比较多，并且有些外部链接可能还会遇到特殊的“网络问题”。所以，对于只关心学习 Kubernetes 本身知识点、不太关注如何手工部署 Kubernetes 集群的同学，可以略过本节，直接使用 <a href="https://github.com/kubernetes/minikube">MiniKube</a> 或者 <a href="https://github.com/kubernetes-sigs/kind">Kind</a>，来在本地启动简单的 Kubernetes 集群进行后面的学习即可。如果是使用 MiniKube 的话，阿里云还维护了一个<a href="https://github.com/AliyunContainerService/minikube">国内版的 MiniKube</a>，这对于在国内的同学来说会比较友好。</p><p>在上一篇文章中，我介绍了kubeadm这个Kubernetes半官方管理工具的工作原理。既然kubeadm的初衷是让Kubernetes集群的部署不再让人头疼，那么这篇文章，我们就来使用它部署一个完整的Kubernetes集群吧。</p><blockquote>
<p>备注：这里所说的“完整”，指的是这个集群具备Kubernetes项目在GitHub上已经发布的所有功能，并能够模拟生产环境的所有使用需求。但并不代表这个集群是生产级别可用的：类似于高可用、授权、多租户、灾难备份等生产级别集群的功能暂时不在本篇文章的讨论范围。<br>
目前，kubeadm的高可用部署<a href="https://kubernetes.io/docs/setup/independent/high-availability/">已经有了第一个发布</a>。但是，这个特性还没有GA（生产可用），所以包括了大量的手动工作，跟我们所预期的一键部署还有一定距离。GA的日期预计是2018年底到2019年初。届时，如果有机会我会再和你分享这部分内容。</p>
</blockquote><!-- [[[read_end]]] --><p>这次部署，我不会依赖于任何公有云或私有云的能力，而是完全在Bare-metal环境中完成。这样的部署经验会更有普适性。而在后续的讲解中，如非特殊强调，我也都会以本次搭建的这个集群为基础。</p><h2>准备工作</h2><p>首先，准备机器。最直接的办法，自然是到公有云上申请几个虚拟机。当然，如果条件允许的话，拿几台本地的物理服务器来组集群是最好不过了。这些机器只要满足如下几个条件即可：</p><ol>
<li>
<p>满足安装Docker项目所需的要求，比如64位的Linux操作系统、3.10及以上的内核版本；</p>
</li>
<li>
<p>x86或者ARM架构均可；</p>
</li>
<li>
<p>机器之间网络互通，这是将来容器之间网络互通的前提；</p>
</li>
<li>
<p>有外网访问权限，因为需要拉取镜像；</p>
</li>
<li>
<p>能够访问到<code>gcr.io、quay.io</code>这两个docker registry，因为有小部分镜像需要在这里拉取；</p>
</li>
<li>
<p>单机可用资源建议2核CPU、8 GB内存或以上，再小的话问题也不大，但是能调度的Pod数量就比较有限了；</p>
</li>
<li>
<p>30 GB或以上的可用磁盘空间，这主要是留给Docker镜像和日志文件用的。</p>
</li>
</ol><p>在本次部署中，我准备的机器配置如下：</p><ol>
<li>
<p>2核CPU、 7.5 GB内存；</p>
</li>
<li>
<p>30 GB磁盘；</p>
</li>
<li>
<p>Ubuntu 16.04；</p>
</li>
<li>
<p>内网互通；</p>
</li>
<li>
<p>外网访问权限不受限制。</p>
</li>
</ol><blockquote>
<p>备注：在开始部署前，我推荐你先花几分钟时间，回忆一下Kubernetes的架构。</p>
</blockquote><p>然后，我再和你介绍一下今天实践的目标：</p><ol>
<li>
<p>在所有节点上安装Docker和kubeadm；</p>
</li>
<li>
<p>部署Kubernetes Master；</p>
</li>
<li>
<p>部署容器网络插件；</p>
</li>
<li>
<p>部署Kubernetes Worker；</p>
</li>
<li>
<p>部署Dashboard可视化插件；</p>
</li>
<li>
<p>部署容器存储插件。</p>
</li>
</ol><p>好了，现在，就来开始这次集群部署之旅吧！</p><h2>安装kubeadm和Docker</h2><p>我在上一篇文章《 Kubernetes一键部署利器：kubeadm》中，已经介绍过kubeadm的基础用法，它的一键安装非常方便，我们只需要添加kubeadm的源，然后直接使用apt-get安装即可，具体流程如下所示：</p><blockquote>
<p>备注：为了方便讲解，我后续都会直接在root用户下进行操作。</p>
</blockquote><pre><code>$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
$ cat &lt;&lt;EOF &gt; /etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
$ apt-get update
$ apt-get install -y docker.io kubeadm
</code></pre><blockquote>
<p>提示：如果 apt.kubernetes.io 因为网络问题访问不到，可以换成中科大的 Ubuntu 镜像源deb <a href="http://mirrors.ustc.edu.cn/kubernetes/apt">http://mirrors.ustc.edu.cn/kubernetes/apt</a> kubernetes-xenial main。</p>
</blockquote><p>在上述安装kubeadm的过程中，kubeadm和kubelet、kubectl、kubernetes-cni这几个二进制文件都会被自动安装好。</p><p>另外，这里我直接使用Ubuntu的docker.io的安装源，原因是Docker公司每次发布的最新的Docker CE（社区版）产品往往还没有经过Kubernetes项目的验证，可能会有兼容性方面的问题。</p><h2>部署Kubernetes的Master节点</h2><p>在上一篇文章中，我已经介绍过kubeadm可以一键部署Master节点。不过，在本篇文章中既然要部署一个“完整”的Kubernetes集群，那我们不妨稍微提高一下难度：通过配置文件来开启一些实验性功能。</p><p>所以，这里我编写了一个给kubeadm用的YAML文件（名叫：kubeadm.yaml）：</p><pre><code>apiVersion: kubeadm.k8s.io/v1alpha1
kind: MasterConfiguration
controllerManagerExtraArgs:
  horizontal-pod-autoscaler-use-rest-clients: &quot;true&quot;
  horizontal-pod-autoscaler-sync-period: &quot;10s&quot;
  node-monitor-grace-period: &quot;10s&quot;
apiServerExtraArgs:
  runtime-config: &quot;api/all=true&quot;
kubernetesVersion: &quot;stable-1.11&quot;
</code></pre><p>这个配置中，我给kube-controller-manager设置了：</p><pre><code>horizontal-pod-autoscaler-use-rest-clients: &quot;true&quot;
</code></pre><p>这意味着，将来部署的kube-controller-manager能够使用自定义资源（Custom Metrics）进行自动水平扩展。这是我后面文章中会重点介绍的一个内容。</p><p>其中，“stable-1.11”就是kubeadm帮我们部署的Kubernetes版本号，即：Kubernetes release 1.11最新的稳定版，在我的环境下，它是v1.11.1。你也可以直接指定这个版本，比如：kubernetesVersion: “v1.11.1”。</p><p>然后，我们只需要执行一句指令：</p><pre><code>$ kubeadm init --config kubeadm.yaml
</code></pre><p>就可以完成Kubernetes Master的部署了，这个过程只需要几分钟。部署完成后，kubeadm会生成一行指令：</p><pre><code>kubeadm join 10.168.0.2:6443 --token 00bwbx.uvnaa2ewjflwu1ry --discovery-token-ca-cert-hash sha256:00eb62a2a6020f94132e3fe1ab721349bbcd3e9b94da9654cfe15f2985ebd711
</code></pre><p>这个kubeadm join命令，就是用来给这个Master节点添加更多工作节点（Worker）的命令。我们在后面部署Worker节点的时候马上会用到它，所以找一个地方把这条命令记录下来。</p><p>此外，kubeadm还会提示我们第一次使用Kubernetes集群所需要的配置命令：</p><pre><code>mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
</code></pre><p>而需要这些配置命令的原因是：Kubernetes集群默认需要加密方式访问。所以，这几条命令，就是将刚刚部署生成的Kubernetes集群的安全配置文件，保存到当前用户的.kube目录下，kubectl默认会使用这个目录下的授权信息访问Kubernetes集群。</p><p>如果不这么做的话，我们每次都需要通过export KUBECONFIG环境变量告诉kubectl这个安全配置文件的位置。</p><p>现在，我们就可以使用kubectl get命令来查看当前唯一一个节点的状态了：</p><pre><code>$ kubectl get nodes

NAME      STATUS     ROLES     AGE       VERSION
master    NotReady   master    1d        v1.11.1
</code></pre><p>可以看到，这个get指令输出的结果里，Master节点的状态是NotReady，这是为什么呢？</p><p>在调试Kubernetes集群时，最重要的手段就是用kubectl describe来查看这个节点（Node）对象的详细信息、状态和事件（Event），我们来试一下：</p><pre><code>$ kubectl describe node master

...
Conditions:
...

Ready   False ... KubeletNotReady  runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized
</code></pre><p>通过kubectl describe指令的输出，我们可以看到NodeNotReady的原因在于，我们尚未部署任何网络插件。</p><p>另外，我们还可以通过kubectl检查这个节点上各个系统Pod的状态，其中，kube-system是Kubernetes项目预留的系统Pod的工作空间（Namepsace，注意它并不是Linux Namespace，它只是Kubernetes划分不同工作空间的单位）：</p><pre><code>$ kubectl get pods -n kube-system

NAME               READY   STATUS   RESTARTS  AGE
coredns-78fcdf6894-j9s52     0/1    Pending  0     1h
coredns-78fcdf6894-jm4wf     0/1    Pending  0     1h
etcd-master           1/1    Running  0     2s
kube-apiserver-master      1/1    Running  0     1s
kube-controller-manager-master  0/1    Pending  0     1s
kube-proxy-xbd47         1/1    NodeLost  0     1h
kube-scheduler-master      1/1    Running  0     1s
</code></pre><p>可以看到，CoreDNS、kube-controller-manager等依赖于网络的Pod都处于Pending状态，即调度失败。这当然是符合预期的：因为这个Master节点的网络尚未就绪。</p><h2>部署网络插件</h2><p>在Kubernetes项目“一切皆容器”的设计理念指导下，部署网络插件非常简单，只需要执行一句kubectl apply指令，以Weave为例：</p><pre><code>$ kubectl apply -f https://git.io/weave-kube-1.6
</code></pre><p>部署完成后，我们可以通过kubectl get重新检查Pod的状态：</p><pre><code>$ kubectl get pods -n kube-system

NAME                             READY     STATUS    RESTARTS   AGE
coredns-78fcdf6894-j9s52         1/1       Running   0          1d
coredns-78fcdf6894-jm4wf         1/1       Running   0          1d
etcd-master                      1/1       Running   0          9s
kube-apiserver-master            1/1       Running   0          9s
kube-controller-manager-master   1/1       Running   0          9s
kube-proxy-xbd47                 1/1       Running   0          1d
kube-scheduler-master            1/1       Running   0          9s
weave-net-cmk27                  2/2       Running   0          19s
</code></pre><p>可以看到，所有的系统Pod都成功启动了，而刚刚部署的Weave网络插件则在kube-system下面新建了一个名叫weave-net-cmk27的Pod，一般来说，这些Pod就是容器网络插件在每个节点上的控制组件。</p><p>Kubernetes支持容器网络插件，使用的是一个名叫CNI的通用接口，它也是当前容器网络的事实标准，市面上的所有容器网络开源项目都可以通过CNI接入Kubernetes，比如Flannel、Calico、Canal、Romana等等，它们的部署方式也都是类似的“一键部署”。关于这些开源项目的实现细节和差异，我会在后续的网络部分详细介绍。</p><p>至此，Kubernetes的Master节点就部署完成了。如果你只需要一个单节点的Kubernetes，现在你就可以使用了。不过，在默认情况下，Kubernetes的Master节点是不能运行用户Pod的，所以还需要额外做一个小操作。在本篇的最后部分，我会介绍到它。</p><h2>部署Kubernetes的Worker节点</h2><p>Kubernetes的Worker节点跟Master节点几乎是相同的，它们运行着的都是一个kubelet组件。唯一的区别在于，在kubeadm init的过程中，kubelet启动后，Master节点上还会自动运行kube-apiserver、kube-scheduler、kube-controller-manger这三个系统Pod。</p><p>所以，相比之下，部署Worker节点反而是最简单的，只需要两步即可完成。</p><p>第一步，在所有Worker节点上执行“安装kubeadm和Docker”一节的所有步骤。</p><p>第二步，执行部署Master节点时生成的kubeadm join指令：</p><pre><code>$ kubeadm join 10.168.0.2:6443 --token 00bwbx.uvnaa2ewjflwu1ry --discovery-token-ca-cert-hash sha256:00eb62a2a6020f94132e3fe1ab721349bbcd3e9b94da9654cfe15f2985ebd711
</code></pre><h2>通过Taint/Toleration调整Master执行Pod的策略</h2><p>我在前面提到过，默认情况下Master节点是不允许运行用户Pod的。而Kubernetes做到这一点，依靠的是Kubernetes的Taint/Toleration机制。</p><p>它的原理非常简单：一旦某个节点被加上了一个Taint，即被“打上了污点”，那么所有Pod就都不能在这个节点上运行，因为Kubernetes的Pod都有“洁癖”。</p><p>除非，有个别的Pod声明自己能“容忍”这个“污点”，即声明了Toleration，它才可以在这个节点上运行。</p><p>其中，为节点打上“污点”（Taint）的命令是：</p><pre><code>$ kubectl taint nodes node1 foo=bar:NoSchedule
</code></pre><p>这时，该node1节点上就会增加一个键值对格式的Taint，即：foo=bar:NoSchedule。其中值里面的NoSchedule，意味着这个Taint只会在调度新Pod时产生作用，而不会影响已经在node1上运行的Pod，哪怕它们没有Toleration。</p><p>那么Pod又如何声明Toleration呢？</p><p>我们只要在Pod的.yaml文件中的spec部分，加入tolerations字段即可：</p><pre><code>apiVersion: v1
kind: Pod
...
spec:
  tolerations:
  - key: &quot;foo&quot;
    operator: &quot;Equal&quot;
    value: &quot;bar&quot;
    effect: &quot;NoSchedule&quot;
</code></pre><p>这个Toleration的含义是，这个Pod能“容忍”所有键值对为foo=bar的Taint（ operator: “Equal”，“等于”操作）。</p><p>现在回到我们已经搭建的集群上来。这时，如果你通过kubectl describe检查一下Master节点的Taint字段，就会有所发现了：</p><pre><code>$ kubectl describe node master

Name:               master
Roles:              master
Taints:             node-role.kubernetes.io/master:NoSchedule
</code></pre><p>可以看到，Master节点默认被加上了<code>node-role.kubernetes.io/master:NoSchedule</code>这样一个“污点”，其中“键”是<code>node-role.kubernetes.io/master</code>，而没有提供“值”。</p><p>此时，你就需要像下面这样用“Exists”操作符（operator: “Exists”，“存在”即可）来说明，该Pod能够容忍所有以foo为键的Taint，才能让这个Pod运行在该Master节点上：</p><pre><code>apiVersion: v1
kind: Pod
...
spec:
  tolerations:
  - key: &quot;foo&quot;
    operator: &quot;Exists&quot;
    effect: &quot;NoSchedule&quot;
</code></pre><p>当然，如果你就是想要一个单节点的Kubernetes，删除这个Taint才是正确的选择：</p><pre><code>$ kubectl taint nodes --all node-role.kubernetes.io/master-
</code></pre><p>如上所示，我们在“<code>node-role.kubernetes.io/master</code>”这个键后面加上了一个短横线“-”，这个格式就意味着移除所有以“<code>node-role.kubernetes.io/master</code>”为键的Taint。</p><p>到了这一步，一个基本完整的Kubernetes集群就部署完毕了。是不是很简单呢？</p><p>有了kubeadm这样的原生管理工具，Kubernetes的部署已经被大大简化。更重要的是，像证书、授权、各个组件的配置等部署中最麻烦的操作，kubeadm都已经帮你完成了。</p><p>接下来，我们再在这个Kubernetes集群上安装一些其他的辅助插件，比如Dashboard和存储插件。</p><h2>部署Dashboard可视化插件</h2><p>在Kubernetes社区中，有一个很受欢迎的Dashboard项目，它可以给用户提供一个可视化的Web界面来查看当前集群的各种信息。毫不意外，它的部署也相当简单：</p><pre><code>$ kubectl apply -f 
$ $ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-rc6/aio/deploy/recommended.yaml
</code></pre><p>部署完成之后，我们就可以查看Dashboard对应的Pod的状态了：</p><pre><code>$ kubectl get pods -n kube-system

kubernetes-dashboard-6948bdb78-f67xk   1/1       Running   0          1m
</code></pre><p>需要注意的是，由于Dashboard是一个Web Server，很多人经常会在自己的公有云上无意地暴露Dashboard的端口，从而造成安全隐患。所以，1.7版本之后的Dashboard项目部署完成后，默认只能通过Proxy的方式在本地访问。具体的操作，你可以查看Dashboard项目的<a href="https://github.com/kubernetes/dashboard">官方文档</a>。</p><p>而如果你想从集群外访问这个Dashboard的话，就需要用到Ingress，我会在后面的文章中专门介绍这部分内容。</p><h2>部署容器存储插件</h2><p>接下来，让我们完成这个Kubernetes集群的最后一块拼图：容器持久化存储。</p><p>我在前面介绍容器原理时已经提到过，很多时候我们需要用数据卷（Volume）把外面宿主机上的目录或者文件挂载进容器的Mount Namespace中，从而达到容器和宿主机共享这些目录或者文件的目的。容器里的应用，也就可以在这些数据卷中新建和写入文件。</p><p>可是，如果你在某一台机器上启动的一个容器，显然无法看到其他机器上的容器在它们的数据卷里写入的文件。<strong>这是容器最典型的特征之一：无状态。</strong></p><p>而容器的持久化存储，就是用来保存容器存储状态的重要手段：存储插件会在容器里挂载一个基于网络或者其他机制的远程数据卷，使得在容器里创建的文件，实际上是保存在远程存储服务器上，或者以分布式的方式保存在多个节点上，而与当前宿主机没有任何绑定关系。这样，无论你在其他哪个宿主机上启动新的容器，都可以请求挂载指定的持久化存储卷，从而访问到数据卷里保存的内容。<strong>这就是“持久化”的含义。</strong></p><p>由于Kubernetes本身的松耦合设计，绝大多数存储项目，比如Ceph、GlusterFS、NFS等，都可以为Kubernetes提供持久化存储能力。在这次的部署实战中，我会选择部署一个很重要的Kubernetes存储插件项目：Rook。</p><p>Rook项目是一个基于Ceph的Kubernetes存储插件（它后期也在加入对更多存储实现的支持）。不过，不同于对Ceph的简单封装，Rook在自己的实现中加入了水平扩展、迁移、灾难备份、监控等大量的企业级功能，使得这个项目变成了一个完整的、生产级别可用的容器存储插件。</p><p>得益于容器化技术，用几条指令，Rook就可以把复杂的Ceph存储后端部署起来：</p><pre><code>$ kubectl apply -f https://raw.githubusercontent.com/rook/rook/master/cluster/examples/kubernetes/ceph/common.yaml

$ kubectl apply -f https://raw.githubusercontent.com/rook/rook/master/cluster/examples/kubernetes/ceph/operator.yaml

$ kubectl apply -f https://raw.githubusercontent.com/rook/rook/master/cluster/examples/kubernetes/ceph/cluster.yaml
</code></pre><p>在部署完成后，你就可以看到Rook项目会将自己的Pod放置在由它自己管理的两个Namespace当中：</p><pre><code>$ kubectl get pods -n rook-ceph-system
NAME                                  READY     STATUS    RESTARTS   AGE
rook-ceph-agent-7cv62                 1/1       Running   0          15s
rook-ceph-operator-78d498c68c-7fj72   1/1       Running   0          44s
rook-discover-2ctcv                   1/1       Running   0          15s

$ kubectl get pods -n rook-ceph
NAME                   READY     STATUS    RESTARTS   AGE
rook-ceph-mon0-kxnzh   1/1       Running   0          13s
rook-ceph-mon1-7dn2t   1/1       Running   0          2s
</code></pre><p>这样，一个基于Rook持久化存储集群就以容器的方式运行起来了，而接下来在Kubernetes项目上创建的所有Pod就能够通过Persistent Volume（PV）和Persistent Volume Claim（PVC）的方式，在容器里挂载由Ceph提供的数据卷了。</p><p>而Rook项目，则会负责这些数据卷的生命周期管理、灾难备份等运维工作。关于这些容器持久化存储的知识，我会在后续章节中专门讲解。</p><p>这时候，你可能会有个疑问：为什么我要选择Rook项目呢？</p><p>其实，是因为这个项目很有前途。</p><p>如果你去研究一下Rook项目的实现，就会发现它巧妙地依赖了Kubernetes提供的编排能力，合理的使用了很多诸如Operator、CRD等重要的扩展特性（这些特性我都会在后面的文章中逐一讲解到）。这使得Rook项目，成为了目前社区中基于Kubernetes API构建的最完善也最成熟的容器存储插件。我相信，这样的发展路线，很快就会得到整个社区的推崇。</p><blockquote>
<p>备注：其实，在很多时候，大家说的所谓“云原生”，就是“Kubernetes原生”的意思。而像Rook、Istio这样的项目，正是贯彻这个思路的典范。在我们后面讲解了声明式API之后，相信你对这些项目的设计思想会有更深刻的体会。</p>
</blockquote><h2>总结</h2><p>在本篇文章中，我们完全从0开始，在Bare-metal环境下使用kubeadm工具部署了一个完整的Kubernetes集群：这个集群有一个Master节点和多个Worker节点；使用Weave作为容器网络插件；使用Rook作为容器持久化存储插件；使用Dashboard插件提供了可视化的Web界面。</p><p>这个集群，也将会是我进行后续讲解所依赖的集群环境，并且在后面的讲解中，我还会给它安装更多的插件，添加更多的新能力。</p><p>另外，这个集群的部署过程并不像传说中那么繁琐，这主要得益于：</p><ol>
<li>
<p>kubeadm项目大大简化了部署Kubernetes的准备工作，尤其是配置文件、证书、二进制文件的准备和制作，以及集群版本管理等操作，都被kubeadm接管了。</p>
</li>
<li>
<p>Kubernetes本身“一切皆容器”的设计思想，加上良好的可扩展机制，使得插件的部署非常简便。</p>
</li>
</ol><p>上述思想，也是开发和使用Kubernetes的重要指导思想，即：基于Kubernetes开展工作时，你一定要优先考虑这两个问题：</p><ol>
<li>
<p>我的工作是不是可以容器化？</p>
</li>
<li>
<p>我的工作是不是可以借助Kubernetes API和可扩展机制来完成？</p>
</li>
</ol><p>而一旦这项工作能够基于Kubernetes实现容器化，就很有可能像上面的部署过程一样，大幅简化原本复杂的运维工作。对于时间宝贵的技术人员来说，这个变化的重要性是不言而喻的。</p><h2>思考题</h2><ol>
<li>
<p>你是否使用其他工具部署过Kubernetes项目？经历如何？</p>
</li>
<li>
<p>你是否知道Kubernetes项目当前（v1.11）能够有效管理的集群规模是多少个节点？你在生产环境中希望部署或者正在部署的集群规模又是多少个节点呢？</p>
</li>
</ol><p>感谢你的收听，欢迎你给我留言，也欢迎分享给更多的朋友一起阅读。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/39/bc/2764927e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>L</span>
  </div>
  <div class="_2_QraFYR_0">centos7 单机k8s终于在墙内搭好了，写了篇教程欢迎后来者直接享用<br>https:&#47;&#47;www.datayang.com&#47;article&#47;45</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-05 13:04:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/7e/5c/06e79f14.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>xxx</span>
  </div>
  <div class="_2_QraFYR_0">真的是写的最好的专栏</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-17 09:01:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/c3/d1/bdf895bf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>penng</span>
  </div>
  <div class="_2_QraFYR_0">我说2021年11月才开始看这个课程，这是kubuadm安装的版本已经1.22.3了。也遇到了很多问题，这里总结一下，未来的新同学供参考。<br>1. 使用yum 安装kubeadm和docker后，这俩使用的cgroup驱动不一致。需要在指定docker的cgroup驱动为system。添加如下配置文件，并重启docker服务。<br>	 vim &#47;etc&#47;docker&#47;daemon.json<br>{<br>        &quot;exec-opts&quot;: [&quot;native.cgroupdriver=systemd&quot;],<br>        &quot;log-driver&quot;: &quot;json-file&quot;,<br>        &quot;log-opts&quot;: {<br>                &quot;max-size&quot;: &quot;100m&quot;<br>        }<br>}<br>2. 在写kubeadm.yaml文件时候，老师为了让kube-controller-manager 能够使用自定义资源进行自动水平扩展。加了：horizontal-pod-autoscaler-use-rest-clients: &quot;true&quot;。其实版本1.20以上应该，已经不支持该参数，会导致kube-controller-manager怎么也启动不起来。且新版本的yaml文件的配置格式也有一些变化，我的配置文件如下：<br>vim kubeadm.yaml<br>apiVersion: kubeadm.k8s.io&#47;v1beta3  #版本信息参考kubeadm config print init-defaults命令结果<br>kind: ClusterConfiguration<br>kubernetesVersion: 1.22.3  #版本信息参考kubeadm config print init-defaults命令结果<br>imageRepository: registry.aliyuncs.com&#47;google_containers #配置国内镜像<br><br>apiServer:<br>  extraArgs:<br>    runtime-config: &quot;api&#47;all=true&quot;<br><br>#controllerManager:<br>#  extraArgs:<br>#    horizontal-pod-autoscaler-use-rest-clients: &quot;true&quot; #开启kube-controller-manager能够使用自定义资源（Custom Metrics）进行自动水平扩展,但是高版本不支持该参数需要去掉。<br>#    horizontal-pod-autoscaler-sync-period: &quot;10s&quot;<br>#    node-monitor-grace-period: &quot;10s&quot;  <br><br>etcd:<br>  local:<br>    dataDir: &#47;data&#47;k8s&#47;etcd<br><br>3. 使用kubeadm init后，能成功生成token。但是使用 kubeadm get cs命令看到kube- schedule和kube-controller-manager组件始终起不来，报错日志就是连接127.0.0.1:10251和127.0.0.1:10252超时。这时候需要在&#47;etc&#47;kubernetes&#47;manifests&#47;kube-controller-manager.yaml 和kube-scheduler.yaml文件的--port=0这一行注释掉。并重启kubelet。systemctl restart kubelet<br><br>4. 其他安装镜像或者插件网络或版本兼容问题，其他留言太精彩都留下了解决方案。可以参考，谢谢大家。<br><br>我把我遇到的问题和解决方案也分享处理，供大家参考</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-09 10:44:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/d9/6f/553aeb40.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>杰魁</span>
  </div>
  <div class="_2_QraFYR_0">用了虚拟机，科学上网的问题一个接着一个，搞了一个星期烦死了，最后直接买了美国的3台服务器，只需几分钟就完成了这个章节的所有部署。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-16 20:43:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/ac/62/37912d51.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>东方奇骥</span>
  </div>
  <div class="_2_QraFYR_0">国内，用阿里云源安装就可以了，速度很快：<br>1. apt-get update &amp;&amp; apt-get install -y apt-transport-https<br>2. curl https:&#47;&#47;mirrors.aliyun.com&#47;kubernetes&#47;apt&#47;doc&#47;apt-key.gpg | apt-key add - <br>3. 将deb https:&#47;&#47;mirrors.aliyun.com&#47;kubernetes&#47;apt&#47; kubernetes-xenial main 加入到下面文件中，没有就创建<br>&#47;etc&#47;apt&#47;sources.list.d&#47;kubernetes.list<br>4.apt-get update<br>5.apt-get install -y kubelet kubeadm kubectl</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-24 13:30:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c4/92/338b5609.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Roy Liang</span>
  </div>
  <div class="_2_QraFYR_0">kubeadm最新版本已经为1.12，看到上面很多人遇到提示版本不对，重新安装低版本就好了<br>apt remove kubelet kubectl kubeadm<br>apt install kubelet=1.11.3-00<br>apt install kubectl=1.11.3-00<br>apt install kubeadm=1.11.3-00</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对对对</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-29 15:03:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/8e/76/6d55e26f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张晓辉</span>
  </div>
  <div class="_2_QraFYR_0">最新版本的kubeadm.yaml:<br>cat &lt;&lt;EOF &gt; kubeadm.yaml<br>apiVersion: kubeadm.k8s.io&#47;v1beta2<br>kind: ClusterConfiguration<br>controllerManager:<br>    extraArgs:<br>        horizontal-pod-autoscaler-use-rest-clients: &quot;true&quot;<br>        horizontal-pod-autoscaler-sync-period: &quot;10s&quot;<br>        node-monitor-grace-period: &quot;10s&quot;<br>apiServer:<br>    extraArgs:<br>        runtime-config: &quot;api&#47;all=true&quot;<br>kubernetesVersion: &quot;v1.17.0&quot;<br>EOF<br><br>同时，部署网络插件的命令： <br>kubectl apply -f &quot;https:&#47;&#47;cloud.weave.works&#47;k8s&#47;net?k8s-version=$(kubectl version | base64 | tr -d &#39;\n&#39;)&quot;<br><br>&quot;kubeadm config print init-defaults&quot;这个命令可以告诉我们kubeadm.yaml版本信息。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-14 22:22:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/98/bc/e22fcc1b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>龙哥</span>
  </div>
  <div class="_2_QraFYR_0">对于GFW，不能直接给kubeadm上代理，而是要让docker daemon翻，正确姿势参见：<br>https:&#47;&#47;stackoverflow.com&#47;questions&#47;26550360&#47;docker-ubuntu-behind-proxy</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-18 20:15:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/c2/cf/573a0fdc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>___</span>
  </div>
  <div class="_2_QraFYR_0">kubeadm 1.14 配置文件<br><br>apiVersion: kubeadm.k8s.io&#47;v1beta1<br>kind: ClusterConfiguration<br>controllerManager:<br>    extraArgs:<br>        horizontal-pod-autoscaler-use-rest-clients: &quot;true&quot;<br>        horizontal-pod-autoscaler-sync-period: &quot;10s&quot;<br>        node-monitor-grace-period: &quot;10s&quot;<br>apiServer:<br>    extraArgs:<br>        runtime-config: &quot;api&#47;all=true&quot;<br>kubernetesVersion: &quot;stable-1.14&quot;<br><br>相关文档：https:&#47;&#47;godoc.org&#47;k8s.io&#47;kubernetes&#47;cmd&#47;kubeadm&#47;app&#47;apis&#47;kubeadm&#47;v1beta1<br><br>备注这个针对版本号为1.14的</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-31 20:20:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/f6/4e/0066303c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>cuikt</span>
  </div>
  <div class="_2_QraFYR_0">CentOS Linux release 7.6.1810 (Core) <br>Kubernetes  v1.14.1<br>按照文档部署Rook时会报错<br>Error from server (NotFound): error when creating &quot;operator.yaml&quot;: namespaces &quot;rook-ceph&quot; not found<br><br>error: unable to recognize &quot;cluster.yaml&quot;: no matches for kind &quot;CephCluster&quot; in version &quot;ceph.rook.io&#47;v1&quot;<br><br>解决方式：先apply common.yaml  再执行后续apply<br>kubectl apply -f https:&#47;&#47;raw.githubusercontent.com&#47;rook&#47;rook&#47;master&#47;cluster&#47;examples&#47;kubernetes&#47;ceph&#47;common.yaml</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-25 11:36:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/82/c5/4152b39e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>JefferLiu</span>
  </div>
  <div class="_2_QraFYR_0">ubuntu 换成这个源<br>deb http:&#47;&#47;mirrors.ustc.edu.cn&#47;kubernetes&#47;apt kubernetes-xenial main</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-17 22:04:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/e6/d8/8939848a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Andy</span>
  </div>
  <div class="_2_QraFYR_0">最新的部署ceph cluster需要多一条部署命令才可以部署成功：<br>1. kubectl apply -f https:&#47;&#47;raw.githubusercontent.com&#47;rook&#47;rook&#47;master&#47;cluster&#47;examples&#47;kubernetes&#47;ceph&#47;common.yaml<br>2. kubectl apply -f https:&#47;&#47;raw.githubusercontent.com&#47;rook&#47;rook&#47;master&#47;cluster&#47;examples&#47;kubernetes&#47;ceph&#47;operator.yaml<br>3. kubectl apply -f https:&#47;&#47;raw.githubusercontent.com&#47;rook&#47;rook&#47;master&#47;cluster&#47;examples&#47;kubernetes&#47;ceph&#47;cluster.yaml</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 又更新啦，@编辑</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-17 16:53:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTIGzSM6be2xCNS00kQYHDgXO3icOoOSsvnz3FiaCov5Kgs6oaXkBicLbicuEerJjiaNPWxB0FTVmdur3kg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Xiye</span>
  </div>
  <div class="_2_QraFYR_0">1. 使用过Minikube在自己本机装过K8S，步骤很简单。这个应该比较适合开发使用，不用花太多时间在部署上，但是因为是SingleNode，肯定是不能用于生产。<br>2. 网上查了官方，支持最多5000节点规模的集群。目前我们项目用得3个Master，3个Work Node。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-17 11:06:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/98/d9/e634ec30.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我还是猴子</span>
  </div>
  <div class="_2_QraFYR_0">kubernetes 1.12.2&#47;docker-ce 18.09  <br><br><br>老师的文档是老版本的，看了留言很多 apiversion错误的需要调整老师给的kubeadm init 的kubeadm.yaml文件<br>老版本：<br>apiVersion: kubeadm.k8s.io&#47;v1alpha1<br>kind: MasterConfiguration<br>controllerManagerExtraArgs:<br>  horizontal-pod-autoscaler-use-rest-clients: &quot;true&quot;<br>  horizontal-pod-autoscaler-sync-period: &quot;10s&quot;<br>  node-monitor-grace-period: &quot;10s&quot;<br>apiServerExtraArgs:<br>  runtime-config: &quot;api&#47;all=true&quot;<br>kubernetesVersion: &quot;stable-1.11&quot;<br><br>新版本：<br>apiVersion: kubeadm.k8s.io&#47;v1alpha2<br>kind: MasterConfiguration<br>controllerManagerExtraArgs:<br>  horizontal-pod-autoscaler-use-rest-clients: &quot;true&quot;<br>  horizontal-pod-autoscaler-sync-period: &quot;10s&quot;<br>  node-monitor-grace-period: &quot;10s&quot;<br>apiServerExtraArgs:<br>  runtime-config: &quot;api&#47;all=true&quot;<br>kubernetesVersion: &quot;stable-1.12&quot;<br>需要注意<br>v1alpha2 will be removed in Kubernetes 1.13.  1.13启用v1alpha3  后续有同学也要注意</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-20 15:51:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/53/3d/1189e48a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>微思</span>
  </div>
  <div class="_2_QraFYR_0">老师您好！专网，不能访问互联网的情况下，离线部署kubernetes集群？有没有好的方法和建议？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-17 10:38:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTIp6Ln5VriaBK6uVt3lEOq3UXTVfb4sv2KS3fEH1CiaCeJ69hUcmS35nibJEcRTa1KTa3W5uk60TVM1Q/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>郎晓伟</span>
  </div>
  <div class="_2_QraFYR_0">kubeadm 1.21.1 配置文件kubeadm.yaml：<br><br>apiVersion: kubeadm.k8s.io&#47;v1beta2<br>kind: ClusterConfiguration<br>controllerManager:<br>  extraArgs:<br>    horizontal-pod-autoscaler-use-rest-clients: &quot;true&quot;<br>    horizontal-pod-autoscaler-sync-period: &quot;10s&quot;<br>    node-monitor-grace-period: &quot;10s&quot;<br>apiServer:<br>  extraArgs:<br>    runtime-config: &quot;api&#47;all=true&quot;<br>kubernetesVersion: v1.21.1<br>imageRepository: registry.aliyuncs.com&#47;google_containers<br><br>以上配置文件适用于最新版本的K8s 1.21.1，和Docker 20.10.7 截止2021-06-04。<br><br>注意：这篇文章中使用的版本已经严重落后，而且我第一次部署的时候时候k8s1.11.0和Dcoker 15，后期会遇到各种版本不符的问题，最后直接卡在存储插件Ceph的安装，所以建议不要使用旧版本了，直接上最新的，同时建议作者更新一下文章中使用的版本。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-04 10:52:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/1c/01/5aaaf5b6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Ben</span>
  </div>
  <div class="_2_QraFYR_0">kubeadm 1.18配置<br><br>apiVersion: kubeadm.k8s.io&#47;v1beta2<br>kind: ClusterConfiguration<br>controllerManager:<br>    extraArgs:<br>        horizontal-pod-autoscaler-use-rest-clients: &quot;true&quot;<br>        horizontal-pod-autoscaler-sync-period: &quot;10s&quot;<br>        node-monitor-grace-period: &quot;10s&quot;<br>apiServer:<br>    extraArgs:<br>        runtime-config: &quot;api&#47;all=true&quot;<br>kubernetesVersion: &quot;stable-1.18&quot;<br><br>swap error: sudo swapoff -a</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-19 15:58:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/1e/d7/7d28a531.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>飞机翅膀上</span>
  </div>
  <div class="_2_QraFYR_0">1.20 kubeadm.yaml 最新配置<br><br>apiVersion: kubeadm.k8s.io&#47;v1beta2<br>kind: ClusterConfiguration<br>controllerManager:<br>  extraArgs:<br>    horizontal-pod-autoscaler-use-rest-clients: &quot;true&quot;<br>    horizontal-pod-autoscaler-sync-period: &quot;10s&quot;<br>    node-monitor-grace-period: &quot;10s&quot;<br>apiServer:<br>  extraArgs:<br>    runtime-config: &quot;api&#47;all=true&quot;<br>kubernetesVersion: v1.20.0<br>imageRepository: registry.aliyuncs.com&#47;google_containers</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-27 16:42:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/24/d2/40353046.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>zeroxus</span>
  </div>
  <div class="_2_QraFYR_0">使用配置文件，如果版本不一致，可以用`kubeadm config print init-defaults`查看本机上装的kubeadm的默认支持的配置，比如kubeadm v.1.13，它的apiVersion是kubeadm.k8s.io&#47;v1beta1，kubernetesVersion: v1.13.0</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-07 09:22:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/9c/ec/5da5c049.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Terry Hu</span>
  </div>
  <div class="_2_QraFYR_0">我也为大家提供一些comments:<br>Kubeadm的token 23小时后过期，所以如果你worker端和master 端不是在同一天做的，有可能会出现如下错误:<br>[kubelet] Downloading configuration for the kubelet from the &quot;kubelet-config-1.11&quot; ConfigMap in the kube-system namespace<br>Unauthorized<br><br>Stackoverflow上有这个解:<br>https:&#47;&#47;stackoverflow.com&#47;questions&#47;52823871&#47;unable-to-join-the-worker-node-to-k8-master-node<br><br>简单来说，在master 上重新kubeadm token create一下，替换原来的token 就好了<br><br>另外，如果出现<br>[preflight] Some fatal errors occurred:<br>        [ERROR FileAvailable--etc-kubernetes-pki-ca.crt]: &#47;etc&#47;kubernetes&#47;pki&#47;ca.crt already exists<br>        [ERROR FileAvailable--etc-kubernetes-bootstrap-kubelet.conf]: &#47;etc&#47;kubernetes&#47;bootstrap-kubelet.conf already exists<br>[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`<br><br>把那2个文件mv挪走就好了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-01 17:56:40</div>
  </div>
</div>
</div>
</li>
</ul>