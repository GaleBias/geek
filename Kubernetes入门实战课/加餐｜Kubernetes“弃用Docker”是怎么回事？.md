<audio title="加餐｜Kubernetes“弃用Docker”是怎么回事？" src="https://static001.geekbang.org/resource/audio/df/86/df39c071f6a2475e605b41ca7b77c286.mp3" controls="controls"></audio> 
<p>你好，我是Chrono。</p><p>在“入门篇”学习容器技术的过程中，我看到有不少同学留言问Kubernetes“弃用Docker”的事情，担心现在学Docker是否还有价值，是否现在就应该切换到containerd或者是其他runtime。</p><p>这些疑虑的确是有些道理。两年前，Kubernetes放出消息要“弃用Docker”的时候，确确实实在Kubernetes社区里掀起了一场“轩然大波”，影响甚至波及到社区之外，也导致Kubernetes不得不写了好几篇博客来反复解释这么做的原因。</p><p>两年过去了，虽然最新的Kubernetes 1.24已经达成了“弃用”的目标，但很多人对这件事似乎还是没有非常清晰的认识。所以今天，我们就来聊聊这个话题，我也讲讲我的一些看法。</p><p><img src="https://static001.geekbang.org/resource/image/ec/9a/ece7f8245a02a5ca52a51c79b6f3ea9a.png?wh=1162x600" alt="图片" title="图片来自网络"></p><h2>什么是CRI</h2><p>要了解Kubernetes为什么要“弃用Docker”，还得追根溯源，回头去看Kubernetes的发展历史。</p><p>2014年，Docker正如日中天，在容器领域没有任何对手，而这时Kubernetes才刚刚诞生，虽然背后有Google和Borg的支持，但还是比较弱小的。所以，Kubernetes很自然就选择了在Docker上运行，毕竟“背靠大树好乘凉”，同时也能趁机“养精蓄锐”逐步发展壮大自己。</p><!-- [[[read_end]]] --><p>时间一转眼到了2016年，CNCF已经成立一年了，而Kubernetes也已经发布了1.0版，可以正式用于生产环境，这些都标志着Kubernetes已经成长起来了，不再需要“看脸色吃饭”。于是它就宣布加入了CNCF，成为了第一个CNCF托管项目，想要借助基金会的力量联合其他厂商，一起来“扳倒”Docker。</p><p>那它是怎么做的呢？</p><p>在2016年底的1.5版里，Kubernetes引入了一个新的接口标准：CRI ，Container Runtime Interface。</p><p>CRI采用了ProtoBuffer和gPRC，规定kubelet该如何调用容器运行时去管理容器和镜像，但这是一套全新的接口，和之前的Docker调用完全不兼容。</p><p>Kubernetes意思很明显，就是不想再绑定在Docker上了，允许在底层接入其他容器技术（比如rkt、kata等），随时可以把Docker“踢开”。</p><p>但是这个时候Docker已经非常成熟，而且市场的惯性也非常强大，各大云厂商不可能一下子就把Docker全部替换掉。所以Kubernetes也只能同时提供<strong>一个“折中”方案，在kubelet和Docker中间加入一个“适配器”，把Docker的接口转换成符合CRI标准的接口</strong>（<a href="https://kubernetes.io/blog/2016/12/container-runtime-interface-cri-in-kubernetes/">图片来源</a>）：</p><p><img src="https://static001.geekbang.org/resource/image/11/ef/11e3de04b296248711455f22ce5578ef.png?wh=572x136" alt="图片"></p><p>因为这个“适配器”夹在kubelet和Docker之间，所以就被形象地称为是“shim”，也就是“垫片”的意思。</p><p>有了 CRI和shim，虽然Kubernetes还使用Docker作为底层运行时，但也具备了和Docker解耦的条件，从此就拉开了“弃用Docker”这场大戏的帷幕。</p><h2>什么是containerd</h2><p>面对Kubernetes“咄咄逼人”的架势，Docker是看在眼里痛在心里，虽然有苦心经营了多年的社区和用户群，但公司的体量太小，实在是没有足够的实力与大公司相抗衡。</p><p>不过Docker也没有“坐以待毙”，而是采取了“断臂求生”的策略，推动自身的重构，<strong>把原本单体架构的Docker Engine拆分成了多个模块，其中的Docker daemon部分就捐献给了CNCF，形成了containerd</strong>。</p><p>containerd作为CNCF的托管项目，自然是要符合CRI标准的。但Docker出于自己诸多原因的考虑，它只是在Docker Engine里调用了containerd，外部的接口仍然保持不变，也就是说还不与CRI兼容。</p><p>由于Docker的“固执己见”，这时Kubernetes里就出现了两种调用链：</p><ul>
<li>第一种是用CRI接口调用dockershim，然后dockershim调用Docker，Docker再走containerd去操作容器。</li>
<li>第二种是用CRI接口直接调用containerd去操作容器。</li>
</ul><p><img src="https://static001.geekbang.org/resource/image/a8/9b/a8abfe5a55d0fa8b383867cc6062089b.png?wh=1920x627" alt="图片" title="图片来自网络"></p><p>显然，由于都是用containerd来管理容器，所以这两种调用链的最终效果是完全一样的，但是第二种方式省去了dockershim和Docker Engine两个环节，更加简洁明了，损耗更少，性能也会提升一些。</p><p>在2018年Kubernetes 1.10发布的时候，containerd也更新到了1.1版，正式与Kubernetes集成，同时还发表了一篇博客文章（<a href="https://kubernetes.io/blog/2018/05/24/kubernetes-containerd-integration-goes-ga/">https://kubernetes.io/blog/2018/05/24/kubernetes-containerd-integration-goes-ga/</a>），展示了一些性能测试数据：</p><p><img src="https://static001.geekbang.org/resource/image/6f/e9/6fd065d916e5815e044c10738746ace9.jpg?wh=1784x591" alt="图片"></p><p>从这些数据可以看到，containerd1.1相比当时的Docker 18.03，Pod的启动延迟降低了大约20%，CPU使用率降低了68%，内存使用率降低了12%，这是一个相当大的性能改善，对于云厂商非常有诱惑力。</p><h2>正式“弃用Docker”</h2><p>有了CRI和containerd这两件强大的武器，胜利的天平已经明显向Kubernetes倾斜了。</p><p>又是两年之后，到了2020年，Kubernetes 1.20终于正式向Docker“宣战”：kubelet将弃用Docker支持，并会在未来的版本中彻底删除。</p><p>但由于Docker几乎成为了容器技术的代名词，而且Kubernetes也已经使用Docker很多年，这个声明在不断传播的过程中很快就“变味”了，“kubelet将弃用Docker支持”被简化成了更吸引眼球的“Kubernetes将弃用Docker”。</p><p>这自然就在IT界引起了恐慌，“不明真相的广大群众”纷纷表示震惊：用了这么久的Docker突然就不能用了，Kubernetes为什么要如此对待Docker？之前在Docker上的投入会不会就全归零了？现有的大量镜像该怎么办？</p><p>其实，如果你理解了前面讲的CRI和containerd这两个项目，就会知道Kubernetes的这个举动也没有什么值得大惊小怪的，一切都是“水到渠成”的：<strong>它实际上只是“弃用了dockershim”这个小组件，也就是说把dockershim移出了kubelet，并不是“弃用了Docker”这个软件产品。</strong></p><p>所以，“弃用Docker”对Kubernetes和Docker来说都不会有什么太大的影响，因为他们两个都早已经把下层都改成了开源的containerd，原来的Docker镜像和容器仍然会正常运行，唯一的变化就是Kubernetes绕过了Docker，直接调用Docker内部的containerd而已。</p><p>这个关系你可以参考下面的<a href="https://kubernetes.io/blog/2018/05/24/kubernetes-containerd-integration-goes-ga/">这张图</a>来理解：</p><p><img src="https://static001.geekbang.org/resource/image/97/e8/970a234bd610b55340505dac74b026e8.png?wh=740x680" alt="图片"></p><p>当然，影响也不是完全没有。如果Kubernetes直接使用containerd来操纵容器，那么它就是一个与Docker独立的工作环境，彼此都不能访问对方管理的容器和镜像。换句话说，使用命令 <code>docker ps</code> 就看不到在Kubernetes里运行的容器了。</p><p>这对有的人来说可能需要稍微习惯一下，改用新的工具 <code>crictl</code>，不过用来查看容器、镜像的子命令还是一样的，比如 <code>ps</code>、<code>images</code> 等等，适应起来难度不大（但如果我们一直用kubectl来管理Kubernetes的话，这就是没有任何影响了）。</p><p>“宣战”之后，Kubernetes原本打算用一年的时间完成“弃用Docker”的工作，但它也确实低估了Docker的根基，到了1.23版还是没能移除dockershim，不得已又往后推迟了半年，终于在今年5月份发布的1.24版把dockershim的代码从kubelet里删掉了。</p><p>自此，Kubernetes彻底和Docker“分道扬镳”，今后就是“大路朝天，各走一边”。</p><h2>Docker的未来</h2><p>那么，Docker的未来会是怎么样的呢？难道云原生时代就没有它的立足之地了吗？</p><p>这个问题的答案很显然是否定的。</p><p>作为容器技术的初创者，Docker的历史地位无人能够质疑，虽然现在Kubernetes不再默认绑定Docker，但Docker还是能够以其他的形式与Kubernetes共存的。</p><p>首先，因为<strong>容器镜像格式已经被标准化</strong>了（OCI规范，Open Container Initiative），Docker镜像仍然可以在Kubernetes里正常使用，原来的开发测试、CI/CD流程都不需要改动，我们仍然可以拉取Docker Hub上的镜像，或者编写Dockerfile来打包应用。</p><p>其次，<strong>Docker是一个完整的软件产品线</strong>，不止是containerd，它还包括了镜像构建、分发、测试等许多服务，甚至在Docker Desktop里还内置了Kubernetes。</p><p>单就容器开发的便利性来讲，Docker还是暂时难以被替代的，广大云原生开发者可以在这个熟悉的环境里继续工作，利用Docker来开发运行在Kubernetes里的应用。</p><p>再次，虽然Kubernetes已经不再包含dockershim，但Docker公司却把这部分代码接管了过来，另建了一个叫<strong>cri-dockerd</strong>（<a href="https://github.com/mirantis/cri-dockerd">https://github.com/mirantis/cri-dockerd</a>）的项目，作用也是一样的，把Docker Engine适配成CRI接口，这样kubelet就又可以通过它来操作Docker了，就仿佛是一切从未发生过。</p><p>综合来看，Docker虽然在容器编排战争里落败，被Kubernetes排挤到了角落，但它仍然具有强韧的生命力，多年来积累的众多忠实用户和数量庞大的应用镜像是它的最大资本和后盾，足以支持它在另一条不与Kubernetes正面交锋的道路上走下去。</p><p>而对于我们这些初学者来说，Docker方便易用，具有完善的工具链和友好的交互界面，市面上很难找到能够与它媲美的软件了，应该说是入门学习容器技术和云原生的“不二之选”。至于Kubernetes底层用的什么，我们又何必太过于执着和关心呢？</p><h2>课下作业</h2><p>虽然今天的内容是加餐，但我还是给你留个思考题吧，我们可以一起讨论一下：</p><p>Docker重构自身，分离出containerd，这是否算是一种“自掘坟墓”的行为呢？如果没有containerd，那现在的情形会是怎么样的呢？</p><p>欢迎在留言区参与讨论，如果觉得这篇文章对你有帮助，也欢迎你分享给朋友。我们下节课再见。</p><p><img src="https://static001.geekbang.org/resource/image/a5/f2/a561d280091d2b59a935c6be38f646f2.jpg?wh=1920x1888" alt="图片"></p>
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
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/WrANpwBMr6DsGAE207QVs0YgfthMXy3MuEKJxR8icYibpGDCI1YX4DcpDq1EsTvlP8ffK1ibJDvmkX9LUU4yE8X0w/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>星垂平野阔</span>
  </div>
  <div class="_2_QraFYR_0">docker分离containerd是一个很聪明的举动！与其将来被人分离或者抛弃不用，不如我主动革新，把Kubernates绑在我的战车上，这样cri的第一选择仍然是docker的自己人。<br>一时的退让是为了更好的将来。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-13 08:20:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/67/d6/2f5cb85c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>xmr</span>
  </div>
  <div class="_2_QraFYR_0">1.主要还是docker公司话语权逐渐式微，你没有话语权说你不行就不行，docker不得不换一种方式(containerd)陪在你身边。<br>2.容器技术本身门槛就不高，形成不了技术壁垒，没有containerd还会有containere、containerf之类的代替者。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-13 14:15:20</div>
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
  <div class="_2_QraFYR_0">老师请教一个问题，很是疑惑。<br>问题是这样的，runc是一个按OCI规范运行容器的cli工具，如运行时containerd默认使用runc。<br>我看podman也是通过conmon调用runc运行容器，请问下podman运行容器使用的容器运行时是什么？像docker现在就是通过containerd来调用runc，所以containerd就是docker容器的运行时，从这个角度来看conmon是不是podman容器的运行时？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: runc是OCI规范中的容器底层部分，也就是原来docker的libcontainer，它调用namespace、cgroup等系统接口创建容器。<br><br>podman不太了解。运行时这个概念比较模糊，从Kubernetes角度来看containerd就是运行时，再往下runc有是运行时。<br><br>我觉得概念上不用太纠结，把精力用在更有价值的地方。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-14 16:41:39</div>
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
  <div class="_2_QraFYR_0">containerd 实现了 CRI 规范，来管理镜像和运行容器，默认和 k8s 绑定，那 k8s 还可以自由替换运行时吗？ 做到 可插拔</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Kubernetes现在没有默认绑定在containerd上，cri接口通用，换哪个都可以。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-13 21:28:43</div>
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
  <div class="_2_QraFYR_0">容器运行时只是对Linux的namespace, cgroup, rootfs进行了产品化打包的一种技术，原则上来讲，如果Docker不把运行时containerd分离标准化CRI出来，Kubernetes其实也可以造出新轮子，毕竟现在的容器运行时也有很多，像cri-o, kata, mcr等，当然不知道会不会有啥版权侵犯到moby的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-13 20:59:49</div>
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
  <div class="_2_QraFYR_0">请教老师两个问题：<br>Q1：有人提出了这个观点：“数据库不适合用docker，不管是mysql还是ES”，这个观点对吗？<br>Q2：关于容器ID和imageID，我的理解是：容器ID是随机，在同一台宿主机上每次创建的ID都不同，而且同一个镜像在不同机器上创建的ID也不同。但image ID是唯一的、固定的，对于同一个版本，第N次下载和第M次下载的image ID是相同的，同一个image，下载到不同的机器上，image ID也是相同的。 我的理解对吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.这个不能说全对也全错，我不是数据库专家，但这些产品早都有docker镜像了。<br><br>2.镜像是只读的，所以image id必然不可变。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-13 20:20:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2e/09/d0/8609bddc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>戒贪嗔痴</span>
  </div>
  <div class="_2_QraFYR_0">为啥突然冒出个containerd让我一时不知所措，它的作用是什么，是作为守护进程存在吗？既然分离出的containerd没有兼容的接口，Google完全可以不用的，可能是不想重复造轮子吧。我就感觉怎么冒出个containerd就挺突然的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: containerd就是原来的docker daemon，是用来管理调度容器的，不是凭空出现。<br><br>它是Kubernetes的必备组件container-runtime，不过也可以替换成CRI-O，因为都符合cri。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-13 19:16:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/10/bb/f1061601.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Demon.Lee</span>
  </div>
  <div class="_2_QraFYR_0">看完罗老师这篇，再推荐一下隔壁周老师的（ https:&#47;&#47;time.geekbang.org&#47;column&#47;article&#47;351014 ）一篇。在两位大佬的加持下，Kubernetes 与 Docker 相爱想杀的故事，想不懂都难 😂</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: nice！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-08 20:20:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/0c/86/8e52afb8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>花花大脸猫</span>
  </div>
  <div class="_2_QraFYR_0">没办法呀。。毕竟依托于k8s的环境运行，别人给了标准，你不调整，那只能被替换了！毕竟谁也不是不可替代的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-28 11:34:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/f0/50/c348c2ea.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>吕伟</span>
  </div>
  <div class="_2_QraFYR_0">是不是可以这么理解“K8S弃用docker”的事情：<br>K8S在1.24版本后去除了容器使用docker的默认支持，要使用docker作为容器节点只是比以前版本稍微多做一些操作（如：加装插件或者修改哪里的配置），然后也是可以正常像以前一样正常使用docker的。这点是因为docker容器技术符合CRI标准，就算表面脱离了，实际也是兼容。<br><br>最后有一点好奇，K8S弃用docker作默认，那么是哪个容器上位成功，但是说只是K8S初始化时要多做选型配置而尔。就好比window系统默认预装ie浏览器，然后有个超纯净版要自行安装浏览器，这个比喻对吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Kubernetes定义CRI就是要和下层的容器运行时解耦，所以不会对哪个产品有偏向，只是containerd源自Docker，用户的习惯影响会更大一些。<br><br>现在Kubernetes没有默认的容器运行时，必须用户自己安装，所以后面的比喻不太贴切。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-18 11:48:54</div>
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
  <div class="_2_QraFYR_0">课外小贴士中的“正式毕业”是什么意思呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个应该是cncf基金会的术语，大概意思表示得到了业界的普遍承认，可以用于生产环境，具体的可以再查一下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-13 12:33:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/1d/de/62bfa83f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>aoe</span>
  </div>
  <div class="_2_QraFYR_0">放心了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-13 09:55:10</div>
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
  <div class="_2_QraFYR_0">我第一反应也是自掘坟墓，我知道是胳膊拧不过大腿，但也可以拧一拧啊。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 各抒己见，我还是很钦佩docker的勇气的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-13 08:45:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/27/ce/58/71ed845f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Dexter</span>
  </div>
  <div class="_2_QraFYR_0">IDL —-接口描述语言，那schema是什么意思？一些接口格式或者规范吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: schema没太合适的翻译，差不多就是格式、规范的意思。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-13 08:08:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/d0/51/f1c9ae2d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Christopher</span>
  </div>
  <div class="_2_QraFYR_0">感谢作者把这个弃用docker的疑惑解决了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: thanks。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-13 07:53:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/ab/c3/60a3255a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Marvin</span>
  </div>
  <div class="_2_QraFYR_0">上面有张图片写有dockerd, 如果containerd是原来的docker deamon, 那这个dockerd是什么呢，谢谢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: dockerd是原来的docker接口，提供docker兼容服务，与containerd是不同的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-22 13:06:04</div>
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
  <div class="_2_QraFYR_0">课程由浅入深，娓娓道来，能看出作者从零小白基础入门角度考虑，非常好的课程。点赞。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: many thanks.</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-28 10:04:47</div>
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
  <div class="_2_QraFYR_0">这故事讲的太详细生动了！好奇老师都是从哪里听说这些的呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 有博客，还有当时的一些讨论帖子，再适当加上一些自己的演绎，笑。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-17 10:36:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1c/39/85/c6110f83.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>黄骏</span>
  </div>
  <div class="_2_QraFYR_0">我是说为啥rockylinux 9.0虚机里多了cri-dockerd和containerd、podman呢，原来有这一段历史。<br>老师的演示环境，是默认非root用户吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，我一般很少用root，习惯了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-03 16:23:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/87/5f/6bf8b74a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Kepler</span>
  </div>
  <div class="_2_QraFYR_0">Kubernetes 正式从CNCF毕业，这里的“毕业”是什么意思？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: cncf基金会的术语，大概意思表示得到了业界的普遍承认，可以用于生产环境。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-29 21:02:42</div>
  </div>
</div>
</div>
</li>
</ul>