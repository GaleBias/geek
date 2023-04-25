<audio title="18 _ worker间的通信法宝：最重要的数据结构之shared dict" src="https://static001.geekbang.org/resource/audio/b7/a8/b7a770648c696dd5b49fac1a887615a8.mp3" controls="controls"></audio> 
<p>你好，我是温铭。</p><p>前面我们讲过，在 Lua 中， table 是唯一的数据结构。与之对应的一个事实是，共享内存字典shared dict，是你在 OpenResty 编程中最为重要的数据结构。它不仅支持数据的存放和读取，还支持原子计数和队列操作。</p><p>基于 shared dict，你可以实现多个 worker 之间的缓存和通信，以及限流限速、流量统计等功能。你可以把 shared dict 当作简单的 Redis 来使用，只不过 shared dict 中的数据不能持久化，所以你存放在其中的数据，一定要考虑到丢失的情况。</p><h2>数据共享的几种方式</h2><p>在编写 OpenResty Lua 代码的过程中，你不可避免地会遇到，在一个请求的不同阶段、不同 worker 之间共享数据的情况，还可能需要在 Lua 和 C 代码之间共享数据。</p><p>所以，在正式介绍 shared dict 的 API 之前，先让我们了解一下，OpenResty 中常见的几种数据共享的方法；并学会根据实际情况，选择较为合适的数据共享方式。</p><p><strong>第一种是 Nginx 中的变量</strong>。它可以在 Nginx C 模块之间共享数据，自然的，也可以在 C 模块和 OpenResty 提供的 <code>lua-nginx-module</code> 之间共享数据，比如下面这段代码：</p><!-- [[[read_end]]] --><pre><code>location /foo {
     set $my_var ''; # this line is required to create $my_var at config time
     content_by_lua_block {
         ngx.var.my_var = 123;
         ...
     }
 }
</code></pre><p>不过，使用 Nginx 变量这种方式来共享数据是比较慢的，因为它涉及到 hash 查找和内存分配。同时，这种方法有其局限性，只能用来存储字符串，不能支持复杂的 Lua 类型。</p><p><strong>第二种是<code>ngx.ctx</code>，可以在同一个请求的不同阶段之间共享数据</strong>。它其实就是一个普通的 Lua 的 table，所以速度很快，还可以存储各种 Lua 的对象。它的生命周期是请求级别的，当一个请求结束的时候，<code>ngx.ctx</code> 也会跟着被销毁掉。</p><p>下面是一个典型的使用场景，我们用 <code>ngx.ctx</code> 来缓存 <code>Nginx 变量</code> 这种昂贵的调用，并在不同阶段都可以使用到它：</p><pre><code>location /test {
     rewrite_by_lua_block {
         ngx.ctx.host = ngx.var.host
     }
     access_by_lua_block {
        if (ngx.ctx.host == 'openresty.org') then
            ngx.ctx.host = 'test.com'
        end
     }
     content_by_lua_block {
         ngx.say(ngx.ctx.host)
     }
 }
</code></pre><p>这时，如果你使用 curl 访问的话：</p><pre><code>curl -i 127.0.0.1:8080/test -H 'host:openresty.org'
</code></pre><p>就会打印出 <code>test.com</code>，可以表明 <code>ngx.ctx</code> 的确是在不同阶段共享了数据。当然，你还可以自己动手修改上面的例子，保存 table 等更复杂的对象，而非简单的字符串，看看它是否满足你的预期。</p><p>不过，这里需要特别注意的是，正因为 <code>ngx.ctx</code> 的生命周期是请求级别的，所以它并不能在模块级别进行缓存。比如，我在 <code>foo.lua</code> 文件中这样使用就是错误的：</p><pre><code>local ngx_ctx = ngx.ctx

local function bar()
    ngx_ctx.host =  'test.com'
end
</code></pre><p>我们应该在函数级别进行调用和缓存：</p><pre><code>local ngx = ngx

local function bar()
    ngx_ctx.host =  'test.com'
