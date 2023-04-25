<audio title="17｜更真实的云原生：实际搭建多节点的Kubernetes集群" src="https://static001.geekbang.org/resource/audio/94/02/9484943a2ae2e0a09cd6e6438be29902.mp3" controls="controls"></audio> 
<p>你好，我是Chrono。</p><p>到今天，你学习这个专栏的进度就已经过半了，在前面的“入门篇”我们了解了Docker和容器技术，在“初级篇”我们掌握了Kubernetes的基本对象、原理和操作方法，一路走下来收获很多。</p><p>现在你应该对Kubernetes和容器编排有了一些初步的认识，那么接下来，让我们继续深入研究Kubernetes的其他API对象，也就是那些在Docker中不存在的但对云计算、集群管理至关重要的概念。</p><p>不过在那之前，我们还需要有一个比minikube更真实的Kubernetes环境，它应该是一个多节点的Kubernetes集群，这样更贴近现实中的生产系统，能够让我们尽快地拥有实际的集群使用经验。</p><p>所以在今天的这节课里，我们就来暂时忘掉minikube，改用kubeadm（<a href="https://kubernetes.io/zh/docs/reference/setup-tools/kubeadm/">https://kubernetes.io/zh/docs/reference/setup-tools/kubeadm/</a>）搭建出一个新的Kubernetes集群，一起来看看更真实的云原生环境。</p><h2>什么是kubeadm</h2><p>前面的几节课里我们使用的都是minikube，它非常简单易用，不需要什么配置工作，就能够在单机环境里创建出一个功能完善的Kubernetes集群，给学习、开发、测试都带来了极大的便利。</p><!-- [[[read_end]]] --><p>不过minikube还是太“迷你”了，方便的同时也隐藏了很多细节，离真正生产环境里的计算集群有一些差距，毕竟许多需求、任务只有在多节点的大集群里才能够遇到，相比起来，minikube真的只能算是一个“玩具”。</p><p>那么，多节点的Kubernetes集群是怎么从无到有地创建出来的呢？</p><p><a href="https://time.geekbang.org/column/article/529800">第10讲</a>说过Kubernetes是很多模块构成的，而实现核心功能的组件像apiserver、etcd、scheduler等本质上都是可执行文件，所以也可以采用和其他系统差不多的方式，使用Shell脚本或者Ansible等工具打包发布到服务器上。</p><p>不过Kubernetes里的这些组件的配置和相互关系实在是太复杂了，用Shell、Ansible来部署的难度很高，需要具有相当专业的运维管理知识才能配置、搭建好集群，而且即使这样，搭建的过程也非常麻烦。</p><p>为了简化Kubernetes的部署工作，让它能够更“接地气”，社区里就出现了一个专门用来在集群中安装Kubernetes的工具，名字就叫“<strong>kubeadm</strong>”，意思就是“Kubernetes管理员”。</p><p><img src="https://static001.geekbang.org/resource/image/f2/88/f27c7938cba21215621ac33635d63288.jpg?wh=1044x640" alt="图片"></p><p>kubeadm，原理和minikube类似，也是用容器和镜像来封装Kubernetes的各种组件，但它的目标不是单机部署，而是要能够轻松地在集群环境里部署Kubernetes，并且让这个集群接近甚至达到生产级质量。</p><p>而在保持这个高水准的同时，kubeadm还具有了和minikube一样的易用性，只要很少的几条命令，如 <code>init</code>、<code>join</code>、<code>upgrade</code>、<code>reset</code> 就能够完成Kubernetes集群的管理维护工作，这让它不仅适用于集群管理员，也适用于开发、测试人员。</p><h2>实验环境的架构是什么样的</h2><p>在使用kubeadm搭建实验环境之前，我们先来看看集群的架构设计，也就是说要准备好集群所需的硬件设施。</p><p>这里我画了一张系统架构图，图里一共有3台主机，当然它们都是使用虚拟机软件VirtualBox/VMWare虚拟出来的，下面我来详细说明一下：</p><p><img src="https://static001.geekbang.org/resource/image/yy/3e/yyf5db64d398b4d5dyyd5e8e23ece53e.jpg?wh=1920x1294" alt="图片"></p><p>所谓的多节点集群，要求服务器应该有两台或者更多，为了简化我们只取最小值，所以这个Kubernetes集群就只有两台主机，一台是Master节点，另一台是Worker节点。当然，在完全掌握了kubeadm的用法之后，你可以在这个集群里添加更多的节点。</p><p>Master节点需要运行apiserver、etcd、scheduler、controller-manager等组件，管理整个集群，所以对配置要求比较高，至少是2核CPU、4GB的内存。</p><p><img src="https://static001.geekbang.org/resource/image/d1/3c/d19a8ceafd4db10a5yy35c623384ba3c.png?wh=1504x920" alt="图片"></p><p>而Worker节点没有管理工作，只运行业务应用，所以配置可以低一些，为了节省资源我给它分配了1核CPU和1GB的内存，可以说是低到不能再低了。</p><p><img src="https://static001.geekbang.org/resource/image/ee/f3/eeee60b6e29d7b6c4c74f913ac663ef3.png?wh=1504x1024" alt="图片"></p><p>基于模拟生产环境的考虑，在Kubernetes集群之外还需要有一台起辅助作用的服务器。</p><p>它的名字叫Console，意思是控制台，我们要在上面安装命令行工具kubectl，所有对Kubernetes集群的管理命令都是从这台主机发出去的。这也比较符合实际情况，因为安全的原因，集群里的主机部署好之后应该尽量少直接登录上去操作。</p><p>要提醒你的是，Console这台主机只是逻辑上的概念，不一定要是独立，你在实际安装部署的时候完全可以复用之前minikube的虚拟机，或者直接使用Master/Worker节点作为控制台。</p><p>这3台主机共同组成了我们的实验环境，所以在配置的时候要注意它们的网络选项，必须是在同一个网段，你可以再回顾一下<a href="https://time.geekbang.org/column/article/528614">课前准备</a>，保证它们使用的是同一个“Host-Only”（VirtualBox）或者“自定”（VMWare Fusion）网络。</p><h2>安装前的准备工作</h2><p>不过有了架构图里的这些主机之后，我们还不能立即开始使用kubeadm安装Kubernetes，因为Kubernetes对系统有一些特殊要求，我们必须还要在Master和Worker节点上做一些准备。</p><p>这些工作的详细信息你都可以在Kubernetes的官网上找到，但它们分散在不同的文档里，比较凌乱，所以我把它们整合到了这里，包括改主机名、改Docker配置、改网络设置、改交换分区这四步。</p><p>第一，由于Kubernetes使用主机名来区分集群里的节点，所以每个节点的hostname必须不能重名。你需要修改“<strong>/etc/hostname</strong>”这个文件，把它改成容易辨识的名字，比如Master节点就叫 <code>master</code>，Worker节点就叫 <code>worker</code>：</p><pre><code class="language-plain">sudo vi /etc/hostname
</code></pre><p>第二，虽然Kubernetes目前支持多种容器运行时，但Docker还是最方便最易用的一种，所以我们仍然继续使用Docker作为Kubernetes的底层支持，使用 <code>apt</code> 安装Docker Engine（可参考<a href="https://time.geekbang.org/column/article/528619">第1讲</a>）。</p><p>安装完成后需要你再对Docker的配置做一点修改，在“<strong>/etc/docker/daemon.json</strong>”里把cgroup的驱动程序改成 <code>systemd</code> ，然后重启Docker的守护进程，具体的操作我列在了下面：</p><pre><code class="language-bash">cat &lt;&lt;EOF | sudo tee /etc/docker/daemon.json
{
&nbsp; "exec-opts": ["native.cgroupdriver=systemd"],
&nbsp; "log-driver": "json-file",
&nbsp; "log-opts": {
&nbsp; &nbsp; "max-size": "100m"
&nbsp; },
&nbsp; "storage-driver": "overlay2"
}
EOF

