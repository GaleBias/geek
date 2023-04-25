<audio title="05 _ 内存快照：宕机后，Redis如何实现快速恢复？" src="https://static001.geekbang.org/resource/audio/d0/b5/d01c043fa6903cf91efea4e974e3a8b5.mp3" controls="controls"></audio> 
<p>你好，我是蒋德钧。</p><p>上节课，我们学习了Redis避免数据丢失的AOF方法。这个方法的好处，是每次执行只需要记录操作命令，需要持久化的数据量不大。一般而言，只要你采用的不是always的持久化策略，就不会对性能造成太大影响。</p><p>但是，也正因为记录的是操作命令，而不是实际的数据，所以，用AOF方法进行故障恢复的时候，需要逐一把操作日志都执行一遍。如果操作日志非常多，Redis就会恢复得很缓慢，影响到正常使用。这当然不是理想的结果。那么，还有没有既可以保证可靠性，还能在宕机时实现快速恢复的其他方法呢？</p><p>当然有了，这就是我们今天要一起学习的另一种持久化方法：<strong>内存快照</strong>。所谓内存快照，就是指内存中的数据在某一个时刻的状态记录。这就类似于照片，当你给朋友拍照时，一张照片就能把朋友一瞬间的形象完全记下来。</p><p>对Redis来说，它实现类似照片记录效果的方式，就是把某一时刻的状态以文件的形式写到磁盘上，也就是快照。这样一来，即使宕机，快照文件也不会丢失，数据的可靠性也就得到了保证。这个快照文件就称为RDB文件，其中，RDB就是Redis DataBase的缩写。</p><p>和AOF相比，RDB记录的是某一时刻的数据，并不是操作，所以，在做数据恢复时，我们可以直接把RDB文件读入内存，很快地完成恢复。听起来好像很不错，但内存快照也并不是最优选项。为什么这么说呢？</p><!-- [[[read_end]]] --><p>我们还要考虑两个关键问题：</p><ul>
<li>对哪些数据做快照？这关系到快照的执行效率问题；</li>
<li>做快照时，数据还能被增删改吗？这关系到Redis是否被阻塞，能否同时正常处理请求。</li>
</ul><p>这么说可能你还不太好理解，我还是拿拍照片来举例子。我们在拍照时，通常要关注两个问题：</p><ul>
<li>如何取景？也就是说，我们打算把哪些人、哪些物拍到照片中；</li>
<li>在按快门前，要记着提醒朋友不要乱动，否则拍出来的照片就模糊了。</li>
</ul><p>你看，这两个问题是不是非常重要呢？那么，接下来，我们就来具体地聊一聊。先说“取景”问题，也就是我们对哪些数据做快照。</p><h2>给哪些内存数据做快照？</h2><p>Redis的数据都在内存中，为了提供所有数据的可靠性保证，它执行的是<strong>全量快照</strong>，也就是说，把内存中的所有数据都记录到磁盘中，这就类似于给100个人拍合影，把每一个人都拍进照片里。这样做的好处是，一次性记录了所有数据，一个都不少。</p><p>当你给一个人拍照时，只用协调一个人就够了，但是，拍100人的大合影，却需要协调100个人的位置、状态，等等，这当然会更费时费力。同样，给内存的全量数据做快照，把它们全部写入磁盘也会花费很多时间。而且，全量数据越多，RDB文件就越大，往磁盘上写数据的时间开销就越大。</p><p>对于Redis而言，它的单线程模型就决定了，我们要尽量避免所有会阻塞主线程的操作，所以，针对任何操作，我们都会提一个灵魂之问：“它会阻塞主线程吗?”RDB文件的生成是否会阻塞主线程，这就关系到是否会降低Redis的性能。</p><p>Redis提供了两个命令来生成RDB文件，分别是save和bgsave。</p><ul>
<li>save：在主线程中执行，会导致阻塞；</li>
<li>bgsave：创建一个子进程，专门用于写入RDB文件，避免了主线程的阻塞，这也是Redis RDB文件生成的默认配置。</li>
</ul><p>好了，这个时候，我们就可以通过bgsave命令来执行全量快照，这既提供了数据的可靠性保证，也避免了对Redis的性能影响。</p><p>接下来，我们要关注的问题就是，在对内存数据做快照时，这些数据还能“动”吗? 也就是说，这些数据还能被修改吗？ 这个问题非常重要，这是因为，如果数据能被修改，那就意味着Redis还能正常处理写操作。否则，所有写操作都得等到快照完了才能执行，性能一下子就降低了。</p><h2>快照时数据能修改吗?</h2><p>在给别人拍照时，一旦对方动了，那么这张照片就拍糊了，我们就需要重拍，所以我们当然希望对方保持不动。对于内存快照而言，我们也不希望数据“动”。</p><p>举个例子。我们在时刻t给内存做快照，假设内存数据量是4GB，磁盘的写入带宽是0.2GB/s，简单来说，至少需要20s（4/0.2 = 20）才能做完。如果在时刻t+5s时，一个还没有被写入磁盘的内存数据A，被修改成了A’，那么就会破坏快照的完整性，因为A’不是时刻t时的状态。因此，和拍照类似，我们在做快照时也不希望数据“动”，也就是不能被修改。</p><p>但是，如果快照执行期间数据不能被修改，是会有潜在问题的。对于刚刚的例子来说，在做快照的20s时间里，如果这4GB的数据都不能被修改，Redis就不能处理对这些数据的写操作，那无疑就会给业务服务造成巨大的影响。</p><p>你可能会想到，可以用bgsave避免阻塞啊。这里我就要说到一个常见的误区了，<strong>避免阻塞和正常处理写操作并不是一回事</strong>。此时，主线程的确没有阻塞，可以正常接收请求，但是，为了保证快照完整性，它只能处理读操作，因为不能修改正在执行快照的数据。</p><p>为了快照而暂停写操作，肯定是不能接受的。所以这个时候，Redis就会借助操作系统提供的写时复制技术（Copy-On-Write, COW），在执行快照的同时，正常处理写操作。</p><p>简单来说，bgsave子进程是由主线程fork生成的，可以共享主线程的所有内存数据。bgsave子进程运行后，开始读取主线程的内存数据，并把它们写入RDB文件。</p><p>此时，如果主线程对这些数据也都是读操作（例如图中的键值对A），那么，主线程和bgsave子进程相互不影响。但是，如果主线程要修改一块数据（例如图中的键值对C），那么，这块数据就会被复制一份，生成该数据的副本（键值对C’）。然后，主线程在这个数据副本上进行修改。同时，bgsave子进程可以继续把原来的数据（键值对C）写入RDB文件。</p><p><img src="https://static001.geekbang.org/resource/image/a2/58/a2e5a3571e200cb771ed8a1cd14d5558.jpg?wh=13333*7500" alt="" title="写时复制机制保证快照期间数据可修改"></p><p>这既保证了快照的完整性，也允许主线程同时对数据进行修改，避免了对正常业务的影响。</p><p>到这里，我们就解决了对“哪些数据做快照”以及“做快照时数据能否修改”这两大问题：Redis会使用bgsave对当前内存中的所有数据做快照，这个操作是子进程在后台完成的，这就允许主线程同时可以修改数据。</p><p>现在，我们再来看另一个问题：多久做一次快照？我们在拍照的时候，还有项技术叫“连拍”，可以记录人或物连续多个瞬间的状态。那么，快照也适合“连拍”吗？</p><h2>可以每秒做一次快照吗？</h2><p>对于快照来说，所谓“连拍”就是指连续地做快照。这样一来，快照的间隔时间变得很短，即使某一时刻发生宕机了，因为上一时刻快照刚执行，丢失的数据也不会太多。但是，这其中的快照间隔时间就很关键了。</p><p>如下图所示，我们先在T0时刻做了一次快照，然后又在T0+t时刻做了一次快照，在这期间，数据块5和9被修改了。如果在t这段时间内，机器宕机了，那么，只能按照T0时刻的快照进行恢复。此时，数据块5和9的修改值因为没有快照记录，就无法恢复了。</p><p><img src="https://static001.geekbang.org/resource/image/71/ab/711c873a61bafde79b25c110735289ab.jpg?wh=3292*1244" alt="" title="快照机制下的数据丢失"></p><p>所以，要想尽可能恢复数据，t值就要尽可能小，t越小，就越像“连拍”。那么，t值可以小到什么程度呢，比如说是不是可以每秒做一次快照？毕竟，每次快照都是由bgsave子进程在后台执行，也不会阻塞主线程。</p><p>这种想法其实是错误的。虽然bgsave执行时不阻塞主线程，但是，<strong>如果频繁地执行全量快照，也会带来两方面的开销</strong>。</p><p>一方面，频繁将全量数据写入磁盘，会给磁盘带来很大压力，多个快照竞争有限的磁盘带宽，前一个快照还没有做完，后一个又开始做了，容易造成恶性循环。</p><p>另一方面，bgsave子进程需要通过fork操作从主线程创建出来。虽然，子进程在创建后不会再阻塞主线程，但是，fork这个创建过程本身会阻塞主线程，而且主线程的内存越大，阻塞时间越长。如果频繁fork出bgsave子进程，这就会频繁阻塞主线程了（所以，在Redis中如果有一个bgsave在运行，就不会再启动第二个bgsave子进程）。那么，有什么其他好方法吗？</p><p>此时，我们可以做增量快照，所谓增量快照，就是指，做了一次全量快照后，后续的快照只对修改的数据进行快照记录，这样可以避免每次全量快照的开销。</p><p>在第一次做完全量快照后，T1和T2时刻如果再做快照，我们只需要将被修改的数据写入快照文件就行。但是，这么做的前提是，<strong>我们需要记住哪些数据被修改了</strong>。你可不要小瞧这个“记住”功能，它需要我们使用额外的元数据信息去记录哪些数据被修改了，这会带来额外的空间开销问题。如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/8a/a5/8a1d515269cd23595ee1813e8dff28a5.jpg?wh=4000*1828" alt="" title="增量快照示意图"></p><p>如果我们对每一个键值对的修改，都做个记录，那么，如果有1万个被修改的键值对，我们就需要有1万条额外的记录。而且，有的时候，键值对非常小，比如只有32字节，而记录它被修改的元数据信息，可能就需要8字节，这样的画，为了“记住”修改，引入的额外空间开销比较大。这对于内存资源宝贵的Redis来说，有些得不偿失。</p><p>到这里，你可以发现，虽然跟AOF相比，快照的恢复速度快，但是，快照的频率不好把握，如果频率太低，两次快照间一旦宕机，就可能有比较多的数据丢失。如果频率太高，又会产生额外开销，那么，还有什么方法既能利用RDB的快速恢复，又能以较小的开销做到尽量少丢数据呢？</p><p>Redis 4.0中提出了一个<strong>混合使用AOF日志和内存快照</strong>的方法。简单来说，内存快照以一定的频率执行，在两次快照之间，使用AOF日志记录这期间的所有命令操作。</p><p>这样一来，快照不用很频繁地执行，这就避免了频繁fork对主线程的影响。而且，AOF日志也只用记录两次快照间的操作，也就是说，不需要记录所有操作了，因此，就不会出现文件过大的情况了，也可以避免重写开销。</p><p>如下图所示，T1和T2时刻的修改，用AOF日志记录，等到第二次做全量快照时，就可以清空AOF日志，因为此时的修改都已经记录到快照中了，恢复时就不再用日志了。</p><p><img src="https://static001.geekbang.org/resource/image/e4/20/e4c5846616c19fe03dbf528437beb320.jpg?wh=3508*2183" alt="" title="内存快照和AOF混合使用"></p><p>这个方法既能享受到RDB文件快速恢复的好处，又能享受到AOF只记录操作命令的简单优势，颇有点“鱼和熊掌可以兼得”的感觉，建议你在实践中用起来。</p><h2>小结</h2><p>这节课，我们学习了Redis用于避免数据丢失的内存快照方法。这个方法的优势在于，可以快速恢复数据库，也就是只需要把RDB文件直接读入内存，这就避免了AOF需要顺序、逐一重新执行操作命令带来的低效性能问题。</p><p>不过，内存快照也有它的局限性。它拍的是一张内存的“大合影”，不可避免地会耗时耗力。虽然，Redis设计了bgsave和写时复制方式，尽可能减少了内存快照对正常读写的影响，但是，频繁快照仍然是不太能接受的。而混合使用RDB和AOF，正好可以取两者之长，避两者之短，以较小的性能开销保证数据可靠性和性能。</p><p>最后，关于AOF和RDB的选择问题，我想再给你提三点建议：</p><ul>
<li>数据不能丢失时，内存快照和AOF的混合使用是一个很好的选择；</li>
<li>如果允许分钟级别的数据丢失，可以只使用RDB；</li>
<li>如果只用AOF，优先使用everysec的配置选项，因为它在可靠性和性能之间取了一个平衡。</li>
</ul><h2>每课一问</h2><p>我曾碰到过这么一个场景：我们使用一个2核CPU、4GB内存、500GB磁盘的云主机运行Redis，Redis数据库的数据量大小差不多是2GB，我们使用了RDB做持久化保证。当时Redis的运行负载以修改操作为主，写读比例差不多在8:2左右，也就是说，如果有100个请求，80个请求执行的是修改操作。你觉得，在这个场景下，用RDB做持久化有什么风险吗？你能帮着一起分析分析吗？</p><p>到这里，关于持久化我们就讲完了，这块儿内容是熟练掌握Redis的基础，建议你一定好好学习下这两节课。如果你觉得有收获，希望你能帮我分享给更多的人，帮助更多人解决持久化的问题。</p>
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
  <div class="_2_QraFYR_0">2核CPU、4GB内存、500G磁盘，Redis实例占用2GB，写读比例为8:2，此时做RDB持久化，产生的风险主要在于 CPU资源 和 内存资源 这2方面：<br><br>	a、内存资源风险：Redis fork子进程做RDB持久化，由于写的比例为80%，那么在持久化过程中，“写实复制”会重新分配整个实例80%的内存副本，大约需要重新分配1.6GB内存空间，这样整个系统的内存使用接近饱和，如果此时父进程又有大量新key写入，很快机器内存就会被吃光，如果机器开启了Swap机制，那么Redis会有一部分数据被换到磁盘上，当Redis访问这部分在磁盘上的数据时，性能会急剧下降，已经达不到高性能的标准（可以理解为武功被废）。如果机器没有开启Swap，会直接触发OOM，父子进程会面临被系统kill掉的风险。<br><br>	b、CPU资源风险：虽然子进程在做RDB持久化，但生成RDB快照过程会消耗大量的CPU资源，虽然Redis处理处理请求是单线程的，但Redis Server还有其他线程在后台工作，例如AOF每秒刷盘、异步关闭文件描述符这些操作。由于机器只有2核CPU，这也就意味着父进程占用了超过一半的CPU资源，此时子进程做RDB持久化，可能会产生CPU竞争，导致的结果就是父进程处理请求延迟增大，子进程生成RDB快照的时间也会变长，整个Redis Server性能下降。<br><br>	c、另外，可以再延伸一下，老师的问题没有提到Redis进程是否绑定了CPU，如果绑定了CPU，那么子进程会继承父进程的CPU亲和性属性，子进程必然会与父进程争夺同一个CPU资源，整个Redis Server的性能必然会受到影响！所以如果Redis需要开启定时RDB和AOF重写，进程一定不要绑定CPU。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 除了考虑了内存风险，还考虑了CPU风险。赞！先置个顶 :D<br><br>关于绑核的操作，后面再和Kaito同学聊聊，绑核也有些值得探讨的地方的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-14 01:23:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/a4/2b/3ba9f64b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Devin</span>
  </div>
  <div class="_2_QraFYR_0">老师全文中都是使用的“主线程”而不是“主进程”，评论中大家有的用的是“主线程”有的是“主进程”。请问下老师为啥不是用的“主进程”？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Redis启动后本身是一个进程，它的主体工作（接收请求、服务请求读写数据）也是在这个进程中完成的，咱们有些同学称Redis为主进程是可以的。<br><br>同时，Redis这个进程属于单线程的进程，也就是说进程主体工作没有用多个线程来运行，所以我一般把它也称为主线程，突显它的单线程模式。<br><br>有的程序启动后，会在进程中启动多个线程来处理工作，这个时候我就不会称它为主线程了，因为没有一个线程是单独做主要工作的。<br><br>希望能解答了你的问题。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-24 14:58:47</div>
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
  <div class="_2_QraFYR_0">1，作者讲了什么？<br>作者在本章讲了redis两种持久化方式中的RDB方式<br>2，作者是怎么把这事给讲明白的？<br>为了让大家明白RDB的快照的概念，作者举了拍照片，照合影的例子<br>3，为了讲明白，作者讲了哪些要点，有哪些亮点？<br>（1）亮点1：作者解释快照使用了拍合影的例子，让我很好的理解快照的概念，以及内存数据大小对快照产生的影响<br>（2）要点1：RDB快照，将此时内存中的所有的数据写入磁盘<br>（3）要点2：生成快照有两种方式：sava和bgsava，save是主进程执行，生成时会阻塞redis，只能执行查找。bgsave是由主进程fork出子进程执行，<br>（4）要点3：子进程在被fork处理时，与主进程共享同一份内存，但在生成快照时采取COW机制，确保不会阻塞主进程的数据读写<br>（5）要点4：RDB的执行频率很重要，这会影响到数据的完整性和Redis的性能稳定性。所以4.0后有了aof和rdb混合的数据持久化机制<br>4，对于作者所讲，我有哪些发散性思考？<br>作者开篇提到的两个问题：快照什么数据，快照有何影响，具体的场景，才能讨论出具体的技术方案，我个人认为，脱离场景谈方案是在自嗨<br><br>5，将来有哪些场景，我能够使用到它？<br>我们项目的redis持久化使用的方式就是aof和rdb混合，前一段时间，做过集群升级扩容。把每台8c,30G内存,5主5从，升级改造成为8c,15G内存,15主15从。这样搞主要是因为之前的集群内存占用太高，导致数据持久化失败<br>6，读者评论的收获：<br>定这个专栏，真是觉得捡到宝了，大神@Kaito写的评论实在漂亮，每次都要读好几遍，读完都有赏心悦目的愉悦感，期望自己有一天也可像他那样出色<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-15 11:19:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_Lin</span>
  </div>
  <div class="_2_QraFYR_0">文章中写时复制那里，复制的是主线程修改之前的数据还是主线程修改之后的呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 子进程读到的是主线程修改前的数据。<br><br>我在文章中介绍写时复制时，说法上有点偏它能达到的效果了，可能让大家理解有误了，抱歉！<br><br>我再解释下，文章中说“这块数据就会被复制一份，生成该数据的副本”，这个操作在实际执行过程中，是子进程复制了主线程的页表，所以通过页表映射，能读到主线程的原始数据，而当有新数据写入或数据修改时，主线程会把新数据或修改后的数据写到一个新的物理内存地址上，并修改主线程自己的页表映射。所以，子进程读到的类似于原始数据的一个副本，而主线程也可以正常进行修改。<br><br>希望能解答你的疑惑。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-16 16:20:12</div>
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
  <div class="_2_QraFYR_0">Kaito的回答为啥老让我觉得自己那么菜呢������<br><br>我稍微补充下老师对于 ”混合使用 AOF 日志和内存快照“这块的东西：<br>在redis4.0以前，redis AOF的重写机制是指令整合（老师上一节课已经说过），但是在redis4.0以后，redis的 AOF 重写的时候就直接把 RDB 的内容写到 AOF 文件开头，将增量的以指令的方式Append到AOF，这样做的好处是可以结合 RDB 和 AOF 的优点, 快速加载同时避免丢失过多的数据。当然缺点也是有的， AOF 里面的 RDB 部分就是压缩格式不再是 AOF 格式，可读性较差。Redis服务在读取AOF文件的怎么判断是否AOF文件中是否包含RDB，它会查看是否以 REDIS 开头；人为的看的话，也可以看到以REDIS开头，RDB的文件也打开也是乱码。<br><br>可以通过aof-use-rdb-preamble 配置去设置改功能。<br><br># When rewriting the AOF file, Redis is able to use an RDB preamble in the<br># AOF file for faster rewrites and recoveries. When this option is turned<br># on the rewritten AOF file is composed of two different stanzas:<br>#<br>#   [RDB file][AOF tail]<br>#<br># When loading Redis recognizes that the AOF file starts with the &quot;REDIS&quot;<br># string and loads the prefixed RDB file, and continues loading the AOF<br># tail.<br>aof-use-rdb-preamble yes<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-14 09:26:10</div>
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
  <div class="_2_QraFYR_0">感觉老师这里是不是说的有点问题？<br>１．fork()本身应该是比较快的吧？因为COW的存在，只需要部分数据（局部变量）的复制。真正阻塞的是bgsave在持久化过程中写RDB的时候，因为同时要服务写请求，所以主线程要复制对应内存。<br>２．这个复制为什么不能让fork()出来的子线程去做呢？这样不就不阻塞主线程了吗？<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以看下第10讲的答疑，有对COW的底层机制做了更加详细的介绍。<br><br>第5讲介绍COW时，有点偏于介绍COW的效果了。实际上，fork本身这个操作执行时，内核需要给子进程拷贝主线程的页表。如果主线程的内存大，页表也相应大，拷贝页表耗时长，会阻塞主线程。<br><br>bgsave保存RDB时，如果有写请求，主线程会把新数据写到新的物理地址，此时的阻塞会来自于主线程申请新内存空间以及复制原数据。<br><br>如果是子进程做复制，而主线程直接改数据的话，会有问题：1. 如果子进程还没有把一块数据写入RDB时，主线程就修改了数据，那么就快照完整性就被破坏了；2. 子进程复制数据时，也需要加锁，避免主线程同时修改，如果此时，主线程正好有写请求要处理，主线程同样会被阻塞。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-29 10:43:05</div>
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
  <div class="_2_QraFYR_0">解答：<br>系统的QPS未知，两种情况吧：<br>1. QPS较低，不会有什么问题<br>2. QPS较高，首先由于写多读少，造成更多的写时拷贝，导致更多的内存占用。如果采用增量快照，需要增加额外的内存开销；再则，写RDB文件，OS会分配一些Cache用于磁盘写，进一步加剧内存资源的消耗。<br>由于频繁的写RDB文件，造成较大的磁盘IO开销，消耗CPU</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 非常赞！考虑到了根据不同的QPS进行分析。<br><br>我再提一个维度，你可以考虑下，就是修改的键值对的范围，也就是说写操作是针对一小部分键值对，还是针对大部分键值对的。你觉得这个维度会有影响么？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-15 13:52:23</div>
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
  <div class="_2_QraFYR_0"> 【内存风险】：2 核 CPU、4GB 内存、500GB 磁盘， 2GB数据，在操作系统虚拟内存的支持下，fork出一个子进程会贡献主进程的虚拟内存映射的物理空间，这个是MMU实现的不属于Redis的产物，但是当发生数据修改的时候，MMU会将子进程的物理内存复制独立出来（写时拷贝技术）。在 8:2的独写比例中实际需要的物理内存可能会达到 1.6 +1.6 = 3.2 。假设开启swap的情况下，在物理内存不能满足程序运行的时候，虚拟内存技术会将内存中的数据保存在磁盘上，这样会导致Redis性能下降。<br><br>对于绑核问题，我认同Kaito同学的说法。不过我认为的问题是因为云主机的原因：一般大型服务器是使用QPI总线，NMUA架构。NUMA中，虽然内存直接绑定在CPU上，但是由于内存被平均分配在了各个组上。只有当CPU访问自身直接绑定的内存对应的物理地址时，才会有较短的响应时间。而如果需要访问其他CPU 绑定的内存的数据时，就需要通过特殊的通道访问，响应时间就相比之前变慢了（甚至有些服务器宁愿使用swap也不走特殊通道），假如当前云主机比较繁忙的情况下，这样就会导致性能下降。（大部分互联网公司都使用了服务器虚拟化技术）</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-17 01:19:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83eppZl39m2knwLH6PIia5YQTOWSOTGhy8ZZAutUIrxKOYFCtLLLYb1OZvIVVLzL7Y8eglKFe4Sib9D7g/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>漫步oo0云端</span>
  </div>
  <div class="_2_QraFYR_0">我想提一个傻问题，我作为初学者想问，如果redis服务挂了，备份有什么用？能恢复的前提不是服务还存活吗？难道服务挂了会自动拉起服务？自动还原吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果服务挂了，我们可以让Redis实例自动重启。此时，如果没有数据备份的话，再启动时，所有的数据都需要重新写入，这个过程会比较耗时。而如果有备份的话，Redis再启动后，可以直接读入备份数据，对于这些数据的读写操作就可以很快服务了。不知道有没有解答你的疑惑。<br><br>愿意提出来的问题都是好问题哈 :)</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-14 07:06:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/54/9a/76c0af70.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>每天晒白牙</span>
  </div>
  <div class="_2_QraFYR_0">Redis持久化<br>AOF<br>AOF是一种通过记录操作命令的的方式来达到持久化的目的，正是因为记录操作命令而不是数据，所以在恢复数据的时候需要 redo 这些操作命令(当然 AOF 也有重写机制从而使命令减少)，如果操作日志太多，恢复过程就会很慢，可能会影响 Redis 的正常使用<br><br>RDB<br>RDB 是一种内存快照，把内存的数据在某一时刻的状态记录下来，所以通过 RDB 恢复数据只需要把 RDB 文件读入内存就可以恢复数据(具体实现细节还没去了解)<br><br>但这里有两个需要注意的问题<br>1.对哪些数据做快照，关系到快照的执行顺序<br>2.做快照时，还能对数据做增删改吗？这会关系到 Redis 是否阻塞，因为如果在做快照时，还能对数据做修改，说明 Redis 还能处理写请求，如果不能对数据做修改，就不能处理写请求，要等执行完快照才能处理写请求，这样会影响性能<br><br>来看第一个问题<br>RDB 是对全量数据需要快照，全量数据会使 RDB 文件大，发文件写到磁盘就会耗时，因为 Redis 是单线程，会不会阻塞主线程？(这一点始终都要考虑的点)<br>Redis 实现 RDB 的方式有两种<br>①save:在主线程中执行，会导致阻塞<br>②bgsave:创建子线程来执行，不会阻塞，这是默认的<br>所以可以使用 bgsave 来对全量内存做快照，不影响主进程<br><br>来看第二个问题<br>在做快照时，我们是不希望数据被修改的，但是如果在做快照过程中，Redis 不能处理写操作，这对业务是会造成影响的，但上面刚说完 bgsave 进行快照是非阻塞的呀，这是一个常见的误区：避免阻塞和正常的处理写操作不是一回事。用 bgsave 时，主线程是没有被阻塞，可以正常处理请求，但为了保证快照的完整性，只能处理读请求，因为不能修改正在执行快照的数据。显然为了快照而暂停写是不能被接受的。Redis 采用操作系统提供的写时复制技术（Copy-On-Write 即 COW），在执行快照的同时，可以正常的处理写操作<br>bgsave 子线程是由主线程 fork 生成，可以共享主线程的所有内存数据，所以 bgsave 子线程是读取主线程的内存数据并把他们写入 RDB 文件的<br>如果主线程对这些数据只执行读操作，两个线程是互不影响的。如果主线程要执行写造作，那么这个数据就会被复制一份，生成一个副本，然后 bgsave 子线程会把这个副本数据写入 RDB 文件，这样主线程就可以修改原来的数据了。这样既保证了快照的完整性，也保证了正常的业务进行<br><br>那如何更好的使用 RDB 呢？<br>RDB 的频率不好把握，如果频率太低，两次快照间一旦有宕机，就可能会丢失很多数据。如果频率太高，又会产生额外开销，主要体现在两个方面<br>①频繁将全量数据写入磁盘，会给磁盘带来压力，多个快照竞争有效的磁盘带宽<br>②bgsave 子线程是通过 fork 主线程得来，前面也说了，bgsave 子线程是不会阻塞主线程的，但 fork 这个创建过程是会阻塞主线程的，而且主线程内存越大，阻塞时间越长<br><br>最好的方法是全量快照+增量快照，即进行一次 RDB 后，后面的增量修改用 AOF 记录两次全量快照之间的写操作，这样可以避免频繁 fork 对主线程的影响。同时避免 AOF 文件过大，重写的开销</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-17 10:03:38</div>
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
  <div class="_2_QraFYR_0">很奇怪，对于RDB和AOF混合搭配的策略，为什么不把AOF应用于RDB生成增量快照呢？而非要再次生成全量快照？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-14 16:43:14</div>
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
  <div class="_2_QraFYR_0">1.写太多，COW需要复制的东西太多，内存占用问题；<br>2.CPU太少，redis后台还有很多线程在后台工作，会产生CPU竞争。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-14 10:20:55</div>
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
  <div class="_2_QraFYR_0">老师，请教一下，bgsave命令只能是手动执行么？没配置中只看到了save,没有bgsave的配置</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以自动执行。Redis配置文件中的save选项是用来配置bgsave的触发条件的，例如<br>save 60 10000<br><br>如果60s内有至少10000个键值对的修改，就会自动触发bgsave了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-29 17:39:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/8e/49/10ef002d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>周翔在山麓（Xiang Zhou）</span>
  </div>
  <div class="_2_QraFYR_0">这一讲真的很好, aof 相当于数据库的 binlog, rdb 相当于redo log. 知识互相映射, 加强了学习. 我又看了一遍mysql 的数据恢复机制. 同学们还记得吗?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 学习时把知识贯通起来理解是个好方法！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-01 17:50:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/24/d7/146f484b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小宇子2B</span>
  </div>
  <div class="_2_QraFYR_0">做RDB期间是写时复制的 2GB的数据 80%都是写请求 也就是大概要复制出来1.6GB数据，加上本身数据2GB ，已经达到3.6GB，去掉操作系统本身的内存占用 机器所剩内存已经不多了 容易发生OOM</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 分析的不错，这种情况下，内存的潜在压力风险比较大。<br><br>另外，Kaito同学还分析了CPU资源的风险，也可以看看。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-14 00:28:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/20/21/aa/66d71e6b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>benzhang</span>
  </div>
  <div class="_2_QraFYR_0">你好，关于这一点我有点疑问<br>“另一方面，bgsave 子进程需要通过 fork 操作从主线程创建出来。虽然，子进程在创建后不会再阻塞主线程，但是，fork 这个创建过程本身会阻塞主线程，而且主线程的内存越大，阻塞时间越长。如果频繁 fork 出 bgsave 子进程，这就会频繁阻塞主线程了。那么，有什么其他好方法吗？”<br>我一直以为bgsave这个子进程只需要创建一次而已。从上面这段话的意思看，也就是说每写一次快照就需要fork一个bgsave子进程吗？如果是的话，哪如何解决写入冲突呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-07 10:49:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJnugUNWBtcszhJg3Q0hqEMSHftKco2TqCG78blZ3ibjncjZ64NbibGia5l4NB0DUibIq0BCZ03JvkoNA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek__e15575f5b6ec</span>
  </div>
  <div class="_2_QraFYR_0">最近遇到个问题，docker安装的redis，运行一段时间之后，日志显示会出现几十次的DB saved on disk<br>时间间隔几百毫秒到几秒之间 然后会出现redis部分数据丢失  完全不知道怎么排查 有没有大佬能够解惑</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-20 19:26:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKiauonyMORA2s43W7mogGDH4WYjW0gBJtYmUa9icTB6aMPGqibicEKlLoQmLKLWEctwHzthbTZkKR20w/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Spring4J</span>
  </div>
  <div class="_2_QraFYR_0">由于修改操作占大部分比例，为了尽可能保证宕机时数据的完整性，快照的间隔就不能太长，而间隔太短又会带来很多的性能开销，所以对于这种特点的数据，不适合使用RDB的持久化方式</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不错，不错！可以再想想看，在问题里的云主机上，哪怕我们先不考虑快照频率问题，单就做一次快照本身，是否可能还会有其他问题？：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-14 11:13:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2c/53/12/ed39ec11.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>go big</span>
  </div>
  <div class="_2_QraFYR_0">哈哈</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-19 20:36:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2b/54/a8/3b334406.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>冷山</span>
  </div>
  <div class="_2_QraFYR_0">哎呀，总结一下：<br>不管是RDB还是AOF。他们都需要fork出一个子进程进行日志的操作。<br>而两者使用子进程来操作日志时、都存在两个阻塞点：<br>1. fork的过程中、如果实例数据比较多，主线程的页表就比较大、复制主线程的页表给子进程这个操作就比较耗时（这里可以理解为主线程把自己的内存空间中的数据遗传给子进程使用），会阻塞主线程。<br><br>2. RDB在生成日志和AOF在重写时，为了支持主线程同时可以对原有数据进行写操作：两者都借助了操作系统的：写时复制机制。 写时复制机制就是主线程在修改原有数据时，它会动用遗传给子进程的数据，而是拷贝一份数据副本（以页为单位拷贝），然后对这个副本进行操作。  假设主线程在子进程使用的原有数据A上做修改，如果此时数据A还没有被AOF重写或者被生成RDB快照，那么就会导致子进程重写&#47;生成RDB快照的过程失败，或者说是脏数据嘛。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-30 16:07:12</div>
  </div>
</div>
</div>
</li>
</ul>