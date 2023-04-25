<audio title="21 _ poll：另一种IO多路复用" src="https://static001.geekbang.org/resource/audio/c4/37/c42fac60c23b348ecd1ecf46ef1f3937.mp3" controls="controls"></audio> 
<p>你好，我是盛延敏，这是网络编程实战第21讲，欢迎回来。</p><p>上一讲我们讲到了I/O多路复用技术，并以select为核心，展示了I/O多路复用技术的能力。select方法是多个UNIX平台支持的非常常见的I/O多路复用技术，它通过描述符集合来表示检测的I/O对象，通过三个不同的描述符集合来描述I/O事件 ：可读、可写和异常。但是select有一个缺点，那就是所支持的文件描述符的个数是有限的。在Linux系统中，select的默认最大值为1024。</p><p>那么有没有别的I/O多路复用技术可以突破文件描述符个数限制呢？当然有，这就是poll函数。这一讲，我们就来学习一下另一种I/O多路复用的技术：poll。</p><h2>poll函数介绍</h2><p>poll是除了select之外，另一种普遍使用的I/O多路复用技术，和select相比，它和内核交互的数据结构有所变化，另外，也突破了文件描述符的个数限制。</p><p>下面是poll函数的原型：</p><pre><code>int poll(struct pollfd *fds, unsigned long nfds, int timeout); 
　　　
返回值：若有就绪描述符则为其数目，若超时则为0，若出错则为-1
</code></pre><p>这个函数里面输入了三个参数，第一个参数是一个pollfd的数组。其中pollfd的结构如下：</p><pre><code>struct pollfd {
    int    fd;       /* file descriptor */
    short  events;   /* events to look for */
    short  revents;  /* events returned */
 };
</code></pre><p>这个结构体由三个部分组成，首先是描述符fd，然后是描述符上待检测的事件类型events，注意这里的events可以表示多个不同的事件，具体的实现可以通过使用二进制掩码位操作来完成，例如，POLLIN和POLLOUT可以表示读和写事件。</p><!-- [[[read_end]]] --><pre><code>#define    POLLIN    0x0001    /* any readable data available */
#define    POLLPRI   0x0002    /* OOB/Urgent readable data */
#define    POLLOUT   0x0004    /* file descriptor is writeable */
</code></pre><p>和select非常不同的地方在于，poll每次检测之后的结果不会修改原来的传入值，而是将结果保留在revents字段中，这样就不需要每次检测完都得重置待检测的描述字和感兴趣的事件。我们可以把revents理解成“returned events”。</p><p>events类型的事件可以分为两大类。</p><p>第一类是可读事件，有以下几种：</p><pre><code>#define POLLIN     0x0001    /* any readable data available */
#define POLLPRI    0x0002    /* OOB/Urgent readable data */
#define POLLRDNORM 0x0040    /* non-OOB/URG data available */
#define POLLRDBAND 0x0080    /* OOB/Urgent readable data */
</code></pre><p>一般我们在程序里面有POLLIN即可。套接字可读事件和select的readset基本一致，是系统内核通知应用程序有数据可以读，通过read函数执行操作不会被阻塞。</p><p>第二类是可写事件，有以下几种：</p><pre><code>#define POLLOUT    0x0004    /* file descriptor is writeable */
#define POLLWRNORM POLLOUT   /* no write type differentiation */
#define POLLWRBAND 0x0100    /* OOB/Urgent data can be written */
</code></pre><p>一般我们在程序里面统一使用POLLOUT。套接字可写事件和select的writeset基本一致，是系统内核通知套接字缓冲区已准备好，通过write函数执行写操作不会被阻塞。</p><p>以上两大类的事件都可以在“returned events”得到复用。还有另一大类事件，没有办法通过poll向系统内核递交检测请求，只能通过“returned events”来加以检测，这类事件是各种错误事件。</p><pre><code>#define POLLERR    0x0008    /* 一些错误发送 */
#define POLLHUP    0x0010    /* 描述符挂起*/
#define POLLNVAL   0x0020    /* 请求的事件无效*/
</code></pre><p>我们再回过头看一下poll函数的原型。参数nfds描述的是数组fds的大小，简单说，就是向poll申请的事件检测的个数。</p><p>最后一个参数timeout，描述了poll的行为。</p><p>如果是一个&lt;0的数，表示在有事件发生之前永远等待；如果是0，表示不阻塞进程，立即返回；如果是一个&gt;0的数，表示poll调用方等待指定的毫秒数后返回。</p><p>关于返回值，当有错误发生时，poll函数的返回值为-1；如果在指定的时间到达之前没有任何事件发生，则返回0，否则就返回检测到的事件个数，也就是“returned events”中非0的描述符个数。</p><p>poll函数有一点非常好，如果我们<strong>不想对某个pollfd结构进行事件检测，</strong>可以把它对应的pollfd结构的fd成员设置成一个负值。这样，poll函数将忽略这样的events事件，检测完成以后，所对应的“returned events”的成员值也将设置为0。</p><p>和select函数对比一下，我们发现poll函数和select不一样的地方就是，在select里面，文件描述符的个数已经随着fd_set的实现而固定，没有办法对此进行配置；而在poll函数里，我们可以控制pollfd结构的数组大小，这意味着我们可以突破原来select函数最大描述符的限制，在这种情况下，应用程序调用者需要分配pollfd数组并通知poll函数该数组的大小。</p><h2>基于poll的服务器程序</h2><p>下面我们将开发一个基于poll的服务器程序。这个程序可以同时处理多个客户端连接，并且一旦有客户端数据接收后，同步地回显回去。这已经是一个颇具高并发处理的服务器原型了，再加上后面讲到的非阻塞I/O和多线程等技术，基本上就是可使用的准生产级别了。</p><p>所以，让我们打起精神，一起来看这个程序。</p><pre><code>#define INIT_SIZE 128

