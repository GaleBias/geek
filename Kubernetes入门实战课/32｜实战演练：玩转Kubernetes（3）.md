<audio title="32｜实战演练：玩转Kubernetes（3）" src="https://static001.geekbang.org/resource/audio/6e/f2/6e8f32130cf704efd9362035f6f937f2.mp3" controls="controls"></audio> 
<p>你好，我是Chrono。</p><p>到今天，我们的“高级篇”课程也要结束了。比起前面的“初级篇”“中级篇”来说，这里的知识点比较多，难度也要高一些。如果你能够一篇不漏地学习下来，相信一定对Kubernetes有更深层次的认识和理解。</p><p>今天的这节课还是来对前面的知识做回顾与总结，提炼出文章里的学习要点和重点，你也可以顺便检验一下自己的掌握程度，试试在不回看课程的情况下，自己能不能流畅说出关联的操作细节。</p><p>复习之后，我们就来进行最后一次实战演练了。首先会继续改进贯穿课程始终的WordPress网站，把MariaDB改成StatefulSet，加上NFS持久化存储；然后我们会在Kubernetes集群里安装Dashboard，综合实践Ingress、namespace的用法。</p><h2>要点回顾一：API对象</h2><p>“高级篇”可以分成三个部分，第一部分讲的是PersistentVolume、StatefulSet等API对象。</p><p>（<a href="https://time.geekbang.org/column/article/542376">24讲</a>）<strong>PersistentVolume简称PV，是Kubernetes对持久化存储的抽象</strong>，代表了LocalDisk、NFS、Ceph等存储设备，和CPU、内存一样，属于集群的公共资源。</p><p>因为不同存储设备之间的差异很大，为了更好地描述PV特征，就出现了StorageClass，它的作用是分类存储设备，让我们更容易去选择PV对象。</p><!-- [[[read_end]]] --><p>PV一般由系统管理员来创建，我们如果要使用PV就要用PVC（PersistentVolumeClaim）去申请，说清楚需求的容量、访问模式等参数，然后Kubernetes就会查找最合适的PV分配给我们使用。</p><p>（<a href="https://time.geekbang.org/column/article/542458">25讲</a>）手动创建PV的工作量很大，麻烦而且容易出错，所以就有了“动态存储卷”的概念，需要<strong>在StorageClass里绑定一个Provisioner对象，由它来代替人工，根据PVC自动创建出符合要求的PV</strong>。</p><p>有了PV和PVC，我们就可以在Pod里用“persistentVolumeClaim”来引用PVC，创建出可供容器使用的Volume，然后在容器里用“volumeMounts”把它挂载到某个路径上，这样容器就可以读写PV，实现数据的持久化存储了。</p><p>（<a href="https://time.geekbang.org/column/article/547750">26讲</a>）持久化存储的一个重要应用领域就是保存应用的状态数据，<strong>管理有状态的应用，就要使用新的对象StatefulSet</strong>，可以认为它是管理无状态应用对象Deployment的一个特例。</p><p>StatefulSet对象的YAML描述和Deployment非常像，“spec”里只是多了一个“serviceName”字段，但它部署应用的方式却与Deployment差距很大。</p><p>Deployment创建的Pod是随机的名字，而StatefulSet会对Pod顺序编号、顺序创建，保证应用有一个确定的启动先后次序，这样就可以实现主从、主备等关系。</p><p>在使用Service为StatefulSet创建服务的时候，它也会为每个Pod单独创建域名，同样也是顺序编号，保证Pod有稳定的网络标识，外部用户就可以用这个域名来准确地访问到某个具体的Pod。</p><p>StatefulSet还使用“volumeClaimTemplates”字段来定义持久化存储，里面其实就是一个PVC，每个Pod可以用这个模板来生成自己的PVC去申请PV，实现存储卷与Pod的独立绑定。</p><p>通过<strong>启动顺序、稳定域名和存储模板</strong>这三个关键能力，StatefulSet就可以很好地处理Redis、MySQL等有状态应用了。</p><h2>要点回顾二：应用管理</h2><p>“高级篇”第二部分讲的是应用管理，包括滚动更新、资源配额和健康检查等内容。</p><p>（<a href="https://time.geekbang.org/column/article/547301">27讲</a>）在Kubernetes里部署好应用后，我们还需要对它做持续的运维管理，其中一项任务是版本的更新和回退。</p><p>版本更新很简单，只要编写一个新的YAML（Deployment、DaemonSet、StatefulSet），再用 <code>kubectl apply</code> 应用就可以了。Kubernetes采用的是<strong>“滚动更新”策略，实际上是两个同步进行的“扩容”和“缩容”动作</strong>，这样在更新的过程中始终会有Pod处于可用状态，能够平稳地对外提供服务。</p><p>应用的更新历史可以用命令 <code>kubectl rollout history</code> 查看，如果有什么意外，就可以用 <code>kubectl rollout undo</code> 来回退。这两个命令相当于给我们的更新流程上了一个保险，可以放心大胆操作，失败就用“S/L大法”。</p><p>（<a href="https://time.geekbang.org/column/article/548736">28讲</a>）为了让Pod里的容器能够稳定运行，我们可以采用<strong>资源配额</strong>和<strong>检查探针</strong>这两种手段。</p><p>资源配额能够限制容器申请的CPU和内存数量，不至于过多或者过少，保持在一个合理的程度，更有利于Kubernetes调度。</p><p>检查探针是Kubernetes内置的应用监控工具，有Startup、Liveness、Readiness三种，分别探测启动、存活、就绪状态，探测的方式也有exec、tcpSocket、httpGet三种。组合运用这些就可以灵活地检查容器的状态，Kubernetes发现不可用就会重启容器，让应用在总体上处于健康水平。</p><h2>要点回顾三：集群管理</h2><p>“高级篇”第三部分讲的是集群管理，有名字空间、系统监控和网络通信等知识点。</p><p>（<a href="https://time.geekbang.org/column/article/548750">29讲</a>）Kubernetes的集群里虽然有很多计算资源，但毕竟是有限的，除了要给Pod加上资源配额，我们也要为集群加上资源配额，方法就是用名字空间，把整体的资源池切分成多个小块，按需分配给不同的用户使用。</p><p>名字空间的资源配额使用的是“ResourceQuota”，除了基本的CPU和内存，它还能够限制存储容量和各种API对象的数量，这样就可以避免多用户互相挤占，更高效地利用集群资源。</p><p>（<a href="https://time.geekbang.org/column/article/550598">30讲</a>）系统监控是集群管理的另一个重要方面，Kubernetes提供了Metrics Server和Prometheus两个工具：</p><ul>
<li><strong>Metrics Server</strong>专门用来收集Kubernetes核心资源指标，可以用 <code>kubectl top</code> 来查看集群的状态，它也是水平自动伸缩对象HorizontalPodAutoscaler的前提条件。</li>
<li><strong>Prometheus</strong>，继Kubernetes之后的第二个CNCF毕业项目，是云原生监控领域的“事实标准”，在集群里部署之后就可以用Grafana可视化监控各种指标，还可以集成自动报警等功能。</li>
</ul><p>（<a href="https://time.geekbang.org/column/article/551711">31讲</a>）对于底层的基础网络设施，Kubernetes定义了平坦的网络模型“IP-per-pod”，实现它就要符合CNI标准。常用的网络插件有Flannel、Calico、Cilium等，Flannel使用Overlay模式，性能较低，Calico使用Route模式，性能较高。</p><p>现在，“高级篇”的众多知识要点我们都完整地过了一遍，你是否已经都理解、掌握了它们呢？</p><h2>搭建WordPress网站</h2><p>接下来我们就来在<a href="https://time.geekbang.org/column/article/539420">第22讲</a>的基础上继续优化WordPress网站，其中的关键是让数据库MariaDB实现数据持久化。</p><p>网站的整体架构图变化不大，前面的Nginx、WordPress还是原样，只需要修改MariaDB：</p><p><img src="https://static001.geekbang.org/resource/image/7c/1b/7cd3726d03ae12172b9073d1abf9fe1b.jpg?wh=1920x967" alt="图片"></p><p>因为MariaDB由Deployment改成了StatefulSet，所以我们要修改YAML，添加“serviceName”“volumeClaimTemplates”这两个字段，定义网络标识和NFS动态存储卷，然后在容器部分用“volumeMounts”挂载到容器里的数据目录“/var/lib/mysql”。</p><p>修改后的YAML就是这个样子：</p><pre><code class="language-yaml">apiVersion: apps/v1
kind: StatefulSet
metadata:
&nbsp; labels:
&nbsp; &nbsp; app: maria-sts
&nbsp; name: maria-sts

