<audio title="07 _ What 还有本地套接字？" src="https://static001.geekbang.org/resource/audio/07/fc/0732f7b5f140e400a6742884e65fadfc.mp3" controls="controls"></audio> 
<p>你好，我是盛延敏，这里是网络编程实战第7讲，欢迎回来。</p><p>上一篇文章中，我们讲了UDP。很多同学都知道TCP和UDP，但是对本地套接字却不甚了解。</p><p>实际上，本地套接字是IPC，也就是本地进程间通信的一种实现方式。除了本地套接字以外，其它技术，诸如管道、共享消息队列等也是进程间通信的常用方法，但因为本地套接字开发便捷，接受度高，所以普遍适用于在同一台主机上进程间通信的各种场景。</p><p>那么今天我们就来学习下本地套接字方面的知识，并且利用本地套接字完成可靠字节流和数据报两种协议。</p><h2>从例子开始</h2><p>现在最火的云计算技术是什么？无疑是Kubernetes和Docker。在Kubernetes和Docker的技术体系中，有很多优秀的设计，比如Kubernetes的CRI（Container Runtime Interface），其思想是将Kubernetes的主要逻辑和Container Runtime的实现解耦。</p><p>我们可以通过netstat命令查看Linux系统内的本地套接字状况，下面这张图列出了路径为/var/run/dockershim.socket的stream类型的本地套接字，可以清楚地看到开启这个套接字的进程为kubelet。kubelet是Kubernetes的一个组件，这个组件负责将控制器和调度器的命令转化为单机上的容器实例。为了实现和容器运行时的解耦，kubelet设计了基于本地套接字的客户端-服务器GRPC调用。</p><!-- [[[read_end]]] --><p><img src="https://static001.geekbang.org/resource/image/c7/6b/c75a8467a84f30e523917f28f2f4266b.jpg?wh=2998*1346" alt=""><br>
眼尖的同学可能发现列表里还有docker-containerd.sock等其他本地套接字，是的，Docker其实也是大量使用了本地套接字技术来构建的。</p><p>如果我们在/var/run目录下将会看到docker使用的本地套接字描述符:</p><p><img src="https://static001.geekbang.org/resource/image/a0/4d/a0e6f8ca0f9c5727f554323a26a9c14d.jpg?wh=1843*394" alt=""></p><h2>本地套接字概述</h2><p>本地套接字一般也叫做UNIX域套接字，最新的规范已经改叫本地套接字。在前面的TCP/UDP例子中，我们经常使用127.0.0.1完成客户端进程和服务器端进程同时在本机上的通信，那么，这里的本地套接字又是什么呢？</p><p>本地套接字是一种特殊类型的套接字，和TCP/UDP套接字不同。TCP/UDP即使在本地地址通信，也要走系统网络协议栈，而本地套接字，严格意义上说提供了一种单主机跨进程间调用的手段，减少了协议栈实现的复杂度，效率比TCP/UDP套接字都要高许多。类似的IPC机制还有UNIX管道、共享内存和RPC调用等。</p><p>比如X  Window实现，如果发现是本地连接，就会走本地套接字，工作效率非常高。</p><p>现在你可以回忆一下，在前面介绍套接字地址时，我们讲到了本地地址，这个本地地址就是本地套接字专属的。</p><p><img src="https://static001.geekbang.org/resource/image/ed/58/ed49b0f1b658e82cb07a6e1e81f36b58.png?wh=3979*3770" alt=""></p><h2>本地字节流套接字</h2><p>我们先从字节流本地套接字开始。</p><p>这是一个字节流类型的本地套接字服务器端例子。在这个例子中，服务器程序打开本地套接字后，接收客户端发送来的字节流，并往客户端回送了新的字节流。</p><pre><code>#include  &quot;lib/common.h&quot;

