<audio title="11 _ 剖析Lua唯一的数据结构table和metatable特性" src="https://static001.geekbang.org/resource/audio/37/b7/37b91fa264c2af0acabd16235e1472b7.mp3" controls="controls"></audio> 
<p>你好，我是温铭。今天我们一起学习下LuaJIT 中唯一的数据结构：<code>table</code>。</p><p>和其他具有丰富数据结构的脚本语言不同，LuaJIT 中只有 <code>table</code> 这一个数据结构，并没有区分开数组、哈希、集合等概念，而是揉在了一起。让我们先温习下之前提到过的一个例子：</p><pre><code>local color = {first = &quot;red&quot;, &quot;blue&quot;, third = &quot;green&quot;, &quot;yellow&quot;}
print(color[&quot;first&quot;])                 --&gt; output: red
print(color[1])                         --&gt; output: blue
print(color[&quot;third&quot;])                --&gt; output: green
print(color[2])                         --&gt; output: yellow
print(color[3])                         --&gt; output: nil
</code></pre><p>这个例子中， <code>color</code> 这个 table 包含了数组和哈希，并且可以互不干扰地进行访问。比如，你可以用 <code>ipairs</code> 函数，只遍历数组部分的内容：</p><pre><code>$ resty -e 'local color = {first = &quot;red&quot;, &quot;blue&quot;, third = &quot;green&quot;, &quot;yellow&quot;}
for k, v in ipairs(color) do
     print(k)
