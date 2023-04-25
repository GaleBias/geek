<audio title="41 _ lua-resty-x 封装，让你远离多级缓存之痛" src="https://static001.geekbang.org/resource/audio/18/10/18fb26d6243c091597db8cf4af421510.mp3" controls="controls"></audio> 
<p>你好，我是温铭。</p><p>前面两节课中，我们已经学习了 OpenResty 中的缓存，以及容易出错的缓存风暴问题，这些都是属于偏基础的一些知识。在实际的项目开发中，开发者自然更希望有一个已经把各种细节处理好并隐藏起来的开箱即用的库，可以拿来直接开发业务代码。</p><p>这其实就是分工的一个好处，基础组件的开发者，重心在于架构灵活、性能极致、代码稳定，并不需要关心上层业务逻辑；而应用层的工程师，更关心的是业务实现和快速迭代，并不希望被底层的各种技术细节分心。这中间的鸿沟，就需要有一些封装库来填平了。</p><p>OpenResty 中的缓存，也面临一样的问题。共享字典和 lru 缓存足够稳定和高效，但需要处理太多的细节。如果没有一些好用的封装，那么到达应用开发工程师的“最后一公里”，就会变得比较痛苦。这个时候，就要体现社区的重要性了。一个活跃的社区，会主动去发现鸿沟，并迅速地填平。</p><h2>lua-resty-memcached-shdict</h2><p>让我们回到缓存的封装上来。<code>lua-resty-memcached-shdict</code> 是 OpenResty 官方的一个项目，它使用 shared dict 为 memcached 做了一层封装，处理了缓存风暴和过期数据等细节。如果你的缓存数据正好存储在后端的 memcached 中，那么你可以尝试使用这个库。</p><!-- [[[read_end]]] --><p>它虽然是 OpenResty 官方开发的库，但默认并没有打进 OpenResty 的包中。如果你想在本地测试，需要先把它的<a href="https://github.com/openresty/lua-resty-memcached-shdict">源码</a>下载到本地 OpenResty 的查找路径下。</p><p>这个封装库，其实和我们上节课中提到的解决方案是一样的。它使用 <code>lua-resty-lock</code> 来做到互斥，在缓存失效的情况下，只有一个请求去 memcached 中获取数据，避免缓存风暴。如果没有获取到最新数据，则使用 stale 数据返回给终端。</p><p>不过，这个 lua-resty 库虽说是 OpenResty 官方的项目，但也并不完美。首先，它没有测试案例覆盖，这就意味着代码质量无法得到持续的保证；其次，它暴露的接口参数过多，有 11 个必填参数和 7 个选填参数：</p><pre><code>local memc_fetch, memc_store =
    shdict_memc.gen_memc_methods{
        tag = &quot;my memcached server tag&quot;,
        debug_logger = dlog,
        warn_logger = warn,
        error_logger = error_log,

        locks_shdict_name = &quot;some_lua_shared_dict_name&quot;,

        shdict_set = meta_shdict_set,  
        shdict_get = meta_shdict_get,  

        disable_shdict = false,  -- optional, default false

        memc_host = &quot;127.0.0.1&quot;,
        memc_port = 11211,
        memc_timeout = 200,  -- in ms
        memc_conn_pool_size = 5,
        memc_fetch_retries = 2,  -- optional, default 1
        memc_fetch_retry_delay = 100, -- in ms, optional, default to 100 (ms)

        memc_conn_max_idle_time = 10 * 1000,  -- in ms, for in-pool connections,optional, default to nil

        memc_store_retries = 2,  -- optional, default to 1
        memc_store_retry_delay = 100,  -- in ms, optional, default to 100 (ms)

        store_ttl = 1,  -- in seconds, optional, default to 0 (i.e., never expires)
    }
