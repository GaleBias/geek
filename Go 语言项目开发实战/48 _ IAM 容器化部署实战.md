<audio title="48 _ IAM 容器化部署实战" src="https://static001.geekbang.org/resource/audio/11/f7/119a19c72ded45ee09c7f703351721f7.mp3" controls="controls"></audio> 
<p>你好，我是孔令飞。</p><p>在 <a href="https://time.geekbang.org/column/article/415606">45讲</a>中，我介绍了一种基于Kubernetes的云原生架构设计方案。在云原生架构中，我们是通过Docker + Kubernetes来部署云原生应用的。那么这一讲，我就手把手教你如何在Kubernetes集群中部署好IAM应用。因为步骤比较多，所以希望你能跟着我完成每一个操作步骤。相信在实操的过程中，你也会学到更多的知识。</p><h2>准备工作</h2><p>在部署IAM应用之前，我们需要做以下准备工作：</p><ol>
<li>开通腾讯云容器服务镜像仓库。</li>
<li>安装并配置Docker。</li>
<li>准备一个Kubernetes集群。</li>
</ol><h3>开通腾讯云容器服务镜像仓库</h3><p>在Kubernetes集群中部署IAM应用，需要从镜像仓库下载指定的IAM镜像，所以首先需要有一个镜像仓库来托管IAM的镜像。我们可以选择将IAM镜像托管到<a href="https://hub.docker.com/">DockerHub</a>上，这也是docker运行时默认获取镜像的地址。</p><p>但因为DockerHub服务部署在国外，国内访问速度很慢。所以，我建议将IAM镜像托管在国内的镜像仓库中。这里我们可以选择腾讯云提供的镜像仓库服务，访问地址为<a href="https://console.cloud.tencent.com/tke2/registry">容器镜像服务个人版</a>。</p><p>如果你已经有腾讯云的镜像仓库，可以忽略腾讯云镜像仓库开通步骤。</p><p>在开通腾讯云镜像仓库之前，你需要<a href="https://cloud.tencent.com/document/product/378/17985">注册腾讯云账号</a>，并完成<a href="https://cloud.tencent.com/document/product/378/3629">实名认证</a>。</p><!-- [[[read_end]]] --><p>开通腾讯云镜像仓库的具体步骤如下：</p><p><strong>第一步，开通个人版镜像仓库。</strong></p><ol>
<li>
<p>登录<a href="https://console.cloud.tencent.com/tke2">容器服务控制台</a>，选择左侧导航栏中的【镜像仓库】&gt;【<a href="https://console.cloud.tencent.com/tke2/registry/user/self">个人版</a>】。</p>
</li>
<li>
<p>根据以下提示，填写相关信息，并单击<strong>【开通】</strong>进行初始化。如下图所示：</p>
</li>
</ol><p><img src="https://static001.geekbang.org/resource/image/1d/8c/1d4bdfe89bc049d224e7002c23a0ea8c.png?wh=1920x569" alt="图片"></p><ul>
<li><strong>用户名：</strong>默认是当前用户的账号ID，是你登录到腾讯云Docker镜像仓库的身份，可在 <a href="https://console.cloud.tencent.com/developer">账号信息</a> 页面获取。</li>
<li><strong>密码：</strong>是你登录到腾讯云 Docker 镜像仓库的凭证。</li>
</ul><p>这里需要你记录用户名及密码，用于推送及拉取镜像。假如我们开通的镜像仓库，用户名为<code>10000099xxxx</code>，密码为<code>iam59!z$</code>。</p><p>这里要注意，<code>10000099xxxx</code><strong>要替换成你镜像仓库的用户名。</strong></p><p><strong>第二步，登录到腾讯云Registry</strong><strong>（镜像仓库）。</strong></p><p>在我们开通完Registry，就可以登录Registry了。可以通过以下命令来登录腾讯云Registry：</p><pre><code class="language-bash">$ docker login --username=[username] ccr.ccs.tencentyun.com
</code></pre><p>这里的username是腾讯云账号 ID，开通时已注册，可在 <a href="https://console.cloud.tencent.com/developer">账号信息</a> 页面获取。docker命令会在后面安装。<br>
<strong>第三步，新建镜像仓库命名空间</strong><strong>。</strong></p><p>如果想使用镜像仓库，那么你首先需要创建一个用来创建镜像的命名空间。上一步，我们开通了镜像仓库，就可以在“命名空间”页签新建命名空间了，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/1e/5a/1e79852fdc5ee1a094bd42efdf21015a.png?wh=1920x522" alt="图片"></p><p>上图中，我们创建了一个叫<code>marmotedu</code>的命名空间。</p><p>这里，镜像仓库服务、命名空间、镜像仓库、标签这几个概念你可能弄不清楚。接下来，我详细介绍下四者的关系，关系如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/ab/3b/abe8358d3ea2851bc32c80fd9519c63b.jpg?wh=2248x1581" alt=""></p><p>先来看下我们使用镜像仓库的格式：<code>&lt;镜像仓库服务地址&gt;/&lt;命名空间&gt;/&lt;镜像仓库&gt;:&lt;标签&gt;</code>，例如<code>ccr.ccs.tencentyun.com/marmotedu/iam-apiserver-amd64:v1.1.0</code>。</p><p>如果想使用一个Docker镜像，我们首先需要开通一个镜像仓库服务（Registry），镜像仓库服务都会对外提供一个固定的地址供你访问。在Registry中，我们（User）可以创建一个或多个命名空间（Namespace），命名空间也可以简单理解为镜像仓库逻辑上的一个分组。</p><p>接下来，就可以在Namespace中创建一个或多个镜像仓库，例如<code>iam-apiserver-amd64</code>、<code>iam-authz-server-amd64</code>、<code>iam-pump-amd64</code>等。针对每一个镜像仓库，又可以创建多个标签（Tag），例如<code>v1.0.1</code>、<code>v1.0.2</code>等。</p><p><code>&lt;镜像仓库&gt;:&lt;标签&gt;</code>又称为镜像。镜像又分为私有镜像和公有镜像，公有镜像可供所有能访问Registry的用户下载使用，私有镜像只提供给通过授权的用户使用。</p><h3>安装Docker</h3><p>开通完镜像仓库之后，我们还需要安装Docker，用来构建和测试Docker镜像。下面我来讲解下具体的安装步骤。</p><p><strong>第一步，安装Docker前置条件检查。</strong></p><p>需要确保CentOS系统启用了<code>centos-extras</code> yum源，默认情况下已经启用，检查方式如下：</p><pre><code class="language-bash">$ cat /etc/yum.repos.d/CentOS-Extras.repo
# Qcloud-Extras.repo

