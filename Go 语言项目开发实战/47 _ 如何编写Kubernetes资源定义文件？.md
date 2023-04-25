<audio title="47 _ 如何编写Kubernetes资源定义文件？" src="https://static001.geekbang.org/resource/audio/86/b5/86f78c4e67de7f2476cc8443c6427ab5.mp3" controls="controls"></audio> 
<p>你好，我是孔令飞。</p><p>在接下来的48讲，我会介绍如何基于腾讯云EKS来部署IAM应用。EKS其实是一个标准的Kubernetes集群，在Kubernetes集群中部署应用，需要编写Kubernetes资源的YAML（Yet Another Markup Language）定义文件，例如Service、Deployment、ConfigMap、Secret、StatefulSet等。</p><p>这些YAML定义文件里面有很多配置项需要我们去配置，其中一些也比较难理解。为了你在学习下一讲时更轻松，这一讲我们先学习下如何编写Kubernetes YAML文件。</p><h2>为什么选择YAML格式来定义Kubernetes资源？</h2><p>首先解释一下，我们为什么使用YAML格式来定义Kubernetes的各类资源呢？这是因为YAML格式和其他格式（例如XML、JSON等）相比，不仅能够支持丰富的数据，而且结构清晰、层次分明、表达性极强、易于维护，非常适合拿来供开发者配置和管理Kubernetes资源。</p><p>其实Kubernetes支持YAML和JSON两种格式，JSON格式通常用来作为接口之间消息传递的数据格式，YAML格式则用于资源的配置和管理。YAML和JSON这两种格式是可以相互转换的，你可以通过在线工具<a href="https://www.json2yaml.com/convert-yaml-to-json">json2yaml</a>，来自动转换YAML和JSON数据格式。</p><!-- [[[read_end]]] --><p>例如，下面是一个YAML文件中的内容：</p><pre><code class="language-yaml">apiVersion: v1
kind: Service
metadata:
  name: iam-apiserver
spec:
  clusterIP: 192.168.0.231
  externalTrafficPolicy: Cluster
  ports:
  - name: https
    nodePort: 30443
    port: 8443
    protocol: TCP
    targetPort: 8443
  selector:
    app: iam-apiserver
  sessionAffinity: None
  type: NodePort
