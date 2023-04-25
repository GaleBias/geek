<audio title="11｜YAML：Kubernetes世界里的通用语" src="https://static001.geekbang.org/resource/audio/c9/66/c90de7084e64edacf93704yy2dac2f66.mp3" controls="controls"></audio> 
<p>你好，我是Chrono。</p><p>在上次课里，我们一起研究了Kubernetes的内部架构和组成，知道它分为控制面和数据面。控制面管理集群，数据面跑业务应用，节点内部又有apiserver、etcd、scheduler、kubelet、kube-proxy等组件，它们互相协作来维护整个集群的稳定运行。</p><p>这套独特的Master/Node架构是Kubernetes得以安身立命的根本，但仅依靠这套“内功心法”是不是就能够随意仗剑走天涯了呢？</p><p>显然不行。就像许多武侠、玄幻作品里的人物一样，Kubernetes也需要一份“招式秘籍”才能把自己的“内功”完全发挥出来，只有内外兼修才能够达到笑傲江湖的境界。</p><p>而这份“招式秘籍”，就是Kubernetes世界里的标准工作语言YAML，所以今天，我就来讲讲为什么要有YAML、它是个什么样子、该怎么使用。</p><h2>声明式与命令式是怎么回事</h2><p>Kubernetes使用的YAML语言有一个非常关键的特性，叫“声明式”（Declarative），对应的有另外一个词：“命令式”（Imperative）。</p><p>所以在详细了解YAML之前，我们得先来看看“<strong>声明式</strong>”与“<strong>命令式</strong>”这两种工作方式，它们在计算机世界里的关系有点像小说里的“剑宗”与“气宗”。</p><!-- [[[read_end]]] --><p>我们在入门篇里学习的Docker命令和Dockerfile就属于“命令式”，大多数编程语言也属于命令式，它的特点是交互性强，注重顺序和过程，你必须“告诉”计算机每步该做什么，所有的步骤都列清楚，这样程序才能够一步步走下去，最后完成任务，显得计算机有点“笨”。</p><p>“声明式”，在Kubernetes出现之前比较少见，它与“命令式”完全相反，不关心具体的过程，更注重结果。我们不需要“教”计算机该怎么做，只要告诉它一个目标状态，它自己就会想办法去完成任务，相比起来自动化、智能化程度更高。</p><p>这两个概念比较抽象，不太好理解，也是Kubernetes初学者经常遇到的障碍之一。Kubernetes官网上特意以空调为例，解说“声明式”的原理，但我感觉还是没有说得太清楚，所以这里我就再以“打车”来形象地解释一下“命令式”和“声明式”的区别。</p><p><img src="https://static001.geekbang.org/resource/image/78/52/781e4bcc7c8c89263cca2710488a8152.jpg?wh=1200x750" alt="图片"></p><p>假设你要打车去高铁站，但司机不熟悉路况，你就只好不厌其烦地告诉他该走哪条路、在哪个路口转向、在哪里进出主路、停哪个站口。虽然最后到达了目的地，但这一路上也费了很多口舌，发出了无数的“命令”。很显然，这段路程就属于“命令式”。</p><p>现在我们来换一种方式，同样是去高铁站，但司机经验丰富，他知道哪里有拥堵、哪条路的红绿灯多、哪段路有临时管控、哪里可以抄小道，此时你再多嘴无疑会干扰他的正常驾驶，所以，你只要给他一个“声明”：我要去高铁站，接下来就可以舒舒服服地躺在后座上休息，顺利到达目的地了。</p><p>在这个“打车”的例子里，Kubernetes就是这样的一位熟练的司机，Master/Node架构让它对整个集群的状态了如指掌，内部的众多组件和插件也能够自动监控管理应用。</p><p>这个时候我们再用“命令式”跟它打交道就不太合适了，因为它知道的信息比我们更多更全面，不需要我们这个外行去指导它这个内行，所以我们最好是做一个“<strong>甩手掌柜</strong>”，用“声明式”把任务的目标告诉它，比如使用哪个镜像、什么时候运行，让它自己去处理执行过程中的细节。</p><p>那么，该用什么方式去给Kubernetes发出一个“声明”呢？</p><p>容器技术里的Shell脚本和Dockerfile可以很好地描述“命令式”，但对于“声明式”就不太合适了，这个时候，我们需要使用专门的YAML语言。</p><h2>什么是YAML</h2><p>YAML语言创建于2001年，比XML晚了三年。XML你应该知道吧，它是一种类似HTML的标签式语言，有很多繁文缛节。而YAML虽然在名字上模仿了XML，但实质上与XML完全不同，更适合人类阅读，计算机解析起来也很容易。</p><p>YAML的官网（<a href="https://yaml.org/">https://yaml.org/</a>）有对语言规范的完整介绍，所以我就不在这里列举语言的细节了，只讲一些与Kubernetes相关的要点，帮助你快速掌握。</p><p>你需要知道，<strong>YAML是JSON的超集</strong>，支持整数、浮点数、布尔、字符串、数组和对象等数据类型。也就是说，任何合法的JSON文档也都是YAML文档，如果你了解JSON，那么学习YAML会容易很多。</p><p>但和JSON比起来，YAML的语法更简单，形式也更清晰紧凑，比如：</p><ul>
<li>使用空白与缩进表示层次（有点类似Python），可以不使用花括号和方括号。</li>
<li>可以使用 <code>#</code> 书写注释，比起JSON是很大的改进。</li>
<li>对象（字典）的格式与JSON基本相同，但Key不需要使用双引号。</li>
<li>数组（列表）是使用 <code>-</code> 开头的清单形式（有点类似MarkDown）。</li>
<li>表示对象的 <code>:</code> 和表示数组的 <code>-</code> 后面都必须要有空格。</li>
<li>可以使用 <code>---</code> 在一个文件里分隔多个YAML对象。</li>
</ul><p>下面我们来看几个YAML的简单示例。</p><p>首先是数组，它使用 <code>-</code> 列出了三种操作系统：</p><pre><code class="language-yaml"># YAML数组(列表)
OS:
  - linux
  - macOS
  - Windows
