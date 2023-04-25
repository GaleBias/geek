<audio title="32 _ 自己动手写高性能HTTP服务器（一）：设计和思路" src="https://static001.geekbang.org/resource/audio/fe/f1/fe2a0af5589cbe6648d8dcd62c55a6f1.mp3" controls="controls"></audio> 
<p>你好，我是盛延敏，这里是网络编程实战第32讲，欢迎回来。</p><p>从这一讲开始，我们进入实战篇，开启一个高性能HTTP服务器的编写之旅。</p><p>在开始编写高性能HTTP服务器之前，我们先要构建一个支持TCP的高性能网络编程框架，完成这个TCP高性能网络框架之后，再增加HTTP特性的支持就比较容易了，这样就可以很快开发出一个高性能的HTTP服务器程序。</p><h2>设计需求</h2><p>在第三个模块性能篇中，我们已经使用这个网络编程框架完成了多个应用程序的开发，这也等于对网络编程框架提出了编程接口方面的需求。综合之前的使用经验，TCP高性能网络框架需要满足的需求有以下三点。</p><p>第一，采用reactor模型，可以灵活使用poll/epoll作为事件分发实现。</p><p>第二，必须支持多线程，从而可以支持单线程单reactor模式，也可以支持多线程主-从reactor模式。可以将套接字上的I/O事件分离到多个线程上。</p><p>第三，封装读写操作到Buffer对象中。</p><p>按照这三个需求，正好可以把整体设计思路分成三块来讲解，分别包括反应堆模式设计、I/O模型和多线程模型设计、数据读写封装和buffer。今天我们主要讲一下主要的设计思路和数据结构，以及反应堆模式设计。</p><!-- [[[read_end]]] --><h2>主要设计思路</h2><h3>反应堆模式设计</h3><p>反应堆模式，按照性能篇的讲解，主要是设计一个基于事件分发和回调的反应堆框架。这个框架里面的主要对象包括：</p><ul>
<li>
<h3>event_loop</h3>
</li>
</ul><p>你可以把event_loop这个对象理解成和一个线程绑定的无限事件循环，你会在各种语言里看到event_loop这个抽象。这是什么意思呢？简单来说，它就是一个无限循环着的事件分发器，一旦有事件发生，它就会回调预先定义好的回调函数，完成事件的处理。</p><p>具体来说，event_loop使用poll或者epoll方法将一个线程阻塞，等待各种I/O事件的发生。</p><ul>
<li>
<h3>channel</h3>
</li>
</ul><p>对各种注册到event_loop上的对象，我们抽象成channel来表示，例如注册到event_loop上的监听事件，注册到event_loop上的套接字读写事件等。在各种语言的API里，你都会看到channel这个对象，大体上它们表达的意思跟我们这里的设计思路是比较一致的。</p><ul>
<li>
<h3>acceptor</h3>
</li>
</ul><p>acceptor对象表示的是服务器端监听器，acceptor对象最终会作为一个channel对象，注册到event_loop上，以便进行连接完成的事件分发和检测。</p><ul>
<li>
<h3>event_dispatcher</h3>
</li>
</ul><p>event_dispatcher是对事件分发机制的一种抽象，也就是说，可以实现一个基于poll的poll_dispatcher，也可以实现一个基于epoll的epoll_dispatcher。在这里，我们统一设计一个event_dispatcher结构体，来抽象这些行为。</p><ul>
<li>
<h3>channel_map</h3>
</li>
</ul><p>channel_map保存了描述字到channel的映射，这样就可以在事件发生时，根据事件类型对应的套接字快速找到channel对象里的事件处理函数。</p><h3>I/O模型和多线程模型设计</h3><p>I/O线程和多线程模型，主要解决event_loop的线程运行问题，以及事件分发和回调的线程执行问题。</p><ul>
<li>
<h3>thread_pool</h3>
</li>
</ul><p>thread_pool维护了一个sub-reactor的线程列表，它可以提供给主reactor线程使用，每次当有新的连接建立时，可以从thread_pool里获取一个线程，以便用它来完成对新连接套接字的read/write事件注册，将I/O线程和主reactor线程分离。</p><ul>
<li>
<h3>event_loop_thread</h3>
</li>
</ul><p>event_loop_thread是reactor的线程实现，连接套接字的read/write事件检测都是在这个线程里完成的。</p><h3>Buffer和数据读写</h3><ul>
<li>
<h3>buffer</h3>
</li>
</ul><p>buffer对象屏蔽了对套接字进行的写和读的操作，如果没有buffer对象，连接套接字的read/write事件都需要和字节流直接打交道，这显然是不友好的。所以，我们也提供了一个基本的buffer对象，用来表示从连接套接字收取的数据，以及应用程序即将需要发送出去的数据。</p><ul>
<li>
<h3>tcp_connection</h3>
</li>
</ul><p>tcp_connection这个对象描述的是已建立的TCP连接。它的属性包括接收缓冲区、发送缓冲区、channel对象等。这些都是一个TCP连接的天然属性。</p><p>tcp_connection是大部分应用程序和我们的高性能框架直接打交道的数据结构。我们不想把最下层的channel对象暴露给应用程序，因为抽象的channel对象不仅仅可以表示tcp_connection，前面提到的监听套接字也是一个channel对象，后面提到的唤醒socketpair也是一个 channel对象。所以，我们设计了tcp_connection这个对象，希望可以提供给用户比较清晰的编程入口。</p><h2>反应堆模式设计</h2><h3>概述</h3><p>下面，我们详细讲解一下以event_loop为核心的反应堆模式设计。这里有一张event_loop的运行详图，你可以对照这张图来理解。</p><p><img src="https://static001.geekbang.org/resource/image/7a/61/7ab9f89544aba2021a9d2ceb94ad9661.jpg?wh=3499*1264" alt=""></p><p>当event_loop_run完成之后，线程进入循环，首先执行dispatch事件分发，一旦有事件发生，就会调用channel_event_activate函数，在这个函数中完成事件回调函数eventReadcallback和eventWritecallback的调用，最后再进行event_loop_handle_pending_channel，用来修改当前监听的事件列表，完成这个部分之后，又进入了事件分发循环。</p><h3>event_loop分析</h3><p>说event_loop是整个反应堆模式设计的核心，一点也不为过。先看一下event_loop的数据结构。</p><p>在这个数据结构中，最重要的莫过于event_dispatcher对象了。你可以简单地把event_dispatcher理解为poll或者epoll，它可以让我们的线程挂起，等待事件的发生。</p><p>这里有一个小技巧，就是event_dispatcher_data，它被定义为一个void *类型，可以按照我们的需求，任意放置一个我们需要的对象指针。这样，针对不同的实现，例如poll或者epoll，都可以根据需求，放置不同的数据对象。</p><p>event_loop中还保留了几个跟多线程有关的对象，如owner_thread_id是保留了每个event loop的线程ID，mutex和con是用来进行线程同步的。</p><p>socketPair是父线程用来通知子线程有新的事件需要处理。pending_head和pending_tail是保留在子线程内的需要处理的新事件。</p><pre><code>struct event_loop {
    int quit;
    const struct event_dispatcher *eventDispatcher;

    /** 对应的event_dispatcher的数据. */
    void *event_dispatcher_data;
    struct channel_map *channelMap;

    int is_handle_pending;
    struct channel_element *pending_head;
    struct channel_element *pending_tail;

    pthread_t owner_thread_id;
    pthread_mutex_t mutex;
    pthread_cond_t cond;
    int socketPair[2];
    char *thread_name;
};
</code></pre><p>下面我们看一下event_loop最主要的方法event_loop_run方法，前面提到过，event_loop就是一个无限while循环，不断地在分发事件。</p><pre><code>/**
 *
 * 1.参数验证
 * 2.调用dispatcher来进行事件分发,分发完回调事件处理函数
 */