</code></pre><p>它对应的JSON格式的文件内容为：</p><pre><code class="language-json">{
  "apiVersion": "v1",
  "kind": "Service",
  "metadata": {
    "name": "iam-apiserver"
  },
  "spec": {
    "clusterIP": "192.168.0.231",
    "externalTrafficPolicy": "Cluster",
    "ports": [
      {
        "name": "https",
        "nodePort": 30443,
        "port": 8443,
        "protocol": "TCP",
        "targetPort": 8443
      }
    ],
    "selector": {
      "app": "iam-apiserver"
    },
    "sessionAffinity": "None",
    "type": "NodePort"
  }
}
</code></pre><p>我就是通过<code>json2yaml</code>在线工具，来转换YAML和JSON的，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/0f/02/0ffac271b296d1cc407941cfc3139702.png?wh=1920x780" alt="图片"></p><p>在编写Kubernetes资源定义文件的过程中，如果因为YAML格式文件中的配置项缩进太深，导致不容易判断配置项的层级，那么，你就可以将其转换成JSON格式，通过JSON格式来判断配置型的层级。</p><p>如果想学习更多关于YAML的知识，你可以参考<a href="https://yaml.org/spec/1.2/spec.html">YAML 1.2 (3rd Edition)</a>。这里，可以先看看我整理的YAML基本语法：</p><ul>
<li>属性和值都是大小写敏感的。</li>
<li>使用缩进表示层级关系。</li>
<li>禁止使用Tab键缩进，只允许使用空格，建议两个空格作为一个层级的缩进。元素左对齐，就说明对齐的两个元素属于同一个级别。</li>
<li>使用 <code>#</code> 进行注释，直到行尾。</li>
<li><code>key: value</code>格式的定义中，冒号后要有一个空格。</li>
<li>短横线表示列表项，使用一个短横线加一个空格；多个项使用同样的缩进级别作为同一列表。</li>
<li>使用 <code>---</code> 表示一个新的YAML文件开始。</li>
</ul><p>现在你知道了，Kubernetes支持YAML和JSON两种格式，它们是可以相互转换的。但鉴于YAML格式的各项优点，我建议你使用YAML格式来定义Kubernetes的各类资源。</p><h2>Kubernetes 资源定义概述</h2><p>Kubernetes中有很多内置的资源，常用的资源有Deployment、StatefulSet、ConfigMap、Service、Secret、Nodes、Pods、Events、Jobs、DaemonSets等。除此之外，Kubernetes还有其他一些资源。如果你觉得Kubernetes内置的资源满足不了需求，还可以自定义资源。</p><p>Kubernetes的资源清单可以通过执行以下命令来查看：</p><pre><code class="language-bash">$ kubectl api-resources
NAME                              SHORTNAMES   APIVERSION                        NAMESPACED   KIND
bindings                                       v1                                true         Binding
componentstatuses                 cs           v1                                false        ComponentStatus
configmaps                        cm           v1                                true         ConfigMap
endpoints                         ep           v1                                true         Endpoints
events                            ev           v1                                true         Event
</code></pre><p>上述输出中，各列的含义如下。</p><ul>
<li>NAME：资源名称。</li>
<li>SHORTNAMES：资源名称简写。</li>
<li>APIVERSION：资源的API版本，也称为group。</li>
<li>NAMESPACED：资源是否具有Namespace属性。</li>
<li>KIND：资源类别。</li>
</ul><p>这些资源有一些共同的配置，也有一些特有的配置。这里，我们先来看下这些资源共同的配置。</p><p>下面这些配置是Kubernetes各类资源都具备的：</p><pre><code class="language-yaml">---
apiVersion: &lt;string&gt; # string类型，指定group的名称，默认为core。可以使用 `kubectl api-versions` 命令，来获取当前kubernetes版本支持的所有group。
kind: &lt;string&gt; # string类型，资源类别。
metadata: &lt;Object&gt; # 资源的元数据。
  name: &lt;string&gt; # string类型，资源名称。
  namespace: &lt;string&gt; # string类型，资源所属的命名空间。
  lables: &lt; map[string]string&gt; # map类型，资源的标签。
  annotations: &lt; map[string]string&gt; # map类型，资源的标注。
  selfLink: &lt;string&gt; # 资源的 REST API路径，格式为：/api/&lt;group&gt;/namespaces/&lt;namespace&gt;/&lt;type&gt;/&lt;name&gt;。例如：/api/v1/namespaces/default/services/iam-apiserver
spec: &lt;Object&gt; # 定义用户期望的资源状态（disired state）。
status: &lt;Object&gt; # 资源当前的状态，以只读的方式显示资源的最近状态。这个字段由kubernetes维护，用户无法定义。
</code></pre><p>你可以通过<code>kubectl explain &lt;object&gt;</code>命令来查看Object资源对象介绍，并通过<code>kubectl explain &lt;object1&gt;.&lt;object2&gt;</code>来查看<code>&lt;object1&gt;</code>的子对象<code>&lt;object2&gt;</code>的资源介绍，例如：</p><pre><code class="language-bash">$ kubectl explain service
$ kubectl explain service.spec
$ kubectl explain service.spec.ports
</code></pre><p>Kubernetes资源定义YAML文件，支持以下数据类型：</p><ul>
<li>string，表示字符串类型。</li>
<li>object，表示一个对象，需要嵌套多层字段。</li>
<li>map[string]string，表示由key:value组成的映射。</li>
<li>[]string，表示字串列表。</li>
<li>[]object，表示对象列表。</li>
<li>boolean，表示布尔类型。</li>
<li>integer，表示整型。</li>
</ul><h2>常用的Kubernetes资源定义</h2><p>上面说了，Kubernetes中有很多资源，其中Pod、Deployment、Service、ConfigMap这4类是比较常用的资源，我来一个个介绍下。</p><h3>Pod资源定义</h3><p>下面是一个Pod的YAML定义：</p><pre><code class="language-yaml">apiVersion: v1   # 必须 版本号， 常用v1  apps/v1
kind: Pod	 # 必须
metadata:  # 必须，元数据
  name: string  # 必须，名称
  namespace: string # 必须，命名空间，默认上default,生产环境为了安全性建议新建命名空间分类存放
  labels:   # 非必须，标签，列表值
    - name: string
  annotations:  # 非必须，注解，列表值
    - name: string
