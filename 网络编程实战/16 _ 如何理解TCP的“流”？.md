<audio title="16 _ 如何理解TCP的“流”？" src="https://static001.geekbang.org/resource/audio/d1/cb/d1471106b130f34c1fce97b4f1f312cb.mp3" controls="controls"></audio> 
<p>你好，我是盛延敏，这里是网络编程实战第16讲，欢迎回来。</p><p>上一讲我们讲到了使用SO_REUSEADDR套接字选项，可以让服务器满足快速重启的需求。在这一讲里，我们回到数据的收发这个主题，谈一谈如何理解TCP的数据流特性。</p><h2>TCP是一种流式协议</h2><p>在前面的章节中，我们讲的都是单个客户端-服务器的例子，可能会给你造成一种错觉，好像TCP是一种应答形式的数据传输过程，比如发送端一次发送network和program这样的报文，在前面的例子中，我们看到的结果基本是这样的：</p><p>发送端：network ----&gt; 接收端回应：Hi, network</p><p>发送端：program -----&gt; 接收端回应：Hi, program</p><p>这其实是一个假象，之所以会这样，是因为网络条件比较好，而且发送的数据也比较少。</p><p>为了让大家理解TCP数据是流式的这个特性，我们分别从发送端和接收端来阐述。</p><p>我们知道，在发送端，当我们调用send函数完成数据“发送”以后，数据并没有被真正从网络上发送出去，只是从应用程序拷贝到了操作系统内核协议栈中，至于什么时候真正被发送，取决于发送窗口、拥塞窗口以及当前发送缓冲区的大小等条件。也就是说，我们不能假设每次send调用发送的数据，都会作为一个整体完整地被发送出去。</p><!-- [[[read_end]]] --><p>如果我们考虑实际网络传输过程中的各种影响，假设发送端陆续调用send函数先后发送network和program报文，那么实际的发送很有可能是这个样子的。</p><p>第一种情况，一次性将network和program在一个TCP分组中发送出去，像这样：</p><pre><code>...xxxnetworkprogramxxx...
</code></pre><p>第二种情况，program的部分随network在一个TCP分组中发送出去，像这样：</p><p>TCP分组1：</p><pre><code>...xxxxxnetworkpro
</code></pre><p>TCP分组2：</p><pre><code>gramxxxxxxxxxx...
</code></pre><p>第三种情况，network的一部分随TCP分组被发送出去，另一部分和program一起随另一个TCP分组发送出去，像这样。</p><p>TCP分组1：</p><pre><code>...xxxxxxxxxxxnet
</code></pre><p>TCP分组2：</p><pre><code>workprogramxxx...
</code></pre><p>实际上类似的组合可以枚举出无数种。不管是哪一种，核心的问题就是，我们不知道network和program这两个报文是如何进行TCP分组传输的。换言之，我们在发送数据的时候，不应该假设“数据流和TCP分组是一种映射关系”。就好像在前面，我们似乎觉得network这个报文一定对应一个TCP分组，这是完全不正确的。</p><p>如果我们再来看客户端，数据流的特征更明显。</p><p>我们知道，接收端缓冲区保留了没有被取走的数据，随着应用程序不断从接收端缓冲区读出数据，接收端缓冲区就可以容纳更多新的数据。如果我们使用recv从接收端缓冲区读取数据，发送端缓冲区的数据是以字节流的方式存在的，无论发送端如何构造TCP分组，接收端最终收到的字节流总是像下面这样：</p><pre><code>xxxxxxxxxxxxxxxxxnetworkprogramxxxxxxxxxxxx
</code></pre><p>关于接收端字节流，有两点需要注意：</p><p>第一，这里netwrok和program的顺序肯定是会保持的，也就是说，先调用send函数发送的字节，总在后调用send函数发送字节的前面，这个是由TCP严格保证的；</p><p>第二，如果发送过程中有TCP分组丢失，但是其后续分组陆续到达，那么TCP协议栈会缓存后续分组，直到前面丢失的分组到达，最终，形成可以被应用程序读取的数据流。</p><h2>网络字节排序</h2><p>我们知道计算机最终保存和传输，用的都是0101这样的二进制数据，字节流在网络上的传输，也是通过二进制来完成的。</p><p>从二进制到字节是通过编码完成的，比如著名的ASCII编码，通过一个字节8个比特对常用的西方字母进行了编码。</p><p>这里有一个有趣的问题，如果需要传输数字，比如0x0201，对应的二进制为0000001000000001，那么两个字节的数据到底是先传0x01，还是相反？</p><p><img src="https://static001.geekbang.org/resource/image/79/e6/79ada2f154205f5170cf8e69bf9f59e6.png?wh=3291*881" alt=""><br>
在计算机发展的历史上，对于如何存储这个数据没有形成标准。比如这里讲到的问题，不同的系统就会有两种存法，一种是将0x02高字节存放在起始地址，这个叫做<strong>大端字节序</strong>（Big-Endian）。另一种相反，将0x01低字节存放在起始地址，这个叫做<strong>小端字节序</strong>（Little-Endian）。</p><p>但是在网络传输中，必须保证双方都用同一种标准来表达，这就好比我们打电话时说的是同一种语言，否则双方不能顺畅地沟通。这个标准就涉及到了网络字节序的选择问题，对于网络字节序，必须二选一。我们可以看到网络协议使用的是大端字节序，我个人觉得大端字节序比较符合人类的思维习惯，你可以想象手写一个多位数字，从开始往小位写，自然会先写大位，比如写12, 1234，这个样子。</p><p>为了保证网络字节序一致，POSIX标准提供了如下的转换函数：</p><pre><code>uint16_t htons (uint16_t hostshort)
uint16_t ntohs (uint16_t netshort)
uint32_t htonl (uint32_t hostlong)
uint32_t ntohl (uint32_t netlong)
</code></pre><p>这里函数中的n代表的就是network，h代表的是host，s表示的是short，l表示的是long，分别表示16位和32位的整数。</p><p>这些函数可以帮助我们在主机（host）和网络（network）的格式间灵活转换。当使用这些函数时，我们并不需要关心主机到底是什么样的字节顺序，只要使用函数给定值进行网络字节序和主机字节序的转换就可以了。</p><p>你可以想象，如果碰巧我们的系统本身是大端字节序，和网络字节序一样，那么使用上述所有的函数进行转换的时候，结果都仅仅是一个空实现，直接返回。</p><p>比如这样：</p><pre><code># if __BYTE_ORDER == __BIG_ENDIAN
/* The host byte order is the same as network byte order,
   so these functions are all just identity.  */
