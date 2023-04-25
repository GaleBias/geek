<audio title="48 _ 微服务API网关搭建三步曲（二）" src="https://static001.geekbang.org/resource/audio/2d/1f/2dec6a2604e3b17d2fc506d67e4f671f.mp3" controls="controls"></audio> 
<p>你好，我是温铭。</p><p>在明白了微服务 API 网关的核心组件和抽象概念后，我们就要开始技术选型，并动手去实现它了。今天，我们就分别来看下，路由、插件、schema 和存储这四个核心组件的技术选型问题。</p><h2>存储</h2><p>上节课我提到过，存储是底层非常关键的基础组件，它会影响到配置如何同步、集群如何伸缩、高可用如何保证等核心的问题，所以，我们把它放在最开始的位置来选型。</p><p>我们先来看看，已有的 API 网关是把数据存储在哪里的。Kong 是把数据储存在PostgreSQL 或者 Cassandra 中，而同样基于 OpenResty 的 Orange，则是存储在 MySQL 中。不过，这种选择还是有很多缺陷的。</p><p>第一，储存需要单独做高可用方案。PostgreSQL、MySQL 数据库虽然有自己的高可用方案，但你还需要 DBA 和机器资源，在发生故障时也很难做到快速切换。</p><p>第二，只能轮询数据库来获取配置变更，无法做到推送。这不仅会增加数据库资源的消耗，同时变更的实时性也会大打折扣。</p><p>第三，需要自己维护历史版本，并考虑回退和升级。如果用户发布了一个变更，后续可能会有回滚操作，这时候你就需要在代码层面，自己做两个版本之间的 diff，以便配置的回滚。同时，在系统自身升级的时候，还可能会修改数据库的表结构，所以代码层面就需要考虑到新旧版本的兼容和数据升级。</p><!-- [[[read_end]]] --><p>第四，提高了代码的复杂度。在实现网关的功能之外，你还需要为了前面 3 个缺陷，在代码层面去打上补丁，这显然会让代码的可读性降低不少。</p><p>第五，增加了部署和运维的难度。部署和维护一个关系型数据库并不是一件简单的事情，如果是一个数据库集群那就更加复杂了，并且我们也无法做到快速扩容和缩容。</p><p>针对这样的情况，我们应该如何选择呢？</p><p>我们不妨回到 API 网关的原始需求上来，这里存储的都是简单的配置信息，uri、插件参数、上游地址等，并没有涉及到复杂的联表操作，也不需要严格的事务保证。显然，这种情况下使用关系型数据库，可不就是“杀鸡焉用宰牛刀”吗？</p><p>事实上，本着最小化够用并且更贴近 K8s 的原则，etcd 就是一个恰到好处的选型了：</p><ul>
<li>API 网关的配置数据每秒钟的变化次数不会很多，etcd 在性能上是足够的；</li>
<li>集群和动态伸缩方面，更是 etcd 天生的优势；</li>
<li>etcd还具备 watch 的接口，不用轮询去获取变更。</li>
</ul><p>其实还有一点，可以让我们更加放心地选择 etcd——它已经是 K8s 体系中保存配置的默认选型了，显然已经经过了很多比 API 网关更加复杂的场景的验证。</p><h2>路由</h2><p>路由也是非常重要的技术选型，所有的请求都由路由筛选出需要加载的插件列表，逐个运行后，再转发给指定的上游。不过，考虑到路由规则可能会比较多，所以路由这里的技术选型，我们需要着重从算法的时间复杂度上去考量。</p><p>我们先来看下，在 OpenResty 下有哪些现成的路由可以拿来使用。老规矩，让我们在 <code>awesome-resty</code> 的项目中逐个查找一遍，这其中就有专门的 <code>Routing Libraries</code>：</p><pre><code>    •    lua-resty-route — A URL routing library for OpenResty supporting multiple route matchers, middleware, and HTTP and WebSockets handlers to mention a few of its features
    •    router.lua — A barebones router for Lua, it matches URLs and executes Lua functions
    •    lua-resty-r3 — libr3 OpenResty implementation, libr3 is a high-performance path dispatching library. It compiles your route paths into a prefix tree (trie). By using the constructed prefix trie in the start-up time, you may dispatch your routes with efficiency
    •    lua-resty-libr3 — High-performance path dispatching library base on libr3 for OpenResty
