<audio title="04｜创建容器镜像：如何编写正确、高效的Dockerfile" src="https://static001.geekbang.org/resource/audio/f5/5e/f553115886b85799a126e5ab46508f5e.mp3" controls="controls"></audio> 
<p>你好，我是Chrono。</p><p>上一次的课程里我们一起学习了容器化的应用，也就是被打包成镜像的应用程序，然后再用各种Docker命令来运行、管理它们。</p><p>那么这又会带来一个疑问：这些镜像是怎么创建出来的？我们能不能够制作属于自己的镜像呢？</p><p>所以今天，我就来讲解镜像的内部机制，还有高效、正确地编写Dockerfile制作容器镜像的方法。</p><h2>镜像的内部机制是什么</h2><p>现在你应该知道，镜像就是一个打包文件，里面包含了应用程序还有它运行所依赖的环境，例如文件系统、环境变量、配置参数等等。</p><p>环境变量、配置参数这些东西还是比较简单的，随便用一个manifest清单就可以管理，真正麻烦的是文件系统。为了保证容器运行环境的一致性，镜像必须把应用程序所在操作系统的根目录，也就是rootfs，都包含进来。</p><p>虽然这些文件里不包含系统内核（因为容器共享了宿主机的内核），但如果每个镜像都重复做这样的打包操作，仍然会导致大量的冗余。可以想象，如果有一千个镜像，都基于Ubuntu系统打包，那么这些镜像里就会重复一千次Ubuntu根目录，对磁盘存储、网络传输都是很大的浪费。</p><p>很自然的，我们就会想到，应该把重复的部分抽取出来，只存放一份Ubuntu根目录文件，然后让这一千个镜像以某种方式共享这部分数据。</p><!-- [[[read_end]]] --><p>这个思路，也正是容器镜像的一个重大创新点：分层，术语叫“<strong>Layer</strong>”。</p><p>容器镜像内部并不是一个平坦的结构，而是由许多的镜像层组成的，每层都是只读不可修改的一组文件，相同的层可以在镜像之间共享，然后多个层像搭积木一样堆叠起来，再使用一种叫“<strong>Union FS联合文件系统</strong>”的技术把它们合并在一起，就形成了容器最终看到的文件系统（<a href="https://linoxide.com/wp-content/uploads/2015/03/docker-filesystems-busyboxrw.png">图片来源</a>）。</p><p><img src="https://static001.geekbang.org/resource/image/c7/3f/c750a7795ff4787c6639dd42bf0a473f.png?wh=800x600" alt="图片"></p><p>我来拿大家都熟悉的千层糕做一个形象的比喻吧。</p><p>千层糕也是由很多层叠加在一起的，从最上面可以看到每层里面镶嵌的葡萄干、核桃、杏仁、青丝等，每一层糕就相当于一个Layer，干果就好比是Layer里的各个文件。但如果某两层的同一个位置都有干果，也就是有文件同名，那么我们就只能看到上层的文件，而下层的就被屏蔽了。</p><p>你可以用命令 <code>docker inspect</code> 来查看镜像的分层信息，比如nginx:alpine镜像：</p><pre><code class="language-plain">docker inspect nginx:alpine
</code></pre><p>它的分层信息在“RootFS”部分：<br>
<img src="https://static001.geekbang.org/resource/image/5y/b7/5yybd821a12ec1323f6ea8bb5a5c4ab7.png?wh=1920x592" alt="图片"></p><p>通过这张截图就可以看到，nginx:alpine镜像里一共有6个Layer。</p><p>相信你现在也就明白，之前在使用 <code>docker pull</code>、<code>docker rmi</code> 等命令操作镜像的时候，那些“奇怪”的输出信息是什么了，其实就是镜像里的各个Layer。Docker会检查是否有重复的层，如果本地已经存在就不会重复下载，如果层被其他镜像共享就不会删除，这样就可以节约磁盘和网络成本。</p><h2>Dockerfile是什么</h2><p>知道了容器镜像的内部结构和基本原理，我们就可以来学习如何自己动手制作容器镜像了，也就是自己打包应用。</p><p>在之前我们讲容器的时候，曾经说过容器就是“小板房”，镜像就是“样板间”。那么，要造出这个“样板间”，就必然要有一个“施工图纸”，由它来规定如何建造地基、铺设水电、开窗搭门等动作。这个“施工图纸”就是“<strong>Dockerfile</strong>”。</p><p>比起容器、镜像来说，Dockerfile非常普通，它就是一个纯文本，里面记录了一系列的构建指令，比如选择基础镜像、拷贝文件、运行脚本等等，每个指令都会生成一个Layer，而Docker顺序执行这个文件里的所有步骤，最后就会创建出一个新的镜像出来。</p><p>我们来看一个最简单的Dockerfile实例：</p><pre><code class="language-plain"># Dockerfile.busybox
FROM busybox                  # 选择基础镜像
CMD echo "hello world"        # 启动容器时默认运行的命令
</code></pre><p>这个文件里只有两条指令。</p><p>第一条指令是 <code>FROM</code>，所有的Dockerfile都要从它开始，表示选择构建使用的基础镜像，相当于“打地基”，这里我们使用的是busybox。</p><p>第二条指令是 <code>CMD</code>，它指定 <code>docker run</code> 启动容器时默认运行的命令，这里我们使用了echo命令，输出“hello world”字符串。</p><p>现在有了Dockerfile这张“施工图纸”，我们就可以请出“施工队”了，用 <code>docker build</code> 命令来创建出镜像：</p><pre><code class="language-plain">docker build -f Dockerfile.busybox .

