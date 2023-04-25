<audio title="23 _ Linux利器：epoll的前世今生" src="https://static001.geekbang.org/resource/audio/48/6c/4820058629fb5514b1736a629b9fdd6c.mp3" controls="controls"></audio> 
<p>你好，我是盛延敏，这里是网络编程实战第23讲，欢迎回来。</p><p>性能篇的前三讲，非阻塞I/O加上I/O多路复用，已经渐渐帮助我们在高性能网络编程这个领域搭建了初步的基石。但是，离最终的目标还差那么一点，如果说I/O多路复用帮我们打开了高性能网络编程的窗口，那么今天的主题——epoll，将为我们增添足够的动力。</p><p>这里有放置了一张图，这张图来自The Linux Programming Interface(No Starch Press)。这张图直观地为我们展示了select、poll、epoll几种不同的I/O复用技术在面对不同文件描述符大小时的表现差异。</p><p><img src="https://static001.geekbang.org/resource/image/fd/60/fd2e25f72a5103ef78c05c7ad2dab060.png?wh=681*129" alt=""><br>
从图中可以明显地看到，epoll的性能是最好的，即使在多达10000个文件描述的情况下，其性能的下降和有10个文件描述符的情况相比，差别也不是很大。而随着文件描述符的增大，常规的select和poll方法性能逐渐变得很差。</p><p>那么，epoll究竟使用了什么样的“魔法”，取得了如此令人惊讶的效果呢？接下来，我们就来一起分析一下。</p><h2>epoll的用法</h2><p>在分析对比epoll、poll和select几种技术之前，我们先看一下怎么使用epoll来完成一个服务器程序，具体的原理我将在29讲中进行讲解。</p><!-- [[[read_end]]] --><p>epoll可以说是和poll非常相似的一种I/O多路复用技术，有些朋友将epoll归为异步I/O，我觉得这是不正确的。本质上epoll还是一种I/O多路复用技术， epoll通过监控注册的多个描述字，来进行I/O事件的分发处理。不同于poll的是，epoll不仅提供了默认的level-triggered（条件触发）机制，还提供了性能更为强劲的edge-triggered（边缘触发）机制。至于这两种机制的区别，我会在后面详细展开。</p><p>使用epoll进行网络程序的编写，需要三个步骤，分别是epoll_create，epoll_ctl和epoll_wait。接下来我对这几个API详细展开讲一下。</p><h3>epoll_create</h3><pre><code>int epoll_create(int size);
int epoll_create1(int flags);
        返回值: 若成功返回一个大于0的值，表示epoll实例；若返回-1表示出错
</code></pre><p>epoll_create()方法创建了一个epoll实例，从Linux 2.6.8开始，参数size被自动忽略，但是该值仍需要一个大于0的整数。这个epoll实例被用来调用epoll_ctl和epoll_wait，如果这个epoll实例不再需要，比如服务器正常关机，需要调用close()方法释放epoll实例，这样系统内核可以回收epoll实例所分配使用的内核资源。</p><p>关于这个参数size，在一开始的epoll_create实现中，是用来告知内核期望监控的文件描述字大小，然后内核使用这部分的信息来初始化内核数据结构，在新的实现中，这个参数不再被需要，因为内核可以动态分配需要的内核数据结构。我们只需要注意，每次将size设置成一个大于0的整数就可以了。</p><p>epoll_create1()的用法和epoll_create()基本一致，如果epoll_create1()的输入flags为0，则和epoll_create()一样，内核自动忽略。可以增加如EPOLL_CLOEXEC的额外选项，如果你有兴趣的话，可以研究一下这个选项有什么意义。</p><h3>epoll_ctl</h3><pre><code> int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
        返回值: 若成功返回0；若返回-1表示出错
