<audio title="34 _ Kubernetes网络模型与CNI网络插件" src="https://static001.geekbang.org/resource/audio/d3/33/d38ef6da2fbe39a55edc25ca41814233.mp3" controls="controls"></audio> 
<p>你好，我是张磊。今天我和你分享的主题是：Kubernetes网络模型与CNI网络插件。</p><p>在上一篇文章中，我以Flannel项目为例，为你详细讲解了容器跨主机网络的两种实现方法：UDP和VXLAN。</p><p>不难看到，这些例子有一个共性，那就是用户的容器都连接在docker0网桥上。而网络插件则在宿主机上创建了一个特殊的设备（UDP模式创建的是TUN设备，VXLAN模式创建的则是VTEP设备），docker0与这个设备之间，通过IP转发（路由表）进行协作。</p><p>然后，网络插件真正要做的事情，则是通过某种方法，把不同宿主机上的特殊设备连通，从而达到容器跨主机通信的目的。</p><p>实际上，上面这个流程，也正是Kubernetes对容器网络的主要处理方法。只不过，Kubernetes是通过一个叫作CNI的接口，维护了一个单独的网桥来代替docker0。这个网桥的名字就叫作：CNI网桥，它在宿主机上的设备名称默认是：cni0。</p><p>以Flannel的VXLAN模式为例，在Kubernetes环境里，它的工作方式跟我们在上一篇文章中讲解的没有任何不同。只不过，docker0网桥被替换成了CNI网桥而已，如下所示：</p><p><img src="https://static001.geekbang.org/resource/image/9f/8c/9f11d8716f6d895ff6d1c813d460488c.jpg?wh=1767*933" alt=""></p><p>在这里，Kubernetes为Flannel分配的子网范围是10.244.0.0/16。这个参数可以在部署的时候指定，比如：</p><!-- [[[read_end]]] --><pre><code>$ kubeadm init --pod-network-cidr=10.244.0.0/16
</code></pre><p>也可以在部署完成后，通过修改kube-controller-manager的配置文件来指定。</p><p>这时候，假设Infra-container-1要访问Infra-container-2（也就是Pod-1要访问Pod-2），这个IP包的源地址就是10.244.0.2，目的IP地址是10.244.1.3。而此时，Infra-container-1里的eth0设备，同样是以Veth Pair的方式连接在Node 1的cni0网桥上。所以这个IP包就会经过cni0网桥出现在宿主机上。</p><p>此时，Node 1上的路由表，如下所示：</p><pre><code># 在Node 1上
$ route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
...
10.244.0.0      0.0.0.0         255.255.255.0   U     0      0        0 cni0
10.244.1.0      10.244.1.0      255.255.255.0   UG    0      0        0 flannel.1
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
</code></pre><p>因为我们的IP包的目的IP地址是10.244.1.3，所以它只能匹配到第二条规则，也就是10.244.1.0对应的这条路由规则。</p><p>可以看到，这条规则指定了本机的flannel.1设备进行处理。并且，flannel.1在处理完后，要将IP包转发到的网关（Gateway），正是“隧道”另一端的VTEP设备，也就是Node 2的flannel.1设备。所以，接下来的流程，就跟上一篇文章中介绍过的Flannel VXLAN模式完全一样了。</p><p>需要注意的是，CNI网桥只是接管所有CNI插件负责的、即Kubernetes创建的容器（Pod）。而此时，如果你用docker run单独启动一个容器，那么Docker项目还是会把这个容器连接到docker0网桥上。所以这个容器的IP地址，一定是属于docker0网桥的172.17.0.0/16网段。</p><p>Kubernetes之所以要设置这样一个与docker0网桥功能几乎一样的CNI网桥，主要原因包括两个方面：</p><ul>
<li>一方面，Kubernetes项目并没有使用Docker的网络模型（CNM），所以它并不希望、也不具备配置docker0网桥的能力；</li>
<li>另一方面，这还与Kubernetes如何配置Pod，也就是Infra容器的Network Namespace密切相关。</li>
</ul><p>我们知道，Kubernetes创建一个Pod的第一步，就是创建并启动一个Infra容器，用来“hold”住这个Pod的Network Namespace（这里，你可以再回顾一下专栏第13篇文章<a href="https://time.geekbang.org/column/article/40092">《为什么我们需要Pod？》</a>中的相关内容）。</p><p>所以，CNI的设计思想，就是：<strong>Kubernetes在启动Infra容器之后，就可以直接调用CNI网络插件，为这个Infra容器的Network Namespace，配置符合预期的网络栈。</strong></p><blockquote>
<p>备注：在前面第32篇文章<a href="https://time.geekbang.org/column/article/64948">《浅谈容器网络》</a>中，我讲解单机容器网络时，已经和你分享过，一个Network Namespace的网络栈包括：网卡（Network Interface）、回环设备（Loopback Device）、路由表（Routing Table）和iptables规则。</p>
</blockquote><p>那么，这个网络栈的配置工作又是如何完成的呢？</p><p>为了回答这个问题，我们就需要从CNI插件的部署和实现方式谈起了。</p><p>我们在部署Kubernetes的时候，有一个步骤是安装kubernetes-cni包，它的目的就是在宿主机上安装<strong>CNI插件所需的基础可执行文件</strong>。</p><p>在安装完成后，你可以在宿主机的/opt/cni/bin目录下看到它们，如下所示：</p><pre><code>$ ls -al /opt/cni/bin/
total 73088
-rwxr-xr-x 1 root root  3890407 Aug 17  2017 bridge
-rwxr-xr-x 1 root root  9921982 Aug 17  2017 dhcp
-rwxr-xr-x 1 root root  2814104 Aug 17  2017 flannel
-rwxr-xr-x 1 root root  2991965 Aug 17  2017 host-local
-rwxr-xr-x 1 root root  3475802 Aug 17  2017 ipvlan
-rwxr-xr-x 1 root root  3026388 Aug 17  2017 loopback
-rwxr-xr-x 1 root root  3520724 Aug 17  2017 macvlan
-rwxr-xr-x 1 root root  3470464 Aug 17  2017 portmap
-rwxr-xr-x 1 root root  3877986 Aug 17  2017 ptp
-rwxr-xr-x 1 root root  2605279 Aug 17  2017 sample
-rwxr-xr-x 1 root root  2808402 Aug 17  2017 tuning
-rwxr-xr-x 1 root root  3475750 Aug 17  2017 vlan
</code></pre><p>这些CNI的基础可执行文件，按照功能可以分为三类：</p><p><strong>第一类，叫作Main插件，它是用来创建具体网络设备的二进制文件</strong>。比如，bridge（网桥设备）、ipvlan、loopback（lo设备）、macvlan、ptp（Veth Pair设备），以及vlan。</p><p>我在前面提到过的Flannel、Weave等项目，都属于“网桥”类型的CNI插件。所以在具体的实现中，它们往往会调用bridge这个二进制文件。这个流程，我马上就会详细介绍到。</p><p><strong>第二类，叫作IPAM（IP Address Management）插件，它是负责分配IP地址的二进制文件</strong>。比如，dhcp，这个文件会向DHCP服务器发起请求；host-local，则会使用预先配置的IP地址段来进行分配。</p><p><strong>第三类，是由CNI社区维护的内置CNI插件</strong>。比如：flannel，就是专门为Flannel项目提供的CNI插件；tuning，是一个通过sysctl调整网络设备参数的二进制文件；portmap，是一个通过iptables配置端口映射的二进制文件；bandwidth，是一个使用Token Bucket Filter (TBF) 来进行限流的二进制文件。</p><p>从这些二进制文件中，我们可以看到，如果要实现一个给Kubernetes用的容器网络方案，其实需要做两部分工作，以Flannel项目为例：</p><p><strong>首先，实现这个网络方案本身</strong>。这一部分需要编写的，其实就是flanneld进程里的主要逻辑。比如，创建和配置flannel.1设备、配置宿主机路由、配置ARP和FDB表里的信息等等。</p><p><strong>然后，实现该网络方案对应的CNI插件</strong>。这一部分主要需要做的，就是配置Infra容器里面的网络栈，并把它连接在CNI网桥上。</p><p>由于Flannel项目对应的CNI插件已经被内置了，所以它无需再单独安装。而对于Weave、Calico等其他项目来说，我们就必须在安装插件的时候，把对应的CNI插件的可执行文件放在/opt/cni/bin/目录下。</p><blockquote>
<p>实际上，对于Weave、Calico这样的网络方案来说，它们的DaemonSet只需要挂载宿主机的/opt/cni/bin/，就可以实现插件可执行文件的安装了。你可以想一下具体应该怎么做，就当作一个课后小问题留给你去实践了。</p>
</blockquote><p>接下来，你就需要在宿主机上安装flanneld（网络方案本身）。而在这个过程中，flanneld启动后会在每台宿主机上生成它对应的<strong>CNI配置文件</strong>（它其实是一个ConfigMap），从而告诉Kubernetes，这个集群要使用Flannel作为容器网络方案。</p><p>这个CNI配置文件的内容如下所示：</p><pre><code>$ cat /etc/cni/net.d/10-flannel.conflist 
{
  &quot;name&quot;: &quot;cbr0&quot;,
  &quot;plugins&quot;: [
    {
      &quot;type&quot;: &quot;flannel&quot;,
      &quot;delegate&quot;: {
        &quot;hairpinMode&quot;: true,
        &quot;isDefaultGateway&quot;: true
      }
    },
    {
      &quot;type&quot;: &quot;portmap&quot;,
      &quot;capabilities&quot;: {
        &quot;portMappings&quot;: true
      }
    }
  ]
}
</code></pre><p>需要注意的是，在Kubernetes中，处理容器网络相关的逻辑并不会在kubelet主干代码里执行，而是会在具体的CRI（Container Runtime Interface，容器运行时接口）实现里完成。对于Docker项目来说，它的CRI实现叫作dockershim，你可以在kubelet的代码里找到它。</p><p>所以，接下来dockershim会加载上述的CNI配置文件。</p><p>需要注意，Kubernetes目前不支持多个CNI插件混用。如果你在CNI配置目录（/etc/cni/net.d）里放置了多个CNI配置文件的话，dockershim只会加载按字母顺序排序的第一个插件。</p><p>但另一方面，CNI允许你在一个CNI配置文件里，通过plugins字段，定义多个插件进行协作。</p><p>比如，在我们上面这个例子里，Flannel项目就指定了flannel和portmap这两个插件。</p><p><strong>这时候，dockershim会把这个CNI配置文件加载起来，并且把列表里的第一个插件、也就是flannel插件，设置为默认插件</strong>。而在后面的执行过程中，flannel和portmap插件会按照定义顺序被调用，从而依次完成“配置容器网络”和“配置端口映射”这两步操作。</p><p>接下来，我就来为你讲解一下这样一个CNI插件的工作原理。</p><p>当kubelet组件需要创建Pod的时候，它第一个创建的一定是Infra容器。所以在这一步，dockershim就会先调用Docker API创建并启动Infra容器，紧接着执行一个叫作SetUpPod的方法。这个方法的作用就是：为CNI插件准备参数，然后调用CNI插件为Infra容器配置网络。</p><p>这里要调用的CNI插件，就是/opt/cni/bin/flannel；而调用它所需要的参数，分为两部分。</p><p><strong>第一部分，是由dockershim设置的一组CNI环境变量。</strong></p><p>其中，最重要的环境变量参数叫作：CNI_COMMAND。它的取值只有两种：ADD和DEL。</p><p><strong>这个ADD和DEL操作，就是CNI插件唯一需要实现的两个方法。</strong></p><p>其中ADD操作的含义是：把容器添加到CNI网络里；DEL操作的含义则是：把容器从CNI网络里移除掉。</p><p>而对于网桥类型的CNI插件来说，这两个操作意味着把容器以Veth Pair的方式“插”到CNI网桥上，或者从网桥上“拔”掉。</p><p>接下来，我以ADD操作为重点进行讲解。</p><p>CNI的ADD操作需要的参数包括：容器里网卡的名字eth0（CNI_IFNAME）、Pod的Network Namespace文件的路径（CNI_NETNS）、容器的ID（CNI_CONTAINERID）等。这些参数都属于上述环境变量里的内容。其中，Pod（Infra容器）的Network Namespace文件的路径，我在前面讲解容器基础的时候提到过，即：/proc/&lt;容器进程的PID&gt;/ns/net。</p><blockquote>
<p>备注：这里你也可以再回顾下专栏第8篇文章<a href="https://time.geekbang.org/column/article/18119">《白话容器基础（四）：重新认识Docker容器》</a>中的相关内容。</p>
</blockquote><p>除此之外，在 CNI 环境变量里，还有一个叫作CNI_ARGS的参数。通过这个参数，CRI实现（比如dockershim）就可以以Key-Value的格式，传递自定义信息给网络插件。这是用户将来自定义CNI协议的一个重要方法。</p><p><strong>第二部分，则是dockershim从CNI配置文件里加载到的、默认插件的配置信息。</strong></p><p>这个配置信息在CNI中被叫作Network Configuration，它的完整定义你可以参考<a href="https://github.com/containernetworking/cni/blob/master/SPEC.md#network-configuration">这个文档</a>。dockershim会把Network Configuration以JSON数据的格式，通过标准输入（stdin）的方式传递给Flannel CNI插件。</p><p>而有了这两部分参数，Flannel CNI插件实现ADD操作的过程就非常简单了。</p><p>不过，需要注意的是，Flannel的CNI配置文件（ /etc/cni/net.d/10-flannel.conflist）里有这么一个字段，叫作delegate：</p><pre><code>...
     &quot;delegate&quot;: {
        &quot;hairpinMode&quot;: true,
        &quot;isDefaultGateway&quot;: true
      }
