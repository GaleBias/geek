<audio title="26 _ 基于角色的权限控制：RBAC" src="https://static001.geekbang.org/resource/audio/d8/db/d88c2fed2e06621a4af161aa391a54db.mp3" controls="controls"></audio> 
<p>你好，我是张磊。今天我和你分享的主题是：基于角色的权限控制之RBAC。</p><p>在前面的文章中，我已经为你讲解了很多种Kubernetes内置的编排对象，以及对应的控制器模式的实现原理。此外，我还剖析了自定义API资源类型和控制器的编写方式。</p><p>这时候，你可能已经冒出了这样一个想法：控制器模式看起来好像也不难嘛，我能不能自己写一个编排对象呢？</p><p>答案当然是可以的。而且，这才是Kubernetes项目最具吸引力的地方。</p><p>毕竟，在互联网级别的大规模集群里，Kubernetes内置的编排对象，很难做到完全满足所有需求。所以，很多实际的容器化工作，都会要求你设计一个自己的编排对象，实现自己的控制器模式。</p><p>而在Kubernetes项目里，我们可以基于插件机制来完成这些工作，而完全不需要修改任何一行代码。</p><p>不过，你要通过一个外部插件，在Kubernetes里新增和操作API对象，那么就必须先了解一个非常重要的知识：RBAC。</p><p>我们知道，Kubernetes中所有的API对象，都保存在Etcd里。可是，对这些API对象的操作，却一定都是通过访问kube-apiserver实现的。其中一个非常重要的原因，就是你需要APIServer来帮助你做授权工作。</p><!-- [[[read_end]]] --><p>而<strong>在Kubernetes项目中，负责完成授权（Authorization）工作的机制，就是RBAC</strong>：基于角色的访问控制（Role-Based Access Control）。</p><p>如果你直接查看Kubernetes项目中关于RBAC的文档的话，可能会感觉非常复杂。但实际上，等到你用到这些RBAC的细节时，再去查阅也不迟。</p><p>而在这里，我只希望你能明确三个最基本的概念。</p><ol>
<li>
<p>Role：角色，它其实是一组规则，定义了一组对Kubernetes API对象的操作权限。</p>
</li>
<li>
<p>Subject：被作用者，既可以是“人”，也可以是“机器”，也可以是你在Kubernetes里定义的“用户”。</p>
</li>
<li>
<p>RoleBinding：定义了“被作用者”和“角色”的绑定关系。</p>
</li>
</ol><p>而这三个概念，其实就是整个RBAC体系的核心所在。</p><p><span class="orange">我先来讲解一下Role。</span></p><p>实际上，Role本身就是一个Kubernetes的API对象，定义如下所示：</p><pre><code>kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: mynamespace
  name: example-role
rules:
- apiGroups: [&quot;&quot;]
  resources: [&quot;pods&quot;]
  verbs: [&quot;get&quot;, &quot;watch&quot;, &quot;list&quot;]
</code></pre><p>首先，这个Role对象指定了它能产生作用的Namepace是：mynamespace。</p><p>Namespace是Kubernetes项目里的一个逻辑管理单位。不同Namespace的API对象，在通过kubectl命令进行操作的时候，是互相隔离开的。</p><p>比如，kubectl get pods -n mynamespace。</p><p>当然，这仅限于逻辑上的“隔离”，Namespace并不会提供任何实际的隔离或者多租户能力。而在前面文章中用到的大多数例子里，我都没有指定Namespace，那就是使用的是默认Namespace：default。</p><p>然后，这个Role对象的rules字段，就是它所定义的权限规则。在上面的例子里，这条规则的含义就是：允许“被作用者”，对mynamespace下面的Pod对象，进行GET、WATCH和LIST操作。</p><p>那么，<span class="orange">这个具体的“被作用者”又是如何指定的呢？这就需要通过RoleBinding来实现了。</span></p><p>当然，RoleBinding本身也是一个Kubernetes的API对象。它的定义如下所示：</p><pre><code>kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: example-rolebinding
  namespace: mynamespace
subjects:
- kind: User
  name: example-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: example-role
  apiGroup: rbac.authorization.k8s.io