</code></pre><p>在创建完epoll实例之后，可以通过调用epoll_ctl往这个epoll实例增加或删除监控的事件。函数epll_ctl有4个入口参数。</p><p>第一个参数epfd是刚刚调用epoll_create创建的epoll实例描述字，可以简单理解成是epoll句柄。</p><p>第二个参数表示增加还是删除一个监控事件，它有三个选项可供选择：</p><ul>
<li>EPOLL_CTL_ADD： 向epoll实例注册文件描述符对应的事件；</li>
<li>EPOLL_CTL_DEL：向epoll实例删除文件描述符对应的事件；</li>
<li>EPOLL_CTL_MOD： 修改文件描述符对应的事件。</li>
</ul><p>第三个参数是注册的事件的文件描述符，比如一个监听套接字。</p><p>第四个参数表示的是注册的事件类型，并且可以在这个结构体里设置用户需要的数据，其中最为常见的是使用联合结构里的fd字段，表示事件所对应的文件描述符。</p><pre><code>typedef union epoll_data {
     void        *ptr;
     int          fd;
     uint32_t     u32;
     uint64_t     u64;
 } epoll_data_t;

 struct epoll_event {
     uint32_t     events;      /* Epoll events */
     epoll_data_t data;        /* User data variable */
 };
</code></pre><p>我们在前面介绍poll的时候已经接触过基于mask的事件类型了，这里epoll仍旧使用了同样的机制，我们重点看一下这几种事件类型：</p><ul>
<li>EPOLLIN：表示对应的文件描述字可以读；</li>
<li>EPOLLOUT：表示对应的文件描述字可以写；</li>
<li>EPOLLRDHUP：表示套接字的一端已经关闭，或者半关闭；</li>
<li>EPOLLHUP：表示对应的文件描述字被挂起；</li>
<li>EPOLLET：设置为edge-triggered，默认为level-triggered。</li>
</ul><h3>epoll_wait</h3><pre><code>int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
  返回值: 成功返回的是一个大于0的数，表示事件的个数；返回0表示的是超时时间到；若出错返回-1.
</code></pre><p>epoll_wait()函数类似之前的poll和select函数，调用者进程被挂起，在等待内核I/O事件的分发。</p><p>这个函数的第一个参数是epoll实例描述字，也就是epoll句柄。</p><p>第二个参数返回给用户空间需要处理的I/O事件，这是一个数组，数组的大小由epoll_wait的返回值决定，这个数组的每个元素都是一个需要待处理的I/O事件，其中events表示具体的事件类型，事件类型取值和epoll_ctl可设置的值一样，这个epoll_event结构体里的data值就是在epoll_ctl那里设置的data，也就是用户空间和内核空间调用时需要的数据。</p><p>第三个参数是一个大于0的整数，表示epoll_wait可以返回的最大事件值。</p><p>第四个参数是epoll_wait阻塞调用的超时值，如果这个值设置为-1，表示不超时；如果设置为0则立即返回，即使没有任何I/O事件发生。</p><h2>epoll例子</h2><h3>代码解析</h3><p>下面我们把原先基于poll的服务器端程序改造成基于epoll的：</p><pre><code>#include &quot;lib/common.h&quot;

#define MAXEVENTS 128

char rot13_char(char c) {
    if ((c &gt;= 'a' &amp;&amp; c &lt;= 'm') || (c &gt;= 'A' &amp;&amp; c &lt;= 'M'))
        return c + 13;
    else if ((c &gt;= 'n' &amp;&amp; c &lt;= 'z') || (c &gt;= 'N' &amp;&amp; c &lt;= 'Z'))
        return c - 13;
    else
        return c;
}

