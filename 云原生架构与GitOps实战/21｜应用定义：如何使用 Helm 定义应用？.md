<audio title="21｜应用定义：如何使用 Helm 定义应用？" src="https://static001.geekbang.org/resource/audio/59/6b/5984b8732c490f9803438815a5ca806b.mp3" controls="controls"></audio> 
<p>你好，我是王炜。</p><p>在上一节课，我们学习了如何使用 Kustomize 定义应用。在将示例应用改造成 Kustomize 应用的过程中，我介绍了 Kustomize base 和 overlay 的概念，并通过 3 个不同环境的配置差异来说明如何对 base 通用资源的字段值进行覆盖。</p><p>在使用 Kustomize 对某个对象进行覆写时，你可能注意到了一个细节，那就是我们需要了解 base 目录下通用 Kubernetes 对象的具体细节，例如工作负载的名称和类型，以及字段的结构层级。当业务应用比较简单的时候，由于 Kubernetes 对象并不多，所以提前了解这些细节并没有太大的问题。</p><p>但是，当业务应用变得复杂，例如有数十个微服务场景时，那么 Kubernetes 对象可能会有上百个之多，这时候 Kustomize 的应用定义方式可能就会变得难以维护，尤其是当我们在 &nbsp;kustomization.yaml 文件定义大量覆写操作时，这种隐式的定义方式会让人产生迷惑。</p><p>其次，如果我们站在应用的发行角度来说，你会发现 Kustomize 对最终用户暴露所有Kubernetes 对象概念的方式太过于底层，我们可能需要一种更上层的应用定义方式。</p><p><strong>所以，社区诞生了另一种应用定义方式：Helm。</strong></p><!-- [[[read_end]]] --><p>Helm 是一种真正意义上的 Kubernetes 应用的包管理工具，它对最终用户屏蔽了 Kubernetes 对象概念，将复杂度左移到了应用开发者侧，终端用户只需要提供安装参数，就可以将应用安装到 Kubernetes 集群内。</p><p>在这节课，我仍然以示例应用为例子，把它从原始的 Kubernetes Manifest 改造成 Helm 应用，在实践的过程中带你进一步了解 Helm 相关的概念以及具体用法。当然，要想在一节课内完整介绍 Helm 是有困难的，所以我还是从实战入手，尽可能精简内容，以便让你快速掌握 Helm 的基本使用方法。</p><p>在进入实战之前，你需要在本地安装 Helm，具体你可以参考<a href="https://time.geekbang.org/column/article/624150">第 19 讲</a>的内容，同时将<a href="https://github.com/lyzhang1999/kubernetes-example.git">示例应用</a>代码仓库克隆到本地，仓库中也包含了这节课的代码，供你参考。</p><h2>实战简介和基本概念</h2><p>我们先来看将示例应用改造成 Helm 之后，我们期望得到的效果，如下图所示。</p><p><img src="https://static001.geekbang.org/resource/image/10/b6/10824b9b9bda823cbe28e2203325c1b6.jpg?wh=1920x1038" alt="图片"></p><p>我们希望实现和上一讲 Kustomize 一样的效果，将同一个 Helm Chart 部署到 Dev（开发）、Staging（预发布）、Prod（生产） 三个环境时，控制不同环境的 HPA、数据库以及镜像版本配置。</p><p>环境差异方面和上一讲提到的一致，除了 Prod 环境以外，其他两个环境都使用在集群内部署的 postgres 数据库，另外，三个环境的 HPA CPU 配置也不同。</p><p>上面的图中出现两个新的概念：Helm Chart 和 values.yaml。接下来我们简单介绍一下它们。</p><h3>Helm Chart 和 values.yaml</h3><p>Chart 是 Helm 的一种应用封装格式，它由一些特定文件和目录组成。为了方便 Helm Chart 存储、分发和下载，它采用 tgz 的格式对文件和目录进行打包。</p><p>在第 19 讲中，我们通过 Helm 命令安装了 Cert-manager，实际上 Helm 会先从指定的仓库下载 tgz 格式的 Helm Chart，然后再进行安装。</p><p>一个标准的 Helm Chart 目录结构是下面这样的。</p><pre><code class="language-powershell">$ ls
Chart.yaml&nbsp; templates&nbsp; &nbsp;values.yaml
</code></pre><p>其中，Chart.yaml 文件是 Helm Chart 的描述文件，例如名称、描述和版本等。</p><p>templates 目录用来存放模板文件，你可以把它视作 Kubernetes Manifest，但它和 Manifest 最大的区别是，模板文件可以包含变量，变量的值则来自于 values.yaml 文件定义的内容。</p><p>values.yaml 文件是安装参数定义文件，它不是必需的。在 Helm Chart 被打包成 tgz 包时，如果 templates 目录下的 Kubernetes Manifest 包含变量，那么你需要通过它来提供默认的安装参数。作为最终用户，当安装某一个 Helm Chart 的时候，也可以提供额外的 YAML 文件来覆盖默认值。比如在上面的期望效果图中，我们为同一个 Helm Chart 提供不同的安装参数，就可以得到具有配置差异的多套环境。</p><h3>Helm Release</h3><p>Helm Release 实际上是一个“安装”阶段的概念，它指的是本次安装的唯一标识（名称）。</p><p>我们知道 Helm Chart 实际上是一个应用安装包，只有在安装（实例化）它时才会生效。它可以在同一个集群中甚至是同一个命名空间下安装多次，所以我们就需要为每次安装都起一个唯一的名字，这就是 Helm Release Name。</p><h2>改造示例应用</h2><p>接下来，我们开始改造示例应用，<strong>下面所有的命令都默认在示例应用根目录下执行。</strong></p><h3>创建 Helm Chart 目录结构</h3><p>首先，进入示例应用目录并创建 helm 目录。</p><pre><code class="language-powershell">$ cd kubernetes-example &amp;&amp; mkdir helm
</code></pre><p>然后，我们根据 Helm Chart 的目录结构进一步创建 templates 目录、Chart.yaml 以及 values.yaml。</p><pre><code class="language-powershell">$ mkdir helm/templates &amp;&amp; touch helm/Chart.yaml &amp;&amp; touch helm/values.yaml
</code></pre><p>现在，helm 目录结构是下面这样。</p><pre><code class="language-powershell">$ ls helm &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;
Chart.yaml&nbsp; templates&nbsp; &nbsp;values.yaml
</code></pre><p>到这里，Helm Chart 目录结构就创建完成了。</p><h3>配置 Chart.yaml 内容</h3><p>接下来，我们需要将 Helm Chart 的基础信息写入 Chart.yaml 文件中，将下面的内容复制到 Chart.yaml 文件内。</p><pre><code class="language-powershell">apiVersion: v2
name: kubernetes-example
description: A Helm chart for Kubernetes
type: application
version: 0.1.0
appVersion: "0.1.0"
</code></pre><p>其中，apiVersion 字段设置为 v2，代表使用 Helm 3 来安装应用。</p><p>name 表示 Helm Chart 的名称，当使用 helm install 命令安装 Helm Chart 时，指定的名称也就是这里配置的名称。</p><p>description 表示 Helm Chart 的描述信息，你可以根据需要填写。</p><p>type 表示类型，这里我们将其固定为 application，代表 Kubernetes 应用。</p><p>version 表示我们打包的 Helm Chart 的版本，当使用 helm install 时，可以指定这里定义的版本号。Helm Chart 的版本就是通过这个字段来管理的，当我们发布新的 Chart 时，需要更新这里的版本号。</p><p>appVersion 和 Helm Chart 无关，它用于定义应用的版本号，建立当前 Helm Chart 和应用版本的关系。</p><h3>最简单的 Helm Chart</h3><p>之前我们提到过，helm/tamplates 目录是用来存放模板文件的，这个模板文件也可以是 Kubernetes Manifest。所以，我们现在尝试不使用 Helm Chart 的模板功能，而是直接将 deploy 目录下的 Kubernetes Manifest 复制到 helm/tamplates 目录下。</p><pre><code class="language-powershell">$ cp -r deploy/ helm/templates/
</code></pre><p>现在，helm 目录的结构如下。</p><pre><code class="language-powershell">helm
├── Chart.yaml
├── templates
│&nbsp; &nbsp;└── deploy
│&nbsp; &nbsp; &nbsp; &nbsp;├── backend.yaml
│&nbsp; &nbsp; &nbsp; &nbsp;├── database.yaml
│&nbsp; &nbsp; &nbsp; &nbsp;├── frontend.yaml
│&nbsp; &nbsp; &nbsp; &nbsp;├── hpa.yaml
│&nbsp; &nbsp; &nbsp; &nbsp;└── ingress.yaml
└── values.yaml
</code></pre><p>其中，values.yaml 的文件内容为空。</p><p>到这里，<strong>一个最简单的 Helm Chart 就编写完成了</strong>。在这个 Helm Chart 中，templates 目录下的 Manifest 的内容是确定的，安装这个 Helm Chart 等同于使用 kubectl apply 命令，接下来我们尝试使用 helm install 命令来安装这个 Helm Chart。</p><pre><code class="language-powershell">$ helm install my-kubernetes-example ./helm --namespace example --create-namespace
NAME: my-kubernetes-example
LAST DEPLOYED: Thu Oct 20 15:55:31 2022
NAMESPACE: example
STATUS: deployed
REVISION: 1
TEST SUITE: None
</code></pre><p>在上面这条命令中，我们指定了应用的名称为 my-kubernetes-example，Helm Chart 目录为 ./helm 目录，并且为应用指定了命名空间为 example。要注意的是，example 命名空间并不存在，所以我同时使用 --create-namespace 来让 Helm 自动创建这个命名空间。</p><p><strong>此外，这里还有一个非常重要的概念：Release Name</strong>。在安装时，我们需要指定 Release Name 也就是 my-kubernetes-example，它和 Helm Chart Name 有本质的区别。Release Name 是在安装时指定的，Helm Chart Name 在定义阶段就已经固定了。</p><p>注意，命令运行完成后，只能代表 Helm 已经将 Manifest 应用到了集群内，并不能表示应用已经就绪了。接下来 Kubernetes 集群会完成拉取镜像和 Pod 调度的操作，这些都是异步的。</p><h3>使用模板变量</h3><p>不过，刚才改造的最简单的 Helm Chart 并不能满足我们的目标。到目前为止，它只是一个纯静态的应用，无法应对多环境对配置差异的需求。</p><p>要将这个静态的 Helm Chart 改造成参数动态可控制的，<strong>我们需要用到模板变量和 values.yaml</strong>。</p><p>还记得我之前提到的 values.yaml 的概念吗？模板变量的值都会引自这个文件。在这个例子中，根据我们对不同环境配置差异化的要求，我抽象了这几个可配置项：</p><ul>
<li>镜像版本</li>
<li>HPA CPU 平均使用率</li>
<li>是否启用集群内数据库</li>
<li>数据库连接地址、账号和密码</li>
</ul><p>这些可配置项都需要从 values.yaml 文件中读取，所以，你需要将下面的内容复制到 helm/values.yaml 文件内。</p><pre><code class="language-powershell">frontend:
  image: lyzhang1999/frontend
  tag: latest
  autoscaling:
    averageUtilization: 90

