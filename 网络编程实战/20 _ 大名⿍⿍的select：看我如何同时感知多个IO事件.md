<audio title="20 _ 大名⿍⿍的select：看我如何同时感知多个IO事件" src="https://static001.geekbang.org/resource/audio/56/f2/560bf3007f63c911ce286cf778379df2.mp3" controls="controls"></audio> 
<p>你好，我是盛延敏，这里是网络编程实战的第20讲，欢迎回来。</p><p>这一讲是性能篇的第一讲。在性能篇里，我们将把注意力放到如何设计高并发高性能的网络服务器程序上。我希望通过这一模块的学习，让你能够掌握多路复用、异步I/O、多线程等知识，从而可以写出支持并发10K以上的高性能网络服务器程序。</p><p>还等什么呢？让我们开始吧。</p><h2>什么是I/O多路复用</h2><p>在<a href="https://time.geekbang.org/column/article/126126">第11讲</a>中，我们设计了这样一个应用程序，该程序从标准输入接收数据输入，然后通过套接字发送出去，同时，该程序也通过套接字接收对方发送的数据流。</p><p>我们可以使用fgets方法等待标准输入，但是一旦这样做，就没有办法在套接字有数据的时候读出数据；我们也可以使用read方法等待套接字有数据返回，但是这样做，也没有办法在标准输入有数据的情况下，读入数据并发送给对方。</p><p>I/O多路复用的设计初衷就是解决这样的场景。我们可以把标准输入、套接字等都看做I/O的一路，多路复用的意思，就是在任何一路I/O有“事件”发生的情况下，通知应用程序去处理相应的I/O事件，这样我们的程序就变成了“多面手”，在同一时刻仿佛可以处理多个I/O事件。</p><p>像刚才的例子，使用I/O复用以后，如果标准输入有数据，立即从标准输入读入数据，通过套接字发送出去；如果套接字有数据可以读，立即可以读出数据。</p><!-- [[[read_end]]] --><p>select函数就是这样一种常见的I/O多路复用技术，我们将在后面继续讲解其他的多路复用技术。使用select函数，通知内核挂起进程，当一个或多个I/O事件发生后，控制权返还给应用程序，由应用程序进行I/O事件的处理。</p><p>这些I/O事件的类型非常多，比如：</p><ul>
<li>标准输入文件描述符准备好可以读。</li>
<li>监听套接字准备好，新的连接已经建立成功。</li>
<li>已连接套接字准备好可以写。</li>
<li>如果一个I/O事件等待超过了10秒，发生了超时事件。</li>
</ul><h2>select函数的使用方法</h2><p>select函数的使用方法有点复杂，我们先看一下它的声明：</p><pre><code>int select(int maxfd, fd_set *readset, fd_set *writeset, fd_set *exceptset, const struct timeval *timeout);

