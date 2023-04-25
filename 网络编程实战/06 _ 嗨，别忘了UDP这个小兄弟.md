<audio title="06 _ 嗨，别忘了UDP这个小兄弟" src="https://static001.geekbang.org/resource/audio/12/02/1264e068377b6ef9955d5d0774fb0902.mp3" controls="controls"></audio> 
<p>你好，我是盛延敏，这里是网络编程实战第6讲，欢迎回来。</p><p>前面几讲我们讲述了TCP方面的编程知识，这一讲我们来讲讲UDP方面的编程知识。</p><p>如果说TCP是网络协议的“大哥”，那么UDP可以说是“小兄弟”。这个小兄弟和大哥比，有什么差异呢？</p><p>首先，UDP是一种“数据报”协议，而TCP是一种面向连接的“数据流”协议。</p><p>TCP可以用日常生活中打电话的场景打比方，前面也多次用到了这样的例子。在这个例子中，拨打号码、接通电话、开始交流，分别对应了TCP的三次握手和报文传送。一旦双方的连接建立，那么双方对话时，一定知道彼此是谁。这个时候我们就说，这种对话是有上下文的。</p><p>同样的，我们也可以给UDP找一个类似的例子，这个例子就是邮寄明信片。在这个例子中，发信方在明信片中填上了接收方的地址和邮编，投递到邮局的邮筒之后，就可以不管了。发信方也可以给这个接收方再邮寄第二张、第三张，甚至是第四张明信片，但是这几张明信片之间是没有任何关系的，他们的到达顺序也是不保证的，有可能最后寄出的第四张明信片最先到达接收者的手中，因为没有序号，接收者也不知道这是第四张寄出的明信片；而且，即使接收方没有收到明信片，也没有办法重新邮寄一遍该明信片。</p><!-- [[[read_end]]] --><p>这两个简单的例子，道出了UDP和TCP之间最大的区别。</p><p>TCP是一个面向连接的协议，TCP在IP报文的基础上，增加了诸如重传、确认、有序传输、拥塞控制等能力，通信的双方是在一个确定的上下文中工作的。</p><p>而UDP则不同，UDP没有这样一个确定的上下文，它是一个不可靠的通信协议，没有重传和确认，没有有序控制，也没有拥塞控制。我们可以简单地理解为，在IP报文的基础上，UDP增加的能力有限。</p><p>UDP不保证报文的有效传递，不保证报文的有序，也就是说使用UDP的时候，我们需要做好丢包、重传、报文组装等工作。</p><p>既然如此，为什么我们还要使用UDP协议呢？</p><p>答案很简单，因为UDP比较简单，适合的场景还是比较多的，我们常见的DNS服务，SNMP服务都是基于UDP协议的，这些场景对时延、丢包都不是特别敏感。另外多人通信的场景，如聊天室、多人游戏等，也都会使用到UDP协议。</p><h2>UDP编程</h2><p>UDP和TCP编程非常不同，下面这张图是UDP程序设计时的主要过程。</p><p><img src="https://static001.geekbang.org/resource/image/84/30/8416f0055bedce10a3c7d0416cc1f430.png?wh=4034*2631" alt=""><br>
我们看到服务器端创建UDP 套接字之后，绑定到本地端口，调用recvfrom函数等待客户端的报文发送；客户端创建套接字之后，调用sendto函数往目标地址和端口发送UDP报文，然后客户端和服务器端进入互相应答过程。</p><p>recvfrom和sendto是UDP用来接收和发送报文的两个主要函数：</p><pre><code>#include &lt;sys/socket.h&gt;

ssize_t recvfrom(int sockfd, void *buff, size_t nbytes, int flags, 
　　　　　　　　　　struct sockaddr *from, socklen_t *addrlen); 

ssize_t sendto(int sockfd, const void *buff, size_t nbytes, int flags,
                const struct sockaddr *to, socklen_t addrlen); 
