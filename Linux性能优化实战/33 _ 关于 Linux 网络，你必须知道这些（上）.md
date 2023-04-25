<audio title="33 _ 关于 Linux 网络，你必须知道这些（上）" src="https://static001.geekbang.org/resource/audio/37/39/377ecc334777ee0d28cb8ca30de4b039.mp3" controls="controls"></audio> 
<p>你好，我是倪朋飞。</p><p>前几节，我们一起学习了文件系统和磁盘 I/O 的工作原理，以及相应的性能分析和优化方法。接下来，我们将进入下一个重要模块—— Linux 的网络子系统。</p><p>由于网络处理的流程最复杂，跟我们前面讲到的进程调度、中断处理、内存管理以及 I/O 等都密不可分，所以，我把网络模块作为最后一个资源模块来讲解。</p><p>同 CPU、内存以及 I/O 一样，网络也是 Linux 系统最核心的功能。网络是一种把不同计算机或网络设备连接到一起的技术，它本质上是一种进程间通信方式，特别是跨系统的进程间通信，必须要通过网络才能进行。随着高并发、分布式、云计算、微服务等技术的普及，网络的性能也变得越来越重要。</p><p>那么，Linux 网络又是怎么工作的呢？又有哪些指标衡量网络的性能呢？接下来的两篇文章，我将带你一起学习 Linux 网络的工作原理和性能指标。</p><h2>网络模型</h2><p>说到网络，我想你肯定经常提起七层负载均衡、四层负载均衡，或者三层设备、二层设备等等。那么，这里说的二层、三层、四层、七层又都是什么意思呢？</p><p>实际上，这些层都来自国际标准化组织制定的<strong>开放式系统互联通信参考模型</strong>（Open System Interconnection Reference Model），简称为 OSI 网络模型。</p><!-- [[[read_end]]] --><p>为了解决网络互联中异构设备的兼容性问题，并解耦复杂的网络包处理流程，OSI 模型把网络互联的框架分为应用层、表示层、会话层、传输层、网络层、数据链路层以及物理层等七层，每个层负责不同的功能。其中，</p><ul>
<li>
<p>应用层，负责为应用程序提供统一的接口。</p>
</li>
<li>
<p>表示层，负责把数据转换成兼容接收系统的格式。</p>
</li>
<li>
<p>会话层，负责维护计算机之间的通信连接。</p>
</li>
<li>
<p>传输层，负责为数据加上传输表头，形成数据包。</p>
</li>
<li>
<p>网络层，负责数据的路由和转发。</p>
</li>
<li>
<p>数据链路层，负责MAC寻址、错误侦测和改错。</p>
</li>
<li>
<p>物理层，负责在物理网络中传输数据帧。</p>
</li>
</ul><p>但是 OSI 模型还是太复杂了，也没能提供一个可实现的方法。所以，在 Linux 中，我们实际上使用的是另一个更实用的四层模型，即 TCP/IP 网络模型。</p><p>TCP/IP 模型，把网络互联的框架分为应用层、传输层、网络层、网络接口层等四层，其中，</p><ul>
<li>
<p>应用层，负责向用户提供一组应用程序，比如 HTTP、FTP、DNS 等。</p>
</li>
<li>
<p>传输层，负责端到端的通信，比如 TCP、UDP 等。</p>
</li>
<li>
<p>网络层，负责网络包的封装、寻址和路由，比如 IP、ICMP 等。</p>
</li>
<li>
<p>网络接口层，负责网络包在物理网络中的传输，比如 MAC 寻址、错误侦测以及通过网卡传输网络帧等。</p>
</li>
</ul><p>为了帮你更形象理解TCP/IP 与 OSI 模型的关系，我画了一张图，如下所示：</p><p><img src="https://static001.geekbang.org/resource/image/f2/bd/f2dbfb5500c2aa7c47de6216ee7098bd.png?wh=591*521" alt=""></p><p>当然了，虽说 Linux 实际按照 TCP/IP 模型，实现了网络协议栈，但在平时的学习交流中，我们习惯上还是用 OSI 七层模型来描述。比如，说到七层和四层负载均衡，对应的分别是 OSI 模型中的应用层和传输层（而它们对应到 TCP/IP 模型中，实际上是四层和三层）。</p><p>TCP/IP 模型包括了大量的网络协议，这些协议的原理，也是我们每个人必须掌握的核心基础知识。如果你不太熟练，推荐你去学《TCP/IP 详解》的卷一和卷二，或者学习极客时间出品的《<a href="https://time.geekbang.org/course/intro/85">趣谈网络协议</a>》专栏。</p><h2>Linux网络栈</h2><p>有了 TCP/IP 模型后，在进行网络传输时，数据包就会按照协议栈，对上一层发来的数据进行逐层处理；然后封装上该层的协议头，再发送给下一层。</p><p>当然，网络包在每一层的处理逻辑，都取决于各层采用的网络协议。比如在应用层，一个提供 REST API 的应用，可以使用 HTTP 协议，把它需要传输的 JSON 数据封装到 HTTP 协议中，然后向下传递给 TCP 层。</p><p>而封装做的事情就很简单了，只是在原来的负载前后，增加固定格式的元数据，原始的负载数据并不会被修改。</p><p>比如，以通过 TCP 协议通信的网络包为例，通过下面这张图，我们可以看到，应用程序数据在每个层的封装格式。</p><p><img src="https://static001.geekbang.org/resource/image/c8/79/c8dfe80acc44ba1aa9df327c54349e79.png?wh=525*254" alt=""></p><p>其中：</p><ul>
<li>
<p>传输层在应用程序数据前面增加了 TCP 头；</p>
</li>
<li>
<p>网络层在 TCP 数据包前增加了 IP 头；</p>
</li>
<li>
<p>而网络接口层，又在 IP 数据包前后分别增加了帧头和帧尾。</p>
</li>
</ul><p>这些新增的头部和尾部，都按照特定的协议格式填充，想了解具体格式，你可以查看协议的文档。 比如，你可以查看<a href="https://zh.wikipedia.org/wiki/%E4%BC%A0%E8%BE%93%E6%8E%A7%E5%88%B6%E5%8D%8F%E8%AE%AE#%E5%B0%81%E5%8C%85%E7%B5%90%E6%A7%8B">这里</a>，了解 TCP 头的格式。</p><p>这些新增的头部和尾部，增加了网络包的大小，但我们都知道，物理链路中并不能传输任意大小的数据包。网络接口配置的最大传输单元（MTU），就规定了最大的 IP 包大小。在我们最常用的以太网中，MTU 默认值是 1500（这也是 Linux 的默认值）。</p><p>一旦网络包超过 MTU 的大小，就会在网络层分片，以保证分片后的 IP 包不大于MTU 值。显然，MTU 越大，需要的分包也就越少，自然，网络吞吐能力就越好。</p><p>理解了 TCP/IP 网络模型和网络包的封装原理后，你很容易能想到，Linux 内核中的网络栈，其实也类似于 TCP/IP 的四层结构。如下图所示，就是 Linux 通用 IP 网络栈的示意图：</p><p><img src="https://static001.geekbang.org/resource/image/c7/ac/c7b5b16539f90caabb537362ee7c27ac.png?wh=1092*1316" alt=""></p><p>（图片参考《性能之巅》图 10.7 通用 IP 网络栈绘制）</p><p>我们从上到下来看这个网络栈，你可以发现，</p><ul>
<li>
<p>最上层的应用程序，需要通过系统调用，来跟套接字接口进行交互；</p>
</li>
<li>
<p>套接字的下面，就是我们前面提到的传输层、网络层和网络接口层；</p>
</li>
<li>
<p>最底层，则是网卡驱动程序以及物理网卡设备。</p>
</li>
</ul><p>这里我简单说一下网卡。网卡是发送和接收网络包的基本设备。在系统启动过程中，网卡通过内核中的网卡驱动程序注册到系统中。而在网络收发过程中，内核通过中断跟网卡进行交互。</p><p>再结合前面提到的 Linux 网络栈，可以看出，网络包的处理非常复杂。所以，网卡硬中断只处理最核心的网卡数据读取或发送，而协议栈中的大部分逻辑，都会放到软中断中处理。</p><h2>Linux网络收发流程</h2><p>了解了 Linux 网络栈后，我们再来看看， Linux 到底是怎么收发网络包的。</p><blockquote>
<p>注意，以下内容都以物理网卡为例。事实上，Linux 还支持众多的虚拟网络设备，而它们的网络收发流程会有一些差别。</p>
</blockquote><h3>网络包的接收流程</h3><p>我们先来看网络包的接收流程。</p><p>当一个网络帧到达网卡后，网卡会通过 DMA 方式，把这个网络包放到收包队列中；然后通过硬中断，告诉中断处理程序已经收到了网络包。</p><p>接着，网卡中断处理程序会为网络帧分配内核数据结构（sk_buff），并将其拷贝到 sk_buff 缓冲区中；然后再通过软中断，通知内核收到了新的网络帧。</p><p>接下来，内核协议栈从缓冲区中取出网络帧，并通过网络协议栈，从下到上逐层处理这个网络帧。比如，</p><ul>
<li>
<p>在链路层检查报文的合法性，找出上层协议的类型（比如 IPv4 还是 IPv6），再去掉帧头、帧尾，然后交给网络层。</p>
</li>
<li>
<p>网络层取出 IP 头，判断网络包下一步的走向，比如是交给上层处理还是转发。当网络层确认这个包是要发送到本机后，就会取出上层协议的类型（比如 TCP 还是 UDP），去掉 IP 头，再交给传输层处理。</p>
</li>
<li>
<p>传输层取出 TCP 头或者 UDP 头后，根据 &lt;源 IP、源端口、目的 IP、目的端口&gt; 四元组作为标识，找出对应的 Socket，并把数据拷贝到 Socket 的接收缓存中。</p>
</li>
</ul><p>最后，应用程序就可以使用 Socket 接口，读取到新接收到的数据了。</p><p>为了更清晰表示这个流程，我画了一张图，这张图的左半部分表示接收流程，而图中的粉色箭头则表示网络包的处理路径。</p><p><img src="https://static001.geekbang.org/resource/image/3a/65/3af644b6d463869ece19786a4634f765.png?wh=1826*1118" alt=""></p><h3>网络包的发送流程</h3><p>了解网络包的接收流程后，就很容易理解网络包的发送流程。网络包的发送流程就是上图的右半部分，很容易发现，网络包的发送方向，正好跟接收方向相反。</p><p>首先，应用程序调用 Socket API（比如 sendmsg）发送网络包。</p><p>由于这是一个系统调用，所以会陷入到内核态的套接字层中。套接字层会把数据包放到 Socket 发送缓冲区中。</p><p>接下来，网络协议栈从 Socket 发送缓冲区中，取出数据包；再按照 TCP/IP 栈，从上到下逐层处理。比如，传输层和网络层，分别为其增加 TCP 头和 IP 头，执行路由查找确认下一跳的 IP，并按照 MTU 大小进行分片。</p><p>分片后的网络包，再送到网络接口层，进行物理地址寻址，以找到下一跳的 MAC 地址。然后添加帧头和帧尾，放到发包队列中。这一切完成后，会有软中断通知驱动程序：发包队列中有新的网络帧需要发送。</p><p>最后，驱动程序通过 DMA ，从发包队列中读出网络帧，并通过物理网卡把它发送出去。</p><h2><strong>小结</strong></h2><p>在今天的文章中，我带你一起梳理了 Linux 网络的工作原理。</p><p>多台服务器通过网卡、交换机、路由器等网络设备连接到一起，构成了相互连接的网络。由于网络设备的异构性和网络协议的复杂性，国际标准化组织定义了一个七层的 OSI 网络模型，但是这个模型过于复杂，实际工作中的事实标准，是更为实用的 TCP/IP 模型。</p><p>TCP/IP 模型，把网络互联的框架，分为应用层、传输层、网络层、网络接口层等四层，这也是 Linux 网络栈最核心的构成部分。</p><ul>
<li>
<p>应用程序通过套接字接口发送数据包，先要在网络协议栈中从上到下进行逐层处理，最终再送到网卡发送出去。</p>
</li>
<li>
<p>而接收时，同样先经过网络栈从下到上的逐层处理，最终才会送到应用程序。</p>
</li>
</ul><p>了解了Linux 网络的基本原理和收发流程后，你肯定迫不及待想知道，如何去观察网络的性能情况。那么，具体来说，哪些指标可以衡量 Linux 的网络性能呢？别急，我将在下一节中为你详细讲解。</p><h2>思考</h2><p>最后，我想请你来聊聊你所理解的 Linux 网络。你碰到过哪些网络相关的性能瓶颈？你又是怎么样来分析它们的呢？你可以结合今天学到的网络知识，提出自己的观点。</p><p>欢迎在留言区和我讨论，也欢迎你把这篇文章分享给你的同事、朋友。我们一起在实战中演练，在交流中进步。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83ernR4NKI5tejJAV3HMTF3gszBBUAjkjLO2QYic2gx5dMGelFv4LWibib7CUGexmMcMp5HiaaibmOH3dyHg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>渡渡鸟_linux</span>
  </div>
  <div class="_2_QraFYR_0">我结合网络上查阅的资料和文章中的内容，总结了下网卡收发报文的过程，不知道是否正确：<br>1. 内核分配一个主内存地址段（DMA缓冲区)，网卡设备可以在DMA缓冲区中读写数据<br>2. 当来了一个网络包，网卡将网络包写入DMA缓冲区，写完后通知CPU产生硬中断<br>3. 硬中断处理程序锁定当前DMA缓冲区，然后将网络包拷贝到另一块内存区，清空并解锁当前DMA缓冲区，然后通知软中断去处理网络包。<br>-----<br>当发送数据包时，与上述相反。链路层将数据包封装完毕后，放入网卡的DMA缓冲区，并调用系统硬中断，通知网卡从缓冲区读取并发送数据。<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-07 22:01:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/39/fa/a7edbc72.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>安排</span>
  </div>
  <div class="_2_QraFYR_0">当一个网络帧到达网卡后，网卡会通过 DMA 方式，把这个网络包放到收包队列中；然后通过硬中断，告诉中断处理程序已经收到了网络包。<br><br>接着，网卡中断处理程序会为网络帧分配内核数据结构（sk_buff），并将其拷贝到 sk_buff 缓冲区中；然后再通过软中断，通知内核收到了新的网络帧。<br><br>接下来，内核协议栈从缓冲区中取出网络帧，并通过网络协议栈，从下到上逐层处理这个网络帧。<br><br><br><br>老师你好，上面的一段话有些疑问想请教一下。<br><br>收包队列是属于哪里的存储空间，是属于物理内存吗，还是网卡中的存储空间，通过dma方式把数据放到收包队列，我猜这个收包队列是物理内存中的空间。这个收包队列是由内核管理的吧，也就是跟某一个进程的用户空间地址没关系？   <br><br>那sk_buf缓冲区又是哪里的存储空间，为什么还要把收包队列拷贝到这个缓冲区呢，这个缓冲区是协议栈维护的吗？也属于内核，跟进程的用户空间地址有关系吗？<br><br><br>socket的接收发送缓冲区是映射到进程的用户空间地址的吗？还是由协议栈为每个socket在内核中维护的缓冲区？<br><br>还有上面说到的这些缓冲区跟cache和buf有什么关系？会被回收吗？<br><br><br>内核协议栈的运行是通过一个内核线程的方式来运行的吗？是否可以看到这个线程的名字？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 问题比较多，放到答疑篇里面统一回复吧</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-26 09:52:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/69/88/528442b0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Dale</span>
  </div>
  <div class="_2_QraFYR_0">网络报文传需要在用户态和内核态来回切换，导致性能下降。业界使用零拷贝或intel的dpdk来提高性能。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-12 08:13:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://wx.qlogo.cn/mmopen/vi_32/PiajxSqBRaEKQMM4m7NHuicr55aRiblTSEWIYe0QqbpyHweaoAbG7j2v7UUElqqeP3Ihrm3UfDPDRb1Hv8LvPwXqA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ninuxer</span>
  </div>
  <div class="_2_QraFYR_0">打卡day35<br>有一次业务反馈有些请求无法正常响应，后来花了两天时间才发现ifconfig看网卡的drop的包不断增长，后来发现是跟开启了内核的timestamp参数有关</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-13 19:47:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/AkO5s3tJhibth9nelCNdU5qD4J3aEn8OpBhOHluicWgEj1SbcGC6e9rccK8DrfJtRibJT5g6iamfIibt5xX7ketDF6w/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Penn</span>
  </div>
  <div class="_2_QraFYR_0">中断不均，连接跟踪打满</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯 这是最常见的两个问题</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-06 06:20:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKq0oQVibKcmYJqmpqaNNQibVgia7EsEgW65LZJIpDZBMc7FyMcs7J1JmFCtp06pY8ibbcpW4ibRtG7Frg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>zhoufeng</span>
  </div>
  <div class="_2_QraFYR_0">老师好，一直不太明白skb_buff和sk_buff的区别，这两者有关系吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: sk_buff一般是说内核数据接口，而 skb则是套接字缓存(socket buffer)</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-10 17:00:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJUP6ibuQssqJBNtQdSaFNhzzibdf7I3nyVGCeJPoDYqfsRndqRY19GpOJCOibMXQmOv2EchtHh0SXow/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_00d753</span>
  </div>
  <div class="_2_QraFYR_0">收数据的时候，从网卡到应用层socket。需要一次硬中断+一次软中断。<br>发数据的时候只需要一次软中断。<br>是这样吗？老师</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-22 08:50:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/92/01/c723d180.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>饼子</span>
  </div>
  <div class="_2_QraFYR_0">遇到了程序分配大量链接，占用完程序最大打开文件数量，使用lsof 查看分析的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯嗯，文件数量或者连接数量都可以看到</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-20 21:08:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/83/69/77256c74.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>空白</span>
  </div>
  <div class="_2_QraFYR_0">一直不太理解，网络包是如何具体交付给对应的线程的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 系统调用</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-14 17:04:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/5b/ae/3d639ea4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>佳</span>
  </div>
  <div class="_2_QraFYR_0">使用InfiniBand网卡和InfiniBand交换机的时候， mtu如果配置65520的时候，通过http下载对象存储小文件比较慢，但是配置9000的时候大小文件都比较快。https:&#47;&#47;github.com&#47;antirez&#47;redis&#47;issues&#47;2385 Redis works very slow with MTU higher than packet size. 请问老师是什么原因<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: MTU大小的问题都是分片和重组导致的。后面有案例讲到分析内核中网络协议栈的行为，你可以到时候试着分析下这种场景</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-13 20:01:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/61/14/2f9fec68.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>空空</span>
  </div>
  <div class="_2_QraFYR_0">老师过年好！<br>曾经在Linux3.10测试netlink收发包效率，发现一个问题，正常情况下每收一个包大概需要10us，但是每隔8秒会出现一次收包时间30-50ms，就是因为固定间隔8秒会出现一次收包时间过长，导致收包效率降低。请教一下老师每隔8秒系统会做什么？或者是因为什么系统配置？希望老师解答一下疑惑，谢谢！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个不好说，后面有讲到延迟增大的分析思路，到时候可以分析看看</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-08 11:31:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/4d/fe/882eaf0f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>威</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，请问最后一张图下方的两个大圈圈代表的是什么意思，是代表loop吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: ring buffer</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-06 14:45:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/fa/b4/6892eabe.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_33409b</span>
  </div>
  <div class="_2_QraFYR_0">系统出口带宽被打满，导致大量请求超时</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: DDoS 😊</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-06 13:26:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/12/f9/7e6e3ac6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_04e22a</span>
  </div>
  <div class="_2_QraFYR_0">网关机器带宽被打满，查看发送机器有TCP重传</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-24 20:45:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/CV9kk5M26pdIuAxwdXvj90ewKECzdSmzO4ibP6iaLXY50hICibefmib4qGvu1wCSfXuRobFC86z7W3OcfncpV8Uevw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_25565b</span>
  </div>
  <div class="_2_QraFYR_0">老师好，netstat 执行一次 Recv-Q 和Send-Q 有值就说明堆积吗？还是用watch 看变化情况确定堆积？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-02 21:56:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/04/8a/ff94bd60.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>涛涛</span>
  </div>
  <div class="_2_QraFYR_0">接受网络包的时候，数据拷贝了好几次，所以后来有了零拷贝？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 为了性能，只在必须要拷贝的时候才去复制</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-28 14:20:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/3d/77/45e5e06d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>胡鹏</span>
  </div>
  <div class="_2_QraFYR_0">我所知道的网络问题，就是服务器被ddos攻击，小规模，可以防，，，大规模防不了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，大规模必须要专业的网络设备来抗</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-20 19:50:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/da/6a/91bd13de.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张挺</span>
  </div>
  <div class="_2_QraFYR_0">您好，请问，数据从网卡到应用程序或者应用程序到网卡，都会同时触发硬中断和软中断吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-20 13:45:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/86/fa/4bcd7365.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>玉剑冰锋</span>
  </div>
  <div class="_2_QraFYR_0">您好老师，IP包分片，一个IP包分成多个分片，是如何保证接收方收一个完整的数据包的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 根据 MTU 分片，分片里面会包含分片信息，所以接收后还可以组合起来。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-15 07:59:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/57/6e/b6795c44.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>夜空中最亮的星</span>
  </div>
  <div class="_2_QraFYR_0">新年好，给老师拜个晚年，过年的课都落下了，抓紧时间赶上来。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢，新春快乐！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-14 17:27:37</div>
  </div>
</div>
</div>
</li>
</ul>