int main(int argc, char **argv) {
    if (argc != 2) {
        error(1, 0, &quot;usage: unixstreamserver &lt;local_path&gt;&quot;);
    }

    int listenfd, connfd;
    socklen_t clilen;
    struct sockaddr_un cliaddr, servaddr;

    listenfd = socket(AF_LOCAL, SOCK_STREAM, 0);
    if (listenfd &lt; 0) {
        error(1, errno, &quot;socket create failed&quot;);
    }

    char *local_path = argv[1];
    unlink(local_path);
    bzero(&amp;servaddr, sizeof(servaddr));
    servaddr.sun_family = AF_LOCAL;
    strcpy(servaddr.sun_path, local_path);

    if (bind(listenfd, (struct sockaddr *) &amp;servaddr, sizeof(servaddr)) &lt; 0) {
        error(1, errno, &quot;bind failed&quot;);
    }

    if (listen(listenfd, LISTENQ) &lt; 0) {
        error(1, errno, &quot;listen failed&quot;);
    }

    clilen = sizeof(cliaddr);
    if ((connfd = accept(listenfd, (struct sockaddr *) &amp;cliaddr, &amp;clilen)) &lt; 0) {
        if (errno == EINTR)
            error(1, errno, &quot;accept failed&quot;);        /* back to for() */
        else
            error(1, errno, &quot;accept failed&quot;);
    }

    char buf[BUFFER_SIZE];

    while (1) {
        bzero(buf, sizeof(buf));
        if (read(connfd, buf, BUFFER_SIZE) == 0) {
            printf(&quot;client quit&quot;);
            break;
        }
        printf(&quot;Receive: %s&quot;, buf);

        char send_line[MAXLINE];
        sprintf(send_line, &quot;Hi, %s&quot;, buf);

        int nbytes = sizeof(send_line);

        if (write(connfd, send_line, nbytes) != nbytes)
            error(1, errno, &quot;write error&quot;);
    }

    close(listenfd);
    close(connfd);

    exit(0);

}
</code></pre><p>我对这个程序做一个详细的解释：</p><ul>
<li>第12～15行非常关键，<strong>这里创建的套接字类型，注意是AF_LOCAL，并且使用字节流格式</strong>。你现在可以回忆一下，TCP的类型是AF_INET和字节流类型；UDP的类型是AF_INET和数据报类型。在前面的文章中，我们提到AF_UNIX也是可以的，基本上可以认为和AF_LOCAL是等价的。</li>
<li>第17～21行创建了一个本地地址，这里的本地地址和IPv4、IPv6地址可以对应，数据类型为sockaddr_un，这个数据类型中的sun_family需要填写为AF_LOCAL，最为关键的是需要对sun_path设置一个本地文件路径。我们这里还做了一个unlink操作，以便把存在的文件删除掉，这样可以保持幂等性。</li>
<li>第23～29行，分别执行bind和listen操作，这样就监听在一个本地文件路径标识的套接字上，这和普通的TCP服务端程序没什么区别。</li>
<li>第41～56行，使用read和write函数从套接字中按照字节流的方式读取和发送数据。</li>
</ul><p>我在这里着重强调一下本地文件路径。关于本地文件路径，需要明确一点，它必须是“绝对路径”，这样的话，编写好的程序可以在任何目录里被启动和管理。如果是“相对路径”，为了保持同样的目的，这个程序的启动路径就必须固定，这样一来，对程序的管理反而是一个很大的负担。</p><p>另外还要明确一点，这个本地文件，必须是一个“文件”，不能是一个“目录”。如果文件不存在，后面bind操作时会自动创建这个文件。</p><p>还有一点需要牢记，在Linux下，任何文件操作都有权限的概念，应用程序启动时也有应用属主。如果当前启动程序的用户权限不能创建文件，你猜猜会发生什么呢？这里我先卖个关子，一会演示的时候你就会看到结果。</p><p>下面我们再看一下客户端程序。</p><pre><code>#include &quot;lib/common.h&quot;

