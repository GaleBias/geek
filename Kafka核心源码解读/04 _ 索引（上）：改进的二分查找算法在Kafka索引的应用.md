<audio title="04 _ 索引（上）：改进的二分查找算法在Kafka索引的应用" src="https://static001.geekbang.org/resource/audio/eb/54/eb42991816f9ddad5d462d8155407354.mp3" controls="controls"></audio> 
<p>你好，我是胡夕。今天，我来带你学习一下Kafka源码中的索引对象，以及改进版二分查找算法（Binary Search Algorithm）在索引中的应用。</p><h2>为什么要阅读索引源码？</h2><p>坦率地说，你在Kafka中直接接触索引或索引文件的场景可能不是很多。索引是一个很神秘的组件，Kafka官方文档也没有怎么提过它。你可能会说，既然这样，我还有必要读索引对象的源码吗？其实是非常有必要的！我给你分享一个真实的例子。</p><p>有一次，我用Kafka的DumpLogSegments类去查看底层日志文件和索引文件的内容时，发现了一个奇怪的现象——查看日志文件的内容不需要sudo权限，而查看索引文件的内容必须要有sudo权限，如下所示：</p><pre><code>  $ sudo ./kafka-run-class.sh kafka.tools.DumpLogSegments  --files ./00000000000000000000.index
    Dumping 00000000000000000000.index
    offset: 0 position: 0
    
    
    $ ./kafka-run-class.sh kafka.tools.DumpLogSegments --files 00000000000000000000.index
    Dumping 00000000000000000000.index
    Exception in thread &quot;main&quot; java.io.FileNotFoundException: 00000000000000000000.index (Permission denied)
    ......
</code></pre><p>看了索引源码之后，我才知道，原来Kafka读取索引文件时使用的打开方式是rw。实际上，读取文件不需要w权限，只要r权限就行了。这显然是Kafka的一个Bug。你看，通过阅读源码，我找到了问题的根本原因，还顺便修复了Kafka的一个问题（<a href="https://issues.apache.org/jira/browse/KAFKA-5104">KAFKA-5104</a>）。</p><p>除了能帮我们解决实际问题之外，索引这个组件的源码还有一个亮点，那就是<strong>它应用了耳熟能详的二分查找算法来快速定位索引项</strong>。关于算法，我一直觉得很遗憾的是，<strong>我们平时太注重算法本身，却忽略了它们在实际场景中的应用</strong>。</p><!-- [[[read_end]]] --><p>比如说，我们学习了太多的排序算法，但是，对于普通的应用开发人员来说，亲自使用这些算法完成编程任务的机会实在太少了。说起数组排序，你可能只记得调用Collections.sort方法了，但它底层应用了什么排序算法，其实并不清楚。</p><p>难得的是，Kafka的索引组件中应用了二分查找算法，而且社区还针对Kafka自身的特点对其进行了改良。这难道不值得我们好好学上一学吗？！话不多说，现在我们就开始学习吧。</p><h2>索引类图及源文件组织架构</h2><p>在Kafka源码中，跟索引相关的源码文件有5个，它们都位于core包的/src/main/scala/kafka/log路径下。我们一一来看下。</p><ul>
<li>AbstractIndex.scala：它定义了最顶层的抽象类，这个类封装了所有索引类型的公共操作。</li>
<li>LazyIndex.scala：它定义了AbstractIndex上的一个包装类，实现索引项延迟加载。这个类主要是为了提高性能。</li>
<li>OffsetIndex.scala：定义位移索引，保存“&lt;位移值，文件磁盘物理位置&gt;”对。</li>
<li>TimeIndex.scala：定义时间戳索引，保存“&lt;时间戳，位移值&gt;”对。</li>
<li>TransactionIndex.scala：定义事务索引，为已中止事务（Aborted Transcation）保存重要的元数据信息。只有启用Kafka事务后，这个索引才有可能出现。</li>
</ul><p>这些类的关系如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/34/e8/347480a2d1ae659d0ecf590b01d091e8.jpg?wh=2069*1169" alt=""></p><p>其中，OffsetIndex、TimeIndex和TransactionIndex都继承了AbstractIndex类，而上层的LazyIndex仅仅是包装了一个AbstractIndex的实现类，用于延迟加载。就像我之前说的，LazyIndex的作用是为了提升性能，并没有什么功能上的改进。</p><p>所以今天，我先和你讲一讲AbstractIndex这个抽象父类的代码。下节课，我再重点和你分享具体的索引实现类。</p><h2>AbstractIndex代码结构</h2><p>我们先来看下AbstractIndex的类定义：</p><pre><code>  abstract class AbstractIndex(@volatile var file: File, val baseOffset: Long, val maxIndexSize: Int = -1, val writable: Boolean) extends Closeable {
    ......
    }        
