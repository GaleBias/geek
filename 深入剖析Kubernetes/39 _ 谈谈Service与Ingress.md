<audio title="39 _ 谈谈Service与Ingress" src="https://static001.geekbang.org/resource/audio/7b/97/7b6585028a2841253d4a7dab2ce1ee97.mp3" controls="controls"></audio> 
<p>你好，我是张磊。今天我和你分享的主题是：谈谈Service与Ingress。</p><p>在上一篇文章中，我为你详细讲解了将Service暴露给外界的三种方法。其中有一个叫作LoadBalancer类型的Service，它会为你在Cloud Provider（比如：Google Cloud或者OpenStack）里创建一个与该Service对应的负载均衡服务。</p><p>但是，相信你也应该能感受到，由于每个 Service 都要有一个负载均衡服务，所以这个做法实际上既浪费成本又高。作为用户，我其实更希望看到Kubernetes为我内置一个全局的负载均衡器。然后，通过我访问的URL，把请求转发给不同的后端Service。</p><p><strong>这种全局的、为了代理不同后端Service而设置的负载均衡服务，就是Kubernetes里的Ingress服务。</strong></p><p>所以，Ingress的功能其实很容易理解：<strong>所谓Ingress，就是Service的“Service”。</strong></p><p>举个例子，假如我现在有这样一个站点：<code>https://cafe.example.com</code>。其中，<code>https://cafe.example.com/coffee</code>，对应的是“咖啡点餐系统”。而，<code>https://cafe.example.com/tea</code>，对应的则是“茶水点餐系统”。这两个系统，分别由名叫coffee和tea这样两个Deployment来提供服务。</p><!-- [[[read_end]]] --><p>那么现在，<span class="orange">我如何能使用Kubernetes的Ingress来创建一个统一的负载均衡器，从而实现当用户访问不同的域名时，能够访问到不同的Deployment呢？</span></p><p>上述功能，在Kubernetes里就需要通过Ingress对象来描述，如下所示：</p><pre><code>apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: cafe-ingress
spec:
  tls:
  - hosts:
    - cafe.example.com
    secretName: cafe-secret
  rules:
  - host: cafe.example.com
    http:
      paths:
      - path: /tea
        backend:
          serviceName: tea-svc
          servicePort: 80
      - path: /coffee
        backend:
          serviceName: coffee-svc
          servicePort: 80
</code></pre><p>在上面这个名叫cafe-ingress.yaml文件中，最值得我们关注的，是rules字段。在Kubernetes里，这个字段叫作：<strong>IngressRule</strong>。</p><p>IngressRule的Key，就叫做：host。它必须是一个标准的域名格式（Fully Qualified Domain Name）的字符串，而不能是IP地址。</p><blockquote>
<p>备注：Fully Qualified Domain Name的具体格式，可以参考<a href="https://tools.ietf.org/html/rfc3986">RFC 3986</a>标准。</p>
</blockquote><p>而host字段定义的值，就是这个Ingress的入口。这也就意味着，当用户访问cafe.example.com的时候，实际上访问到的是这个Ingress对象。这样，Kubernetes就能使用IngressRule来对你的请求进行下一步转发。</p><p>而接下来IngressRule规则的定义，则依赖于path字段。你可以简单地理解为，这里的每一个path都对应一个后端Service。所以在我们的例子里，我定义了两个path，它们分别对应coffee和tea这两个Deployment的Service（即：coffee-svc和tea-svc）。</p><p><strong>通过上面的讲解，不难看到，所谓Ingress对象，其实就是Kubernetes项目对“反向代理”的一种抽象。</strong></p><p>一个Ingress对象的主要内容，实际上就是一个“反向代理”服务（比如：Nginx）的配置文件的描述。而这个代理服务对应的转发规则，就是IngressRule。</p><p>这就是为什么在每条IngressRule里，需要有一个host字段来作为这条IngressRule的入口，然后还需要有一系列path字段来声明具体的转发策略。这其实跟Nginx、HAproxy等项目的配置文件的写法是一致的。</p><p>而有了Ingress这样一个统一的抽象，Kubernetes的用户就无需关心Ingress的具体细节了。</p><p>在实际的使用中，你只需要从社区里选择一个具体的Ingress Controller，把它部署在Kubernetes集群里即可。</p><p>然后，这个Ingress Controller会根据你定义的Ingress对象，提供对应的代理能力。目前，业界常用的各种反向代理项目，比如Nginx、HAProxy、Envoy、Traefik等，都已经为Kubernetes专门维护了对应的Ingress Controller。</p><p>接下来，<span class="orange">我就以最常用的Nginx Ingress Controller为例，在我们前面用kubeadm部署的Bare-metal环境中，和你实践一下Ingress机制的使用过程。</span></p><p>部署Nginx Ingress Controller的方法非常简单，如下所示：</p><pre><code>$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/mandatory.yaml
</code></pre><p>其中，在<a href="https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/mandatory.yaml">mandatory.yaml</a>这个文件里，正是Nginx官方为你维护的Ingress Controller的定义。我们来看一下它的内容：</p><pre><code>kind: ConfigMap
apiVersion: v1
metadata:
  name: nginx-configuration
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
      app.kubernetes.io/part-of: ingress-nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/name: ingress-nginx
        app.kubernetes.io/part-of: ingress-nginx
      annotations:
        ...
    spec:
      serviceAccountName: nginx-ingress-serviceaccount
      containers:
        - name: nginx-ingress-controller
          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.20.0
          args:
            - /nginx-ingress-controller
            - --configmap=$(POD_NAMESPACE)/nginx-configuration
            - --publish-service=$(POD_NAMESPACE)/ingress-nginx
            - --annotations-prefix=nginx.ingress.kubernetes.io
          securityContext:
            capabilities:
              drop:
                - ALL
              add:
                - NET_BIND_SERVICE
            # www-data -&gt; 33
            runAsUser: 33
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
            - name: http
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - name: http
              containerPort: 80
            - name: https
              containerPort: 443