</code></pre><p>你可以看到，这里面包含了四个路由库的实现。前面两个路由都是纯 Lua 实现，相对比较简单，所以有不少功能的欠缺，还不能达到生成的要求。</p><p>后面两个库，其实都是基于 libr3 这个 C 库，并使用 FFI 的方式做了一层封装，而 libr3 自身使用的是前缀树。这种算法和存储了多少条规则的数目 N 无关，只和匹配数据的长度 K 有关，所以时间复杂度为 O(K)。</p><p>但是， libr3 也是有缺点的，它的匹配规则和我们熟悉的 Nginx location 的规则不同，而且不支持回调。这样，我们就没有办法根据请求头、cookie、Nginx 变量来设置路由的条件，对于 API 网关的场景来说显然不够灵活。</p><p>不过，虽说我们尝试从 <code>awesome-resty</code> 中找到可用路由库的努力没有成功，但 libr3 的实现，还是给我们指引了一个新的方向：用 C 来实现前缀树以及 FFI 封装，这样应该可以接近时间复杂度和代码性能上的最优方案。</p><p>正好， Redis 的作者开源了一个基数树，也就是压缩前缀树的 <a href="https://github.com/antirez/rax">C 实现</a>。顺藤摸瓜，我们还可以找到 rax 在 OpenResty 中可用的 <a href="https://github.com/iresty/lua-resty-radixtree">FFI 封装库</a>，它的示例代码如下：</p><pre><code>local radix = require(&quot;resty.radixtree&quot;)
local rx = radix.new({
    {
        path = &quot;/aa&quot;,
        host = &quot;foo.com&quot;,
        method = {&quot;GET&quot;, &quot;POST&quot;},
        remote_addr = &quot;127.0.0.1&quot;,
    },
    {
        path = &quot;/bb*&quot;,
        host = {&quot;*.bar.com&quot;, &quot;gloo.com&quot;},
        method = {&quot;GET&quot;, &quot;POST&quot;, &quot;PUT&quot;},
        remote_addr = &quot;fe80:fe80::/64&quot;,
        vars = {&quot;arg_name&quot;, &quot;jack&quot;},
    }
})

ngx.say(rx:match(&quot;/aa&quot;, {host = &quot;foo.com&quot;,
                     method = &quot;GET&quot;,
                     remote_addr = &quot;127.0.0.1&quot;
                    }))
