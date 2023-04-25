<audio title="13 _ 为什么我们需要Pod？" src="https://static001.geekbang.org/resource/audio/a6/b6/a6fbf30060d0a184da6ad973f24e88b6.mp3" controls="controls"></audio> 
<p>你好，我是张磊。今天我和你分享的主题是：为什么我们需要Pod。</p><p>在前面的文章中，我详细介绍了在Kubernetes里部署一个应用的过程。在这些讲解中，我提到了这样一个知识点：Pod，是Kubernetes项目中最小的API对象。如果换一个更专业的说法，我们可以这样描述：Pod，是Kubernetes项目的原子调度单位。</p><p>不过，我相信你在学习和使用Kubernetes项目的过程中，已经不止一次地想要问这样一个问题：<span class="orange">为什么我们会需要Pod？</span></p><p>是啊，我们在前面已经花了很多精力去解读Linux容器的原理、分析了Docker容器的本质，终于，“Namespace做隔离，Cgroups做限制，rootfs做文件系统”这样的“三句箴言”可以朗朗上口了，<strong>为什么Kubernetes项目又突然搞出一个Pod来呢？</strong></p><p>要回答这个问题，我们还是要一起回忆一下我曾经反复强调的一个问题：容器的本质到底是什么？</p><p>你现在应该可以不假思索地回答出来：容器的本质是进程。</p><p>没错。容器，就是未来云计算系统中的进程；容器镜像就是这个系统里的“.exe”安装包。那么Kubernetes呢？</p><p>你应该也能立刻回答上来：Kubernetes就是操作系统！</p><!-- [[[read_end]]] --><p>非常正确。</p><p>现在，就让我们登录到一台Linux机器里，执行一条如下所示的命令：</p><pre><code>$ pstree -g
</code></pre><p>这条命令的作用，是展示当前系统中正在运行的进程的树状结构。它的返回结果如下所示：</p><pre><code>systemd(1)-+-accounts-daemon(1984)-+-{gdbus}(1984)
           | `-{gmain}(1984)
           |-acpid(2044)
          ...      
           |-lxcfs(1936)-+-{lxcfs}(1936)
           | `-{lxcfs}(1936)
           |-mdadm(2135)
           |-ntpd(2358)
           |-polkitd(2128)-+-{gdbus}(2128)
           | `-{gmain}(2128)
           |-rsyslogd(1632)-+-{in:imklog}(1632)
           |  |-{in:imuxsock) S 1(1632)
           | `-{rs:main Q:Reg}(1632)
           |-snapd(1942)-+-{snapd}(1942)
           |  |-{snapd}(1942)
           |  |-{snapd}(1942)
           |  |-{snapd}(1942)
           |  |-{snapd}(1942)
