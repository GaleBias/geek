<audio title="33 _ 性能提升10倍的秘诀：必须用好 table" src="https://static001.geekbang.org/resource/audio/a1/54/a15e8b77f80f9f0f0ce474e313dac854.mp3" controls="controls"></audio> 
<p>你好，我是温铭。</p><p>在 OpenResty 中，除了字符串经常出现性能问题外，table 也是性能的拦路虎。在之前的章节中，我们零零散散地介绍过 table 相关的函数，但并没有专门提到它对性能方面的提升。今天，我就带你专门来聊聊，table 操作对性能的影响。</p><p>不同于对字符串的熟悉，开发者对于 table 相关的性能优化知之甚少，这主要有两个方面的原因。</p><ul>
<li>其一，OpenResty 中使用的是 Lua ，是自己的 LuaJIT 分支，不是标准的 LuaJIT，也不是标准的 Lua。而大部分开发者并不知道它们之间的区别，倾向于使用标准 Lua 的 table 库来写 OpenResty 代码。</li>
<li>其二，在标准 LuaJIT 和 OpenResty 自己的 LuaJIT 分支中，table 操作相关的文档都藏得非常深，开发者很难找到；而且文档中也没有示例代码，需要开发者自己去开源项目中寻找示例。</li>
</ul><p>这就形成了比较高的认知壁垒，导致了两极分化的结果——资深的 OpenResty 开发者能够写出很高性能的代码，而刚入门的则会怀疑 OpenResty 的高性能是不是一个泡沫。当然，等你学习完这节课的内容，你就可以轻松地戳破这层窗户纸，让性能提升 10 倍不是梦。</p><p>在详细介绍 table 优化之前，我想先强调的一点是，table 相关的优化，有一个自己的简单原则：</p><!-- [[[read_end]]] --><p><strong>尽量复用，避免不必要的 table 创建。</strong></p><p>你先记住这一点，下面，我们就从 table 的创建、元素的插入、清空、循环使用等方面，分别来介绍相关的优化。</p><h2>预先生成数组</h2><p>第一步，自然是创建数组。在 Lua 中，我们创建数组的方式很简单：</p><pre><code>local t = {}
</code></pre><p>上面这行代码，就创建了一个空数组；当然，你也可以在创建的时候，就加上初始化的数据：</p><pre><code>local color = {first = &quot;red&quot;, &quot;blue&quot;, third = &quot;green&quot;, &quot;yellow&quot;}
</code></pre><p>不过，第二种写法对于性能的损失比较大，原因在于每次新增和删除数组元素的时候，都会涉及到数组的空间分配、<code>resize</code> 和 <code>rehash</code>。</p><p>那么应该如何优化呢？空间换时间，是一种常见的优化思路。既然这里的性能瓶颈是动态分配数组空间，那么优化的方向，就可以是预先生成一个指定大小的数组。这样做虽然可能会浪费一部分的内存空间，但多次的空间分配、<code>resize</code> 和 <code>rehash</code> 等动作，就可以合并为一次完成了，效率高了不少。</p><p>事实上，LuaJIT 中的 <code>table.new(narray, nhash)</code> 函数，就是因此而新增的。</p><p>这个函数，会预先分配好指定的数组和哈希的空间大小，而不是在插入元素时自增长，这也是它的两个参数 <code>narray</code> 和 <code>nhash</code> 的含义。</p><p>下面我们通过一个简单的例子，来看下具体的使用。因为这个函数是 LuaJIT 扩展出来的，所以，在使用它之前，我们需要先 <code>require</code> 一下：</p><pre><code>local new_tab = require &quot;table.new&quot;
 local t = new_tab(100, 0)
 for i = 1, 100 do
   t[i] = i
 end
</code></pre><p>另外，因为之前的 OpenResty 并没有完全绑定 LuaJIT，还支持标准 Lua，所以有些旧的代码会做这方面的兼容。如果没有找到 <code>table.new</code> 这个函数，就会模拟出来一个空的函数，来保证调用方的统一。</p><pre><code>local ok, new_tab = pcall(require, &quot;table.new&quot;)
  if not ok then
    new_tab = function (narr, nrec) return {} end
  end
</code></pre><h2>自己计算 table 下标</h2><p>有了 table 对象之后，下一步就是向它里面增加元素了。最直接的方法，就是调用 <code>table.insert</code> 这个函数来插入元素：</p><pre><code>local new_tab = require &quot;table.new&quot;
 local t = new_tab(100, 0)
 for i = 1, 100 do
   table.insert(t, i)
 end
