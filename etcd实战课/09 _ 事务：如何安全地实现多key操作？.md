<audio title="09 _ 事务：如何安全地实现多key操作？" src="https://static001.geekbang.org/resource/audio/f2/a7/f2fe6bd7faae53297a3a389899f525a7.mp3" controls="controls"></audio> 
<p>你好，我是唐聪。</p><p>在软件开发过程中，我们经常会遇到需要批量执行多个key操作的业务场景，比如转账案例中，Alice给Bob转账100元，Alice账号减少100，Bob账号增加100，这涉及到多个key的原子更新。</p><p>无论发生任何故障，我们应用层期望的结果是，要么两个操作一起成功，要么两个一起失败。我们无法容忍出现一个成功，一个失败的情况。那么etcd是如何解决多key原子更新问题呢？</p><p>这正是我今天要和你分享的主题——事务，它就是为了<strong>简化应用层的编程模型</strong>而诞生的。我将通过转账案例为你剖析etcd事务实现，让你了解etcd如何实现事务ACID特性的，以及MVCC版本号在事务中的重要作用。希望通过本节课，帮助你在业务开发中正确使用事务，保证软件代码的正确性。</p><h2>事务特性初体验及API</h2><p>如何使用etcd实现Alice向Bob转账功能呢？</p><p>在etcd v2的时候， etcd提供了CAS（Compare and swap），然而其只支持单key，不支持多key，因此无法满足类似转账场景的需求。严格意义上说CAS称不上事务，无法实现事务的各个隔离级别。</p><p>etcd v3为了解决多key的原子操作问题，提供了全新迷你事务API，同时基于MVCC版本号，它可以实现各种隔离级别的事务。它的基本结构如下：</p><!-- [[[read_end]]] --><pre><code>client.Txn(ctx).If(cmp1, cmp2, ...).Then(op1, op2, ...,).Else(op1, op2, …)
</code></pre><p>从上面结构中你可以看到，<strong>事务API由If语句、Then语句、Else语句组成</strong>，这与我们平时常见的MySQL事务完全不一样。</p><p>它的基本原理是，在If语句中，你可以添加一系列的条件表达式，若条件表达式全部通过检查，则执行Then语句的get/put/delete等操作，否则执行Else的get/put/delete等操作。</p><p>那么If语句支持哪些检查项呢？</p><p>首先是<strong>key的最近一次修改版本号mod_revision</strong>，简称mod。你可以通过它检查key最近一次被修改时的版本号是否符合你的预期。比如当你查询到Alice账号资金为100元时，它的mod_revision是v1，当你发起转账操作时，你得确保Alice账号上的100元未被挪用，这就可以通过mod(“Alice”) = “v1” 条件表达式来保障转账安全性。</p><p>其次是<strong>key的创建版本号create_revision</strong>，简称create。你可以通过它检查key是否已存在。比如在分布式锁场景里，只有分布式锁key(lock)不存在的时候，你才能发起put操作创建锁，这时你可以通过create(“lock”) = "0"来判断，因为一个key不存在的话它的create_revision版本号就是0。</p><p>接着是<strong>key的修改次数version</strong>。你可以通过它检查key的修改次数是否符合预期。比如你期望key在修改次数小于3时，才能发起某些操作时，可以通过version(“key”) &lt; "3"来判断。</p><p>最后是<strong>key的value值</strong>。你可以通过检查key的value值是否符合预期，然后发起某些操作。比如期望Alice的账号资金为200, value(“Alice”) = “200”。</p><p>If语句通过以上MVCC版本号、value值、各种比较运算符(等于、大于、小于、不等于)，实现了灵活的比较的功能，满足你各类业务场景诉求。</p><p>下面我给出了一个使用etcdctl的txn事务命令，基于以上介绍的特性，初步实现的一个Alice向Bob转账100元的事务。</p><p>Alice和Bob初始账上资金分别都为200元，事务首先判断Alice账号资金是否为200，若是则执行转账操作，不是则返回最新资金。etcd是如何执行这个事务的呢？<strong>这个事务实现上有哪些问题呢？</strong></p><pre><code>$ etcdctl txn -i
compares: //对应If语句
value(&quot;Alice&quot;) = &quot;200&quot; //判断Alice账号资金是否为200


