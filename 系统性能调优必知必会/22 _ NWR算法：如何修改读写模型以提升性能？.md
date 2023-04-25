<audio title="22 _ NWR算法：如何修改读写模型以提升性能？" src="https://static001.geekbang.org/resource/audio/da/d5/da638bf0ab1924f7014b1259334f43d5.mp3" controls="controls"></audio> 
<p>你好，我是陶辉。</p><p>前两讲我们介绍数据库的扩展时，写请求仍然在操作中心化的Master单点，这在很多业务场景下都是不可接受的。这一讲我将介绍对于无单点的去中心化系统非常有用的NWR算法，它可以灵活地平衡一致性与性能。</p><p>最初我们仅在单机上部署数据库，一旦性能到达瓶颈，我们可以基于AKF Y轴将读写分离，这样多个Slave从库将读操作分流后，写操作就可以独享Master主库的全部性能。然而主库作为中心化的单点，一旦宕机，未及时同步到从库的数据就有可能丢失。而且，这一架构下，主库的故障还会导致整个系统瘫痪。</p><p>去中心化系统中没有“Master主库”这一概念，数据存放在多个Replication冗余节点上，且这些节点间地位均等，所以没有单点问题。为了保持强一致性，系统可以要求修改数据时，必须同时写入所有冗余节点，才能向客户端返回成功。但这样系统的可用性一定很成问题，毕竟大规模分布式系统中，出现故障是常态，写入全部节点的操作根本无法容错，任何1个节点宕机都会造成写操作失败。而且，同步节点过多也会导致写操作性能低下。</p><p>NWR算法提供了一个很棒的读写模型，可以解决上述问题。这里的“NWR”，是指在去中心化系统中将1份数据存放在N个节点上，每次操作时，写W个节点、读R个节点，只要调整W、R与N的关系，就能动态地平衡一致性与性能。NWR在NoSQL数据库中有很广泛的应用，比如Amazon的Dynamo和开源的Cassandra，这些数据库往往跨越多个IDC数据中心，包含成千上万个物理机节点，适用于海量数据的存储与处理。</p><!-- [[[read_end]]] --><p>这一讲，我们将介绍NWR算法的原理，包括它是怎样调整读写模型来提升性能的，以及Cassandra数据库是如何使用NWR算法的。</p><h2>从鸽巢原理到NWR算法</h2><p>NWR算法是由鸽巢原理得来的：如果10只鸽子放入9个鸽巢，那么有1个鸽巢内至少有2只鸽子，这就是鸽巢原理，如下图所示：</p><p><a href="https://zh.wikipedia.org/wiki/%E9%B4%BF%E5%B7%A2%E5%8E%9F%E7%90%86"><img src="https://static001.geekbang.org/resource/image/83/17/835a454f1ecb8d6edb5a1c2059082d17.jpg?wh=1246*1010" alt="" title="图片来源：https://zh.wikipedia.org/wiki/%E9%B4%BF%E5%B7%A2%E5%8E%9F%E7%90%86"></a></p><p>你可以用反证法证明它。鸽巢原理虽然简单，但它有许多很有用的推论。比如<a href="https://time.geekbang.org/column/article/232351">[第3课]</a> 介绍了很多解决哈希表冲突的方案，那么，哈希表有没有可能完全不出现冲突呢？<strong>鸽巢原理告诉我们，只要哈希函数输入主键的值范围大于输出索引，出现冲突的概率就一定大于0；只要存放元素的数量超过哈希桶的数量，就必然会发生冲突。</strong></p><p>基于鸽巢原理，David K. Gifford在1979年首次提出了<a href="https://en.wikipedia.org/wiki/Quorum_(distributed_computing)">Quorum</a> 算法（参见《<a href="https://dl.acm.org/doi/epdf/10.1145/800215.806583">Weighted Voting for Replicated Data</a>》论文），解决去中心化系统冗余数据的一致性问题。而Quorum算法提出，如果冗余数据存放在N个节点上，且每次写操作成功写入W个节点（其他N - W个节点将异步地同步数据），而读操作则从R个节点中选择并读出正确的数据，只要确保W + R &gt; N，同1条数据的读、写操作就不能并发执行，这样客户端就总能读到最新写入的数据。特别是当W &gt; N/2时，同1条数据的修改必然是顺序执行的。这样，分布式系统就具备了强一致性，这也是NWR算法的由来。</p><p>比如，若N为3，那么设置W和R为2时，在保障系统强一致性的同时，还允许3个节点中1个节点宕机后，系统仍然可以提供读、写服务，这样的系统具备了很高的可用性。当然，R和W的数值并不需要一致，如何调整它们，取决于读、写请求数量的比例。比如当N为5时，如果系统读多写少时，可以将W设为4，而R设为2，这样读操作的性能会更好。</p><p>NWR算法最早应用在Amazon推出的<a href="https://en.wikipedia.org/wiki/Dynamo_(storage_system)">Dynamo</a> 数据库中，你可以参见2007年Amazon发表的<a href="https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf">《Dynamo: Amazon’s Highly Available Key-value Store》</a>论文。2008年Dynamo的作者Avinash Lakshman跳槽到FaceBook，开发了Dynamo的开源版数据库<a href="https://zh.wikipedia.org/wiki/Cassandra">Cassandra</a>，它是目前最流行的NoSQL数据库之一，在Apple、Netflix、360等公司得到了广泛的应用。想必你对NWR算法的很多细节并不清楚，那么接下来我们以Cassandra为例，看看NWR是如何应用在实际工程中的。</p><h2>Cassandra数据库是如何使用NWR算法的？</h2><p>1个Cassandra分布式系统可以由多个IDC数据中心、数万个服务器节点构成，这些节点间使用RPC框架通信，由于Cassandra推出时gRPC（参见<a href="https://time.geekbang.org/column/article/247812">[第18课]</a>）还没有诞生，因此它使用的是性能相对较低的Thrift RPC框架（Thrift的优点是支持的开发语言更多）。同时，Cassandra虽然使用宽列存储模型（每行最多可以包含<a href="https://docs.datastax.com/en/cql-oss/3.x/cql/cql_reference/refLimits.html">20亿列</a>数据），但<strong>数据的分布是基于行Key进行的</strong>，它和Dynamo一样使用了一致性哈希算法，将Key对应的数据存储在多个节点中。关于一致性哈希算法，我们会在 [第24课] 再详细介绍。</p><p>Cassandra对客户端提供一种类SQL的<a href="https://cassandra.apache.org/doc/latest/cql/index.html">CQL</a> 语言，你可以使用下面这行CQL语句设定数据存储的冗余节点个数，也就是NWR算法中的N（也称为Replication Factor）：</p><pre><code>CREATE KEYSPACE excalibur
  WITH REPLICATION = {'class' : 'NetworkTopologyStrategy', 'dc1' : 3};
