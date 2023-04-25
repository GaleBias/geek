<audio title="34 _ Nginx：高性能的Web服务器" src="https://static001.geekbang.org/resource/audio/c2/b4/c244add2f3ad05a959d875e667c336b4.mp3" controls="controls"></audio> 
<p>经过前面几大模块的学习，你已经完全掌握了HTTP的所有知识，那么接下来请收拾一下行囊，整理一下装备，跟我一起去探索HTTP之外的广阔天地。</p><p>现在的互联网非常发达，用户越来越多，网速越来越快，HTTPS的安全加密、HTTP/2的多路复用等特性都对Web服务器提出了非常高的要求。一个好的Web服务器必须要具备稳定、快速、易扩展、易维护等特性，才能够让网站“立于不败之地”。</p><p>那么，在搭建网站的时候，应该选择什么样的服务器软件呢？</p><p>在开头的几讲里我也提到过，Web服务器就那么几款，目前市面上主流的只有两个：Apache和Nginx，两者合计占据了近90%的市场份额。</p><p>今天我要说的就是其中的Nginx，它是Web服务器的“后起之秀”，虽然比Apache小了10岁，但增长速度十分迅猛，已经达到了与Apache“平起平坐”的地位，而在“Top Million”网站中更是超过了Apache，拥有超过50%的用户（<a href="https://w3techs.com/technologies/cross/web_server/ranking">参考数据</a>）。</p><p><img src="https://static001.geekbang.org/resource/image/c5/0b/c5df0592cc8aef91ba961f7fab5a4a0b.png?wh=1222*340" alt="unpreview"></p><p>在这里必须要说一下Nginx的正确发音，它应该读成“Engine X”，但我个人感觉“X”念起来太“拗口”，还是比较倾向于读做“Engine ks”，这也与UNIX、Linux的发音一致。</p><p>作为一个Web服务器，Nginx的功能非常完善，完美支持HTTP/1、HTTPS和HTTP/2，而且还在不断进步。当前的主线版本已经发展到了1.17，正在进行HTTP/3的研发，或许一年之后就能在Nginx上跑HTTP/3了。</p><!-- [[[read_end]]] --><p>Nginx也是我个人的主要研究领域，我也写过相关的书，按理来说今天的课程应该是“手拿把攥”，但真正动笔的时候还是有些犹豫的：很多要点都已经在书里写过了，这次的专栏如果再重复相同的内容就不免有“骗稿费”的嫌疑，应该有些“不一样的东西”。</p><p>所以我决定抛开书本，换个角度，结合HTTP协议来讲Nginx，带你窥视一下HTTP处理的内幕，看看Web服务器的工作原理。</p><h2>进程池</h2><p>你也许听说过，Nginx是个<span class="orange">“轻量级”的Web服务器</span>，那么这个所谓的“轻量级”是什么意思呢？</p><p>“轻量级”是相对于“重量级”而言的。“重量级”就是指服务器进程很“重”，占用很多资源，当处理HTTP请求时会消耗大量的CPU和内存，受到这些资源的限制很难提高性能。</p><p>而Nginx作为“轻量级”的服务器，它的CPU、内存占用都非常少，同样的资源配置下就能够为更多的用户提供服务，其奥秘在于它独特的工作模式。</p><p><img src="https://static001.geekbang.org/resource/image/3e/c1/3e94fbd78ed043e88c443f6416f99dc1.png?wh=1236*997" alt=""></p><p>在Nginx之前，Web服务器的工作模式大多是“Per-Process”或者“Per-Thread”，对每一个请求使用单独的进程或者线程处理。这就存在创建进程或线程的成本，还会有进程、线程“上下文切换”的额外开销。如果请求数量很多，CPU就会在多个进程、线程之间切换时“疲于奔命”，平白地浪费了计算时间。</p><p>Nginx则完全不同，“一反惯例”地没有使用多线程，而是使用了“<strong>进程池+单线程</strong>”的工作模式。</p><p>Nginx在启动的时候会预先创建好固定数量的worker进程，在之后的运行过程中不会再fork出新进程，这就是进程池，而且可以自动把进程“绑定”到独立的CPU上，这样就完全消除了进程创建和切换的成本，能够充分利用多核CPU的计算能力。</p><p>在进程池之上，还有一个“master”进程，专门用来管理进程池。它的作用有点像是supervisor（一个用Python编写的进程管理工具），用来监控进程，自动恢复发生异常的worker，保持进程池的稳定和服务能力。</p><p>不过master进程完全是Nginx自行用C语言实现的，这就摆脱了外部的依赖，简化了Nginx的部署和配置。</p><h2>I/O多路复用</h2><p>如果你用Java、C等语言写过程序，一定很熟悉“多线程”的概念，使用多线程能够很容易实现并发处理。</p><p>但多线程也有一些缺点，除了刚才说到的“上下文切换”成本，还有编程模型复杂、数据竞争、同步等问题，写出正确、快速的多线程程序并不是一件容易的事情。</p><p>所以Nginx就选择了单线程的方式，带来的好处就是开发简单，没有互斥锁的成本，减少系统消耗。</p><p>那么，疑问也就产生了：为什么单线程的Nginx，处理能力却能够超越其他多线程的服务器呢？</p><p>这要归功于Nginx利用了Linux内核里的一件“神兵利器”，<strong>I/O多路复用接口</strong>，“大名鼎鼎”的epoll。</p><p>“多路复用”这个词我们已经在之前的HTTP/2、HTTP/3里遇到过好几次，如果你理解了那里的“多路复用”，那么面对Nginx的epoll“多路复用”也就好办了。</p><p>Web服务器从根本上来说是“I/O密集型”而不是“CPU密集型”，处理能力的关键在于网络收发而不是CPU计算（这里暂时不考虑HTTPS的加解密），而网络I/O会因为各式各样的原因不得不等待，比如数据还没到达、对端没有响应、缓冲区满发不出去等等。</p><p>这种情形就有点像是HTTP里的“队头阻塞”。对于一般的单线程来说CPU就会“停下来”，造成浪费。而多线程的解决思路有点类似“并发连接”，虽然有的线程可能阻塞，但由于多个线程并行，总体上看阻塞的情况就不会太严重了。</p><p>Nginx里使用的epoll，就好像是HTTP/2里的“多路复用”技术，它把多个HTTP请求处理打散成碎片，都“复用”到一个单线程里，不按照先来后到的顺序处理，而是只当连接上真正可读、可写的时候才处理，如果可能发生阻塞就立刻切换出去，处理其他的请求。</p><p>通过这种方式，Nginx就完全消除了I/O阻塞，把CPU利用得“满满当当”，又因为网络收发并不会消耗太多CPU计算能力，也不需要切换进程、线程，所以整体的CPU负载是相当低的。</p><p>这里我画了一张Nginx“I/O多路复用”的示意图，你可以看到，它的形式与HTTP/2的流非常相似，每个请求处理单独来看是分散、阻塞的，但因为都复用到了一个线程里，所以资源的利用率非常高。</p><p><img src="https://static001.geekbang.org/resource/image/4c/59/4c6832cdce34133c9ed89237fb9d5059.png?wh=2211*954" alt=""></p><p>epoll还有一个特点，大量的连接管理工作都是在操作系统内核里做的，这就减轻了应用程序的负担，所以Nginx可以为每个连接只分配很小的内存维护状态，即使有几万、几十万的并发连接也只会消耗几百M内存，而其他的Web服务器这个时候早就“Memory not enough”了。</p><h2>多阶段处理</h2><p>有了“进程池”和“I/O多路复用”，Nginx是如何处理HTTP请求的呢？</p><p>Nginx在内部也采用的是“<strong>化整为零</strong>”的思路，把整个Web服务器分解成了多个“功能模块”，就好像是乐高积木，可以在配置文件里任意拼接搭建，从而实现了高度的灵活性和扩展性。</p><p>Nginx的HTTP处理有四大类模块：</p><ol>
<li>handler模块：直接处理HTTP请求；</li>
<li>filter模块：不直接处理请求，而是加工过滤响应报文；</li>
<li>upstream模块：实现反向代理功能，转发请求到其他服务器；</li>
<li>balance模块：实现反向代理时的负载均衡算法。</li>
</ol><p>因为upstream模块和balance模块实现的是代理功能，Nginx作为“中间人”，运行机制比较复杂，所以我今天只讲handler模块和filter模块。</p><p>不知道你有没有了解过“设计模式”这方面的知识，其中有一个非常有用的模式叫做“<strong>职责链</strong>”。它就好像是工厂里的流水线，原料从一头流入，线上有许多工人会进行各种加工处理，最后从另一头出来的就是完整的产品。</p><p>Nginx里的handler模块和filter模块就是按照“职责链”模式设计和组织的，HTTP请求报文就是“原材料”，各种模块就是工厂里的工人，走完模块构成的“流水线”，出来的就是处理完成的响应报文。</p><p>下面的这张图显示了Nginx的“流水线”，在Nginx里的术语叫“阶段式处理”（Phases），一共有11个阶段，每个阶段里又有许多各司其职的模块。</p><p><img src="https://static001.geekbang.org/resource/image/41/30/41318c867fda8a536d0e3db6f9987030.png?wh=1224*843" alt=""></p><p>我简单列几个与我们的课程相关的模块吧：</p><ul>
<li>charset模块实现了字符集编码转换；（<a href="https://time.geekbang.org/column/article/104024">第15讲</a>）</li>
<li>chunked模块实现了响应数据的分块传输；（<a href="https://time.geekbang.org/column/article/104456">第16讲</a>）</li>
<li>range模块实现了范围请求，只返回数据的一部分；（<a href="https://time.geekbang.org/column/article/104456">第16讲</a>）</li>
<li>rewrite模块实现了重定向和跳转，还可以使用内置变量自定义跳转的URI；（<a href="https://time.geekbang.org/column/article/105614">第18讲</a>）</li>
<li>not_modified模块检查头字段“if-Modified-Since”和“If-None-Match”，处理条件请求；（<a href="https://time.geekbang.org/column/article/106804">第20讲</a>）</li>
<li>realip模块处理“X-Real-IP”“X-Forwarded-For”等字段，获取客户端的真实IP地址；（<a href="https://time.geekbang.org/column/article/107577">第21讲</a>）</li>
<li>ssl模块实现了SSL/TLS协议支持，读取磁盘上的证书和私钥，实现TLS握手和SNI、ALPN等扩展功能；（<a href="https://time.geekbang.org/column/article/108643">安全篇</a>）</li>
<li>http_v2模块实现了完整的HTTP/2协议。（<a href="https://time.geekbang.org/column/article/112036">飞翔篇</a>）</li>
</ul><p>在这张图里，你还可以看到limit_conn、limit_req、access、log等其他模块，它们实现的是限流限速、访问控制、日志等功能，不在HTTP协议规定之内，但对于运行在现实世界的Web服务器却是必备的。</p><p>如果你有C语言基础，感兴趣的话可以下载Nginx的源码，在代码级别仔细看看HTTP的处理过程。</p><h2>小结</h2><ol>
<li><span class="orange">Nginx是一个高性能的Web服务器，它非常的轻量级，消耗的CPU、内存很少；</span></li>
<li><span class="orange">Nginx采用“master/workers”进程池架构，不使用多线程，消除了进程、线程切换的成本；</span></li>
<li><span class="orange">Nginx基于epoll实现了“I/O多路复用”，不会阻塞，所以性能很高；</span></li>
<li><span class="orange">Nginx使用了“职责链”模式，多个模块分工合作，自由组合，以流水线的方式处理HTTP请求。</span></li>
</ol><h2>课下作业</h2><ol>
<li>你是怎么理解进程、线程上下文切换时的成本的，为什么Nginx要尽量避免？</li>
<li>试着自己描述一下Nginx用进程、epoll、模块流水线处理HTTP请求的过程。</li>
</ol><p>欢迎你把自己的学习体会写在留言区，与我和其他同学一起讨论。如果你觉得有所收获，也欢迎把文章分享给你的朋友。</p><p><img src="https://static001.geekbang.org/resource/image/4c/3d/4c7bceb80a8027389705e9d6ec9eb43d.png?wh=1769*3085" alt="unpreview"></p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4d/fd/0aa0e39f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>许童童</span>
  </div>
  <div class="_2_QraFYR_0">你是怎么理解进程、线程上下文切换时的成本的，为什么 Nginx 要尽量避免？<br>当从一个任务切换到另一个任务，当前任务的上下文，如堆栈，指令指针等都要保存起来，以便下次任务时恢复，然后再把另一个任务的堆栈加载进来，如果有大量的上下文切换，就会影响性能。<br><br>试着自己描述一下 Nginx 用进程、epoll、模块流水线处理 HTTP 请求的过程。<br>Nginx 启动进程，一个master，多个worker，创建epoll，监听端口，多路复用来管理http请求，http请求到达worker内部，通过模块流水线处理，最后返回http响应。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: √</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-14 14:42:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/81/4e/d71092f4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>夏目</span>
  </div>
  <div class="_2_QraFYR_0">好像高性能的服务都是这样玩的，nginx这个架构类似于netty中的多线程reactor模式，redis则是单线程reactor</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: nginx也是单线程的，和redis一样自己封装了epoll。单线程的好处是没有race condition，处理简单。<br><br>nginx比redis高明的一点是多进程，提高了稳定性和并发能力。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-10 16:48:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/9b/a8/6a391c66.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Leon📷</span>
  </div>
  <div class="_2_QraFYR_0">一个线程的时间片没用完就系统调用被系统调度切换出去，浪费了剩余的时间片，nginx通过epoll和注册回调，和非阻塞io自己在用户态主动切换上下文，充分利用了系统分配给进程或者线程的时间片，所以对系统资源利用很充分</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-14 07:11:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/9d/a4/e481ae48.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lesserror</span>
  </div>
  <div class="_2_QraFYR_0">老师，以下问题，麻烦回答一下，谢谢：<br><br>1. 把进程“绑定”到独立的 CPU 上。意思是一个CPU专门负责管理进程嘛？<br><br>2. 不过 master 进程完全是 Nginx 自行用 C 语言实现的，这就摆脱了外部的依赖，简化了 Nginx 的部署和配置。这句话没理解。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.unix&#47;linux有个特别的功能，可以让进程“绑定”在一个cpu上运行，不会被操作系统调度到其他cpu上跑，这样就减少了切换的成本，提高运行效率。不是管理进程的意思。配置指令是“worker_cpu_affinity”。<br><br>2.在unix上有很多服务管理程序，比如systemd、supervisor，可以实现进程监控、自动重启等。而Nginx的master进程实现了同样的功能，就不需要这样的外部程序来管理进程，保持服务的稳定性。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-26 15:31:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/75/00/618b20da.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>徐海浪</span>
  </div>
  <div class="_2_QraFYR_0">多线程就好比一条流水线有多个机械手，把一件事情中途交给其他线程处理，要交接处理中间状态信息。<br>单进程就好比一条流水线只有一个机械手，切换时间片时暂停状态就可以，不用交接信息，减少无用功，所以效率高。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-15 18:46:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/bd/b0/8b808d33.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>fakership</span>
  </div>
  <div class="_2_QraFYR_0">老师，有个问题咨询下<br>虽然nginx是使用了epoll做了io的多路复用，但对于队头阻塞的话感觉并没有帮助啊，因为还是要等io事件回调后发送http响应报文，所以还是阻塞了下一个请求。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，但这完全是两个不相关的事情。<br><br>队头阻塞是http&#47;1固有的问题，无论是什么web服务器都无法解决，是对单个客户端而言的。<br><br>而Nginx的epoll则是解决了多客户端并发请求的问题，避免一个客户端阻塞其他客户端的处理，可以支持海量客户端访问服务器。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-16 23:21:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/79/4b/740f91ca.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>-W.LI-</span>
  </div>
  <div class="_2_QraFYR_0">老师好!我打算学习nginx，有适合初学者的书推荐么?Java工程师，c全忘了。<br>线程切换开销:线程切换需要进行系统调用。需要从用户态-&gt;内核态-&gt;用户态。上下文切换，需要保存寄存器中的信息，以便于完成系统调用后还原现场。会多跑很多指令，出入栈会比寄存器慢很多。相对来说开销就很大了。<br>nginx和redis一样采用单线程模型。是因为cpu计算不可能是它们瓶颈(所以有些耗cpu资源高的计算不适合放在nginx上做会导致响应时间变长)?进程池+单线程是指，每个worker进程都是单线程是么?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.Nginx的内容很多，看你想学哪方面了。如果是单纯的运维操作网上的资料有很多，如果是想学Nginx开发和源码就看《Nginx完全开发指南》吧。<br><br>2.说的很对，看Nginx源码可以学到很多高性能编程的技巧。<br><br>3.Nginx里也可以使用多线程，但需要“魔改”。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-14 09:42:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/a4/7e/963c037c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Aaron</span>
  </div>
  <div class="_2_QraFYR_0">对『进程池 + 单线程』的模式还是不太透彻。<br><br>我理解，『单线程』指的是所有 HTTP 请求放在同一个线程里通过『I&#47;O 多路复用』的技术处理，实际就是高度集中（无阻塞）地占用了 CPU（核心）地运算能力。<br><br>那么，既然请求是单线程的，那进程池地作用又是什么呢？如果是多进程的，不就又回到进程间上下文切换的消耗问题了吗？<br><br>另，Nginx 通过 cpu affinity 将进程绑定到 CPU，假设是单 CPU，将三个 worker 进程绑定到同一个物理 CPU 地意义又在哪呢？<br><br>个人认为效率最高的方式，是按照 CPU 的核心数量创建一个『线程池』，将所有请求分配到『线程池』内不同的线程，这样在『I&#47;O 多路复用』的加持下能跑满 CPU 的性能。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.单线程理解的很对。进程池里的每个进程都是独立的，崩溃不会影响整体服务，如果是多线程，那么线程崩溃进程也就完蛋了。<br><br>2.多进程分散运行在多个cpu上，彼此不干扰，就不会出现进程上下文切换。<br><br>3.cpu affinity 是可选的，对于单cpu就没有开启的必要，反而会增加进程切换的成本。<br><br>4.刚才说，单进程多线程的缺点就是不够稳定，一个线程出问题，整个进程都受影响。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-01 18:12:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/64/9b/d1ab239e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>J.Smile</span>
  </div>
  <div class="_2_QraFYR_0">说一下http2和nginx的多路复用区别和联系：<br>http2的多路复用：多个请求复用同一个连接并行传输数据，且每个请求抽象为流传输的对象为帧序列。<br>nginx的IO多路复用：将多个线程的请求打散，汇入同一个线程中传输，epoll监听到事件通道可读或者可写的时候取出或者写入数据，所以nginx的IO多路复用是基于linux内核epoll实现的一种事件监听机制，是NIO非阻塞IO。<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 说的很好。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-20 18:46:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/9b/a8/6a391c66.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Leon📷</span>
  </div>
  <div class="_2_QraFYR_0">切换cpu需要保存线程的上下文，然后再切回去，这是开销</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: √</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-14 07:10:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/WtHCCMoLJ2DvzqQwPYZyj2RlN7eibTLMHDMTSO4xIKjfKR1Eh9L98AMkkZY7FmegWyGLahRQJ5ibPzeeFtfpeSow/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>脱缰的野马__</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，tomcat不主流吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: tomcat应该算是java容器吧，主要是实现业务，不是专门的web服务器。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-20 19:11:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/85/49/585c69c4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>皮特尔</span>
  </div>
  <div class="_2_QraFYR_0">Nginx这种异步处理方式叫“协程”吧？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不是。<br><br>Nginx是用纯C开发的，里面没有协程的概念，它内部用的是epoll事件机制，reactor并发模式，有ready事件就回调。<br><br>OpenResty把lua的协程和epoll事件机制结合在了一起，但两者还是不能混为一谈。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-09 14:17:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/35/51/c616f95a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>阿锋</span>
  </div>
  <div class="_2_QraFYR_0">缓存服务器，是属于正向代理还是反向代理，还是根据情况而定。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 正向代理和反向代理是根据它所在的位置来定义的，靠近客户端就是正向，靠近服务器就是反向。<br><br>代理与缓存是不相关的，代理可以没有缓存功能。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-14 16:17:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTIBrZt8rUzS24vibOdKH8icdxVMHLSNgRaYovGxbuEqJILacAicyLPzLgXkbhPhowibv6plHkDNDywxdA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>zyd-githuber</span>
  </div>
  <div class="_2_QraFYR_0">感觉和nodejs的单线程机制非常像</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Nginx可是2002年就开始开发了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-09 09:15:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/63/1b/83ac7733.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>忧天小鸡</span>
  </div>
  <div class="_2_QraFYR_0">这里说的nx的epoll是指模仿epoll的交互逻辑，还是指从epoll的base上做了对tcp的改装？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Nginx调用操作系统的epoll接口，来处理tcp事件，本质上epoll和tcp没有直接关系，但tcp会有读写事件，就可以利用epoll来处理。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-30 19:23:07</div>
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
  <div class="_2_QraFYR_0">线程上下文的切换消耗感觉主要是用户态和内核态不断切换。也就是堆栈，指令指针之类的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对，CPU要保存当前状态，再恢复原来的状态，但当线程多的时候，累积的成本就很高了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-27 11:42:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/79/fa/48b481fe.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>zero</span>
  </div>
  <div class="_2_QraFYR_0">老师，您好，我想写博客，我写的博客里面能盗一下您的图么（您的图做的太直观了一看就懂了），我会著名图片的出处😇😇</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个要联系极客时间吧，版权在他们那里。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-29 12:07:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/10/de/dbf2abde.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>萤火之森</span>
  </div>
  <div class="_2_QraFYR_0">多进程 对缓存管理的数据竞争如何处理？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感兴趣可以看Nginx的源码，这个问题太大了，不太好细说。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-03 20:57:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2b/96/1f/a4e3f3a5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>三千世界</span>
  </div>
  <div class="_2_QraFYR_0">老师我想问一下，nginx为什么要设计让多个worker进程竞争accpet，这样导致 惊群 问题，还要加锁来解决，反而造成了性能下降。<br>所以，为什么不让master通过epoll监听有连接可以accept，通过调度，找一个不怎么忙的worker，然后通过管道通知这个worker呢，这样就不会出现惊群问题了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: accept mutex设计的目的是多worker进程之间负载均衡，避免有的worker处理的连接太多。<br><br>初衷是好的，在NGINX初期也确实很有效果，但到了现在，并发越来越多，它的锁成本就显得高了。<br><br>目前NGINX不推荐使用accept mutex，而是改用Linux系统内核的reuseport来实现负载均衡。<br><br>你说的master监听的方式是很传统的做法，效率更低。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-29 18:11:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/4faqHgQSawd4VzAtSv0IWDddm9NucYWibRpxejWPH5RUO310qv8pAFmc0rh0Qu6QiahlTutGZpia8VaqP2w6icybiag/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>爱编程的运维</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，nginx采用IO多路复用技术，使用单线程处理多个IO流数据流<br>是不是也可以多线程+IO多路复用技术？多个线程处理多个IO数据流<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 当然可以，像envoy，还有NGINX Unit都是多线程+io多路复用。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-05 17:49:17</div>
  </div>
</div>
</div>
</li>
</ul>