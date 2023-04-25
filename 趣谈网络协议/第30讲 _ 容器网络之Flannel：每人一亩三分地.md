<audio title="第30讲 _ 容器网络之Flannel：每人一亩三分地" src="https://static001.geekbang.org/resource/audio/43/d9/43537d3108145697419700fac4ad6ad9.mp3" controls="controls"></audio> 
<p>上一节我们讲了容器网络的模型，以及如何通过NAT的方式与物理网络进行互通。</p><p>每一台物理机上面安装好了Docker以后，都会默认分配一个172.17.0.0/16的网段。一台机器上新创建的第一个容器，一般都会给172.17.0.2这个地址，当然一台机器这样玩玩倒也没啥问题。但是容器里面是要部署应用的，就像上一节讲过的一样，它既然是集装箱，里面就需要装载货物。</p><p>如果这个应用是比较传统的单体应用，自己就一个进程，所有的代码逻辑都在这个进程里面，上面的模式没有任何问题，只要通过NAT就能访问进来。</p><p>但是因为无法解决快速迭代和高并发的问题，单体应用越来越跟不上时代发展的需要了。</p><p>你可以回想一下，无论是各种网络直播平台，还是共享单车，是不是都是很短时间内就要积累大量用户，否则就会错过风口。所以应用需要在很短的时间内快速迭代，不断调整，满足用户体验；还要在很短的时间内，具有支撑高并发请求的能力。</p><p>单体应用作为个人英雄主义的时代已经过去了。如果所有的代码都在一个工程里面，开发的时候必然存在大量冲突，上线的时候，需要开大会进行协调，一个月上线一次就很不错了。而且所有的流量都让一个进程扛，怎么也扛不住啊！</p><p>没办法，一个字：拆！拆开了，每个子模块独自变化，减少相互影响。拆开了，原来一个进程扛流量，现在多个进程一起扛。所以，微服务就是从个人英雄主义，变成集团军作战。</p><!-- [[[read_end]]] --><p>容器作为集装箱，可以保证应用在不同的环境中快速迁移，提高迭代的效率。但是如果要形成容器集团军，还需要一个集团军作战的调度平台，这就是Kubernetes。它可以灵活地将一个容器调度到任何一台机器上，并且当某个应用扛不住的时候，只要在Kubernetes上修改容器的副本数，一个应用马上就能变八个，而且都能提供服务。</p><p>然而集团军作战有个重要的问题，就是通信。这里面包含两个问题，第一个是集团军的A部队如何实时地知道B部队的位置变化，第二个是两个部队之间如何相互通信。</p><p>第一个问题位置变化，往往是通过一个称为注册中心的地方统一管理的，这个是应用自己做的。当一个应用启动的时候，将自己所在环境的IP地址和端口，注册到注册中心指挥部，这样其他的应用请求它的时候，到指挥部问一下它在哪里就好了。当某个应用发生了变化，例如一台机器挂了，容器要迁移到另一台机器，这个时候IP改变了，应用会重新注册，则其他的应用请求它的时候，还是能够从指挥部得到最新的位置。</p><p><img src="https://static001.geekbang.org/resource/image/a0/0d/a0763d50fc4e8dcec37ae25a2f6cc60d.jpeg?wh=1582*1080" alt=""></p><p>接下来是如何相互通信的问题。NAT这种模式，在多个主机的场景下，是存在很大问题的。在物理机A上的应用A看到的IP地址是容器A的，是172.17.0.2，在物理机B上的应用B看到的IP地址是容器B的，不巧也是172.17.0.2，当它们都注册到注册中心的时候，注册中心就是这个图里这样子。</p><p><img src="https://static001.geekbang.org/resource/image/e2/dd/e20596506dd34122e302a7cfc8bb85dd.jpg?wh=1920*847" alt=""></p><p>这个时候，应用A要访问应用B，当应用A从注册中心将应用B的IP地址读出来的时候，就彻底困惑了，这不是自己访问自己吗？</p><p>怎么解决这个问题呢？一种办法是不去注册容器内的IP地址，而是注册所在物理机的IP地址，端口也要是物理机上映射的端口。</p><p><img src="https://static001.geekbang.org/resource/image/8f/18/8fabf1de2a7d346856a032dbf2417b18.jpg?wh=2478*1268" alt=""></p><p>这样存在的问题是，应用是在容器里面的，它怎么知道物理机上的IP地址和端口呢？这明明是运维人员配置的，除非应用配合，读取容器平台的接口获得这个IP和端口。一方面，大部分分布式框架都是容器诞生之前就有了，它们不会适配这种场景；另一方面，让容器内的应用意识到容器外的环境，本来就是非常不好的设计。</p><p>说好的集装箱，说好的随意迁移呢？难道要让集装箱内的货物意识到自己传的信息？而且本来Tomcat都是监听8080端口的，结果到了物理机上，就不能大家都用这个端口了，否则端口就冲突了，因而就需要随机分配端口，于是在注册中心就出现了各种各样奇怪的端口。无论是注册中心，还是调用方都会觉得很奇怪，而且不是默认的端口，很多情况下也容易出错。</p><p>Kubernetes作为集团军作战管理平台，提出指导意见，说网络模型要变平，但是没说怎么实现。于是业界就涌现了大量的方案，Flannel就是其中之一。</p><p>对于IP冲突的问题，如果每一个物理机都是网段172.17.0.0/16，肯定会冲突啊，但是这个网段实在太大了，一台物理机上根本启动不了这么多的容器，所以能不能每台物理机在这个大网段里面，抠出一个小的网段，每个物理机网段都不同，自己看好自己的一亩三分地，谁也不和谁冲突。</p><p>例如物理机A是网段172.17.8.0/24，物理机B是网段172.17.9.0/24，这样两台机器上启动的容器IP肯定不一样，而且就看IP地址，我们就一下子识别出，这个容器是本机的，还是远程的，如果是远程的，也能从网段一下子就识别出它归哪台物理机管，太方便了。</p><p>接下来的问题，就是<strong>物理机A上的容器如何访问到物理机B上的容器呢？</strong></p><p>你是不是想到了熟悉的场景？虚拟机也需要跨物理机互通，往往通过Overlay的方式，容器是不是也可以这样做呢？</p><p><strong>这里我要说Flannel使用UDP实现Overlay网络的方案。</strong></p><p><img src="https://static001.geekbang.org/resource/image/07/71/07217a9ee64e1970ac04de9080505871.jpeg?wh=1920*817" alt=""></p><p>在物理机A上的容器A里面，能看到的容器的IP地址是172.17.8.2/24，里面设置了默认的路由规则default via 172.17.8.1 dev eth0。</p><p>如果容器A要访问172.17.9.2，就会发往这个默认的网关172.17.8.1。172.17.8.1就是物理机上面docker0网桥的IP地址，这台物理机上的所有容器都是连接到这个网桥的。</p><p>在物理机上面，查看路由策略，会有这样一条172.17.0.0/24 via 172.17.0.0 dev flannel.1，也就是说发往172.17.9.2的网络包会被转发到flannel.1这个网卡。</p><p>这个网卡是怎么出来的呢？在每台物理机上，都会跑一个flanneld进程，这个进程打开一个/dev/net/tun字符设备的时候，就出现了这个网卡。</p><p>你有没有想起qemu-kvm，打开这个字符设备的时候，物理机上也会出现一个网卡，所有发到这个网卡上的网络包会被qemu-kvm接收进来，变成二进制串。只不过接下来qemu-kvm会模拟一个虚拟机里面的网卡，将二进制的串变成网络包，发给虚拟机里面的网卡。但是flanneld不用这样做，所有发到flannel.1这个网卡的包都会被flanneld进程读进去，接下来flanneld要对网络包进行处理。</p><p>物理机A上的flanneld会将网络包封装在UDP包里面，然后外层加上物理机A和物理机B的IP地址，发送给物理机B上的flanneld。</p><p>为什么是UDP呢？因为不想在flanneld之间建立两两连接，而UDP没有连接的概念，任何一台机器都能发给另一台。</p><p>物理机B上的flanneld收到包之后，解开UDP的包，将里面的网络包拿出来，从物理机B的flannel.1网卡发出去。</p><p>在物理机B上，有路由规则172.17.9.0/24 dev docker0 proto kernel scope link src 172.17.9.1。</p><p>将包发给docker0，docker0将包转给容器B。通信成功。</p><p>上面的过程连通性没有问题，但是由于全部在用户态，所以性能差了一些。</p><p>跨物理机的连通性问题，在虚拟机那里有成熟的方案，就是VXLAN，那<strong>能不能Flannel也用VXLAN呢</strong>？</p><p>当然可以了。如果使用VXLAN，就不需要打开一个TUN设备了，而是要建立一个VXLAN的VTEP。如何建立呢？可以通过netlink通知内核建立一个VTEP的网卡flannel.1。在我们讲OpenvSwitch的时候提过，netlink是一种用户态和内核态通信的机制。</p><p>当网络包从物理机A上的容器A发送给物理机B上的容器B，在容器A里面通过默认路由到达物理机A上的docker0网卡，然后根据路由规则，在物理机A上，将包转发给flannel.1。这个时候flannel.1就是一个VXLAN的VTEP了，它将网络包进行封装。</p><p>内部的MAC地址这样写：源为物理机A的flannel.1的MAC地址，目标为物理机B的flannel.1的MAC地址，在外面加上VXLAN的头。</p><p>外层的IP地址这样写：源为物理机A的IP地址，目标为物理机B的IP地址，外面加上物理机的MAC地址。</p><p>这样就能通过VXLAN将包转发到另一台机器，从物理机B的flannel.1上解包，变成内部的网络包，通过物理机B上的路由转发到docker0，然后转发到容器B里面。通信成功。</p><p><img src="https://static001.geekbang.org/resource/image/01/79/01f86f6049eef051d48e2e235fa43d79.jpeg?wh=1920*835" alt=""></p><h2>小结</h2><p>好了，今天的内容就到这里，我来总结一下。</p><ul>
<li>
<p>基于NAT的容器网络模型在微服务架构下有两个问题，一个是IP重叠，一个是端口冲突，需要通过Overlay网络的机制保持跨节点的连通性。</p>
</li>
<li>
<p>Flannel是跨节点容器网络方案之一，它提供的Overlay方案主要有两种方式，一种是UDP在用户态封装，一种是VXLAN在内核态封装，而VXLAN的性能更好一些。</p>
</li>
</ul><p>最后，给你留两个问题：</p><ol>
<li>
<p>通过Flannel的网络模型可以实现容器与容器直接跨主机的互相访问，那你知道如果容器内部访问外部的服务应该怎么融合到这个网络模型中吗？</p>
</li>
<li>
<p>基于Overlay的网络毕竟做了一次网络虚拟化，有没有更加高性能的方案呢？</p>
</li>
</ol><p>我们的专栏更新到第30讲，不知你掌握得如何？每节课后我留的思考题，你都有没有认真思考，并在留言区写下答案呢？我会从<strong>已发布的文章中选出一批认真留言的同学</strong>，赠送<span class="orange">学习奖励礼券</span>和我整理的<span class="orange">独家网络协议知识图谱</span>。</p><p>欢迎你留言和我讨论。趣谈网络协议，我们下期见！</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/b6/11/e8506a04.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小宇宙</span>
  </div>
  <div class="_2_QraFYR_0">flannel的backend除了UDP和vxlan还有一种模式就是host-gw，通过主机路由的方式，将请求发送到容器外部的应用，但是有个约束就是宿主机要和其他物理机在同一个vlan或者局域网中，这种模式不需要封包和解包，因此更加高效。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-27 12:18:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/d6/60/f21b2164.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jacy</span>
  </div>
  <div class="_2_QraFYR_0">回答自己提的udp丢包的问题，希望老师帮忙看看是否正确。<br>flannel实际上是将docker出来的包再加udp封装，以支持二层网络在三层网络中传输。udp确实可能丢包，丢包是发生在flannel层上，丢包后内层的docker（如果被封装的是tcp）无ack,源docker的虚拟网卡还会按tcp协议进行重发。无非是按flannel原理多来几次。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: udp会丢的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-14 08:07:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/e9/29/629d9bb0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>天王</span>
  </div>
  <div class="_2_QraFYR_0">1 快速迭代，单体应用不能满足快速迭代和高并发的要求，所以需要拆成微服务，容器作为集装箱，可以保证应用在不同的环境快速迁移，还需要一个容器的调度平台，可以将容器快速的调度到任意服务器，这个调度平台就是k8s。2 微服务之间存在服务调用的问题，就像集团军作战，需要解决各个部队位置和部队之间通讯的问题，2.1 位置问题用注册中心，但是可能会有ip端口冲突，Flannel是为了解决这种问题的技术，给每个物理机分配一小段网络段，每个物理机的容器只使用属于自己的网络段，2.2 部队之间通讯 容器网络互相访问，Flannel使用UDP实现Overlay网络，每台物理机上都跑一个flannelid进程，打开dev&#47;net&#47;tun设备的时候，就会有这个网卡，所有发到flannel.1的网络包，都会被flannelid进程截获，会讲网络包封装进udp包，发到b的flannel.1，b的flannelid收到网络包以后，解开，由flannel.1发出去，通过dock0给到容器b。通讯比较慢，Flannel使用VXLAN技术，建立一个VXLAN的VTEP，通过netlink通知内核建立一个VTEP的网卡flannel.1，A物理机上的flannel.1就是vxlan的vtep，将网络包封装，通过vxlan将包转到另一台服务器上，b的flannel.1接收到，解包，变成内部的网络包，通过物理机上的路由转发到docker0，然后转发到容器B里面，通信成功。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-30 13:22:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/23/5b/983408b9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>悟空聊架构</span>
  </div>
  <div class="_2_QraFYR_0">“例如物理机 A 是网段 172.17.8.0&#47;24，物理机 B 是网段 172.17.9.0&#47;24”，这里应该是指物理机A给docker分配的网段吧？老师，这个容易造成误解哦……</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-03 19:50:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/a0/6e/85512d27.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>刘工的一号马由</span>
  </div>
  <div class="_2_QraFYR_0">1默认路由发到容器网络的网关<br>2underlay网络</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-07-26 09:30:31</div>
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
  <div class="_2_QraFYR_0">想问下老师，后一种使用VTEP，为什么flannel.id里面内部源地址配置的是flannel.id的mac地址，而不是使用第一种打开dev&#47;net&#47;tun时候源地址写的是容器A的源地址。<br>为什么两种情况下不一样了，谢谢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这是flannel的一个问题，会造成很多困扰</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-22 09:16:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/a1/3a/9e48ce31.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小宇</span>
  </div>
  <div class="_2_QraFYR_0">老师好，我这里有一个疑问，第28讲的VXLAN，虚拟机通信时包的结构， VTEP的设备ip为什么在VXLAN头之外，我的理解VTEP设备的ip应该和flannel的VXLAN模式一样，在VXLAN头里面。<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: vxlan的格式都是一样的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-23 09:29:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/99/af/d29273e2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>饭粒</span>
  </div>
  <div class="_2_QraFYR_0">#  tcpdump -i eth0 dst 192.168.1.7 -w  dst.pcap<br>抓了下 vxlan 类型的 flannel 容器间的包，确实是 udp 的封包，但是没有看到 vxlan 的信息，不知道这要怎么看？<br>Frame 3: 201 bytes on wire (1608 bits), 201 bytes captured (1608 bits)<br>Ethernet II, Src: fa:16:3e:08:a8:46 (fa:16:3e:08:a8:46), Dst: fa:16:3e:5a:de:91 (fa:16:3e:5a:de:91)<br>Internet Protocol Version 4, Src: 192.168.1.4, Dst: 192.168.1.7<br>User Datagram Protocol, Src Port: 54228, Dst Port: 8472<br>Data (159 bytes)</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-07 14:43:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/3a/e1/b6b311cb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>╯梦深处゛</span>
  </div>
  <div class="_2_QraFYR_0">对于 IP 冲突的问题，如果每一个物理机都是网段 172.17.0.0&#47;16，肯定会冲突啊，但是这个网段实在太大了，一台物理机上根本启动不了这么多的容器，所以能不能每台物理机在这个大网段里面，抠出一个小的网段，每个物理机网段都不同，自己看好自己的一亩三分地，谁也不和谁冲突。<br>-----------------------------------------------------------------------------------------------------<br>老师，不同节点之间不一定都是不同网段的，相反很多时候，一个K8S集群的所有节点都是在一个网段，请问在这种场景下，如果避免这种冲突的问题呢？<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 一个K8S集群的所有节点都是在一个大网段里面，但是不同的主机分了小网段。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-23 16:42:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/85/ed/905b052f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>超超</span>
  </div>
  <div class="_2_QraFYR_0">回答问题1：是不是在dockerX网卡上做NAT？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-08 23:40:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/d6/60/f21b2164.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jacy</span>
  </div>
  <div class="_2_QraFYR_0">udp丢包能接受吗</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-13 23:02:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/46/26/fa3bb8e6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>⊙▽⊙</span>
  </div>
  <div class="_2_QraFYR_0">老师我想问下，如何知道容器宿主机的地址，这样才可以在对数据包进行二次封装的时候把目的地址填写进去</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-29 18:45:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/10/78/29bd3f1e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>王子瑞Aliloke有事电联</span>
  </div>
  <div class="_2_QraFYR_0">老刘讲的太好了！！！这个专栏超值，开眼界了，而且讲的通俗易懂-虽然还有很多知识我没有懂，但开眼界了。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-14 20:55:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/d0/ee/f5c5e191.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>LYy</span>
  </div>
  <div class="_2_QraFYR_0">Flannel原理总结:<br>1. 网络打平: 主机间独立虚拟网段，不使用NAT(存在IP&#47;端口冲突问题);<br>2. 网络互通: 通过Overlay(基于UDP or VXLAN)，配合虚拟网段、路由表实现；<br>3. E2E:<br>  a) Overlay via UDP: <br>  容器A虚IP -{路由表}-&gt; 容器eth0(基于veth pair) -&gt; docker0(基于veth pair) -{路由表}-&gt; flannel.1网卡(打开&#47;dev&#47;net&#47;tun字符设备) -&gt; flanneld读取 -&gt; flanneld进行UDP封包 -&gt; Node1 eth0 -&gt; Node2 eth0 -&gt; flanneld进行UDP解包 -&gt; flannel.1网卡 -&gt; docker0 -&gt; 容器eth0 -&gt; 容器B虚IP<br>  b) VXLAN: <br>  容器A虚IP -{路由表}-&gt; 容器eth0(基于veth pair) -&gt; docker0(基于veth pair) -{路由表}-&gt; flannel.1网卡(创建VTEP网卡) -&gt; flannel.1进行VXLAN封包 -&gt; Node1 eth0 -&gt; Node2 eth0 -&gt; flannel.1进行VXLAN解包 -&gt; flannel.1网卡 -&gt; docker0 -&gt; 容器B虚IP<br>4. flanneld职责:<br>  a) Overlay via UDP:<br>      - 维护Node间网络规划信息<br>      - 创建veth pair?(存疑)<br>      - 配置路由规则<br>      - 创建flannel.1网卡(通过打开&#47;dev&#47;net&#47;tun字符设备)<br>      - 读取flannel.1网络流量，进行UDP封包&#47;解包<br>  b) VXLAN<br>      - 维护Node间网络规划信息<br>      - 创建veth pair?(存疑)<br>      - 配置路由规则(应该除了网段还有各Node flannel.1的MAC信息)<br>      - 创建flannel.1网卡(创建VTEP网卡, 基于netlink)<br>5. 封包结构:<br>  a) UDP封包: Node eth0 MAC头|Node IP头|UDP头(flanneld port)|容器IP头|Payload<br>  b) VXLAN封包: Node eth0 MAC头|Node IP头|UDP头(?port)|VXLAN头|flannel.1 MAC头|容器IP头|Payload<br><br>flannel方案简析:<br>1) Overlay via UDP基于用户态处理网络流量，性能较差;<br>2) 数据面与控制面未分析, 规模增长后可能存在管理问题。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-06 11:48:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/47/00/3202bdf0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>piboye</span>
  </div>
  <div class="_2_QraFYR_0">vxlan要通过组播来实现广播，k8s有完整的集群信息，flannel是不是可以不用组播也能实现arp的解析？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-18 23:52:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/rurTzy9obgda82kG3FTrszfzuIRQH2Mljc36u9KZLnOcJEtjY1NqdEROjpkLZia8Lu97OKhoIIicHu4xoiclHpOAA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_536b07</span>
  </div>
  <div class="_2_QraFYR_0">容器网络主流应该是k8s的网络实现，涉及到namespace 如果做pod内容器网络共享，如何通过svc访问pod，sts如何固定dns，没有结合k8s单独把网络插件拎出来讲，就很懵逼</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-11 17:08:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/rurTzy9obgda82kG3FTrszfzuIRQH2Mljc36u9KZLnOcJEtjY1NqdEROjpkLZia8Lu97OKhoIIicHu4xoiclHpOAA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_536b07</span>
  </div>
  <div class="_2_QraFYR_0">容器网络，讲的的太模糊了，容器集群注册的一般是podip，一般都会遇到跨集群访问的问题，这才是开发人员要去解决的重点</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-11 16:57:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/28/23/e8/9f445339.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>章潘</span>
  </div>
  <div class="_2_QraFYR_0">ip，mac，vlan，vxlan等等都可以理解为对网络数据或者资源的标记。从这个角度出发，容器中的IP或端口冲突问题，是因为在同一个域用了相同的标签。所以要解决冲突问题，方式就太多了。选择不同的网段是方式之一。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-13 20:13:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2a/f9/73/01eafd3c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>singularity of space time</span>
  </div>
  <div class="_2_QraFYR_0">老师您好<br>在原文的表述中“物理机 B 上的 flanneld 收到包之后，解开 UDP 的包，将里面的网络包拿出来，从物理机 B 的 flannel.1 网卡发出去”<br>这里写“发出去”感觉并不恰当，它应该是写入&#47;dev&#47;net&#47;tun设备，然后被flannel.1网卡接收到，再进入宿主机协议栈，然后经过路由发往docker0网卡，从docker0网卡中出去进入网关，再得到容器内部<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-25 12:15:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/45/83/93d389ba.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我是谁</span>
  </div>
  <div class="_2_QraFYR_0">老师好，有个问题。<br>28讲的时候，vxlan报文的外层ip和外层mac地址都是vtep设备的，但是这讲提到的flannel的vxlan方案中，外层ip和外层mac地址都是宿主机的，vtep设备的mac地址反而放在里内层，这是一个问题。另一个问题是内层mac地址不应该是虚拟机的吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-18 17:19:06</div>
  </div>
</div>
</div>
</li>
</ul>