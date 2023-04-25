<audio title="09 _ 为什么 lua-resty-core 性能更高一些？" src="https://static001.geekbang.org/resource/audio/9d/03/9df43767fc424ac63ecff81a69748703.mp3" controls="controls"></audio> 
<p>你好，我是温铭。</p><p>前面两节课我们说了，Lua 是一种嵌入式开发语言，核心保持了短小精悍，你可以在 Redis、NGINX 中嵌入 Lua，来帮助你更灵活地完成业务逻辑。同时，Lua 也可以调用已有的 C 函数和数据结构，避免重复造轮子。</p><p>在 Lua 中，你可以用 Lua C API 来调用 C 函数，而在 LuaJIT 中还可以使用 FFI。对 OpenResty 而言：</p><ul>
<li>在核心的 <code>lua-nginx-module</code> 中，调用 C 函数的 API，都是使用 Lua C API 来完成的；</li>
<li>而在 <code>lua-resty-core</code> 中，则是把 <code>lua-nginx-module</code> 已有的部分 API，使用 FFI 的模式重新实现了一遍。</li>
</ul><p>看到这里你估计纳闷了：为什么要用 FFI 重新实现一遍？</p><p>别着急，让我们以 <a href="https://github.com/openresty/lua-nginx-module#ngxdecode_base64">ngx.base64_decode</a> 这个很简单的 API 为例，一起看下 Lua C API 和 FFI 的实现有何不同之处，这样你也可以对它们的性能有个直观的认识。</p><h2>Lua CFunction</h2><p>我们先来看下， <code>lua-nginx-module</code> 中用 Lua C API 是如何实现的。我们在项目的代码中搜索 <code>decode_base64</code>，可以找到它的代码实现在 <code>ngx_http_lua_string.c</code> 中：</p><pre><code>lua_pushcfunction(L, ngx_http_lua_ngx_decode_base64);
lua_setfield(L, -2, &quot;decode_base64&quot;);
</code></pre><!-- [[[read_end]]] --><p>上面的代码看着就头大，不过还好，我们不用深究那两个 <code>lua_</code> 开头的函数，以及它们参数的具体作用，只需要知道一点——这里注册了一个 CFunction：<code>ngx_http_lua_ngx_decode_base64</code>， 而它与 <code>ngx.base64_decode</code> 这个对外暴露的 API 是对应关系。</p><p>我们继续“按图索骥”，在这个 C 文件中搜索 <code>ngx_http_lua_ngx_decode_base64</code>，它定义在文件的开始位置：</p><pre><code>static int ngx_http_lua_ngx_decode_base64(lua_State *L);
</code></pre><p>对于那些能够被 Lua 调用的 C 函数来说，它的接口必须遵循 Lua 要求的形式，也就是 <code>typedef int (*lua_CFunction)(lua_State* L)</code>。它包含的参数是 <code>lua_State</code> 类型的指针 L ；它的返回值类型是一个整型，表示返回值的数量，而非返回值自身。</p><p>它的实现如下（这里我已经去掉了错误处理的代码）：</p><pre><code>static int
 ngx_http_lua_ngx_decode_base64(lua_State *L)
 {
     ngx_str_t p, src;
 
    src.data = (u_char *) luaL_checklstring(L, 1, &amp;src.len);
 
     p.len = ngx_base64_decoded_length(src.len);
 
     p.data = lua_newuserdata(L, p.len);
 
     if (ngx_decode_base64(&amp;p, &amp;src) == NGX_OK) {
         lua_pushlstring(L, (char *) p.data, p.len);
 
     } else {
         lua_pushnil(L);
     }
 
     return 1;
 }
</code></pre><p>这段代码中，最主要的是 <code>ngx_base64_decoded_length</code> 和 <code>ngx_decode_base64</code>， 它们都是 NGINX 自身提供的 C 函数。</p><p>我们知道，用 C 编写的函数，无法把返回值传给 Lua 代码，而是需要通过栈，来传递 Lua 和 C 之间的调用参数和返回值。这也是为什么，会有很多我们一眼无法看懂的代码。同时，这些代码也不能被 JIT 跟踪到，所以对于 LuaJIT 而言，这些操作是处于黑盒中的，没法进行优化。</p><h2>LuaJIT FFI</h2><p>而 FFI 则不同。FFI 的交互部分是用 Lua 实现的，这部分代码可以被 JIT 跟踪到，并进行优化；当然，代码也会更加简洁易懂。</p><p>我们还是以 <code>base64_decode</code>为例，它的 FFI 实现分散在两个仓库中： <code>lua-resty-core</code> 和 <code>lua-nginx-module</code>。我们先来看下前者里面<a href="https://github.com/openresty/lua-resty-core/blob/master/lib/resty/core/base64.lua#L72">实现的代码</a>：</p><pre><code>ngx.decode_base64 = function (s)
     local slen = #s
     local dlen = base64_decoded_length(slen)
     
     local dst = get_string_buf(dlen)
     local pdlen = get_size_ptr()
     local ok = C.ngx_http_lua_ffi_decode_base64(s, slen, dst, pdlen)
     if ok == 0 then
         return nil
     end
     return ffi_string(dst, pdlen[0])
 end