Sending build context to Docker daemon&nbsp; &nbsp;7.68kB
Step 1/2 : FROM busybox
&nbsp;---&gt; d38589532d97
Step 2/2 : CMD echo "hello world"
&nbsp;---&gt; Running in c5a762edd1c8
Removing intermediate container c5a762edd1c8
&nbsp;---&gt; b61882f42db7
Successfully built b61882f42db7
</code></pre><p>你需要特别注意命令的格式，用 <code>-f</code> 参数指定Dockerfile文件名，后面必须跟一个文件路径，叫做“<strong>构建上下文</strong>”（build’s context），这里只是一个简单的点号，表示当前路径的意思。</p><p>接下来，你就会看到Docker会逐行地读取并执行Dockerfile里的指令，依次创建镜像层，再生成完整的镜像。</p><p>新的镜像暂时还没有名字（用 <code>docker images</code> 会看到是 <code>&lt;none&gt;</code>），但我们可以直接使用“IMAGE ID”来查看或者运行：</p><pre><code class="language-plain">docker inspect b61
docker run b61
</code></pre><h2>怎样编写正确、高效的Dockerfile</h2><p>大概了解了Dockerfile之后，我再来讲讲编写Dockerfile的一些常用指令和最佳实践，帮你在今后的工作中把它写好、用好。</p><p>首先因为构建镜像的第一条指令必须是 <code>FROM</code>，所以基础镜像的选择非常关键。如果关注的是镜像的安全和大小，那么一般会选择Alpine；如果关注的是应用的运行稳定性，那么可能会选择Ubuntu、Debian、CentOS。</p><pre><code class="language-plain">FROM alpine:3.15                # 选择Alpine镜像
FROM ubuntu:bionic              # 选择Ubuntu镜像
</code></pre><p>我们在本机上开发测试时会产生一些源码、配置等文件，需要打包进镜像里，这时可以使用 <code>COPY</code> 命令，它的用法和Linux的cp差不多，不过拷贝的源文件必须是“<strong>构建上下文</strong>”路径里的，不能随意指定文件。也就是说，如果要从本机向镜像拷贝文件，就必须把这些文件放到一个专门的目录，然后在 <code>docker build</code> 里指定“构建上下文”到这个目录才行。</p><p>这里有两个 <code>COPY</code> 命令示例，你可以看一下：</p><pre><code class="language-plain">COPY ./a.txt  /tmp/a.txt    # 把构建上下文里的a.txt拷贝到镜像的/tmp目录
COPY /etc/hosts  /tmp       # 错误！不能使用构建上下文之外的文件
</code></pre><p>接下来要说的就是Dockerfile里最重要的一个指令 <code>RUN</code> ，它可以执行任意的Shell命令，比如更新系统、安装应用、下载文件、创建目录、编译程序等等，实现任意的镜像构建步骤，非常灵活。</p><p><code>RUN</code> 通常会是Dockerfile里最复杂的指令，会包含很多的Shell命令，但Dockerfile里一条指令只能是一行，所以有的 <code>RUN</code> 指令会在每行的末尾使用续行符 <code>\</code>，命令之间也会用 <code>&amp;&amp;</code> 来连接，这样保证在逻辑上是一行，就像下面这样：</p><pre><code class="language-plain">RUN apt-get update \
&nbsp; &nbsp; &amp;&amp; apt-get install -y \
&nbsp; &nbsp; &nbsp; &nbsp; build-essential \
&nbsp; &nbsp; &nbsp; &nbsp; curl \
&nbsp; &nbsp; &nbsp; &nbsp; make \
&nbsp; &nbsp; &nbsp; &nbsp; unzip \
&nbsp; &nbsp; &amp;&amp; cd /tmp \
&nbsp; &nbsp; &amp;&amp; curl -fSL xxx.tar.gz -o xxx.tar.gz\
&nbsp; &nbsp; &amp;&amp; tar xzf xxx.tar.gz \
&nbsp; &nbsp; &amp;&amp; cd xxx \
&nbsp; &nbsp; &amp;&amp; ./config \
&nbsp; &nbsp; &amp;&amp; make \
    &amp;&amp; make clean