spec:  # 必须，容器的详细定义
  containers:  #必须，容器列表，
    - name: string　　　#必须，容器1的名称
      image: string		#必须，容器1所用的镜像
      imagePullPolicy: [Always|Never|IfNotPresent]  #非必须，镜像拉取策略，默认是Always
      command: [string]  # 非必须 列表值，如果不指定，则是一镜像打包时使用的启动命令
      args:　[string] # 非必须，启动参数
      workingDir: string # 非必须，容器内的工作目录
      volumeMounts: # 非必须，挂载到容器内的存储卷配置
        - name: string  # 非必须，存储卷名字，需与【@1】处定义的名字一致
          readOnly: boolean #非必须，定义读写模式，默认是读写
      ports: # 非必须，需要暴露的端口
        - name: string  # 非必须 端口名称
          containerPort: int  # 非必须 端口号
          hostPort: int # 非必须 宿主机需要监听的端口号，设置此值时，同一台宿主机不能存在同一端口号的pod， 建议不要设置此值
          proctocol: [tcp|udp]  # 非必须 端口使用的协议，默认是tcp
      env: # 非必须 环境变量
        - name: string # 非必须 ，环境变量名称
          value: string  # 非必须，环境变量键值对
      resources:  # 非必须，资源限制
        limits:  # 非必须，限制的容器使用资源的最大值，超过此值容器会推出
          cpu: string # 非必须，cpu资源，单位是core，从0.1开始
          memory: string 内存限制，单位为MiB,GiB
        requests:  # 非必须，启动时分配的资源
          cpu: string 
          memory: string
      livenessProbe:   # 非必须，容器健康检查的探针探测方式
        exec: # 探测命令
          command: [string] # 探测命令或者脚本
        httpGet: # httpGet方式
          path: string  # 探测路径，例如 http://ip:port/path
          port: number  
          host: string  
          scheme: string
          httpHeaders:
            - name: string
              value: string
          tcpSocket:  # tcpSocket方式，检查端口是否存在
            port: number
          initialDelaySeconds: 0 #容器启动完成多少秒后的再进行首次探测，单位为s
          timeoutSeconds: 0  #探测响应超时的时间,默认是1s,如果失败，则认为容器不健康，会重启该容器
          periodSeconds: 0  # 探测间隔时间，默认是10s
          successThreshold: 0  # 
          failureThreshold: 0
        securityContext:
          privileged: false
        restartPolicy: [Always|Never|OnFailure]  # 容器重启的策略，
        nodeSelector: object  # 指定运行的宿主机
        imagePullSecrets:  # 容器下载时使用的Secrets名称，需要与valumes.secret中定义的一致
          - name: string
        hostNetwork: false
        volumes: ## 挂载的共享存储卷类型
          - name: string  # 非必须，【@1】
          emptyDir: {}
          hostPath:
            path: string
          secret:  # 类型为secret的存储卷，使用内部的secret内的items值作为环境变量
            secrectName: string
            items:
              - key: string
                path: string
            configMap:  ## 类型为configMap的存储卷
              name: string
              items:
                - key: string
                  path: string
</code></pre><p>Pod是Kubernetes中最重要的资源，我们可以通过Pod YAML定义来创建一个Pod，也可以通过DaemonSet、Deployment、ReplicaSet、StatefulSet、Job、CronJob来创建Pod。</p><h3>Deployment资源定义</h3><p>Deployment资源定义YAML文件如下：</p><pre><code class="language-yaml">apiVersion: apps/v1
kind: Deployment
metadata:
  labels: # 设定资源的标签
    app: iam-apiserver
  name: iam-apiserver
  namespace: default
