<audio title="23｜如何监听镜像版本变化触发 GitOps？" src="https://static001.geekbang.org/resource/audio/03/96/03d2a0b24207b6416f9e2c32116c5896.mp3" controls="controls"></audio> 
<p>你好，我是王炜。</p><p>上一节课，我带你学习了如何使用 ArgoCD 来创建生产可用的 GitOps 工作流。值得注意的是，我们创建的 GitOps 工作流有下面两个重要的特点。</p><ol>
<li>源码和 Helm Chart 在同一个仓库下。</li>
<li>在 CI 阶段更新了 Helm Chart 镜像版本。</li>
</ol><p>在开发和发布分工明确的团队中，我更推荐你将源码和应用定义分离，考虑到安全性和发布的严谨性，也尽量不要通过 CI 直接修改应用定义。</p><p>更合理的研发规范设计应该是这样的：开发负责编写代码，并通过 CI 生成制品，也就是 Docker 镜像，并对生成的制品负责。而基础架构部门或者 SRE 团队则对应用定义负责。在发布环节，<strong>开发可以随时控制要发布的镜像版本，而无需关注其他的应用细节</strong>，他们之间的工作流程图如下。</p><p><img src="https://static001.geekbang.org/resource/image/ca/00/ca9a2a6b7c938738ae56153906e4b200.jpg?wh=1920x1267" alt="图片"></p><p>从上面这张工作流程图我们可以看出，开发和 SRE 团队各司其职，只操作和自己相关的 Git 仓库，互不干扰。但 SRE 团队要怎么知道开发团队什么时候发布以及发布什么版本的镜像呢？</p><p>最原始的办法是：开发在需要发布的时候将镜像版本告诉 SRE 团队，SRE 团队手动修改 Helm Chart 镜像版本并推送到 Git 仓库，等待 ArgoCD 同步完成。</p><p>这种办法虽然有效，但沟通效率低且容易出错，<strong>我们需要一种自动化的机制来替代这个过程</strong>。</p><!-- [[[read_end]]] --><p>借助 ArgoCD Image Updater，我们可以让 ArgoCD 自动监控镜像仓库的更新情况，一旦工作负载的镜像版本有更新，ArgoCD 就会自动将工作负载升级为新的镜像版本，并且还可以自动将镜像的版本号回写到 Helm Chart 仓库中，保持应用定义和集群状态的一致性。</p><p>这节课，我会进一步改造在上一节课创建的 GitOps 工作流，并加入 ArgoCD Image Updater，实现自动监听镜像变更以及回写 Helm Chart。</p><p>在开始今天的学习之前，你需要做好如下准备。</p><ul>
<li>按照第一章<a href="https://time.geekbang.org/column/article/612571">第 2 讲</a>的内容在本地配置好 Kind 集群，安装 Ingress-Nginx，并暴露 80 和 443 端口。</li>
<li>配置好 Kubectl，使其能够访问 Kind 集群。</li>
<li>按照上一节课的内容在集群内安装 ArgoCD 和 CLI 工具。</li>
<li>克隆 <a href="https://github.com/lyzhang1999/kubernetes-example">kubernetes-example</a> 示例应用代码并推送到自己的 GitHub 仓库中，然后按照<a href="https://time.geekbang.org/column/article/622743">第 16 讲</a>的内容配置好 GitHub Action 和 DockerHub Registry。</li>
<li>将示例应用 .github/workflows/argocd-image-updater.yaml 文件的 env.DOCKERHUB_USERNAME 字段修改为你的 DockerHub 用户名。</li>
</ul><h2>工作流总览</h2><p>在正式进入实战之前，我先简单介绍一下我们最终要实现的效果，如下图所示。</p><p><img src="https://static001.geekbang.org/resource/image/c2/27/c23e00e27754a19a91321c85bfba5e27.jpg?wh=1920x1400" alt="图片"></p><p>相比较上一节课的 GitOps 工作流，这节课实现的效果主要有下面两个差异。</p><ol>
<li>将一个 Git 仓库拆分成了两个，一个存放源码，一个存放 Helm Chart。</li>
<li>不再使用 CI 更新 Helm Chart 镜像版本，而是使用 ArgoCD Image Updater 来自动监控镜像仓库的变更。</li>
</ol><p>此外，由于在日常开发中，我们一般会采用多分支进行开发，这就随时可能产生新的镜像版本。为了将开发过程和需要发布到生产环境的镜像区分开，我们会为 Main 分支构建出来的镜像增加一个 Prefix 标识，例如 <code>main-${commit_Id}</code>，并配置 ArgoCD Image Updater 只监控包含特定标识的镜像版本。</p><p>最终实现的效果是，当开发将代码提交到 Git 仓库 Main 分支后，将触发自动构建，并将新的镜像版本推送到镜像仓库。ArgoCD Image Updater 会以 Poll 的方式每 2 分钟检查一次工作负载的镜像是否有新的版本，如果有，那么就将工作负载的镜像更新为最新版本，并将镜像版本号写入到存放 Helm Chart 的仓库中。</p><h2>安装和配置 ArgoCD Image Updater</h2><p>监听镜像版本的变更需要用到 ArgoCD Image Updater，而它要求和 ArgoCD 一起协同工作，所以在安装之前，请务必先确保在集群内已经安装好 ArgoCD。</p><h3>安装 ArgoCD Image Updater</h3><p>你可以通过下面的命令进行安装。</p><pre><code class="language-plain">kubectl apply -n argocd -f https://ghproxy.com/https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/manifests/install.yaml
serviceaccount/argocd-image-updater created
role.rbac.authorization.k8s.io/argocd-image-updater created
rolebinding.rbac.authorization.k8s.io/argocd-image-updater created
......
</code></pre><h3>创建 Image Pull Secret（可选）</h3><p>由于 ArgoCD 会主动 Poll 镜像仓库来检查是否存在新版本，如果你使用的是私有镜像仓库，那么你需要创建 Secret 对象，以便为 ArgoCD 提供访问镜像仓库的权限。</p><p>以 DockerHub 仓库为例，执行下面的命令来创建 Secret 对象。</p><pre><code class="language-plain">$ kubectl create -n argocd secret docker-registry dockerhub-secret \
&nbsp; --docker-username $DOCKER_USERNAME \
&nbsp; --docker-password $DOCKER_PERSONAL_TOKEN \
&nbsp; --docker-server "https://registry-1.docker.io"