</code></pre><p>有的时候在Dockerfile里写这种超长的 <code>RUN</code> 指令很不美观，而且一旦写错了，每次调试都要重新构建也很麻烦，所以你可以采用一种变通的技巧：<strong>把这些Shell命令集中到一个脚本文件里，用 <code>COPY</code> 命令拷贝进去再用 <code>RUN</code> 来执行</strong>：</p><pre><code class="language-plain">COPY setup.sh  /tmp/                # 拷贝脚本到/tmp目录

RUN cd /tmp &amp;&amp; chmod +x setup.sh \  # 添加执行权限
    &amp;&amp; ./setup.sh &amp;&amp; rm setup.sh    # 运行脚本然后再删除
</code></pre><p><code>RUN</code> 指令实际上就是Shell编程，如果你对它有所了解，就应该知道它有变量的概念，可以实现参数化运行，这在Dockerfile里也可以做到，需要使用两个指令 <code>ARG</code> 和<code> ENV</code>。</p><p><strong>它们区别在于 <code>ARG</code> 创建的变量只在镜像构建过程中可见，容器运行时不可见，而 <code>ENV</code> 创建的变量不仅能够在构建镜像的过程中使用，在容器运行时也能够以环境变量的形式被应用程序使用。</strong></p><p>下面是一个简单的例子，使用 <code>ARG</code> 定义了基础镜像的名字（可以用在“FROM”指令里），使用 <code>ENV</code> 定义了两个环境变量：</p><pre><code class="language-plain">ARG IMAGE_BASE="node"
ARG IMAGE_TAG="alpine"