</code></pre><p>从中你也可以看出， <code>lua-resty-radixtree</code> 支持根据 uri、host、http method、http header、Nginx 变量、IP 地址等多个维度，作为路由查找的条件；同时，基数树的时间复杂度为 O(K)，性能远比现有 API 网关常用的“遍历+hash 缓存”的方式，来得更为高效。</p><h2>schema</h2><p>schema 的选择其实要容易得多，我们在前面介绍过的 <code>lua-rapidjson</code> ，就是非常好的一个选择。这部分你完全没有必要自己去写一个，json schema 已经足够强大了。下面就是一个简单的示例：</p><pre><code>local schema = {
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
</code></pre><h2>插件</h2><p>有了上面存储、路由和 schema 的基础，上层的插件应该如何实现，其实就清晰多了。插件并没有现成的开源库可以使用，需要我们自己来实现。插件在设计的时候，主要有三个方面需要我们考虑清楚。</p><p>首先是如何挂载。我们希望插件可以挂载到 <code>rewrite</code>、<code>access</code>、<code>header filer</code>、<code>body filter</code> 和 <code>log</code>阶段，甚至在 <code>balancer</code> 阶段也可以设置自己的负载均衡算法。所以，我们应该在 Nginx 的配置文件中暴露这些阶段，并在对插件的实现中预留好接口。</p><p>其次是如何获取配置的变更。由于没有关系型数据库的束缚，插件参数的变更可以通过 etcd 的 watch 来实现，这会让整体框架的代码逻辑变得更加明了易懂。</p><p>最后是插件的优先级。具体来说，比如，身份认证和限流限速的插件，应该先执行哪一个呢？绑定在 route 和绑定在 service 上的插件发生冲突时，又应该以哪一个为准呢？这些都是我们需要考虑到位的。</p><p>在梳理清楚插件的这三个问题后，我们就可以得到插件内部的一个流程图了：</p><p><img src="https://static001.geekbang.org/resource/image/d1/13/d18243966a4973ff8409dd45bf83dc13.png?wh=974*1182" alt=""></p><h2>架构</h2><p>自然，当微服务 API 网关的这些关键组件都确定了之后，用户请求的处理流程，也就随之尘埃落定了。这里我画了一张图来表示这个流程：</p><p><img src="https://static001.geekbang.org/resource/image/7f/89/7f2b50689a86d382a4c9340b4edb9489.png?wh=2843*1674" alt=""></p><p>从这个图中我们可以看出，当一个用户请求进入 API 网关时，</p><ul>
<li>首先，会根据请求的方法、uri、host、请求头等条件，去路由规则中进行匹配。如果命中了某条路由规则，就会从 etcd 中获取对应的插件列表。</li>
<li>然后，和本地开启的插件列表进行交集，得到最终可以运行的插件列表。</li>
<li>再接着，根据插件的优先级，逐个运行插件。</li>
<li>最后，根据上游的健康检查和负载均衡算法，把这个请求发送给上游。</li>
</ul><p>当架构设计完成后，我们就胸有成竹，可以去编写具体的代码了。这其实就像盖房子一样，只有在你拥有设计的蓝图和坚实的地基之后，才能去做砖瓦堆砌的具体工作。</p><h2>写在最后</h2><p>其实，通过这两节课的学习，我们已经做好了产品定位和技术选型这两件最重要的事情，它们都比具体的编码实现更为关键，也希望你可以更用心地去考虑和选择。</p><p>那么，在你的实际工作中，你是否使用过 API 网关呢？你们公司又是如何做 API 网关的选型的呢？欢迎留言和我分享你的经历和收获，也欢迎你把这篇文章分享出去，和更多的人一起交流、进步。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/8d/c5/898b13b4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>亢（知行合一的路上）</span>
  </div>
  <div class="_2_QraFYR_0">老师提到 etcd 作为配置信息的存储，并提供 watch 的功能，这个和消息队列有点类似了，而且是支持持久化的消息队列，如果数据量小的时候，是不是也可以作为 MQ 使用呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-27 20:51:59</div>
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
  <div class="_2_QraFYR_0">没有使用过API网关，所以特别买这个课学习一下。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-13 19:15:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/48/8f/7ecd4eed.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>FF</span>
  </div>
  <div class="_2_QraFYR_0">关于 lua 代码的编译和执行，请个温老师几个问题。<br><br>问题一。lua 模块中，用 local 关键字定义在**函数外部**的变量，每个 worker 进程只对其在第一次外部请求时进行初始化吗 ？我验证的时候发现有时输出初始值，有时输出改变后的新值，怀疑是不同 worker 进程的输出 ？有点疑惑在此跟你确认下。<br><br><br>问题二。对于非函数的代码块呢？对同个 worker 进程来讲，每次请求都会执行这种代码吗 ？还是仅第一次请求时执行 ？验证时同样结果不是很确定，有时执行，有时又不执行。困惑<br><br>类似 lua 代码编译机制，代码在编译器中缓存刷新机制，温老师有无资料分享下 ？我翻了 lua 的官网没找到类似这方面的介绍。<br><br>感谢。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 前面两个问题，是否可以提供下具体的示例代码呢？只看文字并不直观。<br>关于 Lua 编译器的细节，我推荐《自己动手实现 Lua 虚拟机、编译器和标准库》这本书，里面很详细的介绍 Lua 虚拟机和编译器的实现。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-13 18:02:09</div>
  </div>
</div>
</div>
</li>
</ul>