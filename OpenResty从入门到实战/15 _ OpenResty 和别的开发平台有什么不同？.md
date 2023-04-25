<audio title="15 _ OpenResty 和别的开发平台有什么不同？" src="https://static001.geekbang.org/resource/audio/9d/4c/9d7d6d7ab40c4c0794113ddff514804c.mp3" controls="controls"></audio> 
<p>你好，我是温铭。</p><p>上一模块中， 你已经学习了 OpenResty 的两个基石：NGINX 和 LuaJIT，相信你已经摩拳擦掌，准备开始学习 OpenResty 提供的 API 了吧？</p><p>不过，别着急，在这之前，你还需要再花一点儿时间，来熟悉下 OpenResty 的原理和基本概念。</p><h2>原理</h2><p>在前面的 LuaJIT 内容中，你已经见过下面这个架构图：</p><p><img src="https://static001.geekbang.org/resource/image/14/f0/14ab2f0c81c170234ab739cb700a62f0.png?wh=1146*828" alt=""></p><p>这里我再详细解释一下。</p><p>OpenResty 的 master 和 worker 进程中，都包含一个 LuaJIT VM。在同一个进程内的所有协程，都会共享这个 VM，并在这个 VM 中运行 Lua 代码。</p><p>而在同一个时间点上，每个 worker 进程只能处理一个用户的请求，也就是只有一个协程在运行。看到这里，你可能会有一个疑问：NGINX 既然能够支持 C10K （上万并发），不是需要同时处理一万个请求吗？</p><p>当然不是，NGINX 实际上是通过 epoll 的事件驱动，来减少等待和空转，才尽可能地让 CPU 资源都用于处理用户的请求。毕竟，只有单个的请求被足够快地处理完，整体才能达到高性能的目的。如果采用的是多线程模式，让一个请求对应一个线程，那么在 C10K 的情况下，资源很容易就会被耗尽的。</p><p>在 OpenResty 层面，Lua 的协程会与 NGINX 的事件机制相互配合。如果 Lua 代码中出现类似查询 MySQL 数据库这样的 I/O 操作，就会先调用 Lua 协程的 yield 把自己挂起，然后在 NGINX 中注册回调；在 I/O 操作完成（也可能是超时或者出错）后，再由 NGINX 回调 resume 来唤醒 Lua 协程。这样就完成了 Lua 协程和 NGINX 事件驱动的配合，避免在 Lua 代码中写回调。</p><!-- [[[read_end]]] --><p>我们可以来看下面这张图，描述了这整个流程。其中，<code>lua_yield</code> 和 <code>lua_resume</code> 都属于 Lua 提供的 <code>lua_CFunction</code>。</p><p><img src="https://static001.geekbang.org/resource/image/fa/34/fae1008edb43c7476cf2f20da9928234.png?wh=1728*764" alt=""></p><p>另外一个方面，如果 Lua 代码中没有 I/O 或者 sleep 操作，比如全是密集的加解密运算，那么 Lua 协程就会一直占用 LuaJIT VM，直到处理完整个请求。</p><p>下面我提供了 <code>ngx.sleep</code> 的一段源码，可以帮你更清晰理解这一点。 这段代码位于 <code>ngx_http_lua_sleep.c</code> 中，你可以在 <code>lua-nginx-module</code> 项目的  <a href="https://github.com/openresty/lua-nginx-module/tree/master/src">src 目录</a>中找到它。</p><p>在<code>ngx_http_lua_sleep.c</code> 中，我们可以看到 sleep 函数的具体实现。你需要先通过 C 函数 <code>ngx_http_lua_ngx_sleep</code>，来注册 <code>ngx.sleep</code> 这个 Lua API：</p><pre><code>void
ngx_http_lua_inject_sleep_api(lua_State *L)
{
     lua_pushcfunction(L, ngx_http_lua_ngx_sleep);
     lua_setfield(L, -2, &quot;sleep&quot;);
}
</code></pre><p>下面便是 sleep 的主函数，这里我只摘取了几行主要的代码：</p><pre><code>static int ngx_http_lua_ngx_sleep(lua_State *L)
{
    coctx-&gt;sleep.handler = ngx_http_lua_sleep_handler;
    ngx_add_timer(&amp;coctx-&gt;sleep, (ngx_msec_t) delay);
    return lua_yield(L, 0);
}
</code></pre><p>你可以看到：</p><ul>
<li>这里先增加了 <code>ngx_http_lua_sleep_handler</code> 这个回调函数；</li>
<li>然后调用 <code>ngx_add_timer</code> 这个 NGINX 提供的接口，向 NGINX 的事件循环中增加一个定时器；</li>
<li>最后使用 <code>lua_yield</code> 把 Lua 协程挂起，把控制权交给 NGINX 的事件循环。</li>
</ul><p>当 sleep 操作完成后， <code>ngx_http_lua_sleep_handler</code> 这个回调函数就被触发了。它里面调用了 <code>ngx_http_lua_sleep_resume</code>, 并最终使用 <code>lua_resume</code> 唤醒了 Lua 协程。更具体的调用过程，你可以自己去代码里面检索，这里我就不展开描述了。</p><p><code>ngx.sleep</code> 只是最简单的一个示例，不过通过对它的剖析，你可以看出 <code>lua-nginx-module</code> 模块的基本原理。</p><h2>基本概念</h2><p>分析完原理之后，让我们一起温故而知新，回忆下 OpenResty 中<strong>阶段</strong>和<strong>非阻塞</strong>这两个重要的概念。</p><p>OpenResty 和 NGINX 一样，都有阶段的概念，并且每个阶段都有自己不同的作用：</p><ul>
<li><code>set_by_lua</code>，用于设置变量；</li>
<li><code>rewrite_by_lua</code>，用于转发、重定向等；</li>
<li><code>access_by_lua</code>，用于准入、权限等；</li>
<li><code>content_by_lua</code>，用于生成返回内容；</li>
<li><code>header_filter_by_lua</code>，用于应答头过滤处理；</li>
<li><code>body_filter_by_lua</code>，用于应答体过滤处理；</li>
<li><code>log_by_lua</code>，用于日志记录。</li>
</ul><p>当然，如果你的代码逻辑并不复杂，都放在 rewrite 或者 content 阶段执行，也是可以的。</p><p>不过需要注意，OpenResty 的 API 是有阶段使用限制的。每一个 API 都有一个与之对应的使用阶段列表，如果你超范围使用就会报错。这与其他的开发语言有很大的不同。</p><p>举个例子，这里我还是以 <code>ngx.sleep</code> 为例。通过查阅文档，我知道它只能用于下面列出的上下文中，并不包括 log 阶段：</p><pre><code>context: rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.*, ssl_certificate_by_lua*, ssl_session_fetch_by_lua*_
</code></pre><p>而如果你不知道这一点，在它不支持的 log 阶段使用 sleep 的话:</p><pre><code>location / {
    log_by_lua_block {
        ngx.sleep(1)
     }
}
</code></pre><p>在 NGINX 的错误日志中，就会出现 error 级别的提示：</p><pre><code>[error] 62666#0: *6 failed to run log_by_lua*: log_by_lua(nginx.conf:14):2: API disabled in the context of log_by_lua*
stack traceback:
    [C]: in function 'sleep'
