<audio title="04｜如何借助GitOps实现应用秒级自动发布和回滚？" src="https://static001.geekbang.org/resource/audio/53/f8/53880df5407a20130503f3b80535e6f8.mp3" controls="controls"></audio> 
<p>你好，我是王炜。</p><p>在上一节课，我为你介绍了 K8s 在自愈和自动扩容方面的强大能力，它们对提升业务稳定性有非常大的帮助。</p><p>其实，除了保障业务稳定以外，在日常软件研发的生命周期中，还有非常重要的一环：发布和回滚。发布周期是体现研发效率的一项重要指标，更早更快的发布有利于我们及时发现问题。</p><p>在我们有了关于容器化、K8s 工作负载的基础之后，这节课，我们来看看 K8s 应用发布的一般做法，此外，我还会带你从零开始构建 GitOps 工作流，体验 GitOps 在应用发布上为我们带来的全新体验。</p><p>在正式开始之前，你需要做好以下准备：</p><ul>
<li>准备一台电脑（首选 Linux 或 macOS，Windows 系统注意操作差异）；</li>
<li><a href="https://docs.docker.com/engine/install/">安装 Docker</a>；</li>
<li><a href="https://kubernetes.io/docs/tasks/tools/">安装 Kubectl</a>；</li>
<li>按照上一节课的内容在本地 Kind 集群安装 Ingress-Nginx。</li>
</ul><h2>传统 K8s 应用发布流程</h2><p>还记得在上节课学习的如何创建 Deployment 工作负载吗？下面这段 Deployment Manifest 可以帮助你复习一下：</p><pre><code class="language-yaml">apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: hello-world-flask
  name: hello-world-flask
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello-world-flask
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: hello-world-flask
    spec:
      containers:
      - image: lyzhang1999/hello-world-flask:latest
        name: hello-world-flask
        resources: {}
status: {}
</code></pre><!-- [[[read_end]]] --><p>当我们在部署 Deployment 工作负载的时候，Image 字段同时指定了镜像名称和版本号。在发布应用的过程中，一般会先修改 Manifest 镜像版本，再使用 kubectl apply 重新将 Manifest 应用到集群来更新应用。</p><p>你可能会问，那在升级应用的过程中，新老版本的切换会导致服务中断吗？答案当然是不会的，并且 K8s 将会自动处理，无需人工干预。</p><p><strong>接下来，我们进入实战环节。我们先尝试通过手动的方式来更新应用，这也是传统 K8s 发布应用的过程。</strong></p><p>通常，更新应用可以使用下面三种方式：</p><ul>
<li>使用 kubectl set image 命令；</li>
<li>修改本地的 Manifest ；</li>
<li>修改集群内 Manifest 。</li>
</ul><h3>通过 kubectl set image 命令更新应用</h3><p>要想更新应用，最简单的方式是通过 kubectl set image 来更新集群内已经存在工作负载的镜像版本，例如更新 hello-world-flask Deployment 工作负载：</p><pre><code class="language-powershell">$ kubectl set image deployment/hello-world-flask hello-world-flask=lyzhang1999/hello-world-flask:v1
deployment.apps/hello-world-flask image updated
</code></pre><p>为了方便你动手实践，我已经给你制作了 hello-world-flask:v1 版本的镜像，新镜像版本修改了 Python 的返回内容，你可以直接使用。</p><p>当 K8s 接收到镜像更新的指令时，K8s 会用新的镜像版本重新创建 Pod。你可以使用 kubectl get pods 来查看 Pod 的更新情况：</p><pre><code class="language-powershell">$ kubectl get pods
NAME&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; READY&nbsp; &nbsp;STATUS&nbsp; &nbsp; RESTARTS&nbsp; &nbsp;AGE
hello-world-flask-8f94845dc-qsm8b&nbsp; &nbsp;1/1&nbsp; &nbsp; &nbsp;Running&nbsp; &nbsp;0&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; 3m38s
hello-world-flask-8f94845dc-spd6j&nbsp; &nbsp;1/1&nbsp; &nbsp; &nbsp;Running&nbsp; &nbsp;0&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; 3m21s
hello-world-flask-64dd645c57-rfhw5&nbsp; &nbsp;0/1&nbsp; &nbsp; &nbsp;ContainerCreating&nbsp; &nbsp;0&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; 1s
hello-world-flask-64dd645c57-ml74f&nbsp; &nbsp;0/1&nbsp; &nbsp; &nbsp;ContainerCreating&nbsp; &nbsp;0&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; 0s
</code></pre><p>在更新 Pod 的过程中，K8s 会确保先创建新的 Pod ，然后再终止旧镜像版本的 Pod。Pod 的副本数始终保持在我们在 Deployment Manifest 中配置的 2 。</p><p>现在，你可以打开浏览器访问 127.0.0.1 ，查看返回内容：</p><pre><code class="language-powershell">Hello, my v1 version docker images! hello-world-flask-8f94845dc-bpgnp
</code></pre><p>通过返回内容我们可以发现，新镜像对应的 Pod 已经替换了老的 Pod，这也意味着我们的应用更新成功了。</p><p><strong>从本质上来看，kubectl set image 是修改了集群内已部署的 Deployment 工作负载的 Image 字段，继而触发了 K8s 对 Pod 的变更。</strong></p><p>有时候，我们在一次发布中希望变更的内容不仅仅是镜像版本，可能还有副本数、端口号等等。这时候，我们可以对新的 Manifest 文件再执行一次 kubectl apply ，K8s 会比对它们之间的差异，然后做出变更。</p><h3>通过修改本地的 Manifest 更新应用</h3><p>除了使用 kubectl set image 来更新应用， 我们还可以修改本地的 Manifest 文件并将其重新应用到集群来更新。</p><p>以 hello-world-flask Deployment 为例，我们重新把镜像版本修改为 latest，将下面的内容保存为 new-hello-world-flask.yaml：</p><pre><code class="language-powershell">apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: hello-world-flask
  name: hello-world-flask
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello-world-flask
  template:
    metadata:
      labels:
        app: hello-world-flask
    spec:
      containers:
      - image: lyzhang1999/hello-world-flask:latest
        name: hello-world-flask
