<audio title="20 _ 超越 Web 服务器：特权进程和定时任务" src="https://static001.geekbang.org/resource/audio/8e/38/8e18455afada7424d471e975ed42ff38.mp3" controls="controls"></audio> 
<p>你好，我是温铭。</p><p>前面我们介绍了 OpenResty API、共享字典缓存和 cosocket。它们实现的功能，都还在 Nginx 和 Web 服务器的范畴之内，算是提供了开发成本更低、更容易维护的一种实现，提供了可编程的 Web 服务器。</p><p>不过，OpenResty并不满足于此。我们今天就挑选几个，OpenResty 中超越 Web 服务器的功能来介绍一下。它们分别是定时任务、特权进程和非阻塞的 ngx.pipe。</p><h2>定时任务</h2><p>在 OpenResty 中，我们有时候需要在后台定期地执行某些任务，比如同步数据、清理日志等。如果让你来设计，你会怎么做呢？最容易想到的方法，便是对外提供一个 API 接口，在接口中完成这些任务；然后用系统的 crontab 定时调用 curl，来访问这个接口，进而曲线地实现这个需求。</p><p>不过，这样一来不仅会有割裂感，也会给运维带来更高的复杂度。所以， OpenResty 提供了 <code>ngx.timer</code> 来解决这类需求。你可以把<code>ngx.timer</code> ，看作是 OpenResty 模拟的客户端请求，用以触发对应的回调函数。</p><p>其实，OpenResty 的定时任务可以分为下面两种：</p><ul>
<li><code>ngx.timer.at</code>，用来执行一次性的定时任务；</li>
<li><code>ngx.time.every</code>，用来执行固定周期的定时任务。</li>
</ul><!-- [[[read_end]]] --><p>还记得上节课最后我留下的思考题吗？问题是如何突破 <code>init_worker_by_lua</code> 中不能使用 cosocket 的限制，这个答案其实就是 <code>ngx.timer</code>。</p><p>下面这段代码，就是启动了一个延时为 0 的定时任务。它启动了回调函数 <code>handler</code>，并在这个函数中，用 cosocket 去访问一个网站：</p><pre><code>init_worker_by_lua_block {
        local function handler()
            local sock = ngx.socket.tcp()
            local ok, err = sock:connect(“www.baidu.com&quot;, 80)
        end

        local ok, err = ngx.timer.at(0, handler)
    }
</code></pre><p>这样，我们就绕过了 cosocket 在这个阶段不能使用的限制。</p><p>再回到这部分开头时我们提到的的用户需求，<code>ngx.timer.at</code> 并没有解决周期性运行这个需求，在上面的代码示例中，它是一个一次性的任务。</p><p>那么，又该如何做到周期性运行呢？表面上来看，基于 <code>ngx.timer.at</code> 这个API 的话，你有两个选择：</p><ul>
<li>你可以在回调函数中，使用一个 while true 的死循环，执行完任务后 sleep 一段时间，自己来实现周期任务；</li>
<li>你还可以在回调函数的最后，再创建另外一个新的 timer。</li>
</ul><p>不过，在做出选择之前，有一点我们需要先明确下：timer 的本质是一个请求，虽然这个请求不是终端发起的；而对于请求来讲，在完成自己的任务后它就要退出，不能一直常驻，否则很容易造成各种资源的泄漏。</p><p>所以，第一种使用 while true 来自行实现周期任务的方案并不靠谱。第二种方案虽然是可行的，但递归地创建 timer ，并不容易让人理解。</p><p>那么，是否有更好的方案呢？其实，OpenResty 后面新增的 <code>ngx.time.every</code> API，就是专门为了解决这个问题而出现的，它是更加接近 crontab 的解决方案。</p><p>但美中不足的是，在启动了一个 timer 之后，你就再也没有机会来取消这个定时任务了，毕竟<code>ngx.timer.cancel</code> 还是一个 todo 的功能。</p><p>这时候，你就会面临一个问题：定时任务是在后台运行的，并且无法取消；如果定时任务的数量很多，就很容易耗尽系统资源。</p><p>所以，OpenResty 提供了 <code>lua_max_pending_timers</code> 和 <code>lua_max_running_timers</code> 这两个指令，来对其进行限制。前者代表等待执行的定时任务的最大值，后者代表当前正在运行的定时任务的最大值。</p><p>你也可以通过 Lua API，来获取当前等待执行和正在执行的定时任务的值，下面是两个示例：</p><pre><code>content_by_lua_block {
            ngx.timer.at(3, function() end)
            ngx.say(ngx.timer.pending_count())
        }
