<audio title="01 _ CPU缓存：怎样写代码能够让CPU执行得更快？" src="https://static001.geekbang.org/resource/audio/7b/1d/7b26aa9f7d8ba80b35c0ecaba009151d.mp3" controls="controls"></audio> 
<p>你好，我是陶辉。</p><p>这是课程的第一讲，我们先从主机最重要的部件CPU开始，聊聊如何通过提升CPU缓存的命中率来优化程序的性能。</p><p>任何代码的执行都依赖CPU，通常，使用好CPU是操作系统内核的工作。然而，当我们编写计算密集型的程序时，CPU的执行效率就开始变得至关重要。由于CPU缓存由更快的SRAM构成（内存是由DRAM构成的），而且离CPU核心更近，如果运算时需要的输入数据是从CPU缓存，而不是内存中读取时，运算速度就会快很多。所以，了解CPU缓存对性能的影响，便能够更有效地编写我们的代码，优化程序性能。</p><p>然而，很多同学并不清楚CPU缓存的运行规则，不知道如何写代码才能够配合CPU缓存的工作方式，这样，便放弃了可以大幅提升核心计算代码执行速度的机会。而且，越是底层的优化，适用范围越广，CPU缓存便是如此，它的运行规则对分布式集群里各种操作系统、编程语言都有效。所以，一旦你能掌握它，集群中巨大的主机数量便能够放大优化效果。</p><p>接下来，我们就看看，CPU缓存结构到底是什么样的，又该如何优化它？</p><h2>CPU的多级缓存</h2><p>刚刚我们提到，CPU缓存离CPU核心更近，由于电子信号传输是需要时间的，所以离CPU核心越近，缓存的读写速度就越快。但CPU的空间很狭小，离CPU越近缓存大小受到的限制也越大。所以，综合硬件布局、性能等因素，CPU缓存通常分为大小不等的三级缓存。</p><!-- [[[read_end]]] --><p>CPU缓存的材质SRAM比内存使用的DRAM贵许多，所以不同于内存动辄以GB计算，它的大小是以MB来计算的。比如，在我的Linux系统上，离CPU最近的一级缓存是32KB，二级缓存是256KB，最大的三级缓存则是20MB（Windows系统查看缓存大小可以用wmic cpu指令，或者用<a href="https://www.cpuid.com/softwares/cpu-z.html">CPU-Z</a>这个工具）。</p><p><img src="https://static001.geekbang.org/resource/image/de/87/deff13454dcb6b15e1ac4f6f538c4987.png?wh=1379*352" alt=""></p><p>你可能注意到，三级缓存要比一、二级缓存大许多倍，这是因为当下的CPU都是多核心的，每个核心都有自己的一、二级缓存，但三级缓存却是一颗CPU上所有核心共享的。</p><p>程序执行时，会先将内存中的数据载入到共享的三级缓存中，再进入每颗核心独有的二级缓存，最后进入最快的一级缓存，之后才会被CPU使用，就像下面这张图。</p><p><img src="https://static001.geekbang.org/resource/image/92/0c/9277d79155cd7f925c27f9c37e0b240c.jpg?wh=3749*2433" alt=""></p><p>缓存要比内存快很多。CPU访问一次内存通常需要100个时钟周期以上，而访问一级缓存只需要4~5个时钟周期，二级缓存大约12个时钟周期，三级缓存大约30个时钟周期（对于2GHZ主频的CPU来说，一个时钟周期是0.5纳秒。你可以在LZMA的<a href="https://www.7-cpu.com/">Benchmark</a>中找到几种典型CPU缓存的访问速度）。</p><p>如果CPU所要操作的数据在缓存中，则直接读取，这称为缓存命中。命中缓存会带来很大的性能提升，<strong>因此，我们的代码优化目标是提升CPU缓存的命中率。</strong></p><p>当然，缓存命中率是很笼统的，具体优化时还得一分为二。比如，你在查看CPU缓存时会发现有2个一级缓存（比如Linux上就是上图中的index0和index1），这是因为，CPU会区别对待指令与数据。比如，“1+1=2”这个运算，“+”就是指令，会放在一级指令缓存中，而“1”这个输入数字，则放在一级数据缓存中。虽然在冯诺依曼计算机体系结构中，代码指令与数据是放在一起的，但执行时却是分开进入指令缓存与数据缓存的，因此我们要分开来看二者的缓存命中率。</p><h2>提升数据缓存的命中率</h2><p>我们先来看数据的访问顺序是如何影响缓存命中率的。</p><p>比如现在要遍历二维数组，其定义如下（这里我用的是伪代码，在GitHub上我为你准备了可运行验证的C/C++、Java<a href="https://github.com/russelltao/geektime_distrib_perf/tree/master/1-cpu_cache/traverse_2d_array">示例代码</a>，你可以参照它们编写出其他语言的可执行代码）：</p><pre><code>       int array[N][N];
</code></pre><p>你可以思考一下，用array[j][i]和array[i][j]访问数组元素，哪一种性能更快？</p><pre><code>       for(i = 0; i &lt; N; i+=1) {
           for(j = 0; j &lt; N; j+=1) {
               array[i][j] = 0;
           }
       }
