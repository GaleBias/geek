<p><video poster="https://static001.geekbang.org/resource/image/b8/a2/b8e479499551550984792f338043a8a2.jpg" preload="none" controls=""><source src="https://media001.geekbang.org/customerTrans/fe4a99b62946f2c31c2095c167b26f9c/32bb5df8-16d13f123cf-0000-0000-01d-dbacd.mp4" type="video/mp4"><source src="https://media001.geekbang.org/71f11392644a4bd19a04901804f3faa2/b54cb30cb2b648d1a3e6392ecc351626-e6d62449b88ad07255d04f874d679181-sd.m3u8" type="application/x-mpegURL"><source src="https://media001.geekbang.org/71f11392644a4bd19a04901804f3faa2/b54cb30cb2b648d1a3e6392ecc351626-32e46e96bebe336fa79a9419934ca38a-hd.m3u8" type="application/x-mpegURL"></video></p><p>你好，我是温铭。</p><p>今天的内容，我特意安排成了视频的形式来讲解。不过，在你看视频之前，我想先问你这么几个问题：</p><ul>
<li>在真实的项目中，你会配置 nginx.conf，以便和 Lua 代码联动吗？</li>
<li>你清楚 OpenResty 的代码结构该如何组织吗？</li>
</ul><p>这两个问题，也是今天视频课要解决的核心内容，希望你可以先自己思考一下，并带着问题来学习今天的视频内容。</p><p>同时，我会给出相应的文字介绍，方便你在听完视频内容后，及时总结与复习。下面是今天这节课的文字介绍部分。</p><h2>今日核心</h2><p><a href="https://github.com/openresty/opm/">opm</a> 是 OpenResty 中为数不多的网站类项目，而里面的代码，基本上是由 OpenResty 的作者亲自操刀完成的。</p><p>很多 OpenResty 的使用者并不清楚，如何在真实的项目中去配置 nginx.conf， 以及如何组织 Lua 的代码结构。确实，在这方面可以参考的开源项目并不多，给学习使用带了不小的阻力。</p><p>不过，借助今天的这个项目，你就可以克服这一点了。你将会熟悉一个OpenResty 项目的结构和开发流程，还能看到 OpenResty 的作者是如何编写业务类 Lua 代码的。</p><p>opm 还涉及到数据库的操作，它后台数据的储存，使用的是PostgreSQL ，你可以顺便了解下 OpenResty 和数据库是如何交互的。</p><!-- [[[read_end]]] --><p>除此之外，这个项目还涉及到一些简单的性能优化，也是为了后面专门设立的性能优化内容做个铺垫。</p><p>最后，浏览完 opm 这个项目后，你可以自行看下另外一个类似的项目，那就是 OpenResty 的官方网站：<a href="https://github.com/openresty/openresty.org">https://github.com/openresty/openresty.org</a>。</p><h2>课件参考</h2><p>今天的课件已经上传到了我的GitHub上，你可以自己下载学习。</p><p>链接如下：<a href="https://github.com/iresty/geektime-slides">https://github.com/iresty/geektime-slides</a></p><p>如果有不清楚的地方，你可以在留言区提问，另也可以在留言区分享你的学习心得。期待与你的对话，也欢迎你把这篇文章分享给你的同事、朋友，我们一起交流、一起进步。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/30/8a/b5ca7286.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>业余草</span>
  </div>
  <div class="_2_QraFYR_0">讲师是在秀发际线😊</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: ：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-06 14:08:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/0b/e8/1deb2efc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>洁</span>
  </div>
  <div class="_2_QraFYR_0">lua_package_path &quot;$prefix&#47;lua&#47;?.lua;$prefix&#47;lua&#47;vendor&#47;?.lua;;&quot;;对于这个路径的$prefix还是有一点不太理解，可以在具体一点吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: $prefix 就是 nginx 启动时候 -p 后面的路径。比如：nginx -p &#47;usr&#47;local&#47;openresty, 那么 $prefix 的值就是  &#47;usr&#47;local&#47;openresty</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-05 14:05:22</div>
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
  <div class="_2_QraFYR_0">lua_package_path &quot;$prefix&#47;lua&#47;?.lua;$prefix&#47;lua&#47;vendor&#47;?.lua;;&quot;;<br><br>这个后面为什么有两个 分号呢 ？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 两个分号的意思是默认的查找路径</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-05 08:17:49</div>
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
  <div class="_2_QraFYR_0">变量前面要加local，函数前面是不是也应该加：local function _M.do_upload() end</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-16 17:05:18</div>
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
  <div class="_2_QraFYR_0">1.需要在lua_package_path中配置lua的源码路径<br>2.参照opm和openresty.org，源码结构均为util&#47;,conf&#47;,templates&#47;,lua&#47;<br><br>老师，学习openresty需要对nginx了解到什么程度啊</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 后面有 nginx 的基础知识章节介绍，对于 nginx 能看懂配置，知道它的大概原理就行了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-17 23:01:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/70/67/0c1359c2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>qinsi</span>
  </div>
  <div class="_2_QraFYR_0">luarocks.org是通过Lapis框架用MoonScript写的：https:&#47;&#47;luarocks.org&#47;about</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-06 23:17:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/81/88/1dc092cf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>X.</span>
  </div>
  <div class="_2_QraFYR_0">大佬好，我用OR学习部署orangeAPI网关碰见点问题。<br>我搭建的实验环境是CentOS 7.6.1810，听你的在官网用yum安装的最新版OR没用编译安装（编译安装之前问题更多，，）然后nginx -v resty -v 都可以查看到信息，nginx也起来了，浏览网页也能看到OpenResty<br>启动orange的时候最后一步提示：<br>[INFO] ORANGE_CONF=&#47;usr&#47;local&#47;orange&#47;conf&#47;orange.conf nginx -p &#47;usr&#47;local&#47;orange -c &#47;usr&#47;local&#47;orange&#47;conf&#47;nginx.conf <br>nginx: [error] init_by_lua error: init_by_lua:2: module &#39;orange.orange&#39; not found:<br>1：我的lua_package_path &quot;$prefix&#47;usr&#47;local&#47;lor&#47;?.lua;;$prefix&#47;usr&#47;local&#47;orange&#47;?.lua;;&quot;; 是这么配置的，我看orange需要lor的支持，这个路径是只需要配置lor的lua路径，还是要把orange lua的路径也写上，就是说以后用的lua不止一个，是一起写在这么。<br>2：我看他意思是init_by_lua找不到orange.orange，我的是这样的<br> init_by_lua_block {<br>        local orange = require(&quot;orange.orange&quot;)<br>        local env_orange_conf = os.getenv(&quot;ORANGE_CONF&quot;)<br>        print(string.char(27) .. &quot;[34m&quot; .. &quot;[INFO]&quot; .. string.char(27).. &quot;[0m&quot;, [[the env[ORANGE_CONF] is ]], env_orange_conf)<br><br>        -- Here, you can also use the absolute path, eg: local confige_file = &quot;&#47;home&#47;openresty&#47;orange&#47;conf&#47;orange.conf&quot;<br>        local config_file = env_orange_conf or ngx.config.prefix().. &quot;&#47;conf&#47;orange.conf&quot;<br>        local config, store = orange.init({<br>            config = config_file<br>        })<br>我看你视频里说nginx.conf尽量少配置，我基本没动只改了lua_package_path ，但是orange默认的init_by_lua_block {<br>        local orange = require(&quot;orange.orange&quot;) 这里面不是一个文件，是orange.orange，他有lua的文件在orange的文件夹下还有个orange文件夹 这里面才有Lua文件 应该是启动文件吧，起不来跟这个 . 有关么。 <br>万分感谢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: orange 我不太清楚，建议最好到 orange 的 issue 里面提问。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-17 16:19:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/2f/4c/04441552.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小鱼👍</span>
  </div>
  <div class="_2_QraFYR_0">opm命令报错<br>Can&#39;t locate Digest&#47;MD5.pm in @INC (@INC contains: &#47;usr&#47;local&#47;lib64&#47;perl5 &#47;usr&#47;local&#47;share&#47;perl5 &#47;usr&#47;lib64&#47;perl5&#47;vendor_perl &#47;usr&#47;share&#47;perl5&#47;vendor_perl &#47;usr&#47;lib64&#47;perl5 &#47;usr&#47;share&#47;perl5 .) at &#47;usr&#47;bin&#47;opm line 16.<br>BEGIN failed--compilation aborted at &#47;usr&#47;bin&#47;opm line 16</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 怎么安装的呢？这个是少了 perl 的库，需要用 cpanm 这样的包管理器安装下</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-11 16:28:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83epgKOrnIOAjzXJgb0f0ljTZLeqrMXYaHic1MKQnPbAzxSKgYxd7K2DlqRW8SibTkwV2MAUZ4OlgRnNw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小羊</span>
  </div>
  <div class="_2_QraFYR_0">i = i+1 下标计算 ， 最后的 print 没有听懂。。。  为什么有性能优化？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 大概的原因是跳过了 Lua 层面的字符串拼接，print 函数不仅接受字符串作为参数，也接受数组作为参数。后面性能优化章节还会继续这个话题。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-07 22:10:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/9f/6f/0e341408.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>亞</span>
  </div>
  <div class="_2_QraFYR_0">老师，作为一个刚接触的菜鸟。感觉都不是太理解。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 慢慢来，先有一个大概的印象</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-05 13:20:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/48/61/803d5bbf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>nanyun</span>
  </div>
  <div class="_2_QraFYR_0">视频的效果好很多，赞。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-05 07:45:12</div>
  </div>
</div>
</div>
</li>
</ul>