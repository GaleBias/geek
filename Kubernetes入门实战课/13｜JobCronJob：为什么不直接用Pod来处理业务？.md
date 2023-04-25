<audio title="13｜JobCronJob：为什么不直接用Pod来处理业务？" src="https://static001.geekbang.org/resource/audio/38/9d/38f4559a190f877034538b6eccb4ef9d.mp3" controls="controls"></audio> 
<p>你好，我是Chrono。</p><p>在上次的课里我们学习了Kubernetes的核心对象Pod，用来编排一个或多个容器，让这些容器共享网络、存储等资源，总是共同调度，从而紧密协同工作。</p><p>因为Pod比容器更能够表示实际的应用，所以Kubernetes不会在容器层面来编排业务，而是把Pod作为在集群里调度运维的最小单位。</p><p>前面我们也看到了一张Kubernetes的资源对象关系图，以Pod为中心，延伸出了很多表示各种业务的其他资源对象。那么你会不会有这样的疑问：Pod的功能已经足够完善了，为什么还要定义这些额外的对象呢？为什么不直接在Pod里添加功能，来处理业务需求呢？</p><p>这个问题体现了Google对大规模计算集群管理的深度思考，今天我就说说Kubernetes基于Pod的设计理念，先从最简单的两种对象——Job和CronJob讲起。</p><h2>为什么不直接使用Pod</h2><p>现在你应该知道，Kubernetes使用的是RESTful API，把集群中的各种业务都抽象为HTTP资源对象，那么在这个层次之上，我们就可以使用面向对象的方式来考虑问题。</p><p>如果你有一些编程方面的经验，就会知道面向对象编程（OOP），它把一切都视为高内聚的对象，强调对象之间互相通信来完成任务。</p><!-- [[[read_end]]] --><p>虽然面向对象的设计思想多用于软件开发，但它放到Kubernetes里却意外地合适。因为Kubernetes使用YAML来描述资源，把业务简化成了一个个的对象，内部有属性，外部有联系，也需要互相协作，只不过我们不需要编程，完全由Kubernetes自动处理（其实Kubernetes的Go语言内部实现就大量应用了面向对象）。</p><p>面向对象的设计有许多基本原则，其中有两条我认为比较恰当地描述了Kubernetes对象设计思路，一个是“<strong>单一职责</strong>”，另一个是“<strong>组合优于继承</strong>”。</p><p>“单一职责”的意思是对象应该只专注于做好一件事情，不要贪大求全，保持足够小的粒度才更方便复用和管理。</p><p>“组合优于继承”的意思是应该尽量让对象在运行时产生联系，保持松耦合，而不要用硬编码的方式固定对象的关系。</p><p>应用这两条原则，我们再来看Kubernetes的资源对象就会很清晰了。因为Pod已经是一个相对完善的对象，专门负责管理容器，那么我们就不应该再“画蛇添足”地盲目为它扩充功能，而是要保持它的独立性，容器之外的功能就需要定义其他的对象，把Pod作为它的一个成员“组合”进去。</p><p>这样每种Kubernetes对象就可以只关注自己的业务领域，只做自己最擅长的事情，其他的工作交给其他对象来处理，既不“缺位”也不“越位”，既有分工又有协作，从而以最小成本实现最大收益。</p><h2>为什么要有Job/CronJob</h2><p>现在我们来看看Kubernetes里的两种新对象：Job和CronJob，它们就组合了Pod，实现了对离线业务的处理。</p><p>上次课讲Pod的时候我们运行了两个Pod：Nginx和busybox，它们分别代表了Kubernetes里的两大类业务。一类是像Nginx这样长时间运行的“<strong>在线业务</strong>”，另一类是像busybox这样短时间运行的“<strong>离线业务</strong>”。</p><p>“在线业务”类型的应用有很多，比如Nginx、Node.js、MySQL、Redis等等，一旦运行起来基本上不会停，也就是永远在线。</p><p>而“离线业务”类型的应用也并不少见，它们一般不直接服务于外部用户，只对内部用户有意义，比如日志分析、数据建模、视频转码等等，虽然计算量很大，但只会运行一段时间。“离线业务”的特点是<strong>必定会退出</strong>，不会无期限地运行下去，所以它的调度策略也就与“在线业务”存在很大的不同，需要考虑运行超时、状态检查、失败重试、获取计算结果等管理事项。</p><p>而这些业务特性与容器管理没有必然的联系，如果由Pod来实现就会承担不必要的义务，违反了“单一职责”，所以我们应该把这部分功能分离到另外一个对象上实现，让这个对象去控制Pod的运行，完成附加的工作。</p><p>“离线业务”也可以分为两种。一种是“<strong>临时任务</strong>”，跑完就完事了，下次有需求了说一声再重新安排；另一种是“<strong>定时任务</strong>”，可以按时按点周期运行，不需要过多干预。</p><p>对应到Kubernetes里，“临时任务”就是API对象<strong>Job</strong>，“定时任务”就是API对象<strong>CronJob</strong>，使用这两个对象你就能够在Kubernetes里调度管理任意的离线业务了。</p><p>由于Job和CronJob都属于离线业务，所以它们也比较相似。我们先学习通常只会运行一次的Job对象以及如何操作。</p><h3>如何使用YAML描述Job</h3><p>Job的YAML“文件头”部分还是那几个必备字段，我就不再重复解释了，简单说一下：</p><ul>
<li>apiVersion不是 <code>v1</code>，而是 <code>batch/v1</code>。</li>
<li>kind是 <code>Job</code>，这个和对象的名字是一致的。</li>
<li>metadata里仍然要有 <code>name</code> 标记名字，也可以用 <code>labels</code> 添加任意的标签。</li>
</ul><p>如果记不住这些也不要紧，你还可以使用命令 <code>kubectl explain job</code> 来看它的字段说明。不过想要生成YAML样板文件的话不能使用 <code>kubectl run</code>，因为 <code>kubectl run</code> 只能创建Pod，要创建Pod以外的其他API对象，需要使用命令 <code>kubectl create</code>，再加上对象的类型名。</p><p>比如用busybox创建一个“echo-job”，命令就是这样的：</p><pre><code class="language-bash">export out="--dry-run=client -o yaml"              # 定义Shell变量
kubectl create job echo-job --image=busybox $out
</code></pre><p>会生成一个基本的YAML文件，保存之后做点修改，就有了一个Job对象：</p><pre><code class="language-yaml">apiVersion: batch/v1
kind: Job
metadata:
&nbsp; name: echo-job