spec:
  progressDeadlineSeconds: 10 # 指定多少时间内不能完成滚动升级就视为失败，滚动升级自动取消
  replicas: 1 # 声明副本数，建议 &gt;= 2
  revisionHistoryLimit: 5 # 设置保留的历史版本个数，默认是10
  selector: # 选择器
    matchLabels: # 匹配标签
      app: iam-apiserver # 标签格式为key: value对
  strategy: # 指定部署策略
    rollingUpdate:
      maxSurge: 1 # 最大额外可以存在的副本数，可以为百分比，也可以为整数
      maxUnavailable: 1 # 表示在更新过程中能够进入不可用状态的 Pod 的最大值，可以为百分比，也可以为整数
    type: RollingUpdate # 更新策略，包括：重建(Recreate)、RollingUpdate(滚动更新)
  template: # 指定Pod创建模板。注意：以下定义为Pod的资源定义
    metadata: # 指定Pod的元数据
      labels: # 指定Pod的标签
        app: iam-apiserver
    spec:
      affinity:
        podAntiAffinity: # Pod反亲和性，尽量避免同一个应用调度到相同Node
          preferredDuringSchedulingIgnoredDuringExecution: # 软需求
          - podAffinityTerm:
              labelSelector:
                matchExpressions: # 有多个选项，只有同时满足这些条件的节点才能运行 Pod
                - key: app
                  operator: In # 设定标签键与一组值的关系，In、NotIn、Exists、DoesNotExist
                  values:
                  - iam-apiserver
              topologyKey: kubernetes.io/hostname
            weight: 100 # weight 字段值的范围是1-100。
      containers:
      - command: # 指定运行命令
        - /opt/iam/bin/iam-apiserver # 运行参数
        - --config=/etc/iam/iam-apiserver.yaml
        image: ccr.ccs.tencentyun.com/lkccc/iam-apiserver-amd64:v1.0.6 # 镜像名，遵守镜像命名规范
        imagePullPolicy: Always # 镜像拉取策略。IfNotPresent：优先使用本地镜像；Never：使用本地镜像，本地镜像不存在，则报错；Always：默认值，每次都重新拉取镜像
        # lifecycle: # kubernetes支持postStart和preStop事件。当一个容器启动后，Kubernetes将立即发送postStart事件；在容器被终结之前，Kubernetes将发送一个preStop事件
        name: iam-apiserver # 容器名称，与应用名称保持一致
        ports: # 端口设置
        - containerPort: 8443 # 容器暴露的端口
          name: secure # 端口名称
          protocol: TCP # 协议，TCP和UDP
        livenessProbe: # 存活检查，检查容器是否正常，不正常则重启实例
          httpGet: # HTTP请求检查方法
            path: /healthz # 请求路径
            port: 8080 # 检查端口
            scheme: HTTP # 检查协议
          initialDelaySeconds: 5 # 启动延时，容器延时启动健康检查的时间
          periodSeconds: 10 # 间隔时间，进行健康检查的时间间隔
          successThreshold: 1 # 健康阈值，表示后端容器从失败到成功的连续健康检查成功次数
          failureThreshold: 1 # 不健康阈值，表示后端容器从成功到失败的连续健康检查成功次数
          timeoutSeconds: 3 # 响应超时，每次健康检查响应的最大超时时间
        readinessProbe: # 就绪检查，检查容器是否就绪，不就绪则停止转发流量到当前实例
          httpGet: # HTTP请求检查方法
            path: /healthz # 请求路径
            port: 8080 # 检查端口
            scheme: HTTP # 检查协议
          initialDelaySeconds: 5 # 启动延时，容器延时启动健康检查的时间
          periodSeconds: 10 # 间隔时间，进行健康检查的时间间隔
          successThreshold: 1 # 健康阈值，表示后端容器从失败到成功的连续健康检查成功次数
          failureThreshold: 1 # 不健康阈值，表示后端容器从成功到失败的连续健康检查成功次数
          timeoutSeconds: 3 # 响应超时，每次健康检查响应的最大超时时间
        startupProbe: # 启动探针，可以知道应用程序容器什么时候启动了
          failureThreshold: 10
          httpGet:
            path: /healthz
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 5
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 3
        resources: # 资源管理
          limits: # limits用于设置容器使用资源的最大上限,避免异常情况下节点资源消耗过多
            cpu: "1" # 设置cpu limit，1核心 = 1000m
            memory: 1Gi # 设置memory limit，1G = 1024Mi
          requests: # requests用于预分配资源,当集群中的节点没有request所要求的资源数量时,容器会创建失败
            cpu: 250m # 设置cpu request
            memory: 500Mi # 设置memory request
        terminationMessagePath: /dev/termination-log # 容器终止时消息保存路径
        terminationMessagePolicy: File # 仅从终止消息文件中检索终止消息
        volumeMounts: # 挂载日志卷
        - mountPath: /etc/iam/iam-apiserver.yaml # 容器内挂载镜像路径
          name: iam # 引用的卷名称
          subPath: iam-apiserver.yaml # 指定所引用的卷内的子路径，而不是其根路径。
        - mountPath: /etc/iam/cert
          name: iam-cert
      dnsPolicy: ClusterFirst
      restartPolicy: Always # 重启策略，Always、OnFailure、Never
      schedulerName: default-scheduler # 指定调度器的名字
      imagePullSecrets: # 在Pod中设置ImagePullSecrets只有提供自己密钥的Pod才能访问私有仓库
        - name: ccr-registry # 镜像仓库的Secrets需要在集群中手动创建
      securityContext: {} # 指定安全上下文
      terminationGracePeriodSeconds: 5 # 优雅关闭时间，这个时间内优雅关闭未结束，k8s 强制 kill
      volumes: # 配置数据卷，类型详见https://kubernetes.io/zh/docs/concepts/storage/volumes
      - configMap: # configMap 类型的数据卷
          defaultMode: 420 #权限设置0~0777，默认0664
          items:
          - key: iam-apiserver.yaml
            path: iam-apiserver.yaml
          name: iam # configmap名称
        name: iam # 设置卷名称，与volumeMounts名称对应
      - configMap:
          defaultMode: 420
          name: iam-cert
        name: iam-cert