</code></pre><p>在我给出的GitHub地址上的C++代码实现中，前者array[j][i]执行的时间是后者array[i][j]的8倍之多（请参考<a href="https://github.com/russelltao/geektime_distrib_perf/tree/master/1-cpu_cache/traverse_2d_array">traverse_2d_array.cpp</a>，如果使用Python代码，traverse_2d_array.py由于数组容器的差异，性能差距不会那么大）。</p><p>为什么会有这么大的差距呢？这是因为二维数组array所占用的内存是连续的，比如若长度N的值为2，那么内存中从前至后各元素的顺序是：</p><pre><code>array[0][0]，array[0][1]，array[1][0]，array[1][1]。
</code></pre><p>如果用array[i][j]访问数组元素，则完全与上述内存中元素顺序一致，因此访问array[0][0]时，缓存已经把紧随其后的3个元素也载入了，CPU通过快速的缓存来读取后续3个元素就可以。如果用array[j][i]来访问，访问的顺序就是：</p><pre><code>array[0][0]，array[1][0]，array[0][1]，array[1][1]
</code></pre><p>此时内存是跳跃访问的，如果N的数值很大，那么操作array[j][i]时，是没有办法把array[j+1][i]也读入缓存的。</p><p>到这里我们还有2个问题没有搞明白：</p><ol>
<li>为什么两者的执行时间有约7、8倍的差距呢？</li>
<li>载入array[0][0]元素时，缓存一次性会载入多少元素呢？</li>
</ol><p>其实这两个问题的答案都与CPU Cache Line相关，它定义了缓存一次载入数据的大小，Linux上你可以通过coherency_line_size配置查看它，通常是64字节。</p><p><img src="https://static001.geekbang.org/resource/image/7d/de/7dc8d0c5a1461d9aed086e7a112c01de.png?wh=1226*64" alt=""></p><p>因此，我测试的服务器一次会载入64字节至缓存中。当载入array[0][0]时，若它们占用的内存不足64字节，CPU就会顺序地补足后续元素。顺序访问的array[i][j]因为利用了这一特点，所以就会比array[j][i]要快。也正因为这样，当元素类型是4个字节的整数时，性能就会比8个字节的高精度浮点数时速度更快，因为缓存一次载入的元素会更多。</p><p><strong>因此，遇到这种遍历访问数组的情况时，按照内存布局顺序访问将会带来很大的性能提升。</strong></p><p>再来看为什么执行时间相差8倍。在二维数组中，其实第一维元素存放的是地址，第二维存放的才是目标元素。由于64位操作系统的地址占用8个字节（32位操作系统是4个字节），因此，每批Cache Line最多也就能载入不到8个二维数组元素，所以性能差距大约接近8倍。（用不同的步长访问数组，也能验证CPU Cache Line对性能的影响，可参考我给你准备的<a href="https://github.com/russelltao/geektime_distrib_perf/tree/master/1-cpu_cache/traverse_1d_array">Github</a>上的测试代码）。</p><p>关于CPU Cache Line的应用其实非常广泛，如果你用过Nginx，会发现它是用哈希表来存放域名、HTTP头部等数据的，这样访问速度非常快，而哈希表里桶的大小如server_names_hash_bucket_size，它默认就等于CPU Cache Line的值。由于所存放的字符串长度不能大于桶的大小，所以当需要存放更长的字符串时，就需要修改桶大小，但Nginx官网上明确建议它应该是CPU Cache Line的整数倍。</p><p><img src="https://static001.geekbang.org/resource/image/4f/2b/4fa0080e0f688bd484fe701686e6262b.png?wh=1124*211" alt=""></p><p>为什么要做这样的要求呢？就是因为按照cpu cache line（比如64字节）来访问内存时，不会出现多核CPU下的伪共享问题，可以<strong>尽量减少访问内存的次数</strong>。比如，若桶大小为64字节，那么根据地址获取字符串时只需要访问一次内存，而桶大小为50字节，会导致最坏2次访问内存，而70字节最坏会有3次访问内存。</p><p>如果你在用Linux操作系统，可以通过一个名叫Perf的工具直观地验证缓存命中的情况（可以用yum install perf或者apt-get install perf安装这个工具，这个<a href="http://www.brendangregg.com/perf.html">网址</a>中有大量案例可供参考）。</p><p>执行perf stat可以统计出进程运行时的系统信息（通过-e选项指定要统计的事件，如果要查看三级缓存总的命中率，可以指定缓存未命中cache-misses事件，以及读取缓存次数cache-references事件，两者相除就是缓存的未命中率，用1相减就是命中率。类似的，通过L1-dcache-load-misses和L1-dcache-loads可以得到L1缓存的命中率），此时你会发现array[i][j]的缓存命中率远高于array[j][i]。</p><p>当然，perf stat还可以通过指令执行速度反映出两种访问方式的优劣，如下图所示（instructions事件指明了进程执行的总指令数，而cycles事件指明了运行的时钟周期，二者相除就可以得到每时钟周期所执行的指令数，缩写为IPC。如果缓存未命中，则CPU要等待内存的慢速读取，因此IPC就会很低。array[i][j]的IPC值也比array[j][i]要高得多）：</p><p><img src="https://static001.geekbang.org/resource/image/29/1c/29d4a9fa5b8ad4515d7129d71987b01c.png?wh=1662*235" alt=""><br>
<img src="https://static001.geekbang.org/resource/image/94/3d/9476f52cfc63825e7ec836580e12c53d.png?wh=1646*242" alt=""></p><h2>提升指令缓存的命中率</h2><p>说完数据的缓存命中率，再来看指令的缓存命中率该如何提升。</p><p>我们还是用一个例子来看一下。比如，有一个元素为0到255之间随机数字组成的数组：</p><pre><code>int array[N];
for (i = 0; i &lt; TESTN; i++) array[i] = rand() % 256;
</code></pre><p>接下来要对它做两个操作：一是循环遍历数组，判断每个数字是否小于128，如果小于则把元素的值置为0；二是将数组排序。那么，先排序再遍历速度快，还是先遍历再排序速度快呢？</p><pre><code>for(i = 0; i &lt; N; i++) {
       if (array [i] &lt; 128) array[i] = 0;
}
sort(array, array +N);
</code></pre><p>我先给出答案：先排序的遍历时间只有后排序的三分之一（参考GitHub中的<a href="https://github.com/russelltao/geektime_distrib_perf/tree/master/1-cpu_cache/branch_predict">branch_predict.cpp代码</a>）。为什么会这样呢？这是因为循环中有大量的if条件分支，而CPU<strong>含有分支预测器</strong>。</p><p>当代码中出现if、switch等语句时，意味着此时至少可以选择跳转到两段不同的指令去执行。如果分支预测器可以预测接下来要在哪段代码执行（比如if还是else中的指令），就可以提前把这些指令放在缓存中，CPU执行时就会很快。当数组中的元素完全随机时，分支预测器无法有效工作，而当array数组有序时，分支预测器会动态地根据历史命中数据对未来进行预测，命中率就会非常高。</p><p>究竟有多高呢？我们还是用Linux上的perf来做个验证。使用 -e选项指明branch-loads事件和branch-load-misses事件，它们分别表示分支预测的次数，以及预测失败的次数。通过L1-icache-load-misses也能查看到一级缓存中指令的未命中情况。</p><p>下图是我在GitHub上为你准备的验证程序执行的perf分支预测统计数据（代码见<a href="https://github.com/russelltao/geektime_distrib_perf/tree/master/1-cpu_cache/branch_predict">这里</a>），你可以看到，先排序的话分支预测的成功率非常高，而且一级指令缓存的未命中率也有大幅下降。</p><p><img src="https://static001.geekbang.org/resource/image/29/72/2902b3e08edbd1015b1e9ecfe08c4472.png?wh=554*90" alt=""></p><p><img src="https://static001.geekbang.org/resource/image/95/60/9503d2c8f7deb3647eebb8d68d317e60.png?wh=554*96" alt=""></p><p>C/C++语言中编译器还给应用程序员提供了显式预测分支概率的工具，如果if中的条件表达式判断为“真”的概率非常高，我们可以用likely宏把它括在里面，反之则可以用unlikely宏。当然，CPU自身的条件预测已经非常准了，仅当我们确信CPU条件预测不会准，且我们能够知晓实际概率时，才需要加入这两个宏。</p><pre><code>#define likely(x) __builtin_expect(!!(x), 1) 
#define unlikely(x) __builtin_expect(!!(x), 0)
if (likely(a == 1)) …
</code></pre><h2>提升多核CPU下的缓存命中率</h2><p>前面我们都是面向一个CPU核心谈数据及指令缓存的，然而现代CPU几乎都是多核的。虽然三级缓存面向所有核心，但一、二级缓存是每颗核心独享的。我们知道，即使只有一个CPU核心，现代分时操作系统都支持许多进程同时运行。这是因为操作系统把时间切成了许多片，微观上各进程按时间片交替地占用CPU，这造成宏观上看起来各程序同时在执行。</p><p>因此，若进程A在时间片1里使用CPU核心1，自然也填满了核心1的一、二级缓存，当时间片1结束后，操作系统会让进程A让出CPU，基于效率并兼顾公平的策略重新调度CPU核心1，以防止某些进程饿死。如果此时CPU核心1繁忙，而CPU核心2空闲，则进程A很可能会被调度到CPU核心2上运行，这样，即使我们对代码优化得再好，也只能在一个时间片内高效地使用CPU一、二级缓存了，下一个时间片便面临着缓存效率的问题。</p><p>因此，操作系统提供了将进程或者线程绑定到某一颗CPU上运行的能力。如Linux上提供了sched_setaffinity方法实现这一功能，其他操作系统也有类似功能的API可用。我在GitHub上提供了一个示例程序（代码见<a href="https://github.com/russelltao/geektime_distrib_perf/tree/master/1-cpu_cache/cpu_migrate">这里</a>），你可以看到，当多线程同时执行密集计算，且CPU缓存命中率很高时，如果将每个线程分别绑定在不同的CPU核心上，性能便会获得非常可观的提升。Perf工具也提供了cpu-migrations事件，它可以显示进程从不同的CPU核心上迁移的次数。</p><h2>小结</h2><p>今天我给你介绍了CPU缓存对程序性能的影响。这是很底层的性能优化，它对各种编程语言做密集计算时都有效。</p><p>CPU缓存分为数据缓存与指令缓存，对于数据缓存，我们应在循环体中尽量操作同一块内存上的数据，由于缓存是根据CPU Cache Line批量操作数据的，所以顺序地操作连续内存数据时也有性能提升。</p><p>对于指令缓存，有规律的条件分支能够让CPU的分支预测发挥作用，进一步提升执行效率。对于多核系统，如果进程的缓存命中率非常高，则可以考虑绑定CPU来提升缓存命中率。</p><h2>思考题</h2><p>最后请你思考下，多线程并行访问不同的变量，这些变量在内存布局是相邻的（比如类中的多个变量），此时CPU缓存就会失效，为什么？又该如何解决呢？欢迎你在留言区与大家一起探讨。</p><p>感谢阅读，如果你觉得这节课对你有一些启发，也欢迎把它分享给你的朋友。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/82/3d/356fc3d6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>忆水寒</span>
  </div>
  <div class="_2_QraFYR_0">因为在多核CPU时代，CPU有“缓存一致性”原则，也就是说每个处理器（核）都会通过嗅探在总线上传播的数据来检查自己的缓存值是不是过期了。如果过期了，则失效。比如声明volitate，当变量被修改，则会立即要求写入系统内存。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 讲得非常好！忆水寒同学对底层知识理解地很透彻！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-27 19:07:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/f6/00/2a248fd8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>二星球</span>
  </div>
  <div class="_2_QraFYR_0">一片连续的内存被加载到不同cpu核心中（就是同一个cache line在不同的cpu核心），其中一个cpu核心中修改cache line,其它核心都失效，加锁也是加在cache  line上，其它核心线程也被锁住，降低了性能。解决办法是填充无用字节数，使分开</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 完全正确！二星球同学用过填充法写代码么？或者看到过这样的开源代码？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-27 23:39:10</div>
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
  <div class="_2_QraFYR_0">配置 nginx server_names_bucket_siz 的大小<br>而桶大小为 50 字节，会导致最坏 2 次访问内存，而 70 字节最坏会有 3 次访问内存。<br>----------------------------------------------------<br>为什么 50字节会访问2次内存呢？ 不是可以加载 64k数据到缓存，包含了 50个字节，一次不就够了吗？<br>70k 也是同样的问题，为什么是3次啊</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你好一步，这里的原理其实与思考题是一致的，因为当所有bucket连续时，某个50字节的bucket一定会横跨2个cpu cache line，比如第2个bucket在内存上占用50-100字节，这样当CPU1访问第1个bucket时，会把第2个bucket的数据也载入，这样CPU2访问第2个bucket时，若第1个bucket发生变化，就会导致CPU2必须重新读入bucket。类似的，某个70个字节的bucket一定会占用到3个cpu cache line。文章中写得不够细，我再稍微调整补充下好了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-28 17:51:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/82/3d/356fc3d6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>忆水寒</span>
  </div>
  <div class="_2_QraFYR_0">这个文章其实讲解的很细致，来龙去脉都说清楚了。不错！<br>其实每篇文章能讲到这个地步，作为读者（也可以称为学生）每篇能够学到一个哪怕很小的知识点，那也是值得的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 最初这篇文章有6千字，想在一篇文章中讲清楚CPU缓存太难啦，后来编辑小姐姐协助我一点点删下来，还是担心读者看不懂，找不到重点。<br>忆水寒同学的知识很扎实，你点赞我就放心啦^_^。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-27 19:10:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4a/6f/e36b3908.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>xzy</span>
  </div>
  <div class="_2_QraFYR_0">思考题：<br>数据从内存加载到高速缓存中，以块为基本单元（一个块64字节），相邻的两个变量很可能在同一块中，当这个数据块分别加载到两颗cpu的高速缓存中时，只要一个cpu对该块（高速缓存中缓存的块）进行写操作，那么另一cpu缓存的该块将失效。<br>可以通过将两个变量放到不同的缓存块中，来解决这个问题</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，路阳同学思路完全正确！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-27 21:16:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/0c/52/f25c3636.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>长脖子树</span>
  </div>
  <div class="_2_QraFYR_0">多线程并行访问不同的变量,  cpu 缓存的失效需要基于其中有一个 core 将 L1&#47;L2 cache 写回主存, 但这个时间是不固定的. 除非你使用  lock 等强制刷新到 主存 , 而其他 core 上的缓存行会置为 Invalid , 这就要提到 缓存一致性 MESIF 协议 <br>如果这些变量在内存布局中相邻的, 很有可能在同一个 cache line 中,  要避免竞争, 就要避免在同一个 缓存行中, 比如 java 中的方法:  手动在这个变量前后 padding 间隔开一定字节(一般 64 字节) 或者 @Contended 标记这个变量 </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你好长脖子树，你的答案完全正确，而且非常详细，长脖子树提到的Intel MESIF 协议，以及Java中的@Contended注解非常专业，请其他同学参考！！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-28 23:43:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/c8/34/fb871b2c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>海罗沃德</span>
  </div>
  <div class="_2_QraFYR_0">第一篇就进入知识盲区了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你好海罗，这门课虽然只有30讲，但会涵盖绝大部分性能优化点，所以每篇文章都会涉及不同的知识。这一篇我希望你能明白CPU缓存的用途，这是成为高手必须了解的知识。<br>比如，我在protobuf那一讲中，还会讲到protobuf是怎么利用缓存的。所以，你可以先有这么一个概念，在后续用到的时候，再回过头来看，就更容易理解了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-27 20:03:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/18/81/83b6ade2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>好吃不贵</span>
  </div>
  <div class="_2_QraFYR_0">思考题猜测是False sharing导致的，非常热的数据最好cache line对齐。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，好吃不贵的答案更加简洁！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-28 11:56:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/63/0b/ad56aeb4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>AI</span>
  </div>
  <div class="_2_QraFYR_0">CPU性能优化的4个点：<br>1.按顺序访问数据（操作连续内存）：利用数据缓存，提高读数据缓存的命中率。<br>2.有规律的条件分支（如数据集先排序再处理）：利用指令缓存，提高读指令缓存的命中率。<br>3.数据按缓存行大小填充&#47;对齐（通常为64字节）：防止伪共享，提高并发处理能力和缓存命中率。<br>4.对于多核处理器，如果缓存命中率很高，可以考虑进行CPU绑定。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢二进制之路的全面总结！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-25 20:42:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/30/6e/581b0307.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>鲤鲤鱼</span>
  </div>
  <div class="_2_QraFYR_0">陶老师我们集群有一个问题，某一台物理机的CPU会被Hadoop yarn的查询任务打满，并且占用最多的pid在不停的变化，我查看了TIME_WAIT的个数好像也不是很多，在顶峰的时候还没达到一万，能够持续一两个小时。这个问题您有没有什么思路呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 解决性能问题，一般有两种方法：经验派和“理论”派，前者就是基于自己的经验概率，将能想到的优化方法都试一遍，这种方式通常又有效又快速，但无法解决复杂的问题。<br>所谓理论派，就是沿着固定的思路，使用二分法，从高至低慢慢下沉到细节。具体到你的问题，我建议你先看看，CPU占用是用户态还是系统态，用户态的话就要分析代码了，系统态还要进一步分析。火焰图通常是个很好的办法，虽然搭能画火焰图的环境很麻烦，但这种底层方法很有效的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-27 18:53:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTICBNZjA9hW65x6g9b2iaicKUJW5gxFxtgPXH9Cqp6eyFfY1sD2hVY4dZrY5pmoK2r1KZEiaaIKocdZQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>赖阿甘</span>
  </div>
  <div class="_2_QraFYR_0">此时内存是跳跃访问的，如果 N 的数值很大，那么操作 array[j][i]时，是没有办法把 array[j][i+1]也读入缓存的。<br>---------------------------------------------------------------------------------------------<br>老师是不是写错了，应该是”那么操作 array[j][i]时，是没有办法把 array[j+1][i]（而不是array[j][i+1]）也读入缓存的。”</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你好赖阿甘，你读得很仔细，完全正确，我笔误啦，非常感谢你的提醒，稍候我会联系编辑小姐姐更正的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-28 00:05:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/67/dd/55aa6e07.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>罗帮奎</span>
  </div>
  <div class="_2_QraFYR_0">多线程并发，如果一个core上的线程改变了变量，而其他core正好映射到相同的cacheLine上，则底层硬件会宣布所有其他core的缓存行失效。其它core下次需要重新去从主存中读取数据，来获取新的cacheLine。在多核系统上，这样会有严重的性能问题</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 正确！^_^</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-30 15:05:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/d5/1f/9fbd95ac.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>+1day</span>
  </div>
  <div class="_2_QraFYR_0">陶老师您好，不太熟悉底层的知识，问题可能有点小白，没有在网上搜索到答案，希望您能解答，感谢。<br>1. 执行时间相差 8 倍中，64 位操作系统缓存一次载入 64 字节数据的话，为什么每批 Cache Line 最多只能载入不到 8 个二维数组元素呢？元素类型是整型，大小为 4 字节的话，一次不是可以读取 16 个元素的吗？<br>2. Nginx 的例子中，桶的大小为 50 字节，一次载入 64 字节不应该可以一次访问就加载到缓存中了吗？是不是这里所说的缓存单纯指的一级缓存呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-28 23:06:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4f/3f/6f62f982.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>王坤祥</span>
  </div>
  <div class="_2_QraFYR_0">前几天学习了一下计算机组成原理。大部分都能听懂，开心~~~<br><br>本节讲到的性能优化实际上是涉及到了计算机组成原理中的【内存的局部性原理】以及【cpu的分支预测】。<br><br>对于课后的问题，因为多线程操作某些共享变量，涉及到变量的有效性问题（是否过期），如果变量在一个线程被修改，其他核心中的缓存失效啦。其他线程调用该变量的时候会从内存中重新加载到缓存。<br><br>所谓的如何解决，应该是解决缓存失效和保持数据一致性的问题，应该满足两点：<br>1. 写传播，即通知其他核心，某个缓存失效，需要从内存读取一下<br>2. 保证事务串行化，事务请求的顺序不能变化<br>我看资料了解到解决方案是基于总线嗅探机制的MESI协议来解决数据一致性问题。<br><br>在Java中，volatile 会确保我们对于这个变量的读取和写入，都一定会同步到主内存里，而不是从 Cache 里面读取，保证了数据一致性问题。 <br><br>这是我最近学习计算机组成原理后见解，不知道自己理解的有没有问题。有问题希望陶辉老师指正一下。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你好ByiProX，能够从留言中感受到你最近的进步，非常棒！      你的理解都是正确的，关于思考题你可以看下二星球的留言，他说得很清楚^_^</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-27 23:48:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/64/05/6989dce6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我来也</span>
  </div>
  <div class="_2_QraFYR_0">老师代码准备的真多！<br><br>思考题有同学已经回答的非常准确了。<br>虽然已经看过了 linux性能优化 计算机组成原理 和 性能工程高手课 ，但看起老师的文章还是很有意思。<br>一些知识也加深了印象。<br><br>perf工具看来是要找个时间好好看看了。<br>最早是在linux性能优化专栏看到用到，今天在一篇公众号上看别人用这个快速定位了线上有问题的死循环函数，今天老师又提到了用它看命中率。<br><br>工具用好了真的是方便，lsof之前也没用过，后来用了几次觉得非常好用，现在就经常用了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我来也同学每篇必有留言被编辑小姐姐选为精品，^_^<br>perf没有侵入性，使用简单，效果非常棒，值得你投入精力好好学一学！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-27 23:14:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/39/fa/a7edbc72.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>安排</span>
  </div>
  <div class="_2_QraFYR_0">内存伪共享问题，可以通过填充无效数据解决。<br>伪共享：假设cache line是64字节，我们在一个64字节的并且和cache line 对齐后的内存中放入两个4字节的整数A和B，然后线程a和b分别访问A和B，在内存层面的语义是这两个线程分别独享一块内存区域，操作时互不干扰，但是在缓存cache line层面他们是共享一个cache line的，是一个&quot;原子的数&quot;，这就是伪共享。<br>         缓存层面的伪共享的一致性由硬件保证，对程序员透明，也就是对这个&quot;原子数&quot;的操作不用显式加锁，但是伪共享降低程序效率。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 完全正确！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-30 08:11:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/99/36/80d3f12b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Oliver</span>
  </div>
  <div class="_2_QraFYR_0">对比开篇树图，numa架构貌似还没说，后续会提到吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你好Oliver，第1讲的初稿是有提的，但因为numa架构其实是讲访问非本地主存时，性能的降低问题，所以与CPU缓存关系不太直接，后面删除了。本来后面也不会再提到，不过中间有个10道面试题，我跟编辑商量下能不能放在那里提下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-28 13:20:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/cf/96/251c0cee.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>xindoo</span>
  </div>
  <div class="_2_QraFYR_0">https:&#47;&#47;zxs.io&#47;s&#47;o 我之前写过一篇博客详细介绍了cpu分支预测和性能差异，有兴趣可以参考下。 </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢分享！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-28 09:02:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/21/20/1299e137.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>秋天</span>
  </div>
  <div class="_2_QraFYR_0">因为cache-line的大小是64kb，由于单个变量站的大小占用的缓存大小达不到64kb。所以存在多个内存变量共享一个缓存行。导致多线程访问不同变量的时候 缓存行失效。解决方法是采用缓存行填充的方式，让每个线程的不同变量，占用不同的缓存行，提高命中率。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-03 12:48:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/be/af/93e14e9d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>扁舟</span>
  </div>
  <div class="_2_QraFYR_0">老师，您前面回答了评论中的 ”而桶大小为 50 字节，会导致最坏 2 次访问内存，而 70 字节最坏会有 3 次访问内存。”，但我仍然有几个不解<br>1.想问一下缓存行加载数据时 每一次加载数据都会加载 一个缓存行大小的数据吗？每次加载都会直到填满这个缓存行为止吗？如果所有缓存行满了的话，是LRU那样的替换吗？<br>2. 64字节恰好符合一个缓存行大小于是只要1个字节，54字节小于一个缓存行，他又是怎么加载的呢，70字节大于1个小于2个又是如何加载的呢。<br>希望老师能过解答一下这心中的疑问。谢谢老师</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-10 10:52:58</div>
  </div>
</div>
</div>
</li>
</ul>