</code></pre><p>或者是先获取当前数组的长度，通过下标的方式来插入元素：</p><pre><code>local new_tab = require &quot;table.new&quot;
 local t = new_tab(100, 0)
 for i = 1, 100 do
   t[#t + 1] = i
 end
</code></pre><p>不过，这两种方式都需要先计算数组的长度，然后再新增元素。显然，这个操作是 O(n) 的时间复杂度。就拿上面代码的例子来说，for 循环会计算 100 次数组的长度，这样下来性能自然不乐观，并且数组越大时，性能也会越低。</p><p>这一点又该如何解决呢？让我们看下 <code>lua-resty-redis</code> 这个官方的库是如何做的吧：</p><pre><code>local function _gen_req(args)
    local nargs = #args


    local req = new_tab(nargs * 5 + 1, 0)
    req[1] = &quot;*&quot; .. nargs .. &quot;\r\n&quot;
    local nbits = 2


    for i = 1, nargs do
        local arg = args[i]
        req[nbits] = &quot;$&quot;
        req[nbits + 1] = #arg
        req[nbits + 2] = &quot;\r\n&quot;
        req[nbits + 3] = arg
        req[nbits + 4] = &quot;\r\n&quot;
        nbits = nbits + 5
    end
    return req
en
</code></pre><p>这个函数预先生成了数组 <code>req</code>，它的大小由函数的入参来决定，这样就可以保证尽量不浪费空间。</p><p>然后，它使用 <code>nbits</code> 这个变量，来自己维护 <code>req</code> 的下标，自然就抛弃了 Lua 内置的 <code>table.insert</code> 函数和获取长度的操作符 <code>#</code>。你可以看到，在 for 循环中，<code>nbits + 1</code> 等一些运算，就是直接用下标的方式插入元素；并在最后用 <code>nbits = nbits + 5</code> ，让下标保持一个正确的值。</p><p>这种的好处很明显，它省略了获取数组大小这个 O(n) 的操作，而是直接用下标访问，时间复杂度也变成了 O(1) 。当然，缺点也一样明显，那就是降低了代码的可读性，并且出错概率大大提高，可以说，这是一把双刃剑。</p><h2>循环使用单个 table</h2><p>既然 table 这么来之不易，我们自然要好好珍惜，尽量做到重复使用。不过，循环利用也是有条件的。我们先要把 table 中原有的数据清理干净，以免对下一个使用者造成污染。</p><p>这时，<code>table.clear</code> 函数就派上用场了。从它的名字你就能看出它的作用，它会把数组中的所有数据清空，但数组的大小不会变。也就是说，你用 <code>table.new(narray, nhash)</code> 生了一个长度为 100 的数组，clear 后，长度还是 100。</p><p>为了让你能够更清楚它的实现，下面我给出了一个代码示例，它兼容了标准 Lua：</p><pre><code>local ok, clear_tab = pcall(require, &quot;table.clear&quot;)
  if not ok then
    clear_tab = function (tab)
      for k, _ in pairs(tab) do
        tab[k] = nil
      end
    end
  end
</code></pre><p>可以看到，clear 函数实际上就是把每一个元素都置为了nil。</p><p>一般来说，我们会把这种循环使用的 table，放在一个模块的 top level 中。这样，在你使用模块中的函数的时候，就可以根据自己的实际情况来决定，到底是直接使用，还是 clear 后再使用。</p><p>比如我们来看一个实际应用的例子。下面这段  <a href="https://github.com/iresty/apisix/blob/master/lua/apisix/plugin.lua">伪代码</a>  取自开源的微服务 API 网关 APISIX，这是它在加载插件时候的逻辑：</p><pre><code>local local_plugins = {}


function load()
    core.table.clear(local_plugins)


    local local_conf = core.config.local_conf()
    local plugin_names = local_conf.plugins


    local processed = {}
    for _, name in ipairs(plugin_names) do
        if processed[name] == nil then
            processed[name] = true
            insert_tab(local_plugins, name)
        end
    end


    return local_plugins
</code></pre><p>你可以看到，<code>local_plugins</code> 这个数组，是 plugin 这个模块的 top level 变量。在 load 这个加载插件函数的开始位置， table 就会被清空，然后根据当前的情况生成新的插件列表。</p><h2>table 池</h2><p>到现在，你就掌握了对单个 table 循环使用的优化方法了。那么更进一步，你还可以用缓存池的方式来保存多个 table，以便随用随取，官方提供的 <code>lua-tablepool</code> 正是出于这个目的。</p><p>下面这段代码，展示了 table 池的基本使用方法。我们可以从指定的池子中获取一个 table，使用完以后再释放回去：</p><pre><code>local tablepool = require &quot;tablepool&quot;
 local tablepool_fetch = tablepool.fetch
 local tablepool_release = tablepool.release


local pool_name = &quot;some_tag&quot; 
local function do_sth()
     local t = tablepool_fetch(pool_name, 10, 0)
     -- -- using t for some purposes
    tablepool_release(pool_name, t) 
end
</code></pre><p>显然，tablepool 中会用到前面我们介绍过的几个方法，而且它的代码只有不到一百行，所以，如果你学有余力，我十分推荐你可以自己搜索并研究一下。这里，我主要介绍下它的两个 API。</p><p>第一个是 fetch 方法，它的参数和 table.new 基本一样，只是多了一个 <code>pool_name</code>。如果池子中没有空闲的数组，fetch 方法就会调用 table.new 来新建一个数组。</p><pre><code>tablepool.fetch(pool_name, narr, nrec)
</code></pre><p>第二个是 release 这个把 table 放回池子的函数。在它的参数中，最后的 <code>no_clear</code> ，用来配置是否要调用 table.clear 把数组清空。</p><pre><code>tablepool.release(pool_name, tb, [no_clear])
</code></pre><p>你看，我们前面介绍到的方法，到这里是不是就全部串联起来了？</p><p>不过，注意不要因此滥用tablepool。tablepool 在实际项目中的使用并不多，比如 Kong 中就没有用到，APISIX 也只有少数几个调用。大多数情况下，不用 tablepool 的这层封装，也是足够我们使用的。</p><h2>写在最后</h2><p>性能优化，是 OpenResty 中的硬骨头，也是我们大家关注的热点。今天我介绍了table相关的性能优化技巧，希望能对你的实际项目有所帮助。</p><p>最后给你留一个作业题：你可以自己做个性能测试，对比下使用 table 相关优化技巧前后的性能差异吗？欢迎留言和我交流，你的做法和观点都是我希望听到的声音，也欢迎你把这篇文章分享出去，让更多的人一起参与进来。</p><p></p>
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
  <div class="_2_QraFYR_0">尝试了以下性能测试：<br>（1）预先生成数组<br>resty -e &#39;<br>local begin = ngx.now()<br>local t = {}<br>for i = 1,100000000  do<br>  table.insert(t, i)<br>end<br>ngx.update_time()<br>print(ngx.now()-begin)<br>&#39;<br>用时：21.62700009346<br>resty -e &#39;<br>local new_tab = require &quot;table.new&quot;<br>local begin = ngx.now()<br>local t = new_tab(100000000, 0)<br>for i = 1,100000000  do<br>  table.insert(t, i)<br>end<br>ngx.update_time()<br>print(ngx.now()-begin)<br>&#39;<br>用时：7.595999956131<br><br>（2）自己计算table下标<br>--使用insert函数<br>resty -e &#39;<br>local new_tab = require &quot;table.new&quot;<br>local begin = ngx.now()<br>local t = new_tab(100000000, 0)<br>for i = 1,100000000  do<br>  table.insert(t, i)<br>end<br>ngx.update_time()<br>print(ngx.now()-begin)<br>&#39;<br>用时：7.6459999084473<br>--自己计算下标<br>resty -e &#39;<br>local new_tab = require &quot;table.new&quot;<br>local begin = ngx.now()<br>local t = new_tab(100000000, 0)<br>for i = 1,100000000  do<br>  t[i] = i<br>end<br>ngx.update_time()<br>print(ngx.now()-begin)<br>&#39;<br>用时：0.33599996566772<br><br>（3）循环使用单个table<br>--分别建立两个table<br>resty -e &#39;<br>local new_tab = require &quot;table.new&quot;<br>local begin = ngx.now()<br>local t1 = new_tab(100000000, 0)<br>for i = 1,100000000  do<br>  t1[i] = i<br>end<br>local t2 = new_tab(100000000, 0)<br>for i = 1,100000000  do<br>  t2[i] = i<br>end<br>ngx.update_time()<br>print(ngx.now()-begin)<br>&#39;<br>用时：0.66700005531311<br><br>--复用table<br>resty -e &#39;<br>local new_tab = require &quot;table.new&quot;<br>local begin = ngx.now()<br>local t = new_tab(100000000, 0)<br>for i = 1,100000000  do<br>  t[i] = i<br>end<br>for i = 1,100000000  do<br>  t[i] = i<br>end<br>ngx.update_time()<br>print(ngx.now()-begin)<br>&#39;<br>用时：0.40800023078918</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-11 23:02:00</div>
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
  <div class="_2_QraFYR_0">OpenResty是从哪个版本开始完全绑定LuaJIT的呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我记得是 1.15 这个版本开始就只支持自己的 LuaJIT 了。lua 和官方的 luajit 都不再支持了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-25 17:05:30</div>
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
  <div class="_2_QraFYR_0">table. clean函数的实现是循环遍历每一个元素，然后置空，相比重新创建一个table来说效率更高。这个是和table的大小无关的是吧，table的重利用的效率总是高于新建一个表格。<br>这样也有一个要求，重利用的table的大小需要满足多重情况。APISIX中的表格变量都没有使用table. new 来创建，是因为大小不确定吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: APISIX 里面是做了一层封装：core.table.new()，其实底层还是 table.new</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-25 18:58:44</div>
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
  <div class="_2_QraFYR_0">老师，APISIX中的load函数少个end. </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 多谢</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-25 18:51:46</div>
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
  <div class="_2_QraFYR_0">老师，文中有这样一段话：<br>也就是说，你用 table.new(narray, nhash) 生了一个长度为 100 的数组，clear 后，长度还是 100。<br>哈哈，生了一个数组</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 好眼神：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-25 18:36:30</div>
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
  <div class="_2_QraFYR_0">老师好，文章中提到table. new 函数属于LuaJIT的扩展，是OpenResty维护的LuaJIT分支的扩展还是就是LuaJIT的扩展？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个函数是 LuaJIT 的扩展</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-25 17:03:45</div>
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
  <div class="_2_QraFYR_0">老师，数组以空间换时间创建时，应该是没有resize 和rehash 操作的，而不是这些操作一次完成的是吧？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，如果不超过 table.new 的预设大小，是没有这些操作的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-25 16:52:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/db/75/8264d2c3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>见习勇者</span>
  </div>
  <div class="_2_QraFYR_0">问老师，您好，我有个疑问，您举例的table复用的阶段是在init_worker阶段，这个阶段只会在初始化worker的时候调用一次，并不会影响实际的请求，有必要进行这样的优化么？对后续的请求处理有提升么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 算是一个习惯了，如果是 init worker 阶段确实优化不了多少</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-13 14:25:30</div>
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
  <div class="_2_QraFYR_0">老师，lua最大支持的数是2的53次方，那如何处理ipv6地址相关的需求呢，就是128位大数的加减乘除，比较等运算，目前lua好像都不支持，对于这种情况怎么处理比较好呢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-11 12:10:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/aa/a5/194613c1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>dingjiayi</span>
  </div>
  <div class="_2_QraFYR_0">温老师，有没有比较优雅的方式去切割 access.log 和 error.log<br>现在我看到网上介绍的方案都是 lograte 或者crontab 启动定时任务去切割<br>想请问nginx 有自带切割日志的功能吗？或者有比crontab 更优雅的方案吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你可以使用 OpenResty 的特权进程来做日志文件的切割，等于自己用 timer.every 来实现了crontab的功能。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-10 15:09:12</div>
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
  <div class="_2_QraFYR_0">table优化总结：<br>预先定义大小：空间换时间<br>table池：对象复用<br>计算下标：改进算法</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，很多优化都是抠出来的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-09 16:00:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/TTPicMEZ5s1zKiaKYecIljicRV9dibITPM0958W3VuNHXlTQic2Dj6XYGibF7dqrG3JWr0LMx7hY9zXGzuvTDNGw8KAA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>阳光梦</span>
  </div>
  <div class="_2_QraFYR_0">您好，使用标准nginx和luajit，有没有您说的这样优化方法？谢谢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这里面讲的优化方法，是适用于 LuaJIT 的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-09 07:40:13</div>
  </div>
</div>
</div>
</li>
</ul>