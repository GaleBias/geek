<audio title="28｜应用保障：如何让Pod运行得更健康？" src="https://static001.geekbang.org/resource/audio/30/22/309d06feeyy1019834de4c69d1c70d22.mp3" controls="controls"></audio> 
<p>你好，我是Chrono。</p><p>在前面这么多节的课程中，我们都是在研究如何使用各种API对象来管理、操作Pod，而对Pod本身的关注却不是太多。</p><p>作为Kubernetes里的核心概念和原子调度单位，Pod的主要职责是管理容器，以逻辑主机、容器集合、进程组的形式来代表应用，它的重要性是不言而喻的。</p><p>那么今天我们回过头来，在之前那些上层API对象的基础上，一起来看看在Kubernetes里配置Pod的两种方法：资源配额Resources、检查探针Probe，它们能够给Pod添加各种运行保障，让应用运行得更健康。</p><h2>容器资源配额</h2><p>早在<a href="https://time.geekbang.org/column/article/528640">第2讲</a>的时候我们就说过，创建容器有三大隔离技术：namespace、cgroup、chroot。其中的namespace实现了独立的进程空间，chroot实现了独立的文件系统，但唯独没有看到cgroup的具体应用。</p><p>cgroup的作用是管控CPU、内存，保证容器不会无节制地占用基础资源，进而影响到系统里的其他应用。</p><p>不过，容器总是要使用CPU和内存的，该怎么处理好需求与限制这两者之间的关系呢？</p><p>Kubernetes的做法与我们在<a href="https://time.geekbang.org/column/article/542376">第24讲</a>里提到的PersistentVolumeClaim用法有些类似，就是容器需要先提出一个“书面申请”，Kubernetes再依据这个“申请”决定资源是否分配和如何分配。</p><!-- [[[read_end]]] --><p>但是CPU、内存与存储卷有明显的不同，因为它是直接“内置”在系统里的，不像硬盘那样需要“外挂”，所以申请和管理的过程也就会简单很多。</p><p>具体的申请方法很简单，<strong>只要在Pod容器的描述部分添加一个新字段 <code>resources</code> 就可以了</strong>，它就相当于申请资源的 <code>Claim</code>。</p><p>来看一个YAML示例：</p><pre><code class="language-yaml">apiVersion: v1
kind: Pod
metadata:
&nbsp; name: ngx-pod-resources

spec:
&nbsp; containers:
&nbsp; - image: nginx:alpine
&nbsp; &nbsp; name: ngx