int main(int argc, char **argv) {
    int listen_fd, socket_fd;
    int n, i;
    int efd;
    struct epoll_event event;
    struct epoll_event *events;

    listen_fd = tcp_nonblocking_server_listen(SERV_PORT);

    efd = epoll_create1(0);
    if (efd == -1) {
        error(1, errno, &quot;epoll create failed&quot;);
    }

    event.data.fd = listen_fd;
    event.events = EPOLLIN | EPOLLET;
    if (epoll_ctl(efd, EPOLL_CTL_ADD, listen_fd, &amp;event) == -1) {
        error(1, errno, &quot;epoll_ctl add listen fd failed&quot;);
    }

    /* Buffer where events are returned */
    events = calloc(MAXEVENTS, sizeof(event));

    while (1) {
        n = epoll_wait(efd, events, MAXEVENTS, -1);
        printf(&quot;epoll_wait wakeup\n&quot;);
        for (i = 0; i &lt; n; i++) {
            if ((events[i].events &amp; EPOLLERR) ||
                (events[i].events &amp; EPOLLHUP) ||
                (!(events[i].events &amp; EPOLLIN))) {
                fprintf(stderr, &quot;epoll error\n&quot;);
                close(events[i].data.fd);
                continue;
            } else if (listen_fd == events[i].data.fd) {
                struct sockaddr_storage ss;
                socklen_t slen = sizeof(ss);
                int fd = accept(listen_fd, (struct sockaddr *) &amp;ss, &amp;slen);
                if (fd &lt; 0) {
                    error(1, errno, &quot;accept failed&quot;);
                } else {
                    make_nonblocking(fd);
                    event.data.fd = fd;
                    event.events = EPOLLIN | EPOLLET; //edge-triggered
                    if (epoll_ctl(efd, EPOLL_CTL_ADD, fd, &amp;event) == -1) {
                        error(1, errno, &quot;epoll_ctl add connection fd failed&quot;);
                    }
                }
                continue;
            } else {
                socket_fd = events[i].data.fd;
                printf(&quot;get event on socket fd == %d \n&quot;, socket_fd);
                while (1) {
                    char buf[512];
                    if ((n = read(socket_fd, buf, sizeof(buf))) &lt; 0) {
                        if (errno != EAGAIN) {
                            error(1, errno, &quot;read error&quot;);
                            close(socket_fd);
                        }
                        break;
                    } else if (n == 0) {
                        close(socket_fd);
                        break;
                    } else {
                        for (i = 0; i &lt; n; ++i) {
                            buf[i] = rot13_char(buf[i]);
                        }
                        if (write(socket_fd, buf, n) &lt; 0) {
                            error(1, errno, &quot;write error&quot;);
                        }
                    }
                }
            }
        }
    }

    free(events);
    close(listen_fd);
}
</code></pre><p>程序的第23行调用epoll_create0创建了一个epoll实例。</p><p>28-32行，调用epoll_ctl将监听套接字对应的I/O事件进行了注册，这样在有新的连接建立之后，就可以感知到。注意这里使用的是edge-triggered（边缘触发）。</p><p>35行为返回的event数组分配了内存。</p><p>主循环调用epoll_wait函数分发I/O事件，当epoll_wait成功返回时，通过遍历返回的event数组，就直接可以知道发生的I/O事件。</p><p>第41-46行判断了各种错误情况。</p><p>第47-61行是监听套接字上有事件发生的情况下，调用accept获取已建立连接，并将该连接设置为非阻塞，再调用epoll_ctl把已连接套接字对应的可读事件注册到epoll实例中。这里我们使用了event_data里面的fd字段，将连接套接字存储其中。</p><p>第63-84行，处理了已连接套接字上的可读事件，读取字节流，编码后再回应给客户端。</p><h3>实验</h3><p>启动该服务器：</p><pre><code>$./epoll01
epoll_wait wakeup
epoll_wait wakeup
epoll_wait wakeup
get event on socket fd == 6
epoll_wait wakeup
get event on socket fd == 5
epoll_wait wakeup
get event on socket fd == 5
epoll_wait wakeup
get event on socket fd == 6
epoll_wait wakeup
get event on socket fd == 6
epoll_wait wakeup
get event on socket fd == 6
epoll_wait wakeup
get event on socket fd == 5
</code></pre><p>再启动几个telnet客户端，可以看到有连接建立情况下，epoll_wait迅速从挂起状态结束；并且套接字上有数据可读时，epoll_wait也迅速结束挂起状态，这时候通过read可以读取套接字接收缓冲区上的数据。</p><pre><code>$telnet 127.0.0.1 43211
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
fasfsafas
snfsfnsnf
^]
telnet&gt; quit
Connection closed.
</code></pre><h2>edge-triggered VS level-triggered</h2><p>对于edge-triggered和level-triggered， 官方的说法是一个是边缘触发，一个是条件触发。也有文章从电子脉冲角度来解读的，总体上，给初学者的带来的感受是理解上有困难。</p><p>这里有两个程序，我们用这个程序来说明一下这两者之间的不同。</p><p>在这两个程序里，即使已连接套接字上有数据可读，我们也不调用read函数去读，只是简单地打印出一句话。</p><p>第一个程序我们设置为edge-triggered，即边缘触发。开启这个服务器程序，用telnet连接上，输入一些字符，我们看到，服务器端只从epoll_wait中苏醒过一次，就是第一次有数据可读的时候。</p><pre><code>$./epoll02
epoll_wait wakeup
epoll_wait wakeup
get event on socket fd == 5
</code></pre><pre><code>$telnet 127.0.0.1 43211
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
asfafas
</code></pre><p>第二个程序我们设置为level-triggered，即条件触发。然后按照同样的步骤来一次，观察服务器端，这一次我们可以看到，服务器端不断地从epoll_wait中苏醒，告诉我们有数据需要读取。</p><pre><code>$./epoll03
epoll_wait wakeup
epoll_wait wakeup
get event on socket fd == 5
epoll_wait wakeup
get event on socket fd == 5
epoll_wait wakeup
get event on socket fd == 5
epoll_wait wakeup
get event on socket fd == 5
...
</code></pre><p>这就是两者的区别，条件触发的意思是只要满足事件的条件，比如有数据需要读，就一直不断地把这个事件传递给用户；而边缘触发的意思是只有第一次满足条件的时候才触发，之后就不会再传递同样的事件了。</p><p>一般我们认为，边缘触发的效率比条件触发的效率要高，这一点也是epoll的杀手锏之一。</p><h2>epoll的历史</h2><p>早在Linux实现epoll之前，Windows系统就已经在1994年引入了IOCP，这是一个异步I/O模型，用来支持高并发的网络I/O，而著名的FreeBSD在2000年引入了Kqueue——一个I/O事件分发框架。</p><p>Linux在2002年引入了epoll，不过相关工作的讨论和设计早在2000年就开始了。如果你感兴趣的话，可以<a href="http:// <a href=" http:="" lkml.iu.edu="" hypermail="" linux="" kernel="" 0010.3="" 0003.html"="">http://lkml.iu.edu/hypermail/linux/kernel/0010.3/0003.html</a>"&gt;点击这里看一下里面的讨论。</p><p>为什么Linux不把FreeBSD的kqueue直接移植过来，而是另辟蹊径创立了epoll呢？</p><p>让我们先看下kqueue的用法，kqueue也需要先创建一个名叫kqueue的对象，然后通过这个对象，调用kevent函数增加感兴趣的事件，同时，也是通过这个kevent函数来等待事件的发生。</p><pre><code>int kqueue(void);
int kevent(int kq, const struct kevent *changelist, int nchanges,
　　　　　 struct kevent *eventlist, int nevents,
　　　　　 const struct timespec *timeout);
void EV_SET(struct kevent *kev, uintptr_t ident, short filter,
　　　　　　u_short flags, u_int fflags, intptr_t data, void *udata);