</code></pre><p>我们先来看一下recvfrom函数。</p><p>sockfd、buff和nbytes是前三个参数。sockfd是本地创建的套接字描述符，buff指向本地的缓存，nbytes表示最大接收数据字节。</p><p>第四个参数flags是和I/O相关的参数，这里我们还用不到，设置为0。</p><p>后面两个参数from和addrlen，实际上是返回对端发送方的地址和端口等信息，这和TCP非常不一样，TCP是通过accept函数拿到的描述字信息来决定对端的信息。另外UDP报文每次接收都会获取对端的信息，也就是说报文和报文之间是没有上下文的。</p><p>函数的返回值告诉我们实际接收的字节数。</p><p>接下来看一下sendto函数。</p><p>sendto函数中的前三个参数为sockfd、buff和nbytes。sockfd是本地创建的套接字描述符，buff指向发送的缓存，nbytes表示发送字节数。第四个参数flags依旧设置为0。</p><p>后面两个参数to和addrlen，表示发送的对端地址和端口等信息。</p><p>函数的返回值告诉我们实际发送的字节数。</p><p>我们知道， TCP的发送和接收每次都是在一个上下文中，类似这样的过程：</p><p>A连接上: 接收→发送→接收→发送→…</p><p>B连接上: 接收→发送→接收→发送→ …</p><p>而UDP的每次接收和发送都是一个独立的上下文，类似这样：</p><p>接收A→发送A→接收B→发送B →接收C→发送C→ …</p><h2>UDP服务端例子</h2><p>我们先来看一个UDP服务器端的例子：</p><pre><code>#include &quot;lib/common.h&quot;

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
    count = 0;

    signal(SIGINT, recvfrom_int);

    struct sockaddr_in client_addr;
    client_len = sizeof(client_addr);
    for (;;) {
        int n = recvfrom(socket_fd, message, MAXLINE, 0, (struct sockaddr *) &amp;client_addr, &amp;client_len);
        message[n] = 0;
        printf(&quot;received %d bytes: %s\n&quot;, n, message);

        char send_line[MAXLINE];
        sprintf(send_line, &quot;Hi, %s&quot;, message);

        sendto(socket_fd, send_line, strlen(send_line), 0, (struct sockaddr *) &amp;client_addr, client_len);

        count++;
    }

}
</code></pre><p>程序的12～13行，首先创建一个套接字，注意这里的套接字类型是“SOCK_DGRAM”，表示的是UDP数据报。</p><p>15～21行和TCP服务器端类似，绑定数据报套接字到本地的一个端口上。</p><p>27行为该服务器创建了一个信号处理函数，以便在响应“Ctrl+C”退出时，打印出收到的报文总数。</p><p>31～42行是该服务器端的主体，通过调用recvfrom函数获取客户端发送的报文，之后我们对收到的报文进行重新改造，加上“Hi”的前缀，再通过sendto函数发送给客户端对端。</p><h2>UDP客户端例子</h2><p>接下来我们再来构建一个对应的UDP客户端。在这个例子中，从标准输入中读取输入的字符串后，发送给服务端，并且把服务端经过处理的报文打印到标准输出上。</p><pre><code>#include &quot;lib/common.h&quot;

# define    MAXLINE     4096

