<audio title="34 _ 特别放送：OpenResty编码指南" src="https://static001.geekbang.org/resource/audio/c1/8b/c16e45390e340c66277c9d9ae9bb0d8b.mp3" controls="controls"></audio> 
<p>你好，我是温铭。</p><p>很多开发语言都有自己的编码规范，来告诉开发者这个领域内一些约定俗成的东西，让大家写的代码风格保持一致，并且避免一些常见的陷阱。这对于新手来说是非常友好的，可以让初学者快速准确地上手。比如 Python 的 PEP 80，就是其中的典范，几乎所有的 Python 开发者都阅读过这份 Python 作者执笔的编码规范。</p><p><strong>让开发者统一思想，按照规范来写代码，是一件非常重要的事情</strong>。OpenResty 还没有自己的编码规范，有些开发者在提交 PR 后，会在代码风格上被反复 review 和要求修改，消耗了大量本可避免的时间和精力。</p><p>其实，在 OpenResty 中，也有两个可以帮你自动化检测代码风格的工具：luacheck 和 lj-releng。前者是 Lua 和 OpenResty 世界通用的检测工具，后者则是 OpenResty 自己用 perl 写的代码检测工具。</p><p>对我自己来说，我会在 VS Code 编辑器中安装 luacheck 的插件，这样在我写代码的时候就有工具来自动提示；而在项目的 CI 中，则是会把这两个工具都运行一遍，比如：</p><pre><code>luacheck -q lua

