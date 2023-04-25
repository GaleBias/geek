<audio title="04 案例篇 _ 如何处理Page Cache容易回收引起的业务性能问题？" src="https://static001.geekbang.org/resource/audio/5a/a7/5a1097233a4e86558c16e9320190c7a7.mp3" controls="controls"></audio> 
<p>你好，我是邵亚方。我们在前一节课讲了Page Cache难以回收导致的load飙高问题，这类问题是很直观的，相信很多人都遭遇过。这节课，我们则是来讲相反的一些问题，即Page Cache太容易回收而引起的一些问题。</p><p>这类问题因为不直观所以陷阱会很多，应用开发者和运维人员会更容易踩坑，也正因为这类问题不直观，所以他们往往是一而再再而三地中招之后，才搞清楚问题到底是怎么回事。</p><p>我把大家经常遇到的这类问题做个总结，大致可以分为两方面：</p><ul>
<li>误操作而导致Page Cache被回收掉，进而导致业务性能下降明显；</li>
<li>内核的一些机制导致业务Page Cache被回收，从而引起性能下降。</li>
</ul><p>如果你的业务对Page Cache比较敏感，比如说你的业务数据对延迟很敏感，或者再具体一点，你的业务指标对TP99（99分位）要求较高，那你对于这类性能问题应该多多少少有所接触。当然，这并不意味着业务对延迟不敏感，你就不需要关注这些问题了，关注这类问题会让你对业务行为理解更深刻。</p><p>言归正传，我们来看下发生在生产环境中的案例。</p><h2>对Page Cache操作不当产生的业务性能下降</h2><p>我们先从一个相对简单的案例说起，一起分析下误操作导致Page Cache被回收掉的情况，它具体是怎样发生的。</p><!-- [[[read_end]]] --><p>我们知道，对于Page Cache而言，是可以通过drop_cache来清掉的，很多人在看到系统中存在非常多的Page Cache时会习惯使用drop_cache来清理它们，但是这样做是会有一些负面影响的，比如说这些Page Cache被清理掉后可能会引起系统性能下降。为什么？</p><p>其实这和inode有关，那inode是什么意思呢？inode是内存中对磁盘文件的索引，进程在查找或者读取文件时就是通过inode来进行操作的，我们用下面这张图来表示一下这种关系：</p><p><img src="https://static001.geekbang.org/resource/image/7e/99/7ef7747f71ef236e8cf5f9378d80da99.jpg?wh=3500*2564" alt=""></p><p>如上图所示，进程会通过inode来找到文件的地址空间（address_space），然后结合文件偏移（会转换成page index）来找具体的Page。如果该Page存在，那就说明文件内容已经被读取到了内存；如果该Page不存在那就说明不在内存中，需要到磁盘中去读取。你可以理解为inode是Pagecache Page（页缓存的页）的宿主（host），如果inode不存在了，那么PageCache Page也就不存在了。</p><p>如果你使用过drop_cache来释放inode的话，应该会清楚它有几个控制选项，我们可以通过写入不同的数值来释放不同类型的cache（用户数据Page Cache，内核数据Slab，或者二者都释放），这些选项你可以去看<a href="https://www.kernel.org/doc/Documentation/sysctl/vm.txt">Kernel Documentation的描述</a>。</p><p><img src="https://static001.geekbang.org/resource/image/ab/7d/ab748f14be603df45c2570fe8e24707d.jpg?wh=3779*1660" alt=""></p><p>于是这样就引入了一个容易被我们忽略的问题：<strong>当我们执行echo 2来drop slab的时候</strong><strong>，</strong><strong>它也会把Page Cache给drop掉</strong>，很多运维人员都会忽视掉这一点。</p><p>在系统内存紧张的时候，运维人员或者开发人员会想要通过drop_caches的方式来释放一些内存，但是由于他们清楚Page Cache被释放掉会影响业务性能，所以就期望只去drop slab而不去drop pagecache。于是很多人这个时候就运行 echo 2 &gt; /proc/sys/vm/drop_caches，但是结果却出乎了他们的意料：Page Cache也被释放掉了，业务性能产生了明显的下降。</p><p>很多人都遇到过这个场景：系统正在运行着，忽然Page Cache被释放掉了，由于不清楚释放的原因，所以很多人就会怀疑，是不是由其他人/程序执行了drop_caches导致的。那有没有办法来观察这个inode释放引起Page Cache被释放的行为呢？答案是有的。关于这一点，我们在下一节课会讲到。我们先来分析下如何观察是否有人或者有程序执行过drop_caches。</p><p>由于drop_caches是一种内存事件，内核会在/proc/vmstat中来记录这一事件，所以我们可以通过/proc/vmstat来判断是否有执行过drop_caches。</p><pre><code>$ grep drop /proc/vmstat
drop_pagecache 3
drop_slab 2
</code></pre><p>如上所示，它们分别意味着pagecache被drop了3次（通过echo 1 或者echo 3），slab被drop了2次（通过echo 2或者echo 3）。如果这两个值在问题发生前后没有变化，那就可以排除是有人执行了drop_caches；否则可以认为是因为drop_caches引起的Page Cache被回收。</p><p>针对这类问题，你除了在执行drop cache前三思而后行之外，还有其他的一些根治的解决方案。在讲这些解决方案之前，我们先来看一个更加复杂一点的案例，它们有一些共性，解决方案也类似，只是接下来这个案例涉及的内核机制更加复杂。</p><h2>内核机制引起Page Cache被回收而产生的业务性能下降</h2><p>我们在前面已经提到过，在内存紧张的时候会触发内存回收，内存回收会尝试去回收reclaimable（可以被回收的）内存，这部分内存既包含Page Cache又包含reclaimable kernel memory(比如slab)。我们可以用下图来简单描述这个过程：</p><p><img src="https://static001.geekbang.org/resource/image/d3/3c/d33a7264358f20fb063cb49fc3f3163c.jpg?wh=3500*2870" alt=""></p><p>我简单来解释一下这个图。Reclaimer是指回收者，它可以是内核线程（包括kswapd）也可以是用户线程。回收的时候，它会依次来扫描pagecache page和slab page中有哪些可以被回收的，如果有的话就会尝试去回收，如果没有的话就跳过。在扫描可回收page的过程中回收者一开始扫描的较少，然后逐渐增加扫描比例直至全部都被扫描完。这就是内存回收的大致过程。</p><p>接下来我所要讲述的案例就发生在“relcaim slab”中，我们从前一个案例已然知道，如果inode被回收的话，那么它对应的Page Cache也都会被回收掉，所以如果业务进程读取的文件对应的inode被回收了，那么该文件所有的Page Cache都会被释放掉，这也是容易引起性能问题的地方。</p><p>那这个行为是否有办法观察？这同样也是可以通过/proc/vmstat来观察的，/proc/vmstat简直无所不能（这也是为什么我会在之前说内核开发者更习惯去观察/proc/vmstat）。</p><pre><code> $ grep inodesteal /proc/vmstat 
 pginodesteal 114341
 kswapd_inodesteal 1291853
