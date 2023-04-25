<audio title="第5讲 _ 从物理层到MAC层：如何在宿舍里自己组网玩联机游戏？" src="https://static001.geekbang.org/resource/audio/c0/4e/c01745571018a2a0dd62c5fa269d644e.mp3" controls="controls"></audio> 
<p>上一节，我们见证了IP地址的诞生，或者说是整个操作系统的诞生。一旦机器有了IP，就可以在网络的环境里和其他的机器展开沟通了。</p><p>故事就从我的大学宿舍开始讲起吧。作为一个八零后，我要暴露年龄了。</p><p>我们宿舍四个人，大一的时候学校不让上网，不给开通网络。但是，宿舍有一个人比较有钱，率先买了一台电脑。那买了电脑干什么呢？</p><p>首先，有单机游戏可以打，比如说《拳皇》。两个人用一个键盘，照样打得火热。后来有第二个人买了电脑，那两台电脑能不能连接起来呢？你会说，当然能啊，买个路由器不就行了。</p><p>现在一台家用路由器非常便宜，一百多块的事情。那时候路由器绝对是奢侈品。一直到大四，我们宿舍都没有买路由器。可能是因为那时候技术没有现在这么发达，导致我对网络技术的认知是逐渐深入的，而且每一层都是实实在在接触到的。</p><h2>第一层（物理层）</h2><p>使用路由器，是在第三层上。我们先从第一层物理层开始说。</p><p>物理层能折腾啥？现在的同学可能想不到，我们当时去学校配电脑的地方买网线，卖网线的师傅都会问，你的网线是要电脑连电脑啊，还是电脑连网口啊？</p><p>我们要的是电脑连电脑。这种方式就是一根网线，有两个头。一头插在一台电脑的网卡上，另一头插在另一台电脑的网卡上。但是在当时，普通的网线这样是通不了的，所以水晶头要做交叉线，用的就是所谓的<strong>1－3</strong>、<strong>2－6交叉接法</strong>。</p><!-- [[[read_end]]] --><p>水晶头的第1、2和第3、6脚，它们分别起着收、发信号的作用。将一端的1号和3号线、2号和6号线互换一下位置，就能够在物理层实现一端发送的信号，另一端能收到。</p><p>当然电脑连电脑，除了网线要交叉，还需要配置这两台电脑的IP地址、子网掩码和默认网关。这三个概念上一节详细描述过了。要想两台电脑能够通信，这三项必须配置成为一个网络，可以一个是192.168.0.1/24，另一个是192.168.0.2/24，否则是不通的。</p><p>这里我想问你一个问题，两台电脑之间的网络包，包含MAC层吗？当然包含，要完整。IP层要封装了MAC层才能将包放入物理层。</p><p>到此为止，两台电脑已经构成了一个最小的<strong>局域网</strong>，也即<strong>LAN。</strong>可以玩联机局域网游戏啦！</p><p>等到第三个哥们也买了一台电脑，怎么把三台电脑连在一起呢？</p><p>先别说交换机，当时交换机也贵。有一个叫做<strong>Hub</strong>的东西，也就是<strong>集线器</strong>。这种设备有多个口，可以将宿舍里的多台电脑连接起来。但是，和交换机不同，集线器没有大脑，它完全在物理层工作。它会将自己收到的每一个字节，都复制到其他端口上去。这是第一层物理层联通的方案。</p><h2>第二层（数据链路层）</h2><p>你可能已经发现问题了。Hub采取的是广播的模式，如果每一台电脑发出的包，宿舍的每个电脑都能收到，那就麻烦了。这就需要解决几个问题：</p><ol>
<li>这个包是发给谁的？谁应该接收？</li>
<li>大家都在发，会不会产生混乱？有没有谁先发、谁后发的规则？</li>
<li>如果发送的时候出现了错误，怎么办？</li>
</ol><p>这几个问题，都是第二层，数据链路层，也即MAC层要解决的问题。<strong>MAC</strong>的全称是<strong>Medium Access Control</strong>，即<strong>媒体访问控制。<strong>控制什么呢？其实就是控制在往媒体上发数据的时候，谁先发、谁后发的问题。防止发生混乱。这解决的是第二个问题。这个问题中的规则，学名叫</strong>多路访问</strong>。有很多算法可以解决这个问题。就像车管所管束马路上跑的车，能想的办法都想过了。</p><p>比如接下来这三种方式：</p><ul>
<li>
<p>方式一：分多个车道。每个车一个车道，你走你的，我走我的。这在计算机网络里叫作<strong>信道划分；</strong></p>
</li>
<li>
<p>方式二：今天单号出行，明天双号出行，轮着来。这在计算机网络里叫作<strong>轮流协议；</strong></p>
</li>
<li>
<p>方式三：不管三七二十一，有事儿先出门，发现特堵，就回去。错过高峰再出。我们叫作<strong>随机接入协议。</strong>著名的以太网，用的就是这个方式。</p>
</li>
</ul><p>解决了第二个问题，就是解决了媒体接入控制的问题，MAC的问题也就解决好了。这和MAC地址没什么关系。</p><p>接下来要解决第一个问题：发给谁，谁接收？这里用到一个物理地址，叫作<strong>链路层地址。<strong>但是因为第二层主要解决媒体接入控制的问题，所以它常被称为</strong>MAC地址</strong>。</p><p>解决第一个问题就牵扯到第二层的网络包<strong>格式</strong>。对于以太网，第二层的最开始，就是目标的MAC地址和源的MAC地址。</p><p><img src="https://static001.geekbang.org/resource/image/80/41/8072e4885b0cbc6cb5384ea84d487e41.jpg?wh=2443*601" alt=""></p><p>接下来是<strong>类型</strong>，大部分的类型是IP数据包，然后IP里面包含TCP、UDP，以及HTTP等，这都是里层封装的事情。</p><p>有了这个目标MAC地址，数据包在链路上广播，MAC的网卡才能发现，这个包是给它的。MAC的网卡把包收进来，然后打开IP包，发现IP地址也是自己的，再打开TCP包，发现端口是自己，也就是80，而nginx就是监听80。</p><p>于是将请求提交给nginx，nginx返回一个网页。然后将网页需要发回请求的机器。然后层层封装，最后到MAC层。因为来的时候有源MAC地址，返回的时候，源MAC就变成了目标MAC，再返给请求的机器。</p><p>对于以太网，第二层的最后面是<strong>CRC</strong>，也就是<strong>循环冗余检测</strong>。通过XOR异或的算法，来计算整个包是否在发送的过程中出现了错误，主要解决第三个问题。</p><p>这里还有一个没有解决的问题，当源机器知道目标机器的时候，可以将目标地址放入包里面，如果不知道呢？一个广播的网络里面接入了N台机器，我怎么知道每个MAC地址是谁呢？这就是<strong>ARP协议</strong>，也就是已知IP地址，求MAC地址的协议。</p><p><img src="https://static001.geekbang.org/resource/image/56/37/561324a275460a4abbc15e73a476e037.jpg?wh=2323*1951" alt=""></p><p>在一个局域网里面，当知道了IP地址，不知道MAC怎么办呢？靠“吼”。</p><p><img src="https://static001.geekbang.org/resource/image/48/ad/485b5902066131de547acbcf3579c4ad.jpg?wh=2623*2203" alt=""></p><p>广而告之，发送一个广播包，谁是这个IP谁来回答。具体询问和回答的报文就像下面这样：</p><p><img src="https://static001.geekbang.org/resource/image/1f/9b/1f7cfe6046c5df606cfbb6bb6c7f899b.jpg?wh=2078*694" alt=""></p><p>为了避免每次都用ARP请求，机器本地也会进行ARP缓存。当然机器会不断地上线下线，IP也可能会变，所以ARP的MAC地址缓存过一段时间就会过期。</p><h2>局域网</h2><p>好了，至此我们宿舍四个电脑就组成了一个局域网。用Hub连接起来，就可以玩局域网版的《魔兽争霸》了。</p><p><img src="https://static001.geekbang.org/resource/image/33/ac/33d180e376439ca10e3f126eb2e36bac.jpg?wh=390*590" alt=""></p><p>打开游戏，进入“局域网选项”，选择一张地图，点击“创建游戏”，就可以进入这张地图的房间中。等同一个局域网里的其他小伙伴加入后，游戏就可以开始了。</p><p>这种组网的方法，对一个宿舍来说没有问题，但是一旦机器数目增多，问题就出现了。因为Hub是广播的，不管某个接口是否需要，所有的Bit都会被发送出去，然后让主机来判断是不是需要。这种方式路上的车少就没问题，车一多，产生冲突的概率就提高了。而且把不需要的包转发过去，纯属浪费。看来Hub这种不管三七二十一都转发的设备是不行了，需要点儿智能的。因为每个口都只连接一台电脑，这台电脑又不怎么换IP和MAC地址，只要记住这台电脑的MAC地址，如果目标MAC地址不是这台电脑的，这个口就不用转发了。</p><p>谁能知道目标MAC地址是否就是连接某个口的电脑的MAC地址呢？这就需要一个能把MAC头拿下来，检查一下目标MAC地址，然后根据策略转发的设备，按第二节课中讲过的，这个设备显然是个二层设备，我们称为<strong>交换机</strong>。</p><p>交换机怎么知道每个口的电脑的MAC地址呢？这需要交换机会学习。</p><p>一台MAC1电脑将一个包发送给另一台MAC2电脑，当这个包到达交换机的时候，一开始交换机也不知道MAC2的电脑在哪个口，所以没办法，它只能将包转发给除了来的那个口之外的其他所有的口。但是，这个时候，交换机会干一件非常聪明的事情，就是交换机会记住，MAC1是来自一个明确的口。以后有包的目的地址是MAC1的，直接发送到这个口就可以了。</p><p>当交换机作为一个关卡一样，过了一段时间之后，就有了整个网络的一个结构了，这个时候，基本上不用广播了，全部可以准确转发。当然，每个机器的IP地址会变，所在的口也会变，因而交换机上的学习的结果，我们称为<strong>转发表</strong>，是有一个过期时间的。</p><p>有了交换机，一般来说，你接个几十台、上百台机器打游戏，应该没啥问题。你可以组个战队了。能上网了，就可以玩网游了。</p><h2>小结</h2><p>好了，今天的内容差不多了，我们来总结一下，有三个重点需要你记住：</p><p>第一，MAC层是用来解决多路访问的堵车问题的；</p><p>第二，ARP是通过吼的方式来寻找目标MAC地址的，吼完之后记住一段时间，这个叫作缓存；</p><p>第三，交换机是有MAC地址学习能力的，学完了它就知道谁在哪儿了，不用广播了。</p><p>最后，给你留两个思考题吧。</p><ol>
<li>在二层中我们讲了ARP协议，即已知IP地址求MAC；还有一种RARP协议，即已知MAC求IP的，你知道它可以用来干什么吗？</li>
<li>如果一个局域网里面有多个交换机，ARP广播的模式会出现什么问题呢？</li>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/d6/42/bafec9b7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>盖</span>
  </div>
  <div class="_2_QraFYR_0">ARP广播时，交换机会将一个端口收到的包转发到其它所有的端口上。<br>比如数据包经过交换机A到达交换机B，交换机B又将包复制为多份广播出去。<br>如果整个局域网存在一个环路，使得数据包又重新回到了最开始的交换机A，这个包又会被A再次复制多份广播出去。<br>如此循环，数据包会不停得转发，而且越来越多，最终占满带宽，或者使解析协议的硬件过载，行成广播风暴。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-05-29 17:46:15</div>
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
  <div class="_2_QraFYR_0">之前有无盘工作站，即没有硬盘的机器，无法持久化ip地址到本地，但有网卡，所以可以用RARP协议来获取IP地址。RARP可以用于局域网管理员想指定机器IP（与机器绑定，不可变），又不想每台机器去设置静态IP的情况，可以在RARP服务器上配置MAC和IP对应的ARP表，不过获取每台机器的MAC地址，好像也挺麻烦的。这个协议现在应该用得不多了吧，都用BOOTP或者DHCP了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对的，赞</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-05-29 11:11:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/6a/06/66831563.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>阡陌</span>
  </div>
  <div class="_2_QraFYR_0">不得不说，看留言也能学到很多东西</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 高手还是很多的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-05-30 08:32:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/fe/8f/466f880d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>没心没肺</span>
  </div>
  <div class="_2_QraFYR_0">Hub：<br>1.一个广播域，一个冲突域。<br>2.传输数据的过程中易产生冲突，带宽利用率不高<br>Switch：<br>1.在划分vlan的前提下可以实现多个广播域，每个接口都是一个单独的冲突域<br>2.通过自我学习的方法可以构建出CAM表，并基于CAM进行转发数据。<br>3.支持生成树算法。可以构建出物理有环，逻辑无环的网络，网络冗余和数据传输效率都甩Hub好几条街。SW是目前组网的基本设备之一。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-05-28 09:56:44</div>
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
  <div class="_2_QraFYR_0">一直坚持看到第五讲，我理解能力太差了，感觉还是一头雾水唉……</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-05-29 20:18:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/69/df/71563d52.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>戴劼 DAI JIE🤪</span>
  </div>
  <div class="_2_QraFYR_0">当年上课学习记住了交叉线和直连线的区别，工作后有一次两台机器对拷，发现网卡能自适应直连线，懵逼了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，现在自适应了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-07 01:37:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/46/00/51f09059.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>hujunr</span>
  </div>
  <div class="_2_QraFYR_0">当时用的交换机，把一条网线的2端同时接到交换机了，结果所有电脑都连不上网了，这是为什么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: arp广播塞满了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-07-30 14:46:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/d2/f7/fabad335.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>天边的一只鱼</span>
  </div>
  <div class="_2_QraFYR_0">看了前几章，个人理解下访问外网ip的流程，不知道对不对，  我现在在公司的内网想要访问一个北京的外网ip， 首先把我自己的ip地址，mac地址，端口，外网的ip地址，端口，在内网吼一下，被公司网关收到，判断下这个ip是不是内网的， 不是的话，添加上公司自己的mac地址，然后往更上一层吼一下(某个区域电信的网关)，然后这个区域的电信网管判断下ip是不是我这一片的,再试再加上自己的mac地址，再层层往上吼，一直找到这个ip为止。   不知道这么裂解对不对，刘老师。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-12 10:53:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/6a/03/cb597311.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>远心</span>
  </div>
  <div class="_2_QraFYR_0">用网线直接连接两台计算机的方式，如何知道另一台计算机的 MAC  地址？使用 ARP 协议吗？也就是说其实每一台计算机都安装着 ARP Client&#47;Server 吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是arp，内核里面就有这部分逻辑</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-15 18:46:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/42/45/cdfe6842.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Z3</span>
  </div>
  <div class="_2_QraFYR_0">当年玩魔兽经常出现他建房我看不见，我建房他能看见之类的问题。 这些可能是应用层的问题吗？<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个，场景不在了，很难分析</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-05-30 15:12:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/e9/29/629d9bb0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>天王</span>
  </div>
  <div class="_2_QraFYR_0">网络分层第一层物理层，第二层数据链路层。物理层提供设备，即数据传输的通路，将多台设备连接起来。设备如网线，接线头，hub等。数据链路层，即mac层，medium access controller，媒体访问控制，需要解决几个问题1谁先发谁后发，2发给谁，谁接收，3 发送出现了错误，怎么办。1 .1信道划分，各走各的道 1.2 轮流 1.3 ， 随机接入协议，错开高峰，不行就等。2.1 mac地址即数据链路层地址，第二层数据包格式，依次是mac地址，目标mac，源mac，数据类型比如ip数据包，数据包在链路层上广播，目标mac的机器会找到，去掉mac包，看下ip是自己的，则认为是发给自己的，再打开tcp包，发现端口是80，会交给nginx，让nginx处理。第二层最后面是CRC，循环冗余检测，来计算发送过程中是否发生错误。现在有ip，没mac，需要将mac放入数据包。需要ARP协议，通过ip找mac，1 查找本地ARP表，有的直接返回，2 广播ARP请求，靠吼，3等着应答。hub是广播的，什么都会发出去，让主机判断，需不需要。交换机比较智能，他有记忆功能，一开始不知道哪个mac对应哪台电脑，一开始会都广播，后来有应答之后，会记住哪台电脑，以后就直接发到那台电脑上。<br>- [ ] <br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-05 08:55:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/74/ea/10661bdc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>kevinsu</span>
  </div>
  <div class="_2_QraFYR_0">是不是可以理解成交换机是具备学习功能的hub</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-25 16:43:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/85/0a/e564e572.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>N_H</span>
  </div>
  <div class="_2_QraFYR_0">老师，根据你前面的几个讲解，我理解到，机器A知道机器B的ip无法准确进行通信，因为ip可以在局域网内进行分配。但是知道mac地址肯定是能进行通信的，因为mac地址是唯一的，无论这个机器到哪里去了，我都能通过mac地址找到这个机器，既然这样，那为什么还需要ip地址？<br>这是我看了几期课程以来一直的疑问。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: mac是局域网的定位，ip是跨网络的定位</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-28 11:12:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/fc/75/af67e5cc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>黑猫紧张</span>
  </div>
  <div class="_2_QraFYR_0">ARP协议是在局域网中的，那么在公网中二层如何获得目标mac地址？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 公网也是一个个的局域网组成的呀</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-01 12:21:08</div>
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
  <div class="_2_QraFYR_0">文章开头说到：一旦机器有了 IP，就可以在网络的环境里和其他的机器展开沟通了。<br><br>我有个疑问：机器没有ip，就不能和其它机器通信了吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不能呀，至少不能通过TCP&#47;IP协议栈进行通信，其他通信方式不在这门课讨论之列</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-15 08:29:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/8d/a6/22c37c91.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>楊_宵夜</span>
  </div>
  <div class="_2_QraFYR_0">连续看了好几篇！真的是大道至简！逻辑无比清晰！细节的深度刚刚好！赞！以太网也是个很有趣的东西，记得在知乎上看到过，最离奇的Bug都有哪些。其中一个关于以太网的是说，某个大学因为建筑施工，挖掘机的声波频率影响了光纤电缆中的信号传输，导致信号最多只能传输520多公里哈哈哈，也就是说520公里以外的服务器都访问不到了。<br>这里细节肯定记错了，抛个砖，有兴趣的各位可以在知乎搜搜😂😂</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢啦</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-08 13:13:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/3c/fb/ac114c4c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>高磊</span>
  </div>
  <div class="_2_QraFYR_0">局域网玩的是魔兽争霸，不是魔兽世界</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 哈哈</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-11 10:48:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/df/d7/ee3da208.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Yuki</span>
  </div>
  <div class="_2_QraFYR_0">非科班出身，理解很困难。但每天坚持看一篇，多看看一定能慢慢理解～</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以几天看一篇的，慢慢消化，不图快</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-13 09:12:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/42/52/19553613.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>刘培培</span>
  </div>
  <div class="_2_QraFYR_0">ARAP 协议：https:&#47;&#47;tools.ietf.org&#47;html&#47;rfc903 因为太难实现，被后来的 BOOTP 协议：https:&#47;&#47;tools.ietf.org&#47;html&#47;rfc951 和 DHCP 协议：https:&#47;&#47;tools.ietf.org&#47;html&#47;rfc2131  替代。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 牛</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-26 09:13:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/ed/4d/1d1a1a00.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>magict4</span>
  </div>
  <div class="_2_QraFYR_0">两台电脑直连的情况下，谁是网关呢？电脑可以充当网关吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不用配置网关就能通</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-18 07:41:34</div>
  </div>
</div>
</div>
</li>
</ul>