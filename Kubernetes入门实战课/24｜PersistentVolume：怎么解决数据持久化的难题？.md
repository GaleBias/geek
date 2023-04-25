<audio title="24｜PersistentVolume：怎么解决数据持久化的难题？" src="https://static001.geekbang.org/resource/audio/bd/32/bd5a63b4b278091624af904f97f70832.mp3" controls="controls"></audio> 
<p>你好，我是Chrono。</p><p>经过了“初级篇”和“中级篇”的学习，相信你对Kubernetes的认识已经比较全面了，那么在接下来的“高级篇”里，我们再进一步，探索Kubernetes更深层次的知识点和更高级的应用技巧。</p><p>今天就先从PersistentVolume讲起。</p><p>早在<a href="https://time.geekbang.org/column/article/533395">第14讲</a>介绍ConfigMap/Secret的时候，我们就遇到过Kubernetes里的Volume存储卷的概念，它使用字段 <code>volumes</code> 和 <code>volumeMounts</code>，相当于是给Pod挂载了一个“虚拟盘”，把配置信息以文件的形式注入进Pod供进程使用。</p><p>不过，那个时候的Volume只能存放较少的数据，离真正的“虚拟盘”还差得很远。</p><p>今天我们就一起来了解Volume的高级用法，看看Kubernetes管理存储资源的API对象PersistentVolume、PersistentVolumeClaim、StorageClass，然后使用本地磁盘来创建实际可用的存储卷。</p><h2>什么是PersistentVolume</h2><p>在刚完成的“中级篇”实战中（<a href="https://time.geekbang.org/column/article/539420">22讲</a>），我们在Kubernetes集群里搭建了WordPress网站，但其中存在一个很严重的问题：Pod没有持久化功能，导致MariaDB无法“永久”存储数据。</p><!-- [[[read_end]]] --><p>因为Pod里的容器是由镜像产生的，而镜像文件本身是只读的，进程要读写磁盘只能用一个临时的存储空间，一旦Pod销毁，临时存储也就会立即回收释放，数据也就丢失了。</p><p>为了保证即使Pod销毁后重建数据依然存在，我们就需要找出一个解决方案，让Pod用上真正的“虚拟盘”。怎么办呢？</p><p>其实，Kubernetes的Volume对数据存储已经给出了一个很好的抽象，它只是定义了有这么一个“存储卷”，而这个“存储卷”是什么类型、有多大容量、怎么存储，我们都可以自由发挥。Pod不需要关心那些专业、复杂的细节，只要设置好 <code>volumeMounts</code>，就可以把Volume加载进容器里使用。</p><p>所以，Kubernetes就顺着Volume的概念，延伸出了<strong>PersistentVolume</strong>对象，它专门用来表示持久存储设备，但隐藏了存储的底层实现，我们只需要知道它能安全可靠地保管数据就可以了（由于PersistentVolume这个词很长，一般都把它简称为PV）。</p><p>那么，集群里的PV都从哪里来呢？</p><p><strong>作为存储的抽象，PV实际上就是一些存储设备、文件系统</strong>，比如Ceph、GlusterFS、NFS，甚至是本地磁盘，管理它们已经超出了Kubernetes的能力范围，所以，一般会由系统管理员单独维护，然后再在Kubernetes里创建对应的PV。</p><p>要注意的是，PV属于集群的系统资源，是和Node平级的一种对象，Pod对它没有管理权，只有使用权。</p><h2>什么是PersistentVolumeClaim/StorageClass</h2><p>现在有了PV，我们是不是可以直接在Pod里挂载使用了呢？</p><p>还不行。因为不同存储设备的差异实在是太大了：有的速度快，有的速度慢；有的可以共享读写，有的只能独占读写；有的容量小，只有几百MB，有的容量大到TB、PB级别……</p><p>这么多种存储设备，只用一个PV对象来管理还是有点太勉强了，不符合“单一职责”的原则，让Pod直接去选择PV也很不灵活。于是Kubernetes就又增加了两个新对象，<strong>PersistentVolumeClaim</strong>和<strong>StorageClass</strong>，用的还是“中间层”的思想，把存储卷的分配管理过程再次细化。</p><p>我们看这两个新对象。</p><p>PersistentVolumeClaim，简称PVC，从名字上看比较好理解，就是用来向Kubernetes申请存储资源的。PVC是给Pod使用的对象，它相当于是Pod的代理，代表Pod向系统申请PV。一旦资源申请成功，Kubernetes就会把PV和PVC关联在一起，这个动作叫做“<strong>绑定</strong>”（bind）。</p><p>但是，系统里的存储资源非常多，如果要PVC去直接遍历查找合适的PV也很麻烦，所以就要用到StorageClass。</p><p>StorageClass的作用有点像<a href="https://time.geekbang.org/column/article/538760">第21讲</a>里的IngressClass，它抽象了特定类型的存储系统（比如Ceph、NFS），在PVC和PV之间充当“协调人”的角色，帮助PVC找到合适的PV。也就是说它可以简化Pod挂载“虚拟盘”的过程，让Pod看不到PV的实现细节。</p><p><img src="https://static001.geekbang.org/resource/image/5e/22/5e21d007a6152ec9594919300c2b6e22.jpg?wh=1920x1053" alt="图片"></p><p>如果看到这里，你觉得还是差点理解也不要着急，我们找个生活中的例子来类比一下。毕竟和常用的CPU、内存比起来，我们对存储系统的认识还是比较少的，所以Kubernetes里，PV、PVC和StorageClass这三个新概念也不是特别好掌握。</p><p>看例子，假设你在公司里想要10张纸打印资料，于是你给前台打电话讲清楚了需求。</p><ul>
<li>“打电话”这个动作，就相当于PVC，向Kubernetes申请存储资源。</li>
<li>前台里有各种牌子的办公用纸，大小、规格也不一样，这就相当于StorageClass。</li>
<li>前台根据你的需要，挑选了一个品牌，再从库存里拿出一包A4纸，可能不止10张，但也能够满足要求，就在登记表上新添了一条记录，写上你在某天申领了办公用品。这个过程就是PVC到PV的绑定。</li>
<li>而最后到你手里的A4纸包，就是PV存储对象。</li>
</ul><p>好，大概了解了这些API对象，我们接下来可以结合YAML描述和实际操作再慢慢体会。</p><h2>如何使用YAML描述PersistentVolume</h2><p>Kubernetes里有很多种类型的PV，我们先看看最容易的本机存储“<strong>HostPath</strong>”，它和Docker里挂载本地目录的 <code>-v</code> 参数非常类似，可以用它来初步认识一下PV的用法。</p><p>因为Pod会在集群的任意节点上运行，所以首先，我们要作为系统管理员在每个节点上创建一个目录，它将会作为本地存储卷挂载到Pod里。</p><p>为了省事，我就在 <code>/tmp</code> 里建立名字是 <code>host-10m-pv</code> 的目录，表示一个只有10MB容量的存储设备。</p><p>有了存储，我们就可以使用YAML来描述这个PV对象了。</p><p>不过很遗憾，你不能用 <code>kubectl create</code> 直接创建PV对象，<strong>只能用 <code>kubectl api-resources</code>、<code>kubectl explain</code> 查看PV的字段说明，手动编写PV 的YAML描述文件</strong>。</p><p>下面我给出一个YAML示例，你可以把它作为样板，编辑出自己的PV：</p><pre><code class="language-yaml">apiVersion: v1
kind: PersistentVolume
metadata:
&nbsp; name: host-10m-pv

