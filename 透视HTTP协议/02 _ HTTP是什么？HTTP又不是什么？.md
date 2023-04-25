<audio title="02 _ HTTP是什么？HTTP又不是什么？" src="https://static001.geekbang.org/resource/audio/bd/46/bd2f893babbff670fb54341068f21a46.mp3" controls="controls"></audio> 
<p>首先我来问出这个问题：“你觉得HTTP是什么呢？”</p><p>你可能会不假思索、脱口而出：“HTTP就是超文本传输协议，也就是<strong>H</strong>yper<strong>T</strong>ext <strong>T</strong>ransfer <strong>P</strong>rotocol。”</p><p>回答非常正确！我必须由衷地恭喜你：能给出这个答案，就表明你具有至少50%HTTP相关的知识储备，应该算得上是“半个专家”了。</p><p>不过让我们换个对话场景，假设不是我，而是由一位面试官问出刚才的问题呢？</p><p><img src="https://static001.geekbang.org/resource/image/b4/bf/b4de4be0f7dfd4185464bb5a1d6df0bf.png?wh=1142*640" alt="unpreview"></p><p>显然，这个答案有点过于简单了，不能让他满意，他肯定会再追问你一些问题：</p><ul>
<li>你是怎么理解HTTP字面上的“超文本”和“传输协议”的？</li>
<li>能否谈一下你对HTTP的认识？越多越好。</li>
<li>HTTP有什么特点？有什么优点和缺点？</li>
<li>HTTP下层都有哪些协议？是如何工作的？</li>
<li>……</li>
</ul><p>几乎所有面试时问到的HTTP相关问题，都可以从这个最简单的“HTTP是什么？”引出来。</p><p>所以，今天的话题就从这里开始，深度地解答一下“<strong>HTTP是什么？</strong>”，以及延伸出来的第二个问题“<strong>HTTP不是什么？</strong>”</p><h2>HTTP是什么</h2><p>咱们中国有个成语“人如其名”，意思是一个人的性格和特点是与他的名字相符的。</p><p>先看一下HTTP的名字：“<strong>超文本传输协议</strong>”，它可以拆成三个部分，分别是：“<strong>超文本</strong>”“<strong>传输</strong>”和“<strong>协议</strong>”。我们从后往前来逐个解析，理解了这三个词，我们也就明白了什么是HTTP。</p><!-- [[[read_end]]] --><p><img src="https://static001.geekbang.org/resource/image/d6/20/d697ba915bcca40a11b8a25571516720.jpg?wh=5000*1727" alt="unpreview"></p><p>首先，HTTP是一个<strong>协议</strong>。不过，协议又是什么呢？</p><p>其实“协议”并不仅限于计算机世界，现实生活中也随处可见。例如，你在刚毕业时会签一个“三方协议”，找房子时会签一个“租房协议”，公司入职时还可能会签一个“保密协议”，工作中使用的各种软件也都带着各自的“许可协议”。</p><p>刚才说的这几个都是“协议”，本质上与HTTP是相同的，那么“协议”有什么特点呢？</p><p>第一点，协议必须要有两个或多个参与者，也就是“协”。</p><p>如果<strong>只有</strong>你一个人，那你自然可以想干什么就干什么，想怎么玩就怎么玩，不会干涉其他人，其他人也不会干涉你，也就不需要所谓的“协议”。但是，一旦有了两个以上的参与者出现，为了保证最基本的顺畅交流，协议就自然而然地出现了。</p><p>例如，为了保证你顺利就业，“三方协议”里的参与者有三个：你、公司和学校；为了保证你顺利入住，“租房协议”里的参与者有两个：你和房东。</p><p>第二点，协议是对参与者的一种行为约定和规范，也就是“议”。</p><p>协议意味着有多个参与者为了达成某个共同的目的而站在了一起，除了要无疑义地沟通交流之外，还必须明确地规定各方的“责、权、利”，约定该做什么不该做什么，先做什么后做什么，做错了怎么办，有没有补救措施等等。例如，“租房协议”里就约定了，租期多少个月，每月租金多少，押金是多少，水电费谁来付，违约应如何处理等等。</p><p>好，到这里，你应该能够明白HTTP的第一层含义了。</p><p><span class="orange">HTTP是一个用在计算机世界里的协议。它使用计算机能够理解的语言确立了一种计算机之间交流通信的规范，以及相关的各种控制和错误处理方式。</span></p><p>接下来我们看HTTP字面里的第二部分：“<strong>传输</strong>”。</p><p>计算机和网络世界里有数不清的各种角色：CPU、内存、总线、磁盘、操作系统、浏览器、网关、服务器……这些角色之间相互通信也必然会有各式各样、五花八门的协议，用处也各不相同，例如广播协议、寻址协议、路由协议、隧道协议、选举协议等等。</p><p>HTTP是一个“<strong>传输协议</strong>”，所谓的“传输”（Transfer）其实很好理解，就是把一堆东西从A点搬到B点，或者从B点搬到A点，即“A&lt;===&gt;B”。</p><p>别小看了这个简单的动作，它也至少包含了两项重要的信息。</p><p>第一点，HTTP协议是一个“<strong>双向协议</strong>”。</p><p>也就是说，有两个最基本的参与者A和B，从A开始到B结束，数据在A和B之间双向而不是单向流动。通常我们把先发起传输动作的A叫做<strong>请求方</strong>，把后接到传输的B叫做<strong>应答方</strong>或者<strong>响应方</strong>。拿我们最常见的上网冲浪来举例子，浏览器就是请求方A，网易、新浪这些网站就是应答方B。双方约定用HTTP协议来通信，于是浏览器把一些数据发送给网站，网站再把一些数据发回给浏览器，最后展现在屏幕上，你就可以看到各种有意思的新闻、视频了。</p><p>第二点，数据虽然是在A和B之间传输，但并没有限制只有A和B这两个角色，允许中间有“中转”或者“接力”。</p><p>这样，传输方式就从“A&lt;===&gt;B”，变成了“A&lt;=&gt;X&lt;=&gt;Y&lt;=&gt;Z&lt;=&gt;B”，A到B的传输过程中可以存在任意多个“中间人”，而这些中间人也都遵从HTTP协议，只要不打扰基本的数据传输，就可以添加任意的额外功能，例如安全认证、数据压缩、编码转换等等，优化整个传输过程。</p><p>说到这里，你差不多应该能够明白HTTP的第二层含义了。</p><p><span class="orange">HTTP是一个在计算机世界里专门用来在两点之间传输数据的约定和规范。</span></p><p>讲完了“协议”和“传输”，现在，我们终于到HTTP字面里的第三部分：“<strong>超文本</strong>”。</p><p>既然HTTP是一个“传输协议”，那么它传输的“超文本”到底是什么呢？我还是用两点来进一步解释。</p><p>所谓“<strong>文本</strong>”（Text），就表示HTTP传输的不是TCP/UDP这些底层协议里被切分的杂乱无章的二进制包（datagram），而是完整的、有意义的数据，可以被浏览器、服务器这样的上层应用程序处理。</p><p>在互联网早期，“文本”只是简单的字符文字，但发展到现在，“文本”的涵义已经被大大地扩展了，图片、音频、视频、甚至是压缩包，在HTTP眼里都可以算做是“文本”。</p><p>所谓“<strong>超文本</strong>”，就是“超越了普通文本的文本”，它是文字、图片、音频和视频等的混合体，最关键的是含有“超链接”，能够从一个“超文本”跳跃到另一个“超文本”，形成复杂的非线性、网状的结构关系。</p><p>对于“超文本”，我们最熟悉的就应该是HTML了，它本身只是纯文字文件，但内部用很多标签定义了对图片、音频、视频等的链接，再经过浏览器的解释，呈现在我们面前的就是一个含有多种视听信息的页面。</p><p>OK，经过了对HTTP里这三个名词的详细解释，下次当你再面对面试官时，就可以给出比“超文本传输协议”这七个字更准确更有技术含量的答案：“<span class="orange">HTTP是一个在计算机世界里专门在两点之间传输文字、图片、音频、视频等超文本数据的约定和规范</span>”。</p><h2>HTTP不是什么</h2><p>现在你对“<strong>HTTP是什么？</strong>”应该有了比较清晰的认识，紧接着的问题就是“<strong>HTTP不是什么？</strong>”，等价的问题是“HTTP不能干什么？”。想想看，你能回答出来吗？</p><p>因为HTTP是一个协议，是一种计算机间通信的规范，所以它<strong>不存在“单独的实体”</strong>。它不是浏览器、手机APP那样的应用程序，也不是Windows、Linux那样的操作系统，更不是Apache、Nginx、Tomcat那样的Web服务器。</p><p>但HTTP又与应用程序、操作系统、Web服务器密切相关，在它们之间的通信过程中存在，而且是一种“动态的存在”，是发生在网络连接、传输超文本数据时的一个“动态过程”。</p><p><strong>HTTP不是互联网</strong>。</p><p>互联网（Internet）是遍布于全球的许多网络互相连接而形成的一个巨大的国际网络，在它上面存放着各式各样的资源，也对应着各式各样的协议，例如超文本资源使用HTTP，普通文件使用FTP，电子邮件使用SMTP和POP3等。</p><p>但毫无疑问，HTTP是构建互联网的一块重要拼图，而且是占比最大的那一块。</p><p><strong>HTTP不是编程语言</strong>。</p><p>编程语言是人与计算机沟通交流所使用的语言，而HTTP是计算机与计算机沟通交流的语言，我们无法使用HTTP来编程，但可以反过来，用编程语言去实现HTTP，告诉计算机如何用HTTP来与外界通信。</p><p>很多流行的编程语言都支持编写HTTP相关的服务或应用，例如使用Java在Tomcat里编写Web服务，使用PHP在后端实现页面模板渲染，使用JavaScript在前端实现动态页面更新，你是否也会其中的一两种呢？</p><p><strong>HTTP不是HTML</strong>，这个可能要特别强调一下，千万不要把HTTP与HTML混为一谈，虽然这两者经常是同时出现。</p><p>HTML是超文本的载体，是一种标记语言，使用各种标签描述文字、图片、超链接等资源，并且可以嵌入CSS、JavaScript等技术实现复杂的动态效果。单论次数，在互联网上HTTP传输最多的可能就是HTML，但要是论数据量，HTML可能要往后排了，图片、音频、视频这些类型的资源显然更大。</p><p><strong>HTTP不是一个孤立的协议</strong>。</p><p>俗话说“一个好汉三个帮”，HTTP也是如此。</p><p>在互联网世界里，HTTP通常跑在TCP/IP协议栈之上，依靠IP协议实现寻址和路由、TCP协议实现可靠数据传输、DNS协议实现域名查找、SSL/TLS协议实现安全通信。此外，还有一些协议依赖于HTTP，例如WebSocket、HTTPDNS等。这些协议相互交织，构成了一个协议网，而HTTP则处于中心地位。</p><h2>小结</h2><ol>
<li><span class="orange">HTTP是一个用在计算机世界里的协议，它确立了一种计算机之间交流通信的规范，以及相关的各种控制和错误处理方式。</span></li>
<li><span class="orange">HTTP专门用来在两点之间传输数据，不能用于广播、寻址或路由。</span></li>
<li><span class="orange">HTTP传输的是文字、图片、音频、视频等超文本数据。</span></li>
<li><span class="orange">HTTP是构建互联网的重要基础技术，它没有实体，依赖许多其他的技术来实现，但同时许多技术也都依赖于它。</span></li>
</ol><p>把这些综合起来，使用递归缩写方式（模仿PHP），我们可以把HTTP定义为“<strong>与HTTP协议相关的所有应用层技术的总和</strong>”。</p><p>这里我画了一个思维导图，也可以算是这个专栏系列文章的“知识地图”。</p><p><img src="https://static001.geekbang.org/resource/image/27/cc/2781919e73f5d258ff1dc371af632acc.png?wh=2065*4011" alt=""></p><p>你可以对照这张图，看一下哪些部分是自己熟悉的，哪些部分是陌生的，又有哪些部分是想要进一步了解的，下一讲我会详细讲解这张图。</p><h2>课下作业</h2><ol>
<li>有一种流行的说法：“HTTP是用于从互联网服务器传输超文本到本地浏览器的协议”，你认为这种说法对吗？对在哪里，又错在哪里？</li>
<li>你能再说出几个“HTTP不是什么”吗？</li>
</ol><p>欢迎你通过留言分享答案，与我和其他同学一起讨论。如果你觉得有所收获，欢迎你把文章分享给你的朋友。</p><p><img src="https://static001.geekbang.org/resource/image/eb/3a/ebcadbddbc7e0c146dbe7617a844e83a.png?wh=750*824" alt="unpreview"></p><p></p>
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
  <div class="_2_QraFYR_0">问题一：<br>我觉得这种说法是错误的。<br>    理由：HTTP是在计算机世界里，用于两点之间之间传输超文本的协议。这两点并不限定于是服务器还是浏览器。可以是从浏览器到服务器，也可以从服务器到服务器，也可以是浏览器到浏览器。并不能描述成从服务器到浏览器。<br>问题二：<br>HTTP不是软件、不是网址（暂时想到的比较少）<br>总结：<br>协议：HTTP是一个用在计算机世界里的协议。它使用计算机能够理解的语言确立了一种计算机之间交流通信的规范，以及相关的各种控制和错误处理方式<br> 传输：HTTP是一个在计算机世界里专门用来在两点之间传输数据的约定和规范<br>        1、HTTP协议是一个“双向协议”<br>        2、不限定两个角色，允许有中转或接力A&lt;=&gt;X&lt;=&gt;Y&lt;=&gt;Z&lt;=&gt;B    <br>文本：完整的有意义的数据，可以被上层应用程序处理<br>        包括但不限于 文字、图片、音频、压缩包<br>超文本：超越了普通文本的文本。是文字、图片、音频和视频等的混合体。最关键的是含有超链接。能从一个超文本跳跃到另一个超文本。形成复杂的非线性、网状的结构关系。<br><br>HTTP是一个在计算机世界里专门在两点之间传输文字、图片、音频、视频等超文本数据的约定和规范。<br><br>HTTP不是互联网、不是编程语言、不是HTML，不是一个孤立的协议<br>    HTTP通常跑在TCP&#47;IP协议栈之上，依靠IP实现寻址和路由、TCP协议实现可靠数据传输、DNS协议实现域名查找、SSL&#47;TLS协议实现安全通信。此外，还有一些协议依赖于HTTP，例如WebSocket、HTTPDNS等。这些协议相互交织，构成了一个协议网，而HTTP则处于中心地位。<br><br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 总结的很好，也很早啊。<br>指出一点误解：两个浏览器不能通信。服务器可以当客户端，但浏览器只是客户端。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-31 00:41:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/e0/0c/c6151e22.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>团结屯儿王二狗</span>
  </div>
  <div class="_2_QraFYR_0">所谓的专家是用大家能听懂的语言，把复杂的知识讲明白。看的出来峰哥简单的背后是巨大的知识储备，感觉很用心，不错。期待后面文章能够让大家循序渐进、由浅入深，已关注，哈哈哈</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是啊，讲清楚讲明白太不容易了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-04 15:26:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/23/74/e0b9807f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小米</span>
  </div>
  <div class="_2_QraFYR_0">这是我看过的讲HTTP最通俗易懂的文章，忍不住要点赞！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 非常感谢。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-31 01:03:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/48/d2/eb4c3649.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>A-Lang</span>
  </div>
  <div class="_2_QraFYR_0">这个课程感觉很适合基础的同学学习!不知道后面老师会不会逐渐深入讲解一些深层次的东西</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 讲深了当然可以，但如果上来就是RFC估计会“吓跑”很多人，所以还是循序渐进比较好。<br>后面的进阶、安全、飞翔都有比较深的干货，有具体的需求也可以提，如果感兴趣的人多就多加点料。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-31 07:59:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/f7/72/2d35f80c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>xing.org1^</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，请问小结第二条，说http是在两点之间进行传输数据。我的疑惑是：http不是协议吗？我就按照老师的比喻把他理解为“协议”、“合同”了，如果就是纸上的约定，只是一个规范的话，http怎么做传输数据的事情呢？另外http又是怎么做到的呢？<br><br>我的网络知识真的是小白，问的很幼稚还请见谅:)</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你理解的很对，既然是约定，只要大家都遵守，那么协议就生效了。<br><br>就像红绿灯，它只是有颜色转换，怎么就能管理交通呢，你可以对比理解一下。<br><br>计算机依据http的规范去做，发请求收响应，就实现了传输数据。如果不按照http规范，就不能完成通信。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-07 09:02:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/07/fa/62186c97.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>恩佐</span>
  </div>
  <div class="_2_QraFYR_0">老师你的知识导图里有错误<br>错误在HHTTP&#47;2里的gRPC<br>您写的是gRFC</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢指正，人老了，手抖了，笑。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-18 03:43:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/7f/a2/8ccf5c85.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>XThundering</span>
  </div>
  <div class="_2_QraFYR_0">有个小问题，为什么文章说HTTP通常跑在 TCP&#47;IP协议栈之上，请问还有其它协议栈吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 有的，比如在UNIX上可以用Domain Socket，还有SSL&#47;TLS。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-31 12:19:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/76/a7/374e86a7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>欢乐的小马驹</span>
  </div>
  <div class="_2_QraFYR_0">我定了快二十个专栏。这是唯一一个对所有人回复的专栏 👍</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 呵呵，感谢夸奖。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-18 20:04:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/b7/cb/18f12eae.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>不靠谱～</span>
  </div>
  <div class="_2_QraFYR_0">1 错误的说法，Http可以在任意两点间进行传输。只是从服务器传输到浏览器这种形式比较常见。<br>2 http不是一种服务，不是一种语言，不是一种网络。只是一种协议，一种约定。<br><br>感谢老师分享</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-31 07:22:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/fb/2d/e6548e48.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>tokamak</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，我想用Linux C++写一个HTTP Client，但有个问题：当我用socket套接字接收HTTP 响应报文时，会调用recv(int sockfd, void *buf, size_t len, int flags);，这里的len填多少合适呢？开源代码里有填1个字节的，也有填4096个字节的，你怎么看这个问题？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: len参数是buf的长度，你开了多少就填多少，实际接收到的数据长度在函数返回值里。<br>调用示例可以参考Nginx源码的ngx_recv.c。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-31 01:06:03</div>
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
  <div class="_2_QraFYR_0">作者写的很用心！点赞👍<br>HTTP 协议是双向的。服务器 -&gt; 客户端，客户端 -&gt; 服务器。<br>期待后面的内容</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: thanks.</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-31 11:01:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/09/da/9b2f84d9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>毕竟养猪能致富</span>
  </div>
  <div class="_2_QraFYR_0">罗老师，我今年7月毕业，我也是软件工程专业的。感觉看了你的课程真的懂了很多，讲得非常详细，越看越想看，本来都准备睡觉了，结果没睡着，起来又看了遍。😂希望在后面的课程能学到更多知识。嘻嘻</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 坚持就会有收获，keep going。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-01 00:56:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/7e/bb/019c18fc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>徐云天</span>
  </div>
  <div class="_2_QraFYR_0">总而言之，http是一个通信协议，它有它的规范。不会限制在某个平台。任何计算机，都可以使用它。计算机程序之间的通信可以使用它。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-31 07:16:00</div>
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
  <div class="_2_QraFYR_0">1. 「用于从互联网服务器传输超文本到本地浏览器」的说法太过片面，HTTP 是在两点之间，即服务端与客户端，而客户端不仅包括本地浏览器，服务器也可以作为客户端，其他的 App、小程序等应用程序也属于客户端。<br>2. HTTP 不是软件：HTTP 是没有实体的协议，而软件是一种具体实现。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: √</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-04 00:06:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/8b/6c/004026f4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>个人学习</span>
  </div>
  <div class="_2_QraFYR_0">罗老师，您好，有个疑问，HTTP 是在两点之间传输数据，这个「两点」是理解为两个终端设备之间吗？但是，我们学习网络协议，知道数据是一层一层传输，从 A 终端的网络层-&gt;....-&gt;物理层，然后到 B 终端的物理层-&gt;...-&gt; 网络层。而  HTTP 协议在这个过程仅仅能接触到的只是「客户端」以及「传输层协议」呀。所以，这个两点是否能够理解为只是「客户端」和「传输层协议」之间的数据传输？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 网络分很多层次，但在http来看它并不关心，下层是怎么样它都无所谓，在http这一层来看就是两个端点：客户端和服务器，中间经过了多少路由网关是不考虑的。<br><br>这个就是抽象的力量，当然理论上是这么说，实际上当然是层次收发的。<br><br>后面还会讲http与协议栈，到时候可以再问。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-31 16:01:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/2a/d5/62dfbc11.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>patsun</span>
  </div>
  <div class="_2_QraFYR_0"><br>HTTP可应用的个体是两个或者两个以上，对象可以是服务器与服务器、服务器与本地浏览器，本地浏览器与本地浏览器。<br><br>http不是网址</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 最后一个不对，两个浏览器不能通信。服务器可以当客户端，但浏览器只是客户端。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-31 08:33:08</div>
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
  <div class="_2_QraFYR_0">编程语言是人与计算机沟通交流所使用的语言，而 HTTP 是计算机与计算机沟通交流的语言。--记下来</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-17 12:48:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/d2/17/b52417a6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>迷途</span>
  </div>
  <div class="_2_QraFYR_0">打卡</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: go on。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-30 11:27:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/8d/0e/5e97bbef.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>半橙汁</span>
  </div>
  <div class="_2_QraFYR_0">问题1:<br>    对错各参一点，狭隘的理解http的部分功能确实是从互联网服务器传输超文本到本地浏览器的协议；结合本章学到的内容，不正确的理解在于太过片面，HTTP可以被定义为：‘与 HTTP 协议相关的所有应用层技术的总和’<br>问题2：<br>    http不是静止存在的，不是单独的存在的，不是一门单独的语言。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: great</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-04 10:48:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/8SdpYbicwXVXt0fIN7L0f2TSGIScQIhWXT7vTze9GHBsjTvDyyQW9KEPsKBpRNs4anV61oF59BZqHf586b3o4ibw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Leolee</span>
  </div>
  <div class="_2_QraFYR_0">半路出家的小白程序员报道，这个课程说得非常明白，连我这种基础不这么好的都听懂了而且入脑了。<br>HTTP（HyperText Transfer Protocol：超文本 传输 协议 （超媒体传输协议）；<br>超文本（HyperText）：超越了普通文本的文本，可以包含图片、视频、音频、文字，还包含了超链接，可以在当前超文本跳转到其他超文本（这个也是普通文本做不到的，与超文本的根本区别。<br>传输（Transfer）：在两点之间传输数据，这两点之间可以有多个支点节点，但不能用来寻址、广播、路由。<br>协议（Protocol）：双向协议，协议两方分别为请求方以及响应方（应答方）。浏览器A发送一些数据给网站B，网站再把一些数据传输给浏览器，呈现在电脑浏览器上的就是我们看到的各种内容了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 继续加油，只要努力就能成功。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-02 14:09:45</div>
  </div>
</div>
</div>
</li>
</ul>