end
'
</code></pre><p><code>table</code> 的操作是如此重要，以至于 LuaJIT 对标准 Lua 5.1 的 table 库做了扩展，而 OpenResty 又对 LuaJIT 的 table 库做了更进一步的扩展。下面，我们就一起来分别看下这些库函数。</p><h2>table 库函数</h2><p>先来看标准table 库函数。Lua 5.1 中自带的 table 库函数并不多，我们可以大概浏览一遍。</p><h3><code>table.getn</code> 获取元素个数</h3><p>我们在 <code>标准 Lua 和 LuaJIT</code> 章节中曾经提到过，想正确地获取到 table 所有元素的个数，在 LuaJIT 中是一个老大难问题。</p><p>对于序列，你用<code>table.getn</code> 或者一元操作符 <code>#</code> ，就可以正确返回元素的个数。比如下面这个例子，就会返回我们预期中的 3。</p><!-- [[[read_end]]] --><pre><code>$ resty -e 'local t = { 1, 2, 3 }
print(table.getn(t)) '
</code></pre><p>而对于不是序列的 table，就无法返回正确的值。比如第二个例子，返回的就是 1。</p><pre><code>$ resty -e 'local t = { 1, a = 2 }
print(#t) '
</code></pre><p>不过，幸运的是，这种难以理解的函数，已经被 LuaJIT 的扩展替代，后面我们会提到。所以在 OpenResty 的环境下，除非你明确知道，你正在获取序列的长度，否则请不要使用函数 <code>table.getn</code> 和一元操作符 <code>#</code> 。</p><p>另外，<code>table.getn</code> 和一元操作符 <code>#</code> 并不是 O(1) 的时间复杂度，而是 O(n)，这也是尽量避免使用它们的另外一个理由。</p><h3><code>table.remove</code> 删除指定元素</h3><p>第二个我们来看<code>table.remove</code> 函数，它的作用是在 table 中根据下标来删除元素，也就是说只能删除 table 中数组部分的元素。我们还是来看<code>color</code>的例子：</p><pre><code>$ resty -e 'local color = {first = &quot;red&quot;, &quot;blue&quot;, third = &quot;green&quot;, &quot;yellow&quot;}
  table.remove(color, 1)
  for k, v in pairs(color) do
      print(v)
  end'
</code></pre><p>这段代码会把下标为 1 的 <code>blue</code> 删除掉。你可能会问，那该如何删除 table 中的哈希部分呢？也很简单，把 key 对应的 value 设置为 <code>nil</code> 即可。这样，<code>color</code>这个例子中，<code>third</code> 对应的<code>green</code>就被删除了。</p><pre><code>$ resty -e 'local color = {first = &quot;red&quot;, &quot;blue&quot;, third = &quot;green&quot;, &quot;yellow&quot;}
  color.third = nil
  for k, v in pairs(color) do
      print(v)
  end'
</code></pre><h3><code>table.concat</code> 元素拼接函数</h3><p>第三个我们来看<code>table.concat</code> 元素拼接函数。它可以按照下标，把 table 中的元素拼接起来。既然这里又是根据下标来操作的，那么显然还是针对 table 的数组部分。同样还是<code>color</code>这个例子：</p><pre><code>$ resty -e 'local color = {first = &quot;red&quot;, &quot;blue&quot;, third = &quot;green&quot;, &quot;yellow&quot;}
print(table.concat(color, &quot;, &quot;))'
</code></pre><p>使用<code>table.concat</code>函数后，它输出的是 <code>blue, yellow</code>，哈希的部分被跳过了。</p><p>另外，这个函数还可以指定下标的起始位置来做拼接，比如下面这样的写法：</p><pre><code>$ resty -e 'local color = {first = &quot;red&quot;, &quot;blue&quot;, third = &quot;green&quot;, &quot;yellow&quot;, &quot;orange&quot;}
print(table.concat(color, &quot;, &quot;, 2, 3))'
</code></pre><p>这次输出是 <code>yellow, orange</code>，跳过了 <code>blue</code>。</p><p>你可能觉得这些操作还挺简单的，不过，我要说的是，函数不可貌相，海水不可。千万不要小看这个看上去没有太大用处的函数，在做性能优化时，它却会有意想不到的作用，也是我们后面性能优化章节中的主角之一。</p><h3><code>table.insert</code> 插入一个元素</h3><p>最后我们来看<code>table.insert</code> 函数。它可以下标插入一个新的元素，自然，影响的还是 table 的数组部分。还是用<code>color</code>例子来说明：</p><pre><code>$ resty -e 'local color = {first = &quot;red&quot;, &quot;blue&quot;, third = &quot;green&quot;, &quot;yellow&quot;}
table.insert(color, 1,  &quot;orange&quot;)
print(color[1])
'
</code></pre><p>你可以看到， color 的第一个元素变为了 orange。当然，你也可以不指定下标，这样就会默认插入队尾。</p><p>这里我必须说明的是，<code>table.insert</code> 虽然是一个很常见的操作，但性能并不乐观。如果你不是根据指定下标来插入元素，那么每次都需要调用 LuaJIT 的 <code>lj_tab_len</code> 来获取数组的长度，以便插入队尾。正如我们在 <code>table.getn</code> 中提到的，获取 table 长度的时间复杂度为 O(n) 。</p><p>所以，对于<code>table.insert</code> 操作，我们应该尽量避免在热代码中使用，比如：</p><pre><code>local t = {}
for i = 1, 10000 do
     table.insert(t, i)
end
</code></pre><h2>LuaJIT 的 table 扩展函数</h2><p>接下来我们来看LuaJIT 的 table 扩展函数。LuaJIT 在标准 Lua 的基础上，扩展了两个很有用的 table 函数，分别用来新建和清空一个 table，下面我具体来介绍一下。</p><h3><code>table.new(narray, nhash)</code> 新建 table</h3><p>第一个是<code>table.new(narray, nhash)</code> 函数。这个函数，会预先分配好指定的数组和哈希的空间大小，而不是在插入元素时自增长，这也是它的两个参数 <code>narray</code> 和 <code>nhash</code> 的含义。自增长是一个代价比较高的操作，会涉及到空间分配、<code>resize</code> 和 <code>rehash</code> 等，我们应该尽量避免。</p><p>这里注意，<code>table.new</code> 的文档并没有出现在 LuaJIT 的官网，而是深藏在 GitHub 项目的<a href="https://github.com/openresty/luajit2/blob/v2.1-agentzh/doc/extensions.html">扩展文档</a>中，即使你用谷歌也难觅其踪迹，所以知道的工程师并不多。</p><p>下面是一个简单的例子，我来带你看下它该怎么用。首先要说明，这个函数是扩展出来的，所以在使用它之前，你需要先 <code>require</code> 一下：</p><pre><code>local new_tab = require &quot;table.new&quot;
local t = new_tab(100, 0)
for i = 1, 100 do
   t[i] = i
end
</code></pre><p>你可以看到，这段代码新建了一个 table，里面包含 100 个数组元素和 0 个哈希元素。当然，你也可以根据实际需要，新建一个同时包含 100 个数组元素和 50 个 哈希元素的 table，这都是合法的：</p><pre><code>local t = new_tab(100, 50)
</code></pre><p>另外，超出预设的空间大小，也可以正常使用，只不过性能会退化，也就失去了使用 <code>table.new</code> 的意义。</p><p>比如下面这个例子，我们预设大小为 100，而实际上却使用了 200：</p><pre><code>local new_tab = require &quot;table.new&quot;
local t = new_tab(100, 0)
for i = 1, 200 do
   t[i] = i
end
</code></pre><p>所以，你需要根据实际场景，来预设好 <code>table.new</code> 中数组和哈希空间的大小，这样才能在性能和内存占用上找到一个平衡点。</p><h3><code>table.clear()</code> 清空 table</h3><p>第二个我们来看清空函数<code>table.clear()</code> 。它用来清空某个 table 里的所有数据，但并不会释放数组和哈希部分占用的内存。所以，它在循环利用 Lua table 时非常有用，可以避免反复创建和销毁 table 的开销。</p><pre><code>$ resty -e 'local clear_tab =require &quot;table.clear&quot;
local color = {first = &quot;red&quot;, &quot;blue&quot;, third = &quot;green&quot;, &quot;yellow&quot;}
clear_tab(color)
for k, v in pairs(color) do
     print(k)
end'
</code></pre><p>不过，事实上，能使用这个函数的场景并不算多，大多数情况下，我们还是应该把这个任务交给 LuaJIT GC 去完成。</p><h2>OpenResty 的 table 扩展函数</h2><p>开头我提到过，OpenResty 自己维护的 LuaJIT 分支，也对 table 做了扩展，它<a href="https://github.com/openresty/luajit2/#new-api">新增了几个 API</a>：<code>table.isempty</code>、<code>table.isarray</code>、 <code>table.nkeys</code> 和 <code>table.clone</code>。</p><p>需要注意的是，在使用这几个新增的 API 前，请记住检查你使用的 OpenResty 的版本，这些API 大都只能在 OpenResty 1.15.8.1 之后的版本中使用。这是因为， OpenResty 在 1.15.8.1 版本之前，已经有一年左右没有发布新版本了，而这些 API 是在这个发布间隔中新增的。</p><p>文章中我已经附上了链接，这里我就只用 <code>table.nkeys</code> 来举例说明下，其他的三个 API 从命名上来说都非常容易理解，你自己翻阅 GitHub 上的文档就可以明白了。不得不说，OpenResty 的文档质量非常高，其中包含了代码示例、能否被 JIT、需要注意的事项等，比起 Lua 和 LuaJIT 的文档，着实高了好几个数量级。</p><p>好的，回到<code>table.nkeys</code>函数上，它的命名可能会让你迷惑，不过，它实际上是获取 table 长度的函数，返回的是 table 的元素个数，包括数组和哈希部分的元素。因此，我们可以用它来替代 <code>table.getn</code>，比如下面这样来用：</p><pre><code>local nkeys = require &quot;table.nkeys&quot;

print(nkeys({}))  -- 0
print(nkeys({ &quot;a&quot;, nil, &quot;b&quot; }))  -- 2
print(nkeys({ dog = 3, cat = 4, bird = nil }))  -- 2
print(nkeys({ &quot;a&quot;, dog = 3, cat = 4 }))  -- 3
</code></pre><h2>元表</h2><p>讲完了table函数，我们再来看下由 <code>table</code> 引申出来的 <code>元表</code>（metatable）。元表是 Lua 中独有的概念，在实际项目中的使用非常广泛。不夸张地说，在几乎所有的 <code>lua-resty-*</code> 库中，你都能看到它的身影。</p><p>元表的表现行为类似于操作符重载，比如我们可以重载 <code>__add</code>，来计算两个 Lua 数组的并集；或者重载 <code>__tostring</code>，来定义转换为字符串的函数。</p><p>而Lua 提供了两个处理元表的函数：</p><ul>
<li>第一个是<code>setmetatable(table, metatable)</code>, 用于为一个 table 设置元表；</li>
<li>第二个是<code>getmetatable(table)</code>，用于获取 table 的元表。</li>
</ul><p>介绍了这么半天，你可能更关心它的作用，我们接着就来看下元表具体有什么用处。下面是一段真实项目里的代码：</p><pre><code>$ resty -e ' local version = {
  major = 1,
  minor = 1,
  patch = 1
  }
