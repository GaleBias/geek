<audio title="03 _ 高性能IO模型：为什么单线程Redis能那么快？" src="https://static001.geekbang.org/resource/audio/08/43/080cc05798a394b3d1f6e1fc764dc843.mp3" controls="controls"></audio> 
<p>你好，我是蒋德钧。</p><p>今天，我们来探讨一个很多人都很关心的问题：“为什么单线程的Redis能那么快？”</p><p>首先，我要和你厘清一个事实，我们通常说，Redis是单线程，主要是指<strong>Redis的网络IO和键值对读写是由一个线程来完成的，这也是Redis对外提供键值存储服务的主要流程</strong>。但Redis的其他功能，比如持久化、异步删除、集群数据同步等，其实是由额外的线程执行的。</p><p>所以，严格来说，Redis并不是单线程，但是我们一般把Redis称为单线程高性能，这样显得“酷”些。接下来，我也会把Redis称为单线程模式。而且，这也会促使你紧接着提问：“为什么用单线程？为什么单线程能这么快？”</p><p>要弄明白这个问题，我们就要深入地学习下Redis的单线程设计机制以及多路复用机制。之后你在调优Redis性能时，也能更有针对性地避免会导致Redis单线程阻塞的操作，例如执行复杂度高的命令。</p><p>好了，话不多说，接下来，我们就先来学习下Redis采用单线程的原因。</p><h2>Redis为什么用单线程？</h2><p>要更好地理解Redis为什么用单线程，我们就要先了解多线程的开销。</p><h3>多线程的开销</h3><p>日常写程序时，我们经常会听到一种说法：“使用多线程，可以增加系统吞吐率，或是可以增加系统扩展性。”的确，对于一个多线程的系统来说，在有合理的资源分配的情况下，可以增加系统中处理请求操作的资源实体，进而提升系统能够同时处理的请求数，即吞吐率。下面的左图是我们采用多线程时所期待的结果。</p><!-- [[[read_end]]] --><p>但是，请你注意，通常情况下，在我们采用多线程后，如果没有良好的系统设计，实际得到的结果，其实是右图所展示的那样。我们刚开始增加线程数时，系统吞吐率会增加，但是，再进一步增加线程时，系统吞吐率就增长迟缓了，有时甚至还会出现下降的情况。</p><p><img src="https://static001.geekbang.org/resource/image/cb/33/cbd394e62219cc5a6d9ae64035e51733.jpg?wh=2284*858" alt="" title="线程数与系统吞吐率"></p><p>为什么会出现这种情况呢？一个关键的瓶颈在于，系统中通常会存在被多线程同时访问的共享资源，比如一个共享的数据结构。当有多个线程要修改这个共享资源时，为了保证共享资源的正确性，就需要有额外的机制进行保证，而这个额外的机制，就会带来额外的开销。</p><p>拿Redis来说，在上节课中，我提到过，Redis有List的数据类型，并提供出队（LPOP）和入队（LPUSH）操作。假设Redis采用多线程设计，如下图所示，现在有两个线程A和B，线程A对一个List做LPUSH操作，并对队列长度加1。同时，线程B对该List执行LPOP操作，并对队列长度减1。为了保证队列长度的正确性，Redis需要让线程A和B的LPUSH和LPOP串行执行，这样一来，Redis可以无误地记录它们对List长度的修改。否则，我们可能就会得到错误的长度结果。这就是<strong>多线程编程模式面临的共享资源的并发访问控制问题</strong>。</p><p><img src="https://static001.geekbang.org/resource/image/30/08/303255dcce6d0837bf7e2440df0f8e08.jpg?wh=3537*2250" alt="" title="多线程并发访问Redis"></p><p>并发访问控制一直是多线程开发中的一个难点问题，如果没有精细的设计，比如说，只是简单地采用一个粗粒度互斥锁，就会出现不理想的结果：即使增加了线程，大部分线程也在等待获取访问共享资源的互斥锁，并行变串行，系统吞吐率并没有随着线程的增加而增加。</p><p>而且，采用多线程开发一般会引入同步原语来保护共享资源的并发访问，这也会降低系统代码的易调试性和可维护性。为了避免这些问题，Redis直接采用了单线程模式。</p><p>讲到这里，你应该已经明白了“Redis为什么用单线程”，那么，接下来，我们就来看看，为什么单线程Redis能获得高性能。</p><h2>单线程Redis为什么那么快？</h2><p>通常来说，单线程的处理能力要比多线程差很多，但是Redis却能使用单线程模型达到每秒数十万级别的处理能力，这是为什么呢？其实，这是Redis多方面设计选择的一个综合结果。</p><p>一方面，Redis的大部分操作在内存上完成，再加上它采用了高效的数据结构，例如哈希表和跳表，这是它实现高性能的一个重要原因。另一方面，就是Redis采用了<strong>多路复用机制</strong>，使其在网络IO操作中能并发处理大量的客户端请求，实现高吞吐率。接下来，我们就重点学习下多路复用机制。</p><p>首先，我们要弄明白网络操作的基本IO模型和潜在的阻塞点。毕竟，Redis采用单线程进行IO，如果线程被阻塞了，就无法进行多路复用了。</p><h3>基本IO模型与阻塞点</h3><p>你还记得我在<a href="https://time.geekbang.org/column/article/268262">第一节课</a>介绍的具有网络框架的SimpleKV吗？</p><p>以Get请求为例，SimpleKV为了处理一个Get请求，需要监听客户端请求（bind/listen），和客户端建立连接（accept），从socket中读取请求（recv），解析客户端发送请求（parse），根据请求类型读取键值数据（get），最后给客户端返回结果，即向socket中写回数据（send）。</p><p>下图显示了这一过程，其中，bind/listen、accept、recv、parse和send属于网络IO处理，而get属于键值数据操作。既然Redis是单线程，那么，最基本的一种实现是在一个线程中依次执行上面说的这些操作。</p><p><img src="https://static001.geekbang.org/resource/image/e1/c9/e18499ab244e4428a0e60b4da6575bc9.jpg?wh=2700*1493" alt="" title="Redis基本IO模型"></p><p>但是，在这里的网络IO操作中，有潜在的阻塞点，分别是accept()和recv()。当Redis监听到一个客户端有连接请求，但一直未能成功建立起连接时，会阻塞在accept()函数这里，导致其他客户端无法和Redis建立连接。类似的，当Redis通过recv()从一个客户端读取数据时，如果数据一直没有到达，Redis也会一直阻塞在recv()。</p><p>这就导致Redis整个线程阻塞，无法处理其他客户端请求，效率很低。不过，幸运的是，socket网络模型本身支持非阻塞模式。</p><h3>非阻塞模式</h3><p>Socket网络模型的非阻塞模式设置，主要体现在三个关键的函数调用上，如果想要使用socket非阻塞模式，就必须要了解这三个函数的调用返回类型和设置模式。接下来，我们就重点学习下它们。</p><p>在socket模型中，不同操作调用后会返回不同的套接字类型。socket()方法会返回主动套接字，然后调用listen()方法，将主动套接字转化为监听套接字，此时，可以监听来自客户端的连接请求。最后，调用accept()方法接收到达的客户端连接，并返回已连接套接字。</p><p><img src="https://static001.geekbang.org/resource/image/1c/4a/1ccc62ab3eb2a63c4965027b4248f34a.jpg?wh=3744*1077" alt="" title="Redis套接字类型与非阻塞设置"></p><p>针对监听套接字，我们可以设置非阻塞模式：当Redis调用accept()但一直未有连接请求到达时，Redis线程可以返回处理其他操作，而不用一直等待。但是，你要注意的是，调用accept()时，已经存在监听套接字了。</p><p>虽然Redis线程可以不用继续等待，但是总得有机制继续在监听套接字上等待后续连接请求，并在有请求时通知Redis。</p><p>类似的，我们也可以针对已连接套接字设置非阻塞模式：Redis调用recv()后，如果已连接套接字上一直没有数据到达，Redis线程同样可以返回处理其他操作。我们也需要有机制继续监听该已连接套接字，并在有数据达到时通知Redis。</p><p>这样才能保证Redis线程，既不会像基本IO模型中一直在阻塞点等待，也不会导致Redis无法处理实际到达的连接请求或数据。</p><p>到此，Linux中的IO多路复用机制就要登场了。</p><h3>基于多路复用的高性能I/O模型</h3><p>Linux中的IO多路复用机制是指一个线程处理多个IO流，就是我们经常听到的select/epoll机制。简单来说，在Redis只运行单线程的情况下，<strong>该机制允许内核中，同时存在多个监听套接字和已连接套接字</strong>。内核会一直监听这些套接字上的连接请求或数据请求。一旦有请求到达，就会交给Redis线程处理，这就实现了一个Redis线程处理多个IO流的效果。</p><p>下图就是基于多路复用的Redis IO模型。图中的多个FD就是刚才所说的多个套接字。Redis网络框架调用epoll机制，让内核监听这些套接字。此时，Redis线程不会阻塞在某一个特定的监听或已连接套接字上，也就是说，不会阻塞在某一个特定的客户端请求处理上。正因为此，Redis可以同时和多个客户端连接并处理请求，从而提升并发性。</p><p><img src="https://static001.geekbang.org/resource/image/00/ea/00ff790d4f6225aaeeebba34a71d8bea.jpg?wh=3472*2250" alt="" title="基于多路复用的Redis高性能IO模型"></p><p>为了在请求到达时能通知到Redis线程，select/epoll提供了<strong>基于事件的回调机制</strong>，即<strong>针对不同事件的发生，调用相应的处理函数</strong>。</p><p>那么，回调机制是怎么工作的呢？其实，select/epoll一旦监测到FD上有请求到达时，就会触发相应的事件。</p><p>这些事件会被放进一个事件队列，Redis单线程对该事件队列不断进行处理。这样一来，Redis无需一直轮询是否有请求实际发生，这就可以避免造成CPU资源浪费。同时，Redis在对事件队列中的事件进行处理时，会调用相应的处理函数，这就实现了基于事件的回调。因为Redis一直在对事件队列进行处理，所以能及时响应客户端请求，提升Redis的响应性能。</p><p>为了方便你理解，我再以连接请求和读数据请求为例，具体解释一下。</p><p>这两个请求分别对应Accept事件和Read事件，Redis分别对这两个事件注册accept和get回调函数。当Linux内核监听到有连接请求或读数据请求时，就会触发Accept事件和Read事件，此时，内核就会回调Redis相应的accept和get函数进行处理。</p><p>这就像病人去医院瞧病。在医生实际诊断前，每个病人（等同于请求）都需要先分诊、测体温、登记等。如果这些工作都由医生来完成，医生的工作效率就会很低。所以，医院都设置了分诊台，分诊台会一直处理这些诊断前的工作（类似于Linux内核监听请求），然后再转交给医生做实际诊断。这样即使一个医生（相当于Redis单线程），效率也能提升。</p><p>不过，需要注意的是，即使你的应用场景中部署了不同的操作系统，多路复用机制也是适用的。因为这个机制的实现有很多种，既有基于Linux系统下的select和epoll实现，也有基于FreeBSD的kqueue实现，以及基于Solaris的evport实现，这样，你可以根据Redis实际运行的操作系统，选择相应的多路复用实现。</p><h2>小结</h2><p>今天，我们重点学习了Redis线程的三个问题：“Redis真的只有单线程吗？”“为什么用单线程？”“单线程为什么这么快？”</p><p>现在，我们知道了，Redis单线程是指它对网络IO和数据读写的操作采用了一个线程，而采用单线程的一个核心原因是避免多线程开发的并发控制问题。单线程的Redis也能获得高性能，跟多路复用的IO模型密切相关，因为这避免了accept()和send()/recv()潜在的网络IO操作阻塞点。</p><p>搞懂了这些，你就走在了很多人的前面。如果你身边还有不清楚这几个问题的朋友，欢迎你分享给他/她，解决他们的困惑。</p><p>另外，我也剧透下，可能你也注意到了，2020年5月，Redis 6.0的稳定版发布了，Redis  6.0中提出了多线程模型。那么，这个多线程模型和这节课所说的IO模型有什么关联？会引入复杂的并发控制问题吗？会给Redis 6.0带来多大提升？关于这些问题，我会在后面的课程中和你具体介绍。</p><h2>每课一问</h2><p>这节课，我给你提个小问题，在“Redis基本IO模型”图中，你觉得还有哪些潜在的性能瓶颈吗？欢迎在留言区写下你的思考和答案，我们一起交流讨论。</p>
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
  <div class="_2_QraFYR_0">Redis单线程处理IO请求性能瓶颈主要包括2个方面：<br><br>1、任意一个请求在server中一旦发生耗时，都会影响整个server的性能，也就是说后面的请求都要等前面这个耗时请求处理完成，自己才能被处理到。耗时的操作包括以下几种：<br>	a、操作bigkey：写入一个bigkey在分配内存时需要消耗更多的时间，同样，删除bigkey释放内存同样会产生耗时；<br>	b、使用复杂度过高的命令：例如SORT&#47;SUNION&#47;ZUNIONSTORE，或者O(N)命令，但是N很大，例如lrange key 0 -1一次查询全量数据；<br>	c、大量key集中过期：Redis的过期机制也是在主线程中执行的，大量key集中过期会导致处理一个请求时，耗时都在删除过期key，耗时变长；<br>	d、淘汰策略：淘汰策略也是在主线程执行的，当内存超过Redis内存上限后，每次写入都需要淘汰一些key，也会造成耗时变长；<br>	e、AOF刷盘开启always机制：每次写入都需要把这个操作刷到磁盘，写磁盘的速度远比写内存慢，会拖慢Redis的性能；<br>	f、主从全量同步生成RDB：虽然采用fork子进程生成数据快照，但fork这一瞬间也是会阻塞整个线程的，实例越大，阻塞时间越久；<br>2、并发量非常大时，单线程读写客户端IO数据存在性能瓶颈，虽然采用IO多路复用机制，但是读写客户端数据依旧是同步IO，只能单线程依次读取客户端的数据，无法利用到CPU多核。<br><br>针对问题1，一方面需要业务人员去规避，一方面Redis在4.0推出了lazy-free机制，把bigkey释放内存的耗时操作放在了异步线程中执行，降低对主线程的影响。<br><br>针对问题2，Redis在6.0推出了多线程，可以在高并发场景下利用CPU多核多线程读写客户端数据，进一步提升server性能，当然，只是针对客户端的读写是并行的，每个命令的真正操作依旧是单线程的。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-10 11:37:32</div>
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
  <div class="_2_QraFYR_0">1.big key的操作。<br>2.潜在的大量数据操作，比如 key *或者get all之类的操作，所以才引入了scan的相关操作。<br>3.特殊的场景，大量的客户端接入。<br><br><br>简单介绍下select poll epoll的区别，select和poll本质上没啥区别，就是文件描述符数量的限制，select根据不同的系统，文件描述符限制为1024或者2048，poll没有数量限制。他两都是把文件描述符集合保存在用户态，每次把集合传入内核态，内核态返回ready的文件描述符。<br>epoll是通过epoll_create和epoll_ctl和epoll_await三个系统调用完成的，每当接入一个文件描述符，通过ctl添加到内核维护的红黑树中，通过事件机制，当数据ready后，从红黑树移动到链表，通过await获取链表中准备好数据的fd，程序去处理。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-10 09:15:17</div>
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
  <div class="_2_QraFYR_0">Redis 的单线程指 Redis 的网络 IO 和键值对读写由一个线程来完成的（这是 Redis 对外提供键值对存储服务的主要流程）<br>Redis 的持久化、异步删除、集群数据同步等功能是由其他线程而不是主线程来执行的，所以严格来说，Redis 并不是单线程<br><br>为什么用单线程？<br>多线程会有共享资源的并发访问控制问题，为了避免这些问题，Redis 采用了单线程的模式，而且采用单线程对于 Redis 的内部实现的复杂度大大降低<br><br>为什么单线程就挺快？<br>1.Redis 大部分操作是在内存上完成，并且采用了高效的数据结构如哈希表和跳表<br>2.Redis 采用多路复用，能保证在网络 IO 中可以并发处理大量的客户端请求，实现高吞吐率<br><br>Redis 6.0 版本为什么又引入了多线程？<br>Redis 的瓶颈不在 CPU ，而在内存和网络，内存不够可以增加内存或通过数据结构等进行优化<br>但 Redis 的网络 IO 的读写占用了发部分 CPU 的时间，如果可以把网络处理改成多线程的方式，性能会有很大提升<br>所以总结下 Redis 6.0 版本引入多线程有两个原因<br>1.充分利用服务器的多核资源<br>2.多线程分摊 Redis 同步 IO 读写负荷<br><br>执行命令还是由单线程顺序执行，只是处理网络数据读写采用了多线程，而且 IO 线程要么同时读 Socket ，要么同时写 Socket ，不会同时读写</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-13 09:57:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/63/c6/0167415c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>林肯</span>
  </div>
  <div class="_2_QraFYR_0">反复看了很多遍终于明白了：关键点在于accpet和recv时可能会阻塞线程，使用IO多路复用技术可以让线程先处理其他事情，等需要的资源到位后epoll会调用回调函数通知线程，然后线程再去处理存&#47;取数据；这样一个redis服务端线程就可以同时处理多个客户端请求了。<br>redis之所以适合用多路复用技术有一个很重要的原因时它是在内存中处理数据速度极快，这时io成了瓶颈。为什么Mysql不用多路复用技术呢？因为Mysql的主要性能瓶颈在于数据的存&#47;取，优化方向不一样。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-15 11:15:40</div>
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
  <div class="_2_QraFYR_0">1，作者讲了什么？<br>	redis实现单线程实现高性能IO的设计机制<br>2，作者是怎么把这事给讲明白的？<br>	作者首先从简单的网络通信socket讲起，引出了非阻塞socket，由此谈到了著名的I&#47;O多路复用，Linux内核的select&#47;epoll机制<br>3，为了讲明白，作者讲了哪些要点?有哪些亮点？<br>	（1）首先声明“redis单线程”这个概念的具体含义<br>	（2）引入具体业务场景：redis的数据读取，事件处理机制模型<br>	（3）解析单线程相对多线程带来的优势，已及多线程所特有的问题<br>	（4）基于redis单线程的，设计机制，引出了网络socket的问题</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-11 19:40:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/fb/86/a380cbad.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>柳磊</span>
  </div>
  <div class="_2_QraFYR_0">作者您好，引用文中一段话“我们知道了，Redis 单线程是指它对网络 IO 和数据读写的操作采用了一个线程”，我有个疑问，redis为什么要网络IO与业务处理（读写）用一个线程？而不用Netty中常见的Reactor线程模型，把io线程（netty中的boss线程）与业务处理线程（netty中的work线程）分开，业务处理线程只开启一个线程，也不会有共享资源竞争的问题。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 在Redis 6.0版本前，Redis用一个线程实现网络请求的解析和读写处理，我个人觉得主要还是这种线程模型实现简单。<br><br>不过随着网络硬件越来越快后，网络请求收发更快了，所以从Redis 6.0开始，网络请求解析也是由专门的线程处理，从而支持快速网络读写。而读写处理仍然由单个主线程执行，这是为了避免多线程协调的开销。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-15 10:03:20</div>
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
  <div class="_2_QraFYR_0">虽然单线程很快，没有锁的单线程更快借助CPU的多级缓存可以把性能发挥到最大。但是随着访问量的增加，以及数据量的增加，IO的写入写出会成为性能瓶颈。10个socket的IO吞吐处理肯定比1000个socket吞吐处理的快，为了解决这个问题，Redis6引入了IO多线程的方式以及client缓冲区，在实际指令处理还是单线程模式。在IO上变成的了【主线程】带着众多【IO线程】进行IO，IO线程听从主线程的指挥是写入还是写出。Read的时候IO线程会和主线程一起读取并且解析命令（RESP协议）存入缓冲区，写的时候会从缓冲区写出到Socket。IO线程听从主线程的指挥，在同一个时间点上主线程和IO线程会一起写出或者读取，并且主线程会等待IO线程的结束。但是这种模式的多线程会面临一给NUMA陷阱的问题，在最近的Redis版本中加强了IO线程和CPU的亲和性解决了这个问题。（不过目前官方在默认情况下并不推荐使用多线程IO模式，需要手动开启）</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-10 14:12:52</div>
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
  <div class="_2_QraFYR_0">Redis的网络模式是单reactor模式。non-blocking io + epoll</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-17 19:30:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTIiao0orF0gDeDCwnAEicrCgY6NickyOJ8ialw0GiavInZL0DMctRYlZicj4bLMNTtBmFtH4eIiaVfr8DPVw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_84971a</span>
  </div>
  <div class="_2_QraFYR_0">老师在讲解redis网络IO模型的时候，如果可以结合epoll的多路复用机制，顺便提一下redis源码里面的实现，相信可以让人理解的更加深入一些；</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-30 14:00:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/1f/66/59e0647a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>万历十五年</span>
  </div>
  <div class="_2_QraFYR_0">个人理解，IO多路复用简单说是IO阻塞或非阻塞的都不准确。严格来说应用程序从网络读取数据到数据可用，分两个阶段：第一阶段读网络数据到内核，第二阶段读内核数据到用户态。IO多路复用解决了第一阶段阻塞问题，而第二阶段的读取阻塞的串行读。为了进一步提高REDIS的吞吐量，REDIS6.0使用多线程利用多CPU的优势解决第二阶段的阻塞。说的不对的地方，请斧正。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-23 11:48:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLzvaL724GwtzZ5mcldUnlicicSlI8BXL9icRZbUOB10qjRMlmog7UTvwxSBHXagnPGGR1BYdjWcGGSg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>wwj</span>
  </div>
  <div class="_2_QraFYR_0">这篇的io模型和我了解到的不一样，既然是select&#47;epoll的模式，那应该就是Reactor设计模式，哪来的回调，回调肯定设计到多个线程，单线程模式在用户层不可能有回调的，如果是在内核层的话，是有aio模式，但select&#47;epoll明显不是aio的实现</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-27 10:59:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/PiajxSqBRaELZPnUAiajaR5C25EDLWeJURggyiaOP5GGPe2qlwpQcm5e3ybib8OsP4tvddFDLVRSNNGL5I3SFPJHsA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>null</span>
  </div>
  <div class="_2_QraFYR_0">re: <br>原文一：Redis 在对事件队列中的事件进行处理时，会调用相应的处理函数，这就实现了基于事件的回调。<br><br>原文二：此时，内核就会回调 Redis 相应的 accept 和 get 函数进行处理。<br><br><br>是redis 调用accept 和 get函数，还是内核吖？谢谢老师。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-24 09:50:38</div>
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
  <div class="_2_QraFYR_0">Redis的事件处理队列只有一个吗？不同的事件的优先级都是一样的吗？只是简单的按照对接的先进先出的特性依次进行处理的吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-11 21:21:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/59/8a/82587883.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>竹真</span>
  </div>
  <div class="_2_QraFYR_0">作者您好，读完您的文章还有点疑惑，Redis读取客户端数据和读内存是一个线程？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Redis 6.0前的版本是用一个线程来读取网络请求并进行解析，并根据请求的具体命令操作进行数据读写的。Redis 6.0开始，网络请求的解析可以用多线程来执行，但是读写内存还是一个线程。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-06 10:01:28</div>
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
  <div class="_2_QraFYR_0">老师分析的 redis io 模型中，redis 线程是循环处理每个事件的。如果其中一个事件比较耗时，会影响后面事件的及时处理。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，所以如果有慢操作的话，就会影响其他操作了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-14 08:32:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/22/72/1f/9ddfeff7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>文进</span>
  </div>
  <div class="_2_QraFYR_0">IO多路的单线程模型：<br>1，redis启动时，向epoll注册还未使用的FD的可连接事件。<br>2，连接事件产生时，epoll机制自动将其放入自己的可连接事件队列中。<br>3，redis线程调用epoll的wait，获取所有事件，拷贝存入自己用户线程的内存队列中。<br>4，遍历这些内存队列，通过事件分派器，交由对应事件处理器处理。如是可连接事件则交由对应的连接应答事件处理器处理。可读事件交给命令请求处理器，可写事件交给命令回复处理器处理。<br>5，对应处理器执行相应逻辑。执行完成时，再次向epoll注册对应事件，比如连接事件的那个FD，接下来要注册可读事件，并重新注册可连接事件。命令请求处理器执行完后，要将FD注册可写事件。命令回复处理器执行完后，需要将FD取消注册可写事件。<br>6，产生了新的注册（如可读事件）之后，又回到了2，等待新的事件产生。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-28 10:31:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/fd/94/0247f945.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>咸鱼</span>
  </div>
  <div class="_2_QraFYR_0">这章让我对IO多路复用的理解又深了些</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-10 00:31:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/c7/f8/6b311ad9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>BrightLoong</span>
  </div>
  <div class="_2_QraFYR_0">Redis 单线程是指它对网络 IO 和数据读写的操作采用了一个线程<br>对这句话不是很理解，总觉得是处理网络IO是一个线程，然后把事件放入队列；读写操作又是一个线程，从队列中处理请求。求解答</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 在Redis 6.0前，网络请求的解析和数据读写都是由主线程来完成的，这也是我们称之为单线程的原因。<br><br>从Redis 6.0开始，网络请求的解析是由其他线程完成，然后把解析后的请求交由主线程进行实际的内存读写。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-12 18:11:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/7b/5f/3400d01b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Y、先生</span>
  </div>
  <div class="_2_QraFYR_0">现在，我们知道了，Redis 单线程是指它对网络 IO 和数据读写的操作采用了一个线程。 <br>这里的网络IO是指redis处理事件队列的阶段么，数据读写对应的是回调函数，是这样理解吗。 这部分很困惑，希望老师帮忙确认下</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-24 20:03:08</div>
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
  <div class="_2_QraFYR_0">忘记提另一个问题，既然老师说道select和epoll，为什么不提一下poll呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-12 14:49:31</div>
  </div>
</div>
</div>
</li>
</ul>