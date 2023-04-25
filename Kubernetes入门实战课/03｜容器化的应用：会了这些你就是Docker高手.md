<audio title="03｜容器化的应用：会了这些你就是Docker高手" src="https://static001.geekbang.org/resource/audio/af/99/af7e91b4e25988b255206b81a5643499.mp3" controls="controls"></audio> 
<p>你好，我是Chrono。</p><p>在上一次课里，我们了解了容器技术中最核心的概念：容器，知道它就是一个系统中被隔离的特殊环境，进程可以在其中不受干扰地运行。我们也可以把这段描述再简化一点：<strong>容器就是被隔离的进程</strong>。</p><p>相比笨重的虚拟机，容器有许多优点，那我们应该如何创建并运行容器呢？是要用Linux内核里的namespace、cgroup、chroot三件套吗？</p><p>当然不会，那样的方式实在是太原始了，所以今天，我们就以Docker为例，来看看什么是容器化的应用，怎么来操纵容器化的应用。</p><h2>什么是容器化的应用</h2><p>之前我们运行容器的时候，显然不是从零开始的，而是要先拉取一个“镜像”（image），再从这个“镜像”来启动容器，像<a href="https://time.geekbang.org/column/article/528619">第一节课</a>这样：</p><pre><code class="language-plain">docker pull busybox      
docker run busybox echo hello world
</code></pre><p>那么，这个“镜像”到底是什么东西呢？它又和“容器”有什么关系呢？</p><p>其实我们在其他场合中也曾经见到过“镜像”这个词，比如最常见的光盘镜像，重装电脑时使用的硬盘镜像，还有虚拟机系统镜像。这些“镜像”都有一些相同点：只读，不允许修改，以标准格式存储了一系列的文件，然后在需要的时候再从中提取出数据运行起来。</p><!-- [[[read_end]]] --><p>容器技术里的镜像也是同样的道理。因为容器是由操作系统动态创建的，那么必然就可以用一种办法把它的初始环境给固化下来，保存成一个静态的文件，相当于是把容器给“拍扁”了，这样就可以非常方便地存放、传输、版本化管理了。</p><p><img src="https://static001.geekbang.org/resource/image/59/33/59a1cd035e21fe297072b20475d3c833.jpg?wh=1418x759" alt="图片"></p><p>如果还拿之前的“小板房”来做比喻的话，那么镜像就可以说是一个“样板间”，把运行进程所需要的文件系统、依赖库、环境变量、启动参数等所有信息打包整合到了一起。之后镜像文件无论放在哪里，操作系统都能根据这个“样板间”快速重建容器，应用程序看到的就会是一致的运行环境了。</p><p>从功能上来看，镜像和常见的tar、rpm、deb等安装包一样，都打包了应用程序，<strong>但最大的不同点在于它里面不仅有基本的可执行文件，还有应用运行时的整个系统环境。这就让镜像具有了非常好的跨平台便携性和兼容性</strong>，能够让开发者在一个系统上开发（例如Ubuntu），然后打包成镜像，再去另一个系统上运行（例如CentOS），完全不需要考虑环境依赖的问题，是一种更高级的应用打包方式。</p><p>理解了这一点，我们再回过头来看看第一节课里运行的Docker命令。</p><p><code>docker pull busybox</code> ，就是获取了一个打包了busybox应用的镜像，里面固化了busybox程序和它所需的完整运行环境。</p><p><code>docker run&nbsp;busybox echo hello world</code> ，就是提取镜像里的各种信息，运用namespace、cgroup、chroot技术创建出隔离环境，然后再运行busybox的 <code>echo</code> 命令，输出 <code>hello world</code> 的字符串。</p><p>这两个步骤，由于是基于标准的Linux系统调用和只读的镜像文件，所以，无论是在哪种操作系统上，或者是使用哪种容器实现技术，都会得到完全一致的结果。</p><p>推而广之，任何应用都能够用这种形式打包再分发后运行，这也是无数开发者梦寐以求的“一次编写，到处运行（Build once, Run anywhere）”的至高境界。所以，<strong>所谓的“容器化的应用”，或者“应用的容器化”，就是指应用程序不再直接和操作系统打交道，而是封装成镜像，再交给容器环境去运行</strong>。</p><p>现在你就应该知道了，镜像就是静态的应用容器，容器就是动态的应用镜像，两者互相依存，互相转化，密不可分。</p><p>之前的那张Docker官方架构图你还有印象吧，我们在第一节课曾经简单地介绍过。可以看到，在Docker里的核心处理对象就是镜像（image）和容器（container）：</p><p><img src="https://static001.geekbang.org/resource/image/c8/fe/c8116066bdbf295a7c9fc25b87755dfe.jpg?wh=1920x1048" alt="图片"></p><p>好，理解了什么是容器化的应用，接下来我们再来学习怎么操纵容器化的应用。因为镜像是容器运行的根本，先有镜像才有容器，所以先来看看关于镜像的一些常用命令。</p><h2>常用的镜像操作有哪些</h2><p>在前面的课程里你应该已经了解了两个基本命令，<code>docker pull</code> 从远端仓库拉取镜像，<code>docker images</code> 列出当前本地已有的镜像。</p><p><code>docker pull</code> 的用法还是比较简单的，和普通的下载非常像，不过我们需要知道镜像的命名规则，这样才能准确地获取到我们想要的容器镜像。</p><p><strong>镜像的完整名字由两个部分组成，名字和标签，中间用 <code>:</code> 连接起来。</strong></p><p>名字表明了应用的身份，比如busybox、Alpine、Nginx、Redis等等。标签（tag）则可以理解成是为了区分不同版本的应用而做的额外标记，任何字符串都可以，比如3.15是纯数字的版本号、jammy是项目代号、1.21-alpine是版本号加操作系统名等等。其中有一个比较特殊的标签叫“latest”，它是默认的标签，如果只提供名字没有附带标签，那么就会使用这个默认的“latest”标签。</p><p>那么现在，你就可以把名字和标签组合起来，使用 <code>docker pull</code> 来拉取一些镜像了：</p><pre><code class="language-plain">docker pull alpine:3.15
docker pull ubuntu:jammy
docker pull nginx:1.21-alpine
docker pull nginx:alpine
docker pull redis
</code></pre><p>有了这些镜像之后，我们再用 <code>docker images</code> 命令来看看它们的具体信息：</p><p><img src="https://static001.geekbang.org/resource/image/3c/00/3c6e24139acc6d791c189879a7608c00.png?wh=1818x608" alt="图片"></p><p>在这个列表里，你可以看到，REPOSITORY列就是镜像的名字，TAG就是这个镜像的标签，那么第三列“IMAGE ID”又是什么意思呢？</p><p>它可以说是镜像唯一的标识，就好像是身份证号一样。比如这里我们可以用“ubuntu:jammy”来表示Ubuntu 22.04镜像，同样也可以用它的ID“d4c2c……”来表示。</p><p>另外，你可能还会注意到，截图里的两个镜像“nginx:1.21-alpine”和“nginx:alpine”的IMAGE ID是一样的，都是“a63aa……”。这其实也很好理解，这就像是人的身份证号码是唯一的，但可以有大名、小名、昵称、绰号，同一个镜像也可以打上不同的标签，这样应用在不同的场合就更容易理解。</p><p>IMAGE ID还有一个好处，因为它是十六进制形式且唯一，Docker特意为它提供了“短路”操作，在本地使用镜像的时候，我们不用像名字那样要完全写出来这一长串数字，通常只需要写出前三位就能够快速定位，在镜像数量比较少的时候用两位甚至一位数字也许就可以了。</p><p>来看另一个镜像操作命令 <code>docker rmi</code> ，它用来删除不再使用的镜像，可以节约磁盘空间，注意命令 <code>rmi</code> ，实际上是“remove image”的简写。</p><p>下面我们就来试验一下，使用名字和IMAGE ID来删除镜像：</p><pre><code class="language-plain">docker rmi redis    
docker rmi d4c
</code></pre><p>这里的第一个 <code>rmi</code> 删除了Redis镜像，因为没有显式写出标签，默认使用的就是“latest”。第二个 <code>rmi</code> 没有给出名字，而是直接使用了IMAGE ID的前三位，也就是“d4c”，Docker就会直接找到这个ID前缀的镜像然后删除。</p><p>Docker里与镜像相关的命令还有很多，不过以上的 <code>docker pull</code>、<code>docker images</code>、<code>docker rmi</code> 就是最常用的三个了，其他的命令我们后续课程会陆续介绍。</p><p><img src="https://static001.geekbang.org/resource/image/27/19/27364161a8d3c1f960a91e07b5094419.jpg?wh=1920x963" alt="图片"></p><h2>常用的容器操作有哪些</h2><p>现在我们已经在本地存放了镜像，就可以使用 <code>docker run</code> 命令把这些静态的应用运行起来，变成动态的容器了。</p><p>基本的格式是“<strong>docker run 设置参数</strong>”，再跟上“<strong>镜像名或ID</strong>”，后面可能还会有附加的“<strong>运行命令</strong>”。</p><p>比如这个命令：</p><pre><code class="language-plain">docker run -h srv alpine hostname
</code></pre><p>这里的 <code>-h srv</code> 就是容器的运行参数，<code>alpine</code> 是镜像名，它后面的 <code>hostname</code> 表示要在容器里运行的“hostname”这个程序，输出主机名。</p><p><code>docker run</code> 是最复杂的一个容器操作命令，有非常多的额外参数用来调整容器的运行状态，你可以加上 <code>--help</code> 来看它的帮助信息，今天我只说几个最常用的参数。</p><p><code>-it</code> 表示开启一个交互式操作的Shell，这样可以直接进入容器内部，就好像是登录虚拟机一样。（它实际上是“-i”和“-t”两个参数的组合形式）</p><p><code>-d</code> 表示让容器在后台运行，这在我们启动Nginx、Redis等服务器程序的时候非常有用。</p><p><code>--name</code> 可以为容器起一个名字，方便我们查看，不过它不是必须的，如果不用这个参数，Docker会分配一个随机的名字。</p><p>下面我们就来练习一下这三个参数，分别运行Nginx、Redis和Ubuntu：</p><pre><code class="language-plain">docker run -d nginx:alpine            # 后台运行Nginx
docker run -d --name red_srv redis    # 后台运行Redis
docker run -it --name ubuntu 2e6 sh   # 使用IMAGE ID，登录Ubuntu18.04
</code></pre><p>因为第三个命令使用的是 <code>-it</code> 而不是 <code>-d</code> ，所以它会进入容器里的Ubuntu系统，我们需要另外开一个终端窗口，使用 <code>docker ps</code> 命令来查看容器的运行状态：</p><p><img src="https://static001.geekbang.org/resource/image/18/d9/18a20772328c55e22ae3529f2b7f70d9.png?wh=1920x197" alt="图片"></p><p>可以看到，每一个容器也会有一个“CONTAINER ID”，它的作用和镜像的“IMAGE ID”是一样的，唯一标识了容器。</p><p>对于正在运行中的容器，我们可以使用 <code>docker exec</code> 命令在里面执行另一个程序，效果和 <code>docker run</code> 很类似，但因为容器已经存在，所以不会创建新的容器。它最常见的用法是使用 <code>-it</code> 参数打开一个Shell，从而进入容器内部，例如：</p><pre><code class="language-plain">docker exec -it red_srv sh
</code></pre><p>这样我们就“登录”进了Redis容器，可以很方便地查看服务的运行状态或者日志。</p><p>运行中的容器还可以使用 <code>docker stop</code> 命令来强制停止，这里我们仍然可以使用容器名字，不过或许用“CONTAINER ID”的前三位数字会更加方便。</p><pre><code class="language-plain">docker stop ed4 d60 45c
</code></pre><p>容器被停止后使用 <code>docker ps</code> 命令就看不到了，不过容器并没有被彻底销毁，我们可以使用 <code>docker ps -a</code> 命令查看系统里所有的容器，当然也包括已经停止运行的容器：</p><p><img src="https://static001.geekbang.org/resource/image/61/4d/616d93a9998fee3b958ca892yy33d14d.png?wh=1920x204" alt="图片"></p><p>这些停止运行的容器可以用 <code>docker start</code> 再次启动运行，如果你确定不再需要它们，可以使用 <code>docker rm</code> 命令来彻底删除。</p><p><strong>注意，这个命令与 <code>docker rmi</code> 非常像，区别在于它没有后面的字母“i”，所以只会删除容器，不删除镜像。</strong></p><p>下面我们就来运行 <code>docker rm</code> 命令，使用“CONTAINER ID”的前两位数字来删除这些容器：</p><pre><code class="language-plain">docker rm ed d6 45
</code></pre><p>执行删除命令之后，再用 <code>docker ps -a</code> 查看列表就会发现这些容器已经彻底消失了。</p><p>你可能会感觉这样的容器管理方式很麻烦，启动后要ps看ID再删除，如果稍微不注意，系统就会遗留非常多的“死”容器，占用系统资源，有没有什么办法能够让Docker自动删除不需要的容器呢？</p><p>办法当然有，就是在执行 <code>docker run</code> 命令的时候加上一个 <code>--rm</code> 参数，这就会告诉Docker不保存容器，只要运行完毕就自动清除，省去了我们手工管理容器的麻烦。</p><p>我们还是用刚才的Nginx、Redis和Ubuntu这三个容器来试验一下，加上 <code>--rm</code> 参数（省略了name参数）：</p><pre><code class="language-plain">docker run -d --rm nginx:alpine 
docker run -d --rm redis
docker run -it --rm 2e6 sh 
</code></pre><p>然后我们用 <code>docker stop</code> 停止容器，再用 <code>docker ps -a</code> ，就会发现不需要我们再手动执行 <code>docker rm</code> ，Docker已经自动删除了这三个容器。</p><p><img src="https://static001.geekbang.org/resource/image/c8/85/c8cd008e91aaff2cd91e0392b0079085.jpg?wh=1920x1747" alt="图片"></p><h2>小结</h2><p>好了，今天我们一起学习了容器化的应用，然后使用Docker实际操作了镜像和容器，运行了被容器化的Alpine、Nginx、Redis等应用。</p><p>镜像是容器的静态形式，它打包了应用程序的所有运行依赖项，方便保存和传输。使用容器技术运行镜像，就形成了动态的容器，由于镜像只读不可修改，所以应用程序的运行环境总是一致的。</p><p>而容器化的应用就是指以镜像的形式打包应用程序，然后在容器环境里从镜像启动容器。</p><p>由于Docker的命令比较多，而且每个命令还有许多参数，一节课里很难把它们都详细说清楚，希望你课下参考Docker自带的帮助或者官网文档（<a href="https://docs.docker.com/reference/">https://docs.docker.com/reference/</a>），再多加实操练习，相信你一定能够成为Docker高手。</p><p>我这里就对今天的镜像操作和容器操作做个小结：</p><ol>
<li>常用的镜像操作有 <code>docker pull</code>、<code>docker images</code>、<code>docker rmi</code>，分别是拉取镜像、查看镜像和删除镜像。</li>
<li>用来启动容器的 <code>docker run</code> 是最常用的命令，它有很多参数用来调整容器的运行状态，对于后台服务来说应该使用 <code>-d</code>。</li>
<li><code>docker exec</code> 命令可以在容器内部执行任意程序，对于调试排错特别有用。</li>
<li>其他常用的容器操作还有 <code>docker ps</code>、<code>docker stop</code>、<code>docker rm</code>，用来查看容器、停止容器和删除容器。</li>
</ol><h2>课下作业</h2><p>最后是课下作业时间，给你留两个思考题：</p><ol>
<li>说一说你对容器镜像的理解，它与rpm、deb安装包有哪些不同和优缺点。</li>
<li>你觉得 <code>docker run</code> 和 <code>docker exec</code> 的区别在哪里，应该怎么使用它们？</li>
</ol><p>欢迎在留言区参与讨论，据说打字发言能把自己学到的新知识再加工一遍，可以显著提升理解哦。</p><p>我们下节课再见。<br>
<img src="https://static001.geekbang.org/resource/image/74/f9/7405faa28109b810cace4975eb3a4ef9.jpg?wh=1920x2481" alt=""></p>
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
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/WrANpwBMr6DsGAE207QVs0YgfthMXy3MuEKJxR8icYibpGDCI1YX4DcpDq1EsTvlP8ffK1ibJDvmkX9LUU4yE8X0w/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>星垂平野阔</span>
  </div>
  <div class="_2_QraFYR_0">作业1：<br>容器镜像比起这些安装包的差别就在于通用，不同linux版本下的安装包还不同。<br>作业2:<br>run是针对容器本身启动，而exec是进入了容器内部去跑命令，相当于进去操作系统跑应用。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-27 10:01:14</div>
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
  <div class="_2_QraFYR_0">老师后面的课程是会用k8s带领我们模拟真实场景，部署应用吗？<br><br>「纸上得来终觉浅」。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 会部署WordPress应用，但要是真实场景就不太好找了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-27 16:21:15</div>
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
  <div class="_2_QraFYR_0">我实操了下，nginx:alpine 和 nginx:1.21-alpine image_id是不一样的，我猜是被更新了导致image_id不一样了，因为created也不一样</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的。nginx:alpine始终是最新版本的Nginx，现在是1.23了，我当时是一样的。<br><br>你可以试试nginx:alpine和nginx:1.23-alpine 。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-27 09:19:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJk3PElN2J96DtyWuIg6xPSs3zRFsIMibOvIn5kuRkESORsRIkDJMUekymI2wiaYiaP0UzibXWEl0aLYw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Bill</span>
  </div>
  <div class="_2_QraFYR_0">1.容器镜像通过分层打包，安装所有依赖包，并可以在主机上共享使用，减少存储空间需求，它与 rpm、deb 安装包作为某一个功能的所有依赖包安装，聚焦某个命令的上下文，容器是整个应用的打包。<br>2.docker run利用镜像运行容器，拥有丰富的启动参数，如挂载volume，端口映射等。是容器运行启动的基础。docker exec启动session，在一个已运行的容器中执行命令，仅当PID 1进程存在时运行，容器重启后，session将失效。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-27 08:11:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/05/8a/e7c5a7e2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>sky</span>
  </div>
  <div class="_2_QraFYR_0">还有一些命令docker  save,docker  load,docker stats,docker cp也很有用</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: save、load、cp等命令后面会讲，docker命令太多了，全讲不现实，得尽快进入Kubernetes。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-09 10:11:58</div>
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
  <div class="_2_QraFYR_0">老师，课外贴士中的第4条如何删除呢？有时候强制删除也没用</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 把多余的tag依次删除，最后就可以直接删除镜像了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-27 00:34:30</div>
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
  <div class="_2_QraFYR_0">1. 课外贴士的第四条，有同学问了，怎么删除，老师的回答我没太明白，最佳实践是如何操作呢？<br><br>2. 想听听老师的回答：docker run 和 docker exec 的区别在哪里？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1. 用rmi删除镜像，不用image id，而是用名字加标签，当其他的名字加标签的引用都没有的时候就可以直接删除了。<br><br>2.这个删除镜像没有什么最佳实践，这个其实是docker防止误删除的一个保险。<br><br>3.简单来说，docker run是用命令启动一个容器，而docker exec是在已有容器里执行命令</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-27 16:45:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/63/1b/83ac7733.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>忧天小鸡</span>
  </div>
  <div class="_2_QraFYR_0">苦于没有docker入门，耗费大量时间，这教程真是太creat了。<br>大佬的课我全入了，对你的讲述感觉十分易懂，不需要绕弯理解，nice的。<br>等我cpp入门，去试试你们公司</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great，努力学习进步吧。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-27 09:48:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/20/fb/96/791d0f5e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小伙儿</span>
  </div>
  <div class="_2_QraFYR_0">docker镜像和rpm包的区别<br><br>镜像在打包推送到仓库后不管在那个操作系统中都能运行，而rpm包不行<br>docker镜像中包含了完整的应用依赖和系统环境，而rpm包则没有<br>镜像比较能节约磁盘空间，如果镜像的部分层已经在本地中有了，就可以直接复用，rpm包不行<br><br><br>docker的run和exec的区别<br><br>run是从镜像创建运行一个容器的必备命令，exec则是在已经运行的容器中执行另外一个程序，他们的优先级是先run后exec。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-16 09:09:46</div>
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
  <div class="_2_QraFYR_0">老师好，想问一个问题，那就是k8s的container和docker容器有什么区别吗，我使用dockerfile打包一个镜像，在docker环境中是可以打包成功，但是放到使用k8s的jenkins流水线上，就无法打包成功，云平台相关工程师告诉我可能是k8s和docker的不兼容导致的，想请问一下这个问题</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 镜像都用的是OCI标准，不存在兼容问题，应该从其他方面找原因。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-27 21:16:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/21/56/91669d26.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ReCharge</span>
  </div>
  <div class="_2_QraFYR_0">买了老师很多的专栏了，每次都受益匪浅，老师能分享下学习新知识的方法么？如何能够做到这么深入浅出的。。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 在结束语里写了一点心得，简单来说就是多记笔记多练习，勤思考，不过说起来容易做起来就难了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-09 21:15:14</div>
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
  <div class="_2_QraFYR_0">之前一直以为IMAGE ID是随机的，又学到了，原来是跟镜像文件相关的，特意去看了一下，不同的机器上的同一个image的同一个tag，IMAGE ID确实是一样的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-22 16:30:32</div>
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
  <div class="_2_QraFYR_0">为什么我刚执行docker run -it --name ubuntu ad0 sh  ，用docker ps -a 查到他的状态就是Exited？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 执行的是“sh”，退出shell自然就结束运行了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-13 23:26:55</div>
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
  <div class="_2_QraFYR_0">docker 镜像运行以后，如果想要删除镜像，需要通过 docker rm [xxx containerId]（可以通过 docker ps -a 查看 containerId）删除容器，才能通过 docker rmi [xxx imageId] 删除镜像。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，这是因为容器引用了镜像，有引用就不能直接删除镜像。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-07 19:02:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/88/07/7804f4cc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>逗逼师父</span>
  </div>
  <div class="_2_QraFYR_0">1. 容器镜像就是预先配置好的一个带有操作系统的环境；rpm, deb包与当前操作系统是强依赖关系，还会依赖于其他应用等，而容器镜像只要在装有docker的机器上pull下来就能用了，运行起来就是一套独立的环境。<br>2. run是用指定镜像创建一个新的容器并运行；exec是在已经运行的容器中执行命令。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-29 22:31:19</div>
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
  <div class="_2_QraFYR_0">老师，执行docker images 命令，展示栏的 CREATED 代表的是这个 镜像仓库最近的版本更新情况吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是这个镜像的创建时间，说是镜像仓库最近的版本更新情况勉强也可以。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-27 16:51:47</div>
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
  <div class="_2_QraFYR_0">1.容器镜像更像是一个完整的操作系统+环境、rpm、deb是软件安装包；<br>2.docker run是运行一个容器，docker exec是进入容器；</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-02 14:19:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/36/0c/a5/344c1ea2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>V.</span>
  </div>
  <div class="_2_QraFYR_0">&quot;docker run -it --name ubuntu 2e6 sh   # 使用IMAGE ID，登录Ubuntu18.04&quot;<br>这个是不是错了呀 会提示找不到2e6 这个镜像，我改成“--name 2e6 ubuntu ” 就可以启动了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 2e6是Ubuntu 1804的image id，需要看一下自己的docker环境里的image id是什么，有可能不是这个数字。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-22 11:40:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/32/50/8d/ded8482f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>一久</span>
  </div>
  <div class="_2_QraFYR_0">问题一：<br>容器镜像可以是一个打包好的应用系统文件，里面包含了基础镜像和服务依赖，可以跨同架构的多个发行版Linux系统上运行；rpm和deb包是Linux某个功能的安装包文件，在支持相应包管理的系统下才可以安装使用<br>问题二：<br>docker run是运行一个容器；docker exec是进入已运行的容器</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-17 03:08:08</div>
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
  <div class="_2_QraFYR_0">补充一下：<br>一次停止所有容器：docker stop `docker ps -a -q`<br>一次删除所有容器：docker rm    `docker ps -a -q`</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-15 15:26:24</div>
  </div>
</div>
</div>
</li>
</ul>