</code></pre><p>可以看到，这个RoleBinding对象里定义了一个subjects字段，即“被作用者”。它的类型是User，即Kubernetes里的用户。这个用户的名字是example-user。</p><p>可是，在Kubernetes中，其实并没有一个叫作“User”的API对象。而且，我们在前面和部署使用Kubernetes的流程里，既不需要User，也没有创建过User。</p><p><strong>这个User到底是从哪里来的呢？</strong></p><p>实际上，Kubernetes里的“User”，也就是“用户”，只是一个授权系统里的逻辑概念。它需要通过外部认证服务，比如Keystone，来提供。或者，你也可以直接给APIServer指定一个用户名、密码文件。那么Kubernetes的授权系统，就能够从这个文件里找到对应的“用户”了。当然，在大多数私有的使用环境中，我们只要使用Kubernetes提供的内置“用户”，就足够了。这部分知识，我后面马上会讲到。</p><p>接下来，我们会看到一个roleRef字段。正是通过这个字段，<span class="orange">RoleBinding对象就可以直接通过名字，来引用我们前面定义的Role对象（example-role），从而定义了“被作用者（Subject）”和“角色（Role）”之间的绑定关系。</span></p><p>需要再次提醒的是，Role和RoleBinding对象都是Namespaced对象（Namespaced Object），它们对权限的限制规则仅在它们自己的Namespace内有效，roleRef也只能引用当前Namespace里的Role对象。</p><p>那么，<strong>对于非Namespaced（Non-namespaced）对象（比如：Node），或者，某一个Role想要作用于所有的Namespace的时候，我们又该如何去做授权呢？</strong></p><p>这时候，我们就必须要使用ClusterRole和ClusterRoleBinding这两个组合了。这两个API对象的用法跟Role和RoleBinding完全一样。只不过，它们的定义里，没有了Namespace字段，如下所示：</p><pre><code>kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: example-clusterrole
rules:
- apiGroups: [&quot;&quot;]
  resources: [&quot;pods&quot;]
  verbs: [&quot;get&quot;, &quot;watch&quot;, &quot;list&quot;]
</code></pre><pre><code>kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: example-clusterrolebinding
subjects:
- kind: User
  name: example-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: example-clusterrole
  apiGroup: rbac.authorization.k8s.io
</code></pre><p>上面的例子里的ClusterRole和ClusterRoleBinding的组合，意味着名叫example-user的用户，拥有对所有Namespace里的Pod进行GET、WATCH和LIST操作的权限。</p><p>更进一步地，在Role或者ClusterRole里面，如果要赋予用户example-user所有权限，那你就可以给它指定一个verbs字段的全集，如下所示：</p><pre><code>verbs: [&quot;get&quot;, &quot;list&quot;, &quot;watch&quot;, &quot;create&quot;, &quot;update&quot;, &quot;patch&quot;, &quot;delete&quot;]
</code></pre><p>这些就是当前Kubernetes（v1.11）里能够对API对象进行的所有操作了。</p><p>类似地，Role对象的rules字段也可以进一步细化。比如，你可以只针对某一个具体的对象进行权限设置，如下所示：</p><pre><code>rules:
- apiGroups: [&quot;&quot;]
  resources: [&quot;configmaps&quot;]
  resourceNames: [&quot;my-config&quot;]
  verbs: [&quot;get&quot;]
</code></pre><p>这个例子就表示，这条规则的“被作用者”，只对名叫“my-config”的ConfigMap对象，有进行GET操作的权限。</p><p>而正如我前面介绍过的，在大多数时候，我们其实都不太使用“用户”这个功能，而是直接使用Kubernetes里的“内置用户”。</p><p><span class="orange">这个由Kubernetes负责管理的“内置用户”，正是我们前面曾经提到过的：ServiceAccount。</span></p><p>接下来，我通过一个具体的实例来为你讲解一下为ServiceAccount分配权限的过程。</p><p><strong>首先，我们要定义一个ServiceAccount</strong>。它的API对象非常简单，如下所示：</p><pre><code>apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: mynamespace
  name: example-sa
</code></pre><p>可以看到，一个最简单的ServiceAccount对象只需要Name和Namespace这两个最基本的字段。</p><p><strong>然后，我们通过编写RoleBinding的YAML文件，来为这个ServiceAccount分配权限：</strong></p><pre><code>kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: example-rolebinding
  namespace: mynamespace
subjects:
- kind: ServiceAccount
  name: example-sa
  namespace: mynamespace
