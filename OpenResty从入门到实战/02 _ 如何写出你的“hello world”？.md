<audio title="02 _ 如何写出你的“hello world”？" src="https://static001.geekbang.org/resource/audio/0d/0b/0df176a972a52a2555d75a11d2152e0b.mp3" controls="controls"></audio> 
<p>你好，我是温铭。今天起，就要开始我们的正式学习之旅。</p><p>每当我们开始学习一个新的开发语言或者平台，都会从最简单的<code>hello world</code>开始，OpenResty 也不例外。让我们先跳过安装的步骤，直接看下，最简单的 OpenResty 程序是怎么编写和运行的：</p><pre><code>$ resty -e &quot;ngx.say('hello world')&quot;
hello world
</code></pre><p>这应该是你见过的最简单的那种 hello world 代码写法，和 Python 类似：</p><pre><code>$ python -c 'print(&quot;hello world&quot;)'
hello world
</code></pre><p>这背后其实是 OpenResty 哲学的一种体现，代码要足够简洁，也好让你打消“从入门到放弃“的念头。我们今天的内容，就专门围绕着这行代码来展开聊一聊。</p><p>上一节我们讲过，OpenResty 是基于 NGINX 的。那你现在是不是有一个疑问：为什么这里看不到 NGINX 的影子？别着急，我们加一行代码，看看 <code>resty</code>背后真正运行的是什么：</p><pre><code>resty -e &quot;ngx.say('hello world'); ngx.sleep(10)&quot; &amp;
</code></pre><p>我们加了一行 sleep 休眠的代码，让 resty 运行的程序打印出字符串后，并不退出。这样，我们就有机会一探究竟：</p><pre><code>$ ps -ef | grep nginx
501 25468 25462   0  7:24下午 ttys000    0:00.01 /usr/local/Cellar/openresty/''1.13.6.2/nginx/sbin/nginx -p /tmp/resty_AfNwigQVOB/ -c conf/nginx.conf
</code></pre><p>终于看了熟悉的 NGINX 进程。看来，<code>resty</code> 本质上是启动了一个 NGINX 服务，那么<code>resty</code> 又是一个什么程序呢？我先卖个关子，咱后面再讲。</p><p>你的机器上可能还没有安装 OpenResty，所以，接下来，我们先回到开头跳过的安装步骤，把 OpenResty 安装完成后再继续。</p><!-- [[[read_end]]] --><h2>OpenResty 的安装</h2><p>和其他的开源软件一样，OpenResty 的安装有多种方法，比如使用操作系统的包管理器、源码编译或者 docker 镜像。我推荐你优先使用 yum、apt-get、brew 这类包管理系统，来安装 OpenResty。这里我们使用 Mac 系统来做示例：</p><pre><code>brew tap openresty/brew
brew install openresty
</code></pre><p>使用其他操作系统也是类似的，先要在包管理器中添加 OpenResty 的仓库地址，然后用包管理工具来安装。具体步骤，你可以参考<a href="https://openresty.org/en/linux-packages.html">官方文档</a>。</p><p>不过，这看似简单的安装背后，其实有两个问题：</p><ul>
<li>
<p>为什么我不推荐使用源码来安装呢？</p>
</li>
<li>
<p>为什么不能直接从操作系统的官方仓库安装，而是需要先设置另外一个仓库地址？</p>
</li>
</ul><p>对于这两个问题，你不妨先自己想一想。</p><p>这里我想补充一句。在这门课程里面，我会在表象背后提出很多的“为什么”，希望你可以一边学新东西一边思考，结果是否正确并不重要。独立思考在技术领域也是稀缺的，由于每个人技术领域和深度的不同，在任何课程中老师都会不可避免地带有个人观点以及知识的错漏。只有在学习过程中多问几个为什么，融会贯通，才能逐渐形成自己的技术体系。</p><p>很多工程师都有源码的情节，多年前的我也是一样。在使用一个开源项目的时候，我总是希望能够自己手工从源码开始 configure 和 make，并修改一些编译参数，感觉这样做才能最适合这台机器的环境，才能把性能发挥到极致。</p><p>但现实并非如此，每次源码编译，我都会遇到各种诡异的环境问题，磕磕绊绊才能安装好。现在我想明白了，我们的最初目的其实是用开源项目来解决业务需求，不应该浪费时间和环境鏖战，更何况包管理器和容器技术，正是为了帮我们解决这些问题。</p><p>言归正传，给你说说我的看法。使用 OpenResty 源码安装，不仅仅步骤繁琐，需要自行解决 PCRE、OpenSSL 等外部依赖，而且还需要手工对 OpenSSL 打上对应版本的补丁。不然就会在处理 SSL session 时，带来功能上的缺失，比如像<code>ngx.sleep</code>这类会导致 yield 的 Lua API 就没法使用。这部分内容如果你还想深入了解，可以参考[<a href="https://github.com/openresty/lua-nginx-module#ssl_session_fetch_by_lua_block">官方文档</a>]来获取更详细的信息。</p><p>从 OpenResty 自己维护的 OpenSSL [<a href="https://github.com/openresty/openresty-packaging/blob/master/rpm/SPECS/openresty-openssl.spec">打包脚本</a>]中，就可以看到这些补丁。而在 OpenResty 升级 OpenSSL 版本时，都需要重新生成对应的补丁，并进行完整的回归测试。</p><pre><code>Source0: https://www.openssl.org/source/openssl-%{version}.tar.gz

