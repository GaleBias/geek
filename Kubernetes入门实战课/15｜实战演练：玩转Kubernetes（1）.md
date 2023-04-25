<audio title="15｜实战演练：玩转Kubernetes（1）" src="https://static001.geekbang.org/resource/audio/97/e5/971343e64de4b61fd3e7e5091b073de5.mp3" controls="controls"></audio> 
<p>你好，我是Chrono。</p><p>经过两个星期的学习，到今天我们的“初级篇”也快要结束了。</p><p>和之前的“入门篇”一样，在这次课里，我也会对前面学过的知识做一个比较全面的回顾，毕竟Kubernetes领域里有很多新名词、新术语、新架构，知识点多且杂，这样的总结复习就更有必要。</p><p>接下来我还是先简要列举一下“初级篇”里讲到的Kubernetes要点，然后再综合运用这些知识，演示一个实战项目——还是搭建WordPress网站，不过这次不是在Docker里，而是在Kubernetes集群里。</p><h2>Kubernetes技术要点回顾</h2><p>容器技术开启了云原生的大潮，但成熟的容器技术，到生产环境的应用部署的时候，却显得“步履维艰”。因为容器只是针对单个进程的隔离和封装，而实际的应用场景却是要求许多的应用进程互相协同工作，其中的各种关系和需求非常复杂，在容器这个技术层次很难掌控。</p><p>为了解决这个问题，<strong>容器编排</strong>（Container Orchestration）就出现了，它可以说是以前的运维工作在云原生世界的落地实践，本质上还是在集群里调度管理应用程序，只不过管理的主体由人变成了计算机，管理的目标由原生进程变成了容器和镜像。</p><p>而现在，容器编排领域的王者就是——Kubernetes。</p><!-- [[[read_end]]] --><p>Kubernetes源自Borg系统，它凝聚了Google的内部经验和CNCF的社区智慧，所以战胜了竞争对手Apache Mesos和Docker Swarm，成为了容器编排领域的事实标准，也成为了云原生时代的基础操作系统，学习云原生就必须要掌握Kubernetes。</p><p>（<a href="https://time.geekbang.org/column/article/529800">10讲</a>）Kubernetes的<strong>Master/Node架构</strong>是它具有自动化运维能力的关键，也对我们的学习至关重要，这里我再用另一张参考架构图来简略说明一下它的运行机制（<a href="https://kubernetes.io/blog/2018/07/18/11-ways-not-to-get-hacked">图片来源</a>）：</p><p><img src="https://static001.geekbang.org/resource/image/f4/05/f429ca7114eebf140632409f3fbcbb05.png?wh=1475x852" alt="图片"></p><p>Kubernetes把集群里的计算资源定义为节点（Node），其中又划分成控制面和数据面两类。</p><ul>
<li>控制面是Master节点，负责管理集群和运维监控应用，里面的核心组件是<strong>apiserver、etcd、scheduler、controller-manager</strong>。</li>
<li>数据面是Worker节点，受Master节点的管控，里面的核心组件是<strong>kubelet、kube-proxy、container-runtime</strong>。</li>
</ul><p>此外，Kubernetes还支持插件机制，能够灵活扩展各项功能，常用的插件有DNS和Dashboard。</p><p>为了更好地管理集群和业务应用，Kubernetes从现实世界中抽象出了许多概念，称为“<strong>API对象</strong>”，描述这些对象就需要使用<strong>YAML</strong>语言。</p><p>YAML是JSON的超集，但语法更简洁，表现能力更强，更重要的是它以“<strong>声明式</strong>”来表述对象的状态，不涉及具体的操作细节，这样Kubernetes就能够依靠存储在etcd里集群的状态信息，不断地“调控”对象，直至实际状态与期望状态相同，这个过程就是Kubernetes的自动化运维管理（<a href="https://time.geekbang.org/column/article/529813">11讲</a>）。</p><p>Kubernetes里有很多的API对象，其中最核心的对象是“<strong>Pod</strong>”，它捆绑了一组存在密切协作关系的容器，容器之间共享网络和存储，在集群里必须一起调度一起运行。通过Pod这个概念，Kubernetes就简化了对容器的管理工作，其他的所有任务都是通过对Pod这个最小单位的再包装来实现的（<a href="https://time.geekbang.org/column/article/531551">12讲</a>）。</p><p>除了核心的Pod对象，基于“单一职责”和“对象组合”这两个基本原则，我们又学习了4个比较简单的API对象，分别是<strong>Job/CronJob</strong>和<strong>ConfigMap</strong>/<strong>Secret</strong>。</p><ul>
<li>Job/CronJob对应的是离线作业，它们逐层包装了Pod，添加了作业控制和定时规则（<a href="https://time.geekbang.org/column/article/531566">13讲</a>）。</li>
<li>ConfigMap/Secret对应的是配置信息，需要以环境变量或者存储卷的形式注入进Pod，然后进程才能在运行时使用（<a href="https://time.geekbang.org/column/article/533395">14讲</a>）。</li>
</ul><p>和Docker类似，Kubernetes也提供一个客户端工具，名字叫“<strong>kubectl</strong>”，它直接与Master节点的apiserver通信，把YAML文件发送给RESTful接口，从而触发Kubernetes的对象管理工作流程。</p><p>kubectl的命令很多，查看自带文档可以用 <code>api-resources</code>、<code>explain</code> ，查看对象状态可以用 <code>get</code>、<code>describe</code>、<code>logs</code> ，操作对象可以用 <code>run</code>、<code>apply</code>、<code>exec</code>、<code>delete</code> 等等（<a href="https://time.geekbang.org/column/article/529780">09讲</a>）。</p><p>使用YAML描述API对象也有固定的格式，必须写的“头字段”是“<strong>apiVersion</strong>”“<strong>kind</strong>”“<strong>metadata</strong>”，它们表示对象的版本、种类和名字等元信息。实体对象如Pod、Job、CronJob会再有“<strong>spec</strong>”字段描述对象的期望状态，最基本的就是容器信息，非实体对象如ConfigMap、Secret使用的是“<strong>data</strong>”字段，记录一些静态的字符串信息。</p><p>好了，“初级篇”里的Kubernetes知识要点我们就基本总结完了，如果你发现哪部分不太清楚，可以课后再多复习一下前面的课程加以巩固。</p><h2>WordPress网站基本架构</h2><p>下面我们就在Kubernetes集群里再搭建出一个WordPress网站，用的镜像还是“入门篇”里的那三个应用：WordPress、MariaDB、Nginx，不过当时我们是直接以容器的形式来使用它们，现在要改成Pod的形式，让它们运行在Kubernetes里。</p><p>我还是画了一张简单的架构图，来说明这个系统的内部逻辑关系：</p><p><img src="https://static001.geekbang.org/resource/image/3d/cc/3d9d09078f1200a84c63a7cea2f40bcc.jpg?wh=1920x865" alt="图片"></p><p>从这张图中你可以看到，网站的大体架构是没有变化的，毕竟应用还是那三个，它们的调用依赖关系也必然没有变化。</p><p>那么Kubernetes系统和Docker系统的区别又在哪里呢？</p><p>关键就在<strong>对应用的封装</strong>和<strong>网络环境</strong>这两点上。</p><p>现在WordPress、MariaDB这两个应用被封装成了Pod（由于它们都是在线业务，所以Job/CronJob在这里派不上用场），运行所需的环境变量也都被改写成ConfigMap，统一用“声明式”来管理，比起Shell脚本更容易阅读和版本化管理。</p><p>另外，Kubernetes集群在内部维护了一个自己的专用网络，这个网络和外界隔离，要用特殊的“端口转发”方式来传递数据，还需要在集群之外用Nginx反向代理这个地址，这样才能实现内外沟通，对比Docker的直接端口映射，这里略微麻烦了一些。</p><h2>WordPress网站搭建步骤</h2><p>了解基本架构之后，接下来我们就逐步搭建这个网站系统，总共需要4步。</p><p><strong>第一步</strong>当然是要编排MariaDB对象，它的具体运行需求可以参考“入门篇”的实战演练课，这里我就不再重复了。</p><p>MariaDB需要4个环境变量，比如数据库名、用户名、密码等，在Docker里我们是在命令行里使用参数 <code>--env</code>，而在Kubernetes里我们就应该使用ConfigMap，为此需要定义一个 <code>maria-cm</code> 对象：</p><pre><code class="language-yaml">apiVersion: v1
kind: ConfigMap
metadata:
&nbsp; name: maria-cm