spec:
&nbsp; # headless svc
&nbsp; serviceName: maria-svc

&nbsp; # pvc
&nbsp; volumeClaimTemplates:
&nbsp; - metadata:
&nbsp; &nbsp; &nbsp; name: maria-100m-pvc
&nbsp; &nbsp; spec:
&nbsp; &nbsp; &nbsp; storageClassName: nfs-client
&nbsp; &nbsp; &nbsp; accessModes:
&nbsp; &nbsp; &nbsp; &nbsp; - ReadWriteMany
&nbsp; &nbsp; &nbsp; resources:
&nbsp; &nbsp; &nbsp; &nbsp; requests:
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; storage: 100Mi

&nbsp; replicas: 1
&nbsp; selector:
&nbsp; &nbsp; matchLabels:
&nbsp; &nbsp; &nbsp; app: maria-sts

&nbsp; template:
&nbsp; &nbsp; metadata:
&nbsp; &nbsp; &nbsp; labels:
&nbsp; &nbsp; &nbsp; &nbsp; app: maria-sts
&nbsp; &nbsp; spec:
&nbsp; &nbsp; &nbsp; containers:
&nbsp; &nbsp; &nbsp; - image: mariadb:10
&nbsp; &nbsp; &nbsp; &nbsp; name: mariadb
&nbsp; &nbsp; &nbsp; &nbsp; imagePullPolicy: IfNotPresent
&nbsp; &nbsp; &nbsp; &nbsp; ports:
&nbsp; &nbsp; &nbsp; &nbsp; - containerPort: 3306

