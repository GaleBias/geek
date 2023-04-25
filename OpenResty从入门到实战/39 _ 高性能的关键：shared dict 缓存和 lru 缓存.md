<audio title="39 _ 高性能的关键：shared dict 缓存和 lru 缓存" src="https://static001.geekbang.org/resource/audio/d0/f6/d03110e6462211c716e17d52f14b61f6.mp3" controls="controls"></audio> 
<p>你好，我是温铭。</p><p>在前面几节课中，我已经把 OpenResty 自身的优化技巧和性能调优的工具都介绍过了，分别涉及到字符串、table、Lua API、LuaJIT、SystemTap、火焰图等。</p><p>这些都是系统优化的基石，需要你好好掌握。但是，只懂得它们，还是不足以面对真实的业务场景。在一个稍微复杂一些的业务中，保持高性能是一个系统性的工作，并不仅仅是代码和网关层面的优化。它会涉及到数据库、网络、协议、缓存、磁盘等各个方面，这也正是架构师存在的意义。</p><p>今天这节课，就让我们一起来看下，性能优化中扮演非常关键角色的组件——缓存，看看它在 OpenResty 中是如何使用和进行优化的。</p><h2>缓存</h2><p>在硬件层面，大部分的计算机硬件都会用缓存来提高速度，比如 CPU 会有多级缓存，RAID 卡也有读写缓存。而在软件层面，我们用的数据库就是一个缓存设计非常好的例子。在 SQL 语句的优化、索引设计以及磁盘读写的各个地方，都有缓存。</p><p>这里，我也建议你在设计自己的缓存之前，先去了解下 MySQL 里面的各种缓存机制。我给你推荐的资料是《High Performance MySQL》 这本非常棒的书。我在多年前负责数据库的时候，从这本书中获益良多，而且后来不少其他的优化场景，也借鉴了 MySQL 的设计。</p><!-- [[[read_end]]] --><p>回到缓存上来说，我们知道，一个生产环境的缓存系统，需要根据自己的业务场景和系统瓶颈，来找出最好的方案。这是一门平衡的艺术。</p><p>一般来说，缓存有两个原则。</p><ul>
<li>一是越靠近用户的请求越好。比如，能用本地缓存的就不要发送 HTTP 请求，能用 CDN 缓存的就不要打到源站，能用 OpenResty 缓存的就不要打到数据库。</li>
<li>二是尽量使用本进程和本机的缓存解决。因为跨了进程和机器甚至机房，缓存的网络开销就会非常大，这一点在高并发的时候会非常明显。</li>
</ul><p>自然，在OpenResty 中，缓存的设计和使用也遵循这两个原则。OpenResty 中有两个缓存的组件：shared dict 缓存和 lru 缓存。前者只能缓存字符串对象，缓存的数据有且只有一份，每一个 worker 都可以进行访问，所以常用于 worker 之间的数据通信。后者则可以缓存所有的 Lua 对象，但只能在单个 worker 进程内访问，有多少个 worker，就会有多少份缓存数据。</p><p>下面这两个简单的表格，可以说明 shared dict 和 lru 缓存的区别：</p><p><img src="https://static001.geekbang.org/resource/image/ba/cd/baa08571c1ca48585d14a6141e2f00cd.png?wh=973*174" alt=""><br>
<img src="https://static001.geekbang.org/resource/image/c3/a9/c3f7415b4cb793a556f27b08da370ca9.png?wh=973*174" alt=""><br>
shared dict 和 lru 缓存，并没有哪一个更好的说法，而是应该根据你的场景来配合使用。</p><ul>
<li>如果你没有 worker 之间共享数据的需求，那么lru 可以缓存数组、函数等复杂的数据类型，并且性能最高，自然是首选。</li>
<li>但如果你需要在 worker 之间共享数据，那就可以在 lru 缓存的基础上，加上 shared dict 的缓存，构成两级缓存的架构。</li>
</ul><p>接下来，我们具体来看看这两种缓存方式。</p><h2>共享字典缓存</h2><p>在 Lua 章节中，我们已经对 shared dict 做了具体的介绍，这里先简单回顾下它的使用方法：</p><pre><code>$ resty --shdict='dogs 1m' -e 'local dict = ngx.shared.dogs
                               dict:set(&quot;Tom&quot;, 56)
                               print(dict:get(&quot;Tom&quot;))'
