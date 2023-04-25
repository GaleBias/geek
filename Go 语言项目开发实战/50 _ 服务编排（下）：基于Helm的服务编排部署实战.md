<audio title="50 _ 服务编排（下）：基于Helm的服务编排部署实战" src="https://static001.geekbang.org/resource/audio/b8/59/b85c076e1d050ed2ac9e818df5f25c59.mp3" controls="controls"></audio> 
<p>你好，我是孔令飞。</p><p>上一讲，我介绍了 Helm 的基础知识，并带着你部署了一个简单的应用。掌握Helm的基础知识之后，今天我们就来实战下，一起通过Helm部署一个IAM应用。</p><p>通过Helm部署IAM应用，首先需要制作IAM Chart包，然后通过Chart包来一键部署IAM应用。在实际开发中，我们需要将应用部署在不同的环境中，所以我也会给你演示下如何在多环境中部署IAM应用。</p><h2>制作IAM Chart包</h2><p>在部署IAM应用之前，我们首先需要制作一个IAM Chart包。</p><p>我们假设IAM项目源码根目录为<code>${IAM_ROOT}</code>，进入 <code>${IAM_ROOT}/deployments</code>目录，在该目录下创建Chart包。具体创建流程分为四个步骤，下面我来详细介绍下。</p><p><strong>第一步，</strong>创建一个模板Chart。</p><p>Chart是一个组织在文件目录中的集合，目录名称就是Chart名称（没有版本信息）。你可以看看这个 <a href="https://helm.sh/zh/docs/topics/charts">Chart 开发指南</a> ，它介绍了如何开发你自己的Chart。</p><p>不过，这里你也可以使用 <code>helm create</code> 命令来快速创建一个模板Chart，并基于该Chart进行修改，得到你自己的Chart。创建命令如下：</p><!-- [[[read_end]]] --><pre><code class="language-bash">$ helm create iam
</code></pre><p><code>helm create iam</code>会在当前目录下生成一个<code>iam</code>目录，里面存放的就是Chart文件。Chart目录结构及文件如下：</p><pre><code class="language-bash">$ tree -FC iam/
├── charts/                            # [可选]: 该目录中放置当前Chart依赖的其他Chart
├── Chart.yaml                         # YAML文件，用于描述Chart的基本信息，包括名称版本等
├── templates/                         # [可选]: 部署文件模版目录，模版使用的值来自values.yaml和由Tiller提供的值
│&nbsp;&nbsp; ├── deployment.yaml                # Kubernetes Deployment object
│&nbsp;&nbsp; ├── _helpers.tpl                   # 用于修改Kubernetes objcet配置的模板
│&nbsp;&nbsp; ├── hpa.yaml                       # Kubernetes HPA object
│&nbsp;&nbsp; ├── ingress.yaml                   # Kubernetes Ingress object
│&nbsp;&nbsp; ├── NOTES.txt                      # [可选]: 放置Chart的使用指南
│&nbsp;&nbsp; ├── serviceaccount.yaml
│&nbsp;&nbsp; ├── service.yaml
│&nbsp;&nbsp; └── tests/                         # 定义了一些测试资源
│&nbsp;&nbsp;     └── test-connection.yaml
└── values.yaml                        # Chart的默认配置文件
</code></pre><p>上面的目录中，有两个比较重要的文件：</p><ul>
<li>Chart.yaml 文件</li>
<li>templates目录</li>
</ul><p>下面我来详细介绍下这两个文件。<strong>我们先来看Chart.yaml 文件。</strong></p><p>Chart.yaml用来描述Chart的基本信息，包括名称、版本等，内容如下：</p><pre><code class="language-yaml">apiVersion: Chart API 版本 （必需）
name: Chart名称 （必需）
version: 语义化版本（必需）
kubeVersion: 兼容Kubernetes版本的语义化版本（可选）
description: 对这个项目的一句话描述（可选）
type: Chart类型 （可选）
keywords:
  - 关于项目的一组关键字（可选）
home: 项目home页面的URL （可选）
sources:
  - 项目源码的URL列表（可选）