&nbsp; &nbsp; resources:
&nbsp; &nbsp; &nbsp; requests:
&nbsp; &nbsp; &nbsp; &nbsp; cpu: 10m
&nbsp; &nbsp; &nbsp; &nbsp; memory: 100Mi
&nbsp; &nbsp; &nbsp; limits:
&nbsp; &nbsp; &nbsp; &nbsp; cpu: 20m
&nbsp; &nbsp; &nbsp; &nbsp; memory: 200Mi
</code></pre><p>这个YAML文件定义了一个Nginx Pod，我们需要重点学习的是 <code>containers.resources</code>，它下面有两个字段：</p><ul>
<li>“<strong>requests</strong>”，意思是容器要申请的资源，也就是说要求Kubernetes在创建Pod的时候必须分配这里列出的资源，否则容器就无法运行。</li>
<li>“<strong>limits</strong>”，意思是容器使用资源的上限，不能超过设定值，否则就有可能被强制停止运行。</li>
</ul><p>在请求 <code>cpu</code> 和 <code>memory</code> 这两种资源的时候，你需要特别注意它们的表示方式。</p><p>内存的写法和磁盘容量一样，使用 <code>Ki</code>、<code>Mi</code>、<code>Gi</code> 来表示 <code>KB</code>、<code>MB</code>、<code>GB</code>，比如 <code>512Ki</code>、<code>100Mi</code>、<code>0.5Gi</code> 等。</p><p>而CPU因为在计算机中数量有限，非常宝贵，所以Kubernetes允许容器精细分割CPU，即可以1个、2个地完整使用CPU，也可以用小数0.1、0.2的方式来部分使用CPU。这其实是效仿了UNIX“时间片”的用法，意思是进程最多可以占用多少CPU时间。</p><p>不过CPU时间也不能无限分割，<strong>Kubernetes里CPU的最小使用单位是0.001，为了方便表示用了一个特别的单位 <code>m</code></strong>，也就是“milli”“毫”的意思，比如说500m就相当于0.5。</p><p>现在我们再来看这个YAML，你就应该明白了，它向系统申请的是1%的CPU时间和100MB的内存，运行时的资源上限是2%CPU时间和200MB内存。有了这个申请，Kubernetes就会在集群中查找最符合这个资源要求的节点去运行Pod。</p><p>下面是我在<a href="https://www.freecodecamp.org/news/how-to-leverage-the-power-of-kubernetes-to-optimise-your-hosting-costs-c2e168a232a2/">网上</a>找的一张动图，Kubernetes会根据每个Pod声明的需求，像搭积木或者玩俄罗斯方块一样，把节点尽量“塞满”，充分利用每个节点的资源，让集群的效益最大化。</p><p><img src="https://static001.geekbang.org/resource/image/39/91/397bfabd8234f8d859ca877a58f0d191.gif?wh=800x765" alt="图片"></p><p>你可能会有疑问：如果Pod不写 <code>resources</code> 字段，Kubernetes会如何处理呢？</p><p>这就意味着Pod对运行的资源要求“既没有下限，也没有上限”，Kubernetes不用管CPU和内存是否足够，可以把Pod调度到任意的节点上，而且后续Pod运行时也可以无限制地使用CPU和内存。</p><p>我们课程里是实验环境，这样做是当然是没有问题的，但如果是生产环境就很危险了，Pod可能会因为资源不足而运行缓慢，或者是占用太多资源而影响其他应用，所以我们应当合理评估Pod的资源使用情况，尽量为Pod加上限制。</p><p>看到这里估计你会继续追问：如果预估错误，Pod申请的资源太多，系统无法满足会怎么样呢？</p><p>让我们来试一下吧，先删除Pod的资源限制 <code>resources.limits</code>，把 <code>resources.request.cpu</code> 改成比较极端的“10”，也就是要求10个CPU：</p><pre><code class="language-yaml">  ...
  
