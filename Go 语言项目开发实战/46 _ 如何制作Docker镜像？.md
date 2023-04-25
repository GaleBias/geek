<audio title="46 _ 如何制作Docker镜像？" src="https://static001.geekbang.org/resource/audio/47/93/4719e8f7b06362279c0b66ff39df0793.mp3" controls="controls"></audio> 
<p>你好，我是孔令飞。</p><p>要落地云原生架构，其中的一个核心点是通过容器来部署我们的应用。如果要使用容器来部署应用，那么制作应用的Docker镜像就是我们绕不开的关键一步。今天，我就来详细介绍下如何制作Docker镜像。</p><p>在这一讲中，我会先讲解下Docker镜像的构建原理和方式，然后介绍Dockerfile的指令，以及如何编写Dockerfile文件。最后，介绍下编写Dockerfile文件时要遵循的一些最佳实践。</p><h2>Docker镜像的构建原理和方式</h2><p>首先，我们来看下Docker镜像构建的原理和方式。</p><p>我们可以用多种方式来构建一个Docker镜像，最常用的有两种：</p><ul>
<li>通过<code>docker commit</code>命令，基于一个已存在的容器构建出镜像。</li>
<li>编写Dockerfile文件，并使用<code>docker build</code>命令来构建镜像。</li>
</ul><p>上面这两种方法中，镜像构建的底层原理是相同的，都是通过下面3个步骤来构建镜像：</p><ol>
<li>基于原镜像，启动一个Docker容器。</li>
<li>在容器中进行一些操作，例如执行命令、安装文件等。由这些操作产生的文件变更都会被记录在容器的存储层中。</li>
<li>将容器存储层的变更commit到新的镜像层中，并添加到原镜像上。</li>
</ol><p>下面，我们来具体讲解这两种构建Docker镜像的方式。</p><!-- [[[read_end]]] --><h3>通过<code>docker commit</code>命令构建镜像</h3><p>我们可以通过<code>docker commit</code>来构建一个镜像，命令的格式为<code>docker commit [选项] [&lt;仓库名&gt;[:&lt;标签&gt;]]</code>。</p><p>下图中，我们通过4个步骤构建了Docker镜像<code>ccr.ccs.tencentyun.com/marmotedu/iam-apiserver-amd64:test</code>：</p><p><img src="https://static001.geekbang.org/resource/image/23/11/23619b513ff043792c7374acf7781d11.png?wh=1920x344" alt="图片"></p><p>具体步骤如下：</p><ol>
<li>执行<code>docker ps</code>获取需要构建镜像的容器ID <code>48d1dbb89a7f</code>。</li>
<li>执行<code>docker pause 48d1dbb89a7f</code>暂停<code>48d1dbb89a7f</code>容器的运行。</li>
<li>执行<code>docker commit 48d1dbb89a7f ccr.ccs.tencentyun.com/marmotedu/iam-apiserver-amd64:test</code>，基于容器ID <code>48d1dbb89a7f</code>构建Docker镜像。</li>
<li>执行<code>docker images ccr.ccs.tencentyun.com/marmotedu/iam-apiserver-amd64:test</code>，查看镜像是否成功构建。</li>
</ol><p>这种镜像构建方式通常用在下面两个场景中：</p><ul>
<li>构建临时的测试镜像；</li>
<li>容器被入侵后，使用<code>docker commit</code>，基于被入侵的容器构建镜像，从而保留现场，方便以后追溯。</li>
</ul><p>除了这两种场景，我不建议你使用<code>docker commit</code>来构建生产现网环境的镜像。我这么说的主要原因有两个：</p><ul>
<li>使用<code>docker commit</code>构建的镜像包含了编译构建、安装软件，以及程序运行产生的大量无用文件，这会导致镜像体积很大，非常臃肿。</li>
<li>使用<code>docker commit</code>构建的镜像会丢失掉所有对该镜像的操作历史，无法还原镜像的构建过程，不利于镜像的维护。</li>
</ul><p>下面，我们再来看看如何使用<code>Dockerfile</code>来构建镜像。</p><h3>通过<code>Dockerfile</code>来构建镜像</h3><p>在实际开发中，使用<code>Dockerfile</code>来构建是最常用，也最标准的镜像构建方法。<code>Dockerfile</code>是Docker用来构建镜像的文本文件，里面包含了一系列用来构建镜像的指令。</p><p><code>docker build</code>命令会读取<code>Dockerfile</code>的内容，并将<code>Dockerfile</code>的内容发送给Docker引擎，最终Docker引擎会解析<code>Dockerfile</code>中的每一条指令，构建出需要的镜像。</p><p><code>docker build</code>的命令格式为<code>docker build [OPTIONS] PATH | URL | -</code>。<code>PATH</code>、<code>URL</code>、<code>-</code>指出了构建镜像的上下文（context），context中包含了构建镜像需要的<code>Dockerfile</code>文件和其他文件。默认情况下，Docker构建引擎会查找context中名为<code>Dockerfile</code>的文件，但你可以通过<code>-f, --file</code>选项，手动指定<code>Dockerfile</code>文件。例如：</p><pre><code class="language-bash"> $ docker build -f Dockerfile -t ccr.ccs.tencentyun.com/marmotedu/iam-apiserver-amd64:test .
</code></pre><p>使用Dockerfile构建镜像，本质上也是通过镜像创建容器，并在容器中执行相应的指令，然后停止容器，提交存储层的文件变更。和用<code>docker commit</code>构建镜像的方式相比，它有三个好处：</p><ul>
<li>Dockerfile 包含了镜像制作的完整操作流程，其他开发者可以通过 Dockerfile 了解并复现制作过程。</li>
<li>Dockerfile 中的每一条指令都会创建新的镜像层，这些镜像可以被 Docker Daemnon 缓存。再次制作镜像时，Docker 会尽量复用缓存的镜像层（using cache），而不是重新逐层构建，这样可以节省时间和磁盘空间。</li>
<li>Dockerfile 的操作流程可以通过<code>docker image history [镜像名称]</code>查询，方便开发者查看变更记录。</li>
</ul><p>这里，我们通过一个示例，来详细介绍下通过<code>Dockerfile</code>构建镜像的流程。</p><p><strong>首先，</strong>我们需要编写一个<code>Dockerfile</code>文件。下面是iam-apiserver的<a href="https://github.com/marmotedu/iam/blob/v1.0.8/build/docker/iam-apiserver/Dockerfile">Dockerfile</a>文件内容：</p><pre><code class="language-dockerfile">FROM centos:centos8
LABEL maintainer="&lt;colin404@foxmail.com&gt;"

RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
RUN echo "Asia/Shanghai" &gt; /etc/timezone

WORKDIR /opt/iam
COPY iam-apiserver /opt/iam/bin/

ENTRYPOINT ["/opt/iam/bin/iam-apiserver"]
</code></pre><p>这里选择<code>centos:centos8</code>作为基础镜像，是因为<code>centos:centos8</code>镜像中包含了基本的排障工具，例如<code>vi</code>、<code>cat</code>、<code>curl</code>、<code>mkdir</code>、<code>cp</code>等工具。</p><p><strong>接着，</strong>执行<code>docker build</code>命令来构建镜像：</p><pre><code class="language-bash">$ docker build -f Dockerfile -t ccr.ccs.tencentyun.com/marmotedu/iam-apiserver-amd64:test .
</code></pre><p>执行<code>docker build</code>后的构建流程为：</p><p><strong>第一步</strong>，<code>docker build</code>会将context中的文件打包传给Docker daemon。如果context中有<code>.dockerignore</code>文件，则会从上传列表中删除满足<code>.dockerignore</code>规则的文件。</p><p>这里有个例外，如果<code>.dockerignore</code>文件中有<code>.dockerignore</code>或者<code>Dockerfile</code>，<code>docker build</code>命令在排除文件时会忽略掉这两个文件。如果指定了镜像的tag，还会对repository和tag进行验证。</p><p><strong>第二步</strong>，<code>docker build</code>命令向Docker server发送HTTP请求，请求Docker server构建镜像，请求中包含了需要的context信息。</p><p><strong>第三步</strong>，Docker server接收到构建请求之后，会执行以下流程来构建镜像：</p><ol>
<li>创建一个临时目录，并将context中的文件解压到该目录下。</li>
<li>读取并解析Dockerfile，遍历其中的指令，根据命令类型分发到不同的模块去执行。</li>
<li>Docker构建引擎为每一条指令创建一个临时容器，在临时容器中执行指令，然后commit容器，生成一个新的镜像层。</li>
<li>最后，将所有指令构建出的镜像层合并，形成build的最后结果。最后一次commit生成的镜像ID就是最终的镜像ID。</li>
</ol><p>为了提高构建效率，<code>docker build</code>默认会缓存已有的镜像层。如果构建镜像时发现某个镜像层已经被缓存，就会直接使用该缓存镜像，而不用重新构建。如果不希望使用缓存的镜像，可以在执行<code>docker build</code>命令时，指定<code>--no-cache=true</code>参数。</p><p>Docker匹配缓存镜像的规则为：遍历缓存中的基础镜像及其子镜像，检查这些镜像的构建指令是否和当前指令完全一致，如果不一样，则说明缓存不匹配。对于<code>ADD</code>、<code>COPY</code>指令，还会根据文件的校验和（checksum）来判断添加到镜像中的文件是否相同，如果不相同，则说明缓存不匹配。</p><p>这里要注意，缓存匹配检查不会检查容器中的文件。比如，当使用<code>RUN apt-get -y update</code>命令更新了容器中的文件时，缓存策略并不会检查这些文件，来判断缓存是否匹配。</p><p>最后，我们可以通过<code>docker history</code>命令来查看镜像的构建历史，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/f3/06/f32a0e2f90a36995a561747519897806.png?wh=1854x345" alt="图片"></p><h3>其他制作镜像方式</h3><p>上面介绍的是两种最常用的镜像构建方式，还有一些其他的镜像创建方式，这里我简单介绍两种。</p><ol>
<li>通过<code>docker save</code>和<code>docker load</code>命令构建</li>
</ol><p><code>docker save</code>用来将镜像保存为一个tar文件，<code>docker load</code>用来将tar格式的镜像文件加载到当前机器上，例如：</p><pre><code class="language-bash"># 在 A 机器上执行，并将 nginx-v1.0.0.tar.gz 复制到 B 机器
$ docker save nginx | gzip &gt; nginx-v1.0.0.tar.gz