version = setmetatable(version, {
    __tostring = function(t)
      return string.format(&quot;%d.%d.%d&quot;, t.major, t.minor, t.patch)
    end
  })
  print(tostring(version))
'
</code></pre><p>我们首先定义了一个 名为 <code>version</code>的table ，你可以看到，这段代码的目的，是想把 <code>version</code> 中的版本号打印出来。但是，我们并不能直接打印 <code>version</code>，你可以试着操作一下，就会发现，直接打印的话，只会输出这个 table 的地址。</p><pre><code>print(tostring(version))
</code></pre><p>所以，我们需要自定义这个 table 的字符串转换函数，也就是 <code>__tostring</code>，到这一步也就是元表的用武之地了。我们用 <code>setmetatable</code> ，重新设置 <code>version</code> 这个 table 的 <code>__tostring</code> 方法，就可以打印出版本号: 1.1.1。</p><p>其实，除了 <code>__tostring</code> 之外，在实际项目中，我们还经常重载元表中的以下两个元方法（metamethod）。</p><p><strong>其中一个是<code>__index</code></strong>。我们在 table 中查找一个元素时，首先会直接从 table 中查询，如果没有找到，就继续到元表的 <code>__index</code> 中查询。</p><p>比如下面这个例子，我们把 <code>patch</code> 从 <code>version</code> 这个 table 中去掉：</p><pre><code>$ resty -e ' local version = {
  major = 1,
  minor = 1
  }