int event_loop_run(struct event_loop *eventLoop) {
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

        //handle the pending channel
        event_loop_handle_pending_channel(eventLoop);
    }

    yolanda_msgx(&quot;event loop end, %s&quot;, eventLoop-&gt;thread_name);
    return 0;
}
</code></pre><p>代码很明显地反映了这一点，这里我们在event_loop不退出的情况下，一直在循环，循环体中调用了dispatcher对象的dispatch方法来等待事件的发生。</p><h3>event_dispacher分析</h3><p>为了实现不同的事件分发机制，这里把poll、epoll等抽象成了一个event_dispatcher结构。event_dispatcher的具体实现有poll_dispatcher和epoll_dispatcher两种，实现的方法和性能篇<a href="https://time.geekbang.org/column/article/140520">21</a><a href="https://time.geekbang.org/column/article/140520">讲</a>和<a href="https://time.geekbang.org/column/article/141573">22讲</a>类似，这里就不再赘述，你如果有兴趣的话，可以直接研读代码。</p><pre><code>/** 抽象的event_dispatcher结构体，对应的实现如select,poll,epoll等I/O复用. */
struct event_dispatcher {
    /**  对应实现 */
    const char *name;

    /**  初始化函数 */
    void *(*init)(struct event_loop * eventLoop);

