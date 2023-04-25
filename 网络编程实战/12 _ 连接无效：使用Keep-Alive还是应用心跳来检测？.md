<audio title="12 _ 连接无效：使用Keep-Alive还是应用心跳来检测？" src="https://static001.geekbang.org/resource/audio/58/60/586941b4a8a67086dc4151bf00541d60.mp3" controls="controls"></audio> 
<p>你好，我是盛延敏，这里是网络编程实战第12讲，欢迎回来。</p><p>上一篇文章中，我们讲到了如何使用close和shutdown来完成连接的关闭，在大多数情况下，我们会优选shutdown来完成对连接一个方向的关闭，待对端处理完之后，再完成另外一个方向的关闭。</p><p>在很多情况下，连接的一端需要一直感知连接的状态，如果连接无效了，应用程序可能需要报错，或者重新发起连接等。</p><p>在这一篇文章中，我将带你体验一下对连接状态的检测，并提供检测连接状态的最佳实践。</p><h2>从一个例子开始</h2><p>让我们用一个例子开始今天的话题。</p><p>我之前做过一个基于NATS消息系统的项目，多个消息的提供者 （pub）和订阅者（sub）都连到NATS消息系统，通过这个系统来完成消息的投递和订阅处理。</p><p>突然有一天，线上报了一个故障，一个流程不能正常处理。经排查，发现消息正确地投递到了NATS服务端，但是消息订阅者没有收到该消息，也没能做出处理，导致流程没能进行下去。</p><p>通过观察消息订阅者后发现，消息订阅者到NATS服务端的连接虽然显示是“正常”的，但实际上，这个连接已经是无效的了。为什么呢？这是因为NATS服务器崩溃过，NATS服务器和消息订阅者之间的连接中断FIN包，由于异常情况，没能够正常到达消息订阅者，这样造成的结果就是消息订阅者一直维护着一个“过时的”连接，不会收到NATS服务器发送来的消息。</p><!-- [[[read_end]]] --><p>这个故障的根本原因在于，作为NATS服务器的客户端，消息订阅者没有及时对连接的有效性进行检测，这样就造成了问题。</p><p>保持对连接有效性的检测，是我们在实战中必须要注意的一个点。</p><h2>TCP Keep-Alive选项</h2><p>很多刚接触TCP编程的人会惊讶地发现，在没有数据读写的“静默”的连接上，是没有办法发现TCP连接是有效还是无效的。比如客户端突然崩溃，服务器端可能在几天内都维护着一个无用的 TCP连接。前面提到的例子就是这样的一个场景。</p><p>那么有没有办法开启类似的“轮询”机制，让TCP告诉我们，连接是不是“活着”的呢？</p><p>这就是TCP保持活跃机制所要解决的问题。实际上，TCP有一个保持活跃的机制叫做Keep-Alive。</p><p>这个机制的原理是这样的：</p><p>定义一个时间段，在这个时间段内，如果没有任何连接相关的活动，TCP保活机制会开始作用，每隔一个时间间隔，发送一个探测报文，该探测报文包含的数据非常少，如果连续几个探测报文都没有得到响应，则认为当前的TCP连接已经死亡，系统内核将错误信息通知给上层应用程序。</p><p>上述的可定义变量，分别被称为保活时间、保活时间间隔和保活探测次数。在Linux系统中，这些变量分别对应sysctl变量<code>net.ipv4.tcp_keepalive_time</code>、<code>net.ipv4.tcp_keepalive_intvl</code>、 <code>net.ipv4.tcp_keepalve_probes</code>，默认设置是7200秒（2小时）、75秒和9次探测。</p><p>如果开启了TCP保活，需要考虑以下几种情况：</p><p>第一种，对端程序是正常工作的。当TCP保活的探测报文发送给对端, 对端会正常响应，这样TCP保活时间会被重置，等待下一个TCP保活时间的到来。</p><p>第二种，对端程序崩溃并重启。当TCP保活的探测报文发送给对端后，对端是可以响应的，但由于没有该连接的有效信息，会产生一个RST报文，这样很快就会发现TCP连接已经被重置。</p><p>第三种，是对端程序崩溃，或对端由于其他原因导致报文不可达。当TCP保活的探测报文发送给对端后，石沉大海，没有响应，连续几次，达到保活探测次数后，TCP会报告该TCP连接已经死亡。</p><p>TCP保活机制默认是关闭的，当我们选择打开时，可以分别在连接的两个方向上开启，也可以单独在一个方向上开启。如果开启服务器端到客户端的检测，就可以在客户端非正常断连的情况下清除在服务器端保留的“脏数据”；而开启客户端到服务器端的检测，就可以在服务器无响应的情况下，重新发起连接。</p><p>为什么TCP不提供一个频率很好的保活机制呢？我的理解是早期的网络带宽非常有限，如果提供一个频率很高的保活机制，对有限的带宽是一个比较严重的浪费。</p><h2>应用层探活</h2><p>如果使用TCP自身的keep-Alive机制，在Linux系统中，最少需要经过2小时11分15秒才可以发现一个“死亡”连接。这个时间是怎么计算出来的呢？其实是通过2小时，加上75秒乘以9的总和。实际上，对很多对时延要求敏感的系统中，这个时间间隔是不可接受的。</p><p>所以，必须在应用程序这一层来寻找更好的解决方案。</p><p>我们可以通过在应用程序中模拟TCP Keep-Alive机制，来完成在应用层的连接探活。</p><p>我们可以设计一个PING-PONG的机制，需要保活的一方，比如客户端，在保活时间达到后，发起对连接的PING操作，如果服务器端对PING操作有回应，则重新设置保活时间，否则对探测次数进行计数，如果最终探测次数达到了保活探测次数预先设置的值之后，则认为连接已经无效。</p><p>这里有两个比较关键的点：</p><p>第一个是需要使用定时器，这可以通过使用I/O复用自身的机制来实现；第二个是需要设计一个PING-PONG的协议。</p><p>下面我们尝试来完成这样的一个设计。</p><h3>消息格式设计</h3><p>我们的程序是客户端来发起保活，为此定义了一个消息对象。你可以看到这个消息对象，这个消息对象是一个结构体，前4个字节标识了消息类型，为了简单，这里设计了<code>MSG_PING</code>、<code>MSG_PONG</code>、<code>MSG_TYPE 1</code>和<code>MSG_TYPE 2</code>四种消息类型。</p><pre><code>typedef struct {
    u_int32_t type;
    char data[1024];
} messageObject;

