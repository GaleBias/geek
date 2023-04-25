<audio title="加餐（六）_  Redis的使用规范小建议" src="https://static001.geekbang.org/resource/audio/bf/a4/bff585ayy164fd5c3a454f1bfbe32ba4.mp3" controls="controls"></audio> 
<p>你好，我是蒋德钧。</p><p>今天的加餐，我们来聊一个轻松点儿的话题，我来给你介绍一下Redis的使用规范，包括键值对使用、业务数据保存和命令使用规范。</p><p>毕竟，高性能和节省内存，是我们的两个目标，只有规范地使用Redis，才能真正实现这两个目标。如果说之前的内容教会了你怎么用，那么今天的内容，就是帮助你用好Redis，尽量不出错。</p><p>好了，话不多说，我们来看下键值对的使用规范。</p><h2>键值对使用规范</h2><p>关于键值对的使用规范，我主要想和你说两个方面：</p><ol>
<li>key的命名规范，只有命名规范，才能提供可读性强、可维护性好的key，方便日常管理；</li>
<li>value的设计规范，包括避免bigkey、选择高效序列化方法和压缩方法、使用整数对象共享池、数据类型选择。</li>
</ol><h3>规范一：key的命名规范</h3><p>一个Redis实例默认可以支持16个数据库，我们可以把不同的业务数据分散保存到不同的数据库中。</p><p>但是，在使用不同数据库时，客户端需要使用SELECT命令进行数据库切换，相当于增加了一个额外的操作。</p><p>其实，我们可以通过合理命名key，减少这个操作。具体的做法是，把业务名作为前缀，然后用冒号分隔，再加上具体的业务数据名。这样一来，我们可以通过key的前缀区分不同的业务数据，就不用在多个数据库间来回切换了。</p><!-- [[[read_end]]] --><p>我给你举个简单的小例子，看看具体怎么命名key。</p><p>比如说，如果我们要统计网页的独立访客量，就可以用下面的代码设置key，这就表示，这个数据对应的业务是统计unique visitor（独立访客量），而且对应的页面编号是1024。</p><pre><code>uv:page:1024
</code></pre><p>这里有一个地方需要注意一下。key本身是字符串，底层的数据结构是SDS。SDS结构中会包含字符串长度、分配空间大小等元数据信息。从Redis 3.2版本开始，<strong>当key字符串的长度增加时，SDS中的元数据也会占用更多内存空间</strong>。</p><p>所以，<strong>我们在设置key的名称时，要注意控制key的长度</strong>。否则，如果key很长的话，就会消耗较多内存空间，而且，SDS元数据也会额外消耗一定的内存空间。</p><p>SDS结构中的字符串长度和元数据大小的对应关系如下表所示：</p><p><img src="https://static001.geekbang.org/resource/image/5a/0c/5afd9d57af8edd1ae69e62b1c998050c.jpg?wh=1966*784" alt=""></p><p>为了减少key占用的内存空间，我给你一个小建议：对于业务名或业务数据名，可以使用相应的英文单词的首字母表示，（比如user用u表示，message用m），或者是用缩写表示（例如unique visitor使用uv）。</p><h3>规范二：避免使用bigkey</h3><p>Redis是使用单线程读写数据，bigkey的读写操作会阻塞线程，降低Redis的处理效率。所以，在应用Redis时，关于value的设计规范，非常重要的一点就是避免bigkey。</p><p>bigkey通常有两种情况。</p><ul>
<li>情况一：键值对的值大小本身就很大，例如value为1MB的String类型数据。为了避免String类型的bigkey，在业务层，我们要尽量把String类型的数据大小控制在10KB以下。</li>
<li>情况二：键值对的值是集合类型，集合元素个数非常多，例如包含100万个元素的Hash集合类型数据。为了避免集合类型的bigkey，我给你的设计规范建议是，<strong>尽量把集合类型的元素个数控制在1万以下</strong>。</li>
</ul><p>当然，这些建议只是为了尽量避免bigkey，如果业务层的String类型数据确实很大，我们还可以通过数据压缩来减小数据大小；如果集合类型的元素的确很多，我们可以将一个大集合拆分成多个小集合来保存。</p><p>这里，还有个地方需要注意下，Redis的4种集合类型List、Hash、Set和Sorted Set，在集合元素个数小于一定的阈值时，会使用内存紧凑型的底层数据结构进行保存，从而节省内存。例如，假设Hash集合的hash-max-ziplist-entries配置项是1000，如果Hash集合元素个数不超过1000，就会使用ziplist保存数据。</p><p>紧凑型数据结构虽然可以节省内存，但是会在一定程度上导致数据的读写性能下降。所以，如果业务应用更加需要保持高性能访问，而不是节省内存的话，在不会导致bigkey的前提下，你就不用刻意控制集合元素个数了。</p><h3>规范三：使用高效序列化方法和压缩方法</h3><p>为了节省内存，除了采用紧凑型数据结构以外，我们还可以遵循两个使用规范，分别是使用高效的序列化方法和压缩方法，这样可以减少value的大小。</p><p>Redis中的字符串都是使用二进制安全的字节数组来保存的，所以，我们可以把业务数据序列化成二进制数据写入到Redis中。</p><p>但是，<strong>不同的序列化方法，在序列化速度和数据序列化后的占用内存空间这两个方面，效果是不一样的</strong>。比如说，protostuff和kryo这两种序列化方法，就要比Java内置的序列化方法（java-build-in-serializer）效率更高。</p><p>此外，业务应用有时会使用字符串形式的XML和JSON格式保存数据。</p><p>这样做的好处是，这两种格式的可读性好，便于调试，不同的开发语言都支持这两种格式的解析。</p><p>缺点在于，XML和JSON格式的数据占用的内存空间比较大。为了避免数据占用过大的内存空间，我建议使用压缩工具（例如snappy或gzip），把数据压缩后再写入Redis，这样就可以节省内存空间了。</p><h3>规范四：使用整数对象共享池</h3><p>整数是常用的数据类型，Redis内部维护了0到9999这1万个整数对象，并把这些整数作为一个共享池使用。</p><p>换句话说，如果一个键值对中有0到9999范围的整数，Redis就不会为这个键值对专门创建整数对象了，而是会复用共享池中的整数对象。</p><p>这样一来，即使大量键值对保存了0到9999范围内的整数，在Redis实例中，其实只保存了一份整数对象，可以节省内存空间。</p><p>基于这个特点，我建议你，在满足业务数据需求的前提下，能用整数时就尽量用整数，这样可以节省实例内存。</p><p>那什么时候不能用整数对象共享池呢？主要有两种情况。</p><p>第一种情况是，<strong>如果Redis中设置了maxmemory，而且启用了LRU策略（allkeys-lru或volatile-lru策略），那么，整数对象共享池就无法使用了</strong>。这是因为，LRU策略需要统计每个键值对的使用时间，如果不同的键值对都共享使用一个整数对象，LRU策略就无法进行统计了。</p><p>第二种情况是，如果集合类型数据采用ziplist编码，而集合元素是整数，这个时候，也不能使用共享池。因为ziplist使用了紧凑型内存结构，判断整数对象的共享情况效率低。</p><p>好了，到这里，我们了解了和键值对使用相关的四种规范，遵循这四种规范，最直接的好处就是可以节省内存空间。接下来，我们再来了解下，在实际保存数据时，该遵循哪些规范。</p><h2>数据保存规范</h2><h3>规范一：使用Redis保存热数据</h3><p>为了提供高性能访问，Redis是把所有数据保存到内存中的。</p><p>虽然Redis支持使用RDB快照和AOF日志持久化保存数据，但是，这两个机制都是用来提供数据可靠性保证的，并不是用来扩充数据容量的。而且，内存成本本身就比较高，如果把业务数据都保存在Redis中，会带来较大的内存成本压力。</p><p>所以，一般来说，在实际应用Redis时，我们会更多地把它作为缓存保存热数据，这样既可以充分利用Redis的高性能特性，还可以把宝贵的内存资源用在服务热数据上，就是俗话说的“好钢用在刀刃上”。</p><h3>规范二：不同的业务数据分实例存储</h3><p>虽然我们可以使用key的前缀把不同业务的数据区分开，但是，如果所有业务的数据量都很大，而且访问特征也不一样，我们把这些数据保存在同一个实例上时，这些数据的操作就会相互干扰。</p><p>你可以想象这样一个场景：假如数据采集业务使用Redis保存数据时，以写操作为主，而用户统计业务使用Redis时，是以读查询为主，如果这两个业务数据混在一起保存，读写操作相互干扰，肯定会导致业务响应变慢。</p><p>那么，<strong>我建议你把不同的业务数据放到不同的 Redis 实例中</strong>。这样一来，既可以避免单实例的内存使用量过大，也可以避免不同业务的操作相互干扰。</p><h3>规范三：在数据保存时，要设置过期时间</h3><p>对于Redis来说，内存是非常宝贵的资源，而且，Redis通常用于保存热数据。热数据一般都有使用的时效性。</p><p>所以，在数据保存时，我建议你根据业务使用数据的时长，设置数据的过期时间。不然的话，写入Redis的数据会一直占用内存，如果数据持续增多，就可能达到机器的内存上限，造成内存溢出，导致服务崩溃。</p><h3>规范四：控制Redis实例的容量</h3><p>Redis单实例的内存大小都不要太大，根据我自己的经验值，建议你设置在 2~6GB 。这样一来，无论是RDB快照，还是主从集群进行数据同步，都能很快完成，不会阻塞正常请求的处理。</p><h2>命令使用规范</h2><p>最后，我们再来看下在使用Redis命令时要遵守什么规范。</p><h3>规范一：线上禁用部分命令</h3><p>Redis 是单线程处理请求操作，如果我们执行一些涉及大量操作、耗时长的命令，就会严重阻塞主线程，导致其它请求无法得到正常处理，这类命令主要有3种。</p><ul>
<li><strong>KEYS</strong>，按照键值对的key内容进行匹配，返回符合匹配条件的键值对，该命令需要对Redis的全局哈希表进行全表扫描，严重阻塞Redis主线程；</li>
<li><strong>FLUSHALL</strong>，删除Redis实例上的所有数据，如果数据量很大，会严重阻塞Redis主线程；</li>
<li><strong>FLUSHDB</strong>，删除当前数据库中的数据，如果数据量很大，同样会阻塞Redis主线程。</li>
</ul><p>所以，我们在线上应用Redis时，就需要禁用这些命令。<strong>具体的做法是，管理员用rename-command命令在配置文件中对这些命令进行重命名，让客户端无法使用这些命令</strong>。</p><p>当然，你还可以使用其它命令替代这3个命令。</p><ul>
<li>对于KEYS命令来说，你可以用SCAN命令代替KEYS命令，分批返回符合条件的键值对，避免造成主线程阻塞；</li>
<li>对于FLUSHALL、FLUSHDB命令来说，你可以加上ASYNC选项，让这两个命令使用后台线程异步删除数据，可以避免阻塞主线程。</li>
</ul><h3>规范二：慎用MONITOR命令</h3><p>Redis的MONITOR命令在执行后，会持续输出监测到的各个命令操作，所以，我们通常会用MONITOR命令返回的结果，检查命令的执行情况。</p><p>但是，MONITOR命令会把监控到的内容持续写入输出缓冲区。如果线上命令的操作很多，输出缓冲区很快就会溢出了，这就会对Redis性能造成影响，甚至引起服务崩溃。</p><p>所以，除非十分需要监测某些命令的执行（例如，Redis性能突然变慢，我们想查看下客户端执行了哪些命令），你可以偶尔在短时间内使用下MONITOR命令，否则，我建议你不要使用MONITOR命令。</p><h3>规范三：慎用全量操作命令</h3><p>对于集合类型的数据来说，如果想要获得集合中的所有元素，一般不建议使用全量操作的命令（例如Hash类型的HGETALL、Set类型的SMEMBERS）。这些操作会对Hash和Set类型的底层数据结构进行全量扫描，如果集合类型数据较多的话，就会阻塞Redis主线程。</p><p>如果想要获得集合类型的全量数据，我给你三个小建议。</p><ul>
<li>第一个建议是，你可以使用SSCAN、HSCAN命令分批返回集合中的数据，减少对主线程的阻塞。</li>
<li>第二个建议是，你可以化整为零，把一个大的Hash集合拆分成多个小的Hash集合。这个操作对应到业务层，就是对业务数据进行拆分，按照时间、地域、用户ID等属性把一个大集合的业务数据拆分成多个小集合数据。例如，当你统计用户的访问情况时，就可以按照天的粒度，把每天的数据作为一个Hash集合。</li>
<li>最后一个建议是，如果集合类型保存的是业务数据的多个属性，而每次查询时，也需要返回这些属性，那么，你可以使用String类型，将这些属性序列化后保存，每次直接返回String数据就行，不用再对集合类型做全量扫描了。</li>
</ul><h2>小结</h2><p>这节课，我围绕Redis应用时的高性能访问和节省内存空间这两个目标，分别在键值对使用、命令使用和数据保存三方面向你介绍了11个规范。</p><p>我按照强制、推荐、建议这三个类别，把这些规范分了下类，如下表所示：</p><p><img src="https://static001.geekbang.org/resource/image/f2/fd/f2513c69d7757830e7f3e3c831fcdcfd.jpg?wh=2612*1819" alt=""></p><p>我来解释一下这3个类别的规范。</p><ul>
<li>强制类别的规范：这表示，如果不按照规范内容来执行，就会给Redis的应用带来极大的负面影响，例如性能受损。</li>
<li>推荐类别的规范：这个规范的内容能有效提升性能、节省内存空间，或者是增加开发和运维的便捷性，你可以直接应用到实践中。</li>
<li>建议类别的规范：这类规范内容和实际业务应用相关，我只是从我的经历或经验给你一个建议，你需要结合自己的业务场景参考使用。</li>
</ul><p>我再多说一句，你一定要熟练掌握这些使用规范，并且真正地把它们应用到你的Redis使用场景中，提高Redis的使用效率。</p><h2>每课一问</h2><p>按照惯例，我给你提个小问题，你在日常应用Redis时，有遵循过什么好的使用规范吗？</p><p>欢迎在留言区分享一下你常用的使用规范，我们一起交流讨论。如果你觉得今天的内容对你有所帮助，也欢迎你分享给你的朋友或同事。我们下节课见。</p>
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
  <div class="_2_QraFYR_0">我总结的 Redis 使用规范分为两大方面，主要包括业务层面和运维层面。<br><br>业务层面主要面向的业务开发人员：<br><br>1、key 的长度尽量短，节省内存空间<br>2、避免 bigkey，防止阻塞主线程<br>3、4.0+版本建议开启 lazy-free<br>4、把 Redis 当作缓存使用，设置过期时间<br>5、不使用复杂度过高的命令，例如SORT、SINTER、SINTERSTORE、ZUNIONSTORE、ZINTERSTORE<br>6、查询数据尽量不一次性查询全量，写入大量数据建议分多批写入<br>7、批量操作建议 MGET&#47;MSET 替代 GET&#47;SET，HMGET&#47;HMSET 替代 HGET&#47;HSET<br>8、禁止使用 KEYS&#47;FLUSHALL&#47;FLUSHDB 命令<br>9、避免集中过期 key<br>10、根据业务场景选择合适的淘汰策略<br>11、使用连接池操作 Redis，并设置合理的参数，避免短连接<br>12、只使用 db0，减少 SELECT 命令的消耗<br>13、读请求量很大时，建议读写分离，写请求量很大，建议使用切片集群<br><br>运维层面主要面向的是 DBA 运维人员：<br><br>1、按业务线部署实例，避免多个业务线混合部署，出问题影响其他业务<br>2、保证机器有足够的 CPU、内存、带宽、磁盘资源<br>3、建议部署主从集群，并分布在不同机器上，slave 设置为 readonly<br>4、主从节点所部署的机器各自独立，尽量避免交叉部署，对从节点做维护时，不会影响到主节点<br>5、推荐部署哨兵集群实现故障自动切换，哨兵节点分布在不同机器上<br>6、提前做好容量规划，防止主从全量同步时，实例使用内存突增导致内存不足<br>7、做好机器 CPU、内存、带宽、磁盘监控，资源不足时及时报警，任意资源不足都会影响 Redis 性能<br>8、实例设置最大连接数，防止过多客户端连接导致实例负载过高，影响性能<br>9、单个实例内存建议控制在 10G 以下，大实例在主从全量同步、备份时有阻塞风险<br>10、设置合理的 slowlog 阈值，并对其进行监控，slowlog 过多需及时报警<br>11、设置合理的 repl-backlog，降低主从全量同步的概率<br>12、设置合理的 slave client-output-buffer-limit，避免主从复制中断情况发生<br>13、推荐在从节点上备份，不影响主节点性能<br>14、不开启 AOF 或开启 AOF 配置为每秒刷盘，避免磁盘 IO 拖慢 Redis 性能<br>15、调整 maxmemory 时，注意主从节点的调整顺序，顺序错误会导致主从数据不一致<br>16、对实例部署监控，采集 INFO 信息时采用长连接，避免频繁的短连接<br>17、做好实例运行时监控，重点关注 expired_keys、evicted_keys、latest_fork_usec，这些指标短时突增可能会有阻塞风险<br>18、扫描线上实例时，记得设置休眠时间，避免过高 OPS 产生性能抖动</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-18 19:39:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/41/5e/9d2953a3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>zhou</span>
  </div>
  <div class="_2_QraFYR_0">还有一个规范：不要把 Redis 当数据库使用</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-18 08:32:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/8d/55/1345dff3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>独自等待</span>
  </div>
  <div class="_2_QraFYR_0">【在集合元素个数小于一定的阈值时，会使用内存紧凑型的底层数据结构进行保存，从而节省内存。例如，假设 Hash 集合的 hash-max-ziplist-entries 配置项是 1000，如果 Hash 集合元素个数不超过 1000，就会使用 ziplist 保存数据。紧凑型数据结构虽然可以节省内存，但是会在一定程度上导致数据的读写性能下降】<br>请问这个怎么理解，内存连续读写性能不应该更好吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-18 10:35:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/92/6d/becd841a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>escray</span>
  </div>
  <div class="_2_QraFYR_0">好像没怎么看到过 bigkey 的标准定义，之前误以为真的是&quot;key&quot;太大，后来才发现是 value 太大。<br><br>键值对使用规范<br><br>- key 命名规范：业务名作前缀，冒号分隔，加业务数据名，尽量避免数据库切换。key 的长度最好不超过 31（SDS结构元数据大小 1 字节）<br><br>- 避免使用 bigkey：String 类型的数据控制在 10KB 一下，集合类型的元素个数控制在 1 万以内。<br><br>- 使用高效的序列化和压缩方法：protostuff 和 kryo 优于 Java 内置的 java-build-in-serializer；如果使用 XML 或者 JSON 可以考虑压缩，snappy 或 gzip<br><br>- 使用整数对象共享池：在满足业务数据需求的前提下，尽量用整数<br><br>数据保存规范<br><br>- 使用 Redis 保存热数据<br>- 不同的业务数据分实例存储<br>- 保存数据时，设置过期时间<br>- 控制 Redis 实例的容量，2~6 GB<br><br>命令使用规范<br><br>- 线上禁用部分命令：KEYS、FLUSHALL、FLUSHDB<br>- 慎用 MONITOR<br>- 慎用全量操作命令<br><br>课代表 @Kaito 大神的 Redis 使用规范值得收藏。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-22 21:39:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/41/6c/687c5dfb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>叶子。</span>
  </div>
  <div class="_2_QraFYR_0">从 Redis 3.2 版本开始，当 key 字符串的长度增加时，SDS 中的元数据也会占用更多内存空间<br>请问这句话怎么理解，之前讲SDS的时候说的好像是 长度和实际分配长度分别占用4B？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-18 14:22:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/22/bd/9b/366bb87b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>飞龙</span>
  </div>
  <div class="_2_QraFYR_0">实际业务中遇到过一个困扰，就是项目起初只是记录每个用户的一个标识，用了HASH，但是随便项目的飞速发展，用户数量到了5000W，导致 这个HASH里的元素上千万级别了，单这一个就占了1个多G的内存，后面也不太好根据业务分散</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-24 11:21:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/0d/5a/e60f4125.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>camel</span>
  </div>
  <div class="_2_QraFYR_0">请问老师，如果一个hash非常大，比如超过10w个记录，避免了hgetall也会导致阻塞很久吗？ 也就是说hget操作的性能与hash.size有没有关系，是什么复杂度的关系？（算法层面上能理解的是应该没关系，因为hash的查询操作是O(1)复杂度）</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-15 21:52:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/0d/5a/e60f4125.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>camel</span>
  </div>
  <div class="_2_QraFYR_0">请问为什么特别强调sds元数据的大小？key超过32，元数据才从1到3，相对于本身key大小而言元数据不是才占很小比例吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-15 21:49:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/23/5b/983408b9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>悟空聊架构</span>
  </div>
  <div class="_2_QraFYR_0">总结得很棒，还需要在实践中不断地总结。谢谢老师！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-14 07:25:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/1d/de/62bfa83f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>aoe</span>
  </div>
  <div class="_2_QraFYR_0">非常实用，感谢老师</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-17 22:20:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTIIMRcKFGZTZCZCmm6ibb8KrjFaXxAm90R1Uic6lpBTN0HKKGQnzto6hnyQBfTNcYEoXAnJxhRQnUEg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_4d3b66</span>
  </div>
  <div class="_2_QraFYR_0">就文中hash集合过大，我不明白的是hash不是key  field value吗，一个key怎么可能过大呢，是存储了上万字段吗</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-13 09:18:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/7d/41/3c5b770b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>喵喵喵</span>
  </div>
  <div class="_2_QraFYR_0">打卡</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-30 10:22:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/d4/d6/1d4543ac.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>云海</span>
  </div>
  <div class="_2_QraFYR_0">建议里的Redis实例容量控制在2~6GB，这个数据是如何而来的呢，现在很多大型集群动辄几十上百G。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-22 00:41:33</div>
  </div>
</div>
</div>
</li>
</ul>