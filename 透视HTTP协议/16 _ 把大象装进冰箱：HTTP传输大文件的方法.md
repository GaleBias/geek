<audio title="16 _ 把大象装进冰箱：HTTP传输大文件的方法" src="https://static001.geekbang.org/resource/audio/ba/6a/ba4551c47d8a72e53add314d06f3b46a.mp3" controls="controls"></audio> 
<p>上次我们谈到了HTTP报文里的body，知道了HTTP可以传输很多种类的数据，不仅是文本，也能传输图片、音频和视频。</p><p>早期互联网上传输的基本上都是只有几K大小的文本和小图片，现在的情况则大有不同。网页里包含的信息实在是太多了，随随便便一个主页HTML就有可能上百K，高质量的图片都以M论，更不要说那些电影、电视剧了，几G、几十G都有可能。</p><p>相比之下，100M的光纤固网或者4G移动网络在这些大文件的压力下都变成了“小水管”，无论是上传还是下载，都会把网络传输链路挤的“满满当当”。</p><p>所以，如何在有限的带宽下高效快捷地传输这些大文件就成了一个重要的课题。这就好比是已经打开了冰箱门（建立连接），该怎么把大象（文件）塞进去再关上门（完成传输）呢？</p><p>今天我们就一起看看HTTP协议里有哪些手段能解决这个问题。</p><h2>数据压缩</h2><p>还记得上一讲中说到的“数据类型与编码”吗？如果你还有印象的话，肯定能够想到一个最基本的解决方案，那就是“<strong>数据压缩</strong>”，把大象变成小猪佩奇，再放进冰箱。</p><p>通常浏览器在发送请求时都会带着“<strong>Accept-Encoding</strong>”头字段，里面是浏览器支持的压缩格式列表，例如gzip、deflate、br等，这样服务器就可以从中选择一种压缩算法，放进“<strong>Content-Encoding</strong>”响应头里，再把原数据压缩后发给浏览器。</p><!-- [[[read_end]]] --><p>如果压缩率能有50%，也就是说100K的数据能够压缩成50K的大小，那么就相当于在带宽不变的情况下网速提升了一倍，加速的效果是非常明显的。</p><p>不过这个解决方法也有个缺点，gzip等压缩算法通常只对文本文件有较好的压缩率，而图片、音频视频等多媒体数据本身就已经是高度压缩的，再用gzip处理也不会变小（甚至还有可能会增大一点），所以它就失效了。</p><p>不过数据压缩在处理文本的时候效果还是很好的，所以各大网站的服务器都会使用这个手段作为“保底”。例如，在Nginx里就会使用“gzip on”指令，启用对“text/html”的压缩。</p><h2>分块传输</h2><p>在数据压缩之外，还能有什么办法来解决大文件的问题呢？</p><p>压缩是把大文件整体变小，我们可以反过来思考，如果大文件整体不能变小，那就把它“拆开”，分解成多个小块，把这些小块分批发给浏览器，浏览器收到后再组装复原。</p><p>这样浏览器和服务器都不用在内存里保存文件的全部，每次只收发一小部分，网络也不会被大文件长时间占用，内存、带宽等资源也就节省下来了。</p><p>这种“<strong>化整为零</strong>”的思路在HTTP协议里就是“<strong>chunked</strong>”分块传输编码，在响应报文里用头字段“<strong>Transfer-Encoding: chunked</strong>”来表示，意思是报文里的body部分不是一次性发过来的，而是分成了许多的块（chunk）逐个发送。</p><p>这就好比是用魔法把大象变成“乐高积木”，拆散了逐个装进冰箱，到达目的地后再施法拼起来“满血复活”。</p><p>分块传输也可以用于“流式数据”，例如由数据库动态生成的表单页面，这种情况下body数据的长度是未知的，无法在头字段“<strong>Content-Length</strong>”里给出确切的长度，所以也只能用chunked方式分块发送。</p><p>“Transfer-Encoding: chunked”和“Content-Length”这两个字段是<strong>互斥的</strong>，也就是说响应报文里这两个字段不能同时出现，一个响应报文的传输要么是长度已知，要么是长度未知（chunked），这一点你一定要记住。</p><p>下面我们来看一下分块传输的编码规则，其实也很简单，同样采用了明文的方式，很类似响应头。</p><ol>
<li>每个分块包含两个部分，长度头和数据块；</li>
<li>长度头是以CRLF（回车换行，即\r\n）结尾的一行明文，用16进制数字表示长度；</li>
<li>数据块紧跟在长度头后，最后也用CRLF结尾，但数据不包含CRLF；</li>
<li>最后用一个长度为0的块表示结束，即“0\r\n\r\n”。</li>
</ol><p>听起来好像有点难懂，看一下图就好理解了：</p><p><img src="https://static001.geekbang.org/resource/image/25/10/25e7b09cf8cb4eaebba42b4598192410.png?wh=3000*1681" alt=""></p><p>实验环境里的URI“/16-1”简单地模拟了分块传输，可以用Chrome访问这个地址看一下效果：</p><p><img src="https://static001.geekbang.org/resource/image/e1/db/e183bf93f4759b74c8ee974acbcaf9db.png?wh=3000*1681" alt=""></p><p>不过浏览器在收到分块传输的数据后会自动按照规则去掉分块编码，重新组装出内容，所以想要看到服务器发出的原始报文形态就得用Telnet手工发送请求（或者用Wireshark抓包）：</p><pre><code>GET /16-1 HTTP/1.1
Host: www.chrono.com
</code></pre><p>因为Telnet只是收到响应报文就完事了，不会解析分块数据，所以可以很清楚地看到响应报文里的chunked数据格式：先是一行16进制长度，然后是数据，然后再是16进制长度和数据，如此重复，最后是0长度分块结束。</p><p><img src="https://static001.geekbang.org/resource/image/66/02/66a6d229058c7072ab5b28ef518da302.png?wh=3000*1681" alt=""></p><h2>范围请求</h2><p>有了分块传输编码，服务器就可以轻松地收发大文件了，但对于上G的超大文件，还有一些问题需要考虑。</p><p>比如，你在看当下正热播的某穿越剧，想跳过片头，直接看正片，或者有段剧情很无聊，想拖动进度条快进几分钟，这实际上是想获取一个大文件其中的片段数据，而分块传输并没有这个能力。</p><p>HTTP协议为了满足这样的需求，提出了“<strong>范围请求</strong>”（range requests）的概念，允许客户端在请求头里使用专用字段来表示只获取文件的一部分，相当于是<strong>客户端的“化整为零”</strong>。</p><p>范围请求不是Web服务器必备的功能，可以实现也可以不实现，所以服务器必须在响应头里使用字段“<strong>Accept-Ranges: bytes</strong>”明确告知客户端：“我是支持范围请求的”。</p><p>如果不支持的话该怎么办呢？服务器可以发送“Accept-Ranges: none”，或者干脆不发送“Accept-Ranges”字段，这样客户端就认为服务器没有实现范围请求功能，只能老老实实地收发整块文件了。</p><p>请求头<strong>Range</strong>是HTTP范围请求的专用字段，格式是“<strong>bytes=x-y</strong>”，其中的x和y是以字节为单位的数据范围。</p><p>要注意x、y表示的是“偏移量”，范围必须从0计数，例如前10个字节表示为“0-9”，第二个10字节表示为“10-19”，而“0-10”实际上是前11个字节。</p><p>Range的格式也很灵活，起点x和终点y可以省略，能够很方便地表示正数或者倒数的范围。假设文件是100个字节，那么：</p><ul>
<li>“0-”表示从文档起点到文档终点，相当于“0-99”，即整个文件；</li>
<li>“10-”是从第10个字节开始到文档末尾，相当于“10-99”；</li>
<li>“-1”是文档的最后一个字节，相当于“99-99”；</li>
<li>“-10”是从文档末尾倒数10个字节，相当于“90-99”。</li>
</ul><p>服务器收到Range字段后，需要做四件事。</p><p>第一，它必须检查范围是否合法，比如文件只有100个字节，但请求“200-300”，这就是范围越界了。服务器就会返回状态码<strong>416</strong>，意思是“你的范围请求有误，我无法处理，请再检查一下”。</p><p>第二，如果范围正确，服务器就可以根据Range头计算偏移量，读取文件的片段了，返回状态码“<strong>206 Partial Content</strong>”，和200的意思差不多，但表示body只是原数据的一部分。</p><p>第三，服务器要添加一个响应头字段<strong>Content-Range</strong>，告诉片段的实际偏移量和资源的总大小，格式是“<strong>bytes x-y/length</strong>”，与Range头区别在没有“=”，范围后多了总长度。例如，对于“0-10”的范围请求，值就是“bytes 0-10/100”。</p><p>最后剩下的就是发送数据了，直接把片段用TCP发给客户端，一个范围请求就算是处理完了。</p><p>你可以用实验环境的URI“/16-2”来测试范围请求，它处理的对象是“/mime/a.txt”。不过我们不能用Chrome浏览器，因为它没有编辑HTTP请求头的功能（这点上不如Firefox方便），所以还是要用Telnet。</p><p>例如下面的这个请求使用Range字段获取了文件的前32个字节：</p><pre><code>GET /16-2 HTTP/1.1
Host: www.chrono.com
Range: bytes=0-31
</code></pre><p>返回的数据是（去掉了几个无关字段）：</p><pre><code>HTTP/1.1 206 Partial Content
Content-Length: 32
Accept-Ranges: bytes
Content-Range: bytes 0-31/96

