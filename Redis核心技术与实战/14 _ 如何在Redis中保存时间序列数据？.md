<audio title="14 _ 如何在Redis中保存时间序列数据？" src="https://static001.geekbang.org/resource/audio/9b/de/9b2bd52da8e40203cab8bd933e4588de.mp3" controls="controls"></audio> 
<p>你好，我是蒋德钧。</p><p>我们现在做互联网产品的时候，都有这么一个需求：记录用户在网站或者App上的点击行为数据，来分析用户行为。这里的数据一般包括用户ID、行为类型（例如浏览、登录、下单等）、行为发生的时间戳：</p><pre><code>UserID, Type, TimeStamp
</code></pre><p>我之前做过的一个物联网项目的数据存取需求，和这个很相似。我们需要周期性地统计近万台设备的实时状态，包括设备ID、压力、温度、湿度，以及对应的时间戳：</p><pre><code>DeviceID, Pressure, Temperature, Humidity, TimeStamp
</code></pre><p>这些与发生时间相关的一组数据，就是时间序列数据。<strong>这些数据的特点是没有严格的关系模型，记录的信息可以表示成键和值的关系</strong>（例如，一个设备ID对应一条记录），所以，并不需要专门用关系型数据库（例如MySQL）来保存。而Redis的键值数据模型，正好可以满足这里的数据存取需求。Redis基于自身数据结构以及扩展模块，提供了两种解决方案。</p><p>这节课，我就以物联网场景中统计设备状态指标值为例，和你聊聊不同解决方案的做法和优缺点。</p><p>俗话说，“知己知彼，百战百胜”，我们就先从时间序列数据的读写特点开始，看看到底应该采用什么样的数据类型来保存吧。</p><h2>时间序列数据的读写特点</h2><p>在实际应用中，时间序列数据通常是持续高并发写入的，例如，需要连续记录数万个设备的实时状态值。同时，时间序列数据的写入主要就是插入新数据，而不是更新一个已存在的数据，也就是说，一个时间序列数据被记录后通常就不会变了，因为它就代表了一个设备在某个时刻的状态值（例如，一个设备在某个时刻的温度测量值，一旦记录下来，这个值本身就不会再变了）。</p><!-- [[[read_end]]] --><p>所以，<strong>这种数据的写入特点很简单，就是插入数据快，这就要求我们选择的数据类型，在进行数据插入时，复杂度要低，尽量不要阻塞</strong>。看到这儿，你可能第一时间会想到用Redis的String、Hash类型来保存，因为它们的插入复杂度都是O(1)，是个不错的选择。但是，我在<a href="https://time.geekbang.org/column/article/279649">第11讲</a>中说过，String类型在记录小数据时（例如刚才例子中的设备温度值），元数据的内存开销比较大，不太适合保存大量数据。</p><p>那我们再看看，时间序列数据的“读”操作有什么特点。</p><p>我们在查询时间序列数据时，既有对单条记录的查询（例如查询某个设备在某一个时刻的运行状态信息，对应的就是这个设备的一条记录），也有对某个时间范围内的数据的查询（例如每天早上8点到10点的所有设备的状态信息）。</p><p>除此之外，还有一些更复杂的查询，比如对某个时间范围内的数据做聚合计算。这里的聚合计算，就是对符合查询条件的所有数据做计算，包括计算均值、最大/最小值、求和等。例如，我们要计算某个时间段内的设备压力的最大值，来判断是否有故障发生。</p><p>那用一个词概括时间序列数据的“读”，就是查询模式多。</p><p>弄清楚了时间序列数据的读写特点，接下来我们就看看如何在Redis中保存这些数据。我们来分析下：针对时间序列数据的“写要快”，Redis的高性能写特性直接就可以满足了；而针对“查询模式多”，也就是要支持单点查询、范围查询和聚合计算，Redis提供了保存时间序列数据的两种方案，分别可以基于Hash和Sorted Set实现，以及基于RedisTimeSeries模块实现。</p><p>接下来，我们先学习下第一种方案。</p><h2>基于Hash和Sorted Set保存时间序列数据</h2><p>Hash和Sorted Set组合的方式有一个明显的好处：它们是Redis内在的数据类型，代码成熟和性能稳定。所以，基于这两个数据类型保存时间序列数据，系统稳定性是可以预期的。</p><p>不过，在前面学习的场景中，我们都是使用一个数据类型来存取数据，那么，<strong>为什么保存时间序列数据，要同时使用这两种类型？这是我们要回答的第一个问题。</strong></p><p>关于Hash类型，我们都知道，它有一个特点是，可以实现对单键的快速查询。这就满足了时间序列数据的单键查询需求。我们可以把时间戳作为Hash集合的key，把记录的设备状态值作为Hash集合的value。</p><p>可以看下用Hash集合记录设备的温度值的示意图：</p><p><img src="https://static001.geekbang.org/resource/image/f2/be/f2e7bc4586be59aa5e7e78a5599830be.jpg?wh=2556*1530" alt=""></p><p>当我们想要查询某个时间点或者是多个时间点上的温度数据时，直接使用HGET命令或者HMGET命令，就可以分别获得Hash集合中的一个key和多个key的value值了。</p><p>举个例子。我们用HGET命令查询202008030905这个时刻的温度值，使用HMGET查询202008030905、202008030907、202008030908这三个时刻的温度值，如下所示：</p><pre><code>HGET device:temperature 202008030905
&quot;25.1&quot;

