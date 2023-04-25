<audio title="05｜镜像仓库：该怎样用好Docker Hub这个宝藏" src="https://static001.geekbang.org/resource/audio/c9/46/c9a8f783a22029438f2b450643c04f46.mp3" controls="controls"></audio> 
<p>你好，我是Chrono。</p><p>上一次课里我们学习了“Dockerfile”和“docker build”的用法，知道了如何创建自己的镜像。那么镜像文件应该如何管理呢，具体来说，应该如何存储、检索、分发、共享镜像呢？不解决这些问题，我们的容器化应用还是无法顺利地实施。</p><p>今天，我就来谈一下这个话题，聊聊什么是镜像仓库，还有该怎么用好镜像仓库。</p><h2>什么是镜像仓库（Registry）</h2><p>之前我们已经用过 <code>docker pull</code> 命令拉取镜像，也说过有一个“镜像仓库”（Registry）的概念，那到底什么是镜像仓库呢？</p><p>还是来看Docker的官方架构图（它真的非常重要）：</p><p><img src="https://static001.geekbang.org/resource/image/c8/fe/c8116066bdbf295a7c9fc25b87755dfe.jpg?wh=1920x1048" alt="图片"></p><p>图里右边的区域就是镜像仓库，术语叫Registry，直译就是“注册中心”，意思是所有镜像的Repository都在这里登记保管，就像是一个巨大的档案馆。</p><p>然后我们再来看左边的“docker pull”，虚线显示了它的工作流程，先到“Docker daemon”，再到Registry，只有当Registry里存有镜像才能真正把它下载到本地。</p><p>当然了，拉取镜像只是镜像仓库最基本的一个功能，它还会提供更多的功能，比如上传、查询、删除等等，是一个全面的镜像管理服务站点。</p><!-- [[[read_end]]] --><p>你也可以把镜像仓库类比成手机上的应用商店，里面分门别类存放了许多容器化的应用，需要什么去找一下就行了。有了它，我们使用镜像才能够免除后顾之忧。</p><h2>什么是Docker Hub</h2><p>不过，你有没有注意到，在使用 <code>docker pull</code> 获取镜像的时候，我们并没有明确地指定镜像仓库。在这种情况下，Docker就会使用一个默认的镜像仓库，也就是大名鼎鼎的“<strong>Docker Hub</strong>”（<a href="https://hub.docker.com/">https://hub.docker.com/</a>）。</p><p>Docker Hub是Docker公司搭建的官方Registry服务，创立于2014年6月，和Docker 1.0同时发布。它号称是世界上最大的镜像仓库，和GitHub一样，几乎成为了容器世界的基础设施。</p><p>Docker Hub里面不仅有Docker自己打包的镜像，而且还对公众免费开放，任何人都可以上传自己的作品。经过这8年的发展，Docker Hub已经不再是一个单纯的镜像仓库了，更应该说是一个丰富而繁荣的容器社区。</p><p>你可以看看下面的这张截图，里面列出的都是下载量超过10亿次（1 Billion）的最受欢迎的应用程序，比如Nginx、MongoDB、Node.js、Redis、OpenJDK等等。显然，把这些容器化的应用引入到我们自己的系统里，就像是站在了巨人的肩膀上，一开始就会有一个高水平的起点。</p><p><img src="https://static001.geekbang.org/resource/image/d4/e3/d47cc4d3f867069b055a47628acac2e3.png?wh=1920x890" alt="图片"></p><p>但和GitHub、App Store一样，面向所有人公开的Docker Hub也有一个不可避免的缺点，就是“良莠不齐”。</p><p>在Docker Hub搜索框里输入关键字，比如Nginx、MySQL，它立即就会给出几百几千个搜索结果，有点“乱花迷人眼”的感觉，这么多镜像，应该如何挑选出最适合自己的呢？下面我就来说说自己在这方面的一些经验。</p><h2>如何在Docker Hub上挑选镜像</h2><p>首先，你应该知道，在Docker Hub上有<strong>官方镜像</strong>、<strong>认证镜像</strong>和<strong>非官方镜像</strong>的区别。</p><p>官方镜像是指Docker公司官方提供的高质量镜像（<a href="https://github.com/docker-library/official-images">https://github.com/docker-library/official-images</a>），都经过了严格的漏洞扫描和安全检测，支持x86_64、arm64等多种硬件架构，还具有清晰易读的文档，一般来说是我们构建镜像的首选，也是我们编写Dockerfile的最佳范例。</p><p>官方镜像目前有大约100多个，基本上囊括了现在的各种流行技术，下面就是官方的Nginx镜像网页截图：</p><p><img src="https://static001.geekbang.org/resource/image/10/f3/109fc664da4f5124c4758b0e8f9c95f3.png?wh=1376x754" alt="图片"></p><p>你会看到，官方镜像会有一个特殊的“<strong>Official image</strong>”的标记，这就表示这个镜像经过了Docker公司的认证，有专门的团队负责审核、发布和更新，质量上绝对可以放心。</p><p>第二类是认证镜像，标记是“<strong>Verified publisher</strong>”，也就是认证发行商，比如Bitnami、Rancher、Ubuntu等。它们都是颇具规模的大公司，具有不逊于Docker公司的实力，所以就在Docker Hub上开了个认证账号，发布自己打包的镜像，有点类似我们微博上的“大V”。</p><p><img src="https://static001.geekbang.org/resource/image/57/f2/576d07439fc85d2bc461953f31a084f2.png?wh=1058x542" alt="图片"></p><p>这些镜像有公司背书，当然也很值得信赖，不过它们难免会带上一些各自公司的“烙印”，比如Bitnami的镜像就统一以“minideb”为基础，灵活性上比Docker官方镜像略差，有的时候也许会不符合我们的需求。</p><p>除了官方镜像和认证镜像，剩下的就都属于非官方镜像了，不过这里面也可以分出两类。</p><p>第一类是“<strong>半官方</strong>”镜像。因为成为“Verified publisher”是要给Docker公司交钱的，而很多公司不想花这笔“冤枉钱”，所以只在Docker Hub上开了公司账号，但并不加入认证。</p><p>这里我以OpenResty为例，看一下它的Docker Hub页面，可以看到显示的是OpenResty官方发布，但并没有经过Docker正式认证，所以难免就会存在一些风险，有被“冒名顶替”的可能，需要我们在使用的时候留心鉴别一下。不过一般来说，这种“半官方”镜像也是比较可靠的。</p><p><img src="https://static001.geekbang.org/resource/image/6d/3c/6d94d351137fb72cab36d73e8eea1f3c.png?wh=1496x566" alt="图片"></p><p>第二类就是纯粹的“<strong>民间</strong>”镜像了，通常是个人上传到Docker Hub的，因为条件所限，测试不完全甚至没有测试，质量上难以得到保证，下载的时候需要小心谨慎。</p><p>除了查看镜像是否为官方认证，我们还应该再结合其他的条件来判断镜像质量是否足够好。做法和GitHub差不多，就是看它的<strong>下载量、星数、还有更新历史</strong>，简单来说就是“好评”数量。</p><p>一般来说下载量是最重要的参考依据，好的镜像下载量通常都在百万级别（超过1M），而有的镜像虽然也是官方认证，但缺乏维护，更新不及时，用的人很少，星数、下载数都寥寥无几，那么还是应该选择下载量最多的镜像，通俗来说就是“随大流”。</p><p>下面的这张截图就是OpenResty在Docker Hub上的搜索结果。可以看到，有两个认证发行商的镜像（Bitnami、IBM），但下载量都很少，还有一个“民间”镜像下载量虽然超过了1M，但更新时间是3年前，所以毫无疑问，我们应该选择排在第三位，但下载量超过10M、有360多个星的“半官方”镜像。</p><p><img src="https://static001.geekbang.org/resource/image/5c/93/5c0b39da3bba66955e2byydcbe0d8593.png?wh=1878x1530" alt="图片"></p><p>看了这么多Docker Hub上的镜像，你一定注意到了，应用都是一样的名字，比如都是Nginx、Redis、OpenResty，该怎么区分不同作者打包出的镜像呢？</p><p>如果你熟悉GitHub，就会发现Docker Hub也使用了同样的规则，就是“<strong>用户名/应用名</strong>”的形式，比如 <code>bitnami/nginx</code>、<code>ubuntu/nginx</code>、<code>rancher/nginx</code> 等等。</p><p>所以，我们在使用 <code>docker pull</code> 下载这些非官方镜像的时候，就必须把用户名也带上，否则默认就会使用官方镜像：</p><pre><code class="language-plain">docker pull bitnami/nginx
docker pull ubuntu/nginx
</code></pre><h2>Docker Hub上镜像命名的规则是什么</h2><p>确定了要使用的镜像还不够，因为镜像还会有许多不同的版本，也就是“标签”（tag）。</p><p>直接使用默认的“latest”虽然简单方便，但在生产环境里是一种非常不负责任的做法，会导致版本不可控。所以我们还需要理解Docker Hub上标签命名的含义，才能够挑选出最适合我们自己的镜像版本。</p><p>下面我就拿官方的Redis镜像作为例子，解释一下这些标签都是什么意思。</p><p><img src="https://static001.geekbang.org/resource/image/1d/d5/1dd392b8f286507b83cd31400d5dccd5.png?wh=1796x846" alt="图片"></p><p>通常来说，镜像标签的格式是<strong>应用的版本号加上操作系统</strong>。</p><p>版本号你应该比较了解吧，基本上都是<strong>主版本号+次版本号+补丁号</strong>的形式，有的还会在正式发布前出rc版（候选版本，release candidate）。而操作系统的情况略微复杂一些，因为各个Linux发行版的命名方式“花样”太多了。</p><p>Alpine、CentOS的命名比较简单明了，就是数字的版本号，像这里的 <code>alpine3.15</code> ，而Ubuntu、Debian则采用了代号的形式。比如Ubuntu 18.04是 <code>bionic</code>，Ubuntu 20.04是 <code>focal</code>，Debian 9是 <code>stretch</code>，Debian 10是 <code>buster</code>，Debian 11是 <code>bullseye</code>。</p><p><strong>另外，有的标签还会加上 <code>slim</code>、<code>fat</code>，来进一步表示这个镜像的内容是经过精简的，还是包含了较多的辅助工具</strong>。通常 <code>slim</code> 镜像会比较小，运行效率高，而 <code>fat</code> 镜像会比较大，适合用来开发调试。</p><p>下面我就列出几个标签的例子来说明一下。</p><ul>
<li>nginx:1.21.6-alpine，表示版本号是1.21.6，基础镜像是最新的Alpine。</li>
<li>redis:7.0-rc-bullseye，表示版本号是7.0候选版，基础镜像是Debian 11。</li>
<li>node:17-buster-slim，表示版本号是17，基础镜像是精简的Debian 10。</li>
</ul><h2>该怎么上传自己的镜像</h2><p>现在，我想你应该对如何在Docker Hub上选择镜像有了比较全面的了解，那么接下来的问题就是，我们自己用Dockerfile创建的镜像该如何上传到Docker Hub上呢？</p><p>这件事其实一点也不难，只需要4个步骤就能完成。</p><p>第一步，你需要在Docker Hub上注册一个用户，这个就不必再多说了。</p><p>第二步，你需要在本机上使用 <code>docker login</code> 命令，用刚才注册的用户名和密码认证身份登录，像这里就用了我的用户名“chronolaw”：<br>
<img src="https://static001.geekbang.org/resource/image/43/03/436c49175de3b19b2473a0a3f37cd603.png?wh=1920x359" alt="图片"></p><p>第三步很关键，需要使用 <code>docker tag</code> 命令，给镜像改成带用户名的完整名字，表示镜像是属于这个用户的。或者简单一点，直接用 <code>docker build -t</code> 在创建镜像的时候就起好名字。</p><p>这里我就用上次课里的镜像“ngx-app”作为例子，给它改名成 <code>chronolaw/ngx-app:1.0</code>：</p><pre><code class="language-plain">docker tag ngx-app chronolaw/ngx-app:1.0
</code></pre><p><img src="https://static001.geekbang.org/resource/image/a0/60/a0b97300bb53da4c1df12b785dc52260.png?wh=1884x246" alt="图片"></p><p>第四步，用 <code>docker push</code> 把这个镜像推上去，我们的镜像发布工作就大功告成了：</p><pre><code class="language-plain">docker push chronolaw/ngx-app:1.0
</code></pre><p><img src="https://static001.geekbang.org/resource/image/51/a6/517dee3b1d6d0fd9809014cd4b9bbca6.png?wh=1544x668" alt="图片"></p><p>你还可以登录Docker Hub网站验证一下镜像发布的效果，可以看到它会自动为我们生成一个页面模板，里面还可以进一步丰富完善，比如添加描述信息、使用说明等等：<br>
<img src="https://static001.geekbang.org/resource/image/76/d1/76f1f566c029d2a3743d79d80cdeddd1.png?wh=1920x715" alt="图片"></p><p>现在你就可以把这个镜像的名字（用户名/应用名:标签）告诉你的同事，让他去用 <code>docker pull</code> 下载部署了。</p><h2>离线环境该怎么办</h2><p>使用Docker Hub来管理镜像的确是非常方便，不过有一种场景下它却是无法发挥作用，那就是企业内网的离线环境，连不上外网，自然也就不能使用 <code>docker push</code>、<code>docker pull</code>  来推送拉取镜像了。</p><p>那这种情况有没有解决办法呢？</p><p>方法当然有，而且有很多。最佳的方法就是在内网环境里仿造Docker Hub，创建一个自己的私有Registry服务，由它来管理我们的镜像，就像我们自己搭建GitLab做版本管理一样。</p><p>自建Registry已经有很多成熟的解决方案，比如Docker Registry，还有CNCF Harbor，不过使用它们还需要一些目前没有讲到的知识，步骤也有点繁琐，所以我会在后续的课程里再介绍。</p><p>下面我讲讲存储、分发镜像的一种“笨”办法，虽然比较“原始”，但简单易行，可以作为临时的应急手段。</p><p>Docker提供了 <code>save</code> 和 <code>load</code> 这两个镜像归档命令，可以把镜像导出成压缩包，或者从压缩包导入Docker，而压缩包是非常容易保管和传输的，可以联机拷贝，FTP共享，甚至存在U盘上随身携带。</p><p>需要注意的是，这两个命令默认使用标准流作为输入输出（为了方便Linux管道操作），所以一般会用 <code>-o</code>、<code>-i</code> 参数来使用文件的形式，例如：</p><pre><code class="language-plain">docker save ngx-app:latest -o ngx.tar
docker load -i ngx.tar
</code></pre><h2>小结</h2><p>好了，今天我们一起学习了镜像仓库，了解了Docker Hub的使用方法，整理一下要点方便你加深理解：</p><ol>
<li>镜像仓库（Registry）是一个提供综合镜像服务的网站，最基本的功能是上传和下载。</li>
<li>Docker Hub是目前最大的镜像仓库，拥有许多高质量的镜像。上面的镜像非常多，选择的标准有官方认证、下载量、星数等，需要综合评估。</li>
<li>镜像也有很多版本，应该根据版本号和操作系统仔细确认合适的标签。</li>
<li>在Docker Hub注册之后就可以上传自己的镜像，用 <code>docker tag</code> 打上标签再用 <code>docker push</code> 推送。</li>
<li>离线环境可以自己搭建私有镜像仓库，或者使用 <code>docker save</code> 把镜像存成压缩包，再用 <code>docker load</code> 从压缩包恢复成镜像。</li>
</ol><h2>课下作业</h2><p>最后是课下作业时间，给你留两个思考题：</p><ol>
<li>很多应用（如Nginx、Redis、Go）都已经有了Docker官方镜像，为什么其他公司（Bitnami、Rancher）还要重复劳动，发布自己打包的镜像呢？</li>
<li>你能否对比一下GitHub和Docker Hub，说说它们两个在功能、服务对象、影响范围等方面的异同点呢？</li>
</ol><p>记得在留言区留言参与讨论哦，如果我看到，也会第一时间给你回复。我们下节课再见。</p><p><img src="https://static001.geekbang.org/resource/image/aa/39/aa948a0f7deea9b572a5536bfb1e1039.jpg?wh=1920x2580" alt=""></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/cd/e0/c85bb948.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>朱雯</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，在文中提到的arm架构和x86架构支持，请问一下，能否使用dockerfile创建同时支持两种服务的镜像呢。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以的，用manifest的方式，在一个标签里存不同架构的镜像，可以搜一下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-01 08:56:49</div>
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
  <div class="_2_QraFYR_0">1.我猜是把自己的工具打包进去或者官方镜像满足不了他们自己的需求<br>2.github 和docker hub都是仓库，不过一个是代码仓库，一个是容器仓库，面向的都是程序员或者是计算机爱好者，都提供了存储和分发功能</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-01 08:25:43</div>
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
  <div class="_2_QraFYR_0">老师，很期待后面的自建镜像仓库啊 </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Registry都已经容器化了，自建非常简单，难的是围绕它的管理工作。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-01 16:52:29</div>
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
  <div class="_2_QraFYR_0">思考题：<br>1. 感觉还是为了方便用户，就拿 ubuntu 举例，官方的基本上就是一个空操作系统，而商业公司就会在其中配置一些环境或安装一些跟公司相关的应用，用户 pull 下来直接使用即可无需从头配置<br><br>2. 一个主要是为了管理代码，一个主要为了管理容器。代码仓库主要是面向开发人员，让开发人员能够更好更方便地提出问题、审查代码、流程版本控制等等。而容器仓库主要是面向运维人员，这里面相比代码仓库少了很多的环节，毕竟面对的是一个直接 pull 下来就可以使用的应用，不需要过多的审核和提示等等。个人感觉还是 GitHub 的影响范围更大吧，毕竟所有的应用归根结底都是程序，而并不是所有的程序都需要打包成镜像<br><br>最后想请教老师，能否指定镜像仓库而不是从默认的 dockerHub 上面抓取镜像？是使用 `docker pull --platform` 指令吗？有名的镜像仓库有哪些？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 说得很好，GitHub是代码仓库，dockerhub是应用仓库。<br><br>指定仓库加上网址就可以了，比如gcr.io quay.io。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-03 07:49:05</div>
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
  <div class="_2_QraFYR_0">请问老师：<br><br>docker官网的内容我感觉很多，如何找到重点快速学习呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这也是官网的通病，大而全，不好找重点，建议用关键字搜索。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-01 21:19:46</div>
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
  <div class="_2_QraFYR_0">老师的专栏很不错，不过看到的比较晚。知道以后就抓紧赶，终于同步了。<br>前面几篇的学习中，积累了如下几个问题：<br>Q1：04篇中，如果run命令有多行，即包含多个“\”以及多个“&amp;&amp;”，那么，最后是生成一个layer还是多个layer？（文中有一句“每个指令都会生成一个 Layer”）。<br>Q2：04篇的问题：基于某个系统创建的镜像，可以运行在其他系统上吗？比如，基于ubuntu18创建的镜像，可以运行在centos等系统上吗？<br>Q3：05篇的问题：除了docker Hub以及国外的其他几个仓库外，国内有docker仓库吗？<br>Q4：“课前准备”篇中，提到了VMWare Fusion。 我用的虚拟机是VMWare workstation16，这个可以吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.虽然是多行，但因为有续行符，所以docker仍然会看做是一行，只会有一层。<br><br>2.当然可以，镜像都是标准格式，只需要容器运行时，不关心下面的操作系统。<br><br>3.国内的不太了解，抱歉。<br><br>4.当然可以，不过VMWare workstation没用过，但应该都差不多。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-01 15:02:04</div>
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
  <div class="_2_QraFYR_0">1. 其他公司有自己的环境配置需求，还可以顺便刷存在感。<br>2. Github是代码托管，侧重服务软件的开发阶段，范围主要是使用开源软件的开发者；DockerHub是托管镜像，侧重服务软件的部署，使用阶段，范围主要是运维工作者。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-02 21:42:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/45/a9/3d48d6a2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Lorry</span>
  </div>
  <div class="_2_QraFYR_0">国内Docker镜像仓库一般都是配置阿里云的吧，老师应该提一下，否则拉取镜像太慢了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 说实在的，国内的我基本不用，所以不是很了解它们的用法，抱歉了，可以参考一下其他同学的回复。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-03 12:42:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/30/92/9f/d5255fe8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>snake</span>
  </div>
  <div class="_2_QraFYR_0">1. 加点私活或者官方镜像某些功能不满足特定的需求<br>2. GitHub可以上传自己或者公司的开源代码，Docker Hub只是docker的镜像管理，功能比GitHub少</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-03 15:06:27</div>
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
  <div class="_2_QraFYR_0">请教下老师，是不是可以这么理解：Registry 是镜像仓库的标准化定义，而 Docker Hub 是 Registry 的一种实现，其它容器厂商也可以实现自己的 Registry？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Registry是一个概念，不是标准，当然docker自己的Registry已经做的很好了。<br><br>后面的理解没错，docker hub、gcr.io都是Registry的一种实现。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-11 16:40:27</div>
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
  <div class="_2_QraFYR_0">docker官方镜像也有用户名，library, 一般省略！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-03 06:26:48</div>
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
  <div class="_2_QraFYR_0">Docker继承git和github的好多优秀的概念，pull&#47;push&#47;commit&#47;tag&#47;。。。。<br><br>想问一下老师，在编程中经常会有一些循环依赖的问题，想请问一下，会不会Dockerfile中也有，例如 A from B， B from C， C from A，这种问题存在吗？<br><br>假想一下，alpine的镜像被投毒了，是不是整个docker image镜像世界崩塌了一半？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.这个应该不会，目前看来没发生过。<br><br>2.很有可能，所以要尽量找官方镜像，镜像也要有安全检查的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-22 18:14:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/6b/e9/7620ae7e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>雨落～紫竹</span>
  </div>
  <div class="_2_QraFYR_0">打卡</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-10 22:55:22</div>
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
  <div class="_2_QraFYR_0">老师，请问下，如果我要选redis不同版本对应的镜像，该怎么查找？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 在docker hub上搜Redis，页面上有详细的列表和说明</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-07 15:51:46</div>
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
  <div class="_2_QraFYR_0">上传镜像，是不是相当于git，会把我的各种服务，还有参数配置都上传上去？<br>那是不是可以直接当git用了？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 只要是打包进镜像的都会上传到docker hub里，相当于是上传压缩包到GitHub。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-04 14:43:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/34/1c/9dd631fd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小宇哥_程振宇</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，前面的内容已经读完了，希望快快更新呦，有点迫不及待了呢~</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 学习的事情急不得，要有一个吸收沉淀的过程。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-03 15:49:49</div>
  </div>
</div>
</div>
</li>
</ul>