</code></pre><p>Delegate字段的意思是，这个CNI插件并不会自己做事儿，而是会调用Delegate指定的某种CNI内置插件来完成。对于Flannel来说，它调用的Delegate插件，就是前面介绍到的CNI bridge插件。</p><p>所以说，dockershim对Flannel CNI插件的调用，其实就是走了个过场。Flannel CNI插件唯一需要做的，就是对dockershim传来的Network Configuration进行补充。比如，将Delegate的Type字段设置为bridge，将Delegate的IPAM字段设置为host-local等。</p><p>经过Flannel CNI插件补充后的、完整的Delegate字段如下所示：</p><pre><code>{
    &quot;hairpinMode&quot;:true,
    &quot;ipMasq&quot;:false,
    &quot;ipam&quot;:{
        &quot;routes&quot;:[
            {
                &quot;dst&quot;:&quot;10.244.0.0/16&quot;
            }
        ],
        &quot;subnet&quot;:&quot;10.244.1.0/24&quot;,
        &quot;type&quot;:&quot;host-local&quot;
    },
    &quot;isDefaultGateway&quot;:true,
    &quot;isGateway&quot;:true,
    &quot;mtu&quot;:1410,
    &quot;name&quot;:&quot;cbr0&quot;,
    &quot;type&quot;:&quot;bridge&quot;
}
</code></pre><p>其中，ipam字段里的信息，比如10.244.1.0/24，读取自Flannel在宿主机上生成的Flannel配置文件，即：宿主机上的/run/flannel/subnet.env文件。</p><p>接下来，Flannel CNI插件就会调用CNI bridge插件，也就是执行：/opt/cni/bin/bridge二进制文件。</p><p>这一次，调用CNI bridge插件需要的两部分参数的第一部分、也就是CNI环境变量，并没有变化。所以，它里面的CNI_COMMAND参数的值还是“ADD”。</p><p>而第二部分Network Configration，正是上面补充好的Delegate字段。Flannel CNI插件会把Delegate字段的内容以标准输入（stdin）的方式传递给CNI bridge插件。</p><blockquote>
<p>此外，Flannel CNI插件还会把Delegate字段以JSON文件的方式，保存在/var/lib/cni/flannel目录下。这是为了给后面删除容器调用DEL操作时使用的。</p>
</blockquote><p>有了这两部分参数，接下来CNI bridge插件就可以“代表”Flannel，进行“将容器加入到CNI网络里”这一步操作了。而这一部分内容，与容器Network Namespace密切相关，所以我要为你详细讲解一下。</p><p>首先，CNI bridge插件会在宿主机上检查CNI网桥是否存在。如果没有的话，那就创建它。这相当于在宿主机上执行：</p><pre><code># 在宿主机上
$ ip link add cni0 type bridge
$ ip link set cni0 up
</code></pre><p>接下来，CNI bridge插件会通过Infra容器的Network Namespace文件，进入到这个Network Namespace里面，然后创建一对Veth Pair设备。</p><p>紧接着，它会把这个Veth Pair的其中一端，“移动”到宿主机上。这相当于在容器里执行如下所示的命令：</p><pre><code>#在容器里

