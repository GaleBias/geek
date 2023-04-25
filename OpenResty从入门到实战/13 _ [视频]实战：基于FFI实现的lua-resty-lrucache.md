<p><video poster="https://static001.geekbang.org/resource/image/6a/f7/6ada085b44eddf37506b25ad188541f7.jpg" preload="none" controls=""><source src="https://media001.geekbang.org/customerTrans/fe4a99b62946f2c31c2095c167b26f9c/30d99c0d-16d14089303-0000-0000-01d-dbacd.mp4" type="video/mp4"><source src="https://media001.geekbang.org/2ce11b32e3e740ff9580185d8c972303/a01ad13390fe4afe8856df5fb5d284a2-f2f547049c69fa0d4502ab36d42ea2fa-sd.m3u8" type="application/x-mpegURL"><source src="https://media001.geekbang.org/2ce11b32e3e740ff9580185d8c972303/a01ad13390fe4afe8856df5fb5d284a2-2528b0077e78173fd8892de4d7b8c96d-hd.m3u8" type="application/x-mpegURL"></video></p><p>你好，我是温铭。</p><p>今天的内容，我同样会以视频的形式来讲解。不过，在你进行视频学习之前，我想先问你这么几个问题：</p><ul>
<li>lua-resty-lrucache 内部最重要的数据结构是什么？</li>
<li>lua-resty-lrucache 有两种 FFI 的实现，我们今天讲的这一种更适合什么场景？</li>
</ul><p>这几个问题，也是今天视频课要解决的核心内容，希望你可以先自己思考一下，并带着问题来学习今天的视频内容。</p><p>同时，我会给出相应的文字介绍，方便你在听完视频内容后，及时总结与复习。下面是今天这节课的文字介绍部分。</p><h2>今日核心</h2><p><a href="https://github.com/openresty/lua-resty-lrucache">lua-resty-lrucache</a> 是一个使用 LuaJIT FFI 实现的 LRU 缓存库，可以在 worker 内缓存各种类型的数据。功能与之类似的是 shared dict，但 shared dict 只能存储字符串类型的数据。在大多数实际情况下，这两种缓存是配合在一起使用的——lrucache 作为一级缓存，shared dict 作为二级缓存。</p><p>lrucache 的实现，并没有涉及到 OpenResty 的 Lua API。所以，即使你以前没有用过OpenResty，也可以通过这个项目来学习如何使用 LuaJIT 的 FFI。</p><p>lrucache 仓库中包含了两种实现方案，一种是使用 Lua table 来实现缓存，另外一种则是使用 hash 表来实现。前者更适合命中率高的情况，后者适合命中率低的情况。两个方案没有哪个更好，要看你的线上环境更适合哪一个。</p><!-- [[[read_end]]] --><p>通过今天这个项目，你可以弄清楚要如何使用 FFI，并了解一个完整的 lua-resty 库应该包括哪些必要的内容。当然，我顺道也会介绍下 travis 的使用。</p><p>最后，还是想强调一点，在你面对一个陌生的开源项目时，文档和测试案例永远是最好的上手方式。而你后期如果要阅读源码，也不要先去抠细节，而是应该先去看主要的数据结构，围绕重点逐层深入。</p><h2>课件参考</h2><p>今天的课件已经上传到了我的GitHub上，你可以自己下载学习。</p><p>链接如下：<a href="https://github.com/iresty/geektime-slides">https://github.com/iresty/geektime-slides</a></p><p>如果有不清楚的地方，你可以在留言区提问，另也可以在留言区分享你的学习心得。期待与你的对话，也欢迎你把这篇文章分享给你的同事、朋友，我们一起交流、一起进步。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/9b/a8/6a391c66.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Leon📷</span>
  </div>
  <div class="_2_QraFYR_0">专栏订阅人好少，跟陶辉老师的nginx一样，到后面都没什么人留言了，曲高和寡，不过留下来的都是未来的大牛，学好了未来在工作和求职过程中多了一份竞争力</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-30 06:49:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/c6/ed/89a2dc13.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>丢了个丢丢丢</span>
  </div>
  <div class="_2_QraFYR_0">老师讲的好好哦！！！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-24 10:41:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/9e/c8/d261f700.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>yswang</span>
  </div>
  <div class="_2_QraFYR_0">温大，我之前使用 openresty + ffmpeg 做了一个音频格式换行http服务，我使用先写一个 shell 转换脚本文件，例如：audio_converter.sh (sh文件中调用 ffmepg -i 转换命令)，然后使用 lua 的 os.execute(&quot;&#47;bin&#47;sh audio_converter.sh  a.mp3  b.ogg&quot;) 来调用 shell 脚本文件间接调用  ffmpeg 指令进行转换。 今天看到了 ffi ，ffi 可以调用 C 函数，请问，ffi 可以直接调用 ffmpeg 指令吗？如果可以的话，我就可以省去写一个 shell 文件来间接调用了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 有一个 ffmpeg 的库：https:&#47;&#47;github.com&#47;daurnimator&#47;ffmpeg-lua-ffi，你可以试下是否满足需求</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-26 14:59:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/95/11/eb431e52.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>沈康</span>
  </div>
  <div class="_2_QraFYR_0">HAHA，刚学完陶辉老师的nginx，跃跃欲试啊</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-05 22:53:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/fd/1a/be30769f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>dhf</span>
  </div>
  <div class="_2_QraFYR_0">get方法里维护lru的时候，不需要考虑线程安全吗</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-23 20:12:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/1e/c2/7e67107f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>微众_廖卫军</span>
  </div>
  <div class="_2_QraFYR_0">温铭老师，您好！ FFI调用自己封装的函数库，ffi.load(name[,global])，是否只能是动态库？不能调用静态库吗？  谢谢！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-23 13:14:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/z5OUcd5RD7gMIevhHORHFcQsXNlZ4UxIU2jS6UwyUIM0w0nXI57nZKCXsibmN2o71WLtg5YXuJKeMNkItEzEaGg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>chrisforbt</span>
  </div>
  <div class="_2_QraFYR_0">老师，我想问个问题。我计划将用户请求的参数+路径+方式做为key，把数据存在lru-cache、share dict以及redis中，mysql数据源更新了，我就得将缓存全删了。那我是应该把mysql的数据源和前面生成的key关联起来，一旦更新了数据源，就去找相关的key，将其删除吗？还有更佳的缓存更新方式吗，期待您的回复。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你这种是主动更新，MySQL 数据变化后主动通知前面的缓存。主动更新的话，也不用主动去删除缓存，而是在 key 里面加入一个版本号，数据变化后版本号变了，那么在缓存中就查找不到数据，这时候就回源到 MySQL 了。<br>也可以被动更新，根据你的业务特性，给缓存设置一个过期时间，定期的去数据库查询。<br>第一种版本号的方法是比较合适的，当然实现起来比被动更新复杂一些。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-15 21:22:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJrvZAOiahN7YnSviaDANNu1uHIvQEEE8icl9libuibhuIZgEOt28KG2jKYqu6uRNILicibb8jM6icHicUXj0A/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>emen</span>
  </div>
  <div class="_2_QraFYR_0">温铭老师您好，关于lrucache在使用过程中有个疑惑烦请老师看看。<br>openresty环境：1个master进程、X个worker进程<br>在init_worker阶段中注册了缓存信息如：cache.set(&quot;key1&quot;,&quot;val1&quot;)<br>在content阶段修改了缓存信息：cache.set(&quot;key1&quot;,&quot;val2&quot;)<br>此操作是否仅为更新了当前worker的缓存信息，而无法跨worker更新。<br>如需完成跨worker更新应如何处理呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 改用 shared dict，或者是使用 lua-resty-worker-event 这样的库</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-10 15:48:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/99/f0/d9343049.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>星亦辰</span>
  </div>
  <div class="_2_QraFYR_0">请教老师一个问题。<br><br>我用c语言实现了跨平台的so类库。 用ffi.load 来加载。 <br>那么，在openrestry 中会有几个 so 函数的实例呢？ <br>按照操作系统的知识,共享库在系统中只有一个实例。<br><br>如果只有一个，那么我们在不同的worker 中调用so 里的函数中，对于一个so 内的资源是否存在冲突呢？ </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: OpenResty 中的 worker 是互相独立的，如果你在 worker 中使用 ffi.load 来加载 so，那么有几个 worker 进程，就会有几个 so 实例。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-04 11:52:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/f0/21/104b9565.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小飞哥 ‍超級會員</span>
  </div>
  <div class="_2_QraFYR_0">都有视频了，温老师，能否现场演示一下， 让我们见识 openresty 的复杂性。 一些文档库 官网描述等，能否现场打开了解下。  我们自己打开后，就没有然后了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我的理解，OpenResty 的复杂性在于各个子项目之间的配合，还是需要有特定的场景和业务需求来驱动，才会更有感觉。你可以参与到 OpenResty 相关的开源项目中来，切身体会下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-02 22:21:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/93/14/7e078b32.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>. 。o O 〇</span>
  </div>
  <div class="_2_QraFYR_0">老师好，我碰到一个问题，我用的lrucache，会卡在get set那，并且worker进程cpu100%，整个worker不工作。换成lrucache.pureffi就好了，不知道老师碰到过没</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个没有遇到过。是否是这个 bug：https:&#47;&#47;github.com&#47;openresty&#47;luajit2&#47;issues&#47;42？ 用最新的 OpenResty 版本是否可以重现呢？<br>如果还是有问题，欢迎给官方提交 issue</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-26 08:03:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/da/a9/8fdfff70.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>冰沁宇诺</span>
  </div>
  <div class="_2_QraFYR_0">老师，如果我想从数据库中初始化一些基本不变的数据到shared_dict中，应该怎么做，init_by_lua 阶段可以吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以的，在这个阶段启动一个 timer 去读取数据库，然后放到 shared dict 中。但要注意，这个数据是可以丢失的，代码中要处理这种异常。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-24 18:20:12</div>
  </div>
</div>
</div>
</li>
</ul>