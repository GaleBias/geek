<audio title="36 _ WAF：保护我们的网络服务" src="https://static001.geekbang.org/resource/audio/f5/0f/f53e1775cb784eeb10197f3bd6fa1b0f.mp3" controls="controls"></audio> 
<p>在前些天的“安全篇”里，我谈到了HTTPS，它使用了SSL/TLS协议，加密整个通信过程，能够防止恶意窃听和窜改，保护我们的数据安全。</p><p>但HTTPS只是网络安全中很小的一部分，仅仅保证了“通信链路安全”，让第三方无法得知传输的内容。在通信链路的两端，也就是客户端和服务器，它是无法提供保护的。</p><p>因为HTTP是一个开放的协议，Web服务都运行在公网上，任何人都可以访问，所以天然就会成为黑客的攻击目标。</p><p>而且黑客的本领比我们想象的还要大得多。虽然不能在传输过程中做手脚，但他们还可以“假扮”成合法的用户访问系统，然后伺机搞破坏。</p><h2>Web服务遇到的威胁</h2><p>黑客都有哪些手段来攻击Web服务呢？我给你大概列出几种常见的方式。</p><p>第一种叫“<strong>DDoS</strong>”攻击（distributed denial-of-service attack），有时候也叫“洪水攻击”。</p><p>黑客会控制许多“僵尸”计算机，向目标服务器发起大量无效请求。因为服务器无法区分正常用户和黑客，只能“照单全收”，这样就挤占了正常用户所应有的资源。如果黑客的攻击强度很大，就会像“洪水”一样对网站的服务能力造成冲击，耗尽带宽、CPU和内存，导致网站完全无法提供正常服务。</p><p>“DDoS”攻击方式比较“简单粗暴”，虽然很有效，但不涉及HTTP协议内部的细节，“技术含量”比较低，不过下面要说的几种手段就不一样了。</p><!-- [[[read_end]]] --><p>网站后台的Web服务经常会提取出HTTP报文里的各种信息，应用于业务，有时会缺乏严格的检查。因为HTTP报文在语义结构上非常松散、灵活，URI里的query字符串、头字段、body数据都可以任意设置，这就带来了安全隐患，给了黑客“<strong>代码注入</strong>”的可能性。</p><p>黑客可以精心编制HTTP请求报文，发送给服务器，服务程序如果没有做防备，就会“上当受骗”，执行黑客设定的代码。</p><p>“<strong>SQL注入</strong>”（SQL injection）应该算是最著名的一种“代码注入”攻击了，它利用了服务器字符串拼接形成SQL语句的漏洞，构造出非正常的SQL语句，获取数据库内部的敏感信息。</p><p>另一种“<strong>HTTP头注入</strong>”攻击的方式也是类似的原理，它在“Host”“User-Agent”“X-Forwarded-For”等字段里加入了恶意数据或代码，服务端程序如果解析不当，就会执行预设的恶意代码。</p><p>在之前的<a href="https://time.geekbang.org/column/article/106034">第19讲</a>里，也说过一种利用Cookie的攻击手段，“<strong>跨站脚本</strong>”（XSS）攻击，它属于“JS代码注入”，利用JavaScript脚本获取未设防的Cookie。</p><h2>网络应用防火墙</h2><p>面对这么多的黑客攻击手段，我们应该怎么防御呢？</p><p>这就要用到“<strong>网络应用防火墙</strong>”（Web Application Firewall）了，简称为“<strong>WAF</strong>”。</p><p>你可能对传统的“防火墙”比较熟悉。传统“防火墙”工作在三层或者四层，隔离了外网和内网，使用预设的规则，只允许某些特定IP地址和端口号的数据包通过，拒绝不符合条件的数据流入或流出内网，实质上是<strong>一种网络数据过滤设备</strong>。</p><p>WAF也是一种“防火墙”，但它工作在七层，看到的不仅是IP地址和端口号，还能看到整个HTTP报文，所以就能够对报文内容做更深入细致的审核，使用更复杂的条件、规则来过滤数据。</p><p>说白了，WAF就是一种“<strong>HTTP入侵检测和防御系统</strong>”。</p><p><img src="https://static001.geekbang.org/resource/image/e8/a3/e8369d077454e5b92e3722e7090551a3.png?wh=1192*648" alt=""></p><p>WAF都能干什么呢？</p><p>通常一款产品能够称为WAF，要具备下面的一些功能：</p><ul>
<li>IP黑名单和白名单，拒绝黑名单上地址的访问，或者只允许白名单上的用户访问；</li>
<li>URI黑名单和白名单，与IP黑白名单类似，允许或禁止对某些URI的访问；</li>
<li>防护DDoS攻击，对特定的IP地址限连限速；</li>
<li>过滤请求报文，防御“代码注入”攻击；</li>
<li>过滤响应报文，防御敏感信息外泄；</li>
<li>审计日志，记录所有检测到的入侵操作。</li>
</ul><p>听起来WAF好像很高深，但如果你理解了它的工作原理，其实也不难。</p><p>它就像是平时编写程序时必须要做的函数入口参数检查，拿到HTTP请求、响应报文，用字符串处理函数看看有没有关键字、敏感词，或者用正则表达式做一下模式匹配，命中了规则就执行对应的动作，比如返回403/404。</p><p>如果你比较熟悉Apache、Nginx、OpenResty，可以自己改改配置文件，写点JS或者Lua代码，就能够实现基本的WAF功能。</p><p>比如说，在Nginx里实现IP地址黑名单，可以利用“map”指令，从变量$remote_addr获取IP地址，在黑名单上就映射为值1，然后在“if”指令里判断：</p><pre><code>map $remote_addr $blocked {
    default       0;
    &quot;1.2.3.4&quot;     1;
    &quot;5.6.7.8&quot;     1;
}


