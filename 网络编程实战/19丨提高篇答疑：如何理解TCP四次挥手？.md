<audio title="19丨提高篇答疑：如何理解TCP四次挥手？" src="https://static001.geekbang.org/resource/audio/59/c8/5945e61482c0d49f1bd93e1cae2584c8.mp3" controls="controls"></audio> 
<p>你好，我是盛延敏，这里是网络编程实战第19讲，欢迎回来。</p><p>这一篇文章是提高篇的答疑部分，也是提高篇的最后一篇文章。非常感谢大家的积极评论与留言，让每一篇文章的留言区都成为学习互动的好地方。在今天的内容里，我将针对大家的问题做一次集中回答，希望能帮助你解决前面碰到的一些问题。</p><p>这部分，我将采用Q&amp;A的形式来展开。</p><h2>如何理解TCP四次挥手？</h2><p>TCP建立一个连接需3次握手，而终止一个连接则需要四次挥手。四次挥手的整个过程是这样的：</p><p><img src="https://static001.geekbang.org/resource/image/b8/ea/b8911347d23251b6b0ca07c6ec03a1ea.png?wh=828*592" alt=""><br>
首先，一方应用程序调用close，我们称该方为主动关闭方，该端的TCP发送一个FIN包，表示需要关闭连接。之后主动关闭方进入FIN_WAIT_1状态。</p><p>接着，接收到这个FIN包的对端执行被动关闭。这个FIN由TCP协议栈处理，我们知道，TCP协议栈为FIN包插入一个文件结束符EOF到接收缓冲区中，应用程序可以通过read调用来感知这个FIN包。一定要注意，这个EOF会被放在<strong>已排队等候的其他已接收的数据之后</strong>，这就意味着接收端应用程序需要处理这种异常情况，因为EOF表示在该连接上再无额外数据到达。此时，被动关闭方进入CLOSE_WAIT状态。</p><p>接下来，被动关闭方将读到这个EOF，于是，应用程序也调用close关闭它的套接字，这导致它的TCP也发送一个FIN包。这样，被动关闭方将进入LAST_ACK状态。</p><!-- [[[read_end]]] --><p>最终，主动关闭方接收到对方的FIN包，并确认这个FIN包。主动关闭方进入TIME_WAIT状态，而接收到ACK的被动关闭方则进入CLOSED状态。经过2MSL时间之后，主动关闭方也进入CLOSED状态。</p><p>你可以看到，每个方向都需要一个FIN和一个ACK，因此通常被称为四次挥手。</p><p>当然，这中间使用shutdown，执行一端到另一端的半关闭也是可以的。</p><p>当套接字被关闭时，TCP为其所在端发送一个FIN包。在大多数情况下，这是由应用进程调用close而发生的，值得注意的是，一个进程无论是正常退出（exit或者main函数返回），还是非正常退出（比如，收到SIGKILL信号关闭，就是我们常常干的kill -9），所有该进程打开的描述符都会被系统关闭，这也导致TCP描述符对应的连接上发出一个FIN包。</p><p>无论是客户端还是服务器，任何一端都可以发起主动关闭。大多数真实情况是客户端执行主动关闭，你可能不会想到的是，HTTP/1.0却是由服务器发起主动关闭的。</p><h2>最大分组 MSL是TCP 分组在网络中存活的最长时间吗？</h2><p>MSL是任何IP数据报能够在因特网中存活的最长时间。其实它的实现不是靠计时器来完成的，在每个数据报里都包含有一个被称为TTL（time to live）的8位字段，它的最大值为255。TTL可译为“生存时间”，这个生存时间由源主机设置初始值，它表示的是一个IP数据报可以经过的最大跳跃数，每经过一个路由器，就相当于经过了一跳，它的值就减1，当此值减为0时，则所在的路由器会将其丢弃，同时发送ICMP报文通知源主机。RFC793中规定MSL的时间为2分钟，Linux实际设置为30秒。</p><h2>关于listen函数中参数backlog的释义问题</h2><p>我们该如何理解listen函数中的参数backlog？如果backlog表示的是未完成连接队列的大小，那么已完成连接的队列的大小有限制吗？如果都是已经建立连接的状态，那么并发取决于已完成连接的队列的大小吗？</p><p>backlog的值含义从来就没有被严格定义过。原先Linux实现中，backlog参数定义了该套接字对应的未完成连接队列的最大长度 （pending connections)。如果一个连接到达时，该队列已满，客户端将会接收一个ECONNREFUSED的错误信息，如果支持重传，该请求可能会被忽略，之后会进行一次重传。</p><p>从Linux 2.2开始，backlog的参数内核有了新的语义，它现在定义的是已完成连接队列的最大长度，表示的是已建立的连接（established connection），正在等待被接收（accept调用返回），而不是原先的未完成队列的最大长度。现在，未完成队列的最大长度值可以通过 /proc/sys/net/ipv4/tcp_max_syn_backlog完成修改，默认值为128。</p><p>至于已完成连接队列，如果声明的backlog参数比 /proc/sys/net/core/somaxconn的参数要大，那么就会使用我们声明的那个值。实际上，这个默认的值为128。注意在Linux 2.4.25之前，这个值是不可以修改的一个固定值，大小也是128。</p><p>设计良好的程序，在128固定值的情况下也是可以支持成千上万的并发连接的，这取决于I/O分发的效率，以及多线程程序的设计。在后面的性能篇里，我们的目标就是设计这样的程序。</p><h2>UDP连接和断开套接字的过程是怎样的？</h2><p>UDP连接套接字不是发起连接请求的过程，而是记录目的地址和端口到套接字的映射关系。</p><p>断开套接字则相反，将删除原来记录的映射关系。</p><h2>在UDP中不进行connect，为什么客户端会收到信息？</h2><p>有人说，如果按照我在文章中的说法，UDP只有connect才建立socket和IP地址的映射，那么如果不进行connect，收到信息后内核又如何把数据交给对应的socket？</p><p>这个问题非常有意思。我刚刚看到这个问题的时候，心里也在想，是啊，我是不是说错了？</p><p>其实呢，这对应了两个不同的API场景。</p><p>第一个场景就是我这里讨论的connect场景，在这个场景里，我们讨论的是ICMP报文和socket之间的定位。我们知道，ICMP报文发送的是一个不可达的信息，不可达的信息是通过<strong>目的地址和端口</strong>来区分的，如果没有connect操作，<strong>目的地址和端口</strong>就没有办法和socket套接字进行对应，所以，即使收到了ICMP报文，内核也没有办法通知到对应的应用程序，告诉它连接地址不可达。</p><p>那么为什么在不connect的情况下，我们的客户端又可以收到服务器回显的信息了？</p><p>这就涉及到了第二个场景，也就是报文发送的场景。注意服务器端程序，先通过recvfrom函数调用获取了客户端的地址和端口信息，这当然是可以的，因为UDP报文里面包含了这部分信息。然后我们看到服务器端又通过调用sendto函数，把客户端的地址和端口信息告诉了内核协议栈，可以肯定的是，之后发送的UDP报文就带上了<strong>客户端的地址和端口信息</strong>，通过客户端的地址和端口信息，可以找到对应的套接字和应用程序，完成数据的收发。</p><pre><code>//服务器端程序，先通过recvfrom函数调用获取了客户端的地址和端口信息
int n = recvfrom(socket_fd, message, MAXLINE, 0, (struct sockaddr *) &amp;client_addr, &amp;client_len);
message[n] = 0;
printf(&quot;received %d bytes: %s\n&quot;, n, message);