dependencies: # chart 必要条件列表 （可选）
  - name: Chart名称 (nginx)
    version: Chart版本 ("1.2.3")
    repository: （可选）仓库URL ("https://example.com/charts") 或别名 ("@repo-name")
    condition: （可选） 解析为布尔值的YAML路径，用于启用/禁用Chart(e.g. subchart1.enabled )
    tags: # （可选）
      - 用于一次启用/禁用 一组Chart的tag
    import-values: # （可选）
      - ImportValue 保存源值到导入父键的映射。每项可以是字符串或者一对子/父列表项
    alias: （可选） Chart中使用的别名。当你要多次添加相同的Chart时会很有用
maintainers: # （可选）
  - name: 维护者名字 （每个维护者都需要）
    email: 维护者邮箱 （每个维护者可选）
    url: 维护者URL （每个维护者可选）
icon: 用作icon的SVG或PNG图片URL （可选）
appVersion: 包含的应用版本（可选）。不需要是语义化，建议使用引号
deprecated: 不被推荐的Chart（可选，布尔值）
annotations:
  example: 按名称输入的批注列表 （可选）.
</code></pre><p><strong>我们再来看下<strong><strong>templates目录</strong></strong>这个文件。</strong></p><p>templates目录中包含了应用中各个Kubernetes资源的YAML格式资源定义模板，例如：</p><pre><code class="language-yaml">apiVersion: v1
kind: Service
metadata:
  labels:
    app: {{ .Values.pump.name }}
  name: {{ .Values.pump.name }}
spec:
  ports:
  - name: http
    protocol: TCP
    {{- toYaml .Values.pump.service.http| nindent 4 }}
  selector:
    app: {{ .Values.pump.name }}
  sessionAffinity: None
  type: {{ .Values.serviceType }}
</code></pre><p><code>{{ .Values.pump.name }}</code>会被<code>deployments/iam/values.yaml</code>文件中<code>pump.name</code>的值替换。上面的模版语法扩展了 Go <code>text/template</code>包的语法：</p><pre><code class="language-yaml"># 这种方式定义的模版，会去除test模版尾部所有的空行
{{- define "test"}}
模版内容
{{- end}}

# 去除test模版头部的第一个空行
{{- template "test" }}
</code></pre><p>下面是用于YAML文件前置空格的语法：</p><pre><code class="language-bash"># 这种方式定义的模版，会去除test模版头部和尾部所有的空行
{{- define "test" -}}
模版内容
{{- end -}}