data:
&nbsp; DATABASE: 'db'
&nbsp; USER: 'wp'
&nbsp; PASSWORD: '123'
&nbsp; ROOT_PASSWORD: '123'
</code></pre><p>然后我们定义Pod对象 <code>maria-pod</code>，把配置信息注入Pod，让MariaDB运行时从环境变量读取这些信息：</p><pre><code class="language-yaml">apiVersion: v1
kind: Pod
metadata:
&nbsp; name: maria-pod
&nbsp; labels:
&nbsp; &nbsp; app: wordpress
&nbsp; &nbsp; role: database

spec:
&nbsp; containers:
&nbsp; - image: mariadb:10
&nbsp; &nbsp; name: maria
&nbsp; &nbsp; imagePullPolicy: IfNotPresent
&nbsp; &nbsp; ports:
&nbsp; &nbsp; - containerPort: 3306

&nbsp; &nbsp; envFrom:
&nbsp; &nbsp; - prefix: 'MARIADB_'
&nbsp; &nbsp; &nbsp; configMapRef:
&nbsp; &nbsp; &nbsp; &nbsp; name: maria-cm
</code></pre><p>注意这里我们使用了一个新的字段“<strong>envFrom</strong>”，这是因为ConfigMap里的信息比较多，如果用 <code>env.valueFrom</code> 一个个地写会非常麻烦，容易出错，而 <code>envFrom</code> 可以一次性地把ConfigMap里的字段全导入进Pod，并且能够指定变量名的前缀（即这里的 <code>MARIADB_</code>），非常方便。</p><p>使用 <code>kubectl apply</code> 创建这个对象之后，可以用 <code>kubectl get pod</code> 查看它的状态，如果想要获取IP地址需要加上参数 <code>-o wide</code> ：</p><pre><code class="language-plain">kubectl apply -f mariadb-pod.yml
kubectl get pod -o wide
</code></pre><p><img src="https://static001.geekbang.org/resource/image/3f/98/3fb0242f97c782f79ecf8ba845c81798.png?wh=1788x362" alt="图片"></p><p>现在数据库就成功地在Kubernetes集群里跑起来了，IP地址是“172.17.0.2”，注意这个地址和Docker的不同，是Kubernetes里的私有网段。</p><p>接着是<strong>第二步</strong>，编排WordPress对象，还是先用ConfigMap定义它的环境变量：</p><pre><code class="language-yaml">apiVersion: v1
kind: ConfigMap
metadata:
&nbsp; name: wp-cm