roleRef:
  kind: Role
  name: example-role
  apiGroup: rbac.authorization.k8s.io
</code></pre><p>可以看到，在这个RoleBinding对象里，subjects字段的类型（kind），不再是一个User，而是一个名叫example-sa的ServiceAccount。而roleRef引用的Role对象，依然名叫example-role，也就是我在这篇文章一开始定义的Role对象。</p><p><strong>接着，我们用kubectl命令创建这三个对象：</strong></p><pre><code>$ kubectl create -f svc-account.yaml
$ kubectl create -f role-binding.yaml
$ kubectl create -f role.yaml
</code></pre><p>然后，我们来查看一下这个ServiceAccount的详细信息：</p><pre><code>$ kubectl get sa -n mynamespace -o yaml
- apiVersion: v1
  kind: ServiceAccount
  metadata:
    creationTimestamp: 2018-09-08T12:59:17Z
    name: example-sa
    namespace: mynamespace
    resourceVersion: &quot;409327&quot;
    ...
  secrets:
  - name: example-sa-token-vmfg6
</code></pre><p>可以看到，Kubernetes会为一个ServiceAccount自动创建并分配一个Secret对象，即：上述ServiceAcount定义里最下面的secrets字段。</p><p>这个Secret，就是这个ServiceAccount对应的、用来跟APIServer进行交互的授权文件，我们一般称它为：Token。Token文件的内容一般是证书或者密码，它以一个Secret对象的方式保存在Etcd当中。</p><p>这时候，用户的Pod，就可以声明使用这个ServiceAccount了，比如下面这个例子：</p><pre><code>apiVersion: v1
kind: Pod
metadata:
  namespace: mynamespace
  name: sa-token-test
spec:
  containers:
  - name: nginx
    image: nginx:1.7.9
  serviceAccountName: example-sa
</code></pre><p>在这个例子里，我定义了Pod要使用的要使用的ServiceAccount的名字是：example-sa。</p><p>等这个Pod运行起来之后，我们就可以看到，该ServiceAccount的token，也就是一个Secret对象，被Kubernetes自动挂载到了容器的/var/run/secrets/kubernetes.io/serviceaccount目录下，如下所示：</p><pre><code>$ kubectl describe pod sa-token-test -n mynamespace
Name:               sa-token-test
Namespace:          mynamespace
...
Containers:
  nginx:
    ...
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from example-sa-token-vmfg6 (ro)
</code></pre><p>这时候，我们可以通过kubectl exec查看到这个目录里的文件：</p><pre><code>$ kubectl exec -it sa-token-test -n mynamespace -- /bin/bash
root@sa-token-test:/# ls /var/run/secrets/kubernetes.io/serviceaccount
ca.crt namespace  token
</code></pre><p>如上所示，容器里的应用，就可以使用这个ca.crt来访问APIServer了。更重要的是，此时它只能够做GET、WATCH和LIST操作。因为example-sa这个ServiceAccount的权限，已经被我们绑定了Role做了限制。</p><p>此外，我在第15篇文章<a href="https://time.geekbang.org/column/article/40466">《深入解析Pod对象（二）：使用进阶》</a>中曾经提到过，如果一个Pod没有声明serviceAccountName，Kubernetes会自动在它的Namespace下创建一个名叫default的默认ServiceAccount，然后分配给这个Pod。</p><p>但在这种情况下，这个默认ServiceAccount并没有关联任何Role。也就是说，此时它有访问APIServer的绝大多数权限。当然，这个访问所需要的Token，还是默认ServiceAccount对应的Secret对象为它提供的，如下所示。</p><pre><code>$kubectl describe sa default
Name:                default
Namespace:           default
Labels:              &lt;none&gt;
Annotations:         &lt;none&gt;
Image pull secrets:  &lt;none&gt;
Mountable secrets:   default-token-s8rbq
Tokens:              default-token-s8rbq
Events:              &lt;none&gt;

$ kubectl get secret
NAME                  TYPE                                  DATA      AGE
kubernetes.io/service-account-token   3         82d

