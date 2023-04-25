<audio title="07 _ 带你快速上手 Lua" src="https://static001.geekbang.org/resource/audio/61/ab/61cbc24102eb4f04301c96a0f2f845ab.mp3" controls="controls"></audio> 
<p>你好，我是温铭。</p><p>在大概了解 NGINX 的基础知识后，接下来，我们就要来进一步学习 Lua了。它是 OpenResty 中使用的编程语言，掌握它的基本语法还是很有必要的。</p><p>Lua 是一个小巧精妙的脚本语言，诞生于巴西的大学实验室，这个名字在葡萄牙语里的含义是“美丽的月亮”。从作者所在的国家来看，NGINX 诞生于俄罗斯，Lua 诞生于巴西，OpenResty 诞生于中国，这三门同样精巧的开源技术都出自金砖国家，而不是欧美，也是挺有趣的一件事。</p><p>回到Lua语言上。事实上，Lua 在设计之初，就把自己定位为一个简单、轻量、可嵌入的胶水语言，没有走大而全的路线。虽然你平常工作中可能没有直接编写 Lua 代码，但 Lua 的使用其实非常广泛。很多的网游，比如魔兽世界，都会采用 Lua 来编写插件；而键值数据库 Redis 则是内置了 Lua 来控制逻辑。</p><p>另一方面，虽然 Lua 自身的库比较简单，但它可以方便地调用 C 库，大量成熟的 C 代码都可以为其所用。比如在 OpenResty 中，很多时候都需要你调用 NGINX 和 OpenSSL 的 C 函数，而这都得益于 Lua 和 LuaJIT 这种方便调用 C 库的能力。</p><p>下面，我带你来快速熟悉下 Lua 的数据类型和语法，以便你后面更顺畅地学习 OpenResty。</p><!-- [[[read_end]]] --><h2>环境和 hello world</h2><p>我们不用专门去安装标准 Lua 5.1 之类的环境，因为 OpenResty 已经不再支持标准 Lua，而只支持 LuaJIT。这里我介绍的 Lua 语法，也是和 LuaJIT 兼容的部分，而不是基于最新的 Lua 5.3，这一点需要你特别注意。</p><p>在 OpenResty 的安装目录下，你可以找到 LuaJIT 的目录和可执行文件。我这里是 Mac 环境，使用 brew 安装 OpenResty，所以你本地的路径很可能和下面的不同：</p><pre><code>$ ll /usr/local/Cellar/openresty/1.13.6.2/luajit/bin/luajit
 lrwxr-xr-x  1 ming  admin    18B  4  2 14:54 /usr/local/Cellar/openresty/1.13.6.2/luajit/bin/luajit -&gt; luajit-2.1.0-beta3
</code></pre><p>你也可以在系统的可执行文件目录中找到它：</p><pre><code>$ which luajit
 /usr/local/bin/luajit
</code></pre><p>并查看 LuaJIT 的版本号：</p><pre><code>$ luajit -v
 LuaJIT 2.1.0-beta2 -- Copyright (C) 2005-2017 Mike Pall. http://luajit.org/
</code></pre><p>查清楚这些信息后，你可以新建一个 <code>1.lua</code> 文件，并用 luajit 来运行其中的 hello world 代码：</p><pre><code>$ cat 1.lua
print(&quot;hello world&quot;)

$ luajit 1.lua
 hello world
</code></pre><p>当然，你还可以使用 <code>resty</code> 来直接运行，要知道，它最终也是用 LuaJIT 来执行的：</p><pre><code>$ resty -e 'print(&quot;hello world&quot;)'
 hello world
</code></pre><p>上述两种运行 hello world 的方式都是可行的。不顾对我来说，我更喜欢 <code>resty</code> 这种方式，因为后面很多 OpenResty 的代码，也都是通过 <code>resty</code> 来运行的。</p><h2>数据类型</h2><p>Lua 中的数据类型不多，你可以通过 <code>type</code> 函数来返回一个值的类型，比如下面这样的操作：</p><pre><code>$ resty -e 'print(type(&quot;hello world&quot;)) 
 print(type(print)) 
 print(type(true)) 
 print(type(360.0))
 print(type({}))
 print(type(nil))
 '
