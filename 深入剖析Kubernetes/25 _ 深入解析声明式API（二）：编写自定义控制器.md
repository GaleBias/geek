<audio title="25 _ 深入解析声明式API（二）：编写自定义控制器" src="https://static001.geekbang.org/resource/audio/d6/17/d67edba5c9541e954c18dfbaadbbfc17.mp3" controls="controls"></audio> 
<p>你好，我是张磊。今天我和你分享的主题是：深入解析声明式API之编写自定义控制器。</p><p>在上一篇文章中，我和你详细分享了Kubernetes中声明式API的实现原理，并且通过一个添加Network对象的实例，为你讲述了在Kubernetes里添加API资源的过程。</p><p>在今天的这篇文章中，我就继续和你一起完成剩下一半的工作，即：为Network这个自定义API对象编写一个自定义控制器（Custom Controller）。</p><p>正如我在上一篇文章结尾处提到的，“声明式API”并不像“命令式API”那样有着明显的执行逻辑。这就使得<strong>基于声明式API的业务功能实现，往往需要通过控制器模式来“监视”API对象的变化（比如，创建或者删除Network），然后以此来决定实际要执行的具体工作。</strong></p><p>接下来，我就和你一起通过编写代码来实现这个过程。这个项目和上一篇文章里的代码是同一个项目，你可以从<a href="https://github.com/resouer/k8s-controller-custom-resource">这个GitHub库</a>里找到它们。我在代码里还加上了丰富的注释，你可以随时参考。</p><p>总得来说，编写自定义控制器代码的过程包括：编写main函数、编写自定义控制器的定义，以及编写控制器里的业务逻辑三个部分。</p><p><span class="orange">首先，我们来编写这个自定义控制器的main函数。</span></p><!-- [[[read_end]]] --><p>main函数的主要工作就是，定义并初始化一个自定义控制器（Custom Controller），然后启动它。这部分代码的主要内容如下所示：</p><pre><code>func main() {
  ...
  
  cfg, err := clientcmd.BuildConfigFromFlags(masterURL, kubeconfig)
  ...
  kubeClient, err := kubernetes.NewForConfig(cfg)
  ...
  networkClient, err := clientset.NewForConfig(cfg)
  ...
  
  networkInformerFactory := informers.NewSharedInformerFactory(networkClient, ...)
  
  controller := NewController(kubeClient, networkClient,
  networkInformerFactory.Samplecrd().V1().Networks())
  
  go networkInformerFactory.Start(stopCh)
 
  if err = controller.Run(2, stopCh); err != nil {
    glog.Fatalf(&quot;Error running controller: %s&quot;, err.Error())
  }
}
</code></pre><p>可以看到，这个main函数主要通过三步完成了初始化并启动一个自定义控制器的工作。</p><p><strong>第一步</strong>：main函数根据我提供的Master配置（APIServer的地址端口和kubeconfig的路径），创建一个Kubernetes的client（kubeClient）和Network对象的client（networkClient）。</p><p>但是，如果我没有提供Master配置呢？</p><p>这时，main函数会直接使用一种名叫<strong>InClusterConfig</strong>的方式来创建这个client。这个方式，会假设你的自定义控制器是以Pod的方式运行在Kubernetes集群里的。</p><p>而我在第15篇文章<a href="https://time.geekbang.org/column/article/40466">《深入解析Pod对象（二）：使用进阶》</a>中曾经提到过，Kubernetes 里所有的Pod都会以Volume的方式自动挂载Kubernetes的默认ServiceAccount。所以，这个控制器就会直接使用默认ServiceAccount数据卷里的授权信息，来访问APIServer。</p><p><strong>第二步</strong>：main函数为Network对象创建一个叫作InformerFactory（即：networkInformerFactory）的工厂，并使用它生成一个Network对象的Informer，传递给控制器。</p><p><strong>第三步</strong>：main函数启动上述的Informer，然后执行controller.Run，启动自定义控制器。</p><p>至此，main函数就结束了。</p><p>看到这，你可能会感到非常困惑：编写自定义控制器的过程难道就这么简单吗？这个Informer又是个什么东西呢？</p><p>别着急。</p><p>接下来，我就为你<strong>详细解释一下这个自定义控制器的工作原理。</strong></p><p>在Kubernetes项目中，一个自定义控制器的工作原理，可以用下面这样一幅流程图来表示（在后面的叙述中，我会用“示意图”来指代它）：</p><p><img src="https://static001.geekbang.org/resource/image/32/c3/32e545dcd4664a3f36e95af83b571ec3.png?wh=1846*822" alt=""></p><center><span class="reference">图1 自定义控制器的工作流程示意图</span></center><p>我们先从这幅示意图的最左边看起。</p><p><strong>这个控制器要做的第一件事，是从Kubernetes的APIServer里获取它所关心的对象，也就是我定义的Network对象</strong>。</p><p>这个操作，依靠的是一个叫作Informer（可以翻译为：通知器）的代码库完成的。Informer与API对象是一一对应的，所以我传递给自定义控制器的，正是一个Network对象的Informer（Network Informer）。</p><p>不知你是否已经注意到，我在创建这个Informer工厂的时候，需要给它传递一个networkClient。</p><p>事实上，Network Informer正是使用这个networkClient，跟APIServer建立了连接。不过，真正负责维护这个连接的，则是Informer所使用的Reflector包。</p><p>更具体地说，Reflector使用的是一种叫作<strong>ListAndWatch</strong>的方法，来“获取”并“监听”这些Network对象实例的变化。</p><p>在ListAndWatch机制下，一旦APIServer端有新的Network实例被创建、删除或者更新，Reflector都会收到“事件通知”。这时，该事件及它对应的API对象这个组合，就被称为增量（Delta），它会被放进一个Delta FIFO Queue（即：增量先进先出队列）中。</p><p>而另一方面，Informe会不断地从这个Delta FIFO Queue里读取（Pop）增量。每拿到一个增量，Informer就会判断这个增量里的事件类型，然后创建或者更新本地对象的缓存。这个缓存，在Kubernetes里一般被叫作Store。</p><p>比如，如果事件类型是Added（添加对象），那么Informer就会通过一个叫作Indexer的库把这个增量里的API对象保存在本地缓存中，并为它创建索引。相反，如果增量的事件类型是Deleted（删除对象），那么Informer就会从本地缓存中删除这个对象。</p><p>这个<strong>同步本地缓存的工作，是Informer的第一个职责，也是它最重要的职责。</strong></p><p>而<strong>Informer的第二个职责，则是根据这些事件的类型，触发事先注册好的ResourceEventHandler</strong>。这些Handler，需要在创建控制器的时候注册给它对应的Informer。</p><p>接下来，我们就来<span class="orange">编写这个控制器的定义</span>，它的主要内容如下所示：</p><pre><code>func NewController(
  kubeclientset kubernetes.Interface,
  networkclientset clientset.Interface,
  networkInformer informers.NetworkInformer) *Controller {
  ...
  controller := &amp;Controller{
    kubeclientset:    kubeclientset,
    networkclientset: networkclientset,
    networksLister:   networkInformer.Lister(),
    networksSynced:   networkInformer.Informer().HasSynced,
    workqueue:        workqueue.NewNamedRateLimitingQueue(...,  &quot;Networks&quot;),
    ...
  }
    networkInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
    AddFunc: controller.enqueueNetwork,
    UpdateFunc: func(old, new interface{}) {
      oldNetwork := old.(*samplecrdv1.Network)
      newNetwork := new.(*samplecrdv1.Network)
      if oldNetwork.ResourceVersion == newNetwork.ResourceVersion {
        return
      }
      controller.enqueueNetwork(new)
    },
    DeleteFunc: controller.enqueueNetworkForDelete,
 return controller
}
</code></pre><p><strong>我前面在main函数里创建了两个client（kubeclientset和networkclientset），然后在这段代码里，使用这两个client和前面创建的Informer，初始化了自定义控制器。</strong></p><p>值得注意的是，在这个自定义控制器里，我还设置了一个工作队列（work queue），它正是处于示意图中间位置的WorkQueue。这个工作队列的作用是，负责同步Informer和控制循环之间的数据。</p><blockquote>
<p>实际上，Kubernetes项目为我们提供了很多个工作队列的实现，你可以根据需要选择合适的库直接使用。</p>
</blockquote><p><strong>然后，我为networkInformer注册了三个Handler（AddFunc、UpdateFunc和DeleteFunc），分别对应API对象的“添加”“更新”和“删除”事件。而具体的处理操作，都是将该事件对应的API对象加入到工作队列中。</strong></p><p>需要注意的是，实际入队的并不是API对象本身，而是它们的Key，即：该API对象的<code>&lt;namespace&gt;/&lt;name&gt;</code>。</p><p>而我们后面即将编写的控制循环，则会不断地从这个工作队列里拿到这些Key，然后开始执行真正的控制逻辑。</p><p>综合上面的讲述，你现在应该就能明白，<strong>所谓Informer，其实就是一个带有本地缓存和索引机制的、可以注册EventHandler的client</strong>。它是自定义控制器跟APIServer进行数据同步的重要组件。</p><p>更具体地说，Informer通过一种叫作ListAndWatch的方法，把APIServer中的API对象缓存在了本地，并负责更新和维护这个缓存。</p><p>其中，ListAndWatch方法的含义是：首先，通过APIServer的LIST API“获取”所有最新版本的API对象；然后，再通过WATCH API来“监听”所有这些API对象的变化。</p><p>而通过监听到的事件变化，Informer就可以实时地更新本地缓存，并且调用这些事件对应的EventHandler了。</p><p>此外，在这个过程中，每经过resyncPeriod指定的时间，Informer维护的本地缓存，都会使用最近一次LIST返回的结果强制更新一次，从而保证缓存的有效性。在Kubernetes中，这个缓存强制更新的操作就叫作：resync。</p><p>需要注意的是，这个定时resync操作，也会触发Informer注册的“更新”事件。但此时，这个“更新”事件对应的Network对象实际上并没有发生变化，即：新、旧两个Network对象的ResourceVersion是一样的。在这种情况下，Informer就不需要对这个更新事件再做进一步的处理了。</p><p>这也是为什么我在上面的UpdateFunc方法里，先判断了一下新、旧两个Network对象的版本（ResourceVersion）是否发生了变化，然后才开始进行的入队操作。</p><p>以上，就是Kubernetes中的Informer库的工作原理了。</p><p>接下来，我们就来到了示意图中最后面的控制循环（Control Loop）部分，也正是我在main函数最后调用controller.Run()启动的“控制循环”。它的主要内容如下所示：</p><pre><code>func (c *Controller) Run(threadiness int, stopCh &lt;-chan struct{}) error {
 ...
  if ok := cache.WaitForCacheSync(stopCh, c.networksSynced); !ok {
    return fmt.Errorf(&quot;failed to wait for caches to sync&quot;)
  }
  
  ...
  for i := 0; i &lt; threadiness; i++ {
    go wait.Until(c.runWorker, time.Second, stopCh)
  }
  
  ...
  return nil
}
</code></pre><p>可以看到，启动控制循环的逻辑非常简单：</p><ul>
<li>首先，等待Informer完成一次本地缓存的数据同步操作；</li>
<li>然后，直接通过goroutine启动一个（或者并发启动多个）“无限循环”的任务。</li>
</ul><p>而这个“无限循环”任务的每一个循环周期，执行的正是我们真正关心的业务逻辑。</p><p>所以接下来，我们就来<span class="orange">编写这个自定义控制器的业务逻辑</span>，它的主要内容如下所示：</p><pre><code>func (c *Controller) runWorker() {
  for c.processNextWorkItem() {
  }
}