    /** 通知dispatcher新增一个channel事件*/
    int (*add)(struct event_loop * eventLoop, struct channel * channel);

    /** 通知dispatcher删除一个channel事件*/
    int (*del)(struct event_loop * eventLoop, struct channel * channel);

    /** 通知dispatcher更新channel对应的事件*/
    int (*update)(struct event_loop * eventLoop, struct channel * channel);

    /** 实现事件分发，然后调用event_loop的event_activate方法执行callback*/
    int (*dispatch)(struct event_loop * eventLoop, struct timeval *);

    /** 清除数据 */
    void (*clear)(struct event_loop * eventLoop);
};
</code></pre><h3>channel对象分析</h3><p>channel对象是用来和event_dispather进行交互的最主要的结构体，它抽象了事件分发。一个channel对应一个描述字，描述字上可以有READ可读事件，也可以有WRITE可写事件。channel对象绑定了事件处理函数event_read_callback和event_write_callback。</p><pre><code>typedef int (*event_read_callback)(void *data);

typedef int (*event_write_callback)(void *data);

struct channel {
    int fd;
    int events;   //表示event类型

    event_read_callback eventReadCallback;
    event_write_callback eventWriteCallback;
    void *data; //callback data, 可能是event_loop，也可能是tcp_server或者tcp_connection
};
</code></pre><h3>channel_map对象分析</h3><p>event_dispatcher在获得活动事件列表之后，需要通过文件描述字找到对应的channel，从而回调channel上的事件处理函数event_read_callback和event_write_callback，为此，设计了channel_map对象。</p><pre><code>/**
 * channel映射表, key为对应的socket描述字
 */
struct channel_map {
    void **entries;

    /* The number of entries available in entries */
    int nentries;
};
</code></pre><p>channel_map对象是一个数组，数组的下标即为描述字，数组的元素为channel对象的地址。</p><p>比如描述字3对应的channel，就可以这样直接得到。</p><pre><code>struct chanenl * channel = map-&gt;entries[3];
</code></pre><p>这样，当event_dispatcher需要回调channel上的读、写函数时，调用channel_event_activate就可以，下面是channel_event_activate的实现，在找到了对应的channel对象之后，根据事件类型，回调了读函数或者写函数。注意，这里使用了EVENT_READ和EVENT_WRITE来抽象了poll和epoll的所有读写事件类型。</p><pre><code>int channel_event_activate(struct event_loop *eventLoop, int fd, int revents) {
    struct channel_map *map = eventLoop-&gt;channelMap;
    yolanda_msgx(&quot;activate channel fd == %d, revents=%d, %s&quot;, fd, revents, eventLoop-&gt;thread_name);

    if (fd &lt; 0)
        return 0;

    if (fd &gt;= map-&gt;nentries)return (-1);

    struct channel *channel = map-&gt;entries[fd];
    assert(fd == channel-&gt;fd);

    if (revents &amp; (EVENT_READ)) {
        if (channel-&gt;eventReadCallback) channel-&gt;eventReadCallback(channel-&gt;data);
    }
    if (revents &amp; (EVENT_WRITE)) {
        if (channel-&gt;eventWriteCallback) channel-&gt;eventWriteCallback(channel-&gt;data);
    }

    return 0;
}
</code></pre><h3>增加、删除、修改channel event</h3><p>那么如何增加新的channel event事件呢？下面这几个函数是用来增加、删除和修改channel event事件的。</p><pre><code>int event_loop_add_channel_event(struct event_loop *eventLoop, int fd, struct channel *channel1);

int event_loop_remove_channel_event(struct event_loop *eventLoop, int fd, struct channel *channel1);

int event_loop_update_channel_event(struct event_loop *eventLoop, int fd, struct channel *channel1);
</code></pre><p>前面三个函数提供了入口能力，而真正的实现则落在这三个函数上：</p><pre><code>int event_loop_handle_pending_add(struct event_loop *eventLoop, int fd, struct channel *channel);

int event_loop_handle_pending_remove(struct event_loop *eventLoop, int fd, struct channel *channel);

