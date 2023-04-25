<audio title="16｜网络爬虫： 一次HTTP请求的魔幻旅途" src="https://static001.geekbang.org/resource/audio/df/85/dfb23f68e4297b1b56bbc638317f0885.mp3" controls="controls"></audio> 
<p>你好，我是郑建勋。</p><p>上节课，我讲解了开发一个Go项目需要遵守的编程规范。接下来我们就要开始正式书写爬虫实战项目了。</p><p>这个项目的核心是通过HTTP协议与目标网站通信，然后发送请求并获取目标网站的对应资源。在下面两节课，我会带着你从一个最简单的HTTP请求入手，一步步理解请求背后发生的故事。</p><p>在课程正式开始之前，我想先和你分享一段话：“经验用来对待特殊场景，方法论用来处理通用场景，没有经验可能会慢一些，没有方法论可能寸步难行。”</p><p>而为了更好地理解爬虫项目可能遇到的难题，并且在解决网络问题时有方法论的支撑，我们需要掌握网络分层协议与层层封装的流转过程、网络数据包的路由过程，操作系统收发包的处理过程。另外，还要熟悉HTTP协议以及Go标准库对HTTP协议的巧妙封装。</p><h2>最简单的HTTP服务器与请求</h2><p>为了方便开发者使用，Go语言对网络库和HTTP库的封装可以说是费尽心力。在平时，三行核心代码就能够写出一个HTTP的服务器或是HTTP请求，但其实Go标准库内部进行了大量处理。</p><p>下面这个例子是借助Go HTTP标准库书写的一个最简单的HTTP服务器。</p><pre><code class="language-plain">package main

import (
	"fmt"
	"net/http"
)

func hello(w http.ResponseWriter, _ *http.Request) {
	fmt.Fprintf(w, "Hello")
}

func main() {
    // 访问路由到hello函数
	http.HandleFunc("/", hello)
    // 监听本地8080端口
	http.ListenAndServe("0.0.0.0:8080", nil)
}
</code></pre><!-- [[[read_end]]] --><p>下面则是一个最简单的HTTP请求服务，它会访问百度的网址并打印内容。</p><pre><code class="language-plain">package main

import (
	"fmt"
	"io/ioutil"
	"net/http"
)

