<audio title="第3讲 _ ifconfig：最熟悉又陌生的命令行" src="https://static001.geekbang.org/resource/audio/2b/da/2be5885efb35636c8351bf2058aba2da.mp3" controls="controls"></audio> 
<p>上一节结尾给你留的一个思考题是，你知道怎么查看IP地址吗？</p><p>当面试听到这个问题的时候，面试者常常会觉得走错了房间。我面试的是技术岗位啊，怎么问这么简单的问题？</p><p>的确，即便没有专业学过计算机的人，只要倒腾过电脑，重装过系统，大多也会知道这个问题的答案：在Windows上是ipconfig，在Linux上是ifconfig。</p><p>那你知道在Linux上还有什么其他命令可以查看IP地址吗？答案是ip addr。如果回答不上来这个问题，那你可能没怎么用过Linux。</p><p>那你知道ifconfig和ip addr的区别吗？这是一个有关net-tools和iproute2的“历史”故事，你刚来到第三节，暂时不用了解这么细，但这也是一个常考的知识点。</p><p>想象一下，你登录进入一个被裁剪过的非常小的Linux系统中，发现既没有ifconfig命令，也没有ip addr命令，你是不是感觉这个系统压根儿没法用？这个时候，你可以自行安装net-tools和iproute2这两个工具。当然，大多数时候这两个命令是系统自带的。</p><p>安装好后，我们来运行一下ip addr。不出意外，应该会输出下面的内容。</p><pre><code>root@test:~# ip addr
1: lo: &lt;LOOPBACK,UP,LOWER_UP&gt; mtu 65536 qdisc noqueue state UNKNOWN group default 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: &lt;BROADCAST,MULTICAST,UP,LOWER_UP&gt; mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether fa:16:3e:c7:79:75 brd ff:ff:ff:ff:ff:ff
    inet 10.100.122.2/24 brd 10.100.122.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:fec7:7975/64 scope link 
       valid_lft forever preferred_lft forever