# 在 B 机器上执行
$ docker load -i nginx-v1.0.0.tar.gz
</code></pre><p>通过上面的命令，我们就在机器B上创建了<code>nginx</code>镜像。</p><ol start="2">
<li>通过<code>docker export</code>和<code>docker import</code>命令构建</li>
</ol><p>我们先通过<code>docker export</code> 保存镜像，再通过<code>docker import</code> 加载镜像，具体命令如下：</p><pre><code class="language-bash"># 在 A 机器上执行，并将 nginx-v1.0.0.tar.gz 复制到 B 机器
$ docker export nginx &gt; nginx-v1.0.0.tar.gz

# 在 B 机器上执行
$ docker import - nginx:v1.0.0 nginx-v1.0.0.tar.gz
</code></pre><p>通过<code>docker export</code>导出的镜像和通过<code>docker save</code>保存的镜像相比，会丢失掉所有的镜像构建历史。在实际生产环境中，我不建议你通过<code>docker save</code>和<code>docker export</code>这两种方式来创建镜像。我比较推荐的方式是：在A机器上将镜像push到镜像仓库，在B机器上从镜像仓库pull该镜像。</p><h2>Dockerfile指令介绍</h2><p>上面，我介绍了一些与Docker镜像构建有关的基础知识。在实际生产环境中，我们标准的做法是通过Dockerfile来构建镜像，这就要求你会编写Dockerfile文件。接下来，我就详细介绍下如何编写Dockerfile文件。</p><p>Dockerfile指令的基本格式如下：</p><pre><code class="language-dockerfile"># Comment
INSTRUCTION arguments
</code></pre><p><code>INSTRUCTION</code>是指令，不区分大小写，但我的建议是指令都大写，这样可以与参数进行区分。Dockerfile中，以 <code>#</code> 开头的行是注释，而在其他位置出现的 <code>#</code> 会被当成参数，例如：</p><pre><code class="language-dockerfile"># Comment
RUN echo 'hello world # dockerfile'
</code></pre><p>一个Dockerfile文件中包含了多条指令，这些指令可以分为5类。</p><ul>
<li>定义基础镜像的指令：<strong>FROM</strong>；</li>
<li>定义镜像维护者的指令：<strong>MAINTAINER</strong>（可选）；</li>
<li>定义镜像构建过程的指令：<strong>COPY</strong>、ADD、<strong>RUN</strong>、USER、<strong>WORKDIR</strong>、ARG、<strong>ENV</strong>、VOLUME、<strong>ONBUILD</strong>；</li>
<li>定义容器启动时执行命令的指令：<strong>CMD</strong>、<strong>ENTRYPOINT</strong>；</li>
<li>其他指令：EXPOSE、HEALTHCHECK、STOPSIGNAL。</li>
</ul><p>其中，加粗的指令是编写Dockerfile时经常用到的指令，需要你重点了解下。我把这些常用Dockerfile指令的介绍放在了GitHub上，你可以看看这个<a href="https://github.com/marmotedu/geekbang-go/blob/master/Dockerfile%E6%8C%87%E4%BB%A4%E8%AF%A6%E8%A7%A3.md">Dockerfile指令详解</a>。</p><p>下面是一个Dockerfile示例：</p><pre><code class="language-dockerfile"># 第一行必须指定构建该镜像所基于的容器镜像
FROM centos:centos8

