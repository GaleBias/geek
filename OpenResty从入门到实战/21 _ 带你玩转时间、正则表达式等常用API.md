<audio title="21 _ 带你玩转时间、正则表达式等常用API" src="https://static001.geekbang.org/resource/audio/3f/7b/3fbd10a52dad9e660748b05f114fde7b.mp3" controls="controls"></audio> 
<p>你好，我是温铭。在前面几节课中，你已经熟悉了不少OpenResty 中重要的 Lua API 了，今天我们再来了解下其他一些通用的 API，主要和正则表达式、时间、进程等相关。</p><h2>正则</h2><p>先来看下最常用，也是最重要的正则。在 OpenResty 中，我们应该一直使用 <code>ngx.re.*</code> 提供的一系列 API，来处理和正则表达式相关的逻辑，而不是用 Lua 自带的模式匹配。这不仅是出于性能方面的考虑，还因为Lua 自带的正则是自成体系的，并非 PCRE 规范，这对于绝大部分开发者来说都是徒增烦恼。</p><p>在前面的课程中，你已经多多少少接触过一些 <code>ngx.re.*</code> 的 API了，文档也写得非常详细，我就不再一一列举了。这里，我再单独强调两个内容。</p><h3>ngx.re.split</h3><p>第一个是<code>ngx.re.split</code>。字符串切割是很常见的功能，OpenResty 也提供了对应的 API，但在社区的 QQ 交流群中，很多开发者都找不到这样的函数，只能选择自己手写。</p><p>为什么呢？其实， <code>ngx.re.split</code> 这个 API 并不在 lua-nginx-module 中，而是在 lua-resty-core 里面；并且它也不在 lua-resty-core 首页的文档中，而是在 <code>lua-resty-core/lib/ngx/re.md</code> 这个第三级目录的文档中出现的。多种原因，导致很多开发者完全不知道这个 API 的存在。</p><!-- [[[read_end]]] --><p>类似这种“藏在深闺无人识“的 API，还有我们前面提到过的 <code>ngx_resp.add_header</code>、<code>enable_privileged_agent</code> 等等。那么怎么来最快地解决这种问题呢？除了阅读 lua-resty-core 首页文档外，你还需要把 <code>lua-resty-core/lib/ngx/</code> 这个目录下的 <code>.md</code> 格式的文档也通读一遍才行。</p><p>我们前面夸了很多 OpenResty 文档做得好的地方，不过，这一点上，也就是在一个页面能够查询到完整的 API 列表，确实还有很大的改进空间。</p><h3>lua_regex_match_limit</h3><p>第二个，我想介绍一下<code>lua_regex_match_limit</code>。我们之前并没有花专门的篇幅，来讲 OpenResty 提供的 Nginx 指令，因为大部分情况下我们使用默认值就足够了，它们也没有在运行时去修改的必要性。不过，我们今天要讲的这个，和正则表达式相关的<code>lua_regex_match_limit</code> 指令，却是一个例外。</p><p>我们知道，如果我使用的正则引擎是基于回溯的 NFA 来实现的，那么就有可能出现灾难性回溯（Catastrophic Backtracking），即正则在匹配的时候回溯过多，造成 CPU 100%，正常服务被阻塞。</p><p>一旦发生灾难性回溯，我们就需要用 gdb 分析 dump，或者 systemtap 分析线上环境才能定位，而且事先也不容易发现，因为只有特别的请求才会触发。这显然就给攻击者带来了可趁之机，ReDoS（RegEx Denial of Service）就是指的这类攻击。</p><p>如果你对如何自动化发现和彻底解决这个问题感兴趣，可以参考我之前在公众号写的一篇文章：<a href="https://mp.weixin.qq.com/s/K9d60kjDdFn6ZwIdsLjqOw">如何彻底避免正则表达式的灾难性回溯</a>？</p><p>今天在这里，我主要给你介绍下，如何在 OpenResty 中简单有效地规避，也就是使用下面这行代码：</p><pre><code>lua_regex_match_limit 100000;
</code></pre><p><code>lua_regex_match_limit</code> ，就是用来限制 PCRE 正则引擎的回溯次数的。这样，即使出现了灾难性回溯，后果也会被限制在一个范围内，不会导致你的 CPU 满载。</p><p>这里我简单说一下，这个指令的默认值是 0，也就是不做限制。如果你没有替换 OpenResty 自带的正则引擎，并且还涉及到了比较多的复杂的正则表达式，你可以考虑重新设置这个 Nginx 指令的值。</p><h2>时间 API</h2><p>接下来我们说说时间 API。OpenResty 提供了 10 个左右和时间相关的 API，从这个数量你也可见它的重要性。一般来说，最常用的时间 API就是 <code>ngx.now</code>，它可以打印出当前的时间戳，比如下面这行代码：</p><pre><code>resty -e 'ngx.say(ngx.now())'
</code></pre><p>从打印的结果可以看出，<code>ngx.now</code> 包括了小数部分，所以更加精准。而与之相关的 <code>ngx.time</code> 则只返回了整数部分的值。至于其他的 <code>ngx.localtime</code>、<code>ngx.utctime</code>、<code>ngx.cookie_time</code> 和 <code>ngx.http_time</code> ，主要是返回和处理时间的不同格式。具体用到的话，你可以查阅文档，本身并不难理解，我就没有必要专门来讲了。</p><p>不过，值得一提的是，<strong>这些返回当前时间的 API，如果没有非阻塞网络 IO 操作来触发，便会一直返回缓存的值，而不是像我们想的那样，能够返回当前的实时时间</strong>。可以看看下面这个示例代码：</p><pre><code>$ resty -e 'ngx.say(ngx.now())
os.execute(&quot;sleep 1&quot;)
ngx.say(ngx.now())'
</code></pre><p>在两次调用 <code>ngx.now</code> 之间，我们使用 Lua 的阻塞函数 sleep 了 1 秒钟，但从打印的结果来看，这两次返回的时间戳却是一模一样的。</p><p>那么，如果换成是非阻塞的 sleep 函数呢？比如下面这段新的代码：</p><pre><code>$ resty -e 'ngx.say(ngx.now())
ngx.sleep(1)
ngx.say(ngx.now())'
</code></pre><p>显然，它就会打印出不同的时间戳了。这里顺带引出了 <code>ngx.sleep</code> ，这个非阻塞的 sleep 函数。这个函数除了可以休眠指定的时间外，还有另外一个特别的用处。</p><p>举个例子，比如你有一段正在做密集运算的代码，需要花费比较多的时间，那么在这段时间内，这段代码对应的请求就会一直占用着 worker 和 CPU 资源，导致其他请求需要排队，无法得到及时的响应。这时，我们就可以在其中穿插 <code>ngx.sleep(0)</code>，使这段代码让出控制权，让其他请求也可以得到处理。</p><h2>worker 和进程 API</h2><p>再来看worker 和进程相关的API。OpenResty 提供了 <code>ngx.worker.*</code> 和 <code>ngx.process.*</code> 这些 API， 来获取 worker 和进程相关的信息。其中，前者和 Nginx worker 进程有关，后者则是泛指所有的 Nginx 进程，不仅有 worker 进程，还有 master 进程和特权进程等等。</p><p>事实上，<code>ngx.worker.*</code> 由 lua-nginx-module 提供，而<code>ngx.process.*</code> 则是由 lua-resty-core 提供。还记得上节课我们留的作业题吗，如何保证在多 worker 的情况下，只启动一个 timer？其实，这就需要用到 <code>ngx.worker.id</code> 这个 API 了。你可以在启动 timer 之前，先做一个简单的判断：</p><pre><code>if ngx.worker.id == 0 then
    start_timer()
