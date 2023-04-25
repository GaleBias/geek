<audio title="19｜自托管构建：如何使用 Harbor 搭建企业级镜像仓库？" src="https://static001.geekbang.org/resource/audio/b5/d2/b5cc83c1381558b8736d220c6e2057d2.mp3" controls="controls"></audio> 
<p>你好，我是王炜。</p><p>在上一节课，我带你学习了 Tekton 的基本概念以及如何使用 Tekton 构建镜像。这种方案最大的优势在于可以直接在 Kubernetes 集群中构建，我们不再需要单独为镜像构建付费。</p><p>但是，在之前的构建方案中，我们是将镜像推送到了 Docker Hub 镜像仓库。实际上，Docker Hub 也是一个收费的服务，对于免费用户来说，它限制每 6 小时最多拉取 200 次镜像，显然，这对团队来说是完全不够用的。</p><p>在这节课，我就带你学习如何使用 Harbor 来搭建企业级的镜像仓库，将它集成到我们上一节课创建的 Tekton Pipeline 流程中，最终替换 Docker Hub，进一步降低镜像存储的成本。此外，在安装 Harbor 的过程中，我还会首次介绍 Helm 工具的使用方法。</p><p>在开始今天的学习之前，你需要按照上一节课的内容准备好一个云厂商的 Kubernetes 集群，安装 Ingress-Nginx 和 Tekton，并配置好 Pipeline 和 GitHub Webook。</p><p>此外，在生产环境下，Harbor 一般都会开启 TLS，<strong>所以你还需要准备一个可用的域名。</strong></p><p>下面，我们进入今天的实战环节。</p><h2>安装 Helm</h2><p>在我们之前的实践中，像是安装 Tekton 和 Ingress-Nginx 都是通过 Kubernetes Manifest 来完成的。实际上，安装 Kubernetes 应用并不只有一种方案，这里我们介绍第二种方案。<strong>Helm</strong>。</p><!-- [[[read_end]]] --><p>这节课，我们先学会使用Helm就可以了，更详细的内容我们会在后续的课程做介绍。</p><p>要使用 Helm，首先需要安装它，你可以通过<a href="https://helm.sh/docs/intro/install/">这个链接</a>查看不同平台的安装方法，这里我使用官方提供的脚本来安装。</p><pre><code class="language-powershell">$ curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
</code></pre><p>安装好 Helm 之后，在正式使用之前，你还要确保本地 Kubectl 和集群的连通性，Helm 和 Kubectl 默认读取的 Kubeconfig 文件路径都是 ~/.kube/config。</p><h2>安装 Cert-manager</h2><p>接下来我们安装 Cert-manager，它会为我们自动签发免费的 Let’s Encrypt HTTPS 证书，并在过期前自动续期。</p><p>首先，运行 helm repo add 命令添加官方 Helm 仓库。</p><pre><code class="language-powershell">$ helm repo add jetstack https://charts.jetstack.io
"jetstack" has been added to your repositories
</code></pre><p>然后，运行 helm repo update 更新本地缓存。</p><pre><code class="language-powershell">$ helm repo update
...Successfully got an update from the "jetstack" chart repository
</code></pre><p>接下来，运行 helm install 来安装 Cert-manager。</p><pre><code class="language-powershell">$ helm install cert-manager jetstack/cert-manager \
--namespace cert-manager \
--create-namespace \
--version v1.10.0 \
--set ingressShim.defaultIssuerName=letsencrypt-prod \
--set ingressShim.defaultIssuerKind=ClusterIssuer \
--set ingressShim.defaultIssuerGroup=cert-manager.io \
--set installCRDs=true

NAME: cert-manager
LAST DEPLOYED: Mon Oct 17 21:26:44 2022
NAMESPACE: cert-manager
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
cert-manager v1.10.0 has been deployed successfully!
</code></pre><p>此外，还需要为 Cert-manager 创建 ClusterIssuer，用来提供签发机构。将下面的内容保存为 cluster-issuer.yaml。</p><pre><code class="language-powershell">apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: "wangwei@gmail.com"
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:    
    - http01:
        ingress:
          class: nginx
