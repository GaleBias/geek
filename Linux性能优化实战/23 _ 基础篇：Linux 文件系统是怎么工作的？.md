<audio title="23 _ 基础篇：Linux 文件系统是怎么工作的？" src="https://static001.geekbang.org/resource/audio/8b/90/8b7e1b50b634ca14c9c204fe16fd8a90.mp3" controls="controls"></audio> 
<p>你好，我是倪朋飞。</p><p>通过前面CPU和内存模块的学习，我相信，你已经掌握了CPU和内存的性能分析以及优化思路。从这一节开始，我们将进入下一个重要模块——文件系统和磁盘的I/O性能。</p><p>同CPU、内存一样，磁盘和文件系统的管理，也是操作系统最核心的功能。</p><ul>
<li>
<p>磁盘为系统提供了最基本的持久化存储。</p>
</li>
<li>
<p>文件系统则在磁盘的基础上，提供了一个用来管理文件的树状结构。</p>
</li>
</ul><p>那么，磁盘和文件系统是怎么工作的呢？又有哪些指标可以衡量它们的性能呢？</p><p>今天，我就带你先来看看，Linux文件系统的工作原理。磁盘的工作原理，我们下一节再来学习。</p><h2>索引节点和目录项</h2><p>文件系统，本身是对存储设备上的文件，进行组织管理的机制。组织方式不同，就会形成不同的文件系统。</p><p>你要记住最重要的一点，在Linux中一切皆文件。不仅普通的文件和目录，就连块设备、套接字、管道等，也都要通过统一的文件系统来管理。</p><p>为了方便管理，Linux文件系统为每个文件都分配两个数据结构，索引节点（index node）和目录项（directory entry）。它们主要用来记录文件的元信息和目录结构。</p><ul>
<li>
<p>索引节点，简称为inode，用来记录文件的元数据，比如inode编号、文件大小、访问权限、修改日期、数据的位置等。索引节点和文件一一对应，它跟文件内容一样，都会被持久化存储到磁盘中。所以记住，索引节点同样占用磁盘空间。</p>
</li>
<li>
<p>目录项，简称为dentry，用来记录文件的名字、索引节点指针以及与其他目录项的关联关系。多个关联的目录项，就构成了文件系统的目录结构。不过，不同于索引节点，目录项是由内核维护的一个内存数据结构，所以通常也被叫做目录项缓存。</p>
</li>
</ul><!-- [[[read_end]]] --><p>换句话说，索引节点是每个文件的唯一标志，而目录项维护的正是文件系统的树状结构。目录项和索引节点的关系是多对一，你可以简单理解为，一个文件可以有多个别名。</p><p>举个例子，通过硬链接为文件创建的别名，就会对应不同的目录项，不过这些目录项本质上还是链接同一个文件，所以，它们的索引节点相同。</p><p>索引节点和目录项纪录了文件的元数据，以及文件间的目录关系，那么具体来说，文件数据到底是怎么存储的呢？是不是直接写到磁盘中就好了呢？</p><p>实际上，磁盘读写的最小单位是扇区，然而扇区只有512B 大小，如果每次都读写这么小的单位，效率一定很低。所以，文件系统又把连续的扇区组成了逻辑块，然后每次都以逻辑块为最小单元，来管理数据。常见的逻辑块大小为4KB，也就是由连续的8个扇区组成。</p><p>为了帮助你理解目录项、索引节点以及文件数据的关系，我画了一张示意图。你可以对照着这张图，来回忆刚刚讲过的内容，把知识和细节串联起来。</p><p><img src="https://static001.geekbang.org/resource/image/32/47/328d942a38230a973f11bae67307be47.png?wh=836*507" alt=""></p><p>不过，这里有两点需要你注意。</p><p>第一，目录项本身就是一个内存缓存，而索引节点则是存储在磁盘中的数据。在前面的Buffer和Cache原理中，我曾经提到过，为了协调慢速磁盘与快速CPU的性能差异，文件内容会缓存到页缓存Cache中。</p><p>那么，你应该想到，这些索引节点自然也会缓存到内存中，加速文件的访问。</p><p>第二，磁盘在执行文件系统格式化时，会被分成三个存储区域，超级块、索引节点区和数据块区。其中，</p><ul>
<li>
<p>超级块，存储整个文件系统的状态。</p>
</li>
<li>
<p>索引节点区，用来存储索引节点。</p>
</li>
<li>
<p>数据块区，则用来存储文件数据。</p>
</li>
</ul><h2>虚拟文件系统</h2><p>目录项、索引节点、逻辑块以及超级块，构成了Linux文件系统的四大基本要素。不过，为了支持各种不同的文件系统，Linux内核在用户进程和文件系统的中间，又引入了一个抽象层，也就是虚拟文件系统VFS（Virtual File System）。</p><p>VFS 定义了一组所有文件系统都支持的数据结构和标准接口。这样，用户进程和内核中的其他子系统，只需要跟VFS 提供的统一接口进行交互就可以了，而不需要再关心底层各种文件系统的实现细节。</p><p>这里，我画了一张Linux文件系统的架构图，帮你更好地理解系统调用、VFS、缓存、文件系统以及块存储之间的关系。</p><p><img src="https://static001.geekbang.org/resource/image/72/12/728b7b39252a1e23a7a223cdf4aa1612.png?wh=530*590" alt=""></p><p>通过这张图，你可以看到，在VFS的下方，Linux支持各种各样的文件系统，如Ext4、XFS、NFS等等。按照存储位置的不同，这些文件系统可以分为三类。</p><ul>
<li>
<p>第一类是基于磁盘的文件系统，也就是把数据直接存储在计算机本地挂载的磁盘中。常见的Ext4、XFS、OverlayFS等，都是这类文件系统。</p>
</li>
<li>
<p>第二类是基于内存的文件系统，也就是我们常说的虚拟文件系统。这类文件系统，不需要任何磁盘分配存储空间，但会占用内存。我们经常用到的 /proc 文件系统，其实就是一种最常见的虚拟文件系统。此外，/sys 文件系统也属于这一类，主要向用户空间导出层次化的内核对象。</p>
</li>
<li>
<p>第三类是网络文件系统，也就是用来访问其他计算机数据的文件系统，比如NFS、SMB、iSCSI等。</p>
</li>
</ul><p>这些文件系统，要先挂载到 VFS 目录树中的某个子目录（称为挂载点），然后才能访问其中的文件。拿第一类，也就是基于磁盘的文件系统为例，在安装系统时，要先挂载一个根目录（/），在根目录下再把其他文件系统（比如其他的磁盘分区、/proc文件系统、/sys文件系统、NFS等）挂载进来。</p><h2>文件系统I/O</h2><p>把文件系统挂载到挂载点后，你就能通过挂载点，再去访问它管理的文件了。VFS 提供了一组标准的文件访问接口。这些接口以系统调用的方式，提供给应用程序使用。</p><p>就拿cat 命令来说，它首先调用 open() ，打开一个文件；然后调用 read() ，读取文件的内容；最后再调用 write()  ，把文件内容输出到控制台的标准输出中：</p><pre><code>int open(const char *pathname, int flags, mode_t mode); 
ssize_t read(int fd, void *buf, size_t count); 
ssize_t write(int fd, const void *buf, size_t count); 
</code></pre><p>文件读写方式的各种差异，导致 I/O的分类多种多样。最常见的有，缓冲与非缓冲I/O、直接与非直接I/O、阻塞与非阻塞I/O、同步与异步I/O等。 接下来，我们就详细看这四种分类。</p><p>第一种，根据是否利用标准库缓存，可以把文件I/O分为缓冲I/O与非缓冲I/O。</p><ul>
<li>
<p>缓冲I/O，是指利用标准库缓存来加速文件的访问，而标准库内部再通过系统调度访问文件。</p>
</li>
<li>
<p>非缓冲I/O，是指直接通过系统调用来访问文件，不再经过标准库缓存。</p>
</li>
</ul><p>注意，这里所说的“缓冲”，是指标准库内部实现的缓存。比方说，你可能见到过，很多程序遇到换行时才真正输出，而换行前的内容，其实就是被标准库暂时缓存了起来。</p><p>无论缓冲I/O还是非缓冲I/O，它们最终还是要经过系统调用来访问文件。而根据上一节内容，我们知道，系统调用后，还会通过页缓存，来减少磁盘的I/O操作。</p><p>第二，根据是否利用操作系统的页缓存，可以把文件I/O分为直接I/O与非直接I/O。</p><ul>
<li>
<p>直接I/O，是指跳过操作系统的页缓存，直接跟文件系统交互来访问文件。</p>
</li>
<li>
<p>非直接I/O正好相反，文件读写时，先要经过系统的页缓存，然后再由内核或额外的系统调用，真正写入磁盘。</p>
</li>
</ul><p>想要实现直接I/O，需要你在系统调用中，指定 O_DIRECT 标志。如果没有设置过，默认的是非直接I/O。</p><p>不过要注意，直接I/O、非直接I/O，本质上还是和文件系统交互。如果是在数据库等场景中，你还会看到，跳过文件系统读写磁盘的情况，也就是我们通常所说的裸I/O。</p><p>第三，根据应用程序是否阻塞自身运行，可以把文件I/O分为阻塞I/O和非阻塞I/O：</p><ul>
<li>
<p>所谓阻塞I/O，是指应用程序执行I/O操作后，如果没有获得响应，就会阻塞当前线程，自然就不能执行其他任务。</p>
</li>
<li>
<p>所谓非阻塞I/O，是指应用程序执行I/O操作后，不会阻塞当前的线程，可以继续执行其他的任务，随后再通过轮询或者事件通知的形式，获取调用的结果。</p>
</li>
</ul><p>比方说，访问管道或者网络套接字时，设置 O_NONBLOCK 标志，就表示用非阻塞方式访问；而如果不做任何设置，默认的就是阻塞访问。</p><p>第四，根据是否等待响应结果，可以把文件I/O分为同步和异步I/O：</p><ul>
<li>
<p>所谓同步I/O，是指应用程序执行I/O操作后，要一直等到整个I/O完成后，才能获得I/O响应。</p>
</li>
<li>
<p>所谓异步I/O，是指应用程序执行I/O操作后，不用等待完成和完成后的响应，而是继续执行就可以。等到这次 I/O完成后，响应会用事件通知的方式，告诉应用程序。</p>
</li>
</ul><p>举个例子，在操作文件时，如果你设置了 O_SYNC 或者 O_DSYNC 标志，就代表同步I/O。如果设置了O_DSYNC，就要等文件数据写入磁盘后，才能返回；而O_SYNC，则是在O_DSYNC基础上，要求文件元数据也要写入磁盘后，才能返回。</p><p>再比如，在访问管道或者网络套接字时，设置了O_ASYNC选项后，相应的I/O就是异步I/O。这样，内核会再通过SIGIO或者SIGPOLL，来通知进程文件是否可读写。</p><p>你可能发现了，这里的好多概念也经常出现在网络编程中。比如非阻塞I/O，通常会跟select/poll配合，用在网络套接字的I/O中。</p><p>你也应该可以理解，“Linux 一切皆文件”的深刻含义。无论是普通文件和块设备、还是网络套接字和管道等，它们都通过统一的VFS 接口来访问。</p><h2>性能观测</h2><p>学了这么多文件系统的原理，你估计也是迫不及待想上手，观察一下文件系统的性能情况了。</p><p>接下来，打开一个终端，SSH登录到服务器上，然后跟我一起来探索，如何观测文件系统的性能。</p><h3>容量</h3><p>对文件系统来说，最常见的一个问题就是空间不足。当然，你可能本身就知道，用 df 命令，就能查看文件系统的磁盘空间使用情况。比如：</p><pre><code>$ df /dev/sda1 
Filesystem     1K-blocks    Used Available Use% Mounted on 
/dev/sda1       30308240 3167020  27124836  11% / 
</code></pre><p>你可以看到，我的根文件系统只使用了11%的空间。这里还要注意，总空间用1K-blocks的数量来表示，你可以给df加上-h选项，以获得更好的可读性：</p><pre><code>$ df -h /dev/sda1 
Filesystem      Size  Used Avail Use% Mounted on 
/dev/sda1        29G  3.1G   26G  11% / 
</code></pre><p>不过有时候，明明你碰到了空间不足的问题，可是用df查看磁盘空间后，却发现剩余空间还有很多。这是怎么回事呢？</p><p>不知道你还记不记得，刚才我强调的一个细节。除了文件数据，索引节点也占用磁盘空间。你可以给df命令加上 -i 参数，查看索引节点的使用情况，如下所示：</p><pre><code>$ df -i /dev/sda1 
Filesystem      Inodes  IUsed   IFree IUse% Mounted on 
/dev/sda1      3870720 157460 3713260    5% / 
</code></pre><p>索引节点的容量，（也就是Inode个数）是在格式化磁盘时设定好的，一般由格式化工具自动生成。当你发现索引节点空间不足，但磁盘空间充足时，很可能就是过多小文件导致的。</p><p>所以，一般来说，删除这些小文件，或者把它们移动到索引节点充足的其他磁盘中，就可以解决这个问题。</p><h3>缓存</h3><p>在前面Cache案例中，我已经介绍过，可以用 free 或 vmstat，来观察页缓存的大小。复习一下，free输出的Cache，是页缓存和可回收Slab缓存的和，你可以从 /proc/meminfo ，直接得到它们的大小：</p><pre><code>$ cat /proc/meminfo | grep -E &quot;SReclaimable|Cached&quot; 
Cached:           748316 kB 
SwapCached:            0 kB 
SReclaimable:     179508 kB 
</code></pre><p>话说回来，文件系统中的目录项和索引节点缓存，又该如何观察呢？</p><p>实际上，内核使用Slab机制，管理目录项和索引节点的缓存。/proc/meminfo只给出了Slab的整体大小，具体到每一种Slab缓存，还要查看/proc/slabinfo这个文件。</p><p>比如，运行下面的命令，你就可以得到，所有目录项和各种文件系统索引节点的缓存情况：</p><pre><code>$ cat /proc/slabinfo | grep -E '^#|dentry|inode' 
# name            &lt;active_objs&gt; &lt;num_objs&gt; &lt;objsize&gt; &lt;objperslab&gt; &lt;pagesperslab&gt; : tunables &lt;limit&gt; &lt;batchcount&gt; &lt;sharedfactor&gt; : slabdata &lt;active_slabs&gt; &lt;num_slabs&gt; &lt;sharedavail&gt; 
xfs_inode              0      0    960   17    4 : tunables    0    0    0 : slabdata      0      0      0 
... 
ext4_inode_cache   32104  34590   1088   15    4 : tunables    0    0    0 : slabdata   2306   2306      0hugetlbfs_inode_cache     13     13    624   13    2 : tunables    0    0    0 : slabdata      1      1      0 
sock_inode_cache    1190   1242    704   23    4 : tunables    0    0    0 : slabdata     54     54      0 
shmem_inode_cache   1622   2139    712   23    4 : tunables    0    0    0 : slabdata     93     93      0 
proc_inode_cache    3560   4080    680   12    2 : tunables    0    0    0 : slabdata    340    340      0 
inode_cache        25172  25818    608   13    2 : tunables    0    0    0 : slabdata   1986   1986      0 
dentry             76050 121296    192   21    1 : tunables    0    0    0 : slabdata   5776   5776      0 
</code></pre><p>这个界面中，dentry行表示目录项缓存，inode_cache行，表示VFS索引节点缓存，其余的则是各种文件系统的索引节点缓存。</p><p>/proc/slabinfo 的列比较多，具体含义你可以查询  man slabinfo。在实际性能分析中，我们更常使用 slabtop  ，来找到占用内存最多的缓存类型。</p><p>比如，下面就是我运行slabtop得到的结果：</p><pre><code># 按下c按照缓存大小排序，按下a按照活跃对象数排序 
$ slabtop 
Active / Total Objects (% used)    : 277970 / 358914 (77.4%) 
Active / Total Slabs (% used)      : 12414 / 12414 (100.0%) 
Active / Total Caches (% used)     : 83 / 135 (61.5%) 
Active / Total Size (% used)       : 57816.88K / 73307.70K (78.9%) 
Minimum / Average / Maximum Object : 0.01K / 0.20K / 22.88K 

  OBJS ACTIVE  USE OBJ SIZE  SLABS OBJ/SLAB CACHE SIZE NAME 
