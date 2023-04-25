<audio title="17 _ 为什么CPU结构也会影响Redis的性能？" src="https://static001.geekbang.org/resource/audio/1e/6c/1e6bc30079078d1598c077262d1a3b6c.mp3" controls="controls"></audio> 
<p>你好，我是蒋德钧。</p><p>很多人都认为Redis和CPU的关系很简单，就是Redis的线程在CPU上运行，CPU快，Redis处理请求的速度也很快。</p><p>这种认知其实是片面的。CPU的多核架构以及多CPU架构，也会影响到Redis的性能。如果不了解CPU对Redis的影响，在对Redis的性能进行调优时，就可能会遗漏一些调优方法，不能把Redis的性能发挥到极限。</p><p>今天，我们就来学习下目前主流服务器的CPU架构，以及基于CPU多核架构和多CPU架构优化Redis性能的方法。</p><h2>主流的CPU架构</h2><p>要了解CPU对Redis具体有什么影响，我们得先了解一下CPU架构。</p><p>一个CPU处理器中一般有多个运行核心，我们把一个运行核心称为一个物理核，每个物理核都可以运行应用程序。每个物理核都拥有私有的一级缓存（Level 1 cache，简称L1 cache），包括一级指令缓存和一级数据缓存，以及私有的二级缓存（Level 2 cache，简称L2 cache）。</p><p>这里提到了一个概念，就是物理核的私有缓存。它其实是指缓存空间只能被当前的这个物理核使用，其他的物理核无法对这个核的缓存空间进行数据存取。我们来看一下CPU物理核的架构。</p><p><img src="https://static001.geekbang.org/resource/image/c2/3a/c2d620c012a82e825570df631a7fbc3a.jpg?wh=2114*1167" alt=""></p><p>因为L1和L2缓存是每个物理核私有的，所以，当数据或指令保存在L1、L2缓存时，物理核访问它们的延迟不超过10纳秒，速度非常快。那么，如果Redis把要运行的指令或存取的数据保存在L1和L2缓存的话，就能高速地访问这些指令和数据。</p><!-- [[[read_end]]] --><p>但是，这些L1和L2缓存的大小受限于处理器的制造技术，一般只有KB级别，存不下太多的数据。如果L1、L2缓存中没有所需的数据，应用程序就需要访问内存来获取数据。而应用程序的访存延迟一般在百纳秒级别，是访问L1、L2缓存的延迟的近10倍，不可避免地会对性能造成影响。</p><p>所以，不同的物理核还会共享一个共同的三级缓存（Level 3 cache，简称为L3 cache）。L3缓存能够使用的存储资源比较多，所以一般比较大，能达到几MB到几十MB，这就能让应用程序缓存更多的数据。当L1、L2缓存中没有数据缓存时，可以访问L3，尽可能避免访问内存。</p><p>另外，现在主流的CPU处理器中，每个物理核通常都会运行两个超线程，也叫作逻辑核。同一个物理核的逻辑核会共享使用L1、L2缓存。</p><p>为了方便你理解，我用一张图展示一下物理核和逻辑核，以及一级、二级缓存的关系。</p><p><img src="https://static001.geekbang.org/resource/image/d9/09/d9689a38cbe67c3008d8ba99663c2f09.jpg?wh=3065*1633" alt=""></p><p>在主流的服务器上，一个CPU处理器会有10到20多个物理核。同时，为了提升服务器的处理能力，服务器上通常还会有多个CPU处理器（也称为多CPU Socket），每个处理器有自己的物理核（包括L1、L2缓存），L3缓存，以及连接的内存，同时，不同处理器间通过总线连接。</p><p>下图显示的就是多CPU Socket的架构，图中有两个Socket，每个Socket有两个物理核。</p><p><img src="https://static001.geekbang.org/resource/image/5c/3d/5ceb2ab6f61c064284c8f8811431bc3d.jpg?wh=3000*1252" alt=""></p><p><strong>在多CPU架构上，应用程序可以在不同的处理器上运行</strong>。在刚才的图中，Redis可以先在Socket  1上运行一段时间，然后再被调度到Socket  2上运行。</p><p>但是，有个地方需要你注意一下：如果应用程序先在一个Socket上运行，并且把数据保存到了内存，然后被调度到另一个Socket上运行，此时，应用程序再进行内存访问时，就需要访问之前Socket上连接的内存，这种访问属于<strong>远端内存访问</strong>。<strong>和访问Socket直接连接的内存相比，远端内存访问会增加应用程序的延迟。</strong></p><p>在多CPU架构下，一个应用程序访问所在Socket的本地内存和访问远端内存的延迟并不一致，所以，我们也把这个架构称为非统一内存访问架构（Non-Uniform Memory Access，NUMA架构）。</p><p>到这里，我们就知道了主流的CPU多核架构和多CPU架构，我们来简单总结下CPU架构对应用程序运行的影响。</p><ul>
<li>L1、L2缓存中的指令和数据的访问速度很快，所以，充分利用L1、L2缓存，可以有效缩短应用程序的执行时间；</li>
<li>在NUMA架构下，如果应用程序从一个Socket上调度到另一个Socket上，就可能会出现远端内存访问的情况，这会直接增加应用程序的执行时间。</li>
</ul><p>接下来，我们就先来了解下CPU多核是如何影响Redis性能的。</p><h2>CPU多核对Redis性能的影响</h2><p>在一个CPU核上运行时，应用程序需要记录自身使用的软硬件资源信息（例如栈指针、CPU核的寄存器值等），我们把这些信息称为<strong>运行时信息</strong>。同时，应用程序访问最频繁的指令和数据还会被缓存到L1、L2缓存上，以便提升执行速度。</p><p>但是，在多核CPU的场景下，一旦应用程序需要在一个新的CPU核上运行，那么，运行时信息就需要重新加载到新的CPU核上。而且，新的CPU核的L1、L2缓存也需要重新加载数据和指令，这会导致程序的运行时间增加。</p><p>说到这儿，我想跟你分享一个我曾经在多核CPU环境下对Redis性能进行调优的案例。希望借助这个案例，帮你全方位地了解到多核CPU对Redis的性能的影响。</p><p>当时，我们的项目需求是要对Redis的99%尾延迟进行优化，要求GET尾延迟小于300微秒，PUT尾延迟小于500微秒。</p><p>可能有同学不太清楚99%尾延迟是啥，我先解释一下。我们把所有请求的处理延迟从小到大排个序，<strong>99%的请求延迟小于的值就是99%尾延迟</strong>。比如说，我们有1000个请求，假设按请求延迟从小到大排序后，第991个请求的延迟实测值是1ms，而前990个请求的延迟都小于1ms，所以，这里的99%尾延迟就是1ms。</p><p>刚开始的时候，我们使用GET/PUT复杂度为O(1)的String类型进行数据存取，同时关闭了RDB和AOF，而且，Redis实例中没有保存集合类型的其他数据，也就没有bigkey操作，避免了可能导致延迟增加的许多情况。</p><p>但是，即使这样，我们在一台有24个CPU核的服务器上运行Redis实例，GET和PUT的99%尾延迟分别是504微秒和1175微秒，明显大于我们设定的目标。</p><p>后来，我们仔细检测了Redis实例运行时的服务器CPU的状态指标值，这才发现，CPU的context switch次数比较多。</p><p>context switch是指线程的上下文切换，这里的上下文就是线程的运行时信息。在CPU多核的环境中，一个线程先在一个CPU核上运行，之后又切换到另一个CPU核上运行，这时就会发生context switch。</p><p>当context switch发生后，Redis主线程的运行时信息需要被重新加载到另一个CPU核上，而且，此时，另一个CPU核上的L1、L2缓存中，并没有Redis实例之前运行时频繁访问的指令和数据，所以，这些指令和数据都需要重新从L3缓存，甚至是内存中加载。这个重新加载的过程是需要花费一定时间的。而且，Redis实例需要等待这个重新加载的过程完成后，才能开始处理请求，所以，这也会导致一些请求的处理时间增加。</p><p>如果在CPU多核场景下，Redis实例被频繁调度到不同CPU核上运行的话，那么，对Redis实例的请求处理时间影响就更大了。<strong>每调度一次，一些请求就会受到运行时信息、指令和数据重新加载过程的影响，这就会导致某些请求的延迟明显高于其他请求</strong>。分析到这里，我们就知道了刚刚的例子中99%尾延迟的值始终降不下来的原因。</p><p>所以，我们要避免Redis总是在不同CPU核上来回调度执行。于是，我们尝试着把Redis实例和CPU核绑定了，让一个Redis实例固定运行在一个CPU核上。我们可以使用<strong>taskset命令</strong>把一个程序绑定在一个核上运行。</p><p>比如说，我们执行下面的命令，就把Redis实例绑在了0号核上，其中，“-c”选项用于设置要绑定的核编号。</p><pre><code>taskset -c 0 ./redis-server
</code></pre><p>绑定以后，我们进行了测试。我们发现，Redis实例的GET和PUT的99%尾延迟一下子就分别降到了260微秒和482微秒，达到了我们期望的目标。</p><p>我们来看一下绑核前后的Redis的99%尾延迟。</p><p><img src="https://static001.geekbang.org/resource/image/eb/57/eb72b9f58052d6a6023d3e1dac522157.jpg?wh=2760*477" alt=""></p><p>可以看到，在CPU多核的环境下，通过绑定Redis实例和CPU核，可以有效降低Redis的尾延迟。当然，绑核不仅对降低尾延迟有好处，同样也能降低平均延迟、提升吞吐率，进而提升Redis性能。</p><p>接下来，我们再来看看多CPU架构，也就是NUMA架构，对Redis性能的影响。</p><h2>CPU的NUMA架构对Redis性能的影响</h2><p>在实际应用Redis时，我经常看到一种做法，为了提升Redis的网络性能，把操作系统的网络中断处理程序和CPU核绑定。这个做法可以避免网络中断处理程序在不同核上来回调度执行，的确能有效提升Redis的网络处理性能。</p><p>但是，网络中断程序是要和Redis实例进行网络数据交互的，一旦把网络中断程序绑核后，我们就需要注意Redis实例是绑在哪个核上了，这会关系到Redis访问网络数据的效率高低。</p><p>我们先来看下Redis实例和网络中断程序的数据交互：网络中断处理程序从网卡硬件中读取数据，并把数据写入到操作系统内核维护的一块内存缓冲区。内核会通过epoll机制触发事件，通知Redis实例，Redis实例再把数据从内核的内存缓冲区拷贝到自己的内存空间，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/87/d2/8753ce6985fd08bb9cf9a3813c8b2cd2.jpg?wh=2108*1295" alt=""></p><p>那么，在CPU的NUMA架构下，当网络中断处理程序、Redis实例分别和CPU核绑定后，就会有一个潜在的风险：<strong>如果网络中断处理程序和Redis实例各自所绑的CPU核不在同一个CPU Socket上，那么，Redis实例读取网络数据时，就需要跨CPU Socket访问内存，这个过程会花费较多时间。</strong></p><p>这么说可能有点抽象，我再借助一张图来解释下。</p><p><img src="https://static001.geekbang.org/resource/image/30/b0/30cd42yy86debc0eb6e7c5b069533ab0.jpg?wh=3000*1251" alt=""></p><p>可以看到，图中的网络中断处理程序被绑在了CPU Socket 1的某个核上，而Redis实例则被绑在了CPU Socket  2上。此时，网络中断处理程序读取到的网络数据，被保存在CPU Socket  1的本地内存中，当Redis实例要访问网络数据时，就需要Socket 2通过总线把内存访问命令发送到 Socket 1上，进行远程访问，时间开销比较大。</p><p>我们曾经做过测试，和访问CPU Socket本地内存相比，跨CPU Socket的内存访问延迟增加了18%，这自然会导致Redis处理请求的延迟增加。</p><p>所以，为了避免Redis跨CPU Socket访问网络数据，我们最好把网络中断程序和Redis实例绑在同一个CPU Socket上，这样一来，Redis实例就可以直接从本地内存读取网络数据了，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/41/79/41f02b2afb08ec54249680e8cac30179.jpg?wh=2874*1479" alt=""></p><p>不过，需要注意的是，<strong>在CPU的NUMA架构下，对CPU核的编号规则，并不是先把一个CPU Socket中的所有逻辑核编完，再对下一个CPU Socket中的逻辑核编码，而是先给每个CPU Socket中每个物理核的第一个逻辑核依次编号，再给每个CPU Socket中的物理核的第二个逻辑核依次编号。</strong></p><p>我给你举个例子。假设有2个CPU Socket，每个Socket上有6个物理核，每个物理核又有2个逻辑核，总共24个逻辑核。我们可以执行<strong>lscpu命令</strong>，查看到这些核的编号：</p><pre><code>lscpu