</code></pre><p>接下来，执行 kubectl apply -f new-hello-world-flask.yaml 来更新应用：</p><pre><code class="language-powershell">$ kubectl apply -f new-hello-world-flask.yaml
deployment.apps/hello-world-flask configured
</code></pre><p>也就是说，kubectl apply 命令会自动处理两种情况：</p><ol>
<li>如果该资源不存在，那就创建资源；</li>
<li>如果资源存在，那就更新资源。</li>
</ol><p>到这里，我相信有一些同学可能会有疑问，如果我本地的 Manifest 找不到了，我可以直接更新集群内已经存在的 Manifest 吗？答案是肯定的。也就是说，我们还可以直接编辑集群内的 Manifest 来更新应用，这就是更新应用的第三种方式。</p><h3>通过修改集群内 Manifest 更新应用</h3><p>以 hello-world-flask Deployment 为例，要直接修改集群内已部署的 Manifest，你可以使用 kubectl edit 命令：</p><pre><code class="language-powershell">$ kubectl edit deployment hello-world-flask
</code></pre><p>执行命令后，kubectl 会自动为我们下载集群内的 Manifest 到本地，并且用 VI 编辑器自动打开。你可以进入 VI 的编辑模式修改任何字段，<strong>保存退出后修改即时生效。</strong></p><p><strong>总结来说，要更新 K8s 的工作负载，我们可以修改本地的 Manifest，再使用 kubectl apply 将它重新应用到集群内，或者通过 kubectl edit 命令直接修改集群内已存在的工作负载。</strong></p><p>在实际项目的实践中，负责更新应用的同学早期可能会在自己的电脑上操作，然后把这部分操作挪到 CI 过程，例如使用 Jenkins 来执行。</p><p>但是，随着项目的发展，我们会需要发布流程更加自动化、安全、可追溯。这时候，我们应该考虑用 GitOps 的方式来发布应用。</p><h2>从零搭建 GitOps 发布工作流</h2><p>在正式搭建 GitOps 之前，我想先让你对 GitOps 有个简单的理解。通俗来说，GitOps 就是以 Git 版本控制为理念的 DevOps 实践。</p><p>对于这节课要设计的 GitOps 发布工作流来说，我们会将 Manifest 存储在 Git 仓库中作为期望状态，一旦修改并提交了 Manifest ，那么 GitOps 工作流就会<strong>自动比对 Git 仓库和集群内工作负载的实际差异</strong>，并进行部署。</p><h3>安装 FluxCD 并创建工作流</h3><p>要实现 GitOps 工作流，首先我们需要一个能够帮助我们监听 Git 仓库变化，自动部署的工具。这节课，我以 FluxCD 为例，带你一步步构建出一个 GitOps 工作流。</p><p><strong>接下来，我们进入实战环节。</strong></p><p>首先，我们需要在集群内安装 FluxCD：</p><pre><code class="language-powershell">$ kubectl apply -f https://ghproxy.com/https://raw.githubusercontent.com/lyzhang1999/resource/main/fluxcd/fluxcd.yaml
</code></pre><p>由于安装 FluxCD 的工作负载比较多，你可以使用 kubectl wait 来等待安装完成：</p><pre><code class="language-powershell">$ kubectl wait --for=condition=available --timeout=300s --all deployments -n flux-system
deployment.apps/helm-controller condition met
deployment.apps/image-automation-controller condition met
deployment.apps/image-reflector-controller condition met
deployment.apps/kustomize-controller condition met
deployment.apps/notification-controller condition met
deployment.apps/source-controller condition met
</code></pre><p>接下来，在本地创建 fluxcd-demo 目录：</p><pre><code class="language-powershell">$ mkdir fluxcd-demo &amp;&amp; cd fluxcd-demo
</code></pre><p>然后，我们在 fluxcd-demo 目录下创建 deployment.yaml 文件，并将下面的内容保存到这个文件里：</p><pre><code class="language-yaml">apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: hello-world-flask
  name: hello-world-flask
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello-world-flask
  template:
    metadata:
      labels:
        app: hello-world-flask
    spec:
      containers:
      - image: lyzhang1999/hello-world-flask:latest
        name: hello-world-flask