secret/dockerhub-secret created
</code></pre><p>注意将 <code>$DOCKER_USERNAME</code> 和 <code>$DOCKER_PERSONAL_TOKEN</code> 替换为 Docker Hub 用户名和个人凭据。</p><p>如果你忘记如何创建 Docker Personal Token 了，可以查看<a href="https://time.geekbang.org/column/article/622743">第 16 讲</a>的内容。另外关于如何为其他镜像仓库类型配置凭据，你可以查看<a href="https://argocd-image-updater.readthedocs.io/en/stable/configuration/registries/">这份文档</a>。</p><h2>创建 Helm Chart 仓库</h2><p>接下来，我们需要为示例应用的 helm 目录单独创建一个 Git 仓库，在将 <a href="https://github.com/lyzhang1999/kubernetes-example">kubernetes-example</a>克隆到本地后，执行下面的命令。</p><pre><code class="language-plain"> $ cp -r ./kubernetes-example/helm ./kubernetes-example-helm
</code></pre><p>然后，进入 kubernetes-example-helm 目录并初始化 Git。</p><pre><code class="language-plain">$ cd kubernetes-example-helm &amp;&amp; git init
</code></pre><p>前往 GitHub 创建一个新的仓库，将其命名为 kubernetes-example-helm。</p><p><img src="https://static001.geekbang.org/resource/image/5a/4c/5a8502bba504a6597d539b7298304e4c.png?wh=1484x1084" alt="图片"></p><p>将 kubernetes-example-helm 提交到远端仓库中。</p><pre><code class="language-plain">$ git add .
$ git commit -m "first commit"
$ git branch -M main
$ git remote add origin https://github.com/lyzhang1999/kubernetes-example-helm.git
$ git push -u origin main
</code></pre><h2>创建 ArgoCD Application</h2><p>创建好 kubernetes-example-helm 仓库之后，接下来我们需要使用它创建一个新的应用。</p><h3>删除旧应用（可选）</h3><p>在正式创建新的应用之前，为了避免 Ingress 策略冲突，如果你已经按照上节课的内容创建了 ArgoCD example 应用，需要先删除应用及其资源，你可以使用下面的命令来删除应用。</p><pre><code class="language-plain">$ argocd app delete example --cascade
</code></pre><h3>配置仓库访问权限</h3><p>此外，上节课我们创建 ArgoCD 应用时，虽然同样配置了仓库访问权限，但这里的步骤额外还实现了一个重要的功能：为 ArgoCD Image Updater 提供回写 kubernetes-example-helm 仓库的权限。</p><p>要配置仓库访问权限，你可以使用 argocd repo add 命令。</p><pre><code class="language-plain">$ argocd repo add https://github.com/lyzhang1999/kubernetes-example-helm.git --username $USERNAME --password $PASSWORD
Repository 'https://github.com/lyzhang1999/kubernetes-example-helm.git' added
</code></pre><p>注意要将仓库地址修改为你新创建的用于存放 Helm Chart 的 GitHub 仓库地址，并将 <code>$USERNAME</code> 替换为 GitHub 账户 ID，将 <code>$PASSWORD</code> 替换为 GitHub Personal Token。你可以在<a href="https://github.com/settings/tokens">这个页面</a>创建 GitHub Personal Token，并赋予仓库相关权限。</p><h3>创建 ArgoCD 应用</h3><p>接下来我们正式创建 ArgoCD 应用。在上一节课中，我们是使用 argocd app create 命令创建的 ArgoCD 应用 。实际上，它会创建一个特殊类型的资源，也就是 ArgoCD Application，它和 K8s 其他标准的资源对象一样，也是使用 YAML 来定义的。</p><p>在这里，我们直接使用 YAML 来创建新的 Application，将下面的文件内容保存为 application.yaml。</p><pre><code class="language-yaml">apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: example
  annotations:
    argocd-image-updater.argoproj.io/backend.allow-tags: regexp:^main
    argocd-image-updater.argoproj.io/backend.helm.image-name: backend.image
    argocd-image-updater.argoproj.io/backend.helm.image-tag: backend.tag
    argocd-image-updater.argoproj.io/backend.pull-secret: pullsecret:argocd/dockerhub-secret
    argocd-image-updater.argoproj.io/frontend.allow-tags: regexp:^main
    argocd-image-updater.argoproj.io/frontend.helm.image-name: frontend.image
    argocd-image-updater.argoproj.io/frontend.helm.image-tag: frontend.tag
    argocd-image-updater.argoproj.io/frontend.pull-secret: pullsecret:argocd/dockerhub-secret
    argocd-image-updater.argoproj.io/image-list: frontend=lyzhang1999/frontend, backend=lyzhang1999/backend
    argocd-image-updater.argoproj.io/update-strategy: latest
    argocd-image-updater.argoproj.io/write-back-method: git