// this is a plain text json doc
</code></pre><p>有了范围请求之后，HTTP处理大文件就更加轻松了，看视频时可以根据时间点计算出文件的Range，不用下载整个文件，直接精确获取片段所在的数据内容。</p><p>不仅看视频的拖拽进度需要范围请求，常用的下载工具里的多段下载、断点续传也是基于它实现的，要点是：</p><ul>
<li>先发个HEAD，看服务器是否支持范围请求，同时获取文件的大小；</li>
<li>开N个线程，每个线程使用Range字段划分出各自负责下载的片段，发请求传输数据；</li>
<li>下载意外中断也不怕，不必重头再来一遍，只要根据上次的下载记录，用Range请求剩下的那一部分就可以了。</li>
</ul><h2>多段数据</h2><p>刚才说的范围请求一次只获取一个片段，其实它还支持在Range头里使用多个“x-y”，一次性获取多个片段数据。</p><p>这种情况需要使用一种特殊的MIME类型：“<strong>multipart/byteranges</strong>”，表示报文的body是由多段字节序列组成的，并且还要用一个参数“<strong>boundary=xxx</strong>”给出段之间的分隔标记。</p><p>多段数据的格式与分块传输也比较类似，但它需要用分隔标记boundary来区分不同的片段，可以通过图来对比一下。</p><p><img src="https://static001.geekbang.org/resource/image/ff/37/fffa3a65e367c496428f3c0c4dac8a37.png?wh=3000*1681" alt=""></p><p>每一个分段必须以“- -boundary”开始（前面加两个“-”），之后要用“Content-Type”和“Content-Range”标记这段数据的类型和所在范围，然后就像普通的响应头一样以回车换行结束，再加上分段数据，最后用一个“- -boundary- -”（前后各有两个“-”）表示所有的分段结束。</p><p>例如，我们在实验环境里用Telnet发出有两个范围的请求：</p><pre><code>GET /16-2 HTTP/1.1
Host: www.chrono.com
Range: bytes=0-9, 20-29
</code></pre><p>得到的就会是下面这样：</p><pre><code>HTTP/1.1 206 Partial Content
Content-Type: multipart/byteranges; boundary=00000000001
Content-Length: 189
Connection: keep-alive
Accept-Ranges: bytes