</code></pre><p>这段代码会打印出 1，表示有 1 个计划任务正在等待被执行。</p><pre><code>content_by_lua_block {
            ngx.timer.at(0.1, function() ngx.sleep(0.3) end)
            ngx.sleep(0.2)
            ngx.say(ngx.timer.running_count())
        }
</code></pre><p>这段代码会打印出 1，表示有 1 个计划任务正在运行中。</p><h2>特权进程</h2><p>接着来看特权进程。我们都知道 Nginx 主要分为 master 进程和 worker 进程，其中，真正处理用户请求的是 worker 进程。我们可以通过 <code>lua-resty-core</code> 中提供的 <code>process.type</code> API ，获取到进程的类型。比如，你可以用 <code>resty</code> 运行下面这个函数：</p><pre><code>$ resty -e 'local process = require &quot;ngx.process&quot;
ngx.say(&quot;process type:&quot;, process.type())'
</code></pre><p>你会看到，它返回的结果不是 <code>worker</code>， 而是 <code>single</code>。这意味 <code>resty</code> 启动的 Nginx 只有 worker 进程，没有 master 进程。其实，事实也是如此。在 <code>resty</code> 的实现中，你可以看到，下面这样的一行配置， 关闭了 master 进程：</p><pre><code>master_process off;
</code></pre><p>而OpenResty 在 Nginx 的基础上进行了扩展，增加了特权进程：privileged agent。特权进程很特别：</p><ul>
<li>它不监听任何端口，这就意味着不会对外提供任何服务；</li>
<li>它拥有和 master 进程一样的权限，一般来说是 <code>root</code> 用户的权限，这就让它可以做很多 worker 进程不可能完成的任务；</li>
<li>特权进程只能在 <code>init_by_lua</code> 上下文中开启；</li>
<li>另外，特权进程只有运行在 <code>init_worker_by_lua</code> 上下文中才有意义，因为没有请求触发，也就不会走到<code>content</code>、<code>access</code> 等上下文去。</li>
</ul><p>下面，我们来看一个开启特权进程的示例：</p><pre><code>init_by_lua_block {
    local process = require &quot;ngx.process&quot;

    local ok, err = process.enable_privileged_agent()
    if not ok then
        ngx.log(ngx.ERR, &quot;enables privileged agent failed error:&quot;, err)
    end
}
</code></pre><p>通过这段代码开启特权进程后，再去启动 OpenResty 服务，我们就可以看到，Nginx 的进程中多了特权进程的身影：</p><pre><code>nginx: master process
nginx: worker process
nginx: privileged agent process
</code></pre><p>不过，如果特权只在 <code>init_worker_by_lua</code> 阶段运行一次，显然不是一个好主意，那我们应该怎么来触发特权进程呢？</p><p>没错，答案就藏在刚刚讲过的知识里。既然它不监听端口，也就是不能被终端请求触发，那就只有使用我们刚才介绍的 <code>ngx.timer</code> ，来周期性地触发了：</p><pre><code>init_worker_by_lua_block {
    local process = require &quot;ngx.process&quot;

    local function reload(premature)
        local f, err = io.open(ngx.config.prefix() .. &quot;/logs/nginx.pid&quot;, &quot;r&quot;)
        if not f then
            return
        end
        local pid = f:read()
        f:close()
        os.execute(&quot;kill -HUP &quot; .. pid)
    end

    if process.type() == &quot;privileged agent&quot; then
         local ok, err = ngx.timer.every(5, reload)
        if not ok then
            ngx.log(ngx.ERR, err)
        end
    end
}
</code></pre><p>上面这段代码，实现了每 5 秒给 master 进程发送 HUP 信号量的功能。自然，你也可以在此基础上实现更多有趣的功能，比如轮询数据库，看是否有特权进程的任务并执行。因为特权进程是 root 权限，这显然就有点儿“后门”程序的意味了。</p><h2>非阻塞的 ngx.pipe</h2><p>最后我们来看非阻塞的 ngx.pipe。刚刚讲过的这个代码示例中，我们使用了 Lua 的标准库，来执行外部命令行，把信号发送给了 master 进程：</p><pre><code>os.execute(&quot;kill -HUP &quot; .. pid) 
</code></pre><p>这种操作自然是会阻塞的。那么，在 OpenResty 中，是否有非阻塞的方法来调用外部程序呢？毕竟，要知道，如果你是把 OpenResty 当做一个完整的开发平台，而非 Web 服务器来使用的话，这就是你的刚需了。</p><p>为此，<code>lua-resty-shell</code> 库应运而生，使用它来调用命令行就是非阻塞的：</p><pre><code>$ resty -e 'local shell = require &quot;resty.shell&quot;
local ok, stdout, stderr, reason, status =
    shell.run([[echo &quot;hello, world&quot;]])
    ngx.say(stdout)
