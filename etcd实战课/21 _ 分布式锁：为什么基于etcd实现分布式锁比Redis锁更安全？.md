<audio title="21 _ 分布式锁：为什么基于etcd实现分布式锁比Redis锁更安全？" src="https://static001.geekbang.org/resource/audio/09/b4/09366811dbe46c29964612d885f6d6b4.mp3" controls="controls"></audio> 
<p>你好，我是唐聪。</p><p>在软件开发过程中，我们经常会遇到各种场景要求对共享资源进行互斥操作，否则整个系统的数据一致性就会出现问题。典型场景如商品库存操作、Kubernertes调度器为Pod分配运行的Node。</p><p>那要如何实现对共享资源进行互斥操作呢？</p><p>锁就是其中一个非常通用的解决方案。在单节点多线程环境，你使用本地的互斥锁就可以完成资源的互斥操作。然而单节点存在单点故障，为了保证服务高可用，你需要多节点部署。在多节点部署的分布式架构中，你就需要使用分布式锁来解决资源互斥操作了。</p><p>但是为什么有的业务使用了分布式锁还会出现各种严重超卖事故呢？分布式锁的实现和使用过程需要注意什么？</p><p>今天，我就和你聊聊分布式锁背后的故事，我将通过一个茅台超卖的案例，为你介绍基于Redis实现的分布锁优缺点，引出分布式锁的核心要素，对比分布式锁的几种业界典型实现方案，深入剖析etcd分布式锁的实现。</p><p>希望通过这节课，让你了解etcd分布式锁的应用场景、核心原理，在业务开发过程中，优雅、合理的使用分布式锁去解决各类资源互斥、并发操作问题。</p><h2>从茅台超卖案例看分布式锁要素</h2><p>首先我们从去年一个因Redis分布式锁实现问题导致<a href="https://juejin.cn/post/6854573212831842311">茅台超卖案例</a>说起，在这个网友分享的真实案例中，因茅台的稀缺性，事件最终定级为P0级生产事故，后果影响严重。</p><!-- [[[read_end]]] --><p>那么它是如何导致超卖的呢？</p><p>首先和你简单介绍下此案例中的Redis简易分布式锁实现方案，它使用了Redis SET命令来实现。</p><pre><code>SET key value [EX seconds|PX milliseconds|EXAT timestamp|PXAT milliseconds-timestamp|KEEPTTL] [NX|XX] 
[GET]
</code></pre><p>简单给你介绍下SET命令重点参数含义：</p><ul>
<li><code>EX</code>  设置过期时间，单位秒；</li>
<li><code>NX</code> 当key不存在的时候，才设置key；</li>
<li><code>XX</code> 当key存在的时候，才设置key。</li>
</ul><p>此业务就是基于Set key value EX 10 NX命令来实现的分布式锁，并通过JAVA的try-finally语句，执行Del key语句来释放锁，简易流程如下：</p><pre><code># 对资源key加锁，key不存在时创建，并且设置，10秒自动过期
SET key value EX 10 NX
业务逻辑流程1，校验用户身份
业务逻辑流程2，查询并校验库存(get and compare)
业务逻辑流程3，库存&gt;0，扣减库存(Decr stock)，生成秒杀茅台订单

# 释放锁
Del key
</code></pre><p>以上流程中其实存在以下思考点:</p><ul>
<li>NX参数有什么作用?</li>
<li>为什么需要原子的设置key及过期时间？</li>
<li>为什么基于Set key value EX 10 NX命令还出现了超卖呢?</li>
<li>为什么大家都比较喜欢使用Redis作为分布式锁实现？</li>
</ul><p>首先来看第一个问题，NX参数的作用。NX参数是为了保证当分布式锁不存在时，只有一个client能写入此key成功，获取到此锁。我们使用分布式锁的目的就是希望在高并发系统中，有一种互斥机制来防止彼此相互干扰，保证数据的一致性。</p><p><strong>因此分布式锁的第一核心要素就是互斥性、安全性。在同一时间内，不允许多个client同时获得锁。</strong></p><p>再看第二个问题，假设我们未设置key自动过期时间，在Set key value NX后，如果程序crash或者发生网络分区后无法与Redis节点通信，毫无疑问其他client将永远无法获得锁。这将导致死锁，服务出现中断。</p><p>有的同学意识到这个问题后，使用如下SETNX和EXPIRE命令去设置key和过期时间，这也是不正确的，因为你无法保证SETNX和EXPIRE命令的原子性。</p><pre><code># 对资源key加锁，key不存在时创建
SETNX key value
# 设置KEY过期时间
EXPIRE key 10
业务逻辑流程