# 可以在test模版每一行的头部增加4个空格，用于YAML文件的对齐
{{ include "test" | indent 4}}
</code></pre><p>最后，这里有三点需要你注意：</p><ul>
<li>Chart名称必须是小写字母和数字，单词之间可以使用横杠<code>-</code>分隔，Chart名称中不能用大写字母，也不能用下划线，<code>.</code>号也不行。</li>
<li>尽可能使用<a href="https://semver.org/">SemVer 2</a>来表示版本号。</li>
<li>YAML 文件应该按照双空格的形式缩进(一定不要使用tab键)。</li>
</ul><p><strong>第二步，</strong>编辑 <code>iam</code> 目录下的Chart文件。</p><p>我们可以基于<code>helm create</code>生成的模板Chart来构建自己的Chart包。这里我们添加了创建iam-apiserver、iam-authz-server、iam-pump、iamctl服务需要的YAML格式的Kubernetes资源文件模板：</p><pre><code class="language-bash">$ ls -1 iam/templates/*.yaml
iam/templates/hpa.yaml                                   # Kubernetes HPA模板文件
iam/templates/iam-apiserver-deployment.yaml              # iam-apiserver服务deployment模板文件
iam/templates/iam-apiserver-service.yaml                 # iam-apiserver服务service模板文件
iam/templates/iam-authz-server-deployment.yaml           # iam-authz-server服务deployment模板文件
iam/templates/iam-authz-server-service.yaml              # iam-authz-server服务service模板文件
iam/templates/iamctl-deployment.yaml                     # iamctl服务deployment模板文件
iam/templates/iam-pump-deployment.yaml                   # iam-pump服务deployment模板文件
iam/templates/iam-pump-service.yaml                      # iam-pump服务service模板文件
</code></pre><p>模板的具体内容，你可以查看<a href="https://github.com/marmotedu/iam/tree/v1.1.0/deployments/iam/templates">deployments/iam/templates/</a>。</p><p>在编辑 Chart 时，我们可以通过 <code>helm lint</code> 验证格式是否正确，例如：</p><pre><code class="language-bash">$ helm lint iam
==&gt; Linting iam

1 chart(s) linted, 0 chart(s) failed
</code></pre><p><code>0 chart(s) failed</code> 说明当前Iam Chart包是通过校验的。</p><p><strong>第三步，</strong>修改Chart的配置文件，添加自定义配置。</p><p>我们可以编辑<code>deployments/iam/values.yaml</code>文件，定制自己的配置。具体配置你可以参考<a href="https://github.com/marmotedu/iam/blob/v1.1.0/deployments/iam/values.yaml">deployments/iam/values.yaml</a>。</p><p>在修改 <code>values.yaml</code> 文件时，你可以参考下面这些最佳实践。</p><ul>
<li>变量名称以小写字母开头，单词按驼峰区分，例如<code>chickenNoodleSoup</code>。</li>
<li>给所有字符串类型的值加上引号。</li>
<li>为了避免整数转换问题，将整型存储为字符串更好，并用 <code>{{ int $value }}</code> 在模板中将字符串转回整型。</li>
<li><code>values.yaml</code>中定义的每个属性都应该文档化。文档字符串应该以它要描述的属性开头，并至少给出一句描述。例如：</li>
</ul><pre><code class="language-yaml"># serverHost is the host name for the webserver
serverHost: example
# serverPort is the HTTP listener port for the webserver
serverPort: 9191
</code></pre><p>这里需要注意，所有的Helm内置变量都以大写字母开头，以便与用户定义的value进行区分，例如<code>.Release.Name</code>、<code>.Capabilities.KubeVersion</code>。</p><p>为了安全，values.yaml中只配置Kubernetes资源相关的配置项，例如Deployment副本数、Service端口等。至于iam-apiserver、iam-authz-server、iam-pump、iamctl组件的配置文件，我们创建单独的ConfigMap，并在Deployment中引用。</p><p><strong>第四步，</strong>打包Chart，并上传到Chart仓库中。</p><p>这是一个可选步骤，可以根据你的实际需要来选择。如果想了解具体操作，你可以查看 <a href="https://helm.sh/zh/docs/topics/chart_repository">Helm chart 仓库</a>获取更多信息。</p><p>最后，IAM应用的Chart包见<a href="https://github.com/marmotedu/iam/tree/v1.1.0/deployments/iam">deployments/iam</a>。</p><h2>IAM Chart部署</h2><p>上面，我们制作了IAM应用的Chart包，接下来我们就使用这个Chart包来一键创建IAM应用。IAM Chart部署一共分为10个步骤，你可以跟着我一步步操作下。</p><p><strong>第一步，</strong>配置<code>scripts/install/environment.sh</code>。</p><p><code>scripts/install/environment.sh</code>文件中包含了各类自定义配置，你主要配置下面这些跟数据库相关的就可以，其他配置使用默认值。</p><ul>
<li>MariaDB配置：environment.sh文件中以<code>MARIADB_</code>开头的变量。</li>
<li>Redis配置：environment.sh文件中以<code>REDIS_</code>开头的变量。</li>
<li>MongoDB配置：environment.sh文件中以<code>MONGO_</code>开头的变量。</li>
</ul><p><strong>第二步，</strong>创建IAM应用的配置文件。</p><pre><code class="language-bash">$ cd ${IAM_ROOT}
$ make gen.defaultconfigs # 生成iam-apiserver、iam-authz-server、iam-pump、iamctl组件的默认配置文件
$ make gen.ca # 生成 CA 证书
</code></pre><p>上面的命令会将IAM的配置文件存放在目录<code>${IAM_ROOT}/_output/configs/</code>下。</p><p><strong>第三步，</strong>创建 <code>iam</code> 命名空间。</p><p>我们将IAM应用涉及到的各类资源都创建在<code>iam</code>命名空间中。将IAM资源创建在独立的命名空间中，不仅方便维护，还可以有效避免影响其他Kubernetes资源。</p><pre><code class="language-bash">$ kubectl create namespace iam
</code></pre><p><strong>第四步，</strong>将IAM各服务的配置文件，以ConfigMap资源的形式保存在Kubernetes集群中。</p><pre><code class="language-bash">$ kubectl -n iam create configmap iam --from-file=${IAM_ROOT}/_output/configs/
$ kubectl -n iam get configmap iam
NAME   DATA   AGE
iam    4      13s
</code></pre><p><strong>第五步，</strong>将IAM各服务使用的证书文件，以ConfigMap资源的形式保存在Kubernetes集群中。</p><pre><code class="language-bash">$ kubectl -n iam create configmap iam-cert --from-file=${IAM_ROOT}/_output/cert
$ kubectl -n iam get configmap iam-cert
NAME       DATA   AGE
iam-cert   14     12s
</code></pre><p><strong>第六步，</strong>创建镜像仓库访问密钥。</p><p>在准备阶段，我们开通了<a href="http://ccr.ccs.tencentyun.com">腾讯云镜像仓库服务</a>，并创建了用户<code>10000099``xxxx</code>，密码为<code>iam59!z$</code>。</p><p>接下来，我们就可以创建docker-registry secret了。Kubernetes在下载Docker镜像时，需要docker-registry secret来进行认证。创建命令如下：</p><pre><code class="language-bash">$ kubectl -n iam create secret docker-registry ccr-registry --docker-server=ccr.ccs.tencentyun.com --docker-username=10000099xxxx --docker-password='iam59!z$'
</code></pre><p><strong>第七步，</strong>创建Docker镜像，并Push到镜像仓库。</p><pre><code class="language-bash">$ make push REGISTRY_PREFIX=ccr.ccs.tencentyun.com/marmotedu VERSION=v1.1.0
</code></pre><p><strong>第八步，</strong>安装IAM Chart包。</p><p>在<a href="https://time.geekbang.org/column/article/420940">49讲</a>里，我介绍了4种安装Chart包的方法。这里，我们通过未打包的IAM Chart路径来安装，安装方法如下：</p><pre><code class="language-bash">$ cd ${IAM_ROOT}
$ helm -n iam install iam deployments/iam
NAME: iam
LAST DEPLOYED: Sat Aug 21 17:46:56 2021
NAMESPACE: iam
STATUS: deployed
REVISION: 1
TEST SUITE: None
</code></pre><p>执行 <code>helm install</code> 后，Kubernetes会自动部署应用，等到IAM应用的Pod都处在 <code>Running</code> 状态时，说明IAM应用已经成功安装：</p><pre><code class="language-bash">$ kubectl -n iam get pods|grep iam
iam-apiserver-cb4ff955-hs827&nbsp; &nbsp; &nbsp; &nbsp; 1/1&nbsp; &nbsp; &nbsp;Running&nbsp; &nbsp;0&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; 66s
iam-authz-server-7fccc7db8d-chwnn&nbsp; &nbsp;1/1&nbsp; &nbsp; &nbsp;Running&nbsp; &nbsp;0&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; 66s
iam-pump-78b57b4464-rrlbf&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;1/1&nbsp; &nbsp; &nbsp;Running&nbsp; &nbsp;0&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; 66s
iamctl-59fdc4995-xrzhn&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; 1/1&nbsp; &nbsp; &nbsp;Running&nbsp; &nbsp;0&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; 66s
</code></pre><p><strong>第九步，</strong>测试IAM应用。</p><p>我们通过<code>helm install</code>在<code>iam</code>命令空间下创建了一个测试Deployment <code>iamctl</code>。你可以登陆<code>iamctl</code> Deployment所创建出来的Pod，执行一些运维操作和冒烟测试。登陆命令如下：</p><pre><code class="language-bash">$ kubectl -n iam exec -it `kubectl -n iam get pods -l app=iamctl | awk '/iamctl/{print $1}'` -- bash
</code></pre><p>登陆到<code>iamctl-xxxxxxxxxx-xxxxx</code> Pod中后，你就可以执行运维操作和冒烟测试了。</p><p><strong>先来看运维操作。</strong>iamctl工具以子命令的方式对外提供功能，你可以使用它提供的各类功能，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/69/y2/693f608aa571cbfd6e06c8cfdb242yy2.png?wh=1920x337" alt="图片"></p><p><strong>再来看冒烟测试：</strong></p><pre><code class="language-bash"># cd /opt/iam/scripts/install
# ./test.sh iam::test::smoke
</code></pre><p>如果<code>./test.sh iam::test::smoke</code>命令打印的输出中，最后一行为<code>congratulations, smoke test passed!</code>字符串，就说明IAM应用安装成功。如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/9d/8c/9dcc557952b3586f7b37b065bf2bd58c.png?wh=1920x314" alt="图片"></p><p><strong>第十步，</strong>销毁EKS集群的资源。</p><pre><code class="language-bash">$ kubectl delete namespace iam
</code></pre><p>你可以根据需要选择是否删除EKS集群，如果不需要了就可以选择删除。</p><h2>IAM应用多环境部署</h2><p>在实际的项目开发中，我们需要将IAM应用部署到不同的环境中，不同环境的配置文件是不同的，那么IAM项目是如何进行多环境部署的呢？</p><p>IAM项目在<a href="">configs</a>目录下创建了多个Helm values文件（格式为<code>values-{envName}-env.yaml</code>）：</p><ul>
<li>values-test-env.yaml，测试环境Helm values文件。</li>
<li>values-pre-env.yaml，预发环境Helm values文件。</li>
<li>values-prod-env.yaml，生产环境Helm values文件。</li>
</ul><p>在部署IAM应用时，我们在命令行指定<code>-f</code>参数，例如：</p><pre><code class="language-bash">$ helm -n iam install -f configs/values-test-env.yaml iam deployments/iam # 安装到测试环境。
</code></pre><h2>总结</h2><p>这一讲，我们通过 <code>helm create iam</code> 创建了一个模板Chart，并基于这个模板Chart包进行了二次开发，最终创建了IAM应用的Helm Chart包：<a href="https://github.com/marmotedu/iam/tree/v1.1.0/deployments/iam">deployments/iam</a>。</p><p>有了Helm Chart包，我们就可以通过 <code>helm -n iam install iam deployments/iam</code> 命令来一键部署好整个IAM应用。当IAM应用中的所有Pod都处在 <code>Running</code> 状态后，说明IAM应用被成功部署。</p><p>最后，我们可以登录iamctl容器，执行 <code>test.sh iam::test::smoke</code> 命令，来对IAM应用进行冒烟测试。</p><h2>课后练习</h2><ol>
<li>试着在Helm Chart中加入MariaDB、MongoDB、Redis模板，通过Helm一键部署好整个IAM应用。</li>
<li>试着通过 <code>helm</code> 命令升级、回滚和删除IAM应用。</li>
</ol><p>欢迎你在留言区与我交流讨论，我们下一讲见。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/64/05/6989dce6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我来也</span>
  </div>
  <div class="_2_QraFYR_0">“给所有字符串类型的值加上引号。”<br>深有体会，很多开源的chart也可能存在这种问题。比如pvc的名称，没有用quote加引号，用户如果非得来一个全数字的pvc就悲剧了。helm会默认转换为数字类型。<br><br>我一般调试时使用helm upgrade --install —debug —dry-run。如果可以看到渲染后的yaml，调试起来还是蛮方便的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 好办法！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-25 10:15:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/87/64/3882d90d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>yandongxiao</span>
  </div>
  <div class="_2_QraFYR_0">总结：<br>制作chart的流程：<br>1. 使用 helm create 命令创建一个 chart；<br>2. chart 的目录结构：Chart.yaml, values.yaml, templates, charts 等。<br>3. templates 目录中包含资源的定义文件，使用了 go template 语法，有点凌乱。<br>4. 建议 values.yaml 文件中个，给所有字符串类型的值加上引号；使用字符串来表示整型，通过 {{ int $values }} 方式来引用。<br>5. 使用 helm lint 或者 helm install --dry-run 方式，验证helm package的格式，但内容上不一定符合你预期。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 6666</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-05 12:43:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/30/5b/82e3952c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Wongkakui</span>
  </div>
  <div class="_2_QraFYR_0">有个疑问，一般集群内的资源都是运维管理的，但我们项目使用到的资源可以通过helm values维护在自己代码仓库，运维缺改不了，这种情况有最佳实践吗</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-05 22:03:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>GeekCoder</span>
  </div>
  <div class="_2_QraFYR_0">一个应用有多个服务，其中一个服务改动之后，得重新打一个chart包？全部重新部署？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: helm chart是幂等部署的，只要每一个服务的镜像tag不变，Kubernetes就不会去部署</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-25 17:13:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/05/92/b609f7e3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>骨汤鸡蛋面</span>
  </div>
  <div class="_2_QraFYR_0">对于一些yaml配置，如果pro 使用，test 不需要，该如何配置呢。比如node 亲和性配置，可能线上要配，测试环境就无所谓了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以在yaml模板中使用if语句来判断</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-08 10:21:29</div>
  </div>
</div>
</div>
</li>
</ul>