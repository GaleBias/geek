<audio title="51 _ 基于 GitHub Actions 的 CI 实战" src="https://static001.geekbang.org/resource/audio/72/be/7234dbb53dae77ff314b56abc42bf5be.mp3" controls="controls"></audio> 
<p>你好，我是孔令飞。这是本专栏正文的最后一讲了，恭喜你坚持到了最后！</p><p>在Go项目开发中，我们要频繁地执行静态代码检查、测试、编译、构建等操作。如果每一步我们都手动执行，效率低不说，还容易出错。所以，我们通常借助CI系统来自动化执行这些操作。</p><p>当前业界有很多优秀的CI系统可供选择，例如 <a href="https://circleci.com/">CircleCI</a>、<a href="https://travis-ci.org/">TravisCI</a>、<a href="https://github.com/jenkinsci/jenkins">Jenkins</a>、<a href="https://coding.net/">CODING</a>、<a href="https://github.com/features/actions">GitHub Actions</a> 等。这些系统在设计上大同小异，为了减少你的学习成本，我选择了相对来说容易实践的GitHub Actions，来给你展示如何通过CI来让工作自动化。</p><p>这一讲，我会先介绍下GitHub Actions及其用法，再向你展示一个CI示例，最后给你演示下IAM是如何构建CI任务的。</p><h2>GitHub Actions的基本用法</h2><p>GitHub Actions是GitHub为托管在github.com站点的项目提供的持续集成服务，于2018年10月推出。</p><p>GitHub Actions具有以下功能特性：</p><ul>
<li>提供原子的actions配置和组合actions的workflow配置两种能力。</li>
<li>全局配置基于<a href="https://help.github.com/en/articles/migrating-github-actions-from-hcl-syntax-to-yaml-syntax">YAML配置</a>，兼容主流CI/CD工具配置。</li>
<li>Actions/Workflows基于<a href="https://help.github.com/en/articles/events-that-trigger-workflows">事件触发</a>，包括Event restrictions、Webhook events、Scheduled events、External events。</li>
<li>提供可供运行的托管容器服务，包括Docker、VM，可运行Linux、macOS、Windows主流系统。</li>
<li>提供主流语言的支持，包括Node.js、Python、Java、Ruby、PHP、Go、Rust、.NET。</li>
<li>提供实时日志流程，方便调试。</li>
<li>提供<a href="https://help.github.com/en/articles/about-github-actions#discovering-actions-in-the-github-community">平台内置的Actions</a>与第三方提供的Actions，开箱即用。</li>
</ul><!-- [[[read_end]]] --><h3>GitHub Actions的基本概念</h3><p>在构建持续集成任务时，我们会在任务中心完成各种操作，比如克隆代码、编译代码、运行单元测试、构建和发布镜像等。GitHub把这些操作称为Actions。</p><p>Actions在很多项目中是可以共享的，GitHub允许开发者将这些可共享的Actions上传到<a href="https://github.com/marketplace?type=actions">GitHub的官方Actions市场</a>，开发者在Actions市场中可以搜索到他人提交的 Actions。另外，还有一个 <a href="https://github.com/sdras/awesome-actions">awesome actions</a> 的仓库，里面也有不少的Action可供开发者使用。如果你需要某个 Action，不必自己写复杂的脚本，直接引用他人写好的 Action 即可。整个持续集成过程，就变成了一个 Actions 的组合。</p><p>Action其实是一个独立的脚本，可以将Action存放在GitHub代码仓库中，通过<code>&lt;userName&gt;/&lt;repoName&gt;</code>的语法引用 Action。例如，<code>actions/checkout@v2</code>表示<code>https://github.com/actions/checkout</code>这个仓库，tag是v2。<code>actions/checkout@v2</code>也代表一个 Action，作用是安装 Go编译环境。GitHub 官方的 Actions 都放在 <a href="https://github.com/actions">github.com/actions</a> 里面。</p><p>GitHub Actions 有一些自己的术语，下面我来介绍下。</p><ul>
<li>workflow（工作流程）：一个  <code>.yml</code>  文件对应一个 workflow，也就是一次持续集成。一个 GitHub 仓库可以包含多个 workflow，只要是在  <code>.github/workflow</code>  目录下的  <code>.yml</code>  文件都会被 GitHub 执行。</li>
<li>job（任务）：一个 workflow 由一个或多个 job 构成，每个 job 代表一个持续集成任务。</li>
<li>step（步骤）：每个 job 由多个 step 构成，一步步完成。</li>
<li>action（动作）：每个 step 可以依次执行一个或多个命令（action）。</li>
<li>on：一个 workflow 的触发条件，决定了当前的 workflow 在什么时候被执行。</li>
</ul><h3>workflow文件介绍</h3><p>GitHub Actions 配置文件存放在代码仓库的<code>.github/workflows</code>目录下，文件后缀为<code>.yml</code>，支持创建多个文件，文件名可以任意取，比如<code>iam.yml</code>。GitHub 只要发现<code>.github/workflows</code>目录里面有<code>.yml</code>文件，就会自动运行该文件，如果运行过程中存在问题，会以邮件的形式通知到你。</p><p>workflow 文件的配置字段非常多，如果你想详细了解，可以查看<a href="https://docs.github.com/cn/actions/reference/workflow-syntax-for-github-actions">官方文档</a>。这里，我来介绍一些基本的配置字段。</p><ol>
<li><code>name</code></li>
</ol><p><code>name</code>字段是 workflow 的名称。如果省略该字段，默认为当前 workflow 的文件名。</p><pre><code class="language-yaml">name: GitHub Actions Demo
</code></pre><ol start="2">
<li><code>on</code></li>
</ol><p><code>on</code>字段指定触发 workflow 的条件，通常是某些事件。</p><pre><code class="language-yaml">on: push
</code></pre><p>上面的配置意思是，<code>push</code>事件触发 workflow。<code>on</code>字段也可以是事件的数组，例如:</p><pre><code class="language-yaml">on: [push, pull_request]
</code></pre><p>上面的配置意思是，<code>push</code>事件或<code>pull_request</code>事件都可以触发 workflow。</p><p>想了解完整的事件列表，你可以查看<a href="https://docs.github.com/en/actions/reference/events-that-trigger-workflows">官方文档</a>。除了代码库事件，GitHub Actions 也支持外部事件触发，或者定时运行。</p><ol start="3">
<li><code>on.&lt;push|pull_request&gt;.&lt;tags|branches&gt;</code></li>
</ol><p>指定触发事件时，我们可以限定分支或标签。</p><pre><code class="language-yaml">on:
  push:
    branches:
      - master
