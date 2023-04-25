<audio title="07 _ MVCC：如何实现多版本并发控制？" src="https://static001.geekbang.org/resource/audio/a8/a2/a8b1a403865a93f6330e8127975d1da2.mp3" controls="controls"></audio> 
<p>你好，我是唐聪。</p><p>在<a href="https://time.geekbang.org/column/article/335204">01</a>课里，我和你介绍etcd v2时，提到过它存在的若干局限，如仅保留最新版本key-value数据、丢弃历史版本。而etcd核心特性watch又依赖历史版本，因此etcd v2为了缓解这个问题，会在内存中维护一个较短的全局事件滑动窗口，保留最近的1000条变更事件。但是在集群写请求较多等场景下，它依然无法提供可靠的Watch机制。</p><p>那么不可靠的etcd v2事件机制，在etcd v3中是如何解决的呢？</p><p>我今天要和你分享的MVCC（Multiversion concurrency control）机制，正是为解决这个问题而诞生的。</p><p>MVCC机制的核心思想是保存一个key-value数据的多个历史版本，etcd基于它不仅实现了可靠的Watch机制，避免了client频繁发起List Pod等expensive request操作，保障etcd集群稳定性。而且MVCC还能以较低的并发控制开销，实现各类隔离级别的事务，保障事务的安全性，是事务特性的基础。</p><p>希望通过本节课，帮助你搞懂MVCC含义和MVCC机制下key-value数据的更新、查询、删除原理，了解treeIndex索引模块、boltdb模块是如何相互协作，实现保存一个key-value数据多个历史版本。</p><!-- [[[read_end]]] --><h2>什么是MVCC</h2><p>首先和你聊聊什么是MVCC，从名字上理解，它是一个基于多版本技术实现的一种并发控制机制。那常见的并发机制有哪些？MVCC的优点在哪里呢？</p><p>提到并发控制机制你可能就没那么陌生了，比如数据库中的悲观锁，也就是通过锁机制确保同一时刻只能有一个事务对数据进行修改操作，常见的实现方案有读写锁、互斥锁、两阶段锁等。</p><p>悲观锁是一种事先预防机制，它悲观地认为多个并发事务可能会发生冲突，因此它要求事务必须先获得锁，才能进行修改数据操作。但是悲观锁粒度过大、高并发场景下大量事务会阻塞等，会导致服务性能较差。</p><p><strong>MVCC机制正是基于多版本技术实现的一种乐观锁机制</strong>，它乐观地认为数据不会发生冲突，但是当事务提交时，具备检测数据是否冲突的能力。</p><p>在MVCC数据库中，你更新一个key-value数据的时候，它并不会直接覆盖原数据，而是新增一个版本来存储新的数据，每个数据都有一个版本号。版本号它是一个逻辑时间，为了方便你深入理解版本号意义，在下面我给你画了一个etcd MVCC版本号时间序列图。</p><p>从图中你可以看到，随着时间增长，你每次修改操作，版本号都会递增。每修改一次，生成一条新的数据记录。<strong>当你指定版本号读取数据时，它实际上访问的是版本号生成那个时间点的快照数据</strong>。当你删除数据的时候，它实际也是新增一条带删除标识的数据记录。</p><p><img src="https://static001.geekbang.org/resource/image/1f/2c/1fbf4aa426c8b78570ed310a8c9e2c2c.png?wh=1920*806" alt=""></p><h2>MVCC特性初体验</h2><p>了解完什么是MVCC后，我先通过几个简单命令，带你初体验下MVCC特性，看看它是如何帮助你查询历史修改记录，以及找回不小心删除的key的。</p><p>启动一个空集群，更新两次key hello后，如何获取key hello的上一个版本值呢？ 删除key hello后，还能读到历史版本吗?</p><p>如下面的命令所示，第一次key hello更新完后，我们通过get命令获取下它的key-value详细信息。正如你所看到的，除了key、value信息，还有各类版本号，我后面会详细和你介绍它们的含义。这里我们重点关注mod_revision，它表示key最后一次修改时的etcd版本号。</p><p>当我们再次更新key hello为world2后，然后通过查询时指定key第一次更新后的版本号，你会发现我们查询到了第一次更新的值，甚至我们执行删除key hello后，依然可以获得到这个值。那么etcd是如何实现的呢?</p><pre><code># 更新key hello为world1
$ etcdctl put hello world1
OK
# 通过指定输出模式为json,查看key hello更新后的详细信息
$ etcdctl get hello -w=json
{
    &quot;kvs&quot;:[
        {
            &quot;key&quot;:&quot;aGVsbG8=&quot;,
            &quot;create_revision&quot;:2,
            &quot;mod_revision&quot;:2,
            &quot;version&quot;:1,
            &quot;value&quot;:&quot;d29ybGQx&quot;
        }
    ],
    &quot;count&quot;:1
}
# 再次修改key hello为world2
$ etcdctl put hello world2
OK
# 确认修改成功,最新值为wolrd2
$ etcdctl get hello
hello
world2
# 指定查询版本号,获得了hello上一次修改的值
$ etcdctl get hello --rev=2
hello
world1
# 删除key hello
$ etcdctl del  hello
1
# 删除后指定查询版本号3,获得了hello删除前的值
$ etcdctl get hello --rev=3
hello
world2
</code></pre><h2>整体架构</h2><p>在详细和你介绍etcd如何实现MVCC特性前，我先和你从整体上介绍下MVCC模块。下图是MVCC模块的一个整体架构图，整个MVCC特性由treeIndex、Backend/boltdb组成。</p><p>当你执行MVCC特性初体验中的put命令后，请求经过gRPC KV Server、Raft模块流转，对应的日志条目被提交后，Apply模块开始执行此日志内容。</p><p><img src="https://static001.geekbang.org/resource/image/f5/2c/f5799da8d51a381527068a95bb13592c.png?wh=1920*1064" alt=""></p><p>Apply模块通过MVCC模块来执行put请求，持久化key-value数据。MVCC模块将请求请划分成两个类别，分别是读事务（ReadTxn）和写事务（WriteTxn）。读事务负责处理range请求，写事务负责put/delete操作。读写事务基于treeIndex、Backend/boltdb提供的能力，实现对key-value的增删改查功能。</p><p>treeIndex模块基于内存版B-tree实现了key索引管理，它保存了用户key与版本号（revision）的映射关系等信息。</p><p>Backend模块负责etcd的key-value持久化存储，主要由ReadTx、BatchTx、Buffer组成，ReadTx定义了抽象的读事务接口，BatchTx在ReadTx之上定义了抽象的写事务接口，Buffer是数据缓存区。</p><p>etcd设计上支持多种Backend实现，目前实现的Backend是boltdb。boltdb是一个基于B+ tree实现的、支持事务的key-value嵌入式数据库。</p><p>treeIndex与boltdb关系你可参考下图。当你发起一个get hello命令时，从treeIndex中获取key的版本号，然后再通过这个版本号，从boltdb获取value信息。boltdb的value是包含用户key-value、各种版本号、lease信息的结构体。</p><p><img src="https://static001.geekbang.org/resource/image/e7/8f/e713636c6cf9c46c7c19f677232d858f.png?wh=1920*903" alt=""></p><p>接下来我和你重点聊聊treeIndex模块的原理与核心数据结构。</p><h2>treeIndex原理</h2><p>为什么需要treeIndex模块呢?</p><p>对于etcd v2来说，当你通过etcdctl发起一个put hello操作时，etcd v2直接更新内存树，这就导致历史版本直接被覆盖，无法支持保存key的历史版本。在etcd v3中引入treeIndex模块正是为了解决这个问题，支持保存key的历史版本，提供稳定的Watch机制和事务隔离等能力。</p><p>那etcd v3又是如何基于treeIndex模块，实现保存key的历史版本的呢?</p><p>在02节课里，我们提到过etcd在每次修改key时会生成一个全局递增的版本号（revision），然后通过数据结构B-tree保存用户key与版本号之间的关系，再以版本号作为boltdb key，以用户的key-value等信息作为boltdb value，保存到boltdb。</p><p>下面我就为你介绍下，etcd保存用户key与版本号映射关系的数据结构B-tree，为什么etcd使用它而不使用哈希表、平衡二叉树？</p><p>从etcd的功能特性上分析， 因etcd支持范围查询，因此保存索引的数据结构也必须支持范围查询才行。所以哈希表不适合，而B-tree支持范围查询。</p><p>从性能上分析，平横二叉树每个节点只能容纳一个数据、导致树的高度较高，而B-tree每个节点可以容纳多个数据，树的高度更低，更扁平，涉及的查找次数更少，具有优越的增、删、改、查性能。</p><p>Google的开源项目btree，使用Go语言实现了一个内存版的B-tree，对外提供了简单易用的接口。etcd正是基于btree库实现了一个名为treeIndex的索引模块，通过它来查询、保存用户key与版本号之间的关系。</p><p>下图是个最大度（degree &gt; 1，简称d）为5的B-tree，度是B-tree中的一个核心参数，它决定了你每个节点上的数据量多少、节点的“胖”、“瘦”程度。</p><p>从图中你可以看到，节点越胖，意味着一个节点可以存储更多数据，树的高度越低。在一个度为d的B-tree中，节点保存的最大key数为2d - 1，否则需要进行平衡、分裂操作。这里你要注意的是在etcd treeIndex模块中，创建的是最大度32的B-tree，也就是一个叶子节点最多可以保存63个key。</p><p><img src="https://static001.geekbang.org/resource/image/44/74/448c8a2bb3b5d2d48dfb6ea585172c74.png?wh=1920*934" alt=""></p><p>从图中你可以看到，你通过put/txn命令写入的一系列key，treeIndex模块基于B-tree将其组织起来，节点之间基于用户key比较大小。当你查找一个key k95时，通过B-tree的特性，你仅需通过图中流程1和2两次快速比较，就可快速找到k95所在的节点。</p><p>在treeIndex中，每个节点的key是一个keyIndex结构，etcd就是通过它保存了用户的key与版本号的映射关系。</p><p>那么keyIndex结构包含哪些信息呢？下面是字段说明，你可以参考一下。</p><pre><code>type keyIndex struct {
   key         []byte //用户的key名称，比如我们案例中的&quot;hello&quot;
   modified    revision //最后一次修改key时的etcd版本号,比如我们案例中的刚写入hello为world1时的，版本号为2
   generations []generation //generation保存了一个key若干代版本号信息，每代中包含对key的多次修改的版本号列表
}
</code></pre><p>keyIndex中包含用户的key、最后一次修改key时的etcd版本号、key的若干代（generation）版本号信息，每代中包含对key的多次修改的版本号列表。那我们要如何理解generations？为什么它是个数组呢?</p><p>generations表示一个key从创建到删除的过程，每代对应key的一个生命周期的开始与结束。当你第一次创建一个key时，会生成第0代，后续的修改操作都是在往第0代中追加修改版本号。当你把key删除后，它就会生成新的第1代，一个key不断经历创建、删除的过程，它就会生成多个代。</p><p>generation结构详细信息如下：</p><pre><code>type generation struct {
   ver     int64    //表示此key的修改次数
   created revision //表示generation结构创建时的版本号
   revs    []revision //每次修改key时的revision追加到此数组
}