int event_loop_handle_pending_update(struct event_loop *eventLoop, int fd, struct channel *channel);
</code></pre><p>我们看一下其中的一个实现，event_loop_handle_pending_add在当前event_loop的channel_map里增加一个新的key-value对，key是文件描述字，value是channel对象的地址。之后调用event_dispatcher对象的add方法增加channel event事件。注意这个方法总在当前的I/O线程中执行。</p><pre><code>// in the i/o thread
int event_loop_handle_pending_add(struct event_loop *eventLoop, int fd, struct channel *channel) {
    yolanda_msgx(&quot;add channel fd == %d, %s&quot;, fd, eventLoop-&gt;thread_name);
    struct channel_map *map = eventLoop-&gt;channelMap;

    if (fd &lt; 0)
        return 0;

    if (fd &gt;= map-&gt;nentries) {
        if (map_make_space(map, fd, sizeof(struct channel *)) == -1)
            return (-1);
    }

    //第一次创建，增加
    if ((map)-&gt;entries[fd] == NULL) {
        map-&gt;entries[fd] = channel;
        //add channel
        struct event_dispatcher *eventDispatcher = eventLoop-&gt;eventDispatcher;
        eventDispatcher-&gt;add(eventLoop, channel);
        return 1;
    }

    return 0;
}
</code></pre><h2>总结</h2><p>在这一讲里，我们介绍了高性能网络编程框架的主要设计思路和基本数据结构，以及反应堆设计相关的具体做法。在接下来的章节中，我们将继续编写高性能网络编程框架的线程模型以及读写Buffer部分。</p><h2>思考题</h2><p>和往常一样，给你留两道思考题:</p><p>第一道，如果你有兴趣，不妨实现一个select_dispatcher对象，用select方法实现定义好的event_dispatcher接口；</p><p>第二道，仔细研读channel_map实现中的map_make_space部分，说说你的理解。</p><p>欢迎你在评论区写下你的思考，也欢迎把这篇文章分享给你的朋友或者同事，一起交流一下。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/23/66/413c0bb5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>LDxy</span>
  </div>
  <div class="_2_QraFYR_0">event_loop_handle_pending_add函数中，<br>map-&gt;entries[fd] = calloc(1, sizeof(struct channel *));<br>map-&gt;entries[fd] = channel;<br>这两行都给map-&gt;entries[fd] 赋值，后一行不是覆盖上一行的赋值了么？有何用意？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这块是我的疏忽，应该直接赋值的，可能是开始我撰写的时候channel对象的初始化放到了event_loop_handle_pending_add函数中，后来把channel对象的初始化重构到外面，这里忘记删掉了。<br><br>已经更新文稿(待周一编辑更新)和github代码，感谢指正。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-21 19:30:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/00/4e/be2b206b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>吴小智</span>
  </div>
  <div class="_2_QraFYR_0">map_make_space() 函数里 realloc() 和 memset() 两个函数用的很巧妙啊，realloc() 用来扩容，且把旧的内容搬过去，memset() 用来给新申请的内存赋 0 值。赞，C 语言太强大了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 显然是看进去了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-22 11:16:38</div>
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
  <div class="_2_QraFYR_0">第二道题 就是一个扩容啊 类似std的vector自动扩容 而且每次成倍的增长</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-21 11:26:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/9c/62/f625b2bb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>酸葡萄</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，问个基础的问题：<br>epoll_dispatcher和poll_dispatcher都有，在添加，删除，更新事件时都有如下的逻辑，其中if条件中的判断怎么理解啊？<br>if (channel1-&gt;events &amp; EVENT_READ) {<br>        events = events | POLLRDNORM;<br>    }<br><br>    if (channel1-&gt;events &amp; EVENT_WRITE) {<br>        events = events | POLLWRNORM;<br>    }</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这里是位与操作，举个例子，EVENT_READ可能为二进制的00000010，如果有可读事件发生，那么在这个位上的bit值一定位1，这样位与的结果就说明这个事件发生了。之所以采用位与，而不是位或，是因为只需要关心这一种类型的事件。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-01 00:47:00</div>
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
  <div class="_2_QraFYR_0">int map_make_space(struct channel_map *map, int slot, int msize) {<br>    if (map-&gt;nentries &lt;= slot) {<br>        int nentries = map-&gt;nentries ? map-&gt;nentries : 32;<br>        void **tmp;<br><br>        while (nentries &lt;= slot)<br>            nentries &lt;&lt;= 1;<br><br>        tmp = (void **) realloc(map-&gt;entries, nentries * msize);<br>        if (tmp == NULL)<br>            return (-1);<br><br>        memset(&amp;tmp[map-&gt;nentries], 0,<br>               (nentries - map-&gt;nentries) * msize);<br><br>        map-&gt;nentries = nentries;<br>        map-&gt;entries = tmp;<br>    }<br><br>    return (0);<br>}<br>老师，fd不一定是连续的吧，这样会浪费内存存储空间吧？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 很好的问题。<br><br>第一，你可以看看实际分配的fd，大概会是什么样子；<br>第二，除了这个方法，你有别的更高效的方法吗？因为从fd来查找数据，需要非常的快；<br><br>我个人的判断，这点内存不算什么，因为我在设计这个结构时，大部分数据都是指针类型的。<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-07 11:26:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/25/2b/ca/71ff1fd7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>谁家内存泄露了</span>
  </div>
  <div class="_2_QraFYR_0">老师好，请问您的代码中关于锁的使用，我想知道您关于每个loop都设计了一个锁，可是这几个mutex都是局部变量吧？他们的作用范围是什么样的呢？这里想不清楚，请指点一下！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 所有的作用范围是全局的，而mutex锁看情况，有些是线程级别的，比如这里：<br>    pthread_mutex_lock(&amp;eventLoopThread-&gt;mutex);</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-12 22:03:46</div>
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
  <div class="_2_QraFYR_0">如果Channel是一个管道，他连接着哪两个对象？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 连接着client端的套接字和server端的套接字。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-18 21:27:17</div>
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
  <div class="_2_QraFYR_0">看到map_make_space里面的realloc函数，突然有个疑问，既然操作系统底层支持直接在原数组上扩充内存，为什么Java不支持直接在原数组上扩容呢，ArrayList每次扩容都要重新拷贝一份原来的数据。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 好问题，我试着解读一下:<br>第一，Java有JVM实现，在Java的世界里，它的对象是统一被JVM管理的，包括GC，对象管理，基于这一层考虑，它不可能使用系统级别的对象内存管理，这两个没有办法调和，就像你举的例子，如果我们创建一个类似ReallocList对象，那么这个对象的内存管理完全不是JVM那套了；<br><br>第二，JVM是一个跨OS的实现，我不知道是否Windows也有类似realloc函数，这样就需要JVM做到跨OS的直接内存接管，这和Java的思想是不一致的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-22 16:44:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/29/b0/d3/200e82ff.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>功夫熊猫</span>
  </div>
  <div class="_2_QraFYR_0">老师，套接字不是用于进城通信的嘛，线程也能用？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-27 18:44:42</div>
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
  <div class="_2_QraFYR_0">再来看看 重温重温。感谢老师 这个教程 是我入门网络编程的领路人。我是做iOS开发的 因为一些原因转到网络这边 一开始一头懵逼 学习了老师的教程就清晰了很多。后面接触到的知识 老师的教程都能引申到 真的很赞。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-26 00:30:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/28/97/dc/8eacc8f1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>漠博嵩</span>
  </div>
  <div class="_2_QraFYR_0">感觉就是仿照netty框架做的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 还真不是。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-24 08:41:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLmWgscKlnjXiaBugNJ2ozMmZibAEKichZv7OfGwQX9voDicVy2qnKtlm5kWQAKZ414vFohR8FV5N9ZhA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>菜鸡</span>
  </div>
  <div class="_2_QraFYR_0">第二个问题有点疑问。channel_map中元素的空间大小是与fd的值正相关的，而不是跟当前在线的连接数量正相关，这样做是不是有点浪费内存？比如经历了很多次连接、断开之后，fd返回的值比较大，而此时只有几个未断开的连接，那么channel_map有必要申请那么大的内存空间嘛？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 还好吧，channel_map中的元素没几个字节，比起复杂的压缩算法，这点实在是微不足道。而且，你也没办法预测后面的连接情况，准备好一个上限比较大的fd存储空间，其实是效率比较高的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-08 21:58:44</div>
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
  <div class="_2_QraFYR_0">用sock对通知 唤醒会不会增加逻辑线程或主线程的系统调用次数 限制了吞吐量呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不会。因为唤醒是kernel干的，只不过是多了一路I&#47;O而已。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-12 11:06:28</div>
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
  <div class="_2_QraFYR_0">我看了下定义，channel_element就像是个链表节点，为什么不用C++来做这块呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 因为是C的代码，通篇都是C语言，不需要学习C++知识。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-18 16:15:19</div>
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
  <div class="_2_QraFYR_0">老师请问这个channel就相当于libevent中的event结构体吧？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，差不多的意思。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-05 21:42:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/49/96/7523cdb6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>spark</span>
  </div>
  <div class="_2_QraFYR_0">盛老师好: 为什么要在下面这个函数中lock和unlock? 不是每个线程都对应一个自己的event_loop吗?<br>这样的话event_loop就不是shared resource。<br>int event_loop_handle_pending_channel(struct event_loop *eventLoop) {<br>    &#47;&#47;get the lock<br>    pthread_mutex_lock(&amp;eventLoop-&gt;mutex);<br>    eventLoop-&gt;is_handle_pending = 1;<br><br>    struct channel_element *channelElement = eventLoop-&gt;pending_head;<br>    while (channelElement != NULL) {<br>        &#47;&#47;save into event_map<br>        struct channel *channel = channelElement-&gt;channel;<br>        int fd = channel-&gt;fd;<br>        if (channelElement-&gt;type == 1) {<br>            event_loop_handle_pending_add(eventLoop, fd, channel);<br>        } else if (channelElement-&gt;type == 2) {<br>            event_loop_handle_pending_remove(eventLoop, fd, channel);<br>        } else if (channelElement-&gt;type == 3) {<br>            event_loop_handle_pending_update(eventLoop, fd, channel);<br>        }<br>        channelElement = channelElement-&gt;next;<br>    }<br><br>    eventLoop-&gt;pending_head = eventLoop-&gt;pending_tail = NULL;<br>    eventLoop-&gt;is_handle_pending = 0;<br><br>    &#47;&#47;release the lock<br>    pthread_mutex_unlock(&amp;eventLoop-&gt;mutex);<br><br>    return 0;<br>}</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的呀，因为每个线程对应自己的event_loop对象，而发起event_loop_handle_pending_channel操作的线程可能是不同的线程，所以使用的是线程级别的lock，也就是event_loop里面的lock。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-25 21:56:34</div>
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
  <div class="_2_QraFYR_0">channel_map这里map-&gt;entries是一个数组，数组的下标是fd,数组的元素是channel的地址，如果新增的fd跳变很大的话比如从3变成了100，会不会浪费了很多的空间</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是一个问题。不过，第一，跳变不是非常巨大，这个可以从实际的程序运行可以看到fd的增长，一个合理的解释是OS也是在&quot;智能&quot;的分配文件描述字；第二，即使跳变，一个channel地址也就是8个字节(64bit OS)，占用的内存也不是特别多。<br><br>好处fd到channel的查询，是非常非常快的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-23 22:50:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/ab/cb/55eddaf1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>胤</span>
  </div>
  <div class="_2_QraFYR_0">问个c语言的问题，比如event_loop_handle_pending_channel这个函数，返回值是int类型，但是除了函数最后是个return 0，其他地方没有错误处理，为什么要返回0？还是就是一种习惯？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 好问题。<br><br>按照道理说，返回值非0表示有错误信息，只是这里我没有返回而已。个人习惯哈，你可以改成无返回值。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-04 21:52:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/b3/17/19ea024f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>chs</span>
  </div>
  <div class="_2_QraFYR_0">老师，您为了支持poll和epoll，抽象出了struct event_dispatcher结构体，然后在epoll_dispatcher.c 和poll_dispatcher.c中分别实现struct event_dispatcher中定义的接口。请问epoll_dispatcher.c中的 const struct event_dispatcher epoll_dispatcher变量 和poll_dispatcher.c中的const struct event_dispatcher poll_dispatcher变量怎样让其他文件知道其定义的。我自己写的提示上边两个变量未定义。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个是通过下面的方式在头文件中声明：<br><br>extern const struct event_dispatcher poll_dispatcher;<br>extern const struct event_dispatcher epoll_dispatcher;<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-24 12:40:40</div>
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
  <div class="_2_QraFYR_0">请问channel里的fd也需要设置为非阻塞吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: channel里的fd是在有连接建立时创建好的，当然，也是被设置为非阻塞的，这里channle对象不需要关系fd的属性。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-22 23:17:23</div>
  </div>
</div>
</div>
</li>
</ul>