</code></pre><p>所以，在你使用 API 之前，一定记得要先查阅文档，确定其能否在代码的上下文中使用。</p><p>复习了阶段的概念后，我们再来回顾下非阻塞。首先明确一点，由 OpenResty 提供的所有 API，都是非阻塞的。</p><p>我继续以 sleep 1 秒这个需求为例来说明。如果你要在 Lua 中实现它，你需要这样做：</p><pre><code>function sleep(s)
   local ntime = os.time() + s
   repeat until os.time() &gt; ntime
end
</code></pre><p>因为标准 Lua 没有直接的 sleep 函数，所以这里我用一个循环，来不停地判断是否达到指定的时间。这个实现就是阻塞的，在 sleep 的这一秒钟时间内，Lua 正在做无用功，而其他需要处理的请求，只能在一边傻傻地等待。</p><p>不过，要是换成 <code>ngx.sleep(1)</code> 来实现的话，根据上面我们分析过的源码，在这一秒钟的时间内，OpenResty 依然可以去处理其他请求（比如 B 请求），当前请求（我们叫它 A 请求）的上下文会被保存起来，并由 NGINX 的事件机制来唤醒，再回到 A 请求，这样 CPU 就一直处于真正的工作状态。</p><h2>变量和生命周期</h2><p>除了这两个重要概念外，<strong>变量的生命周期</strong>，也是 OpenResty 开发中容易出错的地方。</p><p>前面说过，在 OpenResty 中，我推荐你把所有变量都声明为局部变量，并用 luacheck 和 lua-releng 这样的工具来检测全局变量。这其实对于模块来说也是一样的，比如下面这样的写法：</p><pre><code>local ngx_re = require &quot;ngx.re&quot;
</code></pre><p>其实，在 OpenResty 中，除了 <code>init_by_lua</code> 和 <code>init_worker_by_lua</code> 这两个阶段外，其余阶段都会设置一个隔离的全局变量表，以免在处理过程中污染了其他请求。即使在这两个可以定义全局变量的阶段，你也应该尽量避免去定义全局变量。</p><p>通常来说，试图用全局变量来解决的问题，其实更应该用模块的变量来解决，而且还会更加清晰。下面是一个模块中变量的示例：</p><pre><code>local _M = {}