HMGET device:temperature 202008030905 202008030907 202008030908
1) &quot;25.1&quot;
2) &quot;25.9&quot;
3) &quot;24.9&quot;
</code></pre><p>你看，用Hash类型来实现单键的查询很简单。但是，<strong>Hash类型有个短板：它并不支持对数据进行范围查询。</strong></p><p>虽然时间序列数据是按时间递增顺序插入Hash集合中的，但Hash类型的底层结构是哈希表，并没有对数据进行有序索引。所以，如果要对Hash类型进行范围查询的话，就需要扫描Hash集合中的所有数据，再把这些数据取回到客户端进行排序，然后，才能在客户端得到所查询范围内的数据。显然，查询效率很低。</p><p>为了能同时支持按时间戳范围的查询，可以用Sorted Set来保存时间序列数据，因为它能够根据元素的权重分数来排序。我们可以把时间戳作为Sorted Set集合的元素分数，把时间点上记录的数据作为元素本身。</p><p>我还是以保存设备温度的时间序列数据为例，进行解释。下图显示了用Sorted Set集合保存的结果。</p><p><img src="https://static001.geekbang.org/resource/image/9e/7a/9e1214dbd5b42c5b3452ea73efc8c67a.jpg?wh=2564*1530" alt=""></p><p>使用Sorted Set保存数据后，我们就可以使用ZRANGEBYSCORE命令，按照输入的最大时间戳和最小时间戳来查询这个时间范围内的温度值了。如下所示，我们来查询一下在2020年8月3日9点7分到9点10分间的所有温度值：</p><pre><code>ZRANGEBYSCORE device:temperature 202008030907 202008030910
1) &quot;25.9&quot;
2) &quot;24.9&quot;
3) &quot;25.3&quot;
4) &quot;25.2&quot;
</code></pre><p>现在我们知道了，同时使用Hash和Sorted Set，可以满足单个时间点和一个时间范围内的数据查询需求了，但是我们又会面临一个新的问题，<strong>也就是我们要解答的第二个问题：如何保证写入Hash和Sorted Set是一个原子性的操作呢？</strong></p><p>所谓“原子性的操作”，就是指我们执行多个写命令操作时（例如用HSET命令和ZADD命令分别把数据写入Hash和Sorted Set），这些命令操作要么全部完成，要么都不完成。</p><p>只有保证了写操作的原子性，才能保证同一个时间序列数据，在Hash和Sorted Set中，要么都保存了，要么都没保存。否则，就可能出现Hash集合中有时间序列数据，而Sorted Set中没有，那么，在进行范围查询时，就没有办法满足查询需求了。</p><p>那Redis是怎么保证原子性操作的呢？这里就涉及到了Redis用来实现简单的事务的MULTI和EXEC命令。当多个命令及其参数本身无误时，MULTI和EXEC命令可以保证执行这些命令时的原子性。关于Redis的事务支持和原子性保证的异常情况，我会在第30讲中向你介绍，这节课，我们只要了解一下MULTI和EXEC这两个命令的使用方法就行了。</p><ul>
<li>MULTI命令：表示一系列原子性操作的开始。收到这个命令后，Redis就知道，接下来再收到的命令需要放到一个内部队列中，后续一起执行，保证原子性。</li>
<li>EXEC命令：表示一系列原子性操作的结束。一旦Redis收到了这个命令，就表示所有要保证原子性的命令操作都已经发送完成了。此时，Redis开始执行刚才放到内部队列中的所有命令操作。</li>
</ul><p>你可以看下下面这张示意图，命令1到命令N是在MULTI命令后、EXEC命令前发送的，它们会被一起执行，保证原子性。</p><p><img src="https://static001.geekbang.org/resource/image/c0/62/c0e2fd5834113cef92f2f68e7462a262.jpg?wh=2326*1338" alt=""></p><p>以保存设备状态信息的需求为例，我们执行下面的代码，把设备在2020年8月3日9时5分的温度，分别用HSET命令和ZADD命令写入Hash集合和Sorted Set集合。</p><pre><code>127.0.0.1:6379&gt; MULTI
OK