</code></pre><p>最后，在 Github 或 Gitlab 中创建 fluxcd-demo 仓库。为了方便测试，我们需要将仓库设置为公开权限，主分支为 Main，并将我们创建的 Manifest 推送至远端仓库：</p><pre><code class="language-powershell">$ ls
deployment.yaml
$ git init
......
Initialized empty Git repository in /Users/wangwei/Downloads/fluxcd-demo/.git/
$ git add -A &amp;&amp; git commit -m "Add deployment"
[master (root-commit) 538f858] Add deployment
&nbsp;1 file changed, 19 insertions(+)
&nbsp;create mode 100644 deployment.yaml
$ git branch -M main
$ git remote add origin https://github.com/lyzhang1999/fluxcd-demo.git
$ git push -u origin main
</code></pre><p>这是我的<a href="https://github.com/lyzhang1999/fluxcd-demo">仓库地址</a>，你可以参考一下。</p><p>下一步，我们为 FluxCD 创建仓库连接信息，将下面的内容保存为 fluxcd-repo.yaml：</p><pre><code class="language-yaml">apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: GitRepository
metadata:
  name: hello-world-flask
spec:
  interval: 5s
  ref:
    branch: main
  url: https://github.com/lyzhang1999/fluxcd-demo
</code></pre><p>注意，要将 URL 字段修改为你实际仓库的地址并使用 HTTPS 协议，branch 字段设置 main 分支。这里的 interval 代表每 5 秒钟主动拉取一次仓库并把它作为制品存储。</p><p>接着，使用 kubectl apply 将其 GitRepository 对象部署到集群内：</p><pre><code class="language-powershell">$ kubectl apply -f fluxcd-repo.yaml
gitrepository.source.toolkit.fluxcd.io/hello-world-flask created
</code></pre><p>你可以使用 kubectl get gitrepository 来检查配置是否生效：</p><pre><code class="language-powershell">$ kubectl get gitrepository
NAME&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; URL&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; AGE&nbsp; &nbsp;READY&nbsp; &nbsp;STATUS
hello-world-flask&nbsp; &nbsp;https://github.com/lyzhang1999/fluxcd-demo&nbsp; &nbsp;5s&nbsp; &nbsp; True&nbsp; &nbsp; stored artifact for revision 'main/8260f5a0ac1e4ccdba64e074d1ee2c154956f12d'
</code></pre><p>接下来，我们还需要为 FluxCD 创建部署策略。将下面的内容保存为 fluxcd-kustomize.yaml：</p><pre><code class="language-yaml">apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: hello-world-flask
spec:
  interval: 5s
  path: ./
  prune: true
  sourceRef:
    kind: GitRepository
    name: hello-world-flask
  targetNamespace: default
