<p><video poster="https://static001.geekbang.org/resource/image/0b/02/0bf3ba61bafc3a97514996c701c99e02.jpg" preload="none" controls=""><source src="https://media001.geekbang.org/customerTrans/fe4a99b62946f2c31c2095c167b26f9c/1d56d7e0-16ce8221564-0000-0000-01d-dbacd.mp4" type="video/mp4"><source src="https://media001.geekbang.org/98c47d3154564eb08b624deb1aa4e260/d32353463ce34201b74a0b3807160f6b-cf761b25fd08f06e963332465b9b0f4c-sd.m3u8" type="application/x-mpegURL"><source src="https://media001.geekbang.org/98c47d3154564eb08b624deb1aa4e260/d32353463ce34201b74a0b3807160f6b-335f34e72825be78cd1db071594c6c08-hd.m3u8" type="application/x-mpegURL"></video></p><p>你好，我是温铭。</p><p>今天是我们专栏中的最后一节视频课了，后面内容仍然以图文形式呈现。老规矩，为了更有针对性地学习，在你进行视频学习之前，我想先问你这么几个问题：</p><ul>
<li>你测试过 OpenResty 程序的性能吗？如何才能科学地找到性能瓶颈？</li>
<li>如何看懂火焰图的信息，并与 Lua 代码相对应呢？</li>
</ul><p>这几个问题，也是今天视频课要解决的核心内容，希望你可以先自己思考一下，并带着问题来学习今天的视频内容。</p><p>同时，我会给出相应的文字介绍，方便你在听完视频内容后，及时总结与复习。下面是今天这节课的文字介绍部分。</p><h2>今日核心</h2><p>今天的视频课，我会用一个开源的小项目来演示一下，如何通过 wrk 和火焰图来优化代码，这个项目地址为：<a href="https://github.com/iresty/lua-performance-demo">https://github.com/iresty/lua-performance-demo</a>。</p><p>视频中的环境是 Ubuntu 16.04，其中的 systemtap 和 wrk 工具，都是使用 apt-get 来安装的，不推荐你用源码来安装。</p><p>这里的demo 有几个不同的版本，我会用 wrk 来压测每一个版本的 qps。同时，在压测过程中，我都会使用 stapxx 来生成火焰图，并用火焰图来指导我们去优化哪一个函数和代码块。</p><p>最后的结果是，我们会看到一个性能提升 10 倍以上的版本，当然，这其中的优化方式，都是在专栏前面课程中提到过的。建议你可以 clone 这个 demo 项目，来复现我在视频中的操作，加深对 wrk、火焰图和性能优化的理解。</p><!-- [[[read_end]]] --><p>要知道，性能优化并不是感性和直觉的判断，而是需要科学的数据来做指导的。这里的数据，不仅仅是指 qps 等最终的性能指标，也包括了用数据来定位具体的瓶颈。</p><h2>课件参考</h2><p>今天的课件已经上传到了我的GitHub上，你可以自己下载学习。</p><p>链接如下：<a href="https://github.com/iresty/geektime-slides">https://github.com/iresty/geektime-slides</a></p><p>如果有不清楚的地方，你可以在留言区提问，另也可以在留言区分享你的学习心得。期待与你的对话，也欢迎你把这篇文章分享给你的同事、朋友，我们一起交流、一起进步。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4d/fd/0aa0e39f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>许童童</span>
  </div>
  <div class="_2_QraFYR_0">老师这个案例还是挺不错的，手把手带你用火焰图做性能优化。<br>过早的优化是万恶这源，还要注意优化和代码可读性间的平衡。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，OpenResty 代码的优化做到极致就容易影响可读性。实际的项目，一般会在上面多做一层封装，把优化的细节隐藏下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-21 16:37:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/da/36/ac0ff6a7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>wusiration</span>
  </div>
  <div class="_2_QraFYR_0">老师的这个案例，基本上把整个性能分析的流程给讲清楚了，周末搭一下环境尝试性能分析一下</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 多动手：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-22 23:32:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/f2/21/00600713.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小侠</span>
  </div>
  <div class="_2_QraFYR_0">火焰图很直观</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-18 12:03:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/79/7e/c38ac02f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>北冥Master</span>
  </div>
  <div class="_2_QraFYR_0">.&#47;samples&#47;lj-lua-stacks.sxx --arg time=5 --skip-badvars -x 2871915 &gt; ~&#47;perf.bt<br>Found exact match for libluajit: &#47;usr&#47;local&#47;openresty&#47;luajit&#47;lib&#47;libluajit-5.1.so.2.1.0<br>In file included from &#47;usr&#47;share&#47;systemtap&#47;runtime&#47;linux&#47;runtime.h:201:0,<br>                 from &#47;usr&#47;share&#47;systemtap&#47;runtime&#47;runtime.h:24,<br>                 from &#47;tmp&#47;stapdiQN45&#47;stap_b7d4c96a12d824bc289ba86f5d42b8c9_43901_src.c:26:<br>&#47;usr&#47;share&#47;systemtap&#47;runtime&#47;linux&#47;access_process_vm.h: In function ‘__access_process_vm_’:<br>&#47;usr&#47;share&#47;systemtap&#47;runtime&#47;linux&#47;access_process_vm.h:24:8: error: implicit declaration of function ‘get_task_mm’ [-Werror=implicit-function-declaration]<br>   mm = get_task_mm (tsk);<br>        ^~~~~~~~~~~<br>&#47;usr&#47;share&#47;systemtap&#47;runtime&#47;linux&#47;access_process_vm.h:24:6: error: assignment makes pointer from integer without a cast [-Werror=int-conversion]<br>   mm = get_task_mm (tsk);<br>      ^<br>&#47;usr&#47;share&#47;systemtap&#47;runtime&#47;linux&#47;access_process_vm.h:35:29: error: passing argument 1 of ‘get_user_pages’ makes integer from pointer without a cast [-Werror=int-conversion]<br>       ret = get_user_pages (tsk, mm, addr, 1, write, 1, &amp;page, &amp;vma);<br>                             ^~~<br>In file included from .&#47;include&#47;linux&#47;pid_namespace.h:7:0,<br>                 from .&#47;include&#47;linux&#47;ptrace.h:10,<br>                 from .&#47;include&#47;linux&#47;ftrace.h:14,<br>                 from .&#47;include&#47;linux&#47;kprobes.h:42,<br>                 from &#47;usr&#47;share&#47;systemtap&#47;runtime&#47;linux&#47;runtime.h:21,</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-04 22:26:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/79/7e/c38ac02f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>北冥Master</span>
  </div>
  <div class="_2_QraFYR_0">我的环境执行lj-lua-stackxx 报错，debian9环境，systemtap应该装什么版本？我装的2.6</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-04 22:23:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/59/28/62ee741f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>埼玉</span>
  </div>
  <div class="_2_QraFYR_0">老师，我哪里操作有问题吗？<br><br>报错信息：<br><br>Found exact match for libluajit: &#47;usr&#47;local&#47;openresty&#47;luajit&#47;lib&#47;libluajit-5.1.so.2.1.0<br>WARNING: cannot find module &#47;usr&#47;local&#47;openresty&#47;nginx&#47;sbin&#47;nginx debuginfo: No DWARF information found [man warning::debuginfo]<br>WARNING: cannot find module &#47;usr&#47;local&#47;openresty&#47;luajit&#47;lib&#47;libluajit-5.1.so.2.1.0 debuginfo: No DWARF information found [man warning::debuginfo]<br>semantic error: while processing function luajit_G<br><br>semantic error: type definition &#39;lua_State&#39; not found in &#39;&#47;usr&#47;local&#47;openresty&#47;luajit&#47;lib&#47;libluajit-5.1.so.2.1.0&#39;: operator &#39;@cast&#39; at stapxx-0fF6qb3V&#47;luajit.stp:162:12<br>        source:     return @cast(L, &quot;lua_State&quot;, &quot;&#47;usr&#47;local&#47;openresty&#47;luajit&#47;lib&#47;libluajit-5.1.so.2.1.0&quot;)-&gt;glref-&gt;ptr32<br>                           ^<br><br>Pass 2: analysis failed.  [man error::pass2]<br>Number of similar warning messages suppressed: 634.<br>Rerun with -v to see them.<br>Tip: &#47;usr&#47;share&#47;doc&#47;systemtap&#47;README.Debian should help you get started.<br>ERROR: No stack counts found<br><br>环境：<br>Ubuntu 16.04.6<br>openresty&#47;1.15.8.2</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: OpenResty 的 1.15.8 中开启了 LuaJIT64 模式，火焰图的工具一直没有跟着升级，所以存在不兼容的问题。有两个方法可以解决：<br>1. 使用 OpenResty 1.13 的版本；<br>2. 自己编译 OpenResty 1.15.8，把LuaJIT 64 模式关闭。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-15 22:51:03</div>
  </div>
</div>
</div>
</li>
</ul>