<audio title="第10讲 _ UDP协议：因性善而简单，难免碰到“城会玩”" src="https://static001.geekbang.org/resource/audio/60/83/6046c29969d44eeb559bf99d0a366983.mp3" controls="controls"></audio> 
<p>讲完了IP层以后，接下来我们开始讲传输层。传输层里比较重要的两个协议，一个是TCP，一个是UDP。对于不从事底层开发的人员来讲，或者对于开发应用的人来讲，最常用的就是这两个协议。由于面试的时候，这两个协议经常会被放在一起问，因而我在讲的时候，也会结合着来讲。</p>
<h2>TCP和UDP有哪些区别？</h2>
<p>一般面试的时候我问这两个协议的区别，大部分人会回答，TCP是面向连接的，UDP是面向无连接的。</p>
<p>什么叫面向连接，什么叫无连接呢？在互通之前，面向连接的协议会先建立连接。例如，TCP会三次握手，而UDP不会。为什么要建立连接呢？你TCP三次握手，我UDP也可以发三个包玩玩，有什么区别吗？</p>
<p><strong>所谓的建立连接，是为了在客户端和服务端维护连接，而建立一定的数据结构来维护双方交互的状态，用这样的数据结构来保证所谓的面向连接的特性。</strong></p>
<p>例如，<strong>TCP提供可靠交付</strong>。通过TCP连接传输的数据，无差错、不丢失、不重复、并且按序到达。我们都知道IP包是没有任何可靠性保证的，一旦发出去，就像西天取经，走丢了、被妖怪吃了，都只能随它去。但是TCP号称能做到那个连接维护的程序做的事情，这个下两节我会详细描述。而<strong>UDP继承了IP包的特性，不保证不丢失，不保证按顺序到达。</strong></p>
<p>再如，<strong>TCP是面向字节流的</strong>。发送的时候发的是一个流，没头没尾。IP包可不是一个流，而是一个个的IP包。之所以变成了流，这也是TCP自己的状态维护做的事情。而<strong>UDP继承了IP的特性，基于数据报的，一个一个地发，一个一个地收。</strong></p>
<p>还有<strong>TCP是可以有拥塞控制的</strong>。它意识到包丢弃了或者网络的环境不好了，就会根据情况调整自己的行为，看看是不是发快了，要不要发慢点。<strong>UDP就不会，应用让我发，我就发，管它洪水滔天。</strong></p>
<p>因而<strong>TCP其实是一个有状态服务</strong>，通俗地讲就是有脑子的，里面精确地记着发送了没有，接收到没有，发送到哪个了，应该接收哪个了，错一点儿都不行。而<strong>UDP则是无状态服务</strong>。通俗地说是没脑子的，天真无邪的，发出去就发出去了。</p>
<p>我们可以这样比喻，如果MAC层定义了本地局域网的传输行为，IP层定义了整个网络端到端的传输行为，这两层基本定义了这样的基因：网络传输是以包为单位的，二层叫帧，网络层叫包，传输层叫段。我们笼统地称为包。包单独传输，自行选路，在不同的设备封装解封装，不保证到达。基于这个基因，生下来的孩子UDP完全继承了这些特性，几乎没有自己的思想。</p>
<h2>UDP包头是什么样的？</h2>
<p>我们来看一下UDP包头。</p>
<p>前面章节我已经讲过包的传输过程，这里不再赘述。当我发送的UDP包到达目标机器后，发现MAC地址匹配，于是就取下来，将剩下的包传给处理IP层的代码。把IP头取下来，发现目标IP匹配，接下来呢？这里面的数据包是给谁呢？</p>
<p>发送的时候，我知道我发的是一个UDP的包，收到的那台机器咋知道的呢？所以在IP头里面有个8位协议，这里会存放，数据里面到底是TCP还是UDP，当然这里是UDP。于是，如果我们知道UDP头的格式，就能从数据里面，将它解析出来。解析出来以后呢？数据给谁处理呢？</p>
<p>处理完传输层的事情，内核的事情基本就干完了，里面的数据应该交给应用程序自己去处理，可是一台机器上跑着这么多的应用程序，应该给谁呢？</p>
<p>无论应用程序写的使用TCP传数据，还是UDP传数据，都要监听一个端口。正是这个端口，用来区分应用程序，要不说端口不能冲突呢。两个应用监听一个端口，到时候包给谁呀？所以，按理说，无论是TCP还是UDP包头里面应该有端口号，根据端口号，将数据交给相应的应用程序。</p>
<p><img src="https://static001.geekbang.org/resource/image/2c/84/2c9a109f3be308dea901004a5a3b4c84.jpg?wh=2183*1103" alt="" /></p>
<p>当我们看到UDP包头的时候，发现的确有端口号，有源端口号和目标端口号。因为是两端通信嘛，这很好理解。但是你还会发现，UDP除了端口号，再没有其他的了。和下两节要讲的TCP头比起来，这个简直简单得一塌糊涂啊！</p>
<h2>UDP的三大特点</h2>
<p>UDP就像小孩子一样，有以下这些特点：</p>
<!-- [[[read_end]]] -->
<p>第一，<strong>沟通简单</strong>，不需要一肚子花花肠子（大量的数据结构、处理逻辑、包头字段）。前提是它相信网络世界是美好的，秉承性善论，相信网络通路默认就是很容易送达的，不容易被丢弃的。</p>
<p>第二，<strong>轻信他人</strong>。它不会建立连接，虽然有端口号，但是监听在这个地方，谁都可以传给他数据，他也可以传给任何人数据，甚至可以同时传给多个人数据。</p>
<p>第三，<strong>愣头青，做事不懂权变</strong>。不知道什么时候该坚持，什么时候该退让。它不会根据网络的情况进行发包的拥塞控制，无论网络丢包丢成啥样了，它该怎么发还怎么发。</p>
<h2>UDP的三大使用场景</h2>
<p>基于UDP这种“小孩子”的特点，我们可以考虑在以下的场景中使用。</p>
<p>第一，<strong>需要资源少，在网络情况比较好的内网，或者对于丢包不敏感的应用</strong>。这很好理解，就像如果你是领导，你会让你们组刚毕业的小朋友去做一些没有那么难的项目，打一些没有那么难的客户，或者做一些失败了也能忍受的实验性项目。</p>
<p>我们在第四节讲的DHCP就是基于UDP协议的。一般的获取IP地址都是内网请求，而且一次获取不到IP又没事，过一会儿还有机会。我们讲过PXE可以在启动的时候自动安装操作系统，操作系统镜像的下载使用的TFTP，这个也是基于UDP协议的。在还没有操作系统的时候，客户端拥有的资源很少，不适合维护一个复杂的状态机，而且因为是内网，一般也没啥问题。</p>
<p>第二，<strong>不需要一对一沟通，建立连接，而是可以广播的应用</strong>。咱们小时候人都很简单，大家在班级里面，谁成绩好，谁写作好，应该表扬谁惩罚谁，谁得几个小红花都是当着全班的面讲的，公平公正公开。长大了人心复杂了，薪水、奖金要背靠背，和员工一对一沟通。</p>
<p>UDP的不面向连接的功能，可以使得可以承载广播或者多播的协议。DHCP就是一种广播的形式，就是基于UDP协议的，而广播包的格式前面说过了。</p>
<p>对于多播，我们在讲IP地址的时候，讲过一个D类地址，也即组播地址，使用这个地址，可以将包组播给一批机器。当一台机器上的某个进程想监听某个组播地址的时候，需要发送IGMP包，所在网络的路由器就能收到这个包，知道有个机器上有个进程在监听这个组播地址。当路由器收到这个组播地址的时候，会将包转发给这台机器，这样就实现了跨路由器的组播。</p>
<p>在后面云中网络部分，有一个协议VXLAN，也是需要用到组播，也是基于UDP协议的。</p>
<p>第三，<strong>需要处理速度快，时延低，可以容忍少数丢包，但是要求即便网络拥塞，也毫不退缩，一往无前的时候</strong>。记得曾国藩建立湘军的时候，专门招出生牛犊不怕虎的新兵，而不用那些“老油条”的八旗兵，就是因为八旗兵经历的事情多，遇到敌军不敢舍死忘生。</p>
<p>同理，UDP简单、处理速度快，不像TCP那样，操这么多的心，各种重传啊，保证顺序啊，前面的不收到，后面的没法处理啊。不然等这些事情做完了，时延早就上去了。而TCP在网络不好出现丢包的时候，拥塞控制策略会主动的退缩，降低发送速度，这就相当于本来环境就差，还自断臂膀，用户本来就卡，这下更卡了。</p>
<p>当前很多应用都是要求低时延的，它们可不想用TCP如此复杂的机制，而是想根据自己的场景，实现自己的可靠和连接保证。例如，如果应用自己觉得，有的包丢了就丢了，没必要重传了，就可以算了，有的比较重要，则应用自己重传，而不依赖于TCP。有的前面的包没到，后面的包到了，那就先给客户展示后面的嘛，干嘛非得等到齐了呢？如果网络不好，丢了包，那不能退缩啊，要尽快传啊，速度不能降下来啊，要挤占带宽，抢在客户失去耐心之前到达。</p>
<p>由于UDP十分简单，基本啥都没做，也就给了应用“城会玩”的机会。就像在和平年代，每个人应该有独立的思考和行为，应该可靠并且礼让；但是如果在战争年代，往往不太需要过于独立的思考，而需要士兵简单服从命令就可以了。</p>
<p>曾国藩说哪支部队需要诱敌牺牲，也就牺牲了，相当于包丢了就丢了。两军狭路相逢的时候，曾国藩说上，没有带宽也要上，这才给了曾国藩运筹帷幄，城会玩的机会。同理如果你实现的应用需要有自己的连接策略，可靠保证，时延要求，使用UDP，然后再应用层实现这些是再好不过了。</p>
<h2>基于UDP的“城会玩”的五个例子</h2>
<p>我列举几种“城会玩”的例子。</p>
<h3>“城会玩”一：网页或者APP的访问</h3>
<p>原来访问网页和手机APP都是基于HTTP协议的。HTTP协议是基于TCP的，建立连接都需要多次交互，对于时延比较大的目前主流的移动互联网来讲，建立一次连接需要的时间会比较长，然而既然是移动中，TCP可能还会断了重连，也是很耗时的。而且目前的HTTP协议，往往采取多个数据通道共享一个连接的情况，这样本来为了加快传输速度，但是TCP的严格顺序策略使得哪怕共享通道，前一个不来，后一个和前一个即便没关系，也要等着，时延也会加大。</p>
<p>而<strong>QUIC</strong>（全称<strong>Quick UDP Internet Connections</strong>，<strong>快速UDP互联网连接</strong>）是Google提出的一种基于UDP改进的通信协议，其目的是降低网络通信的延迟，提供更好的用户互动体验。</p>
<p>QUIC在应用层上，会自己实现快速连接建立、减少重传时延，自适应拥塞控制，是应用层“城会玩”的代表。这一节主要是讲UDP，QUIC我们放到应用层去讲。</p>
<h3>“城会玩”二：流媒体的协议</h3>
<p>现在直播比较火，直播协议多使用RTMP，这个协议我们后面的章节也会讲，而这个RTMP协议也是基于TCP的。TCP的严格顺序传输要保证前一个收到了，下一个才能确认，如果前一个收不到，下一个就算包已经收到了，在缓存里面，也需要等着。对于直播来讲，这显然是不合适的，因为老的视频帧丢了其实也就丢了，就算再传过来用户也不在意了，他们要看新的了，如果老是没来就等着，卡顿了，新的也看不了，那就会丢失客户，所以直播，实时性比较比较重要，宁可丢包，也不要卡顿的。</p>
<p>另外，对于丢包，其实对于视频播放来讲，有的包可以丢，有的包不能丢，因为视频的连续帧里面，有的帧重要，有的不重要，如果必须要丢包，隔几个帧丢一个，其实看视频的人不会感知，但是如果连续丢帧，就会感知了，因而在网络不好的情况下，应用希望选择性的丢帧。</p>
<p>还有就是当网络不好的时候，TCP协议会主动降低发送速度，这对本来当时就卡的看视频来讲是要命的，应该应用层马上重传，而不是主动让步。因而，很多直播应用，都基于UDP实现了自己的视频传输协议。</p>
<h3>“城会玩”三：实时游戏</h3>
<p>游戏有一个特点，就是实时性比较高。快一秒你干掉别人，慢一秒你被别人爆头，所以很多职业玩家会买非常专业的鼠标和键盘，争分夺秒。</p>
<p>因而，实时游戏中客户端和服务端要建立长连接，来保证实时传输。但是游戏玩家很多，服务器却不多。由于维护TCP连接需要在内核维护一些数据结构，因而一台机器能够支撑的TCP连接数目是有限的，然后UDP由于是没有连接的，在异步IO机制引入之前，常常是应对海量客户端连接的策略。</p>
<p>另外还是TCP的强顺序问题，对战的游戏，对网络的要求很简单，玩家通过客户端发送给服务器鼠标和键盘行走的位置，服务器会处理每个用户发送过来的所有场景，处理完再返回给客户端，客户端解析响应，渲染最新的场景展示给玩家。</p>
<p>如果出现一个数据包丢失，所有事情都需要停下来等待这个数据包重发。客户端会出现等待接收数据，然而玩家并不关心过期的数据，激战中卡1秒，等能动了都已经死了。</p>
<p>游戏对实时要求较为严格的情况下，采用自定义的可靠UDP协议，自定义重传策略，能够把丢包产生的延迟降到最低，尽量减少网络问题对游戏性造成的影响。</p>
<h3>“城会玩”四：IoT物联网</h3>
<p>一方面，物联网领域终端资源少，很可能只是个内存非常小的嵌入式系统，而维护TCP协议代价太大；另一方面，物联网对实时性要求也很高，而TCP还是因为上面的那些原因导致时延大。Google旗下的Nest建立Thread Group，推出了物联网通信协议Thread，就是基于UDP协议的。</p>
<h3>“城会玩”五：移动通信领域</h3>
<p>在4G网络里，移动流量上网的数据面对的协议GTP-U是基于UDP的。因为移动网络协议比较复杂，而GTP协议本身就包含复杂的手机上线下线的通信协议。如果基于TCP，TCP的机制就显得非常多余，这部分协议我会在后面的章节单独讲解。</p>
<h2>小结</h2>
<p>好了，这节就到这里了，我们来总结一下：</p>
<ul>
<li>
<p>如果将TCP比作成熟的社会人，UDP则是头脑简单的小朋友。TCP复杂，UDP简单；TCP维护连接，UDP谁都相信；TCP会坚持知进退；UDP愣头青一个，勇往直前；</p>
</li>
<li>
<p>UDP虽然简单，但它有简单的用法。它可以用在环境简单、需要多播、应用层自己控制传输的地方。例如DHCP、VXLAN、QUIC等。</p>
</li>
</ul>
<p>最后，给你留两个思考题吧。</p>
<ol>
<li>
<p>都说TCP是面向连接的，在计算机看来，怎么样才算一个连接呢？</p>
</li>
<li>
<p>你知道TCP的连接是如何建立，又是如何关闭的吗？</p>
</li>
</ol>
<p>欢迎你留言和讨论。趣谈网络协议，我们下期见！</p>

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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/b0/c8/342a063f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>iiwoai</span>
  </div>
  <div class="_2_QraFYR_0">每次最后的问题，应该在第二次讲课的时候给答案说出来啊</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-23 21:00:13</div>
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
  <div class="_2_QraFYR_0">网络_10<br># 作业<br>- 连接：在自己监听的端口接收到连接的请求，然后经过“三次握手”，维护一定的数据结构和对方的信息，确认了该信息：我发的内容对方会接收，对方发的内容我也会接收，直到连接断开。<br>- 断开：经过“四次挥手”确保双方都知道且同意对方断开连接，然后在remove为对方维护的数据结构和信息，对方之后发送的包也不会接收，直到 再次连接。<br><br>我看到有的同学说，TCP是建立了一座桥，我认为这个比喻不恰当，TCP更好的比喻是在码头上增加了记录人员，核查人员和督导人员，至于IP层和数据链路层，它没有任何改造。<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个比喻太好了，对的TCP不是桥，是在码头上增加了记录人员，核查人员和督导人员</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-07 14:59:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/8e/c4/8d1150f3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Richie</span>
  </div>
  <div class="_2_QraFYR_0">TCP&#47;UDP建立连接的本质就是在客户端和服务端各自维护一定的数据结构（一种状态机），来记录和维护这个“连接”的状态 。并不是真的会在这两个端之间有一条类似“网络专线”这么一个东西（在学网络协议之前脑海里是这么想象的）。<br><br>在IP层，网络情况该不稳定还是不稳定，数据传输走的是什么路径上层是控制不了的，TCP能做的只能是做更多判断，更多重试，更多拥塞控制之类的东西。<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 理解的太对了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-30 21:09:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/40/88/b36caddb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>仁者</span>
  </div>
  <div class="_2_QraFYR_0">我们可以把发送端和接收端比作河流的两端，把传输的数据包比作运送的石料。TCP 是先搭桥（即建立连接）再一车一车地运（即面向数据流），确保可以顺利到达河对岸，当遇到桥上运输车辆较多时可以自行控制快慢（即拥堵控制）；UDP 则是靠手一个一个地扔（即无连接、基于数据报），不管货物能否顺利到达河对岸，也不关心扔的快慢频率。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-21 10:54:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/61/bc/88a905a5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>亮点</span>
  </div>
  <div class="_2_QraFYR_0">第一个问题:TCP连接是通过三次握手建立连接，四次挥手释放连接，这里的连接是指彼此可以感知到对方的存在，计算机两端表现为socket,有对应的接受缓存和发送缓存，有相应的拥塞控制策略</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-08 09:30:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/2e/ac/7f056518.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Mr.Peng</span>
  </div>
  <div class="_2_QraFYR_0">对包分析的工具及书籍分享一些！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-10 11:03:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/a4/01/22729368.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Havid</span>
  </div>
  <div class="_2_QraFYR_0">包最终都是通过链路层、物量层这样子一个一个出去的，所以连接只是一种逻辑术语，并不存在像管道那样子的东西，连接在这里相当于双方一种约定，双方按说好的规矩维护状态。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-14 16:10:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/80/a0/26a8d76b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Chok Wah</span>
  </div>
  <div class="_2_QraFYR_0">可以详细讲讲UDP常见小技巧，面试常问。<br>比如UDP发过去怎么确认收到；<br>基于UDP封装的协议一般会做哪些措施。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-12 22:15:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/7d/03/4e71c307.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>蓝色理想</span>
  </div>
  <div class="_2_QraFYR_0">似懂非懂，看来还是没懂╯□╰…如果有代码演示看看效果就很好了👍</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-08 10:57:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/6f/a7/0bc9616f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>耿老的竹林</span>
  </div>
  <div class="_2_QraFYR_0"><br>UDP协议看起来简单，恰恰提供给了巨大的扩展空间。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-22 20:19:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/48/bf/a9c9b1a1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Yang</span>
  </div>
  <div class="_2_QraFYR_0">连接:3次握手之后，咱俩之间确认自己发的东西对方都能收到，然后咱俩各自维护这种状态！连接就建立了……</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-08 13:01:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/4d/dc/87809ad2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>埃罗芒阿老师</span>
  </div>
  <div class="_2_QraFYR_0">1.两端各自记录对方的IP端口序列号连接状态等，并且维护连接状态<br>2.三次握手四次挥手</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-09 00:23:28</div>
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
  <div class="_2_QraFYR_0">以前觉得UDT应用不广，听了课之后觉得UDT的应用很多，但实际怎么查看应用用的是TCP还是UDP<br>呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-07-03 08:36:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/50/1c/26dc1927.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>次郎</span>
  </div>
  <div class="_2_QraFYR_0">能不能讲解一些打洞之类的知识？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-07-02 15:31:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/62/0b/13ee0753.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小q</span>
  </div>
  <div class="_2_QraFYR_0">上面说可以推荐本分接析包的书籍，是什么书呀</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-15 14:20:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/88/cd/2c3808ce.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Yangjing</span>
  </div>
  <div class="_2_QraFYR_0">后面可以讲一下实际的分析不？比如用工具 wireshark 对包进行分析讲解，自己能看懂一部分简单的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 因为是音频课程，所以不太适合对包进行分析讲解，但是可以推荐本书，有很多书已经非常好了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-08 13:28:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/11/3e/925aa996.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>HelloBug</span>
  </div>
  <div class="_2_QraFYR_0">老师好，如何理解HTTP协议的多数据通道共享一个连接？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 一个tcp连接</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-26 09:18:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/43/6a/3ea0c553.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Eric</span>
  </div>
  <div class="_2_QraFYR_0">深入浅出，少见的课程，很好！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-08 07:56:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/22/fd/56c4fb54.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Skye</span>
  </div>
  <div class="_2_QraFYR_0">请问老师，TCP自面向字节流，和UDP的数据包具体形式是怎么样的。面向字节流之后下层还怎么封装呀</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-25 19:12:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/66/34/0508d9e4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>u</span>
  </div>
  <div class="_2_QraFYR_0">听一遍不够，要多听几遍</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-09 23:18:20</div>
  </div>
</div>
</div>
</li>
</ul>