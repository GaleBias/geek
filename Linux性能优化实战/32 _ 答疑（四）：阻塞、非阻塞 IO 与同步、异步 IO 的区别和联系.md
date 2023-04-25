<audio title="32 _ 答疑（四）：阻塞、非阻塞 IO 与同步、异步 IO 的区别和联系" src="https://static001.geekbang.org/resource/audio/29/aa/294a44f3338344bb8d6bbd41a8d78faa.mp3" controls="controls"></audio> 
<p>你好，我是倪朋飞。</p><p>专栏更新至今，四大基础模块的第三个模块——文件系统和磁盘 I/O 篇，我们就已经学完了。很开心你还没有掉队，仍然在积极学习思考和实践操作，并且热情地留言与讨论。</p><p>今天是性能优化的第四期。照例，我从 I/O 模块的留言中摘出了一些典型问题，作为今天的答疑内容，集中回复。同样的，为了便于你学习理解，它们并不是严格按照文章顺序排列的。</p><p>每个问题，我都附上了留言区提问的截屏。如果你需要回顾内容原文，可以扫描每个问题右下方的二维码查看。</p><h2>问题1：阻塞、非阻塞 I/O 与同步、异步 I/O 的区别和联系</h2><p><img src="https://static001.geekbang.org/resource/image/1c/b0/1c3237118d1c55792ac0d9cc23f14bb0.png?wh=600*700" alt=""></p><p>在<a href="https://time.geekbang.org/column/article/76876">文件系统的工作原理</a>篇中，我曾经介绍了阻塞、非阻塞 I/O 以及同步、异步 I/O 的含义，这里我们再简单回顾一下。</p><p>首先我们来看阻塞和非阻塞 I/O。根据应用程序是否阻塞自身运行，可以把 I/O 分为阻塞 I/O 和非阻塞 I/O。</p><ul>
<li>
<p>所谓阻塞I/O，是指应用程序在执行I/O操作后，如果没有获得响应，就会阻塞当前线程，不能执行其他任务。</p>
</li>
<li>
<p>所谓非阻塞I/O，是指应用程序在执行I/O操作后，不会阻塞当前的线程，可以继续执行其他的任务。</p>
</li>
</ul><p>再来看同步 I/O 和异步 I/O。根据 I/O 响应的通知方式的不同，可以把文件 I/O 分为同步 I/O 和异步 I/O。</p><!-- [[[read_end]]] --><ul>
<li>
<p>所谓同步 I/O，是指收到 I/O 请求后，系统不会立刻响应应用程序；等到处理完成，系统才会通过系统调用的方式，告诉应用程序 I/O 结果。</p>
</li>
<li>
<p>所谓异步 I/O，是指收到 I/O 请求后，系统会先告诉应用程序  I/O 请求已经收到，随后再去异步处理；等处理完成后，系统再通过事件通知的方式，告诉应用程序结果。</p>
</li>
</ul><p>你可以看出，阻塞/非阻塞和同步/异步，其实就是两个不同角度的 I/O 划分方式。它们描述的对象也不同，阻塞/非阻塞针对的是 I/O 调用者（即应用程序），而同步/异步针对的是 I/O 执行者（即系统）。</p><p>我举个例子来进一步解释下。比如在 Linux I/O 调用中，</p><ul>
<li>
<p>系统调用 read 是同步读，所以，在没有得到磁盘数据前，read 不会响应应用程序。</p>
</li>
<li>
<p>而 aio_read 是异步读，系统收到 AIO 读请求后不等处理就返回了，而具体的 read 结果，再通过回调异步通知应用程序。</p>
</li>
</ul><p>再如，在网络套接字的接口中，</p><ul>
<li>
<p>使用 send() 直接向套接字发送数据时，如果套接字没有设置 O_NONBLOCK 标识，那么 send() 操作就会一直阻塞，当前线程也没法去做其他事情。</p>
</li>
<li>
<p>当然，如果你用了 epoll，系统会告诉你这个套接字的状态，那就可以用非阻塞的方式使用。当这个套接字不可写的时候，你可以去做其他事情，比如读写其他套接字。</p>
</li>
</ul><h2>问题2：“文件系统”课后思考</h2><p><img src="https://static001.geekbang.org/resource/image/40/a6/40c924ea4b11e12d6d34181a00f292a6.jpg?wh=750*1170" alt=""></p><p>在<a href="https://time.geekbang.org/column/article/76876">文件系统原理</a><a href="https://time.geekbang.org/column/article/76876">文章</a>的最后，我给你留了一道思考题，那就是执行 find 命令时，会不会导致系统的缓存升高呢？如果会导致，升高的又是哪种类型的缓存呢？</p><p>关于这个问题，白华和 coyang 的答案已经很准确了。通过学习Linux 文件系统的原理，我们知道，文件名以及文件之间的目录关系，都放在目录项缓存中。而这是一个基于内存的数据结构，会根据需要动态构建。所以，查找文件时，Linux 就会动态构建不在缓存中的目录项结构，导致 dentry 缓存升高。</p><p><img src="https://static001.geekbang.org/resource/image/48/c5/488110263a9c7ff801a3e04c010f0bc5.png?wh=600*1202" alt=""><img src="https://static001.geekbang.org/resource/image/57/58/57e4cf5a42a91392ebebf106f992a858.png?wh=600*1438" alt=""></p><p>事实上，除了目录项缓存增加，Buffer 的使用也会增加。如果你用 vmstat 观察一下，会发现 Buffer 和 Cache 都在增长：</p><pre><code>$ vmstat 1
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 0  1      0 7563744   6024 225944    0    0  3736     0  574 3249  3  5 89  3  0
 1  0      0 7542792  14736 236856    0    0  8708     0 13494 32335  8 19 66  7  0
 0  1      0 7494452  27280 272284    0    0 12544     0 4550 17084  5 15 68 13  0
 0  1      0 7475084  42380 276320    0    0 15096     0 2541 14253  2  6 78 13  0
 0  1      0 7455728  57600 280436    0    0 15220     0 2025 14518  2  6 70 22  0
