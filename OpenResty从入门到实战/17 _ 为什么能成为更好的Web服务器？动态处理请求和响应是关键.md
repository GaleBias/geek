<audio title="17 _ 为什么能成为更好的Web服务器？动态处理请求和响应是关键" src="https://static001.geekbang.org/resource/audio/48/6e/4896687fb4708633494c82751b8f1b6e.mp3" controls="controls"></audio> 
<p>你好，我是温铭。经过前面内容的铺垫后， 相信你已经对 OpenResty 的概念和如何学习它有了基本的认识。今天这节课，我们来看一下 OpenResty 如何处理终端请求和响应。</p><p>虽然 OpenResty 是基于 NGINX 的 Web 服务器，但它与 NGINX 却有本质的不同：NGINX 由静态的配置文件驱动，而 OpenResty 是由 Lua API 驱动的，所以能提供更多的灵活性和可编程性。</p><p>下面，就让我来带你领略 Lua API 带来的好处吧。</p><h2>API 分类</h2><p>首先我们要知道，OpenResty 的 API 主要分为下面几个大类：</p><ul>
<li>处理请求和响应；</li>
<li>SSL 相关；</li>
<li>shared dict；</li>
<li>cosocket；</li>
<li>处理四层流量；</li>
<li>process 和 worker；</li>
<li>获取 NGINX 变量和配置；</li>
<li>字符串、时间、编解码等通用功能。</li>
</ul><p>这里，我建议你同时打开 OpenResty 的 Lua API 文档，对照着其中的 <a href="https://github.com/openresty/lua-nginx-module/#nginx-api-for-lua">API 列表</a>  ，看看是否能和这个分类联系起来。</p><p>OpenResty 的 API 不仅仅存在于 lua-nginx-module 项目中，也存在于 lua-resty-core 项目中，比如 ngx.ssl、ngx.base64、ngx.errlog、ngx.process、ngx.re.split、ngx.resp.add_header、ngx.balancer、ngx.semaphore、ngx.ocsp 这些 API 。</p><!-- [[[read_end]]] --><p>而对于不在 lua-nginx-module 项目中的 API，你需要单独 require 才能使用。举个例子，比如你想使用 split 这个字符串分割函数，就需要按照下面的方法来调用：</p><pre><code>$ resty -e 'local ngx_re = require &quot;ngx.re&quot;
 local res, err = ngx_re.split(&quot;a,b,c,d&quot;, &quot;,&quot;, nil, {pos = 5})
 print(res)
 '