$ kubectl describe secret default-token-s8rbq
Name:         default-token-s8rbq
Namespace:    default
Labels:       &lt;none&gt;
Annotations:  kubernetes.io/service-account.name=default
              kubernetes.io/service-account.uid=ffcb12b2-917f-11e8-abde-42010aa80002

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  7 bytes
token:      &lt;TOKEN数据&gt;
</code></pre><p>可以看到，Kubernetes会自动为默认ServiceAccount创建并绑定一个特殊的Secret：它的类型是<code>kubernetes.io/service-account-token</code>；它的Annotation字段，声明了<code>kubernetes.io/service-account.name=default</code>，即这个Secret会跟同一Namespace下名叫default的ServiceAccount进行绑定。</p><p>所以，在生产环境中，我强烈建议你为所有Namespace下的默认ServiceAccount，绑定一个只读权限的Role。这个具体怎么做，就当作思考题留给你了。</p><p>除了前面使用的“用户”（User），<span class="orange">Kubernetes还拥有“用户组”（Group）的概念，也就是一组“用户”的意思。</span>如果你为Kubernetes配置了外部认证服务的话，这个“用户组”的概念就会由外部认证服务提供。</p><p>而对于Kubernetes的内置“用户”ServiceAccount来说，上述“用户组”的概念也同样适用。</p><p>实际上，一个ServiceAccount，在Kubernetes里对应的“用户”的名字是：</p><pre><code>system:serviceaccount:&lt;Namespace名字&gt;:&lt;ServiceAccount名字&gt;
</code></pre><p>而它对应的内置“用户组”的名字，就是：</p><pre><code>system:serviceaccounts:&lt;Namespace名字&gt;
</code></pre><p><strong>这两个对应关系，请你一定要牢记。</strong></p><p>比如，现在我们可以在RoleBinding里定义如下的subjects：</p><pre><code>subjects:
- kind: Group
  name: system:serviceaccounts:mynamespace
  apiGroup: rbac.authorization.k8s.io
</code></pre><p>这就意味着这个Role的权限规则，作用于mynamespace里的所有ServiceAccount。这就用到了“用户组”的概念。</p><p>而下面这个例子：</p><pre><code>subjects:
- kind: Group
  name: system:serviceaccounts
  apiGroup: rbac.authorization.k8s.io
</code></pre><p>就意味着这个Role的权限规则，作用于整个系统里的所有ServiceAccount。</p><p>最后，值得一提的是，<strong>在Kubernetes中已经内置了很多个为系统保留的ClusterRole，它们的名字都以system:开头</strong>。你可以通过kubectl get clusterroles查看到它们。</p><p>一般来说，这些系统ClusterRole，是绑定给Kubernetes系统组件对应的ServiceAccount使用的。</p><p>比如，其中一个名叫system:kube-scheduler的ClusterRole，定义的权限规则是kube-scheduler（Kubernetes的调度器组件）运行所需要的必要权限。你可以通过如下指令查看这些权限的列表：</p><pre><code>$ kubectl describe clusterrole system:kube-scheduler
Name:         system:kube-scheduler
...
PolicyRule:
  Resources                    Non-Resource URLs Resource Names    Verbs
  ---------                    -----------------  --------------    -----
...
  services                     []                 []                [get list watch]
  replicasets.apps             []                 []                [get list watch]
  statefulsets.apps            []                 []                [get list watch]
  replicasets.extensions       []                 []                [get list watch]
  poddisruptionbudgets.policy  []                 []                [get list watch]
  pods/status                  []                 []                [patch update]
</code></pre><p>这个system:kube-scheduler的ClusterRole，就会被绑定给kube-system Namesapce下名叫kube-scheduler的ServiceAccount，它正是Kubernetes调度器的Pod声明使用的ServiceAccount。</p><p>除此之外，Kubernetes还提供了四个预先定义好的ClusterRole来供用户直接使用：</p><ol>
<li>
<p>cluster-admin；</p>
</li>
<li>
<p>admin；</p>
</li>
<li>
<p>edit；</p>
</li>
<li>
<p>view。</p>
</li>
</ol><p>通过它们的名字，你应该能大致猜出它们都定义了哪些权限。比如，这个名叫view的ClusterRole，就规定了被作用者只有Kubernetes API的只读权限。</p><p>而我还要提醒你的是，上面这个cluster-admin角色，对应的是整个Kubernetes项目中的最高权限（verbs=*），如下所示：</p><pre><code>$ kubectl describe clusterrole cluster-admin -n kube-system
Name:         cluster-admin
Labels:       kubernetes.io/bootstrapping=rbac-defaults
Annotations:  rbac.authorization.kubernetes.io/autoupdate=true
PolicyRule:
  Resources  Non-Resource URLs Resource Names  Verbs
  ---------  -----------------  --------------  -----
  *.*        []                 []              [*]
             [*]                []              [*]