int main(int argc, char **argv) {
    if (argc != 2) {
        error(1, 0, &quot;usage: unixstreamclient &lt;local_path&gt;&quot;);
    }

    int sockfd;
    struct sockaddr_un servaddr;

    sockfd = socket(AF_LOCAL, SOCK_STREAM, 0);
    if (sockfd &lt; 0) {
        error(1, errno, &quot;create socket failed&quot;);
    }

    bzero(&amp;servaddr, sizeof(servaddr));
    servaddr.sun_family = AF_LOCAL;
    strcpy(servaddr.sun_path, argv[1]);

    if (connect(sockfd, (struct sockaddr *) &amp;servaddr, sizeof(servaddr)) &lt; 0) {
        error(1, errno, &quot;connect failed&quot;);
    }

    char send_line[MAXLINE];
    bzero(send_line, MAXLINE);
    char recv_line[MAXLINE];

    while (fgets(send_line, MAXLINE, stdin) != NULL) {

        int nbytes = sizeof(send_line);
        if (write(sockfd, send_line, nbytes) != nbytes)
            error(1, errno, &quot;write error&quot;);

        if (read(sockfd, recv_line, MAXLINE) == 0)
            error(1, errno, &quot;server terminated prematurely&quot;);

        fputs(recv_line, stdout);
    }

    exit(0);
}
</code></pre><p>下面我带大家理解一下这个客户端程序。</p><ul>
<li>11～14行创建了一个本地套接字，和前面服务器端程序一样，用的也是字节流类型SOCK_STREAM。</li>
<li>16～18行初始化目标服务器端的地址。我们知道在TCP编程中，使用的是服务器的IP地址和端口作为目标，在本地套接字中则使用文件路径作为目标标识，sun_path这个字段标识的是目标文件路径，所以这里需要对sun_path进行初始化。</li>
<li>20行和TCP客户端一样，发起对目标套接字的connect调用，不过由于是本地套接字，并不会有三次握手。</li>
<li>28～38行从标准输入中读取字符串，向服务器端发送，之后将服务器端传输过来的字符打印到标准输出上。</li>
</ul><p>总体上，我们可以看到，本地字节流套接字和TCP服务器端、客户端编程最大的差异就是套接字类型的不同。本地字节流套接字识别服务器不再通过IP地址和端口，而是通过本地文件。</p><p>接下来，我们就运行这个程序来加深对此的理解。</p><h3>只启动客户端</h3><p>第一个场景中，我们只启动客户端程序：</p><pre><code>$ ./unixstreamclient /tmp/unixstream.sock
connect failed: No such file or directory (2)
</code></pre><p>我们看到，由于没有启动服务器端，没有一个本地套接字在/tmp/unixstream.sock这个文件上监听，客户端直接报错，提示我们没有文件存在。</p><h3>服务器端监听在无权限的文件路径上</h3><p>还记得我们在前面卖的关子吗？在Linux下，执行任何应用程序都有应用属主的概念。在这里，我们让服务器端程序的应用属主没有/var/lib/目录的权限，然后试着启动一下这个服务器程序 ：</p><pre><code>$ ./unixstreamserver /var/lib/unixstream.sock
bind failed: Permission denied (13)
</code></pre><p>这个结果告诉我们启动服务器端程序的用户，必须对本地监听路径有权限。这个结果和你期望的一致吗？</p><p>试一下root用户启动该程序：</p><pre><code>sudo ./unixstreamserver /var/lib/unixstream.sock
(阻塞运行中)
</code></pre><p>我们看到，服务器端程序正常运行了。</p><p>打开另外一个shell，我们看到/var/lib下创建了一个本地文件，大小为0，而且文件的最后结尾有一个（=）号。其实这就是bind的时候自动创建出来的文件。</p><pre><code>$ ls -al /var/lib/unixstream.sock
rwxr-xr-x 1 root root 0 Jul 15 12:41 /var/lib/unixstream.sock=
</code></pre><p>如果我们使用netstat命令查看UNIX域套接字，就会发现unixstreamserver这个进程，监听在/var/lib/unixstream.sock这个文件路径上。</p><p><img src="https://static001.geekbang.org/resource/image/58/b1/58d259d15b7012645d168a9c5d9f3fb1.jpg?wh=2369*334" alt=""><br>
看看，很简单吧，我们写的程序和鼎鼎大名的Kubernetes运行在同一机器上，原理和行为完全一致。</p><h3>服务器-客户端应答</h3><p>现在，我们让服务器和客户端都正常启动，并且客户端依次发送字符：</p><pre><code>$./unixstreamserver /tmp/unixstream.sock
Receive: g1
Receive: g2
Receive: g3
client quit
</code></pre><pre><code>$./unixstreamclient /tmp/unixstream.sock
g1
Hi, g1
g2
Hi, g2
g3
Hi, g3
^C
</code></pre><p>我们可以看到，服务器端陆续收到客户端发送的字节，同时，客户端也收到了服务器端的应答；最后，当我们使用Ctrl+C，让客户端程序退出时，服务器端也正常退出。</p><h2>本地数据报套接字</h2><p>我们再来看下在本地套接字上使用数据报的服务器端例子：</p><pre><code>#include  &quot;lib/common.h&quot;