spec:
  destination:
    namespace: gitops-example-updater
    server: https://kubernetes.default.svc
  project: default
  source:
    path: .
    repoURL: https://github.com/lyzhang1999/kubernetes-example-helm.git
    targetRevision: main
  syncPolicy:
    automated: {}
    syncOptions:
      - CreateNamespace=true
</code></pre><p>然后，使用 kubectl apply 命令创建 ArgoCD Application，效果等同于使用 argocd app create 命令创建应用。</p><pre><code class="language-yaml">$ kubectl apply -n argocd -f application.yaml
application.argoproj.io/example created
</code></pre><p>ArgoCD Image Updater 通过 Application Annotations 标签来实现对应的功能，我简单解释一下每一个标签的作用。</p><ul>
<li>argocd-image-updater.argoproj.io/image-list：指定需要监听的镜像，这里我们指定示例应用的前后端镜像 lyzhang1999/frontend 和 lyzhang1999/backend，同时为前后端镜像法指定了别名，分别为 frontend 和 backend。这里的别名非常重要，会影响下面所有的设置。</li>
<li>argocd-image-updater.argoproj.io/update-strategy：指定镜像更新策略。注意，<strong>latest 并不代表监听 latest 镜像版本</strong>，而是以最新推送的镜像作为更新策略。此外，semver 策略可以识别最高语义化版本的标签，digest 策略可以用来区分同一 Tag 下不同镜像 digest 的变更。</li>
<li>argocd-image-updater.argoproj.io/write-back-method：表示将镜像版本回写到镜像仓库。注意，这里对仓库的写权限来源于使用 argocd repo add 命令为 ArgoCD 配置的仓库访问权限。</li>
<li>argocd-image-updater.argoproj.io/&lt;镜像别名&gt;.pull-secret：为不同的镜像别名指定镜像拉取凭据。</li>
<li>argocd-image-updater.argoproj.io/&lt;镜像别名&gt;.allow-tags：配置符合更新条件的镜像 Tag，在这里我们使用正则表达式匹配那些镜像 Tag 以 main 开头的镜像版本，其他镜像版本则忽略。</li>
<li>argocd-image-updater.argoproj.io/&lt;镜像别名&gt;.helm.image-name：配置 Helm Chart values.yaml 镜像名称所在的节点，在示例应用中，backend.image 和 frontend.image 是values.yaml 配置镜像名称的节点，ArgoCD 在回写仓库时会覆盖这个值。</li>
<li>argocd-image-updater.argoproj.io/&lt;镜像别名&gt;.helm.image-tag：配置 Helm Chart values.yaml 镜像版本所在的节点，在示例应用中，backend.tag 和 frontend.tag 是 values.yaml 配置镜像版本的节点，ArgoCD 在回写仓库时将会覆盖这个值。</li>
</ul><h2>体验 GitOps 工作流</h2><p>接下来，你可以尝试修改 frontend/src/App.js 文件，例如修改文件第 49 行的“Hi! I am a geekbang”内容。修改完成后，<strong>将代码推送到 GitHub 的 main 分支</strong>。</p><p>此时会触发两个 GitHub Action 工作流。其中，当 build-every-branch 工作流被触发时，它将构建 Tag 为 main 开头的镜像版本，并将其推送到镜像仓库中，如下图所示。</p><p><img src="https://static001.geekbang.org/resource/image/a5/66/a571d8f3f53ce2547d8e9df97b789b66.png?wh=1920x407" alt="图片"></p><p>和我们上一节课介绍的另一个 GitHub Action 工作流不同的是，它也不会去主动修改 kubernetes-example-helm 仓库的 values.yaml 文件，在完成镜像推送后工作流也就结束了。</p><p>与此同时，ArgoCD Image Updater 将会每 2 分钟从镜像仓库检索 frontend 和 backend 的镜像版本，一旦发现有新的并且以 main 开头的镜像版本，它将自动使用新版本来更新集群内工作负载的镜像，并将镜像版本回写到 kubernetes-example-helm 仓库。</p><p>在回写时，ArgoCD Image Updater 并不会直接修改仓库的 values.yaml 文件，而是会创建一个专门用于覆盖 Helm Chart values.yaml 的 .argocd-source-example.yaml 文件。</p><p><img src="https://static001.geekbang.org/resource/image/15/0e/15df9ed2a0d349c7b6448fe0be99cb0e.png?wh=1818x522" alt="图片"></p><p>当我们看到这个文件时，说明 ArgoCD Image Updater 已经触发了镜像更新，并且成功将镜像版本回写到了镜像仓库。同时，这个文件记录了详细的覆盖 values.yaml 值的策略。</p><pre><code class="language-yaml">helm:
&nbsp; parameters:
&nbsp; - name: frontend.image
&nbsp; &nbsp; value: lyzhang1999/frontend
&nbsp; &nbsp; forcestring: true
&nbsp; - name: frontend.tag
&nbsp; &nbsp; value: main-b99bc73
&nbsp; &nbsp; forcestring: true
&nbsp; - name: backend.image
&nbsp; &nbsp; value: lyzhang1999/backend
&nbsp; &nbsp; forcestring: true
&nbsp; - name: backend.tag
&nbsp; &nbsp; value: main-b99bc73
&nbsp; &nbsp; forcestring: true
</code></pre><p>这样，当 ArgoCD 在做自动同步时，会将这份文件的内容覆盖 values.yaml 对应的值，比如 frontend.tag 的值会被覆盖为 main-b99bc73，这样，回写后的 Helm Chart 和集群内资源对象就仍然能够保持一致性。</p><p>到这里，我们就完成了通过监听新镜像版本来触发 GitOps 工作流的整个过程。</p><h2>总结</h2><p>在这节课，我为你介绍了如何使用 ArgoCD Image Updater 实现自动监听镜像版本并触发 GitOps 工作流。</p><p>在这个例子中，我们将应用源码和应用定义（Helm Chart）拆分成了两个仓库，在开发和发布角色相对独立的研发流程下，这种方式既能够保持他们之间职责独立，权责清晰，同时还保留了开发随时发布生产环境的能力。</p><p>值得注意的是，在实际的业务场景中，我们一般会使用多分支的模式来开发。这意味着每个分支的每个提交都会产生新的镜像版本，所以，为了区分开发过程的镜像和需要被发布到生产环境的镜像，我在这节课的例子中约定了以 main 开头的镜像版本即为需要发布到生产环境的镜像版本。你可以根据项目的实际情况做调整，例如使用诸如 v1.0.0 的版本号来区分，同时更新 argocd-image-updater.argoproj.io/&lt;镜像别名&gt;.allow-tags 字段的正则表达式。</p><p>总结来说，在这种仓库分离的场景下，要将开发者的提交和发布过程自动化地连接起来，双方需要重点关注生产环境的镜像版本策略，并将它配置到 ArgoCD 应用的 Annotations 注解内，这样便能够让 ArgoCD Image Updater 代替人工来自动监控需要发布到生产环境的镜像。</p><h2>思考题</h2><p>最后，给你留一道思考题吧。</p><p>如果在 Helm Chart 内使用了固定的 latest 镜像版本，并且在 CI 过程也只会覆盖更新 latest 版本的镜像，这种场景下如何配置 ArgoCD Image Updater 的 update-strategy 的策略呢？</p><p>提示：当新的镜像覆盖 latest 版本后，digest 会产生变化。</p><p>欢迎你给我留言交流讨论，你也可以把这节课分享给更多的朋友一起阅读。我们下节课见。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/a3/49/4a488f4c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>农民园丁</span>
  </div>
  <div class="_2_QraFYR_0">老师，我用的是自建的gitlab和harbor（自签名证书），CI部分没有问题，build了main开头的镜像并推送到了harbor中，但是CD部分不能更新应用，在pod argocd-image-updater中的日志如下：<br>time=&quot;2023-02-02T06:11:46Z&quot; level=error msg=&quot;Could not get tags from registry: Get \&quot;https:&#47;&#47;harbor.imustyckz.com&#47;v2&#47;\&quot;: x509: certificate signed by unknown authority&quot; alias=frontend application=example image_name=richey&#47;frontend image_tag=95717a65 registry=harbor.imustyckz.com<br>time=&quot;2023-02-02T06:11:46Z&quot; level=error msg=&quot;Could not get tags from registry: Get \&quot;https:&#47;&#47;harbor.imustyckz.com&#47;v2&#47;\&quot;: x509: certificate signed by unknown authority&quot; alias=backend application=example image_name=richey&#47;backend image_tag=95717a65 registry=harbor.imustyckz.com<br>time=&quot;2023-02-02T06:11:46Z&quot; level=info msg=&quot;Processing results: applications=1 images_considered=2 images_skipped=0 images_updated=0 errors=2&quot;<br><br>看提示是这个pod没有信任自签名证书，搞了2天还不行，请教怎么解决呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Harbor 需要通过 Cert-manager 颁发 TLS 证书，如果还是有错误的话，可以在 argocd-image-updater 项目提交一个 issue：https:&#47;&#47;github.com&#47;argoproj-labs&#47;argocd-image-updater&#47;issues</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-02 14:18:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/76/f5/e3f5bd8d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>宝仔</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，这种helm chart gitops 是不是不同的环境的values文件都要存在git仓库里？比方说我预发环境有staging-values.yaml文件，生产有production-values.yaml文件，因为会存在感配置的情况</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 每个环境所需要的配置文件是不同的，比如测试环境你可能用集群内的数据库，生产环境用云数据库。<br>配置文件并不是一定要存放在 Git 仓库中，比如你可以在创建 ArgoCD 应用的时候配置额外的环境参数。<br>但我建议将不同环境的配置都存储到仓库里，优点有很多，最大的优点是由于 Git 仓库是唯一可信源，所以在未来你要做环境迁移的时候，可以随时拉起一套环境。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-02 09:34:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/ab/ca/bb1ebf5d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>m1k3</span>
  </div>
  <div class="_2_QraFYR_0">argocd-image-updater.argoproj.io&#47;update-strategy：digest 策略可以用来区分同一 Tag 下不同镜像 digest 的变更。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍🏻正确！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-31 09:26:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTIFEQibKmCdPwDP9CcOuEdVNiatE0ekwBZKt5utJhzDKlZiciaEnRTN48eoNzXZpXTDEXMkwU3GQbSDLQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_06ea70</span>
  </div>
  <div class="_2_QraFYR_0">老师，我镜像版本是用$CI_COMMIT_SHORT_SHA打的，镜像更新策略用的latest，但ArgoCD Image Updater发现最新的镜像是按$CI_COMMIT_SHORT_SHA的ASCII码取的，而不是最后推送到镜像仓库的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Latest 策略会拿最近一次推送到镜像仓库的镜像来进行更新。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-18 17:08:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/b9/9b/d2989ff5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>HZI.HUI</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，通常情况下，我们可能需要对部署结果进行通知，比如通知到钉钉里。我用argocd notifications 做了通知，但是这只知道应用的同步结果，无法做到具体哪个pod 部署结果的通知，可以提供下有什么解决思路吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以参考这里定制通知模板：https:&#47;&#47;argocd-notifications.readthedocs.io&#47;en&#47;stable&#47;templates&#47;</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-15 16:59:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/6e/fe/79955244.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>LbbNiu</span>
  </div>
  <div class="_2_QraFYR_0">为什么按专栏的操作部署后，frontend 的Pod数一直在增加呢，增加到了10个pod</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可能是 frontend 的 hpa 表现差异导致的，你可以尝试修改 hpa 对象的阈值或者先临时删除它。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-02 11:26:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/b9/9b/d2989ff5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>HZI.HUI</span>
  </div>
  <div class="_2_QraFYR_0">ArgoCD Image Updater  默认每2分钟检索一次镜像是否需要更新。这个时间可以可以自定义设置吗？或者是否可以改为webhook推送的方式？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以自定义镜像拉取的间隔时间，具体方法是为 Argocd image updater 的工作负载添加启动参数 --interval 并指定时间，参考这里：https:&#47;&#47;github.com&#47;argoproj-labs&#47;argocd-image-updater&#47;blob&#47;master&#47;docs&#47;install&#47;reference.md#flags</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-22 10:54:52</div>
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
  <div class="_2_QraFYR_0">对于文中的<br>“ArgoCD Image Updater 会以 Poll 的方式每 2 分钟检查一次工作负载的镜像是否有新的版本，如果有，那么就将工作负载的镜像更新为最新版本，并将镜像版本号写入到存放 Helm Chart 的仓库中。”<br>“当 ArgoCD 在做自动同步时，会将这份文件的内容覆盖 values.yaml 对应的值，比如 frontend.tag 的值会被覆盖为 main-b99bc73，这样，回写后的 Helm Chart 和集群内资源对象就仍然能够保持一致性”<br>这两部分说明，该怎么理解？是<br>a. ArgoCD先更新Chart仓库，也就是添加&#47;更新.argocd-source-example.yaml，然后使用Chart原本的所有文件加上这个.argocd-source-example.yaml计算出工作负载对象的预期状态，以此去与k8s集群中的工作负载做对比和更新<br>还是<br>b. 先更新k8s集群中的工作负载，然后再更新Chart仓库中的文件<br>个人理解应该是a，因为ArgoCD应该是对比应用定义和k8s中的工作负载，应用定义的变更可能有多种，不止是镜像版本的自动更新，还可能有其他人工手动做的变更，ArgoCD应该是都能监控、比对、更新。但是，从上边两段描述看，感觉像是b……请您指教</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 先更新集群工作负载的镜像版本，再回写。<br>换个角度，你可以这么考虑，回写仓库是可选的。如果不回写，那么 image updater 为了更新工作负载的镜像，就必须要修改集群内 Manifest image 字段。<br>这样就清楚了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-21 18:21:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/cBh6rmNsSIbHEAGKiaq25yz9tqGuJEjbIYn2K0uFBLEe8lBNjL3SUOicibPbAO5SdH6TxV65kcCpK6FOB1hBr3PBQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>gyl1989113</span>
  </div>
  <div class="_2_QraFYR_0">问下老师build-every-branch 的触发逻辑写到哪的呢。。build.yaml里没找到呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这部分在 argocd-image-updater.yaml 文件下定义了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-14 11:06:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/2a/ff/a9d72102.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>BertGeek</span>
  </div>
  <div class="_2_QraFYR_0">非常好的专栏，及时雨呀！<br>上周只是基本测试了argocd demo，在阿里云ack 中的使用，<br>1. jenkins 构建镜像推送到harbor 仓库，然后改动对应应用 yaml文件中的镜像版本<br>2. argocd 手动sync ，更新应用deployment 镜像版本<br><br>本文提到的  ArgoCD Image Updater  ，更智能和高级的处理了这部分镜像变更。<br><br><br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢认可。<br>是的，image updater 是更安全和更高效的做法。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-09 17:49:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/e9/26/afc08398.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Amosヾ</span>
  </div>
  <div class="_2_QraFYR_0">如果紧急或特殊情况去集群修改了 deploy，是否可以回写到仓库呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 没办法。试想一下，你的 Deployment 可能是通过 Helm 渲染出来的，如果修改了集群的 Deployment，ArgoCD 是没法知道要改 Helm Chart 的哪个字段的。<br>GitOps 的核心理念是 Git 作为唯一可信源，任何修改都应该产生提交记录，以便回溯。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-01 00:29:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/27/ff/e4/927547a9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>无名无姓</span>
  </div>
  <div class="_2_QraFYR_0">应该会触发吧</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-30 09:08:29</div>
  </div>
</div>
</div>
</li>
</ul>