</code></pre><p>会打印出如下内容：</p><pre><code> string
 function
 boolean
 number
 table
 nil
</code></pre><p>这几种就是 Lua 中的基本数据类型了。下面我们来简单介绍一下它们。</p><h3>字符串</h3><p>在 Lua 中，字符串是不可变的值，如果你要修改某个字符串，就等于创建了一个新的字符串。这种做法显然有利有弊：好处是即使同一个字符串出现了很多次，在内存中也只有一份；但劣势也很明显，如果你想修改、拼接字符串，会额外地创建很多不必要的字符串。</p><p>我们举一个例子，来说明这个弊端。下面这段代码，是把 1 到 10 这些数字当作字符串拼接起来。对了，在 Lua 中，我们使用两个点号来表示字符串的相加：</p><pre><code>$ resty -e 'local s  = &quot;&quot;
 for i = 1, 10 do
     s = s .. tostring(i)
 end
 print(s)'
</code></pre><p>这里我们循环了 10 次，但只有最后一次是我们想要的，而中间新建的 9 个字符串都是无用的。它们不仅占用了额外的空间，也消耗了不必要的 CPU 运算。</p><p>当然，在后面的性能优化章节，我们会有对应的方法来解决它。</p><p>另外，在 Lua 中，你有三种方式可以表达一个字符串：单引号、双引号，以及长括号（<code>[[]]</code>）。前面两种都比较好理解，别的语言一般也这么用，那么长括号有什么用处呢？</p><p>我们看一个具体的示例：</p><pre><code>$ resty -e 'print([[string has \n and \r]])'
 string has \n and \r
</code></pre><p>你可以看到，长括号中的字符串不会做任何的转义处理。</p><p>你也许会问另外一个问题：如果上面那段字符串中包括了长括号本身，又该怎么处理呢？答案很简单，就是在长括号中间增加一个或者多个 <code>=</code> 符号：</p><pre><code>$ resty -e 'print([=[ string has a [[]]. ]=])'
  string has a [[]].
</code></pre><h3>布尔值</h3><p>这个很简单，true 和 false。但在 Lua 中，只有 nil 和 false 为假，其他都为真，包括 0 和空字符串也为真。我们可以用下面的代码印证一下：</p><pre><code>$ resty -e 'local a = 0
 if a then
   print(&quot;true&quot;)
 end
 a = &quot;&quot;
 if a then
   print(&quot;true&quot;)
 end'
</code></pre><p>这种判断方式和很多常见的开发语言并不一致，所以，为了避免在这种问题上出错，你可以显式地写明比较的对象，比如下面这样：</p><pre><code>$ resty -e 'local a = 0
 if a == false then
   print(&quot;true&quot;)
 end
 '

</code></pre><h3>数字</h3><p>Lua 的 number 类型，是用双精度浮点数来实现的。值得一提的是，LuaJIT 支持 <code>dual-number</code>（双数）模式，也就是说， LuaJIT 会根据上下文来用整型来存储整数，而用双精度浮点数来存放浮点数。</p><p>此外，LuaJIT 还支持<code>长长整型</code>的大整数，比如下面的例子：</p><pre><code>$ resty -e 'print(9223372036854775807LL - 1)'
9223372036854775806LL
</code></pre><h3>函数</h3><p>函数在 Lua 中是一等公民，你可以把函数存放在一个变量中，也可以当作另外一个函数的入参和出参。</p><p>比如，下面两个函数的声明是完全等价的：</p><pre><code>function foo()
 end
</code></pre><p>和</p><pre><code>foo = function ()
 end
