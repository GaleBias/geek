<audio title="44 _ OpenResty 的杀手锏：动态" src="https://static001.geekbang.org/resource/audio/b1/44/b175c2d393a0cdabfec9081ce1afbc44.mp3" controls="controls"></audio> 
<p>你好，我是温铭。</p><p>到目前为止，和 OpenResty 性能相关的内容，我就差不多快要介绍完了。我相信，掌握并灵活运用好这些优化技巧，一定可以让你的代码性能提升一个数量级。今天，在性能优化的最后一个部分，我来讲一讲 OpenResty 中被普遍低估的一种能力：动态。</p><p>让我们先来看下什么是动态，以及它和性能之间有什么样的关系。</p><p>这里的动态，指的是程序可以在运行时、在不重新加载的情况下，去修改参数、配置，乃至修改自身的代码。具体到 Nginx 和 OpenResty 领域，你去修改上游、SSL 证书、限流限速阈值，而不用重启服务，就属于实现了动态。至于动态和性能之间的关系，很显然，如果这几类操作不能动态地完成，那么频繁的 reload Nginx 服务，自然就会带来性能的损耗。</p><p>不过，我们知道，开源版本的 Nginx 并不支持动态特性，所以，你要对上游、SSL 证书做变更，就必须通过修改配置文件、重启服务的方式才能生效。而商业版本的 Nginx Plus 提供了部分动态的能力，你可以用 REST API 来完成更新，但这最多算是一个不够彻底的改良。</p><p>但是，在 OpenResty 中，这些桎梏都是不存在的，动态可以说就是 OpenResty 的杀手锏。你可能纳闷儿，为什么基于 Nginx 的 OpenResty 却可以支持动态呢？原因也很简单，Nginx 的逻辑是通过 C 模块来完成的，而 OpenResty 是通过脚本语言 Lua 来完成的——脚本语言的一大优势，便是运行时可以去做动态地改变。</p><!-- [[[read_end]]] --><h2>动态加载代码</h2><p>下面我们就来看看，如何在 OpenResty 中动态地加载 Lua 代码：</p><pre><code>resty -e 'local s = [[ngx.say(&quot;hello world&quot;)]]
local func, err = loadstring(s)
func()'
</code></pre><p>你没有看错，只要短短的两三行代码，就可以把一个字符串变为一个 Lua 函数，并运行起来。我们进一步仔细看下这几行代码，我来简单解读一下：</p><ul>
<li>首先，我们声明了一个字符串，它的内容是一段合法的 Lua 代码，把 <code>hello world</code> 打印出来；</li>
<li>然后，使用 Lua 中的 <code>loadstring</code> 函数，把字符串对象转为函数对象<code>func</code>；</li>
<li>最后，在函数名的后面加上括号，把 <code>func</code> 执行起来，打印出 <code>hello world</code> 来。</li>
</ul><p>当然，在这段代码的基础之上，我们还可以扩展出更多好玩和实用的功能。接下来，我就带你一起来“尝尝鲜”。</p><h2>功能一：FaaS</h2><p>首先是函数即服务，这是近年来很热门的技术方向，我们看下在 OpenResty 中如何实现。在刚刚的代码中，字符串是一段 Lua 代码，我们还可以把它改成一个 Lua 函数：</p><pre><code>local s = [[
 return function()
     ngx.say(&quot;hello world&quot;)
end
]]
</code></pre><p>我们讲过，函数在 Lua 中是一等公民，这段代码便是返回了一个匿名函数。在执行这个匿名函数时，我们使用 <code>pcall</code> 做了一层保护。<code>pcall</code> 会在保护模式下运行函数，并捕获其中的异常，如果正常就返回 true 和执行的结果，如果失败就返回 false 和错误信息，也就是下面这段代码：</p><pre><code>local func1, err = loadstring(s)
local ret, func = pcall(func1)
</code></pre><p>自然，把上面的两部分结合起来，就会得到完整的、可运行的示例：</p><pre><code>resty -e 'local s = [[
 return function()
    ngx.say(&quot;hello world&quot;)
