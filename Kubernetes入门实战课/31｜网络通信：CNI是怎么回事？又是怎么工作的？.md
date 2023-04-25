<audio title="31｜网络通信：CNI是怎么回事？又是怎么工作的？" src="https://static001.geekbang.org/resource/audio/16/ae/16b4ecda9177cc7d0ef367055368e7ae.mp3" controls="controls"></audio> 
<p>你好，我是Chrono。</p><p>到现在，我们对Kubernetes已经非常熟悉了，它是一个集群操作系统，能够管理大量计算节点和运行在里面的应用。不过，还有一个很重要的基础知识我们还没有学习，那就是“网络通信”。</p><p>早在“入门篇”的<a href="https://time.geekbang.org/column/article/528692">第6讲</a>里，我们就简单介绍过Docker的网络模式，然后在“中级篇”的<a href="https://time.geekbang.org/column/article/534762">第17讲</a>，我们又为Kubernetes安装了一个网络插件Flannel。这些都与网络相关，但也只是浅尝辄止，并没有太多深究。</p><p>如果你是一个喜欢刨根问底的人，会不会很好奇：Flannel到底是如何工作的呢？它为什么能够让Kubernetes集群正常通信呢？还有没有其他网络插件呢？</p><p>今天我们就来聊一下这个话题，讲讲Kubernetes的网络接口标准CNI，以及Calico、Cilium等性能更好的网络插件。</p><h2>Kubernetes的网络模型</h2><p>在学习Kubernetes的网络之前，我们还是要先简单回顾一下Docker的网络知识。</p><p>你对Docker的null、host和bridge三种网络模式还有印象吗？这里我重新画了一张图，描述了Docker里最常用的bridge网络模式：</p><p><img src="https://static001.geekbang.org/resource/image/0b/85/0b7954a362b9e04db8b588fbed5b7185.jpg?wh=1920x1148" alt="图片"></p><p>Docker会创建一个名字叫“docker0”的网桥，默认是私有网段“172.17.0.0/16”。每个容器都会创建一个虚拟网卡对（veth pair），两个虚拟网卡分别“插”在容器和网桥上，这样容器之间就可以互联互通了。</p><!-- [[[read_end]]] --><p>Docker的网络方案简单有效，但问题是它只局限在单机环境里工作，跨主机通信非常困难（需要做端口映射和网络地址转换）。</p><p>针对Docker的网络缺陷，Kubernetes提出了一个自己的网络模型“<strong>IP-per-pod</strong>”，能够很好地适应集群系统的网络需求，它有下面的这4点基本假设：</p><ul>
<li>集群里的每个Pod都会有唯一的一个IP地址。</li>
<li>Pod里的所有容器共享这个IP地址。</li>
<li>集群里的所有Pod都属于同一个网段。</li>
<li>Pod直接可以基于IP地址直接访问另一个Pod，不需要做麻烦的网络地址转换（NAT）。</li>
</ul><p>我画了一张Kubernetes网络模型的示意图，你可以看一下：</p><p><img src="https://static001.geekbang.org/resource/image/81/6c/81d67c2f0a6e97b847c306c16048c06c.jpg?wh=1920x1114" alt="图片"></p><p>这个网络让Pod摆脱了主机的硬限制，是一个“平坦”的网络模型，很好理解，通信自然也非常简单。</p><p>因为Pod都具有独立的IP地址，相当于一台虚拟机，而且直连互通，也就可以很容易地实施域名解析、负载均衡、服务发现等工作，以前的运维经验都能够直接使用，对应用的管理和迁移都非常友好。</p><h2>什么是CNI</h2><p>Kubernetes定义的这个网络模型很完美，但要把这个模型落地实现就不那么容易了。所以Kubernetes就专门制定了一个标准：<strong>CNI</strong>（Container Networking Interface）。</p><p>CNI为网络插件定义了一系列通用接口，开发者只要遵循这个规范就可以接入Kubernetes，为Pod创建虚拟网卡、分配IP地址、设置路由规则，最后就能够实现“IP-per-pod”网络模型。</p><p>依据实现技术的不同，CNI插件可以大致上分成“<strong>Overlay</strong>”“<strong>Route</strong>”和“<strong>Underlay</strong>”三种。</p><p><strong>Overlay</strong>的原意是“覆盖”，是指它构建了一个工作在真实底层网络之上的“逻辑网络”，把原始的Pod网络数据封包，再通过下层网络发送出去，到了目的地再拆包。因为这个特点，它对底层网络的要求低，适应性强，缺点就是有额外的传输成本，性能较低。</p><p><strong>Route</strong>也是在底层网络之上工作，但它没有封包和拆包，而是使用系统内置的路由功能来实现Pod跨主机通信。它的好处是性能高，不过对底层网络的依赖性比较强，如果底层不支持就没办法工作了。</p><p><strong>Underlay</strong>就是直接用底层网络来实现CNI，也就是说Pod和宿主机都在一个网络里，Pod和宿主机是平等的。它对底层的硬件和网络的依赖性是最强的，因而不够灵活，但性能最高。</p><p>自从2015年CNI发布以来，由于它的接口定义宽松，有很大的自由发挥空间，所以社区里就涌现出了非常多的网络插件，我们之前在<a href="https://time.geekbang.org/column/article/534762">第17讲</a>里提到的Flannel就是其中之一。</p><p><strong>Flannel</strong>（<a href="https://github.com/flannel-io/flannel/">https://github.com/flannel-io/flannel/</a>）由CoreOS公司（已被Redhat收购）开发，最早是一种Overlay模式的网络插件，使用UDP和VXLAN技术，后来又用Host-Gateway技术支持了Route模式。Flannel简单易用，是Kubernetes里最流行的CNI插件，但它在性能方面表现不是太好，所以一般不建议在生产环境里使用。</p><p>现在还有两个常用CNI插件：Calico、Cilium，我们做个简略的介绍。</p><p><img src="https://static001.geekbang.org/resource/image/a9/7a/a96dd70ef544e4a69ff6f705a79acb7a.png?wh=1920x746" alt="图片"></p><p><strong>Calico</strong>（<a href="https://github.com/projectcalico/calico">https://github.com/projectcalico/calico</a>）是一种Route模式的网络插件，使用BGP协议（Border Gateway Protocol）来维护路由信息，性能要比Flannel好，而且支持多种网络策略，具备数据加密、安全隔离、流量整形等功能。</p><p><strong>Cilium</strong>（<a href="https://github.com/cilium/cilium">https://github.com/cilium/cilium</a>）是一个比较新的网络插件，同时支持Overlay模式和Route模式，它的特点是深度使用了Linux eBPF技术，在内核层次操作网络数据，所以性能很高，可以灵活实现各种功能。在2021年它加入了CNCF，成为了孵化项目，是非常有前途的CNI插件。</p><h2>CNI插件是怎么工作的</h2><p>Flannel比较简单，我们先以它为例看看CNI在Kubernetes里的工作方式。</p><p>这里必须要说明一点，计算机网络很复杂，有IP地址、MAC地址、网段、网卡、网桥、路由等许许多多的概念，而且数据会流经多个设备，理清楚脉络比较麻烦，今天我们会做一个大概的描述，不会讲那些太底层的细节。</p><p>我们先来在实验环境里用Deployment创建3个Nginx Pod，作为研究对象：</p><pre><code class="language-plain">kubectl create deploy ngx-dep --image=nginx:alpine --replicas=3
</code></pre><p>使用命令 <code>kubectl get pod</code> 可以看到，有两个Pod运行在master节点上，IP地址分别是“10.10.0.3”“10.10.0.4”，另一个Pod运行在worker节点上，IP地址是“10.10.1.77”：</p><p><img src="https://static001.geekbang.org/resource/image/e6/b8/e63ecfb640e7a032a27817c0b7ff49b8.png?wh=1920x281" alt="图片"></p><p>Flannel默认使用的是基于VXLAN的Overlay模式，整个集群的网络结构我画了一张示意图，你可以对比一下Docker的网络结构：</p><p><img src="https://static001.geekbang.org/resource/image/96/b7/96ffd51d7c843596f6736d23467888b7.jpg?wh=1920x1037" alt="图片"></p><p>从单机的角度来看的话，Flannel的网络结构和Docker几乎是一模一样的，只不过网桥换成了“cni0”，而不是“docker0”。</p><p>接下来我们来操作一下，看看Pod里的虚拟网卡是如何接入cni0网桥的。</p><p>在Pod里执行命令 <code>ip addr</code> 就可以看到它里面的虚拟网卡“eth0”：</p><p><img src="https://static001.geekbang.org/resource/image/b8/84/b85c5c010689b1e3b7075aa2e0d2bc84.png?wh=1920x416" alt="图片"></p><p>你需要注意它的形式，第一个数字“3”是序号，意思是第3号设备，“@if45”就是它另一端连接的虚拟网卡，序号是45。</p><p>因为这个Pod的宿主机是master，我们就要登录到master节点，看看这个节点上的网络情况，同样还是用命令 <code>ip addr</code>：</p><p><img src="https://static001.geekbang.org/resource/image/bb/e9/bb8342853bab79aea1842eae5f48bde9.png?wh=1920x389" alt="图片"></p><p>这里就可以看到宿主机（master）节点上的第45号设备了，它的名字是 <code>veth41586979@if3</code>，“veth”表示它是一个虚拟网卡，而后面的“@if3”就是Pod里对应的3号设备，也就是“eth0”网卡了。</p><p><strong>那么“cni0”网桥的信息该怎么查看呢？这需要在宿主机（master）上使用命令 <code>brctl show</code>：</strong></p><p><img src="https://static001.geekbang.org/resource/image/13/b3/13563817d53f094fe6fd6d734c7c49b3.png?wh=1920x387" alt="图片"></p><p>从这张截图里，你可以发现“cni0”网桥上有4个虚拟网卡，第三个就是“veth41586979”，所以这个网卡就被“插”在了“cni0”网桥上，然后因为虚拟网卡的“结对”特性，Pod也就连上了“cni0”网桥。</p><p>单纯用Linux命令不太容易看清楚网卡和网桥的联系，所以我把它们整合在了下面的图里，加上了虚线标记，这样你就能更清晰地理解Pod、veth和cni0的引用关系了：</p><p><img src="https://static001.geekbang.org/resource/image/e3/14/e3c4f523cee0e39b74e94d1b96e5a014.jpg?wh=1920x1176" alt="图片"></p><p>使用同样的方式，你可以知道另一个Pod “10.10.0.4”的网卡是 <code>veth2b3ef56d@if3</code>，它也在“cni0”网桥上，所以借助这个网桥，本机的Pod就可以直接通信。</p><p>弄清楚了本机网络，我们再来看跨主机的网络，它的关键是节点的路由表，用命令 <code>route</code> 查看：</p><p><img src="https://static001.geekbang.org/resource/image/df/39/df13160c3885b59233c0d90823cde239.png?wh=1920x489" alt="图片"></p><p>它告诉我们有这些信息：</p><ul>
<li>10.10.0.0/24网段的数据，都要走cni0设备，也就是“cni0”网桥。</li>
<li>10.10.1.0/24网段的数据，都要走flannel.1设备，也就是Flannel。</li>
<li>192.168.10.0/24网段的数据，都要走ens160设备，也就是我们宿主机的网卡。</li>
</ul><p>假设我们要从master节点的“10.10.0.3”访问worker节点的“10.10.1.77”，因为master节点的“cni0”网桥管理的只是“10.10.0.0/24”这个网段，所以按照路由表，凡是“10.10.1.0/24”都要让flannel.1来处理，这样就进入了Flannel插件的工作流程。</p><p>然后Flannel就要来决定应该如何把数据发到另一个节点，在各种表里去查询。因为这个过程比较枯燥，我就不详细说了，你可以参考下面的示意图，用到的命令有 <code>ip neighbor</code>、<code>bridge fdb</code> 等等：</p><p><img src="https://static001.geekbang.org/resource/image/8e/7b/8e2f69cb47cd0bf32e20a8420e9b577b.png?wh=1920x1112" alt="图片"></p><p>Flannel得到的结果就是要把数据发到“192.168.10.220”，也就是worker节点，所以它就会在原始网络包前面加上这些额外的信息，封装成VXLAN报文，用“ens160”网卡发出去，worker节点收到后再拆包，执行类似的反向处理，就可以把数据交给真正的目标Pod了。</p><h2>使用Calico网络插件</h2><p>看到这里，是不是觉得Flannel的Overlay处理流程非常复杂，绕来绕去很容易让人头晕，那下面我们就来看看另一个Route模式的插件Calico。</p><p>你可以在Calico的网站（<a href="https://www.tigera.io/project-calico/">https://www.tigera.io/project-calico/</a>）上找到它的安装方式，我选择的是“本地自助安装（Self-managed on-premises）”，可以直接下载YAML文件：</p><pre><code class="language-plain">wget https://projectcalico.docs.tigera.io/manifests/calico.yaml
</code></pre><p>由于Calico使用的镜像较大，为了加快安装速度，可以考虑在每个节点上预先使用 <code>docker pull</code> 拉取镜像：</p><pre><code class="language-plain">docker pull calico/cni:v3.23.1
docker pull calico/node:v3.23.1
docker pull calico/kube-controllers:v3.23.1
</code></pre><p>Calico的安装非常简单，只需要用 <code>kubectl apply</code> 就可以（记得安装之前最好把Flannel删除）：</p><pre><code class="language-plain">kubectl apply -f calico.yaml
</code></pre><p>安装之后我们来查看一下Calico的运行状态，注意它也是在“kube-system”名字空间：</p><p><img src="https://static001.geekbang.org/resource/image/98/95/983c6a271a394d0febc83f21c108e195.png?wh=1520x304" alt="图片"></p><p>我们仍然创建3个Nginx Pod来做实验：</p><pre><code class="language-plain">kubectl create deploy ngx-dep --image=nginx:alpine --replicas=3
</code></pre><p>我们会看到master节点上有两个Pod，worker节点上有一个Pod，但它们的IP地址与刚才Flannel的明显不一样了，分别是“10.10.219.*”和“10.10.171.*”，这说明Calico的IP地址分配策略和Flannel是不同的：</p><p><img src="https://static001.geekbang.org/resource/image/aa/81/aa8917fd298cc633fbdf190e3a767581.png?wh=1920x262" alt="图片"></p><p>然后我们来看看Pod里的网卡情况，你会发现虽然还是有虚拟网卡，但宿主机上的网卡名字变成了 <code>calica17a7ab6ab@if4</code>，而且并没有连接到“cni0”网桥上：</p><p><img src="https://static001.geekbang.org/resource/image/75/69/756e9ea5e78489c386219722f302c969.jpg?wh=1920x1236" alt="图片"></p><p>这是不是很奇怪？</p><p>其实这是Calico的工作模式导致的正常现象。因为Calico不是Overlay模式，而是Route模式，所以它就没有用Flannel那一套，而是<strong>在宿主机上创建路由规则，让数据包不经过网桥直接“跳”到目标网卡去</strong>。</p><p>来看一下节点上的路由表就能明白：</p><p><img src="https://static001.geekbang.org/resource/image/0f/d3/0f94b581ff1e72b08103bfbe900e53d3.png?wh=1920x663" alt="图片"></p><p>假设Pod A“10.10.219.67”要访问Pod B“10.10.219.68”，那么查路由表，知道要走“cali051dd144e34”这个设备，而它恰好就在Pod B里，所以数据就会直接进Pod B的网卡，省去了网桥的中间步骤。</p><p>Calico的网络架构我也画了一张示意图，你可以再对比Flannel来学习：</p><p><img src="https://static001.geekbang.org/resource/image/yy/c7/yyb9c0ee93730542ebb5475a734991c7.jpg?wh=1920x1012" alt="图片"></p><p>至于在Calico里跨主机通信是如何路由的，你完全可以对照着路由表，一步步地“跳”到目标Pod去（提示：tunl0设备）。</p><h2>小结</h2><p>好说了这么多，你应该看到了，Kubernetes的整个网络数据传输过程有大量的细节，非常多的环节都参与其中，想把它彻底弄明白还真不是件容易的事情。</p><p>不过好在CNI通过“依赖倒置”的原则把这些工作都交给插件去解决了，不管下层是什么样的环境，不管插件是怎么实现的，我们在Kubernetes集群里只会有一个干净、整洁的网络空间。</p><p>我来简单小结一下今天的内容：</p><ol>
<li>Kubernetes使用的是“IP-per-pod”网络模型，每个Pod都会有唯一的IP地址，所以简单易管理。</li>
<li>CNI是Kubernetes定义的网络插件接口标准，按照实现方式可以分成“Overlay”“Route”和“Underlay”三种，常见的CNI插件有Flannel、Calico和Cilium。</li>
<li>Flannel支持Overlay模式，它使用了cni0网桥和flannel.1设备，本机通信直接走cni0，跨主机通信会把原始数据包封装成VXLAN包再走宿主机网卡发送，有性能损失。</li>
<li>Calico支持Route模式，它不使用cni0网桥，而是创建路由规则，把数据包直接发送到目标网卡，所以性能高。</li>
</ol><h2>课下作业</h2><p>最后是课下作业时间，给你留两个思考题：</p><ol>
<li>Kubernetes没有内置网络实现，而是用CNI定义了标准接口，这么做的好处在哪里？</li>
<li>你对Flannel和Calico这两个网络插件的工作模式有什么样的看法？</li>
</ol><p>欢迎在留言区发言参与讨论，这是最后一节知识点学习课，下节课我们进入回顾总结，曙光就在前方，期待你在马上到来的实操课和视频课中见证自己的成长。下节课见。</p><p><img src="https://static001.geekbang.org/resource/image/2a/18/2a93942d9b761589a7b72282e15ba318.jpg?wh=1920x2789" alt="图片"></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/99/87/98ebb20e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>dao</span>
  </div>
  <div class="_2_QraFYR_0">试着思考了一下：<br>1，定义规范&#47;标准，也就是接口（interface），把具体实现&#47;扩展交给社区&#47;第三方，然后使用插件（addon）的方式在 Kubernets 里应用，这真是个省力的方式！不需要去关注千变万化的底层运行环境。<br><br>2，Flannel 默认是 Overlay 模式基于Linux VxLan，数据包在跨节点间传输有封包和拆包的额外步骤，同节点的 Pod 间数据传输直接通过虚拟网桥，比如 cni0；不同节点的 Pod 间的数据传输需要借助 Flannel.1 (VTEP virtual tunnel endpoint) 分发。<br>Calico 也有多种工作模式，默认是 IPinIP，同节点的 Pod 间直接通过虚拟网卡结合路由表传输，跨节点间的 Pod 需要IP层的封装，数据包通过IP隧道传输，如 tunl0。多节点间的路由通过BGP协议共享。<br><br>在节点上抓包观察同节点 Pod 和 跨节点 Pod 数据传输：<br>```bash<br>kubectl exec -it ngx-dep-bfbb5f64b-87sm4 -- curl 10.244.225.25<br>sudo tcpdump -n -s0 -i any host 10.244.225.25<br>kubectl exec -it ngx-dep-bfbb5f64b-87sm4 -- curl 10.244.185.207<br>sudo tcpdump -n -s0 -i any host 10.244.185.207<br>```</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-04 12:59:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/f7/b1/982ea185.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Frank</span>
  </div>
  <div class="_2_QraFYR_0">生产环境建议用哪个网络插件呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 也不能一概而论，如果性能够用flannel就可以了，如果要强调性能还有其他的网络管理需求就可以用calico。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-09 17:45:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/30/65/d6/20670fd5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Obscure</span>
  </div>
  <div class="_2_QraFYR_0">网络基础不行，这一节内容基本看不懂。。。咋整。。。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 没事，我的网络基础也很差，如果不是专门做网络的了解个大概就行了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-07 22:32:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/63/d5/a300899a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>杨丁</span>
  </div>
  <div class="_2_QraFYR_0">老师好，您画图用的啥工具啊</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">编辑回复: 画分析图用的亿图图示或者keynote，思维导图用的Xmind</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-06 18:10:44</div>
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
  <div class="_2_QraFYR_0">hostNetwork是不是属于underlay？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 它是kubernetes的一个属性定义，与cni没有关系。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-10 20:47:10</div>
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
  <div class="_2_QraFYR_0">引用原文 ： &quot;这里我重新画了一张图，描述了 Docker 里最常用的 bridge 网络模式：&quot;<br>该图中的最下方的位置，ens160是表示啥？请老师帮忙解答下，谢谢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是虚拟机的网卡。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-29 14:23:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2a/3b/29/0f86235e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>明月夜</span>
  </div>
  <div class="_2_QraFYR_0">老师好，在节点上（在Pod外）好像也是能访问某个Pod的ip的，这种情况的IP寻址是不是也和在Pod里面的寻址一样？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 应该是一样的，都是一个网络模型，这个还没仔细看过。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-18 22:43:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/42/df/a034455d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>罗耀龙@坐忘</span>
  </div>
  <div class="_2_QraFYR_0">按照老师的教程，成功从flannel换成calico了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-03 08:24:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/42/df/a034455d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>罗耀龙@坐忘</span>
  </div>
  <div class="_2_QraFYR_0">老师，calico的版本是v3.24.1了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 几个小的版本差异应该不是很大，对于学习来说不用太在意。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-03 00:59:04</div>
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
  <div class="_2_QraFYR_0">请教老师一个问题：<br>Q1：怎么修改Prometheus的镜像源格式？<br>第30课的问题。<br>我的虚拟机上：prometheus-adapter的状态是CrashLoopBackOff。<br>针对这个问题，老师的回答是：“exec format error”这样的感觉都是镜像的格式不对，比如amd64&#47;arm64，换个镜像试试。<br><br>我是按照老师的如下要求修改的：<br>image: chronolaw&#47;kube-state-metrics:v2.5.0<br>image: chronolaw&#47;prometheus-adapter:v0.9.1<br><br>我的电脑是win10, intel的CPU，应该是x86系列，请问应该怎么修改上面的image语句？<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 就是改YAML 里的image字段，要看docker hub上的镜像是否支持硬件体系，一般来说x86的都支持，arm64的会少一些。<br><br>可以参考一下其他同学的留言。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-02 15:22:53</div>
  </div>
</div>
</div>
</li>
</ul>