<audio title="09｜走近云原生：如何在本机搭建小巧完备的Kubernetes环境" src="https://static001.geekbang.org/resource/audio/fd/02/fd0719d78645776f519d19711acd9a02.mp3" controls="controls"></audio> 
<p>你好，我是Chrono。</p><p>在前面的“入门篇”里，我们学习了以Docker为代表的容器技术，做好了充分的准备，那么今天我们就来看看什么是容器编排、什么是Kubernetes，还有应该怎么在自己的电脑上搭建出一个小巧完善的Kubernetes环境，一起走近云原生。</p><h2>什么是容器编排</h2><p>容器技术的核心概念是容器、镜像、仓库，使用这三大基本要素我们就可以轻松地完成应用的打包、分发工作，实现“一次开发，到处运行”的梦想。</p><p>不过，当我们熟练地掌握了容器技术，信心满满地要在服务器集群里大规模实施的时候，却会发现容器技术的创新只是解决了运维部署工作中一个很小的问题。现实生产环境的复杂程度实在是太高了，除了最基本的安装，还会有各式各样的需求，比如服务发现、负载均衡、状态监控、健康检查、扩容缩容、应用迁移、高可用等等。</p><p><img src="https://static001.geekbang.org/resource/image/47/da/4790335b7fdd6a29d2cdda3yy3e337da.png?wh=900x551" alt="图片" title="图片来自网络"></p><p>虽然容器技术开启了云原生时代，但它也只走出了一小步，再继续前进就无能为力了，因为这已经不再是隔离一两个进程的普通问题，而是要隔离数不清的进程，还有它们之间互相通信、互相协作的超级问题，困难程度可以说是指数级别的上升。</p><p>这些容器之上的管理、调度工作，就是这些年最流行的词汇：“<strong>容器编排</strong>”（Container Orchestration）。</p><!-- [[[read_end]]] --><p>容器编排这个词听起来好像挺高大上，但如果你理解了之后就会发现其实也并不神秘。像我们在上次课里使用Docker部署WordPress网站的时候，把Nginx、WordPress、MariaDB这三个容器理清次序、配好IP地址去运行，就是最初级的一种“容器编排”，只不过这是纯手工操作，比较原始、粗糙。</p><p>面对单机上的几个容器，“人肉”编排调度还可以应付，但如果规模上到几百台服务器、成千上万的容器，处理它们之间的复杂联系就必须要依靠计算机了，而目前计算机用来调度管理的“事实标准”，就是我们专栏的主角：Kubernetes。</p><h2>什么是Kubernetes</h2><p>现在大家谈到容器都会说是Docker，但其实早在Docker之前，Google在公司内部就使用了类似的技术（cgroup就是Google开发再提交给Linux内核的），只不过不叫容器。</p><p>作为世界上最大的搜索引擎，Google拥有数量庞大的服务器集群，为了提高资源利用率和部署运维效率，它专门开发了一个集群应用管理系统，代号Borg，在底层支持整个公司的运转。</p><p>2014年，Google内部系统要“升级换代”，从原来的Borg切换到Omega，于是按照惯例，Google会发表公开论文。</p><p>因为之前在发表MapReduce、BigTable、GFS时吃过亏（被Yahoo开发的Hadoop占领了市场），所以Google决定借着Docker的“东风”，在发论文的同时，把C++开发的Borg系统用Go语言重写并开源，于是Kubernetes就这样诞生了。</p><p><img src="https://static001.geekbang.org/resource/image/eb/bc/ebba08c9d360cb01a332d1720e97f1bc.png?wh=1074x508" alt=""></p><p>由于Kubernetes背后有Borg系统十多年生产环境经验的支持，技术底蕴深厚，理论水平也非常高，一经推出就引起了轰动。然后在2015年，Google又联合Linux基金会成立了CNCF（Cloud Native Computing Foundation，云原生基金会），并把Kubernetes捐献出来作为种子项目。</p><p>有了Google和Linux这两大家族的保驾护航，再加上宽容开放的社区，作为CNCF的“头把交椅”，Kubernetes旗下很快就汇集了众多行业精英，仅用了两年的时间就打败了同期的竞争对手Apache Mesos和Docker Swarm，成为了这个领域的唯一霸主。</p><p>那么，Kubernetes到底能够为我们做什么呢？</p><p>简单来说，Kubernetes就是一个<strong>生产级别的容器编排平台和集群管理系统</strong>，不仅能够创建、调度容器，还能够监控、管理服务器，它凝聚了Google等大公司和开源社区的集体智慧，从而让中小型公司也可以具备轻松运维海量计算节点——也就是“云计算”的能力。</p><h2>什么是minikube</h2><p>Kubernetes一般都运行在大规模的计算集群上，管理很严格，这就对我们个人来说造成了一定的障碍，没有实际操作环境怎么能够学好用好呢？</p><p>好在Kubernetes充分考虑到了这方面的需求，提供了一些快速搭建Kubernetes环境的工具，在官网（<a href="https://kubernetes.io/zh/docs/tasks/tools/">https://kubernetes.io/zh/docs/tasks/tools/</a>）上推荐的有两个：<strong>kind</strong>和<strong>minikube</strong>，它们都可以在本机上运行完整的Kubernetes环境。</p><p>我说一下对这两个工具的个人看法，供你参考。</p><p>kind基于Docker，意思是“Kubernetes in Docker”。它功能少，用法简单，也因此运行速度快，容易上手。不过它缺少很多Kubernetes的标准功能，例如仪表盘、网络插件，也很难定制化，所以我认为它比较适合有经验的Kubernetes用户做快速开发测试，不太适合学习研究。</p><p>不选kind还有一个原因，它的名字与Kubernetes YAML配置里的字段 <code>kind</code> 重名，会对初学者造成误解，干扰学习。</p><p>再来看minikube，从名字就能够看出来，它是一个“迷你”版本的Kubernetes，自从2016年发布以来一直在积极地开发维护，紧跟Kubernetes的版本更新，同时也兼容较旧的版本（最多只到之前的6个小版本）。</p><p>minikube最大特点就是“小而美”，可执行文件仅有不到100MB，运行镜像也不过1GB，但就在这么小的空间里却集成了Kubernetes的绝大多数功能特性，不仅有核心的容器编排功能，还有丰富的插件，例如Dashboard、GPU、Ingress、Istio、Kong、Registry等等，综合来看非常完善。</p><p>所以，我建议你在这个专栏里选择minikube来学习Kubernetes。</p><h2>如何搭建minikube环境</h2><p>minikube支持Mac、Windows、Linux这三种主流平台，你可以在它的官网（<a href="https://minikube.sigs.k8s.io/docs">https://minikube.sigs.k8s.io</a>）找到详细的安装说明，当然在我们这里就只用虚拟机里的Linux了。</p><p>minikube的最新版本是1.25.2，支持的Kubernetes版本是1.23.3，所以我们就选定它作为我们初级篇的学习工具。</p><p>minikube不包含在系统自带的apt/yum软件仓库里，我们只能自己去网上找安装包。不过因为它是用Go语言开发的，整体就是一个二进制文件，没有多余的依赖，所以安装过程也非常简单，只需要用curl或者wget下载就行。</p><p>minikube的官网提供了各种系统的安装命令，通常就是下载、拷贝这两步，不过你需要注意一下本机电脑的硬件架构，Intel芯片要选择带“<strong>amd64</strong>”后缀，Apple M1芯片要选择“<strong>arm64</strong>”后缀，选错了就会因为CPU指令集不同而无法运行：</p><p><img src="https://static001.geekbang.org/resource/image/d5/84/d526aa920fba9bee9856177495a1c884.png?wh=1920x1004" alt="图片"></p><p>我也把官网上Linux系统安装的命令抄在了这里，你可以直接拷贝后安装：</p><pre><code class="language-bash"># Intel x86_64
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64

