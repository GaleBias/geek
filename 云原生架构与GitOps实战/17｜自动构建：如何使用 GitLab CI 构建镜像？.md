<audio title="17｜自动构建：如何使用 GitLab CI 构建镜像？" src="https://static001.geekbang.org/resource/audio/98/14/981ab2359ee20e333d3c0ff85fb09e14.mp3" controls="controls"></audio> 
<p>你好，我是王炜。</p><p>在上一节课，我们学习了如何使用 GitHub Action 自动构建镜像，我们通过为示例应用配置 GitHub Action 工作流，实现了自动构建，并将镜像推送到了 Docker Hub 镜像仓库。</p><p>但是，要使用 GitHub Action 构建镜像，前提条件是你需要使用 GitHub 作为代码仓库，那么，如果我所在的团队使用的是 GitLab 要怎么做呢？</p><p>这节课，我会带你学习如何使用 GitLab CI 来自动构建镜像。我还是以示例应用为例，使用 SaaS 版的 GitLab 从零配置 CI 流水线。</p><p>需要注意的是，有些团队是以自托管的方式来使用 GitLab 的，也就是我们常说的私有部署的方式，它和 SaaS 版本的差异不大。如果你用的是私有化部署版本，同样可以按照这节课的流程来实践。</p><h2>GitLab CI 简介</h2><p>在正式使用 GitLab CI 之前，你需要先了解一些基本概念，你可以结合下面这张图来理解。</p><p><img src="https://static001.geekbang.org/resource/image/bd/68/bd3ac77e391a02eb63ab2cc6f8b07368.jpg?wh=1920x1043" alt="图片"></p><p>这张图中出现了 Pipeline、Stage 和 Job 这几个概念，接下来我们分别了解一下。</p><h3>Pipeline</h3><p>Pipeline 指的是流水线，在 GitLab 中，当有新提交推送到仓库中时，会自动触发流水线。流水线包含一组 Stage 和 Job 的定义，它们负责执行具体的逻辑。</p><!-- [[[read_end]]] --><p>在 GitLab 中，Pipeline 是通过仓库根目录下的 .gitlab-ci.yml 文件来定义的。</p><p>此外，Pipeline 在全局也可以配置运行镜像、全局变量和额外服务镜像。</p><h3>Stage</h3><p>Stage 字面上的意思是“阶段”。在 GitLab CI 中，至少需要包含一个 Stage，上面这张图中有三个 Stage，分别是 Stage1、Stage2 和 Stage3，不同的 Stage 是按照定义的顺序依次执行的。如果其中一个 Stage 失败，则整个 Pipeline 都将失败，后续的 Stage 也都不会再继续执行。</p><h3>Job</h3><p>Job 字面上的意思是“任务”。实际上，Job 的作用是定义具体需要执行的 Shell 脚本，同时，Job 可以被关联到一个 Stage 上。当 Stage 执行时，它所关联的 Job 也会并行执行。</p><p>以自动构建镜像为例，我们可能需要在 1 个 Job 中定义 2 个 Shell 脚本步骤，它们分别是：</p><ul>
<li>运行 docker build 构建镜像</li>
<li>运行 docker push 来推送镜像</li>
</ul><h3>费用</h3><p>和 GitHub Action 一样，GitLab 也不能无限免费使用。对于 GitLab 免费账户，每个月有 400 分钟的 GitLab CI/CD 时长可供使用，超出时长则需要按量付费，你可以在<a href="https://about.gitlab.com/pricing/">这里</a>查看详细的计费策略。</p><h2>为示例应用创建 GitLab CI Pipeline</h2><p>在简单学习了 GitLab CI 相关概念之后，<strong>接下来我们进入到实战环节</strong>。</p><p>我仍然以示例应用为例，介绍如何配置自动构建示例应用的前后端镜像流水线。在这个例子中，我们创建的流水线将实现以下这些步骤。</p><ul>
<li>运行 docker login 登录到 Docker Hub。</li>
<li>运行 docker build 来构建前后端应用的镜像。</li>
<li>运行 docker push 推送镜像。<br>
接下来，我们开始创建 GitLab CI Pipeline。</li>
</ul><h3>创建 .gitlab-ci.yml 文件</h3><p>首先，将示例应用仓库克隆到本地。</p><pre><code class="language-yaml">$ git clone https://github.com/lyzhang1999/kubernetes-example.git
</code></pre><p>进入 kubernetes-example 目录。</p><pre><code class="language-yaml">$ cd kubernetes-example
</code></pre><p>然后，将下面的内容保存到 .gitlab-ci.yml 文件内。</p><pre><code class="language-yaml">stages:
  - build
  
