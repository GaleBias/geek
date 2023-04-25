<audio title="18 _ 四通八达：HTTP的重定向和跳转" src="https://static001.geekbang.org/resource/audio/ce/f0/ce5c9c5647eb34a1af654866d8533bf0.mp3" controls="controls"></audio> 
<p>在专栏<a href="https://time.geekbang.org/column/article/97837">第1讲</a>时我曾经说过，为了实现在互联网上构建超链接文档系统的设想，蒂姆·伯纳斯-李发明了万维网，使用HTTP协议传输“超文本”，让全世界的人都能够自由地共享信息。</p><p>“超文本”里含有“超链接”，可以从一个“超文本”跳跃到另一个“超文本”，对线性结构的传统文档是一个根本性的变革。</p><p>能够使用“超链接”在网络上任意地跳转也是万维网的一个关键特性。它把分散在世界各地的文档连接在一起，形成了复杂的网状结构，用户可以在查看时随意点击链接、转换页面。再加上浏览器又提供了“前进”“后退”“书签”等辅助功能，让用户在文档间跳转时更加方便，有了更多的主动性和交互性。</p><p>那么，点击页面“链接”时的跳转是怎样的呢？具体一点，比如在Nginx的主页上点了一下“download”链接，会发生什么呢？</p><p>结合之前的课程，稍微思考一下你就能得到答案：浏览器首先要解析链接文字里的URI。</p><pre><code>http://nginx.org/en/download.html
</code></pre><p>再用这个URI发起一个新的HTTP请求，获取响应报文后就会切换显示内容，渲染出新URI指向的页面。</p><p>这样的跳转动作是由浏览器的使用者主动发起的，可以称为“<strong>主动跳转</strong>”，但还有一类跳转是由服务器来发起的，浏览器使用者无法控制，相对地就可以称为“<strong>被动跳转</strong>”，这在HTTP协议里有个专门的名词，叫做“<strong>重定向</strong>”（Redirection）。</p><!-- [[[read_end]]] --><h2>重定向的过程</h2><p>其实之前我们就已经见过重定向了，在<a href="https://time.geekbang.org/column/article/102483">第12讲</a>里3××状态码时就说过，301是“永久重定向”，302是“临时重定向”，浏览器收到这两个状态码就会跳转到新的URI。</p><p>那么，它们是怎么做到的呢？难道仅仅用这两个代码就能够实现跳转页面吗？</p><p>先在实验环境里看一下重定向的过程吧，用Chrome访问URI “/18-1”，它会使用302立即跳转到“/index.html”。</p><p><img src="https://static001.geekbang.org/resource/image/ad/2d/ad5eb7546ee7ef62a9987120934b592d.png?wh=2100*652" alt=""></p><p>从这个实验可以看到，这一次“重定向”实际上发送了两次HTTP请求，第一个请求返回了302，然后第二个请求就被重定向到了“/index.html”。但如果不用开发者工具的话，你是完全看不到这个跳转过程的，也就是说，重定向是“用户无感知”的。</p><p>我们再来看看第一个请求返回的响应报文：</p><p><img src="https://static001.geekbang.org/resource/image/a4/ac/a4276db758bf90f63fd5a5c2357af4ac.png?wh=1900*739" alt=""></p><p>这里出现了一个新的头字段“Location: /index.html”，它就是301/302重定向跳转的秘密所在。</p><p>“<strong>Location</strong>”字段属于响应字段，必须出现在响应报文里。但只有配合301/302状态码才有意义，它<strong>标记了服务器要求重定向的URI</strong>，这里就是要求浏览器跳转到“index.html”。</p><p>浏览器收到301/302报文，会检查响应头里有没有“Location”。如果有，就从字段值里提取出URI，发出新的HTTP请求，相当于自动替我们点击了这个链接。</p><p>在“Location”里的URI既可以使用绝对URI，也可以使用相对URI。所谓“绝对URI”，就是完整形式的URI，包括scheme、host:port、path等。所谓“相对URI”，就是省略了scheme和host:port，只有path和query部分，是不完整的，但可以从请求上下文里计算得到。</p><p>例如，刚才的实验例子里的“Location: /index.html”用的就是相对URI。它没有说明访问URI的协议和主机，但因为是由“<a href="http://www.chrono.com/18-1">http://www.chrono.com/18-1</a>”重定向返回的响应报文，所以浏览器就可以拼出完整的URI：</p><pre><code>http://www.chrono.com/index.html
</code></pre><p>实验环境的URI“/18-1”还支持使用query参数“dst=xxx”，指明重定向的URI，你可以用这种形式再多试几次重定向，看看浏览器是如何工作的。</p><pre><code>http://www.chrono.com/18-1?dst=/15-1?name=a.json
http://www.chrono.com/18-1?dst=/17-1
</code></pre><p>注意，在重定向时如果只是在站内跳转，你可以放心地使用相对URI。但如果要跳转到站外，就必须用绝对URI。</p><p>例如，如果想跳转到Nginx官网，就必须在“nginx.org”前把“http://”都写出来，否则浏览器会按照相对URI去理解，得到的就会是一个不存在的URI“<a href="http://www.chrono.com/nginx.org%E2%80%9D">http://www.chrono.com/nginx.org”</a></p><pre><code>http://www.chrono.com/18-1?dst=nginx.org           #错误
http://www.chrono.com/18-1?dst=http://nginx.org    #正确
</code></pre><p><img src="https://static001.geekbang.org/resource/image/00/aa/006059602ee75b176a80429f49ffc9aa.png?wh=1900*826" alt=""></p><p>那么，如果301/302跳转时没有Location字段会怎么样呢？</p><p>这个你也可以自己试一下，使用第12讲里的URI“/12-1”，查询参数用“code=302”：</p><pre><code>http://www.chrono.com/12-1?code=302
</code></pre><h2>重定向状态码</h2><p>刚才我把重定向的过程基本讲完了，现在来说一下重定向用到的状态码。</p><p>最常见的重定向状态码就是301和302，另外还有几个不太常见的，例如303、307、308等。它们最终的效果都差不多，让浏览器跳转到新的URI，但语义上有一些细微的差别，使用的时候要特别注意。</p><p><strong>301</strong>俗称“永久重定向”（Moved Permanently），意思是原URI已经“永久”性地不存在了，今后的所有请求都必须改用新的URI。</p><p>浏览器看到301，就知道原来的URI“过时”了，就会做适当的优化。比如历史记录、更新书签，下次可能就会直接用新的URI访问，省去了再次跳转的成本。搜索引擎的爬虫看到301，也会更新索引库，不再使用老的URI。</p><p><strong>302</strong>俗称“临时重定向”（“Moved Temporarily”），意思是原URI处于“临时维护”状态，新的URI是起“顶包”作用的“临时工”。</p><p>浏览器或者爬虫看到302，会认为原来的URI仍然有效，但暂时不可用，所以只会执行简单的跳转页面，不记录新的URI，也不会有其他的多余动作，下次访问还是用原URI。</p><p>301/302是最常用的重定向状态码，在3××里剩下的几个还有：</p><ul>
<li>303 See Other：类似302，但要求重定向后的请求改为GET方法，访问一个结果页面，避免POST/PUT重复操作；</li>
<li>307 Temporary Redirect：类似302，但重定向后请求里的方法和实体不允许变动，含义比302更明确；</li>
<li>308 Permanent Redirect：类似307，不允许重定向后的请求变动，但它是301“永久重定向”的含义。</li>
</ul><p>不过这三个状态码的接受程度较低，有的浏览器和服务器可能不支持，开发时应当慎重，测试确认浏览器的实际效果后才能使用。</p><h2>重定向的应用场景</h2><p>理解了重定向的工作原理和状态码的含义，我们就可以<strong>在服务器端拥有主动权</strong>，控制浏览器的行为，不过要怎么利用重定向才好呢？</p><p>使用重定向跳转，核心是要理解“<strong>重定向</strong>”和“<strong>永久/临时</strong>”这两个关键词。</p><p>先来看什么时候需要重定向。</p><p>一个最常见的原因就是“<strong>资源不可用</strong>”，需要用另一个新的URI来代替。</p><p>至于不可用的原因那就很多了。例如域名变更、服务器变更、网站改版、系统维护，这些都会导致原URI指向的资源无法访问，为了避免出现404，就需要用重定向跳转到新的URI，继续为网民提供服务。</p><p>另一个原因就是“<strong>避免重复</strong>”，让多个网址都跳转到一个URI，增加访问入口的同时还不会增加额外的工作量。</p><p>例如，有的网站都会申请多个名称类似的域名，然后把它们再重定向到主站上。比如，你可以访问一下“qq.com”“github.com ”“bing.com”（记得事先清理缓存），看看它是如何重定向的。</p><p>决定要实行重定向后接下来要考虑的就是“永久”和“临时”的问题了，也就是选择301还是302。</p><p>301的含义是“<strong>永久</strong>”的。</p><p>如果域名、服务器、网站架构发生了大幅度的改变，比如启用了新域名、服务器切换到了新机房、网站目录层次重构，这些都算是“永久性”的改变。原来的URI已经不能用了，必须用301“永久重定向”，通知浏览器和搜索引擎更新到新地址，这也是搜索引擎优化（SEO）要考虑的因素之一。</p><p>302的含义是“<strong>临时</strong>”的。</p><p>原来的URI在将来的某个时间点还会恢复正常，常见的应用场景就是系统维护，把网站重定向到一个通知页面，告诉用户过一会儿再来访问。另一种用法就是“服务降级”，比如在双十一促销的时候，把订单查询、领积分等不重要的功能入口暂时关闭，保证核心服务能够正常运行。</p><h2>重定向的相关问题</h2><p>重定向的用途很多，掌握了重定向，就能够在架设网站时获得更多的灵活性，不过在使用时还需要注意两个问题。</p><p>第一个问题是“<strong>性能损耗</strong>”。很明显，重定向的机制决定了一个跳转会有两次请求-应答，比正常的访问多了一次。</p><p>虽然301/302报文很小，但大量的跳转对服务器的影响也是不可忽视的。站内重定向还好说，可以长连接复用，站外重定向就要开两个连接，如果网络连接质量差，那成本可就高多了，会严重影响用户的体验。</p><p>所以重定向应当适度使用，决不能滥用。</p><p>第二个问题是“<strong>循环跳转</strong>”。如果重定向的策略设置欠考虑，可能会出现“A=&gt;B=&gt;C=&gt;A”的无限循环，不停地在这个链路里转圈圈，后果可想而知。</p><p>所以HTTP协议特别规定，浏览器必须具有检测“循环跳转”的能力，在发现这种情况时应当停止发送请求并给出错误提示。</p><p>实验环境的URI“/18-2”就模拟了这样的一个“循环跳转”，它跳转到“/18-1”，并用参数“dst=/18-2”再跳回自己，实现了两个URI的无限循环。</p><p>使用Chrome访问这个地址，会得到“该网页无法正常运作”的结果：</p><p><img src="https://static001.geekbang.org/resource/image/4b/2f/4b91aeea08d90f173c62493934e5f52f.png?wh=1700*973" alt=""></p><h2>小结</h2><p>今天我们学习了HTTP里的重定向和跳转，简单小结一下这次的内容：</p><ol>
<li><span class="orange">重定向是服务器发起的跳转，要求客户端改用新的URI重新发送请求，通常会自动进行，用户是无感知的；</span></li>
<li><span class="orange">301/302是最常用的重定向状态码，分别是“永久重定向”和“临时重定向”；</span></li>
<li><span class="orange">响应头字段Location指示了要跳转的URI，可以用绝对或相对的形式；</span></li>
<li><span class="orange">重定向可以把一个URI指向另一个URI，也可以把多个URI指向同一个URI，用途很多；</span></li>
<li><span class="orange">使用重定向时需要当心性能损耗，还要避免出现循环跳转。</span></li>
</ol><h2>课下作业</h2><ol>
<li>301和302非常相似，试着结合第12讲，用自己的理解再描述一下两者的异同点。</li>
<li>你能结合自己的实际情况，再列出几个应当使用重定向的场景吗？</li>
</ol><p>欢迎你把自己的学习体会写在留言区，与我和其他同学一起讨论。如果你觉得有所收获，也欢迎把文章分享给你的朋友。</p><p><img src="https://static001.geekbang.org/resource/image/9b/05/9b873d25e33f86bb2818fc8b50fbff05.png?wh=1769*3215" alt="unpreview"></p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/67/f4/9a1feb59.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>钱</span>
  </div>
  <div class="_2_QraFYR_0">之前面试官好像比较喜欢问外部重定向和内部重定向的区别？<br>外部重定向，服务器会把重定向的地址给浏览器，然后浏览器再次的发起请求，地址栏的地址变化了。<br>内部重定向，服务器会直接把重定向的资源返给浏览器，不需要再次在浏览器发起请求，地址栏的地址不变。<br>重定向我的经验，主要用在未登录或者权限不足的场景，跳转到对应的登录或提升页面之中。<br>当然，通用的SSO也是这样做的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1.内部重定向对于用户来说成本低，因为只在网站服务器内部跳转，所谓的router。<br><br>2.对，这是最常见的应用场景之一。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-29 18:13:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_5b0e47</span>
  </div>
  <div class="_2_QraFYR_0">笔记：<br>主动跳转：跳转动作是由浏览器的使用者主动发起的;<br>被动跳转：跳转动作是由服务器发起的，浏览器使用者无法控制。<br><br>1、重定向状态码<br><br>301：俗称“永久重定向”，原URI已经“永久”性地不存在了，今后的所有请求都必须改用新的URI.<br>302: 俗称“临时重定向”，原URI处于“临时维护”状态，新的URI是起“顶包”作用的临时工。<br>303 See Other: 类似302，但要求重定向后的请求改为GET方法，访问一个结果页面，避免POST&#47;PUT重复操作；<br>307 Temporary Redirect: 类似302，但重定向后请求里的方法和实体不允许变动，含义比302更明确；<br>308 Permanent Redirect: 类似307，不允许重定向后的请求变动，但它是301“永久重定向”的含义<br><br>2、重定向的应用场景<br>一个最常见的原因就是“资源不可用”，需要用另一个新的URI来代替。<br>不可用的原因：如域名变更、服务器变更、网站改版、系统维护。<br>另一个原因就是“避免重复”，让多个网址都跳转到一个URI，增加访问入口的同时还不会增加额外的工作量。如：有的网站会申请多个名称类似的域名，然后把它们重定向到主站上。<br><br>3、重定向的相关问题<br>第一个问题是“性能损耗”。重定向的机制决定了一个跳转会有两次请求-应答，比正常的访问多了一次。<br>第二个问题是“循环跳转”。如果重定向的策略设置欠考虑，可能会出现“A=&gt;B=&gt;C=&gt;A”的无限循环。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-27 14:58:16</div>
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
  <div class="_2_QraFYR_0">老师，使用301会比302有较大的性能提升么</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对于单次请求来说是没什么差别的，但浏览器会对301做优化，后续的请求就不会再有跳转动作，所以会快一些。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-19 09:55:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/81/4e/d71092f4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>夏目</span>
  </div>
  <div class="_2_QraFYR_0">1、301用于废弃原地址跳转新地址，302用于暂时无法访问原地址跳转新地址，两者都需要浏览器重新发起一次请求<br>2、最开始接触重定向的时候就是用于未登录跳转登录页了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 说的很好。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-03 10:56:20</div>
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
  <div class="_2_QraFYR_0">老师，这个301，302, 303重定向要求前后协议一致吗？http不能调转https?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 没有要求，当然可以跳转到https。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-18 09:24:58</div>
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
  <div class="_2_QraFYR_0">站外重定向就要开两个连接，如果网络连接质量差，那成本可就高多了，会严重影响用户的体验。<br>------------》<br>老师这个我不理解，站外重定向，比如重定向到其他网站，那客户端的连接跟自己的服务端应该就没有什么关系了吧？为什么还有两个连接呢？不是客户端应该跟其他网站的服务器连接吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是这样的：首先与本站是一个连接，然后到外站，就必须再开一个连接，这样就是两个连接了，两次tcp握手，对于客户端成本就很高了，当然服务器是无所谓的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-22 18:31:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/11/a2/33be69a6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>毛毛</span>
  </div>
  <div class="_2_QraFYR_0">重定向和转发的区别和用途，以后章节会讲吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你说的“转发”指什么？是代理吗？如果是的话很快就会讲到。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-08 09:13:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/20/e9/fe/bc38c919.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lll</span>
  </div>
  <div class="_2_QraFYR_0">“另一个原因就是“避免重复”，让多个网址都跳转到一个 URI，增加访问入口的同时还不会增加额外的工作量。”这句话怎么理解呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 比如说Google就注册了很多的域名，比如goo.gl、g.cn等等，但都指向的是同一个服务，这样对于用户来说就很方便，随便记一个就能访问。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-19 10:39:08</div>
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
  <div class="_2_QraFYR_0">重定向报文里还可以用Refresh字段，实现延时重定向，例如”Refresh: 5; url=xxx“告诉浏览器5秒钟后再跳转<br><br>与跳转有关的还有一个”Referer“和”Refereer-Policy“（注意前者是个拼写错误，但已经”将错就错“），表示浏览器跳转的来源（即引用地址），可用于统计分析和防盗链</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 把重要的信息当做笔记保存起来，这个习惯很好。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-09 10:42:36</div>
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
  <div class="_2_QraFYR_0">1.好比去寄快递，我去到常去的寄送点，发现寄送店有一块告示栏，被告知当前地点近期处于维修状态，需要前往另一个临时寄送点办理（302状态码），临时地址即为location字段值。或者被告知当前寄送店已永久搬迁至新地址（301状态码），临时地址为location字段值。两者都表示当前访问地址已失效，区别在于一个为临时的，短期的，另一个为永久性的。<br>2.比如商城类的页面，需要浏览个人中心或者订单列表等页面时需要进行登录态校验，如果没有登录或者登录态失效了，需要重定向到登录页。某个页面进行了重构，且url发生了变化，由于老url遍布站点的很多页面，不好直接修改跳转url，此时将老url重定向到新url是比较合适的。<br>有个疑问请教一下老师：<br>内部重定向和外部重定向一般在使用场合上有什么区别？<br>查了一下 资料说内部重定向不会造成浏览器地址栏url的变化，实际对客户端是无感知的，只是代理转发到另一个url。个人感觉又不太对，因为资料上又说nginx的重定向是属于内部，但是实际客户端url确实变了，比较矛盾</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 说的挺好。<br><br>内部重定向是服务器内部处理流程的跳转，不会发给客户端让客户端去跳转。<br><br>Nginx同时支持外部和内部重定向，写法是不同的，区别就在于客户端是否有感知。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-14 16:05:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/c9/fd/5ac43929.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>天方夜</span>
  </div>
  <div class="_2_QraFYR_0">有同学提到 ajax 中的重定向，这是个有点复杂的问题。简单说，xhr 方式，存在 Location 的情况下只能看到重定向后的最终结果；fetch 方式，可以看到 Location，而且对于重定向有不同的处理模式，可以设置。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: ajax属于前端技术了，我了解的不多，欢迎同学补充交流。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-21 00:33:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/c9/fd/5ac43929.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>天方夜</span>
  </div>
  <div class="_2_QraFYR_0">从短域名跳长域名（z.cn - www.amazon.cn），从 http 跳 https，分别应当用哪种重定向，有没有最佳实践呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我觉得正文里的可以参考，就是看这个跳转是否是“临时”的，如果是永久条件就用301，而跟长短域名、http到https没有特别的关系。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-20 23:35:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTI8mFt5wSkia31qc8paRg2uPSB6AuEBDricrSxvFBuTpP3NnnflekpJ7wqvN0nRrJyu7zVbzd7Lwjxw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_steven_wang</span>
  </div>
  <div class="_2_QraFYR_0">Sso就会用重定向引导用户登录。<br>但如果location中没有地址，那浏览器也不知跳转到那，会出错。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-19 20:50:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/26/ba/a9/7432b796.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>coral</span>
  </div>
  <div class="_2_QraFYR_0">一个问题：<br>打开qq.com的时候，浏览器进行了两次跳转：<br>http:&#47;&#47;qq.com =&gt; https:&#47;&#47;qq.com =&gt;https:&#47;&#47;www.qq.com <br>访问google.ca的时候也有类似的现象<br>http:&#47;&#47;google.ca&#47; =&gt; http:&#47;&#47;www.google.ca&#47; =&gt; https:&#47;&#47;www.google.ca&#47;?gws_rd=ssl<br><br>为什么要设计中间的那一层呢？从第一步直接redirect到最后一个url不行吗？<br><br>作业<br>1. 301 和 302 非常相似，试着结合第 12 讲，用自己的理解再描述一下两者的异同点。<br>同：两者都会让访问者知道，当前url不可用，并且会在location字段里回复你要去的的url。<br>异：<br>301: “永久重定向”（Moved Permanently），表示原来的页面永久不可用，所有优化的程序都可以更新了，如浏览器可以更新bookmark，爬虫更新搜索库。<br>302: “临时重定向”（“Moved Temporarily”），要访问的页面临时不可用了，优化程序不需更新，临时跳转一下即可。<br><br>2. 你能结合自己的实际情况，再列出几个应当使用重定向的场景吗？<br>需要更换域名&#47;页面url的时候，可以给旧的url设置一个重定向，导到新的url去，这样曾经bookmark旧url的用户也可以访问到新的内容而不会看到报错页面。<br>还有把http页面重定向到https、把不带www的页面重定向到www.开头的页面也是很常见的操作。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 现在的网站都有这么一个趋势，用简短的域名，不用写出签名的“www”，但真正的域名还是有“www”的，所以就要用一个跳转。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-17 00:50:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/e5/39/951f89c8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>信信</span>
  </div>
  <div class="_2_QraFYR_0">文中提到的所有链接都返回200，和访问http:&#47;&#47;www.chrono.com&#47;一个效果。。。。。。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 建议打开开发者工具，看看uri是如何处理的。<br><br>比如http:&#47;&#47;www.chrono.com&#47;18-1?dst=&#47;15-1?name=a.json，应该是跳转到15-1。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-24 13:35:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/22/89/73397ccb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>响雨</span>
  </div>
  <div class="_2_QraFYR_0">我这边要做一个web升级，在升级过程中要展示升级进度，就打算301重定向到另一个服务来展示升级的进度</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不错。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-09 18:53:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/92/e4/abb7bfe3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>TerryGoForIt</span>
  </div>
  <div class="_2_QraFYR_0">重定向可以应用于实现负载均衡。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对，但多了一次请求的成本，比较重。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-08 09:07:16</div>
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
  <div class="_2_QraFYR_0">老师您好，<br>重定向，我一般使用在移动PC互切的情况下会使用，因为使用到了域名会不一样。还有一种情况会在判断浏览器的时候会使用到重定向，比如IE。。。<br><br>但是有一个问题，302是临时重定向，想问一下浏览器在每次访问的时候，都会直接访问原先URI吗？还是会有什么过期时间呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 302不改变原uri，所以每次都会找原uri，成本较高，应当尽量少用。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-08 08:44:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/07/53/05aa9573.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>keep it simple</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，提交几个问题：<br>1.在自己的测试中，保存了一个https:&#47;&#47;bing.com的书签，访问后得到301，location为cn.bing.com，但书签中的地址并没有改变，还是https:&#47;&#47;bing.com。是否意味着我使用的浏览器没有遵循301的规范？<br>2.关于303的使用场景是什么？如果原请求方式就是POST，用于上传一组数据，然后服务器返回303，浏览器端只能用GET，那需要上传的数据就无法被上传了吧？<br>3.关于307的描述，言外之意是302可以改变请求里的方法和实体，但服务端只返回location的情况下，浏览器也不会改变请求方法和实体吧？<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.这个没有错，书签是一个正确的地址，只是bing.com给了你一个301跳转，是否更新书签是浏览器的问题，这已经不是http协议的问题了，可以更新也可以不更新。<br><br>2.303可以用来防止客户端重复post，比如post一次，后续再多次post都转向一个固定的等待响应页面。<br><br>3.302、307这些都是对客户端的“指示”，表示服务器希望客户端接下来要怎么做最合适，决定权还是在客户端。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-21 14:12:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/2e/53/bf62683f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>狼的诱惑</span>
  </div>
  <div class="_2_QraFYR_0">老师好，又来请教问题了<br>1.状态码301，302是有些客户端遵循http规范默认支持吗？比如浏览器，是不是浏览器解析了返回状态码，解析到301或302然后解析出地响应头&#47;15-1地址然后又发起了一次http请求？能举例那些客户端不支持重定向吗？<br>2.我觉得看文章的同时，我们是不是结合着老师提供的实验源码，来了解整个来龙去脉，这样会更容易理解？虽然老师用的lua，但还是可以勉强看懂的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.浏览器为了方便用户访问网页，肯定是要自动跳转到新的uri的。如果你用Python等语言自己实现客户端，那就可以自己定义处理策略了。<br><br>2.实验环境的lua源码都很简单，只有最小的服务器逻辑，如果能参照着看是最好的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-12 22:12:00</div>
  </div>
</div>
</div>
</li>
</ul>