backend:
  image: lyzhang1999/backend
  tag: latest
  autoscaling:
    averageUtilization: 90

database:
  enabled: true
  uri: pg-service
  username: postgres
  password: postgres
  
</code></pre><p>除了 values.yaml，我们还需要让 helm/templates 目录下的文件能够读取到 values.yaml 的内容，这就需要模板变量了。</p><p>举一个最简单的例子，要读取 values.yaml 中的 frontend.image 字段，可以通过 “{{ .Values.frontend.image }}” 模板变量来获取值。</p><p>所以，要将这个“静态”的 Helm Chart 改造成“动态”的，<strong>我们只需要用模板变量来替换 templates 目录下需要实现“动态”的字段。</strong></p><p>了解原理后，我们来修改 helm/templates/backend.yaml 文件，用模板替换需要从 values.yaml 读取的字段。</p><pre><code class="language-powershell">apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  ......
spec:
  ......
    spec:
      containers:
      - name: flask-backend
        image: "{{ .Values.backend.image }}:{{ .Values.backend.tag }}"
        env:
        - name: DATABASE_URI
          value: "{{ .Values.database.uri }}"
        - name: DATABASE_USERNAME
          value: "{{ .Values.database.username }}"
        - name: DATABASE_PASSWORD
          value: "{{ .Values.database.password }}"
