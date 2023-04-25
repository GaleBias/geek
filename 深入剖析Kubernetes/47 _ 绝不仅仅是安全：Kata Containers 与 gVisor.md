<audio title="47 _ 绝不仅仅是安全：Kata Containers 与 gVisor" src="https://static001.geekbang.org/resource/audio/82/24/82703193a84be5389db43f9867223d24.mp3" controls="controls"></audio> 
<p>你好，我是张磊。今天我和你分享的主题是：绝不仅仅是安全之Kata Containers 与 gVisor。</p><p>在上一篇文章中，我为你详细地讲解了 kubelet 和 CRI 的设计和具体的工作原理。而在讲解 CRI 的诞生背景时，我也提到过，这其中的一个重要推动力，就是基于虚拟化或者独立内核的安全容器项目的逐渐成熟。</p><p>使用虚拟化技术来做一个像 Docker 一样的容器项目，并不是一个新鲜的主意。早在 Docker 项目发布之后，Google 公司就开源了一个实验性的项目，叫作 novm。这，可以算是试图使用常规的虚拟化技术来运行 Docker 镜像的第一次尝试。不过，novm 在开源后不久，就被放弃了，这对于 Google 公司来说或许不算是什么新鲜事，但是 novm 的昙花一现，还是激发出了很多内核开发者的灵感。</p><p>所以在2015年，几乎在同一个星期，Intel OTC （Open Source Technology Center） 和国内的 HyperHQ 团队同时开源了两个基于虚拟化技术的容器实现，分别叫做 Intel Clear Container 和 runV 项目。</p><p>而在2017年，借着 Kubernetes 的东风，这两个相似的容器运行时项目在中立基金会的撮合下最终合并，就成了现在大家耳熟能详的 Kata Containers 项目。 由于 Kata Containers 的本质就是一个精简后的轻量级虚拟机，所以它的特点，就是“像虚拟机一样安全，像容器一样敏捷”。</p><!-- [[[read_end]]] --><p>而在2018年，Google 公司则发布了一个名叫 gVisor 的项目。gVisor 项目给容器进程配置一个用 Go 语言实现的、运行在用户态的、极小的“独立内核”。这个内核对容器进程暴露 Linux 内核 ABI，扮演着“Guest Kernel”的角色，从而达到了将容器和宿主机隔离开的目的。</p><p>不难看到，无论是 Kata Containers，还是 gVisor，它们实现安全容器的方法其实是殊途同归的。这两种容器实现的本质，都是给进程分配了一个独立的操作系统内核，从而避免了让容器共享宿主机的内核。这样，容器进程能够看到的攻击面，就从整个宿主机内核变成了一个极小的、独立的、以容器为单位的内核，从而有效解决了容器进程发生“逃逸”或者夺取整个宿主机的控制权的问题。这个原理，可以用如下所示的示意图来表示清楚。</p><p><img src="https://static001.geekbang.org/resource/image/95/1d/959c4c40c767acb6a3ffe6e144202e1d.png?wh=1108*783" alt=""></p><p>而它们的区别在于，Kata Containers 使用的是传统的虚拟化技术，通过虚拟硬件模拟出了一台“小虚拟机”，然后在这个小虚拟机里安装了一个裁剪后的 Linux 内核来实现强隔离。</p><p>而 gVisor 的做法则更加激进，Google 的工程师直接用 Go 语言“模拟”出了一个运行在用户态的操作系统内核，然后通过这个模拟的内核来代替容器进程向宿主机发起有限的、可控的系统调用。</p><p>接下来，我就来为你详细解读一下 KataContainers 和 gVisor 具体的设计原理。</p><p><strong>首先，我们来看 KataContainers</strong>。它的工作原理可以用如下所示的示意图来描述。</p><p><img src="https://static001.geekbang.org/resource/image/8d/89/8d7bbc8acaf27adff890f0be637df889.png?wh=1876*514" alt=""></p><p>我们前面说过，Kata Containers 的本质，就是一个轻量化虚拟机。所以当你启动一个 Kata Containers 之后，你其实就会看到一个正常的虚拟机在运行。这也就意味着，一个标准的虚拟机管理程序（Virtual Machine Manager, VMM）是运行 Kata Containers 必备的一个组件。在我们上面图中，使用的 VMM 就是 Qemu。</p><p>而使用了虚拟机作为进程的隔离环境之后，Kata Containers 原生就带有了 Pod 的概念。即：这个Kata Containers 启动的虚拟机，就是一个 Pod；而用户定义的容器，就是运行在这个轻量级虚拟机里的进程。在具体实现上，Kata Containers 的虚拟机里会有一个特殊的 Init 进程负责管理虚拟机里面的用户容器，并且只为这些容器开启 Mount Namespace。所以，这些用户容器之间，原生就是共享 Network 以及其他Namespace 的。</p><p>此外，为了跟上层编排框架比如 Kubernetes 进行对接，Kata Containers 项目会启动一系列跟用户容器对应的 shim 进程，来负责操作这些用户容器的生命周期。当然，这些操作，实际上还是要靠虚拟机里的 Init 进程来帮你做到。</p><p>而在具体的架构上，Kata Containers的实现方式同一个正常的虚拟机其实也非常类似。这里的原理，可以用如下所示的一幅示意图来表示。</p><p><img src="https://static001.geekbang.org/resource/image/16/f3/1684d0d89c170c2f8e6d050919c883f3.jpg?wh=2284*1900" alt=""></p><p>可以看到，当 Kata Containers 运行起来之后，虚拟机里的用户进程（容器），实际上只能看到虚拟机里的、被裁减过的 Guest Kernel，以及通过 Hypervisor 虚拟出来的硬件设备。</p><p>而为了能够对这个虚拟机的 I/O 性能进行优化，Kata Containers 也会通过 vhost 技术（比如：vhost-user）来实现 Guest 与 Host 之间的高效的网络通信，并且使用 PCI Passthrough （PCI 穿透）技术来让 Guest 里的进程直接访问到宿主机上的物理设备。这些架构设计与实现，其实跟常规虚拟机的优化手段是基本一致的。</p><p>相比之下，gVisor 的设计其实要更加“激进”一些。它的原理，可以用如下所示的示意图来表示清楚。</p><p><img src="https://static001.geekbang.org/resource/image/2f/7b/2f7903a7c494ddf6989d00c794bd7a7b.png?wh=1082*298" alt=""></p><p>gVisor工作的核心，在于它为应用进程、也就是用户容器，启动了一个名叫 Sentry 的进程。 而Sentry 进程的主要职责，就是提供一个传统的操作系统内核的能力，即：运行用户程序，执行系统调用。所以说，Sentry 并不是使用 Go 语言重新实现了一个完整的 Linux 内核，而只是一个对应用进程“冒充”内核的系统组件。</p><p>在这种设计思想下，我们就不难理解，Sentry 其实需要自己实现一个完整的 Linux 内核网络栈，以便处理应用进程的通信请求。然后，把封装好的二层帧直接发送给 Kubernetes 设置的 Pod 的Network Namespace 即可。</p><p>此外，Sentry 对于Volume 的操作，则需要通过 9p 协议交给一个叫做 Gofer 的代理进程来完成。Gofer 会代替应用进程直接操作宿主机上的文件，并依靠seccomp机制将自己的能力限制在最小集，从而防止恶意应用进程通过 Gofer 来从容器中“逃逸”出去。</p><p>而在具体的实现上，gVisor 的 Sentry 进程，其实还分为两种不同的实现方式。这里的工作原理，可以用下面的示意图来描述清楚。</p><p><img src="https://static001.geekbang.org/resource/image/5a/b8/5a1d6e0291306417864033b3f40f74b8.png?wh=1102*730" alt=""></p><p><strong>第一种实现方式</strong>，是使用Ptrace机制来拦截用户应用的系统调用（System Call），然后把这些系统调用交给 Sentry 来进行处理。</p><p>这个过程，对于应用进程来说，是完全透明的。而 Sentry 接下来，则会扮演操作系统的角色，在用户态执行用户程序，然后仅在需要的时候，才向宿主机发起 Sentry 自己所需要执行的系统调用。这，就是 gVisor 对用户应用进程进行强隔离的主要手段。不过， Ptrace 进行系统调用拦截的性能实在是太差，仅能供 Demo 时使用。</p><p>而<strong>第二种实现方式</strong>，则更加具有普适性。它的工作原理如下图所示。</p><p><img src="https://static001.geekbang.org/resource/image/3f/bf/3faf90550425378be91eb8cd2f0c63bf.png?wh=947*810" alt=""></p><p>在这种实现里，Sentry 会使用 KVM 来进行系统调用的拦截，这个性能比 Ptrace 就要好很多了。</p><p>当然，为了能够做到这一点，Sentry 进程就必须扮演一个 Guest Kernel 的角色，负责执行用户程序，发起系统调用。而这些系统调用被 KVM 拦截下来，还是继续交给 Sentry 进行处理。只不过在这时候，Sentry 就切换成了一个普通的宿主机进程的角色，来向宿主机发起它所需要的系统调用。</p><p>可以看到，<strong>在这种实现里，Sentry 并不会真的像虚拟机那样去虚拟出硬件设备、安装 Guest 操作系统。它只是借助 KVM 进行系统调用的拦截，以及处理地址空间切换等细节。</strong></p><p>值得一提的是，在 Google 内部，他们也是使用的第二种基于 Hypervisor 的gVisor 实现。只不过 Google 内部有自己研发的 Hypervisor，所以要比 KVM 实现的性能还要好。</p><p>通过以上的讲述，相信你对 Kata Containers 和 gVisor 的实现原理，已经有一个感性的认识了。需要指出的是，到目前为止，gVisor 的实现依然不是非常完善，有很多 Linux系统调用它还不支持；有很多应用，在 gVisor 里还没办法运行起来。 此外，gVisor也暂时没有实现一个 Pod 多个容器的支持。当然，在后面的发展中，这些工程问题一定会逐渐解决掉的。</p><p>另外，你可能还听说过 AWS 在2018年末发布的一个叫做 Firecracker 的安全容器项目。这个项目的核心，其实是一个用 Rust 语言重新编写的 VMM（即：虚拟机管理器）。这就意味着， Firecracker 和 Kata Containers 的本质原理，其实是一样的。只不过， Kata Containers 默认使用的 VMM 是 Qemu，而 Firecracker，则使用自己编写的 VMM。所以，理论上，Kata Containers 也可以使用 Firecracker 运行起来。</p><h2>总结</h2><p>在本篇文章中，我为你详细地介绍了拥有独立内核的安全容器项目，对比了 KataContainers 和 gVisor 的设计与实现细节。</p><p>在性能上，KataContainers 和 KVM 实现的 gVisor 基本不分伯仲，在启动速度和占用资源上，基于用户态内核的 gVisor 还略胜一筹。但是，对于系统调用密集的应用，比如重 I/O 或者重网络的应用，gVisor 就会因为需要频繁拦截系统调用而出现性能急剧下降的情况。此外，gVisor 由于要自己使用 Sentry 去模拟一个Linux 内核，所以它能支持的系统调用是有限的，只是 Linux 系统调用的一个子集。</p><p>不过，gVisor 虽然现在没有任何优势，但是这种通过在用户态运行一个操作系统内核，来为应用进程提供强隔离的思路，的确是未来安全容器进一步演化的一个非常有前途的方向。</p><p>值得一提的是，Kata Containers 团队在 gVisor 之前，就已经 Demo 了一个名叫 Linuxd 的项目。这个项目，使用了 User Mode Linux (UML)技术，在用户态运行起了一个真正的 Linux Kernel 来为应用进程提供强隔离，从而避免了重新实现 Linux Kernel 带来的各种麻烦。</p><p>有兴趣的话，你可以<a href="https://lc32018.sched.com/event/ER8x/run-linux-kernel-as-a-daemon-lai-jiangshan-hypersh">在这里查看</a>这个演讲。我相信，这个方向，应该才是安全容器进化的未来。这比 Unikernels 这种根本不适合实际场景中使用的思路，要靠谱得多。</p><blockquote>
<p>本篇图片出处均引自<a href="https://www.openstack.org/assets/presentation-media/kata-containers-and-gvisor-a-quantitave-comparison.pdf"> Kata Containers 的官方对比资料</a>。</p>
</blockquote><h2>思考题</h2><p>安全容器的意义，绝不仅仅止于安全。你可以想象一下这样一个场景：比如，你的宿主机的 Linux 内核版本是3.6，但是应用却必须要求 Linux 内核版本是4.0。这时候，你就可以把这个应用运行在一个 KataContainers 里。那么请问，你觉得使用 gVisor 是否也能提供这种能力呢？原因是什么呢？</p><p>感谢你的收听，欢迎你给我留言，也欢迎分享给更多的朋友一起阅读。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/03/5e/818a8b1b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Alex</span>
  </div>
  <div class="_2_QraFYR_0">回答最后的问题：<br>提供不了；因为gVisor是基于用户态内核的，无法真正做到与宿主机内核不一致的请求响应，因此满足不了对高版本内核请求的需求</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-10 10:52:13</div>
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
  <div class="_2_QraFYR_0">get到了重点：“gVisor 虽然现在没有任何优势，但是这种通过在用户态运行一个操作系统内核，来为应用进程提供强隔离的思路，的确是未来安全容器进一步演化的一个非常有前途的方向。”</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 聪明</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-15 09:39:13</div>
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
  <div class="_2_QraFYR_0">无法提供，gVisor说白了只是一层代理，不是真正的内核；但是这也是google的牛逼之处，把所有的系统调用和协议都代理出来，不是大神级的技术专家，这种活根本做不了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-30 15:58:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/1c/e3/b8c5bfa2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>🐵</span>
  </div>
  <div class="_2_QraFYR_0">老师，你好，能不能推荐下K8S可以通过哪些项目管理KVM虚拟机</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: redhat 的 kubevirt</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-08 23:40:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/a5/08/10b18682.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>混沌渺无极</span>
  </div>
  <div class="_2_QraFYR_0">思考题理解:<br>gvisor是宿主机kernel的&quot;客户端&quot;，不管怎么拦截容器的调用，最终还是要转为对宿主机kernel的调用，因此低版本的宿主机kernel没有的能力返回容器想要的响应。这时候还是虚拟机管用，因为甚至能在Linux上虚拟出window。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-10 03:49:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/GDYkD2X7pXSKUSaUFC8u3TBPaakaibnOBV2NYDc2TNfb8Su9icFMwSod6iaQX5iaGU2gT6xkPuhXeWvY8KaVEZAYzg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>extraterrestrial！！</span>
  </div>
  <div class="_2_QraFYR_0">之前的留言不能编辑，还向请教个问题：gvisor拦截系统调用，但是系统调用是内核态才能执行，gvisor拦截下来除了直接执行系统调用还有啥操作空间么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个就很典型的用户态kernel技术啊，可以参考user mode linux</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-15 16:46:35</div>
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
  <div class="_2_QraFYR_0">Kata Container本质就是一个精简后的轻量级虚拟机，所以它的特点，就是“像虚拟机一样安全，像容器一样敏捷”。2018年，Google发布了gVisor，给容器进程配置了一个用Go语言实现的“极小独立内核”。<br><br>无论是Kata Container还是gVisor，实现安全的方式本质上都是让容器独立一个操作系统内核，避免共享宿主机内核。他们的主要区别是，Kata 通过硬件虚拟化出一台“小虚拟机”，在“小虚拟机”安装裁剪后的Linux内核，而gVisor则是“模拟”出了一个运行在用户态的操作系统内核，通过这个内核来限制容器进程向宿主机发起的调用。<br><br>gVisor虽然现在没有任何优势，但是这种通过在用户态运行一个操作系统内核，来为应用进程提供强隔离的思路，的确是未来安全容器进一步演化的一个非常有前途的方向。<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-23 19:58:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/GDYkD2X7pXSKUSaUFC8u3TBPaakaibnOBV2NYDc2TNfb8Su9icFMwSod6iaQX5iaGU2gT6xkPuhXeWvY8KaVEZAYzg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>extraterrestrial！！</span>
  </div>
  <div class="_2_QraFYR_0">问个初级问题：看下来觉得kata使用了qemu，和我自己在qemu上跑个正常的ubuntu啥的系统好像没啥区别，就是**做了了精剪**? 这样精剪以后是个什么内核，功能够用么，会不会性能和现在的container差很多之类的...</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: linux内核，大部分够用，性能要看具体哪种情况，有没有开优化</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-15 16:38:36</div>
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
  <div class="_2_QraFYR_0">请教一下老师，Singularity容器是不是也是基于用户态的容器呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-06 08:42:26</div>
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
  <div class="_2_QraFYR_0">Kata容器对vmm的层的依赖不仅是qemu吧？还需要kvm吧？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-20 12:14:04</div>
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
  <div class="_2_QraFYR_0">不能运行在gVisor中,gVisor本质上是委托给了实际的宿主机内核去执行,有些高版本的api,低版本再如何也无法实现</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-17 22:26:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/7b/2a/7d8b5943.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>LH</span>
  </div>
  <div class="_2_QraFYR_0">要掌握的东西太多了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-16 14:11:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/ca/d5/a2cd57c9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小明root</span>
  </div>
  <div class="_2_QraFYR_0">后续课程中会讲解route这个概念么？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-10 15:58:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTIbKCwLRY5icgW3WtLn0JD7EDksicGFqLAsXTm89SjNhR0KIt3N5iaQEmeic4Ld50Yxsceicia5kBODibZPA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>starnop</span>
  </div>
  <div class="_2_QraFYR_0">磊哥您好，有个疑问请教一下，kata结合了runv与clear container，那么runv与clear container之间的区别是什么呢？各有什么优势？kata相比于他们又做了哪些优化呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 没啥大区别，所以才合并了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-10 09:53:12</div>
  </div>
</div>
</div>
</li>
</ul>