func (c *Controller) processNextWorkItem() bool {
  obj, shutdown := c.workqueue.Get()
  
  ...
  
  err := func(obj interface{}) error {
    ...
    if err := c.syncHandler(key); err != nil {
     return fmt.Errorf(&quot;error syncing '%s': %s&quot;, key, err.Error())
    }
    
    c.workqueue.Forget(obj)
    ...
    return nil
  }(obj)
  
  ...
  
  return true
}

func (c *Controller) syncHandler(key string) error {

  namespace, name, err := cache.SplitMetaNamespaceKey(key)
  ...
  
  network, err := c.networksLister.Networks(namespace).Get(name)
  if err != nil {
    if errors.IsNotFound(err) {
      glog.Warningf(&quot;Network does not exist in local cache: %s/%s, will delete it from Neutron ...&quot;,
      namespace, name)
      
      glog.Warningf(&quot;Network: %s/%s does not exist in local cache, will delete it from Neutron ...&quot;,
    namespace, name)
    
     // FIX ME: call Neutron API to delete this network by name.
     //
     // neutron.Delete(namespace, name)
     
     return nil
  }
    ...
    
    return err
  }
  
  glog.Infof(&quot;[Neutron] Try to process network: %#v ...&quot;, network)
  
  // FIX ME: Do diff().
  //
  // actualNetwork, exists := neutron.Get(namespace, name)
  //
  // if !exists {
  //   neutron.Create(namespace, name)
  // } else if !reflect.DeepEqual(actualNetwork, network) {
  //   neutron.Update(namespace, name)
  // }
  
  return nil
}
</code></pre><p>可以看到，在这个执行周期里（processNextWorkItem），我们<strong>首先</strong>从工作队列里出队（workqueue.Get）了一个成员，也就是一个Key（Network对象的：namespace/name）。</p><p><strong>然后</strong>，在syncHandler方法中，我使用这个Key，尝试从Informer维护的缓存中拿到了它所对应的Network对象。</p><p>可以看到，在这里，我使用了networksLister来尝试获取这个Key对应的Network对象。这个操作，其实就是在访问本地缓存的索引。实际上，在Kubernetes的源码中，你会经常看到控制器从各种Lister里获取对象，比如：podLister、nodeLister等等，它们使用的都是Informer和缓存机制。</p><p>而如果控制循环从缓存中拿不到这个对象（即：networkLister返回了IsNotFound错误），那就意味着这个Network对象的Key是通过前面的“删除”事件添加进工作队列的。所以，尽管队列里有这个Key，但是对应的Network对象已经被删除了。</p><p>这时候，我就需要调用Neutron的API，把这个Key对应的Neutron网络从真实的集群里删除掉。</p><p><strong>而如果能够获取到对应的Network对象，我就可以执行控制器模式里的对比“期望状态”和“实际状态”的逻辑了。</strong></p><p>其中，自定义控制器“千辛万苦”拿到的这个Network对象，<strong>正是APIServer里保存的“期望状态”</strong>，即：用户通过YAML文件提交到APIServer里的信息。当然，在我们的例子里，它已经被Informer缓存在了本地。</p><p><strong>那么，“实际状态”又从哪里来呢？</strong></p><p>当然是来自于实际的集群了。</p><p>所以，我们的控制循环需要通过Neutron API来查询实际的网络情况。</p><p>比如，我可以先通过Neutron来查询这个Network对象对应的真实网络是否存在。</p><ul>
<li>如果不存在，这就是一个典型的“期望状态”与“实际状态”不一致的情形。这时，我就需要使用这个Network对象里的信息（比如：CIDR和Gateway），调用Neutron API来创建真实的网络。</li>
<li>如果存在，那么，我就要读取这个真实网络的信息，判断它是否跟Network对象里的信息一致，从而决定我是否要通过Neutron来更新这个已经存在的真实网络。</li>
</ul><p>这样，我就通过对比“期望状态”和“实际状态”的差异，完成了一次调协（Reconcile）的过程。</p><p>至此，一个完整的自定义API对象和它所对应的自定义控制器，就编写完毕了。</p><blockquote>
<p>备注：与Neutron相关的业务代码并不是本篇文章的重点，所以我仅仅通过注释里的伪代码为你表述了这部分内容。如果你对这些代码感兴趣的话，可以自行完成。最简单的情况，你可以自己编写一个Neutron Mock，然后输出对应的操作日志。</p>
</blockquote><p>接下来，<span class="orange">我们就一起来把这个项目运行起来，查看一下它的工作情况。</span></p><p>你可以自己编译这个项目，也可以直接使用我编译好的二进制文件（samplecrd-controller）。编译并启动这个项目的具体流程如下所示：</p><pre><code># Clone repo
$ git clone https://github.com/resouer/k8s-controller-custom-resource$ cd k8s-controller-custom-resource

### Skip this part if you don't want to build
# Install dependency
$ go get github.com/tools/godep
$ godep restore
# Build
$ go build -o samplecrd-controller .

$ ./samplecrd-controller -kubeconfig=$HOME/.kube/config -alsologtostderr=true
I0915 12:50:29.051349   27159 controller.go:84] Setting up event handlers
I0915 12:50:29.051615   27159 controller.go:113] Starting Network control loop
I0915 12:50:29.051630   27159 controller.go:116] Waiting for informer caches to sync
E0915 12:50:29.066745   27159 reflector.go:134] github.com/resouer/k8s-controller-custom-resource/pkg/client/informers/externalversions/factory.go:117: Failed to list *v1.Network: the server could not find the requested resource (get networks.samplecrd.k8s.io)
...
</code></pre><p>你可以看到，自定义控制器被启动后，一开始会报错。</p><p>这是因为，此时Network对象的CRD还没有被创建出来，所以Informer去APIServer里“获取”（List）Network对象时，并不能找到Network这个API资源类型的定义，即：</p><pre><code>Failed to list *v1.Network: the server could not find the requested resource (get networks.samplecrd.k8s.io)
</code></pre><p>所以，接下来我就需要创建Network对象的CRD，这个操作在上一篇文章里已经介绍过了。</p><p>在另一个shell窗口里执行：</p><pre><code>$ kubectl apply -f crd/network.yaml
</code></pre><p>这时候，你就会看到控制器的日志恢复了正常，控制循环启动成功：</p><pre><code>...
I0915 12:50:29.051630   27159 controller.go:116] Waiting for informer caches to sync
...
I0915 12:52:54.346854   25245 controller.go:121] Starting workers
I0915 12:52:54.346914   25245 controller.go:127] Started workers
</code></pre><p>接下来，我就可以进行Network对象的增删改查操作了。</p><p>首先，创建一个Network对象：</p><pre><code>$ cat example/example-network.yaml 
apiVersion: samplecrd.k8s.io/v1
kind: Network
metadata:
  name: example-network
