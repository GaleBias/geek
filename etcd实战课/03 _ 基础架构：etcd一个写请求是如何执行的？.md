<audio title="03 _ 基础架构：etcd一个写请求是如何执行的？" src="https://static001.geekbang.org/resource/audio/69/02/695df87f7b86f380b30f26a89d285f02.mp3" controls="controls"></audio> 
<p>你好，我是唐聪。</p><p>在上一节课里，我通过分析etcd的一个读请求执行流程，给你介绍了etcd的基础架构，让你初步了解了在etcd的读请求流程中，各个模块是如何紧密协作，执行查询语句，返回数据给client。</p><p>那么etcd一个写请求执行流程又是怎样的呢？在执行写请求过程中，如果进程crash了，如何保证数据不丢、命令不重复执行呢？</p><p>今天我就和你聊聊etcd写过程中是如何解决这些问题的。希望通过这节课，让你了解一个key-value写入的原理，对etcd的基础架构中涉及写请求相关的模块有一定的理解，同时能触类旁通，当你在软件项目开发过程中遇到类似数据安全、幂等性等问题时，能设计出良好的方案解决它。</p><h2>整体架构</h2><p><img src="https://static001.geekbang.org/resource/image/8b/72/8b6dfa84bf8291369ea1803387906c72.png?wh=1920*1265" alt=""></p><p>为了让你能够更直观地理解etcd的写请求流程，我在如上的架构图中，用序号标识了下面的一个put hello为world的写请求的简要执行流程，帮助你从整体上快速了解一个写请求的全貌。</p><pre><code>etcdctl put hello world --endpoints http://127.0.0.1:2379
OK