</code></pre><p><strong>注意，这里你需要将 spec.acme.email 替换为你真实的邮箱地址。</strong>然后运行 kubectl apply 提交到集群内。</p><pre><code class="language-powershell">$ kubectl apply -f cluster-issuer.yaml
clusterissuer.cert-manager.io/letsencrypt-prod created
</code></pre><p>到这里，Cert-manager 就已经配置好了。</p><h2>安装和配置 Harbor</h2><h3>安装 Harbor</h3><p>现在，我们同样使用 Helm 来安装 Harbor，首先添加 Harbor 官方仓库。</p><pre><code class="language-powershell">$ helm repo add harbor https://helm.goharbor.io
"harbor" has been added to your repositories
</code></pre><p>然后，更新本地 Helm 缓存。</p><pre><code class="language-powershell">$ helm repo update
...Successfully got an update from the "jetstack" chart repository
...Successfully got an update from the "harbor" chart repository
</code></pre><p>接下来，由于我们需要定制化安装 Harbor，所以需要修改 Harbor 的安装参数，将下面的内容保存为 values.yaml。</p><pre><code class="language-yaml">expose:
  type: ingress
  tls:
    enabled: true
    certSource: secret
    secret:
      secretName: "harbor-secret-tls"
      notarySecretName: "notary-secret-tls"
  ingress:
    hosts:
      core: harbor.n7t.dev
      notary: notary.n7t.dev
    className: nginx
    annotations:
      kubernetes.io/tls-acme: "true"
persistence:
  persistentVolumeClaim:
    registry:
      size: 20Gi
    chartmuseum:
      size: 10Gi
    jobservice:
      jobLog:
        size: 10Gi
      scanDataExports:
        size: 10Gi
    database:
      size: 10Gi
    redis:
      size: 10Gi
    trivy:
      size: 10Gi
