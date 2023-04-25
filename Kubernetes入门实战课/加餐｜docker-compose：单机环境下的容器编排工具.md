<audio title="加餐｜docker-compose：单机环境下的容器编排工具" src="https://static001.geekbang.org/resource/audio/1f/c1/1f41e6e169d3b48f678e43c34f6d72c1.mp3" controls="controls"></audio> 
<p>你好，我是Chrono。</p><p>我们的课程学到了这里，你已经对Kubernetes有相当程度的了解了吧。</p><p>作为云原生时代的操作系统，Kubernetes源自Docker又超越了Docker，依靠着它的master/node架构，掌控成百上千台的计算节点，然后使用YAML语言定义各种API对象来编排调度容器，实现了对现代应用的管理。</p><p>不过，你有没有觉得，在Docker和Kubernetes之间，是否还缺了一点什么东西呢？</p><p>Kubernetes的确是非常强大的容器编排平台，但强大的功能也伴随着复杂度和成本的提升，不说那几十个用途各异的API对象，单单说把Kubernetes运行起来搭建一个小型的集群，就需要耗费不少精力。但是，有的时候，我们只是想快速启动一组容器来执行简单的开发、测试工作，并不想承担Kubernetes里apiserver、scheduler、etcd这些组件的运行成本。</p><p>显然，在这种简易任务的应用场景里，Kubernetes就显得有些“笨重”了。即使是“玩具”性质的minikube、kind，对电脑也有比较高的要求，会“吃”掉不少的计算资源，属于“大材小用”。</p><p>那到底有没有这样的工具，既像Docker一样轻巧易用，又像Kubernetes一样具备容器编排能力呢？</p><!-- [[[read_end]]] --><p>今天我就来介绍docker-compose，它恰好满足了刚才的需求，是一个在单机环境里轻量级的容器编排工具，填补了Docker和Kubernetes之间的空白位置。</p><h2>什么是docker-compose</h2><p>还是让我们从Docker诞生那会讲起。</p><p>在Docker把容器技术大众化之后，Docker周边涌现出了数不胜数的扩展、增强产品，其中有一个名字叫“Fig”的小项目格外令人瞩目。</p><p>Fig为Docker引入了“容器编排”的概念，使用YAML来定义容器的启动参数、先后顺序和依赖关系，让用户不再有Docker冗长命令行的烦恼，第一次见识到了“声明式”的威力。</p><p>Docker公司也很快意识到了Fig这个小工具的价值，于是就在2014年7月把它买了下来，集成进Docker内部，然后改名成了“docker-compose”。</p><p><img src="https://static001.geekbang.org/resource/image/ec/ab/ecb1194e994c0a127d4818310dac14ab.png?wh=400x300" alt="图片" title="图片来自网络"></p><p>从这段简短的历史中你可以看到，虽然docker-compose也是容器编排技术，也使用YAML，但它的基因与Kubernetes完全不同，走的是Docker的技术路线，所以在设计理念和使用方法上有差异就不足为怪了。</p><p>docker-compose自身的定位是管理和运行多个Docker容器的工具，很显然，它没有Kubernetes那么“宏伟”的目标，只是用来方便用户使用Docker而已，所以学习难度比较低，上手容易，很多概念都是与Docker命令一一对应的。</p><p>但这有时候也会给我们带来困扰，毕竟docker-compose和Kubernetes同属容器编排领域，用法不一致就容易导致认知冲突、混乱。考虑到这一点，我们在学习docker-compose的时候就要把握一个“度”，够用就行，不要太过深究，否则会对Kubernetes的学习造成一些不良影响。</p><h2>如何使用docker-compose</h2><p>docker-compose的安装非常简单，它在GitHub（<a href="https://github.com/docker/compose">https://github.com/docker/compose</a>）上提供了多种形式的二进制可执行文件，支持Windows、macOS、Linux等操作系统，也支持x86_64、arm64等硬件架构，可以直接下载。</p><p>在Linux上安装的Shell命令我放在这里了，用的是最新的2.6.1版本：</p><pre><code class="language-bash"># intel x86_64
sudo curl -SL https://github.com/docker/compose/releases/download/v2.6.1/docker-compose-linux-x86_64 \
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; -o /usr/local/bin/docker-compose

# apple m1
sudo curl -SL https://github.com/docker/compose/releases/download/v2.6.1/docker-compose-linux-aarch64 \
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; -o /usr/local/bin/docker-compose