struct kevent {
　uintptr_t　ident;　　　/* identifier (e.g., file descriptor) */
　short　　　 filter;　　/* filter type (e.g., EVFILT_READ) */
　u_short　　 flags;　　　/* action flags (e.g., EV_ADD) */
　u_int　　　　fflags;　　/* filter-specific flags */
　intptr_t　　 data;　　　/* filter-specific data */
　void　　　　 *udata;　　 /* opaque user data */
};
</code></pre><p>Linus在他最初的设想里，提到了这么一句话，也就是说他觉得类似select或poll的数组方式是可以的，而队列方式则是不可取的。</p><p>So sticky arrays of events are good, while queues are bad. Let’s take that as one of the fundamentals.</p><p>在最初的设计里，Linus等于把keque里面的kevent函数拆分了两个部分，一部分负责事件绑定，通过bind_event函数来实现；另一部分负责事件等待，通过get_events来实现。</p><pre><code>struct event {
     unsigned long id; /* file descriptor ID the event is on */
     unsigned long event; /* bitmask of active events */
};

int bind_event(int fd, struct event *event);
int get_events(struct event * event_array, int maxnr, struct timeval *tmout);
</code></pre><p>和最终的epoll实现相比，前者类似epoll_ctl，后者类似epoll_wait，不过原始的设计里没有考虑到创建epoll句柄，在最终的实现里增加了epoll_create，支持了epoll句柄的创建。</p><p>2002年，epoll最终在Linux 2.5.44中首次出现，在2.6中趋于稳定，为Linux的高性能网络I/O画上了一段句号。</p><h2>总结</h2><p>Linux中epoll的出现，为高性能网络编程补齐了最后一块拼图。epoll通过改进的接口设计，避免了用户态-内核态频繁的数据拷贝，大大提高了系统性能。在使用epoll的时候，我们一定要理解条件触发和边缘触发两种模式。条件触发的意思是只要满足事件的条件，比如有数据需要读，就一直不断地把这个事件传递给用户；而边缘触发的意思是只有第一次满足条件的时候才触发，之后就不会再传递同样的事件了。</p><h2>思考题</h2><p>理解完了epoll，和往常一样，我给你布置两道思考题：</p><p>第一道，你不妨试着修改一下第20讲中select的例子，即在已连接套接字上有数据可读，也不调用read函数去读，看一看你的结果，你认为select是边缘触发的，还是条件触发的？</p><p>第二道，同样的修改一下第21讲poll的例子，看看你的结果，你认为poll是边缘触发的，还是条件触发的？</p><p>你可以在GitHub上上传你的代码，并写出你的疑惑，我会和你一起交流，也欢迎把这篇文章分享给你的朋友或者同事，一起交流一下。</p>
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
  <div class="_2_QraFYR_0">我回想和对了poll和epoll的代码 觉得效率问题主要出现在 epoll返回的是有事件发生的数组,而poll返回的是准备好的个数,每次poll函数返回都要遍历注册的描述符结合数组 尤其是数量越大遍历次数就越多 我觉得性能差异在这里 抛开阻塞和阻塞i&#47;o层面</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这是一个很重要的点，恭喜你悟到了 :)</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-08 17:03:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLLOldQf1VmafqYpeDj2RFdWfo3O4KV5jjHIzaSnmuJf4RjyAtibIkbzuYTX1NGyvznm9asmiaHvyVw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>moob</span>
  </div>
  <div class="_2_QraFYR_0">边缘触发的情况， 如果epoll_wait时提供的events数组太小，那么会错过事件？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果你读了后面答疑部分的源码就会明白，答案是否定的。如果你设置一个很小的events数组，会影响事件的时效性，也就是说，可能10秒前的一个I&#47;O事件，现在你才收到，但是不会错过事件，其原因是事件总在内核中记录着。你不收，内核里也有；你收的快一点，时效性就越好，表现出对用户的体验就越好。<br><br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-15 10:26:17</div>
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
  <div class="_2_QraFYR_0">https:&#47;&#47;github.com&#47;linuxxiaoyu&#47;block<br>执行后都会重复打印”need read socket_fd“和“poll need to read”，所以select和poll应该都是条件触发</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 就喜欢你这样努力完成作业，实际写代码的样子。赞赞赞。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-21 15:33:37</div>
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
  <div class="_2_QraFYR_0">之前对epoll01.c中第66行while(1)很困惑，不明白是从哪里跳出这个循环的。后来通过gdb调试分析，发现在55行把socket_fd设置为了非阻塞，然后在68行调用read时，socket_fd没有数据可读会直接返回-1(errno = EAGAIN),从而在73行通过break跳出了while(1)循环。希望对同样有此困惑的同学有所帮助。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: gdb都用上了，牛~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-21 10:21:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/ea/df/ecd66509.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>空想家</span>
  </div>
  <div class="_2_QraFYR_0">LT + non-blocking 和 ET + non-blocking 有什么区别吗？性能谁更好一点？<br><br>epoll 的惊群问题会讲吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: LT和ET的区别在文稿里已经给出了。<br><br>我理解的惊群问题是当一个网络套接字上有事件发生时，多个线程或者进程会感知，从而引发一群线程干活。<br><br>我认为设计良好的程序应该避免这样的问题，比如一个套接字只被一个线程所管理。我们后面的框架设计也是遵循了这个原则。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-30 09:55:18</div>
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
  <div class="_2_QraFYR_0">期待后面分享 select&#47;poll&#47;epoll三者产生，随着处理的文件描述符越多性能差异越大的原因？<br><br>看文章内容和评论，目前的依据有三个<br>1：避免了用户态-内核态频繁的数据拷贝，这个老师会在后面讲，猜测是零拷贝技术<br>2：边缘条件触发比条件触发性能更优，select&#47;poll都是条件触发，这个老师在评论中又讲有人测试过，其实他俩性能之差，并不如理论上讲的那么大，这需要测试下<br>3：从poll和epoll的返回值角度来思考，epoll返回的是有事件发生的数组，poll返回的是事件就绪的个数，每次poll返回都需要遍历注册的文件描述符的结果数组，尤其数据量越大遍历次数就越多，这个也是poll相对于epoll慢的原因<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-24 15:50:35</div>
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
  <div class="_2_QraFYR_0">经测试seclet与poll都是水平触发</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-28 20:54:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/9b/88/34c171f1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>herongwei</span>
  </div>
  <div class="_2_QraFYR_0">老师，您好，在总结里：“避免了用户态 - 内核态频繁的数据拷贝”这句话是什么意思的呢？文稿中好像没有提到 epoll 的数据拷贝？频繁指的是什么呢？另外，对比之前的 ，select，poll，epoll 三者在数据拷贝之间的区别又有什么不同呢？希望老师回答一下，谢谢！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 在后面的章节里对select、poll、epoll这几个技术进行了不同程度的比较，你可以接着往下读。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-24 09:53:12</div>
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
  <div class="_2_QraFYR_0">为何在水平触发的epoll03，当tcpclient都退出了，还说有数据可读呢，get event on socket fd ==5，二是请问为何每次都从五开始呢，前面0-5分别是谁占用了呢，谢谢。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 因为这部分数据没有被处理掉，而这个正是条件触发的精髓。<br><br>至于描述字，我是这么理解的，0,1,2分别被stdin, stdout和stderr所用，3被监听套接字所用，本地连接的客户端描述字是4，那么本地已连接套接字就是5了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-03 20:57:46</div>
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
  <div class="_2_QraFYR_0">提个小建议，能否把代码解说（例如：第 41-46 行判断了各种错误情况）作为注释放在代码里？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这里主要是因为播报员无法通读代码，所以把代码解说拿出来了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-30 02:42:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ray</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，<br>想跟您请教什么时候会触发EPOLLOUT事件，我们的课程范例好像都只有EPOLLIN事件。<br>当读事件触发后，为什么不用为fd设置EPOLLOUT事件，就可以直接将资料写回fd，这样我们要怎么知道一个fd是否可写呢？<br><br>谢谢老师的解答！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 非常好的问题。<br><br>实际上，确实需要通过注册EPOLLOUT事件，让内核告诉我们可以往某个fd上写数据，课程只是没有展现这部分的能力而已，而在lib&#47;epoll_dispatcher.c中是会注册EPOLLOUT事件的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-27 21:12:09</div>
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
  <div class="_2_QraFYR_0">请问如果我用epoll_ctl显式更改event事件，那么epoll_wait会不会检测到，并装载到events数组中？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 当然会，自动更新，动态检测，这就是NB的地方啊。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-23 17:48:21</div>
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
  <div class="_2_QraFYR_0">错误事件需要我们手动注册吗 还是说系统在遇到错误发生会通过设置events字段来通知我们<br>压根就不需要我们注册？<br>poll也是这样吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 错误事件是不需要注册的，需要在事件检测程序里，针对错误做对应的处理。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-19 01:04:38</div>
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
  <div class="_2_QraFYR_0">老师，边沿触发的时候，如果没有把数据全部读完，剩余的未读数据后续是什么情况。在缓存区中会影响socket后续的读写操作吗，比如这会请求其他资源，触发另一个读时间，会不会读到前面未读的数据？还是说不影响，内核在新的读操作之后会清理掉？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 当然会读到之前的数据，因为TCP是流式数据，像水流一样，连绵不绝，之前的数据不会被扔掉。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-02 18:35:28</div>
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
  <div class="_2_QraFYR_0">思考题第一道：select 是条件触发，修改后的代码：https:&#47;&#47;github.com&#47;YoungYo&#47;yolanda&#47;blob&#47;master&#47;chap-20&#47;select02.c<br>思考题第二道：poll 是条件触发，修改后的代码：https:&#47;&#47;github.com&#47;YoungYo&#47;yolanda&#47;blob&#47;master&#47;chap-21&#47;pollserver03.c</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-23 18:23:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/0b/34/f41d73a4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>王盛武</span>
  </div>
  <div class="_2_QraFYR_0">边缘触发不是很理解，只通知第一次，如果报文没读完，后续的数据怎么办？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你的程序要接着把它读完，否则就会没完没了的收到可读事件。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-06 21:50:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/a6/72/526fd3df.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ICE FROG</span>
  </div>
  <div class="_2_QraFYR_0">边缘触发相同的条件只会触发一次，那来自两个客户端的连接事件会受影响吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不会。触发是绑定到不同的套接字的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-08 19:44:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/61/2d/65f016bf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>常文龙</span>
  </div>
  <div class="_2_QraFYR_0">老师，您好。我看几个例子都只注册了读时间，epoll的例子也只注册了EPOLLIN事件，在处理完EPOLLIN事件后立马做了写操作，这个写操作时，写缓冲区会不会还没准备好？这里可以通过监听EPOLLOUT来做写操作吗？一般什么场景下回去监听写事件呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 好问题，实际上应该处理完读之后，再注册一个EPOLLOUT事件来进行可写判断，当缓冲区真正准备好的时候，在去进行写操作。这在多个连接同时写的时候尤其重要。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-11 16:13:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/a6/a7/b52de33e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>leving</span>
  </div>
  <div class="_2_QraFYR_0">想问下老师，epoll的边缘触发比水平触发的效率要高，请问下，边缘触发效率高在哪里？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 在代码解析那篇里，我详细分析了边缘触发和水平触发实现上的差异，通俗一点说，边缘触发就是通知你一下，解析来应用程序要自己处理，而水平触发则是你没有处理的话，一直不断的通知你处理。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-18 17:28:58</div>
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
  <div class="_2_QraFYR_0">老师给的例子，是不是把close(efd)，给掉了？还是不用关闭</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 应该是忘记关了 :)</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-20 11:16:45</div>
  </div>
</div>
</div>
</li>
</ul>