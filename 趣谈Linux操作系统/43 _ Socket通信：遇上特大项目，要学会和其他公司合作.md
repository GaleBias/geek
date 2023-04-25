<audio title="43 _ Socket通信：遇上特大项目，要学会和其他公司合作" src="https://static001.geekbang.org/resource/audio/1a/07/1a90cc49c2b9f27bc2ed6159f1e6f107.mp3" controls="controls"></audio> 
<p>上一篇预习文章说了这么多，现在我们终于可以来看一下，在应用层，我们应该如何使用socket的接口来进行通信。</p><p>如果你对socket相关的网络协议原理不是非常了解，建议你先去看一看上一篇的预习文章，再来看这一篇的内容，就会比较轻松。</p><p>按照前一篇文章说的分层机制，我们可以想到，socket接口大多数情况下操作的是传输层，更底层的协议不用它来操心，这就是分层的好处。</p><p>在传输层有两个主流的协议TCP和UDP，所以我们的socket程序设计也是主要操作这两个协议。这两个协议的区别是什么呢？通常的答案是下面这样的。</p><ul>
<li>TCP是面向连接的，UDP是面向无连接的。</li>
<li>TCP提供可靠交付，无差错、不丢失、不重复、并且按序到达；UDP不提供可靠交付，不保证不丢失，不保证按顺序到达。</li>
<li>TCP是面向字节流的，发送时发的是一个流，没头没尾；UDP是面向数据报的，一个一个地发送。</li>
<li>TCP是可以提供流量控制和拥塞控制的，既防止对端被压垮，也防止网络被压垮。</li>
</ul><p>这些答案没有问题，但是没有到达本质，也经常让人产生错觉。例如，下面这些问题，你看看你是否了解？</p><ul>
<li>所谓的连接，容易让人误以为，使用TCP会使得两端之间的通路和使用UDP不一样，那我们会在沿途建立一条线表示这个连接吗？</li>
<li>我从中国访问美国网站，中间这么多环节，我怎么保证连接不断呢？</li>
<li>中间有个网络管理员拔了一根网线不就断了吗？我不能控制它，它也不会通知我，我一个个人电脑怎么能够保持连接呢？</li>
<li>还让我做流量控制和拥塞控制，我既管不了中间的链路，也管不了对端的服务器呀，我怎么能够做到？</li>
<li>按照网络分层，TCP和UDP都是基于IP协议的，IP都不能保证可靠，说丢就丢，TCP怎么能够保证呢？</li>
<li>IP层都是一个包一个包地发送，TCP怎么就变成流了？</li>
</ul><!-- [[[read_end]]] --><p>从本质上来讲，所谓的<strong>建立连接</strong>，其实是为了在客户端和服务端维护连接，而建立一定的数据结构来维护双方交互的状态，并用这样的数据结构来保证面向连接的特性。TCP无法左右中间的任何通路，也没有什么虚拟的连接，中间的通路根本意识不到两端使用了TCP还是UDP。</p><p>所谓的<strong>连接</strong>，就是两端数据结构状态的协同，两边的状态能够对得上。符合TCP协议的规则，就认为连接存在；两面状态对不上，连接就算断了。</p><p>流量控制和拥塞控制其实就是根据收到的对端的网络包，调整两端数据结构的状态。TCP协议的设计理论上认为，这样调整了数据结构的状态，就能进行流量控制和拥塞控制了，其实在通路上是不是真的做到了，谁也管不着。</p><p>所谓的<strong>可靠</strong>，也是两端的数据结构做的事情。不丢失其实是数据结构在“点名”，顺序到达其实是数据结构在“排序”，面向数据流其实是数据结构将零散的包，按照顺序捏成一个流发给应用层。总而言之，“连接”两个字让人误以为功夫在通路，其实功夫在两端。</p><p>当然，无论是用socket操作TCP，还是UDP，我们首先都要调用socket函数。</p><pre><code>int socket(int domain, int type, int protocol);
</code></pre><p>socket函数用于创建一个socket的文件描述符，唯一标识一个socket。我们把它叫作文件描述符，因为在内核中，我们会创建类似文件系统的数据结构，并且后续的操作都有用到它。</p><p>socket函数有三个参数。</p><ul>
<li>domain：表示使用什么IP层协议。AF_INET表示IPv4，AF_INET6表示IPv6。</li>
<li>type：表示socket类型。SOCK_STREAM，顾名思义就是TCP面向流的，SOCK_DGRAM就是UDP面向数据报的，SOCK_RAW可以直接操作IP层，或者非TCP和UDP的协议。例如ICMP。</li>
<li>protocol表示的协议，包括IPPROTO_TCP、IPPTOTO_UDP。</li>
</ul><p>通信结束后，我们还要像关闭文件一样，关闭socket。</p><h2>针对TCP应该如何编程？</h2><p>接下来我们来看，针对TCP，我们应该如何编程。</p><p><img src="https://static001.geekbang.org/resource/image/99/da/997e39e5574252ada22220e4b3646dda.png?wh=1303*1603" alt=""></p><p>TCP的服务端要先监听一个端口，一般是先调用bind函数，给这个socket赋予一个端口和IP地址。</p><pre><code>int bind(int sockfd, const struct sockaddr *addr,socklen_t addrlen);

