<audio title="10 _ Kubernetes一键部署利器：kubeadm" src="https://static001.geekbang.org/resource/audio/53/99/53e5baf328edf104e60797b9f7d9bb99.mp3" controls="controls"></audio> 
<p>你好，我是张磊。今天我和你分享的主题是：Kubernetes一键部署利器之kubeadm。</p><p>通过前面几篇文章的内容，我其实阐述了这样一个思想：<strong>要真正发挥容器技术的实力，你就不能仅仅局限于对Linux容器本身的钻研和使用。</strong></p><p>这些知识更适合作为你的技术储备，以便在需要的时候可以帮你更快地定位问题，并解决问题。</p><p>而更深入地学习容器技术的关键在于，<strong>如何使用这些技术来“容器化”你的应用。</strong></p><p>比如，我们的应用既可能是Java Web和MySQL这样的组合，也可能是Cassandra这样的分布式系统。而要使用容器把后者运行起来，你单单通过Docker把一个Cassandra镜像跑起来是没用的。</p><p>要把Cassandra应用容器化的关键，在于如何处理好这些Cassandra容器之间的编排关系。比如，哪些Cassandra容器是主，哪些是从？主从容器如何区分？它们之间又如何进行自动发现和通信？Cassandra容器的持久化数据又如何保持，等等。</p><p>这也是为什么我们要反复强调Kubernetes项目的主要原因：这个项目体现出来的容器化“表达能力”，具有独有的先进性和完备性。这就使得它不仅能运行Java Web与MySQL这样的常规组合，还能够处理Cassandra容器集群等复杂编排问题。所以，对这种编排能力的剖析、解读和最佳实践，将是本专栏最重要的一部分内容。</p><!-- [[[read_end]]] --><p>不过，万事开头难。</p><p>作为一个典型的分布式项目，Kubernetes的部署一直以来都是挡在初学者前面的一只“拦路虎”。尤其是在Kubernetes项目发布初期，它的部署完全要依靠一堆由社区维护的脚本。</p><p>其实，Kubernetes作为一个Golang项目，已经免去了很多类似于Python项目要安装语言级别依赖的麻烦。但是，除了将各个组件编译成二进制文件外，用户还要负责为这些二进制文件编写对应的配置文件、配置自启动脚本，以及为kube-apiserver配置授权文件等等诸多运维工作。</p><p>目前，各大云厂商最常用的部署的方法，是使用SaltStack、Ansible等运维工具自动化地执行这些步骤。</p><p>但即使这样，这个部署过程依然非常繁琐。因为，SaltStack这类专业运维工具本身的学习成本，就可能比Kubernetes项目还要高。</p><p><strong>难道Kubernetes项目就没有简单的部署方法了吗？</strong></p><p>这个问题，在Kubernetes社区里一直没有得到足够重视。直到2017年，在志愿者的推动下，社区才终于发起了一个独立的部署工具，名叫：<a href="https://github.com/kubernetes/kubeadm">kubeadm</a>。</p><p>这个项目的目的，就是要让用户能够通过这样两条指令完成一个Kubernetes集群的部署：</p><pre><code># 创建一个Master节点
$ kubeadm init