</code></pre><p>当然，这可能会给你带来一个困惑：在 lua-nginx-module 项目中，明明有 ngx.re.sub、ngx.re.find 等好几个 ngx.re 开头的 API，为什么单单是 ngx.re.split 这个 API ，需要 require 后才能使用呢？</p><p>事实上，在前面 lua-resty-core 章节中，我们也提到过，OpenResty 新的 API 都是通过 FFI 的方式在 <code>lua-rety-core</code> 仓库中实现的，所以难免就会存在这种割裂感。自然，我也很期待lua-nginx-module 和 lua-resty-core 这两个项目以后可以合并，彻底解决此类问题。</p><h2>请求</h2><p>接下来，我们具体了解下OpenResty 是如何处理终端请求和响应的。先来看下处理请求的 API，不过，以 ngx.req 开头的 API 有 20 多个，该怎么下手呢？</p><p>我们知道，HTTP 请求报文由三部分组成：请求行、请求头和请求体，所以下面我就按照这三部分来对 API 做介绍。</p><h3>请求行</h3><p>首先是请求行，HTTP 的请求行中包含请求方法、URI 和 HTTP 协议版本。在 NGINX 中，你可以通过内置变量的方式，来获取其中的值；而在 OpenResty 中对应的则是 <code>ngx.var.*</code> 这个 API。我们来看两个例子。</p><ul>
<li><code>$scheme</code> 这个内置变量，在 NGINX 中代表协议的名字，是 “http” 或者 “https”；而在 OpenResty 中，你可以通过 <code>ngx.var.scheme</code> 来返回同样的值。</li>
<li><code>$request_method</code> 代表的是请求的方法，“GET”、“POST” 等；而在 OpenResty 中，你可以通过 <code>ngx.var. request_method</code> 来返回同样的值。</li>
</ul><p>至于完整的 NGINX 内置变量列表，你可以访问 NGINX 的官方文档来获取：<a href="http://nginx.org/en/docs/http/ngx_http_core_module.html#variables">http://nginx.org/en/docs/http/ngx_http_core_module.html#variables</a>。</p><p>那么问题就来了：既然可以通过<code>ngx.var.*</code> 这种返回变量值的方法，来得到请求行中的数据，为什么 OpenResty 还要单独提供针对请求行的 API 呢？</p><p>这其实是很多方面因素的综合考虑结果：</p><ul>
<li>首先是对性能的考虑。<code>ngx.var</code> 的效率不高，不建议反复读取；</li>
<li>也有对程序友好的考虑，<code>ngx.var</code> 返回的是字符串，而非 Lua 对象，遇到获取 args 这种可能返回多个值的情况，就不好处理了；</li>
<li>另外是对灵活性的考虑，绝大部分的 <code>ngx.var</code> 是只读的，只有很少数的变量是可写的，比如 <code>$args</code> 和 <code>limit_rate</code>，可很多时候，我们会有修改 method、URI 和 args 的需求。</li>
</ul><p>所以， OpenResty 提供了多个专门操作请求行的 API，它们可以对请求行进行改写，以便后续的重定向等操作。</p><p>我们先来看下，如何通过 API 来获取 HTTP 协议版本号。OpenResty 的 API <code>ngx.req.http_version</code> 和 NGINX 的 <code>$server_protocol</code> 变量的作用一样，都是返回 HTTP 协议的版本号。不过这个 API 的返回值是数字格式，而非字符串，可能的值是 2.0、1.0、1.1 和 0.9，如果结果不在这几个值的范围内，就会返回 nil。</p><p>再来看下获取请求行中的请求方法。刚才我们提到过，<code>ngx.req.get_method</code> 和 NGINX 的 <code>$request_method</code> 变量的作用、返回值一样，都是字符串格式的方法名。</p><p>但是，改写当前 HTTP 请求方法的 API，也就是 <code>ngx.req.set_method</code>，它接受的参数格式却并非字符串，而是内置的数字常量。比如，下面的代码，把请求方法改写为 POST：</p><pre><code>ngx.req.set_method(ngx.HTTP_POST)
</code></pre><p>为了验证 <code>ngx.HTTP_POST</code> 这个内置常量，确实是数字而非字符串，你可以打印出它的值，看输出是否为 8：</p><pre><code>$ resty -e 'print(ngx.HTTP_POST)'
</code></pre><p>这样一来，get 方法的返回值为字符串，而set 方法的输入值却是数字，就很容易让你在写代码的时候想当然了。如果是 set 时候传值混淆的情况还好，API 会崩溃报出 500 的错误；但如果是下面这种判断逻辑的代码：</p><pre><code>if (ngx.req.get_method() == ngx.HTTP_POST) then
    -- do something
 end
