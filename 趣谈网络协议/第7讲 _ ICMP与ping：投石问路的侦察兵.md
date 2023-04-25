<audio title="第7讲 _ ICMP与ping：投石问路的侦察兵" src="https://static001.geekbang.org/resource/audio/24/ad/2486b041bd60d50f2c45a19ce827e8ad.mp3" controls="controls"></audio> 
<p>无论是在宿舍，还是在办公室，或者运维一个数据中心，我们常常会遇到网络不通的问题。那台机器明明就在那里，你甚至都可以通过机器的终端连上去看。它看着好好的，可是就是连不上去，究竟是哪里出了问题呢？</p><h2>ICMP协议的格式</h2><p>一般情况下，你会想到ping一下。那你知道ping是如何工作的吗？</p><p>ping是基于ICMP协议工作的。<strong>ICMP</strong>全称<strong>Internet Control Message Protocol</strong>，就是<strong>互联网控制报文协议</strong>。这里面的关键词是“控制”，那具体是怎么控制的呢？</p><p>网络包在异常复杂的网络环境中传输时，常常会遇到各种各样的问题。当遇到问题的时候，总不能“死个不明不白”，要传出消息来，报告情况，这样才可以调整传输策略。这就相当于我们经常看到的电视剧里，古代行军的时候，为将为帅者需要通过侦察兵、哨探或传令兵等人肉的方式来掌握情况，控制整个战局。</p><p>ICMP报文是封装在IP包里面的。因为传输指令的时候，肯定需要源地址和目标地址。它本身非常简单。因为作为侦查兵，要轻装上阵，不能携带大量的包袱。</p><p><img src="https://static001.geekbang.org/resource/image/20/e2/201589bb205c5b00ad42e0081aa46fe2.jpg?wh=3043*1243" alt=""></p><p>ICMP报文有很多的类型，不同的类型有不同的代码。<strong>最常用的类型是主动请求为8，主动请求的应答为0</strong>。</p><h2>查询报文类型</h2><p>我们经常在电视剧里听到这样的话：主帅说，来人哪！前方战事如何，快去派人打探，一有情况，立即通报！</p><!-- [[[read_end]]] --><p>这种是主帅发起的，主动查看敌情，对应ICMP的<strong>查询报文类型</strong>。例如，常用的<strong>ping就是查询报文，是一种主动请求，并且获得主动应答的ICMP协议。</strong>所以，ping发的包也是符合ICMP协议格式的，只不过它在后面增加了自己的格式。</p><p>对ping的主动请求，进行网络抓包，称为<strong>ICMP ECHO REQUEST。<strong>同理主动请求的回复，称为</strong>ICMP ECHO REPLY</strong>。比起原生的ICMP，这里面多了两个字段，一个是<strong>标识符</strong>。这个很好理解，你派出去两队侦查兵，一队是侦查战况的，一队是去查找水源的，要有个标识才能区分。另一个是<strong>序号</strong>，你派出去的侦查兵，都要编个号。如果派出去10个，回来10个，就说明前方战况不错；如果派出去10个，回来2个，说明情况可能不妙。</p><p>在选项数据中，ping还会存放发送请求的时间值，来计算往返时间，说明路程的长短。</p><h2>差错报文类型</h2><p>当然也有另外一种方式，就是差错报文。</p><p>主帅骑马走着走着，突然来了一匹快马，上面的小兵气喘吁吁的：报告主公，不好啦！张将军遭遇埋伏，全军覆没啦！这种是异常情况发起的，来报告发生了不好的事情，对应ICMP的<strong>差错报文类型</strong>。</p><p>我举几个ICMP差错报文的例子：<strong>终点不可达为3，源抑制为4，超时为11，重定向为5</strong>。这些都是什么意思呢？我给你具体解释一下。</p><p><strong>第一种是终点不可达</strong>。小兵：报告主公，您让把粮草送到张将军那里，结果没有送到。</p><p>如果你是主公，你肯定会问，为啥送不到？具体的原因在代码中表示就是，网络不可达代码为0，主机不可达代码为1，协议不可达代码为2，端口不可达代码为3，需要进行分片但设置了不分片位代码为4。</p><p>具体的场景就像这样：</p><ul>
<li>网络不可达：主公，找不到地方呀？</li>
<li>主机不可达：主公，找到地方没这个人呀？</li>
<li>协议不可达：主公，找到地方，找到人，口号没对上，人家天王盖地虎，我说12345！</li>
<li>端口不可达：主公，找到地方，找到人，对了口号，事儿没对上，我去送粮草，人家说他们在等救兵。</li>
<li>需要进行分片但设置了不分片位：主公，走到一半，山路狭窄，想换小车，但是您的将令，严禁换小车，就没办法送到了。</li>
</ul><p><strong>第二种是源站抑制</strong>，也就是让源站放慢发送速度。小兵：报告主公，您粮草送的太多了吃不完。</p><p><strong>第三种是时间超时</strong>，也就是超过网络包的生存时间还是没到。小兵：报告主公，送粮草的人，自己把粮草吃完了，还没找到地方，已经饿死啦。</p><p><strong>第四种是路由重定向</strong>，也就是让下次发给另一个路由器。小兵：报告主公，上次送粮草的人本来只要走一站地铁，非得从五环绕，下次别这样了啊。</p><p>差错报文的结构相对复杂一些。除了前面还是IP，ICMP的前8字节不变，后面则跟上出错的那个IP包的IP头和IP正文的前8个字节。</p><p>而且这类侦查兵特别恪尽职守，不但自己返回来报信，还把一部分遗物也带回来。</p><ul>
<li>侦察兵：报告主公，张将军已经战死沙场，这是张将军的印信和佩剑。</li>
<li>主公：神马？张将军是怎么死的（可以查看ICMP的前8字节）？没错，这是张将军的剑，是他的剑（IP数据包的头及正文前8字节）。</li>
</ul><h2>ping：查询报文类型的使用</h2><p>接下来，我们重点来看ping的发送和接收过程。</p><p><img src="https://static001.geekbang.org/resource/image/57/21/57a77fb89bc4a5653842276c70c0d621.jpg?wh=4463*2786" alt=""></p><p>假定主机A的IP地址是192.168.1.1，主机B的IP地址是192.168.1.2，它们都在同一个子网。那当你在主机A上运行“ping 192.168.1.2”后，会发生什么呢?</p><p>ping命令执行的时候，源主机首先会构建一个ICMP请求数据包，ICMP数据包内包含多个字段。最重要的是两个，第一个是<strong>类型字段</strong>，对于请求数据包而言该字段为 8；另外一个是<strong>顺序号</strong>，主要用于区分连续ping的时候发出的多个数据包。每发出一个请求数据包，顺序号会自动加1。为了能够计算往返时间RTT，它会在报文的数据部分插入发送时间。</p><p>然后，由ICMP协议将这个数据包连同地址192.168.1.2一起交给IP层。IP层将以192.168.1.2作为目的地址，本机IP地址作为源地址，加上一些其他控制信息，构建一个IP数据包。</p><p>接下来，需要加入MAC头。如果在本节ARP映射表中查找出IP地址192.168.1.2所对应的MAC地址，则可以直接使用；如果没有，则需要发送ARP协议查询MAC地址，获得MAC地址后，由数据链路层构建一个数据帧，目的地址是IP层传过来的MAC地址，源地址则是本机的MAC地址；还要附加上一些控制信息，依据以太网的介质访问规则，将它们传送出去。</p><p>主机B收到这个数据帧后，先检查它的目的MAC地址，并和本机的MAC地址对比，如符合，则接收，否则就丢弃。接收后检查该数据帧，将IP数据包从帧中提取出来，交给本机的IP层。同样，IP层检查后，将有用的信息提取后交给ICMP协议。</p><p>主机B会构建一个 ICMP 应答包，应答数据包的类型字段为 0，顺序号为接收到的请求数据包中的顺序号，然后再发送出去给主机A。</p><p>在规定的时候间内，源主机如果没有接到 ICMP 的应答包，则说明目标主机不可达；如果接收到了 ICMP 应答包，则说明目标主机可达。此时，源主机会检查，用当前时刻减去该数据包最初从源主机上发出的时刻，就是 ICMP 数据包的时间延迟。</p><p>当然这只是最简单的，同一个局域网里面的情况。如果跨网段的话，还会涉及网关的转发、路由器的转发等等。但是对于ICMP的头来讲，是没什么影响的。会影响的是根据目标IP地址，选择路由的下一跳，还有每经过一个路由器到达一个新的局域网，需要换MAC头里面的MAC地址。这个过程后面几节会详细描述，这里暂时不多说。</p><p>如果在自己的可控范围之内，当遇到网络不通的问题的时候，除了直接ping目标的IP地址之外，还应该有一个清晰的网络拓扑图。并且从理论上来讲，应该要清楚地知道一个网络包从源地址到目标地址都需要经过哪些设备，然后逐个ping中间的这些设备或者机器。如果可能的话，在这些关键点，通过tcpdump -i eth0 icmp，查看包有没有到达某个点，回复的包到达了哪个点，可以更加容易推断出错的位置。</p><p>经常会遇到一个问题，如果不在我们的控制范围内，很多中间设备都是禁止ping的，但是ping不通不代表网络不通。这个时候就要使用telnet，通过其他协议来测试网络是否通，这个就不在本篇的讲述范围了。</p><p>说了这么多，你应该可以看出ping这个程序是使用了ICMP里面的ECHO REQUEST和ECHO REPLY类型的。</p><h2>Traceroute：差错报文类型的使用</h2><p>那其他的类型呢？是不是只有真正遇到错误的时候，才能收到呢？那也不是，有一个程序Traceroute，是个“大骗子”。它会使用ICMP的规则，故意制造一些能够产生错误的场景。</p><p>所以，<strong>Traceroute的第一个作用就是故意设置特殊的TTL，来追踪去往目的地时沿途经过的路由器</strong>。Traceroute的参数指向某个目的IP地址，它会发送一个UDP的数据包。将TTL设置成1，也就是说一旦遇到一个路由器或者一个关卡，就表示它“牺牲”了。</p><p>如果中间的路由器不止一个，当然碰到第一个就“牺牲”。于是，返回一个ICMP包，也就是网络差错包，类型是时间超时。那大军前行就带一顿饭，试一试走多远会被饿死，然后找个哨探回来报告，那我就知道大军只带一顿饭能走多远了。</p><p>接下来，将TTL设置为2。第一关过了，第二关就“牺牲”了，那我就知道第二关有多远。如此反复，直到到达目的主机。这样，Traceroute就拿到了所有的路由器IP。当然，有的路由器压根不会回这个ICMP。这也是Traceroute一个公网的地址，看不到中间路由的原因。</p><p>怎么知道UDP有没有到达目的主机呢？Traceroute程序会发送一份UDP数据报给目的主机，但它会选择一个不可能的值作为UDP端口号（大于30000）。当该数据报到达时，将使目的主机的 UDP模块产生一份“端口不可达”错误ICMP报文。如果数据报没有到达，则可能是超时。</p><p>这就相当于故意派人去西天如来那里去请一本《道德经》，结果人家信佛不信道，消息就会被打出来。被打的消息传回来，你就知道西天是能够到达的。为什么不去取《心经》呢？因为UDP是无连接的。也就是说这人一派出去，你就得不到任何音信。你无法区别到底是半路走丢了，还是真的信佛遁入空门了，只有让人家打出来，你才会得到消息。</p><p><strong>Traceroute还有一个作用是故意设置不分片，从而确定路径的MTU。</strong>要做的工作首先是发送分组，并设置“不分片”标志。发送的第一个分组的长度正好与出口MTU相等。如果中间遇到窄的关口会被卡住，会发送ICMP网络差错包，类型为“需要进行分片但设置了不分片位”。其实，这是人家故意的好吧，每次收到ICMP“不能分片”差错时就减小分组的长度，直到到达目标主机。</p><h2>小结</h2><p>好了，这一节内容差不多了，我来总结一下：</p><ul>
<li>ICMP相当于网络世界的侦察兵。我讲了两种类型的ICMP报文，一种是主动探查的查询报文，一种异常报告的差错报文；</li>
<li>ping使用查询报文，Traceroute使用差错报文。</li>
</ul><p>最后，给你留两个思考题吧。</p><ol>
<li>当发送的报文出问题的时候，会发送一个ICMP的差错报文来报告错误，但是如果ICMP的差错报文也出问题了呢？</li>
<li>这一节只说了一个局域网互相ping的情况。如果跨路由器、跨网关的过程会是什么样的呢？</li>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/21/0f/b61dc67e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>^_^</span>
  </div>
  <div class="_2_QraFYR_0">一点个人感觉:老师的比喻有点太用力了，不如简单通俗的讲解原理.<br>另外错别字和语句通顺程度是会影响理解的，前几期有时也有这个感觉。<br>专业水平老师还是很牛的，学到很多知识。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-03 10:55:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/49/b8/fb19aa6a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Marnie</span>
  </div>
  <div class="_2_QraFYR_0">TTL的作用没有讲清楚，特此记录一下。<br>TTL是网络包里的一个值，它告诉路由器包在网络中太长时间是否需要被丢弃。TTL最初的设想是，设置超时时间，超过此范围则被丢弃。每个路由器要将TTL减一，TTL通常表示被丢弃前经过的路由器的个数。当TTL变为0时，该路由器丢弃该包，并发送一个ICMP包给最初的发送者。 tranceroute差错报文会使用</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-29 16:47:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/ec/1c/d323b066.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>knull</span>
  </div>
  <div class="_2_QraFYR_0">许多人问：tracerouter发udp，为啥出错回icmp？<br>正常情况下，协议栈能正常走到udp，当然正常返回udp。<br>但是，你主机不可达，是ip层的（还没到udp）。ip层，当然只知道回icmp。报文分片错误也是同理。<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-12 09:26:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/c2/14/bfd5e94e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>简</span>
  </div>
  <div class="_2_QraFYR_0">老师我觉得跟多人说你比喻太抽象，我觉得他们底子太差，需要恶补。我觉得你比喻真好！我原来把一本网络基础书看完了，所以一说就懂，你这么说我觉得我六脉打通了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-16 23:22:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/d6/42/bafec9b7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>盖</span>
  </div>
  <div class="_2_QraFYR_0">好像不发ICMP差错报文的一种情况就是ICMP 的差错报文出错，其它还有目的地址为广播时或者源地址不唯一时也不发差错报文。<br><br>前面评论问udp为什么返回ICMP报文的，ICMP一般认为属于网络层的，和IP同一层，是管理和控制IP的一种协议，而UDP和TCP是传输层，所以UDP出错可以返回ICMP差错报文</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-01 11:24:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/08/1c/ef15e661.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span> 臣馟飞扬</span>
  </div>
  <div class="_2_QraFYR_0">百度一下traceroute的原理也很简单啊，看了比喻反而更乱了，希望不是为了比喻而比喻：<br>Traceroute程序的设计是利用ICMP及IP header的TTL（Time To Live）栏位（field）。首先，traceroute送出一个TTL是1的IP datagram（其实，每次送出的为3个40字节的包，包括源地址，目的地址和包发出的时间标签）到目的地，当路径上的第一个路由器（router）收到这个datagram时，它将TTL减1。此时，TTL变为0了，所以该路由器会将此datagram丢掉，并送回一个「ICMP time exceeded」消息（包括发IP包的源地址，IP包的所有内容及路由器的IP地址），traceroute 收到这个消息后，便知道这个路由器存在于这个路径上，接着traceroute 再送出另一个TTL是2 的datagram，发现第2 个路由器...... traceroute 每次将送出的datagram的TTL 加1来发现另一个路由器，这个重复的动作一直持续到某个datagram 抵达目的地。当datagram到达目的地后，该主机并不会送回ICMP time exceeded消息，因为它已是目的地了，那么traceroute如何得知目的地到达了呢？<br><br>Traceroute在送出UDP datagrams到目的地时，它所选择送达的port number 是一个一般应用程序都不会用的号码（30000 以上），所以当此UDP datagram 到达目的地后该主机会送回一个「ICMP port unreachable」的消息，而当traceroute 收到这个消息时，便知道目的地已经到达了。所以traceroute 在Server端也是没有所谓的Daemon 程式。<br><br>Traceroute提取发 ICMP TTL到期消息设备的IP地址并作域名解析。每次 ，Traceroute都打印出一系列数据,包括所经过的路由设备的域名及 IP地址,三个包每次来回所花时间。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-10 20:21:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/cd/17/59d4d531.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>秦俊山</span>
  </div>
  <div class="_2_QraFYR_0">非计算机专业出身的，听得一脸懵逼。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-02 08:53:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/52/0e/abb7bfe3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>手撕油条</span>
  </div>
  <div class="_2_QraFYR_0">越听越想听，一周三更听不爽</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-01 08:57:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/49/86/554cc50e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张张张 💭</span>
  </div>
  <div class="_2_QraFYR_0">看了这么多，我觉得可以先说原理性描述，然后再比喻，这样可以对比着理解，这样应该更好些</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-27 11:26:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/51/19/86f30954.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>番茄尼玛</span>
  </div>
  <div class="_2_QraFYR_0">同觉得比喻有些用力过猛了，希望老师能在比喻之后再用技术语言解释一下。比如这篇文章中，终点不可达的几个场景，我就没想通网络不可达和主机不可达有什么区别，还有协议不可达和端口不可达有什么区别</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-15 17:45:35</div>
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
  <div class="_2_QraFYR_0">老师，最好能够抓包截图给大家看，有图有真相，可惜留言不能发图片，不然我就把ping 和traceroute的包细节发出来给大家看看，哈哈</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-26 11:24:51</div>
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
  <div class="_2_QraFYR_0">我觉得比喻很好，方便理解，聪明的人才愿意打比方</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-02 21:12:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/3a/6e/e39e90ca.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>大坏狐狸</span>
  </div>
  <div class="_2_QraFYR_0">1.对于携带ICMP差错报文的数据报，不再产生ICMP差错报文。 <br>  如果主机A发送了一个ICMP的数据报文给主机B，数据在传输过程中经过其中一个路由器出现错误，由于该路由器已经接收到一个ICMP数据报文，所以不会再产生一个ICMP差错报文。<br><br>2.对于分片的数据报，如果不是第一个分片，则不产生ICMP差错报文 <br>  对于主机A发送了一个分片的数据，如果路由设备或主机接收到的分片数据不是第一个分片数据，不会产生ICMP差错报文。<br><br>3.对于具有多播地址的数据报，不产生ICMP差错报文 <br>  如果一个ip地址是一个广播地址的话，不会产生ICMP差错报文。<br><br>4.对于具有特殊地址如（127.0.0.0或0.0.0.0）的数据报，不产生ICMP差错报文<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-02 16:31:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/04/8d/005c2ff3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>weineel</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，以我的经验使用tcp&#47;udp协议可以用scoket api编程，但是文中提到ping程序使用了ip协议，那ping程序的实现是不是用到了其他网络编程接口？<br><br>换句话说，每一层的协议都有相应的api可以调用吗？有的话又是什么呢？<br><br>但又感觉我们写的程序一般是应用层用socket就行了…</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不是每一层都有接口的，tcp ip协议栈在内核里面，内核的最外层是系统调用，所以你看起来就只能调用socket了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-02 07:46:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/49/b8/fb19aa6a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Marnie</span>
  </div>
  <div class="_2_QraFYR_0">差错报文就是故意找茬，制造错误让别人打回去。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-29 16:17:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/90/26/6a37f007.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>YXsong</span>
  </div>
  <div class="_2_QraFYR_0">没基础的小白听的云里雾里呀，好难🤯</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-07-22 22:23:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/52/bb/225e70a6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>hunterlodge</span>
  </div>
  <div class="_2_QraFYR_0">差错报文中的端口不可达是指什么端口呢？为什么ICMP协议会有端口概念呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-16 08:15:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/11/4b/fa64f061.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>xfan</span>
  </div>
  <div class="_2_QraFYR_0">如果网络不可达，是谁回复的差错报文呢，是网关吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 网络不可达也是到达了某个地方，发现走不下去了，到哪里哪里返回，一般是某个路由器</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-09 17:17:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/49/b8/fb19aa6a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Marnie</span>
  </div>
  <div class="_2_QraFYR_0">感觉查询报文就像编程中的正常流程，差错报文相当于测试程序。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-04 11:41:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/fb/51/870a6fcb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Trust me ҉҉҉҉҉҉҉❀</span>
  </div>
  <div class="_2_QraFYR_0">tracerouter发送的包是什么程序给的响应？  是目标主机的traceroute程序？还是ICMP</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 没有服务端，全部在制造错误</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-01 10:55:09</div>
  </div>
</div>
</div>
</li>
</ul>