<audio title="37 _ 找到容器不容易：Service、DNS与服务发现" src="https://static001.geekbang.org/resource/audio/d4/45/d4b42d64264d45ef71b9559d2d0fa145.mp3" controls="controls"></audio> 
<p>你好，我是张磊。今天我和你分享的主题是：找到容器不容易之Service、DNS与服务发现。</p><p>在前面的文章中，我们已经多次使用到了Service这个Kubernetes里重要的服务对象。而Kubernetes之所以需要Service，一方面是因为Pod的IP不是固定的，另一方面则是因为一组Pod实例之间总会有负载均衡的需求。</p><p>一个最典型的Service定义，如下所示：</p><pre><code>apiVersion: v1
kind: Service
metadata:
  name: hostnames
spec:
  selector:
    app: hostnames
  ports:
  - name: default
    protocol: TCP
    port: 80
    targetPort: 9376
</code></pre><p>这个Service的例子，相信你不会陌生。其中，我使用了selector字段来声明这个Service只代理携带了app=hostnames标签的Pod。并且，这个Service的80端口，代理的是Pod的9376端口。</p><p>然后，我们的应用的Deployment，如下所示：</p><pre><code>apiVersion: apps/v1
kind: Deployment
metadata:
  name: hostnames
spec:
  selector:
    matchLabels:
      app: hostnames
  replicas: 3
  template:
    metadata:
      labels:
        app: hostnames
    spec:
      containers:
      - name: hostnames
        image: k8s.gcr.io/serve_hostname
        ports:
        - containerPort: 9376
          protocol: TCP
</code></pre><p>这个应用的作用，就是每次访问9376端口时，返回它自己的hostname。</p><p>而被selector选中的Pod，就称为Service的Endpoints，你可以使用kubectl get ep命令看到它们，如下所示：</p><pre><code>$ kubectl get endpoints hostnames
NAME        ENDPOINTS
hostnames   10.244.0.5:9376,10.244.0.6:9376,10.244.0.7:9376
</code></pre><p>需要注意的是，只有处于Running状态，且readinessProbe检查通过的Pod，才会出现在Service的Endpoints列表里。并且，当某一个Pod出现问题时，Kubernetes会自动把它从Service里摘除掉。</p><!-- [[[read_end]]] --><p>而此时，通过该Service的VIP地址10.0.1.175，你就可以访问到它所代理的Pod了：</p><pre><code>$ kubectl get svc hostnames
NAME        TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
hostnames   ClusterIP   10.0.1.175   &lt;none&gt;        80/TCP    5s

$ curl 10.0.1.175:80
hostnames-0uton

$ curl 10.0.1.175:80
hostnames-yp2kp

