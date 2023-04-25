<audio title="45 _ 不得不提的能力外延：OpenResty常用的第三方库" src="https://static001.geekbang.org/resource/audio/41/9b/41837000b5f03e3f0a3beb229a8fb19b.mp3" controls="controls"></audio> 
<p>你好，我是温铭。</p><p>对于开发语言和平台来讲，很多时候的学习，其实是对标准库和第三方库的学习，语法本身并不用花费很多的时间。这对 OpenResty 来说也是一样的，学完它自己的 API 和性能优化技巧后，就需要各种 lua-resty 库，来帮助我们把 OpenResty 的能力外延，以应用到更多的场景中去。</p><h2>去哪里找 lua-resty 库？</h2><p>和 PHP、Python、JavaScript 相比，当前 OpenResty 的标准库和第三方库还比较贫瘠，找出合适的 lua-resty 库还不是一件容易的事情。不过，这里仍然有两个推荐的渠道，可以帮你更快地找到它们。</p><p>我首先推荐的是由 Aapo 维护的 <code>awesome-resty</code> <a href="https://github.com/bungle/awesome-resty">仓库</a>，这个仓库分门别类地整理了和 OpenResty 相关的库，可以说是包罗万象，包括了 Nginx 的 C 模块、lua-resty 库、Web 框架、路由库、模板、测试框架等，是你寻找 OpenResty 资源的首选。</p><p>当然，如果你在 Aapo 的仓库中没有找到合适的库，那么还可以去 luarocks、opm和 GitHub 碰碰运气。有一些开源时间不长的、或者关注不多的库，可能就藏在其中。</p><p>在前面的课程中，我们已经接触了不少有用的库，比如 lua-resty-mlcache、lua-resty-traffic、lua-resty-shell 等。今天，在 OpenResty 性能优化部分的最后一节课，我们再来认识 3 个独具特色的周边库，它们都是由社区的开发者贡献的。</p><!-- [[[read_end]]] --><h2>ngx.var 的性能提升</h2><p>首先让我们来看一个 C 模块：<a href="https://github.com/iresty/lua-var-nginx-module">lua-var-nginx-module</a>。前面我曾经提到过，<code>ngx.var</code> 是一个性能损耗比较大的操作，在实际使用时，我们需要用 <code>ngx.ctx</code> 来做一层缓存。</p><p>那有没有什么方法，可以彻底解决 <code>ngx.var</code> 的性能问题呢？</p><p>这个 C 模块，就在这个方面做了一些尝试，效果也很显著，性能比起<code>ngx.var</code> 提升了 5 倍。它采用的是 FFI 的方式，所以，你需要在编译 OpenResty 的时候，先加上编译选项：</p><pre><code>./configure --prefix=/opt/openresty \
         --add-module=/path/to/lua-var-nginx-module
