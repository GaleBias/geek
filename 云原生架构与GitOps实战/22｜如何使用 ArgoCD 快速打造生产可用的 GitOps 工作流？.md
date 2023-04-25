<audio title="22｜如何使用 ArgoCD 快速打造生产可用的 GitOps 工作流？" src="https://static001.geekbang.org/resource/audio/79/ae/795f5b4580fc78f8cba53026d6430cae.mp3" controls="controls"></audio> 
<p>你好，我是王炜。</p><p>在之前的课程中，我们学习了如何使用 Kustomize 和 Helm 和来定义应用。其中，Helm Chart在实际工作中使用较多，它有两种存储方式，一种是以源码的方式存储在 Git 仓库中，另一种是以 tgz 压缩包的方式存储在专用的 OCI 仓库中。</p><p>此外，在自动化镜像构建部分，我还向你介绍了如何使用 GitHub Action、GitLab CI 以及自托管 Tekton 来自动构建镜像。</p><p>当我们具备这些 GitOps 的核心基础后，接下来我们就可以根据实际场景，选择合适的技术栈来构建 GitOps 流水线了。</p><p>在这节课，我仍然会以示例应用为例，使用 GitHub Action 和 Helm 分别作为自动构建镜像和应用定义的工具，并通过 ArgoCD 来构建一个完整的 GitOps 工作流。</p><p>在开始今天的学习之前，你需要准备好下面这几个条件。</p><ul>
<li>按照<a href="https://time.geekbang.org/column/article/612571">第 2 讲</a>的内容在本地配置好 Kind 集群，安装 Ingress-Nginx，<strong>并暴露 80 和 443 端口。</strong></li>
<li>配置好 Kubectl，使其能够访问 Kind 集群。</li>
<li>克隆 <a href="https://github.com/lyzhang1999/kubernetes-example">kubernetes-example</a> 示例应用代码并推送到自己的 GitHub 仓库中，然后按照<a href="https://time.geekbang.org/column/article/622743">第 16 讲</a>的内容配置好 GitHub Action 和 DockerHub Registry。</li>
</ul><!-- [[[read_end]]] --><h2>ArgoCD</h2><p>在“从零上手 GitOps”这一章，我使用了 FluxCD 来构建 GitOps 流水线。FluxCD 的主要特点是比较轻量，但同时也缺少友好的 UI 控制台。相比较而言，在社区和维护方面，ArgoCD 更为活跃。所以，<strong>在生产环境下，我推荐你使用 ArgoCD 来构建 GitOps 工作流</strong>。</p><h3>安装 ArgoCD</h3><p>要使用 ArgoCD，首先需要在 Kind 集群安装它，你可以通过下面的命令来安装。</p><p>首先，创建 argocd 命名空间。</p><pre><code class="language-powershell">$ kubectl create namespace argocd
namespace/argocd created
</code></pre><p>然后，部署 ArgoCD。</p><pre><code class="language-plain">$ kubectl apply -n argocd -f https://ghproxy.com/https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
customresourcedefinition.apiextensions.k8s.io/applications.argoproj.io created
customresourcedefinition.apiextensions.k8s.io/applicationsets.argoproj.io created
customresourcedefinition.apiextensions.k8s.io/appprojects.argoproj.io created
......
</code></pre><p>最后，等待 argocd 命名空间所有的工作负载处于就绪状态。</p><pre><code class="language-plain">$ kubectl wait --for=condition=Ready pods --all -n argocd --timeout 300s
pod/argocd-application-controller-0 condition met
pod/argocd-applicationset-controller-57bfc6fdb8-x5jxc condition met
......
</code></pre><p>请注意，由于云厂商 Kubernetes 版本存在差异，所以如果你在安装过程中发现 argocd-repo-server 工作负载一直无法启动，可以尝试删除 argocd-repo-server Deployment seccompProfile 节点的内容。</p><pre><code class="language-plain">apiVersion: apps/v1
kind: Deployment
metadata:
  name: argocd-repo-server
spec:
  template:
    spec:
      securityContext:
        seccompProfile:
          type: RuntimeDefault