</code></pre><p>这个行为对应的事件是inodesteal，就是上面这两个事件，其中kswapd_inodesteal是指在kswapd回收的过程中，因为回收inode而释放的pagecache page个数；pginodesteal是指kswapd之外其他线程在回收过程中，因为回收inode而释放的pagecache page个数。所以在你发现业务的Page Cache被释放掉后，你可以通过观察来发现是否因为该事件导致的。</p><p>在明白了Page Cache被回收掉是如何发生的，以及知道了该如何观察之后，我们来看下该如何解决这类问题。</p><h2>如何避免Page Cache被回收而引起的性能问题？</h2><p>我们在分析一些问题时，往往都会想这个问题是我的模块有问题呢，还是别人的模块有问题。也就是说，是需要修改我的模块来解决问题还是需要修改其他模块来解决问题。与此类似，避免Page Cache里相对比较重要的数据被回收掉的思路也是有两种：</p><ul>
<li>从应用代码层面来优化；</li>
<li>从系统层面来调整。</li>
</ul><p>从应用程序代码层面来解决是相对比较彻底的方案，因为应用更清楚哪些Page Cache是重要的，哪些是不重要的，所以就可以明确地来对读写文件过程中产生的Page Cache区别对待。比如说，对于重要的数据，可以通过mlock(2)来保护它，防止被回收以及被drop；对于不重要的数据（比如日志），那可以通过madvise(2)告诉内核来立即释放这些Page Cache。</p><p>我们来看一个通过mlock(2)来保护重要数据防止被回收或者被drop的例子：</p><pre><code>#include &lt;sys/mman.h&gt;
#include &lt;sys/types.h&gt;
#include &lt;sys/stat.h&gt;
#include &lt;unistd.h&gt;
#include &lt;string.h&gt;
#include &lt;fcntl.h&gt;