$ curl 10.0.1.175:80
hostnames-bvc05
</code></pre><p>这个VIP地址是Kubernetes自动为Service分配的。而像上面这样，通过三次连续不断地访问Service的VIP地址和代理端口80，它就为我们依次返回了三个Pod的hostname。这也正印证了Service提供的是Round Robin方式的负载均衡。对于这种方式，我们称为：ClusterIP模式的Service。</p><p>你可能一直比较好奇，Kubernetes里的Service究竟是如何工作的呢？</p><p>实际上，<strong>Service是由kube-proxy组件，加上iptables来共同实现的。</strong></p><p>举个例子，对于我们前面创建的名叫hostnames的Service来说，一旦它被提交给Kubernetes，那么kube-proxy就可以通过Service的Informer感知到这样一个Service对象的添加。而作为对这个事件的响应，它就会在宿主机上创建这样一条iptables规则（你可以通过iptables-save看到它），如下所示：</p><pre><code>-A KUBE-SERVICES -d 10.0.1.175/32 -p tcp -m comment --comment &quot;default/hostnames: cluster IP&quot; -m tcp --dport 80 -j KUBE-SVC-NWV5X2332I4OT4T3
</code></pre><p>可以看到，这条iptables规则的含义是：凡是目的地址是10.0.1.175、目的端口是80的IP包，都应该跳转到另外一条名叫KUBE-SVC-NWV5X2332I4OT4T3的iptables链进行处理。</p><p>而我们前面已经看到，10.0.1.175正是这个Service的VIP。所以这一条规则，就为这个Service设置了一个固定的入口地址。并且，由于10.0.1.175只是一条iptables规则上的配置，并没有真正的网络设备，所以你ping这个地址，是不会有任何响应的。</p><p>那么，我们即将跳转到的KUBE-SVC-NWV5X2332I4OT4T3规则，又有什么作用呢？</p><p>实际上，它是一组规则的集合，如下所示：</p><pre><code>-A KUBE-SVC-NWV5X2332I4OT4T3 -m comment --comment &quot;default/hostnames:&quot; -m statistic --mode random --probability 0.33332999982 -j KUBE-SEP-WNBA2IHDGP2BOBGZ
-A KUBE-SVC-NWV5X2332I4OT4T3 -m comment --comment &quot;default/hostnames:&quot; -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-X3P2623AGDH6CDF3
-A KUBE-SVC-NWV5X2332I4OT4T3 -m comment --comment &quot;default/hostnames:&quot; -j KUBE-SEP-57KPRZ3JQVENLNBR
</code></pre><p>可以看到，这一组规则，实际上是一组随机模式（–mode random）的iptables链。</p><p>而随机转发的目的地，分别是KUBE-SEP-WNBA2IHDGP2BOBGZ、KUBE-SEP-X3P2623AGDH6CDF3和KUBE-SEP-57KPRZ3JQVENLNBR。</p><p>而这三条链指向的最终目的地，其实就是这个Service代理的三个Pod。所以这一组规则，就是Service实现负载均衡的位置。</p><p>需要注意的是，iptables规则的匹配是从上到下逐条进行的，所以为了保证上述三条规则每条被选中的概率都相同，我们应该将它们的probability字段的值分别设置为1/3（0.333…）、1/2和1。</p><p>这么设置的原理很简单：第一条规则被选中的概率就是1/3；而如果第一条规则没有被选中，那么这时候就只剩下两条规则了，所以第二条规则的probability就必须设置为1/2；类似地，最后一条就必须设置为1。</p><p>你可以想一下，如果把这三条规则的probability字段的值都设置成1/3，最终每条规则被选中的概率会变成多少。</p><p>通过查看上述三条链的明细，我们就很容易理解Service进行转发的具体原理了，如下所示：</p><pre><code>-A KUBE-SEP-57KPRZ3JQVENLNBR -s 10.244.3.6/32 -m comment --comment &quot;default/hostnames:&quot; -j MARK --set-xmark 0x00004000/0x00004000
-A KUBE-SEP-57KPRZ3JQVENLNBR -p tcp -m comment --comment &quot;default/hostnames:&quot; -m tcp -j DNAT --to-destination 10.244.3.6:9376

-A KUBE-SEP-WNBA2IHDGP2BOBGZ -s 10.244.1.7/32 -m comment --comment &quot;default/hostnames:&quot; -j MARK --set-xmark 0x00004000/0x00004000
-A KUBE-SEP-WNBA2IHDGP2BOBGZ -p tcp -m comment --comment &quot;default/hostnames:&quot; -m tcp -j DNAT --to-destination 10.244.1.7:9376