</code></pre><p>generation结构中包含此key的修改次数、generation创建时的版本号、对此key的修改版本号记录列表。</p><p>你需要注意的是版本号（revision）并不是一个简单的整数，而是一个结构体。revision结构及含义如下：</p><pre><code>type revision struct {
   main int64    // 一个全局递增的主版本号，随put/txn/delete事务递增，一个事务内的key main版本号是一致的
   sub int64    // 一个事务内的子版本号，从0开始随事务内put/delete操作递增
}
</code></pre><p>revision包含main和sub两个字段，main是全局递增的版本号，它是个etcd逻辑时钟，随着put/txn/delete等事务递增。sub是一个事务内的子版本号，从0开始随事务内的put/delete操作递增。</p><p>比如启动一个空集群，全局版本号默认为1，执行下面的txn事务，它包含两次put、一次get操作，那么按照我们上面介绍的原理，全局版本号随读写事务自增，因此是main为2，sub随事务内的put/delete操作递增，因此key hello的revison为{2,0}，key world的revision为{2,1}。</p><pre><code>$ etcdctl txn -i
compares:


success requests (get，put，del):
put hello 1
get hello
put world 2
</code></pre><p>介绍完treeIndex基本原理、核心数据结构后，我们再看看在MVCC特性初体验中的更新、查询、删除key案例里，treeIndex与boltdb是如何协作，完成以上key-value操作的?</p><h2>MVCC更新key原理</h2><p>当你通过etcdctl发起一个put hello操作时，如下面的put事务流程图流程一所示，在put写事务中，首先它需要从treeIndex模块中查询key的keyIndex索引信息，keyIndex中存储了key的创建版本号、修改的次数等信息，这些信息在事务中发挥着重要作用，因此会存储在boltdb的value中。</p><p>在我们的案例中，因为是第一次创建hello key，此时keyIndex索引为空。</p><p><img src="https://static001.geekbang.org/resource/image/84/e1/84377555cb4150ea7286c9ef3c5e17e1.png?wh=1920*1063" alt=""></p><p>其次etcd会根据当前的全局版本号（空集群启动时默认为1）自增，生成put hello操作对应的版本号revision{2,0}，这就是boltdb的key。</p><p>boltdb的value是mvccpb.KeyValue结构体，它是由用户key、value、create_revision、mod_revision、version、lease组成。它们的含义分别如下：</p><ul>
<li>create_revision表示此key创建时的版本号。在我们的案例中，key hello是第一次创建，那么值就是2。当你再次修改key hello的时候，写事务会从treeIndex模块查询hello第一次创建的版本号，也就是keyIndex.generations[i].created字段，赋值给create_revision字段；</li>
<li>mod_revision表示key最后一次修改时的版本号，即put操作发生时的全局版本号加1；</li>
<li>version表示此key的修改次数。每次修改的时候，写事务会从treeIndex模块查询hello已经历过的修改次数，也就是keyIndex.generations[i].ver字段，将ver字段值加1后，赋值给version字段。</li>
</ul><p>填充好boltdb的KeyValue结构体后，这时就可以通过Backend的写事务batchTx接口将key{2,0},value为mvccpb.KeyValue保存到boltdb的缓存中，并同步更新buffer，如上图中的流程二所示。</p><p>此时存储到boltdb中的key、value数据如下：</p><p><img src="https://static001.geekbang.org/resource/image/a2/ba/a245b18eabc86ea83a71349f49bdceba.jpg?wh=1807*397" alt=""></p><p>然后put事务需将本次修改的版本号与用户key的映射关系保存到treeIndex模块中，也就是上图中的流程三。</p><p>因为key hello是首次创建，treeIndex模块它会生成key hello对应的keyIndex对象，并填充相关数据结构。</p><p>keyIndex填充后的结果如下所示：</p><pre><code>key hello的keyIndex:
key:     &quot;hello&quot;
modified: &lt;2,0&gt;
generations:
[{ver:1,created:&lt;2,0&gt;,revisions: [&lt;2,0&gt;]} ]
</code></pre><p>我们来简易分析一下上面的结果。</p><ul>
<li>key为hello，modified为最后一次修改版本号&lt;2,0&gt;，key hello是首次创建的，因此新增一个generation代跟踪它的生命周期、修改记录；</li>
<li>generation的ver表示修改次数，首次创建为1，后续随着修改操作递增；</li>
<li>generation.created表示创建generation时的版本号为&lt;2,0&gt;；</li>
<li>revision数组保存对此key修改的版本号列表，每次修改都会将将相应的版本号追加到revisions数组中。</li>
</ul><p>通过以上流程，一个put操作终于完成。</p><p>但是此时数据还并未持久化，为了提升etcd的写吞吐量、性能，一般情况下（默认堆积的写事务数大于1万才在写事务结束时同步持久化），数据持久化由Backend的异步goroutine完成，它通过事务批量提交，定时将boltdb页缓存中的脏数据提交到持久化存储磁盘中，也就是下图中的黑色虚线框住的流程四。</p><p><img src="https://static001.geekbang.org/resource/image/5d/a2/5de49651cedf4595648aeba3c131cea2.png?wh=1920*1059" alt=""></p><h2>MVCC查询key原理</h2><p>完成put hello为world1操作后，这时你通过etcdctl发起一个get hello操作，MVCC模块首先会创建一个读事务对象（TxnRead），在etcd 3.4中Backend实现了ConcurrentReadTx， 也就是并发读特性。</p><p>并发读特性的核心原理是创建读事务对象时，它会全量拷贝当前写事务未提交的buffer数据，并发的读写事务不再阻塞在一个buffer资源锁上，实现了全并发读。</p><p><img src="https://static001.geekbang.org/resource/image/55/ee/55998d8a1f3091076a9119d85e7175ee.png?wh=1920*1070" alt=""></p><p>如上图所示，在读事务中，它首先需要根据key从treeIndex模块获取版本号，因我们未带版本号读，默认是读取最新的数据。treeIndex模块从B-tree中，根据key查找到keyIndex对象后，匹配有效的generation，返回generation的revisions数组中最后一个版本号{2,0}给读事务对象。</p><p>读事务对象根据此版本号为key，通过Backend的并发读事务（ConcurrentReadTx）接口，优先从buffer中查询，命中则直接返回，否则从boltdb中查询此key的value信息。</p><p>那指定版本号读取历史记录又是怎么实现的呢？</p><p>当你再次发起一个put hello为world2修改操作时，key hello对应的keyIndex的结果如下面所示，keyIndex.modified字段更新为&lt;3,0&gt;，generation的revision数组追加最新的版本号&lt;3,0&gt;，ver修改为2。</p><pre><code>key hello的keyIndex:
key:     &quot;hello&quot;
modified: &lt;3,0&gt;
generations:
[{ver:2,created:&lt;2,0&gt;,revisions: [&lt;2,0&gt;,&lt;3,0&gt;]}]
</code></pre><p>boltdb插入一个新的key revision{3,0}，此时存储到boltdb中的key-value数据如下：</p><p><img src="https://static001.geekbang.org/resource/image/8b/f7/8bec06d61622f2a99ea9dd2f78e693f7.jpg?wh=1432*477" alt=""></p><p>这时你再发起一个指定历史版本号为2的读请求时，实际是读版本号为2的时间点的快照数据。treeIndex模块会遍历generation内的历史版本号，返回小于等于2的最大历史版本号，在我们这个案例中，也就是revision{2,0}，以它作为boltdb的key，从boltdb中查询出value即可。</p><h2>MVCC删除key原理</h2><p>介绍完MVCC更新、查询key的原理后，我们接着往下看。当你执行etcdctl del hello命令时，etcd会立刻从treeIndex和boltdb中删除此数据吗？还是增加一个标记实现延迟删除（lazy delete）呢？</p><p>答案为etcd实现的是延期删除模式，原理与key更新类似。</p><p>与更新key不一样之处在于，一方面，生成的boltdb key版本号{4,0,t}追加了删除标识（tombstone,简写t），boltdb value变成只含用户key的KeyValue结构体。另一方面treeIndex模块也会给此key hello对应的keyIndex对象，追加一个空的generation对象，表示此索引对应的key被删除了。</p><p>当你再次查询hello的时候，treeIndex模块根据key hello查找到keyindex对象后，若发现其存在空的generation对象，并且查询的版本号大于等于被删除时的版本号，则会返回空。</p><p>etcdctl hello操作后的keyIndex的结果如下面所示：</p><pre><code>key hello的keyIndex:
key:     &quot;hello&quot;
modified: &lt;4,0&gt;
generations:
[
{ver:3,created:&lt;2,0&gt;,revisions: [&lt;2,0&gt;,&lt;3,0&gt;,&lt;4,0&gt;(t)]}，             
{empty}
]
</code></pre><p>boltdb此时会插入一个新的key revision{4,0,t}，此时存储到boltdb中的key-value数据如下：</p><p><img src="https://static001.geekbang.org/resource/image/da/17/da4e5bc5033619dda296c022ac6yyc17.jpg?wh=1611*581" alt=""></p><p>那么key打上删除标记后有哪些用途呢？什么时候会真正删除它呢？</p><p>一方面删除key时会生成events，Watch模块根据key的删除标识，会生成对应的Delete事件。</p><p>另一方面，当你重启etcd，遍历boltdb中的key构建treeIndex内存树时，你需要知道哪些key是已经被删除的，并为对应的key索引生成tombstone标识。而真正删除treeIndex中的索引对象、boltdb中的key是通过压缩(compactor)组件异步完成。</p><p>正因为etcd的删除key操作是基于以上延期删除原理实现的，因此只要压缩组件未回收历史版本，我们就能从etcd中找回误删的数据。</p><h2>小结</h2><p>最后我们来小结下今天的内容，我通过MVCC特性初体验中的更新、查询、删除key案例，为你分析了MVCC整体架构、核心模块，它由treeIndex、boltdb组成。</p><p>treeIndex模块基于Google开源的btree库实现，它的核心数据结构keyIndex，保存了用户key与版本号关系。每次修改key都会生成新的版本号，生成新的boltdb key-value。boltdb的key为版本号，value包含用户key-value、各种版本号、lease的mvccpb.KeyValue结构体。</p><p>当你未带版本号查询key时，etcd返回的是key最新版本数据。当你指定版本号读取数据时，etcd实际上返回的是版本号生成那个时间点的快照数据。</p><p>删除一个数据时，etcd并未真正删除它，而是基于lazy delete实现的异步删除。删除原理本质上与更新操作类似，只不过boltdb的key会打上删除标记，keyIndex索引中追加空的generation。真正删除key是通过etcd的压缩组件去异步实现的，在后面的课程里我会继续和你深入介绍。</p><p>基于以上原理特性的实现，etcd实现了保存key历史版本的功能，是高可靠Watch机制的基础。基于key-value中的各种版本号信息，etcd可提供各种级别的简易事务隔离能力。基于Backend/boltdb提供的MVCC机制，etcd可实现读写不冲突。</p><h2>思考题</h2><p>你认为etcd为什么删除使用lazy delete方式呢？ 相比同步delete,各有什么优缺点？当你突然删除大量key后，db大小是立刻增加还是减少呢？</p><p>感谢你的阅读，如果你认为这节课的内容有收获，也欢迎把它分享给你的朋友，谢谢。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/51/1d/50085e60.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>五味子</span>
  </div>
  <div class="_2_QraFYR_0">我理解etcd采用延迟删除，1是为了保证key对应的watcher能够获取到key的所有状态信息，留给watcher时间做相应的处理。2是实时从boltdb删除key，会可能触发树的不平衡，影响其他读写请求的性能。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞，理解很到位</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-04 16:01:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/aa/9a/92d2df36.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>tianfeiyu</span>
  </div>
  <div class="_2_QraFYR_0">一般情况下（默认堆积的写事务数大于 1 万才在写事务结束时同步持久化），数据持久化由 Backend 的异步 goroutine 完成，它通过事务批量提交，定时将 boltdb 页缓存中的脏数据提交到持久化存储磁盘中<br>---<br>如果etcd集群突然挂了，如何保证这部分未持久化的数据不会丢呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 重启时会重放wal日志中已提交的日志条目再次执行</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-07 21:26:23</div>
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
  <div class="_2_QraFYR_0">并发读特性的核心原理是创建读事务对象时，它会全量拷贝当前写事务未提交的 buffer 数据，并发的读写事务不再阻塞在一个 buffer 资源锁上，实现了全并发读。<br><br>是否可以具体讲讲全量拷贝如何实现全并发读？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-01 15:34:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c4/eb/2285a345.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>花晨少年</span>
  </div>
  <div class="_2_QraFYR_0">并发读特性的核心原理是创建读事务对象时，它会全量拷贝当前写事务未提交的 buffer 数据，并发的读写事务不再阻塞在一个 buffer 资源锁上，实现了全并发读。<br>---------------<br>写事务未提交，为什么读事务要去读这个脏数据呢？另外写事务的写buffer是这个事务所有操作一起写bufeer吗，我们保证原子的写呢，在不锁的情况下？<br>我理解是如果对读事务来看，想让写事务具有原子性，应该必须得加锁吧。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 第一个问题，未提交的buffer，并不是脏数据，参考下03 etcd写原理，为了优化性能，etcd并不会来一个put请求就发起一次boltdb事务提交，将数据持久化到db文件，而是将多个put和txn等请求合并异步提交(未大量写操作堆积时)，它们一方面会更新buffer，一方面会更新boltdb内存数据结构。这里对两个事务可能让你困惑了，一个是你应用层发起的txn等操作，一个是boltdb事务，后者一个事务可提交多个txn和put等操作。<br>第二个问题，有加锁的哈，每个写事务都需要获取一个mvcc全局写锁才能更新哈</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-18 20:30:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/46/0f/f6cfc659.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>mckee</span>
  </div>
  <div class="_2_QraFYR_0">思考题：<br>etcd 为什么删除使用 lazy delete 方式呢？相比同步 delete, 各有什么优缺点？<br>etcd要保存key的历史版本，直接删除就不能支持revision查询了；<br>lazy方式性能更高，空闲空间可以再利用；<br><br>当你突然删除大量 key 后，db 大小是立刻增加还是减少呢？<br>应该会增大，etcd不会立即把空间返回系统而是维护起来后续使用，维护空闲页面应该需要一些内存；</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-05 16:07:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83ep075ibtmxMf3eOYlBJ96CE9TEelLUwePaLqp8M75gWHEcM3za0voylA0oe9y3NiaboPB891rypRt7w/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>shuff1e</span>
  </div>
  <div class="_2_QraFYR_0">当你再次查询 hello 的时候，treeIndex 模块根据 key hello 查找到 keyindex 对象后，若发现其存在空的 generation 对象，并且查询的版本号大于被删除时的版本号，则会返回空。<br>---<br>如果删除了之后，又重新写入了。<br>查询的最新的版本号，还是会返回最新的数据的吧。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，是的，新写入后会生成新的generation, 匹配generation过程中会优先匹配最新的一代，然后从中返回最后一次修改的版本号，就可从boltdb查询到最新的数据</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-03 21:11:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/aa/9a/92d2df36.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>tianfeiyu</span>
  </div>
  <div class="_2_QraFYR_0">一般情况下（默认堆积的写事务数大于 1 万才在写事务结束时同步持久化），数据持久化由 Backend 的异步 goroutine 完成，它通过事务批量提交，定时将 boltdb 页缓存中的脏数据提交到持久化存储磁盘中<br>---<br>如果 etcd 集群突然挂了，如何保证未持久化的这部分数据不会丢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-07 21:24:26</div>
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
  <div class="_2_QraFYR_0"> 老师每讲内容太丰富了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-03 09:16:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/20/db/50/c2d07bb1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>L。</span>
  </div>
  <div class="_2_QraFYR_0">老师，请教个问题，treeindex是采用btree来实现的索引，那么请问一下， 这个索引是在内存中，还是会落盘</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-06 19:16:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/20/c3/94/e89ebc50.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>神毓逍遥</span>
  </div>
  <div class="_2_QraFYR_0">结合MySQL的MVCC一起看</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-13 20:07:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/61/e6/fedd20dc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>mmm</span>
  </div>
  <div class="_2_QraFYR_0">“基于 Backend&#47;boltdb 提供的 MVCC 机制，etcd 可实现读写不冲突。”<br>老师buffer由于全拷贝实现了并发读，那treeindex和boltdb读写如何做到不冲突呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-04 09:41:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/20/c3/34/58426785.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>预见</span>
  </div>
  <div class="_2_QraFYR_0">这跟mysql的很多实现都好像啊</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-21 21:17:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/c4/03/f753fda7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>JianXu</span>
  </div>
  <div class="_2_QraFYR_0">“当你再次发起一个 put hello 为 world2 修改操作时，key hello 对应的 keyIndex 的结果如下面所示，keyIndex.modified 字段更新为 &lt;3,0&gt;，generation 的 revision 数组追加最新的版本号 &lt;3,0&gt;，ver 修改为 2。”<br><br>老师，hello world 和 hello world2 属于同一个transaction 吗？如果属于，那是不是revision 数组最新版本是&lt;3,1&gt; 呢？只有在不属于一个Tx 的时候才是 &lt;3,0&gt; </div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-27 16:09:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/uQoCBsia00Dr1g05SCZ69esjDwJWP4QGbckxNZAO44xg4Hu2YjDROoITtvcLr23ae9SrE5tVR95U8ricVMicdnUIw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_439a6d</span>
  </div>
  <div class="_2_QraFYR_0">在 MVCC put  key 的过程中，要是 treeIndex 和 boltdb 的修改成功了（从上面看起来，应该是可以向用户返回 put 成功），持久化失败了，这时候会不会导致丢数据？是因为，多个 etcd 节点之间进行了数据同步，单个节点死掉以后，其他节点还是会持久化，所以保证了不会丢数据吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-14 17:21:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/uQoCBsia00Dr1g05SCZ69esjDwJWP4QGbckxNZAO44xg4Hu2YjDROoITtvcLr23ae9SrE5tVR95U8ricVMicdnUIw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_439a6d</span>
  </div>
  <div class="_2_QraFYR_0">revision 中main 是个全局递增的int64 整数，随着用户不断对etcd 中的数据进行操作，main 不会溢出吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-14 17:08:35</div>
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
  <div class="_2_QraFYR_0">boltdb 使用的 唯一key，怎么产生的？用的什么算法？是整个etcd集群从有第一个写请求后，从1开始自增得到的？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-11 13:17:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/oiboHpgukqib2ASXeU0H7W15zhRusOohD37CFCxSOnZfeOppkicDpOLVVmQYlbpw6rGib2Ib8smSFHZiaXXa7OJhHTQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>scott</span>
  </div>
  <div class="_2_QraFYR_0">感谢分享，我有两个疑问：<br>1. ConcurrentReadTx会全量拷贝buffer，这样就不会阻塞写事务，但是这个buffer不会是stale吗？因为写事务会writeback buffer，但是好像没有看到更新ConcurrentReadTx拷贝的buffer。<br>2. `server&#47;etcdserver&#47;apply.go`文件中的`applierV3backend.Txn`方法，在执行pb.TxnRequest事务时，先去执行compareToPath方法，判断是走Then还是Else分支中的`[]*RequestOp`，这个过程是启动一个readTx，然后关闭此readTx，接着开启一个writeTX执行applyTxn，但是有没有可能前面判断的compareToPath已经被更改了，导致执行的分支不准确？<br>谢谢！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-18 23:31:56</div>
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
  <div class="_2_QraFYR_0">数据持久化由 Backend 的异步 goroutine 完成，它通过事务批量提交，定时将 boltdb 页缓存中的脏数据提交到持久化存储磁盘中<br>请问下，此时的异步批量提交是一次性将所有的脏数据都持久化的存储磁盘还是每次持久化部分脏数据？<br>个人理解是全部刷入,因为部分刷入无法准备记录consistent index，请作者解答下，谢谢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-26 23:54:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/b9/bb/71c0f013.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Ching</span>
  </div>
  <div class="_2_QraFYR_0">“全局版本号随读写事务自增，因此是 main 为 2，sub 随事务内的 put&#47;delete 操作递增，因此 key hello 的 revison 为{2,0}，key world 的 revision 为{2,1}。” 老师请问一下，读事物也会递增全局版本号吗？然后这个子版本号，在这个例子里有两个put，为什么不是递增2呢？是一个事物内的全部写操作只看作1次子版本递增吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: ching你好, 读事务不会的，一个txn写事务只会递增一次全局版本号，若有若干个写操作，sub子版本号会递增多次</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-18 22:51:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/28/70/13/accdc2df.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>M⃰I⃰ ⃰m⃰a⃰n⃰c⃰h⃰i⃰</span>
  </div>
  <div class="_2_QraFYR_0">在一个度为 d 的 B-tree 中，节点保存的最大 key 数为 2d - 1<br>--这里最大key数不应该是d-1么？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-02 15:23:58</div>
  </div>
</div>
</div>
</li>
</ul>