<audio title="开篇词 _ OpenResty，为你打开高性能开发的大门" src="https://static001.geekbang.org/resource/audio/91/17/9136152349edc27d6f9d81980bbf2917.mp3" controls="controls"></audio> 
<p>你好，我是温铭，OpenResty 软件基金会主席，曾任某开源商业公司合伙人，前 360 开源技术委员会委员，在互联网安全公司工作了 10 年，负责开发过云查杀、反钓鱼和企业安全产品。接下来的几个月，我会带着你系统地学习一下 OpenResty。</p><h2>为什么学习 OpenResty</h2><p>为什么学习 OpenResty，这是开篇的第一个问题。我们正身处技术日新月异的时代，经常听到周围的工程师开玩笑说，学不动了。人的精力有限，选择学习某个技术都会有机会成本。最好的选择，是从你工作中涉及到的部分出发，学以致用。</p><p>对于服务端工程师来说，如果你的工作中涉及到 NGINX、高性能、高并发、动态控制、性能测试和分析等，那么不管开发语言和平台是什么，这门 OpenResty 课程都会让你有所裨益。如果你之前没有接触过 OpenResty，我确信它会给你打开另外一个服务端世界的大门。</p><p>OpenResty 是一个兼具开发效率和性能的服务端开发平台，<strong>虽然它基于 NGINX 实现，但其适用范围，早已远远超出反向代理和负载均衡</strong>。</p><p>它的核心是基于 NGINX 的一个 C 模块（lua-nginx-module），该模块将 LuaJIT 嵌入到 NGINX 服务器中，并对外提供一套完整的 Lua API，透明地支持非阻塞 I/O，提供了轻量级线程、定时器等高级抽象。同时，围绕这个模块，OpenResty 构建了一套完备的测试框架、调试技术以及由 Lua 实现的周边功能库。</p><!-- [[[read_end]]] --><p>你可以用 Lua 语言来进行字符串和数值运算、查询数据库、发送 HTTP 请求、执行定时任务、调用外部命令等，还可以用 FFI 的方式调用外部 C 函数。这基本上可以满足服务端开发需要的所有功能。</p><p>掌握好了 OpenResty，你就可以同时拥有脚本语言的开发效率和迭代速度，以及 NGINX C 模块的高并发和高性能优势。</p><h2>我与OpenResty的渊源</h2><p>说了这么多OpenResty的特点，我又是怎样与它结缘的呢？其实，我是在 2012 年开始接触OpenResty的，那会儿我正在为一个新的系统做技术选型，作为一个 Python 的忠实粉丝，我不喜欢 NGINX C 模块的艰涩，却希望得到它的高性能，鱼与熊掌想兼得。该怎么办呢？</p><p>经过一番搜寻后，我发现了 Python 社区“大妈” ZQ 的一篇介绍 OpenResty 的文章，可以说是如获至宝。不过，兴奋只持续了很短的时间，因为之后的我，就像是无头苍蝇一样，开始在黑暗中摸索着缓慢前行。踩了数不清的坑后，我才真正拿下了OpenResty。</p><p>和很多工程师不同的是，我喜欢写文章，在大学期间就一直维护着自己的技术博客。有一天晚上加班时，我发现身边一位工程师在用 GitHub 记录 ELK 的使用心得，并发布到了 GitBook 上。原来 GitHub 还可以开源书籍，而不只是代码！</p><p>我一下子就被点燃了，当晚就列出了《OpenResty 最佳实践》的目录，并开始“鼓动”周围的工程师加入。我们从未宣传过这个开源项目，但它慢慢变成了 OpenResty 入门者的最佳伙伴。</p><p>不过，在加入 OpenResty Inc. 后，我才逐渐发现，能写出正确的 OpenResty 代码并避免常见的坑，和写出高性能、优质的 OpenResty 代码之间，还相差了十万八千里。<strong>而跨越这个巨大鸿沟的法宝零件，散落在 OpenResty 开源项目的源码、文档、issue、PR、幻灯片、邮件列表中，需要你把它们串联成真正的法宝——一个完整的学习体系和知识图谱</strong>。</p><p>那如何才能体系化学习OpenResty呢？在 OpenResty 的技术交流群里面，很多工程师都曾经有过这样的困惑。</p><p>事实上，OpenResty 的学习资料还比较少，官方只有 API 文档，并没有提供入门和进阶的文档，而网上能找到的资料也不够系统。可以说，绝大部分的 OpenResty 使用者都是在摸着石头过河，过程很痛苦。</p><p>因此，我与极客时间合作了这个专栏，目的很明确，就是让你轻松快速地入门，并给你描绘出 OpenResty 的全貌，帮你建立知识体系，带你真正掌握OpenResty这款开发利器。</p><h2>学习这个专栏需要什么基础？</h2><p>OpenResty 是在 NGINX 和 LuaJIT 的基础上搭建的，所以我们肯定需要 NGINX 和 LuaJIT 的基础知识。</p><p>但你只需要很少的 NGINX 知识，就足够开始 OpenResty 之旅了。少到什么程度呢？涉及到的 NGINX 知识，我只用一节课就介绍完了。即使你完全没有接触过 NGINX，也可以跟着课程的节奏，逐步学习 OpenResty。</p><p>要知道，OpenResty 并不等同于 NGINX，OpenResty 这个项目的目的之一，就是让你感知不到 NGINX 的存在。</p><p>而从编程语言来看，Lua 是一种很容易理解的语言，你只要能够看懂它的代码，就可以完成本专栏的学习，并不需要能够独立写出复杂的 Lua 代码。同样的，我也会花少数几节课的时间，带你入门Lua，达到OpenResty 的使用水准。</p><h2>从实战中来，到实战中去</h2><p>实践出真知，这句话用在互联网技术的学习上很恰当。</p><p>和理论偏多的书籍不同，专栏的形式本身更偏重于实战。专栏中出现的不少代码，都源自开源 OpenResty 的测试案例，以及实际的开源项目。引用这些实际案例，就是希望你在入门之初，就能接触到最优秀的代码，了解到最真实的使用场景。</p><p>同时，我还会在专栏文章中，穿插多个视频课程。视频课的内容，都取自真实开源项目的功能点和 PR。通过视频，你会亲眼看到，刚刚学到的知识是如何在实际中使用的。</p><p>专栏最后的实战部分，则是我们的真实“战场”。我会带你一起，用 OpenResty 从零搭建一个微服务 API 网关。根据我们在社区中的统计，接近一半的 OpenResty 使用者，都把 OpenResty 用在 API 网关的开发上，Kong 和 orange 则是 OpenResty 领域中最流行的两个开源网关项目。你想自己从头搭建一个更简单、更高性能的 API 网关吗？一起来吧。</p><p>从实际的开源项目中学习，再到实际的开源项目中去实战，将实战融入完整的知识体系，这便是我的教学理念，希望你喜欢这种方式。</p><p>万尺高楼平地起，接下来，我会和你一起来逐步掌握 OpenResty，Enjoy！</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/d0/46/1a9229b3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>NEVER SETTLE</span>
  </div>
  <div class="_2_QraFYR_0">目前负责广告引擎系统的API网关，就是用Openresty搭建的，Lua语言开发起来很方便，让人欲罢不能的感觉，再加上Nginx高性能和高并发，后端负载均衡，感觉两者结合就是绝配。1、一个值得赞的是，上线时候直接reload一下就可以，根本不会出现停服那种情况。2、现在用Lua写代码习惯了，偶尔用下C++，感觉写起来太费劲了，比如json序列化与反序列化，Lua直接cjson.decode和cjson.encode，结果用C++我就不多说了，尤其json结构复杂，嵌套很深那种，真的有点费劲。3、共享字典sharedict用起来真的很方便，所有worker进程共享，自带锁机制，不用担心竞争的情况。4、Lua虽然用起来方便，但不注意真的会踩很多坑，之前有段时间流量同期没有什么增长，但是CPU利用率涨了10%，后来我们reload一下，结果就正常了。过段时间，问题复现了，我用perf工具查了下，发现一个字符串操作的函数占用率很高，后来查了下，是用字符串连接符..调用的，后来结合当时上线的代码，发现添加了许多业务日志，而日志里面字段都是用连接符..连接的，后来改成concat了，目前问题没发生过了。5、请问温老师，openresty现在用可靠的protobuf库吗，之前对接外部模块，api是protobuf协议，当时需求急，没有咋调研，我直接用最粗爆方式，用ffi方式调用C++，Lua给C++传json，C++json解码之后，把参数配成pb，返给Lua，反之亦然，感觉有点麻烦，请温铭老师指点迷津。<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Lua 性能相关的坑不少，后面会专门介绍。<br>据我说知，Lua 中 protobuf 的库都是基于 FFI 来做的，OpenResty 并没有专门的这方面的库。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-22 21:51:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/b8/3c/1a294619.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Panda</span>
  </div>
  <div class="_2_QraFYR_0">一直用 Openresty 做网关  轻量 稳定 高性能   期待了很久的专利 终于上架了 哈哈哈 👍🏻</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-23 00:30:45</div>
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
  <div class="_2_QraFYR_0">OpenResty 借助 Lua 语言，插上翅膀。OpenResty 为什么不借助其他脚本语言呢？比如 Shell 等。我通篇文章看下来一直在说 OpenResty 的优势，但是没有比较，只能脑补。很空洞。就我一个有这样的感觉吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: OpenResty 的第一个版本是把 perl 嵌入了 NGINX，但性能很差。NGINX 官方把 js 嵌入进来，也有一些开源项目把 php 嵌入 NGINX。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-27 15:29:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/90/46/c1ecf3b9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>咆哮</span>
  </div>
  <div class="_2_QraFYR_0">用了openresty两年，陆陆续续给公司做了两个版本的waf、api网关，希望学习提升自己</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: waf 和 api 网关都是 OpenResty 擅长的方向。技术选型不错：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-22 19:34:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83eqtOatxItTKgtHYS4cdYZ3KlibvFu3HJwAYhia9ohfvliciaxXnk3vOVMDM593OIIAYysMzPnJZZmHVHw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Peter</span>
  </div>
  <div class="_2_QraFYR_0">跟着温铭老师学 OpenResty，期待已久的课程～</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-22 17:36:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/dd/e0/034ce26f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Fouy_飞虎</span>
  </div>
  <div class="_2_QraFYR_0">2年前就开始接触OpenResty了，自己做了两个小网站。支持一把！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-22 17:29:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/be/b6/68f708fb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>starting</span>
  </div>
  <div class="_2_QraFYR_0">国内就缺少这种干货资料，这次正好有同类项目，期待后面的课程内容！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-22 20:22:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/5f/73/bb3dc468.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>拒绝</span>
  </div>
  <div class="_2_QraFYR_0">涨知识了，我是一名java后端开发；只知道有nginx，lua。在实际工作中看到有人用lua脚本实现基于redis的分布式锁，当时我在想api也可以实现，为什么要使用lua。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-22 18:13:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/6b/80/dd2b3b3f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>冯思鸣</span>
  </div>
  <div class="_2_QraFYR_0">现在的KONG API 网关就是基于openresty+ lua去开发的吧？基于KONG做扩展是否会更方便，毕竟它提供了很多现成的扩展。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，Kong 就是基于 OpenResty 的。基于 Kong 做插件也是 OK 的，如果你的需求是做网关的话。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-29 09:13:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/69/c9/abd2928c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Vicwan</span>
  </div>
  <div class="_2_QraFYR_0">还没开课就已经订阅，我是多么好学啊</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-22 21:13:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/d2/cc/d54b7e5b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>guoew</span>
  </div>
  <div class="_2_QraFYR_0"> 前段时间接触到OpenResty最佳实践，非常不错！期待温铭老师的课程。OpenResty，Enjoy！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-22 20:37:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/4c/c8/bed1e08a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>辣椒</span>
  </div>
  <div class="_2_QraFYR_0">非常期待，拓展视野的好机会，绝对物有所值</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-22 19:45:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/ff/63/042aaa14.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>故事、自己写</span>
  </div>
  <div class="_2_QraFYR_0">挺期待的，坑到底有多深~</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-22 19:45:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/59/1e/5f77ce78.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>吃草🐴~</span>
  </div>
  <div class="_2_QraFYR_0">说来惭愧，我是从买这门课起开始接触 OpenResty 的，哈哈。目前工作中还用不到这个，是兴趣驱动学习。希望学完课程后，能运用到工作中~😃</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以在新项目中试试，或者替代现有的 NGINX 服务也是不错的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-28 09:17:15</div>
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
  <div class="_2_QraFYR_0">老师，我们有个音视频后台服务没有过载保护，可以使用openrestry来开发网关ma</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: OpenResty 很适合限流限速的场景，可以针对不同的请求，来动态控制。后面也会有章节讲到。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-27 09:56:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/64/c4/f19016bb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ahmacoin</span>
  </div>
  <div class="_2_QraFYR_0">期待实践中使用</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-22 18:40:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/f1/7b/0f9d7a2f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>沈奇才</span>
  </div>
  <div class="_2_QraFYR_0">用于等到这门课了。。感谢老师的分享。。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-22 17:22:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/c3/6a/3272e095.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>李春恒</span>
  </div>
  <div class="_2_QraFYR_0">收到了推送，上来看看，工作上也一直在用openresty搞事情。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-22 17:20:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/ae/8c/125954e0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>权奥</span>
  </div>
  <div class="_2_QraFYR_0">第二层思考</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-22 17:17:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/16/77/30e059c8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>kindnull</span>
  </div>
  <div class="_2_QraFYR_0">学习了，以前公司用openresty结合nginx二次开发支持大并发。现在终于有机会好好学习了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-11 15:52:53</div>
  </div>
</div>
</div>
</li>
</ul>