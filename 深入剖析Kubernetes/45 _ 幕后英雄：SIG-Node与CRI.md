<audio title="45 _ 幕后英雄：SIG-Node与CRI" src="https://static001.geekbang.org/resource/audio/dd/3a/dd28bbef572731c1cc9edaaae166d13a.mp3" controls="controls"></audio> 
<p>你好，我是张磊。今天我和你分享的主题是：幕后英雄之SIG-Node与CRI。</p><p>在前面的文章中，我为你详细讲解了关于 Kubernetes 调度和资源管理相关的内容。实际上，在调度这一步完成后，Kubernetes 就需要负责将这个调度成功的 Pod，在宿主机上创建出来，并把它所定义的各个容器启动起来。这些，都是 kubelet 这个核心组件的主要功能。</p><p>在接下来三篇文章中，我就深入到 kubelet 里面，为你详细剖析一下 Kubernetes 对容器运行时的管理能力。</p><p>在 Kubernetes 社区里，与 kubelet 以及容器运行时管理相关的内容，都属于 SIG-Node 的范畴。如果你经常参与社区的话，你可能会觉得，相比于其他每天都热闹非凡的 SIG小组，SIG-Node 是 Kubernetes 里相对沉寂也不太发声的一个小组，小组里的成员也很少在外面公开宣讲。</p><p>不过，正如我前面所介绍的，SIG-Node以及 kubelet，其实是 Kubernetes整套体系里非常核心的一个部分。 毕竟，它们才是 Kubernetes 这样一个容器编排与管理系统，跟容器打交道的主要“场所”。</p><p>而 kubelet 这个组件本身，也是 Kubernetes 里面第二个不可被替代的组件（第一个不可被替代的组件当然是 kube-apiserver）。也就是说，<strong>无论如何，我都不太建议你对 kubelet 的代码进行大量的改动。保持 kubelet 跟上游基本一致的重要性，就跟保持 kube-apiserver 跟上游一致是一个道理。</strong></p><!-- [[[read_end]]] --><p>当然， kubelet 本身，也是按照“控制器”模式来工作的。它实际的工作原理，可以用如下所示的一幅示意图来表示清楚。</p><p><img src="https://static001.geekbang.org/resource/image/91/03/914e097aed10b9ff39b509759f8b1d03.png?wh=1333*954" alt=""><br>
可以看到，kubelet 的工作核心，就是一个控制循环，即：SyncLoop（图中的大圆圈）。而驱动这个控制循环运行的事件，包括四种：</p><ol>
<li>
<p>Pod 更新事件；</p>
</li>
<li>
<p>Pod 生命周期变化；</p>
</li>
<li>
<p>kubelet 本身设置的执行周期；</p>
</li>
<li>
<p>定时的清理事件。</p>
</li>
</ol><p>所以，跟其他控制器类似，kubelet 启动的时候，要做的第一件事情，就是设置 Listers，也就是注册它所关心的各种事件的 Informer。这些 Informer，就是 SyncLoop 需要处理的数据的来源。</p><p>此外，kubelet 还负责维护着很多很多其他的子控制循环（也就是图中的小圆圈）。这些控制循环的名字，一般被称作某某 Manager，比如 Volume Manager、Image Manager、Node Status Manager等等。</p><p>不难想到，这些控制循环的责任，就是通过控制器模式，完成 kubelet 的某项具体职责。比如 Node Status Manager，就负责响应 Node 的状态变化，然后将 Node 的状态收集起来，并通过 Heartbeat 的方式上报给 APIServer。再比如 CPU Manager，就负责维护该 Node 的 CPU 核的信息，以便在 Pod 通过 cpuset 的方式请求 CPU 核的时候，能够正确地管理 CPU 核的使用量和可用量。</p><p>那么这个 <strong>SyncLoop，又是如何根据 Pod 对象的变化，来进行容器操作的呢？</strong></p><p>实际上，kubelet 也是通过 Watch机制，监听了与自己相关的 Pod 对象的变化。当然，这个 Watch 的过滤条件是该 Pod 的 nodeName 字段与自己相同。kubelet 会把这些 Pod 的信息缓存在自己的内存里。</p><p>而当一个 Pod 完成调度、与一个 Node 绑定起来之后， 这个 Pod 的变化就会触发 kubelet 在控制循环里注册的 Handler，也就是上图中的 HandlePods 部分。此时，通过检查该 Pod 在 kubelet 内存里的状态，kubelet 就能够判断出这是一个新调度过来的 Pod，从而触发 Handler 里 ADD 事件对应的处理逻辑。</p><p>在具体的处理过程当中，kubelet 会启动一个名叫 Pod Update Worker的、单独的 Goroutine 来完成对 Pod 的处理工作。</p><p>比如，如果是 ADD 事件的话，kubelet 就会为这个新的 Pod 生成对应的 Pod Status，检查 Pod 所声明使用的 Volume 是不是已经准备好。然后，调用下层的容器运行时（比如 Docker），开始创建这个 Pod 所定义的容器。</p><p>而如果是 UPDATE 事件的话，kubelet 就会根据 Pod 对象具体的变更情况，调用下层容器运行时进行容器的重建工作。</p><p>在这里需要注意的是，<strong>kubelet 调用下层容器运行时的执行过程，并不会直接调用 Docker 的 API，而是通过一组叫作 CRI（Container Runtime Interface，容器运行时接口）的 gRPC 接口来间接执行的。</strong></p><p>Kubernetes 项目之所以要在 kubelet 中引入这样一层单独的抽象，当然是为了对 Kubernetes 屏蔽下层容器运行时的差异。实际上，对于 1.6版本之前的 Kubernetes 来说，它就是直接调用Docker 的 API 来创建和管理容器的。</p><p>但是，正如我在本专栏开始介绍容器背景的时候提到过的，Docker 项目风靡全球后不久，CoreOS 公司就推出了 rkt 项目来与 Docker 正面竞争。在这种背景下，Kubernetes 项目的默认容器运行时，自然也就成了两家公司角逐的重要战场。</p><p>毋庸置疑，Docker 项目必然是 Kubernetes 项目最依赖的容器运行时。但凭借与 Google 公司非同一般的关系，CoreOS 公司还是在2016年成功地将对 rkt 容器的支持，直接添加进了 kubelet 的主干代码里。</p><p>不过，这个“赶鸭子上架”的举动，并没有为 rkt 项目带来更多的用户，反而给 kubelet 的维护人员，带来了巨大的负担。</p><p>不难想象，在这种情况下， <strong>kubelet 任何一次重要功能的更新，都不得不考虑Docker 和 rkt 这两种容器运行时的处理场景，然后分别更新 Docker 和 rkt 两部分代码。</strong></p><p>更让人为难的是，由于 rkt 项目实在太小众，kubelet 团队所有与 rkt 相关的代码修改，都必须依赖于 CoreOS 的员工才能做到。这不仅拖慢了 kubelet 的开发周期，也给项目的稳定性带来了巨大的隐患。</p><p>与此同时，在2016年，Kata Containers 项目的前身runV项目也开始逐渐成熟，这种基于虚拟化技术的强隔离容器，与 Kubernetes 和 Linux 容器项目之间具有良好的互补关系。所以，<strong>在 Kubernetes 上游，对虚拟化容器的支持很快就被提上了日程。</strong></p><p>不过，虽然虚拟化容器运行时有各种优点，但它与 Linux 容器截然不同的实现方式，使得它跟 Kubernetes 的集成工作，比 rkt 要复杂得多。如果此时，再把对runV支持的代码也一起添加到 kubelet 当中，那么接下来kubelet 的维护工作就可以说完全没办法正常进行了。</p><p>所以，在2016年，SIG-Node 决定开始动手解决上述问题。而解决办法也很容易想到，那就是把 kubelet 对容器的操作，统一地抽象成一个接口。这样，kubelet 就只需要跟这个接口打交道了。而作为具体的容器项目，比如 Docker、 rkt、runV，它们就只需要自己提供一个该接口的实现，然后对 kubelet 暴露出 gRPC 服务即可。</p><p>这一层统一的容器操作接口，就是 CRI了。我会在下一篇文章中，为你详细讲解 CRI 的设计与具体的实现原理。</p><p>而在有了 CRI 之后，Kubernetes 以及 kubelet 本身的架构，就可以用如下所示的一幅示意图来描述。</p><p><img src="https://static001.geekbang.org/resource/image/51/fe/5161bd6201942f7a1ed6d70d7d55acfe.png?wh=1940*1005" alt=""><br>
可以看到，当 Kubernetes 通过编排能力创建了一个 Pod 之后，调度器会为这个 Pod 选择一个具体的节点来运行。这时候，kubelet 当然就会通过前面讲解过的 SyncLoop 来判断需要执行的具体操作，比如创建一个Pod。那么此时，kubelet 实际上就会调用一个叫作 GenericRuntime 的通用组件来发起创建 Pod 的 CRI 请求。</p><p>那么，<strong>这个 CRI 请求，又该由谁来响应呢？</strong></p><p>如果你使用的容器项目是 Docker 的话，那么负责响应这个请求的就是一个叫作 dockershim 的组件。它会把 CRI 请求里的内容拿出来，然后组装成 Docker API 请求发给 Docker Daemon。</p><p>需要注意的是，在 Kubernetes 目前的实现里，dockershim 依然是 kubelet 代码的一部分。当然，在将来，dockershim 肯定会被从 kubelet 里移出来，甚至直接被废弃掉。</p><p>而更普遍的场景，就是你需要在每台宿主机上单独安装一个负责响应 CRI 的组件，这个组件，一般被称作CRI shim。顾名思义，CRI shim 的工作，就是扮演 kubelet 与容器项目之间的“垫片”（shim）。所以它的作用非常单一，那就是实现 CRI 规定的每个接口，然后把具体的 CRI 请求“翻译”成对后端容器项目的请求或者操作。</p><h2>总结</h2><p>在本篇文章中，我首先为你介绍了 SIG-Node 的职责，以及 kubelet 这个组件的工作原理。</p><p>接下来，我为你重点讲解了 kubelet 究竟是如何将 Kubernetes 对应用的定义，一步步转换成最终对 Docker 或者其他容器项目的API 请求的。</p><p>不难看到，在这个过程中，kubelet 的 SyncLoop 和 CRI 的设计，是其中最重要的两个关键点。也正是基于以上设计，SyncLoop 本身就要求这个控制循环是绝对不可以被阻塞的。所以，凡是在 kubelet 里有可能会耗费大量时间的操作，比如准备 Pod 的 Volume、拉取镜像等，SyncLoop 都会开启单独的 Goroutine 来进行操作。</p><h2>思考题</h2><p>请问，在你的项目中，你是如何部署 kubelet 这个组件的？为什么要这么做呢？</p><p>感谢你的收听，欢迎你给我留言，也欢迎分享给更多的朋友一起阅读。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/f8/ba/14e05601.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>约书亚</span>
  </div>
  <div class="_2_QraFYR_0">整整2年前的内容了，这两天最大的新闻就是k8s将弃用docker，移除dockershim，所以特意跑来留言</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-05 17:51:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/ibYCL2nbctJDpBLwZXlKIX7Z4XUicBm1ibIbLd0dWMYibxtshnEzOWAl5LC7JgMcjSet5B30s2HUpiabhYyyFWiaWd1Q/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>hjydxy</span>
  </div>
  <div class="_2_QraFYR_0">请教老师一个问题，现在有一个linux的k8s集群，一个windows的k8s集群，可不可以把这两个集群统一组建成一个新的联邦集群，统一调度？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-06 10:33:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/e1/91/195f6f4c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Slide</span>
  </div>
  <div class="_2_QraFYR_0">这专栏从19年年底开始看，回头来回看完了好几遍，每一次都有新的理解。。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-04 15:38:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/ac/c8/4b1c0d40.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>勤劳的小胖子-libo</span>
  </div>
  <div class="_2_QraFYR_0">&quot;当 Kubernetes 通过编排能力创建了一个 Pod 之后，调 度器会为这个Pod选择一个具体的节点来运行。这时候，kubelet会。。。。创建一个Pod&quot;<br>请教，这个kubelet以及随后的CRI grpc-&gt;dockershim&#47;CRIshim 是在具体的node上面运行的吗？还是通过master来远程调用的？<br>另外，为什么与dockershim并列的叫remote(no-op)，难道都是不在同一个node的远程创建？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 在node上。两套代码。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-05 10:56:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/57/c4/8a799b4e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>zhenLEE@Saddam</span>
  </div>
  <div class="_2_QraFYR_0">dockershim由docker开源维护了，现在看来，有必要看看contained了 </div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-16 13:57:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/73/90/9118f46d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>chenhz</span>
  </div>
  <div class="_2_QraFYR_0">使用 ansible-playbook 部署到 Node 。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-06 09:57:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/55/f3/4e8fecaa.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>普罗@庞铮</span>
  </div>
  <div class="_2_QraFYR_0">原生部署或者容器部署，不过这里的容器不能和k8s使用的容器重叠。😁</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-22 15:57:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_9ca34e</span>
  </div>
  <div class="_2_QraFYR_0">老师，我用kubeadm 搭建了一个k8s集群，又搭建了一个Ceph rbd，两个集群不在相同的机器上，我现在手动创建PV，PVC，CS，并绑定pod没有问题，但是我想实现PVC动态绑定CS却不行，我看网上说需要安装rbd-privi插件，但是插件配好后，还是不能动态创建pv，to；通过log日志查看，好像是访问不到coredns，请问老师有好的解决方法麽？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-06 21:01:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/81/1e/d266733a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>wutz</span>
  </div>
  <div class="_2_QraFYR_0">kubernetes 1.13 最近刚发布，kubeadm 也 GA 了，为何其 HA 支持被移除了？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这就是我之前说的，做不完，不好做啊</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-05 17:29:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLyfjHfjulibFGPTewSZZHm2M8yfI7BZmO9vLUFoagveCw3DWYDss7y1CecKia7lT5yb9KoAmsya2zg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Goteswille</span>
  </div>
  <div class="_2_QraFYR_0">打卡</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-05 11:36:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/3f/0d/1e8dbb2c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>怀揣梦想的学渣</span>
  </div>
  <div class="_2_QraFYR_0">正如作者讲的，Dockershim确实要删除。今天在官方文档看到“Dockershim removed from Kubernetes<br>As of Kubernetes 1.24, Dockershim is no longer included in Kubernetes. ”</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-06 09:36:37</div>
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
  <div class="_2_QraFYR_0">跟不上了，我要快点看！！！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-06 17:46:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/uqaRIfRCAhJ6t1z92XYEzTHlzk8IVNVXIS13zP2m2qlPZSAauSz81Ru1H0jUnnspAwfVjvlUtMXd9XA7LnKic3PDcIFH8xEQp/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_842c94</span>
  </div>
  <div class="_2_QraFYR_0">实际上，kubelet 也是通过 Watch 机制，监听了与自己相关的 Pod 对象的变化。当然，这个 Watch 的过滤条件是该 Pod 的 nodeName 字段与自己相同。kubelet 会把这些 Pod 的信息缓存在自己的内存里。<br><br>kubelet不是运行在Node上的吗，他监听的是所有的Pod？还是说只监听本节点的Pod信息</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-24 12:53:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/22/69/09f7a8a2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Don Wang</span>
  </div>
  <div class="_2_QraFYR_0">第三遍，第四遍都是哪里有需求，看哪里了...</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-02 10:53:51</div>
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
  <div class="_2_QraFYR_0">第四十三课:幕后英雄:SIG-Node和CRI<br>kube-apiserver和kubelet是k8s中最核心的两个组建。其中kubelet是属于SIG-Node的范畴。<br><br>kubelet的工作核心是一个叫做SyncLoop的控制循环。驱动这个控制循环运行的事件包括四种:<br>1. Pod更新事件<br>2. Pod生命周期变化<br>3. kubelet本身设置的执行周期<br>4. 定时的清理事件<br><br>kubelet启动时要做的第一件事就是设置Listers，注册它所关心的的Informer.这些Informer就是SyncLoop需要处理的数据的来源。kubelet还负责很多子控制循环，比如Volume Manager, Image Manager, Node status Manager.kubelet还会通过Watch机制来监听关心的Pod对象的变化。<br><br>CRI是负责kubeblet和底层容器打交道的统一接口<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-04 08:48:34</div>
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
  <div class="_2_QraFYR_0">本文收获：所有的异步任务，都需要队列这种逻辑结构。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-13 16:54:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/8f/5b/2f16ca95.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>悟空</span>
  </div>
  <div class="_2_QraFYR_0">各位朋友看到这里 相信大家都对k8s有所收获，这也是我第二遍学习，每一遍都有不同的收获，我建了一个k8s的学习群，日常分享学习心得和答疑解惑 帮助大家一起成长，欢迎大家加入进来一起进步，个人微信 coder_wukong 欢迎加好友</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-03 05:49:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/d4/f3/129d6dfe.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>李二木</span>
  </div>
  <div class="_2_QraFYR_0">SyncLoop 感觉跟netty的eventloop 有点像。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-13 22:18:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/0d/c2/37662333.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>啊琼คิดถึง</span>
  </div>
  <div class="_2_QraFYR_0">打卡，老师讲的真好！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-13 17:13:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/20/fe/54/51976e52.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ଘ琳̌琳̌ଓ</span>
  </div>
  <div class="_2_QraFYR_0">张老师，我们这边遇到一个问题。就是在实际应用时，频繁在服务器上启动和销毁POD，会出现POD创建和销毁的速度变慢，甚至出现POD一直为ContainerCreating状态。通过查看是由于systemctl list-units --all | wc -l过多导致的问题，其中，一个月积累的dhcp-interface@veth00665a8.service                                                                             loaded    failed   failed  DHCP interface veth00665a8  个数超过10万个，请问有没有什么解决方法。谢谢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-23 17:42:19</div>
  </div>
</div>
</div>
</li>
</ul>