# define ntohl(x) (x)
# define ntohs(x) (x)
# define htonl(x) (x)
# define htons(x) (x)
</code></pre><h2>报文读取和解析</h2><p>应该看到，报文是以字节流的形式呈现给应用程序的，那么随之而来的一个问题就是，应用程序如何解读字节流呢？</p><p>这就要说到报文格式和解析了。报文格式实际上定义了字节的组织形式，发送端和接收端都按照统一的报文格式进行数据传输和解析，这样就可以保证彼此能够完成交流。</p><p>只有知道了报文格式，接收端才能针对性地进行报文读取和解析工作。</p><p>报文格式最重要的是如何确定报文的边界。常见的报文格式有两种方法，一种是发送端把要发送的报文长度预先通过报文告知给接收端；另一种是通过一些特殊的字符来进行边界的划分。</p><h2>显式编码报文长度</h2><h3>报文格式</h3><p>下面我们来看一个例子，这个例子是把要发送的报文长度预先通过报文告知接收端：</p><p><img src="https://static001.geekbang.org/resource/image/33/15/33805892d57843a1f22830d8636e1315.png?wh=1304*146" alt=""><br>
由图可以看出，这个报文的格式很简单，首先4个字节大小的消息长度，其目的是将真正发送的字节流的大小显式通过报文告知接收端，接下来是4个字节大小的消息类型，而真正需要发送的数据则紧随其后。</p><h3>发送报文</h3><p>发送端的程序如下：</p><pre><code>int main(int argc, char **argv) {
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

    struct {
        u_int32_t message_length;
        u_int32_t message_type;
        char buf[128];
    } message;

    int n;

    while (fgets(message.buf, sizeof(message.buf), stdin) != NULL) {
        n = strlen(message.buf);
        message.message_length = htonl(n);
        message.message_type = 1;
        if (send(socket_fd, (char *) &amp;message, sizeof(message.message_length) + sizeof(message.message_type) + n, 0) &lt;
            0)
            error(1, errno, &quot;send failure&quot;);

    }
    exit(0);
}
</code></pre><p>程序的1-20行是常规的创建套接字和地址，建立连接的过程。我们重点往下看，21-25行就是图示的报文格式转化为结构体，29-37行从标准输入读入数据，分别对消息长度、类型进行了初始化，注意这里使用了htonl函数将字节大小转化为了网络字节顺序，这一点很重要。最后我们看到23行实际发送的字节流大小为消息长度4字节，加上消息类型4字节，以及标准输入的字符串大小。</p><h3>解析报文：程序</h3><p>下面给出的是服务器端的程序，和客户端不一样的是，服务器端需要对报文进行解析。</p><pre><code>static int count;

