<audio title="第4讲 _ DHCP与PXE：IP是怎么来的，又是怎么没的？" src="https://static001.geekbang.org/resource/audio/8f/35/8fcc98e770f39dc43af82f717ffd4f35.mp3" controls="controls"></audio> 
<p>上一节，我们讲了IP的一些基本概念。如果需要和其他机器通讯，我们就需要一个通讯地址，我们需要给网卡配置这么一个地址。</p><h2>如何配置IP地址？</h2><p>那如何配置呢？如果有相关的知识和积累，你可以用命令行自己配置一个地址。可以使用ifconfig，也可以使用ip addr。设置好了以后，用这两个命令，将网卡up一下，就可以开始工作了。</p><p><strong>使用net-tools：</strong></p><pre><code>$ sudo ifconfig eth1 10.0.0.1/24
$ sudo ifconfig eth1 up
</code></pre><p><strong>使用iproute2：</strong></p><pre><code>$ sudo ip addr add 10.0.0.1/24 dev eth1
$ sudo ip link set up eth1
</code></pre><p>你可能会问了，自己配置这个自由度太大了吧，我是不是配置什么都可以？如果配置一个和谁都不搭边的地址呢？例如，旁边的机器都是192.168.1.x，我非得配置一个16.158.23.6，会出现什么现象呢？</p><p>不会出现任何现象，就是包发不出去呗。为什么发不出去呢？我来举例说明。</p><p>192.168.1.6就在你这台机器的旁边，甚至是在同一个交换机上，而你把机器的地址设为了16.158.23.6。在这台机器上，你企图去ping192.168.1.6，你觉得只要将包发出去，同一个交换机的另一台机器马上就能收到，对不对？</p><p>可是Linux系统不是这样的，它没你想的那么智能。你用肉眼看到那台机器就在旁边，它则需要根据自己的逻辑进行处理。</p><!-- [[[read_end]]] --><p>还记得我们在第二节说过的原则吗？<strong>只要是在网络上跑的包，都是完整的，可以有下层没上层，绝对不可能有上层没下层。</strong></p><p>所以，你看着它有自己的源IP地址16.158.23.6，也有目标IP地址192.168.1.6，但是包发不出去，这是因为MAC层还没填。</p><p>自己的MAC地址自己知道，这个容易。但是目标MAC填什么呢？是不是填192.168.1.6这台机器的MAC地址呢？</p><p>当然不是。Linux首先会判断，要去的这个地址和我是一个网段的吗，或者和我的一个网卡是同一网段的吗？只有是一个网段的，它才会发送ARP请求，获取MAC地址。如果发现不是呢？</p><p><strong>Linux默认的逻辑是，如果这是一个跨网段的调用，它便不会直接将包发送到网络上，而是企图将包发送到网关。</strong></p><p>如果你配置了网关的话，Linux会获取网关的MAC地址，然后将包发出去。对于192.168.1.6这台机器来讲，虽然路过它家门的这个包，目标IP是它，但是无奈MAC地址不是它的，所以它的网卡是不会把包收进去的。</p><p>如果没有配置网关呢？那包压根就发不出去。</p><p>如果将网关配置为192.168.1.6呢？不可能，Linux不会让你配置成功的，因为网关要和当前的网络至少一个网卡是同一个网段的，怎么可能16.158.23.6的网关是192.168.1.6呢？</p><p>所以，当你需要手动配置一台机器的网络IP时，一定要好好问问你的网络管理员。如果在机房里面，要去网络管理员那里申请，让他给你分配一段正确的IP地址。当然，真正配置的时候，一定不是直接用命令配置的，而是放在一个配置文件里面。<strong>不同系统的配置文件格式不同，但是无非就是CIDR、子网掩码、广播地址和网关地址</strong>。</p><h2>动态主机配置协议（DHCP）</h2><p>原来配置IP有这么多门道儿啊。你可能会问了，配置了IP之后一般不能变的，配置一个服务端的机器还可以，但是如果是客户端的机器呢？我抱着一台笔记本电脑在公司里走来走去，或者白天来晚上走，每次使用都要配置IP地址，那可怎么办？还有人事、行政等非技术人员，如果公司所有的电脑都需要IT人员配置，肯定忙不过来啊。</p><p>因此，我们需要有一个自动配置的协议，也就是<strong>动态主机配置协议（Dynamic Host Configuration Protocol）</strong>，简称<strong>DHCP</strong>。</p><p>有了这个协议，网络管理员就轻松多了。他只需要配置一段共享的IP地址。每一台新接入的机器都通过DHCP协议，来这个共享的IP地址里申请，然后自动配置好就可以了。等人走了，或者用完了，还回去，这样其他的机器也能用。</p><p>所以说，<strong>如果是数据中心里面的服务器，IP一旦配置好，基本不会变，这就相当于买房自己装修。DHCP的方式就相当于租房。你不用装修，都是帮你配置好的。你暂时用一下，用完退租就可以了。</strong></p><h2>解析DHCP的工作方式</h2><p>当一台机器新加入一个网络的时候，肯定一脸懵，啥情况都不知道，只知道自己的MAC地址。怎么办？先吼一句，我来啦，有人吗？这时候的沟通基本靠“吼”。这一步，我们称为<strong>DHCP Discover。</strong></p><p>新来的机器使用IP地址0.0.0.0发送了一个广播包，目的IP地址为255.255.255.255。广播包封装了UDP，UDP封装了BOOTP。其实DHCP是BOOTP的增强版，但是如果你去抓包的话，很可能看到的名称还是BOOTP协议。</p><p>在这个广播包里面，新人大声喊：我是新来的（Boot request），我的MAC地址是这个，我还没有IP，谁能给租给我个IP地址！</p><p>格式就像这样：</p><p><img src="https://static001.geekbang.org/resource/image/90/81/90b4d41ee38e891031705d987d5d8481.jpg?wh=1405*1141" alt=""></p><p>如果一个网络管理员在网络里面配置了<strong>DHCP Server</strong>的话，他就相当于这些IP的管理员。他立刻能知道来了一个“新人”。这个时候，我们可以体会MAC地址唯一的重要性了。当一台机器带着自己的MAC地址加入一个网络的时候，MAC是它唯一的身份，如果连这个都重复了，就没办法配置了。</p><p>只有MAC唯一，IP管理员才能知道这是一个新人，需要租给它一个IP地址，这个过程我们称为<strong>DHCP Offer</strong>。同时，DHCP Server为此客户保留为它提供的IP地址，从而不会为其他DHCP客户分配此IP地址。</p><p>DHCP Offer的格式就像这样，里面有给新人分配的地址。</p><p><img src="https://static001.geekbang.org/resource/image/a5/6b/a52c8c87b925b52059febe9dfcd6be6b.jpg?wh=1405*1141" alt=""></p><p>DHCP Server仍然使用广播地址作为目的地址，因为，此时请求分配IP的新人还没有自己的IP。DHCP Server回复说，我分配了一个可用的IP给你，你看如何？除此之外，服务器还发送了子网掩码、网关和IP地址租用期等信息。</p><p>新来的机器很开心，它的“吼”得到了回复，并且有人愿意租给它一个IP地址了，这意味着它可以在网络上立足了。当然更令人开心的是，如果有多个DHCP Server，这台新机器会收到多个IP地址，简直受宠若惊。</p><p>它会选择其中一个DHCP Offer，一般是最先到达的那个，并且会向网络发送一个DHCP Request广播数据包，包中包含客户端的MAC地址、接受的租约中的IP地址、提供此租约的DHCP服务器地址等，并告诉所有DHCP Server它将接受哪一台服务器提供的IP地址，告诉其他DHCP服务器，谢谢你们的接纳，并请求撤销它们提供的IP地址，以便提供给下一个IP租用请求者。</p><p><img src="https://static001.geekbang.org/resource/image/cd/fa/cdbcaad24e1a4d24dd724e38f6f043fa.jpg?wh=1405*1141" alt=""></p><p>此时，由于还没有得到DHCP Server的最后确认，客户端仍然使用0.0.0.0为源IP地址、255.255.255.255为目标地址进行广播。在BOOTP里面，接受某个DHCP Server的分配的IP。</p><p>当DHCP Server接收到客户机的DHCP request之后，会广播返回给客户机一个DHCP ACK消息包，表明已经接受客户机的选择，并将这一IP地址的合法租用信息和其他的配置信息都放入该广播包，发给客户机，欢迎它加入网络大家庭。</p><p><img src="https://static001.geekbang.org/resource/image/cc/a9/cca8b0baa4749bb359e453b1b482e1a9.jpg?wh=1405*1141" alt=""></p><p>最终租约达成的时候，还是需要广播一下，让大家都知道。</p><h2>IP地址的收回和续租</h2><p>既然是租房子，就是有租期的。租期到了，管理员就要将IP收回。</p><p>如果不用的话，收回就收回了。就像你租房子一样，如果还要续租的话，不能到了时间再续租，而是要提前一段时间给房东说。DHCP也是这样。</p><p>客户机会在租期过去50%的时候，直接向为其提供IP地址的DHCP Server发送DHCP request消息包。客户机接收到该服务器回应的DHCP ACK消息包，会根据包中所提供的新的租期以及其他已经更新的TCP/IP参数，更新自己的配置。这样，IP租用更新就完成了。</p><p>好了，一切看起来完美。DHCP协议大部分人都知道，但是其实里面隐藏着一个细节，很多人可能不会去注意。接下来，我就讲一个有意思的事情：网络管理员不仅能自动分配IP地址，还能帮你自动安装操作系统！</p><h2>预启动执行环境（PXE）</h2><p>普通的笔记本电脑，一般不会有这种需求。因为你拿到电脑时，就已经有操作系统了，即便你自己重装操作系统，也不是很麻烦的事情。但是，在数据中心里就不一样了。数据中心里面的管理员可能一下子就拿到几百台空的机器，一个个安装操作系统，会累死的。</p><p>所以管理员希望的不仅仅是自动分配IP地址，还要自动安装系统。装好系统之后自动分配IP地址，直接启动就能用了，这样当然最好了！</p><p>这事儿其实仔细一想，还是挺有难度的。安装操作系统，应该有个光盘吧。数据中心里不能用光盘吧，想了一个办法就是，可以将光盘里面要安装的操作系统放在一个服务器上，让客户端去下载。但是客户端放在哪里呢？它怎么知道去哪个服务器上下载呢？客户端总得安装在一个操作系统上呀，可是这个客户端本来就是用来安装操作系统的呀？</p><p>其实，这个过程和操作系统启动的过程有点儿像。首先，启动BIOS。这是一个特别小的小系统，只能干特别小的一件事情。其实就是读取硬盘的MBR启动扇区，将GRUB启动起来；然后将权力交给GRUB，GRUB加载内核、加载作为根文件系统的initramfs文件；然后将权力交给内核；最后内核启动，初始化整个操作系统。</p><p>那我们安装操作系统的过程，只能插在BIOS启动之后了。因为没安装系统之前，连启动扇区都没有。因而这个过程叫做<strong>预启动执行环境（Pre-boot Execution Environment）</strong>，简称<strong>PXE。</strong></p><p>PXE协议分为客户端和服务器端，由于还没有操作系统，只能先把客户端放在BIOS里面。当计算机启动时，BIOS把PXE客户端调入内存里面，就可以连接到服务端做一些操作了。</p><p>首先，PXE客户端自己也需要有个IP地址。因为PXE的客户端启动起来，就可以发送一个DHCP的请求，让DHCP Server给它分配一个地址。PXE客户端有了自己的地址，那它怎么知道PXE服务器在哪里呢？对于其他的协议，都好办，要有人告诉他。例如，告诉浏览器要访问的IP地址，或者在配置中告诉它；例如，微服务之间的相互调用。</p><p>但是PXE客户端启动的时候，啥都没有。好在DHCP Server除了分配IP地址以外，还可以做一些其他的事情。这里有一个DHCP Server的一个样例配置：</p><pre><code>ddns-update-style interim;
ignore client-updates;
allow booting;
allow bootp;
subnet 192.168.1.0 netmask 255.255.255.0
{
option routers 192.168.1.1;
option subnet-mask 255.255.255.0;
option time-offset -18000;
default-lease-time 21600;
max-lease-time 43200;
range dynamic-bootp 192.168.1.240 192.168.1.250;
filename &quot;pxelinux.0&quot;;
next-server 192.168.1.180;
}
</code></pre><p>按照上面的原理，默认的DHCP Server是需要配置的，无非是我们配置IP的时候所需要的IP地址段、子网掩码、网关地址、租期等。如果想使用PXE，则需要配置next-server，指向PXE服务器的地址，另外要配置初始启动文件filename。</p><p>这样PXE客户端启动之后，发送DHCP请求之后，除了能得到一个IP地址，还可以知道PXE服务器在哪里，也可以知道如何从PXE服务器上下载某个文件，去初始化操作系统。</p><h2>解析PXE的工作过程</h2><p>接下来我们来详细看一下PXE的工作过程。</p><p>首先，启动PXE客户端。第一步是通过DHCP协议告诉DHCP Server，我刚来，一穷二白，啥都没有。DHCP Server便租给它一个IP地址，同时也给它PXE服务器的地址、启动文件pxelinux.0。</p><p>其次，PXE客户端知道要去PXE服务器下载这个文件后，就可以初始化机器。于是便开始下载，下载的时候使用的是TFTP协议。所以PXE服务器上，往往还需要有一个TFTP服务器。PXE客户端向TFTP服务器请求下载这个文件，TFTP服务器说好啊，于是就将这个文件传给它。</p><p>然后，PXE客户端收到这个文件后，就开始执行这个文件。这个文件会指示PXE客户端，向TFTP服务器请求计算机的配置信息pxelinux.cfg。TFTP服务器会给PXE客户端一个配置文件，里面会说内核在哪里、initramfs在哪里。PXE客户端会请求这些文件。</p><p>最后，启动Linux内核。一旦启动了操作系统，以后就啥都好办了。</p><p><img src="https://static001.geekbang.org/resource/image/bb/8e/bbc2b660bba0ad00b5d1179db158498e.jpg?wh=2083*3001" alt=""></p><h2>小结</h2><p>好了，这一节就到这里了。我来总结一下今天的内容：</p><ul>
<li>
<p>DHCP协议主要是用来给客户租用IP地址，和房产中介很像，要商谈、签约、续租，广播还不能“抢单”；</p>
</li>
<li>
<p>DHCP协议能给客户推荐“装修队”PXE，能够安装操作系统，这个在云计算领域大有用处。</p>
</li>
</ul><p>最后，学完了这一节，给你留两个思考题吧。</p><ol>
<li>PXE协议可以用来安装操作系统，但是如果每次重启都安装操作系统，就会很麻烦。你知道如何使得第一次安装操作系统，后面就正常启动吗？</li>
<li>现在上网很简单了，买个家用路由器，连上WIFI，给DHCP分配一个IP地址，就可以上网了。那你是否用过更原始的方法自己组过简单的网呢？说来听听。</li>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/53/19/965c845c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>袁沛</span>
  </div>
  <div class="_2_QraFYR_0">20年前大学宿舍里绕了好多同轴电缆的10M以太网，上BBS用IP，玩星际争霸用IPX。那时候没有DHCP，每栋楼有个哥们负责分配IP。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-05-25 07:29:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83epmicgRSX8C0bABLic1MA6ssqOtqAu0vqkWgshsAfOhWXDnmmkfVicPBlJs6j6DRVuZx61Ldia7617YXg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>xcodeproj</span>
  </div>
  <div class="_2_QraFYR_0">新机器进来申请分配ip地址，老师你说在dhcp request的时候这台机器以0.0.0.0为源地址发出请求，那如果有多台机器同时申请呢？DHCP server如何分辨啊</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-05-25 21:30:31</div>
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
  <div class="_2_QraFYR_0">以问答写笔记：<br><br>1. 正确配置IP?<br>	<br>CIDR、子网掩码、广播地址和网关地址。<br>	<br>2. 在跨网段调用中，是如何获取目标IP的mac地址的？<br>	<br>从源IP网关获取所在网关mac,<br>然后又替换为目标IP所在网段网关的mac,<br>最后是目标IP的mac地址<br>	<br>3. 手动配置麻烦，怎么办？<br>	<br>DHCP！Dynamic Host Configuration Protocol！<br>DHCP, 让你配置IP，如同自动房产中介。<br>	<br>4. 如果新来的，房子是空的(没有操作系统)，怎么办？<br>	<br>PXE， Pre-boot Execution Environment.<br>&quot;装修队&quot;PXE，帮你安装操作系统。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-07-07 06:21:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/36/9e/8f031100.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ERIC</span>
  </div>
  <div class="_2_QraFYR_0">刘老师你好，文章关于DHCP可能是有两处错误。DHCP Offer 和 DHCP ACK都不是广播包，而是直接发到客户机的网卡上的。这是wiki上的链接：<br>https:&#47;&#47;en.wikipedia.org&#47;wiki&#47;Dynamic_Host_Configuration_Protocol#DHCP_offer<br>https:&#47;&#47;en.wikipedia.org&#47;wiki&#47;Dynamic_Host_Configuration_Protocol#DHCP_acknowledgement<br><br>另外我自己也抓了包验证，https:&#47;&#47;baixiang.oss-cn-shenzhen.aliyuncs.com&#47;dhcp&#47;dhcp.png。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个在答疑环节讲过啦</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-01 11:28:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/67/95/5bd8911f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>kevinwang527</span>
  </div>
  <div class="_2_QraFYR_0">现在一般电脑的网卡几乎都支持PXE启动， PXE client 就在网卡的 ROM 中，当计算机引导时，BIOS 把 PXE client 调入内存执行。<br>安装完成后，将提示重新引导计算机。这个时候，在重新引导的过程中将BIOS修改回从硬盘启动就可以了。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-28 15:43:06</div>
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
  <div class="_2_QraFYR_0">请问在 Offer 和 ACK 阶段，为什么 DHCP Server 给新机器的数据包中，MAC 头里用的是广播地址（ff:ff:ff:ff:ff:ff）而不是新机器的 MAC 地址？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-16 05:02:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/FMwyx76xm95LgNQKtepBbNVMz011ibAjM42N2PicvqU9tib9n43AURiaq6CKCqEoGo9iahsNNsTSiaqANMmfCbK0kZhQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>机器人</span>
  </div>
  <div class="_2_QraFYR_0">那么跨网段调用中，是如何获取目标IP 的mac地址的？根据讲解推理应该是从源IP网关获取所在网关<br>mac,然后又替换为目标IP所在网段网关的mac,最后是目标IP的mac地址，不知对否</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-05-25 08:18:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/97/39/60d6a10d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>天涯囧侠</span>
  </div>
  <div class="_2_QraFYR_0">在一个有dhcp的网络里，如果我手动配置了一个IP，dhcp Server会知道这个信息，并不再分配这个IP吗？会的话具体是怎样交互的呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 有可能冲突的，所以办公网里面一般禁止配置静态ip</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-05-25 10:27:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/d0/f1/432e0476.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>X</span>
  </div>
  <div class="_2_QraFYR_0">进入BIOS设置页面，有一项PXE Boot to LAN，若设置为Enabled则表示计算机从网络启动，从PXE服务端下载配置文件和操作系统内核进行启动；若设置为Disabled则表示从本地启动，启动动BIOS后，会去寻找启动扇区，如果没有安装操作系统，就会找不到启动扇区，这个时候就启动不起来。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，还有一种服务端的配置</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-05-27 19:00:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/7d/95/dd73022c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我是曾经那个少年</span>
  </div>
  <div class="_2_QraFYR_0">看了虽然懂了，但是对于一个做软件开发的，不知道怎么去实战！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 最后会有一个实验管理的搭建，一台机器足以</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-05-25 16:51:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/f8/ba/14e05601.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>约书亚</span>
  </div>
  <div class="_2_QraFYR_0">跨网段的通信，一般都是ip包头的目标地址是最终目标地址，但2层包头的目标地址总是下一个网关的，是么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-05-25 08:10:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/4c/fd/8022b3a2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>penghuster</span>
  </div>
  <div class="_2_QraFYR_0">请教一下，pxe客户端请求的IP，是否最终会直接用于系统</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不会的，系统起来后配置ip是他自己的事情</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-01 16:08:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/54/cf/fddcf843.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>芋头</span>
  </div>
  <div class="_2_QraFYR_0">要是以前大学老师能够讲得如此精彩，易懂，大学就不会白学了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-01 08:25:55</div>
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
  <div class="_2_QraFYR_0">DHCP Request究竟使用广播还是单播取决于DHCP Offer包中Broadcast位的设置值。该位置1则使用广播发送，置0则使用单播发送。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-05-25 15:34:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/f8/bd/16545faf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>陈浩佳</span>
  </div>
  <div class="_2_QraFYR_0">我分享一个我最近遇到的问题:<br>最近我们的设备增加了dhcp自动分配地址的功能。我把几台设备连到同个路由器上，但是发现每台设备最后分配到的ip都是一样的，我登录了路由器里面查看，显示的设备列表确实是ip都是一致的，mac地址是不一致的。。。。所以就觉得有点奇怪。不过这里要说明的是，设备的mac地址是我们自己程序里面设置的，网卡不带mac地址的----最后查看代码发现，我们设备代码是先启动了dhcp客户端，后面再设置了mac地址，这里就有问题了，所以，我把它倒过来，先设置mac地址，再启动dhcp客户端，这样就解决问题了。。。由于原先启动dhcp的时候还未设置mac地址，所以默认的mac地址都是一致的，所以获取的ip都是一致的。但是，这里也说明一个问题，路由器列表上的mac地址不一定就是分配ip时的mac地址，如果分配到ip后再去修改mac地址，也是会同步到路由器上的，但是不会重新分配ip。<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞，活学活用</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-07 13:15:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/3f/97/8d7a6460.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>卡卡</span>
  </div>
  <div class="_2_QraFYR_0">pxe要去tftp下载初始文件，那么pxe自己是不是也需要一个tftp客户端？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，不过tftp很轻量</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-05-25 19:51:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>呵呵</span>
  </div>
  <div class="_2_QraFYR_0">pxe客户端是放在哪里的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: bios</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-05-25 08:52:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/14/b5/d5cb9fad.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>zhy</span>
  </div>
  <div class="_2_QraFYR_0">之前一直不清楚dhcp是干嘛的→_→终于明白了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-05-25 08:56:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/84/63/062d0f28.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Tristen陈涛</span>
  </div>
  <div class="_2_QraFYR_0">文章开头，源 IP 地址 16.158.23.6，目标 IP 地址 192.168.1.6 发包的问题，还是不太懂<br>此种情况下的结果是： 网关收到 16.158.23.6 包后直接拒绝了还是有别的处理？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-05-29 22:32:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/4f/94/05044c31.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>踢车牛</span>
  </div>
  <div class="_2_QraFYR_0">老师，你好，上面你提到网关至少和当前一个网络的一个网卡在同一个网段内，这么说网关上可以配置多个网卡，<br>假如网关上有两个网卡，其中一个是192.168.1.6，另一个是<br>16.158.23.X,这样包可以发出去么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以的，网关上有路由表</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-05 09:13:53</div>
  </div>
</div>
</div>
</li>
</ul>