</code></pre><p>你会发现，相比 CFunction，FFI 实现的代码清爽了很多，它具体的实现是 <code>lua-nginx-module</code> 仓库中的<code>ngx_http_lua_ffi_decode_base64</code>，如果你对这里感兴趣，可以自己去查看这个函数的实现，特别简单，这里我就不贴代码了。</p><p>不过，细心的你，是否从上面的代码片段中，发现函数命名的一些规律了呢？</p><p>没错，OpenResty 中的函数都是有命名规范的，你可以通过命名推测出它的用处。比如：</p><ul>
<li><code>ngx_http_lua_ffi_</code> ，是用 FFI 来处理 NGINX HTTP 请求的 Lua 函数；</li>
<li><code>ngx_http_lua_ngx_</code> ，是用 Cfunction 来处理 NGINX HTTP 请求的 Lua 函数；</li>
<li>其他 <code>ngx_</code> 和 <code>lua_</code> 开头的函数，则分别属于 NGINX 和 Lua 的内置函数。</li>
</ul><p>更进一步，OpenResty 中的 C 代码，也有着严格的代码规范，这里我推荐阅读<a href="https://openresty.org/cn/c-coding-style-guide.html">官方的 C 代码风格指南</a>。对于有意学习 OpenResty 的 C 代码并提交 PR 的开发者来说，这是必备的一篇文档。否则，即使你的 PR 写得再好，也会因为代码风格问题被反复评论并要求修改。</p><p>关于 FFI 更多的 API 和细节，推荐你阅读 LuaJIT <a href="http://luajit.org/ext_ffi_tutorial.html">官方的教程</a> 和 <a href="http://luajit.org/ext_ffi_api.html">文档</a>。技术专栏并不能代替官方文档，我也只能在有限的时间内帮你指出学习的路径，少走一些弯路，硬骨头还是需要你自己去啃的。</p><h2>LuaJIT FFI GC</h2><p>使用 FFI 的时候，我们可能会迷惑：在 FFI 中申请的内存，到底由谁来管理呢？是应该我们在 C 里面手动释放，还是 LuaJIT 自动回收呢？</p><p>这里有个简单的原则：LuaJIT 只负责由自己分配的资源；而 <code>ffi.C</code> 是 C 库的命名空间，所以，使用 <code>ffi.C</code> 分配的空间不由 LuaJIT 负责，需要你自己手动释放。</p><p>举个例子，比如你使用 <code>ffi.C.malloc</code> 申请了一块内存，那你就需要用配对的 <code>ffi.C.free</code> 来释放。LuaJIT 的官方文档中有一个对应的示例：</p><pre><code>local p = ffi.gc(ffi.C.malloc(n), ffi.C.free)
 ...
 p = nil -- Last reference to p is gone.
 -- GC will eventually run finalizer: ffi.C.free(p)