</code></pre><p>这段代码可以算是 hello world 的另外一种写法了，它调用系统的 <code>echo</code> 命令来完成输出。类似的，你可以用 <code>resty.shell</code> ，来替代 Lua 中的 <code>os.execute</code> 调用。</p><p>我们知道，<code>lua-resty-shell</code> 的底层实现，依赖了 <code>lua-resty-core</code> 中的 [<a href="https://github.com/openresty/lua-resty-core/blob/master/lib/ngx/pipe.md">ngx.pipe</a>] API，所以，这个使用 <code>lua-resty-shell</code> 打印出 <code>hello wrold</code> 的示例，改用 <code>ngx.pipe</code> ，可以写成下面这样：</p><pre><code>$ resty -e 'local ngx_pipe = require &quot;ngx.pipe&quot;
local proc = ngx_pipe.spawn({&quot;echo&quot;, &quot;hello world&quot;})
local data, err = proc:stdout_read_line()
ngx.say(data)'
</code></pre><p>这其实也就是 <code>lua-resty-shell</code> 底层的实现代码了。你可以去查看 <code>ngx.pipe</code> 的文档和测试案例，来获取更多的使用方法，这里我就不再赘述了。</p><h2>写在最后</h2><p>到此，今天的主要内容我就讲完了。从上面的几个功能，我们可以看出，OpenResty 在做一个更好用的 Nginx 的前提下，也在尝试往通用平台的方向上靠拢，希望开发者能够尽量统一技术栈，都用 OpenResty 来解决开发需求。这对于运维来说是相当友好的，因为只要部署一个 OpenResty 就可以了，维护成本更低。</p><p>最后，给你留一个思考题。由于可能会存在多个 Nginx worker，那么 timer 就会在每个 worker 中都运行一次，这在大多数场景下都是不能接受的。我们应该如何保证 timer 只能运行一次呢？</p><p>欢迎留言说说你的解决方法，也欢迎你把这篇文章分享给你的同事、朋友，我们一起交流，一起进步。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/da/36/ac0ff6a7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>wusiration</span>
  </div>
  <div class="_2_QraFYR_0">通过shared dict判断互斥，存在正在执行中的timer还没更新执行状态，另一个worker继续去执行；<br>通过判断worker的id指定某个worker执行的方式应该可以实现这一需求</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-13 22:47:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4b/24/8c9eaf7f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Seven</span>
  </div>
  <div class="_2_QraFYR_0">我想在init_worker_by_lua阶段通过timer启动一个websocket客户端一直循环收发数据会不会有问题呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: while True 的循环吗？我的建议是跑一段时间，比如 5 分钟，就退掉，然后启动一个新的客户端。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-14 01:19:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/e2/d8/f0562ede.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>manatee</span>
  </div>
  <div class="_2_QraFYR_0">想请问下特权进程是怎么回事，启动or本身就是普通用户。如何获取root权限呢，另外特权进程的使用场景有哪些可以介绍下吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 特权进程和master 进程的权限一样，如果 master 是普通用户，那特权进程也不可能拿到 root 权限。<br>一般用特权进程来清理日志、重启 OpenResty 自身等需要高权限的任务。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-10 10:03:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/e2/d8/f0562ede.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>manatee</span>
  </div>
  <div class="_2_QraFYR_0">可以通过查看worker id，在指定worker下执行</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，没错</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-10 08:13:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/0d/7c/6f96fca0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>过千帆</span>
  </div>
  <div class="_2_QraFYR_0">timer，运行环境，怎么启动的，为什么会每个worker中运行？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-30 16:13:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/b0/1c/2e30eeb8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>旺旺</span>
  </div>
  <div class="_2_QraFYR_0">老师，在init_worker_by_lua_block里<br>local ok, err = ngx.timer.every(30, reset_server_list)<br>ok, err = ngx.timer.at(0, reget_ssl_certificate)<br>ok, err = ngx.timer.every(3600, reget_ssl_certificate)<br>写了三个定时器，怎么执行的效果不是想象中那样呢？<br>本来是想着一开始的时候就执行一次reget_ssl_certificate，然后每隔1个小时再执行一次reget_ssl_certificate的。<br>现在就算过了30秒reset_server_list也不执行。<br>如果只写一个“local ok, err = ngx.timer.every(30, reset_server_list)”是可以的。<br>意思是ngx.timer.every和ngx.timer.at不能同时混用吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是可以混用的，它们之间不会互相影响的。如果你确定这里有bug，可以整理一个最小的复现代码，给 OpenResty 提交 issue</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-23 20:59:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/b0/1c/2e30eeb8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>旺旺</span>
  </div>
  <div class="_2_QraFYR_0">发现在init_by_lua_block里面开了特权进程后，如果init_worker_by_lua_block又有io.open创建文件操作，那么后面在worker进程里面第一次创建文件时的用户也是root了，然后后面worker里面写文件的时候，会报:<br>failed to open file in append mode. error message: &#47;tmp&#47;wscmd_response.log: Permission denied,<br>因为&#47;tmp&#47;wscmd_response.log一开始是用root身份创建的，后面再用nobody去写的时候，就会报错。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-18 16:40:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/d8/c6/2b2a58cf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>搞怪者😘 😒 😏 👿</span>
  </div>
  <div class="_2_QraFYR_0">这个定时器跟用curl是一样的嘛，怎么实现毫秒级定时，如果在压力很大的环境下，这样的定时器不就会消耗端口资源吗</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-17 10:15:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/b0/1c/2e30eeb8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>旺旺</span>
  </div>
  <div class="_2_QraFYR_0">        local f, err = io.open(ngx.config.prefix() .. &quot;&#47;logs&#47;nginx.pid&quot;, &quot;r&quot;)<br>老师，这个代码也是Lua 的标准库，是不是也是阻塞的，又改采用什么方式优化呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 磁盘 IO 没有什么优化的方法，这里有一个使用 nginx threads pool 来模拟实现 &quot;非阻塞&quot;的方案：https:&#47;&#47;github.com&#47;tokers&#47;lua-io-nginx-module，你可以参考下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-29 15:20:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/99/74/0203bf17.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>英雄</span>
  </div>
  <div class="_2_QraFYR_0">如果不能while true ，那websocket如何等待请求呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以 while true，但是有一个阈值，比如循环 1000 次之后退出循环，重新来一次。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-23 11:41:11</div>
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
  <div class="_2_QraFYR_0">老师好，在讲ngx.timer的时候，说如果在回调函数里使用while true+sleep的方式循环执行任务，因为timer本质是一个请求，上面所说的实现会导致这个请求常驻，这些都是可以理解的，后面说会导致资源的泄露，这个怎么理解呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 泄漏这个词可能不太恰当，就是会导致很多 Lua 或者 C 对象无法得到释放，长期运行会有很多碎片。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-14 18:01:30</div>
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
  <div class="_2_QraFYR_0">local function reload(premature)，老师，这个函数的参数premature是什么意思，在这段代码中有什么用呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: ngx.timer 的文档是有提到它的作用的：<br>the premature argument takes a boolean value indicating whether it is a premature timer expiration or not.<br>Premature timer expiration happens when the Nginx worker process is trying to shut down, as in an Nginx configuration reload triggered by the HUP signal or in an Nginx server shutdown. When the Nginx worker is trying to shut down, one can no longer call ngx.timer.at to create new timers with nonzero delays and in that case ngx.timer.at will return a &quot;conditional false&quot; value and a string describing the error, that is, &quot;process exiting&quot;.<br>简单的说，就是在 Nginx 重启后者关闭的时候，就不要再去创建 timer 了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-14 13:00:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/ee/3c/a2b67971.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Rye</span>
  </div>
  <div class="_2_QraFYR_0">ngx.worker.id() == 0 应该是第一个worker</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，没错</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-11 08:52:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/95/00/087d9e46.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>chundonglinlin</span>
  </div>
  <div class="_2_QraFYR_0">worker slot</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-10 11:01:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/99/f0/d9343049.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>星亦辰</span>
  </div>
  <div class="_2_QraFYR_0">share dict存取执行状态，就可以完成互斥了 </div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-10 09:26:57</div>
  </div>
</div>
</div>
</li>
</ul>