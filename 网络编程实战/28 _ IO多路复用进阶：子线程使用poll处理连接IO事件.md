<audio title="28 _ IO多路复用进阶：子线程使用poll处理连接IO事件" src="https://static001.geekbang.org/resource/audio/f3/e5/f30571eec9ba0ed5e361501f816742e5.mp3" controls="controls"></audio> 
<p>你好，我是盛延敏，这里是网络编程实战第28讲，欢迎回来。</p><p>在前面的第27讲中，我们引入了reactor反应堆模式，并且让reactor反应堆同时分发Acceptor上的连接建立事件和已建立连接的I/O事件。</p><p>我们仔细想想这种模式，在发起连接请求的客户端非常多的情况下，有一个地方是有问题的，那就是单reactor线程既分发连接建立，又分发已建立连接的I/O，有点忙不过来，在实战中的表现可能就是客户端连接成功率偏低。</p><p>再者，新的硬件技术不断发展，多核多路CPU已经得到极大的应用，单reactor反应堆模式看着大把的CPU资源却不用，有点可惜。</p><p>这一讲我们就将acceptor上的连接建立事件和已建立连接的I/O事件分离，形成所谓的主-从reactor模式。</p><h2>主-从reactor模式</h2><p>下面的这张图描述了主-从reactor模式是如何工作的。</p><p>主-从这个模式的核心思想是，主反应堆线程只负责分发Acceptor连接建立，已连接套接字上的I/O事件交给sub-reactor负责分发。其中sub-reactor的数量，可以根据CPU的核数来灵活设置。</p><p>比如一个四核CPU，我们可以设置sub-reactor为4。相当于有4个身手不凡的反应堆线程同时在工作，这大大增强了I/O分发处理的效率。而且，同一个套接字事件分发只会出现在一个反应堆线程中，这会大大减少并发处理的锁开销。</p><!-- [[[read_end]]] --><p><img src="https://static001.geekbang.org/resource/image/92/2a/9269551b14c51dc9605f43d441c5a92a.png?wh=1026*1108" alt=""><br>
我来解释一下这张图，我们的主反应堆线程一直在感知连接建立的事件，如果有连接成功建立，主反应堆线程通过accept方法获取已连接套接字，接下来会按照一定的算法选取一个从反应堆线程，并把已连接套接字加入到选择好的从反应堆线程中。</p><p>主反应堆线程唯一的工作，就是调用accept获取已连接套接字，以及将已连接套接字加入到从反应堆线程中。不过，这里还有一个小问题，主反应堆线程和从反应堆线程，是两个不同的线程，如何把已连接套接字加入到另外一个线程中呢？更令人沮丧的是，此时从反应堆线程或许处于事件分发的无限循环之中，在这种情况下应该怎么办呢？</p><p>我在这里先卖个关子，这是高性能网络程序框架要解决的问题。在实战篇里，我将为这些问题一一解开答案。</p><h2>主-从reactor+worker threads模式</h2><p>如果说主-从reactor模式解决了I/O分发的高效率问题，那么work threads就解决了业务逻辑和I/O分发之间的耦合问题。把这两个策略组装在一起，就是实战中普遍采用的模式。大名鼎鼎的Netty，就是把这种模式发挥到极致的一种实现。不过要注意Netty里面提到的worker线程，其实就是我们这里说的从reactor线程，并不是处理具体业务逻辑的worker线程。</p><p>下面贴的一段代码就是常见的Netty初始化代码，这里Boss  Group就是acceptor主反应堆，workerGroup就是从反应堆。而处理业务逻辑的线程，通常都是通过使用Netty的程序开发者进行设计和定制，一般来说，业务逻辑线程需要从workerGroup线程中分离，以便支持更高的并发度。</p><pre><code>public final class TelnetServer {
    static final int PORT = Integer.parseInt(System.getProperty(&quot;port&quot;, SSL? &quot;8992&quot; : &quot;8023&quot;));

    public static void main(String[] args) throws Exception {
        //产生一个reactor线程，只负责accetpor的对应处理
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        //产生一个reactor线程，负责处理已连接套接字的I/O事件分发
        EventLoopGroup workerGroup = new NioEventLoopGroup(1);
        try {
           //标准的Netty初始，通过serverbootstrap完成线程池、channel以及对应的handler设置，注意这里讲bossGroup和workerGroup作为参数设置
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup)
             .channel(NioServerSocketChannel.class)
             .handler(new LoggingHandler(LogLevel.INFO))
             .childHandler(new TelnetServerInitializer(sslCtx));

            //开启两个reactor线程无限循环处理
            b.bind(PORT).sync().channel().closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}
</code></pre><p><img src="https://static001.geekbang.org/resource/image/1e/b4/1e647269a5f51497bd5488b2a44444b4.png?wh=3340*6055" alt=""><br>
这张图解释了主-从反应堆下加上worker线程池的处理模式。</p><p>主-从反应堆跟上面介绍的做法是一样的。和上面不一样的是，这里将decode、compute、encode等CPU密集型的工作从I/O线程中拿走，这些工作交给worker线程池来处理，而且这些工作拆分成了一个个子任务进行。encode之后完成的结果再由sub-reactor的I/O线程发送出去。</p><h2>样例程序</h2><pre><code>#include &lt;lib/acceptor.h&gt;
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
</code></pre><p>我们的样例程序几乎和第27讲的一样，唯一的不同是在创建TCPServer时，线程的数量设置不再是0，而是4。这里线程是4，说明是一个主acceptor线程，4个从reactor线程，每一个线程都跟一个event_loop一一绑定。</p><p>你可能会问，这么简单就完成了主、从线程的配置？</p><p>答案是YES。这其实是设计框架需要考虑的地方，一个框架不仅要考虑性能、扩展性，也需要考虑可用性。可用性部分就是程序开发者如何使用框架。如果我是一个开发者，我肯定关心框架的使用方式是不是足够方便，配置是不是足够灵活等。</p><p>像这里，可以根据需求灵活地配置主、从反应堆线程，就是一个易用性的体现。当然，因为时间有限，我没有考虑woker线程的部分，这部分其实应该是应用程序自己来设计考虑。网络编程框架通过回调函数暴露了交互的接口，这里应用程序开发者完全可以在onMessage方法里面获取一个子线程来处理encode、compute和encode的工作，像下面的示范代码一样。</p><pre><code>//数据读到buffer之后的callback
int onMessage(struct buffer *input, struct tcp_connection *tcpConnection) {
    printf(&quot;get message from tcp connection %s\n&quot;, tcpConnection-&gt;name);
    printf(&quot;%s&quot;, input-&gt;data);
    //取出一个线程来负责decode、compute和encode
    struct buffer *output = thread_handle(input);
    //处理完之后再通过reactor I/O线程发送数据
    tcp_connection_send_buffer(tcpConnection, output);
    return 
</code></pre><h2>样例程序结果</h2><p>我们启动这个服务器端程序，你可以从服务器端的输出上看到使用了poll作为事件分发方式。</p><p>多打开几个telnet客户端交互，main-thread只负责新的连接建立，每个客户端数据的收发由不同的子线程Thread-1、Thread-2、Thread-3和Thread-4来提供服务。</p><p>这里由于使用了子线程进行I/O处理，主线程可以专注于新连接处理，从而大大提高了客户端连接成功率。</p><pre><code>$./poll-server-multithreads
[msg] set poll as dispatcher
[msg] add channel fd == 4, main thread
[msg] poll added channel fd==4
[msg] set poll as dispatcher
[msg] add channel fd == 7, main thread
[msg] poll added channel fd==7
[msg] event loop thread init and signal, Thread-1
[msg] event loop run, Thread-1
[msg] event loop thread started, Thread-1
[msg] set poll as dispatcher
[msg] add channel fd == 9, main thread
[msg] poll added channel fd==9
[msg] event loop thread init and signal, Thread-2
[msg] event loop run, Thread-2
[msg] event loop thread started, Thread-2
[msg] set poll as dispatcher
[msg] add channel fd == 11, main thread
[msg] poll added channel fd==11
[msg] event loop thread init and signal, Thread-3
[msg] event loop thread started, Thread-3
[msg] set poll as dispatcher
[msg] event loop run, Thread-3
[msg] add channel fd == 13, main thread
[msg] poll added channel fd==13
[msg] event loop thread init and signal, Thread-4
[msg] event loop run, Thread-4
[msg] event loop thread started, Thread-4
[msg] add channel fd == 5, main thread
[msg] poll added channel fd==5
[msg] event loop run, main thread
[msg] get message channel i==1, fd==5
[msg] activate channel fd == 5, revents=2, main thread
[msg] new connection established, socket == 14
connection completed
[msg] get message channel i==0, fd==7
[msg] activate channel fd == 7, revents=2, Thread-1
[msg] wakeup, Thread-1
[msg] add channel fd == 14, Thread-1
[msg] poll added channel fd==14
[msg] get message channel i==1, fd==14
[msg] activate channel fd == 14, revents=2, Thread-1
get message from tcp connection connection-14
fasfas
[msg] get message channel i==1, fd==14
[msg] activate channel fd == 14, revents=2, Thread-1
get message from tcp connection connection-14
fasfas
asfa
[msg] get message channel i==1, fd==5
[msg] activate channel fd == 5, revents=2, main thread
[msg] new connection established, socket == 15
connection completed
[msg] get message channel i==0, fd==9
[msg] activate channel fd == 9, revents=2, Thread-2
[msg] wakeup, Thread-2
[msg] add channel fd == 15, Thread-2
[msg] poll added channel fd==15
[msg] get message channel i==1, fd==15
[msg] activate channel fd == 15, revents=2, Thread-2
get message from tcp connection connection-15
afasdfasf
[msg] get message channel i==1, fd==15
[msg] activate channel fd == 15, revents=2, Thread-2
get message from tcp connection connection-15
afasdfasf
safsafa
[msg] get message channel i==1, fd==15
[msg] activate channel fd == 15, revents=2, Thread-2
[msg] poll delete channel fd==15
connection closed
[msg] get message channel i==1, fd==5
[msg] activate channel fd == 5, revents=2, main thread
[msg] new connection established, socket == 16
connection completed
[msg] get message channel i==0, fd==11
[msg] activate channel fd == 11, revents=2, Thread-3
[msg] wakeup, Thread-3
[msg] add channel fd == 16, Thread-3
[msg] poll added channel fd==16
[msg] get message channel i==1, fd==16
[msg] activate channel fd == 16, revents=2, Thread-3
get message from tcp connection connection-16
fdasfasdf
[msg] get message channel i==1, fd==14
[msg] activate channel fd == 14, revents=2, Thread-1
[msg] poll delete channel fd==14
connection closed
[msg] get message channel i==1, fd==5
[msg] activate channel fd == 5, revents=2, main thread
[msg] new connection established, socket == 17
connection completed
[msg] get message channel i==0, fd==13
[msg] activate channel fd == 13, revents=2, Thread-4
[msg] wakeup, Thread-4
[msg] add channel fd == 17, Thread-4
[msg] poll added channel fd==17
[msg] get message channel i==1, fd==17
[msg] activate channel fd == 17, revents=2, Thread-4
get message from tcp connection connection-17
qreqwrq
[msg] get message channel i==1, fd==16
[msg] activate channel fd == 16, revents=2, Thread-3
[msg] poll delete channel fd==16
connection closed
[msg] get message channel i==1, fd==5
[msg] activate channel fd == 5, revents=2, main thread
[msg] new connection established, socket == 18
connection completed
[msg] get message channel i==0, fd==7
[msg] activate channel fd == 7, revents=2, Thread-1
[msg] wakeup, Thread-1
[msg] add channel fd == 18, Thread-1
[msg] poll added channel fd==18
[msg] get message channel i==1, fd==18
[msg] activate channel fd == 18, revents=2, Thread-1
get message from tcp connection connection-18
fasgasdg
^C
</code></pre><h2>总结</h2><p>本讲主要讲述了主从reactor模式，主从reactor模式中，主reactor只负责连接建立的处理，而把已连接套接字的I/O事件分发交给从reactor线程处理，这大大提高了客户端连接的处理能力。从Netty的实现上来看，也遵循了这一原则。</p><h2>思考题</h2><p>和往常一样，给你留两道思考题：</p><p>第一道，从日志输出中，你还可以看到main-thread首先加入了fd为4的套接字，这个是监听套接字，很好理解。可是这里的main-thread又加入了一个fd为7的套接字，这个套接字是干什么用的呢？</p><p>第二道，你可以试着修改一下服务器端的代码，把decode-compute-encode部分使用线程或者线程池来处理。</p><p>欢迎你在评论区写下你的思考，或者在GitHub上上传修改过的代码，我会和你一起交流，也欢迎把这篇文章分享给你的朋友或者同事，一起交流一下。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/67/f4/9a1feb59.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>钱</span>
  </div>
  <div class="_2_QraFYR_0">1：阻塞IO+多进程——实现简单，性能一般<br><br>2：阻塞IO+多线程——相比于阻塞IO+多进程，减少了上下文切换所带来的开销，性能有所提高。<br><br>3：阻塞IO+线程池——相比于阻塞IO+多线程，减少了线程频繁创建和销毁的开销，性能有了进一步的提高。<br><br>4：Reactor+线程池——相比于阻塞IO+线程池，采用了更加先进的事件驱动设计思想，资源占用少、效率高、扩展性强，是支持高性能高并发场景的利器。<br><br>5：主从Reactor+线程池——相比于Reactor+线程池，将连接建立事件和已建立连接的各种IO事件分离，主Reactor只负责处理连接事件，从Reactor只负责处理各种IO事件，这样能增加客户端连接的成功率，并且可以充分利用现在多CPU的资源特性进一步的提高IO事件的处理效率。<br><br><br>6：主 - 从Reactor模式的核心思想是，主Reactor线程只负责分发 Acceptor 连接建立，已连接套接字上的 I&#47;O 事件交给 从Reactor 负责分发。其中 sub-reactor 的数量，可以根据 CPU 的核数来灵活设置。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 总结的很到位，有点惊艳 😁</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-24 21:59:04</div>
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
  <div class="_2_QraFYR_0">老师您好，<br>如果在worker thread pool里面的thread在执行工作时，又遇到了I&#47;O。是不是也可以在worker thread pool里面加入epoll来轮询？但通常在worker thread里面遇到的I&#47;O应该都已经不是network I&#47;O了，而是sql、读写file、或是向第三方发起api，我不是很确定能否用epoll来处理。<br><br>有在google上查到，worker thread或worker process若遇到I&#47;O，似乎会用一种叫作coroutine的方式来切换cpu的使用权。此种切换方式，不涉及kernel，全是在应用程序做切换。<br><br>这边想请教老师，对在worker thread里面遇到I&#47;O问题时的处理方式或是心得是什么？<br><br>谢谢老师的分享！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 正如你所说，一般我们这里说的worker都是正经干苦力活的，如encode&#47;decode，业务逻辑等，在网络编程范式下，我们不推荐I&#47;O操作又混在worker线程里面。<br><br>而你提的routine的方式，应该是一种I&#47;O处理的编程方式，当我们使用这样routine的时候，如果有I&#47;O操作，对应的cpu资源被切换回去，实际上又回到了I&#47;O事件驱动的范式。这里的routine本身是被语言自己所封装的I&#47;O事件驱动机制所包装的，你可以认为在这种情况下，语言(如C++&#47;Golang)实现了内生的事件驱动机制，让我们可以直接关注之前的encode&#47;decode和业务逻辑的编码。<br><br>不管技术怎么变化，cpu、线程、事件驱动，这些概念和实现都是实实在在存在的，为了让我们写代码更加的简单和直接，将这些复杂的概念藏在后面，通过新的编程范式来达到这样的目的，是现代程序语言发展的必然。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-12 15:42:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/b8/c8/950fb2c9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>马不停蹄</span>
  </div>
  <div class="_2_QraFYR_0">学习 netty 的时候了解到 reactor 模式，netty 的 （单 、主从）reactor 可以灵活配置，老师讲的模式真的是和 netty 设计一样 ，这次学习算是真正搞明白了哈哈</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Java的封装是非常漂亮，倘若能理解原理，就会更加容易理解它的封装了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-12 17:35:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>刘系</span>
  </div>
  <div class="_2_QraFYR_0">老师，我试验了程序，发现有一个问题。<br>服务器程序启动后输出结果与文章中的不一样。<br> .&#47;poll-server-multithreads <br>[msg] set poll as dispatcher, main thread<br>[msg] add channel fd == 4, main thread<br>[msg] poll added channel fd==4, main thread<br>[msg] set poll as dispatcher, Thread-1<br>[msg] add channel fd == 8, Thread-1<br>[msg] poll added channel fd==8, Thread-1<br>[msg] event loop thread init and signal, Thread-1<br>[msg] event loop run, Thread-1<br>[msg] event loop thread started, Thread-1<br>[msg] set poll as dispatcher, Thread-2<br>[msg] add channel fd == 10, Thread-2<br>[msg] poll added channel fd==10, Thread-2<br>[msg] event loop thread init and signal, Thread-2<br>[msg] event loop run, Thread-2<br>[msg] event loop thread started, Thread-2<br>[msg] set poll as dispatcher, Thread-3<br>[msg] add channel fd == 19, Thread-3<br>[msg] poll added channel fd==19, Thread-3<br>[msg] event loop thread init and signal, Thread-3<br>[msg] event loop run, Thread-3<br>[msg] event loop thread started, Thread-3<br>[msg] set poll as dispatcher, Thread-4<br>[msg] add channel fd == 21, Thread-4<br>[msg] poll added channel fd==21, Thread-4<br>[msg] event loop thread init and signal, Thread-4<br>[msg] event loop run, Thread-4<br>[msg] event loop thread started, Thread-4<br>[msg] add channel fd == 6, main thread<br>[msg] poll added channel fd==6, main thread<br>[msg] event loop run, main thread<br>各个子线程启动后创建的套接字对是添加在子线程的eventloop上的，而不是像文章中的全是添加在主线程中。<br>从我阅读代码来看，确实也是添加在子线程中。不知道哪里不对？<br>主线程给子线程下发连接套接字是通过主线程调用event_loop_add_channel_event完成的，当主线程中发现eventloop和自己不是同一个线程，就通过给这个evenloop的套接字对发送一个“a”产生事件唤醒，然后子线程处理pending_channel，实现在子线程中添加连接套接字。<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我怎么觉的你的结果是对的呢？有可能我文章中贴的信息不够全，造成了一定的误导。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-17 22:52:50</div>
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
  <div class="_2_QraFYR_0">我觉得老师这里onMessage回调中使用线程池方式有误，这里解码，处理，编码是串行操作的，多线程并不能带来性能的提升，主线程还是会阻塞不释放的，我觉得最佳的做法是，解码交给线程池去做，然后返回，解码完成后注册进sub-reactor中再交由下一个业务处理，业务处理，编码同上，实现解耦充分利用多线程</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 非常同意，这里不是使用有误，只是作为一个例子，在线程里统一处理了解码、处理和编码。你的说法是对的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-03 19:08:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/cf/10/9fa2e5ba.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>进击的巨人</span>
  </div>
  <div class="_2_QraFYR_0">Netty的主从reactor分别对应bossGroup和workerGroup，workerGroup处理非accept的io事件，至于业务逻辑是否交给另外的线程池处理，可以理解为netty并没有支持，原因是因为业务逻辑都需要开发者自己自定义提供，但在这点上，netty通过ChannelHandler+pipline提供了io事件和业务逻辑分离的能力，需要开发者添加自定义ChannelHandler，实现io事件到业务逻辑处理的线程分离。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，netty确实是这样设计的，很多东西最后都是殊途同归。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-15 16:35:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/ea/3c/24cb4bde.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>疯狂的石头</span>
  </div>
  <div class="_2_QraFYR_0">看老师源码，channel，buffer各种对象，调来调去的，给我调懵了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 最后一个部分会讲这部分的设计，不要晕哈。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-06 21:11:22</div>
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
  <div class="_2_QraFYR_0">event_loop.c --- struct event_loop *event_loop_init_with_name(char *thread_name)：<br><br>#ifdef EPOLL_ENABLE<br>    yolanda_msgx(&quot;set epoll as dispatcher, %s&quot;, eventLoop-&gt;thread_name);<br>    eventLoop-&gt;eventDispatcher = &amp;epoll_dispatcher;<br>#else<br>    yolanda_msgx(&quot;set poll as dispatcher, %s&quot;, eventLoop-&gt;thread_name);<br>    eventLoop-&gt;eventDispatcher = &amp;poll_dispatcher;<br>#endif<br>    eventLoop-&gt;event_dispatcher_data = eventLoop-&gt;eventDispatcher-&gt;init(eventLoop);<br><br>没找到 EPOLL_ENABLE 的定义，老师怎么考虑的！！这里的话是否只能在event_loop.h 所包含的头文件中去找定义？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个是通过CMake来定义的，通过CMake的check来检验是否enable epoll，这个宏出现在动态生成的头文件中。<br><br># check epoll and add config.h for the macro compilation<br>include(CheckSymbolExists)<br>check_symbol_exists(epoll_create &quot;sys&#47;epoll.h&quot; EPOLL_EXISTS)<br>if (EPOLL_EXISTS)<br>    # Linux下设置为epoll<br>    set(EPOLL_ENABLE 1 CACHE INTERNAL &quot;enable epoll&quot;)<br><br>    # Linux下也设置为poll<br>#    set(EPOLL_ENABLE &quot;&quot; CACHE INTERNAL &quot;not enable epoll&quot;)<br>else ()<br>    set(EPOLL_ENABLE &quot;&quot; CACHE INTERNAL &quot;not enable epoll&quot;)<br>endif ()</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-06 19:07:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/97/b7/d5a83264.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>李朝辉</span>
  </div>
  <div class="_2_QraFYR_0">fd为7的套接字应该是socketpair()调用创建的主-从reactor套接字对中，从reactor线程写，主reactor线程读的套接字，作用的话，个人推测应该是从reactor线程中的连接套接字关闭了（即连接断开了），将这样的事件反馈给主reactor，以通知主reactor线程，我已经准备好接收下一个连接套接字？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 接近真相了，后续章节会揭开答案。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-12 21:54:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/97/b7/d5a83264.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>李朝辉</span>
  </div>
  <div class="_2_QraFYR_0">4核cpu，主reactor要占掉一个，只有3个可以分配给从核心。<br>按照老师的说法，是因为主reactor的工作相对比较简单，所以占用内核的时间很少，所以将从reactor分配满，然后最大化对连接套接字的处理能力吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我其实没想这么多，一般而言，worker线程的个数保持和cpu核一致，是一个比较常见的做法，例如nginx。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-12 21:11:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/4f/1e/e2b7a9ba.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>川云</span>
  </div>
  <div class="_2_QraFYR_0">可不可以把调用poll代码的位置展示一下</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 调用poll的代码已经封装在框架中，具体可以看<br>https:&#47;&#47;github.com&#47;froghui&#47;yolanda<br><br>lib&#47;event_dispatcher.h<br>lib&#47;poll_dispatcher.h<br>lib&#47;poll_dispatcher.c</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-11 22:08:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>王蓬勃</span>
  </div>
  <div class="_2_QraFYR_0">老师 请问那个event_loop_do_channel_event函数什么时候才进入不是同一个线程的判断中去？看不明白了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这段判断，判断的依据是当前的线程id是event_loop里的记录，是不是一致的。<br>if (!isInSameThread(eventLoop)) {<br>        event_loop_wakeup(eventLoop);<br>    } else {<br>        event_loop_handle_pending_channel(eventLoop);<br>    }</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-20 19:38:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/22/26/81/036e6579.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>这一行，30年</span>
  </div>
  <div class="_2_QraFYR_0"><br>   <br>#include &lt;lib&#47;acceptor.h&gt;<br>#include &quot;lib&#47;common.h&quot;<br>#include &quot;lib&#47;event_loop.h&quot;<br>#include &quot;lib&#47;tcp_server.h&quot;<br><br>把老师的代码copy过去，这些类库都报错，不用老师引用的宏用什么宏？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你是用什么方法编译的？我的工程统一用CMake编译，应该没问题。<br><br>另外，这里的库都在源代码的lib目录下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-09 14:54:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/dd/01/803f3750.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>企鹅</span>
  </div>
  <div class="_2_QraFYR_0">老师，主reactor只分发acceptor上建立连接的事件，不应该是client-&gt;acceptor -&gt; master reactor么，图上是client-&gt;master reactor-&gt;acceptor这里看晕了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: master reactor 反应堆线程，就是主acceptor，这两个意思接近，一个是从设计模式角度，另一个是从程序设计功能角度。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-17 21:30:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/90/e6/5eb07352.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Morton</span>
  </div>
  <div class="_2_QraFYR_0">老师，Reactor线程池占用了一部分cpu核，然后worker线程如果用线程池又会占用一部分cpu核，假设8核机器应该怎么分配线程池？reactor占4个worker线程占4个？<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你说的是一个分法，主要还是看你worker线程干活的多少，最好还是经过实际压测的结果来决定cpu分配比。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-25 09:08:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJUzv6S9wroyXaoFIwvC1mdDiav4BVS4BbPTuwtvWibthL5PyMuxFNicY06QJMZicVpib7E88S19nH4I9Q/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>木子皿</span>
  </div>
  <div class="_2_QraFYR_0">终于把整个代码流程走通了，太不容易了，不过还只是看得懂，写出来还是很难！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 先看懂，再试着改</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-19 14:28:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJUzv6S9wroyXaoFIwvC1mdDiav4BVS4BbPTuwtvWibthL5PyMuxFNicY06QJMZicVpib7E88S19nH4I9Q/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>木子皿</span>
  </div>
  <div class="_2_QraFYR_0">坚持坚持，无数次想放弃！快要结束了！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 坚持就是胜利 ✌️</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-19 09:45:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/17/68/1592a02d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我的名字不叫1988</span>
  </div>
  <div class="_2_QraFYR_0">老师，github上面的源码，lib&#47;poll_dispacher.c文件里面的poll_add、poll_del、poll_update等函数里面的“if (i &gt; INIT_POLL_SIZE)” 判断有问题，因为 for 循环结束之后，i 的可能的最大值为INIT_POLL_SIZE，所以永远不可能大于INIT_POLL_SIZE</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，我看到了，改起来~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-14 10:28:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/5e/3b/845fb641.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jhren</span>
  </div>
  <div class="_2_QraFYR_0">请问老师，我看见有人在发送端使用httpcomponents I&#47;O reactor，请问合理吗？<br><br>https:&#47;&#47;hc.apache.org&#47;httpcomponents-core-ga&#47;tutorial&#47;html&#47;nio.html#d5e477</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 合理啊，客户端也可以事件驱动。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-30 15:45:23</div>
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
  <div class="_2_QraFYR_0">老师，请问能不能说一下上一讲和这一讲的代码中的channel是干什么用的？一直没看明白</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: channel是一个抽象，表示一个连接通道，有可能是tcp连接，也有可能是内部的一个实现(如sockertpair)，你可以把它和connection做一个有效关联。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-25 19:09:48</div>
  </div>
</div>
</div>
</li>
</ul>