&nbsp; &nbsp; &nbsp; &nbsp; envFrom:
&nbsp; &nbsp; &nbsp; &nbsp; - prefix: 'MARIADB_'
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; configMapRef:
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; name: maria-cm

&nbsp; &nbsp; &nbsp; &nbsp; volumeMounts:
&nbsp; &nbsp; &nbsp; &nbsp; - name: maria-100m-pvc
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; mountPath: /var/lib/mysql
</code></pre><p>改完MariaDB，我们还要再对WordPress做一点小修改。</p><p>还记得吗？StatefulSet管理的每个Pod都有自己的域名，所以要把WordPress的环境变量改成MariaDB的新名字，也就是“<strong>maria-sts-0.maria-svc</strong>”：</p><pre><code class="language-yaml">apiVersion: v1
kind: ConfigMap
metadata:
&nbsp; name: wp-cm

data:
&nbsp; HOST: 'maria-sts-0.maria-svc'  #注意这里
&nbsp; USER: 'wp'
&nbsp; PASSWORD: '123'
&nbsp; NAME: 'db'
</code></pre><p>改完这两个YAML，我们就可以逐个创建MariaDB、WordPress、Ingress等对象了。</p><p>和之前一样，访问NodePort的“30088”端口，或者是用Ingress Controller的“wp.test”域名，都可以进入WordPress网站：</p><p><img src="https://static001.geekbang.org/resource/image/fc/46/fc3b52f96f138f01b23e3a7487730746.png?wh=916x1166" alt="图片"></p><p>StatefulSet的持久化存储是否生效了呢？</p><p>你可以把这些对象都删除后重新创建，再进入网站，看看是否原来的数据依然存在。或者更简单一点，直接查看NFS的存储目录，应该可以看到MariaDB生成的一些数据库文件：</p><p><img src="https://static001.geekbang.org/resource/image/42/18/428886b77e4797dc7ded5a43yyc0b218.png?wh=1920x124" alt="图片"></p><p>这两种方式都能够证明，我们的MariaDB使用StatefulSet部署后数据已经保存在了磁盘上，不会因为对象的销毁而丢失。</p><p>到这里，第一个小实践你就已经完成了，给自己鼓鼓劲，我们一起来做第二个实践，在Kubernetes集群里安装Dashboard。</p><h2>部署Dashboard</h2><p>在“初级篇”的实战演练课里（<a href="https://time.geekbang.org/column/article/534644">第15讲</a>），我简单介绍了Kubernetes的图形管理界面，也就是Dashboard，不知道你是否还有印象。当时Dashboard是直接内置在minikube里的，不需要安装，一个命令启动，就能在浏览器里直观地管理Kubernetes集群了，非常方便。</p><p>那现在我们用kubeadm部署了实际的多节点集群，能否也用上Dashboard呢？接下来我就带你来一起动手，从零开始安装Dashboard。</p><p>首先，你应该先去Dashboard的项目网站（<a href="https://github.com/kubernetes/dashboard">https://github.com/kubernetes/dashboard</a>），看一下它的说明文档，了解一下它的基本情况。</p><p>它的安装很简单，只需要一个YAML文件，可以直接下载：</p><pre><code class="language-plain">wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.6.0/aio/deploy/recommended.yaml
</code></pre><p>这个YAML里包含了很多对象，虽然文件比较大，但现在的你应该基本都能够看懂了，要点有这么几个：</p><ul>
<li>所有的对象都属于“kubernetes-dashboard”名字空间。</li>
<li>Dashboard使用Deployment部署了一个实例，端口号是8443。</li>
<li>容器启用了Liveness探针，使用HTTPS方式检查存活状态。</li>
<li>Service对象使用的是443端口，它映射了Dashboard的8443端口。</li>
</ul><p>使用命令 <code>kubectl apply</code> 就可以轻松部署Dashboard了：</p><pre><code class="language-plain">kubectl apply -f dashboard.yaml
</code></pre><p><img src="https://static001.geekbang.org/resource/image/c5/79/c56f8936e187047a2b7d100f7ae0f779.png?wh=1586x250" alt="图片"></p><h2>部署Ingress/Ingress Controller</h2><p>不过，为了给我们的实战增加一点难度，我们可以在前面配一个Ingress入口，用反向代理的方式来访问它。</p><p>由于Dashboard默认使用的是加密的HTTPS协议，拒绝明文HTTP访问，所以我们要先生成证书，让Ingress也走HTTPS协议。</p><p>简单起见，我直接用Linux里的命令行工具“openssl”来生成一个自签名的证书（如果你有条件，也可以考虑找CA网站申请免费证书）：</p><pre><code class="language-plain">openssl req -x509 -days 365 -out k8s.test.crt -keyout k8s.test.key \
&nbsp; -newkey rsa:2048 -nodes -sha256 \
&nbsp; &nbsp; -subj '/CN=k8s.test' -extensions EXT -config &lt;( \
&nbsp; &nbsp; &nbsp; &nbsp;printf "[dn]\nCN=k8s.test\n[req]\ndistinguished_name = dn\n[EXT]\nsubjectAltName=DNS:k8s.test\nkeyUsage=digitalSignature\nextendedKeyUsage=serverAuth")
</code></pre><p>openssl的命令比较长，我简单解释一下：它生成的是一个X509格式的证书，有效期365天，私钥是RSA2048位，摘要算法是SHA256，签发的网站是“k8s.test”。</p><p>运行命令行后会生成两个文件，一个是证书“k8s.test.crt”，另一个是私钥“k8s.test.key”，我们需要把这两个文件存入Kubernetes里供Ingress使用。</p><p>因为这两个文件属于机密信息，存储的方式当然就是用Secret了。你仍然可以用命令 <code>kubectl create secret</code> 来自动创建YAML，不过类型不是“generic”，而是“tls”，同时还要用 <code>-n</code> 指定名字空间，用 <code>--cert</code>、<code>--key</code> 指定文件：</p><pre><code class="language-plain">export out="--dry-run=client -o yaml"
kubectl create secret tls dash-tls -n kubernetes-dashboard --cert=k8s.test.crt --key=k8s.test.key $out &gt; cert.yml
</code></pre><p>出来的YAML大概是这个样子：</p><pre><code class="language-yaml">apiVersion: v1
kind: Secret
metadata:
&nbsp; name: dash-tls
&nbsp; namespace: kubernetes-dashboard
type: kubernetes.io/tls