-A KUBE-SEP-X3P2623AGDH6CDF3 -s 10.244.2.3/32 -m comment --comment &quot;default/hostnames:&quot; -j MARK --set-xmark 0x00004000/0x00004000
-A KUBE-SEP-X3P2623AGDH6CDF3 -p tcp -m comment --comment &quot;default/hostnames:&quot; -m tcp -j DNAT --to-destination 10.244.2.3:9376
</code></pre><p>可以看到，这三条链，其实是三条DNAT规则。但在DNAT规则之前，iptables对流入的IP包还设置了一个“标志”（–set-xmark）。这个“标志”的作用，我会在下一篇文章再为你讲解。</p><p>而DNAT规则的作用，就是在PREROUTING检查点之前，也就是在路由之前，将流入IP包的目的地址和端口，改成–to-destination所指定的新的目的地址和端口。可以看到，这个目的地址和端口，正是被代理Pod的IP地址和端口。</p><p>这样，访问Service VIP的IP包经过上述iptables处理之后，就已经变成了访问具体某一个后端Pod的IP包了。不难理解，这些Endpoints对应的iptables规则，正是kube-proxy通过监听Pod的变化事件，在宿主机上生成并维护的。</p><p>以上，就是Service最基本的工作原理。</p><p>此外，你可能已经听说过，Kubernetes的kube-proxy还支持一种叫作IPVS的模式。这又是怎么一回事儿呢？</p><p>其实，通过上面的讲解，你可以看到，kube-proxy通过iptables处理Service的过程，其实需要在宿主机上设置相当多的iptables规则。而且，kube-proxy还需要在控制循环里不断地刷新这些规则来确保它们始终是正确的。</p><p>不难想到，当你的宿主机上有大量Pod的时候，成百上千条iptables规则不断地被刷新，会大量占用该宿主机的CPU资源，甚至会让宿主机“卡”在这个过程中。所以说，<strong>一直以来，基于iptables的Service实现，都是制约Kubernetes项目承载更多量级的Pod的主要障碍。</strong></p><p>而IPVS模式的Service，就是解决这个问题的一个行之有效的方法。</p><p>IPVS模式的工作原理，其实跟iptables模式类似。当我们创建了前面的Service之后，kube-proxy首先会在宿主机上创建一个虚拟网卡（叫作：kube-ipvs0），并为它分配Service VIP作为IP地址，如下所示：</p><pre><code># ip addr
  ...
  73：kube-ipvs0：&lt;BROADCAST,NOARP&gt;  mtu 1500 qdisc noop state DOWN qlen 1000
  link/ether  1a:ce:f5:5f:c1:4d brd ff:ff:ff:ff:ff:ff
  inet 10.0.1.175/32  scope global kube-ipvs0
  valid_lft forever  preferred_lft forever
</code></pre><p>而接下来，kube-proxy就会通过Linux的IPVS模块，为这个IP地址设置三个IPVS虚拟主机，并设置这三个虚拟主机之间使用轮询模式(rr)来作为负载均衡策略。我们可以通过ipvsadm查看到这个设置，如下所示：</p><pre><code># ipvsadm -ln
 IP Virtual Server version 1.2.1 (size=4096)
  Prot LocalAddress:Port Scheduler Flags
    -&gt;  RemoteAddress:Port           Forward  Weight ActiveConn InActConn     
  TCP  10.102.128.4:80 rr
    -&gt;  10.244.3.6:9376    Masq    1       0          0         
    -&gt;  10.244.1.7:9376    Masq    1       0          0
    -&gt;  10.244.2.3:9376    Masq    1       0          0
