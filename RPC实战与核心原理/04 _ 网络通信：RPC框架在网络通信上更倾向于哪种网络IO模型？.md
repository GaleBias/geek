<audio title="04 _ 网络通信：RPC框架在网络通信上更倾向于哪种网络IO模型？" src="https://static001.geekbang.org/resource/audio/27/31/273f5c57e767a5122a7e92074a410431.mp3" controls="controls"></audio> 
<p>你好，我是何小锋。在上一讲我讲解了RPC框架中的序列化，通过上一讲，我们知道由于网络传输的数据都是二进制数据，所以我们要传递对象，就必须将对象进行序列化，而RPC框架在序列化的选择上，我们更关注序列化协议的安全性、通用性、兼容性，其次才关注序列化协议的性能、效率、空间开销。承接上一讲，这一讲，我要专门讲解下RPC框架中的网络通信，这也是我们在开篇词中就强调过的重要内容。</p><p>那么网络通信在RPC调用中起到什么作用呢？</p><p>我在<a href="https://time.geekbang.org/column/article/199650">[第 01 讲]</a> 中讲过，RPC是解决进程间通信的一种方式。一次RPC调用，本质就是服务消费者与服务提供者间的一次网络信息交换的过程。服务调用者通过网络IO发送一条请求消息，服务提供者接收并解析，处理完相关的业务逻辑之后，再发送一条响应消息给服务调用者，服务调用者接收并解析响应消息，处理完相关的响应逻辑，一次RPC调用便结束了。可以说，网络通信是整个RPC调用流程的基础。</p><h2>常见的网络IO模型</h2><p>那说到网络通信，就不得不提一下网络IO模型。为什么要讲网络IO模型呢？因为所谓的两台PC机之间的网络通信，实际上就是两台PC机对网络IO的操作。</p><p>常见的网络IO模型分为四种：同步阻塞IO（BIO）、同步非阻塞IO（NIO）、IO多路复用和异步非阻塞IO（AIO）。在这四种IO模型中，只有AIO为异步IO，其他都是同步IO。</p><!-- [[[read_end]]] --><p>其中，最常用的就是同步阻塞IO和IO多路复用，这一点通过了解它们的机制，你会get到。至于其他两种IO模型，因为不常用，则不作为本讲的重点，有兴趣的话我们可以在留言区中讨论。</p><h3>阻塞IO（blocking IO）</h3><p>同步阻塞IO是最简单、最常见的IO模型，在Linux中，默认情况下所有的socket都是blocking的，先看下操作流程。</p><p>首先，应用进程发起IO系统调用后，应用进程被阻塞，转到内核空间处理。之后，内核开始等待数据，等待到数据之后，再将内核中的数据拷贝到用户内存中，整个IO处理完毕后返回进程。最后应用的进程解除阻塞状态，运行业务逻辑。</p><p>这里我们可以看到，系统内核处理IO操作分为两个阶段——等待数据和拷贝数据。而在这两个阶段中，应用进程中IO操作的线程会一直都处于阻塞状态，如果是基于Java多线程开发，那么每一个IO操作都要占用线程，直至IO操作结束。</p><p>这个流程就好比我们去餐厅吃饭，我们到达餐厅，向服务员点餐，之后要一直在餐厅等待后厨将菜做好，然后服务员会将菜端给我们，我们才能享用。</p><h3>IO多路复用（IO multiplexing）</h3><p>多路复用IO是在高并发场景中使用最为广泛的一种IO模型，如Java的NIO、Redis、Nginx的底层实现就是此类IO模型的应用，经典的Reactor模式也是基于此类IO模型。</p><p>那么什么是IO多路复用呢？通过字面上的理解，多路就是指多个通道，也就是多个网络连接的IO，而复用就是指多个通道复用在一个复用器上。</p><p>多个网络连接的IO可以注册到一个复用器（select）上，当用户进程调用了select，那么整个进程会被阻塞。同时，内核会“监视”所有select负责的socket，当任何一个socket中的数据准备好了，select就会返回。这个时候用户进程再调用read操作，将数据从内核中拷贝到用户进程。</p><p>这里我们可以看到，当用户进程发起了select调用，进程会被阻塞，当发现该select负责的socket有准备好的数据时才返回，之后才发起一次read，整个流程要比阻塞IO要复杂，似乎也更浪费性能。但它最大的优势在于，用户可以在一个线程内同时处理多个socket的IO请求。用户可以注册多个socket，然后不断地调用select读取被激活的socket，即可达到在同一个线程内同时处理多个IO请求的目的。而在同步阻塞模型中，必须通过多线程的方式才能达到这个目的。</p><p>同样好比我们去餐厅吃饭，这次我们是几个人一起去的，我们专门留了一个人在餐厅排号等位，其他人就去逛街了，等排号的朋友通知我们可以吃饭了，我们就直接去享用了。</p><h3>为什么说阻塞IO和IO多路复用最为常用？</h3><p>了解完二者的机制，我们就可以回到起初的问题了——我为什么说阻塞IO和IO多路复用最为常用。对比这四种网络IO模型：阻塞IO、非阻塞IO、IO多路复用、异步IO。实际在网络IO的应用上，需要的是系统内核的支持以及编程语言的支持。</p><p>在系统内核的支持上，现在大多数系统内核都会支持阻塞IO、非阻塞IO和IO多路复用，但像信号驱动IO、异步IO，只有高版本的Linux系统内核才会支持。</p><p>在编程语言上，无论C++还是Java，在高性能的网络编程框架的编写上，大多数都是基于Reactor模式，其中最为典型的便是Java的Netty框架，而Reactor模式是基于IO多路复用的。当然，在非高并发场景下，同步阻塞IO是最为常见的。</p><p>综合来讲，在这四种常用的IO模型中，应用最多的、系统内核与编程语言支持最为完善的，便是阻塞IO和IO多路复用。这两种IO模型，已经可以满足绝大多数网络IO的应用场景。</p><h3>RPC框架在网络通信上倾向选择哪种网络IO模型？</h3><p>讲完了这两种最常用的网络IO模型，我们可以看看它们都适合什么样的场景。</p><p>IO多路复用更适合高并发的场景，可以用较少的进程（线程）处理较多的socket的IO请求，但使用难度比较高。当然高级的编程语言支持得还是比较好的，比如Java语言有很多的开源框架对Java原生API做了封装，如Netty框架，使用非常简便；而GO语言，语言本身对IO多路复用的封装就已经很简洁了。</p><p>而阻塞IO与IO多路复用相比，阻塞IO每处理一个socket的IO请求都会阻塞进程（线程），但使用难度较低。在并发量较低、业务逻辑只需要同步进行IO操作的场景下，阻塞IO已经满足了需求，并且不需要发起select调用，开销上还要比IO多路复用低。</p><p>RPC调用在大多数的情况下，是一个高并发调用的场景，考虑到系统内核的支持、编程语言的支持以及IO模型本身的特点，在RPC框架的实现中，在网络通信的处理上，我们会选择IO多路复用的方式。开发语言的网络通信框架的选型上，我们最优的选择是基于Reactor模式实现的框架，如Java语言，首选的框架便是Netty框架（Java还有很多其他NIO框架，但目前Netty应用得最为广泛），并且在Linux环境下，也要开启epoll来提升系统性能（Windows环境下是无法开启epoll的，因为系统内核不支持）。</p><p>了解完以上内容，我们可以继续看这样一个关键问题——零拷贝。在我们应用的过程中，他是非常重要的。</p><h2>什么是零拷贝？</h2><p>刚才讲阻塞IO的时候我讲到，系统内核处理IO操作分为两个阶段——等待数据和拷贝数据。等待数据，就是系统内核在等待网卡接收到数据后，把数据写到内核中；而拷贝数据，就是系统内核在获取到数据后，将数据拷贝到用户进程的空间中。以下是具体流程：</p><p><img src="https://static001.geekbang.org/resource/image/cd/8a/cdf3358f751d2d71564ab58d4f78bc8a.jpg?wh=4146*1478" alt="" title="网络IO读写流程"></p><p>应用进程的每一次写操作，都会把数据写到用户空间的缓冲区中，再由CPU将数据拷贝到系统内核的缓冲区中，之后再由DMA将这份数据拷贝到网卡中，最后由网卡发送出去。这里我们可以看到，一次写操作数据要拷贝两次才能通过网卡发送出去，而用户进程的读操作则是将整个流程反过来，数据同样会拷贝两次才能让应用程序读取到数据。</p><p>应用进程的一次完整的读写操作，都需要在用户空间与内核空间中来回拷贝，并且每一次拷贝，都需要CPU进行一次上下文切换（由用户进程切换到系统内核，或由系统内核切换到用户进程），这样是不是很浪费CPU和性能呢？那有没有什么方式，可以减少进程间的数据拷贝，提高数据传输的效率呢？</p><p>这时我们就需要零拷贝（Zero-copy）技术。</p><p>所谓的零拷贝，就是取消用户空间与内核空间之间的数据拷贝操作，应用进程每一次的读写操作，都可以通过一种方式，让应用进程向用户空间写入或者读取数据，就如同直接向内核空间写入或者读取数据一样，再通过DMA将内核中的数据拷贝到网卡，或将网卡中的数据copy到内核。</p><p>那怎么做到零拷贝？你想一下是不是用户空间与内核空间都将数据写到一个地方，就不需要拷贝了？此时你有没有想到虚拟内存？</p><p><img src="https://static001.geekbang.org/resource/image/00/79/0017969e25ed01f650d7879ac0a2cc79.jpg?wh=3938*1853" alt="" title="虚拟内存"></p><p>零拷贝有两种解决方式，分别是 mmap+write 方式和 sendfile 方式，mmap+write方式的核心原理就是通过虚拟内存来解决的。这两种实现方式都不难，市面上可查阅的资料也很多，在此就不详述了，有问题，可以在留言区中解决。</p><h2>Netty中的零拷贝</h2><p>了解完零拷贝，我们再看看Netty中的零拷贝。</p><p>我刚才讲到，RPC框架在网络通信框架的选型上，我们最优的选择是基于Reactor模式实现的框架，如Java语言，首选的便是Netty框架。那么Netty框架是否也有零拷贝机制呢？Netty框架中的零拷贝和我之前讲的零拷贝又有什么不同呢？</p><p>刚才我讲的零拷贝是操作系统层面上的零拷贝，主要目标是避免用户空间与内核空间之间的数据拷贝操作，可以提升CPU的利用率。</p><p>而Netty的零拷贝则不大一样，他完全站在了用户空间上，也就是JVM上，它的零拷贝主要是偏向于数据操作的优化上。</p><p><strong>那么Netty这么做的意义是什么呢？</strong></p><p>回想下<a href="https://time.geekbang.org/column/article/199651">[第 02 讲]</a>，在这一讲中我讲解了RPC框架如何去设计协议，其中我讲到：在传输过程中，RPC并不会把请求参数的所有二进制数据整体一下子发送到对端机器上，中间可能会拆分成好几个数据包，也可能会合并其他请求的数据包，所以消息都需要有边界。那么一端的机器收到消息之后，就需要对数据包进行处理，根据边界对数据包进行分割和合并，最终获得一条完整的消息。</p><p>那收到消息后，对数据包的分割和合并，是在用户空间完成，还是在内核空间完成的呢？</p><p>当然是在用户空间，因为对数据包的处理工作都是由应用程序来处理的，那么这里有没有可能存在数据的拷贝操作？可能会存在，当然不是在用户空间与内核空间之间的拷贝，是用户空间内部内存中的拷贝处理操作。Netty的零拷贝就是为了解决这个问题，在用户空间对数据操作进行优化。</p><p>那么Netty是怎么对数据操作进行优化的呢？</p><ul>
<li>Netty 提供了 CompositeByteBuf 类，它可以将多个 ByteBuf 合并为一个逻辑上的  ByteBuf，避免了各个 ByteBuf 之间的拷贝。</li>
<li>ByteBuf 支持 slice 操作，因此可以将 ByteBuf 分解为多个共享同一个存储区域的 ByteBuf，避免了内存的拷贝。</li>
<li>通过 wrap 操作，我们可以将 byte[] 数组、ByteBuf、ByteBuffer  等包装成一个 Netty ByteBuf 对象, 进而避免拷贝操作。</li>
</ul><p>Netty框架中很多内部的ChannelHandler实现类，都是通过CompositeByteBuf、slice、wrap操作来处理TCP传输中的拆包与粘包问题的。</p><p>那么Netty有没有解决用户空间与内核空间之间的数据拷贝问题的方法呢？</p><p>Netty  的  ByteBuffer 可以采用 Direct Buffers，使用堆外直接内存进行Socket的读写操作，最终的效果与我刚才讲解的虚拟内存所实现的效果是一样的。</p><p>Netty  还提供  FileRegion  中包装  NIO  的  FileChannel.transferTo()  方法实现了零拷贝，这与Linux  中的  sendfile  方式在原理上也是一样的。</p><h2>总结</h2><p>今天我们详细地介绍了阻塞IO与IO多路复用，拓展了零拷贝相关的知识以及Netty框架中的零拷贝。</p><p>考虑到系统内核的支持、编程语言的支持以及IO模型本身的特点，RPC框架在网络通信的处理上，我们更倾向选择IO多路复用的方式。</p><p>零拷贝带来的好处就是避免没必要的CPU拷贝，让CPU解脱出来去做其他的事，同时也减少了CPU在用户空间与内核空间之间的上下文切换，从而提升了网络通信效率与应用程序的整体性能。</p><p>而Netty的零拷贝与操作系统的零拷贝是有些区别的，Netty的零拷贝偏向于用户空间中对数据操作的优化，这对处理TCP传输中的拆包粘包问题有着重要的意义，对应用程序处理请求数据与返回数据也有重要的意义。</p><p>在 RPC框架的开发与使用过程中，我们要深入了解网络通信相关的原理知识，尽量做到零拷贝，如使用Netty框架；我们要合理使用ByteBuf子类，做到完全零拷贝，提升RPC框架的整体性能。</p><h2>课后思考</h2><p>回想一下，你所接触的开源中间件框架有哪些框架在网络通信上做到了零拷贝？都是使用哪种方式实现的零拷贝？</p><p>欢迎留言和我分享你的思考和疑惑，也欢迎你把文章分享给你的朋友，邀请他加入学习。我们下节课再见！</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/0f/bf/ee93c4cf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>雨霖铃声声慢</span>
  </div>
  <div class="_2_QraFYR_0">IO多路复用分为select，poll和epoll，文中描述的应该是select的过程，nigix，redis等使用的是epoll，所以光只看这个文章的话会比较迷惑，文中写的太粗了。<br>对于课后思考，目前很多的主流的需要通信的中间件都差不多都实现了零拷贝，如Kfaka，RocketMQ等。kafka的零拷贝是通过java.nio.channels.FileChannel中的transferTo方法来实现的，transferTo方法底层是基于操作系统的sendfile这个system call来实现的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 系统层面零拷贝跟应用层零拷贝还是需要区分的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-01 11:22:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/51/b4/0d402ae8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>南桥畂翊</span>
  </div>
  <div class="_2_QraFYR_0">所谓的零拷贝，就是取消用户空间与内核空间之间的数据拷贝操作，应用进程每一次的读写操作，可以通过一种方式，直接将数据写入内核或从内核中读取数据，再通过 DMA 将内核中的数据拷贝到网卡，或将网卡中的数据 copy 到内核。<br><br><br>老师，上述说直接将数据写入内核或从内核中读取数据，这部分内存不是属于内核态空间的吧？应该说只是一块物理内存，用户态虚拟地址和内核态虚拟地址都作了页表映射</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是不太严谨，已改</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-24 09:27:39</div>
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
  <div class="_2_QraFYR_0">老师，我理解的IO多路复用，是应用线程一直再调用select，读取内核准备好数据的socket，所以应用线程是阻塞的，但是老师你文章中举的那个例子，不用应用(用户)打电话去询问的，而是内核(餐馆)打电话通知的，在这期间你还可以去干其他的事情，我感觉你这个案例是异步IO非阻塞的</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-26 21:12:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/9e/42/ab4f65c8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>པག་ཏོན་།</span>
  </div>
  <div class="_2_QraFYR_0">写的非常粗，可见作者对网络知识了解并不深入，买这个课是真的交了智商税</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-28 16:01:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/67/0e/c77ad9b1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>eason2017</span>
  </div>
  <div class="_2_QraFYR_0">Nginx  sendfile方式<br>Kafka  应该是mmap方式，适合小文件</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-17 16:26:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/07/d2/0d7ee298.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>惘 闻</span>
  </div>
  <div class="_2_QraFYR_0">感觉在零拷贝这块讲的不细致,没有图来展示到底是那块做的优化,单单文字描述让人有点误解,而且对比另外几个课程讲解零拷贝的感觉这块老师似乎还讲错了,用户态到内核态的零拷贝.看完这节课疑问更多了....难受</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-01 11:06:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/36/e7/c9eda4e7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>stonyliu</span>
  </div>
  <div class="_2_QraFYR_0">零拷贝这块对于一个非JAVA程序员来说就等于没说啥，给给题目自己查吧</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-29 20:47:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c7/e6/11f21cb4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>川杰</span>
  </div>
  <div class="_2_QraFYR_0">老师好，有句话不大理解：但它最大的优势在于，用户可以在一个线程内同时处理多个socket的IO请求；<br>1、这个线程指的是维护select的线程吗？<br>2、为什么用户可以在一个线程内处理多个socket的IO请求？我理解，这里的用户，应该指的是客户端调用方，那么多个socket，应该是指，其他客户端调用方发送过来的、且IO还未准备好的socket，都被放在了这个select里，然后因为这个用户的select调用，某个IO完成的socket被返回了。如果我理解的没错，那应该还是，只处理了一个请求啊？（那个IO完成的socket）<br><br>3、Reactor模式是为了应对高并发场景的，假设一个极端情况：请求A过来，处理IO稍微花了点时间，后面就没有任何请求过来了，那么请求A是不是永远得不到响应了？因为Reactor是时间驱动的，请求A的socket被放在select里了，没有新的事件触发它去返回；<br>还是说内核会监视，处理完之后就主动返回给客户端？<br>如果内核会主动返回给客户端，那么为什么说：当用户发起了select调用，进程会被阻塞，当发现该select负责的socket有准备好的数据时才返回，之后才发起一次read。<br>麻烦解答下，谢谢！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-26 23:20:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/21/2f/37c96c64.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>陈掌门</span>
  </div>
  <div class="_2_QraFYR_0">讲的真好，全是干货</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-15 13:33:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/06/7e/daa187da.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Q.E.D</span>
  </div>
  <div class="_2_QraFYR_0">tcp不是流协议吗，为什么会有粘包这种说法。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-24 11:45:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/07/d2/0d7ee298.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>惘 闻</span>
  </div>
  <div class="_2_QraFYR_0">缓冲区也是属于用户空间的啊,从缓冲区到内核空间还是需要一次拷贝的啊.为什么这里就说是取消了.只有sendfile可以做到取消用户空间到内核空间的数据拷贝吧?啊啊啊啊,能不能再精致严谨一点呀.</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-01 11:08:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/67/f4/9a1feb59.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>钱</span>
  </div>
  <div class="_2_QraFYR_0">网络IO通信模型和零拷贝，这两个知识点在不同的课程上都有讲解，重要性不言而喻，虽然讲解的都算是高手吧！每个人的讲解方式都不一样，理解起来也是有的容易有的费劲。有此推测完全把这块都系统的弄明白，也能讲出来让他人弄明白是比较困难的。不知道别人什么感受，个人觉得何老师讲解的不是很细致和全面，我不太容量理解 之前觉得这块自己理解了，现在发现并没有。我需要找本书，在好好学习一下。<br><br>不过快的原理还是比较容易理解的，不同的网络IO通信模型之所以，有快慢且相差很大，主要是因为，一个是否消除或减少了线程阻塞等待的过程，另一个是否实现了一个链接供多个线程通信复用。零拷贝之所以快是因为它在做减法，省去一次拷贝的动作，少做事尤其是少做费时间的事情速度自然就快起来了。不过具体到每一步，他们是怎么做到的还有待继续学习，计算机是比较复杂的，光理解信息是怎么表示？怎么存储？怎么传输？怎么运算？都比较烧脑了。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-12 09:02:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/79/04/748ee8c9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>想出家的小和尚</span>
  </div>
  <div class="_2_QraFYR_0">老师，直接内存给我的概念很模糊，他指的到底是什么？和jvm中的堆内内存，堆外内存，用户空间，内核空间有什么关系？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 用户空间和内核空间是系统层面划分的；堆内和堆外是针对jvm进程来讲的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-31 23:21:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/f8/ba/37c24a08.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>学习学习学习学习学习学习学习</span>
  </div>
  <div class="_2_QraFYR_0">**网络io中的零拷贝**<br><br>系统内核处理 IO 操作分为两个阶段——等待数据和拷贝数据。<br><br>- 等待数据，就是系统内核在等待网卡接收到数据后，把数据写到内核中<br>- 拷贝数据，就是系统内核在获取到数据后，将数据拷贝到用户进程的空间中。<br><br>应用进程的每一次写操作，都会把数据写到用户空间的缓冲区中，再由 CPU 将数据拷贝到系统内核的缓冲区中，之后再由 DMA 将这份数据拷贝到网卡中，最后由网卡发送出去。一次写操作数据要拷贝两次才能通过网卡发送出去<br><br>- 零拷贝技术<br>    - 零拷贝，就是取消用户空间与内核空间之间的数据拷贝操作，应用进程每一次的读写操作，都可以通过一种方式，让应用进程向用户空间写入或者读取数据，就如同直接向内核空间写入或者读取数据一样，再通过 DMA 将内核中的数据拷贝到网卡，或将网卡中的数据 copy 到内核。<br>- 零拷贝实现<br>    - mmap+write 方式，核心原理是通过虚拟内存来解决的<br>    - sendfile 方式<br>- Netty零拷贝实现：<br>    - 用户空间数据操作零拷贝优化<br>        - 收到数据包后，在对数据包进行处理时，需要根据协议，处理数据包，在进行处理时，免不了需要进行在用户空间内部内存中进行拷贝处理，Netty就是在用户空间中对数据操作进行优化<br>        - Netty 提供了 CompositeByteBuf 类，它可以将多个 ByteBuf 合并为一个逻辑上的  ByteBuf，避免了各个 ByteBuf 之间的拷贝。<br>        - ByteBuf 支持 slice 操作，因此可以将 ByteBuf 分解为多个共享同一个存储区域的 ByteBuf，避免了内存的拷贝。<br>        - 通过 wrap 操作，我们可以将 byte[] 数组、ByteBuf、ByteBuffer  等包装成一个 Netty ByteBuf 对象, 进而避免拷贝操作。<br>    - 用户空间与内核空间之间零拷贝优化<br>        - Netty  的  ByteBuffer 可以采用 Direct Buffers，使用堆外直接内存进行 Socket 的读写操作，效果和虚拟内存所实现的效果是一样的。<br>        - Netty  还提供  FileRegion  中包装  NIO  的  FileChannel.transferTo()  方法实现了零拷贝，这与 Linux  中的  sendfile  方式在原理上也是一样的。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-22 19:37:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/c2/fe/038a076e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>阿卧</span>
  </div>
  <div class="_2_QraFYR_0">阻塞IO：<br>1. 阻塞等待：多线程进行IO读取，需要阻塞等待<br>2. 内存两次拷贝：从设备（磁盘或者网络）拷贝到用户空间，再从用户空间拷贝到内核空间<br>IO多路复用<br>1. 一个复用器（selector）监听有多个通道（channel）。实现非阻塞式IO读取、写入<br>2. 内存直接拷贝（derict buffers），直接从用户空间拷贝到内核空间</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 很棒</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-01 10:53:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/ec/e0/d072d6f0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>bigdudo</span>
  </div>
  <div class="_2_QraFYR_0">前面的BIO，是一个连接一个线程的处理方式，每条线程自己去监听自己的连接，自己去阻塞并accpet。在这里多路复用，可以理解为是一种批量处理的方式，在一条线程里批量处理多个连接，哪个连接有连接上来了，就向用户进程返回有效的连接。这就是一开始说的多路复用的概念，一个线程处理多个连接。多路复用后面的优化迭代也主要是在批量处理这块的数据模型选型和fd有连接上来的回调的不断优化，从纯遍历方式的select（bitmap承载，最大1024）和poll（数组承载，相对于select，解决了1024有限连接弊端），到cpu终端信号回调的epoll（红黑树管理，有效连接获取放到双向链表并返回个用户进程） 至于用户进程这边怎么处理，是单线程一个一个处理recv&#47;send 还是多线程一个线程一个连接的处理recv&#47;send，这是用户进程的处理方式，也是属于我们常知道的netty这些io中间件的范畴（当然最重要的也是包括基础IO类型的选型也是属于此）。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-03 19:00:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/1R7lHGBvwPTVfb3BAQrPX3AhsYWnXyicbUJUYDgWagWxMGTnsNFvKibzeJ8v7fF2vJLQGf2EY9dV07rnn5Mhv9Uw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>山头</span>
  </div>
  <div class="_2_QraFYR_0">峰哥，记得前几节你说的是rpc框架是异步发送请求的，现在要用多路复用同步请求了。有点迷糊了，能给串串吗</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-17 07:33:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/39/2d/e367c30b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>金十数据</span>
  </div>
  <div class="_2_QraFYR_0">使用零拷贝的前提是没有用异步吗</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-11 17:31:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/f2/62/f873cd8f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>tongmin_tsai</span>
  </div>
  <div class="_2_QraFYR_0">老师，看两张图的对比，原来是应用缓冲区拷贝到内核缓冲区，零拷贝是应用缓冲区拷贝到虚拟内存缓冲区，对比的话，实际上还是有缓冲区的拷贝，怎么理解？老师说零拷贝带来的好处就是避免没必要的 CPU 拷贝，看图上面，感觉反而在应用缓冲区与内核缓冲区之间还加入了一个缓冲区，那不是效率更低了吗？望老师解答</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-26 09:41:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>嘻嘻</span>
  </div>
  <div class="_2_QraFYR_0">老师，compositeBytebuf那一段，用一个大的字节数组存新读到的片段，不断移动读写指针确定读写位置，循环利用相当于一个ringbuf 不也可以？为什么要搞那么复杂？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-22 18:27:13</div>
  </div>
</div>
</div>
</li>
</ul>