# 将一个Node节点加入到当前集群中
$ kubeadm join &lt;Master节点的IP和端口&gt;
</code></pre><p>是不是非常方便呢？</p><p>不过，你可能也会有所顾虑：<strong>Kubernetes的功能那么多，这样一键部署出来的集群，能用于生产环境吗？</strong></p><p>为了回答这个问题，在今天这篇文章，我就先和你介绍一下kubeadm的工作原理吧。</p><h2>kubeadm的工作原理</h2><p>在上一篇文章《从容器到容器云：谈谈Kubernetes的本质》中，我已经详细介绍了Kubernetes的架构和它的组件。在部署时，它的每一个组件都是一个需要被执行的、单独的二进制文件。所以不难想象，SaltStack这样的运维工具或者由社区维护的脚本的功能，就是要把这些二进制文件传输到指定的机器当中，然后编写控制脚本来启停这些组件。</p><p>不过，在理解了容器技术之后，你可能已经萌生出了这样一个想法，<strong>为什么不用容器部署Kubernetes呢？</strong></p><p>这样，我只要给每个Kubernetes组件做一个容器镜像，然后在每台宿主机上用docker run指令启动这些组件容器，部署不就完成了吗？</p><p>事实上，在Kubernetes早期的部署脚本里，确实有一个脚本就是用Docker部署Kubernetes项目的，这个脚本相比于SaltStack等的部署方式，也的确简单了不少。</p><p>但是，<strong>这样做会带来一个很麻烦的问题，即：如何容器化kubelet。</strong></p><p>我在上一篇文章中，已经提到kubelet是Kubernetes项目用来操作Docker等容器运行时的核心组件。可是，除了跟容器运行时打交道外，kubelet在配置容器网络、管理容器数据卷时，都需要直接操作宿主机。</p><p>而如果现在kubelet本身就运行在一个容器里，那么直接操作宿主机就会变得很麻烦。对于网络配置来说还好，kubelet容器可以通过不开启Network Namespace（即Docker的host network模式）的方式，直接共享宿主机的网络栈。可是，要让kubelet隔着容器的Mount Namespace和文件系统，操作宿主机的文件系统，就有点儿困难了。</p><p>比如，如果用户想要使用NFS做容器的持久化数据卷，那么kubelet就需要在容器进行绑定挂载前，在宿主机的指定目录上，先挂载NFS的远程目录。</p><p>可是，这时候问题来了。由于现在kubelet是运行在容器里的，这就意味着它要做的这个“mount -F nfs”命令，被隔离在了一个单独的Mount Namespace中。即，kubelet做的挂载操作，不能被“传播”到宿主机上。</p><p>对于这个问题，有人说，可以使用setns()系统调用，在宿主机的Mount Namespace中执行这些挂载操作；也有人说，应该让Docker支持一个–mnt=host的参数。</p><p>但是，到目前为止，在容器里运行kubelet，依然没有很好的解决办法，我也不推荐你用容器去部署Kubernetes项目。</p><p>正因为如此，kubeadm选择了一种妥协方案：</p><blockquote>
<p>把kubelet直接运行在宿主机上，然后使用容器部署其他的Kubernetes组件。</p>
</blockquote><p>所以，你使用kubeadm的第一步，是在机器上手动安装kubeadm、kubelet和kubectl这三个二进制文件。当然，kubeadm的作者已经为各个发行版的Linux准备好了安装包，所以你只需要执行：</p><pre><code>$ apt-get install kubeadm
</code></pre><p>就可以了。</p><p>接下来，你就可以使用“kubeadm init”部署Master节点了。</p><h2>kubeadm init的工作流程</h2><p>当你执行kubeadm init指令后，<strong>kubeadm首先要做的，是一系列的检查工作，以确定这台机器可以用来部署Kubernetes</strong>。这一步检查，我们称为“Preflight Checks”，它可以为你省掉很多后续的麻烦。</p><p>其实，Preflight Checks包括了很多方面，比如：</p><ul>
<li>Linux内核的版本必须是否是3.10以上？</li>
<li>Linux Cgroups模块是否可用？</li>
<li>机器的hostname是否标准？在Kubernetes项目里，机器的名字以及一切存储在Etcd中的API对象，都必须使用标准的DNS命名（RFC 1123）。</li>
<li>用户安装的kubeadm和kubelet的版本是否匹配？</li>
<li>机器上是不是已经安装了Kubernetes的二进制文件？</li>
<li>Kubernetes的工作端口10250/10251/10252端口是不是已经被占用？</li>
<li>ip、mount等Linux指令是否存在？</li>
<li>Docker是否已经安装？</li>
<li>……</li>
</ul><p><strong>在通过了Preflight Checks之后，kubeadm要为你做的，是生成Kubernetes对外提供服务所需的各种证书和对应的目录。</strong></p><p>Kubernetes对外提供服务时，除非专门开启“不安全模式”，否则都要通过HTTPS才能访问kube-apiserver。这就需要为Kubernetes集群配置好证书文件。</p><p>kubeadm为Kubernetes项目生成的证书文件都放在Master节点的/etc/kubernetes/pki目录下。在这个目录下，最主要的证书文件是ca.crt和对应的私钥ca.key。</p><p>此外，用户使用kubectl获取容器日志等streaming操作时，需要通过kube-apiserver向kubelet发起请求，这个连接也必须是安全的。kubeadm为这一步生成的是apiserver-kubelet-client.crt文件，对应的私钥是apiserver-kubelet-client.key。</p><p>除此之外，Kubernetes集群中还有Aggregate APIServer等特性，也需要用到专门的证书，这里我就不再一一列举了。需要指出的是，你可以选择不让kubeadm为你生成这些证书，而是拷贝现有的证书到如下证书的目录里：</p><pre><code>/etc/kubernetes/pki/ca.{crt,key}
</code></pre><p>这时，kubeadm就会跳过证书生成的步骤，把它完全交给用户处理。</p><p><strong>证书生成后，kubeadm接下来会为其他组件生成访问kube-apiserver所需的配置文件</strong>。这些文件的路径是：/etc/kubernetes/xxx.conf：</p><pre><code>ls /etc/kubernetes/
admin.conf  controller-manager.conf  kubelet.conf  scheduler.conf
</code></pre><p>这些文件里面记录的是，当前这个Master节点的服务器地址、监听端口、证书目录等信息。这样，对应的客户端（比如scheduler，kubelet等），可以直接加载相应的文件，使用里面的信息与kube-apiserver建立安全连接。</p><p><strong>接下来，kubeadm会为Master组件生成Pod配置文件</strong>。我已经在上一篇文章中和你介绍过Kubernetes有三个Master组件kube-apiserver、kube-controller-manager、kube-scheduler，而它们都会被使用Pod的方式部署起来。</p><p>你可能会有些疑问：这时，Kubernetes集群尚不存在，难道kubeadm会直接执行docker run来启动这些容器吗？</p><p>当然不是。</p><p>在Kubernetes中，有一种特殊的容器启动方法叫做“Static Pod”。它允许你把要部署的Pod的YAML文件放在一个指定的目录里。这样，当这台机器上的kubelet启动时，它会自动检查这个目录，加载所有的Pod YAML文件，然后在这台机器上启动它们。</p><p>从这一点也可以看出，kubelet在Kubernetes项目中的地位非常高，在设计上它就是一个完全独立的组件，而其他Master组件，则更像是辅助性的系统容器。</p><p>在kubeadm中，Master组件的YAML文件会被生成在/etc/kubernetes/manifests路径下。比如，kube-apiserver.yaml：</p><pre><code>apiVersion: v1
kind: Pod
metadata:
  annotations:
    scheduler.alpha.kubernetes.io/critical-pod: &quot;&quot;
  creationTimestamp: null
  labels:
    component: kube-apiserver
    tier: control-plane
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    - --authorization-mode=Node,RBAC
    - --runtime-config=api/all=true
    - --advertise-address=10.168.0.2
    ...
    - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
    - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
    image: k8s.gcr.io/kube-apiserver-amd64:v1.11.1
    imagePullPolicy: IfNotPresent
    livenessProbe:
      ...
    name: kube-apiserver
    resources:
      requests:
        cpu: 250m
    volumeMounts:
    - mountPath: /usr/share/ca-certificates
      name: usr-share-ca-certificates
      readOnly: true
    ...
  hostNetwork: true
  priorityClassName: system-cluster-critical
  volumes:
  - hostPath:
      path: /etc/ca-certificates
      type: DirectoryOrCreate
    name: etc-ca-certificates
  ...