返回：若有就绪描述符则为其数目，若超时则为0，若出错则为-1
</code></pre><p>在这个函数中，maxfd表示的是待测试的描述符基数，它的值是待测试的最大描述符加1。比如现在的select待测试的描述符集合是{0,1,4}，那么maxfd就是5，为啥是5，而不是4呢? 我会在下面进行解释。</p><p>紧接着的是三个描述符集合，分别是读描述符集合readset、写描述符集合writeset和异常描述符集合exceptset，这三个分别通知内核，在哪些描述符上检测数据可以读，可以写和有异常发生。</p><p>那么如何设置这些描述符集合呢？以下的宏可以帮助到我们。</p><pre><code>void FD_ZERO(fd_set *fdset);　　　　　　
void FD_SET(int fd, fd_set *fdset);　　
void FD_CLR(int fd, fd_set *fdset);　　　
int  FD_ISSET(int fd, fd_set *fdset);
</code></pre><p>如果你刚刚入门，理解这些宏可能有些困难。没有关系，我们可以这样想象，下面一个向量代表了一个描述符集合，其中，这个向量的每个元素都是二进制数中的0或者1。</p><pre><code>a[maxfd-1], ..., a[1], a[0]
</code></pre><p>我们按照这样的思路来理解这些宏：</p><ul>
<li>FD_ZERO用来将这个向量的所有元素都设置成0；</li>
<li>FD_SET用来把对应套接字fd的元素，a[fd]设置成1；</li>
<li>FD_CLR用来把对应套接字fd的元素，a[fd]设置成0；</li>
<li>FD_ISSET对这个向量进行检测，判断出对应套接字的元素a[fd]是0还是1。</li>
</ul><p>其中0代表不需要处理，1代表需要处理。</p><p>怎么样，是不是感觉豁然开朗了？</p><p>实际上，很多系统是用一个整型数组来表示一个描述字集合的，一个32位的整型数可以表示32个描述字，例如第一个整型数表示0-31描述字，第二个整型数可以表示32-63描述字，以此类推。</p><p>这个时候再来理解为什么描述字集合{0,1,4}，对应的maxfd是5，而不是4，就比较方便了。</p><p>因为这个向量对应的是下面这样的：</p><pre><code>a[4],a[3],a[2],a[1],a[0]
</code></pre><p>待测试的描述符个数显然是5， 而不是4。</p><p>三个描述符集合中的每一个都可以设置成空，这样就表示不需要内核进行相关的检测。</p><p>最后一个参数是timeval结构体时间：</p><pre><code>struct timeval {
  long   tv_sec; /* seconds */
  long   tv_usec; /* microseconds */
};
</code></pre><p>这个参数设置成不同的值，会有不同的可能：</p><p>第一个可能是设置成空(NULL)，表示如果没有I/O事件发生，则select一直等待下去。</p><p>第二个可能是设置一个非零的值，这个表示等待固定的一段时间后从select阻塞调用中返回，这在<a href="https://time.geekbang.org/column/article/127900">第12讲</a>超时的例子里曾经使用过。</p><p>第三个可能是将tv_sec和tv_usec都设置成0，表示根本不等待，检测完毕立即返回。这种情况使用得比较少。</p><h2>程序例子</h2><p>下面是一个具体的程序例子，我们通过这个例子来理解select函数。</p><pre><code>int main(int argc, char **argv) {
    if (argc != 2) {
        error(1, 0, &quot;usage: select01 &lt;IPaddress&gt;&quot;);
    }
    int socket_fd = tcp_client(argv[1], SERV_PORT);

    char recv_line[MAXLINE], send_line[MAXLINE];
    int n;

    fd_set readmask;
    fd_set allreads;
    FD_ZERO(&amp;allreads);
    FD_SET(0, &amp;allreads);
    FD_SET(socket_fd, &amp;allreads);

    for (;;) {
        readmask = allreads;
        int rc = select(socket_fd + 1, &amp;readmask, NULL, NULL, NULL);

        if (rc &lt;= 0) {
            error(1, errno, &quot;select failed&quot;);
        }

        if (FD_ISSET(socket_fd, &amp;readmask)) {
            n = read(socket_fd, recv_line, MAXLINE);
            if (n &lt; 0) {
                error(1, errno, &quot;read error&quot;);
            } else if (n == 0) {
                error(1, 0, &quot;server terminated \n&quot;);
            }
            recv_line[n] = 0;
            fputs(recv_line, stdout);
            fputs(&quot;\n&quot;, stdout);
        }

        if (FD_ISSET(STDIN_FILENO, &amp;readmask)) {
            if (fgets(send_line, MAXLINE, stdin) != NULL) {
                int i = strlen(send_line);
                if (send_line[i - 1] == '\n') {
                    send_line[i - 1] = 0;
                }

                printf(&quot;now sending %s\n&quot;, send_line);
                size_t rt = write(socket_fd, send_line, strlen(send_line));
                if (rt &lt; 0) {
                    error(1, errno, &quot;write failed &quot;);
                }
                printf(&quot;send bytes: %zu \n&quot;, rt);
            }
        }
    }

}
</code></pre><p>程序的12行通过FD_ZERO初始化了一个描述符集合，这个描述符读集合是空的：</p><p><img src="https://static001.geekbang.org/resource/image/ce/68/cea07eee264c1abf69c04aacfae56c68.png?wh=628*192" alt=""><br>
接下来程序的第13和14行，分别使用FD_SET将描述符0，即标准输入，以及连接套接字描述符3设置为待检测：</p><p><img src="https://static001.geekbang.org/resource/image/71/f2/714f4fb84ab9afb39e51f6bcfc18def2.png?wh=640*200" alt=""><br>
接下来的16-51行是循环检测，这里我们没有阻塞在fgets或read调用，而是通过select来检测套接字描述字有数据可读，或者标准输入有数据可读。比如，当用户通过标准输入使得标准输入描述符可读时，返回的readmask的值为：</p><p><img src="https://static001.geekbang.org/resource/image/b9/bd/b90d1df438847d5e11d80485a23817bd.png?wh=632*194" alt=""><br>
这个时候select调用返回，可以使用FD_ISSET来判断哪个描述符准备好可读了。如上图所示，这个时候是标准输入可读，37-51行程序读入后发送给对端。</p><p>如果是连接描述字准备好可读了，第24行判断为真，使用read将套接字数据读出。</p><p>我们需要注意的是，这个程序的17-18行非常重要，初学者很容易在这里掉坑里去。</p><p>第17行是每次测试完之后，重新设置待测试的描述符集合。你可以看到上面的例子，在select测试之前的数据是{0,3}，select测试之后就变成了{0}。</p><p>这是因为select调用每次完成测试之后，内核都会修改描述符集合，通过修改完的描述符集合来和应用程序交互，应用程序使用FD_ISSET来对每个描述符进行判断，从而知道什么样的事件发生。</p><p>第18行则是使用socket_fd+1来表示待测试的描述符基数。切记需要+1。</p><h2>套接字描述符就绪条件</h2><p>当我们说select测试返回，某个套接字准备好可读，表示什么样的事件发生呢？</p><p>第一种情况是套接字接收缓冲区有数据可以读，如果我们使用read函数去执行读操作，肯定不会被阻塞，而是会直接读到这部分数据。</p><p>第二种情况是对方发送了FIN，使用read函数执行读操作，不会被阻塞，直接返回0。</p><p>第三种情况是针对一个监听套接字而言的，有已经完成的连接建立，此时使用accept函数去执行不会阻塞，直接返回已经完成的连接。</p><p>第四种情况是套接字有错误待处理，使用read函数去执行读操作，不阻塞，且返回-1。</p><p>总结成一句话就是，内核通知我们套接字有数据可以读了，使用read函数不会阻塞。</p><p>不知道你是不是和我一样，刚开始理解某个套接字可写的时候，会有一个错觉，总是从应用程序角度出发去理解套接字可写，我开始是这样想的，当应用程序完成相应的计算，有数据准备发送给对端了，可以往套接字写，对应的就是套接字可写。</p><p>其实这个理解是非常不正确的，select检测套接字可写，<strong>完全是基于套接字本身的特性来说</strong>的，具体来说有以下几种情况。</p><p>第一种是套接字发送缓冲区足够大，如果我们使用阻塞套接字进行write操作，将不会被阻塞，直接返回。</p><p>第二种是连接的写半边已经关闭，如果继续进行写操作将会产生SIGPIPE信号。</p><p>第三种是套接字上有错误待处理，使用write函数去执行写操作，不阻塞，且返回-1。</p><p>总结成一句话就是，内核通知我们套接字可以往里写了，使用write函数就不会阻塞。</p><h2>总结</h2><p>今天我讲了select函数的使用。select函数提供了最基本的I/O多路复用方法，在使用select时，我们需要建立两个重要的认识：</p><ul>
<li>描述符基数是当前最大描述符+1；</li>
<li>每次select调用完成之后，记得要重置待测试集合。</li>
</ul><h2>思考题</h2><p>和往常一样，给你布置两道思考题：</p><p>第一道， select可以对诸如UNIX管道(pipe)这样的描述字进行检测么？如果可以，检测的就绪条件是什么呢？</p><p>第二道，根据我们前面的描述，一个描述符集合哪些描述符被设置为1，需要进行检测是完全可以知道的，你认为select函数里一定需要传入描述字基数这个值么？请你分析一下这样设计的目的又是什么呢？</p><p>欢迎你在评论区写下你的思考，也欢迎把这篇文章分享给你的朋友或者同事，一起交流一下。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/7e/05/431d380f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>拂尘</span>
  </div>
  <div class="_2_QraFYR_0">我一直很好奇，为啥说select函数对fd有1024的限制，找了点资料共勉：<br>首先，man select，搜索FD_SETSIZE会看到如下的内容<br>An fd_set is a fixed size buffer. Executing FD_CLR() or FD_SET() with a value of fd that is negative or is equal to or larger than FD_SETSIZE will result in undefined behavior. Moreover, POSIX requires fd to be a valid file descriptor.<br>其中最关键的是FD_SETSIZE，是在bitmap位图运算的时候会受到他的影响<br>其次，sys&#47;select.h头文件有如下定义：<br>#define FD_SETSIZE __FD_SETSIZE<br>typesizes.h头文件有如下定义：<br>#define __FD_SETSIZE 1024<br><br>由此，终于看到了1024的准确限制。<br><br>同时man里也说明了一个限制，不是0-1023的fd会导致未定义的行为。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞，为大家找到了原始的出处，证明我不是在瞎BB，哈哈。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-29 23:39:02</div>
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
  <div class="_2_QraFYR_0">第一道：可以，就绪条件是有数据可读(检测可读事件)。是否可以监测可写事件不太清楚，没有实验过。<br><br>第二道：不一定需要传入，那样的话内核中for循环需要遍历整个集合，效率低。传入基数可以减小遍历范围，提高效率。<br><br>当然，api既然设计成这样了，那肯定需要传入一个数了。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-23 00:20:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/0b/a7/6ef32187.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Keep-Moving</span>
  </div>
  <div class="_2_QraFYR_0">allreads = {0, 3};<br>老师，这一步是怎么实现的？没看出来<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 逐个解释一下： <br>1.FD_ZERO(&amp;allreads);  <br>所有的位置设置为0；<br>  <br>2. FD_SET(0, &amp;allreads);<br>将描述字0的对应位置设置为1；<br>  <br>3.FD_SET(socket_fd, &amp;allreads);<br>将监听套接字的对应位置设置为1。<br><br>这样就得到了allreads = {0, 3}。<br><br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-23 11:17:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/f0/61/68462a07.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>无名</span>
  </div>
  <div class="_2_QraFYR_0">对于套接字可写状态中说的：套接字发送缓冲区足够大，怎么样算足够大呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 实际上，只要有一个字节可以被写入，就是状态可写的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-04 13:25:55</div>
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
  <div class="_2_QraFYR_0">1：I&#47;O 多路复用的设计初衷就是解决这样的场景，把标准输入、套接字等都看做 I&#47;O 的一路，多路复用的意思，就是在任何一路 I&#47;O 有“事件”发生的情况下，通知应用程序去处理相应的 I&#47;O 事件，这样我们的程序就变成了“多面手”，在同一时刻仿佛可以处理多个 I&#47;O 事件。<br>2：select 函数就是这样一种常见的 I&#47;O 多路复用技术，使用 select 函数，通知内核挂起进程，当一个或多个 I&#47;O 事件发生后，控制权返还给应用程序，由应用程序进行 I&#47;O 事件的处理。<br><br>int select(int maxfd, fd_set *readset, fd_set *writeset, fd_set *exceptset, const struct timeval *timeout);<br><br>返回：若有就绪描述符则为其数目，若超时则为 0，若出错则为 -1<br><br>在这个函数中，maxfd 表示的是待测试的描述符基数，它的值是待测试的最大描述符加 1。<br>紧接着的是三个描述符集合，分别是读描述符集合 readset、写描述符集合 writeset 和异常描述符集合 exceptset，这三个分别通知内核，在哪些描述符上检测数据可以读，可以写和有异常发生。<br>三个描述符集合中的每一个都可以设置成空，这样就表示不需要内核进行相关的检测。<br>timeout设置成不同的值，会有不同的可能：<br>第一个可能是设置成空 (NULL)，表示如果没有 I&#47;O 事件发生，则 select 一直等待下去。<br>第二个可能是设置一个非零的值，这个表示等待固定的一段时间后从 select 阻塞调用中返回。<br>第三个可能是将 tv_sec 和 tv_usec 都设置成 0，表示根本不等待，检测完毕立即返回。这种情况使用得比较少。<br><br>3：内核通知我们套接字有数据可以读了，使用 read 函数不会阻塞。<br>内核通知我们套接字可以往里写了，使用 write 函数就不会阻塞。<br><br>读了几遍，感觉还是没有抓住核心，所以，就将文中的要点摘录下来。<br>对IO多路复用的大概理解是，通过select函数去监听一组文件描述符，如果有事件就绪就交给应用程序去做对应的处理。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 总结得已经很到位了呀</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-24 12:03:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/f0/61/68462a07.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>无名</span>
  </div>
  <div class="_2_QraFYR_0">size_t rt = write(socket_fd, send_line, strlen(send_line));<br>if (rt &lt; 0) {<br>     error(1, errno, &quot;write failed &quot;);<br> }<br>这个代码中有错吧，应该将size_t改为sszie_t，size_t为unsigned long，这样错误-1被转换了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，感谢指出。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-27 11:33:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/93/85/f5d9474c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>乔丹</span>
  </div>
  <div class="_2_QraFYR_0">老师，两个疑问：<br>1. 为什么socket_fd一定是3呢？ <br>2. 如果socket_fd = 2000, 那么传入select函数的值就是2001了， 这样不是大于1024了吗？<br>这个点我没有想通。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1.这里是举一个例子，因为0,1,2分别是标准输入，标准输出和标准错误，3是接下来的第一个常见描述字。<br><br>2.select确实不能支持大于1024的描述字。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-20 22:31:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/0e/ed/1c662e93.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>莫珣</span>
  </div>
  <div class="_2_QraFYR_0">我有些疑问，select的FD数组大小默认是1024，但是Linux的文件描述符大小一定不是1024，假设现在使用ulimit将一个进程可以打开的文件数设置成了65535，那么大于1024的文件描述符怎么加到FD数组中去呢，如果按照文本里说的，文件描述符代表数组下标的话不就加不进去了？<br><br>第二个问题，套接字有两个属性，接收低水位线和发送低水位线，当接收缓冲区中待接收的字节数大于接收低水位线，一个可读事件产生，那么如果永远都不能达到接收低水位线呢？<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 第一个问题，我理解是加不进去的，你不妨设计一个程序验证一下。<br><br>第二个问题，首先，这个值是可以调整的，我记得默认值即使1个byte，也就是说有数据就可以感知到；第二，如果一直达不到接收watermark，我理解不是一个正常的网络交互过程，正常的网络交互肯定是像流一样，不断有数据接收。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-23 08:46:42</div>
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
  <div class="_2_QraFYR_0">老师，哪种场景下需要多路复用　“写描述符”　呢？ 什么时候能写应用程序不知道吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 写描述符是当发送套接字缓冲区有空间时，要知道，应用程序不是什么时刻都可以不断网发送套接字缓冲区打收据，这样会把缓冲区打爆，所以多路复用写的意思就是告诉应用程序什么时候应该往发送套接字缓冲区打数据。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-16 20:42:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/31/17/02fc18b1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>麻雀</span>
  </div>
  <div class="_2_QraFYR_0">您好，<br>第一，想问下select是不是能够在处理数据的同时继续轮询（监听）是否有新的套接字来到，它的内部是不是多线程呢？因为accept就是因为单线程在处理数据时，不能对这段时间内到来的套接字进行监听。 <br>第二，FD_SET它是一个unsigned long数组，那么它怎么实现Bitmap，只是对数组的每个元素例如fd_set[10]对文件描述符为10的套接字来数据的时候设置为1吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 第一，是可以的。这个机制是操作系统实现的，你可以把操作系统理解成一个&quot;巨大&quot;的无限循环处理器，无论是有数据需要读写，还是有新的套接字连接达到，这个巨大的无限循环处理器都是可以快速感知到(通过各种软硬件机制，比如中断)，这样你就可以明白，它的内部并不是多线程实现的。<br><br>第二，你的理解是正确的，就是对每个位来设置0或者1。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-30 09:52:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/e6/3a/382cf024.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>rongyefeng</span>
  </div>
  <div class="_2_QraFYR_0">“第一种是套接字发送缓冲区足够大，如果我们使用非阻塞套接字进行 write 操作，将不会被阻塞，直接返回。”<br>老师，请问这里是不是应该写成“如果我们使用阻塞套接字进行write操作......”才对？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果从套接字特性来说，确实是阻塞套接字，已经提交勘误。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-19 21:26:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/2e/f4/1e4d6941.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>打奥特曼的小怪兽</span>
  </div>
  <div class="_2_QraFYR_0">关于 FD_SET() 函数，debug看了下内存结构，{0,3} 如果设置了，实际上存储的是 2^0 + 2^3 = 9,并不会像图示的在每个位置上设置1。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我的意思就是00001001，在bit位上设置为1, 转换为10进制就是9。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-06 14:49:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/40/aa/49bbb007.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>нáпの゛</span>
  </div>
  <div class="_2_QraFYR_0">第一道题，理解管道也是文件，往管道输入数据和输出数据对应可读可写的就绪条件。<br>第二道题，我理解fd_set本身是数组，如果不传入描述字基数，无法得知fd_set的具体大小，应该是无法进行遍历操作的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 基本正确哦。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-01 11:03:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/4f/a8/fa10583f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>imsunv</span>
  </div>
  <div class="_2_QraFYR_0">内核通知我们套接字可以往里写了，使用 write 函数就不会阻塞 。<br>那么如果写的内容超过了 缓冲区的大小，会阻塞么</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不会阻塞，write函数会返回实际写入缓冲区的字节大小。实际上的策略就是&quot;尽最大可能&quot;写入。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-09 22:04:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/95/12/33028a3f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小仙女</span>
  </div>
  <div class="_2_QraFYR_0">int select(int maxfd, fd_set *readset, fd_set *writeset, fd_set *exceptset, const struct timeval *timeout);<br>这里的fd_set 是什么结构<br><br>0:标准输入<br>1：标准输出<br>2：标准错误<br>3：socket<br><br>是这样吗？？<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你罗列的是文件描述符的种类，这个是没错的。不过fd_set是通过mask位来表示描述字的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-23 12:02:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/a5/f0/8648c464.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Joker</span>
  </div>
  <div class="_2_QraFYR_0">小明原来只在一个家书店里等着，后来发现等着无聊，回家，然后在去书店等；后来发现别的书店，索性就好几家一起问，问了这个去下一家，看看哪家书到了，就先买哪一家的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 哈哈，这也可以。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-17 02:23:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/af/00/9b49f42b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>skye</span>
  </div>
  <div class="_2_QraFYR_0">用select做多路复用，如果不用多线程，其中一路阻塞或者死锁了，那其它路就无法处理了，所以单线程处理的前提时没有阻塞和死锁，这样理解对吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我明白你想表达的是select之后处理事件的时候可能会阻塞，导致select不到其他事件，这点理解是对的。<br><br>至于单线程处理是不是一定没有阻塞(死锁我不太明白这里指的是具体什么情况)，我倒觉得不一定，当然，非阻塞效果可能更好一些。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-08 21:40:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/f2/e2/f48d094a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我也曾是少年</span>
  </div>
  <div class="_2_QraFYR_0">老师，我看了一部分开源代码，golang的，我发现大多数有名的项目他们并发写套接字的时候，都是用一个阻塞对列，既向一个没有容量的channel中写，只有接收端接了，发送端才会继续往下面走，我觉得别人这么做肯定是有原因的，但是我摸不透，所以将这问题定位到并发写套接字上，不知老师对这问题怎么看</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-20 00:24:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/41/ec/d0e2bfa4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Navelwort、</span>
  </div>
  <div class="_2_QraFYR_0">如果select 检测到可写事件，但是缓冲区还不够大，不能完成应用层数据的全部拷贝，如果是阻塞类套接字，那write函数还是会阻塞吧？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-03 16:42:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/50/35/079d04c8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>向东</span>
  </div>
  <div class="_2_QraFYR_0">32位整数，那么该数组的第一个元素对应于描述字0~31，第二个元素对应于描述字32~63，依此类推。 没读懂，解答一下？多谢🙏</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 32位整数，一共有32个bit位，每个bit位可以表示两种状态，0或者1，如果开启检测就将bit设置为1，否则设置为0。像下面这样：<br>00000000 00000000 00000000 10010010<br><br>这个32bit分别表示了描述字7，4和1设置为1，其他的设置为0。这里表示的对应描述字0-31。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-23 16:38:06</div>
  </div>
</div>
</div>
</li>
</ul>