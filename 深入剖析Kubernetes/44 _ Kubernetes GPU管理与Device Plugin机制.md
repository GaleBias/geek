<audio title="44 _ Kubernetes GPU管理与Device Plugin机制" src="https://static001.geekbang.org/resource/audio/a8/d6/a8db9e648b0fe173c7e5e6f8a05dc3d6.mp3" controls="controls"></audio> 
<p>你好，我是张磊。今天我和你分享的主题是：Kubernetes GPU管理与Device Plugin机制。</p><p>2016年，随着 AlphaGo 的走红和TensorFlow 项目的异军突起，一场名为 AI 的技术革命迅速从学术界蔓延到了工业界，所谓的 AI 元年，就此拉开帷幕。</p><p>当然，机器学习或者说人工智能，并不是什么新鲜的概念。而这次热潮的背后，云计算服务的普及与成熟，以及算力的巨大提升，其实正是将人工智能从象牙塔带到工业界的一个重要推手。</p><p>而与之相对应的，从2016年开始，Kubernetes 社区就不断收到来自不同渠道的大量诉求，希望能够在 Kubernetes 集群上运行 TensorFlow 等机器学习框架所创建的训练（Training）和服务（Serving）任务。而这些诉求中，除了前面我为你讲解过的 Job、Operator 等离线作业管理需要用到的编排概念之外，还有一个亟待实现的功能，就是对 GPU 等硬件加速设备管理的支持。</p><p>不过， 正如同 TensorFlow 之于 Google 的战略意义一样，<strong>GPU 支持对于 Kubernetes 项目来说，其实也有着超过技术本身的考虑</strong>。所以，尽管在硬件加速器这个领域里，Kubernetes 上游有着不少来自 NVIDIA 和 Intel 等芯片厂商的工程师，但这个特性本身，却从一开始就是以 Google Cloud 的需求为主导来推进的。</p><!-- [[[read_end]]] --><p>而对于云的用户来说，在 GPU 的支持上，他们最基本的诉求其实非常简单：我只要在 Pod 的 YAML 里面，声明某容器需要的 GPU 个数，那么Kubernetes 为我创建的容器里就应该出现对应的 GPU 设备，以及它对应的驱动目录。</p><p>以 NVIDIA 的 GPU 设备为例，上面的需求就意味着当用户的容器被创建之后，这个容器里必须出现如下两部分设备和目录：</p><ol>
<li>
<p>GPU 设备，比如 /dev/nvidia0；</p>
</li>
<li>
<p>GPU 驱动目录，比如/usr/local/nvidia/*。</p>
</li>
</ol><p>其中，GPU 设备路径，正是该容器启动时的 Devices 参数；而驱动目录，则是该容器启动时的 Volume 参数。所以，在 Kubernetes 的GPU 支持的实现里，kubelet 实际上就是将上述两部分内容，设置在了创建该容器的 CRI （Container Runtime Interface）参数里面。这样，等到该容器启动之后，对应的容器里就会出现 GPU 设备和驱动的路径了。</p><p>不过，Kubernetes 在 Pod 的 API 对象里，并没有为 GPU 专门设置一个资源类型字段，而是使用了一种叫作 Extended Resource（ER）的特殊字段来负责传递 GPU 的信息。比如下面这个例子：</p><pre><code>apiVersion: v1
kind: Pod
metadata:
  name: cuda-vector-add
spec:
  restartPolicy: OnFailure
  containers:
    - name: cuda-vector-add
      image: &quot;k8s.gcr.io/cuda-vector-add:v0.1&quot;
      resources:
        limits:
          nvidia.com/gpu: 1
</code></pre><p>可以看到，在上述 Pod 的 limits 字段里，这个资源的名称是<code>nvidia.com/gpu</code>，它的值是1。也就是说，这个 Pod 声明了自己要使用一个 NVIDIA 类型的GPU。</p><p>而在 kube-scheduler 里面，它其实并不关心这个字段的具体含义，只会在计算的时候，一律将调度器里保存的该类型资源的可用量，直接减去 Pod 声明的数值即可。所以说，Extended Resource，其实是 Kubernetes 为用户设置的一种对自定义资源的支持。</p><p>当然，为了能够让调度器知道这个自定义类型的资源在每台宿主机上的可用量，宿主机节点本身，就必须能够向 API Server 汇报该类型资源的可用数量。在 Kubernetes 里，各种类型的资源可用量，其实是 Node 对象Status 字段的内容，比如下面这个例子：</p><pre><code>apiVersion: v1
kind: Node
metadata:
  name: node-1
...
Status:
  Capacity:
   cpu:  2
   memory:  2049008Ki
</code></pre><p>而为了能够在上述 Status 字段里添加自定义资源的数据，你就必须使用 PATCH API 来对该 Node 对象进行更新，加上你的自定义资源的数量。这个 PATCH 操作，可以简单地使用 curl 命令来发起，如下所示：</p><pre><code># 启动 Kubernetes 的客户端 proxy，这样你就可以直接使用 curl 来跟 Kubernetes  的API Server 进行交互了
$ kubectl proxy

# 执行 PACTH 操作
$ curl --header &quot;Content-Type: application/json-patch+json&quot; \
--request PATCH \
--data '[{&quot;op&quot;: &quot;add&quot;, &quot;path&quot;: &quot;/status/capacity/nvidia.com/gpu&quot;, &quot;value&quot;: &quot;1&quot;}]' \
http://localhost:8001/api/v1/nodes/&lt;your-node-name&gt;/status
</code></pre><p>PATCH 操作完成后，你就可以看到 Node 的 Status 变成了如下所示的内容：</p><pre><code>apiVersion: v1
kind: Node
...
Status:
  Capacity:
   cpu:  2
   memory:  2049008Ki
   nvidia.com/gpu: 1
</code></pre><p>这样在调度器里，它就能够在缓存里记录下node-1上的<code>nvidia.com/gpu</code>类型的资源的数量是1。</p><p>当然，在 Kubernetes 的 GPU 支持方案里，你并不需要真正去做上述关于 Extended Resource 的这些操作。在 Kubernetes 中，对所有硬件加速设备进行管理的功能，都是由一种叫作 Device Plugin的插件来负责的。这其中，当然也就包括了对该硬件的 Extended Resource 进行汇报的逻辑。</p><p>Kubernetes 的 Device Plugin 机制，我可以用如下所示的一幅示意图来和你解释清楚。</p><p><img src="https://static001.geekbang.org/resource/image/10/10/10a472b64f9daf24f63df4e3ae24cd10.jpg?wh=1640*808" alt=""></p><p>我们先从这幅示意图的右侧开始看起。</p><p>首先，对于每一种硬件设备，都需要有它所对应的 Device Plugin 进行管理，这些 Device Plugin，都通过gRPC 的方式，同 kubelet 连接起来。以 NVIDIA GPU 为例，它对应的插件叫作<a href="https://github.com/NVIDIA/k8s-device-plugin"><code>NVIDIA GPU device plugin</code></a>。</p><p>这个 Device Plugin 会通过一个叫作 ListAndWatch的 API，定期向 kubelet 汇报该 Node 上 GPU 的列表。比如，在我们的例子里，一共有三个GPU（GPU0、GPU1和 GPU2）。这样，kubelet 在拿到这个列表之后，就可以直接在它向 APIServer 发送的心跳里，以 Extended Resource 的方式，加上这些 GPU 的数量，比如<code>nvidia.com/gpu=3</code>。所以说，用户在这里是不需要关心 GPU 信息向上的汇报流程的。</p><p>需要注意的是，ListAndWatch向上汇报的信息，只有本机上 GPU 的 ID 列表，而不会有任何关于 GPU 设备本身的信息。而且 kubelet 在向 API Server 汇报的时候，只会汇报该 GPU 对应的Extended Resource 的数量。当然，kubelet 本身，会将这个 GPU 的 ID 列表保存在自己的内存里，并通过 ListAndWatch API 定时更新。</p><p>而当一个 Pod 想要使用一个 GPU 的时候，它只需要像我在本文一开始给出的例子一样，在 Pod 的 limits 字段声明<code>nvidia.com/gpu: 1</code>。那么接下来，Kubernetes 的调度器就会从它的缓存里，寻找 GPU 数量满足条件的 Node，然后将缓存里的 GPU 数量减1，完成Pod 与 Node 的绑定。</p><p>这个调度成功后的 Pod信息，自然就会被对应的 kubelet 拿来进行容器操作。而当 kubelet 发现这个 Pod 的容器请求一个 GPU 的时候，kubelet 就会从自己持有的 GPU列表里，为这个容器分配一个GPU。此时，kubelet 就会向本机的 Device Plugin 发起一个 Allocate() 请求。这个请求携带的参数，正是即将分配给该容器的设备 ID 列表。</p><p>当 Device Plugin 收到 Allocate 请求之后，它就会根据kubelet 传递过来的设备 ID，从Device Plugin 里找到这些设备对应的设备路径和驱动目录。当然，这些信息，正是 Device Plugin 周期性的从本机查询到的。比如，在 NVIDIA Device Plugin 的实现里，它会定期访问 nvidia-docker 插件，从而获取到本机的 GPU 信息。</p><p>而被分配GPU对应的设备路径和驱动目录信息被返回给 kubelet 之后，kubelet 就完成了为一个容器分配 GPU 的操作。接下来，kubelet 会把这些信息追加在创建该容器所对应的 CRI 请求当中。这样，当这个 CRI 请求发给 Docker 之后，Docker 为你创建出来的容器里，就会出现这个 GPU 设备，并把它所需要的驱动目录挂载进去。</p><p>至此，Kubernetes 为一个Pod 分配一个 GPU 的流程就完成了。</p><p>对于其他类型硬件来说，要想在 Kubernetes 所管理的容器里使用这些硬件的话，也需要遵循上述 Device Plugin 的流程来实现如下所示的Allocate和 ListAndWatch API：</p><pre><code>  service DevicePlugin {
        // ListAndWatch returns a stream of List of Devices
        // Whenever a Device state change or a Device disappears, ListAndWatch
        // returns the new list
        rpc ListAndWatch(Empty) returns (stream ListAndWatchResponse) {}
        // Allocate is called during container creation so that the Device
        // Plugin can run device specific operations and instruct Kubelet
        // of the steps to make the Device available in the container
        rpc Allocate(AllocateRequest) returns (AllocateResponse) {}
  }
</code></pre><p>目前，Kubernetes社区里已经实现了很多硬件插件，比如<a href="https://github.com/intel/intel-device-plugins-for-kubernetes">FPGA</a>、<a href="https://github.com/intel/sriov-network-device-plugin">SRIOV</a>、<a href="https://github.com/hustcat/k8s-rdma-device-plugin">RDMA</a>等等。感兴趣的话，你可以点击这些链接来查看这些 Device Plugin 的实现。</p><h2>总结</h2><p>在本篇文章中，我为你详细讲述了 Kubernetes 对 GPU 的管理方式，以及它所需要使用的 Device Plugin 机制。</p><p>需要指出的是，Device Plugin 的设计，长期以来都是以 Google Cloud 的用户需求为主导的，所以，它的整套工作机制和流程上，实际上跟学术界和工业界的真实场景还有着不小的差异。</p><p>这里最大的问题在于，GPU 等硬件设备的调度工作，实际上是由 kubelet 完成的。即，kubelet 会负责从它所持有的硬件设备列表中，为容器挑选一个硬件设备，然后调用 Device Plugin 的 Allocate API 来完成这个分配操作。可以看到，在整条链路中，调度器扮演的角色，仅仅是为 Pod 寻找到可用的、支持这种硬件设备的节点而已。</p><p>这就使得，Kubernetes 里对硬件设备的管理，只能处理“设备个数”这唯一一种情况。一旦你的设备是异构的、不能简单地用“数目”去描述具体使用需求的时候，比如，“我的 Pod 想要运行在计算能力最强的那个 GPU 上”，Device Plugin 就完全不能处理了。</p><p>更不用说，在很多场景下，我们其实希望在调度器进行调度的时候，就可以根据整个集群里的某种硬件设备的全局分布，做出一个最佳的调度选择。</p><p>此外，上述 Device Plugin 的设计，也使得 Kubernetes 里，缺乏一种能够对 Device 进行描述的 API 对象。这就使得如果你的硬件设备本身的属性比较复杂，并且 Pod 也关心这些硬件的属性的话，那么 Device Plugin 也是完全没有办法支持的。</p><p>更为棘手的是，在Device Plugin 的设计和实现中，Google 的工程师们一直不太愿意为 Allocate 和 ListAndWatch API 添加可扩展性的参数。这就使得，当你确实需要处理一些比较复杂的硬件设备使用需求时，是没有办法通过扩展 Device Plugin 的 API来实现的。</p><p>针对这些问题，RedHat 在社区里曾经大力推进过 <a href="https://github.com/kubernetes/community/pull/2265">ResourceClass</a>的设计，试图将硬件设备的管理功能上浮到 API 层和调度层。但是，由于各方势力的反对，这个提议最后不了了之了。</p><p>所以说，目前 Kubernetes 本身的 Device Plugin 的设计，实际上能覆盖的场景是非常单一的，属于“可用”但是“不好用”的状态。并且， Device Plugin 的 API 本身的可扩展性也不是很好。这也就解释了为什么像 NVIDIA 这样的硬件厂商，实际上并没有完全基于上游的 Kubernetes 代码来实现自己的 GPU 解决方案，而是做了一定的改动，也就是 fork。这，实属不得已而为之。</p><h2>思考题</h2><p>请你结合自己的需求谈一谈，你希望如何对当前的 Device Plugin进行改进呢？或者说，你觉得当前的设计已经完全够用了吗？</p><p>感谢你的收听，欢迎你给我留言，也欢迎分享给更多的朋友一起阅读。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/bb/83/39c48a58.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Eric</span>
  </div>
  <div class="_2_QraFYR_0">单块GPU资源都不能共享，还得自己fork一份device plugin维护虚拟化的GPU。 社区有时候办事真的不利索</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我已经吐槽的很委婉了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-05 00:57:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/32/0b/81ae214b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>凌</span>
  </div>
  <div class="_2_QraFYR_0">https:&#47;&#47;mp.weixin.qq.com&#47;s&#47;NU8Cj6DL8wEKFzVYhuyzbQ</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-27 13:55:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/88/a0/562c2626.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小河</span>
  </div>
  <div class="_2_QraFYR_0">hi，张老师，我现在将gpu的服务迁移到kubernetes上，对外提供的是gRRC接口，我使用了ingres-nginx对gRPC进行负载均衡，但是发现支持并不好，又想使用Istio以sidecar模式代理gPRC，但是又觉得太重，请问目前有什么较好的方案在kuberntes支持对gRPC的负载均衡么😀</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-04 11:41:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/a2/94/ae0a60d8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>江山未</span>
  </div>
  <div class="_2_QraFYR_0">GPU共享及虚拟化，可以搜索一下Orion VGPU</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-22 19:40:51</div>
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
  <div class="_2_QraFYR_0">Kuberentes通过Extended Resource来支持自定义资源，比如GPU。为了让调度器知道这种自定义资源在各Node上的数量，需要的Node里添加自定义资源的数量。实际上，这些信息并不需要人工去维护，所有的硬件加速设备的管理都通过Device Plugin插件来支持，也包括对该硬件的Extended Resource进行汇报的逻辑。<br><br>Device Plugin 、kubelet、调度器如何协同工作：<br><br>汇报资源： Device Plugin通过gRPC与本机kubelet连接 -&gt;  Device Plugin定期向kubelet汇报设备信息，比如GPU的数量 -&gt; kubelet 向APIServer发送的心跳中，以Extended Reousrce的方式加上这些设备信息，比如GPU的数量 <br><br>调度： Pod申明需要一个GPU -&gt; 调度器找到GPU数量满足条件的node -&gt; Pod绑定到对应的Node上 -&gt; kubelet发现需要拉起一个Pod，且该Pod需要GPU -&gt; kubelet向 Device Plugin 发起 Allocate()请求 -&gt; Device Plugin根据kubelet传递过来的需求，找到这些设备对应的设备路径和驱动目录，并返回给kubelet -&gt; kubelet将这些信息追加在创建Pod所对应的CRI请求中 -&gt; 容器创建完成之后，就会出现这个GPU设备（设备路径+驱动目录）-&gt; 调度完成<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-20 22:28:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/4a/2f/42aa48d7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>勇敢的心</span>
  </div>
  <div class="_2_QraFYR_0">所以目前是无法实现多用户同时共享单块GPU咯？有没有可以实现这一功能的Magic？还有，目前可能实现GPU或者CPU数量的动态改变吗，在不重建pod的情况下？期待老师的解答</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-13 07:49:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/S8nYMkG2uByU9IpbAExZwLAa1no0RgKeeqjfBns0fVuBGU3GlVwia5BKujIX4648h9N2OMsyVVCLbKibje06HicvQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>乱愣黎</span>
  </div>
  <div class="_2_QraFYR_0">1、device plugin只能通过patch操作来实现device信息的添加吗？能否在节点添加的时候自动添加<br>2、在第1点的情况下，在服务器持续集成的情况下，新旧设备device信息肯定是会不一致的，如何解决device plugin机制无法区分设备属性的情况？<br>    以本篇文章的内容来看，可以这么设置<br>    批次A使用nvidia.com&#47;GP100=4，批次B使用amd.com&#47;VEGA64=4<br>    这样编写资源需求和新旧设备交替都需要人为指定，这样对于运维来说很难受啊<br>3、是否能把GPU抽象成类似于CPU的时间片，将整个GPU计算能力池化，然后根据pod.spec.containers.resources里面的require和limits字段来分配GPU计算资源</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-05 00:02:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/oyZfdsQL71tLcX1Eiav4NOMxa2yRSQmQNFzm7SV2aicfNkXoIN80DN2Iafpnmu2WVPBdlHylOWwElrVA8A8X71qQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>hlzhu1983</span>
  </div>
  <div class="_2_QraFYR_0">张老师，问一下现在k8s关于GPU资源调度粒度是否能像CPU调度粒度那么细？现在还只能按照1块GPU卡来分配GPU资源吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 很粗粒度呢</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-03 09:58:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/2e/72/145c10db.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>每日都想上班</span>
  </div>
  <div class="_2_QraFYR_0">今天爆出kubenetes安全漏洞需要升级，请问要如何升级<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-04 14:04:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/40/c3/e545ba80.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张振宇</span>
  </div>
  <div class="_2_QraFYR_0">老师，我们的2个pod经常出现共用一张gpu卡的情况，导致性能互相影响，求解救。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-15 09:38:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c0/49/e2a18264.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>PatHoo</span>
  </div>
  <div class="_2_QraFYR_0">请问现在K8S支持按硬件拓扑结构调度了吗? </div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-01 15:53:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/00/24/6ff7cb37.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>硕</span>
  </div>
  <div class="_2_QraFYR_0">刚公司需要 使用nvdia-docker 管理 gpu 用于部署ai 图像 这就来了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-03 00:48:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/uUwMSbicsSdrzWpnCBJKMgOpA6MzgztaqNr4w9kiciaH7wFlcsjd97cYhduMXyYicdV9r0vTTmqPReTYr6ia2S15meA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_4acba3</span>
  </div>
  <div class="_2_QraFYR_0">请教一下，如果根据GPU显存资源来调度有什么好办法吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-19 23:33:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>行道者</span>
  </div>
  <div class="_2_QraFYR_0">老师，请问如何管理多个不同k8s集群的GPU资源，业界有这样的方案吗？谢谢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-17 09:27:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/d9/ff/b23018a6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Heaven</span>
  </div>
  <div class="_2_QraFYR_0">现在的资源必然不够用,因为只能按照整数类型的分配,导致很多时候,不能共享资源<br>无法做到按需修改api</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-16 14:19:20</div>
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
  <div class="_2_QraFYR_0">按需增加api, google把这一块完全开放出来，应该是唯一的道路</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-23 16:33:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/2f/c6/b7448158.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Tarjintor</span>
  </div>
  <div class="_2_QraFYR_0">那么，理论上，可以做到对一个进程组的gpu使用百分比做限制吗？<br>之前对docker做介绍的时候，说过可以限制一个cpu所能使用的百分比</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-05 16:09:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/48/7b/bc7fcac2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Hank</span>
  </div>
  <div class="_2_QraFYR_0">kubeflow 能否解决此事呢？    粗颗粒 转换成细粒度</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-15 14:52:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/ad/6e/f08676bf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>🔜</span>
  </div>
  <div class="_2_QraFYR_0">[root@bigdata-k8s-master-1 ~]# curl --header &quot;Content-Type: application&#47;json-patch+json&quot; \<br>&gt; --request PATCH \<br>&gt; --data &#39;[{&quot;op&quot;: &quot;add&quot;, &quot;path&quot;: &quot;&#47;status&#47;capacity&#47;nvidia.com&#47;gpu&quot;, &quot;value&quot;: &quot;1&quot;}]&#39; \<br>&gt; http:&#47;&#47;localhost:8001&#47;api&#47;v1&#47;nodes&#47;k8s-master-01&#47;status<br>{<br>  &quot;kind&quot;: &quot;Status&quot;,<br>  &quot;apiVersion&quot;: &quot;v1&quot;,<br>  &quot;metadata&quot;: {<br><br>  },<br>  &quot;status&quot;: &quot;Failure&quot;,<br>  &quot;message&quot;: &quot;jsonpatch add operation does not apply: doc is missing path: &#47;status&#47;capacity&#47;nvidia.com&#47;gpu&quot;,<br>  &quot;code&quot;: 500<br><br>什么原因</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-13 17:20:09</div>
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
  <div class="_2_QraFYR_0">社区就是江湖，开源不是免费。<br>差异性如何体现，lol</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-22 14:08:25</div>
  </div>
</div>
</div>
</li>
</ul>