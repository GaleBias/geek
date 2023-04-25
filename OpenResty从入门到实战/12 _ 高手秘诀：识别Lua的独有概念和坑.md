<audio title="12 _ 高手秘诀：识别Lua的独有概念和坑" src="https://static001.geekbang.org/resource/audio/f7/14/f738771f4d119db9326fc0607719e414.mp3" controls="controls"></audio> 
<p>你好，我是温铭。</p><p>上一节中，我们一起了解了 LuaJIT 中 table 相关的库函数。除了这些常用的函数外，今天我再为你介绍一些Lua 独有的或不太常用的概念，以及 OpenResty 中常见的 Lua 的坑。</p><h2>弱表</h2><p>首先是 <code>弱表</code>（weak table），它是 Lua 中很独特的一个概念，和垃圾回收相关。和其他高级语言一样，Lua 是自动垃圾回收的，你不用关心具体的实现，也不用显式 GC。没有被引用到的空间，会被垃圾收集器自动完成回收。</p><p>但简单的引用计数还不太够用，有时候我们需要一种更灵活的机制。举个例子，我们把一个 Lua 的对象 <code>Foo</code>（table 或者函数）插入到 table <code>tb</code> 中，这就会产生对这个对象 <code>Foo</code> 的引用。即使没有其他地方引用 <code>Foo</code>，<code>tb</code> 对它的引用也还一直存在，那么 GC 就没有办法回收 <code>Foo</code> 所占用的内存。这时候，我们就只有两种选择：</p><ul>
<li>一是手工释放 <code>Foo</code>；</li>
<li>二是让它常驻内存。</li>
</ul><p>比如下面这段代码：</p><pre><code>$ resty -e 'local tb = {}
tb[1] = {red}
tb[2] = function() print(&quot;func&quot;) end
print(#tb) -- 2

collectgarbage()
print(#tb) -- 2

table.remove(tb, 1)
print(#tb) -- 1
</code></pre><p>不过，你肯定不希望，内存一直被用不到的对象占用着吧，特别是 LuaJIT 中还有 2G 内存的上限。而手工释放的时机并不好把握，也会增加代码的复杂度。</p><p>那么这时候，就轮到弱表来大显身手了。看它的名字，弱表，首先它是一个表，然后这个表里面的所有元素都是弱引用。概念总是抽象的，让我们先来看一段稍加修改后的代码：</p><!-- [[[read_end]]] --><pre><code>$ resty -e 'local tb = {}
tb[1] = {red}
tb[2] = function() print(&quot;func&quot;) end
setmetatable(tb, {__mode = &quot;v&quot;})
print(#tb)  -- 2

collectgarbage()
print(#tb) -- 0
'
</code></pre><p>可以看到，没有被使用的对象都被 GC 了。这其中，最重要的就是下面这一行代码：</p><pre><code>setmetatable(tb, {__mode = &quot;v&quot;})
</code></pre><p>是不是似曾相识？这不就是元表的操作吗！没错，当一个 table 的元表中存在 <code>__mode</code> 字段时，这个 table 就是弱表（weak table）了。</p><ul>
<li>如果 <code>__mode</code> 的值是 <code>k</code>，那就意味着这个 table 的 <code>键</code> 是弱引用。</li>
<li>如果 <code>__mode</code> 的值是 <code>v</code>，那就意味着这个 table 的 <code>值</code> 是弱引用。</li>
<li>当然，你也可以设置为 <code>kv</code>，表明这个表的键和值都是弱引用。</li>
</ul><p>这三者中的任意一种弱表，只要它的 <code>键</code> 或者 <code>值</code> 被回收了，那么对应的<strong>整个</strong><code>键值</code> 对象都会被回收。</p><p>在上面的代码示例中，<code>__mode</code> 的值 <code>v</code>，而<code>tb</code> 是一个数组，数组的 <code>value</code> 则是 table 和函数对象，所以可以被自动回收。不过，如果你把<code>__mode</code> 的值改为 <code>k</code>，就不会 GC 了，比如看下面这段代码：</p><pre><code>$ resty -e 'local tb = {}
tb[1] = {red}
tb[2] = function() print(&quot;func&quot;) end
setmetatable(tb, {__mode = &quot;k&quot;})
print(#tb)  -- 2

collectgarbage()
print(#tb) -- 2
'
</code></pre><p>请注意，这里我们只演示了 <code>value</code> 为弱引用的弱表，也就是数组类型的弱表。自然，你同样可以把对象作为 <code>key</code>，来构建哈希表类型的弱表，比如下面这样写：</p><pre><code>$ resty -e 'local tb = {}
tb[{color = red}] = &quot;red&quot;
local fc = function() print(&quot;func&quot;) end
tb[fc] = &quot;func&quot;
fc = nil

setmetatable(tb, {__mode = &quot;k&quot;})
for k,v in pairs(tb) do
     print(v)
end

collectgarbage()
print(&quot;----------&quot;)
for k,v in pairs(tb) do
     print(v)
end
'
</code></pre><p>在手动调用 <code>collectgarbage()</code> 进行强制 GC 后，<code>tb</code> 整个 table 里面的元素，就已经全部被回收了。当然，在实际的代码中，我们大可不必手动调用 <code>collectgarbage()</code>，它会在后台自动运行，无须我们担心。</p><p>不过，既然提到了 <code>collectgarbage()</code> 这个函数，我就再多说几句。这个函数其实可以传入多个不同的选项，且默认是 <code>collect</code>，即完整的 GC。另一个比较有用的是 <code>count</code>，它可以返回 Lua 占用的内存空间大小。这个统计数据很有用，可以让你看出是否存在内存泄漏，也可以提醒我们不要接近 2G 的上限值。</p><p>弱表相关的代码，在实际应用中会写得比较复杂，不太容易理解，相对应的，也会隐藏更多的 bug。具体有哪些呢？不必着急，后面内容，我会专门介绍一个开源项目中，使用弱表带来的内存泄漏问题。</p><h2>闭包和 upvalue</h2><p>再来看闭包和 upvalue。前面我强调过，在 Lua 中，所有的值都是一等公民，包含函数也是。这就意味着函数可以保存在变量中，当作参数传递，以及作为另一个函数的返回值。比如在上面弱表中出现的这段示例代码：</p><pre><code>tb[2] = function() print(&quot;func&quot;) end
</code></pre><p>其实就是把一个匿名函数，作为 table 的值给存储了起来。</p><p>在 Lua 中，下面这段代码中动两个函数的定义是完全等价的。不过注意，后者是把函数赋值给一个变量，这也是我们经常会用到的一种方式：</p><pre><code>local function foo() print(&quot;foo&quot;) end
local foo = fuction() print(&quot;foo&quot;) end
</code></pre><p>另外，Lua 支持把一个函数写在另外一个函数里面，即嵌套函数，比如下面的示例代码：</p><pre><code>$ resty -e '
local function foo()
     local i = 1
     local function bar()
         i = i + 1
         print(i)
     end
     return bar
end

local fn = foo()
print(fn()) -- 2
'
</code></pre><p>你可以看到， <code>bar</code> 这个函数可以读取函数 <code>foo</code> 里面的局部变量 <code>i</code>，并修改它的值，即使这个变量并不在 <code>bar</code> 里面定义。这个特性叫做词法作用域（lexical scoping）。</p><p>事实上，Lua 的这些特性正是闭包的基础。所谓<code>闭包</code> ，简单地理解，它其实是一个函数，不过它访问了另外一个函数词法作用域中的变量。</p><p>如果按照闭包的定义来看，Lua 的所有函数实际上都是闭包，即使你没有嵌套。这是因为 Lua 编译器会把 Lua 脚本外面，再包装一层主函数。比如下面这几行简单的代码段：</p><pre><code>local foo, bar
local function fn()
     foo = 1
     bar = 2
end
</code></pre><p>在编译后，就会变为下面的样子：</p><pre><code>function main(...)
     local foo, bar
     local function fn()
         foo = 1
         bar = 2
     end
end
</code></pre><p>而函数 <code>fn</code> 捕获了主函数的两个局部变量，因此也是闭包。</p><p>当然，我们知道，很多语言中都有闭包的概念，它并非 Lua 独有，你也可以对比着来加深理解。只有理解了闭包，你才能明白我们接下来要讲的 upvalue。</p><p>upvalue 就是 Lua 中独有的概念了。从字面意思来看，可以翻译成 <code>上面的值</code>。实际上，upvalue 就是闭包中捕获的自己词法作用域外的那个变量。还是继续看上面那段代码：</p><pre><code>local foo, bar
local function fn()
     foo = 1
     bar = 2
end
</code></pre><p>你可以看到，函数 <code>fn</code> 捕获了两个不在自己词法作用域的局部变量 <code>foo</code> 和 <code>bar</code>，而这两个变量，实际上就是函数 <code>fn</code> 的 upvalue。</p><h2>常见的坑</h2><p>介绍了 Lua 中的几个概念后，我再来说说，在 OpenResty 开发中遇到的那些和 Lua 相关的坑。</p><p>在前面内容中，我们提到了一些 Lua 和其他开发语言不同的点，比如下标从 1 开始、默认全局变量等等。在 OpenResty 实际的代码开发中，我们还会遇到更多和 Lua、 LuaJIT 相关的问题点， 下面我会讲其中一些比较常见的。</p><p>这里要先提醒一下，即使你知道了所有的 <code>坑</code>，但不可避免的，估计还是要自己踩过之后才能印象深刻。当然，不同的是，你能够更块地从坑里面爬出来，并找到症结所在。</p><h3>下标从 0 开始还是从 1 开始</h3><p>第一个坑，Lua 的下标是从 1 开始的，这点我们之前反复提及过。但我不得不说，这并非事实的全部。</p><p>因为在 LuaJIT 中，使用 <code>ffi.new</code> 创建的数组，下标又是从 0 开始的:</p><pre><code>local buf = ffi_new(&quot;char[?]&quot;, 128)
</code></pre><p>所以，如果你要访问上面这段代码中 <code>buf</code> 这个 cdata，请记得下标从 0 开始，而不是 1。在使用 FFI 和 C 交互的时候，一定要特别注意这个地方。</p><h3>正则模式匹配</h3><p>第二个坑，正则模式匹配问题。OpenResty 中并行着两套字符串匹配方法：Lua 自带的 <code>sting</code> 库，以及 OpenResty 提供的 <code>ngx.re.*</code> API。</p><p>其中， Lua 正则模式匹配是自己独有的格式，和 PCRE 的写法不同。下面是一个简单的示例：</p><pre><code>resty -e 'print(string.match(&quot;foo 123 bar&quot;, &quot;%d%d%d&quot;))'  — 123
</code></pre><p>这段代码从字符串中提取了数字部分，你会发现，它和我们的熟悉的正则表达式完全不同。Lua 自带的正则匹配库，不仅代码维护成本高，而且性能低——不能被 JIT，而且被编译过一次的模式也不会被缓存。</p><p>所以，在你使用 Lua 内置的 string 库去做 find、match 等操作时，如果有类似正则这样的需求，不用犹豫，请直接使用 OpenResty 提供的 <code>ngx.re</code> 来替代。只有在查找固定字符串的时候，我们才考虑使用 plain 模式来调用 string 库。</p><p><strong>这里我有一个建议：在 OpenResty 中，我们总是优先使用 OpenResty 的 API，然后是 LuaJIT 的 API，使用 Lua 库则需要慎之又慎</strong>。</p><h3>json 编码时无法区分 array 和 dict</h3><p>第三个坑，json 编码时无法区分 array 和 dict。由于 Lua 中只有 table 这一个数据结构，所以在 json 对空 table 编码的时候，自然就无法确定编码为数组还是字典：</p><pre><code>resty -e 'local cjson = require &quot;cjson&quot;
local t = {}
print(cjson.encode(t))
'
</code></pre><p>比如上面这段代码，它的输出是 <code>{}</code>，由此可见， OpenResty 的 cjson 库，默认把空 table 当做字典来编码。当然，我们可以通过 <code>encode_empty_table_as_object</code> 这个函数，来修改这个全局的默认值：</p><pre><code>resty -e 'local cjson = require &quot;cjson&quot;
cjson.encode_empty_table_as_object(false)
local t = {}
print(cjson.encode(t))
'
</code></pre><p>这次，空 table 就被编码为了数组：<code>[]</code>。</p><p>不过，全局这种设置的影响面比较大，那能不能指定某个 table 的编码规则呢？答案自然是可以的，我们有两种方法可以做到。</p><p>第一种方法，把 <code>cjson.empty_array</code> 这个 userdata 赋值给指定 table。这样，在 json 编码的时候，它就会被当做空数组来处理：</p><pre><code>$ resty -e 'local cjson = require &quot;cjson&quot;
local t = cjson.empty_array
print(cjson.encode(t))
'
</code></pre><p>不过，有时候我们并不确定，这个指定的 table 是否一直为空。我们希望当它为空的时候编码为数组，那么就要用到 <code>cjson.empty_array_mt</code> 这个函数，也就是我们的第二个方法。</p><p>它会标记好指定的 table，当 table 为空时编码为数组。从<code>cjson.empty_array_mt</code> 这个命名你也可以看出，它是通过 metatable 的方式进行设置的，比如下面这段代码操作：</p><pre><code>$ resty -e 'local cjson = require &quot;cjson&quot;
local t = {}
setmetatable(t, cjson.empty_array_mt)
print(cjson.encode(t))
t = {123}
print(cjson.encode(t))
'
</code></pre><p>你可以在本地执行一下这段代码，看看输出和你预期的是否一致。</p><h3>变量的个数限制</h3><p>再来看第四个坑，变量的个数限制问题。 Lua 中，一个函数的局部变量的个数，和 upvalue 的个数都是有上限的，你可以从 Lua 的源码中得到印证：</p><pre><code>
/*
@@ LUAI_MAXVARS is the maximum number of local variables per function
@* (must be smaller than 250).
*/
#define LUAI_MAXVARS            200


/*
@@ LUAI_MAXUPVALUES is the maximum number of upvalues per function
@* (must be smaller than 250).
*/
#define LUAI_MAXUPVALUES        60
</code></pre><p>这两个阈值，分别被硬编码为 200 和 60。虽说你可以手动修改源码来调整这两个值，不过最大也只能设置为 250。</p><p>一般情况下，我们不会超过这个阈值，但写 OpenResty 代码的时候，你还是要留意这个事情，不要过多地使用局部变量和 upvalue，而是要尽可能地使用 <code>do .. end</code> 做一层封装，来减少局部变量和 upvalue 的个数。</p><p>比如我们来看下面这段伪码：</p><pre><code>local re_find = ngx.re.find
  function foo() ... end
function bar() ... end
function fn() ... end
</code></pre><p>如果只有函数 <code>foo</code> 使用到了 <code>re_find</code>， 那么我们可以这样改造下：</p><pre><code>do
     local re_find = ngx.re.find
     function foo() ... end
end
function bar() ... end
function fn() ... end
</code></pre><p>这样一来，在 <code>main</code> 函数的层面上，就少了 <code>re_find</code> 这个局部变量。这在单个的大的 Lua 文件中，算是一个优化技巧。</p><h2>写在最后</h2><p>从“多问几个为什么”的角度出发，Lua 中 250 这个阈值是从何而来的呢？这算是我们今天的思考题，欢迎你留言说下你的看法，也欢迎你把这篇文章分享给你的同事、朋友，我们一起交流，一起进步。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/7d/17/179b24f4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>燕羽阳</span>
  </div>
  <div class="_2_QraFYR_0">lua指令是32位，其中6位作为操作码，8位作为本地变量和upvalue寻址（即256个）。类似的限制还有函数中只能定义262144个常量（2^18）</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍 这些设计到 Lua 虚拟机指令，感兴趣的同学推荐看看张秀宏老师写的《自己动手实现Lua：虚拟机、编译器和标准库》</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-01 23:21:02</div>
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
  <div class="_2_QraFYR_0">为什么table的下标是数字时，设置弱表属性是key，不会对其进行垃圾回收，而是table和function时会对其进行垃圾回收？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-26 13:02:26</div>
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
  <div class="_2_QraFYR_0">开篇讲解弱表的例子中，如果table中有对Foo的引用，不是说明Foo占用的内存是有用的吗？为什么要释放Foo占用的内存？什么时候会对其进行了引用，反倒希望设置弱表属性，希望GC对其进行垃圾回收呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-26 13:02:19</div>
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
  <div class="_2_QraFYR_0">老师。有个问题，既然优先使用 OpenResty 的 API，文中提到<br>”在查找固定字符串的时候，我们才考虑使用 plain 模式来调用 string 库“<br>这种时候为什么还要考虑使用 plain 模式来调用 string 库呢，也直接使用ngx.re.find不就可以了，反正我们优先使用 OpenResty 的 API</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-02 19:44:57</div>
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
  <div class="_2_QraFYR_0">local t = cjson.empty_array<br>这行代码没看懂</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这是把 cjson 这个模块的 empty_array 函数，赋值给了 t 这个变量。在 Lua 中函数是一等公民，可以像变量一样使用</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-11 11:51:36</div>
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
  <div class="_2_QraFYR_0">老师，我通过resty -e 去测试的时候发现upvalue是有60的限制，但是我用相同的代码通过lua test.lua这种运行方式去测试，发现upvalue没有60限制，函数内部的局部变量两种运行方式到是都有200限制的。<br>lua环境如下：<br>$ lua -v<br>Lua 5.2.4  Copyright (C) 1994-2015 Lua.org, PUC-Rio<br>openresty环境如下：<br>$ resty -v<br>resty 0.23<br>nginx version: openresty&#47;1.15.8.1<br>built by clang 10.0.0 (clang-1000.10.44.4)<br>built with OpenSSL 1.1.0j  20 Nov 2018<br>这是因为luajit和lua的原因吗</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-04 13:29:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJ61zTDmLk7IhLJn6seBPOwsVaKIWUWaxk5YmsdYBZUOYMQCsyl9iaQVSg9U5qJVLLOCFUoLUuYnRA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>fjpcode</span>
  </div>
  <div class="_2_QraFYR_0">既然函数是一等公民，和变量一样，那么在一个函数中调用另一个函数是否也属于闭包的范畴。全局变量是否也属于upvalue。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-09 13:11:30</div>
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
  <div class="_2_QraFYR_0">coroutine 0:<br>  &#47;usr&#47;local&#47;openresty&#47;nginx&#47;lua&#47;hello.lua: in main chunk, client: 127.0.0.1, server: localhost, request: &quot;GET &#47;lua HTTP&#47;1.1&quot;, host: &quot;localhost&quot;<br>2019&#47;07&#47;09 14:21:09 [error] 854136#854136: *16 lua entry thread aborted: runtime error: &#47;usr&#47;local&#47;openresty&#47;nginx&#47;lua&#47;hello.lua:17: attempt to index global &#39;cjson&#39; (a nil v<br>stack traceback,老师，这个是怎么回事</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 能否贴下具体的代码？给个 github 的 gist 地址就行</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-09 14:23:30</div>
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
  <div class="_2_QraFYR_0">上面有个例子， 发现是有两个结果的， 一个是2， 另一个结果是空， 而例子只列出一个结果， 而 第一个print 才是2， 第二个print 是空。<br><br>localhost: ~&#47;geektime&#47;lua $ resty -e &#39;<br>&gt; local function foo()<br>&gt;      local i = 1<br>&gt;      local function bar()<br>&gt;          i = i + 1<br>&gt;          print(i)<br>&gt;      end<br>&gt;      return bar<br>&gt; end<br>&gt; local fn = foo()<br>&gt; print(fn())<br>&gt; &#39;<br>2<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-02 21:05:01</div>
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
  <div class="_2_QraFYR_0">老师，typo：”即使这个变量并不在 foo 里面定义“，应该是”即使这个变量并不在 bar 里面定义“~~</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 多谢反馈，已经修改：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-26 13:02:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/45/2a/c4413de4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>soooldier</span>
  </div>
  <div class="_2_QraFYR_0">看完后对week table还是完全懵的，有没有更浅显易懂的文章呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以看 Lua 作者写的那本书，就是《Lua 程序设计》，里面专门有讲 weak table</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-24 22:46:59</div>
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
  <div class="_2_QraFYR_0">写在最后的do end 是不是表示代码块的意思？<br>do  end封装后是不是其它function是无法访问的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，除非你在 do end 之前预定义了一个变量，比如：<br>local f<br>do <br>function f()<br>...<br>end<br><br>end -- do</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-22 20:46:18</div>
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
  <div class="_2_QraFYR_0">local cjson = require(&#39;cjson&#39;)<br>local tb1 = {}<br>setmetatable(tb1, cjson.empty_array_mt)<br>print(cjson.encode(tb1))<br>tb1[&#39;key&#39;] = &#39;11&#39;<br>print(cjson.encode(tb1))<br><br>老师这样写之后打印出来 4个 [] 空数组，最后那个那个不是赋值了吗？ 为什么还是空数组呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-22 08:10:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/97/69/80945634.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>罐头瓶子</span>
  </div>
  <div class="_2_QraFYR_0">upvalue 上线250这个是考虑到变量查找效率的问题？如果local变量过多可以放在table里面加快查找效率？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 其实上限是 256， 这个是 Lua 虚拟机的指令占用的大小决定的。超过 256 的话，Lua 虚拟机就不支持了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-21 22:50:13</div>
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
  <div class="_2_QraFYR_0">老师，我这么理解对不对：<br>OpenResty的API包括Nginx API和LuaJIT API，Nginx API主要是lua-nginx-module等*-nginx-module C模块提供的API，而LuaJIT API主要是lua-resty-*各种Lua库所提供的API</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-21 11:44:15</div>
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
  <div class="_2_QraFYR_0">老师，文中提到的OpenResty的API和LuaJIT的API，你看我这么理解对不对：<br>OpenResty的API，是指lua-nginx-module模块提供的API<br>LuaJIT的API，是指lua-resty-core和各种lua-resty-*项目中提供的API</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: OpenResty 的 API 指的是lua-nginx-module和lua-resty-core提供的接口，都是 `ngx` 来头的；LuaJIT 的 API 是扩展了 Lua 内置的库，比如 `table.new` 这种</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-21 10:25:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/cf/96/251c0cee.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>xindoo</span>
  </div>
  <div class="_2_QraFYR_0">我立即lua中的弱表和java中的WeakHashMap类似</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-21 08:51:33</div>
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
  <div class="_2_QraFYR_0">当 __mode = k 时 ，table 的键k是弱引用, 这句话不知道怎么理解， 键也可以计数吗？ 键也可以弱引用吗？<br>table 的 值 v是弱引用, 可以理解的， 当回收的时候 赋值为 nil</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-21 08:28:37</div>
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
  <div class="_2_QraFYR_0">lua weak table  可以类比 js 的 weak set ,其中的元素不会加入引用计数，当元素没有其他引用的时候，就会被 gc 掉</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-21 08:25:04</div>
  </div>
</div>
</div>
</li>
</ul>