&nbsp; &nbsp; resources:
&nbsp; &nbsp; &nbsp; requests:
&nbsp; &nbsp; &nbsp; &nbsp; cpu: 10
</code></pre><p>然后使用 <code>kubectl apply</code> 创建这个Pod，你可能会惊奇地发现，虽然我们的Kubernetes集群里只有3个CPU，但Pod也能创建成功。</p><p>不过我们再用 <code>kubectl get pod</code> 去查看的话，就会发现它处于“Pending”状态，实际上并没有真正被调度运行：</p><p><img src="https://static001.geekbang.org/resource/image/b1/d4/b1154e089533df5cfabc18c7e9c442d4.png?wh=1380x176" alt="图片"></p><p>使用命令 <code>kubectl describe</code> 来查看具体原因，会发现有这么一句提示：</p><p><img src="https://static001.geekbang.org/resource/image/95/02/9577c36e53c723b8e28ddb2d5e77e502.png?wh=1920x156" alt="图片"></p><p>这就很明确地告诉我们Kubernetes调度失败，当前集群里的所有节点都无法运行这个Pod，因为它要求的CPU实在是太多了。</p><h2>什么是容器状态探针</h2><p>现在，我们使用 <code>resources</code> 字段加上资源配额之后，Pod在Kubernetes里的运行就有了初步保障，Kubernetes会监控Pod的资源使用情况，让它既不会“饿死”也不会“撑死”。</p><p>但这只是最初级的运行保障，如果你开发或者运维过实际的后台服务就会知道，一个程序即使正常启动了，它也有可能因为某些原因无法对外提供服务。其中最常见的情况就是运行时发生“死锁”或者“死循环”的故障，这个时候从外部来看进程一切都是正常的，但内部已经是一团糟了。</p><p>所以，我们还希望Kubernetes这个“保姆”能够更细致地监控Pod的状态，除了保证崩溃重启，还必须要能够探查到Pod的内部运行状态，定时给应用做“体检”，让应用时刻保持“健康”，能够满负荷稳定工作。</p><p>那应该用什么手段来检查应用的健康状态呢？</p><p>因为应用程序各式各样，对于外界来说就是一个<strong>黑盒子</strong>，只能看到启动、运行、停止这三个基本状态，此外就没有什么好的办法来知道它内部是否正常了。</p><p>所以，我们必须把应用变成<strong>灰盒子</strong>，让部分内部信息对外可见，这样Kubernetes才能够探查到内部的状态。</p><p>这么说起来，检查的过程倒是有点像现在我们很熟悉的核酸检测，Kubernetes用一根小棉签在应用的“检查口”里提取点数据，就可以从这些信息来判断应用是否“健康”了，这项功能也就被形象地命名为“<strong>探针</strong>”（Probe），也可以叫“探测器”。</p><p>Kubernetes为检查应用状态定义了三种探针，它们分别对应容器不同的状态：</p><ul>
<li><strong>Startup</strong>，启动探针，用来检查应用是否已经启动成功，适合那些有大量初始化工作要做，启动很慢的应用。</li>
<li><strong>Liveness</strong>，存活探针，用来检查应用是否正常运行，是否存在死锁、死循环。</li>
<li><strong>Readiness</strong>，就绪探针，用来检查应用是否可以接收流量，是否能够对外提供服务。</li>
</ul><p>你需要注意这三种探针是递进的关系：应用程序先启动，加载完配置文件等基本的初始化数据就进入了Startup状态，之后如果没有什么异常就是Liveness存活状态，但可能有一些准备工作没有完成，还不一定能对外提供服务，只有到最后的Readiness状态才是一个容器最健康可用的状态。</p><p>初次接触这三种状态可能有点难理解，我画了一张图，你可以看一下状态与探针的对应关系：</p><p><img src="https://static001.geekbang.org/resource/image/ea/84/eaff5e640171984a4b1b2285982ee184.jpg?wh=1920x1000" alt="图片"></p><p>那Kubernetes具体是如何使用状态和探针来管理容器的呢？</p><p>如果一个Pod里的容器配置了探针，<strong>Kubernetes在启动容器后就会不断地调用探针来检查容器的状态</strong>：</p><ul>
<li>如果Startup探针失败，Kubernetes会认为容器没有正常启动，就会尝试反复重启，当然其后面的Liveness探针和Readiness探针也不会启动。</li>
<li>如果Liveness探针失败，Kubernetes就会认为容器发生了异常，也会重启容器。</li>
<li>如果Readiness探针失败，Kubernetes会认为容器虽然在运行，但内部有错误，不能正常提供服务，就会把容器从Service对象的负载均衡集合中排除，不会给它分配流量。</li>
</ul><p>知道了Kubernetes对这三种状态的处理方式，我们就可以在开发应用的时候编写适当的检查机制，让Kubernetes用“探针”定时为应用做“体检”了。</p><p>在刚才图的基础上，我又补充了Kubernetes的处理动作，看这张图你就能很好地理解容器探针的工作流程了：</p><p><img src="https://static001.geekbang.org/resource/image/64/d9/64fde55dd2eab68f9968ff34218646d9.jpg?wh=1920x1200" alt="图片"></p><h2>如何使用容器状态探针</h2><p>掌握了资源配额和检查探针的概念，我们进入今天的高潮部分，看看如何在Pod的YAML描述文件里定义探针。</p><p>startupProbe、livenessProbe、readinessProbe这三种探针的配置方式都是一样的，关键字段有这么几个：</p><ul>
<li><strong>periodSeconds</strong>，执行探测动作的时间间隔，默认是10秒探测一次。</li>
<li><strong>timeoutSeconds</strong>，探测动作的超时时间，如果超时就认为探测失败，默认是1秒。</li>
<li><strong>successThreshold</strong>，连续几次探测成功才认为是正常，对于startupProbe和livenessProbe来说它只能是1。</li>
<li><strong>failureThreshold</strong>，连续探测失败几次才认为是真正发生了异常，默认是3次。</li>
</ul><p>至于探测方式，Kubernetes支持3种：Shell、TCP Socket、HTTP GET，它们也需要在探针里配置：</p><ul>
<li><strong>exec</strong>，执行一个Linux命令，比如ps、cat等等，和container的command字段很类似。</li>
<li><strong>tcpSocket</strong>，使用TCP协议尝试连接容器的指定端口。</li>
<li><strong>httpGet</strong>，连接端口并发送HTTP GET请求。</li>
</ul><p>要使用这些探针，我们必须要在开发应用时预留出“检查口”，这样Kubernetes才能调用探针获取信息。这里我还是以Nginx作为示例，用ConfigMap编写一个配置文件：</p><pre><code class="language-yaml">apiVersion: v1
kind: ConfigMap
metadata:
&nbsp; name: ngx-conf