end
</code></pre><p><code>ngx.ctx</code> 还有很多的细节，后面的性能优化部分，我们再继续探讨。</p><p>接着往下看，<strong>第三种方法是使用<code>模块级别的变量</code>，在同一个 worker 内的所有请求之间共享数据</strong>。跟前面的 Nginx 变量和 <code>ngx.ctx</code> 不一样，这种方法有些不太好理解。不过别着急，概念抽象，代码先行，让我们先来看个例子，弄明白什么是 <code>模块级别的变量</code>：</p><pre><code>-- mydata.lua
 local _M = {}

 local data = {
     dog = 3,
     cat = 4,
     pig = 5,
 }

 function _M.get_age(name)
     return data[name]
 end

 return _M
</code></pre><p>在 nginx.conf 的配置如下：</p><pre><code>location /lua {
     content_by_lua_block {
         local mydata = require &quot;mydata&quot;
         ngx.say(mydata.get_age(&quot;dog&quot;))
     }
 }
</code></pre><p>在这个示例中，<code>mydata</code> 就是一个模块，它只会被 worker 进程加载一次，之后，这个 worker 处理的所有请求，都会共享 <code>mydata</code> 模块的代码和数据。</p><p>自然，<code>mydata</code> 模块中的 <code>data</code> 这个变量，就是 <code>模块级别的变量</code>，它位于模块的 top level，也就是模块最开始的位置，所有函数都可以访问到它。</p><p>所以，你可以把需要在请求间共享的数据，放在模块的 top level 变量中。不过，需要特别注意的是，一般我们只用这种方式来保存<strong>只读的数据</strong>。如果涉及到写操作，你就要非常小心了，因为可能会有 <strong>race condition</strong>，这是<strong>非常难以定位的 bug</strong>。</p><p>我们可以通过下面这个最简化的例子来体会下：</p><pre><code>-- mydata.lua
 local _M = {}

 local data = {
     dog = 3,
     cat = 4,
     pig = 5,
 }

 function _M.incr_age(name)
     data[name]  = data[name] + 1
    return data[name]
 end

 return _M
</code></pre><p>在模块中，我们增加了 <code>incr_age</code> 这个函数，它会对 data 这个表的数据进行修改。</p><p>然后，在调用的代码中，我们增加了最关键的一行 <code>ngx.sleep(5)</code>，这个 sleep 是一个 yield 操作：</p><pre><code>location /lua {
     content_by_lua_block {
         local mydata = require &quot;mydata&quot;
         ngx.say(mydata. incr_age(&quot;dog&quot;))
         ngx.sleep(5) -- yield API
         ngx.say(mydata. incr_age(&quot;dog&quot;))
     }
 }