spec:
&nbsp; template:
&nbsp; &nbsp; spec:
&nbsp; &nbsp; &nbsp; restartPolicy: OnFailure
&nbsp; &nbsp; &nbsp; containers:
&nbsp; &nbsp; &nbsp; - image: busybox
&nbsp; &nbsp; &nbsp; &nbsp; name: echo-job
&nbsp; &nbsp; &nbsp; &nbsp; imagePullPolicy: IfNotPresent
&nbsp; &nbsp; &nbsp; &nbsp; command: ["/bin/echo"]
&nbsp; &nbsp; &nbsp; &nbsp; args: ["hello", "world"]
</code></pre><p>你会注意到Job的描述与Pod很像，但又有些不一样，主要的区别就在“spec”字段里，多了一个 <code>template</code> 字段，然后又是一个“spec”，显得有点怪。</p><p>如果你理解了刚才说的面向对象设计思想，就会明白这种做法的道理。它其实就是在Job对象里应用了组合模式，<code>template</code> 字段定义了一个“<strong>应用模板</strong>”，里面嵌入了一个Pod，这样Job就可以从这个模板来创建出Pod。</p><p>而这个Pod因为受Job的管理控制，不直接和apiserver打交道，也就没必要重复apiVersion等“头字段”，只需要定义好关键的 <code>spec</code>，描述清楚容器相关的信息就可以了，可以说是一个“无头”的Pod对象。</p><p>为了辅助你理解，我把Job对象重新组织了一下，用不同的颜色来区分字段，这样你就能够很容易看出来，其实这个“echo-job”里并没有太多额外的功能，只是把Pod做了个简单的包装：</p><p><img src="https://static001.geekbang.org/resource/image/9b/28/9b780905a824d2103d4ayyc79267ae28.jpg?wh=1920x2141" alt="图片"></p><p>总的来说，这里的Pod工作非常简单，在 <code>containers</code> 里写好名字和镜像，<code>command </code>执行 <code>/bin/echo</code>，输出“hello world”。</p><p>不过，因为Job业务的特殊性，所以我们还要在 <code>spec</code> 里多加一个字段 <code>restartPolicy</code>，确定Pod运行失败时的策略，<code>OnFailure</code> 是失败原地重启容器，而 <code>Never</code> 则是不重启容器，让Job去重新调度生成一个新的Pod。</p><h3>如何在Kubernetes里操作Job</h3><p>现在让我们来创建Job对象，运行这个简单的离线作业，用的命令还是 <code>kubectl apply</code>：</p><pre><code class="language-plain">kubectl apply -f job.yml
</code></pre><p>创建之后Kubernetes就会从YAML的模板定义中提取Pod，在Job的控制下运行Pod，你可以用 <code>kubectl get job</code>、<code>kubectl get pod</code> 来分别查看Job和Pod的状态：</p><pre><code class="language-plain">kubectl get job
kubectl get pod
</code></pre><p><img src="https://static001.geekbang.org/resource/image/33/55/33ac80cb9f5dd91d1affc133e56efc55.png?wh=1382x368" alt="图片"></p><p>可以看到，因为Pod被Job管理，它就不会反复重启报错了，而是会显示为 <code>Completed</code> 表示任务完成，而Job里也会列出运行成功的作业数量，这里只有一个作业，所以就是 <code>1/1</code>。</p><p>你还可以看到，Pod被自动关联了一个名字，用的是Job的名字（echo-job）再加上一个随机字符串（pb5gh），这当然也是Job管理的“功劳”，免去了我们手工定义的麻烦，这样我们就可以使用命令 <code>kubectl logs</code> 来获取Pod的运行结果：</p><p><img src="https://static001.geekbang.org/resource/image/81/b5/81224cedf0acf209b746a1162d09b3b5.png?wh=1114x118" alt="图片"></p><p>到这里，你可能会觉得，经过了Job、Pod对容器的两次封装，虽然从概念上很清晰，但好像并没有带来什么实际的好处，和直接跑容器也差不了多少。</p><p>其实Kubernetes的这套YAML描述对象的框架提供了非常多的灵活性，可以在Job级别、Pod级别添加任意的字段来定制业务，这种优势是简单的容器技术无法相比的。</p><p>这里我列出几个控制离线作业的重要字段，其他更详细的信息可以参考Job文档：</p><ul>
<li><strong>activeDeadlineSeconds</strong>，设置Pod运行的超时时间。</li>
<li><strong>backoffLimit</strong>，设置Pod的失败重试次数。</li>
<li><strong>completions</strong>，Job完成需要运行多少个Pod，默认是1个。</li>
<li><strong>parallelism</strong>，它与completions相关，表示允许并发运行的Pod数量，避免过多占用资源。</li>
</ul><p>要注意这4个字段并不在 <code>template</code> 字段下，而是在 <code>spec</code> 字段下，所以它们是属于Job级别的，用来控制模板里的Pod对象。</p><p>下面我再创建一个Job对象，名字叫“sleep-job”，它随机睡眠一段时间再退出，模拟运行时间较长的作业（比如MapReduce）。Job的参数设置成15秒超时，最多重试2次，总共需要运行完4个Pod，但同一时刻最多并发2个Pod：</p><pre><code class="language-yaml">apiVersion: batch/v1
kind: Job
metadata:
&nbsp; name: sleep-job