int main(int argc, char **argv) {
    if (argc != 2) {
        error(1, 0, &quot;usage: udpclient &lt;IPaddress&gt;&quot;);
    }
    
    int socket_fd;
    socket_fd = socket(AF_INET, SOCK_DGRAM, 0);

    struct sockaddr_in server_addr;
    bzero(&amp;server_addr, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(SERV_PORT);
    inet_pton(AF_INET, argv[1], &amp;server_addr.sin_addr);

    socklen_t server_len = sizeof(server_addr);

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
            error(1, errno, &quot;send failed &quot;);
        }
        printf(&quot;send bytes: %zu \n&quot;, rt);

        len = 0;
        n = recvfrom(socket_fd, recv_line, MAXLINE, 0, reply_addr, &amp;len);
        if (n &lt; 0)
            error(1, errno, &quot;recvfrom failed&quot;);
        recv_line[n] = 0;
        fputs(recv_line, stdout);
        fputs(&quot;\n&quot;, stdout);
    }

    exit(0);
}
</code></pre><p>10～11行创建一个类型为“SOCK_DGRAM”的套接字。</p><p>13～17行，初始化目标服务器的地址和端口。</p><p>28～51行为程序主体，从标准输入中读取的字符进行处理后，调用sendto函数发送给目标服务器端，然后再次调用recvfrom函数接收目标服务器发送过来的新报文，并将其打印到标准输出上。</p><p>为了让你更好地理解UDP和TCP之间的差别，我们模拟一下UDP的三种运行场景，你不妨思考一下这三种场景的结果和TCP的到底有什么不同？</p><h2>场景一：只运行客户端</h2><p>如果我们只运行客户端，程序会一直阻塞在recvfrom上。</p><pre><code>$ ./udpclient 127.0.0.1
1
now sending g1
send bytes: 2
&lt;阻塞在这里&gt;
</code></pre><p>还记得TCP程序吗？如果不开启服务端，TCP客户端的connect函数会直接返回“Connection refused”报错信息。而在UDP程序里，则会一直阻塞在这里。</p><h2>场景二：先开启服务端，再开启客户端</h2><p>在这个场景里，我们先开启服务端在端口侦听，然后再开启客户端：</p><pre><code>$./udpserver
received 2 bytes: g1
received 2 bytes: g2
</code></pre><pre><code>$./udpclient 127.0.0.1
g1
now sending g1
send bytes: 2
Hi, g1
g2
now sending g2
send bytes: 2
Hi, g2
</code></pre><p>我们在客户端一次输入g1、g2，服务器端在屏幕上打印出收到的字符，并且可以看到，我们的客户端也收到了服务端的回应：“Hi,g1”和“Hi,g2”。</p><h2>场景三: 开启服务端，再一次开启两个客户端</h2><p>这个实验中，在服务端开启之后，依次开启两个客户端，并发送报文。</p><p>服务端：</p><pre><code>$./udpserver
received 2 bytes: g1
received 2 bytes: g2
received 2 bytes: g3
received 2 bytes: g4
</code></pre><p>第一个客户端：</p><pre><code>$./udpclient 127.0.0.1
now sending g1
send bytes: 2
Hi, g1
g3
now sending g3
send bytes: 2
Hi, g3
</code></pre><p>第二个客户端：</p><pre><code>$./udpclient 127.0.0.1
now sending g2
send bytes: 2
Hi, g2
g4
now sending g4
send bytes: 2
Hi, g4
</code></pre><p>我们看到，两个客户端发送的报文，依次都被服务端收到，并且客户端也可以收到服务端处理之后的报文。</p><p>如果我们此时把服务器端进程杀死，就可以看到信号函数在进程退出之前，打印出服务器端接收到的报文个数。</p><pre><code>$ ./udpserver
received 2 bytes: g1
received 2 bytes: g2
received 2 bytes: g3
received 2 bytes: g4
^C
received 4 datagrams
</code></pre><p>之后，我们再重启服务器端进程，并使用客户端1和客户端2继续发送新的报文，我们可以看到和TCP非常不同的结果。</p><p>以下就是服务器端的输出，服务器端重启后可以继续收到客户端的报文，这在TCP里是不可以的，TCP断联之后必须重新连接才可以发送报文信息。但是UDP报文的“无连接”的特点，可以在UDP服务器重启之后，继续进行报文的发送，这就是UDP报文“无上下文”的最好说明。</p><pre><code>$ ./udpserver
received 2 bytes: g1
received 2 bytes: g2
received 2 bytes: g3
received 2 bytes: g4
^C
received 4 datagrams
$ ./udpserver
received 2 bytes: g5
received 2 bytes: g6
</code></pre><p>第一个客户端：</p><pre><code>$./udpclient 127.0.0.1
now sending g1
send bytes: 2
Hi, g1
g3
now sending g3
send bytes: 2
Hi, g3
g5
now sending g5
send bytes: 2
Hi, g5
</code></pre><p>第二个客户端：</p><pre><code>$./udpclient 127.0.0.1
now sending g2
send bytes: 2
Hi, g2
g4
now sending g4
send bytes: 2
Hi, g4
g6
now sending g6
send bytes: 2
Hi, g6
</code></pre><h2>总结</h2><p>在这一讲里，我介绍了UDP程序的例子，我们需要重点关注以下两点：</p><ul>
<li>UDP是无连接的数据报程序，和TCP不同，不需要三次握手建立一条连接。</li>
<li>UDP程序通过recvfrom和sendto函数直接收发数据报报文。</li>
</ul><h2>思考题</h2><p>最后给你留两个思考题吧。在第一个场景中，recvfrom一直处于阻塞状态中，这是非常不合理的，你觉得这种情形应该怎么处理呢？另外，既然UDP是请求-应答模式的，那么请求中的UDP报文最大可以是多大呢？</p><p>欢迎你在评论区写下你的思考，我会和你一起讨论。也欢迎把这篇文章分享给你的朋友或者同事，一起讨论一下UDP这个协议。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4d/fd/0aa0e39f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>许童童</span>
  </div>
  <div class="_2_QraFYR_0">第一个场景中，可以添加超时时间做处理，当然你也可以自己实现一个复杂的请求-确认模式，那这样就跟TCP类似了，HTTP&#47;3就是这样做的。<br><br>用UDP协议发送时，用sendto函数最大能发送数据的长度为：65535- IP头(20) - UDP头(8)＝65507字节。用sendto函数发送数据时，如果发送数据长度大于该值，则函数会返回错误。  <br>由于IP有最大MTU，因此，<br>UDP 包的大小应该是 1500 - IP头(20) - UDP头(8) = 1472(Bytes)<br>TCP 包的大小应该是 1500 - IP头(20) - TCP头(20) = 1460 (Bytes)</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-14 11:44:46</div>
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
  <div class="_2_QraFYR_0">老师，UDP被IP层分包发送后，对端如何保证UDP包整个组合的？比如用UDP发送3000字节，假设拆分2个MTU发送，后一个先到服务端，前一个后到服务端，那应用层接收的时候，UDP怎么组装的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 很简单，1-1500为一个seq=1的包，1501-3000为seq=2的包，根据sequence组装就可以了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-30 09:13:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/23/74/b81c9f8c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>何赫赫</span>
  </div>
  <div class="_2_QraFYR_0">老师，UDP没有发送缓冲区这个概念吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 实际上不存在UDP发送缓冲区，因为发往UDP发送缓冲区的包只要超过一定阈值(值很小)就可以发往对端。所以我们一般认为UDP是没有发送缓冲区的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-09 21:59:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/ed/86/5855aaa4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>刘明</span>
  </div>
  <div class="_2_QraFYR_0">1、一直阻塞会导致程序无法正常退出，可以使用接收超时、IO多路复用的超时机制。<br><br>2、IP和UDP头中都有16bit的长度字段，最长65535字节，去掉头部长度得到UDP数据净荷长度：65535-20-8=65507字节。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-14 08:31:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/1e/74/636ea0f3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>你好</span>
  </div>
  <div class="_2_QraFYR_0">多人聊天室使用UDP，消息发出后怎么保证消息可以被收到呀，UDP不是不可靠传输嘛，中间丢了消息咋办呀</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 第一，丢了就丢了，反正UDP就是不可靠的；<br><br>第二，给每条消息加个sequence，收到后再确认，一段时间内没收到，就重发。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-30 13:41:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/81/e6/6cafed37.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>旅途</span>
  </div>
  <div class="_2_QraFYR_0">第一个思考题 另起一个线程进行recvfrom</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我的本意是使用超时或者非阻塞来解</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-29 01:28:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/ca/e3/447aff89.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>记事本</span>
  </div>
  <div class="_2_QraFYR_0">recv from会导致进程一直阻塞，可以通过setsocketopt接口设置TIMEOUT  超时后退出.</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-14 08:37:40</div>
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
  <div class="_2_QraFYR_0">udp之所以sendto要目的地址是因为他是非连接的，否则不知道将数据发给谁，recvfrom要发送方地址也是因为udp是非连接的，有了from内核就可以判定将数据上传给谁（应用进程）。是这样吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 基本是这样的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-03 13:51:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/a9/8f/a998a456.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>范龙dragon</span>
  </div>
  <div class="_2_QraFYR_0">客户端代码的29行sendline数组之前没有初始化数组元素为0，直接用strlen应该会有问题吧，strlen不是以0作为结束标志吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 好问题，fgets函数会&quot;默默&quot;的给我们加上\0</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-18 23:43:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/16/27/38a32199.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Godblesser</span>
  </div>
  <div class="_2_QraFYR_0">recvfrom函数可以设置超时，当限定时间内未接收到服务器端回复，打印超时消息并退出</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-14 08:40:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/6c/a8/1922a0f5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>郑祖煌</span>
  </div>
  <div class="_2_QraFYR_0">1.可以用非阻塞状态来做，或者用select 用户复用的弄。就是不阻塞，而是一定的时间内去轮询看有没有包到达。或者说当服务器有包达到事件通知客户端，客户端再去服务器那边读取数据。<br>2. 用UDP协议发送时，用sendto函数最大能发送数据的长度为：如果是本地套接字的话就是655356。如果是非本地套接字而是走网络端的话<br>由于IP有最大MTU，因此，<br>UDP 包的大小应该是 1500 - IP头(20) - UDP头(8) = 1472(Bytes)<br>TCP 包的大小应该是 1500 - IP头(20) - TCP头(20) = 1460 (Bytes)</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-12 17:26:53</div>
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
  <div class="_2_QraFYR_0">小结<br>1：UDP是一个不可靠的通信协议，没有重传和确认，没有有序控制，也没有拥塞控制。<br>2：UDP 不保证报文的有效传递，不保证报文的有序，也就是说使用 UDP 的时候，我们需要做好丢包、重传、报文组装等工作。<br>3： UDP 相对TCP比较简单，适合的场景还是比较多的，我们常见的 DNS 服务，SNMP 服务都是基于 UDP 协议的，这些场景对时延、丢包都不是特别敏感。另外多人通信的场景，如聊天室、多人游戏等，也都会使用到 UDP 协议。<br>4：把UDP协议比做发生明信片确实生动形象，我告诉邮局送到那些地址，具体送到没有，需要接收端主动的响应（注意：这里的响应者更靠上层）协议本身不会进行重传&#47;确认&#47;时序&#47;拥塞等控制。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-22 07:45:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKzqiaZnBw2myRWY802u48Rw3W2zDtKoFQ6vN63m4FdyjibM21FfaOYe8MbMpemUdxXJeQH6fRdVbZA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>kissingers</span>
  </div>
  <div class="_2_QraFYR_0">老师，TCP 流，UDP包，流的说法怎么理解？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 流就像水流，一直持续不断的流淌，只不过流淌的是0101这样的比特流。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-06 08:37:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/f0/ca/4560f06b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>zhchnchn</span>
  </div>
  <div class="_2_QraFYR_0">如果不开启服务端，TCP 客户端的 connect 函数会直接返回“Connection refused”报错信息。而在 UDP 程序里，则会一直阻塞在这里。<br>-------------------------<br>这个地方不太理解。请问老师,对TCP来说，收到“Connection refused”报错信息，表明收到了服务端的RST包，如果服务端不开启，谁负责发送RST包呢?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不是RST包，RST的意思是connection reset；这里connection refused是对方的TCP协议栈发送的，工作在操作系统内核中。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-14 10:37:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/7e/3d/e97e661c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>江永枫</span>
  </div>
  <div class="_2_QraFYR_0">关于阻塞io，可以考虑使用多线程模型去提升性能，或者结合io多路复用来处理能力。<br><br><br>https:&#47;&#47;m.php.cn&#47;article&#47;410029.html</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 很快就会讲到了 ：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-14 08:51:52</div>
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
  <div class="_2_QraFYR_0">老师好，有 3 个问题请教一下：<br>1. TCP 有序传输的意思是：需要等到当前发送的包 ACK 之后，才发下一个包么？还是说可以一直发消息，收到 ACK 之后再确认对应的包发送成功？<br><br>2. 关于 TCP 和 UDP 连接，我可以这么理解么？<br>TCP 连接之后直到关闭，这期间发的消息，比如 client 发给(send 函数) server，<br>然后 server 回复(不知道用啥函数写回，也是 send 函数么？) client；<br>client 又继续发给 server，继续重复刚才的步骤....，<br>走的都是同一个连接。<br><br>而 UDP，client 发给（sendto 函数） server 是一个连接。而 server 回复（sendto 函数） client，又是另一个连接。<br><br>3. 下面的循环发送，我甚至是可以动态更改对端 client_addr 地址和端口信息的吧？<br>for (;;) {<br>  sendto(socket_fd, send_line, strlen(send_line), 0, (struct sockaddr *) &amp;client_addr, client_len);<br>}</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. TCP有序传输的意思是对于传送的字节流，接收端总是保证按照发送的顺序来接收；<br><br>2.TCP是面向连接的，UDP没有连接的概念；<br><br>3.当然可以，但是意义不大。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-16 21:42:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/df/0e/4e2b06d5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>流浪在寂寞古城</span>
  </div>
  <div class="_2_QraFYR_0">https:&#47;&#47;github.com&#47;worsun&#47;study&#47;tree&#47;master&#47;hack_time&#47;socket_code&#47;6</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-31 16:36:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/19/28/f4b4ed22.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>马留</span>
  </div>
  <div class="_2_QraFYR_0">如果对udp套接字调用了listen函数，会怎么样呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 改下程序，试试就知道了，最好把你的结果贴上来，大家一起研究。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-29 17:33:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q3auHgzwzM7zuDYFIutbSPc4eEtcMhdNBTI1FRR7q0xrGh2X1QdiaNxvAV31HcRUsjPWLaaWftqgwTnVoiaica8Nw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>胜辉（大V）</span>
  </div>
  <div class="_2_QraFYR_0">示例程序在我的测试机(ubuntu 20.04）上有几个报错。除了include需要添加多个库，还有就是sprintf不分需要改为如下：<br>        char send_line[MAXLINE+4];<br>        sprintf(send_line, &quot;Hi, %s&quot;, message);<br><br>因为&quot;Hi, &quot;本身占用了额外的4个字节，所以导致send_line跟&quot;Hi, &quot;+message的长度不匹配，需要修改send_line为message长度加4。不知道其他同学的测试环境那里如何？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我觉得不用啊，程序是用了strlen重新计算了新的send_line的长度的。<br><br>sendto(socket_fd, send_line, strlen(send_line), 0, (struct sockaddr *) &amp;client_addr, client_len);<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-25 01:29:12</div>
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
  <div class="_2_QraFYR_0">如果不开启服务端，而在 UDP 程序里，则会一直阻塞在这里（sendto）。<br><br>阻塞是为了等待消息发出去之后的 ACK 确认么？<br>不是说把明信片放到邮筒就不管了么？<br>邮筒可以认为是 IP 协议栈（即网络接口层？）吧？把 udp 包传递给 IP 层就不管了。<br><br>谢谢老师</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 阻塞是因为服务端没有开启，无法从服务端得到响应，你看一下程序原文，sendto已经执行成功了；<br><br>明信片的比喻对的，这里sendto已经成功执行了，相当于把明信片放到邮筒了，原程序里是希望通过recvfrom来或得服务端的反馈。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-16 21:59:47</div>
  </div>
</div>
</div>
</li>
</ul>