</code></pre><p>如果没有这行 sleep 代码（也可以是其他的非阻塞 IO 操作，比如访问 Redis 等），就不会有 yield 操作，也就不会产生竞争，那么，最后输出的数字就是顺序的。</p><p>但当我们加了这行代码后，哪怕只是在 sleep 的 5 秒钟内，也很可能就有其他请求调用了<code>mydata. incr_age</code> 函数，修改了变量的值，从而导致最后输出的数字不连续。要知道，在实际的代码中，逻辑不会这么简单，bug 的定位也一定会困难得多。</p><p>所以，除非你很确定这中间没有 yield 操作，不会把控制权交给 Nginx 事件循环，否则，我建议你还是保持对模块级别变量的只读。</p><p><strong>第四种，也是最后一种方法，用 shared dict 来共享数据，这些数据可以在多个 worker 之间共享。</strong></p><p>这种方法是基于红黑树实现的，性能很好，但也有自己的局限性——你必须事先在 Nginx 的配置文件中，声明共享内存的大小，并且这不能在运行期更改：</p><pre><code>lua_shared_dict dogs 10m;
</code></pre><p>shared dict 同样只能缓存字符串类型的数据，不支持复杂的 Lua 数据类型。这也就意味着，当我需要存放 table 等复杂的数据类型时，我将不得不使用 json 或者其他的方法，来序列化和反序列化，这自然会带来不小的性能损耗。</p><p>总之，还是那句话，这里并没有银弹，不存在一种完美的数据共享方式，你需要根据需求和场景，来组合多个方法来使用。</p><h2>共享字典</h2><p>上面数据共享的部分，我们花了很多的篇幅来学，有的人可能纳闷儿：它们看上去和 shared dict 没有直接关系，是不是有些文不对题呢？</p><p>事实并非如此，你可以自己想一下，为什么 OpenResty 中要有 shared dict 的存在呢？</p><p>回忆一下刚刚讲的几种方法，前面三种数据共享的范围都是在请求级别，或者单个 worker 级别。所以，在当前的 OpenResty 的实现中，只有 shared dict 可以完成 worker 间的数据共享，并借此实现 worker 之间的通信，这也是它存在的价值。</p><p>在我看来，明白一个技术为何存在，并弄清楚它和别的类似技术之间的差异和优势，远比你只会熟练调用它提供的 API 更为重要。这种技术视野，会给你带来一定程度的远见和洞察力，这也可以说是工程师和架构师的一个重要区别。</p><p>回到共享字典本身，它对外提供了 20多个 Lua API，不过所有的这些 API 都是原子操作，你不用担心多个 worker 和高并发的情况下的竞争问题。</p><p>这些 API 都有官方详细的<a href="https://github.com/openresty/lua-nginx-module#ngxshareddict">文档</a>，我就不再一一赘述了。这里我想再强调一下，任何技术课程的学习，都不能代替对官方文档的仔细研读。这些耗时的笨功夫，每个人都省不掉的。</p><p>继续看shared dict 的 API，这些 API可以分为下面三个大类，也就是字典读写类、队列操作类和管理类这三种。</p><h3>字典读写类</h3><p>首先来看字典读写类。在最初的版本中，只有字典读写类的 API，它们也是共享字典最常用的功能。下面是一个最简单的示例：</p><pre><code>$ resty --shdict='dogs 1m' -e 'local dict = ngx.shared.dogs
                               dict:set(&quot;Tom&quot;, 56)
                               print(dict:get(&quot;Tom&quot;))'
</code></pre><p>除了 set 外，OpenResty 还提供了 <code>safe_set</code>、<code>add</code>、<code>safe_add</code>、<code>replace</code> 这四种写入的方法。这里<code>safe</code> 前缀的含义是，在内存占满的情况下，不根据 LRU 淘汰旧的数据，而是写入失败并返回 <code>no memory</code> 的错误信息。</p><p>除了 get 外，OpenResty 还提供了 <code>get_stale</code> 的读取数据的方法，相比 <code>get</code> 方法，它多了一个过期数据的返回值：</p><pre><code>value, flags, stale = ngx.shared.DICT:get_stale(key)
</code></pre><p>你还可以调用 <code>delete</code> 方法来删除指定的 key，它和 <code>set(key, nil)</code> 是等价的。</p><h3>队列操作类</h3><p>再来看队列操作，它是 OpenResty 后续新增的功能，提供了和 Redis 类似的接口。队列中的每一个元素，都用 <code>ngx_http_lua_shdict_list_node_t</code> 来描述：</p><pre><code>typedef struct { 
    ngx_queue_t queue; 
    uint32_t value_len; 
    uint8_t value_type; 
    u_char data[1]; 
} ngx_http_lua_shdict_list_node_t;
</code></pre><p>我把这些队列操作 API 的 <a href="https://github.com/openresty/lua-nginx-module/pull/586/files">PR</a>  贴在了文章中，如果你对此感兴趣，可以跟着文档、测试案例和源码，来分析具体的实现。</p><p>不过，下面这 5 个队列 API，在文档中并没有对应的代码示例，这里我简单介绍一下：</p><ul>
<li>lpush/rpush，表示在队列两端增加元素；</li>
<li>lpop/rpop，表示在队列两端弹出元素；</li>
<li>llen，表示返回队列的元素数量。</li>
</ul><p>别忘了我们上节课讲过的另一个利器——测试案例。如果文档中没有，我们通常可以在测试案例中找到对应的代码。队列相关的测试，正是在 <code>145-shdict-list.t</code> 这个文件中：</p><pre><code>=== TEST 1: lpush &amp; lpop
--- http_config
    lua_shared_dict dogs 1m;