Architecture: x86_64
...
NUMA node0 CPU(s): 0-5,12-17
NUMA node1 CPU(s): 6-11,18-23
...
</code></pre><p>可以看到，NUMA node0的CPU核编号是0到5、12到17。其中，0到5是node0上的6个物理核中的第一个逻辑核的编号，12到17是相应物理核中的第二个逻辑核编号。NUMA node1的CPU核编号规则和node0一样。</p><p>所以，在绑核时，我们一定要注意，不能想当然地认为第一个Socket上的12个逻辑核的编号就是0到11。否则，网络中断程序和Redis实例就可能绑在了不同的CPU Socket上。</p><p>比如说，如果我们把网络中断程序和Redis实例分别绑到编号为1和7的CPU核上，此时，它们仍然是在2个CPU Socket上，Redis实例仍然需要跨Socket读取网络数据。</p><p><strong>所以，你一定要注意NUMA架构下CPU核的编号方法，这样才不会绑错核。</strong></p><p>我们先简单地总结下刚刚学习的内容。在CPU多核的场景下，用taskset命令把Redis实例和一个核绑定，可以减少Redis实例在不同核上被来回调度执行的开销，避免较高的尾延迟；在多CPU的NUMA架构下，如果你对网络中断程序做了绑核操作，建议你同时把Redis实例和网络中断程序绑在同一个CPU Socket的不同核上，这样可以避免Redis跨Socket访问内存中的网络数据的时间开销。</p><p>不过，“硬币都是有两面的”，绑核也存在一定的风险。接下来，我们就来了解下它的潜在风险点和解决方案。</p><h2>绑核的风险和解决方案</h2><p>Redis除了主线程以外，还有用于RDB生成和AOF重写的子进程（可以回顾看下<a href="https://time.geekbang.org/column/article/271754">第4讲</a>和<a href="https://time.geekbang.org/column/article/271839">第5讲</a>）。此外，我们还在<a href="https://time.geekbang.org/column/article/285000">第16讲</a>学习了Redis的后台线程。</p><p>当我们把Redis实例绑到一个CPU逻辑核上时，就会导致子进程、后台线程和Redis主线程竞争CPU资源，一旦子进程或后台线程占用CPU时，主线程就会被阻塞，导致Redis请求延迟增加。</p><p>针对这种情况，我来给你介绍两种解决方案，分别是<strong>一个Redis实例对应绑一个物理核和优化Redis源码。</strong></p><p><strong>方案一：一个Redis实例对应绑一个物理核</strong></p><p>在给Redis实例绑核时，我们不要把一个实例和一个逻辑核绑定，而要和一个物理核绑定，也就是说，把一个物理核的2个逻辑核都用上。</p><p>我们还是以刚才的NUMA架构为例，NUMA node0的CPU核编号是0到5、12到17。其中，编号0和12、1和13、2和14等都是表示一个物理核的2个逻辑核。所以，在绑核时，我们使用属于同一个物理核的2个逻辑核进行绑核操作。例如，我们执行下面的命令，就把Redis实例绑定到了逻辑核0和12上，而这两个核正好都属于物理核1。</p><pre><code>taskset -c 0,12 ./redis-server
</code></pre><p>和只绑一个逻辑核相比，把Redis实例和物理核绑定，可以让主线程、子进程、后台线程共享使用2个逻辑核，可以在一定程度上缓解CPU资源竞争。但是，因为只用了2个逻辑核，它们相互之间的CPU竞争仍然还会存在。如果你还想进一步减少CPU竞争，我再给你介绍一种方案。</p><p><strong>方案二：优化Redis源码</strong></p><p>这个方案就是通过修改Redis源码，把子进程和后台线程绑到不同的CPU核上。</p><p>如果你对Redis的源码不太熟悉，也没关系，因为这是通过编程实现绑核的一个通用做法。学会了这个方案，你可以在熟悉了源码之后把它用上，也可以应用在其他需要绑核的场景中。</p><p>接下来，我先介绍一下通用的做法，然后，再具体说说可以把这个做法对应到Redis的哪部分源码中。</p><p>通过编程实现绑核时，要用到操作系统提供的1个数据结构cpu_set_t和3个函数CPU_ZERO、CPU_SET和sched_setaffinity，我先来解释下它们。</p><ul>
<li>cpu_set_t数据结构：是一个位图，每一位用来表示服务器上的一个CPU逻辑核。</li>
<li>CPU_ZERO函数：以cpu_set_t结构的位图为输入参数，把位图中所有的位设置为0。</li>
<li>CPU_SET函数：以CPU逻辑核编号和cpu_set_t位图为参数，把位图中和输入的逻辑核编号对应的位设置为1。</li>
<li>sched_setaffinity函数：以进程/线程ID号和cpu_set_t为参数，检查cpu_set_t中哪一位为1，就把输入的ID号所代表的进程/线程绑在对应的逻辑核上。</li>
</ul><p>那么，怎么在编程时把这三个函数结合起来实现绑核呢？很简单，我们分四步走就行。</p><ul>
<li>第一步：创建一个cpu_set_t结构的位图变量；</li>
<li>第二步：使用CPU_ZERO函数，把cpu_set_t结构的位图所有的位都设置为0；</li>
<li>第三步：根据要绑定的逻辑核编号，使用CPU_SET函数，把cpu_set_t结构的位图相应位设置为1；</li>
<li>第四步：使用sched_setaffinity函数，把程序绑定在cpu_set_t结构位图中为1的逻辑核上。</li>
</ul><p>下面，我就具体介绍下，分别把后台线程、子进程绑到不同的核上的做法。</p><p>先说后台线程。为了让你更好地理解编程实现绑核，你可以看下这段示例代码，它实现了为线程绑核的操作：</p><pre><code>//线程函数
void worker(int bind_cpu){
    cpu_set_t cpuset;  //创建位图变量
    CPU_ZERO(&amp;cpu_set); //位图变量所有位设置0
    CPU_SET(bind_cpu, &amp;cpuset); //根据输入的bind_cpu编号，把位图对应为设置为1
    sched_setaffinity(0, sizeof(cpuset), &amp;cpuset); //把程序绑定在cpu_set_t结构位图中为1的逻辑核

    //实际线程函数工作
}

