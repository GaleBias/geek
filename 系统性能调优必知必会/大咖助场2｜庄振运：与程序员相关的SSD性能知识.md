<audio title="大咖助场2｜庄振运：与程序员相关的SSD性能知识" src="https://static001.geekbang.org/resource/audio/03/48/037884390233cb1cc6b8bbaa3f348b48.mp3" controls="controls"></audio> 
<p>你好，我是庄振运。我是<a href="https://time.geekbang.org/column/intro/100041101">《性能工程高手课》</a>的专栏作者，很荣幸受邀来到陶辉老师的专栏做一期分享。今天我们来讲一点SSD相关的性能知识。SSD（Solid State Drive）是硬盘的一种，有时候也叫Flash或者固态硬盘。</p><p>最近几年，SSD的发展和演化非常迅速。随着市场规模的增大和技术的进步，SSD的价格也大幅度降低了。在很多实时的后台系统中，SSD几乎已经成了标准配置了。所以了解它的机制和性能，对你的工作会很有益处的。</p><p>相对于传统硬盘HDD（Hard Disk Drive），SSD有完全不同的内部工作原理和全新的特性。有些机制不太容易理解，而且根据你工作的领域，需要理解的深度也不一样。所以，我把这节课的内容按照由浅入深的原则分成了三个层次。</p><p>第一个层次是关注SSD的外部性能指标；第二个层次是了解它的内部工作机制；第三个层次是设计对SSD友好的应用程序。</p><h2>比HDD更快的硬盘</h2><p>很多人对传统硬盘了解较多，毕竟这种硬盘在业界用了好几十年了，很多教科书里面都讲述过。所以，对SSD的性能，我先用对比的方式带你看看它们的外部性能指标和特性。</p><p>一个硬盘的性能最主要体现在这三个指标：IOPS，带宽/吞吐率和访问延迟。<strong>IOPS</strong> (Input/Output Per Second) ，即每秒钟系统能处理的读写请求数量。<strong>访问延迟</strong>，指的是从发起IO请求到存储系统把IO处理完成的时间间隔。<strong>吞吐率</strong>（Throughput）或者带宽（Bandwidth），衡量的是实际数据传输速率。</p><!-- [[[read_end]]] --><p>对于传统硬盘我们应该比较熟悉它的内部是如何操作的，简单来说，当应用程序发出硬盘IO请求后，这个请求会进入硬盘的IO队列。当轮到这个IO来存取数据时，磁头需要机械运动到数据存放的位置，这就需要磁头寻址到相应的磁道和旋转到相应的扇区，然后才是数据的传输。对于一块普通硬盘而言，随机IO读写延迟就是8毫秒左右，IO带宽大约每秒100MB，而随机IOPS一般是100左右。</p><p>SSD的种类很多，按照技术来说有单层和多层。按照质量和性能来分，有企业级和普通级。根据安装的接口和协议来分，有SAS, SATA, PCIe和NVMe等。</p><p>我用一张表格来对比一下HDD和SSD的三大性能指标的差异。这里考虑比较流行的NVMe协议的SSD。你可以看到，SSD的随机IO延迟比传统硬盘快百倍以上，一般在微妙级别；IO带宽也高很多倍，可以达到每秒几个GB；随机IOPS更是快了上千倍，可以达到几十万。</p><p><img src="https://static001.geekbang.org/resource/image/7c/04/7c0d537e9334e1d622d40f28168e1c04.jpg?wh=1742*432" alt=""></p><h2>SSD的性能特性和机制</h2><p>SSD的内部工作方式和HDD大相径庭，我们先了解几个概念。</p><p><strong>单元（Cell）</strong>、<strong>页面（Page）</strong>、<strong>块（Block）</strong>。当今的主流SSD是基于NAND的，它将数字位存储在单元中。每个SSD单元可以存储一位或多位。对单元的每次擦除都会降低单元的寿命，所以单元只能承受一定数量的擦除。单元存储的位数越多，制造成本就越少，SSD的容量也就越大，但是耐久性（擦除次数）也会降低。</p><p>一个页面包括很多单元，典型的页面大小是4KB，页面也是要读写的最小存储单元。SSD上没有“重写”操作，不像HDD可以直接对任何字节重写覆盖。一个页面一旦写入内容后就不能进行部分重写，必须和其它相邻页面一起被整体擦除重置。</p><p>多个页面组合成块。一个块的典型大小为512KB或1MB，也就是大约128或256页。块是擦除的基本单位，每次擦除都是整个块内的所有页面都被重置。</p><p>了解完以上几个基础概念，我们重点看看<strong>IO和垃圾回收(Garbage Collection)</strong> 。对SSD的IO共有三种类型：读取、写入和擦除。读取和写入以页为单位。IO写入的延迟具体取决于磁盘的历史状态，因为如果SSD已经存储了许多数据，那么对页的写入就经常需要移动已有的数据。一般的读写延迟都很低，在微秒级别，远远低于HDD。擦除是以块为单位的。擦除速度相对很慢，通常为几毫秒。所以对同步的IO，发出IO的应用程序可能会因为块的擦除，而经历很大的写入延迟。为了尽量地减少这样的场景，保持空闲块的阈值对于快速的写响应是很有必要的。SSD的垃圾回收（GC）的目的就在于此。GC可以回收用过的块，这样可以确保以后的页写入可以快速分配到一个全新的页。</p><p><img src="https://static001.geekbang.org/resource/image/4b/70/4b3705248ccb0144516a77849c17dd70.png?wh=508*335" alt=""></p><p><strong>写入放大（Write Amplification, or WA)。</strong> 这是SSD相对于HDD的一个缺点，即实际写入SSD的物理数据量，有可能是应用层写入数据量的多倍。一方面，页级别的写入需要移动已有的数据来腾空页面。另一方面，GC的操作也会移动用户数据来进行块级别的擦除。所以对SSD真正的写操作的数据可能比实际写的数据量大，这就是写入放大。一块SSD只能进行有限的擦除次数，也称为编程/擦除（P/E）周期，所以写入放大效用会缩短SSD的寿命。</p><p><strong>耗损平衡(Wear Leveling) 。</strong>对每一个块而言，一旦达到最大数量，该块就会死亡。对于SLC块，P/E周期的典型数目是十万次；对于MLC块，P/E周期的数目是一万；而对于TLC块，则可能是几千。为了确保SSD的容量和性能，我们需要在擦除次数上保持平衡，SSD控制器具有这种“耗损平衡”机制可以实现这一目标。在损耗平衡期间，数据在各个块之间移动，以实现均衡的损耗，这种机制也会对前面讲的写入放大推波助澜。</p><h2>设计对SSD友好的程序</h2><p>SSD的IO性能相对于HDD来说，IOPS和访问延迟提升了上千倍，吞吐率也是几十倍，但是SSD的缺点也很明显，有三个：贵、容量小、易损耗。随着技术的发展，这三个缺点近几年在弱化。</p><p>现在越来越多的系统采用SSD来减轻应用程序的IO性能瓶颈。许多部署的结果显示，与HDD相比，SSD带来了极大的应用程序的性能提升。但是，在大多数部署方案中，SSD仅被视为“更快的HDD”，SSD的潜力并未得到充分利用。尽管使用SSD作为存储时应用程序可以获得更好的性能，但是这些收益主要归因于SSD提供的更高的IOPS和带宽。</p><p>进一步讲，如果应用程序的设计充分考虑了SSD的内部机制，从而设计为对SSD友好，则可以更大程度地优化SSD，从而进一步提高应用程序性能，也可以延长SSD的寿命而降低运用成本。接下来我们就看看如何在应用程序层进行一系列SSD友好的设计更改。</p><h3>为什么要设计SSD友好的软件和应用程序？</h3><p>SSD友好的程序可以获得三种好处：</p><ul>
<li>提升应用程序性能；</li>
<li>提高SSD的 IO效率；</li>
<li>延长SSD的寿命。</li>
</ul><p>我分别说明一下。</p><p><strong>更好的应用程序性能。</strong>尽管从HDD迁移到SSD通常意味着更好的应用程序性能，这主要是得益于SSD的IO性能更好，但在不更改应用程序设计的情况下简单地采用SSD可能无法获得最佳性能。我们曾经有一个应用程序就是如此。该应用程序需要不断写入文件以保存数据，主要性能瓶颈就是硬盘IO。使用HDD时，最大应用程序吞吐量为每秒142个查询（QPS）。无论对应用程序设计进行各种更改还是调优，这都是可以获得的最好性能。</p><p>当迁移到具有相同应用程序的SSD时，吞吐量提高到2万QPS，速度提高了140倍。这主要来自SSD提供的更高IOPS。在对应用程序设计进行进一步优化使其对SSD友好之后，吞吐量提高到10万QPS，与原来的简单设计相比，提高了4倍。</p><p>这其中的秘密就是使用多个并发线程来执行IO，这就利用了SSD的内部并行性。记住，多个IO线程对HDD毫无益处，因为HDD只有一个磁头。</p><p><strong>更高效的存储IO。</strong> SSD上的最小内部IO单元是一页，比如4KB大小。因此对SSD的单字节读/写必须在页面级进行。应用程序对SSD的写操作可能会导致对SSD上的物理写操作变大，这就是“写放大（WA）”。因为有这个特性，如果应用程序的数据结构或IO对SSD不友好，就会让写放大效果无谓的更大，导致SSD的IO不能被充分利用。</p><p><strong>更长的使用寿命。</strong>SSD会磨损，因为每个存储单元只能维持有限数量的写入擦除周期。实际上，SSD的寿命取决于四个因素：SSD大小、最大擦除周期数、写入放大系数和应用程序写入速率。例如，假设有一个1TB大小的SSD，一个写入速度为100MB每秒的应用程序和一个擦除周期数为1万的SSD。当写入放大倍数为4时，SSD仅可持续10个月。具有3千个擦除周期和写入放大系数为10的SSD只能使用一个月。鉴于SSD相对较高的成本，我们很希望这些应用程序对SSD友好，从而延长SSD的使用寿命。</p><h3>SSD友好的设计原则</h3><p>在设计程序的时候，我们可以把程序设计成对SSD友好，以获得前面提到的三种好处。那有哪些对SSD友好的程序设计呢？我这里总结了四个原则，大体上分为两类：数据结构和IO处理。</p><p><strong>1.数据结构：避免就地更新的优化</strong></p><p>传统HDD的寻址延迟很大，因此，使用HDD的应用程序通常经过优化，以执行不需要寻址的就地更新（比如只在一个文件后面写入）。如下图所示，执行随机更新时，吞吐量一般只能达到约170QPS；而对于同一个HDD，就地更新可以达到280QPS，远高于随机更新（如下左图所示）。</p><p><img src="https://static001.geekbang.org/resource/image/42/fb/42339220b197186548330ayy0b08d9fb.jpg?wh=6638*2112" alt=""></p><p>在设计与SSD配合使用的应用程序时，这些考虑就不再有效了。对SSD而言，随机读写和顺序读写性能类似，就地更新不会获得任何IOPS优势。此外，就地更新实际上会导致SSD性能下降。原因是包含数据的SSD页面无法直接重写，因此在更新存储的数据时，必须先将相应的SSD页面读入SSD缓冲区，然后将数据写入干净页面。 SSD中的“读取-修改-写入”过程与HDD上的直接“仅写入”行为形成鲜明对比。相比之下，SSD上的随机更新就不会引起读取和修改步骤（即仅仅“写入”），因此速度更快。使用SSD，以上相同的应用程序可以通过随机更新或就地更新来达到大约2万QPS（如上右图所示）。</p><p><strong>2.数据结构：将热数据与冷数据分开</strong></p><p>对于几乎所有处理存储的应用程序，磁盘上存储的数据的访问概率均不相同。我们考虑这样一个需要跟踪用户活动的社交网络应用程序，对于用户数据存储，简单的解决方案是基于用户属性（例如注册时间）将所有用户压缩在同一位置（例如某个SSD上的文件），以后需要更新热门用户的活动时，SSD需要在页面级别进行访问（即读取/修改/写入）。因此，如果用户的数据大小小于一页，则附近的用户数据也将一起访问。如果应用程序其实并不需要附近用户的数据，则额外的数据不仅会浪费IO带宽，而且会不必要地磨损SSD。</p><p>为了缓解这种性能问题，在将SSD用作存储设备时，应将热数据与冷数据分开。以不同级别或不同方式来进行分隔，例如，存到不同的文件，文件的不同部分或不同的表。</p><p><strong>3. IO处理：避免长而繁重的写入</strong></p><p>SSD通常具有GC机制，不断地回收存储块以供以后使用。 GC可以后台或前台方式工作。</p><p>SSD控制器通常保持一个空闲块的阈值。每当可用块数下降到阈值以下时，后台GC就会启动。由于后台GC是异步发生的（即非阻塞），因此它不会影响应用程序的IO延迟，但是，如果块的请求速率超过了GC速率，并且后台GC无法跟上，则将触发前台GC。</p><p>在前台GC期间，必须即时擦除（即阻塞）每个块以供应用程序使用，这时发出写操作的应用程序所经历的写延迟会受到影响。具体来说，释放块的前台GC操作可能会花费数毫秒以上的时间，从而导致较大的应用程序IO延迟。出于这个原因，最好避免进行长时间的大量写入操作，以免永远不使用前台GC。</p><p><strong>4. IO处理：避免SSD存储太满</strong></p><p>SSD磁盘存储太满会影响写入放大系数和GC导致的写入性能。在GC期间，需要擦除块以创建空闲块。擦除块前需要移动并保留有效数据才能获得空闲块。有时为了获得一个空闲块，我们需要压缩好几个存储块。每个空闲块的生产需要压缩的块数取决于磁盘的空间使用率。</p><p>假设磁盘满百分比平均为A％，要释放一块，则需要压缩1 /（1-A％）块。显然，SSD的空间使用率越高，将需要移动更多的块以释放一个块，这将占用更多的资源并导致更长的IO等待时间。例如，如果A=80％，则大约移动五个数据块以释放一个块；当A=95％时，将移动约20个块。</p><h2>总结</h2><p>各种存储系统的基础是传统硬盘或者固态硬盘，固态硬盘SSD的IO性能比传统硬盘高很多。如果系统对IOPS或者延迟要求很高，一般都采用SSD。</p><p>现在已经有很多专门针对SSD来设计的文件系统、数据库系统和数据基础架构，它们的性能比使用HDD的系统都有了很大的提升。</p><p>SSD有不同于HDD工作原理，所以在进行应用程序设计的时候，如果可以做到对SSD友好，那么就可以充分发挥SSD的全部性能潜能，应用程序的性能会进一步提高。你可以参考我们今天总结的四个设计原则进行实践。</p><h2>思考题</h2><p>最后给你留几道思考题。你们公司里面，有哪些后台服务和应用是使用SSD作为存储的？而对这些使用SSD的系统，有没有充分考虑SSD的特性，做深层的优化呢（比如降低损耗）？</p><p>感谢阅读，如果今天的内容让你有所收获，欢迎把它分享给你的朋友。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/64/9b/d1ab239e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>J.Smile</span>
  </div>
  <div class="_2_QraFYR_0">公司目前kafka使用的就是ssd</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-17 22:06:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/4d/49/28e73b9c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>明翼</span>
  </div>
  <div class="_2_QraFYR_0">老师，你好，文中的就地更新和随机更新怎么区分，不太理解，能不能举个例子</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-11 09:39:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/l4nngwyggBGqeMXC0micwO8bM1hSttgQXa1Y5frJSqWa8NibDhia5icwPcHM5wOpV3hfsf0UicDY0ypFqnQ3iarG0T1w/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Trident</span>
  </div>
  <div class="_2_QraFYR_0">之前公司采用SSD存储es的热点数据，但是深层次优化没有做</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-11 08:26:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/56/75/6bf38a1e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>坤哥</span>
  </div>
  <div class="_2_QraFYR_0">老师，不明白就地更新会引起读取-修改-写入过程，随机更新仅仅写入过程。随机更新不会碰到已写的页面吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 代庄老师回答下：概率上的差别，就地更新远大于随机更新，特别是SSD空间使用率不满的情况下</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-09 21:51:16</div>
  </div>
</div>
</div>
</li>
</ul>