success requests (get, put, del): //对应Then语句
put Alice 100 //Alice账号初始资金200减100
put Bob 300 //Bob账号初始资金200加100


failure requests (get, put, del): //对应Else语句
get Alice  
get Bob


SUCCESS


OK

OK

</code></pre><h2>整体流程</h2><p><img src="https://static001.geekbang.org/resource/image/e4/d3/e41a4f83bda29599efcf06f6012b0bd3.png?wh=1920*852" alt=""></p><p>在和你介绍上面案例中的etcd事务原理和问题前，我先给你介绍下事务的整体流程，为我们后面介绍etcd事务ACID特性的实现做准备。</p><p>上图是etcd事务的执行流程，当你通过client发起一个txn转账事务操作时，通过gRPC KV Server、Raft模块处理后，在Apply模块执行此事务的时候，它首先对你的事务的If语句进行检查，也就是ApplyCompares操作，如果通过此操作，则执行ApplyTxn/Then语句，否则执行ApplyTxn/Else语句。</p><p>在执行以上操作过程中，它会根据事务是否只读、可写，通过MVCC层的读写事务对象，执行事务中的get/put/delete各操作，也就是我们上一节课介绍的MVCC对key的读写原理。</p><h2>事务ACID特性</h2><p>了解完事务的整体执行流程后，那么etcd应该如何正确实现上面案例中Alice向Bob转账的事务呢？别着急，我们先来了解一下事务的ACID特性。在你了解了etcd事务ACID特性实现后，这个转账事务案例的正确解决方案也就简单了。</p><p>ACID是衡量事务的四个特性，由原子性（Atomicity）、一致性（Consistency）、隔离性（Isolation）、持久性（Durability）组成。接下来我就为你分析ACID特性在etcd中的实现。</p><h3>原子性与持久性</h3><p>事务的原子性（Atomicity）是指在一个事务中，所有请求要么同时成功，要么同时失败。比如在我们的转账案例中，是绝对无法容忍Alice账号扣款成功，但是Bob账号资金到账失败的场景。</p><p>持久性（Durability）是指事务一旦提交，其所做的修改会永久保存在数据库。</p><p>软件系统在运行过程中会遇到各种各样的软硬件故障，如果etcd在执行上面事务过程中，刚执行完扣款命令（put Alice 100）就突然crash了，它是如何保证转账事务的原子性与持久性的呢？</p><p><img src="https://static001.geekbang.org/resource/image/cf/9e/cf94ce8fc0649fe5cce45f8b7468019e.png?wh=1920*949" alt=""></p><p>如上图转账事务流程图所示，etcd在执行一个事务过程中，任何时间点都可能会出现节点crash等异常问题。我在图中给你标注了两个关键的异常时间点，它们分别是T1和T2。接下来我分别为你分析一下etcd在这两个关键时间点异常后，是如何保证事务的原子性和持久性的。</p><h4>T1时间点</h4><p>T1时间点是在Alice账号扣款100元完成时，Bob账号资金还未成功增加时突然发生了crash。</p><p>从前面介绍的etcd写原理和上面流程图我们可知，此时MVCC写事务持有boltdb写锁，仅是将修改提交到了内存中，保证幂等性、防止日志条目重复执行的一致性索引consistent index也并未更新。同时，负责boltdb事务提交的goroutine因无法持有写锁，也并未将事务提交到持久化存储中。</p><p>因此，T1时间点发生crash异常后，事务并未成功执行和持久化任意数据到磁盘上。在节点重启时，etcd  server会重放WAL中的已提交日志条目，再次执行以上转账事务。因此不会出现Alice扣款成功、Bob到帐失败等严重Bug，极大简化了业务的编程复杂度。</p><h4>T2时间点</h4><p>T2时间点是在MVCC写事务完成转账，server返回给client转账成功后，boltdb的事务提交goroutine，批量将事务持久化到磁盘中时发生了crash。这时etcd又是如何保证原子性和持久性的呢?</p><p>我们知道一致性索引consistent index字段值是和key-value数据在一个boltdb事务里同时持久化到磁盘中的。若在boltdb事务提交过程中发生crash了，简单情况是consistent index和key-value数据都更新失败。那么当节点重启，etcd  server重放WAL中已提交日志条目时，同样会再次应用转账事务到状态机中，因此事务的原子性和持久化依然能得到保证。</p><p>更复杂的情况是，当boltdb提交事务的时候，会不会部分数据提交成功，部分数据提交失败呢？这个问题，我将在下一节课通过深入介绍boltdb为你解答。</p><p>了解完etcd事务的原子性和持久性后，那一致性又是怎么一回事呢？事务的一致性难道是指各个节点数据一致性吗？</p><h3>一致性</h3><p>在软件系统中，到处可见一致性（Consistency）的表述，其实在不同场景下，它的含义是不一样的。</p><p>首先分布式系统中多副本数据一致性，它是指各个副本之间的数据是否一致，比如Redis的主备是异步复制的，那么它的一致性是最终一致性的。</p><p>其次是CAP原理中的一致性是指可线性化。核心原理是虽然整个系统是由多副本组成，但是通过线性化能力支持，对client而言就如一个副本，应用程序无需关心系统有多少个副本。</p><p>然后是一致性哈希，它是一种分布式系统中的数据分片算法，具备良好的分散性、平衡性。</p><p>最后是事务中的一致性，它是指事务变更前后，数据库必须满足若干恒等条件的状态约束，<strong>一致性往往是由数据库和业务程序两方面来保障的</strong>。</p><p><strong>在Alice向Bob转账的案例中有哪些恒等状态呢？</strong></p><p>很明显，转账系统内的各账号资金总额，在转账前后应该一致，同时各账号资产不能小于0。</p><p>为了帮助你更好地理解前面转账事务实现的问题，下面我给你画了幅两个并发转账事务的流程图。</p><p>图中有两个并发的转账事务，Mike向Bob转账100元，Alice也向Bob转账100元，按照我们上面的事务实现，从下图可知转账前系统总资金是600元，转账后却只有500元了，因此它无法保证转账前后账号系统内的资产一致性，导致了资产凭空消失，破坏了事务的一致性。</p><p><img src="https://static001.geekbang.org/resource/image/1f/ea/1ff951756c0ffc427e5a064e3cf8caea.png?wh=1920*1153" alt=""></p><p>事务一致性被破坏的根本原因是，事务中缺少对Bob账号资产是否发生变化的判断，这就导致账号资金被覆盖。</p><p>为了确保事务的一致性，一方面，业务程序在转账逻辑里面，需检查转账者资产大于等于转账金额。在事务提交时，通过账号资产的版本号，确保双方账号资产未被其他事务修改。若双方账号资产被其他事务修改，账号资产版本号会检查失败，这时业务可以通过获取最新的资产和版本号，发起新的转账事务流程解决。</p><p>另一方面，etcd会通过WAL日志和consistent index、boltdb事务特性，去确保事务的原子性，因此不会有部分成功部分失败的操作，导致资金凭空消失、新增。</p><p>介绍完事务的原子性和持久化、一致性后，我们再看看etcd又是如何提供各种隔离级别的事务，在转账过程中，其他client能看到转账的中间状态吗(如Alice扣款成功，Bob还未增加时)？</p><h3>隔离性</h3><p>ACID中的I是指Isolation，也就是事务的隔离性，它是指事务在执行过程中的可见性。常见的事务隔离级别有以下四种。</p><p>首先是<strong>未提交读</strong>（Read UnCommitted），也就是一个client能读取到未提交的事务。比如转账事务过程中，Alice账号资金扣除后，Bob账号上资金还未增加，这时如果其他client读取到这种中间状态，它会发现系统总金额钱减少了，破坏了事务一致性的约束。</p><p>其次是<strong>已提交读</strong>（Read Committed），指的是只能读取到已经提交的事务数据，但是存在不可重复读的问题。比如事务开始时，你读取了Alice和Bob资金，这时其他事务修改Alice和Bob账号上的资金，你在事务中再次读取时会读取到最新资金，导致两次读取结果不一样。</p><p>接着是<strong>可重复读</strong>（Repeated Read），它是指在一个事务中，同一个读操作get Alice/Bob在事务的任意时刻都能得到同样的结果，其他修改事务提交后也不会影响你本事务所看到的结果。</p><p>最后是<strong>串行化</strong>（Serializable），它是最高的事务隔离级别，读写相互阻塞，通过牺牲并发能力、串行化来解决事务并发更新过程中的隔离问题。对于串行化我要和你特别补充一点，很多人认为它都是通过读写锁，来实现事务一个个串行提交的，其实这只是在基于锁的并发控制数据库系统实现而已。<strong>为了优化性能，在基于MVCC机制实现的各个数据库系统中，提供了一个名为“可串行化的快照隔离”级别，相比悲观锁而言，它是一种乐观并发控制，通过快照技术实现的类似串行化的效果，事务提交时能检查是否冲突。</strong></p><p>下面我重点和你介绍下未提交读、已提交读、可重复读、串行化快照隔离。</p><h4>未提交读</h4><p>首先是最低的事务隔离级别，未提交读。我们通过如下一个转账事务时间序列图，来分析下一个client能否读取到未提交事务修改的数据，是否存在脏读。</p><p><img src="https://static001.geekbang.org/resource/image/6a/8d/6a526be4949a383fd5263484c706d68d.png?wh=1920*786" alt=""></p><p>图中有两个事务，一个是用户查询Alice和Bob资产的事务，一个是我们执行Alice向Bob转账的事务。</p><p>如图中所示，若在Alice向Bob转账事务执行过程中，etcd server收到了client查询Alice和Bob资产的读请求，显然此时我们无法接受client能读取到一个未提交的事务，因为这对应用程序而言会产生严重的BUG。那么etcd是如何保证不出现这种场景呢？</p><p>我们知道etcd基于boltdb实现读写操作的，读请求由boltdb的读事务处理，你可以理解为快照读。写请求由boltdb写事务处理，etcd定时将一批写操作提交到boltdb并清空buffer。</p><p>由于etcd是批量提交写事务的，而读事务又是快照读，因此当MVCC写事务完成时，它需要更新buffer，这样下一个读请求到达时，才能从buffer中获取到最新数据。</p><p>在我们的场景中，转账事务并未结束，执行put Alice为100的操作不会回写buffer，因此避免了脏读的可能性。用户此刻从boltdb快照读事务中查询到的Alice和Bob资产都为200。</p><p>从以上分析可知，etcd并未使用悲观锁来解决脏读的问题，而是通过MVCC机制来实现读写不阻塞，并解决脏读的问题。</p><h4>已提交读、可重复读</h4><p>比未提交读隔离级别更高的是已提交读，它是指在事务中能读取到已提交数据，但是存在不可重复读的问题。已提交读，也就是说你每次读操作，若未增加任何版本号限制，默认都是当前读，etcd会返回最新已提交的事务结果给你。</p><p>如何理解不可重复读呢?</p><p>在上面用户查询Alice和Bob事务的案例中，第一次查出来资产都是200，第二次是Alice为100，Bob为300，通过读已提交模式，你能及时获取到etcd最新已提交的事务结果，但是出现了不可重复读，两次读出来的Alice和Bob资产不一致。</p><p>那么如何实现可重复读呢？</p><p>你可以通过MVCC快照读，或者参考etcd的事务框架STM实现，它在事务中维护一个读缓存，优先从读缓存中查找，不存在则从etcd查询并更新到缓存中，这样事务中后续读请求都可从缓存中查找，确保了可重复读。</p><p>最后我们再来重点介绍下什么是串行化快照隔离。</p><h4>串行化快照隔离</h4><p>串行化快照隔离是最严格的事务隔离级别，它是指在在事务刚开始时，首先获取etcd当前的版本号rev，事务中后续发出的读请求都带上这个版本号rev，告诉etcd你需要获取那个时间点的快照数据，etcd的MVCC机制就能确保事务中能读取到同一时刻的数据。</p><p><strong>同时，它还要确保事务提交时，你读写的数据都是最新的，未被其他人修改，也就是要增加冲突检测机制。</strong>当事务提交出现冲突的时候依赖client重试解决，安全地实现多key原子更新。</p><p>那么我们应该如何为上面一致性案例中，两个并发转账的事务，增加冲突检测机制呢？</p><p>核心就是我们前面介绍MVCC的版本号，我通过下面的并发转账事务流程图为你解释它是如何工作的。</p><p><img src="https://static001.geekbang.org/resource/image/3b/26/3b4c7fb43e03a38aceb2a8c2d5c92226.png?wh=1920*1011" alt=""></p><p>如上图所示，事务A，Alice向Bob转账100元，事务B，Mike向Bob转账100元，两个事务同时发起转账操作。</p><p>一开始时，Mike的版本号(指mod_revision)是4，Bob版本号是3，Alice版本号是2，资产各自200。为了防止并发写事务冲突，etcd在一个写事务开始时，会独占一个MVCC读写锁。</p><p>事务A会先去etcd查询当前Alice和Bob的资产版本号，用于在事务提交时做冲突检测。在事务A查询后，事务B获得MVCC写锁并完成转账事务，Mike和Bob账号资产分别为100，300，版本号都为5。</p><p>事务B完成后，事务A获得写锁，开始执行事务。</p><p>为了解决并发事务冲突问题，事务A中增加了冲突检测，期望的Alice版本号应为2，Bob为3。结果事务B的修改导致Bob版本号变成了5，因此此事务会执行失败分支，再次查询Alice和Bob版本号和资产，发起新的转账事务，成功通过MVCC冲突检测规则mod(“Alice”) = 2 和 mod(“Bob”) = 5 后，更新Alice账号资产为100，Bob资产为400，完成转账操作。</p><p>通过上面介绍的快照读和MVCC冲突检测检测机制，etcd就可实现串行化快照隔离能力。</p><h3>转账案例应用</h3><p>介绍完etcd事务ACID特性实现后，你很容易发现事务特性初体验中的案例问题了，它缺少了完整事务的冲突检测机制。</p><p>首先你可通过一个事务获取Alice和Bob账号的上资金和版本号，用以判断Alice是否有足够的金额转账给Bob和事务提交时做冲突检测。 你可通过如下etcdctl txn命令，获取Alice和Bob账号的资产和最后一次修改时的版本号(mod_revision):</p><pre><code>$ etcdctl txn -i -w=json
compares:


success requests (get, put, del):
get Alice
get Bob


failure requests (get, put, del):


{
 &quot;kvs&quot;:[
      {
          &quot;key&quot;:&quot;QWxpY2U=&quot;,
          &quot;create_revision&quot;:2,
          &quot;mod_revision&quot;:2,
          &quot;version&quot;:1,
          &quot;value&quot;:&quot;MjAw&quot;
      }
  ],
    ......
  &quot;kvs&quot;:[
      {
          &quot;key&quot;:&quot;Qm9i&quot;,
          &quot;create_revision&quot;:3,
          &quot;mod_revision&quot;:3,
          &quot;version&quot;:1,
          &quot;value&quot;:&quot;MzAw&quot;
      }
  ],
}
</code></pre><p>其次发起资金转账操作，Alice账号减去100，Bob账号增加100。为了保证转账事务的准确性、一致性，提交事务的时候需检查Alice和Bob账号最新修改版本号与读取资金时的一致(compares操作中增加版本号检测)，以保证其他事务未修改两个账号的资金。</p><p>若compares操作通过检查，则执行转账操作，否则执行查询Alice和Bob账号资金操作，命令如下:</p><pre><code>$ etcdctl txn -i
compares:
mod(&quot;Alice&quot;) = &quot;2&quot;
mod(&quot;Bob&quot;) = &quot;3&quot;