</code></pre><p>然后，使用 luarocks 的方式来安装 lua 库：</p><pre><code>luarocks install lua-resty-ngxvar
</code></pre><p>这里调用的方法也很简单，只需要一行 <code>fetch</code> 函数的调用就可以了。它的效果完全等价于原有的 <code>ngx.var.remote_addr</code>，来获取到终端的 IP 地址：</p><pre><code>content_by_lua_block {
    local var = require(&quot;resty.ngxvar&quot;)
    ngx.say(var.fetch(&quot;remote_addr&quot;))
}
</code></pre><p>知道了这些基本操作后，你可能更好奇的是，这个模块到底是怎么做到性能大幅度提升的呢？还是那句老话，源码面前无秘密，就让我们来看看 <code>remote_addr</code> 这个变量在其中是如何获取的吧：</p><pre><code>ngx_int_t 
ngx_http_lua_var_ffi_remote_addr(ngx_http_request_t *r, ngx_str_t *remote_addr) 
{ 
    remote_addr-&gt;len = r-&gt;connection-&gt;addr_text.len; 
    remote_addr-&gt;data = r-&gt;connection-&gt;addr_text.data; 

    return NGX_OK; 
}
</code></pre><p>阅读这段代码后，你会发现，这种 Lua FFI 的方式和 lua-resty-core 的做法如出一辙。它的优点很明显，使用 FFI 的方式来直接获取变量，绕过了 <code>ngx.var</code> 原有的查找逻辑；同时，缺点也很明显，那就是要为每一个希望获取的变量，都增加对应的 C 函数和 FFI 调用，这其实是一个体力活。</p><p>有人可能会问，我为什么会说这是体力活呢？上面的 C 代码看上去不是还挺有含量的吗？我们不妨来看看这几行代码的源头，它们出自  Nginx 代码中的 <code>src/http/ngx_http_variables.c</code>：</p><pre><code>static ngx_int_t
ngx_http_variable_remote_addr(ngx_http_request_t *r,
ngx_http_variable_value_t *v, uintptr_t data)
{
    v-&gt;len = r-&gt;connection-&gt;addr_text.len;
    v-&gt;valid = 1;
    v-&gt;no_cacheable = 0;
    v-&gt;not_found = 0;
    v-&gt;data = r-&gt;connection-&gt;addr_text.data;

    return NGX_OK;
}
</code></pre><p>看到源码后，谜底揭开了！<code>lua-var-nginx-module</code> 其实是 Nginx 变量代码的搬运工，并在外层做了 FFI 的封装，用这种方式达到了性能优化的目的。这其实也是一个很好的思路和优化方向。</p><p>这里我再多说几句，我们学习某个库或者某个工具，一定不要仅仅停留在操作使用的层面，还应该多问问为什么，多看看源码，在底层原理的层面上，我们才能学到更多的设计思想和解决思路。当然，我也非常鼓励你去贡献代码，以支持更多的 Nginx 变量。</p><h2>JSON Schema</h2><p>下面我介绍的是一个 lua-resty 库：<a href="https://github.com/xpol/lua-rapidjson">lua-rapidjson</a> 。它是对 <code>rapidjson</code> 这个腾讯开源的 JSON 库的封装，以性能见长。这里，我们着重介绍下它和 <code>cjson</code> 的不同之处，也就是支持 JSON Schema。</p><p>JSON Schema 是一个通用的标准，借助这个标准，我们就可以精确地描述接口中参数的格式，以及如何校验的问题。下面是一个简单的示例：</p><pre><code>&quot;stringArray&quot;: {
    &quot;type&quot;: &quot;array&quot;,
    &quot;items&quot;: { &quot;type&quot;: &quot;string&quot; },
    &quot;minItems&quot;: 1,
    &quot;uniqueItems&quot;: true
}
</code></pre><p>这段 JSON 准确地描述了 <code>stringArray</code> 这个参数的类型是字符串数组，并且数组不能为空，数组元素也不能重复。</p><p>而<code>lua-rapidjson</code>，则是可以让我们在 OpenResty 中来使用 JSON Schema，这能给接口的校验带来极大的便利。举个例子，比如对于前面介绍过的 limit count 限流接口，我们就可以用下面的 schema 来描述：</p><pre><code>local schema = {
    type = &quot;object&quot;,
    properties = {
        count = {type = &quot;integer&quot;, minimum = 0},
        time_window = {type = &quot;integer&quot;,  minimum = 0},
        key = {type = &quot;string&quot;, enum = {&quot;remote_addr&quot;, &quot;server_addr&quot;}},
        rejected_code = {type = &quot;integer&quot;, minimum = 200, maximum = 600},
    },
    additionalProperties = false,
    required = {&quot;count&quot;, &quot;time_window&quot;, &quot;key&quot;, &quot;rejected_code&quot;},
}
</code></pre><p>你会发现，这可以带来两个十分明显的收益：</p><ul>
<li>对前端来说，前端可以直接复用这个 schema 描述，用于前端页面的开发和参数校验，而不用再去关心后端；</li>
<li>而对后端来说，后端直接使用 <code>lua-rapidjson</code> 的 schema 校验函数 <code>SchemaValidator</code> 就能完成接口合法性的判断，更是无须编写多余的代码。</li>
</ul><h2>worker 间通信</h2><p>最后，我要讲的是可以实现 OpenResty 中 worker 间通信的 <a href="https://github.com/Kong/lua-resty-worker-events">lua-resty</a> 库。OpenResty 的 worker 之间，并没有机制可以直接通信，这显然会带来不少的问题。让我们设想这么一个场景：</p><blockquote>
<p>一个 OpenResty 服务有 24 个 worker 进程，管理员通过 REST HTTP 接口更新了系统的某项配置，这时候只有一个 worker 收到了管理员的更新操作，并把结果写入了数据库，更新了共享字典和自己 worker 内的 lru 缓存。那么，其他 23 个 worker 怎么才能被通知去更新这项配置呢？</p>
</blockquote><p>显然，多个 worker 之间需要一个通知的机制，才能完成上面的这个任务。在 OpenResty 自身不支持的情况下，我们就只能通过共享字典这个跨 worker 可以访问的空间，来曲线救国了。</p><p><code>lua-resty-worker-events</code> 便是这个思路的具体实现。它在共享字典中维护了一个版本号，在有新消息需要发布的时候，给这个版本号加一，并把消息内容放到以版本号为 key 的字典中：</p><pre><code>event_id, err = _dict:incr(KEY_LAST_ID, 1)
success, err = _dict:add(KEY_DATA .. tostring(event_id), json)
</code></pre><p>同时，在后台使用 <code>ngx.timer</code> 创建了一个默认间隔为 1 秒的 polling 循环，来不断地检测版本号是否有变化：</p><pre><code>local event_id, err = get_event_id()
if event_id == _last_event then
    return &quot;done&quot;
