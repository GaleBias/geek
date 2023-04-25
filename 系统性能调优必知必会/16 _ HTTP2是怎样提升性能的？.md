<audio title="16 _ HTTP2是怎样提升性能的？" src="https://static001.geekbang.org/resource/audio/ef/fb/ef86354bab99bd133610c355793d2efb.mp3" controls="controls"></audio> 
<p>你好，我是陶辉。</p><p>上一讲我们从多个角度优化HTTP/1的性能，但获得的收益都较为有限，而直接将其升级到兼容HTTP/1的HTTP/2协议，性能会获得非常大的提升。</p><p>HTTP/2协议既降低了传输时延也提升了并发性，已经被主流站点广泛使用。多数HTTP头部都可以被压缩90%以上的体积，这节约了带宽也提升了用户体验，像Google的高性能协议gRPC也是基于HTTP/2协议实现的。</p><p>目前常用的Web中间件都已支持HTTP/2协议，然而如果你不清楚它的原理，对于Nginx、Tomcat等中间件新增的流、推送、消息优先级等HTTP/2配置项，你就不知是否需要调整。</p><p>同时，许多新协议都会参考HTTP/2优秀的设计，如果你不清楚HTTP/2的性能究竟高在哪里，也就很难对当下其他应用层协议触类旁通。而且，HTTP/2协议也并不是毫无缺点，到2020年3月时它的替代协议<a href="https://zh.wikipedia.org/wiki/HTTP/3">HTTP/3</a> 已经经历了<a href="https://tools.ietf.org/html/draft-ietf-quic-http-27">27个草案</a>，推出在即。HTTP/3的目标是优化传输层协议，它会保留HTTP/2协议在应用层上的优秀设计。如果你不懂HTTP/2，也就很难学会未来的HTTP/3协议。</p><p>所以，这一讲我们就将介绍HTTP/2对HTTP/1.1协议都做了哪些改进，从消息的编码、传输等角度说清楚性能提升点，这样，你就能理解支持HTTP/2的中间件为什么会提供那些参数，以及如何权衡HTTP/2带来的收益与付出的升级成本。</p><!-- [[[read_end]]] --><h2>静态表编码能节约多少带宽？</h2><p>HTTP/1.1协议最为人诟病的是ASCII头部编码效率太低，浪费了大量带宽。HTTP/2使用了静态表、动态表两种编码技术（合称为HPACK），极大地降低了HTTP头部的体积，搞清楚编码流程，你自然就会清楚服务器提供的http2_max_requests等配置参数的意义。</p><p>我们以一个具体的例子来观察编码流程。每一个HTTP/1.1请求都会有Host头部，它指示了站点的域名，比如：</p><pre><code>Host: test.taohui.tech\r\n
</code></pre><p>算上冒号空格以及结尾的\r\n，它占用了24字节。<strong>使用静态表及Huffman编码，可以将它压缩为13字节，也就是节约了46%的带宽！</strong>这是如何做到的呢？</p><p>我用Chrome访问站点test.taohui.tech，并用Wireshark工具抓包（关于如何用Wireshark抓HTTP/2协议的报文，如果你还不太清楚，可参见<a href="https://time.geekbang.org/course/detail/175-104932">《Web协议详解与抓包实战》第51课</a>）后，下图高亮的头部就是第1个请求的Host头部，其中每8个蓝色的二进制位是1个字节，报文中用了13个字节表示Host头部。</p><p><img src="https://static001.geekbang.org/resource/image/09/1f/097e7f4549eb761c96b61368c416981f.png?wh=1529*651" alt=""></p><p>HTTP/2能够用13个字节编码原先的24个字节，是依赖下面这3个技术。</p><p>首先基于二进制编码，就不需要冒号、空格和\r\n作为分隔符，转而用表示长度的1个字节来分隔即可。比如，上图中的01000001就表示Host，而10001011及随后的11个字节表示域名。</p><p>其次，使用静态表来描述Host头部。什么是静态表呢？HTTP/2将61个高频出现的头部，比如描述浏览器的User-Agent、GET或POST方法、返回的200 SUCCESS响应等，分别对应1个数字再构造出1张表，并写入HTTP/2客户端与服务器的代码中。由于它不会变化，所以也称为静态表。</p><p><img src="https://static001.geekbang.org/resource/image/5c/98/5c180e1119c1c0eb66df03a9c10c5398.png?wh=635*850" alt=""></p><p>这样收到01000001时，根据<a href="https://tools.ietf.org/html/rfc7541">RFC7541</a> 规范，前2位为01时，表示这是不包含Value的静态表头部：</p><p><img src="https://static001.geekbang.org/resource/image/cd/37/cdf16023ab2c2f4f67f0039b8da47837.png?wh=752*348" alt=""></p><p>再根据索引000001查到authority头部（Host头部在HTTP/2协议中被改名为authority）。紧跟的字节表示域名，其中首个比特位表示域名是否经过Huffman编码，而后7位表示了域名的长度。在本例中，10001011表示域名共有11个字节（8+2+1=11），且使用了Huffman编码。</p><p>最后，使用静态Huffman编码，可以将16个字节的test.taohui.tech压缩为11个字节，这是怎么做到的呢？根据信息论，高频出现的信息用较短的编码表示后，可以压缩体积。因此，在统计互联网上传输的大量HTTP头部后，HTTP/2依据统计频率将ASCII码重新编码为一张表，参见<a href="https://tools.ietf.org/html/rfc7541#page-27">这里</a>。test.taohui.tech域名用到了10个字符，我把这10个字符的编码列在下表中。</p><p><img src="https://static001.geekbang.org/resource/image/81/de/81d2301553c825a466b1f709924ba6de.jpg?wh=1164*1022" alt=""></p><p>这样，接收端在收到下面这串比特位（最后3位填1补位）后，通过查表（请注意每个字符的颜色与比特位是一一对应的）就可以快速解码为：</p><p><img src="https://static001.geekbang.org/resource/image/57/50/5707f3690f91fe54045f4d8154fe4e50.jpg?wh=1808*338" alt=""></p><p>由于8位的ASCII码最小压缩为5位，所以静态Huffman的最大压缩比只有5/8。关于Huffman编码是如何构造的，你可以参见<a href="https://time.geekbang.org/dailylesson/detail/100028441">每日一课《HTTP/2 能带来哪些性能提升？》</a>。</p><h2>动态表编码能节约多少带宽？</h2><p>虽然静态表已经将24字节的Host头部压缩到13字节，<strong>但动态表可以将它压缩到仅1字节，这就能节省96%的带宽！</strong>那动态表是怎么做到的呢？</p><p>你可能注意到，当下许多页面含有上百个对象，而REST架构的无状态特性，要求下载每个对象时都得携带完整的HTTP头部。如果HTTP/2能在一个连接上传输所有对象，那么只要客户端与服务器按照同样的规则，对首次出现的HTTP头部用一个数字标识，随后再传输它时只传递数字即可，这就可以实现几十倍的压缩率。所有被缓存的头部及其标识数字会构成一张表，它与已经传输过的请求有关，是动态变化的，因此被称为动态表。</p><p>静态表有61项，所以动态表的索引会从62起步。比如下图中的报文中，访问test.taohui.tech的第1个请求有13个头部需要加入动态表。其中，Host: test.taohui.tech被分配到的动态表索引是74（索引号是倒着分配的）。</p><p><img src="https://static001.geekbang.org/resource/image/69/e0/692a5fad16d6acc9746e57b69b4f07e0.png?wh=808*829" alt=""></p><p>这样，后续请求使用到Host头部时，只需传输1个字节11001010即可。其中，首位1表示它在动态表中，而后7位1001010值为64+8+2=74，指向服务器缓存的动态表第74项：</p><p><img src="https://static001.geekbang.org/resource/image/9f/31/9fe864459705513bc361cee5eafd3431.png?wh=876*313" alt=""></p><p>静态表、Huffman编码、动态表共同完成了HTTP/2头部的编码，其中，前两者可以将体积压缩近一半，而后者可以将反复传输的头部压缩95%以上的体积！</p><p><img src="https://static001.geekbang.org/resource/image/c0/0c/c08db9cb2c55cb05293c273b8812020c.png?wh=1117*339" alt=""></p><p>那么，是否要让一条连接传输尽量多的请求呢？并不是这样。动态表会占用很多内存，影响进程的并发能力，所以服务器都会提供类似http2_max_requests这样的配置，限制一个连接上能够传输的请求数量，通过关闭HTTP/2连接来释放内存。<strong>因此，http2_max_requests并不是越大越好，通常我们应当根据用户浏览页面时访问的对象数量来设定这个值。</strong></p><h2>如何并发传输请求？</h2><p>HTTP/1.1中的KeepAlive长连接虽然可以传输很多请求，但它的吞吐量很低，因为在发出请求等待响应的那段时间里，这个长连接不能做任何事！而HTTP/2通过Stream这一设计，允许请求并发传输。因此，HTTP/1.1时代Chrome通过6个连接访问页面的速度，远远比不上HTTP/2单连接的速度，具体测试结果你可以参考这个<a href="https://http2.akamai.com/demo">页面</a>。</p><p>为了理解HTTP/2的并发是怎样实现的，你需要了解Stream、Message、Frame这3个概念。HTTP请求和响应都被称为Message消息，它由HTTP头部和包体构成，承载这二者的叫做Frame帧，它是HTTP/2中的最小实体。Frame的长度是受限的，比如Nginx中默认限制为8K（http2_chunk_size配置），因此我们可以得出2个结论：HTTP消息可以由多个Frame构成，以及1个Frame可以由多个TCP报文构成（TCP MSS通常小于1.5K）。</p><p>再来看Stream流，它与HTTP/1.1中的TCP连接非常相似，当Stream作为短连接时，传输完一个请求和响应后就会关闭；当它作为长连接存在时，多个请求之间必须串行传输。在HTTP/2连接上，理论上可以同时运行无数个Stream，这就是HTTP/2的多路复用能力，它通过Stream实现了请求的并发传输。</p><p><a href="https://developers.google.com/web/fundamentals/performance/http2"><img src="https://static001.geekbang.org/resource/image/b0/c8/b01f470d5d03082159e62a896b9376c8.png?wh=576*468" alt="" title="图片来源：https://developers.google.com/web/fundamentals/performance/http2"></a></p><p>虽然RFC规范并没有限制并发Stream的数量，但服务器通常都会作出限制，比如Nginx就默认限制并发Stream为128个（http2_max_concurrent_streams配置），以防止并发Stream消耗过多的内存，影响了服务器处理其他连接的能力。</p><p>HTTP/2的并发性能比HTTP/1.1通过TCP连接实现并发要高。这是因为，<strong>当HTTP/2实现100个并发Stream时，只经历1次TCP握手、1次TCP慢启动以及1次TLS握手，但100个TCP连接会把上述3个过程都放大100倍！</strong></p><p>HTTP/2还可以为每个Stream配置1到256的权重，权重越高服务器就会为Stream分配更多的内存、流量，这样按照资源渲染的优先级为并发Stream设置权重后，就可以让用户获得更好的体验。而且，Stream间还可以有依赖关系，比如若资源A、B依赖资源C，那么设置传输A、B的Stream依赖传输C的Stream即可，如下图所示：</p><p><a href="https://developers.google.com/web/fundamentals/performance/http2"><img src="https://static001.geekbang.org/resource/image/9c/97/9c068895a9d2dc66810066096172a397.png?wh=968*399" alt="" title="图片来源：https://developers.google.com/web/fundamentals/performance/http2"></a></p><h2>服务器如何主动推送资源？</h2><p>HTTP/1.1不支持服务器主动推送消息，因此当客户端需要获取通知时，只能通过定时器不断地拉取消息。HTTP/2的消息推送结束了无效率的定时拉取，节约了大量带宽和服务器资源。</p><p><img src="https://static001.geekbang.org/resource/image/f0/16/f0dc7a3bfc5709adc434ddafe3649316.png?wh=800*402" alt=""></p><p>HTTP/2的推送是这么实现的。首先，所有客户端发起的请求，必须使用单号Stream承载；其次，所有服务器进行的推送，必须使用双号Stream承载；最后，服务器推送消息时，会通过PUSH_PROMISE帧传输HTTP头部，并通过Promised Stream ID告知客户端，接下来会在哪个双号Stream中发送包体。</p><p><img src="https://static001.geekbang.org/resource/image/a1/62/a1685cc8e24868831f5f2dd961ad3462.png?wh=1036*607" alt=""></p><p>在SDK中调用相应的API即可推送消息，而在Web资源服务器中可以通过配置文件做简单的资源推送。比如在Nginx中，如果你希望客户端访问/a.js时，服务器直接推送/b.js，那么可以这么配置：</p><pre><code>location /a.js { 
  http2_push /b.js; 
}
</code></pre><p>服务器同样也会控制并发推送的Stream数量（如http2_max_concurrent_pushes配置），以减少动态表对内存的占用。</p><h2>小结</h2><p>这一讲我们介绍了HTTP/2的高性能是如何实现的。</p><p>静态表和Huffman编码可以将HTTP头部压缩近一半的体积，但这只是连接上第1个请求的压缩比。后续请求头部通过动态表可以压缩90%以上，这大大提升了编码效率。当然，动态表也会导致内存占用过大，影响服务器的总体并发能力，因此服务器会限制HTTP/2连接的使用时长。</p><p>HTTP/2的另一个优势是实现了Stream并发，这节约了TCP和TLS协议的握手时间，并减少了TCP的慢启动阶段对流量的影响。同时，Stream之间可以用Weight权重调节优先级，还可以直接设置Stream间的依赖关系，这样接收端就可以获得更优秀的体验。</p><p>HTTP/2支持消息推送，从HTTP/1.1的拉模式到推模式，信息传输效率有了巨大的提升。HTTP/2推消息时，会使用PUSH_PROMISE帧传输头部，并用双号的Stream来传递包体，了解这一点对定位复杂的网络问题很有帮助。</p><p>HTTP/2的最大问题来自于它下层的TCP协议。由于TCP是字符流协议，在前1字符未到达时，后接收到的字符只能存放在内核的缓冲区里，即使它们是并发的Stream，应用层的HTTP/2协议也无法收到失序的报文，这就叫做队头阻塞问题。解决方案是放弃TCP协议，转而使用UDP协议作为传输层协议，这就是HTTP/3协议的由来。</p><p><img src="https://static001.geekbang.org/resource/image/38/d1/3862dad08cecc75ca6702c593a3c9ad1.png?wh=1280*635" alt=""></p><h2>思考题</h2><p>最后，留给你一道思考题。为什么HTTP/2要用静态Huffman查表法对字符串编码，基于连接上的历史数据统计信息做动态Huffman编码不是更有效率吗？欢迎你在留言区与我一起探讨。</p><p>感谢阅读，如果你觉得这节课对你有一些启发，也欢迎把它分享给你的朋友。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/43/79/18073134.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>test</span>
  </div>
  <div class="_2_QraFYR_0">protobuf是对请求body进行了压缩，http2是对请求的header进行压缩。http2还可以使用stream方式传输，这些都是protobuf没有解决的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，h2是应用层协议，而pb只是纯粹的消息编码工具</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-12 11:21:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/63/2e/e49116d1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_007</span>
  </div>
  <div class="_2_QraFYR_0">虽然h2做了很多性能上的提升，但还是得结合实际来看，比如上传大文件使用h2的话，性能会不及h1。因为上传的文件大小是一样的，但是h2的流控会限制住每条connection,h2的默认控制窗口在65535字节，尽管tcp没有拥塞窗口的控制，但也要受限于h2自己的流控。也就是一路发送65535字节后就得等服务端ack。所以链接复用只对小请求有效。除非客户端和服务端避免特地避免了上述问题。陶老师，觉得呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你好Geek_007，我认为h2针对的应用场景是并发传输场景，如果只有1个传输的请求，那么讨论h2是没有意义的，它在tcp之上加了那么多约束，肯定没有直接跑在h1上快，特别是你提到的大文件，那么http header的压缩意义也不大了。因此，当客户端并发请求数量为1，且文件很大时，h2不会提升性能，甚至会降低性能。<br>然而，一旦有上百个HTTP请求并发传输时，h2的意义仍然非常大，即使有1个大文件在传输，h2连接仍然允许小文件的并发传输（比如你提到的stream流控）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-10 20:12:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/f9/30/54c71bf9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>不会飞的海燕</span>
  </div>
  <div class="_2_QraFYR_0">老师，既然h2只是对头部进行压缩，那h2+pb压缩Body是不是可以进一步减少体积；我看有的公司使用tcp+pb，这种方式比h2+pb传输效率高吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1、可以的，grpc协议就是h2+pb。<br>2、tcp+pb是非常糟糕的传输协议，应用层协议应该有头部和包体两部分，这样负载均衡、中间件才能监控系统。pb只是编码格式，tcp只是传输层协议，这个系统的监控、高可用一定很有问题。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-26 23:46:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>myrfy</span>
  </div>
  <div class="_2_QraFYR_0">请问老师，h2的推送和websocket有什么区别呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你好myrfy，主要是消息格式的不同。<br>websocket的推送是纯粹的二进制流。<br>h2的推送是HTTP消息，必须含有HTTP头部、包体。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-10 09:08:08</div>
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
  <div class="_2_QraFYR_0">静态表保存了最常用的一些头部，这些不变的头部可以全局保存一份，节约内存，不用每个连接重新构建，也节省构建表的时间。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-10 07:37:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/3XbCueYYVWTiclv8T5tFpwiblOxLphvSZxL4ujMdqVMibZnOiaFK2C5nKRGv407iaAsrI0CDICYVQJtiaITzkjfjbvrQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>有铭</span>
  </div>
  <div class="_2_QraFYR_0">我查了很多资料，都说h2的推送是针对资源的，单次请求，主动传输多个关联资源。目前没发现有暴露api给应用层使用者主动推送消息的案例。所以似乎h2的推送和我们传统意义上的推送不是一个概念，至少它和websocket这种真正的应用层可控双向传输不是一回事。我也希望老师能有案例告诉我h2真能达到websocket那种效果</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你好有铭，h2确实是针对资源的推送，websocket只不过没有定义应用层协议，它只是允许双向传输，至少传输的消息如何编码，它是不管的。而h2要求，推送时仍然要使用http作为推送协议，这是差别。<br>至少你说的案例，Java最基本的Servelet编程，其中HttpServletRequest有一个方法叫newPushBuilder，就可以推送消息。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-11 09:12:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/56/75/6bf38a1e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>坤哥</span>
  </div>
  <div class="_2_QraFYR_0"> 感觉stream 好像rabbitmq  的channel 的概念，也是tcp同一个socket多路复用，不知道理解对吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对的，一个是协议报文上的封装，一个则是语言层的封装</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-08 16:30:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/11/78/4f0cd172.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>妥协</span>
  </div>
  <div class="_2_QraFYR_0">请教老师，多个stream之间并发是通过stream id做隔离的吗？wireshake中看到的一条条记录是frame还是message?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，通过stream id。看到的报文都是frame，message是抽象的逻辑概念。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-29 04:40:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/47/b0/a9b77a1e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>冬风向左吹</span>
  </div>
  <div class="_2_QraFYR_0">请问老师，不知道我的这些理解是不是对的：<br>1、多个stream并发是共用的同一个tcp连接，所以只需要1 次 TCP 握手、1 次 TCP 慢启动以及 1 次 TLS 握手；<br>2、因为多个stream共用了同一个tcp连接，tcp报文是有序的，所以也会有队头阻塞问题；<br>3、同一个stream中的多个请求是串行的；<br>4、首个请求使用静态表和Huffman编码，后续请求头部使用动态表编码；<br><br>疑问：<br>老师的示例中，为什么host头部一会儿在静态表中，一会儿在动态表中？是因为上面第4点的原因吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你好东郭，你的理解前3点都是对的，第4点需要做一下补充：<br>首次出现的HTTP头部（不是首个请求）使用静态表和Huffman，后续请求头部使用动态表编码有一个前提，就是动态表没有超出限制。<br><br>关于host头部，可以这么理解：HTTP头部包括name和value，其中host头部仅有name在静态表中，所以首次出现时使用了静态表，但动态表可以存放name&#47;value，所以第2次出现时就使用了动态表。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-10 07:50:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/07/d2/0d7ee298.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>惘 闻</span>
  </div>
  <div class="_2_QraFYR_0">老师，syream相当于tcp链接，那么h2的并发stream不就相当于同时开了多个tcp链接吗？这样的话和h1有什么区别？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 只是相当于，TCP是由内核维护的，包括发送、接收缓冲区，以及其中的重发定时器、保活定时器，以及滑动窗口、拥塞容器、MSS等各种参数，成本很高。而stream则是应用层概念，多个stream共享1个tcp连接，区别就很大了，比如慢启动就不用重复执行，TLS握手也不会反复执行</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-08 21:23:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/d8/ee/6e7c2264.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Only now</span>
  </div>
  <div class="_2_QraFYR_0">静态编码兼容连接初始阶段，在首次发起连接时，无连接历史统计数据可做参考，此时要能效率的使用编码，那么只好使用预制的编码表，这样可以做到多端，多服务的统一。<br><br>基于连接上的历史数据统计信息做动态是需要额外付出代价的，它同时要求客户端和服务器进行传输统计，且由于并发请求和响应引起的两端同步问题可能导致动态编码表的不一致。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-20 10:14:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/56/75/6bf38a1e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>坤哥</span>
  </div>
  <div class="_2_QraFYR_0">老师，这讲我看了3次也不明白，你的另一课程web socket有这方面的详细展开吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 有的，《Web协议详解与抓包实战》第4部分有20节课都在讲HTTP2协议。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-08 16:26:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/60/de/d752c204.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>鹤鸣</span>
  </div>
  <div class="_2_QraFYR_0">静态表和动态表本身也是占了一定的空间的，在发送报文时，静态表本身不需要随着报文被发送，因为双方已经达成了共识。但是对端不知道动态表是如何编码的，所以动态表则需要随着报文一起发送。也就是说，动态表本身也占了一部分发送的数据量，增大了待发送报文的长度。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你好鹤鸣，前面理解都对，但动态表不需要发送，只要双方对于首次出现的HTTP头部，用同样的规则去构建动态表即可，你可以结合wireshark抓包重读下我在文中的例子</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-14 14:33:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/97/69/80945634.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>罐头瓶子</span>
  </div>
  <div class="_2_QraFYR_0">静态 Huffman 编码可以在第一次传输时就降低头部大小，这部分编码是所有链接都可公用的，服务端客户端可降低动态编码的内存消耗。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-10 07:19:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/93/cd/dbafc7d1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>全麦小面包</span>
  </div>
  <div class="_2_QraFYR_0">老师的例子中，静态Huffman码有5位的，也有6位的，怎么确定当前读的数据是几位呀？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-11 16:48:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/37/a0/032d0828.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>上杉夏香</span>
  </div>
  <div class="_2_QraFYR_0">动态哈夫曼编码。首先需要客户端与服务器相一致，其次因为动态维护，需要占用彼此的内存，另外就是占用cpu资源，每次请求响应都需要动态计算调整。整体效果不如静态提前编码好。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-03 22:51:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/d4/f3/129d6dfe.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>李二木</span>
  </div>
  <div class="_2_QraFYR_0">stream的单号和双号是什么意思？是奇数和偶数区别吗？<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-13 17:11:24</div>
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
  <div class="_2_QraFYR_0">老师，动态表简单理解是不是这样子？<br>比如客户端第一次服务器，客户端将请求的头部对象进行动态表构建，然后发给服务端，服务端这边也会构建动态表。以后客户端再发请求每次就传递数字了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，HTTP2中的动态表就是这样的，两端基于相同的规则，对首次出现的Header构建</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-04 23:18:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/8f/cf/890f82d6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>那时刻</span>
  </div>
  <div class="_2_QraFYR_0">请教一个在HTTP&#47;2里Stream权重的问题，A（权重12）、B（权重8）的Stream依赖传输 C（权重3）Stream, 此时这个Stream的权重还是 3么？如果权重是3的话，那之前A，B所在Stream被丢弃了么？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-10 11:42:12</div>
  </div>
</div>
</div>
</li>
</ul>