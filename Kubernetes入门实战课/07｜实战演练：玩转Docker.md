<audio title="07｜实战演练：玩转Docker" src="https://static001.geekbang.org/resource/audio/ed/df/ed1dceba91d121176cbf355e26602ddf.mp3" controls="controls"></audio> 
<p>你好，我是Chrono。</p><p>学到今天的这次课，我们的“入门篇”就算是告一段落了，有这些容器知识作为基础，很快我们就要正式开始学习Kubernetes。不过在那之前，来对前面的课程做一个回顾和实践，把基础再夯实一下。</p><p>要提醒你的是，Docker相关的内容很多很广，在入门篇中，我只从中挑选出了一些最基本最有用的介绍给你。而且在我看来，我们不需要完全了解Docker的所有功能，我也不建议你对Docker的内部架构细节和具体的命令行参数做过多的了解，太浪费精力，只要会用够用，需要的时候能够查找官方手册就行。</p><p>毕竟我们这门课程的目标是Kubernetes，而Docker只不过是众多容器运行时（Container Runtime）中最出名的一款而已。当然，如果你当前的工作是与Docker深度绑定，那就另当别论了。</p><p>好下面我先把容器技术做一个简要的总结，然后演示两个实战项目：使用Docker部署Registry和WordPress。</p><h2>容器技术要点回顾</h2><p>容器技术是后端应用领域的一项重大创新，它彻底变革了应用的开发、交付与部署方式，是“云原生”的根本（<a href="https://time.geekbang.org/column/article/528619">01讲</a>）。</p><p>容器基于Linux底层的namespace、cgroup、chroot等功能，虽然它们很早就出现了，但直到Docker“横空出世”，把它们整合在一起，容器才真正走近了大众的视野，逐渐为广大开发者所熟知（<a href="https://time.geekbang.org/column/article/528640">02讲</a>）。</p><!-- [[[read_end]]] --><p>容器技术中有三个核心概念：<strong>容器（Container）</strong>、<strong>镜像（Image）</strong>，以及<strong>镜像仓库（Registry）</strong>（<a href="https://time.geekbang.org/column/article/528651">03讲</a>）。</p><p><img src="https://static001.geekbang.org/resource/image/c8/fe/c8116066bdbf295a7c9fc25b87755dfe.jpg?wh=1920x1048" alt="图片"></p><p>从本质上来说，容器属于虚拟化技术的一种，和虚拟机（Virtual Machine）很类似，都能够分拆系统资源，隔离应用进程，但容器更加轻量级，运行效率更高，比虚拟机更适合云计算的需求。</p><p>镜像是容器的静态形式，它把应用程序连同依赖的操作系统、配置文件、环境变量等等都打包到了一起，因而能够在任何系统上运行，免除了很多部署运维和平台迁移的麻烦。</p><p>镜像内部由多个层（Layer）组成，每一层都是一组文件，多个层会使用Union FS技术合并成一个文件系统供容器使用。这种细粒度结构的好处是相同的层可以共享、复用，节约磁盘存储和网络传输的成本，也让构建镜像的工作变得更加容易（<a href="https://time.geekbang.org/column/article/528660">04讲</a>）。</p><p>为了方便管理镜像，就出现了镜像仓库，它集中存放各种容器化的应用，用户可以任意上传下载，是分发镜像的最佳方式（<a href="https://time.geekbang.org/column/article/528677">05讲</a>）。</p><p>目前最知名的公开镜像仓库是Docker Hub，其他的还有quay.io、gcr.io，我们可以在这些网站上找到许多高质量镜像，集成到我们自己的应用系统中。</p><p>容器技术有很多具体的实现，Docker是最初也是最流行的容器技术，它的主要形态是运行在Linux上的“Docker Engine”。我们日常使用的 <code>docker</code> 命令其实只是一个前端工具，它必须与后台服务“Docker daemon”通信才能实现各种功能。</p><p>操作容器的常用命令有 <code>docker ps</code>、<code>docker run</code>、<code>docker exec</code>、<code>docker stop</code> 等；操作镜像的常用命令有 <code>docker images</code>、<code>docker rmi</code>、<code>docker build</code>、<code>docker tag</code> 等；操作镜像仓库的常用命令有 <code>docker pull</code>、<code>docker push</code> 等。</p><p>好简单地回顾了容器技术，下面我们就来综合运用在“入门篇”所学到的各个知识点，开始实战演练，玩转Docker。</p><h2>搭建私有镜像仓库</h2><p>在第5节课讲Docker Hub的时候曾经说过，在离线环境里，我们可以自己搭建私有仓库。但因为镜像仓库是网络服务的形式，当时还没有学到容器网络相关的知识，所以只有到了现在，我们具备了比较完整的Docker知识体系，才能够搭建私有仓库。</p><p>私有镜像仓库有很多现成的解决方案，今天我只选择最简单的Docker Registry，而功能更完善的CNCF Harbor留到后续学习Kubernetes时再介绍。</p><p>你可以在Docker Hub网站上搜索“registry”，找到它的官方页面（<a href="https://registry.hub.docker.com/_/registry/">https://registry.hub.docker.com/_/registry/</a>）：</p><p><img src="https://static001.geekbang.org/resource/image/cb/26/cbab37b3a4f2a621ab81564bcdba1326.png?wh=1408x712" alt="图片"></p><p>Docker Registry的网页上有很详细的说明，包括下载命令、用法等，我们可以完全照着它来操作。</p><p>首先，你需要使用 <code>docker pull</code> 命令拉取镜像：</p><pre><code class="language-plain">docker pull registry
</code></pre><p>然后，我们需要做一个端口映射，对外暴露端口，这样Docker Registry才能提供服务。它的容器内端口是5000，简单起见，我们在外面也使用同样的5000端口，所以运行命令就是 <code>docker run -d -p 5000:5000 registry</code> ：</p><pre><code class="language-plain">docker run -d -p 5000:5000 registry
</code></pre><p>启动Docker Registry之后，你可以使用 <code>docker ps</code> 查看它的运行状态，可以看到它确实把本机的5000端口映射到了容器内的5000端口。</p><p><img src="https://static001.geekbang.org/resource/image/18/01/183e8953bb544c1dbf42022bfacef101.png?wh=1920x123" alt="图片"></p><p>接下来，我们就要使用 <code>docker tag</code> 命令给镜像打标签再上传了。因为上传的目标不是默认的Docker Hub，而是本地的私有仓库，所以镜像的名字前面还必须再加上仓库的地址（域名或者IP地址都行），形式上和HTTP的URL非常像。</p><p>比如在这里，我就把“nginx:alpine”改成了“127.0.0.1:5000/nginx:alpine”：</p><pre><code class="language-plain">docker tag nginx:alpine 127.0.0.1:5000/nginx:alpine
</code></pre><p>现在，这个镜像有了一个附加仓库地址的完整名字，就可以用 <code>docker push</code> 推上去了：</p><pre><code class="language-plain">docker push 127.0.0.1:5000/nginx:alpine
</code></pre><p><img src="https://static001.geekbang.org/resource/image/c8/7f/c851b97a722de6fe2a144ec75c69e27f.png?wh=1920x444" alt="图片"></p><p>为了验证是否已经成功推送，我们可以把刚才打标签的镜像删掉，再重新下载：</p><pre><code class="language-plain">docker rmi  127.0.0.1:5000/nginx:alpine
docker pull 127.0.0.1:5000/nginx:alpine
</code></pre><p><img src="https://static001.geekbang.org/resource/image/54/6e/54a288df48531205236f024ecd91816e.png?wh=1920x272" alt="图片"></p><p>这里 <code>docker pull</code> 确实完成了镜像下载任务，不过因为原来的层原本就已经存在，所以不会有实际的下载动作，只会创建一个新的镜像标签。</p><p>Docker Registry虽然没有图形界面，但提供了RESTful API，也可以发送HTTP请求来查看仓库里的镜像，具体的端点信息可以参考官方文档（<a href="https://docs.docker.com/registry/spec/api/">https://docs.docker.com/registry/spec/api/</a>），下面的这两条curl命令就分别获取了镜像列表和Nginx镜像的标签列表：</p><pre><code class="language-plain">curl 127.1:5000/v2/_catalog
curl 127.1:5000/v2/nginx/tags/list
</code></pre><p><img src="https://static001.geekbang.org/resource/image/c4/e8/c4715261ef7e131b53af31931c1ecee8.png?wh=1294x246" alt="图片"></p><p>可以看到，因为应用被封装到了镜像里，所以我们只用简单的一两条命令就完成了私有仓库的搭建工作，完全不需要复杂的软件安装、环境设置、调试测试等繁琐的操作，这在容器技术出现之前简直是不可想象的。</p><h2>搭建WordPress网站</h2><p>Docker Registry应用比较简单，只用单个容器就运行了一个完整的服务，下面我们再来搭建一个有点复杂的WordPress网站。</p><p>网站需要用到三个容器：WordPress、MariaDB、Nginx，它们都是非常流行的开源项目，在Docker Hub网站上有官方镜像，网页上的说明也很详细，所以具体的搜索过程我就略过了，直接使用 <code>docker pull</code> 拉取它们的镜像：</p><pre><code class="language-plain">docker pull wordpress:5
docker pull mariadb:10
docker pull nginx:alpine
</code></pre><p>我画了一个简单的网络架构图，你可以直观感受一下它们之间的关系：</p><p><img src="https://static001.geekbang.org/resource/image/59/ca/59dfbe961bcd233b83e1c1ec064e2eca.png?wh=1920x643" alt="图片"></p><p>这个系统可以说是比较典型的网站了。MariaDB作为后面的关系型数据库，端口号是3306；WordPress是中间的应用服务器，使用MariaDB来存储数据，它的端口是80；Nginx是前面的反向代理，它对外暴露80端口，然后把请求转发给WordPress。</p><p>我们先来运行MariaDB。根据说明文档，需要配置“MARIADB_DATABASE”等几个环境变量，用 <code>--env</code> 参数来指定启动时的数据库、用户名和密码，这里我指定数据库是“db”，用户名是“wp”，密码是“123”，管理员密码（root password）也是“123”。</p><p>下面就是启动MariaDB的 <code>docker run</code> 命令：</p><pre><code class="language-plain">docker run -d --rm \
&nbsp; &nbsp; --env MARIADB_DATABASE=db \
&nbsp; &nbsp; --env MARIADB_USER=wp \
&nbsp; &nbsp; --env MARIADB_PASSWORD=123 \
&nbsp; &nbsp; --env MARIADB_ROOT_PASSWORD=123 \
&nbsp; &nbsp; mariadb:10
</code></pre><p>启动之后，我们还可以使用 <code>docker exec</code> 命令，执行数据库的客户端工具“mysql”，验证数据库是否正常运行：</p><pre><code class="language-plain">docker exec -it 9ac mysql -u wp -p
</code></pre><p>输入刚才设定的用户名“wp”和密码“123”之后，我们就连接上了MariaDB，可以使用 <code>show databases;</code> 和 <code>show tables;</code> 等命令来查看数据库里的内容。当然，现在肯定是空的。</p><p><img src="https://static001.geekbang.org/resource/image/ef/5e/ef28b4482ef243c215b09dae6889695e.png?wh=1920x1282" alt="图片"></p><p>因为Docker的bridge网络模式的默认网段是“172.17.0.0/16”，宿主机固定是“172.17.0.1”，而且IP地址是顺序分配的，所以如果之前没有其他容器在运行的话，MariaDB容器的IP地址应该就是“172.17.0.2”，这可以通过 <code>docker inspect</code> 命令来验证：</p><pre><code class="language-plain">docker inspect 9ac |grep IPAddress
</code></pre><p><img src="https://static001.geekbang.org/resource/image/f3/e6/f371e89b97a0410acb628ac2889427e6.png?wh=1224x240" alt="图片"></p><p>现在数据库服务已经正常，该运行应用服务器WordPress了，它也要用 <code>--env</code> 参数来指定一些环境变量才能连接到MariaDB，注意“WORDPRESS_DB_HOST”必须是MariaDB的IP地址，否则会无法连接数据库：</p><pre><code class="language-plain">docker run -d --rm \
&nbsp; &nbsp; --env WORDPRESS_DB_HOST=172.17.0.2 \
&nbsp; &nbsp; --env WORDPRESS_DB_USER=wp \
&nbsp; &nbsp; --env WORDPRESS_DB_PASSWORD=123 \
&nbsp; &nbsp; --env WORDPRESS_DB_NAME=db \
&nbsp; &nbsp; wordpress:5
</code></pre><p>WordPress容器在启动的时候并没有使用 <code>-p</code> 参数映射端口号，所以外界是不能直接访问的，我们需要在前面配一个Nginx反向代理，把请求转发给WordPress的80端口。</p><p>配置Nginx反向代理必须要知道WordPress的IP地址，同样可以用 <code>docker inspect</code> 命令查看，如果没有什么意外的话它应该是“172.17.0.3”，所以我们就能够写出如下的配置文件（Nginx的用法可参考其他资料，这里就不展开讲了）：</p><pre><code class="language-plain">server {
&nbsp; listen 80;
&nbsp; default_type text/html;

&nbsp; location / {
&nbsp; &nbsp; &nbsp; proxy_http_version 1.1;
&nbsp; &nbsp; &nbsp; proxy_set_header Host $host;
&nbsp; &nbsp; &nbsp; proxy_pass http://172.17.0.3;
&nbsp; }
}
</code></pre><p>有了这个配置文件，最关键的一步就来了，我们需要用 <code>-p</code> 参数把本机的端口映射到Nginx容器内部的80端口，再用 <code>-v</code> 参数把配置文件挂载到Nginx的“conf.d”目录下。这样，Nginx就会使用刚才编写好的配置文件，在80端口上监听HTTP请求，再转发到WordPress应用：</p><pre><code class="language-plain">docker run -d --rm \
&nbsp; &nbsp; -p 80:80 \
&nbsp; &nbsp; -v `pwd`/wp.conf:/etc/nginx/conf.d/default.conf \
&nbsp; &nbsp; nginx:alpine
</code></pre><p>三个容器都启动之后，我们再用 <code>docker ps</code> 来看看它们的状态：</p><p><img src="https://static001.geekbang.org/resource/image/35/15/353f0f3aed1d1b540312548c29cb6c15.png?wh=1920x198" alt="图片"></p><p>可以看到，WordPress和MariaDB虽然使用了80和3306端口，但被容器隔离，外界不可见，只有Nginx有端口映射，能够从外界的80端口收发数据，网络状态和我们的架构图是一致的。</p><p>现在整个系统就已经在容器环境里运行好了，我们来打开浏览器，输入本机的“127.0.0.1”或者是虚拟机的IP地址（我这里是“<a href="http://192.168.10.208">http://192.168.10.208</a>”），就可以看到WordPress的界面：</p><p><img src="https://static001.geekbang.org/resource/image/a6/31/a63084a8dd95d0034ba72dcb60613531.png?wh=1224x1156" alt="图片"></p><p>在创建基本的用户、初始化网站之后，我们可以再登录MariaDB，看看是否已经有了一些数据：</p><p><img src="https://static001.geekbang.org/resource/image/a2/1e/a22dcabe805471e304a74c715e7fb51e.png?wh=1920x1749" alt="图片"></p><p>可以看到，WordPress已经在数据库里新建了很多的表，这就证明我们的容器化的WordPress网站搭建成功。</p><h2>小结</h2><p>好了，今天我们简单地回顾了一下容器技术，这里有一份思维导图，是对前面所有容器知识要点的总结，你可以对照着用来复习。</p><p><img src="https://static001.geekbang.org/resource/image/79/16/79f8c75e018e0a82eff432786110ef16.jpg?wh=1920x2142" alt="图片"></p><p>我们还使用Docker实际搭建了两个服务：Registry镜像仓库和WordPress网站。</p><p>通过这两个项目的实战演练，你应该能够感受到容器化对后端开发带来的巨大改变，它简化了应用的打包、分发和部署，简单的几条命令就可以完成之前需要编写大量脚本才能完成的任务，对于开发、运维来绝对是一个“福音”。</p><p>不过，在感受容器便利的同时，你有没有注意到它还是存在一些遗憾呢？比如说：</p><ul>
<li>我们还是要手动运行一些命令来启动应用，然后再人工确认运行状态。</li>
<li>运行多个容器组成的应用比较麻烦，需要人工干预（如检查IP地址）才能维护网络通信。</li>
<li>现有的网络模式功能只适合单机，多台服务器上运行应用、负载均衡该怎么做？</li>
<li>如果要增加应用数量该怎么办？这时容器技术完全帮不上忙。</li>
</ul><p>其实，如果我们仔细整理这些运行容器的 <code>docker run</code> 命令，写成脚本，再加上一些Shell、Python编程来实现自动化，也许就能够得到一个勉强可用的解决方案。</p><p>这个方案已经超越了容器技术本身，是在更高的层次上规划容器的运行次序、网络连接、数据持久化等应用要素，也就是现在我们常说的“<strong>容器编排</strong>”（Container Orchestration）的雏形，也正是后面要学习的Kubernetes的主要出发点。</p><h2>课下作业</h2><p>最后是课下作业时间，给你留两个思考题：</p><ol>
<li>学完了“入门篇”，和刚开始相比，你对容器技术有了哪些更深入的思考和理解？</li>
<li>你觉得容器编排应该解决哪些方面的问题？</li>
</ol><p>欢迎积极留言讨论，如果有收获，也欢迎你转发给身边的朋友一起学习。</p><p>下节课是视频课，我会用视频直观演示我们前面学过的操作，我们下节课见。<br>
<img src="https://static001.geekbang.org/resource/image/43/fa/43ccbfc6b629eed231ffec8d6eaf99fa.jpg?wh=1920x2051" alt=""></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/9d/a4/e481ae48.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lesserror</span>
  </div>
  <div class="_2_QraFYR_0">之前对docker的了解很杂乱，知识点很细碎、分散，没有一个整体、清晰的认知。<br><br>看过中文互联网上面别人的一些教程，要么照本宣科，要么浅尝辄止。<br><br>老师的课程虽然没有做到知识点的面面俱到，当然也不可能做到。但是，算是整体上帮我又重新梳理了一遍docker的整体架构，让我对其认识更加清晰了一些。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 建议自己再用思维导图或者其他形式把知识体系梳理一下，这样才能真正掌握。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-06 12:04:28</div>
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
  <div class="_2_QraFYR_0">思考题：<br>1. 相较于之前只知道容器是用来环境隔离，看完入门篇后，对容器技术有了一个比较宏观和基本的了解，列出来如下：<br> 1）知道了什么是镜像，以及镜像和容器的关系<br> 2）知道了 DockerHub 这样的镜像仓库<br> 3）明白了容器和虚拟机的不同<br> 4）懂得如何通过 Dockerfile 来构建自己的镜像<br> 5）理解了 Docker 的整体内部框架 docker client -&gt; docker daemon -&gt; registry<br> 6) 知道了，也实际操作了一些常用的镜像以及容器相关的指令<br>。。。<br><br>感觉学习到的这些东西可以覆盖工作中大多数的场景了，但是这些知识只能说是运用于小规模的东西。想要把容器技术玩的得心应手，还需了解一些容器应用的最佳实践，和一些工程化的理念和工具<br><br>2. 感觉容器编排主要应用于大规模集成应用。可以类比分布式系统，入门篇中讲的知识用在单机应用上是没有问题的，但是规模一旦变大到系统层面，就会出现一些问题，比如如何保证数据一致性？如何保证负载均衡？如何尽可能减少网络故障所带来的影响？如何能保证数据（容器）的持久化等等。。。这些问题需要运用容器编排来解决<br><br><br>另外想请教老师 2 个问题<br><br>文章一开始提到容器运行时（Container Runtime）这个概念，该如何理解？这是和容器绑定的一门技术吗？<br><br>还有就是，我看你在 curl 指令中直接将本地 IP 127.0.0.1 简写成 127.1，是说 curl 中允许这样的简写，还是说这本身就是一个惯例？<br><br>谢谢老师 🙏</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 总结的很好。<br><br>1. 运行时是一个计算机里比较通用的概念，比如Java运行时，可以理解成是一个底层支持库。<br><br>2.“127.0.0.1 ”简写成“127.1”，这是通用的，因为中间的“0”可以“压缩”，其实ipv6里也是这么用的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-10 07:25:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/23/bb/74/edc07099.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>柳成荫</span>
  </div>
  <div class="_2_QraFYR_0">1. 刚开始学容器的时候觉得容器就是一个小的虚拟机，部署一套应用应该可以把中间件和应用都部署到同一个容器中，每个容器都应该对外暴露端口才能被访问，现在觉得有些应用可以不用暴露端口，反而更加安全<br>2. 容器编排应该会解决容器启动、维护的麻烦，应用集群等问题<br>请教一个问题，部署一个java应用，jdk应该安装在宿主机还是应用的容器里面呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.理解的很不错。<br><br>2.应该用jdk的镜像作为基础镜像，然后再打包应用，这样多个Java容器就可以复用底层的jdk了。<br><br>3.有了容器环境，宿主机上除了容器，就不需要再安装其他东西了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-06 10:11:02</div>
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
  <div class="_2_QraFYR_0">老师，有几个小问题：<br><br>Q1：k8s应该算是容易编排技术吧？如果学会了k8s的日常操作，关于docker的使用是不是就可以减少了。了解一个大概就好了，很多操作应该逐渐偏向对k8s的操作？<br><br>Q2: 对于容器化的应用来说，如果想从外部访问对应的服务，是不是必须要做端口映射这一步？宿主机的端口需要唯一性，容器应用的端口随意指定，即使多个容器应用有相同的端口。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.这个要看实际工作需求，目前来说docker了解基本就足够了，Kubernetes要投入更多精力。<br><br>2.对的，容器用namespace把内外网络隔离了，所以外界要访问就必须要有映射这个动作，到了宿主机端口号也同样不能冲突。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-06 12:13:09</div>
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
  <div class="_2_QraFYR_0"><br><br>q1: 容器编排技术是有价值的，我之前以为价值不大，只是改变启动和使用方式，增加一些命令。<br>q2： 容器编排解决的问题是：一些非自动化，而是需要强人工干预的东西，比如网络交互需要知道对方ip地址的情况，虽然可以写自动化脚本，但这个并不通用，所以是一套通用的自动化方案。另外多台机器，自动创建负载均衡，创建路由的配置问题。这些是编排的范围。<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great!</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-06 09:45:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/8e/3f/1f529b26.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>henry</span>
  </div>
  <div class="_2_QraFYR_0">2022&#47;08&#47;16，docker pull mariadb:10，会有问题，docker run 时报错：[ERROR] [Entrypoint]: mariadbd failed while attempting to check config，Can&#39;t initialize timers.<br><br>docker pull mariadb:10.8.2  解决问题，参考如下：<br>https:&#47;&#47;github.com&#47;MariaDB&#47;mariadb-docker&#47;issues&#47;434</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-16 14:33:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/57/ed/d50de13c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>mj4ever</span>
  </div>
  <div class="_2_QraFYR_0">老师的教程中，ng → wp → db，相互之间是通过容器的 IP 地址来访问，尝试以下两种方法，可以不指定 IP 地址，通过容器名：<br><br>1、启动容器时加入了自定义的网络 my_network，类型是 bridge；其原理是容器之间的互联是通过 Docker DNS Server；代码如下<br>docker run -d --rm --name db1 \<br>    --network my_network \<br>    --env MARIADB_DATABASE=db \<br>    --env MARIADB_USER=wp \<br>    --env MARIADB_PASSWORD=123 \<br>    --env MARIADB_ROOT_PASSWORD=123 \<br>    mariadb:10<br><br>docker run -d --rm --name wp1 \<br>    --network my_network \<br>    --env WORDPRESS_DB_HOST=db1 \<br>    --env WORDPRESS_DB_USER=wp \<br>    --env WORDPRESS_DB_PASSWORD=123 \<br>    --env WORDPRESS_DB_NAME=db \<br>    wordpress:5<br><br>vi wp.conf<br>server {<br>  listen 80;<br>  default_type text&#47;html;<br><br>  location &#47; {<br>      proxy_http_version 1.1;<br>      proxy_set_header Host $host;<br>      proxy_pass http:&#47;&#47;wp1;<br>  }<br>}<br><br>docker run -d --rm --name ng1 \<br>    --network my_network \<br>    -p 80:80 \<br>    -v `pwd`&#47;wp.conf:&#47;etc&#47;nginx&#47;conf.d&#47;default.conf \<br>    nginx:alpine<br><br>2、启动 WordPress wp1 时，link 到 db1，即--link db1:db1，  启动 Nginx ng1 时，link 到 wp1，即--link wp1:wp1；其原理是容器之间的互联是通过容器里的 &#47;etc&#47;hosts；代码如下<br>docker run -d --rm --name db1 \<br>    --env MARIADB_DATABASE=db \<br>    --env MARIADB_USER=wp \<br>    --env MARIADB_PASSWORD=123 \<br>    --env MARIADB_ROOT_PASSWORD=123 \<br>    mariadb:10<br><br>docker run -d --rm --name wp1 \<br>    --link db1:db1 \<br>    --env WORDPRESS_DB_HOST=db1 \<br>    --env WORDPRESS_DB_USER=wp \<br>    --env WORDPRESS_DB_PASSWORD=123 \<br>    --env WORDPRESS_DB_NAME=db \<br>    wordpress:5<br><br>vi wp.conf<br>server {<br>  listen 80;<br>  default_type text&#47;html;<br><br>  location &#47; {<br>      proxy_http_version 1.1;<br>      proxy_set_header Host $host;<br>      proxy_pass http:&#47;&#47;wp1;<br>  }<br>}<br><br>docker run -d --rm --name ng1 \<br>    --link wp1:wp1 \<br>    -p 80:80 \<br>    -v `pwd`&#47;wp.conf:&#47;etc&#47;nginx&#47;conf.d&#47;default.conf \<br>    nginx:alpine</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，docker里也可以自定义网络，用起来比IP地址要方便，但我特意没有讲，觉得容易和后面的Kubernetes混淆。<br><br>如果理解了Kubernetes的Service机制，再回头来学docker的自定义网络应该会比较容易。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-17 22:08:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/94/91/6d6ca42f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>怀草诗</span>
  </div>
  <div class="_2_QraFYR_0"><br>docker run -d --rm \<br>    -p 80:80 \<br>    -v `pwd`&#47;wp.conf:&#47;etc&#47;nginx&#47;conf.d&#47;default.conf \<br>    nginx:alpine<br>老师，这个命令中 -v 后面跟的 &#39;pwd&#39; 什么意思？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: `pwd`是shell命令，意思是获取当前目录，注意它用的是反引号，就是“1”左边的那个键。<br><br>如果不熟悉用法可以改用绝对路径，比如&#47;home&#47;xxx&#47;...&#47;wp.conf。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-25 23:18:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/0a/a4/828a431f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张申傲</span>
  </div>
  <div class="_2_QraFYR_0">从头到尾跟到了现在，再加上本节课的实战，深切地感受到了容器化对于开发和运维方式的重塑。老师的课程深入浅出，受益匪浅~</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 共同进步！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-13 16:58:39</div>
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
  <div class="_2_QraFYR_0">小贴士总能带来惊喜</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: my pleasure。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-13 09:47:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83eqF6ViaDyAibEKbcKfWoGXe8lCbb8wqes5g3JezHWNLf4DIl92QwXX43HWv408BxzkOKmKb2HpKJuIw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_b537b2</span>
  </div>
  <div class="_2_QraFYR_0">老师请问下使用Docker Registry搭建本地镜像仓库后用docker pull拉取镜像怎么不是去共有仓库拉取而是默认去本地私有仓库拉取，这中间是不是自动配置的镜像源地址</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: pull的时候加上本地仓库的地址，比如127.0.0.1</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-10 13:18:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83eqw0R25Bt0iahFhEHfnxmzr9iaZf0eLsDQtFUJzgGkYwHTqicU9TydMngrJ4yL7D50awD2VibHBAdqplQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_18dfaf</span>
  </div>
  <div class="_2_QraFYR_0">什么时候更新下一课</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">编辑回复: 每周一三五更新，偶尔还会有额外的加餐掉落:)</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-06 21:11:35</div>
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
  <div class="_2_QraFYR_0">老师，看了图中的nginx ，wd，mariadb 网络架构图<br>想问下：是三个容器共同用一个docker 运行时吗？  在网络方面用bridge模式下，它们三也是独立网络环境吧？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 当然都是在一个docker环境里，每个容器的网络环境是隔离的，但它们的网卡都接在docker0网桥上，所以可以互通。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-06 17:42:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/0c/e1/f663213e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>拾掇拾掇</span>
  </div>
  <div class="_2_QraFYR_0">1.之前都是单个运行mysql容器或者redis容器，没怎么多容器之前联动。今天搭建wordpress让我对容器隔离环境、随意删除、随意构建有了更深的感受<br>2.容器编排解决的应该是一组容器批量构建、整体维护吧</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great，后面再学习Kubernetes继续加深理解。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-06 10:57:22</div>
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
  <div class="_2_QraFYR_0">老师好，最近遇到一个很奇怪的容器问题。那就是同一个镜像，在不同的机器上，里面的文件权限存在不同，但我们记忆中只修改了宿主机的权限，宿主机的文件目录权限不是和容器完全隔离吗，老师有相关思路吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不了解实际场景，不好判断，如果周围有其他熟悉容器的同事可以一起研究。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-06 09:20:24</div>
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
  <div class="_2_QraFYR_0">请教老师几个问题：<br>Q1：镜像的“层”复用的问题：<br>--- 下载两个镜像A和B，这两个镜像都有一个层“M”，那么，这个层“M”在两个镜像中各存在一份，另外，docker会将此层“M”在宿主机上单独存一份，即在宿主机上，层“M”会存在三份，是这样吗？<br>--- 镜像A和B运行的时候，A的容器中有层“M”，B的容器中也有层“M”，是吗？<br>--- 镜像A在同一台宿主机上可以运行多个容器吗？如果可以，比如运行3个容器，那么，每个容器都有层“M”，对吗？<br>Q2：用rmi删除镜像后，镜像不存在了，但其包含的层还存在宿主机上，对吗？<br>这个问题和Q1有点关联，比如下载镜像A，其中含有层“M”，用rmi删除镜像后，镜像不存在了，其包含的层“M”不存在了，但宿主机上其实还有一份层“M”，对吗？<br>Q3：wordpress例子中，为什么nginx可以访问WP？<br>Wp没有对外暴露端口，而nginx对于WP来说就是外部访问者啊，应该不能访问才对啊。<br>Q4：小贴士的第一项中，挂载用法问题： <br> --- 挂载方法：  -v   &#47;home&#47;zhangsan   &#47;var&#47;lib&#47;registry， 其中&#47;home&#47;zhangsan是宿主机上的目录，是这样用吗？  <br>--- 挂载后，是把镜像本身放到&#47;home&#47;zhangsan下面吗？ 还是说，镜像不放在&#47;home&#47;zhangsan下面，但会把镜像用到的数据放到&#47;home&#47;zhangsan下面？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.相同的层在docker里只会保存一份，多个镜像会共享复用，不会有多份。<br><br>2.一个镜像可以启动多个容器，当然每个容器都会看到相同的层。<br><br>3.镜像是由多个层构成的，删除镜像如果层被其他镜像引用就不会删除。<br><br>4.这些容器使用bridge模式，都在一个网段里，当然可以互相访问。<br><br>5.可以看前一讲， -v本地目录：容器目录，容器就会访问被挂载的本地目录，也就是说，Registry容器会把数据存放到本地了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-06 09:08:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/3c/4d/3dec4bfe.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>蔡晓慧</span>
  </div>
  <div class="_2_QraFYR_0">1.之前没有容器技术的时候，部署应用各种环境会存在各种的问题，尤其是我司有C++的项目，需要安装依赖，光调试环境就搞得很头疼，现在有了容器技术，打包成镜像，随处可用，很方便；<br>2.感觉容器编排就是为了大型应用服务的。我们目前有十几个镜像，是用docker-compose用来做项目交付，应对一般场景足够用。但很多公司要求HA，这时候感觉上k8s感觉好一点，可扩展性好，快速扩容，我们也在往这个方向发展，所以自己有空来学习学习。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-02 17:31:36</div>
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
  <div class="_2_QraFYR_0">引用：<br>`curl 127.1:5000&#47;v2&#47;_catalog<br>curl 127.1:5000&#47;v2&#47;nginx&#47;tags&#47;list`<br>地址错误，应该是127.0.0.1吧<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 127.0.0.1可以“缩写”成127.1，中间的0可以省略，试一下就知道了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-17 10:04:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/5c/18/9f35fddb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>后三排</span>
  </div>
  <div class="_2_QraFYR_0">Docker 内安装了 snap，然后 snap install core. 报错 error: cannot communicate with server: Post &quot;http:&#47;&#47;localhost&#47;v2&#47;snaps&#47;core&quot;: dial unix &#47;run&#47;snapd.socket: connect: no such file or directory.<br><br>这？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 没怎么用过snap，也许可以在网上搜一下这个错误信息。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-08 15:27:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/5c/18/9f35fddb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>后三排</span>
  </div>
  <div class="_2_QraFYR_0">对于遇到 System has not been booted with systemd as init system (PID 1). Can&#39;t operate.<br>Failed to connect to bus: Host is down，这样的问题如何处理最好呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 运行容器会有一些特权限制，docker有一些参数可以开启这些特权，需要查一下它的文档。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-08 15:04:44</div>
  </div>
</div>
</div>
</li>
</ul>