spec:
  cidr: &quot;192.168.0.0/16&quot;
  gateway: &quot;192.168.0.1&quot;
  
$ kubectl apply -f example/example-network.yaml 
network.samplecrd.k8s.io/example-network created
</code></pre><p>这时候，查看一下控制器的输出：</p><pre><code>...
I0915 12:50:29.051349   27159 controller.go:84] Setting up event handlers
I0915 12:50:29.051615   27159 controller.go:113] Starting Network control loop
I0915 12:50:29.051630   27159 controller.go:116] Waiting for informer caches to sync
...
I0915 12:52:54.346854   25245 controller.go:121] Starting workers
I0915 12:52:54.346914   25245 controller.go:127] Started workers
I0915 12:53:18.064409   25245 controller.go:229] [Neutron] Try to process network: &amp;v1.Network{TypeMeta:v1.TypeMeta{Kind:&quot;&quot;, APIVersion:&quot;&quot;}, ObjectMeta:v1.ObjectMeta{Name:&quot;example-network&quot;, GenerateName:&quot;&quot;, Namespace:&quot;default&quot;, ... ResourceVersion:&quot;479015&quot;, ... Spec:v1.NetworkSpec{Cidr:&quot;192.168.0.0/16&quot;, Gateway:&quot;192.168.0.1&quot;}} ...
I0915 12:53:18.064650   25245 controller.go:183] Successfully synced 'default/example-network'
...
</code></pre><p>可以看到，我们上面创建example-network的操作，触发了EventHandler的“添加”事件，从而被放进了工作队列。</p><p>紧接着，控制循环就从队列里拿到了这个对象，并且打印出了正在“处理”这个Network对象的日志。</p><p>可以看到，这个Network的ResourceVersion，也就是API对象的版本号，是479015，而它的Spec字段的内容，跟我提交的YAML文件一摸一样，比如，它的CIDR网段是：192.168.0.0/16。</p><p>这时候，我来修改一下这个YAML文件的内容，如下所示：</p><pre><code>$ cat example/example-network.yaml 
apiVersion: samplecrd.k8s.io/v1
kind: Network
metadata:
  name: example-network
