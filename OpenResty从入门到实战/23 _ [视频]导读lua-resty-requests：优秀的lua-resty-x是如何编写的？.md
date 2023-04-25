<p><video poster="https://static001.geekbang.org/resource/image/65/c2/6565cf0a87645948ef66c547192db3c2.jpg" preload="none" controls=""><source src="https://media001.geekbang.org/customerTrans/fe4a99b62946f2c31c2095c167b26f9c/3d0d7df0-16ce81ba96a-0000-0000-01d-dbacd.mp4" type="video/mp4"><source src="https://media001.geekbang.org/ed7db4a5658b4ed59f225967841c31f8/e7f659c5c1864bb3bde42a4c200eab4b-d5090411129213bc3a3a2d388e0daa0b-sd.m3u8" type="application/x-mpegURL"><source src="https://media001.geekbang.org/ed7db4a5658b4ed59f225967841c31f8/e7f659c5c1864bb3bde42a4c200eab4b-3d9bf901ccefc4abb20c7469df8134d4-hd.m3u8" type="application/x-mpegURL"></video></p><p>你好，我是温铭。</p><p>今天的内容，我同样会以视频的形式来讲解。老规矩，在你进行视频学习之前，先问你这么几个问题：</p><ul>
<li>面对多个相同功能的 lua-resty 库，我们应该从哪些方面来选择？</li>
<li>如何来组织一个 lua-resty 的结构？</li>
</ul><p>这几个问题，也是今天视频课要解决的核心内容，希望你可以先自己思考一下，并带着问题来学习今天的视频内容。</p><p>同时，我会给出相应的文字介绍，方便你在听完视频内容后，及时总结与复习。下面是今天这节课的文字介绍部分。</p><h2>今日核心</h2><p>前面我们介绍过的 lua-resty 库都是官方自带的，但在 HTTP client 这个最常用的库上，官方并没有。这时候，我们就得自己来选择一个优秀的第三方库了。</p><p>那么，如何在众多的 lua-resty HTTP client 中，选择一个最好、最适合自己的第三方库呢？</p><p>这时候，你就需要综合考虑活跃度、作者、测试覆盖度、接口封装等各方面的因素了。我最后选择的是 lua-resty-requests（<a href="https://github.com/tokers/lua-resty-requests">https://github.com/tokers/lua-resty-requests</a>），它是由又拍云的工程师 tokers 贡献的，我个人很喜欢它的接口风格，也推荐给你。</p><p>在视频中我会从最简单的 get 接口入手，结合文档、测试案例和源码，来逐步展开。你可以看到一个优秀的 lua-resty 库是如何编写的，有哪些可以借鉴的地方。</p><!-- [[[read_end]]] --><h2>课件参考</h2><p>今天的课件已经上传到了我的GitHub上，你可以自己下载学习。</p><p>链接如下：<a href="https://github.com/iresty/geektime-slides">https://github.com/iresty/geektime-slides</a></p><p>如果有不清楚的地方，你可以在留言区提问，另也可以在留言区分享你的学习心得。期待与你的对话，也欢迎你把这篇文章分享给你的同事、朋友，我们一起交流、一起进步。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/49/f6/ef3e5c81.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>shengsheng</span>
  </div>
  <div class="_2_QraFYR_0">请问有没比较方便的调试过程，例如一些ide， 每次写完都要重启nginx才能看到效果。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: vscode 里面有付费的插件，但我并没有用过。我都是用最原始的 log 和 say 来调试的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-24 14:21:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_e553fa</span>
  </div>
  <div class="_2_QraFYR_0">你们都用什么工具写openresty  代码呢？<br>没有找到任何文章任何人说这个事情啊<br>我现在都是vim里面写任何调试。真的很麻烦</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: OpenResty 的作者使用的是 vim，我用的是 vs code</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-17 09:28:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/57/f6/2c7ac1ad.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Peter</span>
  </div>
  <div class="_2_QraFYR_0">老师lua_package_path 和 LUA_PATH 这两个系统变量好像没讲过啊，具体是什么意思，起什么作用</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-18 07:30:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/42/c5/7913cdb0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>John</span>
  </div>
  <div class="_2_QraFYR_0">老师，请问，resty -e 测试时 如何加载nginx.conf 中的环境配置，比如 lua_package_path</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: resty 有一个 -I 的指令：<br>-I DIR              Add dir to the search paths for Lua libraries.<br>可以添加 lua_package_path 的路径。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-17 14:49:42</div>
  </div>
</div>
</div>
</li>
</ul>