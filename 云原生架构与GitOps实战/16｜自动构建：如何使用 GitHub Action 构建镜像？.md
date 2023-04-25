<audio title="16｜自动构建：如何使用 GitHub Action 构建镜像？" src="https://static001.geekbang.org/resource/audio/f0/64/f05c347c83a85eb9055984e08caa1164.mp3" controls="controls"></audio> 
<p>你好，我是王炜。</p><p>前面几节课，我们一起学习了容器化的最佳实践。从本质上来说，我们一直都在学习编写 Dockerfile 的技巧，以及如何构建出更适合生产环境的镜像。</p><p>在之前的课程中，我们编写完 Dockerfile 之后，会在本地通过 docker build 命令来构建镜像，然后把它推送到 Docker Hub 的镜像仓库中。不过实际上，在完整的 GitOps 的环节中，我们并不会用这种手动的方式来构建镜像，通常我们会使用工具完成自动构建。</p><p>如果你熟悉 DevOps 流程，会知道在提交代码之后会触发一个自动化流程，<strong>它就是我们常说的 CI (Continuous integration)，持续集成</strong>。持续集成会自动帮助我们做一些编译、构建、测试和打包工作。在将业务进行容器化改造之后，我们会有更多构建 Docker 镜像的工作，所以为了提高效率，在 GitOps 工作流中，我们同样可以在持续集成的阶段实现自动化的镜像构建。</p><p>所以，从这节课开始，我将带你学习 GitOps 工作流中的第一个自动化阶段：自动构建镜像。</p><p>这节课我会以 K8s 极简实战中的示例应用为例，带你从零开始配置 GitHub Action 自动构建镜像的工作流，<strong>它是组成 GitOps 工作流中的重要的一环</strong>。</p><!-- [[[read_end]]] --><h2>什么是 GitHub Action？</h2><p>在正式使用 GitHub Action 自动构建镜像之前，你需要先了解一些基本概念，我们直入主题，重点介绍这节课会涉及的概念和用法。</p><p>为了帮助你更好地理解 GitHub Action，我为你画了一张示意图。</p><p><img src="https://static001.geekbang.org/resource/image/6b/c7/6bb3d5a619a352c89be6356d1220edc7.jpg?wh=1863x1026" alt="图片"></p><p>这张图中出现了几个基本概念：Workflow、Event、Job 和 Step，我们分开来讲解。</p><h3>Workflow</h3><p>Workflow 也叫做工作流。其实，GitHub Action 本质上是一个是一个 CI/CD 工作流，要使用工作流，我们首先需要先定义它。和 K8s Manifest 一样，GitHub Action 工作流是通过 YAML 来描述的，你可以在任何 GitHub 仓库创建 .github/workflows 目录，并创建 YAML 文件来定义工作流。</p><p>所有在 .github/workflows 目录创建的工作流文件，都将被 GitHub 自动扫描。在工作流中，通常我们会进一步定义 Event、Job 和 Step 字段，它们被用来定义工作流的触发时机和具体行为。</p><h3>Event</h3><p>Event 从字面上的理解是“事件”的意思，你可以简单地把它理解为定义了“什么时候运行工作流”，也就是工作流的触发器。</p><p>在定义自动化构建镜像的工作流时，我们通常会把 Event 的触发器配置成“当指定分支有新的提交时，自动触发镜像构建”。</p><h3>Jobs</h3><p>Jobs 的字面意思是一个具体的任务，它是一个抽象概念。在工作流中，它并不能直接工作，而是需要通过 Step 来定义具体的行为。此外，你还可以为 Job 定义它的运行的环境，例如 ubuntu。</p><p>在一个 Workflow 当中，你可以定义多个 Job，多个 Job 之间可以并行运行，也可以定义相互依赖关系。在自动构建镜像环节，通常我们只需要定义一个 Job 就够了，所以在上面的示意图中，我只画出了一个 Job。</p><h3>Step</h3><p>Step 隶属于 Jobs，它是工作流中最小的粒度，也是最重要的部分。通常来说，Step 的具体行为是执行一段 Shell 来完成一个功能。在同一个 Job 里，一般我们需要定义多个 Step 才能完成一个完整的 Job，由于它们是在同一个环境下运行的，所以当它们运行时，就等同于在同一台设备上执行一段 Shell。</p><p>以自动构建镜像为例，我们可能需要在 1 个 Job 中定义 3 个 Step。</p><ul>
<li>Step1，克隆仓库的源码。</li>
<li>Step2，运行 docker build 来构建镜像。</li>
<li>Step3，推送到镜像仓库。</li>
</ul><h3>费用</h3><p>GitHub Action 在使用上虽然很方便，但天下并没有免费的午餐。对于 GitHub 免费账户，每个月有 2000 分钟的 GitHub Action 时长可供使用（Linux 环境），超出时长则需要按量付费，你可以在<a href="https://docs.github.com/en/billing/managing-billing-for-github-actions/about-billing-for-github-actions#calculating-minute-and-storage-spending">这里</a>查看详细的计费策略。</p><h2>为示例应用创建 GitHub Action Workflow</h2><p>了解了 GitHub Action 的相关概念，<strong>接下来我们就进入到实战环节了</strong>。</p><p>我以 K8s 极简实战模块的示例应用为例，看看如何配置自动构建示例应用的前后端镜像工作流。在这个例子中，我们创建的工作流将实现以下这些步骤。</p><ul>
<li>当 main 分支有新的提交时，触发工作流。</li>
<li>克隆代码。</li>
<li>初始化 Docker 构建工具链。</li>
<li>登录 Docker Hub。</li>
<li>构建前后端应用镜像，并使用 commit id 作为镜像的 tag。</li>
<li>推送到 Docker Hub 镜像仓库。</li>
</ul><p>下面，我们来为示例应用创建工作流。</p><h3>创建 build.yaml 文件</h3><p>首先，我们要将示例应用仓库克隆到本地。</p><pre><code class="language-yaml">$ git clone https://github.com/lyzhang1999/kubernetes-example.git
</code></pre><p>进入 kubernetes-example 目录。</p><pre><code class="language-yaml">$ cd kubernetes-example
</code></pre><p>然后，在当前目录下新建 .github/workflows 目录。</p><pre><code class="language-yaml">$ mkdir -p .github/workflows
</code></pre><p>接下来，将下面的内容保存到 .github/workflows/build.yaml 文件内。</p><pre><code class="language-yaml">name: build