int main(int argc, char **argv) {
    int listen_fd, connected_fd;
    int ready_number;
    ssize_t n;
    char buf[MAXLINE];
    struct sockaddr_in client_addr;

    listen_fd = tcp_server_listen(SERV_PORT);

    //初始化pollfd数组，这个数组的第一个元素是listen_fd，其余的用来记录将要连接的connect_fd
    struct pollfd event_set[INIT_SIZE];
    event_set[0].fd = listen_fd;
    event_set[0].events = POLLRDNORM;

    // 用-1表示这个数组位置还没有被占用
    int i;
    for (i = 1; i &lt; INIT_SIZE; i++) {
        event_set[i].fd = -1;
    }

    for (;;) {
        if ((ready_number = poll(event_set, INIT_SIZE, -1)) &lt; 0) {
            error(1, errno, &quot;poll failed &quot;);
        }

        if (event_set[0].revents &amp; POLLRDNORM) {
            socklen_t client_len = sizeof(client_addr);
            connected_fd = accept(listen_fd, (struct sockaddr *) &amp;client_addr, &amp;client_len);

            //找到一个可以记录该连接套接字的位置
            for (i = 1; i &lt; INIT_SIZE; i++) {
                if (event_set[i].fd &lt; 0) {
                    event_set[i].fd = connected_fd;
                    event_set[i].events = POLLRDNORM;
                    break;
                }
            }

            if (i == INIT_SIZE) {
                error(1, errno, &quot;can not hold so many clients&quot;);
            }

            if (--ready_number &lt;= 0)
                continue;
        }

        for (i = 1; i &lt; INIT_SIZE; i++) {
            int socket_fd;
            if ((socket_fd = event_set[i].fd) &lt; 0)
                continue;
            if (event_set[i].revents &amp; (POLLRDNORM | POLLERR)) {
                if ((n = read(socket_fd, buf, MAXLINE)) &gt; 0) {
                    if (write(socket_fd, buf, n) &lt; 0) {
                        error(1, errno, &quot;write error&quot;);
                    }
                } else if (n == 0 || errno == ECONNRESET) {
                    close(socket_fd);
                    event_set[i].fd = -1;
                } else {
                    error(1, errno, &quot;read error&quot;);
                }

                if (--ready_number &lt;= 0)
                    break;
            }
        }
    }
}
</code></pre><p>当然，一开始需要创建一个监听套接字，并绑定在本地的地址和端口上，这在第10行调用tcp_server_listen函数来完成。</p><p>在第13行，我初始化了一个pollfd数组，并命名为event_set，之所以叫这个名字，是引用pollfd数组确实代表了检测的事件集合。这里数组的大小固定为INIT_SIZE，这在实际的生产环境肯定是需要改进的。</p><p>我在前面讲过，监听套接字上如果有连接建立完成，也是可以通过 I/O事件复用来检测到的。在第14-15行，将监听套接字listen_fd和对应的POLLRDNORM事件加入到event_set里，表示我们期望系统内核检测监听套接字上的连接建立完成事件。</p><p>在前面介绍poll函数时，我们提到过，如果对应pollfd里的文件描述字fd为负数，poll函数将会忽略这个pollfd，所以我们在第18-21行将event_set数组里其他没有用到的fd统统设置为-1。这里-1也表示了当前pollfd没有被使用的意思。</p><p>下面我们的程序进入一个无限循环，在这个循环体内，第24行调用poll函数来进行事件检测。poll函数传入的参数为event_set数组，数组大小INIT_SIZE和-1。这里之所以传入INIT_SIZE，是因为poll函数已经能保证可以自动忽略fd为-1的pollfd，否则我们每次都需要计算一下event_size里真正需要被检测的元素大小；timeout设置为-1，表示在I/O事件发生之前poll调用一直阻塞。</p><p>如果系统内核检测到监听套接字上的连接建立事件，就进入到第28行的判断分支。我们看到，使用了如event_set[0].revent来和对应的事件类型进行位与操作，这个技巧大家一定要记住，这是因为event都是通过二进制位来进行记录的，位与操作是和对应的二进制位进行操作，一个文件描述字是可以对应到多个事件类型的。</p><p>在这个分支里，调用accept函数获取了连接描述字。接下来，33-38行做了一件事，就是把连接描述字connect_fd也加入到event_set里，而且说明了我们感兴趣的事件类型为POLLRDNORM，也就是套接字上有数据可以读。在这里，我们从数组里查找一个没有没占用的位置，也就是fd为-1的位置，然后把fd设置为新的连接套接字connect_fd。</p><p>如果在数组里找不到这样一个位置，说明我们的event_set已经被很多连接充满了，没有办法接收更多的连接了，这就是第41-42行所做的事情。</p><p>第45-46行是一个加速优化能力，因为poll返回的一个整数，说明了这次I/O事件描述符的个数，如果处理完监听套接字之后，就已经完成了这次I/O复用所要处理的事情，那么我们就可以跳过后面的处理，再次进入poll调用。</p><p>接下来的循环处理是查看event_set里面其他的事件，也就是已连接套接字的可读事件。这是通过遍历event_set数组来完成的。</p><p>如果数组里的pollfd的fd为-1，说明这个pollfd没有递交有效的检测，直接跳过；来到第53行，通过检测revents的事件类型是POLLRDNORM或者POLLERR，我们可以进行读操作。在第54行，读取数据正常之后，再通过write操作回显给客户端；在第58行，如果读到EOF或者是连接重置，则关闭这个连接，并且把event_set对应的pollfd重置；第61行读取数据失败。</p><p>和前面的优化加速处理一样，第65-66行是判断如果事件已经被完全处理完之后，直接跳过对event_set的循环处理，再次来到poll调用。</p><h2>实验</h2><p>我们启动这个服务器程序，然后通过telnet连接到这个服务器程序。为了检验这个服务器程序的I/O复用能力，我们可以多开几个telnet客户端，并且在屏幕上输入各种字符串。</p><p>客户端1：</p><pre><code>$telnet 127.0.0.1 43211
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
a
a
aaaaaaaaaaa
aaaaaaaaaaa
afafasfa
afafasfa
fbaa
fbaa
^]


