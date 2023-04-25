<audio title="33 _ 自己动手写高性能HTTP服务器（二）：IO模型和多线程模型实现" src="https://static001.geekbang.org/resource/audio/16/dc/16941853d20400550cddc171652fcfdc.mp3" controls="controls"></audio> 
<p>你好，我是盛延敏，这里是网络编程实战第33讲，欢迎回来。</p><p>这一讲，我们延续第32讲的话题，继续解析高性能网络编程框架的I/O模型和多线程模型设计部分。</p><h2>多线程设计的几个考虑</h2><p>在我们的设计中，main reactor线程是一个acceptor线程，这个线程一旦创建，会以event_loop形式阻塞在event_dispatcher的dispatch方法上，实际上，它在等待监听套接字上的事件发生，也就是已完成的连接，一旦有连接完成，就会创建出连接对象tcp_connection，以及channel对象等。</p><p>当用户期望使用多个sub-reactor子线程时，主线程会创建多个子线程，每个子线程在创建之后，按照主线程指定的启动函数立即运行，并进行初始化。随之而来的问题是，<strong>主线程如何判断子线程已经完成初始化并启动，继续执行下去呢？这是一个需要解决的重点问题。</strong></p><p>在设置了多个线程的情况下，需要将新创建的已连接套接字对应的读写事件交给一个sub-reactor线程处理。所以，这里从thread_pool中取出一个线程，<strong>通知这个线程有新的事件加入。而这个线程很可能是处于事件分发的阻塞调用之中，如何协调主线程数据写入给子线程，这是另一个需要解决的重点问题。</strong></p><!-- [[[read_end]]] --><p>子线程是一个event_loop线程，它阻塞在dispatch上，一旦有事件发生，它就会查找channel_map，找到对应的处理函数并执行它。之后它就会增加、删除或修改pending事件，再次进入下一轮的dispatch。</p><p>这张图阐述了线程的运行关系。</p><p><img src="https://static001.geekbang.org/resource/image/55/14/55bb7ef8659395e39395b109dbd28f14.png?wh=1122*968" alt=""><br>
为了方便你理解，我把对应的函数实现列在了另外一张图中。</p><p><img src="https://static001.geekbang.org/resource/image/da/ca/dac29d3a8fc4f26a09af9e18fc16b2ca.jpg?wh=3500*3002" alt=""></p><h2>主线程等待多个sub-reactor子线程初始化完</h2><p>主线程需要等待子线程完成初始化，也就是需要获取子线程对应数据的反馈，而子线程初始化也是对这部分数据进行初始化，实际上这是一个多线程的通知问题。采用的做法在<a href="https://time.geekbang.org/column/article/145464">前面</a>讲多线程的时候也提到过，使用mutex和condition两个主要武器。</p><p>下面这段代码是主线程发起的子线程创建，调用event_loop_thread_init对每个子线程初始化，之后调用event_loop_thread_start来启动子线程。注意，如果应用程序指定的线程池大小为0，则直接返回，这样acceptor和I/O事件都会在同一个主线程里处理，就退化为单reactor模式。</p><pre><code>//一定是main thread发起
void thread_pool_start(struct thread_pool *threadPool) {
    assert(!threadPool-&gt;started);
    assertInSameThread(threadPool-&gt;mainLoop);

    threadPool-&gt;started = 1;
    void *tmp;

    if (threadPool-&gt;thread_number &lt;= 0) {
        return;
    }

    threadPool-&gt;eventLoopThreads = malloc(threadPool-&gt;thread_number * sizeof(struct event_loop_thread));
    for (int i = 0; i &lt; threadPool-&gt;thread_number; ++i) {
        event_loop_thread_init(&amp;threadPool-&gt;eventLoopThreads[i], i);
        event_loop_thread_start(&amp;threadPool-&gt;eventLoopThreads[i]);
    }
}
</code></pre><p>我们再看一下event_loop_thread_start这个方法，这个方法一定是主线程运行的。这里我使用了pthread_create创建了子线程，子线程一旦创建，立即执行event_loop_thread_run，我们稍后将看到，event_loop_thread_run进行了子线程的初始化工作。这个函数最重要的部分是使用了pthread_mutex_lock和pthread_mutex_unlock进行了加锁和解锁，并使用了pthread_cond_wait来守候eventLoopThread中的eventLoop的变量。</p><pre><code>//由主线程调用，初始化一个子线程，并且让子线程开始运行event_loop
struct event_loop *event_loop_thread_start(struct event_loop_thread *eventLoopThread) {
    pthread_create(&amp;eventLoopThread-&gt;thread_tid, NULL, &amp;event_loop_thread_run, eventLoopThread);