</code></pre><p>注意，由于腾讯云的 PVC 最小 10G 起售，所以，除了 registry 的容量以外，我将安装参数中其他的持久化卷容量都配置为了 10G，你可以根据安装集群的实际情况做调整。</p><p>另外，我还为 Harbor 配置了 ingress 访问域名，分别是 harbor.n7t.dev 和 notary.n7t.dev，<strong>你需要将它们分别替换成你的真实域名。</strong></p><p>然后，再通过 helm install 命令来安装 Harbor，<strong>并指定参数配置文件 values.yaml</strong>。</p><pre><code class="language-powershell">$ helm install harbor harbor/harbor -f values.yaml --namespace harbor --create-namespace
NAME: harbor
LAST DEPLOYED: Mon Oct 17 21:53:28 2022
NAMESPACE: harbor
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Please wait for several minutes for Harbor deployment to complete.
Then you should be able to visit the Harbor portal at https://core.harbor.domain
For more details, please visit https://github.com/goharbor/harbor
</code></pre><p>等待所有 Pod 处于就绪状态。</p><pre><code class="language-powershell">$ kubectl wait --for=condition=Ready pods --all -n harbor --timeout 600s
pod/cm-acme-http-solver-4v4zz condition met
pod/harbor-chartmuseum-79f49f5b4-8f4mb condition met
pod/harbor-core-6c6bc7fb4f-4822f condition met
pod/harbor-database-0 condition met
pod/harbor-jobservice-85d448d5c9-gn5pk condition met
pod/harbor-notary-server-848bcc7ccd-m5m5v condition met
pod/harbor-notary-signer-6897444589-6vssq condition met
pod/harbor-portal-588b64cbdb-gqlbn condition met
pod/harbor-redis-0 condition met
pod/harbor-registry-5c7d58c87c-6bgsj condition met
pod/harbor-trivy-0 condition met
</code></pre><p>到这里，Harbor 就已经安装完成了。</p><h3>配置 DNS 解析</h3><p>接下来，我们为域名配置 DNS 解析。首先，获取 Ingress-Nginx Loadbalancer 的外网 IP。</p><pre><code class="language-powershell">$ kubectl get services --namespace ingress-nginx ingress-nginx-controller --output jsonpath='{.status.loadBalancer.ingress[0].ip}'
43.135.82.249
</code></pre><p>然后，为域名配置 DNS 解析。在这个例子中，我需要分别为 harbor.n7t.dev 和 notary.n7t.dev 配置 A 记录，并指向 43.135.82.249。</p><h3>访问 Harbor Dashboard</h3><p>在访问 Harbor Dashboard 之前，首先我们要确认 Cert-manager 是否已经成功签发了 HTTPS 证书，你可以通过 kubectl get certificate 命令来确认。</p><pre><code class="language-yaml">$ kubectl get certificate -A&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
NAMESPACE&nbsp; &nbsp;NAME&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; READY&nbsp; &nbsp;SECRET&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; AGE
harbor&nbsp; &nbsp; &nbsp; harbor-secret-tls&nbsp; &nbsp;True&nbsp; &nbsp; harbor-secret-tls&nbsp; &nbsp;8s
harbor&nbsp; &nbsp; &nbsp; notary-secret-tls&nbsp; &nbsp;True&nbsp; &nbsp; notary-secret-tls&nbsp; &nbsp;8s
</code></pre><p>由于我们在部署 Harbor 的时候需要配置两个域名，所以这里会出现两个证书。当这两个证书的 Ready 状态都为 True 时，说明 HTTPS 证书已经签发成功了。此外，<strong>Cert-manager 自动从 Ingress 对象中读取了 tls 配置</strong>，还自动创建了名为 harbor-secret-tls 和 notary-secret-tls 两个包含证书信息的 Secret。</p><p>接下来，打开 <a href="https://harbor.n7t.dev">https://harbor.n7t.dev</a> 进入 Harbor Dashboard，使用默认账号 admin 和 Harbor12345 即可登录控制台。</p><p><img src="https://static001.geekbang.org/resource/image/36/06/366ca200b019c17960c1c44032a3cf06.png?wh=1920x1041" alt="图片"></p><p>Harbor 已经自动为我们创建了 library 项目，我们在后续的阶段将直接使用它。</p><h3>推送镜像测试</h3><p>现在，让我们来尝试将本地的镜像推送到 Harbor 仓库。首先，在本地拉取 busybox 镜像。</p><pre><code class="language-yaml">$ docker pull busybox
Using default tag: latest
latest: Pulling from library/busybox
f5b7ce95afea: Pull complete
Digest: sha256:9810966b5f712084ea05bf28fc8ba2c8fb110baa2531a10e2da52c1efc504698
Status: Downloaded newer image for busybox:latest
docker.io/library/busybox:latest
</code></pre><p>然后，运行 docker login 命令登录到 Harbor 仓库，使用默认的账号密码。</p><pre><code class="language-yaml">$ docker login harbor.n7t.dev
username: admin
password: Harbor12345
Login Succeeded
</code></pre><p>接下来，重新给 busybox 镜像打标签，指向 Harbor 镜像仓库。</p><pre><code class="language-yaml">$ docker tag busybox:latest harbor.n7t.dev/library/busybox:latest
</code></pre><p>和推送到 Docker Hub 的 Tag 相比，推送到 Harbor 需要指定完整的<strong>镜像仓库地址、项目名和镜像名</strong>。在这里，我使用了默认的 library 项目，当然你也可以新建一个项目，并将 library 替换为新的项目名。</p><p>最后，将镜像推送到仓库。</p><pre><code class="language-yaml">$ docker push harbor.n7t.dev/library/busybox:latest
The push refers to repository [harbor.n7t.dev/library/busybox]
0b16ab2571f4: Pushed
latest: digest: sha256:7bd0c945d7e4cc2ce5c21d449ba07eb89c8e6c28085edbcf6f5fa4bf90e7eedc size: 527
</code></pre><p>镜像推送成功后，访问 Harbor 控制台，进入 library 项目详情，你将看到我们刚才推送的镜像。</p><p><img src="https://static001.geekbang.org/resource/image/fc/c1/fc9bc8754c476e31937fdf269d4c25c1.png?wh=1920x1041" alt="图片"></p><p>到这里，Harbor 镜像仓库就已经配置好了。</p><h2>在 Tekton Pipeline 中使用 Harbor</h2><p>要在 Tekton Pipeline 中使用 Harbor，我们需要将 Pipeline 中的 spec.params.registry_url 变量值由 docker.io 修改为 harbor.n7t.dev，并且将 spec.params.registry_mirror 变量值修改为 library。你可以使用 kubectl edit 命令来修改。</p><pre><code class="language-yaml">$ kubectl edit Pipeline github-trigger-pipeline
......
&nbsp; params:
&nbsp; - default: harbor.n7t.dev  # 修改为 harbor.n7t.dev
&nbsp; &nbsp; name: registry_url
&nbsp; &nbsp; type: string
  - default: "library"  # 修改为 library