Patch0: https://raw.githubusercontent.com/openresty/openresty/master/patches/openssl-1.1.0d-sess_set_get_cb_yield.patch
Patch1: https://raw.githubusercontent.com/openresty/openresty/master/patches/openssl-1.1.0j-parallel_build_fix.patch
</code></pre><p>同时，我们可以看下 OpenResty 在 CentOS 中的<a href="https://github.com/openresty/openresty-packaging/blob/master/rpm/SPECS/openresty.spec">[打包脚本]</a>，看看是否还有其他隐藏的点：</p><pre><code>BuildRequires: perl-File-Temp
BuildRequires: gcc, make, perl, systemtap-sdt-devel
BuildRequires: openresty-zlib-devel &gt;= 1.2.11-3
BuildRequires: openresty-openssl-devel &gt;= 1.1.0h-1
BuildRequires: openresty-pcre-devel &gt;= 8.42-1
Requires: openresty-zlib &gt;= 1.2.11-3
Requires: openresty-openssl &gt;= 1.1.0h-1
Requires: openresty-pcre &gt;= 8.42-1
</code></pre><p>从这里可以看出，OpenResty 不仅维护了自己的 OpenSSL 版本，还维护了自己的 zlib 和 PCRE 版本。不过后面两个只是调整了编译参数，并没有维护自己的补丁。</p><p>所以，综合这些因素，我不推荐你自行源码编译 OpenResty，除非你已经很清楚这些细节。</p><p>为什么不推荐源码安装，你现在应该已经很清楚了。其实我们在回答第一个问题时，也顺带回答了第二个问题：为什么不能直接从操作系统的官方仓库安装，而是需要先设置另外一个仓库地址？</p><p>这是因为，官方仓库不愿意接受第三方维护的 OpenSSL、PCRE 和 zlib 包，这会导致其他使用者的困惑，不知道选用哪一个合适。另一方面，OpenResty 又需要指定版本的 OpenSSL、PCRE 库才能正常运行，而系统默认自带的版本都比较旧。</p><h2>OpenResty CLI</h2><p>安装完 OpenResty 后，默认就已经把 OpenResty 的 CLI：<code>resty</code> 安装好了。<code>resty</code>是个 1000 多行的 Perl 脚本，之前我们提到过，OpenResty 的周边工具都是 Perl 编写的，这个是由 OpenResty 作者的技术偏好决定的。</p><pre><code>$ which resty
/usr/local/bin/resty
$ head -n 1 /usr/local/bin/resty
 #!/usr/bin/env perl
</code></pre><p><code>resty</code> 的功能很强大，想了解完整的列表，你可以查看<code>resty -h</code>或者[<a href="https://github.com/openresty/resty-cli">官方文档</a>]。下面，我挑两个有意思的功能介绍一下。</p><pre><code>$ resty --shdict='dogs 1m' -e 'local dict = ngx.shared.dogs
                               dict:set(&quot;Tom&quot;, 56)
                               print(dict:get(&quot;Tom&quot;))'