_M.color = {
      red = 1,
      blue = 2,
      green = 3
  }

  return _M
</code></pre><p>我在一个名为 hello.lua 的文件中定义了一个模块，模块包含了 color 这个 table。然后，我又在 nginx.conf 中增加了对应的配置：</p><pre><code>location / {
    content_by_lua_block {
        local hello = require &quot;hello&quot;
        ngx.say(hello.color.green)
     }
}
</code></pre><p>这段配置会在 content 阶段中 require 这个模块，并把 green 的值作为 http 请求返回体打印出来。</p><p>你可能会好奇，模块变量为什么这么神奇呢？</p><p>实际上，在同一 worker 进程中，模块只会被加载一次；之后这个 worker 处理的所有请求，就可以共享模块中的数据了。我们说“全局”的数据很适合封装在模块内，是因为 OpenResty 的 worker 之间完全隔离，所以每个 worker 都会独立地对模块进行加载，而模块的数据也不能跨越 worker。</p><p>至于应该如何处理 worker 之间需要共享的数据，我会留到后面的章节来讲解，这里你先不必深究。</p><p>不过，这里也有一个很容易出错的地方，那就是<strong>访问模块变量的时候，你最好保持只读，而不要尝试去修改，不然在高并发的情况下会出现 race</strong>。这种 bug 依靠单元测试是无法发现的，它在线上偶尔会出现，并且很难定位。</p><p>举个例子，模块变量 green 当前的值是 3，而你在代码中做了加 1 的操作，那么现在 green 的值是 4 吗？不一定，它可能是 4，也可能是 5 或者是 6。因为在对模块变量进行写操作的时候，OpenResty 并不会加锁，这时就会产生竞争，模块变量的值就会被多个请求同时更新。</p><p>说完了全局变量、局部变量和模块变量，最后我们再来讲讲跨阶段的变量。</p><p>有些情况下，我们需要的是跨越阶段的、可以读写的变量。而像我们熟悉的 NGINX 中 <code>$host</code>、<code>$scheme</code> 等变量，虽然满足跨越阶段的条件，但却无法做到动态创建，你必须先在配置文件中定义才能使用它们。比如下面这样的写法：</p><pre><code>location /foo {
      set $my_var ; # 需要先创建 $my_var 变量
      content_by_lua_block {
          ngx.var.my_var = 123
      }
  }
</code></pre><p>OpenResty 提供了 <code>ngx.ctx</code>，来解决这类问题。它是一个 Lua table，可以用来存储基于请求的 Lua 数据，且生存周期与当前请求相同。我们来看下官方文档中的这个示例：</p><pre><code>location /test {
      rewrite_by_lua_block {
          ngx.ctx.foo = 76
      }
      access_by_lua_block {
          ngx.ctx.foo = ngx.ctx.foo + 3
      }
      content_by_lua_block {
          ngx.say(ngx.ctx.foo)
      }
  }