</code></pre><p>关于一个Pod的YAML文件怎么写、里面的字段如何解读，我会在后续专门的文章中为你详细分析。在这里，你只需要关注这样几个信息：</p><ol>
<li>
<p>这个Pod里只定义了一个容器，它使用的镜像是：<code>k8s.gcr.io/kube-apiserver-amd64:v1.11.1</code> 。这个镜像是Kubernetes官方维护的一个组件镜像。</p>
</li>
<li>
<p>这个容器的启动命令（commands）是kube-apiserver --authorization-mode=Node,RBAC …，这样一句非常长的命令。其实，它就是容器里kube-apiserver这个二进制文件再加上指定的配置参数而已。</p>
</li>
<li>
<p>如果你要修改一个已有集群的kube-apiserver的配置，需要修改这个YAML文件。</p>
</li>
<li>
<p>这些组件的参数也可以在部署时指定，我很快就会讲到。</p>
</li>
</ol><p>在这一步完成后，kubeadm还会再生成一个Etcd的Pod YAML文件，用来通过同样的Static Pod的方式启动Etcd。所以，最后Master组件的Pod YAML文件如下所示：</p><pre><code>$ ls /etc/kubernetes/manifests/
etcd.yaml  kube-apiserver.yaml  kube-controller-manager.yaml  kube-scheduler.yaml
</code></pre><p>而一旦这些YAML文件出现在被kubelet监视的/etc/kubernetes/manifests目录下，kubelet就会自动创建这些YAML文件中定义的Pod，即Master组件的容器。</p><p>Master容器启动后，kubeadm会通过检查localhost:6443/healthz这个Master组件的健康检查URL，等待Master组件完全运行起来。</p><p><strong>然后，kubeadm就会为集群生成一个bootstrap token</strong>。在后面，只要持有这个token，任何一个安装了kubelet和kubadm的节点，都可以通过kubeadm join加入到这个集群当中。</p><p>这个token的值和使用方法，会在kubeadm init结束后被打印出来。</p><p><strong>在token生成之后，kubeadm会将ca.crt等Master节点的重要信息，通过ConfigMap的方式保存在Etcd当中，供后续部署Node节点使用</strong>。这个ConfigMap的名字是cluster-info。</p><p><strong>kubeadm init的最后一步，就是安装默认插件</strong>。Kubernetes默认kube-proxy和DNS这两个插件是必须安装的。它们分别用来提供整个集群的服务发现和DNS功能。其实，这两个插件也只是两个容器镜像而已，所以kubeadm只要用Kubernetes客户端创建两个Pod就可以了。</p><h2>kubeadm join的工作流程</h2><p>这个流程其实非常简单，kubeadm init生成bootstrap token之后，你就可以在任意一台安装了kubelet和kubeadm的机器上执行kubeadm join了。</p><p>可是，为什么执行kubeadm join需要这样一个token呢？</p><p>因为，任何一台机器想要成为Kubernetes集群中的一个节点，就必须在集群的kube-apiserver上注册。可是，要想跟apiserver打交道，这台机器就必须要获取到相应的证书文件（CA文件）。可是，为了能够一键安装，我们就不能让用户去Master节点上手动拷贝这些文件。</p><p>所以，kubeadm至少需要发起一次“不安全模式”的访问到kube-apiserver，从而拿到保存在ConfigMap中的cluster-info（它保存了APIServer的授权信息）。而bootstrap token，扮演的就是这个过程中的安全验证的角色。</p><p>只要有了cluster-info里的kube-apiserver的地址、端口、证书，kubelet就可以以“安全模式”连接到apiserver上，这样一个新的节点就部署完成了。</p><p>接下来，你只要在其他节点上重复这个指令就可以了。</p><h2>配置kubeadm的部署参数</h2><p>我在前面讲了kubeadm部署Kubernetes集群最关键的两个步骤，kubeadm init和kubeadm join。相信你一定会有这样的疑问：kubeadm确实简单易用，可是我又该如何定制我的集群组件参数呢？</p><p>比如，我要指定kube-apiserver的启动参数，该怎么办？</p><p>在这里，我强烈推荐你在使用kubeadm init部署Master节点时，使用下面这条指令：</p><pre><code>$ kubeadm init --config kubeadm.yaml
</code></pre><p>这时，你就可以给kubeadm提供一个YAML文件（比如，kubeadm.yaml），它的内容如下所示（我仅列举了主要部分）：</p><pre><code>apiVersion: kubeadm.k8s.io/v1alpha2
kind: MasterConfiguration
kubernetesVersion: v1.11.0
api:
  advertiseAddress: 192.168.0.102
  bindPort: 6443
  ...