</code></pre><p>这里，Buffer 的增长是因为，构建目录项缓存所需的元数据（比如文件名称、索引节点等），需要从文件系统中读取。</p><h2>问题3：“磁盘 I/O 延迟”课后思考</h2><p>在<a href="https://time.geekbang.org/column/article/78409">磁盘 I/O 延迟案例</a>的最后，我给你留了一道思考题。</p><p>我们通过 iostat ，确认磁盘 I/O 已经出现了性能瓶颈，还用 pidstat 找出了大量磁盘 I/O 的进程。但是，随后使用 strace 跟踪这个进程，却找不到任何 write 系统调用。这是为什么呢？</p><p><img src="https://static001.geekbang.org/resource/image/64/09/6408b3aa2aa9a98a930d1a5b2e2fef09.jpg?wh=750*1551" alt=""></p><p>很多同学的留言都准确回答了这个问题。比如，划时代和 jeff 的留言都指出，在这个场景中，我们需要加 -f 选项，以便跟踪多进程和多线程的系统调用情况。</p><p><img src="https://static001.geekbang.org/resource/image/e4/55/e4e9a070022f7b49cb8d5554b9a60055.png?wh=600*700" alt=""><img src="https://static001.geekbang.org/resource/image/71/05/71a6df4144ce59d9e1a01c26453acf05.png?wh=600*700" alt=""></p><p>你看，仅仅是不恰当的选项，都可能会导致性能工具“犯错”，呈现这种看起来不合逻辑的结果。非常高兴看到，这么多同学已经掌握了性能工具使用的核心思路——弄清楚工具本身的原理和问题。</p><h2>问题4：“MySQL 案例”课后思考</h2><p>在 <a href="https://time.geekbang.org/column/article/78633">MySQL 案例</a>的最后，我给你留了一个思考题。</p><p>为什么 DataService 应用停止后，即使仍没有索引，MySQL 的查询速度还是快了很多，并且磁盘 I/O 瓶颈也消失了呢？</p><p><img src="https://static001.geekbang.org/resource/image/92/78/924fbc974313b1e0fe6b8d14e7a44178.png?wh=600*700" alt=""></p><p>ninuxer 的留言基本解释了这个问题，不过还不够完善。</p><p>事实上，当你看到 DataService 在修改 <em>/proc/sys/vm/drop_caches</em>  时，就应该想到前面学过的 Cache 的作用。</p><p>我们知道，案例应用访问的数据表，基于 MyISAM 引擎，而 MyISAM 的一个特点，就是只在内存中缓存索引，并不缓存数据。所以，在查询语句无法使用索引时，就需要数据表从数据库文件读入内存，然后再进行处理。</p><p>所以，如果你用 vmstat 工具，观察缓存和 I/O 的变化趋势，就会发现下面这样的结果：</p><pre><code>$ vmstat 1

procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st

# 备注： DataService正在运行
0  1      0 7293416    132 366704    0    0 32516    12   36  546  1  3 49 48  0
 0  1      0 7260772    132 399256    0    0 32640     0   37  463  1  1 49 48  0
 0  1      0 7228088    132 432088    0    0 32640     0   30  477  0  1 49 49  0
 0  0      0 7306560    132 353084    0    0 20572     4   90  574  1  4 69 27  0
 0  2      0 7282300    132 368536    0    0 15468     0   32  304  0  0 79 20  0

# 备注：DataService从这里开始停止
 0  0      0 7241852   1360 424164    0    0   864   320  133 1266  1  1 94  5  0
 0  1      0 7228956   1368 437400    0    0 13328     0   45  366  0  0 83 17  0
 0  1      0 7196320   1368 470148    0    0 32640     0   33  413  1  1 50 49  0
...
 0  0      0 6747540   1368 918576    0    0 29056     0   42  568  0  0 56 44  0
 0  0      0 6747540   1368 918576    0    0     0     0   40  141  1  0 100  0  0
</code></pre><p>在 DataService 停止前，cache 会连续增长三次后再降回去，这正是因为 DataService 每隔3秒清理一次页缓存。而 DataService 停止后，cache 就会不停地增长，直到增长为 918576 后，就不再变了。</p><p>这时，磁盘的读（bi）降低到 0，同时，iowait（wa）也降低到 0，这说明，此时的所有数据都已经在系统的缓存中了。我们知道，缓存是内存的一部分，它的访问速度比磁盘快得多，这也就能解释，为什么 MySQL 的查询速度变快了很多。</p><p>从这个案例，你会发现，MySQL 的 MyISAM 引擎，本身并不缓存数据，而要依赖系统缓存来加速磁盘 I/O 的访问。一旦系统中还有其他应用同时运行，MyISAM 引擎就很难充分利用系统缓存。因为系统缓存可能被其他应用程序占用，甚至直接被清理掉。</p><p>所以，一般来说，我并不建议，把应用程序的性能优化完全建立在系统缓存上。还是那句话，最好能在应用程序的内部分配内存，构建完全自主控制的缓存，比如 MySQL 的 InnoDB 引擎，就同时缓存了索引和数据；或者，可以使用第三方的缓存应用，比如 Memcached、Redis 等。</p><p>今天主要回答这些问题，同时也欢迎你继续在留言区写下疑问和感想，我会持续不断地解答。希望借助每一次的答疑，可以和你一起，把文章知识内化为你的能力，我们不仅在实战中演练，也要在交流中进步。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/e7/2a/ccac99dd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>eagle</span>
  </div>
  <div class="_2_QraFYR_0">我根据我们自己实际应用中遇到的情况，试着回复一下两个问题：<br>安小依 的问题，df -h 显示占用100%，而关闭应用程序后，再次df -h是85%，这一般是因为该应用程序还有指向已删除文件的文件指针没有关闭，典型的比如日志文件，虽然在操作系统中用rm命令删除了，在相应的目录中已经没有该文件了，但如果应用中还有对应的文件指针没有关闭，则实际硬盘空间还不会释放，而应用程序被关闭时，实际空间才会释放。问题中更像是有些apk文件或处理后文件的文件指针没有释放。这种情况也可以通过 lsof | grep deleted 来找到这些文件。<br>lvy的out of memory的问题，可以先用free或top看一下可用内存是否确实没有了，如果确实是没有内存了，那再去研究内存的问题；还有一种常见情况，内存是充足的，文件描述符的个数或进程数达到上限了，那就得调整 ulimit，可以通过 ulimit -a (注意要用php的用户）来查看，关注open files和max user processes，这两个默认很小，1k和4k，建议调整到加两个0.</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢😊</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-26 18:08:16</div>
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
  <div class="_2_QraFYR_0">打卡day33<br>感恩作者带来的分享，提前祝新年快乐！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 新年快乐！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-01 08:04:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/93/43/0e84492d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Maxwell</span>
  </div>
  <div class="_2_QraFYR_0">Windows和linux有很大区别吧？如果想深入了解windows，有什么可以推荐的书吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Windows书籍最推荐的是《Windows Internals 7th edition》</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-14 09:40:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Ivy</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，我最近在生产环境遇到一个问题，centos7频繁报错tcp out of memory ，访问页面时css文件响应头200，但是响应正文为空，我猜测就是因为tcp问题，有时候又能正常返回，每次重启php fpm就能解决问题，cat &#47;proc&#47;net&#47;sockstat 的时候tcp 行mem值在fpm重启前后差距很大，同时tw状态的连接也很多，alloc也很大，我该怎么去找原因？能看到每个tw状态的连接占用多少tcp 内存吗？或者怎么查询php fpm为何没有释放tcp内存？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以考虑调整 tcp_max_tw_buckets、ip_conntrack_max、ip_conntrack 这些内核选项</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-19 09:26:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/e7/88/c8b4ad9c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>没有昵称</span>
  </div>
  <div class="_2_QraFYR_0">我觉得同步异步io的提问者并不是想要理解字面意思，而是想要了解内部的工作模式，之前看过一篇文章讲解的比较好，把io分成了几个阶段，不同类型的io每个阶段都干了什么，回头再找下</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 欢迎分享一下链接</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-01 10:03:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/fb/72/3fe64bc5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>LA</span>
  </div>
  <div class="_2_QraFYR_0">老师，看了您的文章，有个问题一直在困扰这我。文章所说进程不可中断状态有可能是因为等待io响应，那这里的等待io响应包括等待从套接字读取数据么？如果是包括的话对于阻塞io来讲岂不是只要有阻塞进程就一直处在不可中断状态，从而无法被kill信号杀掉？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不包括套接字</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-01 00:44:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/15/4d/bbfda6b7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>笃定</span>
  </div>
  <div class="_2_QraFYR_0">老师，有一个疑问，如果我的应用程序使用的是异步非阻塞IO调用方式，那么我发起IO请求获取大量的数据，因为是异步非阻塞的；可以不马上得到数据而继续执行其他任务，这样的话，是不是我这个系统上就不会出现CPUIOWAIT升高的情况呢（系统上就只跑这一个程序的情况下）</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-07 16:25:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/41/59/78042964.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Cryhard</span>
  </div>
  <div class="_2_QraFYR_0">复习一下，仍然会有新的收获！谢谢！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-19 08:12:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Stephen</span>
  </div>
  <div class="_2_QraFYR_0">《UNIX网络编程》提到的5种IO模型中，除了异步IO模型没有阻塞操作外，其他四种IO模型（阻塞IO、非阻塞IO、IO多路复用、信号驱动IO）都有阻塞操作。是不是可以这么理解:<br>同步IO一定有阻塞可能有非阻塞，<br>异步IO一定是非阻塞；<br>有阻塞一定是同步IO，<br>有非阻塞可能是同步IO或异步IO<br>求大佬解答问题</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-03 18:47:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/70/23/972dcd30.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>allan</span>
  </div>
  <div class="_2_QraFYR_0">原文：DataService 停止后，bi iowait 都降到0，说明此时的所有数据都已经在系统的缓存中了。<br><br>这里所有数据指的是 数据库文件 中的数据 是吗？不包括索引。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，数据库文件</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-24 12:45:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/cf/5c/d4e19eb6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>安小依</span>
  </div>
  <div class="_2_QraFYR_0">老师，今天遇见了一个问题: 系统使用 df -h 显示磁盘占用100%了，而且应用程序(这是一个不停下载 apk 文件、解压缩并分析 apk文件的应用程序)在命令行也提示磁盘空间不足了。但是，关闭应用程序后，再次 df-h 统计，却发现这次磁盘占用是 85%，释放了 15%大约150G 的空间…能大概推测出来为什么关闭应用后，磁盘空间突然多了的原因吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 解压缩很可疑，有没有看看这些apk解压后的大小？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-01 00:49:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/9b/a8/6a391c66.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Leon📷</span>
  </div>
  <div class="_2_QraFYR_0">老师，我们的测试环境机器我从几个指标看只有系统盘每秒写的数据量比测试环境多，为什么比测试环境卡很多，进程也只是测试环境一倍而已，使用vmstat pidstat,top,发现只有线上机器进程数多一倍，io写入量是测试机器10倍，测试配置4核16G，线上32核，256G，磁盘随机读写都是79MB&#47;s左右<br>测试17时50分54秒     0         1      3.85     16.61      4.86  systemd<br>线上05:50:51 PM     0         1    151.52   1922.47    210.85  systemd<br>top<br>top - 17:57:54 up 24 days,  6:32,  3 users,  load average: 2.06, 2.07, 2.41<br>Tasks: 974 total,   2 running, 970 sleeping,   0 stopped,   2 zombie<br>%Cpu(s):  2.2 us,  4.1 sy,  0.1 ni, 65.6 id, 27.7 wa,  0.0 hi,  0.0 si,  0.3 st<br>KiB Mem : 16249556 total,  2730324 free,  8055928 used,  5463304 buff&#47;cache<br>KiB Swap:        0 total,        0 free,        0 used.  7338032 avail Mem<br>线上 top - 17:58:03 up 73 days,  8:41,  2 users,  load average: 4.84, 3.40, 2.94<br>Tasks: 2651 total,   1 running, 2650 sleeping,   0 stopped,   0 zombie<br>%Cpu(s):  1.4 us,  0.5 sy,  0.0 ni, 92.9 id,  5.1 wa,  0.0 hi,  0.1 si,  0.0 st<br>KiB Mem : 26385616+total, 88973160 free, 23977900 used, 15090508+buff&#47;cache<br>KiB Swap:        0 total,        0 free,        0 used. 23713659+avail Mem<br>难道IO就是线上机器卡的原因<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 除了CPU、内存、磁盘之外，网络也可能是个原因。另外，对 I&#47;O，还可以用 iostat 看一下其他指标是不是有什么线索</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-01 17:59:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/c2/e0/7188aa0a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>blackpiglet</span>
  </div>
  <div class="_2_QraFYR_0">老师，在工作中遇到了 Ubuntu 16.04 系统死机的问题，和性能优化并不直接相关，不过还是想问一下遇到这种问题该如何分析。我能想到的步骤是：<br>1. 看 &#47;var&#47;crash 下是否有 kernel panic 的记录；<br>2. 看 &#47;var&#47;log&#47;syslog 下是否有应用程序异常记录；<br>3. 看服务器上主要的应用程序日志，是否有异常；<br>4. 查看是否有 coredump 文件；<br>5. 查看 IPMI 日志，是否有硬件异常。<br><br>有的时候这一趟下来，还是没有什么收获，请问老师有没有其他需要注意的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 系统日志肯定是最有效的，一般都会留下一些线索。死机的时候，如果可以通过IPMI看到服务器的Console，那也有可能看到一些线索（比如内核中的错误或者内核栈等等）。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-01 11:19:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/9b/a8/6a391c66.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Leon📷</span>
  </div>
  <div class="_2_QraFYR_0">老师，读文件系统的内容不会引起buffer升高吧，读块设备会引起，我做了文章的实验发现<br> r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st<br> 1  0      0 1620788      0 431512    0    0   348    70  415  585  2  4 94  0  0<br> 0  0      0 1618488      0 431948    0    0   480     0 1605 2056  1  4 96  0  0<br> 0  0      0 1619524      0 431788    0    0    16     0 1157 1674  1  2 97  0  0<br> 2  0      0 1499696      0 548464    0    0 116905   281 5084 7062  3 14 82  1  0<br> 2  0      0 1495664      0 552444    0    0  4964   125 2996 4413  2  6 92  0  0<br> 2  0      0 1329960      0 646564    0    0 34028     0 8495 10589 21 24 53  1  0<br> 2  0      0 1152440      0 769524    0    0 142805   206 13584 16541 19 32 48  1  0<br> 3  0      0 1112028      0 783200    0    0 44753    86 14794 20490 19 23 57  0  0<br> 0  0      0 1050900      0 809624    0    0 36540     0 8927 13517  5 20 75  0  0<br> 0  0      0 1050892      0 809636    0    0     0     0 1277 1879  1  2 97  0  0<br> 0  0      0 1050632      0 809644    0    0     0     0 1344 1953  1  2 97  0  0<br>buffer并没有升高</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，读文件内容的时候不会的。文中指的是执行 find 命令查找文件的情景</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-01 09:00:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_77df57</span>
  </div>
  <div class="_2_QraFYR_0">同步异步，阻塞非阻塞是不是可以这样理解：<br>当调用者（即应用程序）是阻塞时，此时执行者（即系统）是同步执行。<br>当调用者（即应用程序）是非阻塞时，此时执行者（即系统）是异步执行。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-14 10:38:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/25/02/d4/1e0bb504.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Peter</span>
  </div>
  <div class="_2_QraFYR_0">阻塞、非阻塞 I&#47;O 以及同步、异步 I&#47;O的区别和 同步、异步、阻塞、非阻塞之间区别是不是同一个概念的？ 我之前看过还有同步阻塞、同步非阻塞、异步阻塞、异步非阻塞等</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-17 14:14:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/bf/8f/51f044dc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>谛听</span>
  </div>
  <div class="_2_QraFYR_0">有个疑问，rust 或者 node.js 或者 python 中的 async&#47;await 是同步的还是异步的，是阻塞的还是非阻塞的？有 async 关键字，应该是异步的，可是会等待 await 返回结果后才会继续执行，又觉得是同步的。因为会等待 await 返回结果后才往下走，应该是阻塞的，但实际上其它的 future 也在执行，又不是阻塞的，有点混乱，希望能解答一下</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-08 23:08:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/b2/e0/d856f5a4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>鱼</span>
  </div>
  <div class="_2_QraFYR_0">- 阻塞&#47;非阻塞：针对的是数据未就绪操作是否能够立即返回的划分。非阻塞调用后立马返回，随后通过轮询或者事件得知数据是否就绪<br>- 同步&#47;异步：针对的数据就绪后的获取&#47;写入操作过程是否立刻返回的划分。同步操作有用户线程同步完成，异步操作（比如read）由于用户线程需要立刻返回，数据拷贝的操作由内核完成。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-20 08:47:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83ersRGspOZwfckQcnzQxOzUYdw36wufiaQIic4hfmPrN5arOTuPF7aTz0leNSibs8C3nc3aDuh8CcMtOw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>curry30</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，文中说的“Buffer 的增长是因为，构建目录项缓存所需的元数据（比如文件名称、索引节点等），需要从文件系统中读取”，这里“文件系统中读取”描述有点疑惑，文件系统中读取的话，增长的cache，磁盘中读取的话才是增长Buffer吧，不是很能理解。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-15 08:51:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/3d/a0/acf6b165.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>奋斗</span>
  </div>
  <div class="_2_QraFYR_0">我觉得同步io和异步io最大区别是:同步io需要用户线程去内核主动读取数据，异步io是内核已经将数据返回给用户，用户直接使用即可</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-10 18:41:21</div>
  </div>
</div>
</div>
</li>
</ul>