#define MSG_PING          1
#define MSG_PONG          2
#define MSG_TYPE1        11
#define MSG_TYPE2        21
</code></pre><h3>客户端程序设计</h3><p>客户端完全模拟TCP Keep-Alive的机制，在保活时间达到后，探活次数增加1，同时向服务器端发送PING格式的消息，此后以预设的保活时间间隔，不断地向服务器端发送PING格式的消息。如果能收到服务器端的应答，则结束保活，将保活时间置为0。</p><p>这里我们使用select I/O复用函数自带的定时器，select函数将在后面详细介绍。</p><pre><code>#include &quot;lib/common.h&quot;
#include &quot;message_objecte.h&quot;

#define    MAXLINE     4096
#define    KEEP_ALIVE_TIME  10
#define    KEEP_ALIVE_INTERVAL  3
#define    KEEP_ALIVE_PROBETIMES  3


int main(int argc, char **argv) {
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

    char recv_line[MAXLINE + 1];
    int n;

    fd_set readmask;
    fd_set allreads;

    struct timeval tv;
    int heartbeats = 0;

    tv.tv_sec = KEEP_ALIVE_TIME;
    tv.tv_usec = 0;

    messageObject messageObject;

    FD_ZERO(&amp;allreads);
    FD_SET(socket_fd, &amp;allreads);
    for (;;) {
        readmask = allreads;
        int rc = select(socket_fd + 1, &amp;readmask, NULL, NULL, &amp;tv);
        if (rc &lt; 0) {
            error(1, errno, &quot;select failed&quot;);
        }
        if (rc == 0) {
            if (++heartbeats &gt; KEEP_ALIVE_PROBETIMES) {
                error(1, 0, &quot;connection dead\n&quot;);
            }
            printf(&quot;sending heartbeat #%d\n&quot;, heartbeats);
            messageObject.type = htonl(MSG_PING);
            rc = send(socket_fd, (char *) &amp;messageObject, sizeof(messageObject), 0);
            if (rc &lt; 0) {
                error(1, errno, &quot;send failure&quot;);
            }
            tv.tv_sec = KEEP_ALIVE_INTERVAL;
            continue;
        }
        if (FD_ISSET(socket_fd, &amp;readmask)) {
            n = read(socket_fd, recv_line, MAXLINE);
            if (n &lt; 0) {
                error(1, errno, &quot;read error&quot;);
            } else if (n == 0) {
                error(1, 0, &quot;server terminated \n&quot;);
            }
            printf(&quot;received heartbeat, make heartbeats to 0 \n&quot;);
            heartbeats = 0;
            tv.tv_sec = KEEP_ALIVE_TIME;
        }
    }
}
</code></pre><p>这个程序主要分成三大部分：</p><p>第一部分为套接字的创建和连接建立：</p><ul>
<li>15-16行，创建了TCP套接字；</li>
<li>18-22行，创建了IPv4目标地址，其实就是服务器端地址，注意这里使用的是传入参数作为服务器地址；</li>
<li>24-28行，向服务器端发起连接。</li>
</ul><p>第二部分为select定时器准备：</p><ul>
<li>39-40行，设置了超时时间为KEEP_ALIVE_TIME，这相当于保活时间；</li>
<li>44-45行，初始化select函数的套接字。</li>
</ul><p>最重要的为第三部分，这一部分需要处理心跳报文：</p><ul>
<li>48行调用select函数，感知I/O事件。这里的I/O事件，除了套接字上的读操作之外，还有在39-40行设置的超时事件。当KEEP_ALIVE_TIME这段时间到达之后，select函数会返回0，于是进入53-63行的处理；</li>
<li>在53-63行，客户端已经在KEEP_ALIVE_TIME这段时间内没有收到任何对当前连接的反馈，于是发起PING消息，尝试问服务器端：“喂，你还活着吗？”这里我们通过传送一个类型为MSG_PING的消息对象来完成PING操作，之后我们会看到服务器端程序如何响应这个PING操作；</li>
<li>第65-74行是客户端在接收到服务器端程序之后的处理。为了简单，这里就没有再进行报文格式的转换和分析。在实际的工作中，这里其实是需要对报文进行解析后处理的，只有是PONG类型的回应，我们才认为是PING探活的结果。这里认为既然收到服务器端的报文，那么连接就是正常的，所以会对探活计数器和探活时间都置零，等待下一次探活时间的来临。</li>
</ul><h3>服务器端程序设计</h3><p>服务器端的程序接受一个参数，这个参数设置的比较大，可以模拟连接没有响应的情况。服务器端程序在接收到客户端发送来的各种消息后，进行处理，其中如果发现是PING类型的消息，在休眠一段时间后回复一个PONG消息，告诉客户端：“嗯，我还活着。”当然，如果这个休眠时间很长的话，那么客户端就无法快速知道服务器端是否存活，这是我们模拟连接无响应的一个手段而已，实际情况下，应该是系统崩溃，或者网络异常。</p><pre><code>#include &quot;lib/common.h&quot;
#include &quot;message_objecte.h&quot;

static int count;

int main(int argc, char **argv) {
    if (argc != 2) {
        error(1, 0, &quot;usage: tcpsever &lt;sleepingtime&gt;&quot;);
    }

    int sleepingTime = atoi(argv[1]);

    int listenfd;
    listenfd = socket(AF_INET, SOCK_STREAM, 0);

    struct sockaddr_in server_addr;
    bzero(&amp;server_addr, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    server_addr.sin_port = htons(SERV_PORT);

    int rt1 = bind(listenfd, (struct sockaddr *) &amp;server_addr, sizeof(server_addr));
    if (rt1 &lt; 0) {
        error(1, errno, &quot;bind failed &quot;);
    }

    int rt2 = listen(listenfd, LISTENQ);
    if (rt2 &lt; 0) {
        error(1, errno, &quot;listen failed &quot;);
    }

    int connfd;
    struct sockaddr_in client_addr;
    socklen_t client_len = sizeof(client_addr);

    if ((connfd = accept(listenfd, (struct sockaddr *) &amp;client_addr, &amp;client_len)) &lt; 0) {
        error(1, errno, &quot;bind failed &quot;);
    }

    messageObject message;
    count = 0;

    for (;;) {
        int n = read(connfd, (char *) &amp;message, sizeof(messageObject));
        if (n &lt; 0) {
            error(1, errno, &quot;error read&quot;);
        } else if (n == 0) {
            error(1, 0, &quot;client closed \n&quot;);
        }

        printf(&quot;received %d bytes\n&quot;, n);
        count++;

        switch (ntohl(message.type)) {
            case MSG_TYPE1 :
                printf(&quot;process  MSG_TYPE1 \n&quot;);
                break;

            case MSG_TYPE2 :
                printf(&quot;process  MSG_TYPE2 \n&quot;);
                break;

            case MSG_PING: {
                messageObject pong_message;
                pong_message.type = MSG_PONG;
                sleep(sleepingTime);
                ssize_t rc = send(connfd, (char *) &amp;pong_message, sizeof(pong_message), 0);
                if (rc &lt; 0)
                    error(1, errno, &quot;send failure&quot;);
                break;
            }

            default :
                error(1, 0, &quot;unknown message type (%d)\n&quot;, ntohl(message.type));
        }

    }

}
</code></pre><p>服务器端程序主要分为两个部分。</p><p>第一部分为监听过程的建立，包括7-38行； 第13-14行先创建一个本地TCP监听套接字；16-20行绑定该套接字到本地端口和ANY地址上；第27-38行分别调用listen和accept完成被动套接字转换和监听。</p><p>第二部分为43行到77行，从建立的连接套接字上读取数据，解析报文，根据消息类型进行不同的处理。</p><ul>
<li>55-57行为处理MSG_TYPE1的消息；</li>
<li>59-61行为处理MSG_TYPE2的消息；</li>
<li>重点是64-72行处理MSG_PING类型的消息。通过休眠来模拟响应是否及时，然后调用send函数发送一个PONG报文，向客户端表示“还活着”的意思；</li>
<li>74行为异常处理，因为消息格式不认识，所以程序出错退出。</li>
</ul><h2>实验</h2><p>基于上面的程序设计，让我们分别做两个不同的实验：</p><p>第一次实验，服务器端休眠时间为60秒。</p><p>我们看到，客户端在发送了三次心跳检测报文PING报文后，判断出连接无效，直接退出了。之所以造成这样的结果，是因为在这段时间内没有接收到来自服务器端的任何PONG报文。当然，实际工作的程序，可能需要不一样的处理，比如重新发起连接。</p><pre><code>$./pingclient 127.0.0.1
sending heartbeat #1
sending heartbeat #2
sending heartbeat #3
connection dead
</code></pre><pre><code>$./pingserver 60
received 1028 bytes
received 1028 bytes
</code></pre><p>第二次实验，我们让服务器端休眠时间为5秒。</p><p>我们看到，由于这一次服务器端在心跳检测过程中，及时地进行了响应，客户端一直都会认为连接是正常的。</p><pre><code>$./pingclient 127.0.0.1
sending heartbeat #1
sending heartbeat #2
received heartbeat, make heartbeats to 0
received heartbeat, make heartbeats to 0
sending heartbeat #1
sending heartbeat #2
received heartbeat, make heartbeats to 0
received heartbeat, make heartbeats to 0
</code></pre><pre><code>$./pingserver 5
received 1028 bytes
received 1028 bytes
received 1028 bytes
received 1028 bytes
</code></pre><h2>总结</h2><p>通过今天的文章，我们能看到虽然TCP没有提供系统的保活能力，让应用程序可以方便地感知连接的存活，但是，我们可以在应用程序里灵活地建立这种机制。一般来说，这种机制的建立依赖于系统定时器，以及恰当的应用层报文协议。比如，使用心跳包就是这样一种保持Keep Alive的机制。</p><h2>思考题</h2><p>和往常一样，我留两道思考题：</p><p>你可以看到今天的内容主要是针对TCP的探活，那么你觉得这样的方法是否同样适用于UDP呢？</p><p>第二道题是，有人说额外的探活报文占用了有限的带宽，对此你是怎么想的呢？而且，为什么需要多次探活才能决定一个TCP连接是否已经死亡呢？</p><p>欢迎你在评论区写下你的思考，我会和你一起交流。也欢迎把这篇文章分享给你的朋友或者同事，与他们一起讨论一下这两个问题吧。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/df/1e/cea897e8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>传说中的成大大</span>
  </div>
  <div class="_2_QraFYR_0">思考题<br>1. udp不需要连接 所以没有必要心跳包<br>2. 我觉得还是很有必要判定存活 像以前网吧打游戏 朋友的电脑突然蓝屏死机 朋友的角色还残留于游戏中,所以服务器为了判定他是否真的存活还是需要一个心跳包 隔了一段时间过后把朋友角色踢下线</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 2是一个很好的例子。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-28 15:34:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJ61zTDmLk7IhLJn6seBPOwsVaKIWUWaxk5YmsdYBZUOYMQCsyl9iaQVSg9U5qJVLLOCFUoLUuYnRA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>fjpcode</span>
  </div>
  <div class="_2_QraFYR_0">1.UDP里面各方并不会维护一个socket上下文状态是无连接的，如果为了连接而保活是不必要的，如果为了探测对端是否正常工作而做ping-pong也是可行的。<br>2.额外的探活报文是会占用一些带宽资源，可根据实际业务场景，适当增加保活时间，降低探活频率，简化ping-pong协议。 <br>3.多次探活是为了防止误伤，避免ping包在网络中丢失掉了，而误认为对端死亡。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-28 12:51:23</div>
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
  <div class="_2_QraFYR_0">想到HTTP Header也能设置Connection: Keep-Alive，也是应用层协议，是不是底层实现也类似于定时器+Ping Pong的思路？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: HTTP的 keep-alive是为了使http变成长连接，在此前的http 1.0中，每次http的请求-响应之后，tcp连接就会被释放掉，这显然是非常浪费的，于是通过加入keep-alive，使得http连接不会被立即释放。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-17 19:28:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/6b/1b/4b397b80.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>云师兄</span>
  </div>
  <div class="_2_QraFYR_0">文章中提到保活有两个方向，实际应用中，会有两个方向同时探测的场景吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 当然有。服务器端要探活client来保证自己不会维护无效连接，客户端来探活保持自己是不是可以持续申请资源。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-17 09:24:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/a1/69/0ddda908.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>满怀</span>
  </div>
  <div class="_2_QraFYR_0">老师 我想问一下 看您在回复当中有说 虽然TCP本身的keep-alive机制可以设置保活时间，保活探测时间间隔以及探测次数，但是应用层会无法感知，那么这种情况下会怎么处理呢 就是文中所给出的三种情况吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，大部分的应用程序开发者都会选择自己在应用层处理连接有效性的检测。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-20 19:40:46</div>
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
  <div class="_2_QraFYR_0">老师  同步连接可以实现心跳包么 如果不能的话  那同步连接如果因为客户端崩溃 没有通过四次挥手结束连接  服务端还堵塞在接收数据  那么这样如何判断对方已经离开呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 首先，客户端应用程序崩溃是可以有FIN包的，如果有的话，read阻塞就可以返回了；其次，如果真的是客户端机器跪了，那么是没有FIN包发出的，这个时候，我们只好一直在那里傻等。<br><br>且慢，还有别的辙，那就是不要用阻塞I&#47;O，不要在哪里傻傻等待。使用I&#47;O复用就可以办到的，往后看就会明白了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-06 09:37:55</div>
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
  <div class="_2_QraFYR_0">服务器端恢复心跳包这段   <br>case MSG_PING:   <br>{  <br>    messageObject pong_message;     <br>    pong_message.type = MSG_PONG;  <br>    sleep(sleepingTime);      <br>     ssize_t rc = send(connfd, (char *) &amp;pong_message, sizeof(pong_message), 0);     <br>老师，这个type是int类型，应该将其转为网络序才对吧？ pong_message.type = htonl(MSG_PONG);     <br>而且你的客户端程序也是有转换的，是不是服务器端这里忘记了？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Bingo。确实是我忘记了:)</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-18 21:18:38</div>
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
  <div class="_2_QraFYR_0">对于协议栈中的TCP的keep-alive是可以手动配置的，全局配置通过修改net.ipv4下的等参数；局部配置可以通过setsockopt修改socket选项。在工作中遇到过一个oracle,windows服务器的保活时间大于，我们设备状态连接表的保活时间，导致，一段时间后重连，导致服务器报错问题。当时最最快的处理办法是，修改window默认的保活时间小于状态连接失效时间。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 学习了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-17 09:12:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/fd/81/1864f266.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>石将从</span>
  </div>
  <div class="_2_QraFYR_0">为啥这句套接字要加1呢？int rc = select(socket_fd + 1, &amp;readmask, NULL, NULL, &amp;tv);</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这是因为，嗯，跟select实现有关，我再后面讲select时详细剖析，现在先记住吧。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-29 00:34:01</div>
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
  <div class="_2_QraFYR_0">这个是客户端探活服务器是否还存活，还可以用服务器来探活客户端是否还存活。我原先也有接触过这个心跳，不过实现的机制比较简单，就是客户端连上服务器了，客户端要每个M秒向服务器发包，服务器如果在M+10秒内没收到这个心跳包就判定客户端已经死亡。就会在应用层方面判定客户端已经失去连接。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 没有服务器到客户端的反向连接检测么？M+10秒有点狠啊。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-02 11:28:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/31/1a/fd82b2d5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>刘士涛</span>
  </div>
  <div class="_2_QraFYR_0">请问老师一个tcp长连接，网络发生故障， client和server都还没有断开这个连接， 这时候client调用send发送数据是个什么行为？我理解send进缓冲区是一直成功的， 但是由于网络不通， 数据不会发出去。这时候如果想要发现这个tcp断开就只能client应用层设置一个超时， 超时后调用shutdown， 有其他的方式可以感知到这个tcp断开么</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 应用层设置ping-pang的操作，如果在一段时间内没有应用层心跳报文，认为tcp已经断开。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-19 15:39:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/wwM75BhyU43UYOJ6fZCZgY6pfNPGHHRlooPLQEtDGUNic4aLRHWmBRTpIiblBAFheUVm9Sw8HWAChcFsnVM2sd5Q/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_d6f50f</span>
  </div>
  <div class="_2_QraFYR_0">老师，第二次运行程序，客户端的定时器设定为每10秒钟进行一次select返回0，此时进行探测包发送。而服务器端延迟5秒进行应答，为什么客户端每次都能发送两个包后才清零？不应该是只进行一个包的发送，服务器端延迟5秒就给出应答，然后清零吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是这样的，<br><br>当客户端10秒钟select超时时间到达，第一次进入heartbeat，发送报文给服务器端，同时客户端把下一次select超时时间设置为3秒(KEEP_ALIVE_INTERVAL)；<br><br>由于服务器端是5秒之后才回复，3秒之后，第二次heartbeat时间到，客户端发送第二个heartbeat。<br><br>5秒之后，第一次的heartbeat回复到，客户端把超时时间又重新设置为10秒。<br><br>再过5秒之后，第二次的heartbeat回复到，客户端把超时时间再次设置为10秒。<br><br>如此反复。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-05 21:49:07</div>
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
  <div class="_2_QraFYR_0">tcp探活的目的之一是判断该连接是否还需要，以此决定是维持还是释放该连接资源。udp无连接，也不占用服务器或者客户端的资源，因此业务无需探活</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-04 16:55:36</div>
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
  <div class="_2_QraFYR_0">我看到大家对于第二个问题的答案都提到了为了避免探活包丢包 所以要发多个探活包。但是我觉得只要发了探活包对方就一定能收到，就算丢了 发送端也会重传。而大家说的发送多个探活包的原因 是因为重传需要等到计时器超时才传，而如果网络堵塞的话可能会出现频繁丢包 那么服务端可能需要很久才能知道对方已经离开 发多个探活包的话 就减少了这个等待的时间 这样理解对么</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 探活包的多次发送，还是为了减少误判的概率，你说的是一种情况，但我觉得多此探活肯定会拉长对一个&quot;无效&quot;连接的判断时间的。这个是一个tradeoff，所以大多数程序都把这个作为选项让使用者自己配置。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-29 16:04:07</div>
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
  <div class="_2_QraFYR_0">TCP本身的keep-alive的时间是可以自己设置的吗？如果是可以自己设置的，为何还需要自己实现这个机制？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以设置的，但是有时候应用层需要感知处理这样的异常</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-28 08:34:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKSjC36vSdPiaibQqrVicXYk7pvia1JKpY8aib9DBNMBHIPVxPE19wP9MTm63akRp6uYBjFibEk6XytrgNg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张贺龙</span>
  </div>
  <div class="_2_QraFYR_0">保活是应用层的动作可以用于udp，在数据中心集群中多个主机的通讯也有使用udp通讯的场景，保活的作用还是相当大的；保活会占用额外带宽，但由于只在无业务消息时传送，因此不会影响正常业务信息的传送；多次确认是为了避免偶然网络抖动造成的消息传送超时</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-19 07:44:05</div>
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
  <div class="_2_QraFYR_0">老师你在留言里面回复，很多实际的场景都是自己在应用层加心跳，这个是指http的keep-alive么？如果是在java开发的应用，这个心跳是指客户端定时向服务端发起请求么？可不可以结合一下java应用举一个例子</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: http的keep-alive不是我这里的心跳。<br>我这里的意思是指，通过在客户端-服务器之间发送定期的报文，用来检查连接的有效性。不一定是客户端发给服务器，服务器端也可以发送给客户端。在很多程序里都叫做&quot;ping-pang&quot;报文。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-11 18:29:06</div>
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
  <div class="_2_QraFYR_0">老师我想问一下， 连接无效：使用Keep-Alive还是应用心跳来检测？<br>是不是应该使用应用心跳来检测，比使用Keep-Alive更好啊？<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 只能说，很多实际的场景都是自己在应用层加心跳来解的。没有好不好，是大家总结出来的行之有效的方法。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-19 11:17:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/ee/67/03c95ec2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>守望</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，有个问题想请教一下，假设服务端重启了，起来后还是绑定同样的ip和端口，而客户端仍然发送，如何最大程度的避免客户端发送数据丢失呢？这种心跳检测貌似发现过慢。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你说的这种情况，连接是需要重建建立的，心跳检测如果觉得慢，可以通过设置不同的处理逻辑来加速，比如一旦发现连接异常，就立即启动一个补偿机制，短时间内进行重连。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-11 08:44:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJKj3GbvevFibxwJibTqm16NaE8MXibwDUlnt5tt73KF9WS2uypha2m1Myxic6Q47Zaj2DZOwia3AgicO7Q/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>饭</span>
  </div>
  <div class="_2_QraFYR_0">老师，一台服务器同时在线的tcp&#47;ip连接数，理论上是不是最多只有65535个?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你的结论是来自端口号的限制么？如果是这样的话，我觉得是一个错误的结论。后面章节讲到的C10K&#47;C10M问题，就是在研究单机上的连接达到10K、10M的技术。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-28 16:11:33</div>
  </div>
</div>
</div>
</li>
</ul>