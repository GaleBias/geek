<audio title="08 _ 服务发现：到底是要CP还是AP？" src="https://static001.geekbang.org/resource/audio/8d/93/8d7ee317944e33a924f75445ceaa3393.mp3" controls="controls"></audio> 
<p>你好，我是何小锋。在上一讲中，我讲了“怎么设计一个灵活的RPC框架”，总结起来，就是怎么在RPC框架中应用插件，用插件方式构造一个基于微内核的RPC框架，其关键点就是“插件化”。</p><p>今天，我要和你聊聊RPC里面的“服务发现”在超大规模集群的场景下所面临的挑战。</p><h2>为什么需要服务发现？</h2><p>先举个例子，假如你要给一位以前从未合作过的同事发邮件请求帮助，但你却没有他的邮箱地址。这个时候你会怎么办呢？如果是我，我会选择去看公司的企业“通信录”。</p><p>同理，为了高可用，在生产环境中服务提供方都是以集群的方式对外提供服务，集群里面的这些IP随时可能变化，我们也需要用一本“通信录”及时获取到对应的服务节点，这个获取的过程我们一般叫作“服务发现”。</p><p>对于服务调用方和服务提供方来说，其契约就是接口，相当于“通信录”中的姓名，服务节点就是提供该契约的一个具体实例。服务IP集合作为“通信录”中的地址，从而可以通过接口获取服务IP的集合来完成服务的发现。这就是我要说的RPC框架的服务发现机制，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/51/5d/514dc04df2b8b2f3130b7d44776a825d.jpg?wh=2746*1445" alt="" title="RPC服务发现原理图"></p><ol>
<li>服务注册：在服务提供方启动的时候，将对外暴露的接口注册到注册中心之中，注册中心将这个服务节点的IP和接口保存下来。</li>
<li>服务订阅：在服务调用方启动的时候，去注册中心查找并订阅服务提供方的IP，然后缓存到本地，并用于后续的远程调用。</li>
</ol><!-- [[[read_end]]] --><h2>为什么不使用DNS？</h2><p>既然服务发现这么“厉害”，那是不是很难实现啊？其实类似机制一直在我们身边，我们回想下服务发现的本质，就是完成了接口跟服务提供者IP的映射。那我们能不能把服务提供者IP统一换成一个域名啊，利用已经成熟的DNS机制来实现？</p><p>好，先带着这个问题，简单地看下DNS的流程：</p><p><img src="https://static001.geekbang.org/resource/image/3b/18/3b6a23f392b9b8d6fcf31803a5b4ef18.jpg?wh=5273*1884" alt="" title="DNS查询流程"></p><p>如果我们用DNS来实现服务发现，所有的服务提供者节点都配置在了同一个域名下，调用方的确可以通过DNS拿到随机的一个服务提供者的IP，并与之建立长连接，这看上去并没有太大问题，但在我们业界为什么很少用到这种方案呢？不知道你想过这个问题没有，如果没有，现在可以停下来想想这样两个问题：</p><ul>
<li>如果这个IP端口下线了，服务调用者能否及时摘除服务节点呢？</li>
<li>如果在之前已经上线了一部分服务节点，这时我突然对这个服务进行扩容，那么新上线的服务节点能否及时接收到流量呢？</li>
</ul><p>这两个问题的答案都是：“不能”。这是因为为了提升性能和减少DNS服务的压力，DNS采取了多级缓存机制，一般配置的缓存时间较长，特别是JVM的默认缓存是永久有效的，所以说服务调用者不能及时感知到服务节点的变化。</p><p>这时你可能会想，我是不是可以加一个负载均衡设备呢？将域名绑定到这台负载均衡设备上，通过DNS拿到负载均衡的IP。这样服务调用的时候，服务调用方就可以直接跟VIP建立连接，然后由VIP机器完成TCP转发，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/d8/b9/d8549f6069a8ca5bd1012a0baf90f6b9.jpg?wh=2553*1299" alt="" title="VIP方案"></p><p>这个方案确实能解决DNS遇到的一些问题，但在RPC场景里面也并不是很合适，原因有以下几点：</p><ul>
<li>搭建负载均衡设备或TCP/IP四层代理，需求额外成本；</li>
<li>请求流量都经过负载均衡设备，多经过一次网络传输，会额外浪费些性能；</li>
<li>负载均衡添加节点和摘除节点，一般都要手动添加，当大批量扩容和下线时，会有大量的人工操作和生效延迟；</li>
<li>我们在服务治理的时候，需要更灵活的负载均衡策略，目前的负载均衡设备的算法还满足不了灵活的需求。</li>
</ul><p>由此可见，DNS或者VIP方案虽然可以充当服务发现的角色，但在RPC场景里面直接用还是很难的。</p><h2>基于ZooKeeper的服务发现</h2><p>那么在RPC里面我们该如何实现呢？我们还是要回到服务发现的本质，就是完成接口跟服务提供者IP之间的映射。这个映射是不是就是一种命名服务？当然，我们还希望注册中心能完成实时变更推送，是不是像开源的ZooKeeper、etcd就可以实现？我很肯定地说“确实可以”。下面我就来介绍下一种基于ZooKeeper的服务发现方式。</p><p>整体的思路很简单，就是搭建一个ZooKeeper集群作为注册中心集群，服务注册的时候只需要服务节点向ZooKeeper节点写入注册信息即可，利用ZooKeeper的Watcher机制完成服务订阅与服务下发功能，整体流程如下图：</p><p><img src="https://static001.geekbang.org/resource/image/50/75/503fabeeae226a722f83e9fb6c0d4075.jpg?wh=4214*1803" alt="" title="基于ZooKeeper服务发现结构图"></p><ol>
<li>服务平台管理端先在ZooKeeper中创建一个服务根路径，可以根据接口名命名（例如：/service/com.demo.xxService），在这个路径再创建服务提供方目录与服务调用方目录（例如：provider、consumer），分别用来存储服务提供方的节点信息和服务调用方的节点信息。</li>
<li>当服务提供方发起注册时，会在服务提供方目录中创建一个临时节点，节点中存储该服务提供方的注册信息。</li>
<li>当服务调用方发起订阅时，则在服务调用方目录中创建一个临时节点，节点中存储该服务调用方的信息，同时服务调用方watch该服务的服务提供方目录（/service/com.demo.xxService/provider）中所有的服务节点数据。</li>
<li>当服务提供方目录下有节点数据发生变更时，ZooKeeper就会通知给发起订阅的服务调用方。</li>
</ol><p>我所在的技术团队早期使用的RPC框架服务发现就是基于ZooKeeper实现的，并且还平稳运行了一年多，但后续团队的微服务化程度越来越高之后，ZooKeeper集群整体压力也越来越高，尤其在集中上线的时候越发明显。“集中爆发”是在一次大规模上线的时候，当时有超大批量的服务节点在同时发起注册操作，ZooKeeper集群的CPU突然飙升，导致ZooKeeper集群不能工作了，而且我们当时也无法立马将ZooKeeper集群重新启动，一直到ZooKeeper集群恢复后业务才能继续上线。</p><p>经过我们的排查，引发这次问题的根本原因就是ZooKeeper本身的性能问题，当连接到ZooKeeper的节点数量特别多，对ZooKeeper读写特别频繁，且ZooKeeper存储的目录达到一定数量的时候，ZooKeeper将不再稳定，CPU持续升高，最终宕机。而宕机之后，由于各业务的节点还在持续发送读写请求，刚一启动，ZooKeeper就因无法承受瞬间的读写压力，马上宕机。</p><p>这次“意外”让我们意识到，ZooKeeper集群性能显然已经无法支撑我们现有规模的服务集群了，我们需要重新考虑服务发现方案。</p><h2>基于消息总线的最终一致性的注册中心</h2><p>我们知道，ZooKeeper的一大特点就是强一致性，ZooKeeper集群的每个节点的数据每次发生更新操作，都会通知其它ZooKeeper节点同时执行更新。它要求保证每个节点的数据能够实时的完全一致，这也就直接导致了ZooKeeper集群性能上的下降。这就好比几个人在玩传递东西的游戏，必须这一轮每个人都拿到东西之后，所有的人才能开始下一轮，而不是说我只要获得到东西之后，就可以直接进行下一轮了。</p><p>而RPC框架的服务发现，在服务节点刚上线时，服务调用方是可以容忍在一段时间之后（比如几秒钟之后）发现这个新上线的节点的。毕竟服务节点刚上线之后的几秒内，甚至更长的一段时间内没有接收到请求流量，对整个服务集群是没有什么影响的，所以我们可以牺牲掉CP（强制一致性），而选择AP（最终一致），来换取整个注册中心集群的性能和稳定性。</p><p>那么是否有一种简单、高效，并且最终一致的更新机制，能代替ZooKeeper那种数据强一致的数据更新机制呢？</p><p>因为要求最终一致性，我们可以考虑采用消息总线机制。注册数据可以全量缓存在每个注册中心内存中，通过消息总线来同步数据。当有一个注册中心节点接收到服务节点注册时，会产生一个消息推送给消息总线，再通过消息总线通知给其它注册中心节点更新数据并进行服务下发，从而达到注册中心间数据最终一致性，具体流程如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/73/ff/73b59c7949ebed2903ede474856062ff.jpg?wh=4256*2276" alt="" title="流程图"></p><ul>
<li>当有服务上线，注册中心节点收到注册请求，服务列表数据发生变化，会生成一个消息，推送给消息总线，每个消息都有整体递增的版本。</li>
<li>消息总线会主动推送消息到各个注册中心，同时注册中心也会定时拉取消息。对于获取到消息的在消息回放模块里面回放，只接受大于本地版本号的消息，小于本地版本号的消息直接丢弃，从而实现最终一致性。</li>
<li>消费者订阅可以从注册中心内存拿到指定接口的全部服务实例，并缓存到消费者的内存里面。</li>
<li>采用推拉模式，消费者可以及时地拿到服务实例增量变化情况，并和内存中的缓存数据进行合并。</li>
</ul><p>为了性能，这里采用了两级缓存，注册中心和消费者的内存缓存，通过异步推拉模式来确保最终一致性。</p><p>另外，你也可能会想到，服务调用方拿到的服务节点不是最新的，所以目标节点存在已经下线或不提供指定接口服务的情况，这个时候有没有问题？这个问题我们放到了RPC框架里面去处理，在服务调用方发送请求到目标节点后，目标节点会进行合法性验证，如果指定接口服务不存在或正在下线，则会拒绝该请求。服务调用方收到拒绝异常后，会安全重试到其它节点。</p><p>通过消息总线的方式，我们就可以完成注册中心集群间数据变更的通知，保证数据的最终一致性，并能及时地触发注册中心的服务下发操作。在RPC领域精耕细作后，你会发现，服务发现的特性是允许我们在设计超大规模集群服务发现系统的时候，舍弃强一致性，更多地考虑系统的健壮性。最终一致性才是分布式系统设计中更为常用的策略。</p><h2>总结</h2><p>今天我分享了RPC框架服务发现机制，以及如何用ZooKeeper完成“服务发现”，还有ZooKeeper在超大规模集群下作为注册中心所存在的问题。</p><p>通常我们可以使用ZooKeeper、etcd或者分布式缓存（如Hazelcast）来解决事件通知问题，但当集群达到一定规模之后，依赖的ZooKeeper集群、etcd集群可能就不稳定了，无法满足我们的需求。</p><p>在超大规模的服务集群下，注册中心所面临的挑战就是超大批量服务节点同时上下线，注册中心集群接受到大量服务变更请求，集群间各节点间需要同步大量服务节点数据，最终导致如下问题：</p><ul>
<li>注册中心负载过高；</li>
<li>各节点数据不一致；</li>
<li>服务下发不及时或下发错误的服务节点列表。</li>
</ul><p>RPC框架依赖的注册中心的服务数据的一致性其实并不需要满足CP，只要满足AP即可。我们就是采用“消息总线”的通知机制，来保证注册中心数据的最终一致性，来解决这些问题的。</p><p>另外，在今天的内容中，很多知识点不只可以应用到RPC框架的“服务发现”中。例如服务节点数据的推送采用增量更新的方式，这种方式提高了注册中心“服务下发”的效率，而这种方式，你还可以利用在其它地方，比如统一配置中心，用此方式可以提升统一配置中心下发配置的效率。</p><h2>课后思考</h2><p>目前服务提供者上线后会自动注册到注册中心，服务调用方会自动感知到新增的实例，并且流量会很快打到该新增的实例。如果我想把某些服务提供者实例的流量切走，除了下线实例，你有没有想到其它更便捷的办法呢？</p><p>欢迎留言和我分享你的思考和疑惑，也欢迎你把文章分享给你的朋友，邀请他加入学习。我们下节课再见！</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/69/c6/513df085.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>强哥</span>
  </div>
  <div class="_2_QraFYR_0">文章描述zk时不是很准确，zk并不是强一致性，而且顺序一致性。也算最终一致性的一种。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-21 19:07:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/54/9a/76c0af70.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>每天晒白牙</span>
  </div>
  <div class="_2_QraFYR_0">今天这个思考题好像是在说负载均衡策略，那是不是可以加个权重，把想下线的ip权重置为0，这样服务调用方就不会调用这个节点了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-06 07:49:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c3/08/28c327d0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>冰河时代</span>
  </div>
  <div class="_2_QraFYR_0">记得之前在京东的时候，服务挂了，在注册中心上还得要手动删除下死亡节点，如果zk的话，服务没了，就代表会话也没了，临时节点的特性，应该会被通知到呀？为什么还要手动删除呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 临时节点是需要等到超时时间之后才删除的，不够实时。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-21 17:15:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/0f/bf/ee93c4cf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>雨霖铃声声慢</span>
  </div>
  <div class="_2_QraFYR_0">如果要能切换流量，那么要服务端配置有权重负载均衡策略，这样服务器端可以通过调整权重来安排流量</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 很好的思路</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-07 13:04:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/69/cc/747c7629.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>🌀Pick Monster 🌀</span>
  </div>
  <div class="_2_QraFYR_0">消息总栈类似一个队列，队列表示是递增的数字，注册中心集群的任何一个节点接收到注册请求，都会把服务提供者信息发给消息总栈，消息总栈会像队列以先进先出的原则推送消息给所有注册中心集群节点，集群节点接收到消息后会比较自己内存中的当前版本，保存版本大的，这种方式有很强的实效性，注册中心集群也可以从消息总栈拉取消息，确保数据AP，个人理解这是为了防止消息未接收到导致个别节点数据不准确，因为服务提供者可以向任意一个节点发送注册请求，从而降低了单个注册中心的压力，而注册和注册中心同步是异步的，也解决了集中注册的压力，在Zookeeper中，因为Zookeeper注册集群的强一致性，导致必须所有节点执行完一次同步，才能执行新的同步，这样导致注册处理性能降低，从而高I&#47;O操作宕机。<br>以上是我的个人理解，老师给看一下是否正确。<br><br>还有一个问题：当集中注册时，消息总栈下发通知给注册中心集群节点，对于单个节点也会不停的收到更新通知，这里也存在高I&#47;O问题，会不会有宕机？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 上面理解的很棒。event bus可以改造成主从模式保证高可用</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-07 23:32:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/42/76/256bbd43.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>松花皮蛋me</span>
  </div>
  <div class="_2_QraFYR_0">路由负载</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 很棒</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-06 12:46:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/7b/98/8f1aecf4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>楼下小黑哥</span>
  </div>
  <div class="_2_QraFYR_0">服务消费者都是从注册中心拉取服务提供者的地址信息，所以我们要切走某些服务提供者数据，只需要将注册中心这些实例的地址信息删除（其实下线应用实例，实际也是去删除注册中心地址信息），然后注册中心反向通知消费者，消费者受到拉取最新提供者地址信息就没有这些实例了。<br>老师，提问一个问题：现有开源注册中心是不是还没有消息总线这种实现方式？消息总线有没有开源实现？<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 通过服务发现来摘除流量是最常见的手段，还可以上下线状态、权重等方式。现成的MQ也是可以充当消息总线来用</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-06 09:34:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJD8FqEJwgFLR345UPmwKibMribfD8rEHrtweQTsKPpkfLiaUCesXrW9Iib0niaibib0th6WcKbsKFoicFS2Q/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>君言</span>
  </div>
  <div class="_2_QraFYR_0">老师，在AP实现中“两级缓存，注册中心和消费者的内存缓存，通过异步推拉模式来确保最终一致性”能展开讲一下具体实现吗？另外请教下CP可以基于zk实现，AP在业内的实现方式有哪些呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 推主要实现callback，拉的动作在客户端，像Eurek属于AP</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-06 21:02:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/51/b4/0d402ae8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>南桥畂翊</span>
  </div>
  <div class="_2_QraFYR_0">消息总线策略是啥，老师能指点下么，怎么保证消息总线全局版本递增</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 最简单的就用时间戳</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-30 20:40:28</div>
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
  <div class="_2_QraFYR_0">小结一下<br>1：注册中心的核心作用？<br>完成服务消费者和服务调用者，两者的路径匹配<br><br>2：注册中心的核心指标？<br>高效、稳定、高可用，使服务消费者及时感知节点的变化<br><br>3：路径匹配需要的信息？<br>服务提供者IP、端口、接口、方法＋服务分组别名<br>服务消费者IP、端口<br>路径匹配可以把分组别名利用上，即使提供者实例上线，不过由于设置的别名和服务消费者需要的不一致流量也不会打过去，什么时候打过去可以通过配置中心来自由的控制。分组内也是有多个服务提供者的，这里可以再利用相关的负载均衡策略来具体分发流量。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 准确👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-14 09:16:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJiaiaQYkUHByDARl4ULduH8Y4hicOpMxGjzPZmJkXK9RYd1oVMICd0icCfarxI4Yvmiap2a8t3Eae3LMg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>etdick</span>
  </div>
  <div class="_2_QraFYR_0">EURAKA就是AP模型</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-12 02:20:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/c2/fe/038a076e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>阿卧</span>
  </div>
  <div class="_2_QraFYR_0">zookeeper注册中心实现原理<br>1. 服务平台向zookeeper创建服务目录<br>2. 服务提供者向zookeeper创建临时节点<br>3. 服务调用者订阅zookeeper，创建临时节点，拉取服务全量数据，watch服务全部节点数据<br>4. zookeeper节点发生变化会通知服务调用者<br><br>切掉服务流量，只需要将注册中心的配置节点下掉就好了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这还是利用了服务发现</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-06 11:21:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/be/d4/ff1c1319.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>金龟</span>
  </div>
  <div class="_2_QraFYR_0">基于消息总线的最终一致性，并不能解决zk 里面目录节点过多的问题吧，只能解决同时写入的问题吧</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-12 12:40:09</div>
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
  <div class="_2_QraFYR_0">擦，有点像大佬面试过我的面试题，一个题之后，大佬说我了解了。。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-08 17:22:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/23/be/81bd6a7b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>knife</span>
  </div>
  <div class="_2_QraFYR_0">稍微做点压测作为参考，我说的未必是对的，zookeeper写要是zab协议的，<br>提交要大多数成功才能提交，所以越多节点效率可能越慢，<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 所以zk有observer</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-14 15:48:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/e3/64/871dd03e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>成熟中的猪</span>
  </div>
  <div class="_2_QraFYR_0">如文中描述： ZooKeeper 的节点数量特别多，对 ZooKeeper 读写特别频繁，且 ZooKeeper 存储的目录达到一定数量的时候，ZooKeeper 将不再稳定，CPU 持续升高，最终宕机。<br>这里有点笼统，是否有量化的压测数据或者实际经验值可以分享？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个跟很多因素有关系，比如zk node数，机器内存大小等</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-09 21:47:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/e5/10/0a94311f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>joker</span>
  </div>
  <div class="_2_QraFYR_0">老师，您好。关于后面切换到AP模式的架构，服务调用方如何知道去哪个注册中心取数据呢？是需要维护一个注册中心的列表吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-19 08:52:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/45/b4/ec66d499.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>oscarwin</span>
  </div>
  <div class="_2_QraFYR_0">如果用消息总线去做同步，那么消息总线本身要是高可用和强一致的，那么要保证消息总线的高可用和强一致同样要采用一致性算法，那么消息总线不会成为新的性能瓶颈吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-16 14:15:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>嘻嘻</span>
  </div>
  <div class="_2_QraFYR_0">老师，消息总线构建ap 型注册中心，不是很理解。是多个注册中心独立提供读写，他们之间通过消息队列做数据同步？那么一致性感觉不好保证，比如服务a 先注册，再反注册，但是分别发到两个注册中心节点，最终同步可能是乱序的哇</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 一般不会出现这种情况，只有连接断开后，那需要重新注册</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-22 19:08:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/07/8c/0d886dcc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>蚂蚁内推+v</span>
  </div>
  <div class="_2_QraFYR_0">请问老师，dubbo的服务发现也是用zk实现的，在大规模时有这个问题呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-08 00:58:06</div>
  </div>
</div>
</div>
</li>
</ul>