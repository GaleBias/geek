<audio title="06 _ OpenResty 中用到的 NGINX 知识" src="https://static001.geekbang.org/resource/audio/83/6c/83a0cd3e6feb685678e9ba02a0c9d46c.mp3" controls="controls"></audio> 
<p>你好，我是温铭。</p><p>通过前面几篇文章的介绍，相信你对 OpenResty 的轮廓已经有了一个大概的认知。下面几节课里，我会带你熟悉下 OpenResty 的两个基石：NGINX 和 LuaJIT。万丈高楼平地起，掌握些这些基础的知识，才能更好地去学习 OpenResty。</p><p>今天我先来讲 NGINX。这里我只会介绍下，OpenResty 中可能会用到的一些 NGINX 基础知识，这些仅仅是 NGINX 很小的一个子集。如果你需要系统和深入学习 NGINX，可以参考陶辉老师的《NGINX 核心知识 100 讲》，这也是极客时间上评价非常高的一门课程。</p><p>说到配置，其实，在 OpenResty 的开发中，我们需要注意下面几点：</p><ul>
<li>要尽可能少地配置 nginx.conf；</li>
<li>避免使用if、set 、rewrite 等多个指令的配合；</li>
<li>能通过 Lua 代码解决的，就别用 NGINX 的配置、变量和模块来解决。</li>
</ul><p>这样可以最大限度地提高可读性、可维护性和可扩展性。</p><p>下面这段 NGINX 配置，就是一个典型的反例，可以说是把配置项当成了代码来使用：</p><pre><code>location ~ ^/mobile/(web/app.htm) {
            set $type $1;
            set $orig_args $args;
            if ( $http_user_Agent ~ &quot;(iPhone|iPad|Android)&quot; ) {
                rewrite  ^/mobile/(.*) http://touch.foo.com/mobile/$1 last;
            }
            proxy_pass http://foo.com/$type?$orig_args;
}
</code></pre><p>这是我们在使用 OpenResty 进行开发时需要避免的。</p><h2><strong>NGINX 配置</strong></h2><p>我们首先来看下 NGINX 的配置文件。NGINX 通过配置文件来控制自身行为，它的配置可以看作是一个简单的 DSL。NGINX 在进程启动的时候读取配置，并加载到内存中。<strong>如果修改了配置文件，需要你重启或者重载 NGINX，再次读取后才能生效</strong>。只有 NGINX 的商业版本，才会在运行时, 以 API 的形式提供部分动态的能力。</p><!-- [[[read_end]]] --><p>我们先来看下面这段配置，里面的内容非常简单，我相信大部分工程师都能看懂：</p><pre><code>worker_processes auto;

pid logs/nginx.pid;
error_log logs/error.log notice;

worker_rlimit_nofile 65535;

events {
    worker_connections 16384;
}

http {
    server {
	listen 80;
	listen 443 ssl;

        location / {
	    proxy_pass https://foo.com;
	    }
    }
}