</code></pre><p>在部署时，你可以根据需要来配置相应的字段，常见的需要配置的字段为：<code>labels</code>、<code>name</code>、<code>namespace</code>、<code>replicas</code>、<code>command</code>、<code>imagePullPolicy</code>、<code>container.name</code>、<code>livenessProbe</code>、<code>readinessProbe</code>、<code>resources</code>、<code>volumeMounts</code>、<code>volumes</code>、<code>imagePullSecrets</code>等。</p><p>另外，在部署应用时，经常需要提供配置文件，供容器内的进程加载使用。最常用的方法是挂载ConfigMap到应用容器中。那么，如何挂载ConfigMap到容器中呢？</p><p>引用 ConfigMap 对象时，你可以在 volume 中通过它的名称来引用。你可以自定义 ConfigMap 中特定条目所要使用的路径。下面的配置就显示了如何将名为 <code>log-config</code> 的 ConfigMap 挂载到名为 <code>configmap-pod</code> 的 Pod 中：</p><pre><code class="language-yaml">apiVersion: v1
kind: Pod
metadata:
  name: configmap-pod
spec:
  containers:
    - name: test
      image: busybox
      volumeMounts:
        - name: config-vol
          mountPath: /etc/config
  volumes:
    - name: config-vol
      configMap:
        name: log-config
        items:
          - key: log_level
            path: log_level
</code></pre><p><code>log-config</code> ConfigMap 以卷的形式挂载，并且存储在 <code>log_level</code> 条目中的所有内容都被挂载到 Pod 的<code>/etc/config/log_level</code> 路径下。 请注意，这个路径来源于卷的 <code>mountPath</code> 和 <code>log_level</code> 键对应的<code>path</code>。</p><p>这里需要注意，在使用 ConfigMap 之前，你首先要创建它。接下来，我们来看下ConfigMap定义。</p><h3>ConfigMap资源定义</h3><p>下面是一个ConfigMap YAML示例：</p><pre><code class="language-yaml">apiVersion: v1
kind: ConfigMap
metadata:
  name: test-config4
data: # 存储配置内容
  db.host: 172.168.10.1 # 存储格式为key: value
  db.port: 3306
