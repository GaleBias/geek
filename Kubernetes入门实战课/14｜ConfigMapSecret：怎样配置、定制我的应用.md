<audio title="14｜ConfigMapSecret：怎样配置、定制我的应用" src="https://static001.geekbang.org/resource/audio/8d/8a/8d985ac77fcae2a1de910917e32f768a.mp3" controls="controls"></audio> 
<p>你好，我是Chrono。</p><p>前两节课里我们学习了Kubernetes里的三种API对象：Pod、Job和CronJob，虽然还没有讲到更高级的其他对象，但使用它们也可以在集群里编排运行一些实际的业务了。</p><p>不过想让业务更顺利地运行，有一个问题不容忽视，那就是应用的配置管理。</p><p>配置文件，你应该有所了解吧，通常来说应用程序都会有一个，它把运行时需要的一些参数从代码中分离出来，让我们在实际运行的时候能更方便地调整优化，比如说Nginx有nginx.conf、Redis有redis.conf、MySQL有my.cnf等等。</p><p>我们在“入门篇”里学习容器技术的时候讲过，可以选择两种管理配置文件的方式。第一种是编写Dockerfile，用 <code>COPY</code> 指令把配置文件打包到镜像里；第二种是在运行时使用 <code>docker cp</code> 或者 <code>docker run -v</code>，把本机的文件拷贝进容器。</p><p>但这两种方式都存在缺陷。第一种方法相当于是在镜像里固定了配置文件，不好修改，不灵活，第二种方法则显得有点“笨拙”，不适合在集群中自动化运维管理。</p><p>对于这个问题Kubernetes有它自己的解决方案，你也应该能够猜得到，当然还是使用YAML语言来定义API对象，再组合起来实现动态配置。</p><!-- [[[read_end]]] --><p>今天我就来讲解Kubernetes里专门用来管理配置信息的两种对象：<strong>ConfigMap</strong>和<strong>Secret</strong>，使用它们来灵活地配置、定制我们的应用。</p><h2>ConfigMap/Secret</h2><p>首先你要知道，应用程序有很多类别的配置信息，但从数据安全的角度来看可以分成两类：</p><ul>
<li>一类是明文配置，也就是不保密，可以任意查询修改，比如服务端口、运行参数、文件路径等等。</li>
<li>另一类则是机密配置，由于涉及敏感信息需要保密，不能随便查看，比如密码、密钥、证书等等。</li>
</ul><p>这两类配置信息本质上都是字符串，只是由于安全性的原因，在存放和使用方面有些差异，所以Kubernetes也就定义了两个API对象，<strong>ConfigMap</strong>用来保存明文配置，<strong>Secret</strong>用来保存秘密配置。</p><h3>什么是ConfigMap</h3><p>先来看ConfigMap，我们仍然可以用命令 <code>kubectl create</code> 来创建一个它的YAML样板。注意，它有简写名字“<strong>cm</strong>”，所以命令行里没必要写出它的全称：</p><pre><code class="language-bash">export out="--dry-run=client -o yaml"        # 定义Shell变量
kubectl create cm info $out
</code></pre><p>得到的样板文件大概是这个样子：</p><pre><code class="language-yaml">apiVersion: v1
kind: ConfigMap
metadata:
&nbsp; name: info
</code></pre><p>你可能会有点惊讶，ConfigMap的YAML和之前我们学过的Pod、Job不一样，除了熟悉的“apiVersion”“kind”“metadata”，居然就没有其他的了，最重要的字段“spec”哪里去了？这是因为ConfigMap存储的是配置数据，是静态的字符串，并不是容器，所以它们就不需要用“spec”字段来说明运行时的“规格”。</p><p>既然ConfigMap要存储数据，我们就需要用另一个含义更明确的字段“<strong>data</strong>”。</p><p>要生成带有“data”字段的YAML样板，你需要在 <code>kubectl create</code> 后面多加一个参数 <code>--from-literal</code> ，表示从字面值生成一些数据：</p><pre><code class="language-bash">kubectl create cm info --from-literal=k=v $out
</code></pre><p><strong>注意，因为在ConfigMap里的数据都是Key-Value结构，所以 <code>--from-literal</code> 参数需要使用 <code>k=v</code> 的形式。</strong></p><p>把YAML样板文件修改一下，再多增添一些Key-Value，就得到了一个比较完整的ConfigMap对象：</p><pre><code class="language-yaml">apiVersion: v1
kind: ConfigMap
metadata:
&nbsp; name: info