stream {
    server {
        listen 53 udp;
    }
}
</code></pre><p>不过，即使是简单的配置，背后也涉及到了一些很重要的基础概念。</p><p>第一，每个指令都有自己适用的上下文（Context），也就是NGINX 配置文件中指令的作用域。</p><p>最上层的是 main，里面是和具体业务无关的一些指令，比如上面出现的 worker_processes、pid 和 error_log，都属于 main 这个上下文。另外，上下文是有层级关系的，比如 location 的上下文是 server，server 的上下文是 http，http 的上下文是 main。</p><p>指令不能运行在错误的上下文中，NGINX 在启动时会检测 nginx.conf 是否合法。比如我们把</p><p><code>listen 80;</code>  从 server 上下文换到 main 上下文，然后启动 NGINX 服务，会看到类似这样的报错：</p><pre><code>&quot;listen&quot; directive is not allowed here ......
</code></pre><p>第二，NGINX 不仅可以处理 HTTP 请求 和 HTTPS 流量，还可以处理 UDP 和 TCP 流量。</p><p>其中，七层的放在 HTTP 中，四层的放在 stream中。在 OpenResty 里面， lua-nginx-module 和 stream-lua-nginx-module 分别和这俩对应。</p><p>这里有一点需要注意，<strong>NGINX 支持的功能，OpenResty 并不一定支持，需要看 OpenResty 的版本号</strong>。OpenResty 的版本号是和 NGINX 保持一致的，所以很容易识别。比如 NGINX 在 2018 年 3 月份发布的 1.13.10 版本中，增加了对 gRPC 的支持，但 OpenResty 在 2019 年 4 月份时的最新版本是 1.13.6.2，由此可以推断 OpenResty 还不支持 gRPC。</p><p>上面 nginx.conf 涉及到的配置指令，都在 NGINX 的核心模块 <a href="http://nginx.org/en/docs/ngx_core_module.html">ngx_core_module</a>、<a href="http://nginx.org/en/docs/http/ngx_http_core_module.html">ngx_http_core_module_</a> 和 <a href="http://nginx.org/en/docs/stream/ngx_stream_core_module.html">ngx_stream_core_module_</a> 中，你可以点击这几个链接去查看具体的文档说明。</p><h2><strong>MASTER-WORKER 模式</strong></h2><p>了解完配置文件，我们再来看下 NGINX 的多进程模式。这里我放了一张图来表示，你可以看到，NGINX 启动后，会有一个 Master 进程和多个 Worker 进程（也可以只有一个 Worker 进程，看你如何配置）。</p><p><img src="https://static001.geekbang.org/resource/image/a7/92/a7304c2c8af0e1e6c54819c97611b992.jpg?wh=1164*916" alt=""></p><p>先来说 Master 进程，一如其名，扮演“管理者”的角色，并不负责处理终端的请求。它是用来管理 Worker 进程的，包括接受管理员发送的信号量、监控 Worker 的运行状态。当 Worker 进程异常退出时，Master 进程会重新启动一个新的 Worker 进程。</p><p>Worker 进程则是“一线员工”，用来处理终端用户的请求。它是从 Master 进程 fork 出来的，彼此之间相互独立，互不影响。多进程的模式比 Apache 多线程的模式要先进很多，没有线程间加锁，也方便调试。即使某个进程崩溃退出了，也不会影响其他 Worker 进程正常工作。</p><p>而 OpenResty 在 NGINX Master-Worker 模式的前提下，又增加了独有的特权进程（privileged agent）。这个进程并不监听任何端口，和 NGINX 的 Master 进程拥有同样的权限，所以可以做一些需要高权限才能完成的任务，比如对本地磁盘文件的一些写操作等。</p><p>如果特权进程与 NGINX 二进制热升级的机制互相配合，OpenResty 就可以实现自我二进制热升级的整个流程，而不依赖任何外部的程序。</p><p>减少对外部程序的依赖，尽量在 OpenResty 进程内解决问题，不仅方便部署、降低运维成本，也可以降低程序出错的概率。可以说，OpenResty 中的特权进程、ngx.pipe 等功能，都是出于这个目的。</p><h2><strong>执行阶段</strong></h2><p>执行阶段也是 NGINX 重要的特性，与 OpenResty 的具体实现密切相关。NGINX 有 11 个执行阶段，我们可以从 ngx_http_core_module.h 的源码中看到：</p><pre><code>typedef enum {
    NGX_HTTP_POST_READ_PHASE = 0,

    NGX_HTTP_SERVER_REWRITE_PHASE,

    NGX_HTTP_FIND_CONFIG_PHASE,
    NGX_HTTP_REWRITE_PHASE,
    NGX_HTTP_POST_REWRITE_PHASE,

    NGX_HTTP_PREACCESS_PHASE,

    NGX_HTTP_ACCESS_PHASE,
    NGX_HTTP_POST_ACCESS_PHASE,

    NGX_HTTP_PRECONTENT_PHASE,

    NGX_HTTP_CONTENT_PHASE,

    NGX_HTTP_LOG_PHASE
} ngx_http_phases;
</code></pre><p>如果你想详细了解这 11 个阶段的作用，可以学习陶辉老师的视频课程，或者 NGINX 文档，这里我就不再赘述。</p><p>不过，巧合的是，OpenResty 也有 11 个 <code>*_by_lua</code>指令，它们和 NGINX 阶段的关系如下图所示（图片来自 lua-nginx-module 文档）：</p><p><img src="https://static001.geekbang.org/resource/image/2a/73/2a05cb2a679bd1c81b44508666e70273.png?wh=1005*910" alt=""></p><p>其中，  <code>init_by_lua</code> 只会在 Master 进程被创建时执行，<code>init_worker_by_lua</code> 只会在每个 Worker 进程被创建时执行。其他的 <code>*_by_lua</code> 指令则是由终端请求触发，会被反复执行。</p><p>所以在 init_by_lua 阶段，我们可以预先加载 Lua 模块和公共的只读数据，这样可以利用操作系统的 COW（copy on write）特性，来节省一些内存。</p><p>对于业务代码来说，其实大部分的操作都可以在 content_by_lua 里面完成，但我更推荐的做法，是根据不同的功能来进行拆分，比如下面这样：</p><ul>
<li>set_by_lua：设置变量；</li>
<li>rewrite_by_lua：转发、重定向等；</li>
<li>access_by_lua：准入、权限等；</li>
<li>content_by_lua：生成返回内容；</li>
<li>header_filter_by_lua：应答头过滤处理；</li>
<li>body_filter_by_lua：应答体过滤处理；</li>
<li>log_by_lua：日志记录。</li>
</ul><p>我举一个例子来说明这样拆分的好处。我们假设，你对外提供了很多明文 API，现在需要增加自定义的加密和解密逻辑。那么请问，你需要修改所有 API 的代码吗？</p><pre><code># 明文协议版本
location /mixed {
    content_by_lua '...';       # 处理请求
}
</code></pre><p>当然不用。事实上，利用阶段的特性，我们只需要简单地在 access 阶段解密，在 body filter 阶段加密就可以了，原来 content 阶段的代码是不用做任何修改的：</p><pre><code># 加密协议版本
location /mixed {
    access_by_lua '...';        # 请求体解密
    content_by_lua '...';       # 处理请求，不需要关心通信协议
    body_filter_by_lua '...';   # 应答体加密
}
</code></pre><h2><strong>二进制热升级</strong></h2><p>最后，我来简单说一下 NGINX 的二进制热升级。我们知道，在你修改完 NGINX 的配置文件后，还需要重启才能生效。但在 NGINX 升级自身版本的时候，却可以做到热升级。这看上去有点儿本末倒置，不过，考虑到 NGINX 是从传统静态的负载均衡、反向代理、文件缓存起家的，这倒也可以理解。</p><p>热升级通过向旧的 Master 进程发送 USR2  和 WINCH 信号量来完成。对于这两步，前者的作用，是启动新的 Master 进程；后者的作用，是逐步关闭 Worker 进程。</p><p>执行完这两步后，新的 Master 和新的 Worker 就已经启动了。不过此时，旧的 Master 并没有退出。不退出的原因也很简单，如果你需要回退，依旧可以给旧的 Master 发送 HUP 信号量。当然，如果你已经确定不需要回退，就可以给旧 Master 发送 KILL 信号量来退出。</p><p>至此，大功告成，二进制的热升级就完成了。</p><p>关于二进制升级，我主要就讲这些。如果你想了解这方面更详细的资料，可以查阅<a href="http://nginx.org/en/docs/control.html#upgrade">官方文档</a>继续学习。</p><h2><strong>课外延伸</strong></h2><p>OpenResty 的作者多年前写过一个 <a href="https://openresty.org/download/agentzh-nginx-tutorials-zhcn.html">NGINX 教程</a>，如果你对此感兴趣，可以自己学习下。这里面的内容比较多，即使看不懂也没有关系，并不会影响你学习 OpenResty。</p><h2>写在最后</h2><p>总的来说，在 OpenResty 中用到的都是 Nginx 的基础知识，主要涉及到配置、主从进程、执行阶段等。而<strong>其他能用 Lua 代码解决的，尽量用代码来解决，而非使用Nginx 的模块和配置</strong>，这是在学习 OpenResty 中的一个思路转变。</p><p>最后，我给你留了一道开放的思考题。Nginx 官方支持 NJS，也就是可以用 JS 写控制部分 Nginx 的逻辑，和 OpenResty 的思路很类似。对此，你是怎么看待的呢？</p><p>欢迎留言和我分享，也欢迎你把这篇文章转发给你的同事、朋友。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/11/3e/925aa996.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>HelloBug</span>
  </div>
  <div class="_2_QraFYR_0">温铭老师，你好。看了这篇文章之后，有下面这些疑问，希望老师答疑解惑下。<br>1.nginx和openresty有什么版本对应关系？记得前两个版本号是相同的。<br>2.为什么openresty的版本越来越小？是为了表达有些功能不支持了吗？<br>3.openresty可执行文件是nginx可执行文件的软链接，本能的以为openresty的热升级就是nginx的热升级，openresty的热升级和nginx的热升级不一样吗？<br>4.nginx热升级步骤没有涉及到外部程序，这里说的热升级中依赖的外部程序是指什么呢？<br>5.init_by_lua预先加载模块，在请求的其他阶段就可以直接使用这个模块，这个模块此时相当于是全局变量？还有一个问题是，如果在一个请求的多个阶段重复加载某一模块，这个模块会重复加载，还是只加载一次？<br>6.nginx修改配置文件，需要重新加载；nginx又支持热部署，请问这里的本末倒置怎么个说法？:)</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. OpenResty 的版本号是跟着它所使用的 Nginx 来确定的，比如 OpenResty 的 1.15.8.1，使用的 Nginx 版本号就是 1.15.8，最后的 1 是 OpenResty 自己的小版本号；<br>2. OpenResty 的版本号是往上增加的，不太清楚越来越小是怎么看出来的呢？<br>3. 热升级步骤和 nginx 一致；<br>4. 这里的外部程序是指：你需要一个 nginx 之外的进程给 nginx 本身发送信号量，nginx 才能升级；而 OpenResty 有了特权进程之后，可以自己给master 进程发送信号量；<br>5. 相当于其他 worker 进程都已经加载过这个模块，不用重复加载；一个模块只会被加载一次，不管有多少请求来访问，和阶段无关；<br>6. 二进制热升级是很少用到的功能，但 nginx 支持了热部署；修改配置文件是常用的功能，但却需要 reload 才能生效。没有把常用的功能做到极致，所以我觉得有些本末倒置。<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-08 17:55:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/51/76/bebe6133.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>TomShine</span>
  </div>
  <div class="_2_QraFYR_0">OpenResty 的作者NGINX 教程可以在这个连接 https:&#47;&#47;openresty.net.cn&#47;agentzh-nginx-guide.html 进行学习</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-10 12:06:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/42/c5/7913cdb0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>John</span>
  </div>
  <div class="_2_QraFYR_0">我正在做一个二次验证的风控，请教一个问题，op如何将一整个request序列化存储起来，并且在风控条件达到后，如滑动验证通过，再将其反序列化发送到上游服务？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感觉这个不用序列化存储，你可以在 access 阶段调用风控服务的 API 接口，把 request 内容传过去，等风控返回后，根据结果在决定是发送到上游，还是拒绝。<br>你可以看下本章节 OpenResty 11 个 `*_by_lua` 指令的图片。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-07 15:41:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJrvZAOiahN7YnSviaDANNu1uHIvQEEE8icl9libuibhuIZgEOt28KG2jKYqu6uRNILicibb8jM6icHicUXj0A/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>emen</span>
  </div>
  <div class="_2_QraFYR_0">温铭老师，您好。学习了这篇文章之后，对body_filter_by_lua存在疑问请老师解惑。拟想根据文中案例对返回报文进行加密，但发现body_filter_by_lua存在执行多次的情况，遇到此情况应如何处理？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: body_filter_by_lua 执行多次是正常的，因为响应体可能是 chunked 返回的。所以，如果你要对响应体整体加密的话，就要改为一次性返回，而不是 chunked 模式。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-27 19:14:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/48/fb/eb91160d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>天天~</span>
  </div>
  <div class="_2_QraFYR_0">njs 的意义感觉在部署的时候大幅度简化运维的步骤。njs 或者没有 luajit 的性能，但对比之下，比 lua 的生态环境好太多太多了，js 的生态和入门的容易。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 没错，除了 njs，还有 PHP 嵌入 nginx 的尝试，这些语言的普及度和生态比 Lua 好很多。OpenResty 要加油</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-07 01:30:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/23/fd/22f45eeb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>徐永健</span>
  </div>
  <div class="_2_QraFYR_0">ngx改配置不需要重启啊。。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 需要 reload</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-07 22:48:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/db/d2/e29f8834.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lidashuang</span>
  </div>
  <div class="_2_QraFYR_0">lua好处语言小巧，js优势是生态丰富</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-16 22:32:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>nicknick</span>
  </div>
  <div class="_2_QraFYR_0">是信号（signal）不是信号量（semaphore）</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-29 09:51:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/3f/97/8d7a6460.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>卡卡</span>
  </div>
  <div class="_2_QraFYR_0">可能想吸引更多开发者，如果推广的好，node.js估计没戏了！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-07 19:44:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/76/f5/e3f5bd8d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>宝仔</span>
  </div>
  <div class="_2_QraFYR_0">apache和nginx都是多进程吧！只是apache有预先开启多少个进程或者动态fork进程</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，多谢指正</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-16 16:55:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/45/6e/11cf22de.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>普罗米修斯</span>
  </div>
  <div class="_2_QraFYR_0">老师，openresty可以只升级nginx吗，可以不升级openresty吧……如果有，请指点下，生产环境有漏洞需要我修复下，谢谢了🙏</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 最好不要怎么做，这样无法和开源的主线保持一致，而且不能保证所有功能是正常的。最好的方法是给官方提交 PR，合并到主线去。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-01 16:02:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/48/61/803d5bbf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>nanyun</span>
  </div>
  <div class="_2_QraFYR_0">你好，一直在关注resty，想问一下nginx后面出的unit，和它对比有哪些优劣点。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: OpenResty 可以无痛的替换 nginx，同时提供了 Lua API 来做动态的控制，这个是它最大的优势。nginx unit 是为了微服务出的产品，两个感觉不在一个层面上。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-16 10:45:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/fa/0c/ee743364.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>冲野</span>
  </div>
  <div class="_2_QraFYR_0">问个低级的问题：热升级的时候，已经建立的连接是继续保持吗？是不是因此才保留旧进程？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 保留旧的 master 进程是为了方便回滚。旧的 worker 进程是不保留的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-13 12:33:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/8d/eb/e98af40f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>life_牛</span>
  </div>
  <div class="_2_QraFYR_0">我服务是https的，做反向代理的话，需要在nginx的https模块吧？如果不在nginx 里面配置https证书如何用https的请求访问服务？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你想实现类似 cloudflare 的 Keyless 功能？不在 Nginx 中配置的话，就要增加代码逻辑，并在远端服务器配置，不然 https 握手就失败了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-10 15:29:24</div>
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
  <div class="_2_QraFYR_0">关于看出openresty 版本越来越小<br>文中有这样一段话：<br>比如 NGINX 在 2018 年 3 月份发布的 1.13.10 版本中，增加了对 gRPC 的支持，但 OpenResty 在 2019 年 4 月份时的最新版本是 1.13.6.2，由此可以推断 OpenResty 还不支持 gRPC。<br><br>我把1.13.10看成了是openresty的版本了😂 <br>不好意思，麻烦老师了~</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-09 14:33:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/fe/4b/87219faf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>叫我图图就可以了</span>
  </div>
  <div class="_2_QraFYR_0">好像有点小错误，apache才是那个多进程的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 多谢指正</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-09 00:49:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/fc/48/b5a57b72.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>mrmsl</span>
  </div>
  <div class="_2_QraFYR_0">Nginx NJS 几乎就要变得跟 OpenResty 几乎一样啦！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 至少方向是对的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-08 12:54:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/f9/41/411b1753.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>石仔</span>
  </div>
  <div class="_2_QraFYR_0">js有大批的语法熟悉用户，只要能力够能就能大量实践</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 百花齐放</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-07 16:41:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/27/81/27c9d811.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>高燕军</span>
  </div>
  <div class="_2_QraFYR_0">加解密的那个例子，有一点像nodejs的web框架express中的中间件模式</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-07 11:21:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/54/6d/53fd4a8c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Kinga</span>
  </div>
  <div class="_2_QraFYR_0">二进制热升级和配置文件reload更新，都需要给master发信号，不理解本末倒置的说法。作者是希望nginx能够自动检查到配置文件有变更，然后自动重新加载吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我的意思是配置文件的修改是一个更频繁的操作，而二进制热升级并不频繁。应该优先实现前者的热更新。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-22 09:40:01</div>
  </div>
</div>
</div>
</li>
</ul>