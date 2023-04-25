<audio title="15 _ 海纳百川：HTTP的实体数据" src="https://static001.geekbang.org/resource/audio/f4/8d/f48b1f2b88e4de06c4551576992bf78d.mp3" controls="controls"></audio> 
<p>你好，我是Chrono。</p><p>今天我要与你分享的话题是“海纳百川：HTTP的实体数据”。</p><p>这一讲是“进阶篇”的第一讲，从今天开始，我会用连续的8讲的篇幅来详细解析HTTP协议里的各种头字段，包括定义、功能、使用方式、注意事项等等。学完了这些课程，你就可以完全掌握HTTP协议。</p><p>在前面的“基础篇”里我们了解了HTTP报文的结构，知道一个HTTP报文是由“header+body”组成的。但那时我们主要研究的是header，没有涉及到body。所以，“进阶篇”的第一讲就从HTTP的body谈起。</p><h2>数据类型与编码</h2><p>在TCP/IP协议栈里，传输数据基本上都是“header+body”的格式。但TCP、UDP因为是传输层的协议，它们不会关心body数据是什么，只要把数据发送到对方就算是完成了任务。</p><p>而HTTP协议则不同，它是应用层的协议，数据到达之后工作只能说是完成了一半，还必须要告诉上层应用这是什么数据才行，否则上层应用就会“不知所措”。</p><p>你可以设想一下，假如HTTP没有告知数据类型的功能，服务器把“一大坨”数据发给了浏览器，浏览器看到的是一个“黑盒子”，这时候该怎么办呢？</p><p>当然，它可以“猜”。因为很多数据都是有固定格式的，所以通过检查数据的前几个字节也许就能知道这是个GIF图片、或者是个MP3音乐文件，但这种方式无疑十分低效，而且有很大几率会检查不出来文件类型。</p><!-- [[[read_end]]] --><p>幸运的是，早在HTTP协议诞生之前就已经有了针对这种问题的解决方案，不过它是用在电子邮件系统里的，让电子邮件可以发送ASCII码以外的任意数据，方案的名字叫做“<strong>多用途互联网邮件扩展</strong>”（Multipurpose Internet Mail Extensions），简称为MIME。</p><p>MIME是一个很大的标准规范，但HTTP只“顺手牵羊”取了其中的一部分，用来标记body的数据类型，这就是我们平常总能听到的“<strong>MIME type</strong>”。</p><p>MIME把数据分成了八大类，每个大类下再细分出多个子类，形式是“type/subtype”的字符串，巧得很，刚好也符合了HTTP明文的特点，所以能够很容易地纳入HTTP头字段里。</p><p>这里简单列举一下在HTTP里经常遇到的几个类别：</p><ol>
<li>text：即文本格式的可读数据，我们最熟悉的应该就是text/html了，表示超文本文档，此外还有纯文本text/plain、样式表text/css等。</li>
<li>image：即图像文件，有image/gif、image/jpeg、image/png等。</li>
<li>audio/video：音频和视频数据，例如audio/mpeg、video/mp4等。</li>
<li>application：数据格式不固定，可能是文本也可能是二进制，必须由上层应用程序来解释。常见的有application/json，application/javascript、application/pdf等，另外，如果实在是不知道数据是什么类型，像刚才说的“黑盒”，就会是application/octet-stream，即不透明的二进制数据。</li>
</ol><p>但仅有MIME type还不够，因为HTTP在传输时为了节约带宽，有时候还会压缩数据，为了不要让浏览器继续“猜”，还需要有一个“Encoding type”，告诉数据是用的什么编码格式，这样对方才能正确解压缩，还原出原始的数据。</p><p>比起MIME type来说，Encoding type就少了很多，常用的只有下面三种：</p><ol>
<li>gzip：GNU zip压缩格式，也是互联网上最流行的压缩格式；</li>
<li>deflate：zlib（deflate）压缩格式，流行程度仅次于gzip；</li>
<li>br：一种专门为HTTP优化的新压缩算法（Brotli）。</li>
</ol><h2>数据类型使用的头字段</h2><p>有了MIME type和Encoding type，无论是浏览器还是服务器就都可以轻松识别出body的类型，也就能够正确处理数据了。</p><p>HTTP协议为此定义了两个Accept请求头字段和两个Content实体头字段，用于客户端和服务器进行“<strong>内容协商</strong>”。也就是说，客户端用Accept头告诉服务器希望接收什么样的数据，而服务器用Content头告诉客户端实际发送了什么样的数据。</p><p><img src="https://static001.geekbang.org/resource/image/51/b9/5191bce1329efa157a6cc37ab9e789b9.png?wh=2000*1121" alt=""></p><p><strong>Accept</strong>字段标记的是客户端可理解的MIME type，可以用“,”做分隔符列出多个类型，让服务器有更多的选择余地，例如下面的这个头：</p><pre><code>Accept: text/html,application/xml,image/webp,image/png
</code></pre><p>这就是告诉服务器：“我能够看懂HTML、XML的文本，还有webp和png的图片，请给我这四类格式的数据”。</p><p>相应的，服务器会在响应报文里用头字段<strong>Content-Type</strong>告诉实体数据的真实类型：</p><pre><code>Content-Type: text/html
Content-Type: image/png
</code></pre><p>这样浏览器看到报文里的类型是“text/html”就知道是HTML文件，会调用排版引擎渲染出页面，看到“image/png”就知道是一个PNG文件，就会在页面上显示出图像。</p><p><strong>Accept-Encoding</strong>字段标记的是客户端支持的压缩格式，例如上面说的gzip、deflate等，同样也可以用“,”列出多个，服务器可以选择其中一种来压缩数据，实际使用的压缩格式放在响应头字段<strong>Content-Encoding</strong>里。</p><pre><code>Accept-Encoding: gzip, deflate, br
Content-Encoding: gzip
</code></pre><p>不过这两个字段是可以省略的，如果请求报文里没有Accept-Encoding字段，就表示客户端不支持压缩数据；如果响应报文里没有Content-Encoding字段，就表示响应数据没有被压缩。</p><h2>语言类型与编码</h2><p>MIME type和Encoding type解决了计算机理解body数据的问题，但互联网遍布全球，不同国家不同地区的人使用了很多不同的语言，虽然都是text/html，但如何让浏览器显示出每个人都可理解可阅读的语言文字呢？</p><p>这实际上就是“国际化”的问题。HTTP采用了与数据类型相似的解决方案，又引入了两个概念：语言类型与字符集。</p><p>所谓的“<strong>语言类型</strong>”就是人类使用的自然语言，例如英语、汉语、日语等，而这些自然语言可能还有下属的地区性方言，所以在需要明确区分的时候也要使用“type-subtype”的形式，不过这里的格式与数据类型不同，<strong>分隔符不是“/”，而是“-”</strong>。</p><p>举几个例子：en表示任意的英语，en-US表示美式英语，en-GB表示英式英语，而zh-CN就表示我们最常使用的汉语。</p><p>关于自然语言的计算机处理还有一个更麻烦的东西叫做“字符集”。</p><p>在计算机发展的早期，各个国家和地区的人们“各自为政”，发明了许多字符编码方式来处理文字，比如英语世界用的ASCII、汉语世界用的GBK、BIG5，日语世界用的Shift_JIS等。同样的一段文字，用一种编码显示正常，换另一种编码后可能就会变得一团糟。</p><p>所以后来就出现了Unicode和UTF-8，把世界上所有的语言都容纳在一种编码方案里，遵循UTF-8字符编码方式的Unicode字符集也成为了互联网上的标准字符集。</p><h2>语言类型使用的头字段</h2><p>同样的，HTTP协议也使用Accept请求头字段和Content实体头字段，用于客户端和服务器就语言与编码进行“<strong>内容协商</strong>”。</p><p><strong>Accept-Language</strong>字段标记了客户端可理解的自然语言，也允许用“,”做分隔符列出多个类型，例如：</p><pre><code>Accept-Language: zh-CN, zh, en
</code></pre><p>这个请求头会告诉服务器：“最好给我zh-CN的汉语文字，如果没有就用其他的汉语方言，如果还没有就给英文”。</p><p>相应的，服务器应该在响应报文里用头字段<strong>Content-Language</strong>告诉客户端实体数据使用的实际语言类型：</p><pre><code>Content-Language: zh-CN
</code></pre><p>字符集在HTTP里使用的请求头字段是<strong>Accept-Charset</strong>，但响应头里却没有对应的Content-Charset，而是在<strong>Content-Type</strong>字段的数据类型后面用“charset=xxx”来表示，这点需要特别注意。</p><p>例如，浏览器请求GBK或UTF-8的字符集，然后服务器返回的是UTF-8编码，就是下面这样：</p><pre><code>Accept-Charset: gbk, utf-8
Content-Type: text/html; charset=utf-8
</code></pre><p>不过现在的浏览器都支持多种字符集，通常不会发送Accept-Charset，而服务器也不会发送Content-Language，因为使用的语言完全可以由字符集推断出来，所以在请求头里一般只会有Accept-Language字段，响应头里只会有Content-Type字段。</p><p><img src="https://static001.geekbang.org/resource/image/0e/10/0e9bcd6922fa8908bdba79d98ae5fa10.png?wh=2000*1121" alt=""></p><h2>内容协商的质量值</h2><p>在HTTP协议里用Accept、Accept-Encoding、Accept-Language等请求头字段进行内容协商的时候，还可以用一种特殊的“q”参数表示权重来设定优先级，这里的“q”是“quality factor”的意思。</p><p>权重的最大值是1，最小值是0.01，默认值是1，如果值是0就表示拒绝。具体的形式是在数据类型或语言代码后面加一个“;”，然后是“q=value”。</p><p>这里要提醒的是“;”的用法，在大多数编程语言里“;”的断句语气要强于“,”，而在HTTP的内容协商里却恰好反了过来，“;”的意义是小于“,”的。</p><p>例如下面的Accept字段：</p><pre><code>Accept: text/html,application/xml;q=0.9,*/*;q=0.8
</code></pre><p>它表示浏览器最希望使用的是HTML文件，权重是1，其次是XML文件，权重是0.9，最后是任意数据类型，权重是0.8。服务器收到请求头后，就会计算权重，再根据自己的实际情况优先输出HTML或者XML。</p><h2>内容协商的结果</h2><p>内容协商的过程是不透明的，每个Web服务器使用的算法都不一样。但有的时候，服务器会在响应头里多加一个<strong>Vary</strong>字段，记录服务器在内容协商时参考的请求头字段，给出一点信息，例如：</p><pre><code>Vary: Accept-Encoding,User-Agent,Accept
</code></pre><p>这个Vary字段表示服务器依据了Accept-Encoding、User-Agent和Accept这三个头字段，然后决定了发回的响应报文。</p><p>Vary字段可以认为是响应报文的一个特殊的“版本标记”。每当Accept等请求头变化时，Vary也会随着响应报文一起变化。也就是说，同一个URI可能会有多个不同的“版本”，主要用在传输链路中间的代理服务器实现缓存服务，这个之后讲“HTTP缓存”时还会再提到。</p><h2>动手实验</h2><p>上面讲完了理论部分，接下来就是实际动手操作了。可以用我们的实验环境，在www目录下有一个mime目录，里面预先存放了几个文件，可以用URI“/15-1?name=file”的形式访问，例如：</p><pre><code>http://www.chrono.com/15-1?name=a.json
http://www.chrono.com/15-1?name=a.xml
</code></pre><p>在Chrome里打开开发者工具，就能够看到Accept和Content头：</p><p><img src="https://static001.geekbang.org/resource/image/b4/9e/b4d1671872876a60ad717821f0fb819e.png?wh=2000*906" alt=""></p><p>你也可以把任意的文件拷贝到mime目录下，比如压缩包、MP3、图片、视频等，再用Chrome访问，观察更多的MIME type。</p><p>有了这些经验后，你还可以离开实验环境，直接访问各大门户网站，看看真实网络世界里的HTTP报文是什么样子的。</p><h2>小结</h2><p>今天我们学习了HTTP里的数据类型和语言类型，在这里为今天的内容做个小结。</p><p><img src="https://static001.geekbang.org/resource/image/b2/58/b2118315a977969ddfcc7ab9d26cb358.png?wh=2000*1009" alt=""></p><ol>
<li><span class="orange">数据类型表示实体数据的内容是什么，使用的是MIME type，相关的头字段是Accept和Content-Type；</span></li>
<li><span class="orange">数据编码表示实体数据的压缩方式，相关的头字段是Accept-Encoding和Content-Encoding；</span></li>
<li><span class="orange">语言类型表示实体数据的自然语言，相关的头字段是Accept-Language和Content-Language；</span></li>
<li><span class="orange">字符集表示实体数据的编码方式，相关的头字段是Accept-Charset和Content-Type；</span></li>
<li><span class="orange">客户端需要在请求头里使用Accept等头字段与服务器进行“内容协商”，要求服务器返回最合适的数据；</span></li>
<li><span class="orange">Accept等头字段可以用“,”顺序列出多个可能的选项，还可以用“;q=”参数来精确指定权重。</span></li>
</ol><h2>课下作业</h2><ol>
<li>试着解释一下这个请求头“Accept-Encoding: gzip, deflate;q=1.0, *;q=0.5, br;q=0”，再模拟一下服务器的响应头。</li>
<li>假设你要使用POST方法向服务器提交一些JSON格式的数据，里面包含有中文，请求头应该是什么样子的呢？</li>
<li>试着用快递发货收货比喻一下MIME、Encoding等概念。</li>
</ol><p>欢迎你把自己的答案写在留言区，与我和其他同学一起讨论。如果你觉得有所收获，欢迎你把文章分享给你的朋友。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/b8/11/26838646.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>彧豪</span>
  </div>
  <div class="_2_QraFYR_0">上周五和服务端做上传图片的时候遇到过这个content-type的问题，上传图片时候我这边需要设置content-type:&quot;image&#47;jpg&quot;，然后传完了，我在预览的时候获取图片的地址，此时比如通过a标签的方式打开新标签预览该图片时才能成功预览，不然如果使用上传的js-sdk设置的默认类型：content-type:&quot;octet-stream&quot;，那么浏览器就不认识这个图片了，转而会下载这个文件(图片)，所以我是不是可以理解为content-type这字段在请求头，和响应头里都能使用？或者上传文件这个业务又不同于一般的请求操作呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，看来是我没说清楚，导致有的同学误会了。<br><br>content-type是实体字段，所以请求和响应里都可以用，作用是指明body数据的类型。<br><br>正文里为了叙述方便，只在服务器的响应报文里出现了content-type，实际上它是个通用字段，如果要发post请求，就需要带上它。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-01 13:04:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/16/c8/980776fc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>走马</span>
  </div>
  <div class="_2_QraFYR_0">accept 表达的是你想要的<br>而你发送 post请求时，你发送的数据是给服务器的，这时候就需要像 服务器会用 content-type 标明它给你的数据类型一样，你也需要用 content- 来表明你给别人的数据的一些属性</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: √</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-04 10:21:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/05/06/f5979d65.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>亚洲舞王.尼古拉斯赵四</span>
  </div>
  <div class="_2_QraFYR_0">1.含义是：我这个请求最希望服务器给我返回的编码方式是gzip和deflate，他们俩在我这是最优的地位，我不接受br的编码方式，如果还有其他的编码方式的话对我来说权重0.5。服务器可能的响应头是<br>HTTP&#47;1.1 200 OK<br>Content-Encoding: gzip<br><br>2.请求头可能是<br>POST &#47;serv&#47;v1&#47;user&#47;auth HTTP&#47;1.1<br>Content-Type: application&#47;json<br>Accept-Language: zh-CN, zh<br>Accept-Charset: gbk, utf-8<br>3.MIME类比快递的话就是你要快递的物品（衣服，食物等），Encoding就是快递你这个物品的包装方式，如果是衣服可能就随意一点一个袋子，如果是食物担心腐烂给你放个冰袋进去<br>不知道回答的对不对，请老师指正</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 回答的不错。<br><br>第二个问题要修改一下，这也怪我没有说清楚。<br><br>content-*字段也可以用在请求报文里，说明请求体的数据类型。在这里不能用accept字段，因为是post，所以要用content-language来指明body的语言类型，在content-type里用charset指明编码类型。<br><br>可以参考其他同学的回答。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-01 16:51:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/b8/78/2828195b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>隰有荷</span>
  </div>
  <div class="_2_QraFYR_0">用post方法请求接口时，在客户端语言的设置上不能使用Accept-Language吗？为什么一定是Contenr-Language呢？是不是Accept-Language只用于get方式时，表明客户端需要的的语言呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: post和get时都可以使用Accept-Language，表示客户端可以理解的语言，要求服务器按照指示返回数据。<br><br>Content-Language表示的是body数据的语言，因为post带有body，所以要用Content-Language来告诉服务器，报文的body是什么。<br><br>如果get报文也有body，那么它也可以使用Content-Language。<br><br>Accept-Language是请求头字段，只要是发请求就可以带。<br><br>Content-Language是实体头字段，只要是有body就可以带。<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-02 08:40:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/1a/66/2d9db9ed.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>苦行僧</span>
  </div>
  <div class="_2_QraFYR_0">现在很多小文件 比如图片 都往云存上放了 千万指定正确content-type 一旦指定错 批量修改太麻烦 而且会影响终端的解析</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 经验之谈！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-02 07:19:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/e0/99/5d603697.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>MJ</span>
  </div>
  <div class="_2_QraFYR_0">老师，每一个并发连接，都需要新建tcp三次握手吗？还是一次tcp握手后，可以并发多个连接？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 每建立一个连接就需要tcp握手，对同一个ip地址+端口，浏览器通常最多建立6个并发连接。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-05 07:34:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/df/6e/267bd6ee.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>1900</span>
  </div>
  <div class="_2_QraFYR_0">“所以后来就出现了 Unicode 和 UTF-8，把世界上所有的语言都容纳在一种编码方案里，UTF-8 也成为了互联网上的标准字符集。”<br><br>这句话最后有点问题吧？Unicode才是字符集，应该是“遵循UTF-8字符编码方式的Unicode字符集也成为了互联网上的标准字符集”，是么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，我说的时候不太准确。utf-8只是编码方案，Unicode是字符集。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-01 17:14:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTImeO3D5b56u2oHuQ13e58OmLexGVjYt6NTn1W4XjALJUrx7vh0FE2OVZvpxiczt8SibZM6TTFp2ASA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>BellenHsin</span>
  </div>
  <div class="_2_QraFYR_0">这篇写的不错</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: thanks。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-01 08:46:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/fc/d7/b102034a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>do it</span>
  </div>
  <div class="_2_QraFYR_0">1、试着解释一下这个请求头“Accept-Encoding: gzip, deflate;q=1.0, *;q=0.5, br;q=0”，再模拟一下服务器的响应头。<br>  ：最好使用gzip,deflate压缩格式，我不接受br压缩，如果都没有的话就选择其他格式<br><br>2、假设你要使用 POST 方法向服务器提交一些 JSON 格式的数据，里面包含有中文，请求头应该是什么样子的呢？<br>      content-type: application&#47;json, charset=gbk<br>      content-language:zh-cn, zh<br><br>3、试着用快递发货收货比喻一下 MIME、Encoding 等概念。<br>     物品的种类（水果、衣服）就是MIME，包装方式就是Encoding</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 回答的很好，继续努力。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-20 23:52:50</div>
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
  <div class="_2_QraFYR_0">老师好!有个问题,之前遇到过一个发送ajax请求。前端忘记在content-type里面指定，application&#47;json。后端接受数据失败。具体表现不太记得了好像都是null。后来前端加了content-type就好了。accept比较好理解就是发起请求放想要接受的内容。content-type是服务器，是响应类型的话。客户端在发送请求时压根就不知道啊，也不应该由客户端来设置。<br>所以我想问的是，accept相关的都是请求头里面的数据<br>content-type相关的都是响应头里的数据么?<br>至于我前面正确的写法应该是在accept里面设置json类型。错写了content-type。框架做了兼容处理(在服务端看起来content-type起作用了)?<br>谢谢老师</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 客户端在发送请求的时候也有义务设置content-type，也应该是知道数据是什么类型的，你设置成json，服务器看到了就好处理。<br><br>content-type是实体字段，请求响应里都可以出现。<br><br>accept是告诉服务器，客户端支持什么类型，防止服务器发过来的数据不认识。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-01 08:57:29</div>
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
  <div class="_2_QraFYR_0">做web开发有两个问题，估计许多人都遇到过，一个是乱码问题，这个是字符集设置不一致导致的具体哪里设置的不一致通常需要观察所有需要数据转换的地方，一般是客户端和服务器端不一致了。另外，就是文件上传，这个格式一定要设置对，否则就会感到莫名其妙。<br>我想请教老师两个问题：<br>1：看到说浏览器最多会有六个连接并发执行，为什么是六个，不会是因为六六大顺吧？<br>2：文件上传的速度和文件的大小密切相关，文件上传的大小限制都是有哪些限制或控制？之前，这是老早了，发现有些框架默认只能是2g</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>1.这个是rfc的规定，其实以前还要小。<br><br>2.http协议对大小没有限制，这些应该都是网站服务器做的限制，比如Nginx就可以设置client body的大小。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-29 09:42:12</div>
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
  <div class="_2_QraFYR_0">老师好!那accept是不是有两个语意<br>1.客户端希望接受(支持)的数据类型<br>2.我发送的数据就是这个类型的。请用这些方式解析?<br>问题:accept指定text。实际传的数据是一个json这样的后台会用text解析。然后拿不到数据是么?在请求头里加content -type这些字段会起作用么?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1.accept是你说的第一个意思，没有第二个意思。<br><br>2.第二个意思应该用Content-Type<br><br>3.看后台逻辑如何处理，数据是肯定可以拿到的，而且json也属于text。<br><br>4.在请求头里可以加content-type字段，表示请求体的数据类型。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-01 09:08:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/18/b3/848ffa10.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Jinlee</span>
  </div>
  <div class="_2_QraFYR_0">1. 含义：服务器，你返回的数据编码方式最好是gzip或者deflate，实在不行返回其他的编码方式也行，但是，我不接受br类型的数据编码方式<br><br>模拟服务器响应头：<br>HTTP&#47;1.1 200 OK<br>Content-Encoding: gzip <br><br>2. 请求头：<br>POST &#47; HTTP&#47;1.1<br>Content-type:text&#47;json; chartset=GBK, utf-8<br><br>3. 收发快递比喻：<br>MIME type就是要发收的具体物品，如文件、生鲜、衣物等。Encoding就是快递的包装方式，如果是文件呢，那我就用专用文件袋给你寄过去；如果是生鲜，那我就给你套个保鲜泡沫寄过去；如果是衣物，那我就给你个一般的快递包装寄过去。<br><br>参考其他优秀同学的答案😬<br><br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 回答的很好。<br><br>第二个问题，少了个content-language，而且charset不能同时使用两种编码。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-07 00:06:48</div>
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
  <div class="_2_QraFYR_0">不错不错，靠谱这篇，天天看这些参数一直不知道具体意思，今天老师讲了以后理解了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 继续努力。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-02 20:36:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/f5/0b/73628618.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>兔嘟嘟</span>
  </div>
  <div class="_2_QraFYR_0">罗老师，我不是很能理解accept-language这类字段的意义，因为我们开发前后端肯定是有一套接口文档的，里面约定了各种开发细节，前后端应该使用什么编码什么语言，为何还要在报文里去提accept-language？<br>总不能前端写好了，发送一个报文给后端，后端程序员慢慢看里面的字段，再开始后端的编程吧？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: http协议当初设计的时候可没有考虑过前后端开发的问题，它的目标是给互联网上的各种人群去阅读，所以就需要有语言编码。<br><br>在你说的开发领域，前后端都约定好了，自然就不需要用accept-language来交换信息了，但对于普通的浏览网页来说，还是要用这个字段的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-20 11:28:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/94/82/d0a417ba.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>蓝配鸡</span>
  </div>
  <div class="_2_QraFYR_0">试着解释一下这个请求头“Accept-Encoding: gzip, deflate;q=1.0, *;q=0.5, br;q=0”，再模拟一下服务器的响应头。<br>我希望用GZIP或者deflate压缩算法，实在不行给我其他的算法也行，但千万别给我br压缩过的数据，我这边可不会这个算法！<br><br>假设你要使用 POST 方法向服务器提交一些 JSON 格式的数据，里面包含有中文，请求头应该是什么样子的呢？<br>content-type: zh-CN, zh, charset=utf-8, application&#47;json<br><br>试着用快递发货收货比喻一下 MIME、Encoding 等概念。<br>比如说我想在某宝上买乐高，<br>MIME 确定了我购买的乐高种类，是成品呢， 还是零散的需要自己拼装。<br>Encoding好比这个乐高的包装方式，某宝可以选择把所有的乐高零件全都放在一个包装里， 也可以分模块把零件放在不同的包装里。 我也可以主动告诉某宝我想要哪一种包装方式。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good，很勤奋啊。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-29 14:53:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/62/05/30ae6faf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>aNught</span>
  </div>
  <div class="_2_QraFYR_0">老师，您好，如果我accep-encoding填写了gzip，那服务端发来的报文是是gzip压缩过的吗，我需要解压才行是吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: accep-encoding:gzip表示客户端支持gzip，但服务器发过来的是否经过压缩需要看content-encoding，也有可能不压缩。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-09 10:54:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/1a/66/2d9db9ed.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>苦行僧</span>
  </div>
  <div class="_2_QraFYR_0">content-type 千万不能填错 否则其他终端解析会存在问题</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 好经验多分享。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-02 07:16:38</div>
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
  <div class="_2_QraFYR_0">1.服务器优先按照gzip和deflate压缩，否则用其他压缩算法，但是不用brotli算法</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: √</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-01 09:05:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/c7/5f/2028aae5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>搏未来</span>
  </div>
  <div class="_2_QraFYR_0">Accept表示客户端能接受什么，Content-*表示提供的具体信息，由于客户端与服务端都可以提供信息，所以Content-*是可以在两端都可以用的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 总结的很到位！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-07 09:01:54</div>
  </div>
</div>
</div>
</li>
</ul>