</code></pre><p>可以看到，在上述YAML文件中，我们定义了一个使用nginx-ingress-controller镜像的Pod。需要注意的是，这个Pod的启动命令需要使用该Pod所在的Namespace作为参数。而这个信息，当然是通过Downward API拿到的，即：Pod的env字段里的定义（env.valueFrom.fieldRef.fieldPath）。</p><p>而这个Pod本身，就是一个监听Ingress对象以及它所代理的后端Service变化的控制器。</p><p>当一个新的Ingress对象由用户创建后，nginx-ingress-controller就会根据Ingress对象里定义的内容，生成一份对应的Nginx配置文件（/etc/nginx/nginx.conf），并使用这个配置文件启动一个 Nginx 服务。</p><p>而一旦Ingress对象被更新，nginx-ingress-controller就会更新这个配置文件。需要注意的是，如果这里只是被代理的 Service 对象被更新，nginx-ingress-controller所管理的 Nginx 服务是不需要重新加载（reload）的。这当然是因为nginx-ingress-controller通过<a href="https://github.com/openresty/lua-nginx-module">Nginx Lua</a>方案实现了Nginx Upstream的动态配置。</p><p>此外，nginx-ingress-controller还允许你通过Kubernetes的ConfigMap对象来对上述 Nginx 配置文件进行定制。这个ConfigMap的名字，需要以参数的方式传递给nginx-ingress-controller。而你在这个 ConfigMap 里添加的字段，将会被合并到最后生成的 Nginx 配置文件当中。</p><p><strong>可以看到，一个Nginx Ingress Controller为你提供的服务，其实是一个可以根据Ingress对象和被代理后端 Service 的变化，来自动进行更新的Nginx负载均衡器。</strong></p><p>当然，为了让用户能够用到这个Nginx，我们就需要创建一个Service来把Nginx Ingress Controller管理的 Nginx 服务暴露出去，如下所示：</p><pre><code>$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/provider/baremetal/service-nodeport.yaml
</code></pre><p>由于我们使用的是Bare-metal环境，所以service-nodeport.yaml文件里的内容，就是一个NodePort类型的Service，如下所示：</p><pre><code>apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  type: NodePort
  ports:
    - name: http
      port: 80
      targetPort: 80
      protocol: TCP
    - name: https
      port: 443
      targetPort: 443
      protocol: TCP
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
</code></pre><p>可以看到，这个Service的唯一工作，就是将所有携带ingress-nginx标签的Pod的80和433端口暴露出去。</p><blockquote>
<p>而如果你是公有云上的环境，你需要创建的就是LoadBalancer类型的Service了。</p>
</blockquote><p><strong>上述操作完成后，你一定要记录下这个Service的访问入口，即：宿主机的地址和NodePort的端口</strong>，如下所示：</p><pre><code>$ kubectl get svc -n ingress-nginx
NAME            TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx   NodePort   10.105.72.96   &lt;none&gt;        80:30044/TCP,443:31453/TCP   3h
</code></pre><p>为了后面方便使用，我会把上述访问入口设置为环境变量：</p><pre><code>$ IC_IP=10.168.0.2 # 任意一台宿主机的地址
$ IC_HTTPS_PORT=31453 # NodePort端口
</code></pre><p><span class="orange">在Ingress Controller和它所需要的Service部署完成后，我们就可以使用它了。</span></p><blockquote>
<p>备注：这个“咖啡厅”Ingress的所有示例文件，都在<a href="https://github.com/resouer/kubernetes-ingress/tree/master/examples/complete-example">这里</a>。</p>
</blockquote><p>首先，我们要在集群里部署我们的应用Pod和它们对应的Service，如下所示：</p><pre><code>$ kubectl create -f cafe.yaml
</code></pre><p>然后，我们需要创建Ingress所需的SSL证书（tls.crt）和密钥（tls.key），这些信息都是通过Secret对象定义好的，如下所示：</p><pre><code>$ kubectl create -f cafe-secret.yaml
</code></pre><p>这一步完成后，我们就可以创建在本篇文章一开始定义的Ingress对象了，如下所示：</p><pre><code>$ kubectl create -f cafe-ingress.yaml
</code></pre><p>这时候，我们就可以查看一下这个Ingress对象的信息，如下所示：</p><pre><code>$ kubectl get ingress
NAME           HOSTS              ADDRESS   PORTS     AGE
cafe-ingress   cafe.example.com             80, 443   2h