</code></pre><p>这个命令显示了这台机器上所有的网卡。大部分的网卡都会有一个IP地址，当然，这不是必须的。在后面的分享中，我们会遇到没有IP地址的情况。</p><!-- [[[read_end]]] --><p><span class="orange">IP地址是一个网卡在网络世界的通讯地址，相当于我们现实世界的门牌号码。</span>既然是门牌号码，不能大家都一样，不然就会起冲突。比方说，假如大家都叫六单元1001号，那快递就找不到地方了。所以，有时候咱们的电脑弹出网络地址冲突，出现上不去网的情况，多半是IP地址冲突了。</p><p>如上输出的结果，10.100.122.2就是一个IP地址。这个地址被点分隔为四个部分，每个部分8个bit，所以IP地址总共是32位。这样产生的IP地址的数量很快就不够用了。因为当时设计IP地址的时候，哪知道今天会有这么多的计算机啊！因为不够用，于是就有了IPv6，也就是上面输出结果里面inet6 fe80::f816:3eff:fec7:7975/64。这个有128位，现在看来是够了，但是未来的事情谁知道呢？</p><p>本来32位的IP地址就不够，还被分成了5类。现在想想，当时分配地址的时候，真是太奢侈了。</p><p><img src="https://static001.geekbang.org/resource/image/fa/5e/fa9a00346b8a83a84f4c00de948a5b5e.jpg?wh=2323*1333" alt=""></p><p>在网络地址中，至少在当时设计的时候，对于A、B、 C类主要分两部分，前面一部分是网络号，后面一部分是主机号。这很好理解，大家都是六单元1001号，我是小区A的六单元1001号，而你是小区B的六单元1001号。</p><p>下面这个表格，详细地展示了A、B、C三类地址所能包含的主机的数量。在后文中，我也会多次借助这个表格来讲解。</p><p><img src="https://static001.geekbang.org/resource/image/6d/4c/6de9d39a68520c8e3b75aa919fadb24c.jpg?wh=2323*673" alt=""></p><p>这里面有个尴尬的事情，就是C类地址能包含的最大主机数量实在太少了，只有254个。当时设计的时候恐怕没想到，现在估计一个网吧都不够用吧。而B类地址能包含的最大主机数量又太多了。6万多台机器放在一个网络下面，一般的企业基本达不到这个规模，闲着的地址就是浪费。</p><h2>无类型域间选路（CIDR）</h2><p>于是有了一个折中的方式叫作<strong>无类型域间选路</strong>，简称<strong>CIDR</strong>。这种方式打破了原来设计的几类地址的做法，将32位的IP地址一分为二，前面是<strong>网络号</strong>，后面是<strong>主机号</strong>。从哪里分呢？你如果注意观察的话可以看到，10.100.122.2/24，这个IP地址中有一个斜杠，斜杠后面有个数字24。这种地址表示形式，就是CIDR。后面24的意思是，32位中，前24位是网络号，后8位是主机号。</p><p>伴随着CIDR存在的，一个是<strong>广播地址</strong>，10.100.122.255。如果发送这个地址，所有10.100.122网络里面的机器都可以收到。另一个是<strong>子网掩码</strong>，255.255.255.0。</p><p>将子网掩码和IP地址进行AND计算。前面三个255，转成二进制都是1。1和任何数值取AND，都是原来数值，因而前三个数不变，为10.100.122。后面一个0，转换成二进制是0，0和任何数值取AND，都是0，因而最后一个数变为0，合起来就是10.100.122.0。这就是<strong>网络号</strong>。<strong>将子网掩码和IP地址按位计算AND，就可得到网络号。</strong></p><h2>公有IP地址和私有IP地址</h2><p>在日常的工作中，几乎不用划分A类、B类或者C类，所以时间长了，很多人就忘记了这个分类，而只记得CIDR。但是有一点还是要注意的，就是公有IP地址和私有IP地址。</p><p><img src="https://static001.geekbang.org/resource/image/df/a9/df90239efec6e35880b9abe55089ffa9.jpg?wh=2359*736" alt=""></p><p>我们继续看上面的表格。表格最右列是私有IP地址段。平时我们看到的数据中心里，办公室、家里或学校的IP地址，一般都是私有IP地址段。因为这些地址允许组织内部的IT人员自己管理、自己分配，而且可以重复。因此，你学校的某个私有IP地址段和我学校的可以是一样的。</p><p>这就像每个小区有自己的楼编号和门牌号，你们小区可以叫6栋，我们小区也叫6栋，没有任何问题。但是一旦出了小区，就需要使用公有IP地址。就像人民路888号，是国家统一分配的，不能两个小区都叫人民路888号。</p><p>公有IP地址有个组织统一分配，你需要去买。如果你搭建一个网站，给你学校的人使用，让你们学校的IT人员给你一个IP地址就行。但是假如你要做一个类似网易163这样的网站，就需要有公有IP地址，这样全世界的人才能访问。</p><p>表格中的192.168.0.x是最常用的私有IP地址。你家里有Wi-Fi，对应就会有一个IP地址。一般你家里地上网设备不会超过256个，所以/24基本就够了。有时候我们也能见到/16的CIDR，这两种是最常见的，也是最容易理解的。</p><p>不需要将十进制转换为二进制32位，就能明显看出192.168.0是网络号，后面是主机号。而整个网络里面的第一个地址192.168.0.1，往往就是你这个私有网络的出口地址。例如，你家里的电脑连接Wi-Fi，Wi-Fi路由器的地址就是192.168.0.1，而192.168.0.255就是广播地址。一旦发送这个地址，整个192.168.0网络里面的所有机器都能收到。</p><p>但是也不总都是这样的情况。因此，其他情况往往就会很难理解，还容易出错。</p><h2>举例：一个容易“犯错”的CIDR</h2><p>我们来看16.158.165.91/22这个CIDR。求一下这个网络的第一个地址、子网掩码和广播地址。</p><p>你要是上来就写16.158.165.1，那就大错特错了。</p><p>/22不是8的整数倍，不好办，只能先变成二进制来看。16.158的部分不会动，它占了前16位。中间的165，变为二进制为‭10100101‬。除了前面的16位，还剩6位。所以，这8位中前6位是网络号，16.158.&lt;101001&gt;，而&lt;01&gt;.91是机器号。</p><p>第一个地址是16.158.&lt;101001&gt;&lt;00&gt;.1，即16.158.164.1。子网掩码是255.255.&lt;111111&gt;&lt;00&gt;.0，即255.255.252.0。广播地址为16.158.&lt;101001&gt;&lt;11&gt;.255，即16.158.167.255。</p><p>这五类地址中，还有一类D类是<strong>组播地址</strong>。使用这一类地址，属于某个组的机器都能收到。这有点类似在公司里面大家都加入了一个邮件组。发送邮件，加入这个组的都能收到。组播地址在后面讲述VXLAN协议的时候会提到。</p><p>讲了这么多，才讲了上面的输出结果中很小的一部分，是不是觉得原来并没有真的理解ip addr呢？我们接着来分析。</p><p>在IP地址的后面有个scope，对于eth0这张网卡来讲，是global，说明这张网卡是可以对外的，可以接收来自各个地方的包。对于lo来讲，是host，说明这张网卡仅仅可以供本机相互通信。</p><p>lo全称是<strong>loopback</strong>，又称<strong>环回接口</strong>，往往会被分配到127.0.0.1这个地址。这个地址用于本机通信，经过内核处理后直接返回，不会在任何网络中出现。</p><h2>MAC地址</h2><p>在IP地址的上一行是link/ether fa:16:3e:c7:79:75 brd ff:ff:ff:ff:ff:ff，这个被称为<strong>MAC地址</strong>，是一个网卡的物理地址，用十六进制，6个byte表示。</p><p>MAC地址是一个很容易让人“误解”的地址。因为MAC地址号称全局唯一，不会有两个网卡有相同的MAC地址，而且网卡自生产出来，就带着这个地址。很多人看到这里就会想，既然这样，整个互联网的通信，全部用MAC地址好了，只要知道了对方的MAC地址，就可以把信息传过去。</p><p>这样当然是不行的。 <strong>一个网络包要从一个地方传到另一个地方，除了要有确定的地址，还需要有定位功能。</strong> 而有门牌号码属性的IP地址，才是有远程定位功能的。</p><p>例如，你去杭州市网商路599号B楼6层找刘超，你在路上问路，可能被问的人不知道B楼是哪个，但是可以给你指网商路怎么去。但是如果你问一个人，你知道这个身份证号的人在哪里吗？可想而知，没有人知道。</p><p><span class="orange">MAC地址更像是身份证，是一个唯一的标识。</span>它的唯一性设计是为了组网的时候，不同的网卡放在一个网络里面的时候，可以不用担心冲突。从硬件角度，保证不同的网卡有不同的标识。</p><p>MAC地址是有一定定位功能的，只不过范围非常有限。你可以根据IP地址，找到杭州市网商路599号B楼6层，但是依然找不到我，你就可以靠吼了，大声喊身份证XXXX的是哪位？我听到了，我就会站起来说，是我啊。但是如果你在上海，到处喊身份证XXXX的是哪位，我不在现场，当然不会回答，因为我在杭州不在上海。</p><p>所以，MAC地址的通信范围比较小，局限在一个子网里面。例如，从192.168.0.2/24访问192.168.0.3/24是可以用MAC地址的。一旦跨子网，即从192.168.0.2/24到192.168.1.2/24，MAC地址就不行了，需要IP地址起作用了。</p><h2>网络设备的状态标识</h2><p>解析完了MAC地址，我们再来看 &lt;BROADCAST,MULTICAST,UP,LOWER_UP&gt;是干什么的？这个叫做<strong>net_device flags</strong>，<strong>网络设备的状态标识</strong>。</p><p>UP表示网卡处于启动的状态；BROADCAST表示这个网卡有广播地址，可以发送广播包；MULTICAST表示网卡可以发送多播包；LOWER_UP表示L1是启动的，也即网线插着呢。MTU1500是指什么意思呢？是哪一层的概念呢？最大传输单元MTU为1500，这是以太网的默认值。</p><p>上一节，我们讲过网络包是层层封装的。MTU是二层MAC层的概念。MAC层有MAC的头，以太网规定正文部分不允许超过1500个字节。正文里面有IP的头、TCP的头、HTTP的头。如果放不下，就需要分片来传输。</p><p>qdisc pfifo_fast是什么意思呢？qdisc全称是<strong>queueing discipline</strong>，中文叫<strong>排队规则</strong>。内核如果需要通过某个网络接口发送数据包，它都需要按照为这个接口配置的qdisc（排队规则）把数据包加入队列。</p><p>最简单的qdisc是pfifo，它不对进入的数据包做任何的处理，数据包采用先入先出的方式通过队列。pfifo_fast稍微复杂一些，它的队列包括三个波段（band）。在每个波段里面，使用先进先出规则。</p><p>三个波段（band）的优先级也不相同。band 0的优先级最高，band 2的最低。如果band 0里面有数据包，系统就不会处理band 1里面的数据包，band 1和band 2之间也是一样。</p><p>数据包是按照服务类型（<strong>Type of Service，TOS</strong>）被分配到三个波段（band）里面的。TOS是IP头里面的一个字段，代表了当前的包是高优先级的，还是低优先级的。</p><p>队列是个好东西，后面我们讲云计算中的网络的时候，会有很多用户共享一个网络出口的情况，这个时候如何排队，每个队列有多粗，队列处理速度应该怎么提升，我都会详细为你讲解。</p><h2>小结</h2><p>怎么样，看起来很简单的一个命令，里面学问很大吧？通过这一节，希望你能记住以下的知识点，后面都能用得上：</p><ul>
<li>
<p>IP是地址，有定位功能；MAC是身份证，无定位功能；</p>
</li>
<li>
<p>CIDR可以用来判断是不是本地人；</p>
</li>
<li>
<p>IP分公有的IP和私有的IP。后面的章节中我会谈到“出国门”，就与这个有关。</p>
</li>
</ul><p>最后，给你留两个思考题。</p><ol>
<li>你知道net-tools和iproute2的“历史”故事吗？</li>
<li>这一节讲的是如何查看IP地址，那你知道IP地址是怎么来的吗？</li>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c5/4d/ac0f0371.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>猿来是你</span>
  </div>
  <div class="_2_QraFYR_0">能讲的详细些吗？非网络科班出身，理解不透彻！不要一带而过！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 第三讲主要通过ip addr命令对于网络相关概念有一个总体的介绍，深入了其中一部分，如果您觉得其他部门讲的粗略还不能理解透彻，到这一节可以先忽略，应该不影响。当时设计讲网络的时候，其实就有个难点，相互关联性太强，二层会依赖四层，四层也会依赖二层，如果每一点都深挖的另一个问题就是一下子深入进去，让初学者晕了。所以我想用的方式是从平时接触到的东西开始逐层深入，如果文中说这里不详述的部分，其实是对当前知识点的理解尚不构成阻碍，等构成阻碍了，就会讲清楚。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-05-27 06:21:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>船新版本</span>
  </div>
  <div class="_2_QraFYR_0">cidr那块将IP和子网掩码都转成二进制列出来对比的话会比较直观很多，第一遍看到这块的时候有点懵</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这段纠结了好久，完全二进制的话，音频就没法读了。现在这样😊好像读起来也有点别扭。是要照顾上班路上只听音频的朋友</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-05-26 14:12:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/6e/c6/4a7b2517.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Will王志翔(大象)</span>
  </div>
  <div class="_2_QraFYR_0">第三讲笔记<br><br><br># 面试考点：<br>	<br>1.  ip addr → 不知道基本没有用Linux<br>2.  ifconfig 和 ip addr 的区别吗？<br>3.  CIDR<br>4.  共有IP和私有IP<br>5.  MAC地址<br>6.  网络设备的状态标识<br><br># 知识点：<br>	<br>## 核心：<br>	<br>1. IP设计时犯的错误？<br>	<br>低估了未来网络的发展，32位地址不够用。于是有了现在IPv6（128位）<br>分类错误。分成了5类。C类太少，B类太多。C类254个，网络都不够；D类6万多，给企业都太多。<br>	<br>2. 那后来者如何弥补IP设计者犯的错误呢？<br>	<br>CIDR，无类型域间选路。<br>打破原来几类地址设计的做法，将32位IP地址一分二，前者网络号，后者主机号。<br>如何分呢？<br>栗子：10.100.122.2&#47;24<br>24 = 前24位是网络号，那么后8位就是主机号。<br>那如何用？<br>如发送行信息给 10.100.122.255<br>所有以 10.100.122... 开头的机器都能收到。<br>于是有了两个概念：<br>广播地址：10.100.122.255<br>子网掩码：255.255.255.0 -&gt; AND 得到网络号。<br>	<br>3. 每一个城市都有人民广场，IP设计是如何解决的？<br>	<br>公有IP地址和私有IP地址。<br>搭建世界人民都可以访问的网站，需要共有IP地址<br>搭建只有学校同学使用饿的网站，只要私有IP地址<br>例子1: Wi-Fi<br> 192.168.0.x 是最常用的私有 IP 地址<br>192.168.0 是网络号<br>192.168.0.1，往往就是你这个私有网络的出口地址<br>192.168.0.255 就是广播地址。一旦发送这个地址，整个 192.168.0 网络里面的所有机器都能收到。<br><br>例子2: 16.158.165.91&#47;22<br><br>4. 如何理解MAC地址？<br>	<br>如果说IP是地址，有定位功能。那Mac就是身份证，唯一识别。<br>	<br>## 琐碎：<br>	<br>5. 讲了ABC，那是D类是什么？<br>	<br>D 类是组播地址。使用这一类地址，属于某个组的机器都能收到。这有点类似在公司里面大家都加入了一个邮件组。发送邮件，加入这个组的都能收到。组播地址在后面讲述 VXLAN 协议的时候会提到。<br>	<br>6. IP地址scope是什么意思？<br>	<br>对于 eth0 这张网卡来讲，是 global，说明这张网卡是可以对外的，可以接收来自各个地方的包。对于 lo 来讲，是 host，说明这张网卡仅仅可以供本机相互通信。<br>	<br>7. 那lo是什么意思？<br>	<br>lo 全称是loopback，又称环回接口，往往会被分配到 127.0.0.1 这个地址。这个地址用于本机通信，经过内核处理后直接返回，不会在任何网络中出现。<br>	<br>8. &lt; BROADCAST,MULTICAST,UP,LOWER_UP &gt; 是干什么的？<br>	<br>net_device flags，网络设备的状态标识。<br>UP 表示网卡处于启动的状态；<br>BROADCAST 表示这个网卡有广播地址，可以发送广播包；<br>MULTICAST 表示网卡可以发送多播包；<br>LOWER_UP 表示 L1 是启动的，也即网线插着呢。<br>	<br>9. MTU1500 是指什么意思呢？是哪一层的概念？<br>	<br>最大传输单元 MTU 为 1500，这是以太网的默认值。<br>MTU 是二层 MAC 层的概念。MAC 层有 MAC 的头，以太网规定连 MAC 头带正文合起来，不允许超过 1500 个字节。<br>	<br>10. qdisc pfifo_fast 是什么意思呢？<br>	<br>排队规则。规定数据包如何进出的。有pfifo, pfifo_fast. <br>	</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-07-07 01:18:04</div>
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
  <div class="_2_QraFYR_0">net-tools起源于BSD，自2001年起，Linux社区已经对其停止维护，而iproute2旨在取代net-tools，并提供了一些新功能。一些Linux发行版已经停止支持net-tools，只支持iproute2。<br>net-tools通过procfs(&#47;proc)和ioctl系统调用去访问和改变内核网络配置，而iproute2则通过netlink套接字接口与内核通讯。<br>net-tools中工具的名字比较杂乱，而iproute2则相对整齐和直观，基本是ip命令加后面的子命令。<br>虽然取代意图很明显，但是这么多年过去了，net-tool依然还在被广泛使用，最好还是两套命令都掌握吧。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 太赞了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-05-23 22:47:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/a6/6b/12d87713.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Jealone</span>
  </div>
  <div class="_2_QraFYR_0">MTU 大小是不包含二层头部和尾部的，MTU 1500表示二层MAC帧大小不超过1518. MAC 头14 字节，尾4字节。可以抓包验证</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-05-29 13:50:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/4e/38/3faa8377.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>登高</span>
  </div>
  <div class="_2_QraFYR_0">mac是身份证，ip是地址<br>透彻</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-05-23 08:35:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/e9/e8/31df61df.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>军秋</span>
  </div>
  <div class="_2_QraFYR_0">现在很多工具都可以更改本机的MAC地址，也就是网络上存在很多MAC地址被更改成一样的，然而并没有出现通讯异常或者混乱这是为什么？<br><br>这是一个别人的留言，老师回答了会出问题，但没回答为什么？<br>MAC在一个局域网内冲突才会影响网络通讯，局域网外是通过IP定位，所以不同局域网的网络设备MAC一样是不会有通讯问题的。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-05-24 23:47:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>来生树</span>
  </div>
  <div class="_2_QraFYR_0">看了3篇，精彩阿，这个课程定价，严重定低了。应该299起嘛</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 哈哈，谢谢</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-05-23 12:36:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83eqoz1fVUGgXYRheScrGHctMxicaxIYnjeoZpI7Ga4rxvia9ykkQ4berKb5oeZjfyo3iaVCxicEejmtibxQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Bill</span>
  </div>
  <div class="_2_QraFYR_0">补充内核恐慌的老梗：<br>不知道有没有内核恐慌的水友(̿▀̿̿Ĺ̯̿̿▀̿ ̿)̄<br><br>1、1.1.1.1 不是测试用的，原来一直没分配，现在被用来做一个DNS了，宣传是比谷歌等公司的dns服务<br>更保护用户隐私。<br><br>2、IP地址255.255.255.255，代表有限广播，它的目标是网络中的所有主机。<br><br>3、IP地址0.0.0.0，通常代表未知的源主机。当主机采用DHCP动态获取IP地址而无法获得合法IP地址时，会用IP地址0.0.0.0来表示源主机IP地址未知。<br><br>4、NID不能以数字127开头。NID 127被保留给内部回送函数,作为本机循环测试使用。<br>例如，使用命令ping 127.0.0.1测试TCP&#47;IP协议栈是否正确安装。在路由器中，同样支持循环测试地址的使用。<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-07-08 03:09:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/4e/f7/618a8e52.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>周磊</span>
  </div>
  <div class="_2_QraFYR_0">大学学的计算机网络课程关于ip地址要比这详细多，但刘老师讲的更为生动，联系生活中的场景做比喻，读后印象深刻。<br>建议读起来困难的同学先了解下二进制以及与十进制转换，再就是找相关的资料补充一下。<br>很多东西第一遍读不懂没关系，无论你不理解或忘记多少，当你在另一个地方再次看到这些东西时，你便会有种亲切感，以前模糊的地方会在这次变得清晰一些。经过多次的接触同一个知识点，你会越来越清楚直到透彻。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞，毕竟大学的课时比较多</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-05-31 01:53:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/7a/0a/0ce5c232.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>吕</span>
  </div>
  <div class="_2_QraFYR_0">请教一下：我阿里云的多台机器，有172.16.2.145 和172.16.3.28这两个是怎么联通的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果&#47;16，就是一个网段的。如果是&#47;24，则中间会有路由器</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-16 16:53:39</div>
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
  <div class="_2_QraFYR_0">A B C 类别表里A类数据有问题 应该是1:0:0:1-126:255:255:254 建议检查以下B 和C类</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: (⊙o⊙)哇，好严谨。A类IP的地址第一个字段范围是0~127，但是由于全0和全1的地址用作特殊用途，实际可指派的第一个字段范围是1~126。所以仔细搜了一下，如果较真的考试题的说法是，A类地址范围和A类有效地址范围。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-01 21:13:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4b/0f/d747ed96.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Steve</span>
  </div>
  <div class="_2_QraFYR_0">推荐两本书籍：《图解 TCP&#47;IP》、《wireshark 数据包分析实战》</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-30 21:24:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/88/9c/cbc463e6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>仰望星空</span>
  </div>
  <div class="_2_QraFYR_0">解题思路<br>1、首先求子网掩码，子网掩码掩盖网络号，暴露主机号，用1表示掩盖，用0表示暴露。22表示网络号有22位，那么主机号有10位，因此子网掩码的二进制的值为：<br>11111111.11111111.11111100.00000000，转换为十进制：255.255.252.0<br>2、再求网络号，将子网掩码和 IP 地址按位计算 AND，就可得到网络号，<br>16.158.165.91的二进制：<br>‭00010000.‬10011110.‭10100101‬.‭01011011‬<br>AND<br>11111111.11111111.11111100.00000000<br>=<br>00010000.10011110.10100100.00000000<br>=<br>16.158.164.0，<br>那么第一个网络号（主机号的最后一位是1）为：<br>00010000.10011110.10100100.00000001=16.158.164.1，<br>最后一个网路号也即广播地址（主机号的所有位是1）为：<br>00010000.10011110.10100111.11111111=16.158.167.255</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-18 11:35:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/43/1a/11b56711.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>秋生</span>
  </div>
  <div class="_2_QraFYR_0">刘老师，您好，您举例说的那个容易犯错的CIDR问题里，为什么第一个地址和子网掩码都是补上00，而广播地址是补上11；本人是个小白，希望能得到您的解答，谢谢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-05-23 11:15:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/6e/c6/4a7b2517.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Will王志翔(大象)</span>
  </div>
  <div class="_2_QraFYR_0">第三讲笔记<br><br><br># 面试考点：<br>	<br>1.  ip addr → 不知道基本没有用Linux<br>2.  ifconfig 和 ip addr 的区别吗？<br>3.  CIDR<br>4.  共有IP和私有IP<br>5.  MAC地址<br>6.  网络设备的状态标识<br><br># 知识点：<br>	<br>## 核心：<br>	<br>1. IP设计时犯的错误？<br>	<br>低估了未来网络的发展，32位地址不够用。于是有了现在IPv6（128位）<br>分类错误。分成了5类。C类太少，B类太多。C类254个，网络都不够；D类6万多，给企业都太多。<br>	<br>2. 那后来者如何弥补IP设计者犯的错误呢？<br>	<br>CIDR，无类型域间选路。<br>打破原来几类地址设计的做法，将32位IP地址一分二，前者网络号，后者主机号。<br>如何分呢？<br>栗子：10.100.122.2&#47;24<br>24 = 前24位是网络号，那么后8位就是主机号。<br>那如何用？<br>如发送行信息给 10.100.122.255<br>所有以 10.100.122... 开头的机器都能收到。<br>于是有了两个概念：<br>广播地址：10.100.122.255<br>子网掩码：255.255.255.0 -&gt; AND 得到网络号。<br>	<br>3. 每一个城市都有人民广场，IP设计是如何解决的？<br>	<br>公有IP地址和私有IP地址。<br>搭建世界人民都可以访问的网站，需要共有IP地址<br>搭建只有学校同学使用饿的网站，只要私有IP地址<br>例子1: Wi-Fi<br> 192.168.0.x 是最常用的私有 IP 地址<br>192.168.0 是网络号<br>192.168.0.1，往往就是你这个私有网络的出口地址<br>192.168.0.255 就是广播地址。一旦发送这个地址，整个 192.168.0 网络里面的所有机器都能收到。<br><br>例子2: 16.158.165.91&#47;22<br><br>4. 如何理解MAC地址？<br>	<br>如果说IP是地址，有定位功能。那Mac就是身份证，唯一识别。<br>	<br>## 琐碎：<br>	<br>5. 讲了ABC，那是D类是什么？<br>	<br>D 类是组播地址。使用这一类地址，属于某个组的机器都能收到。这有点类似在公司里面大家都加入了一个邮件组。发送邮件，加入这个组的都能收到。组播地址在后面讲述 VXLAN 协议的时候会提到。<br>	<br>6. IP地址scope是什么意思？<br>	<br>对于 eth0 这张网卡来讲，是 global，说明这张网卡是可以对外的，可以接收来自各个地方的包。对于 lo 来讲，是 host，说明这张网卡仅仅可以供本机相互通信。<br>	<br>7. 那lo是什么意思？<br>	<br>lo 全称是loopback，又称环回接口，往往会被分配到 127.0.0.1 这个地址。这个地址用于本机通信，经过内核处理后直接返回，不会在任何网络中出现。<br>	<br>8. &lt; BROADCAST,MULTICAST,UP,LOWER_UP &gt; 是干什么的？<br>	<br>net_device flags，网络设备的状态标识。<br>UP 表示网卡处于启动的状态；<br>BROADCAST 表示这个网卡有广播地址，可以发送广播包；<br>MULTICAST 表示网卡可以发送多播包；<br>LOWER_UP 表示 L1 是启动的，也即网线插着呢。<br>	<br>9. MTU1500 是指什么意思呢？是哪一层的概念？<br>	<br>最大传输单元 MTU 为 1500，这是以太网的默认值。<br>MTU 是二层 MAC 层的概念。MAC 层有 MAC 的头，以太网规定连 MAC 头带正文合起来，不允许超过 1500 个字节。<br>	<br>10. qdisc pfifo_fast 是什么意思呢？<br>	<br>排队规则。规定数据包如何进出的。有pfifo, pfifo_fast. <br>	</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-07-07 01:18:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>来生树</span>
  </div>
  <div class="_2_QraFYR_0">采精，唯一不足，这个课程定价严重定低了，哈哈</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-05-23 12:37:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/5b/b9/d07de7c6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>FLOSS</span>
  </div>
  <div class="_2_QraFYR_0">现在很多工具都可以更改本机的MAC地址，也就是网络上存在很多MAC地址被更改成一样的，然而并没有出现通讯异常或者混乱这是为什么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 会啊，如果你创建虚拟机，复制的时候，没有原则重新生成mac，你就发现你连不上了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-05-24 18:47:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/5b/b9/d07de7c6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>FLOSS</span>
  </div>
  <div class="_2_QraFYR_0">还是没理解，我和同事的电脑就是相同的MAC地址，我们再各自的家里上网都是正常的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 哈哈，不在一个局域网，当然没问题了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-05-25 17:53:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Norman</span>
  </div>
  <div class="_2_QraFYR_0">老师可以给推荐一本网络协议相关的书吗？我是小白，之前没有系统学习过网络协议，想好好看一下</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-05-23 11:36:21</div>
  </div>
</div>
</div>
</li>
</ul>