sudo chmod +x /usr/local/bin/docker-compose
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
</code></pre><p>安装完成之后，来看一下它的版本号，命令是 <code>docker-compose version</code>，用法和 <code>docker version</code> 是一样的：</p><pre><code class="language-plain">docker-compose version
</code></pre><p><img src="https://static001.geekbang.org/resource/image/12/79/12a28accca14eec348353521a89d4879.png?wh=996x120" alt="图片"></p><p>接下来，我们就要编写YAML文件，来管理Docker容器了，先用<a href="https://time.geekbang.org/column/article/528740">第7讲</a>里的私有镜像仓库作为示范吧。</p><p>docker-compose里管理容器的核心概念是“<strong>service</strong>”。注意，它与Kubernetes里的 <code>Service</code> 虽然名字很像，但却是完全不同的东西。docker-compose里的“service”就是一个容器化的应用程序，通常是一个后台服务，用YAML定义这些容器的参数和相互之间的关系。</p><p>如果硬要和Kubernetes对比的话，和“service”最像的API对象应该算是Pod里的container了，同样是管理容器运行，但docker-compose的“service”又融合了一些Service、Deployment的特性。</p><p>下面的这个就是私有镜像仓库Registry的YAML文件，关键字段就是“<strong>services</strong>”，对应的Docker命令我也列了出来：</p><pre><code class="language-plain">docker run -d -p 5000:5000 registry
</code></pre><pre><code class="language-yaml">services:
&nbsp; registry:
&nbsp; &nbsp; image: registry
&nbsp; &nbsp; container_name: registry
&nbsp; &nbsp; restart: always
&nbsp; &nbsp; ports:
&nbsp; &nbsp; &nbsp; - 5000:5000
</code></pre><p>把它和Kubernetes对比一下，你会发现它和Pod定义非常像，“services”相当于Pod，而里面的“service”就相当于“spec.containers”：</p><pre><code class="language-yaml">apiVersion: v1
kind: Pod
metadata:
&nbsp; name: ngx-pod
spec:
  restartPolicy: Always
&nbsp; containers:
&nbsp; - image: nginx:alpine
&nbsp; &nbsp; name: ngx
&nbsp; &nbsp; ports:
&nbsp; &nbsp; - containerPort: 80
</code></pre><p>比如用 <code>image</code> 声明镜像，用 <code>ports</code> 声明端口，很容易理解，只是在用法上有些不一样，像端口映射用的就还是Docker的语法。</p><p>由于docker-compose的字段定义在官网（<a href="https://docs.docker.com/compose/compose-file/">https://docs.docker.com/compose/compose-file/</a>）上有详细的说明文档，我就不在这里费口舌解释了，你可以自行参考。</p><p>需要提醒的是，在docker-compose里，每个“service”都有一个自己的名字，它同时也是这个容器的唯一网络标识，有点类似Kubernetes里 <code>Service</code> 域名的作用。</p><p>好，现在我们就可以启动应用了，命令是 <code>docker-compose up -d</code>，同时还要用 <code>-f</code> 参数来指定YAML文件，和 <code>kubectl apply</code> 差不多：</p><pre><code class="language-plain">docker-compose -f reg-compose.yml up -d
</code></pre><p><img src="https://static001.geekbang.org/resource/image/8e/b5/8ed5ba47b5bc6c6dc4b999415772deb5.png?wh=1440x246" alt="图片"></p><p>因为docker-compose在底层还是调用的Docker，所以它启动的容器用 <code>docker ps</code> 也能够看到：</p><p><img src="https://static001.geekbang.org/resource/image/3c/3e/3c89d2d81c380a120d5d05617602c43e.png?wh=1546x158" alt="图片"></p><p>不过，我们用 <code>docker-compose ps</code> 能够看到更多的信息：</p><pre><code class="language-plain">docker-compose -f reg-compose.yml ps
</code></pre><p><img src="https://static001.geekbang.org/resource/image/5f/5f/5fab07a5ba4736a65380729ba392645f.png?wh=1896x186" alt="图片"></p><p>下面我们把Nginx的镜像改个标签，上传到私有仓库里测试一下：</p><pre><code class="language-plain">docker tag nginx:alpine 127.0.0.1:5000/nginx:v1
docker push 127.0.0.1:5000/nginx:v1
</code></pre><p>再用curl查看一下它的标签列表，就可以看到确实上传成功了：</p><pre><code class="language-plain">curl 127.1:5000/v2/nginx/tags/list
</code></pre><p><img src="https://static001.geekbang.org/resource/image/c5/d8/c56a4bfdc87ce8cf945fa055997486d8.png?wh=1300x126" alt="图片"></p><p>想要停止应用，我们需要使用 <code>docker-compose down</code> 命令：</p><pre><code class="language-plain">docker-compose -f reg-compose.yml down
</code></pre><p><img src="https://static001.geekbang.org/resource/image/4c/5e/4c681530438eb50b1e2ba90c0c8de45e.png?wh=1406x246" alt="图片"></p><p>通过这个小例子，我们就成功地把“命令式”的Docker操作，转换成了“声明式”的docker-compose操作，用法与Kubernetes十分接近，同时还没有Kubernetes那些昂贵的运行成本，在单机环境里可以说是最适合不过了。</p><h2>使用docker-compose搭建WordPress网站</h2><p>不过，私有镜像仓库Registry里只有一个容器，不能体现docker-compose容器编排的好处，我们再用它来搭建一次WordPress网站，深入感受一下。</p><p>架构图和第7讲还是一样的：</p><p><img src="https://static001.geekbang.org/resource/image/59/ca/59dfbe961bcd233b83e1c1ec064e2eca.png?wh=1920x643" alt="图片"></p><p>第一步还是定义数据库MariaDB，环境变量的写法与Kubernetes的ConfigMap有点类似，但使用的字段是 <code>environment</code>，直接定义，不用再“绕一下”：</p><pre><code class="language-yaml">services:

&nbsp; mariadb:
&nbsp; &nbsp; image: mariadb:10
&nbsp; &nbsp; container_name: mariadb
&nbsp; &nbsp; restart: always

&nbsp; &nbsp; environment:
&nbsp; &nbsp; &nbsp; MARIADB_DATABASE: db
&nbsp; &nbsp; &nbsp; MARIADB_USER: wp
&nbsp; &nbsp; &nbsp; MARIADB_PASSWORD: 123
&nbsp; &nbsp; &nbsp; MARIADB_ROOT_PASSWORD: 123
</code></pre><p>我们可以再对比第7讲里启动MariaDB的Docker命令，可以发现docker-compose的YAML和命令行是非常像的，几乎可以直接照搬：</p><pre><code class="language-bash">docker run -d --rm \
    --env MARIADB_DATABASE=db \
    --env MARIADB_USER=wp \
    --env MARIADB_PASSWORD=123 \
    --env MARIADB_ROOT_PASSWORD=123 \
    mariadb:10
</code></pre><p>第二步是定义WordPress网站，它也使用 <code>environment</code> 来设置环境变量：</p><pre><code class="language-plain">services:
  ...
  
&nbsp; wordpress:
&nbsp; &nbsp; image: wordpress:5
&nbsp; &nbsp; container_name: wordpress
&nbsp; &nbsp; restart: always

&nbsp; &nbsp; environment:
&nbsp; &nbsp; &nbsp; WORDPRESS_DB_HOST: mariadb  #注意这里，数据库的网络标识
&nbsp; &nbsp; &nbsp; WORDPRESS_DB_USER: wp
&nbsp; &nbsp; &nbsp; WORDPRESS_DB_PASSWORD: 123
&nbsp; &nbsp; &nbsp; WORDPRESS_DB_NAME: db

&nbsp; &nbsp; depends_on:
&nbsp; &nbsp; &nbsp; - mariadb
</code></pre><p>不过，因为docker-compose会自动把MariaDB的名字用做网络标识，所以在连接数据库的时候（字段 <code>WORDPRESS_DB_HOST</code>）就不需要手动指定IP地址了，直接用“service”的名字 <code>mariadb</code> 就行了。这是docker-compose比Docker命令要方便的一个地方，和Kubernetes的域名机制很像。</p><p>WordPress定义里还有一个<strong>值得注意的是字段 <code>depends_on</code>，它用来设置容器的依赖关系，指定容器启动的先后顺序</strong>，这在编排由多个容器组成的应用的时候是一个非常便利的特性。</p><p>第三步就是定义Nginx反向代理了，不过很可惜，docker-compose里没有ConfigMap、Secret这样的概念，要加载配置还是必须用外部文件，无法集成进YAML。</p><p>Nginx的配置文件和第7讲里也差不多，同样的，在 <code>proxy_pass</code> 指令里不需要写IP地址了，直接用WordPress的名字就行：</p><pre><code class="language-plain">server {
&nbsp; listen 80;
&nbsp; default_type text/html;

&nbsp; location / {
&nbsp; &nbsp; &nbsp; proxy_http_version 1.1;
&nbsp; &nbsp; &nbsp; proxy_set_header Host $host;
&nbsp; &nbsp; &nbsp; proxy_pass http://wordpress;  #注意这里，网站的网络标识
&nbsp; }
}
</code></pre><p>然后我们就可以在YAML里定义Nginx了，加载配置文件用的是 <code>volumes</code> 字段，和Kubernetes一样，但里面的语法却又是Docker的形式：</p><pre><code class="language-plain">services:
  ...
  
