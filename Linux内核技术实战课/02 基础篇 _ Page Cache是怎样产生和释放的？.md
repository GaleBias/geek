<audio title="02 基础篇 _ Page Cache是怎样产生和释放的？" src="https://static001.geekbang.org/resource/audio/e6/a8/e6f7e83d26b67cace8722a729fa02ea8.mp3" controls="controls"></audio> 
<p>你好，我是邵亚方。</p><p>上一讲，我们主要讲了“什么是Page Cache”（What），“为什么需要Page Cache”（Why），我们这堂课还需要继续了解一下“How”：<strong>也就是Page Cache是如何产生和释放的。</strong></p><p>在我看来，对Page Cache的“What-Why-How”都有所了解之后，你才会对它引发的问题，比如说Page Cache引起的load飙高问题或者应用程序的RT抖动问题更加了然于胸，从而防范于未然。</p><p>其实，Page Cache是如何产生和释放的，通俗一点的说就是它的“生”（分配）与“死”（释放），即Page Cache的生命周期，那么接下来，我们就先来看一下它是如何“诞生”的。</p><h2>Page Cache是如何“诞生”的？</h2><p>Page Cache的产生有两种不同的方式：</p><ul>
<li>Buffered I/O（标准I/O）；</li>
<li>Memory-Mapped I/O（存储映射I/O）。</li>
</ul><p>这两种方式分别都是如何产生Page Cache的呢？来看下面这张图：</p><p><img src="https://static001.geekbang.org/resource/image/4e/52/4eb820e15a5560dee4b847227ee75752.jpg?wh=3957*2600" alt="" title="Page Cache产生方式示意图"></p><p>从图中你可以看到，虽然二者都能产生Page Cache，但是二者的还是有些差异的：</p><p>标准I/O是写的(write(2))用户缓冲区(Userpace Page对应的内存)，然后再将用户缓冲区里的数据拷贝到内核缓冲区(Pagecache Page对应的内存)；如果是读的(read(2))话则是先从内核缓冲区拷贝到用户缓冲区，再从用户缓冲区读数据，也就是buffer和文件内容不存在任何映射关系。</p><!-- [[[read_end]]] --><p>对于存储映射I/O而言，则是直接将Pagecache Page给映射到用户地址空间，用户直接读写Pagecache Page中内容。</p><p>显然，存储映射I/O要比标准I/O效率高一些，毕竟少了“用户空间到内核空间互相拷贝”的过程。这也是很多应用开发者发现，为什么使用内存映射I/O比标准I/O方式性能要好一些的主要原因。</p><p>我们来用具体的例子演示一下Page Cache是如何“诞生”的，就以其中的标准I/O为例，因为这是我们最常使用的一种方式，如下是一个简单的示例脚本：</p><pre><code>#!/bin/sh

#这是我们用来解析的文件
MEM_FILE=&quot;/proc/meminfo&quot;

#这是在该脚本中将要生成的一个新文件
NEW_FILE=&quot;/home/yafang/dd.write.out&quot;

#我们用来解析的Page Cache的具体项
active=0
inactive=0
pagecache=0

IFS=' '

#从/proc/meminfo中读取File Page Cache的大小
function get_filecache_size()
{
        items=0
        while read line
        do
                if [[ &quot;$line&quot; =~ &quot;Active:&quot; ]]; then
                        read -ra ADDR &lt;&lt;&lt;&quot;$line&quot;
                        active=${ADDR[1]}
                        let &quot;items=$items+1&quot;
                elif [[  &quot;$line&quot; =~ &quot;Inactive:&quot; ]]; then
                        read -ra ADDR &lt;&lt;&lt;&quot;$line&quot;
                        inactive=${ADDR[1]}
                        let &quot;items=$items+1&quot;
                fi  


                if [ $items -eq 2 ]; then
                        break;
                fi  
        done &lt; $MEM_FILE
}