# 维护者信息
MAINTAINER Lingfei Kong &lt;colin404@foxmail.com&gt;

# 镜像的操作指令
RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
RUN echo "Asia/Shanghai" &gt; /etc/timezone
WORKDIR /opt/iam
COPY iam-apiserver /opt/iam/bin/

# 容器启动时执行指令
ENTRYPOINT ["/opt/iam/bin/iam-apiserver"]
</code></pre><p>Docker会顺序解释并执行Dockerfile中的指令，并且第一条指令必须是<code>FROM</code>，<code>FROM</code> 用来指定构建镜像的基础镜像。接下来，一般会指定镜像维护者的信息。后面是镜像操作的指令，最后会通过<code>CMD</code>或者<code>ENTRYPOINT</code>来指定容器启动的命令和参数。</p><h2>Dockerfile最佳实践</h2><p>上面我介绍了Dockerfile的指令，但在编写Dockerfile时，只知道这些指令是不够的，还不能编写一个合格的Dockerfile。我们还需要遵循一些编写 Dockerfile的最佳实践。这里，我总结了一份编写 Dockerfile的最佳实践清单，你可以参考。</p><ol>
<li>
<p>建议所有的Dockerfile指令大写，这样做可以很好地跟在镜像内执行的指令区分开来。</p>
</li>
<li>
<p>在选择基础镜像时，尽量选择官方的镜像，并在满足要求的情况下，尽量选择体积小的镜像。目前，Linux镜像大小有以下关系：<code>busybox &lt; debian &lt; centos &lt; ubuntu</code>。最好确保同一个项目中使用一个统一的基础镜像。如无特殊需求，可以选择使用<code>debian:jessie</code>或者<code>alpine</code>。</p>
</li>
<li>
<p>在构建镜像时，删除不需要的文件，只安装需要的文件，保持镜像干净、轻量。</p>
</li>
<li>
<p>使用更少的层，把相关的内容放到一个层，并使用换行符进行分割。这样可以进一步减小镜像的体积，也方便查看镜像历史。</p>
</li>
<li>
<p>不要在Dockerfile中修改文件的权限。因为如果修改文件的权限，Docker在构建时会重新复制一份，这会导致镜像体积越来越大。</p>
</li>
<li>
<p>给镜像打上标签，标签可以帮助你理解镜像的功能，例如：<code>docker build -t="nginx:3.0-onbuild"</code>。</p>
</li>
<li>
<p><code>FROM</code>指令应该包含tag，例如使用<code>FROM debian:jessie</code>，而不是<code>FROM debian</code>。</p>
</li>
<li>
<p>充分利用缓存。Docker构建引擎会顺序执行Dockerfile中的指令，而且一旦缓存失效，后续命令将不能使用缓存。为了有效地利用缓存，需要尽量将所有的Dockerfile文件中相同的部分都放在前面，而将不同的部分放在后面。</p>
</li>
<li>
<p>优先使用<code>COPY</code>而非<code>ADD</code>指令。和<code>ADD</code>相比，<code>COPY</code> 功能简单，而且也够用。<code>ADD</code>可变的行为会导致该指令的行为不清晰，不利于后期维护和理解。</p>
</li>
<li>
<p>推荐将<code>CMD</code>和<code>ENTRYPOINT</code>指令结合使用，使用execl格式的<code>ENTRYPOINT</code>指令设置固定的默认命令和参数，然后使用<code>CMD</code>指令设置可变的参数。</p>
</li>
<li>
<p>尽量使用Dockerfile共享镜像。通过共享Dockerfile，可以使开发者明确知道Docker镜像的构建过程，并且可以将Dockerfile文件加入版本控制，跟踪起来。</p>
</li>
<li>
<p>使用<code>.dockerignore</code>忽略构建镜像时非必需的文件。忽略无用的文件，可以提高构建速度。</p>
</li>
<li>
<p>使用多阶段构建。多阶段构建可以大幅减小最终镜像的体积。例如，<code>COPY</code>指令中可能包含一些安装包，安装完成之后这些内容就废弃掉。下面是一个简单的多阶段构建示例：</p>
</li>
</ol><pre><code class="language-dockerfile">FROM golang:1.11-alpine AS build

