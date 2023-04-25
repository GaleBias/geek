<audio title="19｜Daemonset：忠实可靠的看门狗" src="https://static001.geekbang.org/resource/audio/2e/9a/2e9d4b72d991d455be0109d0bcd9779a.mp3" controls="controls"></audio> 
<p>你好，我是Chrono。</p><p>上一次课里我们学习了Kubernetes里的一个新API对象Deployment，它代表了在线业务，能够管理多个Pod副本，让应用永远在线，还能够任意扩容缩容。</p><p>虽然Deployment非常有用，但是，它并没有完全解决运维部署应用程序的所有难题。因为和简单的离线业务比起来，在线业务的应用场景太多太复杂，Deployment的功能特性只覆盖了其中的一部分，无法满足其他场景的需求。</p><p>今天我们就来看看另一类代表在线业务API对象：<strong>DaemonSet</strong>，它会在Kubernetes集群的每个节点上都运行一个Pod，就好像是Linux系统里的“守护进程”（Daemon）。</p><h2>为什么要有DaemonSet</h2><p>想知道为什么Kubernetes会引入DaemonSet对象，那就得知道Deployment有哪些不足。</p><p>我们先简单复习一下Deployment，它能够创建任意多个的Pod实例，并且维护这些Pod的正常运行，保证应用始终处于可用状态。</p><p>但是，Deployment并不关心这些Pod会在集群的哪些节点上运行，<strong>在它看来，Pod的运行环境与功能是无关的，只要Pod的数量足够，应用程序应该会正常工作</strong>。</p><!-- [[[read_end]]] --><p>这个假设对于大多数业务来说是没问题的，比如Nginx、WordPress、MySQL，它们不需要知道集群、节点的细节信息，只要配置好环境变量和存储卷，在哪里“跑”都是一样的。</p><p>但是有一些业务比较特殊，它们不是完全独立于系统运行的，而是与主机存在“绑定”关系，必须要依附于节点才能产生价值，比如说：</p><ul>
<li>网络应用（如kube-proxy），必须每个节点都运行一个Pod，否则节点就无法加入Kubernetes网络。</li>
<li>监控应用（如Prometheus），必须每个节点都有一个Pod用来监控节点的状态，实时上报信息。</li>
<li>日志应用（如Fluentd），必须在每个节点上运行一个Pod，才能够搜集容器运行时产生的日志数据。</li>
<li>安全应用，同样的，每个节点都要有一个Pod来执行安全审计、入侵检查、漏洞扫描等工作。</li>
</ul><p>这些业务如果用Deployment来部署就不太合适了，因为Deployment所管理的Pod数量是固定的，而且可能会在集群里“漂移”，但，实际的需求却是要在集群里的每个节点上都运行Pod，也就是说Pod的数量与节点数量保持同步。</p><p>所以，Kubernetes就定义了新的API对象DaemonSet，它在形式上和Deployment类似，都是管理控制Pod，但管理调度策略却不同。DaemonSet的目标是在集群的每个节点上运行且仅运行一个Pod，就好像是为节点配上一只“看门狗”，忠实地“守护”着节点，这就是DaemonSet名字的由来。</p><h2>如何使用YAML描述DaemonSet</h2><p>DaemonSet和Deployment都属于在线业务，所以它们也都是“apps”组，使用命令  <code>kubectl api-resources</code>  可以知道它的简称是 <code>ds</code> ，YAML文件头信息应该是：</p><pre><code class="language-yaml">apiVersion: apps/v1
kind: DaemonSet
metadata:
&nbsp; name: xxx-ds
</code></pre><p>不过非常奇怪，kubectl 不提供自动创建DaemonSet YAML样板的功能，也就是说，我们不能用命令 <code>kubectl create</code> 直接创建出一个DaemonSet对象。</p><p><img src="https://static001.geekbang.org/resource/image/99/7f/99b434fc4089ce23a7e54ed8b857a27f.png?wh=1190x304" alt="图片"></p><p>这个缺点对于我们使用DaemonSet的确造成了不小的麻烦，毕竟如果用 <code>kubectl explain</code> 一个个地去查字段再去写YAML实在是太辛苦了。</p><p>不过，Kubernetes不给我们生成样板文件的机会，我们也可以自己去“抄”。你可以在Kubernetes的官网（<a href="https://kubernetes.io/zh/docs/concepts/workloads/controllers/daemonset/">https://kubernetes.io/zh/docs/concepts/workloads/controllers/daemonset/</a>）上找到一份DaemonSet的YAML示例，把它拷贝下来，再去掉多余的部分，就可以做成自己的一份样板文件，大概是下面的这个样子：</p><pre><code class="language-yaml">apiVersion: apps/v1
kind: DaemonSet
metadata:
&nbsp; name: redis-ds
&nbsp; labels:
&nbsp; &nbsp; app: redis-ds

