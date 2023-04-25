<audio title="35 _ 解读Kubernetes三层网络方案" src="https://static001.geekbang.org/resource/audio/9c/4e/9c2355af02dec4fdf8b506b527783f4e.mp3" controls="controls"></audio> 
<p>你好，我是张磊。今天我和你分享的主题是：解读Kubernetes三层网络方案。</p><p>在上一篇文章中，我以网桥类型的Flannel插件为例，为你讲解了Kubernetes里容器网络和CNI插件的主要工作原理。不过，除了这种模式之外，还有一种纯三层（Pure Layer 3）网络方案非常值得你注意。其中的典型例子，莫过于Flannel的host-gw模式和Calico项目了。</p><p><span class="orange">我们先来看一下Flannel的host-gw模式。</span></p><p>它的工作原理非常简单，我用一张图就可以和你说清楚。为了方便叙述，接下来我会称这张图为“host-gw示意图”。</p><p><img src="https://static001.geekbang.org/resource/image/3d/25/3d8b08411eeb49be2658eb4352206d25.png?wh=2880*1528" alt="" title="图1 Flannel host-gw示意图"></p><p>假设现在，Node 1上的Infra-container-1，要访问Node 2上的Infra-container-2。</p><p>当你设置Flannel使用host-gw模式之后，flanneld会在宿主机上创建这样一条规则，以Node 1为例：</p><pre><code>$ ip route
...
10.244.1.0/24 via 10.168.0.3 dev eth0
</code></pre><p>这条路由规则的含义是：目的IP地址属于10.244.1.0/24网段的IP包，应该经过本机的eth0设备发出去（即：dev eth0）；并且，它下一跳地址（next-hop）是10.168.0.3（即：via 10.168.0.3）。</p><!-- [[[read_end]]] --><p>所谓下一跳地址就是：如果IP包从主机A发到主机B，需要经过路由设备X的中转。那么X的IP地址就应该配置为主机A的下一跳地址。</p><p>而从host-gw示意图中我们可以看到，这个下一跳地址对应的，正是我们的目的宿主机Node 2。</p><p>一旦配置了下一跳地址，那么接下来，当IP包从网络层进入链路层封装成帧的时候，eth0设备就会使用下一跳地址对应的MAC地址，作为该数据帧的目的MAC地址。显然，这个MAC地址，正是Node 2的MAC地址。</p><p>这样，这个数据帧就会从Node 1通过宿主机的二层网络顺利到达Node 2上。</p><p>而Node 2的内核网络栈从二层数据帧里拿到IP包后，会“看到”这个IP包的目的IP地址是10.244.1.3，即Infra-container-2的IP地址。这时候，根据Node 2上的路由表，该目的地址会匹配到第二条路由规则（也就是10.244.1.0对应的路由规则），从而进入cni0网桥，进而进入到Infra-container-2当中。</p><p>可以看到，<strong>host-gw模式的工作原理，其实就是将每个Flannel子网（Flannel Subnet，比如：10.244.1.0/24）的“下一跳”，设置成了该子网对应的宿主机的IP地址。</strong></p><p>也就是说，这台“主机”（Host）会充当这条容器通信路径里的“网关”（Gateway）。这也正是“host-gw”的含义。</p><p>当然，Flannel子网和主机的信息，都是保存在Etcd当中的。flanneld只需要WACTH这些数据的变化，然后实时更新路由表即可。</p><blockquote>
<p>注意：在Kubernetes v1.7之后，类似Flannel、Calico的CNI网络插件都是可以直接连接Kubernetes的APIServer来访问Etcd的，无需额外部署Etcd给它们使用。</p>
</blockquote><p>而在这种模式下，容器通信的过程就免除了额外的封包和解包带来的性能损耗。根据实际的测试，host-gw的性能损失大约在10%左右，而其他所有基于VXLAN“隧道”机制的网络方案，性能损失都在20%~30%左右。</p><p>当然，通过上面的叙述，你也应该看到，host-gw模式能够正常工作的核心，就在于IP包在封装成帧发送出去的时候，会使用路由表里的“下一跳”来设置目的MAC地址。这样，它就会经过二层网络到达目的宿主机。</p><p><strong>所以说，Flannel host-gw模式必须要求集群宿主机之间是二层连通的。</strong></p><p>需要注意的是，宿主机之间二层不连通的情况也是广泛存在的。比如，宿主机分布在了不同的子网（VLAN）里。但是，在一个Kubernetes集群里，宿主机之间必须可以通过IP地址进行通信，也就是说至少是三层可达的。否则的话，你的集群将不满足上一篇文章中提到的宿主机之间IP互通的假设（Kubernetes网络模型）。当然，“三层可达”也可以通过为几个子网设置三层转发来实现。</p><p><span class="orange">而在容器生态中，要说到像Flannel host-gw这样的三层网络方案，我们就不得不提到这个领域里的“龙头老大”Calico项目了。</span></p><p>实际上，Calico项目提供的网络解决方案，与Flannel的host-gw模式，几乎是完全一样的。也就是说，Calico也会在每台宿主机上，添加一个格式如下所示的路由规则：</p><pre><code>&lt;目的容器IP地址段&gt; via &lt;网关的IP地址&gt; dev eth0
</code></pre><p>其中，网关的IP地址，正是目的容器所在宿主机的IP地址。</p><p>而正如前所述，这个三层网络方案得以正常工作的核心，是为每个容器的IP地址，找到它所对应的、“下一跳”的<strong>网关</strong>。</p><p>不过，<strong>不同于Flannel通过Etcd和宿主机上的flanneld来维护路由信息的做法，Calico项目使用了一个“重型武器”来自动地在整个集群中分发路由信息。</strong></p><p>这个“重型武器”，就是BGP。</p><p><strong>BGP的全称是Border Gateway Protocol，即：边界网关协议</strong>。它是一个Linux内核原生就支持的、专门用在大规模数据中心里维护不同的“自治系统”之间路由信息的、无中心的路由协议。</p><p>这个概念可能听起来有点儿“吓人”，但实际上，我可以用一个非常简单的例子来为你讲清楚。</p><p><img src="https://static001.geekbang.org/resource/image/2e/9b/2e4b3bee1d924f4ae25e2c1fd115379b.jpg?wh=1738*893" alt="" title="图2 自治系统"></p><p></p><p>在这个图中，我们有两个自治系统（Autonomous System，简称为AS）：AS 1和AS 2。而所谓的一个自治系统，指的是一个组织管辖下的所有IP网络和路由器的全体。你可以把它想象成一个小公司里的所有主机和路由器。在正常情况下，自治系统之间不会有任何“来往”。</p><p>但是，如果这样两个自治系统里的主机，要通过IP地址直接进行通信，我们就必须使用路由器把这两个自治系统连接起来。</p><p>比如，AS 1里面的主机10.10.0.2，要访问AS 2里面的主机172.17.0.3的话。它发出的IP包，就会先到达自治系统AS 1上的路由器 Router 1。</p><p>而在此时，Router 1的路由表里，有这样一条规则，即：目的地址是172.17.0.2包，应该经过Router 1的C接口，发往网关Router 2（即：自治系统AS 2上的路由器）。</p><p>所以IP包就会到达Router 2上，然后经过Router 2的路由表，从B接口出来到达目的主机172.17.0.3。</p><p>但是反过来，如果主机172.17.0.3要访问10.10.0.2，那么这个IP包，在到达Router 2之后，就不知道该去哪儿了。因为在Router 2的路由表里，并没有关于AS 1自治系统的任何路由规则。</p><p>所以这时候，网络管理员就应该给Router 2也添加一条路由规则，比如：目标地址是10.10.0.2的IP包，应该经过Router 2的C接口，发往网关Router 1。</p><p>像上面这样负责把自治系统连接在一起的路由器，我们就把它形象地称为：<strong>边界网关</strong>。它跟普通路由器的不同之处在于，它的路由表里拥有其他自治系统里的主机路由信息。</p><p>上面的这部分原理，相信你理解起来应该很容易。毕竟，路由器这个设备本身的主要作用，就是连通不同的网络。</p><p>但是，你可以想象一下，假设我们现在的网络拓扑结构非常复杂，每个自治系统都有成千上万个主机、无数个路由器，甚至是由多个公司、多个网络提供商、多个自治系统组成的复合自治系统呢？</p><p>这时候，如果还要依靠人工来对边界网关的路由表进行配置和维护，那是绝对不现实的。</p><p>而这种情况下，BGP大显身手的时刻就到了。</p><p>在使用了BGP之后，你可以认为，在每个边界网关上都会运行着一个小程序，它们会将各自的路由表信息，通过TCP传输给其他的边界网关。而其他边界网关上的这个小程序，则会对收到的这些数据进行分析，然后将需要的信息添加到自己的路由表里。</p><p>这样，图2中Router 2的路由表里，就会自动出现10.10.0.2和10.10.0.3对应的路由规则了。</p><p>所以说，<strong>所谓BGP，就是在大规模网络中实现节点路由信息共享的一种协议。</strong></p><p>而BGP的这个能力，正好可以取代Flannel维护主机上路由表的功能。而且，BGP这种原生就是为大规模网络环境而实现的协议，其可靠性和可扩展性，远非Flannel自己的方案可比。</p><blockquote>
<p>需要注意的是，BGP协议实际上是最复杂的一种路由协议。我在这里的讲述和所举的例子，仅是为了能够帮助你建立对BGP的感性认识，并不代表BGP真正的实现方式。</p>
</blockquote><p>接下来，我们还是回到Calico项目上来。</p><p>在了解了BGP之后，Calico项目的架构就非常容易理解了。它由三个部分组成：</p><ol>
<li>
<p>Calico的CNI插件。这是Calico与Kubernetes对接的部分。我已经在上一篇文章中，和你详细分享了CNI插件的工作原理，这里就不再赘述了。</p>
</li>
<li>
<p>Felix。它是一个DaemonSet，负责在宿主机上插入路由规则（即：写入Linux内核的FIB转发信息库），以及维护Calico所需的网络设备等工作。</p>
</li>
<li>
<p>BIRD。它就是BGP的客户端，专门负责在集群里分发路由规则信息。</p>
</li>
</ol><p><strong>除了对路由信息的维护方式之外，Calico项目与Flannel的host-gw模式的另一个不同之处，就是它不会在宿主机上创建任何网桥设备</strong>。这时候，Calico的工作方式，可以用一幅示意图来描述，如下所示（在接下来的讲述中，我会统一用“BGP示意图”来指代它）：</p><p><img src="https://static001.geekbang.org/resource/image/8d/1b/8db6dee96c4242738ae2878e58cecd1b.jpg?wh=1560*836" alt="" title="图3 Calico工作原理"></p><p>其中的绿色实线标出的路径，就是一个IP包从Node 1上的Container 1，到达Node 2上的Container 4的完整路径。</p><p>可以看到，Calico的CNI插件会为每个容器设置一个Veth Pair设备，然后把其中的一端放置在宿主机上（它的名字以cali前缀开头）。</p><p>此外，由于Calico没有使用CNI的网桥模式，Calico的CNI插件还需要在宿主机上为每个容器的Veth Pair设备配置一条路由规则，用于接收传入的IP包。比如，宿主机Node 2上的Container 4对应的路由规则，如下所示：</p><pre><code>10.233.2.3 dev cali5863f3 scope link
</code></pre><p>即：发往10.233.2.3的IP包，应该进入cali5863f3设备。</p><blockquote>
<p>基于上述原因，Calico项目在宿主机上设置的路由规则，肯定要比Flannel项目多得多。不过，Flannel host-gw模式使用CNI网桥的主要原因，其实是为了跟VXLAN模式保持一致。否则的话，Flannel就需要维护两套CNI插件了。</p>
</blockquote><p>有了这样的Veth Pair设备之后，容器发出的IP包就会经过Veth Pair设备出现在宿主机上。然后，宿主机网络栈就会根据路由规则的下一跳IP地址，把它们转发给正确的网关。接下来的流程就跟Flannel host-gw模式完全一致了。</p><p>其中，这里最核心的“下一跳”路由规则，就是由Calico的Felix进程负责维护的。这些路由规则信息，则是通过BGP Client也就是BIRD组件，使用BGP协议传输而来的。</p><p>而这些通过BGP协议传输的消息，你可以简单地理解为如下格式：</p><pre><code>[BGP消息]
我是宿主机192.168.1.3
10.233.2.0/24网段的容器都在我这里
这些容器的下一跳地址是我
</code></pre><p>不难发现，Calico项目实际上将集群里的所有节点，都当作是边界路由器来处理，它们一起组成了一个全连通的网络，互相之间通过BGP协议交换路由规则。这些节点，我们称为BGP Peer。</p><p>需要注意的是，<strong>Calico维护的网络在默认配置下，是一个被称为“Node-to-Node Mesh”的模式</strong>。这时候，每台宿主机上的BGP Client都需要跟其他所有节点的BGP Client进行通信以便交换路由信息。但是，随着节点数量N的增加，这些连接的数量就会以N²的规模快速增长，从而给集群本身的网络带来巨大的压力。</p><p>所以，Node-to-Node Mesh模式一般推荐用在少于100个节点的集群里。而在更大规模的集群中，你需要用到的是一个叫作Route Reflector的模式。</p><p>在这种模式下，Calico会指定一个或者几个专门的节点，来负责跟所有节点建立BGP连接从而学习到全局的路由规则。而其他节点，只需要跟这几个专门的节点交换路由信息，就可以获得整个集群的路由规则信息了。</p><p>这些专门的节点，就是所谓的Route Reflector节点，它们实际上扮演了“中间代理”的角色，从而把BGP连接的规模控制在N的数量级上。</p><p>此外，我在前面提到过，Flannel host-gw模式最主要的限制，就是要求集群宿主机之间是二层连通的。而这个限制对于Calico来说，也同样存在。</p><p>举个例子，假如我们有两台处于不同子网的宿主机Node 1和Node 2，对应的IP地址分别是192.168.1.2和192.168.2.2。需要注意的是，这两台机器通过路由器实现了三层转发，所以这两个IP地址之间是可以相互通信的。</p><p>而我们现在的需求，还是Container 1要访问Container 4。</p><p>按照我们前面的讲述，Calico会尝试在Node 1上添加如下所示的一条路由规则：</p><pre><code>10.233.2.0/16 via 192.168.2.2 eth0
</code></pre><p>但是，这时候问题就来了。</p><p>上面这条规则里的下一跳地址是192.168.2.2，可是它对应的Node 2跟Node 1却根本不在一个子网里，没办法通过二层网络把IP包发送到下一跳地址。</p><p><strong>在这种情况下，你就需要为Calico打开IPIP模式。</strong></p><p>我把这个模式下容器通信的原理，总结成了一张图片，如下所示（接下来我会称之为：IPIP示意图）：</p><p><img src="https://static001.geekbang.org/resource/image/4d/c9/4dd9ad6415caf68da81562d9542049c9.jpg?wh=1696*1056" alt="" title="图4 Calico IPIP模式工作原理"></p><p>在Calico的IPIP模式下，Felix进程在Node 1上添加的路由规则，会稍微不同，如下所示：</p><pre><code>10.233.2.0/24 via 192.168.2.2 tunl0
</code></pre><p>可以看到，尽管这条规则的下一跳地址仍然是Node 2的IP地址，但这一次，要负责将IP包发出去的设备，变成了tunl0。注意，是T-U-N-L-0，而不是Flannel UDP模式使用的T-U-N-0（tun0），这两种设备的功能是完全不一样的。</p><p>Calico使用的这个tunl0设备，是一个IP隧道（IP tunnel）设备。</p><p>在上面的例子中，IP包进入IP隧道设备之后，就会被Linux内核的IPIP驱动接管。IPIP驱动会将这个IP包直接封装在一个宿主机网络的IP包中，如下所示：</p><p><img src="https://static001.geekbang.org/resource/image/fc/90/fc2b4173782b7a993f4a43a2cb966f90.jpg?wh=2248*1034" alt=""></p><center><span class="reference">图5 IPIP封包方式</span></center><p>其中，经过封装后的新的IP包的目的地址（图5中的Outer IP Header部分），正是原IP包的下一跳地址，即Node 2的IP地址：192.168.2.2。</p><p>而原IP包本身，则会被直接封装成新IP包的Payload。</p><p>这样，原先从容器到Node 2的IP包，就被伪装成了一个从Node 1到Node 2的IP包。</p><p>由于宿主机之间已经使用路由器配置了三层转发，也就是设置了宿主机之间的“下一跳”。所以这个IP包在离开Node 1之后，就可以经过路由器，最终“跳”到Node 2上。</p><p>这时，Node 2的网络内核栈会使用IPIP驱动进行解包，从而拿到原始的IP包。然后，原始IP包就会经过路由规则和Veth Pair设备到达目的容器内部。</p><p>以上，就是Calico项目主要的工作原理了。</p><p>不难看到，当Calico使用IPIP模式的时候，集群的网络性能会因为额外的封包和解包工作而下降。在实际测试中，Calico IPIP模式与Flannel VXLAN模式的性能大致相当。所以，在实际使用时，如非硬性需求，我建议你将所有宿主机节点放在一个子网里，避免使用IPIP。</p><p>不过，通过上面对Calico工作原理的讲述，你应该能发现这样一个事实：</p><p>如果Calico项目能够让宿主机之间的路由设备（也就是网关），也通过BGP协议“学习”到Calico网络里的路由规则，那么从容器发出的IP包，不就可以通过这些设备路由到目的宿主机了么？</p><p>比如，只要在上面“IPIP示意图”中的Node 1上，添加如下所示的一条路由规则：</p><pre><code>10.233.2.0/24 via 192.168.1.1 eth0
</code></pre><p>然后，在Router 1上（192.168.1.1），添加如下所示的一条路由规则：</p><pre><code>10.233.2.0/24 via 192.168.2.1 eth0
</code></pre><p>那么Container 1发出的IP包，就可以通过两次“下一跳”，到达Router 2（192.168.2.1）了。以此类推，我们可以继续在Router 2上添加“下一条”路由，最终把IP包转发到Node 2上。</p><p>遗憾的是，上述流程虽然简单明了，但是在Kubernetes被广泛使用的公有云场景里，却完全不可行。</p><p>这里的原因在于：公有云环境下，宿主机之间的网关，肯定不会允许用户进行干预和设置。</p><blockquote>
<p>当然，在大多数公有云环境下，宿主机（公有云提供的虚拟机）本身往往就是二层连通的，所以这个需求也不强烈。</p>
</blockquote><p>不过，在私有部署的环境下，宿主机属于不同子网（VLAN）反而是更加常见的部署状态。这时候，想办法将宿主机网关也加入到BGP Mesh里从而避免使用IPIP，就成了一个非常迫切的需求。</p><p><span class="orange">而在Calico项目中，它已经为你提供了两种将宿主机网关设置成BGP Peer的解决方案。</span></p><p><strong>第一种方案</strong>，就是所有宿主机都跟宿主机网关建立BGP Peer关系。</p><p>这种方案下，Node 1和Node 2就需要主动跟宿主机网关Router 1和Router 2建立BGP连接。从而将类似于10.233.2.0/24这样的路由信息同步到网关上去。</p><p>需要注意的是，这种方式下，Calico要求宿主机网关必须支持一种叫作Dynamic Neighbors的BGP配置方式。这是因为，在常规的路由器BGP配置里，运维人员必须明确给出所有BGP Peer的IP地址。考虑到Kubernetes集群可能会有成百上千个宿主机，而且还会动态地添加和删除节点，这时候再手动管理路由器的BGP配置就非常麻烦了。而Dynamic Neighbors则允许你给路由器配置一个网段，然后路由器就会自动跟该网段里的主机建立起BGP Peer关系。</p><p>不过，相比之下，我更愿意推荐<strong>第二种方案</strong>。</p><p>这种方案，是使用一个或多个独立组件负责搜集整个集群里的所有路由信息，然后通过BGP协议同步给网关。而我们前面提到，在大规模集群中，Calico本身就推荐使用Route Reflector节点的方式进行组网。所以，这里负责跟宿主机网关进行沟通的独立组件，直接由Route Reflector兼任即可。</p><p>更重要的是，这种情况下网关的BGP Peer个数是有限并且固定的。所以我们就可以直接把这些独立组件配置成路由器的BGP Peer，而无需Dynamic Neighbors的支持。</p><p>当然，这些独立组件的工作原理也很简单：它们只需要WATCH Etcd里的宿主机和对应网段的变化信息，然后把这些信息通过BGP协议分发给网关即可。</p><h2>总结</h2><p>在本篇文章中，我为你详细讲述了Fannel host-gw模式和Calico这两种纯三层网络方案的工作原理。</p><p>需要注意的是，在大规模集群里，三层网络方案在宿主机上的路由规则可能会非常多，这会导致错误排查变得困难。此外，在系统故障的时候，路由规则出现重叠冲突的概率也会变大。</p><p>基于上述原因，如果是在公有云上，由于宿主机网络本身比较“直白”，我一般会推荐更加简单的Flannel host-gw模式。</p><p>但不难看到，在私有部署环境里，Calico项目才能够覆盖更多的场景，并为你提供更加可靠的组网方案和架构思路。</p><h2>思考题</h2><p>你能否能总结一下三层网络方案和“隧道模式”的异同，以及各自的优缺点？</p><p>感谢你的收听，欢迎你给我留言，也欢迎分享给更多的朋友一起阅读。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4c/49/d21c134f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>swordholder</span>
  </div>
  <div class="_2_QraFYR_0">写了两篇文档：<br>Docker单机网络模型动手实验 https:&#47;&#47;github.com&#47;mz1999&#47;blog&#47;blob&#47;master&#47;docs&#47;docker-network-bridge.md<br>Docker跨主机Overlay网络动手实验 https:&#47;&#47;github.com&#47;mz1999&#47;blog&#47;blob&#47;master&#47;docs&#47;docker-overlay-networks.md<br>通过动手实验加深理解容器网络。分享出来希望对小伙伴有所帮助。<br><br>看来学完了这篇，可以再写一个Docker跨主机路由方案动手实验</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-12 09:05:59</div>
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
  <div class="_2_QraFYR_0">三层和隧道的异同：<br>相同之处是都实现了跨主机容器的三层互通，而且都是通过对目的 MAC 地址的操作来实现的；不同之处是三层通过配置下一条主机的路由规则来实现互通，隧道则是通过通过在 IP 包外再封装一层 MAC 包头来实现。<br>三层的优点：少了封包和解包的过程，性能肯定是更高的。<br>三层的缺点：需要自己想办法维护路由规则。<br>隧道的优点：简单，原因是大部分工作都是由 Linux 内核的模块实现了，应用层面工作量较少。<br>隧道的缺点：主要的问题就是性能低。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-12 18:41:20</div>
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
  <div class="_2_QraFYR_0">Kubernetes通过一个叫做CNI的接口，维护了一个单独的网桥来代替docker0。这个网桥的名字就叫作：CNI网桥，它在宿主机上的设备名称默认是：cni0。<br><br>容器“跨主通信”的三种主流实现方法：UDP、host-gw、VXLAN。 之前介绍了UDP和VXLAN，它们都属于隧道模式，需要封装和解封装。接下来介绍一种纯三层网络方案，host-gw模式和Calico项目<br><br>Host-gw模式通过在宿主机上添加一个路由规则： <br><br>    &lt;目的容器IP地址段&gt; via &lt;网关的IP地址&gt; dev eth0<br><br>IP包在封装成帧发出去的时候，会使用路由表里的“下一跳”来设置目的MAC地址。这样，它就会通过二层网络到达目的宿主机。<br>这个三层网络方案得以正常工作的核心，是为每个容器的IP地址，找到它所对应的，“下一跳”的网关。所以说，Flannel host-gw模式必须要求集群宿主机之间是二层连通的，如果宿主机分布在了不同的VLAN里（三层连通），由于需要经过的中间的路由器不一定有相关的路由配置（出于安全考虑，公有云环境下，宿主机之间的网关，肯定不会允许用户进行干预和设置），部分节点就无法找到容器IP的“下一条”网关了，host-gw就无法工作了。<br><br>Calico项目提供的网络解决方案，与Flannel的host-gw模式几乎一样，也会在宿主机上添加一个路由规则：<br><br>    &lt;目的容器IP地址段&gt; via &lt;网关的IP地址&gt; dev eth0<br><br>其中，网关的IP地址，正是目的容器所在宿主机的IP地址，而正如前面所述，这个三层网络方案得以正常工作的核心，是为每个容器的IP地址，找到它所对应的，“下一跳”的网关。区别是如何维护路由信息：<br>Host-gw :  Flannel通过Etcd和宿主机上的flanneld来维护路由信息<br>Calico: 通过BGP（边界网关协议）来实现路由自治，所谓BGP，就是在大规模网络中实现节点路由信息共享的一种协议。<br><br>隧道技术（需要封装包和解包，因为需要伪装成宿主机的IP包，需要三层链通）：Flannel UDP &#47; VXLAN  &#47; Calico IPIP<br>三层网络（不需要封包和解封包，需要二层链通）：Flannel host-gw &#47; Calico 普通模式<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-25 11:27:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/7c/e7/47429b6c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>米开朗基杨</span>
  </div>
  <div class="_2_QraFYR_0">写了篇关于calico开启反射路由模式的文章，希望对大家有所帮助：https:&#47;&#47;www.yangcs.net&#47;posts&#47;calico-rr&#47;</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-30 20:11:59</div>
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
  <div class="_2_QraFYR_0">所以三层可达，二层不可达的k8s集群不能使用Flannel host-gw模式，只能使用其他模式？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-29 21:49:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/94/47/75875257.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>虎虎❤️</span>
  </div>
  <div class="_2_QraFYR_0">三层网络优点： 不需要封包、拆包，传输效率会更高；可以设置复杂的网络策略。<br>隧道模式优点： 不需要在主机间的网关中维护容器路由信息，只需要主机有三层网络的连通性即可。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-12 10:37:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/6c/47/57b6c4e8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>帅</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，问个小白问题，怎么判断集群里的宿主机是否二层互通，或者说怎么实现二层互通？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-21 16:22:13</div>
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
  <div class="_2_QraFYR_0">架设k8s集群后，对操作系统及网络知识有了一个很好的复习，感谢大神</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-12 18:06:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/ef/2a/e9c5c163.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小江哥哥</span>
  </div>
  <div class="_2_QraFYR_0">对于二层联通的网络：<br>可以选择flannel的host-gw或者calico（非ipip模式），将宿主机当作网关，进行直接跳转，区别为calico自己封装了一个组件（BIRD）存储并维护路由关系，flannel维护并借助K8S的etcd存储路由关系。<br><br>对于多个vlan的网络：<br>flannel的vxlan和calico的ipip模式都可以选择，它们都通过内核模块进行了一次封包和解包过程，有内核态到用户态的切换，效率损耗较大（20%-30%）。不同点在于，calico拥有‘重型武器’BGP，可以通过它与宿主机的网关交换，共同维护路由关系，实现伪二层直连，方便自定义网络架构，定制程度比flannel高。<br><br>其他区别：<br>1. flannel的所有模式都使用了cni的网桥模式，不用在宿主机上维护这么多设备对的映射关系，calico则相反。<br>2. 所有网络IP规划分配，路由规则存储维护calico都大包大揽，通过BGP，等组件维护，flannel的路由存储和维护需要依赖K8s的etcd，且会与apiserver交互。<br>3. 在大规模集群中。对于路由规则的跟踪维护，calico有自己的中心化方案（Route Reflector模式），避免了路由信息指数级增长。<br>4. 在非二层直连网络中，calico支持通过与宿主机网关的互动（BGP协议），组成三层网络，避免了使用ipip封包解包，提高网络性能，这点上比flannel定制化程度更高。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-23 14:40:40</div>
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
  <div class="_2_QraFYR_0">三层网络方案简单易懂，开销小，缺点是需要物理互联，连接节点有限；<br>隧道协议更通用，可以打通不同网段，但是开销大，协议复杂，难排错；</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-23 10:29:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/55/86/ca7c94ce.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>羁绊12221</span>
  </div>
  <div class="_2_QraFYR_0">老师好，flannel UDP 隧道封装的是IP包，vxlan封装的是 二层帧，Calico IPIP模式封装的也是IP包。。感觉上使用隧道通信封装IP报文就够了吧？封装二层帧有什么考虑吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 有些网络方案或者硬件，要求必须在二层做一些功能</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-16 15:16:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/dd/6e/f3cfebc5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>峰哥</span>
  </div>
  <div class="_2_QraFYR_0">老师好，有个1.4的K8S集群，sprincloud的微服务，怎么升级到1.10，是生产环境，有什么好的办法，谢谢<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 版本差太远了，升级不了。。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-11 17:30:27</div>
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
  <div class="_2_QraFYR_0">有个疑问，为什么叫 “三层”网络方案呢？<br><br><br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-25 11:27:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/32/c2/8c1a2b00.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>头发乱了</span>
  </div>
  <div class="_2_QraFYR_0">听老师的讲解真是太过瘾了！！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-23 11:10:44</div>
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
  <div class="_2_QraFYR_0">有一个疑问 ，Calico IPIP模式下。IPIP封包前后，IP Header里面的目的IP都是“192.168.2.2”。那为什么封包前不能从Node1发送到Node2 ？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-20 15:33:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/ZwGweFhVUTfOrrYRk6Dic1IBxFyj2ZgsI1UXQeic2B5uJFdjicsIicnKrJts9v7nGUTCQlSKNUpmvYULq5KjqWjU4g/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>freeman</span>
  </div>
  <div class="_2_QraFYR_0">老师，能介绍一下Cilium吗</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-13 18:09:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/a3/ce/12f9d2ae.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>朱升平</span>
  </div>
  <div class="_2_QraFYR_0">网络是短板，得多学习学习！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-12 09:28:01</div>
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
  <div class="_2_QraFYR_0">隧道技术（需要封装包和解包，因为需要伪装成宿主机的IP包，需要三层链通）：Flannel UDP &#47; VXLAN  &#47; Calico IPIP<br>三层网络（不需要封包和解封包，需要二层链通）：Flannel host-gw &#47; Calico </div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-20 18:41:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/a5/98/a65ff31a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>djfhchdh</span>
  </div>
  <div class="_2_QraFYR_0">三层网络方案和“隧道模式”的异同，以及各自的优缺点：三层网络方案要求宿主机二层连通，隧道模式不要求宿主机二层连通；三层网路方案没有封包和解包，性能上比隧道模式要好。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-21 17:48:34</div>
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
  <div class="_2_QraFYR_0">碰到一个奇怪的问题，集群中一台master节点重启后，发现docker images 镜像都被清空了一样，然后我重新pull过来，过一会又没有了，这个是哪里问题呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-13 18:11:50</div>
  </div>
</div>
</div>
</li>
</ul>