spec:
  cidr: &quot;192.168.1.0/16&quot;
  gateway: &quot;192.168.1.1&quot;
</code></pre><p>可以看到，我把这个YAML文件里的CIDR和Gateway字段修改成了192.168.1.0/16网段。</p><p>然后，我们执行了kubectl apply命令来提交这次更新，如下所示：</p><pre><code>$ kubectl apply -f example/example-network.yaml 
network.samplecrd.k8s.io/example-network configured
</code></pre><p>这时候，我们就可以观察一下控制器的输出：</p><pre><code>...
I0915 12:53:51.126029   25245 controller.go:229] [Neutron] Try to process network: &amp;v1.Network{TypeMeta:v1.TypeMeta{Kind:&quot;&quot;, APIVersion:&quot;&quot;}, ObjectMeta:v1.ObjectMeta{Name:&quot;example-network&quot;, GenerateName:&quot;&quot;, Namespace:&quot;default&quot;, ...  ResourceVersion:&quot;479062&quot;, ... Spec:v1.NetworkSpec{Cidr:&quot;192.168.1.0/16&quot;, Gateway:&quot;192.168.1.1&quot;}} ...
I0915 12:53:51.126348   25245 controller.go:183] Successfully synced 'default/example-network'
</code></pre><p>可以看到，这一次，Informer注册的“更新”事件被触发，更新后的Network对象的Key被添加到了工作队列之中。</p><p>所以，接下来控制循环从工作队列里拿到的Network对象，与前一个对象是不同的：它的ResourceVersion的值变成了479062；而Spec里的字段，则变成了192.168.1.0/16网段。</p><p>最后，我再把这个对象删除掉：</p><pre><code>$ kubectl delete -f example/example-network.yaml
</code></pre><p>这一次，在控制器的输出里，我们就可以看到，Informer注册的“删除”事件被触发，并且控制循环“调用”Neutron API“删除”了真实环境里的网络。这个输出如下所示：</p><pre><code>W0915 12:54:09.738464   25245 controller.go:212] Network: default/example-network does not exist in local cache, will delete it from Neutron ...
I0915 12:54:09.738832   25245 controller.go:215] [Neutron] Deleting network: default/example-network ...
I0915 12:54:09.738854   25245 controller.go:183] Successfully synced 'default/example-network'
</code></pre><p>以上，就是编写和使用自定义控制器的全部流程了。</p><p>实际上，<span class="orange">这套流程不仅可以用在自定义API资源上，也完全可以用在Kubernetes原生的默认API对象上。</span></p><p>比如，我们在main函数里，除了创建一个Network Informer外，还可以初始化一个Kubernetes默认API对象的Informer工厂，比如Deployment对象的Informer。这个具体做法如下所示：</p><pre><code>func main() {
  ...
  
  kubeInformerFactory := kubeinformers.NewSharedInformerFactory(kubeClient, time.Second*30)
  
  controller := NewController(kubeClient, exampleClient,
  kubeInformerFactory.Apps().V1().Deployments(),
  networkInformerFactory.Samplecrd().V1().Networks())
  
  go kubeInformerFactory.Start(stopCh)
  ...
}
</code></pre><p>在这段代码中，我们<strong>首先</strong>使用Kubernetes的client（kubeClient）创建了一个工厂；</p><p><strong>然后</strong>，我用跟Network类似的处理方法，生成了一个Deployment Informer；</p><p><strong>接着</strong>，我把Deployment Informer传递给了自定义控制器；当然，我也要调用Start方法来启动这个Deployment Informer。</p><p>而有了这个Deployment Informer后，这个控制器也就持有了所有Deployment对象的信息。接下来，它既可以通过deploymentInformer.Lister()来获取Etcd里的所有Deployment对象，也可以为这个Deployment Informer注册具体的Handler来。</p><p>更重要的是，<strong>这就使得在这个自定义控制器里面，我可以通过对自定义API对象和默认API对象进行协同，从而实现更加复杂的编排功能</strong>。</p><p>比如：用户每创建一个新的Deployment，这个自定义控制器，就可以为它创建一个对应的Network供它使用。</p><p>这些对Kubernetes API编程范式的更高级应用，我就留给你在实际的场景中去探索和实践了。</p><h2>总结</h2><p>在今天这篇文章中，我为你剖析了Kubernetes API编程范式的具体原理，并编写了一个自定义控制器。</p><p>这其中，有如下几个概念和机制，是你一定要理解清楚的：</p><p>所谓的Informer，就是一个自带缓存和索引机制，可以触发Handler的客户端库。这个本地缓存在Kubernetes中一般被称为Store，索引一般被称为Index。</p><p>Informer使用了Reflector包，它是一个可以通过ListAndWatch机制获取并监视API对象变化的客户端封装。</p><p>Reflector和Informer之间，用到了一个“增量先进先出队列”进行协同。而Informer与你要编写的控制循环之间，则使用了一个工作队列来进行协同。</p><p>在实际应用中，除了控制循环之外的所有代码，实际上都是Kubernetes为你自动生成的，即：pkg/client/{informers, listers, clientset}里的内容。</p><p>而这些自动生成的代码，就为我们提供了一个可靠而高效地获取API对象“期望状态”的编程库。</p><p>所以，接下来，作为开发者，你就只需要关注如何拿到“实际状态”，然后如何拿它去跟“期望状态”做对比，从而决定接下来要做的业务逻辑即可。</p><p>以上内容，就是Kubernetes API编程范式的核心思想。</p><h2>思考题</h2><p>请思考一下，为什么Informer和你编写的控制循环之间，一定要使用一个工作队列来进行协作呢？</p><p>感谢你的收听，欢迎你给我留言，也欢迎分享给更多的朋友一起阅读。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/7c/16/4d1e5cc1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>mgxian</span>
  </div>
  <div class="_2_QraFYR_0">Informer 和控制循环分开是为了解耦，防止控制循环执行过慢把Informer 拖死</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-19 09:50:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/8a/28/45cf7f34.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>142</span>
  </div>
  <div class="_2_QraFYR_0">运维人员看起来越来越费力了😢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-31 19:01:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/2f/c6/b7448158.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Tarjintor</span>
  </div>
  <div class="_2_QraFYR_0">不知道现在还有人看回复没，我用了不少operator，重度使用rook&#47;ceph<br>感觉声明式api还是有一些本质不足的地方，主要是<br>1.通常operator都是用来描述复杂有状态集群的，这个集群本身就已经很复杂了<br>2.声明式api通过diff的方式来得出集群要做怎么样的状态变迁，这个过程中，常常会有很多状况不能覆盖到，而如果使用者对此没有深刻的认识，就会越用越糊涂<br>大致来说，如果一个集群本身的状态有n种，那么operator理论上就要覆盖n*(n-1)种变迁的可能性，而这个体力活几乎是不可能完成的，所有很多operator经常性的不支持某些操作，而不是像使用者想象的那样，我改改yaml，apply一下就完事了<br>更重要的是，由于情况太多太复杂，甚至无法通过文档的方式，告诉使用者，状态1的集群，按某种方式修改了yaml之后，是不能变成使用者期待的状态2的<br><br>如果是传统的命令式方式，那么就是所有可能有的功能，都会罗列出来，可能n个状态，只有2n+3个操作是合法的，剩下都是做不到的，这样使用者固然受到了限制，但这种限制也让很多时候的操作变得更加清晰明了,而不是每次改yaml还要思考一下，这么改，operator支持吗？<br><br>当然，声明式api的好处也是明显的，如果这个diff是operator覆盖到的，那么就能极大的减轻使用者的负担，apply下就解脱了<br><br>而这个集群状态变迁的问题是本质复杂的，必然没有可以消除复杂度的解法，无非就是这个复杂度在operator的实现者那里，还是在运维者那里，在operator的实现者那里，好处就是固化的常用的变迁路径，使用起来很方便，但如果operator的开发者没有实现某个状态的变迁路径，而这个本身是可以做到的，这个时候，就比不上命令式灵活，个人感觉就是取舍了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-21 23:05:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/0f/70/c8680841.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Joe Black</span>
  </div>
  <div class="_2_QraFYR_0">看了这两天的文章，感觉k8s的机制实在是太具有普适性了，可以基于它构建各种分布式业务平台。本质上它就是一个分布式对象管理平台。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 说对了。所以说把kubernetes跟swarm mesos各种paas等横向比较没啥实际意义</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-19 12:54:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/8e/88/8cbf2527.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>超</span>
  </div>
  <div class="_2_QraFYR_0">如果一个master 管理的node非常多 通过ListAndWatch 会对master的性能有影响吧</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 所以说kubernetes 当前规模是5000</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-08 09:47:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/ac/c8/4b1c0d40.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>勤劳的小胖子-libo</span>
  </div>
  <div class="_2_QraFYR_0">一般这种工作队列结构主要是为了匹配双方速度不一致，也为了decouple双方。比如典型生产者消费者问题</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-15 20:51:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/a2/60/f3939ab4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>哈哼</span>
  </div>
  <div class="_2_QraFYR_0">自定义的controler这么手动跑着，挂了咋办？，是不是应该准备好镜像，用Deployment跑起来？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-03 22:37:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4e/60/0d5aa340.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>gogo</span>
  </div>
  <div class="_2_QraFYR_0">请问老师，如果用deployment部署一个tag是latest的镜像，怎样进行滚动更新呢？set image的话tag不变，不能出发更新呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 要用滚动更新就不要用latest tag，否则你准备怎么知道当前在用的是哪个版本的镜像？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-19 14:21:52</div>
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
  <div class="_2_QraFYR_0">Kuberentes的API对象由三部分组成，通常可以归结为： &#47;apis&#47;group&#47;version&#47;resource，例如<br><br>    apiVersion: Group&#47;Version<br>    kind: Resource<br><br>APIServer在接收一个请求之后，会按照 &#47;apis&#47;group&#47;version&#47;resource的路径找到对应的Resource类型定义，根据这个类型定义和用户提交的yaml里的字段创建出一个对应的Resource对象<br><br>CRD机制：<br>（1）声明一个CRD，让k8s能够认识和处理所有声明了API是&quot;&#47;apis&#47;group&#47;version&#47;resource&quot;的YAML文件了。包括：组（Group）、版本（Version）、资源类型（Resource）等。<br>（2）编写go代码，让k8s能够识别yaml对象的描述。包括：Spec、Status等<br>（3）使用k8s代码生成工具自动生成clientset、informer和lister<br>（4） 编写一个自定义控制器，来对所关心对象进行相关操作<br><br><br><br>（1）（2）步骤之后，就可以顺利通过kubectl apply xxx.yaml 来创建出对应的CRD和CR了。 但实际上，k8s的etcd中有了这样的对象，却不会进行实际的一些后续操作，因为我们还没有编写对应CRD的控制器。控制器需要：感知所关心对象过的变化，这是通过一个Informer来完成的。而Informer所需要的代码，正是上述（3）步骤生成。<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-07 20:59:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/ef/21/69c181b8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Rain</span>
  </div>
  <div class="_2_QraFYR_0">```此外，在这个过程中，每经过 resyncPeriod 指定的时间，Informer 维护的本地缓存，都会使用最近一次 LIST 返回的结果强制更新一次，从而保证缓存的有效性。在 Kubernetes 中，这个缓存强制更新的操作就叫作：resync。需要注意的是，这个定时 resync 操作，也会触发 Informer 注册的“更新”事件。```<br>老师，这个地方貌似说的有点问题，看代码逻辑，正常情况下，ListAndWatch只会执行一次，即先执行List把数据拉过来，然后更新一次本地缓存，后续就进入Watch阶段，通过监听事件来实时更新本地缓存，而只有在ListAndWatch过程中，因异常退出时，比如apiserver有问题，没法正常ListAndWatch时，才会周期性的尝试进行ListAndWatch，而这个周期也不是resyncPeriod来控制的，而是一个变动的backoff时间，以防止给apiserver造成压力，而真正的resync，其实说的不是周期性更新缓存，而是根据现有的缓存，周期性的触发UpdateFunc，即同步当前状态和目的状态，确保幂等性，这个周期则是由resyncPeriod控制的。所以“每经过resyncPeriod时间，强制更新缓存”，应该是没这个逻辑的吧，还请老师确认下:)<br><br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-08 23:54:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/98/b1/f89a84d0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Tokamak</span>
  </div>
  <div class="_2_QraFYR_0">实际操作环境: k8s v1.22.4, go1.17.4<br>将原始代码clone下来后, 执行 go mod init -&gt; go mod tidy -&gt; go build -o samplecrd-controller .<br><br>运行:<br>.&#47;samplecrd-controller -kubeconfig=$HOME&#47;.kube&#47;config -alsologtostderr=true<br><br>crd 的yaml格式需要修改为:<br>apiVersion: apiextensions.k8s.io&#47;v1<br>kind: CustomResourceDefinition<br>metadata:<br>  name: networks.samplecrd.k8s.io<br>  annotations:<br>    &quot;api-approved.kubernetes.io&quot;: &quot;https:&#47;&#47;github.com&#47;kubernetes&#47;kubernetes&#47;pull&#47;78458&quot;<br>spec:<br>  group: samplecrd.k8s.io<br>  names:<br>    kind: Network<br>    plural: networks<br>    singular: network<br>    shortNames:<br>    - nw<br>  scope: Namespaced<br>  versions:<br>  - served: true<br>    storage: true<br>    name: v1<br>    schema:<br>      openAPIV3Schema:<br>        type: object<br>        properties:<br>          spec:<br>            type: object<br>            properties:<br>              cidr:<br>                type: string<br>              gateway:<br>                type: string<br><br>之后按照文章中的操作即可</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-15 10:53:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLtwSXKialWYQgo1OXWYIsyj4zxK8AbQb1tp6iceZ5cGPYFcoczlubd7VicJPuvWqHrFXJXtUTP4kd9A/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Monokai</span>
  </div>
  <div class="_2_QraFYR_0">处理完api对象的事件后就直接存储在etcd里了么？需不需要再和apiserver打交道？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 需要，所有对etcd的操作都要走apiserver</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-18 14:57:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/6b/56/37a4cea7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>单朋荣</span>
  </div>
  <div class="_2_QraFYR_0">张老师好，很感谢您的课程分享。另外，我想深入做些自定义组件的开发，您一路已经走过来了，想听下您的建议，顺便推荐书，万分感谢！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-21 14:09:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/a9/41/9170f607.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>LinYongRui</span>
  </div>
  <div class="_2_QraFYR_0">张老师您好，请问如果在这个框架下，有人手动删除了一个实际的neutron network，但是本地缓存和apiserver的状态是一致的，那么在period sync的时候，是不是就不会去真正检查实际状态和本地缓存的差别了呢？因为我看到eventhandler的update那边会直接return了？ 谢谢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个逻辑可以自己在handler里处理一下，我没cover这种情况而已</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-10 05:23:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTI2vn8hyjICTCletGs0omz28lhriaZKX2XX9icYzAEon2IEoRnlXqyOia2bEPP0j7T6xexTnr77JJic8w/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_c22199</span>
  </div>
  <div class="_2_QraFYR_0">自定义？打扰打扰，等我过半个月再来</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-19 13:47:04</div>
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
  <div class="_2_QraFYR_0">请问这个控制器是跑在node节点上的？一般哪些控制器是跑在node上哪些是跑在master上呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 无所谓，哪都行</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-25 19:34:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/e9/88/941e488a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>hugeo</span>
  </div>
  <div class="_2_QraFYR_0">厉害了，原来这才是k8s的精髓</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-20 20:18:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/4f/eb/b5bb4227.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>runner</span>
  </div>
  <div class="_2_QraFYR_0">张老师，问个问题，我们公司的docker业务，容器总数上万个，部分容器依赖宿主机配置文件。现在我们想迁k8s 的话，能不改动这些容器，把他们加入pod管理起来么？如果上万容器都重新调度生成的话，这个改动太大了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 给它们写yaml描述起来吧。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-19 19:32:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/3a/27/5d218272.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>八台上</span>
  </div>
  <div class="_2_QraFYR_0">想问一下 工作队列或者delta队列  会持久化吗， 不然controller异常重启的话，队列信息不是丢了吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-16 18:29:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/97/95/aad51e9b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>waterjiao</span>
  </div>
  <div class="_2_QraFYR_0">磊哥，如果控制器一段时间不可用，删除的事件到了，这个资源在etcd中被删掉了，控制器重启后，期望状态和实际状态对不齐了，这个时候是不是还要用缓存的数据和实际数据做对比？如果数据较多时，又该怎么办呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-28 16:24:47</div>
  </div>
</div>
</div>
</li>
</ul>