</code></pre><p>你需要事先在 Nginx 的配置文件中，声明一个内存区 dogs，然后在 Lua 代码中才可以使用。如果你在使用的过程中，发现给 dogs 分配的空间不够用，那么是需要先修改 Nginx 配置文件，然后重新加载 Nginx 才能生效的。因为我们并不能在运行时进行扩容和缩容。</p><p>下面，我们重点聊下，在共享字典缓存中，和性能相关的几个问题。</p><h3>缓存数据的序列化</h3><p>第一个问题，缓存数据的序列化。由于共享字典中只能缓存字符串对象，所以，如果你想要缓存数组，就少不了要在 set 的时候要做一次序列化，在 get 的时候做一次反序列化：</p><pre><code>resty --shdict='dogs 1m' -e 'local dict = ngx.shared.dogs
                        dict:set(&quot;Tom&quot;, require(&quot;cjson&quot;).encode({a=111}))
                        print(require(&quot;cjson&quot;).decode(dict:get(&quot;Tom&quot;)).a)'
</code></pre><p>不过，这类序列化和反序列化操作是非常消耗 CPU 资源的。如果每个请求都有那么几次这种操作，那么，在火焰图上你就能很明显地看到它们的消耗。</p><p>所以，如何在共享字典里避免这种消耗呢？其实这里并没有什么好的办法，要么在业务层面避免把数组放到共享字典里面；要么自己去手工拼接字符串为JSON 格式，当然，这也会带来字符串拼接的性能消耗，以及可能会隐藏更多的 bug 在其中。</p><p>大部分的序列化都是可以在业务层面进行拆解的。你可以把数组的内容打散，分别用字符串的形式存储在共享字典中。如果还不行的话，那么也可以把数组缓存在 lru 中，用内存空间来换取程序的便捷性和性能。</p><p>此外，缓存中的 key 也应该尽量选择短和有意义的，这样不仅可以节省空间，也方便后续的调试。</p><h3>stale 数据</h3><p>共享字典中还有一个 <code>get_stale</code> 的读取数据的方法，相比 <code>get</code> 方法，多了一个过期数据的返回值：</p><pre><code>resty --shdict='dogs 1m' -e 'local dict = ngx.shared.dogs
                            dict:set(&quot;Tom&quot;, 56, 0.01)
                            ngx.sleep(0.02)
                             local val, flags, stale = dict:get_stale(&quot;Tom&quot;)
                            print(val)'