</code></pre><p>同理，修改 helm/templates/frontend.yaml 文件的 image 字段。</p><pre><code class="language-powershell">apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  ......
spec:
  ......
    spec:
      containers:
      - name: react-frontend
        image: "{{ .Values.frontend.image }}:{{ .Values.frontend.tag }}" 
</code></pre><p>此外，还需要修改 helm/templates/hpa.yaml 文件的 averageUtilization 字段。</p><pre><code class="language-powershell">......
metadata:
  name: frontend
spec:
  ......
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: {{ .Values.frontend.autoscaling.averageUtilization }}
---
......
metadata:
  name: backend
spec:
  ......
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: {{ .Values.backend.autoscaling.averageUtilization }}
</code></pre><p>注意，相比较其他的模板变量，在这里我们没有在模板变量的外部使用双引号，这是因为 averageUtilization 字段是一个 integer 类型，而双引号加上模板变量的意思是 string 类型。</p><p>最后，我们希望使用 values.yaml 中的 database.enable 字段来控制是否向集群提交helm/templates/database.yaml 文件。所以我们可以在文件首行和最后一行增加下面的内容。</p><pre><code class="language-powershell">{{- if .Values.database.enabled -}}
......
{{- end }}
</code></pre><p>到这里，我们就成功地将“静态”的 Helm Chart 改造为了“动态”的 Helm Chart。</p><h2>部署</h2><p>在将示例应用改造成 Helm Chart 之后，我们就可以使用 helm install 进行安装了。这里我会将 Helm Chart 分别安装到 helm-staging 和 helm-prod 命名空间，它们对应预发布环境和生产环境，接着我会介绍如何为不同的环境传递不同的参数。</p><h3>部署预发布环境</h3><p>我们为 Helm Chart 创建的 values.yaml 实际上是默认值，在预发布环境，我们希望将前后端的 HPA CPU averageUtilization 从默认的 90 调整为 60，你可以在安装时使用 --set 来调整特定的字段，不需要修改 values.yaml 文件。</p><pre><code class="language-powershell">$ helm install my-kubernetes-example ./helm --namespace helm-staging --create-namespace --set frontend.autoscaling.averageUtilization=60 --set backend.autoscaling.averageUtilization=60
NAME: my-kubernetes-example
LAST DEPLOYED: Thu Oct 20 18:13:34 2022
NAMESPACE: helm-staging
STATUS: deployed
REVISION: 1
TEST SUITE: None
</code></pre><p>在这个安装例子中，我们使用 --set 参数来调整 frontend.autoscaling.averageUtilization&nbsp;字段值，其它的字段值仍然采用 values.yaml 提供的默认值。</p><p>部署完成后，你可以查看我们为预发布环境配置的后端服务 HPA averageUtilization 字段值。</p><pre><code class="language-powershell">$ kubectl get hpa backend -n helm-staging --output jsonpath='{.spec.metrics[0].resource.target.averageUtilization}'
60
</code></pre><p>返回值为 60，和我们配置的安装参数一致。</p><p>同时，你也可以查看是否部署了数据库，也就是 postgres 工作负载。</p><pre><code class="language-powershell">$ kubectl get deployment postgres -n helm-staging
NAME&nbsp; &nbsp; &nbsp; &nbsp;READY&nbsp; &nbsp;UP-TO-DATE&nbsp; &nbsp;AVAILABLE&nbsp; &nbsp;AGE
postgres&nbsp; &nbsp;1/1&nbsp; &nbsp; &nbsp;1&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; 1&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;46s
</code></pre><p>postgres 工作负载存在，符合预期。</p><p>最后，你可以查看 backend Deployment 的 Env 环境变量，以便检查是否使用集群内的数据库实例。</p><pre><code class="language-powershell">$ kubectl get deployment backend -n helm-staging --output jsonpath='{.spec.template.spec.containers[0].env[*]}'