data:
&nbsp; tls.crt: LS0tLS1CRUdJTiBDRVJU...
  tls.key: LS0tLS1CRUdJTiBQUklW...
</code></pre><p>创建这个Secret对象之后，你可以再用 <code>kubectl describe</code> 来检查它的状态：</p><p><img src="https://static001.geekbang.org/resource/image/26/cc/2615d5c6c3yy704cc63c5bf6df5b87cc.png?wh=1904x716" alt="图片"></p><p>接下来我们就来编写Ingress Class和Ingress对象，为了保持名字空间的整齐，也把它放在“kubernetes-dashboard”名字空间里。</p><p>Ingress Class对象很简单，名字是“dash-ink”，指定Controller还是我们之前用的Nginx官方的Ingress Controller：</p><pre><code class="language-yaml">apiVersion: networking.k8s.io/v1
kind: IngressClass

metadata:
&nbsp; name: dash-ink
&nbsp; namespace: kubernetes-dashboard
spec:
&nbsp; controller: nginx.org/ingress-controller
</code></pre><p>Ingress对象可以用 <code>kubectl create</code> 命令自动生成，如果你有点忘记的话，可以回头参考一下<a href="https://time.geekbang.org/column/article/538760">第21讲</a>：</p><pre><code class="language-plain">kubectl create ing dash-ing --rule="k8s.test/=kubernetes-dashboard:443" --class=dash-ink -n kubernetes-dashboard $out
</code></pre><p>但这次因为是HTTPS协议，所以我们要在Ingress里多加一点东西，一个是“<strong>annotations</strong>”字段，指定后端目标是HTTPS服务，另一个是“<strong>tls</strong>”字段，指定域名和证书，也就是刚才创建的Secret：</p><pre><code class="language-yaml">apiVersion: networking.k8s.io/v1
kind: Ingress