#读取File Page Cache的初始大小
get_filecache_size
let filecache=&quot;$active + $inactive&quot;

#写一个新文件，该文件的大小为1048576 KB
dd if=/dev/zero of=$NEW_FILE bs=1024 count=1048576 &amp;&gt; /dev/null

#文件写完后，再次读取File Page Cache的大小
get_filecache_size

#两次的差异可以近似为该新文件内容对应的File Page Cache
#之所以用近似是因为在运行的过程中也可能会有其他Page Cache产生
let size_increased=&quot;$active + $inactive - $filecache&quot;

#输出结果
echo &quot;File size 1048576KB, File Cache increased&quot; $size_inc
</code></pre><p>在这里我提醒你一下，在运行该脚本前你要确保系统中有足够多的free内存（避免内存紧张产生回收行为），最终的测试结果是这样的：</p><blockquote>
<p>File size 1048576KB, File Cache increased 1048648KB</p>
</blockquote><p>通过这个脚本你可以看到，在创建一个文件的过程中，代码中/proc/meminfo里的Active(file)和Inactive(file)这两项会随着文件内容的增加而增加，它们增加的大小跟文件大小是一致的（这里之所以略有不同，是因为系统中还有其他程序在运行）。另外，如果你观察得很仔细的话，你会发现增加的Page Cache是Inactive(File)这一项，<strong>你可以去思考一下为什么会这样？</strong>这里就作为咱们这节课的思考题。</p><p>当然，这个过程看似简单，但是它涉及的内核机制还是很多的，换句话说，可能引起问题的地方还是很多的，我们用一张图简单描述下这个过程：</p><p><img src="https://static001.geekbang.org/resource/image/c7/5e/c728b8a189fb4e35e536cf131c4d9b5e.jpg?wh=3400*3000" alt=""></p><p>这个过程大致可以描述为：首先往用户缓冲区buffer(这是Userspace Page)写入数据，然后buffer中的数据拷贝到内核缓冲区（这是Pagecache Page），如果内核缓冲区中还没有这个Page，就会发生Page Fault会去分配一个Page，拷贝结束后该Pagecache Page是一个Dirty Page（脏页），然后该Dirty Page中的内容会同步到磁盘，同步到磁盘后，该Pagecache Page变为Clean Page并且继续存在系统中。</p><p>我建议你可以将Alloc Page理解为Page Cache的“诞生”，将Dirty Page理解为Page Cache的婴幼儿时期（最容易生病的时期），将Clean Page理解为Page Cache的成年时期（在这个时期就很少会生病了）。</p><p><strong>但是请注意，并不是所有人都有童年的</strong>，比如孙悟空，一出生就是成人了，Page Cache也一样，如果是读文件产生的Page Cache，它的内容跟磁盘内容是一致的，所以它一开始是Clean Page，除非改写了里面的内容才会变成Dirty Page（返老还童）。</p><p>就像我们为了让婴幼儿健康成长，要悉心照料他/她一样，为了提前发现或者预防婴幼儿时期的Page Cache发病，我们也需要一些手段来观测它：</p><pre><code>$ cat /proc/vmstat | egrep &quot;dirty|writeback&quot;
nr_dirty 40
nr_writeback 2
</code></pre><p>如上所示，nr_dirty表示当前系统中积压了多少脏页，nr_writeback则表示有多少脏页正在回写到磁盘中，他们两个的单位都是Page(4KB)。</p><p>通常情况下，小朋友们（Dirty Pages）聚集在一起（脏页积压）不会有什么问题，但在非常时期比如流感期间，就很容易导致聚集的小朋友越多病症就会越严重。与此类似，Dirty Pages如果积压得过多，在某些情况下也会容易引发问题，至于是哪些情况，又会出现哪些问题，我们会在案例篇中具体讲解。</p><p>明白了Page Cache的“诞生”后，我们再来看一下Page Cache的“死亡”：它是如何被释放的？</p><h2>Page Cache是如何“死亡”的？</h2><p>你可以把Page Cache的回收行为(Page Reclaim)理解为Page Cache的“自然死亡”。</p><p>言归正传，我们知道，服务器运行久了后，系统中free的内存会越来越少，用free命令来查看，大部分都会是used内存或者buff/cache内存，比如说下面这台生产环境中服务器的内存使用情况：</p><pre><code>$ free -g
       total  used  free  shared  buff/cache available
