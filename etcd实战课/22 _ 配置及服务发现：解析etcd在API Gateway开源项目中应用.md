<audio title="22 _ 配置及服务发现：解析etcd在API Gateway开源项目中应用" src="https://static001.geekbang.org/resource/audio/0a/5c/0a5a0df51e06568f94d7520ceb6d225c.mp3" controls="controls"></audio> 
<p>你好，我是唐聪。</p><p>在软件开发的过程中，为了提升代码的灵活性和开发效率，我们大量使用配置去控制程序的运行行为。</p><p>从简单的数据库账号密码配置，到<a href="https://github.com/kelseyhightower/confd">confd</a>支持以etcd为后端存储的本地配置及模板管理，再到<a href="https://github.com/apache/apisix">Apache APISIX</a>等API Gateway项目使用etcd存储服务配置、路由信息等，最后到Kubernetes更实现了Secret和ConfigMap资源对象来解决配置管理的问题。</p><p>那么它们是如何实现实时、动态调整服务配置而不需要重启相关服务的呢？</p><p>今天我就和你聊聊etcd在配置和服务发现场景中的应用。我将以开源项目Apache APISIX为例，为你分析服务发现的原理，带你了解etcd的key-value模型，Watch机制，鉴权机制，Lease特性，事务特性在其中的应用。</p><p>希望通过这节课，让你了解etcd在配置系统和服务发现场景工作原理，帮助你选型适合业务场景的配置系统、服务发现组件。同时，在使用Apache APISIX等开源项目过程中遇到etcd相关问题时，你能独立排查、分析，并向社区提交issue和PR解决。</p><h2>服务发现</h2><p>首先和你聊聊服务发现，服务发现是指什么？为什么需要它呢?</p><p>为了搞懂这个问题，我首先和你分享下程序部署架构的演进。</p><!-- [[[read_end]]] --><h3>单体架构</h3><p>在早期软件开发时使用的是单体架构，也就是所有功能耦合在同一个项目中，统一构建、测试、发布。单体架构在项目刚启动的时候，架构简单、开发效率高，比较容易部署、测试。但是随着项目不断增大，它具有若干缺点，比如：</p><ul>
<li>所有功能耦合在同一个项目中，修复一个小Bug就需要发布整个大工程项目，增大引入问题风险。同时随着开发人员增多、单体项目的代码增长、各模块堆砌在一起、代码质量参差不齐，内部复杂度会越来越高，可维护性差。</li>
<li>无法按需针对仅出现瓶颈的功能模块进行弹性扩容，只能作为一个整体继续扩展，因此扩展性较差。</li>
<li>一旦单体应用宕机，将导致所有服务不可用，因此可用性较差。</li>
</ul><h3>分布式及微服务架构</h3><p>如何解决以上痛点呢？</p><p>当然是将单体应用进行拆分，大而化小。如何拆分呢？ 这里我就以一个我曾经参与重构建设的电商系统为案例给你分析一下。在一个单体架构中，完整的电商系统应包括如下模块：</p><ul>
<li>商城系统，负责用户登录、查看及搜索商品、购物车商品管理、优惠券管理、订单管理、支付等功能。</li>
<li>物流及仓储系统，根据用户订单，进行发货、退货、换货等一系列仓储、物流管理。</li>
<li>其他客服系统、客户管理系统等。</li>
</ul><p>因此在分布式架构中，你可以按整体功能，将单体应用垂直拆分成以上三大功能模块，各个功能模块可以选择不同的技术栈实现，按需弹性扩缩容，如下图所示。</p><p><img src="https://static001.geekbang.org/resource/image/ca/20/ca6090e229dde9a0361d6yy2c3df8d20.png?wh=1920*1046" alt=""></p><p>那什么又是微服务架构呢？</p><p>它是对各个功能模块进行更细立度的拆分，比如商城系统模块可以拆分成：</p><ul>
<li>用户鉴权模块；</li>
<li>商品模块；</li>
<li>购物车模块；</li>
<li>优惠券模块；</li>
<li>支付模块；</li>
<li>……</li>
</ul><p>在微服务架构中，每个模块职责更单一、独立部署、开发迭代快，如下图所示。</p><p><img src="https://static001.geekbang.org/resource/image/cf/4a/cf62b7704446c05d8747b4672b5fb74a.png?wh=1920*1048" alt=""></p><p>那么在分布式及微服务架构中，各个模块之间如何及时知道对方网络地址与端口、协议，进行接口调用呢？</p><h3>为什么需要服务发现中间件?</h3><p>其实这个知道的过程，就是服务发现。在早期的时候我们往往通过硬编码、配置文件声明各个依赖模块的网络地址、端口，然而这种方式在分布式及微服务架构中，其运维效率、服务可用性是远远不够的。</p><p>那么我们能否实现通过一个特殊服务就查询到各个服务的后端部署地址呢？ 各服务启动的时候，就自动将IP和Port、协议等信息注册到特殊服务上，当某服务出现异常的时候，特殊服务就自动删除异常实例信息？</p><p>是的，当然可以，这个特殊服务就是注册中心服务，你可以基于etcd、ZooKeeper、consul等实现。</p><h3>etcd服务发现原理</h3><p>那么如何基于etcd实现服务发现呢?</p><p>下面我给出了一个通用的服务发现原理架构图，通过此图，为你介绍下服务发现的基本原理。详细如下：</p><ul>
<li>整体上分为四层，client层、proxy层(可选)、业务server、etcd存储层组成。引入proxy层的原因是使client更轻、逻辑更简单，无需直接访问存储层，同时可通过proxy层支持各种协议。</li>
<li>client层通过负载均衡访问proxy组件。proxy组件启动的时候，通过etcd的Range RPC方法从etcd读取初始化服务配置数据，随后通过Watch接口持续监听后端业务server扩缩容变化，实时修改路由。</li>
<li>proxy组件收到client的请求后，它根据从etcd读取到的对应服务的路由配置、负载均衡算法（比如Round-robin）转发到对应的业务server。</li>
<li>业务server启动的时候，通过etcd的写接口Txn/Put等，注册自身地址信息、协议到高可用的etcd集群上。业务server缩容、故障时，对应的key应能自动从etcd集群删除，因此相关key需要关联lease信息，设置一个合理的TTL，并定时发送keepalive请求给Leader续租，以防止租约及key被淘汰。</li>
</ul><p><img src="https://static001.geekbang.org/resource/image/26/e4/26d0d18c0725de278eeb7505f20642e4.png?wh=1920*1137" alt=""></p><p>当然，在分布式及微服务架构中，我们面对的问题不仅仅是服务发现，还包括如下痛点：</p><ul>
<li>限速；</li>
<li>鉴权；</li>
<li>安全；</li>
<li>日志；</li>
<li>监控；</li>
<li>丰富的发布策略；</li>
<li>链路追踪；</li>
<li>......</li>
</ul><p>为了解决以上痛点，各大公司及社区开发者推出了大量的开源项目。这里我就以国内开发者广泛使用的Apache APISIX项目为例，为你分析etcd在其中的应用，了解下它是怎么玩转服务发现的。</p><h3>Apache APISIX原理</h3><p>Apache APISIX它具备哪些功能呢？</p><p>它的本质是一个无状态、高性能、实时、动态、可水平扩展的API网关。核心原理就是基于你配置的服务信息、路由规则等信息，将收到的请求通过一系列规则后，正确转发给后端的服务。</p><p>Apache APISIX其实就是上面服务发现原理架构图中的proxy组件，如下图红色虚线框所示。</p><p><img src="https://static001.geekbang.org/resource/image/20/fd/20a539bdd37db2d4632c7b0c5f4119fd.png?wh=1920*1246" alt=""></p><p>Apache APISIX详细架构图如下（<a href="https://github.com/apache/apisix">引用自社区项目文档</a>）。从图中你可以看到，它由控制面和数据面组成。</p><p>控制面顾名思义，就是你通过Admin API下发服务、路由、安全配置的操作。控制面默认的服务发现存储是etcd，当然也支持consul、nacos等。</p><p>你如果没有使用过Apache APISIX的话，可以参考下这个<a href="https://github.com/apache/apisix-docker/tree/master/example">example</a>，快速、直观的了解下Apache APISIX是如何通过Admin API下发服务和路由配置的。</p><p>数据面是在实现基于服务路由信息数据转发的基础上，提供了限速、鉴权、安全、日志等一系列功能，也就是解决了我们上面提的分布式及微服务架构中的典型痛点。</p><p><img src="https://static001.geekbang.org/resource/image/83/f4/834502c6ed7e59fe0b4643c11b2d31f4.png?wh=1746*838" alt=""></p><p>那么当我们通过控制面API新增一个服务时，Apache APISIX是是如何实现实时、动态调整服务配置，而不需要重启网关服务的呢？</p><p>下面，我就和你聊聊etcd在Apache APISIX项目中的应用。</p><h3>etcd在Apache APISIX中的应用</h3><p>在搞懂这个问题之前，我们先看看Apache APISIX在etcd中，都存储了哪些数据呢？它的数据存储格式是怎样的？</p><h4>数据存储格式</h4><p>下面我参考Apache APISIX的<a href="https://github.com/apache/apisix-docker/tree/master/example">example</a>案例（apisix:2.3），通过Admin API新增了两个服务、路由规则后，执行如下查看etcd所有key的命令：</p><pre><code>etcdctl get &quot;&quot; --prefix --keys-only
</code></pre><p>etcd输出结果如下：</p><pre><code>/apisix/consumers/
/apisix/data_plane/server_info/f7285805-73e9-4ce4-acc6-a38d619afdc3
/apisix/global_rules/
/apisix/node_status/
/apisix/plugin_metadata/
/apisix/plugins
/apisix/plugins/
/apisix/proto/
/apisix/routes/
/apisix/routes/12
/apisix/routes/22
/apisix/services/
/apisix/services/1
/apisix/services/2
/apisix/ssl/
/apisix/ssl/1
/apisix/ssl/2
/apisix/stream_routes/
/apisix/upstreams/
</code></pre><p>然后我们继续通过etcdctl get命令查看下services都存储了哪些信息呢？</p><pre><code>root@e9d3b477ca1f:/opt/bitnami/etcd# etcdctl get /apisix/services --prefix
/apisix/services/
init_dir
/apisix/services/1
{&quot;update_time&quot;:1614293352,&quot;create_time&quot;:1614293352,&quot;upstream&quot;:{&quot;type&quot;:&quot;roundrobin&quot;,&quot;nodes&quot;:{&quot;172.18.5.12:80&quot;:1},&quot;hash_on&quot;:&quot;vars&quot;,&quot;scheme&quot;:&quot;http&quot;,&quot;pass_host&quot;:&quot;pass&quot;},&quot;id&quot;:&quot;1&quot;}
/apisix/services/2
{&quot;update_time&quot;:1614293361,&quot;create_time&quot;:1614293361,&quot;upstream&quot;:
{&quot;type&quot;:&quot;roundrobin&quot;,&quot;nodes&quot;:{&quot;172.18.5.13:80&quot;:1},&quot;hash_on&quot;:&quot;vars&quot;,&quot;scheme&quot;:&quot;http&quot;,&quot;pass_host&quot;:&quot;pass&quot;},&quot;id&quot;:&quot;2&quot;}
</code></pre><p>从中我们可以总结出如下信息：</p><ul>
<li>Apache APSIX 2.x系列版本使用的是etcd3。</li>
<li>服务、路由、ssl、插件等配置存储格式前缀是/apisix + "/" + 功能特性类型（routes/services/ssl等），我们通过Admin API添加的路由、服务等配置就保存在相应的前缀下。</li>
<li>路由和服务配置的value是个Json对象，其中服务对象包含了id、负载均衡算法、后端节点、协议等信息。</li>
</ul><p>了解完Apache APISIX在etcd中的数据存储格式后，那么它是如何动态、近乎实时地感知到服务配置变化的呢？</p><h4>Watch机制的应用</h4><p>与Kubernetes一样，它们都是通过etcd的<strong>Watch机制</strong>来实现的。</p><p>Apache APISIX在启动的时候，首先会通过Range操作获取网关的配置、路由等信息，随后就通过Watch机制，获取增量变化事件。</p><p>使用Watch机制最容易犯错的地方是什么呢？</p><p>答案是不处理Watch返回的相关错误信息，比如已压缩ErrCompacted错误。Apache APISIX项目在从etcd v2中切换到etcd v3早期的时候，同样也犯了这个错误。</p><p>去年某日收到小伙伴求助，说使用Apache APISIX后，获取不到新的服务配置了，是不是etcd出什么Bug了？</p><p>经过一番交流和查看日志，发现原来是Apache APISIX未处理ErrCompacted错误导致的。根据我们<a href="https://time.geekbang.org/column/article/340226">07</a>Watch原理的介绍，当你请求Watch的版本号已被etcd压缩后，etcd就会取消这个watcher，这时你需要重建watcher，才能继续监听到最新数据变化事件。</p><p>查清楚问题后，小伙伴向社区提交了issue反馈，随后Apache APISIX相关同学通过<a href="https://github.com/apache/apisix/pull/2687">PR 2687</a>修复了此问题，更多信息你可参考Apache APISIX访问etcd<a href="https://github.com/apache/apisix/blob/v2.3/apisix/core/etcd.lua">相关实现代码文件</a>。</p><h4>鉴权机制的应用</h4><p>除了Watch机制，Apache APISIX项目还使用了鉴权，毕竟配置网关是个高危操作，那它是如何使用etcd鉴权机制的呢？ <strong>etcd鉴权机制</strong>中最容易踩的坑是什么呢？</p><p>答案是不复用client和鉴权token，频繁发起Authenticate操作，导致etcd高负载。正如我在<a href="https://time.geekbang.org/column/article/346471">17</a>和你介绍的，一个8核32G的高配节点在100个连接时，Authenticate QPS仅为8。可想而知，你如果不复用token，那么出问题就很自然不过了。</p><p>Apache APISIX是否也踩了这个坑呢？</p><p>Apache APISIX是基于Lua构建的，使用的是<a href="https://github.com/api7/lua-resty-etcd/blob/master/lib/resty/etcd/v3.lua">lua-resty-etcd</a>这个项目访问etcd，从相关<a href="https://github.com/apache/apisix/issues/2899">issue</a>反馈看，的确也踩了这个坑。社区用户反馈后，随后通过复用client、更完善的token复用机制解决了Authenticate的性能瓶颈，详细信息你可参考<a href="https://github.com/apache/apisix/pull/2932">PR 2932</a>、<a href="https://github.com/api7/lua-resty-etcd/pull/100">PR 100</a>。</p><p>除了以上介绍的Watch机制、鉴权机制，Apache APISIX还使用了etcd的Lease特性和事务接口。</p><h4>Lease特性的应用</h4><p>为什么Apache APISIX项目需要Lease特性呢？</p><p>服务发现的核心工作原理是服务启动的时候将地址信息登录到注册中心，服务异常时自动从注册中心删除。</p><p>这是不是跟我们前面<a href="https://time.geekbang.org/column/article/338524">05</a>节介绍的&lt;Lease特性: 如何检测客户端的存活性&gt;应用场景很匹配呢？</p><p>没错，Apache APISIX通过etcd v2的TTL特性、etcd v3的Lease特性来实现类似的效果，它提供的增加服务路由API，支持设置TTL属性，如下面所示：</p><pre><code># Create a route expires after 60 seconds, then it's deleted automatically
$ curl http://127.0.0.1:9080/apisix/admin/routes/2?ttl=60 -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -i -d '
{
    &quot;uri&quot;: &quot;/aa/index.html&quot;,
    &quot;upstream&quot;: {
        &quot;type&quot;: &quot;roundrobin&quot;,
        &quot;nodes&quot;: {
            &quot;39.97.63.215:80&quot;: 1
        }
    }
}'
</code></pre><p>当一个路由设置非0 TTL后，Apache APISIX就会为它创建Lease，关联key，相关代码如下：</p><pre><code>-- lease substitute ttl in v3
local res, err
if ttl then
    local data, grant_err = etcd_cli:grant(tonumber(ttl))
    if not data then
        return nil, grant_err
    end
    res, err = etcd_cli:set(prefix .. key, value, {prev_kv = true, lease = data.body.ID})