</code></pre><p>这段YAML对应的JSON如下：</p><pre><code class="language-json">{
&nbsp; "OS": ["linux", "macOS", "Windows"]
}
</code></pre><p>对比可以看到YAML形式上很简单，没有闭合花括号、方括号的麻烦，每个元素后面也不需要逗号。</p><p>再来看一个YAML对象，声明了1个Master节点，3个Worker节点：</p><pre><code class="language-yaml"># YAML对象(字典)
Kubernetes:
  master: 1
  worker: 3
</code></pre><p>它等价的JSON如下：</p><pre><code class="language-json">{
&nbsp; "Kubernetes": {
&nbsp; &nbsp; "master": 1,
&nbsp; &nbsp; "worker": 3
&nbsp; }
}
</code></pre><p>注意到了吗YAML里的Key都不需要使用双引号，看起来更舒服。</p><p>把YAML的数组、对象组合起来，我们就可以描述出任意的Kubernetes资源对象，第三个例子略微复杂点，你可以自己尝试着解释一下：</p><pre><code class="language-yaml"># 复杂的例子，组合数组和对象
Kubernetes:
  master:
    - apiserver: running
    - etcd: running
  node:
    - kubelet: running
    - kube-proxy: down
    - container-runtime: [docker, containerd, cri-o]
