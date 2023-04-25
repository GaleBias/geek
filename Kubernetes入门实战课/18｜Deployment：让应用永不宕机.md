<audio title="18｜Deployment：让应用永不宕机" src="https://static001.geekbang.org/resource/audio/c9/46/c9cf3df4cyy81662fde526270fc10d46.mp3" controls="controls"></audio> 
<p>你好，我是Chrono。</p><p>在上一节课里，我们使用kubeadm搭建了一个由两个节点组成的小型Kubernetes集群，比起单机的minikube，它更接近真实环境，在这里面做实验我们今后也更容易过渡到生产系统。</p><p>有了这个Kubernetes环境，接下来我们就在“初级篇”里学习的Pod知识基础上，深入研究一些由Pod衍生出来的其他API对象。</p><p>今天要看的API对象名字叫“<strong>Deployment</strong>”，顾名思义，它是专门用来部署应用程序的，能够让应用永不宕机，多用来发布无状态的应用，是Kubernetes里最常用也是最有用的一个对象。</p><h2>为什么要有Deployment</h2><p>在<a href="https://time.geekbang.org/column/article/531566">第13讲</a>里，我们学习了API对象Job和CronJob，它们代表了生产环境中的离线业务，通过对Pod的包装，向Pod添加控制字段，实现了基于Pod运行临时任务和定时任务的功能。</p><p>那么，除了“离线业务”，另一大类业务——也就是“在线业务”，在Kubernetes里应该如何处理呢？</p><p>我们先看看用Pod是否就足够了。因为它在YAML里使用“<strong>containers</strong>”就可以任意编排容器，而且还有一个“<strong>restartPolicy</strong>”字段，默认值就是 <code>Always</code>，可以监控Pod里容器的状态，一旦发生异常，就会自动重启容器。</p><!-- [[[read_end]]] --><p>不过，“restartPolicy”只能保证容器正常工作。不知你有没有想到，如果容器之外的Pod出错了该怎么办呢？比如说，有人不小心用 <code>kubectl delete</code> 误删了Pod，或者Pod运行的节点发生了断电故障，那么Pod就会在集群里彻底消失，对容器的控制也就无从谈起了。</p><p>还有我们也都知道，在线业务远不是单纯启动一个Pod这么简单，还有多实例、高可用、版本更新等许多复杂的操作。比如最简单的多实例需求，为了提高系统的服务能力，应对突发的流量和压力，我们需要创建多个应用的副本，还要即时监控它们的状态。如果还是只使用Pod，那就会又走回手工管理的老路，没有利用好Kubernetes自动化运维的优势。</p><p>其实，解决的办法也很简单，因为Kubernetes已经给我们提供了处理这种问题的思路，就是“单一职责”和“对象组合”。既然Pod管理不了自己，那么我们就再创建一个新的对象，由它来管理Pod，采用和Job/CronJob一样的形式——“对象套对象”。</p><p>这个用来管理Pod，实现在线业务应用的新API对象，就是Deployment。</p><h2>如何使用YAML描述Deployment</h2><p>我们先用命令 <code>kubectl api-resources</code> 来看看Deployment的基本信息：</p><pre><code class="language-plain">kubectl api-resources

