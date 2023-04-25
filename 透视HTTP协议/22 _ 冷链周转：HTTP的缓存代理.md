<audio title="22 _ 冷链周转：HTTP的缓存代理" src="https://static001.geekbang.org/resource/audio/06/97/0623d9e97995f131e46202f146be6f97.mp3" controls="controls"></audio> 
<p>在<a href="https://time.geekbang.org/column/article/106804">第20讲</a>中，我介绍了HTTP的缓存控制，<a href="https://time.geekbang.org/column/article/107577">第21讲</a>我介绍了HTTP的代理服务。那么，把这两者结合起来就是这节课所要说的“<strong>缓存代理</strong>”，也就是支持缓存控制的代理服务。</p><p>之前谈到缓存时，主要讲了客户端（浏览器）上的缓存控制，它能够减少响应时间、节约带宽，提升客户端的用户体验。</p><p>但HTTP传输链路上，不只是客户端有缓存，服务器上的缓存也是非常有价值的，可以让请求不必走完整个后续处理流程，“就近”获得响应结果。</p><p>特别是对于那些“读多写少”的数据，例如突发热点新闻、爆款商品的详情页，一秒钟内可能有成千上万次的请求。即使仅仅缓存数秒钟，也能够把巨大的访问流量挡在外面，让RPS（request per second）降低好几个数量级，减轻应用服务器的并发压力，对性能的改善是非常显著的。</p><p>HTTP的服务器缓存功能主要由代理服务器来实现（即缓存代理），而源服务器系统内部虽然也经常有各种缓存（如Memcache、Redis、Varnish等），但与HTTP没有太多关系，所以这里暂且不说。</p><h2>缓存代理服务</h2><p>我还是沿用“生鲜速递+便利店”的比喻，看看缓存代理是怎么回事。</p><p>便利店作为超市的代理，生意非常红火，顾客和超市双方都对现状非常满意。但时间一长，超市发现还有进一步提升的空间，因为每次便利店接到顾客的请求后都要专车跑一趟超市，还是挺麻烦的。</p><!-- [[[read_end]]] --><p>干脆这样吧，给便利店配发一个大冰柜。水果海鲜什么的都可以放在冰柜里，只要产品在保鲜期内，就允许顾客直接从冰柜提货。这样便利店就可以一次进货多次出货，省去了超市之间的运输成本。</p><p><img src="https://static001.geekbang.org/resource/image/5e/c2/5e8d10b5758685850aeed2a473a6cdc2.png?wh=1580*757" alt=""></p><p>通过这个比喻，你可以看到：在没有缓存的时候，代理服务器每次都是直接转发客户端和服务器的报文，中间不会存储任何数据，只有最简单的中转功能。</p><p>加入了缓存后就不一样了。</p><p>代理服务收到源服务器发来的响应数据后需要做两件事。第一个当然是把报文转发给客户端，而第二个就是把报文存入自己的Cache里。</p><p>下一次再有相同的请求，代理服务器就可以直接发送304或者缓存数据，不必再从源服务器那里获取。这样就降低了客户端的等待时间，同时节约了源服务器的网络带宽。</p><p>在HTTP的缓存体系中，缓存代理的身份十分特殊，它“<span class="orange">既是客户端，又是服务器</span>”，同时也“<span class="orange">既不是客户端，又不是服务器</span>”。</p><p>说它“即是客户端又是服务器”，是因为它面向源服务器时是客户端，在面向客户端时又是服务器，所以它即可以用客户端的缓存控制策略也可以用服务器端的缓存控制策略，也就是说它可以同时使用第20讲的各种“Cache-Control”属性。</p><p>但缓存代理也“即不是客户端又不是服务器”，因为它只是一个数据的“中转站”，并不是真正的数据消费者和生产者，所以还需要有一些新的“Cache-Control”属性来对它做特别的约束。</p><h2>源服务器的缓存控制</h2><p><a href="https://time.geekbang.org/column/article/106804">第20讲</a>介绍了4种服务器端的“Cache-Control”属性：max-age、no-store、no-cache和must-revalidate，你应该还有印象吧？</p><p>这4种缓存属性可以约束客户端，也可以约束代理。</p><p>但客户端和代理是不一样的，客户端的缓存只是用户自己使用，而代理的缓存可能会为非常多的客户端提供服务。所以，需要对它的缓存再多一些限制条件。</p><p>首先，我们要区分客户端上的缓存和代理上的缓存，可以使用两个新属性“<strong>private</strong>”和“<strong>public</strong>”。</p><p>“private”表示缓存只能在客户端保存，是用户“私有”的，不能放在代理上与别人共享。而“public”的意思就是缓存完全开放，谁都可以存，谁都可以用。</p><p>比如你登录论坛，返回的响应报文里用“Set-Cookie”添加了论坛ID，这就属于私人数据，不能存在代理上。不然，别人访问代理获取了被缓存的响应就麻烦了。</p><p>其次，缓存失效后的重新验证也要区分开（即使用条件请求“Last-modified”和“ETag”），“<strong>must-revalidate</strong>”是只要过期就必须回源服务器验证，而新的“<strong>proxy-revalidate</strong>”只要求代理的缓存过期后必须验证，客户端不必回源，只验证到代理这个环节就行了。</p><p>再次，缓存的生存时间可以使用新的“<strong>s-maxage</strong>”（s是share的意思，注意maxage中间没有“-”），只限定在代理上能够存多久，而客户端仍然使用“max-age”。</p><p>还有一个代理专用的属性“<strong>no-transform</strong>”。代理有时候会对缓存下来的数据做一些优化，比如把图片生成png、webp等几种格式，方便今后的请求处理，而“no-transform”就会禁止这样做，不许“偷偷摸摸搞小动作”。</p><p>这些新的缓存控制属性比较复杂，还是用“便利店冷柜”来举例好理解一些。</p><p>水果上贴着标签“private, max-age=5”。这就是说水果不能放进冷柜，必须直接给顾客，保鲜期5天，过期了还得去超市重新进货。</p><p>冻鱼上贴着标签“public, max-age=5, s-maxage=10”。这个的意思就是可以在冰柜里存10天，但顾客那里只能存5天，过期了可以来便利店取，只要在10天之内就不必再找超市。</p><p>排骨上贴着标签“max-age=30, proxy-revalidate, no-transform”。因为缓存默认是public的，那么它在便利店和顾客的冰箱里就都可以存30天，过期后便利店必须去超市进新货，而且不能擅自把“大排”改成“小排”。</p><p>下面的流程图是完整的服务器端缓存控制策略，可以同时控制客户端和代理。</p><p><img src="https://static001.geekbang.org/resource/image/09/35/09266657fa61d0d1a720ae3360fe9535.png?wh=1076*1778" alt=""></p><p>我还要提醒你一点，源服务器在设置完“Cache-Control”后必须要为报文加上“Last-modified”或“ETag”字段。否则，客户端和代理后面就无法使用条件请求来验证缓存是否有效，也就不会有304缓存重定向。</p><h2>客户端的缓存控制</h2><p>说完了服务器端的缓存控制策略，稍微歇一口气，我们再来看看客户端。</p><p>客户端在HTTP缓存体系里要面对的是代理和源服务器，也必须区别对待，这里我就直接上图了，来个“看图说话”。</p><p><img src="https://static001.geekbang.org/resource/image/47/92/47c1a69c800439e478c7a4ed40b8b992.png?wh=1086*1823" alt=""></p><p>max-age、no-store、no-cache这三个属性在<a href="https://time.geekbang.org/column/article/106804">第20讲</a>已经介绍过了，它们也是同样作用于代理和源服务器。</p><p>关于缓存的生存时间，多了两个新属性“<strong>max-stale</strong>”和“<strong>min-fresh</strong>”。</p><p>“max-stale”的意思是如果代理上的缓存过期了也可以接受，但不能过期太多，超过x秒也会不要。“min-fresh”的意思是缓存必须有效，而且必须在x秒后依然有效。</p><p>比如，草莓上贴着标签“max-age=5”，现在已经在冰柜里存了7天。如果有请求“max-stale=2”，意思是过期两天也能接受，所以刚好能卖出去。</p><p>但要是“min-fresh=1”，这是绝对不允许过期的，就不会买走。这时如果有另外一个菠萝是“max-age=10”，那么“7+1&lt;10”，在一天之后还是新鲜的，所以就能卖出去。</p><p>有的时候客户端还会发出一个特别的“<strong>only-if-cached</strong>”属性，表示只接受代理缓存的数据，不接受源服务器的响应。如果代理上没有缓存或者缓存过期，就应该给客户端返回一个504（Gateway Timeout）。</p><h2>实验环境</h2><p>信息量有些大，到这里你是不是有点头疼了，好在我们还有实验环境，用URI“/22-1”试一下吧。</p><p>它设置了“Cache-Control: public, max-age=10, s-maxage=30”，数据可以在浏览器里存10秒，在代理上存30秒，你可以反复刷新，看看代理和源服务器是怎么响应的，同样也可以配合Wireshark抓包。</p><p>代理在响应报文里还额外加了“X-Cache”“X-Hit”等自定义头字段，表示缓存是否命中和命中率，方便你观察缓存代理的工作情况。</p><p><img src="https://static001.geekbang.org/resource/image/4d/e8/4d210fa1adccb7299d632ed7e66391e8.png?wh=1892*884" alt=""></p><h2>其他问题</h2><p>缓存代理的知识就快讲完了，下面再简单说两个相关的问题。</p><p>第一个是“<strong>Vary</strong>”字段，在<a href="https://time.geekbang.org/column/article/104024">第15讲</a>曾经说过，它是内容协商的结果，相当于报文的一个版本标记。</p><p>同一个请求，经过内容协商后可能会有不同的字符集、编码、浏览器等版本。比如，“Vary: Accept-Encoding”“Vary: User-Agent”，缓存代理必须要存储这些不同的版本。</p><p>当再收到相同的请求时，代理就读取缓存里的“Vary”，对比请求头里相应的“ Accept-Encoding”“User-Agent”等字段，如果和上一个请求的完全匹配，比如都是“gzip”“Chrome”，就表示版本一致，可以返回缓存的数据。</p><p>另一个问题是“<strong>Purge</strong>”，也就是“缓存清理”，它对于代理也是非常重要的功能，例如：</p><ul>
<li>过期的数据应该及时淘汰，避免占用空间；</li>
<li>源站的资源有更新，需要删除旧版本，主动换成最新版（即刷新）；</li>
<li>有时候会缓存了一些本不该存储的信息，例如网络谣言或者危险链接，必须尽快把它们删除。</li>
</ul><p>清理缓存的方法有很多，比较常用的一种做法是使用自定义请求方法“PURGE”，发给代理服务器，要求删除URI对应的缓存数据。</p><h2>小结</h2><ol>
<li><span class="orange">计算机领域里最常用的性能优化手段是“时空转换”，也就是“时间换空间”或者“空间换时间”，HTTP缓存属于后者；</span></li>
<li><span class="orange">缓存代理是增加了缓存功能的代理服务，缓存源服务器的数据，分发给下游的客户端；</span></li>
<li><span class="orange">“Cache-Control”字段也可以控制缓存代理，常用的有“private”“s-maxage”“no-transform”等，同样必须配合“Last-modified”“ETag”等字段才能使用；</span></li>
<li><span class="orange">缓存代理有时候也会带来负面影响，缓存不良数据，需要及时刷新或删除。</span></li>
</ol><h2>课下作业</h2><ol>
<li>加入了代理后HTTP的缓存复杂了很多，试着用自己的语言把这些知识再整理一下，画出有缓存代理时浏览器的工作流程图，加深理解。</li>
<li>缓存的时间策略很重要，太大太小都不好，你觉得应该如何设置呢？</li>
</ol><p>欢迎你把自己的学习体会写在留言区，与我和其他同学一起讨论。如果你觉得有所收获，也欢迎把文章分享给你的朋友。</p><p><img src="https://static001.geekbang.org/resource/image/54/b8/54fddf71fc45f1055eff0b59b67dffb8.png?wh=1769*2779" alt="unpreview"></p><p></p>
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
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/PiajxSqBRaEKkDB6420zwODZTJL6icKKTpyFKuVF9GRjj1V5ziaibADbrpDMmicF8Ad5fmBjycibEg3yhpwlVOLzzxRQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Teresa</span>
  </div>
  <div class="_2_QraFYR_0">针对作业一的回答：<br>浏览器拿到一个网址的时候，先判断是否允许缓存，允许会先查看本地缓存:1.有缓存并在缓存可用期那直接拿来用。2.缓存不存在或者不可用 那需要请求。<br>浏览器拿到host，判断：1.ip+port 那直接请求对应的服务器 2.域名 那开展一系列的dns递归查询：先拿dns缓存，没有缓存-&gt;本地dns服务器-&gt;根dns服务器-&gt;顶级dns服务器-&gt;权威dns服务器-&gt;GSLB，查到ip返回最优ip组实现负载均衡，浏览器随机或者轮询取一个ip开始它的http请求之旅。<br>浏览器判断该网页是否允许缓存，然后添加Cache-Control的各种字段no-store是否允许缓存&#47;no-cache缓存必须进行验证&#47;noly-if-cached只接受代理的缓存等,max-age最大生存时间 max-stale 短时间过期可用 min-fresh 最短有效时间等。If-Modified-Since&#47;if-None-Match&#47;Last-modified&#47;ETag等字段用于判断服务端是否有更新。然后将请求发给代理服务器。请求代理服务器，如果是第一次，要经历浏览器和代理服务器的3次tcp握手进行连接，连接成功，发送http请求。<br>代理服务器拿到请求，首先查看是否允许缓存，允许那就查看自己本地缓存有没有，通过查看max-age&#47;max-stale&#47;min-fresh等信息判断是否过期，没有过期直接拿来用，将数据返回给客户端。如果过期了，代理服务器将用客户端的请求，再次像真实服务器进行请求。如果也是第一次连接，需要经历代理服务器和真实服务器的3次tcp握手，连接成功，发送请求。<br>真实服务器收到请求之后，通过if-Modified-Since&#47;Last-Modified&#47;if-None-Match&#47;ETag等字段判断是否有更新，没有更新，直接返回304。如果有更新，则将数据打包http response 返回。返回头字段会添加Cache-Control字段，用来判断缓存的控制策略以及生存周期，no-store不允许缓存&#47;no-cache使用缓存必须先验证&#47;must-revalidate缓存不过期可用过期必须重新请求验证&#47;proxy-revalidate缓存过期只要求代理进行请求验证  private不能在代理层保存只能在客户端保存&#47;public缓存完全开放  s-maxage缓存在代理上可以缓存的时间 no-transform不允许代理对缓存做任何的改动。然后根据业务需求判断该地址是不是需要重定向，如果需要是短期的重定向还是永久的重定向，按需将状态码修改为301或者302。最后真实服务器将数据打包成http相应 回给代理服务器。<br>代理服务器收到真实服务器的回应数据，首先会查看Cache-Control里的字段，是否允许它进行缓存，如果是private，代理服务器不进行缓存，直接返回给客户端。public则根据s-maxage&#47;no-transform进行缓存，如果可以优化并且代理服务器需要优化，那可能会先优化数据，否则同时将数据回发给客户端。<br>客户端收到数据，如果是304，则直接拿缓存数据进行渲染，并修改相关缓存变量，比如时间，以及缓存使用策略。如果收到了301或者302，那么客户端会再次发起新的url请求，进行跳转到最终的页面。<br>最后，底层tcp 经过4次挥手，完成关闭连接。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 整理的非常详细完整，32个赞！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-27 11:37:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/11/f6/36bd007d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>龙宝宝</span>
  </div>
  <div class="_2_QraFYR_0">max-stale相当于延长了过期时间，min-fresh相当于缩短了过期时间，可以这样理解吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以这么理解，也是一个很好的角度。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-29 17:40:43</div>
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
  <div class="_2_QraFYR_0">首先，老师的敬业精神令人钦佩，几乎每问必答，再者，没有给人一种你怎么这么笨这么简单的问题你还问的感受，而且看到有些地方不严谨也会干净利索的说自己可能弄错了。是个经验丰富技术精干为人真诚的大哥形象，工作或生活中有这样的朋友或大哥是一件很幸运的事情。<br><br>缓存的时间策略很重要，太大太小都不好，你觉得应该如何设置呢？<br>我觉得需要根据具体情况来定：<br>如果缓存的内容不变，那可以把缓存时间设置为永久。<br>如果缓存的内容会变化，但周期较长，可以根据她的变化周期来设置，比如：一天或一周<br>如果缓存的内容变化频繁，那缓存的过期时间就需要更短了，比如：一分钟<br>如果缓存的内容随时变化，且没啥规律，那还是不用用了<br>总之是根据场景来的核心是在提速的愿望能实现的前提下，数据也是最新的，否则不如不用缓存，从另一个角度来讲不用缓存几乎是不可能的，缓存在处处使用着，因为计算机本身就在各种各样的使用着缓存。如果完全不用直接从磁盘获取数据，也可以认为是使用缓存的一种特殊情况，缓存的过期时间为零即使即过期。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.我的缺点就是太实在，所以一直只能当小兵，当不了boss，笑。<br><br>2.从http的缓存里也可以学到很多通用的知识，用在系统的其他地方也是可以的，所以说学习协议对程序员真的是一个基本功。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-30 10:09:44</div>
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
  <div class="_2_QraFYR_0">老师您好，我想请问一下，为什么有的地方说cache-control默认是private（比如cache-control的百度百科），有的地方说默认是public（比如您这篇文章），是百度百科的是错误的吗？还是根据场景不同所以默认不同吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我看了一下rfc，对于private和public没有明确的默认值说法，可能是我弄错了，需要再测试看看。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-17 08:16:09</div>
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
  <div class="_2_QraFYR_0">老师您好，<br><br>我想问一下，现在大多数网站都开启了 HTTPS，缓存代理还有用武之地吗？我对 HTTPS 的理解是它是端到端加密。介于客户端与服务器中间的缓存服务器是没法解密，缓存数据的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个需要看缓存代理怎么配置了。<br><br>如果它在四层，只是转发，看不到https的内容，就无法缓存。<br><br>如果它在七层，本身实际上也是一个客户端，能够与后端通信看到https的内容，就可以缓存。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-06 02:38:46</div>
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
  <div class="_2_QraFYR_0">请问时间换空间是什么呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 比如数据压缩，就是时间换空间，增加了计算时间，减少了数据量。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-29 08:05:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/1f/b0/83054a91.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ttsunami</span>
  </div>
  <div class="_2_QraFYR_0">代理服务器真的那么听话吗？ 源给个private，结果自己却偷偷做一些操作也可以吧？如何验证代理服务器的处理是否夹带私心呢？ 纯靠自觉吗？ 哈哈  颇有 将在外军令有所不受的味道</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 说的很对，http不是强制性的规范，代理不遵守也是完全可以的，如果有私下操作我们也很难判断出来，http本身没有这方面的要求。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-10 22:20:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/75/aa/21275b9d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>闫飞</span>
  </div>
  <div class="_2_QraFYR_0">min-fresh的含义是距离过期时间必须不短于约定的时间，保证取到的是短期内不会过时的内容。<br><br>但是这里有个风险是，万一服务器端因为某些原因重新刷新了资源(服务迁移等)，那么怎么反向通知缓存服务器去清理资源呢？尤其是已经返回给客户端的之前标记为fresh的资源?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: http协议里没有对此做出规定，一般的做法是由源站向代理发送pull请求，要求代理主动更新缓存。<br><br>这个pull请求不属于http协议，具体实现就看两者之间的约定了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-18 08:13:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/04/71/0b949a4c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>何用</span>
  </div>
  <div class="_2_QraFYR_0">老师，CDN 服务是不是就是缓存代理的一种应用？还有文中图片的 X-Accel 是什么意思呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1，是的。<br><br>2. X-Accel是自定义字段，和x-powered-by差不多，意思是被谁加速。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-17 09:24:12</div>
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
  <div class="_2_QraFYR_0">如果cache-control中没有 no-store no-cache must-revalidate这时浏览器会怎么处理？<br><br>Max-stale min-fresh 那个优先级高，如果两个都有，那个生效？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.没有这些就按普通的缓存策略来处理，在有效期内直接用，过期发条件请求。<br><br>2.这两个属性本身就是冲突的，如果同时给出，那就由服务器自己决定策略了，rfc本身没有对此做出规定。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-23 06:18:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/75/00/618b20da.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>徐海浪</span>
  </div>
  <div class="_2_QraFYR_0">我们的做法是源服务器跟代理服务器联动，源服务器有文件变化(版本发布)，触发更新代理服务器缓存。如果没有联动的机制，简单粗暴根据应用升级周期设置过期时间也可以，但这样会多做无用功。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 欢迎经验分享！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-17 13:14:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/ee/d4/204d0c6d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>居培波</span>
  </div>
  <div class="_2_QraFYR_0">老师能结合nginx讲下缓存及代理吗？还是后面探索篇有讲。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 只要理解的http的缓存代理，nginx的是比较容易掌握的，可以结合nginx的文档，看proxy相关的指令，逐条对照http的功能。<br><br>单纯讲nginx就有点太大了，讲不过来。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-17 10:42:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/24/b9/b9/9e4d7aa4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>乘风破浪</span>
  </div>
  <div class="_2_QraFYR_0">“must-revalidate”是只要过期就必须回源服务器验证，而新的“proxy-revalidate”只要求代理的缓存过期后必须验证，客户端不必回源，只验证到代理这个环节就行了。<br><br>看了一下MDN关于这两个头字段的解释<br><br>must-revalidate<br>Indicates that once a resource becomes stale, caches must not use their stale copy without successful validation on the origin server.<br>proxy-revalidate<br>Like must-revalidate, but only for shared caches (e.g., proxies). Ignored by private caches.<br><br>感觉MDN说的proxy-revalidate的意思似乎是对于存储在代理服务器上的共享缓存的验证策略，如果过期，必须回源验真，而不是说客户端的缓存【私有缓存】失效后回源到代理服务器验证。<br>而这也反过来推论，must-revalidate是向上游服务器验证cache，验证不一定回源到源服务器。<br><br>---请大师指正。。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我又仔细看了一下rfc7234，感觉是我没有太表述清楚。<br><br>proxy-revalidate意思是客户端可以到存储在代理服务器上的缓存做验证，私有缓存必须回源，其他不必回源。<br><br>同时代理服务器上的缓存失效了当然也必须回源验证。<br><br>表述上稍微有点绕，抱歉。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-03 10:07:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/81/e6/6cafed37.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>旅途</span>
  </div>
  <div class="_2_QraFYR_0">我还要提醒你一点，源服务器在设置完“Cache-Control”后必须要为报文加上“Last-modified”或“ETag”字段。否则，客户端和代理后面就无法使用条件请求来验证缓存是否有效，也就不会有 304 缓存重定向<br><br>老师 这句话 没理解 能再说下吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: “Cache-Control”字段最好和“Last-modified”或“ETag”字段搭配使用，这样客户端就可以更好地利用缓存，用条件请求来验证缓存是否有效。<br><br>如果只有“Cache-Control”，没有“Last-modified”或“ETag”，那么客户端在缓存失效后就无法发出条件请求，就得重新传一遍内容，浪费了带宽。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-12 00:06:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/9d/a4/e481ae48.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lesserror</span>
  </div>
  <div class="_2_QraFYR_0">1. 老师，我看那个完整的服务器端缓存控制策略图，在图上no-cache和must-revalidate两个一般只能存在一个吧？不能同时传递。<br>2. 还有：“比较常用的一种做法是使用自定义请求方法“PURGE””，意思是自己在代码里面写个方法处理缓存的数据，对吧？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.是的。<br><br>2.没错，http允许自定义请求方法和处理逻辑。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-10 10:24:05</div>
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
  <div class="_2_QraFYR_0">1. 文中第一张图服务器缓存控制，其实是服务器根据缓存策略向response中插入各header的过程，缓存服务器，浏览器根据header处理缓存。<br><br>2. 文中客户端缓存过程中，是浏览器根据自己的缓存策略，向服务器发请求时，设置各header。但这些策略浏览器怎么知道呢？是用户通过浏览器设置界面设置吗？还是ajax请求时设置？如果不是前后端分类的应用，怎么设置这些header?<br><br>3. 既然服务器端有自己的缓存策略，那客户端请求上来时，服务器会根据客户端请求header调整策略吗？如果会，是应用服务器自己处理，还是程序员代码预先写好处理逻辑？<br><br>4. 什么情况下客户端会有only-if-cached</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.正确。<br><br>2.http客户端有很多种，浏览器只是其中之一，如果用Python、php自己写客户端，那就可以使用这些缓存策略。<br>浏览器通常有一些最基本的策略，而ajax等就可以自己灵活设置。<br><br>3.服务器控制资源和缓存，它会检查请求资源的有效期，与客户端的请求比对，返回304或者是新的内容，应用服务器和应用服务器都可以设置。<br><br>4.only-if-cached这个我也没有见过具体的应用场景，但既然有这个属性，就应该是有用的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-22 07:01:15</div>
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
  <div class="_2_QraFYR_0">老师请教一个有关302的问题，就是再前端代码中有个post请求，去请求后端服务器，当后端服务器处理完业务逻辑后，需要重定向到另一个网站，返回的是一个302的状态码和响应头Location是另一个网站地址。这时候问题出现了，当返回到前端的时候，浏览器没有自动跳转重定向后的地址，而是当作接口去请求了需要跳转后的地址，然后就出现了跨域的问题<br>这个是什么原因呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以参考一下第18讲，改用303 see other。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-17 20:40:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/a5/98/a65ff31a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>djfhchdh</span>
  </div>
  <div class="_2_QraFYR_0">max-stale和min-fresh还是不太明白~~</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: max-stale是可以接受的过期时间，min-fresh是可以接受的新鲜时间。<br><br>不好理解也没事，这两个属性用的不多，可以以后实际遇到了再体会。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-17 11:04:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/26/eb/d7/90391376.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ifelse</span>
  </div>
  <div class="_2_QraFYR_0">HTTP 的服务器缓存功能主要由代理服务器来实现（即缓存代理），而源服务器系统内部虽然也经常有各种缓存（如 Memcache、Redis、Varnish 等），但与 HTTP 没有太多关系。--记下来</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: very nice</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-28 17:37:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/98/1c/d7a1439e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>KaKaKa</span>
  </div>
  <div class="_2_QraFYR_0">老师，看到评论有这样一个问题，我也好奇<br>【作者回复:<br>1.max-age不能用在请求头里，只能在响应头里指定资源的有效期。】这句我有点疑问，在第20讲里，在客户端的缓存控制里，您说了，【当你点“刷新”按钮的时候，浏览器会在请求头里加一个“Cache-Control: max-age=0”。】</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 啊，看来是头字段太多，连我自己有时候也会弄糊涂，非常抱歉。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-17 04:08:39</div>
  </div>
</div>
</div>
</li>
</ul>