</code></pre><p>可以看到，ConfigMap的YAML定义相对简单些。假设我们将上述YAML文件保存在了<code>iam-configmap.yaml</code>文件中，我们可以执行以下命令，来创建ConfigMap：</p><pre><code class="language-bash">$ kubectl create -f iam-configmap.yaml
</code></pre><p>除此之外，kubectl命令行工具还提供了3种创建ConfigMap的方式。我来分别介绍下。</p><p>1）通过<code>--from-literal</code>参数创建</p><p>创建命令如下：</p><pre><code class="language-bash">$ kubectl create configmap iam-configmap --from-literal=db.host=172.168.10.1 --from-literal=db.port='3306'
</code></pre><p>2）通过<code>--from-file=&lt;文件&gt;</code>参数创建</p><p>创建命令如下：</p><pre><code class="language-bash">$ echo -n 172.168.10.1 &gt; ./db.host
$ echo -n 3306 &gt; ./db.port
$ kubectl create cm iam-configmap --from-file=./db.host --from-file=./db.port
</code></pre><p><code>--from-file</code>的值也可以是一个目录。当值是目录时，目录中的文件名为key，目录的内容为value。</p><p>3）通过<code>--from-env-file</code>参数创建</p><p>创建命令如下：</p><pre><code class="language-bash">$ cat &lt;&lt; EOF &gt; env.txt
db.host=172.168.10.1
db.port=3306
EOF
$ kubectl create cm iam-configmap --from-env-file=env.txt
</code></pre><h3>Service资源定义</h3><p>Service 是 Kubernetes 另一个核心资源。通过创建 Service，可以为一组具有相同功能的容器应用提供一个统一的入口地址，并且将请求负载到后端的各个容器上。Service资源定义YAML文件如下：</p><pre><code class="language-yaml">apiVersion: v1
kind: Service
metadata:
  labels:
    app: iam-apiserver
  name: iam-apiserver
  namespace: default
spec:
  clusterIP: 192.168.0.231 # 虚拟服务地址
  externalTrafficPolicy: Cluster # 表示此服务是否希望将外部流量路由到节点本地或集群范围的端点
  ports: # service需要暴露的端口列表
  - name: https #端口名称
    nodePort: 30443 # 当type = NodePort时，指定映射到物理机的端口号
    port: 8443 # 服务监听的端口号
    protocol: TCP # 端口协议，支持TCP和UDP，默认TCP
    targetPort: 8443 # 需要转发到后端Pod的端口号
  selector: # label selector配置，将选择具有label标签的Pod作为其后端RS
    app: iam-apiserver
  sessionAffinity: None # 是否支持session
  type: NodePort # service的类型，指定service的访问方式，默认为clusterIp
