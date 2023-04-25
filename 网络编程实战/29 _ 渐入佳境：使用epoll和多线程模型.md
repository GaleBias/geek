<audio title="29 _ 渐入佳境：使用epoll和多线程模型" src="https://static001.geekbang.org/resource/audio/10/e7/109a9fa7002421538cb07284da8eeae7.mp3" controls="controls"></audio> 
<p>你好，我是盛延敏，这里是网络编程实战第29讲，欢迎回来。</p><p>在前面的第27讲和第28讲中，我介绍了基于poll事件分发的reactor反应堆模式，以及主从反应堆模式。我们知道，和poll相比，Linux提供的epoll是一种更为高效的事件分发机制。在这一讲里，我们将切换到epoll实现的主从反应堆模式，并且分析一下为什么epoll的性能会强于poll等传统的事件分发机制。</p><h2>如何切换到epoll</h2><p>我已经将所有的代码已经放置到<a href="https://github.com/froghui/yolanda">GitHub</a>上，你可以自行查看或下载。</p><p>我们的网络编程框架是可以同时支持poll和epoll机制的，那么如何开启epoll的支持呢？</p><p>lib/event_loop.c文件的event_loop_init_with_name函数是关键，可以看到，这里是通过宏EPOLL_ENABLE来决定是使用epoll还是poll的。</p><pre><code>struct event_loop *event_loop_init_with_name(char *thread_name) {
  ...
#ifdef EPOLL_ENABLE
    yolanda_msgx(&quot;set epoll as dispatcher, %s&quot;, eventLoop-&gt;thread_name);
    eventLoop-&gt;eventDispatcher = &amp;epoll_dispatcher;
#else
    yolanda_msgx(&quot;set poll as dispatcher, %s&quot;, eventLoop-&gt;thread_name);
    eventLoop-&gt;eventDispatcher = &amp;poll_dispatcher;
#endif
    eventLoop-&gt;event_dispatcher_data = eventLoop-&gt;eventDispatcher-&gt;init(eventLoop);
    ...
}
</code></pre><p>在根目录下的CMakeLists.txt文件里，引入CheckSymbolExists，如果系统里有epoll_create函数和sys/epoll.h，就自动开启EPOLL_ENABLE。如果没有，EPOLL_ENABLE就不会开启，自动使用poll作为默认的事件分发机制。</p><!-- [[[read_end]]] --><pre><code># check epoll and add config.h for the macro compilation
include(CheckSymbolExists)
check_symbol_exists(epoll_create &quot;sys/epoll.h&quot; EPOLL_EXISTS)
if (EPOLL_EXISTS)
    #    Linux下设置为epoll
    set(EPOLL_ENABLE 1 CACHE INTERNAL &quot;enable epoll&quot;)

    #    Linux下也设置为poll
    #    set(EPOLL_ENABLE &quot;&quot; CACHE INTERNAL &quot;not enable epoll&quot;)
else ()
    set(EPOLL_ENABLE &quot;&quot; CACHE INTERNAL &quot;not enable epoll&quot;)
endif ()
</code></pre><p>但是，为了能让编译器使用到这个宏，需要让CMake往config.h文件里写入这个宏的最终值，configure_file命令就是起这个作用的。其中config.h.cmake是一个模板文件，已经预先创建在根目录下。同时还需要让编译器include这个config.h文件。include_directories可以帮我们达成这个目标。</p><pre><code>configure_file(${CMAKE_CURRENT_SOURCE_DIR}/config.h.cmake
        ${CMAKE_CURRENT_BINARY_DIR}/include/config.h)

include_directories(${CMAKE_CURRENT_BINARY_DIR}/include)
</code></pre><p>这样，在Linux下，就会默认使用epoll作为事件分发。</p><p>那么前面的<a href="https://time.geekbang.org/column/article/146664">27讲</a>和<a href="https://time.geekbang.org/column/article/148148">28讲</a>中的程序案例如何改为使用poll的呢？</p><p>我们可以修改CMakeLists.txt文件，把Linux下设置为poll的那段注释下的命令打开，同时关闭掉原先设置为1的命令就可以了。 下面就是具体的示例代码。</p><pre><code># check epoll and add config.h for the macro compilation
include(CheckSymbolExists)
check_symbol_exists(epoll_create &quot;sys/epoll.h&quot; EPOLL_EXISTS)
if (EPOLL_EXISTS)
    #    Linux下也设置为poll
     set(EPOLL_ENABLE &quot;&quot; CACHE INTERNAL &quot;not enable epoll&quot;)
