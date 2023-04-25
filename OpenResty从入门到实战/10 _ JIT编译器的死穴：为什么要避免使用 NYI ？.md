<audio title="10 _ JIT编译器的死穴：为什么要避免使用 NYI ？" src="https://static001.geekbang.org/resource/audio/88/65/8881c2d03baebc07deba8f2295dd5f65.mp3" controls="controls"></audio> 
<p>你好，我是温铭。</p><p>上一节，我们一起了解了 LuaJIT 中的 FFI。如果你的项目中只用到了 OpenResty 提供的 API，没有自己调用 C 函数的需求，那么 FFI 对你而言并没有那么重要，你只需要确保开启了 <code>lua-resty-core</code> 即可。</p><p>但我们今天要讲的 LuaJIT 中 NYI，却是每一个使用 OpenResty 的工程师都逃避不了的关键问题，它对于性能的影响举足轻重。</p><p><strong>你可以很快使用 OpenResty 写出逻辑正确的代码，但不明白 NYI，你就不能写出高效的代码，无法发挥 OpenResty 真正的威力</strong>。这两者的性能差距，至少是一个数量级的。</p><h2>什么是 NYI？</h2><p>那究竟什么是 NYI 呢？先回顾下我们之前提到过的一个知识点：</p><p><strong>LuaJIT 的运行时环境，除了一个汇编实现的 Lua 解释器外，还有一个可以直接生成机器代码的 JIT 编译器。</strong></p><p>LuaJIT 中 JIT 编译器的实现还不完善，有一些原语它还无法编译，因为这些原语实现起来比较困难，再加上 LuaJIT 的作者目前处于半退休状态。这些原语包括常见的 pairs() 函数、unpack() 函数、基于 Lua CFunction 实现的 Lua C 模块等。这样一来，当 JIT 编译器在当前代码路径上遇到它不支持的操作时，便会退回到解释器模式。</p><!-- [[[read_end]]] --><p>而JIT 编译器不支持的这些原语，其实就是我们今天要讲的 NYI，全称为Not Yet Implemented。LuaJIT 的官网上有<a href="http://wiki.luajit.org/NYI">这些 NYI 的完整列表</a>，建议你仔细浏览一遍。当然，目的不是让你背下这个列表的内容，而是让你要在写代码的时候有意识地提醒自己。</p><p>下面，我截取了 NYI 列表中 string 库的几个函数：</p><p><img src="https://static001.geekbang.org/resource/image/1b/91/1b15183f8282ce235379281a961bd991.png?wh=1020*372" alt=""></p><p>其中，<code>string.byte</code> 对应的能否被编译的状态是 <code>yes</code>，表明可以被 JIT，你可以放心大胆地在代码中使用。</p><p><code>string.char</code> 对应的编译状态是 <code>2.1</code>，表明从 LuaJIT 2.1开始支持。我们知道，OpenResty 中的 LuaJIT 是基于 LuaJIT 2.1 的，所以你也可以放心使用。</p><p><code>string.dump</code> 对应的编译状态是 <code>never</code>，即不会被 JIT，会退回到解释器模式。目前来看，未来也没有计划支持这个原语。</p><p><code>string.find</code> 对应的编译状态是 <code>2.1 partial</code>，意思是从 LuaJIT 2.1 开始部分支持，后面的备注中写的是 <code>只支持搜索固定的字符串，不支持模式匹配</code>。所以对于固定字符串的查找，你使用 <code>string.find</code> 是可以被 JIT 的。</p><p>我们自然应该避免使用 NYI，让更多的代码可以被 JIT 编译，这样性能才能得到保证。但在现实环境中，我们有时候不可避免要用到一些 NYI 函数的功能，这时又该怎么办呢？</p><h2>NYI 的替代方案</h2><p>其实，不用担心，大部分 NYI 函数我们都可以敬而远之，通过其他方式来实现它们的功能。接下来，我挑选了几个典型的NYI来讲解，带你了解不同类型的NYI 替代方案。这样，其他的 NYI 你也可以自己触类旁通。</p><h3>1.string.gsub() 函数</h3><p>第一个我们来看string.gsub() 函数。它是 Lua 内置的字符串操作函数，作用是做全局的字符串替换，比如下面这个例子：</p><pre><code>$ resty -e 'local new = string.gsub(&quot;banana&quot;, &quot;a&quot;, &quot;A&quot;); print(new)'
bAnAnA
</code></pre><p>这个函数是一个 NYI 原语，无法被 JIT 编译。</p><p>我们可以尝试在 OpenResty 自己的 API 中寻找替代函数，但对于大多数人来说，记住所有的 API 和用法是不现实的。所以在平时开发中，我都会打开 lua-nginx-module 的 <a href="https://github.com/openresty/lua-nginx-module">GitHub 文档页面</a>。</p><p>比如，针对刚刚的这个例子，我们可以用 <code>gsub</code> 作为关键字，在文档页面中搜索，这时<code>ngx.re.gsub</code> 就会映入眼帘。</p><p>细心的同学可能会问，这里为什么不用之前推荐的 <code>restydoc</code> 工具，来搜索 OpenResty API 呢？你可以尝试下用它来搜索 <code>gsub</code>：</p><pre><code>$ restydoc -s gsub
</code></pre><p>看到了吧，这里并没有返回我们期望的 <code>ngx.re.gsub</code>，而是显示了 Lua 自带的函数。事实上，现阶段而言， <code>restydoc</code> 返回的是唯一的精准匹配的结果，所以它更适合在你明确知道 API 名字的前提下使用。至于模糊的搜索，还是要自己手动在文档中进行。</p><p>回到刚刚的搜索结果，我们看到，<code>ngx.re.gsub</code> 的函数定义如下：</p><blockquote>
<p>newstr, n, err = ngx.re.gsub(subject, regex, replace, options?)</p>
</blockquote><p>这里，函数参数和返回值的命名都带有具体的含义。其实，在 OpenResty 中，我并不推荐你写很多注释，大多数时候，一个好的命名胜过好几行注释。</p><p>对于不熟悉 OpenResty 正则体系的工程师而言，看到最后的变参 <code>options</code> ，你可能会比较困惑。不过，这个变参的解释，并不在此函数中，而是在 <code>ngx.re.match</code> 函数的文档中。</p><p>通过查看参数 <code>options</code> 的文档，你会发现，只要我们把它设置为 <code>jo</code>，就开启了PCRE 的 JIT。这样，使用 <code>ngx.re.gsub</code> 的代码，既可以被 LuaJIT 进行 JIT 编译，也可以被 PCRE JIT 进行 JIT 编译。</p><p>具体的文档内容，我就不再赘述了。不过这里我想强调一点——在翻看文档时，我们一定要有打破砂锅问到底的精神。OpenResty 的文档其实非常完善，仔细阅读文档，就可以解决你大部分的问题。</p><h3>2.string.find() 函数</h3><p>和 <code>string.gsub</code> 不同的是，<code>string.find</code> 在 plain 模式（即固定字符串的查找）下，是可以被JIT 的；而带有正则这种的字符串查找，<code>string.find</code> 并不能被 JIT ，这时就要换用 OpenResty 自己的 API，也就是 <code>ngx.re.find</code> 来完成。</p><p>所以，当你在 OpenResty 中做字符串查找时，首先一定要明确区分，你要查找的是固定的字符串，还是正则表达式。如果是前者，就要用 <code>string.find</code>，并且记得把最后的 plain 设置为 true：</p><pre><code>string.find(&quot;foo bar&quot;, &quot;foo&quot;, 1, true)
</code></pre><p>如果是后者，你应该用 OpenResty 自己的 API，并开启 PCRE 的 JIT 选项：</p><pre><code>ngx.re.find(&quot;foo bar&quot;, &quot;^foo&quot;, &quot;jo&quot;)
</code></pre><p>其实，<strong>这里更适合做一层封装，并把优化选项默认打开，不要让最终的使用者知道这么多细节</strong>。这样，对外就是统一的字符串查找函数了。你可以感受到，有时候选择太多、太灵活并不是一件好事。</p><h3>3.unpack() 函数</h3><p>第三个我们来看unpack() 函数。unpack() 也是要避免使用的函数，特别是不要在循环体中使用。你可以改用数组的下标去访问，比如下面代码的这个例子：</p><pre><code>$ resty -e '
 local a = {100, 200, 300, 400}
 for i = 1, 2 do
    print(unpack(a))
 end'