./utils/lj-releng lua/*.lua lua/apisix/*.lua 
</code></pre><p>毕竟，多一个工具的检测总不是坏事。</p><p>但是，这两个工具更多的是检测全局变量、每行长度等这些最基础的代码风格，离 Python PEP 80 的详细程度还有遥远的距离，并且也没有文档给你参考。</p><!-- [[[read_end]]] --><p>所以今天，我就根据自己在OpenResty 相关开源项目中的经验，总结了一下 OpenResty 的编码风格文档，这个规范也和一些常见的 API 网关比如 Kong、APISIX 的代码风格是一致的。</p><h2>缩进</h2><p>在 OpenResty 中，我们使用 4 个空格作为缩进的标记，虽然 Lua 并没有这样的语法要求。下面是错误和正确的两段代码示例：</p><pre><code>--No
if a then
ngx.say(&quot;hello&quot;)
end
</code></pre><pre><code>--yes
if a then
    ngx.say(&quot;hello&quot;)
end
</code></pre><p>为了方便，你可以在使用的编辑器中，把 tab 改为 4 个空格，来简化操作。</p><h2>空格</h2><p>在操作符的两边，都需要用一个空格来做分隔。下面是错误和正确的两段代码示例：</p><pre><code>--No
local i=1
local s    =    &quot;apisix&quot;
</code></pre><pre><code>--Yes
local i = 1
local s = &quot;apisix&quot;
</code></pre><h2>空行</h2><p>不少开发者会把其他语言的开发习惯带到 OpenResty 中来，比如在行尾增加一个分号：</p><pre><code>--No
if a then
    ngx.say(&quot;hello&quot;);
end;
</code></pre><p>但事实上，增加分号会让 Lua 代码显得非常丑陋，也是没有必要的。同时，你也不要为了节省代码的行数，追求所谓的“简洁”，而把多行代码变为一行。这样做会让你在定位错误的时候，不知道到底是哪一段代码出了问题：</p><pre><code>--No
if a then ngx.say(&quot;hello&quot;) end
</code></pre><pre><code>--yes
if a then
    ngx.say(&quot;hello&quot;)
end
</code></pre><p>另外，函数之间需要用两个空行来做分隔：</p><pre><code>--No
local function foo()
end
 local function bar()
end
</code></pre><pre><code>--Yes
local function foo()
end


 local function bar()
end
</code></pre><p>如果有多个 if elseif 的分支，它们之间也需要一个空行来做分隔：</p><pre><code>--No
if a == 1 then
    foo()    
elseif a== 2 then
    bar()    
elseif a == 3 then
    run()    
else
    error()
end
</code></pre><pre><code>--Yes
if a == 1 then
    foo()

elseif a== 2 then
    bar()

elseif a == 3 then
    run()

else
    error()
end
</code></pre><h2>每行最大长度</h2><p>每行不能超过 80 个字符，如果超过的话，需要你换行并对齐。并且，在换行对齐的时候，我们要体现出上下两行的对应关系。就下面的示例而言，第二行函数的参数，要在第一行左括号的右边。</p><pre><code>--No 
return limit_conn_new(&quot;plugin-limit-conn&quot;, conf.conn, conf.burst, conf.default_conn_delay)
</code></pre><pre><code>--Yes
return limit_conn_new(&quot;plugin-limit-conn&quot;, conf.conn, conf.burst,
                    conf.default_conn_delay)
</code></pre><p>如果是字符串拼接问题的对齐，则需要把 <code>..</code> 放到下一行中：</p><pre><code>--No 
return limit_conn_new(&quot;plugin-limit-conn&quot; ..  &quot;plugin-limit-conn&quot; ..
                    &quot;plugin-limit-conn&quot;)
</code></pre><pre><code>--Yes
return limit_conn_new(&quot;plugin-limit-conn&quot; .. &quot;plugin-limit-conn&quot;
                    .. &quot;plugin-limit-conn&quot;)
</code></pre><h2>变量</h2><p>这一点我前面也多次强调过，我们应该永远使用局部变量，不要使用全局变量：</p><pre><code>--No
i = 1
s = &quot;apisix&quot;
</code></pre><pre><code>--Yes
local i = 1
local s = &quot;apisix&quot;
</code></pre><p>至于变量的命名，应该使用 <code>snake_case</code> 风格：</p><pre><code>--No
local IndexArr = 1
local str_Name = &quot;apisix&quot;
</code></pre><pre><code>--Yes
local index_arr = 1
local str_name = &quot;apisix&quot;
</code></pre><p>而对于常量，则是要使用全部大写的形式：</p><pre><code>--No
local max_int = 65535
local server_name = &quot;apisix&quot;
</code></pre><pre><code>--Yes
local MAX_INT = 65535
local SERVER_NAME = &quot;apisix&quot;
</code></pre><h2>数组</h2><p>在OpenResty中，我们使用<code>table.new</code> 来预先分配数组：</p><pre><code>--No
local t = {}
for i = 1, 100 do
   t[i] = i
 end
</code></pre><pre><code>--Yes 
local new_tab = require &quot;table.new&quot;
 local t = new_tab(100, 0)
 for i = 1, 100 do
   t[i] = i
 end
</code></pre><p>另外注意，一定不要在数组中使用 nil：</p><pre><code>--No
local t = {1, 2, nil, 3}
</code></pre><p>如果一定要使用空值，请用 ngx.null 来表示：</p><pre><code>--Yes
local t = {1, 2, ngx.null, 3}
</code></pre><h2>字符串</h2><p>千万不要在热代码路径上拼接字符串：</p><pre><code>--No
local s = &quot;&quot;
for i = 1, 100000 do
    s = s .. &quot;a&quot;
end
</code></pre><pre><code>--Yes
local t = {}
for i = 1, 100000 do
    t[i] = &quot;a&quot;
end
local s =  table.concat(t, &quot;&quot;)
</code></pre><h2>函数</h2><p>函数的命名也同样遵循 <code>snake_case</code>：</p><pre><code>--No
local function testNginx()
end
</code></pre><pre><code>--Yes
local function test_nginx()
end
</code></pre><p>并且，函数应该尽可能早地返回：</p><pre><code>--No
local function check(age, name)
    local ret = true
    if age &lt; 20 then
        ret = false
    end

    if name == &quot;a&quot; then
        ret = false
    end
    -- do something else 
    return ret 
</code></pre><pre><code>--Yes
local function check(age, name)
    if age &lt; 20 then
        return false
    end

    if name == &quot;a&quot; then
        return false
    end
    -- do something else 
    return true 
</code></pre><h2>模块</h2><p>所有 require 的库都要 local 化：</p><pre><code>--No
local function foo()
    local ok, err = ngx.timer.at(delay, handler)
end
</code></pre><pre><code>--Yes
local timer_at = ngx.timer.at

local function foo()
    local ok, err = timer_at(delay, handler)
end
</code></pre><p>为了风格的统一，require 和 ngx 也需要 local 化：</p><pre><code>--No
local core = require(&quot;apisix.core&quot;)
local timer_at = ngx.timer.at

local function foo()
    local ok, err = timer_at(delay, handler)
end
</code></pre><pre><code>--Yes
local ngx = ngx
local require = require
local core = require(&quot;apisix.core&quot;)
local timer_at = ngx.timer.at

local function foo()
    local ok, err = timer_at(delay, handler)
end
</code></pre><h2>错误处理</h2><p>对于有错误信息返回的函数，我们必须对错误信息进行判断和处理：</p><pre><code>--No
local sock = ngx.socket.tcp()
 local ok = sock:connect(&quot;www.google.com&quot;, 80)
 ngx.say(&quot;successfully connected to google!&quot;)
</code></pre><pre><code>--Yes
local sock = ngx.socket.tcp()
 local ok, err = sock:connect(&quot;www.google.com&quot;, 80)
 if not ok then
     ngx.say(&quot;failed to connect to google: &quot;, err)
     return
 end
 ngx.say(&quot;successfully connected to google!&quot;)
</code></pre><p>而如果是自己编写的函数，错误信息要作为第二个参数，用字符串的格式返回：</p><pre><code>--No
local function foo()
    local ok, err = func()
    if not ok then
        return false
    end
    return true
end
</code></pre><pre><code>--No
local function foo()
    local ok, err = func()
    if not ok then
        return false, {msg = err}
    end
    return true
end
</code></pre><pre><code>--Yes
local function foo()
    local ok, err = func()
    if not ok then
        return false, &quot;failed to call func(): &quot; .. err
    end
    return true
end
</code></pre><h2>写在最后</h2><p>这个编程规范算是一个最初版本，我会公开到 <a href="https://github.com/apache/incubator-apisix/blob/v1.3/CODE_STYLE.md">GitHub</a> 中来持续更新和维护。如果文中没有包含到你想知道的规范，非常欢迎你留言提问，我来给你解答。也欢迎你把这篇规范分享出去，让更多的OpenResty使用者参与进来。</p>
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
  <div class="_2_QraFYR_0">local ngx = ngx 这种方式，luacheck会报警告：accessing undefined variable ngx<br>local ngx = require &quot;ngx&quot;  这样就不会报警告了，而且也能用，老师，这样可以吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 就是这样的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-13 17:59:53</div>
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
  <div class="_2_QraFYR_0">老师你好 <br>++操作符之间是否需要空格？<br>i++ 还是<br>i ++</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Lua 中并不支持 ++ 这个操作符，我们一般都是用 i = i + 1来替代</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-12 16:06:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2e/e5/a0/8b3e4acc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>鱼鱼</span>
  </div>
  <div class="_2_QraFYR_0">luacheck可以-std ngx_lua，这样不用写成require ngx。另外ci可以额外传参数 -only 0 1 这样就只会检查lua的报错和全局变量。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-04 22:36:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_Wx.71</span>
  </div>
  <div class="_2_QraFYR_0">lj-releng 检查时候会对ngx,setmetatable,string 等等标准库函数报错。。。这是什么原因引起的呢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-02 15:03:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/79/7e/c38ac02f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>北冥Master</span>
  </div>
  <div class="_2_QraFYR_0">Require库local化的意义？避免修改库导致污染其他加载该库的代码？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-20 23:24:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/3c/84/608f679b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>连边</span>
  </div>
  <div class="_2_QraFYR_0">一门语言，规范还是很重要的。老师有心了。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-14 09:00:43</div>
  </div>
</div>
</div>
</li>
</ul>