NAME&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;SHORTNAMES&nbsp; &nbsp;APIVERSION&nbsp; &nbsp;NAMESPACED&nbsp; &nbsp;KIND
deployments&nbsp; deploy&nbsp; &nbsp; &nbsp; &nbsp;apps/v1&nbsp; &nbsp; &nbsp; true&nbsp; &nbsp; &nbsp; &nbsp; Deployment
</code></pre><p>从它的输出信息里可以知道，Deployment的简称是“<strong>deploy</strong>”，它的apiVersion是“<strong>apps/v1</strong>”，kind是“<strong>Deployment</strong>”。</p><p>所以，依据前面学习Pod、Job的经验，你就应该知道Deployment的YAML文件头该怎么写了：</p><pre><code class="language-yaml">apiVersion: apps/v1
kind: Deployment
metadata:
&nbsp; name: xxx-dep
</code></pre><p>当然了，我们还是可以使用命令 <code>kubectl create</code> 来创建Deployment的YAML样板，免去反复手工输入的麻烦。</p><p>创建Deployment样板的方式和Job也差不多，先指定类型是<strong>Deployment</strong>（简写<strong>deploy</strong>），然后是它的名字，再用 <code>--image</code> 参数指定镜像名字。</p><p>比如下面的这条命令，我就创建了一个名字叫 <code>ngx-dep</code> 的对象，使用的镜像是 <code>nginx:alpine</code>：</p><pre><code class="language-plain">export out="--dry-run=client -o yaml"
kubectl create deploy ngx-dep --image=nginx:alpine $out
</code></pre><p>得到的Deployment样板大概是下面的这个样子：</p><pre><code class="language-yaml">apiVersion: apps/v1
kind: Deployment
metadata:
&nbsp; labels:
&nbsp; &nbsp; app: ngx-dep
&nbsp; name: ngx-dep
&nbsp;&nbsp;
spec:
&nbsp; replicas: 2
&nbsp; selector:
&nbsp; &nbsp; matchLabels:
&nbsp; &nbsp; &nbsp; app: ngx-dep
&nbsp; &nbsp; &nbsp;&nbsp;
&nbsp; template:
&nbsp; &nbsp; metadata:
&nbsp; &nbsp; &nbsp; labels:
&nbsp; &nbsp; &nbsp; &nbsp; app: ngx-dep
&nbsp; &nbsp; spec:
&nbsp; &nbsp; &nbsp; containers:
&nbsp; &nbsp; &nbsp; - image: nginx:alpine
&nbsp; &nbsp; &nbsp; &nbsp; name: nginx
</code></pre><p>把它和Job/CronJob对比一下，你会发现有相似也有不同。相似的地方是都有“<strong>spec</strong>”“<strong>template</strong>”字段，“template”字段里也是一个Pod；不同的地方在于它的“spec”部分多了 <code>replicas</code>、<code>selector</code> 这两个新字段，聪明的你应该会猜到，这或许就会是Deployment特殊能力的根本。</p><p>没错，这两个新字段就是Deployment实现多实例、高可用等功能的关键所在。</p><h2>Deployment的关键字段</h2><p>先看 <code>replicas</code> 字段。它的含义比较简单明了，就是“副本数量”的意思，也就是说，指定要在Kubernetes集群里运行多少个Pod实例。</p><p>有了这个字段，就相当于为Kubernetes明确了应用部署的“期望状态”，Deployment对象就可以扮演运维监控人员的角色，自动地在集群里调整Pod的数量。</p><p>比如，Deployment对象刚创建出来的时候，Pod数量肯定是0，那么它就会根据YAML文件里的Pod模板，逐个创建出要求数量的Pod。</p><p>接下来Kubernetes还会持续地监控Pod的运行状态，万一有Pod发生意外消失了，数量不满足“期望状态”，它就会通过apiserver、scheduler等核心组件去选择新的节点，创建出新的Pod，直至数量与“期望状态”一致。</p><p>这里面的工作流程很复杂，但对于我们这些外部用户来说，设置起来却是非常简单，只需要一个 <code>replicas</code> 字段就搞定了，不需要再用人工监控管理，整个过程完全自动化。</p><p>下面我们再来看另一个关键字段 <code>selector</code>，它的作用是“筛选”出要被Deployment管理的Pod对象，下属字段“<strong>matchLabels</strong>”定义了Pod对象应该携带的label，它必须和“template”里Pod定义的“labels”完全相同，否则Deployment就会找不到要控制的Pod对象，apiserver也会告诉你YAML格式校验错误无法创建。</p><p>这个 <code>selector</code> 字段的用法初看起来好像是有点多余，为了保证Deployment成功创建，我们必须在YAML里把label重复写两次：一次是在“<strong>selector.matchLabels</strong>”，另一次是在“<strong>template.matadata</strong>”。像在这里，你就要在这两个地方连续写 <code>app: ngx-dep</code> ：</p><pre><code class="language-yaml">...
spec:
&nbsp; replicas: 2
&nbsp; selector:
&nbsp; &nbsp; matchLabels:
&nbsp; &nbsp; &nbsp; app: ngx-dep
&nbsp; &nbsp; &nbsp;&nbsp;
&nbsp; template:
&nbsp; &nbsp; metadata:
&nbsp; &nbsp; &nbsp; labels:
&nbsp; &nbsp; &nbsp; &nbsp; app: ngx-dep
&nbsp; &nbsp; ...
</code></pre><p>你也许会产生疑问：为什么要这么麻烦？为什么不能像Job对象一样，直接用“template”里定义好的Pod就行了呢？</p><p>这是因为在线业务和离线业务的应用场景差异很大。离线业务中的Pod基本上是一次性的，只与这个业务有关，紧紧地绑定在Job对象里，一般不会被其他对象所使用。</p><p>而在线业务就要复杂得多了，因为Pod永远在线，除了要在Deployment里部署运行，还可能会被其他的API对象引用来管理，比如负责负载均衡的Service对象。</p><p>所以Deployment和Pod实际上是一种松散的组合关系，Deployment实际上并不“持有”Pod对象，它只是帮助Pod对象能够有足够的副本数量运行，仅此而已。如果像Job那样，把Pod在模板里“写死”，那么其他的对象再想要去管理这些Pod就无能为力了。</p><p>好明白了这一点，那我们该用什么方式来描述Deployment和Pod的组合关系呢？</p><p>Kubernetes采用的是这种“贴标签”的方式，通过在API对象的“metadata”元信息里加各种标签（labels），我们就可以使用类似关系数据库里查询语句的方式，筛选出具有特定标识的那些对象。<strong>通过标签这种设计，Kubernetes就解除了Deployment和模板里Pod的强绑定，把组合关系变成了“弱引用”</strong>。</p><p>虽然话是这么说，但对于很多Kubernetes的初学者来说，理解Deployment里的spec定义还是一个难点。</p><p>所以我还是画了一张图，用不同的颜色来区分Deployment YAML里的字段，并且用虚线特别标记了 <code>matchLabels</code> 和 <code>labels</code> 之间的联系，希望能够帮助你理解Deployment与被它管理的Pod的组合关系。</p><p><img src="https://static001.geekbang.org/resource/image/1f/b0/1f1fdcd112a07cce85757e27fbcc1bb0.jpg?wh=1920x2316" alt="图片"></p><h2>如何使用kubectl操作Deployment</h2><p>把Deployment的YAML写好之后，我们就可以用 <code>kubectl apply</code> 来创建对象了：</p><pre><code class="language-plain">kubectl apply -f deploy.yml
</code></pre><p>要查看Deployment的状态，仍然是用 <code>kubectl get</code> 命令：</p><pre><code class="language-plain">kubectl get deploy
</code></pre><p><img src="https://static001.geekbang.org/resource/image/a5/72/a5b3f8a4c6ac5560dc9dfyybfb257872.png?wh=1222x184" alt="图片"></p><p>它显示的信息都很重要：</p><ul>
<li>READY表示运行的Pod数量，前面的数字是当前数量，后面的数字是期望数量，所以“2/2”的意思就是要求有两个Pod运行，现在已经启动了两个Pod。</li>
<li>UP-TO-DATE指的是当前已经更新到最新状态的Pod数量。因为如果要部署的Pod数量很多或者Pod启动比较慢，Deployment完全生效需要一个过程，UP-TO-DATE就表示现在有多少个Pod已经完成了部署，达成了模板里的“期望状态”。</li>
<li>AVAILABLE要比READY、UP-TO-DATE更进一步，不仅要求已经运行，还必须是健康状态，能够正常对外提供服务，它才是我们最关心的Deployment指标。</li>
<li>最后一个AGE就简单了，表示Deployment从创建到现在所经过的时间，也就是运行的时间。</li>
</ul><p>因为Deployment管理的是Pod，我们最终用的也是Pod，所以还需要用 <code>kubectl get pod</code> 命令来看看Pod的状态：</p><pre><code class="language-plain">kubectl get pod
</code></pre><p><img src="https://static001.geekbang.org/resource/image/4e/cb/4e47298ab0fa443e2c8936ac8ed9e5cb.png?wh=1554x244" alt="图片"></p><p>从截图里你可以看到，被Deployment管理的Pod自动带上了名字，命名的规则是Deployment的名字加上两串随机数（其实是Pod模板的Hash值）。</p><p>好，到现在对象创建成功，Deployment和Pod的状态也都没问题，可以正常服务，我们是时候检验一下Deployment部署的效果了，看看是否如前面所说的，Deployment部署的应用真的可以做到“永不宕机”？</p><p>来尝试一下吧，让我们用 <code>kubectl delete</code> 删除一个Pod，模拟一下Pod发生故障的情景：</p><pre><code class="language-plain">kubectl delete pod ngx-dep-6796688696-jm6tt
</code></pre><p>然后再查看Pod的状态：</p><pre><code class="language-plain">kubectl get pod
</code></pre><p><img src="https://static001.geekbang.org/resource/image/44/80/4467538713d83434bf6ff983acde1c80.png?wh=1562x248" alt="图片"></p><p>你就会“惊喜”地发现，被删除的Pod确实是消失了，但Kubernetes在Deployment的管理之下，很快又创建出了一个新的Pod，保证了应用实例的数量始终是我们在YAML里定义的数量。</p><p>这就证明，Deployment确实实现了它预定的目标，能够让应用“永远在线”“永不宕机”。</p><p><strong>在Deployment部署成功之后，你还可以随时调整Pod的数量，实现所谓的“应用伸缩”</strong>。这项工作在Kubernetes出现之前对于运维来说是一件很困难的事情，而现在由于有了Deployment就变得轻而易举了。</p><p><code>kubectl scale</code> 是专门用于实现“扩容”和“缩容”的命令，你只要用参数 <code>--replicas</code> 指定需要的副本数量，Kubernetes就会自动增加或者删除Pod，让最终的Pod数量达到“期望状态”。</p><p>比如下面的这条命令，就把Nginx应用扩容到了5个：</p><pre><code class="language-plain">kubectl scale --replicas=5 deploy ngx-dep
</code></pre><p><img src="https://static001.geekbang.org/resource/image/84/c4/843cc2d702b4e4034bb3a2f2f988fdc4.png?wh=1486x302" alt="图片"></p><p>但要注意， <code>kubectl scale</code> 是命令式操作，扩容和缩容只是临时的措施，如果应用需要长时间保持一个确定的Pod数量，最好还是编辑Deployment的YAML文件，改动“replicas”，再以声明式的 <code>kubectl apply</code> 修改对象的状态。</p><p>因为Deployment使用了 <code>selector</code> 字段，这里我就顺便提一下Kubernetes里 <code>labels</code> 字段的使用方法吧。</p><p>之前我们通过 <code>labels</code> 为对象“贴”了各种“标签”，在使用 <code>kubectl get</code> 命令的时候，加上参数 <code>-l</code>，使用 <code>==</code>、<code>!=</code>、<code>in</code>、<code>notin</code> 的表达式，就能够很容易地用“标签”筛选、过滤出所要查找的对象（有点类似社交媒体的 <code>#tag</code> 功能），效果和Deployment里的 <code>selector</code> 字段是一样的。</p><p>看两个例子，第一条命令找出“app”标签是 <code>nginx</code> 的所有Pod，第二条命令找出“app”标签是 <code>ngx</code>、<code>nginx</code>、<code>ngx-dep</code> 的所有Pod：</p><pre><code class="language-plain">kubectl get pod -l app=nginx
kubectl get pod -l 'app in (ngx, nginx, ngx-dep)'
</code></pre><p><img src="https://static001.geekbang.org/resource/image/b0/26/b07ba6a3a9207a5a998c237a6ef49d26.png?wh=1692x546" alt="图片"></p><h2>小结</h2><p>好了，今天我们学习了Kubernetes里的一个重要的对象：Deployment，它表示的是在线业务，和Job/CronJob的结构类似，也包装了Pod对象，通过添加额外的控制功能实现了应用永不宕机，你也可以再对比一下<a href="https://time.geekbang.org/column/article/531566">第13讲</a>来加深对它的理解。</p><p>我再简单小结一下今天的内容：</p><ol>
<li>Pod只能管理容器，不能管理自身，所以就出现了Deployment，由它来管理Pod。</li>
<li>Deployment里有三个关键字段，其中的template和Job一样，定义了要运行的Pod模板。</li>
<li>replicas字段定义了Pod的“期望数量”，Kubernetes会自动维护Pod数量到正常水平。</li>
<li>selector字段定义了基于labels筛选Pod的规则，它必须与template里Pod的labels一致。</li>
<li>创建Deployment使用命令 <code>kubectl apply</code>，应用的扩容、缩容使用命令 <code>kubectl scale</code>。</li>
</ol><p>学了Deployment这个API对象，我们今后就不应该再使用“裸Pod”了。即使我们只运行一个Pod，也要以Deployment的方式来创建它，虽然它的 <code>replicas</code> 字段值是1，但Deployment会保证应用永远在线。</p><p>另外，作为Kubernetes里最常用的对象，Deployment的本事还不止这些，它还支持滚动更新、版本回退，自动伸缩等高级功能，这些在“高级篇”里我们再详细学习。</p><h2>课下作业</h2><p>最后是课下作业时间，给你留两个思考题：</p><ol>
<li>如果把Deployment里的 <code>replicas</code> 字段设置成0会有什么效果？有什么意义呢？</li>
<li>你觉得Deployment能够应用在哪些场景里？有没有什么缺点或者不足呢？</li>
</ol><p>欢迎在留言区分享你的想法。</p><p>这一章我们学习的Kubernetes高级对象，对云计算、集群管理非常重要。多多思考，打好基础，我们继续深入。下节课再见。</p><p><img src="https://static001.geekbang.org/resource/image/22/7f/22a054fac2709bbcaabe209aa6fff47f.jpg?wh=1920x2635" alt=""></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/1b/f8/01e7fc0e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>郑小鹿</span>
  </div>
  <div class="_2_QraFYR_0">问题回答<br>1、如果把 Deployment 里的 replicas 字段设置成 0 会有什么效果？有什么意义呢？<br>做了下实验，效果如下：<br>$ kubectl get po -n nginx-deploy                   <br>No resources found in default namespace.<br>$ kubectl get deploy                  <br>NAME               READY   UP-TO-DATE   AVAILABLE   AGE<br>nginx-deployment   0&#47;0     0            0  <br><br>意义：关闭服务的同时，又可以保留服务的配置，下次想要重新部署的时候只需要修改deployment就可以快速上线。<br><br>2、你觉得 Deployment 能够应用在哪些场景里？有没有什么缺点或者不足呢？<br>使用场景：用在部署无状态服务，部署升级，对服务的扩缩容；多个Deployment 可以实现金丝雀发布<br><br>不足：Deployment把所有pod都认为是一样的服务，前后没有顺序，没有依赖关系，同时认为所有部署节点也是一样的，不会做特殊处理等<br><br>疑问：Deployment变更副本数时，是先删除pod，然后再重建pod，如果服务启停时间比较长，会出现什么问题不？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 回答的很好。<br><br>这个其实是后面要讲的滚动更新，Deployment会保证任何时候都会有足够数量的pod处于可用状态，保证应用正常对外提供服务。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-01 10:44:24</div>
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
  <div class="_2_QraFYR_0">懂后恍然大悟，不懂时举步维艰，学习的快乐大抵如此</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: nice</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-02 11:29:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/cb/3d/b290414d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>岁月长</span>
  </div>
  <div class="_2_QraFYR_0">回答问题1:<br>之前在公司的时候，有时候会把服务下线，这个时候就会把 replicas 字段改为 0，观察一段时间没问题后在把配置删除，如果有报错也方便马上恢复</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-15 14:09:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKx6EdicYYuYK745brMa9yAlkZs2YmzxRAm4BQ2kw9GbtcC8ebnQlyBfIJnGjH57ib4HVlQIpSbTrBw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>dst</span>
  </div>
  <div class="_2_QraFYR_0">回答一下问题2，deploy是只能用在应用是无状态的场景下，对于有状态的应用它就无能为力了，需要使用其他的api</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good </p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-01 08:44:32</div>
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
  <div class="_2_QraFYR_0">老师，我想线创建一个pod，然后直接使用ngx-aadep来管理老的pod，这样的方式不行吗，你课程里说，pod不属于deployment。那我就单独创建，但是显示我语法错误。<br><br>cat ngx-aadep.yaml <br>kind: Deployment<br>metadata:<br>  creationTimestamp: null<br>  labels:<br>    app: ngx-aa<br>  name: ngx-aa<br>spec:<br>  replicas: 1<br>  selector:<br>    matchLabels:<br>      app: ngx-aa<br>cat ngx-aapod.yaml<br>apiVersion: v1<br>kind: Pod<br>metadata:<br>  creationTimestamp: null<br>  labels:<br>    run: ngx-aa<br>    app: ngx-aa<br>  name: ngx-aa<br>spec:<br>  containers:<br>  - image: nginx:alpine<br>    name: ngx-aa<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Pod是由Deployment模板创建出来的，它受Deployment管控，单独创建Pod无法纳入Deployment的管理，因为Kubernetes就是这么规定的运行机制。<br><br>“pod不属于deployment”这个说法可能带来了一些误解，这个实际上是相对于Job来说的，不是强绑定关系。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-02 22:33:12</div>
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
  <div class="_2_QraFYR_0">我有个疑惑，如果像部署redis, etcd等集群模式，比如3个pod, 对应的集群里应该会有个master，像这种有状态的服务，如果采用deployment模式部署会有影响吗，还是单独部署3个pod, 望大家指点</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不能用Deployment，应该用StatefulSet，高级篇会讲。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-18 21:36:24</div>
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
  <div class="_2_QraFYR_0">1. 设置为0，就是pod没了，deployment还有，看同学们回答是保留配置，这个不错。<br>2. 管理无状态服务，什么叫有状态，什么叫无状态，我不太理解。<br><br>另外，我刚刚突发奇想，deployment只留一个头加上select配置，然后里面的pod对象单独取出来，建立一个文件，pod可以建立，但是deloyment无法建立，但是提示我是语法错误，其实我不太理解。既然这两服务是独立了，我为啥不能这么做呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 后面讲StatefulSet的时候就可以理解了。<br><br>Pod YAML 只能创建一个对象，而Deployment里的pod定义是一个“模板”，可以创建出任意多个对象。而且Kubernetes就是规定要这么使用Deployment，不这样做当然就报错了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-02 22:27:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2f/85/6f/1654f4b9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>nc_ops</span>
  </div>
  <div class="_2_QraFYR_0">对这句话有个疑问，“kubectl scale 是命令式操作，扩容和缩容只是临时的措施，如果应用需要长时间保持一个确定的 Pod 数量，最好还是编辑 Deployment 的 YAML 文件”<br>我刚实验通过kubectl scale去扩容pod数量，然后通过kubectl delete去删除一个pod，立马又会新生成一个pod，所以通过kubectl scale也是能保持一个确定的pod数量的吧？通过yaml文件去改变副本的好处准确来说应该是让整个生产环境里只有一份配置的描述，避免当kubectl scale执行后，实际deployment规格与yaml文件里不一致，避免让运维引发混淆</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，就是这个意思。kubectl scale使用后会让Deployment的状态和YAML 不一致，所以说是临时的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-28 18:06:26</div>
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
  <div class="_2_QraFYR_0">请教老师两个问题：<br>Q1：有状态的应用怎么发布？<br>既然Deplayment是用来发布无状态的应用，那有状态的应用怎么发布？<br>k83不能发布有状态的应用吗？<br><br>Q2：怎么访问用Deployment创建的Nginx？<br>我用Deployment成功创建了两个nginx，一个IP是172.17.0.11,请问怎么访问该Nginx？（最好能给出具体的操作方法）。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.用StatefulSet，高级篇里讲。<br><br>2.用Service对象，后面会讲到。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-01 23:05:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/1b/f8/01e7fc0e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>郑小鹿</span>
  </div>
  <div class="_2_QraFYR_0">「 下属字段“matchLabels”定义了 Pod 对象应该携带的 label，它必须和“template”里 Pod 定义的“labels”完全相同 」<br><br>老师这个应该是指某个标签的内容完全一样吧。selector.matchLabels”是“template.matadata”中“labels”的子集。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是完全一样，不是子集，可以自己改一下试试。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-01 14:13:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/e9/26/afc08398.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Amosヾ</span>
  </div>
  <div class="_2_QraFYR_0">第二个问题，如果是有状态应用不同Pod间互相访问，而deployment生成的pod最后带有不固定的哈希字符串，无法唯一确定某个pod，此时deploy不适用了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，需要用StatefulSet来管理。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-01 09:07:37</div>
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
  <div class="_2_QraFYR_0">第一个问题：将replicas设置为0，对应应用的pod数量为0，应用停止服务。这样的方式保存了deployment，如果需要启动应用，将replicas设置为需要的数量即可<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great </p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-01 00:32:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2a/3b/29/0f86235e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>明月夜</span>
  </div>
  <div class="_2_QraFYR_0">老师好，我有一些疑问：<br>1. 既然Deployment 只是帮助 Pod 对象能够有足够的副本数量运行，我尝试着在这个deployment之外，单独起一个pod，设置一样label，明面上pod数量增加了，不符合deployment里replicas的预期，多出来的pod应该被销毁才对，但实际并不是这样，这个独立的pod并没有影响到这个deployment，为什么？<br>2. 我看了service那一章，service里也是在selector字段下指明要代理的pod的标签，我也做了同样的试验，在deployment之外，单独起一个pod，设置一样的标签，service除了能代理deployment的pod外，这个独立的pod也能被代理，为什么会有这种不一致性？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. deployment使用的是它自己的模板创建的Pod，独立启动的Pod与它没有从属关系，所以无法管理。<br><br>2. service不管理pod，只是使用label来查找pod，所以是可以找到所有一样label的Pod。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-30 23:31:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_e46db2</span>
  </div>
  <div class="_2_QraFYR_0">一个node默认最多110个pod，如果机器资源有限情况下，比如创建了30个就占满了，k8s会自动限制node保证可用吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 当然了，kubernetes会监控每个node上的资源，node资源不足就不会调度。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-15 09:58:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/ce/05/4c493ef9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>李泽文</span>
  </div>
  <div class="_2_QraFYR_0">老师，我还是不太理解Job和Deployment的区别，什么是在线业务，什么是离线业务？通过对Job和Deployment的对别，感觉都差不多。Job里可以通过配置实现pod的总数量，并发数量，这个跟Deployment的replicas有什么区别？在Job里我们可以配置pod运行失败的重启策略，这个跟Deployment的动态扩缩容又有什么区别？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Job运行的任务有时限，必定会终止，Deployment是长期运行的服务，不会自动停止。<br><br>Job一般是一些离线计算任务，比如MapReduce，而Deployment就是Nginx、Redis这样的服务应用。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-05 15:58:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/23/b9/6f/b40d1acf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>mkcaptain</span>
  </div>
  <div class="_2_QraFYR_0">老师，问个问题<br>既然deployment和pod定义是松散的连接关系，也有用label指向了具体pod<br>那deployment的template部分能去掉么。也就是说deployment能只用label去管理已经运行的pod么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: template肯定不能去掉，不然就无法创建pod了，这个是定义决定的。<br><br>至于第二个问题，可以自己试一下，不过可能不会有人这么用。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-11 10:11:10</div>
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
  <div class="_2_QraFYR_0">deployment 提供的多实例，在对外提供服务的时候是只有一个应用，还是多个应用同时提供服务呢？ 它是支持负债均衡吗？ 如果是，那与service提供的负债均衡有什么区别？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Deployment只是负责管理多个应用实例，提供服务是Service和Ingress的事情。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-21 13:21:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/45/c3/775fe460.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>rubys_</span>
  </div>
  <div class="_2_QraFYR_0">老师，我有一个疑问，就是能不能把 pod 理解为一个进程，如果我的应用在之前没有使用 k8s 部署的时候是启动了 100 个进程，现在换做 k8s 来部署的话，是不是要设置 replicas 为 100 才有 100 个进程来对外提供服务</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: pod里可能会有多个容器，也就是进程，理解成进程组是比较合适的，而且有的应用还是多进程的，比如Nginx，所以把pod看做是一个应用实例更好。<br><br>不过一般pod里只有一个进程，所以你这么理解也是可以的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-19 17:27:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/b9/4b/2376a469.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>DN</span>
  </div>
  <div class="_2_QraFYR_0">想请教一下老师，在master通过 deployment部署的应用， kubectl exec -it nginx-deployment-85658cc69f-l68tx -- &#47;bin&#47;bash 报错connect: connection refused， 但其他的命令 apply describe等都正常，想问一下可能是什么原因，感谢感谢。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Nginx镜像里没有bash，一般我们都用标准的sh。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-07 10:20:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/dotGbXAlAZg0bhCq4P96A40mdyavzR33jSqIHk8xLlic4B5PYNDIP5MEa1Fk9yxzdz9scHUM7IUNR71nVZNoV7Q/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>yhtyht308</span>
  </div>
  <div class="_2_QraFYR_0">使用 kubectl get 命令通过参数l查找标签名，是对应pod的labels，不是name。可以通过“kubectl get pod --show-labels”来查找。<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: kubectl的参数很多，课程正文里不可能全讲到，欢迎大家补充有用、常用的命令。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-02 17:30:17</div>
  </div>
</div>
</div>
</li>
</ul>