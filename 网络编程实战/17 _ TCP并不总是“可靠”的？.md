<audio title="17 _ TCP并不总是“可靠”的？" src="https://static001.geekbang.org/resource/audio/ae/58/ae172aaf12d8df6378e52eee03aaa658.mp3" controls="controls"></audio> 
<p>你好，我是盛延敏，这里是网络编程实战第17讲，欢迎回来。</p><p>在前面一讲中，我们讲到如何理解TCP数据流的本质，进而引出了报文格式和解析。在这一讲里，我们讨论通过如何增强读写操作，以处理各种“不可靠”的场景。</p><h2>TCP是可靠的？</h2><p>你可能会认为，TCP是一种可靠的协议，这种可靠体现在端到端的通信上。这似乎给我们带来了一种错觉，从发送端来看，应用程序通过调用send函数发送的数据流总能可靠地到达接收端；而从接收端来看，总是可以把对端发送的数据流完整无损地传递给应用程序来处理。</p><p>事实上，如果我们对TCP传输环节进行详细的分析，你就会沮丧地发现，上述论断是不正确的。</p><p>前面我们已经了解，发送端通过调用send函数之后，数据流并没有马上通过网络传输出去，而是存储在套接字的发送缓冲区中，由网络协议栈决定何时发送、如何发送。当对应的数据发送给接收端，接收端回应ACK，存储在发送缓冲区的这部分数据就可以删除了，但是，发送端并无法获取对应数据流的ACK情况，也就是说，发送端没有办法判断对端的接收方是否已经接收发送的数据流，如果需要知道这部分信息，就必须在应用层自己添加处理逻辑，例如显式的报文确认机制。</p><p>从接收端来说，也没有办法保证ACK过的数据部分可以被应用程序处理，因为数据需要接收端程序从接收缓冲区中拷贝，可能出现的状况是，已经ACK的数据保存在接收端缓冲区中，接收端处理程序突然崩溃了，这部分数据就没有办法被应用程序继续处理。</p><!-- [[[read_end]]] --><p>你有没有发现，TCP协议实现并没有提供给上层应用程序过多的异常处理细节，或者说，TCP协议反映链路异常的能力偏弱，这其实是有原因的。要知道，TCP诞生之初，就是为美国国防部服务的，考虑到军事作战的实际需要，TCP不希望暴露更多的异常细节，而是能够以无人值守、自我恢复的方式运作。</p><p>TCP连接建立之后，能感知TCP链路的方式是有限的，一种是以read为核心的读操作，另一种是以write为核心的写操作。接下来，我们就看下如何通过读写操作来感知异常情况，以及对应的处理方式。</p><h2>故障模式总结</h2><p>在实际情景中，我们会碰到各种异常的情况。在这里我把这几种异常情况归结为两大类：</p><p><img src="https://static001.geekbang.org/resource/image/39/af/39b060fa90628db95fd33305dc6fc7af.png?wh=1466*490" alt=""><br>
第一类，是对端无FIN包发送出来的情况；第二类是对端有FIN包发送出来。而这两大类情况又可以根据应用程序的场景细分，接下来我们详细讨论。</p><h2>网络中断造成的对端无FIN包</h2><p>很多原因都会造成网络中断，在这种情况下，TCP程序并不能及时感知到异常信息。除非网络中的其他设备，如路由器发出一条ICMP报文，说明目的网络或主机不可达，这个时候通过read或write调用就会返回Unreachable的错误。</p><p>可惜大多数时候并不是如此，在没有ICMP报文的情况下，TCP程序并不能理解感应到连接异常。如果程序是阻塞在read调用上，那么很不幸，程序无法从异常中恢复。这显然是非常不合理的，不过，我们可以通过给read操作设置超时来解决，在接下来的第18讲中，我会讲到具体的方法。</p><p>如果程序先调用了write操作发送了一段数据流，接下来阻塞在read调用上，结果会非常不同。Linux系统的TCP协议栈会不断尝试将发送缓冲区的数据发送出去，大概在重传12次、合计时间约为9分钟之后，协议栈会标识该连接异常，这时，阻塞的read调用会返回一条TIMEOUT的错误信息。如果此时程序还执着地往这条连接写数据，写操作会立即失败，返回一个SIGPIPE信号给应用程序。</p><h2>系统崩溃造成的对端无FIN包</h2><p>当系统突然崩溃，如断电时，网络连接上来不及发出任何东西。这里和通过系统调用杀死应用程序非常不同的是，没有任何FIN包被发送出来。</p><p>这种情况和网络中断造成的结果非常类似，在没有ICMP报文的情况下，TCP程序只能通过read和write调用得到网络连接异常的信息，超时错误是一个常见的结果。</p><p>不过还有一种情况需要考虑，那就是系统在崩溃之后又重启，当重传的TCP分组到达重启后的系统，由于系统中没有该TCP分组对应的连接数据，系统会返回一个RST重置分节，TCP程序通过read或write调用可以分别对RST进行错误处理。</p><p>如果是阻塞的read调用，会立即返回一个错误，错误信息为连接重置（Connection Reset）。</p><p>如果是一次write操作，也会立即失败，应用程序会被返回一个SIGPIPE信号。</p><h2>对端有FIN包发出</h2><p>对端如果有FIN包发出，可能的场景是对端调用了close或shutdown显式地关闭了连接，也可能是对端应用程序崩溃，操作系统内核代为清理所发出的。从应用程序角度上看，无法区分是哪种情形。</p><p>阻塞的read操作在完成正常接收的数据读取之后，FIN包会通过返回一个EOF来完成通知，此时，read调用返回值为0。这里强调一点，收到FIN包之后read操作不会立即返回。你可以这样理解，收到FIN包相当于往接收缓冲区里放置了一个EOF符号，之前已经在接收缓冲区的有效数据不会受到影响。</p><p>为了展示这些特性，我分别编写了服务器端和客户端程序。</p><pre><code>//服务端程序
int main(int argc, char **argv) {
    int connfd;
    char buf[1024];

    connfd = tcp_server(SERV_PORT);

    for (;;) {
        int n = read(connfd, buf, 1024);
        if (n &lt; 0) {
            error(1, errno, &quot;error read&quot;);
        } else if (n == 0) {
            error(1, 0, &quot;client closed \n&quot;);
        }

        sleep(5);

        int write_nc = send(connfd, buf, n, 0);
        printf(&quot;send bytes: %zu \n&quot;, write_nc);
        if (write_nc &lt; 0) {
            error(1, errno, &quot;error write&quot;);
        }
    }

    exit(0);
}
</code></pre><p>服务端程序是一个简单的应答程序，在收到数据流之后回显给客户端，在此之前，休眠5秒，以便完成后面的实验验证。</p><p>客户端程序从标准输入读入，将读入的字符串传输给服务器端：</p><pre><code>//客户端程序
int main(int argc, char **argv) {
    if (argc != 2) {
        error(1, 0, &quot;usage: reliable_client01 &lt;IPaddress&gt;&quot;);
    }

    int socket_fd = tcp_client(argv[1], SERV_PORT);
    char buf[128];
    int len;
    int rc;

    while (fgets(buf, sizeof(buf), stdin) != NULL) {
        len = strlen(buf);
        rc = send(socket_fd, buf, len, 0);
        if (rc &lt; 0)
            error(1, errno, &quot;write failed&quot;);
        rc = read(socket_fd, buf, sizeof(buf));
        if (rc &lt; 0)
            error(1, errno, &quot;read failed&quot;);
        else if (rc == 0)
            error(1, 0, &quot;peer connection closed\n&quot;);
        else
            fputs(buf, stdout);
    }
    exit(0);
}
</code></pre><h3>read直接感知FIN包</h3><p>我们依次启动服务器端和客户端程序，在客户端输入good字符之后，迅速结束掉服务器端程序，这里需要赶在服务器端从睡眠中苏醒之前杀死服务器程序。</p><p>屏幕上打印出：peer connection closed。客户端程序正常退出。</p><pre><code>$./reliable_client01 127.0.0.1
$ good
$ peer connection closed
</code></pre><p>这说明客户端程序通过read调用，感知到了服务端发送的FIN包，于是正常退出了客户端程序。</p><p><img src="https://static001.geekbang.org/resource/image/b0/ec/b0922e1b1824f1e4735f2788eb3527ec.png?wh=636*466" alt=""><br>
注意如果我们的速度不够快，导致服务器端从睡眠中苏醒，并成功将报文发送出来后，客户端会正常显示，此时我们停留，等待标准输入。如果不继续通过read或write操作对套接字进行读写，是无法感知服务器端已经关闭套接字这个事实的。</p><h3>通过write产生RST，read调用感知RST</h3><p>这一次，我们仍然依次启动服务器端和客户端程序，在客户端输入bad字符之后，等待一段时间，直到客户端正确显示了服务端的回应“bad”字符之后，再杀死服务器程序。客户端再次输入bad2，这时屏幕上打印出”peer connection closed“。</p><p>这是这个案例的屏幕输出和时序图。</p><pre><code>$./reliable_client01 127.0.0.1
$bad
$bad
$bad2
$peer connection closed
</code></pre><p><img src="https://static001.geekbang.org/resource/image/a9/f2/a95d3b87a9a93421774d7aeade8efbf2.png?wh=1438*760" alt=""><br>
在很多书籍和文章中，对这个程序的解读是，收到FIN包的客户端继续合法地向服务器端发送数据，服务器端在无法定位该TCP连接信息的情况下，发送了RST信息，当程序调用read操作时，内核会将RST错误信息通知给应用程序。这是一个典型的write操作造成异常，再通过read操作来感知异常的样例。</p><p>不过，我在Linux 4.4内核上实验这个程序，多次的结果都是，内核正常将EOF信息通知给应用程序，而不是RST错误信息。</p><p>我又在Max OS 10.13.6上尝试这个程序，read操作可以返回RST异常信息。输出和时序图也已经给出。</p><pre><code>$./reliable_client01 127.0.0.1
$bad
$bad
$bad2
$read failed: Connection reset by peer (54)
</code></pre><h3>向一个已关闭连接连续写，最终导致SIGPIPE</h3><p>为了模拟这个过程，我对服务器端程序和客户端程序都做了如下修改。</p><pre><code>nt main(int argc, char **argv) {
    int connfd;
    char buf[1024];
    int time = 0;

    connfd = tcp_server(SERV_PORT);

    while (1) {
        int n = read(connfd, buf, 1024);
        if (n &lt; 0) {
            error(1, errno, &quot;error read&quot;);
        } else if (n == 0) {
            error(1, 0, &quot;client closed \n&quot;);
        }

        time++;
        fprintf(stdout, &quot;1K read for %d \n&quot;, time);
        usleep(1000);
    }

    exit(0);
}
</code></pre><p>服务器端每次读取1K数据后休眠1秒，以模拟处理数据的过程。</p><p>客户端程序在第8行注册了SIGPIPE的信号处理程序，在第14-22行客户端程序一直循环发送数据流。</p><pre><code>int main(int argc, char **argv) {
    if (argc != 2) {
        error(1, 0, &quot;usage: reliable_client02 &lt;IPaddress&gt;&quot;);
    }

    int socket_fd = tcp_client(argv[1], SERV_PORT);

    signal(SIGPIPE, SIG_IGN);

    char *msg = &quot;network programming&quot;;
    ssize_t n_written;

    int count = 10000000;
    while (count &gt; 0) {
        n_written = send(socket_fd, msg, strlen(msg), 0);
        fprintf(stdout, &quot;send into buffer %ld \n&quot;, n_written);
        if (n_written &lt;= 0) {
            error(1, errno, &quot;send error&quot;);
            return -1;
        }
        count--;
    }
    return 0;
}
</code></pre><p>如果在服务端读取数据并处理过程中，突然杀死服务器进程，我们会看到客<strong>户端很快也会退出</strong>，并在屏幕上打印出“Connection reset by peer”的提示。</p><pre><code>$./reliable_client02 127.0.0.1
$send into buffer 5917291
$send into buffer -1
$send: Connection reset by peer
</code></pre><p>这是因为服务端程序被杀死之后，操作系统内核会做一些清理的事情，为这个套接字发送一个FIN包，但是，客户端在收到FIN包之后，没有read操作，还是会继续往这个套接字写入数据。这是因为根据TCP协议，连接是双向的，收到对方的FIN包只意味着<strong>对方不会再发送任何消息</strong>。 在一个双方正常关闭的流程中，收到FIN包的一端将剩余数据发送给对面（通过一次或多次write），然后关闭套接字。</p><p>当数据到达服务器端时，操作系统内核发现这是一个指向关闭的套接字，会再次向客户端发送一个RST包，对于发送端而言如果此时再执行write操作，立即会返回一个RST错误信息。</p><p>你可以看到针对这个全过程的一张描述图，你可以参考这张图好好理解一下这个过程。</p><p><img src="https://static001.geekbang.org/resource/image/eb/42/ebf533a453573b85ff03a46103fc5b42.png?wh=1436*528" alt=""><br>
以上是在Linux 4.4内核上测试的结果。</p><p>在很多书籍和文章中，对这个实验的期望结果不是这样的。大部分的教程是这样说的：在第二次write操作时，由于服务器端无法查询到对应的TCP连接信息，于是发送了一个RST包给客户端，客户端第二次操作时，应用程序会收到一个SIGPIPE信号。如果不捕捉这个信号，应用程序会在毫无征兆的情况下直接退出。</p><p>我在Max OS 10.13.6上尝试这个程序，得到的结果确实如此。你可以看到屏幕显示和时序图。</p><pre><code>#send into buffer 19 
#send into buffer -1 
#send error: Broken pipe (32)
</code></pre><p>这说明，Linux4.4的实现和类BSD的实现已经非常不一样了。限于时间的关系，我没有仔细对比其他版本的Linux，还不清楚是新的内核特性，但有一点是可以肯定的，我们需要记得为SIGPIPE注册处理函数，通过write操作感知RST的错误信息，这样可以保证我们的应用程序在Linux 4.4和Mac OS上都能正常处理异常。</p><h2>总结</h2><p>在这一讲中，我们意识到TCP并不是那么“可靠”的。我把故障分为两大类，一类是对端无FIN包，需要通过巡检或超时来发现；另一类是对端有FIN包发出，需要通过增强read或write操作的异常处理，帮助我们发现此类异常。</p><h2>思考题</h2><p>和往常一样，给大家布置两道思考题。</p><p>第一道，你不妨在你的Linux系统中重新模拟一下今天文章里的实验，看看运行结果是否和我的一样。欢迎你把内核版本和结果贴在评论里。</p><p>第二道题是，如果服务器主机正常关闭，已连接的程序会发生什么呢？</p><p>你不妨思考一下这两道题，欢迎你在评论区写下你的模拟结果和思考，我会和你一起交流，也欢迎把这篇文章分享给你的朋友或者同事，一起交流一下。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/cb/61/b62d8a3b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张立华</span>
  </div>
  <div class="_2_QraFYR_0">这篇文章提醒我们的是，从代码的角度将注意三点：<br>1，程序启动的时候，忽略 SIGPIPE<br>2，read, write 要判断返回值，根据返回值知道socket断开了<br>3，应用层使用 心跳包，心跳包达到超时阀值，则认定socket断开了<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-20 15:55:12</div>
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
  <div class="_2_QraFYR_0">看到后面我好像理解了我上面那个提问,当崩溃重启过后是重新三次握手建立连接,创建新的套接字,只是在网络上传输的包,因为是通过ip地址和端口方式进行的寻址,所以新连接上去的客户端会接收到之前还没接收到的包,然后新连接的客户端没有这些包的tcp分组信息所以就会给服务器端(对端)发送一个RST</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 正解。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-09 17:16:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/9f/1d/ec173090.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>melon</span>
  </div>
  <div class="_2_QraFYR_0">最后的例子没有触发SIGPIPE，是因为老师例子设计的有点儿瑕疵。client 在不断的发送数据，server 则每次接受数据之后都 sleep 一会，也就导致接收速度小于发送速度，进而导致 server 终止的时候接收缓冲区还有数据没有被读取，server 终止触发 close 调用，close 调用时如果接收缓冲区有尚未被应用程序读取的数据时，不发 FIN 包，直接发 RST 包。client 每次发送数据之后 sleep 2秒，再试就会出SIGPIPE 了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 所以需要在客户端程序里加sleep就可以了？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-15 08:37:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/83/fb/621adceb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>linker</span>
  </div>
  <div class="_2_QraFYR_0">我的电脑，结果也是一样的，<br>第二个问题，服务器正常关闭，客户端应该是受到了fin包，read返回eOF,wirte返回rst</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 说明新版的内核在这些特性上有所改进。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-21 20:43:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/45/23/28311447.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>盘尼西林</span>
  </div>
  <div class="_2_QraFYR_0">没有理解 reset by peer 和 broken pipe 的区别。。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 一个是TCP RST的状态异常，一个是SIGPIPE 信号将进程进行回收。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-06 13:43:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/42/62/536aef06.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>tim</span>
  </div>
  <div class="_2_QraFYR_0">&gt;&gt;&quot;但是，发送端并无法获取对应数据流的 ACK 情况&quot;<br>对上面这段话不理解，TCP 的 ACK不是带着序号的吗？发送端根据这个序号能计算出是哪次发送的ACK。<br>哪位大牛能解释一下吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这里是从应用层报文数据角度出发来说的，因为数据是一个流，没有办法判断数据的哪些部分没对端收到。<br><br>从TCP角度来说，确实是通过ACK来感知TCP包的接收情况的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-09 13:14:14</div>
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
  <div class="_2_QraFYR_0">老师，你文章的案例默认fd都是阻塞的吧，如果是非阻塞的话，返回的n &lt; 0 不一定是错误啊</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 到现在为止，都是阻塞的，后面会切成非阻塞的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-13 08:38:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/12/ce/a8c8b5e8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Jason</span>
  </div>
  <div class="_2_QraFYR_0">error(1, 0, &quot;usage: reliable_client01 &lt;IPaddress&gt;&quot;);这个error函数，具体是什么作用？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 根据错误码打印错误信息，并且可以退出当前程序。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-06 00:20:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/64/86/f5a9403a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>多襄丸</span>
  </div>
  <div class="_2_QraFYR_0">老师 我提一个read直接感知FIN包的疑问哈:<br><br>我停留在 stdin这里 等我输入完之后，就能调用read感知到对端已经关闭了呀？ 是因为等到stdin之后，再感知是不是太晚了呀？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这就是为什么需要使用select、poll等事件分发机制，正确的解法是一旦有事件发生，比如这里read读到EOF，就应用直接去处理此类事件，而不是阻塞在这里等待用户的输入。好消息是，很快我们就会详细学习这部分内容了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-16 09:23:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/ff/3f/bbb8a88c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>徐凯</span>
  </div>
  <div class="_2_QraFYR_0">第二题  客户端--------服务器<br><br>1.  客户端发送FIN包，处于发送缓冲区的数据会逐一发送（可能通过一次或多次write操作发送），FIN包处于这段数据的末尾，当数据到达接收端的接收缓冲区时，FIN起到了一个结束符的作用，当接收端接收数据时遇到FIN包，read操作返回EOF通知应用层。然后接收端返回一个ACK表示对这次发送的确认。（此时客户端进入FIN_WAIT1，服务端进入CLOSE_WAIT状态）<br><br>2.  客户端接收到ACK之后，关闭自己的发送通道，客户端此时处于半关闭状态。等待服务器发送FIN包。<br><br>  (客户端进入FIN_WAIT2状态)<br><br>3.  服务端发送FIN包，同上类似处于发送缓冲区的内容会连同FIN包一起发过去，当客户端接收成功后同时将FIN解析为EOF信号使得上层调用返回。（客户端进入TIME_WAIT状态 服务端进入LAST_ACK状态）<br><br>4. 客户端等待2MSL的时间，在此期间向服务器发送ACK。如果丢包进行重传。如果服务器收到ACK后 服务器进入CLOSED状态 客户端也进入CLOSED状态。<br><br>5. 连接关闭<br>我想问一下  如果最后一次挥手一直丢包  在2MSL的时间内都没到  TCP会咋办  会重置计时器么 还是就不管了直接关闭呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我猜想是会直接关闭的，没有对ACK的ACK包。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-11 16:30:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/dd/60/eae432c6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>yusuf</span>
  </div>
  <div class="_2_QraFYR_0"># uname -a<br>Linux tst 3.10.0-957.21.3.el7.x86_64 #1 SMP Tue Jun 18 16:35:19 UTC 2019 x86_64 x86_64 x86_64 GNU&#47;Linux<br># <br># .&#47;reliable_client01 127.0.0.1<br>good<br>peer connection closed<br># <br># .&#47;reliable_client01 127.0.0.1<br>bad<br>bad<br>bad2<br>peer connection closed<br># <br># .&#47;reliable_client02 127.0.0.1<br>send into buffer 19 <br>send into buffer -1 <br>send error: Connection reset by peer (104)<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 基本和我的Linux下结果一致。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-09 10:58:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_573990</span>
  </div>
  <div class="_2_QraFYR_0">做实验时想抓包，输入<br>#sudo tcpdump host 127.0.0.1<br>然后分别开启server和client做实验，完成后得到结果<br>0 packet captured<br>请问老师这个抓包命令应该怎么写</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-30 19:13:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/28/03/9c/5a0b8825.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Bin Watson</span>
  </div>
  <div class="_2_QraFYR_0">在内核4.15.0-158-generic版本中，在服务器关闭后，客户端是返SIGPIPE错误。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 内核的版本，确实是会影响某些细微的行为。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-07 18:50:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/f2/73/1c7bceae.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>乔纳森</span>
  </div>
  <div class="_2_QraFYR_0">客户端和服务端keep-alive 的timeout 时间不同的情况下，有可能出现老师说的场景，服务端keep-alive 超时时间小于客户端，先发FIN 报文，在服务端发FIN 报文的同时，客户端也想服务端发送P 数据包，这个时候，服务端就会收到P数据报后，就会向客户端回复一个R包；</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-18 08:59:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/9d/ab/6589d91a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>林林</span>
  </div>
  <div class="_2_QraFYR_0">当对应的数据发送给接收端，接收端回应 ACK，存储在发送缓冲区的这部分数据就可以删除了，但是，发送端并无法获取对应数据流的 ACK 情况，也就是说，发送端没有办法判断对端的接收方是否已经接收发送的数据流，如果需要知道这部分信息，就必须在应用层自己添加处理逻辑，例如显式的报文确认机制<br><br><br>不是很明白，为什么发送端无法获取对应数据流的ACK情况？  不是收到对方回的ACK包了吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这里想要表达的是，从应用层协议出发，没有办法保证应用层数据流，被完美的处理过和应答过，如果要做到这一点，是需要应用层加以处理逻辑的。而TCP这一层，只能做到报文级别的接收和发送，而不是真正的程序处理。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-23 00:59:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>highfly029</span>
  </div>
  <div class="_2_QraFYR_0">老师好，请教个问题，对于网络中断导致的故障，我们的处理是服务端因为timeout而主动close连接，在timeout之前的这段时间内，服务端会不断的发送消息，如果保证这段时间的消息不丢失？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你都已经网络中断了，只能是通过客户端的response来确定服务端消息是不是发送出去了，肯定没办法保证消息不丢了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-09 17:28:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/d0/15/c5dc2b0d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>James</span>
  </div>
  <div class="_2_QraFYR_0">想问下接收端为什么接受不到ACK,你说从应用层的角度看，网络层保证可靠性就可以了，应用层还需要再次保证吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对于很多高可靠性的场景，考虑到各种复杂的条件，应用层是需要保证的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-16 10:35:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/c1/39/11904266.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Steiner</span>
  </div>
  <div class="_2_QraFYR_0">write触发SIGPIPE后，程序会自动退出，还是继续执行？？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果没有捕捉到这个信号，就会自动退出。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-29 22:52:50</div>
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
  <div class="_2_QraFYR_0">本系列的代码中send 和 write 都有用，搜了一下, 确实可以替换<br><br>There should be no difference. Quoting from man 2 send:<br><br>The only difference between send() and write() is the presence of flags. With zero flags parameter, send() is equivalent to write().<br><br>https:&#47;&#47;stackoverflow.com&#47;questions&#47;1100432&#47;performance-impact-of-using-write-instead-of-send-when-writing-to-a-socket</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 基本是等价的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-10 00:40:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/d5/a4/5dd93357.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Passion</span>
  </div>
  <div class="_2_QraFYR_0">老师 我的数据通过send函数写入内核发送缓冲区没问题， 我抓取发送的端口 发现与我写入的数据对不上 怎么定位  比如我写入缓冲区100sql  发出去的只找到80个sql</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以通过抓包来看</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-11 21:04:20</div>
  </div>
</div>
</div>
</li>
</ul>