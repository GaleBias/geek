<audio title="14丨UDP也可以是“已连接”？" src="https://static001.geekbang.org/resource/audio/fe/20/fe209d5ee5eee6cab978d3f2887c3920.mp3" controls="controls"></audio> 
<p>你好，我是盛延敏，这里是网络编程实战的第14讲，欢迎回来。</p><p>在前面的基础篇中，我们已经接触到了UDP数据报协议相关的知识，在我们的脑海里，已经深深印上了“<strong>UDP 等于无连接协议</strong>”的特性。那么看到这一讲的题目，你是不是觉得有点困惑？没关系，和我一起进入“已连接”的UDP的世界，回头再看这个标题，相信你就会恍然大悟。</p><h2>从一个例子开始</h2><p>我们先从一个客户端例子开始，在这个例子中，客户端在UDP套接字上调用connect函数，之后将标准输入的字符串发送到服务器端，并从服务器端接收处理后的报文。当然，向服务器端发送和接收报文是通过调用函数sendto和recvfrom来完成的。</p><pre><code>#include &quot;lib/common.h&quot;
# define    MAXLINE     4096

int main(int argc, char **argv) {
    if (argc != 2) {
        error(1, 0, &quot;usage: udpclient1 &lt;IPaddress&gt;&quot;);
    }

    int socket_fd;
    socket_fd = socket(AF_INET, SOCK_DGRAM, 0);

    struct sockaddr_in server_addr;
    bzero(&amp;server_addr, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(SERV_PORT);
    inet_pton(AF_INET, argv[1], &amp;server_addr.sin_addr);

    socklen_t server_len = sizeof(server_addr);

    if (connect(socket_fd, (struct sockaddr *) &amp;server_addr, server_len)) {
        error(1, errno, &quot;connect failed&quot;);
    }

    struct sockaddr *reply_addr;
    reply_addr = malloc(server_len);

    char send_line[MAXLINE], recv_line[MAXLINE + 1];
    socklen_t len;
    int n;

    while (fgets(send_line, MAXLINE, stdin) != NULL) {
        int i = strlen(send_line);
        if (send_line[i - 1] == '\n') {
            send_line[i - 1] = 0;
        }

        printf(&quot;now sending %s\n&quot;, send_line);
        size_t rt = sendto(socket_fd, send_line, strlen(send_line), 0, (struct sockaddr *) &amp;server_addr, server_len);
        if (rt &lt; 0) {
            error(1, errno, &quot;sendto failed&quot;);
        }
        printf(&quot;send bytes: %zu \n&quot;, rt);
        
        len = 0;
        recv_line[0] = 0;
        n = recvfrom(socket_fd, recv_line, MAXLINE, 0, reply_addr, &amp;len);
        if (n &lt; 0)
            error(1, errno, &quot;recvfrom failed&quot;);
        recv_line[n] = 0;
        fputs(recv_line, stdout);
        fputs(&quot;\n&quot;, stdout);
    }

    exit(0);
}
</code></pre><p>我对这个程序做一个简单的解释：</p><ul>
<li>9-10行创建了一个UDP套接字；</li>
<li>12-16行创建了一个IPv4地址，绑定到指定端口和IP；</li>
<li><strong>20-22行调用connect将UDP套接字和IPv4地址进行了“绑定”，这里connect函数的名称有点让人误解，其实可能更好的选择是叫做setpeername</strong>；</li>
<li>31-55行是程序的主体，读取标准输入字符串后，调用sendto发送给对端；之后调用recvfrom等待对端的响应，并把对端响应信息打印到标准输出。</li>
</ul><!-- [[[read_end]]] --><p>在没有开启服务端的情况下，我们运行一下这个程序：</p><pre><code>$ ./udpconnectclient 127.0.0.1
g1
now sending g1
send bytes: 2
recvfrom failed: Connection refused (111)
</code></pre><p>看到这里你会不会觉得很奇怪？不是说好UDP是“无连接”的协议吗？不是说好UDP客户端只会阻塞在recvfrom这样的调用上吗？怎么这里冒出一个“Connection refused”的错误呢？</p><p>别着急，下面就跟着我的思路慢慢去解开这个谜团。</p><h2>UDP connect的作用</h2><p>从前面的例子中，你会发现，我们可以对UDP套接字调用connect函数，但是和TCP connect调用引起TCP三次握手，建立TCP有效连接不同，UDP connect函数的调用，并不会引起和服务器目标端的网络交互，也就是说，并不会触发所谓的“握手”报文发送和应答。</p><p>那么对UDP套接字进行connect操作到底有什么意义呢？</p><p>其实上面的例子已经给出了答案，这主要是为了让应用程序能够接收“异步错误”的信息。</p><p>如果我们回想一下第6篇不调用connect操作的客户端程序，在服务器端不开启的情况下，客户端程序是不会报错的，程序只会阻塞在recvfrom上，等待返回（或者超时）。</p><p>在这里，我们通过对UDP套接字进行connect操作，将UDP套接字建立了“上下文”，该套接字和服务器端的地址和端口产生了联系，正是这种绑定关系给了操作系统内核必要的信息，能够将操作系统内核收到的信息和对应的套接字进行关联。</p><p>我们可以展开讨论一下。</p><p>事实上，当我们调用sendto或者send操作函数时，应用程序报文被发送，我们的应用程序返回，操作系统内核接管了该报文，之后操作系统开始尝试往对应的地址和端口发送，因为对应的地址和端口不可达，一个ICMP报文会返回给操作系统内核，该ICMP报文含有目的地址和端口等信息。</p><p>如果我们不进行connect操作，建立（UDP套接字——目的地址+端口）之间的映射关系，操作系统内核就没有办法把ICMP不可达的信息和UDP套接字进行关联，也就没有办法将ICMP信息通知给应用程序。</p><p>如果我们进行了connect操作，帮助操作系统内核从容建立了（UDP套接字——目的地址+端口）之间的映射关系，当收到一个ICMP不可达报文时，操作系统内核可以从映射表中找出是哪个UDP套接字拥有该目的地址和端口，别忘了套接字在操作系统内部是全局唯一的，当我们在该套接字上再次调用recvfrom或recv方法时，就可以收到操作系统内核返回的“Connection Refused”的信息。</p><h2>收发函数</h2><p>在对UDP进行connect之后，关于收发函数的使用，很多书籍是这样推荐的：</p><ul>
<li>使用send或write函数来发送，如果使用sendto需要把相关的to地址信息置零；</li>
<li>使用recv或read函数来接收，如果使用recvfrom需要把对应的from地址信息置零。</li>
</ul><p>其实不同的UNIX实现对此表现出来的行为不尽相同。</p><p>在我的Linux 4.4.0环境中，使用sendto和recvfrom，系统会自动忽略to和from信息。在我的macOS 10.13中，确实需要遵守这样的规定，使用sendto或recvfrom会得到一些奇怪的结果，切回send和recv后正常。</p><p>考虑到兼容性，我们也推荐这些常规做法。所以在接下来的程序中，我会使用这样的做法来实现。</p><h2>服务器端connect的例子</h2><p>一般来说，服务器端不会主动发起connect操作，因为一旦如此，服务器端就只能响应一个客户端了。不过，有时候也不排除这样的情形，一旦一个客户端和服务器端发送UDP报文之后，该服务器端就要服务于这个唯一的客户端。</p><p>一个类似的服务器端程序如下：</p><pre><code>#include &quot;lib/common.h&quot;

static int count;

static void recvfrom_int(int signo) {
    printf(&quot;\nreceived %d datagrams\n&quot;, count);
    exit(0);
}

int main(int argc, char **argv) {
    int socket_fd;
    socket_fd = socket(AF_INET, SOCK_DGRAM, 0);

    struct sockaddr_in server_addr;
    bzero(&amp;server_addr, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    server_addr.sin_port = htons(SERV_PORT);

    bind(socket_fd, (struct sockaddr *) &amp;server_addr, sizeof(server_addr));

    socklen_t client_len;
    char message[MAXLINE];
    message[0] = 0;
    count = 0;

    signal(SIGINT, recvfrom_int);

    struct sockaddr_in client_addr;
    client_len = sizeof(client_addr);

    int n = recvfrom(socket_fd, message, MAXLINE, 0, (struct sockaddr *) &amp;client_addr, &amp;client_len);
    if (n &lt; 0) {
        error(1, errno, &quot;recvfrom failed&quot;);
    }
    message[n] = 0;
    printf(&quot;received %d bytes: %s\n&quot;, n, message);

    if (connect(socket_fd, (struct sockaddr *) &amp;client_addr, client_len)) {
        error(1, errno, &quot;connect failed&quot;);
    }

    while (strncmp(message, &quot;goodbye&quot;, 7) != 0) {
        char send_line[MAXLINE];
        sprintf(send_line, &quot;Hi, %s&quot;, message);

        size_t rt = send(socket_fd, send_line, strlen(send_line), 0);
        if (rt &lt; 0) {
            error(1, errno, &quot;send failed &quot;);
        }
        printf(&quot;send bytes: %zu \n&quot;, rt);

        size_t rc = recv(socket_fd, message, MAXLINE, 0);
        if (rc &lt; 0) {
            error(1, errno, &quot;recv failed&quot;);
        }
        
        count++;
    }

    exit(0);
}
</code></pre><p>我对这个程序做下解释：</p><ul>
<li>11-12行创建UDP套接字；</li>
<li>14-18行创建IPv4地址，绑定到ANY和对应端口；</li>
<li>20行绑定UDP套接字和IPv4地址；</li>
<li>27行为该程序注册一个信号处理函数，以响应Ctrl+C信号量操作；</li>
<li>32-37行调用recvfrom等待客户端报文到达，并将客户端信息保持到client_addr中；</li>
<li><strong>39-41行调用connect操作，将UDP套接字和客户端client_addr进行绑定</strong>；</li>
<li>43-59行是程序的主体，对接收的信息进行重新处理，加上”Hi“前缀后发送给客户端，并持续不断地从客户端接收报文，该过程一直持续，直到客户端发送“goodbye”报文为止。</li>
</ul><p>注意这里所有收发函数都使用了send和recv。</p><p>接下来我们实现一个connect的客户端程序：</p><pre><code>#include &quot;lib/common.h&quot;
# define    MAXLINE     4096

int main(int argc, char **argv) {
    if (argc != 2) {
        error(1, 0, &quot;usage: udpclient3 &lt;IPaddress&gt;&quot;);
    }

    int socket_fd;
    socket_fd = socket(AF_INET, SOCK_DGRAM, 0);

    struct sockaddr_in server_addr;
    bzero(&amp;server_addr, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(SERV_PORT);
    inet_pton(AF_INET, argv[1], &amp;server_addr.sin_addr);

    socklen_t server_len = sizeof(server_addr);

    if (connect(socket_fd, (struct sockaddr *) &amp;server_addr, server_len)) {
        error(1, errno, &quot;connect failed&quot;);
    }

    char send_line[MAXLINE], recv_line[MAXLINE + 1];
    int n;

    while (fgets(send_line, MAXLINE, stdin) != NULL) {
        int i = strlen(send_line);
        if (send_line[i - 1] == '\n') {
            send_line[i - 1] = 0;
        }

        printf(&quot;now sending %s\n&quot;, send_line);
        size_t rt = send(socket_fd, send_line, strlen(send_line), 0);
        if (rt &lt; 0) {
            error(1, errno, &quot;send failed &quot;);
        }
        printf(&quot;send bytes: %zu \n&quot;, rt);

        recv_line[0] = 0;
        n = recv(socket_fd, recv_line, MAXLINE, 0);
        if (n &lt; 0)
            error(1, errno, &quot;recv failed&quot;);
        recv_line[n] = 0;
        fputs(recv_line, stdout);
        fputs(&quot;\n&quot;, stdout);
    }

    exit(0);
}
</code></pre><p>我对这个客户端程序做一下解读：</p><ul>
<li>9-10行创建了一个UDP套接字；</li>
<li>12-16行创建了一个IPv4地址，绑定到指定端口和IP；</li>
<li><strong>20-22行调用connect将UDP套接字和IPv4地址进行了“绑定”</strong>；</li>
<li>27-46行是程序的主体，读取标准输入字符串后，调用send发送给对端；之后调用recv等待对端的响应，并把对端响应信息打印到标准输出。</li>
</ul><p>注意这里所有收发函数也都使用了send和recv。</p><p>接下来，我们先启动服务器端程序，然后依次开启两个客户端，分别是客户端1、客户端2，并且让客户端1先发送UDP报文。</p><p>服务器端：</p><pre><code>$ ./udpconnectserver
received 2 bytes: g1
send bytes: 6
</code></pre><p>客户端1：</p><pre><code> ./udpconnectclient2 127.0.0.1
g1
now sending g1
send bytes: 2
Hi, g1
</code></pre><p>客户端2：</p><pre><code>./udpconnectclient2 127.0.0.1
g2
now sending g2
send bytes: 2
recv failed: Connection refused (111)
</code></pre><p>我们看到，客户端1先发送报文，服务端随之通过connect和客户端1进行了“绑定”，这样，客户端2从操作系统内核得到了ICMP的错误，该错误在recv函数中返回，显示了“Connection refused”的错误信息。</p><h2>性能考虑</h2><p>一般来说，客户端通过connect绑定服务端的地址和端口，对UDP而言，可以有一定程度的性能提升。</p><p>这是为什么呢？</p><p>因为如果不使用connect方式，每次发送报文都会需要这样的过程：</p><p>连接套接字→发送报文→断开套接字→连接套接字→发送报文→断开套接字 →………</p><p>而如果使用connect方式，就会变成下面这样：</p><p>连接套接字→发送报文→发送报文→……→最后断开套接字</p><p>我们知道，连接套接字是需要一定开销的，比如需要查找路由表信息。所以，UDP客户端程序通过connect可以获得一定的性能提升。</p><h2>总结</h2><p>在今天的内容里，我对UDP套接字调用connect方法进行了深入的分析。之所以对UDP使用connect，绑定本地地址和端口，是为了让我们的程序可以快速获取异步错误信息的通知，同时也可以获得一定性能上的提升。</p><h2>思考题</h2><p>在本讲的最后，按照惯例，给你留两个思考题：</p><ol>
<li>可以对一个UDP 套接字进行多次connect操作吗? 你不妨动手试试，看看结果。</li>
<li>如果想使用多播或广播，我们应该怎么去使用connect呢？</li>
</ol><p>欢迎你在评论区写下你的思考，也欢迎把这篇文章分享给你的朋友或者同事，一起交流一下。</p>
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
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83eqyicZYyW7ahaXgXUD8ZAS8x0t8jx5rYLhwbUCJiawRepKIZfsLdkxdQ9XQMo99c1UDibmNVfFnAqwPg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>程序水果宝</span>
  </div>
  <div class="_2_QraFYR_0">对于recvfrom函数，我们可以看成是TCP中accept函数和read函数的结合，前三个参数是read的参数，后两个参数是accept的参数。对于sendto函数，则可以看成是TCP中connect函数和send函数的结合，前三个参数是send的参数，后两个参数则是connect的参数。所以udp在发送和接收数据的过程中都会建立套接字连接，只不过每次调用sendto发送完数据后，内核都会将临时保存的对端地址数据删除掉，也就是断开套接字，从而就会出现老师所说的那个循环</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-02 22:56:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/5kv7IqibneNnMLqtWZQR5f1et8lJmoxiaU43Ttzz3zqW7QzBqMkib8GCtImKsms7PPbWmTB51xRnZQAnRPfA1wVaw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_63bb29</span>
  </div>
  <div class="_2_QraFYR_0">老师，面试过程中问道udp如何实现可靠性，这个怎么答呀。要求具体每部实现</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我能想到的：<br>1.udp可以增加消息编号；<br>2.对每个消息编号提供ACK，在udp应用层增加应答机制；<br>3.没有应答的增加重传机制<br>4.增加缓存，ACK完的才从缓存中清除</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-21 09:26:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/ea/e7/9ce305ec.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Sancho</span>
  </div>
  <div class="_2_QraFYR_0">老师，你好。我有两个疑问：<br>1.不进行connect操作，UDP套接字与服务端的地址和端口就没有产生关系，那recvfrom是怎么收到对应的报文呢？<br>2.UDP的connect操作，会引发内核的ICMP报文发送？如果不是，ICMP是在什么时机下发送的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1.是通过sendto函数来绑定服务端地址的，之后再通过recvfrom引用到之前的socket，这样收到的报文就是指定的服务地址和端口了；<br>2.不是connect导致ICMP报文，而是对应的地址和端口不可达时，一个 ICMP 报文会返回。connect只是将这个信息传递变得可能了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-09 08:08:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/55/e6/87197b10.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>GeekAmI</span>
  </div>
  <div class="_2_QraFYR_0">问题1：亲测可以；<br>问题2：可以参考https:&#47;&#47;yq.aliyun.com&#47;articles&#47;523036。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-22 15:36:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/1e/ae/a6d5e24a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>🐗Jinx</span>
  </div>
  <div class="_2_QraFYR_0">对于广播的话，先把广播的option打开。然后再 connect 255.255.255.255 对吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的。UDP的广播地址是固定的为255.255.255.255。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-20 08:40:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/b3/c5/7fc124e2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Liam</span>
  </div>
  <div class="_2_QraFYR_0">按照老师的说法，只有connect才建立socket和ip地址的映射；那么，如果不进行connect，收到信息后内核又是如何把数据交给对应的socket呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 在答疑篇里统一回复了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-02 08:47:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/f6/e3/e4bcd69e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>沉淀的梦想</span>
  </div>
  <div class="_2_QraFYR_0">还是不太理解为什么UDP的sendto方法会有一个&quot;连接&quot;过程的性能损耗，直接按照目标地址发过去不就可以了吗？我的理解是操作系统会先用ICMP协议探一探目标地址是否存在，然后再用UDP协议发送具体的数据，不知道理解的对不？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我不觉得会发ICMP来探一谈。ICMP是用的时候才触发的。<br><br>这里我想表达的是操作系统协议栈在每次sendto的时候都会需要一个地址初始化的过程，如果这个过程省略掉了，是可以得到一点点性能的提升的。当然，其实这个是没有那么大的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-04 19:38:35</div>
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
  <div class="_2_QraFYR_0">udp 连接套接字 这个是什么过程？ 断开套接字这又是什么过程呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 没有断开，这里都是一个系统调用，告诉了一些系统内核信息而已。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-02 15:00:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/07/71/9d4eead3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>德鲁小叔</span>
  </div>
  <div class="_2_QraFYR_0">发现在输入完goodbye之后，服务端执行exit,后面client再去请求的时候又会被阻塞而不是返回错误，是因connect是单次操作吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是因为之前发送成功了，所以没有ICMP不可达的报文收到，因而进入了阻塞状态。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-16 08:11:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/4d/54/9c214885.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>kylexy_0817</span>
  </div>
  <div class="_2_QraFYR_0">1、因为UDP调用connect并不会真正创建链接，所以多次调用都不会有问题。<br>2、connect到路由器中的广播地址</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-05 21:46:28</div>
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
  <div class="_2_QraFYR_0">connect 将一个socket绑定到一个udp的客户端进程，其他的udp客户端进程想要再次绑定该socket(4元组)发送数据的时候就会报错。所以connect起到了 &quot;声明式&quot;独占的作用?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 也可以这么说。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-07 17:30:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83ep1aHicNquR3ETTicbInlCpfawcDMB8ILYyzegVTubTgQ0w6icarsK7fglpZVr7VfiaJaQ0eokNAHVYLA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>一个戒</span>
  </div>
  <div class="_2_QraFYR_0">老师，请问第一个程序中，在没有开启服务端的情况下开启客户端，不会在第20行connect的时候就error报错了吗？为什么还能接收标准输入并send出去？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个是UDP哦，不是TCP。 通常在服务器端不开启的情况下，UDP客户端程序是不会报错的，程序只会阻塞在 recvfrom 上，等待返回（或者超时）。UDP的connect并不是真正的conncect操作，它只是给UDP 套接字建立了“上下文”。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-29 10:24:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJWFdKjyLOXtCzowmdCUFHezNlnux4NPWmYsqETjiaHNbnmb7xdzibDncZqP06nNbpN4AhmD76cpicfw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>fhs</span>
  </div>
  <div class="_2_QraFYR_0">请问一下，在客户端的代码中，在send之前进行了connect。测试发现，不启动服务端的情况下，使用使用客户端send的函数的返回值是输入的字节数，既然文章说到&quot;因为对应的地址和端口不可达，一个ICMP报文会返回给操作系统内核，该ICMP报文含有目的地址和端口等信息&quot;，那为什么send函数返回是成功，而recv就返回失败呢？send的时候不应该也能知道icmp报文返回错误而返回错误么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个要从函数设计的角度来说，在UDP通信中, sendto的目的是将报文通过网络传送给对端，并不考虑是否能发送成功，仅仅考虑的是把报文通过缓冲区发送出去；而recvfrom则是一个阻塞调用，它是需要知道是否成功的，包括超时，包括ICMP报文返回错误。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-10 17:07:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/26/97/40/2cd6180b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>钱佳慧</span>
  </div>
  <div class="_2_QraFYR_0">老师，对于问题一我有个疑问，udp能否同时connect多条连接呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-22 15:59:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/26/97/40/2cd6180b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>钱佳慧</span>
  </div>
  <div class="_2_QraFYR_0">udp的多播跟组播跟connect有关吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-22 12:58:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83er5SNsSoiaZw4Qzd2ctH4vtibHQordcLrYsX43oFZFloRTId0op617mcGlrvGx33U8ic2LTgdicoEFPvQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Frankey</span>
  </div>
  <div class="_2_QraFYR_0">连接套接字是需要一定开销的，比如需要查找路由表信息。<br>这个路由表信息是什么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 网络层 A-&gt;B B-&gt;C，这些是需要路由信息的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-23 17:34:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/9f/11/9c55033e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>MuteX</span>
  </div>
  <div class="_2_QraFYR_0">提一个细节，sendto&#47;recvfrom和send&#47;recv的返回值类型是ssize_t，也就是long，而文章例子中很多地方用的是size_t，也就是unsigned long，很显然会导致问题。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 疏漏了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-23 13:23:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83er4rbCWDxib3FHibYBouTwZqZBH6h5IgvjibEiaBv4Ceekib9SYg0peBBlFGu8hDuGvwjKp6LNznvEAibYw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>DonaldTrumpppppppppp</span>
  </div>
  <div class="_2_QraFYR_0">不调用connect时，有个评论问题老师你是这么答的。1.是通过sendto函数来绑定服务端地址的，之后再通过recvfrom引用到之前的socket，这样收到的报文就是指定的服务地址和端口了。  难道recvfrom不是收到后再填充from的地址的吗，还可以指定从某个服务端收数据？任何一个服务端只要知道了客户端的ip和端口就能发，客户端没有拒绝的权力吧</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这是对UDP而言的，通过sendto函数客户端指定了发送的服务端地址，再通过recvfrom就可以从之前指定的服务端地址来接收数据了。这里，没有说可以任意从某个服务端收数据的，而是从之前指定的服务端收数据的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-23 09:45:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/fa/20/0f06b080.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>凌空飞起的剪刀腿</span>
  </div>
  <div class="_2_QraFYR_0">还是不太理解为什么UDP的sendto方法会有一个&quot;连接&quot;过程的性能损耗，直接按照目标地址发过去不就可以了吗？我的理解是操作系统会先用ICMP协议探一探目标地址是否存在，然后再用UDP协议发送具体的数据，不知道理解的对不？<br>我也同意这个说法，可能客户端提前connect可以解决不用sendto以后，是否可以到达服务器，省的堵塞在recv函数里面了，同时使用connect可以提前判断服务器是否存在，至于是否会节省性能这需要抓包分析</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 好吧，大家都有这个疑问，我只是在说，每次调用sendto都需要进行地址初始化，其实这个是没有那么大的。我们就把性能损耗这个&quot;圆&quot;过去吧 😢</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-20 11:24:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKn2fx2UTaWgMl3fSOSicJEDOibbtYicHUVSG8JsA8j6Njibc9j3YVSvHtMZb2Z20l4NmjibiaSv8m7hz9w/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_de83f6</span>
  </div>
  <div class="_2_QraFYR_0">为什么我在第一个实例中，没有打开服务器，然后客户端connect并且send了之后，在recv的地方阻塞了，并没有收到connect失败的消息呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 看看你是不是没有权限run这个程序，或者你的本地socket路径不可以创建。总之是哪里出错了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-03 21:33:59</div>
  </div>
</div>
</div>
</li>
</ul>