[extras]
name=Qcloud-$releasever - Extras
baseurl=http://mirrors.tencentyun.com/centos/$releasever/extras/$basearch/os/
gpgcheck=1
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-Qcloud-8
</code></pre><p>如果<code>/etc/yum.repos.d/CentOS-Extras.repo</code>文件存在，且文件中<code>extras</code>部分的<code>enabled</code>配置项值为<code>1</code>，说明已经启用了<code>centos-extras</code> yum源。如果<code>/etc/yum.repos.d/CentOS-Extras.repo</code> 文件不存在，或者<code>enabled</code> 不为1，则需要创建<code>/etc/yum.repos.d/CentOS-Extras.repo</code> 文件，并将上述内容复制进去。</p><p><strong>第二步，安装docker。</strong></p><p>Docker官方文档 <a href="https://docs.docker.com/engine/install/centos/">Install Docker Engine on CentOS</a>提供了3种安装方法:</p><ul>
<li>通过Yum源安装。</li>
<li>通过RPM包安装</li>
<li>通过脚本安装。</li>
</ul><p>这里，我们选择最简单的安装方式：<strong>通过Yum源安装</strong>。它具体又分为下面3个步骤。</p><ol>
<li>安装docker。</li>
</ol><pre><code class="language-bash">$ sudo yum install -y yum-utils # 1. 安装 `yum-utils` 包，该包提供了 `yum-config-manager` 工具
$ sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo # 2. 安装 `docker-ce.repo` yum 源
$ sudo yum-config-manager --enable docker-ce-nightly docker-ce-test # 3. 启用 `nightly` 和 `test` yum 源
$ sudo yum install -y docker-ce docker-ce-cli containerd.io # 4. 安装最新版本的 docker 引擎和 containerd
</code></pre><ol start="2">
<li>启动docker。</li>
</ol><p>可以通过以下命令来启动 docker：</p><pre><code class="language-bash">$ sudo systemctl start docker
</code></pre><p>docker的配置文件是 <code>/etc/docker/daemon.json</code> ，这个配置文件默认是没有的，需要我们手动创建：</p><pre><code class="language-bash">$ sudo tee /etc/docker/daemon.json &lt;&lt; EOF
{
&nbsp; "bip": "172.16.0.1/24",
&nbsp; "registry-mirrors": [],
&nbsp; "graph": "/data/lib/docker"
}
EOF
</code></pre><p>这里，我来解释下常用的配置参数。</p><ul>
<li>registry-mirrors：仓库地址，可以根据需要修改为指定的地址。</li>
<li>graph：镜像、容器的存储路径，默认是/var/lib/docker。如果你的 <code>/</code> 目录存储空间满足不了需求，需要设置graph为更大的目录。</li>
<li>bip：指定容器的IP网段。</li>
</ul><p>配置完成后，需要重启Docker：</p><pre><code class="language-bash">$ sudo systemctl restart docker
</code></pre><ol start="3">
<li>测试Docker是否安装成功。</li>
</ol><pre><code class="language-bash">$ sudo docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
b8dfde127a29: Pull complete
Digest: sha256:0fe98d7debd9049c50b597ef1f85b7c1e8cc81f59c8d623fcb2250e8bec85b38
Status: Downloaded newer image for hello-world:latest
...
Hello from Docker!
This message shows that your installation appears to be working correctly.
....
</code></pre><p><code>docker run hello-world</code>命令会下载<code>hello-world</code>镜像，并启动容器，打印安装成功提示信息后退出。</p><p>这里注意，如果你通过Yum源安装失败，可以尝试Docker官方文档 <a href="https://docs.docker.com/engine/install/centos/">Install Docker Engine on CentOS</a>提供的其他方式安装。</p><p><strong>第三步，安装后配置。</strong></p><p>安装成功后，我们还需要做一些其他配置。主要有两个，一个是配置docker，使其可通过<code>non-root</code>用户使用，另一个是配置docker开机启动。</p><ol>
<li>使用<code>non-root</code>用户操作Docker</li>
</ol><p>我们在Linux系统上操作，为了安全，需要以普通用户的身份登录系统并执行操作。所以，我们需要配置docker，使它可以被<code>non-root</code>用户使用。具体配置方法如下：</p><pre><code class="language-bash">$ sudo groupadd docker # 1. 创建`docker`用户组
$ sudo usermod -aG docker $USER # 2. 将当前用户添加到`docker`用户组下
$ newgrp docker # 3. 重新加载组成员身份
$ docker run hello-world # 4. 确认能够以普通用户使用docker
</code></pre><p>如果在执行 <code>sudo groupadd docker</code> 时报 <code>groupadd: group 'docker' already exists</code> 错误，说明 <code>docker</code> 组已经存在了，可以忽略这个报错。</p><p>如果你在将用户添加到 docker 组之前，使用sudo运行过docker命令，你可能会看到以下错误：</p><pre><code class="language-plain">WARNING: Error loading config file: /home/user/.docker/config.json -
stat /home/user/.docker/config.json: permission denied
</code></pre><p>这个错误，我们可以通过删除<code>~/.docker/</code>目录来解决，或者通过以下命令更改<code>~/.docker/</code>目录的所有者和权限：</p><pre><code class="language-bash">$ sudo chown "$USER":"$USER" /home/"$USER"/.docker -R
$ sudo chmod g+rwx "$HOME/.docker" -R
</code></pre><ol start="2">
<li>配置docker开机启动</li>
</ol><p>大部分Linux发行版（RHEL、CentOS、Fedora、Debian、Ubuntu 16.04及更高版本）使用systemd来管理服务，包括指定开启时启动的服务。在Debian和Ubuntu上，Docker默认配置为开机启动。</p><p>在其他系统，我们需要手动配置Docker开机启动，配置方式如下（分别需要配置docker和containerd服务）：</p><p>要在引导时为其他发行版自动启动 Docker 和 Containerd，你可以使用以下命令：</p><pre><code class="language-bash">$ sudo systemctl enable docker.service # 设置 docker 开机启动
$ sudo systemctl enable containerd.service # 设置 containerd 开机启动
</code></pre><p>如果要禁止<code>docker</code>、<code>containerd</code>开启启动，可以用这个命令：</p><pre><code class="language-bash">$ sudo systemctl disable docker.service # 禁止 docker 开机启动
$ sudo systemctl disable containerd.service # 禁止 containerd 开机启动
</code></pre><h3>准备一个Kubernetes集群</h3><p>安装完Docker之后，还需要一个Kubernetes集群，来调度Docker容器。安装Kubernetes集群极其复杂，这里选择一种最简单的方式来准备一个Kubernetes集群：购买一个腾讯云Serverless 集群。</p><p>腾讯云Serverless 集群是腾讯云容器服务推出的无须用户购买节点即可部署工作负载的集群类型。你可以把它理解为一个标准的Kubernetes集群，不同的是Serverless集群是由腾讯云容器服务团队创建和维护，你只需要访问集群，部署你的资源，并按照容器真实的资源使用量支付费用即可。你可以登录腾讯云容器服务控制台（<a href="https://console.cloud.tencent.com/tke2">https://console.cloud.tencent.com/tke2</a>）购买Serverless集群。</p><p>如果你想自己搭建Kubernetes集群，这里建议你购买3台腾讯云CVM机器，并参照<a href="https://github.com/opsnull/follow-me-install-kubernetes-cluster">follow-me-install-kubernetes-cluster</a>教程来一步步搭建Kubernetes集群，CVM机器建议的最小配置如下：<br>
<img src="https://static001.geekbang.org/resource/image/88/0b/885d2431e7f2d9bcd02b32d33yye930b.jpg?wh=1920x876" alt=""></p><h4>EKS简介</h4><p>我先简单介绍一下EKS是什么。EKS（Elastic Kubernetes Service）即腾讯云弹性容器服务，是腾讯云容器服务推出的无须用户购买节点即可部署工作负载的服务模式。它完全兼容原生的 Kubernetes，支持使用原生方式创建及管理资源，按照容器真实的资源使用量计费。弹性容器服务 EKS 还扩展支持腾讯云的存储及网络等产品，同时确保用户容器的安全隔离，开箱即用。</p><h4>EKS费用</h4><p>那它是如何收费呢？EKS 是全托管的Serverless Kubernetes 服务，不会收取托管的 Master、etcd 等资源的费用。弹性集群内运行的工作负载采用后付费的按量计费模式，费用根据实际配置的资源量按使用时间计算。也就是说：<strong>Kubernetes集群本身是免费的，只有运行工作负载，消耗节点资源时收费。</strong></p><p>EKS有3种计费模式：预留券、按量计费、竞价模式，这里我建议选择<strong>按量计费</strong>。按量计费，支持按秒计费，按小时结算，随时购买随时释放，从专栏学习的角度来说，费用是最低的。EKS 会根据工作负载申请的 CPU、内存数值以及工作负载的运行时间来核算费用，具体定价，你可以参考：<a href="https://buy.cloud.tencent.com/price/eks">定价|弹性容器服务</a>。</p><p>这里我通过例子来说明一下费用问题，IAM应用会部署4个Deployment，每个Deployment一个副本：</p><ul>
<li>iam-apiserver：IAM REST API服务，提供用户、密钥、策略资源的CURD功能的API接口。</li>
<li>iam-authz-server：IAM资源授权服务，对外提供资源授权接口。</li>
<li>iam-pump：IAM数据清洗服务，从Redis中获取授权日志，处理后保存在MongoDB中。</li>
<li>iamctl：IAM应用的测试服务，登陆iamctl Pod可以执行iamctl命令和smoke测试脚本，完成对IAM应用的运维和测试。</li>
</ul><p>上述4个Deployment中的<strong>Pod配置均为0.25核、512Mi内存</strong>。</p><p>这里，我们根据EKS的费用计算公式 <code>费用 = 相关计费项配置 × 资源单位时间价格 × 运行时间</code> 计算IAM部署一天的费用：</p><pre><code>总费用 = (4 x 1) x (0.25 x 0.12 + 0.5 x 0.05) x 24 = 4.8 元
</code></pre><p>也就是按最低配置部署IAM应用，运行一天的费用是4.8元（一瓶水的钱，就能学到如何将IAM应用部署在Kubernetes平台上，很值！）。你可能想这个计算公式里每个数值都代表什么呢？我来解释一下，其中：</p><ul>
<li>(4 x 1)：Kubernetes Pod总个数（一共是4个Deployment，每个Pod 1个副本）。</li>
<li>0.25 x 0.12：连续运行1小时的CPU配置费用。</li>
<li>0.5 x 0.05：连续运行1小时的内存配置费用。</li>
<li>24：24小时，也即一天。</li>
</ul><p>这里需要注意，为了帮助你节省费用，上述配置都是最低配置。在实际生产环境中，建议的配置如下：<br>
<img src="https://static001.geekbang.org/resource/image/1d/35/1d20c7eeb9dba8f53194969b0da5e835.jpg?wh=1920x718" alt=""><br>
因为iam-pump组件是有状态的，并且目前没有实现抢占机制，所以副本数需要设置为1。</p><p>另外，Intel按量计费的配置费用见下图：</p><p><img src="https://static001.geekbang.org/resource/image/64/59/64a5f89f4cbea95e50691455a06b1e59.png?wh=1372x226" alt="图片"></p><p>在这里有个很重要的事情提醒你：<strong>学完本节课，销毁这些Deployment，避免被继续扣费。建议腾讯云账户余额不要超过50元。</strong></p><h4>申请EKS集群</h4><p>了解了EKS以及费用相关的问题，接下来我们看看如何申请EKS集群。你可以通过以下5步来申请EKS集群。在正式申请前，请先确保腾讯云账户有大于 <strong>10 元</strong>的账户余额，否则在创建和使用EKS集群的过程中可能会因为费用不足而报错。</p><ol>
<li>创建腾讯云弹性集群</li>
</ol><p>具体步骤如下：</p><p>首先，登录容器服务控制台，选择左侧导航栏中的【<a href="https://console.cloud.tencent.com/tke2/ecluster">弹性集群</a>】。<br>
然后，在页面上方选择需创建弹性集群的地域，并单击【新建】。在“创建弹性集群”页面，根据以下提示设置集群信息。如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/37/8c/3740a98a6140b9c85517bdd27f30268c.png?wh=1920x950" alt="图片"></p><p>页面中各选择项的意思，我来给你解释一下：</p><ul>
<li><strong>集群名称：</strong>创建的弹性集群名称，不超过60个字符。</li>
<li><strong>Kubernetes版本：</strong>弹性集群支持1.12以上的多个 Kubernetes 版本选择，建议选择最新的版本。</li>
<li><strong>所在地域：</strong>建议你根据所在地理位置选择靠近的地域，可降低访问延迟，提高下载速度。</li>
<li><strong>集群网络：</strong>已创建的弹性集群 VPC 网络，你可以选择私有网络中的子网用于弹性集群的容器网络，详情请见 <a href="https://cloud.tencent.com/document/product/215/20046">私有网络（VPC）</a> 。</li>
<li><strong>容器网络：</strong>为集群内容器分配在容器网络地址范围内的 IP 地址。弹性集群的 Pod 会直接占用 VPC 子网 IP，请尽量选择 IP 数量充足且与其他产品使用无冲突的子网。</li>
<li><strong>Service CIDR：</strong>集群的 ClusterIP Service 默认分配在所选 VPC 子网中，请尽量选择 IP 数量充足且与其他产品使用无冲突的子网。</li>
<li><strong>集群描述：</strong>创建集群的相关信息，该信息将显示在“集群信息”页面。</li>
</ul><p>设置完成后，单击【完成】即可开始创建，可在“弹性集群”列表页面查看集群的创建进度。</p><p>等待弹性集群创建完成，创建完成后的弹性集群页面如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/1c/3e/1c533956986531f0245b27864f17dc3e.png?wh=1920x376" alt="图片"></p><p>我们创建的弹性集群ID为<strong>cls-dc6sdos4</strong>。</p><ol start="2">
<li>开启外网访问</li>
</ol><p>如果想访问EKS集群，需要先开启EKS的外网访问能力，开启方法如下：</p><p>登录容器服务控制台 -&gt; 选择左侧导航栏中的【<a href="https://console.cloud.tencent.com/tke2/ecluster">弹性集群</a>】 -&gt; 进入<strong>cls-dc6sdos4</strong> 集群的详情页中 -&gt; 选择【基本信息】 -&gt; 点击【外网访问】按钮。如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/23/c6/235679797980de87432a0d4b05dd83c6.png?wh=1920x764" alt="图片"></p><p>这里要注意，开启外网访问时，为了安全，需要设置允许访问kube-apiserver的IP段。为了避免不必要的错误，外网访问地址我们设置为<code>0.0.0.0/0</code>  。如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/ea/21/ea627942e5a432401e5959ee90c06221.png?wh=920x441" alt="图片"></p><p>注意，只有测试时才可这么设置为 <code>0.0.0.0/0</code> ，如果是生产环境，建议严格限制可以访问kube-apiserver的来源IP。</p><ol start="3">
<li>安装kubectl命令行工具</li>
</ol><p>如果要访问EKS（标准的Kubernetes集群），比较高效的方式是通过Kubernetes提供的命令行工具kubectl来访问。所以，还需要安装kubectl工具。</p><p>安装方式如下：</p><pre><code class="language-bash">$ curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
$ mkdir -p $HOME/bin
$ mv kubectl $HOME/bin
$ chmod +x $HOME/bin/kubectl
</code></pre><p>具体可参考<a href="https://kubernetes.io/docs/tasks/tools/install-kubectl/">安装和设置 kubectl</a>。<br>
你可以通过以下命令来配置kubectl的bash自动补全：</p><pre><code class="language-bash">$ kubectl completion bash &gt; $HOME/.kube-completion.bash
$ echo 'source $HOME/.kube-completion.bash' &gt;&gt; ~/.bashrc
$ bash
</code></pre><ol start="4">
<li>下载并安装kubeconfig</li>
</ol><p>安装完kubectl工具之后，需要配置kubectl所读取的配置文件。</p><p>这里注意，在上一步，我们开启了外网访问，开启后EKS会生成一个kubeconfig配置（ <code>kubeconfig</code> 即为kubectl的配置文件）。我们可以从页面下载并安装。</p><p>在弹性集群的基本信息页面，点击【复制】按钮，复制kubeconfig文件内容，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/bc/bf/bc314604334dd56b7337a905c9b6bfbf.png?wh=1775x731" alt="图片"></p><p>复制后，将粘贴板的内容保存在<code>$HOME/.kube/config</code>文件中。需要先执行<code>mkdir -p $HOME/.kube</code>创建<code>.kube</code>目录，再将粘贴版中的内容写到 <code>config</code> 文件中。</p><p>你可以通过以下命令，来测试kubectl工具是否成功安装和配置：</p><pre><code class="language-bash">$ kubectl get nodes
NAME                    STATUS   ROLES    AGE    VERSION
eklet-subnet-lowt256k   Ready    &lt;none&gt;   2d1h   v2.5.21
</code></pre><p>如果输出了Kubernetes的eklet节点，并且节点状态为<strong>Ready</strong>，说明Kubernetes集群运行正常，并且kubectl安装和配置正确。</p><ol start="5">
<li>EKS集群开通集群内服务访问外网能力</li>
</ol><p>因为IAM应用中的数据库：MariaDB、Redis、MongoDB可能需要通过外网访问，所以还需要开通EKS中Pod访问外网的能力。</p><p>EKS支持通过配置 <a href="https://cloud.tencent.com/document/product/215/4975">NAT 网关</a> 和 <a href="https://cloud.tencent.com/document/product/215/4954">路由表</a> 来实现集群内服务访问外网。具体开启步骤，需要你查看腾讯云官方文档：<a href="https://cloud.tencent.com/document/product/457/48710">通过 NAT 网关访问外网</a>。</p><p>在开通过程中有以下两点需要你注意：</p><ul>
<li>在<strong>创建指向 NAT 网关的路由表</strong>步骤中，目的端要选择：0.0.0.0/0。</li>
<li>在<strong>关联子网至路由表</strong>步骤中，只关联创建EKS集群时选择的子网。</li>
</ul><p>如果你的数据库需要通过外网访问，这里一定要确保EKS集群成功开通集群内服务访问外网能力，否则部署IAM应用时会因为访问不了数据库而失败。</p><h2>安装IAM应用</h2><p>上面，我们开通了镜像仓库、安装了Docker引擎、安装和配置了Kubernetes集群，那么接下来，我们就来看下如何将IAM应用部署到Kubernetes集群中。</p><p>假设IAM项目仓库根目录路径为 <code>$IAM_ROOT</code>，具体安装步骤如下：</p><ol>
<li>配置<code>scripts/install/environment.sh</code></li>
</ol><p><code>scripts/install/environment.sh</code>文件中包含了各类自定义配置。你可能需要配置跟数据库相关的配置（当然，也可以都使用默认值）：</p><ul>
<li>MariaDB配置：environment.sh文件中以<code>MARIADB_</code>开头的变量。</li>
<li>Redis配置：environment.sh文件中以<code>REDIS_</code>开头的变量。</li>
<li>MongoDB配置：environment.sh文件中以<code>MONGO_</code>开头的变量。</li>
</ul><p>其他配置，使用默认值。</p><ol start="2">
<li>创建IAM应用的配置文件</li>
</ol><pre><code class="language-bash">$ cd ${IAM_ROOT}
$ make gen.defaultconfigs # 生成iam-apiserver、iam-authz-server、iam-pump、iamctl组件的默认配置文件
$ make gen.ca # 生成 CA 证书
</code></pre><p>上述命令会将IAM的配置文件存放在这个<code>${IAM_ROOT}/_output/configs/</code>目录下。</p><ol start="3">
<li>创建IAM命名空间</li>
</ol><p>我们将IAM应用涉及到的各类资源，都创建在<code>iam</code>命名空间中。将IAM资源创建在独立的命名空间中，不仅方便维护，还可以有效避免影响其他Kubernetes资源。</p><pre><code class="language-bash">$ kubectl create namespace iam
</code></pre><ol start="4">
<li>将IAM各服务的配置文件，以ConfigMap资源的形式保存在Kubernetes集群中</li>
</ol><pre><code class="language-bash">$ kubectl -n iam create configmap iam --from-file=${IAM_ROOT}/_output/configs/
$ kubectl -n iam get configmap iam
NAME   DATA   AGE
iam    4      13s
</code></pre><p>执行<code>kubectl -n iam get configmap iam</code>命令，可以成功获取创建的<code>iam</code> configmap。</p><p>如果你觉得每次执行<code>kubectl</code>命令都要指定<code>-n iam</code>选项很繁琐，你可以使用以下命令，将<code>kubectl</code>上下文环境中的命名空间指定为<code>iam</code>。设置后，执行<code>kubectl</code>命令，默认在<code>iam</code>命名空间下执行：</p><pre><code class="language-bash">$ kubectl config set-context `kubectl config current-context` --namespace=iam
</code></pre><ol start="5">
<li>将IAM各服务使用的证书文件，以ConfigMap资源的形式创建在Kubernetes集群中</li>
</ol><pre><code class="language-bash">$ kubectl -n iam create configmap iam-cert --from-file=${IAM_ROOT}/_output/cert
$ kubectl -n iam get configmap iam-cert
NAME       DATA   AGE
iam-cert   14     12s
</code></pre><p>执行<code>kubectl -n iam get configmap iam-cert</code>命令，可以成功获取创建的<code>iam-cert</code> configmap。</p><ol start="6">
<li>创建镜像仓库访问密钥</li>
</ol><p>在准备阶段，我们开通了腾讯云镜像仓库服务（访问地址为ccr.ccs.tencentyun.com），并创建了用户<code>10000099xxxx</code>，其密码为<code>iam59!z$</code>。</p><p>接下来，我们就可以创建docker-registry secret。Kubernetes在下载Docker镜像时，需要docker-registry secret来进行认证。创建命令如下：</p><pre><code class="language-bash">$ kubectl -n iam create secret docker-registry ccr-registry --docker-server=ccr.ccs.tencentyun.com --docker-username=10000099xxxx --docker-password='iam59!z$'
</code></pre><ol start="7">
<li>创建Docker镜像，并Push到镜像仓库</li>
</ol><p>将镜像Push到CCR镜像仓库，需要确保你已经登录到腾讯云CCR镜像仓库，如果没登录，可以执行以下命令来登录：</p><pre><code class="language-bash">$ docker login --username=[username] ccr.ccs.tencentyun.com
</code></pre><p>执行 <code>make push</code> 命令构建镜像，并将镜像Push到CCR镜像仓库：</p><pre><code class="language-bash">$ make push REGISTRY_PREFIX=ccr.ccs.tencentyun.com/marmotedu VERSION=v1.1.0
</code></pre><p>上述命令，会构建iam-apiserver-amd64、iam-authz-server-amd64、iam-pump-amd64、iamctl-amd64 四个镜像，并将这些镜像Push到腾讯云镜像仓库的<code>marmotedu</code>命名空间下。</p><p>构建的镜像如下：</p><pre><code class="language-bash">$ docker images|grep marmotedu
ccr.ccs.tencentyun.com/marmotedu/iam-pump-amd64           v1.1.0   e078d340e3fb        10 seconds ago      244MB
ccr.ccs.tencentyun.com/marmotedu/iam-apiserver-amd64      v1.1.0   5e90b67cc949        2 minutes ago       239MB
ccr.ccs.tencentyun.com/marmotedu/iam-authz-server-amd64   v1.1.0   6796b02be68c        2 minutes ago       238MB
ccr.ccs.tencentyun.com/marmotedu/iamctl-amd64             v1.1.0   320a77d525e3        2 minutes ago       235MB
</code></pre><ol start="8">
<li>修改 <code>${IAM_ROOT}/deployments/iam.yaml</code> 配置</li>
</ol><p>这里请你注意，如果在上一个步骤中，你构建的镜像tag不是 <code>v1.1.0</code> ，那么你需要修改  <code>${IAM_ROOT}/deployments/iam.yaml</code> 文件，并将iam-apiserver-amd64、 iam-authz-server-amd64、 iam-pump-amd64、iamctl-amd64 镜像的tag修改成你构建镜像时指定的tag。</p><ol start="9">
<li>部署IAM应用</li>
</ol><pre><code class="language-bash">$ kubectl -n iam apply -f ${IAM_ROOT}/deployments/iam.yaml
</code></pre><p>执行上述命令，会在<code>iam</code>命令空间下，创建一系列Kubernetes资源，可以使用以下命令，来获取这些资源的状态：</p><pre><code class="language-bash">$ kubectl -n iam get all
NAME                                    READY   STATUS    RESTARTS   AGE
pod/iam-apiserver-d8dc48596-wkhpl       1/1     Running   0          94m
pod/iam-authz-server-6bc899c747-fbpbk   1/1     Running   0          94m
pod/iam-pump-7dcbfd4f59-2w9vk           1/1     Running   0          94m
pod/iamctl-6fc46b8ccb-gs62l             1/1     Running   1          98m