int main(int argc, char **argv) {
    if (argc != 2) {
        error(1, 0, &quot;usage: unixdataserver &lt;local_path&gt;&quot;);
    }

    int socket_fd;
    socket_fd = socket(AF_LOCAL, SOCK_DGRAM, 0);
    if (socket_fd &lt; 0) {
        error(1, errno, &quot;socket create failed&quot;);
    }

    struct sockaddr_un servaddr;
    char *local_path = argv[1];
    unlink(local_path);
    bzero(&amp;servaddr, sizeof(servaddr));
    servaddr.sun_family = AF_LOCAL;
    strcpy(servaddr.sun_path, local_path);

    if (bind(socket_fd, (struct sockaddr *) &amp;servaddr, sizeof(servaddr)) &lt; 0) {
        error(1, errno, &quot;bind failed&quot;);
    }

    char buf[BUFFER_SIZE];
    struct sockaddr_un client_addr;
    socklen_t client_len = sizeof(client_addr);
    while (1) {
        bzero(buf, sizeof(buf));
        if (recvfrom(socket_fd, buf, BUFFER_SIZE, 0, (struct sockadd *) &amp;client_addr, &amp;client_len) == 0) {
            printf(&quot;client quit&quot;);
            break;
        }
        printf(&quot;Receive: %s \n&quot;, buf);

        char send_line[MAXLINE];
        bzero(send_line, MAXLINE);
        sprintf(send_line, &quot;Hi, %s&quot;, buf);

        size_t nbytes = strlen(send_line);
        printf(&quot;now sending: %s \n&quot;, send_line);

        if (sendto(socket_fd, send_line, nbytes, 0, (struct sockadd *) &amp;client_addr, client_len) != nbytes)
            error(1, errno, &quot;sendto error&quot;);
    }

    close(socket_fd);

    exit(0);
}
</code></pre><p>本地数据报套接字和前面的字节流本地套接字有以下几点不同：</p><ul>
<li>第9行创建的本地套接字，<strong>这里创建的套接字类型，注意是AF_LOCAL</strong>，协议类型为SOCK_DGRAM。</li>
<li>21～23行bind到本地地址之后，没有再调用listen和accept，回忆一下，这其实和UDP的性质一样。</li>
<li>28～45行使用recvfrom和sendto来进行数据报的收发，不再是read和send，这其实也和UDP网络程序一致。</li>
</ul><p>然后我们再看一下客户端的例子：</p><pre><code>#include &quot;lib/common.h&quot;

