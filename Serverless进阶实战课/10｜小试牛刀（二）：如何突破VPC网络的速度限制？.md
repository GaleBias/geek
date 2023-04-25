<audio title="10｜小试牛刀（二）：如何突破VPC网络的速度限制？" src="https://static001.geekbang.org/resource/audio/89/16/89b54439dc31a3fd06dfe6b044540416.mp3" controls="controls"></audio> 
<p>你好，我是静远。</p><p>在<a href="https://time.geekbang.org/column/article/563691">冷启动</a>这节课里，我跟你聊到了影响函数第一次执行速度的因素有很多，其中一个关键因素就是：VPC网络的建联时间。</p><p>我们知道，VPC是云厂商提供给用户的一个安全隔离的网络环境，用户可以在VPC空间中部署自己的服务，如Redis、数据库、MQ等。而我们的函数通常是创建在函数专属的VPC中的。一个生产级别的函数服务应用，少不了需要和其他的资源和中间件打交道。那么，跨VPC之间的通信效率就显得尤为重要了。</p><p>今天这节课，我就针对VPC之间的通信效率，通过案例实操的方式，为你剖析函数在跨VPC上的技术要点。</p><p>希望通过这节课，能够让你得到一些在Serverless的实战和后续优化工作的启发，能举一反三地找到更多优化方法，进一步突破函数访问速度的上限。</p><h2>跨VPC问题的演进</h2><p>针对云上网络连接的方式，相信你已经了解到不少，如专线（高速通道）、VPN网关、对等连接等方式。这些方式，有的用来解决IDC和VPC的连接，有的用来解决VPC与VPC之间的互通。</p><p>这其中，对等连接的确可以解决函数跨VPC的问题，但我们试想一下，如果每个用户都通过这种方式来解决这种问题，那么平台每次都需要搭建两端的连接，并且还可能会遇到IP网段冲突的问题。这样的做法显然不具备普适性，成本也过高。</p><!-- [[[read_end]]] --><p>为此，针对Serverless的函数计算场景，我们通常会通过弹性网卡（ENI）的方式来打通。弹性网卡是绑定在云主机或者容器实例上的，也就是说，我们是通过弹性网卡ENI来关联用户的VPC和用户的设备实例，使得设备和VPC互通的。</p><p><img src="https://static001.geekbang.org/resource/image/b5/9b/b50e677c4be24c8c456yybf536ca139b.jpg?wh=1920x735" alt="图片" title="绑定容器实例的方式"></p><p><img src="https://static001.geekbang.org/resource/image/56/3c/56547df7a2fa28yy82cb9fb6361aba3c.jpg?wh=1920x702" alt="图片" title="绑定主机的方式"></p><p>但实际上，我们暂且抛开这两者在网络速率上的差别，不论是在容器实例还是在主机上，他们都会面临同样的问题。</p><ul>
<li><strong>冷启动耗时较长：</strong>弹性网卡这种虚拟化的资源，它的创建时间是避免不了的；</li>
<li><strong>资源受限：</strong>正因为资源虚拟化，也就意味着要依赖于硬件的基本参数，受限于内核、CPU等因素，为此，几乎所有的云厂商都会限制VPC内弹性网卡的数量，也就从一定程度上导致了此方法的局限性；</li>
<li><strong>资源浪费：</strong>由于容器或者主机上要绑定弹性网卡，也就意味着多个实例就得占用多个网卡资源，造成资源浪费。</li>
</ul><p>我们可以看出，它们虽然解决了跨VPC的问题，但还是会带来性能和资源成本上的压力。那么，我们是否可以不创建这么多弹性网卡，探索一种共用的方式呢？通常我们立马能想到的方式，就是出口代理了。</p><h2>什么是代理？</h2><p>如何代理呢？我们可以了解一下它的整体流程。</p><p>简单来说，就是我们在函数计算的集群与用户的VPC之间添加Proxy Node，在Proxy Node上构建网卡（如eth1），网卡会<strong>提前在用户VPC所在的子网Subnet中创建，但租户的权限属于函数计算VPC</strong>。</p><p>有了弹性网卡，接下来就需要解决函数到代理节点的打通问题了，也就是在函数的主机和代理主机之间构建一条链路，将数据包传递过去，且保证源IP和目的IP保持不变。</p><p>我们通常采用的是隧道技术。隧道打通后，我们还得注意两件事情。其一就是在Proxy Node中多张网卡的转发处理；其二，由于函数通常以容器的方式运行，所以需要有一个容器网络转发的过程。</p><p>接下来，我将一步一步介绍它们是如何起作用的。</p><h2>动手实操</h2><p>这节课，我会通过一个打通跨VPC网络的实验案例，带你实现函数通过Proxy机器代理的方式访问用户VPC的资源的全过程。在这个案例中，我选取了百度智能云服务器BCC来进行实验。</p><p>在实验之前，我们需要准备两个VPC网段，模拟创建函数集群和用户集群，并在云上购买三台云服务器BCC，其中2台BCC在VPC1，另外1台在VPC2。下面这个图可以清晰的描述部署的逻辑：</p><p><img src="https://static001.geekbang.org/resource/image/26/3c/2607byy178c3724603ce01d9efb1943c.jpg?wh=1920x1188" alt="图片"></p><h3>隧道技术</h3><p>准备工作完成之后，我们来看隧道技术。</p><p>什么是隧道？在网络中，隧道是一种使用网络不支持的协议，构建网络之间高效和安全的链接，实现在网络中传输数据的方法。</p><p>内核中的tunnel可以通过ip tunnel help看到，当前有五个：IPIP、GRE、SIT、ISATAP和VTI。</p><p><img src="https://static001.geekbang.org/resource/image/fe/f1/fef8b20ef074bf4f373bebe2ff66b5f1.png?wh=1920x645" alt="图片"></p><p>本节课，我们选择使用GRE（Generic Routing Encapsulation），它是IPIP（IP in IP）协议的更进一步版本，从官方的描述上来看，可以取代IPIP。隧道技术本身的概念和区别不是本节的重点。不过，如果感兴趣的话，你也可以通过我文末给的延伸阅读深入了解。</p><p>接下来我们来创建隧道，使得func bcc1和proxy bcc能通过隧道打通连接。我们先在func bcc1中执行如下命令来创建GRE隧道：</p><pre><code class="language-powershell"># 加载GRE模块
modprobe ip_gre&nbsp;
lsmod | grep gre
# 创建gre模式的通道tunnel，关联上IP并赋予虚拟本端IP为10.0.0.3
ip tunnel add gre0 mode gre local 192.168.80.8 remote 192.168.4.4&nbsp; dev eth0
ip addr add 10.0.0.3 peer 10.0.0.4 dev mgre0
ip link set dev mgre0 up
</code></pre><p>在proxy bcc上执行如下命令来打通GRE隧道：</p><pre><code class="language-powershell">modprobe ip_gre
lsmod | grep gre
ip tunnel add mgre0 mode gre local 192.168.4.4 dev eth0
ip link set dev mgre0 up
ip address add 10.0.0.4/24 dev mgre0
ip neigh replace 10.0.0.3 lladdr 192.168.80.8 dev mgre0
</code></pre><p>最后，我们还可以通过ping命令来验证proxy bcc与func bcc1的GRE隧道是否打通。</p><p><img src="https://static001.geekbang.org/resource/image/bb/7c/bb561b4dd8ff0b2a014f7cccb9eea67c.jpg?wh=1920x583" alt="图片"></p><p>到这里，我们隧道的打通任务已经完成一半了，下面，我们要看proxy bcc如何与user bcc打通。</p><h3>转发技术</h3><p>我们注意到，proxy bcc和user bcc是不在一个VPC集群内的。那我们要怎么解决这个问题呢？</p><p>构建的逻辑就是在user vpc下构建一个虚拟网卡eth1，而eth1，又隶属于proxy bcc下。这个方法，和我们在上面“跨VPC的演进”中讲到的“动态创建网卡”的方法是一致的。</p><p>这里需要说明一点，如果你是一个业务函数的开发者，可能云厂商没有开放相应的特权账号，没有创建弹性网卡的能力，实操不了，那么，这里你只要了解过程即可。如果你是云平台维护工作者，可以体验一下这其中的操作方式。</p><p>弹性网卡创建并绑定好后，我们会发现，有2个网卡存在于proxy bcc中。</p><p><img src="https://static001.geekbang.org/resource/image/74/59/74108bd630497f16d9e35beea8461759.jpg?wh=1920x840" alt="图片"></p><p>我们通过ping命令，可以验证是否能够和user bcc打通。这里要注意，proxy bcc与user bcc可以互通是因为弹性网卡是基于user bcc所在的VPC的子网Subnet创建的，但租户还是属于proxy bcc。</p><p><img src="https://static001.geekbang.org/resource/image/83/60/833aeb5f01d840389252d1a424851260.jpg?wh=1920x569" alt="图片"></p><p>打通之后，我们就需要用到转发技术了，也就是将func bcc1的数据包通过proxy bcc的请求到达user bcc。</p><p>首先，我们需要通过将ip_forward设置为1，开启proxy bcc的转发能力。</p><pre><code class="language-powershell">echo 1 &gt; /proc/sys/net/ipv4/ip_forward
</code></pre><p>这里需要说明一下，出于安全考虑，Linux系统默认的是禁止数据包转发的。所谓转发，就是当主机拥有多于一块网卡时，其中一块收到数据包，根据数据包的目的IP地址将包发往本机的另一网卡，该网卡即会根据路由表继续发送数据包。</p><p>我们这里因为是eth0接收云主机的数据，通过eth1来发送出去，所以需要开启转发能力。当然，为了能让func bcc1的路由触达user bcc，我们还需要继续完善路由规则。</p><p>我们在func bcc1中通过ip route add命令添加一条到user bcc子网172.16.101.0/24的路由：</p><pre><code class="language-powershell">ip route add 172.16.101.0/24 dev mgre0
</code></pre><p>此时 func bcc1 访问 user bcc 172.16.101.4 仍然访问不通，在 proxy bcc 侧监听 eth1 流量可以发现，eth1 发往用户 VPC 的包的源 IP 仍是 func1 bcc gre0 设备的 IP 10.0.0.3，这个 IP实际只存在于我们 bcc 内部，因此用户 bcc 收到请求后，它的返回流量无法路由到这个 IP。</p><p>因此，我们这里需要在proxy bcc添加一条 iptables 规则，在 POSTROUTING 链里把从 eth1 出口的流量的源 IP SNAT 设置为 eth1 的IP 地址，即 172.16.101.4。</p><pre><code class="language-powershell">iptables -t nat -A POSTROUTING -o eth1 -j MASQUERADE
</code></pre><p>到这里，我们就可以在func bcc1中ping通user bcc了。</p><p><img src="https://static001.geekbang.org/resource/image/78/c2/78507536ae69dc708b8a68d67f3d1cc2.jpg?wh=1920x550" alt="图片"></p><h3>容器网络</h3><p>在虚机上，我们已经操作体验一遍了，但通常我们的函数是在容器实例里面运行的 的，容器在访问用户vpc服务的时候，是存在一点差别的。我们也可以验证一下，会发现，pod 发起的请求正常路由到了 proxy bcc 下的 tunnel 设备，但是没有继续转发到 eth1 中。</p><p>为什么会出现这样的情况呢？通常是因为Pod的IP和我们在proxy bcc上构建的路由规则是不一致的。至于怎么解答，我先不做回答，你可以先思考一下。我们先来看一个概念：ip-masq-agent。</p><p>ip-masq-agent 是一个用来管理 IP 伪装的扩展，即管理Kubernetes集群节点中 IP 网段的 SNAT 规则。它以 DaemonSet 形式部署在Kubernetes的集群中，在每个节点上启动一个代理程序。代理程序通过配置 iptables 规则，再将容器流量发送到集群节点的 IP 和集群 IP 范围之外的目标时，就可以选择以容器自己的 IP 地址或伪装成节点的 IP 地址作为出向流量的源地址。</p><p>看完ip-masq-agent的解说，相信你已经有答案了。解决办法，就是把 Pod IP SNAT 设置成 gre0 设备的 IP，再转给 proxy bcc，再对 proxy bcc 隐藏 Pod的IP即可。</p><p>ip-masq-agent 的配置来自于 configMap，可以通过 kubectl edit cm 命令来查看或修改配置。</p><p>到这里，我们在Pod中也能访问user bcc的服务了。</p><h3>高可用</h3><p>如果我们要上生产环境，你还需要考虑服务的高可用。在我们这节课反复提到的案例图中，我只画了一个Proxy Node，用来实验还可以，但生产环境肯定是不够的。为此，我们需要为每个VPC/Subnet建立一对主备Proxy。主备Proxy要使用HaVip进行容灾切换。</p><p>HaVip是一种可以独立创建和释放的内网IP，可以与高可用软件（例如，KeepAlived）配合使用，搭建高可用主备服务，提高业务的可用性。</p><p>另外，从Serverless的角度看，你还需要依据函数访问的流量对Proxy Node进行自动扩缩容。当流量增大时，需要自动对Proxy Node进行扩容；当流量降低时，需要自动对Proxy Node进行缩容。</p><p>我们继续在上面的架构图上进一步完善一下，<strong>增加主备Proxy Node和容器Pod之后的架构示意图</strong>如下所示。</p><p><img src="https://static001.geekbang.org/resource/image/aa/41/aaf8c2e1dec3ea5e6712a5374493ab41.jpg?wh=1920x1149" alt="图片"></p><h2>小结</h2><p>最后，我来小结一下我们今天的内容。</p><p>这节课主要围绕着优化函数在跨VPC场景的访问速度来做讲解。我们学习了跨VPC打通的演进过程以及代理的四大要素，最后，也结合一个案例的实操过程，了解了函数代理技术的实现过程。</p><p>在演进过程的回顾里，我们会发现，对等连接由于其在函数计算上的构建不具备普适性，只有在极少数大客户的场景下会使用。而弹性的网卡实时创建，虽然具备一定的通用能力，但由于其高延迟和对资源的浪费，我们也不太推荐使用。</p><p>这两个技术，给我们来带来的思考是，<strong>如果我们能提前准备好访问的通路且可复用，是不是就可以解决这样的问题了？是的。为此，我们提出了函数代理技术。</strong></p><p>其实，代理技术需要解决的技术卡点还不少，主要包括我们提到的隧道通路、网卡转发、容器网络转发技术，以及最后在生产环境下，还需要考虑到的高可用的问题。我们可以通过高可用的虚拟IP和主备Proxy集群解决。同时，从弹性的角度，我也建议你引入扩缩容的能力，这个能力，你也可以贯穿在整个Serverless的学习生涯中，可以说，它就是Serverless的精髓所在。</p><p>最后，函数计算由于其胶水语言的特性，跨介质跨网络的访问是不可忽视的一件事情。虽然我们已经在不断的演进解决，但函数代理技术也有其缺点的，比如资源一定程度上的增加。因此，希望通过这节课的内容和思考的过程，能让你打开眼界，提出更多的优化手段，让Serverless的技术积淀越来越深厚。</p><h2>思考题</h2><p>好了，这节课到这里也就结束了，最后我给你留了一个思考题。</p><p>如果我们在隧道打通那里，采用IPIP隧道协议，会有什么问题？</p><p>欢迎在留言区写下你的思考和答案，我们一起交流讨论。感谢你的阅读，也欢迎你把这节课分享给更多的朋友一起交流学习。</p><p></p><h2>延伸阅读</h2><ul>
<li>你可以通过<a href="https://docs.aws.amazon.com/zh_cn/vpc/latest/peering/what-is-vpc-peering.html">VPC的对等链接</a>，继续深入了解一下对等链接的概念。</li>
<li>这一篇<a href="https://www.cloudflare.com/zh-cn/learning/network-layer/what-is-tunneling/">隧道技术</a>，可以帮助你进一步了解隧道的知识要点。</li>
<li>腾讯云的这一篇<a href="https://cloud.tencent.com/developer/article/1461707">VPC网络优化</a>的文章，也是值得我们一看的。</li>
<li>文中提到的ip-masq-agent，感兴趣的话，你可以继续看这一篇<a href="http://www.coderdocument.com/docs/kubernetes/v1.14/tasks/administer_cluster/ip_masquerade_agent_user_guide.html">ip-mast-agent指南</a>，写得也是非常详细的。</li>
</ul>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/6b/e2/84e33802.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>马洪超</span>
  </div>
  <div class="_2_QraFYR_0">func bcc和proxy bcc本来就是通的，为啥要加一层隧道呢，直接把proxy bcc做网关，不就直接可以访问user bcc了吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是互通的，内核转发，目的是封装原始ip 包送达到proxy bcc </p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-28 17:17:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Wang Yifei</span>
  </div>
  <div class="_2_QraFYR_0">“此时 func bcc1 访问 user bcc 172.16.101.4 仍然访问不通，在 proxy bcc 侧监听 eth1 流量可以发现，eth1 发往用户 VPC 的包的源 IP 仍是 func1 bcc gre0 设备的 IP 10.0.0.3”<br><br>这里有个问题，proxy bbc的gre0设备是不是会把gre封装的包头去掉后，继续根据payload进行路由转发？ 如果是这样的话，payload的源IP为什么还是10.0.0.3，而不是192.168.80.8呢？<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是192.168.80.8，10.0.0.3是隧道的ip </p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-04 13:17:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Wang Yifei</span>
  </div>
  <div class="_2_QraFYR_0">没有找到Kubernetes中相关CM的配置<br>但我理解，是否也可以这样实现：在func bcc1中添加一条iptables规则，把发送给172.16.101.0&#47;24流量的Source IP修改为10.0.0.3。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 跨vpc 是不可以的哈</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-04 11:39:34</div>
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
  <div class="_2_QraFYR_0">网卡会提前在用户 VPC 所在的子网 Subnet 中创建，但租户的权限属于函数计算 VPC<br>==========<br>这个是怎么实现的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 首先需要用户授权，然后平台在创建的时候通过特权账号从网卡后台服务侧申请创建，简单来说，就是用户，函数计算平台，网卡后台服务三者针对弹性网卡创建的一个关系。这个关系基础上的访问属于云厂商的特性</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-20 19:08:26</div>
  </div>
</div>
</div>
</li>
</ul>