</code></pre><h3>安装 ArgoCD CLI</h3><p>为了更加方便地配置 ArgoCD，官方还为我们提供了 CLI 工具。不同操作系统的安装方法有所差异，这里以 MacOS 和 Windows 为例简单介绍。</p><p>MacOS 可以通过 Brew 来安装。</p><pre><code class="language-plain">$ brew install argocd
</code></pre><p>也可以在<a href="https://github.com/argoproj/argo-cd/releases">这个链接</a>下载最新版本的 Binary 二进制文件，并移动到 /usr/local/bin/ 目录下。</p><p>Windows 则需要在下载过后，将可执行文件移动到 PATH 下。</p><p>更详细的方法可以查看<a href="https://argo-cd.readthedocs.io/en/stable/cli_installation/">这份文档。</a></p><h3>本地访问 ArgoCD</h3><p>要在本地访问 ArgoCD，最简单的方式是通过端口转发来完成。你可以使用下面的命令来进行端口转发。</p><pre><code class="language-plain">$ kubectl port-forward service/argocd-server 8080:80 -n argocd
Forwarding from 127.0.0.1:8080 -&gt; 8080
Forwarding from [::1]:8080 -&gt; 8080
</code></pre><p>接下来，使用浏览器打开：<a href="http://127.0.0.1:8080">http://127.0.0.1:8080</a>，这样就可以访问 ArgoCD 控制台了，如下图所示。</p><p><img src="https://static001.geekbang.org/resource/image/de/14/dee91919a11f2b01615738a899eab414.png?wh=1920x1010" alt=""></p><p>ArgoCD 的默认账号为 admin，密码可以通过下面的命令来获取。</p><pre><code class="language-plain">$ kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
Gn4b2PFG6vKm1ADm
</code></pre><p>登录后，即可访问 ArgoCD 的控制台。</p><h2>GitOps 工作流总览</h2><p>到这里，你是不是已经迫不及待想要构建工作流了？别急，在创建 GitOps 工作流之前，我们先来认识一下一个完整 GitOps 工作流都需要哪些关键步骤。</p><p><img src="https://static001.geekbang.org/resource/image/d5/1a/d597b74ec83be42c2308d251c7868c1a.jpg?wh=1920x604" alt=""></p><p>我们可以把这个完整的 GitOps 工作流分成三个部分来看。</p><p>第一部分是开发者推送代码到 GitHub 仓库，然后触发 GitHub Action 自动构建。</p><p>第二部分是 GitHub Action 自动构建，它包括下面三个步骤。</p><pre><code>1. 构建示例应用的镜像。
2. 将示例应用的镜像推送到 Docker Registry 镜像仓库。
3. 更新代码仓库中 Helm Chart values.yaml 文件的镜像版本。
</code></pre><p>第三部分的核心是 ArgoCD，它包括下面两个步骤。</p><pre><code>4. 通过定期 Poll 的方式持续拉取 Git 仓库，并判断是否有新的 commit。
5. 从 Git 仓库获取 Kubernetes 对象，与集群对象进行实时比较，自动更新集群内有差异的资源。
</code></pre><p>在之前的课程中，我们已经为示例应用创建好了 GitHub Action 来自动构建镜像，但还缺少自动更新 Helm Chart values.yaml 文件的镜像版本逻辑，我会在稍后进行配置。</p><p>现在，我们开始创建 GitOps 工作流中的第三部分，也就是创建 ArgoCD 应用，实现 Kubernetes 资源的自动同步。</p><h2>创建 ArgoCD 应用</h2><p>我们以示例应用为例子来创建 ArgoCD 应用，这里主要分成两个步骤。</p><ol>
<li>配置仓库访问权限。</li>
<li>创建 ArgoCD 应用。</li>
</ol><p>其中，如果你的示例应用仓库是公开的，可以跳过第一步。</p><h3>配置 ArgoCD 仓库访问权限（可选）</h3><p>在实际场景下，我们存放应用定义的仓库一般都是私有仓库，这就需要为 ArgoCD 配置仓库访问权限。</p><p>你可以通过下面的 ArgoCD CLI 工具来为 ArgoCD 添加仓库访问权限。</p><p>在使用 ArgoCD CLI 工具之前，你需要先执行 argocd login 命令登录。</p><pre><code class="language-plain">$ argocd login 127.0.0.1:8080 --insecure
Username: admin
Password:
'admin:login' logged in successfully
</code></pre><p>注意，这里我们指定了 ArgoCD 的服务端地址为 127.0.0.1:8080，并且使用了 --insecur 参数来跳过 SSL 认证，你需要保持在上面运行的端口转发命令才能够顺利登录。</p><p>登录成功后，通过 argocd repo add 命令添加你的示例应用仓库。</p><pre><code class="language-plain">$ argocd repo add https://github.com/lyzhang1999/kubernetes-example.git --username $USERNAME --password $PASSWORD
Repository 'https://github.com/lyzhang1999/kubernetes-example.git' added
</code></pre><p>这里要注意将仓库地址修改为你实际的 GitHub 仓库地址，并将 <code>$USERNAME</code> 替换为 GitHub 账户 ID，将 <code>$PASSWORD</code> 替换为 GitHub Personal Token。你可以在<a href="https://github.com/settings/tokens">这个页面</a>创建 GitHub Personal Token，并赋予仓库相关权限，如下图所示。</p><p><img src="https://static001.geekbang.org/resource/image/ba/35/ba7703a5429af8f1b0ccf34174114035.png?wh=1608x1056" alt="图片"></p><h3>创建 ArgoCD 应用</h3><p>接下来，就可以创建 ArgoCD 应用了。ArgoCD 同时支持使用 Helm Chart、Kustomize 和 Manifest 来创建应用，这里我们以示例应用的 Helm Chart 为例。</p><p>你可以通过 argocd app create 命令来创建应用。</p><pre><code class="language-plain">$ argocd app create example --sync-policy automated --repo https://github.com/lyzhang1999/kubernetes-example.git --revision main --path helm --dest-namespace gitops-example --dest-server https://kubernetes.default.svc --sync-option CreateNamespace=true
application 'example' created
</code></pre><p>这里我简单解释一下每个参数的作用。</p><p>–sync-policy 参数代表设置自动同步策略。automated 的含义是自动同步，也就是说当集群内的资源和 Git 仓库 Helm Chart 定义的资源有差异时，ArgoCD 会自动执行同步操作，实时确保集群资源和 Helm Chart 的一致性。</p><p>–repo 参数表示 Helm Chart 的仓库地址。这里的值是示例应用的仓库地址，注意需要替换成你实际的 Git 仓库地址。</p><p>–revision 参数表示需要跟踪的分支或者 Tag，这里我们让 ArgoCD 跟踪 main 分支的改动。</p><p>–path 参数表示 Helm Chart 的路径。在示例应用中，存放 Helm Chart 的目录是 helm 目录。</p><p>–dest-namespace 参数表示命名空间。这里指定了 gitops-example 命名空间，注意，这是一个不存在的命名空间，所以我们额外通过 --sync-option 参数来让 ArgoCD 自动创建这个命名空间。</p><p>最后，–dest-server 参数表示要部署的集群，<a href="https://kubernetes.default.svc">https://kubernetes.default.svc</a> 表示 ArgoCD 所在的集群。</p><h3>查看 ArgoCD 同步状态</h3><p>创建好应用之后，GitOps 工作流中的自动同步部分也就建立起来了。现在，你可以打开 ArgoCD 控制台，进入左侧的“Application”菜单来查看示例应用详情。</p><p><img src="https://static001.geekbang.org/resource/image/23/34/23df1a812a1a4ea33ddaf3d63ba97b34.png?wh=4320x2272" alt="图片"></p><p>在应用详情页面，我们需要重点关注三个状态。</p><ol>
<li>
<p><strong>APP HEALTH：</strong>应用整体的健康状态，它包含下面三个值。</p>
<ol>
<li>Progressing：处理中</li>
<li>Healthy：健康状态</li>
<li>Degraded：宕机</li>
</ol>
</li>
<li>
<p><strong>CURRENT SYNC STATUS：</strong> 应用定义和集群对象的差异状态，也包含下面三个值。</p>
<ol>
<li>Synced：完全同步</li>
<li>OutOfSync：存在差异</li>
<li>Unknown：未知</li>
</ol>
</li>
<li>
<p><strong>LAST SYNC RESULT：</strong>最后一次同步到 Git 仓库的信息，包括 Commit ID 和提交者信息。</p>
</li>
</ol><h3>访问应用</h3><p>当应用健康状态变为 Healthy 之后，我们就可以访问应用了。</p><p>在这之前，如果你已经在 example 命名空间下手动部署了示例应用，为了避免 Ingress 策略冲突，你需要先删除这个命名空间。</p><pre><code class="language-plain">$ kubectl delete ns example
</code></pre><p>然后，使用浏览器访问 <a href="http://127.0.0.1">http://127.0.0.1</a>，你应该能看到示例应用的界面，如下图所示。</p><p><img src="https://static001.geekbang.org/resource/image/5b/58/5b67ae11fa94d69132955524d9dcyy58.png?wh=4320x2272" alt="图片"></p><p>到这里，ArgoCD 部分就配置完成了。</p><h2>连接 GitOps 工作流</h2><p>在完成 ArgoCD 的应用配置之后，我们就已经将示例应用的 Helm Chart 定义和集群资源关联起来了，但整个 GitOps 工作流还缺少非常重要的一部分，就是我在上面提到的自动更新 Helm Chart values.yaml 文件镜像版本的部分，我在下面这张示意图中用“❌”把这个环节标记了出来。</p><p><img src="https://static001.geekbang.org/resource/image/d2/0f/d214c5416ab33f2f876b74baecf4550f.jpg?wh=1920x607" alt=""></p><p>在这部分工作流没有打通之前，提交的新代码虽然会构建出新的镜像，但是 Helm Chart 定义的镜像版本并不会产生变化，<strong>这会导致 ArgoCD 不能自动更新集群内工作负载的镜像版本</strong>。</p><p>要解决这个问题，我们还需要在 GitHub Action 中添加自动修改 Helm Chart 并重新推送到仓库操作。</p><p>接下来，我们修改示例应用的 .github/workflows/build.yaml 文件，在“Build frontend and push”阶段后面添加一个新的阶段，代码如下。</p><pre><code class="language-powershell">- name: Update helm values.yaml
&nbsp; uses: fjogeleit/yaml-update-action@main
&nbsp; with:
&nbsp; &nbsp; valueFile: 'helm/values.yaml'
&nbsp; &nbsp; commitChange: true
    branch: main
    message: 'Update Image Version to ${{ steps.vars.outputs.sha_short }}'