struct sockaddr_in {
  __kernel_sa_family_t	sin_family;	/* Address family		*/
  __be16		sin_port;	/* Port number			*/
  struct in_addr	sin_addr;	/* Internet address		*/

  /* Pad to size of `struct sockaddr'. */
  unsigned char		__pad[__SOCK_SIZE__ - sizeof(short int) -
			sizeof(unsigned short int) - sizeof(struct in_addr)];
};

struct in_addr {
	__be32	s_addr;
};
</code></pre><p>其中，sockfd是上面我们创建的socket文件描述符。在sockaddr_in结构中，sin_family设置为AF_INET，表示IPv4；sin_port是端口号；sin_addr是IP地址。</p><p>服务端所在的服务器可能有多个网卡、多个地址，可以选择监听在一个地址，也可以监听0.0.0.0表示所有的地址都监听。服务端一般要监听在一个众所周知的端口上，例如，Nginx一般是80，Tomcat一般是8080。</p><p>客户端要访问服务端，肯定事先要知道服务端的端口。无论是电商，还是游戏，还是视频，如果你仔细观察，会发现都有一个这样的端口。可能你会发现，客户端不需要bind，因为浏览器嘛，随机分配一个端口就可以了，只有你主动去连接别人，别人不会主动连接你，没有人关心客户端监听到了哪里。</p><p>如果你看上面代码中的数据结构，里面的变量名称都有“be”两个字母，代表的意思是“big-endian”。如果在网络上传输超过1 Byte的类型，就要区分<strong>大端</strong>（Big Endian）和<strong>小端</strong>（Little Endian）。</p><p>假设，我们要在32位4 Bytes的一个空间存放整数1，很显然只要1 Byte放1，其他3 Bytes放0就可以了。那问题是，最后一个Byte放1呢，还是第一个Byte放1呢？或者说，1作为最低位，应该放在32位的最后一个位置呢，还是放在第一个位置呢？</p><p>最低位放在最后一个位置，我们叫作小端，最低位放在第一个位置，叫作大端。TCP/IP栈是按照大端来设计的，而x86机器多按照小端来设计，因而发出去时需要做一个转换。</p><p>接下来，就要建立TCP的连接了，也就是著名的三次握手，其实就是将客户端和服务端的状态通过三次网络交互，达到初始状态是协同的状态。下图就是三次握手的序列图以及对应的状态转换。</p><p><img src="https://static001.geekbang.org/resource/image/0e/a4/0ef257133471e95bd334383e0155fda4.png?wh=1423*1063" alt=""></p><p>接下来，服务端要调用listen进入LISTEN状态，等待客户端进行连接。</p><pre><code>int listen(int sockfd, int backlog);
</code></pre><p>连接的建立过程，也即三次握手，是TCP层的动作，是在内核完成的，应用层不需要参与。</p><p>接着，服务端只需要调用accept，等待内核完成了至少一个连接的建立，才返回。如果没有一个连接完成了三次握手，accept就一直等待；如果有多个客户端发起连接，并且在内核里面完成了多个三次握手，建立了多个连接，这些连接会被放在一个队列里面。accept会从队列里面取出一个来进行处理。如果想进一步处理其他连接，需要调用多次accept，所以accept往往在一个循环里面。</p><pre><code>int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
</code></pre><p>接下来，客户端可以通过connect函数发起连接。</p><pre><code>int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
</code></pre><p>我们先在参数中指明要连接的IP地址和端口号，然后发起三次握手。内核会给客户端分配一个临时的端口。一旦握手成功，服务端的accept就会返回另一个socket。</p><p>这里需要注意的是，监听的socket和真正用来传送数据的socket，是两个socket，一个叫作<strong>监听socket</strong>，一个叫作<strong>已连接socket</strong>。成功连接建立之后，双方开始通过read和write函数来读写数据，就像往一个文件流里面写东西一样。</p><h2>针对UDP应该如何编程？</h2><p>接下来我们来看，针对UDP应该如何编程。</p><p><img src="https://static001.geekbang.org/resource/image/28/b2/283b0e1c21f0277ba5b4b5cbcaca03b2.png?wh=1303*1243" alt=""></p><p>UDP是没有连接的，所以不需要三次握手，也就不需要调用listen和connect，但是UDP的交互仍然需要IP地址和端口号，因而也需要bind。</p><p>对于UDP来讲，没有所谓的连接维护，也没有所谓的连接的发起方和接收方，甚至都不存在客户端和服务端的概念，大家就都是客户端，也同时都是服务端。只要有一个socket，多台机器就可以任意通信，不存在哪两台机器是属于一个连接的概念。因此，每一个UDP的socket都需要bind。每次通信时，调用sendto和recvfrom，都要传入IP地址和端口。</p><pre><code>ssize_t sendto(int sockfd, const void *buf, size_t len, int flags, const struct sockaddr *dest_addr, socklen_t addrlen);

