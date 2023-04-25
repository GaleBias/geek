<audio title="04 _ AOF日志：宕机了，Redis如何避免数据丢失？" src="https://static001.geekbang.org/resource/audio/8e/5c/8ea8a96310fb035d222657e917cc2b5c.mp3" controls="controls"></audio> 
<p>你好，我是蒋德钧。</p><p>如果有人问你：“你会把Redis用在什么业务场景下？”我想你大概率会说：“我会把它当作缓存使用，因为它把后端数据库中的数据存储在内存中，然后直接从内存中读取数据，响应速度会非常快。”没错，这确实是Redis的一个普遍使用场景，但是，这里也有一个绝对不能忽略的问题：<strong>一旦服务器宕机，内存中的数据将全部丢失。</strong></p><p>我们很容易想到的一个解决方案是，从后端数据库恢复这些数据，但这种方式存在两个问题：一是，需要频繁访问数据库，会给数据库带来巨大的压力；二是，这些数据是从慢速数据库中读取出来的，性能肯定比不上从Redis中读取，导致使用这些数据的应用程序响应变慢。所以，对Redis来说，实现数据的持久化，避免从后端数据库中进行恢复，是至关重要的。</p><p>目前，Redis的持久化主要有两大机制，即AOF（Append Only File）日志和RDB快照。在接下来的两节课里，我们就分别学习一下吧。这节课，我们先重点学习下AOF日志。</p><h2>AOF日志是如何实现的？</h2><p>说到日志，我们比较熟悉的是数据库的写前日志（Write Ahead Log, WAL），也就是说，在实际写数据前，先把修改的数据记到日志文件中，以便故障时进行恢复。不过，AOF日志正好相反，它是写后日志，“写后”的意思是Redis是先执行命令，把数据写入内存，然后才记录日志，如下图所示：</p><!-- [[[read_end]]] --><p><img src="https://static001.geekbang.org/resource/image/40/1f/407f2686083afc37351cfd9107319a1f.jpg?wh=3218*1789" alt="" title="Redis AOF操作过程"></p><p>那AOF为什么要先执行命令再记日志呢？要回答这个问题，我们要先知道AOF里记录了什么内容。</p><p>传统数据库的日志，例如redo log（重做日志），记录的是修改后的数据，而AOF里记录的是Redis收到的每一条命令，这些命令是以文本形式保存的。</p><p>我们以Redis收到“set testkey testvalue”命令后记录的日志为例，看看AOF日志的内容。其中，“<code>*3</code>”表示当前命令有三个部分，每部分都是由“<code>$+数字</code>”开头，后面紧跟着具体的命令、键或值。这里，“数字”表示这部分中的命令、键或值一共有多少字节。例如，“<code>$3 set</code>”表示这部分有3个字节，也就是“set”命令。</p><p><img src="https://static001.geekbang.org/resource/image/4d/9f/4d120bee623642e75fdf1c0700623a9f.jpg?wh=3094*2250" alt="" title="Redis AOF日志内容"></p><p>但是，为了避免额外的检查开销，Redis在向AOF里面记录日志的时候，并不会先去对这些命令进行语法检查。所以，如果先记日志再执行命令的话，日志中就有可能记录了错误的命令，Redis在使用日志恢复数据时，就可能会出错。</p><p>而写后日志这种方式，就是先让系统执行命令，只有命令能执行成功，才会被记录到日志中，否则，系统就会直接向客户端报错。所以，Redis使用写后日志这一方式的一大好处是，可以避免出现记录错误命令的情况。</p><p>除此之外，AOF还有一个好处：它是在命令执行后才记录日志，所以<strong>不会阻塞当前的写操作</strong>。</p><p>不过，AOF也有两个潜在的风险。</p><p>首先，如果刚执行完一个命令，还没有来得及记日志就宕机了，那么这个命令和相应的数据就有丢失的风险。如果此时Redis是用作缓存，还可以从后端数据库重新读入数据进行恢复，但是，如果Redis是直接用作数据库的话，此时，因为命令没有记入日志，所以就无法用日志进行恢复了。</p><p>其次，AOF虽然避免了对当前命令的阻塞，但可能会给下一个操作带来阻塞风险。这是因为，AOF日志也是在主线程中执行的，如果在把日志文件写入磁盘时，磁盘写压力大，就会导致写盘很慢，进而导致后续的操作也无法执行了。</p><p>仔细分析的话，你就会发现，这两个风险都是和AOF写回磁盘的时机相关的。这也就意味着，如果我们能够控制一个写命令执行完后AOF日志写回磁盘的时机，这两个风险就解除了。</p><h2>三种写回策略</h2><p>其实，对于这个问题，AOF机制给我们提供了三个选择，也就是AOF配置项appendfsync的三个可选值。</p><ul>
<li><strong>Always</strong>，同步写回：每个写命令执行完，立马同步地将日志写回磁盘；</li>
<li><strong>Everysec</strong>，每秒写回：每个写命令执行完，只是先把日志写到AOF文件的内存缓冲区，每隔一秒把缓冲区中的内容写入磁盘；</li>
<li><strong>No</strong>，操作系统控制的写回：每个写命令执行完，只是先把日志写到AOF文件的内存缓冲区，由操作系统决定何时将缓冲区内容写回磁盘。</li>
</ul><p>针对避免主线程阻塞和减少数据丢失问题，这三种写回策略都无法做到两全其美。我们来分析下其中的原因。</p><ul>
<li>“同步写回”可以做到基本不丢数据，但是它在每一个写命令后都有一个慢速的落盘操作，不可避免地会影响主线程性能；</li>
<li>虽然“操作系统控制的写回”在写完缓冲区后，就可以继续执行后续的命令，但是落盘的时机已经不在Redis手中了，只要AOF记录没有写回磁盘，一旦宕机对应的数据就丢失了；</li>
<li>“每秒写回”采用一秒写回一次的频率，避免了“同步写回”的性能开销，虽然减少了对系统性能的影响，但是如果发生宕机，上一秒内未落盘的命令操作仍然会丢失。所以，这只能算是，在避免影响主线程性能和避免数据丢失两者间取了个折中。</li>
</ul><p>我把这三种策略的写回时机，以及优缺点汇总在了一张表格里，以方便你随时查看。</p><p><img src="https://static001.geekbang.org/resource/image/72/f8/72f547f18dbac788c7d11yy167d7ebf8.jpg?wh=2284*682" alt=""></p><p>到这里，我们就可以根据系统对高性能和高可靠性的要求，来选择使用哪种写回策略了。总结一下就是：想要获得高性能，就选择No策略；如果想要得到高可靠性保证，就选择Always策略；如果允许数据有一点丢失，又希望性能别受太大影响的话，那么就选择Everysec策略。</p><p>但是，按照系统的性能需求选定了写回策略，并不是“高枕无忧”了。毕竟，AOF是以文件的形式在记录接收到的所有写命令。随着接收的写命令越来越多，AOF文件会越来越大。这也就意味着，我们一定要小心AOF文件过大带来的性能问题。</p><p>这里的“性能问题”，主要在于以下三个方面：一是，文件系统本身对文件大小有限制，无法保存过大的文件；二是，如果文件太大，之后再往里面追加命令记录的话，效率也会变低；三是，如果发生宕机，AOF中记录的命令要一个个被重新执行，用于故障恢复，如果日志文件太大，整个恢复过程就会非常缓慢，这就会影响到Redis的正常使用。</p><p>所以，我们就要采取一定的控制手段，这个时候，<strong>AOF重写机制</strong>就登场了。</p><h2>日志文件太大了怎么办？</h2><p>简单来说，AOF重写机制就是在重写时，Redis根据数据库的现状创建一个新的AOF文件，也就是说，读取数据库中的所有键值对，然后对每一个键值对用一条命令记录它的写入。比如说，当读取了键值对“testkey”: “testvalue”之后，重写机制会记录set testkey testvalue这条命令。这样，当需要恢复时，可以重新执行该命令，实现“testkey”: “testvalue”的写入。</p><p>为什么重写机制可以把日志文件变小呢? 实际上，重写机制具有“多变一”功能。所谓的“多变一”，也就是说，旧日志文件中的多条命令，在重写后的新日志中变成了一条命令。</p><p>我们知道，AOF文件是以追加的方式，逐一记录接收到的写命令的。当一个键值对被多条写命令反复修改时，AOF文件会记录相应的多条命令。但是，在重写的时候，是根据这个键值对当前的最新状态，为它生成对应的写入命令。这样一来，一个键值对在重写日志中只用一条命令就行了，而且，在日志恢复时，只用执行这条命令，就可以直接完成这个键值对的写入了。</p><p>下面这张图就是一个例子：</p><p><img src="https://static001.geekbang.org/resource/image/65/08/6528c699fdcf40b404af57040bb8d208.jpg?wh=4000*1088" alt="" title="AOF重写减少日志大小"></p><p>当我们对一个列表先后做了6次修改操作后，列表的最后状态是[“D”, “C”, “N”]，此时，只用LPUSH u:list “N”, “C”, "D"这一条命令就能实现该数据的恢复，这就节省了五条命令的空间。对于被修改过成百上千次的键值对来说，重写能节省的空间当然就更大了。</p><p>不过，虽然AOF重写后，日志文件会缩小，但是，要把整个数据库的最新数据的操作日志都写回磁盘，仍然是一个非常耗时的过程。这时，我们就要继续关注另一个问题了：重写会不会阻塞主线程？</p><h2>AOF重写会阻塞吗?</h2><p>和AOF日志由主线程写回不同，重写过程是由后台子进程bgrewriteaof来完成的，这也是为了避免阻塞主线程，导致数据库性能下降。</p><p>我把重写的过程总结为“<strong>一个拷贝，两处日志</strong>”。</p><p>“一个拷贝”就是指，每次执行重写时，主线程fork出后台的bgrewriteaof子进程。此时，fork会把主线程的内存拷贝一份给bgrewriteaof子进程，这里面就包含了数据库的最新数据。然后，bgrewriteaof子进程就可以在不影响主线程的情况下，逐一把拷贝的数据写成操作，记入重写日志。</p><p>“两处日志”又是什么呢？</p><p>因为主线程未阻塞，仍然可以处理新来的操作。此时，如果有写操作，第一处日志就是指正在使用的AOF日志，Redis会把这个操作写到它的缓冲区。这样一来，即使宕机了，这个AOF日志的操作仍然是齐全的，可以用于恢复。</p><p>而第二处日志，就是指新的AOF重写日志。这个操作也会被写到重写日志的缓冲区。这样，重写日志也不会丢失最新的操作。等到拷贝数据的所有操作记录重写完成后，重写日志记录的这些最新操作也会写入新的AOF文件，以保证数据库最新状态的记录。此时，我们就可以用新的AOF文件替代旧文件了。</p><p><img src="https://static001.geekbang.org/resource/image/6b/e8/6b054eb1aed0734bd81ddab9a31d0be8.jpg?wh=3688*1920" alt="" title="AOF非阻塞的重写过程"></p><p>总结来说，每次AOF重写时，Redis会先执行一个内存拷贝，用于重写；然后，使用两个日志保证在重写过程中，新写入的数据不会丢失。而且，因为Redis采用额外的线程进行数据重写，所以，这个过程并不会阻塞主线程。</p><h2>小结</h2><p>这节课，我向你介绍了Redis用于避免数据丢失的AOF方法。这个方法通过逐一记录操作命令，在恢复时再逐一执行命令的方式，保证了数据的可靠性。</p><p>这个方法看似“简单”，但也是充分考虑了对Redis性能的影响。总结来说，它提供了AOF日志的三种写回策略，分别是Always、Everysec和No，这三种策略在可靠性上是从高到低，而在性能上则是从低到高。</p><p>此外，为了避免日志文件过大，Redis还提供了AOF重写机制，直接根据数据库里数据的最新状态，生成这些数据的插入命令，作为新日志。这个过程通过后台线程完成，避免了对主线程的阻塞。</p><p>其中，三种写回策略体现了系统设计中的一个重要原则 ，即trade-off，或者称为“取舍”，指的就是在性能和可靠性保证之间做取舍。我认为，这是做系统设计和开发的一个关键哲学，我也非常希望，你能充分地理解这个原则，并在日常开发中加以应用。</p><p>不过，你可能也注意到了，落盘时机和重写机制都是在“记日志”这一过程中发挥作用的。例如，落盘时机的选择可以避免记日志时阻塞主线程，重写可以避免日志文件过大。但是，在“用日志”的过程中，也就是使用AOF进行故障恢复时，我们仍然需要把所有的操作记录都运行一遍。再加上Redis的单线程设计，这些命令操作只能一条一条按顺序执行，这个“重放”的过程就会很慢了。</p><p>那么，有没有既能避免数据丢失，又能更快地恢复的方法呢？当然有，那就是RDB快照了。下节课，我们就一起学习一下，敬请期待。</p><h2>每课一问</h2><p>这节课，我给你提两个小问题：</p><ol>
<li>AOF日志重写的时候，是由bgrewriteaof子进程来完成的，不用主线程参与，我们今天说的非阻塞也是指子进程的执行不阻塞主线程。但是，你觉得，这个重写过程有没有其他潜在的阻塞风险呢？如果有的话，会在哪里阻塞？</li>
<li>AOF重写也有一个重写日志，为什么它不共享使用AOF本身的日志呢？</li>
</ol><p>希望你能好好思考一下这两个问题，欢迎在留言区分享你的答案。另外，也欢迎你把这节课的内容转发出去，和更多的人一起交流讨论。</p>
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
  <div class="_2_QraFYR_0">问题1，Redis采用fork子进程重写AOF文件时，潜在的阻塞风险包括：fork子进程 和 AOF重写过程中父进程产生写入的场景，下面依次介绍。<br><br>	a、fork子进程，fork这个瞬间一定是会阻塞主线程的（注意，fork时并不会一次性拷贝所有内存数据给子进程，老师文章写的是拷贝所有内存数据给子进程，我个人认为是有歧义的），fork采用操作系统提供的写实复制(Copy On Write)机制，就是为了避免一次性拷贝大量内存数据给子进程造成的长时间阻塞问题，但fork子进程需要拷贝进程必要的数据结构，其中有一项就是拷贝内存页表（虚拟内存和物理内存的映射索引表），这个拷贝过程会消耗大量CPU资源，拷贝完成之前整个进程是会阻塞的，阻塞时间取决于整个实例的内存大小，实例越大，内存页表越大，fork阻塞时间越久。拷贝内存页表完成后，子进程与父进程指向相同的内存地址空间，也就是说此时虽然产生了子进程，但是并没有申请与父进程相同的内存大小。那什么时候父子进程才会真正内存分离呢？“写实复制”顾名思义，就是在写发生时，才真正拷贝内存真正的数据，这个过程中，父进程也可能会产生阻塞的风险，就是下面介绍的场景。<br><br>	b、fork出的子进程指向与父进程相同的内存地址空间，此时子进程就可以执行AOF重写，把内存中的所有数据写入到AOF文件中。但是此时父进程依旧是会有流量写入的，如果父进程操作的是一个已经存在的key，那么这个时候父进程就会真正拷贝这个key对应的内存数据，申请新的内存空间，这样逐渐地，父子进程内存数据开始分离，父子进程逐渐拥有各自独立的内存空间。因为内存分配是以页为单位进行分配的，默认4k，如果父进程此时操作的是一个bigkey，重新申请大块内存耗时会变长，可能会产阻塞风险。另外，如果操作系统开启了内存大页机制(Huge Page，页面大小2M)，那么父进程申请内存时阻塞的概率将会大大提高，所以在Redis机器上需要关闭Huge Page机制。Redis每次fork生成RDB或AOF重写完成后，都可以在Redis log中看到父进程重新申请了多大的内存空间。<br><br>问题2，AOF重写不复用AOF本身的日志，一个原因是父子进程写同一个文件必然会产生竞争问题，控制竞争就意味着会影响父进程的性能。二是如果AOF重写过程中失败了，那么原本的AOF文件相当于被污染了，无法做恢复使用。所以Redis AOF重写一个新文件，重写失败的话，直接删除这个文件就好了，不会对原先的AOF文件产生影响。等重写完成之后，直接替换旧文件即可。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 非常赞！这个回答一定要置顶！<br><br>而且Kaito同学的不少问题回答都非常仔细和精彩，非常值得一读！<br><br>这里要谢谢Kaito同学指出的文章中的歧义：fork子进程时，子进程是会拷贝父进程的页表，即虚实映射关系，而不会拷贝物理内存。子进程复制了父进程页表，也能共享访问父进程的内存数据了，此时，类似于有了父进程的所有内存数据。我的描述不太严谨了，非常感谢指出！<br><br>Kaito同学还提到了Huge page。这个特性大家在使用Redis也要注意。Huge page对提升TLB命中率比较友好，因为在相同的内存容量下，使用huge page可以减少页表项，TLB就可以缓存更多的页表项，能减少TLB miss的开销。<br><br>但是，这个机制对于Redis这种喜欢用fork的系统来说，的确不太友好，尤其是在Redis的写入请求比较多的情况下。因为fork后，父进程修改数据采用写时复制，复制的粒度为一个内存页。如果只是修改一个256B的数据，父进程需要读原来的内存页，然后再映射到新的物理地址写入。一读一写会造成读写放大。如果内存页越大（例如2MB的大页），那么读写放大也就越严重，对Redis性能造成影响。<br><br>Huge page在实际使用Redis时是建议关掉的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-12 11:38:16</div>
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
  <div class="_2_QraFYR_0">1，作者讲了什么？<br>本章讲了Redis两种持久化机制之一：AOF机制原理<br>aof日志记录了redis所有增删改的操作，保存在磁盘上，当redis宕机，需要恢复内存中的数据时，可以通过读取aop日志恢复数据，从而避免因redis异常导致的数据丢失<br><br>2，作者是怎么把这事给讲明白的？<br>（1）作者先讲述redis宕机会导致内存数据丢失，需要有一种机制在redis重启后恢复数据。<br>（2）介绍了AOF通过记录每一个对redis数据进行增删改的操作日志，可以实现这种功能<br>（2）介绍了AOF的运行机制，数据保存机制，以及由此带来的优点和缺点<br>3，为了讲明白，作者讲了哪些要点，有哪些亮点？<br>（1）亮点：记录操作的时机分为：“写前日志和写后日志”，这个是我之前所不知道的<br>（2）要点1：AOF是写后日志，这样带来的好处是，记录的所有操作命令都是正确的，不需要额外的语法检查，确保redis重启时能够正确的读取回复数据<br>（3）要点2：AOF日志写入磁盘是比较影响性能的，为了平衡性能与数据安全，开发了三种机制：①：立即写入②：按秒写入③：系统写入<br>（4）要点3：AOF日志会变得巨大，所以Redis提供了日志重整的机制，通过读取内存中的数据重新产生一份数据写入日志<br>4，对于作者所讲，我有哪些发散性的思考？<br>作者说系统设计“取舍”二字非常重要，这是我之前未曾意识到的。作者讲了fork子进程机制，是Linux系统的一个能力，在刘超的课中讲过，这鼓舞了我继续学习的信心<br>5，将来有哪些场景，我可以应用上它？<br>目前还没有机会直接操作生产的redis配置，但现在要学习，争取将来可以直接操作<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-14 09:06:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/f5/57/ce10fb1b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>天天向上</span>
  </div>
  <div class="_2_QraFYR_0">什么时候会触发AOF 重写呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 有两个配置项在控制AOF重写的触发时机：<br>1. auto-aof-rewrite-min-size: 表示运行AOF重写时文件的最小大小，默认为64MB<br>2. auto-aof-rewrite-percentage: 这个值的计算方法是：当前AOF文件大小和上一次重写后AOF文件大小的差值，再除以上一次重写后AOF文件大小。也就是当前AOF文件比上一次重写后AOF文件的增量大小，和上一次重写后AOF文件大小的比值。<br><br>AOF文件大小同时超出上面这两个配置项时，会触发AOF重写。<br><br>也可以看下@GEEKBANG_3036760的留言</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-29 00:00:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://wx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLZ09TltdQiboR0kc1Kzp4X7ul2t01sWMG3cA7kT4X1Uvibes5bAEXx72veJ6uEMmUFq8FtEQvwl6FQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>GEEKBANG_3036760</span>
  </div>
  <div class="_2_QraFYR_0">为了减小aof文件的体量，可以手动发送“bgrewriteaof”指令，通过子进程生成更小体积的aof，然后替换掉旧的、大体量的aof文件。<br><br>也可以配置自动触发<br><br>　　1）auto-aof-rewrite-percentage 100<br><br>　　2）auto-aof-rewrite-min-size 64mb<br><br>　　这两个配置项的意思是，在aof文件体量超过64mb，且比上次重写后的体量增加了100%时自动触发重写。我们可以修改这些参数达到自己的实际要求</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-29 11:52:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/a5/30/4be78ce7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>徐鹏</span>
  </div>
  <div class="_2_QraFYR_0">有几个问题想请教一下：<br>1、文中多处提到bgrewriteaof 子进程，这个有点迷糊，主线程fork出来的bgrewriteaof是子线程还是子进程？<br>2、AOF重写会拷贝一份完整的内存数据，这个会导致内存占用直接翻倍吗？<br>3、如果一个key设置了过期时间，在利用AOF文件恢复数据时，key已经过期了这个是如何处理的呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-12 09:01:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/26/38/ef063dc2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Darren</span>
  </div>
  <div class="_2_QraFYR_0">AOF工作原理：<br>1、Redis 执行 fork() ，现在同时拥有父进程和子进程。<br>2、子进程开始将新 AOF 文件的内容写入到临时文件。<br>3、对于所有新执行的写入命令，父进程一边将它们累积到一个内存缓存中，一边将这些改动追加到现有 AOF 文件的末尾,这样样即使在重写的中途发生停机，现有的 AOF 文件也还是安全的。<br>4、当子进程完成重写工作时，它给父进程发送一个信号，父进程在接收到信号之后，将内存缓存中的所有数据追加到新 AOF 文件的末尾。<br>5、搞定！现在 Redis 原子地用新文件替换旧文件，之后所有命令都会直接追加到新 AOF 文件的末尾。<br><br>试着讨论下留言童鞋的几个问题<br><br>一、其中老师在文中提到：“因为主线程未阻塞，仍然可以处理新来的操作。此时，如果有写操作，第一处日志就是指正在使用的 AOF 日志，Redis 会把这个操作写到它的缓冲区。这样一来，即使宕机了，这个 AOF 日志的操作仍然是齐全的，可以用于恢复。”<br>这里面说到 “Redis 会把这个操作写到它的缓冲区，这样一来，即使宕机了，这个 AOF 日志的操作仍然是齐全的”，其实对于有些人有理解起来可能不是那么好理解，因为写入缓冲区为什么还不都是数据；<br><br>我的理解其实这个就是写入缓冲区，只不过是由appendfsync策略决定的，所以说的不丢失数据指的是不会因为子进程额外丢失数据。<br><br>二、AOF重新只是回拷贝引用(指针)，不会拷贝数据本身，因此量其实不大，那写入的时候怎么办呢，写时复制，即新开辟空间保存修改的值，因此需要额外的内存，但绝对不是redis现在占有的2倍。<br><br>三、AOF对于过期key不会特殊处理，因为Redis keys过期有两种方式：被动和主动方式。<br>	当一些客户端尝试访问它时，key会被发现并主动的过期。<br>	<br>	当然，这样是不够的，因为有些过期的keys，永远不会访问他们。 无论如何，这些keys应该过期，所以定时随机测试设置keys的过期时间。所有这些过期的keys将会从删除。<br>	具体就是Redis每秒10次做的事情：<br>		测试随机的20个keys进行相关过期检测。<br>		删除所有已经过期的keys。<br>		如果有多于25%的keys过期，重复步奏1.<br><br>至于课后问题，看了 @与路同飞 童鞋的答案，没有更好的答案，就不回答了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-12 10:41:14</div>
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
  <div class="_2_QraFYR_0">问题一:在AOF重写期间，Redis运行的命令会被积累在缓冲区，待AOF重写结束后会进行回放，在高并发情况下缓冲区积累可能会很大，这样就会导致阻塞，Redis后来通过Linux管道技术让aof期间就能同时进行回放，这样aof重写结束后只需要回放少量剩余的数据即可<br><br>问题二：对于任何文件系统都是不推荐并发修改文件的，例如hadoop的租约机制，Redis也是这样，避免重写发生故障，导致文件格式错乱最后aof文件损坏无法使用，所以Redis的做法是同时写两份文件，最后通过修改文件名的方式，保证文件切换的原子性<br><br>这里需要纠正一下老师前面的口误，就是Redis是通过使用操作系统的fork()方式创建进程，不是线程，也由于这个原因，主进程和fork出来的子进程的资源是不共享的，所以也出现Redis使用pipe管道技术来同步主子进程的aof增量数据</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-12 12:41:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/ae/0c/f39f847a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>D</span>
  </div>
  <div class="_2_QraFYR_0">AOF 是什么的缩写， 还是说就是这个名字？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: AOF的全称是Append Only File，表示文件只能追加写。 Redis记日志时，就是用追加写文件的方式记录写命令操作的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-12 19:26:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/WtHCCMoLJ2DvzqQwPYZyj2RlN7eibTLMHDMTSO4xIKjfKR1Eh9L98AMkkZY7FmegWyGLahRQJ5ibPzeeFtfpeSow/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>脱缰的野马__</span>
  </div>
  <div class="_2_QraFYR_0">文章前面说到redo log日志记录的是修改后的数据，但是在丁奇老师的MySQL实战中讲解的是redo log记录是对数据的操作记录，修改后的数据是保存在内存的change buffer中的</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-12 14:12:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/20/2d/0f/a00f2a32.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Archer30</span>
  </div>
  <div class="_2_QraFYR_0">问题1回答：如果子进程写入事件过长，并且这段事件，会导致AOF重写日志，积累过多，当新的AOF文件完成后，还是需要写入大量AOF重写日志里的内容，可能会导致阻塞。<br><br>问题2回答：我觉得评论区里的大部分回答 防止锁竞争 ，应该是把问题理解错了，父子两个进程本来就没有需要竞争的数据，老师所指的两个日志应该是“AOF缓冲区”和&quot;AOF重写缓冲区&quot;，而不是磁盘上的AOF文件，之所有另外有一个&quot;AOF重写缓冲区&quot;，是因为重写期间，主进程AOF还在继续工作，还是会同步到旧的AOF文件中，同步成功后，“AOF缓冲区”会被清除，会被清除，会被清除！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-12 16:49:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/12/13/e103a6e3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>扩散性百万咸面包</span>
  </div>
  <div class="_2_QraFYR_0">图是不是画错了。为什么主线程和AOF重写缓冲连起来了呢？不是应该bgrewriteaof来写吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 主线程收到写命令后，会把这个写命令写入AOF重写缓冲区，这是由主线程来写的，所以连线在一起了：）<br><br>bgrewriteaof主要是写新日志的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-12 10:33:54</div>
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
  <div class="_2_QraFYR_0">Always，everysec，No，这三种模式就是 CAP 理论的体现。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-23 10:08:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/0c/66/7a2313ba.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ruier</span>
  </div>
  <div class="_2_QraFYR_0">有个Django写的Redis管理小系统，有兴趣的朋友可以看一看，名字叫repoll<br>https:&#47;&#47;github.com&#47;NaNShaner&#47;repoll</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-12 20:59:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/60/4f/db0e62b3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Daiver</span>
  </div>
  <div class="_2_QraFYR_0">AOF重写过程，主线程接收到了新的请求，并将日志写入缓冲区，如果宕机了，缓冲区的内容还是会丢失的。子进程在写日志时，重写缓冲区也是可能会丢失的，为啥说AOF日志还是齐全的，怎么可以用于恢复呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-10 16:59:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/8e/a6/c3286b61.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Java垒墙工程师</span>
  </div>
  <div class="_2_QraFYR_0">试听完了，彻底入坑</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-03 16:47:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2a/25/ca/96734ade.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>随机漫步的傻瓜</span>
  </div>
  <div class="_2_QraFYR_0">感觉老师没有刻意把进程和线程分的特别清楚</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-23 13:13:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTIL1K9WKIkvsdWBwthxz7M08hvqrbeibtzq9rKreZ8wEtHJa2F6x8C1f0dicibMOH6FUW2ayrBYlsEqQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>杨小羊快跑</span>
  </div>
  <div class="_2_QraFYR_0">开启AOF，有可能导致Redis hang住。日志里也有体现：Asynchronous AOF fsync is taking too long (disk is busy?). Writing the AOF buffer without waiting for fsync to complete, this may slow down Redis <br>主要原因：Linux规定执行write(2)时，如果对同一个文件正在执行fdatasync(2)将kernel buffer写入物理磁盘，或者有system wide sync在执行，write(2)会被Block住，整个Redis被Block住。（注：Redis处理完每个事件后都会调用write(2)将变化写入kernel的buffer）</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-23 15:54:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/60/85/f72f1d94.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>与路同飞</span>
  </div>
  <div class="_2_QraFYR_0">1.子线程重新AOF日志完成时会向主线程发送信号处理函数，会完成 （1）将AOF重写缓冲区的内容写入到新的AOF文件中。（2）将新的AOF文件改名，原子地替换现有的AOF文件。完成以后才会重新处理客户端请求。<br>2.不共享AOF本身的日志是防止锁竞争，类似于redis rehash。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-12 08:06:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/64/be/12c37d15.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>CityAnimal</span>
  </div>
  <div class="_2_QraFYR_0">笔记打卡<br><br>    * [ ] 原理<br>        * [ ] 写后日志<br>            * [ ] 好处：<br>                * [ ] 避免出现记录错误命令的情况<br>                * [ ] 不会阻塞当前的写操作<br>            * [ ] 风险<br>                * [ ] 执行完命令后宕机 =&amp;gt; 数据丢失<br>                * [ ] 给下一个操作带来阻塞风险<br>                    * [ ] AOF 日志也是在主线程中执行的<br>        * [ ] 内容：收到的每一条命令，以文本形式保存<br>            * [ ] 命令：set testkey testvalue<br>            * [ ] aof: *3\n$3\nset\n$7\ntestkey\n$9\ntestvalue<br>                * [ ] *3 : 当前命令有三个部分<br>                * [ ] 每部分都是由“$+数字”开头，后面紧跟着具体的命令、键或值<br>    * [ ] 写回策略 &amp;lt;= 配置项 appendfsync<br>        * [ ] Always(同步写回)<br>            * [ ] 基本不丢数据<br>            * [ ] 影响主线程性能<br>        * [ ] Everysec(每秒写回)<br>            * [ ] 性能适中<br>            * [ ] 如果发生宕机，上一秒内未落盘的命令操作仍然会丢失<br>        * [ ] No(操作系统控制的写回)<br>            * [ ] 性能好<br>            * [ ] 一旦宕机前AOF 记录没有落盘，对应的数据就丢失了<br>    * [ ] AOF 文件过大<br>        * [ ] 性能问题<br>            * [ ] 文件系统对文件大小有限制<br>            * [ ] 追加命令记录效率低<br>            * [ ] 故障恢复非常缓慢<br>        * [ ] 方案：AOF 重写机制<br>            * [ ] 原理：根据redis的现状创建一个新的 AOF 文件<br>            * [ ] !!! 非常耗时 (不阻塞主线程)<br>                * [ ] 由后台子进程 bgrewriteaof 来完成的<br>                * [ ] 一个拷贝<br>                    * [ ] 主线程 fork 出 bgrewriteaof 子进程，并将内存拷贝给 bgrewriteaof<br>                * [ ] 两处日志<br>                    * [ ] 正在使用的 AOF 日志<br>                    * [ ] 新的 AOF 重写日志</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-12 19:19:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/ec/cd/52753b9e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Anony</span>
  </div>
  <div class="_2_QraFYR_0">上节说redis单线程处理网络IO和键值对的读写，持久化是由额外线程处理的。那AOF由额外线程处理的话，为什么会影响主线程呢？和主线程之间是什么关系？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-12 09:04:07</div>
  </div>
</div>
</div>
</li>
</ul>