</code></pre><p>在上面的配置中，interval 参数表示 FluxCD 会每 5 秒钟运行一次工作负载差异对比，path 参数表示我们的 deployment.yaml 位于仓库的根目录中。FluxCD在对比期望状态和集群实际状态的时候，如果发现差异就会触发重新部署。</p><p>我们再次使用 kubectl apply 将 Kustomization 对象部署到集群内：</p><pre><code class="language-powershell">$ kubectl apply -f fluxcd-kustomize.yaml
kustomization.kustomize.toolkit.fluxcd.io/hello-world-flask created
</code></pre><p>同样地，你可以使用 kubectl get kustomization 来检查配置是否生效：</p><pre><code class="language-powershell">$ kubectl get kustomization
NAME&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; AGE&nbsp; &nbsp; &nbsp;READY&nbsp; &nbsp;STATUS
hello-world-flask&nbsp; &nbsp;8m21s&nbsp; &nbsp;True&nbsp; &nbsp; Applied revision: main/8260f5a0ac1e4ccdba64e074d1ee2c154956f12d
</code></pre><p>配置完成后，接下来，<strong>我们就可以正式体验 GitOps 的秒级自动发布和回滚了。</strong></p><h3>自动发布</h3><p>现在，我们修改 fluxcd-demo 仓库的 deployment.yaml 文件，将 image 字段的镜像版本从 latest 修改为 v1：</p><pre><code class="language-powershell">......
    spec:
      containers:
      - image: lyzhang1999/hello-world-flask:v1 # 修改此处
        name: hello-world-flask
