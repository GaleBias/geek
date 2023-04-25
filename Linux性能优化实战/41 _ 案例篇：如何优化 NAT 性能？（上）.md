<audio title="41 _ 案例篇：如何优化 NAT 性能？（上）" src="https://static001.geekbang.org/resource/audio/59/cb/59e35c709ed73a88e940b6ceec1a01cb.mp3" controls="controls"></audio> 
<p>你好，我是倪朋飞。</p><p>上一节，我们探究了网络延迟增大问题的分析方法，并通过一个案例，掌握了如何用 hping3、tcpdump、Wireshark、strace 等工具，来排查和定位问题的根源。</p><p>简单回顾一下，网络延迟是最核心的网络性能指标。由于网络传输、网络包处理等各种因素的影响，网络延迟不可避免。但过大的网络延迟，会直接影响用户的体验。</p><p>所以，在发现网络延迟增大的情况后，你可以先从路由、网络包的收发、网络包的处理，再到应用程序等，从各个层级分析网络延迟，等到找出网络延迟的来源层级后，再深入定位瓶颈所在。</p><p>今天，我再带你来看看，另一个可能导致网络延迟的因素，即网络地址转换（Network Address Translation），缩写为 NAT。</p><p>接下来，我们先来学习 NAT 的工作原理，并弄清楚如何优化 NAT 带来的潜在性能问题。</p><h2>NAT原理</h2><p>NAT 技术可以重写 IP 数据包的源 IP 或者目的 IP，被普遍地用来解决公网 IP 地址短缺的问题。它的主要原理就是，网络中的多台主机，通过共享同一个公网 IP 地址，来访问外网资源。同时，由于 NAT 屏蔽了内网网络，自然也就为局域网中的机器提供了安全隔离。</p><p>你既可以在支持网络地址转换的路由器（称为 NAT 网关）中配置 NAT，也可以在 Linux 服务器中配置 NAT。如果采用第二种方式，Linux 服务器实际上充当的是“软”路由器的角色。</p><!-- [[[read_end]]] --><p>NAT 的主要目的，是实现地址转换。根据实现方式的不同，NAT 可以分为三类：</p><ul>
<li>
<p>静态 NAT，即内网 IP 与公网 IP 是一对一的永久映射关系；</p>
</li>
<li>
<p>动态 NAT，即内网 IP 从公网 IP 池中，动态选择一个进行映射；</p>
</li>
<li>
<p>网络地址端口转换 NAPT（Network Address and Port Translation），即把内网 IP 映射到公网 IP 的不同端口上，让多个内网 IP 可以共享同一个公网 IP 地址。</p>
</li>
</ul><p>NAPT 是目前最流行的 NAT 类型，我们在 Linux 中配置的 NAT 也是这种类型。而根据转换方式的不同，我们又可以把 NAPT 分为三类。</p><p>第一类是源地址转换SNAT，即目的地址不变，只替换源 IP 或源端口。SNAT 主要用于，多个内网 IP 共享同一个公网 IP ，来访问外网资源的场景。</p><p>第二类是目的地址转换DNAT，即源 IP 保持不变，只替换目的 IP 或者目的端口。DNAT 主要通过公网 IP 的不同端口号，来访问内网的多种服务，同时会隐藏后端服务器的真实 IP 地址。</p><p>第三类是双向地址转换，即同时使用 SNAT 和 DNAT。当接收到网络包时，执行 DNAT，把目的 IP 转换为内网 IP；而在发送网络包时，执行 SNAT，把源 IP 替换为外部 IP。</p><p>双向地址转换，其实就是外网 IP 与内网 IP 的一对一映射关系，所以常用在虚拟化环境中，为虚拟机分配浮动的公网 IP 地址。</p><p>为了帮你理解 NAPT，我画了一张图。我们假设：</p><ul>
<li>
<p>本地服务器的内网 IP 地址为 192.168.0.2；</p>
</li>
<li>
<p>NAT 网关中的公网 IP 地址为 100.100.100.100；</p>
</li>
<li>
<p>要访问的目的服务器 baidu.com 的地址为 123.125.115.110。</p>
</li>
</ul><p>那么 SNAT 和 DNAT 的过程，就如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/c7/e4/c743105dc7bd955a4a300d6b55b7a0e4.png?wh=712*610" alt=""></p><p>从图中，你可以发现：</p><ul>
<li>
<p>当服务器访问 baidu.com 时，NAT 网关会把源地址，从服务器的内网 IP 192.168.0.2 替换成公网 IP 地址 100.100.100.100，然后才发送给 baidu.com；</p>
</li>
<li>
<p>当 baidu.com 发回响应包时，NAT 网关又会把目的地址，从公网 IP 地址 100.100.100.100 替换成服务器内网 IP 192.168.0.2，然后再发送给内网中的服务器。</p>
</li>
</ul><p>了解了 NAT 的原理后，我们再来看看，如何在 Linux 中实现 NAT 的功能。</p><h2>iptables与NAT</h2><p>Linux 内核提供的 Netfilter 框架，允许对网络数据包进行修改（比如 NAT）和过滤（比如防火墙）。在这个基础上，iptables、ip6tables、ebtables 等工具，又提供了更易用的命令行接口，以便系统管理员配置和管理 NAT、防火墙的规则。</p><p>其中，iptables 就是最常用的一种配置工具。要掌握 iptables 的原理和使用方法，最核心的就是弄清楚，网络数据包通过 Netfilter 时的工作流向，下面这张图就展示了这一过程。</p><p><img src="https://static001.geekbang.org/resource/image/c6/56/c6de40c5bd304132a1b508ba669e7b56.png?wh=1415*435" alt=""><br>
（图片来自 <a href="https://en.wikipedia.org/wiki/Iptables">Wikipedia</a>）</p><p>在这张图中，绿色背景的方框，表示表（table），用来管理链。Linux 支持 4 种表，包括 filter（用于过滤）、nat（用于NAT）、mangle（用于修改分组数据） 和 raw（用于原始数据包）等。</p><p>跟 table 一起的白色背景方框，则表示链（chain），用来管理具体的 iptables 规则。每个表中可以包含多条链，比如：</p><ul>
<li>
<p>filter 表中，内置 INPUT、OUTPUT 和 FORWARD 链；</p>
</li>
<li>
<p>nat 表中，内置PREROUTING、POSTROUTING、OUTPUT 等。</p>
</li>
</ul><p>当然，你也可以根据需要，创建你自己的链。</p><p>灰色的 conntrack，表示连接跟踪模块。它通过内核中的连接跟踪表（也就是哈希表），记录网络连接的状态，是 iptables 状态过滤（-m state）和 NAT 的实现基础。</p><p>iptables 的所有规则，就会放到这些表和链中，并按照图中顺序和规则的优先级顺序来执行。</p><p>针对今天的主题，要实现 NAT 功能，主要是在 nat 表进行操作。而 nat 表内置了三个链：</p><ul>
<li>
<p>PREROUTING，用于路由判断前所执行的规则，比如，对接收到的数据包进行 DNAT。</p>
</li>
<li>
<p>POSTROUTING，用于路由判断后所执行的规则，比如，对发送或转发的数据包进行 SNAT 或 MASQUERADE。</p>
</li>
<li>
<p>OUTPUT，类似于 PREROUTING，但只处理从本机发送出去的包。</p>
</li>
</ul><p>熟悉 iptables 中的表和链后，相应的 NAT 规则就比较简单了。我们还以 NAPT 的三个分类为例，来具体解读一下。</p><h3><strong>SNAT</strong></h3><p>根据刚才内容，我们知道，SNAT 需要在 nat 表的 POSTROUTING 链中配置。我们常用两种方式来配置它。</p><p>第一种方法，是为一个子网统一配置 SNAT，并由 Linux 选择默认的出口 IP。这实际上就是经常说的 MASQUERADE：</p><pre><code>$ iptables -t nat -A POSTROUTING -s 192.168.0.0/16 -j MASQUERADE
</code></pre><p>第二种方法，是为具体的 IP 地址配置 SNAT，并指定转换后的源地址：</p><pre><code>$ iptables -t nat -A POSTROUTING -s 192.168.0.2 -j SNAT --to-source 100.100.100.100
</code></pre><h3><strong>DNAT</strong></h3><p>再来看DNAT，显然，DNAT 需要在 nat 表的 PREROUTING 或者 OUTPUT 链中配置，其中， PREROUTING 链更常用一些（因为它还可以用于转发的包）。</p><pre><code>$ iptables -t nat -A PREROUTING -d 100.100.100.100 -j DNAT --to-destination 192.168.0.2
</code></pre><h3><strong>双向地址转换</strong></h3><p>双向地址转换，就是同时添加 SNAT 和 DNAT 规则，为公网 IP 和内网 IP 实现一对一的映射关系，即：</p><pre><code>$ iptables -t nat -A POSTROUTING -s 192.168.0.2 -j SNAT --to-source 100.100.100.100
$ iptables -t nat -A PREROUTING -d 100.100.100.100 -j DNAT --to-destination 192.168.0.2
</code></pre><p>在使用 iptables 配置 NAT 规则时，Linux 需要转发来自其他 IP 的网络包，所以你千万不要忘记开启 Linux 的 IP 转发功能。</p><p>你可以执行下面的命令，查看这一功能是否开启。如果输出的结果是 1，就表示已经开启了 IP 转发：</p><pre><code>$ sysctl net.ipv4.ip_forward
net.ipv4.ip_forward = 1
</code></pre><p>如果还没开启，你可以执行下面的命令，手动开启：</p><pre><code>$ sysctl -w net.ipv4.ip_forward=1
net.ipv4.ip_forward = 1
</code></pre><p>当然，为了避免重启后配置丢失，不要忘记将配置写入 /etc/sysctl.conf 文件中：</p><pre><code>$ cat /etc/sysctl.conf | grep ip_forward
net.ipv4.ip_forward=1
</code></pre><p>讲了这么多的原理，那当碰到 NAT 的性能问题时，又该怎么办呢？结合我们今天学过的 NAT 原理，你先自己想想，动手试试，下节课我们继续“分解”。</p><h2>小结</h2><p>今天，我们一起学习了 Linux 网络地址转换 NAT 的原理。</p><p>NAT 技术能够重写 IP 数据包的源 IP 或目的 IP，所以普遍用来解决公网 IP 地址短缺的问题。它可以让网络中的多台主机，通过共享同一个公网 IP 地址，来访问外网资源。同时，由于 NAT 屏蔽了内网网络，也为局域网中机器起到安全隔离的作用。</p><p>Linux 中的NAT ，基于内核的连接跟踪模块实现。所以，它维护每个连接状态的同时，也会带来很高的性能成本。具体 NAT 性能问题的分析方法，我们将在下节课继续学习。</p><h2>思考</h2><p>最后，给你留一个思考题。MASQUERADE 是最常用的一种 SNAT 规则，常用来为多个内网 IP 地址提供共享的出口 IP。</p><p>假设现在有一台 Linux 服务器，使用了 MASQUERADE 的方式，为内网的所有 IP 提供出口访问功能。那么，</p><ul>
<li>
<p>当多个内网 IP 地址的端口号相同时，MASQUERADE 还可以正常工作吗？</p>
</li>
<li>
<p>如果内网 IP 地址数量或请求数比较多，这种方式有没有什么隐患呢？</p>
</li>
</ul><p>欢迎在留言区和我讨论，也欢迎你把这篇文章分享给你的同事、朋友。我们一起在实战中演练，在交流中进步。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/8c/2b/3ab96998.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>青石</span>
  </div>
  <div class="_2_QraFYR_0">问题1：Linux的NAT时给予内核的连接跟踪模块实现，保留了源IP、源端口、目的IP、目的端口之间的关系，多个内网IP地址的端口相同，但是IP不同，再nf_conntrack中对应不同的记录，所以MASQUERADE可以正常工作。<br><br>问题2：NAT方式所有流量都要经过NAT服务器，所以NAT服务器本身的软中断导致CPU负载、网络流量、文件句柄、端口号上限、nf_conntrack table full都可能是性能瓶颈。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-19 22:11:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/64/05/6989dce6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我来也</span>
  </div>
  <div class="_2_QraFYR_0">[D41打卡]<br>在已有的项目经验中,还未涉及到过NAT.<br>倒是本地的虚拟机环境下,或者路由器上,会看到nat相关选项.<br><br>问题一:当多个内网 IP 地址的端口号相同时，MASQUERADE 还可以正常工作吗？<br>我觉得是可以正常工作的,要不然就不会允许设置ip地址段了.😁[纯属猜测哈]<br>在路由器上做端口映射时,一个外网端口只能对应一个内网的IP.<br>但是反方向,nat在转换源地址时,应该会记录原来的连接信息吧.要不然收到包该给谁发呢.<br><br>问题二:如果内网 IP 地址数量或请求数比较多，这种方式有没有什么隐患呢？<br>根据之前的经验,在请求数过多时,会导致CPU软中断上升.<br>再谷歌了下,有看到说:<br>iptables的conntrack表满了导致访问网站很慢.[https:&#47;&#47;my.oschina.net&#47;jean&#47;blog&#47;189935]<br>    ```kernel 用 ip_conntrack 模块来记录 iptables 网络包的状态，并保存到 table 里（这个 table 在内存里），如果网络状况繁忙，比如高连接，高并发连接等会导致逐步占用这个 table 可用空间。```<br><br>优化Linux NAT网关[https:&#47;&#47;tech.youzan.com&#47;linux_nat&#47;]<br>    ```net.netfilter.nfconntrackbuckets 这个参数，默认有点小，连接数多了以后，势必造成“哈希冲突”增加，“哈希处理”性能下降。（ 是这样吗？）```<br><br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍两个回答都是正确的，第二个还不完整，可以再考虑深入一点（提示：还有其他很多内核资源限制）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-25 10:44:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/f8/4b/5ae62b10.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_b04b12</span>
  </div>
  <div class="_2_QraFYR_0">1）iptalbes的三个表<br><br>filter 这个表主要用于过滤包的，是系统预设的表，这个表也是阿铭用的最多的。内建三个链INPUT、OUTPUT以及FORWARD。INPUT作用于进入本机的包；OUTPUT作用于本机送出的包；FORWARD作用于那些跟本机无关的包。<br><br>nat 主要用处是网络地址转换，也有三个链。PREROUTING 链的作用是在包刚刚到达防火墙时改变它的目的地址，如果需要的话。OUTPUT链改变本地产生的包的目的地址。POSTROUTING链在包就要离开防火墙之前改变其源地址。该表阿铭用的不多，但有时候会用到。<br><br>mangle 这个表主要是用于给数据包打标记，然后根据标记去操作哪些包。这个表几乎不怎么用。除非你想成为一个高级网络工程师，否则你就没有必要花费很多心思在它上面。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-06 00:02:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTL9hlAIKQ1sGDu16oWLOHyCSicr18XibygQSMLMjuDvKk73deDlH9aMphFsj41WYJh121aniaqBLiaMNg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>腾达</span>
  </div>
  <div class="_2_QraFYR_0">大伙儿都掉队了吗？有深度的问题留言越来越少，有价值的问题回答也少了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，有深度的留言还是前面多一些</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-25 21:19:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/f8/4b/5ae62b10.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_b04b12</span>
  </div>
  <div class="_2_QraFYR_0">http:&#47;&#47;www.apelearn.com&#47;study_v2&#47;chapter16.html#id3  这个链接是我初次接触linux的时候学习的课程，对小白蛮友好</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-06 00:03:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/xzHDjCSFicNY3MUMECtNz6sM8yDJhBoyGk5IRoOtUat6ZIkGzxjqEqwqKYWMD3GjehScKvMjicGOGDog5FF18oyg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>李逍遥</span>
  </div>
  <div class="_2_QraFYR_0">有个疑问， 看了访问baidu.com的例子，发包和收报都是需要NAT的，那是不是只配置SNAT或DNAT，就不能正常访问外网或被访问了呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不是的，内核有连接跟踪，知道每个请求的来源和目的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-26 18:22:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/aa/30/acc91f01.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>honnkyou</span>
  </div>
  <div class="_2_QraFYR_0">&quot;再来看 DNAT，显然，DNAT 需要在 nat 表的 PREROUTING 或者 OUTPUT 链中配置，其中， PREROUTING 链更常用一些（因为它还可以用于转发的包）。&quot;<br>DNAT不是转换到达包的目的地址吗？也可以在OUTPUT链中配置吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯嗯，可以的，OUTPUT用于从本机发出的包</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-18 21:11:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLzvaL724GwtzZ5mcldUnlicicSlI8BXL9icRZbUOB10qjRMlmog7UTvwxSBHXagnPGGR1BYdjWcGGSg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>wwj</span>
  </div>
  <div class="_2_QraFYR_0">nat的三种类型有什么本质的区别、和链接追踪的联系有是什么</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 主要是管理方式的区别，比如静态肯定比动态管理要简单的多；跟连接跟踪没有直接关系（只是Linux使用了连接跟踪）。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-28 12:43:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/f6/24/547439f1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ghostwritten</span>
  </div>
  <div class="_2_QraFYR_0">1. NAT 可以分为三类：<br>静态 NAT，即内网 IP 与公网 IP 是一对一的永久映射关系；<br>动态 NAT，即内网 IP 从公网 IP 池中，动态选择一个进行映射；<br>网络地址端口转换 NAPT（Network Address and Port Translation），即把内网 IP 映射到公网 IP 的不同端口上，让多个内网 IP 可以共享同一个公网 IP 地址。<br><br>2.  NAPT 分为三类：<br>第一类是源地址转换 SNAT，即目的地址不变，只替换源 IP 或源端口。SNAT 主要用于，多个内网 IP 共享同一个公网 IP来访问外网资源的场景。<br>第二类是目的地址转换 DNAT，即源 IP 保持不变，只替换目的 IP 或者目的端口。DNAT 主要通过公网 IP 的不同端口号，来访问内网的多种服务，同时会隐藏后端服务器的真实 IP 地址。<br>第三类是双向地址转换，即同时使用 SNAT 和 DNAT。当接收到网络包时，执行 DNAT，把目的 IP 转换为内网 IP；而在发送网络包时，执行 SNAT，把源 IP 替换为外部 IP。<br><br>3. nat 表内置了三个链<br>PREROUTING`，用于路由判断前所执行的规则，比如，对接收到的数据包进行 DNAT；<br>`POSTROUTING`，用于路由判断后所执行的规则，比如，对发送或转发的数据包进行 SNAT 或 MASQUERADE。<br> `OUTPUT`，类似于 PREROUTING，但只处理从本机发送出去的包。<br><br>4. SNAT<br>iptables -t nat -A POSTROUTING -s 192.168.0.0&#47;16 -j MASQUERADE<br>iptables -t nat -A POSTROUTING -s 192.168.0.2 -j SNAT --to-source 100.100.100.100<br><br>5. DNAT<br>iptables -t nat -A PREROUTING -d 100.100.100.100 -j DNAT --to-destination 192.168.0.2<br><br>6. 双向地址转换<br>$ iptables -t nat -A POSTROUTING -s 192.168.0.2 -j SNAT --to-source 100.100.100.100<br>$ iptables -t nat -A PREROUTING -d 100.100.100.100 -j DNAT --to-destination 192.168.0.2<br><br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-05 11:36:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/c0/ce/fc41ad5e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>陳先森</span>
  </div>
  <div class="_2_QraFYR_0">第一个问题是可以的，外网端口映射内网端口是一对多。就跟之前老师留言的一样，服务端对端口没有65535个端口限制但是客户端有。<br>第二个问题会有隐患，请求数量多的时候会导致软路由服务器的资源和性能下降，甚至延时和超时都有可能以至于达到系统资源的瓶颈系统的软连接数达到瓶颈或者无法正常工作</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-29 22:55:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/88/17/68aa48cb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>曾经的十字镐</span>
  </div>
  <div class="_2_QraFYR_0">我们有部分业务需要向外同步数据，但又必须保证内网服务器的安全，所以我就使用了nat网络，单纯知道外网ip是无法攻击的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 并不是这样的，通过NAT还是可以访问内部服务的，如果服务本身设计有问题，还是一样受到攻击</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-01 13:33:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/a8/8b/c5c234b6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小庄.Jerry</span>
  </div>
  <div class="_2_QraFYR_0">最近我们的一个客户，遇到问题2的问题了。该公司很多用户同时加入我们的会议系统，一般来说，客户会访问我们部署在当地数据中心的服务器，结果很多用户访问到我们数据美国数据中心的服务器了，导致糟糕的体验。<br>我们的网络team给的解决方案:禁用了我们服务器的tcp_tw_recycle。<br>看了man tcp的介绍，对于NAT网络中，要求禁掉tcp_tw_recycle。但是对于个中的原理还不是很明白，希望老师可以帮忙解释下</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: tcp_tw_recycle本身的实现就有问题，一定要禁止掉，并且在新的内核（4.1以后）里也被直接删除了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-27 08:38:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://wx.qlogo.cn/mmopen/vi_32/PiajxSqBRaEKQMM4m7NHuicr55aRiblTSEWIYe0QqbpyHweaoAbG7j2v7UUElqqeP3Ihrm3UfDPDRb1Hv8LvPwXqA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ninuxer</span>
  </div>
  <div class="_2_QraFYR_0">打卡day43<br>工作场景没用到nat，基本都是基于4层或7层的反代<br>针对第一个问题，是可以的，第二个问题不可以，我认为是有连接追踪表，文件数量，端口数量的限制</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-27 08:09:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/57/6e/b6795c44.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>夜空中最亮的星</span>
  </div>
  <div class="_2_QraFYR_0">第一个问题：是可以正常工作的，是由（源地址：源端口   目的地址：目的端口 ）来标记的<br>第二个问题：会把这台linux主机撑爆了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-25 21:31:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/93/43/0e84492d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Maxwell</span>
  </div>
  <div class="_2_QraFYR_0">vmware中虚拟机网络选择NAT模式后，IP地址经常变动，有什么方法解决么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: VM里面可以配置固定IP，或者也可以换其他模式</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-25 11:03:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/b2/91/714c0f07.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>zero</span>
  </div>
  <div class="_2_QraFYR_0">以前看iptables，一脸懵逼，现在知道了一丢丢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-02 15:10:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/79/d5/4a7971fc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Ethan</span>
  </div>
  <div class="_2_QraFYR_0">老师有两个问题，感觉需要重新讨论下：<br><br>第一个关于 SNAT 和 DNAT 的解释：<br><br>原文中说：<br><br>SNAT 主要用于，多个内网 IP 共享同一个公网 IP ，来访问外网资源的场景。<br>DNAT 主要通过公网 IP 的不同端口号，来访问内网的多种服务，同时会隐藏后端服务器的真实 IP 地址。<br><br>这里给人的感觉好像是 SNAT 和 DNAT 是两个不同的应用场景。但我理解，并不是这样。<br>首先内网的服务器想和外网通信，不可能是单方向的通信，必定涉及到一来一回。<br><br>如果单单做 SNAT，这里仅仅解决的是发送数据包时，把源地址进行替换和外网通信。<br>但外网想和内网的服务器通信，还涉及到回包的过程，将外网的目的地址（也就是发送的源地址）进行转换，也就是 DNAT。<br><br>所以通过 SNAT 和 DNAT 结合起来才能实现内网访问外网，以及隐藏后端服务器地址的作用。<br><br><br>第二个是关于一个同学的提问：<br><br>提问如下：<br>有个疑问， 看了访问baidu.com的例子，发包和收报都是需要NAT的，那是不是只配置SNAT或DNAT，就不能正常访问外网或被访问了呢？<br>老师回答：<br>作者回复: 不是的，内核有连接跟踪，知道每个请求的来源和目的<br><br>我和老师的看法不一样，我认为这个同学说的是对的，如果仅仅配置 SNAT 或者 DNAT 是无法通信的，原因就在于上面的解释，通信是一来一回的过程。<br>除非，我们隐含 SNAT 包含了 DNAT 的步骤，或者 DNAT 包含了 SNAT 的步骤。<br><br>并且这里做了实验验证：<br>使用 docker 以 run 的形式起了一个 nginx 容易，并暴露 80 端口给到外网。<br>这时外面是可以和 nginx 正常交互的。<br><br>并且查看 iptables 中 nat 表的，PREROUTING 和 POSTROUTING 链。<br>可以找到 PREROUTING 做了 DNAT。<br>POSTROUTING 做了MASQUERADE，就是动态的 snat.<br><br>当我手动把 DNAT 删除，光留下 snat 时，这时是无法正常通信的。<br>在容器内抓包后发现，流量根本进入不到容器内。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-06 11:37:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/90/f1/7f2b5e16.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>CHN-Lee-玉米</span>
  </div>
  <div class="_2_QraFYR_0">曾经遇过内网大量机器通过一个公网IP地址访问另一个公网IP，导致端口号用满的情况</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-06 23:52:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/c4/35/2cc10d43.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Wade_阿伟</span>
  </div>
  <div class="_2_QraFYR_0">对于第一个问题：当多个内网 IP 地址的端口号相同时，MASQUERADE 还可以正常工作吗？答案是肯定可以工作的。因为netfilter_conntrack中记录的是五元组信息，包括源端口、源Ip、目的端口、目的ip等信息。即便源端口相等，也能组成不同的连接记录。<br>第二个问题：如果内网 IP 地址数量或请求数比较多，会导致服务器整体的性能下降，包括导致cpu的软中断过多，nf_conntrack连接跟踪表中记录过多，导致跟踪表溢出。导致服务器table over full，从而导致服务器出现响应延时并丢包等问题。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-03 11:40:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/16/f0/3051cc84.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>兵临城下</span>
  </div>
  <div class="_2_QraFYR_0">老师，我想问一下。系统使用的，阿里云SLB（负载）+ECS，ECS上设置 iptables 禁用某个IP，如何能启作用？默认是内网的SLB的IP</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-30 10:39:48</div>
  </div>
</div>
</div>
</li>
</ul>