</code></pre><p>AbstractIndex定义了4个属性字段。由于是一个抽象基类，它的所有子类自动地继承了这4个字段。也就是说，Kafka所有类型的索引对象都定义了这些属性。我先给你解释下这些属性的含义。</p><ol>
<li><strong>索引文件</strong>（file）。每个索引对象在磁盘上都对应了一个索引文件。你可能注意到了，这个字段是var型，说明它是可以被修改的。难道索引对象还能动态更换底层的索引文件吗？是的。自1.1.0版本之后，Kafka允许迁移底层的日志路径，所以，索引文件自然要是可以更换的。</li>
<li><strong>起始位移值</strong>（baseOffset）。索引对象对应日志段对象的起始位移值。举个例子，如果你查看Kafka日志路径的话，就会发现，日志文件和索引文件都是成组出现的。比如说，如果日志文件是00000000000000000123.log，正常情况下，一定还有一组索引文件00000000000000000123.index、00000000000000000123.timeindex等。这里的“123”就是这组文件的起始位移值，也就是baseOffset值。</li>
<li><strong>索引文件最大字节数</strong>（maxIndexSize）。它控制索引文件的最大长度。Kafka源码传入该参数的值是Broker端参数segment.index.bytes的值，即10MB。这就是在默认情况下，所有Kafka索引文件大小都是10MB的原因。</li>
<li><strong>索引文件打开方式</strong>（writable）。“True”表示以“读写”方式打开，“False”表示以“只读”方式打开。如果我没记错的话，这个参数应该是我加上去的，就是为了修复我刚刚提到的那个Bug。</li>
</ol><p>AbstractIndex是抽象的索引对象类。可以说，它是承载索引项的容器，而每个继承它的子类负责定义具体的索引项结构。比如，OffsetIndex的索引项是&lt;位移值，物理磁盘位置&gt;对，TimeIndex的索引项是&lt;时间戳，位移值&gt;对。基于这样的设计理念，AbstractIndex类中定义了一个抽象方法entrySize来表示不同索引项的大小，如下所示：</p><pre><code> protected def entrySize: Int
</code></pre><p>子类实现该方法时需要给定自己索引项的大小，对于OffsetIndex而言，该值就是8；对于TimeIndex而言，该值是12，如下所示：</p><pre><code>    // OffsetIndex
    override def entrySize = 8
    // TimeIndex
    override def entrySize = 12