sudo systemctl enable docker
sudo systemctl daemon-reload
sudo systemctl restart docker
</code></pre><p>第三，为了让Kubernetes能够检查、转发网络流量，你需要修改iptables的配置，启用“br_netfilter”模块：</p><pre><code class="language-bash">cat &lt;&lt;EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat &lt;&lt;EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward=1 # better than modify /etc/sysctl.conf
EOF

sudo sysctl --system
</code></pre><p>第四，你需要修改“<strong>/etc/fstab</strong>”，关闭Linux的swap分区，提升Kubernetes的性能：</p><pre><code class="language-plain">sudo swapoff -a
sudo sed -ri '/\sswap\s/s/^#?/#/' /etc/fstab
</code></pre><p>完成之后，最好记得重启一下系统，然后给虚拟机拍个快照做备份，避免后续的操作失误导致重复劳动。</p><h2>安装kubeadm</h2><p>好，现在我们就要安装kubeadm了，在Master节点和Worker节点上都要做这一步。</p><p>kubeadm可以直接从Google自己的软件仓库下载安装，但国内的网络不稳定，很难下载成功，需要改用其他的软件源，这里我选择了国内的某云厂商：</p><pre><code class="language-plain">sudo apt install -y apt-transport-https ca-certificates curl

curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add -

