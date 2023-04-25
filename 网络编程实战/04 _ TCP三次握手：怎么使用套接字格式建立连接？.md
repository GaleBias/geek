<audio title="04 _ TCP三次握手：怎么使用套接字格式建立连接？" src="https://static001.geekbang.org/resource/audio/f2/3e/f2354a6313ce726e7f3b86cf1428ca3e.mp3" controls="controls"></audio> 
<p>你好，我是盛延敏，这里是网络编程实战第4讲，欢迎回来。</p><p>在上一讲里我们介绍了IPv4、IPv6以及本地套接字格式，这一讲我们来讲一讲怎么使用这些套接字格式完成连接的建立，当然，经典的TCP三次握手理论也会贯穿其中。我希望经过这一讲的讲解，你会牢牢记住TCP三次握手和客户端、服务器模型。</p><p>让我们先从服务器端开始。</p><h2>服务端准备连接的过程</h2><h3>创建套接字</h3><p>要创建一个可用的套接字，需要使用下面的函数：</p><pre><code>int socket(int domain, int type, int protocol)
</code></pre><p>domain就是指PF_INET、PF_INET6以及PF_LOCAL等，表示什么样的套接字。</p><p>type可用的值是：</p><ul>
<li><strong>SOCK_STREAM: 表示的是字节流，对应TCP；</strong></li>
<li><strong>SOCK_DGRAM： 表示的是数据报，对应UDP；</strong></li>
<li><strong>SOCK_RAW: 表示的是原始套接字。</strong></li>
</ul><p>参数protocol原本是用来指定通信协议的，但现在基本废弃。因为协议已经通过前面两个参数指定完成。protocol目前一般写成0即可。</p><h3>bind: 设定电话号码</h3><p>创建出来的套接字如果需要被别人使用，就需要调用bind函数把套接字和套接字地址绑定，就像去电信局登记我们的电话号码一样。</p><p>调用bind函数的方式如下：</p><pre><code>bind(int fd, sockaddr * addr, socklen_t len)
</code></pre><p>我们需要注意到bind函数后面的第二个参数是通用地址格式<code>sockaddr * addr</code>。这里有一个地方值得注意，那就是虽然接收的是通用地址格式，实际上传入的参数可能是IPv4、IPv6或者本地套接字格式。bind函数会根据len字段判断传入的参数addr该怎么解析，len字段表示的就是传入的地址长度，它是一个可变值。</p><!-- [[[read_end]]] --><p>这里其实可以把bind函数理解成这样：</p><pre><code>bind(int fd, void * addr, socklen_t len)
</code></pre><p>不过BSD设计套接字的时候大约是1982年，那个时候的C语言还没有<code>void *</code>的支持，为了解决这个问题，BSD的设计者们创造性地设计了通用地址格式来作为支持bind和accept等这些函数的参数。</p><p>对于使用者来说，每次需要将IPv4、IPv6或者本地套接字格式转化为通用套接字格式，就像下面的IPv4套接字地址格式的例子一样：</p><pre><code>struct sockaddr_in name;
bind (sock, (struct sockaddr *) &amp;name, sizeof (name)
</code></pre><p>对于实现者来说，可根据该地址结构的前两个字节判断出是哪种地址。为了处理长度可变的结构，需要读取函数里的第三个参数，也就是len字段，这样就可以对地址进行解析和判断了。</p><p>设置bind的时候，对地址和端口可以有多种处理方式。</p><p>我们可以把地址设置成本机的IP地址，这相当告诉操作系统内核，仅仅对目标IP是本机IP地址的IP包进行处理。但是这样写的程序在部署时有一个问题，我们编写应用程序时并不清楚自己的应用程序将会被部署到哪台机器上。这个时候，可以利用<strong>通配地址</strong>的能力帮助我们解决这个问题。通配地址相当于告诉操作系统内核：“Hi，我可不挑活，只要目标地址是咱们的都可以。”比如一台机器有两块网卡，IP地址分别是202.61.22.55和192.168.1.11，那么向这两个IP请求的请求包都会被我们编写的应用程序处理。</p><p>那么该如何设置通配地址呢？</p><p>对于IPv4的地址来说，使用INADDR_ANY来完成通配地址的设置；对于IPv6的地址来说，使用IN6ADDR_ANY来完成通配地址的设置。</p><pre><code>struct sockaddr_in name;
name.sin_addr.s_addr = htonl (INADDR_ANY); /* IPV4通配地址 */
</code></pre><p>除了地址，还有端口。如果把端口设置成0，就相当于把端口的选择权交给操作系统内核来处理，操作系统内核会根据一定的算法选择一个空闲的端口，完成套接字的绑定。这在服务器端不常使用。</p><p>一般来说，服务器端的程序一定要绑定到一个众所周知的端口上。服务器端的IP地址和端口数据，相当于打电话拨号时需要知道的对方号码，如果没有电话号码，就没有办法和对方建立连接。</p><p>我们来看一个初始化IPv4 TCP 套接字的例子:</p><pre><code>#include &lt;stdio.h&gt;
#include &lt;stdlib.h&gt;
#include &lt;sys/socket.h&gt;
#include &lt;netinet/in.h&gt;


int make_socket (uint16_t port)
{
  int sock;
  struct sockaddr_in name;


  /* 创建字节流类型的IPV4 socket. */
  sock = socket (PF_INET, SOCK_STREAM, 0);
  if (sock &lt; 0)
    {
      perror (&quot;socket&quot;);
      exit (EXIT_FAILURE);
    }


  /* 绑定到port和ip. */
  name.sin_family = AF_INET; /* IPV4 */
  name.sin_port = htons (port);  /* 指定端口 */
  name.sin_addr.s_addr = htonl (INADDR_ANY); /* 通配地址 */
  /* 把IPV4地址转换成通用地址格式，同时传递长度 */
  if (bind (sock, (struct sockaddr *) &amp;name, sizeof (name)) &lt; 0)
    {
      perror (&quot;bind&quot;);
      exit (EXIT_FAILURE);
    }


  return sock;
}
</code></pre><h3>listen：接上电话线，一切准备就绪</h3><p>bind函数只是让我们的套接字和地址关联，如同登记了电话号码。如果要让别人打通电话，还需要我们把电话设备接入电话线，让服务器真正处于可接听的状态，这个过程需要依赖listen函数。</p><p>初始化创建的套接字，可以认为是一个"主动"套接字，其目的是之后主动发起请求（通过调用connect函数，后面会讲到）。通过listen函数，可以将原来的"主动"套接字转换为"被动"套接字，告诉操作系统内核：“我这个套接字是用来等待用户请求的。”当然，操作系统内核会为此做好接收用户请求的一切准备，比如完成连接队列。</p><p>listen函数的原型是这样的：</p><pre><code>int listen (int socketfd, int backlog)
</code></pre><p>我来稍微解释一下。第一个参数socketdf为套接字描述符，第二个参数backlog，在Linux中表示已完成(ESTABLISHED)且未accept的队列大小，这个参数的大小决定了可以接收的并发数目。这个参数越大，并发数目理论上也会越大。但是参数过大也会占用过多的系统资源，一些系统，比如Linux并不允许对这个参数进行改变。对于backlog整个参数的设置有一些最佳实践，这里就不展开，后面结合具体的实例进行解读。</p><h3>accept: 电话铃响起了……</h3><p>当客户端的连接请求到达时，服务器端应答成功，连接建立，这个时候操作系统内核需要把这个事件通知到应用程序，并让应用程序感知到这个连接。这个过程，就好比电信运营商完成了一次电话连接的建立, 应答方的电话铃声响起，通知有人拨打了号码，这个时候就需要拿起电话筒开始应答。</p><p>连接建立之后，你可以把accept这个函数看成是操作系统内核和应用程序之间的桥梁。它的原型是：</p><pre><code>int accept(int listensockfd, struct sockaddr *cliaddr, socklen_t *addrlen)
</code></pre><p>函数的第一个参数listensockfd是套接字，可以叫它为listen套接字，因为这就是前面通过bind，listen一系列操作而得到的套接字。函数的返回值有两个部分，第一个部分cliadd是通过指针方式获取的客户端的地址，addrlen告诉我们地址的大小，这可以理解成当我们拿起电话机时，看到了来电显示，知道了对方的号码；另一个部分是函数的返回值，这个返回值是一个全新的描述字，代表了与客户端的连接。</p><p>这里一定要注意有两个套接字描述字，第一个是监听套接字描述字listensockfd，它是作为输入参数存在的；第二个是返回的已连接套接字描述字。</p><p>你可能会问，为什么要把两个套接字分开呢？用一个不是挺好的么？</p><p>这里和打电话的情形非常不一样的地方就在于，打电话一旦有一个连接建立，别人是不能再打进来的，只会得到语音播报：“您拨的电话正在通话中。”而网络程序的一个重要特征就是并发处理，不可能一个应用程序运行之后只能服务一个客户，如果是这样， 双11抢购得需要多少服务器才能满足全国 “剁手党 ” 的需求？</p><p>所以监听套接字一直都存在，它是要为成千上万的客户来服务的，直到这个监听套接字关闭；而一旦一个客户和服务器连接成功，完成了TCP三次握手，操作系统内核就为这个客户生成一个已连接套接字，让应用服务器使用这个<strong>已连接套接字</strong>和客户进行通信处理。如果应用服务器完成了对这个客户的服务，比如一次网购下单，一次付款成功，那么关闭的就是<strong>已连接套接字</strong>，这样就完成了TCP连接的释放。请注意，这个时候释放的只是这一个客户连接，其它被服务的客户连接可能还存在。最重要的是，监听套接字一直都处于“监听”状态，等待新的客户请求到达并服务。</p><h2>客户端发起连接的过程</h2><p>前面讲述的bind、listen以及accept的过程，是典型的服务器端的过程。下面我来讲下客户端发起连接请求的过程。</p><p>第一步还是和服务端一样，要建立一个套接字，方法和前面是一样的。</p><p>不一样的是客户端需要调用connect向服务端发起请求。</p><h3>connect: 拨打电话</h3><p>客户端和服务器端的连接建立，是通过connect函数完成的。这是connect的构建函数：</p><pre><code>int connect(int sockfd, const struct sockaddr *servaddr, socklen_t addrlen)
</code></pre><p>函数的第一个参数sockfd是连接套接字，通过前面讲述的socket函数创建。第二个、第三个参数servaddr和addrlen分别代表指向套接字地址结构的指针和该结构的大小。套接字地址结构必须含有服务器的IP地址和端口号。</p><p>客户在调用函数connect前不必非得调用bind函数，因为如果需要的话，内核会确定源IP地址，并按照一定的算法选择一个临时端口作为源端口。</p><p>如果是TCP套接字，那么调用connect函数将激发TCP的三次握手过程，而且仅在连接建立成功或出错时才返回。其中出错返回可能有以下几种情况：</p><ol>
<li>三次握手无法建立，客户端发出的SYN包没有任何响应，于是返回TIMEOUT错误。这种情况比较常见的原因是对应的服务端IP写错。</li>
<li>客户端收到了RST（复位）回答，这时候客户端会立即返回CONNECTION REFUSED错误。这种情况比较常见于客户端发送连接请求时的请求端口写错，因为RST是TCP在发生错误时发送的一种TCP分节。产生RST的三个条件是：目的地为某端口的SYN到达，然而该端口上没有正在监听的服务器（如前所述）；TCP想取消一个已有连接；TCP接收到一个根本不存在的连接上的分节。</li>
<li>客户发出的SYN包在网络上引起了"destination unreachable"，即目的不可达的错误。这种情况比较常见的原因是客户端和服务器端路由不通。</li>
</ol><p>根据不同的返回值，我们可以做进一步的排查。</p><h2>著名的TCP三次握手: 这一次不用背记</h2><p><img src="https://static001.geekbang.org/resource/image/65/29/65cef2c44480910871a0b66cac1d5529.png?wh=952*540" alt=""><br>
你在各个场合都会了解到著名的TCP三次握手，可能还会被要求背下三次握手整个过程，但背后的原理和过程可能未必真正理解。我们刚刚学习了服务端和客户端连接的主要函数，下面结合这些函数讲解一下TCP三次握手的过程。这样我相信你不用背，也能根据理解轻松掌握这部分的知识。</p><p>这里我们使用的网络编程模型都是阻塞式的。所谓阻塞式，就是调用发起后不会直接返回，由操作系统内核处理之后才会返回。 相对的，还有一种叫做非阻塞式的，我们在后面的章节里会讲到。</p><h2>TCP三次握手的解读</h2><p>我们先看一下最初的过程，服务器端通过socket，bind和listen完成了被动套接字的准备工作，被动的意思就是等着别人来连接，然后调用accept，就会阻塞在这里，等待客户端的连接来临；客户端通过调用socket和connect函数之后，也会阻塞。接下来的事情是由操作系统内核完成的，更具体一点的说，是操作系统内核网络协议栈在工作。</p><p>下面是具体的过程：</p><ol>
<li>客户端的协议栈向服务器端发送了SYN包，并告诉服务器端当前发送序列号j，客户端进入SYNC_SENT状态；</li>
<li>服务器端的协议栈收到这个包之后，和客户端进行ACK应答，应答的值为j+1，表示对SYN包j的确认，同时服务器也发送一个SYN包，告诉客户端当前我的发送序列号为k，服务器端进入SYNC_RCVD状态；</li>
<li>客户端协议栈收到ACK之后，使得应用程序从connect调用返回，表示客户端到服务器端的单向连接建立成功，客户端的状态为ESTABLISHED，同时客户端协议栈也会对服务器端的SYN包进行应答，应答数据为k+1；</li>
<li>应答包到达服务器端后，服务器端协议栈使得accept阻塞调用返回，这个时候服务器端到客户端的单向连接也建立成功，服务器端也进入ESTABLISHED状态。</li>
</ol><p>形象一点的比喻是这样的，有A和B想进行通话：</p><ul>
<li>A先对B说：“喂，你在么？我在的，我的口令是j。”</li>
<li>B收到之后大声回答：“我收到你的口令j并准备好了，你准备好了吗？我的口令是k。”</li>
<li>A收到之后也大声回答：“我收到你的口令k并准备好了，我们开始吧。”</li>
</ul><p>可以看到，这样的应答过程总共进行了三次，这就是TCP连接建立之所以被叫为“三次握手”的原因了。</p><h2>总结</h2><p>这一讲我们分别从服务端和客户端的角度，讲述了如何创建套接字，并利用套接字完成TCP连接的建立。</p><ul>
<li>服务器端通过创建socket，bind，listen完成初始化，通过accept完成连接的建立。</li>
<li>客户端通过创建socket，connect发起连接建立请求。</li>
</ul><p>在下一讲里，我们将真正地开始客户端-服务端数据交互的过程。</p><h2>思考题</h2><p>最后给你布置两道思考题。</p><p>第一道是关于阻塞调用的，既然有阻塞调用，就应该有非阻塞调用，那么如何使用非阻塞调用套接字呢？使用的场景又是哪里呢？</p><p>第二道是关于客户端的，客户端发起connect调用之前，可以调用bind函数么？</p><p>欢迎你在评论区与我分享你的答案，如果这篇文章帮助你理解TCP三次握手，也欢迎你点击“请朋友读”，把这篇文章分享给你的朋友或者同事。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/14/97/8a3aa317.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>疾风知劲草</span>
  </div>
  <div class="_2_QraFYR_0">之前看过一些文章解释，为什么tcp建立连接需要三次握手，解释如下<br><br>tcp连接的双方要确保各自的收发消息的能力都是正常的。<br>客户端第一次发送握手消息到服务端，<br>服务端接收到握手消息后把ack和自己的syn一同发送给客户端，这是第二次握手，<br>当客户端接收到服务端发送来的第二次握手消息后，客户端可以确认“服务端的收发能力OK，客户端的收发能力OK”，但是服务端只能确认“客户端的发送OK，服务端的接收OK”，<br>所以还需要第三次握手，客户端收到服务端的第二次握手消息后，发起第三次握手消息，服务端收到客户端发送的第三次握手消息后，就能够确定“服务端的发送OK，客户端的接收OK”，<br>至此，客户端和服务端都能够确认自己和对方的收发能力OK，，tcp连接建立完成。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-09 10:05:14</div>
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
  <div class="_2_QraFYR_0">这个问题的本质是, 信道不可靠, 但是通信双发需要就某个问题达成一致. 而要解决这个问题, 无论你在消息中包含什么信息, 三次通信是理论上的最小值. 所以三次握手不是TCP本身的要求, 而是为了满足&quot;在不可靠信道上可靠地传输信息&quot;这一需求所导致的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-09 09:55:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/17/4d/7e13ec93.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>彭俊</span>
  </div>
  <div class="_2_QraFYR_0">如何使用非阻塞调用套接字:使用fcntl函数设置套接字的属性fcntl(fd, F_SETFL, flags)；<br>非阻塞调用的 使用的场景：程序在调用返回之前，需要做其他事情，可以选择用定时轮询或事件通知的方式获取调用结果。<br>是否可以调用bind 函数：可以，但是调用bind函数，也就是客户端指定了端口号，这样容易造成端口冲突，所以客户端不调用bind函数，让系统自动选择空闲端口比较好</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-11 15:43:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/9d/c0/cb5341ec.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>leesper</span>
  </div>
  <div class="_2_QraFYR_0">思考题1：非阻塞调用的场景就是高性能服务器编程！我所有的调用都不需要等待对方准备好了再返回，而是立即返回，那么我怎么知道是否准备好了？就是把这些fd注册到类似select或者epoll这样的调用中，变多个fd阻塞为一个fd阻塞，只要有任何一个fd准备好了，select或者epoll都会返回，然后我们在从中取出准备好了的fd进行各种IO操作，从容自然 ^o^</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞，下面的内容会讲到这部分了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-19 17:43:09</div>
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
  <div class="_2_QraFYR_0">关于三次握手<br>几句话解释清楚<br>1.信道不安全 保证通信需要一来一回<br>2.客户端的来回和服务端的来回 共四次 这是最多四次<br>3.客户端的回和服务端的来合并成一个，就是那个sync k ack j+1<br>4.这样就是三次握手<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 很清楚，很简洁。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-02 10:37:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/d2/a5/7acbd63a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>eddy</span>
  </div>
  <div class="_2_QraFYR_0">有个问题, 一次面试中遇到的, 客户端的第三次应答, 服务器没有收到, 然后客户端开始发消息给服务器, 这时候服务器和客户端的表现是什么? 客户端会收到什么返回?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 客户端发的消息是什么?如果三次应答服务器没有收到，服务器端连接没有建立，客户端误认为建立了，发送的报文应该会被设置为连接RST。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-09 15:24:33</div>
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
  <div class="_2_QraFYR_0">1. 非阻塞套接字往往配合多路复用机制，达到提高CPU利用率，实现高并发的目的。<br>2. 客户端做bind当然是可以的，但是因为连接请求往往是由客户端主动发起的，所以在客户端做bind显得不那么必要，还要承担端口冲突的风险。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-09 11:47:19</div>
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
  <div class="_2_QraFYR_0">学完这节，感觉TCP的三次握手机制，更明白了一些，相信此生都会记得，养成了翻评论的习惯，感觉有些同学的解释比老师的还要通俗易懂。<br>我的小结如下，假设只有客户端C和服务端S，两台机器：<br>1：啥是TCP的三次握手机制？<br>TCP的三次握手机制，是C和S使用TCP协议进行通信，他们在正式建立链接前，先进行了三次简单的通信，只有这三次简单的同学成功了，才建立正式的连接，之所以说是简单的通信，因为这三次通信发送的消息比较简单，就是固定格式的同步报文和确认报文<br><br>2：TCP的三次握手机制的目的是啥？<br>TCP协议的设计目标之一，就是保证通信信道的安全，而TCP的三次握手机制就是用于确认信道是否安全的手段之一，确认信道安全，最基本的首先要确认C和S都具备信息首发的能力吧！也就是C要确定S能收发信息，S要确定C能收发信息，咋确认呢？那就发送一条消息实验一下呗！所以，就有了TCP的三次握手机制，TCP的三次握手机制的核心目标就是要确认C和S之间的通信信道是安全的。<br><br>3：为啥是三次？<br>按道理来讲，C要确认S是否能够正确的收发消息，需要发生一条消息给S，然后接收到S的一条确认收到的消息才行，这一来一回就是两条消息。同理，S要确认C是否能够正确的收发消息，也需要这么玩。这样就需要两趟一来一回，总共需要四次通信，其实这么玩思维上一点负载都是没有的自然而然。<br>不过只是为了确认C和S能否正常通信的话，就如此设计，被聪明一看到就会骂傻X，人类孜孜不倦所追求是更快、更高、更强，计算机世界中这种追求更加的强烈，将S的确认收到C发送的消息和S能正常发生的消息一次性的都发给C岂不是更好，虽然增加了点消息的内容，但是相对于消息的传输消耗而言还是非常少的，而且从整体消耗上看，是减少了一次通信的过程，性能想必会更好。<br>很明显一次握手、两次握手都确认不了C和S的收发消息的能力是否OK。三次握手是比较简洁有效的方式，大于三次之上的握手机制也可以确认C和S是否能够正常通信，不过有些浪费资源了，毕竟三次就能搞定的事情，没必要搞三次至少，毕竟对于性能的追求我们是纳秒必争的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 🐂。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-20 08:54:16</div>
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
  <div class="_2_QraFYR_0">Linux内核2.2之后，socket backlog参数的形为改变了，现在它指等待accept的完全建立的套接字的队列长度，而不是不完全连接请求的数量。 不完全连接的长度可以使用&#47;proc&#47;sys&#47;net&#47;ipv4&#47;tcp_max_syn_backlog设置。<br>老师，这里你讲的backlog是未完成连接的队列，是不是有误？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我查看了一下资料，在Linux中backlog确实表示的是已完成(ESTABLISHED)且未accept的队列大小。而FreeBSD的实现是不同的，已经勘误修改。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-14 16:47:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/3c/24/ceac00af.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>dan</span>
  </div>
  <div class="_2_QraFYR_0">&quot;请问能听见吗&quot;&quot;我能听见你的声音，你能听见我的声音吗&quot;<br>A先对B：你在么？我在的，我发一个消息看你能不能收到，我发J；<br>B收到后，回答：我收到了你发的J，你的发送和我的接收功能正常,回你J+1;并且，我给你发个消息K，看我的发送和你的接收是否正常？<br>A收到后，回答：我收到了你发的J+1和K，我回你K+1，告诉你的发送和我的接收正常；<br>通过前2次，表明：起点的发送和终点的接收，功能正常；<br>通过后2次，表明：终点的发送和起点的接收，功能正常；<br>由此保证：双方可以发送和接收对方的信息。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 解说的非常形象，点赞。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-05 08:26:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/78/6b/9451a800.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>weeeee</span>
  </div>
  <div class="_2_QraFYR_0">老师好，有一个问题想请教您一下<br>服务器端的连接进入ESTABLISHED 状态，课程上说的是accept返回后才进入，但是我自己实验发现只要客户端发送ack后，也就是第三次握手完成就进入了ESTABLISHED 状态，和accept没有关系，请问一下老师是我哪里理解错了吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我的理解是accept之后会返回三次握手成功进入ESTABLISHED状态的连接，确实是先进入ESTABLISHED状态，然后从accept返回</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-13 11:05:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/a0/da/4f50f1b2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Knight²º¹⁸</span>
  </div>
  <div class="_2_QraFYR_0">老师我有一个问题，编程语言中的IO模型和操作系统中的IO关系是什么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 就是要和操作系统I&#47;O打交道，编程语言的I&#47;O模型是通过抽象和设计，总结出的一套规范。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-11 16:44:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/fd/91/65ff3154.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>_stuView</span>
  </div>
  <div class="_2_QraFYR_0">sockfd里的这个fd代表什么</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: file description，文件描述符。UNIX世界里万物皆文件。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-09 07:05:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/59/00/6d14972a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Arthur.Li</span>
  </div>
  <div class="_2_QraFYR_0">SYN：建立连接<br>FIN：关闭连接<br>ACK：响应<br>PSH：有DATA数据传输<br>RST：连接重置</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-02 19:10:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/bf/bd/0c40979f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>一周思进</span>
  </div>
  <div class="_2_QraFYR_0">刚好最近写了个通过man帮助手册编写基础tcp服务器<br>https:&#47;&#47;mp.weixin.qq.com&#47;s&#47;bNdfXNQcZ3z_WzDWL5yQfA</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-09 18:23:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/b6/6c/bae0461c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>浦上清风</span>
  </div>
  <div class="_2_QraFYR_0">1. 非阻塞 == 异步通信 ？？？<br>2. 可以是可以，但是不安全？？？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1 不是等价的<br>2 没有必要，还徒增了端口冲突的危险。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-09 17:49:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/99/f0/d9343049.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>星亦辰</span>
  </div>
  <div class="_2_QraFYR_0">思考题2 <br><br>客户端可以bind 指定使用固定端口来连接。 <br>没有bind 则会产生一个随机的端口来完成连接请求。<br><br>想到一个比较有意思的事情：<br><br>客户端bind 以后，对内网进行端口扫描，表象则是，远程随机端口，到本地固定端口完成通信。看似，是本地开启了服务。如果bind 80 443 22这些常规端口，则可以迷惑安全人员 😄 </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 想多了，可以区分出来是不是本地监听端口(被动套接字)的，而且这个在大多数情况下不被允许的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-09 07:48:05</div>
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
  <div class="_2_QraFYR_0">1、非阻塞调用为了提高网络处理性能，在select，poll和epoll等IO中使用。<br>2、客户端其实可以bind，但是没必要bind。客户端调用connect成功后系统会随机分配一个可用端口。这里涉及到端口选择算法和列维模型。<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-25 20:25:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/81/e6/6cafed37.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>旅途</span>
  </div>
  <div class="_2_QraFYR_0">老师 两个问题麻烦解答下<br>1.如果服务端 使用随机端口 那么客户端时怎么知道的 不使用随机的话 是客户端要先知道服务端的端口然后配置进去吗<br>2.三次握手时 每次通信的消息都加一  这个目的是什么 每次返回一个&quot;1&quot; 或者&quot;success&quot; 是不是也行?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1.客户端一定需要预先知道待连接的服务器端的地址和端口，至于怎么让客户端知道，通过配置，启动参数，都是可以的；<br>2.加1的目的是为了对这个报文进行唯一性确认，防止报文串掉了，这样我们就无法知道谁跟谁了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-28 13:19:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/50/ae/11dad6f3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>啦啦的小猪</span>
  </div>
  <div class="_2_QraFYR_0">listen的第二个参数backlog的值是规定ACCEPT队列大小的;当ACCEPT队列满了时后来的客户端，就会被放到syn队列中</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 每个系统的实现和解释不尽相同，你说的是一个很典型的实现。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-13 00:49:33</div>
  </div>
</div>
</div>
</li>
</ul>