</code></pre><p>上面的配置指定，只有<code>master</code>分支发生<code>push</code>事件时，才会触发 workflow。</p><ol start="4">
<li><code>jobs.&lt;job_id&gt;.name</code></li>
</ol><p>workflow 文件的主体是<code>jobs</code>字段，表示要执行的一项或多项任务。</p><p><code>jobs</code>字段里面，需要写出每一项任务的<code>job_id</code>，具体名称自定义。<code>job_id</code>里面的<code>name</code>字段是任务的说明。</p><pre><code class="language-yaml">jobs:
  my_first_job:
    name: My first job
  my_second_job:
    name: My second job
</code></pre><p>上面的代码中，<code>jobs</code>字段包含两项任务，<code>job_id</code>分别是<code>my_first_job</code>和<code>my_second_job</code>。</p><ol start="5">
<li><code>jobs.&lt;job_id&gt;.needs</code></li>
</ol><p><code>needs</code>字段指定当前任务的依赖关系，即运行顺序。</p><pre><code class="language-yaml">jobs:
  job1:
  job2:
    needs: job1
  job3:
    needs: [job1, job2]
</code></pre><p>上面的代码中，<code>job1</code>必须先于<code>job2</code>完成，而<code>job3</code>等待<code>job1</code>和<code>job2</code>完成后才能运行。因此，这个 workflow 的运行顺序为：<code>job1</code>、<code>job2</code>、<code>job3</code>。</p><ol start="6">
<li><code>jobs.&lt;job_id&gt;.runs-on</code></li>
</ol><p><code>runs-on</code>字段指定运行所需要的虚拟机环境，它是必填字段。目前可用的虚拟机如下：</p><ul>
<li>ubuntu-latest、ubuntu-18.04或ubuntu-16.04。</li>
<li>windows-latest、windows-2019或windows-2016。</li>
<li>macOS-latest或macOS-10.14。</li>
</ul><p>下面的配置指定虚拟机环境为<code>ubuntu-18.04</code>。</p><pre><code class="language-yaml">runs-on: ubuntu-18.04
</code></pre><ol start="7">
<li><code>jobs.&lt;job_id&gt;.steps</code></li>
</ol><p><code>steps</code>字段指定每个 Job 的运行步骤，可以包含一个或多个步骤。每个步骤都可以指定下面三个字段。</p><ul>
<li><code>jobs.&lt;job_id&gt;.steps.name</code>：步骤名称。</li>
<li><code>jobs.&lt;job_id&gt;.steps.run</code>：该步骤运行的命令或者 action。</li>
<li><code>jobs.&lt;job_id&gt;.steps.env</code>：该步骤所需的环境变量。</li>
</ul><p>下面是一个完整的 workflow 文件的范例：</p><pre><code class="language-yaml">name: Greeting from Mona
on: push