......
</code></pre><p>然后，我们将修改推送到远端仓库：</p><pre><code class="language-powershell">$ git add -A &amp;&amp; git commit -m "Update image tag to v1"
$ git push origin main
</code></pre><p>你可以使用 kubectl describe kustomization hello-world-flask 查看触发重新部署的事件：</p><pre><code class="language-powershell">$ kubectl describe kustomization hello-world-flask
......
Status:
&nbsp; Conditions:
&nbsp; &nbsp; Last Transition Time:&nbsp; 2022-09-10T03:46:37Z
&nbsp; &nbsp; Message:&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;Applied revision: main/8260f5a0ac1e4ccdba64e074d1ee2c154956f12d
&nbsp; &nbsp; Reason:&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; ReconciliationSucceeded
&nbsp; &nbsp; Status:&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; True
&nbsp; &nbsp; Type:&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; Ready
&nbsp; Inventory:
&nbsp; &nbsp; Entries:
&nbsp; &nbsp; &nbsp; Id:&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;default_hello-world-flask_apps_Deployment
&nbsp; &nbsp; &nbsp; V:&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; v1
&nbsp; Last Applied Revision:&nbsp; &nbsp; main/8260f5a0ac1e4ccdba64e074d1ee2c154956f12d
&nbsp; Last Attempted Revision:&nbsp; main/8260f5a0ac1e4ccdba64e074d1ee2c154956f12d
&nbsp; Observed Generation:&nbsp; &nbsp; &nbsp; 1
......
</code></pre><p>从返回的结果可以看出，我们将镜像版本修改为了 v1，并且，FluxCD 最后一次部署仓库的 Commit ID 是 8260f5a0ac1e4ccdba64e074d1ee2c154956f12d，这对应了我们最后一次的提交记录，说明变更已经生效了。</p><p>现在，我们打开浏览器访问 127.0.0.1，可以看到 v1 镜像版本的输出内容：</p><pre><code class="language-powershell">Hello, my v1 version docker images! hello-world-flask-6d7b779cd4-spf4q
</code></pre><p>通过上面的配置，我们让 FluxCD 自动完成了监听修改、比较和重新部署三个过程。怎么样，GitOps 的发布流程是不是比手动发布方便多了呢？</p><p>接下来我们再感受一下 GitOps 的快速回滚能力。</p><h3>发布回滚</h3><p>既然 GitOps 工作流中，Git 仓库是描述期望状态的唯一可信源，那么我们是不是只要对 Git 仓库执行回滚，就可以实现发布回滚呢？</p><p><strong>我们通过实战来验证一下这个猜想。</strong></p><p>要回滚 fluxcd-demo 仓库，首先需要找到上一次的提交记录。我们可以使用 git log 来查看它：</p><pre><code class="language-powershell">$ git log
commit 900357f4cfec28e3f80fde239906c1af4b807be6 (HEAD -&gt; main, origin/main)
Author: wangwei &lt;434533508@qq.com&gt;
Date:&nbsp; &nbsp;Sat Sep 10 11:24:22 2022 +0800
  
&nbsp; &nbsp; Update image tag to v1

commit 75f39dc58101b2406d4aaacf276e4d7b2d429fc9
Author: wangwei &lt;434533508@qq.com&gt;
Date:&nbsp; &nbsp;Sat Sep 10 10:35:41 2022 +0800

&nbsp; &nbsp; first commit
</code></pre><p>可以看到，上一次的 commit id 为 75f39dc58101b2406d4aaacf276e4d7b2d429fc9，接下来使用 git reset 来回滚到上一次提交，并强制推送到 Git 仓库：</p><pre><code class="language-powershell">$ git reset --hard 75f39dc58101b2406d4aaacf276e4d7b2d429fc9
HEAD is now at 538f858 Add deployment