</code></pre><p>这种代码是可以正常运行的，不会报出任何错误，甚至在 code review 时也很难发现。不幸的是，我就犯过类似的错误，对此记忆犹新：当时已经经过了两轮 code review，还有不完整的测试案例尝试覆盖，然而，最终还是因为线上环境异常才追踪到了这里。</p><p>碰到这类情况，除了自己多小心，或者再多一层封装外，并没有什么有效的方法来解决。平常你在设计自己的业务 API 时，也可以多做一些这方面的考虑，尽量保持 get、set 方法的参数格式一致，即使这会牺牲一些性能。</p><p>另外，在改写请求行的方法中，还有 <code>ngx.req.set_uri</code> 和 <code>ngx.req.set_uri_args</code> 这两个 API，可以用来改写 uri 和 args。我们来看下这个 NGINX 配置：</p><pre><code> rewrite ^ /foo?a=3? break;
</code></pre><p>那么，如何用等价的 Lua API 来解决呢？答案就是下面这两行代码。</p><pre><code>ngx.req.set_uri_args(&quot;a=3&quot;)
ngx.req.set_uri(&quot;/foo&quot;)
</code></pre><p>其实，如果你看过官方文档，就会发现 <code>ngx.req.set_uri</code> 还有第二个参数：jump，默认是 false。如果设置为 true，就等同于把 rewrite 指令的 flag 设置为 <code>last</code>，而非上面示例中的 <code>break</code>。</p><p>不过，我个人并不喜欢 rewrite 指令的 flag 配置，看不懂也记不住，远没有代码来的直观和好维护。</p><h3>请求头</h3><p>再来看下和请求头有关的 API。我们知道，HTTP 的请求头是 <code>key : value</code> 格式的，比如：</p><pre><code>Accept: text/css,*/*;q=0.1
Accept-Encoding: gzip, deflate, br
</code></pre><p>在OpenResty 中，你可以使用 <code>ngx.req.get_headers</code> 来解析和获取请求头，返回值的类型则是 table：</p><pre><code>local h, err = ngx.req.get_headers()
 
  if err == &quot;truncated&quot; then
      -- one can choose to ignore or reject the current request here
  end
 
  for k, v in pairs(h) do
      ...
  end
</code></pre><p>这里默认返回前 100 个 header，如果请求头超过了 100 个，就会返回 <code>truncated</code> 的错误信息，由开发者自己决定如何处理。你可能会好奇为什么会有这样的处理，这一点先留个悬念，在后面安全漏洞的章节中我会提到。</p><p>不过，需要注意的是，OpenResty 并没有提供获取某一个指定请求头的 API，也就是没有 <code>ngx.req.header['host']</code> 这种形式。如果你有这样的需求，那就需要借助 NGINX 的变量 <code>$http_xxx</code> 来实现了，那么在 OpenResty 中，就是 <code>ngx.var.http_xxx</code> 这样的获取方式。</p><p>看完了获取请求头，我们再来看看应该如何改写和删除请求头，这两种操作的 API 其实都很直观：</p><pre><code>ngx.req.set_header(&quot;Content-Type&quot;, &quot;text/css&quot;)
ngx.req.clear_header(&quot;Content-Type&quot;)
</code></pre><p>当然，官方文档中也提到了其他方法来删除请求头，比如把 header 的值设置为 nil等，但为了代码更加清晰的考虑，我还是推荐统一用 <code>clear_header</code> 来操作。</p><h3>请求体</h3><p>最后来看请求体。出于性能考虑，OpenResty 不会主动读取请求体的内容，除非你在 nginx.conf 中强制开启了 <code>lua_need_request_body</code> 指令。此外，对于比较大的请求体，OpenResty 会把内容保存在磁盘的临时文件中，所以读取请求体的完整流程是下面这样的：</p><pre><code>ngx.req.read_body()
local data = ngx.req.get_body_data()
if not data then
    local tmp_file = ngx.req.get_body_file()
     -- io.open(tmp_file)
     -- ...
 end
</code></pre><p>这段代码中有读取磁盘文件的 IO 阻塞操作。你应该根据实际情况来调整 <code>client_body_buffer_size</code> 配置的大小（64 位系统下默认是 16 KB），尽量减少阻塞的操作；你也可以把 <code>client_body_buffer_size</code> 和 <code>client_max_body_size</code> 配置成一样的，完全在内存中来处理，当然，这取决于你内存的大小和处理的并发请求数。</p><p>另外，请求体也可以被改写，<code>ngx.req.set_body_data</code> 和 <code>ngx.req.set_body_file</code> 这两个API，分别接受字符串和本地磁盘文件做为输入参数，来完成请求体的改写。不过，这类操作并不常见，你可以查看文档来获取更详细的内容。</p><h2>响应</h2><p>处理完请求后，我们就需要发送响应返回给客户端了。和请求报文一样，响应报文也由几个部分组成，即状态行、响应头和响应体。同样的，接下来我会按照这三部分来介绍相应的API。</p><h3>状态行</h3><p>状态行中，我们主要关注的是状态码。在默认情况下，返回的 HTTP 状态码是 200，也就是 OpenResty 中内置的常量 <code>ngx.HTTP_OK</code>。但在代码的世界中，处理异常情况的代码总是占比最多的。</p><p>如果你检测了请求报文，发现这是一个恶意的请求，那么你需要终止请求:</p><pre><code> ngx.exit(ngx.HTTP_BAD_REQUEST)
</code></pre><p>不过，OpenResty 的 HTTP 状态码中，有一个特别的常量：<code>ngx.OK</code>。当 <code>ngx.exit(ngx.OK)</code> 时，请求会退出当前处理阶段，进入下一个阶段，而不是直接返回给客户端。</p><p>当然，你也可以选择不退出，只使用 <code>ngx.status</code> 来改写状态码，比如下面这样的写法：</p><pre><code>ngx.status = ngx.HTTP_FORBIDDEN
</code></pre><p>如果你想了解更多的状态码常量，可以从<a href="https://github.com/openresty/lua-nginx-module/#http-status-constants">文档</a>中查询到。</p><h3>响应头</h3><p>说到响应头，其实，你有两种方法来设置它。第一种是最简单的：</p><pre><code>ngx.header.content_type = 'text/plain'
ngx.header[&quot;X-My-Header&quot;] = 'blah blah'
ngx.header[&quot;X-My-Header&quot;] = nil -- 删除
</code></pre><p>这里的 ngx.header 保存了响应头的信息，可以读取、修改和删除。</p><p>第二种设置响应头的方法是 <code>ngx_resp.add_header</code> ，来自 lua-resty-core 仓库，它可以增加一个头信息，用下面的方法来调用：</p><pre><code>local ngx_resp = require &quot;ngx.resp&quot;
ngx_resp.add_header(&quot;Foo&quot;, &quot;bar&quot;)
</code></pre><p>与第一种方法的不同之处在于，add header 不会覆盖已经存在的同名字段。</p><h3>响应体</h3><p>最后看下响应体，在 OpenResty 中，你可以使用 <code>ngx.say</code> 和 <code>ngx.print</code> 来输出响应体：</p><pre><code>ngx.say('hello, world')
</code></pre><p>这两个 API 的功能是一致的，唯一的不同在于， <code>ngx.say</code> 会在最后多一个换行符。</p><p>为了避免字符串拼接的低效，<code>ngx.say / ngx.print</code> 不仅支持字符串作为参数，也支持数组格式：</p><pre><code>$ resty -e 'ngx.say({&quot;hello&quot;, &quot;, &quot;, &quot;world&quot;})'
 hello, world
</code></pre><p>这样在 Lua 层面就跳过了字符串的拼接，把这个它不擅长的事情丢给了 C 函数去处理。</p><h2>写在最后</h2><p>到此，让我们回顾下今天的内容。我们按照请求报文和响应报文的内容，依次介绍了与之相关的 OpenResty API。你可以看得出来，和 NGINX 的指令相比，OpenResty API更加灵活和强大。</p><p>那么，在你处理 HTTP 请求时，OpenResty 提供的 Lua API 是否足够满足你的需求呢？欢迎留言一起探讨，也欢迎你把这篇文章分享给你的同事、朋友，我们一起交流，一起进步。</p><p></p>
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
  <div class="_2_QraFYR_0">老师，文中的这句“OpenResty 的 HTTP 状态码中，有一个特别的常量：ngx.OK。当 ngx.exit(ngx.OK) 时，请求会退出当前处理阶段，进入下一个阶段，而不是直接返回给客户端。”<br>ngx.OK应该不能算是HTTP状态码吧，它对应的值是0；<br>我下面的理解对不对：<br>ngx.exit(ngx.OK)、ngx.exit(ngx.ERROR)和ngx.exit(ngx.DECLINED)时，请求会退出当前处理阶段，进入下一个阶段；<br>而当ngx.exit(ngx.HTTP_*)以ngx.HTTP_*各种HTTP状态码作为参数时，会直接响应给客户端。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: ngx.ok 确实不是http状态码，它是 OpenResty 中的一个常量，值是0.<br>ngx.exit 的官方文档正好可以解答你的问题：<br>When status &gt;= 200 (i.e., ngx.HTTP_OK and above), it will interrupt the execution of the current request and return status code to nginx.<br><br>When status == 0 (i.e., ngx.OK), it will only quit the current phase handler (or the content handler if the content_by_lua* directive is used) and continue to run later phases (if any) for the current request.<br><br>不过，里面并没有提到对于ngx.exit(ngx.ERROR)和ngx.exit(ngx.DECLINED)是如何处理的，我们可以自己来做个测试：<br>location &#47;lua {<br>        rewrite_by_lua &quot;ngx.exit(ngx.ERROR)&quot;;<br>        echo hello;<br>    }<br>访问这个 location，可以看到 http 响应码为空，响应体也是空。并没有引入下一个执行阶段。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-08 00:52:15</div>
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
  <div class="_2_QraFYR_0">利用ngx.exit返回非200时，请求会退出当前请求处理阶段，然后直接返回给客户端。在测试的时候发现一个特点是无论ngx.exit返回什么应答码，log_by_lua*阶段都会执行。本来以为直接返回给客户端，不会执行log阶段的，因为log阶段也是处理流程中的一个阶段。其实log阶段是在返回给客户端之后才会执行的一个阶段是吧。仔细想来也应该是这样：）</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-03 18:27:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/c0/11/3d59119f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小强</span>
  </div>
  <div class="_2_QraFYR_0">老师，openresty 可以做api鉴权吗？现在项目中网关用的spring cloud gateway和jwt，只做鉴权和代理，性能较差，适合往openresty迁移吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 当然可以，现在流行的 API 网关不少都是基于 OpenResty 的。JWT 的可以参考这里：https:&#47;&#47;github.com&#47;iresty&#47;apisix&#47;blob&#47;master&#47;doc&#47;plugins&#47;jwt-auth-cn.md</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-30 20:20:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/pJJpwhFRpXm1NFJ8HuiapckC8IibS0eHY99XqnAUtpicHXDdiajK3ObthbnenMTwRDGgvfv86LPEjKq06wzd9o8sHQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_c0ea3b</span>
  </div>
  <div class="_2_QraFYR_0">老师，请问作为反向代理时，获取上游服务器的响应，一般是通过ngx.arg[1]这样吗？对于较大的响应，ngx.arg[1]可能需要获取多次，性能上可能不是最优。像一般软WAF在处理响应进行规制匹配时，是怎么获取响应的呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-04 19:55:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/64/9b/d1ab239e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>J.Smile</span>
  </div>
  <div class="_2_QraFYR_0">看到这里，才觉得接近了实战。^_^</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-05 17:17:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKsz8j0bAayjSne9iakvjzUmvUdxWEbsM9iasQ74spGFayIgbSE232sH2LOWmaKtx1WqAFDiaYgVPwIQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>2xshu</span>
  </div>
  <div class="_2_QraFYR_0">老师请教个 ngx.location.capture问题，看lua api官方文档如下：<br>This API function (as well as ngx.location.capture_multi) always buffers the whole response body of the subrequest in memory.<br>想问下，这个缓存的response body，是parent request结束就会释放掉吗？有什么办法可以验证我的猜想吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，在 OpenResty 中 Lua 层面的内存都是自动管理的。<br>是否可以使用反证法，如果在结束的时候没有释放，那么内存就是一直增长。用 wrk 或者 ab 发送几百个请求，然后看看内存的占用。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-05 15:31:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/2xoGmvlQ9qfSibVpPJyyaEriavuWzXnuECrJITmGGHnGVuTibUuBho43Uib3Y5qgORHeSTxnOOSicxs0FV3HGvTpF0A/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>psoracle</span>
  </div>
  <div class="_2_QraFYR_0">一直没搞明白为什么luajit提供的shell命令行中不能使用local 本地变量，可以定义声明，但使用时还是尝试从global index中查找，所以找不到。<br>![](http:&#47;&#47;wimgss.oss-cn-hangzhou.aliyuncs.com&#47;2019-07-04-024647.png)</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 官方 LuaJIT 并没有这个问题，OpenResty 带的 LuaJIT 确实会如此。我也不太清楚这么做的原因，可能是个 bug，你可以到https:&#47;&#47;github.com&#47;openresty&#47;luajit2提交 issue 询问下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-04 10:48:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/f0/21/104b9565.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小飞哥 ‍超級會員</span>
  </div>
  <div class="_2_QraFYR_0">看到这里，已经不知道应该如何操作了。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-03 23:43:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/2xoGmvlQ9qfSibVpPJyyaEriavuWzXnuECrJITmGGHnGVuTibUuBho43Uib3Y5qgORHeSTxnOOSicxs0FV3HGvTpF0A/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>psoracle</span>
  </div>
  <div class="_2_QraFYR_0">如果在resty中熟悉处理http请求的api，如ngx.req.http_version，我是使用resty启动一个mini http server，代码中打印当前访问的http 信息，使用curl -XMETHOD来调用<br>![](http:&#47;&#47;wimgss.oss-cn-hangzhou.aliyuncs.com&#47;2019-07-03-034013.png)</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: content by lua 这个是 nginx 的指令，不能在 resty 里面调用</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-03 11:42:01</div>
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
  <div class="_2_QraFYR_0">对于不同contenttype的请求体，有什么需要特别注意的吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 并没有特别要注意的地方</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-03 08:28:18</div>
  </div>
</div>
</div>
</li>
</ul>