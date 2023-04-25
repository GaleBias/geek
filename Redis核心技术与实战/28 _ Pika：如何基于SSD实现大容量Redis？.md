<audio title="28 _ Pika：如何基于SSD实现大容量Redis？" src="https://static001.geekbang.org/resource/audio/2e/7c/2eafb25d314553776cc36d5c4212787c.mp3" controls="controls"></audio> 
<p>你好，我是蒋德钧。</p><p>我们在应用Redis时，随着业务数据的增加（比如说电商业务中，随着用户规模和商品数量的增加），就需要Redis能保存更多的数据。你可能会想到使用Redis切片集群，把数据分散保存到多个实例上。但是这样做的话，会有一个问题，如果要保存的数据总量很大，但是每个实例保存的数据量较小的话，就会导致集群的实例规模增加，这会让集群的运维管理变得复杂，增加开销。</p><p>你可能又会说，我们可以通过增加Redis单实例的内存容量，形成大内存实例，每个实例可以保存更多的数据，这样一来，在保存相同的数据总量时，所需要的大内存实例的个数就会减少，就可以节省开销。</p><p>这是一个好主意，但这也并不是完美的方案：基于大内存的大容量实例在实例恢复、主从同步过程中会引起一系列潜在问题，例如恢复时间增长、主从切换开销大、缓冲区易溢出。</p><p>那怎么办呢？我推荐你使用固态硬盘（Solid State Drive，SSD）。它的成本很低（每GB的成本约是内存的十分之一），而且容量大，读写速度快，我们可以基于SSD来实现大容量的Redis实例。360公司DBA和基础架构组联合开发的Pika<a href="https://github.com/Qihoo360/pika">键值数据库</a>，正好实现了这一需求。</p><p>Pika在刚开始设计的时候，就有两个目标：一是，单实例可以保存大容量数据，同时避免了实例恢复和主从同步时的潜在问题；二是，和Redis数据类型保持兼容，可以支持使用Redis的应用平滑地迁移到Pika上。所以，如果你一直在使用Redis，并且想使用SSD来扩展单实例容量，Pika就是一个很好的选择。</p><!-- [[[read_end]]] --><p>这节课，我就和你聊聊Pika。在介绍Pika前，我先给你具体解释下基于大内存实现大容量Redis实例的潜在问题。只有知道了这些问题，我们才能选择更合适的方案。另外呢，我还会带你一步步分析下Pika是如何实现刚刚我们所说的两个设计目标，解决这些问题的。</p><h2>大内存Redis实例的潜在问题</h2><p>Redis使用内存保存数据，内存容量增加后，就会带来两方面的潜在问题，分别是，内存快照RDB生成和恢复效率低，以及主从节点全量同步时长增加、缓冲区易溢出。我来一一解释下，</p><p>我们先看内存快照RDB受到的影响。内存大小和内存快照RDB的关系是非常直接的：实例内存容量大，RDB文件也会相应增大，那么，RDB文件生成时的fork时长就会增加，这就会导致Redis实例阻塞。而且，RDB文件增大后，使用RDB进行恢复的时长也会增加，会导致Redis较长时间无法对外提供服务。</p><p>接下来我们再来看下主从同步受到的影响，</p><p>主从节点间的同步的第一步就是要做全量同步。全量同步是主节点生成RDB文件，并传给从节点，从节点再进行加载。试想一下，如果RDB文件很大，肯定会导致全量同步的时长增加，效率不高，而且还可能会导致复制缓冲区溢出。一旦缓冲区溢出了，主从节点间就会又开始全量同步，影响业务应用的正常使用。如果我们增加复制缓冲区的容量，这又会消耗宝贵的内存资源。</p><p>此外，如果主库发生了故障，进行主从切换后，其他从库都需要和新主库进行一次全量同步。如果RDB文件很大，也会导致主从切换的过程耗时增加，同样会影响业务的可用性。</p><p>那么，Pika是如何解决这两方面的问题呢？这就要提到Pika中的关键模块RocksDB、binlog机制和Nemo了，这些模块都是Pika架构中的重要组成部分。所以，接下来，我们就来先看下Pika的整体架构。</p><h2>Pika的整体架构</h2><p>Pika键值数据库的整体架构中包括了五部分，分别是网络框架、Pika线程模块、Nemo存储模块、RocksDB和binlog机制，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/a1/e7/a1421b8dbca6bb1ee9b6c1be7a929ae7.jpg?wh=1486*1469" alt=""></p><p>这五个部分分别实现了不同的功能，下面我一个个来介绍下。</p><p>首先，网络框架主要负责底层网络请求的接收和发送。Pika的网络框架是对操作系统底层的网络函数进行了封装。Pika在进行网络通信时，可以直接调用网络框架封装好的函数。</p><p>其次，Pika线程模块采用了多线程模型来具体处理客户端请求，包括一个请求分发线程（DispatchThread）、一组工作线程（WorkerThread）以及一个线程池（ThreadPool）。</p><p>请求分发线程专门监听网络端口，一旦接收到客户端的连接请求后，就和客户端建立连接，并把连接交由工作线程处理。工作线程负责接收客户端连接上发送的具体命令请求，并把命令请求封装成Task，再交给线程池中的线程，由这些线程进行实际的数据存取处理，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/46/06/4627f13848167cdaa3b30370d9b80a06.jpg?wh=3000*1676" alt=""></p><p>在实际应用Pika的时候，我们可以通过增加工作线程数和线程池中的线程数，来提升Pika的请求处理吞吐率，进而满足业务层对数据处理性能的需求。</p><p>Nemo模块很容易理解，它实现了Pika和Redis的数据类型兼容。这样一来，当我们把Redis服务迁移到Pika时，不用修改业务应用中操作Redis的代码，而且还可以继续应用运维Redis的经验，这使得Pika的学习成本就较低。Nemo模块对数据类型的具体转换机制是我们要重点关心的，下面我会具体介绍。</p><p>最后，我们再来看看RocksDB提供的基于SSD保存数据的功能。它使得Pika可以不用大容量的内存，就能保存更多数据，还避免了使用内存快照。而且，Pika使用binlog机制记录写命令，用于主从节点的命令同步，避免了刚刚所说的大内存实例在主从同步过程中的潜在问题。</p><p>接下来，我们就来具体了解下，Pika是如何使用RocksDB和binlog机制的。</p><h2>Pika如何基于SSD保存更多数据？</h2><p>为了把数据保存到SSD，Pika使用了业界广泛应用的持久化键值数据库<a href="https://rocksdb.org/">RocksDB</a>。RocksDB本身的实现机制较为复杂，你不需要全部弄明白，你只要记住RocksDB的基本数据读写机制，对于学习了解Pika来说，就已经足够了。下面我来解释下这个基本读写机制。</p><p>下面我结合一张图片，来给你具体介绍下RocksDB写入数据的基本流程。</p><p><img src="https://static001.geekbang.org/resource/image/95/1d/95d97d3cf0f1555b65b47fb256b7b81d.jpg?wh=3000*2139" alt=""></p><p>当Pika需要保存数据时，RocksDB会使用两小块内存空间（Memtable1和Memtable2）来交替缓存写入的数据。Memtable的大小可以设置，一个Memtable的大小一般为几MB或几十MB。当有数据要写入RocksDB时，RocksDB会先把数据写入到Memtable1。等到Memtable1写满后，RocksDB再把数据以文件的形式，快速写入底层的SSD。同时，RocksDB会使用Memtable2来代替Memtable1，缓存新写入的数据。等到Memtable1的数据都写入SSD了，RocksDB会在Memtable2写满后，再用Memtable1缓存新写入的数据。</p><p>这么一分析你就知道了，RocksDB会先用Memtable缓存数据，再将数据快速写入SSD，即使数据量再大，所有数据也都能保存到SSD中。而且，Memtable本身容量不大，即使RocksDB使用了两个Memtable，也不会占用过多的内存，这样一来，Pika在保存大容量数据时，也不用占据太大的内存空间了。</p><p>当Pika需要读取数据的时候，RocksDB会先在Memtable中查询是否有要读取的数据。这是因为，最新的数据都是先写入到Memtable中的。如果Memtable中没有要读取的数据，RocksDB会再查询保存在SSD上的数据文件，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/aa/3b/aa70655efbb767af499a83bd6521ee3b.jpg?wh=2479*971" alt=""></p><p>到这里，你就了解了，当使用了RocksDB保存数据后，Pika就可以把大量数据保存到大容量的SSD上了，实现了大容量实例。不过，我刚才向你介绍过，当使用大内存实例保存大量数据时，Redis会面临RDB生成和恢复的效率问题，以及主从同步时的效率和缓冲区溢出问题。那么，当Pika保存大量数据时，还会面临相同的问题吗？</p><p>其实不会了，我们来分析一下。</p><p>一方面，Pika基于RocksDB保存了数据文件，直接读取数据文件就能恢复，不需要再通过内存快照进行恢复了。而且，Pika从库在进行全量同步时，可以直接从主库拷贝数据文件，不需要使用内存快照，这样一来，Pika就避免了大内存快照生成效率低的问题。</p><p>另一方面，Pika使用了binlog机制实现增量命令同步，既节省了内存，还避免了缓冲区溢出的问题。binlog是保存在SSD上的文件，Pika接收到写命令后，在把数据写入Memtable时，也会把命令操作写到binlog文件中。和Redis类似，当全量同步结束后，从库会从binlog中把尚未同步的命令读取过来，这样就可以和主库的数据保持一致。当进行增量同步时，从库也是把自己已经复制的偏移量发给主库，主库把尚未同步的命令发给从库，来保持主从库的数据一致。</p><p>不过，和Redis使用缓冲区相比，使用binlog好处是非常明显的：binlog是保存在SSD上的文件，文件大小不像缓冲区，会受到内存容量的较多限制。而且，当binlog文件增大后，还可以通过轮替操作，生成新的binlog文件，再把旧的binlog文件独立保存。这样一来，即使Pika实例保存了大量的数据，在同步过程中也不会出现缓冲区溢出的问题了。</p><p>现在，我们先简单小结下。Pika使用RocksDB把大量数据保存到了SSD，同时避免了内存快照的生成和恢复问题。而且，Pika使用binlog机制进行主从同步，避免大内存时的影响，Pika的第一个设计目标就实现了。</p><p>接下来，我们再来看Pika是如何实现第二个设计目标的，也就是如何和Redis兼容。毕竟，如果不兼容的话，原来使用Redis的业务就无法平滑迁移到Pika上使用了，也就没办法利用Pika保存大容量数据的优势了。</p><h2>Pika如何实现Redis数据类型兼容？</h2><p>Pika的底层存储使用了RocksDB来保存数据，但是，RocksDB只提供了单值的键值对类型，RocksDB键值对中的值就是单个值，而Redis键值对中的值还可以是集合类型。</p><p>对于Redis的String类型来说，它本身就是单值的键值对，我们直接用RocksDB保存就行。但是，对于集合类型来说，我们就无法直接把集合保存为单值的键值对，而是需要进行转换操作。</p><p>为了保持和Redis的兼容性，Pika的Nemo模块就负责把Redis的集合类型转换成单值的键值对。简单来说，我们可以把Redis的集合类型分成两类：</p><ul>
<li>一类是List和Set类型，它们的集合中也只有单值；</li>
<li>另一类是Hash和Sorted Set类型，它们的集合中的元素是成对的，其中，Hash集合元素是field-value类型，而Sorted Set集合元素是member-score类型。</li>
</ul><p>Nemo模块通过转换操作，把这4种集合类型的元素表示为单值的键值对。具体怎么转换呢？下面我们来分别看下每种类型的转换。</p><p>首先我们来看List类型。在Pika中，List集合的key被嵌入到了单值键值对的键当中，用key字段表示；而List集合的元素值，则被嵌入到单值键值对的值当中，用value字段表示。因为List集合中的元素是有序的，所以，Nemo模块还在单值键值对的key后面增加了sequence字段，表示当前元素在List中的顺序，同时，还在value的前面增加了previous sequence和next sequence这两个字段，分别表示当前元素的前一个元素和后一个元素。</p><p>此外，在单值键值对的key前面，Nemo模块还增加了一个值“l”，表示当前数据是List类型，以及增加了一个1字节的size字段，表示List集合key的大小。在单值键值对的value后面，Nemo模块还增加了version和ttl字段，分别表示当前数据的版本号和剩余存活时间（用来支持过期key功能），如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/06/05/066465f1a28b6f14a42c1fc3a3f73105.jpg?wh=3000*1056" alt=""></p><p>我们再来看看Set集合。</p><p>Set集合的key和元素member值，都被嵌入到了Pika单值键值对的键当中，分别用key和member字段表示。同时，和List集合类似，单值键值对的key前面有值“s”，用来表示数据是Set类型，同时还有size字段，用来表示key的大小。Pika单值键值对的值只保存了数据的版本信息和剩余存活时间，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/aa/71/aa20c1456526dbf3f7d30f9d865f0f71.jpg?wh=2318*936" alt=""></p><p>对于Hash类型来说，Hash集合的key被嵌入到单值键值对的键当中，用key字段表示，而Hash集合元素的field也被嵌入到单值键值对的键当中，紧接着key字段，用field字段表示。Hash集合元素的value则是嵌入到单值键值对的值当中，并且也带有版本信息和剩余存活时间，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/63/b9/6378f7045393ae342632189a4ab601b9.jpg?wh=2813*851" alt=""></p><p>最后，对于Sorted Set类型来说，该类型是需要能够按照集合元素的score值排序的，而RocksDB只支持按照单值键值对的键来排序。所以，Nemo模块在转换数据时，就把Sorted Set集合key、元素的score和member值都嵌入到了单值键值对的键当中，此时，单值键值对中的值只保存了数据的版本信息和剩余存活时间，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/a0/a8/a0bc4d00a5d95e7fd2699945ff7a56a8.jpg?wh=3000*940" alt=""></p><p>采用了上面的转换方式之后，Pika不仅能兼容支持Redis的数据类型，而且还保留了这些数据类型的特征，例如List的元素保序、Sorted Set的元素按score排序。了解了Pika的转换机制后，你就会明白，如果你有业务应用计划从使用Redis切换到使用Pika，就不用担心面临因为操作接口不兼容而要修改业务应用的问题了。</p><p>经过刚刚的分析，我们可以知道，Pika能够基于SSD保存大容量数据，而且和Redis兼容，这是它的两个优势。接下来，我们再来看看，跟Redis相比，Pika的其他优势，以及潜在的不足。当在实际应用Pika时，Pika的不足之处是你需要特别注意的地方，这些可能都需要你进行系统配置或参数上的调优。</p><h2>Pika的其他优势与不足</h2><p>跟Redis相比，Pika最大的特点就是使用了SSD来保存数据，这个特点能带来的最直接好处就是，Pika单实例能保存更多的数据了，实现了实例数据扩容。</p><p>除此之外，Pika使用SSD来保存数据，还有额外的两个优势。</p><p>首先，<strong>实例重启快</strong>。Pika的数据在写入数据库时，是会保存到SSD上的。当Pika实例重启时，可以直接从SSD上的数据文件中读取数据，不需要像Redis一样，从RDB文件全部重新加载数据或是从AOF文件中全部回放操作，这极大地提高了Pika实例的重启速度，可以快速处理业务应用请求。</p><p>另外，主从库重新执行全量同步的风险低。Pika通过binlog机制实现写命令的增量同步，不再受内存缓冲区大小的限制，所以，即使在数据量很大导致主从库同步耗时很长的情况下，Pika也不用担心缓冲区溢出而触发的主从库重新全量同步。</p><p>但是，就像我在前面的课程中和你说的，“硬币都是有正反两面的”，Pika也有自身的一些不足。</p><p>虽然它保持了Redis操作接口，也能实现数据库扩容，但是，当把数据保存到SSD上后，会降低数据的访问性能。这是因为，数据操作毕竟不能在内存中直接执行了，而是要在底层的SSD中进行存取，这肯定会影响，Pika的性能。而且，我们还需要把binlog机制记录的写命令同步到SSD上，这会降低Pika的写性能。</p><p>不过，Pika的多线程模型，可以同时使用多个线程进行数据读写，这在一定程度上弥补了从SSD存取数据造成的性能损失。当然，你也可以使用高配的SSD来提升访问性能，进而减少读写SSD对Pika性能的影响。</p><p>为了帮助你更直观地了解Pika的性能情况，我再给你提供一张表，这是Pika<a href="https://github.com/Qihoo360/pika/wiki/3.2.x-Performance">官网</a>上提供的测试数据。</p><p><img src="https://static001.geekbang.org/resource/image/6f/c5/6fed4a269a79325efd6fa4fb17fc44c5.jpg?wh=1456*716" alt=""></p><p>这些数据是在Pika 3.2版本中，String和Hash类型在多线程情况下的基本操作性能结果。从表中可以看到，在不写binlog时，Pika的SET/GET、HSET/HGET的性能都能达到200K OPS以上，而一旦增加了写binlog操作，SET和HSET操作性能大约下降了41%，只有约120K OPS。</p><p>所以，我们在使用Pika时，需要在单实例扩容的必要性和可能的性能损失间做个权衡。如果保存大容量数据是我们的首要需求，那么，Pika是一个不错的解决方案。</p><h2>小结</h2><p>这节课，我们学习了基于SSD给Redis单实例进行扩容的技术方案Pika。跟Redis相比，Pika的好处非常明显：既支持Redis操作接口，又能支持保存大容量的数据。如果你原来就在应用Redis，现在想进行扩容，那么，Pika无疑是一个很好的选择，无论是代码迁移还是运维管理，Pika基本不需要额外的工作量。</p><p>不过，Pika毕竟是把数据保存到了SSD上，数据访问要读写SSD，所以，读写性能要弱于Redis。针对这一点，我给你提供两个降低读写SSD对Pika的性能影响的小建议：</p><ol>
<li>利用Pika的多线程模型，增加线程数量，提升Pika的并发请求处理能力；</li>
<li>为Pika配置高配的SSD，提升SSD自身的访问性能。</li>
</ol><p>最后，我想再给你一个小提示。Pika本身提供了很多工具，可以帮助我们把Redis数据迁移到Pika，或者是把Redis请求转发给Pika。比如说，我们使用aof_to_pika命令，并且指定Redis的AOF文件以及Pika的连接信息，就可以把Redis数据迁移到Pika中了，如下所示：</p><pre><code>aof_to_pika -i [Redis AOF文件] -h [Pika IP] -p [Pika port] -a [认证信息]
</code></pre><p>关于这些工具的信息，你都可以直接在Pika的<a href="https://github.com/Qihoo360/pika/wiki">GitHub</a>上找到。而且，Pika本身也还在迭代开发中，我也建议你多去看看GitHub，进一步地了解它。这样，你就可以获得Pika的最新进展，也能更好地把它应用到你的业务实践中。</p><h2>每课一问</h2><p>按照惯例，我给你提个小问题。这节课，我向你介绍的是使用SSD作为内存容量的扩展，增加Redis实例的数据保存量，我想请你来聊一聊，我们可以使用机械硬盘来作为实例容量扩展吗，有什么好处或不足吗？</p><p>欢迎在留言区写下你的思考和答案，我们一起交流讨论。如果你觉得今天的内容对你有所帮助，也欢迎你分享给你的朋友或同事。我们下节课见。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/90/8a/288f9f94.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Kaito</span>
  </div>
  <div class="_2_QraFYR_0">是否可以使用机械硬盘作为Redis的内存容量的扩展？<br><br>我觉得也是可以的。机械硬盘相较于固态硬盘的优点是，成本更低、容量更大、寿命更长。<br><br>1、成本：机械硬盘是电磁存储，固态硬盘是半导体电容颗粒组成，相同容量下机械硬盘成本是固态硬盘的1&#47;3。<br>2、容量：相同成本下，机械硬盘可使用的容量更大。<br>3、寿命：固态硬盘的电容颗粒擦写次数有限，超过一定次数后会不可用。相同ops情况下，机械硬盘的寿命要比固态硬盘的寿命更长。<br><br>但机械硬盘相较于固态硬盘的缺点也很明显，就是速度慢。<br><br>机械硬盘在读写数据时，需要通过转动磁盘和磁头等机械方式完成，而固态硬盘是直接通过电信号保存和控制数据的读写，速度非常快。<br><br>如果对于访问延迟要求不高，对容量和成本比较关注的场景，可以把Pika部署在机械硬盘上使用。<br><br>另外，关于Pika的使用场景，它并不能代替Redis，而是作为Redis的补充，在需要大容量存储（50G数据量以上）、访问延迟要求不苛刻的业务场景下使用。在使用之前，最好是根据自己的业务情况，先做好调研和性能测试，评估后决定是否使用。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-21 01:40:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c4/92/338b5609.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Roy Liang</span>
  </div>
  <div class="_2_QraFYR_0">SSD使用寿命和擦写次数相关，我们有的业务数据量和访问量特别大，使用SSD一年就得换了，这样实际成本降不下来。所以，请问有没有可能开发一套混合存储系统，热数据存储而不是缓存在Redis，冷数据存储在Pika，Redis中的热数据淘汰时，自动写入Pika，Pika的冷数据加载时，自动写入Redis，这样业务代码几乎就不用做数据一致性维护相关的内容。不知道这个系统是否存在可行性？其中的技术风险可能在哪里？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个方案是可行的，相当于冷热分层存储了，而且Pika和Redis的数据模型兼容，所以数据迁移起来会比较方便直接。<br><br>我想到的关键技术问题：1. 数据冷热的区分方法，这有点类似于缓存算法，把经常访问的数据要能够筛选出来；2. 数据的迁移开销，数据迁移时需要避免对Redis前端读写请求的阻塞，开发这个系统时可能要考虑使用额外线程来迁移Redis数据；3. 数据迁移时数据读写正确性的保证，例如数据正在迁移，是否还能正常读写。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-26 14:44:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/ec/13/49e98289.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>neohope</span>
  </div>
  <div class="_2_QraFYR_0">用机械硬盘阵列来做缓存，其实没有必要，速度太慢了：<br>1、用redis来做缓存，就是因为mysql慢，所以加了一层<br>2、redis比mysql快的最根本原因，不是redis技术强悍，而是内存比磁盘快<br>3、mysql本身就是把热数据放到内存里，冷数据存到磁盘阵列上，并且做了很多数据查找的优化<br>4、pika利用的是SSD比磁盘快，比内存便宜，才找到了自己的生存空间，其根本原因也不是RocksDB技术先进；<br>5、Pika的用磁盘做缓存，从磁盘上查找记录，比mysql也快不了太多，解决不了什么问题，而且还引入了新的故障点，没有太多价值<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-19 16:51:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/a0/c4/4ab49f4e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>孔祥鑫</span>
  </div>
  <div class="_2_QraFYR_0">有个地方没搞明白，读ssd上的数据文件，和redis读rdb文件相比，两者都是文件，为什么会比redis快呢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-16 08:19:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_9a0c9f</span>
  </div>
  <div class="_2_QraFYR_0">pika从性能上比当然不然redis,但是它你补了redis几个不足，那么pika在真是项目中都应用在什么场景呢？，与ssd的mysql比优势在哪里？除了o(1)的操作。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-21 23:00:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/49/a5/e4c1c2d4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小文同学</span>
  </div>
  <div class="_2_QraFYR_0">机械硬盘和 SSD 在性能上最大的区别是因为机械硬盘的随机访问涉及到机器臂的移动和寻址，所以非常慢。但我在了解 Pika 对文件的使用上，其实有考虑过这方面的问题，通过 A、B两块内存交换使用，以一块内存为单位进行磁盘写，在这个情况下。我认为机械硬盘的写性能并不一定比 SSD 差很多。<br><br>在读数据的情况下，假如没有热点数据，和存在大量的随机读，则机械硬盘的读性能会显得很差，反之，由于有一大块的内存做缓冲，读性能或许并不会太差，这取决于业务场景。<br><br>剩余机械硬盘的某些优势还是比较明显，Kaito 班长的留言已经非常明白，顾不复述。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-12 00:11:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/cf/81/96f656ef.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>杨逸林</span>
  </div>
  <div class="_2_QraFYR_0">Redis 服务器有没有必要上 ECC 内存？<br>还有如果那个  Memtable 有没有极端情况，Memtable1 还在写入 SSD，Memtable2 已经满了，这怎么办？虽然写入 SSD 很快，我是说如果。<br><br>binlog 机制在 MySQL 那已经玩烂了。。。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: ECC内存是需要的，Redis数据都在内存中，如果内存出错，会有影响的。<br><br>关于Memtable的问题是个好问题。在你说的情况下，Memtable1还在写SSD，Memtable2已经满了，此时相当于有两个Memtable要刷盘，RocksDB会对前端的写进行限速。<br><br>RocksDB中有个配置项是max_write_buffer_number，如果需要往盘上刷的memtable大于这个配置项，就会进行写限速了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-25 00:50:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/24/05/a6/098dfae1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Q, Q</span>
  </div>
  <div class="_2_QraFYR_0">pika实际是使用ssd提高性能，那还不如把redis层去掉，mysql直接上ssd岂不是更加方便？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-01 15:50:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/c1/c6/1456274a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>麦呆小石头</span>
  </div>
  <div class="_2_QraFYR_0">pika 的存储机制，集合类型的性能会低很多啊，hash 和跳表的特性都没了吧</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-02 10:48:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/71/3d/da8dc880.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>游弋云端</span>
  </div>
  <div class="_2_QraFYR_0">可以考虑内存、SSD、HDD做分级存储，当前这对系统要求就更高了，需要识别数据的冷温热，再做不同介质间的动态迁移，甚至可以做一些访问预测来做预加载和调级。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 分级存储时，数据的冷热度识别是一个主要要考虑的问题。通常可以用缓存算法策略来筛选冷热数据，目前也有些尝试是用机器学习的方法来预测数据冷热度，进而再放到不同的介质中。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-21 16:10:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/b3/49/79024ed2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Tr丶Zoey</span>
  </div>
  <div class="_2_QraFYR_0">有没有人用ssdb的啊？😂<br>老大让我们用这个代替redis，说没问题，结果用下来坑真的多。。。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-02 20:57:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/23/f8/24fcccea.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>💢 星星💢</span>
  </div>
  <div class="_2_QraFYR_0">Pika 的多线程模型，可以同时使用多个线程进行数据读写，这在一定程度上弥补了从 SSD 存取数据造成的性能损失，多线程读写，会涉及到数据竞争吧，pika这一块是否有优雅的处理呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-12 10:27:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/29/cd/58/06a8ce36.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Jackson</span>
  </div>
  <div class="_2_QraFYR_0">这里如果用持久化内存的话读写性能如何？redis6.0在多线程和这个优劣是啥</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-09 08:54:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/WxLKJlXCibwqO92vB8XTicLQiahrhuUEqP7yT9dearZxLzbia7oMdsLmon5J4LJyTfIWchHY3bKfibm1lS1aZarZs4Q/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jie</span>
  </div>
  <div class="_2_QraFYR_0">mysql上ssd 比rocksdb 上 ssd 收益更大吧 毕竟一个随机写,一个顺序写.</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-08 00:11:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/74/aa/178a6797.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>阿昕</span>
  </div>
  <div class="_2_QraFYR_0">好处：便宜，成本低；不足：速度慢；</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-14 17:34:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/b2/e0/d856f5a4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>鱼</span>
  </div>
  <div class="_2_QraFYR_0">由于是文件连续读写，Pika最大的问题并不是数据写入的时候吧。而是读取key的时候要去RocketsDB的SSTable里面挨个读文件。文章没有提及。相比之下，写入的性能降低和读取的性能降低比较不值一提啊。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-07 09:33:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/cc/b9/98070739.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>黄泓哲</span>
  </div>
  <div class="_2_QraFYR_0">这让我想起来前几年的混合硬盘，机械+NAND</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-22 11:05:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_5b378d</span>
  </div>
  <div class="_2_QraFYR_0">有一点不理解:RDB 文件生成时的 fork 时长就会增加<br>不是用另外一个进程去生成 RDB 文件嘛,咋会阻塞 redis 实例呢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-18 16:55:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/2a/6e/fb980b6b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>沈寅</span>
  </div>
  <div class="_2_QraFYR_0">redis操作都是原子的，pika内部使用多线程，还能保证原子性吗？ </div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-20 11:08:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/33/74/d9d143fa.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>silentyears</span>
  </div>
  <div class="_2_QraFYR_0">老师，pika这样存储集合类型的方式不太明白，拿Hash来说，field存储在键中，value存储在值中，怎么快速定位呢？难道是hash后计算出来位置m，在键中“数组”位置m上找field，同样在值中“数组”位置m上找value？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-13 11:59:49</div>
  </div>
</div>
</div>
</li>
</ul>