data:
&nbsp; count: '10'
&nbsp; debug: 'on'
&nbsp; path: '/etc/systemd'
&nbsp; greeting: |
&nbsp; &nbsp; say hello to kubernetes.
</code></pre><p>现在就可以使用 <code>kubectl apply</code> 把这个YAML交给Kubernetes，让它创建ConfigMap对象了：</p><pre><code class="language-bash">kubectl apply&nbsp;-f cm.yml
</code></pre><p>创建成功后，我们还是可以用 <code>kubectl get</code>、<code>kubectl describe</code> 来查看ConfigMap的状态：</p><pre><code class="language-bash">kubectl get cm
kubectl describe cm info
</code></pre><p><img src="https://static001.geekbang.org/resource/image/a6/78/a61239d55a93a5cd9da7148297d22878.png?wh=782x184" alt="图片"></p><p><img src="https://static001.geekbang.org/resource/image/34/48/343c94dacb9f872721597e99b346b148.png?wh=1042x1272" alt="图片"></p><p>你可以看到，现在ConfigMap的Key-Value信息就已经存入了etcd数据库，后续就可以被其他API对象使用。</p><h3>什么是Secret</h3><p>了解了ConfigMap对象，我们再来看Secret对象就会容易很多，它和ConfigMap的结构和用法很类似，不过在Kubernetes里Secret对象又细分出很多类，比如：</p><ul>
<li>访问私有镜像仓库的认证信息</li>
<li>身份识别的凭证信息</li>
<li>HTTPS通信的证书和私钥</li>
<li>一般的机密信息（格式由用户自行解释）</li>
</ul><p>前几种我们现在暂时用不到，所以就只使用最后一种，创建YAML样板的命令是 <code>kubectl create secret generic</code> ，同样，也要使用参数 <code>--from-literal</code> 给出Key-Value值：</p><pre><code class="language-bash">kubectl create secret generic user --from-literal=name=root $out
</code></pre><p>得到的Secret对象大概是这个样子：</p><pre><code class="language-yaml">apiVersion: v1
kind: Secret
metadata:
&nbsp; name: user

data:
&nbsp; name: cm9vdA==
</code></pre><p>Secret对象第一眼的感觉和ConfigMap非常相似，只是“kind”字段由“ConfigMap”变成了“Secret”，后面同样也是“data”字段，里面也是Key-Value的数据。</p><p>不过，既然它的名字是Secret，我们就不能像ConfigMap那样直接保存明文了，需要对数据“做点手脚”。你会发现，这里的“name”值是一串“乱码”，而不是刚才在命令行里写的明文“root”。</p><p>这串“乱码”就是Secret与ConfigMap的不同之处，不让用户直接看到原始数据，起到一定的保密作用。不过它的手法非常简单，只是做了Base64编码，根本算不上真正的加密，所以我们完全可以绕开kubectl，自己用Linux小工具“base64”来对数据编码，然后写入YAML文件，比如：</p><pre><code class="language-bash">echo -n "123456" | base64
MTIzNDU2
</code></pre><p>要注意这条命令里的 <code>echo</code> ，必须要加参数 <code>-n</code> 去掉字符串里隐含的换行符，否则Base64编码出来的字符串就是错误的。</p><p>我们再来重新编辑Secret的YAML，为它添加两个新的数据，方式可以是参数 <code>--from-literal</code> 自动编码，也可以是自己手动编码：</p><pre><code class="language-yaml">apiVersion: v1
kind: Secret
metadata:
&nbsp; name: user

