<audio title="11 _ “万金油”的String，为什么不好用了？" src="https://static001.geekbang.org/resource/audio/9a/3b/9a7e6698a3d102e66ac5fd92f3f4b33b.mp3" controls="controls"></audio> 
<p>你好，我是蒋德钧。</p><p>从今天开始，我们就要进入“实践篇”了。接下来，我们会用5节课的时间学习“数据结构”。我会介绍节省内存开销以及保存和统计海量数据的数据类型及其底层数据结构，还会围绕典型的应用场景（例如地址位置查询、时间序列数据库读写和消息队列存取），跟你分享使用Redis的数据类型和module扩展功能来满足需求的具体方案。</p><p>今天，我们先了解下String类型的内存空间消耗问题，以及选择节省内存开销的数据类型的解决方案。</p><p>先跟你分享一个我曾经遇到的需求。</p><p>当时，我们要开发一个图片存储系统，要求这个系统能快速地记录图片ID和图片在存储系统中保存时的ID（可以直接叫作图片存储对象ID）。同时，还要能够根据图片ID快速查找到图片存储对象ID。</p><p>因为图片数量巨大，所以我们就用10位数来表示图片ID和图片存储对象ID，例如，图片ID为1101000051，它在存储系统中对应的ID号是3301000051。</p><pre><code>photo_id: 1101000051
photo_obj_id: 3301000051
</code></pre><p>可以看到，图片ID和图片存储对象ID正好一一对应，是典型的“键-单值”模式。所谓的“单值”，就是指键值对中的值就是一个值，而不是一个集合，这和String类型提供的“一个键对应一个值的数据”的保存形式刚好契合。</p><!-- [[[read_end]]] --><p>而且，String类型可以保存二进制字节流，就像“万金油”一样，只要把数据转成二进制字节数组，就可以保存了。</p><p>所以，我们的第一个方案就是用String保存数据。我们把图片ID和图片存储对象ID分别作为键值对的key和value来保存，其中，图片存储对象ID用了String类型。</p><p>刚开始，我们保存了1亿张图片，大约用了6.4GB的内存。但是，随着图片数据量的不断增加，我们的Redis内存使用量也在增加，结果就遇到了大内存Redis实例因为生成RDB而响应变慢的问题。很显然，String类型并不是一种好的选择，我们还需要进一步寻找能节省内存开销的数据类型方案。</p><p>在这个过程中，我深入地研究了String类型的底层结构，找到了它内存开销大的原因，对“万金油”的String类型有了全新的认知：String类型并不是适用于所有场合的，它有一个明显的短板，就是它保存数据时所消耗的内存空间较多。</p><p>同时，我还仔细研究了集合类型的数据结构。我发现，集合类型有非常节省内存空间的底层实现结构，但是，集合类型保存的数据模式，是一个键对应一系列值，并不适合直接保存单值的键值对。所以，我们就使用二级编码的方法，实现了用集合类型保存单值键值对，Redis实例的内存空间消耗明显下降了。</p><p>这节课，我就把在解决这个问题时学到的经验和方法分享给你，包括String类型的内存空间消耗在哪儿了、用什么数据结构可以节省内存，以及如何用集合类型保存单值键值对。如果你在使用String类型时也遇到了内存空间消耗较多的问题，就可以尝试下今天的解决方案了。</p><p>接下来，我们先来看看String类型的内存都消耗在哪里了。</p><h2>为什么String类型内存开销大？</h2><p>在刚才的案例中，我们保存了1亿张图片的信息，用了约6.4GB的内存，一个图片ID和图片存储对象ID的记录平均用了64字节。</p><p>但问题是，一组图片ID及其存储对象ID的记录，实际只需要16字节就可以了。</p><p>我们来分析一下。图片ID和图片存储对象ID都是10位数，我们可以用两个8字节的Long类型表示这两个ID。因为8字节的Long类型最大可以表示2的64次方的数值，所以肯定可以表示10位数。但是，为什么String类型却用了64字节呢？</p><p>其实，除了记录实际数据，String类型还需要额外的内存空间记录数据长度、空间使用等信息，这些信息也叫作元数据。当实际保存的数据较小时，元数据的空间开销就显得比较大了，有点“喧宾夺主”的意思。</p><p>那么，String类型具体是怎么保存数据的呢？我来解释一下。</p><p>当你保存64位有符号整数时，String类型会把它保存为一个8字节的Long类型整数，这种保存方式通常也叫作int编码方式。</p><p>但是，当你保存的数据中包含字符时，String类型就会用简单动态字符串（Simple Dynamic String，SDS）结构体来保存，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/37/57/37c6a8d5abd65906368e7c4a6b938657.jpg?wh=1926*1400" alt=""></p><ul>
<li><strong>buf</strong>：字节数组，保存实际数据。为了表示字节数组的结束，Redis会自动在数组最后加一个“\0”，这就会额外占用1个字节的开销。</li>
<li><strong>len</strong>：占4个字节，表示buf的已用长度。</li>
<li><strong>alloc</strong>：也占个4字节，表示buf的实际分配长度，一般大于len。</li>
</ul><p>可以看到，在SDS中，buf保存实际数据，而len和alloc本身其实是SDS结构体的额外开销。</p><p>另外，对于String类型来说，除了SDS的额外开销，还有一个来自于RedisObject结构体的开销。</p><p>因为Redis的数据类型有很多，而且，不同数据类型都有些相同的元数据要记录（比如最后一次访问的时间、被引用的次数等），所以，Redis会用一个RedisObject结构体来统一记录这些元数据，同时指向实际数据。</p><p>一个RedisObject包含了8字节的元数据和一个8字节指针，这个指针再进一步指向具体数据类型的实际数据所在，例如指向String类型的SDS结构所在的内存地址，可以看一下下面的示意图。关于RedisObject的具体结构细节，我会在后面的课程中详细介绍，现在你只要了解它的基本结构和元数据开销就行了。</p><p><img src="https://static001.geekbang.org/resource/image/34/57/3409948e9d3e8aa5cd7cafb9b66c2857.jpg?wh=2214*1656" alt=""></p><p>为了节省内存空间，Redis还对Long类型整数和SDS的内存布局做了专门的设计。</p><p>一方面，当保存的是Long类型整数时，RedisObject中的指针就直接赋值为整数数据了，这样就不用额外的指针再指向整数了，节省了指针的空间开销。</p><p>另一方面，当保存的是字符串数据，并且字符串小于等于44字节时，RedisObject中的元数据、指针和SDS是一块连续的内存区域，这样就可以避免内存碎片。这种布局方式也被称为embstr编码方式。</p><p>当然，当字符串大于44字节时，SDS的数据量就开始变多了，Redis就不再把SDS和RedisObject布局在一起了，而是会给SDS分配独立的空间，并用指针指向SDS结构。这种布局方式被称为raw编码模式。</p><p>为了帮助你理解int、embstr和raw这三种编码模式，我画了一张示意图，如下所示：</p><p><img src="https://static001.geekbang.org/resource/image/ce/e3/ce83d1346c9642fdbbf5ffbe701bfbe3.jpg?wh=3072*1938" alt=""></p><p>好了，知道了RedisObject所包含的额外元数据开销，现在，我们就可以计算String类型的内存使用量了。</p><p>因为10位数的图片ID和图片存储对象ID是Long类型整数，所以可以直接用int编码的RedisObject保存。每个int编码的RedisObject元数据部分占8字节，指针部分被直接赋值为8字节的整数了。此时，每个ID会使用16字节，加起来一共是32字节。但是，另外的32字节去哪儿了呢？</p><p>我在<a href="https://time.geekbang.org/column/article/268253">第2讲</a>中说过，Redis会使用一个全局哈希表保存所有键值对，哈希表的每一项是一个dictEntry的结构体，用来指向一个键值对。dictEntry结构中有三个8字节的指针，分别指向key、value以及下一个dictEntry，三个指针共24字节，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/b6/e7/b6cbc5161388fdf4c9b49f3802ef53e7.jpg?wh=2219*1371" alt=""></p><p>但是，这三个指针只有24字节，为什么会占用了32字节呢？这就要提到Redis使用的内存分配库jemalloc了。</p><p>jemalloc在分配内存时，会根据我们申请的字节数N，找一个比N大，但是最接近N的2的幂次数作为分配的空间，这样可以减少频繁分配的次数。</p><p>举个例子。如果你申请6字节空间，jemalloc实际会分配8字节空间；如果你申请24字节空间，jemalloc则会分配32字节。所以，在我们刚刚说的场景里，dictEntry结构就占用了32字节。</p><p>好了，到这儿，你应该就能理解，为什么用String类型保存图片ID和图片存储对象ID时需要用64个字节了。</p><p>你看，明明有效信息只有16字节，使用String类型保存时，却需要64字节的内存空间，有48字节都没有用于保存实际的数据。我们来换算下，如果要保存的图片有1亿张，那么1亿条的图片ID记录就需要6.4GB内存空间，其中有4.8GB的内存空间都用来保存元数据了，额外的内存空间开销很大。那么，有没有更加节省内存的方法呢？</p><h2>用什么数据结构可以节省内存？</h2><p>Redis有一种底层数据结构，叫压缩列表（ziplist），这是一种非常节省内存的结构。</p><p>我们先回顾下压缩列表的构成。表头有三个字段zlbytes、zltail和zllen，分别表示列表长度、列表尾的偏移量，以及列表中的entry个数。压缩列表尾还有一个zlend，表示列表结束。</p><p><img src="https://static001.geekbang.org/resource/image/f6/9f/f6d4df5f7d6e80de29e2c6446b02429f.jpg?wh=3457*972" alt=""></p><p>压缩列表之所以能节省内存，就在于它是用一系列连续的entry保存数据。每个entry的元数据包括下面几部分。</p><ul>
<li><strong>prev_len</strong>，表示前一个entry的长度。prev_len有两种取值情况：1字节或5字节。取值1字节时，表示上一个entry的长度小于254字节。虽然1字节的值能表示的数值范围是0到255，但是压缩列表中zlend的取值默认是255，因此，就默认用255表示整个压缩列表的结束，其他表示长度的地方就不能再用255这个值了。所以，当上一个entry长度小于254字节时，prev_len取值为1字节，否则，就取值为5字节。</li>
<li><strong>len</strong>：表示自身长度，4字节；</li>
<li><strong>encoding</strong>：表示编码方式，1字节；</li>
<li><strong>content</strong>：保存实际数据。</li>
</ul><p>这些entry会挨个儿放置在内存中，不需要再用额外的指针进行连接，这样就可以节省指针所占用的空间。</p><p>我们以保存图片存储对象ID为例，来分析一下压缩列表是如何节省内存空间的。</p><p>每个entry保存一个图片存储对象ID（8字节），此时，每个entry的prev_len只需要1个字节就行，因为每个entry的前一个entry长度都只有8字节，小于254字节。这样一来，一个图片的存储对象ID所占用的内存大小是14字节（1+4+1+8=14），实际分配16字节。</p><p>Redis基于压缩列表实现了List、Hash和Sorted Set这样的集合类型，这样做的最大好处就是节省了dictEntry的开销。当你用String类型时，一个键值对就有一个dictEntry，要用32字节空间。但采用集合类型时，一个key就对应一个集合的数据，能保存的数据多了很多，但也只用了一个dictEntry，这样就节省了内存。</p><p>这个方案听起来很好，但还存在一个问题：在用集合类型保存键值对时，一个键对应了一个集合的数据，但是在我们的场景中，一个图片ID只对应一个图片的存储对象ID，我们该怎么用集合类型呢？换句话说，在一个键对应一个值（也就是单值键值对）的情况下，我们该怎么用集合类型来保存这种单值键值对呢？</p><h2>如何用集合类型保存单值的键值对？</h2><p>在保存单值的键值对时，可以采用基于Hash类型的二级编码方法。这里说的二级编码，就是把一个单值的数据拆分成两部分，前一部分作为Hash集合的key，后一部分作为Hash集合的value，这样一来，我们就可以把单值数据保存到Hash集合中了。</p><p>以图片ID 1101000060和图片存储对象ID 3302000080为例，我们可以把图片ID的前7位（1101000）作为Hash类型的键，把图片ID的最后3位（060）和图片存储对象ID分别作为Hash类型值中的key和value。</p><p>按照这种设计方法，我在Redis中插入了一组图片ID及其存储对象ID的记录，并且用info命令查看了内存开销，我发现，增加一条记录后，内存占用只增加了16字节，如下所示：</p><pre><code>127.0.0.1:6379&gt; info memory
# Memory
used_memory:1039120
127.0.0.1:6379&gt; hset 1101000 060 3302000080
(integer) 1
127.0.0.1:6379&gt; info memory
# Memory
used_memory:1039136
</code></pre><p>在使用String类型时，每个记录需要消耗64字节，这种方式却只用了16字节，所使用的内存空间是原来的1/4，满足了我们节省内存空间的需求。</p><p>不过，你可能也会有疑惑：“二级编码一定要把图片ID的前7位作为Hash类型的键，把最后3位作为Hash类型值中的key吗？”<strong>其实，二级编码方法中采用的ID长度是有讲究的</strong>。</p><p>在<a href="https://time.geekbang.org/column/article/268253">第2讲</a>中，我介绍过Redis Hash类型的两种底层实现结构，分别是压缩列表和哈希表。</p><p>那么，Hash类型底层结构什么时候使用压缩列表，什么时候使用哈希表呢？其实，Hash类型设置了用压缩列表保存数据时的两个阈值，一旦超过了阈值，Hash类型就会用哈希表来保存数据了。</p><p>这两个阈值分别对应以下两个配置项：</p><ul>
<li>hash-max-ziplist-entries：表示用压缩列表保存时哈希集合中的最大元素个数。</li>
<li>hash-max-ziplist-value：表示用压缩列表保存时哈希集合中单个元素的最大长度。</li>
</ul><p>如果我们往Hash集合中写入的元素个数超过了hash-max-ziplist-entries，或者写入的单个元素大小超过了hash-max-ziplist-value，Redis就会自动把Hash类型的实现结构由压缩列表转为哈希表。</p><p>一旦从压缩列表转为了哈希表，Hash类型就会一直用哈希表进行保存，而不会再转回压缩列表了。在节省内存空间方面，哈希表就没有压缩列表那么高效了。</p><p><strong>为了能充分使用压缩列表的精简内存布局，我们一般要控制保存在Hash集合中的元素个数</strong>。所以，在刚才的二级编码中，我们只用图片ID最后3位作为Hash集合的key，也就保证了Hash集合的元素个数不超过1000，同时，我们把hash-max-ziplist-entries设置为1000，这样一来，Hash集合就可以一直使用压缩列表来节省内存空间了。</p><h2>小结</h2><p>这节课，我们打破了对String的认知误区，以前，我们认为String是“万金油”，什么场合都适用，但是，在保存的键值对本身占用的内存空间不大时（例如这节课里提到的的图片ID和图片存储对象ID），String类型的元数据开销就占据主导了，这里面包括了RedisObject结构、SDS结构、dictEntry结构的内存开销。</p><p>针对这种情况，我们可以使用压缩列表保存数据。当然，使用Hash这种集合类型保存单值键值对的数据时，我们需要将单值数据拆分成两部分，分别作为Hash集合的键和值，就像刚才案例中用二级编码来表示图片ID，希望你能把这个方法用到自己的场景中。</p><p>最后，我还想再给你提供一个小方法：如果你想知道键值对采用不同类型保存时的内存开销，可以在<a href="http://www.redis.cn/redis_memory/">这个网址</a>里输入你的键值对长度和使用的数据类型，这样就能知道实际消耗的内存大小了。建议你把这个小工具用起来，它可以帮助你充分地节省内存。</p><h2>每课一问</h2><p>按照惯例，给你提个小问题：除了String类型和Hash类型，你觉得，还有其他合适的类型可以应用在这节课所说的保存图片的例子吗？</p><p>欢迎在留言区写下你的思考和答案，我们一起交流讨论，也欢迎你把今天的内容分享给你的朋友。</p>
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
  <div class="_2_QraFYR_0">保存图片的例子，除了用String和Hash存储之外，还可以用Sorted Set存储（勉强）。<br><br>Sorted Set与Hash类似，当元素数量少于zset-max-ziplist-entries，并且每个元素内存占用小于zset-max-ziplist-value时，默认也采用ziplist结构存储。我们可以把zset-max-ziplist-entries参数设置为1000，这样Sorted Set默认就会使用ziplist存储了，member和score也会紧凑排列存储，可以节省内存空间。<br><br>使用zadd 1101000 3302000080 060命令存储图片ID和对象ID的映射关系，查询时使用zscore 1101000 060获取结果。<br><br>但是Sorted Set使用ziplist存储时的缺点是，这个ziplist是需要按照score排序的（为了方便zrange和zrevrange命令的使用），所以在插入一个元素时，需要先根据score找到对应的位置，然后把member和score插入进去，这也意味着Sorted Set插入元素的性能没有Hash高（这也是前面说勉强能用Sorte Set存储的原因）。而Hash在插入元素时，只需要将新的元素插入到ziplist的尾部即可，不需要定位到指定位置。<br><br>不管是使用Hash还是Sorted Set，当采用ziplist方式存储时，虽然可以节省内存空间，但是在查询指定元素时，都要遍历整个ziplist，找到指定的元素。所以使用ziplist方式存储时，虽然可以利用CPU高速缓存，但也不适合存储过多的数据（hash-max-ziplist-entries和zset-max-ziplist-entries不宜设置过大），否则查询性能就会下降比较厉害。整体来说，这样的方案就是时间换空间，我们需要权衡使用。<br><br>当使用ziplist存储时，我们尽量存储int数据，ziplist在设计时每个entry都进行了优化，针对要存储的数据，会尽量选择占用内存小的方式存储（整数比字符串在存储时占用内存更小），这也有利于我们节省Redis的内存。还有，因为ziplist是每个元素紧凑排列，而且每个元素存储了上一个元素的长度，所以当修改其中一个元素超过一定大小时，会引发多个元素的级联调整（前面一个元素发生大的变动，后面的元素都要重新排列位置，重新分配内存），这也会引发性能问题，需要注意。<br><br>另外，使用Hash和Sorted Set存储时，虽然节省了内存空间，但是设置过期变得困难（无法控制每个元素的过期，只能整个key设置过期，或者业务层单独维护每个元素过期删除的逻辑，但比较复杂）。而使用String虽然占用内存多，但是每个key都可以单独设置过期时间，还可以设置maxmemory和淘汰策略，以这种方式控制整个实例的内存上限。<br><br>所以在选用Hash和Sorted Set存储时，意味着把Redis当做数据库使用，这样就需要务必保证Redis的可靠性（做好备份、主从副本），防止实例宕机引发数据丢失的风险。而采用String存储时，可以把Redis当做缓存使用，每个key设置过期时间，同时设置maxmemory和淘汰策略，控制整个实例的内存上限，这种方案需要在数据库层（例如MySQL）也存储一份映射关系，当Redis中的缓存过期或被淘汰时，需要从数据库中重新查询重建缓存，同时需要保证数据库和缓存的一致性，这些逻辑也需要编写业务代码实现。<br><br>总之，各有利弊，我们需要根据实际场景进行选择。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-31 11:36:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/7d/8e/bb16d414.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Wangxi</span>
  </div>
  <div class="_2_QraFYR_0">实测老师的例子，长度7位数，共100万条数据。使用string占用70mb，使用hash ziplist只占用9mb。效果非常明显。redis版本6.0.6</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞 实践的态度！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-31 18:06:30</div>
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
  <div class="_2_QraFYR_0">一，作者讲了什么？<br>    Redis的String类型数据结构，及其底层实现<br>二，作者是怎么把这事给说明白的？<br>    1，通过一个图片存储的案例，讲通过合理利用Redis的数据结构，降低资源消耗<br><br>三，为了讲明白，作者讲了哪些要点？有哪些亮点？<br>1，亮点1：String类型的数据占用内存，分别是被谁占用了<br>2，亮点2：可以巧妙的利用Redis的底层数据结构特性，降低资源消耗<br>3，要点1： Simple Dynamic String结构体（<br>                  buf：字节数组，为了表示字节结束，会在结尾增加“\0”<br>                  len： 占4个字节，表示buf的已用长度<br>                  alloc：占4个字节，表示buf实际分配的长度，一般大于len）<br><br>4，要点2： RedisObject 结构体（<br>                   元数据：8字节（用于记录最后一次访问时间，被引用次数。。。）<br>                   指针：8字节，指向具体数据类型的实际数据所在 ）<br><br>5，要点3：dicEntry 结构体（    <br>                  key：8个字节指针，指向key<br>                  value：8个字节指针，指向value<br>                  next：指向下一个dicEntry）<br>6，要点4：ziplist(压缩列表)（<br>                 zlbytes：在表头，表示列表长度<br>                 zltail：在表头，表示列尾偏移量<br>                 zllen：在表头，表示列表中<br>                 entry：保存数据对象模型<br>                 zlend：在表尾，表示列表结束）<br>entry：（<br>              prev_len：表示一个entry的长度，有两种取值方式：1字节或5字节。<br>                     1字节表示一个entry小于254字节，255是zlend的默认值，所以不使用。<br>              len：表示自身长度，4字节<br>              encodeing：表示编码方式，1字节<br>              content：保存实际数据）<br><br>5，要点4：String类型的内存空间消耗<br>①，保存Long类型时，指针直接保存整数数据值，可以节省空间开销（被称为：int编码）<br>②，保存字符串，且不大于44字节时，RedisObject的元数据，指针和SDS是连续的，可以避免内存碎片（被称为：embstr编码）<br>③，当保存的字符串大于44字节时，SDS的数据量变多，Redis会给SDS分配独立的空间，并用指针指向SDS结构（被称为：raw编码）<br>④，Redis使用一个全局哈希表保存所以键值对，哈希表的每一项都是一个dicEntry，每个dicEntry占用32字节空间<br>⑤，dicEntry自身24字节，但会占用32字节空间，是因为Redis使用了内存分配库jemalloc。<br>                    jemalloc在分配内存时，会根据申请的字节数N，找一个比N大，但最接近N的2的幂次数作为分配空间，这样可以减少频繁分配内存的次数<br><br>4，要点5：使用什么数据结构可以节省内存？<br>①， 压缩列表，是一种非常节省内存的数据结构，因为他使用连续的内存空间保存数据，不需要额外的指针进行连接<br>②，Redis基于压缩列表实现List，Hash，Sorted Set集合类型，最大的好处是节省了dicEntry开销<br><br>5，要点6：如何使用集合类型保存键值对？<br>①，Hash类型设置了用压缩列表保存数据时的两个阀值，一旦超过就会将压缩列表转为哈希表，且不可回退<br>②，hash-max-ziplist-entries：表示用压缩列表保存哈希集合中的最大元素个数 <br>③，hash-max-ziplist-value：表示用压缩列表保存时，哈希集合中单个元素的最大长度<br><br>四，对于作者所讲，我有哪些发散性思考？<br>    看了老师讲解，做了笔记，又看了黄建宏写的《Redis 设计与实现》<br>有这样的讲解： <br>        当哈希对象可以同时满足以下两个条件时， 哈希对象使用 ziplist 编码：<br>	1. 哈希对象保存的所有键值对的键和值的字符串长度都小于 64 字节；<br>	2. 哈希对象保存的键值对数量小于 512 个；<br><br>五，在将来的哪些场景中，我能够使用它？<br>    这次学习Redis数据结构特性有了更多了解，在以后可以更加有信心根据业务需要，选取特定的数据结构<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-09 12:34:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/08/22/ac1b8a34.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>super BB💨🐷</span>
  </div>
  <div class="_2_QraFYR_0">老师，我之前看到《redis设计与实现》中提出SDS 的结构体的中没有alloc字段，书中的提到的是free,用来表示buf数组未使用的字节长度</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 学习的很仔细！<br><br>《Redis设计与实现》这本书分析的代码是Redis3.0的源码，在Redis3.0.4源码中，SDS结构体里还是用的free表示未使用空间。<br><br>但是应该差不多是Redis3.2.0开始，SDS结构体开始使用alloc字段了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-11 10:05:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/7a/0a/0ce5c232.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>吕</span>
  </div>
  <div class="_2_QraFYR_0">我有一个疑惑，老师，文中的案例，这么大的数据量，为什么采用redis这种内存数据库来存储数据么呢，感觉它的业务场景还是不很清楚？直接采用mysql存储会有什么问题么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这是个好问题。<br><br>其实这个图片ID和存储对象ID对应关系的存储，就是用在分布式存储系统中的一个小的元数据服务，访问模式也比较简单，key-value的PUT、GET就行，但是要求请求响应快。Redis很轻量级，而且速度也快，所以用的Redis。<br><br>MySQL用在这个场景中显得有些太重了，这个场景里面没有关系模型，也没有事务需求和复杂查询，上MySQL不太需要。图片数量再增加时，MySQL的表就太大了，插入效率会降低。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-01 07:45:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/d8/17/367943f4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>吴永祺</span>
  </div>
  <div class="_2_QraFYR_0">hset 1101000 060 3302000080 操作为何只占用16字节？哈希键（060）、值（3302000080）两者各占用一个entry，按文中介绍，应该至少占用28字节。<br><br>其中原因，我认为很可能是文中对ziplist entry的介绍有误，参考下面GitHub文章，entry中并没有len字段，entry长度由encoding表示。所以例子中虽然创建两个entry，但总长度是小于16的。<br><br>参考：https:&#47;&#47;github.com&#47;zpoint&#47;Redis-Internals&#47;blob&#47;5.0&#47;Object&#47;hash&#47;hash_cn.md</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-14 16:14:58</div>
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
  <div class="_2_QraFYR_0">hset 1101000 060 3302000080<br>这条记录只消耗 16 字节没明白，压缩列表保存一个对象需要 14 字节，060、3302000080 都需要保存，那应该至少大于 28 字节</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-01 14:36:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/9b/6e/edd2da0c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>蓝魔丶</span>
  </div>
  <div class="_2_QraFYR_0">老师，测试环境：redis5.0.4<br>1.实践采用String方案：set 1101000052 3301000051，查看内存增加的72，而不是64，是为什么？<br>2.实践采用Hash方案：hset 1101000 060 3302000080 查看内存增加88，再次添加key-value，才是满足增加16</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-02 16:39:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/28/8f/f4d14c03.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Hm_</span>
  </div>
  <div class="_2_QraFYR_0">老师有个地方不是很理解，文中讲到String需要dictEntry来保存键值关系，那么hash结构最外层的那个key没有dictEntry来维护吗？因为我记得之前讲得Redis是用一个大的hash来维护所有的键值对的，所以感觉这和dictEntry所占用的内存是一样的吧</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: dictEntry是Redis全局哈希表中的表项，包含了key和value的指针，这里的value可以是String，List，Hash等数据类型。<br><br>你说的hash结构最外层的key，如果是指全局哈希表中的key的话，指向key的指针是已经包含在dictEntry这个结构中了，同时key本身的数据结构是RedisObject。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-14 22:37:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1c/f1/e5/c8f4e000.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>aworker</span>
  </div>
  <div class="_2_QraFYR_0">看好多小伙伴里面对embstr的临界点是44字节的算法有疑问，给大家解释下：<br><br>首先说下redisObject的数据结构包含如下字段：<br>type：表明redisobjct的类型如果string，hash，set等，占4字节。<br>encoding：编码类型，占4字节。<br>lru:记录此object最近被访问的时间数据和过期逻辑有关，4字节。<br>refcount:记录指向此object的指针数量，用来表示不同指针相同数据的情况，4个字节。<br>ptr：真正指向具体数据的指针，8字节。<br><br>在3.2版本以前的sds结构如下：<br>len：表示sds中buf已经使用的长度，4个字节。<br>free：表示sds中buf没有使用的长度，4个字节。<br>buf：数组，真正存储数据的地方，会有1字节的‘\0’，表示数据的结尾。<br><br>如果想让string类型用embstr编码那么需要满足如下条件：<br><br>64-16（redisObjct除去ptr后最小使用的内存）-（4+4+1）（sds最小内存需求）=39 。<br><br>redis 3.2及其以后的版本sds会根据实际使用的buf长度，采用不同的sds对象表示，这里只说下小于等64字节的sds对象结构：<br>len：表示buf的使用长度，1字节。<br>alloc：表示分配给buf的总长度，1字节。<br>flag：表示具体的sds类型，1个字节。<br>buf：真正存储数据的地方，肯定有1字节的‘\0’表示结束符。<br><br>如果想让redis3.2以后的版本使用embstr编码的string需要buf满足的最大值为：<br>64-16（redisObject最小值）-（1+1+1+1）（sds最小用的内存）= 44字节。<br><br>这里需要澄清一点：相较于raw编码，当redis采用 embstr编码的时候，redis会吧redisObjct的元数据和sds连续存在，这就节约了ptr指针的内存使用，这也是redis要分embstr和raw的原因。老师的图中embstr编码也有8字节的指针，这个应该是不准确的。<br><br>随着版本升级也能看出redis开发者对高效数据结构的极致追求：在3.2版本以前的sds中用4个字节表示buf的已用长度和未使用长度，但对于embstr编码的sds是有些浪费的，因为buf最大值是39字节，1字节就可以表示255的长度了，所以3.2以后的embstr编码的sds的 len和alloc都是一个字节。<br><br><br><br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-06 18:34:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/dd/9b/0bc44a78.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>yyl</span>
  </div>
  <div class="_2_QraFYR_0">“在节省内存空间方面，哈希表就没有压缩列表那么高效了”<br>在内存空间的开销上，也许哈希表没有压缩列表高效<br>但是哈希表的查询效率，要比压缩列表高。<br>在对查询效率高的场景中，可以考虑空间换时间<br><br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 其实，在Redis的设计和使用上，是一个典型的“系统”思维，也就是权衡（trade-off），根据自己的业务场景、数据量、访问特征，来进行选择。<br>我们自己做系统研发，这是个核心思想 ：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-03 22:10:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/36/2c/8bd4be3a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小喵喵</span>
  </div>
  <div class="_2_QraFYR_0">老师，请教下，这样拆分的话，如何重复了咋办呢？<br>以图片 ID 1101000060 和图片存储对象 ID 3302000080 为例，我们可以把图片 ID 的前 7 位（1101000）作为 Hash 类型的键，把图片 ID 的最后 3 位（060）和图片存储对象 ID 分别作为 Hash 类型值中的 key 和 value。<br>比如：两张图片分别为：图片 ID 1101000060 图片存储对象 ID 3302000080；<br>                                     图片 ID 1101001060 图片存储对象 ID 3302000081<br>这个时候最后 3 位（060）的key是冲突的的，但是它的图片存储对象 ID不同。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我们是会把图片 ID 的前7位作为键值对的key，Hash集合是键值对的value，在你举的例子中，图片ID 1101000060和1101001060。它们的前7位分别是1101000和1101001，对应了两个键值对。所以，它们图片ID的后3位虽然相同，都是060，也是在两个Hash集合中的，不会冲突的。<br><br>你看看是不是呢？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-01 10:43:59</div>
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
  <div class="_2_QraFYR_0">虽然压缩列表可以节约内存，但是set和get的时间复杂度为O(N)，一个时间换空间的方法。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-02 22:07:17</div>
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
  <div class="_2_QraFYR_0">使用 http:&#47;&#47;www.redis.cn&#47;redis_memory&#47; 这个网站来计算 文章中 一亿张图片ID消耗的内存， 为什么得出来 9269.71M,  9点多个 G呢？ 1亿个 string , string 的 key 和 value 分别是 8个 字节<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-01 20:28:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2c/6b/6c/3e80afaf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>HappyHasson</span>
  </div>
  <div class="_2_QraFYR_0">hset 1101000 060 3302000080  为什么这条语句执行之后内存增加了16B？<br><br>老师的前提是执行命令之前已经有了hash key 1101000。然后插入fieldkey:060   fieldvalue:3302000080<br>fieldkey和fieldvalue各分配一个ziplist entry，hset时，会调用ziplistPush函数先把fieldkey放到ziplist表尾，然后再放fieldvalue。之所以是16字节，是老师讲解的有点问题。ziplist entry包含三个字段previous_entry_length、encoding、content。没有老师说的len这个固定4字节的字段<br>previous_entry_length取值规则：https:&#47;&#47;github.com&#47;zpoint&#47;Redis-Internals&#47;blob&#47;5.0&#47;Object&#47;hash&#47;prevlen.png<br>encoding取值规则：https:&#47;&#47;github.com&#47;zpoint&#47;Redis-Internals&#47;blob&#47;5.0&#47;Object&#47;hash&#47;encoding.png<br>所以这个命令的fieldkey占用字节：1+1+1=3、fieldvalue占用字节：1+1+8(64位整数表示3302000080)</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-03 02:10:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLzvaL724GwtzZ5mcldUnlicicSlI8BXL9icRZbUOB10qjRMlmog7UTvwxSBHXagnPGGR1BYdjWcGGSg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>wwj</span>
  </div>
  <div class="_2_QraFYR_0">这样是不是有点本位倒置了，缓存本来就是以空间换时间的，这样节省了空间，但是时间复杂度也上去了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-27 17:13:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/e8/55/92f82281.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>MClink</span>
  </div>
  <div class="_2_QraFYR_0">老师，底层数据结构的转换是怎么实现的呢？是单纯的开一个新的数据结构再把数据复制过去吗？再释放之前的数据结构的内存，复制过程中有修改值的话要怎么处理，复制过程中不就两倍内存消耗了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-31 08:24:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/fe/83/df562574.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>慎独明强</span>
  </div>
  <div class="_2_QraFYR_0">看了Redis设计与实现，有讲SDS这一块，对于老师分析的内容，自己心里有印象，再结合老师今天的实践案例，前面的知识还没有吃透<br>😂😂</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-31 07:59:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/45/ab/7dec2448.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我不用网名</span>
  </div>
  <div class="_2_QraFYR_0">看了很久, 一直没有静下心来仔细品味, 今天又重新将课程内容梳理了一遍,有几个问题, 希望老师能解答一下:<br>1.  redis把sds使用raw编码还是embstr编码的临界值是44, 这个44是如何计算出来的呢? 按文档中sds len(4) + alloc(4) + 1(\0)  + redisObject(元素数据8+指针8) = 25, jemalloc 将超过64字节的使用raw编码, 这样的话, buff 的最大值 64-25=39.    这也是我看网上其它资料时写的reids.3.2版本之前使用的值. 3.2及以后使用44.  老师文档中sds各字段的大小是不是标错了?<br>2.  hash类型使用ziplist存储数据时, key&#47;value的映射关系是发何保持的呢?<br>     我自己有如下猜想,存储结构是不是这样?<br>     zlbytes zltail zllen key1 value1 key2 value2 ...  zlend<br>     如果是这样存储的话, 又有以下两个问题困绕着我:<br>     在hash中通过某个key获取对应value时,需要遍历整个压缩列表吧. 这会不会有性能影响?<br>     key与value都紧挨着存储, entry里面并没有字段用于标记该entry是key还是value, 假如 key与value是相同的字段串时, 即有两个相同的entry, reids如何识别哪个是key,哪个是value呢?<br>     对于后面一个问题,我自己可以猜想一下的, 就是key与value是成对出现的, key 永远是在寄数位 value永远是在偶数位. 这样也可以分辨, 如果是我来实现的话,可能会这样. 但我不懂c,没看过源码, 请老师专业解答我才放心.<br>    感谢!<br>     </div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-19 21:39:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/28/4a/a7/ab7998b1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>zachary</span>
  </div>
  <div class="_2_QraFYR_0">通过查看redis的源码，sds数据结构这块老师讲的不是很对。可能跟redis的版本有关吧。redis在高版本对sds做了优化，sds将不再直接使用结构体。sds底层通过sdshdr_5, sdshdr_8, sdshdr_16, sdshdr_32, sdshdr_64来实现。这么做的目的明显是为了更加节省空间。以下是源码，版本4.0.9， 文件sds.h。<br><br>typedef char *sds;<br><br>&#47;* Note: sdshdr5 is never used, we just access the flags byte directly.<br> * However is here to document the layout of type 5 SDS strings. *&#47;<br>struct __attribute__ ((__packed__)) sdshdr5 {<br>    unsigned char flags; &#47;* 3 lsb of type, and 5 msb of string length *&#47;<br>    char buf[];<br>};<br>struct __attribute__ ((__packed__)) sdshdr8 {<br>    uint8_t len; &#47;* used *&#47;<br>    uint8_t alloc; &#47;* excluding the header and null terminator *&#47;<br>    unsigned char flags; &#47;* 3 lsb of type, 5 unused bits *&#47;<br>    char buf[];<br>};<br>struct __attribute__ ((__packed__)) sdshdr16 {<br>    uint16_t len; &#47;* used *&#47;<br>    uint16_t alloc; &#47;* excluding the header and null terminator *&#47;<br>    unsigned char flags; &#47;* 3 lsb of type, 5 unused bits *&#47;<br>    char buf[];<br>};<br>struct __attribute__ ((__packed__)) sdshdr32 {<br>    uint32_t len; &#47;* used *&#47;<br>    uint32_t alloc; &#47;* excluding the header and null terminator *&#47;<br>    unsigned char flags; &#47;* 3 lsb of type, 5 unused bits *&#47;<br>    char buf[];<br>};<br>struct __attribute__ ((__packed__)) sdshdr64 {<br>    uint64_t len; &#47;* used *&#47;<br>    uint64_t alloc; &#47;* excluding the header and null terminator *&#47;<br>    unsigned char flags; &#47;* 3 lsb of type, 5 unused bits *&#47;<br>    char buf[];<br>};<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-25 18:44:17</div>
  </div>
</div>
</div>
</li>
</ul>