etcd:
  local:
    dataDir: /var/lib/etcd
    image: &quot;&quot;
imageRepository: k8s.gcr.io
kubeProxy:
  config:
    bindAddress: 0.0.0.0
    ...
kubeletConfiguration:
  baseConfig:
    address: 0.0.0.0
    ...
networking:
  dnsDomain: cluster.local
  podSubnet: &quot;&quot;
  serviceSubnet: 10.96.0.0/12
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  ...
</code></pre><p>通过制定这样一个部署参数配置文件，你就可以很方便地在这个文件里填写各种自定义的部署参数了。比如，我现在要指定kube-apiserver的参数，那么我只要在这个文件里加上这样一段信息：</p><pre><code>...
apiServerExtraArgs:
  advertise-address: 192.168.0.103
  anonymous-auth: false
  enable-admission-plugins: AlwaysPullImages,DefaultStorageClass
  audit-log-path: /home/johndoe/audit.log
</code></pre><p>然后，kubeadm就会使用上面这些信息替换/etc/kubernetes/manifests/kube-apiserver.yaml里的command字段里的参数了。</p><p>而这个YAML文件提供的可配置项远不止这些。比如，你还可以修改kubelet和kube-proxy的配置，修改Kubernetes使用的基础镜像的URL（默认的<code>k8s.gcr.io/xxx</code>镜像URL在国内访问是有困难的），指定自己的证书文件，指定特殊的容器运行时等等。这些配置项，就留给你在后续实践中探索了。</p><h2>总结</h2><p>在今天的这次分享中，我重点介绍了kubeadm这个部署工具的工作原理和使用方法。紧接着，我会在下一篇文章中，使用它一步步地部署一个完整的Kubernetes集群。</p><p>从今天的分享中，你可以看到，kubeadm的设计非常简洁。并且，它在实现每一步部署功能时，都在最大程度地重用Kubernetes已有的功能，这也就使得我们在使用kubeadm部署Kubernetes项目时，非常有“原生”的感觉，一点都不会感到突兀。</p><p>而kubeadm的源代码，直接就在kubernetes/cmd/kubeadm目录下，是Kubernetes项目的一部分。其中，app/phases文件夹下的代码，对应的就是我在这篇文章中详细介绍的每一个具体步骤。</p><p>看到这里，你可能会猜想，kubeadm的作者一定是Google公司的某个“大神”吧。</p><p>实际上，kubeadm几乎完全是一位高中生的作品。他叫Lucas Käldström，芬兰人，今年只有18岁。kubeadm，是他17岁时用业余时间完成的一个社区项目。</p><p>所以说，开源社区的魅力也在于此：一个成功的开源项目，总能够吸引到全世界最厉害的贡献者参与其中。尽管参与者的总体水平参差不齐，而且频繁的开源活动又显得杂乱无章难以管控，但一个有足够热度的社区最终的收敛方向，却一定是代码越来越完善、Bug越来越少、功能越来越强大。</p><p>最后，我再来回答一下我在今天这次分享开始提到的问题：kubeadm能够用于生产环境吗？</p><p>到目前为止（2018年9月），这个问题的答案是：不能。</p><p>因为kubeadm目前最欠缺的是，一键部署一个高可用的Kubernetes集群，即：Etcd、Master组件都应该是多节点集群，而不是现在这样的单点。这，当然也正是kubeadm接下来发展的主要方向。</p><p>另一方面，Lucas也正在积极地把kubeadm phases开放给用户，即：用户可以更加自由地定制kubeadm的每一个部署步骤。这些举措，都可以让这个项目更加完善，我对它的发展走向也充满了信心。</p><p>当然，如果你有部署规模化生产环境的需求，我推荐使用<a href="https://github.com/kubernetes/kops">kops</a>或者SaltStack这样更复杂的部署工具。但，在本专栏接下来的讲解中，我都会以kubeadm为依据进行讲述。</p><ul>
<li>一方面，作为Kubernetes项目的原生部署工具，kubeadm对Kubernetes项目特性的使用和集成，确实要比其他项目“技高一筹”，非常值得我们学习和借鉴；</li>
<li>另一方面，kubeadm的部署方法，不会涉及到太多的运维工作，也不需要我们额外学习复杂的部署工具。而它部署的Kubernetes集群，跟一个完全使用二进制文件搭建起来的集群几乎没有任何区别。</li>
</ul><p>因此，使用kubeadm去部署一个Kubernetes集群，对于你理解Kubernetes组件的工作方式和架构，最好不过了。</p><h2>思考题</h2><ol>
<li>
<p>在Linux上为一个类似kube-apiserver的Web Server制作证书，你知道可以用哪些工具实现吗？</p>
</li>
<li>
<p>回忆一下我在前面文章中分享的Kubernetes架构，你能够说出Kubernetes各个功能组件之间（包含Etcd），都有哪些建立连接或者调用的方式吗？（比如：HTTP/HTTPS，远程调用等等）</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/5b/54/0c06dbde.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Antergone</span>
  </div>
  <div class="_2_QraFYR_0">有一个ansible playbook可以推荐给大家。<br>https:&#47;&#47;github.com&#47;gjmzj&#47;kubeasz<br>初学者可以跟着一步步看原理，后期还可以自己定制化。主要是容易产生兴趣。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-14 12:45:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/74/50/59d429c9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>MiracleWong</span>
  </div>
  <div class="_2_QraFYR_0">我也补充一个可用于部署生产级别的Kubernetes的开源项目：https:&#47;&#47;github.com&#47;kubernetes-incubator&#47;kubespray   我们公司正在使用。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-16 19:46:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/59/e5/17bb59ef.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>maojing</span>
  </div>
  <div class="_2_QraFYR_0">楼主在这里有点鄙视swarm了，swarm在一定程度上可以解决很多问题。没有最好的，只有最合适的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Windows server简单易用，界面友好，一直都它的受众和细分市场，但你还是会毫不犹豫的选择Linux server。为什么呢？<br><br>基础设施这件事情，一旦做出选择，整个公司就载进去了。<br><br>这种场合下，为了政治正确强调“只有最合适的”，在我看来，是不负责任的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-23 20:33:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/4c/20/ce900ac6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张应罗</span>
  </div>
  <div class="_2_QraFYR_0">其实国内同学们用kubeadm安装集群最大的拦路虎在于有几个镜像没法下载，我建议大家先手动把镜像pull 下来，从阿里的镜像源上，然后tag成安装所需的镜像名称，这样你发现安装过程会异常顺利</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 没错。kubeadm拉取镜像的url是可配置的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-21 17:54:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/eb/2f/63004f1a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>zz@zz</span>
  </div>
  <div class="_2_QraFYR_0">kbueadm init 遇到问题的同学，可以从报错日志中获得需要的 镜像列表<br><br>- No internet connection is available so the kubelet cannot pull or find the following control plane images:<br>                                - k8s.gcr.io&#47;kube-apiserver-amd64:v1.11.4<br>                                - k8s.gcr.io&#47;kube-controller-manager-amd64:v1.11.4<br>                                - k8s.gcr.io&#47;kube-scheduler-amd64:v1.11.4<br>                                - k8s.gcr.io&#47;etcd-amd64:3.2.18<br>                                - You can check or miligate this in beforehand with &quot;kubeadm config images pull&quot; to make sure the images<br><br>或者使用如下命令<br>kubeadm config images list<br><br>然后使用下边的脚本拉镜像并tag成指定Google的镜像<br><br>images=(<br>    k8s.gcr.io&#47;kube-apiserver-amd64:v1.11.4<br>    k8s.gcr.io&#47;kube-controller-manager-amd64:v1.11.4<br>    k8s.gcr.io&#47;kube-scheduler-amd64:v1.11.4<br>    k8s.gcr.io&#47;kube-proxy-amd64:v1.11.4<br>    k8s.gcr.io&#47;pause:3.1<br>    k8s.gcr.io&#47;etcd-amd64:3.2.18<br>    k8s.gcr.io&#47;coredns:1.1.3<br>)<br><br>for imageName in ${images[@]} ; do<br>    docker pull registry.cn-hangzhou.aliyuncs.com&#47;google_containers&#47;$imageName<br>    docker tag registry.cn-hangzhou.aliyuncs.com&#47;google_containers&#47;$imageName k8s.gcr.io&#47;$imageName<br>done<br><br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-03 22:29:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/c2/e0/7188aa0a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>blackpiglet</span>
  </div>
  <div class="_2_QraFYR_0">1. Linux 下生成证书，主流的选择应该是 OpenSSL，还可以使用 GnuGPG，或者 keybase。<br>2. Kubernetes 组件之间的交互方式：HTTP&#47;HTTPS、gRPC、DNS、系统调用等。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 看来是搞过证书啊</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-20 13:04:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/7a/49/8e6fe04d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>yandd</span>
  </div>
  <div class="_2_QraFYR_0">推荐个k8s实验平台 https:&#47;&#47;console.magicsandbox.com，可能需要fan qiang才能访问</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-15 22:04:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/db/26/54f2c164.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>靠人品去赢</span>
  </div>
  <div class="_2_QraFYR_0">阿里云 + CentoS 安装kubeadm亲测有效（实际就是调整为阿里的镜像）<br>（1）yum install -y ebtables socat（安装依赖，知道是什么依赖的，为什么要依赖的大佬补充一下）<br>（2）# 配置源<br>cat &lt;&lt;EOF &gt; &#47;etc&#47;yum.repos.d&#47;kubernetes.repo<br>[kubernetes]<br>name=Kubernetes<br>baseurl=https:&#47;&#47;mirrors.aliyun.com&#47;kubernetes&#47;yum&#47;repos&#47;kubernetes-el7-x86_64<br>enabled=1<br>gpgcheck=1<br>repo_gpgcheck=1<br>gpgkey=https:&#47;&#47;mirrors.aliyun.com&#47;kubernetes&#47;yum&#47;doc&#47;yum-key.gpg https:&#47;&#47;mirrors.aliyun.com&#47;kubernetes&#47;yum&#47;doc&#47;rpm-package-key.gpg<br>EOF<br>（cat修改文件，实际上重要的是修改镜像源）<br>（3）yum install -y kubelet kubeadm kubectl（这个centos不用apt-get其实算是个常识了，一味按照文中来是不行的，毕竟人家也说了Ubuntu）<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-13 17:01:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/05/d8/cd269378.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>一叶</span>
  </div>
  <div class="_2_QraFYR_0">我们的生产环境是二进制安装的，把安装步骤写成脚本就会方便很多，分享一个二进制安装脚本：<br>https:&#47;&#47;github.com&#47;SongCF&#47;kubesh<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-23 11:18:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/73/63/f008057a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>alex</span>
  </div>
  <div class="_2_QraFYR_0">对墙经验丰富的人来了，可以用下面这个镜像<br>https:&#47;&#47;github.com&#47;anjia0532&#47;gcr.io_mirror</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你好，楼道门开一下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-26 17:00:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>包治结巴</span>
  </div>
  <div class="_2_QraFYR_0">张老师，后面能讲讲怎么用二进制部署kubernetes吗？毕竟kubeadm不适用于生产环境，用二进制部署还是挺复杂的，恳请老师不吝讲解一下吧。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我不建议直接使用二进制文件部署。而建议你花时间了解一下kubeadm的高可用部署，它现在已经初具雏形了。宝贵的时间应该用在刀刃上。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-14 08:59:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/cc/7e/0d050964.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Rootrl</span>
  </div>
  <div class="_2_QraFYR_0">Kubeadm已经生产可用了：https:&#47;&#47;kubernetes.io&#47;blog&#47;2018&#47;12&#47;04&#47;production-ready-kubernetes-cluster-creation-with-kubeadm&#47;</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-02 11:37:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/76/f5/e3f5bd8d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>宝仔</span>
  </div>
  <div class="_2_QraFYR_0">etcd可以先部署，然后初始化的时候通过kubeadm.yaml指定已经部署好的etcd。高可用可以通过部署三个master节点来解决！现在有个问题，通过kubeadm部署生成的apiserver证书默认有效期是一年，官方是认为需要通过kubeadm upgrade 每年升级一次kubernetes，升级的时候也会更新证书，请问老师这个有解决方法吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: HA的etcd也是可以用kubeadm部署的，当然，external etcd有助于你自己做运维。你可以直接改证书生成的步骤，但我当然推荐你执行upgrade，这个操作是必须的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-15 09:06:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/a6/b2/89aae33a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>志远</span>
  </div>
  <div class="_2_QraFYR_0">kubernetes 原来是一个芬兰的高中生写的，不禁让我想起祖师爷 linus 也是芬兰人，哎~</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-15 13:54:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/a0/30/5a7247eb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>eden</span>
  </div>
  <div class="_2_QraFYR_0">有个开源项目kubespray，支持k8s高可用部署，利用ansible来实现，这个项目部署的集群可以用于生产吧</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-14 07:34:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/43/fa/b4324501.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>forever</span>
  </div>
  <div class="_2_QraFYR_0">老师 minikuke和kubeadm有什么区别吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 前者是本地玩玩用的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-24 14:36:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/a0/30/5a7247eb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>eden</span>
  </div>
  <div class="_2_QraFYR_0">一周更新三章，有点迫不及待了。不过讲得确实不错，期待后面更精彩的内容。想请教一个问题，你怎么看待openstack和k8s的关系，哪个技术门槛更高，为什么现在公司更倾向于用k8s来实现自己的云，而openstack有被k8s取代的趋势。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 当然是openstack门槛更高。90%用户要的是paas。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-14 07:42:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/77/02/5b501b89.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>TONNY</span>
  </div>
  <div class="_2_QraFYR_0">解决拉取google镜像问题，有两种方式推荐<br>1. 拉取hub.docker上gcrxio同步的k8s镜像到本地修改repository为k8s.gcr.io后，使用kubeadmin顺利安装<br>2. 使用阿里的容器镜像服务，拉取hub.docker上gcrxio同步的k8s镜像推送到你的镜像库中。安装时，kubeadm init with a configuration file，在configuration file中修改相关的镜像地址为阿里容器服务的镜像地址</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-15 10:10:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4e/60/0d5aa340.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>gogo</span>
  </div>
  <div class="_2_QraFYR_0">老师，您好，我现在对公有云、私有云、容器云之间的区别和联系不太清楚，网上查了些资料还上没搞清楚，能帮忙简单描述下吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 放公网上给别人用的，放私网里给自己用的。提供容器服务的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-14 11:19:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83ercmNryEicqDS73icpUu7W0BnZ7ZIia6jR7kdVMIzH0q1d7L8EKAYWeTJcribibGcHnJzpsjRFxAe26egQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>pytimer</span>
  </div>
  <div class="_2_QraFYR_0">制作证书的工具: cfssl openssl easyrsa</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞。而是这些工具的共同点就是，难用，不够傻瓜……</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-14 07:02:20</div>
  </div>
</div>
</div>
</li>
</ul>