</code></pre><p>你可以看到，我们定义了一个变量 <code>foo</code>，存放在 <code>ngx.ctx</code> 中。这个变量跨越了 rewrite、access 和 content 三个阶段，最终在 content 阶段打印出了值，并且是我们预期的 79。</p><p>当然，<code>ngx.ctx</code> 也有自己的局限性：</p><ul>
<li>比如说，使用 <code>ngx.location.capture</code> 创建的子请求，会有自己独立的 <code>ngx.ctx</code> 数据，和父请求的 <code>ngx.ctx</code> 互不影响；</li>
<li>再如，使用 <code>ngx.exec</code> 创建的内部重定向，会销毁原始请求的 <code>ngx.ctx</code>，重新生成空白的 <code>ngx.ctx</code>。</li>
</ul><p>这两个局限，在官方文档中都有详细的<a href="https://github.com/openresty/lua-nginx-module#ngxctx">代码示例</a>，如果你有兴趣可以自行查阅。</p><h2>写在最后</h2><p>最后，我再多说几句。这节课，我们学习的是 OpenResty 的原理和几个重要的概念，不过，你并不需要背得滚瓜烂熟，毕竟，这些概念总是在和实际需求以及代码结合在一起时，才会变得有意义并生动起来。</p><p>不知道你是如何理解的呢？欢迎留言和我一起探讨，也欢迎你把这篇文章分享给你的同事、朋友，我们一起交流，一起进步。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/dd/09/feca820a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>helloworld</span>
  </div>
  <div class="_2_QraFYR_0">说一段比较绕口的废话来理解协程、异步：<br>openresty中的一个worker进程中只有一个线程，该线程是轻量化的线程，也可以叫微线程，也可以称为协程，所谓协程就是处理协程对象的线程，所谓协程对象就是异步化的代码片段，所谓异步化就是遇到网络I&#47;O磁盘I&#47;O等耗时I&#47;O操作时使CPU停止等待I&#47;O，马上去处理下一个协程对象，避免白白浪费CPU时间，这就是非阻塞，具体说应该叫CPU非阻塞，当之前的协程对象I&#47;O完成后，CPU在回去继续处理该协程对象剩余的代码，当然了继续处理过程中遇到I&#47;O阻塞还会继续中断跳出去执行其他协程对象，往复...</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-28 11:06:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/5a/37/8775d714.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jackstraw</span>
  </div>
  <div class="_2_QraFYR_0">老师说的，“访问模块变量的时候，你最好保持只读，而不要尝试去修改，不然在高并发的情况下会出现 race”这个问题有点疑惑。既然每个worker同一时间只会有一个请求在进行中，那么同一时间只会有一个请求正在操作这个模块变量啊？怎么会出现race？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果修改的中途有 yield 操作，就可能会有 race。没有 yield 的话自然没有。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-12 14:56:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/TTPicMEZ5s1zKiaKYecIljicRV9dibITPM0958W3VuNHXlTQic2Dj6XYGibF7dqrG3JWr0LMx7hY9zXGzuvTDNGw8KAA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>阳光梦</span>
  </div>
  <div class="_2_QraFYR_0">老师，基于worker的缓存模块，对数据改写操作时候也有竞争？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-28 22:19:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/14/28/9e3edef0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>DC</span>
  </div>
  <div class="_2_QraFYR_0">openresty的日志路径是如何配置的，这个问题一直困扰我，ngx.log打印不同级别的日志，文件位置可以配置吗，默认是error.log吗</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-15 21:27:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/fb/3a/3dc7c61c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>꧁༺༻꧂</span>
  </div>
  <div class="_2_QraFYR_0">温大，请教个问题，我运行实例，require(&quot;hello&quot;) 提示找不到，错误信息如下：<br> &#47;home&#47;jin&#47;geektime&#47;lua&#47;import.lua:1: module &#39;data&#39; not found:<br>	no field package.preload[&#39;data&#39;]<br>	no file &#39;&#47;usr&#47;local&#47;openresty&#47;site&#47;lualib&#47;data.ljbc&#39;<br>	no file &#39;&#47;usr&#47;local&#47;openresty&#47;site&#47;lualib&#47;data&#47;init.ljbc&#39;<br>	no file &#39;&#47;usr&#47;local&#47;openresty&#47;lualib&#47;data.ljbc&#39;<br>	no file &#39;&#47;usr&#47;local&#47;openresty&#47;lualib&#47;data&#47;init.ljbc&#39;<br>	no file &#39;&#47;usr&#47;local&#47;openresty&#47;site&#47;lualib&#47;data.lua&#39;<br>	no file &#39;&#47;usr&#47;local&#47;openresty&#47;site&#47;lualib&#47;data&#47;init.lua&#39;<br>	no file &#39;&#47;usr&#47;local&#47;openresty&#47;lualib&#47;data.lua&#39;<br>	no file &#39;&#47;usr&#47;local&#47;openresty&#47;lualib&#47;data&#47;init.lua&#39;<br>	no file &#39;.&#47;data.lua&#39;<br>	no file &#39;&#47;usr&#47;local&#47;openresty&#47;luajit&#47;share&#47;luajit-2.1.0-beta3&#47;data.lua&#39;<br>	no file &#39;&#47;usr&#47;local&#47;share&#47;lua&#47;5.1&#47;data.lua&#39;<br>	no file &#39;&#47;usr&#47;local&#47;share&#47;lua&#47;5.1&#47;data&#47;init.lua&#39;<br>	no file &#39;&#47;usr&#47;local&#47;openresty&#47;luajit&#47;share&#47;lua&#47;5.1&#47;data.lua&#39;<br>	no file &#39;&#47;usr&#47;local&#47;openresty&#47;luajit&#47;share&#47;lua&#47;5.1&#47;data&#47;init.lua&#39;<br>	no file &#39;&#47;usr&#47;local&#47;openresty&#47;site&#47;lualib&#47;data.so&#39;<br>	no file &#39;&#47;usr&#47;local&#47;openresty&#47;lualib&#47;data.so&#39;<br>	no file &#39;.&#47;data.so&#39;<br>	no file &#39;&#47;usr&#47;local&#47;lib&#47;lua&#47;5.1&#47;data.so&#39;<br>	no file &#39;&#47;usr&#47;local&#47;openresty&#47;luajit&#47;lib&#47;lua&#47;5.1&#47;data.so&#39;<br>	no file &#39;&#47;usr&#47;local&#47;lib&#47;lua&#47;5.1&#47;loadall.so&#39;<br>stack traceback:<br>coroutine 0:<br>	[C]: in function &#39;require&#39;<br>	&#47;home&#47;jin&#47;geektime&#47;lua&#47;import.lua:1: in main chunk, client: 127.0.0.1, server: , request: &quot;GET &#47;import HTTP&#47;1.1&quot;, host: &quot;127.0.0.1:8080&quot;<br><br>但是 &quot;.&#47;data.lua&quot; 明明是存在的<br>--------<br>$ cat lua&#47;data.lua<br>local _M = {}<br><br>_M.color = {<br>	red = 1,<br>	blue = 2,<br>	green = 3<br>}<br><br>return _M<br><br>------<br>$ cat lua&#47;import.lua<br>local hello = require (&quot;data&quot;)<br>ngx.say(hello.color.green)<br><br>------<br>$ cat conf&#47;nginx.conf<br>......<br>location &#47;import {<br>	content_by_lua_file lua&#47;import.lua;<br>}<br>....<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-09 21:36:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/eb/4f/6a97b1cd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>猪小擎</span>
  </div>
  <div class="_2_QraFYR_0">同步和异步，阻塞和非阻塞，这都是没有好和坏的区别的，以为异步非阻塞比同步阻塞好，很片面，而且不对。如果硬要说阻塞和非阻塞哪个更好一点，应该是阻塞比非阻塞更好一点。用一个cpu空转的代码来做阻塞的例子，这也太黑阻塞了吧。空转不等于阻塞，空转用100%CPU，阻塞不消耗CPU。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-06 13:51:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/kicrNmbT02m0ibb2vSibTuUzLz0YiaAicgMkya8GJibXO5neocoXqnrtTblVKVZNibAsYib312sXImYLETzI9WlVv5oozA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_xiaoer</span>
  </div>
  <div class="_2_QraFYR_0">老师说的，“访问模块变量的时候，你最好保持只读，而不要尝试去修改，不然在高并发的情况下会出现 race”这个问题有点疑惑。既然每个worker同一时间只会有一个请求在进行中，那么同一时间只会有一个请求正在操作这个模块变量啊？怎么会出现race？<br><br>------<br>作者回复: 如果修改的中途有 yield 操作，就可能会有 race。没有 yield 的话自然没有。<br><br>-----<br>同问，这里的yield指什么？是lua协程主动yield吗，还是其它条件触发被动yield的？能不能给个yield导致race的例子？谢谢！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-13 12:07:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/14/28/9e3edef0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>DC</span>
  </div>
  <div class="_2_QraFYR_0">老师 openresty中把请求体作为日志打印到本地 能指定路径和文件名 这样一个需求 我翻阅了各种说明 始终没有好的做法 其他技术栈很容易实现 这方面能给个指引吗</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-15 22:57:46</div>
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
  <div class="_2_QraFYR_0">老师，编译openresty时候出现 error SSE 4.2 not found这个错误，网上有说cpu太老 如何处理这样的问题，还是可以忽略</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: OpenResty 的一些特性用到了 SSE 4.2 指令集，这个只能换机器了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-12 16:20:45</div>
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
  <div class="_2_QraFYR_0">location配置如下：<br>location &#47; {<br>            access_by_lua_file lua&#47;access.lua;<br>            content_by_lua_file lua&#47;content.lua;<br>        }<br><br>access_by_lua_file内容如下:<br>ngx.var.res = &quot;test&quot;<br>content_by_lua_file内容如下：<br>ngx.say(ngx.var.res)<br>老师你好，这样子也说可以做到变量跨阶段读写的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-05 10:53:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/4e/82/67ece8b8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>上善若水</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，看你的最佳实践中，遇到一个问题请教一下，在引入第三方resty库时，我根据配置请求不是返回的百度首页，是一个<br>&lt;!DOCTYPE html&gt;<br>&lt;!--STATUS OK--&gt;<br>是什么问题？多谢。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是否有更详细的信息？可以放到 gist 里面</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-02 00:53:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/48/8f/7ecd4eed.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>FF</span>
  </div>
  <div class="_2_QraFYR_0">温老师能否针对 OR 进程和协程的交互和他俩的协作关系展开讲讲。<br><br>比如在同一个时间点上，每个 worker 进程只能处理一个用户的请求，也就是只有一个协程在运行。那如果协程阻塞，这个 worker 进程是否也在阻塞状态 ？协程是不是由 OR 的 worker 进程来创建 ，单个请求完后就消亡 ？<br><br>或者这方面有什么资料可以推荐下吗？<br><br>感谢。盼复。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 其实不是只有一个协程，是有一个父协程和子协程。这方面推荐 codedump 老师的源码分析：https:&#47;&#47;www.codedump.info&#47;post&#47;20190501-lua-stream&#47;，内容质量很高</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-01 09:32:38</div>
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
  <div class="_2_QraFYR_0">文中这样一段话：“其余阶段都会设置一个隔离的全局变量表，以免在处理过程中污染了其他请求”。前半句似乎是要表达防止在处理的各个阶段互相干扰。后半句这样说，是因为同一个worker进程中的不同的请求其实是共享全局变量的是吧，也是同一个进程里的全局变量本来就是公用的。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-30 12:32:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/TTPicMEZ5s1zKiaKYecIljicRV9dibITPM0958W3VuNHXlTQic2Dj6XYGibF7dqrG3JWr0LMx7hY9zXGzuvTDNGw8KAA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>阳光梦</span>
  </div>
  <div class="_2_QraFYR_0">讲的好👍</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-28 22:14:27</div>
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
  <div class="_2_QraFYR_0">老师，init 阶段创建的全局变量，使用content 阶段的api进行修改，是否安全？<br>比如实现无reload的配置推送：<br>-- 在init_by_lua 定义全局的配置文件table<br>CONFIG = {a = 1};<br><br>-- 在access_by_lua中使用全局的CONFIG实现业务逻辑<br>if CONFIG[&#39;a&#39;] = 1 then<br>    ngx.exit(ngx.HTTP_FORBIDDEN) <br>end <br><br>-- 为了动态跟新全局配置，提供一个API，发送get请求更新 &#47;set-config?a =2<br>location = &#47;set-config {<br>      content_by_lua_block &#39;<br>          local args = ngx.req.get_uri_args()<br>          CONFIG[&#39;a&#39;] = args[&#39;a&#39;]<br>     &#39;<br>}</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-28 16:39:55</div>
  </div>
</div>
</div>
</li>
</ul>