end
</code></pre><p>这样，我们就能实现只启动一个 timer的目的了。这里注意，worker id 是从 0 开始返回的，这和 Lua 中数组下标从 1 开始并不相同，千万不要混淆了。</p><p>至于其他 worker 和 process 相关的 API，并没有什么特别需要注意的地方，就交给你自己去学习和练习了。</p><h2>真值和空值</h2><p>最后我们来看看，真值和空值的问题。在 OpenResty 中，真值与空值的判断，一直是个让人头痛、也比较混乱的点。</p><p>我们先看来下 Lua 中真值的定义：<strong>除了 nil 和 false 之外，都是真值。</strong></p><p>所以，真值也就包括了：0、空字符串、空表等等。</p><p>再来看下 Lua 中的空值（nil），它是未定义的意思，比如你申明了一个变量，但还没有初始化，它的值就是 nil：</p><pre><code>$ resty -e 'local a
ngx.say(type(a))'
</code></pre><p>而 nil 也是 Lua 中的一种数据类型。</p><p>明白了这两点后，我们现在就来具体看看，基于这两个定义，衍生出来的其他坑。</p><h3>ngx.null</h3><p>第一个坑是<code>ngx.null</code>。因为 Lua 的 nil 无法作为 table 的 value，所以 OpenResty 引入了 <code>ngx.null</code>，作为 table 中的空值：</p><pre><code>$ resty -e  'print(ngx.null)'
null
</code></pre><pre><code>$ resty -e 'print(type(ngx.null))'
userdata
</code></pre><p>从上面两段代码你可以看出，<code>ngx.null</code> 被打印出来是 null，而它的类型是 userdata。那么，可以把它当作假值吗？当然不行，事实上，<code>ngx.null</code> 的布尔值为真：</p><pre><code>$ resty -e 'if ngx.null then
ngx.say(&quot;true&quot;)
end'
</code></pre><p>所以，要谨记，<strong>只有 nil 和 false 是假值</strong>。如果你遗漏了这一点，就很容易踩坑，比如你在使用 lua-resty-redis 的时候，做了下面这个判断：</p><pre><code>local res, err = red:get(&quot;dog&quot;)
if not res then
    res = res + &quot;test&quot;