metadata:
&nbsp; name: dash-ing
&nbsp; namespace: kubernetes-dashboard
&nbsp; annotations:
&nbsp; &nbsp; nginx.org/ssl-services: "kubernetes-dashboard"

spec:
&nbsp; ingressClassName: dash-ink

&nbsp; tls:
&nbsp; &nbsp; - hosts:
&nbsp; &nbsp; &nbsp; - k8s.test
&nbsp; &nbsp; &nbsp; secretName: dash-tls

&nbsp; rules:
&nbsp; - host: k8s.test
&nbsp; &nbsp; http:
&nbsp; &nbsp; &nbsp; paths:
&nbsp; &nbsp; &nbsp; - path: /
&nbsp; &nbsp; &nbsp; &nbsp; pathType: Prefix
&nbsp; &nbsp; &nbsp; &nbsp; backend:
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; service:
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; name: kubernetes-dashboard
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; port:
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; number: 443
</code></pre><p>最后一个对象，就是Ingress Controller了，还是拿现成的模板修改，记得要把“args”里的Ingress Class改成我们自己的“dash-ink”：</p><pre><code class="language-yaml">apiVersion: apps/v1
kind: Deployment
metadata:
&nbsp; name: dash-kic-dep
&nbsp; namespace: nginx-ingress

spec:
  ...