{"name":"DATABASE_URI","value":"pg-service"} {"name":"DATABASE_USERNAME","value":"postgres"} {"name":"DATABASE_PASSWORD","value":"postgres"}
</code></pre><p>返回结果同样符合预期。</p><h3>部署生产环境</h3><p>部署到生产环境的例子相对来说配置项会更多，除了需要修改 database.enable 字段，禁用集群内数据库实例以外，还需要修改数据库连接的三个环境变量。所以，我们使用另一种安装参数传递方式：<strong>使用文件传递。</strong></p><p>要使用文件来传递安装参数，首先需要准备这个文件。你需要将下面的内容保存为 helm/values-prod.yaml 文件。</p><pre><code class="language-powershell">frontend:
  autoscaling:
    averageUtilization: 50

backend:
  autoscaling:
    averageUtilization: 50

database:
  enabled: false
  uri: 10.10.10.10
  username: external_postgres
  password: external_postgres
</code></pre><p>在 values-prod.yaml 文件内，我们只需要提供覆写的 Key 而不需要原样复制默认的 values.yaml 文件内容，这个操作和 Kustomize 的 Patch 操作有一点类似。</p><p>接下来，我们使用 helm install 命令来安装它，同时指定新的 values-prod.yaml 文件作为安装参数。</p><pre><code class="language-powershell">$ helm install my-kubernetes-example ./helm -f ./helm/values-prod.yaml --namespace helm-prod --create-namespace
NAME: my-kubernetes-example
LAST DEPLOYED: Thu Oct 20 20:21:07 2022
NAMESPACE: helm-prod
STATUS: deployed
REVISION: 1
TEST SUITE: None
</code></pre><p>部署完成后，你可以查看我们为生产环境配置的后端服务 HPA averageUtilization 字段值。</p><pre><code class="language-powershell">$ kubectl get hpa backend -n helm-staging --output jsonpath='{.spec.metrics[0].resource.target.averageUtilization}'
50
</code></pre><p>返回值为 50，和我们在 values-prod.yaml 文件中定义的安装参数一致。</p><p>同时，你也可以查看是否部署了数据库，也就是 postgres 工作负载。</p><pre><code class="language-powershell">$ kubectl get deployment postgres -n helm-prod&nbsp; &nbsp;
Error from server (NotFound): deployments.apps "postgres" not found
</code></pre><p>可以发现，postgres 工作负载并不存在，符合预期。</p><p>最后，你可以查看 backend Deployment 的 Env 环境变量，检查是否使用了外部数据库。</p><pre><code class="language-powershell">$ kubectl get deployment backend -n helm-prod --output jsonpath='{.spec.template.spec.containers[0].env[*]}'

{"name":"DATABASE_URI","value":"10.10.10.10"} {"name":"DATABASE_USERNAME","value":"external_postgres"} {"name":"DATABASE_PASSWORD","value":"external_postgres"}
</code></pre><p>返回结果同样符合预期。</p><p>到这里，将实例应用改造成 Helm Chart 的工作已经全部完成了。</p><h2>发布 Helm Chart</h2><p>在 Helm Chart 编写完成之后，我们便能够在本地安装它了。不过，我们通常还会有和其他人分享 Helm Chart 的需求。</p><p>还记得我们在<a href="https://time.geekbang.org/column/article/624150">第 19 讲</a>提到的安装 Cert-manager 的例子吗？我们首先执行 helm repo add 命令添加了一个 Helm 仓库，然后使用 helm install 直接安装了一个远端仓库的 Helm Chart，这种方式和我们上面介绍的指定 Helm Chart 目录的安装方式并不相同。</p><p>那么，我们如何实现和 Cert-manager 相同的安装方式呢？</p><p>很简单，只需要将我们在上面创建的 Helm Chart 打包并且上传到 Helm 仓库中即可。</p><p>这里我以 GitHub Package 为例，介绍如何将 Helm Chart 上传到镜像仓库。</p><h3>创建 GitHub Token</h3><p>要将 Helm Chart 推送到 GitHub Package，首先我们需要创建一个具备推送权限的 Token，你可以在<a href="https://github.com/settings/tokens/new">这个链接</a>创建，并勾选 write:packages 权限。</p><p><img src="https://static001.geekbang.org/resource/image/13/14/13bdb1e9a119f081b9504df985de4d14.png?wh=1600x686" alt="图片"></p><p>点击“Genarate token”按钮生成 Token并复制。</p><p><img src="https://static001.geekbang.org/resource/image/2a/c0/2a486455276d180239dd7930b799fec0.png?wh=1626x514" alt="图片"></p><h3>推送 Helm Chart</h3><p>在推送之前，还需要使用 GitHub ID 和刚才创建的 Token 登录到 GitHub Package。</p><pre><code class="language-powershell">$ helm registry login -u lyzhang1999 https://ghcr.io
Password: token here
Login Succeeded
</code></pre><p>请注意，由于 GitHub Package 使用的是 OCI 标准的存储格式，如果你使用的 helm 版本小于 3.8.0，则需要在运行这条命令之前增加 HELM_EXPERIMENTAL_OCI=1 的环境变量启用实验性功能。</p><p>然后，返回到示例应用的根目录下，执行 helm package 命令来打包 Helm Chart。</p><pre><code class="language-powershell">$ helm package ./helm
Successfully packaged chart and saved it to: /Users/weiwang/Downloads/kubernetes-example/kubernetes-example-0.1.0.tgz
</code></pre><p>这条命令会将 helm 目录打包，并生成 kubernetes-example-0.1.0.tgz 文件。</p><p>接下来，就可以使用 helm push 命令推送到 GitHub Package 了。</p><pre><code class="language-powershell">$ helm push kubernetes-example-0.1.0.tgz oci://ghcr.io/lyzhang1999/helm