</code></pre><p>不难发现，在一个真正的操作系统里，进程并不是“孤苦伶仃”地独自运行的，而是以进程组的方式，“有原则地”组织在一起。比如，这里有一个叫作rsyslogd的程序，它负责的是Linux操作系统里的日志处理。可以看到，rsyslogd的主程序main，和它要用到的内核日志模块imklog等，同属于1632进程组。这些进程相互协作，共同完成rsyslogd程序的职责。</p><blockquote>
<p>注意：我在本篇中提到的“进程”，比如，rsyslogd对应的imklog，imuxsock和main，严格意义上来说，其实是Linux 操作系统语境下的“线程”。这些线程，或者说，轻量级进程之间，可以共享文件、信号、数据内存、甚至部分代码，从而紧密协作共同完成一个程序的职责。所以同理，我提到的“进程组”，对应的也是 Linux 操作系统语境下的“线程组”。这种命名关系与实际情况的不一致，是Linux 发展历史中的一个遗留问题。对这个话题感兴趣的同学，可以阅读<a href="https://www.ibm.com/developerworks/cn/linux/kernel/l-thread/index.html">这篇技术文章</a>来了解一下。</p>
</blockquote><p>而Kubernetes项目所做的，其实就是将“进程组”的概念映射到了容器技术中，并使其成为了这个云计算“操作系统”里的“一等公民”。</p><p>Kubernetes项目之所以要这么做的原因，我在前面介绍Kubernetes和Borg的关系时曾经提到过：在Borg项目的开发和实践过程中，Google公司的工程师们发现，他们部署的应用，往往都存在着类似于“进程和进程组”的关系。更具体地说，就是这些应用之间有着密切的协作关系，使得它们必须部署在同一台机器上。</p><p>而如果事先没有“组”的概念，像这样的运维关系就会非常难以处理。</p><p>我还是以前面的rsyslogd为例子。已知rsyslogd由三个进程组成：一个imklog模块，一个imuxsock模块，一个rsyslogd自己的main函数主进程。这三个进程一定要运行在同一台机器上，否则，它们之间基于Socket的通信和文件交换，都会出现问题。</p><p>现在，我要把rsyslogd这个应用给容器化，由于受限于容器的“单进程模型”，这三个模块必须被分别制作成三个不同的容器。而在这三个容器运行的时候，它们设置的内存配额都是1 GB。</p><blockquote>
<p>再次强调一下：容器的“单进程模型”，并不是指容器里只能运行“一个”进程，而是指容器没有管理多个进程的能力。这是因为容器里PID=1的进程就是应用本身，其他的进程都是这个PID=1进程的子进程。可是，用户编写的应用，并不能够像正常操作系统里的init进程或者systemd那样拥有进程管理的功能。比如，你的应用是一个Java Web程序（PID=1），然后你执行docker exec在后台启动了一个Nginx进程（PID=3）。可是，当这个Nginx进程异常退出的时候，你该怎么知道呢？这个进程退出后的垃圾收集工作，又应该由谁去做呢？</p>
</blockquote><p>假设我们的Kubernetes集群上有两个节点：node-1上有3 GB可用内存，node-2有2.5 GB可用内存。</p><p>这时，假设我要用Docker Swarm来运行这个rsyslogd程序。为了能够让这三个容器都运行在同一台机器上，我就必须在另外两个容器上设置一个affinity=main（与main容器有亲密性）的约束，即：它们俩必须和main容器运行在同一台机器上。</p><p>然后，我顺序执行：“docker run main”“docker run imklog”和“docker run imuxsock”，创建这三个容器。</p><p>这样，这三个容器都会进入Swarm的待调度队列。然后，main容器和imklog容器都先后出队并被调度到了node-2上（这个情况是完全有可能的）。</p><p>可是，当imuxsock容器出队开始被调度时，Swarm就有点懵了：node-2上的可用资源只有0.5 GB了，并不足以运行imuxsock容器；可是，根据affinity=main的约束，imuxsock容器又只能运行在node-2上。</p><p>这就是一个典型的成组调度（gang scheduling）没有被妥善处理的例子。</p><p>在工业界和学术界，关于这个问题的讨论可谓旷日持久，也产生了很多可供选择的解决方案。</p><p>比如，Mesos中就有一个资源囤积（resource hoarding）的机制，会在所有设置了Affinity约束的任务都达到时，才开始对它们统一进行调度。而在Google Omega论文中，则提出了使用乐观调度处理冲突的方法，即：先不管这些冲突，而是通过精心设计的回滚机制在出现了冲突之后解决问题。</p><p>可是这些方法都谈不上完美。资源囤积带来了不可避免的调度效率损失和死锁的可能性；而乐观调度的复杂程度，则不是常规技术团队所能驾驭的。</p><p>但是，到了Kubernetes项目里，这样的问题就迎刃而解了：Pod是Kubernetes里的原子调度单位。这就意味着，Kubernetes项目的调度器，是统一按照Pod而非容器的资源需求进行计算的。</p><p>所以，像imklog、imuxsock和main函数主进程这样的三个容器，正是一个典型的由三个容器组成的Pod。Kubernetes项目在调度时，自然就会去选择可用内存等于3 GB的node-1节点进行绑定，而根本不会考虑node-2。</p><p>像这样容器间的紧密协作，我们可以称为“超亲密关系”。这些具有“超亲密关系”容器的典型特征包括但不限于：互相之间会发生直接的文件交换、使用localhost或者Socket文件进行本地通信、会发生非常频繁的远程调用、需要共享某些Linux Namespace（比如，一个容器要加入另一个容器的Network Namespace）等等。</p><p>这也就意味着，并不是所有有“关系”的容器都属于同一个Pod。比如，PHP应用容器和MySQL虽然会发生访问关系，但并没有必要、也不应该部署在同一台机器上，它们更适合做成两个Pod。</p><p>不过，相信此时你可能会有<strong>第二个疑问：</strong></p><p>对于初学者来说，一般都是先学会了用Docker这种单容器的工具，才会开始接触Pod。</p><p>而如果Pod的设计只是出于调度上的考虑，那么Kubernetes项目似乎完全没有必要非得把Pod作为“一等公民”吧？这不是故意增加用户的学习门槛吗？</p><p>没错，如果只是处理“超亲密关系”这样的调度问题，有Borg和Omega论文珠玉在前，Kubernetes项目肯定可以在调度器层面给它解决掉。</p><p>不过，Pod在Kubernetes项目里还有更重要的意义，那就是：<strong>容器设计模式</strong>。</p><p>为了理解这一层含义，我就必须先给你介绍一下<span class="orange">Pod的实现原理。</span></p><p><strong>首先，关于Pod最重要的一个事实是：它只是一个逻辑概念。</strong></p><p>也就是说，Kubernetes真正处理的，还是宿主机操作系统上Linux容器的Namespace和Cgroups，而并不存在一个所谓的Pod的边界或者隔离环境。</p><p>那么，Pod又是怎么被“创建”出来的呢？</p><p>答案是：Pod，其实是一组共享了某些资源的容器。</p><p>具体的说：<strong>Pod里的所有容器，共享的是同一个Network Namespace，并且可以声明共享同一个Volume。</strong></p><p>那这么来看的话，一个有A、B两个容器的Pod，不就是等同于一个容器（容器A）共享另外一个容器（容器B）的网络和Volume的玩儿法么？</p><p>这好像通过docker run --net --volumes-from这样的命令就能实现嘛，比如：</p><pre><code>$ docker run --net=B --volumes-from=B --name=A image-A ...
</code></pre><p>但是，你有没有考虑过，如果真这样做的话，容器B就必须比容器A先启动，这样一个Pod里的多个容器就不是对等关系，而是拓扑关系了。</p><p>所以，在Kubernetes项目里，Pod的实现需要使用一个中间容器，这个容器叫作Infra容器。在这个Pod中，Infra容器永远都是第一个被创建的容器，而其他用户定义的容器，则通过Join Network Namespace的方式，与Infra容器关联在一起。这样的组织关系，可以用下面这样一个示意图来表达：</p><p><img src="https://static001.geekbang.org/resource/image/8c/cf/8c016391b4b17923f38547c498e434cf.png?wh=490*665" alt=""><br>
如上图所示，这个Pod里有两个用户容器A和B，还有一个Infra容器。很容易理解，在Kubernetes项目里，Infra容器一定要占用极少的资源，所以它使用的是一个非常特殊的镜像，叫作：<code>k8s.gcr.io/pause</code>。这个镜像是一个用汇编语言编写的、永远处于“暂停”状态的容器，解压后的大小也只有100~200 KB左右。</p><p>而在Infra容器“Hold住”Network Namespace后，用户容器就可以加入到Infra容器的Network Namespace当中了。所以，如果你查看这些容器在宿主机上的Namespace文件（这个Namespace文件的路径，我已经在前面的内容中介绍过），它们指向的值一定是完全一样的。</p><p>这也就意味着，对于Pod里的容器A和容器B来说：</p><ul>
<li>它们可以直接使用localhost进行通信；</li>
<li>它们看到的网络设备跟Infra容器看到的完全一样；</li>
<li>一个Pod只有一个IP地址，也就是这个Pod的Network Namespace对应的IP地址；</li>
<li>当然，其他的所有网络资源，都是一个Pod一份，并且被该Pod中的所有容器共享；</li>
<li>Pod的生命周期只跟Infra容器一致，而与容器A和B无关。</li>
</ul><p>而对于同一个Pod里面的所有用户容器来说，它们的进出流量，也可以认为都是通过Infra容器完成的。这一点很重要，因为<strong>将来如果你要为Kubernetes开发一个网络插件时，应该重点考虑的是如何配置这个Pod的Network Namespace，而不是每一个用户容器如何使用你的网络配置，这是没有意义的。</strong></p><p>这就意味着，如果你的网络插件需要在容器里安装某些包或者配置才能完成的话，是不可取的：Infra容器镜像的rootfs里几乎什么都没有，没有你随意发挥的空间。当然，这同时也意味着你的网络插件完全不必关心用户容器的启动与否，而只需要关注如何配置Pod，也就是Infra容器的Network Namespace即可。</p><p>有了这个设计之后，共享Volume就简单多了：Kubernetes项目只要把所有Volume的定义都设计在Pod层级即可。</p><p>这样，一个Volume对应的宿主机目录对于Pod来说就只有一个，Pod里的容器只要声明挂载这个Volume，就一定可以共享这个Volume对应的宿主机目录。比如下面这个例子：</p><pre><code>apiVersion: v1
kind: Pod
metadata:
  name: two-containers
