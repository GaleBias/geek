<audio title="22 _ 撬动离线业务：Job与CronJob" src="https://static001.geekbang.org/resource/audio/4c/b0/4c14724d4047a3094b87fcec1f8239b0.mp3" controls="controls"></audio> 
<p>你好，我是张磊。今天我和你分享的主题是：撬动离线业务之Job与CronJob。</p><p>在前面的几篇文章中，我和你详细分享了Deployment、StatefulSet，以及DaemonSet这三个编排概念。你有没有发现它们的共同之处呢？</p><p>实际上，它们主要编排的对象，都是“在线业务”，即：Long Running Task（长作业）。比如，我在前面举例时常用的Nginx、Tomcat，以及MySQL等等。这些应用一旦运行起来，除非出错或者停止，它的容器进程会一直保持在Running状态。</p><p>但是，有一类作业显然不满足这样的条件，这就是“离线业务”，或者叫作Batch Job（计算业务）。这种业务在计算完成后就直接退出了，而此时如果你依然用Deployment来管理这种业务的话，就会发现Pod会在计算结束后退出，然后被Deployment Controller不断地重启；而像“滚动更新”这样的编排功能，更无从谈起了。</p><p>所以，早在Borg项目中，Google就已经对作业进行了分类处理，提出了LRS（Long Running Service）和Batch Jobs两种作业形态，对它们进行“分别管理”和“混合调度”。</p><p>不过，在2015年Borg论文刚刚发布的时候，Kubernetes项目并不支持对Batch Job的管理。直到v1.4版本之后，社区才逐步设计出了一个用来描述离线业务的API对象，它的名字就是：Job。</p><!-- [[[read_end]]] --><p><span class="orange">Job API对象的定义非常简单，我来举个例子</span>，如下所示：</p><pre><code>apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
      - name: pi
        image: resouer/ubuntu-bc 
        command: [&quot;sh&quot;, &quot;-c&quot;, &quot;echo 'scale=10000; 4*a(1)' | bc -l &quot;]
      restartPolicy: Never
  backoffLimit: 4
</code></pre><p>此时，相信你对Kubernetes的API对象已经不再陌生了。在这个Job的YAML文件里，你肯定一眼就会看到一位“老熟人”：Pod模板，即spec.template字段。</p><p>在这个Pod模板中，我定义了一个Ubuntu镜像的容器（准确地说，是一个安装了bc命令的Ubuntu镜像），它运行的程序是：</p><pre><code>echo &quot;scale=10000; 4*a(1)&quot; | bc -l 
</code></pre><p>其中，bc命令是Linux里的“计算器”；-l表示，我现在要使用标准数学库；而a(1)，则是调用数学库中的arctangent函数，计算atan(1)。这是什么意思呢？</p><p>中学知识告诉我们：<code>tan(π/4) = 1</code>。所以，<code>4*atan(1)</code>正好就是π，也就是3.1415926…。</p><blockquote>
<p>备注：如果你不熟悉这个知识也不必担心，我也是在查阅资料后才知道的。</p>
</blockquote><p>所以，这其实就是一个计算π值的容器。而通过scale=10000，我指定了输出的小数点后的位数是10000。在我的计算机上，这个计算大概用时1分54秒。</p><p>但是，跟其他控制器不同的是，Job对象并不要求你定义一个spec.selector来描述要控制哪些Pod。具体原因，我马上会讲解到。</p><p>现在，我们就可以创建这个Job了：</p><pre><code>$ kubectl create -f job.yaml
</code></pre><p>在成功创建后，我们来查看一下这个Job对象，如下所示：</p><pre><code>$ kubectl describe jobs/pi
Name:             pi
Namespace:        default
Selector:         controller-uid=c2db599a-2c9d-11e6-b324-0209dc45a495
Labels:           controller-uid=c2db599a-2c9d-11e6-b324-0209dc45a495
                  job-name=pi