</code></pre><p>上面这行CQL语句设置了每行数据在数据中心DC1中存储3份冗余，即N = 3，接下来我们通过下面的CQL语句，将读R、写W的节点数都设置为1：</p><pre><code>cqlsh&gt; CONSISTENCY ONE
Consistency level set to ONE.
cqlsh&gt; CONSISTENCY
Current consistency level is ONE.
</code></pre><p><strong>此时，Cassandra的性能最高，但达成最终一致性的耗时最长，丢数据风险也最大。</strong>如果业务上对丢失少量数据不太在意，可以采用这种模型。此时修改数据时，客户端会并发地向3个存储节点写入数据，但只要1个节点返回成功，Cassandra就会向客户端返回写入成功，如下图所示：</p><p><a href="https://docs.datastax.com/en/cassandra-oss/3.x/cassandra/dml"><img src="https://static001.geekbang.org/resource/image/74/1d/742a430b92bb3b235294805b7073991d.png?wh=606*415" alt="" title="该图片及以下图片来源：https://docs.datastax.com/en/cassandra-oss/3.x/cassandra/dml"></a></p><p>上图中的系统由12个主机节点构成，由于数据采用一致性哈希算法分片，故构成了一个节点环。其中，本次写入的数据被分布到1、3、6这3个节点中存储。客户端可以随机连接到系统中的任何一个节点访问Cassandra，此时该节点被称为Coordinator Node，由它根据NWR的值来选择一致性模型，访问存储节点。</p><p>再来看读取数据的流程。下图中，作为Coordinator Node的节点10首先试图读取节点1中的数据，但发现节点1已经宕机，于是改选节点6并获取到数据，由于R = 1于是立刻向客户端返回成功。</p><p><img src="https://static001.geekbang.org/resource/image/50/d6/5038a63ce8a5cd23fcb6ba2e14b59cd6.jpg?wh=609*443" alt=""></p><p>如果我们将R、W都设置成2，这就满足了R + W &gt; N(3)的场景，此时系统具备了强一致性。客户端读写数据时，必须有2个节点返回，才算操作成功。比如下图中读取数据时，只有接收到节点1、节点6的返回，操作才算成功。</p><p><img src="https://static001.geekbang.org/resource/image/8d/0f/8dc00f0a82676cb54d21880e7b60c20f.jpg?wh=610*457" alt=""></p><p>上图中的蓝色线叫做Read repair，如果节点3上的数据不一致，那么本次读操作可以将它修复为正确的数据。说完正常场景，我们再来看当一个节点出现异常时，NWR是如何保持强一致性的。</p><p>下图中，客户端1在第2步，同时向3个存储节点写入了数据，由于节点1、3返回成功，所以写入操作实际已经完成了，但是节点6由于网络故障，却一直没有收到Coordinator Node发来的写入操作。在强一致性的约束下，客户端2在第5步发起的读请求，必须能获取到第2步写入的数据。然而，客户端2连接的Coordinator Node与客户端1不同，它选择了节点3和节点6，这两个节点上的数据并不一致。<strong>根据不同的timestamp时间戳，Coordinator Node发现节点3上的数据才是最后写入的数据，因此选择其上的数据返回客户端。这也叫Last-Write-Win策略。</strong></p><p><a href="https://blog.scottlogic.com/2017/10/06/cassandra-eventual-consistency.html"><img src="https://static001.geekbang.org/resource/image/4b/fa/4bc3308298395b7a57d9d540a79aa7fa.jpg?wh=7306*3486" alt="" title="图片来源：https://blog.scottlogic.com/2017/10/06/cassandra-eventual-consistency.html"></a></p><p>Cassandra提供了一个简单的方法，用于设置读写节点数量都过半，满足强一致性的要求，如下所示：</p><pre><code>cqlsh&gt; CONSISTENCY QUORUM
Consistency level set to QUORUM.
cqlsh&gt; CONSISTENCY
Current consistency level is QUORUM.
</code></pre><p>最后我们再来看看多数据中心的部署方式。下图中2个数据中心各设置N = 3，其中R、W则采用QUORUM一致性模型。当客户端发起写请求到达节点10这个Coordinator Node后，它选择本IDC Alpha的1、3、6节点存储数据，其中节点3、6返回成功后，IDC Alpha便更新成功。同时找到另一IDC Beta的节点11存储数据，并由节点11将数据同步给节点4和节点8。其中，只要节点4返回成功，IDC Beta也就成功更新了数据，此时Coordinator Node会向客户端返回写入成功。</p><p><img src="https://static001.geekbang.org/resource/image/5f/00/5fe2fa80bb20e04d25c41ed5986c0c00.jpg?wh=618*811" alt=""></p><p>读取数据时，这2个IDC内必须由4个存储节点返回数据，才满足QUORUM一致性的要求。下图中，Coordinator Node获取到了IDC Alpha中节点1、3、6的返回，以及IDC Beta中节点11的返回，就可以基于timestamp时间戳选择最新的数据返回客户端。而且Coordinator Node会并发地发起Read repair，试图修复IDC Beta中可能存在不一致的节点4和8。</p><p><img src="https://static001.geekbang.org/resource/image/59/19/59564438445fb26d2e8993a50a23df19.jpg?wh=614*892" alt=""></p><p>Cassandra还有许多一致性模型，比如LOCAL_QUORUM只要求本地IDC内有多数节点响应即可，而EACH_QUORUM则要求每个IDC内都必须有多数节点返回成功（注意，这与上图中IDC Alpha中有3个节点返回，而IDC Beta则只有1个节点返回的QUORUM是不同的）。你可以从<a href="https://docs.datastax.com/en/cassandra-oss/3.x/cassandra/dml/dmlConfigConsistency.html">这个页面</a>找到Cassandra支持的所有一致性模型，但无论如何变化，都只是在引入数据中心、机架等概念后，局部性地调节NWR而已。</p><h2>小结</h2><p>这一讲我们介绍了鸽巢原理，以及由此推导出的NWR算法，并以流行的NoSQL数据库Cassandra为例，介绍了NWR在分布式系统中的实践。</p><p>当鸽子的数量超过了鸽巢后，就要注定某一个鸽巢内一定含有两只以上的鸽子，同样的道理，只要读、写操作涉及的节点超过半数，就注定读写操作总包含一个含有正确数据的节点。NWR算法将这一原理一般化为：只要读节点数R + 写节点数W &gt; 存储节点数N，特别是W &gt; N/2时，就能使去中心的分布式系统获得强一致性。</p><p>支持上万节点的Cassandra数据库，就使用了NWR算法来保持一致性。当然，Cassandra支持多种一致性模型，当你需要更强劲的性能时，你可以令R + W &lt; N，当业务变化导致需要增强系统的一致性时，你可以实时地修改R、W。Cassandra也支持跨数据中心部署，此时的一致性模型更为复杂，但仍然将NWR算法作为实现基础。</p><h2>思考题</h2><p>最后给你留一道讨论题。你还知道哪些有状态服务使用了NWR算法吗？它与NWR在Cassandra中的应用有何不同？欢迎你在留言区中分享，也期待你能从大家的留言中总结出更一般化的规律。</p><p>感谢你的阅读，如果你觉得这节课对你有一些启发，也欢迎把它分享给你的朋友。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/7d/02/4862f849.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>杉松壁</span>
  </div>
  <div class="_2_QraFYR_0">kafka是不是也是NWR模型</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，Kafka在选举leader时，使用了简化版的NWR模型</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-02 09:59:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/17/39/3274257b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ple</span>
  </div>
  <div class="_2_QraFYR_0">老师的风格很不一样 很专业 硬核,当真说了一些其他地方不容易看到的. </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: ^_^</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-30 23:25:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/2f/ca/cbce6e94.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>梦想的优惠券</span>
  </div>
  <div class="_2_QraFYR_0">zk就有quorum机制，之前只知道NW😂，原来还需要R才能保证强一致性，，，<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: ^_^</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-08 12:56:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/09/9d/8af6cb1e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>臭猫</span>
  </div>
  <div class="_2_QraFYR_0">不同的进程请求写入，是如何协调顺序的呢？比如进程A要set X = 3;进程B要set X =4;</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 文稿中倒数第3张图，就是2个进程并发写入的，其中client1、client2就是2个进程，基于Last-Write-Win 策略来协调的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-30 11:31:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/f9/30/54c71bf9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>不会飞的海燕</span>
  </div>
  <div class="_2_QraFYR_0">根据鸽巢原理是不是可以推论出，把9只鸽子放到10个笼子里至少有一个笼子是空的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-26 11:11:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4a/15/106eaaa8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>stackWarn</span>
  </div>
  <div class="_2_QraFYR_0">keepalived 的配置 quorum up 估计也是同化了这个思想  </div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-18 10:27:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/50/2b/2344cdaa.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>第一装甲集群司令克莱斯特</span>
  </div>
  <div class="_2_QraFYR_0">真不错</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-08 10:02:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/f4/49/2add4f6b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>北极的大企鹅</span>
  </div>
  <div class="_2_QraFYR_0">感谢作者的耐心分享和指导</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-09 13:17:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/56/75/6bf38a1e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>坤哥</span>
  </div>
  <div class="_2_QraFYR_0">陶老师，异步同步剩下的n-w个节点没说呢？  last-write-win 前提各个服务时间戳一致。对于剩下的n-w个节点  谁作leader去同步呢？又怎样同步？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-09 16:30:18</div>
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
  <div class="_2_QraFYR_0">NWR算法总结：当鸽子的数量超过了鸽巢后，就要注定某一个鸽巢内一定含有两只以上的鸽子，同样的道理，只要读、写操作涉及的节点超过半数，就注定读写操作总包含一个含有正确数据的节点。NWR 算法将这一原理一般化为：只要读节点数 R + 写节点数 W &gt; 存储节点数 N，特别是 W &gt; N&#47;2 时，就能使去中心的分布式系统获得强一致性。<br>--------------------------------<br>鸽臼原理推导出的NWR一致性算法，真的大开眼界了。nice！<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: ^_^</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-08 18:20:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/75/dd/9ead6e69.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>黄海峰</span>
  </div>
  <div class="_2_QraFYR_0">最后一个多中心例子，“数据中心各设置 N = 3，其中 R、W 则采用 QUORUM 一致性模型”。。。r w采用quorum模型那意思到底是r w分别设为多少呢？？为何写的时候alpha要两个返回，beta中心一个返回就行了呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-01 16:40:23</div>
  </div>
</div>
</div>
</li>
</ul>