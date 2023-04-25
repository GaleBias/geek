<audio title="02 _ 数据结构：快速的Redis有哪些慢操作？" src="https://static001.geekbang.org/resource/audio/64/a3/64793ee06a6fe2cdc1023189f5f538a3.mp3" controls="controls"></audio> 
<p>你好，我是蒋德钧。</p><p>一提到Redis，我们的脑子里马上就会出现一个词：“快。”但是你有没有想过，Redis的快，到底是快在哪里呢？实际上，这里有一个重要的表现：它接收到一个键值对操作后，能以<strong>微秒级别</strong>的速度找到数据，并快速完成操作。</p><p>数据库这么多，为啥Redis能有这么突出的表现呢？一方面，这是因为它是内存数据库，所有操作都在内存上完成，内存的访问速度本身就很快。另一方面，这要归功于它的数据结构。这是因为，键值对是按一定的数据结构来组织的，操作键值对最终就是对数据结构进行增删改查操作，所以高效的数据结构是Redis快速处理数据的基础。这节课，我就来和你聊聊数据结构。</p><p>说到这儿，你肯定会说：“这个我知道，不就是String（字符串）、List（列表）、Hash（哈希）、Set（集合）和Sorted Set（有序集合）吗？”其实，这些只是Redis键值对中值的数据类型，也就是数据的保存形式。而这里，我们说的数据结构，是要去看看它们的底层实现。</p><p>简单来说，底层数据结构一共有6种，分别是简单动态字符串、双向链表、压缩列表、哈希表、跳表和整数数组。它们和数据类型的对应关系如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/82/01/8219f7yy651e566d47cc9f661b399f01.jpg?wh=4000*1188" alt=""></p><p>可以看到，String类型的底层实现只有一种数据结构，也就是简单动态字符串。而List、Hash、Set和Sorted Set这四种数据类型，都有两种底层实现结构。通常情况下，我们会把这四种类型称为集合类型，它们的特点是<strong>一个键对应了一个集合的数据</strong>。</p><!-- [[[read_end]]] --><p>看到这里，其实有些问题已经值得我们去考虑了：</p><ul>
<li>这些数据结构都是值的底层实现，键和值本身之间用什么结构组织？</li>
<li>为什么集合类型有那么多的底层结构，它们都是怎么组织数据的，都很快吗？</li>
<li>什么是简单动态字符串，和常用的字符串是一回事吗？</li>
</ul><p>接下来，我就和你聊聊前两个问题。这样，你不仅可以知道Redis“快”的基本原理，还可以借此理解Redis中有哪些潜在的“慢操作”，最大化Redis的性能优势。而关于简单动态字符串，我会在后面的课程中再和你讨论。</p><p>我们先来看看键和值之间是用什么结构组织的。</p><h2>键和值用什么结构组织？</h2><p>为了实现从键到值的快速访问，Redis使用了一个哈希表来保存所有键值对。</p><p>一个哈希表，其实就是一个数组，数组的每个元素称为一个哈希桶。所以，我们常说，一个哈希表是由多个哈希桶组成的，每个哈希桶中保存了键值对数据。</p><p>看到这里，你可能会问了：“如果值是集合类型的话，作为数组元素的哈希桶怎么来保存呢？”其实，哈希桶中的元素保存的并不是值本身，而是指向具体值的指针。这也就是说，不管值是String，还是集合类型，哈希桶中的元素都是指向它们的指针。</p><p>在下图中，可以看到，哈希桶中的entry元素中保存了<code>*key</code>和<code>*value</code>指针，分别指向了实际的键和值，这样一来，即使值是一个集合，也可以通过<code>*value</code>指针被查找到。</p><p><img src="https://static001.geekbang.org/resource/image/1c/5f/1cc8eaed5d1ca4e3cdbaa5a3d48dfb5f.jpg?wh=1773*875" alt=""></p><p>因为这个哈希表保存了所有的键值对，所以，我也把它称为<strong>全局哈希表</strong>。哈希表的最大好处很明显，就是让我们可以用O(1)的时间复杂度来快速查找到键值对——我们只需要计算键的哈希值，就可以知道它所对应的哈希桶位置，然后就可以访问相应的entry元素。</p><p>你看，这个查找过程主要依赖于哈希计算，和数据量的多少并没有直接关系。也就是说，不管哈希表里有10万个键还是100万个键，我们只需要一次计算就能找到相应的键。</p><p>但是，如果你只是了解了哈希表的O(1)复杂度和快速查找特性，那么，当你往Redis中写入大量数据后，就可能发现操作有时候会突然变慢了。这其实是因为你忽略了一个潜在的风险点，那就是<strong>哈希表的冲突问题和rehash可能带来的操作阻塞。</strong></p><h3>为什么哈希表操作变慢了？</h3><p>当你往哈希表中写入更多数据时，哈希冲突是不可避免的问题。这里的哈希冲突，也就是指，两个key的哈希值和哈希桶计算对应关系时，正好落在了同一个哈希桶中。</p><p>毕竟，哈希桶的个数通常要少于key的数量，这也就是说，难免会有一些key的哈希值对应到了同一个哈希桶中。</p><p>Redis解决哈希冲突的方式，就是链式哈希。链式哈希也很容易理解，就是指<strong>同一个哈希桶中的多个元素用一个链表来保存，它们之间依次用指针连接</strong>。</p><p>如下图所示：entry1、entry2和entry3都需要保存在哈希桶3中，导致了哈希冲突。此时，entry1元素会通过一个<code>*next</code>指针指向entry2，同样，entry2也会通过<code>*next</code>指针指向entry3。这样一来，即使哈希桶3中的元素有100个，我们也可以通过entry元素中的指针，把它们连起来。这就形成了一个链表，也叫作哈希冲突链。</p><p><img src="https://static001.geekbang.org/resource/image/8a/28/8ac4cc6cf94968a502161f85d072e428.jpg?wh=1233*1093" alt=""></p><p>但是，这里依然存在一个问题，哈希冲突链上的元素只能通过指针逐一查找再操作。如果哈希表里写入的数据越来越多，哈希冲突可能也会越来越多，这就会导致某些哈希冲突链过长，进而导致这个链上的元素查找耗时长，效率降低。对于追求“快”的Redis来说，这是不太能接受的。</p><p>所以，Redis会对哈希表做rehash操作。rehash也就是增加现有的哈希桶数量，让逐渐增多的entry元素能在更多的桶之间分散保存，减少单个桶中的元素数量，从而减少单个桶中的冲突。那具体怎么做呢？</p><p>其实，为了使rehash操作更高效，Redis默认使用了两个全局哈希表：哈希表1和哈希表2。一开始，当你刚插入数据时，默认使用哈希表1，此时的哈希表2并没有被分配空间。随着数据逐步增多，Redis开始执行rehash，这个过程分为三步：</p><ol>
<li>给哈希表2分配更大的空间，例如是当前哈希表1大小的两倍；</li>
<li>把哈希表1中的数据重新映射并拷贝到哈希表2中；</li>
<li>释放哈希表1的空间。</li>
</ol><p>到此，我们就可以从哈希表1切换到哈希表2，用增大的哈希表2保存更多数据，而原来的哈希表1留作下一次rehash扩容备用。</p><p>这个过程看似简单，但是第二步涉及大量的数据拷贝，如果一次性把哈希表1中的数据都迁移完，会造成Redis线程阻塞，无法服务其他请求。此时，Redis就无法快速访问数据了。</p><p>为了避免这个问题，Redis采用了<strong>渐进式rehash</strong>。</p><p>简单来说就是在第二步拷贝数据时，Redis仍然正常处理客户端请求，每处理一个请求时，从哈希表1中的第一个索引位置开始，顺带着将这个索引位置上的所有entries拷贝到哈希表2中；等处理下一个请求时，再顺带拷贝哈希表1中的下一个索引位置的entries。如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/73/0c/73fb212d0b0928d96a0d7d6ayy76da0c.jpg?wh=1770*1079" alt="" title="渐进式rehash"></p><p>这样就巧妙地把一次性大量拷贝的开销，分摊到了多次处理请求的过程中，避免了耗时操作，保证了数据的快速访问。</p><p>好了，到这里，你应该就能理解，Redis的键和值是怎么通过哈希表组织的了。对于String类型来说，找到哈希桶就能直接增删改查了，所以，哈希表的O(1)操作复杂度也就是它的复杂度了。</p><p>但是，对于集合类型来说，即使找到哈希桶了，还要在集合中再进一步操作。接下来，我们来看集合类型的操作效率又是怎样的。</p><h2>集合数据操作效率</h2><p>和String类型不同，一个集合类型的值，第一步是通过全局哈希表找到对应的哈希桶位置，第二步是在集合中再增删改查。那么，集合的操作效率和哪些因素相关呢？</p><p>首先，与集合的底层数据结构有关。例如，使用哈希表实现的集合，要比使用链表实现的集合访问效率更高。其次，操作效率和这些操作本身的执行特点有关，比如读写一个元素的操作要比读写所有元素的效率高。</p><p>接下来，我们就分别聊聊集合类型的底层数据结构和操作复杂度。</p><h3>有哪些底层数据结构？</h3><p>刚才，我也和你介绍过，集合类型的底层数据结构主要有5种：整数数组、双向链表、哈希表、压缩列表和跳表。</p><p>其中，哈希表的操作特点我们刚刚已经学过了；整数数组和双向链表也很常见，它们的操作特征都是顺序读写，也就是通过数组下标或者链表的指针逐个元素访问，操作复杂度基本是O(N)，操作效率比较低；压缩列表和跳表我们平时接触得可能不多，但它们也是Redis重要的数据结构，所以我来重点解释一下。</p><p>压缩列表实际上类似于一个数组，数组中的每一个元素都对应保存一个数据。和数组不同的是，压缩列表在表头有三个字段zlbytes、zltail和zllen，分别表示列表长度、列表尾的偏移量和列表中的entry个数；压缩列表在表尾还有一个zlend，表示列表结束。</p><p><img src="https://static001.geekbang.org/resource/image/95/a0/9587e483f6ea82f560ff10484aaca4a0.jpg?wh=1583*315" alt=""></p><p>在压缩列表中，如果我们要查找定位第一个元素和最后一个元素，可以通过表头三个字段的长度直接定位，复杂度是O(1)。而查找其他元素时，就没有这么高效了，只能逐个查找，此时的复杂度就是O(N)了。</p><p>我们再来看下跳表。</p><p>有序链表只能逐一查找元素，导致操作起来非常缓慢，于是就出现了跳表。具体来说，跳表在链表的基础上，<strong>增加了多级索引，通过索引位置的几个跳转，实现数据的快速定位</strong>，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/1e/b4/1eca7135d38de2yy16681c2bbc4f3fb4.jpg?wh=1878*1126" alt="" title="跳表的快速查找过程"></p><p>如果我们要在链表中查找33这个元素，只能从头开始遍历链表，查找6次，直到找到33为止。此时，复杂度是O(N)，查找效率很低。</p><p>为了提高查找速度，我们来增加一级索引：从第一个元素开始，每两个元素选一个出来作为索引。这些索引再通过指针指向原始的链表。例如，从前两个元素中抽取元素1作为一级索引，从第三、四个元素中抽取元素11作为一级索引。此时，我们只需要4次查找就能定位到元素33了。</p><p>如果我们还想再快，可以再增加二级索引：从一级索引中，再抽取部分元素作为二级索引。例如，从一级索引中抽取1、27、100作为二级索引，二级索引指向一级索引。这样，我们只需要3次查找，就能定位到元素33了。</p><p>可以看到，这个查找过程就是在多级索引上跳来跳去，最后定位到元素。这也正好符合“跳”表的叫法。当数据量很大时，跳表的查找复杂度就是O(logN)。</p><p>好了，我们现在可以按照查找的时间复杂度给这些数据结构分下类了：</p><p><img src="https://static001.geekbang.org/resource/image/fb/f0/fb7e3612ddee8a0ea49b7c40673a0cf0.jpg?wh=1332*750" alt=""></p><h3>不同操作的复杂度</h3><p>集合类型的操作类型很多，有读写单个集合元素的，例如HGET、HSET，也有操作多个元素的，例如SADD，还有对整个集合进行遍历操作的，例如SMEMBERS。这么多操作，它们的复杂度也各不相同。而复杂度的高低又是我们选择集合类型的重要依据。</p><p>我总结了一个“四句口诀”，希望能帮助你快速记住集合常见操作的复杂度。这样你在使用过程中，就可以提前规避高复杂度操作了。</p><ul>
<li>单元素操作是基础；</li>
<li>范围操作非常耗时；</li>
<li>统计操作通常高效；</li>
<li>例外情况只有几个。</li>
</ul><p>第一，<strong>单元素操作，是指每一种集合类型对单个数据实现的增删改查操作</strong>。例如，Hash类型的HGET、HSET和HDEL，Set类型的SADD、SREM、SRANDMEMBER等。这些操作的复杂度由集合采用的数据结构决定，例如，HGET、HSET和HDEL是对哈希表做操作，所以它们的复杂度都是O(1)；Set类型用哈希表作为底层数据结构时，它的SADD、SREM、SRANDMEMBER复杂度也是O(1)。</p><p>这里，有个地方你需要注意一下，集合类型支持同时对多个元素进行增删改查，例如Hash类型的HMGET和HMSET，Set类型的SADD也支持同时增加多个元素。此时，这些操作的复杂度，就是由单个元素操作复杂度和元素个数决定的。例如，HMSET增加M个元素时，复杂度就从O(1)变成O(M)了。</p><p>第二，<strong>范围操作，是指集合类型中的遍历操作，可以返回集合中的所有数据</strong>，比如Hash类型的HGETALL和Set类型的SMEMBERS，或者返回一个范围内的部分数据，比如List类型的LRANGE和ZSet类型的ZRANGE。<strong>这类操作的复杂度一般是O(N)，比较耗时，我们应该尽量避免</strong>。</p><p>不过，Redis从2.8版本开始提供了SCAN系列操作（包括HSCAN，SSCAN和ZSCAN），这类操作实现了渐进式遍历，每次只返回有限数量的数据。这样一来，相比于HGETALL、SMEMBERS这类操作来说，就避免了一次性返回所有元素而导致的Redis阻塞。</p><p>第三，统计操作，是指<strong>集合类型对集合中所有元素个数的记录</strong>，例如LLEN和SCARD。这类操作复杂度只有O(1)，这是因为当集合类型采用压缩列表、双向链表、整数数组这些数据结构时，这些结构中专门记录了元素的个数统计，因此可以高效地完成相关操作。</p><p>第四，例外情况，是指某些数据结构的特殊记录，例如<strong>压缩列表和双向链表都会记录表头和表尾的偏移量</strong>。这样一来，对于List类型的LPOP、RPOP、LPUSH、RPUSH这四个操作来说，它们是在列表的头尾增删元素，这就可以通过偏移量直接定位，所以它们的复杂度也只有O(1)，可以实现快速操作。</p><h2>小结</h2><p>这节课，我们学习了Redis的底层数据结构，这既包括了Redis中用来保存每个键和值的全局哈希表结构，也包括了支持集合类型实现的双向链表、压缩列表、整数数组、哈希表和跳表这五大底层结构。</p><p>Redis之所以能快速操作键值对，一方面是因为O(1)复杂度的哈希表被广泛使用，包括String、Hash和Set，它们的操作复杂度基本由哈希表决定，另一方面，Sorted Set也采用了O(logN)复杂度的跳表。不过，集合类型的范围操作，因为要遍历底层数据结构，复杂度通常是O(N)。这里，我的建议是：<strong>用其他命令来替代</strong>，例如可以用SCAN来代替，避免在Redis内部产生费时的全集合遍历操作。</p><p>当然，我们不能忘了复杂度较高的List类型，它的两种底层实现结构：双向链表和压缩列表的操作复杂度都是O(N)。因此，我的建议是：<strong>因地制宜地使用List类型</strong>。例如，既然它的POP/PUSH效率很高，那么就将它主要用于FIFO队列场景，而不是作为一个可以随机读写的集合。</p><p>Redis数据类型丰富，每个类型的操作繁多，我们通常无法一下子记住所有操作的复杂度。所以，最好的办法就是<strong>掌握原理，以不变应万变</strong>。这里，你可以看到，一旦掌握了数据结构基本原理，你可以从原理上推断不同操作的复杂度，即使这个操作你不一定熟悉。这样一来，你不用死记硬背，也能快速合理地做出选择了。</p><h2>每课一问</h2><p>整数数组和压缩列表在查找时间复杂度方面并没有很大的优势，那为什么Redis还会把它们作为底层数据结构呢？</p><p>数据结构是了解Redis性能的必修课，如果你身边还有不太清楚数据结构的朋友，欢迎你把今天的内容分享给他/她，期待你在留言区和我交流讨论。</p>
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
  <div class="_2_QraFYR_0">两方面原因：<br><br>1、内存利用率，数组和压缩列表都是非常紧凑的数据结构，它比链表占用的内存要更少。Redis是内存数据库，大量数据存到内存中，此时需要做尽可能的优化，提高内存的利用率。<br><br>2、数组对CPU高速缓存支持更友好，所以Redis在设计时，集合数据元素较少情况下，默认采用内存紧凑排列的方式存储，同时利用CPU高速缓存不会降低访问速度。当数据元素超过设定阈值后，避免查询时间复杂度太高，转为哈希和跳表数据结构存储，保证查询效率。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 回答的很好！<br><br>我再提个小问题，如果在数组上是随机访问，对CPU高速缓存还友好不？：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-04 14:04:19</div>
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
  <div class="_2_QraFYR_0">一，作者讲了什么？<br>    1，Redis的底层数据结构<br><br>二，作者是怎么把这事给讲明白的？<br>    1，讲了Redis的数据结构：数据的保存形式与底层数据结构<br>    2，由数据结构的异同点，引出数据操作的快慢原因<br>    <br>三，为了讲明白，作者讲了哪些要点？有哪些亮点？<br>    1，亮点1：string，list，set，hast,sortset都只是数据的保存形式，底层的数据结构是：简单动态字符串，双向链表，压缩列表，哈希表，跳表，整数数组<br>    2，亮点2：Redis使用了一个哈希表保存所有的键值对<br>    3，要点1：五种数据形式的底层实现<br>            a，string：简单动态字符串<br>            b，list：双向链表，压缩列表<br>            c，hash：压缩列表，哈希表<br>            d，Sorted Set：压缩列表，跳表<br>            e，set：哈希表，整数数组<br>    4，要点2：List ,hash，set ,sorted set被统称为集合类型，一个键对应了一个集合的数据<br>    5，要点3：集合类型的键和值之间的结构组织<br>            a：Redis使用一个哈希表保存所有键值对，一个哈希表实则是一个数组，数组的每个元素称为哈希桶。<br>            b：哈希桶中的元素保存的不是值的本身，而是指向具体值的指针<br>    6，要点4：哈希冲突解决<br>            a：Redis的hash表是全局的，所以当写入大量的key时，将会带来哈希冲突，已经rehash可能带来的操作阻塞<br>            b：Redis解决hash冲突的方式，是链式哈希：同一个哈希桶中的多个元素用一个链表来保存<br>            c：当哈希冲突链过长时，Redis会对hash表进行rehash操作。rehash就是增加现有的hash桶数量，分散entry元素。<br>    7，要点5：rehash机制<br>            a：为了使rehash操作更高效，Redis默认使用了两个全局哈希表：哈希表1和哈希表2，起始时hash2没有分配空间<br>            b：随着数据增多，Redis执行分三步执行rehash;<br>                1，给hash2分配更大的内存空间，如是hash1的两倍<br>                2，把hash1中的数据重新映射并拷贝到哈希表2中<br>                3，释放hash1的空间<br>    8，要点6：渐进式rehash<br>            a：由于步骤2重新映射非常耗时，会阻塞redis<br>            b：讲集中迁移数据，改成每处理一个请求时，就从hash1中的第一个索引位置，顺带将这个索引位置上的所有entries拷贝到hash2中。<br>    9，要点7 ：压缩列表，跳表的特点<br>            a：压缩列表类似于一个数组，不同的是:压缩列表在表头有三个字段zlbytes,zltail和zllen分别表示长度，列表尾的偏移量和列表中的entry的个数，压缩列表尾部还有一个zlend，表示列表结束<br>                所以压缩列表定位第一个和最后一个是O(1),但其他就是O(n)<br>            b：跳表：是在链表的基础上增加了多级索引，通过索引的几次跳转，实现数据快速定位<br><br>四，对于作者所讲，我有哪些发散性思考？<br><br>五，在将来的哪些场景中，我能够用到它？<br><br>六，评论区收获<br>    1，数组和压缩列表可以提升内存利用率，因为他们的数据结构紧凑<br>    2，数组对CPU高速缓存支持友好，当数据元素超过阈值时，会转为hash和跳表，保证查询效率</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-25 08:41:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/34/b6/0feb574b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我的黄金时代</span>
  </div>
  <div class="_2_QraFYR_0">来自互联网 ：因为在进行渐进式 rehash 的过程中， 字典会同时使用 ht[0] 和 ht[1] 两个哈希表， 所以在渐进式 rehash 进行期间， 字典的删除（delete）、查找（find）、更新（update）等操作会在两个哈希表上进行： 比如说， 要在字典里面查找一个键的话， 程序会先在 ht[0] 里面进行查找， 如果没找到的话， 就会继续到 ht[1] 里面进行查找， 诸如此类。<br>另外， 在渐进式 rehash 执行期间， 新添加到字典的键值对一律会被保存到 ht[1] 里面， 而 ht[0] 则不再进行任何添加操作： 这一措施保证了 ht[0] 包含的键值对数量会只减不增， 并随着 rehash 操作的执行而最终变成空表。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-19 19:29:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/d0/69/5dbdc245.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张德</span>
  </div>
  <div class="_2_QraFYR_0">同问  老师  如果渐进式哈希操作  如果有一个value操作很长时间段都没被查到  那渐进式哈希会持续非常长的时间吗   还是会在空闲的时候  也给挪到扩容的hash表里面</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 渐进式rehash执行时，除了根据键值对的操作来进行数据迁移，Redis本身还会有一个定时任务在执行rehash，如果没有键值对操作时，这个定时任务会周期性地（例如每100ms一次）搬移一些数据到新的哈希表中，这样可以缩短整个rehash的过程。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-24 17:05:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/f6/9c/b457a937.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>不能扮演天使</span>
  </div>
  <div class="_2_QraFYR_0">Redis的List底层使用压缩列表本质上是将所有元素紧挨着存储，所以分配的是一块连续的内存空间，虽然数据结构本身没有时间复杂度的优势，但是这样节省空间而且也能避免一些内存碎片；</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞！所以不同的设计选择都是有背后的考虑。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-03 18:15:23</div>
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
  <div class="_2_QraFYR_0">请问老师关于zset的问题，您在文中提到Sorted Set内部实现数据结构是跳表和压缩列表。但是我从zset代码看到这样的注释<br>The elements are added to a hash table mapping Redis objects to scores. At the same time the elements are added to a skip list mapping scores to Redis objects <br>按照注释应该还有hash table来额外存储数据吧，这样在zset里查找单个元素，可以从O(logN)降低为O(1)。不知道我理解的是否正确？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 看得非常仔细，赞一个！<br><br>zset有个ZSCORE的操作，用于返回单个集合member的分数，它的操作复杂度是O(1)，这就是收益于你这看到的hash table。这个hash table保存了集合元素和相应的分数，所以做ZSCORE操作时，直接查这个表就可以，复杂度就降为O(1)了。<br><br>而跳表主要服务范围操作，提供O(logN)的复杂度。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-06 09:46:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/26/34/891dd45b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>宙斯</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，hash表是全局的，这里的全局怎样理解？是‘存放key的数组全局只有一个‘是这样理解吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个全局是指Redis数据库中的所有key和value，是由一个哈希表来索引的。通过在这个哈希表中查询key，就可以找到对应的value。然后根据value的具体类型（例如Hash，Set，List等），再通过value的底层数据结构来读取具体的value数据，例如List通过双向链表来读取数据。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-14 00:29:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/1b/4d/2cc44d9a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>刘忽悠</span>
  </div>
  <div class="_2_QraFYR_0">redis底层的数据压缩搞的很细致，像sds，根据字节长度划分的很细致，另外采用的c99特性的动态数组，对短字符串进行一次性的内存分配；跳表设计的也很秀，简单明了，进行范围查询很方便；dict的扩容没细看，但是看了一下数据结构，应该是为了避免发生扩容的时候出现整体copy；<br>个人觉得老师应该把sds，dict等具体数据结构的源码也贴上</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: dict的渐进式rehash是为了避免扩容时的整体拷贝，这会给内存带来较大压力。<br><br>SDS我们后面还会再聊：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-03 18:35:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/5f/bf/4bd3eb4b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>米 虫</span>
  </div>
  <div class="_2_QraFYR_0">要是加餐中来一偏，一个redis指令的执行过程，那大局观就更深刻了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我们后面的课程会涉及这个过程，再耐心等等哈</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-04 22:30:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/3a/82/1ff83a38.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>牛牛</span>
  </div>
  <div class="_2_QraFYR_0">老师、早安, <br>今天的问题 @Kaito 同学回答的很好、尝试回答下: 数组上随机访问是否对cpu缓存友好 ?<br><br>数组对cpu缓存友好的原因是: cpu预读取一个cache line大小的数据, 数组数据排列紧凑、相同大小空间保存的元素更多, 访问下一个元素时、恰好已经在cpu缓存了. 如果是随机访问、就不能充分利用cpu缓存了, 拿int元素举例: 一个元素4byte, CacheLine 假设64byte, 可以预读取 16个挨着的元素, 如果下次随机访问的元素不在这16个元素里、就需要重新从内存读取了.<br><br><br>想请教几个问题:<br><br>1. rehash是将访问元素所在索引位置上的所有entries都 copy 到 hash表2 ?<br>   如果有索引位置一直没访问到、它会一直留着 hash表1 中 ？<br>   如果是, 再次rehash时这部分没被挪走的索引位置怎么处理 ?<br>   如果不是、那什么时候(时机)触发这部分位置的rehash呢 ?<br><br>2. rehash过程中、内存占用会多于原所占内存的2倍 ?<br>   (ht2的内存是ht1的2倍, 原ht1的空间还未释放)<br>   我记得redis设计实现上说 ht2的内存会与ht1实际使用的键值对的数量有关, 扩容好像是 &gt;= ht1.used * 2 的第一个 2^n; 缩容好像是 &gt;= ht1.used 的第一个 2^n<br>   <br>3. rehash完成之后, hash表1 留作下次rehash备用、但会把占用的内存释放掉, 这么理解对不 ？<br><br>4. rehash时 为什么是 `复制`, 而不是 `移动`, 这个是有什么考虑吗 ？<br>   我的理解: 移动需要释放原空间, 每个元素都单独释放会导致大量的碎片内存、多次释放也比一次释放效率更低. 不知道是不是考虑错了~~~<br>   </div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-13 09:05:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/aa/a5/194613c1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>dingjiayi</span>
  </div>
  <div class="_2_QraFYR_0">老师好，请问 rehash 期间，新的 操作请求（增删改查）到达，redis 是如何处理的？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-05 22:51:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/ce/d7/8168e1bf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小飞同学</span>
  </div>
  <div class="_2_QraFYR_0">小问题:<br>1.rehash的时候为啥存储位置会发生变化？一个对象的hashCode始终是一样的。  还是说rehash是对槽进行取模 <br>2.跳表找元素没太看明白，是二分查找么？怎么感觉找33那个元素不止3次<br>课后一问:<br>因为数组占用内存连续，不需要随机读取。同时碎片化问题也不需要考虑。<br><br>希望得到老师的解答，谢谢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 一个对象保存到哈希桶时，实际是要用它的hash值对哈希桶个数取模的，所以rehash时，桶个数发生了变化，那么对象的存储位置也会有所变化。<br><br>跳表的查找通过不同层的指针来跳转，指针比较多时，类似于二分查找了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-04 22:05:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/25/7f/473d5a77.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>曾轼麟</span>
  </div>
  <div class="_2_QraFYR_0">intset 和 ziplist 如果直接使用确实是时间复杂度上不是很高效，但是结合Redis的使用场景，大部分Redis的数据都是零散碎片化的，通过这两种数据结构可以提高内存利用率，但是为了防止过度使用这两种数据结构Redis其实都设有阈值或者搭配使用的，例如：ziplist是和quicklist一起使用的，在quicklist中ziplist是一个个节点，而且quicklist为每个节点的大小都设置有阈值避免单个节点过大，从而导致性能下降</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-03 23:29:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/75/46/815e56c8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>QFY</span>
  </div>
  <div class="_2_QraFYR_0">老师，关于rehash那里有两个问题：<br>1.redis切换全局hash表是否是在将原有hash表的全部内容拷贝完成后切换<br>2.如果在rehash过程中，已拷贝过的位置后面又有新的冲突值过来了该怎么办</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-06 16:04:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/1b/4d/2cc44d9a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>刘忽悠</span>
  </div>
  <div class="_2_QraFYR_0">至于问题答案，采用压缩列表或者是整数集合，都是数据量比较小的情况，所以一次能够分配到足够大的内存，而压缩列表和整数集合本身的数据结构也是线性的，对cpu的缓存更友好一些，所以真正的执行的时间因为高速缓存的关系，速度更快</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 提个小问题哈，如果压缩列表也是随机跳着访问，对CPU缓存还友好不？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-03 18:41:57</div>
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
  <div class="_2_QraFYR_0">Redis 采用了渐进式 rehash，那么什么时候进行新的全局Hash表的切换呢？当旧的Hash表格的负载因子达到一定值的时候？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-11 21:59:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/48/cf/8c88e6c0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Nemo</span>
  </div>
  <div class="_2_QraFYR_0">各种数据结构访问和修改的时间复杂度：https:&#47;&#47;www.bigocheatsheet.com&#47;</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-07 11:40:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/ea/ff/d1eb00e3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>二哥不再迷茫</span>
  </div>
  <div class="_2_QraFYR_0">跳表快速查询33过程中疑问，<br>一级索引， <br>1， 在一级索引中比较1和33的大小，1小于33，<br>2，继续比较11和33的大小，11比33小，<br>3，继续比较27和33的大小， 27比33小，<br>4，继续比较50和33的大小，50比33大，<br>5，从33位置进入底层数据，比较33和33大小，相等，找到33.<br>5次比较大小，应该是5次查找到33吧。<br>同理二级索引需要5次查找到33。<br>不是应该这样吗？老师</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-15 12:56:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/12/70/10faf04b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Lywane</span>
  </div>
  <div class="_2_QraFYR_0">rehash不仅全局有，单独的值为HASH类型的数据也会有吧</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，哈希表这个结构，整个数据库空间用它来保存键值对，同时HASH类型的键值对也用它作为底层结构。所以，rehash也是两种情况下都有的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-17 09:58:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/82/1a/64ec25ff.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>陈靖</span>
  </div>
  <div class="_2_QraFYR_0">老师的项目，是要把这些数据结构全部撸一遍吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-10 00:18:51</div>
  </div>
</div>
</div>
</li>
</ul>