Annotations:      &lt;none&gt;
Parallelism:      1
Completions:      1
..
Pods Statuses:    0 Running / 1 Succeeded / 0 Failed
Pod Template:
  Labels:       controller-uid=c2db599a-2c9d-11e6-b324-0209dc45a495
                job-name=pi
  Containers:
   ...
  Volumes:              &lt;none&gt;
Events:
  FirstSeen    LastSeen    Count    From            SubobjectPath    Type        Reason            Message
  ---------    --------    -----    ----            -------------    --------    ------            -------
  1m           1m          1        {job-controller }                Normal      SuccessfulCreate  Created pod: pi-rq5rl
</code></pre><p>可以看到，这个Job对象在创建后，它的Pod模板，被自动加上了一个controller-uid=&lt;一个随机字符串&gt;这样的Label。而这个Job对象本身，则被自动加上了这个Label对应的Selector，从而 保证了Job与它所管理的Pod之间的匹配关系。</p><p>而Job Controller之所以要使用这种携带了UID的Label，就是为了避免不同Job对象所管理的Pod发生重合。需要注意的是，<strong>这种自动生成的Label对用户来说并不友好，所以不太适合推广到Deployment等长作业编排对象上。</strong></p><p>接下来，我们可以看到这个Job创建的Pod进入了Running状态，这意味着它正在计算Pi的值。</p><pre><code>$ kubectl get pods
NAME                                READY     STATUS    RESTARTS   AGE
pi-rq5rl                            1/1       Running   0          10s
</code></pre><p>而几分钟后计算结束，这个Pod就会进入Completed状态：</p><pre><code>$ kubectl get pods
NAME                                READY     STATUS      RESTARTS   AGE
pi-rq5rl                            0/1       Completed   0          4m
</code></pre><p>这也是我们需要在Pod模板中定义restartPolicy=Never的原因：离线计算的Pod永远都不应该被重启，否则它们会再重新计算一遍。</p><blockquote>
<p>事实上，restartPolicy在Job对象里只允许被设置为Never和OnFailure；而在Deployment对象里，restartPolicy则只允许被设置为Always。</p>
</blockquote><p>此时，我们通过kubectl logs查看一下这个Pod的日志，就可以看到计算得到的Pi值已经被打印了出来：</p><pre><code>$ kubectl logs pi-rq5rl
3.141592653589793238462643383279...
</code></pre><p>这时候，你一定会想到这样一个问题，<strong>如果这个离线作业失败了要怎么办？</strong></p><p>比如，我们在这个例子中<strong>定义了restartPolicy=Never，那么离线作业失败后Job Controller就会不断地尝试创建一个新Pod</strong>，如下所示：</p><pre><code>$ kubectl get pods
NAME                                READY     STATUS              RESTARTS   AGE
pi-55h89                            0/1       ContainerCreating   0          2s
pi-tqbcz                            0/1       Error               0          5s
</code></pre><p>可以看到，这时候会不断地有新Pod被创建出来。</p><p>当然，这个尝试肯定不能无限进行下去。所以，我们就在Job对象的spec.backoffLimit字段里定义了重试次数为4（即，backoffLimit=4），而这个字段的默认值是6。</p><p>需要注意的是，Job Controller重新创建Pod的间隔是呈指数增加的，即下一次重新创建Pod的动作会分别发生在10 s、20 s、40 s …后。</p><p>而如果你<strong>定义的restartPolicy=OnFailure，那么离线作业失败后，Job Controller就不会去尝试创建新的Pod。但是，它会不断地尝试重启Pod里的容器</strong>。这也正好对应了restartPolicy的含义（你也可以借此机会再回顾一下第15篇文章<a href="https://time.geekbang.org/column/article/40466">《深入解析Pod对象（二）：使用进阶》</a>中的相关内容）。</p><p>如前所述，当一个Job的Pod运行结束后，它会进入Completed状态。但是，如果这个Pod因为某种原因一直不肯结束呢？</p><p>在Job的API对象里，有一个spec.activeDeadlineSeconds字段可以设置最长运行时间，比如：</p><pre><code>spec:
 backoffLimit: 5
 activeDeadlineSeconds: 100
