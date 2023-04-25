<audio title="特别放送 _ 基于 Kubernetes 的云原生应用管理，到底应该怎么做？" src="https://static001.geekbang.org/resource/audio/fa/13/fa1d1648d2fe20ee31382103e9e8e513.mp3" controls="controls"></audio> 
<p>你好，我是张磊。</p><p>虽然《深入剖析 Kubernetes》专栏已经完结了一段时间了，但是在留言中，很多同学依然在不时地推敲与消化专栏里的知识和案例。对此我非常开心，同时也看到大家在实践 Kubernetes的过程中存在的很多问题。所以在接下来的一段时间里，我会以 Kubernetes 最为重要的一个主线能力作为专题，对专栏内容从广度和深度两个方向上进行一系列延伸与拓展。希望这些内容，能够帮助你在探索这个全世界最受欢迎的开源生态的过程中，更加深刻地理解到 Kubernetes 项目的内涵与本质。</p><p>随着 Kubernetes 项目的日趋成熟与稳定，越来越多的人都在问我这样一个问题：现在的 Kubernetes 项目里，最有价值的部分到底是哪些呢？</p><p>为了回答这个问题，我们不妨一起回到第13篇文章<a href="https://time.geekbang.org/column/article/40092">《为什么我</a><a href="https://time.geekbang.org/column/article/40092">们需要P</a><a href="https://time.geekbang.org/column/article/40092">od？》</a>中，来看一下几个非常典型的用户提问。</p><blockquote>
<p>用户一：关于升级War和Tomcat那块，也是先修改yaml，然后Kubenertes执行升级命令，pod会重新启动，生产也是按照这种方式吗？所以这种情况下，如果只是升级个War包，或者加一个新的War包，Tomcat也要重新启动？这就不是完全松耦合了？</p>
</blockquote><blockquote>
<p>用户二：WAR包的例子并没有解决频发打包的问题吧? WAR包变动后, geektime/sample:v2包仍然需要重新打包。这和东西一股脑装在tomcat中后, 重新打tomcat 并没有差太多吧?</p>
</blockquote><!-- [[[read_end]]] --><blockquote>
<p>用户三：关于部署war包和tomcat，在升级war的时候，先修改yaml，然后Kubernetes会重启整个pod，然后按照定义好的容器启动顺序流程走下去？正常生产是按照这种方式进行升级的吗？</p>
</blockquote><p>在《为什么我们需要Pod？》这篇文章中，为了讲解 Pod 里容器间关系（即：容器设计模式）的典型场景，我举了一个“WAR 包与 Web 服务器解耦”的例子。在这个例子中，我既没有让你通过 Volume 的方式将 WAR 包挂载到 Tomcat 容器中，也没有建议你把 WAR 包和 Tomcat 打包在一个镜像里，而是用了一个 InitContainer 将 WAR 包“注入”给了Tomcat 容器。</p><p>不过，不同用户面对的场景不同，对问题的思考角度也不一样。所以在这一节里，大家提出了很多不同维度的问题。这些问题总结起来，其实无外乎有两个疑惑：</p><ol>
<li>如果 WAR 包更新了，那不是也得重新制作 WAR 包容器的镜像么？这和重新打 Tomcat 镜像有很大区别吗？</li>
<li>当用户通过 YAML 文件将 WAR 包镜像更新后，整个 Pod 不会重建么？Tomcat 需要重启么？</li>
</ol><p>这里的两个问题，实际上都聚焦在了这样一个对于 Kubernetes 项目至关重要的核心问题之上：<strong>基于 Kubernetes 的应用管理，到底应该怎么做？</strong></p><p>比如，对于第一个问题，在不同规模、不同架构的组织中，可能有着不同的看法。一般来说，如果组织的规模不大、发布和迭代次数不多的话，将 WAR 包（应用代码）的发布流程和 Tomcat （Web 服务器）的发布流程解耦，实际上很难有较强的体感。在这些团队中，Tomcat 本身很可能就是开发人员自己负责管理的，甚至被认为是应用的一部分，无需进行很明确的分离。</p><p>而对于更多的组织来说，Tomcat 作为全公司通用的 Web 服务器，往往有一个专门的小团队兼职甚至全职负责维护。这不仅包括了版本管理、统一升级和安全补丁等工作，还会包括全公司通用的性能优化甚至定制化内容。</p><p>在这种场景下，WAR 包的发布流水线（制作 WAR包镜像的流水线），和 Tomcat 的发布流水线（制作 Tomcat 镜像的流水线）其实是通过两个完全独立的团队在负责维护，彼此之间可能都不知晓。</p><p>这时候，在 Pod 的定义中直接将两个容器解耦，相比于每次发布前都必须先将两个镜像“融合”成一个镜像然后再发布，就要自动化得多了。这个原因是显而易见的：开发人员不需要额外维护一个“重新打包”应用的脚本、甚至手动地去做这个步骤了。</p><p><strong>这正是上述设计模式带来的第一个好处：自动化。</strong></p><p>当然，正如另外一些用户指出的那样，这个“解耦”的工作，貌似也可以通过把 WAR 包以 Volume 的方式挂载进 Tomcat 容器来完成，对吧？</p><p>然而，相比于 Volume 挂载的方式，通过在 Pod 定义中解耦上述两个容器，其实还会带来<strong>另一个更重要的好处，叫作：自描述。</strong></p><p>为了解释这个好处，我们不妨来重新看一下这个 Pod 的定义：</p><pre><code>apiVersion: v1
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
</code></pre><p>现在，我来问你这样一个问题：<strong>这个 Pod 里，应用的版本是多少？Tomcat 的版本又是多少？</strong></p><p>相信你一眼就能看出来：应用版本是 v2，Tomcat 的版本是 7.0.42-v2。</p><p><strong>没错！所以我们说，一个良好编写的 Pod的 YAML 文件应该是“自描述”的，它直接描述了这个应用本身的所有信息。</strong></p><p>但是，如果我们改用 Volume 挂载的方式来解耦WAR 包和 Tomcat 服务器，这个 Pod 的 YAML 文件会变成什么样子呢？如下所示：</p><pre><code>apiVersion: v1
kind: Pod
metadata:
  name: javaweb-2