spec:
&nbsp; storageClassName: host-test
&nbsp; accessModes:
&nbsp; - ReadWriteOnce
&nbsp; capacity:
&nbsp; &nbsp; storage: 10Mi
&nbsp; hostPath:
&nbsp; &nbsp; path: /tmp/host-10m-pv/
</code></pre><p>PV对象的文件头部分很简单，还是API对象的“老一套”，我就不再详细解释了，重点看它的 <code>spec</code> 部分，每个字段都很重要，描述了存储的详细信息。</p><p>“<strong>storageClassName</strong>”就是刚才说过的，对存储类型的抽象StorageClass。这个PV是我们手动管理的，名字可以任意起，这里我写的是 <code>host-test</code>，你也可以把它改成 <code>manual</code>、<code>hand-work</code> 之类的词汇。</p><p>“<strong>accessModes</strong>”定义了存储设备的访问模式，简单来说就是虚拟盘的读写权限，和Linux的文件访问模式差不多，目前Kubernetes里有3种：</p><ul>
<li>ReadWriteOnce：存储卷可读可写，但只能被一个节点上的Pod挂载。</li>
<li>ReadOnlyMany：存储卷只读不可写，可以被任意节点上的Pod多次挂载。</li>
<li>ReadWriteMany：存储卷可读可写，也可以被任意节点上的Pod多次挂载。</li>
</ul><p>你要注意，这3种访问模式限制的对象是节点而不是Pod，因为存储是系统级别的概念，不属于Pod里的进程。</p><p>显然，本地目录只能是在本机使用，所以这个PV使用了 <code>ReadWriteOnce</code>。</p><p>第三个字段“<strong>capacity</strong>”就很好理解了，表示存储设备的容量，这里我设置为10MB。</p><p>再次提醒你注意，Kubernetes里定义存储容量使用的是国际标准，我们日常习惯使用的KB/MB/GB的基数是1024，要写成Ki/Mi/Gi，一定要小心不要写错了，否则单位不一致实际容量就会对不上。</p><p>最后一个字段“<strong>hostPath</strong>”最简单，它指定了存储卷的本地路径，也就是我们在节点上创建的目录。</p><p>用这些字段把PV的类型、访问模式、容量、存储位置都描述清楚，一个存储设备就创建好了。</p><h2>如何使用YAML描述PersistentVolumeClaim</h2><p>有了PV，就表示集群里有了这么一个持久化存储可以供Pod使用，我们需要再定义PVC对象，向Kubernetes申请存储。</p><p>下面这份YAML就是一个PVC，要求使用一个5MB的存储设备，访问模式是 <code>ReadWriteOnce</code>：</p><pre><code class="language-yaml">apiVersion: v1
kind: PersistentVolumeClaim
metadata:
&nbsp; name: host-5m-pvc

