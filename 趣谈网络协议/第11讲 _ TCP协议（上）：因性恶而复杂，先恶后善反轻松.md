<audio title="第11讲 _ TCP协议（上）：因性恶而复杂，先恶后善反轻松" src="https://static001.geekbang.org/resource/audio/50/32/50f570ae1182a6dd989401393329cc32.mp3" controls="controls"></audio> 
<p>上一节，我们讲的UDP，基本上包括了传输层所必须的端口字段。它就像我们小时候一样简单，相信“网之初，性本善，不丢包，不乱序”。</p><p>后来呢，我们都慢慢长大，了解了社会的残酷，变得复杂而成熟，就像TCP协议一样。它之所以这么复杂，那是因为它秉承的是“性恶论”。它天然认为网络环境是恶劣的，丢包、乱序、重传，拥塞都是常有的事情，一言不合就可能送达不了，因而要从算法层面来保证可靠性。</p><h2>TCP包头格式</h2><p>我们先来看TCP头的格式。从这个图上可以看出，它比UDP复杂得多。</p><p><img src="https://static001.geekbang.org/resource/image/64/bf/642947c94d6682a042ad981bfba39fbf.jpg?wh=2203*1618" alt=""></p><p>首先，源端口号和目标端口号是不可少的，这一点和UDP是一样的。如果没有这两个端口号。数据就不知道应该发给哪个应用。</p><p>接下来是包的序号。为什么要给包编号呢？当然是为了解决乱序的问题。不编好号怎么确认哪个应该先来，哪个应该后到呢。编号是为了解决乱序问题。既然是社会老司机，做事当然要稳重，一件件来，面临再复杂的情况，也临危不乱。</p><p>还应该有的就是确认序号。发出去的包应该有确认，要不然我怎么知道对方有没有收到呢？如果没有收到就应该重新发送，直到送达。这个可以解决不丢包的问题。作为老司机，做事当然要靠谱，答应了就要做到，暂时做不到也要有个回复。</p><p>TCP是靠谱的协议，但是这不能说明它面临的网络环境好。从IP层面来讲，如果网络状况的确那么差，是没有任何可靠性保证的，而作为IP的上一层TCP也无能为力，唯一能做的就是更加努力，不断重传，通过各种算法保证。也就是说，对于TCP来讲，IP层你丢不丢包，我管不着，但是我在我的层面上，会努力保证可靠性。</p><!-- [[[read_end]]] --><p>这有点像如果你在北京，和客户约十点见面，那么你应该清楚堵车是常态，你干预不了，也控制不了，你唯一能做的就是早走。打车不行就改乘地铁，尽力不失约。</p><p>接下来有一些状态位。例如SYN是发起一个连接，ACK是回复，RST是重新连接，FIN是结束连接等。TCP是面向连接的，因而双方要维护连接的状态，这些带状态位的包的发送，会引起双方的状态变更。</p><p>不像小时候，随便一个不认识的小朋友都能玩在一起，人大了，就变得礼貌，优雅而警觉，人与人遇到会互相热情的寒暄，离开会不舍地道别，但是人与人之间的信任会经过多次交互才能建立。</p><p>还有一个重要的就是窗口大小。TCP要做流量控制，通信双方各声明一个窗口，标识自己当前能够的处理能力，别发送的太快，撑死我，也别发的太慢，饿死我。</p><p>作为老司机，做事情要有分寸，待人要把握尺度，既能适当提出自己的要求，又不强人所难。除了做流量控制以外，TCP还会做拥塞控制，对于真正的通路堵车不堵车，它无能为力，唯一能做的就是控制自己，也即控制发送的速度。不能改变世界，就改变自己嘛。</p><p>作为老司机，要会自我控制，知进退，知道什么时候应该坚持，什么时候应该让步。</p><p>通过对TCP头的解析，我们知道要掌握TCP协议，重点应该关注以下几个问题：</p><ul>
<li>
<p>顺序问题 ，稳重不乱；</p>
</li>
<li>
<p>丢包问题，承诺靠谱；</p>
</li>
<li>
<p>连接维护，有始有终；</p>
</li>
<li>
<p>流量控制，把握分寸；</p>
</li>
<li>
<p>拥塞控制，知进知退。</p>
</li>
</ul><h2>TCP的三次握手</h2><p>所有的问题，首先都要先建立一个连接，所以我们先来看连接维护问题。</p><p>TCP的连接建立，我们常常称为三次握手。</p><p>A：您好，我是A。</p><p>B：您好A，我是B。</p><p>A：您好B。</p><p>我们也常称为“请求-&gt;应答-&gt;应答之应答”的三个回合。这个看起来简单，其实里面还是有很多的学问，很多的细节。</p><p>首先，为什么要三次，而不是两次？按说两个人打招呼，一来一回就可以了啊？为了可靠，为什么不是四次？</p><p>我们还是假设这个通路是非常不可靠的，A要发起一个连接，当发了第一个请求杳无音信的时候，会有很多的可能性，比如第一个请求包丢了，再如没有丢，但是绕了弯路，超时了，还有B没有响应，不想和我连接。</p><p>A不能确认结果，于是再发，再发。终于，有一个请求包到了B，但是请求包到了B的这个事情，目前A还是不知道的，A还有可能再发。</p><p>B收到了请求包，就知道了A的存在，并且知道A要和它建立连接。如果B不乐意建立连接，则A会重试一阵后放弃，连接建立失败，没有问题；如果B是乐意建立连接的，则会发送应答包给A。</p><p>当然对于B来说，这个应答包也是一入网络深似海，不知道能不能到达A。这个时候B自然不能认为连接是建立好了，因为应答包仍然会丢，会绕弯路，或者A已经挂了都有可能。</p><p>而且这个时候B还能碰到一个诡异的现象就是，A和B原来建立了连接，做了简单通信后，结束了连接。还记得吗？A建立连接的时候，请求包重复发了几次，有的请求包绕了一大圈又回来了，B会认为这也是一个正常的的请求的话，因此建立了连接，可以想象，这个连接不会进行下去，也没有个终结的时候，纯属单相思了。因而两次握手肯定不行。</p><p>B发送的应答可能会发送多次，但是只要一次到达A，A就认为连接已经建立了，因为对于A来讲，他的消息有去有回。A会给B发送应答之应答，而B也在等这个消息，才能确认连接的建立，只有等到了这个消息，对于B来讲，才算它的消息有去有回。</p><p>当然A发给B的应答之应答也会丢，也会绕路，甚至B挂了。按理来说，还应该有个应答之应答之应答，这样下去就没底了。所以四次握手是可以的，四十次都可以，关键四百次也不能保证就真的可靠了。只要双方的消息都有去有回，就基本可以了。</p><p>好在大部分情况下，A和B建立了连接之后，A会马上发送数据的，一旦A发送数据，则很多问题都得到了解决。例如A发给B的应答丢了，当A后续发送的数据到达的时候，B可以认为这个连接已经建立，或者B压根就挂了，A发送的数据，会报错，说B不可达，A就知道B出事情了。</p><p>当然你可以说A比较坏，就是不发数据，建立连接后空着。我们在程序设计的时候，可以要求开启keepalive机制，即使没有真实的数据包，也有探活包。</p><p>另外，你作为服务端B的程序设计者，对于A这种长时间不发包的客户端，可以主动关闭，从而空出资源来给其他客户端使用。</p><p>三次握手除了双方建立连接外，主要还是为了沟通一件事情，就是<strong>TCP包的序号的问题</strong>。</p><p>A要告诉B，我这面发起的包的序号起始是从哪个号开始的，B同样也要告诉A，B发起的包的序号起始是从哪个号开始的。为什么序号不能都从1开始呢？因为这样往往会出现冲突。</p><p>例如，A连上B之后，发送了1、2、3三个包，但是发送3的时候，中间丢了，或者绕路了，于是重新发送，后来A掉线了，重新连上B后，序号又从1开始，然后发送2，但是压根没想发送3，但是上次绕路的那个3又回来了，发给了B，B自然认为，这就是下一个包，于是发生了错误。</p><p>因而，每个连接都要有不同的序号。这个序号的起始序号是随着时间变化的，可以看成一个32位的计数器，每4微秒加一，如果计算一下，如果到重复，需要4个多小时，那个绕路的包早就死翘翘了，因为我们都知道IP包头里面有个TTL，也即生存时间。</p><p>好了，双方终于建立了信任，建立了连接。前面也说过，为了维护这个连接，双方都要维护一个状态机，在连接建立的过程中，双方的状态变化时序图就像这样。</p><p><img src="https://static001.geekbang.org/resource/image/c0/08/c067fe62f49e8152368c7be9d91adc08.jpg?wh=1693*1093" alt=""></p><p>一开始，客户端和服务端都处于CLOSED状态。先是服务端主动监听某个端口，处于LISTEN状态。然后客户端主动发起连接SYN，之后处于SYN-SENT状态。服务端收到发起的连接，返回SYN，并且ACK客户端的SYN，之后处于SYN-RCVD状态。客户端收到服务端发送的SYN和ACK之后，发送ACK的ACK，之后处于ESTABLISHED状态，因为它一发一收成功了。服务端收到ACK的ACK之后，处于ESTABLISHED状态，因为它也一发一收了。</p><h2>TCP四次挥手</h2><p>好了，说完了连接，接下来说一说“拜拜”，好说好散。这常被称为四次挥手。</p><p>A：B啊，我不想玩了。</p><p>B：哦，你不想玩了啊，我知道了。</p><p>这个时候，还只是A不想玩了，也即A不会再发送数据，但是B能不能在ACK的时候，直接关闭呢？当然不可以了，很有可能A是发完了最后的数据就准备不玩了，但是B还没做完自己的事情，还是可以发送数据的，所以称为半关闭的状态。</p><p>这个时候A可以选择不再接收数据了，也可以选择最后再接收一段数据，等待B也主动关闭。</p><p>B：A啊，好吧，我也不玩了，拜拜。</p><p>A：好的，拜拜。</p><p>这样整个连接就关闭了。但是这个过程有没有异常情况呢？当然有，上面是和平分手的场面。</p><p>A开始说“不玩了”，B说“知道了”，这个回合，是没什么问题的，因为在此之前，双方还处于合作的状态，如果A说“不玩了”，没有收到回复，则A会重新发送“不玩了”。但是这个回合结束之后，就有可能出现异常情况了，因为已经有一方率先撕破脸。</p><p>一种情况是，A说完“不玩了”之后，直接跑路，是会有问题的，因为B还没有发起结束，而如果A跑路，B就算发起结束，也得不到回答，B就不知道该怎么办了。另一种情况是，A说完“不玩了”，B直接跑路，也是有问题的，因为A不知道B是还有事情要处理，还是过一会儿会发送结束。</p><p>那怎么解决这些问题呢？TCP协议专门设计了几个状态来处理这些问题。我们来看断开连接的时候的<strong>状态时序图</strong>。</p><p><img src="https://static001.geekbang.org/resource/image/bf/13/bf1254f85d527c77cc4088a35ac11d13.jpg?wh=1693*1534" alt=""></p><p>断开的时候，我们可以看到，当A说“不玩了”，就进入FIN_WAIT_1的状态，B收到“A不玩”的消息后，发送知道了，就进入CLOSE_WAIT的状态。</p><p>A收到“B说知道了”，就进入FIN_WAIT_2的状态，如果这个时候B直接跑路，则A将永远在这个状态。TCP协议里面并没有对这个状态的处理，但是Linux有，可以调整tcp_fin_timeout这个参数，设置一个超时时间。</p><p>如果B没有跑路，发送了“B也不玩了”的请求到达A时，A发送“知道B也不玩了”的ACK后，从FIN_WAIT_2状态结束，按说A可以跑路了，但是最后的这个ACK万一B收不到呢？则B会重新发一个“B不玩了”，这个时候A已经跑路了的话，B就再也收不到ACK了，因而TCP协议要求A最后等待一段时间TIME_WAIT，这个时间要足够长，长到如果B没收到ACK的话，“B说不玩了”会重发的，A会重新发一个ACK并且足够时间到达B。</p><p>A直接跑路还有一个问题是，A的端口就直接空出来了，但是B不知道，B原来发过的很多包很可能还在路上，如果A的端口被一个新的应用占用了，这个新的应用会收到上个连接中B发过来的包，虽然序列号是重新生成的，但是这里要上一个双保险，防止产生混乱，因而也需要等足够长的时间，等到原来B发送的所有的包都死翘翘，再空出端口来。</p><p>等待的时间设为2MSL，<strong>MSL</strong>是<strong>Maximum Segment Lifetime</strong>，<strong>报文最大生存时间</strong>，它是任何报文在网络上存在的最长时间，超过这个时间报文将被丢弃。因为TCP报文基于是IP协议的，而IP头中有一个TTL域，是IP数据报可以经过的最大路由数，每经过一个处理他的路由器此值就减1，当此值为0则数据报将被丢弃，同时发送ICMP报文通知源主机。协议规定MSL为2分钟，实际应用中常用的是30秒，1分钟和2分钟等。</p><p>还有一个异常情况就是，B超过了2MSL的时间，依然没有收到它发的FIN的ACK，怎么办呢？按照TCP的原理，B当然还会重发FIN，这个时候A再收到这个包之后，A就表示，我已经在这里等了这么长时间了，已经仁至义尽了，之后的我就都不认了，于是就直接发送RST，B就知道A早就跑了。</p><h2>TCP状态机</h2><p>将连接建立和连接断开的两个时序状态图综合起来，就是这个著名的TCP的状态机。学习的时候比较建议将这个状态机和时序状态机对照着看，不然容易晕。</p><p><img src="https://static001.geekbang.org/resource/image/fd/2a/fd45f9ad6ed575ea6bfdaafeb3bfb62a.jpg?wh=2447*2684" alt=""></p><p>在这个图中，加黑加粗的部分，是上面说到的主要流程，其中阿拉伯数字的序号，是连接过程中的顺序，而大写中文数字的序号，是连接断开过程中的顺序。加粗的实线是客户端A的状态变迁，加粗的虚线是服务端B的状态变迁。</p><h2>小结</h2><p>好了，这一节就到这里了，我来做一个总结：</p><ul>
<li>
<p>TCP包头很复杂，但是主要关注五个问题，顺序问题，丢包问题，连接维护，流量控制，拥塞控制；</p>
</li>
<li>
<p>连接的建立是经过三次握手，断开的时候四次挥手，一定要掌握的我画的那个状态图。</p>
</li>
</ul><p>最后，给你留两个思考题。</p><ol>
<li>
<p>TCP的连接有这么多的状态，你知道如何在系统中查看某个连接的状态吗？</p>
</li>
<li>
<p>这一节仅仅讲了连接维护问题，其实为了维护连接的状态，还有其他的数据结构来处理其他的四个问题，那你知道是什么吗？</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/34/e1/d1f201ae.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>code4j</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，昨天阿里云出故障，恢复后，我们调用阿里云服务的时后出现了调用出异常  connection reset。netstat看了下这个ip发现都是timewait，链接不多，但是始终无法连接放对方的服务。按照今天的内容，难道是我的程序关闭主动关闭链接后没有发出最后的ack吗？之前都没有问题，很不解</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-28 10:05:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/87/dd/4f53f95d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>进阶的码农</span>
  </div>
  <div class="_2_QraFYR_0">状态机图里的不加粗虚线看不懂什么意思 麻烦老师点拨下</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 其他非主流过程</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-11 20:18:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/63/14/06eff9a4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Jerry银银</span>
  </div>
  <div class="_2_QraFYR_0">看了大家的留言，很有感触，对于同一个技术点，技术的原理是一样的，可是每个人都有自己的理解，和自己独有的知识关联，因为每个人过往所积累的知识体系有所不同。<br><br>我对TCP，甚至整个网络知识，喜欢用「信息论」中的一些理论去推演它们。我觉得在现在和未来的时代，「信息论」是一把利器。<br><br>信息论中，有个很重要的思想：要想消除信息的不确定性，就得引入信息。将这个思想应用到TCP中，很容易理解TCP的三次握手和四次挥手的必要性：它们的存在以及复杂度，就是为了消除不确定性，这里我们叫「不可靠性」吧。拿三次握手举例：<br>为了描述方便，将通信的两端用字母A和B替代。A要往B发数据，A要确定两件事：<br>1. B在“那儿”，并且能接受数据 —— B确实存在，并且是个“活人”，能听得见<br>2. B能回应  —— B能发数据，能说话<br>为了消除这两个不确定性，所以必须有前两次握手，即A发送了数据，B收到了，并且能回应——“ACK”<br><br>同样的，对于B来说，它也要消除以上两个不确定性，通过前两次握手，B知道了A能说，但是不能确定A能听，这就是第三次握手的必要性。<br><br>当然你可能会问，增加第四次握手有没有必要？从信息论的角度来说，已经不需要了，因为它的增加也无法再提高「确定性」</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-17 10:31:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/52/46/3d4f32f7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>TaoLR</span>
  </div>
  <div class="_2_QraFYR_0">旧书不厌百回读，熟读深思子自知啊。<br><br>再一次认识下TCP这位老司机，这可能是我读过的最好懂的讲TCP的文章了。同时有了一个很爽的触点，我发现但凡复杂点儿的东西，状态数据都复杂很多。还有一个，也是最重要的，我从TCP中再一次认识到了一个做人的道理，像孔子说的：“不怨天，不尤人”。人与人相处，主要是“我，你，我和你的关系”这三个处理对象，“你”这个我管不了，“我”的成长亦需要时间，我想到的是“我和你的关系”，这个状态的维护，如果能“无尤”，即不抱怨。有时候自己做多点儿，更靠谱点，那人和人的这个连接不是会更靠谱么？这可能是我从TCP这儿学到的最棒的东西了。<br><br>感谢老师的讲解，让我有了新的想法和收获。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-23 15:12:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/f5/0a/077b9922.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>krugle</span>
  </div>
  <div class="_2_QraFYR_0">流量控制和拥塞控制什么区别</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 一个是对另一端的，一个是针对网络的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-17 23:49:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/a0/57/3a729755.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>灯盖</span>
  </div>
  <div class="_2_QraFYR_0">流量控制是照顾通信对象<br>拥塞控制是照顾通信环境</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个总结好</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-13 07:13:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/pc41FOKAiabVaaKiawibEm7zglvnsYBnYeRiaSAElf9ciczovXmXmI0hOeR6U9RULFtMoqX5kobNttvwXCLsUM9Hbcg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>monkay</span>
  </div>
  <div class="_2_QraFYR_0">如果是建立链接了，数据传输过程链接断了，客户端和服务器端各自会是什么状态？<br>或者我可以这样理解么，所谓的链接根本是不存在的，双方握手之后，数据传输还是跟udp一样，只是tcp在维护顺序、流量之类的控制</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，连接就是两端的状态维护，中间过程没有所谓的连接，一旦传输失败，一端收到消息，才知道状态的变化</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-11 12:10:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/45/f5/b3d7febf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Adolph</span>
  </div>
  <div class="_2_QraFYR_0">为什么要三次握手<br>三次握手的目的是建立可靠的通信信道，说到通讯，简单来说就是数据的发送与接收，而三次握手最主要的目的就是双方确认自己与对方的发送与接收是正常的。<br><br>第一次握手：Client 什么都不能确认；Server 确认了对方发送正常<br><br>第二次握手：Client 确认了：自己发送、接收正常，对方发送、接收正常；Server 确认了：自己接收正常，对方发送正常<br><br>第三次握手：Client 确认了：自己发送、接收正常，对方发送、接收正常；Server 确认了：自己发送、接收正常，对方发送接收正常<br><br>所以三次握手就能确认双发收发功能都正常，缺一不可。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-27 17:10:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/45/f5/b3d7febf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Adolph</span>
  </div>
  <div class="_2_QraFYR_0">为什么要四次挥手<br>任何一方都可以在数据传送结束后发出连接释放的通知，待对方确认后进入半关闭状态。当另一方也没有数据再发送的时候，则发出连接释放通知，对方确认后就完全关闭了TCP连接。<br><br>举个例子：A 和 B 打电话，通话即将结束后，A 说“我没啥要说的了”，B回答“我知道了”，但是 B 可能还会有要说的话，A 不能要求 B 跟着自己的节奏结束通话，于是 B 可能又巴拉巴拉说了一通，最后 B 说“我说完了”，A 回答“知道了”，这样通话才算结束。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-27 17:10:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/1e/b7/b20ab184.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>麋鹿在泛舟</span>
  </div>
  <div class="_2_QraFYR_0">多谢分享，精彩。扫除了我之前很多的疑问。tcp连接的断开比建立复杂一些，本质上是因为资源的申请（初始化）本身就比资源的释放简单，以c++为例，构造函数初始化对象很简单，而析构函数则要考虑所有资源安全有序的释放，tcp断连时序中除了断开这一重要动作，另外重要的潜台词是“我要断开连接了 你感觉把收尾工作做了”</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-11 08:59:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/1e/b7/b20ab184.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>麋鹿在泛舟</span>
  </div>
  <div class="_2_QraFYR_0">评论区不能互评，如下两个问题是我的看法，不对请指出 多谢<br><br><br>我们做一个基于tcp的“物联网”应用（中国移动网络），如上面所说tcp层面已经会自动重传数据了，业务层面上还有必要再重传吗？如果是的话，业务需要多久重传一次？<br><br>--- TCP的重传是网络层面和粒度的，业务层面需要看具体业务，比如发送失败可能对端在重启，一个重启时间是1min，那就没有必要每秒都发送检测啊.<br><br>1、‘序号的起始序号随时间变化，...重复需要4个多小时’，老师这个重复时间怎么计算出来的呢？每4ms加1，如果有两个TCP链接都在这个4ms内建立，是不是就是相同的起始序列号呢。<br>答:序号的随时间变化，主要是为了区分同一个链接发送序号混淆的问题，两个链接的话，端口或者IP肯定都不一样了.2、报文最大生存时间（MSL）和IP协议的路由条数（TTL）什么关系呢，报文当前耗时怎么计算？TCP层有存储相应时间？<br>答:都和报文生存有关，前者是时间维度的概念，后者是经过路由跳数，不是时间单位.<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-14 07:29:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/60/be/68ce2fd0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小田</span>
  </div>
  <div class="_2_QraFYR_0">人大了，就变得礼貌，优雅而警觉</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-12 08:53:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/47/66/29a9acec.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>挖坑的张师傅</span>
  </div>
  <div class="_2_QraFYR_0">可以用 netstat 或者 lsof 命令 grep 一下 establish listen close_wait 等这些查看</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-11 07:55:26</div>
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
  <div class="_2_QraFYR_0">网络_11<br>读完今天呢内容后，有一个强烈的感受：技术的细节非常生动。<br>之前对于TCP的感知就是简单的“三次握手”，“四次挥手”，觉得自己掌握了精髓，但随便一个问题就懵了，比如，<br>- 客户端什么时候建立连接？<br>&gt; 根据以前的认知，会以为是“三次握手”后，双方同时建立连接。很显然是做不到的，客户端不知道“应答的应答”有没有到达，以及什么时候到达。。。<br>- 客户端什么时候断开连接？<br>&gt; 不仔细思考的话，就会说“四次挥手”之后喽，但事实上，客户端发出最后的应答(第四次“挥手”)后，永远无法知道有没有到达。于是有了2MSL的等待，在不确定的网络中，把问题最大程度地解决。<br><br>TCP的状态机，以及很多的设计细节，都是为了解决不稳定的网络问题，让我们看到了在无法改变不稳定的底层网络时，人类的智慧是如果建立一个基本可靠稳定的网络的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-07 16:16:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/67/0e/c77ad9b1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>eason2017</span>
  </div>
  <div class="_2_QraFYR_0">老师好，我看书中说是计数器每4微妙就加1的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-07-04 23:07:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/4d/62/bfbf8ee3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>走过全世界。</span>
  </div>
  <div class="_2_QraFYR_0">三次握手连接就是：<br>A：你瞅啥<br>B：瞅你咋地<br>A：再瞅一个试试<br>然后就可以开始亲密友好互动了😏</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 哈哈哈</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-03 08:43:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/6e/7d/370b0d4c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>西部世界</span>
  </div>
  <div class="_2_QraFYR_0">我也纠结了4ms加一重复一次的时间是4个多小时的问题。<br>具体如下:<br>4ms是4微秒，等于100万分之一秒，32位无符号数的最大值是max=4294967296(注意区分有符号整数最大值)<br>公式如下4&#47;1000000*max&#47;60&#47;60=4.772小时。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-27 16:42:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/54/04/c143ae3c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>korey</span>
  </div>
  <div class="_2_QraFYR_0">敲黑板，这是面试重点</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-07-24 22:06:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/7d/d8/d7c77764.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>HunterYuan</span>
  </div>
  <div class="_2_QraFYR_0">老师回答，@正是那朵玫瑰的第一个问题，感觉稍微有点问题，我理解，不是每次数据交互的会发ack，有些ack是可以合并的，用于减少数据包量，通过wirshark抓包，也证实了。理由是，因为ack号是表示已收到的数据量，也就是说，它是告诉发送方目前已接收的数据的最后一个位置在哪里，因此当需要连续发送ack时，只要发送最后一个ack号就可以了，中间的可以全部省略。同样的机制还有ack号通知和窗口更新通知合并。这是我的理解，若有问题，希望老师多多指教。谢谢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-07-14 11:03:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/fe/c5/3467cf94.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>正是那朵玫瑰</span>
  </div>
  <div class="_2_QraFYR_0">老师有几个疑问：<br>1、当三次握手建立连接后，每次数据交互都还会ack吗？比如建立连接后，客户端发送数据包给服务器端，服务器成功收到数据包后会发送ack给客户端么？<br>2、如果建立连接后，客户端和服务器端没有任何数据交互，双方也不主动关闭连接，理论上这个连接会一直存在么？<br>3、2基础上，如果连接一直会在，双方又没有任何数据交互，若一方突然跑路了，另一方怎么知道对方已经不在了呢？在java scoket编程中，我开发客户端与服务器端代码，双方建立连接后，不发送任何数据，当我强制关闭一端时，另一端会收到一个强制关闭异常，这是如何知道对方已经强制关闭了呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，每次都ack。可以有keepalive，如果不交互，两边都不像释放，那就数据结构一直占用内存，对网络没啥影响。必须是发送数据的时候，才知道跑路的事情。你所谓的强制关是怎么关？有可能还是发送了fin的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-12 08:57:16</div>
  </div>
</div>
</div>
</li>
</ul>