ssize_t recvfrom(int sockfd, void *buf, size_t len, int flags, struct sockaddr *src_addr, socklen_t *addrlen);
</code></pre><h2>总结时刻</h2><p>这一节我们讲了网络协议的基本原理和socket系统调用，这里请你重点关注TCP协议的系统调用。</p><p>通过学习，我们知道，socket系统调用是用户态和内核态的接口，网络协议的四层以下都是在内核中的。很多的书籍会讲如何开发一个高性能的socket程序，但是这不是我们这门课的重点，所以我们主要看内核里面的机制就行了。</p><p>因此，你需要记住TCP协议的socket调用的过程。我们接下来就按照这个顺序，依次回忆一下这些系统调用到内核都做了什么：</p><ul>
<li>服务端和客户端都调用socket，得到文件描述符；</li>
<li>服务端调用listen，进行监听；</li>
<li>服务端调用accept，等待客户端连接；</li>
<li>客户端调用connect，连接服务端；</li>
<li>服务端accept返回用于传输的socket的文件描述符；</li>
<li>客户端调用write写入数据；</li>
<li>服务端调用read读取数据。</li>
</ul><p><img src="https://static001.geekbang.org/resource/image/d3/5c/d34e667d1c3340deb8c82a2d44f2a65c.png?wh=1786*1375" alt=""></p><h2>课堂练习</h2><p>请你根据今天讲的socket系统调用，写一个简单的socket程序来传输一个字符串。</p><p>欢迎留言和我分享你的疑惑和见解 ，也欢迎可以收藏本节内容，反复研读。你也可以把今天的内容分享给你的朋友，和他一起学习和进步。</p><p><img src="https://static001.geekbang.org/resource/image/8c/37/8c0a95fa07a8b9a1abfd394479bdd637.jpg?wh=1110*659" alt=""></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/39/fa/a7edbc72.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>安排</span>
  </div>
  <div class="_2_QraFYR_0">老师，可不可以在答疑篇，增加一个select,poll,epoll的内核机制分析？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-05 00:35:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/5e/96/a03175bc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>莫名</span>
  </div>
  <div class="_2_QraFYR_0">『连接功夫在两端，而不在通路。通过两端的底层sock结构体维持状态信息。』老师这句话总结很到位结合socket系统调用的源码分析会更加容易理解。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-29 09:30:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83eqyicZYyW7ahaXgXUD8ZAS8x0t8jx5rYLhwbUCJiawRepKIZfsLdkxdQ9XQMo99c1UDibmNVfFnAqwPg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>程序水果宝</span>
  </div>
  <div class="_2_QraFYR_0">为什么要两个socket？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 需要两个数据结构保存不同的状态</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-11 23:34:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/50/4a/04fef27f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>kdb_reboot</span>
  </div>
  <div class="_2_QraFYR_0">老师厉害了, 依然在更新;<br>最近我有时间学习这个专栏了, 但是目前只跟到第十课, 把专栏作为引子,每天的阅读量还是很大的<br>然后, 我有个问题: 专栏更新完老师还会答疑吗?因为进度原因,可能还没学到最后面,专栏已经更新完了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个专栏比较硬核</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-05 22:29:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/83/0c/b9e39db4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>韩俊臣</span>
  </div>
  <div class="_2_QraFYR_0">撑过进程间通信后，这里终于多少能看懂了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-14 08:19:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/df/ca/7c223fce.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>天使也有爱</span>
  </div>
  <div class="_2_QraFYR_0">看了趣谈网络协议专栏，在结合这里看，感觉对网络通信知识有了更深的理解</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-22 12:29:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/4c/8f/a90b3969.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>oldman</span>
  </div>
  <div class="_2_QraFYR_0">老师，有个问题，一直没有想明白，希望老师看到之后给解答一下，我知道服务端会维护一个连接的队列，比如这个队列里面是a,b,c,d,e,f,g这个样的多个连接，那当客户端有请求过来，比如说某一个请求过来，服务端是怎么区分他是a还是b或者c对应的连接呢？谢谢。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 每个连接都是不同的端口号</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-08 14:43:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/97/83/845b48e2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Allen_</span>
  </div>
  <div class="_2_QraFYR_0">这一章解决了我一年都问不到的答案 谢谢老师</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-06 17:18:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/93/cc/dfe92ee1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>宋桓公</span>
  </div>
  <div class="_2_QraFYR_0">原来0000，是指监听一个服务器的全部网卡，soga</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-14 07:49:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/a2/55/1092ebb8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>边城路远</span>
  </div>
  <div class="_2_QraFYR_0">老师好，多个网卡可以bind多次吗</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-30 18:57:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/ea/a9/0a917f2c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Sunny</span>
  </div>
  <div class="_2_QraFYR_0">监听 socket 在服务端，已连接 socket 在客户端？💔</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-12 20:00:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/d0/d0/a6c6069d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>坚</span>
  </div>
  <div class="_2_QraFYR_0">老师，UDP流程的图片中，客户端不需要bind吧？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-05 21:45:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>K菌无惨</span>
  </div>
  <div class="_2_QraFYR_0">不错 讲的很清晰</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-18 16:53:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/79/14/dc3e49d1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_29c23f</span>
  </div>
  <div class="_2_QraFYR_0">三次握手是在listern阶段还是accept阶段。listen不阻塞，执行完后直接在accept下阻塞等待三次握手。那三次握手应该在accept阶段才 啊</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-22 12:16:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/7a/93/c9302518.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>高志强</span>
  </div>
  <div class="_2_QraFYR_0">用户态和内核态交互要通过socket操作tcp或udp协议，所以socket必然存在，可以这样理解么  老师</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-15 22:03:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/c5/71/f7c43b49.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>风向北吹</span>
  </div>
  <div class="_2_QraFYR_0">在编写java socket发现一个问题，客户端通过socket发送数据到通过socket接收数据的过程中，必须调用socket. shutdownOutput()才可以收到，而服务端发送到接收或者接收到发送中间转换却不需要一个关流的操作，这是为什么呀</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-13 08:07:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/b4/ee/c3ff8615.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>恒</span>
  </div>
  <div class="_2_QraFYR_0">老师，UDP不需要listen和connect，那是不是UDP从头到尾就一个socket就够了，不像TCP区分监听socket和已连接socket？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-11 08:45:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/db/26/54f2c164.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>靠人品去赢</span>
  </div>
  <div class="_2_QraFYR_0">TCP面向连接，HTTP无状态总是搞混，问一下基于TCP的HTTP为什么不设置一个状态依赖的东西，要靠cookie和session来帮忙呢？<br>这个UDP上学的时候知道实时通话视频会用到，丢包丢多了是不是就是我们感觉“卡卡的”掉帧的情况。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-10 16:49:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/15/5c/aa3f8306.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>潇是潇洒的洒</span>
  </div>
  <div class="_2_QraFYR_0">老师我有一个疑问，服务端和客户端都调用 socket，得到文件描述符。这里是服务端和客户端分别打开了不同的文件，然后各自写对方的文件，读自己的文件，还是说打开是同一个文件，读和写。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不同的文件</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-02 17:03:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/d1/15/7d47de48.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>咖啡猫口里的咖啡猫🐱</span>
  </div>
  <div class="_2_QraFYR_0">其实我对tcp和udp的理解就是tcp协议栈由分段maxsegment(握手阶段的附加)，自己尽量来处理最大MTU问题，尽量防止ip分片，对端网络层组包，从而导致的tcp应用使用协议需要考虑处理分包，粘包。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以offload给硬件</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-18 16:37:59</div>
  </div>
</div>
</div>
</li>
</ul>