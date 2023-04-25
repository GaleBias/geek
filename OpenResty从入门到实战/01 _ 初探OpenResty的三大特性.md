<audio title="01 _ 初探OpenResty的三大特性" src="https://static001.geekbang.org/resource/audio/74/55/747214d33b71126a7e00818541410a55.mp3" controls="controls"></audio> 
<p>你好，我是温铭。</p><p>开篇词里我们说过，OpenResty的优势显而易见。不过，在具体学习之前，让我们先简单回顾下 OpenResty 的发展过程，这有助于你对后面内容有更好的理解。</p><h2>OpenResty的发展</h2><p>OpenResty 并不像其他的开发语言一样从零开始搭建，而是基于成熟的开源组件——NGINX 和 LuaJIT。OpenResty 诞生于 2007 年，不过，它的第一个版本并没有选择 Lua，而是用了 Perl，这跟作者章亦春的技术偏好有很大关系。</p><p>但 Perl 的性能远远不能达到要求，于是，在第二个版本中，Perl 就被 Lua 给替换了。 不过，<strong>在 OpenResty 官方的项目中，Perl 依然占据着重要的角色，OpenResty 工程化方面都是用 Perl 来构建，比如测试框架、Linter、CLI 等</strong>，后面我们也会逐步介绍。</p><p>后来，章亦春离开了淘宝，加入了美国的 CDN 公司 Cloudflare。因为 OpenResty 高性能和动态的优势很适合 CDN 的业务需求，很快， OpenResty 就成为 CDN 的技术标准。 通过丰富的 lua-resty 库，OpenResty 开始逐渐摆脱 NGINX 的影子，形成自己的生态体系，在 API 网关、软WAF 等领域被广泛使用。</p><!-- [[[read_end]]] --><p>其实，我经常说，OpenResty 是一个被广泛使用的技术，但它并不能算得上是热门技术，这听上去有点矛盾，到底什么意思呢？</p><p>说它应用广，是因为 OpenResty 现在是全球排名第五的 Web 服务器。我们经常用到的 12306 的余票查询功能，或者是京东的商品详情页，这些高流量的背后，其实都是 OpenResty 在默默地提供服务。</p><p>说它并不热门，那是因为使用 OpenResty 来构建业务系统的比例并不高。使用者大都用OpenResty来处理入口流量，并没有深入到业务里面去，自然，对于 OpenResty 的使用也是浅尝辄止，满足当前的需求就可以了。这当然也与 OpenResty 没有像 Java、Python 那样有成熟的 Web 框架和生态有关。</p><p>说了这么多，接下来，我重点来介绍下，OpenResty 这个开源项目值得称道和学习的几个地方。</p><h2>OpenResty的三大特性</h2><h3>详尽的文档和测试用例</h3><p>没错，文档和测试是判断开源项目是否靠谱的关键指标，甚至是排在代码质量和性能之前的。</p><p>OpenResty 的文档非常详细，作者把每一个需要注意的点都写在了文档中。绝大部分时候，我们只需要仔细查看文档，就能解决遇到的问题，而不用谷歌搜索或者是跟踪到源码中。为了方便起见，OpenResty 还自带了一个命令行工具<code>restydoc</code>，专门用来帮助你通过 shell 查看文档，避免编码过程被打断。</p><p>不过，文档中只会有一两个通用的代码片段，并没有完整和复杂的示例，到哪里可以找到这样的例子呢？</p><p>对于 OpenResty 来说，自然是<code>/t</code>目录，它里面就是所有的测试案例。每一个测试案例都包含完整的 NGINX 配置和 Lua 代码，以及测试的输入数据和预期的输出数据。不过，OpenResty 使用的测试框架，与其他断言风格的测试框架完全不同，后面我会用专门章节来做介绍。</p><h3>同步非阻塞</h3><p>协程，是很多脚本语言为了提升性能，在近几年新增的特性。但它们实现得并不完美，有些是语法糖，有些还需要显式的关键字声明。</p><p>OpenResty 则没有历史包袱，在诞生之初就支持了协程，并基于此实现了<strong>同步非阻塞</strong>的编程模式。这一点是很重要的，毕竟，程序员也是人，代码应该更符合人的思维习惯。显式的回调和异步关键字会打断思路，也给调试带来了困难。</p><p>这里我解释一下，什么是同步非阻塞。先说同步，这个很简单，就是按照代码来顺序执行。比如下面这段伪码：</p><pre><code>local res, err  = query-mysql(sql)
local value, err = query-redis(key)
</code></pre><p>在同一请求连接中，如果要等 MySQL 的查询结果返回后，才能继续去查询 Redis，那就是同步；如果不用等 MySQL 的返回，就能继续往下走，去查询 Redis，那就是异步。对于 OpenResty 来说，绝大部分都是同步操作，只有 <code>ngx.timer</code> 这种后台定时器相关的 API，才是异步操作。</p><p>再来说说非阻塞，这是一个很容易和“异步”混淆的概念。这里我们说的“阻塞”，特指阻塞操作系统线程。我们继续看上面的例子，假设查询 MySQL 需要1s 的时间，如果在这1s 内，操作系统的资源（CPU）是空闲着并傻傻地等待返回，那就是阻塞；如果 CPU 趁机去处理其他连接的请求，那就是非阻塞。非阻塞也是 C10K、C100K 这些高并发能够实现的关键。</p><p>同步非阻塞这个概念很重要，建议你仔细琢磨一下。我认为，这一概念最好不要通过类比来理解，因为不恰当的类比，很可能把你搞得更糊涂。</p><p>在 OpenResty 中，上面的伪码就可以直接实现同步非阻塞，而不用任何显式的关键字。这里也再次体现了，让开发者用起来更简单，是 OpenResty 的理念之一。</p><h3>动态</h3><p>OpenResty 有一个非常大的优势，并且还没有被充分挖掘，就是它的<strong>动态</strong>。</p><p>传统的 Web 服务器，比如 NGINX，如果发生任何的变动，都需要你去修改磁盘上的配置文件，然后重新加载才能生效，这也是因为它们并没有提供 API，来控制运行时的行为。所以，在需要频繁变动的微服务领域，NGINX 虽然有多次尝试，但毫无建树。而异军突起的 Envoy， 正是凭着 xDS 这种动态控制的 API，大有对 NGINX 造成降维攻击的威胁。</p><p>和 NGINX 、 Envoy 不同的是，OpenResty 是由脚本语言 Lua 来控制逻辑的，而动态，便是 Lua 天生的优势。通过 OpenResty 中 lua-nginx-module 模块中提供的 Lua API，我们可以动态地控制路由、上游、SSL 证书、请求、响应等。甚至更进一步，你可以在不重启 OpenResty 的前提下，修改业务的处理逻辑，并不局限于 OpenResty 提供的 Lua API。</p><p>这里有一个很合适的类比，可以帮你理解上面关于动态的说明。你可以把 Web 服务器当做是一个正在高速公路上飞驰的汽车，NGINX 需要停车才能更换轮胎，更换车漆颜色；Envoy 可以一边跑一边换轮胎和颜色；而 OpenResty 除了具备前者能力外，还可以在不停车的情况下，直接把汽车从 SUV 变成跑车。</p><p>显然，掌握这种“逆天”的能力后，OpenResty 的能力圈和想象力就扩展到了其他领域，比如  Serverless 和边缘计算等。</p><h2>你学习的重点在哪里？</h2><p>讲了这么多OpenResty的重点特性，你又该怎么学呢？我认为，学习需要抓重点，围绕主线来展开，而不是眉毛胡子一把抓，这样，你才能构建出脉络清晰的知识体系。</p><p>要知道，不管多么全面的课程，都不可能覆盖所有问题，更不能直接帮你解决线上的每个 bug 和异常。</p><p>回到OpenResty的学习，在我看来，想要学好 OpenResty，你必须理解下面8个重点：</p><ul>
<li>
<p>同步非阻塞的编程模式；</p>
</li>
<li>
<p>不同阶段的作用；</p>
</li>
<li>
<p>LuaJIT 和 Lua 的不同之处；</p>
</li>
<li>
<p>OpenResty API 和周边库；</p>
</li>
<li>
<p>协程和 cosocket；</p>
</li>
<li>
<p>单元测试框架和性能测试工具；</p>
</li>
<li>
<p>火焰图和周边工具链；</p>
</li>
<li>
<p>性能优化。</p>
</li>
</ul><p>这些内容正是我们学习的重点，在专栏的各个模块中我都会分别讲到。在学习的过程中，我希望你能举一反三，并且根据自己的兴趣点和背景，有针对性地深入阅读某些章节。</p><p>如果你是 OpenResty 的初学者，那么你可以完全跟着专栏的进度，在自己的环境中安装 OpenResty，运行并修改示例代码。要记住，你的重点在于构建 OpenResty 的全貌，而非死磕某个知识点。当然，如果你有疑问的地方，随时可以在留言区提出，我会解答你的困惑。</p><p>如果你正在项目中使用 OpenResty，那就太棒了，相信你在阅读 LuaJIT 和性能优化章节时，一定会有更多的共鸣，更能应用到实际，在你的项目中看到优化前后的性能指标变化。</p><p>另外，如果你想要给 OpenResty 以及周边库贡献代码，那么最大的门槛，并不是对 OpenResty 原理的理解，或者是如何编写 NGINX C 模块的问题，而是测试案例和代码规范。我见过太多 OpenResty 的代码贡献者（也包括我自己），在一个 PR 上反复修改测试案例和代码风格，这其中有太多鲜为人知的潜规则。所以，专栏的代码规范和单元测试部分，就是为你准备的。</p><p>而如果你是测试工程师，即使你不使用 OpenResty，OpenResty 的测试框架和性能分析工具集，也必能给你非常多的启发。毕竟，OpenResty 在测试上面的投入和积累是相当深厚的。</p><h2>写在最后</h2><p>欢迎你留言和我分享你的 OpenResty 学习之路，在这期间，你又走过哪些弯路呢？也欢迎你把这篇文章转发给你的同事、朋友。</p><p>还是那句话，在学习的过程中，你有任何疑问，都可以在专栏中留言，我会第一时间给你答复。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/3b/36/2d61e080.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>行者</span>
  </div>
  <div class="_2_QraFYR_0">同步异步说的是代码，调用就有返回是同步，反之是异步。<br>阻塞非阻塞说的是cpu，apu要等待就是阻塞，反之非阻塞。<br><br>非阻塞并不能缩减rt时间，其最大的优点是可以服务更多的请求，达到c100k。<br><br>针对ff同学的问题，阻塞非阻塞对于代码来说是，仅仅是底层实现不同。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-27 09:35:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/25/f7/4cc60573.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>zhang</span>
  </div>
  <div class="_2_QraFYR_0">分享一篇以前学openresty 时写的笔记，当时处于一个阅读过nginx源码，但是没有实际使用或者开发nginx的情况，另外个人的描述描述能力也比较差，很多知识储备不足。<br><br>当时写这篇笔记并不是对源码进行解读，只是站在一个有什么功能，我应该如何实现它，它是如何做的，这样一个角度去分析的。<br><br>希望这篇笔记可以让大家有一定收获，也希望我们可以互相扶持，一起坚持下去，学好这门课程。<br><br><br>http:&#47;&#47;note.youdao.com&#47;noteshare?id=965c9f034a82ffb0f8b4de6ca81f3e73</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍 一起学习</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-27 09:40:13</div>
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
  <div class="_2_QraFYR_0">看了下 openresty github 仓库 https:&#47;&#47;github.com&#47;openresty&#47;openresty 发现 t文件夹下没有什么测试文件，这个是需要看每个相关的模块的仓库吗？ 又看了 https:&#47;&#47;github.com&#47;openresty&#47;lua-nginx-module 这个模块发现 t文件夹下是有测试文件的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: https:&#47;&#47;github.com&#47;openresty&#47;openresty 是用于打源码包的项目，所以测试案例不多。<br>是的，需要看这个子项目的仓库。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-27 08:02:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/48/8f/7ecd4eed.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>FF</span>
  </div>
  <div class="_2_QraFYR_0">请教下温老师，关于阻塞&#47;非阻塞。<br><br>如果 CPU 趁机去处理其他连接的请求，那就是非阻塞。<br><br>但对于用户线程来讲，怎么理解这个非阻塞呢？<br><br>理解1，这个查询的用户线程是不是还得阻塞等待 1 秒钟等待返回？这样的话应用的性能还是会不理想？<br><br>理解2，用户线程也是非阻塞，操作系统线程非阻塞返回后，用户的数据不一定有，这个时候用户线程要轮询去调用查询，直到有数据。这样的话，对于应用来讲，性能不是一样不理想？<br><br>哪种理解是对的呢？但无论哪种，用户应用性能可能都提不上理想，这样的话为何非阻塞是C10K，C100K 实现的关键呢？<br><br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 理解 1 是对的。站在用户请求的角度，非阻塞并不会减少处理的时间，但是会减少等待的时间。OpenResty 的每个 worker 同一时间只在处理一个请求，如果阻塞了，这个 worker 上的其他请求都需要等待。<br>C10K 要解决的是高并发的问题，是服务端的整体性能。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-27 08:35:20</div>
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
  <div class="_2_QraFYR_0">总结：OpenResty 的三大特性。<br>1、详尽的文档和测试用例（这个不能算特性吧）。<br>2、同步非阻塞。<br>3、动态。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-29 15:51:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/4b/4b/97926cba.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Luciano李鑫</span>
  </div>
  <div class="_2_QraFYR_0">pr是啥意思<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是 GitHub 中 Pull Request 的缩写</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-04 20:27:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/47/ab/3311945c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>shonm</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，上面的代码中，如果是非阻塞的，他不是立马返回吗，怎么又会等1秒，怎么做到同步呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 非阻塞的自然不会等待 1 秒，但这 1 秒钟的时间内，CPU 是去处理其他请求的逻辑，并且把当前请求挂起。<br>等数据库返回了结果后，才唤醒之前的请求，这样就做到了同步。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-02 23:17:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/1c/d7/f6bcb019.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>馬偉偉💫</span>
  </div>
  <div class="_2_QraFYR_0">期待已久，第一次听说这技术就是在网易云课堂老师讲的课，买了书准备学习老师就开了极客时间的专栏，结合书集和老师的专栏希望能有所收获。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 一起学习</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-27 06:17:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83eo2SjCeylLv0P3Glle5277kA4b8cAuxr1NrC0njPKEqzSpB8IEicHB29GicFFwG1qiaxs4hxRiaBmoibVw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>阳仔</span>
  </div>
  <div class="_2_QraFYR_0">Lua到底怎么读？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我读“路啦”</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-03 21:13:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/12/56/4abadfc3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>LoveDr.kang</span>
  </div>
  <div class="_2_QraFYR_0">不要通过类比去理解同步非阻塞，从这句话就感觉老师很务实，网上很多举例子的真的特别误导人，尤其一些没实际经验的新手，还是要从概念入手，多多体会，才能领会精髓。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-28 08:38:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>许多子</span>
  </div>
  <div class="_2_QraFYR_0">请问openresty可不可以就当作nginx来使用呢？不写lua的情况下，用来搭建web服务器，这两者有没有区别呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 当然可以，OpenResty 是基于 NGINX 的。但需要注意的是，OpenResty 的版本一般会落后于 NGINX。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-28 12:35:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/35/56/59ae9888.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小刚</span>
  </div>
  <div class="_2_QraFYR_0">代码实现是同步与异步，阻塞是线程调用过程</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-27 23:59:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/21/0b/1a/b793ca36.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>舟 leo</span>
  </div>
  <div class="_2_QraFYR_0">老师 有没有OpenResty的微信交流群呀</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-23 21:27:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/66/77/194ba21d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lzh</span>
  </div>
  <div class="_2_QraFYR_0">阻塞和非阻塞那里，我觉得文中有点没讲清楚。结合APUE第14章和自己的理解补充一下，仅供参考。<br><br>阻塞，意味着这个进程会被挂起；非阻塞，如果IO能读，就返回数据，err=nil；不能读则返回err != nil。<br><br>这里说mysql操作要1s（这个1s内，对mysql请求都会返回err!=nil），然后CPU趁机去处理其他请求，openresty在这里可能是用了类似yeild之类的方式（协程），当mysql返回err时，yeild到其他函数中执行，进程一直都在跑，没有被挂起。<br><br>如果当前进程就只有操作mysql一个协程（简单理解就是进程只有这一个专门query mysql的函数要执行），那这1s就会反复yeild出去然后又进入这个query函数，相当于一直在query mysql，然后mysq一直返回err。这样就是非阻塞的，在mysql没有返回数据前，进程是不会query redis的，这样就是同步的。<br><br>之前乍看“cpu趁机处理其他请求”这句话，会让我以为进程被挂起，这就感觉与非阻塞矛盾，像这种协程的方式，如果能按yeild来说说，那瞬间就能理解了。<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-08 16:17:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/97/31/76485177.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>贺钧威</span>
  </div>
  <div class="_2_QraFYR_0">老师好，想问下 nginx 是异步非阻塞的，openresty是同步非阻塞的，那他们之间的同步异步的差异具体是指什么</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-24 18:53:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_b2b3f5</span>
  </div>
  <div class="_2_QraFYR_0">老师，我刚入门OR, 有个小问题请教一下，ngx.utctime()与系统时间一致，ngx.localtime()比系统时间+8小时，怎么才能设置正常呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-25 10:51:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTK1zZ7icsZIYadtCb5T0BCzBLOWToNgQ2C7hPU5N5zpVbUeKfvlBjLibWeibxg0HN7icprEWaWF23S9qw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek__CherryKing</span>
  </div>
  <div class="_2_QraFYR_0">初学入坑打卡</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-27 00:59:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/cc/21/e3c45732.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lcp0578</span>
  </div>
  <div class="_2_QraFYR_0">不错👍</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-21 22:56:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/5a/37/8775d714.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jackstraw</span>
  </div>
  <div class="_2_QraFYR_0">老师有遇到在高并发的情况下，lua代码不执行的情况么？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-19 11:32:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/01/b1/658b8540.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_67aae8</span>
  </div>
  <div class="_2_QraFYR_0">做爬虫的话，怎么样？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-11 08:35:05</div>
  </div>
</div>
</div>
</li>
</ul>