# 安装依赖包
RUN go get github.com/golang/mock/mockgen

# 复制源码并执行build，此处当文件有变化会产生新的一层镜像层
COPY . /go/src/iam/
RUN go build -o /bin/iam

# 缩小到一层镜像
FROM busybox
COPY --from=build /bin/iam /bin/iam
ENTRYPOINT ["/bin/iam"]
CMD ["--help"]
</code></pre><h2>总结</h2><p>如果你想使用Docker容器来部署应用，那么就需要制作Docker镜像。今天，我介绍了如何制作Docker镜像。</p><p>你可以使用这两种方式来构建Docker镜像：</p><ul>
<li>通过 <code>docker commit</code> 命令，基于一个已存在的容器构建出镜像。</li>
<li>通过编写Dockerfile文件，并使用 <code>docker build</code> 命令来构建镜像。</li>
</ul><p>这两种方法中，镜像构建的底层原理是相同的：</p><ol>
<li>
<p>基于原镜像启动一个Docker容器。</p>
</li>
<li>
<p>在容器中进行一些操作，例如执行命令、安装文件等，由这些操作产生的文件变更都会被记录在容器的存储层中。</p>
</li>
<li>
<p>将容器存储层的变更commit到新的镜像层中，并添加到原镜像上。</p>
</li>
</ol><p>此外，我们还可以使用 <code>docker save</code> / <code>docker load</code> 和 <code>docker export</code> / <code>docker import</code> 来复制Docker镜像。</p><p>在实际生产环境中，我们标准的做法是通过Dockerfile来构建镜像。使用Dockerfile构建镜像，就需要你编写Dockerfile文件。Dockerfile支持多个指令，这些指令可以分为5类，对指令的具体介绍你可以再返回复习一遍。</p><p>另外，我们在构建Docker镜像时，也要遵循一些最佳实践，具体你可以参考我给你总结的最佳实践清单。</p><h2>课后练习</h2><ol>
<li>
<p>思考下，为什么在编写Dockerfile时，“把相关的内容放到一个层，使用换行符 <code>\</code> 进行分割”可以减小镜像的体积？</p>
</li>
<li>
<p>尝试一下，为你正在开发的应用编写Dockerfile文件，并成功构建出Docker镜像。</p>
</li>
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
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q3auHgzwzM5suA7q5mM40ULTY5OlQpoerPRMQD8NcMbKxDHhNmjQNUCngkSJEzRvMVDibAHw2whGZxAFlibzribOA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jxlwqq</span>
  </div>
  <div class="_2_QraFYR_0">自荐一个dockerfile的写法：<br><br>```Dockerfile<br># 多阶段构建：提升构建速度，减少镜像大小<br><br># 从官方仓库中获取 1.17 的 Go 基础镜像<br>FROM golang:1.17-alpine AS builder<br><br># 设置工作目录<br>WORKDIR &#47;workspace<br><br># 安装项目依赖<br>COPY go.mod go.mod<br>COPY go.sum go.sum<br>RUN go mod download<br><br># 复制项目文件，这一步按需复制<br>COPY . .<br><br># 构建名为&quot;app&quot;的二进制文件<br>RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -a -o app main.go<br><br># 获取 Distroless 镜像，只有 650 kB 的大小，是常用的 alpine:latest 的 1&#47;4<br>FROM gcr.io&#47;distroless&#47;static:nonroot<br># 设置工作目录<br>WORKDIR &#47;<br># 将上一阶段构建好的二进制文件复制到本阶段中<br>COPY --from=builder &#47;workspace&#47;app .<br># 设置监听端口<br>EXPOSE 8080<br># 配置启动命令<br>ENTRYPOINT [&quot;&#47;app&quot;]<br>```</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 6666</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-13 22:19:50</div>
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
  <div class="_2_QraFYR_0">官方文档中最佳实践有介绍，RUN, COPY, ADD 三个指令会创建层，其他指令会创建一个中间镜像，并且不会影响镜像大小。这样我们说的指令合并也就是以这三个指令为主。当然了docker history查看构建历史与镜像大小，更为易读和简约</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 666</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-12 00:16:06</div>
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
  <div class="_2_QraFYR_0">感觉介绍IAM项目本身的相关内容少了点，像Docker相关的知识，其实给大家推荐一下资料就可以了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 主要考虑到有些读者基础比较薄弱，需要一些基本的docker知识。如果推荐一些资料，读者要从中找出跟iam相关的内容学习，并且这些资料，大而全，读完再来学习专栏周期会比较久。<br><br>我直接列出了一些核心知识，整理后供读者学习，能节省读者不少时间。<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-04 14:43:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/51/84/5b7d4d95.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>冷峰</span>
  </div>
  <div class="_2_QraFYR_0">go get 依赖 git 的吧， 不装 git , go get 能运行吗？ </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不安装git，go get基本不能运行。课程一开始就带着你们安装了git</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-06 22:46:47</div>
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
  <div class="_2_QraFYR_0">总结：<br>容器镜像的构建主要是通过Dockerfile来完成，构建过程如下：<br>1. 创建一个临时目录，将Context中的文件解压到该目录；<br>2. 执行Dockfile中的命令，一般是顺序执行，如果是多阶段构建，也会并行执行指令。<br>3. Docker Daemon 会为每条指令创建一个临时容器，执行该指令，生成镜像层，并缓存该镜像层。<br>4. 最终，将所有镜像层进行合并，生成最后的容器镜像。<br><br>最佳实践：<br>Dockerfile指令大写；From 选择官方容器镜像，指定镜像的tag；<br>使用尽量少地层：构建时将相同指令的内容放到一层；不要在Dockerfile中修改文件权限；采用多阶段构建，大幅减少容器镜像的体积。<br>充分利用缓存：不易修改的指令放在前面。<br>推荐使用COPY而非ADD；ENTRYPOINT和CMD结合使用。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-05 10:46:54</div>
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
  <div class="_2_QraFYR_0">老师想确认下，第二阶段的FROM busybox是会覆盖掉第一阶段的FROM是嘛，只是用第一阶段进行编译而已，然后用第二阶段的镜像去运行app</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你说的是对的。<br>第一阶段只是用来编译第二阶段需要的文件。docker后面运行的镜像是由第二阶段来构建的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-16 20:49:21</div>
  </div>
</div>
</div>
</li>
</ul>