ENV PATH=$PATH:/tmp
ENV DEBUG=OFF
</code></pre><p>还有一个重要的指令是 <code>EXPOSE</code>，它用来声明容器对外服务的端口号，对现在基于Node.js、Tomcat、Nginx、Go等开发的微服务系统来说非常有用：</p><pre><code class="language-plain">EXPOSE 443           # 默认是tcp协议
EXPOSE 53/udp        # 可以指定udp协议
</code></pre><p>讲了这些Dockerfile指令之后，我还要特别强调一下，因为每个指令都会生成一个镜像层，所以Dockerfile里最好不要滥用指令，尽量精简合并，否则太多的层会导致镜像臃肿不堪。</p><h2>docker build是怎么工作的</h2><p>Dockerfile必须要经过 <code>docker build</code> 才能生效，所以我们再来看看 <code>docker build</code> 的详细用法。</p><p>刚才在构建镜像的时候，你是否对“构建上下文”这个词感到有些困惑呢？它到底是什么含义呢？</p><p>我觉得用Docker的官方架构图来理解会比较清楚（注意图中与“docker build”关联的虚线）。</p><p>因为命令行“docker”是一个简单的客户端，真正的镜像构建工作是由服务器端的“Docker daemon”来完成的，所以“docker”客户端就只能把“构建上下文”目录打包上传（显示信息 <code>Sending build context to Docker daemon</code> ），这样服务器才能够获取本地的这些文件。</p><p><img src="https://static001.geekbang.org/resource/image/c8/fe/c8116066bdbf295a7c9fc25b87755dfe.jpg?wh=1920x1048" alt="图片"></p><p>明白了这一点，你就会知道，“构建上下文”其实与Dockerfile并没有直接的关系，它其实指定了要打包进镜像的一些依赖文件。而 <code>COPY</code> 命令也只能使用基于“构建上下文”的相对路径，因为“Docker daemon”看不到本地环境，只能看到打包上传的那些文件。</p><p>但这个机制也会导致一些麻烦，如果目录里有的文件（例如readme/.git/.svn等）不需要拷贝进镜像，docker也会一股脑地打包上传，效率很低。</p><p>为了避免这种问题，你可以在“构建上下文”目录里再建立一个 <code>.dockerignore</code> 文件，语法与 <code>.gitignore</code> 类似，排除那些不需要的文件。</p><p>下面是一个简单的示例，表示不打包上传后缀是“swp”“sh”的文件：</p><pre><code class="language-plain"># docker ignore
*.swp
*.sh
</code></pre><p>另外关于Dockerfile，一般应该在命令行里使用 <code>-f</code> 来显式指定。但如果省略这个参数，<code>docker build</code> 就会在当前目录下找名字是 <code>Dockerfile</code> 的文件。所以，如果只有一个构建目标的话，文件直接叫“Dockerfile”是最省事的。</p><p>现在我们使用 <code>docker build</code> 应该就没什么难点了，不过构建出来的镜像只有“IMAGE ID”没有名字，不是很方便。</p><p>为此你可以加上一个 <code>-t</code> 参数，也就是指定镜像的标签（tag），这样Docker就会在构建完成后自动给镜像添加名字。当然，名字必须要符合上节课里的命名规范，用 <code>:</code> 分隔名字和标签，如果不提供标签默认就是“latest”。</p><h2>小结</h2><p>好了，今天我们一起学习了容器镜像的内部结构，重点理解<strong>容器镜像是由多个只读的Layer构成的，同一个Layer可以被不同的镜像共享</strong>，减少了存储和传输的成本。</p><p>如何编写Dockerfile内容稍微多一点，我再简单做个小结：</p><ol>
<li>创建镜像需要编写Dockerfile，写清楚创建镜像的步骤，每个指令都会生成一个Layer。</li>
<li>Dockerfile里，第一个指令必须是 <code>FROM</code>，用来选择基础镜像，常用的有Alpine、Ubuntu等。其他常用的指令有：<code>COPY</code>、<code>RUN</code>、<code>EXPOSE</code>，分别是拷贝文件，运行Shell命令，声明服务端口号。</li>
<li><code>docker build</code> 需要用 <code>-f</code> 来指定Dockerfile，如果不指定就使用当前目录下名字是“Dockerfile”的文件。</li>
<li><code>docker build</code> 需要指定“构建上下文”，其中的文件会打包上传到Docker daemon，所以尽量不要在“构建上下文”中存放多余的文件。</li>
<li>创建镜像的时候应当尽量使用 <code>-t</code> 参数，为镜像起一个有意义的名字，方便管理。</li>
</ol><p>今天讲了不少，但关于创建镜像还有很多高级技巧等待你去探索，比如使用缓存、多阶段构建等等，你可以再参考Docker官方文档（<a href="https://docs.docker.com/engine/reference/builder/">https://docs.docker.com/engine/reference/builder/</a>），或者一些知名应用的镜像（如Nginx、Redis、Node.js等）进一步学习。</p><h2>课下作业</h2><p>最后是课下作业时间，这里有一个完整的Dockerfile示例，你可以尝试着去解释一下它的含义，然后再自己构建一下：</p><pre><code class="language-plain"># Dockerfile
# docker build -t ngx-app .
# docker build -t ngx-app:1.0 .