data:
&nbsp; default.conf: |
&nbsp; &nbsp; server {
&nbsp; &nbsp; &nbsp; listen 80;
&nbsp; &nbsp; &nbsp; location = /ready {
&nbsp; &nbsp; &nbsp; &nbsp; return 200 'I am ready';
&nbsp; &nbsp; &nbsp; }
&nbsp; &nbsp; }
</code></pre><p>你可能不是太熟悉Nginx的配置语法，我简单解释一下。</p><p>在这个配置文件里，我们启用了80端口，然后用 <code>location</code> 指令定义了HTTP路径 <code>/ready</code>，它作为对外暴露的“检查口”，用来检测就绪状态，返回简单的200状态码和一个字符串表示工作正常。</p><p>现在我们来看一下Pod里三种探针的具体定义：</p><pre><code class="language-yaml">apiVersion: v1
kind: Pod
metadata:
&nbsp; name: ngx-pod-probe

spec:
&nbsp; volumes:
&nbsp; - name: ngx-conf-vol
&nbsp; &nbsp; configMap:
&nbsp; &nbsp; &nbsp; name: ngx-conf

&nbsp; containers:
&nbsp; - image: nginx:alpine
&nbsp; &nbsp; name: ngx
&nbsp; &nbsp; ports:
&nbsp; &nbsp; - containerPort: 80
&nbsp; &nbsp; volumeMounts:
&nbsp; &nbsp; - mountPath: /etc/nginx/conf.d
&nbsp; &nbsp; &nbsp; name: ngx-conf-vol

&nbsp; &nbsp; startupProbe:
&nbsp; &nbsp; &nbsp; periodSeconds: 1
&nbsp; &nbsp; &nbsp; exec:
&nbsp; &nbsp; &nbsp; &nbsp; command: ["cat", "/var/run/nginx.pid"]

&nbsp; &nbsp; livenessProbe:
&nbsp; &nbsp; &nbsp; periodSeconds: 10
&nbsp; &nbsp; &nbsp; tcpSocket:
&nbsp; &nbsp; &nbsp; &nbsp; port: 80