</code></pre><p>上面，我介绍了常用的Kubernetes YAML的内容。我们在部署应用的时候，是需要手动编写这些文件的。接下来，我就讲解一些在编写过程中常用的编写技巧。</p><h2>YAML文件编写技巧</h2><p>这里我主要介绍三个技巧。</p><p>1）使用在线的工具来自动生成模板YAML文件。</p><p>YAML文件很复杂，完全从0开始编写一个YAML定义文件，工作量大、容易出错，也没必要。我比较推荐的方式是，使用一些工具来自动生成所需的YAML。</p><p>这里我推荐使用<a href="https://k8syaml.com/">k8syaml</a>工具。<code>k8syaml</code>是一个在线的YAML生成工具，当前能够生成Deployment、StatefulSet、DaemonSet类型的YAML文件。<code>k8syaml</code>具有默认值，并且有对各字段详细的说明，可以供我们填参时参考。</p><p>2）使用<code>kubectl run</code>命令获取YAML模板：</p><pre><code class="language-yaml">$ kubectl run --dry-run=client --image=nginx nginx -o yaml &gt; my-nginx.yaml
$ cat my-nginx.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
</code></pre><p>然后，我们可以基于这个模板，来修改配置，形成最终的YAML文件。</p><p>3）导出集群中已有的资源描述。</p><p>有时候，如果我们想创建一个Kubernetes资源，并且发现该资源跟集群中已经创建的资源描述相近或者一致的时候，可以选择导出集群中已经创建资源的YAML描述，并基于导出的YAML文件进行修改，获得所需的YAML。例如：</p><pre><code class="language-bash">$ kubectl get deployment iam-apiserver -o yaml &gt; iam-authz-server.yaml
</code></pre><p>接着，修改<code>iam-authz-server.yaml</code>。通常，我们需要删除Kubernetes自动添加的字段，例如<code>kubectl.kubernetes.io/last-applied-configuration</code>、<code>deployment.kubernetes.io/revision</code>、<code>creationTimestamp</code>、<code>generation</code>、<code>resourceVersion</code>、<code>selfLink</code>、<code>uid</code>、<code>status</code>。</p><p>这些技巧可以帮助我们更好地编写和使用Kubernetes YAML。</p><h2>使用Kubernetes YAML时的一些推荐工具</h2><p>接下来，我再介绍一些比较流行的工具，你可以根据自己的需要进行选择。</p><h3>kubeval</h3><p><a href="https://github.com/instrumenta/kubeval">kubeval</a>可以用来验证Kubernetes YAML是否符合Kubernetes API模式。</p><p>安装方法如下：</p><pre><code class="language-bash">$ wget https://github.com/instrumenta/kubeval/releases/latest/download/kubeval-linux-amd64.tar.gz
$ tar xf kubeval-linux-amd64.tar.gz
$ mv kubeval $HOME/bin
</code></pre><p>安装完成后，我们对Kubernetes YAML文件进行验证：</p><pre><code class="language-bash">$ kubeval deployments/iam.invalid.yaml
ERR  - iam/templates/iam-configmap.yaml: Duplicate 'ConfigMap' resource 'iam' in namespace ''
</code></pre><p>根据提示，查看<code>iam.yaml</code>，发现在<code>iam.yaml</code>文件中，我们定义了两个同名的<code>iam</code> ConfigMap：</p><pre><code class="language-yaml">apiVersion: v1
kind: ConfigMap
metadata:
  name: iam
data:
  {}
---
# Source: iam/templates/iam-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: iam
data:
  iam-: ""
  iam-apiserver.yaml: |
    ...