spec:
&nbsp; selector:
&nbsp; &nbsp; matchLabels:
&nbsp; &nbsp; &nbsp; name: redis-ds

&nbsp; template:
&nbsp; &nbsp; metadata:
&nbsp; &nbsp; &nbsp; labels:
&nbsp; &nbsp; &nbsp; &nbsp; name: redis-ds
&nbsp; &nbsp; spec:
&nbsp; &nbsp; &nbsp; containers:
&nbsp; &nbsp; &nbsp; - image: redis:5-alpine
&nbsp; &nbsp; &nbsp; &nbsp; name: redis
&nbsp; &nbsp; &nbsp; &nbsp; ports:
&nbsp; &nbsp; &nbsp; &nbsp; - containerPort: 6379
</code></pre><p>这个DaemonSet对象的名字是 <code>redis-ds</code>，镜像是 <code>redis:5-alpine</code>，使用了流行的NoSQL数据库Redis（你也许对它很熟悉）。</p><p>把这份YAML和上节课里的Deployment对象简单对比一下，你会发现：</p><p>前面的 <code>kind</code>、<code>metadata</code> 是对象独有的信息，自然是不同的，但下面的 <code>spec</code> 部分，DaemonSet也有 <code>selector</code> 字段，匹配 <code>template</code> 里Pod的 <code>labels</code> 标签，和Deployment对象几乎一模一样。</p><p>再仔细观察，我们就会看到，DaemonSet在 <code>spec</code> 里没有 <code>replicas</code> 字段，这是它与Deployment的一个关键不同点，意味着它不会在集群里创建多个Pod副本，而是要在每个节点上只创建出一个Pod实例。</p><p>也就是说，DaemonSet仅仅是在Pod的部署调度策略上和Deployment不同，其他的都是相同的，某种程度上我们也可以把DaemonSet看做是Deployment的一个特例。</p><p>我还是把YAML描述文件画了一张图，好让你看清楚与Deployment的差异：</p><p><img src="https://static001.geekbang.org/resource/image/c1/1c/c1dee411aa02f4ff2b8caaf0bd627a1c.jpg?wh=1920x1173" alt="图片"></p><p>了解到这些区别，现在，我们就可以用变通的方法来创建DaemonSet的YAML样板了，你只需要用 <code>kubectl create</code> 先创建出一个Deployment对象，然后把 <code>kind</code> 改成 <code>DaemonSet</code>，再删除 <code>spec.replicas</code> 就行了，比如：</p><pre><code class="language-bash">export out="--dry-run=client -o yaml"