jobs:
  my-job:
    name: My Job
    runs-on: ubuntu-latest
    steps:
    - name: Print a greeting
      env:
        MY_VAR: Hello! My name is
        FIRST_NAME: Lingfei
        LAST_NAME: Kong
      run: |
        echo $MY_VAR $FIRST_NAME $LAST_NAME.
</code></pre><p>上面的代码中，<code>steps</code>字段只包括一个步骤。该步骤先注入三个环境变量，然后执行一条 Bash 命令。</p><ol start="8">
<li><code>uses</code></li>
</ol><p><code>uses</code> 可以引用别人已经创建的 actions，就是上面说的 actions 市场中的 actions。引用格式为<code>userName/repoName@verison</code>，例如<code>uses: actions/setup-go@v1</code>。</p><ol start="9">
<li><code>with</code></li>
</ol><p><code>with</code> 指定actions的输入参数。每个输入参数都是一个键/值对。输入参数被设置为环境变量，该变量的前缀为 <code>INPUT_</code>，并转换为大写。</p><p>这里举个例子：我们定义 <code>hello_world</code> 操作所定义的三个输入参数（<code>first_name</code>、<code>middle_name</code> 和 <code>last_name</code>），这些输入变量将被 <code>hello-world</code> 操作作为 <code>INPUT_FIRST_NAME</code>、<code>INPUT_MIDDLE_NAME</code> 和 <code>INPUT_LAST_NAME</code> 环境变量使用。</p><pre><code class="language-yaml">jobs:
  my_first_job:
    steps:
      - name: My first step
        uses: actions/hello_world@master
        with:
          first_name: Lingfei
          middle_name: Go
          last_name: Kong
</code></pre><ol start="10">
<li><code>run</code></li>
</ol><p><code>run</code>指定执行的命令。可以有多个命令，例如：</p><pre><code class="language-yaml">- name: Build
      run: |
      go mod tidy
      go build -v -o helloci .
</code></pre><ol start="11">
<li><code>id</code></li>
</ol><p><code>id</code>是step的唯一标识。</p><h2>GitHub Actions的进阶用法</h2><p>上面，我介绍了GitHub Actions的一些基本知识，这里我再介绍下GitHub Actions的进阶用法。</p><h3>为工作流加一个Badge</h3><p>在action的面板中，点击<code>Create status badge</code>就可以复制Badge的Markdown内容到README.md中。</p><p>之后，我们就可以直接在README.md中看到当前的构建结果：</p><p><img src="https://static001.geekbang.org/resource/image/45/af/453a97b0776281873dee5671c53347af.png?wh=1280x765" alt="图片"></p><h3>使用构建矩阵</h3><p>如果我们想在多个系统或者多个语言版本上测试构建，就需要设置构建矩阵。例如，我们想在多个操作系统、多个Go版本下跑测试，可以使用如下workflow配置：</p><pre><code class="language-yaml">name: Go Test

on: [push, pull_request]

jobs:

  helloci-build:
    name: Test with go ${{ matrix.go_version }} on ${{ matrix.os }}
    runs-on: ${{ matrix.os }}

    strategy:
      matrix:
        go_version: [1.15, 1.16]
        os: [ubuntu-latest, macOS-latest]

    steps:

      - name: Set up Go ${{ matrix.go_version }}
        uses: actions/setup-go@v2
        with:
          go-version: ${{ matrix.go_version }}
        id: go
