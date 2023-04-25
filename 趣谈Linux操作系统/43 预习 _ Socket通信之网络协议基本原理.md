<audio title="43 预习 _ Socket通信之网络协议基本原理" src="https://static001.geekbang.org/resource/audio/f7/92/f72ff7d906f3210b035248fd2b899892.mp3" controls="controls"></audio> 
<p>上一节我们讲的进程间通信，其实是通过内核的数据结构完成的，主要用于在一台Linux上两个进程之间的通信。但是，一旦超出一台机器的范畴，我们就需要一种跨机器的通信机制。</p><p>一台机器将自己想要表达的内容，按照某种约定好的格式发送出去，当另外一台机器收到这些信息后，也能够按照约定好的格式解析出来，从而准确、可靠地获得发送方想要表达的内容。这种约定好的格式就是<strong>网络协议</strong>（Networking Protocol）。</p><p>我们将要讲的Socket通信以及相关的系统调用、内核机制，都是基于网络协议的，如果不了解网络协议的机制，解析Socket的过程中，你就会迷失方向，因此这一节，我们有必要做一个预习，先来大致讲一下网络协议的基本原理。</p><h2>网络为什么要分层？</h2><p>我们这里先构建一个相对简单的场景，之后几节内容，我们都要基于这个场景进行讲解。</p><p>我们假设这里就涉及三台机器。Linux服务器A和Linux服务器B处于不同的网段，通过中间的Linux服务器作为路由器进行转发。</p><p><img src="https://static001.geekbang.org/resource/image/f6/0e/f6982eb85dc66bd04200474efb3a050e.png?wh=3650*2255" alt=""></p><p>说到网络协议，我们还需要简要介绍一下两种网络协议模型，一种是<strong>OSI的标准七层模型</strong>，一种是<strong>业界标准的TCP/IP模型</strong>。它们的对应关系如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/92/0e/92f8e85f7b9a9f764c71081b56286e0e.png?wh=1783*1843" alt=""></p><p>为什么网络要分层呢？因为网络环境过于复杂，不是一个能够集中控制的体系。全球数以亿记的服务器和设备各有各的体系，但是都可以通过同一套网络协议栈通过切分成多个层次和组合，来满足不同服务器和设备的通信需求。</p><!-- [[[read_end]]] --><p>我们这里简单介绍一下网络协议的几个层次。</p><p>我们从哪一个层次开始呢？从第三层，网络层开始，因为这一层有我们熟悉的IP地址。也因此，这一层我们也叫IP层。</p><p>我们通常看到的IP地址都是这个样子的：192.168.1.100/24。斜杠前面是IP地址，这个地址被点分隔为四个部分，每个部分8位，总共是32位。斜线后面24的意思是，32位中，前24位是网络号，后8位是主机号。</p><p>为什么要这样分呢？我们可以想象，虽然全世界组成一张大的互联网，美国的网站你也能够访问的，但是这个网络不是一整个的。你们小区有一个网络，你们公司也有一个网络，联通、移动、电信运营商也各有各的网络，所以一个大网络是被分成个小的网络。</p><p>那如何区分这些网络呢？这就是网络号的概念。一个网络里面会有多个设备，这些设备的网络号一样，主机号不一样。不信你可以观察一下你家里的手机、电视、电脑。</p><p>连接到网络上的每一个设备都至少有一个IP地址，用于定位这个设备。无论是近在咫尺的你旁边同学的电脑，还是远在天边的电商网站，都可以通过IP地址进行定位。因此，<strong>IP地址类似互联网上的邮寄地址，是有全局定位功能的</strong>。</p><p>就算你要访问美国的一个地址，也可以从你身边的网络出发，通过不断的打听道儿，经过多个网络，最终到达目的地址，和快递员送包裹的过程差不多。打听道儿的协议也在第三层，称为路由协议（Routing protocol），将网络包从一个网络转发给另一个网络的设备称为路由器。</p><p>路由器和路由协议十分复杂，我们这里就不详细讲解了，感兴趣可以去看我写的另一个专栏“趣谈网络协议”里的<a href="https://time.geekbang.org/column/article/8729">相关文章</a>。</p><p>总而言之，第三层干的事情，就是网络包从一个起始的IP地址，沿着路由协议指的道儿，经过多个网络，通过多次路由器转发，到达目标IP地址。</p><p>从第三层，我们往下看，第二层是数据链路层。有时候我们简称为二层或者MAC层。所谓MAC，就是每个网卡都有的唯一的硬件地址（不绝对唯一，相对大概率唯一即可，类比<a href="https://zh.wikipedia.org/wiki/%E9%80%9A%E7%94%A8%E5%94%AF%E4%B8%80%E8%AF%86%E5%88%AB%E7%A0%81">UUID</a>）。这虽然也是一个地址，但是这个地址是没有全局定位功能的。</p><p>就像给你送外卖的小哥，不可能根据手机尾号找到你家，但是手机尾号有本地定位功能的，只不过这个定位主要靠“吼”。外卖小哥到了你的楼层就开始大喊：“尾号xxxx的，你外卖到了！”</p><p>MAC地址的定位功能局限在一个网络里面，也即同一个网络号下的IP地址之间，可以通过MAC进行定位和通信。从IP地址获取MAC地址要通过ARP协议，是通过在本地发送广播包，也就是“吼”，获得的MAC地址。</p><p>由于同一个网络内的机器数量有限，通过MAC地址的好处就是简单。匹配上MAC地址就接收，匹配不上就不接收，没有什么所谓路由协议这样复杂的协议。当然坏处就是，MAC地址的作用范围不能出本地网络，所以一旦跨网络通信，虽然IP地址保持不变，但是MAC地址每经过一个路由器就要换一次。</p><p>我们看前面的图。服务器A发送网络包给服务器B，原IP地址始终是192.168.1.100，目标IP地址始终是192.168.2.100，但是在网络1里面，原MAC地址是MAC1，目标MAC地址是路由器的MAC2，路由器转发之后，原MAC地址是路由器的MAC3，目标MAC地址是MAC4。</p><p>所以第二层干的事情，就是网络包在本地网络中的服务器之间定位及通信的机制。</p><p>我们再往下看，第一层，物理层，这一层就是物理设备。例如连着电脑的网线，我们能连上的WiFi，这一层我们不打算进行分析。</p><p>从第三层往上看，第四层是传输层，这里面有两个著名的协议TCP和UDP。尤其是TCP，更是广泛使用，在IP层的代码逻辑中，仅仅负责数据从一个IP地址发送给另一个IP地址，丢包、乱序、重传、拥塞，这些IP层都不管。处理这些问题的代码逻辑写在了传输层的TCP协议里面。</p><p>我们常称，TCP是可靠传输协议，也是难为它了。因为从第一层到第三层都不可靠，网络包说丢就丢，是TCP这一层通过各种编号、重传等机制，让本来不可靠的网络对于更上层来讲，变得“看起来”可靠。哪有什么应用层岁月静好，只不过TCP层帮你负重前行。</p><p>传输层再往上就是应用层，例如咱们在浏览器里面输入的HTTP，Java服务端写的Servlet，都是这一层的。</p><p>二层到四层都是在Linux内核里面处理的，应用层例如浏览器、Nginx、Tomcat都是用户态的。内核里面对于网络包的处理是不区分应用的。</p><p>从四层再往上，就需要区分网络包发给哪个应用。在传输层的TCP和UDP协议里面，都有端口的概念，不同的应用监听不同的端口。例如，服务端Nginx监听80、Tomcat监听8080；再如客户端浏览器监听一个随机端口，FTP客户端监听另外一个随机端口。</p><p>应用层和内核互通的机制，就是通过Socket系统调用。所以经常有人会问，Socket属于哪一层，其实它哪一层都不属于，它属于操作系统的概念，而非网络协议分层的概念。只不过操作系统选择对于网络协议的实现模式是，二到四层的处理代码在内核里面，七层的处理代码让应用自己去做，两者需要跨内核态和用户态通信，就需要一个系统调用完成这个衔接，这就是Socket。</p><h2>发送数据包</h2><p>网络分完层之后，对于数据包的发送，就是层层封装的过程。</p><p>就像下面的图中展示的一样，在Linux服务器B上部署的服务端Nginx和Tomcat，都是通过Socket监听80和8080端口。这个时候，内核的数据结构就知道了。如果遇到发送到这两个端口的，就发送给这两个进程。</p><p>在Linux服务器A上的客户端，打开一个Firefox连接Ngnix。也是通过Socket，客户端会被分配一个随机端口12345。同理，打开一个Chrome连接Tomcat，同样通过Socket分配随机端口12346。</p><p><img src="https://static001.geekbang.org/resource/image/98/28/98a4496fff94eb02d1b1b8ae88f8dc28.jpeg?wh=4876*2212" alt=""></p><p>在客户端浏览器，我们将请求封装为HTTP协议，通过Socket发送到内核。内核的网络协议栈里面，在TCP层创建用于维护连接、序列号、重传、拥塞控制的数据结构，将HTTP包加上TCP头，发送给IP层，IP层加上IP头，发送给MAC层，MAC层加上MAC头，从硬件网卡发出去。</p><p>网络包会先到达网络1的交换机。我们常称交换机为二层设备，这是因为，交换机只会处理到第二层，然后它会将网络包的MAC头拿下来，发现目标MAC是在自己右面的网口，于是就从这个网口发出去。</p><p>网络包会到达中间的Linux路由器，它左面的网卡会收到网络包，发现MAC地址匹配，就交给IP层，在IP层根据IP头中的信息，在路由表中查找。下一跳在哪里，应该从哪个网口发出去？在这个例子中，最终会从右面的网口发出去。我们常把路由器称为三层设备，因为它只会处理到第三层。</p><p>从路由器右面的网口发出去的包会到网络2的交换机，还是会经历一次二层的处理，转发到交换机右面的网口。</p><p>最终网络包会被转发到Linux服务器B，它发现MAC地址匹配，就将MAC头取下来，交给上一层。IP层发现IP地址匹配，将IP头取下来，交给上一层。TCP层会根据TCP头中的序列号等信息，发现它是一个正确的网络包，就会将网络包缓存起来，等待应用层的读取。</p><p>应用层通过Socket监听某个端口，因而读取的时候，内核会根据TCP头中的端口号，将网络包发给相应的应用。</p><p>HTTP层的头和正文，是应用层来解析的。通过解析，应用层知道了客户端的请求，例如购买一个商品，还是请求一个网页。当应用层处理完HTTP的请求，会将结果仍然封装为HTTP的网络包，通过Socket接口，发送给内核。</p><p>内核会经过层层封装，从物理网口发送出去，经过网络2的交换机，Linux路由器到达网络1，经过网络1的交换机，到达Linux服务器A。在Linux服务器A上，经过层层解封装，通过socket接口，根据客户端的随机端口号，发送给客户端的应用程序，浏览器。于是浏览器就能够显示出一个绚丽多彩的页面了。</p><p>即便在如此简单的一个环境中，网络包的发送过程，竟然如此的复杂。不过这一章后面，我们还是会层层剖析每一层做的事情。</p><h2>总结时刻</h2><p><span class="orange">网络协议是一个大话题，如果你想了解网络协议的方方面面，欢迎你订阅我写的另一个专栏“趣谈网络协议”。</span>这个专栏重点解析在这个网络通信过程中，发送端和接收端的操作系统都做了哪些事情，对于中间通路上的复杂的网络通信逻辑没有做深入解析。</p><p><span class="orange">如果只是为了掌握这一章的内容，这一节我们讲的网络协议的七个层次，你不必每一层的每一个协议都很清楚，只要记住TCP/UDP-&gt;IPv4-&gt;ARP这一条链就可以了</span>，因为后面我们的分析都是重点分析这条链。</p><p>另外，前面那个简单的拓扑图中，网络包的封装、转发、解封装的过程，建议你多看几遍，了熟于心，因为接下来，我们就能从代码层面，看到这个过程。到时候，对应起来，你就比较容易理解。</p><p>了解了Socket的基本原理，下一篇文章，我们就来看一看在Linux操作系统里面Socket系统调用的接口是什么样的。</p><p><img src="https://static001.geekbang.org/resource/image/8c/37/8c0a95fa07a8b9a1abfd394479bdd637.jpg?wh=1110*659" alt=""></p>
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
  <div class="_2_QraFYR_0">老师：问个问题。我们家庭办理的宽带，都是运营商在哪一层把带宽给限制的呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 估计刚出你们家的入口就限制了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-05 15:17:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/vTku9cFYPh2T8DSImQoPRLxgSibcVgCRYqMcEYibexxLkfn9IKhUSAasZ7QoB72SDWym31niah2y00ibRWdHibibib1wQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Regina</span>
  </div>
  <div class="_2_QraFYR_0">为什么握手使用的socket与链接的socket不一样，有什么原因吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 需要不同的数据结构，保存不同的状态</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-22 17:15:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/42/fc/c89243d8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>侯代烨</span>
  </div>
  <div class="_2_QraFYR_0">讲的很详细，从封包到解包过程很详细</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-22 10:24:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/87/3a/7ef24065.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>严志波</span>
  </div>
  <div class="_2_QraFYR_0">为啥要有交换机呢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-14 19:04:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/fd/81/1864f266.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>石将从</span>
  </div>
  <div class="_2_QraFYR_0">这个一下子二层，一下子四层的，我都不知道是指哪一层了，也不知道从上往下数，还是从下往上数，对照了半天，尴尬</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-24 22:37:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/23/66/413c0bb5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>LDxy</span>
  </div>
  <div class="_2_QraFYR_0">为什么称为协议栈呢？这和栈这种数据结构有何关系？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 全栈工程师</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-06 17:45:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1c/f7/62/947004d0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>www</span>
  </div>
  <div class="_2_QraFYR_0">站得高，讲得真清楚，之前一直不太理解Socket的定位</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-06 10:59:48</div>
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
  <div class="_2_QraFYR_0">同一个网络中，网络号一样，主机号不一样，这样能通过ip地址找到设备了，为什么还需要mac地址去找</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-20 08:10:22</div>
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
  <div class="_2_QraFYR_0">ARP请求只局限在相同网段的网络。那假如需要从服务器A ping包 服务器B (图1）。不需要ARP请求嘛？或者是怎么完成连接？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-22 10:04:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/08/32/8137ce04.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>李很奈斯</span>
  </div>
  <div class="_2_QraFYR_0">爱了(⑉°з°)-♡，讲的非常不错</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-12 13:23:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/e7/de/c62cba75.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>无为</span>
  </div>
  <div class="_2_QraFYR_0">ICMP协议应该是在网络层。并不是传输层</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-19 22:40:04</div>
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
  <div class="_2_QraFYR_0">一个数据包里是有两个mac吗，一个是网关的，一个是局域网中pc的吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-14 07:30:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/ac/c8/4b1c0d40.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>勤劳的小胖子-libo</span>
  </div>
  <div class="_2_QraFYR_0">”将一个网络包从一个网络转发到另一个网络的设备称为路由器“。<br>这里面的网络不仅包括eth0&#47;1之类的interface，也包括自定义的interface,比如docker自动生成的docker0.就算一台机器只有一个物理网卡，也能当作路由器，是吧。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 只要一个网卡，没法路由呀。从一个网络转发到另一个网络的意思是从192.168.1.100&#47;24网络到192.168.2.100&#47;24网络，类似这种</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-10 15:14:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/b4/f6/735673f7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>W.jyao</span>
  </div>
  <div class="_2_QraFYR_0">看到留言的问题了，这不就是arp干的事</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-05 12:10:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/e9/0b/1171ac71.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>WL</span>
  </div>
  <div class="_2_QraFYR_0">老师我想问一下在发送数据包的时候, Linux服务A是怎么拿到linux服务器B的mac地址的, Linux服务器B的mac地址是一开始就加上去的吗?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: A不需要知道B的mac地址。只需要得到网关的mac地址</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-05 08:40:58</div>
  </div>
</div>
</div>
</li>
</ul>