spec:
&nbsp; activeDeadlineSeconds: 15
&nbsp; backoffLimit: 2
&nbsp; completions: 4
&nbsp; parallelism: 2

&nbsp; template:
&nbsp; &nbsp; spec:
&nbsp; &nbsp; &nbsp; restartPolicy: OnFailure
&nbsp; &nbsp; &nbsp; containers:
&nbsp; &nbsp; &nbsp; - image: busybox
&nbsp; &nbsp; &nbsp; &nbsp; name: echo-job
&nbsp; &nbsp; &nbsp; &nbsp; imagePullPolicy: IfNotPresent
&nbsp; &nbsp; &nbsp; &nbsp; command:
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; - sh
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; - -c
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; - sleep $(($RANDOM % 10 + 1)) &amp;&amp; echo done
</code></pre><p>使用 <code>kubectl apply</code> 创建Job之后，我们可以用 <code>kubectl get pod -w</code> 来实时观察Pod的状态，看到Pod不断被排队、创建、运行的过程：</p><pre><code class="language-plain">kubectl apply -f sleep-job.yml
kubectl get pod -w
</code></pre><p><img src="https://static001.geekbang.org/resource/image/7d/b7/7d413a0c38065de2063a99e7df2b7eb7.png?wh=1591x1328" alt="图片"></p><p>等到4个Pod都运行完毕，我们再用 <code>kubectl get</code> 来看看Job和Pod的状态：</p><p><img src="https://static001.geekbang.org/resource/image/58/46/58b99356c811bd377acfa4cb921d2446.png?wh=1426x542" alt="图片"></p><p>就会看到Job的完成数量如同我们预期的是4，而4个Pod也都是完成状态。</p><p>显然，“声明式”的Job对象让离线业务的描述变得非常直观，简单的几个字段就可以很好地控制作业的并行度和完成数量，不需要我们去人工监控干预，Kubernetes把这些都自动化实现了。</p><h2>如何使用YAML描述CronJob</h2><p>学习了“临时任务”的Job对象之后，再学习“定时任务”的CronJob对象也就比较容易了，我就直接使用命令 <code>kubectl create</code> 来创建CronJob的样板。</p><p>要注意两点。第一，因为CronJob的名字有点长，所以Kubernetes提供了简写 <code>cj</code>，这个简写也可以使用命令 <code>kubectl api-resources</code> 看到；第二，CronJob需要定时运行，所以我们在命令行里还需要指定参数 <code>--schedule</code>。</p><pre><code class="language-bash">export out="--dry-run=client -o yaml"              # 定义Shell变量
kubectl create cj echo-cj --image=busybox --schedule="" $out
</code></pre><p>然后我们编辑这个YAML样板，生成CronJob对象：</p><pre><code class="language-yaml">apiVersion: batch/v1
kind: CronJob
metadata:
&nbsp; name: echo-cj