</code></pre><p>一旦运行超过了100 s，这个Job的所有Pod都会被终止。并且，你可以在Pod的状态里看到终止的原因是reason: DeadlineExceeded。</p><p>以上，就是一个Job API对象最主要的概念和用法了。不过，离线业务之所以被称为Batch Job，当然是因为它们可以以“Batch”，也就是并行的方式去运行。</p><p>接下来，我就来为你讲解一下<span class="orange">Job Controller对并行作业的控制方法。</span></p><p>在Job对象中，负责并行控制的参数有两个：</p><ol>
<li>
<p>spec.parallelism，它定义的是一个Job在任意时间最多可以启动多少个Pod同时运行；</p>
</li>
<li>
<p>spec.completions，它定义的是Job至少要完成的Pod数目，即Job的最小完成数。</p>
</li>
</ol><p>这两个参数听起来有点儿抽象，所以我准备了一个例子来帮助你理解。</p><p>现在，我在之前计算Pi值的Job里，添加这两个参数：</p><pre><code>apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  parallelism: 2
  completions: 4
  template:
    spec:
      containers:
      - name: pi
        image: resouer/ubuntu-bc
        command: [&quot;sh&quot;, &quot;-c&quot;, &quot;echo 'scale=5000; 4*a(1)' | bc -l &quot;]
      restartPolicy: Never
  backoffLimit: 4
</code></pre><p>这样，我们就指定了这个Job最大的并行数是2，而最小的完成数是4。</p><p>接下来，我们来创建这个Job对象：</p><pre><code>$ kubectl create -f job.yaml
</code></pre><p>可以看到，这个Job其实也维护了两个状态字段，即DESIRED和SUCCESSFUL，如下所示：</p><pre><code>$ kubectl get job
NAME      DESIRED   SUCCESSFUL   AGE
pi        4         0            3s
</code></pre><p>其中，DESIRED的值，正是completions定义的最小完成数。</p><p>然后，我们可以看到，这个Job首先创建了两个并行运行的Pod来计算Pi：</p><pre><code>$ kubectl get pods
NAME       READY     STATUS    RESTARTS   AGE
pi-5mt88   1/1       Running   0          6s
pi-gmcq5   1/1       Running   0          6s
</code></pre><p>而在40 s后，这两个Pod相继完成计算。</p><p>这时我们可以看到，每当有一个Pod完成计算进入Completed状态时，就会有一个新的Pod被自动创建出来，并且快速地从Pending状态进入到ContainerCreating状态：</p><pre><code>$ kubectl get pods
NAME       READY     STATUS    RESTARTS   AGE
pi-gmcq5   0/1       Completed   0         40s
pi-84ww8   0/1       Pending   0         0s
pi-5mt88   0/1       Completed   0         41s
pi-62rbt   0/1       Pending   0         0s

$ kubectl get pods
NAME       READY     STATUS    RESTARTS   AGE
pi-gmcq5   0/1       Completed   0         40s
pi-84ww8   0/1       ContainerCreating   0         0s
pi-5mt88   0/1       Completed   0         41s
pi-62rbt   0/1       ContainerCreating   0         0s
</code></pre><p>紧接着，Job Controller第二次创建出来的两个并行的Pod也进入了Running状态：</p><pre><code>$ kubectl get pods 
NAME       READY     STATUS      RESTARTS   AGE
pi-5mt88   0/1       Completed   0          54s
pi-62rbt   1/1       Running     0          13s
pi-84ww8   1/1       Running     0          14s
pi-gmcq5   0/1       Completed   0          54s
</code></pre><p>最终，后面创建的这两个Pod也完成了计算，进入了Completed状态。</p><p>这时，由于所有的Pod均已经成功退出，这个Job也就执行完了，所以你会看到它的SUCCESSFUL字段的值变成了4：</p><pre><code>$ kubectl get pods 
NAME       READY     STATUS      RESTARTS   AGE
pi-5mt88   0/1       Completed   0          5m
pi-62rbt   0/1       Completed   0          4m
pi-84ww8   0/1       Completed   0          4m
pi-gmcq5   0/1       Completed   0          5m