# 创建一对Veth Pair设备。其中一个叫作eth0，另一个叫作vethb4963f3
$ ip link add eth0 type veth peer name vethb4963f3

# 启动eth0设备
$ ip link set eth0 up 

# 将Veth Pair设备的另一端（也就是vethb4963f3设备）放到宿主机（也就是Host Namespace）里
$ ip link set vethb4963f3 netns $HOST_NS

# 通过Host Namespace，启动宿主机上的vethb4963f3设备
$ ip netns exec $HOST_NS ip link set vethb4963f3 up 
</code></pre><p>这样，vethb4963f3就出现在了宿主机上，而且这个Veth Pair设备的另一端，就是容器里面的eth0。</p><p>当然，你可能已经想到，上述创建Veth Pair设备的操作，其实也可以先在宿主机上执行，然后再把该设备的一端放到容器的Network Namespace里，这个原理是一样的。</p><p>不过，CNI插件之所以要“反着”来，是因为CNI里对Namespace操作函数的设计就是如此，如下所示：</p><pre><code>err := containerNS.Do(func(hostNS ns.NetNS) error {
  ...
  return nil
})
</code></pre><p>这个设计其实很容易理解。在编程时，容器的Namespace是可以直接通过Namespace文件拿到的；而Host Namespace，则是一个隐含在上下文的参数。所以，像上面这样，先通过容器Namespace进入容器里面，然后再反向操作Host Namespace，对于编程来说要更加方便。</p><p>接下来，CNI bridge插件就可以把vethb4963f3设备连接在CNI网桥上。这相当于在宿主机上执行：</p><pre><code># 在宿主机上
$ ip link set vethb4963f3 master cni0
</code></pre><p>在将vethb4963f3设备连接在CNI网桥之后，CNI bridge插件还会为它设置<strong>Hairpin Mode（发夹模式）</strong>。这是因为，在默认情况下，网桥设备是不允许一个数据包从一个端口进来后，再从这个端口发出去的。但是，它允许你为这个端口开启Hairpin Mode，从而取消这个限制。</p><p>这个特性，主要用在容器需要通过<a href="https://en.wikipedia.org/wiki/Network_address_translation">NAT</a>（即：端口映射）的方式，“自己访问自己”的场景下。</p><p>举个例子，比如我们执行docker run -p 8080:80，就是在宿主机上通过iptables设置了一条<a href="http://linux-ip.net/html/nat-dnat.html">DNAT</a>（目的地址转换）转发规则。这条规则的作用是，当宿主机上的进程访问“&lt;宿主机的IP地址&gt;:8080”时，iptables会把该请求直接转发到“&lt;容器的IP地址&gt;:80”上。也就是说，这个请求最终会经过docker0网桥进入容器里面。</p><p>但如果你是在容器里面访问宿主机的8080端口，那么这个容器里发出的IP包会经过vethb4963f3设备（端口）和docker0网桥，来到宿主机上。此时，根据上述DNAT规则，这个IP包又需要回到docker0网桥，并且还是通过vethb4963f3端口进入到容器里。所以，这种情况下，我们就需要开启vethb4963f3端口的Hairpin Mode了。</p><p>所以说，Flannel插件要在CNI配置文件里声明hairpinMode=true。这样，将来这个集群里的Pod才可以通过它自己的Service访问到自己。</p><p>接下来，CNI bridge插件会调用CNI ipam插件，从ipam.subnet字段规定的网段里为容器分配一个可用的IP地址。然后，CNI bridge插件就会把这个IP地址添加在容器的eth0网卡上，同时为容器设置默认路由。这相当于在容器里执行：</p><pre><code># 在容器里
$ ip addr add 10.244.0.2/24 dev eth0
$ ip route add default via 10.244.0.1 dev eth0
</code></pre><p>最后，CNI bridge插件会为CNI网桥添加IP地址。这相当于在宿主机上执行：</p><pre><code># 在宿主机上
$ ip addr add 10.244.0.1/24 dev cni0
</code></pre><p>在执行完上述操作之后，CNI插件会把容器的IP地址等信息返回给dockershim，然后被kubelet添加到Pod的Status字段。</p><p>至此，CNI插件的ADD方法就宣告结束了。接下来的流程，就跟我们上一篇文章中容器跨主机通信的过程完全一致了。</p><p>需要注意的是，对于非网桥类型的CNI插件，上述“将容器添加到CNI网络”的操作流程，以及网络方案本身的工作原理，就都不太一样了。我将会在后续文章中，继续为你分析这部分内容。</p><h2>总结</h2><p>在本篇文章中，我为你详细讲解了Kubernetes中CNI网络的实现原理。根据这个原理，你其实就很容易理解所谓的“Kubernetes网络模型”了：</p><ol>
<li>
<p>所有容器都可以直接使用IP地址与其他容器通信，而无需使用NAT。</p>
</li>
<li>
<p>所有宿主机都可以直接使用IP地址与所有容器通信，而无需使用NAT。反之亦然。</p>
</li>
<li>
<p>容器自己“看到”的自己的IP地址，和别人（宿主机或者容器）看到的地址是完全一样的。</p>
</li>
</ol><p>可以看到，这个网络模型，其实可以用一个字总结，那就是“通”。</p><p>容器与容器之间要“通”，容器与宿主机之间也要“通”。并且，Kubernetes要求这个“通”，还必须是直接基于容器和宿主机的IP地址来进行的。</p><p>当然，考虑到不同用户之间的隔离性，在很多场合下，我们还要求容器之间的网络“不通”。这个问题，我会在后面的文章中会为你解决。</p><h2>思考题</h2><p>请你思考一下，为什么Kubernetes项目不自己实现容器网络，而是要通过 CNI 做一个如此简单的假设呢？</p><p>感谢你的收听，欢迎你给我留言，也欢迎分享给更多的朋友一起阅读。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/c2/e0/7188aa0a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>blackpiglet</span>
  </div>
  <div class="_2_QraFYR_0">思考题：为什么 Kubernetes 项目不自己实现容器网络，而是要通过 CNI 做一个如此简单的假设呢？<br><br>解答：没有亲历 Kubernetes 网络标准化的这个阶段，以下内容都是基于猜测，大家见笑了。<br>最开始我觉得这就是为了提供更多的便利选择，有了 CNI，那么只要符合规则，什么插件都可以用，用户的自由度更高，这是 Google 和 Kubernetes 开放性的体现。但转念一想，如果 Kubernetes 一开始就有官方的解决方案，恐怕也不会有什么不妥，感觉要理解的更深，得追溯到 Kubernetes 创建之初的外部环境和 Google 的开源策略了。Github 上最早的 Kubernetes 版本是 0.4，其中的网络部分，最开始官方的实现方式就是 GCE 执行 salt 脚本创建 bridge，其他环境的推荐的方案是 Flannel 和 OVS。<br>所以我猜测：<br>首先给 Kubernetes 发展的时间是不多的（Docker 已经大红大紫了，再不赶紧就一统天下了），给开发团队的时间只够专心实现编排这种最核心的功能，网络功能恰好盟友 CoreOS 的 Flannel 可以拿过来用，所以也可以认为 Flannel 就是最初 Kubernetes 的官方网络插件。Kubernetes 发展起来之后，Flannel 在有些情况下就不够用了，15 年左右社区里 Calico 和  Weave 冒了出来，基本解决了网络问题，Kubernetes 就更不需要自己花精力来做这件事了，所以推出了 CNI，来做网络插件的标准化。我觉得假如社区里网络一直没有好的解决方案的话，Kubernetes 肯定还是会亲自上阵的。<br>其次，Google 开源项目毕竟也不是做慈善，什么都做的面面俱到，那要消耗更多的成本，当然是越多的外部资源为我所用越好了。感觉推出核心功能，吸引开发者过来做贡献的搞法，也算是巨头们开源的一种套路吧。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 分析的很不错</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-11 23:58:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/6b/56/37a4cea7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>单朋荣</span>
  </div>
  <div class="_2_QraFYR_0">其实本章难点在于实现网络方案对应的CNI插件，即配置Infra容器的网络栈，并连到网桥上。整体流程是：kubelet创建Pod-&gt;创建Infra容器-&gt;调用SetUpPod（）方法，该方法需要为CNI准备参数，然后调用CNI插件（flannel)为Infra配置网络；其中参数来源于1、dockershim设置的一组CNI环境变量；2、dockershim从CNI配置文件里（有flanneld启动后生成，类型为configmap）加载到的、默认插件的配置信息（network configuration)，这里对CNI插件的调用，实际是network configuration进行补充。参数准备好后，调用Flannel CNI-&gt;调用CNI bridge（所需参数即为上面：设置的CNI环境变量和补充的network configuation）来执行具体的操作流程。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-14 14:43:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/a6/d2/ebe20bb5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>阿棠</span>
  </div>
  <div class="_2_QraFYR_0">前几章都很好理解，一到网络这块，就蒙了，没耐心看下去了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-21 09:45:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/6b/56/37a4cea7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>单朋荣</span>
  </div>
  <div class="_2_QraFYR_0">把握几个核心，然后串起来，其它需要的东西再去拿就可以了。<br>问题牵引：<br>网络方案是谁？它和“CNI标准”的关系（实现）是？kubernetes网络配置由谁来完成？（或者说我要怎么做才能实现它？？）<br><br>核心支撑点：<br>1、flannel网络方案本身<br>2、CNI插件，这里是内置的Flannel插件<br>3、dockershim(DRI)<br><br>两个背景知识：<br>1、CNI 的设计思想：Kubernetes 在启动 Infra 容器之后，就可以直接调用 CNI 网络插件，为这个 Infra 容器的 Network Namespace，配置符合预期的网络栈。<br>2、建立网络的“三类”基础组件&#47;可执行文件。<br><br>串线（着重描述三个核心点之间的串联关系）：<br>kubelet 创建 Pod -&gt;创建 Infra 容器。主要是由（CRI）**dockershim **调用 Docker API 创建并启动 Infra 容器-&gt; SetUpPod方法。方法的作用是：1.为 CNI 插件准备参数，2.然后调用 CNI 插件为 Infra 容器配置网络。<br>1.所需参数-&gt;实现ADD&#47;DEL方法-&gt;CNI插件（*flannel插件*)实现。：<br>1.1参数一：由 dockershim 设置的一组 CNI 环境变量，ADD&#47;DEL方法参数。<br>1.2参数二：是 dockershim 从 CNI “配置文件”里加载到的、默认插件的配置信息；由*flannel网络方案本身*安装时生成。<br>2.调用 CNI 插件:<br>引：&quot;dockershim 对 *Flannel CNI 插件*的调用，其实就是走了个过场。Flannel CNI 插件唯一需要做的，就是对 dockershim 传来的 Network Configuration (CNI配置文件）进行补充。&quot;<br>接下来，Flannel CNI 插件-&gt;调用 CNI bridge 插件(参数一：“CNI环境变量&#47;ADD&quot;, 参数二：”Network Confiuration&#47;Delegate&quot;)，--&gt;“代表”Flannel，将容器加入CNI网络（cni0网桥）。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-14 14:01:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/20/10/5786e0d8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>bus801</span>
  </div>
  <div class="_2_QraFYR_0">要是再来一篇calico的就完美了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-27 13:04:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/d2/9d/5839e490.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>LÉON</span>
  </div>
  <div class="_2_QraFYR_0">一直在苦苦自学，在容器的存储还有网络一直困扰。一直在拜读不拜听 受益匪浅 继续努力</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-20 19:39:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/69/88/528442b0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Dale</span>
  </div>
  <div class="_2_QraFYR_0">我认为是在大规模的集群环境中，网络方案是最复杂的，针对不同的的环境和场景，网络需要灵活配置。k8s集群里只关心最终网络可以连通，而不需要在内部去实现各种复杂的网络模块，使用CNI可以方便灵活地自定义网络插件，网络可以独立。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-09 08:39:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/2d/24/28acca15.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>DJH</span>
  </div>
  <div class="_2_QraFYR_0">&quot;实际上，对于 Weave、Calico 这样的网络方案来说，它们的 DaemonSet 只需要挂载宿主机的 &#47;opt&#47;cni&#47;bin&#47;，就可以实现插件可执行文件的安装了。&quot;这个是用hostpath类型的卷实现吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-09 08:29:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>vincent</span>
  </div>
  <div class="_2_QraFYR_0">Kubernetes是否可以给Pod创建多张网卡，分配多个IP？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-29 14:24:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/1e/b4/7f1e7e20.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>毛玉明</span>
  </div>
  <div class="_2_QraFYR_0">k8s里可以不使用cni直接使用docker0的网桥吗，看了下目前公司的集群没有找到cni0的这个设备</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-11 15:51:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/bd/5b/94bb1036.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>燕岭听涛</span>
  </div>
  <div class="_2_QraFYR_0">老师，您好，咨询一个问题：flannel经常出现 no ip address available in range，出现后就只能重置节点。这个是什么原因造成的，为什么pod删除后不回收ip地址？或者还有别的解决办法吗？希望能收到您的回复。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 给个大点的range啊，还有看看ipam用的是啥配置？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-23 10:16:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/92/14/6ca50b3b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>大G来了呦</span>
  </div>
  <div class="_2_QraFYR_0">存储和网络需要好好吸收才行</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-03 17:51:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/01/96/afcb6174.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>冬冬</span>
  </div>
  <div class="_2_QraFYR_0">kubernetes使用cni作为pod的容器间通信的网桥（与docker0功能相同）<br>初始化pod网络流程：创建Infra容器 调用cni插件初始化infra容器网络（插件位置：&#47;opt&#47;cni&#47;bin&#47;flannel），开始 dockershim 设置的一组 CNI 环境变量（枚举值ADD、DELETE），用于表示将容器的VethPair插入或从cni0网桥移除。<br>与此同时，cni bridge插件检查cni网桥在宿主机上是否存在，若不存在则进行创建。接着，cni bridge插件在network namespace创建VethPair，将其中一端插入到宿主机的cni0网桥，另一端直接赋予容器实例eth0，cni插件把容器ip提供给dockershim 被kubelet用于添加到pod的status字段。接下来，cni bridge调用cni ipam插件 从ipam.subnet子网中给容器eth0网卡分配ip地址 同时设置default route配置，最后cni bridge插件为cni网桥设置ip地址。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-19 07:47:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/52/3c/d6fcb93a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张三</span>
  </div>
  <div class="_2_QraFYR_0">精彩！老师不但教知其然，而且完全详细的讲解所以然。谢谢。学习了。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-22 11:22:06</div>
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
  <div class="_2_QraFYR_0">因为现实中的容器网络太多样、太复杂，为了解耦、可扩展性，设计了CNI接口，这个接口实现了共同的功能：为infra容器的network namespace配置网络栈</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-19 14:34:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/fe/fc/e56c9c4a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>嘿！我的gakki</span>
  </div>
  <div class="_2_QraFYR_0">https:&#47;&#47;github.com&#47;y805939188&#47;k8s-cni-test<br>从 0 实现的简单 cni 插件，内附教学，感兴趣的同学可以瞅瞅</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-18 11:08:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/b4/94/2796de72.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>追风筝的人</span>
  </div>
  <div class="_2_QraFYR_0">K8S 太强了  比虚拟机解决方案 先进了很多   用更少的存储 做更多的事情</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-06 18:06:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/b4/94/2796de72.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>追风筝的人</span>
  </div>
  <div class="_2_QraFYR_0">所有容器都可以直接使用 IP 地址与其他容器通信，而无需使用 NAT。所有宿主机都可以直接使用 IP 地址与所有容器通信，而无需使用 NAT。反之亦然。容器自己“看到”的自己的 IP 地址，和别人（宿主机或者容器）看到的地址是完全一样的。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-06 18:05:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/13/8c/c86340ca.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>巴西</span>
  </div>
  <div class="_2_QraFYR_0">网络篇确实难度上来了,需要多看几遍</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-03 15:29:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/00/d3/ef5d2af0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_81c7c9</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，请问在CNI方案出现之前，有其它的容器固定IP方案吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-23 15:53:55</div>
  </div>
</div>
</div>
</li>
</ul>