</code></pre><p>说到这儿，你肯定会问，为什么是8和12呢？我来解释一下。</p><p>在OffsetIndex中，位移值用4个字节来表示，物理磁盘位置也用4个字节来表示，所以总共是8个字节。你可能会说，位移值不是长整型吗，应该是8个字节才对啊。</p><p>还记得AbstractIndex已经保存了baseOffset了吗？这里的位移值，实际上是相对于baseOffset的相对位移值，即真实位移值减去baseOffset的值。下节课我会给你重点讲一下它，这里你只需要知道<strong>使用相对位移值能够有效地节省磁盘空间</strong>就行了。而Broker端参数log.segment.bytes是整型，这说明，Kafka中每个日志段文件的大小不会超过2^32，即4GB，这就说明同一个日志段文件上的位移值减去baseOffset的差值一定在整数范围内。因此，源码只需要4个字节保存就行了。</p><p>同理，TimeIndex中的时间戳类型是长整型，占用8个字节，位移依然使用相对位移值，占用4个字节，因此总共需要12个字节。</p><p>如果有人问你，Kafka中的索引底层的实现原理是什么？你可以大声地告诉他：<strong>内存映射文件，即Java中的MappedByteBuffer</strong>。</p><p>使用内存映射文件的主要优势在于，它有很高的I/O性能，特别是对于索引这样的小文件来说，由于文件内存被直接映射到一段虚拟内存上，访问内存映射文件的速度要快于普通的读写文件速度。</p><p>另外，在很多操作系统中（比如Linux），这段映射的内存区域实际上就是内核的页缓存（Page Cache）。这就意味着，里面的数据不需要重复拷贝到用户态空间，避免了很多不必要的时间、空间消耗。</p><p>在AbstractIndex中，这个MappedByteBuffer就是名为mmap的变量。接下来，我用注释的方式，带你深入了解下这个mmap的主要流程。</p><pre><code> @volatile
      protected var mmap: MappedByteBuffer = {
        // 第1步：创建索引文件
        val newlyCreated = file.createNewFile()
        // 第2步：以writable指定的方式（读写方式或只读方式）打开索引文件
        val raf = if (writable) new RandomAccessFile(file, &quot;rw&quot;) else new RandomAccessFile(file, &quot;r&quot;)
        try {
          if(newlyCreated) {
            if(maxIndexSize &lt; entrySize) // 预设的索引文件大小不能太小，如果连一个索引项都保存不了，直接抛出异常
              throw new IllegalArgumentException(&quot;Invalid max index size: &quot; + maxIndexSize)
            // 第3步：设置索引文件长度，roundDownToExactMultiple计算的是不超过maxIndexSize的最大整数倍entrySize
            // 比如maxIndexSize=1234567，entrySize=8，那么调整后的文件长度为1234560
            raf.setLength(roundDownToExactMultiple(maxIndexSize, entrySize))
          }
    
    
          // 第4步：更新索引长度字段_length
          _length = raf.length()
          // 第5步：创建MappedByteBuffer对象
          val idx = {
            if (writable)
              raf.getChannel.map(FileChannel.MapMode.READ_WRITE, 0, _length)
            else
              raf.getChannel.map(FileChannel.MapMode.READ_ONLY, 0, _length)
          }
          /* set the position in the index for the next entry */
          // 第6步：如果是新创建的索引文件，将MappedByteBuffer对象的当前位置置成0
          // 如果索引文件已存在，将MappedByteBuffer对象的当前位置设置成最后一个索引项所在的位置
          if(newlyCreated)
            idx.position(0)
          else
            idx.position(roundDownToExactMultiple(idx.limit(), entrySize))
          // 第7步：返回创建的MappedByteBuffer对象
          idx
        } finally {
          CoreUtils.swallow(raf.close(), AbstractIndex) // 关闭打开索引文件句柄
        }
      }
</code></pre><p>这些代码最主要的作用就是创建mmap对象。要知道，AbstractIndex其他大部分的操作都是和mmap相关。</p><p>我举两个简单的小例子。</p><p>例1：如果我们要计算索引对象中当前有多少个索引项，那么只需要执行下列计算即可：</p><pre><code>    protected var _entries: Int = mmap.position() / entrySize
</code></pre><p>例2：如果我们要计算索引文件最多能容纳多少个索引项，只要定义下面的变量就行了：</p><pre><code>   private[this] var _maxEntries: Int = mmap.limit() / entrySize
</code></pre><p>再进一步，有了这两个变量，我们就能够很容易地编写一个方法，来判断当前索引文件是否已经写满：</p><pre><code> def isFull: Boolean = _entries &gt;= _maxEntries
</code></pre><p>总之，<strong>AbstractIndex中最重要的就是这个mmap变量了</strong>。事实上，AbstractIndex继承类实现添加索引项的主要逻辑，也就是<strong>向mmap中添加对应的字段</strong>。</p><h2>写入索引项</h2><p>下面这段代码是OffsetIndex的append方法，用于向索引文件中写入新索引项。</p><pre><code> def append(offset: Long, position: Int): Unit = {
        inLock(lock) {
          // 第1步：判断索引文件未写满
          require(!isFull, &quot;Attempt to append to a full index (size = &quot; + _entries + &quot;).&quot;)
          // 第2步：必须满足以下条件之一才允许写入索引项：
          // 条件1：当前索引文件为空
          // 条件2：要写入的位移大于当前所有已写入的索引项的位移——Kafka规定索引项中的位移值必须是单调增加的
          if (_entries == 0 || offset &gt; _lastOffset) {
            trace(s&quot;Adding index entry $offset =&gt; $position to ${file.getAbsolutePath}&quot;)
            mmap.putInt(relativeOffset(offset)) // 第3步A：向mmap中写入相对位移值
            mmap.putInt(position) // 第3步B：向mmap中写入物理位置信息
            // 第4步：更新其他元数据统计信息，如当前索引项计数器_entries和当前索引项最新位移值_lastOffset
            _entries += 1
            _lastOffset = offset
            // 第5步：执行校验。写入的索引项格式必须符合要求，即索引项个数*单个索引项占用字节数匹配当前文件物理大小，否则说明文件已损坏
            require(_entries * entrySize == mmap.position(), entries + &quot; entries but file position in index is &quot; + mmap.position() + &quot;.&quot;)
          } else {
            // 如果第2步中两个条件都不满足，不能执行写入索引项操作，抛出异常
            throw new InvalidOffsetException(s&quot;Attempt to append an offset ($offset) to position $entries no larger than&quot; +
              s&quot; the last offset appended (${_lastOffset}) to ${file.getAbsolutePath}.&quot;)
          }
        }
      }