$ kubectl describe ingress cafe-ingress
Name:             cafe-ingress
Namespace:        default
Address:          
Default backend:  default-http-backend:80 (&lt;none&gt;)
TLS:
  cafe-secret terminates cafe.example.com
Rules:
  Host              Path  Backends
  ----              ----  --------
  cafe.example.com  
                    /tea      tea-svc:80 (&lt;none&gt;)
                    /coffee   coffee-svc:80 (&lt;none&gt;)
Annotations:
Events:
  Type    Reason  Age   From                      Message
  ----    ------  ----  ----                      -------
  Normal  CREATE  4m    nginx-ingress-controller  Ingress default/cafe-ingress
</code></pre><p>可以看到，这个Ingress对象最核心的部分，正是Rules字段。其中，我们定义的Host是<code>cafe.example.com</code>，它有两条转发规则（Path），分别转发给tea-svc和coffee-svc。</p><blockquote>
<p>当然，在Ingress的YAML文件里，你还可以定义多个Host，比如<code>restaurant.example.com</code>、<code>movie.example.com</code>等等，来为更多的域名提供负载均衡服务。</p>
</blockquote><p>接下来，我们就可以通过访问这个Ingress的地址和端口，访问到我们前面部署的应用了，比如，当我们访问<code>https://cafe.example.com:443/coffee</code>时，应该是coffee这个Deployment负责响应我的请求。我们可以来尝试一下：</p><pre><code>$ curl --resolve cafe.example.com:$IC_HTTPS_PORT:$IC_IP https://cafe.example.com:$IC_HTTPS_PORT/coffee --insecureServer address: 10.244.1.56:80
Server name: coffee-7dbb5795f6-vglbv
Date: 03/Nov/2018:03:55:32 +0000
URI: /coffee
Request ID: e487e672673195c573147134167cf898
</code></pre><p>我们可以看到，访问这个URL 得到的返回信息是：Server name: coffee-7dbb5795f6-vglbv。这正是 coffee 这个 Deployment 的名字。</p><p>而当我访问<code>https://cafe.example.com:433/tea</code>的时候，则应该是tea这个Deployment负责响应我的请求（Server name: tea-7d57856c44-lwbnp），如下所示：</p><pre><code>$ curl --resolve cafe.example.com:$IC_HTTPS_PORT:$IC_IP https://cafe.example.com:$IC_HTTPS_PORT/tea --insecure
Server address: 10.244.1.58:80
Server name: tea-7d57856c44-lwbnp
Date: 03/Nov/2018:03:55:52 +0000
URI: /tea
Request ID: 32191f7ea07cb6bb44a1f43b8299415c
</code></pre><p>可以看到，Nginx Ingress Controller为我们创建的Nginx负载均衡器，已经成功地将请求转发给了对应的后端Service。</p><p>以上，就是Kubernetes里Ingress的设计思想和使用方法了。</p><p>不过，你可能会有一个疑问，<strong>如果我的请求没有匹配到任何一条IngressRule，那么会发生什么呢？</strong></p><p>首先，既然Nginx Ingress Controller是用Nginx实现的，那么它当然会为你返回一个 Nginx 的404页面。</p><p>不过，Ingress Controller也允许你通过Pod启动命令里的–default-backend-service参数，设置一条默认规则，比如：–default-backend-service=nginx-default-backend。</p><p>这样，任何匹配失败的请求，就都会被转发到这个名叫nginx-default-backend的Service。所以，你就可以通过部署一个专门的Pod，来为用户返回自定义的404页面了。</p><h2>总结</h2><p>在这篇文章里，我为你详细讲解了Ingress这个概念在Kubernetes里到底是怎么一回事儿。正如我在文章里所描述的，Ingress实际上就是Kubernetes对“反向代理”的抽象。</p><p>目前，Ingress只能工作在七层，而Service只能工作在四层。所以当你想要在Kubernetes里为应用进行TLS配置等HTTP相关的操作时，都必须通过Ingress来进行。</p><p>当然，正如同很多负载均衡项目可以同时提供七层和四层代理一样，将来Ingress的进化中，也会加入四层代理的能力。这样，一个比较完善的“反向代理”机制就比较成熟了。</p><p>而Kubernetes提出Ingress概念的原因其实也非常容易理解，有了Ingress这个抽象，用户就可以根据自己的需求来自由选择Ingress Controller。比如，如果你的应用对代理服务的中断非常敏感，那么你就应该考虑选择类似于Traefik这样支持“热加载”的Ingress Controller实现。</p><p>更重要的是，一旦你对社区里现有的Ingress方案感到不满意，或者你已经有了自己的负载均衡方案时，你只需要做很少的编程工作，就可以实现一个自己的Ingress Controller。</p><p>在实际的生产环境中，Ingress带来的灵活度和自由度，对于使用容器的用户来说，其实是非常有意义的。要知道，当年在Cloud Foundry项目里，不知道有多少人为了给Gorouter组件配置一个TLS而伤透了脑筋。</p><h2>思考题</h2><p>如果我的需求是，当访问<code>www.mysite.com</code>和 <code>forums.mysite.com</code>时，分别访问到不同的Service（比如：site-svc和forums-svc）。那么，这个Ingress该如何定义呢？请你描述出YAML文件中的rules字段。</p><p>感谢你的收听，欢迎你给我留言，也欢迎分享给更多的朋友一起阅读。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/94/47/75875257.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>虎虎❤️</span>
  </div>
  <div class="_2_QraFYR_0">思考题：<br>spec:<br>  rules:<br>  - host: www.mysite.com<br>    http:<br>      paths:<br>      - backend:<br>          serviceName: site-svc<br>          servicePort: 80<br>  - host: forums.mysite.com<br>    http:<br>      paths:<br>      - backend:<br>          serviceName: forums-svc<br>          servicePort: 80</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-21 16:49:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4c/49/d21c134f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>swordholder</span>
  </div>
  <div class="_2_QraFYR_0">总结一下，从集群外访问到服务，要经过3次代理：<br><br>访问请求到达任一宿主机，会根据NodePort生成的iptables规则，跳转到nginx反向代理，<br>请求再按照nginx的配置跳转到后台service，nginx的配置是根据Ingress对象生成的，<br>后台service也是iptables规则，最后跳转到真正提供服务的POD。<br><br>另外，如果想查看nginx-ingress-controller生成的nginx配置，可以这么做：<br><br>$ kubectl get pods -n ingress-nginx <br>NAME                                        READY   STATUS    RESTARTS   AGE<br>nginx-ingress-controller-85d94747dd-lsggm   1&#47;1     Running   0          3h45m<br><br>$ kubectl exec nginx-ingress-controller-85d94747dd-lsggm -it --namespace=&quot;ingress-nginx&quot; -- bash<br><br>$ cat &#47;etc&#47;nginx&#47;nginx.conf<br>...<br>$ exit</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-21 19:16:35</div>
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
  <div class="_2_QraFYR_0">Kubernets中通过Service来实现Pod实例之间的负载均衡和固定VIP的场景。<br><br>Service的工作原理是通过kube-proxy来设置宿主机上的iptables规则来实现的。kube-proxy来观察service的创建，然后通过修改本机的iptables规则，将访问Service VIP的请求转发到真正的Pod上。<br><br>基于iptables规则的Service实现，导致当宿主机上有大量的Pod的时候，成百上千条iptables规则不断刷新占用大量的CPU资源。因此出现了一种新的模式: IPVS，通过Linux的 IPVS模块将大量iptables规则放到了内核态，降低了维护这些规则的代价（部分辅助性的规则无法放到内核态，依旧是iptable形式）。<br><br>Service的DNS记录： &lt;myservice&gt;.&lt;mynamespace&gt;.svc.cluster.local ，当访问这条记录，返回的是Service的VIP或者是所代理Pod的IP地址的集合<br><br>Pod的DNS记录： &lt;pod_hostname&gt;.&lt;subdomain&gt;.&lt;mynamespace&gt;.svc.cluster.local， 注意pod的hostname和subdomain都是在Pod里定义的。<br><br>Service的访问依赖宿主机的kube-proxy生成的iptables规则以及kube-dns生成的DNS记录。外部的宿主机没有kube-proxy和kube-dns应该如何访问对应的Service呢？有以下几种方式：<br><br>NodePort： 比如外部client访问任意一台宿主机的8080端口，就是访问Service所代理Pod的80端口。由接收外部请求的宿主机做转发。<br><br>即：client --&gt; nodeIP:nodePort --&gt; serviceVIP:port --&gt; podIp:targetPort<br><br>LoadBalance：由公有云提供kubernetes服务自带的loadbalancer做负载均衡和外部流量访问的入口<br><br>ExternalName：通过ExternalName或ExternalIp给Service挂在一个公有IP的地址或者域名，当访问这个公有IP地址时，就会转发到Service所代理的Pod服务上<br><br>Ingress是用来给不同Service做负载均衡服务的，也就是Service的Service。<br><br><br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-05 22:33:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/c1/93/e8bfb26e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Dem</span>
  </div>
  <div class="_2_QraFYR_0">Nginx Ingress Controller的mandatory.yaml地址改掉了。现在的命令：<br>kubectl apply -f https:&#47;&#47;raw.githubusercontent.com&#47;kubernetes&#47;ingress-nginx&#47;master&#47;deploy&#47;static&#47;mandatory.yaml</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-12 14:00:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/43/e2/a1ff289c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>wang jl</span>
  </div>
  <div class="_2_QraFYR_0">文章里面这个nginx ingress只是api网关啊，负载均衡体现在哪里？其实还是service做的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-03 11:45:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/dd/6e/f3cfebc5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>峰哥</span>
  </div>
  <div class="_2_QraFYR_0">Ingress只支持7层的话，tcp协议的service怎么处理？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-16 18:44:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/ce/b2/1f914527.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>海盗船长</span>
  </div>
  <div class="_2_QraFYR_0">Nginx Ingress Controller的mandatory.yaml地址改掉了。现在的命令：<br>kubectl apply -f https:&#47;&#47;raw.githubusercontent.com&#47;kubernetes&#47;ingress-nginx&#47;controller-v0.34.1&#47;deploy&#47;static&#47;provider&#47;baremetal&#47;deploy.yaml</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-06 14:54:44</div>
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
  <div class="_2_QraFYR_0">感觉是因为nodeport之类的没有路由功能而出现的ingress，而且ingress还是需要由loadbalance之类的暴露给集群外部访问</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-22 10:15:13</div>
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
  <div class="_2_QraFYR_0">「但是，相信你也应该能感受到，由于每个 Service 都要有一个负载均衡服务，所以这个做法实际上既浪费成本又高。」没错，下周的 KubeCon 我会做一个关于共享 4 层 LoadBalancer 的session :)</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 听起来就很给力</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-05 13:46:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/33/ed/e6a6e60e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>busman</span>
  </div>
  <div class="_2_QraFYR_0">老师，我目前遇到个场景，就是一个非常耗资源的服务，需要容器化，部署在k8s集群中。目前问题就是这种大资源程序怎么打包(也是直接运行在一个容器里吗？)，如何调度？是否有通用方案。<br>这个程序大致是一个深度学习的算法，有tensorflow很耗cpu(监控来看，直接在一个容器里运行，能到十几核)，也会加载很大的模型(内存占用6G)，就这样的场景，老师能点拨一下吗？感谢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-21 21:28:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/61/b9/dbf629c0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Tao</span>
  </div>
  <div class="_2_QraFYR_0">apiVersion: extensions&#47;v1beta1<br>kind: Ingress<br>metadata:<br>  name: cafe-ingress<br>spec:<br>  tls:<br>  - hosts:<br>    - www.mysite.com<br>    - forums.mysite.com<br>    secretName: mysite-secret<br>  rules:<br>  - host: www.mysite.com<br>    http:<br>      paths:<br>      - path: &#47;mysite<br>        backend:<br>          serviceName: site-svc<br>          servicePort: 80<br>  - host: forums.mysite.com<br>    http:<br>      paths:<br>      - path: &#47;forums<br>        backend:<br>          serviceName: site-svc<br>          servicePort: 80<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-10 22:35:52</div>
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
  <div class="_2_QraFYR_0">请问ingress里配置的域名访问地址，那么集群外部怎么根据域名访问到k8s的服务呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-13 11:04:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>学习者</span>
  </div>
  <div class="_2_QraFYR_0">问题描述：前后端分离，但是同一域名下既有前端代码（部署在nginx中，以静态文件方式访问。），又用nginx 的proxy_pass ,根据url （location &#47;xx&#47; {}）反向代理做了后端。<br>方案 1：client -&gt; lb -&gt; ingress -&gt; nginx  (nginx 只部署前端，ingress 配置后端连接pod 的service，)<br>方案2 ： client -&gt; nginx  ( 后端采用nodeport暴露出来，采用问题描述方法部署)<br>请问这两种方案的可行性？或者还有什么更好的方案吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-24 14:10:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/ec/21/b0fe1bfd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Adam</span>
  </div>
  <div class="_2_QraFYR_0">感觉再加上一层ingress，又多了一层转发，性能上会不会损失比较大。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 多层lb的方案不是挺普遍的？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-10 14:54:57</div>
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
  <div class="_2_QraFYR_0">老师，为什么在创建 Ingress 所需的 SSL 证书（tls.crt）和密钥（tls.key）之后，使用curl命令还需要加上--insecure.<br>&quot;curl --resolve cafe.example.com:$IC_HTTPS_PORT:$IC_IP https:&#47;&#47;cafe.example.com:$IC_HTTPS_PORT&#47;tea --insecure&quot;<br><br>我的理解是：创建的SSL证书和密钥没有通过CA验证，所以要加上--secure. 对吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对的，假证书，哈哈</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-27 16:29:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/37/3b/495e2ce6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>陈斯佳</span>
  </div>
  <div class="_2_QraFYR_0">“Ingress是service的service” 太精辟了！一下就理解了ingress存在的意义！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-20 18:36:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/72/26/7387fc89.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>王天明</span>
  </div>
  <div class="_2_QraFYR_0">张老师，curl 那行代码，应该在“--insecureServer”中间换行了。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-23 20:32:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/59/93/aa3cb0fc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>火车飞侠</span>
  </div>
  <div class="_2_QraFYR_0">老师，如果我后端服务是https的，ingress如何定义呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-05 22:45:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/9b/6e/edd2da0c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>蓝魔丶</span>
  </div>
  <div class="_2_QraFYR_0">学习过程中遇到一些一直想不通的问题，麻烦能怎么联系老师解答下吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-24 10:41:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/77/3d/d998a34f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>zylv</span>
  </div>
  <div class="_2_QraFYR_0">ingress-controller 里面如果不配置域名，配置ip,可以吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 公有IP可以</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-21 23:11:41</div>
  </div>
</div>
</div>
</li>
</ul>