end 
</code></pre><p>如果返回值 res 是 nil，就说明函数调用失败了；如果 res 是 ngx.null，就说明 redis 中不存在 <code>dog</code> 这个key。那么，在 <code>dog</code> 这个 key 不存在的情况下，这段代码就 500 崩溃了。</p><h3>cdata:NULL</h3><p>第二个坑是<code>cdata:NULL</code>。当你通过 LuaJIT FFI 接口去调用 C 函数，而这个函数返回一个 NULL 指针，那么你就会遇到另外一种空值，即<code>cdata:NULL</code> 。</p><pre><code>$ resty -e 'local ffi = require &quot;ffi&quot;
local cdata_null = ffi.new(&quot;void*&quot;, nil)
if cdata_null then
    ngx.say(&quot;true&quot;)
end'
</code></pre><p>和 <code>ngx.null</code> 一样，<code>cdata:NULL</code> 也是真值。但更让人匪夷所思的是，下面这段代码，会打印出 true，也就是说<code>cdata:NULL</code> 是和 <code>nil</code> 相等的：</p><pre><code>$ resty -e 'local ffi = require &quot;ffi&quot;
local cdata_null = ffi.new(&quot;void*&quot;, nil)
ngx.say(cdata_null == nil)'
</code></pre><p>那么我们应该如何处理 <code>ngx.null</code> 和 <code>cdata:NULL</code> 呢？显然，让应用层来关心这些闹心事儿是不现实的，最好是做一个二层封装，不要让调用者知道这些细节即可。</p><h3>cjson.null</h3><p>最后，我们再来看下 cjson 中出现的空值。cjson 库会把 json 中的 NULL，解码为 Lua 的 <code>lightuserdata</code>，并用 <code>cjson.null</code> 来表示：</p><pre><code>$ resty -e 'local cjson = require &quot;cjson&quot;
local data = cjson.encode(nil)
local decode_null = cjson.decode(data)
ngx.say(decode_null == cjson.null)'
</code></pre><p>Lua 中的 nil，被 json encode 和 decode 一圈儿之后，就变成了 <code>cjson.null</code>。你可以想得到，它引入的原因和 <code>ngx.null</code> 是一样的，因为 nil 无法在 table 中作为 value。</p><p>到现在为止，看了这么多 OpenResty 中的空值，不知道你蒙圈儿了没？不要慌张，这部分内容多看几遍，自己梳理一下，就不至于晕头转向分不清了。当然，你以后在写类似 <code>if not foo then</code> 的时候，就要多想想，这个条件到底能不能成立了。</p><h2>写在最后</h2><p>学完今天这节课后，OpenResty 中常用的 Lua API 我们就都介绍过了，不知道你是否都清楚了呢？</p><p>最后，留一个思考题给你：在 <code>ngx.now</code> 的示例中，为什么在没有 yield 操作的时候，它的值不会修改呢？欢迎留言分享你的看法，也欢迎你把这篇文章分享出去，我们一起交流，一起进步。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKsz8j0bAayjSne9iakvjzUmvUdxWEbsM9iasQ74spGFayIgbSE232sH2LOWmaKtx1WqAFDiaYgVPwIQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>2xshu</span>
  </div>
  <div class="_2_QraFYR_0">文末的问题，难道是ngx.now()取时间发生在resusme函数恢复堆栈阶段？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Nginx 是以性能优先作为设计理念的，它会把时间缓存下来。从 ngx.now 的源码中我们可以得到印证：<br>static int<br>ngx_http_lua_ngx_now(lua_State *L)<br>{<br>    ngx_time_t              *tp;<br><br>    tp = ngx_timeofday();<br><br>    lua_pushnumber(L, (lua_Number) (tp-&gt;sec + tp-&gt;msec &#47; 1000.0L));<br><br>    return 1;<br>}<br>是调用了 Nginx 中的 ngx_timeofday 函数获取的时间。<br>而这个函数其实是一个宏定义：<br>#define ngx_timeofday()      (ngx_time_t *) ngx_cached_time<br><br>而 ngx_cached_time 的值只在函数 ngx_time_update 中会更新。<br>那问题就简化为： ngx_time_update什么时候会被调用。如果你在 Nginx 的源码中去跟踪它的话，就会发现ngx_time_update的调用比较多，在事件循环中都有出现。<br><br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-12 14:34:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/31/ba/35b3b16c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Marsman</span>
  </div>
  <div class="_2_QraFYR_0">---- 原文start<br>local res, err = red:get(&quot;dog&quot;)<br>if not res then<br>    res = res + &quot;test&quot;<br>end <br>如果 res 是 ngx.null，就说明 redis 中不存在 dog 这个 key。那么，在 dog 这个 key 不存在的情况下，这段代码就 500 崩溃了。<br>----原文end<br><br>这段代码并没有500崩溃。 res是ngx.null ,   not res 相当于 not true，即为false， 所以并没有进入if 代码块， 我想是因为“除了 nil 和 false 之外，都是真值”， 所以ngx.null也是真值</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-16 22:46:43</div>
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
  <div class="_2_QraFYR_0">老师，问一个困惑我的问题：<br>local res, err = red:hmset(&quot;animals&quot;, t)<br>if not res then<br>    ngx.say(&quot;failed to set animals: &quot;, err)<br>    return<br>end<br>例如上面这种代码，其中的return有什么用呢，不加这个return不行吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: return 是很明确的跳出了这个函数，不再执行后面的语句。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-14 16:49:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2d/66/c2/881e527e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>土豆</span>
  </div>
  <div class="_2_QraFYR_0">老师，我这里初始化中，获得到的worker id 为何总是0，获得不到其它的，启动后会同时打印2个====worderID===0<br>worker_processes配置为2<br>以下为关键代码：<br>    init_worker_by_lua_block{<br>        require &quot;init&quot;<br>    }<br><br>init.lua脚本代码<br>local function refreshRedisData(premature)<br>    log(ERR,&quot;====worderID===&quot;,ngx.worker.id())<br>end</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-15 09:39:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/67/e4/42ea7a9a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>姚坤</span>
  </div>
  <div class="_2_QraFYR_0">本地Windows机器运行测试，诡异的发现ngx.worker.id是一个函数，应该写成<br>if ngx.worker.id() == 0 then<br>    start_timer()<br>end<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: windows 下这个函数是有问题的，最好在 Linux、mac 下来使用 OpenResty</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-21 13:13:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/92/87/15931e41.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Joshua</span>
  </div>
  <div class="_2_QraFYR_0">这时，我们就可以在其中穿插 ngx.sleep(0)，使这段代码让出控制权，让其他请求也可以得到处理。<br><br>这里 sleep(0) 为什么 ngx.sleep(0) 会让出控制权呀</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 因为 ngx.sleep 也是一个 yield 操作，会触发 nginx 的事件循环。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-30 21:19:56</div>
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
  <div class="_2_QraFYR_0">看了留言以及回复，也就是说只有在调用了ngx_timer_update的时候，ngx.timer的值才会更新，而调用前者多是在事件循环中，而调用yield 函数通常是添加了一个事件。这样解释了需要yield操作之后，ngx.timer才会更新，是吧。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，没错</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-14 18:20:45</div>
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
  <div class="_2_QraFYR_0">我知道return的用法了，总结了一下：<br><br>return的用法：<br>用在函数中时，return 主要是用于从函数中返回结果，不会终止程序继续执行；<br>用在条件语句中时，return用于终止当前程序的运行，后面的所有代码都不会执行了；<br>用于循环语句时，return用于终止当前程序的运行，后面的所有代码都不会执行了，而break是终止循环继续运行，注意他们的区别；<br>注意return不能直接用于代码文件级别。<br><br>do return end一般用于调试代码的场景使用：<br>用于函数中时，可放置在函数中代码的中间，这样函数剩余的代码部分不会被执行，不会中断程序执行；<br>用于条件、循环语句中时，可放置在代码块的中间，会在此处中断程序执行；<br>直接用于代码文件级别，会在此处中断程序执行。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-14 17:27:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/25/f7/4cc60573.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>zhang</span>
  </div>
  <div class="_2_QraFYR_0">它只获取nginx内部的一个状态，不需要让出cpu。也就不需要lua协程和ngx core之间的切换了。<br><br><br><br>只有期望让出cpu，让ngx corre帮他完成部分操作的api，才会出现上下文的切换，也就是yield和resume</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-12 19:11:48</div>
  </div>
</div>
</div>
</li>
</ul>