</code></pre><p>可以看到，使用<code>kubeval</code>之类的工具，能让我们在部署的早期，不用访问集群就能发现YAML文件的错误。</p><h3>kube-score</h3><p><a href="https://github.com/zegl/kube-score">kube-score</a>能够对Kubernetes YAML进行分析，并根据内置的检查对其评分，这些检查是根据安全建议和最佳实践而选择的，例如：</p><ul>
<li>以非Root用户启动容器。</li>
<li>为Pods设置健康检查。</li>
<li>定义资源请求和限制。</li>
</ul><p>你可以按照这个方法安装：</p><pre><code class="language-bash">$ go get github.com/zegl/kube-score/cmd/kube-score
</code></pre><p>然后，我们对Kubernetes YAML进行评分：</p><pre><code class="language-bash">$ kube-score score -o ci deployments/iam.invalid.yaml
[OK] iam-apiserver apps/v1/Deployment
[OK] iam-apiserver apps/v1/Deployment
[OK] iam-apiserver apps/v1/Deployment
[OK] iam-apiserver apps/v1/Deployment
[CRITICAL] iam-apiserver apps/v1/Deployment: The pod does not have a matching NetworkPolicy
[CRITICAL] iam-apiserver apps/v1/Deployment: Container has the same readiness and liveness probe
[CRITICAL] iam-apiserver apps/v1/Deployment: (iam-apiserver) The pod has a container with a writable root filesystem
[CRITICAL] iam-apiserver apps/v1/Deployment: (iam-apiserver) The container is running with a low user ID
[CRITICAL] iam-apiserver apps/v1/Deployment: (iam-apiserver) The container running with a low group ID
[OK] iam-apiserver apps/v1/Deployment
...
</code></pre><p>检查的结果有<code>OK</code>、<code>SKIPPED</code>、<code>WARNING</code>和<code>CRITICAL</code>。<code>CRITICAL</code>是需要你修复的；<code>WARNING</code>是需要你关注的；<code>SKIPPED</code>是因为某些原因略过的检查；<code>OK</code>是验证通过的。</p><p>如果你想查看详细的错误原因和解决方案，可以使用<code>-o human</code>选项，例如：</p><pre><code class="language-bash">$ kube-score score -o human deployments/iam.invalid.yaml
</code></pre><p>上述命令会检查YAML资源定义文件，如果有不合规的地方会报告级别、类别以及错误详情，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/04/f6/0498529693c6d15c9d9d45cbyy866cf6.png?wh=1920x827" alt="图片"></p><h3></h3><p>当然，除了kubeval、kube-score这两个工具，业界还有其他一些Kubernetes检查工具，例如<a href="https://github.com/stelligent/config-lint">config-lint</a>、<a href="https://github.com/cloud66-oss/copper">copper</a>、<a href="https://github.com/open-policy-agent/conftest">conftest</a>、<a href="https://github.com/FairwindsOps/polaris">polaris</a>等。</p><p>这些工具，我推荐你这么来选择：首先，使用kubeval工具做最基本的YAML文件验证。验证通过之后，我们就可以进行更多的测试。如果你没有特别复杂的YAML验证要求，只需要用到一些最常见的检查策略，这时候可以使用kube-score。如果你有复杂的验证要求，并且希望能够自定义验证策略，则可以考虑使用copper。当然，<code>polaris</code>、<code>config-lint</code>、<code>copper</code>也值得你去尝试下。</p><h2>总结</h2><p>今天，我主要讲了如何编写Kubernetes YAML文件。</p><p>YAML格式具有丰富的数据表达能力、清晰的结构和层次，因此被用于Kubernetes资源的定义文件中。如果你要把应用部署在Kubernetes集群中，就要创建多个关联的K8s资源，如果要创建K8s资源，目前比较多的方式还是编写YAML格式的定义文件。</p><p>这一讲我介绍了K8s中最常用的四种资源（Pod、Deployment、Service、ConfigMap）的YAML定义的写法，你可以常来温习。</p><p>另外，在编写YAML文件时，也有一些技巧。比如，可以通过在线工具<a href="https://k8syaml.com/">k8syaml</a>来自动生成初版的YAML文件，再基于此YAML文件进行二次修改，从而形成终版。</p><p>最后，我还给你分享了编写和使用Kubernetes YAML时，社区提供的多种工具。比如，kubeval可以校验YAML，kube-score可以给YAML文件打分。了解了如何编写Kubernetes YAML文件，下一讲的学习相信你会进行得更顺利。</p><h2>课后练习</h2><ol>
<li>思考一下，如何将ConfigMap中的Key挂载到同一个目录中，文件名为Key名？</li>
<li>使用kubeval检查你正在或之前从事过的项目的K8s YAML定义文件，查看报错，并修改和优化。</li>
</ol><p>欢迎你在留言区和我交流讨论，我们下一讲见。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/28/83/17/df99b53d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>随风而过</span>
  </div>
  <div class="_2_QraFYR_0">覆盖掉挂载的整个目录，使用volumeMount.subPath来声明我们只是挂载单个文件，而不是整个目录，只需要在subPath后面加上我们挂载的单个文件名即可</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 满分！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-14 17:15:34</div>
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
  <div class="_2_QraFYR_0">推荐的工具很实用👍</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-08 09:28:03</div>
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
  <div class="_2_QraFYR_0">总结：<br>YAML规范：属性和值都是大小写敏感的；使用两个空格代表一层缩进；<br>k8syaml: 以交互式的方式，动态生成 Deployment、DaemonSet、StatefulSet 对象；<br>校验 Kubernetes YAML 的工具：kubeeval 验证k8syaml文件的正确性；kubescore 验证 k8syaml 文件的安全性；如果希望自定义验证策略，可以考虑使用 copper。<br>kube-neat 工具 可以将 kubectl xxx -oyaml 导出来的 yaml的 status 部分和部分meta 部分过滤掉；<br>kubectx 和 kubens 快速切换 k8s 环境</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 666</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-05 11:16:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/64/05/6989dce6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我来也</span>
  </div>
  <div class="_2_QraFYR_0">json2yaml 和 yaml2json 过于常用，我是集成到vim的快捷方式中了。<br><br>老师的这个中文注释太详细了，适合新手。😄<br><br>后面这几个工具学习了。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-10 22:30:09</div>
  </div>
</div>
</div>
</li>
</ul>