</code></pre><p>在上面的这个示例中，数据只在共享字典中缓存了 0.01 秒，在 set 后的 0.02 秒后，数据就已经超时了。这时候，通过 get 接口就不会获取到数据了，但通过 <code>get_stale</code> 还可能获取到过期的数据。这里我之所以用“可能”两个字，是因为过期数据所占用的空间，是有一定几率被回收，再给其他数据使用的，这也就是 LRU 算法。</p><p>看到这里，你可能会有疑惑吗：获取已经过期的数据有什么用呢？不要忘记了，我们在 shared dict 中存放的是缓存数据，即使缓存数据过期了，也并不意味着源数据就一定有更新。</p><p>举个例子，数据源存储在 MySQL 中，我们从 MySQL 中获取到数据后，在 shared dict 中设置了 5 秒超时，那么，当这个数据过期后，我们就会有两个选择：</p><ul>
<li>当这个数据不存在时，重新去 MySQL 中再查询一次，把结果放到缓存中；</li>
<li>判断 MySQL 的数据是否发生了变化，如果没有变化，就把缓存中过期的数据读取出来，修改它的过期时间，让它继续生效。</li>
</ul><p>很明显，后者是更优化的方案，这样可以尽可能少地去和 MySQL 交互，让终端的请求都从最快的缓存中获取数据。</p><p>这时候，如何判断数据源中的数据是否发生了变化，就成为了我们需要考虑和解决的问题。接下来，让我们以 lru 缓存为例，看看一个实际的项目是如何来解决这个问题的。</p><h2>lru 缓存</h2><p>lru 缓存的接口只有 5 个：<code>new</code>、<code>set</code>、<code>get</code>、<code>delete</code> 和 <code>flush_all</code>。和上面问题相关的就只有 <code>get</code> 接口，让我们先来了解下这个接口是如何使用的：</p><pre><code>resty -e 'local lrucache = require &quot;resty.lrucache&quot;
local cache, err = lrucache.new(200)
cache:set(&quot;dog&quot;, 32, 0.01)
ngx.sleep(0.02)
local data, stale_data = cache:get(&quot;dog&quot;)
print(stale_data)'
</code></pre><p>你可以看到，在lru 缓存中， get 接口的第二个返回值直接就是 <code>stale_data</code>，而不是像 shared dict 那样分为了 <code>get</code> 和 <code>get_stale</code> 两个不同的 API。这样的接口封装，对于使用过期数据来说显然更加友好。</p><p>在实际的项目中，我们一般推荐使用版本号来区分不同的数据，这样，在数据发声变化后，它的版本号也就跟着变了。比如，在 etcd 中的 <code>modifiedIndex</code> ，就可以拿来当作版本号，来标记数据是否发生了变化。有了版本号的概念后，我们就可以对 lru 缓存做一个简单的二次封装，比如来看下面的伪码，摘自<a href="https://github.com/iresty/apisix/blob/master/lua/apisix/core/lrucache.lua">https://github.com/iresty/apisix/blob/master/lua/apisix/core/lrucache.lua</a> ：</p><pre><code>local function (key, version, create_obj_fun, ...)
    local obj, stale_obj = lru_obj:get(key)
    -- 如果数据没有过期，并且版本没有变化，就直接返回缓存数据
    if obj and obj._cache_ver == version then
        return obj
    end

    -- 如果数据已经过期，但还能获取到，并且版本没有变化，就直接返回缓存中的过期数据
    if stale_obj and stale_obj._cache_ver == version then
        lru_obj:set(key, obj, item_ttl)
        return stale_obj
    end

    -- 如果找不到过期数据，或者版本号有变化，就从数据源获取数据
    local obj, err = create_obj_fun(...)
    obj._cache_ver = version
    lru_obj:set(key, obj, item_ttl)
    return obj, err
