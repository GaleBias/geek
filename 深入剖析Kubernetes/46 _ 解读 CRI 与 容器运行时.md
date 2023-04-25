<audio title="46 _ 解读 CRI 与 容器运行时" src="https://static001.geekbang.org/resource/audio/61/6b/6151026f3abcb3afbb766c2b38ca086b.mp3" controls="controls"></audio> 
<p>你好，我是张磊。今天我和你分享的主题是：解读 CRI 与 容器运行时。</p><p>在上一篇文章中，我为你详细讲解了 kubelet 的工作原理和 CRI 的来龙去脉。在今天这篇文章中，我们就来进一步地、更深入地了解一下 CRI 的设计与工作原理。</p><p>首先，我们先来简要回顾一下有了 CRI 之后，Kubernetes 的架构图，如下所示。</p><p><img src="https://static001.geekbang.org/resource/image/70/38/7016633777ec41da74905bfb91ae7b38.png?wh=1940*1183" alt=""><br>
在上一篇文章中我也提到了，CRI 机制能够发挥作用的核心，就在于每一种容器项目现在都可以自己实现一个 CRI shim，自行对 CRI 请求进行处理。这样，Kubernetes 就有了一个统一的容器抽象层，使得下层容器运行时可以自由地对接进入 Kubernetes 当中。</p><p>所以说，这里的 CRI shim，就是容器项目的维护者们自由发挥的“场地”了。而除了 dockershim之外，其他容器运行时的 CRI shim，都是需要额外部署在宿主机上的。</p><p>举个例子。CNCF 里的 containerd 项目，就可以提供一个典型的 CRI shim 的能力，即：将Kubernetes 发出的 CRI 请求，转换成对 containerd 的调用，然后创建出 runC 容器。而 runC项目，才是负责执行我们前面讲解过的设置容器 Namespace、Cgroups和chroot 等基础操作的组件。所以，这几层的组合关系，可以用如下所示的示意图来描述。</p><!-- [[[read_end]]] --><p><img src="https://static001.geekbang.org/resource/image/62/3d/62c591c4d832d44fed6f76f60be88e3d.png?wh=916*700" alt=""><br>
<strong>而作为一个 CRI shim，containerd 对 CRI 的具体实现，又是怎样的呢？</strong></p><p>我们先来看一下 CRI 这个接口的定义。下面这幅示意图，就展示了 CRI 里主要的待实现接口。</p><p><img src="https://static001.geekbang.org/resource/image/f7/16/f7e86505c09239b80ad05aecfb032e16.png?wh=980*576" alt=""><br>
具体地说，<strong>我们可以把 CRI 分为两组：</strong></p><ul>
<li>
<p>第一组，是 RuntimeService。它提供的接口，主要是跟容器相关的操作。比如，创建和启动容器、删除容器、执行 exec 命令等等。</p>
</li>
<li>
<p>而第二组，则是 ImageService。它提供的接口，主要是容器镜像相关的操作，比如拉取镜像、删除镜像等等。</p>
</li>
</ul><p>关于容器镜像的操作比较简单，所以我们就暂且略过。接下来，我主要为你讲解一下RuntimeService部分。</p><p><strong>在这一部分，CRI 设计的一个重要原则，就是确保这个接口本身，只关注容器，不关注 Pod。</strong>这样做的原因，也很容易理解。</p><p><strong>第一</strong>，Pod 是 Kubernetes 的编排概念，而不是容器运行时的概念。所以，我们就不能假设所有下层容器项目，都能够暴露出可以直接映射为 Pod 的 API。</p><p><strong>第二</strong>，如果 CRI 里引入了关于 Pod 的概念，那么接下来只要 Pod API 对象的字段发生变化，那么CRI 就很有可能需要变更。而在 Kubernetes 开发的前期，Pod 对象的变化还是比较频繁的，但对于CRI 这样的标准接口来说，这个变更频率就有点麻烦了。</p><p>所以，在 CRI 的设计里，并没有一个直接创建 Pod 或者启动 Pod 的接口。</p><p>不过，相信你也已经注意到了，CRI 里还是有一组叫作RunPodSandbox 的接口的。</p><p>这个 PodSandbox，对应的并不是 Kubernetes 里的 Pod API 对象，而只是抽取了 Pod 里的一部分与容器运行时相关的字段，比如HostName、DnsConfig、CgroupParent 等。所以说，PodSandbox 这个接口描述的，其实是 Kubernetes 将 Pod 这个概念映射到容器运行时层面所需要的字段，或者说是一个Pod 对象子集。</p><p>而作为具体的容器项目，你就需要自己决定如何使用这些字段来实现一个 Kubernetes 期望的 Pod模型。这里的原理，可以用如下所示的示意图来表示清楚。</p><p><img src="https://static001.geekbang.org/resource/image/d9/61/d9fb7404c5dc9e0b5c902f74df9d7a61.png?wh=1825*824" alt=""><br>
比如，当我们执行 kubectl run 创建了一个名叫 foo 的、包括了 A、B 两个容器的 Pod 之后。这个Pod 的信息最后来到 kubelet，kubelet 就会按照图中所示的顺序来调用 CRI 接口。</p><p>在具体的 CRI shim 中，这些接口的实现是可以完全不同的。比如，如果是 Docker 项目，dockershim 就会创建出一个名叫 foo 的 Infra容器（pause 容器），用来“hold”住整个 Pod 的 Network Namespace。</p><p>而如果是基于虚拟化技术的容器，比如 Kata Containers 项目，它的 CRI 实现就会直接创建出一个轻量级虚拟机来充当 Pod。</p><p>此外，需要注意的是，在 RunPodSandbox 这个接口的实现中，你还需要调用networkPlugin.SetUpPod(…) 来为这个 Sandbox 设置网络。这个 SetUpPod(…) 方法，实际上就在执行 CNI 插件里的add(…) 方法，也就是我在前面为你讲解过的 CNI 插件为 Pod 创建网络，并且把 Infra 容器加入到网络中的操作。</p><blockquote>
<p>备注：这里，你可以再回顾下第34篇文章<a href="https://time.geekbang.org/column/article/67351">《Kubernetes网络模型与CNI网络插件》</a>中的相关内容。</p>
</blockquote><p>接下来，kubelet 继续调用 CreateContainer 和 StartContainer 接口来创建和启动容器 A、B。对应到 dockershim里，就是直接启动A，B两个 Docker 容器。所以最后，宿主机上会出现三个 Docker 容器组成这一个 Pod。</p><p>而如果是 Kata Containers 的话，CreateContainer和StartContainer接口的实现，就只会在前面创建的轻量级虚拟机里创建两个 A、B 容器对应的 Mount Namespace。所以，最后在宿主机上，只会有一个叫作 foo 的轻量级虚拟机在运行。关于像 Kata Containers 或者 gVisor 这种所谓的安全容器项目，我会在下一篇文章中为你详细介绍。</p><p>除了上述对容器生命周期的实现之外，CRI shim 还有一个重要的工作，就是如何实现 exec、logs 等接口。这些接口跟前面的操作有一个很大的不同，就是这些gRPC 接口调用期间，kubelet 需要跟容器项目维护一个长连接来传输数据。这种 API，我们就称之为Streaming API。</p><p>CRI shim 里对 Streaming API 的实现，依赖于一套独立的 Streaming Server 机制。这一部分原理，可以用如下所示的示意图来为你描述。</p><p><img src="https://static001.geekbang.org/resource/image/a8/ef/a8e7ff6a6b0c9591a0a4f2b8e9e9bdef.png?wh=1937*695" alt=""><br>
可以看到，当我们对一个容器执行 kubectl exec 命令的时候，这个请求首先交给 API Server，然后 API Server 就会调用 kubelet 的 Exec API。</p><p>这时，kubelet就会调用 CRI 的 Exec 接口，而负责响应这个接口的，自然就是具体的 CRI shim。</p><p>但在这一步，CRI shim 并不会直接去调用后端的容器项目（比如 Docker ）来进行处理，而只会返回一个 URL 给 kubelet。这个 URL，就是该 CRI shim 对应的 Streaming Server 的地址和端口。</p><p>而 kubelet 在拿到这个 URL 之后，就会把它以 Redirect 的方式返回给 API Server。所以这时候，API Server 就会通过重定向来向 Streaming Server 发起真正的 /exec 请求，与它建立长连接。</p><p>当然，这个 Streaming Server 本身，是需要通过使用 SIG-Node 为你维护的 Streaming API 库来实现的。并且，Streaming Server 会在 CRI shim 启动时就一起启动。此外，Stream Server 这一部分具体怎么实现，完全可以由 CRI shim 的维护者自行决定。比如，对于Docker 项目来说，dockershim 就是直接调用 Docker 的 Exec API 来作为实现的。</p><p>以上，就是CRI 的设计以及具体的工作原理了。</p><h1>总结</h1><p>在本篇文章中，我为你详细解读了 CRI 的设计和具体工作原理，并为你梳理了实现CRI 接口的核心流程。</p><p>从这些讲解中不难看出，CRI 这个接口的设计，实际上还是比较宽松的。这就意味着，作为容器项目的维护者，我在实现 CRI 的具体接口时，往往拥有着很高的自由度，这个自由度不仅包括了容器的生命周期管理，也包括了如何将 Pod 映射成为我自己的实现，还包括了如何调用 CNI 插件来为 Pod 设置网络的过程。</p><p>所以说，当你对容器这一层有特殊的需求时，我一定优先建议你考虑实现一个自己的 CRI shim ，而不是修改 kubelet 甚至容器项目的代码。这样通过插件的方式定制 Kubernetes 的做法，也是整个 Kubernetes 社区最鼓励和推崇的一个最佳实践。这也正是为什么像 Kata Containers、gVisor 甚至虚拟机这样的“非典型”容器，都可以无缝接入到 Kubernetes 项目里的重要原因。</p><h1>思考题</h1><p>请你思考一下，我前面讲解过的Device Plugin 为容器分配的 GPU 信息，是通过 CRI 的哪个接口传递给 dockershim，最后交给 Docker API 的呢？</p><p>感谢你的收听，欢迎你给我留言，也欢迎分享给更多的朋友一起阅读。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/ac/c8/4b1c0d40.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>勤劳的小胖子-libo</span>
  </div>
  <div class="_2_QraFYR_0">DevicePlugin中的allocate函数是是在container creating的时候被调用，从而device plugin可以执行特定的操作，比如attach设备以及驱动目录。<br>所以应该是使用到了：<br>Createcontainer()这个接口<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-11 09:01:41</div>
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
  <div class="_2_QraFYR_0">kubelet 可以直接对接contained ? 中间不需要额外实现cri shim ？还是containerd 中已经集成了cri shim？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 已经集成啦</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-08 01:33:11</div>
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
  <div class="_2_QraFYR_0">containerd应该会把请求交给contained-shim，然后再调runC吧</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-16 09:50:27</div>
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
  <div class="_2_QraFYR_0">调度器将Pod调度到某一个Node上后，Node上的Kubelet就需要负责将这个Pod拉起来。在Kuberentes社区中，Kubelet以及CRI相关的内容，都属于SIG-Node。<br><br>Kubelet也是通过控制循环来完成各种工作的。kubelet调用下层容器运行时通过一组叫作CRI的gRPC接口来间接执行的。通过CRI， kubelet与具体的容器运行时解耦。在目前的kubelet实现里，内置了dockershim这个CRI的实现，但这部分实现未来肯定会被kubelet移除。未来更普遍的方案是在宿主机上安装负责响应的CRI组件（CRI shim），kubelet负责调用CRI shim，CRI shim把具体的请求“翻译”成对后端容器项目的请求或者操作 。<br><br>不同的CRI shim有不同的容器实现方式，例如：创建了一个名叫foo的、包括了A、B两个容器的Pod<br><br>Docker: 创建出一个名叫foo的Infra容器来hold住整个pod，接着启动A，B两个Docker容器。所以最后，宿主机上会出现三个Docker容器组成这一个Pod<br><br>Kata container: 创建出一个轻量级的虚拟机来hold住整个pod，接着创建A、B容器对应的 Mount Namespace。所以，最后在宿主机上，只会有一个叫做foo的轻量级虚拟机在运行<br><br><br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-22 19:56:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/9b/a8/6a391c66.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Leon📷</span>
  </div>
  <div class="_2_QraFYR_0">docker shim 是不是可以理解成remote+CRI shim的一个k8s内部集成的一种实现</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-16 10:34:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/bb/f6/af833125.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Vinsec</span>
  </div>
  <div class="_2_QraFYR_0">天冷适合搞学习 打卡</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-07 16:23:33</div>
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
  <div class="_2_QraFYR_0">Containerd 中的cri-shim和用来控制runC的containerd-shim有什么区别呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-13 14:19:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/76/5c/416bcce6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>郑然</span>
  </div>
  <div class="_2_QraFYR_0">老师请教两个问题: <br>1. 为什么kubelet要给apiserver返回redirect url呢? 这样做有什么特殊考虑吗?<br>2. 镜像服务, 以及下载完镜像之后, 如何和createcontainer接口发生关联的, 这块的细节能讲讲吗?</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-07 08:53:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/7c/59/dc9bbb21.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Join</span>
  </div>
  <div class="_2_QraFYR_0">对CRI的认识更进一层了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-15 09:19:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/3b/ba/3b30dcde.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>窝窝头</span>
  </div>
  <div class="_2_QraFYR_0">CreateContainer或者RunPodSandbox吧</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-03 19:22:37</div>
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
  <div class="_2_QraFYR_0">第四十四课:解读CRI与容器运行时<br>CRI机制能够发挥作用的核心，就在于每个容器项目现在都能自己实现一个CRI shim，自行对CRI请求进行处理。<br><br>CRI可以分为两组，第一组是RuntimeService。它提供的接口主要是跟容器相关的操作，比如创建或启动容器，删除容器，和执行exec命令等。这组的设计原则是确保这个接口本身只关注容器。第二组是ImageService，它提供的接口主要是容器镜像相关的操作，比如拉取和删除镜像等。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-05 06:25:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/dd/eb/80f9d212.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lttzzlll</span>
  </div>
  <div class="_2_QraFYR_0">k8s中是否可以同时存在 docker, containerd, gVisor 等不同的容器？根据label分配不同的任务。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-13 15:41:39</div>
  </div>
</div>
</div>
</li>
</ul>