</code></pre><p>这其中暴露的绝大部分参数，其实可以通过“新建一个 memcached 的处理函数”的方式来简化。当前这种把所有参数一股脑儿地丢给用户来填写的封装方式并不友好，所以，我也很欢迎有兴趣的开发者，贡献 PR 来做这方面的优化。</p><p>另外，在这个封装库的文档中，其实也提到了进一步的优化方向：</p><ul>
<li>一是使用 <code>lua-resty-lrucache</code> ，来增加 worker 层的缓存，而不仅仅是 server 级别的 shared dict 缓存；</li>
<li>二是使用 <code>ngx.timer</code> ，来做异步的缓存更新操作。</li>
</ul><p>第一个方向其实是很不错的建议，因为 worker 内的缓存性能自然会更好；而第二个建议，就需要你根据自己的实际场景来考量了。不过，一般我并不推荐使用，这不仅是因为 timer 的数量是有限制的，而且如果这里的更新逻辑出错，就再也不会去更新缓存了，影响面比较大。</p><h2>lua-resty-mlcache</h2><p>接下来，我们再来介绍下，在 OpenResty 中被普遍使用的缓存封装： <code>lua-resty-mlcache</code>。它使用 shared dict 和 lua-resty-lrucache ，实现了多层缓存机制。我们下面就通过两段代码示例，来看看这个库如何使用：</p><pre><code>local mlcache = require &quot;resty.mlcache&quot;

local cache, err = mlcache.new(&quot;cache_name&quot;, &quot;cache_dict&quot;, {
    lru_size = 500,    -- size of the L1 (Lua VM) cache
    ttl = 3600,   -- 1h ttl for hits
    neg_ttl  = 30,     -- 30s ttl for misses
})
if not cache then
    error(&quot;failed to create mlcache: &quot; .. err)
end
</code></pre><p>先来看第一段代码。这段代码的开头引入了 mlcache 库，并设置了初始化的参数。我们一般会把这段代码放到 init 阶段，只需要做一次就可以了。</p><p>除了缓冲名和字典名这两个必填的参数外，第三个参数是一个字典，里面 12 个选项都是选填的，不填的话就使用默认值。这种方式显然就比 <code>lua-resty-memcached-shdict</code> 要优雅很多。其实，我们自己来设计接口的话，也最好采用 mlcache 这样的做法——让接口尽可能地简单，同时还保留足够的灵活性。</p><p>下面再来看第二段代码，这是请求处理时的逻辑代码：</p><pre><code>local function fetch_user(id)
    return db:query_user(id)
end

local id = 123
local user , err = cache:get(id , nil , fetch_user , id)
if err then
    ngx.log(ngx.ERR , &quot;failed to fetch user: &quot;, err)
    return
end

if user then
    print(user.id) -- 123
