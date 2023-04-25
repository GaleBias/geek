<audio title="第22讲 _ VPN：朝中有人好做官" src="https://static001.geekbang.org/resource/audio/0b/c8/0b3436bfb8557d895cf50828f73fb5c8.mp3" controls="controls"></audio> 
<p>前面我们讲到了数据中心，里面很复杂，但是有的公司有多个数据中心，需要将多个数据中心连接起来，或者需要办公室和数据中心连接起来。这该怎么办呢？</p><ul>
<li>
<p>第一种方式是走公网，但是公网太不安全，你的隐私可能会被别人偷窥。</p>
</li>
<li>
<p>第二种方式是租用专线的方式把它们连起来，这是土豪的做法，需要花很多钱。</p>
</li>
<li>
<p>第三种方式是用VPN来连接，这种方法比较折中，安全又不贵。</p>
</li>
</ul><p><img src="https://static001.geekbang.org/resource/image/9f/68/9f797934cb5cf40543b716d97e214868.jpg?wh=707*500" alt=""></p><p><strong>VPN</strong>，全名<strong>Virtual Private Network</strong>，<strong>虚拟专用网</strong>，就是利用开放的公众网络，建立专用数据传输通道，将远程的分支机构、移动办公人员等连接起来。</p><h2>VPN是如何工作的？</h2><p>VPN通过隧道技术在公众网络上仿真一条点到点的专线，是通过利用一种协议来传输另外一种协议的技术，这里面涉及三种协议：<strong>乘客协议</strong>、<strong>隧道协议</strong>和<strong>承载协议</strong>。</p><p>我们以IPsec协议为例来说明。</p><p><img src="https://static001.geekbang.org/resource/image/52/7f/52c72a2a29c6b8526a243125e101f77f.jpeg?wh=1498*413" alt=""></p><p>你知道如何通过自驾进行海南游吗？这其中，你的车怎么通过琼州海峡呢？这里用到轮渡，其实这就用到<strong>隧道协议</strong>。</p><p>在广州这边开车是有“协议”的，例如靠右行驶、红灯停、绿灯行，这个就相当于“被封装”的<strong>乘客协议</strong>。当然在海南那面，开车也是同样的协议。这就相当于需要连接在一起的一个公司的两个分部。</p><p>但是在海上坐船航行，也有它的协议，例如要看灯塔、要按航道航行等。这就是外层的<strong>承载协议</strong>。</p><!-- [[[read_end]]] --><p>那我的车如何从广州到海南呢？这就需要你遵循开车的协议，将车开上轮渡，所有通过轮渡的车都关在船舱里面，按照既定的规则排列好，这就是<strong>隧道协议</strong>。</p><p>在大海上，你的车是关在船舱里面的，就像在隧道里面一样，这个时候内部的乘客协议，也即驾驶协议没啥用处，只需要船遵从外层的承载协议，到达海南就可以了。</p><p>到达之后，外部承载协议的任务就结束了，打开船舱，将车开出来，就相当于取下承载协议和隧道协议的头。接下来，在海南该怎么开车，就怎么开车，还是内部的乘客协议起作用。</p><p>在最前面的时候说了，直接使用公网太不安全，所以接下来我们来看一种十分安全的VPN，<strong>IPsec VPN</strong>。这是基于IP协议的<strong>安全隧道协议</strong>，为了保证在公网上面信息的安全，因而采取了一定的机制保证安全性。</p><ul>
<li>
<p>机制一：<strong>私密性</strong>，防止信息泄露给未经授权的个人，通过加密把数据从明文变成无法读懂的密文，从而确保数据的私密性。<br>
前面讲HTTPS的时候，说过加密可以分为对称加密和非对称加密。对称加密速度快一些。而VPN一旦建立，需要传输大量数据，因而我们采取对称加密。但是同样，对称加密还是存在加密密钥如何传输的问题，这里需要用到因特网密钥交换（IKE，Internet Key Exchange）协议。</p>
</li>
<li>
<p>机制二：<strong>完整性</strong>，数据没有被非法篡改，通过对数据进行hash运算，产生类似于指纹的数据摘要，以保证数据的完整性。</p>
</li>
<li>
<p>机制三：<strong>真实性</strong>，数据确实是由特定的对端发出，通过身份认证可以保证数据的真实性。</p>
</li>
</ul><p>那如何保证对方就是真正的那个人呢？</p><ul>
<li>
<p>第一种方法就是<strong>预共享密钥</strong>，也就是双方事先商量好一个暗号，比如“天王盖地虎，宝塔镇河妖”，对上了，就说明是对的。</p>
</li>
<li>
<p>另外一种方法就是<strong>用数字签名来验证</strong>。咋签名呢？当然是使用私钥进行签名，私钥只有我自己有，所以如果对方能用我的数字证书里面的公钥解开，就说明我是我。</p>
</li>
</ul><p>基于以上三个特性，组成了<strong>IPsec VPN的协议簇</strong>。这个协议簇内容比较丰富。</p><p><img src="https://static001.geekbang.org/resource/image/e4/c2/e43f13e5c68c9a5455b3793fb530a4c2.jpeg?wh=1920*742" alt=""></p><p>在这个协议簇里面，有两种协议，这两种协议的区别在于封装网络包的格式不一样。</p><ul>
<li>
<p>一种协议称为<strong>AH</strong>（<strong>Authentication Header</strong>），只能进行数据摘要 ，不能实现数据加密。</p>
</li>
<li>
<p>还有一种<strong>ESP</strong>（<strong>Encapsulating Security Payload</strong>），能够进行数据加密和数据摘要。</p>
</li>
</ul><p>在这个协议簇里面，还有两类算法，分别是<strong>加密算法</strong>和<strong>摘要算法</strong>。</p><p>这个协议簇还包含两大组件，一个用于VPN的双方要进行对称密钥的交换的<strong>IKE组件</strong>，另一个是VPN的双方要对连接进行维护的<strong>SA（Security Association）组件</strong>。</p><h2>IPsec VPN的建立过程</h2><p>下面来看IPsec VPN的建立过程，这个过程分两个阶段。</p><p><strong>第一个阶段，建立IKE自己的SA</strong>。这个SA用来维护一个通过身份认证和安全保护的通道，为第二个阶段提供服务。在这个阶段，通过DH（Diffie-Hellman）算法计算出一个对称密钥K。</p><p>DH算法是一个比较巧妙的算法。客户端和服务端约定两个公开的质数p和q，然后客户端随机产生一个数a作为自己的私钥，服务端随机产生一个b作为自己的私钥，客户端可以根据p、q和a计算出公钥A，服务端根据p、q和b计算出公钥B，然后双方交换公钥A和B。</p><p>到此客户端和服务端可以根据已有的信息，各自独立算出相同的结果K，就是<strong>对称密钥</strong>。但是这个过程，对称密钥从来没有在通道上传输过，只传输了生成密钥的材料，通过这些材料，截获的人是无法算出的。</p><p><img src="https://static001.geekbang.org/resource/image/24/ea/24c89b7703ffcb7ce966f9d259b06eea.jpeg?wh=1920*1080" alt=""></p><p>有了这个对称密钥K，接下来是<strong>第二个阶段，建立IPsec SA</strong>。在这个SA里面，双方会生成一个随机的对称密钥M，由K加密传给对方，然后使用M进行双方接下来通信的数据。对称密钥M是有过期时间的，会过一段时间，重新生成一次，从而防止被破解。</p><p>IPsec SA里面有以下内容：</p><ul>
<li>
<p>SPI（Security Parameter Index），用于标识不同的连接；</p>
</li>
<li>
<p>双方商量好的加密算法、哈希算法和封装模式；</p>
</li>
<li>
<p>生存周期，超过这个周期，就需要重新生成一个IPsec SA，重新生成对称密钥。</p>
</li>
</ul><p><img src="https://static001.geekbang.org/resource/image/4d/75/4dd02eaa9a0242252371248a1b455275.jpeg?wh=1920*835" alt=""></p><p>当IPsec建立好，接下来就可以开始打包封装传输了。</p><p><img src="https://static001.geekbang.org/resource/image/15/65/158af554e14dd93237b136a99d07d165.jpg?wh=4329*6491" alt=""></p><p>左面是原始的IP包，在IP头里面，会指定上一层的协议为TCP。ESP要对IP包进行封装，因而IP头里面的上一层协议为ESP。在ESP的正文里面，ESP的头部有双方商讨好的SPI，以及这次传输的序列号。</p><p>接下来全部是加密的内容。可以通过对称密钥进行解密，解密后在正文的最后，指明了里面的协议是什么。如果是IP，则需要先解析IP头，然后解析TCP头，这是从隧道出来后解封装的过程。</p><p>有了IPsec VPN之后，客户端发送的明文的IP包，都会被加上ESP头和IP头，在公网上传输，由于加密，可以保证不被窃取，到了对端后，去掉ESP的头，进行解密。</p><p><img src="https://static001.geekbang.org/resource/image/48/d2/487d8dde5ec7758b4c169509fba59cd2.jpeg?wh=1920*866" alt=""></p><p>这种点对点的基于IP的VPN，能满足互通的要求，但是速度往往比较慢，这是由底层IP协议的特性决定的。IP不是面向连接的，是尽力而为的协议，每个IP包自由选择路径，到每一个路由器，都自己去找下一跳，丢了就丢了，是靠上一层TCP的重发来保证可靠性。</p><p><img src="https://static001.geekbang.org/resource/image/29/8e/29f4dff1f40a252dda2ac6f9d1d4088e.jpg?wh=747*316" alt=""></p><p>因为IP网络从设计的时候，就认为是不可靠的，所以即使同一个连接，也可能选择不同的道路，这样的好处是，一条道路崩溃的时候，总有其他的路可以走。当然，带来的代价就是，不断的路由查找，效率比较差。</p><p>和IP对应的另一种技术称为ATM。这种协议和IP协议的不同在于，它是面向连接的。你可以说TCP也是面向连接的啊。这两个不同，ATM和IP是一个层次的，和TCP不是一个层次的。</p><p>另外，TCP所谓的面向连接，是不停地重试来保证成功，其实下层的IP还是不面向连接的，丢了就丢了。ATM是传输之前先建立一个连接，形成一个虚拟的通路，一旦连接建立了，所有的包都按照相同的路径走，不会分头行事。</p><p><img src="https://static001.geekbang.org/resource/image/af/48/afea2a2cecb8bfe6c1c40585000f1c48.jpg?wh=756*305" alt=""></p><p>好处是不需要每次都查路由表的，虚拟路径已经建立，打上了标签，后续的包傻傻的跟着走就是了，不用像IP包一样，每个包都思考下一步怎么走，都按相同的路径走，这样效率会高很多。</p><p>但是一旦虚拟路径上的某个路由器坏了，则这个连接就断了，什么也发不过去了，因为其他的包还会按照原来的路径走，都掉坑里了，它们不会选择其他的路径走。</p><p>ATM技术虽然没有成功，但其屏弃了繁琐的路由查找，改为简单快速的标签交换，将具有全局意义的路由表改为只有本地意义的标签表，这些都可以大大提高一台路由器的转发功力。</p><p>有没有一种方式将两者的优点结合起来呢？这就是<strong>多协议标签交换</strong>（<strong>MPLS</strong>，<strong>Multi-Protocol Label Switching</strong>）。MPLS的格式如图所示，在原始的IP头之外，多了MPLS的头，里面可以打标签。</p><p><img src="https://static001.geekbang.org/resource/image/c4/96/c45a0a2f11678f15e126449e64644596.jpeg?wh=1920*560" alt=""></p><p>在二层头里面，有类型字段，0x0800表示IP，0x8847表示MPLS Label。</p><p>在MPLS头里面，首先是标签值占20位，接着是3位实验位，再接下来是1位栈底标志位，表示当前标签是否位于栈底了。这样就允许多个标签被编码到同一个数据包中，形成标签栈。最后是8位TTL存活时间字段，如果标签数据包的出发TTL值为0，那么该数据包在网络中的生命期被认为已经过期了。</p><p>有了标签，还需要设备认这个标签，并且能够根据这个标签转发，这种能够转发标签的路由器称为<strong>标签交换路由器</strong>（LSR，Label Switching Router）。</p><p>这种路由器会有两个表格，一个就是传统的FIB，也即路由表，另一个就是LFIB，标签转发表。有了这两个表，既可以进行普通的路由转发，也可以进行基于标签的转发。</p><p><img src="https://static001.geekbang.org/resource/image/66/97/66df10af27dbyycee673dc47667dbb97.jpeg?wh=1920*767" alt=""></p><p>有了标签转发表，转发的过程如图所示，就不用每次都进行普通路由的查找了。</p><p>这里我们区分MPLS区域和非MPLS区域。在MPLS区域中间，使用标签进行转发，非MPLS区域，使用普通路由转发，在边缘节点上，需要有能力将对于普通路由的转发，变成对于标签的转发。</p><p>例如图中要访问114.1.1.1，在边界上查找普通路由，发现马上要进入MPLS区域了，进去了对应标签1，于是在IP头外面加一个标签1，在区域里面，标签1要变成标签3，标签3到达出口边缘，将标签去掉，按照路由发出。</p><p>这样一个通过标签转换而建立的路径称为LSP，标签交换路径。在一条LSP上，沿数据包传送的方向，相邻的LSR分别叫<strong>上游LSR</strong>（<strong>upstream LSR</strong>）和<strong>下游LSR</strong>（<strong>downstream LSR</strong>）。</p><p>有了标签，转发是很简单的事，但是如何生成标签，却是MPLS中最难修炼的部分。在MPLS秘笈中，这部分被称为<strong>LDP</strong>（<strong>Label Distribution Protocol</strong>），是一个动态的生成标签的协议。</p><p>其实LDP与IP帮派中的路由协议十分相像，通过LSR的交互，互相告知去哪里应该打哪个标签，称为标签分发，往往是从下游开始的。</p><p><img src="https://static001.geekbang.org/resource/image/91/67/91cd0038877da8933365494625280b67.jpeg?wh=1920*870" alt=""></p><p>如果有一个边缘节点发现自己的路由表中出现了新的目的地址，它就要给别人说，我能到达一条新的路径了。</p><p>如果此边缘节点存在上游LSR，并且尚有可供分配的标签，则该节点为新的路径分配标签，并向上游发出标签映射消息，其中包含分配的标签等信息。</p><p>收到标签映射消息的LSR记录相应的标签映射信息，在其标签转发表中增加相应的条目。此LSR为它的上游LSR分配标签，并继续向上游LSR发送标签映射消息。</p><p>当入口LSR收到标签映射消息时，在标签转发表中增加相应的条目。这时，就完成了LSP的建立。有了标签，转发轻松多了，但是这个和VPN什么关系呢？</p><p>可以想象，如果我们VPN通道里面包的转发，都是通过标签的方式进行，效率就会高很多。所以要想个办法把MPLS应用于VPN。</p><p><img src="https://static001.geekbang.org/resource/image/eb/ae/eb01c7d748b99bc94017f667eb74caae.jpeg?wh=1920*724" alt=""></p><p>在MPLS VPN中，网络中的路由器分成以下几类：</p><ul>
<li>
<p>PE（Provider Edge）：运营商网络与客户网络相连的边缘网络设备；</p>
</li>
<li>
<p>CE（Customer Edge）：客户网络与PE相连接的边缘设备；</p>
</li>
<li>
<p>P（Provider）：这里特指运营商网络中除PE之外的其他运营商网络设备。</p>
</li>
</ul><p>为什么要这样分呢？因为我们发现，在运营商网络里面，也即P Router之间，使用标签是没有问题的，因为都在运营商的管控之下，对于网段，路由都可以自己控制。但是一旦客户要接入这个网络，就复杂得多。</p><p>首先是客户地址重复的问题。客户所使用的大多数都是私网的地址(192.168.X.X;10.X.X.X;172.X.X.X)，而且很多情况下都会与其它的客户重复。</p><p>比如，机构A和机构B都使用了192.168.101.0/24网段的地址，这就发生了地址空间重叠（Overlapping Address Spaces）。</p><p>首先困惑的是BGP协议，既然VPN将两个数据中心连起来，应该看起来像一个数据中心一样，那么如何到达另一端需要通过BGP将路由广播过去，传统BGP无法正确处理地址空间重叠的VPN的路由。</p><p>假设机构A和机构B都使用了192.168.101.0/24网段的地址，并各自发布了一条去往此网段的路由，BGP将只会选择其中一条路由，从而导致去往另一个VPN的路由丢失。</p><p>所以PE路由器之间使用特殊的MP-BGP来发布VPN路由，在相互沟通的消息中，在一般32位IPv4的地址之前加上一个客户标示的区分符用于客户地址的区分，这种称为VPN-IPv4地址族，这样PE路由器会收到如下的消息，机构A的192.168.101.0/24应该往这面走，机构B的192.168.101.0/24则应该去另外一个方向。</p><p>另外困惑的是<strong>路由表</strong>，当两个客户的IP包到达PE的时候，PE就困惑了，因为网段是重复的。</p><p>如何区分哪些路由是属于哪些客户VPN内的？如何保证VPN业务路由与普通路由不相互干扰？</p><p>在PE上，可以通过VRF（VPN Routing&amp;Forwarding Instance）建立每个客户一个路由表，与其它VPN客户路由和普通路由相互区分。可以理解为专属于客户的小路由器。</p><p>远端PE通过MP-BGP协议把业务路由放到近端PE，近端PE根据不同的客户选择出相关客户的业务路由放到相应的VRF路由表中。</p><p>VPN报文转发采用两层标签方式：</p><ul>
<li>
<p>第一层（外层）标签在骨干网内部进行交换，指示从PE到对端PE的一条LSP。VPN报文利用这层标签，可以沿LSP到达对端PE；</p>
</li>
<li>
<p>第二层（内层）标签在从对端PE到达CE时使用，在PE上，通过查找VRF表项，指示报文应被送到哪个VPN用户，或者更具体一些，到达哪一个CE。这样，对端PE根据内层标签可以找到转发报文的接口。</p>
</li>
</ul><p><img src="https://static001.geekbang.org/resource/image/43/1f/43e9ee00e3db2b0cd8315058d38d701f.jpeg?wh=1920*839" alt=""></p><p>我们来举一个例子，看MPLS VPN的包发送过程。</p><ol>
<li>
<p>机构A和机构B都发出一个目的地址为192.168.101.0/24的IP报文，分别由各自的CE将报文发送至PE。</p>
</li>
<li>
<p>PE会根据报文到达的接口及目的地址查找VPN实例表项VRF，匹配后将报文转发出去，同时打上内层和外层两个标签。假设通过MP-BGP配置的路由，两个报文在骨干网走相同的路径。</p>
</li>
<li>
<p>MPLS网络利用报文的外层标签，将报文传送到出口PE，报文在到达出口PE 2前一跳时已经被剥离外层标签，仅含内层标签。</p>
</li>
<li>
<p>出口PE根据内层标签和目的地址查找VPN实例表项VRF，确定报文的出接口，将报文转发至各自的CE。</p>
</li>
<li>
<p>CE根据正常的IP转发过程将报文传送到目的地。</p>
</li>
</ol><h2>小结</h2><p>好了，这一节就到这里了，我们来总结一下：</p><ul>
<li>
<p>VPN可以将一个机构的多个数据中心通过隧道的方式连接起来，让机构感觉在一个数据中心里面，就像自驾游通过琼州海峡一样；</p>
</li>
<li>
<p>完全基于软件的IPsec VPN可以保证私密性、完整性、真实性、简单便宜，但是性能稍微差一些；</p>
</li>
<li>
<p>MPLS-VPN综合和IP转发模式和ATM的标签转发模式的优势，性能较好，但是需要从运营商购买。</p>
</li>
</ul><p>接下来，给你留两个思考题：</p><ol>
<li>
<p>当前业务的高可用性和弹性伸缩很重要，所以很多机构都会在自建私有云之外，采购公有云，你知道私有云和公有云应该如何打通吗？</p>
</li>
<li>
<p>前面所有的上网行为，都是基于电脑的，但是移动互联网越来越成为核心，你知道手机上网都需要哪些协议吗？</p>
</li>
</ol><p>我们的专栏更新到第22讲，不知你掌握得如何？每节课后我留的思考题，你都有没有认真思考，并在留言区写下答案呢？我会从<strong>已发布的文章中选出一批认真留言的同学</strong>，赠送<span class="orange">学习奖励礼券</span>和我整理的<span class="orange">独家网络协议知识图谱</span>。</p><p>欢迎你留言和我讨论。趣谈网络协议，我们下期见！</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/61/c3/5f1ec5e5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>徐晨</span>
  </div>
  <div class="_2_QraFYR_0">@Fisher <br>我可以举个例子，比如说我们先约定一种颜色比如黄色（p, q 大质数），这时候我把黄色涂上红色（我的随机数 a）生成新的橙色，你把黄色涂上蓝色（你的随机数 b）生成新的绿色。这时候我们交换橙色和绿色，然后我再在绿色上加上红色生成棕色，同样你拿到橙色后加上蓝色也生成棕色。这就是最终的密钥了。如果有人在中间窃取的话，他只能拿到橙色和绿色，是拿不到最终的密钥棕色的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个例子太好了，我原来一直想不出一个具体的例子</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-24 11:19:40</div>
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
  <div class="_2_QraFYR_0">近期两节内容跳跃好厉害，跟不上了😁 </div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-07-06 20:29:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/be/41/e3ece193.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lovelife</span>
  </div>
  <div class="_2_QraFYR_0">公有云和私有云之间打通主要通过两种方式，一种是专线，这个需要跟运营商购买，而且需要花时间搭建。一种是VPN, VPN包括今天说的IPSec VPN和MPLS VPN。这个还需要在公有云厂商那边购买VPN网关，将其和私有云的网关联通</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-07-10 08:29:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/d8/52/034eb0f1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Liutsing</span>
  </div>
  <div class="_2_QraFYR_0">能讲讲VPN跟ngrok内网穿透的区别么？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-10 08:17:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/60/c7/a147b71b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Fisher</span>
  </div>
  <div class="_2_QraFYR_0">花了两个小时做了笔记，除了最后的 MPLS VPN前面的基本都理解了<br><br>有一个问题，在 DH 算法生成对称密钥 K 的时候，需要交换公开质数 pq 然后生成公钥 AB，交换 AB 生成密钥 K<br><br>这个交换过程虽然没有直接交换密钥，但是如果我是个中间人，拿到了所有的材料，我也是可以生成同样的密钥的吧？那这样怎么保证安全性？还是说我理解的不对，这个过程没法出现中间人？希望老师能够解答一下<br><br>做的笔记：https:&#47;&#47;mubu.com&#47;doc&#47;1cZYndRrAg</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不可以的，私钥截获不到</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-07-07 15:45:26</div>
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
  <div class="_2_QraFYR_0">我有一个问题, 就是私钥交换协议这么厉害,为什么不在HTTPS协议上也加入这个呢? 这样就不需要证书了呀?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 两个的作用不一样，证书是证明非对称秘钥的证明</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-23 16:36:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/1a/e5/6899701e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>favorlm</span>
  </div>
  <div class="_2_QraFYR_0">1通过路由端口映射，或者vpn<br>2.移动端上网需要gprs协议层</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-07-06 09:28:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/29/2a/9079f152.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>谢真</span>
  </div>
  <div class="_2_QraFYR_0">这篇文章讲的太好了，信息量很大，非常解惑，以前只知道概念，不知道有什么用，现在知道协议的由来了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-27 09:28:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/61/4f/e0b71e72.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我是谁</span>
  </div>
  <div class="_2_QraFYR_0">感觉越来越难了，没办法多看几遍吧</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-07-06 09:16:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/f7/b1/982ea185.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Frank</span>
  </div>
  <div class="_2_QraFYR_0">老师，独家网络协议图，大家都需要啊。 公有云与私有云 还是用 Mpls 。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-07-06 08:47:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/58/97/8e14e7d0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>楚人</span>
  </div>
  <div class="_2_QraFYR_0">老师能简单介绍一下专线吗？专线架设等</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-15 09:58:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/37/3f/a9127a73.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>KK</span>
  </div>
  <div class="_2_QraFYR_0">就是一个VPN，居然有这么复杂的交互在里面。真的涨姿势啦。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-06 23:45:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/6a8fRQFxX5VXOpRKyYibsemKwDMexMxkzZOBquPo6T4HOcYicBiaTcqibDoTIhZSjVjF3nKXTEGDYOGPt2xqqwiawjg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>咩咩咩</span>
  </div>
  <div class="_2_QraFYR_0">想弱弱问一下:文中VPN和平时说的挂个VPN翻墙是两个不同的概念吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 一样的，都是隧道</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-28 12:33:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83eov38ZkwCyNoBdr5drgX0cp2eOGCv7ibkhUIqCvcnFk8FyUIS6K4gHXIXh0fu7TB67jaictdDlic4OwQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>珠闪闪</span>
  </div>
  <div class="_2_QraFYR_0">看的有些懵。意图是来了解一下VPN如何连接丝网和公网连接的技术，老师直接讲到报文层次，没有太多的介绍网络如何打通进行业务传输。同求老师如何这么深刻理解这些网络技术的，感叹跟老师的差距真实望尘莫及~</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 哦，搭建vpn没有讲的，不过网上有很多这种文章的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-28 14:30:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/f1/15/8fcf8038.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>William</span>
  </div>
  <div class="_2_QraFYR_0">请问老师打标签的方式为什么比查路由快?打标签每跳路由器还得写数据包，查路由表查找到之后转发出去就行了。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-21 15:42:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/ae/e0/3e636955.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>李博越</span>
  </div>
  <div class="_2_QraFYR_0">最近用minikube本地搭建个环境玩玩，发现了很多镜像pull超时的问题，我把本地iterm设置了socks5代理以后，手工pull镜像还是失败：<br>➜  ~ minikube ssh -- docker pull k8s.gcr.io&#47;pause-amd64:3.1<br>Error response from daemon: Get https:&#47;&#47;k8s.gcr.io&#47;v2&#47;: net&#47;http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)<br>尝试curl看看代理是否成功：<br>➜  ~ curl -ik &quot;https:&#47;&#47;k8s.gcr.io&#47;v2&#47;&quot;<br>HTTP&#47;1.1 200 Connection established<br>尝试ping，却ping不通k8s.gcr.io域名<br>➜  ~ ping k8s.gcr.io<br>PING googlecode.l.googleusercontent.com (64.233.189.82): 56 data bytes<br>Request timeout for icmp_seq 0<br>Request timeout for icmp_seq 1<br>Request timeout for icmp_seq 2<br><br>查了下socks5应该是属于会话层的，而icmp是三层协议，因此ping没有走代理所以翻不了墙，然后curl走应用层因此被代理了所以返回200，但是为啥执行minikube ssh -- docker pull k8s.gcr.io&#47;pause-amd64:3.1拉不下来镜像呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-05 11:31:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/61/44/66cd6f3f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>A7</span>
  </div>
  <div class="_2_QraFYR_0">终于开始上难度了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-07-09 09:50:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/5c/02/e7af1750.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>teddytyy</span>
  </div>
  <div class="_2_QraFYR_0">既然密钥传输是可行的，那https为啥不直接用对称加密？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-22 17:31:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://wx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTIa4aLtpt1wiagBzKGKicQX3KyaulgVzmIdaSj38yYWXPBpK5gjw7Gp4EicTzq34R6raTh1ftAXBHqNA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>大旗</span>
  </div>
  <div class="_2_QraFYR_0">SDW 和MPLS的区别在哪里呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 没研究过SDW </p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-04 10:11:00</div>
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
  <div class="_2_QraFYR_0">IPSec在隧道模式下，新IP地址的如何加上去的呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 隧道里面，也可以看成一个局域网</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-07 17:12:06</div>
  </div>
</div>
</div>
</li>
</ul>