&nbsp; &nbsp; &nbsp; &nbsp; args:
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; - -ingress-class=dash-ink
</code></pre><p>要让我们在外面能够访问Ingress Controller，还要为它再定义一个Service，类型是“NodePort”，端口指定是“30443”：</p><pre><code class="language-yaml">apiVersion: v1
kind: Service
metadata:
&nbsp; name: dash-kic-svc
&nbsp; namespace: nginx-ingress

spec:
&nbsp; ports:
&nbsp; - port: 443
&nbsp; &nbsp; protocol: TCP
&nbsp; &nbsp; targetPort: 443
&nbsp; &nbsp; nodePort: 30443

&nbsp; selector:
&nbsp; &nbsp; app: dash-kic-dep
&nbsp; type: NodePort
</code></pre><p>把上面的Secret、Ingress Class、Ingress、Ingress Controller、Service都创建好之后，我们再来确认一下它们的运行状态：</p><p><img src="https://static001.geekbang.org/resource/image/40/b2/4062d4e5c8c57f74a480ee21ca3717b2.png?wh=1920x821" alt="图片"></p><p>因为这些对象比较多，处于不同的名字空间，关联有点复杂，我画了一个简单的示意图，你可以看一下：</p><p><img src="https://static001.geekbang.org/resource/image/b7/50/b720648a0fefab28fa940b7cd6afb350.jpg?wh=1920x793" alt="图片"></p><h2>访问Dashboard</h2><p>到这里，Dashboard的部署工作就基本完成了。为了能正常访问，我们还要为它创建一个用户，才能登录进Dashboard。</p><p>Dashboard的网站上有一个简单示例（<a href="https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md">https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/creating-sample-user.md</a>），我们直接拿来用就行：</p><pre><code class="language-yaml">apiVersion: v1
kind: ServiceAccount
metadata:
&nbsp; name: admin-user
&nbsp; namespace: kubernetes-dashboard