</code></pre><h3>table</h3><p>table 是 Lua 中唯一的数据结构，自然非常重要，所以后面我会用专门的章节来介绍它。我们可以先来看一个简单的示例代码：</p><pre><code>$ resty -e 'local color = {first = &quot;red&quot;}
print(color[&quot;first&quot;])'
 red
</code></pre><h3>空值</h3><p>在 Lua 中，空值就是 nil。如果你定义了一个变量，但没有赋值，它的默认值就是 nil：</p><pre><code>$ resty -e 'local a
 print(type(a))'
 nil
</code></pre><p>当你真正进入 OpenResty 体系中后，会发现很多种空值，比如 <code>ngx.null</code> 等等，我们后面再细聊。</p><p>Lua的数据类型，我主要就介绍这么多，先给你打个基础。一些需要重点掌握的内容，后面的文章中我们都会继续学习。在练习、使用中学习，永远是吸收新知识最便捷的方式。</p><h2>常用标准库</h2><p>很多时候，我们学习一门语言，其实就是在学习它的标准库。</p><p>Lua 比较小巧，内置的标准库并不多。而且，在 OpenResty 的环境中，Lua 标准库的优先级是很低的。对于同一个功能，我更推荐你优先使用 OpenResty 的 API 来解决，然后是 LuaJIT 的库函数，最后才是标准 Lua 的函数。</p><p><code>OpenResty的API &gt; LuaJIT的库函数 &gt; 标准Lua的函数</code>，这个优先级后面会被反复提及，它不仅关系到是否好用这一点，更会对性能产生非常大的影响。</p><p>不过，尽管如此，在实际的项目开发中，我们还是不可避免会用到一些 Lua 库。这里，我挑选了几个比较常用的标准库做下介绍，如果你想要了解更多内容，可以查阅 Lua 的官方文档。</p><h3>string 库</h3><p>字符串操作是我们最常用到的，也是坑最多的地方。有一个简单的原则，那就是如果涉及到正则表达式的，请一定要使用 OpenResty 提供的 <code>ngx.re.*</code> 来解决，不要用 Lua 的 <code>string.*</code> 处理。这是因为，Lua 的正则独树一帜，不符合 PCRE 的规范，我相信绝大部分工程师是玩不转的。</p><p>其中 <code>string.byte(s [, i [, j ]])</code>，是比较常用到的一个 string 库函数，它返回字符 s[i]、s[i + 1]、s[i + 2]、······、s[j] 所对应的 ASCII 码。i 的默认值为 1，即第一个字节，j 的默认值为 i。</p><p>下面我们来看一段示例代码：</p><pre><code>$ resty -e 'print(string.byte(&quot;abc&quot;, 1, 3))
 print(string.byte(&quot;abc&quot;, 3)) -- 缺少第三个参数，第三个参数默认与第二个相同，此时为 3
 print(string.byte(&quot;abc&quot;))    -- 缺少第二个和第三个参数，此时这两个参数都默认为 1
 '
</code></pre><p>它的输出为：</p><pre><code> 979899
 99
 97
</code></pre><h3>table 库</h3><p>在 OpenResty 的上下文中，对于Lua 自带的 table 库，除了 <code>table.concat</code> 、<code>table.sort</code> 等少数几个函数，大部分我都不推荐使用。至于它们的细节，我们留在 LuaJIT 章节中专门来讲。</p><p>这里我简单提一下<code>table.concat</code> 。<code>table.concat</code>一般用在字符串拼接的场景下，比如下面这个例子。它可以避免生成很多无用的字符串。</p><pre><code>$ resty -e 'local a = {&quot;A&quot;, &quot;b&quot;, &quot;C&quot;}
 print(table.concat(a))'