if ($blocked) {
    return 403 &quot;you are blocked.&quot;;  
}
</code></pre><p>Nginx的配置文件只能静态加载，改名单必须重启，比较麻烦。如果换成OpenResty就会非常方便，在access阶段进行判断，IP地址列表可以使用cosocket连接外部的Redis、MySQL等数据库，实现动态更新：</p><pre><code>local ip_addr = ngx.var.remote_addr

local rds = redis:new()
if rds:get(ip_addr) == 1 then 
    ngx.exit(403) 
end
</code></pre><p>看了上面的两个例子，你是不是有种“跃跃欲试”的冲动了，想自己动手开发一个WAF？</p><p>不过我必须要提醒你，在网络安全领域必须时刻记得“<strong>木桶效应</strong>”（也叫“短板效应”）。网站的整体安全不在于你加固的最强的那个方向，而是在于你可能都没有意识到的“短板”。黑客往往会“避重就轻”，只要发现了网站的一个弱点，就可以“一点突破”，其他方面的安全措施也就都成了“无用功”。</p><p>所以，使用WAF最好“<strong>不要重新发明轮子</strong>”，而是使用现有的、比较成熟的、经过实际考验的WAF产品。</p><h2>全面的WAF解决方案</h2><p>这里我就要“隆重”介绍一下WAF领域里的最顶级产品了：<span class="orange">ModSecurity</span>，它可以说是WAF界“事实上的标准”。</p><p>ModSecurity是一个开源的、生产级的WAF工具包，历史很悠久，比Nginx还要大几岁。它开始于一个私人项目，后来被商业公司Breach Security收购，现在则是由TrustWave公司的SpiderLabs团队负责维护。</p><p>ModSecurity最早是Apache的一个模块，只能运行在Apache上。因为其品质出众，大受欢迎，后来的2.x版添加了Nginx和IIS支持，但因为底层架构存在差异，不够稳定。</p><p>所以，这两年SpiderLabs团队就开发了全新的3.0版本，移除了对Apache架构的依赖，使用新的“连接器”来集成进Apache或者Nginx，比2.x版更加稳定和快速，误报率也更低。</p><p>ModSecurity有两个核心组件。第一个是“<strong>规则引擎</strong>”，它实现了自定义的“SecRule”语言，有自己特定的语法。但“SecRule”主要基于正则表达式，还是不够灵活，所以后来也引入了Lua，实现了脚本化配置。</p><p>ModSecurity的规则引擎使用C++11实现，可以从<a href="https://github.com/SpiderLabs/ModSecurity">GitHub</a>上下载源码，然后集成进Nginx。因为它比较庞大，编译很费时间，所以最好编译成动态模块，在配置文件里用指令“load_module”加载：</p><pre><code>load_module modules/ngx_http_modsecurity_module.so;
</code></pre><p>只有引擎还不够，要让引擎运转起来，还需要完善的防御规则，所以ModSecurity的第二个核心组件就是它的“<strong>规则集</strong>”。</p><p>ModSecurity源码提供一个基本的规则配置文件“<strong>modsecurity.conf-recommended</strong>”，使用前要把它的后缀改成“conf”。</p><p>有了规则集，就可以在Nginx配置文件里加载，然后启动规则引擎：</p><pre><code>modsecurity on;
modsecurity_rules_file /path/to/modsecurity.conf;
</code></pre><p>“modsecurity.conf”文件默认只有检测功能，不提供入侵阻断，这是为了防止误杀误报，把“SecRuleEngine”后面改成“On”就可以开启完全的防护：</p><pre><code>#SecRuleEngine DetectionOnly
SecRuleEngine  On
</code></pre><p>基本的规则集之外，ModSecurity还额外提供一个更完善的规则集，为网站提供全面可靠的保护。这个规则集的全名叫“<strong>OWASP ModSecurity 核心规则集</strong>”（Open Web Application Security Project ModSecurity Core Rule Set），因为名字太长了，所以有时候会简称为“核心规则集”或者“CRS”。</p><p><img src="https://static001.geekbang.org/resource/image/ad/48/add929f8439c64f29db720d30f7de548.png?wh=1142*514" alt=""></p><p>CRS也是完全开源、免费的，可以从GitHub上下载：</p><pre><code>git clone https://github.com/SpiderLabs/owasp-modsecurity-crs.git
</code></pre><p>其中有一个“<strong>crs-setup.conf.example</strong>”的文件，它是CRS的基本配置，可以用“Include”命令添加到“modsecurity.conf”里，然后再添加“rules”里的各种规则。</p><pre><code>Include /path/to/crs-setup.conf
Include /path/to/rules/*.conf
</code></pre><p>你如果有兴趣可以看一下这些配置文件，里面用“SecRule”定义了很多的规则，基本的形式是“SecRule 变量 运算符 动作”。不过ModSecurity的这套语法“自成一体”，比较复杂，要完全掌握不是一朝一夕的事情，我就不详细解释了。</p><p>另外，ModSecurity还有强大的审计日志（Audit Log）功能，记录任何可疑的数据，供事后离线分析。但在生产环境中会遇到大量的攻击，日志会快速增长，消耗磁盘空间，而且写磁盘也会影响Nginx的性能，所以一般建议把它关闭：</p><pre><code>SecAuditEngine off  #RelevantOnly
SecAuditLog /var/log/modsec_audit.log
</code></pre><h2>小结</h2><p>今天我们一起学习了“网络应用防火墙”，也就是WAF，使用它可以加固Web服务。</p><ol>
<li><span class="orange">Web服务通常都运行在公网上，容易受到“DDoS”、“代码注入”等各种黑客攻击，影响正常的服务，所以必须要采取措施加以保护；</span></li>
<li><span class="orange">WAF是一种“HTTP入侵检测和防御系统”，工作在七层，为Web服务提供全面的防护；</span></li>
<li><span class="orange">ModSecurity是一个开源的、生产级的WAF产品，核心组成部分是“规则引擎”和“规则集”，两者的关系有点像杀毒引擎和病毒特征库；</span></li>
<li><span class="orange">WAF实质上是模式匹配与数据过滤，所以会消耗CPU，增加一些计算成本，降低服务能力，使用时需要在安全与性能之间找到一个“平衡点”。</span></li>
</ol><h2>课下作业</h2><ol>
<li>HTTPS为什么不能防御DDoS、代码注入等攻击呢？</li>
<li>你还知道有哪些手段能够抵御网络攻击吗？</li>
</ol><p>欢迎你把自己的学习体会写在留言区，与我和其他同学一起讨论。如果你觉得有所收获，也欢迎把文章分享给你的朋友。</p><p><img src="https://static001.geekbang.org/resource/image/b9/24/b9e48b813c98bb34b4b433b7326ace24.png?wh=1769*3710" alt="unpreview"></p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/75/aa/21275b9d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>闫飞</span>
  </div>
  <div class="_2_QraFYR_0">DDoS攻击从产生攻击对象的协议的角度来看可以分为L4攻击和L7攻击，前者其实是针对TCP状态机的恶意hack，比如攻击三次握手机制的SYN Flood和攻击四次握手的TIME WAIT2等，这些方面的防范超出了HTTPS的范畴，需要特殊的安全网管来过滤，清洗和识别。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-19 08:20:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4d/fd/0aa0e39f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>许童童</span>
  </div>
  <div class="_2_QraFYR_0">HTTPS 为什么不能防御 DDoS、代码注入等攻击呢？<br>DDoS、代码注入本身是遵循HTTPS协议的，它的攻击面不在HTTPS协议层，而在其它层面，所以HTTPS 不能防御此类攻击。<br><br>你还知道有哪些手段能够抵御网络攻击吗？<br>我还知道有CSP内容安全策略，CSRF防御，SYN cookie，流速控制等手段。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-19 14:10:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/af/e6/9c77acff.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我行我素</span>
  </div>
  <div class="_2_QraFYR_0">https是做数据加密防止泄露，而ddos是以数量取胜（伪装成正常请求），</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-19 09:27:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/e4/4f/df6d810d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Maske</span>
  </div>
  <div class="_2_QraFYR_0">https是对数据的进行加密传输，并且保证了数据的有效性，而不关心数据本身是什么，所以不能防止sql注入。DDos本身也是符合http规范的请求，所以https也无法将其识别为攻击。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 说的对。 </p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-27 16:12:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/d9/78/8a328299.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>佳佳大魔王</span>
  </div>
  <div class="_2_QraFYR_0">问题一，我觉得是因为https只提供了对通信过程中的数据进行加密，如果黑客用大量资源充当客户端，对服务器进行大量请求，https是无法阻止的，代码注入也是一样</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-19 08:07:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/86/e3/a31f6869.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span> 尿布</span>
  </div>
  <div class="_2_QraFYR_0">“CC攻击”（Challenge Collapser）是“DDos”的一种，它使用代理服务器发动攻击</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-11 22:51:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/93/ff/87d8de89.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>snake</span>
  </div>
  <div class="_2_QraFYR_0">那能否用waf来做https流量的入侵检测和防御呢？https的密文的，waf应该看不到吧？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 当然可以。<br><br>waf是内容检测，在服务器端必然要看到解密后的数据，比如ModSecurity就是Nginx的一个模块，嵌入了Nginx的处理流程，在接入https流量之后才能起作用。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-23 16:53:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/a4/7e/963c037c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Aaron</span>
  </div>
  <div class="_2_QraFYR_0">这节课内容很重要呀，但有种只讲了冰山一角的感觉。老师能不能自己系统回答一下『你还知道有哪些手段能够抵御网络攻击吗？』呢？好期待官方答案~</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: web安全的话题太大，我也不是这方面的专家，只是比较熟悉waf，其他的手段真就不太了解了。<br><br>不过我觉得，用waf就可以抵御八成的web攻击了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-02 13:49:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/f6/00/2a248fd8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>二星球</span>
  </div>
  <div class="_2_QraFYR_0">老师好，向您请教最近遇到的一个问题，APP应用通过http长链接向后台发送请求，中间有几个代理服务器，偶尔发现app发送的请求返回的状态码是正常的200，但是没有返回值，没有出错，后台也没有收到请求，这是什么原因，该如何解决呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果链路中有多个代理，那么长连接实际上连的是第一个代理服务器，而不是源服务器。<br><br>所以可能是中间的代理出了问题，给了你错误的响应。<br><br>看看能否在代理上加上日志，看看请求的处理过程。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-19 12:44:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83er6OV33jHia3U9LYlZEx2HrpsELeh3KMlqFiaKpSAaaZeBttXRAVvDXUgcufpqJ60bJWGYGNpT7752w/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>dog_brother</span>
  </div>
  <div class="_2_QraFYR_0">项目中使用过基于业务的规则引擎，完全自研的；waf应该是规则引擎比较典型的应用了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对，waf本质上就是各种各样的规则。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-06 14:54:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/PiajxSqBRaEIqzCbNMTJHEVNER4p8bsd8RHv3Sg84Ykd8uYQkOwV57ZqXTLib0AqtP7csKxOrICeFvdUrQWFKl0Q/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张良</span>
  </div>
  <div class="_2_QraFYR_0">测试JS脚本共计	&lt;script&gt;alert(&#39;ok&#39;);&lt;&#47;script&gt;</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-11 17:51:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/bd/b0/8b808d33.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>fakership</span>
  </div>
  <div class="_2_QraFYR_0">老师讲的太干货了 还想着2d看完。。 高估了。。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 学习的事情急不得，尤其是关于网络协议的，慢慢看，有可能的话自己做个心得笔记。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-16 23:35:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/c1/60/fc3689d0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小谢同学</span>
  </div>
  <div class="_2_QraFYR_0">请问老师如果我的业务接入ddos服务后，是否高防节点只会对请求流量进行4层行为的检查？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不是太明白你说的问题，抱歉啦。<br><br>waf只支持对http流量防护，四层和以下的防护它是做不了的，需要用底层的防火墙。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-08 19:02:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/52/3c/d6fcb93a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张三</span>
  </div>
  <div class="_2_QraFYR_0">打卡！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: nice。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-15 09:07:35</div>
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
  <div class="_2_QraFYR_0">HTTPS 为什么不能防御 DDoS、代码注入等攻击呢？<br>因为HTTPS的核心工作是加密解密通过HTTPS传输的内容，保证传输的内容是安全的，至于内容是一个“炸弹”还是一把“匕首”她是管不着的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对，它保证的是内容安全，但不关心内容。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-05 12:05:03</div>
  </div>
</div>
</div>
</li>
</ul>