127.0.0.1:6379&gt; HSET device:temperature 202008030911 26.8
QUEUED

127.0.0.1:6379&gt; ZADD device:temperature 202008030911 26.8
QUEUED

127.0.0.1:6379&gt; EXEC
1) (integer) 1
2) (integer) 1
</code></pre><p>可以看到，首先，Redis收到了客户端执行的MULTI命令。然后，客户端再执行HSET和ZADD命令后，Redis返回的结果为“QUEUED”，表示这两个命令暂时入队，先不执行；执行了EXEC命令后，HSET命令和ZADD命令才真正执行，并返回成功结果（结果值为1）。</p><p>到这里，我们就解决了时间序列数据的单点查询、范围查询问题，并使用MUTLI和EXEC命令保证了Redis能原子性地把数据保存到Hash和Sorted Set中。<strong>接下来，我们需要继续解决第三个问题：如何对时间序列数据进行聚合计算？</strong></p><p>聚合计算一般被用来周期性地统计时间窗口内的数据汇总状态，在实时监控与预警等场景下会频繁执行。</p><p>因为Sorted Set只支持范围查询，无法直接进行聚合计算，所以，我们只能先把时间范围内的数据取回到客户端，然后在客户端自行完成聚合计算。这个方法虽然能完成聚合计算，但是会带来一定的潜在风险，也就是<strong>大量数据在Redis实例和客户端间频繁传输，这会和其他操作命令竞争网络资源，导致其他操作变慢。</strong></p><p>在我们这个物联网项目中，就需要每3分钟统计一下各个设备的温度状态，一旦设备温度超出了设定的阈值，就要进行报警。这是一个典型的聚合计算场景，我们可以来看看这个过程中的数据体量。</p><p>假设我们需要每3分钟计算一次的所有设备各指标的最大值，每个设备每15秒记录一个指标值，1分钟就会记录4个值，3分钟就会有12个值。我们要统计的设备指标数量有33个，所以，单个设备每3分钟记录的指标数据有将近400个（33 * 12 = 396），而设备总数量有1万台，这样一来，每3分钟就有将近400万条（396 * 1万 = 396万）数据需要在客户端和Redis实例间进行传输。</p><p>为了避免客户端和Redis实例间频繁的大量数据传输，我们可以使用RedisTimeSeries来保存时间序列数据。</p><p>RedisTimeSeries支持直接在Redis实例上进行聚合计算。还是以刚才每3分钟算一次最大值为例。在Redis实例上直接聚合计算，那么，对于单个设备的一个指标值来说，每3分钟记录的12条数据可以聚合计算成一个值，单个设备每3分钟也就只有33个聚合值需要传输，1万台设备也只有33万条数据。数据量大约是在客户端做聚合计算的十分之一，很显然，可以减少大量数据传输对Redis实例网络的性能影响。</p><p>所以，如果我们只需要进行单个时间点查询或是对某个时间范围查询的话，适合使用Hash和Sorted Set的组合，它们都是Redis的内在数据结构，性能好，稳定性高。但是，如果我们需要进行大量的聚合计算，同时网络带宽条件不是太好时，Hash和Sorted Set的组合就不太适合了。此时，使用RedisTimeSeries就更加合适一些。</p><p>好了，接下来，我们就来具体学习下RedisTimeSeries。</p><h2>基于RedisTimeSeries模块保存时间序列数据</h2><p>RedisTimeSeries是Redis的一个扩展模块。它专门面向时间序列数据提供了数据类型和访问接口，并且支持在Redis实例上直接对数据进行按时间范围的聚合计算。</p><p>因为RedisTimeSeries不属于Redis的内建功能模块，在使用时，我们需要先把它的源码单独编译成动态链接库redistimeseries.so，再使用loadmodule命令进行加载，如下所示：</p><pre><code>loadmodule redistimeseries.so
</code></pre><p>当用于时间序列数据存取时，RedisTimeSeries的操作主要有5个：</p><ul>
<li>用TS.CREATE命令创建时间序列数据集合；</li>
<li>用TS.ADD命令插入数据；</li>
<li>用TS.GET命令读取最新数据；</li>
<li>用TS.MGET命令按标签过滤查询数据集合；</li>
<li>用TS.RANGE支持聚合计算的范围查询。</li>
</ul><p>下面，我来介绍一下如何使用这5个操作。</p><p><strong>1.用TS.CREATE命令创建一个时间序列数据集合</strong></p><p>在TS.CREATE命令中，我们需要设置时间序列数据集合的key和数据的过期时间（以毫秒为单位）。此外，我们还可以为数据集合设置标签，来表示数据集合的属性。</p><p>例如，我们执行下面的命令，创建一个key为device:temperature、数据有效期为600s的时间序列数据集合。也就是说，这个集合中的数据创建了600s后，就会被自动删除。最后，我们给这个集合设置了一个标签属性{device_id:1}，表明这个数据集合中记录的是属于设备ID号为1的数据。</p><pre><code>TS.CREATE device:temperature RETENTION 600000 LABELS device_id 1
OK
</code></pre><p><strong>2.用TS.ADD命令插入数据，用TS.GET命令读取最新数据</strong></p><p>我们可以用TS.ADD命令往时间序列集合中插入数据，包括时间戳和具体的数值，并使用TS.GET命令读取数据集合中的最新一条数据。</p><p>例如，我们执行下列TS.ADD命令时，就往device:temperature集合中插入了一条数据，记录的是设备在2020年8月3日9时5分的设备温度；再执行TS.GET命令时，就会把刚刚插入的最新数据读取出来。</p><pre><code>TS.ADD device:temperature 1596416700 25.1
1596416700