version = setmetatable(version, {
     __index = function(t, key)
         if key == &quot;patch&quot; then
             return 2
         end
     end,
     __tostring = function(t)
      return string.format(&quot;%d.%d.%d&quot;, t.major, t.minor, t.patch)
    end
  })
  print(tostring(version))
'
</code></pre><p>这样的话，<code>t.patch</code> 其实获取不到值，那么就会走到 <code>__index</code> 这个函数中，结果就会打印出 1.1.2。</p><p>事实上，<code>__index</code> 不仅可以是一个函数，也可以是一个 table。你试着运行下面这段代码，就会看到，它们实现的效果是一样的。</p><pre><code>$ resty -e ' local version = {
  major = 1,
  minor = 1
  }
version = setmetatable(version, {
     __index = {patch = 2},
     __tostring = function(t)
      return string.format(&quot;%d.%d.%d&quot;, t.major, t.minor, t.patch)
    end
  })
  print(tostring(version))
'
</code></pre><p><strong>另一个元方法则是<code>__call</code></strong>。它类似于仿函数，可以让 table 被调用。</p><p>我们还是基于上面打印版本号的代码来做修改，看看如何调用一个 table：</p><pre><code>$ resty -e '
local version = {
  major = 1,
  minor = 1,
  patch = 1
  }

local function print_version(t)
     print(string.format(&quot;%d.%d.%d&quot;, t.major, t.minor, t.patch))
end

version = setmetatable(version,
     {__call = print_version})

  version()
'
</code></pre><p>这段代码中，我们使用 <code>setmetatable</code>，给 <code>version</code> 这个 table 增加了元表，而里面的 <code>__call</code> 元方法指向了函数 <code>print_version</code> 。那么，如果我们尝试把 <code>version</code> 当作函数调用，这里就会执行函数 <code>print_version</code>。</p><p>而 <code>getmetatable</code> 是和 <code>setmetatable</code> 配对的操作，可以获取到已经设置的元表，比如下面这段代码：</p><pre><code>$ resty -e ' local version = {
  major = 1,
  minor = 1
  }