&nbsp; &nbsp; readinessProbe:
&nbsp; &nbsp; &nbsp; periodSeconds: 5
&nbsp; &nbsp; &nbsp; httpGet:
&nbsp; &nbsp; &nbsp; &nbsp; path: /ready
&nbsp; &nbsp; &nbsp; &nbsp; port: 80
</code></pre><p>StartupProbe使用了Shell方式，使用 <code>cat</code> 命令检查Nginx存在磁盘上的进程号文件（/var/run/nginx.pid），如果存在就认为是启动成功，它的执行频率是每秒探测一次。</p><p>LivenessProbe使用了TCP Socket方式，尝试连接Nginx的80端口，每10秒探测一次。</p><p>ReadinessProbe使用的是HTTP GET方式，访问容器的 <code>/ready</code> 路径，每5秒发一次请求。</p><p>现在我们用 <code>kubectl apply</code> 创建这个Pod，然后查看它的状态：</p><p><img src="https://static001.geekbang.org/resource/image/ac/6c/ac6b405074a5e93d33dd7154f299486c.png?wh=1272x174" alt="图片"></p><p>当然，因为这个Nginx应用非常简单，它启动后探针的检查都会是正常的，你可以用 <code>kubectl logs</code> 来查看Nginx的访问日志，里面会记录HTTP GET探针的执行情况：</p><p><img src="https://static001.geekbang.org/resource/image/ed/6b/edf9fb3337bf3dd5a9b2fba8dfbc326b.png?wh=1920x527" alt="图片"></p><p>从截图中你可以看到，Kubernetes正是以大约5秒一次的频率，向URI <code>/ready</code> 发送HTTP请求，不断地检查容器是否处于就绪状态。</p><p>为了验证另两个探针的工作情况，我们可以修改探针，比如把命令改成检查错误的文件、错误的端口号：</p><pre><code class="language-yaml">    startupProbe:
      exec:
        command: ["cat", "nginx.pid"]  #错误的文件

    livenessProbe:
      tcpSocket:
        port: 8080                     #错误的端口号