else ()
    set(EPOLL_ENABLE &quot;&quot; CACHE INTERNAL &quot;not enable epoll&quot;)
endif (
</code></pre><p>不管怎样，现在我们得到了一个Linux下使用epoll作为事件分发的版本，现在让我们使用它来编写程序吧。</p><h2>样例程序</h2><p>我们的样例程序和<a href="https://time.geekbang.org/column/article/148148">第28讲</a>的一模一样，只是现在我们的事件分发机制从poll切换到了epoll。</p><pre><code>#include &lt;lib/acceptor.h&gt;
#include &quot;lib/common.h&quot;
#include &quot;lib/event_loop.h&quot;
#include &quot;lib/tcp_server.h&quot;

char rot13_char(char c) {
    if ((c &gt;= 'a' &amp;&amp; c &lt;= 'm') || (c &gt;= 'A' &amp;&amp; c &lt;= 'M'))
        return c + 13;
    else if ((c &gt;= 'n' &amp;&amp; c &lt;= 'z') || (c &gt;= 'N' &amp;&amp; c &lt;= 'Z'))
        return c - 13;
    else
        return c;
}

//连接建立之后的callback
int onConnectionCompleted(struct tcp_connection *tcpConnection) {
    printf(&quot;connection completed\n&quot;);
    return 0;
}

//数据读到buffer之后的callback
int onMessage(struct buffer *input, struct tcp_connection *tcpConnection) {
    printf(&quot;get message from tcp connection %s\n&quot;, tcpConnection-&gt;name);
    printf(&quot;%s&quot;, input-&gt;data);

    struct buffer *output = buffer_new();
    int size = buffer_readable_size(input);
    for (int i = 0; i &lt; size; i++) {
        buffer_append_char(output, rot13_char(buffer_read_char(input)));
    }
    tcp_connection_send_buffer(tcpConnection, output);
    return 0;
}

//数据通过buffer写完之后的callback
int onWriteCompleted(struct tcp_connection *tcpConnection) {
    printf(&quot;write completed\n&quot;);
    return 0;
}

//连接关闭之后的callback
int onConnectionClosed(struct tcp_connection *tcpConnection) {
    printf(&quot;connection closed\n&quot;);
    return 0;
}

int main(int c, char **v) {
    //主线程event_loop
    struct event_loop *eventLoop = event_loop_init();

    //初始化acceptor
    struct acceptor *acceptor = acceptor_init(SERV_PORT);

    //初始tcp_server，可以指定线程数目，这里线程是4，说明是一个acceptor线程，4个I/O线程，没一个I/O线程
    //tcp_server自己带一个event_loop
    struct TCPserver *tcpServer = tcp_server_init(eventLoop, acceptor, onConnectionCompleted, onMessage,
                                                  onWriteCompleted, onConnectionClosed, 4);
    tcp_server_start(tcpServer);

    // main thread for acceptor
    event_loop_run(eventLoop);
}
</code></pre><p>关于这个程序，之前一直没有讲到的部分是缓冲区对象buffer。这其实也是网络编程框架应该考虑的部分。</p><p>我们希望框架可以对应用程序封装掉套接字读和写的部分，转而提供的是针对缓冲区对象的读和写操作。这样一来，从套接字收取数据、处理异常、发送数据等操作都被类似buffer这样的对象所封装和屏蔽，应用程序所要做的事情就会变得更加简单，从buffer对象中可以获取已接收到的字节流再进行应用层处理，比如这里通过调用buffer_read_char函数从buffer中读取一个字节。</p><p>另外一方面，框架也必须对应用程序提供套接字发送的接口，接口的数据类型类似这里的buffer对象，可以看到，这里先生成了一个buffer对象，之后将编码后的结果填充到buffer对象里，最后调用tcp_connection_send_buffer将buffer对象里的数据通过套接字发送出去。</p><p>这里像onMessage、onConnectionClosed几个回调函数都是运行在子反应堆线程中的，也就是说，刚刚提到的生成buffer对象，encode部分的代码，是在子反应堆线程中执行的。这其实也是回调函数的内涵，回调函数本身只是提供了类似Handlder的处理逻辑，具体执行是由事件分发线程，或者说是event loop线程发起的。</p><p>框架通过一层抽象，让应用程序的开发者只需要看到回调函数，回调函数中的对象，也都是如buffer和tcp_connection这样封装过的对象，这样像套接字、字节流等底层实现的细节就完全由框架来完成了。</p><p>框架帮我们做了很多事情，那这些事情是如何做到的？在第四篇实战篇，我们将一一揭开答案。如果你有兴趣，不妨先看看实现代码。</p><h2>样例程序结果</h2><p>启动服务器，可以从屏幕输出上看到，使用的是epoll作为事件分发器。</p><pre><code>$./epoll-server-multithreads
[msg] set epoll as dispatcher, main thread
[msg] add channel fd == 5, main thread
[msg] set epoll as dispatcher, Thread-1
[msg] add channel fd == 9, Thread-1
[msg] event loop thread init and signal, Thread-1
[msg] event loop run, Thread-1
[msg] event loop thread started, Thread-1
[msg] set epoll as dispatcher, Thread-2
[msg] add channel fd == 12, Thread-2
[msg] event loop thread init and signal, Thread-2
[msg] event loop run, Thread-2
[msg] event loop thread started, Thread-2
[msg] set epoll as dispatcher, Thread-3
[msg] add channel fd == 15, Thread-3
[msg] event loop thread init and signal, Thread-3
[msg] event loop run, Thread-3
[msg] event loop thread started, Thread-3
[msg] set epoll as dispatcher, Thread-4
[msg] add channel fd == 18, Thread-4
[msg] event loop thread init and signal, Thread-4
[msg] event loop run, Thread-4
[msg] event loop thread started, Thread-4
[msg] add channel fd == 6, main thread
[msg] event loop run, main thread
</code></pre><p>开启多个telnet客户端，连接上该服务器, 通过屏幕输入和服务器端交互。</p><pre><code>$telnet 127.0.0.1 43211
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
fafaf
snsns
^]