# change "kind" to DaemonSet
kubectl create deploy redis-ds --image=redis:5-alpine $out
</code></pre><h2>如何在Kubernetes里使用DaemonSet</h2><p>现在，让我们执行命令 <code>kubectl apply</code>，把YAML发送给Kubernetes，让它创建DaemonSet对象，再用 <code>kubectl get</code> 查看对象的状态：</p><p><img src="https://static001.geekbang.org/resource/image/43/f3/4349f1f2aed7f4ffac017ee6064059f3.png?wh=1884x486" alt="图片"></p><p>看这张截图，虽然我们没有指定DaemonSet里Pod要运行的数量，但它自己就会去查找集群里的节点，在节点里创建Pod。因为我们的实验环境里有一个Master一个Worker，而Master默认是不跑应用的，所以DaemonSet就只生成了一个Pod，运行在了“worker”节点上。</p><p>暂停一下，你发现这里有什么不对劲了吗？</p><p>按照DaemonSet的本意，应该在每个节点上都运行一个Pod实例才对，但Master节点却被排除在外了，这就不符合我们当初的设想了。</p><p>显然，DaemonSet没有尽到“看门”的职责，它的设计与Kubernetes集群的工作机制发生了冲突，有没有办法解决呢？</p><p>当然，Kubernetes早就想到了这点，为了应对Pod在某些节点的“调度”和“驱逐”问题，它定义了两个新的概念：<strong>污点</strong>（taint）和<strong>容忍度</strong>（toleration）。</p><h2>什么是污点（taint）和容忍度（toleration）</h2><p>“污点”是Kubernetes节点的一个属性，它的作用也是给节点“贴标签”，但为了不和已有的 <code>labels</code> 字段混淆，就改成了 <code>taint</code>。</p><p>和“污点”相对的，就是Pod的“容忍度”，顾名思义，就是Pod能否“容忍”污点。</p><p>我们把它俩放在一起就比较好理解了。集群里的节点各式各样，有的节点“纯洁无瑕”，没有“污点”；而有的节点因为某种原因粘上了“泥巴”，也就有了“污点”。Pod也脾气各异，有的“洁癖”很严重，不能容忍“污点”，只能挑选“干净”的节点；而有的Pod则比较“大大咧咧”，要求不那么高，可以适当地容忍一些小“污点”。</p><p>这么看来，“污点”和“容忍度”倒是有点像是一个“相亲”的过程。Pod就是一个挑剔的“甲方”，而“乙方”就是集群里的各个节点，Pod会根据自己对“污点”的“容忍程度”来选择合适的目标，比如要求“不抽烟不喝酒”，但可以“无车无房”，最终决定在哪个节点上“落户”。</p><p>Kubernetes在创建集群的时候会自动给节点Node加上一些“污点”，方便Pod的调度和部署。<strong>你可以用 <code>kubectl describe node</code> 来查看Master和Worker的状态</strong>：</p><pre><code class="language-bash">kubectl describe node master

Name:&nbsp; &nbsp; &nbsp;master
Roles:&nbsp; &nbsp; control-plane,master
...
Taints:&nbsp; &nbsp;node-role.kubernetes.io/master:NoSchedule
...

kubectl describe node worker