# Apple arm64
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-arm64

sudo install minikube /usr/local/bin/
</code></pre><p>安装完成之后，你可以执行命令 <code>minikube version</code>，看看它的版本号，验证是否安装成功：</p><pre><code class="language-plain">minikube version
</code></pre><p><img src="https://static001.geekbang.org/resource/image/c0/ec/c01e21cb3835520fd148f063e67605ec.png?wh=1266x188" alt="图片"></p><p>不过minikube只能够搭建Kubernetes环境，要操作Kubernetes，还需要另一个专门的客户端工具“<strong>kubectl</strong>”。</p><p>kubectl的作用有点类似之前我们学习容器技术时候的工具“docker”，它也是一个命令行工具，作用也比较类似，同样是与Kubernetes后台服务通信，把我们的命令转发给Kubernetes，实现容器和集群的管理功能。</p><p>kubectl是一个与Kubernetes、minikube彼此独立的项目，所以不包含在minikube里，但minikube提供了安装它的简化方式，你只需执行下面的这条命令：</p><pre><code class="language-plain">minikube kubectl
</code></pre><p>它就会把与当前Kubernetes版本匹配的kubectl下载下来，存放在内部目录（例如 <code>.minikube/cache/linux/arm64/v1.23.3</code>），然后我们就可以使用它来对Kubernetes“发号施令”了。</p><p>所以，在minikube环境里，我们会用到两个客户端：minikube管理Kubernetes集群环境，kubectl操作实际的Kubernetes功能，和Docker比起来有点复杂。</p><p>我画了一个简单的minikube环境示意图，方便你理解它们的关系。</p><p><img src="https://static001.geekbang.org/resource/image/22/e3/22c4d6ef48a0cf009946ebbbc31b91e3.jpg?wh=1920x1406" alt="图片"></p><h2>实际验证minikube环境</h2><p>前面的工作都做完之后，我们就可以在本机上运行minikube，创建Kubernetes实验环境了。</p><p>使用命令 <code>minikube start</code> 会从Docker Hub上拉取镜像，以当前最新版本的Kubernetes启动集群。不过为了保证实验环境的一致性，我们可以在后面再加上一个参数 <code>--kubernetes-version</code>，明确指定要使用Kubernetes版本。</p><p>这里我使用“1.23.3”，启动命令就是：</p><pre><code class="language-bash">minikube start --kubernetes-version=v1.23.3
</code></pre><p><img src="https://static001.geekbang.org/resource/image/2d/d0/2db1bb67d11892a60b9204fc61e307d0.png?wh=1920x514" alt="图片"></p><p>（它的启动过程使用了比较活泼的表情符号，可能是想表现得平易近人吧，如果不喜欢也可以调整设置关闭它。）</p><p>现在Kubernetes集群就已经在我们本地运行了，你可以使用 <code>minikube status</code>、<code>minikube node list</code>这两个命令来查看集群的状态：</p><pre><code class="language-bash">minikube status
minikube node list
</code></pre><p><img src="https://static001.geekbang.org/resource/image/82/38/827df6e7b3b5836e093c887f753d4938.png?wh=882x604" alt="图片"></p><p>从截图里可以看到，Kubernetes集群里现在只有一个节点，名字就叫“minikube”，类型是“Control Plane”，里面有host、kubelet、apiserver三个服务，IP地址是192.168.49.2。</p><p>你还可以用命令 <code>minikube ssh</code> 登录到这个节点上，虽然它是虚拟的，但用起来和实机也没什么区别：</p><p><img src="https://static001.geekbang.org/resource/image/dd/03/dd04a831f618d70ba77bdaecdb108d03.png?wh=1920x245" alt="图片"></p><p>有了集群，接下来我们就可以使用kubectl来操作一下，初步体会Kubernetes这个容器编排系统，最简单的命令当然就是查看版本：</p><pre><code class="language-bash">kubectl version
</code></pre><p>不过这条命令还不能直接用，因为使用minikube自带的kubectl有一点形式上的限制，要在前面加上minikube的前缀，后面再有个 <code>--</code>，像这样：</p><pre><code class="language-bash">minikube kubectl -- version 
</code></pre><p>为了避免这个不大不小的麻烦，我建议你使用Linux的“<strong>alias</strong>”功能，为它创建一个别名，写到当前用户目录下的 <code>.bashrc</code> 里，也就是这样：</p><pre><code class="language-plain">alias kubectl="minikube kubectl --"
</code></pre><p>另外，kubectl还提供了命令自动补全的功能，你还应该再加上“<strong>kubectl completion</strong>”：</p><pre><code class="language-plain">source &lt;(kubectl completion bash)
</code></pre><p>现在，我们就可以愉快地使用kubectl了：</p><p><img src="https://static001.geekbang.org/resource/image/ab/f9/abdc85efa4d3b25d779faec7c80b5ff9.png?wh=1920x202" alt="图片"></p><p>下面我们<strong>在Kubernetes里运行一个Nginx应用，命令与Docker一样，也是 <code>run</code>，不过形式上有点区别，需要用 <code>--image</code> 指定镜像</strong>，然后Kubernetes会自动拉取并运行：</p><pre><code class="language-plain">kubectl run ngx --image=nginx:alpine
</code></pre><p>这里涉及Kubernetes里的一个非常重要的概念：<strong>Pod</strong>，你可以暂时把它理解成是“穿了马甲”的容器，查看Pod列表需要使用命令 <code>kubectl get pod</code>，它的效果类似 <code>docker ps</code>：</p><p><img src="https://static001.geekbang.org/resource/image/2a/95/2abb91592d0ff740134a5d7665cb7c95.png?wh=1352x304" alt="图片"></p><p>命令执行之后可以看到，在Kubernetes集群里就有了一个名字叫ngx的Pod正在运行，表示我们的这个单节点minikube环境已经搭建成功。</p><h2>小结</h2><p>好了，今天我们先了解了容器编排概念和Kubernetes的历史，然后在Linux虚拟机上安装了minikube和kubectl，运行了一个简单但完整的Kubernetes集群，实现了与云原生的“第一次亲密接触”。</p><p>那什么是云原生呢？这在CNCF上有明确的定义，不过我觉得太学术化了，我也不想机械重复，就讲讲我自己的通俗理解吧。</p><p>所谓的“云”，现在就指的是Kubernetes，那么“云原生”的意思就是应用的开发、部署、运维等一系列工作都要向Kubernetes看齐，使用容器、微服务、声明式API等技术，保证应用的整个生命周期都能够在Kubernetes环境里顺利实施，不需要附加额外的条件。</p><p>换句话说，“云原生”就是Kubernetes里的“原住民”，而不是从其他环境迁过来的“移民”。</p><p>最后照例小结一下今天的内容：</p><ol>
<li>容器技术只解决了应用的打包、安装问题，面对复杂的生产环境就束手无策了，解决之道就是容器编排，它能够组织管理各个应用容器之间的关系，让它们顺利地协同运行。</li>
<li>Kubernetes源自Google内部的Borg系统，也是当前容器编排领域的事实标准。minikube可以在本机搭建Kubernetes环境，功能很完善，适合学习研究。</li>
<li>操作Kubernetes需要使用命令行工具kubectl，只有通过它才能与Kubernetes集群交互。</li>
<li>kubectl的用法与docker类似，也可以拉取镜像运行，但操作的不是简单的容器，而是Pod。</li>
</ol><p>另外还要说一下Kubernetes的官网（<a href="https://kubernetes.io/zh/">https://kubernetes.io/zh/</a>），里面有非常详细的文档，包括概念解释、入门教程、参考手册等等，最难得的是它有全中文版本，我们阅读起来完全不会有语言障碍，希望你有时间多上去看看，及时获取官方第一手知识。</p><h2>课下作业</h2><p>最后是课下作业时间，给你留两个思考题：</p><ol>
<li>你是怎么理解容器编排和Kubernetes的？它们应该能够解决什么问题？</li>
<li>你认为Kubernetes和Docker之间有什么区别？</li>
</ol><p>欢迎积极留言参与讨论，觉得有收获也欢迎你转发给朋友一起学习，我们下节课见。</p><p><img src="https://static001.geekbang.org/resource/image/90/96/90a478eeb6ae8a6ccd988fedc3ab4096.jpg?wh=1920x3272" alt=""></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/23/bb/74/edc07099.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>柳成荫</span>
  </div>
  <div class="_2_QraFYR_0">镜像拉取成功，遇到了几个坑<br>1. docker版本过低，docker升级到20.10.1以上<br>2. 不能用root账号，加上--force<br>3. 镜像拉取不下来，切换到国内镜像，先执行 minikube delete<br>再执行 minikube start --image-mirror-country=&#39;cn&#39;</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-11 14:01:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/27/52/40/db9b0eb2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>自由</span>
  </div>
  <div class="_2_QraFYR_0">1.1、如何理解容器编排？<br>先拆成两个部分，什么是容器？什么是编排？以前，程序运行在物理机或虚拟机中。容器，是现代程序的运行方式。编排就是部署、管理应用程序的系统，能动态地响应变化，例如以下部分功能。<br> - 回滚<br> - 滚动升级<br> - 故障自愈<br> - 自动扩缩容<br>自动完成以上所有任务。需要人工最初进行一些配置，就可以一劳永逸。回顾一下，什么是容器编排，运行容器形式的应用程序，这些应用程序的构建方式，使它们能够实现回滚、滚动升级、故障自愈、自动扩缩容等。<br><br>1.2、如何理解 Kubernetes？<br>举一个例子，寄、收快递的过程。发件人将货物按照快递公司的标准打包，提供基本信息（收货地址等），然后交给快递小哥。其他事情，无需发件人操心了，例如快递用什么交通工具运输、司机走哪条高速等等。快递公司同时提供物流查询、截断快递等服务。重点在于，快递公司仅需要发件人提供基本信息。Kubernetes 也是类似的，将应用程序打包成容器，声明运行方式，交给 Kubernetes 即可，同时它提供了丰富的工具和 API 来控制、观测运行在平台之上的应用程序。<br><br>1.3 容器编排应该能够解决什么问题？<br>屏蔽底层的复杂性。<br><br>2、Kubernetes 和 Docker 之间有什么区别？<br>Docker 应用打包、测试、交付。Kubernetes 基于 Docker 的产物，进行编排、运行。例如现在有 1 个集群，3 个节点。这些节点，都以 Docker 作为容器运行时，Docker 是更偏向底层的技术。Kubernetes 更偏向上层的技术 ，它实现了对容器运行时的抽象，抽象的目的是兼容底层容器运行时（容器进行时技术不仅有 Docker，还有 containerd、kata 等，无论哪种容器运行时，Kubernetes 层面的操作都是一样的）以及解耦，同时还提供了一套容器运行时的标准。抽象的产物是容器运行时接口 CRI。<br><br>说的不对的地方请斧正 : - )<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: amazing！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-12 14:37:22</div>
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
  <div class="_2_QraFYR_0">记录一下我遇到的坑，和评论里面解决方案:<br>1. 不能用root账号，使用普通用户加上--force<br>2. 镜像拉取不下来，切换到国内镜像，先执行 minikube delete<br>再执行 minikube start --image-mirror-country=&#39;cn&#39;  --kubernetes-version=v1.23.3<br>3.  安装一直停留在这里    <br>  - Booting up control plane ...| 调整虚拟机配置，调到4c8g。<br>4. 我自己之前的一个配置错误，将minikube 生成的kubectl 加入到了path中。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: very good.</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-13 18:19:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2a/92/4b/1262f052.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>许飞</span>
  </div>
  <div class="_2_QraFYR_0">minikube start --kubernetes-version=v1.23.3 --image-mirror-country=&#39;cn&#39; --force<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-13 10:46:11</div>
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
  <div class="_2_QraFYR_0">尝试回答一下这两个问题：docker和k8s之间的区别，一个是容器技术，一个是容器编排技术，两者思考的维度是不一样的，就容器而言，容器解决的问题是隔离，是一次打包到处运行的问题，最大的价值就在于镜像的迁移。编排技术则是关注的是整个系统的问题，如果你只关注一个服务，迁移一个服务，那docker就够，但要迁移整个系统以及运维，那就需要编排，包括网络关系，负载均衡，回滚，监控，扩缩容问题则需要容器编排技术。<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-13 18:28:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/6b/e9/7620ae7e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>雨落～紫竹</span>
  </div>
  <div class="_2_QraFYR_0">碰到一坑 minikube start 无法启动 群友建议minikube delete --all --purge 解决</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-23 17:45:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/23/1f/90/bf183d37.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>DmonJ</span>
  </div>
  <div class="_2_QraFYR_0">当前最新的minikube是1.26.1, 文章中的是1.25.2, 可以使用指定下载版本的方式确保安装的版本与文章中的一致:<br>`<br>curl -LO https:&#47;&#47;github.com&#47;kubernetes&#47;minikube&#47;releases&#47;download&#47;v1.25.2&#47;minikube-linux-amd64<br>sudo install minikube-linux-amd64 &#47;usr&#47;local&#47;bin&#47;minikube<br>`<br>全部版本链接:`https:&#47;&#47;github.com&#47;kubernetes&#47;minikube&#47;releases`</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-14 20:52:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKELX1Rd1vmLRWibHib8P95NA87F4zcj8GrHKYQL2RcLDVnxNy1ia2geTWgW6L2pWn2kazrPNZMRVrIg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jxs1211</span>
  </div>
  <div class="_2_QraFYR_0">kubeadmin和minikube有什么区别，用哪个如何选择</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: kubeadm是Kubernetes的安装工具，minikube是一个集成学习环境，中级篇会讲kubeadm，慢慢来。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-11 12:14:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTIERY97h7dmXbtur6rhZWA9Jb3TtSsJh7icDdFjdLmruTXC22qibOVTmW2a04TxMhxqtNJibYL1iaU7yQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_8ac303</span>
  </div>
  <div class="_2_QraFYR_0">1、运行容器：kubectl run ngx --image=nginx:alpine<br>2、查看pod：kubectl get pods -o wide<br>NAME   READY   STATUS  <br>ngx    0&#47;1     ContainerCreating<br>3、结果：拉取镜像拉不下来，一直卡在那，但是用docker pull 单独拉去镜像是没问题的，鼓捣了两三个小时，后来发现了一个参数可以解决这个问题，--registry-mirror=https:&#47;&#47;twm4fpgj.mirror.aliyuncs.com 这部分是指定docker的镜像加速地址（我是用的阿里的地址，每个人的应该是不一样的），原来通过kubectl调用的docker的配置并不是&#47;etc&#47;docker&#47;daemon.json 里面的配置<br>4、重新安装：<br>minikube stop<br>minikube delete --all<br>minikube start --kubernetes-version=v1.23.9 --image-mirror-country=&#39;cn&#39; --registry-mirror=https:&#47;&#47;twm4fpgj.mirror.aliyuncs.com</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可能每个人的网络环境不一样吧，我一般不用国内的镜像网站，docker hub通常直接可以拉取。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-08 22:55:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/26/c4/2b/b3f917ec.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>一颗红心</span>
  </div>
  <div class="_2_QraFYR_0">直接使用官方的最新版，总是失败。<br>指定版本后可以正常启动😅</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-15 11:52:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/27/52/40/db9b0eb2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>自由</span>
  </div>
  <div class="_2_QraFYR_0">`Google 又联合 Linux 基金会成了 CNCF`，应该修改为`Google 又联合 Linux 基金会成立了 CNCF`</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">编辑回复: 哎我看看去，已经修改过来啦，非常感谢～</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-11 12:22:07</div>
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
  <div class="_2_QraFYR_0">老师你好，有几个小问题：<br>1. Hadoop是借鉴那几篇论文的内容吗？然后迅速占领了市场。<br>2. 文章中的“kubelet”服务是做什么的呀？<br>3. “minikube，运行镜像也不过 1GB”。 这里的运行镜像是什么？ 是我们之前学习的 image文件吗？<br>4. source &lt;(kubectl completion bash)  没看懂这个命令后面的部分，能大致说一下么？<br>5. 浏览器输入gcr.io 会跳转到 https:&#47;&#47;cloud.google.com&#47;container-registry&#47; 这个网站，是正确的吗？<br>6. minikube ssh   前面的“minikube”是 节点的名称，对吧？<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.是的，可以搜一下Hadoop的历史，挺有意思的。<br><br>2.kubelet是管理节点的，与apiservr通信，没有它节点就失联了。<br><br>3.minikube把Kubernetes的运行环境打包成了一个镜像，就是docker image，以容器的形式来模拟Kubernetes的节点。<br><br>4.可以看kubectl completion -h的说明<br><br>5.没错，gcr.io是的短域名<br><br>6. 命令行里的第一个minikube是程序名，默认会登录名字叫minikube的节点，可以看它的帮助信息。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-20 10:40:14</div>
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
  <div class="_2_QraFYR_0">老师，我说出本课比较影响体验的一个地方吧：<br><br>现在大家安装的minikube的版本都是：v1.26.0。（这个版本是6月22号发布的）大家七月份学习这个课程的话。<br><br>Release Notes 是这样说的：<br><br>Note: Using a minikube cluster created before 1.26.0, using the docker container-runtime (default), and plan to upgrade the Kubernetes version to v1.24+? After upgrading, you may need to delete &amp; recreate it. none driver users, cri-docker is now required to be installed, install instructions. See issue #14410 for more info.<br><br>Kubernetes version to v1.24+ 以上的版本是一个分水岭，老师应该讲v1.24+ 的版本会比较好一些吧？<br><br>不然按照课程中的方式，大家都只能安装v1.23.3，不知道怎么解决高版本Kubernetes的问题。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我们课程的主旨的入门学习Kubernetes，差一两个版本区别不大，快速上手才是最重要的。<br><br>后续等版本变化大了可以再补充完善。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-18 22:21:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/P5EIPG3R01kEcsSSm0UZlyysg3qak8qWQXlwKKIoCkdxKtyorxD6h4S7bVvNNBM9icynCGvZO0bA5jGNgy3oBiag/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_7e25fd</span>
  </div>
  <div class="_2_QraFYR_0">老师，如何解决镜像拉取的过程当中，速度过慢或者无法下载的问题呢？能加餐镜像加速的知识？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个没什么太好的办法，有的国内有镜像同步网站，其他的就没办法了。<br><br>到中级篇会讲一下拉取Kubernetes镜像的方法。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-11 11:14:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/42/df/a034455d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>罗耀龙@坐忘</span>
  </div>
  <div class="_2_QraFYR_0">按照老师的教程，集群成功拉起。<br>就是拉取镜像时要耐心一点<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-11 02:48:53</div>
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
  <div class="_2_QraFYR_0">思考题：<br><br>1. 容器编排的目的是使容器的管理更加的工程化，解决一些因为容器规模上升所带来的潜在问题，比如容器间通信，容器的负载均衡，等等。Kubernetes 就是让容器编排的最佳实践得以落地的一个有力工具<br><br>2. docker 技术目的是提供创建容器以及容器镜像方法，聚焦的点是容器技术本身。而 K8S 技术目的是解决由于容器规模增大而带来的各种工程化问题，它以容器技术为基础，但是聚焦的点不再是容器技术了，而是上层的系统架构等问题<br><br>老师，source &lt; (kubectl completion bash) 这条命令貌似在 bash 中有语法错误，好像是 &lt; 后面多了个空格，应该是 source &lt;(kubectl completion bash) ？<br><br>另外，minikube 默认是从 gcr.io 抓取镜像，如果 gcr.io 无法访问的话就换成 DockerHub 吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.感谢指正，不应该有空格。<br><br>2. minikube的镜像不在gcr.io上，但其他应用要是在gcr.io上就没办法了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-11 13:49:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/27/52/40/db9b0eb2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>自由</span>
  </div>
  <div class="_2_QraFYR_0">执行 minikube start --kubernetes-version=v1.23.3 至少需要虚拟机分配了 2 个 cpu。关闭虚拟机，然后在 virtualBox 中选中当前虚拟机，右键，进入设置，单击系统，单击处理器，调整处理器数量。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-11 13:20:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/27/ff/e4/927547a9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>无名无姓</span>
  </div>
  <div class="_2_QraFYR_0">更新的够早的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">编辑回复: 准时00:00更新😁</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-11 08:28:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/7e/25/3932dafd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>GeekNEO</span>
  </div>
  <div class="_2_QraFYR_0">docker：23.0.1<br>minikube：1.29.0<br>kubectl：1.26.1<br><br>使用最新版k8s需要执行如下命令：<br>minikube start --driver=docker --container-runtime=containerd</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: nice！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-16 19:11:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/23/45/27/4fbf8f6a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>luke Y</span>
  </div>
  <div class="_2_QraFYR_0">哈哈哈 wsl2(Ubuntu-20.04) 按照老师教的安装成功，不想虚拟机的可以试试😂  不过不管虚拟机还是别的方式别忘了备份</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 只要能够用Linux搭建出环境就好，不一定非要用虚拟机。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-07 17:18:08</div>
  </div>
</div>
</div>
</li>
</ul>