--- config
    location = /test {
        content_by_lua_block {
            local dogs = ngx.shared.dogs

            local len, err = dogs:lpush(&quot;foo&quot;, &quot;bar&quot;)
            if len then
                ngx.say(&quot;push success&quot;)
            else
                ngx.say(&quot;push err: &quot;, err)
            end

            local val, err = dogs:llen(&quot;foo&quot;)
            ngx.say(val, &quot; &quot;, err)

            local val, err = dogs:lpop(&quot;foo&quot;)
            ngx.say(val, &quot; &quot;, err)

            local val, err = dogs:llen(&quot;foo&quot;)
            ngx.say(val, &quot; &quot;, err)

            local val, err = dogs:lpop(&quot;foo&quot;)
            ngx.say(val, &quot; &quot;, err)
        }
    }
--- request
GET /test
--- response_body
push success
1 nil
bar nil
0 nil
nil nil
--- no_error_log
[error]
</code></pre><h3>管理类</h3><p>最后要说的管理类 API 也是后续新增的，属于社区呼声比较高的需求。其中，共享内存的使用情况就是最典型的例子。比如，用户申请了 100M 的空间作为 shared dict，那么这 100M 是否够用呢？里面存放了多少 key？具体是哪些 key 呢？这几个都是非常现实的问题。</p><p>对于这类问题，OpenResty 的官方态度，是希望用户使用火焰图来解决，即非侵入式，保持代码基的高效和整洁，而不是提供侵入式的 API 来直接返回结果。</p><p>但站在使用者友好角度来考虑，这些管理类 API 还是非常有必要的。毕竟开源项目是用来解决产品需求的，并不是展示技术本身的。所以，下面我们就来了解一下，这几个后续增加的管理类 API。</p><p>首先是 <code>get_keys(max_count?)</code>，它默认也只返回前 1024 个 key；如果你把 <code>max_count</code> 设置为 0，那就返回所有 key。</p><p>然后是 <code>capacity</code> 和 <code>free_space</code>，这两个 API 都属于 lua-resty-core 仓库，所以需要你 require 后才能使用：</p><pre><code>require &quot;resty.core.shdict&quot;

 local cats = ngx.shared.cats
 local capacity_bytes = cats:capacity()
 local free_page_bytes = cats:free_space()