version = setmetatable(version, {
     __index = {patch = 2},
     __tostring = function(t)
      return string.format(&quot;%d.%d.%d&quot;, t.major, t.minor, t.patch)
    end
  })
  print(getmetatable(version).__index.patch)
'
</code></pre><p>自然，除了今天讲到的这三个元方法外，还有一些不经常使用的元方法，你可以在遇到的时候再去查阅<a href="http://lua-users.org/wiki/MetamethodsTutorial">文档</a>了解。</p><h2>面向对象</h2><p>最后我们来聊聊面向对象。你可能知道，Lua 并不是一个面向对象（Object Orientation）的语言，但我们可以使用 metatable 来实现 OO。</p><p>我们来看一个实际的例子。<a href="https://github.com/openresty/lua-resty-mysql/blob/master/lib/resty/mysql.lua">lua-resty-mysql</a>  是 OpenResty 官方的 MySQL 客户端，里面就使用元表<strong>模拟</strong>了类和类方法，它的使用方式如下所示：</p><pre><code>$ resty -e 'local mysql = require &quot;resty.mysql&quot; -- 先引用 lua-resty 库
local db, err = mysql:new() -- 新建一个类的实例
db:set_timeout(1000) -- 调用类的方法'
</code></pre><p>你可以直接用 <code>resty</code> 命令行来执行上述代码。这几行代码很好理解，唯一可能给你造成困扰的是：</p><p><strong>在调用类方法的时候，为什么是冒号而不是点号呢？</strong></p><p>其实，在这里冒号和点号都是可以的，<code>db:set_timeout(1000)</code> 和 <code>db.set_timeout(db, 1000)</code> 是完全等价的。冒号是 Lua 中的一个语法糖，可以省略掉函数的第一个参数 <code>self</code>。</p><p>众所周知，源码面前没有秘密，让我们来看看上述几行代码所对应的具体实现，以便你更好理解，如何用元表来模拟面向对象：</p><pre><code>local _M = { _VERSION = '0.21' } -- 使用 table 模拟类
local mt = { __index = _M } -- mt 即 metatable 的缩写，__index 指向类自身

-- 类的构造函数
function _M.new(self) 
     local sock, err = tcp()
     if not sock then
         return nil, err
     end
     return setmetatable({ sock = sock }, mt) -- 使用 table 和 metatable 模拟类的实例
end
 
-- 类的成员函数
 function _M.set_timeout(self, timeout) -- 使用 self 参数，获取要操作的类的实例
     local sock = self.sock
     if not sock then
        return nil, &quot;not initialized&quot;
     end

    return sock:settimeout(timeout)
