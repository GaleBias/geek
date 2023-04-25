<audio title="43 _ Kubernetes默认调度器的优先级与抢占机制" src="https://static001.geekbang.org/resource/audio/dc/1a/dc0441b3e25bbee1e3c9a56cdbaf161a.mp3" controls="controls"></audio> 
<p>你好，我是张磊。今天我和你分享的主题是：Kubernetes默认调度器的优先级与抢占机制。</p><p>在上一篇文章中，我为你详细讲解了 Kubernetes 默认调度器的主要调度算法的工作原理。在本篇文章中，我再来为你讲解一下 Kubernetes 调度器里的另一个重要机制，即：优先级（Priority ）和抢占（Preemption）机制。</p><p><span class="orange">首先需要明确的是，优先级和抢占机制，解决的是 Pod 调度失败时该怎么办的问题。</span></p><p>正常情况下，当一个 Pod 调度失败后，它就会被暂时“搁置”起来，直到 Pod 被更新，或者集群状态发生变化，调度器才会对这个 Pod进行重新调度。</p><p>但在有时候，我们希望的是这样一个场景。当一个高优先级的 Pod 调度失败后，该 Pod 并不会被“搁置”，而是会“挤走”某个 Node 上的一些低优先级的 Pod 。这样就可以保证这个高优先级 Pod 的调度成功。这个特性，其实也是一直以来就存在于 Borg 以及 Mesos 等项目里的一个基本功能。</p><p>而在 Kubernetes 里，优先级和抢占机制是在1.10版本后才逐步可用的。要使用这个机制，你首先需要在 Kubernetes 里提交一个 PriorityClass 的定义，如下所示：</p><pre><code>apiVersion: scheduling.k8s.io/v1beta1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: &quot;This priority class should be used for high priority service pods only.&quot;
</code></pre><!-- [[[read_end]]] --><p>上面这个 YAML 文件，定义的是一个名叫high-priority的 PriorityClass，其中value的值是1000000 （一百万）。</p><p><strong>Kubernetes 规定，优先级是一个32 bit的整数，最大值不超过1000000000（10亿，1 billion），并且值越大代表优先级越高。</strong>而超出10亿的值，其实是被Kubernetes保留下来分配给系统 Pod使用的。显然，这样做的目的，就是保证系统 Pod 不会被用户抢占掉。</p><p>而一旦上述 YAML 文件里的 globalDefault被设置为 true 的话，那就意味着这个 PriorityClass 的值会成为系统的默认值。而如果这个值是 false，就表示我们只希望声明使用该 PriorityClass 的 Pod 拥有值为1000000的优先级，而对于没有声明 PriorityClass 的 Pod来说，它们的优先级就是0。</p><p>在创建了 PriorityClass 对象之后，Pod 就可以声明使用它了，如下所示：</p><pre><code>apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  priorityClassName: high-priority
</code></pre><p>可以看到，这个 Pod 通过priorityClassName字段，声明了要使用名叫high-priority的PriorityClass。当这个 Pod 被提交给 Kubernetes 之后，Kubernetes 的PriorityAdmissionController 就会自动将这个 Pod 的spec.priority字段设置为1000000。</p><p>而我在前面的文章中曾为你介绍过，调度器里维护着一个调度队列。所以，当 Pod 拥有了优先级之后，高优先级的 Pod 就可能会比低优先级的 Pod 提前出队，从而尽早完成调度过程。<strong>这个过程，就是“优先级”这个概念在 Kubernetes 里的主要体现。</strong></p><blockquote>
<p>备注：这里，你可以再回顾一下第41篇文章<a href="https://time.geekbang.org/column/article/69890">《十字路口上的Kubernetes默认调度器》</a>中的相关内容。</p>
</blockquote><p>而当一个高优先级的 Pod 调度失败的时候，调度器的抢占能力就会被触发。这时，调度器就会试图从当前集群里寻找一个节点，使得当这个节点上的一个或者多个低优先级 Pod 被删除后，待调度的高优先级 Pod 就可以被调度到这个节点上。<strong>这个过程，就是“抢占”这个概念在 Kubernetes 里的主要体现。</strong></p><p>为了方便叙述，我接下来会把待调度的高优先级 Pod 称为“抢占者”（Preemptor）。</p><p>当上述抢占过程发生时，抢占者并不会立刻被调度到被抢占的 Node 上。事实上，调度器只会将抢占者的spec.nominatedNodeName字段，设置为被抢占的 Node 的名字。然后，抢占者会重新进入下一个调度周期，然后在新的调度周期里来决定是不是要运行在被抢占的节点上。这当然也就意味着，即使在下一个调度周期，调度器也不会保证抢占者一定会运行在被抢占的节点上。</p><p>这样设计的一个重要原因是，调度器只会通过标准的 DELETE API 来删除被抢占的 Pod，所以，这些 Pod 必然是有一定的“优雅退出”时间（默认是30s）的。而在这段时间里，其他的节点也是有可能变成可调度的，或者直接有新的节点被添加到这个集群中来。所以，鉴于优雅退出期间，集群的可调度性可能会发生的变化，<strong>把抢占者交给下一个调度周期再处理，是一个非常合理的选择。</strong></p><p>而在抢占者等待被调度的过程中，如果有其他更高优先级的 Pod 也要抢占同一个节点，那么调度器就会清空原抢占者的spec.nominatedNodeName字段，从而允许更高优先级的抢占者执行抢占，并且，这也就使得原抢占者本身，也有机会去重新抢占其他节点。这些，都是设置nominatedNodeName字段的主要目的。</p><p>那么，<span class="orange">Kubernetes 调度器里的抢占机制，又是如何设计的呢？</span></p><p>接下来，我就为你详细讲述一下这其中的原理。</p><p>我在前面已经提到过，抢占发生的原因，一定是一个高优先级的 Pod 调度失败。这一次，我们还是称这个 Pod 为“抢占者”，称被抢占的 Pod 为“牺牲者”（victims）。</p><p>而Kubernetes 调度器实现抢占算法的一个最重要的设计，就是在调度队列的实现里，使用了两个不同的队列。</p><p><strong>第一个队列，叫作activeQ。</strong>凡是在 activeQ 里的 Pod，都是下一个调度周期需要调度的对象。所以，当你在 Kubernetes 集群里新创建一个 Pod 的时候，调度器会将这个 Pod 入队到 activeQ 里面。而我在前面提到过的、调度器不断从队列里出队（Pop）一个 Pod 进行调度，实际上都是从 activeQ 里出队的。</p><p><strong>第二个队列，叫作unschedulableQ</strong>，专门用来存放调度失败的 Pod。</p><p>而这里的一个关键点就在于，当一个unschedulableQ里的 Pod 被更新之后，调度器会自动把这个 Pod 移动到activeQ里，从而给这些调度失败的 Pod “重新做人”的机会。</p><p>现在，回到我们的抢占者调度失败这个时间点上来。</p><p>调度失败之后，抢占者就会被放进unschedulableQ里面。</p><p>然后，这次失败事件就会触发<strong>调度器为抢占者寻找牺牲者的流程</strong>。</p><p><strong>第一步</strong>，调度器会检查这次失败事件的原因，来确认抢占是不是可以帮助抢占者找到一个新节点。这是因为有很多 Predicates的失败是不能通过抢占来解决的。比如，PodFitsHost算法（负责的是，检查Pod 的 nodeSelector与 Node 的名字是否匹配），这种情况下，除非 Node 的名字发生变化，否则你即使删除再多的 Pod，抢占者也不可能调度成功。</p><p><strong>第二步</strong>，如果确定抢占可以发生，那么调度器就会把自己缓存的所有节点信息复制一份，然后使用这个副本来模拟抢占过程。</p><p>这里的抢占过程很容易理解。调度器会检查缓存副本里的每一个节点，然后从该节点上最低优先级的Pod开始，逐一“删除”这些Pod。而每删除一个低优先级Pod，调度器都会检查一下抢占者是否能够运行在该 Node 上。一旦可以运行，调度器就记录下这个 Node 的名字和被删除 Pod 的列表，这就是一次抢占过程的结果了。</p><p>当遍历完所有的节点之后，调度器会在上述模拟产生的所有抢占结果里做一个选择，找出最佳结果。而这一步的<strong>判断原则，就是尽量减少抢占对整个系统的影响</strong>。比如，需要抢占的 Pod 越少越好，需要抢占的 Pod 的优先级越低越好，等等。</p><p>在得到了最佳的抢占结果之后，这个结果里的 Node，就是即将被抢占的 Node；被删除的 Pod 列表，就是牺牲者。所以接下来，<strong>调度器就可以真正开始抢占的操作</strong>了，这个过程，可以分为三步。</p><p><strong>第一步</strong>，调度器会检查牺牲者列表，清理这些 Pod 所携带的nominatedNodeName字段。</p><p><strong>第二步</strong>，调度器会把抢占者的nominatedNodeName，设置为被抢占的Node 的名字。</p><p><strong>第三步</strong>，调度器会开启一个 Goroutine，同步地删除牺牲者。</p><p>而第二步对抢占者 Pod 的更新操作，就会触发到我前面提到的“重新做人”的流程，从而让抢占者在下一个调度周期重新进入调度流程。</p><p>所以<strong>接下来，调度器就会通过正常的调度流程把抢占者调度成功</strong>。这也是为什么，我前面会说调度器并不保证抢占的结果：在这个正常的调度流程里，是一切皆有可能的。</p><p>不过，对于任意一个待调度 Pod来说，因为有上述抢占者的存在，它的调度过程，其实是有一些特殊情况需要特殊处理的。</p><p>具体来说，在为某一对 Pod 和 Node 执行 Predicates 算法的时候，如果待检查的 Node 是一个即将被抢占的节点，即：调度队列里有nominatedNodeName字段值是该 Node 名字的 Pod 存在（可以称之为：“潜在的抢占者”）。那么，<strong>调度器就会对这个 Node ，将同样的 Predicates 算法运行两遍。</strong></p><p><strong>第一遍</strong>， 调度器会假设上述“潜在的抢占者”已经运行在这个节点上，然后执行 Predicates 算法；</p><p><strong>第二遍</strong>， 调度器会正常执行Predicates算法，即：不考虑任何“潜在的抢占者”。</p><p>而只有这两遍 Predicates 算法都能通过时，这个 Pod 和 Node 才会被认为是可以绑定（bind）的。</p><p>不难想到，这里需要执行第一遍Predicates算法的原因，是由于InterPodAntiAffinity 规则的存在。</p><p>由于InterPodAntiAffinity规则关心待考察节点上所有 Pod之间的互斥关系，所以我们在执行调度算法时必须考虑，如果抢占者已经存在于待考察 Node 上时，待调度 Pod 还能不能调度成功。</p><p>当然，这也就意味着，我们在这一步只需要考虑那些优先级等于或者大于待调度 Pod 的抢占者。毕竟对于其他较低优先级 Pod 来说，待调度 Pod 总是可以通过抢占运行在待考察 Node 上。</p><p>而我们需要执行第二遍Predicates 算法的原因，则是因为“潜在的抢占者”最后不一定会运行在待考察的 Node 上。关于这一点，我在前面已经讲解过了：Kubernetes调度器并不保证抢占者一定会运行在当初选定的被抢占的 Node 上。</p><p>以上，就是 Kubernetes 默认调度器里优先级和抢占机制的实现原理了。</p><h2>总结</h2><p>在本篇文章中，我为你详细讲述了 Kubernetes 里关于 Pod 的优先级和抢占机制的设计与实现。</p><p>这个特性在v1.11之后已经是Beta了，意味着比较稳定了。所以，我建议你在Kubernetes集群中开启这两个特性，以便实现更高的资源使用率。</p><h2>思考题</h2><p>当整个集群发生可能会影响调度结果的变化（比如，添加或者更新 Node，添加和更新 PV、Service等）时，调度器会执行一个被称为MoveAllToActiveQueue的操作，把所调度失败的 Pod 从 unscheduelableQ 移动到activeQ 里面。请问这是为什么？</p><p>一个相似的问题是，当一个已经调度成功的 Pod 被更新时，调度器则会将unschedulableQ 里所有跟这个 Pod 有 Affinity/Anti-affinity 关系的 Pod，移动到 activeQ 里面。请问这又是为什么呢？</p><p>感谢你的收听，欢迎你给我留言，也欢迎分享给更多的朋友一起阅读。</p>
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
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ch_ort</span>
  </div>
  <div class="_2_QraFYR_0"> 调度器的作用就是为Pod寻找一个合适的Node。<br><br>调度过程：待调度Pod被提交到apiServer -&gt; 更新到etcd -&gt; 调度器Watch etcd感知到有需要调度的pod（Informer） -&gt; 取出待调度Pod的信息 -&gt;Predicates： 挑选出可以运行该Pod的所有Node  -&gt;  Priority：给所有Node打分 -&gt; 将Pod绑定到得分最高的Node上 -&gt; 将Pod信息更新回Etcd -&gt; node的kubelet感知到etcd中有自己node需要拉起的pod -&gt; 取出该Pod信息，做基本的二次检测（端口，资源等） -&gt; 在node 上拉起该pod 。<br><br>Predicates阶段会有很多过滤规则：比如volume相关，node相关，pod相关<br>Priorities阶段会为Node打分，Pod调度到得分最高的Node上，打分规则比如： 空余资源、实际物理剩余、镜像大小、Pod亲和性等<br><br>Kuberentes中可以为Pod设置优先级，高优先级的Pod可以： 1、在调度队列中先出队进行调度 2、调度失败时，触发抢占，调度器为其抢占低优先级Pod的资源。<br><br>Kuberentes默认调度器有两个调度队列：<br>activeQ：凡事在该队列里的Pod，都是下一个调度周期需要调度的<br>unschedulableQ:  存放调度失败的Pod，当里面的Pod更新后就会重新回到activeQ，进行“重新调度”<br><br>默认调度器的抢占过程： 确定要发生抢占 -&gt; 调度器将所有节点信息复制一份，开始模拟抢占 -&gt;  检查副本里的每一个节点，然后从该节点上逐个删除低优先级Pod，直到满足抢占者能运行 -&gt; 找到一个能运行抢占者Pod的node -&gt; 记录下这个Node名字和被删除Pod的列表 -&gt; 模拟抢占结束 -&gt; 开始真正抢占 -&gt; 删除被抢占者的Pod，将抢占者调度到Node上 <br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-20 15:57:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/f7/7a/55618020.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>马若飞</span>
  </div>
  <div class="_2_QraFYR_0">粥变多了，所有的僧可以重新排队领取了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-21 09:20:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/8c/b2/dfdcc8f3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>黄巍</span>
  </div>
  <div class="_2_QraFYR_0">「调度器会开启一个 Goroutine，异步地删除牺牲者。」这里应该是同步的 :)</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">编辑回复: 已更新，感谢反馈~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-05 13:39:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/2d/24/28acca15.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>DJH</span>
  </div>
  <div class="_2_QraFYR_0">因为第一种情况下集群资源发生了变化，原先无法调度的pod可能有了可调度的节点或资源，不再需要通过抢占来实现。<br><br>第二种情况是放pod调度成功后，跟这个pod有亲和性和反亲和性规则的pod需要重新过滤一次可用节点。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-30 08:27:20</div>
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
  <div class="_2_QraFYR_0">Question 1:<br>Add&#47;update new node&#47;pv&#47;pvc&#47;service will cause the change to predicates, which may make pending pods schedulable.<br><br>Question 2:<br>when a bound pod is added, creation of this pod may make pending pods with matching affinity terms schedulable.<br><br>when a bound pod is updated, change of labels may make pending pods with matching affinity terms schedulable.<br><br>when a bound pod is deleted, MatchInterPodAffinity need to be reconsidered for this node, as well as all nodes in its same failure domain.</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-30 12:48:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/04/0d/3dc5683a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>柯察金</span>
  </div>
  <div class="_2_QraFYR_0">为什么在为某一对 Pod 和 Node 执行 Predicates 算法的时候，如果待检查的 Node 是一个即将被抢占的节点，调度器就会对这个 Node ，将同样的 Predicates 算法运行两遍？<br><br>感觉执行第一遍就可以了啊，难道执行第一遍成功了，在执行第二遍的时候还可能会失败吗？感觉第一遍条件比第二遍苛刻啊，如果第一遍 ok 第二遍也会通过的</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-25 17:38:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/aa/9a/92d2df36.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>tianfeiyu</span>
  </div>
  <div class="_2_QraFYR_0">你好，我看了scheduler 代码，若抢占成功应该是更新  status.nominatedNodeName 不是 spec.nominatedNodeName 字段</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-24 10:22:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/e9/91/4219d305.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>初学者</span>
  </div>
  <div class="_2_QraFYR_0">不太理解service与pod scheduling 流程有什么直接关系？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 比如，同一个service 下的pod可以被打散。设置更复杂的service affinity规则。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-02 09:32:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/80/a5/abb7bfe3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>要没时间了</span>
  </div>
  <div class="_2_QraFYR_0">对于抢占调度，更多的是处理“使用了PriorityClass”且“需求资源不足”的Pod的调度失败case。在确定候选的nominatedNode的过程中，一个很重要的步骤就是模拟删除所有低优先级的pod，看剩余的资源是否符合高优先级Pod的需求。<br><br>回忆下上一章提到的调度算法也能够清楚，除了资源不足的原因之外，其他调度失败的原因很难通过抢占来进行恢复。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-20 14:31:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/69/4d/81c44f45.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>拉欧</span>
  </div>
  <div class="_2_QraFYR_0">增加调度效率吧，新增加资源的时候，一定有机会加速调度pod; 而一旦某个pod调度程度，马上检验和其相关的pod是否可调度</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-23 15:55:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/69/88/528442b0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Dale</span>
  </div>
  <div class="_2_QraFYR_0">1、集群有更新，需要将失败的pod重新调度，放到ActiveQ中可以重新触发调度策略<br>2、在predicate阶段，会对pod的node selector进行判断，寻找合适的node节点，需要通过将pod放到ActiveQ中重新触发predicate调度策略</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-13 14:25:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/24/b4/23/0e296758.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>姜尧</span>
  </div>
  <div class="_2_QraFYR_0">pod驱逐是不是也用到了好多这节讲的api？？？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-18 08:40:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/69/11/831cec7d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小寞子。(≥3≤)</span>
  </div>
  <div class="_2_QraFYR_0">没有看代码 但是感觉kubernetes 在schedule方面还是有很多可以优化的空间吧 这些predicate 算法， 如果有几万个pod, 几千个node情况下 还能被几个master node 上面的scheduler 运行么？ cache同步都是个头疼吧。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-01 12:43:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/aa/9a/92d2df36.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>tianfeiyu</span>
  </div>
  <div class="_2_QraFYR_0">想问一下，没有开启优先级的 pod 没有 status.NominatedNodeName 字段，抢占过程这些 pod 也会被抢占吗？<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-23 17:29:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/df/e2/823a04b4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小小笑儿</span>
  </div>
  <div class="_2_QraFYR_0">第一个问题:添加更新node,pv可能让pod变成可调度的状态，就不用走抢占的流程了，service的不太明白。<br>第二个问题:对anti的同上</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-30 08:25:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/77/05/49ea32f0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Unplug</span>
  </div>
  <div class="_2_QraFYR_0">typo: “把所调度失败的 Pod 从 unscheduelableQ 移动到 activeQ 里面。请问这是为什么” 中的 unscheduelableQ 拼错了。应该是 unschedulableQ。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-23 03:37:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/65/3a/bc801fb2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>mqray</span>
  </div>
  <div class="_2_QraFYR_0">为了确保所有的Pod都能够及时地被调度到可用的节点上，从而保证集群的高可用性和稳定性。如果不执行这个操作，那么一些本来可以被调度的Pod可能会一直处于未调度状态，从而导致应用程序无法正常运行。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-07 17:34:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/24/70/4e7751f3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>超级芒果冰</span>
  </div>
  <div class="_2_QraFYR_0">“当然，这也就意味着，我们在这一步只需要考虑那些优先级等于或者大于待调度 Pod 的抢占者。毕竟对于其他较低优先级 Pod 来说，待调度 Pod 总是可以通过抢占运行在待考察 Node 上。”<br>老师这段话不理解，感觉这段话和它上面的描述联系不起来？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-05 00:10:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJrqLEic7DVicYY1s9ldH0vGBialDoplVGpicZUJ0Fdaklw27Frv8Ac67eicb5LibhL74SUxAzlick2nfltA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jiangb</span>
  </div>
  <div class="_2_QraFYR_0">就是操作系统啊</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-22 17:45:40</div>
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
  <div class="_2_QraFYR_0">“第一遍， 调度器会假设上述“潜在的抢占者”已经运行在这个节点上，然后执行 Predicates 算法；<br>第二遍， 调度器会正常执行 Predicates 算法，即：不考虑任何“潜在的抢占者”。”<br><br>对这一段的机制，我觉得如果第一遍能执行通过，第二遍不就是必定会执行通过吗？为啥还要多此一举执行第二遍呢？有没有人和我有同样的疑惑？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-18 14:41:18</div>
  </div>
</div>
</div>
</li>
</ul>