spec:
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
    flexVolume:
      driver: &quot;alicloud/disk&quot;
      fsType: &quot;ext4&quot;
      options:
        volumeId: &quot;d-bp1j17ifxfasvts3tf40&quot;
</code></pre><p>在上面这个例子中，我们就通过了一个名叫“app-volume”的数据卷（Volume），来为我们的 Tomcat 容器提供 WAR 包文件。需要注意的是，这个 Volume 必须是持久化类型的数据卷（比如本例中的阿里云盘），绝不可以是 emptyDir 或者 hostPath 这种临时的宿主机目录，否则一旦 Pod 重调度你的 WAR 包就找不回来了。</p><p>然而，如果这时候我再问你：这个 Pod 里，应用的版本是多少？Tomcat 的版本又是多少？</p><p>这时候，你可能要傻眼了：在这个 Pod YAML 文件里，根本看不到应用的版本啊，它是通过 Volume 挂载而来的！</p><p><strong>也就是说，这个 YAML文件再也没有“自描述”的能力了。</strong></p><p>更为棘手的是，在这样的一个系统中，你肯定是不可能手工地往这个云盘里拷贝 WAR 包的。所以，上面这个Pod 要想真正工作起来，你还必须在外部再维护一个系统，专门负责在云盘里拷贝指定版本的 WAR 包，或者直接在制作这个云盘的过程中把指定 WAR 包打进去。然而，无论怎么做，这个工作都是非常不舒服并且自动化程度极低的，我强烈不推荐。</p><p><strong>要想 “Volume 挂载”的方式真正能工作，可行方法只有一种：那就是写一个专门的 Kubernetes Volume 插件（比如，Flexvolume或者CSI插件）</strong> 。这个插件的特殊之处，在于它在执行完 “Mount 阶段”后，会自动执行一条从远端下载指定 WAR 包文件的命令，从而将 WAR 包正确放置在这个 Volume 里。这个 WAR 包文件的名字和路径，可以通过 Volume 的自定义参数传递，比如：</p><pre><code>...
volumes:
  - name: app-volume
    flexVolume:
      driver: &quot;geektime/war-vol&quot;
      fsType: &quot;ext4&quot;
      options:
        downloadURL: &quot;https://github.com/geektime/sample/releases/download/v2/sample.war&quot;