int main(){
    pthread_t pthread1
    //把创建的pthread1绑在编号为3的逻辑核上
    pthread_create(&amp;pthread1, NULL, (void *)worker, 3);
}
</code></pre><p>对于Redis来说，它是在bio.c文件中的bioProcessBackgroundJobs函数中创建了后台线程。bioProcessBackgroundJobs函数类似于刚刚的例子中的worker函数，在这个函数中实现绑核四步操作，就可以把后台线程绑到和主线程不同的核上了。</p><p>和给线程绑核类似，当我们使用fork创建子进程时，也可以把刚刚说的四步操作实现在fork后的子进程代码中，示例代码如下：</p><pre><code>int main(){
   //用fork创建一个子进程
   pid_t p = fork();
   if(p &lt; 0){
      printf(&quot; fork error\n&quot;);
   }
   //子进程代码部分
   else if(!p){
      cpu_set_t cpuset;  //创建位图变量
      CPU_ZERO(&amp;cpu_set); //位图变量所有位设置0
      CPU_SET(3, &amp;cpuset); //把位图的第3位设置为1
      sched_setaffinity(0, sizeof(cpuset), &amp;cpuset);  //把程序绑定在3号逻辑核
      //实际子进程工作
      exit(0);
   }
   ...
}
</code></pre><p>对于Redis来说，生成RDB和AOF日志重写的子进程分别是下面两个文件的函数中实现的。</p><ul>
<li>rdb.c文件：rdbSaveBackground函数；</li>
<li>aof.c文件：rewriteAppendOnlyFileBackground函数。</li>
</ul><p>这两个函数中都调用了fork创建子进程，所以，我们可以在子进程代码部分加上绑核的四步操作。</p><p>使用源码优化方案，我们既可以实现Redis实例绑核，避免切换核带来的性能影响，还可以让子进程、后台线程和主线程不在同一个核上运行，避免了它们之间的CPU资源竞争。相比使用taskset绑核来说，这个方案可以进一步降低绑核的风险。</p><h2>小结</h2><p>这节课，我们学习了CPU架构对Redis性能的影响。首先，我们了解了目前主流的多核CPU架构，以及NUMA架构。</p><p>在多核CPU架构下，Redis如果在不同的核上运行，就需要频繁地进行上下文切换，这个过程会增加Redis的执行时间，客户端也会观察到较高的尾延迟了。所以，建议你在Redis运行时，把实例和某个核绑定，这样，就能重复利用核上的L1、L2缓存，可以降低响应延迟。</p><p>为了提升Redis的网络性能，我们有时还会把网络中断处理程序和CPU核绑定。在这种情况下，如果服务器使用的是NUMA架构，Redis实例一旦被调度到和中断处理程序不在同一个CPU Socket，就要跨CPU Socket访问网络数据，这就会降低Redis的性能。所以，我建议你把Redis实例和网络中断处理程序绑在同一个CPU Socket下的不同核上，这样可以提升Redis的运行性能。</p><p>虽然绑核可以帮助Redis降低请求执行时间，但是，除了主线程，Redis还有用于RDB和AOF重写的子进程，以及4.0版本之后提供的用于惰性删除的后台线程。当Redis实例和一个逻辑核绑定后，这些子进程和后台线程会和主线程竞争CPU资源，也会对Redis性能造成影响。所以，我给了你两个建议：</p><ul>
<li>如果你不想修改Redis代码，可以把按一个Redis实例一个物理核方式进行绑定，这样，Redis的主线程、子进程和后台线程可以共享使用一个物理核上的两个逻辑核。</li>
<li>如果你很熟悉Redis的源码，就可以在源码中增加绑核操作，把子进程和后台线程绑到不同的核上，这样可以避免对主线程的CPU资源竞争。不过，如果你不熟悉Redis源码，也不用太担心，Redis 6.0出来后，可以支持CPU核绑定的配置操作了，我将在第38讲中向你介绍Redis 6.0的最新特性。</li>
</ul><p>Redis的低延迟是我们永恒的追求目标，而多核CPU和NUMA架构已经成为了目前服务器的主流配置，所以，希望你能掌握绑核优化方案，并把它应用到实践中。</p><h2>每课一问</h2><p>按照惯例，我给你提个小问题。</p><p>在一台有2个CPU Socket（每个Socket 8个物理核）的服务器上，我们部署了有8个实例的Redis切片集群（8个实例都为主节点，没有主备关系），现在有两个方案：</p><ol>
<li>在同一个CPU Socket上运行8个实例，并和8个CPU核绑定；</li>
<li>在2个CPU Socket上各运行4个实例，并和相应Socket上的核绑定。</li>
</ol><p>如果不考虑网络数据读取的影响，你会选择哪个方案呢？</p><p>欢迎在留言区写下你的思考和答案，如果你觉得有所收获，也欢迎你帮我把今天的内容分享给你的朋友。我们下节课见。</p>
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
  <div class="_2_QraFYR_0">这篇文章收获很大！对于CPU结构和如何绑核有了进一步了解。其实在NUMA架构下，不光对于CPU的绑核需要注意，对于内存的使用，也有很多注意点，下面回答课后问题，也会提到NUMA架构下内存方面的注意事项。<br><br>在一台有2个CPU Socket（每个Socket 8个物理核）的服务器上，我们部署了有8个实例的Redis切片集群（8个实例都为主节点，没有主备关系），采用哪种方案绑核最佳？<br><br>我更倾向于的方案是：在两个CPU Socket上各运行4个实例，并和相应Socket上的核绑定。这么做的原因主要从L3 Cache的命中率、内存利用率、避免使用到Swap这三个方面考虑：<br><br>1、由于CPU Socket1和2分别有自己的L3 Cache，如果把所有实例都绑定在同一个CPU Socket上，相当于这些实例共用这一个L3 Cache，另一个CPU Socket的L3 Cache浪费了。这些实例共用一个L3 Cache，会导致Cache中的数据频繁被替换，访问命中率下降，之后只能从内存中读取数据，这会增加访问的延迟。而8个实例分别绑定CPU Socket，可以充分使用2个L3 Cache，提高L3 Cache的命中率，减少从内存读取数据的开销，从而降低延迟。<br><br>2、如果这些实例都绑定在一个CPU Socket，由于采用NUMA架构的原因，所有实例会优先使用这一个节点的内存，当这个节点内存不足时，再经过总线去申请另一个CPU Socket下的内存，此时也会增加延迟。而8个实例分别使用2个CPU Socket，各自在访问内存时都是就近访问，延迟最低。<br><br>3、如果这些实例都绑定在一个CPU Socket，还有一个比较大的风险是：用到Swap的概率将会大大提高。如果这个CPU Socket对应的内存不够了，也可能不会去另一个节点申请内存（操作系统可以配置内存回收策略和Swap使用倾向：本节点回收内存&#47;其他节点申请内存&#47;内存数据换到Swap的倾向程度），而操作系统可能会把这个节点的一部分内存数据换到Swap上从而释放出内存给进程使用（如果没开启Swap可会导致直接OOM）。因为Redis要求性能非常高，如果从Swap中读取数据，此时Redis的性能就会急剧下降，延迟变大。所以8个实例分别绑定CPU Socket，既可以充分使用2个节点的内存，提高内存使用率，而且触发使用Swap的风险也会降低。<br><br>其实我们可以查一下，在NUMA架构下，也经常发生某一个节点内存不够，但其他节点内存充足的情况下，依旧使用到了Swap，进而导致软件性能急剧下降的例子。所以在运维层面，我们也需要关注NUMA架构下的内存使用情况（多个内存节点使用可能不均衡），并合理配置系统参数（内存回收策略&#47;Swap使用倾向），尽量去避免使用到Swap。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-16 00:07:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/23/31/d2/56bf47c8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>薛定谔的猫</span>
  </div>
  <div class="_2_QraFYR_0">小白请教一下，网络中断处理程序是指什么呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 当网卡接收到数据后，会触发网卡中断，用来通知操作系统内核进行数据处理。因此，操作系统内核中用来处理网卡中断事件，把数据从内核的缓冲区拷贝到应用程序缓冲区的程序就是指网卡中断处理程序。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-25 13:24:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_9b08a5</span>
  </div>
  <div class="_2_QraFYR_0">1.作者讲了什么？<br>	在多核CPU架构和NUMA架构下，如何对redis进行优化配置<br>2.作者是怎么把这件事将明白的？<br>	1，讲解了主流的CPU架构，主要有多核CPU架构和NUMA架构两个架构<br>		多核CPU架构： 多个物理核，各物理核使用私有的1、2级缓存，共享3级缓存。物理核可包含2个超线程，称为逻辑核<br>		NUMA架构： 一个服务器上多个cpu，称为CPU Socket，每个cpu socker存在多个物理核。每个socket通过总线连接，并且有用私有的内存空间<br>3.为了讲明白，作者讲了哪些要点，哪些亮点？<br>	1、亮点：将主流的CPU架构进行剖析，使人更好理解cpu的原理，有助于后续redis性能的优化<br>	2、要点：cpu架构：一个cpu一般拥有多个物理核，每个物理核都拥有私有的一级缓存，二级缓存。三级缓存是各物理核共享的缓存空间。而物理核又可以分为多个超线程，称为逻辑核，同一个物理核的逻辑核会共享使用 L1、L2 缓存。<br>	3、要点：一级缓存和二级缓存访问延迟不超过10纳秒，但空间很小，只是KB单位。而应用程序访问内存延迟是百纳秒级别，基本上是一二级缓存的10倍<br>	4、要点：不同的物理核还会共享一个共同的三级缓存，三级缓存空间比较多，为几到几十MB，当 L1、L2 缓存中没有数据缓存时，可以访问 L3，尽可能避免访问内存。<br>	5、要点：多核CPU运行redis实例，会导致context switch，导致增加延迟，可以通过taskset 命令把redis进程绑定到某个cup物理核上。<br>	6、要点：NUMA架构运行redis实例，如果网络中断程序和redis实例运行在不同的socket上，就需要跨 CPU Socket 访问内存，这个过程会花费较多时间。<br>	7、要点：绑核的风险和解决方案：<br>			一个 Redis 实例对应绑一个物理核 ： 将redis服务绑定到一个物理核上，而不是一个逻辑核上，如 taskset -c 0,12 .&#47;redis-server<br>			优化 Redis 源码。<br>4.对于作者所讲的，我有哪些发散性思考？<br>给自己提了几个问题：<br>	1，在多核CPU架构和NUMA架构，那个对于redis来说性能比较好<br>	2，如何设置网络中断处理和redis绑定设置在同个socket上呢？<br><br>5.将来在哪些场景里，我能够使用它？<br><br>6.留言区收获<br>	如果redis实例中内存不足以使用时，会用到swap那会怎么样？（答案来自@kaito 大佬）<br>		因为Redis要求性能非常高，如果从Swap中读取数据，此时Redis的性能就会急剧下降，延迟变大。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-30 15:39:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/23/cd/de/0334fd13.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>许峰</span>
  </div>
  <div class="_2_QraFYR_0">阿里云ecs主机都是vcpus, 这玩意算物理核心吗?<br>比如一个4vcpu, lscpu可以看到<br>NUMA node0 CPU(s):     0-3<br>这么绑?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: ECS主机提供的vCPU是指虚拟核，一般对应一个物理核心上的一个超线程，这是因为底层服务器一般会开启超线程。通常，一个物理核心会对应2个超线程，每个超线程对应一个vCPU。多个vCPU一般是在同一个NUMA节点上。<br><br>如果希望减少CPU超线程对性能的影响，可以通过阿里云SDK的选项关闭超线程。<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-30 20:25:32</div>
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
  <div class="_2_QraFYR_0">课后问题：我会选择方案二。首先一个实例不止有一个线程需要运行，所以方案一肯定会有CPU竞争问题；其次切片集群的通信不是通过内存，而是通过网络IO。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-16 08:49:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/91/c4/bcdcda65.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>明月几时</span>
  </div>
  <div class="_2_QraFYR_0">很多人都认为 Redis 和 CPU 的关系很简单，就是 Redis 的线程在 CPU 上运行，CPU 快，Redis 处理请求的速度也很快。<br>这种认知其实是片面的。CPU 的多核架构以及多 CPU 架构，也会影响到 Redis 的性能。如果不了解 CPU 对 Redis 的影响，在对 Redis 的性能进行调优时，就可能会遗漏一些调优方法，不能把 Redis 的性能发挥到极限。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: CPU有多核，即使单核上也会有超线程技术。除了多核，多处理器会形成NUMA架构，这些都会对系统性能产生影响。<br><br>所以，计算机体系结构的知识点对系统优化还是很有帮助的:)<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-12 23:35:10</div>
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
  <div class="_2_QraFYR_0">有两套房子，就不用挤着睡吧，优选方案二。老师实验用的X86的CPU吧，对于ARM架构来讲，存在着跨DIE和跨P的说法，跨P的访问时延会更高，且多个P之间的访问存在着NUMA distances的说法，不同的布局导致的跨P访问时延也不相同。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-16 15:29:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/5c/8f/551b5624.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小可</span>
  </div>
  <div class="_2_QraFYR_0">这篇文章真是太好了！对cpu有了更多的认识，公司服务lscpu挨个看了一遍，不懂的地方也去查了资料，自己也画了NUMA架构下多个cpu socket示意图，给每个逻辑cpu编号，对照图看怎么绑定网络中断和redis实例到同一个cpu socket，怎么绑定一个redis实例到同一个物理核，非常清晰！还有cpu的架构设计思路也可以应用到我们实际系统架构上，不得不赞叹这些神级设计，也感谢老师心细深入的讲解，真的发现宝藏了，O(∩_∩)O哈哈~</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-04 11:35:38</div>
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
  <div class="_2_QraFYR_0">在 NUMA 架构下，比如有两个 CPU Socket：CPU Socket 1 和 CPU Socket 2，每个 CPU Socket 都有自己的内存，CPU Socket 1 有自己的内存 Mem1，CPU Socket 2 有自己的内存 Mem2。<br><br>Redis 实例在 CPU Socket 1 上执行，网络中断处理程序在 CPU Socket 2 上执行，所以 Redis 实例的数据在内存 Mem1 上，网络中断处理程序的数据在 Mem2上。<br><br>因此 Redis 实例读取网络中断处理程序的内存数据（Mem2）时，是需要远端访问的，比直接访问自己的内存数据（Mem1）要慢。<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-16 14:54:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/20/e9/32/b89fcc77.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>元末</span>
  </div>
  <div class="_2_QraFYR_0">这篇文章很顶</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-13 16:52:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/61/23/30d134cf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Young</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，有个疑问： 即使内核绑定，但是当cpu时间片用尽，context switch依然会发生对吧？ 之后，cache里的数据会被刷掉， 所谓绑定的优势如何保证呢？ 谢谢！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-06 00:16:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/ca/4b/c1ace3aa.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>蚝不鱿鱼</span>
  </div>
  <div class="_2_QraFYR_0">结合隔壁我浩哥的计算机组成原理课程食用本节内容是真的香，感谢钧哥。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-13 16:33:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/77/2a/4f72941e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>cpzhao</span>
  </div>
  <div class="_2_QraFYR_0">挺有收获，以前学习比较少关注系统cpu结构这块。这次顺带也了解cpu亲和度、NUMA结构相关的知识点，希望老师也可以在文章中推荐一些相关知识点的学习链接之类的。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-16 00:11:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/3e/3a/2267d2a3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>hoppo</span>
  </div>
  <div class="_2_QraFYR_0">这篇文章确实收获很大，从CPU核心说到NUMA架构、我原来其实就是抱着 ”Redis 的线程在 CPU 上运行，CPU 越快，Redis 处理请求的速度也越快”相法的。现在想来真是太肤浅了...orz（失意体前屈）<br><br>不过一步一步跟着老师的思路来，还是很容易理解的，读到远端内存访问影响性能的时候，就会想是不是可以分到一个核上；看完了绑核的优点介绍又联系到风险和解决方式，一气呵成，给老师点个赞~</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-15 22:55:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/b6/75/32c19395.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>土豆白菜</span>
  </div>
  <div class="_2_QraFYR_0">老师，我也想问下比如azure redis 能否做这些优化</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-16 23:10:29</div>
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
  <div class="_2_QraFYR_0">请问老师，您文中提到我们仔细检测了 Redis 实例运行时的服务器 CPU 的状态指标值，这才发现，CPU 的 context switch 次数比较多。再遇到这样的问题的时候，排查的点有哪些呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-16 10:25:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/e6/9a/d0725a24.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Athena</span>
  </div>
  <div class="_2_QraFYR_0">这章是真的牛逼，反复看了几次</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-05 11:50:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/69/bc/c22bf219.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>妥妥</span>
  </div>
  <div class="_2_QraFYR_0">老师请教一下，不修改redis源码的情况下，为什么不干脆绑定同一个cpu socket下的三个核心？这样就不会有cpu资源的竞争了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-16 18:33:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2e/3c/08/93bde3ee.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>⚽️</span>
  </div>
  <div class="_2_QraFYR_0">网络中断和cpu怎么绑定啊</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-29 00:50:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/23/4c/89/82a3ee04.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>going</span>
  </div>
  <div class="_2_QraFYR_0">同一个socket运行八个实例。<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-05 09:42:53</div>
  </div>
</div>
</div>
</li>
</ul>