TS.GET device:temperature 
25.1
</code></pre><p><strong>3.用TS.MGET命令按标签过滤查询数据集合</strong></p><p>在保存多个设备的时间序列数据时，我们通常会把不同设备的数据保存到不同集合中。此时，我们就可以使用TS.MGET命令，按照标签查询部分集合中的最新数据。在使用TS.CREATE创建数据集合时，我们可以给集合设置标签属性。当我们进行查询时，就可以在查询条件中对集合标签属性进行匹配，最后的查询结果里只返回匹配上的集合中的最新数据。</p><p>举个例子。假设我们一共用4个集合为4个设备保存时间序列数据，设备的ID号是1、2、3、4，我们在创建数据集合时，把device_id设置为每个集合的标签。此时，我们就可以使用下列TS.MGET命令，以及FILTER设置（这个配置项用来设置集合标签的过滤条件），查询device_id不等于2的所有其他设备的数据集合，并返回各自集合中的最新的一条数据。</p><pre><code>TS.MGET FILTER device_id!=2 
1) 1) &quot;device:temperature:1&quot;
   2) (empty list or set)
   3) 1) (integer) 1596417000
      2) &quot;25.3&quot;
2) 1) &quot;device:temperature:3&quot;
   2) (empty list or set)
   3) 1) (integer) 1596417000
      2) &quot;29.5&quot;