Mem:     125    41     6       0          79        82
Swap:      0     0     0
</code></pre><p>free命令中的buff/cache中的这些就是“活着”的Page Cache，那它们什么时候会“死亡”（被回收）呢？我们来看一张图：</p><p><img src="https://static001.geekbang.org/resource/image/1d/bb/1d430c93e397e23d67d12e28827c4bbb.jpg?wh=3658*2138" alt=""></p><p>你可以看到，应用在申请内存的时候，即使没有free内存，只要还有足够可回收的Page Cache，就可以通过回收Page Cache的方式来申请到内存，<strong>回收的方式主要是两种：直接回收和后台回收。</strong></p><p>那它是具体怎么回收的呢？你要怎么观察呢？其实在我看来，观察Page Cache直接回收和后台回收最简单方便的方式是使用sar：</p><pre><code>$ sar -B 1
02:14:01 PM  pgpgin/s pgpgout/s   fault/s  majflt/s  pgfree/s pgscank/s pgscand/s pgsteal/s    %vmeff


02:14:01 PM      0.14    841.53 106745.40      0.00  41936.13      0.00      0.00      0.00      0.00
02:15:01 PM      5.84    840.97  86713.56      0.00  43612.15    717.81      0.00    717.66     99.98
02:16:01 PM     95.02    816.53 100707.84      0.13  46525.81   3557.90      0.00   3556.14     99.95
02:17:01 PM     10.56    901.38 122726.31      0.27  54936.13   8791.40      0.00   8790.17     99.99
02:18:01 PM    108.14    306.69  96519.75      1.15  67410.50  14315.98     31.48  14319.38     99.80
02:19:01 PM      5.97    489.67  88026.03      0.18  48526.07   1061.53      0.00   1061.42     99.99
</code></pre><p>借助上面这些指标，你可以更加明确地观察内存回收行为，下面是这些指标的具体含义：</p><ul>
<li>pgscank/s : kswapd(后台回收线程)每秒扫描的page个数。</li>
<li>pgscand/s: Application在内存申请过程中每秒直接扫描的page个数。</li>
<li>pgsteal/s: 扫描的page中每秒被回收的个数。</li>
<li>%vmeff: pgsteal/(pgscank+pgscand), 回收效率，越接近100说明系统越安全，越接近0说明系统内存压力越大。</li>
</ul><p>这几个指标也是通过解析/proc/vmstat里面的数据来得出的，对应关系如下：</p><p><img src="https://static001.geekbang.org/resource/image/46/aa/4604ec0da3f87ec003317fb3337fa9aa.jpg?wh=2619*1616" alt=""></p><p>关于这几个指标我说一个小插曲，要知道，如果Linux Kernel本身设计不当会给你带来困扰。所以，如果你观察到应用程序的结果跟你的预期并不一致，也有可能是因为内核设计上存在问题，你可以对内核持适当的怀疑态度哦，下面这个是我最近遇到的一个案例。</p><p>如果你对Linus有所了解的话，应该会知道Linus对Linux Kernel设计的第一原则是“never break the user space”。很多内核开发者在设计内核特性的时候，会忽略掉新特性对应用程序的影响，比如在前段时间就有人(Google的一个内核开发者)提交了一个patch来修改内存回收这些指标的含义，但是最终被我和另外一个人(Facebook的一个内核开发者)把他的这个改动给否决掉了。具体的细节并不是咱们这节课的重点，我就不多说了，我建议你在课下看这个讨论：<a href="https://lore.kernel.org/linux-mm/20200507204913.18661-1-shakeelb@google.com/">[PATCH] mm: vmscan: consistent update to pgsteal and pgscan</a>，可以看一下内核开发者们在设计内核特性时是如何思考的，这会有利于你更加全面的去了解整个系统，从而让你的应用程序更好地融入到系统中。</p><h2>课堂总结</h2><p>以上就是本节课的全部内容了，本节课，我们主要讲了Page Cache是如何“诞生”的，以及如何“死亡”的，我要强调这样几个重点：</p><ul>
<li>Page Cache是在应用程序读写文件的过程中产生的，所以在读写文件之前你需要留意是否还有足够的内存来分配Page Cache；</li>
<li>Page Cache中的脏页很容易引起问题，你要重点注意这一块；</li>
<li>在系统可用内存不足的时候就会回收Page Cache来释放出来内存，我建议你可以通过sar或者/proc/vmstat来观察这个行为从而更好的判断问题是否跟回收有关</li>
</ul><p>总的来说，Page Cache的生命周期对于应用程序而言是相对比较透明的，即它的分配与回收都是由操作系统来进行管理的。正是因为这种“透明”的特征，所以应用程序才会难以控制Page Cache，Page Cache才会容易引发那么多问题。在接下来的案例篇里，我们就来看看究竟会引发什么样子的问题，以及你正确的分析思路是什么样子的。</p><h2>课后作业</h2><p>因为每个人的关注点都不一样，对问题的理解也不一样。假如你是一个应用开发者，你会更加关注应用的性能和稳定性；假如你是一个运维人员，你会更加关注系统的稳定性；假如你是初学内核的开发者，你会想要关注内核的实现机制。</p><p>所以我留了不同的作业题，主题是围绕“Inactive与Active Page Cache的关系”当然了，对应的难度也不同：</p><ul>
<li>
<p>如果你是一名应用开发者，那么我想问问你为什么第一次读写某个文件，Page Cache是Inactive的？如何让它变成Active的呢？在什么情况下Active的又会变成Inactive的呢？明白了这个问题，你会对应用性能调优有更加深入的理解。</p>
</li>
<li>
<p>如果你是一名运维人员，那么建议你思考一下，系统中有哪些控制项可以影响Inactive与Active Page Cache的大小或者二者的比例？</p>
</li>
<li>
<p>如果你是一名初学内核的开发者，那么我想问你，对于匿名页而言，当产生一个匿名页后它会首先放在Active链表上；而对于文件页而言，当产生一个文件页后它会首先放在Inactive链表上。请问为什么会这样子？这是合理的吗？欢迎在留言区分享你的看法。</p>
</li>
</ul><p>感谢你的阅读，如果你认为这节课的内容有收获，也欢迎把它分享给你的朋友，我们下一讲见。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/3b/d7/9d942870.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>邵亚方</span>
  </div>
  <div class="_2_QraFYR_0">课后作业答案：<br>- 为什么第一次读写某个文件，Page Cache 是 Inactive 的？<br>第一次读取文件后，文件内容都是inactive的，只有再次读取这些内容后，才会把它放在active链表上，处于inactive链表上的pagecache在内存紧张是会首先被回收掉，有很多情况下文件内容往往只被读一次，比如日志文件，对于这类典型的one-off文件，它们占用的pagecache需要首先被回收掉；对于业务数据，往往都会读取多次，那么他们就会被放在active链表上，以此来达到保护的目的。<br><br>- 如何让它变成 Active 的呢？<br>第二次读取后，这些内容就会从inactive链表里给promote到active链表里，这也是评论区里有人提到的二次机会法。<br><br>- 在什么情况下 Active 的又会变成 Inactive 的呢？<br>在内存紧张时，会进行内存回收，回收会把inactive list的部分page给回收掉，为了维持inactive&#47;active的平衡，就需要把active list的部分page给demote到inactive list上，demote的原则也是lru。<br><br>- 系统中有哪些控制项可以影响 Inactive 与 Active Page Cache 的大小或者二者的比例？<br>min_free_kbytes会影响整体的pagecache大小;vfs_cache_pressure会影响在回收时回收pagecache和slab的比例; 在开启了swap的情况下，swappiness也会影响pagecache的大小；zone_reclaim_mode会影响node的pagecache大小；extfrag_threshold会影响pagecache的碎片情况。<br><br>- 对于匿名页而言，当产生一个匿名页后它会首先放在 Active 链表上，请问为什么会这样子？这是合理的吗？<br>这是不合理的，内核社区目前在做这一块的改进。具体可以参考https:&#47;&#47;lwn.net&#47;Articles&#47;816771&#47;。<br><br><br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-11 13:40:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>zwb</span>
  </div>
  <div class="_2_QraFYR_0">第二次机会法，避免大量只读一次的文件涌入 active，在需要回收时又从 active 移动到 inactive lru 链表。场景比如编译内核。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 理解的很正确！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-21 15:44:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/65/cf/326c0eea.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>x-ray</span>
  </div>
  <div class="_2_QraFYR_0">读这个确实需要对一些linux基础概念有一个了解。前几天刚读的时候，我连VFS都没有一个概念，读起来非常吃力，到第二章就看得云里雾里。这两天找了点视频把一些基础概念熟悉了下，今天再来看的时候，就感觉比较容易理解了。不过我有一个疑问，既然mmap映射的效率更高，为什么不都用这个呢？是因为标准IO无法像文件那样提前加载一块内存到PageCache吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: mmap与标准io的选择要看具体的场景。很多情况下内存拷贝不会是瓶颈，比如说只写几个或者几百字节的情况下，所以使用哪种都可以；只有在内存拷贝成为瓶颈，比如读写大量文件内容的情况下，比如一次要读写几十上百M，mmap的优势才会提现出来。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-15 23:54:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_162e2a</span>
  </div>
  <div class="_2_QraFYR_0">应用开发者的视角<br>第一次读写文件，PageCache是inactive的，为什么要这样设计？可能内核底层是采用类似LRU链表的设计来管理PageCache, 如果单纯照搬LRU链表的设计的话，当读大文件的时候会将原本属于热点缓存的PageCache冲刷出去，导致性能波动，因此需要对PageCache进行分类，来避免这个问题，即新读入的文件先进入inactive区域</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 理解的很准确 赞！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-27 17:37:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83epUBQibdMCca340MFZOe5I1GwZ0PosPIzA0TPCNzibgH00w45Zmv4jmL0mFRHMUM9FuKiclKOCBjSmsw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_circle</span>
  </div>
  <div class="_2_QraFYR_0">Memory-Mapped I&#47;O（存储映射 I&#47;O）<br> 是否就是零拷贝的概念呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以这么理解，它是文件内容的零拷贝。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-25 09:53:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/2b/09/2171f9a3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小白哥哥</span>
  </div>
  <div class="_2_QraFYR_0">不认同邵老师对于pagecache产生原因的描述，应用程序调用了write，内核会根据fd当前的fpos计算出写文件操作的文件偏移，然后根据偏移去inode-&gt;mapping中找出对应的pagecache页，如果没有的话，分配一页，插入inode-&gt;mapping，然后把write调用中的buffer拷贝到pagecache中，这个过程并不会触发page fault。如果是mmap映射文件，然后直接对内存读写，才会触发page fault，进而驱动内核加载文件内容到对应的page cache中。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: “这个过程并不会触发page fault”，除了这句话之外，你的其他理解是对的。<br>inode mapping中如果没有的话，内核会分配一个page，然后将write调用中的buffe拷贝到这个page 中，这个过程叫做pagein，它属于page fault的一种： major fault。你可以通过sar之类的工具来观察这个指标。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-13 14:23:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_162e2a</span>
  </div>
  <div class="_2_QraFYR_0">如何让它变成Active的呢？多读几次文件，达到系统设计的值后，此文件的PageCache会变成热点数据进入Active区域。<br>在什么情况Active会变成inactive的呢？热点文件太多，且此文件最近没有被读取过，自然就被挤出去了，静态资源服务器，可能会比较经常出现这种情况</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-27 17:40:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJpJXWFP3dNle88WnTkRTsEQkPJmOhepibiaTfhEtMRrbdg5EAWm4EzurA61oKxvCK2ZjMmy1QvmChw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>唐江</span>
  </div>
  <div class="_2_QraFYR_0">什么地方讲了inactive 、active 是个数据结构链表啊！不是一个简单的数字吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: proc接口中提供给用户的只是具体的数据项，这些数据项对应到内核代码则是一些数据结构。对于inactive active这两项而言，他们对应内核代码里的lru list。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-20 07:58:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>地下城勇士</span>
  </div>
  <div class="_2_QraFYR_0">老师的图是用什么工具画的？感觉以后可以尝试一下</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: gliffy。chrome有插件。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-25 09:00:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/c4/35/2cc10d43.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Wade_阿伟</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，看了上面老师的讲述，对存储映射I&#47;O和标准I&#47;O有了一定的理解。但是系统一般什么时候使用存储映射I&#47;O，什么时候使用标准I&#47;O呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 标准io相对而言更方便些，在数据量不大的情况下可以使用；如果数据量较大，此时使用存储映射io会更好些。另外一个考虑因为是对内存的精细管理，如果需要管理，比如说把某些数据锁定在内存中，这个时候使用存储映射io更好些。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-08 09:32:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/33/32/8f304f6c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>--SNIPER</span>
  </div>
  <div class="_2_QraFYR_0">测试了下都是0，能帮忙解释下为什么吗<br>10:39:31 AM  pgpgin&#47;s pgpgout&#47;s   fault&#47;s  majflt&#47;s  pgfree&#47;s pgscank&#47;s pgscand&#47;s pgsteal&#47;s    %vmeff<br>10:39:32 AM      0.00      8.00   1509.00      0.00   3651.00      0.00      0.00      0.00      0.00<br>10:39:33 AM      0.00      0.00   1566.00      0.00   3633.00      0.00      0.00      0.00      0.00<br>10:39:34 AM      0.00     12.00   1920.00      0.00   3815.00      0.00      0.00      0.00      0.00<br>10:39:35 AM      0.00      4.00   7944.00      0.00   6108.00      0.00      0.00      0.00      0.00<br>10:39:36 AM      0.00      0.00    993.00      0.00   3638.00      0.00      0.00      0.00      0.00<br>10:39:37 AM      0.00      0.00   1171.00      0.00   3616.00      0.00      0.00      0.00      0.00<br>10:39:38 AM      0.00      0.00    944.00      0.00   3756.00      0.00      0.00      0.00      0.00<br>10:39:39 AM      0.00     76.00  13438.00      0.00   6632.00      0.00      0.00      0.00      0.00<br>10:39:40 AM      0.00      0.00   7963.00      0.00   5471.00      0.00      0.00      0.00      0.00<br>^C<br><br>10:39:41 AM      0.00      4.76   1592.86      0.00   4565.48      0.00      0.00      0.00      0.00<br>Average:         0.00     10.57   3941.67      0.00   4487.30      0.00      0.00      0.00      0.00</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-11 10:41:06</div>
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
  <div class="_2_QraFYR_0">不是很理解为什么要有alloc page，不是已经有了userspace page了吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-18 16:37:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4a/15/106eaaa8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>stackWarn</span>
  </div>
  <div class="_2_QraFYR_0"> &#47;proc&#47;sys&#47;vm&#47;pagecache_limit_mb 限制使用大小</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-17 20:02:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/37/a0/032d0828.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>上杉夏香</span>
  </div>
  <div class="_2_QraFYR_0">关于课后作业，不清楚 MySQL 的 Buffer Pool 和 操作系统的 Page Cache 谁先谁后。不过他们考虑的都是，作为缓存，想最大次数的被命中。第一次读文件为 Inactive，和 MySQL 查询时，将首次命中的记录 Page 放入 yound 区，都是为了避免新的、可能并不怎么被访问的缓存将之前频繁访问的缓存淘汰出去。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-04 10:53:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/0c/30/bb4bfe9d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lyonger</span>
  </div>
  <div class="_2_QraFYR_0">这一节课把buffer算到了user space ，上一节课又算到了page cache ，而page cache属于kernel space。 有点不懂？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-23 12:02:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJJ7Ft8jtL1CySd4jeZ32kSNvTcHV12s0zN47WmVr6N8GTtbt0rlfyRzFqlOIjjtdFrmoSNu49IiaQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_66675e</span>
  </div>
  <div class="_2_QraFYR_0">生产环境很多没有开启swap。如果没有开启swap，那pagecache是由哪个内核线程回收呢？还是不会回收</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-16 09:57:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/29/b1/10/061bc360.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>悦动音巧</span>
  </div>
  <div class="_2_QraFYR_0">老师，咨询下我的内存都被buffer&#47;cache占用了，执行echo 3释放后，几分钟又被占满了，sar查看都是0<br>[root@test ~]# free   -g<br>              total        used        free      shared  buff&#47;cache   available<br>Mem:             63           1           0           2          61          57<br>Swap:             7           3           4<br><br>[root@test ~]# sar -B 1<br>Linux 3.10.0-514.el7.x86_64 (CRS-DB)    2022年11月11日  _x86_64_        (12 CPU)<br><br>17时42分01秒  pgpgin&#47;s pgpgout&#47;s   fault&#47;s  majflt&#47;s  pgfree&#47;s pgscank&#47;s pgscand&#47;s pgsteal&#47;s    %vmeff<br>17时42分02秒      0.00     32.00     57.00      0.00     79.00      0.00      0.00      0.00      0.00<br>17时42分03秒      0.00     12.00     42.00      0.00     67.00      0.00      0.00      0.00      0.00<br>17时42分04秒      0.00      0.00     32.00      0.00     51.00      0.00      0.00      0.00      0.00<br>17时42分05秒      0.00     32.00     23.00      0.00     40.00      0.00      0.00      0.00      0.00<br>17时42分06秒      0.00      0.00     38.00      0.00     62.00      0.00      0.00      0.00      0.00<br>17时42分07秒      0.00      0.00     23.00      0.00     40.00      0.00      0.00      0.00      0.00</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-11 17:43:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/bibTL8t812ehcBB0cPYqsDycLq37iaXmbzdGwAibkSe4G9r0lDoYxibvnLEhkWNWicPbe70j926FbyibKGPIEMh7ib78Q/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_356c0d</span>
  </div>
  <div class="_2_QraFYR_0">我对内存申请那张图有疑问，申请内存的时候如果内存不够，这时触发了kswapd内核线程去后台清理内存，这时不需要同步等待它清理完吗？按照图片得意思，是唤醒了kswapd之后就立刻走向下一步了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-13 13:49:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/bb/7e/4e86c5a5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Aizen</span>
  </div>
  <div class="_2_QraFYR_0">Page Cache 我问个问题？容器内的Page Cache的回收策略是基于 真是物理机的内核分配不足才回收？还是容器本身的内存不足策略？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-17 19:18:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_b8749d</span>
  </div>
  <div class="_2_QraFYR_0">老师讲的太好了，以前觉的pagecache很神秘，经过老师讲解，有种豁然开朗的感觉</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-24 11:35:05</div>
  </div>
</div>
</div>
</li>
</ul>