&nbsp; &nbsp; name: registry_mirror
&nbsp; &nbsp; type: string

pipeline.tekton.dev/github-trigger-pipeline edited
</code></pre><p>（提示：按下 i 进入编辑模式，修改完成后，按下 ESC 退出编辑模式，然后输入 :wq 保存生效。）</p><p>然后，再修改镜像仓库的凭据，也就是 registry-auth Secret。</p><pre><code class="language-yaml">$ kubectl edit secret registry-auth
apiVersion: v1
data:
&nbsp; password: SGFyYm9yMTIzNDUK   # 修改为 Base64 编码：Harbor12345
&nbsp; username: YWRtaW4K           # 修改为 Base64 编码：admin
kind: Secret

secret/registry-auth edited
</code></pre><p>保存后生效。</p><p>现在，回到本地示例应用 kubernetes-example 目录，向仓库推送一个空的 commit 来触发 Tekton 流水线。</p><pre><code class="language-yaml">$ git commit --allow-empty -m "Trigger Build"
[main e42ac45] Trigger Build

$ git push origin main
</code></pre><p>进入 Tekton Dashboard <a href="http://tekton.k8s.local/">http://tekton.k8s.local/</a>，并查看流水线运行状态。</p><p><img src="https://static001.geekbang.org/resource/image/08/66/087705da1bce016491b814faa355ef66.png?wh=1920x1041" alt="图片"></p><p>当流水线运行结束后，我们进入 Harbor Dashboard 会看到刚才 Tekton 推送的新镜像。</p><p><img src="https://static001.geekbang.org/resource/image/d4/cc/d494cc826214ec0bcd4e069a575a37cc.png?wh=1920x1041" alt="图片"></p><p>到这里，Harbor 的安装和配置就完成了。在最开始，我们是通过最小化的配置安装 Harbor，在生产环境下你可能需要注意一些额外的配置。</p><h2>Harbor 生产建议</h2><p>Harbor 的安装配置相对较多，这里我也提供几点生产建议供你参考。</p><ol>
<li>确认 PVC 是否支持在线扩容。</li>
<li>尽量使用 S3 作为镜像存储系统。</li>
<li>使用外部数据库和 Redis。</li>
<li>开启自动镜像扫描和阻止漏洞镜像。</li>
</ol><p>接下来我们对每一项进行详细的介绍。</p><h3>确认 PVC 是否支持在线扩容</h3><p>如果你是按照这节课的安装方式使用 PVC 持久卷来存储镜像的，那么随着镜像数量的增加，你需要额外注意 Harbor 仓库存储容量的问题。</p><p>一个简单的方案是在 Harbor 安装前，提前确认 StorageClass 是否支持在线扩容，以便后续对存储镜像的持久卷进行动态扩容。你可以使用 kubectl get storageclass 命令来确认。</p><pre><code class="language-yaml">$ kubectl get storageclass
NAME&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; PROVISIONER&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;RECLAIMPOLICY&nbsp; &nbsp;VOLUMEBINDINGMODE&nbsp; &nbsp;ALLOWVOLUMEEXPANSION&nbsp; &nbsp;AGE
cbs (default)&nbsp; &nbsp;com.tencent.cloud.csi.cbs&nbsp; &nbsp;Delete&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; Immediate&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;true
</code></pre><p>在返回内容中，如果 ALLOWVOLUMEEXPANSION 为 true，就说明支持在线扩容。否则，你需要手动为 StorageClass 添加 AllowVolumeExpansion 字段。</p><pre><code class="language-yaml">$ kubectl patch storageclass cbs -p '{"allowVolumeExpansion": true}'
</code></pre><h3>推荐使用 S3 存储镜像</h3><p>除了使用持久卷来存储镜像以外，Harbor 还支持外部存储。如果你希望大规模使用 Harbor 又不想关注存储问题，那么使用外部存储是一个非常的选择。例如使用 AWS S3 存储桶来存储镜像。</p><p>S3 存储方案的优势是，它能为我们提供接近无限存储容量的存储系统，并且按量计费的方式成本也相对可控，同时它还具备高可用性和容灾能力。</p><p>要使用 S3 来存储镜像，你需要在安装时修改 Harbor 的安装配置 values.yaml。</p><pre><code class="language-yaml">expose:
  type: ingress
  tls:
    enabled: true
    certSource: secret
    secret:
      secretName: "harbor-secret-tls"
      notarySecretName: "notary-secret-tls"
  ingress:
    hosts:
      core: harbor.n7t.dev
      notary: notary.n7t.dev
    className: nginx
    annotations:
      kubernetes.io/tls-acme: "true"