&nbsp; &nbsp; changes: |
&nbsp; &nbsp; &nbsp; {
&nbsp; &nbsp; &nbsp; &nbsp; "backend.tag": "${{ steps.vars.outputs.sha_short }}",
&nbsp; &nbsp; &nbsp; &nbsp; "frontend.tag": "${{ steps.vars.outputs.sha_short }}"
&nbsp; &nbsp; &nbsp; }
</code></pre><p>在这里，我使用了 GitHub Action 中 yaml-update-action 插件来修改 values.yaml 文件并把它推送到仓库。如果你是使用 GitLab 或者 Tekton 构建镜像，可以调用 yq 命令行工具来修改 YAML 文件，再使用 git 命令行将变更推送到仓库。</p><p>到这里，<strong>一个完整的 GitOps 工作流就建立好了</strong>。</p><h3>体验 GitOps 工作流</h3><p>接下来，你可以尝试修改 frontend/src/App.js 文件，例如修改文件第 49 行的“Hi! I am a geekbang”。修改完成后，<strong>将代码推送到 GitHub 仓库 main 分支</strong>，此时，GitHub Action 会自动构建镜像，并且还会更新代码仓库中 Helm values.yaml 文件的镜像版本。</p><p><img src="https://static001.geekbang.org/resource/image/2d/e7/2d292107a101c522af9fedbabe8339e7.png?wh=1920x845" alt="图片"></p><p>ArgoCD 默认每 3 分钟会拉取仓库检查是否有新的提交，你也可以在 ArgoCD 控制台手动点击 Sync 按钮来触发同步。</p><p><img src="https://static001.geekbang.org/resource/image/be/14/be46c591ddd73yy3da6d4a930ae9c414.png?wh=1920x409" alt="图片"></p><p>ArgoCD 同步完成后，我们可以在“LAST SYNC RESULT”一栏中看到 GitHub Action 修改 values.yaml 的提交记录，当应用状态为 Healthy 时，我们就可以访问新的应用版本了。</p><p><img src="https://static001.geekbang.org/resource/image/d8/e6/d878f196cd14d116bae01fdfc3f6cbe6.png?wh=4320x2272" alt="图片"></p><p>从截图可以看出，前端界面输出内容为“Hi, I am GitOps workflow”，说明 ArgoCD 已经将新版本的应用部署到集群中了。</p><p>自此，我们体验了提交代码到构建镜像、修改应用定义和更新自动化的全流程。</p><h2>生产建议</h2><p>在生产环境下，我也给你提几个 ArgoCD 的配置建议，你可以根据实际情况来配置。</p><h3>建议一：修改默认密码</h3><p>默认情况下，ArgoCD 会在部署时生成一个随机密码，为了方便记忆和增加密码强度，你可以使用下面的命令来修改密码。</p><pre><code class="language-plain">$ argocd account update-password
*** Enter password of currently logged in user (admin):
*** Enter new password for user admin:
</code></pre><p>要注意是的是，在修改密码前，你需要先使用 argocd login 登录到 ArgoCD 服务端。</p><h3>建议二：配置 Ingress 和 TLS</h3><p>在生产环境下，为了更方便地访问 ArgoCD，你可以为它配置 Ingress，你可以将下面的 Ingress 对象部署到集群内。</p><pre><code class="language-plain">apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server-ingress
  namespace: argocd
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    kubernetes.io/ingress.class: nginx
    kubernetes.io/tls-acme: "true"
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  rules:
  - host: argocd.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service: 
            name: argocd-server
            port:
              name: https
  tls:
  - hosts:
    - argocd.example.com
    secretName: argocd-secret # do not change, this is provided by Argo CD
