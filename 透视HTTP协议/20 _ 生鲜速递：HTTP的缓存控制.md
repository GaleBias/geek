<audio title="20 _ 生鲜速递：HTTP的缓存控制" src="https://static001.geekbang.org/resource/audio/2f/6e/2f82dcbcb3aa772e80b5b2c57f200b6e.mp3" controls="controls"></audio> 
<p>缓存（Cache）是计算机领域里的一个重要概念，是优化系统性能的利器。</p><p>由于链路漫长，网络时延不可控，浏览器使用HTTP获取资源的成本较高。所以，非常有必要把“来之不易”的数据缓存起来，下次再请求的时候尽可能地复用。这样，就可以避免多次请求-应答的通信成本，节约网络带宽，也可以加快响应速度。</p><p>试想一下，如果有几十K甚至几十M的数据，不是从网络而是从本地磁盘获取，那将是多么大的一笔节省，免去多少等待的时间。</p><p>实际上，HTTP传输的每一个环节基本上都会有缓存，非常复杂。</p><p>基于“请求-应答”模式的特点，可以大致分为客户端缓存和服务器端缓存，因为服务器端缓存经常与代理服务“混搭”在一起，所以今天我先讲客户端——也就是浏览器的缓存。</p><h2>服务器的缓存控制</h2><p>为了更好地说明缓存的运行机制，下面我用“生鲜速递”作为比喻，看看缓存是如何工作的。</p><p>夏天到了，天气很热。你想吃西瓜消暑，于是打开冰箱，但很不巧，冰箱是空的。不过没事，现在物流很发达，给生鲜超市打个电话，不一会儿，就给你送来一个8斤的沙瓤大西瓜，上面还贴着标签：“保鲜期5天”。好了，你把它放进冰箱，想吃的时候随时拿出来。</p><p>在这个场景里，“生鲜超市”就是Web服务器，“你”就是浏览器，“冰箱”就是浏览器内部的缓存。整个流程翻译成HTTP就是：</p><!-- [[[read_end]]] --><ol>
<li>浏览器发现缓存无数据，于是发送请求，向服务器获取资源；</li>
<li>服务器响应请求，返回资源，同时标记资源的有效期；</li>
<li>浏览器缓存资源，等待下次重用。</li>
</ol><p><img src="https://static001.geekbang.org/resource/image/a1/5b/a1968821f214df4a3ae16c9b30f99a5b.png?wh=1876*1062" alt=""></p><p>你可以访问实验环境的URI “/20-1”，看看具体的请求-应答过程。</p><p><img src="https://static001.geekbang.org/resource/image/df/d8/dfd2d20670443a782443fc3193ae1cd8.png?wh=1285*1262" alt=""></p><p>服务器标记资源有效期使用的头字段是“<strong>Cache-Control</strong>”，里面的值“<strong>max-age=30</strong>”就是资源的有效时间，相当于告诉浏览器，“这个页面只能缓存30秒，之后就算是过期，不能用。”</p><p>你可能要问了，让浏览器直接缓存数据就好了，为什么要加个有效期呢？</p><p>这是因为网络上的数据随时都在变化，不能保证它稍后的一段时间还是原来的样子。就像生鲜超市给你快递的西瓜，只有5天的保鲜期，过了这个期限最好还是别吃，不然可能会闹肚子。</p><p>“Cache-Control”字段里的“max-age”和上一讲里Cookie有点像，都是标记资源的有效期。</p><p>但我必须提醒你注意，这里的max-age是“<strong>生存时间</strong>”（又叫“新鲜度”“缓存寿命”，类似TTL，Time-To-Live），时间的计算起点是响应报文的创建时刻（即Date字段，也就是离开服务器的时刻），而不是客户端收到报文的时刻，也就是说包含了在链路传输过程中所有节点所停留的时间。</p><p>比如，服务器设定“max-age=5”，但因为网络质量很糟糕，等浏览器收到响应报文已经过去了4秒，那么这个资源在客户端就最多能够再存1秒钟，之后就会失效。</p><p>“max-age”是HTTP缓存控制最常用的属性，此外在响应报文里还可以用其他的属性来更精确地指示浏览器应该如何使用缓存：</p><ul>
<li>no-store：<strong>不允许缓存</strong>，用于某些变化非常频繁的数据，例如秒杀页面；</li>
<li>no-cache：它的字面含义容易与no-store搞混，实际的意思并不是不允许缓存，而是<strong>可以缓存</strong>，但在使用之前必须要去服务器验证是否过期，是否有最新的版本；</li>
<li>must-revalidate：又是一个和no-cache相似的词，它的意思是如果缓存不过期就可以继续使用，但过期了如果还想用就必须去服务器验证。</li>
</ul><p>听的有点糊涂吧。没关系，我拿生鲜速递来举例说明一下：</p><ul>
<li>no-store：买来的西瓜不允许放进冰箱，要么立刻吃，要么立刻扔掉；</li>
<li>no-cache：可以放进冰箱，但吃之前必须问超市有没有更新鲜的，有就吃超市里的；</li>
<li>must-revalidate：可以放进冰箱，保鲜期内可以吃，过期了就要问超市让不让吃。</li>
</ul><p>你看，这超市管的还真多啊，西瓜到了家里怎么吃还得听他。不过没办法，在HTTP协议里服务器就是这样的“霸气”。</p><p>我把服务器的缓存控制策略画了一个流程图，对照着它你就可以在今后的后台开发里明确“Cache-Control”的用法了。</p><p><img src="https://static001.geekbang.org/resource/image/1b/99/1b4f48bc0d8fb9a08b45d1f0deac8a99.png?wh=1100*1425" alt=""></p><h2>客户端的缓存控制</h2><p>现在冰箱里已经有了“缓存”的西瓜，是不是就可以直接开吃了呢？</p><p>你可以在Chrome里点几次“刷新”按钮，估计你会失望，页面上的ID一直在变，根本不是缓存的结果，明明说缓存30秒，怎么就不起作用呢？</p><p>其实不止服务器可以发“Cache-Control”头，浏览器也可以发“Cache-Control”，也就是说请求-应答的双方都可以用这个字段进行缓存控制，互相协商缓存的使用策略。</p><p>当你点“刷新”按钮的时候，浏览器会在请求头里加一个“<strong>Cache-Control: max-age=0</strong>”。因为max-age是“<strong>生存时间</strong>”，max-age=0的意思就是“我要一个最最新鲜的西瓜”，而本地缓存里的数据至少保存了几秒钟，所以浏览器就不会使用缓存，而是向服务器发请求。服务器看到max-age=0，也就会用一个最新生成的报文回应浏览器。</p><p>Ctrl+F5的“强制刷新”又是什么样的呢？</p><p>它其实是发了一个“<strong>Cache-Control: no-cache</strong>”，含义和“max-age=0”基本一样，就看后台的服务器怎么理解，通常两者的效果是相同的。</p><p><img src="https://static001.geekbang.org/resource/image/2f/49/2fc3fa639f44b98d7c19d25604c65249.png?wh=1807*1012" alt=""></p><p>那么，浏览器的缓存究竟什么时候才能生效呢？</p><p>别着急，试着点一下浏览器的“前进”“后退”按钮，再看开发者工具，你就会惊喜地发现“from disk cache”的字样，意思是没有发送网络请求，而是读取的磁盘上的缓存。</p><p>另外，如果用<a href="https://time.geekbang.org/column/article/105614">第18讲</a>里的重定向跳转功能，也可以发现浏览器使用了缓存：</p><pre><code>http://www.chrono.com/18-1?dst=20-1
</code></pre><p><img src="https://static001.geekbang.org/resource/image/f2/06/f2a12669e997ea6dc0f2228bcaf65a06.png?wh=2090*781" alt=""></p><p>这几个操作与刷新有什么区别呢？</p><p>其实也很简单，在“前进”“后退”“跳转”这些重定向动作中浏览器不会“夹带私货”，只用最基本的请求头，没有“Cache-Control”，所以就会检查缓存，直接利用之前的资源，不再进行网络通信。</p><p>这个过程你也可以用Wireshark抓包，看看是否真的没有向服务器发请求。</p><h2>条件请求</h2><p>浏览器用“Cache-Control”做缓存控制只能是刷新数据，不能很好地利用缓存数据，又因为缓存会失效，使用前还必须要去服务器验证是否是最新版。</p><p>那么该怎么做呢？</p><p>浏览器可以用两个连续的请求组成“验证动作”：先是一个HEAD，获取资源的修改时间等元信息，然后与缓存数据比较，如果没有改动就使用缓存，节省网络流量，否则就再发一个GET请求，获取最新的版本。</p><p>但这样的两个请求网络成本太高了，所以HTTP协议就定义了一系列“<strong>If</strong>”开头的“<strong>条件请求</strong>”字段，专门用来检查验证资源是否过期，把两个请求才能完成的工作合并在一个请求里做。而且，验证的责任也交给服务器，浏览器只需“坐享其成”。</p><p>条件请求一共有5个头字段，我们最常用的是“<strong>if-Modified-Since</strong>”和“<strong>If-None-Match</strong>”这两个。需要第一次的响应报文预先提供“<strong>Last-modified</strong>”和“<strong>ETag</strong>”，然后第二次请求时就可以带上缓存里的原值，验证资源是否是最新的。</p><p>如果资源没有变，服务器就回应一个“<strong>304 Not Modified</strong>”，表示缓存依然有效，浏览器就可以更新一下有效期，然后放心大胆地使用缓存了。</p><p><img src="https://static001.geekbang.org/resource/image/b2/37/b239d0804be630ce182e24ea9e4ab237.png?wh=1764*879" alt=""></p><p>“Last-modified”很好理解，就是文件的最后修改时间。ETag是什么呢？</p><p>ETag是“实体标签”（Entity Tag）的缩写，<strong>是资源的一个唯一标识</strong>，主要是用来解决修改时间无法准确区分文件变化的问题。</p><p>比如，一个文件在一秒内修改了多次，但因为修改时间是秒级，所以这一秒内的新版本无法区分。</p><p>再比如，一个文件定期更新，但有时会是同样的内容，实际上没有变化，用修改时间就会误以为发生了变化，传送给浏览器就会浪费带宽。</p><p>使用ETag就可以精确地识别资源的变动情况，让浏览器能够更有效地利用缓存。</p><p>ETag还有“强”“弱”之分。</p><p>强ETag要求资源在字节级别必须完全相符，弱ETag在值前有个“W/”标记，只要求资源在语义上没有变化，但内部可能会有部分发生了改变（例如HTML里的标签顺序调整，或者多了几个空格）。</p><p>还是拿生鲜速递做比喻最容易理解：</p><p>你打电话给超市，“我这个西瓜是3天前买的，还有最新的吗？”。超市看了一下库存，说：“没有啊，我这里都是3天前的。”于是你就知道了，再让超市送货也没用，还是吃冰箱里的西瓜吧。这就是“<strong>if-Modified-Since</strong>”和“<strong>Last-modified</strong>”。</p><p>但你还是想要最新的，就又打电话：“有不是沙瓤的西瓜吗？”，超市告诉你都是沙瓤的（Match），于是你还是只能吃冰箱里的沙瓤西瓜。这就是“<strong>If-None-Match</strong>”和“<strong>弱ETag</strong>”。</p><p>第三次打电话，你说“有不是8斤的沙瓤西瓜吗？”，这回超市给了你满意的答复：“有个10斤的沙瓤西瓜”。于是，你就扔掉了冰箱里的存货，让超市重新送了一个新的大西瓜。这就是“<strong>If-None-Match</strong>”和“<strong>强ETag</strong>”。</p><p>再来看看实验环境的URI “/20-2”。它为资源增加了ETag字段，刷新页面时浏览器就会同时发送缓存控制头“max-age=0”和条件请求头“If-None-Match”，如果缓存有效服务器就会返回304：</p><p><img src="https://static001.geekbang.org/resource/image/30/f9/30965c97bb7433eabe10008fefaeb5f9.png?wh=1812*963" alt=""></p><p>条件请求里其他的三个头字段是“If-Unmodified-Since”“If-Match”和“If-Range”，其实只要你掌握了“if-Modified-Since”和“If-None-Match”，可以轻易地“举一反三”。</p><h2>小结</h2><p>今天我们学习了HTTP的缓存控制和条件请求，用好它们可以减少响应时间、节约网络流量，一起小结一下今天的内容吧：</p><ol>
<li><span class="orange">缓存是优化系统性能的重要手段，HTTP传输的每一个环节中都可以有缓存；</span></li>
<li><span class="orange">服务器使用“Cache-Control”设置缓存策略，常用的是“max-age”，表示资源的有效期；</span></li>
<li><span class="orange">浏览器收到数据就会存入缓存，如果没过期就可以直接使用，过期就要去服务器验证是否仍然可用；</span></li>
<li><span class="orange">验证资源是否失效需要使用“条件请求”，常用的是“if-Modified-Since”和“If-None-Match”，收到304就可以复用缓存里的资源；</span></li>
<li><span class="orange">验证资源是否被修改的条件有两个：“Last-modified”和“ETag”，需要服务器预先在响应报文里设置，搭配条件请求使用；</span></li>
<li><span class="orange">浏览器也可以发送“Cache-Control”字段，使用“max-age=0”或“no_cache”刷新数据。</span></li>
</ol><p>HTTP缓存看上去很复杂，但基本原理说白了就是一句话：“没有消息就是好消息”，“没有请求的请求，才是最快的请求。”</p><h2>课下作业</h2><ol>
<li>Cache 和Cookie都是服务器发给客户端并存储的数据，你能比较一下两者的异同吗？</li>
<li>即使有“Last-modified”和“ETag”，强制刷新（Ctrl+F5）也能够从服务器获取最新数据（返回200而不是304），请你在实验环境里试一下，观察请求头和响应头，解释原因。</li>
</ol><p>欢迎你把自己的学习体会写在留言区，与我和其他同学一起讨论。如果你觉得有所收获，也欢迎把文章分享给你的朋友。</p><p><img src="https://static001.geekbang.org/resource/image/13/4b/1348aa2c81bd5d65ace3aa068b21044b.png?wh=1769*3690" alt="unpreview"></p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/0d/40/f70e5653.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>前端西瓜哥</span>
  </div>
  <div class="_2_QraFYR_0">Cache 和 Cookie 的相同点是：都会保存到浏览器中，并可以设置过期时间。<br>不同点：<br>1. Cookie 会随请求报文发送到服务器，而 Cache 不会，但可能会携带 if-Modified-Since（保存资源的最后修改时间）和 If-None-Match（保存资源唯一标识） 字段来验证资源是否过期。<br>2. Cookie 在浏览器可以通过脚本获取（如果 cookie 没有设置 HttpOnly），Cache 则无法在浏览器中获取（出于安全原因）。<br>3. Cookie 通过响应报文的 Set-Cookie 字段获得，Cache 则是位于 body 中。<br>4. 用途不同。Cookie 常用于身份识别，Cache 则是由浏览器管理，用于节省带宽和加快响应速度。<br>5. Cookie 的 max-age 是从浏览器拿到响应报文时开始计算的，而 Cache 的 max-age 是从响应报文的生成时间（Date 头字段）开始计算。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 总结的非常好。<br><br>第三点感觉有点问题，cache缓存的是完整的报文，不单单是body。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-13 00:06:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/07/d2/d7a200d5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小鸟淫太</span>
  </div>
  <div class="_2_QraFYR_0">1. cookie是方便进行身份识，cache是为了减少网络请求。<br>2. 强制刷新是因为请求头里的 If-Modified-Since 和 If-None-Match 会被清空所以会返回最新数据</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 回答正确，之前是我弄错了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-12 10:55:44</div>
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
  <div class="_2_QraFYR_0">对于第二个问题：发现强制刷新后请求头中 没有了 If-None-Match ，而且 Cache-Control: no-cache<br><br>是这个原因吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对，没有条件请求头，那么服务器就无法处理缓存，就只能返回最新的数据。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-13 10:37:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/04/0e/87adea1d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>DENG永青</span>
  </div>
  <div class="_2_QraFYR_0">Etag的工作原理<br>Etag在服务器上生成后,客户端通过If-Match或者说If-None-Match这个条件判断请求来验证资源是否修改.我们常见的是使用If-None-Match.请求一个文件的流程可能如下：<br>新的请求<br>客户端发起HTTP GET请求一个文件(css ,image,js)；服务器处理请求,返回文件内容和一堆Header(包括Etag,例如&quot;2e681a-6-5d044840&quot;),http头状态码为为200.<br><br>同一个用户第二次这个文件的请求<br>客户端在一次发起HTTP GET请求一个文件,注意这个时候客户端同时发送一个If-None-Match头,这个头中会包括上次这个文件的Etag(例如&quot;2e681a- 6-5d044840&quot;),这时服务器判断发送过来的Etag和自己计算出来的Etag,因此If-None-Match为False,不返回200,返 回304,客户端继续使用本地缓存；<br><br>注意.服务器又设置了Cache-Control:max-age和Expires时,会同时使用,也就是说在完全匹配If-Modified-Since和If-None-Match即检查完修改时间和Etag之后,服务器才能返回304.</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 写的非常详细，点赞。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-07 00:13:37</div>
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
  <div class="_2_QraFYR_0">老师我有几个问题想问一下： <br>1. F5刷新的时候，请求头加上&quot;Cache-Control: max-age=0&quot;，您文章里说，服务器用一个最新生成的报文回应浏览器，那这时候响应返回的应该是&quot;200 OK&quot;吗？为什么我在极客网页版的这个页面刷新后，有个叫&quot;106804&quot;的资源返回的是&quot;304&quot;，但是强制刷新是&quot;200 ok&quot;，产生的效果好像不同呀。这里是不是应该换一种方式说？感觉强制刷新说的有些简单了。<br>2. F5刷新发送的请求头是固定的吗？还是会根据浏览器不同而产生变化？<br>3. 200（from memory cache）和200(from disk cache)是针对内存和硬盘的，他们出现的场景分别是什么呢？<br>4. HTTP缓存有标准性的流程吗？比如说从我输入URL开始，到后续刷新或者强制刷新等？<br>5.对于&quot;must-revalidate&quot;我有疑问，本身存储机制不就是如果不过期的话可以继续使用，过期的话去请求服务器吗？那这个属性还有什么意义呢？<br>6. no-cache,no-store,max-age等属性可以共存吗？<br>问题有点多，因为网上资料质量参差不齐，解释有些也不全相同，所以在这里咨询下老师，希望老师可以解答一下，或者有推荐的讲述HTTP缓存的文章也可以，谢谢老师</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1.强制刷新请求最新的资源，没有条件请求，所以不会有304，都是200。<br><br>2.每个浏览器可能会有不同，但基本的字段是一样的。<br><br>3.缓存的位置不一样，浏览器会分别存放到内存或者硬盘上，所以会显示来源不同。<br><br>4.http只规定了缓存的用法，具体如何存放如何使用就是客户端自己灵活实现了，怎么方便怎么来。<br><br>5.过期后去验证，如果服务器返回304，那么就可以继续重用缓存，而不用下载整个资源。<br><br>6.可以看一下流程图，不是所以的属性都能共存的。当然如果你要是都写上也不是不可以，那浏览器就会“精神错乱”了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-15 22:58:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/44/a7/171c1e86.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>啦啦啦</span>
  </div>
  <div class="_2_QraFYR_0">老师，nocache，每次使用前都需要去浏览器问一下有没有过期，这不也是一次请求吗？那不和没有缓存一个意思吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不一样，如果服务器返回304，是一个很小的报文，这样浏览器就可以直接重用缓存里的数据，可以节约传输带宽。<br><br>nostore每次都会传输完整的报文，成本高。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-12 14:11:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/60/12/268826e6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Marvin</span>
  </div>
  <div class="_2_QraFYR_0">我有一个问题，就拿咱们极客时间的网页来说，会请求一个Id-00001.ts的文件，响应头中指示了cache-control: max-age= 7200， 要一个小时才过期，那么为什么每次刷新都是304, 像这种情况不应该直接200 cacahe from disk才对么？为什么明明没有过期还要去服务器协商呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 刷新时发的是条件请求，不是普通的请求，所以就必须返回304，告诉浏览器内容没有过期，可以继续用缓存。<br><br>普通请求才会直接检查缓存，然后是200 cacahe from disk。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-04 17:38:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/7e/99/c4302030.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Khirye</span>
  </div>
  <div class="_2_QraFYR_0">Hi, 我对缓存控制策略这张流程图有一些疑惑，must-revalidate是指缓存过期之后，必须要向服务器验证缓存，这一步应该是在图中”缓存最多x秒“这个判断之前的吧？<br>因为只有缓存超过了max-age的期限，才会进入”must-revalidate的判断“这一步吧？<br>烦请解惑，谢谢！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这张图是“服务器”的缓存策略，也就是说服务器应该如何设置资源的缓存参数，并不是客户端判断缓存的流程。<br><br>只要不是no-store就必然会设置max-age，所以must-revalidate是max-age的一个附加条件。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-25 13:17:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/9b/a8/6a391c66.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Leon📷</span>
  </div>
  <div class="_2_QraFYR_0">小贴士的nginx计算etag我贴下测试logngx_sprintf(etag-&gt;value.data, &quot;&quot;%xT-%xO&quot;&quot;, r-&gt;headers_out.last_modified_time,<br>r-&gt;headers_out.content_length_n)相信大家看到这里更清晰明了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-21 11:30:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/3b/37/5ae95580.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>风宇殇</span>
  </div>
  <div class="_2_QraFYR_0">这篇文章将缓存讲的比较容易理解。https:&#47;&#47;mp.weixin.qq.com&#47;s&#47;cUqkG3NETmJbglDXfSf0tg</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 挺好的文章，欢迎多来这样的分享。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-10 15:21:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/ia4ZTN1ibN5ovBVIaRYBibvEJW4CxRQde4ribsRF83bGaLDy0tqenD1X4gEAocSiaeb6XA2dQR69Z9hAktHV0bgPOGQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>walle</span>
  </div>
  <div class="_2_QraFYR_0">cache-control 中的 private 是如何识别的呢？是根据session吗还是什么方式开识别是私有缓存呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 缓存策略取决于服务器，它认为这个缓存只能存放在客户端，不能存放在代理上，就设置private。<br><br>与session无关。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-15 10:44:00</div>
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
  <div class="_2_QraFYR_0">请问老师弱ETag是服务器更新时自己判断本次的更新有没有语义的变化，如果语义有变化就重新生成一个ETag，如果没有变化不重新生成直接使用原来的，请问是这样的流程吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 强etag和etag的流程都是一样的，只是计算的方式不同，（即判断是否发生变化的方式不同）。<br><br>你的理解正确。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-12 19:01:26</div>
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
  <div class="_2_QraFYR_0">1.cache的作用为定义浏览器对静态文件如何进行缓存控制，目的是为了有效利用可复用的资源，尽可能减少客户端的请求，优化用户体验减轻服务器响应压力。常用字段值就那么一些，并有各自的含义。cookie的作用是增加了http请求的状态性，让服务器‘认识’当前访问的用户是谁，字段key,value值都可以自定义，比较灵活。<br>2.看了下天猫首页的css., js文件，普通的刷新（F5）操作中，不会在请求头中包含cache-control、if-none-match，if-Modified-Since,刷新会命中缓存文件，属于强缓存。强制刷新（ctrl + f5）在请求头中附加了cache-control: no-cache，为协商缓存，相当于设置max-age=0;所以此时不会使用本地缓存，当前页面所有的请求均是如此</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-16 00:59:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/c5/2e/0326eefc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Larry</span>
  </div>
  <div class="_2_QraFYR_0">如果响应报文什么字段都不设置，单纯的返回数据，是不是也不会缓存？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 按协议来说，没有规定，就取决于客户端，想缓存也可以。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-30 00:41:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/48/92/de629e17.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>游鱼</span>
  </div>
  <div class="_2_QraFYR_0">老师，开发提测后，有时需要清缓存才是最新的，强刷都不管用，这个有解决办法吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 看缓存是在哪里了，因为http的传输链路很长，可能在某个节点的缓存时间长，强制刷新不生效，需要具体分析，不好直接给解决方案。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-20 21:09:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/cd/43/fa115929.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>来自地狱的勇士</span>
  </div>
  <div class="_2_QraFYR_0">老师，既然Etag的算法比较复杂，需要占用服务器资源，那么，实际上服务器会使用Etag吗？看到有的资料说服务器很少会用到Etag，这个说法正确吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Nginx和Apache都有etag，但算法不同，但都不会用特别消耗计算资源的算法。<br><br>其他的web服务器就不太清楚了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-12 17:19:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/6c/d4/85ef1463.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>路漫漫</span>
  </div>
  <div class="_2_QraFYR_0">老师，根据服务器的缓存控制那个图，如果cache-control 设置了 no-cache 或 must-revalidate 那就 必须设置 max-age喽？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 按照协议要求是这样的，不然不知道应该缓存多长时间。<br><br>如果不提供max-age，浏览器也可以估算一个时间，但使用max-age还是最规范的写法。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-16 21:00:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/2a/62/52231841.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Joe</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，如果使用的是强缓存，比如Cache-Control: max-age=36000，那么在有效期内服务器上的文件发生了改变，客户端怎么才能及时获取最新的文件？更改文件指纹是可以获取最新的文件吗？如果可以这个请求流程是什么样的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 使用if系列的条件请求，用时间戳或者etag，发给服务器，服务器来判断，如果没变化就可以直接重用客户端的缓存，否则就发回新的文件。<br><br>可以再仔细看看条件请求，关键就是客户端带上一个小的验证信息，让服务器检查。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-18 16:50:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/f7/87/e7085d32.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>青莲居士</span>
  </div>
  <div class="_2_QraFYR_0">请问下 cache-control 头字段 与 if 系列的请求头字段有依赖关系么 ？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: if系列是条件请求，收到的通常是304这样的报文。<br><br>而max-age等描述的是缓存控制，只要是响应就可以设置，不一定是要有条件请求。<br><br>所以说两者没有必然联系，不过两者经常会同时出现，区分它们的应用场景还是很有必要的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-11 16:05:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/44/b0/c196c056.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>SeaYang</span>
  </div>
  <div class="_2_QraFYR_0">观察了一些网站的资源加载情况，有一些总结，老师帮忙看看呢，辛苦了<br>一、打开一个网页，比如百度、慕课网之类的，并打开开发者工具，切换到network面板<br><br>1、勾选disable cache，刷新，页面及内嵌资源文件的请求头有Cache-Control: no-cache，并且不会发送If-Modified-Since、If-None-Match（响应头设置了Last-Modified、ETag等字段）等请求头，所以文档和内嵌资源文件都会返回完整最新的，这个相当于是强制刷新了吧<br><br>2、取消勾选disable cache，刷新，页面的请求头有Cache-Control: max-age=0，而内嵌资源文件的请求头不会有Cache-Control: max-age=0。至于文档和内嵌资源文件从哪里取分两种情况：<br><br>1）若第一次请求的响应头设置了Cache-Control: no-cache或must-revalidate，且设置了Last-Modified、ETag等字段，则走条件请求，可能返回200，也可能返回304，分别取最新的数据和取本地缓存；若没设置Last-Modified、ETag等字段，则取最新完整数据<br><br>2）若第一次请求的响应头没有设置Cache-Control: no-cache或must-revalidate，文档会获取最新的数据或者304走本地缓存（响应头设置了Last-Modified、ETag等字段）；而内嵌资源稍微有点奇怪，有的资源第一次刷新时返回304，后面刷新就返回200走本地缓存了，而有的资源是一直返回200走本地缓存，不知道为什么？<br>       举个例子，在一个新的标签页，打开开发者工具，勾选disable cache，地址栏输入：https:&#47;&#47;www.imooc.com&#47;course&#47;list，回车待页面完全load，找到图片：https:&#47;&#47;static.mukewang.com&#47;static&#47;img&#47;course&#47;course-recommend2.png，取消勾选disable cache，刷新，发现这张图片返回304，后面继续刷新就返回200取本地缓存了，而对于别的一些图片则一直返回200取的本地缓存<br><br>二、对于文档来说，响应头里面Cache-Control只设置max-age似乎没啥用，刷新的话还是取的最新数据，前进后退只走本地缓存，即使max-age过期了且指定了must-revalidate或no-cache，比如在&#47;20-1和&#47;20-2之间一直前进后退，过了30秒还是取的本地缓存。唯一不确定的是对于文档内嵌资源文件，Cache-Control只设置max-age有没有用，因为一些网站的内嵌资源文件响应头的max-age通常设置的比较大，刷新页面这些内嵌资源文件一般都走的本地缓存，就不知道max-age过期了之后再刷新页面，这些内嵌资源文件还会不会走本地缓存<br><br>三、请求头中同时有If-None-Match、If-Modified-Since、Cache-Control，对于服务器来说，If-None-Match、If-Modified-Since的优先级高，也就是即使请求头有Cache-Control: no-cache，走的也是条件请求，而不是直接返回最新完整数据</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 态度非常认真，大力表扬。<br><br>关于缓存这块，虽然http协议规定的很清楚，但在实现方面，浏览器、服务器又会有各自的一套策略，去尽量做进一步的优化，而且有的可能还不按规矩来，所以会显得比较混乱。<br><br>我们在具体使用缓存的时候就要小心一些，一方面按照标准，另一方面要对不同的浏览器、服务器做测试，保证按照预想的方式处理。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-15 11:25:25</div>
  </div>
</div>
</div>
</li>
</ul>