</code></pre><h3>math 库</h3><p>Lua math 库由一组标准的数学函数构成。数学库的引入，既丰富了 Lua 编程语言的功能，同时也方便了程序的编写。</p><p>在 OpenResty 的实际项目中，我们很少用 Lua 去做数学方面的运算，不过其中和随机数相关的 <code>math.random()</code> 和 <code>math.randomseed()</code> 两个函数，倒是比较常用，比如下面的这段代码，它可以在指定的范围内，随机地生成两个数字。</p><pre><code>$ resty -e 'math.randomseed (os.time()) 
print(math.random())
 print(math.random(100))'
</code></pre><h2>虚变量</h2><p>了解了这些常见的标准库，接下来，我们再来学习一个新的概念——虚变量。</p><p>设想这么一个场景，当一个函数返回多个值的时候，有些返回值我们并不需要，这时候，应该怎么接收这些值呢？</p><p>不知道你是怎么看待这件事的，起码对我来说，要想法设法给这些用不到的变量，去赋予有意义的名字，着实是一件很折磨人的事情。</p><p>还好， Lua 中可以完美地解决这一点。Lua 提供了一个虚变量（dummy variable）的概念， 按照惯例以一个下划线来命名，用来表示丢弃不需要的数值，仅仅起到占位的作用。</p><p>下面我们以 <code>string.find</code> 这个标准库函数为例，来看虚变量的用法。这个标准库函数会返回两个值，分别代表开始和结束的下标。</p><p>如果我们只需要获取开始的下标，那么很简单，只声明一个变量来接收 <code>string.find</code> 的返回值即可：</p><pre><code>$ resty -e 'local start = string.find(&quot;hello&quot;, &quot;he&quot;)
 print(start)'
 1
</code></pre><p>但如果你只想获取结束的下标，那就必须使用虚变量了：</p><pre><code>$ resty -e 'local  _, end_pos = string.find(&quot;hello&quot;, &quot;he&quot;)
 print(end_pos)'
 2
</code></pre><p>除了在返回值里使用，虚变量还经常用于循环中，比如下面这个例子：</p><pre><code>$ resty -e 'for _, v in ipairs({4,5,6}) do
     print(v)
 end'
 4
 5
 6
</code></pre><p>而当有多个返回值需要忽略时，你可以重复使用同一个虚变量。这里我就不举例子了，你可以试着自己写一个这样的示例代码吗？欢迎你把代码贴在留言区里和我分享、交流。</p><h2>写在最后</h2><p>今天，我们一起快速地学习了标准 Lua 的数据结构和语法，相信你对这门简单精巧的语言已经有了初步的了解。下节课，我会带你了解  Lua 和 LuaJIT 的关系，LuaJIT 更是 OpenResty 中的重头戏，值得我们深入挖掘。</p><p>最后，我想再为你留下一道思考题。</p><p>还记得这节课讲math库时，学过的这段代码吗？它可以在指定范围内，随机生成两个数字。</p><pre><code>$ resty -e 'math.randomseed (os.time()) 
print(math.random())
 print(math.random(100))'