</code></pre><p>此外，你还需要安装 Cert-manager，具体方法可以参考<a href="https://time.geekbang.org/column/article/624150">第 19 讲</a>的内容。</p><h3>建议三：使用 Webhook 触发 ArgoCD</h3><p>在这节课的例子中，我们在创建应用的时候提供了参数 --sync-policy=automated。这时候，ArgoCD 会默认以 3 分钟一次的频率来自动拉取仓库的更改，在生产环境下，这个同步频率可能并不能满足快速发布的要求。</p><p>如果 ArgoCD 可以在公网进行访问，那么你就可以使用 ArgoCD 提供的 Webhook 触发方式来解决这个问题了，如下图所示。</p><p><img src="https://static001.geekbang.org/resource/image/e9/9d/e907726d7f2eded0cc2eb3a923538c9d.jpg?wh=1920x617" alt=""></p><p>和主动 Poll 模型不同的是，源码仓库在收到开发者推送代码的事件后，将实时通过 HTTP 请求来通知 ArgoCD，也就是图中红色字体的部分。</p><p>要使用 Webhook 通知的方式，首先你需要在源码仓库进行配置。以 GitHub 为例，首先进入仓库的“Settings”页面，点击左侧的“Webhook”菜单进入配置页面，如下图所示。</p><p><img src="https://static001.geekbang.org/resource/image/cd/0b/cd2676a447fe66666e5yyd2611ea4b0b.png?wh=1564x1608" alt="图片"></p><p>在 Payload URL 中输入你的 ArgoCD Server 外网访问域名，/api/webhook ArgoCD 专门用于接收外部 Webhook 消息的固定路径。</p><p>Content type 选择 application/json，并在 Secret 中输入你要配置的 Webhook 的密钥，这个密钥需要提供给 ArgoCD 来校验 Webhook 来源是否合法。</p><p>接下来点击“Add webhook” 就可以保存了。</p><p>接下来，你还需要为 ArgoCD 提供 GitHub Webhook 密钥，使用下面的命令来编辑 argocd-secret 对象。</p><pre><code class="language-powershell">$ kubectl edit secret argocd-secret -n argocd
</code></pre><p>并将 GitHub Webhook 的密钥加入到 Secret 对象的内容里。</p><pre><code class="language-yaml">apiVersion: v1
kind: Secret
metadata:
  name: argocd-secret
  namespace: argocd