end
</code></pre><p>你可以看到，<code>_M</code> 这个 table 模拟了一个类，初始化时，它只有 <code>_VERSION</code> 这一个成员变量，并在随后定义了 <code>_M.set_timeout</code> 等成员函数。在 <code>_M.new(self)</code> 这个构造函数中，我们返回了一个 table，这个 table 的元表就是 <code>mt</code>，而 <code>mt</code> 的 <code>__index</code> 元方法指向了 <code>_M</code>，这样，返回的这个 table 就模拟了类 <code>_M</code> 的实例。</p><h2>写在最后</h2><p>好的，到这里，今天的主要内容就结束了。事实上，table 和 metatable 会大量地用在 OpenResty 的 <code>lua-resty-*</code> 库以及基于 OpenResty 的开源项目中，我希望通过这节课的学习，可以让你更容易地读懂这些源代码。</p><p>自然，除了 table 外，Lua 中还有其他一些常用的函数，我们下节课再一起来学习。</p><p>最后，我想给你留一个思考题。为什么 <code>lua-resty-mysql</code> 库要模拟 OO 来做一层封装呢？欢迎在留言区一起讨论这个问题，也欢迎你把这篇文章分享给你的同事、朋友，我们一起交流，一起进步。</p><p></p>
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
  <div class="_2_QraFYR_0">lua-resty-mysql 模拟OO，个人理解是便于复用mysql连接，无需反复开启mysql连接；麻烦老师解惑一下</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-01 23:23:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/14/81/11cadd29.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>石头Rocky</span>
  </div>
  <div class="_2_QraFYR_0">setmetatable({ sock = sock }, mt) 构造函数返回的table是{ sock = sock }这个表。 <br>当调用方使用 set_timeout方法时，在表内找不到set_timeout方法，便会去__index所指的_M中去寻找。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-19 19:05:28</div>
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
  <div class="_2_QraFYR_0">直接 lua 运行 table.getn 会报这样的错的 lua: table.lua:7: attempt to call a nil value (field &#39;getn&#39;)<br><br>我查了一下， lua5.1 之后把这方法去掉了，那么为什么 luajit 和 openresty 不把这个方法废弃呢？还保留呢？ </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 因为 LuaJIT 是基于 Lua 5.1的语法来实现的，并没有跟随Lua 5.2 和 5.3 </p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-19 07:12:37</div>
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
  <div class="_2_QraFYR_0">localhost: ~&#47;geektime&#47;lua $ resty -e &#39;local color = {first = &quot;red&quot;, &quot;blue&quot;, third = &quot;green&quot;, &quot;yellow&quot;}<br>&gt;   color.first = nil<br>&gt;   for k, v in pairs(color) do<br>&gt;       print(v)<br>&gt;   end&#39;<br>blue<br>yellow<br>green<br>localhost: ~&#47;geektime&#47;lua $ resty -e &#39;local color = {first = &quot;red&quot;, &quot;blue&quot;, third = &quot;green&quot;, &quot;yellow&quot;}<br>&gt;   table.remove(color, 1)<br>&gt;   for k, v in pairs(color) do<br>&gt;       print(v)<br>&gt;   end&#39;<br>yellow<br>green<br>red<br><br>看结果为什么打印一个是正序，一个是倒序呢？ </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 字典的顺序是不保证的，大部分语言都是如何</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-22 13:38:46</div>
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
  <div class="_2_QraFYR_0">函数不可貌相，海水不可（斗量）<br>:)</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: ：）感谢 fix typo</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-19 16:58:53</div>
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
  <div class="_2_QraFYR_0">一个是代码复用和代码缓存原因，性能可以提高；<br>另外一个是可读性高，便于维护</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-19 16:56:51</div>
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
  <div class="_2_QraFYR_0">对于lua-resty-mysql 模拟 OO原因，理解为 mysql 的连接是需要保持的，需要复用的，当一个连接建立之后是需要复用的，不能频繁的断开连接，建立连接</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-19 07:51:51</div>
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
  <div class="_2_QraFYR_0">在luajit 扩展的table 函数中<br>local new_tab = require(&#39;table.new&#39;)<br>或者 require(&#39;table.clear&#39;)<br><br>用 luajit 去执行 都会报错找不动 moudule的错误的<br>luajit: table_luajit.lua:1: module &#39;table.new&#39; not found:<br>luajit 的版本 2.0.5</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 需要 LuaJIT 2.1 的版本才行, 文档在这里：https:&#47;&#47;github.com&#47;LuaJIT&#47;LuaJIT&#47;blob&#47;v2.1&#47;doc&#47;extensions.html#L218</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-19 07:33:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/d4/4e/5813df2f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Triple Z 💤</span>
  </div>
  <div class="_2_QraFYR_0">`table.nkeys` 和 `table.getn` &#47; `#` 有什么样的性能差异吗？我查了下 NYI 列表，貌似 `getn` 一样可以被 JIT。从实现上来看，貌似 `nkeys` 也是 O(n)。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-16 23:19:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/d4/4e/5813df2f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Triple Z 💤</span>
  </div>
  <div class="_2_QraFYR_0">模拟 OO 封装，有利于进行连接的表示，方便程序员能在上层构建连接池等管理连接的工具。毕竟 db 连接既不能做为全局唯一连接（模块变量），也不应该即用即放。即用即放的方法可能会导致 OpenResty QPS = DB QPS，大流量下非常容易因为新建连接数过高，造成数据库的瘫痪。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-16 22:43:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>无关风月</span>
  </div>
  <div class="_2_QraFYR_0">t.patch 其实获取不到值，那么就会走到 __index 这个函数中，结果就会打印出 1.1.2  怎么打印出2的，不是不存在patch的吗</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-29 14:38:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/3f/97/8d7a6460.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>卡卡</span>
  </div>
  <div class="_2_QraFYR_0">前几节课不是说op的下标是从1开始吗？怎么这节课的例子又从0开始了？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-09 09:46:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/77/ce/f7ccd5ac.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>michael</span>
  </div>
  <div class="_2_QraFYR_0">请教一个问题，在大多数编写面向对象的情况下 采用如下方式<br>local instance ={}<br>self.__index = self<br>setmetatable(instance, self)<br>但是当我想实现一个类似于php的__get魔术方法时如下调用就出现了问题<br>local instance ={data={&quot;apple&quot; =1}}<br>self.__index = function(t,k) <br>if instance.data[k] ~= nil &#47;&#47;如果在data中存在则返回相应value<br> return instance.data[k] &#47;&#47;类似get魔术方法<br>end<br>return self &#47;&#47;这一步出现大问题，后面实例化无法调用类方法<br>setmetatable(instance, self)<br><br>想知道问题的原因和解决方案，非常感谢，还有想加一个咱们Lua的交流群，希望能一起学习。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-08 16:04:42</div>
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
  <div class="_2_QraFYR_0">lua-resty-mysql模拟oo封装一层，我的理解是：1. 屏蔽底层通信细节，统一使用or特色的cosocket实现非阻塞高效IO 2. 接口简单易用，让开发人员把精力聚焦在业务逻辑上，而不是这些底层的通信细节上。这也是OR秉承的里面。 </div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-09 13:15:29</div>
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
  <div class="_2_QraFYR_0">是重载还是重写？java里面的重载是：指同一个类中的多个方法具有相同的名字,但这些方法具有不同的参数列表,即参数的数量或参数类型不能完全相同<br><br>方法重写是：存在子父类之间的,子类定义的方法与父类中的方法具有相同的方法名字,相同的参数表和相同的返回类型 </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 按照你的定义就是重写。Lua 没有类的概念，是模拟出来的，所以做不到重载这么高级。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-11 11:04:29</div>
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
  <div class="_2_QraFYR_0">温铭老师好， 如果table中有nil元素，这个元素是不是不占用内存空间？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-26 10:37:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/90/94/3949d451.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ArtEngr</span>
  </div>
  <div class="_2_QraFYR_0">“另外，超出预设的空间大小，也可以正常使用，只不过性能会退化” 很多时候与外界交互数据，无法准确知道table大小。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以设置大一些，留一些缓存</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-21 14:56:46</div>
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
  <div class="_2_QraFYR_0">老师，能否专门讲解一下lua面向对象是如何实现继承，抽象类的，我看了orange的源码发现不太容易理解，源码如下：<br>-- local BasePlugin = Object:extend()<br>function Object:extend()<br>  local cls = {}<br>  for k, v in pairs(self) do<br>    if k:find(&quot;__&quot;) == 1 then<br>      cls[k] = v<br>    end<br>  end<br>  cls.__index = cls<br>  cls.super = self<br>  setmetatable(cls, self)<br>  return cls<br>end<br><br><br>function Object:implement(...)<br>  for _, cls in pairs({...}) do<br>    for k, v in pairs(cls) do<br>      if self[k] == nil and type(v) == &quot;function&quot; then<br>        self[k] = v<br>      end<br>    end<br>  end<br>end<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不推荐用 Lua 的 table 来模拟实现继承等这种复杂的功能，就像你说的一样，可读性太差，一般人的脑子都转不过来。。。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-20 18:38:18</div>
  </div>
</div>
</div>
</li>
</ul>