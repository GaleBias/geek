<audio title="14 _ 深入解析Pod对象（一）：基本概念" src="https://static001.geekbang.org/resource/audio/40/90/40f7fcf5f97b1f59b70cfbe5ccde9190.mp3" controls="controls"></audio> 
<p>你好，我是张磊。今天我和你分享的主题是：深入解析Pod对象之基本概念。</p>
<p>在上一篇文章中，我详细介绍了Pod这个Kubernetes项目中最重要的概念。而在今天这篇文章中，我会和你分享Pod对象的更多细节。</p>
<p>现在，你已经非常清楚：Pod，而不是容器，才是Kubernetes项目中的最小编排单位。将这个设计落实到API对象上，容器（Container）就成了Pod属性里的一个普通的字段。那么，一个很自然的问题就是：到底哪些属性属于Pod对象，而又有哪些属性属于Container呢？</p>
<p>要彻底理解这个问题，你就一定要牢记我在上一篇文章中提到的一个结论：Pod扮演的是传统部署环境里“虚拟机”的角色。这样的设计，是为了使用户从传统环境（虚拟机环境）向Kubernetes（容器环境）的迁移，更加平滑。</p>
<p>而如果你能把Pod看成传统环境里的“机器”、把容器看作是运行在这个“机器”里的“用户程序”，那么很多关于Pod对象的设计就非常容易理解了。</p>
<p>比如，<strong>凡是调度、网络、存储，以及安全相关的属性，基本上是Pod 级别的。</strong></p>
<p>这些属性的共同特征是，它们描述的是“机器”这个整体，而不是里面运行的“程序”。比如，配置这个“机器”的网卡（即：Pod的网络定义），配置这个“机器”的磁盘（即：Pod的存储定义），配置这个“机器”的防火墙（即：Pod的安全定义）。更不用说，这台“机器”运行在哪个服务器之上（即：Pod的调度）。</p><!-- [[[read_end]]] -->
<p>接下来，我就先为你介绍Pod中几个重要字段的含义和用法。</p>
<p><strong>NodeSelector：是一个供用户将Pod与Node进行绑定的字段</strong>，用法如下所示：</p>
<pre><code>apiVersion: v1
kind: Pod
...
spec:
 nodeSelector:
   disktype: ssd
</code></pre>
<p>这样的一个配置，意味着这个Pod永远只能运行在携带了“disktype: ssd”标签（Label）的节点上；否则，它将调度失败。</p>
<p><strong>NodeName</strong>：一旦Pod的这个字段被赋值，Kubernetes项目就会被认为这个Pod已经经过了调度，调度的结果就是赋值的节点名字。所以，这个字段一般由调度器负责设置，但用户也可以设置它来“骗过”调度器，当然这个做法一般是在测试或者调试的时候才会用到。</p>
<p><strong>HostAliases：定义了Pod的hosts文件（比如/etc/hosts）里的内容</strong>，用法如下：</p>
<pre><code>apiVersion: v1
kind: Pod
...
spec:
  hostAliases:
  - ip: &quot;10.1.2.3&quot;
    hostnames:
    - &quot;foo.remote&quot;
    - &quot;bar.remote&quot;
...
</code></pre>
<p>在这个Pod的YAML文件中，我设置了一组IP和hostname的数据。这样，这个Pod启动后，/etc/hosts文件的内容将如下所示：</p>
<pre><code>cat /etc/hosts
# Kubernetes-managed hosts file.
127.0.0.1 localhost
...
10.244.135.10 hostaliases-pod
10.1.2.3 foo.remote
10.1.2.3 bar.remote
</code></pre>
<p>其中，最下面两行记录，就是我通过HostAliases字段为Pod设置的。需要指出的是，在Kubernetes项目中，如果要设置hosts文件里的内容，一定要通过这种方法。否则，如果直接修改了hosts文件的话，在Pod被删除重建之后，kubelet会自动覆盖掉被修改的内容。</p>
<p>除了上述跟“机器”相关的配置外，你可能也会发现，<strong>凡是跟容器的Linux Namespace相关的属性，也一定是Pod 级别的</strong>。这个原因也很容易理解：Pod的设计，就是要让它里面的容器尽可能多地共享Linux Namespace，仅保留必要的隔离和限制能力。这样，Pod模拟出的效果，就跟虚拟机里程序间的关系非常类似了。</p>
<p>举个例子，在下面这个Pod的YAML文件中，我定义了shareProcessNamespace=true：</p>
<pre><code>apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  shareProcessNamespace: true
  containers:
  - name: nginx
    image: nginx
  - name: shell
    image: busybox
    stdin: true
    tty: true