Pushed: ghcr.io/lyzhang1999/helm/kubernetes-example:0.1.0
Digest: sha256:8a0cc4a2ac00f5b1f7a50d6746d54a2ecc96df6fd419a70614fe2b9b975c4f42
</code></pre><p>命令运行结束后将展示 Digest 字段，就说明 Helm Chart 推送成功了。</p><h3>安装远端仓库的 Helm Chart</h3><p>当我们成功把 Helm Chart 推送到 GitHub Package 之后，就可以直接使用远端仓库来安装 Helm Chart 了。和一般的安装步骤不同的是，由于 GitHub Package 仓库使用的是 OCI 标准的存储方式，所以无需执行 helm repo add 命令添加仓库，可以直接使用 helm install 命令来安装。</p><pre><code class="language-powershell">$ helm install my-kubernetes-example oci://ghcr.io/lyzhang1999/helm/kubernetes-example --version 0.1.0 --namespace remote-helm-staging --create-namespace --set frontend.autoscaling.averageUtilization=60 --set backend.autoscaling.averageUtilization=60

Pulled: ghcr.io/lyzhang1999/helm/kubernetes-example:0.1.0
Digest: sha256:8a0cc4a2ac00f5b1f7a50d6746d54a2ecc96df6fd419a70614fe2b9b975c4f42

NAME: my-kubernetes-example
LAST DEPLOYED: Thu Oct 20 21:38:41 2022
NAMESPACE: remote-helm-staging
STATUS: deployed
REVISION: 1
TEST SUITE: None
</code></pre><p>在上面的安装命令中，oci://ghcr.io/lyzhang1999/helm/kubernetes-example 是 Helm Chart 的完整的地址，并标识了 OCI 关键字。</p><p>另外，version 字段指定的是 Helm Chart 的版本号。在安装时，同样可以使用 --set 或者指定 -f 参数来覆写 values.yaml 的字段。</p><h2>Helm 应用管理</h2><p>通过上面内容的学习，我相信你已经掌握了如何使用 Helm 来定义应用，在实际的工作中，这些知识也基本上够用了。当然，如果你希望深度使用 Helm，那么你还需要继续了解 Helm 应用管理功能和相关命令。</p><p>总结来说，Helm Chart 和 Manifest 之间一个最大的区别是，Helm 从应用的角度出发，提供了应用的管理功能，通常我们在实际使用 Helm 过程中会经常遇到下面几种场景。</p><ul>
<li>调试 Helm Chart。</li>
<li>查看已安装的 Helm Release。</li>
<li>更新 Helm Release。</li>
<li>查看 Helm Release 历史版本。</li>
<li>回滚 Helm Release。</li>
<li>卸载 Helm Release。</li>
</ul><p>接下来我们来看如何使用 Helm 命令行工具来实现这些操作。</p><h3>调试 Helm Chart</h3><p>在编写 Helm Chart 的过程中，为了方便验证，我们会经常渲染完整的 Helm 模板而又不安装它，这时候你就可以使用 helm template 命令来调试 Helm Chart。</p><pre><code class="language-powershell">$ helm template ./helm -f ./helm/values-prod.yaml
---
# Source: kubernetes-example/templates/backend.yaml
apiVersion: v1
kind: Service
......
---
# Source: kubernetes-example/templates/frontend.yaml
apiVersion: v1
kind: Service
......
</code></pre><p>此外，你还可以在运行 helm install 命令时增加 --dry-run 参数来实现同样的效果。</p><pre><code class="language-powershell">$ helm install my-kubernetes-example oci://ghcr.io/lyzhang1999/helm/kubernetes-example --version 0.1.0 --dry-run