</code></pre><p>关于YAML语言的其他知识点我就不再一一细说了，都整理在了这张图里，你可以参考<a href="https://yaml.org">YAML官网</a>，在今后的课程中慢慢体会。</p><p><img src="https://static001.geekbang.org/resource/image/d4/04/d4f3d4cc27a8a4a70d4898b41efebf04.jpg?wh=1920x2030" alt="图片"></p><h2>什么是API对象</h2><p>学到这里还不够，因为YAML语言只相当于“语法”，要与Kubernetes对话，我们还必须有足够的“词汇”来表示“语义”。</p><p>那么应该声明Kubernetes里的哪些东西，才能够让Kubernetes明白我们的意思呢？</p><p>作为一个集群操作系统，Kubernetes归纳总结了Google多年的经验，在理论层面抽象出了很多个概念，用来描述系统的管理运维工作，这些概念就叫做“<strong>API对象</strong>”。说到这个名字，你也许会联想到上次课里讲到的Kubernetes组件<strong>apiserver</strong>。没错，它正是来源于此。</p><p>因为apiserver是Kubernetes系统的唯一入口，外部用户和内部组件都必须和它通信，而它采用了HTTP协议的URL资源理念，API风格也用RESTful的GET/POST/DELETE等等，所以，这些概念很自然地就被称为是“API对象”了。</p><p>那都有哪些API对象呢？</p><p>你可以使用 <code>kubectl api-resources</code> 来查看当前Kubernetes版本支持的所有对象：</p><pre><code class="language-bash">kubectl api-resources
</code></pre><p><img src="https://static001.geekbang.org/resource/image/b2/91/b259e8bfbd0d15b796228d92ede42a91.png?wh=1920x701" alt="图片"></p><p>在输出的“NAME”一栏，就是对象的名字，比如ConfigMap、Pod、Service等等，第二栏“SHORTNAMES”则是这种资源的简写，在我们使用kubectl命令的时候很有用，可以少敲几次键盘，比如Pod可以简写成po，Service可以简写成svc。</p><p>在使用kubectl命令的时候，你还可以加上一个参数 <code>--v=9</code>，它会显示出详细的命令执行过程，清楚地看到发出的HTTP请求，比如：</p><pre><code class="language-bash">kubectl get pod --v=9
</code></pre><p><img src="https://static001.geekbang.org/resource/image/3f/31/3fe4823f6ba10600e63c197487e84931.png?wh=1920x571" alt="图片"></p><p>从截图里可以看到，kubectl客户端等价于调用了curl，向8443端口发送了HTTP GET 请求，URL是 <code>/api/v1/namespaces/default/pods</code>。</p><p>目前的Kubernetes 1.23版本有50多种API对象，全面地描述了集群的节点、应用、配置、服务、账号等等信息，apiserver会把它们都存储在数据库etcd里，然后kubelet、scheduler、controller-manager等组件通过apiserver来操作它们，就在API对象这个抽象层次实现了对整个集群的管理。</p><h2>如何描述API对象</h2><p>现在我们就来看看如何以YAML语言，使用“声明式”在Kubernetes里描述并创建API对象。</p><p>之前我们运行Nginx的命令你还记得吗？使用的是 <code>kubectl run</code>，和Docker一样是“命令式”的：</p><pre><code class="language-plain">kubectl run ngx --image=nginx:alpine
</code></pre><p>我们来把它改写成“声明式”的YAML，说清楚我们想要的Nginx应用是个什么样子，也就是“目标状态”，让Kubernetes自己去决定如何拉取镜像运行：</p><pre><code class="language-yaml">apiVersion: v1
kind: Pod
metadata:
&nbsp; name: ngx-pod
&nbsp; labels:
&nbsp; &nbsp; env: demo
&nbsp; &nbsp; owner: chrono