</code></pre><p>可以看到，这三个IPVS虚拟主机的IP地址和端口，对应的正是三个被代理的Pod。</p><p>这时候，任何发往10.102.128.4:80的请求，就都会被IPVS模块转发到某一个后端Pod上了。</p><p>而相比于iptables，IPVS在内核中的实现其实也是基于Netfilter的NAT模式，所以在转发这一层上，理论上IPVS并没有显著的性能提升。但是，IPVS并不需要在宿主机上为每个Pod设置iptables规则，而是把对这些“规则”的处理放到了内核态，从而极大地降低了维护这些规则的代价。这也正印证了我在前面提到过的，“将重要操作放入内核态”是提高性能的重要手段。</p><blockquote>
<p>备注：这里你可以再回顾下第33篇文章<a href="https://time.geekbang.org/column/article/65287">《深入解析容器跨主机网络》</a>中的相关内容。</p>
</blockquote><p>不过需要注意的是，IPVS模块只负责上述的负载均衡和代理功能。而一个完整的Service流程正常工作所需要的包过滤、SNAT等操作，还是要靠iptables来实现。只不过，这些辅助性的iptables规则数量有限，也不会随着Pod数量的增加而增加。</p><p>所以，在大规模集群里，我非常建议你为kube-proxy设置–proxy-mode=ipvs来开启这个功能。它为Kubernetes集群规模带来的提升，还是非常巨大的。</p><p><strong>此外，我在前面的文章中还介绍过Service与DNS的关系。</strong></p><p>在Kubernetes中，Service和Pod都会被分配对应的DNS A记录（从域名解析IP的记录）。</p><p>对于ClusterIP模式的Service来说（比如我们上面的例子），它的A记录的格式是：<my-svc>.<my-namespace>.svc.cluster.local。当你访问这条A记录的时候，它解析到的就是该Service的VIP地址。</my-namespace></my-svc></p><p>而对于指定了clusterIP=None的Headless Service来说，它的A记录的格式也是：<my-svc>.<my-namespace>.svc.cluster.local。但是，当你访问这条A记录的时候，它返回的是所有被代理的Pod的IP地址的集合。当然，如果你的客户端没办法解析这个集合的话，它可能会只会拿到第一个Pod的IP地址。</my-namespace></my-svc></p><p>此外，对于ClusterIP模式的Service来说，它代理的Pod被自动分配的A记录的格式是：<pod-ip>.<my-namespace>.pod.cluster.local。这条记录指向Pod的IP地址。</my-namespace></pod-ip></p><p>而对Headless Service来说，它代理的Pod被自动分配的A记录的格式是：<my-pod-name>.<my-service-name>.<my-namespace>.svc.cluster.local。这条记录也指向Pod的IP地址。</my-namespace></my-service-name></my-pod-name></p><p>但如果你为Pod指定了Headless Service，并且Pod本身声明了hostname和subdomain字段，那么这时候Pod的A记录就会变成：&lt;pod的hostname&gt;.<subdomain>.<my-namespace>.svc.cluster.local，比如：</my-namespace></subdomain></p><pre><code>apiVersion: v1
kind: Service
metadata:
  name: default-subdomain
spec:
  selector:
    name: busybox
  clusterIP: None
  ports:
  - name: foo
    port: 1234
    targetPort: 1234
---
apiVersion: v1
kind: Pod
metadata:
  name: busybox1
  labels:
    name: busybox
spec:
  hostname: busybox-1
  subdomain: default-subdomain
  containers:
  - image: busybox
    command:
      - sleep
      - &quot;3600&quot;
    name: busybox
