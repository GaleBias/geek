<audio title="10｜设计原理：Dubbo核心设计原理剖析" src="https://static001.geekbang.org/resource/audio/d8/38/d87af3172b7a045551936a4242db2538.mp3" controls="controls"></audio> 
<p>你好，我是丁威。</p><p>这节课，我们来剖析一下Dubbo中一些重要的设计理念。这些设计理念非常重要，在接下来的11和12讲Dubbo案例中也都会用到，所以希望你能跟上我的节奏，好好吸收这些知识。</p><p>微服务架构体系包含的技术要点很多，我们这节课没法覆盖Dubbo的所有设计理念，但我会带着你梳理Dubbo设计理念的整体脉络，把生产实践过程中会频繁用到的底层原理讲透，让你轻松驾驭Dubbo微服务。</p><p>我们这节课的主要内容包括服务注册与动态发现、服务调用、网络通信模型、高度灵活的扩展机制和泛化调用五个部分。</p><h2><strong>服务注册与动态发现</strong></h2><p>我们首先来看一下Dubbo的服务注册与动态发现机制。</p><p>Dubbo的服务注册与发现机制如<a href="https://dubbo.apache.org/imgs/architecture.png">下图</a>所示：</p><p><img src="https://static001.geekbang.org/resource/image/a1/3b/a15364103e405767efbca0719959773b.png?wh=899x531" alt="图片"></p><p>Dubbo中主要包括四类角色，它们分别是注册中心（Registry）、服务调用者&amp;消费端（Consumer）、服务提供者（Provider）和监控中心（Monitor）。</p><p>在实现服务注册与发现时，有三个要点。</p><ol>
<li>服务提供者(Provider)在启动的时候在注册中心(Register)注册服务，注册中心(Registry)会存储服务提供者的相关信息。</li>
<li>服务调用者(Consumer)在启动的时候向注册中心订阅指定服务，注册中心将以某种机制（推或拉）告知消费端服务提供者列表。</li>
<li>当服务提供者数量变化（服务提供者扩容、缩容、宕机等因素）时，注册中心需要以某种方式(推或拉)告知消费端，以便消费端进行正常的负载均衡。</li>
</ol><!-- [[[read_end]]] --><p>Dubbo官方提供了多种注册中心，我们选择使用最为普遍的ZooKeeper进一步理解注册中心的原理。</p><p>我们先来看一下Zookeeper注册中心中的数据存储目录结构。</p><p><img src="https://static001.geekbang.org/resource/image/af/eb/af1712a5fb964c02b6199bb75f21d2eb.png?wh=782x155" alt="图片"></p><p>可以看到，它的目录组织结构为 /dubbo/{ServiceName}，其中，ServiceName表示一个具体的服务，通常用包名+类名表示，在每一个服务名下又会创建四个目录，它们分别是：</p><ul>
<li>providers，服务提供者列表；</li>
<li>consumers，消费者列表；</li>
<li>routers，路由规则列表（一个服务可以设置多个路由规则）；</li>
<li>configurators，动态配置条目。</li>
</ul><p>要说明的是，在Dubbo中，我们可以在不重启消费者、服务提供者的前提下动态修改服务提供者、服务消费者的配置，配置信息发生变化后会存储在configurators子节点中。此时，服务提供者、消费者会动态监听配置信息的变化，变化一旦发生就使用最新的配置重构服务提供者和服务消费者。</p><p>基于Zookeeper注册中心的服务注册与发现有下面三个实现细节。</p><ol>
<li>服务提供者启动时会向注册中心进行注册，具体是在对应服务的providers目录下增加一条记录（临时节点），记录服务提供者的IP、端口等信息。同时服务提供者会监听configurators节点的变化。</li>
<li>服务消费者在启动时会向注册中心订阅服务，具体是在对应服务的consumers目录下增加一条记录（临时节点），记录消费者的IP、端口等信息，同时监听 configurators、routers 目录的变化，所谓的监听就是利用ZooKeeper提供的watch机制。</li>
<li>当有新的服务提供者上线后， providers 目录会增加一条记录，注册中心会将最新的服务提供者列表推送给服务调用方（消费端），这样消费者可以立刻收到通知，知道服务提供者的列表产生了变化。如果一个服务提供者宕机，因为它是临时节点，所以ZooKeeper会把这个节点移除，同样会触发事件，消费端一样能得知最新的服务提供者列表，从而实现路由的动态注册与发现。</li>
</ol><h2><strong>服务调用</strong></h2><p>接下来我们再来看看服务调用。Dubbo的服务调用设计十分优雅，其实现原理图如下：</p><p><img src="https://static001.geekbang.org/resource/image/4d/93/4d4a8605fb86c23575d7e72bf5480b93.jpg?wh=1826x990" alt="图片"></p><p>服务调用重点阐述的是客户端发起一个RPC服务调用时的所有实现细节，<strong>它包括服务发现、故障转移、路由转发、负载均衡等方面，是Dubbo实现灰度发布、多环境隔离的理论指导。</strong></p><p>刚才，我们已经就服务发现做了详细介绍，接下来我们重点关注负载均衡、路由、故障转移这几个方面。</p><p>客户端通过服务发现机制，能动态发现当前存活的服务提供者列表，接下来要考虑的就是如何从服务提供者列表中选择一个服务提供者发起调用，这就是所谓的<strong>负载均衡（LoadBalance）</strong>。</p><p>Dubbo默认提供了随机、加权随机、最少活跃连接、一致性Hash等负载均衡算法。</p><p>值得注意的是，Dubbo不仅提供了负载均衡机制，还提供了智能路由机制，这是实现Dubbo灰度发布的重要理论基础。</p><p>所谓路由机制，是指设置一定的规则对服务提供者列表进行过滤。负载均衡时，只在经过了路由机制的服务提供者列表中进行选择。为了更好地理解路由机制的工作原理，你可以看看下面这张示意图：</p><p><img src="https://static001.geekbang.org/resource/image/e3/ab/e3556fc21cc5676e15cb5290523475ab.jpg?wh=1920x951" alt="图片"></p><p>我们为查找用户信息服务设置了一条路由规则，即“查询机构ID为102的查询用户请求信息将被发送到新版本（192.168.3.102）上。<strong>具体的做法是，在进行负载均衡之前先执行路由选择，按照路由规则对原始的服务提供者列表进行过滤，从中挑选出符合要求的提供者列表，然后再进行负载均衡。</strong></p><p>接下来，客户端就要向服务提供者发起RPC请求调用了。远程服务调用通常涉及到网络等因素，因此并不能保证100%成功，当调用失败时应该采用什么策略呢？</p><p>Dubbo提供了下面五种策略：</p><ul>
<li>failover，失败后选择另外一台服务提供者进行重试，重试次数可配置，通常适合实现幂等服务的场景；</li>
<li>failfast，快速失败，失败后立即返回错误；</li>
<li>failsafe，调用失败后打印错误日志，返回成功，通常用于记录审计日志等场景；</li>
<li>failback，调用失败后，返回成功，但会在后台定时无限次重试，重启后不再重试；</li>
<li>forking，并发调用，收到第一个响应结果后返回给客户端。通常适合实时性要求比较高的场景。但这一策略浪费服务器资源，通常可以通过forks参数设置并发调用度。</li>
</ul><p>如果将服务调用落到底层，就不得不说说网络通信模型了，这部分包含了很多<strong>性能调优手段</strong>。</p><h2><strong>网络通信模型</strong></h2><p>我们先看看Dubbo的网络通信模型，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/ed/59/ed2899d0c47c307f43e40ac7d6788959.jpg?wh=1104x322" alt="图片"></p><p>Dubbo的网络通信模型主要包括<strong>网络通信协议和线程派发机制（Dispatcher）</strong>两部分。</p><p>网络传输通常需要自定义通信协议，我们常用的协议设计方式是 <strong>Header + Body</strong>,其中Header 长度固定，包含一个长度字段，用于记录整个协议包的大小。</p><p>同时，为了提高传输效率，我们一般会对传输数据也就是Body的内容进行序列化与压缩处理。</p><p>Dubbo支持目前支持 java、compactedjava、nativejava、fastjson、fst、hessian2、kryo等序列化协议，生产环境默认为hessian2。</p><p>网络通信模型的另一部分是线程派发机制。Dubbo中会默认创建200个线程处理业务，这时候就需要线程派发机制来指导IO线程与业务线程如何分工。</p><p>Dubbo提供了下面几种线程派发机制：</p><ul>
<li>all，所有的请求转发到业务线程池中执行（IO读写、心跳包除外，因为在Dubbo中这两种请求都必须在IO线程中执行，不能通过配置修改）；</li>
<li>message，只有请求事件在线程池中执行，其他请求在IO线程上执行；</li>
<li>connection ，求事件在线程池中执行，<strong>连接和断开连接的事件排队执行（含一个线程的线程池）；</strong></li>
<li>direct，所有请求直接在IO线程中执行。</li>
</ul><p>为什么线程派发机制有这么多种策略呢？其实这主要是考虑到线程切换带来的开销问题。也就是说，我们希望通过多种策略让线程切换带来的开销小于多线程处理带来的提升。</p><p>我举个例子，Dubbo中的心跳包都必须在IO线程中执行。在处理心跳包时，我们只需直接返回PONG包（OK）就可以了，逻辑非常简单，处理速度也很快。如果将心跳包转换到业务线程池，性能不升反降，因为切换线程会带来额外的性能损耗，得不偿失。</p><p>网络编程中需要遵循一条最佳实践：<strong>IO线程中不能有阻塞操作，通常将阻塞操作转发到业务线程池异步执行</strong>。</p><p>与网络通信协议相关的参数定义在dubbo:protocol，关键的设置属性如下。</p><ul>
<li>threads，业务线程池线程个数，默认为200。</li>
<li>queues，业务线程池队列长度，默认为0，表示不支持排队，如果线程池满，则直接拒绝。该参数与threads配合使用，主要是对服务端进行限流，一旦超过其处理能力，就拒绝请求，快速失败，引导客户端重试。</li>
<li>iothreads：默认为CPU核数再加一，用于处理网络读写。在生产实践中，通常的瓶颈在于业务线程池，如果业务线程无明显瓶颈（jstack日志查询到业务线程基本没怎么干活），但吞吐量已经无法继续提升了，可以考虑调整iothreads，增加IO线程数量，提高IO读写并发度。该值建议保持在“2*CPU核数”以下。</li>
<li>serialization：序列化协议，新版本支持protobuf等高性能序列化机制。</li>
<li>dispatcher：线程派发机制，默认为all。</li>
</ul><h2><strong>高度灵活的扩展机制</strong></h2><p>Dubbo出现之后迅速成为微服务领域最受欢迎的框架，除操作简单这个原因外，还有扩展机制的功劳。Dubbo高度灵活的扩展机制堪称“王者级别的设计”。</p><p>Dubbo的扩展设计主要是基于SPI设计理念，我们来看下具体的实现方案。</p><p>Dubbo所有的底层能力都通过接口来定义。用户在扩展时只需要实现对应的接口，定义一个统一的扩展目录（META-INF.dubbo.internal）存放所有的扩展定义即可。要注意的是，目录下的文件名是需要扩展的接口的全名，像下图这样：</p><p><img src="https://static001.geekbang.org/resource/image/88/92/88b945687b4e4f3b286a70c11b606192.png?wh=1276x291" alt="图片"></p><p>在初次使用对应接口实例时，可以扫描扩展目录中的文件，并根据文件中存储的key-value初始化具体的实例。</p><p>我们以RPC模块为例看一下Dubbo强悍的扩展能力。众所周知，目前gRPC协议以优异的性能表现正在逐步成为RPC领域的王者，很多人误以为gRPC是来革Dubbo的“命”的。其实不然，我们可以认为Dubbo是微服务体系的完整解决方案，而RPC只是微服务体系中的重要一环，Dubbo完全可以吸收gRPC，让gRPC成为Dubbo的远程调用方式。</p><p>具体的做法只需要在dubbo-rpc模块中添加一个dubbo-rpc-grpc模块，然后使用gRPC实现org.apache.dubbo.rpc.protocol接口，并将其配置在扩展目录中：</p><p><img src="https://static001.geekbang.org/resource/image/aa/39/aa4df0df4f85c2dd69b5f6338bd05339.png?wh=1029x278" alt="图片"></p><p>面对gRPC这么强大的功能扩展机制，绝大部分人应该和我一样，都是作为中间件的应用人员，不需要使用模块级别的扩展机制。我们通常只是结合应用场景来进行功能扩展。</p><p>Dubbo在业务功能级别的扩展可以通过Filter机制来实现。Filter的工作机制如下：</p><p><img src="https://static001.geekbang.org/resource/image/f8/5b/f8c442f9769948db206c0cc5fc92f15b.jpg?wh=1920x577" alt="图片"></p><p>这里，<strong>过滤器链的执行时机是在服务消费者发起远程RPC请求之前。</strong>最先执行的是消费端的过滤器链，每一个过滤器可以设置执行顺序。<strong>服务端在解码之后、执行业务逻辑之前，也会首先调用过滤器链。</strong></p><p>在专栏的最后一讲，我还会通过一个全链路压测方案讲解如何利用Filter机制来解决实际问题。</p><h2><strong>泛化调用</strong></h2><p>在这节课的最后，我们再来介绍一下Dubbo的泛化调用机制，它也是实现Dubbo网关的理论基础。</p><p>我们在开发Dubbo应用时通常会包含API、Consumer、Provider三个子模块。</p><p>其中API模块通常定义统一的服务接口，而Consumer、Provider模块都需要显示依赖API模块。这种设计理念虽然将Provider与Consumer进行了解耦合，但对API模块形成了强依赖，如果API模块发生改变，Provider和Consumer必须同时改变。也就是说，一旦API模块发生变化，服务调用方、服务消费方都需要重新部署，这对应用发布来说非常不友好。特别是在网关领域，几乎是不可接受的，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/29/b7/29c50882210f4b473effc086b14749b7.jpg?wh=1920x594" alt="图片"></p><p>公司的微服务在不停地演进，如果网关需要跟着API模块不停地发布新版本，网关的可用性和稳定性都将受到极大挑战。怎么解决这个问题呢？</p><p>这就要说到Dubbo的机制了。泛化调用具体实现原理如下：</p><p><img src="https://static001.geekbang.org/resource/image/2d/6d/2ddd22669748ee4b8cc1028878c5fc6d.jpg?wh=1920x706" alt="图片"></p><p>当服务消费端发生调用时，我们使用Map来存储一个具体的请求参数对象，然后传输到服务提供方。由于服务提供方引入了模型相关的Jar，服务提供方在执行业务方法之前，需要将Map转化成具体的模型对象，然后再执行业务逻辑。</p><p>Dubbo的泛化调用在服务提供方的转化是通过Filter机制统一处理的，服务端并不需要关注消费方采取何种方式进行调用。</p><p>通过泛化调用机制，客户端不再需要依赖服务端的Jar包，服务端可以不断地演变，而不会影响客户端已有服务的运行。</p><h2><strong>总结</strong></h2><p>好了，这节课就讲到这里。我们这节课主要介绍了Dubbo的服务注册与发现、服务调用、网络通信模型、扩展机制还有泛化调用等核心工作机制，了解这些内容可以指导我们更好实践微服务。</p><p>另外，Dubbo框架算是阿里巴巴开源的所有框架中文档最为齐全的框架了，非常值得我们深入学习与研究。如果你想要进一步掌握Dubbo，建议你看看<a href="https://dubbo.apache.org/zh/docs/introduction/">Dubbo官方文档</a>。</p><h2><strong>课后题</strong></h2><ol>
<li>我们将在下节课和你一起聊聊Dubbo的网关设计方案，其中泛化调用是其理论设计基础，所以我们的第一道课后题就是，请你试着先编写一个Dubbo泛化调用的示例。</li>
</ol><p>提示一下，Dubbo提供了dubbo-demo模块，你可以在官方提供的示例中进行泛化调用编写，节省搭建基础项目的时间。</p><ol start="2">
<li>请你尝试通过dubbo-admin运维管理工具动态修改参数，看看它是否可以动态生效。你知道它背后是如何实现的么？</li>
</ol><p>欢迎你在留言区与我交流讨论，我们下节课再见！</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/86/25/25ded6c3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>zhengyu.nie</span>
  </div>
  <div class="_2_QraFYR_0">作者你好，dubbo线程池默认200个线程设置基于什么考虑？你的线程池文章推荐是20个线程</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-26 01:27:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/7c/89/93a4019e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>fantasyer</span>
  </div>
  <div class="_2_QraFYR_0">老师好，网络通信模型这部分提到，iothreads默认为 CPU 核数再加一，如果增加iothreads的话也建议保持在“2*CPU 核数”以下，这里线程数不超过2*CPU核数原因是什么？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-19 23:30:00</div>
  </div>
</div>
</div>
</li>
</ul>