func main() {
	// http请求
	resp, err := http.Get("http://www.baidu.com")
	if err != nil {
		fmt.Println(err)
		return
	}
	// 获取返回的数据
	content, err := ioutil.ReadAll(resp.Body)
	if err != nil {
		fmt.Println(err)
		return
	}
	// 打印返回的数据
	fmt.Println(string(content))
}
</code></pre><h2></h2><h2>分层网络模型</h2><p>在上面这个例子中，http.Get函数想要使用HTTP协议的GET方法获取目标网站的数据。这背后的原理是什么呢？让我们从经典的分层网络模型说起。</p><p>经典的网络模型有两种：OSI 7层网络模型和TCP/IP 4层网络模型。如下图所示，它们都是分层结构，每一层具有不同的功能。数据包会从上到下逐层传递，最终，数据包会从一个系统被传输到另一个系统。</p><p><img src="https://static001.geekbang.org/resource/image/ce/ea/ce4f050934afda3a33e69077a5513bea.jpg?wh=1920x1126" alt="图片" title="OSI网络模型 vs TCP/IP网络模型"></p><p>OSI 7层网络模型是描述两个系统进行网络通信的概念框架，它分为了应用层、表示层、会话层、传输层、网络层、数据链路层和物理层共7个层次。但是因为OSI 7层模型分层太多，而且当时大多数人认为它复杂、低效、（在某种程度上）无法实现，所以OSI 7层模型只是作为理论模型存在。不过，它对于新手理解网络原理仍然非常有用。</p><p>而TCP/IP 4层网络模型是当前的国际标准，分为了应用层、传输层、网络层、网络接入层(Network Access Layer)。有时我们还会听到5层网络模型这种说法，其实那只是把TCP/IP 4层网络模型最底层的网络接入层又拆分了一下（拆分为了链路层与物理层）而已。另外我们还将TLS协议（代表性的是HTTPS协议）认为是特殊的在传输层和应用层之间的协议，下面我分别介绍一下这几个层次。</p><h3>应用层</h3><p>在 TCP/IP 模型中，应用程序层依赖传输层协议来建立和管理主机到主机的数据传输，应用程序层也是与用户交互的地方。应用层不是真正的应用程序，但是它提供了交互的接口。</p><p>例如，我们每天都会使用浏览器查看网页上的信息，但是浏览器是对服务器发过来的HTML和js等文件进行了渲染展示，浏览器内部使用的与服务器交互的HTTP接口就处于应用程序层。其他上层协议如DNS、SSH、SMTP也都位于该层。</p><p>再比如，我们前面发送的HTTP请求，它访问的是域名<a href="https://www.baidu.xn--com-o16s">www.baidu.com，</a>但是我们最终需要将当前的域名转换为服务器的IP地址，这个过程就是DNS协议在发挥作用。</p><h3>传输层</h3><p>传输层为应用程序提供主机到主机的可靠的数据传输服务，TCP 和 UDP 是传输层的主要协议。传输层提供了多路复用、流量控制等多种功能，例如，端口的概念就是传输层引入的，数据包到达机器后，端口可以标识该数据包属于哪一个应用程序。</p><p>HTTP协议底层是基于TCP实现的。TCP是面向连接的协议，有连接时的三次握手与断开时的四次挥手，有传输数据时与对端进行确认接受状态的ACK，还有拥塞控制、失败重传等功能。</p><p><img src="https://static001.geekbang.org/resource/image/c5/48/c51a177690cafb4ec84d2fea0b3f0648.jpg?wh=1920x657" alt="图片" title="TCP的三次握手"></p><h3>TLS协议</h3><p>随着互联网的发展，传统HTTP协议面临很多挑战，其中一个调整就是安全问题。当我们身处不信任的网络中，我们不知道数据包会经历什么样的中间节点，中间人也可能对我们的数据做一些干扰。</p><p>窃听（Eavesdropping）、篡改（Tampering）和重放（Replay）是HTTP协议面临的三类主要攻击。为了应对这样的挑战，TLS（HTTPS）协议诞生了。它的存在主要是为了解决<strong>身份验证与加密的问题。严格来说，TLS是处于应用程序层与传输层之间的第3.5层协议。</strong></p><p>我们以TLS1.0版本为例，解释一下TLS的过程，示意图如下：</p><p><img src="https://static001.geekbang.org/resource/image/da/d3/da1d7b7c4b16eb8e5e7abb5c7103d5d3.jpg?wh=1920x1747" alt="图片" title="TLS协议连接过程"></p><ul>
<li>第1个阶段是TCP的3次握手；</li>
<li>第2个阶段是鉴权，服务器发送数字签名证书给客户端验证；</li>
<li>第3个阶段是协调，客户端验证了服务器的数字签名证书后，双方会协商对称加密的协议；</li>
<li>第4阶段是传输，客户端与服务器都会对传输的数据进行加密传输。</li>
</ul><p>不过，TLS1.0协议的问题之一在于，它在最初握手时有太多次的消息往返（roundtrip），这会导致耗时增加。好在后续升级的协议在很大程序上解决了这个问题，目前最新的版本为TLS1.3协议。</p><h3>网络层</h3><p>网络层负责在互联网之间传输数据，它能够执行路由、数据包的分段和重组等功能。对于要传出的数据包，网络层通过查找路由表选择下一跳的主机地址，然后将封装后的数据包传递给下一个链路层。 一旦数据包被目的地接收，网络层就要将数据包向上传递给适当的传输层协议（例如TCP协议或UDP协议）。</p><p><img src="https://static001.geekbang.org/resource/image/42/f5/42438aa9121fa345ba2f7e703ab12ef5.jpg?wh=1920x1031" alt="图片" title="路由器解析网络层"></p><p>TCP协议的底座IP协议就位于网络层。 IP地址会标识数据包传递给网络上的哪一个主机。IP协议是基于最大传输单元 (MTU) 进行数据包分段的。</p><p>但是，由于数据包在传输过程中本质上是不可靠的，IP 协议无法保证数据包能够正确到达目的地（提供服务可靠性的功能是在传输层和应用层完成的）。如下图，IPv4协议中的Checksum只是校验数据包中Header信息的准确性，但只能确保数据包的准确性，并验证负载数据的完整性。而IPv6协议甚至没有Checksum校验。</p><p><img src="https://static001.geekbang.org/resource/image/16/c1/160bd472624d6d594e4f764d24dbc9c1.jpg?wh=1920x1322" alt="图片" title="IPv4协议"></p><h3>网络接入层</h3><p>TCP/IP 模型中的网络接入层涵盖了OSI模型中的链路层功能，也包括了主机在局域网（LAN）中的通信协议。网络接入层目前使用最广泛的协议是以太网协议（Ethenet协议）。</p><p>为什么呢？我们知道IP协议能够解决路由的问题，但是一台机器的IP地址是动态变化的，不能将IP地址一直映射到同一台机器上。对于IPv4协议，解决这个问题的方法就是利用Address Resolution Protocol （ARP）协议获取IP地址对应的MAC地址。每台机器的MAC地址都是全球唯一的，这样，遵照Ethernet网络协议的不同厂商的设备就可以很容易在局域网中实现互联。</p><p>有了 MAC 地址以后，以太网协议会采用广播形式，将数据帧发给本地网络内所有的主机，主机网卡在接收到数据包后会解析数据包，将数据包链路层中的目标主机 MAC 地址与自身网卡的 MAC 地址进行对比。若地址相同，就接收数据包做下一步处理。若地址不同，则丢弃。</p><p>TCP/IP模型的物理层详细说明了通信介质的物理特性和硬件标准，例如，IEEE 802.3规定了Ethernet网络介质的规范。</p><p>网络的分层模型按照功能进行拆分，有效地将关注点分离开来。一个HTTP数据包逐层传递，在每次向下传递时都要进行一次封装，把上一层传递的数据包加上下一层的Header信息。像洋葱一样层层包裹。</p><p><img src="https://static001.geekbang.org/resource/image/62/cc/6271607c18ff2b49d6038ffc199a39cc.jpg?wh=1920x1154" alt="图片" title="数据包的层层封装"></p><h2>数据传输与路由协议</h2><p>在这个过程中，如果数据包过大，可能会发生分段（fragmentation），它的目的是让数据包不超过链路层的最大传输单元（MTU）。数据包会被放入到对应传出设备的缓冲区队列中，最终被设备传输，离开当前主机。</p><p>在数据包从当前设备传输到对端设备的过程中，可能经历了众多的交换机和路由器。交换机一般只处理第二层链路层的协议，而路由器可以处理第三层网络层的协议，也就是说，路由器会在路由表中查找到下一个跳节点的IP地址。</p><p>由于外部的网络环境十分复杂且一直在动态变化，所以在这个庞大的拓扑结构中，任何一个节点都可能突然下线或者加入，或者IP发生变更。因此，要想动态获得到达目标节点的最短路径，同时保证传输过程不会出现环，需要有协议协调各个路由器。</p><p>多个路由器组成了一个叫做自治系统 (Autonomous System，AS) 的实体。自治系统内部主要包括RIP协议、OSPF协议和IGRP协议，这类协议又称为内部路由协议。而自治系统之间主要是BGP协议，这一类协议被称为外部路由协议。</p><p><img src="https://static001.geekbang.org/resource/image/9d/0e/9d8dd10e96665d7b1840ea1e89cdd70e.jpg?wh=1734x1720" alt="图片" title="路由拓扑 来自 《Computer Networking A Top-Down Approach 7th》"></p><p>自治系统可以作为服务提供商（Internet service provider，ISP）提供商业服务。它可以控制流量和数据包路由路线，并决定为用户提供何种服务质量、服务成本、甚至使用何种关乎政治、安全或经济的路由策略。例如，如果一个 AS 不愿意将流量传送到另一个 AS，它可以强行禁止该路由的策略。</p><p>如上图是现实中某一个网络传输的路由拓扑图，家庭网络要想访问外部互联网，就需要首先接入到当地的ISP。这种分层路由的好处在于节省了路由表大小并减少了路由更新时的流量。如果想了解更多关于路由协议的详细论述，可以参考《Introduction to Computer Networks and Cybersecurity》的第12-13章。如果想仔细了解路由器内部的处理方式，可以参考《Computer Networking A Top-Down Approach 6th》。</p><h2>数据包解析</h2><p>数据包通过物理介质传递给对端主机后，会发生和传输相反的解包过程。解包会将当前数据包层层剥离出来进行校验和处理，并将剥离后的数据传递给上一层。最终负载的HTTP数据会到达应用程序后，开发者需要根据特定的业务需求对数据包进行处理（例如爬虫项目会对HTML数据进行解析，获取结构化信息）。</p><p>当然，我们现在看到的处理流程还是比较抽象的，当数据包到达服务器后，硬件和操作系统分别执行了什么操作？Go HTTP标准库内部如何实现高效的网络处理?在下一节课中，我还会放大这个过程，带着你深入它的细节。</p><p><img src="https://static001.geekbang.org/resource/image/f3/79/f337d9aa190e9678b725dbcaaff67079.jpg?wh=1920x1554" alt="图片"></p><h2>总结</h2><p>互联网构成了我们纷繁复杂的世界，然而网络知识又极具深度和广度。在现实中，很多开发者对于网络知识一知半解，解决网络问题更是只能凭借经验，瞎猫撞上死耗子。</p><p>这节课，我借助一个最简单的HTTP请求，讲解了一个数据包经过的多个网络协议层，我们还看到了外部网络复杂的路由过程。你可以在这个过程中一窥网络的复杂和精妙。</p><p>适合开发网络服务是Go语言的巨大优势之一。我们可以看到，Go语言对网络库的封装帮助我们屏蔽掉了复杂的协议处理问题，开发者可以用简单的代码完成复杂的功能。Go语言对协程的设计和对于I/O多路复用巧妙的封装，实现了同步编程的语义，但背后实则是异步I/O的处理模式。在减轻开发者心理负担的同时，提升了网络I/O的处理效率。</p><p>在下一节，我们还将更进一步，看到操作系统、硬件的数据包处理过程，以及Go HTTP标准库的实现原理。</p><h2>课后题</h2><p>学完这节课，我也给你留一道思考题吧。</p><p>当我们无法访问外部网站的时候，你觉得可能的原因会有哪些，你的排查手段是什么？你可以参考数据包的流转过程，尝试给出尽可能全面的答案。</p><p>欢迎你在留言区留下自己思考的结果，也可以把这节课分享给对这个话题感兴趣的同事和朋友，我们下节课再见！</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/7f/d3/b5896293.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Realm</span>
  </div>
  <div class="_2_QraFYR_0">课程快到1&#47;3了，就那篇爬虫整体设计，感受到是本专栏的重点。<br><br>不是说其他内容不重要，但铺垫太多，是不是会喧宾夺主、稀释专栏的含金量？<br><br>还是希望看到Go网络编程的一些技巧、软件设计的一些哲学、接口抽象、功能编排、组合、扩展、分布式、反爬...<br><br>希望专栏组能关注下！🤩🤩🤩</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 有一句歌词是这样说的：“你想要的，我却不能够给你我全部。我能给的，却又不是你想要拥有的”。 这似乎说的就是课程进行到这里我们之间彼此的困扰。   我确实理解一些同学发现还没有开始看到代码的不满，在这里我回应一下提到的一些质疑。 <br>在 1-3篇我们进行了Go基础知识的回顾和进阶的学习路线。 4-5篇介绍了整个大型项目的流程。 6-7篇介绍了爬虫的内涵以及一个重要的话题为什么Go适合爬虫网络项目。<br>8-13篇我介绍了系统设计的3个主要问题高性能、微服务、分布式从而推导出出14即项目的设计架构和为什么这样去设计。<br>紧接着我们来到了worker开发篇，在这里我首先在15介绍了项目需要遵守的编程规范，并且在16-17介绍了爬虫的核心，一次网络爬取的流程。在18我会再次铺垫一下依赖管理，因为一个项目一开始就需要它。在19我会开始书写项目的第一行代码。 可以看到我想打造的这门专栏是比较全面的，是一条逻辑链，前面这些篇幅都是围绕着整个项目展开的。 我的教育理念可能不是每一个人想要的那样，但是希望保持一些耐心，很快你会看到你提到的这些内容。<br>期待过段时间还能够看到你的反馈。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-15 15:47:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/d2/e2/4d84ac9f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>.</span>
  </div>
  <div class="_2_QraFYR_0">铺垫太多 底层原理虽然很重要但是应该不是大家非常关心的吧  实战内容才是重点</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-15 19:34:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJv3HPde36ll9u5EpEIJyR9jMXE0K7pcuxOlf4HUcbs0po9nkicR0mbXlF1Vdoytj1vxSRCZJGOH7Q/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>愤怒的小猥琐</span>
  </div>
  <div class="_2_QraFYR_0">能不能搞点硬货，js逆向，用golang如何补环境，还有一些爬虫常用的加解密算法都没有，感觉跟专栏的初衷相悖</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-16 19:01:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/22/ae/8a2945c8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>_MISSYOURLOVE</span>
  </div>
  <div class="_2_QraFYR_0">使用ping命令来测试与目标网络的连通性，nslookup查看域名被解析到了那个IP，还可以使用 traceroute命令来对路由进行跟踪</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-16 17:49:25</div>
  </div>
</div>
</div>
</li>
</ul>