$ kubectl get job
NAME      DESIRED   SUCCESSFUL   AGE
pi        4         4            5m
</code></pre><p>通过上述Job的DESIRED和SUCCESSFUL字段的关系，我们就可以很容易地理解<span class="orange">Job Controller的工作原理</span>了。</p><p>首先，Job Controller控制的对象，直接就是Pod。</p><p>其次，Job Controller在控制循环中进行的调谐（Reconcile）操作，是根据实际在Running状态Pod的数目、已经成功退出的Pod的数目，以及parallelism、completions参数的值共同计算出在这个周期里，应该创建或者删除的Pod数目，然后调用Kubernetes API来执行这个操作。</p><p>以创建Pod为例。在上面计算Pi值的这个例子中，当Job一开始创建出来时，实际处于Running状态的Pod数目=0，已经成功退出的Pod数目=0，而用户定义的completions，也就是最终用户需要的Pod数目=4。</p><p>所以，在这个时刻，需要创建的Pod数目 = 最终需要的Pod数目 - 实际在Running状态Pod数目 - 已经成功退出的Pod数目 = 4 - 0 - 0= 4。也就是说，Job Controller需要创建4个Pod来纠正这个不一致状态。</p><p>可是，我们又定义了这个Job的parallelism=2。也就是说，我们规定了每次并发创建的Pod个数不能超过2个。所以，Job Controller会对前面的计算结果做一个修正，修正后的期望创建的Pod数目应该是：2个。</p><p>这时候，Job Controller就会并发地向kube-apiserver发起两个创建Pod的请求。</p><p>类似地，如果在这次调谐周期里，Job Controller发现实际在Running状态的Pod数目，比parallelism还大，那么它就会删除一些Pod，使两者相等。</p><p>综上所述，Job Controller实际上控制了，作业执行的<strong>并行度</strong>，以及总共需要完成的<strong>任务数</strong>这两个重要参数。而在实际使用时，你需要根据作业的特性，来决定并行度（parallelism）和任务数（completions）的合理取值。</p><p>接下来，<span class="orange">我再和你分享三种常用的、使用Job对象的方法。</span></p><p><strong>第一种用法，也是最简单粗暴的用法：外部管理器+Job模板。</strong></p><p>这种模式的特定用法是：把Job的YAML文件定义为一个“模板”，然后用一个外部工具控制这些“模板”来生成Job。这时，Job的定义方式如下所示：</p><pre><code>apiVersion: batch/v1
kind: Job
metadata:
  name: process-item-$ITEM
  labels:
    jobgroup: jobexample
spec:
  template:
    metadata:
      name: jobexample
      labels:
        jobgroup: jobexample
    spec:
      containers:
      - name: c
        image: busybox
        command: [&quot;sh&quot;, &quot;-c&quot;, &quot;echo Processing item $ITEM &amp;&amp; sleep 5&quot;]
      restartPolicy: Never
</code></pre><p>可以看到，我们在这个Job的YAML里，定义了$ITEM这样的“变量”。</p><p>所以，在控制这种Job时，我们只要注意如下两个方面即可：</p><ol>
<li>
<p>创建Job时，替换掉$ITEM这样的变量；</p>
</li>
<li>
<p>所有来自于同一个模板的Job，都有一个jobgroup: jobexample标签，也就是说这一组Job使用这样一个相同的标识。</p>
</li>
</ol><p>而做到第一点非常简单。比如，你可以通过这样一句shell把$ITEM替换掉：</p><pre><code>$ mkdir ./jobs
$ for i in apple banana cherry
do
  cat job-tmpl.yaml | sed &quot;s/\$ITEM/$i/&quot; &gt; ./jobs/job-$i.yaml