telnet&gt; quit
Connection closed.
</code></pre><p>服务端显示不断地从epoll_wait中返回处理I/O事件。</p><pre><code>[msg] epoll_wait wakeup, main thread
[msg] get message channel fd==6 for read, main thread
[msg] activate channel fd == 6, revents=2, main thread
[msg] new connection established, socket == 19
connection completed
[msg] epoll_wait wakeup, Thread-1
[msg] get message channel fd==9 for read, Thread-1
[msg] activate channel fd == 9, revents=2, Thread-1
[msg] wakeup, Thread-1
[msg] add channel fd == 19, Thread-1
[msg] epoll_wait wakeup, Thread-1
[msg] get message channel fd==19 for read, Thread-1
[msg] activate channel fd == 19, revents=2, Thread-1
get message from tcp connection connection-19
afasf
[msg] epoll_wait wakeup, main thread
[msg] get message channel fd==6 for read, main thread
[msg] activate channel fd == 6, revents=2, main thread
[msg] new connection established, socket == 20
connection completed
[msg] epoll_wait wakeup, Thread-2
[msg] get message channel fd==12 for read, Thread-2
[msg] activate channel fd == 12, revents=2, Thread-2
[msg] wakeup, Thread-2
[msg] add channel fd == 20, Thread-2
[msg] epoll_wait wakeup, Thread-2
[msg] get message channel fd==20 for read, Thread-2
[msg] activate channel fd == 20, revents=2, Thread-2
get message from tcp connection connection-20
asfasfas
[msg] epoll_wait wakeup, Thread-2
[msg] get message channel fd==20 for read, Thread-2
[msg] activate channel fd == 20, revents=2, Thread-2
connection closed
[msg] epoll_wait wakeup, main thread
[msg] get message channel fd==6 for read, main thread
[msg] activate channel fd == 6, revents=2, main thread
[msg] new connection established, socket == 21
connection completed
[msg] epoll_wait wakeup, Thread-3
[msg] get message channel fd==15 for read, Thread-3
[msg] activate channel fd == 15, revents=2, Thread-3
[msg] wakeup, Thread-3
[msg] add channel fd == 21, Thread-3
[msg] epoll_wait wakeup, Thread-3
[msg] get message channel fd==21 for read, Thread-3
[msg] activate channel fd == 21, revents=2, Thread-3
get message from tcp connection connection-21
dfasfadsf
[msg] epoll_wait wakeup, Thread-1
[msg] get message channel fd==19 for read, Thread-1
[msg] activate channel fd == 19, revents=2, Thread-1
connection closed
[msg] epoll_wait wakeup, main thread
[msg] get message channel fd==6 for read, main thread
[msg] activate channel fd == 6, revents=2, main thread
[msg] new connection established, socket == 22
connection completed
[msg] epoll_wait wakeup, Thread-4
[msg] get message channel fd==18 for read, Thread-4
[msg] activate channel fd == 18, revents=2, Thread-4
[msg] wakeup, Thread-4
[msg] add channel fd == 22, Thread-4
[msg] epoll_wait wakeup, Thread-4
[msg] get message channel fd==22 for read, Thread-4
[msg] activate channel fd == 22, revents=2, Thread-4
get message from tcp connection connection-22
fafaf
[msg] epoll_wait wakeup, Thread-4
[msg] get message channel fd==22 for read, Thread-4
[msg] activate channel fd == 22, revents=2, Thread-4
connection closed
[msg] epoll_wait wakeup, Thread-3
[msg] get message channel fd==21 for read, Thread-3
[msg] activate channel fd == 21, revents=2, Thread-3
connection closed
</code></pre><p>其中主线程的epoll_wait只处理acceptor套接字的事件，表示的是连接的建立；反应堆子线程的epoll_wait主要处理的是已连接套接字的读写事件。这幅图详细解释了这部分逻辑。</p><p><img src="https://static001.geekbang.org/resource/image/16/dd/167e8e055d690a15f22cee8f114fb5dd.png?wh=1014*1128" alt=""></p><h2>epoll的性能分析</h2><p>epoll的性能凭什么就要比poll或者select好呢？这要从两个角度来说明。</p><p>第一个角度是事件集合。在每次使用poll或select之前，都需要准备一个感兴趣的事件集合，系统内核拿到事件集合，进行分析并在内核空间构建相应的数据结构来完成对事件集合的注册。而epoll则不是这样，epoll维护了一个全局的事件集合，通过epoll句柄，可以操纵这个事件集合，增加、删除或修改这个事件集合里的某个元素。要知道在绝大多数情况下，事件集合的变化没有那么的大，这样操纵系统内核就不需要每次重新扫描事件集合，构建内核空间数据结构。</p><p>第二个角度是就绪列表。每次在使用poll或者select之后，应用程序都需要扫描整个感兴趣的事件集合，从中找出真正活动的事件，这个列表如果增长到10K以上，每次扫描的时间损耗也是惊人的。事实上，很多情况下扫描完一圈，可能发现只有几个真正活动的事件。而epoll则不是这样，epoll返回的直接就是活动的事件列表，应用程序减少了大量的扫描时间。</p><p>此外， epoll还提供了更高级的能力——边缘触发。<a href="https://time.geekbang.org/column/article/143245">第23讲</a>通过一个直观的例子，讲解了边缘触发和条件触发的区别。</p><p>这里再举一个例子说明一下。</p><p>如果某个套接字有100个字节可以读，边缘触发（edge-triggered）和条件触发（level-triggered）都会产生read ready notification事件，如果应用程序只读取了50个字节，边缘触发就会陷入等待；而条件触发则会因为还有50个字节没有读取完，不断地产生read ready notification事件。</p><p>在条件触发下（level-triggered），如果某个套接字缓冲区可以写，会无限次返回write ready notification事件，在这种情况下，如果应用程序没有准备好，不需要发送数据，一定需要解除套接字上的ready notification事件，否则CPU就直接跪了。</p><p>我们简单地总结一下，边缘触发只会产生一次活动事件，性能和效率更高。不过，程序处理起来要更为小心。</p><h2>总结</h2><p>本讲我们将程序框架切换到了epoll的版本，和poll版本相比，只是底层的框架做了更改，上层应用程序不用做任何修改，这也是程序框架强大的地方。和poll相比，epoll从事件集合和就绪列表两个方面加强了程序性能，是Linux下高性能网络程序的首选。</p><h2>思考题</h2><p>最后我给你布置两道思考题：</p><p>第一道，说说你对边缘触发和条件触发的理解。</p><p>第二道，对于边缘触发和条件触发，onMessage函数处理要注意什么？</p><p>欢迎你在评论区写下你的思考，也欢迎把这篇文章分享给你的朋友或者同事，一起交流进步。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/f6/e3/e4bcd69e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>沉淀的梦想</span>
  </div>
  <div class="_2_QraFYR_0">在ET的情况下，write ready notification只会在套接字可写的时候通知一次的话，那个时候应用还没准备好数据，等到应用准备好数据时，却又没有通知了，会不会导致数据滞留发不出去？这种情况是怎么解决的呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你可以再次注册这个write ready的事件啊，不是说只能注册一次就结束了，而是你注册了一次，它就通知你一次；而LT的情况下，可能你注册了一次，它通知你好多次。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-15 14:01:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/3d/f8/b13674e6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>LiYanbin</span>
  </div>
  <div class="_2_QraFYR_0">源代码看起来有点花了点时间，将这部分的代码从抽离了出来，便于大家跟踪代码理解，同时写了简单的makefile。代码地址：https:&#47;&#47;github.com&#47;kevinrsa&#47;epoll_server_multithreads 。如有不妥，联系删除</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: makefile写得不错：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-29 12:28:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/16/97/9e342700.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span> JJ</span>
  </div>
  <div class="_2_QraFYR_0">边缘条件，当套接字缓冲区可写，会不断触发ready notification事件，不是应该条件触发才是这样吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 笔误，已经让编辑勘误了，感谢指正。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-14 08:32:35</div>
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
  <div class="_2_QraFYR_0">如果应用程序只读取了 50 个字节，边缘触发就会陷入等待；<br>这里的陷入等待是什么意思呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不会再继续发送read_notification事件，必须等所有的100个字节被读完，才会发送下一个read_notification事件。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-22 18:56:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/44/04/7904829d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张三说</span>
  </div>
  <div class="_2_QraFYR_0">老师，一直没搞懂ET和LT的性能区别，仅仅因为LT会多提醒一些次数就与ET相差明显的性能吗？一直很纠结这个问题</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 有没有跑例子程序呢？其实不用纠结，最新的测试表明，两者差别其实没有那么大。但是非要比一个差距的话，ET还是效率好一些，但是对应用程序开发者的要求高一些。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-13 15:49:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/dc/19/c058bcbf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>流浪地球</span>
  </div>
  <div class="_2_QraFYR_0">细读了下老师git上的代码，套接字都是设置为非阻塞模式的，但并没有对返回值做判断处理，看上去好像是阻塞式的用法，求解？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可能是考虑不周，有可能的话麻烦提一个MR或者issue，大家一起来改。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-17 16:39:42</div>
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
  <div class="_2_QraFYR_0">27章以及以后源代码的难度提升了一个等级了。看了相当吃力呀。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 多读两遍会好很多</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-08 11:08:35</div>
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
  <div class="_2_QraFYR_0">老师，这个就绪列表是建立在事件集合之上的对吧。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，是所有感兴趣的事件集合。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-16 19:56:01</div>
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
  <div class="_2_QraFYR_0">老师好，<br>针对第2题，目前想到onMessage函数应该要注意，如果当前程序无法处理该通知，应该要想办法再次注册该事件。<br><br>只是具体程序实现就不知道应该怎么写了，可能还要请老师说明一下 哈哈XD<br><br>谢谢老师^^</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 当前的实现并不会主动把I&#47;O读写事件从事件通道上摘除哦，所以并不需要重新注册该事件，onMessage就是一个简单的报文解析函数，所要做到的就是在条件触发情况下读完所有的字节，避免不断的再次被事件驱动。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-12 19:53:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/6d/46/e16291f8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>丁小明</span>
  </div>
  <div class="_2_QraFYR_0">为什么 socket已经有缓冲区了，应用层还要缓冲区呢，比如发送，socket也会合并发送</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 很简单，应用层需要对接收到的byte字节流进行编解码，为了方便，在应用层进行缓冲，之后进行编解码的操作，再送给业务逻辑层来处理。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-10 23:15:46</div>
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
  <div class="_2_QraFYR_0">看到CMake我就完全懵逼。。。。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 还好吧，看一下CMake的文档，以前我一直用的Makefile, CMake也是现学的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-16 16:15:42</div>
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
  <div class="_2_QraFYR_0">老师能不能为这个框架写一份README.md，我对这个实现很感兴趣</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你需要什么样的README.md呢？第四篇会详细讲解这个框架的设计，也行你读完之后，可以写一个README.md push到git上呢？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-14 15:50:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/22/28/e8/7734b8d3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>P</span>
  </div>
  <div class="_2_QraFYR_0">只提一点，所有关于Reactor的图片都不太准确。流程应该是client-&gt;Acceptor-&gt;Poller(select&#47;poll&#47;epoll)，然而文章中所有的Acceptor都放在了后面，令人疑惑。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-12 23:00:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/a9/ce/23f2e185.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Running man</span>
  </div>
  <div class="_2_QraFYR_0">event_loop.c编译链接不上pthread库，有哪位朋友知道如何修改cmakelist，gcc版本是11.2.0 ubuntu系统版本是11.2.0，对应内核版本5.15.0-41</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-30 00:05:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/b9/7c/afe6f1eb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>vv_test</span>
  </div>
  <div class="_2_QraFYR_0">性能对比第一点，是否可以这样理解。select、poll在用户态声明的事件拷贝(我在这里理解拷贝，不是注册，因为下一次调用依旧要传入)到内核态，大量操作copy的情况下耗时不容小觑。而epoll是已经注册到对应的epoll实例。主要是省去了这个copy的时间</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，也是有这方面的考虑，不过更多的还是事件处理的机制和效率的问题。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-28 23:03:10</div>
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
  <div class="_2_QraFYR_0">有个疑问，这个程序与下一章的HTTP服务器的设计，处理连接的时候，服务器什么时候会关闭对端的连接？<br>是不断与客户端交互，客户端发送关闭请求才关闭；还是处理完客户端的请求后，发送响应，再关闭</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 第一种。代码如下:<br><br>int handle_read(void *data) {<br>    struct tcp_connection *tcpConnection = (struct tcp_connection *) data;<br>    struct buffer *input_buffer = tcpConnection-&gt;input_buffer;<br>    struct channel *channel = tcpConnection-&gt;channel;<br><br>    if (buffer_socket_read(input_buffer, channel-&gt;fd) &gt; 0) {<br>        &#47;&#47;应用程序真正读取Buffer里的数据<br>        if (tcpConnection-&gt;messageCallBack != NULL) {<br>            tcpConnection-&gt;messageCallBack(input_buffer, tcpConnection);<br>        }<br>    } else {<br>        handle_connection_closed(tcpConnection);<br>    }<br>}</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-18 20:29:06</div>
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
  <div class="_2_QraFYR_0">老师，所以不删除写事件，就不需要重新注册是吗？每次缓冲区由满变成可写都会通知一次，是这样理解吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，这样的写效率会变低。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-10 14:28:45</div>
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
  <div class="_2_QraFYR_0">第一个角度是事件集合。在每次使用 poll 或 select 之前，都需要准备一个感兴趣的事件集合，系统内核拿到事件集合，进行分析并在内核空间构建相应的数据结构来完成对事件集合的注册。而 epoll 则不是这样，epoll 维护了一个全局的事件集合，通过 epoll 句柄，可以操纵这个事件集合，增加、删除或修改这个事件集合里的某个元素。要知道在绝大多数情况下，事件集合的变化没有那么的大，这样操纵系统内核就不需要每次重新扫描事件集合，构建内核空间数据结构。<br>  老师，这个不是很理解，看了下，前面的epoll实例代码，epoll_wait时，还是需要传入一个events（看起来是初始化了下）的，这个是做什么用的，我理解，epoll对象本身不是已经有它所关联的事件信息了吗（通过epoll_ctrl add进去）</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: epoll_wait返回给用户空间需要处理的 I&#47;O 事件，用这个events来表示，这样我们才知道具体发生了什么事件。具体的例子可以参考第23章。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-17 16:25:24</div>
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
  <div class="_2_QraFYR_0">老师，请问下，我看用poll实现里的结构配图，可以用threadpool来解耦具体业务逻辑，epoll里的配图，没有这个，其实也是可以加的吧，本质上线程池解耦业务这部分应该是通用吧，只是在事件触发， 事件分发机制上的差别吧？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，完全正确。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-17 16:21:58</div>
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
  <div class="_2_QraFYR_0">文稿中的框架示意图，我看到main reactor 和 sub reactor都各自运行了epoll,请问是否各自处理不同的socket？ 如果处理了相同的socket会发生什么吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: main reactor处理的是监听套接字上的事件，sub reactor处理的是已连接套接字上的事件，两个是不重合的。<br><br>如果处理了相同的socket，那么肯定需要通过锁-并发来控制，无形中就增加了处理的开销，降低了程序处理的效率。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-02 09:00:42</div>
  </div>
</div>
</div>
</li>
</ul>