</code></pre><p>然后我们重新创建Pod对象，观察它的状态。</p><p>当StartupProbe探测失败的时候，Kubernetes就会不停地重启容器，现象就是 <code>RESTARTS</code> 次数不停地增加，而livenessProbe和readinessProbePod没有执行，Pod虽然是Running状态，也永远不会READY：</p><p><img src="https://static001.geekbang.org/resource/image/90/7f/900468e4b86c241a53256584e514b47f.png?wh=1348x182" alt="图片"></p><p>因为failureThreshold的次数默认是三次，所以Kubernetes会连续执行三次livenessProbe TCP Socket探测，每次间隔10秒，30秒之后都失败才重启容器：</p><p><img src="https://static001.geekbang.org/resource/image/c3/e1/c31bf2cf6672c62ebd42f305534dbae1.png?wh=1366x178" alt="图片"></p><p>你也可以自己试着改一下readinessProbe，看看它失败时Pod会是什么样的状态。</p><h2>小结</h2><p>好了，今天我们学习了两种为Pod配置运行保障的方式：Resources和Probe。Resources就是为容器加上资源限制，而Probe就是主动健康检查，让Kubernetes实时地监控应用的运行状态。</p><p>再简单小结一下今天的内容：</p><ol>
<li>资源配额使用的是cgroup技术，可以限制容器使用的CPU和内存数量，让Pod合理利用系统资源，也能够让Kubernetes更容易调度Pod。</li>
<li>Kubernetes定义了Startup、Liveness、Readiness三种健康探针，它们分别探测应用的启动、存活和就绪状态。</li>
<li>探测状态可以使用Shell、TCP Socket、HTTP Get三种方式，还可以调整探测的频率和超时时间等参数。</li>
</ol><h2>课下作业</h2><p>最后是课下作业时间，给你留两个思考题：</p><ol>
<li>你能够解释一下Liveness和Readiness这两种探针的区别吗？</li>
<li>你认为Shell、TCP Socket、HTTP GET这三种探测方式各有什么优缺点？</li>
</ol><p>欢迎在下方留言区留言参与讨论，课程快要完结了，感谢你坚持学习了这么久。我们下节课再见。</p><p><img src="https://static001.geekbang.org/resource/image/5e/68/5eef65a1abf0cc4ff70c0e3df7a93168.jpg?wh=1920x2580" alt="图片"></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/99/87/98ebb20e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>dao</span>
  </div>
  <div class="_2_QraFYR_0">思考题：<br>1. <br>Liveness 和 Readiness 都是循环探测，Liveness 探测失败会重启，而 Readiness 探测失败不会重启，可以从 Pod 状态 看出重启次数。<br>两者都可以单独使用，这时差异不大。<br>如果同时使用两者，Liveness 主要是确认应用运行着或者说活着，而 Readiness 是确认应用提供着服务或者说服务就绪着（可以接收流量）。<br><br>2. <br>Shell 是从容器内部探测，TCP Socket 和 HTTP GET 都是在容器外部探测。 TCP Socket 基于端口的探测，端口打开即成功；HTTP GET 更丰富些，可以是端口 + 路径。<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-25 11:48:03</div>
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
  <div class="_2_QraFYR_0">老师请问，Startup、Liveness、Readiness三种探针是按顺序执行还是并行呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Startup成功之后才能执行后两个探针，后面两个是并行的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-26 12:13:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/0c/e1/f663213e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>拾掇拾掇</span>
  </div>
  <div class="_2_QraFYR_0">Shell：不用知道端口；TCP Socket、HTTP GET这2个都得知道端口，用的时候还得显示调用端口吧</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，可以参考一些知名镜像的YAML ，看看它们是怎么做健康检查的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-12 13:11:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>邵涵</span>
  </div>
  <div class="_2_QraFYR_0">如老师原文所示，在startupProbe或livenessProbe探测失败之后，pod的status初始都是running。不过，在容器重启几次之后，pod的status会变为CrashLoopBackOff<br><br>如果startupProbe和livenessProbe探测成功，readinessProbe探测失败，pod的ready一直是0&#47;1，status一直是running，当然，也不会重启<br>NAME            READY   STATUS    RESTARTS   AGE<br>ngx-pod-probe   0&#47;1     Running   0          27m</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-25 11:02:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/45/a9/3d48d6a2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Lorry</span>
  </div>
  <div class="_2_QraFYR_0">Liveness指的是进程是否存在，Readiness则是指是否能够正常提供服务，所以可以使用tcp&#39;协议检测Liveness，进程存在端口即存在，使用http检测服务，只有服务启动了参能够相应请求。<br><br>Shell是系统内进程级别交互，所以只能够本地访问，Tcp可以跨机器访问，但是访问的级别比较低，不能够获得顶层数据，Http是协议层数据，可以拿到应用层的服务信息，但是关注顶层信息，底层故障，顶层无法提供有价值信息了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-03 12:59:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_2ce074</span>
  </div>
  <div class="_2_QraFYR_0">老师 tcpSocket探测是由谁发起的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 当然是Kubernetes，不能内部具体如何运转的就没细研究了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-30 16:47:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/cd/ba/3a348f2d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>YueShi</span>
  </div>
  <div class="_2_QraFYR_0">&quot;postStart&quot;&#39;&quot;preStop&quot;感觉可以做CICD，或者各种webhock钩子，或者简单的通知</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，这些都是Kubernetes为我们提供的回调接口。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-31 18:20:44</div>
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
  <div class="_2_QraFYR_0">请教老师几个问题：<br>Q1：第27讲中，创建4个nginx实例，没有端口冲突问题吗？<br>四个nginx实例，两个在master，两个在worker。同一台机器上有两个pod，而pod的定义是一样的，即端口相同，那么，不存在端口冲突问题吗？<br><br>Q2：第27讲中，创建四个nginx实例后，不能访问nginx的欢迎页。<br>service也创建了。 用虚拟机上浏览器来访问127.0.0.1，失败，<br>执行“kubectl port-forward svc&#47;ngx-svc 8080:80 &amp;”以后，<br>浏览器上访问127.0.0.1:8080，报错：<br>E0826 14:32:28.734987   42769 portforward.go:391] error copying from local connection to remote stream: read tcp4 127.0.0.1:8080-&gt;127.0.0.1:59752: read: connection reset by peer<br><br>Q3：nginx的配置文件中，竖线是什么意思？<br>data: default.conf: |， 这里的竖线是什么意思？ <br><br>Q4：操作系统能够看到的CPU，是指逻辑核吗？还是时间片？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.每个Nginx都是运行在pod里，被容器环境隔离，当然不会冲突，可以复习一下入门篇。<br><br>2. 这个暂时没看出是什么原因。<br><br>3.竖线是YAML 的长字符串语法，看前面的课程。<br><br>4.是逻辑核。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-26 15:26:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/96/5a/45a56b3a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>WenjieXu</span>
  </div>
  <div class="_2_QraFYR_0">老师，如果一个pod需要加入到service中，是否意味着必须要配置readinessProbe？还是默认不配置的话，k8s会认为是up的，放到对应service的ep对象里？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 探针不是必须的，如果没有探针kubernetes就不会检查，直接认为是ready。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-22 22:49:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_60e02d</span>
  </div>
  <div class="_2_QraFYR_0">请教下Pod启动后，什么时候会加入service的负载均衡列表？是startup probe成功后吗？然后readyness probe失败后，会从service中移除，那么，是不是说，startup probe成功到readyness失败期间，流量会进入这个没有ready的Pod呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我的理解是这样的，想要知道更准确的答案就要再去查资料了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-25 14:47:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJZicRP0FZ78kT68wEGeWzPnxrF4s3Ea36XdMA2pj2TAbU3eibVt7KqzS5B7LbWMhRfSc3XEUL3Hrjw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>liubiqianmoney</span>
  </div>
  <div class="_2_QraFYR_0">Cgroup除了限制CPU和内存资源外，可以限制磁盘IOPS吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-30 13:54:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/60/a1/8f003697.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>静心</span>
  </div>
  <div class="_2_QraFYR_0">终于有人把三种probe给讲清楚了，很赞</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: thanks.</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-11 11:30:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/28/1b/f9/018197f1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小江爱学术</span>
  </div>
  <div class="_2_QraFYR_0">老师，有一个疑问，这些探针请求是由k8s发起的，是不是都是在容器外部对容器进行访问的呢，还是说实际上是由容器内自己负责对自己进行健康检查。<br>，</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 探针是外部发起请求，而检查检查的逻辑必须是容器内部的应用来处理。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-09 09:02:58</div>
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
  <div class="_2_QraFYR_0">老师好，我这个yml文件使用探针，直接给我显示&#47;var&#47;run&#47;nginx.pid timeout，但是实际上又是running状态，所以不太理解，请教一下老师是什么情况<br>apiVersion: apps&#47;v1<br>kind: Deployment<br>metadata:<br>  name: nginx-deployment<br>  annotations:<br>    kubernetes.io&#47;change-cause: update to v18 ngx=latest<br>spec:<br>  selector:<br>    matchLabels:<br>      app: nginx<br>  replicas: 2<br>  minReadySeconds: 15<br>  template:<br>    metadata:<br>      labels:<br>        app: nginx<br>    spec:<br>      containers:<br>      - name: nginx<br>        image: nginx:latest<br>        ports:<br>        - containerPort: 80<br>        volumeMounts:<br>        - name: nginx-config<br>          mountPath: &#47;etc&#47;nginx&#47;conf.d<br>        resources:<br>          requests:<br>            cpu: 1m<br>            memory: 100Mi<br>          limits:<br>            cpu: 1m<br>            memory: 200Mi<br>        startupProbe:<br>          periodSeconds: 1<br>          exec:<br>            command: [&quot;cat&quot;, &quot;&#47;var&#47;run&#47;nginx.pid&quot;]<br><br>        livenessProbe:<br>          periodSeconds: 10<br>          tcpSocket:<br>            port: 80<br><br>        readinessProbe:<br>          periodSeconds: 5<br>          httpGet:<br>            path: &#47;ready<br>            port: 80<br>      volumes:<br>      - name: nginx-config<br>        configMap:<br>          name: ngx-conf </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 探针只是一个shell命令，不会出现timeout吧，用logs看看日志具体显示的是什么。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-29 16:08:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/ec/f6/f615ed26.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>一路小跑</span>
  </div>
  <div class="_2_QraFYR_0">请教老师2个问题啊：<br>1.如果还想追加几个自定义的健康检查的类型，应该从哪个方面入手好呢?<br>2. 另外，针对健康检查失败的处理预案，可以自定义么，需要学习哪个API对象？（比如用go做一些定制）</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这应该算是内部的实现机制了，没有细研究过，会go语言的话可以看源码。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-25 20:51:38</div>
  </div>
</div>
</div>
</li>
</ul>