56

</code></pre><p>先来看第一个例子。这个示例结合了 NGINX 配置和 Lua 代码，一起完成了一个共享内存字典的设置和查询。<code>dogs 1m</code> 是 NGINX 的一段配置，声明了一个共享内存空间，名字是 dogs，大小是 1m；在 Lua 代码中用字典的方式使用共享内存。另外还有<code>--http-include</code> 和 <code>--main-include</code>来设置 NGINX 配置文件。所以，上面的例子也可以写为：</p><pre><code>resty --http-conf 'lua_shared_dict dogs 1m;' -e 'local dict = ngx.shared.dogs
                               dict:set(&quot;Tom&quot;, 56)
                               print(dict:get(&quot;Tom&quot;))'
</code></pre><p>OpenResty 世界中常用的调试工具，比如<code>gdb</code>、<code>valgrind</code>、<code>sysetmtap</code>和<code>Mozilla rr</code> ，也可以和 <code>resty</code> 一起配合使用，方便你平时的开发和测试。它们分别对应着 <code>resty</code> 不同的指令，内部的实现其实很简单，就是多套了一层命令行调用。我们以 valgrind 为例：</p><pre><code>$ resty --valgrind  -e &quot;ngx.say('hello world'); &quot;
ERROR: failed to run command &quot;valgrind /usr/local/Cellar/openresty/1.13.6.2/nginx/sbin/nginx -p /tmp/resty_hTFRsFBhVl/ -c conf/nginx.conf&quot;: No such file or directory
</code></pre><p>在后面调试、测试和性能分析的章节，会涉及到这些工具的使用。它们不仅适用于 OpenResty 世界，也是服务端的通用工具，让我们循序渐进地来学习吧。</p><h2>更正式的 hello world</h2><p>最开始我们使用<code>resty</code>写的第一个 OpenResty 程序，没有 master 进程，也不会监听端口。下面，让我们写一个更正式的 hello world。</p><p>写出这样的 OpenResty 程序并不简单，你至少需要三步才能完成：</p><ul>
<li>
<p>创建工作目录；</p>
</li>
<li>
<p>修改 NGINX 的配置文件，把 Lua 代码嵌入其中；</p>
</li>
<li>
<p>启动 OpenResty 服务。</p>
</li>
</ul><p>我们先来创建工作目录。</p><pre><code>mkdir geektime
cd geektime
mkdir logs/ conf/
</code></pre><p>下面是一个最简化的 <code>nginx.conf</code>，在根目录下新增 OpenResty 的<code>content_by_lua</code>指令，里面嵌入了<code>ngx.say</code>的代码：</p><pre><code>events {
    worker_connections 1024;
}

http {
    server {
        listen 8080;
        location / {
            content_by_lua '
                ngx.say(&quot;hello, world&quot;)
            ';
        }
    }
}
</code></pre><p>请先确认下，是否已经把<code>openresty</code>加入到<code>PATH</code>环境中；然后，启动 OpenResty 服务就可以了：</p><pre><code>openresty -p `pwd` -c conf/nginx.conf
</code></pre><p>没有报错的话，OpenResty 的服务就已经成功启动了。你可以打开浏览器，或者使用 curl 命令，来查看结果的返回：</p><pre><code>$ curl -i 127.0.0.1:8080
HTTP/1.1 200 OK
Server: openresty/1.13.6.2
Content-Type: text/plain
Transfer-Encoding: chunked
Connection: keep-alive