</code></pre><p>这段代码中，<code>ffi.C.malloc(n)</code> 申请了一段内存，同时 <code>ffi.gc</code> 就给它注册了一个析构的回调函数 <code>ffi.C.free</code>。这样一来，<code>p</code> 这个 <code>cdata</code> 在被 LuaJIT GC 的时候，就会自动调用 <code>ffi.C.free</code>，来释放 C 级别的内存。而 <code>cdata</code> 是由 LuaJIT 负责 GC的 ，所以上述代码中的 <code>p</code> 会被 LuaJIT 自动释放。</p><p>这里要注意，如果你要在 OpenResty 中申请大块的内存，我更推荐你用 <code>ffi.C.malloc</code> 而不是 <code>ffi.new</code>。原因也很明显：</p><ol>
<li><code>ffi.new</code> 返回的是一个 <code>cdata</code>，这部分内存由 LuaJIT 管理；</li>
<li>LuaJIT GC 的管理内存是有上限的，OpenResty 中的 LuaJIT 并未开启 GC64 选项，所以<strong>单个 worker 内存的上限只有2G</strong>。一旦超过 LuaJIT 的内存管理上限，就会导致报错。</li>
</ol><p><strong>在使用 FFI 的时候，我们还需要特别注意内存泄漏的问题</strong>。不过，凡人皆会犯错，只要是人写的代码，百密一疏，总会出现 bug。那么，有没有什么工具可以检测内存泄漏呢？</p><p>这时候，OpenResty 强大的周边测试和调试工具链就派上用场了。</p><p>我们先来说说测试。在 OpenResty 体系中，我们使用 Valgrind 来检测内存泄漏问题。</p><p>前面课程我们提到过的测试框架 <code>test::nginx</code>，有专门的内存泄漏检测模式去运行单元测试案例集，你只需要设置环境变量 <code>TEST_NGINX_USE_VALGRIND=1</code> 即可。OpenResty 的官方项目在发版本之前，都会在这个模式下完整回归，后面的测试章节中我们再详细介绍。</p><p>而 OpenResty 的 CLI <code>resty</code> 也有 <code>--valgrind</code> 选项，方便你单独运行某段 Lua 代码，即使你没有写测试案例也是没问题的。</p><p>再来看调试工具。</p><p>OpenResty 提供<a href="https://github.com/openresty/stapxx">基于 systemtap 的扩展</a>，来对 OpenResty 程序进行活体的动态分析。你可以在这个项目的工具集中，搜索 <code>gc</code> 这个关键字，会看到 <code>lj-gc</code> 和 <code>lj-gc-objs</code> 这两个工具。</p><p>而对于 core dump 这种离线分析，OpenResty 提供了 <a href="https://github.com/openresty/openresty-gdb-utils">GDB 的工具集</a>，同样你可以在里面搜索 <code>gc</code>，找到 <code>lgc</code>、<code>lgcstat</code> 和 <code>lgcpath</code> 三个工具。</p><p>这些调试工具的具体用法，我们会在后面的调试章节中详细介绍，你先有个印象即可。这样，你遇到内存问题就不会“病急乱投医“，毕竟OpenResty 有专门的工具集，帮你定位和解决这些问题。</p><h2>lua-resty-core</h2><p>从上面的比较中，我们可以看到，FFI 的方式不仅代码更简洁，而且可以被 LuaJIT 优化，显然是更优的选择。其实现实也是如此，实际上，CFunction 的实现方式已经被 OpenResty 废弃，相关的实现也从代码库中移除了。现在新的 API，都通过 FFI 的方式，在 <code>lua-resty-core</code> 仓库中实现。</p><p>在 OpenResty 2019 年 5 月份发布的 1.15.8.1 版本前，<code>lua-resty-core</code> 默认是不开启的，而这不仅会带来性能损失，更严重的是会造成潜在的 bug。所以，我强烈推荐还在使用历史版本的用户，都手动开启 <code>lua-resty-core</code>。你只需要在 <code>init_by_lua</code> 阶段，增加一行代码就可以了：</p><pre><code>require &quot;resty.core&quot;
</code></pre><p>当然，姗姗来迟的 1.15.8.1 版本中，已经增加了 <code>lua_load_resty_core</code> 指令，默认开启了 <code>lua-resty-core</code>。我个人感觉，OpenResty 对于 <code>lua-resty-core</code> 的开启还是过于谨慎了，开源项目应该尽早把类似的功能设置为默认开启。</p><p><code>lua-resty-core</code> 中不仅重新实现了部分 lua-nginx-module 项目中的 API，比如 <code>ngx.re.match</code>、<code>ngx.md5</code> 等，还实现了不少新的 API，比如 ngx.ssl、ngx.base64、ngx.errlog、ngx.process、ngx.re.split、ngx.resp.add_header、ngx.balancer、ngx.semaphore 等等，我们在后面的 OpenResty API 章节中会介绍到。</p><h2>写在最后</h2><p>讲了这么多内容，最后我还是想说，FFI 虽然好，却也并不是性能银弹。它之所以高效，主要原因就是可以被 JIT 追踪并优化。如果你写的 Lua 代码不能被 JIT，而是需要在解释模式下执行，那么 FFI 的效率反而会更低。</p><p>那么到底有哪些操作可以被 JIT，哪些不能呢？怎样才可以避免写出不能被 JIT 的代码呢？下一节我来揭晓这个问题。</p><p>最后，给你留一个需要动手的作业题：你可以找一两个lua-nginx-module 和 lua-resty-core 中都存在的 API，然后性能测试比较一下两者的差异吗？你可以看下 FFI 的性能提升到底有多大。</p><p>欢迎留言和我分享你的思考、收获，也欢迎你把这篇文章分享给你的同事、朋友，一起交流，一起进步。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/27/cc/99349a54.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>黑铁打野王</span>
  </div>
  <div class="_2_QraFYR_0">nginx version: openresty&#47;1.13.6.2<br>1. lua-nginx-module<br>resty -e &quot;local start = ngx.now();<br>for _ =1, 1000000000 do      <br>    ngx.encode_base64(&#39;123456&#39;)<br>end<br>ngx.update_time();<br>ngx.say(ngx.now() - start)&quot;<br>79.530000209808<br>2. lua-resty-core<br>resty -e &quot;require &#39;resty.core&#39;;<br>local start = ngx.now(); <br>for _ =1, 1000000000 do<br>    ngx.encode_base64(&#39;123456&#39;)<br>end<br>ngx.update_time();<br>ngx.say(ngx.now() - start)&quot;<br>8.944000005722<br>一个79.5s 一个8.9s</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 差不多 10 倍的性能差异</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-18 18:30:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/a3/0b/ece949c6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>微风吹了个吹</span>
  </div>
  <div class="_2_QraFYR_0">温老师， 1.15.8的opnresty默认是开启GC64的，是不是可以理解为这里ffi.C.malloc与ffi.new就没啥区别了？官方的更新日志```change: we now enable the GC64 mode by default in our bundled LuaJIT build for x86_64 architectures; this can be disabled using --without-luajit-gc64. Thanks Thibault Charbonnier for the patch```</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-17 10:53:33</div>
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
  <div class="_2_QraFYR_0">假设使用decode_base64函数，lua-resty-core不开启的时候，使用的函数就是lua-nginx-module中实现的，而开启的话就是使用lua-resty-core中实现的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-15 11:29:22</div>
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
  <div class="_2_QraFYR_0">看不懂</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 知道 lua-resty-core 是使用 FFI 实现的 API 就行了，细节看不懂没有关系。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-15 15:27:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/41/38/4f89095b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>写点啥呢</span>
  </div>
  <div class="_2_QraFYR_0">请问老师，ffi和cfunction的性能差异是不是主要是有LuaJIT的实时编译优化带来的？除此之外还有哪些因素导致了这两者之间的性能差异呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 主要是这个原因</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-29 09:59:55</div>
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
  <div class="_2_QraFYR_0">老师，lua-resty-core和lua-nginx-module各自都有哪些API，怎么查呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 要分别查看这两个仓库的文档了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-14 10:18:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_e553fa</span>
  </div>
  <div class="_2_QraFYR_0">沙发。希望多点实例。还有讲讲全局变量申请使用的一些替代方案。不是否定了不给解决办法</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以使用模块中的 top level 局部变量来替代，这个后面会提到。<br>在 OpenResty 中所有变量都要加 local。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-14 08:37:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/9b/a8/6a391c66.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Leon📷</span>
  </div>
  <div class="_2_QraFYR_0">跨语言调用的话，函数调用返回都是通过函数栈来传递嘛，比如go调用C，是不是有点主题无关哈，希望老师解答下</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-02 18:25:37</div>
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
  <div class="_2_QraFYR_0">我是1.15.8.1版本的，用了前面同学给的代码，跑起来，时间相差很少，一个38.133000135422秒，一个37.325000047684秒。这是为啥？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 因为从1.15.8.1版本开始，默认开了 lua-resty-core</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-19 20:36:42</div>
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
  <div class="_2_QraFYR_0">如何理解FFI的交互部分是用lua实现的，这部分代码可以被JIT跟踪到？<br>LuaCFuncion：ngx_http_lua_ngx_decode_base64整个函数都不能被JIT追踪到，而函数ngx.decode_base64可以被JIT追踪到，只有里面的函数C.ngx_http_lua_ffi_decode_base不能被追踪到是吗？也就是C.ngx_http_lua_ffi_decode_base函数可以被JIT编译优化？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: LuaJIT 是运行 Lua 代码的，自然只能 JIT 它其中的 Lua 代码，CFunction 的自然无能为力。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-15 11:27:34</div>
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
  <div class="_2_QraFYR_0">这个不太了解，openresty 工作中要的少，还没上手的，不知道怎么去比较性能</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-14 21:30:44</div>
  </div>
</div>
</div>
</li>
</ul>