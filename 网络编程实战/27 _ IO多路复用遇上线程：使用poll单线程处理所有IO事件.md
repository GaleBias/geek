<audio title="27 _ IO多路复用遇上线程：使用poll单线程处理所有IO事件" src="https://static001.geekbang.org/resource/audio/45/01/4576a09c92b2d1d90e1b6373db513001.mp3" controls="controls"></audio> 
<p>你好，我是盛延敏，这里是网络编程实战第27讲，欢迎回来。</p><p>我在前面两讲里，分别使用了fork进程和pthread线程来处理多并发，这两种技术使用简单，但是性能却会随着并发数的上涨而快速下降，并不能满足极端高并发的需求。就像第24讲中讲到的一样，这个时候我们需要寻找更好的解决之道，这个解决之道基本的思想就是I/O事件分发。</p><p><span class="orange">关于代码，你可以去<a href="https://github.com/froghui/yolanda">GitHub</a>上查看或下载完整代码。</span></p><h2>重温事件驱动</h2><h3>基于事件的程序设计: GUI、Web</h3><p>事件驱动的好处是占用资源少，效率高，可扩展性强，是支持高性能高并发的不二之选。</p><p>如果你熟悉GUI编程的话，你就会知道，GUI设定了一系列的控件，如Button、Label、文本框等，当我们设计基于控件的程序时，一般都会给Button的点击安排一个函数，类似这样：</p><pre><code>//按钮点击的事件处理
void onButtonClick(){
  
}
</code></pre><p>这个设计的思想是，一个无限循环的事件分发线程在后台运行，一旦用户在界面上产生了某种操作，例如点击了某个Button，或者点击了某个文本框，一个事件会被产生并放置到事件队列中，这个事件会有一个类似前面的onButtonClick回调函数。事件分发线程的任务，就是为每个发生的事件找到对应的事件回调函数并执行它。这样，一个基于事件驱动的GUI程序就可以完美地工作了。</p><!-- [[[read_end]]] --><p>还有一个类似的例子是Web编程领域。同样的，Web程序会在Web界面上放置各种界面元素，例如Label、文本框、按钮等，和GUI程序类似，给感兴趣的界面元素设计JavaScript回调函数，当用户操作时，对应的JavaScript回调函数会被执行，完成某个计算或操作。这样，一个基于事件驱动的Web程序就可以在浏览器中完美地工作了。</p><p>在第24讲中，我们已经提到，通过使用poll、epoll等I/O分发技术，可以设计出基于套接字的事件驱动程序，从而满足高性能、高并发的需求。</p><p>事件驱动模型，也被叫做反应堆模型（reactor），或者是Event loop模型。这个模型的核心有两点。</p><p>第一，它存在一个无限循环的事件分发线程，或者叫做reactor线程、Event loop线程。这个事件分发线程的背后，就是poll、epoll等I/O分发技术的使用。</p><p>第二，所有的I/O操作都可以抽象成事件，每个事件必须有回调函数来处理。acceptor上有连接建立成功、已连接套接字上发送缓冲区空出可以写、通信管道pipe上有数据可以读，这些都是一个个事件，通过事件分发，这些事件都可以一一被检测，并调用对应的回调函数加以处理。</p><h2>几种I/O模型和线程模型设计</h2><p>任何一个网络程序，所做的事情可以总结成下面几种：</p><ul>
<li>read：从套接字收取数据；</li>
<li>decode：对收到的数据进行解析；</li>
<li>compute：根据解析之后的内容，进行计算和处理；</li>
<li>encode：将处理之后的结果，按照约定的格式进行编码；</li>
<li>send：最后，通过套接字把结果发送出去。</li>
</ul><p>这几个过程和套接字最相关的是read和send这两种。接下来，我们总结一下已经学过的几种支持多并发的网络编程技术，引出我们今天的话题，使用poll单线程处理所有I/O。</p><h3>fork</h3><p>第25讲中，我们使用fork来创建子进程，为每个到达的客户连接服务。这张图很好地解释了这个设计模式，可想而知的是，随着客户数的变多，fork的子进程也越来越多，即使客户和服务器之间的交互比较少，这样的子进程也不能被销毁，一直需要存在。使用fork的方式处理非常简单，它的缺点是处理效率不高，fork子进程的开销太大。</p><p><img src="https://static001.geekbang.org/resource/image/f1/1c/f1045858bc79c5064903c25c6388051c.png?wh=1052*748" alt=""></p><h3>pthread</h3><p>第26讲中，我们使用了pthread_create创建子线程，因为线程是比进程更轻量级的执行单位，所以它的效率相比fork的方式，有一定的提高。但是，每次创建一个线程的开销仍然是不小的，因此，引入了线程池的概念，预先创建出一个线程池，在每次新连接达到时，从线程池挑选出一个线程为之服务，很好地解决了线程创建的开销。但是，这个模式还是没有解决空闲连接占用资源的问题，如果一个连接在一定时间内没有数据交互，这个连接还是要占用一定的线程资源，直到这个连接消亡为止。</p><p><img src="https://static001.geekbang.org/resource/image/1c/2c/1c07131ab6ca03d3a5a9092ef20e0b2c.png?wh=1006*708" alt=""></p><h3>single reactor thread</h3><p>前面讲到，事件驱动模式是解决高性能、高并发比较好的一种模式。为什么呢？</p><p>因为这种模式是符合大规模生产的需求的。我们的生活中遍地都是类似的模式。比如你去咖啡店喝咖啡，你点了一杯咖啡在一旁喝着，服务员也不会管你，等你有续杯需求的时候，再去和服务员提（触发事件），服务员满足了你的需求，你就继续可以喝着咖啡玩手机。整个柜台的服务方式就是一个事件驱动的方式。</p><p>这里有一张图，解释了这一讲的设计模式。一个reactor线程上同时负责分发acceptor的事件、已连接套接字的I/O事件。</p><p><img src="https://static001.geekbang.org/resource/image/b8/33/b8627a1a1d32da4b55ac74d4f0230f33.png?wh=1006*616" alt=""></p><h3>single reactor thread + worker threads</h3><p>但是上述的设计模式有一个问题，和I/O事件处理相比，应用程序的业务逻辑处理是比较耗时的，比如XML文件的解析、数据库记录的查找、文件资料的读取和传输、计算型工作的处理等，这些工作相对而言比较独立，它们会拖慢整个反应堆模式的执行效率。</p><p>所以，将这些decode、compute、enode型工作放置到另外的线程池中，和反应堆线程解耦，是一个比较明智的选择。反应堆线程只负责处理I/O相关的工作，业务逻辑相关的工作都被裁剪成一个一个的小任务，放到线程池里由空闲的线程来执行。当结果完成后，再交给反应堆线程，由反应堆线程通过套接字将结果发送出去。</p><p><img src="https://static001.geekbang.org/resource/image/7e/23/7e4505bb75fef4a4bb945e6dc3040823.png?wh=988*842" alt=""></p><h2>样例程序</h2><p>从今天开始，我们会接触到为本课程量身定制的网络编程框架。使用这个网络编程框架的样例程序如下：</p><pre><code>#include &lt;lib/acceptor.h&gt;
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

    //初始tcp_server，可以指定线程数目，如果线程是0，就只有一个线程，既负责acceptor，也负责I/O
    struct TCPserver *tcpServer = tcp_server_init(eventLoop, acceptor, onConnectionCompleted, onMessage,
                                                  onWriteCompleted, onConnectionClosed, 0);
    tcp_server_start(tcpServer);

    // main thread for acceptor
    event_loop_run(eventLoop);
}
</code></pre><p>这个程序的main函数部分只有几行, 因为是第一次接触到，稍微展开介绍一下。</p><p>第49行创建了一个event_loop，即reactor对象，这个event_loop和线程相关联，每个event_loop在线程里执行的是一个无限循环，以便完成事件的分发。</p><p>第52行初始化了acceptor，用来监听在某个端口上。</p><p>第55行创建了一个TCPServer，创建的时候可以指定线程数目，这里线程是0，就只有一个线程，既负责acceptor的连接处理，也负责已连接套接字的I/O处理。这里比较重要的是传入了几个回调函数，分别对应了连接建立完成、数据读取完成、数据发送完成、连接关闭完成几种操作，通过回调函数，让业务程序可以聚焦在业务层开发。</p><p>第57行开启监听。</p><p>第60行运行event_loop无限循环，等待acceptor上有连接建立、新连接上有数据可读等。</p><h2>样例程序结果</h2><p>运行这个服务器程序，开启两个telnet客户端，我们看到服务器端的输出如下：</p><pre><code> $./poll-server-onethread
[msg] set poll as dispatcher
[msg] add channel fd == 4, main thread
[msg] poll added channel fd==4
[msg] add channel fd == 5, main thread
[msg] poll added channel fd==5
[msg] event loop run, main thread
[msg] get message channel i==1, fd==5
[msg] activate channel fd == 5, revents=2, main thread
[msg] new connection established, socket == 6
connection completed
[msg] add channel fd == 6, main thread
[msg] poll added channel fd==6
[msg] get message channel i==2, fd==6
[msg] activate channel fd == 6, revents=2, main thread
get message from tcp connection connection-6
afadsfaf
[msg] get message channel i==2, fd==6
[msg] activate channel fd == 6, revents=2, main thread
get message from tcp connection connection-6
afadsfaf
fdafasf
[msg] get message channel i==1, fd==5
[msg] activate channel fd == 5, revents=2, main thread
[msg] new connection established, socket == 7
connection completed
[msg] add channel fd == 7, main thread
[msg] poll added channel fd==7
[msg] get message channel i==3, fd==7
[msg] activate channel fd == 7, revents=2, main thread
get message from tcp connection connection-7
sfasggwqe
[msg] get message channel i==3, fd==7
[msg] activate channel fd == 7, revents=2, main thread
[msg] poll delete channel fd==7
connection closed
[msg] get message channel i==2, fd==6
[msg] activate channel fd == 6, revents=2, main thread
[msg] poll delete channel fd==6
connection closed
</code></pre><p>这里自始至终都只有一个main thread在工作，可见，单线程的reactor处理多个连接时也可以表现良好。</p><h2>总结</h2><p>这一讲我们总结了几种不同的I/O模型和线程模型设计，并比较了各自不同的优缺点。从这一讲开始，我们将使用自己编写的编程框架来完成业务开发，这一讲使用了poll来处理所有的I/O事件，在下一讲里，我们将会看到如何把acceptor的连接事件和已连接套接字的I/O事件交由不同的线程处理，而这个分离，不过是在应用程序层简单的参数配置而已。</p><h2>思考题</h2><p>和往常一样，给你留两道思考题：</p><ol>
<li>你可以试着修改一下onMessage方法，把它变为期中作业中提到的cd、ls等command实现。</li>
<li>文章里服务器端的decode-compute-encode是在哪里实现的？你有什么办法来解决业务逻辑和I/O逻辑混在一起么？</li>
</ol><p>欢迎你在评论区写下你的思考，或者在GitHub上上传你的代码，也欢迎把这篇文章分享给你的朋友或者同事，一起交流一下。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/9d/ab/6589d91a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>林林</span>
  </div>
  <div class="_2_QraFYR_0">worker thread 和 reactor thread之间怎么进行数据传递？是要利用队列+锁吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这就是两个普通的producer-consumer线程关系，使用队列+锁的方式是一个比较常见的实现方式。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-26 09:03:35</div>
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
  <div class="_2_QraFYR_0">1：事件驱动模型的设计思想是啥？<br>事件驱动模型的设计的思想是，一个无限循环的事件分发线程在后台运行，一旦做了某种操作触发了一个事件，这个事件就会被放置到事件队列中，事件分发线程的任务，就为这个发生的事件找到对应的事件回调函数并执行它。<br>这里有个疑问，事件分发线程怎么找到事件的回调函数，并调用它的？<br><br>2：事件驱动模型的优势是啥？<br>事件驱动的好处是占用资源少，效率高，可扩展性强，是支持高性能高并发的不二之选。<br>老师好，请问占用资源少这个结论是怎么得出来的？<br><br>3：IO网络通信是怎么实现事件驱动模型的？<br>通过使用 poll、epoll 等 I&#47;O 分发技术，可以设计出基于套接字的事件驱动程序，从而满足高性能、高并发的需求。<br><br>4：Reactor模型是啥玩意？<br>Reactor模型（中文叫做反应堆模型）也就是事件驱动模型或者是 Event loop 模型。<br>这个模型的核心有两点。<br>第一，它存在一个无限循环的事件分发线程，或者叫做 reactor 线程、Event loop 线程。这个事件分发线程的背后，就是 poll、epoll 等 I&#47;O 分发技术的使用。<br>第二，所有的 I&#47;O 操作都可以抽象成事件，每个事件必须有回调函数来处理。acceptor 上有连接建立成功、已连接套接字上发送缓冲区空出可以写、通信管道 pipe 上有数据可以读，这些都是一个个事件，通过事件分发，这些事件都可以一一被检测，并调用对应的回调函数加以处理。<br>5：Reactor模型——解决了空闲连接占用资源的问题，Reactor线程只负责处理 I&#47;O 相关的工作，业务逻辑相关的工作都被裁剪成一个一个的小任务，放到线程池里由空闲的线程来执行。当结果完成后，再交给反应堆线程，由Reactor线程通过套接字将结果发送出去。<br>所以，这个模式性能更优。<br><br>6：阻塞IO+多进程——实现简单，性能一般<br>7：阻塞IO+多线程——相比于阻塞IO+多进程，减少了上下文切换所带来的开销，性能有所提高。<br>8：阻塞IO+线程池——相比于阻塞IO+多线程，减少了线程频繁创建和销毁的开销，性能有了进一步的提高。<br>9：Reactor+线程池——相比于阻塞IO+线程池，采用了更加先进的事件驱动设计思想，资源占用少、效率高、扩展性强，是支持高性能高并发场景的利器。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 第一个问题，回调函数和套接字对应的，通过套接字找到对应的回调函数；<br><br>第二个问题，因为是事件驱动，不需要分配固定的资源，仅仅使用几个线程就可以支持上万的连接，每个线程的利用率得到了最大提升。<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-24 20:24:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>fxzhang</span>
  </div>
  <div class="_2_QraFYR_0">老师可否讲解linux下如何开发的，最近想换工作，但是之前都在windows下面开发，想自学一下linux下是如何开发的，但是有一种找不到开头不知道该怎么学习的感觉，很无力</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 先学习一下Linux下的安装、配置、管理，把工作环境放到Linux下面，让Linux成为你的工作效率机器；<br><br>其次，慢慢学习Bash，感受一下Linux的能力；<br><br>接下来就是学习一些 Linux下的程序设计，如I&#47;O、网络等。<br><br>如果你能把这篇系列的所有代码都改一遍，运行一遍，就是一个良好的开头。<br><br>加油~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-09 21:55:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/1b/4d/2cc44d9a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>刘忽悠</span>
  </div>
  <div class="_2_QraFYR_0">没太理解epoll反应堆模型，和直接eopll的区别是什么？<br>不知道我这么理解对不对，一般使用epoll，假如新连接建立，注册cfd读事件，当事件触发，接着在主线程里面读出来，然后处理，接着发送；epoll反应堆模式是不是仅仅只是在注册事件的时候加了一个对应的回调函数，当事件触发，然后调用回调去处理？相当于统一了一下接口？<br>对于老师说的reactor+threadpool不知道理解的对不对，我个人理解是，当有新连接建立，因为监听描述符注册的是acceptor事件，这时候这个事件触发，触发之后注册新的描述符cfd的read事件，当cfd的读事件触发，这时候在reactor线程（主线程）里面调当初注册的回调函数来处理读事件，读出来之后，然后注销cfd的读事件，这时候把读出来的内容封装成Task，放到线程池的Task队列，通知线程池的工作线程——有任务了，唤醒一个线程对任务进行处理，处理完成之后，这时候注册cfd的写事件，然后work线程处理完成，一般情况下写缓冲区都是可以写，所以这时候在reactor线程里面，cfd的写事件被触发，这时候在reactor线程里面调用对应的回调函数把数据发送过去，接着然后注销写事件，注册读事件，继续监听客户端请求。这样一来，业务逻辑都在线程池里面去做，然后读，写都是通过在主线程，也就是reactor线程里面调用对应的回调函数来完成。<br>不知道这么理解reactor模式+线程池对不对？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 基本是正确的。有一个小小的地方和我的理解不一致，就是对socket的读写，也是可以放到线程池里独立的线程去做，而主reactor线程就是一个事件分发器，不负责I&#47;O操作，因为主reactor线程是一个非常重要的&quot;大脑&quot;，尽量不要让它成为瓶颈。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-01 05:28:13</div>
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
  <div class="_2_QraFYR_0">第一遍看完这篇文章 我就感受颇深 尤其是事件触发 这个模式 然后就想到工作当中的用到的skynet框架底层就是采用事件驱动,某个服务有数据达到 就去触发对应的服务,然后再回想工作当中很多逻辑都抽象成事件,通过一个主循环检测时间然后来触发对应的事件！<br>更重要的一点,专栏下的代码我全部是自己手动实现了一遍 还用上了gdb很开心 很满足<br>第27讲和第24讲 应该重点学习,这两讲都是很重要的理论基础</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你是问题最多的，我想也是收获最大的。<br><br>调试、调试、调试，重要的问题讲三遍 :)</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-15 14:50:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/b3/d6/6101dbd7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>CPP</span>
  </div>
  <div class="_2_QraFYR_0">C语言要是没两把刷子，买了也是浪费钱。再调试也得建立再看懂的基础上......</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: C语言应该是大学期间必修课程吧.......</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-10 20:56:30</div>
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
  <div class="_2_QraFYR_0">这篇文章，多了很多生动的图片，感觉干货满满啊，哈哈，希望后面的课程，也能多加点对应的图片就更好了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我想应该是可以满足你的要求的:)</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-24 20:06:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/e9/86/d34800a4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>heyman</span>
  </div>
  <div class="_2_QraFYR_0"> reactor 线程无限循环，有点像轮询，效率不会很低吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不会。这里不是轮询哦，轮询是消耗cpu时间的，这里是系统提供的事件驱动，看似在无限循环，其实这个时候cpu被调度干其他事了，当真正有事件发生，cpu又会被切换回来，所以效率很高。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-18 11:18:42</div>
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
  <div class="_2_QraFYR_0">思考题第一道：https:&#47;&#47;github.com&#47;YoungYo&#47;yolanda&#47;blob&#47;master&#47;chap-27&#47;poll-server-onethread-homework.c 这是修改后的代码<br>思考题第二道： onMessage 方法就是处理 decode-compute-encode 逻辑的吧？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-25 18:58:58</div>
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
  <div class="_2_QraFYR_0">使用多线程有线程池，使用多进程有进程池吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 没有。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-19 15:37:00</div>
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
  <div class="_2_QraFYR_0">我看到别人的代码用到了老师说的这个思想，在接收消息它采用的是分发订阅模式   通过订阅者的回调来接收消息  没有特定的recv接口露给外界  这种设计思想老师怎么看</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这是框架设计的基本思想，我在后面的框架设计中也是这样的，欢迎继续阅读，你会有更多收获。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-22 15:10:17</div>
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
  <div class="_2_QraFYR_0">第二问根据工作中遇到的情况来看decode-compute-encode应该是业务逻辑,应该是工作线程上处理,觉得处理方式是每个描述符和每个线程做一个映射,并且注册一个回调函数,用这个函数来处理套接字收到消息事件,然后再执行decode-compute-encode逻辑</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-15 15:38:29</div>
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
  <div class="_2_QraFYR_0">之前由于忙着买房子 落下了很多课程 现在都在追,不过不管是追还是慢慢跟 我都会再好好复习的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 加油~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-15 14:51:51</div>
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
  <div class="_2_QraFYR_0">这个线程数为零的时候，threadPool动态申请了一个线程，但实际并没有使用，也没有释放是不是会造成资源泄露？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-26 20:14:05</div>
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
  <div class="_2_QraFYR_0">Synchronous Event Demultiplexer（同步事件分离器）  是这个吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个名词，看起来很熟系......</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-31 13:53:28</div>
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
  <div class="_2_QraFYR_0">不会。这里不是轮询哦，轮询是消耗cpu时间的，这里是系统提供的事件驱动，看似在无限循环，其实这个时候cpu被调度干其他事了。<br>老师，这里系统提供的事件驱动是指什么？既然不是无限轮训，那是怎么样执行呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 说的通俗一点，就是操作系统帮我们把这个事情给干了，操作系统是个多面手，可以干干这个，也干干那个，等到有事件发生的时候，就会腾出手来帮我们处理网络I&#47;O事件了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-31 13:52:27</div>
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
  <div class="_2_QraFYR_0">有一个问题，有一个无线轮询的线程会对CPU的消耗有影响吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 当然，你可以试试哦。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-30 17:44:46</div>
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
  <div class="_2_QraFYR_0">老师你好 这边开始有点晕了。代码里贴的event_loop反应堆就是基于系统轮询（如slect&#47;poll&#47;epoll）+非阻塞封装的吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 基于I&#47;O多路复用+非阻塞。一般的，都把select&#47;poll&#47;epoll叫做I&#47;O多路复用。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-22 17:46:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/25/2d/37/f8733b67.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>S</span>
  </div>
  <div class="_2_QraFYR_0">请问老师，你程序第一句打印add channel fd == 4, main thread，为什么第一个fd是4而不是3？ 0，1，2分别代表输入，输出，错误。那么fd:3被谁占用了？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 好问题。<br><br>是这样的，fd:3其实是被socketpair占了，也就是说fd:4是主reactor占用的描述字，fd:3是这个主reactor对应的socketpair，这是为了让反应堆线程能够随时感知新的事件。<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-18 17:06:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/48/7c/2aaf50e5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>coder</span>
  </div>
  <div class="_2_QraFYR_0">netty也是用的reactor模型</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-08 11:06:31</div>
  </div>
</div>
</div>
</li>
</ul>