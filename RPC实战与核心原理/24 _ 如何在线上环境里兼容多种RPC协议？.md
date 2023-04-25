<audio title="24 _ 如何在线上环境里兼容多种RPC协议？" src="https://static001.geekbang.org/resource/audio/bb/ee/bbd08da839ab498ade98796d2e8f63ee.mp3" controls="controls"></audio> 
<p>你好，我是何小锋。上一讲我们学习了如何在没有接口的情况下完成RPC调用，其关键在于你要理解接口定义在RPC里面的作用。除了我们前面说的，动态代理生成的过程中需要用到接口定义，剩余的其它过程中接口的定义只是被当作元数据来使用，而动态代理在RPC中并不是一个必须的环节，所以在没有接口定义的情况下我们同样也是可以完成RPC调用的。</p><p>回顾完上一讲的重点，咱们就言归正传，切入今天的主题，一起看看如何在线上环境里兼容多种RPC协议。</p><p>看到这个问题后，可能你的第一反应就是，在真实环境中为什么会存在多个协议呢？我们说过，RPC是能够帮助我们屏蔽网络编程细节，实现调用远程方法就跟调用本地一样的体验。大白话说就是，RPC是能够帮助我们在开发过程中完成应用之间的通信，而又不需要我们关心具体通信细节的工具。</p><h2>为什么要支持多协议？</h2><p>既然应用之间的通信都是通过RPC来完成的，而能够完成RPC通信的工具有很多，比如像Web Service、Hessian、gRPC等都可以用来充当RPC使用。这些不同的RPC框架都是随着互联网技术的发展而慢慢涌现出来的，而这些RPC框架可能在不同时期会被我们引入到不同的项目中解决当时应用之间的通信问题，这样就导致我们线上的生成环境中存在各种各样的RPC框架。</p><!-- [[[read_end]]] --><p>很显然，这种混乱使用RPC框架的方式肯定不利于公司技术栈的管理，最明显的一个特点就是我们维护RPC框架的成本越来越高，因为每种RPC框架都需要有专人去负责升级维护。</p><p>为了解决早期遗留的一些技术负债，我们通常会去选择更高级的、更好用的工具来解决，治理RPC框架混乱的问题也是一样。为了解决同时维护多个RPC框架的困难，我们肯定希望能够用统一用一种RPC框架来替代线上所有的RPC框架，这样不仅能降低我们的维护成本，而且还可以让我们在一种RPC上面去精进。</p><p><strong>既然目标明确后，我们该如何实施呢？</strong></p><p>可能你会说这很简单啊，我们只要把所有的应用都改造成新RPC的使用方式，然后同时上线所有改造后的应用就可以了。如果在团队比较小的情况下，这种断崖式的更新可能确实是最快的方法，但如果是在团队比较大的情况下，要想做到同时上线所有改造后的应用，暂且不讨论这种方式是否存在风险，光从多个团队同一时间上线所有应用来看，这也几乎是一件不可能做到的事儿。</p><p>那对于多人团队来说，有什么办法可以让其把多个RPC框架统一到一个工具上呢？我们先看下多人团队在升级过程中所要面临的困难，人数多就意味着要维护的应用会比较多，应用多了之后线上应用之间的调用关系就会相对比较复杂。那这时候如果单纯地把任意一个应用目前使用的RPC框架换成新的RPC框架的话，就需要让所有调用这个应用的调用方去改成新的调用方式。</p><p>通过这种自下而上的滚动升级方式，最终是可以让所有的应用都切换到统一的RPC框架上，但是这种升级方式存在一定的局限性，首先要求我们能够清楚地梳理出各个应用之间的调用关系，只有这样，我们才能按部就班地把所有应用都升级到新的RPC框架上；其次要求应用之间的关系不能存在互相调用的情况，最好的情况就是应用之间的调用关系像一颗树，有一定的层次关系。但实际上我们应用的调用关系可能已经变成了网状结构，这时候想再按照这种方式去推进升级的话，就可能寸步难行了。</p><p>为了解决上面升级过程中遇到的问题，你可能还会想到另外一个方案，那就是在应用升级的过程中，先不移除原有的RPC框架，但同时接入新的RPC框架，让两种RPC同时提供服务，然后等所有的应用都接入完新的RPC以后，再让所有的应用逐步接入到新的RPC上。这样既解决了上面存在的问题，同时也可以让所有的应用都能无序地升级到统一的RPC框架上。</p><p>在保持原有RPC使用方式不变的情况下，同时引入新的RPC框架的思路，是可以让所有的应用最终都能升级到我们想要升级的RPC上，但对于开发人员来说，这样切换成本还是有点儿高，整个过程最少需要两次上线才能彻底地把应用里面的旧RPC都切换成新RPC。</p><p>那有没有更好的方式可以让应用上线一次就可以完成新老RPC的切换呢？关键就在于要让新的RPC能同时支持多种RPC调用，当一个调用方切换到新的RPC之后，调用方和服务提供方之间就可以用新的协议完成调用；当调用方还是用老的RPC进行调用的话，调用方和服务提供方之间就继续沿用老的协议完成调用。对于服务提供方来说，所要处理的请求关系如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/c6/87/c6e87eea6d8f312e949af71b3e1eea87.jpg?wh=2345*1942" alt="" title="调用关系"></p><h2>怎么优雅处理多协议？</h2><p>要让新的RPC同时支持多种RPC调用，关键就在于要让新的RPC能够原地支持多种协议的请求。怎么才能做到？在<a href="https://time.geekbang.org/column/article/199651">[第 02 讲]</a> 我们说过，协议的作用就是用于分割二进制数据流。每种协议约定的数据包格式是不一样的，而且每种协议开头都有一个协议编码，我们一般叫做magic number。</p><p>当RPC收到了数据包后，我们可以先解析出magic number来。获取到magic number后，我们就很容易地找到对应协议的数据格式，然后用对应协议的数据格式去解析收到的二进制数据包。</p><p>协议解析过程就是把一连串的二进制数据变成一个RPC内部对象，但这个对象一般是跟协议相关的，所以为了能让RPC内部处理起来更加方便，我们一般都会把这个协议相关的对象转成一个跟协议无关的RPC对象。这是因为在RPC流程中，当服务提供方收到反序列化后的请求的时候，我们需要根据当前请求的参数找到对应接口的实现类去完成真正的方法调用。如果这个请求参数是跟协议相关的话，那后续RPC的整个处理逻辑就会变得很复杂。</p><p>当完成了真正的方法调用以后，RPC返回的也是一个跟协议无关的通用对象，所以在真正往调用方写回数据的时候，我们同样需要完成一个对象转换的逻辑，只不过这时候是把通用对象转成协议相关的对象。</p><p>在收发数据包的时候，我们通过两次转换实现RPC内部的处理逻辑跟协议无关，同时保证调用方收到的数据格式跟调用请求过来的数据格式是一样的。整个流程如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/43/37/43451aea86fef673c3928230191fac37.jpg?wh=2517*1912" alt="" title="多协议处理流程"></p><h2>总结</h2><p>在我们日常开发的过程中，最难的环节不是从0到1完成一个新应用的开发，而是把一个老应用通过架构升级完成从70分到80分的跳跃。因为在老应用升级的过程中，我们不仅需要考虑既有的功能逻辑，也需要考虑切换到新架构上的成本，这就要求我们在设计新架构的时候要考虑如何让老应用能够平滑地升级，就像在RPC里面支持多协议一样。</p><p>在RPC里面支持多协议，不仅能让我们更从容地推进应用RPC的升级，还能为未来在RPC里面扩展新协议奠定一个良好的基础。所以我们平时在设计应用架构的时候，不仅要考虑应用自身功能的完整性，还需要考虑应用的可运维性，以及是否能平滑升级等一些软性能力。</p><h2>课后思考</h2><p>在RPC里面支持多协议的时候，有一个关键点就是能够识别出不同的协议，并且根据不同的magic number找到不同协议的解析逻辑。如果线上协议存在很多种的话，就需要我们事先在RPC里面内置各种协议，但通过枚举的方式可能会遗漏，不知道针对这种问题你有什么好的办法吗？</p><p>欢迎留言和我分享你的答案，也欢迎你把文章分享给你的朋友，邀请他加入学习。我们下节课再见！</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/57/fb/7f6d64ba.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Desmond</span>
  </div>
  <div class="_2_QraFYR_0">在反序列化后，且调用API前加一个过滤器，识别是什么协议，老协议按老逻辑走，新协议按新协议逻辑走，多个过滤器构成一个调用链</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 思路不错👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-20 20:51:39</div>
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
  <div class="_2_QraFYR_0"><br>RPC架构层次：<br>♻️序列化框架：作用是将方法调用时传入的发送数据从对象转为二进制，序列化一般用在协议里面的payload里面（最简单就是java对象序列化为二进制流封装为消息体然后基于http传输）如jdk、msgpack、protobuf、json、hessian等，推荐首选的还是 Hessian 与 Protobuf。<br>♻️编解码框架：编解码是对网络传输消息进行处理，把二进制的数据(payload)进一步封装（或者拆解）为rpc的协议(消息头+消息体)。<br>♻️协议：包括协议头和协议体，协议的作用就是用于分割二进制数据流。每种协议约定的数据包格式是不一样的。是网络传输数据格式的约定，作用将发送的数据按照一定的规约进行序列化为二进制流，有http&#47;tcp&#47;ftp等协议，grpc就是就是基于http的协议。<br>     -&gt;补充：协议解析过程就是把一连串的二进制数据变成一个 RPC 内部对象，但这个对象一般是跟协议相关的，所以为了能让 RPC内部处理起来更加方便，我们一般都会把这个协议相关的对象转成一个跟协议无关的 RPC 对象。<br>♻️Proxy：作用是让RPC框架根据调用的服务接口提前生成动态代理实现类，实现类似本地的调用感觉。<br>♻️负载均衡框架：结合服务注册中心实现服务端列表的路由选择和调用，如restTemplate、ribbon、feignClient等。<br>♻️熔断降级框架：实现调用过程中的熔断保护和降级函数。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-15 14:23:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/26/38/ef063dc2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Darren</span>
  </div>
  <div class="_2_QraFYR_0">magic number和协议可以缓存到map中，或者redis中，从配置中心读取，当配置中心修改后，热更新到对应的缓存中。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-15 00:14:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/55/f2/ba68d931.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>有米</span>
  </div>
  <div class="_2_QraFYR_0">dubbo就支持多种协议</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-05 16:41:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/f1/55/8ac4f169.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>陈国林</span>
  </div>
  <div class="_2_QraFYR_0">当下云原生微服务框架 Service meth大火，通过在meth层做兼容应该可以解决多RPC协议问题</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: mesh是未来一个大方向</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-26 17:11:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c6/cb/c7541d52.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>cwfighter</span>
  </div>
  <div class="_2_QraFYR_0">插件化，支持自定义协议插件即可</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-03 07:54:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>hillwater</span>
  </div>
  <div class="_2_QraFYR_0">这种是单端口多协议吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-14 15:11:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/f2/f5/b82f410d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Unknown element</span>
  </div>
  <div class="_2_QraFYR_0">“等所有的应用都接入完新的 RPC 以后，再让所有的应用逐步接入到新的 RPC 上”<br>这句话没看懂逻辑</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-04 00:02:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c6/cb/c7541d52.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>cwfighter</span>
  </div>
  <div class="_2_QraFYR_0">还是要结合先住，如果rpc框架已经比较老旧了，还不如直接完全迁移到新的RPC框架做一次技术体系升级。我们目前就在做框架迁移，修改老的框架成本高，稳定性存疑，果断放弃，直接迁移新框架。当然，过程是很漫长的，迁移细节也很多。。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-03 01:12:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/81/66/31c7969e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>随风而逝</span>
  </div>
  <div class="_2_QraFYR_0">课后思考，解析magicnumber拿到协议类型，如果类型遗漏或更新不及时，就需要某种机制去支持我们能动态的获取对应协议的解析规则。想到一种方案，就是增加新的协议时，可将 新的协议及对应的解析规则进行存储，比如说放在一个 “协议配置中心”， 服务方通过magicnumber进协议匹配时，去请求协议配置中心，拉去对应的解析规则，拉取后进行实例后存储在本地。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-28 17:09:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/67/f4/9a1feb59.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>钱</span>
  </div>
  <div class="_2_QraFYR_0">如果线上协议存在很多种的话，就需要我们事先在 RPC 里面内置各种协议，但通过枚举的方式可能会遗漏，不知道针对这种问题你有什么好的办法吗？<br>没get到为什么会出现遗漏的现象？<br>如果真的会遗漏，评论中协议过滤器链的思路很好</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-17 11:25:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/45/26/f54c9888.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>redis</span>
  </div>
  <div class="_2_QraFYR_0">配置外部化，rpc框架一般都支持协议的扩展，可以通过加载外部配置的方式去构建应用支持的协议</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-23 10:34:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/dd/ef/090e13a5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>zero</span>
  </div>
  <div class="_2_QraFYR_0">rpc的调用是微服务间直连调用，请问协议转换这层逻辑放在哪里，换了新协议是指在当前rpc框架更换协议还是直接更换了rpc框架</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 为了平滑升级，需要在新rpc里面去做兼容</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-16 08:37:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/10/75/ff76024c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>那个谁</span>
  </div>
  <div class="_2_QraFYR_0">我想到的一种思路是，上线带监控的rpc，把未包含的协议监控起来，逐步上线支持</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果发现没有支持的，是不是就已经造成业务受损</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-15 06:25:40</div>
  </div>
</div>
</div>
</li>
</ul>