</code></pre><p>在上面这个Service和Pod被创建之后，你就可以通过busybox-1.default-subdomain.default.svc.cluster.local解析到这个Pod的IP地址了。</p><p>需要注意的是，在Kubernetes里，/etc/hosts文件是单独挂载的，这也是为什么kubelet能够对hostname进行修改并且Pod重建后依然有效的原因。这跟Docker的Init层是一个原理。</p><h2>总结</h2><p>在这篇文章里，我为你详细讲解了Service的工作原理。实际上，Service机制，以及Kubernetes里的DNS插件，都是在帮助你解决同样一个问题，即：如何找到我的某一个容器？</p><p>这个问题在平台级项目中，往往就被称作服务发现，即：当我的一个服务（Pod）的IP地址是不固定的且没办法提前获知时，我该如何通过一个固定的方式访问到这个Pod呢？</p><p>而我在这里讲解的、ClusterIP模式的Service为你提供的，就是一个Pod的稳定的IP地址，即VIP。并且，这里Pod和Service的关系是可以通过Label确定的。</p><p>而Headless Service为你提供的，则是一个Pod的稳定的DNS名字，并且，这个名字是可以通过Pod名字和Service名字拼接出来的。</p><p>在实际的场景里，你应该根据自己的具体需求进行合理选择。</p><h2>思考题</h2><p>请问，Kubernetes的Service的负载均衡策略，在iptables和ipvs模式下，都有哪几种？具体工作模式是怎样的？</p><p>感谢你的收听，欢迎你给我留言，也欢迎分享给更多的朋友一起阅读。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/7f/1a/eb8021c3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>追寻云的痕迹</span>
  </div>
  <div class="_2_QraFYR_0">iptables是万恶之源，在复杂系统中，网络处理越简单越好。现在k8s这套玩法，给实际工作中的运维排错带来极大的麻烦。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-16 08:17:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/97/8b/bd318156.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>qingbo</span>
  </div>
  <div class="_2_QraFYR_0">看到也有同学问pod DNS，希望能讲得更详细些。我查阅官方文档及自己实践后的了解是这两种pod有DNS记录：<br>1. statefulset的pod。有人问之前讲DNS的是在哪，就是“20 | 深入理解StatefulSet（三）：有状态应用实践”这一篇。<br>2. pod显式指定了hostname和subdomain，并且有个headless service的名字和subdomain一样。在“27 | 聪明的微创新：Operator工作原理解读”一篇中讲到的etcd operator就是这样让pod拥有了DNS记录。Deployment的pod template也可以指定hostname和subdomain，但是却没办法给每个pod分配不同的hostname。指定hostname和subdomain之后，hostname.subdomain.default.svc.cluster.local这样的域名确实可以解析，但是因为多个pod都是这个FQDN，所以解析出来的效果和headless service一样，多个A记录，也就失去意义了。github上有个issue想让deployment管理的pod也有独立的DNS，好像没得到支持。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-30 09:11:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/ea/72/92e43306.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>纳爱斯</span>
  </div>
  <div class="_2_QraFYR_0">老师，是每个 node 上都会有 iptables 的全部规则吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-11 22:53:59</div>
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
  <div class="_2_QraFYR_0">老师，..svc.cluster.local这些点前面的东西能写全吗？录音听了N次也没记下来。文字不行的话能不能弄个图片？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-16 22:29:09</div>
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
  <div class="_2_QraFYR_0">示例终于都可以工作了，深化理解。<br>一种是通过&lt;serviceName&gt;.&lt;namespace&gt;.svc.cluster.local访问。对应于clusterIP<br>另一种是通过&lt;podName&gt;.&lt;serviceName&gt;.&lt;namesapce&gt;.svc.cluster.local访问,对应于headless service.<br>&#47; # nslookup *.default.svc.cluster.local<br>Server:    10.96.0.10<br>Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local<br><br>Name:      *.default.svc.cluster.local<br>Address 1: 10.244.1.7 busybox-3.default-subdomain.default.svc.cluster.local<br>Address 2: 10.96.0.1 kubernetes.default.svc.cluster.local<br>Address 3: 10.97.103.223 hostnames.default.svc.cluster.local<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-04 14:41:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/a5/98/a65ff31a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>djfhchdh</span>
  </div>
  <div class="_2_QraFYR_0">ipvs负载均衡：round robin<br>    least connection<br>    destination hashing<br>    source hashing<br>    shortest expected delay<br>    never queue<br>    overflow-connection</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-25 17:21:21</div>
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
  <div class="_2_QraFYR_0">请问老师，每个节点会有全部的iptables规则么，还是只有自己所属服务的规则？<br>如果服务是nodePort类型，它会在所有节点上占用端口？还是容器所在的几个节点占用端口？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-16 09:53:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/35/5b/c9d7e8c0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>李康</span>
  </div>
  <div class="_2_QraFYR_0"><br>-A KUBE-SEP-57KPRZ3JQVENLNBR -s 10.244.3.6&#47;32 -m comment --comment &quot;default&#47;hostnames:&quot; -j MARK --set-xmark 0x00004000&#47;0x00004000<br><br>-A KUBE-SEP-WNBA2IHDGP2BOBGZ -s 10.244.1.7&#47;32 -m comment --comment &quot;default&#47;hostnames:&quot; -j MARK --set-xmark 0x00004000&#47;0x00004000<br><br>-A KUBE-SEP-X3P2623AGDH6CDF3 -s 10.244.2.3&#47;32 -m comment --comment &quot;default&#47;hostnames:&quot; -j MARK --set-xmark 0x00004000&#47;0x00004000<br><br>问下为啥源地址是这个啊？现在数据包源地址不应该是客户端的地址吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-23 19:32:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/99/d1/2c6f297f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>mcc</span>
  </div>
  <div class="_2_QraFYR_0">描述一个实际使用中遇到kube-proxy的一个问题。我使用service的nodeport模式对外发布服务，前端使用openresty做代理的，upstream就配置node ip+nodeport。在使用过程中发现openresty经常不定期报104:connection reset by peer when read response head这个错误，从错误看出openstry从nodeport读取数据的时候tcp连接被重置了，使用同一openresty的后端是普通虚拟机的节点的服务却没有这个问题，问题还有个特点就是某个服务长时间没有被访问，第一次点击的时候就会出现，然后后面就好了。nodeport是被kube-proxy监听的，问题就出在openresty与kube-proxy的tcp连接上，能否帮忙分析kube-proxy为何会重置连接？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-16 00:32:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/e6/2a/2ac18fb8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>艾斯Z艾穆</span>
  </div>
  <div class="_2_QraFYR_0">您好，我使用coreDNS插件版本是1.2.6 <br>配置文件的内容：<br>Corefile: |<br>    .:53 {<br>        errors<br>        health<br>        kubernetes cluster.local. in-addr.arpa ip6.arpa {<br>            pods insecure<br>            upstream<br>            fallthrough in-addr.arpa ip6.arpa<br>        }<br>        prometheus :9153<br>        proxy . 100.64.255.100 223.5.5.5 11.125.1.12<br>        cache 30<br>        loop<br>        reload<br>        loadbalance<br>    }<br>在解析公网的域名的时候会有小概率随机出现unknown host，请问会是什么问题</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-03 14:03:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/8c/0c/a36cdf51.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>grep</span>
  </div>
  <div class="_2_QraFYR_0">示例里这里打出来的 endpoints ip：<br>kubectl get endpoints hostnames<br>NAME        ENDPOINTS<br>hostnames   10.244.0.5:9376,10.244.0.6:9376,10.244.0.7:9376<br><br>与下面的 iptables 规则里的 endpoint ip 对不上<br>-A KUBE-SEP-57KPRZ3JQVENLNBR -s 10.244.3.6&#47;32 -m comment --comment &quot;default&#47;hostnames:&quot; -j MARK --set-xmark 0x00004000&#47;0x00004000<br>-A KUBE-SEP-57KPRZ3JQVENLNBR -p tcp -m comment --comment &quot;default&#47;hostnames:&quot; -m tcp -j DNAT --to-destination 10.244.3.6:9376<br><br>-A KUBE-SEP-WNBA2IHDGP2BOBGZ -s 10.244.1.7&#47;32 -m comment --comment &quot;default&#47;hostnames:&quot; -j MARK --set-xmark 0x00004000&#47;0x00004000<br>-A KUBE-SEP-WNBA2IHDGP2BOBGZ -p tcp -m comment --comment &quot;default&#47;hostnames:&quot; -m tcp -j DNAT --to-destination 10.244.1.7:9376<br><br>-A KUBE-SEP-X3P2623AGDH6CDF3 -s 10.244.2.3&#47;32 -m comment --comment &quot;default&#47;hostnames:&quot; -j MARK --set-xmark 0x00004000&#47;0x00004000<br>-A KUBE-SEP-X3P2623AGDH6CDF3 -p tcp -m comment --comment &quot;default&#47;hostnames:&quot; -m tcp -j DNAT --to-destination 10.244.2.3:9376<br><br>是不是中间重新部署过？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-27 17:27:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/6a/f2/fee557fe.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Mr.Cling</span>
  </div>
  <div class="_2_QraFYR_0">这时候，任何发往 10.102.128.4:80 的请求，就都会被 IPVS 模块转发到某一个后端 Pod 上了。<br><br>请问这里的10.102.128.4的IP是什么IP？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-30 22:39:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/45/82/164b9073.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Majorin_Che</span>
  </div>
  <div class="_2_QraFYR_0">所以这里服务发现的方式就是通过label发现pod，是这样理解吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-16 06:24:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/66/6c/4a68a916.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>双叶</span>
  </div>
  <div class="_2_QraFYR_0">不太赞同 ipvs 比 iptables 快是因为把更多的操作放到了内核里面，不管 ipvs 还是 iptables，他的所有逻辑都是在内核里面跑的，只是 iptables 需要遍历规则，而规则数量会跟随 pod 数量增长，导致时间复杂度是 O(n)，而 ipvs 是专门做负载均衡的，时间复杂度是 O(1)。<br><br>这篇文章里面有比较细致的说明：https:&#47;&#47;www.tigera.io&#47;blog&#47;comparing-kube-proxy-modes-iptables-or-ipvs&#47;<br>基本来说，就是 iptables 不能放太多规则</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-11 05:40:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/d9/ff/b23018a6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Heaven</span>
  </div>
  <div class="_2_QraFYR_0">必然会存在着轮询去进行负载均衡的策略以及利用会话保持的进行保证客户端分配固定的pod Ip的模式</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-30 16:47:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/a5/98/a65ff31a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>djfhchdh</span>
  </div>
  <div class="_2_QraFYR_0">iptables的负载均衡分两种：random &#47; nth，random是随机模式，--probability p指定了概率，nth是轮巡模式，--every n和--packet p指定了每n个packet中匹配其中的第p个。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-25 15:21:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>思维决定未来</span>
  </div>
  <div class="_2_QraFYR_0">如果把这三条规则的 probability 字段的值都设置成1&#47;3，那么第一条规则命中几率是1&#47;3，第二条是2&#47;3 * 1&#47;3=2&#47;9，第三条是1&#47;3 * 1&#47;3 = 1&#47;9</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-22 11:32:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/b1/da/88197585.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>甘陵笑笑生</span>
  </div>
  <div class="_2_QraFYR_0">请教一下 service的VIP设置后会变吗 如果变 什么时候会变</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不会的，除非删除svc</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-14 17:57:27</div>
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
  <div class="_2_QraFYR_0">&quot;我在前面的文章中还介绍过 Service 与 DNS 的关系.&quot;可以帮忙指明一下是第几章吗？找了一圈没找到。<br><br>另外：试着通过域名访问hostanmes，不行。通过ip可以。<br>vagrant@kubeadm1:~&#47;37ServiceDns$ curl hostnames.svc.cluster.local:80<br>curl: (6) Could not resolve host: hostnames.svc.cluster.local<br><br>vagrant@kubeadm1:~&#47;37ServiceDns$ curl 10.110.252.216:80<br>hostnames-84985c9fdd-sgwpp<br><br><br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-22 14:23:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/c3/bd/03a16b21.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Long Long☞</span>
  </div>
  <div class="_2_QraFYR_0">老师 现在我遇到一个问题，同一个域名希望在不同的namespace中解析成不同的IP，要怎么实现</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-20 14:20:39</div>
  </div>
</div>
</div>
</li>
</ul>