<audio title="13 _ 小数据包应对之策：理解TCP协议中的动态数据传输" src="https://static001.geekbang.org/resource/audio/ea/9d/eaa3479a08d67268f51167ca5156fd9d.mp3" controls="controls"></audio> 
<p>你好，我是盛延敏，这里是网络编程实战第13讲，欢迎回来。</p><p>在上一篇文章里，我在应用程序中模拟了TCP Keep-Alive机制，完成TCP心跳检测，达到发现不活跃连接的目的。在这一讲里，我们将从TCP角度看待数据流的发送和接收。</p><p>如果你学过计算机网络的话，那么对于发送窗口、接收窗口、拥塞窗口等名词肯定不会陌生，它们各自解决的是什么问题，又是如何解决的？在今天的文章里，我希望能从一个更加通俗易懂的角度进行剖析。</p><h2>调用数据发送接口以后……</h2><p>在前面的内容中，我们已经熟悉如何通过套接字发送数据，比如使用write或者send方法来进行数据流的发送。</p><p>我们已经知道，<strong>调用这些接口并不意味着数据被真正发送到网络上，其实，这些数据只是从应用程序中被拷贝到了系统内核的套接字缓冲区中，或者说是发送缓冲区中</strong>，等待协议栈的处理。至于这些数据是什么时候被发送出去的，对应用程序来说，是无法预知的。对这件事情真正负责的，是运行于操作系统内核的TCP协议栈实现模块。</p><h2>流量控制和生产者-消费者模型</h2><p>我们可以把理想中的TCP协议可以想象成一队运输货物的货车，运送的货物就是TCP数据包，这些货车将数据包从发送端运送到接收端，就这样不断周而复始。</p><!-- [[[read_end]]] --><p>我们仔细想一下，货物达到接收端之后，是需要卸货处理、登记入库的，接收端限于自己的处理能力和仓库规模，是不可能让这队货车以不可控的速度发货的。接收端肯定会和发送端不断地进行信息同步，比如接收端通知发送端：“后面那20车你给我等等，等我这里腾出地方你再继续发货。”</p><p>其实这就是发送窗口和接收窗口的本质，我管这个叫做“TCP的生产者-消费者”模型。</p><p>发送窗口和接收窗口是TCP连接的双方，一个作为生产者，一个作为消费者，为了达到一致协同的生产-消费速率、而产生的算法模型实现。</p><p>说白了，作为TCP发送端，也就是生产者，不能忽略TCP的接收端，也就是消费者的实际状况，不管不顾地把数据包都传送过来。如果都传送过来，消费者来不及消费，必然会丢弃；而丢弃反过来使得生产者又重传，发送更多的数据包，最后导致网络崩溃。</p><p>我想，理解了“TCP的生产者-消费者”模型，再反过来看发送窗口和接收窗口的设计目的和方式，我们就会恍然大悟了。</p><h2>拥塞控制和数据传输</h2><p>TCP的生产者-消费者模型，只是在考虑单个连接的数据传递，但是， TCP数据包是需要经过网卡、交换机、核心路由器等一系列的网络设备的，网络设备本身的能力也是有限的，当多个连接的数据包同时在网络上传送时，势必会发生带宽争抢、数据丢失等，这样，<strong>TCP就必须考虑多个连接共享在有限的带宽上，兼顾效率和公平性的控制</strong>，这就是拥塞控制的本质。</p><p>举个形象一点的例子，有一个货车行驶在半夜三点的大路上，这样的场景是断然不需要拥塞控制的。</p><p>我们可以把网络设备形成的网络信息高速公路和生活中实际的高速公路做个对比。正是因为有多个TCP连接，形成了高速公路上的多队运送货车，高速公路上开始变得熙熙攘攘，这个时候，就需要拥塞控制的接入了。</p><p>在TCP协议中，拥塞控制是通过拥塞窗口来完成的，拥塞窗口的大小会随着网络状况实时调整。</p><p>拥塞控制常用的算法有“慢启动”，它通过一定的规则，慢慢地将网络发送数据的速率增加到一个阈值。超过这个阈值之后，慢启动就结束了，另一个叫做“拥塞避免”的算法登场。在这个阶段，TCP会不断地探测网络状况，并随之不断调整拥塞窗口的大小。</p><p>现在你可以发现，在任何一个时刻，TCP发送缓冲区的数据是否能真正发送出去，<strong>至少</strong>取决于两个因素，一个是<strong>当前的发送窗口大小</strong>，另一个是<strong>拥塞窗口大小</strong>，而TCP协议中总是取两者中最小值作为判断依据。比如当前发送的字节为100，发送窗口的大小是200，拥塞窗口的大小是80，那么取200和80中的最小值，就是80，当前发送的字节数显然是大于拥塞窗口的，结论就是不能发送出去。</p><p>这里千万要分清楚发送窗口和拥塞窗口的区别。</p><p>发送窗口反应了作为单TCP连接、点对点之间的流量控制模型，它是需要和接收端一起共同协调来调整大小的；而拥塞窗口则是反应了作为多个TCP连接共享带宽的拥塞控制模型，它是发送端独立地根据网络状况来动态调整的。</p><h2>一些有趣的场景</h2><p>注意我在前面的表述中，提到了在任何一个时刻里，TCP发送缓冲区的数据是否能真正发送出去，用了“至少两个因素”这个说法，细心的你有没有想过这个问题，除了之前引入的发送窗口、拥塞窗口之外，还有什么其他因素吗？</p><p>我们考虑以下几个有趣的场景：</p><p>第一个场景，接收端处理得急不可待，比如刚刚读入了100个字节，就告诉发送端：“喂，我已经读走100个字节了，你继续发”，在这种情况下，你觉得发送端应该怎么做呢？</p><p>第二个场景是所谓的“交互式”场景，比如我们使用telnet登录到一台服务器上，或者使用SSH和远程的服务器交互，这种情况下，我们在屏幕上敲打了一个命令，等待服务器返回结果，这个过程需要不断和服务器端进行数据传输。这里最大的问题是，每次传输的数据可能都非常小，比如敲打的命令“pwd”，仅仅三个字符。这意味着什么？这就好比，每次叫了一辆大货车，只送了一个小水壶。在这种情况下，你又觉得发送端该怎么做才合理呢？</p><p>第三个场景是从接收端来说的。我们知道，接收端需要对每个接收到的TCP分组进行确认，也就是发送ACK报文，但是ACK报文本身是不带数据的分段，如果一直这样发送大量的ACK报文，就会消耗大量的带宽。之所以会这样，是因为TCP报文、IP报文固有的消息头是不可或缺的，比如两端的地址、端口号、时间戳、序列号等信息， 在这种情形下，你觉得合理的做法是什么？</p><p>TCP之所以复杂，就是因为TCP需要考虑的因素较多。像以上这几个场景，都是TCP需要考虑的情况，一句话概况就是如何有效地利用网络带宽。</p><p>第一个场景也被叫做糊涂窗口综合症，这个场景需要在接收端进行优化。也就是说，接收端不能在接收缓冲区空出一个很小的部分之后，就急吼吼地向发送端发送窗口更新通知，而是需要在自己的缓冲区大到一个合理的值之后，再向发送端发送窗口更新通知。这个合理的值，由对应的RFC规范定义。</p><p>第二个场景需要在发送端进行优化。这个优化的算法叫做Nagle算法，Nagle算法的本质其实就是限制大批量的小数据包同时发送，为此，它提出，在任何一个时刻，未被确认的小数据包不能超过一个。这里的小数据包，指的是长度小于最大报文段长度MSS的TCP分组。这样，发送端就可以把接下来连续的几个小数据包存储起来，等待接收到前一个小数据包的ACK分组之后，再将数据一次性发送出去。</p><p>第三个场景，也是需要在接收端进行优化，这个优化的算法叫做延时ACK。延时ACK在收到数据后并不马上回复，而是累计需要发送的ACK报文，等到有数据需要发送给对端时，将累计的ACK<strong>捎带一并发送出去</strong>。当然，延时ACK机制，不能无限地延时下去，否则发送端误认为数据包没有发送成功，引起重传，反而会占用额外的网络带宽。</p><h2>禁用Nagle算法</h2><p>有没有发现一个很奇怪的组合，即Nagle算法和延时ACK的组合。</p><p>这个组合为什么奇怪呢？我举一个例子你来体会一下。</p><p>比如，客户端分两次将一个请求发送出去，由于请求的第一部分的报文未被确认，Nagle算法开始起作用；同时延时ACK在服务器端起作用，假设延时时间为200ms，服务器等待200ms后，对请求的第一部分进行确认；接下来客户端收到了确认后，Nagle算法解除请求第二部分的阻止，让第二部分得以发送出去，服务器端在收到之后，进行处理应答，同时将第二部分的确认捎带发送出去。</p><p><img src="https://static001.geekbang.org/resource/image/42/eb/42073ad07805783add96ee87aeee8aeb.png?wh=562*512" alt=""><br>
你从这张图中可以看到，Nagle算法和延时确认组合在一起，增大了处理时延，实际上，两个优化彼此在阻止对方。</p><p>从上面的例子可以看到，在有些情况下Nagle算法并不适用， 比如对时延敏感的应用。</p><p>幸运的是，我们可以通过对套接字的修改来关闭Nagle算法。</p><pre><code>int on = 1; 
setsockopt(sock, IPPROTO_TCP, TCP_NODELAY, (void *)&amp;on, sizeof(on)); 
</code></pre><p>值得注意的是，除非我们对此有十足的把握，否则不要轻易改变默认的TCP Nagle算法。因为在现代操作系统中，针对Nagle算法和延时ACK的优化已经非常成熟了，有可能在禁用Nagle算法之后，性能问题反而更加严重。</p><h2>将写操作合并</h2><p>其实前面的例子里，如果我们能将一个请求一次性发送过去，而不是分开两部分独立发送，结果会好很多。所以，在写数据之前，将数据合并到缓冲区，批量发送出去，这是一个比较好的做法。不过，有时候数据会存储在两个不同的缓存中，对此，我们可以使用如下的方法来进行数据的读写操作，从而避免Nagle算法引发的副作用。</p><pre><code>ssize_t writev(int filedes, const struct iovec *iov, int iovcnt)
ssize_t readv(int filedes, const struct iovec *iov, int iovcnt);
</code></pre><p>这两个函数的第二个参数都是指向某个iovec结构数组的一个指针，其中iovec结构定义如下：</p><pre><code>struct iovec {
void *iov_base; /* starting address of buffer */
size_t　iov_len; /* size of buffer */
};”
</code></pre><p>下面的程序展示了集中写的方式：</p><pre><code>int main(int argc, char **argv) {
    if (argc != 2) {
        error(1, 0, &quot;usage: tcpclient &lt;IPaddress&gt;&quot;);
    }

    int socket_fd;
    socket_fd = socket(AF_INET, SOCK_STREAM, 0);

    struct sockaddr_in server_addr;
    bzero(&amp;server_addr, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(SERV_PORT);
    inet_pton(AF_INET, argv[1], &amp;server_addr.sin_addr);

    socklen_t server_len = sizeof(server_addr);
    int connect_rt = connect(socket_fd, (struct sockaddr *) &amp;server_addr, server_len);
    if (connect_rt &lt; 0) {
        error(1, errno, &quot;connect failed &quot;);
    }

    char buf[128];
    struct iovec iov[2];

    char *send_one = &quot;hello,&quot;;
    iov[0].iov_base = send_one;
    iov[0].iov_len = strlen(send_one);
    iov[1].iov_base = buf;
    while (fgets(buf, sizeof(buf), stdin) != NULL) {
        iov[1].iov_len = strlen(buf);
        int n = htonl(iov[1].iov_len);
        if (writev(socket_fd, iov, 2) &lt; 0)
            error(1, errno, &quot;writev failure&quot;);
    }
    exit(0);
}
</code></pre><p>这个程序的前半部分创建套接字，建立连接就不再赘述了。关键的是24-33行，使用了iovec数组，分别写入了两个不同的字符串，一个是“hello,”，另一个通过标准输入读入。</p><p>在启动该程序之前，我们需要启动服务器端程序，在客户端依次输入“world”和“network”：</p><pre><code>world
network
</code></pre><p>接下来我们可以看到服务器端接收到了iovec组成的新的字符串。这里的原理其实就是在调用writev操作时，会自动把几个数组的输入合并成一个有序的字节流，然后发送给对端。</p><pre><code>received 12 bytes: hello,world

received 14 bytes: hello,network
</code></pre><h2>总结</h2><p>今天的内容我重点讲述了TCP流量控制的生产者-消费者模型，你需要记住以下几点：</p><ul>
<li>发送窗口用来控制发送和接收端的流量；阻塞窗口用来控制多条连接公平使用的有限带宽。</li>
<li>小数据包加剧了网络带宽的浪费，为了解决这个问题，引入了如Nagle算法、延时ACK等机制。</li>
<li>在程序设计层面，不要多次频繁地发送小报文，如果有，可以使用writev批量发送。</li>
</ul><h2>思考题</h2><p>和往常一样，留两道思考题：</p><p>针对最后呈现的writev函数，你可以查一查Linux下一次性最多允许数组的大小是多少？</p><p>另外TCP拥塞控制算法是一个非常重要的研究领域，请你查阅下最新的有关这方面的研究，看看有没有新的发现？</p><p>欢迎你在评论区写下你的思考，也欢迎把这篇文章分享给你的朋友或者同事，一起来交流。</p>
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
  <div class="_2_QraFYR_0">非阻塞socket，对于write和send， 返回实际发送的字节数。所以一般在while里不断发送，直到全部发送完毕。    send根据只要根据要发送的buf做个偏移，很方便。   而writev 就很繁琐了啊</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你说的不错，一般我们只在合并缓冲区的时候才需要，绝大多数都是使用write和send 。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-30 11:24:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/27/cd/aa796a0d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Donkey</span>
  </div>
  <div class="_2_QraFYR_0">请教老师一个愚钝问题：大数据循环发送时，那接收方怎么接收才能接收完整的包？不会发生粘包等现象呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我觉得这个问题不愚钝。<br><br>每个包都有一个序列号，通过序列号按顺序就可以还原这个数据流；这个数据流本身也有例如checksum，序号大小等数据，这个数据流所有序列号的包都收到了，就可以完成数据流的拼装了。<br><br>解决粘包问题的关键是区分出数据的边界，我在第16降：如何理解TCP的“流”里讲到了这部分内容，你可以参考一下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-18 11:41:56</div>
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
  <div class="_2_QraFYR_0">TCP 拥塞控制算法,我知道最新的有BBR算法，这个算法在网络包填满路由器缓冲区之前就触发流量控制，而不在丢包后才触发，有效的降低了延迟。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，我也是刚知道这个算法。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-30 11:25:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1c/f3/b3/0ba7a760.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>一凡</span>
  </div>
  <div class="_2_QraFYR_0">telnet是使用Nagle 算法的吗，但是远程操作是实时性的呀？额</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 默认是使用的，批量发送小数据包在系统层面显然是得到充分的优化的，并不是我们想象中的经过很长时间之后才会发送出来，在telnet操作下，这个时间对操作人员是无感的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-13 17:28:01</div>
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
  <div class="_2_QraFYR_0">拥塞控制算法的话，应该是bbr吧，已经被合入linux  4.9内核了。与传统的reno, cubic等策略相比，最大的不同是，bbr不会因为链路噪声而执行乘性减窗，导致延迟过大。实际上，目前gwf的随机丢包的策略也是链路噪声。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Google出品，名声不小。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-04 21:33:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/7c/16/4d1e5cc1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>mgxian</span>
  </div>
  <div class="_2_QraFYR_0">问题1 ： grep &#39;IOV_MAX&#39; &#47;usr&#47;include&#47;limits.h</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-18 18:11:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/be/9a/b0b89be3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>不动声色满心澎湃</span>
  </div>
  <div class="_2_QraFYR_0">writev是减少write的使用次数吧。一次writev中的数据也有可能分多个包发出去。我说的对吗老师</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 当然，我们决定不了发包的次数。只不过合并写的方式，让操作系统可以一次性发送出去的几率更大。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-31 18:42:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/ae/0c/f39f847a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>D</span>
  </div>
  <div class="_2_QraFYR_0">就拥塞控制算法这块，记得前一阵阿里发布一个HPCC算法，盛老师点评一下?谢谢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 哈哈，研究性质的算法，应该还不错吧，不知道什么时候可以实际使用</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-30 15:38:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/23/66/413c0bb5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>LDxy</span>
  </div>
  <div class="_2_QraFYR_0">下载软件通常使用多线程建立多个TCP连接来下载一个大文件，是不是也是为了尽量避免TCP拥塞控制带来的影响，从而充分利用带宽？因为从实际使用来看，下载软件一旦跑满带宽，其他软件基本是是抢不过它的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这是为了加速下载的时间，说白了就是抢带宽。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-30 22:09:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/09/d6/5f366427.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>码农Kevin亮</span>
  </div>
  <div class="_2_QraFYR_0">请问老师，文中提到的小数据包发送的场景二三都是基于同一个目的地的吧？不同目的地的数据包不管多小，也不能合并发送吧？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，基于同一个目的地址。这里是说为了提高网络利用率，不能无限制的发送小数据包，也就是说，多个小数据包会在合适的时机合并成一个大的数据包发送出去。你的理解是相反的？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-17 18:16:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/67/8a/babd74dc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>锦</span>
  </div>
  <div class="_2_QraFYR_0">问题一：Linux中最多允许1024个元素<br>请教一个问题，tcp中有各种窗口，很头晕，比如，发送窗口，接收窗口，通告窗口（Advertised window），滑动窗口，拥塞窗口。为什么要弄这么多窗口呢？都是为了做流量控制吗？如何去理解呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 发送窗口和接收窗口都是通过滑动窗口机制来实现的，这是为了流量控制而引入的概念；而拥塞窗口则是为了拥塞控制而引入的概念。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-30 23:04:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/06/7e/735968e2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>西门吹牛</span>
  </div>
  <div class="_2_QraFYR_0">老师问个问题，客户端向服务端发送数据，在服务端，数据到达 socket 缓存区后，就给户 ack 确认，还是 socket 缓存区中的数据，被应用层从缓存中读取后才返回 ack 确认，我理解的是，只要到达服务端 socket 缓存区就给出 ack</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我同意你的理解。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-25 10:14:10</div>
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
  <div class="_2_QraFYR_0">发送窗口的大小是怎么定的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 协议栈实现的时候有默认值，而且算法可以动态调整；另外，也可以自己调用套接字接口函数来设置，不过一般不建议这么做。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-31 23:50:06</div>
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
  <div class="_2_QraFYR_0">概念较多，不过举个例子就好理解了，比如：在北京和华盛顿之间有一条可以双向传输货物的传送带，计算机的一切都是为了更快的速度和更大的容量，底层的实现更是如此，再对照各种概念就好理解了，他们所做的一切就是为了更有效地利用网络带宽。<br>需要权衡的就是，发多少信息？啥时候发？收发之间怎么实现无缝衔接，都没有无谓的等待，各种设备都满负荷的运转。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-23 16:33:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/81/4e/d71092f4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>夏目</span>
  </div>
  <div class="_2_QraFYR_0">老师，我有个问题，你举的那个例子里面，取发送窗口和拥塞窗口最小值的80，缓冲区数据是100不能发送出去。那么这个缓冲区数据要什么时候才能发送出去呢，是等到发送缓冲区大于100吗？发送窗口是上一次的ack返回时发送端拿到的吗？那发送端怎么知道什么时候大于100的呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 需要等到发现窗口和拥塞窗口都要超过100才可以，发送窗口是在ACK返回时根据一定的算法更新的，一旦更新就会影响发送端的行为。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-04 23:15:43</div>
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
  <div class="_2_QraFYR_0">老师，请教一下一个场景： 如果发送方使用writev()发送多个buf数据，接收端 用read()，而不是readv() 来读取数据，能否把writev的数据原样读取到么？是否 writev&#47;readv 需要配对使用？谢谢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-21 10:58:35</div>
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
  <div class="_2_QraFYR_0">老师，为什么很多网络库都自己设计了一套缓冲区？其最大的目的是为了什么呢？如果自己要实现一个，应该要从哪些方面考虑和入手呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 最大的目的是方便网络程序库的使用者使用，如果没有这个封装，网络程序库基本是不可用的。<br><br>设计的话，你往后面读，后面的章节有一个设计可以参考。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-20 08:27:20</div>
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
  <div class="_2_QraFYR_0">老师举例中的一个请求拆成一个包的例子--可以在代码中处理合并写入缓冲区。那么对于一个请求数据过大，缓冲区放不下一次请求包，不得已拆成两次进行两次请求，这种怎么办？是必然会造成延迟么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 为啥缓冲区放不小呢？除非是传递大文件，需要flush到磁盘上，大部分网络程序处理报文的时候，都会把数据缓存到内存缓冲区处理的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-26 16:20:56</div>
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
  <div class="_2_QraFYR_0">老师，这一章好像没有server端的代码，无法复现。是需要自己实现吗？使用readv接受数据后打印吗？有没有特别的地方呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 有一个batchwrite程序哦。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-18 08:18:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/52/d8/123a4981.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>绿箭侠</span>
  </div>
  <div class="_2_QraFYR_0">老师，代码的例子，服务端数据是怎么分清是两条数据的？根据换行符？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个，怎么说呢，纯粹是为了演示，因为系统很空闲，发送的数据有先有后，处理起来自然有先有后，也就是说，先发送的报文被先接收，后发送的报文后接收，自然就分成了两条数据。<br><br>在实战中，这个情况是不正确的，有可能先发送的后到达，所以需要对报文进行解析，例如你提到的换行符。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-14 17:05:14</div>
  </div>
</div>
</div>
</li>
</ul>