int main(int argc, char **argv) {
    if (argc != 2) {
        error(1, 0, &quot;usage: unixdataclient &lt;local_path&gt;&quot;);
    }

    int sockfd;
    struct sockaddr_un client_addr, server_addr;

    sockfd = socket(AF_LOCAL, SOCK_DGRAM, 0);
    if (sockfd &lt; 0) {
        error(1, errno, &quot;create socket failed&quot;);
    }

    bzero(&amp;client_addr, sizeof(client_addr));        /* bind an address for us */
    client_addr.sun_family = AF_LOCAL;
    strcpy(client_addr.sun_path, tmpnam(NULL));

    if (bind(sockfd, (struct sockaddr *) &amp;client_addr, sizeof(client_addr)) &lt; 0) {
        error(1, errno, &quot;bind failed&quot;);
    }

    bzero(&amp;server_addr, sizeof(server_addr));
    server_addr.sun_family = AF_LOCAL;
    strcpy(server_addr.sun_path, argv[1]);

    char send_line[MAXLINE];
    bzero(send_line, MAXLINE);
    char recv_line[MAXLINE];

    while (fgets(send_line, MAXLINE, stdin) != NULL) {
        int i = strlen(send_line);
        if (send_line[i - 1] == '\n') {
            send_line[i - 1] = 0;
        }
        size_t nbytes = strlen(send_line);
        printf(&quot;now sending %s \n&quot;, send_line);

        if (sendto(sockfd, send_line, nbytes, 0, (struct sockaddr *) &amp;server_addr, sizeof(server_addr)) != nbytes)
            error(1, errno, &quot;sendto error&quot;);

        int n = recvfrom(sockfd, recv_line, MAXLINE, 0, NULL, NULL);
        recv_line[n] = 0;

        fputs(recv_line, stdout);
        fputs(&quot;\n&quot;, stdout);
    }

    exit(0);
}
</code></pre><p>这个程序和UDP网络编程的例子基本是一致的，我们可以把它当作是用本地文件替换了IP地址和端口的UDP程序，不过，这里还是有一个非常大的不同的。</p><p>这个不同点就在16～22行。你可以看到16～22行将本地套接字bind到本地一个路径上，然而UDP客户端程序是不需要这么做的。本地数据报套接字这么做的原因是，它需要指定一个本地路径，以便在服务器端回包时，可以正确地找到地址；而在UDP客户端程序里，数据是可以通过UDP包的本地地址和端口来匹配的。</p><p>下面这段代码就展示了服务器端和客户端通过数据报应答的场景：</p><pre><code> ./unixdataserver /tmp/unixdata.sock
Receive: g1
now sending: Hi, g1
Receive: g2
now sending: Hi, g2
Receive: g3
now sending: Hi, g3
</code></pre><pre><code>$ ./unixdataclient /tmp/unixdata.sock
g1
now sending g1
Hi, g1
g2
now sending g2
Hi, g2
g3
now sending g3
Hi, g3
^C
</code></pre><p>我们可以看到，服务器端陆续收到客户端发送的数据报，同时，客户端也收到了服务器端的应答。</p><h2>总结</h2><p>我在开头已经说过，本地套接字作为常用的进程间通信技术，被用于各种适用于在同一台主机上进程间通信的场景。关于本地套接字，我们需要牢记以下两点：</p><ul>
<li>本地套接字的编程接口和IPv4、IPv6套接字编程接口是一致的，可以支持字节流和数据报两种协议。</li>
<li>本地套接字的实现效率大大高于IPv4和IPv6的字节流、数据报套接字实现。</li>
</ul><h2>思考题</h2><p>讲完本地套接字之后，我给你留几道思考题。</p><ol>
<li>在本地套接字字节流类型的客户端-服务器例子中，我们让服务器端以root账号启动，监听在/var/lib/unixstream.sock这个文件上。如果我们让客户端以普通用户权限启动，客户端可以连接上/var/lib/unixstream.sock吗？为什么呢？</li>
<li>我们看到客户端被杀死后，服务器端也正常退出了。看下退出后打印的日志，你不妨判断一下引起服务器端正常退出的逻辑是什么？</li>
<li>你有没有想过这样一个奇怪的场景：如果自己不小心写错了代码，本地套接字服务器端是SOCK_DGRAM，客户端使用的是SOCK_STREAM，路径和其他都是正确的，你觉得会发生什么呢？</li>
</ol><p>欢迎你在评论区写下你的思考，我会和你一起交流这些问题。如果这篇文章帮你弄懂了本地套接字，不妨把它分享给你的朋友或者同事，一起交流一下它吧！</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/2b/84/07f0c0d6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>supermouse</span>
  </div>
  <div class="_2_QraFYR_0">问题一：连接不上。错误提示是“Permission denied”<br><br>问题二：在服务端的代码中，对收到的客户端发送的数据长度做了判断，如果长度为0，则主动关闭服务端程序。这是杀死客户端后引发服务端关闭的原因。这也说明客户端在被强行终止的时候，会最后向服务端发送一条空消息，告知服务器自己这边的程序关闭了。<br><br>问题三：客户端在连接时会报错，错误提示是“Protocol wrong type for socket (91)”</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 看样子你都尝试了，实践出真知，赞</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-03 20:36:26</div>
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
  <div class="_2_QraFYR_0">问题1：<br>$ sudo .&#47;unixstreamserver &#47;var&#47;lib&#47;unixstream.sock<br><br>$ .&#47;unixstreamclient &#47;var&#47;lib&#47;unixstream.sock<br>connect failed: Permission denied (13)<br><br>问题2：<br>client: Ctrl + C -&gt; exit(0) -&gt; 发送FIN包；<br>server:满足条件read(connfd, buf, BUFFER_SIZE) == 0，结束。<br><br>问题3：<br>$ .&#47;unixdataserver &#47;tmp&#47;unixdata.sock<br><br>$ .&#47;unixstreamclient &#47;tmp&#47;unixdata.sock<br>connect failed: Protocol wrong type for socket (41)</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-17 16:59:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKVUskibDnhMt5MCIJ8227HWkeg2wEEyewps8GuWhWaY5fy7Ya56bu2ktMlxdla3K29Wqia9efCkWaQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>衬衫的价格是19美元</span>
  </div>
  <div class="_2_QraFYR_0">为什么AF_LOCAL+DGRAM的时候,客户端需要bind一个本地文件？<br>因为服务器收到来自客户端的数据想要给客户端回复的时候，需要知道发给谁。在其他情况下，服务器都有办法：<br>1.当使用STREAM时，不管是AF_INET还是AF_LOCAL, 都有连接的概念，所以服务器可以使用原来的连接<br>2.当使用AF_INET时，不管是DGRAM还是STREAM, 都能知道对方的ip+port, 也能确定一个唯一的进程<br>只有AF_LOCAL+DGRAM的时候，没有连接，也没有ip_+port,   只能显式的指定一个标志告诉服务端<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-26 15:59:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/60/07/c2738be7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Sylar.</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，方便把全部代码放git吗，因为示例缺少依赖</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 答疑篇统一回复</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-17 09:51:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4d/fd/0aa0e39f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>许童童</span>
  </div>
  <div class="_2_QraFYR_0">1.没有权限，报permission deny错误。<br>2.代码中有写：<br>if (read(connfd, buf, BUFFER_SIZE) == 0) {<br>            printf(&quot;client quit&quot;);<br>            break;<br>        }<br>3.客户端使用的是 SOCK_STREAM。猜测会因为使用协议不同，会连接不上，报错。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-16 11:15:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>15652790052</span>
  </div>
  <div class="_2_QraFYR_0">1.sock文件具体内容什么呢，为什么需要sock文件？<br>2.本地套接字底层时怎么实现的呢？<br>3.为什么没有了端口的概念？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1.没什么内容，只是一个占位符，告诉客户端要连哪个sock；<br>2.没有仔细研读过kernel的代码，不过我猜是大量复用了TCP&#47;IP层的公共实现；<br>3.因为文件已经告诉我们连接的目标了，所以不需要端口。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-29 21:45:17</div>
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
  <div class="_2_QraFYR_0">文章中说 TCP 和 UDP 的scoket 通讯 是走网络协议的，本地socket 通讯减少了网络协议的实现，是说明本地scoket通讯一点都不走网络协议吗？ 还是会走其中的一部分网络协议？ <br><br>如果本地 socket 通讯不走网络协议，那通讯的标准是什么呢？ 只是对 socket 文件的读写操作吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我是这么理解的，本地套接字本质还是进程间通信，只是借助了套接字的编程语义，比如stream和datagram，最下面肯定不走IP协议的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-17 20:05:31</div>
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
  <div class="_2_QraFYR_0">1、不能连接，需要权限才能对sock文件进行读写<br>2、代码30～33行，服务器读到的数据长度为0时会退出。<br>3、认为会出现如下现象：<br>如果服务器写错了，服务器会输出listen failed，客户端会输出connect failed；<br>如果客户端写错了，服务器会输出client quit，客户端会输出sendto error。<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-16 23:18:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/82/42/8b04d489.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>刘丹</span>
  </div>
  <div class="_2_QraFYR_0">请问老师本地套接字是否支持一对多、多对多通信？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 本地套接字datagram就可以</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-16 08:00:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/4f/a5/71358d7b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>J.M.Liu</span>
  </div>
  <div class="_2_QraFYR_0">datagram client中，为什么还要bind一个本地地址呀？没有看出这个这个本地地址有什么用呀。难道它是用来作为接受数据的缓冲区的吗？好像没有它也并不影响数据收发，换句话说，client_addr.sun_path填什么好像都无所谓啊。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个值是为了回包的需要，常规情况是不需要的，但是在某些情况下还是有点作用的，比如让客户端快速出错。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-30 23:35:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/8b/3e/3a50d491.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>蚂蚁哈哈哈</span>
  </div>
  <div class="_2_QraFYR_0">答案：<br>问题1: 客户端报错： &quot;connect failed: Permission denied&quot;<br>问题2:  下面这段代码中 == 0 的代码表示的就是客户端推出，这时候服务端打印 &quot;client quit&quot;, 然后推出 read 循环，关闭服务端程序。<br><br>`if (read(connfd, buf, BUFFER_SIZE) == 0) {<br>      printf(&quot;client quit&quot;);<br>      break;<br>    }`<br><br>问题三： 客户端报错 &quot;connect failed: Protocol wrong type for socket&quot;</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你的昵称好汗~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-30 08:53:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/31/71/a7b9a2d4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>重小楼不吃素</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，我在讨论区看到“一对多，多对多通信”这样的概念，有点不明白。请问一对多指的是建立一个套接字可以允许多个客户端连接的意思吗？  如果是的话，本地套接字的 SOCK_STREAM 方式也能一对多呀，老师为什么只提到 SOCK_DGRAM 可以？<br>问题二：<br>TCP\UDP\本地套接字 是否都有发送缓冲区、接收缓冲区，TCP的发送&#47;接收缓冲区很好理解，其余两个我没有查找到说得比较透彻的资料</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 第一个问题，一对一你可以理解成两个IP地址，一个对另一个发送数据；而一对多，你可以理解成一个IP地址对很多其他IP地址发送数据。<br><br>第二个问题，我理解UDP是没有发送和接收缓冲区的，数据被直接传递到数据链路层中发送出去。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-24 23:17:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/08/b8/6a296e32.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>夏雨</span>
  </div>
  <div class="_2_QraFYR_0">既然是本地套接字不走网路协议， 那本地套接字的TCP 和 UDP 又有什么去别， UDP在缓存里也不会丢包乱序。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 好问题。我是这么理解的，在本地套接字情况下，所谓SOCK_STREAM和SOCK_DGRAM是指编程方式和非本地套接字下的TCP、UDP一致。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-24 13:47:34</div>
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
  <div class="_2_QraFYR_0">这边举例的本地套接字是一个服务器对应着一个客户端。 那想问一下能不能一个服务器对应着多个客户端。如果对应的多个客户端，他们还是通过一个文件就可以进行通信？ 还是要设置说一对关系要通过一个文件进行通信？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你把文件理解成端口就可以了，多个客户端也是可以通过这个文件和一个服务器端进行通信的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-12 18:16:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/7d/d8/d7c77764.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>HunterYuan</span>
  </div>
  <div class="_2_QraFYR_0">对于第二个问题，第五讲已进行了充分说明：read 函数要求操作系统内核从套接字描述字 socketfd读取最多多少个字节（size），并将结果存储到 buffer 中。返回值告诉我们实际读取的字节数目，也有一些特殊情况，如果返回值为 0，表示 EOF（end-of-file），这在网络中表示对端发送了 FIN 包，要处理断连的情况。<br>也就是说，只有流方式会遇到这种情况：<br>client: Ctrl + C -&gt; exit(0) -&gt; 发送FIN包；<br>server:满足条件read(connfd, buf, BUFFER_SIZE) == 0，结束。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 头像好赞。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-28 09:17:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/87/5f/6bf8b74a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Kepler</span>
  </div>
  <div class="_2_QraFYR_0">lib&#47;common.h 的绝对路径是啥，有这个文件吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: https:&#47;&#47;github.com&#47;froghui&#47;yolanda&#47;blob&#47;master&#47;lib&#47;common.h</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-22 10:23:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/ec/1c/d323b066.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>knull</span>
  </div>
  <div class="_2_QraFYR_0">老师，你好，我练习写了一个echoserver（用AF_UNIX）,客户端主动断链。<br>发现在客户端close的时候，echoserver会读取到数据，并且最后一次write。但是，这个时候连接已经断了，会报错broken pipe。程序直接挂掉了。。。<br>这种情况该怎么处理呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1.加一个SIGPIPE信号处理<br>2.读到EOF就不要write了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-26 15:41:06</div>
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
  <div class="_2_QraFYR_0">请问一下如何能方便地知道一个linux系统提供的函数或者变量的头文件是哪个呢？经常不知道文中的代码的函数或者变量的头文件在哪里</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你可以在linux系统里执行man命令，例如man socket:<br><br>SOCKET(2)                                                              Linux Programmer&#39;s Manual                                                             SOCKET(2)<br><br>NAME<br>       socket - create an endpoint for communication<br><br>SYNOPSIS<br>       #include &lt;sys&#47;types.h&gt;          &#47;* See NOTES *&#47;<br>       #include &lt;sys&#47;socket.h&gt;<br><br>       int socket(int domain, int type, int protocol);</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-17 23:52:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/3e/23/a379f47d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Time Machine</span>
  </div>
  <div class="_2_QraFYR_0">请问下代码去哪儿获取啊？ &quot;lib&#47;common.h&quot; 是个啥包啊</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-15 00:27:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/88nXicqmkJWm3IXVfPfGQSk8SKIBVKjuC4qhzaCkf5Ud88uvKgS4Vf5AzCJ1uaFO0gpPnxdh4CowfhpxV1kSbXw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lixin</span>
  </div>
  <div class="_2_QraFYR_0">可不可以多个客户端连接一个服务端。那一个客户端退出会引起服务端的退出么</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-18 12:42:18</div>
  </div>
</div>
</div>
</li>
</ul>