done
</code></pre><p>这样，一组来自于同一个模板的不同Job的yaml就生成了。接下来，你就可以通过一句kubectl create指令创建这些Job了：</p><pre><code>$ kubectl create -f ./jobs
$ kubectl get pods -l jobgroup=jobexample
NAME                        READY     STATUS      RESTARTS   AGE
process-item-apple-kixwv    0/1       Completed   0          4m
process-item-banana-wrsf7   0/1       Completed   0          4m
process-item-cherry-dnfu9   0/1       Completed   0          4m
</code></pre><p>这个模式看起来虽然很“傻”，但却是Kubernetes社区里使用Job的一个很普遍的模式。</p><p>原因很简单：大多数用户在需要管理Batch Job的时候，都已经有了一套自己的方案，需要做的往往就是集成工作。这时候，Kubernetes项目对这些方案来说最有价值的，就是Job这个API对象。所以，你只需要编写一个外部工具（等同于我们这里的for循环）来管理这些Job即可。</p><p>这种模式最典型的应用，就是TensorFlow社区的KubeFlow项目。</p><p>很容易理解，在这种模式下使用Job对象，completions和parallelism这两个字段都应该使用默认值1，而不应该由我们自行设置。而作业Pod的并行控制，应该完全交由外部工具来进行管理（比如，KubeFlow）。</p><p><strong>第二种用法：拥有固定任务数目的并行Job</strong>。</p><p>这种模式下，我只关心最后是否有指定数目（spec.completions）个任务成功退出。至于执行时的并行度是多少，我并不关心。</p><p>比如，我们这个计算Pi值的例子，就是这样一个典型的、拥有固定任务数目（completions=4）的应用场景。 它的parallelism值是2；或者，你可以干脆不指定parallelism，直接使用默认的并行度（即：1）。</p><p>此外，你还可以使用一个工作队列（Work Queue）进行任务分发。这时，Job的YAML文件定义如下所示：</p><pre><code>apiVersion: batch/v1
kind: Job
metadata:
  name: job-wq-1
spec:
  completions: 8
  parallelism: 2
  template:
    metadata:
      name: job-wq-1
    spec:
      containers:
      - name: c
        image: myrepo/job-wq-1
        env:
        - name: BROKER_URL
          value: amqp://guest:guest@rabbitmq-service:5672
        - name: QUEUE
          value: job1
      restartPolicy: OnFailure
</code></pre><p>我们可以看到，它的completions的值是：8，这意味着我们总共要处理的任务数目是8个。也就是说，总共会有8个任务会被逐一放入工作队列里（你可以运行一个外部小程序作为生产者，来提交任务）。</p><p>在这个实例中，我选择充当工作队列的是一个运行在Kubernetes里的RabbitMQ。所以，我们需要在Pod模板里定义BROKER_URL，来作为消费者。</p><p>所以，一旦你用kubectl create创建了这个Job，它就会以并发度为2的方式，每两个Pod一组，创建出8个Pod。每个Pod都会去连接BROKER_URL，从RabbitMQ里读取任务，然后各自进行处理。这个Pod里的执行逻辑，我们可以用这样一段伪代码来表示：</p><pre><code>/* job-wq-1的伪代码 */
queue := newQueue($BROKER_URL, $QUEUE)
task := queue.Pop()
process(task)
exit
</code></pre><p>可以看到，每个Pod只需要将任务信息读取出来，处理完成，然后退出即可。而作为用户，我只关心最终一共有8个计算任务启动并且退出，只要这个目标达到，我就认为整个Job处理完成了。所以说，这种用法，对应的就是“任务总数固定”的场景。</p><p><strong>第三种用法，也是很常用的一个用法：指定并行度（parallelism），但不设置固定的completions的值。</strong></p><p>此时，你就必须自己想办法，来决定什么时候启动新Pod，什么时候Job才算执行完成。在这种情况下，任务的总数是未知的，所以你不仅需要一个工作队列来负责任务分发，还需要能够判断工作队列已经为空（即：所有的工作已经结束了）。</p><p>这时候，Job的定义基本上没变化，只不过是不再需要定义completions的值了而已：</p><pre><code>apiVersion: batch/v1
kind: Job
metadata:
  name: job-wq-2