    assert(pthread_mutex_lock(&amp;eventLoopThread-&gt;mutex) == 0);

    while (eventLoopThread-&gt;eventLoop == NULL) {
        assert(pthread_cond_wait(&amp;eventLoopThread-&gt;cond, &amp;eventLoopThread-&gt;mutex) == 0);
    }
    assert(pthread_mutex_unlock(&amp;eventLoopThread-&gt;mutex) == 0);

    yolanda_msgx(&quot;event loop thread started, %s&quot;, eventLoopThread-&gt;thread_name);
    return eventLoopThread-&gt;eventLoop;
}
</code></pre><p>为什么要这么做呢？看一下子线程的代码你就会大致明白。子线程执行函数event_loop_thread_run一上来也是进行了加锁，之后初始化event_loop对象，当初始化完成之后，调用了pthread_cond_signal函数来通知此时阻塞在pthread_cond_wait上的主线程。这样，主线程就会从wait中苏醒，代码得以往下执行。子线程本身也通过调用event_loop_run进入了一个无限循环的事件分发执行体中，等待子线程reator上注册过的事件发生。</p><pre><code>void *event_loop_thread_run(void *arg) {
    struct event_loop_thread *eventLoopThread = (struct event_loop_thread *) arg;

    pthread_mutex_lock(&amp;eventLoopThread-&gt;mutex);

    // 初始化化event loop，之后通知主线程
    eventLoopThread-&gt;eventLoop = event_loop_init();
    yolanda_msgx(&quot;event loop thread init and signal, %s&quot;, eventLoopThread-&gt;thread_name);
    pthread_cond_signal(&amp;eventLoopThread-&gt;cond);

    pthread_mutex_unlock(&amp;eventLoopThread-&gt;mutex);

    //子线程event loop run
    eventLoopThread-&gt;eventLoop-&gt;thread_name = eventLoopThread-&gt;thread_name;
    event_loop_run(eventLoopThread-&gt;eventLoop);
}
</code></pre><p>可以看到，这里主线程和子线程共享的变量正是每个event_loop_thread的eventLoop对象，这个对象在初始化的时候为NULL，只有当子线程完成了初始化，才变成一个非NULL的值，这个变化是子线程完成初始化的标志，也是信号量守护的变量。通过使用锁和信号量，解决了主线程和子线程同步的问题。当子线程完成初始化之后，主线程才会继续往下执行。</p><pre><code>struct event_loop_thread {
    struct event_loop *eventLoop;
    pthread_t thread_tid;        /* thread ID */
    pthread_mutex_t mutex;
    pthread_cond_t cond;
    char * thread_name;
    long thread_count;    /* # connections handled */
};
</code></pre><p>你可能会问，主线程是循环在等待每个子线程完成初始化，如果进入第二个循环，等待第二个子线程完成初始化，而此时第二个子线程已经初始化完成了，该怎么办？</p><p>注意我们这里一上来是加锁的，只要取得了这把锁，同时发现event_loop_thread的eventLoop对象已经变成非NULL值，可以肯定第二个线程已经初始化，就直接释放锁往下执行了。</p><p>你可能还会问，在执行pthread_cond_wait的时候，需要持有那把锁么？这里，父线程在调用pthread_cond_wait函数之后，会立即进入睡眠，并释放持有的那把互斥锁。而当父线程再从pthread_cond_wait返回时（这是子线程通过pthread_cond_signal通知达成的），该线程再次持有那把锁。</p><h2>增加已连接套接字事件到sub-reactor线程中</h2><p>前面提到，主线程是一个main reactor线程，这个线程负责检测监听套接字上的事件，当有事件发生时，也就是一个连接已完成建立，如果我们有多个sub-reactor子线程，我们期望的结果是，把这个已连接套接字相关的I/O事件交给sub-reactor子线程负责检测。这样的好处是，main reactor只负责连接套接字的建立，可以一直维持在一个非常高的处理效率，在多核的情况下，多个sub-reactor可以很好地利用上多核处理的优势。</p><p>不过，这里有一个令人苦恼的问题。</p><p>我们知道，sub-reactor线程是一个无限循环的event loop执行体，在没有已注册事件发生的情况下，这个线程阻塞在event_dispatcher的dispatch上。你可以简单地认为阻塞在poll调用或者epoll_wait上，这种情况下，主线程如何能把已连接套接字交给sub-reactor子线程呢？</p><p>当然有办法。</p><p>如果我们能让sub-reactor线程从event_dispatcher的dispatch上返回，再让sub-reactor线程返回之后能够把新的已连接套接字事件注册上，这件事情就算完成了。</p><p>那如何让sub-reactor线程从event_dispatcher的dispatch上返回呢？答案是构建一个类似管道一样的描述字，让event_dispatcher注册该管道描述字，当我们想让sub-reactor线程苏醒时，往管道上发送一个字符就可以了。</p><p>在event_loop_init函数里，调用了socketpair函数创建了套接字对，这个套接字对的作用就是我刚刚说过的，往这个套接字的一端写时，另外一端就可以感知到读的事件。其实，这里也可以直接使用UNIX上的pipe管道，作用是一样的。</p><pre><code>struct event_loop *event_loop_init() {
    ...
    //add the socketfd to event 这里创建的是套接字对，目的是为了唤醒子线程
    eventLoop-&gt;owner_thread_id = pthread_self();
    if (socketpair(AF_UNIX, SOCK_STREAM, 0, eventLoop-&gt;socketPair) &lt; 0) {
        LOG_ERR(&quot;socketpair set fialed&quot;);
    }
    eventLoop-&gt;is_handle_pending = 0;
    eventLoop-&gt;pending_head = NULL;
    eventLoop-&gt;pending_tail = NULL;
    eventLoop-&gt;thread_name = &quot;main thread&quot;;

    struct channel *channel = channel_new(eventLoop-&gt;socketPair[1], EVENT_READ, handleWakeup, NULL, eventLoop);
    event_loop_add_channel_event(eventLoop, eventLoop-&gt;socketPair[1], channel);

    return eventLoop;
}
</code></pre><p>要特别注意的是这句代码，这告诉event_loop的，是注册了socketPair[1]描述字上的READ事件，如果有READ事件发生，就调用handleWakeup函数来完成事件处理。</p><pre><code>struct channel *channel = channel_new(eventLoop-&gt;socketPair[1], EVENT_READ, handleWakeup, NULL, eventLoop);
</code></pre><p>我们来看看这个handleWakeup函数：</p><p>事实上，这个函数就是简单的从socketPair[1]描述字上读取了一个字符而已，除此之外，它什么也没干。它的主要作用就是让子线程从dispatch的阻塞中苏醒。</p><pre><code>int handleWakeup(void * data) {
    struct event_loop *eventLoop = (struct event_loop *) data;
    char one;
    ssize_t n = read(eventLoop-&gt;socketPair[1], &amp;one, sizeof one);
    if (n != sizeof one) {
        LOG_ERR(&quot;handleWakeup  failed&quot;);
    }
    yolanda_msgx(&quot;wakeup, %s&quot;, eventLoop-&gt;thread_name);
}
</code></pre><p>现在，我们再回过头看看，如果有新的连接产生，主线程是怎么操作的？在handle_connection_established中，通过accept调用获取了已连接套接字，将其设置为非阻塞套接字（切记），接下来调用thread_pool_get_loop获取一个event_loop。thread_pool_get_loop的逻辑非常简单，从thread_pool线程池中按照顺序挑选出一个线程来服务。接下来是创建了tcp_connection对象。</p><pre><code>//处理连接已建立的回调函数
int handle_connection_established(void *data) {
    struct TCPserver *tcpServer = (struct TCPserver *) data;
    struct acceptor *acceptor = tcpServer-&gt;acceptor;
    int listenfd = acceptor-&gt;listen_fd;

    struct sockaddr_in client_addr;
    socklen_t client_len = sizeof(client_addr);
    //获取这个已建立的套集字，设置为非阻塞套集字
    int connected_fd = accept(listenfd, (struct sockaddr *) &amp;client_addr, &amp;client_len);
    make_nonblocking(connected_fd);

    yolanda_msgx(&quot;new connection established, socket == %d&quot;, connected_fd);

    //从线程池里选择一个eventloop来服务这个新的连接套接字
    struct event_loop *eventLoop = thread_pool_get_loop(tcpServer-&gt;threadPool);

    // 为这个新建立套接字创建一个tcp_connection对象，并把应用程序的callback函数设置给这个tcp_connection对象
    struct tcp_connection *tcpConnection = tcp_connection_new(connected_fd, eventLoop,tcpServer-&gt;connectionCompletedCallBack,tcpServer-&gt;connectionClosedCallBack,tcpServer-&gt;messageCallBack,tcpServer-&gt;writeCompletedCallBack);
    //callback内部使用
    if (tcpServer-&gt;data != NULL) {
        tcpConnection-&gt;data = tcpServer-&gt;data;
    }
    return 0;
}
</code></pre><p>在调用tcp_connection_new创建tcp_connection对象的代码里，可以看到先是创建了一个channel对象，并注册了READ事件，之后调用event_loop_add_channel_event方法往子线程中增加channel对象。</p><pre><code>tcp_connection_new(int connected_fd, struct event_loop *eventLoop,
                   connection_completed_call_back connectionCompletedCallBack,
                   connection_closed_call_back connectionClosedCallBack,
                   message_call_back messageCallBack, write_completed_call_back writeCompletedCallBack) {
    ...
    //为新的连接对象创建可读事件
    struct channel *channel1 = channel_new(connected_fd, EVENT_READ, handle_read, handle_write, tcpConnection);
    tcpConnection-&gt;channel = channel1;

    //完成对connectionCompleted的函数回调
    if (tcpConnection-&gt;connectionCompletedCallBack != NULL) {
        tcpConnection-&gt;connectionCompletedCallBack(tcpConnection);
    }
  
    //把该套集字对应的channel对象注册到event_loop事件分发器上
    event_loop_add_channel_event(tcpConnection-&gt;eventLoop, connected_fd, tcpConnection-&gt;channel);
    return tcpConnection;
}
</code></pre><p>请注意，到现在为止的操作都是在主线程里执行的。下面的event_loop_do_channel_event也不例外，接下来的行为我期望你是熟悉的，那就是加解锁。</p><p>如果能够获取锁，主线程就会调用event_loop_channel_buffer_nolock往子线程的数据中增加需要处理的channel event对象。所有增加的channel对象以列表的形式维护在子线程的数据结构中。</p><p>接下来的部分是重点，如果当前增加channel event的不是当前event loop线程自己，就会调用event_loop_wakeup函数把event_loop子线程唤醒。唤醒的方法很简单，就是往刚刚的socketPair[0]上写一个字节，别忘了，event_loop已经注册了socketPair[1]的可读事件。如果当前增加channel event的是当前event loop线程自己，则直接调用event_loop_handle_pending_channel处理新增加的channel event事件列表。</p><pre><code>int event_loop_do_channel_event(struct event_loop *eventLoop, int fd, struct channel *channel1, int type) {
    //get the lock
    pthread_mutex_lock(&amp;eventLoop-&gt;mutex);
    assert(eventLoop-&gt;is_handle_pending == 0);
    //往该线程的channel列表里增加新的channel
    event_loop_channel_buffer_nolock(eventLoop, fd, channel1, type);
    //release the lock
    pthread_mutex_unlock(&amp;eventLoop-&gt;mutex);
    //如果是主线程发起操作，则调用event_loop_wakeup唤醒子线程
    if (!isInSameThread(eventLoop)) {
        event_loop_wakeup(eventLoop);
    } else {
        //如果是子线程自己，则直接可以操作
        event_loop_handle_pending_channel(eventLoop);
    }

    return 0;
}
</code></pre><p>如果是event_loop被唤醒之后，接下来也会执行event_loop_handle_pending_channel函数。你可以看到在循环体内从dispatch退出之后，也调用了event_loop_handle_pending_channel函数。</p><pre><code>int event_loop_run(struct event_loop *eventLoop) {
    assert(eventLoop != NULL);

    struct event_dispatcher *dispatcher = eventLoop-&gt;eventDispatcher;

    if (eventLoop-&gt;owner_thread_id != pthread_self()) {
        exit(1);
    }

    yolanda_msgx(&quot;event loop run, %s&quot;, eventLoop-&gt;thread_name);
    struct timeval timeval;
    timeval.tv_sec = 1;

    while (!eventLoop-&gt;quit) {
        //block here to wait I/O event, and get active channels
        dispatcher-&gt;dispatch(eventLoop, &amp;timeval);

        //这里处理pending channel，如果是子线程被唤醒，这个部分也会立即执行到
        event_loop_handle_pending_channel(eventLoop);
    }

    yolanda_msgx(&quot;event loop end, %s&quot;, eventLoop-&gt;thread_name);
    return 0;
}
</code></pre><p>event_loop_handle_pending_channel函数的作用是遍历当前event loop里pending的channel event列表，将它们和event_dispatcher关联起来，从而修改感兴趣的事件集合。</p><p>这里有一个点值得注意，因为event loop线程得到活动事件之后，会回调事件处理函数，这样像onMessage等应用程序代码也会在event loop线程执行，如果这里的业务逻辑过于复杂，就会导致event_loop_handle_pending_channel执行的时间偏后，从而影响I/O的检测。所以，将I/O线程和业务逻辑线程隔离，让I/O线程只负责处理I/O交互，让业务线程处理业务，是一个比较常见的做法。</p><h2>总结</h2><p>在这一讲里，我们重点讲解了框架中涉及多线程的两个重要问题，第一是主线程如何等待多个子线程完成初始化，第二是如何通知处于事件分发中的子线程有新的事件加入、删除、修改。第一个问题通过使用锁和信号量加以解决；第二个问题通过使用socketpair，并将sockerpair作为channel注册到event loop中来解决。</p><h2>思考题</h2><p>和往常一样，给你布置两道思考题：</p><p>第一道， 你可以修改一下代码，让sub-reactor默认的线程个数为cpu*2。</p><p>第二道，当前选择线程的算法是round-robin的算法，你觉得有没有改进的空间？如果改进的话，你可能会怎么做？</p><p>欢迎在评论区写下你的思考，也欢迎把这篇文章分享给你的朋友或者同事，一起交流进步一下。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/9c/62/f625b2bb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>酸葡萄</span>
  </div>
  <div class="_2_QraFYR_0">老师,你好,有个地方不是很明白,<br>为什么event_loop_channel_buffer_nolock(eventLoop, fd, channel1, type);是往子线程的数据中增加需要处理的 channel event 对象呢?<br><br>void event_loop_channel_buffer_nolock(struct event_loop *eventLoop, int fd, struct channel *channel1, int type) {<br>    &#47;&#47;add channel into the pending list<br>    struct channel_element *channelElement = malloc(sizeof(struct channel_element));<br>    channelElement-&gt;channel = channel1;<br>    channelElement-&gt;type = type;&#47;&#47;1 add  (1: add  2: delete)<br>    channelElement-&gt;next = NULL;<br>    &#47;&#47;第一个元素  channel_element是channel的链表，<br>    &#47;&#47; eventLoop pending_head和pending_tail维护的是channelElement的链表<br>    &#47;&#47;这样的话最终还是event_loop包含了channel(event_loop-&gt;channelElement-&gt;channel)<br>    if (eventLoop-&gt;pending_head == NULL) {<br>        eventLoop-&gt;pending_head = eventLoop-&gt;pending_tail = channelElement;<br>    } else {<br>        eventLoop-&gt;pending_tail-&gt;next = channelElement;<br>        eventLoop-&gt;pending_tail = channelElement;<br>    }<br>}<br><br><br>void *event_loop_thread_run(void *arg) {<br>    struct event_loop_thread *eventLoopThread = (struct event_loop_thread *) arg;<br><br>    pthread_mutex_lock(&amp;eventLoopThread-&gt;mutex);<br><br>    &#47;&#47; 初始化化event loop，之后通知主线程<br>    eventLoopThread-&gt;eventLoop = event_loop_init_with_name(eventLoopThread-&gt;thread_name);<br>    yolanda_msgx(&quot;event loop thread init and signal, %s&quot;, eventLoopThread-&gt;thread_name);<br>    pthread_cond_signal(&amp;eventLoopThread-&gt;cond);<br><br>    pthread_mutex_unlock(&amp;eventLoopThread-&gt;mutex);<br><br>    &#47;&#47;子线程event loop run<br>    event_loop_run(eventLoopThread-&gt;eventLoop);<br>}<br>struct event_loop_thread {<br>    struct event_loop *eventLoop;&#47;&#47;主线程和子线程共享<br>    pthread_t thread_tid;        &#47;* thread ID *&#47;<br>    pthread_mutex_t mutex;<br>    pthread_cond_t cond;<br>    char * thread_name;<br>    long thread_count;    &#47;* # connections handled *&#47;<br>};<br><br><br>event_loop_channel_buffer_nolock这个函数中是往eventLoop的链表中注册事件,可是这里的eventLoop是和子线程处理函数<br>event_loop_thread_run中eventLoopThread-&gt;eventLoop不是一个eventLoop啊,这个eventLoopThread-&gt;eventLoop不才是主子线程共享的吗?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我们还是用acceptor线程和I&#47;O线程这样来区分比较好。<br><br>acceptor线程在发现有连接到达后，通过调用event_loop_channel_buffer_nolock函数，往I&#47;O线程的eventLoop里面增加了新的套接字，也就是你说的注册链表。<br><br>这里的关键是每个线程都是一个独立的eventLoop，acceptor有自己的eventLoop，I&#47;O线程有自己的eventLoop。没有主子线程共享eventLoop，一个eventLoop就对应一个线程。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-11 23:17:53</div>
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
  <div class="_2_QraFYR_0">为什么不直接让子线程自己调用accept而要主线程调用呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 那主线程干啥呢？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-11 17:40:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/d0/48/0a865673.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>时间</span>
  </div>
  <div class="_2_QraFYR_0">线程池个数有限，如何处理成千上万的链接？假如线程池共四个线程，正在处理四个链接。再来一个链接如何处理呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 好问题。<br><br>线程不是每时每刻都要干活的，就好比一个流水线工人，只有轮到他的时候，他才需要出力干活。如果这四个连接都在干活，那第五个只好等任意一个线程空闲出来。<br><br>所有的事情，秘诀都在于&quot;分时复用&quot;，比如你的cpu，也就4个core，为啥同时可以打游戏，写文稿，看电影，跑程序.....想通了这个，就想通了如何处理多个连接了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-23 23:49:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/73/9b/67a38926.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>keepgoing</span>
  </div>
  <div class="_2_QraFYR_0">想问问老师关于基础语法的问题，代码里很多地方对象都是相互引用的，比如tcp_connection里引用了channel指针, channel 对象里引用了tcp_connection指针, dispatcher里引用了event_loop指针, event_loop里也引用了dispatcher指针。这样代码编译的时候为什么不会引起报错。。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 为什么会报错呢？每个指针就是一个内存地址。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-24 13:14:04</div>
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
  <div class="_2_QraFYR_0">第一道， 可以直接在应用层上将输入的线程个数*2 。  第二道，(1)可以判断已经创建好的线程 那个线程的事件个数最少，挂在事件最少的那个线程上。 </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-15 11:46:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/23/c1/54ef6885.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>MoonGod</span>
  </div>
  <div class="_2_QraFYR_0">老师关于加锁这里有个疑问，如果加锁的目的是让主线程等待子线程初始化event loop。那不加锁不是也可以达到这个目的吗？主线程while 循环里面不断判断子线程的event loop是否不为null不就可以了？为啥一定要加一把锁呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 好问题， 我答疑统一回答吧。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-23 09:43:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/IPdZZXuHVMibwfZWmm7NiawzeEFGsaRoWjhuN99iaoj5amcRkiaOePo6rH1KJ3jictmNlic4OibkF4I20vOGfwDqcBxfA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>鱼向北游</span>
  </div>
  <div class="_2_QraFYR_0">netty选子线程是两种算法，都是有个原子自增计数，如果线程数不是2的幂用取模，如果是就是按位与线程数减一</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，涨知识了，代码贴一个？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-23 09:37:43</div>
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
  <div class="_2_QraFYR_0">老师你好 最近在用C++重写示例代码。我发现个问题。在C代码中 不同线程同时在poll同一份event_set。如果poll函数时间参数写成-1。多启几个终端执行nc指令。经常会出现没有callback的现象。如果poll不是同一份event_set 就没有这个问题。网上也有别人的一些讨论。对此老师怎么看。望回复。https:&#47;&#47;stackoverflow.com&#47;questions&#47;18891500&#47;multiple-threads-doing-poll-or-select-on-a-single-socket-or-pipe</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-07 22:59:12</div>
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
  <div class="_2_QraFYR_0">老师您好，子线程channel以及channel_element对象都是动态分配的内存，但在连接close后并未看到释放，是否是内存泄露了？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-01 23:00:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/28/d2/c7/b7f52df2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>雨里</span>
  </div>
  <div class="_2_QraFYR_0">没有看明白主从reactor这个主线程是如何唤醒子线程的？？？<br>1、就单reactor而言，主线程创建管道fd，正常来说应该是在epoll_wait上注册0端读事件，往管道1端写数据的方式来唤醒epoll。<br>2、而主从reactor代码来看，主线程和子线程都创建了一对pairfd，主线程的管道1端注册在主线程的epoll上，这样即使往管道中写数据，也只是唤醒主线程，怎么会唤醒子线程呢？？，代码中好像没有将主线程的管道fd一端注册在子线程的epoll上。是不是下面的这行代码导致的<br>eventLoop-&gt;eventDispatcher = &amp;poll_dispatcher;<br>主线程和子线程共用一个同一个poll_dispatcher对象，还是没有看出在哪个地方传递的fd??</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 主线程，可以往子线程的管道上写数据，从而唤醒子线程。<br>传递fd的代码在这个函数里。<br>int event_loop_do_channel_event(struct event_loop *eventLoop, int fd, struct channel *channel1, int type) {<br>    &#47;&#47;get the lock<br>    pthread_mutex_lock(&amp;eventLoop-&gt;mutex);<br>    assert(eventLoop-&gt;is_handle_pending == 0);<br>    event_loop_channel_buffer_nolock(eventLoop, fd, channel1, type);<br>    &#47;&#47;release the lock<br>    pthread_mutex_unlock(&amp;eventLoop-&gt;mutex);<br>    if (!isInSameThread(eventLoop)) {<br>        event_loop_wakeup(eventLoop);<br>    } else {<br>        event_loop_handle_pending_channel(eventLoop);<br>    }<br><br>    return 0;<br>}</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-28 17:20:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/6b/aa/ec09c4b4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>zssdhr</span>
  </div>
  <div class="_2_QraFYR_0">老师，关于 event_loop_thread 有两个问题。<br><br>1. 为什么主线程要等待子线程初始化完成？是担心 tcp_server_init 后、但子线程还未初始化完成时，thread_pool_get_loop 无法找到子线程来处理新来的连接吗？<br><br>2. 文中提到”你可能会问，主线程是循环在等待每个子线程完成初始化，如果进入第二个循环，等待第二个子线程完成初始化，而此时第二个子线程已经初始化完成了，该怎么办？“<br>主线程不是等第一个子线程初始化完成后才会进入下一个循环启动第二个子线程吗？怎么会出现”而此时第二个子线程已经初始化完成了“？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1.可以这么认为；<br>2.你说的没错，是按照顺序的，第一个完成，再第二个，问题是到第二个以后，如何判断第二个已经完成初始化了，因为这里的线程都是异步的，所以有可能线程初始化完成了，才进入判断。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-20 14:35:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/28/b4/3c/e4a08d98.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Janus Pen</span>
  </div>
  <div class="_2_QraFYR_0">老师：event_loop_do_channel_event函数中的event_loop_handle_pending_channel函数调用与event_loop_run函数中的event_loop_handle_pending_channel函数调用是否重复?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不会。<br><br>两个的用处不一样，其中，event_loop_run中，是为了处理所有待添加的事件，这个肯定是在dispatch线程中执行的。<br><br>而event_loop_do_channel_event，是既可能在dispatch线程，也可能不在，而在dispatch线程中，每次调用event_loop_handle_pending_channel完成事件的实时添加，如果不是，就唤醒dispatch线程，让它自己完成添加。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-10 22:39:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/25/85/34/ad4cbfe4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>消失的时光</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，不是很理解为什么要socketpair唤醒，直接把新连接的socket加到epoll里面，有发送就的数据过来，这个线程自己不会醒吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: epoll不允许这么干。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-07 23:53:30</div>
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
  <div class="_2_QraFYR_0">老师你好 关于第二点 是不是相当于没有需要遍历的描述符 导致一直卡在poll或者select上。所以手动构造socketpair作为初始描述符。再添加真正新的描述符时 用socketpair把程序从poll或者select阻塞上解放出来 以获取达到添加描述符的时机？我的理解对吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-05 16:43:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/ed/6c/6fb35017.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>群书</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，逻辑线程写数据到发送队列，同时通知唤醒io线程，这个通知方式目前比较常规的做法是套接字对或者事件fd 实际测试下来 都会增加主线程的系统调用 有什么优化办法呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 常规做法是这样的，我还想不出超常规做法😢。<br><br>这个不会增加系统负担，不用担心。连Netty都是这么干的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-12 11:11:09</div>
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
  <div class="_2_QraFYR_0">主线程和丛线程不是共享内存吗？为什么还要socketpair唤醒呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 唤醒的目的是让主线程那个负责分发的家伙醒来干活，不要继续等待了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-04 07:49:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/82/3d/356fc3d6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>忆水寒</span>
  </div>
  <div class="_2_QraFYR_0">老师，main-EventLoop和sub-EventLoop里面的eventLoop-&gt;eventDispatcher = &amp;epoll_dispatcher;都是指向一个epoll_dispatcher。其中main-EventLoop用于accept新连接，获得新连接封装channel交给某一个sub-EventLoop去处理。假如dispatch有事件，是不是子线程也会从dispatch处惊醒，这是不是有“惊群效应”吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我认为不会。原因是每个eventloop对应一个独立的dispatch。虽然公用的是一个epoll_dispatcher，但是你可以注意到在调用epoll_wait函数时，是取自各自event_loop里的独立数据。<br><br>    epoll_dispatcher_data *epollDispatcherData = (epoll_dispatcher_data *) eventLoop-&gt;event_dispatcher_data;<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-06 17:34:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/d5/70/93a34aa5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_76f04f</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，我有个问题想咨询一下，我看资料说线程或者进程需要绑定内核，减少上下文切换，像这种reactor模型中，如果开辟corenum个线程，一般需要绑定内核吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我认为不需要</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-02 20:08:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/db/64/06d54a80.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>中年男子</span>
  </div>
  <div class="_2_QraFYR_0">event_loop_init中，代码片段：<br>event_loop_add_channel_event(eventLoop, eventLoop-&gt;socketPair[0], channel);<br>其中传入的应该是socketPair[1]<br>文稿中的代码还未修正，另外我认为这个fd作为参数实际上没有意义，event_loop_add_channel_event 往后调用的几个函数里实际上都用不到这个fd，只需要channel 就可以了，因为channel里已有这个fd。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 代码已经修复，文稿我再看下哈。<br><br>你说的是对的，channel里确实有这个fd，这里是为了突出这个作用。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-24 11:27:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/18/2a/9c18a3c4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>wzp</span>
  </div>
  <div class="_2_QraFYR_0">干货满满，有收获</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-19 23:01:29</div>
  </div>
</div>
</div>
</li>
</ul>