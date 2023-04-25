<audio title="12｜Pod：如何理解这个Kubernetes里最核心的概念？" src="https://static001.geekbang.org/resource/audio/d6/ed/d6e6614d39acfa54acda54c6b54f51ed.mp3" controls="controls"></audio> 
<p>你好，我是Chrono。</p><p>前两天我们学习了Kubernetes世界里的工作语言YAML，还编写了一个简短的YAML文件，描述了一个API对象：Pod，它在spec字段里包含了容器的定义。</p><p>那么为什么Kubernetes不直接使用已经非常成熟稳定的容器？为什么要再单独抽象出一个Pod对象？为什么几乎所有人都说Pod是Kubernetes里最核心最基本的概念呢？</p><p>今天我就来逐一解答这些问题，希望你学完今天的这次课，心里面能够有明确的答案。</p><h2>为什么要有Pod</h2><p>Pod这个词原意是“豌豆荚”，后来又延伸出“舱室”“太空舱”等含义，你可以看一下这张图片，形象地来说Pod就是包含了很多组件、成员的一种结构。</p><p><img src="https://static001.geekbang.org/resource/image/06/ba/0608d5d450c503c4102af27518d15bba.png?wh=800x533" alt="图片" title="图片来自网络"></p><p>容器技术我想你现在已经比较熟悉了，它让进程在一个“沙盒”环境里运行，具有良好的隔离性，对应用是一个非常好的封装。</p><p>不过，当容器技术进入到现实的生产环境中时，这种隔离性就带来了一些麻烦。因为很少有应用是完全独立运行的，经常需要几个进程互相协作才能完成任务，比如在“入门篇”里我们搭建WordPress网站的时候，就需要Nginx、WordPress、MariaDB三个容器一起工作。</p><p>WordPress例子里的这三个应用之间的关系还是比较松散的，它们可以分别调度，运行在不同的机器上也能够以IP地址通信。</p><!-- [[[read_end]]] --><p>但还有一些特殊情况，多个应用结合得非常紧密以至于无法把它们拆开。比如，有的应用运行前需要其他应用帮它初始化一些配置，还有就是日志代理，它必须读取另一个应用存储在本地磁盘的文件再转发出去。这些应用如果被强制分离成两个容器，切断联系，就无法正常工作了。</p><p>那么把这些应用都放在一个容器里运行可不可以呢？</p><p>当然可以，但这并不是一种好的做法。因为容器的理念是对应用的独立封装，它里面就应该是一个进程、一个应用，如果里面有多个应用，不仅违背了容器的初衷，也会让容器更难以管理。</p><p><strong>为了解决这样多应用联合运行的问题，同时还要不破坏容器的隔离，就需要在容器外面再建立一个“收纳舱”</strong>，让多个容器既保持相对独立，又能够小范围共享网络、存储等资源，而且永远是“绑在一起”的状态。</p><p>所以，Pod的概念也就呼之欲出了，容器正是“豆荚”里那些小小的“豌豆”，你可以在Pod的YAML里看到，“spec.containers”字段其实是一个数组，里面允许定义多个容器。</p><p>如果再拿之前讲过的“小板房”来比喻的话，Pod就是由客厅、卧室、厨房等预制房间拼装成的一个齐全的生活环境，不仅同样具备易于拆装易于搬迁的优点，而且要比单独的“一居室”功能强大得多，能够让进程“住”得更舒服。</p><h2>为什么Pod是Kubernetes的核心对象</h2><p>因为Pod是对容器的“打包”，里面的容器是一个整体，总是能够一起调度、一起运行，绝不会出现分离的情况，而且Pod属于Kubernetes，可以在不触碰下层容器的情况下任意定制修改。所以有了Pod这个抽象概念，Kubernetes在集群级别上管理应用就会“得心应手”了。</p><p>Kubernetes让Pod去编排处理容器，然后把Pod作为应用调度部署的<strong>最小单位</strong>，Pod也因此成为了Kubernetes世界里的“原子”（当然这个“原子”内部是有结构的，不是铁板一块），基于Pod就可以构建出更多更复杂的业务形态了。</p><p>下面的这张图你也许在其他资料里见过，它从Pod开始，扩展出了Kubernetes里的一些重要API对象，比如配置信息ConfigMap、离线作业Job、多实例部署Deployment等等，它们都分别对应到现实中的各种实际运维需求。</p><p><img src="https://static001.geekbang.org/resource/image/9e/75/9ebab7d513a211a926dd69f7535ac175.png?wh=1478x812" alt="图片"></p><p>不过这张图虽然很经典，参考价值很高，但毕竟有些年头了，随着Kubernetes的发展，它已经不能够全面地描述Kubernetes的资源对象了。</p><p>受这张图的启发，我自己重新画了一份以Pod为中心的Kubernetes资源对象关系图，添加了一些新增的Kubernetes概念，今后我们就依据这张图来探索Kubernetes的各项功能。</p><p><img src="https://static001.geekbang.org/resource/image/b5/cf/b5a7003788cb6f2b1c5c4f6873a8b5cf.jpg?wh=1920x1298" alt="图片"></p><p>从这两张图中你也应该能够看出来，所有的Kubernetes资源都直接或者间接地依附在Pod之上，所有的Kubernetes功能都必须通过Pod来实现，所以Pod理所当然地成为了Kubernetes的核心对象。</p><h2>如何使用YAML描述Pod</h2><p>既然Pod这么重要，那么我们就很有必要来详细了解一下Pod，理解了Pod概念，我们的Kubernetes学习之旅就成功了一半。</p><p>还记得吧，我们始终可以用命令 <code>kubectl explain</code> 来查看任意字段的详细说明，所以接下来我就只简要说说写YAML时Pod里的一些常用字段。</p><p>因为Pod也是API对象，所以它也必然具有<strong>apiVersion</strong>、<strong>kind</strong>、<strong>metadata、spec</strong>这四个基本组成部分。</p><p>“apiVersion”和“kind”这两个字段很简单，对于Pod来说分别是固定的值 <code>v1</code> 和 <code>Pod</code>，而一般来说，“metadata”里应该有 <code>name</code> 和 <code>labels</code> 这两个字段。</p><p>我们在使用Docker创建容器的时候，可以不给容器起名字，但在Kubernetes里，Pod必须要有一个名字，这也是Kubernetes里所有资源对象的一个约定。在课程里，我通常会为Pod名字统一加上 <code>pod</code> 后缀，这样可以和其他类型的资源区分开。</p><p><code>name</code> 只是一个基本的标识，信息有限，所以 <code>labels</code> 字段就派上了用处。它可以添加任意数量的Key-Value，给Pod“贴”上归类的标签，结合 <code>name</code> 就更方便识别和管理了。</p><p>比如说，我们可以根据运行环境，使用标签 <code>env=dev/test/prod</code>，或者根据所在的数据中心，使用标签 <code>region: north/south</code>，还可以根据应用在系统中的层次，使用 <code>tier=front/middle/back</code> ……如此种种，只需要发挥你的想象力。</p><p>下面这段YAML代码就描述了一个简单的Pod，名字是“busy-pod”，再附加上一些标签：</p><pre><code class="language-yaml">apiVersion: v1
kind: Pod
metadata:
&nbsp; name: busy-pod
&nbsp; labels:
&nbsp; &nbsp; owner: chrono
&nbsp; &nbsp; env: demo
&nbsp; &nbsp; region: north
&nbsp; &nbsp; tier: back
</code></pre><p>“metadata”一般写上 <code>name</code> 和 <code>labels</code> 就足够了，而“spec”字段由于需要管理、维护Pod这个Kubernetes的基本调度单元，里面有非常多的关键信息，今天我介绍最重要的“<strong>containers</strong>”，其他的hostname、restartPolicy等字段你可以课后自己查阅文档学习。</p><p>“containers”是一个数组，里面的每一个元素又是一个container对象，也就是容器。</p><p>和Pod一样，container对象也必须要有一个 <code>name</code> 表示名字，然后当然还要有一个 <code>image</code> 字段来说明它使用的镜像，这两个字段是必须要有的，否则Kubernetes会报告数据验证错误。</p><p>container对象的其他字段基本上都可以和“入门篇”学过的Docker、容器技术对应，理解起来难度不大，我就随便列举几个：</p><ul>
<li><strong>ports</strong>：列出容器对外暴露的端口，和Docker的 <code>-p</code> 参数有点像。</li>
<li><strong>imagePullPolicy</strong>：指定镜像的拉取策略，可以是Always/Never/IfNotPresent，一般默认是IfNotPresent，也就是说只有本地不存在才会远程拉取镜像，可以减少网络消耗。</li>
<li><strong>env</strong>：定义Pod的环境变量，和Dockerfile里的 <code>ENV</code> 指令有点类似，但它是运行时指定的，更加灵活可配置。</li>
<li><strong>command</strong>：定义容器启动时要执行的命令，相当于Dockerfile里的 <code>ENTRYPOINT</code> 指令。</li>
<li><strong>args</strong>：它是command运行时的参数，相当于Dockerfile里的 <code>CMD</code> 指令，这两个命令和Docker的含义不同，要特别注意。</li>
</ul><p>现在我们就来编写“busy-pod”的spec部分，添加 <code>env</code>、<code>command</code>、<code>args</code> 等字段：</p><pre><code class="language-yaml">spec:
&nbsp; containers:
&nbsp; - image: busybox:latest
&nbsp; &nbsp; name: busy
&nbsp; &nbsp; imagePullPolicy: IfNotPresent
&nbsp; &nbsp; env:
&nbsp; &nbsp; &nbsp; - name: os
&nbsp; &nbsp; &nbsp; &nbsp; value: "ubuntu"
&nbsp; &nbsp; &nbsp; - name: debug
&nbsp; &nbsp; &nbsp; &nbsp; value: "on"
&nbsp; &nbsp; command:
&nbsp; &nbsp; &nbsp; - /bin/echo
&nbsp; &nbsp; args:
&nbsp; &nbsp; &nbsp; - "$(os), $(debug)"
</code></pre><p>这里我为Pod指定使用镜像busybox:latest，拉取策略是 <code>IfNotPresent</code> ，然后定义了 <code>os</code> 和 <code>debug</code> 两个环境变量，启动命令是 <code>/bin/echo</code>，参数里输出刚才定义的环境变量。</p><p>把这份YAML文件和Docker命令对比一下，你就可以看出，YAML在 <code>spec.containers</code> 字段里用“声明式”把容器的运行状态描述得非常清晰准确，要比 <code>docker run</code> 那长长的命令行要整洁的多，对人、对机器都非常友好。</p><h2>如何使用kubectl操作Pod</h2><p>有了描述Pod的YAML文件，现在我就介绍一下用来操作Pod的kubectl命令。</p><p><code>kubectl apply</code>、<code>kubectl delete</code> 这两个命令在上次课里已经说过了，它们可以使用 <code>-f</code> 参数指定YAML文件创建或者删除Pod，例如：</p><pre><code class="language-plain">kubectl apply -f busy-pod.yml
kubectl delete -f busy-pod.yml
</code></pre><p>不过，因为我们在YAML里定义了“name”字段，所以也可以在删除的时候直接指定名字来删除：</p><pre><code class="language-plain">kubectl delete pod busy-pod
</code></pre><p>和Docker不一样，Kubernetes的Pod不会在前台运行，只能在后台（相当于默认使用了参数 <code>-d</code>），所以输出信息不能直接看到。我们可以用命令 <code>kubectl logs</code>，它会把Pod的标准输出流信息展示给我们看，在这里就会显示出预设的两个环境变量的值：</p><pre><code class="language-plain">kubectl logs busy-pod
</code></pre><p><img src="https://static001.geekbang.org/resource/image/76/f2/76452a603cddaf3cce6706697369d1f2.png?wh=948x124" alt="图片"></p><p>使用命令 <code>kubectl get pod</code> 可以查看Pod列表和运行状态：</p><pre><code class="language-plain">kubectl get pod
</code></pre><p><img src="https://static001.geekbang.org/resource/image/54/9c/544d4d4521yy1e2cyy3b79615cbcc69c.png?wh=1464x184" alt="图片"></p><p>你会发现这个Pod运行有点不正常，状态是“CrashLoopBackOff”，那么我们可以使用命令 <code>kubectl describe</code> 来检查它的详细状态，它在调试排错时很有用：</p><pre><code class="language-plain">kubectl describe pod busy-pod
</code></pre><p><img src="https://static001.geekbang.org/resource/image/78/68/786bb31f3d6d69edd16ddfb540d9ef68.png?wh=1920x294" alt="图片"></p><p>通常需要关注的是末尾的“Events”部分，它显示的是Pod运行过程中的一些关键节点事件。对于这个busy-pod，因为它只执行了一条 <code>echo</code> 命令就退出了，而Kubernetes默认会重启Pod，所以就会进入一个反复停止-启动的循环错误状态。</p><p>因为Kubernetes里运行的应用大部分都是不会主动退出的服务，所以我们可以把这个busy-pod删掉，用上次课里创建的ngx-pod.yml，启动一个Nginx服务，这才是大多数Pod的工作方式。</p><pre><code class="language-plain">kubectl apply -f ngx-pod.yml
</code></pre><p>启动之后，我们再用 <code>kubectl get pod</code> 来查看状态，就会发现它已经是“Running”状态了：</p><p><img src="https://static001.geekbang.org/resource/image/6c/f9/6cd9e20234784f666687ca614873ccf9.png?wh=1124x184" alt="图片"></p><p>命令 <code>kubectl logs</code> 也能够输出Nginx的运行日志：</p><p><img src="https://static001.geekbang.org/resource/image/6c/b1/6c1ce29c29602f111ba39dea6aab95b1.png?wh=1920x415" alt="图片"></p><p>另外，kubectl也提供与docker类似的 <code>cp</code> 和 <code>exec</code> 命令，<code>kubectl cp</code> 可以把本地文件拷贝进Pod，<code>kubectl exec</code> 是进入Pod内部执行Shell命令，用法也差不多。</p><p>比如我有一个“a.txt”文件，那么就可以使用 <code>kubectl cp</code> 拷贝进Pod的“/tmp”目录里：</p><pre><code class="language-plain">echo 'aaa' &gt; a.txt
kubectl cp a.txt ngx-pod:/tmp
</code></pre><p>不过 <code>kubectl exec</code> 的命令格式与Docker有一点小差异，需要在Pod后面加上 <code>--</code>，把kubectl的命令与Shell命令分隔开，你在用的时候需要小心一些：</p><pre><code class="language-plain">kubectl exec -it ngx-pod -- sh
</code></pre><p><img src="https://static001.geekbang.org/resource/image/34/6b/343756ee45533a056fdca97f9fe2dd6b.png?wh=1920x402" alt="图片"></p><h2>小结</h2><p>好了，今天我们一起学习了Kubernetes里最核心最基本的概念Pod，知道了应该如何使用YAML来定制Pod，还有如何使用kubectl命令来创建、删除、查看、调试Pod。</p><p>Pod屏蔽了容器的一些底层细节，同时又具有足够的控制管理能力，比起容器的“细粒度”、虚拟机的“粗粒度”，Pod可以说是“中粒度”，灵活又轻便，非常适合在云计算领域作为应用调度的基本单元，因而成为了Kubernetes世界里构建一切业务的“原子”。</p><p>今天的知识要点我简单列在了下面：</p><ol>
<li>现实中经常会有多个进程密切协作才能完成任务的应用，而仅使用容器很难描述这种关系，所以就出现了Pod，它“打包”一个或多个容器，保证里面的进程能够被整体调度。</li>
<li>Pod是Kubernetes管理应用的最小单位，其他的所有概念都是从Pod衍生出来的。</li>
<li>Pod也应该使用YAML“声明式”描述，关键字段是“spec.containers”，列出名字、镜像、端口等要素，定义内部的容器运行状态。</li>
<li>操作Pod的命令很多与Docker类似，如 <code>kubectl run</code>、<code>kubectl cp</code>、<code>kubectl exec</code> 等，但有的命令有些小差异，使用的时候需要注意。</li>
</ol><p>虽然Pod是Kubernetes的核心概念，非常重要，但事实上在Kubernetes里通常并不会直接创建Pod，因为它只是对容器做了简单的包装，比较脆弱，离复杂的业务需求还有些距离，需要Job、CronJob、Deployment等其他对象增添更多的功能才能投入生产使用。</p><h2>课下作业</h2><p>最后是课下作业时间，给你留两个思考题：</p><ol>
<li>如果没有Pod，直接使用容器来管理应用会有什么样的麻烦？</li>
<li>你觉得Pod和容器之间有什么区别和联系？</li>
</ol><p>欢迎留言参与讨论，如果有收获也欢迎你分享给朋友一起学习。我们下节课再见。</p><p><img src="https://static001.geekbang.org/resource/image/f5/9b/f5f2bfcdc2ce5a94ae5113262351e89b.jpg?wh=1920x2868" alt="图片"></p>
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
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/ibZVAmmdAibBeVpUjzwId8ibgRzNk7fkuR5pgVicB5mFSjjmt2eNadlykVLKCyGA0GxGffbhqLsHnhDRgyzxcKUhjg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>pyhhou</span>
  </div>
  <div class="_2_QraFYR_0">思考题<br><br>1. 可以把 Pod 看作是介于容器之上的一层抽象，之所以需要这一层抽象是因为容器与容器之间有着不确定的关系，有的容器需要与彼此隔离，而有的容器却需要彼此交互。当容器规模增大，容器之间的作用关系就会变得极其复杂，难于管理。Pod 的出现就是为了解决容器管理的问题，让大规模应用下的容器编排变得更加清晰有序，易于维护<br><br>2. 不管是容器还是 Pod，都是虚拟概念。把普通进程或应用加上权限限制就成了容器，再把容器加上权限限制就成了 Pod。说白了，就是不断地抽象封装，这也是软件中解决复杂问题的唯一手段。容器之于Pod，就好比 线程之于进程、函数之于类、文件之于文件夹等等</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: awesome </p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-21 13:25:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/b5/46/2ac4b984.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>三溪</span>
  </div>
  <div class="_2_QraFYR_0">我这里补充下个人遇到的坑，希望对大家有所帮助！<br>当你使用kubectl apply -f指定YAML文件来创建pod时，只要你spec.containers.image里面的tag是latest，那么无论你的imagePullpolicy策略是什么，无论本地是否已存在该镜像文件，一定会先联网查询镜像信息，如果此时正好是私网无法连接互联网时，那么pod就会直接创建失败，你使用describe查询时报错也是无法拉取镜像。<br>只要tag不是latest，比如stable或者1.23什么的具体版本，本地已存在对应镜像文件，这时设置为IfNotPresent和Never才会生效，就可以在私网环境下愉快地离线创建pod。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: awesome</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-27 23:11:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/cd/dc/75ca619d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>江湖十年</span>
  </div>
  <div class="_2_QraFYR_0">imagePullPolicy 默认值并非 IfNotPresent，而是 Always：<br><br>➜ kubectl explain pod.spec.containers.imagePullPolicy<br>KIND:     Pod<br>VERSION:  v1<br><br>FIELD:    imagePullPolicy &lt;string&gt;<br><br>DESCRIPTION:<br>     Image pull policy. One of Always, Never, IfNotPresent. Defaults to Always<br>     if :latest tag is specified, or IfNotPresent otherwise. Cannot be updated.<br>     More info:<br>     https:&#47;&#47;kubernetes.io&#47;docs&#47;concepts&#47;containers&#47;images#updating-images</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Defaults to Alwaysif :latest tag is specified。 只有lastest才是always。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-18 08:42:04</div>
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
  <div class="_2_QraFYR_0">container 对象. env：定义 Pod 的环境变量<br>-------------<br>这里 不应该是 container 的环境变量吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 抱歉，不小心手误了，确实，它属于container字段，是容器的环境变量。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-18 22:41:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/f7/b1/982ea185.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Frank</span>
  </div>
  <div class="_2_QraFYR_0">老师 ，一般情况下，pod里只定义一个容器吗？  还是可以定义多个容器。  它们之间没有什么区别吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 一般是一个容器，但因为containers是一个数组，里面是可以有多个容器的。<br><br>Pod里的多个容器共享IP地址、存储卷、进程空间，pod就类似逻辑主机、虚拟机了，这些容器就可以像在主机里一样协同工作。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-20 08:49:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/22/7a/a9/279c0c39.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Guder</span>
  </div>
  <div class="_2_QraFYR_0">感觉pod有点像是docker-compose</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，不过尽量不要把docker的经验套在Kubernetes上，容易混淆。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-19 18:01:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/e4/15/31fc864e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>咩咩咩</span>
  </div>
  <div class="_2_QraFYR_0">1.若直接使用容器来管理应用，只能一个个容器去调度。应用之间一般都有进程和进程组的关系，而容器又只是单进程模型，无法管理多个进程<br>2.Pod是一组共享了某些资源的容器，只是一个逻辑概念</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-18 16:09:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/71/48/44df7f4e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>凯</span>
  </div>
  <div class="_2_QraFYR_0">想请教一下老师，除了您讲解的内容。我更想知道您学习k8s的路线，您是怎么做到到精通的。我也自己学习，可是学习的都是一下粗略的内容。您是精读k8s的文档么，参加k8s的开源项目贡献代码么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我可不是kubernetes 专家，K8s太大，我也只能挑感兴趣的学，还没到K8s开发的程度。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-21 10:22:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/2c/7e/f1efd18b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>摊牌</span>
  </div>
  <div class="_2_QraFYR_0">个人观点，不知道对不对，我觉得pod就应该叫做k8s自己的容器，如果没有pod, k8s直接调度容器对象，比如docker, 那它就永远要依赖docker; 如果k8s不对容器包装一下，没有类似pod这种粒度的话，k8s就必须选择一种现有容器技术，限制了其现在和未来的灵活性。所以，于公于私，构建pod这个基本调度单元是非常明智的选择，将来如果有新的容器技术代替了docker，但是k8s的基本单元永远是pod</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great，中间层的思想。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-18 17:43:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/mvvjzu4D1gJl8c9lnMMTatOou2EUsWCe4XiclyUOwk2rUawwqd6KKV8z9bSRMnD3ibQPUCIZUQOAkKAaKX0Ncaibw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Apple_d39574</span>
  </div>
  <div class="_2_QraFYR_0">您好，想问一下，为什么要定义容器启动时要执行的命令？不设置会怎么样？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 因为容器本质上就是一个应用程序，没有命令该如何启动应用呢？想清楚这一点应该就能够理解了吧。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-28 10:36:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q3auHgzwzM5viafIuiciaZxwUwhuibXfyeW74wHhJDq13JYibcvktLTznAVgNibCMSArDIEkjSbDmNC1JObUpIwMJibhg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_2c1b7c</span>
  </div>
  <div class="_2_QraFYR_0">向老师请教一个问题，被折磨坏了，<br>docker images 显示本地是有 busybox:stable-glibc 这个版本的，yaml文件中的 imagePullPolicy: IfNotPresent，image: busybox:stable-glibc，<br>根据这个yaml文件创建pod，显示 pod&#47;busybox created，但是 kubectl logs busybox，就显示镜像拉取失败<br>Error from server (BadRequest): container &quot;busybox&quot; in pod &quot;busybox&quot; is waiting to start: trying and failing to pull image<br>不明白为什么本地已经有镜像了，且不是最新的tag，为什么还要去拉取镜像？<br><br>get pod报错：busybox   0&#47;1     CrashLoopBackOff   7 (4m47s ago)   22m<br><br>Events:<br>  Normal   Scheduled  28m                   default-scheduler  Successfully assigned default&#47;busybox to minikube<br>  Warning  Failed     24m                   kubelet            Failed to pull image &quot;busybox:stable-glibc&quot;: rpc error: code = Unknown desc = context canceled<br>  Warning  Failed     24m (x2 over 24m)     kubelet            Error: ErrImagePull<br>  Warning  Failed     24m                   kubelet            Failed to pull image &quot;busybox:stable-glibc&quot;: rpc error: code = Unknown desc = error pulling image configuration: Get &quot;xxx&quot;: net&#47;http: TLS handshake timeout<br>  Normal   BackOff    24m (x2 over 24m)     kubelet            Back-off pulling image &quot;busybox:stable-glibc&quot;<br>  Warning  Failed     24m (x2 over 24m)     kubelet            Error: ImagePullBackOff<br>  Normal   Pulling    24m (x3 over 28m)     kubelet            Pulling image &quot;busybox:stable-glibc&quot;<br>  Normal   Pulled     21m                   kubelet            Successfully pulled image &quot;busybox:stable-glibc&quot; in 2m20.001262902s<br>  Normal   Created    20m (x4 over 21m)     kubelet            Created container busybox<br>  Normal   Started    20m (x4 over 21m)     kubelet            Started container busybox<br>  Normal   Pulled     20m (x3 over 21m)     kubelet            Container image &quot;busybox:stable-glibc&quot; already present on machine<br>  Warning  BackOff    2m55s (x86 over 21m)  kubelet            Back-off restarting failed container</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 要看pod被调度到哪个节点上运行，本地有，不代表节点上一定有，注意Kubernetes是集群管理系统，不要把本地和集群弄混了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-28 19:53:10</div>
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
  <div class="_2_QraFYR_0">请教老师两个问题：<br>Q1:：YAML中可以定义多个POD吗？<br>Q2：k8s集群启动命令每次开机后都要执行吗？<br>       minikube start --kubernetes-version=v1.23.3，这个命令是启动k8s集群，<br>       现在好像每次重启虚拟机后都要执行该命令，必须执行吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.当然可以，多个YAML对象用“---”分隔。<br><br>2.是的，毕竟minikube是一个学习环境，后面的kubeadm就不会这样了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-18 22:40:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/30/c1/2dde6700.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>密码123456</span>
  </div>
  <div class="_2_QraFYR_0">🤣原来是网线，没有插好。不说了。接受惩罚去。找了半天。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: nice</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-18 09:27:49</div>
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
  <div class="_2_QraFYR_0">老师，有几个小问题。<br><br>1. 用标签 env=dev&#47;test&#47;prod，使用标签 region: north&#47;south，tier=front&#47;middle&#47;back，这里的等于号和冒号有什么区别吗？<br><br>2. kubectl delete -f busy-pod.yaml，这里的“yaml” 是不是应该写成 “yml”。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1. 这个文字只是示意，实际在YAML 里写必须用冒号。<br><br>2.看得很仔细，不小心写错了，应该和上面的一样，后缀都是yml。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-27 21:11:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/WrANpwBMr6DsGAE207QVs0YgfthMXy3MuEKJxR8icYibpGDCI1YX4DcpDq1EsTvlP8ffK1ibJDvmkX9LUU4yE8X0w/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>星垂平野阔</span>
  </div>
  <div class="_2_QraFYR_0">1.直接用容器管理会让我们管理容器变得很复杂，每一个容器的配置我们都要用一长串配置信息对容器进行管理和配置，而且容器间的通信貌似也是一个大问题？<br>2.pod就像收纳箱，里面装了不同类型的物品（容器，当我需要的时候直接找到对应的箱子打开就好。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-18 20:41:55</div>
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
  <div class="_2_QraFYR_0">pod.spec.containers中的command与arg分别对应Dockerfile中的ENTRYPOINT与CMD，查资源对比其不同之处有：<br>1. 可用形式不同，前者只有array的形式，类似于exec form，后者还有shell form，即在command中如果需要使用到shell需要显式指定如&#47;bin&#47;sh。<br>2. 环境变量引用方式不同，前者可以通过&#39;$(env_varname)&#39;的方式引用pod.spec.conainers.env中给定的变量，但后者需要在exec form中显式指定shell，并使用&#39;${env_varname}&#39;来引用ENV中设定的环境变量，注意两者引用的形式不同，后者是标准的shell变量引用写法，前者不是。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: awesome</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-18 16:38:52</div>
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
  <div class="_2_QraFYR_0">来回答下作业：<br>1. 如果没有 Pod，直接使用容器来管理应用会有什么样的麻烦？<br>Pod是一个或多个容器的集合，首先，如果Pod中运行的是单容器，则使用Pod包一层的好处是可以管理其中容器的状态如暂停、重启等（因Pod短暂的生命周期，不清楚这么做有什么好处，debug？），比如可以替换容器镜像；其次，如果Pod中运行的是多容器，则这些容器之间共享网络与存储，可以高效的分享资源与本地化服务调用，在调度方面，如果关联容器不使用Pod包一层则会增加调试的复杂度，影响性能，有了Pod则可以统一调度方便管理。<br><br>2. 你觉得 Pod 和容器之间有什么区别和联系？<br>Pod是一个或多个关联容器的组合，它是一逻辑概念，它要解决的是在资源隔离的容器之上，如何实现关联容器之间高效的共享网络与存储。<br>实现层面首先启动infra容器pause用来创建并保持网络空间，与Pod保持一致的生命周期，再启动业务容器加入到此NET空间，在docker运行时中同一Pod中的容器共享ipc&#47;net&#47;user&#47;uts等命名空间。<br>Pod与容器之间的关系类似于Linux进程组与进程的关系。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-18 16:17:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/30/c1/2dde6700.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>密码123456</span>
  </div>
  <div class="_2_QraFYR_0">提示我， image  can&#39;t  be  pulled怎么解决呀？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 尝试先用docker下载</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-18 09:12:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/65/dc/d0c58ce5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>闲渔一下</span>
  </div>
  <div class="_2_QraFYR_0">1.通常应用之间存在超亲密关系，如果只按容器去管理，就把它们割裂开来了，调度的时候只能按一个个容器去调度，而有了pod，可以按照一组容器这样去调度<br>2.pod是一个逻辑概念，由容器组成</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-18 08:54:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2e/97/a9/e3b097f1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>蜘蛛侠</span>
  </div>
  <div class="_2_QraFYR_0">可以这样理解嘛，Pod是不是就是把docker-compose 创建的多个容器放在了一个“房间”</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 还是不太一样的，docker-compose和Pod的理念有些区别， Pod里的容器可以理解成是在一台主机上运行，而docker-compose里的容器是通过IP地址来通信的，关系要松散一些。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-07 09:37:05</div>
  </div>
</div>
</div>
</li>
</ul>