cat &lt;&lt;EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF

sudo apt update
</code></pre><p>更新了软件仓库，我们就可以用 <code>apt install</code> 获取kubeadm、kubelet和kubectl这三个安装必备工具了。apt默认会下载最新版本，但我们也可以指定版本号，比如使用和minikube相同的“1.23.3”：</p><pre><code class="language-plain">sudo apt install -y kubeadm=1.23.3-00 kubelet=1.23.3-00 kubectl=1.23.3-00
</code></pre><p>安装完成之后，你可以用 <code>kubeadm version</code>、<code>kubectl version</code> 来验证版本是否正确：</p><pre><code class="language-plain">kubeadm version
kubectl version --client
</code></pre><p><img src="https://static001.geekbang.org/resource/image/72/c9/72d79f46d9132af0dca110d982eff1c9.png?wh=1408x306" alt="图片"></p><p>另外按照Kubernetes官网的要求，我们最好再使用命令 <code>apt-mark hold</code> ，锁定这三个软件的版本，避免意外升级导致版本错误：</p><pre><code class="language-plain">sudo apt-mark hold kubeadm kubelet kubectl
</code></pre><h2>下载Kubernetes组件镜像</h2><p>前面我说过，kubeadm把apiserver、etcd、scheduler等组件都打包成了镜像，以容器的方式启动Kubernetes，但这些镜像不是放在Docker Hub上，而是放在Google自己的镜像仓库网站gcr.io，而它在国内的访问很困难，直接拉取镜像几乎是不可能的。</p><p>所以我们需要采取一些变通措施，提前把镜像下载到本地。</p><p>使用命令 <code>kubeadm config images list</code> 可以查看安装Kubernetes所需的镜像列表，参数 <code>--kubernetes-version</code> 可以指定版本号：</p><pre><code class="language-plain">kubeadm config images list --kubernetes-version v1.23.3

k8s.gcr.io/kube-apiserver:v1.23.3
k8s.gcr.io/kube-controller-manager:v1.23.3
k8s.gcr.io/kube-scheduler:v1.23.3
k8s.gcr.io/kube-proxy:v1.23.3
k8s.gcr.io/pause:3.6
k8s.gcr.io/etcd:3.5.1-0
k8s.gcr.io/coredns/coredns:v1.8.6
</code></pre><p>知道了镜像的名字和标签就好办了，我们有两种方法可以比较容易地获取这些镜像。</p><p>第一种方法是利用minikube。因为minikube本身也打包了Kubernetes的组件镜像，所以完全可以从它的节点里把这些镜像导出之后再拷贝过来。</p><p>具体做法也很简单，先启动minikube，然后 <code>minikube ssh</code> 登录进虚拟节点，用 <code>docker save -o</code> 命令把相应版本的镜像都保存下来，再用 <code>minikube cp</code> 拷贝到本地，剩下的事情就不用我多说了：</p><p><img src="https://static001.geekbang.org/resource/image/66/4f/6609a62525bbf5d77eb7331f9835244f.png?wh=1848x484" alt="图片"></p><p>这种方法安全可靠，不过操作上麻烦了些，所以就有了第二种方法，从国内的镜像网站下载然后再用 <code>docker tag</code> 改名，能够使用Shell编程实现自动化：</p><pre><code class="language-plain">repo=registry.aliyuncs.com/google_containers

for name in `kubeadm config images list --kubernetes-version v1.23.3`; do