end
]]
local  func1 = loadstring(s)
local ret, func = pcall(func1)
func()'
</code></pre><p>更深入一步，我们还可以把 <code>s</code> 这个包含函数的字符串，改成可以由用户指定的形式，并加上执行它的条件，这样其实就是 FaaS 的原型了。这里，我提供了一个完整的<a href="https://github.com/apache/incubator-apisix/blob/master/apisix/plugins/serverless.lua">实现</a>，如果你对FaaS感兴趣，想要继续研究，推荐你通过这个链接深入学习。</p><h2>功能二：边缘计算</h2><p>OpenResty 的动态不仅可以用于 FaaS，让脚本语言的动态细化到函数级别，还可以在边缘计算上发挥动态的优势。</p><p>得益于 Nginx 和 LuaJIT 良好的多平台支持特性，OpenResty 不仅能运行在 X86 架构下，对于 ARM 的支持也很完善。同时， OpenResty 支持七层和四层的代理，这样一来，常见的各种协议都可以被 OpenResty 解析和代理，这其中也包括了 IoT 中的几种协议。</p><p>因为这些优势，我们便可以把 OpenResty 的触角，从 API 网关、WAF、web 服务器等服务端的领域，伸展到物联网设备、CDN 边缘节点、路由器等最靠近用户的边缘节点上去。</p><p>这并非只是一种畅想，事实上，OpenResty 已经在上述领域中被大量使用了。以 CDN 的边缘节点为例，OpenResty 的最大使用者 CloudFlare 很早就借助 OpenResty 的动态特性，实现了对于 CDN 边缘节点的动态控制。</p><p>CloudFlare 的做法和上面动态加载代码的原理是类似的，大概可以分为下面几个步骤：</p><ul>
<li>首先，从键值数据库集群中获取到有变化的代码文件，获取的方式可以是后台 timer 轮询，也可以是用“发布-订阅”的模式来监听；</li>
<li>然后，用更新的代码文件替换本地磁盘的旧文件，然后使用 <code>loadstring</code> 和 <code>pcall</code>的方式，来更新内存中加载的缓存；</li>
</ul><p>这样，下一个被处理的终端请求，就会走更新后的代码逻辑。</p><p>当然，实际的应用要比上面的步骤考虑更多的细节，比如版本的控制和回退、异常的处理、网络的中断、边缘节点的重启等，但整体的流程是不变的。</p><p>如果把 CloudFlare 的这套做法，从 CDN 边缘节点挪移到其他边缘的场景下，那我们就可以把很多计算能力动态地赋予边缘节点的设备。这不仅可以充分利用边缘节点的计算能力，也可以让用户请求得到更快速的响应。因为边缘节点会把原始数据处理过后，再汇总给远端的服务器，这就大大减少了数据的传输量。</p><p>不过，要把 FaaS 和边缘计算做好，OpenResty 的动态只是一个良好的基础，你还需要考虑自身周边生态的完善和厂商的加入，这就不仅仅是技术的范畴了。</p><h2>动态上游</h2><p>现在，让我们把思绪拉回到 OpenResty 上来，一起来看如何实现动态上游。<code>lua-resty-core</code> 提供了 <code>ngx.balancer</code> 这个库来设置上游，它需要放到 OpenResty 的 <code>balancer</code> 阶段来运行：</p><pre><code>balancer_by_lua_block {
    local balancer = require &quot;ngx.balancer&quot;
    local host = &quot;127.0.0.2&quot;
    local port = 8080

    local ok, err = balancer.set_current_peer(host, port)
    if not ok then
        ngx.log(ngx.ERR, &quot;failed to set the current peer: &quot;, err)
        return ngx.exit(500)
    end
}
</code></pre><p>我来简单解释一下。<code>set_current_peer</code> 函数，就是用来设置上游的 IP 地址和端口的。不过要注意，这里并不支持域名，你需要使用 <code>lua-resty-dns</code> 库来为域名和 IP 做一层解析。</p><p>不过，<code>ngx.balancer</code> 还比较底层，虽然它有设置上游的能力，但动态上游的实现远非如此简单。所以，在 <code>ngx.balancer</code> 前面还需要两个功能：</p><ul>
<li>一是上游的选择算法，究竟是一致性哈希，还是 roundrobin；</li>
<li>二是上游的健康检查机制，这个机制需要剔除掉不健康的上游，并且需要在不健康的上游变健康的时候，重新把它加入进来。</li>
</ul><p>而OpenResty 官方的 <code>lua-resty-balancer</code> <a href="https://github.com/openresty/lua-resty-balancer">这个库</a>中，则包含了 <code>resty.chash</code> 和 <code>resty.roundrobin</code> 两类算法来完成第一个功能，并且有 <code>lua-resty-upstream-healthcheck</code> 来尝试完成第二个功能。</p><p>不过，这其中还是有两个问题。</p><p>第一点，缺少最后一公里的完整实现。把 <code>ngx.balancer</code>、<code>lua-resty-balancer</code> 和 <code>lua-resty-upstream-healthcheck</code> 整合并实现动态上游的功能，还是需要一些工作量的，这就拦住了大部分的开发者。</p><p>第二点，<code>lua-resty-upstream-healthcheck</code> 的实现并不完整，只有被动的健康检查，而没有主动的健康检查。</p><p>简单解释一下，这里的被动健康检查，是指由终端的请求触发，进而分析上游的返回值来作为健康与否的判断条件。如果没有终端请求，那么上游是否健康就无从得知了。而主动健康检查就可以弥补这个缺陷，它使用 <code>ngx.timer</code> 定时去轮询指定的上游接口，来检测健康状态。</p><p>所以，在实际的实践中，我们通常推荐使用 <code>lua-resty-healthcheck</code> 这个<a href="https://github.com/Kong/lua-resty-healthcheck">库</a>，来完成上游的健康检查。它的优点是包含了主动和被动的健康检查，而且在多个项目中都经过了验证，可靠性更高。</p><p>更进一步，新兴的微服务 API 网关APISIX，在 <code>lua-resty-healthcheck</code> 的基础之上，对动态上游做了完整的实现。我们可以参考它的<a href="https://github.com/iresty/apisix/blob/master/lua/apisix/http/balancer.lua">实现</a>，总共只有 200 多行代码，你可以很轻松地把它剥离出来，放到你的自己的项目中使用。</p><h2>写在最后</h2><p>讲了这么多，最后，给你留一个思考题。关于OpenResty 的动态，你觉得还可以在哪些领域和场景来发挥它的这种优势呢？提醒一下，这个章节中介绍的每部分的内容，你都可以展开来做更详细和深入的分析。</p><p>欢迎留言和我讨论，也欢迎你把这篇文章分享出去，和更多的人一起学习、进步。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/48/8f/7ecd4eed.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>FF</span>
  </div>
  <div class="_2_QraFYR_0">关于CloudFlare 实现的动态加载有个疑问：CloudFlare 在完成新文件替换后，如何用 loadstring 函数完成新文件的加载呢 ？loadstring 只能加载字符串，就算把文件内容转成字符串作为 loadstring 的入参，也不行吧，它的返回值是函数引用。那动态加载 lua 文件&#47;模块，OR 是如何做的 ？温老师能否具体讲讲这块，或者 CloudFlare 具是如何实现的 ？<br><br>如果要重新加载一个 lua 文件&#47;模块，以我目前对 OR 的掌握只能想办法重新执行 require ，但貌似用这种方式做不到，具体实现方式不知是如何 ？<br><br>感谢。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 给你一个更具体的示例：<br>resty -e &#39;local s = [[<br>local ngx = ngx<br>local _M = {}<br>function _M.f()<br>	ngx.say(&quot;hello world&quot;)<br>end<br>return _M<br>]]<br>local lua = loadstring(s)<br>local ret, func = pcall(lua)<br>func.f()&#39;<br><br>这里的 `s` 就是一个完整的 Lua 模块，在发现变化的时候，你可以用 loadstring 或者 loadfile 重启加载。你也把可以把获取变化和重新加载用 code_loader 函数做一层包装：<br>```<br>local func = code_loader(name)<br>```<br>而在 code_loader 中我们一般会用 lru cache 对 `s` 做一层缓存。<br>这差不多就是完整的实现了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-04 09:17:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/cnNBSrNVDiagXWXkdWPnRxFh4D8oIbgnbiaA8WhIMkfI2vmemzTX3001YibnaPXHMRmCP5S6cGA7oHHLPHRBricr8g/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Netfeel</span>
  </div>
  <div class="_2_QraFYR_0">loadstring 在NYI列表是never，会不会对性能有很大影响？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 要看调用次数的，热代码才有 JIT 的必要性。loadstring 并不是一个频繁的操作，不会有性能问题的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-04 10:25:25</div>
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
  <div class="_2_QraFYR_0">lua-resty-upstream-healthcheck是不是说反了，应该只有主动健康检查没有被动健康检查</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 多谢指正，确实这里弄错了。lua-resty-upstream-healthcheck带的是 ngx.timer.at 这种主动健康检查的方式，而没有被动健康检查。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-04 08:01:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_838843</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，对于动态还是没有十分明白。举个例子: 比如现在有个连接池功能，由于现有连接数不够了，需要增加一些连接数，并且不能影响正在跑的线上业务。这些配置现在在项目的配置文件里配好了，这时候需要动态发布，这时候这个动态该怎么做呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-12 12:13:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTImmLJCKerl9CI4sTpPDNCUgswp04ybsJ4J6mpJmMlHh43Iibp1RPOLam5PpOv2ZDGcjvGrY94lNRw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Varphp</span>
  </div>
  <div class="_2_QraFYR_0">动态代理怎么做？ 不是跳转301 302就是动态反向代理 想要通过redis来定义路由  这种有可能实现吗😂</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-17 16:02:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/7c/71/ecfde294.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>钛合金猪头</span>
  </div>
  <div class="_2_QraFYR_0">老师，我想实现 类似 tengine 中的 ngx_http_upstream_check_module，这样一旦检测到 unhealthy，就自动把这个unhealthy的节点踢掉，有什么可以参考的吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-27 19:30:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4d/fd/0aa0e39f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>许童童</span>
  </div>
  <div class="_2_QraFYR_0">老师的文章很棒。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-07 14:40:16</div>
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
  <div class="_2_QraFYR_0">老师，loadstring或loadfile这种热更新相关的代码，我们该写在哪里，就是怎么触发？通过resty命令吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以写在 ngx.timer 里面，到数据库中定期的去检测版本号或者修改时间是否有变化。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-06 00:02:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/76/23/31e5e984.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>空知</span>
  </div>
  <div class="_2_QraFYR_0">二进制热升级 也没有停止服务 启动新的配置的程序，不算动态吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 算的，这是 Nginx&#47;OpenResty 自身的升级。但这个过程会关闭旧的 workers，启动新的 workers，可能耗时比较久，而且会丢失缓存。这种二级制热升级应该保持一个很低频的操作。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-04 23:43:10</div>
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
  <div class="_2_QraFYR_0">在使用resty-healthcheck这个库时，如何检获取所有真实服务器的健康状态呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你可以使用 checker:add_target 来设置多个上游节点的 ip 和端口，就像这个测试案例一样：https:&#47;&#47;github.com&#47;Kong&#47;lua-resty-healthcheck&#47;blob&#47;master&#47;t&#47;06-report_http_status.t#L69<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-04 07:51:04</div>
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
  <div class="_2_QraFYR_0">动态上游这块，我的做法是为一个服务设置2个upstream，然后根据路由条件选择不同的upstream，当机器ip有变化修改upstream中的ip即可，请问这样和直接使用balancer_by_lua有什么劣势或坑吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: balancer_by_lua 可以让用户选择负载均衡的算法是roundrobin 还是 chash，或者是其他的算法，比较自由。另外，在你的实现中，上游健康检查也需要自己来实现，有不少额外的工作量。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-04 07:47:41</div>
  </div>
</div>
</div>
</li>
</ul>