# 释放锁
Del key
</code></pre><p><strong>这就是分布式锁第二个核心要素，活性。在实现分布式锁的过程中要考虑到client可能会出现crash或者网络分区，你需要原子申请分布式锁及设置锁的自动过期时间，通过过期、超时等机制自动释放锁，避免出现死锁，导致业务中断。</strong></p><p>再看第三个问题，为什么使用了Set key value EX 10 NX命令，还出现了超卖呢？</p><p>原来是抢购活动开始后，加锁逻辑中的业务流程1访问的用户身份服务出现了高负载，导致阻塞在校验用户身份流程中(超时30秒)，然而锁10秒后就自动过期了，因此其他client能获取到锁。关键是阻塞的请求执行完后，它又把其他client的锁释放掉了，导致进入一个恶性循环。</p><p>因此申请锁时，写入的value应确保唯一性（随机值等）。client在释放锁时，应通过Lua脚本原子校验此锁的value与自己写入的value一致，若一致才能执行释放工作。</p><p>更关键的是库存校验是通过get and compare方式，它压根就无法防止超卖。正确的解决方案应该是通过LUA脚本实现Redis比较库存、扣减库存操作的原子性（或者在每次只能抢购一个的情况下，通过判断<a href="https://redis.io/commands/DECR">Redis Decr命令</a>的返回值即可。此命令会返回扣减后的最新库存，若小于0则表示超卖）。</p><p><strong>从这个问题中我们可以看到，分布式锁实现具备一定的复杂度，它不仅依赖存储服务提供的核心机制，同时依赖业务领域的实现。无论是遭遇高负载、还是宕机、网络分区等故障，都需确保锁的互斥性、安全性，否则就会出现严重的超卖生产事故。</strong></p><p>再看最后一个问题，为什么大家都比较喜欢使用Redis做分布式锁的实现呢?</p><p>考虑到在秒杀等业务场景上存在大量的瞬间、高并发请求，加锁与释放锁的过程应是高性能、高可用的。而Redis核心优点就是快、简单，是随处可见的基础设施，部署、使用也及其方便，因此广受开发者欢迎。</p><p><strong>这就是分布式锁第三个核心要素，高性能、高可用。加锁、释放锁的过程性能开销要尽量低，同时要保证高可用，确保业务不会出现中断。</strong></p><p>那么除了以上案例中人为实现问题导致的锁不安全因素外，基于Redis实现的以上分布式锁还有哪些安全性问题呢？</p><h2>Redis分布式锁问题</h2><p>我们从茅台超卖案例中为你总结出的分布式核心要素（互斥性、安全性、活性、高可用、高性能）说起。</p><p>首先，如果我们的分布式锁跑在单节点的Redis Master节点上，那么它就存在单点故障，无法保证分布式锁的高可用。</p><p>于是我们需要一个主备版的Redis服务，至少具备一个Slave节点。</p><p>我们又知道Redis是基于主备异步复制协议实现的Master-Slave数据同步，如下图所示，若client A执行SET key value EX 10 NX命令，redis-server返回给client A成功后，Redis Master节点突然出现crash等异常，这时候Redis Slave节点还未收到此命令的同步。</p><p><img src="https://static001.geekbang.org/resource/image/cd/45/cd3d4ab1af45c6eb76e7dccd9c666245.png?wh=1920*994" alt=""></p><p>若你部署了Redis Sentinel等主备切换服务，那么它就会以Slave节点提升为主，此时Slave节点因并未执行SET key value EX 10 NX命令，因此它收到client B发起的加锁的此命令后，它也会返回成功给client。</p><p>那么在同一时刻，集群就出现了两个client同时获得锁，分布式锁的互斥性、安全性就被破坏了。</p><p>除了主备切换可能会导致基于Redis实现的分布式锁出现安全性问题，在发生网络分区等场景下也可能会导致出现脑裂，Redis集群出现多个Master，进而也会导致多个client同时获得锁。</p><p>如下图所示，Master节点在可用区1，Slave节点在可用区2，当可用区1和可用区2发生网络分区后，部署在可用区2的Redis Sentinel服务就会将可用区2的Slave提升为Master，而此时可用区1的Master也在对外提供服务。因此集群就出现了脑裂，出现了两个Master，都可对外提供分布式锁申请与释放服务，分布式锁的互斥性被严重破坏。</p><p><img src="https://static001.geekbang.org/resource/image/cb/b1/cb4cb52cf2244d2000884ef5f5ff3db1.png?wh=1920*1026" alt=""></p><p><strong>主备切换、脑裂是Redis分布式锁的两个典型不安全的因素，本质原因是Redis为了满足高性能，采用了主备异步复制协议，同时也与负责主备切换的Redis Sentinel服务是否合理部署有关。</strong></p><p>有没有其他方案解决呢？</p><p>当然有，Redis作者为了解决SET key value [EX] 10 [NX]命令实现分布式锁不安全的问题，提出了<a href="https://redis.io/topics/distlock">RedLock算法</a>。它是基于多个独立的Redis Master节点的一种实现（一般为5）。client依次向各个节点申请锁，若能从多数个节点中申请锁成功并满足一些条件限制，那么client就能获取锁成功。</p><p>它通过独立的N个Master节点，避免了使用主备异步复制协议的缺陷，只要多数Redis节点正常就能正常工作，显著提升了分布式锁的安全性、可用性。</p><p>但是，它的实现建立在一个不安全的系统模型上的，它依赖系统时间，当时钟发生跳跃时，也可能会出现安全性问题。你要有兴趣的话，可以详细阅读下分布式存储专家Martin对<a href="https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html">RedLock的分析文章</a>，Redis作者的也专门写了<a href="http://antirez.com/news/101">一篇文章进行了反驳</a>。</p><h2>分布式锁常见实现方案</h2><p>了解完Redis分布式锁的一系列问题和实现方案后，我们再看看还有哪些典型的分布式锁实现。</p><p>除了Redis分布式锁，其他使用最广的应该是ZooKeeper分布式锁和etcd分布式锁。</p><p>ZooKeeper也是一个典型的分布式元数据存储服务，它的分布式锁实现基于ZooKeeper的临时节点和顺序特性。</p><p>首先什么是临时节点呢？</p><p>临时节点具备数据自动删除的功能。当client与ZooKeeper连接和session断掉时，相应的临时节点就会被删除。</p><p>其次ZooKeeper也提供了Watch特性可监听key的数据变化。</p><p><a href="https://www.usenix.org/legacy/event/atc10/tech/full_papers/Hunt.pdf">使用Zookeeper加锁的伪代码如下</a>：</p><pre><code>Lock
1 n = create(l + “/lock-”, EPHEMERAL|SEQUENTIAL)
2 C = getChildren(l, false)
3 if n is lowest znode in C, exit
4 p = znode in C ordered just before n
5 if exists(p, true) wait for watch event
6 goto 2
Unlock
1 delete(n)
</code></pre><p>接下来我重点给你介绍一下基于etcd的分布式锁实现。</p><h2>etcd分布式锁实现</h2><p>那么基于etcd实现的分布式锁是如何确保安全性、互斥性、活性的呢？</p><h3>事务与锁的安全性</h3><p>从Redis案例中我们可以看到，加锁的过程需要确保安全性、互斥性。比如，当key不存在时才能创建，否则查询相关key信息，而etcd提供的事务能力正好可以满足我们的诉求。</p><p>正如我在<a href="https://time.geekbang.org/column/article/341935">09</a>中给你介绍的事务特性，它由IF语句、Then语句、Else语句组成。其中在IF语句中，支持比较key的是修改版本号mod_revision和创建版本号create_revision。</p><p>在分布式锁场景，你就可以通过key的创建版本号create_revision来检查key是否已存在，因为一个key不存在的话，它的create_revision版本号就是0。</p><p>若create_revision是0，你就可发起put操作创建相关key，具体代码如下:</p><pre><code>txn := client.Txn(ctx).If(v3.Compare(v3.CreateRevision(k), 
&quot;=&quot;, 0))
</code></pre><p>你要注意的是，实现分布式锁的方案有多种，比如你可以通过client是否成功创建一个固定的key，来判断此client是否获得锁，你也可以通过多个client创建prefix相同，名称不一样的key，哪个key的revision最小，最终就是它获得锁。至于谁优谁劣，我作为思考题的一部分，留给大家一起讨论。</p><p>相比Redis基于主备异步复制导致锁的安全性问题，etcd是基于Raft共识算法实现的，一个写请求需要经过集群多数节点确认。因此一旦分布式锁申请返回给client成功后，它一定是持久化到了集群多数节点上，不会出现Redis主备异步复制可能导致丢数据的问题，具备更高的安全性。</p><h3>Lease与锁的活性</h3><p>通过事务实现原子的检查key是否存在、创建key后，我们确保了分布式锁的安全性、互斥性。那么etcd是如何确保锁的活性呢? 也就是发生任何故障，都可避免出现死锁呢？</p><p>正如在<a href="https://time.geekbang.org/column/article/339337">06</a>租约特性中和你介绍的，Lease就是一种活性检测机制，它提供了检测各个客户端存活的能力。你的业务client需定期向etcd服务发送"特殊心跳"汇报健康状态，若你未正常发送心跳，并超过和etcd服务约定的最大存活时间后，就会被etcd服务移除此Lease和其关联的数据。</p><p>通过Lease机制就优雅地解决了client出现crash故障、client与etcd集群网络出现隔离等各类故障场景下的死锁问题。一旦超过Lease TTL，它就能自动被释放，确保了其他client在TTL过期后能正常申请锁，保障了业务的可用性。</p><p>具体代码如下:</p><pre><code>txn := client.Txn(ctx).If(v3.Compare(v3.CreateRevision(k), &quot;=&quot;, 0))
txn = txn.Then(v3.OpPut(k, val, v3.WithLease(s.Lease())))
txn = txn.Else(v3.OpGet(k))
resp, err := txn.Commit()
if err != nil {
    return err
}
</code></pre><h3>Watch与锁的可用性</h3><p>当一个持有锁的client crash故障后，其他client如何快速感知到此锁失效了，快速获得锁呢，最大程度降低锁的不可用时间呢？</p><p>答案是Watch特性。正如在08 Watch特性中和你介绍的，Watch提供了高效的数据监听能力。当其他client收到Watch Delete事件后，就可快速判断自己是否有资格获得锁，极大减少了锁的不可用时间。</p><p>具体代码如下所示：</p><pre><code>var wr v3.WatchResponse
wch := client.Watch(cctx, key, v3.WithRev(rev))
for wr = range wch {
   for _, ev := range wr.Events {
      if ev.Type == mvccpb.DELETE {
         return nil
      }
   }
}
</code></pre><h3>etcd自带的concurrency包</h3><p>为了帮助你简化分布式锁、分布式选举、分布式事务的实现，etcd社区提供了一个名为concurrency包帮助你更简单、正确地使用分布式锁、分布式选举。</p><p>下面我简单为你介绍下分布式锁<a href="https://github.com/etcd-io/etcd/tree/v3.4.9/clientv3/concurrency">concurrency</a>包的使用和实现，它的使用非常简单，如下代码所示，核心流程如下：</p><ul>
<li>首先通过concurrency.NewSession方法创建Session，本质是创建了一个TTL为10的Lease。</li>
<li>其次得到session对象后，通过concurrency.NewMutex创建了一个mutex对象，包含Lease、key prefix等信息。</li>
<li>然后通过mutex对象的Lock方法尝试获取锁。</li>
<li>最后使用结束，可通过mutex对象的Unlock方法释放锁。</li>
</ul><pre><code>cli, err := clientv3.New(clientv3.Config{Endpoints: endpoints})
if err != nil {
   log.Fatal(err)
}
defer cli.Close()
// create two separate sessions for lock competition
s1, err := concurrency.NewSession(cli, concurrency.WithTTL(10))
if err != nil {
   log.Fatal(err)
}
defer s1.Close()
m1 := concurrency.NewMutex(s1, &quot;/my-lock/&quot;)
// acquire lock for s1
if err := m1.Lock(context.TODO()); err != nil {
   log.Fatal(err)
}
fmt.Println(&quot;acquired lock for s1&quot;)
if err := m1.Unlock(context.TODO()); err != nil {
   log.Fatal(err)
}
fmt.Println(&quot;released lock for s1&quot;)
</code></pre><p>那么mutex对象的Lock方法是如何加锁的呢？</p><p>核心还是使用了我们上面介绍的事务和Lease特性，当CreateRevision为0时，它会创建一个prefix为/my-lock的key（ /my-lock + LeaseID)，并获取到/my-lock prefix下面最早创建的一个key（revision最小），分布式锁最终是由写入此key的client获得，其他client则进入等待模式。</p><p>详细代码如下：</p><pre><code>m.myKey = fmt.Sprintf(&quot;%s%x&quot;, m.pfx, s.Lease())
cmp := v3.Compare(v3.CreateRevision(m.myKey), &quot;=&quot;, 0)
// put self in lock waiters via myKey; oldest waiter holds lock
put := v3.OpPut(m.myKey, &quot;&quot;, v3.WithLease(s.Lease()))
// reuse key in case this session already holds the lock
get := v3.OpGet(m.myKey)
// fetch current holder to complete uncontended path with only one RPC
getOwner := v3.OpGet(m.pfx, v3.WithFirstCreate()...)
resp, err := client.Txn(ctx).If(cmp).Then(put, getOwner).Else(get, getOwner).Commit()
if err != nil {
   return err
}
</code></pre><p>那未获得锁的client是如何等待的呢?</p><p>答案是通过Watch机制各自监听prefix相同，revision比自己小的key，因为只有revision比自己小的key释放锁，我才能有机会，获得锁，如下代码所示，其中waitDelete会使用我们上面的介绍的Watch去监听比自己小的key，详细代码可参考<a href="https://github.com/etcd-io/etcd/blob/v3.4.9/clientv3/concurrency/mutex.go">concurrency mutex</a>的实现。</p><pre><code>// wait for deletion revisions prior to myKey
hdr, werr := waitDeletes(ctx, client, m.pfx, m.myRev-1)
// release lock key if wait failed
if werr != nil {
   m.Unlock(client.Ctx())
} else {
   m.hdr = hdr
}
</code></pre><h2>小结</h2><p>最后我们来小结下今天的内容。</p><p>今天我通过一个Redis分布式锁实现问题——茅台超卖案例，给你介绍了分布式锁的三个主要核心要素，它们分别如下：</p><ul>
<li>安全性、互斥性。在同一时间内，不允许多个client同时获得锁。</li>
<li>活性。无论client出现crash还是遭遇网络分区，你都需要确保任意故障场景下，都不会出现死锁，常用的解决方案是超时和自动过期机制。</li>
<li>高可用、高性能。加锁、释放锁的过程性能开销要尽量低，同时要保证高可用，避免单点故障。</li>
</ul><p>随后我通过这个案例，继续和你分析了Redis SET命令创建分布式锁的安全性问题。单Redis Master节点存在单点故障，一主多备Redis实例又因为Redis主备异步复制，当Master节点发生crash时，可能会导致同时多个client持有分布式锁，违反了锁的安全性问题。</p><p>为了优化以上问题，Redis作者提出了RedLock分布式锁，它基于多个独立的Redis Master节点工作，只要一半以上节点存活就能正常工作，同时不依赖Redis主备异步复制，具有良好的安全性、高可用性。然而它的实现依赖于系统时间，当发生时钟跳变的时候，也会出现安全性问题。</p><p>最后我和你重点介绍了etcd的分布式锁实现过程中的一些技术点。它通过etcd事务机制，校验CreateRevision为0才能写入相关key。若多个client同时申请锁，则client通过比较各个key的revision大小，判断是否获得锁，确保了锁的安全性、互斥性。通过Lease机制确保了锁的活性，无论client发生crash还是网络分区，都能保证不会出现死锁。通过Watch机制使其他client能快速感知到原client持有的锁已释放，提升了锁的可用性。最重要的是etcd是基于Raft协议实现的高可靠、强一致存储，正常情况下，不存在Redis主备异步复制协议导致的数据丢失问题。</p><h2>思考题</h2><p>这节课到这里也就结束了，最后我给你留了两个思考题。</p><p>第一，死锁、脑裂、惊群效应是分布式锁的核心问题，你知道它们各自是怎么一回事吗？ZooKeeper和etcd是如何应对这些问题的呢？</p><p>第二，若你锁设置的10秒，如果你的某业务进程抢锁成功后，执行可能会超过10秒才成功，在这过程中如何避免锁被自动释放而出现的安全性问题呢?</p><p>感谢你的阅读，也欢迎你把这篇文章分享给更多的朋友一起阅读。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/9d/84/171b2221.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jeffery</span>
  </div>
  <div class="_2_QraFYR_0">老师太厉害了👍etcd和redis 分析的太太透彻了！终于明白了etcd和redis 锁的区别了…….专栏快接近尾声了……真有点不舍！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞jeffery一直坚持不懈的学习，感谢认可，专栏虽即将结束，但学习与探索未知领域从未有终点，后面有什么有趣的案例、新的体会，我也会通过专栏、微信公众号、博客等各种渠道与大家一起分享交流！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-10 13:27:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/ib3Rzem884S5MOS96THy0gQXcF26PNsnRBpyr3pM5rVibZdYvAibpVvAGfibF1ddpgrteg9fQUsq4vce9EM95Jj97Q/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_604077</span>
  </div>
  <div class="_2_QraFYR_0">死锁、脑裂、惊群效应是分布式锁的核心问题，你知道它们各自是怎么一回事吗？ZooKeeper 和 etcd 是如何应对这些问题的呢？<br>死锁：加锁后由于没有添加锁的过期时间或者锁的时间设置过长，服务异常crash或者服务执行后漏执行释放锁操作，导致锁长时间没有释放<br><br>脑裂：分布式体系结构中的集中式结构（也称为 Master&#47;Slave 架构），一个Master节点多个Slave节点，所有的请求数据处理必须先经过Master中央服务器，由Master统一进行资源和任务调度，中央服务器根据这些信息，将任务下达给节点服务器，节点服务器执行任务，并将结果反馈给中央服务器。脑裂是在某些特殊条件下，如主备切换，Slave在与Master的网络出现故障的时候，Slave会认为Master已经故障，从而成为新的master，而原来的master也没有卸任，从而导致存在两个Master在对外服务，存在多master会导致共享资源的互斥性遭到破坏，出现资源争抢，数据不一致等问题<br><br>惊群效应：指多进程（多线程）在同时阻塞等待同一个事件的时候（休眠状态），如果等待的这个事件发生，那么他就会唤醒等待的所有进程（或者线程），但是最终却只能有一个进程（线程）获得这个时间的“控制权”，对该事件进行处理，而其他进程（线程）获取“控制权”失败，只能重新进入休眠状态，这种现象和性能浪费就叫做惊群效应。（当你往一群鸽子中间扔一块食物，虽然最终只有一个鸽子抢到食物，但所有鸽子都会被惊动来争夺，没有抢到食物的鸽子只好回去继续睡觉， 等待下一块食物到来。这样，每扔一块食物，都会惊动所有的鸽子，即为惊群。）<br><br>etcd如何避免死锁：利用Lease的活性检测机制，它提供了检测各个客户端存活的能力。你的业务 client 需定期向 etcd 服务发送&quot;特殊心跳&quot;汇报健康状态，若你未正常发送心跳，并超过和 etcd 服务约定的最大存活时间后，就会被 etcd 服务移除此 Lease 和其关联的数据。通过 Lease 机制就优雅地解决了 client 出现 crash 故障、client 与 etcd 集群网络出现隔离等各类故障场景下的死锁问题。一旦超过 Lease TTL，它就能自动被释放，确保了其他 client 在 TTL 过期后能正常申请锁，保障了业务的可用性。<br><br>etcd如何避免集群脑裂：在leader选举上采用了过半机制，即得到一半以上的follow才能成为leader，且新leader就任后旧leader必须卸任，所以etcd不存在脑裂问题<br><br>etcd如何避免惊群效应：mutex，通过 Watch 机制各自监听 prefix 相同，revision 比自己小的 key，因为只有 revision 比自己小的 key 释放锁，才能有机会，获得锁</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-16 13:59:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/8f/cf/890f82d6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>那时刻</span>
  </div>
  <div class="_2_QraFYR_0">老师文中提到的思考题，多个client抢同一把锁与多个client各自抢自己的锁。我的想法是多个client抢同一把锁，在client数目多的时候，同一把锁的竞争比较激烈。而多个client各自抢各自的锁，会有锁饥饿问题，比如新的client因其version比较小，更容易获得锁。<br><br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞，我补充一点，多个client通过写同key抢同把锁，主要有惊群效应，同时获取锁性能也低点，毕竟是需要实时写入相关key的，而后者，revision是按时间全局递增的，因此新的client 写入的key revision会比较大，拿锁的顺序可以理解为按时间顺序排队。 </p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-10 10:39:10</div>
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
  <div class="_2_QraFYR_0">唐老师好，etcd有像innodb那样能能控制持久化程度的配置(主要指多久fsync一次磁盘)吗？或者说，etcd可能出现持久化失败(写入磁盘缓存，没fsync)吗？如果持久化失败会有哪些影响？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 有的,etcd提供了如下两个参数可以控制事务提交的行为。<br>--backend-batch-interval &#39;&#39;<br>    BackendBatchInterval is the maximum time before commit the backend transaction.<br>  --backend-batch-limit &#39;0&#39;<br>    BackendBatchLimit is the maximum operations before commit the backend transaction.<br>在etcd v3.4.9中，backend-batch-interval如果你没指定，默认是100ms，对应的异步goroutine将批量、每隔100ms，将boltdb事务进行提交。<br>backend-batch-limit默认是10000，当堆积的put&#47;del等操作若超过10000个，则会同步触发boltdb事务提交<br>，若boltdb事务持久化失败，则会异常退出，重启时会重放最新WAL日志中已提交的日志条目，再次执行。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-17 21:24:45</div>
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
  <div class="_2_QraFYR_0">你要注意的是，实现分布式锁的方案有多种，比如你可以通过 client 是否成功创建一个固定的 key，来判断此 client 是否获得锁，你也可以通过多个 client 创建 prefix 相同，名称不一样的 key，哪个 key 的 revision 最小，最终就是它获得锁。至于谁优谁劣，我作为思考题的一部分，留给大家一起讨论。<br>1. 按照文中介绍concurrency包中用的是prefix<br>2. 如果使用相同的key，我能够想到存在的问题,在释放锁后，会导致获取锁的事务同时发生，事务数量变得很大<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，最主要是惊群效应，所有client都会收到相应key被删除消息，都会尝试发起事务，写入key，性能会比较差</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-11 15:12:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/ib3Rzem884S5MOS96THy0gQXcF26PNsnRBpyr3pM5rVibZdYvAibpVvAGfibF1ddpgrteg9fQUsq4vce9EM95Jj97Q/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_604077</span>
  </div>
  <div class="_2_QraFYR_0">若你锁设置的 10 秒，如果你的某业务进程抢锁成功后，执行可能会超过 10 秒才成功，在这过程中如何避免锁被自动释放而出现的安全性问题呢?<br>加锁时，先设置一个过期时间，然后我们开启一个「守护线程」，定时去检测这个锁的失效时间，如果锁快要过期了，操作共享资源还未完成，那么就自动对锁进行「续期」，重新设置过期时间。<br><br>幸运的是，已经有一个库把这些工作都封装好了：Redisson。<br><br>Redisson 是一个 Java 语言实现的 Redis SDK 客户端，在使用分布式锁时，它就采用了「自动续期」的方案来避免锁过期，这个守护线程我们一般也把它叫做「看门狗」线程。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-16 14:53:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1c/dc/b0/5b652423.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Yang</span>
  </div>
  <div class="_2_QraFYR_0">etcd实现的分布式所能保证高性能不</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-02 22:50:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/78/54/0aafbf5e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>雄哼哼</span>
  </div>
  <div class="_2_QraFYR_0">redis主备切换过程中，备的数据还没追上的时候 应该是禁止写入的，所以redis的set nx是写不进去的</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-16 23:18:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/36/fb/b480f2ac.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>人间理想</span>
  </div>
  <div class="_2_QraFYR_0">讲的真的好，深入浅出</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-08 15:42:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83ep0Cb1HGLBTD57I53ZLsIBnvN3YkJOTkibWyibPoCUM5cbnhqDicm1aKWTUFeI7SEd8REnibfZVWeM3BQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Mr.zhou</span>
  </div>
  <div class="_2_QraFYR_0">我自己使用etcd实现的一个小分布式锁的小栗子，不知道这么实现对不对<br>package main<br><br>import (<br>	&quot;context&quot;<br>	&quot;fmt&quot;<br>	&quot;github.com&#47;coreos&#47;etcd&#47;clientv3&quot;<br>	&quot;github.com&#47;coreos&#47;etcd&#47;clientv3&#47;concurrency&quot;<br>	&quot;sync&quot;<br>)<br><br>func main() {<br>	cli, _ := clientv3.New(clientv3.Config{<br>		Endpoints: []string{&quot;192.168.2.45:2379&quot;, &quot;192.168.2.46:2379&quot;, &quot;192.168.2.47:2379&quot;},<br>	})<br>	defer cli.Close()<br>	session, _ := concurrency.NewSession(cli)<br>	defer session.Close()<br>	mutex := concurrency.NewMutex(session, &quot;&#47;mylock&#47;&quot;)<br><br>	wg := sync.WaitGroup{}<br>	wg.Add(2)<br>	sum := 0<br>	go func() {<br>		defer wg.Done()<br>		for i := 0; i &lt; 100; i++ {<br>			fmt.Println(&quot;1号协程拿到了锁&quot;)<br>			mutex.Lock(context.Background())<br>			sum = sum + 1<br>			fmt.Println(&quot;1号协程中sum是 &quot;, sum)<br>			mutex.Unlock(context.Background())<br>			fmt.Println(&quot;1号协程释放了锁&quot;)<br>		}<br><br>	}()<br>	go func() {<br>		defer wg.Done()<br>		for i := 0; i &lt; 100; i++ {<br>			mutex.Lock(context.Background())<br>			fmt.Println(&quot;2号协程拿到了锁&quot;)<br>			sum = sum - 1<br>			fmt.Println(&quot;2号协程中sum是 &quot;, sum)<br>			mutex.Unlock(context.Background())<br>			fmt.Println(&quot;2号协程释放了锁&quot;)<br>		}<br>	}()<br>	<br>	wg.Wait()<br>	fmt.Println(sum)<br>}<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-21 16:14:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/59/31/3820fbb7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>星星</span>
  </div>
  <div class="_2_QraFYR_0">强行吹 <br>Redisson 或者你自己写个watchdog 都可以解决续期问题 <br>本质上zk 的心跳维持就是 client 在自动续期<br><br>为什么这么多人使用redis 还是因为redis 读写性能远超 zk etcd</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-17 23:46:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/46/12/965a6cc9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>菠萝power</span>
  </div>
  <div class="_2_QraFYR_0">这个和lock API有什么区别吗 server&#47;etcdserver&#47;api&#47;v3lock&#47;v3lockpb&#47;v3lock.proto</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-08 16:16:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/e2/c0/e7a59706.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>chongsheng</span>
  </div>
  <div class="_2_QraFYR_0">如果用etcd，能指定锁的自动释放时间吗？我看NewSession方法里对lease做了keepalive，只要不unlock或者崩溃，是不是一直持有锁呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-28 11:26:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/21/b8/aca814dd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>江山如画</span>
  </div>
  <div class="_2_QraFYR_0">我最近在做的工作中遇到了分布式锁，这篇文章解决了工作中的很多疑惑，忍不住想要分享给同事们，逻辑合理，切中要害，写的太赞了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢分享，最近我在极客时间直播中也分享了更多k8s最佳实践案例, 后面也将整理到专栏中。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-11 17:51:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/07/d2/0d7ee298.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>惘 闻</span>
  </div>
  <div class="_2_QraFYR_0">如果获取锁的client crash了。  其他的client监听到delete后获取到了锁，但是第一个client其实只是因为网络原因没有发送lease，但是它恢复后继续执行代码。就相当于一个资源被两个client操作了啊。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-11 19:54:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/22/0a/7d/ac715471.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>独孤九剑</span>
  </div>
  <div class="_2_QraFYR_0">面试不怕分布式锁问题了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-05 17:41:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/13/45/16c60da2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>@%初%@</span>
  </div>
  <div class="_2_QraFYR_0">我理解多个客户端写一个key，与多个客户端写多个key，可以理解为非公平锁与公平锁。主要还是要看业务场景处理。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，性能也要考虑下</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-13 10:12:17</div>
  </div>
</div>
</div>
</li>
</ul>