static void sig_int(int signo) {
    printf(&quot;\nreceived %d datagrams\n&quot;, count);
    exit(0);
}


int main(int argc, char **argv) {
    int listenfd;
    listenfd = socket(AF_INET, SOCK_STREAM, 0);

    struct sockaddr_in server_addr;
    bzero(&amp;server_addr, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    server_addr.sin_port = htons(SERV_PORT);

    int on = 1;
    setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, &amp;on, sizeof(on));

    int rt1 = bind(listenfd, (struct sockaddr *) &amp;server_addr, sizeof(server_addr));
    if (rt1 &lt; 0) {
        error(1, errno, &quot;bind failed &quot;);
    }

    int rt2 = listen(listenfd, LISTENQ);
    if (rt2 &lt; 0) {
        error(1, errno, &quot;listen failed &quot;);
    }

    signal(SIGPIPE, SIG_IGN);

    int connfd;
    struct sockaddr_in client_addr;
    socklen_t client_len = sizeof(client_addr);

    if ((connfd = accept(listenfd, (struct sockaddr *) &amp;client_addr, &amp;client_len)) &lt; 0) {
        error(1, errno, &quot;bind failed &quot;);
    }

    char buf[128];
    count = 0;

    while (1) {
        int n = read_message(connfd, buf, sizeof(buf));
        if (n &lt; 0) {
            error(1, errno, &quot;error read message&quot;);
        } else if (n == 0) {
            error(1, 0, &quot;client closed \n&quot;);
        }
        buf[n] = 0;
        printf(&quot;received %d bytes: %s\n&quot;, n, buf);
        count++;
    }

    exit(0);

}
</code></pre><p>这个程序1-41行创建套接字，等待连接建立部分和前面基本一致。我们重点看42-55行的部分。45-55行循环处理字节流，调用read_message函数进行报文解析工作，并把报文的主体通过标准输出打印出来。</p><h3>解析报文：readn函数</h3><p>在了解read_message工作原理之前，我们先来看第5讲就引入的一个函数：readn。这里一定要强调的是readn函数的语义，<strong>读取报文预设大小的字节</strong>，readn调用会一直循环，尝试读取预设大小的字节，如果接收缓冲区数据空，readn函数会阻塞在那里，直到有数据到达。</p><pre><code>size_t readn(int fd, void *buffer, size_t length) {
    size_t count;
    ssize_t nread;
    char *ptr;

    ptr = buffer;
    count = length;
    while (count &gt; 0) {
        nread = read(fd, ptr, count);

        if (nread &lt; 0) {
            if (errno == EINTR)
                continue;
            else
                return (-1);
        } else if (nread == 0)
            break;                /* EOF */

        count -= nread;
        ptr += nread;
    }
    return (length - count);        /* return &gt;= 0 */
}
</code></pre><p>readn函数中使用count来表示还需要读取的字符数，如果count一直大于0，说明还没有满足预设的字符大小，循环就会继续。第9行通过read函数来服务最多count个字符。11-17行针对返回值进行出错判断，其中返回值为0的情形是EOF，表示对方连接终止。19-20行要读取的字符数减去这次读到的字符数，同时移动缓冲区指针，这样做的目的是为了确认字符数是否已经读取完毕。</p><h3>解析报文: read_message函数</h3><p>有了readn函数作为基础，我们再看一下read_message对报文的解析处理：</p><pre><code>size_t read_message(int fd, char *buffer, size_t length) {
    u_int32_t msg_length;
    u_int32_t msg_type;
    int rc;

    rc = readn(fd, (char *) &amp;msg_length, sizeof(u_int32_t));
    if (rc != sizeof(u_int32_t))
        return rc &lt; 0 ? -1 : 0;
    msg_length = ntohl(msg_length);

    rc = readn(fd, (char *) &amp;msg_type, sizeof(msg_type));
    if (rc != sizeof(u_int32_t))
        return rc &lt; 0 ? -1 : 0;

    if (msg_length &gt; length) {
        return -1;
    }

    rc = readn(fd, buffer, msg_length);
    if (rc != msg_length)
        return rc &lt; 0 ? -1 : 0;
    return rc;
}
</code></pre><p>在这个函数中，第6行通过调用readn函数获取4个字节的消息长度数据，紧接着，第11行通过调用readn函数获取4个字节的消息类型数据。第15行判断消息的长度是不是太大，如果大到本地缓冲区不能容纳，则直接返回错误；第19行调用readn一次性读取已知长度的消息体。</p><h3>实验</h3><p>我们依次启动作为报文解析的服务器一端，以及作为报文发送的客户端。我们看到，每次客户端发送的报文都可以被服务器端解析出来，在标准输出上的结果验证了这一点。</p><pre><code>$./streamserver
received 8 bytes: network
received 5 bytes: good
</code></pre><pre><code>$./streamclient
network
good
</code></pre><h2>特殊字符作为边界</h2><p>前面我提到了两种报文格式，另外一种报文格式就是通过设置特殊字符作为报文边界。HTTP是一个非常好的例子。</p><p><img src="https://static001.geekbang.org/resource/image/6d/5a/6d91c7c2a0224f5d4bad32a0f488765a.png?wh=942*324" alt=""><br>
HTTP通过设置回车符、换行符作为HTTP报文协议的边界。</p><p>下面的read_line函数就是在尝试读取一行数据，也就是读到回车符<code>\r</code>，或者读到回车换行符<code>\r\n</code>为止。这个函数每次尝试读取一个字节，第9行如果读到了回车符<code>\r</code>，接下来在11行的“观察”下看有没有换行符，如果有就在第12行读取这个换行符；如果没有读到回车符，就在第16-17行将字符放到缓冲区，并移动指针。</p><pre><code>int read_line(int fd, char *buf, int size) {
    int i = 0;
    char c = '\0';
    int n;

    while ((i &lt; size - 1) &amp;&amp; (c != '\n')) {
        n = recv(fd, &amp;c, 1, 0);
        if (n &gt; 0) {
            if (c == '\r') {
                n = recv(fd, &amp;c, 1, MSG_PEEK);
                if ((n &gt; 0) &amp;&amp; (c == '\n'))
                    recv(fd, &amp;c, 1, 0);
                else
                    c = '\n';
            }
            buf[i] = c;
            i++;
        } else
            c = '\n';
    }
    buf[i] = '\0';

    return (i);
}
</code></pre><h2>总结</h2><p>和我们预想的不太一样，TCP数据流特性决定了字节流本身是没有边界的，一般我们通过显式编码报文长度的方式，以及选取特殊字符区分报文边界的方式来进行报文格式的设计。而对报文解析的工作就是要在知道报文格式的情况下，有效地对报文信息进行还原。</p><h2>思考题</h2><p>和往常一样，这里给你留两道思考题，供你消化今天的内容。</p><p>第一道题关于HTTP的报文格式，我们看到，既要处理只有回车的情景，也要处理同时有回车和换行的情景，你知道造成这种情况的原因是什么吗？</p><p>第二道题是，我们这里讲到的报文格式，和TCP分组的报文格式，有什么区别和联系吗？</p><p>欢迎你在评论区写下你的思考，也欢迎把这篇文章分享给你的朋友或者同事，与他们一起交流一下这两个问题吧。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c8/6b/0f3876ef.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>iron_man</span>
  </div>
  <div class="_2_QraFYR_0">一直有个疑问，趁这堂课向老师请教一下，前面客户端发送消息时，消息长度转成网络序了，后面的消息为何没有转成网络序，如果消息里面含有数字呢？如果消息里面全是字符呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 非常好的问题。<br><br>我们在网络传输中，一个常见的方法是把0-9这样的数字，直接用ASCII码作为字符发送出去，在这种情况下，你可以理解成发送出去的都是字符类型的数据，因为是字符类型的数据，就没有所谓的网络顺序了；而如果作为一个数据型数据，比如125，这时候可能就要作为一个4字节的整型数据进行传输，那么就会有字节序的问题了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-07 16:06:55</div>
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
  <div class="_2_QraFYR_0">第一个是不是跟windows文本换行与linux文本换行的字符不同有关 windows上好像是一个换行符一个回车符 linux是一个换行符</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-06 13:09:08</div>
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
  <div class="_2_QraFYR_0">message.message_length = htonl( n );<br>message.message_type = 1;<br>我在自己写代码实现的时候突然想起这两句代码为什么一个需要htonl一个不需要呢？<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我觉得是我错了，应该都需要的，因为我定义的MESSAGE_TYPE是一个int型值。好提醒，能pull一个PR过来么？要不我自己来吧。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-06 14:20:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/93/02/fcab58d1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>JasonZhi</span>
  </div>
  <div class="_2_QraFYR_0">老师，针对粘包问题会有相关的讲解吗？经常会听说相关的名词，但是还是不太懂具体是怎么样的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我理解所谓的粘包是数据报文的边界确定不清晰，造成报文解析的时候有overlap，数据解析不对。<br><br>这个具体的解法讲义里也都有涉及到，通过合理设置报文边界，接收端缓冲报文并在解析时注意上下文，一般不会有大的问题。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-29 23:57:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/ub4icibeRLzff8Nf6ORsolib9KHtmeu3d4cCCAFd3Xgah3v78WfDYQB7WKq9iaIPXPwHBxw7mkBP9wYxDGMT9m1Rbw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>wyf2317</span>
  </div>
  <div class="_2_QraFYR_0">1. windos 和mac linux 的换行不一样。\r 或\r\n<br>2. 所属的层级不一样，应用层和网络层 <br>但本质上就是人为预定的对二进制数据序列化的方式，没有太大的差别，都是通信协议。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-08 20:39:51</div>
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
  <div class="_2_QraFYR_0">老师，关于网络序和主机序，我有3个问题想要请教一下：<br>1.在数据发送的时候，是先发送内存中高地址的数据，还是先发内存中低地址的数据？比如char* sendline=&quot;abcdef&quot;，是从a到f的顺序去发，还是从f到a的顺序去发。<br>2.接受的时候，是现接受到的是网络序中的高地址，还是低地址？<br>3.socket接口，如read，send这些函数，会自动帮我们完成主机序和网络序之间的转换吗，还是必须要自己去转？我看老师你有些数据显式调用了htonl()，有些没有，这是为什么呢？<br>谢谢老师。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1.如果是字符类型数据，肯定是从a到f这样的顺序拷贝到发送缓冲区发送的；<br>2.网络中没有高地址或者低地址，网络中传输的是一个字节流，就像你例子里的abcdef....这样的顺序字节流；<br>3.不会。对于数据型的数据如int需要调研htonl来转换，对于字符类型的数据，不需要转换。因为字符类型的数据，本质是ASCII编码，而int类型的数据则需要决定顺序。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-07 00:53:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/cb/61/b62d8a3b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张立华</span>
  </div>
  <div class="_2_QraFYR_0">我的操作系统是：centos 7.4 64位操作系统。<br><br>short int = 258;<br><br>258=0x0102<br><br>x的地址是（每次运行地址不一样）： 0x7fffffffe33e<br><br>258在内存中：<br><br>低位					 高位						 <br>0x7fffffffe33e			0x7fffffffe33f<br>00000010				00000001<br><br>也就是说，在我的linux电脑上，内存的数据，是小端字节序<br><br>可以写个简单的程序，用gdb调试下，通过 x命令查看内存</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-06 15:28:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/17/96/a10524f5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>计算机漫游</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，您在第一位留言中有如下回答：“我们在网络传输中，一个常见的方法是把0-9这样的数字，直接用ASCII码作为字符发送出去，在这种情况下，你可以理解成发送出去的都是字符类型的数据，因为是字符类型的数据，就没有所谓的网络顺序了”。我对此有些疑问，要说现在网络上普遍以UTF-8编码进行传输的话（而UTF-8是单字节码元，因此字节序无关），我能理解您说的“无所谓网络顺序“，但是如果以其他编码方式传输字符呢？所以我有两个问题：<br>1. 如过通信两端采用UTF-16、UFT-32这些多字节码元编码方式传输是否存在字节序问题？<br>2.字符集编码是否是socket要考虑的问题？我理解socket只负责传输字节流，编码解码由通信两端完成，不知是否正确？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我的意思是，数字可以直接按照二级制进行编码，也可以按照ASCII来进行字符编码，如果是按照字符来进行编码，我认为是没有字节顺序的，只需要把接收到的byte流按照编码格式进行解码即可。<br><br>你的理解是对的，编码解码是需要应用程序来完成的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-05 15:12:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/2b/84/07f0c0d6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>supermouse</span>
  </div>
  <div class="_2_QraFYR_0">思考题第一题：本来想说是因为Unix下的文件的行尾只有\n，而Windows下的文件行尾是\r\n，但是发现老师的代码里考虑的“\r”和“\r\n”这两种情况。所以这一题的答案是考虑到操作系统不同吗？<br>思考题第二题：区别的话应该是所属层级不同吧，我们自己定义的报文格式是用于应用层，而TCP分组的报文格式是用于传输层；而联系就在于，我们自己定义的报文格式是包含在TCP分组的报文格式中的，即TCP分组报文去掉消息头之后，得到的消息体的格式就是我们自己定义的报文格式</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对于第一个，确实在服务器端要考虑的，因为你不知道你的客户端是谁。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-18 16:00:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/cd/aa/33d48789.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>卫江</span>
  </div>
  <div class="_2_QraFYR_0">问题1，window与linux平台对于回车换行的编码不一致。<br>问题2，协议本质来说就是大家协商好，便于沟通的内容形式。所以，tcp与我们自定义的协议本质来说没有什么不用，区别只是针对的业务不同而已！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-06 21:36:20</div>
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
  <div class="_2_QraFYR_0">为什么需要进行端序转换？<br>因为数据传输、存储的最小单位是字节，<br>当我想传输的数据需要一个以上字节才能表示的时候，比如int 类型的 123,<br>这时接收端收到的是按顺序的四个字节,<br>他需要知道如何用这四个字节来还原成一个int,<br>端序转换指定了这个方法，<br>当然，如果传输的是一个字节就能表示的char类型,就不需要转换了<br><br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 正解。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-30 16:35:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83errIIarFicghpKamvkUaJmGdIV488iaOUyUqcTwbQ6IeRS40ZFfIOfb369fgleydAT8pkucHuj2x45A/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>xupeng1644</span>
  </div>
  <div class="_2_QraFYR_0">老师 客户端发送message时 为什么不将messsage_type也转换成网络字节序 而只将message_length转换成网络字节序       </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我认为你是对的，type类型确实也需要转为网络字节顺序。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-15 19:08:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/e0/93/b79a44b8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>zjstone</span>
  </div>
  <div class="_2_QraFYR_0"><br>read_line函数用很多次read操作，效率很低，老师应该发个高效率的版本：)</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你有什么更好的思路么？多次read调用在网络程序开发中很正常。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-08 16:46:38</div>
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
  <div class="_2_QraFYR_0">我们在网络传输中，一个常见的方法是把0-9这样的数字，直接用ASCII码作为字符发送出去，在这种情况下，你可以理解成发送出去的都是字符类型的数据，<br><br>老师，对这段回答，再加上我们项目现在划分微服务，我一直有疑问。我们服务间现在都是用grpc，而不用基于http的restful服务。因为考虑字符文本传输效率是最低的，体积大。比如本例当中125，如果作为数字传输，不是明显1个字节就可以了，如果用asc要3个字节，如果作为中文unicode，好像6个字节。<br>而我们传输的 数据对象中，既有字符类型字段(有中文文本)，也有数字字段。这种情况下，协议栈是怎么传输的了，整体作为字符传送？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这是编码问题，不是网络协议栈的问题。像你说的，可以用UTF-8编码，也可以使用GBK，如果你仔细研究他们，你会发现普遍的现象是，数字、常用字母都是遵循ASCII编码的，也就是8个bit，一个字节就可以搞定，而中文字符一般都是3个字节或以上。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-02 12:20:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/8b/ec/dc03f5ad.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张天屹</span>
  </div>
  <div class="_2_QraFYR_0">有点疑惑  对于两种方案，不管是数字还是换行符，都有可能在报文的内容中出现，怎么区分是报文正文还是分隔符呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这就要仔细设置报文了，你没有注意到回车和换行符在http里都是要escape掉么？也就是转义掉，以避免和真正用来做报文分隔的字符冲突了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-02 13:52:32</div>
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
  <div class="_2_QraFYR_0">老师，关于大小端的问题，服务端和客户端互相发包的情况下，为了安全起见，是不是都应该统一进行大小端处理？比如发包都得先转成大端数据，收包再转成机器的顺序？(无论是字符数据还是数值数据)</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的。实际上就是这样。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-04 09:04:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/a5/73/3ddc7c77.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Brave Shine</span>
  </div>
  <div class="_2_QraFYR_0">1. linux和windows在编码http层面的不同？<br>2. tcp的分组报文是传输层在协议栈层面怎么发tcp包的规范，这里指的是应用层报文</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-06 13:06:56</div>
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
  <div class="_2_QraFYR_0">第1问:<br>大胆猜测因为http是以回车换行作为分界符但是在程序在编码过程中有可能随时只产生一个回车符<br>第2问:<br>tcp分组格式 是tcp层面的东西 而本文提到的报文是应用层方面的东西,联系在于tcp层面的分组可能导致数据流出现意想不到的情况,只有通过应用层处理成想要的情况<br><br>我记得之前公司解析字节流的时候是以字符0作为结束符,通过字符0 分割出一段数据流 然后通过类型转换 比如( char* ) pbuff转换成字符串或者其他类型的数据,编码的时候也是把数据的编码进去最后添加一个字符0</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你们这么狠的，用0来作为边界符？0作为字符串截止符是有自己特殊含义的，不过，你们开心就好。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-06 11:24:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2a/78/06/934b1cd6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>DongGu</span>
  </div>
  <div class="_2_QraFYR_0">想问一下，是不是发送的数据中，有数值类型的，都要改变大小端之后再发送出去，包括接收</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-04 19:11:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/10/bb/f1061601.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Demon.Lee</span>
  </div>
  <div class="_2_QraFYR_0">老师，这里讲的报文格式，报文的读取与解析，可以被理解为是序列化与反序列化吗？<br><br>“将对象的类型、属性类型、属性值一一按照固定的格式写到二进制字节流中来完成序列化，再按照固定的格式一一读出对象的类型、属性类型、属性值，通过这些信息重新创建出一个新的对象，来完成反序列化。”</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-29 20:25:28</div>
  </div>
</div>
</div>
</li>
</ul>