end
</code></pre><p>从这段代码中你可以看到，我们通过引入版本号的概念，在版本号没有变化的情况下，充分利用了过期数据来减少对数据源的压力，达到了性能的最优。</p><p>除此之外，在上面的方案中，其实还有一个潜在的很大优化点，那就是我们把 key 和版本号做了分离，把版本号作为 value 的一个属性。</p><p>我们知道，更常规的做法是把版本号写入 key 中。比如 key 的值是 <code>key_1234</code>，这种做法非常普遍，但在 OpenResty 的环境下，这样其实是存在浪费的。为什么这么说呢？</p><p>举个例子你就明白了。假如版本号每分钟变化一次，那么<code>key_1234</code> 过一分钟就变为了 <code>key_1235</code>，一个小时就会重新生成 60 个不同的 key，以及 60 个 value。这也就意味着， Lua GC 需要回收 59 个键值对背后的 Lua 对象。如果你的更新更加频繁，那么对象的新建和 GC 显然会消耗更多的资源。</p><p>当然，这些消耗也可以很简单地避免，那就是把版本号从 key 挪到 value 中。这样，一个 key 不管更新地多么频繁，也只有固定的两个 Lua 对象。可以看出，这样的优化技巧非常巧妙，不过，简单巧妙的技巧背后，其实需要你对 OpenResty 的 API 以及缓存的机制都有很深入的了解才可以。</p><h2>写在最后</h2><p>诚然，OpenResty 的文档比较详细，但如何和业务组合以产生最大的优化效果，就需要你自己来来体会和领悟了。很多时候，文档中只有一两句的地方，比如 stale data这样的，却会产生巨大的性能差异。</p><p>那么，你在使用 OpenResty 的过程中，是否有过类似的经历呢？欢迎留言和我分享，也欢迎你把这篇文章分享出去，我们一起学习和进步。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/47/cc/04a749e1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Shliesce</span>
  </div>
  <div class="_2_QraFYR_0">正在写一个与consul联动动态更新upstream的插件，我们之前使用的微博upsync模块功能不是太理想。目前我也是把upstream信息放在shared dict中的，因为consul自带的就有long polling和index，我考虑把index和value分别存在两个shared dict中，这样定时器只会频繁地访问index这个shared dict，减少全局锁对其他worker的影响，但是我估计性能还达不到我们产线量级的要求，就像老师说的每次请求都要反序列一次，后续可能会再引入lru cache。。老师有什么好的建议吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 建议看看 lua-resty-mlcache，等于多加一层 lru cache。mlcache提供了 `l1_serializer`，专门用于处理 shared dict 提升到 lru cache 时候对数据的处理，可以尽可能的减少序列化。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-23 11:30:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2a/f5/7a/7351b235.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ch3cknull</span>
  </div>
  <div class="_2_QraFYR_0">lru缓存 示例部分的链接失效了，这个链接可用<br>https:&#47;&#47;github.com&#47;apache&#47;apisix&#47;blob&#47;master&#47;apisix&#47;core&#47;lrucache.lua</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-19 19:38:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/89/46/0b7828a1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小胡子</span>
  </div>
  <div class="_2_QraFYR_0">判断数据是否发生了变化，下面函数中的version怎么来的呢。。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-18 09:42:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/7e/e3/e87db3c0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Cobol</span>
  </div>
  <div class="_2_QraFYR_0">采用 Kong 的 Proxy-Cache 去缓存大文件(&gt;80M)的时候，wget 的速度比直接下载还慢，应该是与 cjson 的反序列化有关。如果要调优，是要调整共享内存大小吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-09 18:40:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/ba/c2/50e24e09.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>卫智雄</span>
  </div>
  <div class="_2_QraFYR_0">嗨~，请教一下， 我使用 ngx.shared.DICT.incr 方法, 发现 value 会跳变<br>下面是我得配置和测试方式<br>```<br>lua_shared_dict counter_ip_dict 100m;<br><br>server {<br>    listen 80;<br>    server_name test.com;<br><br>    access_by_lua_block {<br>        local ngx_var = ngx.var<br>        local counter_ip_dict = ngx.shared.counter_ip_dict<br>        local remoteIp = ngx_var.remote_addr<br><br>        local function limitRate(limitKey, maxReqs, expTime)<br><br>            local ipCount = counter_ip_dict:get(limitKey)<br>            -- ngx.say(ipCount)<br><br>            if ipCount == nil then<br>                counter_ip_dict:add(limitKey, 1, expTime)<br>            else<br>                if ipCount &gt; maxReqs then<br>                    ngx.print(&#39;ipCount: &#39;)<br>                    ngx.say(ipCount)<br>                    ngx.exit(200)<br>                else<br>                    counter_ip_dict:incr(limitKey, 1)<br>                end<br>            end<br><br>        end<br><br>        limitRate(remoteIp, 10, 20)<br>    }<br><br>    location &#47; {<br>        root &#47;data&#47;html;<br>        index index.html;<br>    }<br><br>}<br><br>```<br>```<br>for((i=1;i&lt;12;i++));do echo &quot;-----$i----&quot; ; curl -s -w &quot;http_code: %{http_code}\n&quot; -H &#39;Host: test.com&#39; 127.0.0.1 ; done<br>```</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-11 13:25:23</div>
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
  <div class="_2_QraFYR_0">跟着老师一起精进。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-23 15:13:42</div>
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
  <div class="_2_QraFYR_0">赞👍</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-23 08:58:02</div>
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
  <div class="_2_QraFYR_0">正在写API网关的插件，也用了sharedict，感谢老师的思路。sharedict和lru确实有点难两全的感觉。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-23 08:42:04</div>
  </div>
</div>
</div>
</li>
</ul>