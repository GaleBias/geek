<audio title="18 _ 实战：如何基于Raft从0到1构建一个支持多存储引擎分布式KV服务？" src="https://static001.geekbang.org/resource/audio/85/11/859d136469cbee1454035d0086fe1011.mp3" controls="controls"></audio> 
<p>你好，我是唐聪。</p><p>通过前面课程的学习，我相信你已经对etcd基本架构、核心特性有了一定理解。如果让你基于Raft协议，实现一个简易的类etcd、支持多存储引擎的分布式KV服务，并能满足读多写少、读少写多的不同业务场景诉求，你知道该怎么动手吗？</p><p>纸上得来终觉浅，绝知此事要躬行。</p><p>今天我就和你聊聊如何实现一个类etcd、支持多存储引擎的KV服务，我们将基于etcd自带的<a href="https://github.com/etcd-io/etcd/tree/v3.4.9/contrib/raftexample">raftexample</a>项目快速构建它。</p><p>为了方便后面描述，我把它命名为metcd（表示微型的etcd），它是raftexample的加强版。希望通过metcd这个小小的实战项目，能够帮助你进一步理解etcd乃至分布式存储服务的核心架构、原理、典型问题解决方案。</p><p>同时在这个过程中，我将详细为你介绍etcd的Raft算法工程实现库、不同类型存储引擎的优缺点，拓宽你的知识视野，为你独立分析etcd源码，夯实基础。</p><h2>整体架构设计</h2><p>在和你深入聊代码细节之前，首先我和你从整体上介绍下系统架构。</p><p>下面是我给你画的metcd整体架构设计，它由API层、Raft层的共识模块、逻辑层及存储层组成的状态机组成。</p><p>接下来，我分别和你简要分析下API设计及复制状态机。</p><!-- [[[read_end]]] --><p><img src="https://static001.geekbang.org/resource/image/5e/03/5e9f6882a6f6e357e5c2c5yyffda4e03.png?wh=1920*1166" alt=""></p><h3>API设计</h3><p>API是软件系统对外的语言，它是应用编程接口的缩写，由一组接口定义和协议组成。</p><p>在设计API的时候，我们往往会考虑以下几个因素：</p><ul>
<li>性能。如etcd v2使用的是简单的HTTP/1.x，性能上无法满足大规模Kubernetes集群等场景的诉求，因此etcd v3使用的是基于HTTP/2的gRPC协议。</li>
<li>易用性、可调试性。如有的内部高并发服务为了满足性能等诉求，使用的是UDP协议。相比HTTP协议，UDP协议显然在易用性、可调试性上存在一定的差距。</li>
<li>开发效率、跨平台、可移植性。相比基于裸UDP、TCP协议设计的接口，如果你使用Protobuf等IDL语言，它支持跨平台、代码自动自动生成，开发效率更高。</li>
<li>安全性。如相比HTTP协议，使用HTTPS协议可对通信数据加密更安全，可适用于不安全的网络环境（比如公网传输）。</li>
<li>接口幂等性。幂等性简单来说，就是同样一个接口请求一次与多次的效果一样。若你的接口对外保证幂等性，则可降低使用者的复杂度。</li>
</ul><p>因为我们场景的是POC(Proof of concept)、Demo开发，因此在metcd项目中，我们优先考虑点是易用性、可调试性，选择HTTP/1.x协议，接口上为了满足key-value操作，支持Get和Put接口即可。</p><p>假设metcd项目使用3379端口，Put和Get接口，如下所示。</p><ul>
<li>Put接口，设置key-value</li>
</ul><pre><code>curl -L http://127.0.0.1:3379/hello -XPUT -d world
</code></pre><ul>
<li>Get接口，查询key-value</li>
</ul><pre><code>curl -L http://127.0.0.1:3379/hello
world
</code></pre><h3>复制状态机</h3><p>了解完API设计，那最核心的复制状态机是如何工作的呢？</p><p>我们知道etcd是基于下图复制状态机实现的分布式KV服务，复制状态机由共识模块、日志模块、状态机组成。</p><p><img src="https://static001.geekbang.org/resource/image/5c/4f/5c7a3079032f90120a6b309ee401fc4f.png?wh=605*319" alt=""></p><p>我们的实战项目metcd，也正是使用与之一样的模型，并且使用etcd项目中实现的Raft算法库作为共识模块，此算法库已被广泛应用在etcd、cockroachdb、dgraph等开源项目中。</p><p>以下是复制状态机的写请求流程：</p><ul>
<li>client发起一个写请求（put hello = world）；</li>
<li>server向Raft共识模块提交请求，共识模块生成一个写提案日志条目。若server是Leader，则把日志条目广播给其他节点，并持久化日志条目到WAL中；</li>
<li>当一半以上节点持久化日志条目后，Leader的共识模块将此日志条目标记为已提交（committed），并通知其他节点提交；</li>
<li>server从共识模块获取已经提交的日志条目，异步应用到状态机存储中（boltdb/leveldb/memory），然后返回给client。</li>
</ul><h3>多存储引擎</h3><p>了解完复制状态机模型后，我和你再深入介绍下状态机。状态机中最核心模块当然是存储引擎，那要如何同时支持多种存储引擎呢？</p><p>metcd项目将基于etcd本身自带的raftexample项目进行快速开发，而raftexample本身只支持内存存储。</p><p>因此我们通过将KV存储接口进行抽象化设计，实现支持多存储引擎。KVStore interface的定义如下所示。</p><pre><code>type KVStore interface {
   // LookUp get key value
   Lookup(key string) (string, bool)

   // Propose propose kv request into raft state machine
   Propose(k, v string)

   // ReadCommits consume entry from raft state machine into KvStore map until error
   ReadCommits(commitC &lt;-chan *string, errorC &lt;-chan error)

   // Snapshot return KvStore snapshot
   Snapshot() ([]byte, error)

   // RecoverFromSnapshot recover data from snapshot
   RecoverFromSnapshot(snapshot []byte) error

   // Close close backend databases
   Close() err
}
</code></pre><p>基于KV接口抽象化的设计，我们只需要针对具体的存储引擎，实现对应的操作即可。</p><p>我们期望支持三种存储引擎，分别是内存map、boltdb、leveldb，并做一系列简化设计。一组metcd实例，通过metcd启动时的配置来决定使用哪种存储引擎。不同业务场景不同实例，比如读多写少的存储引擎可使用boltdb，写多读少的可使用leveldb。</p><p>接下来我和你重点介绍下存储引擎的选型及原理。</p><h4>boltdb</h4><p>boltdb是一个基于B+ tree实现的存储引擎库，在<a href="https://time.geekbang.org/column/article/342527?utm_term=zeus18YAD&amp;utm_source=app&amp;utm_medium=geektime">10</a>中我已和你详细介绍过原理。</p><p>boltdb为什么适合读多写少？</p><p>对于读请求而言，一般情况下它可直接从内存中基于B+ tree遍历，快速获取数据返回给client，不涉及经过磁盘I/O。</p><p>对于写请求，它基于B+ tree查找写入位置，更新key-value。事务提交时，写请求包括B+ tree重平衡、分裂、持久化ditry page、持久化freelist、持久化meta page流程。同时，ditry page可能分布在文件的各个位置，它发起的是随机写磁盘I/O。</p><p>因此在boltdb中，完成一个写请求的开销相比读请求是大很多的。正如我在<a href="https://time.geekbang.org/column/article/345588">16</a>和<a href="https://time.geekbang.org/column/article/346471">17</a>中给你介绍的一样，一个3节点的8核16G空集群，线性读性能可以达到19万QPS，而写QPS仅为5万。</p><h4>leveldb</h4><p>那要如何设计适合写多读少的存储引擎呢?</p><p>最简单的思路当然是写内存最快。可是内存有限的，无法支撑大容量的数据存储，不持久化数据会丢失。</p><p>那能否直接将数据顺序追加到文件末尾（AOF）呢？因为磁盘的特点是顺序写性能比较快。</p><p>当然可以。<a href="https://en.wikipedia.org/wiki/Bitcask">Bitcask</a>存储模型就是采用AOF模式，把写请求顺序追加到文件。Facebook的图片存储<a href="https://www.usenix.org/legacy/event/osdi10/tech/full_papers/Beaver.pdf">Haystack</a>根据其论文介绍，也是使用类似的方案来解决大规模写入痛点。</p><p>那在AOF写入模型中如何实现查询数据呢？</p><p>很显然通过遍历文件一个个匹配key是可以的，但是它的性能是极差的。为了实现高性能的查询，最理想的解决方案从直接从内存中查询，但是内存是有限的，那么我们能否通过内存索引来记录一个key-value数据在文件中的偏移量，实现从磁盘快速读取呢？</p><p>是的，这正是<a href="https://en.wikipedia.org/wiki/Bitcask">Bitcask</a>存储模型的查询的实现，它通过内存哈希表维护各个key-value数据的索引，实现了快速查找key-value数据。不过，内存中虽然只保存key索引信息，但是当key较多的时候，其对内存要求依然比较高。</p><p>快速了解完存储引擎提升写性能的核心思路（随机写转化为顺序写）之后，那leveldb它的原理是怎样的呢？与Bitcask存储模型有什么不一样？</p><p>leveldb是基于LSM tree(log-structured merge-tree)实现的key-value存储，它的架构如下图所示（<a href="https://microsoft.github.io/MLOS/notebooks/LevelDbTuning/">引用自微软博客</a>）。</p><p>它提升写性能的核心思路同样是将随机写转化为顺序写磁盘WAL文件和内存，结合了我们上面讨论的写内存和磁盘两种方法。数据持久化到WAL文件是为了确保机器crash后数据不丢失。</p><p><img src="https://static001.geekbang.org/resource/image/05/50/05f01951fe5862a62624b81e2ceea150.png?wh=992*677" alt=""></p><p>那么它要如何解决内存不足和查询的痛点问题呢？</p><p>核心解决方案是分层的设计和基于一系列对象的转换和压缩。接下来我给你分析一下上面架构图写流程和后台compaction任务：</p><ul>
<li>首先写请求顺序写入Log文件(WAL)；</li>
<li>更新内存的Memtable。leveldb Memtable后端数据结构实现是skiplist，skiplist相比平衡二叉树，实现简单却同样拥有高性能的读写；</li>
<li>当Memtable达到一定的阈值时，转换成不可变的Memtable，也就是只读不可写；</li>
<li>leveldb后台Compact任务会将不可变的Memtable生成SSTable文件，它有序地存储一系列key-value数据。注意SST文件按写入时间进行了分层，Level层次越小数据越新。Manifest文件记录了各个SSTable文件处于哪个层级、它的最小与最大key范围；</li>
<li>当某个level下的SSTable文件数目超过一定阈值后，Compact任务会从这个level的SSTable中选择一个文件（level&gt;0），将其和高一层级的level+1的SSTable文件合并；</li>
<li>注意level 0是由Immutable直接生成的，因此level 0 SSTable文件中的key-value存在相互重叠。而level &gt; 0时，在和更高一层SSTable合并过程中，参与的SSTable文件是多个，leveldb会确保各个SSTable中的key-value不重叠。</li>
</ul><p>了解完写流程，读流程也就简单了，核心步骤如下：</p><ul>
<li>从Memtable跳跃表中查询key；</li>
<li>未找到则从Immutable中查找；</li>
<li>Immutable仍未命中，则按照leveldb的分层属性，因level 0 SSTable文件是直接从Immutable生成的，level 0存在特殊性，因此你需要从level 0遍历SSTable查找key；</li>
<li>level 0中若未命中，则从level 1乃至更高的层次查找。level大于0时，各个SSTable中的key是不存在相互重叠的。根据manifest记录的key-value范围信息，可快递定位到具体的SSTable。同时leveldb基于<a href="https://en.wikipedia.org/wiki/Bloom_filter">bloom filter</a>实现了快速筛选SSTable，因此查询效率较高。</li>
</ul><p>更详细原理你可以参考一下<a href="https://github.com/google/leveldb">leveldb</a>源码。</p><h2>实现分析</h2><p>从API设计、复制状态机、多存储引擎支持等几个方面你介绍了metcd架构设计后，接下来我就和你重点介绍下共识模块、状态机支持多存储引擎模块的核心实现要点。</p><h3>Raft算法库</h3><p>共识模块使用的是etcd <a href="https://github.com/etcd-io/etcd/tree/v3.4.9/raft">Raft算法库</a>，它是一个经过大量业务生产环境检验、具备良好可扩展性的共识算法库。</p><p>它提供了哪些接口给你使用? 如何提交一个提案，并且获取Raft共识模块输出结果呢？</p><h4>Raft API</h4><p>Raft作为一个库，它对外最核心的对象是一个名为<a href="https://github.com/etcd-io/etcd/blob/v3.4.9/raft/node.go#L125:L203">Node</a>的数据结构。Node表示Raft集群中的一个节点，它的输入与输出接口如下图所示，下面我重点和你介绍它的几个接口功能：</p><ul>
<li>Campaign，状态转换成Candidate，发起新一轮Leader选举；</li>
<li>Propose，提交提案接口；</li>
<li>Ready，Raft状态机输出接口，它的返回是一个输出Ready数据结构类型的管道，应用需要监听此管道，获取Ready数据，处理其中的各个消息（如持久化未提交的日志条目到WAL中，发送消息给其他节点等）；</li>
<li>Advance，通知Raft状态机，应用已处理上一个输出的Ready数据，等待发送下一个Ready数据；</li>
<li>TransferLeaderShip，尝试将Leader转移到某个节点；</li>
<li>Step，向Raft状态机提交收到的消息，比如当Leader广播完MsgApp消息给Follower节点后，Leader收到Follower节点回复的MsgAppResp消息时，就通过Step接口将此消息提交给Raft状态机驱动其工作；</li>
<li>ReadIndex，用于实现线性读。</li>
</ul><p><img src="https://static001.geekbang.org/resource/image/a7/39/a79a97f8cc8294dcb93f9552fb638f39.png?wh=1920*787" alt=""></p><p>上面提到的Raft状态机的输出<a href="https://github.com/etcd-io/etcd/blob/v3.4.9/raft/node.go#L52:L90">Ready结构</a>含有哪些信息呢? 下图是其详细字段，含义如下：</p><ul>
<li>SoftState，软状态。包括集群Leader和节点状态，不需要持久化到WAL；</li>
<li>pb.HardState，硬状态。与软状态相反，包括了节点当前Term、Vote等信息，需要持久化到WAL中；</li>
<li>ReadStates，用于线性一致性读；</li>
<li>Entries，在向其他节点发送消息之前需持久化到WAL中；</li>
<li>Messages，持久化Entries后，发送给其他节点的消息；</li>
<li>Committed Entries，已提交的日志条目，需要应用到存储状态机中；</li>
<li>Snapshot，快照需保存到持久化存储中；</li>
<li>MustSync，HardState和Entries是否要持久化到WAL中；</li>
</ul><p><img src="https://static001.geekbang.org/resource/image/c0/d6/c0f0b8046a7c8c67c277fed9548251d6.png?wh=1920*786" alt=""></p><p>了解完API后，我们接下来继续看看代码如何使用Raft的Node API。</p><p>正如我在<a href="https://time.geekbang.org/column/article/337604">04</a>中和你介绍的，etcd Raft库的设计抽象了网络、Raft日志存储等模块，它本身并不会进行网络、存储相关的操作，上层应用需结合自己业务场景选择内置的模块或自定义实现网络、存储、日志等模块。</p><p>因此我们在使用Raft库时，需要先自定义好相关网络、存储等模块，再结合上面介绍的Raft Node API，就可以完成一个Node的核心操作了。其数据结构定义如下：</p><pre><code>// A key-value stream backed by raft
type raftNode struct {
   proposeC    &lt;-chan string            // proposed messages (k,v)
   confChangeC &lt;-chan raftpb.ConfChange // proposed cluster config changes
   commitC     chan&lt;- *string           // entries committed to log (k,v)
   errorC      chan&lt;- error             // errors from raft session
   id          int      // client ID for raft session
   ......
   node        raft.Node
   raftStorage *raft.MemoryStorage
   wal         *wal.WAL
   transport *rafthttp.Transport
}
</code></pre><p>这个数据结构名字叫raftNode，它表示Raft集群中的一个节点。它是由我们业务应用层设计的一个组合结构。从结构体定义中你可以看到它包含了Raft核心数据结构Node(raft.Node)、Raft日志条目内存存储模块(raft.MemoryStorage）、WAL持久化模块(wal.WAL)以及网络模块(rafthttp.Transport)。</p><p>同时，它提供了三个核心的管道与业务逻辑模块、存储状态机交互：</p><ul>
<li>proposeC，它用来接收client发送的写请求提案消息；</li>
<li>confChangeC，它用来接收集群配置变化消息；</li>
<li>commitC，它用来输出Raft共识模块已提交的日志条目消息。</li>
</ul><p>在metcd项目中因为我们是直接基于raftexample定制开发，因此日志持久化存储、网络都使用的是etcd自带的WAL和rafthttp模块。</p><p><a href="https://github.com/etcd-io/etcd/blob/v3.4.9/wal/wal.go">WAL</a>模块中提供了核心的保存未持久化的日志条目和快照功能接口，你可以参考<a href="https://time.geekbang.org/column/article/336766">03</a>节写请求中我和你介绍的原理。</p><p><a href="https://github.com/etcd-io/etcd/tree/v3.4.9/etcdserver/api/rafthttp">rafthttp</a>模块基于HTTP协议提供了各个节点间的消息发送能力，metcd使用如下：</p><pre><code>rc.transport = &amp;rafthttp.Transport{
   Logger:      zap.NewExample(),
   ID:          types.ID(rc.id),
   ClusterID:   0x1000,
   Raft:        rc,
   ServerStats: stats.NewServerStats(&quot;&quot;, &quot;&quot;),
   LeaderStats: stats.NewLeaderStats(strconv.Itoa(rc.id)),
   ErrorC:      make(chan error),
}
</code></pre><p>搞清楚Raft模块的输入、输出API，设计好raftNode结构，复用etcd的WAL、网络等模块后，接下来我们就只需要实现如下两个循环逻辑，处理业务层发送给proposeC和confChangeC消息、将Raft的Node输出Ready结构进行相对应的处理即可。精简后的代码如下所示：</p><pre><code>func (rc *raftNode) serveChannels() {
   // send proposals over raft
   go func() {
      confChangeCount := uint64(0)
      for rc.proposeC != nil &amp;&amp; rc.confChangeC != nil {
         select {
         case prop, ok := &lt;-rc.proposeC:
            if !ok {
               rc.proposeC = nil
            } else {
               // blocks until accepted by raft state machine
               rc.node.Propose(context.TODO(), []byte(prop))
            }

         case cc, ok := &lt;-rc.confChangeC:
            if !ok {
               rc.confChangeC = nil
            } else {
               confChangeCount++
               cc.ID = confChangeCount
               rc.node.ProposeConfChange(context.TODO(), cc)
            }
         }
      }
   }()

   // event loop on raft state machine updates
   for {
      select {
      case &lt;-ticker.C:
         rc.node.Tick()

      // store raft entries to wal, then publish over commit channel
      case rd := &lt;-rc.node.Ready():
         rc.wal.Save(rd.HardState, rd.Entries)
         if !raft.IsEmptySnap(rd.Snapshot) {
            rc.saveSnap(rd.Snapshot)
            rc.raftStorage.ApplySnapshot(rd.Snapshot)
            rc.publishSnapshot(rd.Snapshot)
         }
         rc.raftStorage.Append(rd.Entries)
         rc.transport.Send(rd.Messages)
         if ok := rc.publishEntries(rc.entriesToApply(rd.CommittedEntries)); !ok {
            rc.stop()
            return
         }
         rc.maybeTriggerSnapshot()
         rc.node.Advance()
      }
   }
}
</code></pre><p>代码简要分析如下：</p><ul>
<li>从proposeC中取出提案消息，通过raft.Node.Propose API提交提案；</li>
<li>从confChangeC取出配置变更消息，通过raft.Node.ProposeConfChange API提交配置变化消息；</li>
<li>从raft.Node中获取Raft算法状态机输出到Ready结构中，将rd.Entries和rd.HardState通过WAL模块持久化，将rd.Messages通过rafthttp模块，发送给其他节点。将rd.CommittedEntries应用到业务存储状态机。</li>
</ul><p>以上就是Raft实现的核心流程，接下来我来和你聊聊业务存储状态机。</p><h3>支持多存储引擎</h3><p>在整体架构设计时，我和你介绍了为了使metcd项目能支撑多存储引擎，我们将KVStore进行了抽象化设计，因此我们只需要实现各个存储引擎相对应的API即可。</p><p>这里我以Put接口为案例，分别给你介绍下各个存储引擎的实现。</p><h4>boltdb</h4><p>首先是boltdb存储引擎，它的实现如下，你也可以去<a href="https://time.geekbang.org/column/article/342527">10</a>里回顾一下它的API和原理。</p><pre><code>func (s *boltdbKVStore) Put(key, value string) error {
   s.mu.Lock()
   defer s.mu.Unlock()
   // Start a writable transaction.
   tx, err := s.db.Begin(true)
   if err != nil {
      return err
   }
   defer tx.Rollback()

   // Use the transaction...
   bucket, err := tx.CreateBucketIfNotExists([]byte(&quot;keys&quot;))
   if err != nil {
      log.Printf(&quot;failed to put key %s, value %s, err is %v&quot;, key, value, err)
      return err
   }
   err = bucket.Put([]byte(key), []byte(value))
   if err != nil {
      log.Printf(&quot;failed to put key %s, value %s, err is %v&quot;, key, value, err)
      return err
   }

   // Commit the transaction and check for error.
   if err := tx.Commit(); err != nil {
      log.Printf(&quot;failed to commit transaction, key %s, err is %v&quot;, key, err)
      return err
   }
   log.Printf(&quot;backend:%s,put key:%s,value:%s succ&quot;, s.config.backend, key, value)
   return nil
</code></pre><h4>leveldb</h4><p>其次是leveldb，我们使用的是<a href="https://github.com/syndtr/goleveldb">goleveldb</a>，它基于Google开源的c++ <a href="https://github.com/google/leveldb">leveldb</a>版本实现。它提供的常用API如下所示。</p><ul>
<li>通过OpenFile API创建或打开一个leveldb数据库。</li>
</ul><pre><code>db, err := leveldb.OpenFile(&quot;path/to/db&quot;, nil)
...
defer db.Close()
</code></pre><ul>
<li>通过DB.Get/Put/Delete API操作数据。</li>
</ul><pre><code>data, err := db.Get([]byte(&quot;key&quot;), nil)
...
err = db.Put([]byte(&quot;key&quot;), []byte(&quot;value&quot;), nil)
...
err = db.Delete([]byte(&quot;key&quot;), nil)
...
</code></pre><p>了解其接口后，通过goleveldb的库，client调用就非常简单了，下面是metcd项目中，leveldb存储引擎Put接口的实现。</p><pre><code>func (s *leveldbKVStore) Put(key, value string) error {
   err := s.db.Put([]byte(key), []byte(value), nil)
   if err != nil {
      log.Printf(&quot;failed to put key %s, value %s, err is %v&quot;, key, value, err)
      return err
   }
   log.Printf(&quot;backend:%s,put key:%s,value:%s succ&quot;, s.config.backend, key, value)
   return nil
}
</code></pre><h3>读写流程</h3><p>介绍完在metcd项目中如何使用Raft共识模块、支持多存储引擎后，我们再从整体上介绍下在metcd中写入和读取一个key-value的流程。</p><h4>写流程</h4><p>当你通过如下curl命令发起一个写操作时，写流程如下面架构图序号所示:</p><pre><code>curl -L http://127.0.0.1:3379/hello -XPUT -d world

</code></pre><ul>
<li>client通过curl发送HTTP PUT请求到server；</li>
<li>server收到后，将消息写入到KVStore的ProposeC管道；</li>
<li>raftNode循环逻辑将消息通过Raft模块的Propose接口提交；</li>
<li>Raft模块输出Ready结构，server将日志条目持久化后，并发送给其他节点；</li>
<li>集群多数节点持久化此日志条目后，这个日志条目被提交给存储状态机KVStore执行；</li>
<li>KVStore根据启动的backend存储引擎名称，调用对应的Put接口即可。</li>
</ul><p><img src="https://static001.geekbang.org/resource/image/9b/c1/9b84a7e312165de46749e1c4046fc9c1.png?wh=1920*1135" alt=""></p><h4>读流程</h4><p>当你通过如下curl命令发起一个读操作时，读流程如下面架构图序号所示：</p><pre><code>curl -L http://127.0.0.1:3379/hello
world
</code></pre><ul>
<li>client通过curl发送HTTP Get请求到server；</li>
<li>server收到后，根据KVStore的存储引擎，从后端查询出对应的key-value数据。</li>
</ul><p><img src="https://static001.geekbang.org/resource/image/17/b2/1746fbd9e9435d8607e44bea2d2c39b2.png?wh=1920*1187" alt=""></p><h2>小结</h2><p>最后，我来总结下我们今天的内容。我这节课分别从整体架构设计和实现分析，给你介绍了如何基于Raft从0到1构建一个支持多存储引擎的分布式key-value数据库。</p><p>在整体架构设计上，我给你介绍了API设计核心因素，它们分别是性能、易用性、开发效率、安全性、幂等性。其次我和你介绍了复制状态机的原理，它由共识模块、日志模块、存储状态机模块组成。最后我和你深入分析了多存储引擎设计，重点介绍了leveldb原理，它将随机写转换为顺序写日志和内存，通过一系列分层、创新的设计实现了优异的写性能，适合读少写多。</p><p>在实现分析上，我和你重点介绍了Raft算法库的核心对象Node API。对于一个库而言，我们重点关注的是其输入、输出接口，业务逻辑层可通过Propose接口提交提案，通过Ready结构获取Raft算法状态机的输出内容。其次我和你介绍了Raft算法库如何与WAL模块、Raft日志存储模块、网络模块协作完成一个写请求。</p><p>最后为了支持多存储引擎，我们分别基于boltdb、leveldb实现了KVStore相关接口操作，并通过读写流程图，从整体上为你介绍了一个读写请求在metcd中是如何工作的。</p><p>麻雀虽小，五脏俱全。希望能通过这个迷你项目解答你对如何构建一个简易分布式KV服务的疑问，以及让你对etcd的工作原理有更深的理解。</p><h2>思考题</h2><p>你知道<a href="https://github.com/etcd-io/etcd/tree/v3.4.9/contrib/raftexample">raftexample</a>启动的时候是如何工作的吗？它的存储引擎内存map是如何保证数据不丢失的呢？</p><p>感谢你的阅读，如果你认为这节课的内容有收获，也欢迎把它分享给你的朋友，我们下一讲见。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/22/3a/de/e5c30589.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>云原生工程师</span>
  </div>
  <div class="_2_QraFYR_0">设计上存储引擎的介绍，获益匪浅，具体实现上的解读，也搞清了之前几个疑问，etcd足够轻量级，简直就是学习分布式系统的最佳案例，后面抽空自己基于raft搞个小小项目，进一步加深下，老师在这块有什么分布式书籍推荐没</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，自己动手会有更深的理解，书籍推荐《Designing Data-Intensive Applications》，中文名《设计数据密集型应用》，豆瓣评分高达9.7，https:&#47;&#47;book.douban.com&#47;subject&#47;30329536&#47;，非常不错的分布式入门书籍。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-02 08:42:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/2a/b9/2bf8cc89.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>无名氏</span>
  </div>
  <div class="_2_QraFYR_0">大佬 有完整demo可以分享吗😀</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-08 13:01:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/29/36/7d/eab0d26a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>丹尼尔-雪碧</span>
  </div>
  <div class="_2_QraFYR_0">请问大佬，raft模块返回的快照信息，快照是从哪里来的？我理解快照就是从kvStorage来的？所以是当需要生成快照的时候，raft模块调用kvStorage的接口生成快照？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-31 00:44:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/49/fc/5627215c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小何</span>
  </div>
  <div class="_2_QraFYR_0">这个实现的代码在哪里可以下载？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-28 21:22:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/bc/fc/cf98963a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lixg</span>
  </div>
  <div class="_2_QraFYR_0">文中提到“Messages，持久化 Entries 后，发送给其他节点的消息”<br><br>这里的消息是指什么消息，包含什么内容？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-14 21:51:20</div>
  </div>
</div>
</div>
</li>
</ul>