spec:
  restartPolicy: Never
  volumes:
  - name: shared-data
    hostPath:      
      path: /data
  containers:
  - name: nginx-container
    image: nginx
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html
  - name: debian-container
    image: debian
    volumeMounts:
    - name: shared-data
      mountPath: /pod-data
    command: [&quot;/bin/sh&quot;]
    args: [&quot;-c&quot;, &quot;echo Hello from the debian container &gt; /pod-data/index.html&quot;]
</code></pre><p>在这个例子中，debian-container和nginx-container都声明挂载了shared-data这个Volume。而shared-data是hostPath类型。所以，它对应在宿主机上的目录就是：/data。而这个目录，其实就被同时绑定挂载进了上述两个容器当中。</p><p>这就是为什么，nginx-container可以从它的/usr/share/nginx/html目录中，读取到debian-container生成的index.html文件的原因。</p><p><span class="orange">明白了Pod的实现原理后，我们再来讨论“容器设计模式”</span>，就容易多了。</p><p>Pod这种“超亲密关系”容器的设计思想，实际上就是希望，当用户想在一个容器里跑多个功能并不相关的应用时，应该优先考虑它们是不是更应该被描述成一个Pod里的多个容器。</p><p>为了能够掌握这种思考方式，你就应该尽量尝试使用它来描述一些用单个容器难以解决的问题。</p><p><strong>第一个最典型的例子是：WAR包与Web服务器。</strong></p><p>我们现在有一个Java Web应用的WAR包，它需要被放在Tomcat的webapps目录下运行起来。</p><p>假如，你现在只能用Docker来做这件事情，那该如何处理这个组合关系呢？</p><ul>
<li>一种方法是，把WAR包直接放在Tomcat镜像的webapps目录下，做成一个新的镜像运行起来。可是，这时候，如果你要更新WAR包的内容，或者要升级Tomcat镜像，就要重新制作一个新的发布镜像，非常麻烦。</li>
<li>另一种方法是，你压根儿不管WAR包，永远只发布一个Tomcat容器。不过，这个容器的webapps目录，就必须声明一个hostPath类型的Volume，从而把宿主机上的WAR包挂载进Tomcat容器当中运行起来。不过，这样你就必须要解决一个问题，即：如何让每一台宿主机，都预先准备好这个存储有WAR包的目录呢？这样来看，你只能独立维护一套分布式存储系统了。</li>
</ul><p>实际上，有了Pod之后，这样的问题就很容易解决了。我们可以把WAR包和Tomcat分别做成镜像，然后把它们作为一个Pod里的两个容器“组合”在一起。这个Pod的配置文件如下所示：</p><pre><code>apiVersion: v1
kind: Pod
metadata:
  name: javaweb-2