data:
&nbsp; HOST: '172.17.0.2'
&nbsp; USER: 'wp'
&nbsp; PASSWORD: '123'
&nbsp; NAME: 'db'
</code></pre><p>在这个ConfigMap里要注意的是“HOST”字段，它必须是MariaDB Pod的IP地址，如果不写正确WordPress会无法正常连接数据库。</p><p>然后我们再编写WordPress的YAML文件，为了简化环境变量的设置同样使用了 <code>envFrom</code>：</p><pre><code class="language-yaml">apiVersion: v1
kind: Pod
metadata:
&nbsp; name: wp-pod
&nbsp; labels:
&nbsp; &nbsp; app: wordpress
&nbsp; &nbsp; role: website

spec:
&nbsp; containers:
&nbsp; - image: wordpress:5
&nbsp; &nbsp; name: wp-pod
&nbsp; &nbsp; imagePullPolicy: IfNotPresent
&nbsp; &nbsp; ports:
&nbsp; &nbsp; - containerPort: 80

&nbsp; &nbsp; envFrom:
&nbsp; &nbsp; - prefix: 'WORDPRESS_DB_'
&nbsp; &nbsp; &nbsp; configMapRef:
&nbsp; &nbsp; &nbsp; &nbsp; name: wp-cm
</code></pre><p>接着还是用 <code>kubectl apply</code> 创建对象，<code>kubectl get pod</code> 查看它的状态：</p><pre><code class="language-plain">kubectl apply -f wp-pod.yml
kubectl get pod -o wide
</code></pre><p><img src="https://static001.geekbang.org/resource/image/d5/de/d5e8c09e70e90179d651bf3c28abc0de.png?wh=1562x426" alt="图片"></p><p><strong>第三步</strong>是为WordPress Pod映射端口号，让它在集群外可见。</p><p>因为Pod都是运行在Kubernetes内部的私有网段里的，外界无法直接访问，想要对外暴露服务，需要使用一个专门的 <code>kubectl port-forward</code> 命令，它专门负责把本机的端口映射到在目标对象的端口号，有点类似Docker的参数 <code>-p</code>，经常用于Kubernetes的临时调试和测试。</p><p>下面我就把本地的“8080”映射到WordPress Pod的“80”，kubectl会把这个端口的所有数据都转发给集群内部的Pod：</p><pre><code class="language-plain">kubectl port-forward wp-pod 8080:80 &amp;
</code></pre><p><img src="https://static001.geekbang.org/resource/image/d4/be/d445d205ae6f8c966200ffa9ba7f29be.png?wh=1366x240" alt="图片"></p><p>注意在命令的末尾我使用了一个 <code>&amp;</code> 符号，让端口转发工作在后台进行，这样就不会阻碍我们后续的操作。</p><p>如果想关闭端口转发，需要敲命令 <code>fg</code> ，它会把后台的任务带回到前台，然后就可以简单地用“Ctrl + C”来停止转发了。</p><p><strong>第四步</strong>是创建反向代理的Nginx，让我们的网站对外提供服务。</p><p>这是因为WordPress网站使用了URL重定向，直接使用“8080”会导致跳转故障，所以为了让网站正常工作，我们还应该在Kubernetes之外启动Nginx反向代理，保证外界看到的仍然是“80”端口号。（这里的细节和我们的课程关系不大，感兴趣的同学可以留言提问讨论）</p><p>Nginx的配置文件和<a href="https://time.geekbang.org/column/article/528740">第7讲</a>基本一样，只是目标地址变成了“127.0.0.1:8080”，它就是我们在第三步里用 <code>kubectl port-forward</code> 命令创建的本地地址：</p><pre><code class="language-plain">server {
&nbsp; listen 80;
&nbsp; default_type text/html;

&nbsp; location / {
&nbsp; &nbsp; &nbsp; proxy_http_version 1.1;
&nbsp; &nbsp; &nbsp; proxy_set_header Host $host;
&nbsp; &nbsp; &nbsp; proxy_pass http://127.0.0.1:8080;
&nbsp; }
}
</code></pre><p>然后我们用 <code>docker run -v</code> 命令加载这个配置文件，以容器的方式启动这个Nginx代理：</p><pre><code class="language-plain">docker run -d --rm \
&nbsp; &nbsp; --net=host \
&nbsp; &nbsp; -v /tmp/proxy.conf:/etc/nginx/conf.d/default.conf \
&nbsp; &nbsp; nginx:alpine
</code></pre><p><img src="https://static001.geekbang.org/resource/image/9f/51/9f2b16fb58dbe0a358e26042565f9851.png?wh=1920x238" alt="图片"></p><p>有了Nginx的反向代理之后，我们就可以打开浏览器，输入本机的“127.0.0.1”或者是虚拟机的IP地址（我这里仍然是“<a href="http://192.168.10.208">http://192.168.10.208</a>”），看到WordPress的界面：</p><p><img src="https://static001.geekbang.org/resource/image/73/f4/735552be9cf6d45ac41a001252ayyef4.png?wh=1524x1858" alt="图片"></p><p>你也可以在Kubernetes里使用命令 <code>kubectl logs</code> 查看WordPress、MariaDB等Pod的运行日志，来验证它们是否已经正确地响应了请求：</p><p><img src="https://static001.geekbang.org/resource/image/84/62/8498c598e6f3142d490218601acdbc62.png?wh=1920x809" alt="图片"></p><h2>使用Dashboard管理Kubernetes</h2><p>到这里WordPress网站就搭建成功了，我们的主要任务也算是完成了，不过我还想再带你看看Kubernetes的图形管理界面，也就是Dashboard，看看不用命令行该怎么管理Kubernetes。</p><p>启动Dashboard的命令你还记得吗，在第10节课里讲插件的时候曾经说过，需要用minikube，命令是：</p><pre><code class="language-plain">minikube dashboard
</code></pre><p>它会自动打开浏览器界面，显示出当前Kubernetes集群里的工作负载：</p><p><img src="https://static001.geekbang.org/resource/image/53/59/536eeb176a7737c9ed815c10af0fcf59.png?wh=1920x1022" alt="图片"></p><p>点击任意一个Pod的名字，就会进入管理界面，可以看到Pod的详细信息，而右上角有4个很重要的功能，分别可以查看日志、进入Pod内部、编辑Pod和删除Pod，相当于执行 <code>logs</code>、<code>exec</code>、<code>edit</code>、<code>delete</code> 命令，但要比命令行要直观友好的多：</p><p><img src="https://static001.geekbang.org/resource/image/d5/28/d5e5131bfb1d6aae2f026177bf283628.png?wh=1920x781" alt="图片"></p><p>比如说，我点击了第二个按钮，就会在浏览器里开启一个Shell窗口，直接就是Pod的内部Linux环境，在里面可以输入任意的命令，无论是查看状态还是调试都很方便：</p><p><img src="https://static001.geekbang.org/resource/image/46/4c/466c67a48616c946505242d0796ed74c.png?wh=1820x1240" alt="图片"></p><p>ConfigMap/Secret等对象也可以在这里任意查看或编辑：</p><p><img src="https://static001.geekbang.org/resource/image/de/22/defyybc05ed793b7966e1f6b68018022.png?wh=1312x976" alt="图片"></p><p>Dashboard里的可操作的地方还有很多，这里我只是一个非常简单的介绍。虽然你也许已经习惯了使用键盘和命令行，但偶尔换一换口味，改用鼠标和图形界面来管理Kubernetes也是件挺有意思的事情，有机会不妨尝试一下。</p><h2>小结</h2><p>好了，作为“初级篇”的最后一节课，今天我们回顾了一下Kubernetes的知识要点，我还是画一份详细的思维导图，帮助你课后随时复习总结。</p><p><img src="https://static001.geekbang.org/resource/image/87/1f/87a1d338340c8ca771a97d0fyy4b611f.jpg?wh=1920x1877" alt="图片"></p><p>这节课里我们使用Kubernetes搭建了WordPress网站，和第7讲里的Docker比较起来，我们应用了容器编排技术，以“声明式”的YAML来描述应用的状态和它们之间的关系，而不会列出详细的操作步骤，这就降低了我们的心智负担——调度、创建、监控等杂事都交给Kubernetes处理，我们只需“坐享其成”。</p><p>虽然我们朝着云原生的方向迈出了一大步，不过现在我们的容器编排还不够完善，Pod的IP地址还必须手工查找填写，缺少自动的服务发现机制，另外对外暴露服务的方式还很原始，必须要依赖集群外部力量的帮助。</p><p>所以，我们的学习之旅还将继续，在接下来的“中级篇”里，会开始研究更多的API对象，来解决这里遇到的问题。</p><h2>课下作业</h2><p>最后是课下作业时间，给你留两个动手题：</p><ol>
<li>MariaDB、WordPress现在用的是ConfigMap，能否改用Secret来实现呢？</li>
<li>你能否把Nginx代理转换成Pod的形式，让它在Kubernetes里运行呢？</li>
</ol><p>期待能看到你动手体验后的想法，如果觉得有帮助，欢迎分享给自己身边的朋友一起学习。</p><p>下节课就是视频演示的操作课了，我们下节课再见。</p><p><img src="https://static001.geekbang.org/resource/image/3c/ea/3c3036bc56bb9ec14598342e56c11bea.jpg?wh=1920x1544" alt="图片"></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/cd/ba/3a348f2d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>YueShi</span>
  </div>
  <div class="_2_QraFYR_0">Mac上或者Window上，可能出现访问不到问题，这里写了一篇排查问题的思路，期望能帮到各位老板<br><br>https:&#47;&#47;juejin.cn&#47;post&#47;7127679053242302477</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-04 09:46:34</div>
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
  <div class="_2_QraFYR_0">思考题 2，<br><br>试着写出了 Pod 的 YAML 文件<br><br>apiVersion: v1<br>kind: ConfigMap<br>metadata:<br>  name: ngx-cm<br><br>data:<br>  default.conf: |-<br>    server {<br>      listen 80;<br>      default_type text&#47;html;<br><br>      location &#47; {<br>        proxy_http_version 1.1;<br>        proxy_set_header Host $host;<br>        proxy_pass http:&#47;&#47;127.0.0.1:8080;<br>      }<br>    }<br><br>---<br><br>apiVersion: v1<br>kind: Pod<br>metadata:<br>  name: ngx-pod<br>  labels:<br>    app: nginx-alpine<br>    role: website<br><br>spec:<br>  volumes:<br>    - name: conf<br>      configMap:<br>        name: ngx-cm<br><br>  containers:<br>  - volumeMounts:<br>    - mountPath: &#47;etc&#47;nginx&#47;conf.d&#47;<br>      name: conf<br><br>    image: nginx:alpine<br>    name: ngx-pod<br>    imagePullPolicy: IfNotPresent<br>    ports:<br>    - containerPort: 80<br><br>可以成功创建 Pod，nginx 的配置文件也被挂载到指定的位置，但是浏览器还是无法访问 wordpress，不知道是不是我的 YAML 文件中少了什么🤔，还望老师指点</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: YAML写的非常好，不过Nginx配置文件里的proxy_pass，地址不能是127.0.0.1，必须是WordPress Pod的地址。<br><br>还有因为Nginx pod运行在Kubernetes里，是与外界隔离的，必须要用port-forward外界才能看到，可以参考WordPress的做法。<br><br>但这方法比较麻烦，到中级篇我们学习了Service、Ingress就能够轻松解决了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-25 15:27:31</div>
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
  <div class="_2_QraFYR_0">学了这一节，没感觉到容器编排的好用，反而遇到了一个问题，在配置中，我把wordpress链接数据库的host给小写了，结果一直告诉我链接数据库失败，我也没有排查手段，一直失败，我想的是，这个k8s的部署步骤实在是太复杂了，复杂到配置需要一个yaml，容器需要一个yaml，port-forward也需要一个yaml，我想如果是docker直接部署，其实就需要两条命令，所有参数都放到env中，或者放到配置文件中，也比检查yaml文件强，在入门篇中没感受到k8s的好用，只感觉到琐碎，是因为服务太小，还是因为没用好的原因呢<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Kubernetes面对的是大规模集群，其实这些都是可以用ci&#47;cd自动化的，我们现在的学习是从底层做起全手动操作，当然会麻烦一些，到后面会有更高级的对象，比如Deployment。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-28 23:24:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKQuaDVYz2jWN9vrmgbB785SNkmYmxf1xzEG8m8ku3ZvzYSkiaanTjqt7O47ficOQUpGmEy7eImplDw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>椰子</span>
  </div>
  <div class="_2_QraFYR_0">老师，目前Pod配置中使用了固定IP。如果pod被重启了，ip 是否会变化？如果有变化，这样配置是否有点不合理？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，会变化，到后面的中级篇会用Service对象来解决，这里是从docker到Kubernetes的一个过渡阶段。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-25 09:21:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/83/af/1cb42cd3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>马以</span>
  </div>
  <div class="_2_QraFYR_0">写一下第2题吧解题步骤：<br>首先如果我们把docker 形式改成pod形式这里存在两个问题（1）网络问题，（2）nginx配置文件加载问题，<br>第一个问题k8s的插件会处理节点内以及节点间网络通信问题，这个我们这里暂时不需要考虑，<br>那么需要我们处理的就是配置文件问题：这里我们可以用多个方法处理<br>   a: pod挂载宿主机目录，把配置文件放到宿主机目录下，容器启动挂载<br>   b: 使用我们学过的configMap<br>   c：使用pv、pvc 这个后面老师应该会讲到<br>这里我门就使用第二种方法，也是评论里使用最多的方法：<br>yml文件如下：<br>    ngx-cm.yml<br><br>apiVersion: v1<br>kind: ConfigMap<br>metadata:<br>  name: ngx-cm<br>data:<br>  nginx.conf: |<br>    server {<br>      listen 8080;<br>      #default_type text&#47;html;<br><br>      location &#47; {<br>        proxy_http_version 1.1;<br>        proxy_set_header Host $host;<br>        proxy_pass http:&#47;&#47;172.17.0.9:80;<br>      }<br>    }<br><br>nginx.yml:<br><br>apiVersion: v1<br>kind: Pod<br>metadata:<br>  name: ngx-pod<br>  labels:<br>    env: demo<br>    owner: ant<br><br>spec:<br>  volumes:<br>    - name: nginx-conf<br>      configMap:<br>        name: ngx-cm<br>        items:<br>        - key: nginx.conf<br>          path: default.conf<br>  containers:<br>  - image: nginx:alpine<br>    name: ngx<br>    volumeMounts:<br>      - name: nginx-conf<br>        mountPath: &#47;etc&#47;nginx&#47;conf.d<br>    ports:<br>    - containerPort: 8080<br><br>然后使用 kubectl apply -f nginx.yml 创建pod，但是这个时候由于k8s节点还是一个隔离环境，所以我们还是无法访问，<br>所以我们要在网页观察到具体的网页内容，还是要再用docker创建一层nginx代理，server 内容如下：<br>server {<br>  listen 80;<br>  default_type text&#47;html;<br><br>  location &#47; {<br>      proxy_http_version 1.1;<br>      proxy_set_header Host $host;<br>      proxy_pass http:&#47;&#47;127.0.0.1:8888;<br>  }<br>}<br><br>使用 docker run -d --rm --net=host -v &#47;home&#47;ant&#47;kubernetes&#47;pod&#47;nginx-conf&#47;nginx.conf:&#47;etc&#47;nginx&#47;conf.d&#47;default.conf nginx:alpine<br>启动，<br>然后使用 kubectl port-forward ngx-pod 8888:8080 &amp; 做转发<br><br>访问 http:&#47;&#47;192.168.56.208&#47; 正常访问</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: awesome！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-12 17:07:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/14/e7/c534fab3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>wcy</span>
  </div>
  <div class="_2_QraFYR_0">作业1：改用secret可以，这个简单，把data里对应的值用base64编码就行，对应pod的yml改下<br>作业2：nginx改成pod 运行是成功的， 但是port-forward出来，linux虚机上用firefox打开页面是ok的。但在shell终端本地浏览器访问不了，看了下应该是port-forward的问题，映射的端口绑定在Linux虚机127.0.0.1上，所以virtualbox转发不出来。<br>nginx cm:<br>apiVersion: v1<br>kind: ConfigMap<br>metadata:<br>  name: ngx-cm<br>data:<br>  nginx.conf: |<br>    server {<br>    #  listen 80;<br>       listen 8080;<br>    #   default_type text&#47;html;<br><br>       location &#47; {<br>           proxy_http_version 1.1;<br>           proxy_set_header Host $host;<br>           proxy_pass http:&#47;&#47;172.17.0.10:80;<br>       }<br>     }<br><br>nginx pod配置：<br>apiVersion: v1<br>kind: Pod<br>metadata:<br>  name: wp-ngx-pod<br>  labels:<br>    env: test<br><br>spec:<br>  containers:<br>  - image: nginx:alpine<br>    name: wp-ngx<br>    ports:<br>    - containerPort: 8080<br>    volumeMounts:<br>    - mountPath: &#47;etc&#47;nginx&#47;conf.d&#47;<br>      name: myngxconf<br><br>  volumes:<br>  - name: myngxconf<br>    configMap:<br>      name: ngx-cm<br>      items:<br>      - key: nginx.conf<br>        path: default.conf<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-04 11:42:35</div>
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
  <div class="_2_QraFYR_0">有在操作过程中遇到几个问题，还烦请老师指点<br><br>1、前面 2 个步骤中，在 kubectl apply -f *-pod.yml 之前是不是也得 apply 一下之前定义的 configMap 对象？<br><br>2、对于反向代理，不是特别的理解，老师能简单解释一下反向代理，并附带说明这里 Nginx 这里作为反向代理的目的是什么吗？<br><br>3、`minikube dashboard` 无法在命令行显示界面，我将 URL 复制粘贴到 Ubuntu 虚拟机图形界面上的浏览器中，但回复的 JSON 中显示 404？不太清楚有没有更好的方法<br>Opening http:&#47;&#47;127.0.0.1:38539&#47;api&#47;v1&#47;namespaces&#47;kubernetes-dashboard&#47;services&#47;http:kubernetes-dashboard:&#47;proxy&#47; in your default browser...<br>Error: no DISPLAY environment variable specified<br><br>4、对于思考题的 2，我创建一个了 nginx Pod，然后将其配置文件通过 kubectl cp 拷贝到 Pod 容器中的对应位置，但是发现并不起作用，本来想着重启 Pod 让配置生效，但是并没有找到对应的指令。这里是不是需要其他的 K8S API 对象的帮助？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1. ConfigMap已经含在pod yaml里了，是用“---”分隔的多个对象，所以一个文件就可以创建ConfigMap和pod。<br><br>2.反向代理就是在真正网站（WordPress）前的一个代理软件，提供安全防护等功能，可参考《透视HTTP协议》。不加这个其实也可以，主要的目的是模拟真实环境，增加一点部署难度。<br><br>3. minikube dashboard会自动启动浏览器，可以加-h看它的详细用法，也可以粘贴url。<br><br>4.要把配置文件用ConfigMap的形式加载，拷贝不行。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-25 03:41:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>edward</span>
  </div>
  <div class="_2_QraFYR_0">kubectl port-forward 老师请教下 这个命令设置端口转发后，怎么查看和取消？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 因为命令后面有个一个“&amp;”，所以就转入了后台运行，可以用ps + grep来查看，用“fg”带到前台，Ctrl+C停止，或者看到进程号后直接kill。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-08 18:33:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/7c/cf/d5382404.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>RRR</span>
  </div>
  <div class="_2_QraFYR_0">两个问题<br><br>1. port-forward 实际在生产用的多么？应该都是 ingress 吧<br>2. 老师后面能讲讲 helm 和 Istio 吗，他们和 k8s 关系是什么，我看不明白</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1. port-forward只能用于测试，生产环境当然要用Ingress<br><br>2.helm可以理解成是Kubernetes的安装包，istio是运行在Kubernetes上的服务网格。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-27 15:54:51</div>
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
  <div class="_2_QraFYR_0">请教老师几个问题：<br>Q1：MariaDB和WP可以跑在一个POD里面吗？<br><br>Q2：文中的prefix是什么意思？<br>文中定义mariadb-pod.yml的时候，有一个prefix: &#39;MARIADB_&#39;， 这是什么意思？<br>是把configMap中的变量名字前面加上这个前缀作为pod中的变量名称吗？<br>即：pod中的变量名称 = MARIADB_&#39; + configMap中的变量名称。<br>Q3：关于转发的疑问<br>文中有一句：“因为 WordPress 网站使用了 URL 重定向，直接使用“8080”会导致跳转故障”。 如果没有nginx，宿主机上用浏览器直接访问WP的8080端口，则会因为WP内部实现机制而产生错误； 如果加上nginx，nginx去访问WP的8080端口，就没事。 是这样吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.当然可以，Pod里的多容器可以任意组合，但一般做法都是一个应用一个Pod<br><br>2.是的，参考入门篇，MariaDB要求环境变量必须这么命名。<br><br>3.是的，这样外界用的就是80端口，转发的地址端口就不会乱。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-25 22:54:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_39bdd5</span>
  </div>
  <div class="_2_QraFYR_0">kubectl get pod -o wide查看的时候ip显示是none是怎么回事<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 也许是网络插件没有正常工作？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-10 15:42:58</div>
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
  <div class="_2_QraFYR_0">老师，docker run -d --rm \ --net=host ，这里的--net=host 起什么作用啊？加了以后，直接拒绝访问了，之前不都是会端口映射用来访问的吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: --net=host是让docker用本机的网络，不是bridge端口映射，前面应该讲过了。<br><br>如果是拒绝访问，应该找其他的原因。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-02 15:38:36</div>
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
  <div class="_2_QraFYR_0">第二题根据老师的回复修改了下，但是还是连不上<br><br>virtual-machine:~&#47;k8s-testing&#47;wordpress$ cat ngx-pod.yml<br>apiVersion: v1<br>kind: ConfigMap<br>metadata:<br>  name: ngx-cm<br><br>data:<br>  default.conf: |-<br>    server {<br>      listen 80;<br>      default_type text&#47;html;<br><br>      location &#47; {<br>        proxy_http_version 1.1;<br>        proxy_set_header Host $host;<br>        proxy_pass http:&#47;&#47;172.17.0.6:80;<br>      }<br>    }<br><br>---<br><br>apiVersion: v1<br>kind: Pod<br>metadata:<br>  name: ngx-pod<br>  labels:<br>    app: nginx-alpine<br>    role: website<br><br>spec:<br>  volumes:<br>    - name: conf<br>      configMap:<br>        name: ngx-cm<br><br>  containers:<br>  - volumeMounts:<br>    - mountPath: &#47;etc&#47;nginx&#47;conf.d&#47;<br>      name: conf<br><br>    image: nginx:alpine<br>    name: ngx-pod<br>    imagePullPolicy: IfNotPresent<br>    ports:<br>    - containerPort: 80<br><br>port-forward 那里我取消了 wp-pod 的 port-forward, 原因是这里是直接从 ngx-pod bypass 到 wp-pod，所以不需要 forward 外界的端口。但是 ngx-pod 在容器中，还是需要的，指令如下<br><br>kubectl port-forward ngx-pod 8080:80 &amp;<br><br>不知道为什么，从 Mac 浏览器就是连不上。我也进到 ngx-pod 上检查了，配置文件什么的都和上面定义的一样。还烦请老师在帮忙看看 😂<br><br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: port-forward 应该只能本机访问，可以用curl验证一下。它的功能很弱，只能用来测试，要被外界访问还是得用docker启一个Nginx反向代理才行。<br><br>这道题只要动手操作过就行了，没必要一定要出结果，因为后面会有更好的Kubernetes处理方式。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-29 00:17:47</div>
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
  <div class="_2_QraFYR_0">第二题：改是能改，但是就是连不上<br>已经参考了老师给phyhou的回复中的对Nginx配置修改，接着是kubectl port-forward &lt;nginx&gt; 8080:80 &amp;，结果还是连不上<br><br>完整代码：<br>#Nginx.yaml<br>apiVersion: v1<br><br>kind: ConfigMap<br><br>metadata:<br><br>  name: wpma-ngx-cm<br><br><br><br>data:<br><br>  default.conf: |-<br><br>    server {<br><br>      listen 80;<br><br>      default_type text&#47;html;<br><br><br><br>      location &#47; {<br><br>          proxy_http_version 1.1;<br><br>          proxy_set_header Host $host;<br><br>          proxy_pass http:&#47;&#47;172.17.0.6:8080; &#47;&#47;wordpress ip<br><br>      }<br><br>    }<br><br><br><br>---<br><br><br><br>apiVersion: v1<br><br>kind: Pod<br><br>metadata:<br><br>  labels:<br><br>    run: wpma-ngx<br><br>  name: wpma-ngx<br><br><br><br>spec:<br><br>  containers:<br><br>  - image: nginx:alpine<br><br>    name: wpma-ngx<br><br>    imagePullPolicy: IfNotPresent<br><br>    ports:<br><br>    - containerPort: 80<br><br><br><br>    volumeMounts:<br><br>      - name: wpma-ngx-cm<br><br>        mountPath: &quot;etc&#47;nginx&#47;conf.d&quot;<br><br>  volumes:<br><br>    - name: wpma-ngx-cm<br><br>      configMap:<br><br>        name: wpma-ngx-cm</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Nginx配置文件里除了要用WordPress的IP地址，端口号也要是WordPress的端口，也就是80。<br><br>之前在docker用的8080，是因为port-forward映射的是本地端口号8080。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-28 08:15:42</div>
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
  <div class="_2_QraFYR_0">终于能把网站给弄起来了，开心。<br><br>第一题：改成用secret，很简单，就是把老师的示例里的configMapRef改成secretRef<br>完整代码：<br>#maria secret<br><br>apiVersion: v1<br><br>data:<br><br>  DATABASE: ZGI=<br><br>  PASSWORD: MTIz<br><br>  ROOT_PASSWORD: MTIz<br><br>  USER: d3A=<br><br>kind: Secret<br><br>metadata:<br><br>  creationTimestamp: null<br><br>  name: maria-sc<br><br><br><br>---<br><br><br><br>apiVersion: v1<br><br>kind: Pod<br><br>metadata:<br><br>  labels:<br><br>    app: wordpress<br><br>    role: database<br><br>  name: mariasc-pod<br><br><br><br>spec:<br><br>  containers:<br><br>  - image: mariadb:10<br><br>    name: maria<br><br>    imagePullPolicy: IfNotPresent<br><br>    ports:<br><br>    - containerPort: 3306<br><br>  <br><br>    envFrom:<br><br>    - prefix: &#39;MARIADB_&#39;<br><br>      secretRef:<br><br>        name: maria-sc<br><br><br>#WordPress secret<br>apiVersion: v1<br><br>data:<br><br>  HOST: MTcyLjE3LjAuNw==<br><br>  NAME: ZGI=<br><br>  PASSWORD: MTIz<br><br>  USER: d3A=<br><br>kind: Secret<br><br>metadata:<br><br>  creationTimestamp: null<br><br>  name: wp-sc<br><br><br><br>---<br><br><br><br>apiVersion: v1<br><br>kind: Pod<br><br>metadata:<br><br>  labels:<br><br>    app: wordpress<br><br>    role: website<br><br>  name: wpsc-pod<br><br><br><br>spec:<br><br>  containers:<br><br>  - image: wordpress:5<br><br>    name: wordpress<br><br>    imagePullPolicy: IfNotPresent<br><br>    ports:<br><br>    - containerPort: 80<br><br><br><br>    envFrom:<br><br>    - prefix: &#39;WORDPRESS_DB_&#39;<br><br>      secretRef:<br><br>        name: wp-sc<br><br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-28 06:50:33</div>
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
  <div class="_2_QraFYR_0">终于跑通了，感动</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-28 03:13:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/PiajxSqBRaEKSVuNarJuDhBSvHY0giaq6yriceEBKiaKuc04wCYWOuso50noqDexaPJJibJN7PHwvcQppnzsDia1icZkw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Matthew</span>
  </div>
  <div class="_2_QraFYR_0">执行 kubectl apply  -f mariadb-pod.yml 以后，POD的状态一直是 ContainerCreating，是什么原因？应该怎么处理呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 用kubectl describe 检查一下pod，可能是镜像没拉取下来。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-26 00:08:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/89/c8/53085ffb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>tzhiyu</span>
  </div>
  <div class="_2_QraFYR_0">运行下面这行命令之后<br>kubectl apply -f wp-pod.yml<br><br>没有出现老师图片上出现的&quot;configmap&#47;maria-cm created&quot;, 是因为我没有单独写maria-cm.yml导致的吗<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 看一下wp-pod.yml里面是否用“---”分隔包含了多个对象定义，如果是GitHub上的YAML 就应该没错。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-24 00:46:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Gtf3tMDmYRoCL9Eico52ciatacq4PUHfQ8icQIYKV6KLDAJyTa8ZnLXMKf05pEice5RnEagocFobca5zv8jwyPhNKA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_9d43c0</span>
  </div>
  <div class="_2_QraFYR_0">我有个疑问 mariadb启动的时候 拉取 在maria-cm里面声明的配置 这个配置的字段启动的时候是怎么找到的 因为我们在docker启动的时候是会显式的配置启动参数 如果加了统一的前缀不会影响mariadb启动找自己的配置吗 </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: kubernetes启动pod的时候会把configmap以环境变量的形式注入，其实和docker的机制是一样的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-13 21:21:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/db/f7/29cff40f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>星</span>
  </div>
  <div class="_2_QraFYR_0">老师好，请问k8s中的镜像下载地址如何配置呀？是&#47;etc&#47;docker&#47;daemon.json中的地址么？我在apply mariadb的时候，提示镜像下载失败。感觉好像用的不是docker的镜像地址。<br>我把yml中的镜像直接写成image: hub-mirror.c.163.com&#47;library&#47;mariadb:10，还是同样的问题。<br>Warning  Failed     93s (x3 over 4m40s)  kubelet            Failed to pull image &quot;mariadb:10&quot;: rpc error: code = Unknown desc = context canceled<br>  Warning  Failed     93s (x3 over 4m40s)  kubelet            Error: ErrImagePull</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不加指定域名默认就是从dockerhub上拉取，应该还是网络的原因吧，配置下载镜像可以找一些资料看看，我一般都是直接用的dockerhub，sorry。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-13 15:33:23</div>
  </div>
</div>
</div>
</li>
</ul>