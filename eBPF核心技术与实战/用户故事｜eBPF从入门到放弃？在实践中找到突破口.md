<audio title="用户故事｜eBPF从入门到放弃？在实践中找到突破口" src="https://static001.geekbang.org/resource/audio/1c/95/1c35e0ddb448648f07cc83bf955c7a95.mp3" controls="controls"></audio> 
<p>你好，我是小李同学，坐标深圳，是一名嵌入式开发工程师，已经工作 4 年了，目前主要从事维护系统稳定性的相关工作。今天我主要想跟你分享下我学习 eBPF 的“心路历程”，以及在这门课中的一些收获。</p><h2>多次试图“入门”，始终不得其法</h2><p>我最早接触到 eBPF 是在一篇公众号文章上，上面介绍说它是内核调试的一把利器。当时我就想了：这不是跟我平时的工作联系很密切吗？在调试死机、分析代码路径的时候应该都能用到！eBPF 的强大功能让我很激动，当时就想上手试试，看看能不能把这门技术用起来。</p><p>想象很美好，但是真正开始学习的时候却发现有些棘手。我先是找了一堆资料。关于eBPF 的资料网上倒是有很多，但是不够系统，很多资料讲解也不够细致深入，总觉着看起来不太明白。特别是有一些文章，上来直接把整个 ebpf 的原理图一贴，然后就直接懵了。</p><p>我当时想，既然看原理看不懂，那就先跑起来看看吧！我尝试了下大家说的一些适合入门的 eBPF 工具集，结果发现它们都是基于服务器使用的。而我的工作环境基本都是嵌入式平台，像 BCC 这样的工具集没法直接使用。自己折腾了下，环境没有搭建起来，第一次的尝试“入门”就这么宣告失败了。</p><p>再次看到 eBPF，是见有人推荐《BPF 之巅》和《Linux 内核观测技术 BPF》这两本书。推荐的同学说自己收获很大，我就第一时间下单了，希望能从书中找到进入 eBPF 世界的法门。</p><!-- [[[read_end]]] --><p>这次呢，不自己折腾开发环境了，直接照着书上说的，依葫芦画瓢地在 Ubuntu 虚拟机上运行 BPF 工具。简单执行一些命令就可以得到一些内核参数，刚开始还觉得挺有趣，跟了几个章节之后就觉得索然无味。因为在正常系统上无非是看一看参数，没有深入的理解，再加上我平时的工作中没有使用 eBPF 的场景，也就没有什么正反馈和长期坚持的动力。就这样，想起来了才玩玩看，想不起来或者比较忙的时候就搁置，过了一段时间，这次的尝试也就不了了之了。</p><p>虽然前两次的尝试入门都失败了，但我还是觉得不甘心：既然是一门技术，怎么能搞不会呢？相信自己应该是可以掌握它的，只是还没有找到合适的学习方法和关键的突破口。</p><p>这时，我刚好发现极客时间出了 eBPF 的课程，而且还是倪朋飞老师讲的。我是倪老师第一季专栏<a href="https://time.geekbang.org/column/intro/100020901">《Linux性能优化实战》</a>的老用户，在那门课里收获很大。于是我瞬间来了兴趣，第一时间订阅了这个专栏，希望这次能跟着倪老师走进 eBPF 的世界。</p><h2>反复折腾，找到一个突破口</h2><p>就这样，我开始了第三次“eBPF入门”尝试。跟着倪老师的课程，一步步地从搭建环境到原理剖析，再到实战应用，这比自己折腾要系统得多。因为有了大佬的指导，这次我也少走了很多弯路。</p><p>比如，之前我自己折腾的时候踩过一个坑：一看到不懂的就去网上查，有时候一查就发现了更多不理解的地方，往往越查越乱，最后陷入一个“泥塘”中。而在这门课里，倪老师为我们指出了哪些是可以暂时不用深究的，这就让我得以快速掌握最重要的核心原理，而避免陷入一些细节中。</p><p>再说说我最近的学习体会吧。在反复“折腾”的过程中，有时看上去是失败了，但实际上是自己实实在在地踩过了一次坑，为以后的实践积累了经验。至少呢，也能说明“此路不通”。我在运行 eBPF 程序的时候，有时遇到编译问题，有时又遇到跑不起来的问题。但正是在这个不断踩坑又不断地去修复、实践的尝试过程中，往往在某个点上就找到突破口了。总之，学 eBPF 这种实践性很强的技术，就得“折腾”，要多实践、多思考。</p><p>比如，我第一次看到 BCC、bpftrace、libbpf 等概念的时候，根本搞不清楚它们都是啥东东。经历过之前的自行研究后，我准备先跟着课程把它们用起来。基于 bpftrace、BCC 进行编程，简单地把它们当作一套工具链，将代码转换成对应的格式，然后交给内核处理，我们就可以获取到想要的信息。而对于代码格式和里面的函数，刚开始可以先记下来，先不管是什么原理，记住是怎么用的就行。实际上，在专栏的<a href="https://time.geekbang.org/column/article/484207"> 07</a>、<a href="https://time.geekbang.org/column/article/484372">08</a> 两讲中，倪老师已经详细介绍了如何使用 BCC、bpftrace、libbpf 三种方式开发跟踪函数，掌握了这些实践应用，我们也就搞清楚了它们之间的关系。我想，这就是倪老师在这门课里一直强调，要在实践中加深对 eBPF 原理理解的原因吧！</p><p>从开始学这门课的时候，我就希望能够把 eBPF 用在实际工作中。遗憾的是，我平时的工作环境都是在嵌入式系统中，没有 LLVM、Python 等环境，所以 BCC 和 bpftrace 是没法玩了。只能是使用 libbpf 编程方式，但课程中的 libbpf 例子是跑在宿主机上的，似乎也不能用。</p><p>我找了一堆资料后发现，要想在嵌入式系统中跑起来，需要交叉编译生成 libbpf.a，而 libbpf.a 又依赖于 zlib 和 elfutils 两个库。一通折腾之后，我终于生成了 libbpf.a，最后在嵌入式系统中把 “hello world” eBPF 程序跑起来了。基于 libbpf.a，我们又可以对 Linux 内核 tools/bpf/bpftool 进行编译，生成 bpftool，这样在嵌入式系统中也可以使用。</p><p>总之，在学习 eBPF 的道路上，只要找对方向，并且踏出了第一步，就可以慢慢地走下去。这个过程需要自己不断地去折腾、实践。对于课程中讲到的知识点，最好是要能用起来，所以我之前是用 ftrace 的，现在试着改成使用 eBPF 来获取一些内核信息等等。只要从一个小的突破点开始，不断地获取正反馈，就有更大的动力和兴趣走下去了。</p><p>最后要说的是：感谢倪老师，这次总算是入门了。现在我已经能自己进行一些简单的开发，后面需要的就是不断进行实践了。期待倪老师在动态更新阶段的精彩内容，也感谢极客时间，提供一个良好的学习平台。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/e6/ee/8fdbd5db.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Damoncui</span>
  </div>
  <div class="_2_QraFYR_0">更新啦更新啦~ 沙发是我的~</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-07 11:14:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/20/3b/2a/f05e546a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>🐮</span>
  </div>
  <div class="_2_QraFYR_0">倪老师，我也遇到小李同学一样的问题，我也是搞嵌入式设备的，想看看能否在嵌入式设备中把ebpf用起来， 怎么可以联系到他啊，相互交流交流；</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以加到我们的群中交流吗？<br><br>我们有一个交流群，可以关注“漫谈云原生”公众号，回复“联系”（不带引号）加入。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-22 21:03:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/07/8d/3e76560f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>王建峰</span>
  </div>
  <div class="_2_QraFYR_0">GOOD！！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-29 11:40:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/76/b0/14fec62f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>不了峰</span>
  </div>
  <div class="_2_QraFYR_0">老师，想请问一个网络问题<br>两台 Linux主机 （做 Oracle RAC 集群）  REHT 7.5版本。每主机两网卡绑定，方式是:roundrobin 。连同一个交换机。<br><br>遇到的问题是<br>从A机 ping B 机 ： 不规则的掉包。 ping -s 1500 全掉。<br>从B机 ping A 机 ： 结果同样是： 不规则的掉包。 ping -s 1500 全掉。<br><br>从业务网络ping A 正常， ping B机， 结果同上。 怀疑是B机的问题。<br><br>B机现在这个 网卡没有什么流量（因为业务访问流量都转到A机去了）。<br><br>想问：有没有什么方法  排查 B机，或是这个方面的问题？<br>主机负载空， teamdctl team0 stat ,ethtools -S 网卡 ｜grep error  也都没有错。。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-09 09:31:16</div>
  </div>
</div>
</div>
</li>
</ul>