spec:
  initContainers:
  - image: geektime/sample:v2
    name: war
    command: [&quot;cp&quot;, &quot;/sample.war&quot;, &quot;/app&quot;]
    volumeMounts:
    - mountPath: /app
      name: app-volume
  containers:
  - image: geektime/tomcat:7.0
    name: tomcat
    command: [&quot;sh&quot;,&quot;-c&quot;,&quot;/root/apache-tomcat-7.0.42-v2/bin/start.sh&quot;]
    volumeMounts:
    - mountPath: /root/apache-tomcat-7.0.42-v2/webapps
      name: app-volume
    ports:
    - containerPort: 8080
      hostPort: 8001 
  volumes:
  - name: app-volume
    emptyDir: {}
</code></pre><p>在这个Pod中，我们定义了两个容器，第一个容器使用的镜像是geektime/sample:v2，这个镜像里只有一个WAR包（sample.war）放在根目录下。而第二个容器则使用的是一个标准的Tomcat镜像。</p><p>不过，你可能已经注意到，WAR包容器的类型不再是一个普通容器，而是一个Init Container类型的容器。</p><p>在Pod中，所有Init Container定义的容器，都会比spec.containers定义的用户容器先启动。并且，Init Container容器会按顺序逐一启动，而直到它们都启动并且退出了，用户容器才会启动。</p><p>所以，这个Init Container类型的WAR包容器启动后，我执行了一句"cp /sample.war /app"，把应用的WAR包拷贝到/app目录下，然后退出。</p><p>而后这个/app目录，就挂载了一个名叫app-volume的Volume。</p><p>接下来就很关键了。Tomcat容器，同样声明了挂载app-volume到自己的webapps目录下。</p><p>所以，等Tomcat容器启动时，它的webapps目录下就一定会存在sample.war文件：这个文件正是WAR包容器启动时拷贝到这个Volume里面的，而这个Volume是被这两个容器共享的。</p><p>像这样，我们就用一种“组合”方式，解决了WAR包与Tomcat容器之间耦合关系的问题。</p><p>实际上，这个所谓的“组合”操作，正是容器设计模式里最常用的一种模式，它的名字叫：sidecar。</p><p>顾名思义，sidecar指的就是我们可以在一个Pod中，启动一个辅助容器，来完成一些独立于主进程（主容器）之外的工作。</p><p>比如，在我们的这个应用Pod中，Tomcat容器是我们要使用的主容器，而WAR包容器的存在，只是为了给它提供一个WAR包而已。所以，我们用Init Container的方式优先运行WAR包容器，扮演了一个sidecar的角色。</p><p><strong>第二个例子，则是容器的日志收集。</strong></p><p>比如，我现在有一个应用，需要不断地把日志文件输出到容器的/var/log目录中。</p><p>这时，我就可以把一个Pod里的Volume挂载到应用容器的/var/log目录上。</p><p>然后，我在这个Pod里同时运行一个sidecar容器，它也声明挂载同一个Volume到自己的/var/log目录上。</p><p>这样，接下来sidecar容器就只需要做一件事儿，那就是不断地从自己的/var/log目录里读取日志文件，转发到MongoDB或者Elasticsearch中存储起来。这样，一个最基本的日志收集工作就完成了。</p><p>跟第一个例子一样，这个例子中的sidecar的主要工作也是使用共享的Volume来完成对文件的操作。</p><p>但不要忘记，Pod的另一个重要特性是，它的所有容器都共享同一个Network Namespace。这就使得很多与Pod网络相关的配置和管理，也都可以交给sidecar完成，而完全无须干涉用户容器。这里最典型的例子莫过于Istio这个微服务治理项目了。</p><p>Istio项目使用sidecar容器完成微服务治理的原理，我在后面很快会讲解到。</p><blockquote>
<p>备注：Kubernetes社区曾经把“容器设计模式”这个理论，整理成了<a href="https://www.usenix.org/conference/hotcloud16/workshop-program/presentation/burns">一篇小论文</a>，你可以点击链接浏览。</p>
</blockquote><h2>总结</h2><p>在本篇文章中我重点分享了Kubernetes项目中Pod的实现原理。</p><p>Pod是Kubernetes项目与其他单容器项目相比最大的不同，也是一位容器技术初学者需要面对的第一个与常规认知不一致的知识点。</p><p>事实上，直到现在，仍有很多人把容器跟虚拟机相提并论，他们把容器当做性能更好的虚拟机，喜欢讨论如何把应用从虚拟机无缝地迁移到容器中。</p><p>但实际上，无论是从具体的实现原理，还是从使用方法、特性、功能等方面，容器与虚拟机几乎没有任何相似的地方；也不存在一种普遍的方法，能够把虚拟机里的应用无缝迁移到容器中。因为，容器的性能优势，必然伴随着相应缺陷，即：它不能像虚拟机那样，完全模拟本地物理机环境中的部署方法。</p><p>所以，这个“上云”工作的完成，最终还是要靠深入理解容器的本质，即：进程。</p><p>实际上，一个运行在虚拟机里的应用，哪怕再简单，也是被管理在systemd或者supervisord之下的<strong>一组进程，而不是一个进程</strong>。这跟本地物理机上应用的运行方式其实是一样的。这也是为什么，从物理机到虚拟机之间的应用迁移，往往并不困难。</p><p>可是对于容器来说，一个容器永远只能管理一个进程。更确切地说，一个容器，就是一个进程。这是容器技术的“天性”，不可能被修改。所以，将一个原本运行在虚拟机里的应用，“无缝迁移”到容器中的想法，实际上跟容器的本质是相悖的。</p><p>这也是当初Swarm项目无法成长起来的重要原因之一：一旦到了真正的生产环境上，Swarm这种单容器的工作方式，就难以描述真实世界里复杂的应用架构了。</p><p>所以，你现在可以这么理解Pod的本质：</p><blockquote>
<p>Pod，实际上是在扮演传统基础设施里“虚拟机”的角色；而容器，则是这个虚拟机里运行的用户程序。</p>
</blockquote><p>所以下一次，当你需要把一个运行在虚拟机里的应用迁移到Docker容器中时，一定要仔细分析到底有哪些进程（组件）运行在这个虚拟机里。</p><p>然后，你就可以把整个虚拟机想象成为一个Pod，把这些进程分别做成容器镜像，把有顺序关系的容器，定义为Init Container。这才是更加合理的、松耦合的容器编排诀窍，也是从传统应用架构，到“微服务架构”最自然的过渡方式。</p><blockquote>
<p>注意：Pod这个概念，提供的是一种编排思想，而不是具体的技术方案。所以，如果愿意的话，你完全可以使用虚拟机来作为Pod的实现，然后把用户容器都运行在这个虚拟机里。比如，Mirantis公司的<a href="https://github.com/Mirantis/virtlet">virtlet项目</a>就在干这个事情。甚至，你可以去实现一个带有Init进程的容器项目，来模拟传统应用的运行方式。这些工作，在Kubernetes中都是非常轻松的，也是我们后面讲解CRI时会提到的内容。</p>
</blockquote><p>相反的，如果强行把整个应用塞到一个容器里，甚至不惜使用Docker In Docker这种在生产环境中后患无穷的解决方案，恐怕最后往往会得不偿失。</p><h2>思考题</h2><p>除了Network Namespace外，Pod里的容器还可以共享哪些Namespace呢？你能说出共享这些Namesapce的具体应用场景吗？</p><p>感谢你的收听，欢迎你给我留言，也欢迎分享给更多的朋友一起阅读。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/6c/c9/abb7bfe3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>段帅民</span>
  </div>
  <div class="_2_QraFYR_0">这文章读起来像吃脆苹果，爽，这是我订阅专栏中写的最好的，没有之一</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这是我见过最形象的评论……</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-21 09:00:48</div>
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
  <div class="_2_QraFYR_0">pod是一个小家庭，它把密不可分的家庭成员(container)聚在一起，Infra container则是家长，掌管家中共通资源，家庭成员通过sidecar方式互帮互助，其乐融融～</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-30 16:18:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/7c/b2/d2f03436.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Chaos</span>
  </div>
  <div class="_2_QraFYR_0">感谢张磊，这是本人看过解释 Pod 以及为何要有 Pod 这个设计最好的文章（看过估计20篇，这是第一篇中文的）。<br><br>云计算系统的操作系统是 k8s ，容器就相当于是其进程，而 Pod 则是进程组。这样解释给有 Linux 运维底子的人听，基本上一下子就明白过来了。赞！但容器镜像是「 .exe 」安装包似乎有点不妥，个人觉得是二进制可执行安装包似乎更恰当和通用一些。<br><br>Pod 的另一个重要意义，容器设计模式，通过这篇里有了更深的认知，在迁移传统基于虚拟机的应用时起到了关键作用。<br><br>其实 k8s.gcr.io&#47;pause 是前不久读了好几遍（因为讲的比较生涩） Ian Lewis 的 The Almighty Pause Container 这篇才基本明白原理。再读这篇算是巩固和解惑了。<br><br>「而后这个 &#47;app 目录，就挂载了一个名叫 app-volume 的 Volume 」惭愧，作为有18年实践经验的 Linux 老司机居然也愣了几十秒，继续往下看解释后才恍然大悟 volume 和 mountPath 是 bind mount （建议附注一下是绑定挂载）。<br><br>最后的总结也非常棒，「 Pod 实际上是在扮演传统基础设施里&quot;虚拟机&quot;的角色；而容器则是这个虚拟机里的应用程序」。要完成传统基于虚拟机的应用到微服务架构的迁移，核心思想是：分析应用组成（组件、进程），将其拆分成松耦合的容器（以容器镜像方式分发），利用 Init Container 来解决顺序和依赖关系。<br><br>这个课程，光这一篇就值回票价了。居然还附带学习到了 Docker Swarm 的局限性。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-05 20:55:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/18/34/c082419c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>风轨</span>
  </div>
  <div class="_2_QraFYR_0">刚上网查了，Linux 支持7种namespace:<br>1. cgroup用于隔离cgroup根目录;<br>2. IPC用于隔离系统消息队列;<br>3. Network隔离网络;<br>4. Mount隔离挂载点;<br>5. PID隔离进程;<br>6. User隔离用户和用户组;<br>7. UTS隔离主机名nis域名。<br><br>pod里网络和文件系统应该是用得最多的，其他的应用程序基本不用吧？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-21 09:50:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/9a/43/d6291e76.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Q宝的宝</span>
  </div>
  <div class="_2_QraFYR_0">痛快，每一篇都鞭辟入里。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-21 01:02:20</div>
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
  <div class="_2_QraFYR_0">看得太爽了。请教一个问题：<br>war和tamcat这种情况。<br>如果把war独立出来做一个镜像的话，应该用什么做基础镜像呢？<br>我现在做镜像的时候通常都是用debian做基础镜像，但如果只是为了复制这个.war包的话，用debian感觉蛮浪费的。应该怎样做到最小呢，而且要支持cp命令。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 市面上的小镜像多的很啊，busybox，alpine</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-24 00:07:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/de/af/9379c7bd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Racoon</span>
  </div>
  <div class="_2_QraFYR_0">k8s.gcr.io&#47;pause<br>在node节点上一直看到有这个镜像，搞不懂是干嘛的<br>如今一看 豁然开朗</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-09 17:13:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/5e/61/985f3eb7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>songyy</span>
  </div>
  <div class="_2_QraFYR_0">为什么说“Docker in Docker”这种方式在生产环境后患无穷呀？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 因为是坑的二次方</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-16 17:59:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/72/3e/534db55d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>huan</span>
  </div>
  <div class="_2_QraFYR_0">重要理解了这个核心概念，透透透！不但解决了我前几天war和tomcat做一起，完成镜像的频繁打包问题，而且还想到怎么用puppeteer做一个init container,然后注入到nodejs主容器，从而解决chrome headless太大不容易安装的问题，爽爽爽！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-21 09:08:20</div>
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
  <div class="_2_QraFYR_0">讲的真的用心，水平也高。被圈粉，希望一直能出专栏。您一定有一票粉丝支持和跟随的！继续加油。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-21 08:34:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/00/c1/38eb4289.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>王力国</span>
  </div>
  <div class="_2_QraFYR_0">如果写成一本书，一定会是最畅销的 k8s 书籍</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-26 21:50:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/16/77/30e059c8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>kindnull</span>
  </div>
  <div class="_2_QraFYR_0">从事Linux运维工作多年，有一点一直有点不明白，这里到底谁挂载到谁上面：<br>volumeMounts：<br>      -mountpath: &#47;app<br>        name: app-volume<br><br>文中有有这句话说&quot;而后这个 &#47;app 目录，就挂载了一个名叫 app-volume的Volume&quot;<br>这里是说将app-volume挂载到&#47;app上，但是app是个目录，那我想问的是app-volume是个目录还是设备？<br>同样下面：<br>volumeMounts:<br>    -mountpath:&#47;root&#47;apache-tomcat-7.0.42-v2&#47;webapps<br>       name:app-volume<br>---Tomecat容器同样声明了挂载app-volume到自己的webapps目录下<br><br>这里又说把app-volume挂载到webapps下，webapps明显是个目录，那app-volume是个目录吗？<br><br>但是我清楚记得在Linux下。如果要挂载一个分区设备到一个目录或者一个目录到另外一个主机目录下应该是：<br>mount  -t xxfs  src_dir   dest_dir<br>比如： mount  -t xfs &#47;dev&#47;sda   &#47;opt&#47;app<br>           mount -t nfs &#47;share&#47;data  192.168.0.100:&#47;data<br>上面的挂载的app-volume到底是设备还是文件，或者是去的一个别名？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: volume的挂载是bind mount，不是设备，它的功能就是把文件或目录绑定挂载在一起，所以你这里的纠结谁在谁上面是没有意义的……唯一需要明确的是，bind mount的挂载点，是容器volume在宿主机上的目录。这一块在容器基础部分有详细的解释。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-22 14:38:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/14/ee/d72a8222.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>攻城拔寨</span>
  </div>
  <div class="_2_QraFYR_0">war包跟tomcat这个栗子举得恰到好处啊</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-24 18:44:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/3a/43/6abf3b7d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>刘榴</span>
  </div>
  <div class="_2_QraFYR_0">好专栏啊，再写成书继续支持</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-21 08:28:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/f6/8e/c9c94420.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>俊釆</span>
  </div>
  <div class="_2_QraFYR_0">毫不夸张的讲，这是我在极客时间买的最划算的专栏，没有之一。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-18 10:56:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/PUiby8MibibKcMd88OtDq1c0myEILZjap46fyiaOlML0UlNWzj9NTIEXOhXCCR1tcUibG0I6UoGp59Zj8H5EYwzkY9g/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>fldhmily63319</span>
  </div>
  <div class="_2_QraFYR_0">重问： 关于升级war和tomcat那块，也是先修改yml？然后kubenertes执行升级命令，pod会重新启动，生产也是按照这种方式吗？<br><br>所以这种情况下面，如果只是升级个war包，或者加一个新的war包，tomcat也要重新启动？这就不是完全松耦合了？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: kubernetes的更新是patch方式的，可以关注后续声明式api的内容。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-21 20:09:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/9c/96/eeb09aea.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>草原之巅</span>
  </div>
  <div class="_2_QraFYR_0">张磊真牛逼！ 堪称高手中的高手！ </div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-20 17:08:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/05/92/b609f7e3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>骨汤鸡蛋面</span>
  </div>
  <div class="_2_QraFYR_0">张老师，我们公司一直有一个问题： 我们用docker 管理测试环境，对于springboot 项目，一向是java -jar xx.jar 的方式启动，但因为是测试环境，经常代码本身有问题导致java（也就是容器主进程）启动不起来，进而触发健康检查重新调度该服务，然后开发总是抱怨看不到事故现场。我打算自己实现一个程序（我们称为nile），容器启动时先启动nile确保容器可以启动起来，再由nile启动java进程。这时还可以让nile 读取用户配置 自定义设置jvm 参数、nile向zk汇报一些应用情况等。这个做法呢，nile可以算是java 的sidecar，按您文章的说法，nile和java 是拓扑关系而不是对等关系，这个时候我一个pod里分别是nile和java容器是否可能呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果nile不负责管理java进程的话当然可以。否则的话就是同一个容器。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-04 17:25:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/a9/2e/01b2839e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>巩夫建</span>
  </div>
  <div class="_2_QraFYR_0">非常棒，释义了pod的理念。有个小问题，像我之前一个docker中跑tomcat和nginx两个进程，共用同一个文件，如果拆成三部分，第一init container 文件镜像，第二 tomcat镜像，第三 nginx镜像，如何保证init container 启动后启动tomcat，最后启动nginx，还有这种优点有什么?目前只想到镜像高复用，谢谢作者。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: nginx容器启动前做检查即可。主要好处就是解偶。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-21 19:03:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/rUPjvMdUGic56wNdVm0be85REaB7nLwyEC7HiamhmeWsQichtADA3IsFubIqOXMjlsEtmicdpDRC2YhNWVrpPKj1lw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>姜子牙</span>
  </div>
  <div class="_2_QraFYR_0">容器的“单进程模型”，并不是指容器里只能运行“...<br>这一段里说容器的单进程模型。我在跑一个web后端服务，springboot实现。 entrypoint 是java -jar xxxx.jar  .这条命令产生的进程就是PID=1的吗？ 容器只能管理PID=1的。我理解的就是这个进程执行不成功，容器户退出。但是如果我的springboot程序有问题，没跑成功，容器依然跑着呢，为啥?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 因为有JVM</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-22 16:17:17</div>
  </div>
</div>
</div>
</li>
</ul>