spec:
  parallelism: 2
  template:
    metadata:
      name: job-wq-2
    spec:
      containers:
      - name: c
        image: gcr.io/myproject/job-wq-2
        env:
        - name: BROKER_URL
          value: amqp://guest:guest@rabbitmq-service:5672
        - name: QUEUE
          value: job2
      restartPolicy: OnFailure
</code></pre><p>而对应的Pod的逻辑会稍微复杂一些，我可以用这样一段伪代码来描述：</p><pre><code>/* job-wq-2的伪代码 */
for !queue.IsEmpty($BROKER_URL, $QUEUE) {
  task := queue.Pop()
  process(task)
}
print(&quot;Queue empty, exiting&quot;)
exit
</code></pre><p>由于任务数目的总数不固定，所以每一个Pod必须能够知道，自己什么时候可以退出。比如，在这个例子中，我简单地以“队列为空”，作为任务全部完成的标志。所以说，这种用法，对应的是“任务总数不固定”的场景。</p><p>不过，在实际的应用中，你需要处理的条件往往会非常复杂。比如，任务完成后的输出、每个任务Pod之间是不是有资源的竞争和协同等等。</p><p>所以，在今天这篇文章中，我就不再展开Job的用法了。因为，在实际场景里，要么干脆就用第一种用法来自己管理作业；要么，这些任务Pod之间的关系就不那么“单纯”，甚至还是“有状态应用”（比如，任务的输入/输出是在持久化数据卷里）。在这种情况下，我在后面要重点讲解的Operator，加上Job对象一起，可能才能更好地满足实际离线任务的编排需求。</p><p><span class="orange">最后，我再来和你分享一个非常有用的Job对象，叫作：CronJob。</span></p><p>顾名思义，CronJob描述的，正是定时任务。它的API对象，如下所示：</p><pre><code>apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: &quot;*/1 * * * *&quot;
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
</code></pre><p>在这个YAML文件中，最重要的关键词就是<strong>jobTemplate</strong>。看到它，你一定恍然大悟，原来CronJob是一个Job对象的控制器（Controller）！</p><p>没错，CronJob与Job的关系，正如同Deployment与ReplicaSet的关系一样。CronJob是一个专门用来管理Job对象的控制器。只不过，它创建和删除Job的依据，是schedule字段定义的、一个标准的<a href="https://en.wikipedia.org/wiki/Cron">Unix Cron</a>格式的表达式。</p><p>比如，"*/1 * * * *"。</p><p>这个Cron表达式里*/1中的*表示从0开始，/表示“每”，1表示偏移量。所以，它的意思就是：从0开始，每1个时间单位执行一次。</p><p>那么，时间单位又是什么呢？</p><p>Cron表达式中的五个部分分别代表：分钟、小时、日、月、星期。</p><p>所以，上面这句Cron表达式的意思是：从当前开始，每分钟执行一次。</p><p>而这里要执行的内容，就是jobTemplate定义的Job了。</p><p>所以，这个CronJob对象在创建1分钟后，就会有一个Job产生了，如下所示：</p><pre><code>$ kubectl create -f ./cronjob.yaml
cronjob &quot;hello&quot; created