Pulled: ghcr.io/lyzhang1999/helm/kubernetes-example:0.1.0
Digest: sha256:8a0cc4a2ac00f5b1f7a50d6746d54a2ecc96df6fd419a70614fe2b9b975c4f42
NAME: my-kubernetes-example
......
---
# Source: kubernetes-example/templates/database.yaml
......
</code></pre><h3>查看已安装的 Helm Release</h3><p>要查看已安装的 Helm Release，可以使用 helm list 命令。</p><pre><code class="language-powershell">$ helm list -A
NAME&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; NAMESPACE&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;REVISION&nbsp; &nbsp; &nbsp; &nbsp; UPDATED&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;STATUS&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; CHART&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;APP VERSION
my-kubernetes-example&nbsp; &nbsp;helm-prod&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;1&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;2022-10-20 20:21:07.658738 +0800 CST&nbsp; &nbsp; deployed&nbsp; &nbsp; &nbsp; &nbsp; kubernetes-example-0.1.0&nbsp; &nbsp; &nbsp; &nbsp; 0.1.0&nbsp; &nbsp; &nbsp;&nbsp;
my-kubernetes-example&nbsp; &nbsp;helm-staging&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; 1&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;2022-10-20 20:26:13.265669 +0800 CST&nbsp; &nbsp; deployed&nbsp; &nbsp; &nbsp; &nbsp; kubernetes-example-0.1.0&nbsp; &nbsp; &nbsp; &nbsp; 0.1.0&nbsp; &nbsp; &nbsp;&nbsp;
my-kubernetes-example&nbsp; &nbsp;remote-helm-staging&nbsp; &nbsp; &nbsp;1&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;2022-10-20 21:38:41.01361 +0800 CST&nbsp; &nbsp; &nbsp;deployed&nbsp; &nbsp; &nbsp; &nbsp; kubernetes-example-0.1.0&nbsp; &nbsp; &nbsp; &nbsp; 0.1.0

