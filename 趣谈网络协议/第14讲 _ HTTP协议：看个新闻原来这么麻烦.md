<audio title="第14讲 _ HTTP协议：看个新闻原来这么麻烦" src="https://static001.geekbang.org/resource/audio/4f/34/4f71470a46d7f82c603d9d27b2756034.mp3" controls="controls"></audio> 
<p>前面讲述完<strong>传输层</strong>，接下来开始讲<strong>应用层</strong>的协议。从哪里开始讲呢，就从咱们最常用的HTTP协议开始。</p><p>HTTP协议，几乎是每个人上网用的第一个协议，同时也是很容易被人忽略的协议。</p><p>既然说看新闻，咱们就先登录 <a href="http://www.163.com">http://www.163.com</a> 。</p><p><a href="http://www.163.com">http://www.163.com</a> 是个URL，叫作<strong>统一资源定位符</strong>。之所以叫统一，是因为它是有格式的。HTTP称为协议，www.163.com是一个域名，表示互联网上的一个位置。有的URL会有更详细的位置标识，例如 <a href="http://www.163.com/index.html">http://www.163.com/index.html</a> 。正是因为这个东西是统一的，所以当你把这样一个字符串输入到浏览器的框里的时候，浏览器才知道如何进行统一处理。</p><h2>HTTP请求的准备</h2><p>浏览器会将www.163.com这个域名发送给DNS服务器，让它解析为IP地址。有关DNS的过程，其实非常复杂，这个在后面专门介绍DNS的时候，我会详细描述，这里我们先不管，反正它会被解析成为IP地址。那接下来是发送HTTP请求吗？</p><p>不是的，HTTP是基于TCP协议的，当然是要先建立TCP连接了，怎么建立呢？还记得第11节讲过的三次握手吗？</p><p>目前使用的HTTP协议大部分都是1.1。在1.1的协议里面，默认是开启了Keep-Alive的，这样建立的TCP连接，就可以在多次请求中复用。</p><!-- [[[read_end]]] --><p>学习了TCP之后，你应该知道，TCP的三次握手和四次挥手，还是挺费劲的。如果好不容易建立了连接，然后就做了一点儿事情就结束了，有点儿浪费人力和物力。</p><h2>HTTP请求的构建</h2><p>建立了连接以后，浏览器就要发送HTTP的请求。</p><p>请求的格式就像这样。</p><p><img src="https://static001.geekbang.org/resource/image/85/c1/85ebb0396cbaa45ce00b505229e523c1.jpeg?wh=1920*1080" alt=""></p><p>HTTP的报文大概分为三大部分。第一部分是<strong>请求行</strong>，第二部分是请求的<strong>首部</strong>，第三部分才是请求的<strong>正文实体</strong>。</p><h3>第一部分：请求行</h3><p>在请求行中，URL就是 <a href="http://www.163.com">http://www.163.com</a> ，版本为HTTP 1.1。这里要说一下的，就是方法。方法有几种类型。</p><p>对于访问网页来讲，最常用的类型就是<strong>GET</strong>。顾名思义，GET就是去服务器获取一些资源。对于访问网页来讲，要获取的资源往往是一个页面。其实也有很多其他的格式，比如说返回一个JSON字符串，到底要返回什么，是由服务器端的实现决定的。</p><p>例如，在云计算中，如果我们的服务器端要提供一个基于HTTP协议的API，获取所有云主机的列表，这就会使用GET方法得到，返回的可能是一个JSON字符串。字符串里面是一个列表，列表里面是一项的云主机的信息。</p><p>另外一种类型叫做<strong>POST</strong>。它需要主动告诉服务端一些信息，而非获取。要告诉服务端什么呢？一般会放在正文里面。正文可以有各种各样的格式。常见的格式也是JSON。</p><p>例如，我们下一节要讲的支付场景，客户端就需要把“我是谁？我要支付多少？我要买啥？”告诉服务器，这就需要通过POST方法。</p><p>再如，在云计算里，如果我们的服务器端，要提供一个基于HTTP协议的创建云主机的API，也会用到POST方法。这个时候往往需要将“我要创建多大的云主机？多少CPU多少内存？多大硬盘？”这些信息放在JSON字符串里面，通过POST的方法告诉服务器端。</p><p>还有一种类型叫<strong>PUT</strong>，就是向指定资源位置上传最新内容。但是，HTTP的服务器往往是不允许上传文件的，所以PUT和POST就都变成了要传给服务器东西的方法。</p><p>在实际使用过程中，这两者还会有稍许的区别。POST往往是用来创建一个资源的，而PUT往往是用来修改一个资源的。</p><p>例如，云主机已经创建好了，我想对这个云主机打一个标签，说明这个云主机是生产环境的，另外一个云主机是测试环境的。那怎么修改这个标签呢？往往就是用PUT方法。</p><p>再有一种常见的就是<strong>DELETE</strong>。这个顾名思义就是用来删除资源的。例如，我们要删除一个云主机，就会调用DELETE方法。</p><h3>第二部分：首部字段</h3><p>请求行下面就是我们的首部字段。首部是key value，通过冒号分隔。这里面，往往保存了一些非常重要的字段。</p><p>例如，<strong>Accept-Charset</strong>，表示<strong>客户端可以接受的字符集</strong>。防止传过来的是另外的字符集，从而导致出现乱码。</p><p>再如，<strong>Content-Type</strong>是指<strong>正文的格式</strong>。例如，我们进行POST的请求，如果正文是JSON，那么我们就应该将这个值设置为JSON。</p><p>这里需要重点说一下的就是<strong>缓存</strong>。为啥要使用缓存呢？那是因为一个非常大的页面有很多东西。</p><p>例如，我浏览一个商品的详情，里面有这个商品的价格、库存、展示图片、使用手册等等。商品的展示图片会保持较长时间不变，而库存会根据用户购买的情况经常改变。如果图片非常大，而库存数非常小，如果我们每次要更新数据的时候都要刷新整个页面，对于服务器的压力就会很大。</p><p>对于这种高并发场景下的系统，在真正的业务逻辑之前，都需要有个接入层，将这些静态资源的请求拦在最外面。</p><p>这个架构的图就像这样。</p><p><img src="https://static001.geekbang.org/resource/image/ca/1d/caec3ba1086557cbf694c621e7e01e1d.jpeg?wh=1254*1080" alt=""></p><p>其中DNS、CDN我在后面的章节会讲。和这一节关系比较大的就是Nginx这一层，它如何处理HTTP协议呢？对于静态资源，有Vanish缓存层。当缓存过期的时候，才会访问真正的Tomcat应用集群。</p><p>在HTTP头里面，<strong>Cache-control</strong>是用来<strong>控制缓存</strong>的。当客户端发送的请求中包含max-age指令时，如果判定缓存层中，资源的缓存时间数值比指定时间的数值小，那么客户端可以接受缓存的资源；当指定max-age值为0，那么缓存层通常需要将请求转发给应用集群。</p><p>另外，<strong>If-Modified-Since</strong>也是一个关于缓存的。也就是说，如果服务器的资源在某个时间之后更新了，那么客户端就应该下载最新的资源；如果没有更新，服务端会返回“304 Not Modified”的响应，那客户端就不用下载了，也会节省带宽。</p><p>到此为止，我们仅仅是拼凑起了HTTP请求的报文格式，接下来，浏览器会把它交给下一层传输层。怎么交给传输层呢？其实也无非是用Socket这些东西，只不过用的浏览器里，这些程序不需要你自己写，有人已经帮你写好了。</p><h2>HTTP请求的发送</h2><p>HTTP协议是基于TCP协议的，所以它使用面向连接的方式发送请求，通过stream二进制流的方式传给对方。当然，到了TCP层，它会把二进制流变成一个个报文段发送给服务器。</p><p>在发送给每个报文段的时候，都需要对方有一个回应ACK，来保证报文可靠地到达了对方。如果没有回应，那么TCP这一层会进行重新传输，直到可以到达。同一个包有可能被传了好多次，但是HTTP这一层不需要知道这一点，因为是TCP这一层在埋头苦干。</p><p>TCP层发送每一个报文的时候，都需要加上自己的地址（即源地址）和它想要去的地方（即目标地址），将这两个信息放到IP头里面，交给IP层进行传输。</p><p>IP层需要查看目标地址和自己是否是在同一个局域网。如果是，就发送ARP协议来请求这个目标地址对应的MAC地址，然后将源MAC和目标MAC放入MAC头，发送出去即可；如果不在同一个局域网，就需要发送到网关，还要需要发送ARP协议，来获取网关的MAC地址，然后将源MAC和网关MAC放入MAC头，发送出去。</p><p>网关收到包发现MAC符合，取出目标IP地址，根据路由协议找到下一跳的路由器，获取下一跳路由器的MAC地址，将包发给下一跳路由器。</p><p>这样路由器一跳一跳终于到达目标的局域网。这个时候，最后一跳的路由器能够发现，目标地址就在自己的某一个出口的局域网上。于是，在这个局域网上发送ARP，获得这个目标地址的MAC地址，将包发出去。</p><p>目标的机器发现MAC地址符合，就将包收起来；发现IP地址符合，根据IP头中协议项，知道自己上一层是TCP协议，于是解析TCP的头，里面有序列号，需要看一看这个序列包是不是我要的，如果是就放入缓存中然后返回一个ACK，如果不是就丢弃。</p><p>TCP头里面还有端口号，HTTP的服务器正在监听这个端口号。于是，目标机器自然知道是HTTP服务器这个进程想要这个包，于是将包发给HTTP服务器。HTTP服务器的进程看到，原来这个请求是要访问一个网页，于是就把这个网页发给客户端。</p><h2>HTTP返回的构建</h2><p>HTTP的返回报文也是有一定格式的。这也是基于HTTP 1.1的。</p><p><img src="https://static001.geekbang.org/resource/image/6b/63/6bc37ddcb4e7a61ca3275790820f2263.jpeg?wh=1761*937" alt=""></p><p>状态码会反映HTTP请求的结果。“200”意味着大吉大利；而我们最不想见的，就是“404”，也就是“服务端无法响应这个请求”。然后，短语会大概说一下原因。</p><p>接下来是返回首部的<strong>key value</strong>。</p><p>这里面，<strong>Retry-After</strong>表示，告诉客户端应该在多长时间以后再次尝试一下。“503错误”是说“服务暂时不再和这个值配合使用”。</p><p>在返回的头部里面也会有<strong>Content-Type</strong>，表示返回的是HTML，还是JSON。</p><p>构造好了返回的HTTP报文，接下来就是把这个报文发送出去。还是交给Socket去发送，还是交给TCP层，让TCP层将返回的HTML，也分成一个个小的段，并且保证每个段都可靠到达。</p><p>这些段加上TCP头后会交给IP层，然后把刚才的发送过程反向走一遍。虽然两次不一定走相同的路径，但是逻辑过程是一样的，一直到达客户端。</p><p>客户端发现MAC地址符合、IP地址符合，于是就会交给TCP层。根据序列号看是不是自己要的报文段，如果是，则会根据TCP头中的端口号，发给相应的进程。这个进程就是浏览器，浏览器作为客户端也在监听某个端口。</p><p>当浏览器拿到了HTTP的报文。发现返回“200”，一切正常，于是就从正文中将HTML拿出来。HTML是一个标准的网页格式。浏览器只要根据这个格式，展示出一个绚丽多彩的网页。</p><p>这就是一个正常的HTTP请求和返回的完整过程。</p><h2>HTTP 2.0</h2><p>当然HTTP协议也在不断的进化过程中，在HTTP1.1基础上便有了HTTP 2.0。</p><p>HTTP 1.1在应用层以纯文本的形式进行通信。每次通信都要带完整的HTTP的头，而且不考虑pipeline模式的话，每次的过程总是像上面描述的那样一去一回。这样在实时性、并发性上都存在问题。</p><p>为了解决这些问题，HTTP 2.0会对HTTP的头进行一定的压缩，将原来每次都要携带的大量key  value在两端建立一个索引表，对相同的头只发送索引表中的索引。</p><p>另外，HTTP 2.0协议将一个TCP的连接中，切分成多个流，每个流都有自己的ID，而且流可以是客户端发往服务端，也可以是服务端发往客户端。它其实只是一个虚拟的通道。流是有优先级的。</p><p>HTTP 2.0还将所有的传输信息分割为更小的消息和帧，并对它们采用二进制格式编码。常见的帧有<strong>Header帧</strong>，用于传输Header内容，并且会开启一个新的流。再就是<strong>Data帧</strong>，用来传输正文实体。多个Data帧属于同一个流。</p><p>通过这两种机制，HTTP 2.0的客户端可以将多个请求分到不同的流中，然后将请求内容拆成帧，进行二进制传输。这些帧可以打散乱序发送， 然后根据每个帧首部的流标识符重新组装，并且可以根据优先级，决定优先处理哪个流的数据。</p><p>我们来举一个例子。</p><p>假设我们的一个页面要发送三个独立的请求，一个获取css，一个获取js，一个获取图片jpg。如果使用HTTP 1.1就是串行的，但是如果使用HTTP 2.0，就可以在一个连接里，客户端和服务端都可以同时发送多个请求或回应，而且不用按照顺序一对一对应。</p><p><img src="https://static001.geekbang.org/resource/image/9a/1a/9a54f97931377dyy2fde0de93f4ecf1a.jpeg?wh=1401*1080" alt=""></p><p>HTTP 2.0其实是将三个请求变成三个流，将数据分成帧，乱序发送到一个TCP连接中。</p><p><img src="https://static001.geekbang.org/resource/image/3d/d3/3da001fac5701949b94e51caaee887d3.jpeg?wh=1920*651" alt=""></p><p>HTTP 2.0成功解决了HTTP 1.1的队首阻塞问题，同时，也不需要通过HTTP 1.x的pipeline机制用多条TCP连接来实现并行请求与响应；减少了TCP连接数对服务器性能的影响，同时将页面的多个数据css、js、 jpg等通过一个数据链接进行传输，能够加快页面组件的传输速度。</p><h2>QUIC协议的“城会玩”</h2><p>HTTP 2.0虽然大大增加了并发性，但还是有问题的。因为HTTP 2.0也是基于TCP协议的，TCP协议在处理包时是有严格顺序的。</p><p>当其中一个数据包遇到问题，TCP连接需要等待这个包完成重传之后才能继续进行。虽然HTTP 2.0通过多个stream，使得逻辑上一个TCP连接上的并行内容，进行多路数据的传输，然而这中间并没有关联的数据。一前一后，前面stream 2的帧没有收到，后面stream 1的帧也会因此阻塞。</p><p>于是，就又到了从TCP切换到UDP，进行“城会玩”的时候了。这就是Google的QUIC协议，接下来我们来看它是如何“城会玩”的。</p><h3>机制一：自定义连接机制</h3><p>我们都知道，一条TCP连接是由四元组标识的，分别是源 IP、源端口、目的 IP、目的端口。一旦一个元素发生变化时，就需要断开重连，重新连接。在移动互联情况下，当手机信号不稳定或者在WIFI和 移动网络切换时，都会导致重连，从而进行再次的三次握手，导致一定的时延。</p><p>这在TCP是没有办法的，但是基于UDP，就可以在QUIC自己的逻辑里面维护连接的机制，不再以四元组标识，而是以一个64位的随机数作为ID来标识，而且UDP是无连接的，所以当IP或者端口变化的时候，只要ID不变，就不需要重新建立连接。</p><h3>机制二：自定义重传机制</h3><p>前面我们讲过，TCP为了保证可靠性，通过使用<strong>序号</strong>和<strong>应答</strong>机制，来解决顺序问题和丢包问题。</p><p>任何一个序号的包发过去，都要在一定的时间内得到应答，否则一旦超时，就会重发这个序号的包。那怎么样才算超时呢？还记得我们提过的<strong>自适应重传算法</strong>吗？这个超时是通过<strong>采样往返时间RTT</strong>不断调整的。</p><p>其实，在TCP里面超时的采样存在不准确的问题。例如，发送一个包，序号为100，发现没有返回，于是再发送一个100，过一阵返回一个ACK101。这个时候客户端知道这个包肯定收到了，但是往返时间是多少呢？是ACK到达的时间减去后一个100发送的时间，还是减去前一个100发送的时间呢？事实是，第一种算法把时间算短了，第二种算法把时间算长了。</p><p>QUIC也有个序列号，是递增的。任何一个序列号的包只发送一次，下次就要加一了。例如，发送一个包，序号是100，发现没有返回；再次发送的时候，序号就是101了；如果返回的ACK  100，就是对第一个包的响应。如果返回ACK  101就是对第二个包的响应，RTT计算相对准确。</p><p>但是这里有一个问题，就是怎么知道包100和包101发送的是同样的内容呢？QUIC定义了一个offset概念。QUIC既然是面向连接的，也就像TCP一样，是一个数据流，发送的数据在这个数据流里面有个偏移量offset，可以通过offset查看数据发送到了哪里，这样只要这个offset的包没有来，就要重发；如果来了，按照offset拼接，还是能够拼成一个流。</p><p><img src="https://static001.geekbang.org/resource/image/80/2c/805aa4261yyb30a2a0e5a2f06ce5162c.jpeg?wh=1682*1080" alt=""></p><h3>机制三：无阻塞的多路复用</h3><p>有了自定义的连接和重传机制，我们就可以解决上面HTTP  2.0的多路复用问题。</p><p>同HTTP 2.0一样，同一条QUIC连接上可以创建多个stream，来发送多个 HTTP 请求。但是，QUIC是基于UDP的，一个连接上的多个stream之间没有依赖。这样，假如stream2丢了一个UDP包，后面跟着stream3的一个UDP包，虽然stream2的那个包需要重传，但是stream3的包无需等待，就可以发给用户。</p><h3>机制四：自定义流量控制</h3><p>TCP的流量控制是通过<strong>滑动窗口协议</strong>。QUIC的流量控制也是通过window_update，来告诉对端它可以接受的字节数。但是QUIC的窗口是适应自己的多路复用机制的，不但在一个连接上控制窗口，还在一个连接中的每个stream控制窗口。</p><p>还记得吗？在TCP协议中，接收端的窗口的起始点是下一个要接收并且ACK的包，即便后来的包都到了，放在缓存里面，窗口也不能右移，因为TCP的ACK机制是基于序列号的累计应答，一旦ACK了一个序列号，就说明前面的都到了，所以只要前面的没到，后面的到了也不能ACK，就会导致后面的到了，也有可能超时重传，浪费带宽。</p><p>QUIC的ACK是基于offset的，每个offset的包来了，进了缓存，就可以应答，应答后就不会重发，中间的空档会等待到来或者重发即可，而窗口的起始位置为当前收到的最大offset，从这个offset到当前的stream所能容纳的最大缓存，是真正的窗口大小。显然，这样更加准确。</p><p><img src="https://static001.geekbang.org/resource/image/a6/22/a66563b46906e7708cc69a02d43afb22.jpg?wh=795*422" alt=""></p><p>另外，还有整个连接的窗口，需要对于所有的stream的窗口做一个统计。</p><h2>小结</h2><p>好了，今天就讲到这里，我们来总结一下：</p><ul>
<li>
<p><span class="orange">HTTP协议虽然很常用，也很复杂，重点记住GET、POST、 PUT、DELETE这几个方法，以及重要的首部字段；</span></p>
</li>
<li>
<p><span class="orange">HTTP 2.0通过头压缩、分帧、二进制编码、多路复用等技术提升性能；</span></p>
</li>
<li>
<p><span class="orange">QUIC协议通过基于UDP自定义的类似TCP的连接、重试、多路复用、流量控制技术，进一步提升性能。</span></p>
</li>
</ul><p>接下来，给你留两个思考题吧。</p><ol>
<li>
<p>QUIC是一个精巧的协议，所以它肯定不止今天我提到的四种机制，你知道它还有哪些吗？</p>
</li>
<li>
<p>这一节主要讲了如何基于HTTP浏览网页，如果要传输比较敏感的银行卡信息，该怎么办呢？</p>
</li>
</ol><p>欢迎你留言和我讨论。趣谈网络协议，我们下期见！</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/b0/3a/b942f384.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我那么圆</span>
  </div>
  <div class="_2_QraFYR_0">http1.0的队首阻塞<br><br>对于同一个tcp连接，所有的http1.0请求放入队列中，只有前一个请求的响应收到了，然后才能发送下一个请求。<br><br>可见，http1.0的队首组塞发生在客户端。<br><br>3 http1.1的队首阻塞<br><br>对于同一个tcp连接，http1.1允许一次发送多个http1.1请求，也就是说，不必等前一个响应收到，就可以发送下一个请求，这样就解决了http1.0的客户端的队首阻塞。但是，http1.1规定，服务器端的响应的发送要根据请求被接收的顺序排队，也就是说，先接收到的请求的响应也要先发送。这样造成的问题是，如果最先收到的请求的处理时间长的话，响应生成也慢，就会阻塞已经生成了的响应的发送。也会造成队首阻塞。<br><br>可见，http1.1的队首阻塞发生在服务器端。<br><br>4 http2是怎样解决队首阻塞的<br><br>http2无论在客户端还是在服务器端都不需要排队，在同一个tcp连接上，有多个stream，由各个stream发送和接收http请求，各个steam相互独立，互不阻塞。<br><br>只要tcp没有人在用那么就可以发送已经生成的requst或者reponse的数据，在两端都不用等，从而彻底解决了http协议层面的队首阻塞问题。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-29 18:33:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/7f/27/03007a5e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>月饼</span>
  </div>
  <div class="_2_QraFYR_0">既然quic这么牛逼了干嘛还要tcp？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-20 20:55:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>起风了001</span>
  </div>
  <div class="_2_QraFYR_0">以前一直不是很确定Keep-Alive的作用, 今天结合tcp的知识, 终于是彻底搞清楚了. 其实就是浏览器访问服务端之后, 一个http请求的底层是tcp连接, tcp连接要经过三次握手之后,开始传输数据, 而且因为http设置了keep-alive,所以单次http请求完成之后这条tcp连接并不会断开, 而是可以让下一次http请求直接使用.当然keep-alive肯定也有timeout, 超时关闭.</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-21 18:12:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/6e/28/1e307312.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>鲍勃</span>
  </div>
  <div class="_2_QraFYR_0">我是做底层的，传输层还是基本能hold住，现在有点扛不住了😂</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-18 15:42:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/ac/a1/43d83698.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>云学</span>
  </div>
  <div class="_2_QraFYR_0">http2.0的多路复用和4g协议的多harq并发类似，quick的关键改进是把Ack的含义给提纯净了，表达的含义是收到了包，而不是tcp的＂期望包＂</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-19 12:34:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/dc/68/006ba72c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Untitled</span>
  </div>
  <div class="_2_QraFYR_0">QUIC说是基于UDP，无连接的，但是老师又说到是面向连接的，看的晕乎乎的。。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 讲tcp的时候讲了，所谓的连接其实是两边的状态，状态如果不在udp层维护，就可以在应用层维护</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-17 10:25:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/47/90/6828af58.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>偷代码的bug农</span>
  </div>
  <div class="_2_QraFYR_0">一窍不通，云里雾里</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-07-13 08:24:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/c0/25/348b4d76.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>墨萧</span>
  </div>
  <div class="_2_QraFYR_0">每次http都要经过TCP的三次握手四次挥手吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: keepalive就不用</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-18 09:31:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/af/00/9b49f42b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>skye</span>
  </div>
  <div class="_2_QraFYR_0">“一前一后，前面 stream 2 的帧没有收到，后面 stream 1 的帧也会因此阻塞。”这个和队首阻塞的区别是啥，不太明白？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 一个是http层的，一个是tcp层的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-20 09:16:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/0f/c0/e6151cce.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>花仙子</span>
  </div>
  <div class="_2_QraFYR_0">UDP不用保持连接状态，不用建立更多socket，是不是就说服务端只能凭借客户端的源端口号来判定是客户端哪个应用发送的，是吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-11 17:58:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/04/0d/3dc5683a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>柯察金</span>
  </div>
  <div class="_2_QraFYR_0">怎么说呢，感觉听了跟看书效果一样的，比较晦涩。因为平时接触比较多的就是 tcp http ，结果听了感觉对实际开发好像帮助不大，因为都是一个个知识点的感觉，像准备考试。希望能结合实际应用场景，讲解。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 太深入就不适合听了，所以定位还是入门</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-25 15:12:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/53/6f/6051e0f1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Summer___J</span>
  </div>
  <div class="_2_QraFYR_0">老师，目前QUIC协议的典型应用场景是什么？这个协议听上去比TCP智能，随着技术演进、这个协议已经融合到TCP里面了还是仍然是一个独立的协议？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-28 08:08:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/fa/ab/0d39e745.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>李小四</span>
  </div>
  <div class="_2_QraFYR_0">网络_14:<br>开始看到文章有点懵：<br>HTTP&#47;2解决了HTTP&#47;1.1队首阻塞问题.但是HTTP2是基于TCP的，TCP严格要求包的顺序，这不是矛盾吗？<br><br>然后研究了一下，发现是不同层的队首阻塞：<br>1. 应用层的队首阻塞(HTTP&#47;1.1-based head of line blocking):<br>HTTP&#47;1.1可以使用多个TCP连接，但对连接数依然有限制，一次请求要等到连接中其他请求完成后才能开始(Pipeline机制也没能解决好这个问题)，所以没有空闲连接的时候请求被阻塞，这是应用层的阻塞。<br>HTTP&#47;2底层使用了一个TCP连接，上层虚拟了stream，HTTP请求跟stream打交道，无需等待前面的请求完成，这确实解决了应用层的队首阻塞问题。<br><br>2. 传输层的队首阻塞(TCP-based head of line blocking):<br>正如文中所述，TCP的应答是严格有序的，如果前面的包没到，即使后面的到了也不能应答，这样可能会导致后面的包被重传，窗口被“阻塞”在队首，这就是传输层的队首阻塞。<br>不管是HTTP&#47;1.1还是HTTP&#47;2，在传输层都是基于TCP，那么TCP层的队首阻塞问题都是存在的(只能由HTTP&#47;3(based on QUIC)来解决了)，另外，HTTP&#47;2用了单个TCP连接，那么在丢包率严重的场景，表现可能比HTTP&#47;1.1更差。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-22 15:27:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/a7/92/d4ce2462.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>传说中的风一样</span>
  </div>
  <div class="_2_QraFYR_0">cache control部分讲错了，max–age不是这么用的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这里忘了说一下客户端的cache-control机制了，一个是客户端是否本地缓存过期。这里重点强调的类似varnish缓存服务器的行为</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-23 08:09:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/2b/ec/af6d0b10.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>caohuan</span>
  </div>
  <div class="_2_QraFYR_0">本篇所得：1.应用层 http1.0和2.0的使用，url通过DNS转化为IP地址，HTTP是基于TCP协议，TCP需要三次握手 进行连接，四次挥手 断开连接，而QUIC基于 UDP连接，不需要三次握手，节省资源，提高了性能和效率;<br>2.http 包括 请求行 、首部、正文实体，请求行 包含 方法、SP、URL、版本，首部 包含 首部字段名、SP、字段值、cr&#47;if;<br>3.http 很复杂，它基于TCP协议，它包含 get获取资源，put 传输 修改的信息给服务器，post发送 新建的资源 给服务器，delete 删除资源;<br>4.http1.0为串联计算，http2.0可以并联计算，http2.0可以通过 头压缩 分帧、二进制编码、多路复用 来提示性能。<br>5.QUIC协议基于UDP自定义的类型，TCP的连接、重试、多路复用、流量控制 提升性能，对应四个机制，分别为 1）自定义连接机制 2）自定义重传机制 3）无阻塞多路复用 4）自定义流量控制。<br><br>回答老师的问题：第一个问题为提高性能，听老师后面的解答，第二个问题：输送敏感数据 比如银行信息 可以用 加密技术，公钥和私钥 保证安全性。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-04 12:56:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/d6/39/6b45878d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>意无尽</span>
  </div>
  <div class="_2_QraFYR_0">哇，竟然坚持到这里了（虽然一半都还不到），虽然前面也有很多不懂。基本上从第二章开时候每一节都会花费一两个小时去理解，但是花费确实值啊，让我一个网络小白慢慢了解了网络的各个方面，感觉像是打开了另一个奇妙的世界！相当赞，后期还要刷第二遍！学完这个必须继续购买趣谈Linux操作系统！感谢刘超老师！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-30 02:46:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/72/67/aa52812a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>stark</span>
  </div>
  <div class="_2_QraFYR_0">有个地方不是很明白，就是里面说的流数据，比如，我在实际的应用里怎么查看下这些数据什么，比如像top这样的，怎么查看呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: tcpdump，其实dump出来没有所谓的流，都是感觉上的流，还不是一个个的网络包</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-11 14:01:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/de/bf/4df4224d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>shuifa</span>
  </div>
  <div class="_2_QraFYR_0">URI 属于 URL 更高层次的抽象，一种字符串文本标准。<br><br>就是说，URI 属于父类，而 URL 属于 URI 的子类。URL 是 URI 的一个子集。<br><br>二者的区别在于，URI 表示请求服务器的路径，定义这么一个资源。而 URL 同时说明要如何访问这个资源（http:&#47;&#47;）。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-25 08:24:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/be/8c/a5a3788f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>大灰狼</span>
  </div>
  <div class="_2_QraFYR_0">quic推动了tls1.3，实现了tcp的序列号，流控，拥塞控制，重传，实现了http2的多路复用，流控等<br>参考infoq文章。<br>在实际进行中间人的过程中，quic是一个很头疼的协议。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-09 23:47:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/5f/dc/d16e0923.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>heliang</span>
  </div>
  <div class="_2_QraFYR_0">http2.0 并行传输同一个请求不同stream的时候，如果“”前面 stream 2 的帧没有收到，后面 stream1也会阻塞&quot;，是阻塞在tcp重组上吗<br><br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-12 12:23:20</div>
  </div>
</div>
</div>
</li>
</ul>