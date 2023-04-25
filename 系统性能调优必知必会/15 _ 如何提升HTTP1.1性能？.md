<audio title="15 _ 如何提升HTTP1.1性能？" src="https://static001.geekbang.org/resource/audio/71/09/71766c707c12801cc505d62333674409.mp3" controls="controls"></audio> 
<p>你好，我是陶辉。</p><p>上一讲介绍了为应用层信息安全保驾护航的TLS/SSL协议，这一讲我们来看看最常用的应用层协议HTTP/1.1该如何优化。</p><p>由于门槛低、易监控、自表达等特点，HTTP/1.1在互联网诞生之初就成为最广泛使用的应用层协议。然而它的性能却很差，最为人诟病的是HTTP头部的传输占用了大量带宽。由于HTTP头部使用ASCII编码方式，这造成它往往达到几KB，而且滥用的Cookie头部进一步增大了体积。与此同时，REST架构的无状态特性还要求每个请求都得重传HTTP头部，这就使消息的有效信息比重难以提高。</p><p>你可能听说过诸如缓存、长连接、图片拼接、资源压缩等优化HTTP协议性能的方式，这些优化方案有些从底层的传输层优化入手，有些从用户使用浏览器的体验入手，有些则从服务器资源的编码入手，五花八门，导致我们没有系统化地优化思路，往往在性能提升上难尽全功。</p><p>那么，如何全面地提升HTTP/1.1协议的性能呢？我认为在不升级协议的情况下，有3种优化思路：首先是通过缓存避免发送HTTP请求；其次，如果不得不发起请求，那么就得思考如何才能减少请求的个数；最后则是减少服务器响应的体积。</p><p>接下来，我们就沿着这3个思路看看具体的优化方法。</p><!-- [[[read_end]]] --><h2>通过缓存避免发送HTTP请求</h2><p>如果不走网络就能获得HTTP响应，这样性能肯定最高。HTTP协议设计之初就考虑到了这一点，缓存能够让客户端在免于发送HTTP请求的情况下获得服务器的资源。</p><p>缓存到底是如何做到的呢？其实很简单，它从时间维度上做文章，把第1份请求及其响应保存在客户端的本地磁盘上，其中请求的URL作为关键字（部分HTTP头部也会加入关键字，例如确定服务器域名的Host头部），而响应就是值。这样，后续发起相同的请求时，就可以先在本地磁盘上通过关键字查找，如果找到了，就直接将缓存作为服务器响应使用。读取本地磁盘耗时不过几十毫秒，这远比慢了上百倍且不稳定的网络请求快得多。</p><p><img src="https://static001.geekbang.org/resource/image/9d/ab/9dea133d832d8b7ab642bb74b48502ab.png?wh=1007*676" alt=""></p><p>你可能会问，服务器上的资源更新后，客户端并不知道，它若再使用过期的缓存就会出错，这该如何解决？因此，服务器会在发送响应时预估一个过期时间并在响应头部中告诉客户端，而客户端一旦发现缓存过期则重新发起网络请求。HTTP协议控制缓存过期的头部非常多，而且通常这是在服务器端设置的，我会在本专栏的第4部分“分布式系统优化”结合服务器操作再来介绍，这里暂时略过。</p><p>当然，过期的缓存也仍然可以提升性能，如下图所示，当客户端发现缓存过期后，会取出缓存的摘要（摘要是从第1次请求的响应中拿到的），把它放在请求的Etag头部中再发给服务器。而服务器获取到请求后，会将本地资源的摘要与请求中的Etag相比较，如果不同，那么缓存没有价值，重新发送最新资源即可；如果摘要与Etag相同，那么仅返回不含有包体的304 Not Modified响应，告知客户端缓存仍然有效即可，这就省去传递可能高达千百兆的文件资源。</p><p><img src="https://static001.geekbang.org/resource/image/a3/62/a394b7389d0cdb5f0866223681e19b62.png?wh=1048*696" alt=""></p><p>至于Etag摘要究竟是怎样生成的，各类Web服务器的处理方式不尽相同，比如Nginx会将文件大小和修改时间拼接为一个字符串作为文件摘要（详见<a href="https://time.geekbang.org/course/detail/138-76358">《Nginx核心知识100讲》第97课</a>）。过期缓存在分布式系统中可以有效提升可用性，[第25课] 还会站在反向代理的角度介绍过期缓存的用法。</p><p>浏览器上的缓存只能为一个用户使用，故称为私有缓存。代理服务器上的缓存可以被许多用户使用，所以称为共享缓存。可见，共享缓存带来的性能收益被庞大的客户端群体放大了。你可以看到在下面的REST架构图中，表示缓存的$符号（缓存的英文名称是cache，由于它的发音与cash很像，所以许多英文文档中用美元符号来表示缓存）既存在于User Agent浏览器中，也存在于Proxy正向代理服务器和Gateway反向代理上。</p><p><img src="https://static001.geekbang.org/resource/image/b6/ff/b6950aa62483c56438c41bc4b2f43bff.png?wh=748*334" alt="" title="图片来自网络"></p><p>可见，缓存与互联网世界的网络效率密切相关，用好缓存是提升HTTP性能最重要的手段。</p><h2>如何降低HTTP请求的次数？</h2><p>如果不得不发起请求，就应该尽量减少HTTP请求的次数，这可以从减少重定向次数、合并请求、延迟发送请求等3个方面入手。</p><p>首先来看看什么是重定向请求，一个资源由于迁移、维护等原因从url1移至url2后，在客户端访问原先的url1时，服务器不能简单粗暴地返回错误，而是通过302响应码及Location头部，告诉客户端资源已经改到url2了，而客户端再用url2发起新的HTTP请求。</p><p>可见，重定向增加了请求的数量。尤其客户端在公网中时，由于公网速度慢、成本高、路径长、不稳定，而且为了信息安全性还要用TLS协议加密，这些都降低了网络性能。从上面的REST架构图可以看到，HTTP请求会经过多个代理服务器，如果将重定向工作交由代理服务器完成，就能减少网络消耗，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/0d/2a/0d1701737956ce65f3d5ec8fa8009f2a.png?wh=724*842" alt=""></p><p>更进一步，客户端还可以缓存重定向响应。RFC规范定义了5个重定向响应码（如下表所示），其中客户端接收到301和308后都可以将重定向响应缓存至本地，之后客户端会自动用url2替代url1访问网络资源。</p><p><img src="https://static001.geekbang.org/resource/image/85/70/85b55f50434fc1acd2ead603e5c57870.jpg?wh=2324*712" alt=""></p><p>其次我们来看如何合并请求。当多个访问小文件的请求被合并为一个访问大文件的请求时，这样虽然传输的总资源体积未变，但减少请求就意味着减少了重复发送的HTTP头部，同时也减少了TCP连接的数量，因而省去了TCP握手和慢启动过程消耗的时间（参见第12课）。我们具体看几种合并请求的方式。</p><p>一些WEB页面往往含有几十、上百个小图片，用<a href="https://www.tutorialrepublic.com/css-tutorial/css-sprites.php">CSS Image Sprites技术</a>可以将它们合成一张大图片，而浏览器获得后可以根据CSS数据把它切割还原为多张小图片，这可以大幅减少网络消耗。</p><p><a href="https://blog.csdn.net/weixin_38055381/article/details/81504716"><img src="https://static001.geekbang.org/resource/image/e4/ed/e4a760a95542701d99b8a38ccddaebed.png?wh=1502*628" alt="" title="图片来自[黑染枫林的CSDN博客]"></a></p><p>类似地，在服务器端用webpack等打包工具将Javascript、CSS等资源合并为大文件，也能起到同样的效果。</p><p>除此以外，还可以将多媒体资源用base64编码后，以URL的方式嵌入HTML文件中，以减少小请求的个数（参见<a href="https://tools.ietf.org/html/rfc2397">RFC2397</a>）。</p><p><img src="https://static001.geekbang.org/resource/image/27/fb/27aa119e0104f41a6718987cdf71c8fb.png?wh=1299*287" alt=""></p><p>用Chrome浏览器开发者工具的Network面板可以轻松地判断各站点是否使用了这种技术。关于Network面板的用法，我们在此就不赘述了（可参考<a href="https://time.geekbang.org/course/detail/175-93594">《Web协议详解与抓包实战》第9课</a>，那里有详细的介绍）。</p><p><img src="https://static001.geekbang.org/resource/image/ab/74/ab75920b0c43e4f7fcbc5933005afb74.png?wh=534*700" alt=""></p><p>当然，这种合并请求的方式也会带来一个新问题，即，当其中一个资源发生变化后，客户端必须重新下载完整的大文件，这显然会带来额外的网络消耗。在落后的HTTP/1.1协议中，合并请求还算一个不错的解决方案，在下一讲将介绍的HTTP/2出现后，这种技术就没有用武之地了。</p><p>最后，我们还可以从浏览页面的体验角度上，减少HTTP请求的次数。比如，有些HTML页面上引用的资源，其实在当前页面上用不上，而是供后续页面使用的，这就可以使用懒加载（<a href="https://zh.wikipedia.org/wiki/%E6%83%B0%E6%80%A7%E8%BC%89%E5%85%A5">lazy loading</a>） 技术延迟发起请求。</p><p>当不得不发起请求时，还可以从服务器的角度通过减少响应包体的体积来提升性能。</p><h2>如何重新编码以减少响应的大小？</h2><p>减少资源体积的唯一方式是对资源重新编码压缩，其中又分为<a href="https://zh.wikipedia.org/wiki/%E6%97%A0%E6%8D%9F%E6%95%B0%E6%8D%AE%E5%8E%8B%E7%BC%A9">无损压缩</a>与<a href="https://zh.wikipedia.org/wiki/%E6%9C%89%E6%8D%9F%E6%95%B0%E6%8D%AE%E5%8E%8B%E7%BC%A9">有损压缩</a>两种。</p><p>先来看无损压缩，这是指压缩后不会损失任何信息，可以完全恢复到压缩前的原样。因此，文本文件、二进制可执行文件都会使用这类压缩方法。</p><p>源代码也是文本文件，但它有自身的语法规则，所以可以依据语法先做一轮压缩。比如<a href="https://jquery.com/download/">jQuery</a> 是用javascript语言编写的库，而标准版的jQuery.js为了帮助程序员阅读含有许多空格、回车等符号，但机器解释执行时并不需要这些符号。因此，根据语法规则将这些多余的符号去除掉，就可以将jQuery文件的体积压缩到原先的三分之一。</p><p><img src="https://static001.geekbang.org/resource/image/de/ba/de0fc2a64a100bc83fc48eb2606fedba.png?wh=1257*218" alt=""></p><p>接着可以基于信息熵原理进行通用的无损压缩，这需要对原文建立统计模型，将出现频率高的数据用较短的二进制比特序列表示，而将出现频率低的数据用较长的比特序列表示。我们最常见的Huffman算法就是一种执行速度较快的实践，在下一讲的HTTP/2协议中还会用到它。在上图中可以看到，最小版的jQuery文件经过Huffman等算法压缩后，体积还会再缩小三分之二。</p><p>支持无损压缩算法的客户端会在请求中通过Accept-Encoding头部明确地告诉服务器：</p><pre><code>Accept-Encoding: gzip, deflate, br
</code></pre><p>而服务器也会在响应的头部中，告诉客户端包体中的资源使用了何种压缩算法（Nginx开启资源压缩的方式参见《Nginx核心知识100讲》<a href="https://time.geekbang.org/course/detail/138-79618">第131课</a>和<a href="https://time.geekbang.org/course/detail/138-79621">第134课</a>）：</p><pre><code>content-encoding: gzip
</code></pre><p>虽然目前gzip使用最为广泛，但它的压缩效率以及执行速度其实都很一般，Google于2015年推出的<a href="https://zh.wikipedia.org/wiki/Brotli">Brotli</a> 算法在这两方面表现都更优秀（也就是上文中的br），其对比数据如下：</p><p><a href="https://quixdb.github.io/squash-benchmark/"><img src="https://static001.geekbang.org/resource/image/62/cb/62e01433ad8ef23ab698e7c47b7cc8cb.png?wh=1343*813" alt="" title="图片来源：[https://quixdb.github.io/squash-benchmark/]"></a></p><p>再来看有损压缩，它通过牺牲质量来提高压缩比，主要针对的是图片和音视频。HTTP请求可以通过Accept头部中的q质量因子（参见<a href="https://tools.ietf.org/html/rfc7231#section-5.3.2">RFC7231</a>），告诉服务器期望的资源质量：</p><pre><code>Accept: audio/*; q=0.2, audio/basic
</code></pre><p>先来看图片的压缩。目前压缩比较高的开源算法是Google在2010年推出的<a href="https://zh.wikipedia.org/wiki/WebP">WebP格式</a>，你可以在<a href="https://isparta.github.io/compare-webp/index.html#12345">这个页面</a>看到它与png格式图片的对比图。对于大量使用图片的网站，使用它代替传统格式会有显著的性能提升。</p><p>动态的音视频压缩比要比表态的图片高很多！由于音视频数据有时序关系，且时间连续的帧之间变化很小，因此可以在静态的关键帧之后，使用增量数据表达后续的帧，因此在质量略有损失的情况下，音频体积可以压缩到原先的几十分之一，视频体积则可以压缩到几百分之一，比图片的压缩比要高得多。因此，对音视频做有损压缩，能够大幅提升网络传输的性能。</p><p>对响应资源做压缩不只用于HTTP/1.1协议，事实上它对任何信息传输场景都有效，消耗一些CPU计算力在事前或者事中做压缩，通常会给性能带来不错的提升。</p><h2>小结</h2><p>这一讲我们从三个方面介绍了HTTP/1.1协议的优化策略。</p><p>首先，客户端缓存响应，可以在有效期内避免发起HTTP请求。即使缓存过期后，如果服务器端资源未改变，仍然可以通过304响应避免发送包体资源。浏览器上的私有缓存、服务器上的共享缓存，都对HTTP协议的性能提升有很大意义。</p><p>其次是降低请求的数量，如将原本由客户端处理的重定向请求，移至代理服务器处理可以减少重定向请求的数量。或者从体验角度，使用懒加载技术延迟加载部分资源，也可以减少请求数量。再比如，将多个文件合并后再传输，能够少传许多HTTP头部，而且减少TCP连接数量后也省去握手和慢启动的消耗。当然合并文件的副作用是小文件的更新，会导致整个合并后的大文件重传。</p><p>最后可以通过压缩响应来降低传输的字节数，选择更优秀的压缩算法能够有效降低传输量，比如用Brotli无损压缩算法替换gzip，或者用WebP格式替换png等格式图片等。</p><p>但其实在HTTP/1.1协议上做优化效果总是有限的，下一讲我们还将介绍在URL、头部等高层语法上向前兼容的HTTP/2协议，它在性能上有大幅度提升，是如gRPC等应用层协议的基础。</p><h2>思考题</h2><p>除了我今天介绍的方法以外，使用KeepAlive长连接替换短连接也能提升性能，你还知道有哪些提升HTTP/1.1性能的方法吗？欢迎你在留言区与我一起探讨。</p><p>感谢阅读，如果你觉得这节课对你有一些启发，也欢迎把它分享给你的朋友。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/63/2e/e49116d1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_007</span>
  </div>
  <div class="_2_QraFYR_0">FB也搞了一套压缩算法ZSTD，对比起来也比gzip性能强很多，不清楚这些压缩算法的原理是啥，怎么对比，希望老师之后能答疑一下。另外普通的json和pb有不同适合的压缩算法吗？怎么比较呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你好Geek_007，这3个问题我的看法如下：<br>1、压缩算法的原理都是基于香农的信息论，将高频出现的信息用更少的比特编码。虽然原理是一致的，但实现上却有很大的差别，比如Huffman通过建立Huffman树来生成编码，而LZ77却是通过滑动窗口，这造成压缩比、压缩速度都很不相同。<br>2、比较它们的优劣，主要看3个指标：<br>a. 压缩比，比如brotli的压缩比好于ZSTD。<br>b. 压缩与解压的速度，比如ZSTD比gzip速度快<br>c. 浏览器等中间件的支持程度，现在几乎所有浏览器都支持brotli（即br），但ZSTD少有支持<br>3、没有最适合算法，对具体的场景，比较方法参见第2点</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-01 23:08:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/63/2e/e49116d1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_007</span>
  </div>
  <div class="_2_QraFYR_0">图片压缩，腾讯说他们的TPG比Webp压缩的更小呢😀，不过除了他们内部这么说，没见过第三方资料的证明</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 关键还是看浏览器对格式的支持程度。目前，IE、Firefox、Chrome等主流浏览器都支持webp，但TPG就不好说了，除非产品用户主要在使用 QQ浏览器^_^</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-01 04:15:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/9f/3d/b400b98e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>新声带NewVoice</span>
  </div>
  <div class="_2_QraFYR_0">尝试过http1.1 chunk子协议，可以在同一个连接上，服务端向客户端发送多次数据。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 没错，有些消息推送是用chunk实现的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-08 18:21:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/3XbCueYYVWTiclv8T5tFpwiblOxLphvSZxL4ujMdqVMibZnOiaFK2C5nKRGv407iaAsrI0CDICYVQJtiaITzkjfjbvrQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>有铭</span>
  </div>
  <div class="_2_QraFYR_0">Accept 头部中的 q 质量因子<br>======<br>想问一下，当服务器接收到这样的请求后，是背后的应用层处理这个压缩，还是说，支持这个标准的Http服务器本身就可以“透明”的完成这个压缩工作</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-01 15:33:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/85/49/585c69c4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>皮特尔</span>
  </div>
  <div class="_2_QraFYR_0">最近正准备用gzip优化我们的一个后端服务就看到了这一讲，赶快试试Brotli算法</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 在实践中学习，赞！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-01 12:35:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/8f/cf/890f82d6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>那时刻</span>
  </div>
  <div class="_2_QraFYR_0">一般来说客户端和代理服务器之间是https协议，而代理服务器和源站之间可以只使用http协议，减少https协议额外的消息交互。<br>另外在减少http包体方面，老师文中提到图片压缩，而对于非图片的传输，如定义的json结构，可以通过简化json里key，value的字节数来减少传输包体大小。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-01 10:23:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/37/a0/032d0828.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>上杉夏香</span>
  </div>
  <div class="_2_QraFYR_0">如果上述方法都采用了。那么只能提高HTTP请求的并发度了。<br>1、对于客户端来说，多开TCP连接的数量。<br>2、对于服务器来说，可以将相同域名解析到不同的IP，然后提供同样的服务。<br>本质上，都是为了增加TCP连接的数量。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-03 18:56:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/37/a0/032d0828.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>上杉夏香</span>
  </div>
  <div class="_2_QraFYR_0">如果上述方法都尝试过。那么只能提高http请求的并发度了。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-03 18:54:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/39/fa/a7edbc72.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>安排</span>
  </div>
  <div class="_2_QraFYR_0">base64编码的作用是啥呢？只是为了把不可见字符变成可见的吗？搜到的资料都讲base64的实现原理但是并没有讲base64编码的作用的，好像挺多地方用到这个编码了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，就是为了转换可见字符，这样更方便交换</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-01 07:38:31</div>
  </div>
</div>
</div>
</li>
</ul>