end
</code></pre><p>这样，一旦发现有新的事件通知需要处理时，就根据版本号从共享字典中获取消息内容：</p><pre><code>while _last_event &lt; event_id do
    count = count + 1
    _last_event = _last_event + 1
    data, err = _dict:get(KEY_DATA..tostring(_last_event))
end
</code></pre><p>总的来说，虽然 <code>lua-resty-worker-events</code> 会有 1 秒钟的延时，但还是实现了 worker 之间的事件通知机制，瑕不掩瑜。</p><p>不过，在一些实时性要求比较高的场景下，比如消息推送，OpenResty 缺少 worker 进程间直接通信的这个问题，就可能会给你带来一些困扰了。这一点目前没有更好的解决方案，如果你有好的想法，欢迎在 Github 或者 OpenResty 的邮件列表中来一起探讨。OpenResty 的很多功能都是由社区用户来驱动的，这样才能构造一个良性的生态循环。</p><h2>写在最后</h2><p>今天我们介绍的这三个库，都各具特色，也都为 OpenResty 的应用带来了更多的可能性。最后是一个互动话题，你是否发现过一些 OpenResty 周边有意思的库呢？或者对于今天提到的这几个库，你有什么发现或疑惑呢？欢迎留言和我分享，也欢迎你把这篇文章发给你身边的OpenResty使用者，一起交流和进步。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>欢乐马</span>
  </div>
  <div class="_2_QraFYR_0">nginx有一个第三方库nchan可以实现跨worker的消息推送，不确定性能稳不稳定？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-05 23:26:41</div>
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
  <div class="_2_QraFYR_0">实时性要求比较高的场景下 我们都是 worker 里面 bpop 从redis 加载数据，<br>如果要在内存件通讯  我觉得是不是可以在 sharedict lpop 实现一个类似 blpop的函数（阻塞出队列）。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-03 00:18:22</div>
  </div>
</div>
</div>
</li>
</ul>