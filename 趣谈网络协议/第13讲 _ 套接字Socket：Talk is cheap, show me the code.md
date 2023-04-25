<audio title="第13讲 _ 套接字Socket：Talk is cheap, show me the code" src="https://static001.geekbang.org/resource/audio/75/6f/75b47bb7d89a675b096c50ed84d2da6f.mp3" controls="controls"></audio> 
<p>前面讲完了TCP和UDP协议，还没有上手过，这一节咱们讲讲基于TCP和UDP协议的Socket编程。</p><p>在讲TCP和UDP协议的时候，我们分客户端和服务端，在写程序的时候，我们也同样这样分。</p><p>Socket这个名字很有意思，可以作插口或者插槽讲。虽然我们是写软件程序，但是你可以想象为弄一根网线，一头插在客户端，一头插在服务端，然后进行通信。所以在通信之前，双方都要建立一个Socket。</p><p>在建立Socket的时候，应该设置什么参数呢？Socket编程进行的是端到端的通信，往往意识不到中间经过多少局域网，多少路由器，因而能够设置的参数，也只能是端到端协议之上网络层和传输层的。</p><p>在网络层，Socket函数需要指定到底是IPv4还是IPv6，分别对应设置为AF_INET和AF_INET6。另外，还要指定到底是TCP还是UDP。还记得咱们前面讲过的，TCP协议是基于数据流的，所以设置为SOCK_STREAM，而UDP是基于数据报的，因而设置为SOCK_DGRAM。</p><h2>基于TCP协议的Socket程序函数调用过程</h2><p>两端创建了Socket之后，接下来的过程中，TCP和UDP稍有不同，我们先来看TCP。</p><p>TCP的服务端要先监听一个端口，一般是先调用bind函数，给这个Socket赋予一个IP地址和端口。为什么需要端口呢？要知道，你写的是一个应用程序，当一个网络包来的时候，内核要通过TCP头里面的这个端口，来找到你这个应用程序，把包给你。为什么要IP地址呢？有时候，一台机器会有多个网卡，也就会有多个IP地址，你可以选择监听所有的网卡，也可以选择监听一个网卡，这样，只有发给这个网卡的包，才会给你。</p><!-- [[[read_end]]] --><p>当服务端有了IP和端口号，就可以调用listen函数进行监听。在TCP的状态图里面，有一个listen状态，当调用这个函数之后，服务端就进入了这个状态，这个时候客户端就可以发起连接了。</p><p>在内核中，为每个Socket维护两个队列。一个是已经建立了连接的队列，这时候连接三次握手已经完毕，处于established状态；一个是还没有完全建立连接的队列，这个时候三次握手还没完成，处于syn_rcvd的状态。</p><p>接下来，服务端调用accept函数，拿出一个已经完成的连接进行处理。如果还没有完成，就要等着。</p><p>在服务端等待的时候，客户端可以通过connect函数发起连接。先在参数中指明要连接的IP地址和端口号，然后开始发起三次握手。内核会给客户端分配一个临时的端口。一旦握手成功，服务端的accept就会返回另一个Socket。</p><p>这是一个经常考的知识点，就是监听的Socket和真正用来传数据的Socket是两个，一个叫作<strong>监听Socket</strong>，一个叫作<strong>已连接Socket</strong>。</p><p>连接建立成功之后，双方开始通过read和write函数来读写数据，就像往一个文件流里面写东西一样。</p><p>这个图就是基于TCP协议的Socket程序函数调用过程。</p><p><img src="https://static001.geekbang.org/resource/image/87/ea/87c8ae36ae1b42653565008fc47aceea.jpg?wh=1626*2172" alt=""></p><p>说TCP的Socket就是一个文件流，是非常准确的。因为，Socket在Linux中就是以文件的形式存在的。除此之外，还存在文件描述符。写入和读出，也是通过文件描述符。</p><p>在内核中，Socket是一个文件，那对应就有文件描述符。每一个进程都有一个数据结构task_struct，里面指向一个文件描述符数组，来列出这个进程打开的所有文件的文件描述符。文件描述符是一个整数，是这个数组的下标。</p><p>这个数组中的内容是一个指针，指向内核中所有打开的文件的列表。既然是一个文件，就会有一个inode，只不过Socket对应的inode不像真正的文件系统一样，保存在硬盘上的，而是在内存中的。在这个inode中，指向了Socket在内核中的Socket结构。</p><p>在这个结构里面，主要的是两个队列，一个是<strong>发送队列</strong>，一个是<strong>接收队列</strong>。在这两个队列里面保存的是一个缓存sk_buff。这个缓存里面能够看到完整的包的结构。看到这个，是不是能和前面讲过的收发包的场景联系起来了？</p><p>整个数据结构我也画了一张图。<br>
<img src="https://static001.geekbang.org/resource/image/60/13/604f4cb37576990b3f836cb5d7527b13.jpg?wh=3646*2998" alt=""></p><h2>基于UDP协议的Socket程序函数调用过程</h2><p>对于UDP来讲，过程有些不一样。UDP是没有连接的，所以不需要三次握手，也就不需要调用listen和connect，但是，UDP的交互仍然需要IP和端口号，因而也需要bind。UDP是没有维护连接状态的，因而不需要每对连接建立一组Socket，而是只要有一个Socket，就能够和多个客户端通信。也正是因为没有连接状态，每次通信的时候，都调用sendto和recvfrom，都可以传入IP地址和端口。</p><p>这个图的内容就是基于UDP协议的Socket程序函数调用过程。<br>
<img src="https://static001.geekbang.org/resource/image/6b/31/6bbe12c264f5e76a81523eb8787f3931.jpg?wh=1245*1261" alt=""></p><h2>服务器如何接更多的项目？</h2><p>会了这几个基本的Socket函数之后，你就可以轻松地写一个网络交互的程序了。就像上面的过程一样，在建立连接后，进行一个while循环。客户端发了收，服务端收了发。</p><p>当然这只是万里长征的第一步，因为如果使用这种方法，基本上只能一对一沟通。如果你是一个服务器，同时只能服务一个客户，肯定是不行的。这就相当于老板成立一个公司，只有自己一个人，自己亲自上来服务客户，只能干完了一家再干下一家，这样赚不来多少钱。</p><p>那作为老板你就要想了，我最多能接多少项目呢？当然是越多越好。</p><p>我们先来算一下理论值，也就是<strong>最大连接数</strong>，系统会用一个四元组来标识一个TCP连接。</p><pre><code>{本机IP, 本机端口, 对端IP, 对端端口}
</code></pre><p>服务器通常固定在某个本地端口上监听，等待客户端的连接请求。因此，服务端端TCP连接四元组中只有对端IP, 也就是客户端的IP和对端的端口，也即客户端的端口是可变的，因此，最大TCP连接数=客户端IP数×客户端端口数。对IPv4，客户端的IP数最多为2的32次方，客户端的端口数最多为2的16次方，也就是服务端单机最大TCP连接数，约为2的48次方。</p><p>当然，服务端最大并发TCP连接数远不能达到理论上限。首先主要是<strong>文件描述符限制</strong>，按照上面的原理，Socket都是文件，所以首先要通过ulimit配置文件描述符的数目；另一个限制是<strong>内存</strong>，按上面的数据结构，每个TCP连接都要占用一定内存，操作系统是有限的。</p><p>所以，作为老板，在资源有限的情况下，要想接更多的项目，就需要降低每个项目消耗的资源数目。</p><h3>方式一：将项目外包给其他公司（多进程方式）</h3><p>这就相当于你是一个代理，在那里监听来的请求。一旦建立了一个连接，就会有一个已连接Socket，这时候你可以创建一个子进程，然后将基于已连接Socket的交互交给这个新的子进程来做。就像来了一个新的项目，但是项目不一定是你自己做，可以再注册一家子公司，招点人，然后把项目转包给这家子公司做，以后对接就交给这家子公司了，你又可以去接新的项目了。</p><p>这里有一个问题是，如何创建子公司，并如何将项目移交给子公司呢？</p><p>在Linux下，创建子进程使用fork函数。通过名字可以看出，这是在父进程的基础上完全拷贝一个子进程。在Linux内核中，会复制文件描述符的列表，也会复制内存空间，还会复制一条记录当前执行到了哪一行程序的进程。显然，复制的时候在调用fork，复制完毕之后，父进程和子进程都会记录当前刚刚执行完fork。这两个进程刚复制完的时候，几乎一模一样，只是根据fork的返回值来区分到底是父进程，还是子进程。如果返回值是0，则是子进程；如果返回值是其他的整数，就是父进程。</p><p>进程复制过程我画在这里。<br>
<img src="https://static001.geekbang.org/resource/image/18/d0/18070c00ff5d0082yy1fbc32b84e73d0.jpg?wh=4842*3559" alt=""></p><p>因为复制了文件描述符列表，而文件描述符都是指向整个内核统一的打开文件列表的，因而父进程刚才因为accept创建的已连接Socket也是一个文件描述符，同样也会被子进程获得。</p><p>接下来，子进程就可以通过这个已连接Socket和客户端进行互通了，当通信完毕之后，就可以退出进程，那父进程如何知道子进程干完了项目，要退出呢？还记得fork返回的时候，如果是整数就是父进程吗？这个整数就是子进程的ID，父进程可以通过这个ID查看子进程是否完成项目，是否需要退出。</p><h3>方式二：将项目转包给独立的项目组（多线程方式）</h3><p>上面这种方式你应该也能发现问题，如果每次接一个项目，都申请一个新公司，然后干完了，就注销掉这个公司，实在是太麻烦了。毕竟一个新公司要有新公司的资产，有新的办公家具，每次都买了再卖，不划算。</p><p>于是你应该想到了，我们可以使用<strong>线程</strong>。相比于进程来讲，这样要轻量级的多。如果创建进程相当于成立新公司，购买新办公家具，而创建线程，就相当于在同一个公司成立项目组。一个项目做完了，那这个项目组就可以解散，组成另外的项目组，办公家具可以共用。</p><p>在Linux下，通过pthread_create创建一个线程，也是调用do_fork。不同的是，虽然新的线程在task列表会新创建一项，但是很多资源，例如文件描述符列表、进程空间，还是共享的，只不过多了一个引用而已。</p><p><img src="https://static001.geekbang.org/resource/image/a3/64/a36537201678e08ac83e5410562d5f64.jpg?wh=4628*3401" alt=""></p><p>新的线程也可以通过已连接Socket处理请求，从而达到并发处理的目的。</p><p>上面基于进程或者线程模型的，其实还是有问题的。新到来一个TCP连接，就需要分配一个进程或者线程。一台机器无法创建很多进程或者线程。有个<strong>C10K</strong>，它的意思是一台机器要维护1万个连接，就要创建1万个进程或者线程，那么操作系统是无法承受的。如果维持1亿用户在线需要10万台服务器，成本也太高了。</p><p>其实C10K问题就是，你接项目接的太多了，如果每个项目都成立单独的项目组，就要招聘10万人，你肯定养不起，那怎么办呢？</p><h3>方式三：一个项目组支撑多个项目（IO多路复用，一个线程维护多个Socket）</h3><p>当然，一个项目组可以看多个项目了。这个时候，每个项目组都应该有个项目进度墙，将自己组看的项目列在那里，然后每天通过项目墙看每个项目的进度，一旦某个项目有了进展，就派人去盯一下。</p><p>由于Socket是文件描述符，因而某个线程盯的所有的Socket，都放在一个文件描述符集合fd_set中，这就是<strong>项目进度墙</strong>，然后调用select函数来监听文件描述符集合是否有变化。一旦有变化，就会依次查看每个文件描述符。那些发生变化的文件描述符在fd_set对应的位都设为1，表示Socket可读或者可写，从而可以进行读写操作，然后再调用select，接着盯着下一轮的变化。</p><h3>方式四：一个项目组支撑多个项目（IO多路复用，从“派人盯着”到“有事通知”）</h3><p>上面select函数还是有问题的，因为每次Socket所在的文件描述符集合中有Socket发生变化的时候，都需要通过轮询的方式，也就是需要将全部项目都过一遍的方式来查看进度，这大大影响了一个项目组能够支撑的最大的项目数量。因而使用select，能够同时盯的项目数量由FD_SETSIZE限制。</p><p>如果改成事件通知的方式，情况就会好很多，项目组不需要通过轮询挨个盯着这些项目，而是当项目进度发生变化的时候，主动通知项目组，然后项目组再根据项目进展情况做相应的操作。</p><p>能完成这件事情的函数叫epoll，它在内核中的实现不是通过轮询的方式，而是通过注册callback函数的方式，当某个文件描述符发送变化的时候，就会主动通知。<br>
<img src="https://static001.geekbang.org/resource/image/d6/b1/d6efc5c5ee8e48dae0323de380dcf6b1.jpg?wh=4410*2335" alt=""></p><p>如图所示，假设进程打开了Socket m, n, x等多个文件描述符，现在需要通过epoll来监听是否这些Socket都有事件发生。其中epoll_create创建一个epoll对象，也是一个文件，也对应一个文件描述符，同样也对应着打开文件列表中的一项。在这项里面有一个红黑树，在红黑树里，要保存这个epoll要监听的所有Socket。</p><p>当epoll_ctl添加一个Socket的时候，其实是加入这个红黑树，同时红黑树里面的节点指向一个结构，将这个结构挂在被监听的Socket的事件列表中。当一个Socket来了一个事件的时候，可以从这个列表中得到epoll对象，并调用call  back通知它。</p><p>这种通知方式使得监听的Socket数据增加的时候，效率不会大幅度降低，能够同时监听的Socket的数目也非常的多了。上限就为系统定义的、进程打开的最大文件描述符个数。因而，<strong>epoll被称为解决C10K问题的利器</strong>。</p><h2>小结</h2><p>好了，这一节就到这里了，我们来总结一下：</p><ul>
<li>
<p>你需要记住TCP和UDP的Socket的编程中，客户端和服务端都需要调用哪些函数；</p>
</li>
<li>
<p>写一个能够支撑大量连接的高并发的服务端不容易，需要多进程、多线程，而epoll机制能解决C10K问题。</p>
</li>
</ul><p>最后，给你留两个思考题：</p><ol>
<li>
<p>epoll是Linux上的函数，那你知道Windows上对应的机制是什么吗？如果想实现一个跨平台的程序，你知道应该怎么办吗？</p>
</li>
<li>
<p>自己写Socket还是挺复杂的，写个HTTP的应用可能简单一些。那你知道HTTP的工作机制吗？</p>
</li>
</ol><p>欢迎你留言和我讨论。趣谈网络协议，我们下期见！</p>
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
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83erQaN1AkcKQJRTWRop4fLpf2Z2nt1SUk9quVmw3YzIuNGAwnjGoPjQ3kC3XOAPSpFaeG1haeYfhBg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>time_qiao</span>
  </div>
  <div class="_2_QraFYR_0">刘老师看完这节，觉得您的linux功底实在太深厚了，这个系列做完能不能做个linux系列的</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-21 08:16:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/a6/6b/12d87713.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Jealone</span>
  </div>
  <div class="_2_QraFYR_0">Epoll和IOCP还是有本质区别的，IOCP是封装了IO操作的，而epoll只是一个事件通知机制，意味着IOCP的IO操作也可以由内核完成，因此IOCP算异步IO，而基于epoll的仍然是同步IO，于IOCP相对应的不是epoll而是AIO</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-15 09:59:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/29/72/76838c57.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>白杨</span>
  </div>
  <div class="_2_QraFYR_0">我个人觉得还可以讲的更本质一些。<br>不论是同步还是异步，阻塞还是非阻塞，都是用某种方法创建了某种结构，而不同的是，是谁在执行内部的方法，是谁写了标志，是谁触发了事件，是谁在操作io，是谁在等待，是谁在返回。<br>换句话说就是，本来我在做的事情，让你来做吧，本来你在做的事情，让我来做吧。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-27 19:14:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/47/66/29a9acec.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>挖坑的张师傅</span>
  </div>
  <div class="_2_QraFYR_0">第一个问题，windows 上是 iocp，如果想跨平台，要么使用兼容性较好的 select，要么对不同的平台使用不同的调用，kqueue iocp等。<br>第二个问题，HTTP 是构建在 tcp 协议之上的，用 telnet 可以轻松构造一个 HTTP 请求</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-15 08:01:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/81/0f/fce9d8ea.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>明子</span>
  </div>
  <div class="_2_QraFYR_0">这章看的很吃力，可能限于篇幅原因，很多细节都没讲清楚。比如，说需要两个socket（监听socket和已连接socket），但没有说为什么需要两个socket，我去查了《Unix 网络编程》才了解原因；而且需要说明的是，只有服务端需要两个socket。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 只讲了模式，细节足够一本书了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-09 10:31:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>起风了001</span>
  </div>
  <div class="_2_QraFYR_0">之前有专门学过epoll, 然后就特地看了一下socket和tcp连接的关系, 所以这一章看得比较清晰明白.哈哈, 技术真的是多学就能融会贯通的, 不过本章老师说的主要是linux C编程上的, 现在我用的golang写的话, 很多细节都被隐藏了, 不需要使用epoll, 直接Listen(), 然后Accept() , 有新的连接过来的话就会返回一个Conn, 然后使用golang的goroutine就可以开启一个协程处理这个连接, 一个程序可以开100万个都没压力, 所以也就没有epoll这样的概念了.</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: golang好东西</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-21 16:13:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/e8/4d/e7728500.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>自然而然</span>
  </div>
  <div class="_2_QraFYR_0">这篇写的很棒，很见功底</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-07-21 18:11:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/2b/ec/af6d0b10.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>caohuan</span>
  </div>
  <div class="_2_QraFYR_0">听二遍 才懂了一点，通讯里面 确实 内容丰富、技巧很多、设计巧妙。<br><br>本篇所得：1.一台机器有多个网卡，可以选择一个或多个网卡，发送和接受包资源;<br>TCP连接数量=客户端的IP数*客户端的端口数，客户端的最大ip为2**32，客户端的端口数最大为 2**16<br>，所以TCP连接总数最大为2**48;<br><br>2.Socket可以包括 监听Socket和已连接Socket;<br><br>3.TCP的Socket连接 核心为：客户端的connect--服务端的accept---客户端的write---服务端的read---服务端的write----客户端的read;<br>UDP的Socket连接 内容主要为：客户端send to---服务端recvform---服务端sendto--客户端的recvfrom。<br><br>4.一个服务器可以连接有很多客户端进行通讯：（1）多进程，一个进程一个通讯项目;（2）多线程，一个进程对应对个项目，一个线程负责一个通讯项目;（3）多线程，一个线程负责多个通讯项目，使用轮询监听和描述 通讯的记录;（4）多线程，一个线程负责多个通讯项目，使用 注册 主动通知的方法，使用epoll解决C10k问题。<br><br>回答老师的问题：问题1  linux有epoll，window有类似IOCP，跨平台就不清楚;问题二  http使用tcp协议，使用talnet规则构建。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-22 19:54:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/f9/61/0f2e4328.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>灵魂胖子</span>
  </div>
  <div class="_2_QraFYR_0">老师，那个UDP 协议的 Socket 程序函数调用过程中的客户端可以不用调用bind（）吧?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-21 14:42:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/07/8c/0d886dcc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>蚂蚁内推+v</span>
  </div>
  <div class="_2_QraFYR_0">你好，服务端监听socket和连接socket并不是同一个socket,那是同一个端口么，如果是的话，怎么区分收到的数据是哪个socket的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 端口是一个端口。这两个socket都属于同一个进程的。内核可以区分用哪个socket接收。代码里面区分就是。就像如果你自己写代码，你就可以让socket i接收而不是socket j。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-10 09:18:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/62/5e/ef349e39.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Observer</span>
  </div>
  <div class="_2_QraFYR_0">还的继续啃 Unix 网络编程</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对的，这是好书</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-13 14:07:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83eo3DrWeV7ZwRLXrRZg4V3ic1LQYdZ3u1oicDhqPic47vMguvf5QS69roTiaJrwDr5Re3Sy2UyHDWwmsTA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>大光头</span>
  </div>
  <div class="_2_QraFYR_0">windows上有一个叫iocp，但是它还是有点不一样，它是proactor，而epoll是reactor。我觉得iocp效率会更高，epoll通知应用去读取数据，处理完之后还要返回内核发送数据，而完成端口直接在内核态进行处理数据，最后告诉应用，数据已经处理完了。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-15 19:52:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/46/c0/106d98e7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Sam_Deep_Thinking</span>
  </div>
  <div class="_2_QraFYR_0">真心写的好。多谢作者。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-15 14:07:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/df/1e/cea897e8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>传说中的成大大</span>
  </div>
  <div class="_2_QraFYR_0">这篇文章，最突出的一点就是竟然把linux的epoll讲解的这么简单明了，windows下是用iocp完成端口！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-06 10:48:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/20/5d/69170b96.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>灰灰</span>
  </div>
  <div class="_2_QraFYR_0">老师，对socket这块一直理解比较抽象，什么情况下需要socket编程，我印象中从来没写过。盼复</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 很多地方啊，可能都是框架做好了，但是看框架的源代码，还是能看到的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-19 11:23:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/71/ee/31b19304.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小可爱</span>
  </div>
  <div class="_2_QraFYR_0">我有个问题就是，数据来了之后服务端怎么知道这个数据是属于哪个socket呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 端口</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-10 11:29:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTK9RytLsauRVYGjupDIaibibAK5iaicEicONrMFc0O3icAGf5mD1buxoQ2ePPn9YurFhRbuf3AR1qJDy0GQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>星文友</span>
  </div>
  <div class="_2_QraFYR_0">select由文件描述fd_set限制<br>epoll 由文件描述符数量限制 <br>两个差异很大吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 大，文件描述符更大，fdset小的多</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-16 10:57:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/4f/e7/7e51052a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>fumeck.com🍋🌴summer sky</span>
  </div>
  <div class="_2_QraFYR_0">每天上班路上阅读，很充实，对于菜鸟的我收益颇多</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-15 09:58:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/d1/15/7d47de48.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>咖啡猫口里的咖啡猫🐱</span>
  </div>
  <div class="_2_QraFYR_0">看完tcp&#47;ip三卷 结合linux操作，再看一遍文章，刚觉老师 linux内核功力深厚。感觉书第二卷实现跟现在内核版本有些😂区别</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-26 15:28:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/6f/63/abb7bfe3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>扬～</span>
  </div>
  <div class="_2_QraFYR_0">UDP的socket的调用过程中，为什么没有listen呢，要不应用程序怎么监听此端口？要不怎么把UDP的包发往监听此端口的应用程序呢？基于UDP的方式会不会因为连接数过多，造成服务器处理压力增大从而延迟增高呢？因为他们共用一个端口，而不是像TCP那样会复制一个套接字随机分配一个端口用于数据接受和发送。我不知道我说的对不对，希望老师能够解答一下困惑。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: udp没有状态，所以不存在监听这个状态，内存里没有保存连接的任何信息，不占用资源，所以不存在连接数过多的压力，顶多是把网卡打满</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-02 11:43:24</div>
  </div>
</div>
</div>
</li>
</ul>