spec:
&nbsp; schedule: '*/1 * * * *'
&nbsp; jobTemplate:
&nbsp; &nbsp; spec:
&nbsp; &nbsp; &nbsp; template:
&nbsp; &nbsp; &nbsp; &nbsp; spec:
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; restartPolicy: OnFailure
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; containers:
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; - image: busybox
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; name: echo-cj
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; imagePullPolicy: IfNotPresent
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; command: ["/bin/echo"]
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; args: ["hello", "world"]
</code></pre><p>我们还是重点关注它的 <code>spec</code> 字段，你会发现它居然连续有三个 <code>spec</code> 嵌套层次：</p><ul>
<li>第一个 <code>spec</code> 是CronJob自己的对象规格声明</li>
<li>第二个 <code>spec</code> 从属于“jobTemplate”，它定义了一个Job对象。</li>
<li>第三个 <code>spec</code> 从属于“template”，它定义了Job里运行的Pod。</li>
</ul><p>所以，CronJob其实是又组合了Job而生成的新对象，我还是画了一张图，方便你理解它的“套娃”结构：</p><p><img src="https://static001.geekbang.org/resource/image/yy/3c/yy352c661ae37dd116dd12c61932b43c.jpg?wh=1920x2206" alt="图片"></p><p>除了定义Job对象的“<strong>jobTemplate</strong>”字段之外，CronJob还有一个新字段就是“<strong>schedule</strong>”，用来定义任务周期运行的规则。它使用的是标准的Cron语法，指定分钟、小时、天、月、周，和Linux上的crontab是一样的。像在这里我就指定每分钟运行一次，格式具体的含义你可以课后参考Kubernetes官网文档。</p><p>除了名字不同，CronJob和Job的用法几乎是一样的，使用 <code>kubectl apply</code> 创建CronJob，使用 <code>kubectl get cj</code>、<code>kubectl get pod</code> 来查看状态：</p><pre><code class="language-plain">kubectl apply -f cronjob.yml
kubectl get cj
kubectl get pod
</code></pre><p><img src="https://static001.geekbang.org/resource/image/b0/2c/b00fdd8541372fb7a4de00de5ac6342c.png?wh=1644x484" alt="图片"></p><h2>小结</h2><p>好了，今天我们以面向对象思想分析了一下Kubernetes里的资源对象设计，它强调“职责单一”和“对象组合”，简单来说就是“对象套对象”。</p><p>通过这种嵌套方式，Kubernetes里的这些API对象就形成了一个“控制链”：</p><p>CronJob使用定时规则控制Job，Job使用并发数量控制Pod，Pod再定义参数控制容器，容器再隔离控制进程，进程最终实现业务功能，层层递进的形式有点像设计模式里的Decorator（装饰模式），链条里的每个环节都各司其职，在Kubernetes的统一指挥下完成任务。</p><p>小结一下今天的内容：</p><ol>
<li>Pod是Kubernetes的最小调度单元，但为了保持它的独立性，不应该向它添加多余的功能。</li>
<li>Kubernetes为离线业务提供了Job和CronJob两种API对象，分别处理“临时任务”和“定时任务”。</li>
<li>Job的关键字段是 <code>spec.template</code>，里面定义了用来运行业务的Pod模板，其他的重要字段有 <code>completions</code>、<code>parallelism</code> 等</li>
<li>CronJob的关键字段是 <code>spec.jobTemplate</code> 和 <code>spec.schedule</code>，分别定义了Job模板和定时运行的规则。</li>
</ol><h2>课下作业</h2><p>最后是课下作业时间，给你留两个思考题：</p><ol>
<li>你是怎么理解Kubernetes组合对象的方式的？它带来了什么好处？</li>
<li>Job和CronJob的具体应用场景有哪些？能够解决什么样的问题？</li>
</ol><p>欢迎在留言区分享你的疑问和学习心得，如果觉得有收获，也欢迎你分享给朋友一起学习。</p><p>下节课见。</p><p><img src="https://static001.geekbang.org/resource/image/59/7f/597caae147ec2a1852151878fc47ed7f.jpg?wh=1920x2402" alt="图片"></p>
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
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/2xoGmvlQ9qfSibVpPJyyaEriavuWzXnuECrJITmGGHnGVuTibUuBho43Uib3Y5qgORHeSTxnOOSicxs0FV3HGvTpF0A/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>psoracle</span>
  </div>
  <div class="_2_QraFYR_0">回答一下今天的作业：<br>1. 你是怎么理解 Kubernetes 组合对象的方式的？它带来了什么好处？<br>Kubernetes中组合对象，类似于面向对象编程中的继承，即不破坏父对象的功能，又扩展了自己领域场景中的功能，在API层面也简单了，只需要处理自己扩展的功能即可，比在一个对象上做加法进入逻辑判断要优雅很多。<br><br>2. Job 和 CronJob 的具体应用场景有哪些？能够解决什么样的问题？<br>Job与CronJob分别对应一次性调用的任务与周期性定时任务；前者任务只运行一次，比如用在手工触发的场景如数据库备份、恢复与还原，数据同步，安全检查，巡检等；后者用于定时任务，非手工触发，由CronJobController每隔10s遍历需要执行的CronJob，同样也使用在如数据库备份、恢复与还原、数据同步、安全检查、定期巡检以及所有周期性的运维任务。<br>Job与CronJob解决了任务的管理，如执行超时、失败尝试、执行数量与并行数量、任务结果记录等等，方便对任务执行的监控与管理；另外，Pod解决了批处理任务关联打包统一调度，容器解决了任务运行时环境。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-20 10:04:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_b9dad2</span>
  </div>
  <div class="_2_QraFYR_0">课外小贴士里第4条和第6条感觉是有冲突的，这个怎么理解呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 一个是Job，一个是CronJob，而且CronJob是能够控制Job的，官大一级压死人。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-20 13:29:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/0c/86/8e52afb8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>花花大脸猫</span>
  </div>
  <div class="_2_QraFYR_0">不得不服这个设计，为后续扩展带来了无限的可能，而且又不影响现有的pod体系功能！原来面向对象的思想还能在YAML中这么用。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个正是Kubernetes能够战胜swarm、mesos的根本原因。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-29 11:48:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/d0/51/f1c9ae2d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Christopher</span>
  </div>
  <div class="_2_QraFYR_0">终于知道老师昵称的由来了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 有那么一点关系，笑。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-20 10:36:12</div>
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
  <div class="_2_QraFYR_0">我想补充一下关于job配置的一个细节，大家可能复制黏贴罗老师的配置所以不会发现这个问题。<br>job.spec.containers.template.spec.containers.image是不能指定镜像版本号的，只能指定镜像：完整的镜像:版本号只能由pod定义，否则会从互联网拉取镜像，如果能联网当然没事，离线环境会直接报错无法拉取镜像，虽然你本地确实存在该版本的镜像且imagePullPolicy设置为Never或IfNotPresent。<br>比如我是离线环境，job里image配置为：- image: busybox:1.35.0，那么就会报错无法拉取镜像。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个小细节确实比较重要，因为我是一直联网，也没太注意。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-28 11:03:17</div>
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
  <div class="_2_QraFYR_0">Command 用双引号里写命令，不能有空格。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-20 11:01:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/06/7e/735968e2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>西门吹牛</span>
  </div>
  <div class="_2_QraFYR_0">组合的方式能少写很多代码，Java 很多中间件都这么搞，组合优于继承，基于接口而非实现编程，自由组合</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-25 19:39:06</div>
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
  <div class="_2_QraFYR_0">思考题：<br>1. 大的组件有小的组件拼接而成，这样做的其中一个好处就是低耦合，每个组件都是独立的个体，去操作一个组件时不需要理会其他组件的具体内部细节，直接拼接在一起即可。这么做也非常易于维护，比如 Job 中想要更换 Pod 也不需要更改 Job 本身的一些属性<br><br>2. Job 主要用在一些 one-off 的场景，就是需要去处理一些临时的一次性的情况，比如 service 的 setup，文件的构建等等。而 Cronjob 主要用途是去完成一些需要定期更新的任务，比如 一些 daily 的 pipeline，定时的检查，检验系统安全等等<br><br><br>有个问题请教老师，在 sleep-job 的那个例子中，不太理解为什么有时候在 Job 完成后，其中的一个 Pod 会被 Terminate 掉，然后最后只剩 3 个 Pod？用课程中的 YAML 文件试了几次，这种情况有时会发生有时不会，不知道是不是跟 YAML 中 Job 设置的字段有关？<br><br>:~&#47;k8s-testing$ kubectl get pod<br>NAME              READY   STATUS        RESTARTS   AGE<br>echo-job-qqq9k    0&#47;1     Completed     0          7m23s<br>ngx               1&#47;1     Running       0          11d<br>sleep-job-9ktxs   0&#47;1     Completed     0          17s<br>sleep-job-sndgm   0&#47;1     Completed     0          17s<br>sleep-job-tpbw7   1&#47;1     Terminating   0          9s<br>sleep-job-v8x8s   0&#47;1     Completed     0          12s</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 遇到异常的情况可以用describe看看pod，也许能找到原因。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-23 01:59:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/10/bb/f1061601.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Demon.Lee</span>
  </div>
  <div class="_2_QraFYR_0">最近在把业务服务迁移到 Kubernetes 上部署，其中就有一个定时服务，包含 30 个定时 job。我想着直接搬到 Kubernetes 的 CronJob 上来，这样开发团队就少维护一个第三方的开源框架了（用于定时任务调度）。<br><br>但我发现一个问题，导致迁移不了，就是触发频率：kubernetes cronjob 只支持到分钟，不能到秒级调度，即最高是每分钟运行一次任务；但他们的定时任务，有些是每 10 秒运行一次，15 秒一次，或 30 秒一次。<br><br>这种分钟内的调度，搞不定，感觉非常遗憾。<br><br>我在想，标准的 cron 只支持到分钟级别，那么分钟级别以内的定时调度呢？可能就是常驻进程了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 分钟以下级别的还真不能用crontab来调度，得想另外的办法了，比如用shell+sleep。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-29 17:21:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/1d/de/62bfa83f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>aoe</span>
  </div>
  <div class="_2_QraFYR_0">原来一直奇怪为什么有那么多spec、template不停的嵌套？今天终于明白了：不同层级自己描述自己的，相互不影响，不合陌生人说话</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，就是对象套对象的组合，把它转换成图形也许会好理解一些。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-10 12:36:11</div>
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
  <div class="_2_QraFYR_0">1. 不太理解。<br>2. cronjob很好理解，定时任务，备份数据库，跑批啥的，就常用。job是一次性任务，一次性数据拷贝。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good，可以参考其他同学的回答。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-25 18:16:12</div>
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
  <div class="_2_QraFYR_0">cronjob适合增量数据同步，job是一次性的，适合一次性启动停止的任务。<br>想到一个问题，cronjob里面的pod任务自己又起了定时任务，正好两个任务冲突了咋办（套娃警告）？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个就不是kubernetes的问题了，属于业务逻辑问题。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-21 17:13:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/e5/ce/2978a69a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>油菜花</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，请问问一下定时任务触发完成之后会有一个status为completed状态的pod，这个不用自动删除吗？不删除会有什么影响呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 没影响，参考小贴士</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-21 11:46:29</div>
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
  <div class="_2_QraFYR_0">请问下云原生生态体系中有没有关于Job与CronJob管理的项目可以推荐下，能够对任务有一个可视化的统一管理平台？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 自带dashboard ，好像还有个lens</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-21 11:31:17</div>
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
  <div class="_2_QraFYR_0">对于cronjob有些疑惑，在实际业务中，定时任务一般按照定时规则执行相应的业务逻辑；如果把我们传统的定时任务业务服务打成镜像，然后在k8s创建一个cronjob对象，这个cronjob对象只会根据schedule定期创建容器，至于容器内部执行的定时逻辑好像和cronjob并没有直接的关系。换句话说，cronjob只能定期帮你启动服务，启动后，服务内部的事情我不管</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 那是当然的，它就相当于Linux里的crontab，作用也是一样的，只负责定时启动任务。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-20 17:50:09</div>
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
  <div class="_2_QraFYR_0">请教老师两个问题：<br>Q1：echo-job的YAML文件中有两个同样的name。<br>一个是： metadata:   name: echo-job<br>另外一个是：spec:&#47;template:&#47;spec:&#47;containers: name:echo-job<br>这两个name有关系吗？<br><br>Q2：CronJob例子，get pod为什么是3个？<br>文章中get pod的输出结果是3个pod，我的虚拟机中也是输出3个，<br>为什么是3个？cronjob.yml中的内容看不出来哪一项是和POD的数量有关</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1. 没有关系，是kubectl create的时候自动使用了同样的名字，完全可以不一样，我是懒得改，也没有必要改。<br><br>2.默认只保留最近的三次任务，见小贴士。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-20 17:33:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/51/9b/ccea47d9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>安迪密恩</span>
  </div>
  <div class="_2_QraFYR_0">1. 组合优于继承， 解耦；<br>2. 应用场景举例：Job -》历史全量数据导入；CronJob -》每天的增量数据导入。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-20 15:06:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/44/d8/708a0932.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>李一</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，请教个问题，像k8s的cronJob在实际项目中，会替代一些现有项目的定时任务库吗？比如spring job或者quartz</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个得看具体的业务情况了，不能简单地替换。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-20 09:34:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/9b/64/dadf0ca5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>至尊猴</span>
  </div>
  <div class="_2_QraFYR_0">hi Chrono 老师，<br>我在自己环境做sleep-job这个场景的测试时，有job不能complete，请问这种情况怎么定位分析呢？<br><br><br>[wangfeng@fedora kube]$ kubectl apply -f sleep-job.yml <br>job.batch&#47;sleep-job created<br>[wangfeng@fedora kube]$ kubectl get pod -w<br>NAME              READY   STATUS      RESTARTS   AGE<br>echo-job-8vvfz    0&#47;1     Completed   0          13h<br>ngx               1&#47;1     Running     0          19h<br>sleep-job-fsnk4   0&#47;1     Completed   0          7s<br>sleep-job-l656p   1&#47;1     Running     0          7s<br>sleep-job-zfl66   1&#47;1     Running     0          4s<br>sleep-job-zfl66   0&#47;1     Completed   0          7s<br>sleep-job-hqkwk   0&#47;1     Pending     0          0s<br>sleep-job-hqkwk   0&#47;1     Pending     0          0s<br>sleep-job-zfl66   0&#47;1     Completed   0          7s<br>sleep-job-hqkwk   0&#47;1     ContainerCreating   0          0s<br>sleep-job-l656p   0&#47;1     Completed           0          11s<br>sleep-job-l656p   0&#47;1     Completed           0          11s<br>sleep-job-hqkwk   1&#47;1     Running             0          2s<br>sleep-job-hqkwk   1&#47;1     Terminating         0          5s<br>sleep-job-hqkwk   1&#47;1     Terminating         0          5s<br>sleep-job-hqkwk   0&#47;1     Terminating         0          11s<br>sleep-job-hqkwk   0&#47;1     Terminating         0          11s<br>sleep-job-hqkwk   0&#47;1     Terminating         0          11s<br><br>--------------<br>[wangfeng@fedora ~]$ kubectl get job<br>NAME        COMPLETIONS   DURATION   AGE<br>echo-job    1&#47;1           2s         13h<br>sleep-job   3&#47;4           6m31s      6m31s<br>[wangfeng@fedora ~]$ kubectl get pod<br>NAME              READY   STATUS      RESTARTS   AGE<br>echo-job-8vvfz    0&#47;1     Completed   0          13h<br>ngx               1&#47;1     Running     0          20h<br>sleep-job-fsnk4   0&#47;1     Completed   0          6m41s<br>sleep-job-l656p   0&#47;1     Completed   0          6m41s<br>sleep-job-zfl66   0&#47;1     Completed   0          6m38s<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 用describe 和log看看pod的情况试试。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-04 12:11:52</div>
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
  <div class="_2_QraFYR_0">parallelism: 2<br>我把这个定义了 4就可以成功看到pod和job是4个 其他都是2个或者3个 就是不成功</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-20 17:01:17</div>
  </div>
</div>
</div>
</li>
</ul>