persistence:
  imageChartStorage:
    type: s3
    s3:
      region: us-west-1
      bucket: bucketname
      accesskey: AWS_ACCESS_KEY_ID
      secretkey: AWS_SECRET_ACCESS_KEY
      rootdirectory: /harbor
  persistentVolumeClaim:
    chartmuseum:
      size: 10Gi
    jobservice:
      jobLog:
        size: 10Gi
      scanDataExports:
        size: 10Gi
     ......
</code></pre><p>注意，要将 S3 相关配置 region、bucket、accesskey、secretkey 和 rootdirectory 字段修改为实际的值。</p><p>然后，再使用 helm install -f values.yaml 来安装。</p><h3>使用外部数据库和 Redis</h3><p>在安装 Harbor 时，会默认自动安装数据库和 Redis。但是为了保证稳定性和高可用，我建议你使用云厂商提供的 Postgres 和 Redis 托管服务。</p><p>要使用外部数据库和 Redis，你同样可以在 values.yaml 文件中直接指定。</p><pre><code class="language-yaml">expose:
  type: ingress
  tls:
    enabled: true
    certSource: secret
    secret:
      secretName: "harbor-secret-tls"
      notarySecretName: "notary-secret-tls"
  ingress:
    hosts:
      core: harbor.n7t.dev
      notary: notary.n7t.dev
    className: nginx
    annotations:
      kubernetes.io/tls-acme: "true"
database:
  type: external
  external:
	host: "192.168.0.1"
	port: "5432"
	username: "user"
    password: "password"
	coreDatabase: "registry"
	notaryServerDatabase: "notary_server"
	notarySignerDatabase: "notary_signer"
redis:
  type: external
  external:
	addr: "192.168.0.2:6379"
    password: ""
persistence:
  ......