</code></pre><p>返回结果中展示了我们刚才安装的所有 Helm Release 以及它们所在的命名空间。</p><h3>更新 Helm Release</h3><p>要更新 Helm Release，可以使用 helm upgrade 命令，Helm 会自动对比新老版本之间的 Manifest 差异，并执行升级。</p><pre><code class="language-powershell">$ helm upgrade my-kubernetes-example ./helm -n example
Release "my-kubernetes-example" has been upgraded. Happy Helming!
NAME: my-kubernetes-example
LAST DEPLOYED: Thu Oct 20 16:31:25 2022
NAMESPACE: example
STATUS: deployed
REVISION: 2
TEST SUITE: None
</code></pre><h3>查看 Helm Release 历史版本</h3><p>要查看 Helm Release 的历史版本，你可以使用 helm history 命令。</p><pre><code class="language-powershell">$ helm history my-kuebrnetes-example -n example
REVISION&nbsp; &nbsp; &nbsp; &nbsp; UPDATED&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;STATUS&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; CHART&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;APP VERSION&nbsp; &nbsp; &nbsp;DESCRIPTION&nbsp; &nbsp; &nbsp;
1&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;Thu Oct 20 16:09:22 2022&nbsp; &nbsp; &nbsp; &nbsp; superseded&nbsp; &nbsp; &nbsp; kubernetes-example-0.1.0&nbsp; &nbsp; &nbsp; &nbsp; 0.1.0&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;Install complete
2&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;Thu Oct 20 16:31:25 2022&nbsp; &nbsp; &nbsp; &nbsp; deployed&nbsp; &nbsp; &nbsp; &nbsp; kubernetes-example-0.1.0&nbsp; &nbsp; &nbsp; &nbsp; 0.1.0&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;Upgrade complete
</code></pre><p>从返回结果来看，my-kuebrnetes-example Release 有两个版本（REVISION），分别是 1 和 2，我们还能看到每一个版本的状态。</p><h3>回滚 Helm Release</h3><p>当 Helm Release 有多个版本时，你可以通过 helm rollback 命令回滚到指定的版本。</p><pre><code class="language-powershell">$ helm rollback my-kubernetes-example 1 -n example
Rollback was a success! Happy Helming!
</code></pre><h3>卸载 Helm Release</h3><p>最后，要卸载 Helm Release，你可以使用 helm uninstall。</p><pre><code class="language-powershell">$ helm uninstall my-kubernetes-example -n example&nbsp; &nbsp; &nbsp;
release "my-kubernetes-example" uninstalled
</code></pre><h2>总结</h2><p>这节课，我们以示例应用为例子，介绍了如何使用 Helm Chart 来定义应用。</p><p>Helm Chart 实际上是由特定的文件和目录组成的，一个最简单的 Helm Chart 包含 Chart.yaml、values.yaml 和 templates 目录，当我们把这个特定的目录打包为 tgz 压缩文件时，实际上它也就是标准的 Helm Chart 格式。</p><p>相比较 Kustomize 和原生 Manifest，Helm Chart 更多是从“应用”的视角出发的，它为 Kubernetes 应用提供了打包、存储、发行和启动的能力，实际上它就是一个 Kubernetes 的应用包管理工具。此外，Helm 通过模板语言为我们提供了暴露应用关键参数的能力，使用者只需要关注安装参数而不需要去理解内部细节。</p><p>而 Kustomize 和 Manifest 则使用原生的 YAML 和 Kubernetes API 进行交互，不具备包管理的概念，所以在这方面它们之间有着本质的区别。</p><p>那么，<strong>如果把 Kubernetes 比作是操作系统，Helm Chart 其实就可以类比为 Windows 的应用安装包，它们都是应用的一种安装方式。</strong></p><p>在 Helm 的具体使用方面，我还介绍了如何通过 helm install 命令来安装两种类型的 Helm Chart，它们分别是本地目录和远端仓库。在安装时，它们都可以使用 --set 参数来对默认值覆写，也可以使用 -f 参数来指定新的 values.yaml 文件。此外，我还以 GitHub Package 为例子，介绍了如何打包 Helm Chart 并上传到 GitHub Package 仓库中。</p><p>最后，在 Helm 应用管理方面，我希望你能够熟记几条简单的命令，例如 helm list、helm upgrade 和 helm rollback 等，这些命令在工作中都是很常用的。</p><p>到这里，我们对应用定义的讲解就全部结束了，希望你能有所收获。</p><h2>思考题</h2><p>最后，给你留两道思考题吧。</p><ol>
<li>请你结合<a href="https://time.geekbang.org/column/article/622743">第 16 讲</a>的内容，为示例应用配置 GitHub Action，要求是每次提交代码后都自动打包 Helm Chart，并将它上传到 GitHub Package 中。你可以将 GitHub Action Workflow 的 YAML 内容放到留言中。</li>
<li>如何实现在同一个命名空间下对同一个 Helm Chart 安装多个 Helm Release？以示例应用为例，请你分享核心的思路。</li>
</ol><p>欢迎你给我留言交流讨论，你也可以把这节课分享给更多的朋友一起阅读。我们下节课见。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/f6/24/547439f1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ghostwritten</span>
  </div>
  <div class="_2_QraFYR_0">我的 centos  7.9.2009 &amp; helm v3.11.0 没有遇到拒绝问题。<br>$ helm version<br>version.BuildInfo{Version:&quot;v3.11.0&quot;, GitCommit:&quot;472c5736ab01133de504a826bd9ee12cbe4e7904&quot;, GitTreeState:&quot;clean&quot;, GoVersion:&quot;go1.18.10&quot;}<br> $ helm registry login -u ghostwritten https:&#47;&#47;ghcr.io<br>Password:<br>Login Succeeded<br><br>$ helm package .&#47;helm<br>Successfully packaged chart and saved it to: &#47;root&#47;github&#47;kubernetes-example&#47;kubernetes-example-0.1.0.tgz<br><br>$ helm push kubernetes-example-0.1.0.tgz oci:&#47;&#47;ghcr.io&#47;ghostwritten&#47;helm<br>Pushed: ghcr.io&#47;ghostwritten&#47;helm&#47;kubernetes-example:0.1.0<br>Digest: sha256:46bef623e43f4525ebfd25c368dfea69e70efbe7590f1e3eccc321fbb6b16882<br><br>$ helm install my-kubernetes-example oci:&#47;&#47;ghcr.io&#47;ghostwritten&#47;helm&#47;kubernetes-example --version 0.1.0 --namespace remote-helm-staging --create-namespace --set frontend.autoscaling.averageUtilization=60 --set backend.autoscaling.averageUtilization=60<br>Pulled: ghcr.io&#47;ghostwritten&#47;helm&#47;kubernetes-example:0.1.0<br>Digest: sha256:46bef623e43f4525ebfd25c368dfea69e70efbe7590f1e3eccc321fbb6b16882<br>W0202 16:15:56.276585    3957 warnings.go:70] autoscaling&#47;v2beta2 HorizontalPodAutoscaler is deprecated in v1.23+, unavailable in v1.26+; use autoscaling&#47;v2 HorizontalPodAutoscaler<br>AME: my-kubernetes-example<br>LAST DEPLOYED: Thu Feb  2 16:15:55 2023<br>NAMESPACE: remote-helm-staging<br>STATUS: deployed<br>REVISION: 1<br>TEST SUITE: None<br><br>$ k get pods -n remote-helm-staging<br>NAME                        READY   STATUS    RESTARTS      AGE<br>backend-bcb7687c6-s7lxh     1&#47;1     Running   0             29m<br>backend-bcb7687c6-v4cx6     1&#47;1     Running   0             29m<br>frontend-7c59d655fb-p6lpm   1&#47;1     Running   1 (21m ago)   29m<br>frontend-7c59d655fb-xnl8x   1&#47;1     Running   0             29m<br>postgres-7745b57d5d-2nndw   1&#47;1     Running   0             29m</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-02 16:58:36</div>
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
  <div class="_2_QraFYR_0">几个问题麻烦问您一下：<br>1. 虽然和Kustomize相比，Helm的管理粒度是在应用层面了，但是，就如同本节的例子，使用Helm将一套应用部署到多环境时，也和Kustomize一样需要持有对每个环境的特殊配置，虽然Helm的模板变量的方式比Kustomize的manifest补丁文件的方式更灵活、高效，但也避免不了对不同环境分别配置的信息的管理，Helm对这种环境特殊配置信息的管理的最佳实践是什么样的？是将不同环境的values文件通过git保管？还是其他什么方式？<br>2. 对于Helm仓库，是否有需要像Harbor之于docker hub那样部署一个私有的Helm仓库在公司内部用？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 非常棒的两个问题。<br>首先对于多环境的情况，一般的实践是通过多个 values.yaml 文件来实现，例如 values-dev.yaml，values-prod.yaml。<br>对于第二个问题，Harbor 支持存储 Helm Chart，你可以查看这个文档：https:&#47;&#47;goharbor.io&#47;docs&#47;1.10&#47;working-with-projects&#47;working-with-images&#47;managing-helm-charts&#47;</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-16 19:10:47</div>
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
  <div class="_2_QraFYR_0">这一章节要部署自己镜像仓库的镜像，是不是要把value.yaml里的lyzhang1999改成自己的dockerhub用户名哦</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的👍🏻</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-13 16:46:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2a/d1/34/03dc9e03.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>李多</span>
  </div>
  <div class="_2_QraFYR_0">思考1：<br>在 Github Actions 插件库中，有 Helm 发布相关的工具，如：Helm Chart Releaser (https:&#47;&#47;github.com&#47;marketplace&#47;actions&#47;helm-chart-releaser) 。在 steps 中使用 uses: helm&#47;chart-releaser-action@v1.5.0，利用这个工具可以实现 Helm 发布。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍🏻这个插件看起来不错~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-03 14:41:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/b3/24/8246675f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>天地有雪</span>
  </div>
  <div class="_2_QraFYR_0">可以了，原来ubuntu系统，helm 3.11版本，换成centos7 helm 3.8.0 没有问题</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍🏻</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-29 16:08:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/b3/24/8246675f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>天地有雪</span>
  </div>
  <div class="_2_QraFYR_0">helm version<br>version.BuildInfo{Version:&quot;v3.11.0&quot;, GitCommit:&quot;472c5736ab01133de504a826bd9ee12cbe4e7904&quot;, GitTreeState:&quot;clean&quot;, GoVersion:&quot;go1.18.10&quot;}<br><br>1<br>helm registry login -u mlkkkfriend https:&#47;&#47;ghcr.io<br>Password: <br>Error: Get &quot;https:&#47;&#47;ghcr.io&#47;v2&#47;&quot;: denied: denied<br>输入账号，密码无法登录<br><br>2 文章中提到 在推送之前，还需要使用 GitHub ID 和刚才创建的 Token 登录到 GitHub Package。 需要什么操作<br><br>3 echo $CR_PAT | docker login ghcr.io -u 用户 --password-stdin<br>可以登录<br>docker push 可以上传<br><br>4  echo $CR_PAT |  helm registry login -u mlkkkfriend https:&#47;&#47;ghcr.io --password-stdin<br>Login Succeeded<br>可以登录<br>helm push kubernetes-example-0.1.0.tgz oci:&#47;&#47;ghrc.io&#47;mlkkkfriend<br>Error: failed commit on ref &quot;manifest-sha256:3790edf4411c5d6fbf3e40548ebdf78979ab99f5d0206031d436805186f0ae20&quot;: unexpected status: 403 Forbidden<br>上传不了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-29 14:31:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/b3/24/8246675f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>天地有雪</span>
  </div>
  <div class="_2_QraFYR_0">helm version<br>version.BuildInfo{Version:&quot;v3.11.0&quot;, GitCommit:&quot;472c5736ab01133de504a826bd9ee12cbe4e7904&quot;, GitTreeState:&quot;clean&quot;, GoVersion:&quot;go1.18.10&quot;}<br><br>1<br>helm registry login -u mlkkkfriend https:&#47;&#47;ghcr.io<br>Password: <br>Error: Get &quot;https:&#47;&#47;ghcr.io&#47;v2&#47;&quot;: denied: denied<br>输入账号，密码无法登录<br><br>2 文章中提到 在推送之前，还需要使用 GitHub ID 和刚才创建的 Token 登录到 GitHub Package。 需要什么操作<br><br>3 echo $CR_PAT | docker login ghcr.io -u 用户 --password-stdin<br>可以登录<br>docker push 可以上传</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-29 14:26:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/b3/24/8246675f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>天地有雪</span>
  </div>
  <div class="_2_QraFYR_0">老师：请教一下</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-29 14:26:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/b3/24/8246675f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>天地有雪</span>
  </div>
  <div class="_2_QraFYR_0">老师：<br>1 helm registry login -u mlkkkfriend https:&#47;&#47;ghcr.io<br>Password: <br>Error: Get &quot;https:&#47;&#47;ghcr.io&#47;v2&#47;&quot;: denied: denied    </div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-29 14:18:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/b3/24/8246675f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>天地有雪</span>
  </div>
  <div class="_2_QraFYR_0">你好：<br>1 helm registry login -u mlkkkfriend https:&#47;&#47;ghcr.io<br>Password: <br>Error: Get &quot;https:&#47;&#47;ghcr.io&#47;v2&#47;&quot;: denied: denied<br>文章中 在推送之前，还需要使用 GitHub ID 和刚才创建的 Token 登录到 GitHub Package。 需要什么操作</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 需要创建 GitHub Token，检查一下是否赋予了 Token Package 的读写权限。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-29 14:17:39</div>
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
  <div class="_2_QraFYR_0">2、执行安装时使用不同的 helm release name，并且通过命令行参数或 values.yaml 的方式修改 deploy、service 等对象的名字</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍🏻正确！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-25 07:16:17</div>
  </div>
</div>
</div>
</li>
</ul>