&nbsp; &nbsp; src_name=${name#k8s.gcr.io/}
&nbsp; &nbsp; src_name=${src_name#coredns/}

&nbsp; &nbsp; docker pull $repo/$src_name

&nbsp; &nbsp; docker tag $repo/$src_name $name
&nbsp; &nbsp; docker rmi $repo/$src_name
done
</code></pre><p>第二种方法速度快，但也有隐患，万一网站不提供服务，或者改动了镜像就比较危险了。</p><p>所以你可以把这两种方法结合起来，先用脚本从国内镜像仓库下载，然后再用minikube里的镜像做对比，只要IMAGE ID是一样就说明镜像是正确的。</p><p>这张截图就是Kubernetes 1.23.3的镜像列表（amd64/arm64），你在安装时可以参考：</p><p><img src="https://static001.geekbang.org/resource/image/11/5c/11d9d4c91b08d95e82e75406a4d3aa5c.png?wh=1920x353" alt="图片" title="amd64"></p><p><img src="https://static001.geekbang.org/resource/image/52/6c/528d9913620015f594988e648eeac66c.png?wh=1920x420" alt="图片" title="arm64"></p><h2>安装Master节点</h2><p>准备工作都做好了，现在就可以开始正式安装Kubernetes了，我们先从Master节点开始。</p><p>kubeadm的用法非常简单，只需要一个命令 <code>kubeadm init</code> 就可以把组件在Master节点上运行起来，不过它还有很多参数用来调整集群的配置，你可以用 <code>-h</code> 查看。这里我只说一下我们实验环境用到的3个参数：</p><ul>
<li><strong>--pod-network-cidr</strong>，设置集群里Pod的IP地址段。</li>
<li><strong>--apiserver-advertise-address</strong>，设置apiserver的IP地址，对于多网卡服务器来说很重要（比如VirtualBox虚拟机就用了两块网卡），可以指定apiserver在哪个网卡上对外提供服务。</li>
<li><strong>--kubernetes-version</strong>，指定Kubernetes的版本号。</li>
</ul><p>下面的这个安装命令里，我指定了Pod的地址段是“10.10.0.0/16”，apiserver的服务地址是“192.168.10.210”，Kubernetes的版本号是“1.23.3”：</p><pre><code class="language-plain">sudo kubeadm init \
&nbsp; &nbsp; --pod-network-cidr=10.10.0.0/16 \
&nbsp; &nbsp; --apiserver-advertise-address=192.168.10.210 \
&nbsp; &nbsp; --kubernetes-version=v1.23.3
</code></pre><p>因为我们已经提前把镜像下载到了本地，所以kubeadm的安装过程很快就完成了，它还会提示出接下来要做的工作：</p><pre><code class="language-plain">To start using your cluster, you need to run the following as a regular user:

&nbsp; mkdir -p $HOME/.kube
&nbsp; sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
&nbsp; sudo chown $(id -u):$(id -g) $HOME/.kube/config
</code></pre><p>意思是要在本地建立一个“<strong>.kube</strong>”目录，然后拷贝kubectl的配置文件，你只要原样拷贝粘贴就行。</p><p>另外还有一个很重要的“<strong>kubeadm join</strong>”提示，其他节点要加入集群必须要用指令里的token和ca证书，所以这条命令务必拷贝后保存好：</p><pre><code class="language-plain">Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.10.210:6443 --token tv9mkx.tw7it9vphe158e74 \
	--discovery-token-ca-cert-hash sha256:e8721b8630d5b562e23c010c70559a6d3084f629abad6a2920e87855f8fb96f3
</code></pre><p>安装完成后，你就可以使用 <code>kubectl version</code>、<code>kubectl get node</code> 来检查Kubernetes的版本和集群的节点状态了：</p><pre><code class="language-plain">kubectl version
kubectl get node
</code></pre><p><img src="https://static001.geekbang.org/resource/image/c6/09/c63ce96bfyy0e1bc2927d575a66ee209.png?wh=1482x366" alt="图片"></p><p>你会注意到Master节点的状态是“NotReady”，这是由于还缺少网络插件，集群的内部网络还没有正常运作。</p><h2>安装Flannel网络插件</h2><p>Kubernetes定义了CNI标准，有很多网络插件，这里我选择最常用的<strong>Flannel</strong>，可以在它的GitHub仓库里（<a href="https://github.com/flannel-io/flannel/">https://github.com/flannel-io/flannel/</a>）找到相关文档。</p><p>它安装也很简单，只需要使用项目的“<strong>kube-flannel.yml</strong>”在Kubernetes里部署一下就好了。不过因为它应用了Kubernetes的网段地址，你需要修改文件里的“<strong>net-conf.json</strong>”字段，把 <code>Network</code> 改成刚才kubeadm的参数 <code>--pod-network-cidr</code> 设置的地址段。</p><p>比如在这里，就要修改成“10.10.0.0/16”：</p><pre><code class="language-plain">&nbsp; net-conf.json: |
&nbsp; &nbsp; {
&nbsp; &nbsp; &nbsp; "Network": "10.10.0.0/16",
&nbsp; &nbsp; &nbsp; "Backend": {
&nbsp; &nbsp; &nbsp; &nbsp; "Type": "vxlan"
&nbsp; &nbsp; &nbsp; }
&nbsp; &nbsp; }
</code></pre><p>改好后，你就可以用 <code>kubectl apply</code> 来安装Flannel网络了：</p><pre><code class="language-plain">kubectl apply -f kube-flannel.yml
</code></pre><p>稍等一小会，等镜像拉取下来并运行之后，你就可以执行 <code>kubectl get node</code> 来看节点状态：</p><pre><code class="language-plain">kubectl get node
</code></pre><p><img src="https://static001.geekbang.org/resource/image/6a/7a/6a3c852abe5b193a6997b154163ed67a.png?wh=1434x184" alt="图片"></p><p>这时你应该能够看到Master节点的状态是“Ready”，表明节点网络也工作正常了。</p><h2>安装Worker节点</h2><p>如果你成功安装了Master节点，那么Worker节点的安装就简单多了，只需要用之前拷贝的那条 <code>kubeadm join</code> 命令就可以了，记得要用 <code>sudo</code> 来执行：</p><pre><code class="language-plain">sudo \
kubeadm join 192.168.10.210:6443 --token tv9mkx.tw7it9vphe158e74 \
	--discovery-token-ca-cert-hash sha256:e8721b8630d5b562e23c010c70559a6d3084f629abad6a2920e87855f8fb96f3
</code></pre><p>它会连接Master节点，然后拉取镜像，安装网络插件，最后把节点加入集群。</p><p>当然，这个过程中同样也会遇到拉取镜像的问题，你可以如法炮制，提前把镜像下载到Worker节点本地，这样安装过程中就不会再有障碍了。</p><p>Worker节点安装完毕后，执行 <code>kubectl get node</code> ，就会看到两个节点都是“Ready”状态：</p><p><img src="https://static001.geekbang.org/resource/image/f7/26/f756ece9e81af80a7204243f15777026.png?wh=1430x246" alt="图片"></p><p>现在让我们用 <code>kubectl run</code> ，运行Nginx来测试一下：</p><pre><code class="language-plain">kubectl run ngx --image=nginx:alpine
kubectl get pod -o wide
</code></pre><p><img src="https://static001.geekbang.org/resource/image/73/e9/73651e5f178e2daf6eaf7ac262e230e9.png?wh=1584x188" alt="图片"></p><p>会看到Pod运行在Worker节点上，IP地址是“10.10.1.2”，表明我们的Kubernetes集群部署成功。</p><h2>小结</h2><p>好了，把Master节点和Worker节点都安装好，我们今天的任务就算是基本完成了。</p><p>后面Console节点的部署工作更加简单，它只需要安装一个kubectl，然后复制“config”文件就行，你可以直接在Master节点上用“scp”远程拷贝，例如：</p><pre><code class="language-plain">scp `which kubectl` chrono@192.168.10.208:~/
scp ~/.kube/config chrono@192.168.10.208:~/.kube
</code></pre><p>今天的过程多一些，要点我列在了下面：</p><ol>
<li>kubeadm是一个方便易用的Kubernetes工具，能够部署生产级别的Kubernetes集群。</li>
<li>安装Kubernetes之前需要修改主机的配置，包括主机名、Docker配置、网络设置、交换分区等。</li>
<li>Kubernetes的组件镜像存放在gcr.io，国内下载比较麻烦，可以考虑从minikube或者国内镜像网站获取。</li>
<li>安装Master节点需要使用命令 <code>kubeadm init</code>，安装Worker节点需要使用命令 <code>kubeadm join</code>，还要部署Flannel等网络插件才能让集群正常工作。</li>
</ol><p>因为这些操作都是各种Linux命令，全手动敲下来确实很繁琐，所以我把这些步骤都做成了Shell脚本放在了GitHub上（<a href="https://github.com/chronolaw/k8s_study/tree/master/admin">https://github.com/chronolaw/k8s_study/tree/master/admin</a>），你可以下载后直接运行。</p><h2>课下作业</h2><p>最后的课下作业是实际动手操作，请你多花费一些时间，用虚拟机创建出集群节点，再用kubeadm部署出这个多节点的Kubernetes环境，在接下来的“中级篇”和“高级篇”里我们就会在这个Kubernetes集群里做实验。</p><p>安装部署过程中有任何疑问，欢迎在留言区留言，我会第一时间回复你。如果觉得有帮助，也欢迎分享给自己身边的朋友一起学习，下节课见。</p><p><img src="https://static001.geekbang.org/resource/image/d3/41/d3d76937e5f4eb6545a07b96bc731e41.jpg?wh=1920x2513" alt="图片"></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/30/65/d6/20670fd5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Obscure</span>
  </div>
  <div class="_2_QraFYR_0">不需要写脚本来下载镜像啊，一条命令搞定：<br>kubeadm init \<br>--apiserver-advertise-address=192.168.137.100 \<br>--image-repository registry.aliyuncs.com&#47;google_containers \<br>--kubernetes-version v1.23.6 \<br>--service-cidr=10.96.0.0&#47;12 \<br>--pod-network-cidr=10.244.0.0&#47;16</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个方法不错，感谢分享。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-04 18:03:27</div>
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
  <div class="_2_QraFYR_0">终于跑起来了，提醒大家一件事，那就是新节点如何加入到k8s集群中，第一遍执行有提示，后续的话，可以执行kubeadm token create --print-join-command这条命令显示加入方式</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-02 11:19:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/5d/11/e1f36640.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>怀朔</span>
  </div>
  <div class="_2_QraFYR_0">本章质量略显不足 .主要如下几点<br>1、已经在考虑多节点不如把多集群考虑进去 这样才是真正的线上环境？<br>2、因为网络问题 考虑仓库不足 如何直接引用国内云厂商镜像仓库？ 讲解如何把海外镜像复制国内镜像岂不是更好？<br>3、生产中明显最关键是网络插件问题 和cidr块 网络冲突问题 通篇没有讲 令人略显单薄</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.一步步来，不要着急。<br><br>2.国内的我很少用，大家也都懂。<br><br>3.什么都讲就会显得内容太多太杂，毕竟专栏的目标是入门，不是上生产系统。<br><br>4.感谢建议，会和编辑考虑再改进。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-29 00:40:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJo05FofKFWYN3joX4OyCfVrU2kK7xvKdZ4Ho7bof893fE0jXk1OcB5sKLk4C1SviaNlibAiaCtp8aww/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>努力学习不准懈怠</span>
  </div>
  <div class="_2_QraFYR_0">一定要关闭swap，不然kubelet无法启动！！！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-29 16:46:59</div>
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
  <div class="_2_QraFYR_0">1. 装环境时可先装一台然后克隆；<br>2. kubeadm init时指定的apiserver的ip需要是自己环境上master节点的ip地址，如果指定错误，可通过sudo kubeadm reset重置；<br>3. kube-flannel.yml 文件在仓库的Documentation目录下，只用copy到环境上修改下net-config.json里面的Network的地址即可；<br>4. 执行scp ~&#47;.kube&#47;config到console节点时，需要先在console节点上通过mkdir ~&#47;.kube创建下目录</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great </p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-31 19:28:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/95/8a/d74cdda5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>这里的人都叫我八进制</span>
  </div>
  <div class="_2_QraFYR_0">work节点NotReady解决办法<br><br>KubeletNotReady runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized<br>1.  mkdir -p &#47;etc&#47;cni&#47;net.d<br><br>2. vi 10-flannel.conflist<br><br>{<br>  &quot;name&quot;: &quot;cbr0&quot;,<br>  &quot;plugins&quot;: [<br>    {<br>      &quot;type&quot;: &quot;flannel&quot;,<br>      &quot;delegate&quot;: {<br>        &quot;hairpinMode&quot;: true,<br>        &quot;isDefaultGateway&quot;: true<br>      }<br>    },<br>    {<br>      &quot;type&quot;: &quot;portmap&quot;,<br>      &quot;capabilities&quot;: {<br>        &quot;portMappings&quot;: true<br>      }<br>    }<br>  ]<br>}<br><br><br>3.<br><br>systemctl daemon-reload<br><br>systemctl restart kubelet</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 欢迎经验分享。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-19 18:47:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2e/06/ff/047a7150.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>项**</span>
  </div>
  <div class="_2_QraFYR_0">1. --apiserver-advertise-address 参数指定成master的ip地址，不然初始化阶段会超时<br>找这个问题找了很久<br><br>2.初始话失败再重新尝试需要清空环境<br>#重置<br>sudo kubeadm reset<br>#干掉kubelet进程<br>ps -ef|grep kubelet<br>sudo kill -9 进程id<br>#删掉配置目录<br>sudo rm -rf  &#47;etc&#47;kubernetes&#47;manifests&#47;<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great!</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-17 10:35:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/99/f0/d9343049.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>星亦辰</span>
  </div>
  <div class="_2_QraFYR_0">cat &lt;&lt;EOF | sudo tee &#47;etc&#47;yum.repos.d&#47;kubernetes.repo<br>[kubernetes] <br>name=Kubernetes <br>baseurl=https:&#47;&#47;mirrors.aliyun.com&#47;kubernetes&#47;yum&#47;repos&#47;kubernetes-el7-x86_64 <br>enabled=1 <br>gpgcheck=0 <br>repo_gpgcheck=0 <br>EOF<br><br>补充一个 Yum 的源 <br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-01 16:04:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/7a/27/77ca2bc2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小林子</span>
  </div>
  <div class="_2_QraFYR_0">老师为啥不用 Calico 网络插件了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 后面高级篇会用，flannel简单，而且现在不是我们学习的重点。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-29 13:15:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/72/67/aa52812a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>stark</span>
  </div>
  <div class="_2_QraFYR_0">终于弄好了，坑不少 总结写在这里了 希望对朋友们有帮助https:&#47;&#47;blog.csdn.net&#47;xuezhiwu001&#47;article&#47;details&#47;128444657?spm=1001.2014.3001.5501</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-26 15:13:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/89/ad/4efd929a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>老荀</span>
  </div>
  <div class="_2_QraFYR_0">master 节点能显示正常<br>$ kubectl get node<br>NAME     STATUS   ROLES                  AGE     VERSION<br>master   Ready    control-plane,master   21m     v1.23.3<br>worker   Ready    &lt;none&gt;                 3m56s   v1.23.3<br><br>但是 worker 节点就显示<br>The connection to the server localhost:8080 was refused - did you specify the right host or port?<br><br>上面的教程里，worker 节点不就是提前下好镜像和 使用 join 加入集群吗？也没说要 拷贝 什么 配置文件啊，而且 worker 节点下 &#47;etc&#47;kubernetes 下也只有 kubelet.conf 这个配置文件，和 master 节点的不一样（admin.conf) 求助老师</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 要用kubectl就必须要拷贝.kube&#47;config文件，不管是在哪个节点上。<br><br>操作步骤就是kubeadm init之后的那几条命令，和master节点的类似。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-25 09:59:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/94/25/3bf277e5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>陈四丰</span>
  </div>
  <div class="_2_QraFYR_0">安装成功！<br>中间遇到一个“坑”，提醒同学们。<br>在运行https:&#47;&#47;github.com&#47;chronolaw&#47;k8s_study&#47;blob&#47;master&#47;admin&#47;master.sh的时候，一定要把注释掉的# --apiserver-advertise-address=192.168.10.210 换成自己的IP地址，并添加进去，否则会以10.0.x.x的IP运行。<br>感谢罗老师，祝同学们学习顺利！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-22 14:27:53</div>
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
  <div class="_2_QraFYR_0">请教老师几个问题：<br>Q1：脚本为什么对src_name两次赋值？<br>       脚本的for循环里面有如下两行：<br>       src_name=${name#k8s.gcr.io&#47;}    <br>       src_name=${src_name#coredns&#47;}<br>       为什么对同一个src_name两次赋值？<br><br>Q2：同一个虚拟机上是否可以同时按照minikube和kubeadm？<br>Q3：apiserver的IP应该是自己的虚拟机的IP，Pod地址段和虚拟机IP无关，采用私有地址段即可，<br>        是这样吗？<br>        文中提到：“apiserver 的服务地址是“192.168.10.210””，这个地址是作者自己虚拟机的IP，<br>        读者应该换成自己虚拟机的IP，是这样吗？<br>        文中提到：“我指定了 Pod 的地址段是“10.10.0.0&#47;16””，Pod的IP段和虚拟机无关，读者的环境<br>        也可以采用，对吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.这个属于shell编程了，因为coredns镜像的名字比较特别，做了特殊处理。<br><br>2.当然可以，但kubectl的config文件就得做个切换了。<br><br>3.是的，第一个IP地址是节点的地址，必须是你自己的，第二个是集群里的IP地址，可以用这个10.10.0.0。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-31 18:02:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/f8/25/06c86919.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>rexzhao</span>
  </div>
  <div class="_2_QraFYR_0">kube-flannel.yml 文件是在哪个位置？是从哪来的？ 修改的内容对吗好像不是 yml 文件格式。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以在专栏的GitHub仓库里找到，或者直接去flannel项目上获取。<br><br>注意看YAML，它用的是“:|” ，表示的是一个长字符串，内容是json格式。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-29 11:40:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2c/2b/22/441c4e51.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>逐書寄年</span>
  </div>
  <div class="_2_QraFYR_0">感謝老師詳盡的解說！我有的問題是：<br>當我部屬完cluster，也成功加入 woker node 後，想嘗試啟動 nginx pod，使用 `kubectl get pod -o wide` 檢查狀態後，卻發現一直處於 `ContainerCreating` 的狀態。不知道有沒有線索可以知道從何 debug 呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 用describe查看pod的状态，应该有一些提示信息。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-14 22:24:05</div>
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
  <div class="_2_QraFYR_0">sudo kubeadm init \<br>    --pod-network-cidr=10.10.0.0&#47;16 \<br>    --apiserver-advertise-address=192.168.10.210 \<br>    --kubernetes-version=v1.23.3<br><br>--apiserver-advertise-address=192.168.10.210要改为master虚拟机的ip，否则会报错</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，这点一定要注意，192.168.10.210应该是master节点的IP地址。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-07 14:45:57</div>
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
  <div class="_2_QraFYR_0">老师，在配置 Master 节点的时候有问题，报错如下<br><br>ppeng@master:~$ sudo kubeadm init \<br>    --pod-network-cidr=10.10.0.0&#47;16 \<br>    --apiserver-advertise-address=192.168.10.210 \<br>    --kubernetes-version=v1.23.3<br>[sudo] password for ppeng:<br>[init] Using Kubernetes version: v1.23.3<br>[preflight] Running pre-flight checks<br>error execution phase preflight: [preflight] Some fatal errors occurred:<br>	[ERROR Port-10259]: Port 10259 is in use<br>	[ERROR Port-10257]: Port 10257 is in use<br>	[ERROR FileAvailable--etc-kubernetes-manifests-kube-apiserver.yaml]: &#47;etc&#47;kubernetes&#47;manifests&#47;kube-apiserver.yaml already exists<br>	[ERROR FileAvailable--etc-kubernetes-manifests-kube-controller-manager.yaml]: &#47;etc&#47;kubernetes&#47;manifests&#47;kube-controller-manager.yaml already exists<br>	[ERROR FileAvailable--etc-kubernetes-manifests-kube-scheduler.yaml]: &#47;etc&#47;kubernetes&#47;manifests&#47;kube-scheduler.yaml already exists<br>	[ERROR FileAvailable--etc-kubernetes-manifests-etcd.yaml]: &#47;etc&#47;kubernetes&#47;manifests&#47;etcd.yaml already exists<br>	[ERROR Port-10250]: Port 10250 is in use<br>[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`<br>To see the stack trace of this error execute with --v=5 or higher<br><br>前面的步骤都已成功，docker 也成功拉取了镜像，不清楚这里报错的原因是什么</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 建议用kubeadm reset，弄个干净的环境再试试。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-02 01:45:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/b3/d9/cf061262.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>新时代农民工</span>
  </div>
  <div class="_2_QraFYR_0">老师请问下，成功按文中操作搭建集群，在测试运行 Nginx 的时候，通过expose 端口在NodePort， 为何只能在worker的node上访问， 而在master节点上，通过本地ip访问不了呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 应该是master节点不参与业务吧，可以试着把污点去掉。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-01 15:00:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83eqF6ViaDyAibEKbcKfWoGXe8lCbb8wqes5g3JezHWNLf4DIl92QwXX43HWv408BxzkOKmKb2HpKJuIw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_b537b2</span>
  </div>
  <div class="_2_QraFYR_0">入门好教程</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: thanks</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-30 17:04:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/b9/8b/d0763d9a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>进击的土豆</span>
  </div>
  <div class="_2_QraFYR_0">老师，kubeadm和rke的区别是什么</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: kubeadm是Kubernetes的安装部署工具，RKE是rancher公司的Kubernetes发行版，类似GKE，可以搜一下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-29 14:21:14</div>
  </div>
</div>
</div>
</li>
</ul>