</code></pre><p>我使用一张图来总结下append方法的执行流程，希望可以帮你更快速地掌握它的实现：</p><p><img src="https://static001.geekbang.org/resource/image/23/ea/236f0731ff2799f0902ff7293cf6ddea.jpg?wh=2048*2063" alt=""></p><h2>查找索引项</h2><p>索引项的写入逻辑并不复杂，难点在于如何查找索引项。AbstractIndex定义了抽象方法<strong>parseEntry</strong>用于查找给定的索引项，如下所示：</p><pre><code>protected def parseEntry(buffer: ByteBuffer, n: Int): IndexEntry
</code></pre><p>这里的“n”表示要查找给定ByteBuffer中保存的第n个索引项（在Kafka中也称第n个槽）。IndexEntry是源码定义的一个接口，里面有两个方法：indexKey和indexValue，分别返回不同类型索引的&lt;Key，Value&gt;对。</p><p>OffsetIndex实现parseEntry的逻辑如下：</p><pre><code>   override protected def parseEntry(buffer: ByteBuffer, n: Int): OffsetPosition = {
        OffsetPosition(baseOffset + relativeOffset(buffer, n), physical(buffer, n))
      }
</code></pre><p>OffsetPosition是实现IndexEntry的实现类，Key就是之前说的位移值，而Value就是物理磁盘位置值。所以，这里你能看到代码调用了relativeOffset(buffer, n) + baseOffset计算出绝对位移值，之后调用physical(buffer, n)计算物理磁盘位置，最后将它们封装到一起作为一个独立的索引项返回。</p><p><strong>我建议你去看下relativeOffset和physical方法的实现，看看它们是如何计算相对位移值和物理磁盘位置信息的</strong>。</p><p>有了parseEntry方法，我们就能够根据给定的n来查找索引项了。但是，这里还有个问题需要解决，那就是，我们如何确定要找的索引项在第n个槽中呢？其实本质上，这是一个算法问题，也就是如何从一组已排序的数中快速定位符合条件的那个数。</p><h2>二分查找算法</h2><p>到目前为止，从已排序数组中寻找某个数字最快速的算法就是二分查找了，它能做到O(lgN)的时间复杂度。Kafka的索引组件就应用了二分查找算法。</p><p>我先给出原版的实现算法代码。</p><pre><code>  private def indexSlotRangeFor(idx: ByteBuffer, target: Long, searchEntity: IndexSearchEntity): (Int, Int) = {
        // 第1步：如果当前索引为空，直接返回&lt;-1,-1&gt;对
        if(_entries == 0)
          return (-1, -1)
    
    
        // 第2步：要查找的位移值不能小于当前最小位移值
        if(compareIndexEntry(parseEntry(idx, 0), target, searchEntity) &gt; 0)
          return (-1, 0)
    
    
        // binary search for the entry
        // 第3步：执行二分查找算法
        var lo = 0
        var hi = _entries - 1
        while(lo &lt; hi) {
          val mid = ceil(hi/2.0 + lo/2.0).toInt
          val found = parseEntry(idx, mid)
          val compareResult = compareIndexEntry(found, target, searchEntity)
          if(compareResult &gt; 0)
            hi = mid - 1
          else if(compareResult &lt; 0)
            lo = mid
          else
            return (mid, mid)
        }
    
    
        (lo, if (lo == _entries - 1) -1 else lo + 1)
</code></pre><p>这段代码的核心是，第3步的二分查找算法。熟悉Binary Search的话，你对这段代码一定不会感到陌生。</p><p>讲到这里，似乎一切很完美：Kafka索引应用二分查找算法快速定位待查找索引项位置，之后调用parseEntry来读取索引项。不过，这真的就是无懈可击的解决方案了吗？</p><h2>改进版二分查找算法</h2><p>显然不是！我前面说过了，大多数操作系统使用页缓存来实现内存映射，而目前几乎所有的操作系统都使用LRU（Least Recently Used）或类似于LRU的机制来管理页缓存。</p><p>Kafka写入索引文件的方式是在文件末尾追加写入，而几乎所有的索引查询都集中在索引的尾部。这么来看的话，LRU机制是非常适合Kafka的索引访问场景的。</p><p>但，这里有个问题是，当Kafka在查询索引的时候，原版的二分查找算法并没有考虑到缓存的问题，因此很可能会导致一些不必要的缺页中断（Page Fault）。此时，Kafka线程会被阻塞，等待对应的索引项从物理磁盘中读出并放入到页缓存中。</p><p>下面我举个例子来说明一下这个情况。假设Kafka的某个索引占用了操作系统页缓存13个页（Page），如果待查找的位移值位于最后一个页上，也就是Page 12，那么标准的二分查找算法会依次读取页号0、6、9、11和12，具体的推演流程如下所示：</p><p><img src="https://static001.geekbang.org/resource/image/e4/85/e4fc56a301b13afae4dda303e5366085.jpg?wh=3160*2572" alt=""></p><p>通常来说，一个页上保存了成百上千的索引项数据。随着索引文件不断被写入，Page #12不断地被填充新的索引项。如果此时索引查询方都来自ISR副本或Lag很小的消费者，那么这些查询大多集中在对Page #12的查询，因此，Page #0、6、9、11、12一定经常性地被源码访问。也就是说，这些页一定保存在页缓存上。后面当新的索引项填满了Page #12，页缓存就会申请一个新的Page来保存索引项，即Page #13。</p><p>现在，最新索引项保存在Page #13中。如果要查找最新索引项，原版二分查找算法将会依次访问Page #0、7、10、12和13。此时，问题来了：Page 7和10已经很久没有被访问过了，它们大概率不在页缓存中，因此，一旦索引开始征用Page #13，就会发生Page Fault，等待那些冷页数据从磁盘中加载到页缓存。根据国外用户的测试，这种加载过程可能长达1秒。</p><p>显然，这是一个普遍的问题，即每当索引文件占用Page数发生变化时，就会强行变更二分查找的搜索路径，从而出现不在页缓存的冷数据必须要加载到页缓存的情形，而这种加载过程是非常耗时的。</p><p>基于这个问题，社区提出了改进版的二分查找策略，也就是缓存友好的搜索算法。总体的思路是，代码将所有索引项分成两个部分：热区（Warm Area）和冷区（Cold Area），然后分别在这两个区域内执行二分查找算法，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/e1/8e/e135f0a6b3fb43ead51db4ddbad1638e.jpg?wh=2153*1355" alt=""></p><p>乍一看，该算法并没有什么高大上的改进，仅仅是把搜寻区域分成了冷、热两个区域，然后有条件地在不同区域执行普通的二分查找算法罢了。实际上，这个改进版算法提供了一个重要的保证：<strong>它能保证那些经常需要被访问的Page组合是固定的</strong>。</p><p>想想刚才的例子，同样是查询最热的那部分数据，一旦索引占用了更多的Page，要遍历的Page组合就会发生变化。这是导致性能下降的主要原因。</p><p>这个改进版算法的最大好处在于，<strong>查询最热那部分数据所遍历的Page永远是固定的，因此大概率在页缓存中，从而避免无意义的Page Fault</strong>。</p><p>下面我们来看实际的代码。我用注释的方式解释了改进版算法的实现逻辑。一旦你了解了冷区热区的分割原理，剩下的就不难了。</p><pre><code>   private def indexSlotRangeFor(idx: ByteBuffer, target: Long, searchEntity: IndexSearchEntity): (Int, Int) = {
        // 第1步：如果索引为空，直接返回&lt;-1,-1&gt;对
        if(_entries == 0)
          return (-1, -1)
    
    
        // 封装原版的二分查找算法
        def binarySearch(begin: Int, end: Int) : (Int, Int) = {
          // binary search for the entry
          var lo = begin
          var hi = end
          while(lo &lt; hi) {
            val mid = (lo + hi + 1) &gt;&gt;&gt; 1
            val found = parseEntry(idx, mid)
            val compareResult = compareIndexEntry(found, target, searchEntity)
            if(compareResult &gt; 0)
              hi = mid - 1
            else if(compareResult &lt; 0)
              lo = mid
            else
              return (mid, mid)
          }
          (lo, if (lo == _entries - 1) -1 else lo + 1)
        }
    
    
        // 第3步：确认热区首个索引项位于哪个槽。_warmEntries就是所谓的分割线，目前固定为8192字节处
        // 如果是OffsetIndex，_warmEntries = 8192 / 8 = 1024，即第1024个槽
        // 如果是TimeIndex，_warmEntries = 8192 / 12 = 682，即第682个槽
        val firstHotEntry = Math.max(0, _entries - 1 - _warmEntries)
        // 第4步：判断target位移值在热区还是冷区
        if(compareIndexEntry(parseEntry(idx, firstHotEntry), target, searchEntity) &lt; 0) {
          return binarySearch(firstHotEntry, _entries - 1) // 如果在热区，搜索热区
        }
    
    
        // 第5步：确保target位移值不能小于当前最小位移值
        if(compareIndexEntry(parseEntry(idx, 0), target, searchEntity) &gt; 0)
          return (-1, 0)
    
    
        // 第6步：如果在冷区，搜索冷区
        binarySearch(0, firstHotEntry)
</code></pre><h2>总结</h2><p>今天，我带你详细学习了Kafka中的索引机制，以及社区如何应用二分查找算法实现快速定位索引项。有两个重点，你一定要记住。</p><ol>
<li>AbstractIndex：这是Kafka所有类型索引的抽象父类，里面的mmap变量是实现索引机制的核心，你一定要掌握它。</li>
<li>改进版二分查找算法：社区在标准原版的基础上，对二分查找算法根据实际访问场景做了定制化的改进。你需要特别关注改进版在提升缓存性能方面做了哪些努力。改进版能够有效地提升页缓存的使用率，从而在整体上降低物理I/O，缓解系统负载瓶颈。你最好能够从索引这个维度去思考社区在这方面所做的工作。</li>
</ol><p><img src="https://static001.geekbang.org/resource/image/31/22/31e35198818b0bbc05ef333b5897b022.jpg?wh=2581*3727" alt=""></p><p>实际上，无论是AbstractIndex还是它使用的二分查找算法，它们都属于Kafka索引共性的东西，即所有Kafka索引都具备这些特点或特性。那么，你是否想了解不同类型索引之间的区别呢？比如位移索引和时间戳索引有哪些异同之处。这些问题的答案我会在下节课揭晓，你千万不要错过。</p><h2>课后讨论</h2><p>目前，冷区和热区的分割线设定在8192字节处，请结合源码注释以及你自己的理解，讲一讲为什么要设置成8192？</p><p>欢迎你在留言区畅所欲言，跟我交流讨论，也欢迎你把今天的内容分享给你的朋友。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/78/d3/1dc40aa2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Jonah</span>
  </div>
  <div class="_2_QraFYR_0">按照注释讲的分析，主要是为了保证热区的页数小于3个，另外保证in-sync 的查找都能命中</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-21 09:22:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/a7/9a/495cb99a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>胡夕</span>
  </div>
  <div class="_2_QraFYR_0">你好，我是胡夕。我来公布上节课的“课后讨论”题答案啦～<br><br>上节课，咱们重点了解了日志对象的常见操作，课后我让你思考下能否为Log源码添加一个简便方法，统计介于高水位值和LEO值之间的消息总数。以下是我的答案：<br>private[log] def messageCountBetweenHWAndLEO: Long = {<br>    logEndOffset - highWatermarkMetadata.messageOffset<br>  }<br><br><br>okay，你是怎么考虑的呢？我们可以一起讨论下。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-23 10:59:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/67/49/864dba17.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>东风第一枝</span>
  </div>
  <div class="_2_QraFYR_0">课后作业：1. 主流的处理器架构中4096一般是最小的页面大小，这样可以保证页数小于三，在二分查找时，可以命中所有的页面。<br>2. 8KB的索引，可以覆盖4M数据（offset index）或者2.7M数据（time index）足够保证in-sync的查找的内容都在热区</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-21 17:33:04</div>
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
  <div class="_2_QraFYR_0">对于这个特定二分查找算法由于之前的理解错误，我又重新阅读了一遍源码，得出以下几个内容点：<br>1、这个问题是针对【KAFKA-6432】这个问题的优化（优化索引lookup的缓存友好性）<br>2、划分为8kb的最主要原因是因为硬件厂商架构不同的原因：x86-32, x86-64, MIPS,SPARC, Power, ARM这些架构基本上都是4kb的page size 。一般这些架构下最多只可能存在 【3 或者更少】的page 属于所谓的 warm page 所有 8kb是一种折衷后的决定<br>3、这段源码的作者有提示到这块代码是临时方案，因为没办法主动触发page cache 所以目前只能按照厂商标准来做<br>4、这段源码的作者有提到，未来可以通过一个后台线程的方式不断的去把 page 变成warm page，这样做有两个好处：A、能够支持更大超过8kb的warm 区   B、在较低QPS的 topic-partitions 也能使用page cache 的红利<br><br><br>我个人的看法：<br>1、kafka中如果存在太多的低QPS的topic，其实修复这个问题作者的方法不太优雅<br>2、是否可以效仿mysql的方式，自行实现特殊的page cache 呢？这样不会被厂商约束，同时能有更加灵活的定制</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-22 14:58:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/dc/3e/abb7bfe3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小崔</span>
  </div>
  <div class="_2_QraFYR_0">冷热分离如何生效的，刚开始看文章没看懂，又翻了源码注释，才明白了。这里梳理一下我觉得更易懂的说法：大部分查询集中在索引项尾部-&gt;把尾部的8192字节设置为热区，永远保存在缓存中-&gt;如果查询target在热区索引项范围，直接查热区，避免页中断。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，是这个意思👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-01 20:05:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/bc/7c/f6a6fb47.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ByteFeng</span>
  </div>
  <div class="_2_QraFYR_0">胡夕老师，你能不能出个kafka视频课程呀???文字专栏理论性比较强，视频课程的实战性比较强。。。。。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯嗯，之前确实有过对照着IDEA直接讲源码的想法。因为觉得直接演示怎么搭建环境、怎么修改源码、怎么写测试用例可能比文字来的有冲击力。后续我会认真考虑看看的。谢谢您的建议：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-22 18:02:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_14e7f8</span>
  </div>
  <div class="_2_QraFYR_0">而 Broker 端参数 log.segment.bytes 是整型，这说明，Kafka 中每个日志段文件的大小不会超过 2^32，即 4GB，这就说明同一个日志段文件上的位移值减去 baseOffset 的差值一定在整数范围内。<br><br>你好， 关于这里日志大小有点疑问，java中整形最大值是 2^31 - 1 ，所以单个日志最大应该是2G吧？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 单个日志段是2GB，日志当然可以无限大小</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-19 18:37:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1c/4e/88/791d0f5e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lei</span>
  </div>
  <div class="_2_QraFYR_0">日志段有个参数indexIntervalBytes， 可以理解为插了多少条消息之后再建一个索引，由此看出kafka的索引其实是稀疏索引。那也就是说二分查找之后，还有一个顺序遍历查找目标文件的过程。我这样理解对吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯嗯，会有的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-05 17:54:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/23/37/61/51e10a30.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>QAQ</span>
  </div>
  <div class="_2_QraFYR_0">和mysql的热点数据想法类似</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-13 17:56:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/9a/25/3e5e942b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张子涵</span>
  </div>
  <div class="_2_QraFYR_0">读完这章看到问题的时候，完全没有头绪，二分算法及索引冷热分区思想都懂，还是计算机基础比较差，对于8192这样的数据不太敏感，要继续加油</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-13 15:48:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/55/a9/5282a560.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>yic</span>
  </div>
  <div class="_2_QraFYR_0">老师，我想问2个问题：<br>1. 为什么8K index offset可以覆盖4M的数据？<br>2. 8K其实是一个经验值吧？考虑的点主要是non-lagging consumer的性能和占用的缓存空间。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. 8KB = 8092B, 每个位移索引项大小是8B，所以这个热区能够保存1024个索引项。由于默认每4KB写入一个索引项，因此8KB的热区能够覆盖1024 * 4KB = 4MB的日志数据<br>2. 同意~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-04 16:16:22</div>
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
  <div class="_2_QraFYR_0">我对8192的理解是这样的：<br>broker默认的索引文件大小segment.index.bytes = 10 MB，8192刚好大约是10MB的80%，根据2&#47;8定律，最新的20%的数据可能会是热点数据所以设置为8192切分冷区和热区。<br><br>但是这里也有两个问题希望能得到老师的看法：<br>1、如果我修改了segment.index.bytes大小或者修改了Linux的页缓存大小设置，会不会导致冷热区查询性能失去了，毕竟代码是写死的8196.<br>2、如果我频繁范围冷区，当页缓存满的时候，热区本身也会失效假入有两个consumer-group，一共消费能力强的在消息发送出来后能立即消费，另一个消费能力弱，还在消费昨天的数据，的情况下应该还是会出现这种页缓存带来的性能问题吧</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. 所以不要贸然修改这个默认值：）<br><br>2. 这个算法保证主要保证non-lagging consumer的性能。lagging consumer确实享受不到，甚至是拖累其他consumer，因为它会“污染”页缓存</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-25 11:04:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/29/57/61/369a609c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>A</span>
  </div>
  <div class="_2_QraFYR_0">老师，能否告诉一下这个课对应的kafka源码的版本，我是刚买的现在的好多源码都变了；我找了一下咱这个课好像没有说明指定的commit号</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-25 11:54:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/28/d9/be/2af79055.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>loserwang，no ssp</span>
  </div>
  <div class="_2_QraFYR_0">这个索引的二分查找，每个索引文件都有热区，但实际上只要activeSegment才有热区吧？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-08 14:55:31</div>
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
  <div class="_2_QraFYR_0">老师，TransactionIndex好像已经没有继承于，AbstractIndex了。TransactionIndex好像变成了一种特殊的索引了，是独立的，TimeIndex 和 OffsetIndex 是作为泛型给 LazyIndex 使用的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯嗯，源码的确在不断演进中。现在的优化思路是朝着延迟初始化索引，期望快速缩短broker启动时间</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-08 16:44:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>K菌无惨</span>
  </div>
  <div class="_2_QraFYR_0">受益匪浅</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-10 15:40:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_633432</span>
  </div>
  <div class="_2_QraFYR_0">请问老师，leader收到生产请求，向本地写入消息，如果此时消息还在缓存中，还没有同步到磁盘，那么follower和消费者可以消费这部分数据吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以消费到。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-19 10:34:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/e3/8b/27f875ba.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Bryant.C</span>
  </div>
  <div class="_2_QraFYR_0">老师好，如果Kafka集群中存在比较多的lagging consumer，站在Kafka的角度有什么比较好的优化思路呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 更多还是从优化consumer着手，看看是不是处理逻辑过重导致的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-22 12:40:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/7d/63/b9660319.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Eternity°</span>
  </div>
  <div class="_2_QraFYR_0">学习rocketmq是 发现的也是用的内存映射，也算是一个相似点么</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，应该是。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-04 16:12:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>懂码哥(GerryWen)</span>
  </div>
  <div class="_2_QraFYR_0">学到这边开始晕了，读着读着就不知道是什么意思了，怎么办？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 二分查找算法应该还比较熟悉吧，可以对照着标准的算法于Kafka源码比较下</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-30 21:53:17</div>
  </div>
</div>
</div>
</li>
</ul>