# 一分钟后
$ kubectl get jobs
NAME               DESIRED   SUCCESSFUL   AGE
hello-4111706356   1         1         2s
</code></pre><p>此时，CronJob对象会记录下这次Job执行的时间：</p><pre><code>$ kubectl get cronjob hello
NAME      SCHEDULE      SUSPEND   ACTIVE    LAST-SCHEDULE
hello     */1 * * * *   False     0         Thu, 6 Sep 2018 14:34:00 -070
</code></pre><p>需要注意的是，由于定时任务的特殊性，很可能某个Job还没有执行完，另外一个新Job就产生了。这时候，你可以通过spec.concurrencyPolicy字段来定义具体的处理策略。比如：</p><ol>
<li>
<p>concurrencyPolicy=Allow，这也是默认情况，这意味着这些Job可以同时存在；</p>
</li>
<li>
<p>concurrencyPolicy=Forbid，这意味着不会创建新的Pod，该创建周期被跳过；</p>
</li>
<li>
<p>concurrencyPolicy=Replace，这意味着新产生的Job会替换旧的、没有执行完的Job。</p>
</li>
</ol><p>而如果某一次Job创建失败，这次创建就会被标记为“miss”。当在指定的时间窗口内，miss的数目达到100时，那么CronJob会停止再创建这个Job。</p><p>这个时间窗口，可以由spec.startingDeadlineSeconds字段指定。比如startingDeadlineSeconds=200，意味着在过去200 s里，如果miss的数目达到了100次，那么这个Job就不会被创建执行了。</p><h2>总结</h2><p>在今天这篇文章中，我主要和你分享了Job这个离线业务的编排方法，讲解了completions和parallelism字段的含义，以及Job Controller的执行原理。</p><p>紧接着，我通过实例和你分享了Job对象三种常见的使用方法。但是，根据我在社区和生产环境中的经验，大多数情况下用户还是更倾向于自己控制Job对象。所以，相比于这些固定的“模式”，掌握Job的API对象，和它各个字段的准确含义会更加重要。</p><p>最后，我还介绍了一种Job的控制器，叫作：CronJob。这也印证了我在前面的分享中所说的：用一个对象控制另一个对象，是Kubernetes编排的精髓所在。</p><h2>思考题</h2><p>根据Job控制器的工作原理，如果你定义的parallelism比completions还大的话，比如：</p><pre><code> parallelism: 4
 completions: 2