ARG IMAGE_BASE="nginx"
ARG IMAGE_TAG="1.21-alpine"

FROM ${IMAGE_BASE}:${IMAGE_TAG}

COPY ./default.conf /etc/nginx/conf.d/

RUN cd /usr/share/nginx/html \
&nbsp; &nbsp; &amp;&amp; echo "hello nginx" &gt; a.txt

EXPOSE 8081 8082 8083
</code></pre><p>当然还有两个思考题：</p><ol>
<li>镜像里的层都是只读不可修改的，但容器运行的时候经常会写入数据，这个冲突应该怎么解决呢？（答案在本期找）</li>
<li>你能再列举一下镜像的分层结构带来了哪些好处吗？</li>
</ol><p>欢迎积极留言。如果你觉得有收获，也欢迎分享给身边的朋友同事一起讨论学习。</p><p><img src="https://static001.geekbang.org/resource/image/17/24/1705133103a8aaf6c7fed770afa6dc24.jpg?wh=1920x2805" alt=""></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/23/52/66/3e4d4846.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>includestdio.h</span>
  </div>
  <div class="_2_QraFYR_0">关于指令生成层的问题需要再补充哈：只有 RUN, COPY, ADD 会生成新的镜像层，其它指令只会产生临时层，不影响构建大小，官网的镜像构建最佳实践里面有提及<br><br>https:&#47;&#47;docs.docker.com&#47;develop&#47;develop-images&#47;dockerfile_best-practices&#47;<br>Only the instructions RUN, COPY, ADD create layers. Other instructions create temporary intermediate images, and do not increase the size of the build.</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: many thanks。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-29 16:26:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/9c/fb/7fe6df5b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>陈卧虫</span>
  </div>
  <div class="_2_QraFYR_0">1. 创建和修改文件：通过在写时复制实现；删除文件：通过白障实现，也就是通过一个文件记录已经被删除的文件。<br>2. 镜像分层的好处：可以重复使用未被改动的layer，每次修改打包镜像，只需重新构建被改动的部分</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-29 17:12:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/42/18/edc1b373.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>风飘，吾独思</span>
  </div>
  <div class="_2_QraFYR_0">1.容器最上一层是读写层，镜像所有的层是只读层。容器启动后，Docker daemon会在容器的镜像上添加一个读写层。<br>2.容器分层可以共享资源，节约空间，相同的内容只需要加载一份份到内存。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-29 09:03:57</div>
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
  <div class="_2_QraFYR_0">请问下docker pull时输出的layer的值为什么和docker inspect里面layer信息的sha256的值不一样呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: docker pull的layer是压缩数据的sha256，docker inspect是解压后数据的sha256。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-19 23:26:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/9b/0f/ef415e32.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>CK</span>
  </div>
  <div class="_2_QraFYR_0">还是没太理解构建上下文的意思，是指docker build的时候指定路径？比如文中示例docker build -f Dockerfile.busybox .  是一个.表示，我执行文末的课下作业时，显示COPY failed: file not found in build context or excluded by .dockerignore: stat default.conf: file does not exist，是不是docker build时的路径没指定好呢<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: “上下文”这个词确实很难理解，最好直接用英文context，就是要打包进镜像的文件所在的目录。<br><br>如果copy时在这个目录里找不到文件，当然就会报错了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-30 00:28:11</div>
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
  <div class="_2_QraFYR_0">ENTRYPOINT 和 CMD 的本质区别是什么的？ 什么时候用 ENTRYPOINT 什么时候用 CMD？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 看小贴士应该就能够理解它们两个的区别了，ENTRYPOINT是执行的命令， CMD是参数。<br><br>感觉这应该是docker当初的设计小失误，其实用哪个没有什么太严格的区分。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-29 23:03:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTL6RvdCzKZCGibZqanPxlib453uP5oXvMTjR6uJsjfMZsib5ShMicDdgBUr6yHSibSKSKgiazqR6tNibDibibQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_666217</span>
  </div>
  <div class="_2_QraFYR_0">构建包如果出错了，注意注释和内容不要再同一行，会将注释视为参数<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，Dockerfile的注释比较特殊，必须的单独一行，不能在行尾。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-29 17:38:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/1e/db/921367d6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jone</span>
  </div>
  <div class="_2_QraFYR_0">【课下作业】 ，用docker build打好镜像后，即使使用-d，也是会自动退出。如果想要看容器里面刚刚copy和生成的a.txt文件，只能采用： docker run -it ngx-app:1.0 sh ，这样就进入到容器里面了，当然exits之后，就会退出容器了。  老师，这个要怎么理解呢。即使用了-d，也是一次性执行容器。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个要看镜像里是什么了，如果是Nginx、Redis这样的会自动在后台运行，如果是shell脚本那么启动后会立即结束。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-01 00:36:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83er3Ey0Uq2w4wLhVNGGReZKLd06PCDU4ZeefZWFMNvf3LibtibDqzBBpzkYW5n5AkYuJGPa4KEdQ5qgA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_f20da9</span>
  </div>
  <div class="_2_QraFYR_0">老师有时间帮忙看一下，不知道写的对不对。<br><br>dockerfile常用参数：<br>1.ARG：镜像层的环境变量<br>2.FROM：拉取基础镜像<br>3.COPY:拷贝文件<br>4.ADD：拷贝文件、URL、压缩文件等<br>5.EVN：镜像层和容器层参数<br>6.EXPOSE:暴露容器内部端口给外部使用<br>7.RUN：执行shell指令<br>8.CMD：构建完成时执行的指令<br><br>思考题：<br>1.docker采用UNION FS文件系统，将文件系统分为上层和下层。即上层为容器层，下层为镜像层。如果下层有修改，运行容器时，上层会同步修改。如果上层有数据修改（即容器层数据修改），不会影响到下层（即镜像层）。<br>2.好处：共享已存在的layer，如果有新的数据加入，只会增量在最上层新增layer层。减少了网络传输等一些成本。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 写的很好。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-17 23:16:10</div>
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
  <div class="_2_QraFYR_0">1: 构建上下文使用相对路径，是不是当前就是顶层，限制通过 ..&#47; 的方式去到构建上下文的上一层<br><br>2. run的指令会生成新的一层，但是如果 【run之前的指令不一样，例如from】【run里面的东西太多】，是不是大部分run生成的这一层都不能重用呢？这一块比较好奇<br><br>3. build后的名字为  =&gt; =&gt; naming to docker.io&#47;library&#47;homework。      前面的这个docker.Io&#47;library 这个是默认的吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.不可以，因为context会被打包发送给daemon，它就是根。<br><br>2.这个就是docker的build cache，可以再查相关的资料。<br><br>3.是的，默认就是docker hub的library名字，后面会讲。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-22 17:43:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/27/52/40/db9b0eb2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>自由</span>
  </div>
  <div class="_2_QraFYR_0">罗老师，我练习了结尾的作业，有一个问题。为什么构建出来的 ngx-app ，通过 docker inspect xxx 查看，它的 Layers 有 8 层呢？在我看来 Dockerfile 只有 7 个命令。还有一个问题，FROM 是否会构建一个层？如果会，为什么之前的 docker build -f Dockerfile.busybox . 的 image 产物通过 docker inspect 查看只有一层 Layer 呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.镜像会包含基础镜像里的层。<br><br>2.具体层数是如何计算的可能得深入去看docker的内部了，我觉得暂时可以不关心，把docker、Kubernetes都学完了，有必要再深究。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-07 19:14:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2e/4e/35/d5d1aec8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>九暮</span>
  </div>
  <div class="_2_QraFYR_0">RUN cd &#47;usr&#47;share&#47;nginx&#47;html \    &amp;&amp; echo &quot;hello nginx&quot; &gt; a.txt<br>老师，这句话在容器执行，我进入到容器内部结果没有保存下来呢，没有这个文件<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 肯定是有的，Dockerfile里的指令在build的时候就生成在镜像里了，看看目录对不对。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-03 19:22:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/b8/6a/d37ed9ab.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>今天你学习了吗？</span>
  </div>
  <div class="_2_QraFYR_0">没看到说cow的方法呀，这个从哪说到了呢<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 正文里没有写，其实这个也不是需要特别了解的知识点，不影响我们的使用。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-01 00:14:46</div>
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
  <div class="_2_QraFYR_0">老师有个问题 层数多少会影响镜像总大小吗 还是不影响 </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 当然了，每个层都会有一点空间占用，积累多了镜像就会越来越大。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-30 21:56:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/83/af/1cb42cd3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>马以</span>
  </div>
  <div class="_2_QraFYR_0">1: docker在挂载镜像文件的时候除了镜像文件的只读层，还会挂载一个‘可读写层’，在容器运行是，它以copy-on-write的方式，记录容器中的“写”操作；<br>2: a: 镜像分层的好处在于他可以减少制作成本，从而迅速迭代，它使得我们能以一种增量的方式对现有已经存在的镜像做改造，而不是每一次都从0开始重复制作，<br>        降低了技术人员之间的操作成本；<br>    b: 占用空间更下，每次拉取和推送都只操作增量的部分，省时省力；</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: awesome</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-29 21:13:26</div>
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
  <div class="_2_QraFYR_0">作业：<br>1.写的时候往上叠加一层<br>2.避免重复打包<br><br>课后思考：<br>万一打包层数过高怎么办？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 差不多，可以再参考其他同学的回答。<br><br>层数太多就会导致镜像加载效率低，所以要优化Dockerfile。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-29 19:45:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/64/4c/abb7bfe3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>萝卜头王</span>
  </div>
  <div class="_2_QraFYR_0">文章中的dockerfile里面的注释信息(以#开头的内容)，是不是应该放在单独的一行？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，Dockerfile的注释必须是#开头，在文档里只是示意性质的说明。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-29 10:04:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/7e/25/3932dafd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>GeekNEO</span>
  </div>
  <div class="_2_QraFYR_0">default.conf只要在构建的临时目录中存在就可以，不会检查这个文件内容是否真的正确对吧，是否是不是nginx的正确语法配置文件？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: docker build只是打包文件，它不会也无法检查内容是否正确，不过我们也可以写脚本，在run里执行校验检查。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-13 11:46:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/20/27/a6932fbe.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>虢國技醬</span>
  </div>
  <div class="_2_QraFYR_0">&quot;Dockerfile 里，第一个指令必须是 FROM，用来选择基础镜像&quot;<br><br>一直有个疑问，写Dockerfile都必须有个基础镜像，那么依赖的这些基础镜像 的最原始镜像是怎么制作的？<br><br>这里研究了一下文档，同步给大家：<br><br>https:&#47;&#47;docs.docker.com&#47;build&#47;building&#47;base-images&#47;#create-a-simple-parent-image-using-scratch<br><br>两个方式：<br><br>1、Create a full image using tar<br><br>2、Create a simple parent image using scratch<br><br>特别是 scratch 这个：<br><br>You can use Docker’s reserved, minimal image, scratch, as a starting point for building containers. Using the scratch “image” signals to the build process that you want the next command in the Dockerfile to be the first filesystem layer in your image.<br><br>While scratch appears in Docker’s repository on the hub, you can’t pull it, run it, or tag any image with the name scratch. Instead, you can refer to it in your Dockerfile.</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-10 12:24:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jingtong</span>
  </div>
  <div class="_2_QraFYR_0">虽然这些文件里不包含系统内核（因为容器共享了宿主机的内核），但如果每个镜像都重复做这样的打包操作，仍然会导致大量的冗余。<br><br>-----<br><br>共享宿主机内核这里如果是Mac上面跑Linux，这个没法共享内核吧，包括不同版本的内核也没法共享？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: docker的运行原理就是共享内核，mac上的Linux是在虚拟机里，用的还是Linux内核。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-05 16:52:12</div>
  </div>
</div>
</div>
</li>
</ul>