</code></pre><p>上面的workflow配置，通过<code>strategy.matrix</code>配置了该工作流程运行的环境矩阵（格式为<code>go_version.os</code>）：<code>ubuntu-latest.1.15</code>、<code>ubuntu-latest.1.16</code>、<code>macOS-latest.1.15</code>、<code>macOS-latest.1.16</code>。也就是说，会在4台不同配置的服务器上执行该workflow。</p><h3>使用Secrets</h3><p>在构建过程中，我们可能需要用到<code>ssh</code>或者<code>token</code>等敏感数据，而我们不希望这些数据直接暴露在仓库中，此时就可以使用<code>secrets</code>。</p><p>我们在对应项目中选择<code>Settings</code>-&gt; <code>Secrets</code>，就可以创建<code>secret</code>，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/c0/d3/c00b11a1709838c1a205ace7976768d3.png?wh=1920x1046" alt="图片"></p><p>配置文件中的使用方法如下：</p><pre><code class="language-yaml">name: Go Test
on: [push, pull_request]
jobs:
  helloci-build:
    name: Test with go
    runs-on: [ubuntu-latest]
    environment:
      name: helloci
    steps:
      - name: use secrets
        env:
          super_secret: ${{ secrets.YourSecrets }}
</code></pre><p>secret name不区分大小写，所以如果新建secret的名字是name，使用时用 <code>secrets.name</code> 或者 <code>secrets.Name</code> 都是可以的。而且，就算此时直接使用 <code>echo</code> 打印 <code>secret</code> , 控制台也只会打印出<code>*</code>来保护secret。<br>
这里要注意，你的secret是属于某一个环境变量的，所以要指明环境的名字：<code>environment.name</code>。上面的workflow配置中的<code>secrets.YourSecrets</code>属于<code>helloci</code>环境。</p><h3>使用Artifact保存构建产物</h3><p>在构建过程中，我们可能需要输出一些构建产物，比如日志文件、测试结果等。这些产物可以使用Github Actions Artifact 来存储。你可以使用<a href="https://github.com/actions/upload-artifact">action/upload-artifact</a> 和 <a href="https://github.com/actions/download-artifact">download-artifact</a> 进行构建参数的相关操作。</p><p>这里我以输出Jest测试报告为例来演示下如何保存Artifact产物。Jest测试后的测试产物是coverage：</p><pre><code class="language-yaml">steps:
      - run: npm ci
      - run: npm test

      - name: Collect Test Coverage File
        uses: actions/upload-artifact@v1.0.0
        with:
          name: coverage-output
          path: coverage
</code></pre><p>执行成功后，我们就能在对应action面板看到生成的Artifact：</p><p><img src="https://static001.geekbang.org/resource/image/4c/66/4c4a8d6aec12a5dd1cdc80d238472566.png?wh=1280x208" alt="图片"></p><h2>GitHub Actions实战</h2><p>上面，我介绍了GitHub Actions的用法，接下来我们就来实战下，看下使用GitHub Actions的6个具体步骤。</p><p><strong>第一步，</strong>创建一个测试仓库。</p><p>登陆<a href="https://github.com/">GitHub官网</a>，点击<strong>New repository</strong>创建，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/6d/a0/6d76d02f0418671a32f5346fccf616a0.png?wh=1920x810" alt="图片"></p><p>这里，我们创建了一个叫<code>helloci</code>的测试项目。</p><p><strong>第二步，</strong>将新的仓库 clone 下来，并添加一些文件：</p><pre><code class="language-bash">$ git clone https://github.com/marmotedu/helloci
</code></pre><p>你可以克隆<a href="https://github.com/marmotedu/helloci">marmotedu/helloci</a>，并将里面的文件拷贝到你创建的项目仓库中。</p><p><strong>第三步，</strong>创建GitHub Actions workflow配置目录：</p><pre><code class="language-bash">$ mkdir -p .github/workflows                     
</code></pre><p><strong>第四步，</strong>创建GitHub Actions workflow配置。</p><p>在<code>.github/workflows</code>目录下新建<code>helloci.yml</code>文件，内容如下：</p><pre><code class="language-yaml">name: Go Test

on: [push, pull_request]