</code></pre><p>注意，要将 database 和 redis 字段的连接信息修改为实际的内容，并提前创建好 registry、notary_server 和 notary_signer 数据库。</p><h3>开启自动镜像扫描和阻止漏洞镜像</h3><p>最后一个建议。由于Harbor 自带 Trivy 镜像扫描功能，可以帮助我们发现镜像的漏洞，并提供修复建议。因此在生产环境下，我推荐你开启“自动镜像扫描”和“阻止潜在漏洞镜像”功能，你可以进入项目的“配置管理”菜单开启它们。</p><p><img src="https://static001.geekbang.org/resource/image/ac/10/acde313e0579f068321064576ac5fb10.png?wh=1920x1108" alt="图片"></p><p>开启“自动扫描镜像”功能后，所有推送到 Harbor 的镜像都会自动执行扫描，你可以进入镜像详情查看镜像漏洞数量。</p><p><img src="https://static001.geekbang.org/resource/image/3b/db/3bba757da66a4e0d2929d66a4b3116db.png?wh=1920x441" alt="图片"></p><p>进入“Artifacts”详情可以查看漏洞详情和修复建议。</p><p><img src="https://static001.geekbang.org/resource/image/8b/96/8b3d0ac5a486edd9be0d70646160c796.png?wh=1920x1041" alt="图片"></p><p>开启“阻止潜在漏洞镜像”功能之后，接下来我们尝试在本地拉取 frontend 镜像，会发现 Harbor 阻止了这个行为。</p><pre><code class="language-powershell">$ docker pull harbor.n7t.dev/library/frontend:8d64515
Error response from daemon: unknown: current image with 14 vulnerabilities cannot be pulled due to configured policy in 'Prevent images with vulnerability severity of "Low" or higher from running.' To continue with pull, please contact your project administrator to exempt matched vulnerabilities through configuring the CVE allowlist.
</code></pre><h2>总结</h2><p>在这节课，我向你介绍了如何使用 Harbor 搭建企业级镜像仓库，在安装 Harbor 的过程中，我还简单介绍了 Helm 工具的使用方法，包括添加 Helm 仓库和安装应用。</p><p>在生产环境下，我们一般会使用 HTTPS 协议来加密访问请求，所以在安装 Harbor 之前，我向你介绍了 Cert-manager 组件，它可以帮助我们自动签发 Let’s Encrypt HTTPS 证书，并按照 Ingress 的配置生成 Secret。需要注意的是，Let’s Encrypt 证书的有效期是 90 天，不过 Cert-manager 在到期前会自动帮助我们续期，使用起来非常方便。</p><p>其次，我还介绍了如何在 Tekton Pipeline 中使用 Harbor，这里的重点是要修改 Pipeline 的仓库地址以及在 Secret 中配置的镜像仓库用户名和密码，以便 Tekton 能顺利地将镜像推送到 Harbor 仓库中。这里还有一个小细节，当我们将镜像推送到 Docker Hub 时，只需要在镜像 Tag 前面加上用户名前缀，例如 lyzhang1999/frontend:latest 即可。但当我们需要将镜像推送到其他镜像仓库时，则需要把 Tag 配置为完整的地址，例如harbor.n7t.dev/library/frontend:latest 才能够推送。</p><p>最后，我还给你提了 4 个 Harbor 的生产建议，你可以结合项目的实际情况来选择性地采用。</p><p>到这里，自动化镜像构建这个章节就全部结束了。在接下来的课程中，我会为你介绍 Manifest 以外其他两种更高级的应用定义格式：Kustomize 和 Helm Chart。显然，这节课提到的 Cert-manager 和 Harbor 就是以 Helm Chart 的方式来定义的。同时，我也会将示例应用以这两种方式进行封装改造，带你深入了解 Kubernetes 的应用定义。</p><h2>思考题</h2><p>最后，给你留一道思考题吧。</p><p>请你尝试改造<a href="https://time.geekbang.org/column/article/622743">第 16 讲</a>中 GitHub Action Workflow，实现将镜像推送到 Harbor 中。</p><p>提示：为 docker/login-action@v2 插件增加 registry 参数，并将 docker/build-push-action@v3 插件的 Tag 字段修改为包含完整 Harbor 仓库的 URL。</p><p>欢迎你给我留言交流讨论，你也可以把这节课分享给更多的朋友一起阅读。我们下节课见。</p>
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
  <div class="_2_QraFYR_0">记录一次部署遇到的问题：<br>安装 Cert-manager指定了自己的email<br>我www.namesilo.com申请了geekcloudnative.com域名，分别为二级域名 `harbor.geekcloudnative.com` 和 `notary.geekcloudnative.com` 配置 A 记录，并指向自己的ip <br>但我的kubectl get certificate -A  的READY一直是False，<br>NAMESPACE   NAME                READY   SECRET              AGE<br>harbor      harbor-secret-tls   False   harbor-secret-tls   29s<br>harbor      notary-secret-tls   False   notary-secret-tls   29s<br><br>kubectl get certificate -n harbor harbor-secret-tls -oyaml显示 message: Issuing certificate as Secret does not exist，<br><br>k  logs cert-manager-5d595f9fdf-mnw9n -n cert-manager 发现 以下报错日志<br>error&quot;=&quot;failed to perform self check GET request &#39;http:&#47;&#47;harbor.geekcloudnative.com&#47;.well-known&#47;acme-challenge&#47;ldnFkzhB2euqIZdtrTo7-ryM492_HCrmpcnWOBH6TmI&#39;: Get \&quot;http:&#47;&#47;harbor.geekcloudnative.com&#47;.well-known&#47;acme-challenge&#47;ldnFkzhB2euqIZdtrTo7-ryM492_HCrmpcnWOBH6TmI\&quot;: dial tcp: lookup harbor.geekcloudnative.com on 172.16.253.166:53: no such host&quot; <br><br>通过报错可以判断本地没有做域名解析。在&#47;etc&#47;hosts添加<br>&lt;ip&gt; harbor.geekcloudnative.com notary.geekcloudnative.com<br>READY为false得到解决。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍🏻很好的经验分享~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-01 18:54:45</div>
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
  <div class="_2_QraFYR_0">Harbor搭建过程中使用了大量的云厂商组件，而且还有数倍的运维工作。我不明白，为什么不直接用云厂商的镜像库。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，对小团队来说直接用云厂商的镜像仓库更好，全托管。<br>有一些团队可能会考虑存储成本和镜像扫描和安全性，Harbor 在这块做的挺不错的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-30 22:28:18</div>
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
  <div class="_2_QraFYR_0">老师，不需要将证书保存在docker目录下，创建跟域名相同的子目录这个操作吗？我看精选留言第三条的朋友也遇到了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 有具体的问题描述吗？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-24 08:33:46</div>
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
  <div class="_2_QraFYR_0">界面https可以登陆。但命令行登陆：docker login harbor.geekcloudnative.com -u admin -p Harbor12345<br>WARNING! Using --password via the CLI is insecure. Use --password-stdin.<br>Error response from daemon: Get &quot;https:&#47;&#47;harbor.geekcloudnative.com&#47;v2&#47;&quot;: Get &quot;https:&#47;&#47;core.harbor.domain&#47;service&#47;token?account=admin&amp;client_id=docker&amp;offline_token=true&amp;service=harbor-registry&quot;: dial tcp: lookup core.harbor.domain on 8.8.8.8:53: no such host</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-01 22:45:01</div>
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
  <div class="_2_QraFYR_0">kubectl get certificate 我本地报 Issuing certificate as Secret does not exist</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 检查一下安装 Cert-manager 的步骤以及创建 ClusterIssuer 的步骤。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-29 10:29:33</div>
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
  <div class="_2_QraFYR_0">在阿里云的k8s上使用cert-manager需要accesskey来验证域名归属和dns记录验证，文章里没看到 是域名的关系吗，后续跟课程也验证下</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 阿里云没试过，有经验可以分享吗</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-27 09:52:44</div>
  </div>
</div>
</div>
</li>
</ul>