end
</code></pre><p>你可以看到，这里已经把多层缓存都给隐藏了，你只需要使用 mlcache 的对象去获取缓存，并同时设置好缓存失效后的回调函数就可以了。这背后复杂的逻辑，就可以被完全地隐藏了。</p><p>说到这里，你可能好奇，这个库内部究竟是怎么实现的呢？接下来，再让我们来看下这个库的架构和实现。下面这张图，来自 mlcache 的作者 thibault 在 2018 年 OpenResty 大会上演讲的幻灯片：</p><p><img src="https://static001.geekbang.org/resource/image/19/97/19a701636a95e931e6a9a8d0127e4f97.png?wh=1709*1121" alt=""></p><p>从图中你可以看到，mlcache 把数据分为了三层，即L1、L2和L3。</p><p>L1 缓存就是 lua-resty-lrucache。每一个 worker 中都有自己独立的一份，有 N 个 worker，就会有 N 份数据，自然也就存在数据冗余。由于在单 worker 内操作 lrucache 不会触发锁，所以它的性能更高，适合作为第一级缓存。</p><p>L2 缓存是 shared dict。所有的 worker 共用一份缓存数据，在 L1 缓存没有命中的情况下，就会来查询 L2 缓存。ngx.shared.DICT 提供的 API，使用了自旋锁来保证操作的原子性，所以这里我们并不用担心竞争的问题；</p><p>L3 则是在 L2 缓存也没有命中的情况下，需要执行回调函数去外部数据库等数据源查询后，再缓存到 L2 中。在这里，为了避免缓存风暴，它会使用 lua-resty-lock ，来保证只有一个 worker 去数据源获取数据。</p><p>整体而言，从请求的角度来看，</p><ul>
<li>首先会去查询 worker 内的 L1 缓存，如果L1命中就直接返回。</li>
<li>如果L1没有命中或者缓存失效，就会去查询 worker 间的 L2 缓存。如果L2命中就返回，并把结果缓存到 L1 中。</li>
<li>如果L2 也没有命中或者缓存失效，就会调用回调函数，从数据源中查到数据，并写入到 L2 缓存中，这也就是L3数据层的功能。</li>
</ul><p>从这个过程你也可以看出，缓存的更新是由终端请求来被动触发的。即使某个请求获取缓存失败了，后续的请求依然可以触发更新的逻辑，以便最大程度地保证缓存的安全性。</p><p>不过，虽然 mlcache 已经实现得比较完美了，但在现实使用中，其实还有一个痛点——数据的序列化和反序列化。这个其实并不是 mlcache 的问题，而是我们之前反复提到的  lrucache 和 shared dict 之间的差异造成的。在 lrucache 中，我们可以存储 Lua 的各种数据类型，包括 table；但 shared dict 中，我们只能存储字符串。</p><p>L1 也就是 lrucache 缓存，是用户真正接触到的那一层数据，我们自然希望在其中可以缓存各种数据，包括字符串、table、cdata 等。可是，问题在于， L2 中只能存储字符串。那么，当数据从 L2 提升到 L1 的时候，我们就需要做一层转换，也就是从字符串转成我们可以直接给用户的数据类型。</p><p>还好，mlcache 已经考虑到了这种情况，并在 <code>new</code> 和 <code>get</code> 接口中，提供了可选的函数 <code>l1_serializer</code>，专门用于处理 L2 提升到 L1 时对数据的处理。我们可以来看下面的示例代码，它是我从测试案例集中摘选出来的：</p><pre><code>local mlcache = require &quot;resty.mlcache&quot;

local cache, err = mlcache.new(&quot;my_mlcache&quot;, &quot;cache_shm&quot;, {
l1_serializer = function(i)
    return i + 2
end,
})

local function callback()
    return 123456
end