</code></pre><p>它们分别返回的，是共享内存的大小（也就是 <code>lua_shared_dict</code> 中配置的大小）和空闲页的字节数。因为 shared dict 是按照页来分配的，即使 <code>free_space</code> 返回为 0，在已经分配的页面中也可能存在空间，所以它的返回值并不能代表共享内存实际被占用的情况。</p><h2>写在最后</h2><p>在实际的开发中，我们经常会用到多级缓存，OpenResty 的官方项目中也有对缓存的封装。你能找出来是哪几个项目吗？或者你知道一些其他缓存封装的 lua-resty 库吗？</p><p>欢迎留言和我分享，也欢迎你把这篇文章分享给你的同事、朋友，我们一起交流，一起进步。</p><p></p>
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
  <div class="_2_QraFYR_0">老师有两个问题请教：<br>1. ngx.var API是否是lua代码和Nginx C模块之间共享数据的唯一方法？有其他方法吗<br>2. 关于ngx.ctx，官方文档也提到说这个API 的查询也相对有点昂贵：&quot;The ngx.ctx lookup requires relatively expensive metamethod calls and it is much slower than explicitly passing per-request data along by your own function arguments.&quot;，如果这样是否还能说ng.ctx速度很快呢<br><br>另外补充一个文中未提到的，ngx.var可以在子请求中有效，也就是说可以跨location使用，而ngx.ctx不可以。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 其他的方式就是借助外部的储存了，比如 memcached 扥。<br>ngx.ctx 的快是相对于 ngx.var 而言的。<br>多谢补充，ngx.ctx 确实不能跨 location，有一个库 lua-resty-ctxdump 可以解决这个问题。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-24 15:22:02</div>
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
  <div class="_2_QraFYR_0">温铭老师，你好~有以下一些问题想请教一下~<br>1.ngx.var变量的作用域在nginx C模块之间、nginx C和lua-nginx-module模块之间。这个不太理解，从请求的角度来看，是一个工作进程中的单个请求吗？<br>2.文中有描述ngx.ctx是一种昂贵的调用，是虽然ngx.ctx访问速度快，但ngx.ctx在每一个请求中都会占据内存空间的昂贵吗？还有一个问题，ngx.ctx占据的内存空间的大小是动态增长的还是有大小限制的呢？<br>3.操作模块内的变量时，如果两个操作之间有阻塞操作，可能出现竞争。如果两个操作之间没有阻塞操作，恰好CPU时间到，当前进程进入就绪队列，也可能产生竞争的对吧？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. 是的，ngx.var 的生命周期和请求一致，请求结束它也就消失了。它的优势是数据可以在 C 模块和 Lua 代码中传递。<br>2. 原文中没有提到 ngx.ctx 是昂贵的操作吧？可能是我没有表达清楚，是我们会用 ngx.ctx 来替代 ngx.var 这种昂贵的操作，后者才是昂贵的。<br>3. 两个操作之间有 `yield 操作`，可能出现竞争，而不是`阻塞操作`，有阻塞操作是不会出现竞争的。只要不把主动权交给 Nginx 的事件循环，就不会有竞争。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-07 13:21:29</div>
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
  <div class="_2_QraFYR_0">想请问老师现在很多基于插件来实现的网关里面的插件配置是怎么保存在or里的呢？是sharedict吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 有些是共享字典，有些是 lrucache</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-05 22:08:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>zyonline</span>
  </div>
  <div class="_2_QraFYR_0">local ngx = ngx<br><br>local function bar()<br>    ngx_ctx.host =  &#39;test.com&#39;<br>end<br>中 ngx_ctx.host =  &#39;test.com&#39; 是不是应该是ngx.ctx.host = &#39;test.com&#39;</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，本来想写的是：<br>local ngx_ctx = ngx.ctx</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-05 09:21:07</div>
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
  <div class="_2_QraFYR_0">local ngx_ctx = ngx.ctx<br><br>local function bar()<br>    ngx_ctx.host =  &#39;test.com&#39;<br>end<br><br>上面内容放在access.lua中，比如access阶段包含此文件。为何不能执行第一句？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 有什么报错吗？ngx.ctx 是可以在 access 阶段使用的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-05 06:14:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/PiajxSqBRaEIBJYQ73yYqmiaU7Zg0BHPh9gpSglI79Dzcbob7I2tZOhTjbTTCw13KzVusYhLbKkukV9Ru5UfJMxQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_2d276a</span>
  </div>
  <div class="_2_QraFYR_0">这段话“如果没有这行 sleep 代码（也可以是其他的非阻塞 IO 操作，比如访问 Redis 等），就不会有 yield 操作，也就不会产生竞争，那么，最后输出的数字就是顺序的。”中的“非阻塞IO操作”在当前语境中含义是会产生竞争，还是不会产生竞争？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-04 10:39:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/44/64/abb7bfe3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>黑化肥发灰会灰花</span>
  </div>
  <div class="_2_QraFYR_0">请问一下有没办法在http{lua_shared_dict dogs 10m;} 定义的，在stream中获取到值，共享内容跨子系统？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-02 21:36:46</div>
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
  <div class="_2_QraFYR_0">老师您好，`yield`有可能导致race。哪些API（比如ngx.sleep）或哪些操作（比如访问redis）会yield？请问这个有参考文档，或者整理吗，谢谢！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-14 21:17:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/8e/31/28972804.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>阿海</span>
  </div>
  <div class="_2_QraFYR_0">请问老师，openresty有没有提供像redis-cli 这样的命令行工具从 终端来查看shared_dict的实际存储内容吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-10 18:00:44</div>
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
  <div class="_2_QraFYR_0">对于模块级别变量的测试，没有能够模拟出老师说的 race condition 状况。<br>请老师解答<br>同时在浏览器打开两个页面请求 localhost&#47;testmodule<br>多次刷新，每个页面都是稳定输出 4,5<br><br>模块文件如下 m.lua：<br><br>local _M = {}<br>local color = {<br>      red = 1,<br>      blue = 2,<br>      green = 3,<br>}  <br>_M.incGreen = function()<br>	color[&quot;green&quot;] = color[&quot;green&quot;] + 1<br>	return color[&quot;green&quot;]<br>end<br>_M.getGreen = function()<br>	return color[&quot;green&quot;]<br>end<br>return _M<br>----------------<br>config 配置如下<br>location &#47;testmodule {<br>     content_by_lua_block {<br>        local m = require &quot;m&quot;<br>		ngx.say(m.incGreen())<br>		ngx.sleep(5)<br>		ngx.say(m.incGreen())<br>     }<br>}<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-21 11:17:13</div>
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
  <div class="_2_QraFYR_0">老师，还有一个问题，local ngx_ctx = ngx.ctx 这个API缓存语句不能用于模块级别，要用于函数级别，那么local ngx_var = ngx.var 这个是不是也是同样的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-24 17:38:00</div>
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
  <div class="_2_QraFYR_0">关于ngx.ctx不要做模块级别的缓存，我做了以下的测试。<br>myctx.lua文件：<br>local _M = {}<br><br>local ngx_ctx = ngx.ctx<br><br>function _M.bar()<br>    ngx_ctx.host = &#39;test.com&#39;<br>end<br><br>return _M<br><br>nginx.conf文件：<br>worker_processes  1;<br><br>server {<br>    listen 9999;<br><br>    location ~&#47;(?&lt;myurl&gt;.*) {<br>        content_by_lua_block {<br>            local myctx = require &quot;myctx&quot;<br>            if ngx.var.myurl == &quot;1&quot; then<br>                ngx.say(ngx.ctx.host)<br>                ngx.ctx.host = &quot;who&quot;<br>                ngx.say(ngx.ctx.host)<br>                myctx.bar()<br>                ngx.say(ngx.ctx.host)<br>            else<br>                ngx.say(ngx.ctx.host)<br>            end<br>        }<br>    }<br>}<br><br>测试结果如下：<br>[root@localhost nginx]# curl localhost:9999&#47;1<br>nil<br>who<br>test.com<br>[root@localhost nginx]# curl localhost:9999&#47;1<br>nil<br>who<br>who<br>[root@localhost nginx]# curl localhost:9999&#47;1<br>nil<br>who<br>who<br>[root@localhost nginx]# curl localhost:9999&#47;2<br>nil<br>[root@localhost nginx]# curl localhost:9999&#47;2<br>nil<br><br>第一个请求可以理解，第二、第三个请求证明不要做模块级别的缓存。但是为什么这样呢？即使此时ngx_ctx保留的是第一个请求的ngx.ctx，为什么再次设置变量的值ngx_ctx.host就不管用了呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-07 13:32:35</div>
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
  <div class="_2_QraFYR_0">坚持跟读</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-05 17:29:59</div>
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
  <div class="_2_QraFYR_0">请教老师个问题，我想在请求返回给各户端前 拿到上游服务器的IP和端口，也就是upstream的具体哪个节点负责处理的请求，有什么办法么？我在 ngx.balancer 里看大部分是set的方法。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你在 ngx.balancer 的 set_current_peer 时，保存下上游的节点信息？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-05 13:54:15</div>
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
  <div class="_2_QraFYR_0">老师，luajit最大可用内存是2g，在openrety中如何将它提高？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1.15.8 这个最新版本中，默认已经提高了，支持了 64 位</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-05 09:59:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4b/24/8c9eaf7f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Seven</span>
  </div>
  <div class="_2_QraFYR_0">如果用Dict做一个pub&#47;sub, 由于数据不能持久化，那为了在teload或重启时保证数据完整，是不是必须自己做ACK机制证实每个客户端都收到了每一条数据？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-05 08:42:09</div>
  </div>
</div>
</div>
</li>
</ul>