69804  23094   0%    0.19K   3324       21     13296K dentry 
16380  15854   0%    0.59K   1260       13     10080K inode_cache 
58260  55397   0%    0.13K   1942       30      7768K kernfs_node_cache 
   485    413   0%    5.69K     97        5      3104K task_struct 
  1472   1397   0%    2.00K     92       16      2944K kmalloc-2048 
</code></pre><p>从这个结果你可以看到，在我的系统中，目录项和索引节点占用了最多的Slab缓存。不过它们占用的内存其实并不大，加起来也只有23MB左右。</p><h2>小结</h2><p>今天，我带你梳理了Linux文件系统的工作原理。</p><p>文件系统，是对存储设备上的文件，进行组织管理的一种机制。为了支持各类不同的文件系统，Linux在各种文件系统实现上，抽象了一层虚拟文件系统（VFS）。</p><p>VFS 定义了一组所有文件系统都支持的数据结构和标准接口。这样，用户进程和内核中的其他子系统，就只需要跟 VFS 提供的统一接口进行交互。</p><p>为了降低慢速磁盘对性能的影响，文件系统又通过页缓存、目录项缓存以及索引节点缓存，缓和磁盘延迟对应用程序的影响。</p><p>在性能观测方面，今天主要讲了容量和缓存的指标。下一节，我们将会学习Linux磁盘 I/O的工作原理，并掌握磁盘I/O的性能观测方法。</p><h2>思考</h2><p>最后，给你留一个思考题。在实际工作中，我们经常会根据文件名字，查找它所在路径，比如：</p><pre><code>$ find / -name file-name
</code></pre><p>今天的问题就是，这个命令，会不会导致系统的缓存升高呢？如果有影响，又会导致哪种类型的缓存升高呢？你可以结合今天内容，自己先去操作和分析，看看观察到的结果跟你分析的是否一样。</p><p>欢迎在留言区和我讨论，也欢迎把这篇文章分享给你的同事、朋友。我们一起在实战中演练，在交流中进步。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/a3/25/5da16c25.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>coyang</span>
  </div>
  <div class="_2_QraFYR_0">课后题：<br>这个命令，会不会导致系统的缓存升高呢？<br>--&gt; 会的<br>如果有影响，又会导致哪种类型的缓存升高呢？<br>--&gt; &#47;xfs_inode&#47; proc_inode_cache&#47;dentry&#47;inode_cache<br><br>实验步骤：<br>1. 清空缓存：echo 3 &gt; &#47;proc&#47;sys&#47;vm&#47;drop_caches ; sync<br>2. 执行find ： find &#47; -name test<br>3. 发现更新top 4 项是：<br>  OBJS ACTIVE  USE OBJ SIZE  SLABS OBJ&#47;SLAB CACHE SIZE NAME<br> 37400  37400 100%    0.94K   2200       17     35200K xfs_inode<br> 36588  36113  98%    0.64K   3049       12     24392K proc_inode_cache<br>104979 104979 100%    0.19K   4999       21     19996K dentry<br> 18057  18057 100%    0.58K   1389       13     11112K inode_cache<br><br>find &#47; -name 这个命令是全盘扫描（既包括内存文件系统又包含本地的xfs【我的环境没有mount 网络文件系统】），所以 inode cache &amp; dentry &amp; proc inode cache 会升高。<br><br>另外，执行过了一次后再次执行find 就机会没有变化了，执行速度也快了很多，也就是下次的find大部分是依赖cache的结果。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-11 17:30:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/13/52/db1b01fc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>白华</span>
  </div>
  <div class="_2_QraFYR_0">课后题：我找了一个目录下的文件，用的这个命令find &#47; -type f -name copyright   然后slabtop观察，发现dentry的SLABS和SIZE有了明显的提高，所以引起了目录项缓存的升高。在开始的时候dentry有一定的大小，我认为是缓存了&#47;目录下系统基本的目录，但是系统后面下载、创建的内容是没有缓存的，使用查找命令会把这些都查找到然后缓存起来，所以使用find查找大量内容时候会造成性能下降。<br><br>前面看老男孩视频时候了解了inode和block。inode存储这些数据属性信息的，包含不限于文件大小、文件类型、文件权限、拥有者、硬链接数、所属组、修改时间，还包含指向文件实体的指针功能（inode节点---block的对应关系），但是inode惟独不包含文件名。文件名不在inode里，在上级目录的block里；Block来存储实际数据用的，例如照片、视频等普通文件数据。<br><br>今天看到了dentry，定义是用来记录文件的名字、索引节点指针以及与其他目录项的关联关系。但是和老男孩老师讲的有所区别，希望老师帮我解惑 </div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-11 11:14:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/4a/2c/f8451d77.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>石维康</span>
  </div>
  <div class="_2_QraFYR_0">阻塞 I&#47;O 和非阻塞 I&#47;O的概念和同步和异步 I&#47;O的区别是什么?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个在答疑里统一回复吧</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-11 08:22:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/99/ff/046495bb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小成</span>
  </div>
  <div class="_2_QraFYR_0">请问老师，除了目录项以外还有哪些地方保存有文件名，下一节讲到目录项是一个内存缓存，那么不会保存文件名到磁盘上面？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 目录项是一个缓存，不是持久化存储。目录也是一个文件，这个特殊文件保存了该目录的所有文件名与inode的对应关系</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-01 13:11:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/f8/19/05a2695f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>伟忠</span>
  </div>
  <div class="_2_QraFYR_0">机器上 df 查看占用了 200G，但 du 查看发现只有 90G，看网上的办法用 lsof | grep delete 查看，但没有找到，请问老师，这个可能是什么原因呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可能是文件本身已经删除了，但其描述符还被进程占用着，可以查找无效的文件描述符看看</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-12 16:11:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/ccpIPibkaTQfYbO5DGiaWpL86YSHAZfVO55WtJhjV0hb7AuyIMzLyRdLnQZ6tjB0Wars4ib7YX3fhmPh9R81MVKtA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>肘子哥</span>
  </div>
  <div class="_2_QraFYR_0">有个疑惑，如果目录项存在内存中是不是意味着内存故障后，目录就无法访问了呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不会的，还可以从磁盘的持久化数据中重建</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-23 12:42:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/rQOn22bNV0kHpoPWRLRicjQCOkiaYmcVABiaIJxIDWIibSdqWXYTxjcdjiadibIxFsGVp5UE4DBd6Nx2DxjhAdlMIZeQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ThinkerWalker</span>
  </div>
  <div class="_2_QraFYR_0">关于白华提到的:   &quot;前面看老男孩视频时候了解了inode和block。inode存储这些数据属性信息的，包含不限于文件大小、文件类型、文件权限、拥有者、硬链接数、所属组、修改时间，还包含指向文件实体的指针功能（inode节点---block的对应关系），但是inode惟独不包含文件名。文件名不在inode里，在上级目录的block里；Block来存储实际数据用的，例如照片、视频等普通文件数据。&quot;<br><br>我想说,目录项本质上是缓存,缓存是为了加速文件查找和访问的,所以说和老师这里所说的dentry不冲突,没有目录项的时候查找一个文件需要从&#47;一级一级查找. 是这样的吧?倪老师?</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-16 16:24:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/94/e1/f3f0015d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Net Scotte</span>
  </div>
  <div class="_2_QraFYR_0">这里的slab不是很容易理解，我查了一下，slab是Linux的内存管理机制，用于解决小对象（具有相同数据结构和大小的内存单元）的内存管理问题，slabinfo中的每一项都是一种cache（即前文说的目录项缓存、索引节点缓存等），一个cache包括多个slab，slab又包含多个objects，已经分配的称为active object</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-31 21:17:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/f7/9b/1b1e288a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>董文荣</span>
  </div>
  <div class="_2_QraFYR_0">课后题：<br>Q:$ find &#47; -name file-name<br>这个命令，会不会导致系统的缓存升高呢？如果有影响，又会导致哪种类型的缓存升高呢？<br>A:分析<br>1)、&quot;&#47;&quot;代表文件系统的根目录，目录项已经缓存在cached。(通过下面的测试，怀疑应该只是部分目录项的内容缓存在cache中，待验证)<br>2)、因为会匹配值“file-name“，会将索引节点读入缓存进行匹配。<br>因此会导致cached增长。以下是三组测试对比，给出了执行find命令前后，cached变化的对比。<br>命令之前前后，slabtop的执行前后对比:<br>Active &#47; Total Objects (% used)    : 184412 &#47; 240169 (76.8%)<br> Active &#47; Total Size (% used)       : 42926.19K &#47; 59199.82K (72.5%)<br><br>  OBJS ACTIVE  USE OBJ SIZE  SLABS OBJ&#47;SLAB CACHE SIZE NAME                   <br> 11088   2313  20%    0.57K    198	 56	 6336K radix_tree_node<br> 10450   9515  91%    0.58K    190	 55	 6080K inode_cache<br> 27510  12695  46%    0.19K    655	 42	 5240K dentry<br>  4710   1003  21%    1.06K    157	 30	 5024K xfs_inode<br><br><br> Active &#47; Total Objects (% used)    : 1795399 &#47; 1809652 (99.2%)<br> Active &#47; Total Size (% used)       : 1004316.02K &#47; 1007573.47K (99.7%)<br><br>  OBJS ACTIVE  USE OBJ SIZE  SLABS OBJ&#47;SLAB CACHE SIZE NAME                   <br>708420 708420 100%    1.06K  23614	 30    755648K xfs_inode<br>787878 787878 100%    0.19K  18759	 42    150072K dentry<br><br>free命令在find命令执行前后结果对比:<br>[root@localhost ~]# free -m<br>              total        used        free      shared  buff&#47;cache   available<br>Mem:           1824         200        1534          15          89        1500<br>Swap:          2047         196        1851<br>[root@localhost ~]# free -m<br>              total        used        free      shared  buff&#47;cache   available<br>Mem:           1824         480         105          15        1238        1161<br>Swap:          2047         196        1851<br><br>vmstat在find命令执行前后对比：<br>[root@localhost &#47;]# vmstat  2 <br>procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu----<br> r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st<br> 2  0 201208 1570636    136  92168    0    0     5     5   16   19 27  3 70  0  0<br> 0  0 201208 1511788    136 149564    0    0  1702     0  491  599  2  6 90  1  0<br> 0  0 201208 1509428    136 149716    0    0  1106     0  478  801  0  2 98  0  0<br>注:发表长度限制，省略部分测试显示<br><br><br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 原理分析加实践👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-21 16:03:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/13/51/99c88021.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>成为祝福</span>
  </div>
  <div class="_2_QraFYR_0">老师好，请问在slabtop中的inode_cache和ext4_inode_cache有什么区别呢？如果每个文件系统都有inode_cache，整个vfs的有效命名空间都映射到了对应的文件系统，vfs为什么还需要inode_cache呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 一个是VFS 虚拟文件系统的缓存，另一个则是具体的文件系统实现的缓存。<br><br>一个更好理解的例子是：操作系统有文件的缓存，而应用程序还会自己来分配内存缓存数据</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-16 02:35:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/2d/24/28acca15.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>DJH</span>
  </div>
  <div class="_2_QraFYR_0">请教三个问题。<br><br>1. 目录项是维护在内核中的一个内存数据结构，包括文件名。<br>我的问题是：文件名不是也应该存储在磁盘上么？不可能仅仅存在于内存吧？<br><br>2. 缓冲 I&#47;O，是指利用标准库缓存来减速文件的访问...<br>我的问题时：减速文件访问的原因是什么？<br><br>3. 阻塞&#47;非阻塞与同步&#47;非同步的区别是什么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. 目录项是表示目录之间的树状关系，而文件名则会存储到数据部分。<br>2. 不好意思，是个笔误，当然是加速。谢谢指出<br>3. 这个在文章中有简单的介绍，回来在答疑篇中再展开一些</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-11 21:13:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/dc/49/43ae1627.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Ansyear</span>
  </div>
  <div class="_2_QraFYR_0">课后答案：<br>文件名以及文件之间的目录关系，都放在目录项缓存中。而这是一个基于内存的数据结构，会根据需要动态构建。所以，<br>1.查找文件时，Linux 就会动态构建不在缓存中的目录项结构，导致 dentry 缓存升高。<br>2.find还需要找到文件的位置信息，而这些信息存放在inode的元数据中，所以会将索引节点也读入缓存进行匹配，因此会导致inode_cached增长。<br>而dentry缓存和inode_cached又都是包含在cached的一部分，所以通过free命令查看cached会升高</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-05 10:16:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/b8/36/542c96bf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Mr.Strive.Z.H.L</span>
  </div>
  <div class="_2_QraFYR_0">老师您好：<br>关于目录项有一个疑惑：<br><br>通过目录项找到inode节点，从而访问具体的文件内容。其中inode和文件数据块都会被持久化，而目录项竟然不会被持久化，只是放在内存中进行缓存。<br><br>那么是否在每次开机时，内核都会自动构建文件系统完整的目录项，然后进行缓存？？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 按需构建很少一部分目录项就可以了，不需要所有的目录项</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-27 15:37:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_9815f1</span>
  </div>
  <div class="_2_QraFYR_0">你要记住最重要的一点，在 Linux 中一切皆文件。不仅普通的文件和目录，就连块设备、套接字、管道等，也都要通过统一的文件系统来管理。    老师，上次听你 讲块设备和 文件系统的区别： 说块设备读写是绕过文件系统的。 现在是 块设备也通过统一的文件系统来管理。 这有矛盾吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这儿说的是VFS</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-17 18:25:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/4e/0f/c43745e7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>hola</span>
  </div>
  <div class="_2_QraFYR_0">“索引节点和文件一一对应，它跟文件内容一样，都会被持久化存储到磁盘”<br>是一对一关系吗，那么看我的服务器<br>$ df -i &#47;<br>文件系统                                 Inode 已用(I) 可用(I) 已用(I)% 挂载点<br>&#47;dev&#47;mapper&#47;VolGroup-lv_root 655360   98862  556498      16% &#47;<br>意思是这个下面理论最多存放655360个文件  对吗<br><br><br><br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-23 14:15:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/b0/57/a84d633e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>圣诞使者</span>
  </div>
  <div class="_2_QraFYR_0">课后题，我感觉不会，应该只有用了目录的执行(x)权限内核才会缓存dentry，find只是用了目录的读(r)权限。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-11 08:29:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://wx.qlogo.cn/mmopen/vi_32/PiajxSqBRaEKQMM4m7NHuicr55aRiblTSEWIYe0QqbpyHweaoAbG7j2v7UUElqqeP3Ihrm3UfDPDRb1Hv8LvPwXqA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ninuxer</span>
  </div>
  <div class="_2_QraFYR_0">打卡day24<br>阻塞非阻塞，同步异步再次mark下：<br>根据应用程序是否阻塞自身线程的运行，可以把文件 I&#47;O 分为阻塞 I&#47;O 和非阻塞 I&#47;O；<br>根据是否等待响应结果，可以把文件 I&#47;O 分为同步和异步 I&#47;O</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-11 08:15:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/e3/c2/7406eaf1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>xzyeah</span>
  </div>
  <div class="_2_QraFYR_0">老师，我的理解，不会引起内存升高，因为文件名存在于目录项，目录项本身就存在于内存缓存。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-11 07:09:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_9815f1</span>
  </div>
  <div class="_2_QraFYR_0">你要记住最重要的一点，在 Linux 中一切皆文件。不仅普通的文件和目录，就连块设备、套接字、管道等，也都要通过统一的文件系统来管理。   上一章不是说过 块设备绕过文件系统， 现在又说统一由文件系统管理 </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这儿说的“虚拟文件系统（VFS）”，而不是具体的某一个“文件系统”（比如XFS等）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-20 10:44:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/1a/3a/9001e627.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>流水</span>
  </div>
  <div class="_2_QraFYR_0">评论区很多同学在问目录项问题，主要是因为把目录项和目录文件搞混了吧，目录项是内核的缓存对象，目录文件是特殊的文件，也有自己的inode，文件内容里记录了其他文件名及inode的映射关系</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-27 08:29:57</div>
  </div>
</div>
</div>
</li>
</ul>