$ resty -e 'local a = {100, 200, 300, 400}
 for i = 1, 2 do
    print(a[1], a[2], a[3], a[4])
 end'
</code></pre><p>让我们再深究一下 unpack，这次我们可以用<code>restydoc</code> 来搜索一下：</p><pre><code>$ restydoc -s unpack
</code></pre><p>从 unpack 的文档中，你可以看出，<code>unpack (list [, i [, j]])</code> 和 <code>return list[i], list[i+1], , list[j]</code> 是等价的，你可以把 <code>unpack</code> 看成一个语法糖。这样，你完全可以用数组下标的方式来访问，以免打断 LuaJIT 的 JIT 编译。</p><h3>4.pairs() 函数</h3><p>最后我们来看遍历哈希表的 pairs() 函数，它也不能被 JIT 编译。</p><p>不过非常遗憾，这个并没有等价的替代方案，你只能尽量避免使用，或者改用数字下标访问的数组，特别是在热代码路径上不要遍历哈希表。这里我解释一下<strong>代码热路径，它的意思是，这段代码会被返回执行很多次，比如在一个很大的循环里面。</strong></p><p>说完这四个例子，我们来总结一下，要想规避 NYI 原语的使用，你需要注意下面这两点：</p><ul>
<li>请优先使用 OpenResty 提供的 API，而不是 Lua 的标准库函数。这里要牢记， Lua 是嵌入式语言，我们实际上是在 OpenResty 中编程，而不是 Lua。</li>
<li>如果万不得已要使用 NYI 原语，请一定确保它没有在代码热路径上。</li>
</ul><h2>如何检测 NYI？</h2><p>讲了这么多NYI 的规避方案，都是在教你该怎么做。不过，如果到这里戛然而止，那就不太符合 OpenResty 奉行的一个哲学：</p><p><strong>能让机器自动完成的，就不要人工参与。</strong></p><p>人不是机器，总会有疏漏，能够自动化地检测代码中使用到的 NYI，才是工程师价值的一个重要体现。</p><p>这里我推荐，LuaJIT 自带的 <code>jit.dump</code> 和 <code>jit.v</code> 模块。它们都可以打印出 JIT 编译器工作的过程。前者会输出非常详细的信息，可以用来调试 LuaJIT 本身，你可以参考<a href="https://github.com/openresty/luajit2/blob/v2.1-agentzh/src/jit/dump.lua">它的源码</a>来做更深入的了解；后者的输出比较简单，每行对应一个 trace，通常用来检测是否可以被 JIT。</p><p>具体应该怎么操作呢？</p><p>我们可以先在 <code>init_by_lua</code> 中，添加以下两行代码：</p><pre><code>local v = require &quot;jit.v&quot;
v.on(&quot;/tmp/jit.log&quot;)
</code></pre><p>然后，运行你自己的压力测试工具，或者跑几百个单元测试集，让 LuaJIT 足够热，触发 JIT 编译。这些都完成后，再来检查 <code>/tmp/jit.log</code> 的结果。</p><p>当然，这个方法相对比较繁琐，如果你想要简单验证的话， 使用 <code>resty</code> 就足够了，这个 OpenResty 的 CLI 带有相关选项：</p><pre><code>$resty -j v -e 'for i=1, 1000 do
      local newstr, n, err = ngx.re.gsub(&quot;hello, world&quot;, &quot;([a-z])[a-z]+&quot;, &quot;[$0,$1]&quot;, &quot;i&quot;)
 end'
 [TRACE   1 (command line -e):1 stitch C:107bc91fd]
 [TRACE   2 (1/stitch) (command line -e):2 -&gt; 1]