image: docker:20.10.16

variables:
  DOCKER_TLS_CERTDIR: "/certs"
  DOCKERHUB_USERNAME: "lyzhang1999"

services:
  - docker:20.10.16-dind

before_script:
  - docker login -u $DOCKERHUB_USERNAME -p $DOCKERHUB_TOKEN

build_and_push:
  stage: build
  script:
    - docker build -t $DOCKERHUB_USERNAME/frontend:$CI_COMMIT_SHORT_SHA ./frontend
    - docker push $DOCKERHUB_USERNAME/frontend:$CI_COMMIT_SHORT_SHA
    - docker build -t $DOCKERHUB_USERNAME/backend:$CI_COMMIT_SHORT_SHA ./backend
    - docker push $DOCKERHUB_USERNAME/backend:$CI_COMMIT_SHORT_SHA
</code></pre><p>请注意，<strong>你需要将上面的 variables.DOCKERHUB_USERNAME 环境变量替换为你的 Docker Hub 用户名</strong>。</p><p>接下来，结合上面提到的概念，我简单介绍一下这个 Pipeline。</p><p>stages 字段定义了阶段，在这个 Pipeline 中，我们定义了一个 build 阶段。</p><p>image 字段定义了运行镜像，也就是说，GitLab CI 将会使用 docker:20.10.16 镜像来启动一个容器，并在容器内运行 Pipeline。</p><p>variables 字段定义了全局变量，其中， DOCKER_TLS_CERTDIR 变量是用来共享 Docker 证书的。DOCKERHUB_USERNAME 变量是 Docker Hub 的用户名。</p><p>services 字段定义了一个额外的镜像，你可以把它理解成一个额外的容器，它将和 image 字段定义的容器相互协作，这两个容器可以相互访问。</p><p>before_script 定义了 Pipeline 最开始的 Shell 脚本， 它将会在 Job 运行之前执行。在这里，我们运行了 docker login 命令来登录到 Docker Hub，以便获得推送镜像的权限。请注意，<code>$DOCKERHUB_USERNAME</code> 变量的值来源于我们在 variables 定义的值，<code>$DOCKERHUB_TOKEN</code> 是一个在 GitLab UI 界面定义的变量，我们稍后会在 GitLab 平台添加。</p><p>build_and_push 字段定义了一个 Job，“build_and_push” 实际上是 Job 的名称，你也可以更改这个名称。build_and_push.stage 字段定义了 Job 所属的 Stage，也就是 build 阶段。build_and_push.script 字段定义了执行的具体的 Shell 脚本，它们是按顺序执行的。在这里，我们分别构建了 frontend 和 backend 的镜像，并将它们推送到 Docker Hub 仓库。其中，<code>$CI_COMMIT_SHORT_SHA</code> 是一个内置变量，它可以获取到当前的 short commit id。</p><h3>创建 GitLab 仓库并推送</h3><p>创建完 .gitlab-ci.yml 文件后，接下来我们将示例应用推送到 GitLab 上。首先，你需要通过<a href="https://gitlab.com/projects/new">这个页面</a>来为你自己创建新的代码仓库，仓库名设置为 kubernetes-example。</p><p><img src="https://static001.geekbang.org/resource/image/fd/76/fd26925e7b242bfa88b5e24835ec5676.png?wh=1920x879" alt="图片"></p><p>创建完成后，将刚才克隆的 kubernetes-example 仓库的 remote url 配置为你刚才创建仓库的 Git 地址。</p><pre><code class="language-yaml">$ git remote set-url origin YOUR_GITLAB_REPO_URL
</code></pre><p>然后，将 kubernetes-example 推送到你的 GitLab 仓库中。在这之前，你可能需要配置 SSH Key，你可以参考<a href="https://gitlab.com/-/profile/keys">这个链接</a>来配置，这里就不再赘述了。</p><pre><code class="language-yaml">$ git add .
$ git commit -a -m 'first commit'
$ git branch -M main
$ git push -u origin main
</code></pre><h3>创建 Docker Hub Secret</h3><p>创建完 .gitlab-ci.yml 文件后，接下来我们需要创建 Docker Hub Secret，它将会为工作流提供推送镜像的权限。</p><p>首先，使用你注册的账号密码登录 <a href="https://hub.docker.com/">https://hub.docker.com/</a>。然后，点击右上角的“用户名”，选择“Account Settings”，并进入左侧的“Security”菜单。</p><p><img src="https://static001.geekbang.org/resource/image/8e/7a/8ea4a28d9dd98096108c30ac89a8727a.png?wh=1920x879" alt="图片"></p><p>然后，点击右侧的“New Access Token”按钮创建一个新的 Token。</p><p><img src="https://static001.geekbang.org/resource/image/d5/19/d5c0e2a3b07c16213c65c0bc62a35e19.png?wh=1472x1114" alt="图片"></p><p>输入描述，然后点击“Genarate”按钮生成 Token。</p><p><img src="https://static001.geekbang.org/resource/image/eb/97/eb21ee249d7a4302f789e814345e2b97.png?wh=1470x1146" alt="图片"></p><p>点击“Copy and Close”将 Token 复制到剪贴板。<strong>请注意，一旦窗口关闭，我们就无法再次查看这个 Token 了，所以请务必复制并在其他地方保存下来。</strong></p><h3>创建 GitLab CI Variables</h3><p>创建完 Docker Hub Token 之后，我们就可以创建 GitLab CI Variables 了，也就是要为 Pipeline 提供 DOCKERHUB_TOKEN 变量值。</p><p>进入 kubernetes-example 仓库的 Settings 页面，点击左侧的“CI/CD”，然后点击右侧的“Variables”展开菜单，接着点击“Add variable”来创建新的 Variables。如下图所示。</p><p><img src="https://static001.geekbang.org/resource/image/20/35/205c015794d81d03fe09ae70195e8635.png?wh=1920x878" alt="图片"></p><p>在弹出的输入框中，将 Key 填写为 DOCKERHUB_TOKEN。</p><p>将 Value 填写为刚才我们复制的 Docker Hub Token，其他选项保持默认，点击“Add variable”创建变量值，如下图所示。</p><p><img src="https://static001.geekbang.org/resource/image/7c/77/7c02a086ee198ba4f0c7ccfe64c94177.png?wh=1526x1012" alt="图片"></p><h3>触发 GitLab CI Pipeline</h3><p>到这里，准备工作已经全部完成了。请注意，如果你使用的是 GitLab SaaS 版，<strong>那么你需要先绑定信用卡才能使用 CI/CD 的免费额度。</strong></p><p>接下来我们尝试触发 GitLab CI Pipeline。</p><p>首先，向仓库提交一个空 commit。</p><pre><code class="language-yaml">$ git commit --allow-empty -m "Trigger Build"
</code></pre><p>然后，使用 git push 来推送到仓库，<strong>这将触发 Pipeline</strong>。</p><pre><code class="language-yaml">$ git push origin main
</code></pre><p>接下来，进入 kubernetes-example 仓库的“CI/CD”页面，你会看到我们刚才触发的流水线。</p><p><img src="https://static001.geekbang.org/resource/image/1f/a0/1f05fb1bb1a762a13c17afd172b529a0.png?wh=1920x880" alt="图片"></p><p>你可以点击流水线的状态（running）进入流水线详情页面。</p><p><img src="https://static001.geekbang.org/resource/image/12/fe/12db285ef4c4de2a4d66c9abb8ebfefe.png?wh=1920x877" alt="图片"></p><p>在流水线的详情页面，我们能看到流水线的每一个 Job 的状态还有运行时输出的日志。</p><p>当工作流运行完成后，进入到 Docker Hub frontend 或者 backend 镜像的详情页，你将看到刚才 GitLab CI 自动构建并推送的新版本镜像。</p><p><img src="https://static001.geekbang.org/resource/image/68/5d/684cd33dffd31315b7a9043a2d90d05d.png?wh=1920x879" alt="图片"></p><p>到这里，我们便完成了使用 GitLab CI 自动构建镜像。最终实现的效果是，当我们向仓库推送新的提交时，GitLab 流水线将自动构建 frontend 和 backend 镜像，<strong>并且每一个 commit id 都会对应一个镜像版本。</strong></p><h2>总结</h2><p>在这节课，我为你介绍了如何使用 GitLab CI 来自动构建镜像，并讲解了Pipeline、Stage 和 Job几个重要概念。</p><p>GitLab CI 是通过在仓库根目录创建 .gitlab-ci.yml 文件来定义流水线的，这和 GitHub 有明显的差异。在这节课的例子中，.gitlab-ci.yml 文件定义的内容也相对简单，它基本上和我们在本地构建镜像所运行的命令以及顺序是一致的。</p><p>此外，相比较 GitHub Action Workflow，GitLab CI 省略了触发器和检出代码的配置步骤，并且，在 GitLab CI 中我们是通过 DiND 的方式来运行流水线的，也就是在容器的运行环境下启动另一个容器来运行流水线，而 GitHub Action 则是通过虚拟机的方式来运行流水线。</p><p>和 GitHub Action 相比较，它们除了流水线文件内容不一样以外，其他的操作例如创建 GitLab 仓库、创建 Docker Hub Secret 以及创建 GitLab CI Variables 等步骤都是差不多的。</p><p>最终，当我们有新的推送到仓库时，GitLab CI 将运行自动构建镜像的流水线，并且每次提交的 commit id 都会对应一个镜像版本，和 GitHub Action Workflow 一样，<strong>也实现了代码版本和制品版本的对应关系。</strong></p><h2>思考题</h2><p>最后，给你留一道简单的思考题吧。</p><p>请你尝试改造 .gitlab-ci.yml ，使其同时支持构建 linux/amd64 和 linux/arm64 两个平台的镜像，并和我分享改动之后的 YAML。我们下节课见。</p>
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
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83eoUEZtvgjx3Lo8ib1GxBruDJCLXxX0KfRptk7BoBtRebKMA4Chp2tPbiaCwlCQ9hBZ4JnukX1bs9blA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Waylon</span>
  </div>
  <div class="_2_QraFYR_0">stages:<br>  - build<br>  <br>image: docker:20.10.16<br><br>variables:<br>  DOCKER_TLS_CERTDIR: &quot;&#47;certs&quot;<br>  DOCKERHUB_USERNAME: &quot;lyzhang1999&quot;<br>  PLATFORM: &quot;linux&#47;amd64,linux&#47;arm64&quot;<br><br>services:<br>  - docker:20.10.16-dind<br><br>before_script:<br>  - docker buildx create --name builder<br>  - docker buildx use builder<br>  - docker buildx inspect --bootstrap<br>  - docker login -u $DOCKERHUB_USERNAME -p $DOCKERHUB_TOKEN<br><br>build_and_push:<br>  stage: build<br>  script:<br>    - <br>    - docker buildx build --platform $PLATFORM -t $DOCKERHUB_USERNAME&#47;frontend:$CI_COMMIT_SHORT_SHA .&#47;frontend<br>    - docker push $DOCKERHUB_USERNAME&#47;frontend:$CI_COMMIT_SHORT_SHA<br>    - docker buildx build --platform $PLATFORM -t $DOCKERHUB_USERNAME&#47;backend:$CI_COMMIT_SHORT_SHA .&#47;backend<br>    - docker push $DOCKERHUB_USERNAME&#47;backend:$CI_COMMIT_SHORT_SHA</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 正确👍🏻，使用了变量和 before_script，非常棒的例子！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-16 17:32:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/f6/24/547439f1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ghostwritten</span>
  </div>
  <div class="_2_QraFYR_0">遇到两个问题不过解决了：<br>$ docker buildx create --name builder<br>error: could not create a builder instance with TLS data loaded from environment. Please use `docker context create &lt;context-name&gt;` to create a context for current environment and then create a builder instance with `docker buildx create &lt;context-name&gt;`<br>Cleaning up project directory and file based variables<br>00:01<br>ERROR: Job failed: exit code 1<br><br>$ docker push $DOCKERHUB_USERNAME&#47;frontend:$CI_COMMIT_SHORT_SHA<br>The push refers to repository [docker.io&#47;ghostwritten&#47;frontend]<br>An image does not exist locally with the tag: ghostwritten&#47;frontend<br>Cleaning up project directory and file based variables<br>00:01<br>ERROR: Job failed: exit code 1<br><br>这是我的.gitlab-ci.yaml:<br>stages:<br>  - build<br><br>image: docker:20.10.16<br><br>variables:<br>  DOCKER_TLS_CERTDIR: &quot;&#47;certs&quot;<br>  DOCKERHUB_USERNAME: &quot;ghostwritten&quot;<br>  PLATFORM: &quot;linux&#47;amd64,linux&#47;arm64&quot;<br><br>services:<br>  - docker:20.10.16-dind<br><br>before_script:<br>  - docker context create builder<br>  - docker buildx create --name builder --use builder<br>  - docker buildx use builder<br>  - docker buildx inspect --bootstrap<br>  - docker login -u $DOCKERHUB_USERNAME -p $DOCKERHUB_TOKEN<br><br>build_and_push:<br>  stage: build<br>  script:<br>    - docker buildx build --platform $PLATFORM -t $DOCKERHUB_USERNAME&#47;frontend:$CI_COMMIT_SHORT_SHA .&#47;frontend --push<br>    - docker buildx build --platform $PLATFORM -t $DOCKERHUB_USERNAME&#47;backend:$CI_COMMIT_SHORT_SHA .&#47;backend --push</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍🏻</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-29 13:02:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/25/ec/f3/f140576e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Jich</span>
  </div>
  <div class="_2_QraFYR_0">私有化部署的gitlab执行CI需要先安装个runner的吧</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，私有化部署的 GitLab 需要先配置好 runner。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-16 17:30:19</div>
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
  <div class="_2_QraFYR_0">老师怎么使用DinD的方式来进行CI呀。能不能有一篇文章或者教程专门讲一下使用DinD来构建镜像</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 下一节课马上会讲到如何用 Tekton 构建镜像，也就是直接在 K8s 集群中通过 DinD 的方式来构建镜像。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-16 13:40:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/a9/c6/30b29c22.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>那风在极客</span>
  </div>
  <div class="_2_QraFYR_0">DIND 不是被弃用了吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 安全扫描弃用了 DinD 的方式，CI&#47;CD 使用 DinD 的方式运行仍然是官方推荐的方式之一。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-16 11:02:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Gg1doG856ec8BWgFregO01XwjnfygRmXpqb9cJ63JzAyH08yCYkCItuYN71p4Vk0JDhODiaHbGVdmRpeRVIyuuA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_64f925</span>
  </div>
  <div class="_2_QraFYR_0">私有化部署中，如果gitlab-runner 是跑在k8s 上的，那么1.24以及以上版本应该不能使用DinD了，有什么好的解决方案吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以考虑在容器通过 buildkit 命令行工具来构建。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-11 17:37:06</div>
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
  <div class="_2_QraFYR_0">.gitlab-ci.yml ：这个文件是自动生成的么</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 需要自己创建，为了方便大家参考，示例应用里已经包含这个文件了，你可以先删除掉，然后再进行实践。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-16 09:11:34</div>
  </div>
</div>
</div>
</li>
</ul>