Name:&nbsp; &nbsp; &nbsp;worker
Roles:&nbsp; &nbsp; &lt;none&gt;
...
Taints:&nbsp; &nbsp;&lt;none&gt;
...
</code></pre><p>可以看到，Master节点默认有一个 <code>taint</code>，名字是 <code>node-role.kubernetes.io/master</code>，它的效果是 <code>NoSchedule</code>，也就是说这个污点会拒绝Pod调度到本节点上运行，而Worker节点的 <code>taint</code> 字段则是空的。</p><p>这正是Master和Worker在Pod调度策略上的区别所在，通常来说Pod都不能容忍任何“污点”，所以加上了 <code>taint</code> 属性的Master节点也就会无缘Pod了。</p><p>明白了“污点”和“容忍度”的概念，你就知道该怎么让DaemonSet在Master节点（或者任意其他节点）上运行了，方法有两种。</p><p><strong>第一种方法</strong>是去掉Master节点上的 <code>taint</code>，让Master变得和Worker一样“纯洁无瑕”，DaemonSet自然就不需要再区分Master/Worker。</p><p>操作Node上的“污点”属性需要使用命令 <code>kubectl taint</code>，然后指定节点名、污点名和污点的效果，去掉污点要额外加上一个 <code>-</code>。</p><p>比如要去掉Master节点的“NoSchedule”效果，就要用这条命令：</p><pre><code class="language-plain">kubectl taint node master node-role.kubernetes.io/master:NoSchedule-
</code></pre><p><img src="https://static001.geekbang.org/resource/image/e8/0e/e8e877c960e43a407ab0d95963de400e.png?wh=1920x103" alt="图片"></p><p>因为DaemonSet一直在监控集群节点的状态，命令执行后Master节点已经没有了“污点”，所以它立刻就会发现变化，然后就会在Master节点上创建一个“守护”Pod。你可以用 <code>kubectl get</code> 来查看这个变动情况：</p><p><img src="https://static001.geekbang.org/resource/image/44/37/4440c4f05dd7718c52152ef20fc77237.png?wh=1882x422" alt="图片"></p><p>但是，这种方法修改的是Node的状态，影响面会比较大，可能会导致很多Pod都跑到这个节点上运行，所以我们可以保留Node的“污点”，为需要的Pod添加“容忍度”，只让某些Pod运行在个别节点上，实现“精细化”调度。</p><p>这就是<strong>第二种方法</strong>，为Pod添加字段 <code>tolerations</code>，让它能够“容忍”某些“污点”，就可以在任意的节点上运行了。</p><p><code>tolerations</code> 是一个数组，里面可以列出多个被“容忍”的“污点”，需要写清楚“污点”的名字、效果。比较特别是要用 <code>operator</code> 字段指定如何匹配“污点”，一般我们都使用 <code>Exists</code>，也就是说存在这个名字和效果的“污点”。</p><p>如果我们想让DaemonSet里的Pod能够在Master节点上运行，就要写出这样的一个 <code>tolerations</code>，容忍节点的 <code>node-role.kubernetes.io/master:NoSchedule</code> 这个污点：</p><pre><code class="language-plain">tolerations:
- key: node-role.kubernetes.io/master
&nbsp; effect: NoSchedule
&nbsp; operator: Exists
</code></pre><p>现在我们先用 <code>kubectl taint</code> 命令把Master的“污点”加上：</p><pre><code class="language-plain">kubectl taint node master node-role.kubernetes.io/master:NoSchedule
</code></pre><p><img src="https://static001.geekbang.org/resource/image/3e/c4/3eb49484fd460e53a40fb239298077c4.png?wh=1920x103" alt="图片"></p><p>然后我们再重新部署加上了“容忍度”的DaemonSet：</p><pre><code class="language-plain">kubectl apply -f ds.yml
</code></pre><p><img src="https://static001.geekbang.org/resource/image/20/e8/2060a08c2b5572b71780c5f5dyyedae8.png?wh=1888x542" alt="图片"></p><p>你就会看到DaemonSet仍然有两个Pod，分别运行在Master和Worker节点上，与第一种方法的效果相同。</p><p>需要特别说明一下，“容忍度”并不是DaemonSet独有的概念，而是从属于Pod，所以理解了“污点”和“容忍度”之后，你可以在Job/CronJob、Deployment里为它们管理的Pod也加上 <code>tolerations</code>，从而能够更灵活地调度应用。</p><p>至于都有哪些污点、污点有哪些效果我就不细说了，Kubernetes官网文档（<a href="https://kubernetes.io/zh/docs/concepts/scheduling-eviction/taint-and-toleration/">https://kubernetes.io/zh/docs/concepts/scheduling-eviction/taint-and-toleration/</a>）上都列的非常清楚，在理解了工作原理之后，相信你自己学起来也不会太难。</p><h2>什么是静态Pod</h2><p>DaemonSet是在Kubernetes里运行节点专属Pod最常用的方式，但它不是唯一的方式，Kubernetes还支持另外一种叫“<strong>静态Pod</strong>”的应用部署手段。</p><p>“静态Pod”非常特殊，它不受Kubernetes系统的管控，不与apiserver、scheduler发生关系，所以是“静态”的。</p><p>但既然它是Pod，也必然会“跑”在容器运行时上，也会有YAML文件来描述它，而唯一能够管理它的Kubernetes组件也就只有在每个节点上运行的kubelet了。</p><p>“静态Pod”的YAML文件默认都存放在节点的 <code>/etc/kubernetes/manifests</code> 目录下，它是Kubernetes的专用目录。</p><p>下面的这张截图就是Master节点里目录的情况：</p><p><img src="https://static001.geekbang.org/resource/image/f5/c2/f5477bf666beffcaf3b8663d5a5692c2.png?wh=1842x486" alt="图片"></p><p>你可以看到，Kubernetes的4个核心组件apiserver、etcd、scheduler、controller-manager原来都以静态Pod的形式存在的，这也是为什么它们能够先于Kubernetes集群启动的原因。</p><p>如果你有一些DaemonSet无法满足的特殊的需求，可以考虑使用静态Pod，编写一个YAML文件放到这个目录里，节点的kubelet会定期检查目录里的文件，发现变化就会调用容器运行时创建或者删除静态Pod。</p><h2>小结</h2><p>好了，今天我们学习了Kubernetes里部署应用程序的另一种方式：DaemonSet，它与Deployment很类似，差别只在于Pod的调度策略，适用于在系统里运行节点的“守护进程”。</p><p>简单小结一下今天的内容：</p><ol>
<li>DaemonSet的目标是为集群里的每个节点部署唯一的Pod，常用于监控、日志等业务。</li>
<li>DaemonSet的YAML描述与Deployment非常接近，只是没有 <code>replicas</code> 字段。</li>
<li>“污点”和“容忍度”是与DaemonSet相关的两个重要概念，分别从属于Node和Pod，共同决定了Pod的调度策略。</li>
<li>静态Pod也可以实现和DaemonSet同样的效果，但它不受Kubernetes控制，必须在节点上纯手动部署，应当慎用。</li>
</ol><h2>课下作业</h2><p>最后是课下作业时间，给你留两个思考题：</p><ol>
<li>你觉得DaemonSet和Deployment在用法上还有哪些不同？它们分别适用于哪些场景？</li>
<li>你觉得在Kubernetes里应该如何用好“污点”和“容忍度”这两个概念？</li>
</ol><p>欢迎留言分享你的想法，和其他同学一起参与讨论。我们下节课再见。</p><p><img src="https://static001.geekbang.org/resource/image/64/2e/64760f80fbbda9dd72c14a37826c9d2e.jpg?wh=1920x2209" alt=""></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/95/4e/1112248a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>鱼</span>
  </div>
  <div class="_2_QraFYR_0">老师，我想问问如果给一个node加上污点，那么在上面运行的不能容忍该污点的pod会自动跑到其他node上吗。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 已经调度过，运行在节点上的pod就不会再关心taint了，因为taint只在调度的时候生效，也可以自己试验一下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-04 15:13:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/32/a8/d5bf5445.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>郑海成</span>
  </div>
  <div class="_2_QraFYR_0">Q1：1.原理上ds直接own所选择的pod，deploy则是own创建的rs，rs own pod；2.功能上deploy支持在线业务部署功能更多，比如滚动更新和回滚，快速扩缩副本，ds则副本数基本固定；3.使用上ds多用在提供平台侧的能力，deploy则多用在提供业务侧能力，当然平台侧也用得很多<br><br>Q2: taint和tolerence是和调度相关的概念，调度器调度pod是会考虑</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-08 19:25:42</div>
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
  <div class="_2_QraFYR_0">测试结论：当master节点去除污点时，pod会调度到master；master节点增加污点时，pod不会离开，手动删除后不会再增加</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-09 22:04:05</div>
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
  <div class="_2_QraFYR_0">controller-plane是不是为了向openshift看齐</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这其中的关系就不了解了，不过现在IT界的趋势是避讳“master”“slave”，毕竟zzzq。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-03 11:09:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/94/fe/5fbf1bdc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Layne</span>
  </div>
  <div class="_2_QraFYR_0">已实操<br>1.重新添加master污点信息后，master节点上的pod应用不会自动删除；<br>2.污点容忍配置路径位置：kubectl explain ds.spec.template.spec.tolerations</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-06 09:27:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/99/87/98ebb20e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>dao</span>
  </div>
  <div class="_2_QraFYR_0">老师，kubectl get ds -n kube-system 找不到 Flannel ds ，查看了 kube-flannel.yml ，发现名空间已经改到 kube-flannel 。所以要使用 kubectl get ds -n kube-flannel 查看网络插件 Flannel 。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可能是新版的flannel改了，注意下版本号吧，或者手动改一下flannel的YAML 。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-13 11:31:23</div>
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
  <div class="_2_QraFYR_0">请问老师，调度是在什么时候发生的呢，我想到一个问题，一个pod的node加上污点，正常运转的情况你在回答里说，不会跑到其他机器上，那假如pod死亡，会重新根据污点情况进行调度吗，还是按照原先的策略进行调度呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.apply YAML，把对象提交到apiserver调度就会开始。<br><br>2.会重新调度，做试验验证一下就知道了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-17 11:07:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/88/cd/2c3808ce.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Yangjing</span>
  </div>
  <div class="_2_QraFYR_0">类似 Mysql 的适合部署到“静态 Pod”上吗？ </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感觉不适合，它不应该是守护进程类型的应用吧。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-03 23:34:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/9c/31/e4677275.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>潜光隐耀</span>
  </div>
  <div class="_2_QraFYR_0">请问下，DaemonSet的Pod如果被删除，是否能被自动重建，保持高可用呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 当然会自动拉起，和Deployment是一样的，可以试一下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-03 12:16:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2e/09/d0/8609bddc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>戒贪嗔痴</span>
  </div>
  <div class="_2_QraFYR_0">老师，是这样的，由于本人使用的k8s版本是v1.24.0，使用kubectl describe node master得到的结果是<br>Taints:             node-role.kubernetes.io&#47;control-plane:NoSchedule<br>                    node-role.kubernetes.io&#47;master:NoSchedule<br>因为刚开始不知道你最后那张图里说的新版本已经改为control-plane的这种写法了，导致按照教程操作下来2种方法都不能实现把pod调度到master上运行，所以请教下老师是不是taint和toleration都要改为control-plane的写法？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，因为1.24已经不再使用“master”了，要改成“control-plane”，Pod必须能够容忍control-plane上的所有污点才行。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-03 09:48:57</div>
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
  <div class="_2_QraFYR_0">&#47;etc&#47;kubernetes&#47;manifests 是不是意味着这个目录下静态pod的yaml文件损坏或者目录被删除，就代表集群会蹦，假如有两台master，一台master的yaml文件被删除，另外一台正常的master会不会进行yaml文件的同步？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这里存的静态Pod定义，只是本机上的Pod会受影响。<br><br>第二个问题不是太了解，应该不会吧。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-24 14:19:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/50/33/9dcd30c4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>斯蒂芬.赵</span>
  </div>
  <div class="_2_QraFYR_0">想问一下老师DaemonSet 和 Deployment 它们分别适用于哪些场景？<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 重点看它们的部署方式，DaemonSet用途是守护进程，Deployment是可伸缩的应用。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-23 08:38:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_02ce66</span>
  </div>
  <div class="_2_QraFYR_0">老师请教一下使用DaemonSet启动的应用如何关闭呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 直接把DaemonSet对象删掉就行了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-27 08:31:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2c/c7/89/16437396.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>极客酱酱</span>
  </div>
  <div class="_2_QraFYR_0">如果未被忽略的污点中存在至少一个 effect 值为 NoExecute 的污点， 则 Kubernetes 不会将 Pod 调度到该节点（如果 Pod 还未在节点上运行）， 或者将 Pod 从该节点驱逐（如果 Pod 已经在节点上运行）。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-03 17:47:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jntv</span>
  </div>
  <div class="_2_QraFYR_0">1.重新添加master污点信息后，master节点上的pod应用不会自动删除；<br>2.污点容忍配置路径位置：kubectl explain ds.spec.template.spec.tolerations</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-29 11:00:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/8c/bf/182ee8e6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>周Sir</span>
  </div>
  <div class="_2_QraFYR_0">Daemonset固定使用redis镜像吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 任何应用都可以，这里只是用Redis作为示例。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-17 14:07:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/05/93/3c3f2a6d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>安石</span>
  </div>
  <div class="_2_QraFYR_0">容忍度为什么是在pod的字段而不是ds上的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 因为Kubernetes调度的单位是pod，DaemonSet只是管理pod。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-13 20:58:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/2c/68/c299bc71.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>天敌</span>
  </div>
  <div class="_2_QraFYR_0">记录一个问题:<br>master节点的ds报如下错:<br>ailed to create pod sandbox: rpc error: code = Unknown desc = failed to set up sandbox container &quot;85f6cc55c5bf809f10a6af0b888944031a14c159b9fc98b297f19874c22c5646&quot; network for pod &quot;redis-ds-2zhk6&quot;: networkPlugin cni failed to set up pod &quot;redis-ds-2zhk6_default&quot; network: open &#47;run&#47;flannel&#47;subnet.env: no such file or directory<br><br>只需将worker节点的 &#47;run&#47;flannel&#47;subnet.env 拷贝到 master 节点上的 &#47;run&#47;flannel&#47;subnet.env 再删除当前pod即可。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-18 22:22:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/f0/25/d3da7ca9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>wuyang</span>
  </div>
  <div class="_2_QraFYR_0">删除节点taint时，老是报错，有没有大神看下什么问题<br>wuyang@master-1:~$ kubectl describe node master | grep -i Taints<br>Taints:             node-role.kubernetes.io&#47;master:NoSchedule<br>wuyang@master-1:~$ kubectl taint node master node-role.kubernetes.io&#47;master:NoSchedule-node&#47;master untainted<br>error: all resources must be specified before taint changes: untainted<br><br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 注意命令最后要有个减号，表示去掉taint，就是node-role.kubernetes.io&#47;master:NoSchedule-</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-13 23:59:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/9a/e1/7df2ad19.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>fd</span>
  </div>
  <div class="_2_QraFYR_0">所以，Kubernetes 就定义了新的 API 对象 DaemonSet，它在形式上和 Deployment 类似，都是管理控制 Pod，但管理调度策略却不同。DaemonSet 的目标是在集群的每个节点上运行且仅运行一个 Pod，就好像是为节点配上一只“看门狗”，忠实地“守护”着节点，这就是 DaemonSet 名字的由来。<br><br>===============<br>运行且仅运行一个 Pod，那我的业务Pod 就不能运行在这个节点上了么？还是 意思是 DaemonSet 其实是保证每一个节点都会有这个基础的Pod，比如日志收集，监控这类的POD，每一类的基础Pod 由 DaemonSet 保证有而且有一个？<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 看来是我没有表述清楚，抱歉。<br><br>意思是DaemonSet保证这个应用只会在一个节点上运行一个Pod，和Linux的守护进程类似。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-09 12:54:15</div>
  </div>
</div>
</div>
</li>
</ul>