spec:
&nbsp; containers:
&nbsp; - image: nginx:alpine
&nbsp; &nbsp; name: ngx
&nbsp; &nbsp; ports:
&nbsp; &nbsp; - containerPort: 80
</code></pre><p>有了刚才YAML语言知识“打底”，相信你基本上能够把它看明白，知道它是一个Pod，要使用nginx:alpine镜像创建一个容器，开放端口80，而其他的部分，就是Kubernetes对API对象强制的格式要求了。</p><p>因为API对象采用标准的HTTP协议，为了方便理解，我们可以借鉴一下HTTP的报文格式，把API对象的描述分成“header”和“body”两部分。</p><p>“header”包含的是API对象的基本信息，有三个字段：<strong>apiVersion</strong>、<strong>kind</strong>、<strong>metadata</strong>。</p><ul>
<li>apiVersion表示操作这种资源的API版本号，由于Kubernetes的迭代速度很快，不同的版本创建的对象会有差异，为了区分这些版本就需要使用apiVersion这个字段，比如v1、v1alpha1、v1beta1等等。</li>
<li>kind表示资源对象的类型，这个应该很好理解，比如Pod、Node、Job、Service等等。</li>
<li>metadata这个字段顾名思义，表示的是资源的一些“元信息”，也就是用来标记对象，方便Kubernetes管理的一些信息。</li>
</ul><pre><code class="language-yaml">apiVersion: v1
kind: Pod
metadata:
&nbsp; name: ngx-pod
&nbsp; labels:
&nbsp; &nbsp; env: demo
&nbsp; &nbsp; owner: chrono
</code></pre><p>比如在这个YAML示例里就有两个“元信息”，一个是name，给Pod起了个名字叫 <code>ngx-pod</code>，另一个是labels，给Pod“贴”上了一些便于查找的标签，分别是 <code>env</code> 和 <code>owner</code>。</p><p><strong>apiVersion</strong>、<strong>kind</strong>、<strong>metadata</strong>都被kubectl用于生成HTTP请求发给apiserver，你可以用 <code>--v=9</code> 参数在请求的URL里看到它们，比如：</p><pre><code class="language-plain">https://192.168.49.2:8443/api/v1/namespaces/default/pods/ngx-pod
</code></pre><p>和HTTP协议一样，“header”里的apiVersion、kind、metadata这三个字段是任何对象都必须有的，而“body”部分则会与对象特定相关，每种对象会有不同的规格定义，在YAML里就表现为 <code>spec</code> 字段（即specification），表示我们对对象的“期望状态”（desired status）。</p><p>还是来看这个Pod，它的spec里就是一个 <code>containers</code> 数组，里面的每个元素又是一个对象，指定了名字、镜像、端口等信息：</p><pre><code class="language-yaml">spec:
&nbsp; containers:
&nbsp; - image: nginx:alpine
&nbsp; &nbsp; name: ngx
&nbsp; &nbsp; ports:
&nbsp; &nbsp; - containerPort: 80
</code></pre><p>现在把这些字段综合起来，我们就能够看出，这份YAML文档完整地描述了一个类型是Pod的API对象，要求使用v1版本的API接口去管理，其他更具体的名称、标签、状态等细节都记录在了metadata和spec字段等里。</p><p>使用 <code>kubectl apply</code>、<code>kubectl delete</code>，再加上参数 <code>-f</code>，你就可以使用这个YAML文件，创建或者删除对象了：</p><pre><code class="language-plain">kubectl apply -f ngx-pod.yml
kubectl delete -f ngx-pod.yml
</code></pre><p>Kubernetes收到这份“声明式”的数据，再根据HTTP请求里的POST/DELETE等方法，就会自动操作这个资源对象，至于对象在哪个节点上、怎么创建、怎么删除完全不用我们操心。</p><h2>如何编写YAML</h2><p>讲到这里，相信你对如何使用YAML与Kubernetes沟通应该大概了解了，不过疑问也会随之而来：这么多API对象，我们怎么知道该用什么apiVersion、什么kind？metadata、spec里又该写哪些字段呢？还有，YAML看起来简单，写起来却比较麻烦，缩进对齐很容易搞错，有没有什么简单的方法呢？</p><p>这些问题最权威的答案无疑是Kubernetes的官方参考文档（<a href="https://kubernetes.io/docs/reference/kubernetes-api/">https://kubernetes.io/docs/reference/kubernetes-api/</a>），API对象的所有字段都可以在里面找到。不过官方文档内容太多太细，查阅起来有些费劲，所以下面我就介绍几个简单实用的小技巧。</p><p>第一个技巧其实前面已经说过了，就是 <code>kubectl api-resources</code> 命令，它会显示出资源对象相应的API版本和类型，比如Pod的版本是“v1”，Ingress的版本是“networking.k8s.io/v1”，照着它写绝对不会错。</p><p>第二个技巧，是命令 <code>kubectl explain</code>，它相当于是Kubernetes自带的API文档，会给出对象字段的详细说明，这样我们就不必去网上查找了。比如想要看Pod里的字段该怎么写，就可以这样：</p><pre><code class="language-plain">kubectl explain pod
kubectl explain pod.metadata
kubectl explain pod.spec
kubectl explain pod.spec.containers
</code></pre><p><img src="https://static001.geekbang.org/resource/image/53/85/53cf783195be896e7632c1fc6bd24185.png?wh=1920x1068" alt="图片"></p><p>使用前两个技巧编写YAML就基本上没有难度了。</p><p>不过我们还可以让kubectl为我们“代劳”，生成一份“文档样板”，免去我们打字和对齐格式的工作。<strong>这第三个技巧就是kubectl的两个特殊参数 <code>--dry-run=client</code> 和 <code>-o yaml</code>，前者是空运行，后者是生成YAML格式，结合起来使用就会让kubectl不会有实际的创建动作，而只生成YAML文件。</strong></p><p>例如，想要生成一个Pod的YAML样板示例，可以在 <code>kubectl run</code> 后面加上这两个参数：</p><pre><code class="language-plain">kubectl run ngx --image=nginx:alpine --dry-run=client -o yaml
</code></pre><p>就会生成一个绝对正确的YAML文件：</p><pre><code class="language-yaml">apiVersion: v1
kind: Pod
metadata:
&nbsp; creationTimestamp: null
&nbsp; labels:
&nbsp; &nbsp; run: ngx
&nbsp; name: ngx
spec:
&nbsp; containers:
&nbsp; - image: nginx:alpine
&nbsp; &nbsp; name: ngx
&nbsp; &nbsp; resources: {}
&nbsp; dnsPolicy: ClusterFirst
&nbsp; restartPolicy: Always
status: {}
</code></pre><p>接下来你要做的，就是查阅对象的说明文档，添加或者删除字段来定制这个YAML了。</p><p>这个小技巧还可以再进化一下，把这段参数定义成Shell变量（名字任意，比如<span dollar="">$</span>do/<span dollar="">$</span>go，这里用的是<span dollar="">$</span>out），用起来会更省事，比如：</p><pre><code class="language-plain">export out="--dry-run=client -o yaml"
kubectl run ngx --image=nginx:alpine $out
</code></pre><p>今后除了一些特殊情况，我们都不会再使用 <code>kubectl run</code> 这样的命令去直接创建Pod，而是会编写YAML，用“声明式”来描述对象，再用 <code>kubectl apply</code> 去发布YAML来创建对象。</p><h2>小结</h2><p>好了，今天就到这里，我们一起学习了“声明式”和“命令式”的区别、YAML语言的语法、如何用YAML来描述API对象，还有一些编写YAML文件的技巧。</p><p>Kubernetes采用YAML作为工作语言是它有别与其他系统的一大特色，声明式的语言能够更准确更清晰地描述系统状态，避免引入繁琐的操作步骤扰乱系统，与Kubernetes高度自动化的内部结构相得益彰，而且纯文本形式的YAML也很容易版本化，适合CI/CD。</p><p>再小结一下今天的内容要点：</p><ol>
<li>YAML是JSON的超集，支持数组和对象，能够描述复杂的状态，可读性也很好。</li>
<li>Kubernetes把集群里的一切资源都定义为API对象，通过RESTful接口来管理。描述API对象需要使用YAML语言，必须的字段是<strong>apiVersion、kind、metadata</strong>。</li>
<li>命令 <code>kubectl api-resources</code> 可以查看对象的apiVersion和kind，命令 <code>kubectl explain</code> 可以查看对象字段的说明文档。</li>
<li>命令 <code>kubectl apply</code>、<code>kubectl delete</code> 发送HTTP请求，管理API对象。</li>
<li>使用参数 <code>--dry-run=client -o yaml</code> 可以生成对象的YAML模板，简化编写工作。</li>
</ol><h2>课下作业</h2><p>最后是课下作业时间，给你留两个思考题：</p><ol>
<li>你是如何理解“命令式”和“声明式”的？为什么说空调是“声明式”的？</li>
<li>使用 <code>--v=9</code> 参数，试着解释一下YAML是如何被kubectl转换成HTTP请求的。</li>
</ol><p>欢迎在留言区分享你的思考，从今天开始我们就要习惯写YAML来创建对象了，如果学习过程中有任何问题也欢迎留言提问，我会第一时间回复你。下节课再见。</p><p><img src="https://static001.geekbang.org/resource/image/13/36/13dc437dda840dda4850fb72237b8e36.jpg?wh=1920x2868" alt=""></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/f9/8a/fc767da7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>nada</span>
  </div>
  <div class="_2_QraFYR_0">1. 因为空调的使用是我们设置温度，然后空调尽量去达成设置的温度，即我们设置的是我们想要达成的效果，而不像电视遥控器，每次操作都发送一个指令，声明式还有一个特点是容易 patch，拿空调举例，当前气温 30 度，我们先设置了 24 度，然后升高到 26 度，空调并不会先降低到 24 度，再升高到 26 度，而是将两次传递的意图 patch，直接降低到 26 度，k8s 也是这样<br>2. yaml 转换 json 的过程基本上字段一一对应，除了最后会把当前的 json 再序列化为字符串，存储到 &quot;kubectl.kubernetes.io&#47;last-applied-configuration&quot; 这个 annotations 中</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-15 11:31:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/2xoGmvlQ9qfSibVpPJyyaEriavuWzXnuECrJITmGGHnGVuTibUuBho43Uib3Y5qgORHeSTxnOOSicxs0FV3HGvTpF0A/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>psoracle</span>
  </div>
  <div class="_2_QraFYR_0">回答一下作业<br>1. 你是如何理解“命令式”和“声明式”的？为什么说空调是“声明式”的？<br>我现在从kubernetes相关知识点中学习并理解到的“声明式”一个最直白的地方是新增与修改两个命令式操作写成apply，如果创建对象不存在则创建、对象已存在就比较spec进行相应变更，当然k8s的修改实际上是先delete再create。至于kubectl create、kubectl edit、kubectl delete等都是&quot;命令式&quot;操作，告诉k8s应该怎么做。<br>空调是“声明式”的原因是我不知道当前温度是多少，只需要我知道应该开成多少度即可，即我们在操作遥控器之前就知道我们要将空调调成多少度，这是预期值。<br><br>2. 使用 --v=9 参数，试着解释一下 YAML 是如何被 kubectl 转换成 HTTP 请求的。<br>虽然yaml格式是json的超集，但在k8s中的yaml文件最终都是被转换为json格式字符串放在request body中提交到apiserver的，从`kubectl -v=9`对各种操作的调试中可以看到。<br>除此之外，还发现一些有规律的地方，如下：<br>可见简单对象（如pod, configmap, secret, serviceaccount等）调用的接口形式如 `&#47;api&#47;&lt;apiVersion&gt;&#47;namespaces&#47;&lt;namespace&gt;&#47;&lt;kinds&gt;[&#47;&lt;name&gt;]`，其中对象类型为复数形式即`kubectl api-resources`中的name字段，修改、删除与查询具体对象时在URL中有`&#47;&lt;name&gt;`部分，其它如创建、查询所有就没有。对于复合对象（简单对象的包装对象，如replicaset, deployment, statefulset, cronjob等）的URL不同的是以`&#47;apis`开头，说明是属于复合型的接口（组合服务）。<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: awesome</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-15 11:21:27</div>
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
  <div class="_2_QraFYR_0">请问一下老师, —dry-run=client -o yaml生成的YAML默认情况下会将文件保存在哪里啊？没有找到呢…环境：windows+minikube</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 它直接输出在屏幕上，需要用一个重定向功能，命令后面加上 &gt; xxx.yml，或者直接鼠标选择文本。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-15 06:09:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/1d/de/62bfa83f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>aoe</span>
  </div>
  <div class="_2_QraFYR_0">终于对一头雾水的yaml文件有了一个正确的了解，感谢老师</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: my pleasure。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-21 08:37:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/51/9b/ccea47d9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>安迪密恩</span>
  </div>
  <div class="_2_QraFYR_0">这门课太棒了，以前接触的杂乱的知识点在学完这一章之后逐渐变得脉络清晰！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 欢迎分享学习心得。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-20 12:08:24</div>
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
  <div class="_2_QraFYR_0">有了刚才 YAML 语言知识“打底”，相信你基本上能够把它看明白，知道它是一个 Pod，要使用 nginx:alpine 镜像创建一个容器，开放端口 80，而其他的部分，就是 Kubernetes 对 API 对象强制的格式要求了。<br>————<br>查了一下官方文档，其实不是“开放”，好像是是“说明”容器中暴露的端口是80</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 知道这个意思就行，不用太咬文嚼字。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-22 20:06:37</div>
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
  <div class="_2_QraFYR_0">有两个问题想请教老师，<br><br>1. 通过 `kubectl api-resources` 看到的资源名称中其实没有 pod 而是 pods，但是我们依旧可以通过 pod 来访问对应的资源，这里是否可以理解为 k8s 自动帮忙做了单复数转换？<br><br>2. 没有看到 images 相关的资源，k8s 是如何管理 image 的呢？还是说对应的镜像管理被细分到 pod 的字段中？<br><br>3. 文章中（在 “什么是 YAML” 的结尾处）给的最后一个复杂的 YAML 例子，我试着在 https:&#47;&#47;www.bejson.com&#47;json&#47;json2yaml&#47; 上面转换成 JSON，但是平台会报如下错误：<br><br>yaml错误:<br>TypeError: Cannot read properties of undefined (reading &#39;split&#39;)<br><br>感觉像是缩进的问题</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1. pod&#47;pods&#47;po在Kubernetes里都是同义词，至于单复数的语法问题我就不太好解释了。<br><br>2.是的，image在Pod的container里，见下一讲。<br><br>3.我试了可以转换，再查一下看是哪里有问题。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-21 08:15:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/2c/7e/f1efd18b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>摊牌</span>
  </div>
  <div class="_2_QraFYR_0">对于&quot;命令式&quot;和&quot;声明式&quot;，我觉得最本质的区别就是，命令式是通过实际的shell命令来执行完成某项功能，而声明式是仅仅定义一个配置文件，来描述预期的对象状态，再通过特定的命令去加载渲染这个配置文件，去完成对应的功能。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 理解的不错，不过命令式不局限于shell命令，只要是有顺序关系的一系列指令都可以叫命令式。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-16 08:48:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/f7/b1/982ea185.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Frank</span>
  </div>
  <div class="_2_QraFYR_0">有了那个三个技巧，终于可以方便地写YAML 文件了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-15 19:38:35</div>
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
  <div class="_2_QraFYR_0">1. 声明式： 我要结果。命令式：过程要符合要求，有一个不合适的比喻那就是面向对象和面向过程。要结果和要过程。<br>2. 会变成这样一条curl命令，curl -v -XGET  -H &quot;Accept: application&#47;com.github.proto-openapi.spec.v2@v1.0+protobuf&quot; -H &quot;User-Agent: kubectl&#47;v1.23.3 (linux&#47;amd64) kubernetes&#47;816c97a&quot; &#39;https:&#47;&#47;192.168.49.2:8443&#47;openapi&#47;v2?timeout=32s&#39; 不确定是不是要的答案</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-15 11:25:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/77/bd/e7ff842c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>赤色闪电</span>
  </div>
  <div class="_2_QraFYR_0">“声明式”：空调遥控器操控空调设定想要的温度和模式即可。“命令式”：电视遥控器操控电视，按一个钮换一个台。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-15 08:29:46</div>
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
  <div class="_2_QraFYR_0">首先感谢老师三个技巧。这块配置的内容我确实很头大啊。初学时看书都想跳过去看看后面内容。今天看完老师讲的有点意思啊。尤其是我喜欢那种比喻生动的多。<br><br>两种方式进行比较：<br>命令：简单、直观、快捷，上手快。 适合临时测试或实验。<br>yaml文件：文件描述了What，即应用最终要达到的状态。配置文件提供了创建资源的模板，能够重复部署。可以像管理代码一样管理部署。适合正式的、跨环境的、规模化部署。这种方式要求熟悉配置文件的语法，有一定难度。<br>另外 kubectl apply不但能够创建Kubernetes资源，也能对资源进行更新，非常方便。<br><br><br>空调这种有些需求没办法直接达到预期工作，比如制冷、制热、温度调整。 反观：电视，你想换台就换了，声音控制多少按+ -就行了。 所以空调很多功能更像是声明式。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: awesome!</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-06 17:08:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTIuj7Wx21ecNlPHCfBsQIchmFxVSlPepwUiaKh0RMGgDB0aibTM50ibQN06dDmbqjuQZUIdH4qiaRJkgQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_adb513</span>
  </div>
  <div class="_2_QraFYR_0">yaml 以前都是直接抄现成的用法，只知道必须这么用，这里把为什么讲解的很到位，的确让思绪一下明朗了，三要素 和 自定义</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 用面向对象的思想去思考会进一步加深理解。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-02 17:49:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2e/7e/26/162e8e57.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>bruce</span>
  </div>
  <div class="_2_QraFYR_0"><br>apiVersion: v1<br>kind: Pod<br>metadata:<br>  name: ngx-pod<br>  labels:<br>    env: demo<br>    owner: chrono<br><br>spec:<br>  containers:<br>  - image: nginx:alpine<br>    name: ngx<br>    ports:<br>    - containerPort: 80<br><br>metadata里的name和spec里的name有啥区别？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以用kubectl explain看看这两个字段的说明，一个是对象名字，一个是容器名字。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-20 17:35:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/86/cc/eecffb35.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Liang Li</span>
  </div>
  <div class="_2_QraFYR_0">一个好的老师能在一堆复杂的网状知识中，找到非常清晰的线性脉络，并把这些知识给串联表达起来，难得的好文章。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: many thanks.</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-15 19:59:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/28/4a/9d/2aa87437.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>沃德天·泥维森陌·拉莫帅</span>
  </div>
  <div class="_2_QraFYR_0">1. “命令式”是指采用命令式语言编写的程序，这种语言通常会指定程序执行的每一个步骤，并指定具体的操作来完成每一个步骤。<br><br>“声明式”是指采用声明式语言编写的程序。这种语言通常会指定程序的输出，而不是具体的操作。<br><br>例如，如果要编写一个程序来控制空调，那么“命令式”的方法可能是指定每一个步骤，例如打开空调、调整温度、打开风扇等，而“声明式”的方法可能是指定空调最终的状态，例如“调节到25摄氏度”。<br><br>因此，空调可以被认为是“声明式”的，因为它通过声明最终的状态来控制空调的工作方式，而不是通过指定具体的操作。<br><br><br>2. 当 kubectl 读取 YAML 文件时，它会将 YAML 文件转换成对应的 Kubernetes API 对象。比如，一个 YAML 文件可能会声明一个 pod 对象，而 kubectl 可以将这个 pod对象转换成 Kubernetes API 所需的 JSON 格式。<br><br>接下来，kubectl 会将这个转换后的 JSON 对象转换成一个 HTTP 请求，并发送到 Kubernetes API 服务器。Kubernetes API 服务器会根据这个 HTTP 请求来执行相应的操作。<br><br>具体过程：<br>kubectl 会使用一个 HTTP 客户端库（比如 http.Client）来将这个 JSON 对象转换成一个 HTTP 请求。这个 HTTP 请求会包含一些基本信息，比如请求方法（比如 POST）、请求路径（比如 &#47;apis&#47;apps&#47;v1&#47;namespaces&#47;default&#47;deployments）和请求内容（比如转换后的 JSON 对象）。<br><br>一旦这个 HTTP 请求被创建，kubectl 就可以使用这个 HTTP 客户端库来发送这个请求到 Kubernetes API 服务器。Kubernetes API 服务器会根据这个 HTTP 请求来执行相应的操作。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: awesome.</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-11 10:24:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/a6/56/abb7bfe3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>云韵</span>
  </div>
  <div class="_2_QraFYR_0">老师 我执行了这些命令  （环境是 mac m1 直接在本机上运行的 docker minikube）<br>export out=&quot;--dry-run=client -o yaml&quot;<br>kubectl run ngx --image=nginx:alpine $out<br>但是报这个错 error: Invalid dry-run value (client -o yaml). Must be &quot;none&quot;, &quot;server&quot;, or &quot;client&quot;.<br>请老师帮忙解答一下，谢谢<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 已经有很多同学遇到这样的问题，应该是shell对变量处理的不同，mac应该用的是zsh吧，尝试改成bash试试。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-18 14:33:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/60/a1/8f003697.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>静心</span>
  </div>
  <div class="_2_QraFYR_0">很棒，讲的很清楚</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: many thanks.</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-16 10:42:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/10/bb/f1061601.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Demon.Lee</span>
  </div>
  <div class="_2_QraFYR_0">Yaml 在 Json 的基础上又把双引号，大括号各种非必要的字符都给干掉了，信息的有效性更高了，还能表示多个文档。<br><br>Xml —&gt; Json —&gt; Yaml，一路走来，贯穿它们的主线是什么？<br><br>信息的密度越来越高。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-08 20:15:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/10/bb/f1061601.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Demon.Lee</span>
  </div>
  <div class="_2_QraFYR_0">总有一种强迫症，想把<br>```<br>spec:<br>  containers:<br>  - image: nginx:alpine<br>    name: ngx<br>    ports:<br>    - containerPort: 80<br>```<br><br>中的 - 往里缩进两个字符 😂<br><br>```<br>spec:<br>  containers:<br>    - image: nginx:alpine<br>       name: ngx<br>       ports:<br>         - containerPort: 80<br>```</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以的，这个在yaml里完全允许，不过这样就多了一层缩进，有的人就不是太喜欢。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-08 09:31:00</div>
  </div>
</div>
</div>
</li>
</ul>