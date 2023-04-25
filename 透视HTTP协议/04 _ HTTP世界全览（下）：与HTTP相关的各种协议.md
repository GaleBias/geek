<audio title="04 _ HTTP世界全览（下）：与HTTP相关的各种协议" src="https://static001.geekbang.org/resource/audio/99/51/995cf58219a4e99589c7d58ffa66a451.mp3" controls="controls"></audio> 
<p>在上一讲中，我介绍了与HTTP相关的浏览器、服务器、CDN、网络爬虫等应用技术。</p><p>今天要讲的则是比较偏向于理论的各种HTTP相关协议，重点是TCP/IP、DNS、URI、HTTPS等，希望能够帮你理清楚它们与HTTP的关系。</p><p>同样的，我还是画了一张详细的思维导图，你可以点击后仔细查看。</p><p><img src="https://static001.geekbang.org/resource/image/1e/81/1e7533f765d2ede0abfab73cf6b57781.png?wh=1863*2271" alt=""></p><h2>TCP/IP</h2><p>TCP/IP协议是目前网络世界“事实上”的标准通信协议，即使你没有用过也一定听说过，因为它太著名了。</p><p>TCP/IP协议实际上是一系列网络通信协议的统称，其中最核心的两个协议是<strong>TCP</strong>和<strong>IP</strong>，其他的还有UDP、ICMP、ARP等等，共同构成了一个复杂但有层次的协议栈。</p><p>这个协议栈有四层，最上层是“应用层”，最下层是“链接层”，TCP和IP则在中间：<strong>TCP属于“传输层”，IP属于“网际层”</strong>。协议的层级关系模型非常重要，我会在下一讲中再专门讲解，这里先暂时放一放。</p><p><strong>IP协议</strong>是“<strong><span class="orange">I</span></strong>nternet <strong><span class="orange">P</span></strong>rotocol”的缩写，主要目的是解决寻址和路由问题，以及如何在两点间传送数据包。IP协议使用“<strong>IP地址</strong>”的概念来定位互联网上的每一台计算机。可以对比一下现实中的电话系统，你拿着的手机相当于互联网上的计算机，而要打电话就必须接入电话网，由通信公司给你分配一个号码，这个号码就相当于IP地址。</p><!-- [[[read_end]]] --><p>现在我们使用的IP协议大多数是v4版，地址是四个用“.”分隔的数字，例如“192.168.0.1”，总共有2^32，大约42亿个可以分配的地址。看上去好像很多，但互联网的快速发展让地址的分配管理很快就“捉襟见肘”。所以，就又出现了v6版，使用8组“:”分隔的数字作为地址，容量扩大了很多，有2^128个，在未来的几十年里应该是足够用了。</p><p><strong>TCP协议</strong>是“<strong><span class="orange">T</span></strong>ransmission <strong><span class="orange">C</span></strong>ontrol <strong><span class="orange">P</span></strong>rotocol”的缩写，意思是“传输控制协议”，它位于IP协议之上，基于IP协议提供<span class="orange">可靠的、字节流</span>形式的通信，是HTTP协议得以实现的基础。</p><p>“可靠”是指保证数据不丢失，“字节流”是指保证数据完整，所以在TCP协议的两端可以如同操作文件一样访问传输的数据，就像是读写在一个密闭的管道里“流动”的字节。</p><p>在<a href="https://time.geekbang.org/column/article/98128">第2讲</a>时我曾经说过，HTTP是一个"传输协议"，但它不关心寻址、路由、数据完整性等传输细节，而要求这些工作都由下层来处理。因为互联网上最流行的是TCP/IP协议，而它刚好满足HTTP的要求，所以互联网上的HTTP协议就运行在了TCP/IP上，HTTP也就可以更准确地称为“<strong>HTTP over TCP/IP</strong>”。</p><h2>DNS</h2><p>在TCP/IP协议中使用IP地址来标识计算机，数字形式的地址对于计算机来说是方便了，但对于人类来说却既难以记忆又难以输入。</p><p>于是“<strong>域名系统</strong>”（<strong><span class="orange">D</span></strong>omain <strong><span class="orange">N</span></strong>ame <strong><span class="orange">S</span></strong>ystem）出现了，用有意义的名字来作为IP地址的等价替代。设想一下，你是愿意记“95.211.80.227”这样枯燥的数字，还是“nginx.org”这样的词组呢？</p><p>在DNS中，“域名”（Domain Name）又称为“主机名”（Host），为了更好地标记不同国家或组织的主机，让名字更好记，所以被设计成了一个有层次的结构。</p><p>域名用“.”分隔成多个单词，级别从左到右逐级升高，最右边的被称为“顶级域名”。对于顶级域名，可能你随口就能说出几个，例如表示商业公司的“com”、表示教育机构的“edu”，表示国家的“cn”“uk”等，买火车票时的域名还记得吗？是“www.12306.cn”。</p><p><img src="https://static001.geekbang.org/resource/image/36/b3/36b6a41da6e9abc2fc28ee9a305f48b3.jpg?wh=1142*155" alt="unpreview"></p><p>但想要使用TCP/IP协议来通信仍然要使用IP地址，所以需要把域名做一个转换，“映射”到它的真实IP，这就是所谓的“<strong>域名解析</strong>”。</p><p>继续用刚才的打电话做个比喻，你想要打电话给小明，但不知道电话号码，就得在手机里的号码簿里一项一项地找，直到找到小明那一条记录，然后才能查到号码。这里的“小明”就相当于域名，而“电话号码”就相当于IP地址，这个查找的过程就是域名解析。</p><p>域名解析的实际操作要比刚才的例子复杂很多，因为互联网上的电脑实在是太多了。目前全世界有13组根DNS服务器，下面再有许多的顶级DNS、权威DNS和更小的本地DNS，逐层递归地实现域名查询。</p><p>HTTP协议中并没有明确要求必须使用DNS，但实际上为了方便访问互联网上的Web服务器，通常都会使用DNS来定位或标记主机名，间接地把DNS与HTTP绑在了一起。</p><h2>URI/URL</h2><p>有了TCP/IP和DNS，是不是我们就可以任意访问网络上的资源了呢？</p><p>还不行，DNS和IP地址只是标记了互联网上的主机，但主机上有那么多文本、图片、页面，到底要找哪一个呢？就像小明管理了一大堆文档，你怎么告诉他是哪个呢？</p><p>所以就出现了URI（<strong><span class="orange">U</span></strong>niform <strong><span class="orange">R</span></strong>esource <strong><span class="orange">I</span></strong>dentifier），中文名称是 <strong>统一资源标识符</strong>，使用它就能够唯一地标记互联网上资源。</p><p>URI另一个更常用的表现形式是URL（<strong><span class="orange">U</span></strong>niform <strong><span class="orange">R</span></strong>esource <strong><span class="orange">L</span></strong>ocator）， <strong>统一资源定位符</strong>，也就是我们俗称的“网址”，它实际上是URI的一个子集，不过因为这两者几乎是相同的，差异不大，所以通常不会做严格的区分。</p><p>我就拿Nginx网站来举例，看一下URI是什么样子的。</p><pre><code>http://nginx.org/en/download.html
</code></pre><p>你可以看到，URI主要有三个基本的部分构成：</p><ol>
<li>协议名：即访问该资源应当使用的协议，在这里是“http”；</li>
<li>主机名：即互联网上主机的标记，可以是域名或IP地址，在这里是“nginx.org”；</li>
<li>路径：即资源在主机上的位置，使用“/”分隔多级目录，在这里是“/en/download.html”。</li>
</ol><p>还是用打电话来做比喻，你通过电话簿找到了小明，让他把昨天做好的宣传文案快递过来。那么这个过程中你就完成了一次URI资源访问，“小明”就是“主机名”，“昨天做好的宣传文案”就是“路径”，而“快递”，就是你要访问这个资源的“协议名”。</p><h2>HTTPS</h2><p>在TCP/IP、DNS和URI的“加持”之下，HTTP协议终于可以自由地穿梭在互联网世界里，顺利地访问任意的网页了，真的是“好生快活”。</p><p>但且慢，互联网上不仅有“美女”，还有很多的“野兽”。</p><p>假设你打电话找小明要一份广告创意，很不幸，电话被商业间谍给窃听了，他立刻动用种种手段偷窃了你的快递，就在你还在等包裹的时候，他抢先发布了这份广告，给你的公司造成了无形或有形的损失。</p><p>有没有什么办法能够防止这种情况的发生呢？确实有。你可以使用“加密”的方法，比如这样打电话：</p><blockquote>
<p>你：“喂，小明啊，接下来我们改用火星文通话吧。”<br>
小明：“好啊好啊，就用火星文吧。”<br>
你：“巴拉巴拉巴拉巴拉……”<br>
小明：“巴拉巴拉巴拉巴拉……”</p>
</blockquote><p>如果你和小明说的火星文只有你们两个才懂，那么即使窃听到了这段谈话，他也不会知道你们到底在说什么，也就无从破坏你们的通话过程。</p><p>HTTPS就相当于这个比喻中的“火星文”，它的全称是“<strong>HTTP over SSL/TLS</strong>”，也就是运行在SSL/TLS协议上的HTTP。</p><p>注意它的名字，这里是SSL/TLS，而不是TCP/IP，它是一个负责加密通信的安全协议，建立在TCP/IP之上，所以也是个可靠的传输协议，可以被用作HTTP的下层。</p><p>因为HTTPS相当于“HTTP+SSL/TLS+TCP/IP”，其中的“HTTP”和“TCP/IP”我们都已经明白了，只要再了解一下SSL/TLS，HTTPS也就能够轻松掌握。</p><p>SSL的全称是“<strong><span class="orange">S</span></strong>ecure <strong><span class="orange">S</span></strong>ocket <strong><span class="orange">L</span></strong>ayer”，由网景公司发明，当发展到3.0时被标准化，改名为TLS，即“<strong>T</strong>ransport <strong>L</strong>ayer <strong>S</strong>ecurity”，但由于历史的原因还是有很多人称之为SSL/TLS，或者直接简称为SSL。</p><p>SSL使用了许多密码学最先进的研究成果，综合了对称加密、非对称加密、摘要算法、数字签名、数字证书等技术，能够在不安全的环境中为通信的双方创建出一个秘密的、安全的传输通道，为HTTP套上一副坚固的盔甲。</p><p>你可以在今后上网时留心看一下浏览器地址栏，如果有一个小锁头标志，那就表明网站启用了安全的HTTPS协议，而URI里的协议名，也从“http”变成了“https”。</p><h2>代理</h2><p>代理（Proxy）是HTTP协议中请求方和应答方中间的一个环节，作为“中转站”，既可以转发客户端的请求，也可以转发服务器的应答。</p><p>代理有很多的种类，常见的有：</p><ol>
<li>匿名代理：完全“隐匿”了被代理的机器，外界看到的只是代理服务器；</li>
<li>透明代理：顾名思义，它在传输过程中是“透明开放”的，外界既知道代理，也知道客户端；</li>
<li>正向代理：靠近客户端，代表客户端向服务器发送请求；</li>
<li>反向代理：靠近服务器端，代表服务器响应客户端的请求；</li>
</ol><p>上一讲提到的CDN，实际上就是一种代理，它代替源站服务器响应客户端的请求，通常扮演着透明代理和反向代理的角色。</p><p>由于代理在传输过程中插入了一个“中间层”，所以可以在这个环节做很多有意思的事情，比如：</p><ol>
<li>负载均衡：把访问请求均匀分散到多台机器，实现访问集群化；</li>
<li>内容缓存：暂存上下行的数据，减轻后端的压力；</li>
<li>安全防护：隐匿IP,使用WAF等工具抵御网络攻击，保护被代理的机器；</li>
<li>数据处理：提供压缩、加密等额外的功能。</li>
</ol><p>关于HTTP的代理还有一个特殊的“代理协议”（proxy protocol），它由知名的代理软件HAProxy制订，但并不是RFC标准，我也会在之后的课程里专门讲解。</p><h2>小结</h2><p>这次我介绍了与HTTP相关的各种协议，在这里简单小结一下今天的内容。</p><ol>
<li><span class="orange">TCP/IP是网络世界最常用的协议，HTTP通常运行在TCP/IP提供的可靠传输基础上；</span></li>
<li><span class="orange">DNS域名是IP地址的等价替代，需要用域名解析实现到IP地址的映射；</span></li>
<li><span class="orange">URI是用来标记互联网上资源的一个名字，由“协议名+主机名+路径”构成，俗称URL；</span></li>
<li><span class="orange">HTTPS相当于“HTTP+SSL/TLS+TCP/IP”，为HTTP套了一个安全的外壳；</span></li>
<li><span class="orange">代理是HTTP传输过程中的“中转站”，可以实现缓存加速、负载均衡等功能。</span></li>
</ol><p>经过这两讲的学习，相信你应该对HTTP有了一个比较全面的了解，虽然还不是很深入，但已经为后续的学习扫清了障碍。</p><h2>课下作业</h2><ol>
<li>DNS与URI有什么关系？</li>
<li>在讲<strong>代理</strong>时我特意没有举例说明，你能够用引入一个“小强”的角色，通过打电话来比喻一下吗？</li>
</ol><p>欢迎你通过留言分享答案，与我和其他同学一起讨论。如果你觉得有所收获，欢迎你把文章分享给你的朋友。</p><p><img src="https://static001.geekbang.org/resource/image/4e/56/4eab55dc3600071330e088b40cae4856.png?wh=1769*2306" alt="unpreview"></p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/ab/e1/f6b921fa.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>壹笙☞漂泊</span>
  </div>
  <div class="_2_QraFYR_0">课后题：<br>1、URI DNS<br>DNS 是将域名解析出真实IP地址的系统<br>URI 是统一资源标识符，标定了客户端需要访问的资源所处的位置，如果URI中的主机名使用域名，则需要使用DNS来讲域名解析为IP。<br>2、打电话给小明，请小明找小王拿一下客户资料。小明处于代理角色。<br>内容笔记<br>1、四层模型：应用层、传输层、网际层、链接层<br>2、IP协议主要解决寻址和路由问题<br>3、ipv4，地址是四个用“.”分隔的数字，总数有2^32个，大约42亿个可以分配的地址<br>4、ipv6，地址是八个用“:”分隔的数字，总数有2^128个。<br>5、TCP协议位于IP协议之上，基于IP协议提供可靠的(数据不丢失)、字节流(数据完整)形式的通信，是HTTP协议得以实现的基础<br>6、域名系统：为了更好的标记不同国家或组织的主机，域名被设计成了一个有层次的结构<br>7、域名用“.”分隔成多个单词，级别从左到右逐级升高。<br>8、域名解析：将域名做一个转换，映射到它的真实IP<br>9、URI：统一资源标识符；URL：统一资源定位符<br>10、URI主要有三个基本部分构成：协议名、主机名、路径<br>11、HTTPS：运行在SSL&#47;TLS协议上的HTTP<br>12 、SSL&#47;TLS：建立在TCP&#47;IP之上的负责加密通信的安全协议，是可靠的传输协议，可以被用作HTTP的下层<br>13、代理(Proxy)：是HTTP协议中请求方和应答方中间的一个环节。既可以转发客户端的请求，也可以转发服务器的应答。<br>14、代理常见种类：匿名台历、透明代理、正向代理、反向代理<br>15、代理可以做的事：负载均衡、内容缓存、安全防护、数据处理。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 总结的非常详细，也很准确，鼓掌！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-05 10:00:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/67/f4/9a1feb59.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>钱</span>
  </div>
  <div class="_2_QraFYR_0">1：DNS 与 URI 有什么关系？<br>       DNS专门用于域名解析，作用是简化人类记忆数据的复杂度。<br>        URI专门用于标识互联网世界中的资源，作用是帮助找到对应的互联网中资源。<br>         互联网中的电脑通过IP地址来表示，DNS可以把一个域名变成一个IP地址，IP地址是标识资源的一部分，仅定位了具体的电脑，还有继续定位在电脑上的具体位置。<br><br>2：在讲代理时我特意没有举例说明，你能够用引入一个“小强”的角色，通过打电话来比喻一下吗？<br>小强给小明打电话要小红的照片——小明是正向代理<br>小强要小红的照片小明负责处理——小明是反向代理<br><br>网络通信是分布式系统的底座，也是信息交互的法宝<br>TCP——负责数据传输<br>IP——负责标识传输对象<br>DNS——负责简化人类的记忆<br>URI&#47;L——负责标识传输的资源<br>SSL——负责数据传输的安全<br>Proxy——负责信息的中转<br>像极了走标，<br>需要搞清楚从哪到哪——IP<br>需要搞定怎么传输——TCP<br>需要保障货物的安全——SSL<br>需要送货的具体位置——URI<br>需要把目的地的经纬度换成地址名——DNS<br>需要中间中转一下——Proxy<br>HTTP——我不那么多，我向你要什么你就给什么</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: amazing！！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-24 07:34:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTIjTlXUOibjFdIicwDas890lM4Jrsdag4c0t0XLBtopQRvwG8Vj4agbtpPkKGNPH96bDWyibmMiceEibaA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Atomic</span>
  </div>
  <div class="_2_QraFYR_0">打个比方：我让老婆帮我去楼下超市买瓶水，DNS可以帮她找到楼下超市，URI可以帮她找到水放在超市的具体位置</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 比喻的好生动，笑。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-06 07:47:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/5d/e2/3331ad9e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Shine Sunner</span>
  </div>
  <div class="_2_QraFYR_0">1.假如去某个小区找人，DNS可以帮我定位到是哪栋大楼，URI可以帮我定位到是哪个房间。<br>2.<br>正向代理:<br>   假如我【客户端】想找小强【服务端】借钱，但是我不好意思。我去找小李【代理】，然后让小李找小强借。对于小强来说他以为是小李找他借钱，而不是我。<br>   <br>反向代理:<br>  同样是借钱，这回我【客户端】找小李【代理】借钱，小李没钱了，他去找小强【服务端】借钱，然后再把钱借给我，对我来说我认为是小李借钱给我，而不是小强。<br><br>  总结:<br>   正向代理的代理服务器是部署在客户端，而对服务端来说，它以为对它发起请求的是代理服务器，而真正请求的客户端对服务端来说是不可见的。<br>   反向代理的代理服务器是部署在服务端，而对客户端来说，它以为对它做出响应的是代理服务器，而真正响应的服务端对客户端来说是不可见的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 说的非常好，给你点32个赞（笑）。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-01 11:32:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/27/d5/0fd21753.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>一粟</span>
  </div>
  <div class="_2_QraFYR_0">小强家钥匙丢了，需要找一家开锁公司开门。于是小强打电话给114，114给小强提供一家有资质的开锁公司，并将电话转接过去。这里的114就是代理。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: √</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-05 06:57:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/07/8c/0d886dcc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>蚂蚁内推+v</span>
  </div>
  <div class="_2_QraFYR_0">1. URL 包含了协议+主机名+路径，DNS 会将其中的主机名解析为 IP，进而方便根据 IP 协议进行寻址、路由；<br>2. 我们为了更安全的和小明交流，选择通过和小强交流，让其再告诉小明，这是匿名代理，也是正向代理，而如果让小明知道我们的存在则不是匿名代理，是透明代理；小明由于某些原因不能直接响应我们，找了小强来代为响应我们，这是反向代理；<br>3. 另外回答一下楼下同学关于 URI 和 URL 区别的疑惑，URI 是 Identifier，即标识符，URL 是 Location，即定位，所以定位只是标识符的一种，打个比方，我们找到小明可以通过其家庭地址（Location）也可以通过名字（比如上课点名）来找到他，所以后者也可以成为 URN。因此 URL 和 URN 都是 URI 的子集。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 说的很好，不过现在urn用的很少，现在的uri基本上就是url，除非写论文，否则不用特意区分。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-06 09:05:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/52/56/5abb4d94.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>不知道该叫什么</span>
  </div>
  <div class="_2_QraFYR_0">但是我还是没明白URI跟RUL的区别</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: url是uri的子集，url只表示网址，而uri除了表示网址，还能够标记其他的任意东西。<br><br>但在互联网上，这两者是基本等价的，也不需要去钻字眼刻意区分。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-28 10:24:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/90/37/39dcd581.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小葱🤓</span>
  </div>
  <div class="_2_QraFYR_0">别的不想说，请问能调高课程的费用吗？？？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-12 18:00:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/57/4f/6fb51ff1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>一步</span>
  </div>
  <div class="_2_QraFYR_0">Http协议不是依赖tcp&#47;ip的拆包和封包吗？Unix domain socket可以做到吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 当然可以，如果在Linux上跑Nginx，就可以指定用Unix domain socket。<br><br>关键要理解协议栈，http不强制要求下层必须是tcp。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-05 09:41:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/27/d3/0a/92640aae.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我爱夜来香</span>
  </div>
  <div class="_2_QraFYR_0">老师，我有个问题，就是URL由三部分组成，前面的协议名和主机名能理解,后面的路径指的是应用在服务器上的真实路径吗？或者说是由真实路径经过一层封装而形成的?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 注意，uri表示的是网络上的资源，这实际上是一种抽象，意思是在主机上的某个位置有一个资源。<br><br>但这个资源路径不一定会与主机磁盘上的路径完全匹配，可以相同也可以不相同，通常来说会有一个简单的转换，比如映射到不同的目录。<br><br>而且，图片、html等静态资源是可以对应到文件系统的，而动态资源，它根本就没有实体，所以uri就完全是一个标识符的作用，不存在路径。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-02 16:54:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/57/4f/6fb51ff1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>一步</span>
  </div>
  <div class="_2_QraFYR_0">看到老师后面小帖士说的，在unix系统上http可以依赖一种进程间传输的机制Unix domain socket进行传输，这是因为满足了底层的可靠的传输。这句话意思是说，http不一定在tcp&#47;ip之上进行传输？只要底层满足可靠传输的都可以？<br><br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 当然了，这就是http灵活性的体现。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-05 09:04:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/cd/94/0d44361e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Jerryz</span>
  </div>
  <div class="_2_QraFYR_0">1. DNS 可以定位到一台主机，URI 则可以定位到主机上的资源。<br>2. 正向代理：我通过查找小强的电话薄，转话到小明。反向代理：我通过查找小明的电话薄，但是和我通话的实际是小强。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: very nice!</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-12 08:56:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/07/97/980d36e2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Tio Kang</span>
  </div>
  <div class="_2_QraFYR_0">老师，我有一个疑问，一个代理即可以是反向代理也可以正向代理吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 理论上应该是可以的，但实际上应该没有这么用。<br><br>因为正向代理连接的是上网的客户端，反向代理连接的是网站的服务器，代理的对象是不同的，合不到一起。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-02 20:16:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>dingdongfm</span>
  </div>
  <div class="_2_QraFYR_0">不做本地证书校验时，https可以被抓包工具抓到明文包，原因是什么？https不能防止中间人攻击么</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 开头的握手数据是明文的，但后面都是加密的，与证书验证无关。<br><br>https中间人攻击必须要客户端信任根证书，像fiddler就是这么做的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-19 07:59:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/cb/9d/2bc85843.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>　　　　　　　鸟人</span>
  </div>
  <div class="_2_QraFYR_0">a要向b发送消息，实际是先发到代理，由代理发给b。反向由b返回给代理，代理返回给a。<br>那么我向cdn发送评论  此时为正向，然后刷新页面  看到自己写的评论  此时为反向 <br>可以这样理解么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 好像不太正确，每一次的http消息都是一个往返，请求先到服务器，然后服务器发回响应。<br><br>正向代理是指“正”着代理客户端，反向代理是指“逆”着请求的方向代理服务器。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-11 09:32:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/36/d2/c7357723.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>发条橙子 。</span>
  </div>
  <div class="_2_QraFYR_0">老师，我这里有个疑问 。 一个请求由DNS解析到指定的IP ，然后通过URI确定要访问哪些资源。最后通过 TCP&#47;IP 进行路由寻址以及数据的传输。<br>但是一台机子上有多个应用 ， 可能两个相同的应用运行在同一个主机上 ，有着两个不同的进程。 那么根据URI是指定从哪个进程里获取数据呢 。 <br>这时候是不是根据端口号来判定 ， 但是URI上并没有显式的让我们看出是哪个端口号 ？？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: uri会有默认端口号，比如http默认是80，用tcp连接必须要同时指定ip地址和端口。<br><br>服务器进程在指定端口上监听，然后tcp就可以建立连接。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-09 10:51:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/gPNQV6n5ibib3qaWEuiaUY77TpxM4dkvr44PA3xJHc14AZbdl0kvRQhmpwpaQ4I0qZobtZlYbY5ZXuVrGfWyuk3JQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小伙儿爱裸睡</span>
  </div>
  <div class="_2_QraFYR_0">老师，TCP协议作用中的数据不丢失和数据完整有什么区别呢？可能我刚入门，有点抠字眼，还望老师不吝赐教哈。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 简单来说，丢失就不完整了。可以对比一下udp，udp不保证数据完整，会丢包，使用udp的应用需要自己处理丢包，保证数据完整，而使用tcp的应用就不需要考虑这些。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-07 21:58:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/b2/08/92f42622.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>尔冬橙</span>
  </div>
  <div class="_2_QraFYR_0">老师会后面会展开来讲么，比如域名解析过程，CDN调度过程等。现在面试官都问的太深了，如果只了解表面的概念很难以应对。希望老师能挖深一点</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 由于时间、篇幅的限制，讲不了特别深，我尽量吧。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-07 10:33:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/79/4b/740f91ca.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>-W.LI-</span>
  </div>
  <div class="_2_QraFYR_0">URI为了方便拥有记忆可以采用域名代替IP。<br>当用户使用域名访问时，就需要DNS技术找到对应的IP地址。然后找到对应的服务器或者代理。DNS域名解析发生在客户端。服务端接受到的还是用户输入的域名，或者IP。服务器(代理)可开启限制，只采用域名访问。<br>小刚替小明找小张，小刚就是正向代理。<br>小刚说我就是小张(私下问小张)。反向代理</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 说的很好。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-05 23:43:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/8f/9d/abb7bfe3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Carson</span>
  </div>
  <div class="_2_QraFYR_0">Dns负责解析uri中主机名为ip地址，这样才能使用ip协议来完成通信<br><br>在早起电话时代，小强给朋友打电话，要先拨通总机，让总机转接，总机就是代理。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 后一个不太准确，总机是中转的作用，和代理还是不太一样的，代理要能够代替另一方。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-05 19:47:55</div>
  </div>
</div>
</div>
</li>
</ul>