else
    res, err = etcd_cli:set(prefix .. key, value, {prev_kv = true})
end
</code></pre><h4>事务特性的应用</h4><p>介绍完Lease特性在Apache APISIX项目中的应用后，我们再来思考两个问题。为什么它还依赖etcd的事务特性呢？简单的执行put接口有什么问题？</p><p>答案是它跟Kubernetes是一样的使用目的。使用事务是为了防止并发场景下的数据写冲突，比如你可能同时发起两个Patch Admin API去修改配置等。如果简单地使用put接口，就会导致第一个写请求的结果被覆盖。</p><p>Apache APISIX是如何使用事务接口提供的乐观锁机制去解决并发冲突的问题呢？</p><p>核心依然是我们前面课程中一直强调的mod_revision，它会比较事务提交时的mod_revision与预期是否一致，一致才能执行put操作，Apache APISIX相关使用代码如下：</p><pre><code>local compare = {
    {
        key = key,
        target = &quot;MOD&quot;,
        result = &quot;EQUAL&quot;,
        mod_revision = mod_revision,
    }
}
local success = {
    {
        requestPut = {
            key = key,
            value = value,
            lease = lease_id,
        }
    }
}
local res, err = etcd_cli:txn(compare, success)
if not res then
    return nil, err
end
</code></pre><p>关于Apache APISIX事务特性的引入、背景以及更详细的实现，你也可以参考<a href="https://github.com/apache/apisix/pull/2216">PR 2216</a>。</p><h2>小结</h2><p>最后我们来小结下今天的内容。今天我给你介绍了服务部署架构的演进，我们从单体架构的缺陷开始、到分布式及微服务架构的诞生，和你分享了分布式及微服务架构中面临的一系列痛点（如服务发现，鉴权，安全，限速等等）。</p><p>而开源项目Apache APISIX正是一个基于etcd的项目，它为后端存储提供了一系列的解决方案，我通过它的架构图为你介绍了其控制面和数据面的工作原理。</p><p>随后我从数据存储格式、Watch机制、鉴权机制、Lease特性以及事务特性维度，和你分析了它们在Apache APISIX项目中的应用。</p><p>数据存储格式上，APISIX采用典型的prefix + 功能特性组织格式。key是相关配置id，value是个json对象，包含一系列业务所需要的核心数据。你需要注意的是Apache APISIX 1.x版本使用的etcd v2 API，2.x版本使用的是etcd v3 API，要求至少是etcd v3.4版本以上。</p><p>Watch机制上，APISIX依赖它进行配置的动态、实时更新，避免了传统的修改配置，需要服务重启等缺陷。</p><p>鉴权机制上，APISIX使用密码认证，进行多租户认证、授权，防止用户出现越权访问，保护网关服务的安全。</p><p>Lease及事务特性上，APISIX通过Lease来设置自动过期的路由规则，解决服务发现中的节点异常自动剔除等问题，通过事务特性的乐观锁机制来实现并发场景下覆盖更新等问题。</p><p>希望通过本节课的学习，让你从etcd角度更深入了解APISIX项目的原理，了解etcd各个特性在其中的应用，学习它的最佳实践经验和经历的各种坑，避免重复踩坑。在以后的工作中，在你使用APISIX等开源项目遇到etcd相关错误时，能独立分析、排查，甚至给社区提交PR解决。</p><h2>思考题</h2><p>好了，这节课到这里也就结束了，最后我给你留了一个开放的配置系统设计思考题。</p><p>假设老板让你去设计一个大型配置系统，满足公司各个业务场景的诉求，期望的设计目标如下：</p><ul>
<li>高可靠。配置系统的作为核心基础设施，期望可用性能达到99.99%。</li>
<li>高性能。公司业务多，规模大，配置系统应具备高性能、并能水平扩容。</li>
<li>支持多业务、多版本管理、多种发布策略。</li>
</ul><p>你认为etcd适合此业务场景吗？如果适合，分享下你的核心想法、整体架构，如果不适合，你心目中的理想存储和架构又是怎样的呢？</p><p>欢迎大家留言一起讨论，后面我也将在答疑篇中分享我的一些想法和曾经大规模TO C业务中的实践经验。</p><p>感谢你阅读，也欢迎你把这篇文章分享给更多的朋友一起阅读。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/8f/cf/890f82d6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>那时刻</span>
  </div>
  <div class="_2_QraFYR_0">老师的问题，我的个人看法是，采用etcd应该是可行的。<br><br>1.高可靠。etcd基于raft的多副本可以满足。<br>2.高性能。公司业务多，规模大，可以依据不同业务不同etcd的方法，分担etcd的写压力，以及数据存储量有限的问题。各自业务的etcd可以水平扩展。<br>3.支持多业务、多版本管理、多种发布策略。etcd可以做到多版本管理，多发布策略的话，可以级联多个etcd的方法。<br><br>另外，可能更加理想的存储架构方式是采用计算与存储分离的方法，计算部分处理读写以及扩展，存储部分处理多版本，多业务，多发布策略。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，整体想法不错，特别是考虑到数据容量问题，但是还需要考虑更多问题，比如性能问题，一个业务可能数万个client, 直接读etcd性能肯定扛不住的，其次是复杂度管理，比如一个业务分配一个etcd集群，那这过程是手动的还是自动的呢? 能否自动快速就接入一个新业务而无需相关运维操作呢？ 或者有没有更好的存储方案。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-12 11:28:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_daf51a</span>
  </div>
  <div class="_2_QraFYR_0">思考题很棒，我认为etcd并不合适，适合使用可平行扩容的分布式数据库如tidb，运维复杂度不更低点吗，容量也更大，还能支持各种key value大小配置</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-12 18:39:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/40/06/bb31933b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>大爱无疆</span>
  </div>
  <div class="_2_QraFYR_0">做一个etcd proxy，proxy 实现 hash，将不同的&#47;key  映射到后端不同的etcd集群上。<br>这样 相当于实现了 支持分片的 分布式存储。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯啊，这个想法在有的公司落地了，但是不支持跨多节点的事务，有一定局限性，proxy稳定性也是个考验</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-09 13:37:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_ed84ef</span>
  </div>
  <div class="_2_QraFYR_0">给服务路由配Lease跟APISIX提供的健康检查插件有什么区别吗?好像都是服务异常后可自动摘除节点</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-05 18:52:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/41/64/cf13c011.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>刁寿钧</span>
  </div>
  <div class="_2_QraFYR_0">其实还期待讨论下APISIX搭配证书使用etcd的姿势，哈哈</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-15 00:06:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/22/3b/46/3701e908.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Coder</span>
  </div>
  <div class="_2_QraFYR_0">我认为使用支持分片的nosql数据库比较合适，比如redis集群版本，一主两副本、还是很可靠得</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-13 12:51:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/22/3a/de/e5c30589.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>云原生工程师</span>
  </div>
  <div class="_2_QraFYR_0">我认为etcd不太合适，应该使用类似阿里云drds、腾讯云tdsql这样分布式数据库，不过数据推送机制就没了，需要轮询了，但是容量不需要担心，支持多业务就表中增加一个字段或独立的表</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-13 00:47:59</div>
  </div>
</div>
</div>
</li>
</ul>