</code></pre><p>不过，你可能注意到了，这段代码是用当前时间戳作为种子的，那么这种方法是否有问题呢？又该如何生成好的种子呢？要知道，很多时候我们生成的随机数其实并不随机，并且有很大的安全隐患。</p><p>欢迎在留言区来说说你的看法，也欢迎你把这篇文章转发给你的同事、朋友。我们一起交流、一起进步。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/11/3e/925aa996.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>HelloBug</span>
  </div>
  <div class="_2_QraFYR_0">网上有种设置随机数种子的方法：<br>math.randomseed(tostring(os.time()):reverse():sub(1, 6))<br>即将时间值转换为字符串，然后将字符串倒序，然后取其前六位作为种子。之所以这样做的原因是因为当时间变化很小的时候，产生随机数的序列很相似。所以通过这种方法使得即使时间变化很小，由于reverse操作，时间的高位变成低位，低位变成高位，随机数种子的值变化会很大。<br><br>关于这种做法有以下两个问题：<br>1.当前执行os.time()打印的时间是1560251897，总共十位。认为函数中使用sub(1, 6)这里取前6位的中的6并不是一个固定的值，而且并没有什么意义，直接使用math.randomseed(tostring(os.time()):reverse())就能达到想要的效果。难道有其他我没有想到的原因？<br><br>2.这样的做法并不能阻止在同一秒内产生相同的随机数序列，如执行以下代码：<br>math.randomseed(tostring(os.time()):reverse():sub(1, 6)) <br>print(math.random()) <br>print(math.random())<br>math.randomseed(tostring(os.time()):reverse():sub(1, 6)) <br>print(math.random()) <br>print(math.random())<br>输出结果是：<br>0.49256881466135<br>0.0046852543018758<br>0.49256881466135<br>0.0046852543018758<br><br>另一种说法是对计算机的一些操作，如键盘、鼠标操作，会产生一些随机数，这些随机数叫熵。用户可以通过读取&#47;dev&#47;random和&#47;dev&#47;urandom文件来获取这些随机数。只不过读取&#47;dev&#47;random时，如果文件里的熵不足时会阻塞。读取&#47;dev&#47;urandom时，不会阻塞，但不能保证是合适的数据（熵不足时怎么处理未测试）。关于熵的还有其他相关知识，如通过操作鼠标、键盘等可以产生熵，通过cat &#47;proc&#47;sys&#47;kernel&#47;random&#47;entropy_avail操作可以查看有多少熵可以用等。<br>这样的话，通过读取&#47;dev&#47;urandom设置随机数种子，是一种方法，但觉得这种文件读取操作，效率太低。<br><br>另外看留言里有人说通过系统调用，利用芯片电磁噪声来生成随机数。没有搜到是哪个系统调用。<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 很详细了。如果有加密的需求，从 &#47;dev&#47;random 和 &#47;dev&#47;urandom 读取会更安全，毕竟只是 init 的时候读取一次。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-11 19:45:10</div>
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
  <div class="_2_QraFYR_0">返回三个变量，前两个变量重复使用同一个虚变量的例子：<br>resty -e &#39;local function sum(a, b) return a, b, a + b end local _, _, result = sum(1, 2) print(result)&#39;<br><br>os.time返回当前时间的秒数，如果在同一秒内设置当前时间秒数为种子，然后执行随机数生成函数，产生的随机数序列是一样的。如：<br>resty -e &#39;math.randomseed(os.time()) print(math.random()) print(math.random()) math.randomseed(os.time()) print(math.random()) print(math.random())&#39;<br>输出结果是：<br>0.71251659032569<br>0.36755092546457<br>0.71251659032569<br>0.36755092546457<br>可以看到两次产生的随机数序列相同。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-11 19:11:19</div>
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
  <div class="_2_QraFYR_0">明白了， os.time 是秒级别的，如果短时间运行多次会出现相同的随机数</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，没错</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-10 08:24:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/fe/4b/87219faf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>叫我图图就可以了</span>
  </div>
  <div class="_2_QraFYR_0">种子会出现相同的,简单的做法可以用GUID或者UUID之类的做种子吧.</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: uuid 本身也是先有种子，然后通过随机数生成的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-10 00:35:41</div>
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
  <div class="_2_QraFYR_0">请问一下老师，如果我在init_by_lua 中引入某个模块，不加local，作为全局变量，是不是说这个模块就可以在以后rewrite,access 等阶段直接拿来使用？这样做相比较于在各个阶段自己引入模块，是否减少了require的次数，提高了性能？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不会提高性能，模块在单个 worker 中只会加载一次，和是否加了 local 无关。设置为全局变量，很容易出错，比如重名什么的。在OpenResty 中建议所有变量都 local。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-11 10:38:20</div>
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
  <div class="_2_QraFYR_0">机器名+进程ID+线程ID+毫秒时间戳做种子</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-11 08:53:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/e9/0b/1171ac71.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>WL</span>
  </div>
  <div class="_2_QraFYR_0">请问一下老师我这边luajit -v 和  opm -v 都提示命令找不到, openresty的版本是openresty&#47;1.15.8.1, 想问下这种请情况是啥原因, 应该咋解决</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-10 06:46:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/MlmSR4YXUfrNlZdMv7bv1ic64HaxxVKcVtaxjzhXCvNC4XByICCmYUTprhOESzIV8p59N6DnSJ7HywfvGr5nicgA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>mz</span>
  </div>
  <div class="_2_QraFYR_0">我怎么感觉 go 语言那个不需要的返回值的写法就是借鉴的 lua 虚变量</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-10 00:23:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/07/63/c54d40f4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Surjur</span>
  </div>
  <div class="_2_QraFYR_0">温老师，您好：我在看luajit和lua区别的时候有些疑问：<br>1. luajit编译生成的机器码是一直保存在内存里的吗？<br>2. 一般线上环境，提高性能，lua_code_cache是开启的，所以如果lua代码有改动，需要reload nginx，新的代码才能生效，那一旦reload了，之前编译好的机器码还存在吗？<br>3. luajit vm之所以相对lua vm高效，是因为一些热代码少了字节码转机器码的过程，因为已经有编译生成了机器码，直接执行机器码即可，是这样吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-22 15:59:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>nicknick</span>
  </div>
  <div class="_2_QraFYR_0">所谓的虚变量就是一个普通的变量，没有任何特殊之处，只不过是不用它的值而已，单独拿出来做一个概念是否有必要</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-29 10:26:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/30/5d/80/139d3f02.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>JoyZ</span>
  </div>
  <div class="_2_QraFYR_0">老师，请问下我通过openresty在lua代码里print(9223372036854775807LL - 1)这样为什么报错呢？<br>提示：<br>bad argument #1 to &#39;print&#39; (string, number, boolean, or nil expected, got cdata)<br>只有转tostring后这样print(tostring(9223372036854775807LL - 1))才能正确打印</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-01 17:50:41</div>
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
  <div class="_2_QraFYR_0">刚开始学lua脚本的时候，很不习惯，一个函数能同时返回多个变量，等后来发现挺方便的，易于扩展</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-01 18:25:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/4f/1e/cb8ddbe9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>红鲤鱼与绿鲤鱼与驴baci</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，在 Lua 中，有三种方式可以表达一个字符串：单引号、双引号，以及长括号（[[]]），那单引号和双引号之前有什么区别呢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-16 20:29:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/59/8c/ba81a832.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>刀斧手何在</span>
  </div>
  <div class="_2_QraFYR_0">老师，有个问题请教一下。我的代码大概是这样的。<br>```<br>ngx.req.read_body() -- 解析 body 参数之前一定要先读取 body<br>local arg = ngx.req.get_post_args()<br>--ngx.say(type(arg))<br>ngx.print(arg)<br>```<br>然后ngx.say的时候arg是table类型。ngx.print我看文档是说支持打印table的https:&#47;&#47;github.com&#47;openresty&#47;lua-nginx-module#ngxprint。但是我 ngx.print(arg)就会报错。<br>error-log是这样的。<br>```<br>2019&#47;10&#47;06 10:57:27 [error] 42634#1017147: *50 lua entry thread aborted: runtime error: &#47;Users&#47;fang&#47;Code&#47;lua_test&#47;lua_script&#47;params.lua:9: bad argument #1 to &#39;print&#39; (non-array table found)<br>stack traceback:<br>coroutine 0:<br>	[C]: in function &#39;print&#39;<br>	&#47;Users&#47;fang&#47;Code&#47;lua_test&#47;lua_script&#47;params.lua:9: in main chunk, client: 127.0.0.1, server: localhost, request: &quot;POST &#47;print_params HTTP&#47;1.1&quot;, host: &quot;localhost&quot;<br>```<br>提示参数出错 no array or table found 。 那么这个 arg 到底是不是一个table</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-06 11:08:39</div>
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
  <div class="_2_QraFYR_0">思考题：使用系统时间戳作为种子 在分布式环境中机器的时间不一致 导致可能产生的种子是一样的 进而可能导致随机数不再随机。 解决的防止可以使用统一的时间机器 而非 各自机器的时间。 这个问题跟生产唯一性分布式ID有点关联</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-13 10:29:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/d1/7a/ea521259.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>夏天的风</span>
  </div>
  <div class="_2_QraFYR_0">请问一下windows上怎么运行resty，我看用luajit a.lua是可以成功的，用luajit -e &#39;print(&quot;hello&quot;)&#39; 会报错，用resty -e &#39;print(&quot;hello&quot;)&#39; 也会报同样的错误。<br>D:\lua&gt;luajit -e &#39;print(&quot;hello&quot;)&#39;<br>luajit: (command line):1: unexpected symbol near &#39;&#39;print(hello)&#39;&#39;<br> <br>D:\lua&gt;luajit a.lua<br>hello world<br><br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 推荐在 Linux 环境运行专栏的代码，Windows 上OpenResty 自己也是功能受限的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-03 19:34:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/da/36/ac0ff6a7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>wusiration</span>
  </div>
  <div class="_2_QraFYR_0">秒级的os.time在短时间调用中，random函数会以相同的随机种子产生相同的随机数</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-18 23:34:58</div>
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
  <div class="_2_QraFYR_0">localhost: &#47;usr&#47;local&#47;opt&#47;openresty&gt; ll &#47;usr&#47;local&#47;Cellar&#47;openresty&#47;1.15.8.1&#47;luajit&#47;bin&#47;luajit<br>lrwxr-xr-x  1 yuesf  admin  18  6 12 07:04 &#47;usr&#47;local&#47;Cellar&#47;openresty&#47;1.15.8.1&#47;luajit&#47;bin&#47;luajit -&gt; luajit-2.1.0-beta3<br>localhost: &#47;usr&#47;local&#47;opt&#47;openresty&gt; luajit -v<br>-bash: luajit: command not found<br><br>luajit 是不是存在， 为什么 执行luajit 命令说不存在<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-16 17:14:35</div>
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
  <div class="_2_QraFYR_0">为什么我这里没有luajit？<br>localhost: ~&gt; whicht luajit<br>-bash: whicht: command not found<br>localhost: ~&gt; which luajit<br>localhost: ~&gt; luajit -v<br>-bash: luajit: command not found<br>localhost: ~&gt;<br><br>是否我安装有问题<br>localhost: &#47;usr&#47;local&#47;opt&#47;openresty&gt; pwd<br>&#47;usr&#47;local&#47;opt&#47;openresty<br>localhost: &#47;usr&#47;local&#47;opt&#47;openresty&gt; openresty -v<br>nginx version: openresty&#47;1.15.8.1<br>localhost: &#47;usr&#47;local&#47;opt&#47;openresty&gt;</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: luajit 在 &#47;usr&#47;local&#47;openresty&#47;luajit&#47; 目录中，避免污染系统的 luajit</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-16 17:11:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/59/f6/ed66d1c1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>chengzise</span>
  </div>
  <div class="_2_QraFYR_0">1. 用当前时间戳作为种子的，是通用做法没有什么大问题。这种方式利用的是随机数生成函数，随机数重复周期很长而已，不能算是完全随机。 大多数使用环境没什么问题。<br>2. 在要求严格随机数环境中，例如：加解密算法。可以使用系统提供的随机数接口，这种方式利用的是芯片电磁噪声来生成随机数。由于需要经过系统调用，理论上速度没有第一种方式快。<br>可以根据需要选择哪种方式。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 就是要有足够的熵</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-10 19:23:44</div>
  </div>
</div>
</div>
</li>
</ul>