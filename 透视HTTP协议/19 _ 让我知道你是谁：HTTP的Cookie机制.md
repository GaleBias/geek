<audio title="19 _ 让我知道你是谁：HTTP的Cookie机制" src="https://static001.geekbang.org/resource/audio/02/e4/02aa096c0f4c679d4037d358dba0d2e4.mp3" controls="controls"></audio> 
<p>在之前的<a href="https://time.geekbang.org/column/article/103270">第13讲</a>、<a href="https://time.geekbang.org/column/article/103746">第14讲</a>中，我曾经说过，HTTP是“无状态”的，这既是优点也是缺点。优点是服务器没有状态差异，可以很容易地组成集群，而缺点就是无法支持需要记录状态的事务操作。</p><p>好在HTTP协议是可扩展的，后来发明的Cookie技术，给HTTP增加了“记忆能力”。</p><h2>什么是Cookie？</h2><p>不知道你有没有看过克里斯托弗·诺兰导演的一部经典电影《记忆碎片》（Memento），里面的主角患有短期失忆症，记不住最近发生的事情。</p><p><video poster="https://static001.geekbang.org/resource/image/81/25/816df396bae0b37101543f967ff82125.jpeg" preload="none" controls=""><source src="https://media001.geekbang.org/customerTrans/fe4a99b62946f2c31c2095c167b26f9c/11137c6c-16d14222dfe-0000-0000-01d-dbacd.mp4" type="video/mp4"><source src="https://media001.geekbang.org/fccbc2d505ee4d33a5b4c61ddf2d79bd/a20b79548de949b1a9c5adc39d46334f-78b40e38c736ae3917956dc5ead50a1e-sd.m3u8" type="application/x-mpegURL"><source src="https://media001.geekbang.org/fccbc2d505ee4d33a5b4c61ddf2d79bd/a20b79548de949b1a9c5adc39d46334f-689660f8bb4929b7b37d995a528eeae1-hd.m3u8" type="application/x-mpegURL"></video></p><p>比如，电影里有个场景，某人刚跟主角说完话，大闹了一通，过了几分钟再回来，主角却是一脸茫然，完全不记得这个人是谁，刚才又做了什么，只能任人摆布。</p><p>这种情况就很像HTTP里“无状态”的Web服务器，只不过服务器的“失忆症”比他还要严重，连一分钟的记忆也保存不了，请求处理完立刻就忘得一干二净。即使这个请求会让服务器发生500的严重错误，下次来也会依旧“热情招待”。</p><p>如果Web服务器只是用来管理静态文件还好说，对方是谁并不重要，把文件从磁盘读出来发走就可以了。但随着HTTP应用领域的不断扩大，对“记忆能力”的需求也越来越强烈。比如网上论坛、电商购物，都需要“看客下菜”，只有记住用户的身份才能执行发帖子、下订单等一系列会话事务。</p><!-- [[[read_end]]] --><p>那该怎么样让原本无“记忆能力”的服务器拥有“记忆能力”呢？</p><p>看看电影里的主角是怎么做的吧。他通过纹身、贴纸条、立拍得等手段，在外界留下了各种记录，一旦失忆，只要看到这些提示信息，就能够在头脑中快速重建起之前的记忆，从而把因失忆而耽误的事情继续做下去。</p><p>HTTP的Cookie机制也是一样的道理，既然服务器记不住，那就在外部想办法记住。相当于是服务器给每个客户端都贴上一张小纸条，上面写了一些只有服务器才能理解的数据，需要的时候客户端把这些信息发给服务器，服务器看到Cookie，就能够认出对方是谁了。</p><h2>Cookie的工作过程</h2><p>那么，Cookie这张小纸条是怎么传递的呢？</p><p>这要用到两个字段：响应头字段<strong>Set-Cookie</strong>和请求头字段<strong>Cookie</strong>。</p><p>当用户通过浏览器第一次访问服务器的时候，服务器肯定是不知道他的身份的。所以，就要创建一个独特的身份标识数据，格式是“<strong>key=value</strong>”，然后放进Set-Cookie字段里，随着响应报文一同发给浏览器。</p><p>浏览器收到响应报文，看到里面有Set-Cookie，知道这是服务器给的身份标识，于是就保存起来，下次再请求的时候就自动把这个值放进Cookie字段里发给服务器。</p><p>因为第二次请求里面有了Cookie字段，服务器就知道这个用户不是新人，之前来过，就可以拿出Cookie里的值，识别出用户的身份，然后提供个性化的服务。</p><p>不过因为服务器的“记忆能力”实在是太差，一张小纸条经常不够用。所以，服务器有时会在响应头里添加多个Set-Cookie，存储多个“key=value”。但浏览器这边发送时不需要用多个Cookie字段，只要在一行里用“;”隔开就行。</p><p>我画了一张图来描述这个过程，你看过就能理解了。</p><p><img src="https://static001.geekbang.org/resource/image/9f/a4/9f6cca61802d65d063e24aa9ca7c38a4.png?wh=2100*921" alt=""></p><p>从这张图中我们也能够看到，Cookie是由浏览器负责存储的，而不是操作系统。所以，它是“浏览器绑定”的，只能在本浏览器内生效。</p><p>如果你换个浏览器或者换台电脑，新的浏览器里没有服务器对应的Cookie，就好像是脱掉了贴着纸条的衣服，“健忘”的服务器也就认不出来了，只能再走一遍Set-Cookie流程。</p><p>在实验环境里，你可以用Chrome访问URI“/19-1”，实地看一下Cookie工作过程。</p><p>首次访问时服务器会设置两个Cookie。</p><p><img src="https://static001.geekbang.org/resource/image/97/87/974063541e5f9b43893db45ac4ce3687.png?wh=1893*821" alt=""></p><p>然后刷新这个页面，浏览器就会在请求头里自动送出Cookie，服务器就能认出你了。</p><p><img src="https://static001.geekbang.org/resource/image/da/9f/da9b39d88ddd717a6e3feb6637dc3f9f.png?wh=1598*1244" alt=""></p><p>如果换成Firefox等其他浏览器，因为Cookie是存在Chrome里的，所以服务器就又“蒙圈”了，不知道你是谁，就会给Firefox再贴上小纸条。</p><h2>Cookie的属性</h2><p>说到这里，你应该知道了，Cookie就是服务器委托浏览器存储在客户端里的一些数据，而这些数据通常都会记录用户的关键识别信息。所以，就需要在“key=value”外再用一些手段来保护，防止外泄或窃取，这些手段就是Cookie的属性。</p><p>下面这个截图是实验环境“/19-2”的响应头，我来对着这个实际案例讲一下都有哪些常见的Cookie属性。</p><p><img src="https://static001.geekbang.org/resource/image/9d/5d/9dbb8b490714360475911ca04134df5d.png?wh=2091*762" alt=""></p><p>首先，我们应该<strong>设置Cookie的生存周期</strong>，也就是它的有效期，让它只能在一段时间内可用，就像是食品的“保鲜期”，一旦超过这个期限浏览器就认为是Cookie失效，在存储里删除，也不会发送给服务器。</p><p>Cookie的有效期可以使用Expires和Max-Age两个属性来设置。</p><p>“<strong>Expires</strong>”俗称“过期时间”，用的是绝对时间点，可以理解为“截止日期”（deadline）。“<strong>Max-Age</strong>”用的是相对时间，单位是秒，浏览器用收到报文的时间点再加上Max-Age，就可以得到失效的绝对时间。</p><p>Expires和Max-Age可以同时出现，两者的失效时间可以一致，也可以不一致，但浏览器会优先采用Max-Age计算失效期。</p><p>比如在这个例子里，Expires标记的过期时间是“GMT 2019年6月7号8点19分”，而Max-Age则只有10秒，如果现在是6月6号零点，那么Cookie的实际有效期就是“6月6号零点过10秒”。</p><p>其次，我们需要<strong>设置Cookie的作用域</strong>，让浏览器仅发送给特定的服务器和URI，避免被其他网站盗用。</p><p>作用域的设置比较简单，“<strong>Domain</strong>”和“<strong>Path</strong>”指定了Cookie所属的域名和路径，浏览器在发送Cookie前会从URI中提取出host和path部分，对比Cookie的属性。如果不满足条件，就不会在请求头里发送Cookie。</p><p>使用这两个属性可以为不同的域名和路径分别设置各自的Cookie，比如“/19-1”用一个Cookie，“/19-2”再用另外一个Cookie，两者互不干扰。不过现实中为了省事，通常Path就用一个“/”或者直接省略，表示域名下的任意路径都允许使用Cookie，让服务器自己去挑。</p><p>最后要考虑的就是<strong>Cookie的安全性</strong>了，尽量不要让服务器以外的人看到。</p><p>写过前端的同学一定知道，在JS脚本里可以用document.cookie来读写Cookie数据，这就带来了安全隐患，有可能会导致“跨站脚本”（XSS）攻击窃取数据。</p><p>属性“<strong>HttpOnly</strong>”会告诉浏览器，此Cookie只能通过浏览器HTTP协议传输，禁止其他方式访问，浏览器的JS引擎就会禁用document.cookie等一切相关的API，脚本攻击也就无从谈起了。</p><p>另一个属性“<strong>SameSite</strong>”可以防范“跨站请求伪造”（XSRF）攻击，设置成“SameSite=Strict”可以严格限定Cookie不能随着跳转链接跨站发送，而“SameSite=Lax”则略宽松一点，允许GET/HEAD等安全方法，但禁止POST跨站发送。</p><p>还有一个属性叫“<strong>Secure</strong>”，表示这个Cookie仅能用HTTPS协议加密传输，明文的HTTP协议会禁止发送。但Cookie本身不是加密的，浏览器里还是以明文的形式存在。</p><p>Chrome开发者工具是查看Cookie的有力工具，在“Network-Cookies”里可以看到单个页面Cookie的各种属性，另一个“Application”面板里则能够方便地看到全站的所有Cookie。</p><p><img src="https://static001.geekbang.org/resource/image/a8/9d/a8accc7e1836fa348c2fbd29f494069d.png?wh=3683*985" alt=""></p><p><img src="https://static001.geekbang.org/resource/image/37/6e/37fbfef0490a20179c0ee274dccf5e6e.png?wh=2097*795" alt=""></p><h2>Cookie的应用</h2><p>现在回到我们最开始的话题，有了Cookie，服务器就有了“记忆能力”，能够保存“状态”，那么应该如何使用Cookie呢？</p><p>Cookie最基本的一个用途就是<strong>身份识别</strong>，保存用户的登录信息，实现会话事务。</p><p>比如，你用账号和密码登录某电商，登录成功后网站服务器就会发给浏览器一个Cookie，内容大概是“name=yourid”，这样就成功地把身份标签贴在了你身上。</p><p>之后你在网站里随便访问哪件商品的页面，浏览器都会自动把身份Cookie发给服务器，所以服务器总会知道你的身份，一方面免去了重复登录的麻烦，另一方面也能够自动记录你的浏览记录和购物下单（在后台数据库或者也用Cookie），实现了“状态保持”。</p><p>Cookie的另一个常见用途是<strong>广告跟踪</strong>。</p><p>你上网的时候肯定看过很多的广告图片，这些图片背后都是广告商网站（例如Google），它会“偷偷地”给你贴上Cookie小纸条，这样你上其他的网站，别的广告就能用Cookie读出你的身份，然后做行为分析，再推给你广告。</p><p>这种Cookie不是由访问的主站存储的，所以又叫“第三方Cookie”（third-party cookie）。如果广告商势力很大，广告到处都是，那么就比较“恐怖”了，无论你走到哪里它都会通过Cookie认出你来，实现广告“精准打击”。</p><p>为了防止滥用Cookie搜集用户隐私，互联网组织相继提出了DNT（Do Not Track）和P3P（Platform for Privacy Preferences Project），但实际作用不大。</p><h2>小结</h2><p>今天我们学习了HTTP里的Cookie知识。虽然现在已经出现了多种Local Web Storage技术，能够比Cookie存储更多的数据，但Cookie仍然是最通用、兼容性最强的客户端数据存储手段。</p><p>简单小结一下今天的内容：</p><ol>
<li><span class="orange">Cookie是服务器委托浏览器存储的一些数据，让服务器有了“记忆能力”；</span></li>
<li><span class="orange">响应报文使用Set-Cookie字段发送“key=value”形式的Cookie值；</span></li>
<li><span class="orange">请求报文里用Cookie字段发送多个Cookie值；</span></li>
<li><span class="orange">为了保护Cookie，还要给它设置有效期、作用域等属性，常用的有Max-Age、Expires、Domain、HttpOnly等；</span></li>
<li><span class="orange">Cookie最基本的用途是身份识别，实现有状态的会话事务。</span></li>
</ol><p>还要提醒你一点，因为Cookie并不属于HTTP标准（RFC6265，而不是RFC2616/7230），所以语法上与其他字段不太一致，使用的分隔符是“;”，与Accept等字段的“,”不同，小心不要弄错了。</p><h2>课下作业</h2><ol>
<li>如果Cookie的Max-Age属性设置为0，会有什么效果呢？</li>
<li>Cookie的好处已经很清楚了，你觉得它有什么缺点呢？</li>
</ol><p>欢迎你把自己的学习体会写在留言区，与我和其他同学一起讨论。如果你觉得有所收获，也欢迎把文章分享给你的朋友。</p><p><img src="https://static001.geekbang.org/resource/image/f0/97/f03db082760cfa8920b266ce44f52597.png?wh=1769*2877" alt="unpreview"></p><p><img src="https://static001.geekbang.org/resource/image/56/63/56d766fc04654a31536f554b8bde7b63.jpg?wh=1110*659" alt="unpreview"></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/75/00/618b20da.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>徐海浪</span>
  </div>
  <div class="_2_QraFYR_0">1. 如果 Cookie 的 Max-Age 属性设置为 0，会有什么效果呢？<br>设置为0，服务器0秒就让Cookie失效，即立即失效，服务器不存Cookie。<br>2. Cookie 的好处已经很清楚了，你觉得它有什么缺点呢？<br>好处：方便了市民<br>缺点：方便了黑客:)</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: √</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-10 08:59:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/99/2a/f7e19dcc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>放开那个猴子</span>
  </div>
  <div class="_2_QraFYR_0">广告追踪没看明白呀，能否详细讲讲</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是这样的，网站的页面里会嵌入很多广告代码，里面就会访问广告商，在浏览器里存储广告商的cookie。<br><br>你换到其他网站，上面也有这个广告商的广告代码，因为都是一个广告商网站，自然就能够读取之前设置的cookie，也就获得了你的信息。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-10 23:22:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/b6/17/76f29bfb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_66666</span>
  </div>
  <div class="_2_QraFYR_0">既然max-age=0会立即失效，那不就等于无记忆了？那干嘛还用cookie？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: max-age=0是指不能缓存，但在会话期间是可用的，浏览器会话关闭之前可以用cookie记录用户的信息。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-15 19:27:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/0d/40/f70e5653.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>前端西瓜哥</span>
  </div>
  <div class="_2_QraFYR_0">1. （我修改 Lua 文件测试了一下）如果 Max-Age 设置为0，浏览器中该 Cookie 失效，即便这个 Cookie 已存在于浏览器中，且尚未过期。另外 Web 应用开发中，可以通过这种方式消除掉用户的登陆状态，此外记得在服务器的 session 中移除该 cookie 和其对应的用户信息。<br>2. Cookie 的缺点：<br>（1） 不安全。如果被中间人获取到 Cookie，完全将它作为用户凭证冒充用户。解决方案是使用 https 进行加密。<br>（2）有数量和大小限制。另外 Cookie 太大也不好，传输的数据会变大。<br>（3）客户端可能不会保存 Cookie。比如用 telnet 收发数据，用户禁用浏览器 Cookie 保存功能的情况。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-11 22:37:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/e9/0b/1171ac71.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>WL</span>
  </div>
  <div class="_2_QraFYR_0">对于XSS和XSRF一直不是很理解希望老师帮忙解答一下：<br>1. XSS攻击是指第三方的JS代码读取到浏览器A网站的Cookie然后冒充我去访问A网站吗？<br>2.XSRF是指浏览器从A网站跳转到B网站是会带上A网站的Cookie吗？这个不是由Domain和Path已经限定了吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1，是的。<br><br>2.你理解的反了，应该是带上B网站的cookie。<br><br>https:&#47;&#47;developer.mozilla.org&#47;zh-CN&#47;docs&#47;Web&#47;HTTP&#47;Cookies<br><br>这个链接里举的例子也许能够帮助你理解。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-10 19:06:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/7a/ee/257a22fb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>饭饭</span>
  </div>
  <div class="_2_QraFYR_0">Max-age：-1 的时候会永久有效吧 ？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: rfc里有说明，如果max-age &lt;=0，统一按0算，立即过期。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-10 16:36:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/30/8a/b5ca7286.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>业余草</span>
  </div>
  <div class="_2_QraFYR_0">属性“HttpOnly”、“Secure”、“SameSite”很少见，老师可以给几个配套例子，后面答疑篇，可以来个攻防实战！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 其实并不少见，上几个大站，用开发者工具看看就能看到。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-10 14:47:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/e6/3a/382cf024.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>rongyefeng</span>
  </div>
  <div class="_2_QraFYR_0">还有一个属性叫“Secure”，表示这个 Cookie 仅能用 HTTPS 协议加密传输，明文的 HTTP 协议会禁止发送。<br>但 Cookie 本身不是加密的，浏览器里还是以明文的形式存在。<br>这里的“ Cookie 本身”怎么理解？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 意思是cookie在传输过程是被https加密的，不能用明文的http传输，但在本地，它还是明文，没有被加密，还是能够被任意查看。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-28 09:46:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/89/3f/830d74f4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>cp3yqng</span>
  </div>
  <div class="_2_QraFYR_0">域名+路径的方式存储cookie，感觉像只有一台业务服务器，那后台如何过分布式系统呢，用户中心是一个系统，核心业务是其他的系统，这里cookie肯定要共享，应该有一级域名和二级域名等等的概念吧，麻烦老师在解释解释。我本人是做移动端开发的，都是自己把token写在网络底层的请求头中，其实核心思想是一样的，但是缺点就是所有的域名里面都带token，这样也不好，好像还有优化的空间。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 只要cookie设置了domain和path属性，浏览器在访问uri时就会根据这两个属性有选择地发送cookie。<br><br>需要根据自己的业务需求，恰当设置cookie的作用域，太大太小都不好。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-10 08:17:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/76/1d/b4262bdc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>大小兵</span>
  </div>
  <div class="_2_QraFYR_0">要是能把session和token也说一下就好了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个不在http范围之内，而且篇幅有限，还望见谅。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-10 01:17:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKzqiaZnBw2myRWY802u48Rw3W2zDtKoFQ6vN63m4FdyjibM21FfaOYe8MbMpemUdxXJeQH6fRdVbZA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>kissingers</span>
  </div>
  <div class="_2_QraFYR_0">哈哈，就是九品芝麻官里的半个烧饼：你后人凭这个来找我</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: nice。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-10 09:01:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/59/f6/ed66d1c1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>chengzise</span>
  </div>
  <div class="_2_QraFYR_0">老师好，我理解的广告跟踪那个原理是：用户访问A网站，A会给用户设置cookies（T），用户再次访问网站B时，浏览器会带上cookies（T），B网站就能够识别到被A标记的用户。这里A给用户设置的cookies里面是不是把domain设置为B网站？不然凭什么访问B网站的时候，浏览器会带上A设置的cookies。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 在这个场景里B是“第三方cookie”，不是A设置的，所以只要在页面里嵌入了B相关的代码，就会带上B的cookie。<br><br>完全弄清楚可能还需要一点html的知识。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-10 14:15:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/a0/a3/8da99bb0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>业余爱好者</span>
  </div>
  <div class="_2_QraFYR_0">1.Max-Age: 是永久有效的意思。 <br>2.cookie在浏览器端禁用，还有就是安全性，因为在本地是明文存储的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1不对，max-age=0，就是立即过期，不允许缓存，只能在浏览器运行期间有效。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-10 07:22:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/b3/c7/57e789fe.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>quaeast</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，我一只搞不懂cookie和session的区别</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Cookie是存在客户端的，session是存在服务器的，都是用来保存用户的会话信息。<br><br>Cookie属于http协议的规定，而session不是协议规定，只是服务器的一种实现方式。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-15 04:01:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/f8/99/8e760987.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>許敲敲</span>
  </div>
  <div class="_2_QraFYR_0">还有个名词，session 这个适合cookie放在一起的，这个老师能简单解释下嘛？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: session是指服务器保持与客户端的会话过程，利用了cookie来存储会话信息，它是服务器上的概念。<br><br>session本身与http关系不大，是在http的应用过程中出现的，基于cookie，是cookie的一个具体应用实例。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-26 17:09:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/b8/11/26838646.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>彧豪</span>
  </div>
  <div class="_2_QraFYR_0">话说现在还有用cookie的吗？感觉似乎不是太安全吧，毕竟各种明文传输，我司是这么做的：浏览器登录成功, 服务端会设置一个密文的cookie, 然后浏览器再用带着这个cookie值的请求头字段去请求用户信息的接口来获取用户信息, 当然了还有其他的头字段，然后登陆之类的接口中的相应头也看不到set-cookie字段，但是浏览器的cookie中却能被设置上一个cookie键值对，以及是密文的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 有些公司为了兼容性还是用cookie的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-21 18:08:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/9a/0a/da55228e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>院长。</span>
  </div>
  <div class="_2_QraFYR_0">老师我是那个回复URI会跳转的那个，其他的测试案例也都会跳转。<br>您说的实验环境openresty需要我配置什么嘛？我应该是顺着看您的文章的，您专栏里哪一章有介绍我漏看的吗？<br>如果是我漏看的麻烦老师说下在哪一节，谢谢老师啦。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感觉是dns域名解析被缓存了，走了其他真实网站，没有使用本地的hosts文件，可能要清理系统的域名缓存，具体方法可以搜一下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-11 13:31:43</div>
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
  <div class="_2_QraFYR_0">samesite的加入是为了在一定程度上防范CSRF，比如a.com页面中有个恶意表单，提交url为b.com的域名为开头且为银行转账接口地址，且为post请求，用户误操作后会将本地cookie（domain为b.com）发送，导致账户损失，设置samesite为Lax后可避免该cookie的发送，防止该情况出现。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-14 17:32:23</div>
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
  <div class="_2_QraFYR_0">1：如果 Cookie 的 Max-Age 属性设置为 0，会有什么效果呢？<br>有效期为0秒，那就是服务器委托给浏览器暂存的信息，在浏览器一关闭时就失效。<br>如果想设置Cookie永不过期，怎么搞设置一个超大过期时间嘛？特殊值-1行不行？<br><br>2：Cookie 的好处已经很清楚了，你觉得它有什么缺点呢？<br>Cookie好处是能够保持状态信息，同样这个特点也是坏处尤其是被不怀好意的人获取之后。除此之外多传信息总会占用更多的带宽，降低传输效率，不过想传多一点也不行她的大小也是有限制的。所以，出现了别的存储技术来解决这个问题。<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.-1好像和0是等效的，时间长有点记不清了，你可以看一下rfc。<br><br>2.总结的不错。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-29 18:50:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/8a/e7/a6c603cf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>GitHubGanKai</span>
  </div>
  <div class="_2_QraFYR_0">老师，有几个问题我不是很明白1⃣️：如果用户没有登录，浏览器向服务器发送请求的时候，是否还会产生cookie呢？如果会产生，那么用来干嘛呢？2⃣️：有关session，他是保存在服务器上的，如果用户没有登录，服务器是否还会产生session吗？我的理解是：应该不会产生吧！毕竟服务器保存session需要额外的空间！而且用户又没有登录，保存它做什么呢🤔？3⃣️：假设现在有三个浏览器，一个在深圳，一个在北京，一个在上海，它们都是没有登录的，同时访问同一台服务器，假设没有session的存在，会出现服务器返回数据的时候，出现无法一一对应的情况吗？就是，本来这个响应是返回给深圳那个浏览器的，结果错乱的发送给北京的那台浏览器去了！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.cookie并不是需要登录才会产生，而是用来记录网站的信息，比如你不登录，它也可以记录访问时间、历史记录等，可以理解成是当做一个匿名的guest用户。<br><br>2.同理，session在服务器端保存的是用户的信息，只要你登录网站，就可以认为是一个匿名用户，就可以产生session。当然网站也可以不记录，决定权在它自己。<br><br>3.每次http请求都是一个来回，是在一个tcp上发送的，所以肯定不会乱。但因为http是无状态的，如果不用cookie，就无法区分每次请求是否为同一个用户。所以，通常的做法是在cookie里把你当做一个匿名用户。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-13 23:54:53</div>
  </div>
</div>
</div>
</li>
</ul>