type: Opaque
data:
...
stringData:
  # 加入这一项
  webhook.github.secret: my-secret
</code></pre><p>注意，stringData 可以直接输入 Webhook Secret 内容而不需要进行 Base64 编码。</p><p>如果你使用的是其他的代码托管平台，例如 GitLab，可以参考<a href="https://argo-cd.readthedocs.io/en/stable/operator-manual/webhook/">这份文档</a>进行配置。</p><h3>建议四：将源码仓库和应用定义仓库分离</h3><p>为了方便演示，我将示例应用的源码和 Helm Chart 存储在了同一个 Git 仓库，实际上，<strong>这并不是一个好的实践。</strong></p><p>这种方案有两个比较大的问题。首先，当我们手动修改 Helm Chart 并推送到 Git 仓库之后，在业务代码不变的情况下也会触发应用镜像构建，这个过程是没有必要的。</p><p>其次，在有一定规模的团队中，开发和发布过程是分开的，应用定义仓库一般只有基础架构部门或者 SRE 部门具有修改权限，将源码和应用定义放在同一个 Git 仓库不利于权限控制，开发者也很容易误操作。</p><p><strong>所以，基于上面这两个问题，我强烈建议你将业务代码和应用定义分开存储</strong>。</p><h3>建议五：加密 GitOps 中存储的秘钥</h3><p>在示例应用中，我使用的是 DockerHub 公开仓库，所以 Kubernetes 集群不需要镜像拉取凭据就可以拉取到镜像。</p><p>在实际生产环境下，一般我们会使用内部自建例如 Harbor 私有仓库。所以在大部分情况下，我们会在 Helm Chart 里增加一个包含镜像拉取凭据的 Secret 对象。</p><pre><code class="language-plain">apiVersion: v1
kind: Secret
metadata:
&nbsp; name: regcred
type: kubernetes.io/dockerconfigjson
data:
&nbsp; .dockerconfigjson: &gt;-
&nbsp; &nbsp; eyJhdXRocyI6eyJodHRwczovL2luZGV4LmRvY2tlci5pby92MS8iOnsidXNlcm5hbWUiOiJseXpoYW5nMTk5OSIsInBhc3N3b3JkIjoibXktdG9rZW4iLCJhdXRoIjoiYkhsNmFHRnVaekU1T1RrNmJYa3RkRzlyWlc0PSJ9fX0=
</code></pre><p>这样，当 ArgoCD 部署应用时，会一并将拉取凭据部署到集群中，这就解决了镜像拉取权限的问题。</p><p>但是，Secret 对象并没有加密功能，<strong>这可能会导致凭据泄露。</strong>所以，我们需要对这些敏感信息进行加密处理。关于如何加密秘钥，我会在后续“多环境管理和安全”章节为你详细介绍。</p><h2>总结</h2><p>在这节课，我以示例应用为例，将 GitHub Action、应用定义以及 ArgoCD 连接起来，构建了完整的 GitOps 工作流。</p><p>通过这个例子我们会发现，建立完整的 GitOps 工作流涉及到下面这几个技术：</p><ol>
<li>Docker 镜像</li>
<li>CI 构建</li>
<li>镜像仓库</li>
<li>应用定义</li>
<li>Kubernetes</li>
<li>ArgoCD</li>
</ol><p>在建立 GitOps 工作流的过程中，通常有两个难点。<strong>第一是如何进行技术选型和组合，第二是如何将它们连接起来。</strong></p><p>我先总结一下如何解决第一个问题：技术选型和组合。</p><p>在这节课的例子中，我使用了 GitHub Action 作为 CI 构建工具，此外，你还可以选择 GitLab CI 或者自托管的 Tekton 。对于镜像仓库，我使用了 DockerHub 来存储镜像，你也可以使用 GitHub Package 或者自建 Harbor 来存储镜像。最后，在应用定义方面，我使用了 Helm Chart 作为例子，你还可以选择 Kustomize 甚至是 Kubernetes Manifest 。</p><p>在建立 GitOps 的过程中，你可以根据团队实际的情况，对这些技术栈任意组合。</p><p>对于如何连接的问题，它最底层的追问是，在构建好镜像并将其推送到镜像仓库之后，怎么通知 ArgoCD 部署新镜像版本。</p><p>在这节课的例子中，我在 GitHub Action 里修改了 values.yaml 文件，并将其推送到了 Git 仓库，所以会产生新的 commit，进而达到通知 ArgoCD 的效果。</p><p><strong>这种方式虽然能解决问题，但在一些场景下可能并不适合</strong>。例如，在开发和发布相对独立的场景下，因为权限和安全的问题可能并不允许 CI 直接修改应用定义。在这种情况下，我们就可以使用另外一种方式来通知 ArgoCD，也就是 <strong>Argo CD Image Updater。</strong>它可以通过监听镜像版本的更新来触发 ArgoCD 同步，我们会在下一节课详细介绍。</p><h2>思考题</h2><p>最后，给你留一道思考题吧。</p><p>在 ArgoCD 自动同步完成后，应用的“CURRENT SYNC STATUS”会从“Synced”很快变为“OutOfSync”，这是为什么呢？如何解决？</p><p>欢迎你给我留言交流讨论，你也可以把这节课分享给更多的朋友一起阅读。我们下节课见。</p>
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
  <div class="_2zFoi7sd_0"><span>邵涵</span>
  </div>
  <div class="_2_QraFYR_0">对于思考题，CURRENT SYNC STATUS 从 Synced 变为 OutOfSync 可能是应用定义中的Deployment的replicas指定的数量与k8s集群中该Deployment下pod的实际数量不符造成的吗？因为集群中Deployment下的pod实际数量可能是HPA动态管控的<br>如果是这个原因的话，那是有什么方法可以声明忽略对这个数量的比对吗？<br><br>另外，还有两个问题，使用ArgoCD做自动的拉取、比对，和更新k8s集群中对象<br>1. 如何指定给某个环境使用的values-xxx.yaml文件？<br>2. 如果是要同时更新多个环境，是要为每个环境都创建一个ArgoCD应用吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 回答正确。解决方案也非常简单，把 Deployment 的 replicas 字段删除即可，这样该字段会由 HPA 的最小副本数控制。<br>另外两个问题提的非常好，你可以参考 27 讲的内容，多环境自动创建的方法。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-20 20:34:34</div>
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
  <div class="_2_QraFYR_0">argocd repo add https:&#47;&#47;github.com&#47;xxx&#47;kubernetes-example.git --username xxx --password xxx 为什么我执行这句报错rpc error: code = Unknown desc = error testing repository connectivity: authentication required</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 尝试用 git 协议，这个仓库是公开的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-13 10:08:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/ac/66/9bc49bcd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>辣条愛乃</span>
  </div>
  <div class="_2_QraFYR_0">请教下，&quot;连接GitOps工作流&quot;这一步，我使用的gitlab，但是似乎没有找到，类似github的&quot;valueFile&quot;用来修改helm里面的tag id，不知道gitlab是怎么修改的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 把 gitlab ci 当做是一台机器，本质上可以通过 shell 命令实现任何操作，比如编辑 yaml 文件可以用 yq 命令行工具。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-07 10:49:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/ac/66/9bc49bcd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>辣条愛乃</span>
  </div>
  <div class="_2_QraFYR_0">自己搭建的gitlab，但是报错<br>FATA[0003] rpc error: code = Unknown desc = error testing repository connectivity: Get &quot;https:&#47;&#47;www.gittest.com&#47;root&#47;kubernetes-example.git&#47;info&#47;refs?service=git-upload-pack&quot;: dial tcp 128.14.151.194:443: connect: connection refused </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 看着是网络原因导致的，你可以尝试用托管的 GitLab 跑流程。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-28 11:54:05</div>
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
  <div class="_2_QraFYR_0">感谢老师的解答，还有一个问题请教一下：<br>我们目前直接使用jenkins做CI&#47;CD，然后我准备把CD工具替换为argocd。<br>关于&quot;自动更新 Helm Chart values.yaml 文件镜像版本的部分&quot;这块实践，看老师的教程可得github、gitlab-ci都可以实现。但是我暂时不想用gitlab-ci做CI工具，继续使用Jenkins做CI操作的话能否实现&quot;自动更新 Helm Chart values.yaml 文件镜像版本的部分&quot;呢？<br>我目前想到的方案就是通过自己写脚本来实现了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 很容易实现，Jenkins 里通过 shell 命令调用 YQ 工具来更新 helm chart，核心原理不变，只要能更新 values.yaml 即可。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-20 17:49:22</div>
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
  <div class="_2_QraFYR_0">请教一下，假如我有3个java服务，每个服务对应3套环境，test,preview,product，那么我是不是就得给这3个服务分别创建3个application，在argocd上。<br><br>有什么好的方式来解决在argocd上重复创建application的问题吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 非常好的问题，你可以查看第27讲，这里面有提到多环境管理。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-15 15:50:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/63/ae/eb536e1d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>bob</span>
  </div>
  <div class="_2_QraFYR_0">老师，您好！请问后端运行报错会是什么原因？谢谢！<br>Traceback (most recent call last):<br>File &quot;&#47;usr&#47;local&#47;lib&#47;python3.8&#47;runpy.py&quot;, line 194, in _run_module_as_main<br>  return _run_code(code, main_globals, None,<br>File &quot;&#47;usr&#47;local&#47;lib&#47;python3.8&#47;runpy.py&quot;, line 87, in _run_code<br>  exec(code, run_globals)<br>File &quot;&#47;usr&#47;local&#47;lib&#47;python3.8&#47;site-packages&#47;flask&#47;__main__.py&quot;, line 3, in &lt;module&gt;<br>  main()<br>File &quot;&#47;usr&#47;local&#47;lib&#47;python3.8&#47;site-packages&#47;flask&#47;cli.py&quot;, line 1050, in main<br>  cli.main()<br>File &quot;&#47;usr&#47;local&#47;lib&#47;python3.8&#47;site-packages&#47;click&#47;core.py&quot;, line 1055, in main<br>  rv = self.invoke(ctx)<br>File &quot;&#47;usr&#47;local&#47;lib&#47;python3.8&#47;site-packages&#47;click&#47;core.py&quot;, line 1657, in invoke<br>  return _process_result(sub_ctx.command.invoke(sub_ctx))<br>File &quot;&#47;usr&#47;local&#47;lib&#47;python3.8&#47;site-packages&#47;click&#47;core.py&quot;, line 1404, in invoke<br>  return ctx.invoke(self.callback, **ctx.params)<br>File &quot;&#47;usr&#47;local&#47;lib&#47;python3.8&#47;site-packages&#47;click&#47;core.py&quot;, line 760, in invoke<br>  return __callback(*args, **kwargs)<br>File &quot;&#47;usr&#47;local&#47;lib&#47;python3.8&#47;site-packages&#47;click&#47;decorators.py&quot;, line 84, in new_func<br>  return ctx.invoke(f, obj, *args, **kwargs)<br>File &quot;&#47;usr&#47;local&#47;lib&#47;python3.8&#47;site-packages&#47;click&#47;core.py&quot;, line 760, in invoke<br>  return __callback(*args, **kwargs)<br>File &quot;&#47;usr&#47;local&#47;lib&#47;python3.8&#47;site-packages&#47;flask&#47;cli.py&quot;, line 911, in run_command<br>  raise e from None<br>File &quot;&#47;usr&#47;local&#47;lib&#47;python3.8&#47;site-packages&#47;flask&#47;cli.py&quot;, line 897, in run_command<br>  app = info.load_app()<br>File &quot;&#47;usr&#47;local&#47;lib&#47;python3.8&#47;site-packages&#47;flask&#47;cli.py&quot;, line 312, in load_app<br>  app = locate_app(import_name, None, raise_if_not_found=False)<br>File &quot;&#47;usr&#47;local&#47;lib&#47;python3.8&#47;site-packages&#47;flask&#47;cli.py&quot;, line 218, in locate_app<br>  __import__(module_name)<br>File &quot;&#47;app&#47;app.py&quot;, line 18, in &lt;module&gt;<br>  db.init_app(app)<br>File &quot;&#47;usr&#47;local&#47;lib&#47;python3.8&#47;site-packages&#47;flask_sqlalchemy&#47;extension.py&quot;, line 253, in init_app<br>  raise RuntimeError(<br>RuntimeError: A &#39;SQLAlchemy&#39; instance has already been registered on this Flask app. Import and use that instance instead.</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 示例应用的后端服务的镜像版本 latest 和 1276e23 都是运行正常的，请问你使用的镜像版本是？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-02 11:25:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/16/a2/26ed8f44.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>加菲老猫</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，我尝试根据您的范例，在github action中 Update helm values.yaml遇到以下错误，请您给个思路，谢谢！<br><br><br><br>updateFile is deprected, the updated content will be written to the file by default from now on<br>Error: HttpError: Resource not accessible by integration</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 把 updateFile 参数删除即可。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-22 15:50:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1c/55/51/c7bffc64.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Andrew</span>
  </div>
  <div class="_2_QraFYR_0">是否可以通过export环境变量的方式<br><br>比如<br>定义values.yaml<br>image:<br>  tag: _${IMAGE_TAG}<br>然后再deploy的时候通过脚本 export IMAGE_TAG=${DOCKER_IMAGE_ID}</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Values.yaml 文件需要静态的内容。不支持这种方式。可以调研一下 helmfile 是否满足你的诉求。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-11 09:46:22</div>
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
  <div class="_2_QraFYR_0">最近项目也要实现灰度发布，项目落地argocd方案<br>1. argocd 是否需要持久化部署，比如挂载云盘等<br>2. argocd 3个状态是必须ok，才能确定定义资源yaml和集群资源状态一致</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: ArgoCD 会在集群部署一个 Redis 实例，从稳定性的角度考虑可以用云厂商的托管 Redis 服务。其他的都是 CRD 资源持久化在 ETCD，只要保证 K8s 的可用性就可以了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-09 15:25:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/63/ae/eb536e1d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>bob</span>
  </div>
  <div class="_2_QraFYR_0">老师，您好！由于我本地采用的是centos虚拟机，所以需要通过windowns宿主机去访问ArgoCD，所以未使用端口转发方式——kubectl port-forward service&#47;argocd-server 8080:80 -n argocd，而是将argocd-server这个service的类型改为NodePort，但宿主机还是无法访问——http:&#47;&#47;虚拟机IP:nodeport，请问这是什么原因？谢谢！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 原理上没问题，往两个方向去查：<br>1. Windows 宿主机和虚拟机的网络连通性<br>2. 虚拟机的端口是否被防火墙阻止<br><br>建议通过端口转发的方式来访问，只需要确保能和 API Server 的连通性，这是最简单的办法~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-03 08:48:45</div>
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
  <div class="_2_QraFYR_0">这个流程可以放在一个平台里面监控么</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 没有开源方案，可以自研~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-27 13:53:50</div>
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
  <div class="_2_QraFYR_0">有两个问题请教老师<br>1、argocd 页面更改密码、添加 git 仓库、添加私有 git 仓库密钥、添加私有镜像仓库和密钥等是否是持久化存储的？<br>2、argocd 是否有相应的通知机制，比如收到 git 应用仓库 webhook 通知准备更新，开始更新，更新完毕，更新失败等</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. 没记错的话应该是存在 Secret 对象里，是持久化的。<br>2. 参考这个项目：https:&#47;&#47;argocd-notifications.readthedocs.io&#47;en&#47;stable&#47;</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-27 01:54:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/46/d3/e25d104a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>êｗěｎ</span>
  </div>
  <div class="_2_QraFYR_0">猜是K8S会自动添加一些field，argocd 应该可以屏蔽这些吧</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不是主要原因哈，ArgoCD 会屏蔽掉自己的差异，可以从 HPA 和 Replicas 角度看看。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-27 00:43:25</div>
  </div>
</div>
</div>
</li>
</ul>