---

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
&nbsp; name: admin-user
roleRef:
&nbsp; apiGroup: rbac.authorization.k8s.io
&nbsp; kind: ClusterRole
&nbsp; name: cluster-admin
subjects:
- kind: ServiceAccount
&nbsp; name: admin-user
&nbsp; namespace: kubernetes-dashboard
</code></pre><p>这个YAML创建了一个Dashboard的管理员账号，名字叫“admin-user”，使用的是Kubernetes的RBAC机制，就不展开细讲了。</p><p>这个账号不能用简单的“用户名+密码”的方式登录，需要用到一个Token，可以用 <code>kubectl get secret</code>、<code>kubectl describe secret</code> 查到：</p><pre><code class="language-plain">kubectl get secret -n kubernetes-dashboard
kubectl describe secrets -n kubernetes-dashboard admin-user-token-xxxx
</code></pre><p><img src="https://static001.geekbang.org/resource/image/0f/yy/0ffd4627b0efa2ba5774bf5c65faa1yy.png?wh=1920x1051" alt="图片"></p><p>Token是一个很长的字符串，把它拷贝存好，再为它的测试域名“k8s.test”加上域名解析（修改/etc/hosts），然后我们就可以在浏览器里输入网址“<a href="https://k8s.test:30443">https://k8s.test:30443</a>”访问Dashboard了：</p><p><img src="https://static001.geekbang.org/resource/image/c8/5d/c83cd71ab4d6696f5b837ea20056ff5d.png?wh=1920x958" alt="图片"></p><p>下面的两张截图就是我查看集群里“kube-system”名字空间的情况，由于我们之前安装了Metrics Server，所以Dashboard也能够以图形的方式显示CPU和内存状态，有那么一点Prometheus + Grafana的意思：</p><p><img src="https://static001.geekbang.org/resource/image/3c/d9/3ca6e156150a6a06477bb2eb07e00cd9.png?wh=1920x1243" alt="图片"></p><p><img src="https://static001.geekbang.org/resource/image/fa/b2/fae2168c30677d2370e8e71c3d98f1b2.png?wh=1920x1243" alt="图片"></p><h2>小结</h2><p>好了，今天我们一起回顾了“高级篇”里的要点，下面的这张思维导图就是对这些知识点的全面总结，你可以再认真研究一下：</p><p><img src="https://static001.geekbang.org/resource/image/4a/30/4a9bb79b2e54096yyf5c5799837dd930.jpg?wh=1920x1312" alt="图片"></p><p>今天我们有两个实战项目。首先是WordPress，把后端的存储服务MariaDB改造成了StatefulSet，挂载了NFS网盘，这样就实现了一个功能比较完善的网站，达到了基本可用的程度。</p><p>接着我们又在Kubernetes里安装了Dashboard，主要部署在名字空间“kubernetes-dashboard”。Dashboard自身的安装很简单，但我们又为它在前面搭建了一个反向代理，配上了安全证书，进一步实践了Ingress的用法。</p><p>不过这两个项目还没有完全覆盖“高级篇”的内容，你可以再接着改进它们，比如加上健康检查、资源配额、自动水平伸缩等，多动手来巩固所学的知识。</p><h2>课下作业</h2><p>今天的课下作业时间，我想就留给你自己来操作一下这节课里的两个实战演练吧，如果遇到了什么问题，可以在留言区随时提问。</p><p><img src="https://static001.geekbang.org/resource/image/aa/9f/aa71ca9df15c8141f3368cce8b41dc9f.jpg?wh=1920x2358" alt="图片"></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2f/85/6f/1654f4b9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>nc_ops</span>
  </div>
  <div class="_2_QraFYR_0">老师。“还是拿现成的模板修改”，模板在哪里？没找到。是在你发的dashboard项目网站里吗？模板名字是啥？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以参考之前讲Ingress那节课，里面有Ingress Controller的yaml示例。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-16 16:05:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/oib0a89lqtOhJL1UvfUp4uTsRLrDbhoGk9jLiciazxMu0COibJsFCZDypK1ZFcHEJc9d9qgbjvgR41ImL6FNPoVlWA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>stefen</span>
  </div>
  <div class="_2_QraFYR_0">如果能带主从的mariadb去部署wordpress就比较完美一些.</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 主从部署比较麻烦，其实和Kubernetes关系不大，我不是专门做数据库的，这方面就只能藏拙了，如果有心得欢迎分享。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-12 22:03:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/95/af/b7f8dc43.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>拓山</span>
  </div>
  <div class="_2_QraFYR_0">k8s.test需要再kubetcl里配置吗？<br>这个点很困惑</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: k8s.test是一个域名，需要在使用浏览器的电脑上配置域名解析（如&#47;etc&#47;hosts），然后我们才能用浏览器通过域名来访问到kubernetes。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-17 15:14:03</div>
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
  <div class="_2_QraFYR_0">在部署statefulset管理的maria pod时，不要忘了创建service对象，不然`maria-sts-0.maria-svc`是无效的，有可能报Error establishing a database connection<br><br>maria-svc.yml内容如下：<br>apiVersion: v1<br>kind: Service<br>metadata:<br>  name: maria-svc<br><br>spec:<br>  selector:<br>    app: maria-sts<br><br>  ports:<br>  - port: 3306<br>    protocol: TCP<br>    targetPort: 3306</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-14 18:31:08</div>
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
  <div class="_2_QraFYR_0">老师，按照流程，最后通过Ingress是可以访问到（https）的dashboard页面，但是页面是为空，看title以及页面源码确实有dashboard字样，但是没有具体内容，显示为空白页面。<br><br>会是什么原因导致的呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 先不用Ingress，看看直接访问dashboard是否正常，然后再看看是否Ingress配置的哪里有问题。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-26 17:05:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_674ea8</span>
  </div>
  <div class="_2_QraFYR_0">老师，为dashboard配置ingress时，配置好后还是无法通过浏览器使用域名访问（已在hosts添加），浏览器报503：gateway time-out，查看ingress-controller日志显示如下：<br>Host is unreachable) while connecting to upstream, client: 10.10.1.1, server: k8s.test, request: &quot;GET &#47; HTTP&#47;2.0&quot;, upstream: &quot;https:&#47;&#47;10.10.0.19:8443&#47;&quot;<br>其中upstream的地址 10.10.0.19为kubernetes-dashbord的pod地址。<br>请问这种问题是什么原因造成的啊？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 看日志，503是nginx到后端出现错误，可能是kubernetes网络的问题，访问不了Pod，可以先不用Ingress controller，先用NodePort方式试试，再找找中间环节的问题。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-03 17:00:42</div>
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
  <div class="_2_QraFYR_0">为什么kubernetes-dashboard的那些对象要处于2个不同的名字空间呢？有什么用吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 没什么特别的意义，也可以放在一个名字空间里，分成两个主要是和之前的做法保持一致。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-16 16:31:18</div>
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
  <div class="_2_QraFYR_0">分享我遇到的问题：<br>1. 搭建 dashboard ，访问时一直有这个错误 “Client sent an HTTP request to an HTTPS server”，原因是 ingress 没有加上 nginx.org&#47;ssl-services annotation（老师已经提醒了）<br>参考文档 https:&#47;&#47;docs.nginx.com&#47;nginx-ingress-controller&#47;configuration&#47;ingress-resources&#47;advanced-configuration-with-annotations&#47;#backend-services-upstreams<br><br>2. 搭建 wordpress 时，ingress 有端口号，浏览器打开页面无法正常显示，比如主页 https:&#47;&#47;wp.test:30443&#47; ，加载页面资源时会变成 https:&#47;&#47;wp.test&#47;xxxx ，丢失了端口号。这个问题不知道该如何解，请老师&#47;同学帮忙解答，谢谢！<br>（为了验证自己的想法，手动去改了 ingress controller pod 里 nginx 配置，强制设置“ proxy_set_header Host $host:430443; ”，可以凑效。）</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1. Ingress的配置比较复杂，需要多看它的文档。<br><br>2. Ingress代理WordPress的时候比较麻烦，因为它的重定向不太好配置，这个我不是太了解，有更好的方法欢迎分享。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-06 13:50:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/8b/06/fb3be14a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>TableBear</span>
  </div>
  <div class="_2_QraFYR_0">请教老师一个问题：<br>文章后面的给Dashboard绑定Ingrss的时候，IngressClass的名字空间是kubernetes-dashboard，IngressController的名字空间是nginx-ingress。IngressClass和IngressConroller绑定的时候是不是忽略名字空间的？那是不是意味着IngressClass的name必须全局唯一？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个可能得试验一下，我个人对这个也是感觉比较奇怪，可能得从kic的源码里找答案了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-08 19:31:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/33/27/e5a74107.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Da Vinci</span>
  </div>
  <div class="_2_QraFYR_0">按照老师的课程步骤去部署dashboard，但是最后访问的时候返回404，不知道为啥</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可能是Ingress没有配置好，路由不对，把各个环节的YAML 都检查一下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-06 15:35:26</div>
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
  <div class="_2_QraFYR_0">请教老师几个问题：<br>Q1：Prometheus和Skywalking是一类产品吗？<br>Q2：对于常见的CI&#47;CD(持续集成、部署)，k8s等同于CD吗？<br>Q3：用openssl”生成的自签名证书，可以应用于生成环境吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1. Prometheus是应用指标监控，源自BorgMon，Skywalking是全链路追踪，源自Dapper，不是同一个领域。<br><br>2.Kubernetes是一个基础平台，可以在上面运行CI&#47;CD，两者完全不一样。<br><br>3.内部用没问题，自己签名自己信任，对外可以用let&#39;s encrypt。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-06 08:48:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/23/98/ad/f9d755f2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>邓嘉文</span>
  </div>
  <div class="_2_QraFYR_0">第一</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-05 15:02:26</div>
  </div>
</div>
</div>
</li>
</ul>