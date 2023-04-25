<audio title="08 _ LuaJIT分支和标准Lua有什么不同？" src="https://static001.geekbang.org/resource/audio/25/93/257ae12a315f7eee7ac83f171c932e93.mp3" controls="controls"></audio> 
<p>你好，我是温铭。</p><p>这节课，我们来学习下 OpenResty 的另一块基石：LuaJIT。今天主要的篇幅，我会留给 Lua 和 LuaJIT 中重要和鲜为人知的一些知识点。而更多 Lua 语言的基础知识，你可以通过搜索引擎或者 Lua 的书籍自己来学习，这里我推荐 Lua 作者编写的《Lua 程序设计》这本书。</p><p><strong>当然，在 OpenResty 中，写出正确的 LuaJIT 代码的门槛并不高，但要写出高效的 LuaJIT 代码绝非易事</strong>，这里的关键内容，我会在后面 OpenResty 性能优化部分详细介绍。</p><p>我们先来看下 LuaJIT 在 OpenResty 整体架构中的位置：</p><p><img src="https://static001.geekbang.org/resource/image/cd/ef/cdef970a60810548b9c297e6959671ef.png?wh=1180*874" alt=""></p><p>前面我们提到过，OpenResty 的 worker 进程都是 fork master 进程而得到的， 其实， master 进程中的 LuaJIT 虚拟机也会一起 fork 过来。在同一个 worker 内的所有协程，都会共享这个 LuaJIT 虚拟机，Lua 代码的执行也是在这个虚拟机中完成的。</p><p>这可以算是 OpenResty 的基本原理，后面课程我们再详细聊聊。今天我们先来理顺 Lua 和 LuaJIT 的关系。</p><h2>标准 Lua 和 LuaJIT 的关系</h2><p>先把重要的事情放在前面说：</p><p><strong><span class="orange">标准 Lua 和 LuaJIT 是两回事儿，LuaJIT 只是兼容了 Lua 5.1 的语法。</span></strong></p><!-- [[[read_end]]] --><p>标准 Lua 现在的最新版本是 5.3，LuaJIT 的最新版本则是 2.1.0-beta3。在 OpenResty 几年前的老版本中，编译的时候，你可以选择使用标准 Lua VM ，或者 LuaJIT VM 来作为执行环境，不过，现在已经去掉了对标准 Lua 的支持，只支持 LuaJIT。</p><p>LuaJIT 的语法兼容 Lua 5.1，并对 Lua 5.2 和 5.3 做了选择性支持。所以我们应该先学习 Lua 5.1 的语法，并在此基础上学习 LuaJIT 的特性。上节课我已经带你入门了 Lua的基础语法，今天只提及Lua的一些特别之处。</p><p>值得注意的是，OpenResty 并没有直接使用 LuaJIT 官方提供的 2.1.0-beta3 版本，而是在此基础上，扩展了自己的 fork: [openresty-luajit2]：</p><blockquote>
<p>OpenResty 维护了自己的 LuaJIT 分支，并扩展了很多独有的 API。</p>
</blockquote><p>这些独有的 API，都是在实际开发 OpenResty 的过程中，出于性能方面的考虑而增加的。<strong>所以，我们后面提到的 LuaJIT，特指 OpenResty 自己维护的 LuaJIT 分支。</strong></p><h2>为什么选择 LuaJIT？</h2><p>说了这么多 LuaJIT和Lua 的关系，你可能会纳闷儿，为什么不直接使用Lua，而是要用自己维护的LuaJIT呢？其实，最主要的原因，还是LuaJIT的性能优势。</p><p>其实标准 Lua 出于性能考虑，也内置了虚拟机，所以 Lua 代码并不是直接被解释执行的，而是先由 Lua 编译器编译为字节码（Byte Code），然后再由 Lua 虚拟机执行。</p><p>而 LuaJIT 的运行时环境，除了一个汇编实现的 Lua 解释器外，还有一个可以直接生成机器代码的 JIT 编译器。开始的时候，LuaJIT和标准 Lua 一样，Lua 代码被编译为字节码，字节码被 LuaJIT 的解释器解释执行。</p><p>但不同的是，LuaJIT的解释器会在执行字节码的同时，记录一些运行时的统计信息，比如每个 Lua 函数调用入口的实际运行次数，还有每个 Lua 循环的实际执行次数。当这些次数超过某个随机的阈值时，便认为对应的 Lua 函数入口或者对应的 Lua 循环足够热，这时便会触发 JIT 编译器开始工作。</p><p>JIT 编译器会从热函数的入口或者热循环的某个位置开始，尝试编译对应的 Lua 代码路径。编译的过程，是把 LuaJIT 字节码先转换成LuaJIT 自己定义的中间码（IR），然后再生成针对目标体系结构的机器码。</p><p>所以，<strong>所谓 LuaJIT 的性能优化，本质上就是让尽可能多的 Lua 代码可以被 JIT 编译器生成机器码，而不是回退到 Lua 解释器的解释执行模式</strong>。明白了这个道理，你才能理解后面学到的OpenResty 性能优化的本质。</p><h2>Lua 特别之处</h2><p>正如我们上节课介绍的一样，Lua 语言相对简单。对于有其他开发语言背景的工程师来说，注意 到Lua 中一些独特的地方后，你就能很容易的看懂代码逻辑。接下来，我们一起来看Lua语言比较特别的几个地方。</p><h3>1. Lua 的下标从 1 开始</h3><p>Lua 是我知道的唯一一个下标从 1 开始的编程语言。这一点，虽然对于非程序员背景的人来说更好理解，但却容易导致程序的 bug。</p><p>下面是一个例子：</p><pre><code>$ resty -e 't={100}; ngx.say(t[0])'
</code></pre><p>你自然期望打印出 <code>100</code>，或者报错说下标 0 不存在。但结果出乎意料，什么都没有打印出来，也没有报错。既然如此，让我们加上 <code>type</code> 命令，来看下输出到底是什么：</p><pre><code>$ resty -e 't={100};ngx.say(type(t[0]))'
nil
</code></pre><p>原来是空值。事实上，在 OpenResty 中，对于空值的判断和处理也是一个容易让人迷惑的点，后面我们讲到 OpenResty 的时候再细聊。</p><h3>2. 使用 <code>..</code> 来拼接字符串</h3><p>这一点，上节课我也提到过。和大部分语言使用 <code>+</code> 不同，Lua 中使用两个点号来拼接字符串：</p><pre><code>$ resty -e &quot;ngx.say('hello' .. ', world')&quot;
hello, world
</code></pre><p>在实际的项目开发中，我们一般都会使用多种开发语言，而Lua 这种不走寻常路的设计，总是会让开发者的思维，在字符串拼接的时候卡顿一下，也是让人哭笑不得。</p><h3>3. 只有 <code>table</code> 这一种数据结构</h3><p>不同于 Python 这种内置数据结构丰富的语言，Lua 中只有一种数据结构，那就是 table，它里面可以包括数组和哈希表：</p><pre><code>local color = {first = &quot;red&quot;, &quot;blue&quot;, third = &quot;green&quot;, &quot;yellow&quot;}
print(color[&quot;first&quot;])                 --&gt; output: red
print(color[1])                         --&gt; output: blue
print(color[&quot;third&quot;])                --&gt; output: green
print(color[2])                         --&gt; output: yellow
print(color[3])                         --&gt; output: nil
</code></pre><p>如果不显式地用<code>_键值对_</code>的方式赋值，table 就会默认用数字作为下标，从 1 开始。所以 <code>color[1]</code> 就是 blue。</p><p>另外，想在 table 中获取到正确长度，也是一件不容易的事情，我们来看下面这些例子：</p><pre><code>local t1 = { 1, 2, 3 }
print(&quot;Test1 &quot; .. table.getn(t1))