local data = assert(cache:get(&quot;number&quot;, nil, callback))
assert(data == 123458)
</code></pre><p>简单解释一下。在这个案例中，回调函数返回数字 123456；而在 <code>new</code> 中，我们设置的 <code>l1_serializer</code> 函数会在设置 L1 缓存前，把传入的数字加 2，也就是变成 123458。通过这样的序列化函数，数据在 L1 和 L2 之间转换的时候，就可以更加灵活了。</p><p>可以说，mlcache 是一个功能很强大的缓存库，而且<a href="https://github.com/thibaultcha/lua-resty-mlcache">文档</a>也写得非常详尽。今天这节课我只提到了核心的一些原理，更多的使用方法，建议你一定要自己花时间去探索和实践。</p><h2>写在最后</h2><p>有了多层缓存，服务端的性能才能得到最大限度的保证，而这中间又隐藏了很多的细节。这时候，有一个稳定、高效的封装库，我们就能省心很多。我也希望通过今天介绍的这两个封装库，帮助你更好地理解缓存。</p><p>最后给你留一个思考题，共享字典这一层缓存是必须的吗？如果只用 lrucache 是否可以呢？欢迎留言和我分享你的观点，也欢迎你把这篇文章分享出去，和更多的人一起交流和进步。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/e2/d8/f0562ede.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>manatee</span>
  </div>
  <div class="_2_QraFYR_0">想请问下老师多次提到的timer数量限制，这个限制具体是多少呢，从哪里可以查到相关资料呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: lua_max_pending_timers 和 lua_max_running_timers 这两个指令来做控制</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-28 08:03:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/47/cc/04a749e1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Shliesce</span>
  </div>
  <div class="_2_QraFYR_0">在实际的生产应用中，我认为 shared dict 这一层缓存是必须的，貌似大家都只记得lrucache的好，数据格式没限制，不需要反序列化，不需要根据 k&#47;v 体积算内存空间，worker 间独立不相互争抢，没有读写锁，性能高云云，但是却忘记了它最致命的一个弱点就是 lrucache 的生命周期是跟着 worker 走的，每当nginx reload 时，这部分缓存会全部丢失，这时候如果没有 shared dict，那 L3 的数据源分分钟被打挂，当然这是并发比较高的情况下，但是既然用到了缓存，说明业务体量肯定不会小。。不知道我的这个观点对吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 大部分情况下确实如你所说，共享字典在 reload 的时候不会丢失。也有一种特例，如果在 init 阶段或者init_worker阶段就能从 L3 主动获取到所有数据，那么只有 L1 其实也是可以接受的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-01 18:56:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/47/cc/04a749e1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Shliesce</span>
  </div>
  <div class="_2_QraFYR_0">接着我上面的这个观点，最近在设计“联动注册发现中心 动态更新 upstream 并缓存在进程内”的插件时，目前仅做了shared dict这一层的缓存，从初步的压测结果来看，单机 32C 的 cpu 平均使用率 30%可以达到 8w 的 qps，也不知道是不是错觉，性能比我们用 upsync 时还要好。。。<br>重点是在后续增加 lrucache 时，因为考虑到这种场景不存在key expired 的问题，主要是 value 变更比较频繁，且需要保证尽可能的数据一致性，像 mlcache 这种被动更新的逻辑就不太好实现，而且还要考虑到每次 reload 时的lrucache 全量 miss 的场景，最终评估下来我决定采用主动更新缓存的方案。同时因为 nginx 每次 reload 时都会加载 init_by_lua*阶段，所以我打算把主动更新的逻辑做到 init 阶段，借助 master fork worker 时继承内存空间的逻辑提前预热缓存；另外就是每次更新 shared dict 时同步更新 lrucache。暂时还不确定这个方案有没有什么坑，老师能帮忙提几条建议吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 没有明白为什么把主动更新的逻辑放在 init 阶段呢？<br>你可以在 init 阶段获取全量数据，通过长连接监听事件通知，去获取增量的数据。<br>这时候其实你可以只用 lru 缓存，去掉共享字典这个 L2 缓存。<br>APISIX（https:&#47;&#47;github.com&#47;iresty&#47;apisix）也是这么实现的，它的数据源在 etcd，通过 etcd 的 watch 来获取上游的变更。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-01 20:38:30</div>
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
  <div class="_2_QraFYR_0">共享字典这一层缓存不是必须的，但有这一层性能会高很多，因为lrucache 只能在单个worker中共享，而共享字典可以在整个server中共享，所以当一个worker中命中不了时，此时数据可能在另一个worker中存在，如果有server层的缓存，那命中率将大大提高。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-28 15:02:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>🆕</span>
  </div>
  <div class="_2_QraFYR_0">shared.DICT.get使用上下文限制了init*阶段，同样的还有resty.lock库中的部分api，在init阶段也可能会失效，所以想问一下mlcache 是否有同样的上下文使用限制？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-23 09:41:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c9/cf/af720d94.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>豆腐居士</span>
  </div>
  <div class="_2_QraFYR_0">老师好，使用http库访问网络的时候，dns解析这块怎么做 ？目前我的做法是在server里用resolver指定了多个dns IP地址！但是缓存失效后会出现解析域名大量解析超时的情况！tks</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-07 21:48:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/16/28/c11e7aae.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张仕华</span>
  </div>
  <div class="_2_QraFYR_0">多级缓存的更新类似cpu，必须有lrucache层的缓存失效信号，看了看文档也确实是这样。但其实还有个疑问，如何更新db中的数据然后让缓存失效呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-17 09:58:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/10/b7/d8ae07c8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>言身寸飞</span>
  </div>
  <div class="_2_QraFYR_0">我觉得shared dict是必须的。如果只有L1的话，量级大的、缓存经常变化的应用的话work多的话后面的数据会有连接压力吧。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-19 10:07:28</div>
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
  <div class="_2_QraFYR_0">共享字典这一层缓存并非必须，只用lrucache同样可以满足需求。只不过多加一层L2缓存（共享字典），可以使得服务器在L1缓存未命中的情况下，通过多个worker之间的共享数据，以减少读库的次数。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-28 22:41:09</div>
  </div>
</div>
</div>
</li>
</ul>