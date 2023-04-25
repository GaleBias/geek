<audio title="10｜自动化的运维管理：探究Kubernetes工作机制的奥秘" src="https://static001.geekbang.org/resource/audio/aa/85/aaf3b0d1d7b19e6f105b88e3a20cf385.mp3" controls="controls"></audio> 
<p>你好，我是Chrono。</p><p>在上一次课里，我们看到容器技术只实现了应用的打包分发，到运维真正落地实施的时候仍然会遇到很多困难，所以就需要用容器编排技术来解决这些问题，而Kubernetes是这个领域的唯一霸主，已经成为了“事实标准”。</p><p>那么，Kubernetes凭什么能担当这样的领军重任呢？难道仅仅因为它是由Google主导开发的吗？</p><p>今天我就带你一起来看看Kubernetes的内部架构和工作机制，了解它能够傲视群雄的秘密所在。</p><h2>云计算时代的操作系统</h2><p>前面我曾经说过，Kubernetes是<strong>一个生产级别的容器编排平台和集群管理系统</strong>，能够创建、调度容器，监控、管理服务器。</p><p>容器是什么？容器是软件，是应用，是进程。服务器是什么？服务器是硬件，是CPU、内存、硬盘、网卡。那么，既可以管理软件，也可以管理硬件，这样的东西应该是什么？</p><p>你也许会脱口而出：这就是一个操作系统（Operating System）！</p><p>没错，从某种角度来看，Kubernetes可以说是一个集群级别的操作系统，主要功能就是资源管理和作业调度。但Kubernetes不是运行在单机上管理单台计算资源和进程，而是运行在多台服务器上管理几百几千台的计算资源，以及在这些资源上运行的上万上百万的进程，规模要大得多。</p><!-- [[[read_end]]] --><p><img src="https://static001.geekbang.org/resource/image/a0/f1/a09709bdf35516603yy377fc599e0af1.png?wh=800x418" alt="图片" title="图片来自网络"></p><p>所以，你可以把Kubernetes与Linux对比起来学习，而这个新的操作系统里自然会有一系列新名词、新术语，你也需要使用新的思维方式来考虑问题，必要的时候还得和过去的习惯“说再见”。</p><p>Kubernetes这个操作系统与Linux还有一点区别你值得注意。Linux的用户通常是两类人：<strong>Dev</strong>和<strong>Ops</strong>，而在Kubernetes里则只有一类人：<strong>DevOps</strong>。</p><p>在以前的应用实施流程中，开发人员和运维人员分工明确，开发完成后需要编写详细的说明文档，然后交给运维去部署管理，两者之间不能随便“越线”。</p><p>而在Kubernetes这里，开发和运维的界限变得不那么清晰了。由于云原生的兴起，开发人员从一开始就必须考虑后续的部署运维工作，而运维人员也需要在早期介入开发，才能做好应用的运维监控工作。</p><p>这就会导致很多Kubernetes的新用户会面临身份的转变，一开始可能会有点困难。不过不用担心，这也非常正常，任何的学习过程都有个适应期，只要过了最初的概念理解阶段就好了。</p><h2>Kubernetes的基本架构</h2><p>操作系统的一个重要功能就是抽象，从繁琐的底层事务中抽象出一些简洁的概念，然后基于这些概念去管理系统资源。</p><p>Kubernetes也是这样，它的管理目标是大规模的集群和应用，必须要能够把系统抽象到足够高的层次，分解出一些松耦合的对象，才能简化系统模型，减轻用户的心智负担。</p><p>所以，Kubernetes扮演的角色就如同一个“大师级别”的系统管理员，具有丰富的集群运维经验，独创了自己的一套工作方式，不需要太多的外部干预，就能够自主实现原先许多复杂的管理工作。</p><p>下面我们就来看看这位资深管理员的“内功心法”。</p><p>Kubernetes官网上有一张架构图，但我觉得不是太清晰、重点不突出，所以另外找了一份（<a href="https://medium.com/@keshiha/k8s-architecture-bb6964767c12">图片来源</a>）。虽然这张图有点“老”，但对于我们初学Kubernetes还是比较合适的。</p><p><img src="https://static001.geekbang.org/resource/image/34/b7/344e0c6dc2141b12f99e61252110f6b7.png?wh=1278x704" alt="图片"></p><p>Kubernetes采用了现今流行的“<strong>控制面/数据面</strong>”（Control Plane / Data Plane）架构，集群里的计算机被称为“<strong>节点</strong>”（Node），可以是实机也可以是虚机，少量的节点用作控制面来执行集群的管理维护工作，其他的大部分节点都被划归数据面，用来跑业务应用。</p><p>控制面的节点在Kubernetes里叫做<strong>Master Node</strong>，一般简称为<strong>Master</strong>，它是整个集群里最重要的部分，可以说是Kubernetes的大脑和心脏。</p><p>数据面的节点叫做<strong>Worker Node</strong>，一般就简称为<strong>Worker</strong>或者<strong>Node</strong>，相当于Kubernetes的手和脚，在Master的指挥下干活。</p><p>Node的数量非常多，构成了一个资源池，Kubernetes就在这个池里分配资源，调度应用。因为资源被“池化”了，所以管理也就变得比较简单，可以在集群中任意添加或者删除节点。</p><p>在这张架构图里，我们还可以看到有一个kubectl，它就是Kubernetes的客户端工具，用来操作Kubernetes，但它位于集群之外，理论上不属于集群。</p><p>你可以使用命令 <code>kubectl get node</code> 来查看Kubernetes的节点状态：</p><pre><code class="language-plain">kubectl get node
</code></pre><p><img src="https://static001.geekbang.org/resource/image/33/38/33937b919cbc5d6fe4b4e0e203415538.png?wh=1472x180" alt="图片"></p><p>可以看到当前的minikube集群里只有一个Master，那Node怎么不见了？</p><p>这是因为Master和Node的划分不是绝对的。当集群的规模较小，工作负载较少的时候，Master也可以承担Node的工作，就像我们搭建的minikube环境，它就只有一个节点，这个节点既是Master又是Node。</p><h2>节点内部的结构</h2><p>Kubernetes的节点内部也具有复杂的结构，是由很多的模块构成的，这些模块又可以分成组件（Component）和插件（Addon）两类。</p><p>组件实现了Kubernetes的核心功能特性，没有这些组件Kubernetes就无法启动，而插件则是Kubernetes的一些附加功能，属于“锦上添花”，不安装也不会影响Kubernetes的正常运行。</p><p>接下来我先来讲讲Master和Node里的组件，然后再捎带提一下插件，理解了它们的工作流程，你就会明白为什么Kubernetes有如此强大的自动化运维能力。</p><h3>Master里的组件有哪些</h3><p>Master里有4个组件，分别是<strong>apiserver</strong>、<strong>etcd</strong>、<strong>scheduler</strong>、<strong>controller-manager</strong>。</p><p><img src="https://static001.geekbang.org/resource/image/33/c6/330e03a66f636657c0d8695397c508c6.jpg?wh=1278x704" alt="图片"></p><p>apiserver是Master节点——同时也是整个Kubernetes系统的唯一入口，它对外公开了一系列的RESTful API，并且加上了验证、授权等功能，所有其他组件都只能和它直接通信，可以说是Kubernetes里的联络员。</p><p>etcd是一个高可用的分布式Key-Value数据库，用来持久化存储系统里的各种资源对象和状态，相当于Kubernetes里的配置管理员。注意它只与apiserver有直接联系，也就是说任何其他组件想要读写etcd里的数据都必须经过apiserver。</p><p>scheduler负责容器的编排工作，检查节点的资源状态，把Pod调度到最适合的节点上运行，相当于部署人员。因为节点状态和Pod信息都存储在etcd里，所以scheduler必须通过apiserver才能获得。</p><p>controller-manager负责维护容器和节点等资源的状态，实现故障检测、服务迁移、应用伸缩等功能，相当于监控运维人员。同样地，它也必须通过apiserver获得存储在etcd里的信息，才能够实现对资源的各种操作。</p><p>这4个组件也都被容器化了，运行在集群的Pod里，我们可以用kubectl来查看它们的状态，使用命令：</p><pre><code class="language-plain">kubectl get pod -n kube-system
</code></pre><p><img src="https://static001.geekbang.org/resource/image/86/99/868380a89077c45c6ac1918794632399.png?wh=1862x504" alt="图片"></p><p>注意命令行里要用 <code>-n kube-system</code> 参数，表示检查“kube-system”名字空间里的Pod，至于名字空间是什么，我们后面会讲到。</p><h3>Node里的组件有哪些</h3><p>Master里的apiserver、scheduler等组件需要获取节点的各种信息才能够作出管理决策，那这些信息该怎么来呢？</p><p>这就需要Node里的3个组件了，分别是<strong>kubelet</strong>、<strong>kube-proxy</strong>、<strong>container-runtime</strong>。</p><p>kubelet是Node的代理，负责管理Node相关的绝大部分操作，Node上只有它能够与apiserver通信，实现状态报告、命令下发、启停容器等功能，相当于是Node上的一个“小管家”。</p><p>kube-proxy的作用有点特别，它是Node的网络代理，只负责管理容器的网络通信，简单来说就是为Pod转发TCP/UDP数据包，相当于是专职的“小邮差”。</p><p>第三个组件container-runtime我们就比较熟悉了，它是容器和镜像的实际使用者，在kubelet的指挥下创建容器，管理Pod的生命周期，是真正干活的“苦力”。</p><p><img src="https://static001.geekbang.org/resource/image/87/35/87bab507ce8381325e85570f3bc1d935.jpg?wh=1278x704" alt="图片"></p><p>我们一定要注意，因为Kubernetes的定位是容器编排平台，所以它没有限定container-runtime必须是Docker，完全可以替换成任何符合标准的其他容器运行时，例如containerd、CRI-O等等，只不过在这里我们使用的是Docker。</p><p>这3个组件中只有kube-proxy被容器化了，而kubelet因为必须要管理整个节点，容器化会限制它的能力，所以它必须在container-runtime之外运行。</p><p>使用 <code>minikube ssh</code> 命令登录到节点后，可以用 <code>docker ps</code> 看到kube-proxy：</p><pre><code class="language-plain">minikube ssh
docker ps |grep kube-proxy
</code></pre><p><img src="https://static001.geekbang.org/resource/image/1f/a4/1f08490fd66c91b4f5d2172d2d93eba4.png?wh=1920x141" alt="图片"></p><p>而kubelet用 <code>docker ps</code> 是找不到的，需要用操作系统的 <code>ps</code> 命令：</p><pre><code class="language-plain">ps -ef|grep kubelet
</code></pre><p><img src="https://static001.geekbang.org/resource/image/ay/28/ayy7246d222fa723f5f4a1f8edc0eb28.png?wh=1920x95" alt="图片"></p><p>现在，我们再把Node里的组件和Master里的组件放在一起来看，就能够明白Kubernetes的大致工作流程了：</p><ul>
<li>每个Node上的kubelet会定期向apiserver上报节点状态，apiserver再存到etcd里。</li>
<li>每个Node上的kube-proxy实现了TCP/UDP反向代理，让容器对外提供稳定的服务。</li>
<li>scheduler通过apiserver得到当前的节点状态，调度Pod，然后apiserver下发命令给某个Node的kubelet，kubelet调用container-runtime启动容器。</li>
<li>controller-manager也通过apiserver得到实时的节点状态，监控可能的异常情况，再使用相应的手段去调节恢复。</li>
</ul><p><img src="https://static001.geekbang.org/resource/image/34/b7/344e0c6dc2141b12f99e61252110f6b7.png?wh=1278x704" alt="图片"></p><p>其实，这和我们在Kubernetes出现之前的操作流程也差不了多少，但Kubernetes的高明之处就在于把这些都抽象化规范化了。</p><p>于是，这些组件就好像是无数个不知疲倦的运维工程师，把原先繁琐低效的人力工作搬进了高效的计算机里，就能够随时发现集群里的变化和异常，再互相协作，维护集群的健康状态。</p><h3>插件（Addons）有哪些</h3><p>只要服务器节点上运行了apiserver、scheduler、kubelet、kube-proxy、container-runtime等组件，就可以说是一个功能齐全的Kubernetes集群了。</p><p>不过就像Linux一样，操作系统提供的基础功能虽然“可用”，但想达到“好用”的程度，还是要再安装一些附加功能，这在Kubernetes里就是插件（Addon）。</p><p>由于Kubernetes本身的设计非常灵活，所以就有大量的插件用来扩展、增强它对应用和集群的管理能力。</p><p>minikube也支持很多的插件，使用命令 <code>minikube addons list</code> 就可以查看插件列表：</p><pre><code class="language-plain">minikube addons list
</code></pre><p><img src="https://static001.geekbang.org/resource/image/db/2f/dbd588f0cca1ffb93a702a6d4c8f4c2f.png?wh=1920x987" alt="图片"></p><p>插件中我个人认为比较重要的有两个：<strong>DNS</strong>和<strong>Dashboard</strong>。</p><p>DNS你应该比较熟悉吧，它在Kubernetes集群里实现了域名解析服务，能够让我们以域名而不是IP地址的方式来互相通信，是服务发现和负载均衡的基础。由于它对微服务、服务网格等架构至关重要，所以基本上是Kubernetes的必备插件。</p><p>Dashboard就是仪表盘，为Kubernetes提供了一个图形化的操作界面，非常直观友好，虽然大多数Kubernetes工作都是使用命令行kubectl，但有的时候在Dashboard上查看信息也是挺方便的。</p><p>你只要在minikube环境里执行一条简单的命令，就可以自动用浏览器打开Dashboard页面，而且还支持中文：</p><pre><code class="language-plain">minikube dashboard
</code></pre><p><img src="https://static001.geekbang.org/resource/image/60/e6/6077ff98a11705448f875ee00a1d8de6.png?wh=1920x957" alt="图片"></p><h2>小结</h2><p>好了，今天我们一起来研究了Kubernetes的内部架构和工作机制，可以看到它的功能非常完善，实现了大部分常见的运维管理工作，而且是全自动化的，能够节约大量的人力成本。</p><p>由于Kubernetes的抽象程度比较高，有很多陌生的新术语，不太好理解，所以我画了一张思维导图，你可以对照着再加深理解。</p><p><img src="https://static001.geekbang.org/resource/image/65/e1/65d38ac50b4f2f1fd4b6700d5b8e7be1.jpg?wh=1920x1096" alt="图片"></p><p>最后小结一下今天的要点：</p><ol>
<li>Kubernetes能够在集群级别管理应用和服务器，可以认为是一种集群操作系统。它使用“控制面/数据面”的基本架构，Master节点实现管理控制功能，Worker节点运行具体业务。</li>
<li>Kubernetes由很多模块组成，可分为核心的组件和选配的插件两类。</li>
<li>Master里有4个组件，分别是apiserver、etcd、scheduler、controller-manager。</li>
<li>Node里有3个组件，分别是kubelet、kube-proxy、container-runtime。</li>
<li>通常必备的插件有DNS和Dashboard。</li>
</ol><h2>课下作业</h2><p>最后是课下作业时间，给你留两个思考题：</p><ol>
<li>你觉得Kubernetes算得上是一种操作系统吗？和真正的操作系统相比有什么差异？</li>
<li>说说你理解的Kubernetes组件的作用，你觉得哪几个最重要？</li>
</ol><p>欢迎积极留言或者提问，和其他同学一起参与讨论，我们下节课见。</p><p><img src="https://static001.geekbang.org/resource/image/63/e5/6390bdf6a447f77726866d95df1eafe5.jpg?wh=1920x2580" alt=""></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/27/52/40/db9b0eb2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>自由</span>
  </div>
  <div class="_2_QraFYR_0">1. Kubernetes 算得上是一种操作系统吗？<br>算。什么是 OS，例如在物理服务器上，它用于对物理服务器硬件资源的抽象，并对进程进行调度等等。那么 Kubernetes 就是对云上资源的抽象，并对云原生微服务应用进行调度。Kubernetes 可以对公有云、私有云进行统一抽象，并实现对负载的无缝迁移和均衡，不会永远被绑定在某一个特定的云上。<br>OS，能屏蔽底层的复杂性，例如 Linux ，我们不用关心程序运行在哪个 CPU 核心上，OS 已经搞定了。Kubernetes 对云和应用程序进行了类似的管理，无须明确对应用程序在哪个节点或存储卷上进行硬编程。<br><br>2.我理解的 Kubernetes 组件的作用<br>2.1 主节点上的组件<br>2.1.1 API Server<br>负责集群中所有组件通信。访问它必须经过授权于认证。<br><br>2.1.2 集群存储<br>在控制平面中，只有集群存储是有状态的（会持久化的意思），存储集群的配置与状态。Kubernetes 底层用 etcd。etcd 认为一致性比可用性更加重要。对于所有分布式数据库，写操作性的一致性至关重要。etcd 使用 Raft 一致性算法解决这个问题。<br><br>2.1.3 Controller 管理器<br>Controller 管理器实现了控制循环，完成集群监控与事件响应。它负责创建 controller。一般控制循环包括：工作节点 controller、终端 controller 以及副本 controller。集群监控目的是保证集群当前状态与期望状态相匹配。集群监控基础逻辑大致如下：<br>* 获取期望状态。<br>* 观察当前状态。<br>* 判断差异。<br>* 变更消除差异点。<br><br>2.1.4 调度器<br>调度器职责是监听 API Server 来启动工作任务，并分配合适的节点。它的核心是排序系统，该系统有评分机制，将工作分配到分数最高的节点来运行任务。调度器确定可以执行任务的节点后，还会再进行前置校验，例如该节点是否仍然存在、分配的任务需要的端口当前选择的工作节点是否可以访问等，如果无法通过，该节点会被直接忽略，如果调度器最后无法找到合适的工作节点，则当前任务无法被调度，并被标记为暂停状态。 需要特别注意，调度器不负责运行任务，只为任务负责分配合适的工作节点。<br><br>2.2 工作节点上的组件<br><br>2.2.1 Kubelet<br>工作节点的核心部分。新工作节点加入节点后，Kubelet 会被部署到新节点，然后 Kubelet 将当前节点注册到集群中。它还有一个职责，监听 API Server 分配的任务，监听到就执行该任务，并维护一个与控制平面的通信频道。<br><br>2.2.2 容器运行时<br>工作节点需要通过它来获取、启动、停止执行任务依赖的容器，它负责容器管理与运行逻辑。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: amazing!</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-15 09:06:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/9d/a4/e481ae48.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lesserror</span>
  </div>
  <div class="_2_QraFYR_0">老师，有几个小问题：<br><br>1. etcd能取代redis的大部分功能吗？<br><br>2.  在 Kubernetes 里则只有一类人：DevOps。是不是意味着以后对开发或者运维人员都有更大的挑战，毕竟Kubernetes的也很很庞杂的知识要去学习。<br><br>3. 状态信息中的“AGE”代表启动时长吗？<br><br>4. DNS插件在执行命令：minikube addons list 输出的列表中没看到。是不是说这个插件已经集成到k8s中了，不需要单独安装。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.不能，etcd是分布式的，性能自然没有Redis好，用途不同。<br><br>2.是的，不过也可以找准方向，没有必要什么都学，和Linux开发一样，先有个基础。<br><br>3.是的，和http里的age差不多<br><br>4.对。dns已经是必备插件了，不用单独安装。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-25 11:49:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2c/82/ec/99b480e8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>hiDaLao</span>
  </div>
  <div class="_2_QraFYR_0">执行minikube dashboard无法在本机自动用浏览器打开dashboard页面，网上搜到的解决办法：执行<br>kubectl proxy --port=8888 --address=&#39;虚拟机ip&#39; --accept-hosts=&#39;^.*&#39; &amp;<br>然后在本机打开http:&#47;&#47;虚拟机ip:8888&#47;api&#47;v1&#47;namespaces&#47;kubernetes-dashboard&#47;services&#47;http:kubernetes-dashboard:&#47;proxy&#47;#&#47;workloads?namespace=default</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个办法是可行的，但按理来说minikube会直接启动浏览器的，不需要这么麻烦，可以再找找原因。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-21 00:35:38</div>
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
  <div class="_2_QraFYR_0">请教老师几个问题：<br>Q1：Mater的四个组件运行在一台主机上吗？<br>一个mater包括四个组件，这四个组件是运行在同一台主机上吗？即“一个Mater对应一个主机”。或者是另外一种理解：mater是个逻辑概念，可以对应一台主机，也可以对应多台主机，比如两个组件运行在一台主机，另外两个组件运行在另外一台主机，两台主机合起来才是mater。哪一种理解对？<br>Q2：Pod与docker容器是什么关系？<br>一个pod就对应一个docker容器吗？ 比如创建一个nginx容器，那么此容器就对应一个pod。<br>Q3：Pod是集中存储在master上，由master分发到各个node运行吗？<br>Q4：启动k8s后台管理系统后，”node list”查询出的IP是固定的吗？<br>09课文章中，用“minikube node list”查询出来的结果中，IP是“192.168.49.2”。在我的笔记本上，查出来的也是“192.168.49.2”。 请问，这个IP是固定的吗？<br>Q5：09课中，启动nginx后怎么验证启动成功？<br>09课的最后，用“kubectl run ngx --image=nginx:alpine”这个命令启动了nginx，此命令中没有指定端口映射。 请问，怎么验证启动成功？  Localhost:80 吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.如果有多台master节点，那就构成了一个master集群，这些组件就可以在多个节点里随意运行，它们都是独立的服务，用api接口通信，不必限制在同一台主机上。<br><br>2.后面会再专门讲Pod概念。<br><br>3.master负责调度管理，Pod运行在Node上，不是存储的意思，master下发命令给kubelet，kubelet在node上启动Pod。<br><br>4.这个跟网络有关，只能说是凑巧一样。<br><br>5.用kubectl get pod，后面会再讲pod的详细操作。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-13 09:34:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/99/87/98ebb20e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>dao</span>
  </div>
  <div class="_2_QraFYR_0">课后思考：<br>1. kubernetes 算得上是一种操作系统，正如它定位的就是 cloud native 一样。它与传统意义上的操作系统不同的是管理、调度的都是虚拟化&#47;池化的资源，与硬件之间隔了一层宿主操作系统（传统 OS）。<br>2. 哈哈，我觉得这里面的组件没有一个是多余的——都重要。没有 etcd 就没有持久化信息&#47;配置；没有 apiserver 整个集群就失联、失控；没有 kubelet 该节点就失联了；没有节点里的 container runtime 就没有 Pod，也就是没有任何应用服务；没有 kube-proxy，节点的服务就是孤岛；没有 controller manager 就无法管理节点及 Pod ，也就无法对外提供应用服务；没有 scheduler 集群就处于失衡状态。这里 apiserver 、 kubelet 、 kube-proxy 充当不同的桥接作用。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: awesome</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-16 10:20:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/27/52/40/db9b0eb2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>自由</span>
  </div>
  <div class="_2_QraFYR_0">2.2.3 Kube-proxy<br>负责本地集群网络，保障 Pod 间的网络路由与负载均衡</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-15 09:07:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/44/d8/708a0932.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>李一</span>
  </div>
  <div class="_2_QraFYR_0">问一个小白问题，看架构图 ETCD是在Master节点上的，由于ETCD记录了节点信息、ConfigMap、Secret数据，如果Master 节点宕机或数据损坏，K8S 如何保证数据完整性？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 生产环境会有多个master节点，etcd会部署成高可用。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-22 10:33:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1c/55/51/c7bffc64.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Andrew</span>
  </div>
  <div class="_2_QraFYR_0">老师，在真实环境中需要登录到master node才能对集群进行操作吗？<br>一般都会有一个kubeconfig文件，里面记录了server, user等信息，是通过本机直接和master node的api server进行交互吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 任意一台机器，安装了kubectl，然后再有一个config，就可以访问集群。<br><br>看第9讲的架构图，就是集群外面的console。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-15 01:38:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/51/8b/e3b827b7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>羿达</span>
  </div>
  <div class="_2_QraFYR_0">老师，是不是minikube的作用就是提供了以上的基础组件？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 它是基于docker，用镜像启动了Kubernetes。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-13 23:55:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/cd/e0/c85bb948.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>朱雯</span>
  </div>
  <div class="_2_QraFYR_0">1. 什么是操作系统？根据我所学到知识，操作系统是帮助人管理硬件的，具体来说就是管理内存，cpu，硬盘这些内容。文件是硬盘的抽象，内存是内存条的抽象，进程是运行程序的抽象。总的来说，操作系统就是硬件再软件层面如何被抽象和管理的工具。从这个角度来看，k8s不算操作系统，因为k8s并不直接控制硬件，而是控制更高级别的抽象软件。但从结果上来看，k8s可以管理，并且主要是为这个产生的，所以又算是操作系统。与传统的操作系统区别在于，传统操作系统管理可以更多是单机管理，无法跨机器跨集群管理，但k8s可以，因为k8s天然在云上。<br>2. 对于一个操作系统来说，管理的重点在于，网络，计算和存储，网络方面会使用到kube-proxy和apiserver，计算方面有contonller-manager，scheduler，apiserver和kubelet，存储方面主要是etcd，我觉得都挺重要，缺少一环都无法成就操作系统的管理了，但硬要说最重要，可能还是etcd，毕竟其他的一关机就没了，但存储还在。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-13 22:45:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/2xoGmvlQ9qfSibVpPJyyaEriavuWzXnuECrJITmGGHnGVuTibUuBho43Uib3Y5qgORHeSTxnOOSicxs0FV3HGvTpF0A/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>psoracle</span>
  </div>
  <div class="_2_QraFYR_0">回答下思考题：<br>1. 你觉得 Kubernetes 算得上是一种操作系统吗？<br>都说Kubernetes是云原生时代的操作系统，研究不深，说下自己的感受。<br>Kubernetes的核心是容器编排，围绕着调度，发明了Pod，再通过声明式api、控制器这些概念将Pod高效地调度与运行起来。所以说它管理软件（容器，如业务容器，本质是Node OS上的一个进程），确实是的，它控制了Pod的启停、副本等；但说它管理硬件，倒没看出来，像CPU、内存、存储也只参与了调度算法而已。<br><br>2. 和真正的操作系统相比有什么差异？说说你理解的 Kubernetes 组件的作用，你觉得哪几个最重要？<br>a. kube-apiserver，是k8s系统入口，类似于操作系统上的系统调用<br>b. kube-scheduler，调度器，容器编排最主要的组件，负责Pod调度，类似于操作系统上的控制器<br>c. kube-manage-controller，控制器，负责管理k8s中资源对象<br>d. kubelet，Pod生命周期管理，负责Pod运行环境创建，包括各种CRI、CNI等各种manager的管理<br>当然，etcd存储k8s所有对象配置，提供查询与订阅功能，并且性能优秀也是很重要的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-13 21:25:36</div>
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
  <div class="_2_QraFYR_0">一直没有清楚  module 和addon 区别，今天总于明白了😂</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-13 19:40:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/WrANpwBMr6DsGAE207QVs0YgfthMXy3MuEKJxR8icYibpGDCI1YX4DcpDq1EsTvlP8ffK1ibJDvmkX9LUU4yE8X0w/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>星垂平野阔</span>
  </div>
  <div class="_2_QraFYR_0">1.k8s感觉上更像是一个介于操作系统和应用之间的服务，相比传统的操作系统，它提供了更强大的资源抽象功能。<br>2.scheduler和kubelet比较重要。前者提供了pod调度功能，没有它服务就无法部署运行；kubelet是更基础的组件，没有它容器都无法运行，也无法上报节点状态。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-13 08:12:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/7e/25/3932dafd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>GeekNEO</span>
  </div>
  <div class="_2_QraFYR_0">服务器是Ubuntu 22.04.2 LTS Server版本 最小化安装，没有Desktop 无法打开浏览器 故需要开启外部其他机器访问。<br>kubectl proxy --port=8001 --address=&#39;172.17.40.174&#39; --accept-hosts=&#39;^.*&#39; &amp;<br><br>--port 是指虚拟机端口<br>--address 是指虚拟机IP<br>http:&#47;&#47;172.17.40.174:8001&#47;api&#47;v1&#47;namespaces&#47;kubernetes-dashboard&#47;services&#47;http:kubernetes-dashboard:&#47;proxy&#47;</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 出于学习的方便，还是安装桌面版比较省事，即可以ssh登录，也可以直接开浏览器。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-17 19:54:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/cd/db/7467ad23.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Bachue Zhou</span>
  </div>
  <div class="_2_QraFYR_0">请教下，容器被 scheduler 调度，但是实际对容器下发操作命令的是 kubelet，二者的通讯却是通过 apiserver 间接进行的，所以 apiserver 与 kubelet 之间是使用类似于 websocket 的流式通讯协议来通讯的吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以参考第15讲里的Kubernetes结构图，各个组件之间的通信使用的是Restful API。<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-15 09:00:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1c/f0/e9/1ff0a3d5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>...</span>
  </div>
  <div class="_2_QraFYR_0">master节点需要安装kubectl插件吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果要在master节点上用kubectl操作Kubernetes就得安装，一般建议安装，操作起来方便，不装也可以。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-08 08:49:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/b5/46/2ac4b984.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>三溪</span>
  </div>
  <div class="_2_QraFYR_0">罗老师画思维导图是真的强</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: thanks</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-26 09:05:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/ibZVAmmdAibBeVpUjzwId8ibgRzNk7fkuR5pgVicB5mFSjjmt2eNadlykVLKCyGA0GxGffbhqLsHnhDRgyzxcKUhjg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>pyhhou</span>
  </div>
  <div class="_2_QraFYR_0">有几个问题想请教老师：<br><br>1. Pod 和 Node 的关系是什么？可以理解为 Pod 在 Node 上运行和挂载吗？<br><br>2. 当集群中有多个 Master 存在时，Master 和 Worker 是如何进行交流的？如果一个 Worker 可以和多个 Master 进行通信，那么 Master 之间是如何保证数据一致性的？如果 Worker 只能和某一个对应的 master 进行通信，那么 K8S 是如何处理单点故障的？<br><br>3. kube-proxy 的作用和服务对象是谁？是同一个 node 上不同的 pod 吗？我在想 container-runtime 已经包含了对外的网络模块，这里的 kube-proxy 的作用对象应该是局限于集群内部？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1. Pod是应用程序，node是计算节点，所以Pod就被调度到node上运行。<br><br>2.etcd是分布式数据库，一致性由它保证，一般会有多个master节点，和任意一个上面的apiserver通信，都会存到etcd里。<br><br>3. kube-proxy是给Service对象用的，是四层的负载均衡。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-19 11:24:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/57/4f/6fb51ff1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>一步</span>
  </div>
  <div class="_2_QraFYR_0">mac 上执行 kubectl get node, 为什么没有显示 master 节点的？<br>NAME       STATUS   ROLES           AGE   VERSION<br>minikube   Ready    control-plane   2d    v1.24.1</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 改名了，不叫master了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-13 23:07:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/63/1b/83ac7733.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>忧天小鸡</span>
  </div>
  <div class="_2_QraFYR_0">这个结构，会不会产生信息欺骗？<br>apiserver负责太多内容，会不会满负荷，监控面板从etcd读数据，<br>如果kubelet如果一直上报错误的信息，是不是会产生监听错误的情况？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: master可以横向扩容，kubelet上报错误信息也有可能，但这个恐怕无法避免。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-13 22:57:57</div>
  </div>
</div>
</div>
</li>
</ul>