data:
&nbsp; name: cm9vdA==  # root
&nbsp; pwd: MTIzNDU2   # 123456
&nbsp; db: bXlzcWw=    # mysql
</code></pre><p>接下来的创建和查看对象操作和ConfigMap是一样的，使用 <code>kubectl apply</code>、<code>kubectl get</code>、<code>kubectl describe</code>：</p><pre><code class="language-bash">kubectl apply&nbsp; -f secret.yml
kubectl get secret
kubectl describe secret user
</code></pre><p><img src="https://static001.geekbang.org/resource/image/0f/10/0f769ba725d1006c1cb98ed9003d7210.png?wh=1838x250" alt="图片"></p><p><img src="https://static001.geekbang.org/resource/image/59/6c/59ac74796771897e0246a4532789076c.png?wh=1138x782" alt="图片"></p><p>这样一个存储敏感信息的Secret对象也就创建好了，而且因为它是保密的，使用 <code>kubectl describe</code> 不能直接看到内容，只能看到数据的大小，你可以和ConfigMap对比一下。</p><h2>如何使用</h2><p>现在通过编写YAML文件，我们创建了ConfigMap和Secret对象，该怎么在Kubernetes里应用它们呢？</p><p>因为ConfigMap和Secret只是一些存储在etcd里的字符串，所以如果想要在运行时产生效果，就必须要以某种方式“<strong>注入</strong>”到Pod里，让应用去读取。在这方面的处理上Kubernetes和Docker是一样的，也是两种途径：<strong>环境变量</strong>和<strong>加载文件</strong>。</p><p>先看比较简单的环境变量。</p><h3>如何以环境变量的方式使用ConfigMap/Secret</h3><p>在前面讲Pod的时候，说过描述容器的字段“<strong>containers</strong>”里有一个“<strong>env</strong>”，它定义了Pod里容器能够看到的环境变量。</p><p>当时我们只使用了简单的“value”，把环境变量的值写“死”在了YAML里，实际上它还可以使用另一个“<strong>valueFrom</strong>”字段，从ConfigMap或者Secret对象里获取值，这样就实现了把配置信息以环境变量的形式注入进Pod，也就是配置与应用的解耦。</p><p>由于“valueFrom”字段在YAML里的嵌套层次比较深，初次使用最好看一下 <code>kubectl explain</code> 对它的说明：</p><pre><code class="language-plain">kubectl explain pod.spec.containers.env.valueFrom
</code></pre><p>“<strong>valueFrom</strong>”字段指定了环境变量值的来源，可以是“<strong>configMapKeyRef</strong>”或者“<strong>secretKeyRef</strong>”，然后你要再进一步指定应用的ConfigMap/Secret的“<strong>name</strong>”和它里面的“<strong>key</strong>”，要当心的是这个“name”字段是API对象的名字，而不是Key-Value的名字。</p><p>下面我就把引用了ConfigMap和Secret对象的Pod列出来，给你做个示范，为了提醒你注意，我把“<strong>env</strong>”字段提到了前面：</p><pre><code class="language-yaml">apiVersion: v1
kind: Pod
metadata:
&nbsp; name: env-pod

spec:
&nbsp; containers:
&nbsp; - env:
&nbsp; &nbsp; &nbsp; - name: COUNT
&nbsp; &nbsp; &nbsp; &nbsp; valueFrom:
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; configMapKeyRef:
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; name: info
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; key: count
&nbsp; &nbsp; &nbsp; - name: GREETING
&nbsp; &nbsp; &nbsp; &nbsp; valueFrom:
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; configMapKeyRef:
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; name: info
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; key: greeting
&nbsp; &nbsp; &nbsp; - name: USERNAME
&nbsp; &nbsp; &nbsp; &nbsp; valueFrom:
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; secretKeyRef:
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; name: user
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; key: name
&nbsp; &nbsp; &nbsp; - name: PASSWORD
&nbsp; &nbsp; &nbsp; &nbsp; valueFrom:
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; secretKeyRef:
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; name: user
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; key: pwd