telnet&gt; quit
Connection closed.
</code></pre><p>客户端2：</p><pre><code>telnet 127.0.0.1 43211
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
b
b
bbbbbbb
bbbbbbb
bbbbbbb
bbbbbbb
^]


telnet&gt; quit
Connection closed.
</code></pre><p>可以看到，这两个客户端互不影响，每个客户端输入的字符很快会被回显到客户端屏幕上。一个客户端断开连接，也不会影响到其他客户端。</p><h2>总结</h2><p>poll是另一种在各种UNIX系统上被广泛支持的I/O多路复用技术，虽然名声没有select那么响，能力一点不比select差，而且因为可以突破select文件描述符的个数限制，在高并发的场景下尤其占优势。这一讲我们编写了一个基于poll的服务器程序，希望你从中学会poll的用法。</p><h2>思考题</h2><p>和往常一样，给你留两道思考题：</p><p>第一道，在我们的程序里event_set数组的大小固定为INIT_SIZE，这在实际的生产环境肯定是需要改进的。你知道如何改进吗？</p><p>第二道，如果我们进行了改进，那么接下来把连接描述字connect_fd也加入到event_set里，如何配合进行改造呢？</p><p>欢迎你在评论区写下你的思考，也欢迎把这篇文章分享给你的朋友或者同事，一起交流一下。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/81/4e/d71092f4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>夏目</span>
  </div>
  <div class="_2_QraFYR_0">老师，我还是没明白poll和select的本质区别是什么，能否指点一下</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 两者只是编程接口的区别，从内核实现角度来讲，其实本质实现是差不多的，poll客服了select有限文件描述字的缺陷，适用的范围更广一些。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-15 00:39:54</div>
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
  <div class="_2_QraFYR_0">1.采用动态分配数组的方式<br>2.如果内存不够 进行realloc 或者申请一块更大的内存 然后把源数组拷贝过来</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 鼓励动手来一个。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-25 07:54:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/99/27/47aa9dea.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>阿卡牛</span>
  </div>
  <div class="_2_QraFYR_0">还有种信号驱动型I&#47;O，老师可以讲解吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 让内核在描述符就绪时发送SIGIO信号通知我们，这种模型为信号驱动式I&#47;O（signal-driven I&#47;O)，说实话，这个模型在实战中用的是比较少的，作为一个知识点知道就可以了。<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-24 07:44:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/3d/03/b2d9a084.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Hale</span>
  </div>
  <div class="_2_QraFYR_0">能讲讲为什么不用POLLIN来判断套接字可读？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: POLLIN包括了OOB等带外数据的检测，POLLRDNORM则不包括这部分。<br><br>#define	POLLIN		0x0001		&#47;* any readable data available *&#47;<br>#define	POLLRDNORM	0x0040		&#47;* non-OOB&#47;URG data available *&#47;<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-26 09:23:04</div>
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
  <div class="_2_QraFYR_0">老师可否简单讲下底层实现，比如底层是数组，队列，红黑树等。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 好问题，我收集一下素材。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-25 19:45:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/9b/2d/f7fca208.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>fedwing</span>
  </div>
  <div class="_2_QraFYR_0">老师，请教个问题，我看ready_number在29行的if里如果有会--，后面read for循环里，如果处理也--，我是不是可以这样理解，events_set[0]表示listen的套接字，这个套接字里如果有pollin，那么肯定是新连接（而不是普通套接字的读数据），所以这时就是获取对应的连接的文件描述符，将其加入到event_set数组里，用于后续poll的时候，多检测一个文件描述符，如果ready_number在前面的处理--后，还大于0，则表示events_set里其他的文件描述符也有待检测的事件触发，这些就是常规的双端连接对应的套接字，它们pollin的话，就是我们常规意义里的read数据了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-12 15:54:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83eqf54z1ZmqQY1kmJ6t1HAnrqMM3j6WKf0oDeVLhtnA2ZUKY6AX9MK6RjvcO8SiczXy3uU0IzBQ3tpw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_68d3d2</span>
  </div>
  <div class="_2_QraFYR_0">老师我看网络编程里面使用了各种函数，函数里面各种参数，您那里有没有什么文档参考手册啥的可供我们需要时翻阅，光靠脑子记，记不来啊。您平常都是怎么写代码啊，这些函数都是背下来了吗。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Linux下&quot;man xxx&quot;，windows下看MSDN，当然，有一些常见的是要记下来的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-11 22:19:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/fa/84/f01d203a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Simple life</span>
  </div>
  <div class="_2_QraFYR_0">我搞不懂，accept后的fd要加入event_set，然后再遍历取出，直接拿来读写不行吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 因为我们在同时处理多个I&#47;O，一旦一个fd经过accept处理后加入event_set，之后就可以通过一个poll调用来获取多个不同的fd来加以处理。这是event_set的意义。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-31 19:00:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKUcSLVV6ia3dibe7qvTu8Vic1PVs2EibxoUdx930MC7j2Q9A6s4eibMDZlcicMFY0D0icd3RrDorMChu0zw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Tesla</span>
  </div>
  <div class="_2_QraFYR_0">老师 poll不改变传入检测的event的状态，而是返回revent，是出于什么目的呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个设计很合理，一个是输入参数，一个是输出参数，只不多在同一个结构体内。如果只有一个参数，既是输入，也是输出，反到有点奇怪。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-05 12:56:53</div>
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
  <div class="_2_QraFYR_0">第一问: 我觉得需要改进的原因在于他是一个固定死了的值,而很多时候我们都要考虑到扩容的问题,所以可以把所有的描述符push_back到一个vector等类似的容器当中,直接对容器取size就可以获得数量<br>第二问:把新连接上来的connfd添加进去,对上面问题的容器进行一次取size操作就行了<br>通过前面两个问题 我产生了第三个问题<br>我们都知道select 每次循环都需要向内核重新注册一次需要关心的描述符, 在Poll当中他是怎么处理的呢？也是每次都要注册一次吗？新增了描述放到集合当中肯定也需要通知内核啊 ！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: poll也是每次向内核注册了一个描述符集合，做法没有区别。你看到的这段代码，就是新增的描述符<br><br>    &#47;&#47;找到一个可以记录该连接套接字的位置<br>            for (i = 1; i &lt; INIT_SIZE; i++) {<br>                if (event_set[i].fd &lt; 0) {<br>                    event_set[i].fd = connected_fd;<br>                    event_set[i].events = POLLRDNORM;<br>                    break;<br>                }<br>            }</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-30 14:47:56</div>
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
  <div class="_2_QraFYR_0">为什么程序里使用POLLRDNORM而不是POLLIN呢？这两者又何不同？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: #define	POLLRDNORM	0x0040		&#47;* non-OOB&#47;URG data available *&#47;<br>#define	POLLIN		0x0001		&#47;* any readable data available *&#47;<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-25 07:51:30</div>
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
  <div class="_2_QraFYR_0">1)第一道，可以用vector存储所有的连接描述符，然后当需要调用poll的时候，再用vector.size获取数组的大小，然后再创建出fd_set tmp[vector.size]存储所有需要的fd，将他传入到poll函数中。<br><br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-03 10:54:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTIcxz0quUK7Q06aNC3qglvvpTQKOanK3suG0qQkK00Q815zF5oiad1wABibCkm8Lk18LmX8UQoUMS5Q/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>panda</span>
  </div>
  <div class="_2_QraFYR_0">老师，什么情况下会使套接字数目多余select数目呢，我所理解的是一般服务端对一个套接字就会开一个线程，客户端一个进程也不会创建出很多套接字，感觉都不会导致数量过多的情况，求指点</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是这样的，当你写一个服务端程序，需要监听超过1024个客户端连接时，就会超过这个限制。客户端是没有问题的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-04 19:11:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/41/af/4307867a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>JJj</span>
  </div>
  <div class="_2_QraFYR_0">请问下，如果select同时关注可读、可写、异常。那是不是最多支持关注3*1024个IO事件</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你这样理解倒也是可以的。一个描述字可以对应三种不同的事件。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-16 11:37:25</div>
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
  <div class="_2_QraFYR_0">我还是不太明白select和poll进行事件注册的区别,希望老师再给我指点指点</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 区别是编程的接口不一样，原理基本一致，但是select一般来说有文件句柄的现在，poll则没有。我觉得你可以看代码体会一下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-30 16:57:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/61/36/7a28f544.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Jimmy Xiong</span>
  </div>
  <div class="_2_QraFYR_0">请问老师，例子的全代码（可以直接运行起来）哪里可以找得到？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: https:&#47;&#47;github.com&#47;froghui&#47;yolanda</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-28 11:14:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>pippin</span>
  </div>
  <div class="_2_QraFYR_0">套接字和文件描述符有什么区别<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 在Linux系统里，所有都是文件，所以套接字也是文件描述符。当然，文件描述符，也可以有别的，比如说文件、目录等。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-03 08:52:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2a/31/17/ab2c27a6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>菜鸡互啄</span>
  </div>
  <div class="_2_QraFYR_0">老师 28行不是太明白 如果listen_fd有可读事件 为什么说明有连接要accept了？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 因为listen_fd是监听套接字。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-09 21:20:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/28/c5/76/f7c24b63.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>你已经长大了，别皮</span>
  </div>
  <div class="_2_QraFYR_0">老师服务器程序在49-68处理连接事件时，此时如果有新连接来，ready_number是否会++，这样是否会死循环？还是内核会存储起来，等待下一次poll时再往上报？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 等待下一次poll。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-28 23:41:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/79/2d/dbb5570f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>huadanian</span>
  </div>
  <div class="_2_QraFYR_0">请问一下老师，上面代码第53行用于判断的revents的值，在第54行的read之后，是否需要清除掉，否则之后的循环会不会重复判断这个revents的值？？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不需要。每次poll之后，revents的值都是全新从内核拷贝到用户空间的。但是如果这个连接已经close了，需要把对应的fd置为-1。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-21 17:32:16</div>
  </div>
</div>
</div>
</li>
</ul>