local t2 = { 1, a = 2, 3 }
print(&quot;Test2 &quot; .. table.getn(t2))

local t3 = { 1, nil }
print(&quot;Test3 &quot; .. table.getn(t3))

local t4 = { 1, nil, 2 }
print(&quot;Test4 &quot; .. table.getn(t4))
</code></pre><p>使用 <code>resty</code> 运行的结果如下：</p><pre><code>Test1 3
Test2 2
Test3 1
Test4 1
</code></pre><p>你可以看到，除了第一个返回长度为 3 的测试案例外，后面的测试都是我们预期之外的结果。事实上，想要在Lua 中获取 table 长度，必须注意到，只有在 table 是 <code>_序列_</code> 的时候，才能返回正确的值。</p><p>那什么是序列呢？首先序列是数组（array）的子集，也就是说，table 中的元素都可以用正整数下标访问到，不存在键值对的情况。对应到上面的代码中，除了 t2 外，其他的 table 都是 array。</p><p>其次，序列中不包含空洞（hole），即 nil。综合这两点来看，上面的 table 中， t1 是一个序列，而 t3 和 t4 是 array，却不是序列（sequence）。</p><p>到这里，你可能还有一个疑问，为什么 t4 的长度会是 1 呢？其实这是因为，在遇到 nil 时，获取长度的逻辑就不继续往下运行，而是直接返回了。</p><p>不知道你完全看懂了吗？这部分确实相当复杂。那么有没有什么办法可以获取到我们想要的 table 长度呢？自然是有的，OpenResty 在这方面做了扩展，在后面专门的 table 章节我会讲到，这里先留一个悬念。</p><h3>4. 默认是全局变量</h3><p>我想先强调一点，除非你相当确定，否则在 Lua 中声明变量时，前面都要加上 <code>local</code>：</p><pre><code>local s = 'hello'
</code></pre><p>这是因为在 Lua 中，变量默认是全局的，会被放到名为 <code>_G</code> 的 table 中。不加 local 的变量会在全局表中查找，这是昂贵的操作。如果再加上一些变量名的拼写错误，就会造成难以定位的 bug。</p><p>所以，在 OpenResty 编程中，我强烈建议你总是使用 <code>local</code> 来声明变量，即使在 require module 的时候也是一样：</p><pre><code>-- Recommended 
local xxx = require('xxx')