&nbsp; &nbsp; image: busybox
&nbsp; &nbsp; name: busy
&nbsp; &nbsp; imagePullPolicy: IfNotPresent
&nbsp; &nbsp; command: ["/bin/sleep", "300"]
</code></pre><p>这个Pod的名字是“env-pod”，镜像是“busybox”，执行命令sleep睡眠300秒，我们可以在这段时间里使用命令 <code>kubectl exec</code> 进入Pod观察环境变量。</p><p>你需要重点关注的是它的“env”字段，里面定义了4个环境变量，<code>COUNT</code>、<code>GREETING</code>、<code>USERNAME</code>、<code>PASSWORD</code>。</p><p>对于明文配置数据， <code>COUNT</code>、<code>GREETING</code> 引用的是ConfigMap对象，所以使用字段“<strong>configMapKeyRef</strong>”，里面的“name”是ConfigMap对象的名字，也就是之前我们创建的“info”，而“key”字段分别是“info”对象里的 <code>count</code> 和 <code>greeting</code>。</p><p>同样的对于机密配置数据， <code>USERNAME</code>、<code>PASSWORD</code> 引用的是Secret对象，要使用字段“<strong>secretKeyRef</strong>”，再用“name”指定Secret对象的名字 <code>user</code>，用“key”字段应用它里面的 <code>name</code> 和 <code>pwd</code> 。</p><p>这段解释确实是有点绕口令的感觉，因为ConfigMap和Secret在Pod里的组合关系不像Job/CronJob那么简单直接，所以我还是用画图来表示它们的引用关系：</p><p><img src="https://static001.geekbang.org/resource/image/06/9d/0663d692b33c1dee5b08e486d271b69d.jpg?wh=1920x1661" alt="图片"></p><p>从这张图你就应该能够比较清楚地看出Pod与ConfigMap、Secret的“松耦合”关系，它们不是直接嵌套包含，而是使用“KeyRef”字段间接引用对象，这样，同一段配置信息就可以在不同的对象之间共享。</p><p>弄清楚了环境变量的注入方式之后，让我们用 <code>kubectl apply</code> 创建Pod，再用 <code>kubectl exec</code> 进入Pod，验证环境变量是否生效：</p><pre><code class="language-bash">kubectl apply -f env-pod.yml
kubectl exec -it env-pod -- sh