</code></pre><p>在这个例子中， 我就定义了 app-volume 的类型是 geektime/war-vol，在挂载的时候，它会自动从 downloadURL 指定的地址下载指定的  WAR 包，问题解决。</p><p>可以看到，这个 YAML  文件也是“自描述”的：因为你可以通过 downloadURL 等参数知道这个应用到底是什么版本。看到这里，你是不是已经感受到 “Volume 挂载的方式” 实际上一点都不简单呢？</p><p>在明白了我们在 Pod 定义中解耦 WAR 包容器和 Tomcat 容器能够得到的两个好处之后，第一个问题也就回答得差不多了。这个问题的本质，其实是一个关于“ Kubernetes 应用究竟应该如何描述”的问题。</p><p><strong>而这里的原则，最重要的就是“自描述”。</strong></p><p>我们之前已经反复讲解过，Kubernetes 项目最强大的能力，就是“声明式”的应用定义方式。这个“声明式”背后的设计思想，是在YAML 文件（Kubernetes API  对象）中描述应用的“终态”。然后 Kubernetes 负责通过“控制器模式”不断地将整个系统的实际状态向这个“终态”逼近并且达成一致。</p><p>而“声明式”最大的好处是什么呢？</p><p><strong>“声明式”带来最大的好处，其实正是“自动化”</strong>。作为一个 Kubernetes 用户，你只需要在 YAML 里描述清楚这个应用长什么样子，那么剩下的所有事情，就都可以放心地交给 Kubernetes 自动完成了：它会通过控制器模式确保这个系统里的应用状态，最终并且始终跟你在 YAML 文件里的描述完全一致。</p><p>这种<strong>“把简单交给用户，把复杂留给自己”</strong>的精神，正是一个“声明式”项目的精髓所在了。</p><p>这也就意味着，如果你的 YAML 文件不是“自描述”的，那么 Kubernetes 就不能“完全”理解你的应用的“终态”到底是什么样子的，它也就没办法把所有的“复杂”都留给自己。这不，你就得自己去写一个额外 Volume 插件去了。</p><p>回到之前用户提到的第二个问题：当通过 YAML 文件将 WAR 包镜像更新后，整个 Pod 不会重建么？Tomcat 需要重启么？</p><p>实际上，当一个  Pod 里的容器镜像被更新后，kubelet 本身就能够判断究竟是哪个容器需要更新，而不会“无脑”地重建整个Pod。当然，你的 Tomcat 需要配置好 reloadable=“true”，这样就不需要重启 Tomcat 服务器了，这是一个非常常见的做法。</p><p>但是，<strong>这里还有一个细节需要注意</strong>。即使 kubelet 本身能够“智能”地单独重建被更新的容器，但如果你的 Pod 是用  Deployment 管理的话，它会按照自己的发布策略（RolloutStrategy） 来通过重建的方式更新 Pod。</p><p>这时候，如果这个 Pod 被重新调度到其他机器上，那么 kubelet “单独重建被更新的容器”的能力就没办法发挥作用了。所以说，要让这个案例中的“解耦”能力发挥到最佳程度，你还需要一个“原地升级”的功能，即：允许 Kubernetes 在原地进行 Pod 的更新，避免重调度带来的各种麻烦。</p><p>原地升级能力，在 Kubernetes 的默认控制器中都是不支持的。但，这是社区开源控制器项目  <a href="https://github.com/openkruise/kruise">https://</a><a href="https://github.com/openkruise/kruise">github.com/open</a><a href="https://github.com/openkruise/kruise">kru</a><a href="https://github.com/openkruise/kruise">ise/kruise</a> 的重要功能之一，如果你感兴趣的话可以研究一下。</p><h2>总结</h2><p>说到这里，再让我们回头看一下文章最开始大家提出的共性问题：现在的 Kubernetes 项目里，最有价值的部分到底是哪些？这个项目的本质到底在哪部分呢？</p><p>实际上，通过深入地讲解 “Tomcat 与 WAR 包解耦”这个案例，你可以看到 Kubernetes 的“声明式 API”“容器设计模式”“控制器原理”，以及kubelet 的工作机制等很多核心知识点，实际上是可以通过一条主线贯穿起来的。这条主线，从“应用如何描述”开始，到“容器如何运行”结束。</p><p><strong>这条主线，正是 Kubernetes 项目中最具价值的那个部分，即：云原生应用管理（Cloud Native Application Management）</strong>。它是一条连接 Kubernetes 项目绝大多数核心特性的关键线索，也是 Kubernetes 项目乃至整个云原生社区这五年来飞速发展背后唯一不变的精髓。</p><p><img src="https://static001.geekbang.org/resource/image/96/25/96ef8576a26f5e6266c422c0d6519725.jpg?wh=1110*659" alt=""></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/3c/70/0c9a429d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>平常心</span>
  </div>
  <div class="_2_QraFYR_0">购买所有课程里面最好的，没有之一</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-07 21:42:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/0b/35/2c56c29c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>arthur</span>
  </div>
  <div class="_2_QraFYR_0">磊神，开个实战新课吧，肯定火！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-08 12:16:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c7/a3/935e8ad2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>茶客ぶる声</span>
  </div>
  <div class="_2_QraFYR_0">大佬，继续开课吧，开个新课，大伙再买</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 😉考虑下</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-17 12:19:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/ba/a5/266ae965.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Ma.LW</span>
  </div>
  <div class="_2_QraFYR_0">磊哥，根据Kubernetes的动态给我们一直讲下去吧，技术更新不断，我们也想继续前行！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-01 12:14:11</div>
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
  <div class="_2_QraFYR_0">Kubernetes最大价值： 声明式API和控制器模式<br><br>Servless三个特征： 按使用计费、工作流驱动、高可扩展性<br><br>云原生本质： 敏捷、可扩展、可复制，充分利用“云”能力，发挥“云”价值的最佳上云路径<br><br>对于应用如何部署到kuberentes的问题里，有一个例子是Tomcat里运行一个WAR包。有两个问题：<br>1、为什么不把Tomcat和WAR包打包在一个镜像里<br>   放在一个镜像里，耦合太重。任何一方的发布都要修改整个镜像<br>2、为什么不把WAR包放到一个持久化的Volume里，Tomcat容器启动的时候去挂载？<br>  通过Volume挂载，缺少自描述字段（比如版本）。只能知道这里面是一个必要的文件。<br>  当然可以通过自定义开发一个Volume插件，来从指定的描述字段中拉取一个WAR包来实现，但方案较为复杂<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-13 13:22:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/09/e1/100a0526.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>kakj</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，目前看了很多k8s相关的书籍，想继续深入研究是不是要研究源码了，最后能不能出一期关于如何研究k8s源码</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-13 14:17:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/b6/97/93e82345.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>陆培尔</span>
  </div>
  <div class="_2_QraFYR_0">应该后面会讲到helm和kustomize吧，讲真老师觉得这俩哪个是后续的发展方向？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-10 22:39:46</div>
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
  <div class="_2_QraFYR_0">有些课程，读着读着就没有兴趣了，这个课我看了很多遍了，依然还想看。这就是功底啊。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-18 11:16:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/d9/ca/818fde81.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>kevin</span>
  </div>
  <div class="_2_QraFYR_0">这课买的太值了，还可以持续更新！磊哥良心啊</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-11 14:47:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_df0ab0</span>
  </div>
  <div class="_2_QraFYR_0">磊哥，开新课讲一下k8s的生态吧。生产落地实践也非常重要</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-01 11:10:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJVSibAiadiaDMhT4HmdygW8gKNLTZP92zgfeibRX6bMbZzljnYV0askzjZwjGH09Tkicydq11IxSap34w/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>yujiechenglun</span>
  </div>
  <div class="_2_QraFYR_0">购买的课程中最好的，没有之一，课程脉络清晰，每节课都条理分明，重点明了，写作水平很高，听语音也有类似的感觉，思路很清晰，感觉老师不仅</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-21 02:11:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/6b/f7/3a3b82c2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Aspirin</span>
  </div>
  <div class="_2_QraFYR_0">磊神，出一个K8s实战的专栏吧，讲讲实际中如何把各种项目应用部署到云上。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-24 10:41:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/10/a8/a7aedb90.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>叶昊</span>
  </div>
  <div class="_2_QraFYR_0">因为张老师开的这门课，我要下定决心学好go，要更深入学习剖析kubernetes。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-21 07:28:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/77/83/09025610.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>rain</span>
  </div>
  <div class="_2_QraFYR_0">这么久了还更新，点赞一个，希望看的新的文章</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-21 18:08:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/1a/bb/7cc8db2a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>水牛</span>
  </div>
  <div class="_2_QraFYR_0">磊哥讲的太好了！这已经是第三遍学习了，每一次都有新的体会。这里对于 kubernetes 精髓的总结太到位了！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-17 17:43:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/ec/1a/f0dbaad3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>龙</span>
  </div>
  <div class="_2_QraFYR_0">都看玩了，这是第一个看完的教程！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-26 17:14:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1c/08/7b/7f086546.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Alex</span>
  </div>
  <div class="_2_QraFYR_0">真乃业界良心</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-02 11:41:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/32/0b/81ae214b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>凌</span>
  </div>
  <div class="_2_QraFYR_0">大佬再开个serverless.，kubeflow的课吧</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-17 08:17:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/4c/c8/bed1e08a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>辣椒</span>
  </div>
  <div class="_2_QraFYR_0">高屋建瓴，值得读三遍以上</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-01 16:57:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>AliFYB</span>
  </div>
  <div class="_2_QraFYR_0">看了你的课程非常有收获，可否请老师聊聊对FaaS的理解？云原生下FaaS如何与以K8S为代表的容器服务共存，是否会替代微服务，以及应该如何划分各自的作用域？十分感谢！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-15 17:48:15</div>
  </div>
</div>
</div>
</li>
</ul>