</code></pre>
<p>这就意味着这个Pod里的容器要共享PID Namespace。</p>
<p>而在这个YAML文件中，我还定义了两个容器：一个是nginx容器，一个是开启了tty和stdin的shell容器。</p>
<p>我在前面介绍容器基础时，曾经讲解过什么是tty和stdin。而在Pod的YAML文件里声明开启它们俩，其实等同于设置了docker run里的-it（-i即stdin，-t即tty）参数。</p>
<p>如果你还是不太理解它们俩的作用的话，可以直接认为tty就是Linux给用户提供的一个常驻小程序，用于接收用户的标准输入，返回操作系统的标准输出。当然，为了能够在tty中输入信息，你还需要同时开启stdin（标准输入流）。</p>
<p>于是，这个Pod被创建后，你就可以使用shell容器的tty跟这个容器进行交互了。我们一起实践一下：</p>
<pre><code>$ kubectl create -f nginx.yaml
</code></pre>
<p>接下来，我们使用kubectl attach命令，连接到shell容器的tty上：</p>
<pre><code>$ kubectl attach -it nginx -c shell
</code></pre>
<p>这样，我们就可以在shell容器里执行ps指令，查看所有正在运行的进程：</p>
<pre><code>$ kubectl attach -it nginx -c shell
/ # ps ax
PID   USER     TIME  COMMAND
    1 root      0:00 /pause
    8 root      0:00 nginx: master process nginx -g daemon off;
   14 101       0:00 nginx: worker process
   15 root      0:00 sh
   21 root      0:00 ps ax
</code></pre>
<p>可以看到，在这个容器里，我们不仅可以看到它本身的ps ax指令，还可以看到nginx容器的进程，以及Infra容器的/pause进程。这就意味着，整个Pod里的每个容器的进程，对于所有容器来说都是可见的：它们共享了同一个PID Namespace。</p>
<p>类似地，<strong>凡是Pod中的容器要共享宿主机的Namespace，也一定是Pod级别的定义</strong>，比如：</p>
<pre><code>apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  hostNetwork: true
  hostIPC: true
  hostPID: true
  containers:
  - name: nginx
    image: nginx
  - name: shell
    image: busybox
    stdin: true
    tty: true
</code></pre>
<p>在这个Pod中，我定义了共享宿主机的Network、IPC和PID Namespace。这就意味着，这个Pod里的所有容器，会直接使用宿主机的网络、直接与宿主机进行IPC通信、看到宿主机里正在运行的所有进程。</p>
<p>当然，除了这些属性，Pod里最重要的字段当属“Containers”了。而在上一篇文章中，我还介绍过“Init Containers”。其实，这两个字段都属于Pod对容器的定义，内容也完全相同，只是Init Containers的生命周期，会先于所有的Containers，并且严格按照定义的顺序执行。</p>
<p>Kubernetes项目中对Container的定义，和Docker相比并没有什么太大区别。我在前面的容器技术概念入门系列文章中，和你分享的Image（镜像）、Command（启动命令）、workingDir（容器的工作目录）、Ports（容器要开发的端口），以及volumeMounts（容器要挂载的Volume）都是构成Kubernetes项目中Container的主要字段。不过在这里，还有这么几个属性值得你额外关注。</p>
<p><strong>首先，是ImagePullPolicy字段</strong>。它定义了镜像拉取的策略。而它之所以是一个Container级别的属性，是因为容器镜像本来就是Container定义中的一部分。</p>
<p>ImagePullPolicy的值默认是Always，即每次创建Pod都重新拉取一次镜像。另外，当容器的镜像是类似于nginx或者nginx:latest这样的名字时，ImagePullPolicy也会被认为Always。</p>
<p>而如果它的值被定义为Never或者IfNotPresent，则意味着Pod永远不会主动拉取这个镜像，或者只在宿主机上不存在这个镜像时才拉取。</p>
<p><strong>其次，是Lifecycle字段</strong>。它定义的是Container Lifecycle Hooks。顾名思义，Container Lifecycle Hooks的作用，是在容器状态发生变化时触发一系列“钩子”。我们来看这样一个例子：</p>
<pre><code>apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-demo
spec:
  containers:
  - name: lifecycle-demo-container
    image: nginx
    lifecycle:
      postStart:
        exec:
          command: [&quot;/bin/sh&quot;, &quot;-c&quot;, &quot;echo Hello from the postStart handler &gt; /usr/share/message&quot;]
      preStop:
        exec:
          command: [&quot;/usr/sbin/nginx&quot;,&quot;-s&quot;,&quot;quit&quot;]