echo $COUNT
echo $GREETING
echo $USERNAME $PASSWORD
</code></pre><p><img src="https://static001.geekbang.org/resource/image/6f/bb/6f0f711de995010498b6807709a811bb.png?wh=1202x660" alt="图片"></p><p>这张截图就显示了Pod的运行结果，可以看到在Pod里使用 <code>echo</code> 命令确实输出了我们在两个YAML里定义的配置信息，也就证明Pod对象成功组合了ConfigMap和Secret对象。</p><p>以环境变量的方式使用ConfigMap/Secret还是比较简单的，下面来看第二种加载文件的方式。</p><h3>如何以Volume的方式使用ConfigMap/Secret</h3><p>Kubernetes为Pod定义了一个“<strong>Volume</strong>”的概念，可以翻译成是“存储卷”。如果把Pod理解成是一个虚拟机，那么Volume就相当于是虚拟机里的磁盘。</p><p>我们可以为Pod“挂载（mount）”多个Volume，里面存放供Pod访问的数据，这种方式有点类似 <code>docker run -v</code>，虽然用法复杂了一些，但功能也相应强大一些。</p><p>在Pod里挂载Volume很容易，只需要在“<strong>spec</strong>”里增加一个“<strong>volumes</strong>”字段，然后再定义卷的名字和引用的ConfigMap/Secret就可以了。要注意的是Volume属于Pod，不属于容器，所以它和字段“containers”是同级的，都属于“spec”。</p><p>下面让我们来定义两个Volume，分别引用ConfigMap和Secret，名字是 <code>cm-vol</code> 和 <code>sec-vol</code>：</p><pre><code class="language-yaml">spec:
&nbsp; volumes:
&nbsp; - name: cm-vol
&nbsp; &nbsp; configMap:
&nbsp; &nbsp; &nbsp; name: info
&nbsp; - name: sec-vol
&nbsp; &nbsp; secret:
&nbsp; &nbsp; &nbsp; secretName: user
</code></pre><p>有了Volume的定义之后，就可以在容器里挂载了，这要用到“<strong>volumeMounts</strong>”字段，正如它的字面含义，可以把定义好的Volume挂载到容器里的某个路径下，所以需要在里面用“<strong>mountPath</strong>”“<strong>name</strong>”明确地指定挂载路径和Volume的名字。</p><pre><code class="language-yaml">&nbsp; containers:
&nbsp; - volumeMounts:
&nbsp; &nbsp; - mountPath: /tmp/cm-items
&nbsp; &nbsp; &nbsp; name: cm-vol
&nbsp; &nbsp; - mountPath: /tmp/sec-items
&nbsp; &nbsp; &nbsp; name: sec-vol
</code></pre><p>把“<strong>volumes</strong>”和“<strong>volumeMounts</strong>”字段都写好之后，配置信息就可以加载成文件了。这里我还是画了图来表示它们的引用关系：</p><p><img src="https://static001.geekbang.org/resource/image/9d/yy/9d3258da1f40554ae88212db2b4yybyy.jpg?wh=1920x1630" alt="图片"></p><p>你可以看到，挂载Volume的方式和环境变量又不太相同。环境变量是直接引用了ConfigMap/Secret，而Volume又多加了一个环节，需要先用Volume引用ConfigMap/Secret，然后在容器里挂载Volume，有点“兜圈子”“弯弯绕”。</p><p>这种方式的好处在于：以Volume的概念统一抽象了所有的存储，不仅现在支持ConfigMap/Secret，以后还能够支持临时卷、持久卷、动态卷、快照卷等许多形式的存储，扩展性非常好。</p><p>现在我把Pod的完整YAML描述列出来，然后使用 <code>kubectl apply</code> 创建它：</p><pre><code class="language-yaml">apiVersion: v1
kind: Pod
metadata:
&nbsp; name: vol-pod

spec:
&nbsp; volumes:
&nbsp; - name: cm-vol
&nbsp; &nbsp; configMap:
&nbsp; &nbsp; &nbsp; name: info
&nbsp; - name: sec-vol
&nbsp; &nbsp; secret:
&nbsp; &nbsp; &nbsp; secretName: user

&nbsp; containers:
&nbsp; - volumeMounts:
&nbsp; &nbsp; - mountPath: /tmp/cm-items
&nbsp; &nbsp; &nbsp; name: cm-vol
&nbsp; &nbsp; - mountPath: /tmp/sec-items
&nbsp; &nbsp; &nbsp; name: sec-vol