</code></pre><p>其中，<code>resty</code> 的 <code>-j</code> 就是和 LuaJIT 相关的选项；后面的值为 <code>dump</code> 和 <code>v</code>，就对应着开启 <code>jit.dump</code> 和 <code>jit.v</code> 模式。</p><p>在 jit.v 模块的输出中，每一行都是一个成功编译的 trace 对象。刚刚是一个能够被 JIT 的例子，而如果遇到 NYI 原语，输出里面就会指明 NYI，比如下面这个 <code>pairs</code> 的例子：</p><pre><code>$resty -j v -e 'local t = {}
 for i=1,100 do
     t[i] = i
 end
 
 for i=1, 1000 do
     for j=1,1000 do
         for k,v in pairs(t) do
             --
         end
     end
 end'
</code></pre><p>它就不能被 JIT，所以结果里，指明了第 8 行中有 NYI 原语。</p><pre><code> [TRACE   1 (command line -e):2 loop]
 [TRACE --- (command line -e):7 -- NYI: bytecode 72 at (command line -e):8]
</code></pre><h2>写在最后</h2><p>这是我们第一次用比较多的篇幅来谈及 OpenResty 的性能问题。看完这些关于 NYI 的优化，不知道你有什么感想呢？可以留言说说你的看法。</p><p>最后，给你留一道思考题。在讲 string.find() 函数的替代方案时，我有提到过，那里其实<strong>更适合做一层封装，并默认打开优化选项</strong>。那么，这个任务就交给你来小试牛刀了。</p><p>欢迎在留言区写下你的答案，也欢迎你把这篇文章分享给你的同事、朋友，一起交流，一起进步。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/b2/e0/d856f5a4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>鱼</span>
  </div>
  <div class="_2_QraFYR_0">对于一个OpenResty入门者一开始讲这些性能和低层的知识有些枯燥了，毕竟还没写过几个OpenResty的实例，对性能的差异没什么感觉，基础知识还没掌握全更难深入底层。有如学习java的初学者一开始就看《Java编程思想》。可否尝试在后面的实际例子中引出这些知识点。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 后面会专门介绍 OpenResty 的 Lua API。LuaJIT 的这些内容有个印象即可。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-17 10:49:54</div>
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
  <div class="_2_QraFYR_0">封装字符串查找函数：<br>local function new_string_find(src, dst, regex, pos)<br>	if regex then<br>		local ctx = {pos = pos or 1}<br>		return ngx.re.find(src, dst, &quot;jo&quot;, ctx)<br>	else<br>		local pos = pos or 1<br>		return string.find(src, dst, pos, true)<br>	end<br>end</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-17 13:11:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/7d/17/179b24f4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>燕羽阳</span>
  </div>
  <div class="_2_QraFYR_0">1. lua的语法比较简单，如果有编程基础的话，一天就可以入门了。推荐大家两本书《lua程序设计》lua入门必备，《lua设计与实现》深入解释器虚拟机的原理。<br>2. 之前写过一点openresty和kong，但是从没注意过性能问题。今天的NYI，老师讲的非常棒，完整的实战方案，超赞👍<br>3.我对jit完全不熟悉，请问老师，jit是按照函数为单位来编译么？函数中有一个NYI，整个函数就是解释运行么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是按照原语来编译的，也就是 `ipairs` 、`string.find` 这种的颗粒度，并不是你自己写的 function</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-17 22:21:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/27/40/ae886719.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>geekYang</span>
  </div>
  <div class="_2_QraFYR_0">老师，NYI 的替代方案为什么不去看 lua-resty-core，而要在lua-nginx-module中寻找？ ngx.re.gsub 为什么即可以pcre编译，也可以luajit 编译？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-15 23:45:19</div>
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
  <div class="_2_QraFYR_0">老师，下面这个设置lua库的搜索路径的代码，最后为什么要拼接package.path本身啊？<br>package.path = &quot;..&#47;myLuaTest&#47;myLuaCode&#47;?.lua;&quot;..package.path</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: package.path 是默认的查找路径，你可以单独把它 print 出来看下里面的值。不拼接的话，就把默认查找路径全都覆盖掉了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-17 19:44:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/a5/ff/6201122c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_89bbab</span>
  </div>
  <div class="_2_QraFYR_0">希望老师分享一下写 openresty代码时用什么编辑器比较好，代码补全，定义跳转,引用跳转等如何配置。老师你们写openresty的时候是怎么来做的？直接vim操作，还是有更智能些的IDE？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 现在还没啥好用的编辑器，我用的是 vs code</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-17 19:06:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_2b5c15</span>
  </div>
  <div class="_2_QraFYR_0">Accroding from the new NYI list, the paris have been implement. </div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-14 16:22:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/d0/46/1a9229b3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>NEVER SETTLE</span>
  </div>
  <div class="_2_QraFYR_0">经常会发现openresty运行一段时间，性能会差很多。然后reload一下性能会好很多。这个问题一直困扰很久了，不知道如何进行排查。 以前两天的情况，高峰期CPU使用率超过了60%，然后reload一下，CPU使用率就降为30%。 </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 需要用火焰图分析下 on-cpu 才行</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-19 20:01:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/90/d3/da371457.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Forturn</span>
  </div>
  <div class="_2_QraFYR_0">楼上说的有道理，目前还没掌握基本的语法，还不会写一些基本的功能，就去学底层的东西，有点摸不着头脑，也不懂这些。不知道后面会不会具体事例</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 看不懂没有关系，重要的是记得有LuaJIT 和 NYI 这个东西，后面遇到问题方便查找。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-17 11:43:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLUfAPgMtkWQcGk45nLULtb4WtOotH2Mc5ibEiaKcnopzicdAn6WYibo5gQL8bkedRib3YjxfCcZfLJwLA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>MiaoVictor</span>
  </div>
  <div class="_2_QraFYR_0">老师，请问在查看NYI列表是，某些原语后是“2.1 stitch”，这个stitch是什么意思呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-07 23:34:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/96/d0/abb7bfe3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>KoALa</span>
  </div>
  <div class="_2_QraFYR_0">教程中反复提到了“LuaJIT 的作者目前处于半退休状态”，感觉这个情况很不乐观啊...</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Kong 和 OpenResty 的团队中，都有人在逐步接手。开源项目只要有人在使用，就不会死掉，不用担心。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-25 09:30:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/5d/44/74173e02.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>莫然</span>
  </div>
  <div class="_2_QraFYR_0">pairs的例子稍作修改，如下：<br>local t = {}<br>for i=1,100 do<br>     t[i] = i<br>end<br> <br>for j=1,10 do<br>    for k,v in pairs(t) do<br>         --<br>    end<br>end<br>jit.log里输出的是：[TRACE   1 t.lua:123 loop]<br><br>如果改成如下代码：<br>for j=1,100 do<br>    for k,v in pairs(t) do<br>         --<br>    end<br>end<br>jit.log里输出的是：<br>[TRACE   1 t.lua:123 loop]<br>[TRACE --- t.lua:128 -- NYI: bytecode 72 at t.lua:129]<br>[TRACE --- t.lua:128 -- NYI: bytecode 72 at t.lua:129]<br><br>为什么循环10的时候没有NYI的提示？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: LuaJIT 的优化是随机触发的，要足够热才可能尝试去优化</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-11 16:31:40</div>
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
  <div class="_2_QraFYR_0">老师，我在用op实现一个自定义逻辑的风控系统，想请教一下动态配置生效的问题，如何在不reload的情况下使配置生效，我想了两个办法：1，使用全局变量存储配置文件，提供一个api更新全局变量；2，使用ngx.shared.DICT，将配置文件存储在共享内存中。请问是否合理，或者是否有其它思路</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你可以使用一个 timer，定时的查询是否有新的配置，并把新配置写到类似 shared dict 的缓存中。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-18 11:42:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLh8ubWQtDRa6exJtloSwibLliaejpF7434ficyggzukmXE63UlSPvbykoiaVDZo4CbDIIOQsCkicibyn9A/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>涉蓝</span>
  </div>
  <div class="_2_QraFYR_0">对于初学者是不是 要先学 lua -&gt; luajit -&gt; openresty api -&gt; 其他第三方包 <br>按这种先把 文档啥的都扒一遍才行呢？<br>我对于使用有点疑惑，openresty可以做 网站普通后端语言可以做的事 譬如连数据库做前端页面 但这显然并不是它主要适合的部分吧 毕竟其他后端语言一大把，文档生态库好的多的是 <br>所以是API 网关的 开发 或者 nginx 无法配置热修改的 补充吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个问题其实要回到 OpenResty 诞生的那个时候来看， 2007 年，支持同步非阻塞的语言凤毛麟角。<br>即使是现在，后端语言可以达到 OpenResty 这种性能级别的也不多。<br>API 网关和软 WAF 算是开发者的自然选择，OpenResty 其实能做的不止这些。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-17 20:24:43</div>
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
  <div class="_2_QraFYR_0">请教一下老师，当op中的lua规则和nginx配置文件产生冲突，比如nginx配置了rewrite规则，又同时引用了rewrite_by_lua_file,那么这两条规则的优先级是什么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个具体要看 nginx 配置的 rewrite 规则是怎么写的了，是 break 还是 last。这个在官方文档中有注明，并且配了一个示例代码：<br> location &#47;foo {<br>     rewrite ^ &#47;bar;<br>     rewrite_by_lua &#39;ngx.exit(503)&#39;;<br> }<br> location &#47;bar {<br>     ...<br> }<br>上面这个配置中，ngx.exit(503) 是不会被执行的。但是，如果改成：<br>rewrite ^ &#47;bar break；<br><br>ngx.exit(503) 就是可以执行的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-17 14:26:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Mluia3Ebv2LYrLcPBCw0jTINxIricFeiaaE9Q8d1MNEktWEiaiavOicxuTtsaQ8D3l4To5ca6Wh40ibFsz5f9DFJdEKtA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>skrbug</span>
  </div>
  <div class="_2_QraFYR_0">LuaJit 的NYI，官网链接失效了老师</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-29 16:34:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/28/73/9a/5197bbbd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张靳</span>
  </div>
  <div class="_2_QraFYR_0">老师，我想问下，如果是一个完整的API网关，里面写了很多的lua模块，如何比较简单快速的排查NYI？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-02 21:48:35</div>
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
  <div class="_2_QraFYR_0">老师讲的非常好，做技术就要深挖底层原理，而且目前讲的也没有到底层<br>小白就知道急功近利，我只能说项目有问题谁来背锅？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-25 08:41:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/jaOQZZRldayELmckBTq4VQHUgepwn8ibSLuEEB0NNwRnMyIqRRPj3ms38cjj3ySicEzkruV44lxDMdbdgUkn6ibTg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_e59303</span>
  </div>
  <div class="_2_QraFYR_0">什么是代码热路径啊</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-12 10:29:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/24/a8/34cd2c95.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>sheng</span>
  </div>
  <div class="_2_QraFYR_0">这里的一个数量级是指十倍的意思么？<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-09 13:25:22</div>
  </div>
</div>
</div>
</li>
</ul>