on:
  push:
    branches:
      - 'main'

env:
  DOCKERHUB_USERNAME: lyzhang1999

jobs:
  docker:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      - name: Set outputs
        id: vars
        run: echo "::set-output name=sha_short::$(git rev-parse --short HEAD)"
      - name: Set up QEMU
        uses: docker/setup-qemu-action@v2
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
      - name: Login to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ env.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      - name: Build backend and push
        uses: docker/build-push-action@v3
        with:
          context: backend
          push: true
          tags: ${{ env.DOCKERHUB_USERNAME }}/backend:${{ steps.vars.outputs.sha_short }}
      - name: Build frontend and push
        uses: docker/build-push-action@v3
        with:
          context: frontend
          push: true
          tags: ${{ env.DOCKERHUB_USERNAME }}/frontend:${{ steps.vars.outputs.sha_short }}
</code></pre><p>请注意，<strong>你需要将上面的 env.DOCKERHUB_USERNAME 环境变量替换为你的 Docker Hub 用户名</strong>。</p><p>我简单介绍一下这个工作流。</p><p>这里的 name 字段是工作流的名称，它会展示在 GitHub 网页上。</p><p>on.push.branches 字段的值为 main，这代表当 main 分支有新的提交之后，会触发工作流。</p><p>env.DOCKERHUB_USERNAME 是我们为 Job 配置的全局环境变量，用作镜像 Tag 的前缀。</p><p>jobs.docker 字段定义了一个任务，它的运行环境是 ubuntu-latest，并且由 7 个 Step 组成。</p><p>jobs.docker.steps 字段定义了 7 个具体的执行阶段。要特别注意的是，uses 字段代表使用 GitHub Action 的某个插件，例如 actions/checkout@v3 插件会帮助我们检出代码。</p><p>在这个工作流中，这 7 个阶段会具体执行下面几件事。</p><ol>
<li>“Checkout”阶段负责将代码检出到运行环境。</li>
<li>“Set outputs”阶段会输出 sha_short 环境变量，值为 short commit id，这可以方便在后续阶段引用。</li>
<li>“Set up QEMU”和“Set up Docker Buildx”阶段负责初始化 Docker 构建工具链。</li>
<li>“Login to Docker Hub”阶段通过 docker login 来登录到 Docker Hub，以便获得推送镜像的权限。要注意的是，with 字段是向插件传递参数的，在这里我们传递了 username 和 password，值的来源分别是我们定义的环境变量 DOCKERHUB_USERNAME 和 GitHub Action Secret，后者我们还会在稍后进行配置。</li>
<li>“Build backend and push”和“Build frontend and push”阶段负责构建前后端镜像，并且将镜像推送到 Docker Hub，在这个阶段中，我们传递了 context、push 和 tags 参数，context 和 tags 实际上就是 docker build 的参数。在 tags 参数中，我们通过表达式 <code>${{ env.DOCKERHUB_USERNAME }}</code> 和 <code>${{ steps.vars.outputs.sha_short }}</code> 分别读取了 在 YAML 中预定义的 Docker Hub 的用户名，以及在“Set outputs”阶段输出的 short commit id。</li>
</ol><h3>创建 GitHub 仓库并推送</h3><p>创建完 build.yaml 文件后，接下来，我们要把示例应用推送到 GitHub 上。首先，你需要通过<a href="https://github.com/new">这个页面</a>来为自己创建新的代码仓库，仓库名设置为 kubernetes-example。</p><p><img src="https://static001.geekbang.org/resource/image/d7/32/d7a01ee80ef7d4cyy58d6b04393d3632.png?wh=1590x1074" alt="图片"></p><p>创建完成后，将刚才克隆的 kubernetes-example 仓库的 remote url 配置为你刚才创建仓库的 Git 地址。</p><pre><code class="language-yaml">$ git remote set-url origin YOUR_GIT_URL
</code></pre><p>然后，将 kubernetes-example 推送到你的仓库。在这之前，你可能还需要配置 SSH Key，你可以参考<a href="https://docs.github.com/cn/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account">这个链接</a>来配置，这里就不再赘述了。</p><pre><code class="language-yaml">$ git add .
$ git commit -a -m 'first commit'
$ git branch -M main
$ git push -u origin main
</code></pre><h3>创建 Docker Hub Secret</h3><p>创建完 build.yaml 文件后，接下来，我们需要创建 Docker Hub Secret，它将会为工作流提供推送镜像的权限。</p><p>首先，使用你注册的账号密码登录 <a href="https://hub.docker.com/">https://hub.docker.com/</a>。然后，点击右上角的“用户名”，选择“Account Settings”，并进入左侧的“Security”菜单。</p><p><img src="https://static001.geekbang.org/resource/image/b7/16/b70fea1a316d6212c858eayyacb67316.png?wh=1920x879" alt="图片"></p><p>下一步点击右侧的“New Access Token”按钮，创建一个新的 Token。</p><p><img src="https://static001.geekbang.org/resource/image/a1/ba/a11f8af16b193fe8f194133d2634e5ba.png?wh=1472x1114" alt="图片"></p><p>输入描述，然后点击“Genarate”按钮生成 Token。</p><p><img src="https://static001.geekbang.org/resource/image/dc/13/dc0d5c4a9bff3c49d81b3d4ab1a8f513.png?wh=1470x1146" alt="图片"></p><p>点击“Copy and Close”将 Token 复制到剪贴板。<strong>请注意，当窗口关闭后，Token 无法再次查看，所以请在其他地方先保存刚才生成的 Token。</strong></p><h3>创建 GitHub Action Secret</h3><p>创建完 Docker Hub Token 之后，接下来我们就可以创建 GitHub Action Secret 了，也就是说我们要为 Workflow 提供 secrets.DOCKERHUB_TOKEN 变量值。</p><p>进入 kubernetes-example 仓库的 Settings 页面，点击左侧的“Secrets”，进入“Actions”菜单，然后点击右侧“New repository secret”创建新的 Secret。</p><p><img src="https://static001.geekbang.org/resource/image/cy/f6/cyy66105ccd814e72bea4729322744f6.png?wh=1920x877" alt="图片"></p><p>在 Name 输入框中输入 DOCKERHUB_TOKEN，这样在 GitHub Action 的 Step 中，<strong>就可以通过 ${{ secrets.DOCKERHUB_TOKEN }} 表达式来获取它的值</strong>。</p><p>在 Secret 输入框中输入刚才我们复制的 Docker Hub Token，点击“Add secret”创建。</p><h3>触发 GitHub Action Workflow</h3><p>到这里，准备工作已经全部完成了，接下来我们尝试触发 GitHub Action 工作流。还记得我们在工作流配置的 on.push.branches 字段吗？它的值为 main，代表当有新的提交到 main 分支时触发工作流。</p><p>首先，我们向仓库提交一个空 commit。</p><pre><code class="language-yaml">$ git commit --allow-empty -m "Trigger Build"
</code></pre><p>然后，使用 git push 来推送到仓库，<strong>这将触发工作流</strong>。</p><pre><code class="language-yaml">$ git push origin main
</code></pre><p>接下来，进入 kubernetes-example 仓库的“Actions”页面，你将看到我们刚才触发的工作流。</p><p><img src="https://static001.geekbang.org/resource/image/17/fc/17e36a14d86485585ed1e283fa22cffc.png?wh=1920x879" alt="图片"></p><p>你可以点击工作流的标题进入工作流详情页面。</p><p><img src="https://static001.geekbang.org/resource/image/74/ec/7401326152f418e6dc59182753yycdec.png?wh=1920x876" alt="图片"></p><p>在工作流的详情页面，我们能看到工作流的每一个 Step 的状态及其运行时输出的日志。</p><p>当工作流运行完成后，进入到 Docker Hub frontend 或者 backend 镜像的详情页，你将看到刚才 GitHub Action 自动构建并推送的新版本镜像。</p><p><img src="https://static001.geekbang.org/resource/image/e3/94/e3c7aa048fd003222bc2258d4d35f994.png?wh=1920x877" alt="图片"></p><p>到这里，我们便完成了使用 GitHub Action 自动构建镜像的全过程。最终实现效果是，当我们向 main 分支提交代码时，GitHub 工作流将自动构建 frontend 和 backend 镜像，<strong>并且每一个 commit id 对应一个镜像版本。</strong></p><h2>总结</h2><p>总结一下，这节课，我为你介绍了构成 GitOps 工作流的第一个自动化阶段：自动化构建镜像。为了实现自动化构建镜像，我们学习了 GitHub Action 工作流及其基本概念，例如 Workflow、Event、Jobs 和 Steps。</p><p>在介绍 GitHub Action 相关概念时，我故意精简了一部分概念，比如 Runner、多个 Jobs 以及 Jobs 相互依赖的情况。在现阶段，我们只需要掌握最简单的自动构建镜像的 YAML 写法以及相关概念就足够了。</p><p>在实战环节，我们创建了一个 build.yaml 文件用来定义 GitHub Action 工作流，总结来说，它定义了工作流的：</p><ol>
<li>工作流名称</li>
<li>在什么时候触发</li>
<li>在什么环境下运行</li>
<li>具体执行的步骤是什么</li>
</ol><p>需要注意的是，在创建 build.yaml 文件后，你需要创建自己的仓库，并将 kubernetes-example 的内容推送到你的仓库中，以便进行触发工作流的实验。其次，为了给 GitHub Action 工作流赋予镜像仓库的推送权限，我们还需要在 Docker Hub 中创建 Token，并将其配置到仓库的 Secrets 中。在配置时，需要注意 Secret Name 和工作流 Step 中的 ${{ secrets.DOCKERHUB_TOKEN }} 表达式相互对应，以便工作流能够获取到正确的 Secrets。</p><p>配置完成后，当我们向 Main 分支推送新的提交时，GitHub Action 工作流将会被自动触发，工作流会自动构建 frontend 和 backend 镜像，并且会使用当前的 short commit id 作为镜像的 Tag 推送到 Docker Hub 中。</p><p><strong>这意味着，每一个提交都会生成一个 Docker 镜像，实现了代码和制品的对应关系。</strong>这种对应关系给我们带来了非常大的好处，例如当我们要回滚或更新应用时，只需要找到代码的 commit id 就能够找到对应的镜像版本。</p><h2>思考题</h2><p>最后，给你留一道简单的思考题吧。</p><p>docker/build-push-action 插件还有其他的一些高级配置，请你结合 docker/build-push-action@v3 <a href="https://github.com/docker/build-push-action/blob/master/docs/advanced/multi-platform.md">插件文档</a>，尝试改造 build.yaml，使其同时支持构建 linux/amd64 和 linux/arm64 两个平台的镜像，并和我分享改动之后的 YAML。</p><p>欢迎你给我留言交流讨论，你也可以把这节课分享给更多的朋友一起阅读。我们下节课见。</p>
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
  <div class="_2_QraFYR_0">跟着老师的教程做了实验，发现生成了2个镜像，分别是linux&#47;amd64和linux&#47;arm64，这是怎么做到的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我的课程源码里构建了两个平台的镜像，你可以尝试删除.github&#47;workflows&#47;build.yaml 文件里定义的 platforms 字段，这样就只会构建单个平台的镜像了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-13 10:43:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/23/bb/a1a61f7c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>GAC·DU</span>
  </div>
  <div class="_2_QraFYR_0">老师，您在日常工作中，镜像版本是如何定义的？有特殊的规范吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我们的实践是生产镜像用一个特殊的 prefix 标识，比如 release-v1.0.0，其他的用 commit id 作为 tag。<br><br>这里固定的规范，结合 CI 和自动化，选择适合团队的实践就可以了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-13 06:35:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/20/12/e4/57ade29a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>dva</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，能详细解释一下这个语句是什么意思吗？双冒号是什么特殊写法？<br>run: echo &quot;::set-output name=sha_short::$(git rev-parse --short HEAD)&quot;</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 在阶段中输出 sha_short 变量，以便在后续阶段可以通过表达式 ${{ steps.vars.outputs.sha_short }} 来获取这个值。<br>此外，你还可以用这个写法来在阶段中输出值：https:&#47;&#47;docs.github.com&#47;en&#47;actions&#47;using-jobs&#47;defining-outputs-for-jobs</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-28 17:24:09</div>
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
  <div class="_2_QraFYR_0">@v2 @v3是啥意思呢。。插件版本嘛</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-07 18:03:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/57/4f/6fb51ff1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>一步</span>
  </div>
  <div class="_2_QraFYR_0">怎么读取  默认的环境变量？ 使用 ${{env. GITHUB_REPOSITORY}} 读取不到</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以用 ${{ github.repositoryUrl }} 获取，另外所有可用的内置变量可以在这个文档里查询：https:&#47;&#47;docs.github.com&#47;en&#47;actions&#47;learn-github-actions&#47;contexts</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-14 17:34:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/9d/84/171b2221.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jeffery</span>
  </div>
  <div class="_2_QraFYR_0">应该需要这个platforms: linux&#47;amd64,linux&#47;arm64。<br>官网地址：https:&#47;&#47;docs.docker.com&#47;build&#47;ci&#47;github-actions&#47;examples&#47;#multi-platform-images<br>不知道这块...<br>        with:<br>          username: ${{ secrets.DOCKERHUB_USERNAME }}<br>          password: ${{ secrets.DOCKERHUB_TOKEN }}<br>是不是写错了<br>with: <br>username: ${{ env.DOCKERHUB_USERNAME }} <br>password: ${{ secrets.DOCKERHUB_TOKEN }}<br>name: ci<br><br>on:<br>  push:<br>    branches:<br>      - &quot;main&quot;<br><br>jobs:<br>  docker:<br>    runs-on: ubuntu-latest<br>    steps:<br>      -<br>        name: Checkout<br>        uses: actions&#47;checkout@v3<br>      -<br>        name: Set up QEMU<br>        uses: docker&#47;setup-qemu-action@v2<br>      -<br>        name: Set up Docker Buildx<br>        uses: docker&#47;setup-buildx-action@v2<br>      -<br>        name: Login to Docker Hub<br>        uses: docker&#47;login-action@v2<br>        with:<br>          username: ${{ secrets.DOCKERHUB_USERNAME }}<br>          password: ${{ secrets.DOCKERHUB_TOKEN }}<br>      -<br>        name: Build and push<br>        uses: docker&#47;build-push-action@v3<br>        with:<br>          context: .<br>          platforms: linux&#47;amd64,linux&#47;arm64<br>          push: true<br>          tags: user&#47;app:latest<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 回答正确👍🏻<br><br>USERNAME 可以从 env 里读取，也可以配置 github secret 读取。<br><br>在这个例子中我们是用 env 读的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-13 08:57:28</div>
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
  <div class="_2_QraFYR_0">这个在gitlab上面可以使用么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Gitlab 自动构建在下一节课会介绍哦。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-13 08:10:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/23/bb/a1a61f7c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>GAC·DU</span>
  </div>
  <div class="_2_QraFYR_0"><br>name: build<br><br>on:<br>  push:<br>    branches:<br>      - &#39;main&#39;<br><br>env:<br>  DOCKERHUB_USERNAME: lyzhang1999<br><br>jobs:<br>  docker:<br>    runs-on: ubuntu-latest<br>    steps:<br>      - name: Checkout<br>        uses: actions&#47;checkout@v3<br>      - name: Set outputs<br>        id: vars<br>        run: echo &quot;::set-output name=sha_short::$(git rev-parse --short HEAD)&quot;<br>      - name: Set up QEMU<br>        uses: docker&#47;setup-qemu-action@v2<br>      - name: Set up Docker Buildx<br>        uses: docker&#47;setup-buildx-action@v2<br>      - name: Login to Docker Hub<br>        uses: docker&#47;login-action@v2<br>        with:<br>          username: ${{ env.DOCKERHUB_USERNAME }}<br>          password: ${{ secrets.DOCKERHUB_TOKEN }}<br>      - name: Build backend and push<br>        uses: docker&#47;build-push-action@v3<br>        with:<br>          context: backend<br>          platforms: linux&#47;amd64,linux&#47;arm64<br>          push: true<br>          tags: ${{ env.DOCKERHUB_USERNAME }}&#47;backend:${{ steps.vars.outputs.sha_short }}<br>      - name: Build frontend and push<br>        uses: docker&#47;build-push-action@v3<br>        with:<br>          context: frontend<br>          platforms: linux&#47;amd64,linux&#47;arm64<br>          push: true<br>          tags: ${{ env.DOCKERHUB_USERNAME }}&#47;frontend:${{ steps.vars.outputs.sha_short }}</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 回答正确，两个镜像都配置了不同的平台👍🏻</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-13 06:32:56</div>
  </div>
</div>
</div>
</li>
</ul>