</code></pre><p>所以，请你务必要谨慎而小心地使用cluster-admin。</p><h2>总结</h2><p>在今天这篇文章中，我主要为你讲解了基于角色的访问控制（RBAC）。</p><p>其实，你现在已经能够理解，所谓角色（Role），其实就是一组权限规则列表。而我们分配这些权限的方式，就是通过创建RoleBinding对象，将被作用者（subject）和权限列表进行绑定。</p><p>另外，与之对应的ClusterRole和ClusterRoleBinding，则是Kubernetes集群级别的Role和RoleBinding，它们的作用范围不受Namespace限制。</p><p>而尽管权限的被作用者可以有很多种（比如，User、Group等），但在我们平常的使用中，最普遍的用法还是ServiceAccount。所以，Role + RoleBinding + ServiceAccount的权限分配方式是你要重点掌握的内容。我们在后面编写和安装各种插件的时候，会经常用到这个组合。</p><h2>思考题</h2><p>请问，如何为所有Namespace下的默认ServiceAccount（default ServiceAccount），绑定一个只读权限的Role呢？请你提供ClusterRoleBinding（或者RoleBinding）的YAML文件。</p><p>感谢你的收听，欢迎你给我留言，也欢迎分享给更多的朋友一起阅读。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/72/3e/534db55d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>huan</span>
  </div>
  <div class="_2_QraFYR_0">kind: ClusterRoleBinding<br>apiVersion: rbac.authorization.k8s.io&#47;v1<br>metadata:<br>  name: readonly-all-default<br>subjects:<br>- kind: User<br>  name: system.serviceaccount.default<br>roleRef:<br>  kind: ClusterRole<br>  name: view<br>  apiGroup: rbac.authorization.k8s.io</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-22 02:04:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/e8/f9/4412b473.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>喜剧。</span>
  </div>
  <div class="_2_QraFYR_0">kind: ClusterRoleBinding<br>apiVersion: rbac.authorization.k8s.io&#47;v1<br>metadata:<br>name: readonly-all-default<br>subjects:<br>- kind: ServiceAccount<br>name: system.serviceaccount.default<br>roleRef:<br>kind: ClusterRole<br>name: view<br>apiGroup: rbac.authorization.k8s.io<br><br>前面的朋友写的问题在于，default应该是serciveacount</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-28 08:23:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>无痕飞客</span>
  </div>
  <div class="_2_QraFYR_0">老师，怎么优雅的卸载掉kubernetes呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: kubeadm reset</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-22 11:13:55</div>
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
  <div class="_2_QraFYR_0">为什么要生命这类service account，不能直接使用role进行权限分配吗？这个中间代理的好处是啥呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 所有的中间层都是为了解耦</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-11 09:30:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/5d/c8/7ece6f4b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>蹦蹦</span>
  </div>
  <div class="_2_QraFYR_0">kind: ClusterRoleBinding<br>apiVersion: rbac.authorization.k8s.io&#47;v1<br>metadata:<br>  name: readonly-all-default<br>subjects:<br>- kind: ServiceAccount<br>  name: default<br>roleRef:<br>  kind: ClusterRole<br>  name: view<br>  apiGroup: rbac.authorization.k8s.io<br><br>kind是ServiceAccount，不是Group。name直接写default，不指定namespace<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-23 11:39:51</div>
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
  <div class="_2_QraFYR_0">老师还是之前的问题，现在机器上有一个手动起的容器（比如是老的业务容器），想把他加到pod里管理起来，比如pod生成的时候发现已经有这个容器了，就关联这个容器，不再创建了。有办法实现么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: kubernetes 里最小的调度单位是pod，所以不可以的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-23 11:59:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/40/c3/e545ba80.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张振宇</span>
  </div>
  <div class="_2_QraFYR_0">老师rbac怎么结合企业自己的ldap用户数据进行外部登录自研的web平台</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-23 13:38:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/9a/0f/da7ed75a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>芒果少侠</span>
  </div>
  <div class="_2_QraFYR_0">老师，我查阅了相关资料。觉得思考题中的场景（为【所有namespace】下的【default service account】添加只读权限）无法实现。ClusterRoleBinding的subjects字段最多能允许我们给【default namespace】下的【所有service account】或者所有namespace下的所有 service account添加只读权限。不知道老师是否有其他方式实现？恳请回答，感谢。<br><br>思考题请问:<br>如何为所有 Namespace 下的默认 ServiceAccount（default ServiceAccount），绑定一个只读权限的 Role 呢？请你提供 ClusterRoleBinding（或者 RoleBinding）的 YAML 文件。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-04 23:26:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/nhLS152kEs5J65bBpM2fzMn4agfoow8xibFzNSDcmo9Oiby2lNB4hWRcetRWFyY2y05IJu8GbkZer9BUiahtadU0w/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>yuanlinios</span>
  </div>
  <div class="_2_QraFYR_0">虽然 clusterrole&#47;clusterrolebinding 不受 namespace 限制, 但是 serviceaccount 总是存在于 namespace 下. 为一个 namespace 下的 default sa 做只读限制很容易. 那么怎么为&quot;所有&quot; (包括现有的和未来的) 的 namespace 下的 default sa 做只读的限制? 希望给点提示<br><br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-28 17:53:18</div>
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
  <div class="_2_QraFYR_0">但在这种情况下，这个默认 ServiceAccount 并没有关联任何 Role。也就是说，此时它有访问 APIServer 的绝大多数权限。<br>为什么没有关联role，就会有绝大多数权限呢？有一个默认的role么，都有什么权限呢？<br><br>另外，建议在所有的namespace给default serviceaccount绑定view，是出于安全的考虑是么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 就是为了安全。role是用来做限制的，你没限制当然就撒野了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-22 13:09:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/3c/c4/4ee2968a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>艾利特-G</span>
  </div>
  <div class="_2_QraFYR_0">关于课后思考，我看了官方文档的这个例子。<br>&gt; https:&#47;&#47;kubernetes.io&#47;docs&#47;reference&#47;access-authn-authz&#47;rbac&#47;#rolebinding-and-clusterrolebinding<br>暂时得出的结论是通过ClusterRoleBinding做不到让每个NameSpace下的&quot;default&quot; ServiceAccount拥有namespaced resources的只读权限。<br>这里面描述了如何在ClusterRolebinding中将ClusterRole与&quot;manager&quot;这个group绑定。但是这个&quot;manager&quot; group，是一个user的group，不是ServiceAccount的group。<br>我想，将ClusterRolebinding其中的subjects[0].kind属性指定为ServiceAccount，name指定为system:serviceaccount:*:default，应该是无效的吧。<br>如果这样不行的话，那就只有subjects[0].kind属性指定为group，然后name指定为system:serviceaccounts，这将会对所有ServiceAccount绑定&quot;view&quot;这个ClusterRole，包括&quot;default&quot;之外的ServiceAccount。这样会对所有ServiceAccount授权集群级别的只读权限，也就是不能限制在该ServiceAccount所在的Namespace下。<br>如果在每个Namespace下创建RoleBinding，subjects[0].kind属性指定为ServiceAccount，name指定为default，namespace指定为该namespace，则可以使每个NameSpace下的&quot;default&quot; ServiceAccount拥有该Namespace的只读权限。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-18 17:12:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/9a/0f/da7ed75a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>芒果少侠</span>
  </div>
  <div class="_2_QraFYR_0">kind: ClusterRoleBinding<br>apiVersion: rbac.authorization.k8s.io&#47;v1<br>metadata:<br>  name: readonly-all-default<br>subjects:<br>- kind: ServiceAccount<br>  name: default<br>  namespace: system:systemaccounts<br>roleRef:<br>  kind: ClusterRole<br>  name: view<br>  apiGroup: rbac.authorization.k8s.io<br><br>老师，请问这样写对吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-04 22:52:21</div>
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
  <div class="_2_QraFYR_0">老师，我就想问一下，namespace 下的 default sa 没有关联任何 role，它到底有哪些权限呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-09 11:24:30</div>
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
  <div class="_2_QraFYR_0">Prior to Kubernetes 1.6, many deployments used very permissive ABAC policies, including granting full API access to all service accounts.<br><br>Default RBAC policies grant scoped permissions to control-plane components, nodes, and controllers, but grant no permissions to service accounts outside the kube-system namespace (beyond discovery permissions given to all authenticated users).<br><br>Quoting from https:&#47;&#47;kubernetes.io&#47;docs&#47;reference&#47;access-authn-authz&#47;rbac&#47;#service-account-permissions<br><br>按我对官方文档的理解，RBAC策略下 default service account 是不是并没有任何权限，ABAC才会grant full access？<br><br>如果给所有namespace的default service account都赋予view 权限。会不会出现如下风险？<br>Warning: This allows any user with read access to secrets or the ability to create a pod to access super-user credentials.<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，这个风险肯定的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-23 13:46:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/ac/bb/db0f9d8c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>tale</span>
  </div>
  <div class="_2_QraFYR_0">思考题正解在此：<br><br>kind: ClusterRoleBinding<br>apiVersion: rbac.authorization.k8s.io&#47;v1<br>metadata:<br>  name: readonly-all-default<br>subjects:<br>- kind: Group<br>  name: system:serviceaccounts<br>  apiGroup: rbac.authorization.k8s.io<br>roleRef:<br>  kind: ClusterRole<br>  name: view<br>  apiGroup: rbac.authorization.k8s.io<br><br>关于subjects中kind的解释：<br><br>1、kind为ServiceAccount的话，官方文档明确说了必须指定namespace的，所以排除这种方式。<br>2、kind为User时，name中一定会带有&lt;namespace&gt;信息，与题目要求“所有namespace”不符，所以也排除。<br>3、kind为Group时，name中填system:serviceaccounts即可。很多人担心这个Group过大，可能不止包含了所有default“用户”。但经我查明，这个Group只有Get权限，所以符合View规则，即使包含了其他“用户”也无妨。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-01 09:18:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/be/b9/f2481c2c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>诗泽</span>
  </div>
  <div class="_2_QraFYR_0">“实际上，一个 ServiceAccount，在 Kubernetes 里对应的“用户”的名字是：”<br>ServiceAccount 有namespace 但是用户没有，那serviceaccount 与用户的映射关系是怎样的呢，所有namespace 的default ServiceAccount 对应同一个user 吗？即system.serviceaccount.default ？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-14 10:43:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/46/5b/07858c33.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Pixar</span>
  </div>
  <div class="_2_QraFYR_0">role roleBanding serviceAccount 都是 namespaced , 那跨namespace 操作会怎么样？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 找不到对象</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-26 07:59:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/20/c3/94/e89ebc50.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>神毓逍遥</span>
  </div>
  <div class="_2_QraFYR_0">课代表来了，最近使用部署的 AWS 上的项目，由于config 中是 aws 特质的 CLI 命令，需要重新生成一个账号与去哪先，我自己使用的 LEN 工具，图形化界面操作，非常好用；另外这张内容蛮重要的，结合自己的经验主要在，1. 学会如何创建 账号、角色、绑定，方便授权权限，2则理解 <br>system:serviceaccount:&lt;Namespace名字&gt;:&lt;ServiceAccount名字&gt; 这个结构的含义，一定要注意，因为操作中报错，极大的概率会遇到这个问题，理解概念方便排查</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-12 10:09:42</div>
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
  <div class="_2_QraFYR_0">实测通过，所有默认default serviceaccount有了list pod的权限<br><br>#cat example-clusterrolebinding.yaml<br>kind: ClusterRoleBinding<br>apiVersion: rbac.authorization.k8s.io&#47;v1<br>metadata:<br>  name: example-clusterrolebinding<br>subjects:<br>- kind: Group<br>  name: system:serviceaccounts<br>  apiGroup: rbac.authorization.k8s.io<br>roleRef:<br>  kind: ClusterRole<br>  name: example-clusterrole<br>  apiGroup: rbac.authorization.k8s.io</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-19 20:51:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/12/04/5837b21c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Brown羊羊</span>
  </div>
  <div class="_2_QraFYR_0">kind: RoleBinding<br>apiVersion: rbac.authorization.k8s.io&#47;v1<br>metadata:<br>  name: example-rolebinding<br>subjects:<br>- kind: Group<br>  name: system:serviceaccounts<br>roleRef:<br>  kind: ClusterRole<br>  name: view<br>  apiGroup: rbac.authorization.k8s.io</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-28 22:14:39</div>
  </div>
</div>
</div>
</li>
</ul>