</code></pre><p>首先client端通过负载均衡算法选择一个etcd节点，发起gRPC调用。然后etcd节点收到请求后经过gRPC拦截器、Quota模块后，进入KVServer模块，KVServer模块向Raft模块提交一个提案，提案内容为“大家好，请使用put方法执行一个key为hello，value为world的命令”。</p><!-- [[[read_end]]] --><p>随后此提案通过RaftHTTP网络模块转发、经过集群多数节点持久化后，状态会变成已提交，etcdserver从Raft模块获取已提交的日志条目，传递给Apply模块，Apply模块通过MVCC模块执行提案内容，更新状态机。</p><p>与读流程不一样的是写流程还涉及Quota、WAL、Apply三个模块。crash-safe及幂等性也正是基于WAL和Apply流程的consistent index等实现的，因此今天我会重点和你介绍这三个模块。</p><p>下面就让我们沿着写请求执行流程图，从0到1分析一个key-value是如何安全、幂等地持久化到磁盘的。</p><h2>Quota模块</h2><p>首先是流程一client端发起gRPC调用到etcd节点，和读请求不一样的是，写请求需要经过流程二db配额（Quota）模块，它有什么功能呢？</p><p>我们先从此模块的一个常见错误说起，你在使用etcd过程中是否遇到过"etcdserver: mvcc: database space exceeded"错误呢？</p><p>我相信只要你使用过etcd或者Kubernetes，大概率见过这个错误。它是指当前etcd db文件大小超过了配额，当出现此错误后，你的整个集群将不可写入，只读，对业务的影响非常大。</p><p>哪些情况会触发这个错误呢？</p><p>一方面默认db配额仅为2G，当你的业务数据、写入QPS、Kubernetes集群规模增大后，你的etcd db大小就可能会超过2G。</p><p>另一方面我们知道etcd v3是个MVCC数据库，保存了key的历史版本，当你未配置压缩策略的时候，随着数据不断写入，db大小会不断增大，导致超限。</p><p>最后你要特别注意的是，如果你使用的是etcd 3.2.10之前的旧版本，请注意备份可能会触发boltdb的一个Bug，它会导致db大小不断上涨，最终达到配额限制。</p><p>了解完触发Quota限制的原因后，我们再详细了解下Quota模块它是如何工作的。</p><p>当etcd server收到put/txn等写请求的时候，会首先检查下当前etcd db大小加上你请求的key-value大小之和是否超过了配额（quota-backend-bytes）。</p><p>如果超过了配额，它会产生一个告警（Alarm）请求，告警类型是NO SPACE，并通过Raft日志同步给其它节点，告知db无空间了，并将告警持久化存储到db中。</p><p>最终，无论是API层gRPC模块还是负责将Raft侧已提交的日志条目应用到状态机的Apply模块，都拒绝写入，集群只读。</p><p>那遇到这个错误时应该如何解决呢？</p><p>首先当然是调大配额。具体多大合适呢？etcd社区建议不超过8G。遇到过这个错误的你是否还记得，为什么当你把配额（quota-backend-bytes）调大后，集群依然拒绝写入呢?</p><p>原因就是我们前面提到的NO SPACE告警。Apply模块在执行每个命令的时候，都会去检查当前是否存在NO SPACE告警，如果有则拒绝写入。所以还需要你额外发送一个取消告警（etcdctl alarm disarm）的命令，以消除所有告警。</p><p>其次你需要检查etcd的压缩（compact）配置是否开启、配置是否合理。etcd保存了一个key所有变更历史版本，如果没有一个机制去回收旧的版本，那么内存和db大小就会一直膨胀，在etcd里面，压缩模块负责回收旧版本的工作。</p><p>压缩模块支持按多种方式回收旧版本，比如保留最近一段时间内的历史版本。不过你要注意，它仅仅是将旧版本占用的空间打个空闲（Free）标记，后续新的数据写入的时候可复用这块空间，而无需申请新的空间。</p><p>如果你需要回收空间，减少db大小，得使用碎片整理（defrag）， 它会遍历旧的db文件数据，写入到一个新的db文件。但是它对服务性能有较大影响，不建议你在生产集群频繁使用。</p><p>最后你需要注意配额（quota-backend-bytes）的行为，默认'0'就是使用etcd默认的2GB大小，你需要根据你的业务场景适当调优。如果你填的是个小于0的数，就会禁用配额功能，这可能会让你的db大小处于失控，导致性能下降，不建议你禁用配额。</p><h2>KVServer模块</h2><p>通过流程二的配额检查后，请求就从API层转发到了流程三的KVServer模块的put方法，我们知道etcd是基于Raft算法实现节点间数据复制的，因此它需要将put写请求内容打包成一个提案消息，提交给Raft模块。不过KVServer模块在提交提案前，还有如下的一系列检查和限速。</p><h3>Preflight Check</h3><p>为了保证集群稳定性，避免雪崩，任何提交到Raft模块的请求，都会做一些简单的限速判断。如下面的流程图所示，首先，如果Raft模块已提交的日志索引（committed index）比已应用到状态机的日志索引（applied index）超过了5000，那么它就返回一个"etcdserver: too many requests"错误给client。</p><p><img src="https://static001.geekbang.org/resource/image/dc/54/dc8e373e06f2ab5f63a7948c4a6c8554.png?wh=1164*1004" alt=""></p><p>然后它会尝试去获取请求中的鉴权信息，若使用了密码鉴权、请求中携带了token，如果token无效，则返回"auth: invalid auth token"错误给client。</p><p>其次它会检查你写入的包大小是否超过默认的1.5MB， 如果超过了会返回"etcdserver: request is too large"错误给给client。</p><h3>Propose</h3><p>最后通过一系列检查之后，会生成一个唯一的ID，将此请求关联到一个对应的消息通知channel，然后向Raft模块发起（Propose）一个提案（Proposal），提案内容为“大家好，请使用put方法执行一个key为hello，value为world的命令”，也就是整体架构图里的流程四。</p><p>向Raft模块发起提案后，KVServer模块会等待此put请求，等待写入结果通过消息通知channel返回或者超时。etcd默认超时时间是7秒（5秒磁盘IO延时+2*1秒竞选超时时间），如果一个请求超时未返回结果，则可能会出现你熟悉的etcdserver: request timed out错误。</p><h2>WAL模块</h2><p>Raft模块收到提案后，如果当前节点是Follower，它会转发给Leader，只有Leader才能处理写请求。Leader收到提案后，通过Raft模块输出待转发给Follower节点的消息和待持久化的日志条目，日志条目则封装了我们上面所说的put hello提案内容。</p><p>etcdserver从Raft模块获取到以上消息和日志条目后，作为Leader，它会将put提案消息广播给集群各个节点，同时需要把集群Leader任期号、投票信息、已提交索引、提案内容持久化到一个WAL（Write Ahead Log）日志文件中，用于保证集群的一致性、可恢复性，也就是我们图中的流程五模块。</p><p>WAL日志结构是怎样的呢？</p><p><img src="https://static001.geekbang.org/resource/image/47/8d/479dec62ed1c31918a7c6cab8e6aa18d.png?wh=1920*1335" alt=""></p><p>上图是WAL结构，它由多种类型的WAL记录顺序追加写入组成，每个记录由类型、数据、循环冗余校验码组成。不同类型的记录通过Type字段区分，Data为对应记录内容，CRC为循环校验码信息。</p><p>WAL记录类型目前支持5种，分别是文件元数据记录、日志条目记录、状态信息记录、CRC记录、快照记录：</p><ul>
<li>文件元数据记录包含节点ID、集群ID信息，它在WAL文件创建的时候写入；</li>
<li>日志条目记录包含Raft日志信息，如put提案内容；</li>
<li>状态信息记录，包含集群的任期号、节点投票信息等，一个日志文件中会有多条，以最后的记录为准；</li>
<li>CRC记录包含上一个WAL文件的最后的CRC（循环冗余校验码）信息， 在创建、切割WAL文件时，作为第一条记录写入到新的WAL文件， 用于校验数据文件的完整性、准确性等；</li>
<li>快照记录包含快照的任期号、日志索引信息，用于检查快照文件的准确性。</li>
</ul><p>WAL模块又是如何持久化一个put提案的日志条目类型记录呢?</p><p>首先我们来看看put写请求如何封装在Raft日志条目里面。下面是Raft日志条目的数据结构信息，它由以下字段组成：</p><ul>
<li>Term是Leader任期号，随着Leader选举增加；</li>
<li>Index是日志条目的索引，单调递增增加；</li>
<li>Type是日志类型，比如是普通的命令日志（EntryNormal）还是集群配置变更日志（EntryConfChange）；</li>
<li>Data保存我们上面描述的put提案内容。</li>
</ul><pre><code>type Entry struct {
   Term             uint64    `protobuf:&quot;varint，2，opt，name=Term&quot; json:&quot;Term&quot;`
   Index            uint64    `protobuf:&quot;varint，3，opt，name=Index&quot; json:&quot;Index&quot;`
   Type             EntryType `protobuf:&quot;varint，1，opt，name=Type，enum=Raftpb.EntryType&quot; json:&quot;Type&quot;`
   Data             []byte    `protobuf:&quot;bytes，4，opt，name=Data&quot; json:&quot;Data，omitempty&quot;`
}
</code></pre><p>了解完Raft日志条目数据结构后，我们再看WAL模块如何持久化Raft日志条目。它首先先将Raft日志条目内容（含任期号、索引、提案内容）序列化后保存到WAL记录的Data字段， 然后计算Data的CRC值，设置Type为Entry Type， 以上信息就组成了一个完整的WAL记录。</p><p>最后计算WAL记录的长度，顺序先写入WAL长度（Len Field），然后写入记录内容，调用fsync持久化到磁盘，完成将日志条目保存到持久化存储中。</p><p>当一半以上节点持久化此日志条目后， Raft模块就会通过channel告知etcdserver模块，put提案已经被集群多数节点确认，提案状态为已提交，你可以执行此提案内容了。</p><p>于是进入流程六，etcdserver模块从channel取出提案内容，添加到先进先出（FIFO）调度队列，随后通过Apply模块按入队顺序，异步、依次执行提案内容。</p><h2>Apply模块</h2><p>执行put提案内容对应我们架构图中的流程七，其细节图如下。那么Apply模块是如何执行put请求的呢？若put请求提案在执行流程七的时候etcd突然crash了， 重启恢复的时候，etcd是如何找回异常提案，再次执行的呢？</p><p><img src="https://static001.geekbang.org/resource/image/7f/5b/7f13edaf28yy7a6698e647104771235b.png?wh=1920*641" alt=""></p><p>核心就是我们上面介绍的WAL日志，因为提交给Apply模块执行的提案已获得多数节点确认、持久化，etcd重启时，会从WAL中解析出Raft日志条目内容，追加到Raft日志的存储中，并重放已提交的日志提案给Apply模块执行。</p><p>然而这又引发了另外一个问题，如何确保幂等性，防止提案重复执行导致数据混乱呢?</p><p>我们在上一节课里讲到，etcd是个MVCC数据库，每次更新都会生成新的版本号。如果没有幂等性保护，同样的命令，一部分节点执行一次，一部分节点遭遇异常故障后执行多次，则系统的各节点一致性状态无法得到保证，导致数据混乱，这是严重故障。</p><p>因此etcd必须要确保幂等性。怎么做呢？Apply模块从Raft模块获得的日志条目信息里，是否有唯一的字段能标识这个提案？</p><p>答案就是我们上面介绍Raft日志条目中的索引（index）字段。日志条目索引是全局单调递增的，每个日志条目索引对应一个提案， 如果一个命令执行后，我们在db里面也记录下当前已经执行过的日志条目索引，是不是就可以解决幂等性问题呢？</p><p>是的。但是这还不够安全，如果执行命令的请求更新成功了，更新index的请求却失败了，是不是一样会导致异常？</p><p>因此我们在实现上，还需要将两个操作作为原子性事务提交，才能实现幂等。</p><p>正如我们上面的讨论的这样，etcd通过引入一个consistent index的字段，来存储系统当前已经执行过的日志条目索引，实现幂等性。</p><p>Apply模块在执行提案内容前，首先会判断当前提案是否已经执行过了，如果执行了则直接返回，若未执行同时无db配额满告警，则进入到MVCC模块，开始与持久化存储模块打交道。</p><h2>MVCC</h2><p>Apply模块判断此提案未执行后，就会调用MVCC模块来执行提案内容。MVCC主要由两部分组成，一个是内存索引模块treeIndex，保存key的历史版本号信息，另一个是boltdb模块，用来持久化存储key-value数据。那么MVCC模块执行put hello为world命令时，它是如何构建内存索引和保存哪些数据到db呢？</p><h3>treeIndex</h3><p>首先我们来看MVCC的索引模块treeIndex，当收到更新key hello为world的时候，此key的索引版本号信息是怎么生成的呢？需要维护、持久化存储一个全局版本号吗？</p><p>版本号（revision）在etcd里面发挥着重大作用，它是etcd的逻辑时钟。etcd启动的时候默认版本号是1，随着你对key的增、删、改操作而全局单调递增。</p><p>因为boltdb中的key就包含此信息，所以etcd并不需要再去持久化一个全局版本号。我们只需要在启动的时候，从最小值1开始枚举到最大值，未读到数据的时候则结束，最后读出来的版本号即是当前etcd的最大版本号currentRevision。</p><p>MVCC写事务在执行put hello为world的请求时，会基于currentRevision自增生成新的revision如{2,0}，然后从treeIndex模块中查询key的创建版本号、修改次数信息。这些信息将填充到boltdb的value中，同时将用户的hello key和revision等信息存储到B-tree，也就是下面简易写事务图的流程一，整体架构图中的流程八。</p><p><img src="https://static001.geekbang.org/resource/image/a1/ff/a19a06d8f4cc5e488a114090d84116ff.png?wh=1920*1035" alt=""></p><h3>boltdb</h3><p>MVCC写事务自增全局版本号后生成的revision{2,0}，它就是boltdb的key，通过它就可以往boltdb写数据了，进入了整体架构图中的流程九。</p><p>boltdb上一篇我们提过它是一个基于B+tree实现的key-value嵌入式db，它通过提供桶（bucket）机制实现类似MySQL表的逻辑隔离。</p><p>在etcd里面你通过put/txn等KV API操作的数据，全部保存在一个名为key的桶里面，这个key桶在启动etcd的时候会自动创建。</p><p>除了保存用户KV数据的key桶，etcd本身及其它功能需要持久化存储的话，都会创建对应的桶。比如上面我们提到的etcd为了保证日志的幂等性，保存了一个名为consistent index的变量在db里面，它实际上就存储在元数据（meta）桶里面。</p><p>那么写入boltdb的value含有哪些信息呢？</p><p>写入boltdb的value， 并不是简单的"world"，如果只存一个用户value，索引又是保存在易失的内存上，那重启etcd后，我们就丢失了用户的key名，无法构建treeIndex模块了。</p><p>因此为了构建索引和支持Lease等特性，etcd会持久化以下信息:</p><ul>
<li>key名称；</li>
<li>key创建时的版本号（create_revision）、最后一次修改时的版本号（mod_revision）、key自身修改的次数（version）；</li>
<li>value值；</li>
<li>租约信息（后面介绍）。</li>
</ul><p>boltdb value的值就是将含以上信息的结构体序列化成的二进制数据，然后通过boltdb提供的put接口，etcd就快速完成了将你的数据写入boltdb，对应上面简易写事务图的流程二。</p><p>但是put调用成功，就能够代表数据已经持久化到db文件了吗？</p><p>这里需要注意的是，在以上流程中，etcd并未提交事务（commit），因此数据只更新在boltdb所管理的内存数据结构中。</p><p>事务提交的过程，包含B+tree的平衡、分裂，将boltdb的脏数据（dirty page）、元数据信息刷新到磁盘，因此事务提交的开销是昂贵的。如果我们每次更新都提交事务，etcd写性能就会较差。</p><p>那么解决的办法是什么呢？etcd的解决方案是合并再合并。</p><p>首先boltdb key是版本号，put/delete操作时，都会基于当前版本号递增生成新的版本号，因此属于顺序写入，可以调整boltdb的bucket.FillPercent参数，使每个page填充更多数据，减少page的分裂次数并降低db空间。</p><p>其次etcd通过合并多个写事务请求，通常情况下，是异步机制定时（默认每隔100ms）将批量事务一次性提交（pending事务过多才会触发同步提交）， 从而大大提高吞吐量，对应上面简易写事务图的流程三。</p><p>但是这优化又引发了另外的一个问题， 因为事务未提交，读请求可能无法从boltdb获取到最新数据。</p><p>为了解决这个问题，etcd引入了一个bucket buffer来保存暂未提交的事务数据。在更新boltdb的时候，etcd也会同步数据到bucket buffer。因此etcd处理读请求的时候会优先从bucket buffer里面读取，其次再从boltdb读，通过bucket buffer实现读写性能提升，同时保证数据一致性。</p><h2>小结</h2><p>最后我们来小结一下，今天我给你介绍了etcd的写请求流程，重点介绍了Quota、WAL、Apply模块。</p><p>首先我们介绍了Quota模块工作原理和我们熟悉的database space exceeded错误触发原因，写请求导致db大小增加、compact策略不合理、boltdb Bug等都会导致db大小超限。</p><p>其次介绍了WAL模块的存储结构，它由一条条记录顺序写入组成，每个记录含有Type、CRC、Data，每个提案被提交前都会被持久化到WAL文件中，以保证集群的一致性和可恢复性。</p><p>随后我们介绍了Apply模块基于consistent index和事务实现了幂等性，保证了节点在异常情况下不会重复执行重放的提案。</p><p>最后我们介绍了MVCC模块是如何维护索引版本号、重启后如何从boltdb模块中获取内存索引结构的。以及etcd通过异步、批量提交事务机制，以提升写QPS和吞吐量。</p><p>通过以上介绍，希望你对etcd的一个写语句执行流程有个初步的理解，明白WAL模块、Apply模块、MVCC模块三者是如何相互协作的，从而实现在节点遭遇crash等异常情况下，不丢任何已提交的数据、不重复执行任何提案。</p><h2>思考题</h2><p>expensive read请求（如Kubernetes场景中查询大量pod）会影响写请求的性能吗？</p><p>你可以把你的思考和观点写在留言区里，我会在下一篇文章的末尾给出我的答案。</p><p>今天的课程就结束了，希望可以帮助到你，也希望你在下方的留言区和我参与讨论，同时欢迎你把这节课分享给你的朋友或者同事，一起交流一下。</p><h2>02思考题答案</h2><p>上节课我给大家留了一个思考题，评论中有同学说buffer没读到，从boltdb读时会产生磁盘I/O，这是一个常见误区。</p><p>实际上，etcd在启动的时候会通过mmap机制将etcd db文件映射到etcd进程地址空间，并设置了mmap的MAP_POPULATE flag，它会告诉Linux内核预读文件，Linux内核会将文件内容拷贝到物理内存中，此时会产生磁盘I/O。节点内存足够的请求下，后续处理读请求过程中就不会产生磁盘I/IO了。</p><p>若etcd节点内存不足，可能会导致db文件对应的内存页被换出，当读请求命中的页未在内存中时，就会产生缺页异常，导致读过程中产生磁盘IO，你可以通过观察etcd进程的majflt字段来判断etcd是否产生了主缺页中断。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/67/ae/37b492db.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>唐聪</span>
  </div>
  <div class="_2_QraFYR_0">之前修复的存在3年多的不一致bug就是跟本节介绍的写请求幂等性有关，在每讲中，我提及的bug，特性缺点，大部分都是我之前踩过的坑，这些经验希望能帮助大家提前避免不必要的线上问题</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-26 07:27:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/76/7d/04c95885.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Index</span>
  </div>
  <div class="_2_QraFYR_0">之前研究一段时间的etcd的源码，看的七七八八，现在再看这篇文章把之前的很多疑问都解答了，太棒了，etcd是个很优秀的项目，能把这么多的技术点融合在一起，实在是一个很好的开源学习项目。老师有空可以开直播，多聊聊etcd中涉及的技术点的一些学习，从源头上把知识融汇贯通，这样的学习真是酣畅淋漓</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 好的，谢谢你的认可，一起学习加油</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-26 14:06:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/78/df/424bdc4a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>于途</span>
  </div>
  <div class="_2_QraFYR_0">1）首先 boltdb key 是版本号，put&#47;delete 操作时，都会基于当前版本号递增生成新的版本号，因此属于顺序写入，可以调整 boltdb 的 bucket.FillPercent 参数，使每个 page 填充更多数据，减少 page 的分裂次数并降低db空间。   此处的page 和 降低db空间不是很理解，劳烦老师解惑！<br><br>2）关于版本号(revision)的理解：假定全局版本号currentRevision=2，第一次执行 put hello world1，那么版本号为：hello:revision{2,0};第二次执行 put hello world2，此时版本号为：hello:revision{3,1};第3次执行 put hello world3，此时版本号为：hello:revision{4,2}。  不知理解是否有偏差？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 第一个问题，10 boltdb篇会帮助你深入解答第1个问题，稍等<br>第二个问题，{2,0}={major,sub} 2是etcd mvcc事务版本号全局递增，0是事务内子版本号随修改操作递增（比如一个txn事务中多个put&#47;delete操作，其会从0递增)，因此第二次执行put hello world2的时候版本号应是{3,0}, 07 mvcc会详细介绍，明天更新</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-28 16:25:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/e8/ac/7324d5ca.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>七里</span>
  </div>
  <div class="_2_QraFYR_0">幂等部分的“原子性事务”如何实现的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 就是key-value数据与consistent index在同一个boltdb事务中更新，boltdb后面会再单独介绍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-26 16:59:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/ed/b9/510fb3e2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>TS.乔</span>
  </div>
  <div class="_2_QraFYR_0">希望在后面多加一下具体设计思路，以及特性取舍的东西</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，谢谢你的建议，篇幅本身比较长就没继续扩展了，比如读写原理中，treeindex为什么用b-tree而不是其他数据结构，为什么使用boltdb而不是基于lsm树的leveldb等，后面答疑和其他讲我将适当和大家一起讨论</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-26 16:54:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/9d/84/171b2221.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jeffery</span>
  </div>
  <div class="_2_QraFYR_0">原理讲的透彻、为啥applied index超过了 5000，返回一个&quot;etcdserver: too many requests&quot;错误给 client。raft源码定义的最大值吗？谢谢老师</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢，这个限速不是raft模块做的，raft是个单独共识算法库，是etcd server使用raft的时候，基于raft告知的committed index，本身apply模块的applied index做的限速，默认写死了5000</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-26 12:58:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/rPaT2MzkXFmlmMTicGCRk5yVKqibPlloh66ibGJfoLQbgb6ficD3TkPmngR8UCEkrKZf5UbzvLlIglyYXBZibUINQ9Q/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_5a8405</span>
  </div>
  <div class="_2_QraFYR_0">从节点收到Propose请求后会写wal日志吗？那如果最终并没有一半的节点成功响应，那已经写入wal的从节点怎么处理呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞，阅读过程中有深入思考，你可以看看04节raft，其中思考题与你说的类似，05有参考答案</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-10 11:38:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/e1/da/d7f591a7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>励研冰</span>
  </div>
  <div class="_2_QraFYR_0">这篇仔细读了好几遍每次都有不一样的收获跟理解，被很多设计细节跟思路所折服，最后整理了下发现etcd的写流程居然跟mysql中的设计有异曲同工之妙，比如mysql中的redolog，binlog，changebuff，内存缓存，事物合并提交等…….，最后有一个不明白的点是为什么在boltdb提交事物的时候不是用key的mod_revision而是要重新生成新的版本号</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-04 20:20:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/f3/a7/15ee1f00.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>憨憨</span>
  </div>
  <div class="_2_QraFYR_0">讲的真好，干货十足</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢认可😊</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-04 08:57:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/5f/e5/54325854.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>范闲</span>
  </div>
  <div class="_2_QraFYR_0">会有影响。<br>1.如果读写请求都落到bucket buffer上，bucketbuffer需要做锁处理。<br>2.如果读写请求都落到boltdb上，db上的数据是从磁盘加载，同bucket buffer相比性能会下降一个数量级。<br>3.如果既落了bucket buffer，又落到了boltdb上。那么性能受到的影响介于1-2之间。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-25 09:42:15</div>
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
  <div class="_2_QraFYR_0">etcd 通过引入一个 consistent index 的字段，来存储系统当前已经执行过的日志条目索引，实现幂等性。<br>-----<br>consistent index 是一个全局的值吗，单调递增的?还是主要有raft日志应用到状态机，就会存储当前consistent index值吗，判断日志使用已经执行过，是需要进行key查找所有存储的consistent index值吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对，是一个全局单调递增的值，在boltdb中有一个key维护它，etcdserver应用已提交的raft日志条目到状态机时，会查询此日志条目的索引是否大于consistent index，如果大于则同key-value等数据在同boltdb事务中更新它，否则说明此日志条目已执行过。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-17 17:47:50</div>
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
  <div class="_2_QraFYR_0">1. consistent index具体是如何保存的，如何实现跟具体操作实现原子性提交的<br>2. readIndex跟consistent index儒者WAL index什么关系？ ReadIndex一定要处于commited吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-02 14:10:34</div>
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
  <div class="_2_QraFYR_0">文章有点长，但读起来层层递进，有很大收获，发现之前自己所了解的还真不全面，填补了之前的一些空白面，期待后面的精彩内容</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-26 07:42:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/86/39/d12aaabf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>kingstone</span>
  </div>
  <div class="_2_QraFYR_0">请问revision如果超出了上限，revision会如何接着生成？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 好问题，目前etcd没处理这种情况，它的类似是int64,最大值9223372036854775807，看起来是很难达到上限的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-26 01:59:10</div>
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
  <div class="_2_QraFYR_0">一篇都值回票价，醍醐灌顶，作者大佬太牛了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-11 14:00:37</div>
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
  <div class="_2_QraFYR_0">Apply模块针对写请求，是串行执行的还是并发执行的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 串行</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-17 11:30:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/6b/e2/f02e45df.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>恰同学少年。</span>
  </div>
  <div class="_2_QraFYR_0">感觉不太会。<br>1. client有负载均衡不会有很多请求都直接到主？<br>2. v3支持多路复用（这样到主链接数应该不会太多）<br>3. 线性读请求只会向主发获取ReadIndex的请求，还挺轻量的。<br>4. boltdb只有启动时会加载db文件并映射到mmap，内存够就不会有磁盘IO，所以影响也不大。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-20 19:22:57</div>
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
  <div class="_2_QraFYR_0">这里 db 配额（Quota）的限制哪个地方的大小的？ 为什么会有这样的限制呢？ 这个限制是  boltdb 全部数据 持久化后的文件大小吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: boltdb的db文件，你可以在etcd的数据目录snap目录下看到有个名为db的文件，实践篇我会介绍db文件过大又哪些问题，etcd定位就是个小型的关键元数据存储</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-26 23:29:31</div>
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
  <div class="_2_QraFYR_0">compact后没有defrag的话对性能有影响吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: compact是非常常规的操作，只要db文件不是特别大，大小保持稳定就不用defrag哈，后面会详细介绍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-25 18:26:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTIVvyFCLRcfoWfiaJt99K0wiabvicWtQaJdSseVA6QqWyxcvN5nd2TgZqiaUACc94bBvPHZTibnfnZfdtQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_7d539e</span>
  </div>
  <div class="_2_QraFYR_0">说到幂等性时，提到了一个原子操作。执行命令的请求更新成功了，同时更新 index ，两个操作作为一个原子操作。在 boltdb 环节里面提到异步批量提交操作，这个原子操作是在异步提交时同时完成对应日志条目的入库后一起完成的吗？请教下</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，一个boltdb事务中包含多个key-value操作，有用户的，也有etcd所维护的元数据，bucket name不一样而已。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-25 11:37:55</div>
  </div>
</div>
</div>
</li>
</ul>