3) 1) &quot;device:temperature:4&quot;
   2) (empty list or set)
   3) 1) (integer) 1596417000
      2) &quot;30.1&quot;
</code></pre><p><strong>4.用TS.RANGE支持需要聚合计算的范围查询</strong></p><p>最后，在对时间序列数据进行聚合计算时，我们可以使用TS.RANGE命令指定要查询的数据的时间范围，同时用AGGREGATION参数指定要执行的聚合计算类型。RedisTimeSeries支持的聚合计算类型很丰富，包括求均值（avg）、求最大/最小值（max/min），求和（sum）等。</p><p>例如，在执行下列命令时，我们就可以按照每180s的时间窗口，对2020年8月3日9时5分和2020年8月3日9时12分这段时间内的数据进行均值计算了。</p><pre><code>TS.RANGE device:temperature 1596416700 1596417120 AGGREGATION avg 180000
1) 1) (integer) 1596416700
   2) &quot;25.6&quot;
2) 1) (integer) 1596416880
   2) &quot;25.8&quot;
3) 1) (integer) 1596417060
   2) &quot;26.1&quot;
</code></pre><p>与使用Hash和Sorted Set来保存时间序列数据相比，RedisTimeSeries是专门为时间序列数据访问设计的扩展模块，能支持在Redis实例上直接进行聚合计算，以及按标签属性过滤查询数据集合，当我们需要频繁进行聚合计算，以及从大量集合中筛选出特定设备或用户的数据集合时，RedisTimeSeries就可以发挥优势了。</p><h2>小结</h2><p>在这节课，我们一起学习了如何用Redis保存时间序列数据。时间序列数据的写入特点是要能快速写入，而查询的特点有三个：</p><ul>
<li>点查询，根据一个时间戳，查询相应时间的数据；</li>
<li>范围查询，查询起始和截止时间戳范围内的数据；</li>
<li>聚合计算，针对起始和截止时间戳范围内的所有数据进行计算，例如求最大/最小值，求均值等。</li>
</ul><p>关于快速写入的要求，Redis的高性能写特性足以应对了；而针对多样化的查询需求，Redis提供了两种方案。</p><p>第一种方案是，组合使用Redis内置的Hash和Sorted Set类型，把数据同时保存在Hash集合和Sorted Set集合中。这种方案既可以利用Hash类型实现对单键的快速查询，还能利用Sorted Set实现对范围查询的高效支持，一下子满足了时间序列数据的两大查询需求。</p><p>不过，第一种方案也有两个不足：一个是，在执行聚合计算时，我们需要把数据读取到客户端再进行聚合，当有大量数据要聚合时，数据传输开销大；另一个是，所有的数据会在两个数据类型中各保存一份，内存开销不小。不过，我们可以通过设置适当的数据过期时间，释放内存，减小内存压力。</p><p>我们学习的第二种实现方案是使用RedisTimeSeries模块。这是专门为存取时间序列数据而设计的扩展模块。和第一种方案相比，RedisTimeSeries能支持直接在Redis实例上进行多种数据聚合计算，避免了大量数据在实例和客户端间传输。不过，RedisTimeSeries的底层数据结构使用了链表，它的范围查询的复杂度是O(N)级别的，同时，它的TS.GET查询只能返回最新的数据，没有办法像第一种方案的Hash类型一样，可以返回任一时间点的数据。</p><p>所以，组合使用Hash和Sorted Set，或者使用RedisTimeSeries，在支持时间序列数据存取上各有优劣势。我给你的建议是：</p><ul>
<li>如果你的部署环境中网络带宽高、Redis实例内存大，可以优先考虑第一种方案；</li>
<li>如果你的部署环境中网络、内存资源有限，而且数据量大，聚合计算频繁，需要按数据集合属性查询，可以优先考虑第二种方案。</li>
</ul><h2>每课一问</h2><p>按照惯例，我给你提个小问题。</p><p>在这节课上，我提到，我们可以使用Sorted Set保存时间序列数据，把时间戳作为score，把实际的数据作为member，你觉得这样保存数据有没有潜在的风险？另外，如果你是Redis的开发维护者，你会把聚合计算也设计为Sorted Set的一个内在功能吗？</p><p>好了，这节课就到这里，如果你觉得有所收获，欢迎你把今天的内容分享给你的朋友或同事，我们下节课见。</p>
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
  <div class="_2_QraFYR_0">使用Sorted Set保存时序数据，把时间戳作为score，把实际的数据作为member，有什么潜在的风险？<br><br>我目前能想到的风险是，如果对某一个对象的时序数据记录很频繁的话，那么这个key很容易变成一个bigkey，在key过期释放内存时可能引发阻塞风险。所以不能把这个对象的所有时序数据存储在一个key上，而是需要拆分存储，例如可以按天&#47;周&#47;月拆分（根据具体查询需求来定）。当然，拆分key的缺点是，在查询时，可能需要客户端查询多个key后再做聚合才能得到结果。<br><br>如果你是Redis的开发维护者，你会把聚合计算也设计为Sorted Set的内在功能吗？<br><br>不会。因为聚合计算是CPU密集型任务，Redis在处理请求时是单线程的，也就是它在做聚合计算时无法利用到多核CPU来提升计算速度，如果计算量太大，这也会导致Redis的响应延迟变长，影响Redis的性能。Redis的定位就是高性能的内存数据库，要求访问速度极快。所以对于时序数据的存储和聚合计算，我觉得更好的方式是交给时序数据库去做，时序数据库会针对这些存储和计算的场景做针对性优化。<br><br>另外，在使用MULTI和EXEC命令时，建议客户端使用pipeline，当使用pipeline时，客户端会把命令一次性批量发送给服务端，然后让服务端执行，这样可以减少客户端和服务端的来回网络IO次数，提升访问性能。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-07 00:31:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/10/92/760f0964.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>阳明</span>
  </div>
  <div class="_2_QraFYR_0">存在member重复的问题，会对member覆盖</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-07 09:41:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/52/40/e57a736e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>pedro</span>
  </div>
  <div class="_2_QraFYR_0">Hash 和 Sorted Set 的结合让我想到了 LRU 中的 HashMap 和 LinkedList 的结合，二者均取长处碰撞出了不一样的火花，看看毫不沾边的事物，往往具有相同的内涵。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-07 08:57:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_1e8830</span>
  </div>
  <div class="_2_QraFYR_0">老师，你好，问个问题，基于redis的单线程原理lua脚本到底可不可以保证原子性？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Redis会用单线程方式执行Lua脚本，保证脚本执行过程中不被其他命令打断，一般我们称之为以原子性的方式执行。但是有个地方要注意，如果脚本中用redis.call函数时出错了，会导致执行中断，此时，Lua脚本是不会回滚的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-31 21:31:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/fd/fd/326be9bb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>注定非凡</span>
  </div>
  <div class="_2_QraFYR_0">1，作者讲了什么？<br>根据时间序列数据的特点，选择合适的存储方案<br><br>2，作者是怎么把这事给讲明白的？<br>结合具体场景，探讨解决方案<br>    1，介绍需求背景，用户行为，设备监控数据分析<br>    2，介绍数据特点，时间线连续，没有逻辑关系，数据量大<br>    3，介绍操作场景，插入多且快，常单点查询，分组统计聚合<br> <br>3，作者为了把这事给讲清楚，讲了那些要点？有哪些亮点？<br>1，亮点1：先讲清楚需求背景，从实际问题出发，推演出存储时间序列数据适合使用sort set 和hash解决点查询和范围查询的需求<br>2，要点1：同时写入sort set和hash 两种数据类型的存储，需要使用原子操作，可以借助MULTI和 EXEC命令<br>3，要点2：大数据量的聚合统计，会非常消耗网络带宽，所以可以使用RedisTimeSeries模块处理<br><br>4，对于作者所讲，我有哪些发散性思考？<br><br>5，在未来的那些场景中，我能够使用它？<br><br>6，留言区的收获</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-14 22:50:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/43/79/18073134.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>test</span>
  </div>
  <div class="_2_QraFYR_0">redis的事务不是完整的事务，当有一个命令失败时还是会继续往下执行，这是个问题。时序数据还是交给时序数据库来保存比较专业</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-07 08:54:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/cb/f9/75d08ccf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Mr.蜜</span>
  </div>
  <div class="_2_QraFYR_0">Sorted Set还是基于Set集合的，所以如果member值相同，那么ZADD只会更新score，存在数据丢失的风险。我有个问题：既然每三分钟聚合一次计算，为何不直接按时间统计值呢？比方说hincrby，把指定一段时间的温度聚合在一起，可以用lua脚本，实现此类计算，这样既实现了原子性，又不会特别消耗内存，还能实现数据统计。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-19 00:13:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/17/28/6002bfd7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Eric.Lee</span>
  </div>
  <div class="_2_QraFYR_0">有个问题：市面上有成熟的时间序列数据库如：influxdb、Prometheus等。这一讲，我理解是介绍了Redis支持通过加载模块的形式也能支持这种数据类型的存储。单从时间序列管理、功能方面上个人感觉不如专业数据库成熟？还是作者做过这些数据库的比较专门选用的Redis做数据存储？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-27 13:15:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>dfuru</span>
  </div>
  <div class="_2_QraFYR_0">hash和sorted set类型的key不能相同，文件是相同的。<br>127.0.0.1:6379&gt; MULTI<br>OK<br>127.0.0.1:6379&gt; ZADD dev:temp4 2020092201 26.8<br>QUEUED<br>127.0.0.1:6379&gt; HSET dev:temp4 2020092201 26.8<br>QUEUED<br>127.0.0.1:6379&gt; EXEC<br>1) (integer) 1<br>2) (error) WRONGTYPE Operation against a key holding the wrong kind of value<br>127.0.0.1:6379&gt; MULTI<br>OK<br>127.0.0.1:6379&gt; HSET dev:temp5 2020092201 26.8<br>QUEUED<br>127.0.0.1:6379&gt; ZADD dev:temp6 2020092201 26.8<br>QUEUED<br>127.0.0.1:6379&gt; EXEC<br>1) (integer) 1<br>2) (integer) 1<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-22 20:08:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/37/29/b3af57a7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>凯文小猪</span>
  </div>
  <div class="_2_QraFYR_0">虽然有点泼冷水 但实际上点击流数据 通常是用kafka 作为日志来中转的。这里面涉及几个问题：<br>1。数据的维度并不总是固定的。比方说今天要聚合统计 明天可能要最值或基数统计。那这时候用redis存储显然不是第一选择。<br>2. 数据清洗问题。不是所有的数据都是我们期望的 但我们不能要求埋点方来做数据清洗 我们只能要求埋点方尽可能多的上报 才能保证数据是正确的 偏差值 最小。<br>3. 数据量存储问题。比方说亿级流量 每秒点击流是1W qps ，以一个上送数据1KB估算：那么一天数据量有1KB * 10000 * 3600 * 24 = 864GB 实际上无论是用切片 还是集群都存不下</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-26 19:25:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/2c/82/98e2b82a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Reborn 2.0</span>
  </div>
  <div class="_2_QraFYR_0">我想问, 使用sorted set存储, zset不是本身就维护了一个dict么? 为什么还要用hash再存一遍呢?</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-28 17:51:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span></span>
  </div>
  <div class="_2_QraFYR_0">老师，看redis设计与实现这本书中讲的，有序集合编码类型为zset时，同时使用了跳跃表和字典来实现。执行zadd命令，会先调用zsinsert函数将新元素添加到跳跃表，然后调用dictAdd函数，将新元素关联到字典。这样让有序集合的查找和范围操作都尽可能地快。不需要使用方同时去调用hset+zadd</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-29 13:03:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/24/ee/46/7d65ae37.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>木几丶</span>
  </div>
  <div class="_2_QraFYR_0">实际上multi只是redis提供的一个简易的事务操作，如果入队列时就能检测出命令错误的情况，事务会被回滚，而对于在执行时才能检测出错误的情况（如评论中提到的类型错误），前后命令正常执行，事务将不会被回滚，这点跟数据库很不一样 需要特别注意</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-04 17:21:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/f8/ba/14e05601.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>约书亚</span>
  </div>
  <div class="_2_QraFYR_0">以前是做物联网的，看完之后有几点思考：<br>1. 为什么查询指定时间的数据不直接用sorted set的查询操作，虽然是log(N）的时间，比o(1)要慢很多，但这种场景毕竟占比很小。而一般专业时序数据库也确实是这么做的。<br>2. 时序数据库一个常见场景跨时间线的聚合运算，第一个方案的例子都是在解决单条时间线的问题，场景非常单一。在这点上RedisTimeSeries是否可以实现？是否能更好的利用多核并行？<br>3. 比起原子性问题，我觉得redis丢数据的问题更严重。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-30 10:03:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/66/1a/9e9f7d58.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>少年</span>
  </div>
  <div class="_2_QraFYR_0">TS.ADD device:temperature:1 1596416700 25.1<br>key指定device id做demo感觉会更清晰些</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-21 22:18:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/46/a0/aa6d4ecd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张潇赟</span>
  </div>
  <div class="_2_QraFYR_0">这一节中老师的例子有点没说明白：<br>1.hash 和sorted set 不能用同样的key<br>2.mutli exec 并不能保证全部成功，redis对事物的支持只是保证两条命令全部被执行，不保证执行成功。如果有保存hash成功了，保存sorted set失败。redis会维持现状，不会对hash回滚</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-26 19:58:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/bb/56/a506a165.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>蓝萧</span>
  </div>
  <div class="_2_QraFYR_0">个人认为 redis中的stream类型也可以用于保存时间类型数据，stream XADD会自动保存时间戳参数，使用XRANGE也可以添加时间戳用于范围查询，只是无法像timeseries一样做聚合计算，个人认为做聚合计算这种大量消耗CPU的操作，不适合直接放在redis中进行，更适合放在单独的客户端，加上云服务本身默认不支持time series而默认支持stream.综上所述，如果使用redis存储时间类型数据，建议使用stream</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-04 11:52:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/7e/1f/b1d458a9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>iamjohnnyzhuang</span>
  </div>
  <div class="_2_QraFYR_0">课后题：使用 sorted set 时间戳作为score，温度作为 member 的问题应该主要还是 温度大概率会出现相同的，这个时候 zadd 后会覆盖掉原有的数据，个人觉得的一个解决方法就是value使用时间戳_温度 这样存储。<br><br>关于 sorted set + hash 存储时许的方案有个点想的不是很懂，希望有同学或者老师帮忙下：之前看过 zset 的实现，实际上也是 hash + skiplist 实现，skiplist 负责遍历，hash 负责和本文说的一样 依据 score 找到对应的 member（zscore 命令）。 所以感觉多弄了一个hash来存储是不是多此一举呢？<br><br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-21 23:13:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/e8/72/e44c69ef.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>夏虫井蛙</span>
  </div>
  <div class="_2_QraFYR_0">现在很多服务都上云了，用的redis也是供应商提供的服务，一般不自己搭建。RedisTimeSeries以及上一讲的自定义数据类型需要编译加载，一般云供应商不提供这些吧？这时候是不是只能用基础数据类型，没办法用RedisTimeSeries以及自定义数据类型了？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-09 16:25:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_LV</span>
  </div>
  <div class="_2_QraFYR_0">127.0.0.1:6379&gt; multi<br>OK<br>127.0.0.1:6379&gt; HSET device:temperature 202008030911 26.8<br>QUEUED<br>127.0.0.1:6379&gt; ZADD device:temperature 202008030911 26.8<br>QUEUED<br>127.0.0.1:6379&gt; exec<br>1) (integer) 1<br>2) (error) WRONGTYPE Operation against a key holding the wrong kind of value</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-08 17:30:12</div>
  </div>
</div>
</div>
</li>
</ul>