success requests (get, put, del):
put Alice 100
put Bob 300


failure requests (get, put, del):
get Alice
get Bob


SUCCESS


OK

OK
</code></pre><p>到这里我们就完成了一个安全的转账事务操作，从以上流程中你可以发现，自己从0到1实现一个完整的事务还是比较繁琐的，幸运的是，etcd社区基于以上介绍的事务特性，提供了一个简单的事务框架<a href="https://github.com/etcd-io/etcd/blob/v3.4.9/clientv3/concurrency/stm.go">STM</a>，构建了各个事务隔离级别类，帮助你进一步简化应用编程复杂度。</p><h2>小结</h2><p>最后我们来小结下今天的内容。首先我给你介绍了事务API的基本结构，它由If、Then、Else语句组成。</p><p>其中If支持多个比较规则，它是用于事务提交时的冲突检测，比较的对象支持key的<strong>mod_revision</strong>、<strong>create_revision、version、value值</strong>。随后我给你介绍了整个事务执行的基本流程，Apply模块首先执行If的比较规则，为真则执行Then语句，否则执行Else语句。</p><p>接着通过转账案例，四幅转账事务时间序列图，我为你分析了事务的ACID特性，剖析了在etcd中事务的ACID特性的实现。</p><ul>
<li>
<p>原子性是指一个事务要么全部成功要么全部失败，etcd基于WAL日志、consistent index、boltdb的事务能力提供支持。</p>
</li>
<li>
<p>一致性是指事务转账前后的，数据库和应用程序期望的恒等状态应该保持不变，这通过数据库和业务应用程序相互协作完成。</p>
</li>
<li>
<p>持久性是指事务提交后，数据不丢失，</p>
</li>
<li>
<p>隔离性是指事务提交过程中的可见性，etcd不存在脏读，基于MVCC机制、boltdb事务你可以实现可重复读、串行化快照隔离级别的事务，保障并发事务场景中你的数据安全性。</p>
</li>
</ul><h2>思考题</h2><p>在数据库事务中，有各种各样的概念，比如脏读、脏写、不可重复读与读倾斜、幻读与写倾斜、更新丢失、快照隔离、可串行化快照隔离? 你知道它们的含义吗？</p><p>感谢你的阅读，如果你认为这节课的内容有收获，也欢迎把它分享给你的朋友，谢谢。</p>
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
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/ib3Rzem884S5MOS96THy0gQXcF26PNsnRBpyr3pM5rVibZdYvAibpVvAGfibF1ddpgrteg9fQUsq4vce9EM95Jj97Q/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_604077</span>
  </div>
  <div class="_2_QraFYR_0">在数据库事务中，有各种各样的概念，比如脏读、脏写、不可重复读与读倾斜、幻读与写倾斜、更新丢失、快照隔离、可串行化快照隔离? 你知道它们的含义吗？<br>脏读、脏写：<br><br>在读未提交的隔离级别的情况下，事务A执行过程中，事务A对数据资源进行了修改，事务B读取了事务A修改后且未提交的数据。A因为某些原因回滚了操作，B却使用了A对资源修改后的数据，进行了读写等操作。<br><br>不可重复读：<br><br>在读未提交、读提交的隔离级别情况下，事务B读取了两次数据资源，在这两次读取的过程中事务A修改了数据，导致事务B在这两次读取出来的数据不一致。这种在同一个事务中，前后两次读取的数据不一致的现象就是不可重复读。<br><br>幻读：<br><br>在可重复读得隔离级别情况下，事务B前后两次读取同一个范围的数据，在事务B两次读取的过程中事务A新增了数据，导致事务B后一次读取到前一次查询没有看到的行。<br><br>读倾斜、写倾斜：<br><br>读写倾斜是在数据分表不合理的情况下，对某个表的数据存在大量的读取写入的需求，分表不均衡不合理导致的。<br><br>更新丢失：<br><br>第一类丢失，在读未提交的隔离级别情况下，事务A和事务B都对数据进行更新，但是事务A由于某种原因事务回滚了，把已经提交的事务B的更新数据给覆盖了。这种现象就是第一类更新丢失。<br><br>第二类丢失，在可重复读的隔离级别情况下，跟第一类更新丢失有点类似，也是两个事务同时对数据进行更新，但是事务A的更新把已提交的事务B的更新数据给覆盖了。这种现象就是第二类更新丢失。<br><br>快照隔离：<br><br>每个事务都从一个数据库的快照中读数据，如果有些数据在当前事务开始之后，被其他事务改变了值，快照隔离能够保证当前事务无法看到这个新值。<br><br>可串行化快照隔离：<br><br>可串行化快照隔离是在快照隔离级别之上，支持串行化。<br>（讲得特别好，肯定花了特别多的心思跟时间，肯定掉了好多头发🐶，给老师点赞）</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-15 14:42:56</div>
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
  <div class="_2_QraFYR_0">老师分析得透彻，学习了，原来自己平时使用的事务过程中，无意间已经使用了串行化快照隔离级别，还请问一下事务性能相比put接口差异大吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，肯定有不少差异的，事务执行逻辑更复杂，影响性能的因素比较多，实践篇我有介绍，建议根据自己实际业务场景，部署环境，使用etcd自带的benchmark工具压测对比下差异.</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-10 07:34:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLDUJyeq54fiaXAgF62tNeocO3lHsKT4mygEcNoZLnibg6ONKicMgCgUHSfgW8hrMUXlwpNSzR8MHZwg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>types</span>
  </div>
  <div class="_2_QraFYR_0">关于事务txn，请问事务都是在leader执行的嘛？还是根据request中的动作决定？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-25 22:39:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/22/e1/6b/74a8b7d8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Hugh</span>
  </div>
  <div class="_2_QraFYR_0">在快照串行化隔离的案例里面，有个问题想请教一下，首先冲突检测机制是在提交前进行检测的，那么意味着无论提交是否成功，大部分的节点都已经写入了raft log或者WAL，假设这次检测的结果是失败的（比如已经没有足够的钱进行转账），那么就不会提交了（给客户端的反馈也是此次转账失败）。<br>这个时候leader节点突然宕机，其他节点重新开始了选举，那些已经写入此次转账log的节点是有可能成为新的leader，在新的任期中，仍然会尝试提交这次事务，假设这次提交时已经有了足够的金额，则会导致提交成功。但是这次的提交成功对于客户端来说完全不感知，这样子就会导致用户发现自己的金额突然变少了，这个肯定是不符合预期的，老师可以解释一下这中间是不是有哪部分我的理解有问题呢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-21 11:46:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/96/a2/c1596dd8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>🤔</span>
  </div>
  <div class="_2_QraFYR_0">etcd事务可用在分布式事务吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-15 17:32:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLDRPejHodutia9Ud8UZLY8g5lTkKXgf3J104c0jM9aFfAGNoUdxkRLnnWRc5Kd3jIeN3EqXxKFT0g/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>蓝莓侠</span>
  </div>
  <div class="_2_QraFYR_0">串行化快照隔离中的冲突检测，感觉和CAS一样啊，有什么区别吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-14 19:24:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/jA97yib7VetXc4iclOg2gGfZu1fO7efyib2mKeqvIxDdmgLqukusyFzPrbIQeZYR0WDJUicRakgVGroaYC7aWGFrEw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Turing</span>
  </div>
  <div class="_2_QraFYR_0">这篇文章含金量很高, 分布式中的acid和数据库中的acid是不一样的概念. 其实事务提交的时候还是走了读写锁, 读写锁的目的应该是为了回写buffer吧. etcd事务提交, 并不一定立马持久化到磁盘中. </div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-22 16:51:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/11/3e/925aa996.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>HelloBug</span>
  </div>
  <div class="_2_QraFYR_0">&gt;Apply 模块首先执行 If 的比较规则，为真则执行 Then 语句，否则执行 Else 语句<br>这里执行Then里的命令的时候，为什么不可能发生状态的变更呢？例如向Bob转账100，put bob 300时，bob的账户已经是300了，就是说判断的时候还没问题，执行Then的时候就有问题了。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-05 17:53:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/11/3e/925aa996.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>HelloBug</span>
  </div>
  <div class="_2_QraFYR_0">极端一点考虑，冲突检测的时候可能会发生很多次的冲突，这样应用程序如果只进行了一次的冲突检测，是可能出现事务提交失败的情况的，对吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-05 17:47:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/11/3e/925aa996.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>HelloBug</span>
  </div>
  <div class="_2_QraFYR_0">&gt;&gt;为了解决并发事务冲突问题，事务 A 中增加了冲突检测，期望的 Alice 版本号应为 2，Bob 为 3。结果事务 B 的修改导致 Bob 版本号变成了 5<br>这里Bob的版本号应该是变成了4吧？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-05 17:44:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/11/3e/925aa996.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>HelloBug</span>
  </div>
  <div class="_2_QraFYR_0">已提交读和可重复读本身就是互相冲突的，如果每次都能读取已经提交的数据，那重复读的时候，前后两次的数据就可能是不一样的，可重复读无法保证。如果利用文中说的第一次读的时候，将读到的数据放在一个缓存里，然后下一次读的时候，读取到的数据就和上一次一样，这样读取到的数据就不是已提交的。这两个特性不可能同时满足吧？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-05 17:43:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTIAfpw5Mm3UNgMdju8V3WSbQiaHp4RxaG3JTz9Zx3dtL32Rib7zTVn6v7a1OF6KQcmQYnnILrSmee8g/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>降措</span>
  </div>
  <div class="_2_QraFYR_0">多次执行stm事务时，偶尔会提示etcdserver: mvcc: required revision is a future revision，一直没有发现是什么原因</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-16 18:06:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLDUJyeq54fiaXAgF62tNeocO3lHsKT4mygEcNoZLnibg6ONKicMgCgUHSfgW8hrMUXlwpNSzR8MHZwg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>types</span>
  </div>
  <div class="_2_QraFYR_0">etcd事务默认的隔离级别是什么？ 可重复读？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-03 11:04:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/30/85/14c2f16c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>石小</span>
  </div>
  <div class="_2_QraFYR_0">“由于 etcd 是批量提交写事务的，而读事务又是快照读，因此当 MVCC 写事务完成时，它需要更新 buffer，这样下一个读请求到达时，才能从 buffer 中获取到最新数据。”  会不会出站事务完成后更新buffer失败的情况？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-01 08:54:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/57/4f/6fb51ff1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>一步</span>
  </div>
  <div class="_2_QraFYR_0">可串行化的快照隔离: 是每个事物都会生成一个新的快照吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-24 21:06:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/68/83/ecd4e4d6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>WGJ</span>
  </div>
  <div class="_2_QraFYR_0">老师，有个问题就是 在原子性和持久性介绍当中，假如在WAF日志已经提交成功了，执行事务的时候发生了crash，这时候我理解的返回给client是失败的，那么当执行WAF重放日志的时候，该事务又成功了；换句话说，事务操作 返回给client 结果 的时机是什么时候呢，是写 WAF日志成功就返回呢，还是等待事务提交之后才返回呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 返回给client的结果时机可参考03节写流程架构图中第10步，apply成功的时候，会通过管道通知给相应的goroutine,  告知写入结果，然后返回给client。如果事务执行过程中crash了，这时client看到的应该是超时错误，超时意味着，可能写入成功，可能写入失败，业务自己重试时需要进行相应的检查、确保幂等性。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-19 10:40:52</div>
  </div>
</div>
</div>
</li>
</ul>