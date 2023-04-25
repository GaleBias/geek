<audio title="用户故事 _ 小S：学习是人生路上生生不息的活泉" src="https://static001.geekbang.org/resource/audio/c8/77/c8f2a5a10fd4084231787e45d1f25077.mp3" controls="controls"></audio> 
<p>你好，我是小S，来自北方二线城市。我做了七八年的开发，最近一份工作是做安防音视频流媒体方向，其中部分工作内容是负责实际项目的对接任务，所以会涉及到很多的网络相关的问题。</p><h2>我与极客时间的不解之缘</h2><p>我是非科班出身的，最早只是在图书馆借借书看，后来从事了开发工作之后，我就开始买网课学习，可以说，我的整个计算机知识体系都是通过网课+图书构建起来的。</p><p>不过那个时候，我在学习的过程中就发现，这些资料要么是大部头的书、要么是很长的视频，学起来不是很方便，很少有大段的时间能专门坐下来看长视频并仔细分析、记笔记，而且也没太多时间去搜集额外的资料。还有一个问题就是这些资料很多都是良莠不齐、鱼龙混杂的，早期我还会在一些音频App上找，但因为这些App并不是专门做技术内容，所以也不太合适。</p><p>机缘巧合下，我发现了极客时间这个平台，课程上新很快，内容也一直在更新迭代。虽说我大部分计算机基础课之前都补上了，但是现在有更上一层楼的机会，何乐而不为呢，毕竟这种音频+图文的讲解模式还是很适合充实基础知识的。</p><p>并且，极客时间的大部分课程也都是从实际应用的角度出发，拓宽了我们来自二三线以下城市开发者的视野。说实在话，小地方没什么大厂，小公司也不太能用得上新技术，但是我想，<strong>有追求的人是不会想困在一潭死水中的，而是会付诸实际行动去学习、去实践，这才是取得进步的活泉</strong>。</p><!-- [[[read_end]]] --><p>另外，我也比较喜欢极客时间划分的学习路径，像我这样非科班出身的技术人，其实打心底是想要把内功补上的，而即使是科班出身，我想大部分人内心深处也是向往着有一天把这些知识好好串一串吧。</p><h2>我的学习方法</h2><p>说说我用极客时间的方法，希望也能给你带来启发或共鸣。我在极客时间上初步规划了三个学习进阶方向：计算机基础知识、C++工程师（目前的方向）、Go工程师（希望以后兼顾的方向）。</p><p>首先就是<strong>常规刷课</strong>。比如平时通勤、下班休息后，甚至跑步时都可以听一听。然后我一般会在周日下午固定几个小时，用来整理当周学过的内容。当然，实践性强的内容这么学当然不行，需要动手做实验，为此我还买了一个阿里云ECS。另外有些课程老师会推荐一些延伸资料，我平时也会看一下。</p><p>再者就是<strong>善用搜索</strong>。极客时间的课程都是很体系化的，涵盖的技术范围也比较广泛，可以帮我们省去不少埋头苦苦摸索的时间，所以我推荐根据自己平时工作中用到的一些技术来进行扩展。</p><p>比如我之前的公司有做中台的打算，我就去看了《说透中台》这门课。看完后感觉公司并不适合做。这就像当年大学时候的数学建模一样，我分析完数据之后，得出的结论就是它是假设的，压根不成立。但是这样也没关系，因为我们能从中获取很多实践经验，也可以拓宽视野。</p><p>不过，我们也要时刻提醒自己，不要像刘姥姥进大观园，眉毛胡子一把抓，最后可能收效甚微，一定要把握好“量”和“度”。这个其实和健身跑步一个道理，得一步步来，毕竟我们总不能直接练习十项全能。</p><p>最后，还有一个方法，就是<strong>学习网络</strong>，我在工作中经常碰到很多奇奇怪怪的问题，而我一直觉得，网络排查不是一个网络工程师或者一个纯开发就能单独搞定的事，所以我就想深入做一下网络编程、网络原理、网络排查。所以自然而然地，我就开始学习胜辉老师的《网络排查案例课》，收获颇多。</p><h2>我是怎么学习这门课的？</h2><p>要我来说的话，这门《网络排查案例课》，广大运维同胞可太值得看了。毕竟，如果系统出了问题，我们总不能只会“重启大法”吧。胜辉老师讲解的各种案例真可谓是惊涛骇浪，如果这些我们都见过了，那后面我们还能害怕遇到啥问题呢。</p><p>我在工作中出的问题最多、用的最多的，其实还是UDP的问题，平时因为涉及到音视频传输，会遇到各种流媒体协议导致的传输问题。而TCP的问题相对少一些，不过我为了拓宽视野就来看胜辉老师的课了。胜辉老师真的是苦口婆心，手把手从基本的抓包入手，帮我们大大降低了学习门槛，即使是新手小白理解起来也没什么问题。而且后半部分的课程，也都是我现在项目最常用的内容，比如Nginx问题、HTTP状态码问题、重传问题、丢包问题、负载均衡等等。</p><p>在学习这门课的过程中，也让我在实际项目上有了新的收获。比如，我们之前项目碰到过下级服务和上级服务之间使用SIP协议交互（基于UDP的SIP）。下级服务给上级服务的消息一般正常，但是上级服务回复的消息会不定期消失。当时网络的同事说两级服务之间是双线，一条联通、一条电信。后来禁用了一条就恢复正常了。这和老师在第23讲当中说的情况就有相似之处，多链路时某些特有的链路出问题，导致了表面上看似无规则的出错，而我们根据详尽的链路分析，其实就会发现问题所在。</p><h2>写在最后</h2><p>实际上，对于一个日常要为项目排查问题的开发来说，<strong>学习新知识应该是一种必备素质和要求</strong>。现在的技术发展日新月异，很多公司的程序也从普通机房迁到了云上。那我们就只管使用程序吗？遇到问题只管让现场重启吗？其实不太可能。日常的抓包排查网络、看日志、查代码、查接口、分析问题，如果单靠工作上的点滴积累，恐怕会跟不上时代的变迁了。</p><p>实际项目出问题就要解决问题，但不是什么时候和我们配合的团队都是专业团队，也不是什么时候和我们配合的人都是负责任的人，也不是什么时候负责协调的人什么都懂。那么需要的就是，我们有更专业的知识和更丰富的经历。而丰富的知识和经历，我们在这里就可以找到答案。行百里者半九十，希望我们一起加油、一起学习、一起进步！</p>
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
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q3auHgzwzM7zuDYFIutbSPc4eEtcMhdNBTI1FRR7q0xrGh2X1QdiaNxvAV31HcRUsjPWLaaWftqgwTnVoiaica8Nw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>胜辉（大V）</span>
  </div>
  <div class="_2_QraFYR_0">我也深有同感，每个人天赋、背景各不相同，但只要能一直学习，一定能一直进步。做到今天比昨天更好，相信明天比今天更好。这个好，既可以是外界的肯定，更可以是内心的富足。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-01 23:36:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/b4/94/2796de72.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>追风筝的人</span>
  </div>
  <div class="_2_QraFYR_0">行百里者半九十</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 一起加油~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-30 13:16:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/39/f9/b2fe7b63.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>King-ZJ</span>
  </div>
  <div class="_2_QraFYR_0">赞</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-30 22:14:35</div>
  </div>
</div>
</div>
</li>
</ul>