#define FILE_NAME &quot;/home/yafang/test/mmap/data&quot;
#define SIZE (1024*1000*1000)


int main()
{
        int fd; 
        char *p; 
        int ret;


        fd = open(FILE_NAME, O_CREAT|O_RDWR, S_IRUSR|S_IWUSR);
        if (fd &lt; 0)
                return -1; 


        /* Set size of this file */
        ret = ftruncate(fd, SIZE);
        if (ret &lt; 0)
                return -1; 


        /* The current offset is 0, so we don't need to reset the offset. */
        /* lseek(fd, 0, SEEK_CUR); */


        /* Mmap virtual memory */
        p = mmap(0, SIZE, PROT_READ|PROT_WRITE, MAP_FILE|MAP_SHARED, fd, 0); 
        if (!p)
                return -1; 


        /* Alloc physical memory */
        memset(p, 1, SIZE);


        /* Lock these memory to prevent from being reclaimed */
        mlock(p, SIZE);


        /* Wait until we kill it specifically */
        while (1) {
                sleep(10);
        }


        /*
         * Unmap the memory.
         * Actually the kernel will unmap it automatically after the
         * process exits, whatever we call munamp() specifically or not.
         */
        munmap(p, SIZE);

        return 0;
}
</code></pre><p>在这个例子中，我们通过mlock(2)来锁住了读FILE_NAME这个文件内容对应的Page Cache。在运行上述程序之后，我们来看下该如何来观察这种行为：确认这些Page Cache是否被保护住了，被保护了多大。这同样可以通过/proc/meminfo来观察:</p><pre><code>$ egrep &quot;Unevictable|Mlocked&quot; /proc/meminfo 
Unevictable:     1000000 kB
Mlocked:         1000000 kB
</code></pre><p>然后你可以发现，drop_caches或者内存回收是回收不了这些内容的，我们的目的也就达到了。</p><p>在有些情况下，对应用程序而言，修改源码是件比较麻烦的事，如果可以不修改源码来达到目的那就最好不过了。Linux内核同样实现了这种不改应用程序的源码而从系统层面调整来保护重要数据的机制，这个机制就是memory cgroup protection。</p><p>它大致的思路是，将需要保护的应用程序使用memory cgroup来保护起来，这样该应用程序读写文件过程中所产生的Page Cache就会被保护起来不被回收或者最后被回收。memory cgroup protection大致的原理如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/71/cc/71e7347264a54773d902aa92535b87cc.jpg?wh=2200*1642" alt=""></p><p>如上图所示，memory cgroup提供了几个内存水位控制线memory.{min, low, high, max} 。</p><ul>
<li>
<p><strong>memory.max</strong><br>
这是指memory cgroup内的进程最多能够分配的内存，如果不设置的话，就默认不做内存大小的限制。</p>
</li>
<li>
<p><strong>memory.high</strong><br>
如果设置了这一项，当memory cgroup内进程的内存使用量超过了该值后就会立即被回收掉，所以这一项的目的是为了尽快的回收掉不活跃的Page Cache。</p>
</li>
<li>
<p><strong>memory.low</strong><br>
这一项是用来保护重要数据的，当memory cgroup内进程的内存使用量低于了该值后，在内存紧张触发回收后就会先去回收不属于该memory cgroup的Page Cache，等到其他的Page Cache都被回收掉后再来回收这些Page Cache。</p>
</li>
<li>
<p><strong>memory.min</strong><br>
这一项同样是用来保护重要数据的，只不过与memoy.low有所不同的是，当memory cgroup内进程的内存使用量低于该值后，即使其他不在该memory cgroup内的Page Cache都被回收完了也不会去回收这些Page Cache，可以理解为这是用来保护最高优先级的数据的。</p>
</li>
</ul><p>那么，<strong>如果你想要保护你的Page Cache不被回收，你就可以考虑将你的业务进程放在一个memory cgroup中，然后设置memory.{min,low} 来进行保护；与之相反，如果你想要尽快释放你的Page Cache，那你可以考虑设置memory.high来及时的释放掉不活跃的Page Cache。</strong></p><p>更加细节性的一些设置我们就不在这里讨论了，我建议你可以自己动手来设置后观察一下，这样你理解会更深刻。</p><h2>课堂总结</h2><p>我们在前一篇讲到了Page Cache回收困难引起的load飙高问题，这也是很直观的一类问题；在这一篇讲述的则是一类相反的问题，即Page Cache太容易被回收而引起的一些问题，这一类问题是不那么直观的一类问题。</p><p>对于很直观的问题，我们相对比较容易去观察分析，而且由于它们比较容易观察，所以也相对能够得到重视；对于不直观的问题，则不是那么容易观察分析，相对而言它们也容易被忽视。</p><p>外在的特征不明显，并不意味着它产生的影响不严重，就好比皮肤受伤流血了，我们知道需要立即止血这个伤病也能很容易得到控制；如果是内伤，比如心肝脾肺肾有了问题，则容易被忽视，但是这些问题一旦积压久了往往造成很严重的后果。所以对于这类不直观的问题，我们还是需要去重视它，而且尽量做到提前预防，比如说：</p><ul>
<li>如果你的业务对Page Cache所造成的延迟比较敏感，那你最好可以去保护它，比如通过mlock或者memory cgroup来对它们进行保护；</li>
<li>在你不明白Page Cache是因为什么原因被释放时，你可以通过/proc/vmstat里面的一些指标来观察，找到具体的释放原因然后再对症下药的去做优化。</li>
</ul><h2>课后作业</h2><p>这节课给你布置的作业是与mlock相关的，请你思考下，进程调用mlock()来保护内存，然后进程没有运行munlock()就退出了，在进程退出后，这部分内存还被保护吗，为什么？欢迎在留言区分享你的看法。</p><p>感谢你的阅读，如果你认为这节课的内容有收获，也欢迎把它分享给你的朋友，我们下一讲见。</p>
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
  <div class="_2_QraFYR_0">课后作业答案：<br>- 进程调用 mlock() 来保护内存，然后进程没有运行 munlock() 就退出了，在进程退出后，这部分内存还被保护吗，为什么？<br>这部分内容就不被保护了，评论区里有人已经给出了文档里的解释：<br> Memory locks are not inherited by a child created via fork(2) and are<br>       automatically removed (unlocked) during an execve(2) or when the<br>       process terminates.<br>https:&#47;&#47;man7.org&#47;linux&#47;man-pages&#47;man2&#47;mlock.2.html<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-11 13:43:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/03/83/abb7bfe3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>奔跑的熊</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，如何观察单个应用的page cache?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 通过lsof可以查看某个应用打开的文件，然后再使用fincore可以看这些文件有多少内容在pagecache。<br>或者应用程序里直接调用mincore也可以查看page cache有多少。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-02 19:07:00</div>
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
  <div class="_2_QraFYR_0">思考题：<br>在进程退出之后，此部分内存不会再被保护<br>为什么呢，文档是这么说的������<br>       Memory locks are not inherited by a child created via fork(2) and are<br>       automatically removed (unlocked) during an execve(2) or when the<br>       process terminates.<br>https:&#47;&#47;man7.org&#47;linux&#47;man-pages&#47;man2&#47;mlock.2.html</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的 赞！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-28 15:28:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ray</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，请问<br>如果使用echo 2 &gt; &#47;proc&#47;sys&#47;vm&#47;drop_caches释放slab objects (includes dentries and inodes)也会释放掉page cache，那这条指令和单纯释放page cache的echo 1 &gt; &#47;proc&#47;sys&#47;vm&#47;drop_caches指令的区别又在哪里呢？<br><br>谢谢老师的解答^^</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: echo 2会去尝试释放所有可以回收的slab，但是它释放inode的过程中也会释放inode的page cache，其实这个行为在我看来是不合理的，如果inode上面有page cache就不应该被释放，目前内核社区也在针对这个行为做改进，预计它很快会被修改。另外还有很多inode是不可以被释放的，那这部分inode里的page cache也不会被释放。<br>echo 1则只是释放page cache，而不去释放slab，而且在释放pagecache的过程中也不会去考虑inode的情况，也就是说echo 2中有些不会释放的pagecache在这种方式里会被释放。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-28 21:42:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/9b/ba/333b59e5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Linuxer</span>
  </div>
  <div class="_2_QraFYR_0">请问一下邵老师，我看memory cgroup中相关的参数是这些  memory.kmem.limit_in_bytes,memory.kmem.max_usage_in_bytes,memory.usage_in_bytes,memory.soft_limit_in_bytes,memory.memsw.usage_in_bytes.memory.memsw.max_usage_in_bytes请问跟文章中memory.max, memory.high这些是怎么对应的呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这是cgroup1的memcg参数，memory.high和memory.max在原生内核里只在cgroup2中有，如果你们用的是cgroup1 就无法使用这几个参数了。我们在自己的内核里在cgroup1里也添加了high max这些参数，你们也可以考虑添加下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-03 11:00:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/96/47/93838ff7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>青鸟飞鱼</span>
  </div>
  <div class="_2_QraFYR_0">老师，你好，有个疑问，memory cgroup 来对它们进行保护，如何保证我要保护的数据是放在里面的呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 把进程放在memory cgroup中，然后进程读写的文件内容都属于这个memory cgroup中；如果进程读写文件后，再把它放在memory cgroup中，那么文件内容不会属于这个memory cgroup。所以要注意顺序。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-20 11:39:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>从远方过来</span>
  </div>
  <div class="_2_QraFYR_0">老师，我有几个疑问：<br>1. 扫描比例是怎么设置的？和zoneinfo里面的min，low，high有什么的联系么？<br>2. “回收到足够的内存” 是指回收到high水位还是满足这一次内存分配就停止了？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. 扫描比例是在内核里面配置好的 用户无法调整，第一次扫描1&#47;12，第二次扫描1&#47;11，一直到最后一次扫描全部。通过调整内存水位最终就会体现在zoneinfo里的min low high里面，这几个值也是内核根据内存情况来动态调整的。<br>2. 如果是直接回收，那只需要会收到满足需求的内存就可以了；如果是kswapd回收，那会一直回收到high水位。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-27 09:59:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/b4/00/bfc101ee.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Tendrun</span>
  </div>
  <div class="_2_QraFYR_0">想到一个问题，有没有办法针对某个进程进行page cache回收呢，这样就不会影响到其他进程的性能了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-07 19:55:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/ec/ce/3ec9f1a8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>颜卫</span>
  </div>
  <div class="_2_QraFYR_0">老师，你好。看内核代码，memory.high好像不是优先回收，而是在达到这个水位后，每次申请增加一定的时延<br><br>&#47;*<br> * Scheduled by try_charge() to be executed from the userland return path<br> * and reclaims memory over the high limit.<br> *&#47;<br>void mem_cgroup_handle_over_high(void)<br>{<br>    unsigned long penalty_jiffies;<br>    unsigned long pflags;<br>    unsigned int nr_pages = current-&gt;memcg_nr_pages_over_high;<br>    struct mem_cgroup *memcg;<br>    u64 start;<br><br>    if (likely(!nr_pages))<br>        return;<br><br>    sli_memlat_stat_start(&amp;amp;start);<br>    memcg = get_mem_cgroup_from_mm(current-&gt;mm);<br>    reclaim_high(memcg, nr_pages, GFP_KERNEL);<br>    current-&gt;memcg_nr_pages_over_high = 0;<br><br>    &#47;*<br>     * memory.high is breached and reclaim is unable to keep up. Throttle<br>     * allocators proactively to slow down excessive growth.<br>     *&#47;<br>    penalty_jiffies = calculate_high_delay(memcg, nr_pages);<br><br>    &#47;*<br>     * Don&#39;t sleep if the amount of jiffies this memcg owes us is so low<br>     * that it&#39;s not even worth doing, in an attempt to be nice to those who<br>     * go only a small amount over their memory.high value and maybe haven&#39;t<br>     * been aggressively reclaimed enough yet.<br>     *&#47;<br>    if (penalty_jiffies &lt;= HZ &#47; 100)<br>        goto out;<br><br>    &#47;*<br>     * If we exit early, we&#39;re guaranteed to die (since<br>     * schedule_timeout_killable sets TASK_KILLABLE). This means we don&#39;t<br>     * need to account for any ill-begotten jiffies to pay them off later.<br>     *&#47;<br>    psi_memstall_enter(&amp;amp;pflags);<br>    schedule_timeout_killable(penalty_jiffies);<br>    psi_memstall_leave(&amp;amp;pflags);<br><br>out:<br>    sli_memlat_stat_end(MEM_LAT_MEMCG_DIRECT_RECLAIM, start);<br>    css_put(&amp;amp;memcg-&gt;css);<br>}<br><br>是不是我少找了一些地方，回收的时候也会考虑memory.high啊？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-23 21:06:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/de/92/fd30dc6f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>bittcom</span>
  </div>
  <div class="_2_QraFYR_0">老师好，想请教下slab scanned对应用抖动的问题，生产上8C24G的机器，slab经常用到8G多，其中dentry用掉了4G左右，kmalloc-192使用了3G左右，SReclaimable占了8G的2&#47;3左右，经常看到应哟给你抖动都会伴随着很高的slab scanned值，高达100w数值，这块cache引起的问题一般如何分析？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-31 05:41:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_c1d00e</span>
  </div>
  <div class="_2_QraFYR_0">老师，我在多线程的应用里，在mmap后，用mlock锁定，可是在&#47;proc&#47;meminfo中看到的Mlocked项的大小并没有增加；这是为啥呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-08 16:16:52</div>
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
  <div class="_2_QraFYR_0">inode被回收是什么意思？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: inode本身也是slab的一种，在内存不足时，就会去尝试回收slab，回收slab的时候就可能会回收到inode。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-20 19:40:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/68/c5/3cdf2390.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>洛戈</span>
  </div>
  <div class="_2_QraFYR_0">请问老师，使用memory cgroup怎么保护指定的关键数据呢？只能设置min来保留最小的page cache呀，那并不一定能确保关键数据在min之内吧</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对的 关键数据未必会在min之内，min只是用来保护较为活跃的数据。如果你想保护所有的数据 就需要将min和memcg大小配置为相等。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-22 08:53:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://wx.qlogo.cn/mmopen/vi_32/DYAIOgq83eqtXSgThiaEiaEqqic5YIJ7v469nCM3VXiccOJ4SxbYjW91ciczuYYEzcTVtYWaWXaokZqShuLdKsXjnFA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_b85295</span>
  </div>
  <div class="_2_QraFYR_0">老师，关于swap请教一个问题，调整swappiness参数可以控制系统使用swap的情况。我把这个参数改成了0，也就是尽量让机器不使用swap，在echo 1 &gt; &#47;proc&#47;sys&#47;vm&#47;drop_caches 得到大量free内存的情况下，用vmstat -w 1 指标中的swap信息，还是可以看到很多的换进换出，这是为什么，谢谢老师答疑</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: swappiness为0并不能保障不会去使用swap分区，而只是说尽量不去使用swap分区，在内存水位过低的时候还是要会使用swap分区。<br><br>如果已经drop cache了，此时会有很多的free内存，理论上不应该再需要进行swapout，如果有的话，可能会跟node有关？你的系统有多个node？某个node上free内存很少？<br>swapin是还可能会存在的，假如你的swap分区里有数据，而进程接下来需要去读这些数据，就会swapin。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-29 22:17:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/d0/d0/a6c6069d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>坚</span>
  </div>
  <div class="_2_QraFYR_0">老师好，我查看了一下 zoneinfo 其中   pages free=4497，但是min   low hight 分别为5 6 7 ，三者设置得这么接近，有这么小，是否会有什么问题呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这要看具体是什么zone，如果是dma zone的话，由于本身这个zone里的内存较少，所以水位小也是正常的。<br>min low high其实是等差数列，这个差值就主要是由zone的大小来决定的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-02 20:31:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/20/f2/fc/316fdc18.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>王一怂</span>
  </div>
  <div class="_2_QraFYR_0">inode本身也是文件系统的概念，这里的inode应该是内存中的inode，感觉还是区分一下比较好</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞 你理解的很对 我们在这里只是讲述了vfs中的inode，也就是struct inode这个slab，而没有去花篇幅去讲述具体文件系统的inode实例，比如xfs_inode等。各个文件系统在这一块的实现会有所不同，所以我就没有花篇幅来讲，不过vfs inode是其他具体文件系统inode的基础，理解好了它也会帮助你理解具体文件系统的inode。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-02 11:08:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/f5/95/a362f01b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek1560</span>
  </div>
  <div class="_2_QraFYR_0">老师好，请教一个问题，当内核分配内存时，如果没有空闲页，其中slab&#47;slub部分的匿名页会调用shrink_inactive_list 函数会扫描inactive list，将不活跃的page置换到swap分区。但是swap有时几M、几百M甚至几个G，这个内核置换的机制或算法逻辑是啥？(PS本身应用程序或内核不会一次性申请几个G的内存)</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 内存不足时，会把不活跃的匿名页给置换到磁盘。有多少匿名页可以被置换到磁盘swap分区，是由swappiness这个参数来控制的，该值越大就越容易发生swap。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-30 19:00:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/64/9b/d1ab239e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>J.Smile</span>
  </div>
  <div class="_2_QraFYR_0">①”pginodesteal 是指 kswapd 之外其他线程在回收过程中，因为回收 inode 而释放的 pagecache page 个数。“------这里的”kswapd 之外其他线程“有可能就是用户业务线程吧？？<br>②对于java工程师，完全不懂memory cgroup 为何物。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. 对的，既包括内核线程，也包括业务线程。<br>2. memory cgroup是用来控制应用的内存使用情况的，如果你想要现在JAVA进程的内存使用，而又不想修改JAVA代码，那你可以考虑使用它。它对于JAVA程序同样有用。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-28 11:34:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/7a/8b/2c81a375.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>好说</span>
  </div>
  <div class="_2_QraFYR_0">老师，对于io密集型的业务，基本上大部分都是读磁盘，当带宽达到一定量级之后服务器的load会很高，但是当我执行echo &quot;1&quot;&gt;&#47;proc&#47;sys&#47;vm&#47;drop_caches后，服务器load会非常明显的下降，然后过一会就又会升上去，请问老师造成这种现象的原因是什么呢，或者老师可以提示一些排查思路吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这应该是应用在申请内存时触发了直接回收引起的。drop cache之后，就有很多的可用内存，应用就无需回收，但是运行久了，page cache就会积压，导致接下来的内存申请会进行直接回收。<br>你可以在load高时通过sar -B来观察看看pgscand是否有不为0的情况。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-27 19:29:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/e9/53/5c19021b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>梁鹏</span>
  </div>
  <div class="_2_QraFYR_0">邵老师 能否简单的概括一下 page cache 和 slab cache 的 关系，或者推荐一些资料</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: page cache可以理解为是用户数据的缓存，而slab cache则是内核数据的缓存。这一块想深入理解，最好的学习资料就是内核源码以Document目录下的文档。<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-27 11:26:48</div>
  </div>
</div>
</div>
</li>
</ul>