char send_line[MAXLINE];
sprintf(send_line, &quot;Hi, %s&quot;, message);

//服务器端程序调用send函数，把客户端的地址和端口信息告诉了内核
sendto(socket_fd, send_line, strlen(send_line), 0, (struct sockaddr *) &amp;client_addr, client_len);
</code></pre><p>从代码中可以看到，这里的connect的作用是记录<strong>客户端目的地址和端口–套接字</strong>的关系，而之所以能正确收到从服务器端发送的报文，那是因为系统已经记录了<strong>客户端源地址和端口–套接字</strong>的映射关系。</p><h2>我们是否可以对一个 UDP套接字进行多次connect的操作?</h2><p>我们知道，对于TCP套接字，connect只能调用一次。但是，对一个UDP套接字来说，进行多次connect操作是被允许的，这样主要有两个作用。</p><p>第一个作用是可以重新指定新的IP地址和端口号；第二个作用是可以断开一个已连接的套接字。为了断开一个已连接的UDP套接字，第二次调用connect时，调用方需要把套接字地址结构的地址族成员设置为AF_UNSPEC。</p><h2>第11讲中程序和时序图的解惑</h2><p>在11讲中，我们讲了关闭连接的几种方式，有同学对这一篇文章中的程序和时序图存在疑惑，并提出了下面几个问题：</p><ol>
<li>代码运行结果是先显示hi data1，之后才接收到标准输入的close，为什么时序图中画的是先close才接收到hi data1？</li>
<li>当一方主动close之后，另一方发送数据的时候收到RST。主动方缓冲区会把这个数据丢弃吗？这样的话，应用层应该读不到了吧？</li>
<li>代码中SIGPIPE的作用不是忽略吗？为什么服务器端会退出？</li>
<li>主动调用socket的那方关闭了写端，但是还没关闭读端，这时候socket再读到数据是不是就是RST？然后再SIGPIPE？如果是这样的话，为什么不一次性把读写全部关闭呢？</li>
</ol><p>我还是再仔细讲一下这个程序和时序图。</p><p>首先回答问题1。针对close这个例子，时序图里画的close表示的是客户端发起的close调用。</p><p>关于问题2，“Hi, data1”确实是不应该被接收到的，这个数据报即使发送出去也会收到RST回执，应用层是读不到的。</p><p>关于问题3中SIGPIPE的作用，事实上，默认的SIGPIPE忽略行为就是退出程序，什么也不做，当然，实际程序还是要做一些清理工作的。</p><p>问题4的理解是错误的。第二个例子也显示了，如果主动关闭的一方调用shutdown关闭，没有关闭读这一端，主动关闭的一方可以读到对端的数据，注意这个时候主动关闭连接的一方是在使用read方法进行读操作，而不是write写操作，不会有RST的发生，更不会有SIGPIPE的发生。</p><p><img src="https://static001.geekbang.org/resource/image/f2/9a/f283b804c7e33e25a900fedc8c36f09a.png?wh=4152*3256" alt=""></p><h2>总结</h2><p>以上就是提高篇中一些同学的疑问。我们常说，学问学问，有学才有问。我希望通过今天的答疑可以让你加深对文章的理解，为后面的模块做准备。</p><p>这篇文章之后，我们就将进入到专栏中最重要的部分，也就是性能篇和实战篇了，在性能篇和实战篇里，我们将会使用到之前学到的知识，逐渐打造一个高性能的网络程序框架，你，准备好了吗？</p><p>如果你觉得今天的答疑内容对你有所帮助，欢迎把它转发给你的朋友或者同事，一起交流一下。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/39/fa/a7edbc72.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>安排</span>
  </div>
  <div class="_2_QraFYR_0">MSL的值怎么和TTL对应的啊？比如MSL设置为30秒，那怎么计算出TTL的值呢？怎么保证一个报文在网络中真的存活不超过30秒？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复:  TTL与MSL是有关系的但不是简单的相等的关系，MSL要大于等于TTL。<br><br>设备在处理TTL的时候，是需要处理时间的，每次处理TTL时这个字段就都应该被减少，来反应花在处理报文上的时间。比如说处理了1秒减1，处理了2秒减2。这样就可以保证TTL为30的肯定活不过30秒。<br><br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-13 10:42:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_f8c379</span>
  </div>
  <div class="_2_QraFYR_0">signal(SIGPIPE, SIG_IGN); 实际上是忽略信号 而不是 按照默认的信号处理程序即退出</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 因为你给SIGPIPE信号设置了SIG_IGN，所以是忽略信号。我说的是如果没有设置，默认的SIGPIPE处理的方式即为退出。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-04 14:40:46</div>
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
  <div class="_2_QraFYR_0">MSL 是任何 IP 数据报能够在因特网中存活的最长时间。其实它的实现不是靠计时器来完成的，在每个数据报里都包含有一个被称为 TTL（time to live）的 8 位字段，它的最大值为 255。TTL 可译为“生存时间”，这个生存时间由源主机设置初始值，它表示的是一个 IP 数据报可以经过的最大跳跃数，每经过一个路由器，就相当于经过了一跳，它的值就减 1，当此值减为 0 时，则所在的路由器会将其丢弃，同时发送 ICMP 报文通知源主机<br><br>这是否意味着一个IP数据报不可能经过255个路由器？<br>请问一个IP数据报经过多少路由器，这个由谁决定？怎么决定？和网络距离距离有什么关系？<br><br>😅感觉老师的回答和问题，没有完全的对上？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: IP数据报经过多少路由器这个是无法确定的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-23 14:27:30</div>
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
  <div class="_2_QraFYR_0">请问老师，SO_REUSEADDR虽说是重用TIME_WAIT的socket，为什么不能作为TIME_WAIT的解决方案呢？本质上是解决了端口占用问题，而TIME_WAIT的主要弊端不就在与端口占用吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这是两个层面的东西，因为有了TIME_WAIT所以才导致不设置SO_REUSEADDR的服务端程序会出现&quot;Address already in use&quot;的错误，所以我们在服务端程序bind之前一定记得要设置SO_REUSEADDR。<br><br>但是TIME_WAIT本身还是很有价值的，TIME_WAIT 的引入是为了让 TCP 报文得以自然消失，同时为了让被动关闭方能够正常关闭。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-22 20:53:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/49/29/bbeccb9f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>风羽星泉</span>
  </div>
  <div class="_2_QraFYR_0">老师，我找到我提交的问题答案了。 使用 man listen 命令，可以找到下面这一句话：<br>If the backlog argument is greater than the value in &#47;proc&#47;sys&#47;net&#47;core&#47;somaxconn, then it is silently truncated to that value;   也就是说backlog 设置的值大于somaxconn，会被截断为somaxconn 的值。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-17 09:38:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/cf/22/5a483755.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小蛋壳</span>
  </div>
  <div class="_2_QraFYR_0">高性能的网络通信框架，是不是类似netty做的事？。那比如spring mvc或者其他任何应用程序框架其实底层都需要处理网络通讯这块。可以说知名的框架这块其实处理的都很好？ java应用，是tomcat处理网络请求还是spring来处理的？还有nginx</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: netty是Java的网络通信框架，其底层实现还是依赖类似epoll的事件分发机制的。<br><br>spring mvc这类的使用的Java网络编程框架来做的，tomcat也是类似的。<br><br>Nginx是使用类似epoll的机制来自己实现的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-13 08:04:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/48/19/14dd81d9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>铲铲队</span>
  </div>
  <div class="_2_QraFYR_0">至于已完成连接队列，如果声明的 backlog 参数比 &#47;proc&#47;sys&#47;net&#47;core&#47;somaxconn 的参数要大，那么就会使用我们声明的那个值<br>-----》老师，这里是不是有问题呢，原文是这样的： If  the  backlog  argument  is greater than the value in &#47;proc&#47;sys&#47;net&#47;core&#47;somaxconn, then it issilently truncated to that value<br>意思应该是设置的backlog值超过了最大值，则被截断为最大值把。<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 字面意思应该是你说的那种。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-23 11:37:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/49/29/bbeccb9f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>风羽星泉</span>
  </div>
  <div class="_2_QraFYR_0">老师，我修改了程序中的backlog为10，&#47;proc&#47;sys&#47;net&#47;core&#47;somaxconn没有变还是默认值128。测试程序同时发起800个连接请求，用netstat观察每次大概能建立10个连接，最后有大量连接报错“A connection attempt failed because the connected party did not properly respond after a period of time, or established connection failed because connected host has failed to respond.”；<br>当我修改程序中的backlog为1000时，最大可连接数并没有突破128，最后也是报上面那个错误。<br>是不是backlog设置的值只能小于内核somaxconn的值，如果比它大，则取内核设置的值。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 因为你设置太大也没有，系统有个默认值。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-17 09:30:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2c/96/98/89b96cda.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>三年二班邱小东</span>
  </div>
  <div class="_2_QraFYR_0">老师，为FIN包插入EOF的TCP协议栈是主动关闭方插入的，还是被动关闭方插入的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 主动方。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-25 11:35:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/sOuSC65kXWdWBAIIs6uXAD41Ed8Wo8tib81LLVOQJ2oK23TgPDy6x0PGmp7rXwLR3BHOicaKx1zib1DyfpCITK3dw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>GeekYanger</span>
  </div>
  <div class="_2_QraFYR_0">记录一下，TIME_WAIT本身还是很有价值的，TIME_WAIT 的引入是为了让 TCP 报文得以自然消失，同时为了让被动关闭方能够正常关闭。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-27 11:58:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/28/a5/4e/1c89bca4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>landing</span>
  </div>
  <div class="_2_QraFYR_0">&quot;注意服务器端程序，先通过 recvfrom 函数调用获取了客户端的地址和端口信息，这当然是可以的，因为 UDP 报文里面包含了这部分信息。&quot;这里不理解。如果解释了客户端收到服务端的数据是因为服务端第一次接受客户端数据时获得了客户端的地址，然后通过sendto告诉了内核，那么第一步内核为什么把数据给服务端套接字，这一步没有建立映射关系，内核应该也不知道啊</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 内核收到数据是通过服务端套接字来完成的，而要把服务端的数据发送给客户端，需要绑定服务端套接字和客户端地址-端口，这就是通过代码中recvform和sendto来完成的。<br><br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-17 15:27:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/21/54/85/ab5148ce.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>duckman</span>
  </div>
  <div class="_2_QraFYR_0">突然发现了<br><br>无论是TCP&#47;UDP, 建立连接之后, 服务端总是先read, 客户端总是先write。<br><br>所以，有了IO多路复用之后, 是不是 服务端&#47;客户端可以抛弃这个经典的对话模式，完全根据业务层面的需求, 决定是先write还是先read, 而且两个动作是独立的，互不影响。<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你的发现是我的程序例子的模板吧，实际上不一定是这样的。不过你对I&#47;O多路复用的理解倒是有另外一个角度，也算是一个层面的理解吧。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-10 17:02:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/20/dd/6e/8f6f79d2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>YUAN</span>
  </div>
  <div class="_2_QraFYR_0">使用udp时，客户端如何知道服务器端收到了它发送的请求？例如dns查询，客户端发送的请求如果丢失，那么客户端如何知道发生了报文丢失？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: UDP就是不知道的, 使用的时候我们需要容忍这样的丢失。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-03 12:08:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>宋菁</span>
  </div>
  <div class="_2_QraFYR_0">老师好，udp没绑定端口的话，操作系统动态分配端口，有可能会出现两个socket自动绑定到同一个端口的情况吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 为啥会发生呢？系统自动选择端口，它应该会默默记下来使用过的端口的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-03 07:11:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83epKJlW7sqts2ZbPuhMbseTAdvHWnrc4ficAeSZyKibkvn6qyxflPrkKKU3mH6XCNmYvDg11tB6y0pxg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>pc</span>
  </div>
  <div class="_2_QraFYR_0">看了前面的连接关闭、TCP对链路异常的感知，以及四次挥手，串起来后总感觉有点糊涂。看完这篇的几个问题，又稍微明白了一点。想着还是总结着提问一下：<br>1、首先ACK M+1 和 FIN N两次挥手不能合并吧？上学课本还有一些博客不是都有提到，两者之前被动关闭的一方还可以继续发送数据的。<br><br>2、所以TCP四次挥手简单总结为：A端发送FIN包、B端返回ACK；B端继续send data；B端发送FIN包、A端进入TIME_WAIT并发送ACK。<br>针对“被动关闭方继续send data”这点，回应到本文最后一个问题、以及TCP感知链路异常的几种情况———如果A端是close关闭，B端继续send data的时候就会收到RST、再write则触发SIGPIPE信号；如果A端是shutdown(1)关闭，才可以继续接收到B端的数据，B端也正常的继续send data。（就是文中最后的时序图）<br>这样的话，对于是不是可以理解成shutdown关闭的时候才是四次挥手的场景呢？反过来说，关闭的时候并不都是“标准的四次挥手”？<br><br>3、像kill -9这种暴力关闭程序，都应该类似close关闭吧？<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. 能不能合并我没有确切的答案，不过我觉得是可以优化的；<br>2.close和shutdown都是标准的四次挥手<br>3.我同意，不过这种是系统内核回收资源额外带来的结果。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-07 18:45:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/46/27/e0b34360.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>林子</span>
  </div>
  <div class="_2_QraFYR_0">针对UDP connect的问题有个疑问，按照上文说的是为了让内核记录目标ip端口到socket的映射关系，那如果源主机上有两个udp socket都connect同一个目标ip端口，这时如果出现icmp报文时内核是不是会通知这两个socket呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 好问题，我猜是不是会报错？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-08 22:21:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/50/02/cce1cf67.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>awmthink</span>
  </div>
  <div class="_2_QraFYR_0">老师，如果连接双方正好同时close conn fd，会怎么样呢，相当于FIN_WAIT_1时，收到了FIN，会不会直接跳过了FIN_WAIT_2，发送ACK后进入TIME_WAIT了呢。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这样的场景极其罕见，也很难模拟，而且TCP的状态其实是一个状态机驱动模式，肯定有一个先来后来的情况，进入先来的那个状态机驱动模式，就会按照那个模式走下去，而不是突然跳出来。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-08 20:17:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/7e/05/431d380f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>拂尘</span>
  </div>
  <div class="_2_QraFYR_0">老师，文章开头描述4次挥手，被动方读取数据到eof后，向主动方发fin的ack，主动方进入fin_wait_2，这里是不是缺失了？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 确实是没有写，我加上，让编辑改下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-22 15:24:19</div>
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
  <div class="_2_QraFYR_0">四次挥手的ACKM+1 和 FIN N为啥不能合并到一起？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 好问题，事实上为了节省带宽，确实有这样做的，这里为了讲述方便，将其分开。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-12 17:00:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/5f/83/bb728e53.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Douglas</span>
  </div>
  <div class="_2_QraFYR_0">老师好像并没有讲为啥是4次挥手</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 理解挥手的目的就自然理解为啥是4此了，因为每个方向都需要一个 FIN 和一个 ACK，通常被称为四次挥手。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-09 21:06:39</div>
  </div>
</div>
</div>
</li>
</ul>