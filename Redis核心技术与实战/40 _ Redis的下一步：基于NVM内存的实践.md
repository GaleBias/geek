<audio title="40 _ Redis的下一步：基于NVM内存的实践" src="https://static001.geekbang.org/resource/audio/e1/29/e1cdc414c24a5f14c47006a774702129.mp3" controls="controls"></audio> 
<p>你好，我是蒋德钧。</p><p>今天这节课是咱们课程的最后一节课了，我们来聊聊Redis的下一步发展。</p><p>这几年呢，新型非易失存储（Non-Volatile Memory，NVM）器件发展得非常快。NVM器件具有容量大、性能快、能持久化保存数据的特性，这些刚好就是Redis追求的目标。同时，NVM器件像DRAM一样，可以让软件以字节粒度进行寻址访问，所以，在实际应用中，NVM可以作为内存来使用，我们称为NVM内存。</p><p>你肯定会想到，Redis作为内存键值数据库，如果能和NVM内存结合起来使用，就可以充分享受到这些特性。我认为，Redis发展的下一步，就可以基于NVM内存来实现大容量实例，或者是实现快速持久化数据和恢复。这节课，我就带你了解下这个新趋势。</p><p>接下来，我们先来学习下NVM内存的特性，以及软件使用NVM内存的两种模式。在不同的使用模式下，软件能用到的NVM特性是不一样的，所以，掌握这部分知识，可以帮助我们更好地根据业务需求选择适合的模式。</p><h2>NVM内存的特性与使用模式</h2><p>Redis是基于DRAM内存的键值数据库，而跟传统的DRAM内存相比，NVM有三个显著的特点。</p><p>首先，<strong>NVM内存最大的优势是可以直接持久化保存数据</strong>。也就是说，数据保存在NVM内存上后，即使发生了宕机或是掉电，数据仍然存在NVM内存上。但如果数据是保存在DRAM上，那么，掉电后数据就会丢失。</p><!-- [[[read_end]]] --><p>其次，<strong>NVM内存的访问速度接近DRAM的速度</strong>。我实际测试过NVM内存的访问速度，结果显示，它的读延迟大约是200~300ns，而写延迟大约是100ns。在读写带宽方面，单根NVM内存条的写带宽大约是1~2GB/s，而读带宽约是5~6GB/s。当软件系统把数据保存在NVM内存上时，系统仍然可以快速地存取数据。</p><p>最后，<strong>NVM内存的容量很大</strong>。这是因为，NVM器件的密度大，单个NVM的存储单元可以保存更多数据。例如，单根NVM内存条就能达到128GB的容量，最大可以达到512GB，而单根DRAM内存条通常是16GB或32GB。所以，我们可以很轻松地用NVM内存构建TB级别的内存。</p><p>总结来说，NVM内存的特点可以用三句话概括：</p><ul>
<li>能持久化保存数据；</li>
<li>读写速度和DRAM接近；</li>
<li>容量大。</li>
</ul><p>现在，业界已经有了实际的NVM内存产品，就是Intel在2019年4月份时推出的Optane AEP内存条（简称AEP内存）。我们在应用AEP内存时，需要注意的是，AEP内存给软件提供了两种使用模式，分别对应着使用了NVM的容量大和持久化保存数据两个特性，我们来学习下这两种模式。</p><p>第一种是Memory模式。</p><p>这种模式是把NVM内存作为大容量内存来使用的，也就是说，只使用NVM容量大和性能高的特性，没有启用数据持久化的功能。</p><p>例如，我们可以在一台服务器上安装6根NVM内存条，每根512GB，这样我们就可以在单台服务器上获得3TB的内存容量了。</p><p>在Memory模式下，服务器上仍然需要配置DRAM内存，但是，DRAM内存是被CPU用作AEP内存的缓存，DRAM的空间对应用软件不可见。换句话说，<strong>软件系统能使用到的内存空间，就是AEP内存条的空间容量</strong>。</p><p>第二种是App Direct模式。</p><p>这种模式启用了NVM持久化数据的功能。在这种模式下，应用软件把数据写到AEP内存上时，数据就直接持久化保存下来了。所以，使用了App Direct模式的AEP内存，也叫做持久化内存（Persistent Memory，PM）。</p><p>现在呢，我们知道了AEP内存的两种使用模式，那Redis是怎么用的呢？我来给你具体解释一下。</p><h2>基于NVM内存的Redis实践</h2><p>当AEP内存使用Memory模式时，应用软件就可以利用它的大容量特性来保存大量数据，Redis也就可以给上层业务应用提供大容量的实例了。而且，在Memory模式下，Redis可以像在DRAM内存上运行一样，直接在AEP内存上运行，不用修改代码。</p><p>不过，有个地方需要注意下：在Memory模式下，AEP内存的访问延迟会比DRAM高一点。我刚刚提到过，NVM的读延迟大约是200~300ns，而写延迟大约是100ns。所以，在Memory模式下运行Redis实例，实例读性能会有所降低，我们就需要在保存大量数据和读性能较慢两者之间做个取舍。</p><p>那么，当我们使用App Direct模式，把AEP内存用作PM时，Redis又该如何利用PM快速持久化数据的特性呢？这就和Redis的数据可靠性保证需求和现有机制有关了，我们来具体分析下。</p><p>为了保证数据可靠性，Redis设计了RDB和AOF两种机制，把数据持久化保存到硬盘上。</p><p>但是，无论是RDB还是AOF，都需要把数据或命令操作以文件的形式写到硬盘上。对于RDB来说，虽然Redis实例可以通过子进程生成RDB文件，但是，实例主线程fork子进程时，仍然会阻塞主线程。而且，RDB文件的生成需要经过文件系统，文件本身会有一定的操作开销。</p><p>对于AOF日志来说，虽然Redis提供了always、everysec和no三个选项，其中，always选项以fsync的方式落盘保存数据，虽然保证了数据的可靠性，但是面临性能损失的风险。everysec选项避免了每个操作都要实时落盘，改为后台每秒定期落盘。在这种情况下，Redis的写性能得到了改善，但是，应用会面临秒级数据丢失的风险。</p><p>此外，当我们使用RDB文件或AOF文件对Redis进行恢复时，需要把RDB文件加载到内存中，或者是回放AOF中的日志操作。这个恢复过程的效率受到RDB文件大小和AOF文件中的日志操作多少的影响。</p><p>所以，在前面的课程里，我也经常提醒你，不要让单个Redis实例过大，否则会导致RDB文件过大。在主从集群应用中，过大的RDB文件就会导致低效的主从同步。</p><p>我们先简单小结下现在Redis在涉及持久化操作时的问题：</p><ul>
<li>RDB文件创建时的fork操作会阻塞主线程；</li>
<li>AOF文件记录日志时，需要在数据可靠性和写性能之间取得平衡；</li>
<li>使用RDB或AOF恢复数据时，恢复效率受RDB和AOF大小的限制。</li>
</ul><p>但是，如果我们使用持久化内存，就可以充分利用PM快速持久化的特点，来避免RDB和AOF的操作。因为PM支持内存访问，而Redis的操作都是内存操作，那么，我们就可以把Redis直接运行在PM上。同时，数据本身就可以在PM上持久化保存了，我们就不再需要额外的RDB或AOF日志机制来保证数据可靠性了。</p><p>那么，当使用PM来支持Redis的持久化操作时，我们具体该如何实现呢？</p><p>我先介绍下PM的使用方法。</p><p>当服务器中部署了PM后，我们可以在操作系统的/dev目录下看到一个PM设备，如下所示：</p><pre><code> /dev/pmem0
</code></pre><p>然后，我们需要使用ext4-dax文件系统来格式化这个设备：</p><pre><code> mkfs.ext4 /dev/pmem0
</code></pre><p>接着，我们把这个格式化好的设备，挂载到服务器上的一个目录下：</p><pre><code>mount -o dax /dev/pmem0  /mnt/pmem0
</code></pre><p>此时，我们就可以在这个目录下创建文件了。创建好了以后，再把这些文件通过内存映射（mmap）的方式映射到Redis的进程空间。这样一来，我们就可以把Redis接收到的数据直接保存到映射的内存空间上了，而这块内存空间是由PM提供的。所以，数据写入这块空间时，就可以直接被持久化保存了。</p><p>而且，如果要修改或删除数据，PM本身也支持以字节粒度进行数据访问，所以，Redis可以直接在PM上修改或删除数据。</p><p>如果发生了实例故障，Redis宕机了，因为数据本身已经持久化保存在PM上了，所以我们可以直接使用PM上的数据进行实例恢复，而不用再像现在的Redis那样，通过加载RDB文件或是重放AOF日志操作来恢复了，可以实现快速的故障恢复。</p><p>当然，因为PM的读写速度比DRAM慢，所以，<strong>如果使用PM来运行Redis，需要评估下PM提供的访问延迟和访问带宽，是否能满足业务层的需求</strong>。</p><p>我给你举个例子，带你看下如何评估PM带宽对Redis业务的支撑。</p><p>假设业务层需要支持1百万QPS，平均每个请求的大小是2KB，那么，就需要机器能支持2GB/s的带宽（1百万请求操作每秒 * 2KB每请求 = 2GB/s）。如果这些请求正好是写操作的话，那么，单根PM的写带宽可能不太够用了。</p><p>这个时候，我们就可以在一台服务器上使用多根PM内存条，来支撑高带宽的需求。当然，我们也可以使用切片集群，把数据分散保存到多个实例，分担访问压力。</p><p>好了，到这里，我们就掌握了用PM将Redis数据直接持久化保存在内存上的方法。现在，我们既可以在单个实例上使用大容量的PM保存更多的业务数据了，同时，也可以在实例故障后，直接使用PM上保存的数据进行故障恢复。</p><h2>小结</h2><p>这节课我向你介绍了NVM的三大特点：性能高、容量大、数据可以持久化保存。软件系统可以像访问传统DRAM内存一样，访问NVM内存。目前，Intel已经推出了NVM内存产品Optane AEP。</p><p>这款NVM内存产品给软件提供了两种使用模式，分别是Memory模式和App Direct模式。在Memory模式时，Redis可以利用NVM容量大的特点，实现大容量实例，保存更多数据。在使用App Direct模式时，Redis可以直接在持久化内存上进行数据读写，在这种情况下，Redis不用再使用RDB或AOF文件了，数据在机器掉电后也不会丢失。而且，实例可以直接使用持久化内存上的数据进行恢复，恢复速度特别快。</p><p>NVM内存是近年来存储设备领域中一个非常大的变化，它既能持久化保存数据，还能像内存一样快速访问，这必然会给当前基于DRAM和硬盘的系统软件优化带来新的机遇。现在，很多互联网大厂已经开始使用NVM内存了，希望你能够关注这个重要趋势，为未来的发展做好准备。</p><h2>每课一问</h2><p>按照惯例，我给你提个小问题，你觉得有了持久化内存后，还需要Redis主从集群吗?</p><p>欢迎在留言区写下你的思考和答案，我们一起交流讨论。如果你觉得今天的内容对你有所帮助，也欢迎你分享给你的朋友或同事。</p>
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
  <div class="_2_QraFYR_0">有了持久化内存，是否还需要 Redis 主从集群？<br><br>肯定还是需要主从集群的。持久化内存只能解决存储容量和数据恢复问题，关注点在于单个实例。<br><br>而 Redis 主从集群，既可以提升集群的访问性能，还能提高集群的可靠性。<br><br>例如部署多个从节点，采用读写分离的方式，可以分担单个实例的请求压力，提升集群的访问性能。而且当主节点故障时，可以提升从节点为新的主节点，降低故障对应用的影响。<br><br>两者属于不同维度的东西，互不影响。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-25 00:10:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/11/ba/2175bc50.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Mr.Brooks</span>
  </div>
  <div class="_2_QraFYR_0">使用NVM，没有了RDB，主从复制对于新添加的机器，是怎么实现的呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这是个好问题。<br><br>增加从节点时，需要把全库数据拷贝到从节点上。我自己现在考虑的有两种方法：一种还是用写前日志，日志拷贝到从节点进行回放，但是这会带来双写的问题。另一种是，主节点在NVM上做快照，但是不写文件，从节点上直接从主节点的NVM上通过远程内存拷贝来实现复制，这个需要基于RDMA来做。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-04 23:37:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/66/34/0508d9e4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>u</span>
  </div>
  <div class="_2_QraFYR_0">老师，比较好奇应用程序是如何基于持久化内存来恢复自身的状态的，还是说应用程序本身也作为持久化的一部分，在重启后就存在于内存中？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个问题很好。目前，应用程序是把本来要保存到磁盘上的数据保存到持久化内存上了，但是应用程序运行时的堆和栈还是在DRAM上，进程重启这些运行时信息就丢了。<br><br>所以，如果想把应用程序本身的运行时状态，例如堆栈等，也保存到持久化内存上，这个需要对操作系统的内核做修改。目前还没有成熟的方案。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-27 10:58:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/e3/8f/77b5a753.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>好好学习</span>
  </div>
  <div class="_2_QraFYR_0">终于学完了。 我真棒！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-15 01:27:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/20/99/5e/33481a74.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Lemon</span>
  </div>
  <div class="_2_QraFYR_0">肯定还是需要的，两者是互补的。<br><br>NVM 给了数据存储方面的新方案，但目前用作 PM 的读写速度比 DRAM 慢，不使用主从集群仍会有明显的访问瓶颈。【过大的实例在主从同步时会有影响（缓存、带宽）】<br><br>而集群是为了高可用，分散了数据的访问和存储，便于拓展与维护。对于单实例而言，即便单实例恢复的再快，挂了对业务仍会有影响。<br><br>感觉 NVM 内存用作 PM 有点像第 28 将的 Pika，如果把 SSD 换为 NVM ，岂不是都再内存中操作？是否可以解决 Pika 操作慢的缺点？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 回答的很好。<br><br>而且对NVM使用的思考非常赞。NVM的读写延迟（几百ns级别）还是要低于SSD的（几十到几百us级别），所以使用NVM替换SSD是可以解决Pika操作慢的问题。<br><br>不过，NVM的每GB成本还是要高于SSD的，这个在实际应用中要考虑。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-25 10:23:04</div>
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
  <div class="_2_QraFYR_0">需要。主从集群 1读写分离，降低实例压力 2数据冗余，防止介质损坏数据丢失</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对主从集群的作用理解到位。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-30 13:47:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/6c/95/bb237f51.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>李梦嘉</span>
  </div>
  <div class="_2_QraFYR_0">老师，请问有AEP方案redis的最佳实践么，最近在调研这方面</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Intel有在推基于AEP的Redis，可以看下<br>https:&#47;&#47;www.intel.com&#47;content&#47;dam&#47;www&#47;public&#47;us&#47;en&#47;documents&#47;solution-briefs&#47;redis-enterprise-brief.pdf<br><br>另外，github上有基于PMem的Redis实现，是基于Redis 4.0实现，有些旧了，不过可以作为一个参考。<br>https:&#47;&#47;github.com&#47;pmem&#47;pmem-redis<br><br>另外，阿里云上的Tair有基于AEP做扩展，参考<br>https:&#47;&#47;developer.aliyun.com&#47;article&#47;776609<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-04 15:05:57</div>
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
  <div class="_2_QraFYR_0">问题：有了持久化内存，是否还需要 Redis 主从集群？<br>解答：需要，主从集群解决的单点故障问题，而且还能起到一定的负载分担。而NVM解决的是数据丢失</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对主从集群作用的理解很到位 :)</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-25 11:48:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/41/38/4f89095b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>写点啥呢</span>
  </div>
  <div class="_2_QraFYR_0">请问蒋老师，Redis将PM用作内存模式的话，是否需要修改Redis代码。我理解内存模式是对程序透明的，虽然PM可以把数据持久化保存，但是如果Redis进程把它看做内存，如果希望进程启动能够自动回复，就会涉及到进程内存空间的恢复，OS里是没有这个功能的，是不是应该需要Redis来做个事情，才可以直接从PM保存的上一次数据中作为新进程的内存空间，而不再需要通过RDB或者AOF来做数据持久化？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-25 09:14:29</div>
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
  <div class="_2_QraFYR_0">老师好，关于mmap映射的问题，没看明白，有几点不清楚，还望帮忙解答：<br>1、将redis内存空间，用mmap映射到PM的ext系统文件后，保持的就是内存信息吧？那进程信息、线程信息、寄存器状态什么的，需要额外保持吗？<br>2、redis挂掉后，重新启动要如何做呢？<br>3、操作系统重启动后，再启动redis时，还能用这个持久化后的文件吗？要如何做呢？<br>4、用集群的时候，节点间复制要如何做呢？<br>感谢！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-25 15:39:43</div>
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
  <div class="_2_QraFYR_0">由于PM的读写速度存在差异，使用读写分离的主从集群，还是有必要，这样可以分担单实例的处理压力，提升redis整体的性能，所以使用主从集群还是非常有必要的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 大流量情况下，单个实例的压力太大，从节点是可以用来分担读压力的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-21 23:41:19</div>
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
  <div class="_2_QraFYR_0">主从集群仍旧需要。<br>单机失踪了持久化内存只能解决单机的存储容量和数据恢复问题。<br>主从集群可以提高访问速度，提高可靠性，做到HA。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-03 11:13:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/8b/4b/15ab499a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>风轻扬</span>
  </div>
  <div class="_2_QraFYR_0">需要主从集群。NVM再厉害，依然会有单点故障，另外单实例的处理效率也是有限的。对于大业务量的业务场景，横向扩容的能力是必须的。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-03 16:27:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/2e/7e/ebc28e10.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>NULL</span>
  </div>
  <div class="_2_QraFYR_0">关于dram, nvm, ssd的区别, 可以看下Intel Optane AEP的介绍, 在1:18秒处 https:&#47;&#47;www.intel.cn&#47;content&#47;www&#47;cn&#47;zh&#47;architecture-and-technology&#47;optane-dc-persistent-memory.html</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-10 20:37:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/47/00/3202bdf0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>piboye</span>
  </div>
  <div class="_2_QraFYR_0">gpu 内存可以加速redis性能吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-19 02:16:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/47/00/3202bdf0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>piboye</span>
  </div>
  <div class="_2_QraFYR_0">nvm后还要记内存日志更好，一些操作执行到中间步骤的时候失败了，重启之后要有办法回滚。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-14 08:37:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/47/00/3202bdf0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>piboye</span>
  </div>
  <div class="_2_QraFYR_0">解决主从一致性的问题，redis就可以当个分布式db了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-12 00:47:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/7b/62/ec94cee4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小彭</span>
  </div>
  <div class="_2_QraFYR_0">完结撒花</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-12 20:52:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/1e/8c/450fe5cb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>花儿少年</span>
  </div>
  <div class="_2_QraFYR_0">NVM 也是有些缺陷的，也可能会导致数据不一致。例如，在redis的新增一个数据节点，那么会有两步操作，1、分配内存，写入数据；2、将此内存地址的指针指向redis管理；而这两步我们知道在很多情况下都可能是乱序执行的，如果第二步先执行，此时机器crash了，redis就指向了一个无效地址。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-09 14:40:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/d9/ff/b23018a6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Heaven</span>
  </div>
  <div class="_2_QraFYR_0">单机版的Redis必然存在着实例服务器宕机的问题,那么Redis主从集群必然有存在的需求<br>不过有了NVM,使用K8S直接管理也是可以的,毕竟绑定了固定的volume,也可以保证持久化和自动重启</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-08 11:40:18</div>
  </div>
</div>
</div>
</li>
</ul>