</code></pre>
<p>这是一个来自Kubernetes官方文档的Pod的YAML文件。它其实非常简单，只是定义了一个nginx镜像的容器。不过，在这个YAML文件的容器（Containers）部分，你会看到这个容器分别设置了一个postStart和preStop参数。这是什么意思呢？</p>
<p>先说postStart吧。它指的是，在容器启动后，立刻执行一个指定的操作。需要明确的是，postStart定义的操作，虽然是在Docker容器ENTRYPOINT执行之后，但它并不严格保证顺序。也就是说，在postStart启动时，ENTRYPOINT有可能还没有结束。</p>
<p>当然，如果postStart执行超时或者错误，Kubernetes会在该Pod的Events中报出该容器启动失败的错误信息，导致Pod也处于失败的状态。</p>
<p>而类似地，preStop发生的时机，则是容器被杀死之前（比如，收到了SIGKILL信号）。而需要明确的是，preStop操作的执行，是同步的。所以，它会阻塞当前的容器杀死流程，直到这个Hook定义操作完成之后，才允许容器被杀死，这跟postStart不一样。</p>
<p>所以，在这个例子中，我们在容器成功启动之后，在/usr/share/message里写入了一句“欢迎信息”（即postStart定义的操作）。而在这个容器被删除之前，我们则先调用了nginx的退出指令（即preStop定义的操作），从而实现了容器的“优雅退出”。</p>
<p>在熟悉了Pod以及它的Container部分的主要字段之后，我再和你分享一下<strong>这样一个的Pod对象在Kubernetes中的生命周期</strong>。</p>
<p>Pod生命周期的变化，主要体现在Pod API对象的<strong>Status部分</strong>，这是它除了Metadata和Spec之外的第三个重要字段。其中，pod.status.phase，就是Pod的当前状态，它有如下几种可能的情况：</p>
<ol>
<li>
<p>Pending。这个状态意味着，Pod的YAML文件已经提交给了Kubernetes，API对象已经被创建并保存在Etcd当中。但是，这个Pod里有些容器因为某种原因而不能被顺利创建。比如，调度不成功。</p>
</li>
<li>
<p>Running。这个状态下，Pod已经调度成功，跟一个具体的节点绑定。它包含的容器都已经创建成功，并且至少有一个正在运行中。</p>
</li>
<li>
<p>Succeeded。这个状态意味着，Pod里的所有容器都正常运行完毕，并且已经退出了。这种情况在运行一次性任务时最为常见。</p>
</li>
<li>
<p>Failed。这个状态下，Pod里至少有一个容器以不正常的状态（非0的返回码）退出。这个状态的出现，意味着你得想办法Debug这个容器的应用，比如查看Pod的Events和日志。</p>
</li>
<li>
<p>Unknown。这是一个异常状态，意味着Pod的状态不能持续地被kubelet汇报给kube-apiserver，这很有可能是主从节点（Master和Kubelet）间的通信出现了问题。</p>
</li>
</ol>
<p>更进一步地，Pod对象的Status字段，还可以再细分出一组Conditions。这些细分状态的值包括：PodScheduled、Ready、Initialized，以及Unschedulable。它们主要用于描述造成当前Status的具体原因是什么。</p>
<p>比如，Pod当前的Status是Pending，对应的Condition是Unschedulable，这就意味着它的调度出现了问题。</p>
<p>而其中，Ready这个细分状态非常值得我们关注：它意味着Pod不仅已经正常启动（Running状态），而且已经可以对外提供服务了。这两者之间（Running和Ready）是有区别的，你不妨仔细思考一下。</p>
<p>Pod的这些状态信息，是我们判断应用运行情况的重要标准，尤其是Pod进入了非“Running”状态后，你一定要能迅速做出反应，根据它所代表的异常情况开始跟踪和定位，而不是去手忙脚乱地查阅文档。</p>
<h2>总结</h2>
<p>在今天这篇文章中，我详细讲解了Pod API对象，介绍了Pod的核心使用方法，并分析了Pod和Container在字段上的异同。希望这些讲解能够帮你更好地理解和记忆Pod YAML中的核心字段，以及这些字段的准确含义。</p>
<p>实际上，Pod API对象是整个Kubernetes体系中最核心的一个概念，也是后面我讲解各种控制器时都要用到的。</p>
<p>在学习完这篇文章后，我希望你能仔细阅读$GOPATH/src/k8s.io/kubernetes/vendor/k8s.io/api/core/v1/types.go里，type Pod struct ，尤其是PodSpec部分的内容。争取做到下次看到一个Pod的YAML文件时，不再需要查阅文档，就能做到把常用字段及其作用信手拈来。</p>
<p>而在下一篇文章中，我会通过大量的实践，帮助你巩固和进阶关于Pod API对象核心字段的使用方法，敬请期待吧。</p>
<h2>思考题</h2>
<p>你能否举出一些Pod（即容器）的状态是Running，但是应用其实已经停止服务的例子？相信Java Web开发者的亲身体会会比较多吧。</p>
<p>感谢你的收听，欢迎你给我留言，也欢迎分享给更多的朋友一起阅读。</p>
<p></p>

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
  <div class="_2_QraFYR_0">对于 Pod 状态是 Ready，实际上不能提供服务的情况能想到几个例子：<br>1. 程序本身有 bug，本来应该返回 200，但因为代码问题，返回的是500；<br>2. 程序因为内存问题，已经僵死，但进程还在，但无响应；<br>3. Dockerfile 写的不规范，应用程序不是主进程，那么主进程出了什么问题都无法发现；<br>4. 程序出现死循环。<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 课代表来了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-24 17:12:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/66/c4/038f9325.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Jeff.W</span>
  </div>
  <div class="_2_QraFYR_0">POD的直议是豆荚，豆荚中的一个或者多个豆属于同一个家庭，共享一个物理豆荚（可以共享调度、网络、存储，以及安全），每个豆虽然有自己的空间，但是由于之间的缝隙，可以近距离无缝沟通（Linux Namespace相关的属性）。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-12 14:08:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/d2/2f/04882ff8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>龙坤</span>
  </div>
  <div class="_2_QraFYR_0">你好，我进入shell容器，然后执行ps ax，跟例子的结果不一样。例子代码也一样，添加了shareProcessNamespace: true了，为什么不行呢，请问可能出现的原因在哪里<br>&#47; # ps<br>PID   USER     TIME  COMMAND<br>    1 root      0:00 sh<br>   10 root      0:00 ps<br><br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-26 16:40:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/55/73/0b6351b8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>细雨</span>
  </div>
  <div class="_2_QraFYR_0">问一下老师，infra 网络的镜像为什么取名字叫 pause 呀，难道它一直处于“暂停状态”吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对啊</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-03 15:26:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/96/50/bde525b1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>北卡</span>
  </div>
  <div class="_2_QraFYR_0">只要容器没有down掉，pod就会处于running状态。pod只会监控到容器的状态，但不会监控容器里面程序的运行状态。如果程序处于死循环，或者其他bug状态，但并没有异常退出，此刻容器还是会处于存活状态，但实际上程序已经不能工作了。<br>日常使用的感受是这样的，不知道对不。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-27 00:49:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/39/fa/a7edbc72.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>安排</span>
  </div>
  <div class="_2_QraFYR_0">你好，我进入shell容器，然后执行ps ax，跟例子的结果不一样。例子代码也一样，添加了shareProcessNamespace: true了，为什么不行呢，请问可能出现的原因在哪里<br>&#47; # ps<br>PID USER TIME COMMAND<br>1 root 0:00 sh<br>10 root 0:00 ps<br><br>请问怎么开启sharepid功能呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: apiserver 加—feature-gates=PodShareProcessNamespace=true 1.11后已经默认开启了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-05 17:01:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/6b/d8/abb7bfe3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>两两</span>
  </div>
  <div class="_2_QraFYR_0">pod runing好理解，但k8s怎么知道容器runing呢，通过什么标准判断？应用死循环，k8s怎么能感知？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: liveness和readiness啥区别？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-26 08:07:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/9a/52/a51cbdef.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>杨孔来</span>
  </div>
  <div class="_2_QraFYR_0">老师，如果pod中的image更新了（比如 通过jenkins发布了新版本），我想通过重启pod,获取最新的image，有什么命令，可以优雅的重启pod，而不影响当前pod提供的业务吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不是讲了prestop hook了？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-25 14:55:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/75/00/618b20da.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>徐海浪</span>
  </div>
  <div class="_2_QraFYR_0">以前我对容器的认识还不深，竟然用tail -f Catalina.out作为前台进程，这样即使tomcat进程挂掉，容器还是正在运行。应用不可用tomcat进程还在经常会遇到，比如内存溢出，或者应用依赖的数据库等外部系统动荡导致应用不正常。怎么在应用的角度来决定容器是否应该退出？应用提供一个健康检查url，跑前台shell定期检查该url，状态不对则shell退出，从而容器退出。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 吃一堑长一智</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-26 09:29:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/b1/ae/344fbda3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>追风</span>
  </div>
  <div class="_2_QraFYR_0">作者的那句评论。先让让子弹飞一会，让我看出了作者决胜千里之外的眼界。哈哈哈</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-01 09:49:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/15/a6/723854ee.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>姜戈</span>
  </div>
  <div class="_2_QraFYR_0">通过node selector将任务调度到了woker1   成功运行之后 再修改worker1的label,  任务会重新调度吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不会</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-25 18:37:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/ef/dd/599d77c0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>hexinzhe</span>
  </div>
  <div class="_2_QraFYR_0">比较想要知道优雅停机方面的更详细内容，比如说terminationgraceperiodseconds与prestop之间的关系，两者怎么用</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 前者就是sig term的超时时间。后者是要你自己编写逻辑处理的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-25 13:20:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/c8/8f/759b7761.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ethfoo</span>
  </div>
  <div class="_2_QraFYR_0">如果pod加了健康检查，是不是就不关心应用进程是不是容器的初始化进程呢？因为应用进程挂了，虽然容器不会自动退出，但是kubelet会主动去kill掉这个容器</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以这么理解</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-25 10:24:15</div>
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
  <div class="_2_QraFYR_0">各位，中秋节好。<br>如果entrypoint是一个一直运行的命令，那postStart会执行吗？还是启动一个协程成执行entrypoint，然后再运行一个协程执行这个postStart，所以这两个命令的执行状态是独立的，没有真正的先后关系。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 文中不是已经解释了？当然会执行，不管entrypoint。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-24 09:07:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83ercmNryEicqDS73icpUu7W0BnZ7ZIia6jR7kdVMIzH0q1d7L8EKAYWeTJcribibGcHnJzpsjRFxAe26egQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>pytimer</span>
  </div>
  <div class="_2_QraFYR_0">老师，问一下，我看pod.status.phase是running，但是Ready是false，如果我想判断pod状态，要怎么做</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 看events</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-24 08:22:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/ed/1a/ce7f7d54.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>asdf100</span>
  </div>
  <div class="_2_QraFYR_0">本地测试的进入shell容器后执行ps ax，跟例子的结果不一样。没有看到nginx窗口里的nginx进程信息，为什么，我看也有人遇到这个问题.<br>&#47; # ps<br>PID USER TIME COMMAND<br>1 root 0:00 sh<br>10 root 0:00 ps</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你看看sharepid功能开启了没</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-30 10:55:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/7f/a2/8ccf5c85.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>XThundering</span>
  </div>
  <div class="_2_QraFYR_0">文章中有处有问题：&quot;ImagePullPolicy 的值默认是 Always，即...&quot; 这部分和官方文档与实际情况不一致。<br>在官方文档中提到&quot;The default pull policy is IfNotPresent&quot;，我这边在使用中发现的也是这样子的~<br>附一下官方文档链接：https:&#47;&#47;kubernetes.io&#47;docs&#47;concepts&#47;containers&#47;images&#47;</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 最新版本才改的呢</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-17 11:07:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/ea/88/914d0c1b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>短笛</span>
  </div>
  <div class="_2_QraFYR_0">Pod 的意思我理解应该是指 a small herd or school of marine animals, especially whales 而不是豆荚，为什么是鲸群呢？因为 Docker 的 Logo 啊 😂</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-21 16:23:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://wx.qlogo.cn/mmopen/vi_32/eQysJPqzuAoyrWzWyXyYibrAszp6B4WUic7g2TnO8ksfWtCibOEMRqQmCZk4xJwEPQwTRH21W4e2f77FoQTzL6Dhw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>钱涛</span>
  </div>
  <div class="_2_QraFYR_0">Pod 启动设置NodeSelector 是如果设置的Node不存在, 可以看到Pod会被调度，但是永远不会成功，但是启动失败后又没有对应的Pod,导致日志没法describe. 请问怎么找对应的日志呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-10 16:08:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/ff/ad/5020a8c5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Farewell丶</span>
  </div>
  <div class="_2_QraFYR_0">官方文档对 imagePullPolicy 属性的默认情况是这样描述的，好像和文章说的不太一样：<br>“ Defaults to Always if :latest tag is specified, or IfNotPresent otherwise. ”</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-25 17:55:29</div>
  </div>
</div>
</div>
</li>
</ul>