--00000000001
Content-Type: text/plain
Content-Range: bytes 0-9/96

// this is
--00000000001
Content-Type: text/plain
Content-Range: bytes 20-29/96

ext json d
--00000000001--
</code></pre><p>报文里的“- -00000000001”就是多段的分隔符，使用它客户端就可以很容易地区分出多段Range 数据。</p><h2>小结</h2><p>今天我们学习了HTTP传输大文件相关的知识，在这里做一下简单小结：</p><ol>
<li><span class="orange">压缩HTML等文本文件是传输大文件最基本的方法；</span></li>
<li><span class="orange">分块传输可以流式收发数据，节约内存和带宽，使用响应头字段“Transfer-Encoding: chunked”来表示，分块的格式是16进制长度头+数据块；</span></li>
<li><span class="orange">范围请求可以只获取部分数据，即“分块请求”，实现视频拖拽或者断点续传，使用请求头字段“Range”和响应头字段“Content-Range”，响应状态码必须是206；</span></li>
<li><span class="orange">也可以一次请求多个范围，这时候响应报文的数据类型是“multipart/byteranges”，body里的多个部分会用boundary字符串分隔。</span></li>
</ol><p>要注意这四种方法不是互斥的，而是可以混合起来使用，例如压缩后再分块传输，或者分段后再分块，实验环境的URI“/16-3”就模拟了后一种的情形，你可以自己用Telnet试一下。</p><h2>课下作业</h2><ol>
<li>分块传输数据的时候，如果数据里含有回车换行（\r\n）是否会影响分块的处理呢？</li>
<li>如果对一个被gzip的文件执行范围请求，比如“Range: bytes=10-19”，那么这个范围是应用于原文件还是压缩后的文件呢？</li>
</ol><p>欢迎你把自己的学习体会写在留言区，与我和其他同学一起讨论。如果你觉得有所收获，也欢迎把文章分享给你的朋友。</p><p><img src="https://static001.geekbang.org/resource/image/ab/71/ab951899844cef3d1e029ba094c2eb71.png?wh=1769*2881" alt="unpreview"></p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/61/c1/93031a2a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Aaaaaaaaaaayou</span>
  </div>
  <div class="_2_QraFYR_0">老师，有个问题：http交给tcp进行传输的时候本来就会分块，那http分块的意义是什么呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 在http层是看不到tcp的，它不知道下层协议是否会分块，下层是否分块对它来说没有意义，不关心。<br><br>在http里一个报文必须是完整交付，在处理大文件的时候就很不方便，所以就要分块，在http层面方便处理。<br><br>chunked主要是在http的层次来解决问题。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-03 08:47:58</div>
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
  <div class="_2_QraFYR_0">1. 分块传输中数据里含有回车换行（\r\n）不影响分块处理，因为分块前有数据长度说明<br>2. 范围是应用于压缩后的文件</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1正确。<br><br>2需要分情况，看原文件是什么形式。如果原来的文件是gzip的，那就正确。如果原文件是文本，而是在传输过程中被压缩，那么就应用于压缩前的数据。<br><br>总之，range是针对原文件的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-03 11:26:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/3a/d6/485590bd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>赵健</span>
  </div>
  <div class="_2_QraFYR_0">“Transfer-Encoding: chunked”和“Content-Length”这两个字段是互斥的，也就是说响应报文里这两个字段不能同时出现，一个响应报文的传输要么是长度已知，要么是长度未知（chunked），这一点你一定要记住。老师请问下，为啥分块意味着长度未知，后面不是提到块里面有个长度头嘛？而且单个块应该是一次http传输的内容，既然块里有长度头，那这次传输的内容长度也就能算出来，这次http的Content-Length 也就知道啊！是我理解错了吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 举个例子，从GitHub上下载源码包，GitHub要实时压缩实时发送，而不是一下子压缩好再发送，这样body的长度一开始就是未知的。<br><br>所以就要用分块编码，压缩一部分，就发一部分，这部分的长度是已知的，但总长度只有压缩完才能知道。<br><br>chunked编码用在“流式”收发数据的时候，通常数据是即时生成的，也就是动态数据。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-04 00:01:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/c9/9e/ce7c8522.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>秋水共长天一色🌄</span>
  </div>
  <div class="_2_QraFYR_0">老师，我有些问题需要问问您。<br>1.比如我在视频网上看电影，我们经常能看到进度条里面有一条灰色的缓存进度，我是否能理解成这个进度就是分块传输的一个进度显示吗？<br>2.刚刚我有看到评论说过一个问题就是分块传输的时候是由一个请求和一个响应完成的，如果我们在抓一个需要10分钟才能完成分块传输的请求时，我是不是就会看到这个请求在这10分钟内都是一个正在响应的状态吗？<br>3.为什么我们在对一些视频网站看视频抓包的时候却无法捕抓到这个请求呢？<br>4.如果我们在看完视频后在浏览器缓存里发现一些片段式的视频文件，能否就说明这个是用分块传输呢？<br>5.如果我们在看视频拖动进度条到10分30秒，到最后视频会从10分20秒开始播放，能否说明10分30秒的这个分块的头是在10分20秒呢？<br>6.请问多段数据能理解成一次性获取分块传输里多个连续的分块的数据的意思吗？<br>还有就是非常感谢老师把这些知识点讲的那么细，我近期多个面试里都有被问到相关的知识，多亏老师的讲解我才能顺利应付，谢谢老师！！！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.网络视频不一定用的就是http协议，也可能是其他的专用协议，所以不能简单地判断就是分块传输。如果是http协议，对于大文件通常都是range请求，也不一定用chunk分块。<br><br>2.是的，这是由http的请求-应答工作模式决定的。不管是不是chunk，只要响应没有结束，这个来回就不会完成。<br><br>3.也可能用的是其他协议。<br><br>4.应该是range的分段，不是chunk，chunk只是在传输过程中分块，最后到客户端会是一个完整的文件。<br><br>5.视频文件的分段计算比较复杂，课程里面只是作为简单的示例，实际情况不一定就是这么简单。<br><br>6.分段的range和分块的chunk是两个完全无关的概念，不要弄混了。chunk是传输时分成小块逐个发送，range是取大文件中间的一部分。<br><br>7.不客气，后面的https、http&#47;2也继续努力吧。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-29 21:49:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/11/b9/abb7bfe3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小桶</span>
  </div>
  <div class="_2_QraFYR_0">分块传输，客户端只需要发一次请求，还是发多次请求呢？使用分块传输时，客户端与服务器是怎样工作的呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: http传输永远是一个请求一个响应的工作模式，只是响应是chunked分块，body数据不是一次性发过来的，而是分批分块发送，但仍然是在一个报文里。<br><br>客户端发送请求后等待响应，服务器组织数据，分块发送，最后一个分块是结束标志。客户端依次接收分块，收到结束标志后就把数据拼成完整的报文。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-20 16:54:04</div>
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
  <div class="_2_QraFYR_0">对于问题2,range是针对原文的还是压缩后的，可以想象一下看视频的时候，我们拖拽进度条请求的range范围是针对原视频长度的，如果针对压缩后的，那么我们实际拖拽的范围和响应的数据范围就不一致了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 说的很好。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-04 09:08:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/93/5d/91f1d849.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>darren</span>
  </div>
  <div class="_2_QraFYR_0">不分块：http把客户端需要的东西整个交给tcp，由tcp切块后发送给客户端，客户端接受后在tcp层组装完整发给浏览器使用。<br>分块：http把客户端需要的东西切分成1、2、3到n块，然后将1块发给tcp，tcp将块1再次切分后发给客户端，客户端接受后在tcp组装成块1发给http层。然后服务器与客户端用同样的方式发送块2、块3到块n。客户端的http在接收完所有块后组装成一个完整的响应。整个过程使用同一个tcp连接，块1到块n如上是挨个发送的。如果是http2，则基于多路复用技术块1到块n可以同时发送。所以分块抓包http只能抓到一个包，如果抓tcp的包，分不分块，都会抓到很多包。<br>分段：分段就是对某个资源的一部分进行请求（类似于把一个大文件切分成很多小文件，类似压缩中的分卷功能，然后客户端只对这些小文件中的一部分进行请求）<br>分段是对需要哪些资源进行一种说明，分块是一种传输机制，完全不同的两个东西，只是名字比较像。<br>请老师指教理解不正确的地方，另外想问一下老师分块的时候每个块都会复制一次响应头吗，还是只有块1带有请求头。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 总结的非常好，很清晰。<br><br>chunk数据实际上是一个很大的请求，只有一个header，只是逐个地发出去，而不是一次性发。<br><br>可以在实验环境里抓包看看。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-07 11:17:37</div>
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
  <div class="_2_QraFYR_0">老师好!在带宽固定的情况下，范围请求没发提高下载速度。如果服务器对客户端每个累链接限速的情况下，可通过多线程并发下载，提高下载速度是么?还有几个问题<br>分块传输:顺序传一次一小块<br>范围请求:支持跳跃式传输，还可以并发获取不同的range最后合并。<br>多段数据:一次请求多个范围，范围可以不连续是么?如果必须联系的话和请求一个大范围没差别了。<br>这几个拒的例子都是服务端这么返回的。<br>客户端上传的时候怎么使用呢?老师后面会讲么。<br>只读到了这么点，希望老师补充下每个的作用，和解决的问题，谢谢老师。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1，是的，通过多线程并发下载，提高下载速度。<br><br>2，范围可以不连续，例子里就是这样。<br><br>3，客户端上传的时候也可以用chunked、gzip，但不能用range。<br><br>注意这些字段的类型，只要是实体字段，那就在请求响应里都可以用。 </p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-03 09:19:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/6b/b4/47c548fd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>一只鱼</span>
  </div>
  <div class="_2_QraFYR_0">针对课下作业2：<br>情况一：如果服务器上只有 gzip 之后的文件，没有原文件，那范围请求针对的就是 gizp 之后的文件；<br>情况二：如果服务器上有原文件(未压缩)，只是在传输过程中被 gizp , 那范围请求针对的就是未压缩的原文件。<br>这里拓展一下，假如在服务器和客户端之间有一个 cdn , 那么 cdn 缓存的是文件的某个范围吗？cdn 会根据请求头判断缓存里面有没有这个范围的结果，如果有就直接返回，并没有再根据bytes进行计算?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 说的对。<br><br>第二个问题，如果按标准http协议的做法就是只缓存一个范围，但这样的效果不会很好，因为缓存的这部分被重复利用的机会很小。<br><br>更好的方法是cdn返回给客户端的同时，把整个文件都从源站取下来，之后就可以自己计算范围给客户端，不用再从源站取了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-10 11:28:53</div>
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
  <div class="_2_QraFYR_0">老师，请问一下：“分块传输也可以用于“流式数据””。该怎么理解这个“流式数据”这句话呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 流式数据就是stream，虽然还是用一个个的chunk发过来的，但从更高层次上看，它像是从源头持续不断地、慢慢地“流”过来的，而不是一次性、一整块发过来的。<br><br>可以对比一下tcp和udp，tcp是流式数据，而udp是一个个的数据包。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-04 11:37:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/a3/fc/379387a4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ly</span>
  </div>
  <div class="_2_QraFYR_0">有几个小疑问：<br>分块传输：<br>对于一个500Mb的数据，客户端应该是发送N次http请求，每次http请求只传输其中一部分，每次都是采用了分块传输的body格式，那么每次都会重新建立TCP连接吗（三次握手）？<br><br>另外文章提到分块传输中的“流式数据”，这个流式数据怎么理解呢？<br><br>对于多段数据:<br>服务端在响应body里面的每一段都会指定Content-Type和Content-Range，总感觉其中的Content-Type字段是多余的，难道body里面的不同分段，Content-Type可能不一样？<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.请参考一下讲长连接的那一讲，现在的http请求应该都是使用的长连接，不会每次都重新建立连接，否则效率太低。<br><br>2.流式数据就是分块发送，像tcp那样一块一块地发过来，而不是像普通http那样一次完整地全发过来。<br><br>3.多段数据可以是mulitpart，每段数据是有可能不同的，比如发出一个请求，服务器返回了一个txt和json，所以要这么做。当然这种情况比较少见，但协议就必须要考虑到这种情况。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-18 20:00:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/96/b1/141bf83e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>wheat7</span>
  </div>
  <div class="_2_QraFYR_0">chunk的核心问题并不是所谓把大象装进冰箱，是为了解决应用层在没有content-length的时候知道数据在哪里结束，chunk和普通传输方式都是在一个http报文里传输的，只是在body里相当于又加了一层协议或者是编码，数据无论如何是在一头大象里，在一个http报文中传输，大的数据传输使用chunk和不使用传输方式并没有什么区别。 </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 说的很有道理。<br><br>不过两者还是有区别的，chunked主要用于流式数据，一开始不能知道准确的大小。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-12 01:25:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/51/cd/d6fe851f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Gopher</span>
  </div>
  <div class="_2_QraFYR_0">这个专栏质量很棒，老师很负责，知识讲解很通透，很容易就get、解惑了。<br><br>哈哈哈，特此留言就是想说，老师，你认真做事的样子真帅(*•̀ᴗ•́*)و ̑̑</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 承蒙夸奖，不胜感激。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-03 13:58:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/a6/84/92cb4db4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>四月长安</span>
  </div>
  <div class="_2_QraFYR_0">http数据包封装好交给下层tcp协议的时候，应该是作为tcp数据部分所要传输的内容吧，ip协议数据报最大传输65535字节的数据，这65535的数据减去tcp的首部，应该就是tcp所能容纳的负载极限了吧，所以如果是这样的话http数据的分块应该粒度更小才是吧，或许一个tcp负载里边就有好多http分块？请老师指正，感谢🙏</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: http层面的分块与tcp的分块是互不相关的，http的分块可以很大也可以很小，这些数据到了tcp由tcp去自行组织，是合并在一个块里还是跨越多个块都没有关系，我们不需要为此操心。<br><br>其实tcp层再往下的mac层，一个包最多1500字节，但tcp也不需要考虑这个，只要下层能够保证上层正确传输就好。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-16 01:00:33</div>
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
  <div class="_2_QraFYR_0">压缩和分而治之的思路是解决各种大问题的通用思路，思路容易理解，大东西放入小容器，要么把大东西变小也就是压缩，实在变小不了就弄多个容器把大东西分块放入多个容器之中，如果仅是传输之用那就一次传一点，有点类似愚公移山的动作。<br>比较关心大东西怎么拆？传输到对应的地方每一小块又怎么组合起来？比如：内存只有1G要传输10G的大文件，具体怎么拆分呢？是按照大小吗？比如：拆成20个0.5G的文件，如果这样传输到的内容也是需要保持一定的顺序的吧？否则组装也是一个问题，我能想到的最简单的方式就是表上号放置的时候按照序号一个个码放，不过我觉得应该不会这么简单，希望老师能分享一下，这块的具体实现逻辑。<br>目前在做的一个项目就涉及大文件上传、下载、解析的事情，量变引起质变感觉文件的体积大到一定程度，就不是一个简单的文件上传、下载、解析的事情了，需要各种考虑怎么提高性能减短处理时间的问题，还要考虑中间网络断了或者解析数据时遇到不OK的数据怎么处理的问题。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>如果传输超大文件，用http就不太合适了，虽然也可以，但就会麻烦一些。可以考虑用其他方式，比如mq。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-29 12:03:18</div>
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
  <div class="_2_QraFYR_0">响应报文返回的数据太大，所以采用了chunk分块传输的话，那响应报文在传输完成前是什么样子，响应行和头过来了，响应体还在流式传递，那响应体内的数据该怎么展示?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 就是按照格式分成一些片段，逐块发送，浏览器收到后去掉分隔符，再拼成原来的样子。<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-05 12:54:09</div>
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
  <div class="_2_QraFYR_0">1、因为分块数据是明文传输，如果数据里有\r\n，是会影响分块处理的<br>2、个人感觉应该是应用于原文件</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1不对，因为chunked格式里已经有长度了。<br><br>2正确。<br><br>看来有不少同学对第二个问题比较迷惑，我再说具体一点。<br><br>比如说，有一个1M的纯文本，range请求其中的500K，然后服务器编码为gzip（Content-Encoding: gzip），压缩成200k，浏览器收到后解压缩，就得到了这部分的500k数据。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-03 12:04:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/1f/04/1cddf65b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>不二</span>
  </div>
  <div class="_2_QraFYR_0">老师，问一个问题，这篇文章主要讲的是服务器端如何分块传输给客户端数据，或者客户端如何获取部分服务器端的数据， 那web客户端可以分批上传一个大文件的功能吗？类似于云盘中的上传功能</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 上传就要用chunked功能，流式发送数据，服务器流式接收数据。<br><br>range功能只能是下载用，不能上传。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-16 11:42:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/04/c8/3c7af100.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Javatar</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，看完本节内容后，找了一个网络上的pdf，用telnet发了下请求，结果在响应报文中并没有看到chunked头，pdf也是大文件，但是并没有分块传输，该怎么理解？还是说大文件也可以不用分段传输？<br><br>Trying 202.38.64.11...<br>Connected to staff.ustc.edu.cn.<br>Escape character is &#39;^]&#39;.<br><br><br>GET &#47;~bhua&#47;Kurose_Labs_v7.0&#47;Wireshark_HTTP_v7.0.pdf HTTP&#47;1.1<br>Host: staff.ustc.edu.cn<br><br>HTTP&#47;1.1 200 OK<br>Date: Sun, 30 Aug 2020 05:54:44 GMT<br>Server: Apache&#47;2.0.52 (Red Hat)<br>Last-Modified: Thu, 19 Sep 2019 09:00:14 GMT<br>ETag: &quot;40d8086-2392cc-2e81a380&quot;<br>Accept-Ranges: bytes<br>Content-Length: 2331340<br>Connection: close<br>Content-Type: application&#47;pdf<br><br>%PDF-1.3<br>%?????????<br>4 0 obj<br>&lt;&lt; &#47;Length 5 0 R &#47;Filter &#47;FlateDecode &gt;&gt;<br>stream<br>....省略body</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，chunked只是传送大文件的一种方式，不是一定要用，而且大文件的定义也因网站而不同。<br><br>看一下文件头，里面有content-length头，那么这个很明显就是给出了确定长度，就不用chunked了。<br><br>chunked的应用场合主要是流式传送，内容的长度不确定，给不出content-length。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-30 14:16:23</div>
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
  <div class="_2_QraFYR_0">1、使用chunk分段后还能压缩吗？或者说chunk分段分的是压缩后还是压缩前的文件呢？2、使用了chunk，为什么内存、带宽会节省呢？总的数据大小不变吧？内存的话，分段后，前面的数据到达浏览器客户端后，是存在内存还是磁盘呢？如何节省内存呢?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.可以的，先压缩再分块。<br><br>2.分块的好处是每次只处理一小部分数据，比如一次只从磁盘读取10k，而不用把1G的文件都读进内存，发的时候也可以慢慢发，所以就节省了内存、带宽。<br><br>3.对方收到数据肯定是先在内存里，之后可以用各种策略，比如达到一定的大小（比如1M）就落盘存。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-22 14:54:40</div>
  </div>
</div>
</div>
</li>
</ul>