jobs:

  helloci-build:
    name: Test with go ${{ matrix.go_version }} on ${{ matrix.os }}
    runs-on: ${{ matrix.os }}
    environment:
      name: helloci

    strategy:
      matrix:
        go_version: [1.16]
        os: [ubuntu-latest]

    steps:

      - name: Set up Go ${{ matrix.go_version }}
        uses: actions/setup-go@v2
        with:
          go-version: ${{ matrix.go_version }}
        id: go

      - name: Check out code into the Go module directory
        uses: actions/checkout@v2

      - name: Tidy
        run: |
          go mod tidy

      - name: Build
        run: |
          go build -v -o helloci .

      - name: Collect main.go file
        uses: actions/upload-artifact@v1.0.0
        with:
          name: main-output
          path: main.go

      - name: Publish to Registry
        uses: elgohr/Publish-Docker-GitHub-Action@master
        with:
          name: ccr.ccs.tencentyun.com/marmotedu/helloci:beta  # docker image 的名字
          username: ${{ secrets.DOCKER_USERNAME}} # 用户名
          password: ${{ secrets.DOCKER_PASSWORD }} # 密码
          registry: ccr.ccs.tencentyun.com # 腾讯云Registry
          dockerfile: Dockerfile # 指定 Dockerfile 的位置
          tag_names: true # 是否将 release 的 tag 作为 docker image 的 tag
</code></pre><p>上面的workflow文件定义了当GitHub仓库有<code>push</code>、<code>pull_request</code>事件发生时，会触发GitHub Actions工作流程，流程中定义了一个任务（Job）<code>helloci-build</code>，Job中包含了多个步骤（Step），每个步骤又包含一些动作（Action）。</p><p>上面的workflow配置会按顺序执行下面的6个步骤。</p><ol>
<li>准备一个Go编译环境。</li>
<li>从<a href="https://github.com/marmotedu/helloci">marmotedu/helloci</a>下载源码。</li>
<li>添加或删除缺失的依赖包。</li>
<li>编译Go源码。</li>
<li>上传构建产物。</li>
<li>构建镜像，并将镜像push到<code>ccr.ccs.tencentyun.com/marmotedu/helloci:beta</code>。</li>
</ol><p><strong>第五步，</strong>在push代码之前，我们需要先创建<code>DOCKER_USERNAME</code>和<code>DOCKER_PASSWORD</code> secret。</p><p>其中，<code>DOCKER_USERNAME</code>保存腾讯云镜像服务（CCR）的用户名，<code>DOCKER_PASSWORD</code>保存CCR的密码。我们将这两个secret保存在<code>helloci</code> Environments中，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/c0/d3/c00b11a1709838c1a205ace7976768d3.png?wh=1920x1046" alt="图片"></p><p><strong>第六步，</strong>将项目push到GitHub，触发workflow工作流：</p><pre><code class="language-bash">$ git add .
$ git push origin master
</code></pre><p>打开我们的仓库 Actions 标签页，可以发现GitHub Actions workflow正在执行：</p><p><img src="https://static001.geekbang.org/resource/image/1a/8a/1afb7860d68635c5e3eaba4ff8da208a.png?wh=1920x691" alt="图片"></p><p>等workflow执行完，点击 <strong>Go Test</strong> 进入构建详情页面，在详情页面能够看到我们的构建历史：</p><p><img src="https://static001.geekbang.org/resource/image/a4/95/a4b83a122379db4f2fe9538afdfb5a95.png?wh=1920x701" alt="图片"></p><p>然后，选择其中一个构建记录，查看其运行详情（具体可参考<a href="https://github.com/marmotedu/helloci/actions/runs/1144156183">chore: update step name Go Test #10</a>）：</p><p><img src="https://static001.geekbang.org/resource/image/48/4f/481f64aabccf30ed61d0a7c85ab30d4f.png?wh=1920x1084" alt="图片"></p><p>你可以看到，<code>Go Test</code>工作流程执行了6个Job，每个Job执行了下面这些自定义Step：</p><ol>
<li>Set up Go 1.16。</li>
<li>Check out code into the Go module directory。</li>
<li>Tidy。</li>
<li>Build。</li>
<li>Collect main.go file。</li>
<li>Publish to Registry。</li>
</ol><p>其他步骤是GitHub Actions自己添加的步骤：<code>Setup Job</code>、<code>Post Check out code into the Go module directory</code>、<code>Complete job</code>。点击每一个步骤，你都能看到它们的详细输出。</p><h2>IAM GitHub Actions实战</h2><p>接下来，我们再来看下IAM项目的GitHub Actions实战。</p><p>假设IAM项目根目录为 <code>${IAM_ROOT}</code>，它的workflow配置文件为：</p><pre><code class="language-bash">$ cat ${IAM_ROOT}/.github/workflows/iamci.yaml
name: IamCI