-- Avoid
require('xxx')
</code></pre><h2>LuaJIT</h2><p>明白了Lua这四点特别之处，我们继续来说LuaJIT。除了兼容 Lua 5.1 的语法并支持 JIT 外，LuaJIT 还紧密结合了 FFI（Foreign Function Interface），可以让你直接在 Lua 代码中调用外部的 C 函数和使用 C 的数据结构。</p><p>下面是一个最简单的例子：</p><pre><code>local ffi = require(&quot;ffi&quot;)
ffi.cdef[[
int printf(const char *fmt, ...);
]]
ffi.C.printf(&quot;Hello %s!&quot;, &quot;world&quot;)
</code></pre><p>短短这几行代码，就可以直接在 Lua 中调用 C 的 <code>printf</code> 函数，打印出 <code>Hello world!</code>。你可以使用 <code>resty</code> 命令来运行它，看下是否成功。</p><p>类似的，我们可以用 FFI 来调用 NGINX、OpenSSL 的 C 函数，来完成更多的功能。实际上，FFI 方式比传统的 Lua/C API 方式的性能更优，这也是 <code>lua-resty-core</code> 项目存在的意义。下一节我们就来专门讲讲 FFI 和 <code>lua-resty-core</code>。</p><p>此外，出于性能方面的考虑，LuaJIT 还扩展了 table 的相关函数：<code>table.new</code> 和 <code>table.clear</code>。<strong>这是两个在性能优化方面非常重要的函数</strong>，在 OpenResty 的 lua-resty 库中会被频繁使用。不过，由于相关文档藏得非常深，而且没有示例代码，所以熟悉它们的开发者并不多。我们留到性能优化章节专门来讲它们。</p><h2>写在最后</h2><p>让我们来回顾下今天的内容。</p><p>OpenResty 出于性能的考虑，选择了 LuaJIT 而不是标准 Lua，并且维护了自己的 LuaJIT 分支。而 LuaJIT 基于 Lua 5.1 的语法，并选择性地兼容了部分 Lua5.2 和 Lua5.3 的语法，形成了自己的体系。至于你需要掌握的Lua 语法，在下标、字符串拼接、数据结构和变量上，都有自己鲜明的特点，在写代码的时候你应该特别留意。</p><p>你在学习 Lua 和 LuaJIT 的时候，是否遇到一些陷阱和坑呢？欢迎留言一起来聊一聊，我在后面也专门写了一篇文章，来分享我遇到过的那些坑。也欢迎你把这篇文章分享给你的同事、朋友，一起学习，一起进步。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/aa/b1/52d2871f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>轨迹</span>
  </div>
  <div class="_2_QraFYR_0">老师可以讲一下table的内部结构，把原理搞清楚，就比较容易了。<br>譬如这个文章：https:&#47;&#47;blog.csdn.net&#47;wwlcsdn000&#47;article&#47;details&#47;81291756<br>简单的就是两点：<br>1、两种存储，哈希和数组。<br>2、数组以下标覆盖到哈希，如果遇到key冲突，数组覆盖哈希的value。<br>我个人感觉就更好一点。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 多谢补充 ：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-12 11:30:38</div>
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
  <div class="_2_QraFYR_0">resty -e &quot;local t = {1, nil, name=&#39;cjf&#39;, 2} print(table.getn(t))&quot;<br>3<br><br>老师 测试下这个 咋会出来3呢 不是遇见nil就停止了吗</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-13 10:29:20</div>
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
  <div class="_2_QraFYR_0">例子讲的还是挺清楚的，希望继续讲一下具体openresty实际项目的使用场景案例</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-12 11:46:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/68/41/86b109fb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>逍遥</span>
  </div>
  <div class="_2_QraFYR_0">openresty为什么要维护自己的luajit分支呢，为什么不能用luajit</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 有两个原因：一个是 LuaJIT 的作者基本处于退休状态，只修 bug，不怎么加新功能，关于 LuaJIT 的 bug，OpenResty 还是会提交给LuaJIT 官方的；第二个是新增的主要是 OpenResty 优化中遇到的 API，自己维护更容易控制版本和节奏。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-08 21:53:26</div>
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
  <div class="_2_QraFYR_0">上面说的 luajit 先把 字节码转为中间码爱转为 机器码<br>这个有个以为你，字节码不就是机器码吗？ 不都是二进制的东西吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这里的字节码是 LuaJIT 虚拟机执行的一种指令格式；机器码是指 CPU 可以读取的指令格式。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-12 08:05:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/QFE00aXGzaS6ibbfJSJsDrpIkqs0OrIYjzZv6L9vZmMhOlut2j24iaeZb0MCQazToE6FRXN960nNiaTrsmw09YjGw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>岁月如歌</span>
  </div>
  <div class="_2_QraFYR_0">@空知 提出的问题 有相同的疑问？<br>【<br>resty -e &quot;local t = {1, nil, name=&#39;cjf&#39;, 2} print(table.getn(t))&quot;<br>3<br><br>老师 测试下这个 咋会出来3呢 不是遇见nil就停止了吗<br>】 <br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我建议大家绕着走，把 nil 改为 ngx.null 来填充数组。不同的 lua 版本会有不同的行为，我也不太清楚。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-13 16:29:47</div>
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
  <div class="_2_QraFYR_0">1.&quot;在同一个 worker 内的所有协程&quot;,协程就是常说的线程吧？<br>2.$ resty -e &#39;t={100}; ngx.say(t[0])&#39;<br>这代码，变量定义前面需要加local才能运行。<br>3.“对应到上面的代码中，除了 t2 外，其他的 table 都是 array。”<br>这句话写反了吧，应该是“除了 t2 外，其他的 table 都是 序列。”吧<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-12 00:55:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/0c/a6/f2d8b68e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>林潇</span>
  </div>
  <div class="_2_QraFYR_0">学习lua的时候遇到几个不一样的地方：<br>1. if必须要有end，在python和lua之间切换会很不习惯。<br>2. 一般一个对象访问属性是用冒号:，而不是点.，也会经常性写错。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-20 18:33:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Leo</span>
  </div>
  <div class="_2_QraFYR_0">我遇到的坑是在删除table的时候，用数组下标循环删除成员时，每次只删除一个还好，当删除两个连续成员的时候就会成问题<br>后来发现了，检测第二个元素的时候，已经因为上一个元素的删除，导致后一个符合条件的元素往前窜了一个，就想队列一样，导致元素变量索引减1，导致删不掉的bug<br>后来网上找了下成功经验，把队列模型改成堆栈模型，循环从小变大的规则，改成从大到小循环，即便当前索引的元素删掉，最多影响处理完的数据下标发生变化，不会影响到未处理的元素，挺有意思的。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-01 19:06:55</div>
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
  <div class="_2_QraFYR_0">请问，Lua虚拟机跟Lua解释器是一样的吗，只是表述不同？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-10 11:57:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/21/58/2a/fec1ff54.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>wedvefv</span>
  </div>
  <div class="_2_QraFYR_0">接口文件和模块文件都需要 local print=print吗？  为了避免冲全局表找变量吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-20 19:39:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/7a/0a/0ce5c232.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>吕</span>
  </div>
  <div class="_2_QraFYR_0">这个luajit，不是和java的hotspot一样么，just-in-time,一些是解释执行，然后对于一些热点代码，进行编译执行，这是和java hotspot虚拟机一样的机制，不知道是谁学习的谁</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-16 22:20:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/ac/83/2f143a22.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>雪粮</span>
  </div>
  <div class="_2_QraFYR_0">对于LuaJIT VM， 可以理解为Master进程和它Fork的每个子进程中都有一个独立的LuaJIT VM吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-24 14:29:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/b2/c9/b414a77c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>HelloTalk</span>
  </div>
  <div class="_2_QraFYR_0">原来默认写lua 的时候，需要做二进制 位移、抑或的操作，为此自己写了一个bitop的库，发现性能更不上，后面上了 luajit，测试快了很多。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-20 23:53:39</div>
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
  <div class="_2_QraFYR_0">lua if语句的判断方式和ruby貌似一样，nil和false才为假，其他都为真，用多了个人觉得这种判断方式更合理~~</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 主要是OpenResty 中空值的情况比较多</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-24 19:36:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/63/d5/a300899a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>杨丁</span>
  </div>
  <div class="_2_QraFYR_0">内容通俗易懂，赞。老师什么时候出个限流的实战吧</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 后面有专门的限流限速章节</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-12 07:44:23</div>
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
  <div class="_2_QraFYR_0">有遇到一些让人困惑的地方是ngx.null、nil、null、“”。在网上搜索的时候，有看到说null是ngx.null的一个定义。redis的返回值的时候，经常会判断返回结果是否为空，判断的时候是和哪个值进行比较呢？关于这些个值有没有其他一些使用上的坑呢？一直以来都没有有一个明确的认识，所以和老师确认一下。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 在 lua-resty-redis 里面，查找一个key 的时候：<br>local res, err = red:get(&quot;dog&quot;)<br>如果返回值 res 是 nil，就说明函调用失败了；如果 res 是 ngx.null 就说明redis 中不存在 dog 这个key。<br>而在处理 cjson 的时候，又有cjson.null这个值。<br>所以还是要根据对应库的文档来做空值的判断和区分。<br>在写类似 if not res then 这样的代码的时候，要特别留意下，最好改成明确的： if res ~= nil and res ~= false then ，类似这样的，并有对应的测试案例覆盖。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-15 09:46:04</div>
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
  <div class="_2_QraFYR_0">温铭老师，你好~<br>在“为什么选择luaJIT”的讲解中，讲到如果lua函数入口或者lua循环足够热，JIT编译器会从热函数的入口或者热循环的某个位置开始尝试编译。有以下几个问题请教一下老师。<br>1.这里的“尝试”编译的意思是并不是所有的热函数或者热循环都可以被编译，是吧？<br>2.另一个疑问是为什么一开始不对所有的可以被JIT编译的函数或者循环直接进行编译呢？<br>3.现在的JIT发现热点然后进行编译，为什么判断是否是热点时的统计，是根据一个随机的阈值，而不是一个其他，比如经过测试认为比较合理的值？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: LuaJIT 是会记录并统计代码的运行次数，才能知道哪些是热函数和热循环，这就是“尝试”的含义；<br>如果有些函数只调用了一两次，就没有编译的必要性了；<br>第三个问题，我的猜测可能是处于安全或者类似的考虑，就没有写死一个值，而是随机。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-15 09:37:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/fe/fd/28fd7e5f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>shineyyl</span>
  </div>
  <div class="_2_QraFYR_0">$ resty -e &#39;local t; t={100}; ngx.say(t[1]);print(type(t[1]))&#39;<br>100<br>number<br>$ resty -e &#39;local t; t={100}; ngx.say(t[0]);print(type(t[0]))&#39;<br>       #空<br>nil    #nil，需要print打印出来t的数据类型<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-12 17:16:21</div>
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
  <div class="_2_QraFYR_0">老师 文中说了   序列不应该含有  nil的，array 是可以包含 nil</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-12 07:49:49</div>
  </div>
</div>
</div>
</li>
</ul>