&nbsp; nginx:
&nbsp; &nbsp; image: nginx:alpine
&nbsp; &nbsp; container_name: nginx
&nbsp; &nbsp; hostname: nginx
&nbsp; &nbsp; restart: always
&nbsp; &nbsp; ports:
&nbsp; &nbsp; &nbsp; - 80:80
&nbsp; &nbsp; volumes:
&nbsp; &nbsp; &nbsp; - ./wp.conf:/etc/nginx/conf.d/default.conf

&nbsp; &nbsp; depends_on:
&nbsp; &nbsp; &nbsp; - wordpress
</code></pre><p>到这里三个“service”就都定义好了，我们用 <code>docker-compose up -d</code> 启动网站，记得还是要用 <code>-f</code> 参数指定YAML文件：</p><pre><code class="language-plain">docker-compose -f wp-compose.yml up -d
</code></pre><p>启动之后，用 <code>docker-compose ps</code> 来查看状态：</p><p><img src="https://static001.geekbang.org/resource/image/e2/b9/e23953ca3dd05a79a660c7be9509c1b9.png?wh=1426x782" alt="图片"></p><p>我们也可以用 <code>docker-compose exec</code> 来进入容器内部，验证一下这几个容器的网络标识是否工作正常：</p><pre><code class="language-plain">docker-compose -f wp-compose.yml exec -it nginx sh
</code></pre><p><img src="https://static001.geekbang.org/resource/image/60/cb/6019120aa14369ce6cb83e880382c1cb.png?wh=1714x1204" alt="图片"></p><p>从截图里你可以看到，我们分别ping了 <code>mariadb</code> 和 <code>wordpress</code> 这两个服务，网络都是通的，不过它的IP地址段用的是“172.22.0.0/16”，和Docker默认的“172.17.0.0/16”不一样。</p><p>再打开浏览器，输入本机的“127.0.0.1”或者是虚拟机的IP地址（我这里是“<a href="http://192.168.10.208">http://192.168.10.208</a>”），就又可以看到熟悉的WordPress界面了：</p><p><img src="https://static001.geekbang.org/resource/image/db/7d/db87232f578ea8556c452c2557db437d.png?wh=1920x1411" alt="图片"></p><h2>小结</h2><p>好了，今天我们暂时离开了Kubernetes，回头看了一下Docker世界里的容器编排工具docker-compose。</p><p>和Kubernetes比起来，docker-compose有它自己的局限性，比如只能用于单机，编排功能比较简单，缺乏运维监控手段等等。但它也有优点：小巧轻便，对软硬件的要求不高，只要有Docker就能够运行。</p><p>所以虽然Kubernetes已经成为了容器编排领域的霸主，但docker-compose还是有一定的生存空间，像GitHub上就有很多项目提供了docker-compose YAML来快速搭建原型或者测试环境，其中的一个典型就是CNCF Harbor。</p><p>对于我们日常工作来说，docker-compose也是很有用的。如果是只有几个容器的简单应用，用Kubernetes来运行实在是有种“杀鸡用牛刀”的感觉，而用Docker命令、Shell脚本又很不方便，这就是docker-compose出场的时候了，它能够让我们彻底摆脱“命令式”，全面使用“声明式”来操作容器。</p><p>我再简单小结一下今天的内容：</p><ol>
<li>docker-compose源自Fig，是专门用来编排Docker容器的工具。</li>
<li>docker-compose也使用YAML来描述容器，但语法语义更接近Docker命令行。</li>
<li>docker-compose YAML里的关键概念是“service”，它是一个容器化的应用。</li>
<li>docker-compose的命令与Docker类似，比较常用的有 <code>up</code>、<code>ps</code>、<code>down</code>，用来启动、查看和停止应用。</li>
</ol><p>另外，docker-compose里还有不少有用的功能，比如存储卷、自定义网络、特权进程等等，感兴趣的话可以再去看看官网资料。</p><p>欢迎留言交流你的学习想法，我们下节课回归正课，下节课见。</p><p><img src="https://static001.geekbang.org/resource/image/24/47/2477f804c387b66d1f6188a2d7530947.jpg?wh=1920x2225" alt="图片"></p>
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
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTIaexL1b8o76RqM4F2PZhWYGxsic2EuFSWWh5IhibqfdjcDzJbhlcag1z0rECfUo0vZREbMyiaW7P8XA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>青储</span>
  </div>
  <div class="_2_QraFYR_0">这个可以用在小型公司生产线上吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 当然可以，它其实就是对docker的一个易用性包装。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-15 12:55:07</div>
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
  <div class="_2_QraFYR_0">请教老师几个问题：<br>Q1：docker-compose只能用在单机环境，不能用在集群吗？<br>Q2：文中创建的第一个docker是干什么用的？<br>文中用了这个命令“docker run -d -p 5000:5000 registry”，请问创建这个有什么用？<br>是不是这样：用“docker run -d -p 5000:5000 registry”可以启动一个容器。用yaml文件，用“docker-compose -f reg-compose.yml up -d”，也可以达到同样的目的。<br><br>Q3：需要先搭建一个本地registry吗？<br>执行“docker push 127.0.0.1:5000&#47;nginx:v1”后报错：<br>The push refers to repository [127.0.0.1:5000&#47;nginx]<br>Get &quot;http:&#47;&#47;127.0.0.1:5000&#47;v2&#47;&quot;: dial tcp 127.0.0.1:5000: connect: connection refused<br>突然感觉Q2中创建的应用就是本地registry，对吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.docker-compose和docker一样，只能是单机运行，就是对docker的包装。<br><br>2.第一个docker命令是和docker-compose对比用的。<br><br>3.是的，就是先用docker-compose运行了一个Registry，然后测试验证docker push。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-16 23:11:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/8f/52/3eebca1e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Max</span>
  </div>
  <div class="_2_QraFYR_0">docker-compose转写成k8s yaml有什么建议吗？<br>尝试使用了kompose convert工具，发现还是有很多配置无法覆盖到，比如env的引入方式就不一样。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 毕竟两者的理念差距太大，虽然都是YAML ，但也不是能够一一对应的，需要正确理解后改成kubernetes的格式。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-08 23:25:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/cd/db/7467ad23.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Bachue Zhou</span>
  </div>
  <div class="_2_QraFYR_0">相比于 kubernetes，docker-compose 不就是大道至简吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 能力太弱，怎么管理成百上千的集群？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-18 17:43:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/rURvBicplInVqwb9rX21a4IkcKkITIGIo7GE1Tcp3WWU49QtwV53qY8qCKAIpS6x68UmH4STfEcFDJddffGC7lw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>onemao</span>
  </div>
  <div class="_2_QraFYR_0">docker compose对开发来说最大的作用就是本地快速拥有数据库，消息中间件等等，无需单独安装，随时用随时删除。而且只要写好文件放到repo,idea中点一下可以一键运行和初始化，也极大方便本地开发与本地集成测试。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，对于开发测试来说非常方便，可以说是必备的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-26 18:13:27</div>
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
  <div class="_2_QraFYR_0">docker-compose 的默认配置文件名称为： docker-compose.yml<br> -f, --file FILE             Specify an alternate compose file<br>                              (default: docker-compose.yml)<br><br>2. </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1.x docker-comose的默认文件是docker-compose.yml，2.x后默认文件也可以使用compose.yml。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-18 17:54:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/cd/ba/3a348f2d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>YueShi</span>
  </div>
  <div class="_2_QraFYR_0">apt install docker-compose <br>为docker-compose version 1.29.2, build unknown<br>不是最新版的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 从GitHub项目里直接下载二进制文件。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-15 12:00:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/d0/51/f1c9ae2d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Christopher</span>
  </div>
  <div class="_2_QraFYR_0">如果安装成docker compose plugin的形式，即没有中间的横线，harbor安装会有问题，因为检测不到docke-compose😂</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 所以一般还是用传统的docker-compose的形式，和1.x兼容。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-15 07:27:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/e5/42/a994666a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>黄涵宇看起来很好吃</span>
  </div>
  <div class="_2_QraFYR_0">docker-compose -f docker-compose.yml exec -it 无法生效<br>docker exec -it 可以生效<br>原因可能是什么</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这两个命令应该是等价的，看看错误提示是什么。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-28 15:26:34</div>
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
  <div class="_2_QraFYR_0">原来 docker-compose 从 v1.27 版本开始将 version 字段给“干掉”了，再也不用理会 version: &quot;3&quot;，version: &quot;3.9&quot;，version: &quot;2&quot; 了 😂 。<br><br>为啥我之前没觉得这玩意有点反人类呢？嗯，缺少批判性思维。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 其实我觉得加version也算是个合理的决定，但后来可能是觉得带来的麻烦更多，就给去掉了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-01 11:38:34</div>
  </div>
</div>
</div>
</li>
</ul>