on:
  push:
    branchs:
    - '*'
  pull_request:
    types: [opened, reopened]

jobs:

  iamci:
    name: Test with go ${{ matrix.go_version }} on ${{ matrix.os }}
    runs-on: ${{ matrix.os }}
    environment:
      name: iamci

    strategy:
      matrix:
        go_version: [1.16]
        os: [ubuntu-latest]

    steps:

      - name: Set up Go ${{ matrix.go_version }}
        uses: actions/setup-go@v2
        with:
          go-version: ${{ matrix.go_version }}
        id: go

      - name: Check out code into the Go module directory
        uses: actions/checkout@v2

      - name: Run go modules Tidy
        run: |
          make tidy

      - name: Generate all necessary files, such as error code files
        run: |
          make gen

      - name: Check syntax and styling of go sources
        run: |
          make lint

      - name: Run unit test and get test coverage
        run: |
          make cover

      - name: Build source code for host platform
        run: |
          make build

      - name: Collect Test Coverage File
        uses: actions/upload-artifact@v1.0.0
        with:
          name: main-output
          path: _output/coverage.out

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v1

      - name: Login to DockerHub
        uses: docker/login-action@v1
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build docker images for host arch and push images to registry
        run: |
          make push
</code></pre><p>上面的workflow依次执行了以下步骤：</p><ol>
<li>设置Go编译环境。</li>
<li>下载IAM项目源码。</li>
<li>添加/删除不需要的Go包。</li>
<li>生成所有的代码文件。</li>
<li>对IAM源码进行静态代码检查。</li>
<li>运行单元测试用例，并计算单元测试覆盖率是否达标。</li>
<li>编译代码。</li>
<li>收集构建产物<code>_output/coverage.out</code>。</li>
<li>配置Docker构建环境。</li>
<li>登陆DockerHub。</li>
<li>构建Docker镜像，并push到DockerHub。</li>
</ol><p>IamCI workflow运行历史如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/2b/b0/2b542f9101be0c3a83576fb99bf882b0.png?wh=1920x844" alt="图片"></p><p>IamCI workflow的其中一次工作流程运行结果如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/e9/6a/e9ebf13fdb6e4f41a1b00406e646ec6a.png?wh=1920x887" alt="图片"></p><h2>总结</h2><p>在Go项目开发中，我们需要通过CI任务来将需要频繁操作的任务自动化，这不仅可以提高开发效率，还能减少手动操作带来的失误。这一讲，我选择了最易实践的GitHub Actions，来给你演示如何构建CI任务。</p><p>GitHub Actions支持通过push事件来触发CI流程。一个CI流程其实就是一个workflow，workflow中包含多个任务，这些任务是可以并行执行的。一个任务又包含多个步骤，每一步又由多个动作组成。动作（Action）其实是一个命令/脚本，用来完成我们指定的任务，如编译等。</p><p>因为GitHub Actions内容比较多，这一讲只介绍了一些核心的知识，更详细的GitHub Actions教程，你可以参考 <a href="https://docs.github.com/cn/actions">官方中文文档</a>。</p><h2>课后练习</h2><ol>
<li>使用CODING实现IAM的CI任务，并思考下：GitHub Actions和CODING在CI任务构建上，有没有本质的差异？</li>
<li>这一讲，我们借助GitHub Actions实现了CI，请你结合前面所学的知识，实现IAM的CD功能。欢迎提交Pull Request。</li>
</ol><p>这是我们这门课的最后一次练习题了，欢迎把你的思考和想法分享在留言区，也欢迎把课程分享给你的同事、朋友，我们一起交流，一起进步。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/52/40/e57a736e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>pedro</span>
  </div>
  <div class="_2_QraFYR_0">最后一讲留个言，专栏基本覆盖 Go 技术栈的方方面面，还有很多工具的加餐，项目开发规范，云原生，容器等知识，物超所值。<br><br>代码质量很高，学习了很多，一路走来，多谢了～</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢老哥能够坚持学习到最后</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-28 08:32:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/ff/5f/2a16164d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>gopher523</span>
  </div>
  <div class="_2_QraFYR_0">真的物超所值，大厂的工程师 真不一样啊，膜拜了，各种规范 各种设计。唉 如果给我一次重新上大学的机会 我一定要努力校招上。现在社招压力大啊</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-11 14:21:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/28/83/17/df99b53d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>随风而过</span>
  </div>
  <div class="_2_QraFYR_0">整个专栏质量很高，文案虽然有些瑕疵，不影响整体专栏的专业度，专栏介绍了很多编程规范，主要还是云原生范畴内，看完整个专栏有很多反思，对go语言自我的认知有一个全新的提高(比如项目目录参杂其他语言的习惯目录结构来做是错误的，还有代码规范也会参照其他语言来组织)。<br>也到说再见的时候了，希望老师在出高质量的专栏，订阅破万，与君共勉。。。。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢老哥的支持。后面可以随时群内交流Go项目开发技术</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-30 10:25:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/7f/d3/b5896293.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Realm</span>
  </div>
  <div class="_2_QraFYR_0">很专业、很系统，感谢老师的指引。<br><br>内容覆盖了编程技巧、工程化、云原生实践的经验总结，当然还有加餐鸡腿.<br><br>收获很大，谢谢老师！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-29 08:04:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/d7/f1/ce10759d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>wei 丶</span>
  </div>
  <div class="_2_QraFYR_0">老师太强了 感觉涵盖了 各个方面的</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-18 16:00:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/35/7c/b9/80d91f82.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>҉我爱小笨蛋</span>
  </div>
  <div class="_2_QraFYR_0">质量非常高，是我买的所有教材里面，最值的，没有之一！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-24 18:07:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/4d/26/44095eba.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>SuperSu</span>
  </div>
  <div class="_2_QraFYR_0">非常棒，干货满满</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-28 08:56:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2d/7b/b3/7c6665e6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>LM</span>
  </div>
  <div class="_2_QraFYR_0">实现方式很多，但是老师总结了最佳实践，物超所值</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-27 23:30:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/28/82/22/518026cf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Prince_H_23</span>
  </div>
  <div class="_2_QraFYR_0">第一遍还有很多需要消化，期待再次回顾～</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-23 22:02:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKELX1Rd1vmLRWibHib8P95NA87F4zcj8GrHKYQL2RcLDVnxNy1ia2geTWgW6L2pWn2kazrPNZMRVrIg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jxs1211</span>
  </div>
  <div class="_2_QraFYR_0">gitlab对应的工具叫啥</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: GitlabCI</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-22 09:05:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/c9/60/2f7eb4b5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>dairongpeng</span>
  </div>
  <div class="_2_QraFYR_0">物超所值，非常感谢作者分享自己的优秀实践，自身在go语言领域又上一个台阶</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢老哥支持！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-08 09:02:10</div>
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
  <div class="_2_QraFYR_0">感谢分享，非常全面的Go工程实践，建议码农们都来学习学习，不限使用语言：）</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 66666</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-17 00:30:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/96/88/454e401c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>销毁first</span>
  </div>
  <div class="_2_QraFYR_0">堪称go技术栈的软件工程，收货颇丰，谢谢老师</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-04 00:17:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2a/ac/9d/1f697753.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>米兔</span>
  </div>
  <div class="_2_QraFYR_0">专栏内容很丰富，一讲就需要很多时间消化了，谢谢！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-21 11:21:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/67/3a/0dd9ea02.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Summer  空城</span>
  </div>
  <div class="_2_QraFYR_0">老哥，请教个多了服务公用common包的问题，什么样的内容应该抽出来一个独立的sdk包，供各个微服务引用？我们现在只要有两个及以上微服务重复的代码，都放在一个core lib中，甚至于一些rpc调用、配置中心查询都放在这个core lib中。我觉得这是一个不好的设计，及时两个微服务中有一些重复的代码也是ok的。请教下老哥我理解的对么？什么内容可以放在core lib中？麻烦老哥。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感觉还是可以抽离出这样一个common包的。跟项目完全解耦的包都可以放在core lib中。<br><br>但我感觉这个core lib应该要有明确的边界什么样的包可以加入进去，并且形成规范。<br><br>当然包也可以不放在core lib中，比如A项目下有pkg&#47;util&#47;net包，这个包可以供别的项目引用，例如B项目。但如果A项目也引用到B的包，那可能会出现循环引用，也使包的引用变得复杂。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-15 13:05:17</div>
  </div>
</div>
</div>
</li>
</ul>