spec:
&nbsp; storageClassName: host-test
&nbsp; accessModes:
&nbsp; &nbsp; - ReadWriteOnce
&nbsp; resources:
&nbsp; &nbsp; requests:
&nbsp; &nbsp; &nbsp; storage: 5Mi
</code></pre><p>PVC的内容与PV很像，但它不表示实际的存储，而是一个“申请”或者“声明”，spec里的字段描述的是对存储的“期望状态”。</p><p>所以PVC里的 <code>storageClassName</code>、<code>accessModes</code> 和PV是一样的，<strong>但不会有字段 <code>capacity</code>，而是要用 <code>resources.request</code> 表示希望要有多大的容量</strong>。</p><p>这样，Kubernetes就会根据PVC里的描述，去找能够匹配StorageClass和容量的PV，然后把PV和PVC“绑定”在一起，实现存储的分配，和前面打电话要A4纸的过程差不多。</p><h2>如何在Kubernetes里使用PersistentVolume</h2><p>现在我们已经准备好了PV和PVC，就可以让Pod实现持久化存储了。</p><p>首先需要用 <code>kubectl apply</code> 创建PV对象：</p><pre><code class="language-plain">kubectl apply -f host-path-pv.yml
</code></pre><p>然后用 <code>kubectl get</code>  查看它的状态：</p><pre><code class="language-plain">kubectl get pv
</code></pre><p><img src="https://static001.geekbang.org/resource/image/5c/37/5ca80e12c71d162f5707d37bf6009c37.png?wh=1920x150" alt="图片"></p><p>从截图里我们可以看到，这个PV的容量是10MB，访问模式是RWO（ReadWriteOnce），StorageClass是我们自己定义的 <code>host-test</code>，状态显示的是 <code>Available</code>，也就是处于可用状态，可以随时分配给Pod使用。</p><p>接下来我们创建PVC，申请存储资源：</p><pre><code class="language-plain">kubectl apply -f host-path-pvc.yml
kubectl get pvc
</code></pre><p><img src="https://static001.geekbang.org/resource/image/fd/1f/fd6f1cb75f5d349860928594db29a11f.png?wh=1920x367" alt="图片"></p><p>一旦PVC对象创建成功，Kubernetes就会立即通过StorageClass、resources等条件在集群里查找符合要求的PV，如果找到合适的存储对象就会把它俩“绑定”在一起。</p><p>PVC对象申请的是5MB，但现在系统里只有一个10MB的PV，没有更合适的对象，所以Kubernetes也只能把这个PV分配出去，多出的容量就算是“福利”了。</p><p>你会看到这两个对象的状态都是 <code>Bound</code>，也就是说存储申请成功，PVC的实际容量就是PV的容量10MB，而不是最初申请的容量5MB。</p><p>那么，如果我们把PVC的申请容量改大一些会怎么样呢？比如改成100MB：</p><p><img src="https://static001.geekbang.org/resource/image/25/0c/25241a47c63cf629b88590ba1773710c.png?wh=1920x350" alt="图片"></p><p>你会看到PVC会一直处于 <code>Pending</code> 状态，这意味着Kubernetes在系统里没有找到符合要求的存储，无法分配资源，只能等有满足要求的PV才能完成绑定。</p><h2>如何为Pod挂载PersistentVolume</h2><p>PV和PVC绑定好了，有了持久化存储，现在我们就可以为Pod挂载存储卷。用法和<a href="https://time.geekbang.org/column/article/533395">第14讲</a>里差不多，先要在 <code>spec.volumes</code> 定义存储卷，然后在 <code>containers.volumeMounts</code> 挂载进容器。</p><p>不过因为我们用的是PVC，所以<strong>要在 <code>volumes</code> 里用字段 <code>persistentVolumeClaim</code> 指定PVC的名字</strong>。</p><p>下面就是Pod的YAML描述文件，把存储卷挂载到了Nginx容器的 <code>/tmp</code> 目录：</p><pre><code class="language-yaml">apiVersion: v1
kind: Pod
metadata:
&nbsp; name: host-pvc-pod