</code></pre><p>那么，这个Job最开始创建的时候，会同时启动几个Pod呢？原因是什么？</p><p>感谢你的收听，欢迎你给我留言，也欢迎分享给更多的朋友一起阅读。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/44/68/5d1fcf32.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>刘孟</span>
  </div>
  <div class="_2_QraFYR_0">需要创建的 Pod 数目 = 最终需要的 Pod 数目 - 实际在 Running 状态 Pod 数目 - 已经成功退出的 Pod 数目 = 2 - 0 - 0= 2。而parallelism数量为4，2小于4，所以应该会创建2个。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 聪明宝宝</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-12 09:11:09</div>
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
  <div class="_2_QraFYR_0">说到pod的重新启动，我想再请教一个问题：假设我把deployment的restart policy设置成always，假设某个pod中的容器运行失败，那么是重新创建了一个新的pod，还是仅仅重启了pod里的容器？pod的名称和ip地址会变化吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: restartpoliccy当然是针对容器。pod没有重启这个说法。至于ip，那跟用户容器没关系，那是infra container</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-12 13:10:25</div>
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
  <div class="_2_QraFYR_0">问个不相关的问题：configmap 更新，怎么做到不重启 pod 生效</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这得看应用咋写的。configmap volume里的内容已经就是自动更新的。但应用能做到监视文件的更新吗？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-03 19:54:03</div>
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
  <div class="_2_QraFYR_0">job 执行结束，处于 completed 状态之后，还会占用系统资源吗，可以让它执行结束后自动退出吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 需要删除掉，或者设置规则</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-03 09:23:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/48/fc/f46062b6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>abc</span>
  </div>
  <div class="_2_QraFYR_0">请问老师：miss的数目100是默认的吗？哪个参数可以修改呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个是写死的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-12 15:55:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/5a/56/115c6433.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jssfy</span>
  </div>
  <div class="_2_QraFYR_0">请问job成功结束后一直处于completed状态吗？需要手动清吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-25 01:31:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/60/d9/829ac53b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>fangxuan</span>
  </div>
  <div class="_2_QraFYR_0">从公式可以看出，启动的job的最大值由completion决定</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-23 07:52:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/1b/b4/a6db1c1e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>silver</span>
  </div>
  <div class="_2_QraFYR_0">Job的第三种方法中是不是需要有其他process最后去Kill这个Job，否则Job会在Pod退出后不断创建新的Pod？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不需要的。completion没设置默认等于1，所以任何一个pod判断到队列为空退出进入succeed状态，Job controller就不会再创建新pod了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-14 07:58:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/18/e7/d58e287c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>参悟</span>
  </div>
  <div class="_2_QraFYR_0">最近实践遇到的问题，盼请赐教，多套分支的开发，测试环境，是按一套k8s集群按命名空间分区，还是按多套集群，实践方案哪种更好，有何优缺点？如果按一套多空间，会有nodeport冲突的问题，比如数据库需要暴露稳定的端口，方便运维。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 多套集群。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-12 20:57:21</div>
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
  <div class="_2_QraFYR_0">K8S支持编排长期运行作业和执行完即退出的作业：<br><br>（1）支持Long Running Job（长期运行的作业）： Deployment 、StatefulSet、DaemonSet<br><br>（2）这次好Batch Job（执行完即退出的作业）：Job、CronJob<br><br>CronJob与Job关系，正如同Deployment与ReplicaSet的关系一样。CronJob是一个专门用来管理Job对象的控制器。只不过，它创建和删除Job的依据，是schedule字段定义的<br><br><br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-26 16:13:30</div>
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
  <div class="_2_QraFYR_0">并发度为4，意味着可以同时启动不超过4个job。<br><br>completion2 - running0 - completed0 = 2<br><br>所以会启动2个job</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不错</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-12 11:06:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>hochuenw</span>
  </div>
  <div class="_2_QraFYR_0">老师请问kubeflow在哪一部分用到了第一种job的使用方法？他们不是自己写了tf-operator吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你说operator 管理的是啥呢？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-04 15:50:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Nokiak8</span>
  </div>
  <div class="_2_QraFYR_0">老师，请教一个问题，假如Job中定义的pod运行失败，比如有异常。pod就会接着新生成，这样带来的就是会有大量的pod 产生，如何解决这种问题？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 有个字段叫backofflimit</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-18 09:33:07</div>
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
  <div class="_2_QraFYR_0">放miss数量达到100时，cronjob是永远不再创建新的job（相当于整个cronjob失效），亦或只是不再运行miss（错过）的那些job？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 当然是失效了。这是保护措施。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-12 13:51:21</div>
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
  <div class="_2_QraFYR_0">另外我想请教一下，CronJob是定期产生新的Job，还是定期重启同一个Job任务？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 当然是产生新的job</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-12 07:54:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>georgesuper GoodTOGreater</span>
  </div>
  <div class="_2_QraFYR_0">是不是Spark job,hadoop job,k8s job 底层原理都相似？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个差别可就太大了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-20 18:54:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/1f/64/54458855.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Caesar</span>
  </div>
  <div class="_2_QraFYR_0">(1) 若所有容器重启成功，pod应该是running，或者succeeded<br>(2) 若有容器没有成功，看具体原因,pending,failed...</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对头</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-12 09:52:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://wx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTIC5FK1ibcFwyTQ5TugfhJicSsZ3x5GfibRrUNTxpb8IY88wNREl4GlbJqUUibCHAhZp9wqic2eia2Dpgsw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>江</span>
  </div>
  <div class="_2_QraFYR_0">请教一下，如果多个job 间存在拓扑关系，比如有顺序依赖，这个是不是得用外部工具?</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-16 10:42:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/55/fe/ab541300.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小猪</span>
  </div>
  <div class="_2_QraFYR_0">我想把job在每一个物理机上定期执行，来删除指定目录下的日志文件，就像DaemonSet那样部署到每一个物理机。这种需求怎么使用job处理？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-03 11:49:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/83/ae/c082bb25.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>大星星</span>
  </div>
  <div class="_2_QraFYR_0">你好，磊哥，我想问下job运行后，会有字段controller-uid。这个东西和node节点有关系么，还是它只是用来标识job。job应该是调度器随便调度一个节点，执行job吧，谢谢。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-20 17:06:02</div>
  </div>
</div>
</div>
</li>
</ul>