NAME                       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
service/iam-apiserver      ClusterIP   192.168.0.174   &lt;none&gt;        8443/TCP,8080/TCP,8081/TCP   101m
service/iam-authz-server   ClusterIP   192.168.0.76    &lt;none&gt;        9443/TCP,9090/TCP            101m
service/iam-pump           ClusterIP   192.168.0.155   &lt;none&gt;        7070/TCP                     101m

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/iam-apiserver      1/1     1            1           101m
deployment.apps/iam-authz-server   1/1     1            1           101m
deployment.apps/iam-pump           1/1     1            1           101m
deployment.apps/iamctl             1/1     1            1           101m

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/iam-apiserver-d8dc48596       1         1         1       101m
replicaset.apps/iam-authz-server-6bc899c747   1         1         1       101m
replicaset.apps/iam-pump-7dcbfd4f59           1         1         1       101m
replicaset.apps/iamctl-6fc46b8ccb             1         1         1       101m
</code></pre><p>我们看到<code>pod/iam-apiserver-d8dc48596-wkhpl</code>、<code>pod/iam-authz-server-6bc899c747-fbpbk</code>、<code>pod/iam-pump-7dcbfd4f59-2w9vk</code>、<code>pod/iamctl-6fc46b8ccb-gs62l</code> 4个Pod都处在<code>Running</code>状态，说明服务都成功启动。</p><h2>测试IAM应用</h2><p>我们在<code>iam</code>命令空间下创建了一个测试Deployment <code>iamctl</code>。你可以登陆<code>iamctl</code> Deployment所创建出来的Pod，执行一些运维操作和冒烟测试。登陆命令如下：</p><pre><code class="language-bash">$ kubectl -n iam exec -it `kubectl -n iam get pods -l app=iamctl | awk '/iamctl/{print $1}'` -- bash
</code></pre><p>登陆到<code>iamctl-xxxxxxxxxx-xxxxx</code> Pod中后，就可以执行运维操作和冒烟测试了。</p><ol>
<li>运维操作</li>
</ol><p>在iamctl容器中，你可以使用iamctl工具提供的各类功能，iamctl以子命令的方式对外提供功能。命令执行效果见下图：</p><p><img src="https://static001.geekbang.org/resource/image/44/7e/4460dfd5cd7f7fae5cbb0c64e605367e.png?wh=1920x433" alt="图片"></p><ol start="2">
<li>冒烟测试</li>
</ol><pre><code class="language-bash"># cd /opt/iam/scripts/install
# ./test.sh iam::test::smoke
</code></pre><p>如果<code>./test.sh iam::test::smoke</code>命令，打印的输出中，最后一行为<code>congratulations, smoke test passed!</code>字符串，说明IAM应用安装成功。如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/b9/2e/b9688e6b0609571a06401da412b63d2e.png?wh=1920x603" alt="图片"></p><h2>销毁Serverless集群及其资源</h2><p>好了，到这里，你已经成功在Serverless集群中部署了IAM应用，Serverless的使命也就完成了。接下来，为避免账户被持续扣费，需要删除Serverless内的资源和集群。</p><ol>
<li>删除Serverless内创建的IAM资源</li>
</ol><pre><code class="language-bash">$ kubectl delete namespace iam
</code></pre><p>因为删除Namespace会删除Namespace下的所有资源，所以上述命令执行时间会久点。</p><ol start="2">
<li>删除Serverless集群</li>
</ol><p>登录腾讯云容器服务控制台，选择所创建的Serverless集群，删除即可。</p><h2>总结</h2><p>云原生架构设计中，需要将IAM应用部署到Kubernetes集群中。所以，首先需要你准备一个Kubernetes集群。你可以自己购买腾讯云CVM机器搭建Kubernetes集群，但这种方式费用高、操作复杂。所以，我建议你直接申请一个Serverless集群来部署IAM应用。</p><p>Serverless集群是一个标准的Kubernetes集群，可以快速申请，并免运维。Serverless集群只收取实际的资源使用费用。在专栏学习过程中，部署IAM应用期间产生的资源使用费用其实是很低的，所以推荐使用这种方式来部署IAM应用。</p><p>有了Kubernetes集群，就可以直接通过以下命令来部署整个IAM应用：</p><pre><code class="language-bash">$ kubectl -n iam apply -f ${IAM_ROOT}/deployments/iam.yaml
</code></pre><p>应用部署起来之后，我们可以登陆到 <code>iamctl-xxxxxxxxxx-xxxxx</code>Pod，并执行以下命令来测试整个IAM应用是否被成功部署：</p><pre><code class="language-bash"># cd /opt/iam/scripts/install
# ./test.sh iam::test::smoke
</code></pre><h2>课后练习</h2><ol>
<li>思考下，如何将MariaDB、MongoDB、Redis实现容器化？</li>
<li>思考下，如何更相信IAM应用中的iam-apiserver服务，试着更新这个服务。</li>
</ol><p>欢迎你在留言区与我交流讨论，我们下一讲见。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/05/11/06ae74fa.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Mr.凉</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，这块有点没懂，为什么执行make push 就会生成下面四个镜像？<br><br>$ make push REGISTRY_PREFIX=ccr.ccs.tencentyun.com&#47;marmotedu VERSION=v1.1.0<br><br>上述命令，会构建 iam-apiserver-amd64、iam-authz-server-amd64、iam-pump-amd64、iamctl-amd64 四个镜像<br><br><br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: make push会执行scripts&#47;make-rules&#47;image.mk下的image.push规则，改规则依赖如下：image.verify go.build.verify $(addprefix image.push., $(addprefix $(IMAGE_PLAT)., $(IMAGES)))<br><br>也就是执行make push前会依次执行以下操作：<br>1. 检查机器是否安装了docker，以及docker的版本是否符合预期<br>2. 检查是否安装了go，以及go版本是否符合预期<br>3. 构建镜像（image.build.%）<br>4. 执行docker push上传镜像到指定的镜像仓库</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-28 15:32:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/87/64/3882d90d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>yandongxiao</span>
  </div>
  <div class="_2_QraFYR_0">总结：<br>1. 开通容器服务镜像仓库，自己搭建的话可以使用 harbor<br>2. 安装并配置Docker，配置主要包括，docker 通过非 root 用户使用；配置 docker 开启启动；<br>3. 准备一个 Kubernetes 集群。可以参见 follow-me-install-kubernetes-cluster 教程</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复:  follow-me-install-kubernetes-cluster 是一个非常好的教程，搭建Kubernetes，建议参考下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-05 11:35:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/7f/d3/b5896293.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Realm</span>
  </div>
  <div class="_2_QraFYR_0">老师好！iam-apiserver使用到MySQL和cache，在k8s集群中部署以上4个应用，iam-apiserver的Pod连不上MySQL，应用拉不起来。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以看下《EKS 集群开通集群内服务访问外网能力》这部分，EKS集群需要开启外网访问能力，开启后可以通过Pod访问MySQL。<br><br>注意：<br>1. MySQL需要bind 0.0.0.0地址<br>2. 并且授权的来源IP最好设置为%，也就是任意来源IP可访问数据库<br><br>这个说明我稍后会更新到专栏中</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-26 09:57:27</div>
  </div>
</div>
</div>
</li>
</ul>