spec:
&nbsp; volumes:
&nbsp; - name: host-pvc-vol
&nbsp; &nbsp; persistentVolumeClaim:
&nbsp; &nbsp; &nbsp; claimName: host-5m-pvc

&nbsp; containers:
&nbsp; &nbsp; - name: ngx-pvc-pod
&nbsp; &nbsp; &nbsp; image: nginx:alpine
&nbsp; &nbsp; &nbsp; ports:
&nbsp; &nbsp; &nbsp; - containerPort: 80
&nbsp; &nbsp; &nbsp; volumeMounts:
&nbsp; &nbsp; &nbsp; - name: host-pvc-vol
&nbsp; &nbsp; &nbsp; &nbsp; mountPath: /tmp
</code></pre><p>我把Pod和PVC/PV的关系画成了图（省略了字段accessModes），你可以从图里看出它们是如何联系起来的：</p><p><img src="https://static001.geekbang.org/resource/image/a4/d8/a4d709808a0ef729604c884c50748bd8.jpg?wh=1920x1310" alt="图片"></p><p>现在我们创建这个Pod，查看它的状态：</p><pre><code class="language-plain">kubectl apply -f host-path-pod.yml
kubectl get pod -o wide
</code></pre><p><img src="https://static001.geekbang.org/resource/image/d4/9d/d4a2771c2c32597a4e5e2e60823c159d.png?wh=1846x192" alt="图片"></p><p>它被Kubernetes调到了worker节点上，那么PV是否确实挂载成功了呢？让我们用 <code>kubectl exec</code> 进入容器，执行一些命令看看：</p><p><img src="https://static001.geekbang.org/resource/image/c4/24/c42a618688eee98555cda33c5c1d6824.png?wh=1320x364" alt="图片"></p><p>容器的 <code>/tmp</code> 目录里生成了一个 <code>a.txt</code> 的文件，根据PV的定义，它就应该落在worker节点的磁盘上，所以我们就登录worker节点检查一下：</p><p><img src="https://static001.geekbang.org/resource/image/9d/c4/9dc40b80e2e4edb2d9449e2d43b02ac4.png?wh=988x242" alt="图片"></p><p>你会看到确实在worker节点的本地目录有一个 <code>a.txt</code> 的文件，再对一下时间，就可以确认是刚才在Pod里生成的文件。</p><p>因为Pod产生的数据已经通过PV存在了磁盘上，所以如果Pod删除后再重新创建，挂载存储卷时会依然使用这个目录，数据保持不变，也就实现了持久化存储。</p><p>不过还有一点小问题，因为这个PV是HostPath类型，只在本节点存储，如果Pod重建时被调度到了其他节点上，那么即使加载了本地目录，也不会是之前的存储位置，持久化功能也就失效了。</p><p>所以，HostPath类型的PV一般用来做测试，或者是用于DaemonSet这样与节点关系比较密切的应用，我们下节课再讲实现真正任意的数据持久化。</p><h2>小结</h2><p>好了，今天我们一起学习了Kubernetes里应对持久化存储的解决方案，一共有三个API对象，分别是PersistentVolume、PersistentVolumeClaim、StorageClass。它们管理的是集群里的存储资源，简单来说就是磁盘，Pod必须通过它们才能够实现数据持久化。</p><p>再小结一下今天的主要内容：</p><ol>
<li>PersistentVolume简称为PV，是Kubernetes对存储设备的抽象，由系统管理员维护，需要描述清楚存储设备的类型、访问模式、容量等信息。</li>
<li>PersistentVolumeClaim简称为PVC，代表Pod向系统申请存储资源，它声明对存储的要求，Kubernetes会查找最合适的PV然后绑定。</li>
<li>StorageClass抽象特定类型的存储系统，归类分组PV对象，用来简化PV/PVC的绑定过程。</li>
<li>HostPath是最简单的一种PV，数据存储在节点本地，速度快但不能跟随Pod迁移。</li>
</ol><h2>课下作业</h2><p>最后是课下作业时间，给你留两个思考题：</p><ol>
<li>HostPath类型的PV要求节点上必须有相应的目录，如果这个目录不存在（比如忘记创建了）会怎么样呢？</li>
<li>你对使用PV/PVC/StorageClass这三个对象来分配存储的流程有什么看法？它们的抽象是好还是坏？</li>
</ol><p>进阶高手是需要自驱的，在这最后的高级篇，非常期待看到你的思考。我们下节课再见。</p><p><img src="https://static001.geekbang.org/resource/image/25/41/25a1d7b3e9841c886781afb44b351341.jpg?wh=1920x1634" alt="图片"></p>
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
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKotsBr2icbYNYlRSlicGUD1H7lulSTQUAiclsEz9gnG5kCW9qeDwdYtlRMXic3V6sj9UrfKLPJnQojag/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ppd0705</span>
  </div>
  <div class="_2_QraFYR_0">如果目录没有创建， pod 会一直pending中。 我在master节点创建了目录 但 pod 没起来，查了半天才想起来pod 启动在worker节点，需要在worker 节点创建目录</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，没有目录，也就是相当于没有存储空间，pv就无法创建，这个是host-path的一个坑，只有自己趟过才能印象深刻。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-18 14:35:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/0c/8e/8a39ee55.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>文古</span>
  </div>
  <div class="_2_QraFYR_0">老师，我个人感觉比较迷茫：k8s需要学习到什么程度才能上岗？学习云原生的线路可以说一下吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 云原生的范围太广了，可以类比一下Linux，它是一个基础，有了这个基础，你才能继续在它上面做其他的事情，一定要结合自己的实际情况。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-18 09:18:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/2b/8c/a6c1ec31.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>作草分茶</span>
  </div>
  <div class="_2_QraFYR_0">老师，怎么没有storageClass的配置呀</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这种简单的不用定义也可以</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-18 00:09:55</div>
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
  <div class="_2_QraFYR_0">1. 对我来说，有点奇怪，和大家的答案是不太相同的，就是创建了pv和pvc，pod挂载之后，host-10m-pv这个文件夹自动创建了。<br>2. 这些概念相当复杂，除了生活实例理解外，并不理解系统中有什么好处。不过我想来想去，觉得storageclass是需要的.因为pv只进行硬件或者相关服务级别的抽象，pvc则只管请求，在数量比较多的pv的场景中，很容易选择到不想要的pv，比如1m请求了100m的，造成资源浪费，所以stroageclass是有必要存在的，但可以设计为一个选择器.<br><br><br>说说一些测试的操作。<br>1. 建立了一个pv 10m,一个pvc请求，5m。顺利请求到，这是和课程中一样的。<br>2. 建立两个pv,10m,两个pvc请求，都是5m，各自请求到一个，1和2的操作都使用了同同样的storageClassName。<br>3. 建立一个pv 10m，两个pvc请求，其中一个请求不到，显示为没有资源，除非再造一个，就可以被请求到。<br>4. 建了pv和pvc，先删除pv，不删除pvc，pv一直会显示terming状态，但始终不会消失，删除pvc后，删除成功。<br>感觉对这些概念有些理解，但更多的是迷糊、<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这也是为什么把PV&#47;PVC放在高级篇的原因之一，它的确比较复杂，我们现在只要对它有个大概的了解就行了，真正要用好可能还是要与实际的工作去结合来学习，而且很多时候PV&#47;PVC都已经是配置好的状态，除非是系统管理员，一般我们掌握了概念，直接使用就可以。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-06 17:16:10</div>
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
  <div class="_2_QraFYR_0">老师，想请问一下文稿的图是用什么软件画的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">编辑回复: Keynote，思维导图用的Xmind</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-18 13:30:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKotsBr2icbYNYlRSlicGUD1H7lulSTQUAiclsEz9gnG5kCW9qeDwdYtlRMXic3V6sj9UrfKLPJnQojag/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ppd0705</span>
  </div>
  <div class="_2_QraFYR_0">跑个题： 题图是有什么关联吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">编辑回复: 因为持久化 -&gt; 所以挑了一幅持久到墙上的画……哈哈哈哈，主要就是最近的知识点好难找合适的头图，累了，这张凑合一下</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-18 13:07:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/29/87/e1/b85dce85.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>无尽蔚蓝</span>
  </div>
  <div class="_2_QraFYR_0">“Kubernetes 里定义存储容量使用的是国际标准，我们日常习惯使用的 KB&#47;MB&#47;GB 的基数是 1024，要写成 Ki&#47;Mi&#47;Gi，一定要小心不要写错了，否则单位不一致实际容量就会对不上。”<br> Ki&#47;Mi&#47;Gi什么意思？<br><br>chatGPT的回答：<br>Ki&#47;Mi&#47;Gi 是二进制单位前缀，用于表示存储容量。它们的含义如下：<br><br>1 KiB (kibibyte) = 2^10 bytes = 1,024 bytes<br>1 MiB (mebibyte) = 2^20 bytes = 1,048,576 bytes<br>1 GiB (gibibyte) = 2^30 bytes = 1,073,741,824 bytes<br><br>这些单位前缀在计算机领域中用于准确地表示存储容量。与之相对的是基于十进制的单位前缀，如 KB&#47;MB&#47;GB，它们使用的是 10 的幂，而不是 2 的幂。因此，1 KB 实际上是 1,000 bytes，1 MB 是 1,000,000 bytes，1 GB 是 1,000,000,000 bytes。<br><br>在 Kubernetes 中，使用 Ki&#47;Mi&#47;Gi 单位来定义存储容量可以避免使用基于十进制的单位前缀时可能引起的容量计算不准确的问题。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Ki&#47;Mi&#47;Gi 是国际标准定义的单位名称，有点怪，习惯就好了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-17 22:12:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/29/87/e1/b85dce85.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>无尽蔚蓝</span>
  </div>
  <div class="_2_QraFYR_0">列一些常用的storageClassName会更好理解：<br><br>chatGPT的回答：<br>以下是一些常见的 Kubernetes 存储类（StorageClass）的名称：<br><br>standard：标准存储类，用于提供基本的存储能力，适用于一般用途。<br>gp2：亚马逊 Web 服务（AWS）上的一种存储类型，适用于需要快速访问和较高性能的应用程序。<br>rbd：Ceph 存储系统的一种存储类型，用于提供分布式块存储。<br>glusterfs：GlusterFS 分布式文件系统的一种存储类型，用于提供高可用性和可扩展性的存储解决方案。<br>nfs：网络文件系统（NFS）的一种存储类型，用于提供共享文件存储。<br>azure-disk：Azure 上的一种存储类型，适用于需要高可用性和可扩展性的应用程序。<br>local-storage：本地存储的一种存储类型，用于将本地磁盘挂载到 Kubernetes Pod 中。<br>请注意，这些存储类型的可用性可能因为云服务提供商的不同而有所不同。此外，还可以创建自定义存储类型来满足特定的存储需求。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 紧跟科技最前沿，great。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-25 12:43:55</div>
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
  <div class="_2_QraFYR_0">为了省事，我没有在 &#47;tmp 里建立名字是 host-10m-pv 的目录，最后发现他自己创建了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 应该不行吧，没有目录pv不会创建成功。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-14 23:03:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/58/6f/4acc5b43.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>PeiXy</span>
  </div>
  <div class="_2_QraFYR_0">老师我现在的理解是、我默认先创建好sc，然后直接提交pvc，再由pvc去申请pv…在没有适合的pv的情况下会主动去创建一个pv对吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 必须有Provisioner才能自动创建，否则pv只能由管理员手动分配，没有合适的pv就只能等待。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-17 22:53:18</div>
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
  <div class="_2_QraFYR_0">对于作业“HostPath 类型的 PV 要求节点上必须有相应的目录，如果这个目录不存在（比如忘记创建了）会怎么样呢？”，看评论区同学测试的结果有不同<br>我的测试结果是这样：<br>1. 用kubectl delete -f清掉本节中创建的pod、pvc、pv，手动删掉之前在节点机器上手动创建的目录&#47;tmp&#47;host-10m-pv（以及里边的a.txt文件）<br>2. 用本节这些yaml文件，再次按顺序创建pv和pvc，都能创建成功，截止此时，在节点机器上查看，还是没有目录&#47;tmp&#47;host-10m-pv的<br>3. 创建pod，也是创建成功，此时，在节点机器上，就自动创建了目录&#47;tmp&#47;host-10m-pv<br>[abc@k8s3 ~]$ ls -ld &#47;tmp&#47;host*<br>drwxr-xr-x. 2 root root 6 Oct 18 18:31 &#47;tmp&#47;host-10m-pv<br>4. 进入pod，在存储卷挂载路径&#47;tmp下创建文件a.txt，该文件也会出现在节点机器的目录&#47;tmp&#47;host-10m-pv下</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个和我试验的结果不同，我的是如果没有本地目录pv就无法创建成功，这个行为有点令人困惑。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-18 19:05:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/qCv5IcP1lkO2jicrTic9KicycZXZ7WylG49GZHJCibuFQfBlJMsCpVHARuaLxIB23f3enRL4ls6EOr9wxu40K0Hl8Q/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>tcyi</span>
  </div>
  <div class="_2_QraFYR_0">为什么我执行完老师提供的host-path-pv.yml后&#47;tmp&#47;host-10m-pv&#47;这个目录自动创建了呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个应该是不会的，host path只能由管理员自己创建，可以清除目录再仔细观察一下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-01 12:38:07</div>
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
  <div class="_2_QraFYR_0">这 3 种访问模式限制的对象是节点而不是 Pod，因为存储是系统级别的概念，不属于 Pod 里的进程  --- 这句话怎么理解，文章里对这三种访问模式的解释，就是针对pod</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 存储是在节点上分配的，只是给Pod使用，Pod不能管理存储。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-02 10:59:08</div>
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
  <div class="_2_QraFYR_0">如果我删除后，磁盘容量回收是如何操作的呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个不是kubernetes关心的问题，由PV后面的存储系统来管理。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-27 20:08:51</div>
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
  <div class="_2_QraFYR_0">版本：1.26.3<br>HostPath不用手动创建目录，yml定义了会自动创建目录，亲测是这样的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-27 19:54:07</div>
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
  <div class="_2_QraFYR_0">Ki&#47;Mi&#47;Gi不是IEC标准吗，国际标准是kB MB GB啊 而且是以10进制计算，只不过像Windows系统的kB MB GB还是以什么1024来规定，而不是1000来定义，历史遗留问题。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-27 17:13:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/3c/4d/3dec4bfe.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>蔡晓慧</span>
  </div>
  <div class="_2_QraFYR_0">想复现老师的PVC 会一直处于 Pending 状态，需先kubectl delete -f host-path-pvc.yml 然后再apply<br>要不然会报错误only dynamically provisioned pvc can be resized and the storageclass that provisions the pvc must support resize<br><br>应该是hostpath这种的不支持动态扩容吧？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-15 15:39:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/dbqU4jVNf6mmicdZ18eXgHOrd0icRfsarXAV8c5ZqOHopJ4CZ8QtuXESeLL5erHEHCibwP5Udz0RicaspGb4MNSb8g/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ecolife_zhi</span>
  </div>
  <div class="_2_QraFYR_0">试验了一下，host path会自动创建，不需要手动在worker节点创建</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 也许是kubernetes版本不同？我试验的时候是必须要有的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-08 10:55:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/56/2f/4518f8e1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>放不下荣华富贵</span>
  </div>
  <div class="_2_QraFYR_0">我这边依然用minikube的话，a.txt并没有创建到宿主机上，find &#47;tmp 最近创建的txt文件也没有。但是delete pod再启动，a.txt依然存在</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 应该是minikube的虚拟机节点，要用minikube ssh登录进去看。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-29 09:17:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/65/da/29fe3dde.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小宝</span>
  </div>
  <div class="_2_QraFYR_0">例子中的：<br>spec: storageClassName: host-test<br>这个StorageClass哪里来的，貌似例子中没定义。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个StorageClass是自己定义的，名字可以任意起。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-23 11:14:12</div>
  </div>
</div>
</div>
</li>
</ul>