hello, world
</code></pre><p>到这里，恭喜你，一个真正的 OpenResty 程序就完成了。</p><h2>总结</h2><p>让我们回顾下今天讲的内容。我们通过一行简单的 <code>hello, world</code> 代码，延展到OpenResty 的安装和 CLI，并在最后启动了 OpenResty 进程，运行了一个真正的后端程序。</p><p>其中， <code>resty</code> 是我们后面会频繁使用到的命令行工具，课程中的演示代码都是用它来运行的，而不是启动后台的 OpenResty 服务。</p><p>更为重要的是，OpenResty 的背后隐藏了非常多的文化和技术细节，它就像漂浮在海面上的一座冰山。我希望能够通过这门课程，给你展示更全面、更立体的 OpenResty，而不仅仅是它对外暴露出来的 API。</p><h2>思考</h2><p>最后，我给你留一个作业题。我们现在的做法，是把 Lua 代码写在 NGINX 配置文件中。不过，如果代码越来越多，那代码的可读性和可维护性就无法保证了。</p><p>你有什么方法来解决这个问题吗？欢迎留言和我分享，也欢迎你把这篇文章转发给你的同事、朋友。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/95/9a/15571548.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_144c1d</span>
  </div>
  <div class="_2_QraFYR_0">先将代码打包成`.lua` 文件<br>使用配置文件指令 `content_by_lua_file` 引用<br>类库代码 如果有 reqiure 的需求<br>可利用 `lua_package_path` 和 `lua_package_cpath` 设定类库加载目录</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-29 02:29:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/56/1a/abb7bfe3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>cylim</span>
  </div>
  <div class="_2_QraFYR_0">既然是基础课，应该提醒我们关闭server。<br>openresty -s quit -p `pwd` -c conf&#47;nginx.conf<br><br><br>把lua代码写在其他文件上，然后带入nginx.conf使用。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 欢迎补充：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-29 09:19:37</div>
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
  <div class="_2_QraFYR_0">编译安装PART1:<br>老师说的对，对于大多数项目来说都没有必要自己折腾编译安装，费时费力。不过有时候自己动手编译安装也是必须的，因为根据具体项目的需要，比如CDN项目，官方的默认编译选项就缺少一些必要的模块，比如ngx_cache_purge模块，如果需要对ipv6做支持，还需要nginx的--with-ipv6编译选项，等等。下面是我根据官方打包文件总结的centos平台下编译安装最新版本的openresty具体流程。centos6下完美运行，7也应该不是问题。给各位小伙伴在需要的时候参考，少走弯路，节省宝贵时间。<br># 安装 pcre<br>wget ftp:&#47;&#47;ftp.csx.cam.ac.uk&#47;pub&#47;software&#47;programming&#47;pcre&#47;pcre-8.42.tar.bz2    <br>tar xjf pcre-8.42.tar.bz2 <br>cd pcre-8.42<br> .&#47;configure --prefix=&#47;usr&#47;local&#47;openresty&#47;pcre \<br> --disable-cpp --enable-jit \<br> --enable-utf --enable-unicode-properties<br>make -j24 V=1 &gt; &#47;dev&#47;stderr<br>make install<br><br>rm -rf &#47;usr&#47;local&#47;openresty&#47;pcre&#47;bin<br>rm -rf &#47;usr&#47;local&#47;openresty&#47;pcre&#47;share<br>rm -f  &#47;usr&#47;local&#47;openresty&#47;pcre&#47;lib&#47;*.la<br>rm -f  &#47;usr&#47;local&#47;openresty&#47;pcre&#47;lib&#47;*pcrecpp*<br>rm -f  &#47;usr&#47;local&#47;openresty&#47;pcre&#47;lib&#47;*pcreposix*<br>rm -rf &#47;usr&#47;local&#47;openresty&#47;pcre&#47;lib&#47;pkgconfig<br><br># 安装zlib<br>cd &#47;usr&#47;local&#47;src<br>wget http:&#47;&#47;www.zlib.net&#47;zlib-1.2.11.tar.xz<br>tar xf zlib-1.2.11.tar.xz <br>cd zlib-1.2.11<br> .&#47;configure --prefix=&#47;usr&#47;local&#47;openresty&#47;zlib<br>make -j24 \<br>CFLAGS=&#39;-O3 -D_LARGEFILE64_SOURCE=1 -DHAVE_HIDDEN -g&#39; \<br>SFLAGS=&#39;-O3 -fPIC -D_LARGEFILE64_SOURCE=1 -DHAVE_HIDDEN -g&#39; &gt; &#47;dev&#47;stderr <br>make install<br>rm -rf &#47;usr&#47;local&#47;openresty&#47;zlib&#47;share&#47;<br>rm -f &#47;usr&#47;local&#47;openresty&#47;zlib&#47;lib&#47;*.la<br>rm -rf &#47;usr&#47;local&#47;openresty&#47;zlib&#47;lib&#47;pkgconfig&#47;<br><br># 安装openssl<br>cd &#47;usr&#47;local&#47;src<br>wget https:&#47;&#47;www.openssl.org&#47;source&#47;openssl-1.1.0j.tar.gz<br>wget https:&#47;&#47;raw.githubusercontent.com&#47;openresty&#47;openresty&#47;master&#47;patches&#47;openssl-1.1.0d-sess_set_get_cb_yield.patch --no-check-certificate<br>wget https:&#47;&#47;raw.githubusercontent.com&#47;openresty&#47;openresty&#47;master&#47;patches&#47;openssl-1.1.0j-parallel_build_fix.patch --no-check-certificate<br>tar zxf openssl-1.1.0j.tar.gz <br>cd openssl-1.1.0j<br>继续见PART2</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-21 19:38:08</div>
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
  <div class="_2_QraFYR_0">编译安装PART3:<br># 最后开始编译安装openresty<br>cd &#47;usr&#47;local&#47;src<br>wget https:&#47;&#47;openresty.org&#47;download&#47;openresty-1.15.8.1.tar.gz # 如果下载出现ssl报错，yum update wget，再下载<br>tar zxf openresty-1.15.8.1.tar.gz<br>cd openresty-1.15.8.1<br>.&#47;configure \<br>    --prefix=&#47;usr&#47;local&#47;openresty \<br>    --with-cc-opt=&quot;-DNGX_LUA_ABORT_AT_PANIC \<br>                -I&#47;usrl&#47;local&#47;openresty&#47;zlib&#47;include \<br>                -I&#47;usr&#47;local&#47;openresty&#47;pcre&#47;include \<br>                -I&#47;usr&#47;local&#47;openresty&#47;openssl&#47;include&quot; \<br>    --with-ld-opt=&quot;-L&#47;usr&#47;local&#47;openresty&#47;zlib&#47;lib \<br>                -L&#47;usr&#47;local&#47;openresty&#47;pcre&#47;lib \<br>                -L&#47;usr&#47;local&#47;openresty&#47;openssl&#47;lib \<br>                -Wl,-rpath,&#47;usr&#47;local&#47;openresty&#47;zlib&#47;lib:&#47;usr&#47;local&#47;openresty&#47;pcre&#47;lib:&#47;usr&#47;local&#47;openresty&#47;openssl&#47;lib&quot; \<br>    --with-pcre-jit \<br>    --without-http_rds_json_module \<br>    --without-http_rds_csv_module \<br>    --without-lua_rds_parser \<br>    --with-stream \<br>    --with-stream_ssl_module \<br>    --with-stream_ssl_preread_module \<br>    --with-http_v2_module \<br>    --without-mail_pop3_module \<br>    --without-mail_imap_module \<br>    --without-mail_smtp_module \<br>    --with-http_stub_status_module \<br>    --with-http_realip_module \<br>    --with-http_addition_module \<br>    --with-http_auth_request_module \<br>    --with-http_secure_link_module \<br>    --with-http_random_index_module \<br>    --with-http_gzip_static_module \<br>    --with-http_sub_module \<br>    --with-http_dav_module \<br>    --with-http_flv_module \<br>    --with-http_mp4_module \<br>    --with-http_gunzip_module \<br>    --with-threads \<br>    --with-luajit-xcflags=&#39;-DLUAJIT_NUMMODE=2 -DLUAJIT_ENABLE_LUA52COMPAT&#39; \<br>    -j24<br>make -j24<br>make install<br><br>rm -rf &#47;usr&#47;local&#47;openresty&#47;luajit&#47;share&#47;man<br>rm -rf &#47;usr&#47;local&#47;openresty&#47;luajit&#47;lib&#47;libluajit-5.1.a<br><br>[END]</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-21 19:40:33</div>
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
  <div class="_2_QraFYR_0">编译安装PART2:<br>patch -p1 &lt; ..&#47;openssl-1.1.0d-sess_set_get_cb_yield.patch <br>patch -p1 &lt; ..&#47;openssl-1.1.0j-parallel_build_fix.patch <br>.&#47;config \<br>    no-threads shared zlib -g \<br>    enable-ssl3 enable-ssl3-method \<br>    --prefix=&#47;usr&#47;local&#47;openresty&#47;openssl \<br>    --libdir=lib \<br>    -I%&#47;usr&#47;local&#47;openresty&#47;zlib&#47;include \<br>    -L%&#47;usr&#47;local&#47;openresty&#47;zlib&#47;lib \<br>    -Wl,-rpath,&#47;usr&#47;local&#47;openresty&#47;zlib&#47;lib:&#47;usr&#47;local&#47;openresty&#47;openssl&#47;lib<br>make -j24<br>make install_sw<br><br>rm -f &#47;usr&#47;local&#47;openresty&#47;openssl&#47;bin&#47;c_rehash<br>rm -rf &#47;usr&#47;local&#47;openresty&#47;openssl&#47;lib&#47;pkgconfig<br><br>继续见PART3</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-21 19:39:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLCuE3s45as7KQgkia9un9Jm3xllqd1tUV0ugKg6Iial7Es6prgHJMpibjfcwdrJApXbhR1SOaibZu09w/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>aaron</span>
  </div>
  <div class="_2_QraFYR_0">我使用yum安装了openresty之后并没有resty工具，我也没发现-p &#39;pwd&#39;的意义何在，-c nginx.conf也启动不起来，最后我是用openresty -c &#47;opt&#47;geektime&#47;conf&#47;nginx.conf启动的。。。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: centos 下需要单独安装：sudo yum install openresty-resty</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-30 19:55:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/cb/98/be4b2e33.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>MOIC💅</span>
  </div>
  <div class="_2_QraFYR_0">openresty -p `pwd` -c .&#47;conf&#47;nginx.conf 启动<br>openresty -p `pwd` -c .&#47;conf&#47;nginx.conf -s stop 停止<br>lsof -i:端口号    查看服务状态</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-05 16:01:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/30/8a/b5ca7286.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>业余草</span>
  </div>
  <div class="_2_QraFYR_0">content_by_lua指令看起来非常怪！没有用{}</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，可以用 content_by_lua_block</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-29 15:57:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/cd/15/e589a84f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>逗鹅冤</span>
  </div>
  <div class="_2_QraFYR_0">老师好，有个问题请教一下<br>openresty -v<br>nginx version: openresty&#47;1.15.8.1<br>which openresty<br>&#47;usr&#47;local&#47;bin&#47;openresty<br>which resty<br>&#47;usr&#47;local&#47;bin&#47;resty<br>这些都没问题<br>which luajit<br>luajit not found<br>luajit为什么没有呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: luajit 的可执行文件并没有被拷贝到&#47;usr&#47;local&#47;bin目录下，所有找不到。这个是为了避免了已经安装的 luajit 冲突，毕竟 OpenResty 自带的 LuaJIT 是自己维护的版本</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-15 16:58:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/11/3e/925aa996.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>HelloBug</span>
  </div>
  <div class="_2_QraFYR_0">温铭老师，你好，有读者问vscode有没有openresty扩展，你说你用的lua扩展，这个是什么意思呢？还有老师你用的是什么IDE呀？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我用的是 vs code，用的是 lua 和 luacheck 两个插件</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-04 18:51:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/11/3e/925aa996.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>HelloBug</span>
  </div>
  <div class="_2_QraFYR_0">温铭老师好，假设操作系统中已经安装openssl&#47;pcre&#47;zlib，使用openresty的仓库地址，然后使用包管理器安装openresty，这个时候操作系统里有几个openssl&#47;pcre&#47;zlib呢？只有一个的话，是不是openresy维护的openssl&#47;pcre&#47;zlib?如果是的话，升级或者说安装操作系统中的更新版本的openssl（不是升级openresy维护的openssl）,能否升级成功呢？如果升级成功，openresty执行的时候是否会出错呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: openresty 维护的 openssl&#47;pcre&#47;zlib 都会安装在 &#47;usr&#47;local&#47;openresty&#47; 目录下，并不会冲突。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-04 17:24:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/b4/f1/61cd0653.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>moshufenmo</span>
  </div>
  <div class="_2_QraFYR_0">请问老师，开发openresty使用什么IDE? 一直在用sublime，但是lua文件一多，相互间引用关系就很难查看</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 没有什么特别好用的，我用的是微软的 vscode</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-29 23:29:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/bb/c9/37924ad4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>天天向上</span>
  </div>
  <div class="_2_QraFYR_0">mac brew安装貌似很费劲 网上很多方法都报错</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: brew 安装也需要先指定 OpenResty 的仓库地址，具体请查看openresty.org 的文档</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-29 08:50:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Leo</span>
  </div>
  <div class="_2_QraFYR_0">看了第一页就买了课，只因为文笔，一看就知道是我想要的，文笔简洁明了，准确周全。思路清晰，一看就是有条理的技术天才。<br>技术之佳作，文笔之生动。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-30 16:13:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/56/2d/b685567c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>1978</span>
  </div>
  <div class="_2_QraFYR_0">不知道有没有跟我一样，文章最后的 nginx -p `pwd` -c con&#47;nginx.conf 中的pwd不是password，是当前的目录。可以通过nginx -h 查看 -p命令是什么意思。<br>同时，关于PATH的设置，我的是Mac环境，使用brew安装的openresty，<br>安装路径应该是在:&#47;usr&#47;local&#47;Cellar&#47;openresty&#47;1.19.3.1_1<br>原生nginx配置文件地址为:&#47;usr&#47;local&#47;etc&#47;openresty<br>应该在文件.bash_profile中去设置PATH，<br>如果不设置PATH，也可以使用&#47;usr&#47;local&#47;Cellar&#47;openresty&#47;1.19.3.1_1&#47;nginx -p `pwd` -c conf&#47;nginx.conf</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-10 00:25:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/95/11/eb431e52.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>沈康</span>
  </div>
  <div class="_2_QraFYR_0">因为项目在内部，无法连接互联网<br>只能使用源码安装完成，我的经验是：<br>在官网下载源码：http:&#47;&#47;openresty.org&#47;cn&#47; <br>在github对应版本（history中匹配）的打包文件中查找相关库依赖版本，补丁集，而且打包文件里面，安装完成后删除安装路径都有，稍微改成自己的就可以了：https:&#47;&#47;github.com&#47;openresty&#47;openresty-packaging&#47;tree&#47;master&#47;rpm&#47;SPECS<br><br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-26 22:11:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4c/bd/9e568308.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Jim</span>
  </div>
  <div class="_2_QraFYR_0">windows环境下，是用openresty压缩包的nginx.exe程序来启动的：nginx.exe -p {your_path} -c conf\nginx.conf，在它的readme.txt文件可以看到说明</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-11 10:13:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLMDBq7lqg9ZasC4f21R0axKJRVCBImPKlQF8yOicLLXIsNgsZxsVyN1mbvFOL6eVPluTNgJofwZeA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Run</span>
  </div>
  <div class="_2_QraFYR_0">最近要为公司的Kong开发插件,需要动态创建更新几千条service,router,customer,还特么是并发场景,Kong本身没有批量创建的API,只能手动撸插件加队列了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-10 23:09:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/b6/46/b17cbaff.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>王斌</span>
  </div>
  <div class="_2_QraFYR_0">茅塞顿开！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-31 01:02:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_2e428a</span>
  </div>
  <div class="_2_QraFYR_0">window下执行 resty -e &quot;ngx.say(&#39;hello world&#39;)&quot;报错<br>nginx: [emerg] invalid number of arguments in &quot;env&quot; directive in C:\Users\apple\AppData\Local\Temp\Xeq7Dxxyp4&#47;conf&#47;nginx.conf:22是什么原因呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-02 16:47:00</div>
  </div>
</div>
</div>
</li>
</ul>