&nbsp; &nbsp; image: busybox
&nbsp; &nbsp; name: busy
&nbsp; &nbsp; imagePullPolicy: IfNotPresent
&nbsp; &nbsp; command: ["/bin/sleep", "300"]
</code></pre><p>创建之后，我们还是用 <code>kubectl exec</code> 进入Pod，看看配置信息被加载成了什么形式：</p><pre><code class="language-bash">kubectl apply -f vol-pod.yml
kubectl get pod
kubectl exec -it vol-pod -- sh
</code></pre><p><img src="https://static001.geekbang.org/resource/image/9f/67/9fdc3a7bafcfa0fa277b7c7bed891967.png?wh=1192x728" alt="图片"></p><p>你会看到，ConfigMap和Secret都变成了目录的形式，而它们里面的Key-Value变成了一个个的文件，而文件名就是Key。</p><p>因为这种形式上的差异，以Volume的方式来使用ConfigMap/Secret，就和环境变量不太一样。环境变量用法简单，更适合存放简短的字符串，而Volume更适合存放大数据量的配置文件，在Pod里加载成文件后让应用直接读取使用。</p><h2>小结</h2><p>好了，今天我们学习了两种在Kubernetes里管理配置信息的API对象ConfigMap和Secret，它们分别代表了明文信息和机密敏感信息，存储在etcd里，在需要的时候可以注入Pod供Pod使用。</p><p>简单小结一下今天的要点：</p><ol>
<li>ConfigMap记录了一些Key-Value格式的字符串数据，描述字段是“data”，不是“spec”。</li>
<li>Secret与ConfigMap很类似，也使用“data”保存字符串数据，但它要求数据必须是Base64编码，起到一定的保密效果。</li>
<li>在Pod的“env.valueFrom”字段中可以引用ConfigMap和Secret，把它们变成应用可以访问的环境变量。</li>
<li>在Pod的“spec.volumes”字段中可以引用ConfigMap和Secret，把它们变成存储卷，然后在“spec.containers.volumeMounts”字段中加载成文件的形式。</li>
<li>ConfigMap和Secret对存储数据的大小没有限制，但小数据用环境变量比较适合，大数据应该用存储卷，可根据具体场景灵活应用。</li>
</ol><h2>课下作业</h2><p>最后是课下作业时间，给你留两个思考题：</p><ol>
<li>说一说你对ConfigMap和Secret这两个对象的理解，它们有什么异同点？</li>
<li>如果我们修改了ConfigMap/Secret的YAML，然后使用 <code>kubectl apply</code> 命令更新对象，那么Pod里关联的信息是否会同步更新呢？你可以自己验证看看。</li>
</ol><p>欢迎在留言区分享你的学习所得，下节课是这个章节的实战课，我们下节课再见。</p><p><img src="https://static001.geekbang.org/resource/image/0f/47/0f4c7f7d64d6a08885353459ed99eb47.jpg?wh=1920x2402" alt="图片"></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/cd/dc/75ca619d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>江湖十年</span>
  </div>
  <div class="_2_QraFYR_0">ConfigMap 和 Secret 对存储数据的大小是有限制的，限制为 1MiB，官方文档：https:&#47;&#47;kubernetes.io&#47;zh-cn&#47;docs&#47;concepts&#47;configuration&#47;configmap&#47;#motivation</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢补充。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-22 08:08:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/ed/3a/ab8faba0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>陶乐思</span>
  </div>
  <div class="_2_QraFYR_0">用env和volume两种方式创建的pod都尝试了，修改了ConfigMap&#47;Secret 的 YAML后 kubectl apply，Pod里的value并不会改变。<br>所以这两种方式在创建pod的时候，其实都是一次性拷贝，POD controller manager只会管理POD这个层级，并不会发现POD之下层级发生的变化。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-23 06:17:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/cd/e0/c85bb948.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>朱雯</span>
  </div>
  <div class="_2_QraFYR_0">1. configmap 和secret相同点有很多，其中一点就是键值对数据存储。不同点就是一个加密，一个不加密，但是secret只支持base64加密吗，可以支持其他格式加密吗？如果都是base64加密，存在破解的可能性吗。是否不安全，如果不是可以选择不同的加密方式，会不会安全点。<br>2. 我看大家的答案是pod并不会修改里面的值，可我测试的结果是env-pod确实不会修改值，实际上应该是大家说的一次性拷贝，但另外一个vol-pod我测试的结果是，会改变值啊，难道我测试的方式有什么问题吗?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.base64只是一种编码方式，不是加密，Kubernetes里可以配置成对secret加密。<br><br>2.环境变量是pod启动时注入的，所以不会改，volume的方式会改，两种方式不一样。<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-27 22:52:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/WrANpwBMr6DsGAE207QVs0YgfthMXy3MuEKJxR8icYibpGDCI1YX4DcpDq1EsTvlP8ffK1ibJDvmkX9LUU4yE8X0w/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>星垂平野阔</span>
  </div>
  <div class="_2_QraFYR_0">把configmap挂进了pod里面，然后重新deploy了下configmap，发现pod里面的变量还是原来的，没有同步更新。<br>个人猜测应该是挂载的同时已经把configmap的内容引入pod内部，除非pod重启，不然不会随着它更新。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great </p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-22 22:21:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/14/b9/47377590.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Jasper</span>
  </div>
  <div class="_2_QraFYR_0">迫不及待的看完，期待下一节</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: nice</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-22 10:00:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/1b/f8/01e7fc0e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>郑小鹿</span>
  </div>
  <div class="_2_QraFYR_0">课后问题回答：<br>1、说一说你对 ConfigMap 和 Secret 这两个对象的理解，它们有什么异同点？<br>相同点：<br>都可以用来把配置数据和服务程序分离<br>都是一种用于存储的API对象<br>都以键值对k-v的方式存储数据<br>都可以作为数据卷挂载在其他API对象上使用<br>都不适合存储大数据，每个 ConfigMap &#47;Secret 最多支持存储1MB的数据，毕竟对内存有消耗<br><br><br>不同点：<br>ConfigMap一般存储非机密信息<br>Secret用于存储机密信息，默认是Base64编码方式对value字符进行处理。<br>Secret保存在etcd中内容是未经过加密的，对于Secret资源的权限要做好控制，可以通过RBAC规则来限制或者是使用其他加密方式<br><br><br>2、如果我们修改了 ConfigMap&#47;Secret 的 YAML，然后使用 kubectl apply 命令更新对象，那么 Pod 里关联的信息是否会同步更新呢？你可以自己验证看看。<br>如果 ConfigMap 是作为环境变量方式使用的，那数据不会被自动更新。 想要更新这些数据需要重新启动 Pod。<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-28 16:55:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/9d/a4/e481ae48.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lesserror</span>
  </div>
  <div class="_2_QraFYR_0">老师，有几个小问题：<br><br>1. 像配置这块儿，有多少个不同类型的配置，就需要定义多少个不同的Volume进行挂载吗？例如ConfigMap 和 Secret 这里挂载了两个Volume。<br><br>2. vol-pod   0&#47;1     CrashLoopBackOf   当pod变成这个状态的时候，只能删除了再重新创建吗？ 使用命令 ：kubectl delete -f vol-pod.yml。<br><br>3. 课外小贴士的最后一条，没太明白，老师能再说说看吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.是的，一个对象只能挂成一个volume。<br><br>2.是的，不过删之前最好用describe看看它为什么出错，不然可能删除后也不能恢复。<br><br>3.加载volome的文件名就是ConfigMap、secret的key名，有的时候就不合适，可以用这种方式来改名。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-01 13:25:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/ibZVAmmdAibBeVpUjzwId8ibgRzNk7fkuR5pgVicB5mFSjjmt2eNadlykVLKCyGA0GxGffbhqLsHnhDRgyzxcKUhjg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>pyhhou</span>
  </div>
  <div class="_2_QraFYR_0">想请教老师几个问题：<br><br>1、`echo -n &quot;123456&quot; | base64` 加 -n 仅仅是为了去掉换行符吗，`&quot;123456&quot;` 中并没有换行符，为什么加 -n 与不加 -n 的结果有区别？<br><br>2、构建 Pod 的时候，Secret 中的变量会被自动解码，K8S 是如何知道该用何种方式进行解码？需要通过 Secret 对象中的参数进行指定吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1. echo会默认加上一个换行符，看不到但是有，所以-n就可以去掉。<br><br>2.Secret的规范就是用base64编码，所以不需要指定。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-23 08:11:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/42/df/a034455d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>罗耀龙@坐忘</span>
  </div>
  <div class="_2_QraFYR_0">yaml文件还真不好写，我对着课文写minikube运行不了，用老师的一下就过了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 一开始肯定容易写错，特别是多个嵌套字段的层次，可以先用kubectl create创建样板，写多了就熟能生巧。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-22 11:44:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/0c/86/8e52afb8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>花花大脸猫</span>
  </div>
  <div class="_2_QraFYR_0">请教老师一个问题：使用kubectl apply 命令更新对象，证明pod中的关联信息不会同步更新。那有没有机制能够在更新对象之后 同步更新pod中的关联信息的？类似以配置中心的推送一样的，不需要重启pod。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个可能要自己用程序来实现了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-31 22:51:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/cd/e0/c85bb948.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>朱雯</span>
  </div>
  <div class="_2_QraFYR_0">vol-env的逻辑应该是和docker -v挂载是一样的，docker -v是宿主机一样的，两者改变是同步的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-27 23:00:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2c/82/ec/99b480e8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>hiDaLao</span>
  </div>
  <div class="_2_QraFYR_0">另外我发现使用kubectl describe pod时，所有的pod都挂载了一个类型为Projected的kube-api-access-xxx的特殊volume，请问下老师这个volume时用来做授权的吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，它与rbac和ServiceAccount有关。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-26 23:37:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2c/82/ec/99b480e8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>hiDaLao</span>
  </div>
  <div class="_2_QraFYR_0">思考1:<br>configMap和secret都可以用阿里管理配置信息，都支持env和volume两种形式在pod中使用。不过configMap使用明文保存配置信息，secret采用加密方式保存；<br>思考2:<br>我的验证结果跟前面几位同学有点不一样，使用env的方式，修改ConfigMap和Secret的YAML然后kubectl apply后pod中关联的信息确实不会同步更新；但如果使用volume的话，这些关联信息是会更新的，只是更新的不及时，猜测后台有一个定时任务负责处理。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-26 23:33:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/85/ac/10d68f01.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>江同学</span>
  </div>
  <div class="_2_QraFYR_0">如果我们修改了 ConfigMap&#47;Secret 的 YAML，然后使用 kubectl apply 命令更新对象，那么 Pod 里关联的信息不会同步更新，需要把使用到的pod删除，重新使用apply命令更新对象才会更新到更改的信息</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-22 19:28:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/25/87/f3a69d1b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>peter</span>
  </div>
  <div class="_2_QraFYR_0">请教老师几个问题：<br>Q1：“kube-root-ca.crt”是什么configMap对象<br>按照文中的第一个对象创建例子，创建了一个info对象，用kubectl get cm查看，<br>除了info对象以外，还有一个对象kube-root-ca.crt，这是谁创建的？<br><br>Q2：k8s默认使用Base64进行加密，文中用的linux工具，也是Base64，<br>不都是一样吗？<br><br>Q3：etcd是k8s默认就启动的吗？我启动k8s时，用命令“minikube start --kubernetes-version=v1.23.3”，<br>并没有显式启动etcd。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.当然是Kubernetes自己了。<br><br>2.我们自己用base64工具可以更加自由方便。<br><br>3.是的，etcd是核心组件，Kubernetes集群启动就会运行。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-22 16:52:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/27/ce/58/71ed845f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Dexter</span>
  </div>
  <div class="_2_QraFYR_0">Linux中环境变量🈶️限制，不能食用“-”</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-22 07:12:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/89/ba/009ee13c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>霍霍</span>
  </div>
  <div class="_2_QraFYR_0">相见恨晚，老师带着，感觉k8s的概念学起来轻松愉快</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great.</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-17 20:27:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2c/c7/89/16437396.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>极客酱酱</span>
  </div>
  <div class="_2_QraFYR_0">实测：修改了 ConfigMap&#47;Secret 的 YAML，然后使用 kubectl apply后，通过env环境变量的方式加载的配置不会更新，通过mount文件方式加载的配置会在延时约20s后更新配置。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-08 22:15:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2e/de/7e/549a899d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>李童心小组</span>
  </div>
  <div class="_2_QraFYR_0">vol-pod 会更新，但感觉不是实时的，要等一段时间，是这样的吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，对于这个特性我没有过多关注，感兴趣可以自己深入研究。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-16 16:52:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/24/c4/51/5bca1604.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>aLong</span>
  </div>
  <div class="_2_QraFYR_0">1. ConfigMap与Secret很类似，两人使用方式上基本相同。主要不同凸显在数据的编码方式，Secret是base64，ConfigMap是明文。 <br>2. 当Secret更新后，通过Volume形式ConfigMap、Secret是会更新的，与其相反就是ENV形式。 ENV形式使用方面比Volume显得方便，无需读取文件。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-10 14:18:49</div>
  </div>
</div>
</div>
</li>
</ul>