$ git push origin main -f
Total 0 (delta 0), reused 0 (delta 0), pack-reused 0
To https://github.com/lyzhang1999/fluxcd-demo.git
&nbsp;+ 8260f5a...538f858 main -&gt; main (forced update)
</code></pre><p>再次使用 kubectl describe kustomization hello-world-flask 查看触发重新部署的事件：</p><pre><code class="language-powershell">......
Status:
&nbsp; Conditions:
&nbsp; &nbsp; Last Transition Time:&nbsp; 2022-09-10T03:51:28Z
&nbsp; &nbsp; Message:&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;Applied revision: main/538f858909663f4be3a62760cb571529eb50a831
&nbsp; &nbsp; Reason:&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; ReconciliationSucceeded
&nbsp; &nbsp; Status:&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; True
&nbsp; &nbsp; Type:&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; Ready
&nbsp; Inventory:
&nbsp; &nbsp; Entries:
&nbsp; &nbsp; &nbsp; Id:&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;default_hello-world-flask_apps_Deployment
&nbsp; &nbsp; &nbsp; V:&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; latest
&nbsp; Last Applied Revision:&nbsp; &nbsp; main/538f858909663f4be3a62760cb571529eb50a831
&nbsp; Last Attempted Revision:&nbsp; main/538f858909663f4be3a62760cb571529eb50a831
&nbsp; Observed Generation:&nbsp; &nbsp; &nbsp; 1
......
</code></pre><p>从返回结果的 Last Applied Revision 可以看出，FluxCD 已经检查到了变更，并已经进行了同步。</p><p>再次打开浏览器访问 127.0.0.1，可以看到返回结果已回滚到了 latest 镜像对应的内容：</p><pre><code class="language-powershell">Hello, my first docker images! hello-world-flask-56fbff68c8-c8dc4
</code></pre><p>到这里，我们就成功实现了 GitOps 的发布和回滚。</p><h2>总结</h2><p>这节课，我为你归纳了 K8s 更新应用镜像的 3 种基本操作，他们分别是：</p><ol>
<li>kubectl set image；</li>
<li>修改本地 Manifest 并重新执行 kubectl apply -f；</li>
<li>通过 kubectl edit 直接修改集群内的工作负载。</li>
</ol><p>这种手动更新应用的方法效率非常低，最重要的是很难回溯，会让应用回滚变得困难。所以，我们引入了一种全新 GitOps 工作流的发布方式来解决这些问题。</p><p>在这节课的实战当中，我们只实现了 GitOps 环节中的一小部分，我希望通过这个小小的试炼，让你认识到 GitOps 的价值。</p><p>在实际项目中，构建端到端的 GitOps 工作流其实还有非常多的细节，例如如何在修改代码后自动构建并推送镜像，如何自动更新 Manifest 仓库等，这些进阶的内容我都会在后续的课程中详细介绍。</p><p>另外，能实现 GitOps 的工具其实并不止 FluxCD，在你为实际项目构建生产级的 GitOps 工作流时，我推荐你使用 ArgoCD，这也是我们这个专栏接下来会重点介绍的内容。</p><p>最后，在前面几节课里，我们引出了非常多 K8s 相关的概念，例如工作负载、Service、Ingress、HPA 等等，为了快速实战并让你感受 K8s 和 GitOps 的价值，之前我并没有详细解释这些概念。但当你要将真实的项目迁移到 K8s 的时候，这些内容是我们必须要熟练掌握的。</p><p>所以，为了让你在工作过程中对 K8s 更加得心应手，我会在接下来第二模块为你提供零基础的 K8s 极简入门教程。我会详细介绍之前出现过的 K8s 常用对象，让你真正掌握他们，扫除将公司项目迁移到 K8s 的技术障碍，<strong>迈出 GitOps 的第一步。</strong></p><h2>思考题</h2><p>最后，给你留一道思考题吧。</p><p>请你分享一下你们现在使用的发布方案是什么？相比 GitOps 的发布方式，你认为它有哪些优缺点呢？</p><p>欢迎你给我留言交流讨论，你也可以把这节课分享给更多的朋友一起阅读。我们下节课见。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/23/52/66/3e4d4846.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>includestdio.h</span>
  </div>
  <div class="_2_QraFYR_0">我们目前是通过gitlab-ci和自己搭建的CD平台实现流水线，代码变更后ci发起镜像构建，并推送到镜像仓库，CD平台通过hook监听到CI流水线完成，重建Pod并拉取最新的镜像。 缺点是#我们的镜像版本号只区分了 dev mirror prod，回滚不是很优雅，只能是通过gitlab回滚代码重新构建镜像，重新触发CD，效率低，对开发同学也不太友好。最近正在着手看怎么调整，实现通过镜像版本号回滚。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 欢迎你关注后续的课程，你的问题在第 17 讲中有提到。<br>我建议你先使用 git commit id 作为镜像版本号，这样可以把代码版本和镜像对应起来，回滚的话只要找到 commit id 就可以了。对于生产镜像，则可以采用额外的策略，例如 prod-commit_id 把他和常规开发镜像区分开。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-13 00:11:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/ff/28/040f6f01.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Y</span>
  </div>
  <div class="_2_QraFYR_0">我本地windows实验成功了，感谢大佬</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 加油，后续课程还有更好玩的实验。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-15 11:14:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/c4/03/f753fda7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>JianXu</span>
  </div>
  <div class="_2_QraFYR_0">老师，你能介绍一下为什么flux 能成功吗？他比其他gitops 方案到底强在哪里</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: FluxCD 在 11 月底刚通过 CNCF 的评审，进入了毕业阶段。<br><br>这意味着社区已经在生产环境下大规模采用了 FluxCD，它的稳定性得到了充分的验证。<br><br>GitOps 领域目前两大工具中，FluxCD 的强项在于做集成，而 ArgoCD 更适合工程实践。这两款工具都非常优秀，所以在专栏里我都有进行介绍。<br><br>关于 ArgoCD，在后续第 22 讲中会深入介绍，期待我们再见面。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-13 22:47:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/65/e8/d1e52dbb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>IF-Processing</span>
  </div>
  <div class="_2_QraFYR_0">我目前使用的方案是Gitlab-ci将代码检查，编译，制作docker镜像一套走完后，自动推送到生产的镜像仓库，之后进行手工部署和回滚操作。目前有痛点，主要是目前运行的应用是SpringCloud体系的应用程序，每次发版时，需要手工维护Nacos配置，数据变更脚本等功能。请教下，如果使用GitOps体系进行操作时，是否对于运行的目标体系有要求呢？我理解如果是使用K8S的configmap之类的信息进行配置，使用Service进行负载均衡的话，应该很好实现配置与代码一起部署。但是如果是这种依赖Nacos配置注册中心的微服务体系，GitOps体系也能很好的支撑吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以考虑使用 Helm 来封装应用，利用 Helm pre-install 在部署前执行特定的变更配置的 Job，这样就可以做到和 GitOps 工作流进行结合了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-17 22:46:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/90/95/86b21093.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>浅浅</span>
  </div>
  <div class="_2_QraFYR_0">老师好，内容太棒了！请问什么时候更新后面的内容呀</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 今天已经更新了哦，每周一、三、五更新，期待和你再见面~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-17 20:01:23</div>
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
  <div class="_2_QraFYR_0">我们现在使用的是在Jenkins上通过Terraform部署Helm chart，只要修改Terraform里的镜像版本号就能部署或回滚应用。类似这个lab: https:&#47;&#47;github.com&#47;chance2021&#47;devopsdaydayup&#47;blob&#47;main&#47;004-TerraformDockerDeployment&#47;README.md</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 也是一种方案。<br><br>不过在工程实践中不建议在 CI（Jenkins）里干持续部署的活，本质上持续部署需要更多的能力支持。比如在界面上应该能够很方便看出应用版本、健康状态、应用资源拓扑等，在部署能力支持上，可能还需要蓝绿部署、灰度和金丝雀发布，甚至是结合人工审核定制发布工作流。<br><br>这部分的内容可以在第 22、24、25 和 26 将深入了解。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-13 11:33:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/43/05/3fbf26cf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Noel ZHANG</span>
  </div>
  <div class="_2_QraFYR_0">我们用的是spinnaker， 配置不难，完全可以terraform创建，里面集成了helm和customize，可以用Jenkins job，docker repository, pub&#47;sub 自动触发。用惯了觉得挺好用，但是听孟凡杰说用的人不多。。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Spinnaker 在国内确实用的比较少，不过它在国外很流行，也是老牌的持续部署工具，整体相对比较重。<br><br>如果对 Spinnaker 感兴趣，可以看看我写的 《Soinnaker 实战》这本书。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-11 22:39:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/83/51/aa521f2a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>大圈</span>
  </div>
  <div class="_2_QraFYR_0">实践起来！很棒</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 加油~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-16 14:42:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://wx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLRibTnDs0ZjFrAtfzcwDDFnaX1DEY8qKoFczP8e8ucAdTr7C33bYFDYxpN8VRhgEVsDrBwILO8Msw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>booboo</span>
  </div>
  <div class="_2_QraFYR_0">我把fluxcd-repo.yaml里的url改成我自己的仓库地址，执行kubectl apply -f fluxcd-repo.yaml之后用kubectl get gitrepository查看状态，报这个错“failed to checkout and determine revision: unable to clone &#39;https:&#47;&#47;jihulab.com&#47;xxxx&#47;fluxcd-demo&#39;: authentication required“。本地跟jihulab的ssh连接是正常的。报这个错是什么原因？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 需要把仓库权限设置为公开，私有仓库需要为 fluxcd 提供权限，参考这个链接：https:&#47;&#47;fluxcd.io&#47;flux&#47;components&#47;source&#47;gitrepositories&#47;#basic-access-authentication</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-12 22:23:16</div>
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
  <div class="_2_QraFYR_0">老师的QQ号暴露了😁</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 没关系，很久不用了😁</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-12 19:47:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/qftso2tiat4Y6LB5dxynrqm54aprlPGQBEuPsFLoyEr8JLKoJAmjtFePG8YzaDqlk5UVIsIUMMIH7Yg7iaWhTnmQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_2cd417</span>
  </div>
  <div class="_2_QraFYR_0">修改代码仓库后我的页面跟着修改了，实现了自动部署。 但是kubectl describe kustomization hello-world-flask输出的版本一直是v1没有变，为什么呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 示例里是从 latest 的镜像版本修改为 v1，如果你也是按照这个例子的话看到 v1 说明已经自动更新了。<br><br>如果不是这个问题，你还可以检查一下集群和 github 的连通性。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-28 14:27:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/qftso2tiat4Y6LB5dxynrqm54aprlPGQBEuPsFLoyEr8JLKoJAmjtFePG8YzaDqlk5UVIsIUMMIH7Yg7iaWhTnmQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_2cd417</span>
  </div>
  <div class="_2_QraFYR_0">修改代码仓库后我，我的页面跟着修改了，实现了自动部署。但是为什么kubectl describe kustomization hello-world-flask</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-28 14:25:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/54/21/0bac2254.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>橙汁</span>
  </div>
  <div class="_2_QraFYR_0">牛逼牛逼 思路清晰</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-20 16:16:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/23/52/66/3e4d4846.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>includestdio.h</span>
  </div>
  <div class="_2_QraFYR_0">“接下来，执行 kubectl apply -f new-hello-worlad-flask.yaml 来更新应用” world写错了 ，还有下面的命令</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢指正</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-19 10:45:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/23/52/66/3e4d4846.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>includestdio.h</span>
  </div>
  <div class="_2_QraFYR_0">接下来，执行 kubectl apply -f new-hello-worlad-flask.yaml 来更新应用</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-19 10:44:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/21/8c/78/25eeacd7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>🍓 雲</span>
  </div>
  <div class="_2_QraFYR_0">Fluxed 如果想回推退某个helm chart 版本，该怎么做比较优雅</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 一般做法是回退存储 Helm Chart 的 Git 仓库，FluxCD 会比较差异，并实现回退。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-17 23:20:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/65/e8/d1e52dbb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>IF-Processing</span>
  </div>
  <div class="_2_QraFYR_0">大佬，在线等，催更.. ：）</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 今天更新了，很高兴再见到你~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-17 22:49:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/29/01/203fcb5d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Fchen</span>
  </div>
  <div class="_2_QraFYR_0">这个gitops阶段看起来并不完整，最直接的是缺少了镜像构建阶段的流程。对于企业生产使用，还是简单了点，我们是自研的，可以根据不同角色，不同业务场景，不同发布场景等做一些适配</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，本章介绍的是一个最简单的例子。企业级的 GitOps 工作流在 22 讲中有介绍，期待再次看到你的回复~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-14 22:48:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/d9/36/92d8eb91.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Promise</span>
  </div>
  <div class="_2_QraFYR_0">我们目前使用的方案是gitlab+Tekton+Argocd的方案但是我只搞了80%还没有完成在打包完镜像以后修改Helm部分。Tekton负责CI的部分，通过git提交触发gitlab的的push事件，Tekton监听push事件，使用doud的方式打包镜像。Argocd负责CD的部分监听gitlab上Helm项目的变化，然后自动部署到k8s上。老师会讲kaniko打包镜像吗？还有CICD项目很多时使用Tekton如何管理，是一个项目创建一个pipline吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这部分内容在第 18 讲中有介绍，期待你的留言。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-14 16:18:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/dd/09/feca820a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>helloworld</span>
  </div>
  <div class="_2_QraFYR_0">good</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-12 21:22:34</div>
  </div>
</div>
</div>
</li>
</ul>