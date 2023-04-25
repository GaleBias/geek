<audio title="04 _ 如何管理第三方包？从包管理工具luarocks和opm说起" src="https://static001.geekbang.org/resource/audio/b3/96/b3c3cad0a84d56ab548a91a66b35ad96.mp3" controls="controls"></audio> 
<p>你好，我是温铭。</p><p>在上一节中，我们大概了解了下 OpenResty 官方的一些项目。不过，如果我们把 OpenResty 用于生产环境，显然，OpenResty 安装包自带的这些库是远远不够的，比如没有 lua-resty 库来发起HTTP请求，也没有办法和 Kafka 交互。</p><p>那么应该怎么办呢？本节我们就来一起了解下，应该从什么渠道来找到这些第三方库。</p><p>这里，我再次强调下，OpenResty 并不是 NGINX 的 fork，也不是在 NGINX 的基础上加了一些常用库重新打包，而<strong>只是把 NGINX 当作底层的网络库来使用</strong>。</p><p>当你使用 NGINX 的时候，是不会想着如何发起自定义的HTTP请求，以及如何与 Kafka 交互的。而在 OpenResty 的世界中，由于 cosocket 的存在，开发者可以轻松地写出 lua-resty-http 和 lua-resty-kafka ，来处理这类需求，就像你用 Python、PHP 这类的开发语言一样。</p><p>另外，还有一个建议告诉你：你不应该使用任何 Lua 世界的库来解决上述问题，而是应该使用 cosocket 的 lua-resty-* 库。<strong>Lua 世界的库很可能会带来阻塞</strong>，让原本高性能的服务，直接下降几个数量级。这是 OpenResty 初学者的常见错误，而且并不容易觉察到。</p><!-- [[[read_end]]] --><p>那我们怎么找到这些非阻塞的 lua-resty-* 库呢？接下来，我来为你介绍下面几种途径。</p><h2><strong>OPM</strong></h2><p><a href="https://opm.openresty.org/">OPM</a>（OpenResty Package Manager）是 OpenResty 自带的包管理器，在你安装好 OpenResty 之后，就可以直接使用。我们可以试着去找找发送 http 请求的库 <code>$ opm search http</code></p><p>第一次查询可能会比较慢，需要几秒钟的时间。opm.openresty.org 会从 PostgreSQL 数据库中做一次查询，并把结果缓存一段时间。search 具体的返回结果比较长，我们这里只看下第一条返回值：</p><pre><code>openresty/lua-resty-upload    			Streaming reader and parser for HTTP file uploading based on ngx_lua cosocket
</code></pre><p>呃，看到这个结果，你可能会疑惑：这个 lua-resty-upload 包和发送 http 有什么关系呢？</p><p>原来，OPM做搜索的时候，是用后面的关键字同时搜索了包的名字和包的简介。这也是为什么上面的搜索会持续几秒，因为它在 PostgreSQL 里面做了字符串的全文搜索。</p><p>不过，不管怎么说，这个返回并不友好。让我们修改下关键字，重新搜索下：</p><pre><code>$ opm search lua-resty-http
ledgetech/lua-resty-http                          Lua HTTP client cosocket driver for OpenResty/ngx_lua
pintsized/lua-resty-http                          Lua HTTP client cosocket driver for OpenResty/ngx_lua
agentzh/lua-resty-http                            Lua HTTP client cosocket driver for OpenResty/ngx_lua
</code></pre><p>其实，在 OpenResty 世界中，如果你使用 cosocket 实现了一个包，那么就要使用 lua-resty- 这个前缀，算是一个不成文的规定。</p><p>回过头来看刚刚的搜索结果，OPM 使用了贡献者的 GitHub 仓库地址作为包名，即 GitHub ID / repo name。上面返回了三个 lua-resty-http 第三方库，我们应该选择哪一个呢？</p><p>眼尖的你，可能已经发现了 agentzh 这个 ID，没错，这就是 OpenResty 作者春哥本人。在选择这个包之前，我们看下它的 star 数和最后更新时间：只有十几个 star，最后一次更新是在 2016 年。很明显，这是个被放弃的坑。更深入地看下，pintsized/lua-resty-http 和 ledgetech/lua-resty-http 其实指向了同一个仓库。所以，不管你选哪个都是一样的。</p><p>同时 <a href="https://opm.openresty.org/">OPM 的网站</a> 也相对简单，没有提供包的下载次数，也没有这个包的依赖关系。你需要花费更多的时间，来甄别出到底使用哪些 lua-resty 库才是正确的选择，而这些本应该是维护者的事情。</p><h2><strong>LUAROCKS</strong></h2><p><a href="https://luarocks.org">LuaRocks</a> 是 OpenResty 世界的另一个包管理器，诞生在 OPM 之前。不同于 OPM 里只包含 OpenResty 相关的包，LuaRocks 里面还包含 Lua 世界的库。举个例子，LuaRocks 里面的 LuaSQL-MySQL，就是 Lua 世界中连接 MySQL 的包，并不能用在 OpenResty 中。</p><p>还是以HTTP库为例，我们尝试用 LuaRocks 来试一试查找：</p><pre><code>$ luarocks search http
</code></pre><p>你可以看到，也是返回了一大堆包。</p><p>我们不妨再换个关键字：</p><pre><code>$ luarocks search lua-resty-http
</code></pre><p>这次只返回了一个包。我们可以到 LuaRocks 的网站上，去看看<a href="https://luarocks.org/modules/pintsized/lua-resty-http">这个包的详细信息</a>，下面是网站页面的截图：</p><p><img src="https://static001.geekbang.org/resource/image/ba/95/ba5cbaae9a7a9ab1fbd05099dc7e9695.jpg?wh=1170*838" alt=""></p><p>这里面包含了作者、License、GitHub 地址、下载次数、功能简介、历史版本、依赖等。和 OPM 不同的是，LuaRocks 并没有直接使用 GitHub 的用户信息，而是需要开发者单独在 LuaRocks 上进行注册。</p><p>其实，开源的 API 网关项目 Kong，就是使用 LuaRocks 来进行包的管理，并且还把 LuaRocks 的作者收归麾下。我们接着就来简单看下，Kong 的包管理配置是怎么写的。</p><p>目前 Kong 的最新版本是 1.1.1， 你可以在 <a href="https://github.com/Kong/kong">https://github.com/Kong/kong</a> 的项目下找到最新的 .rockspec 后缀的文件。</p><pre><code>package = &quot;kong&quot;
version = &quot;1.1.1-0&quot;
supported_platforms = {&quot;linux&quot;, &quot;macosx&quot;}
source = {
  url = &quot;git://github.com/Kong/kong&quot;,
  tag = &quot;1.1.1&quot;
}
description = {
  summary = &quot;Kong is a scalable and customizable API Management Layer built on top of Nginx.&quot;,
  homepage = &quot;https://konghq.com&quot;,
  license = &quot;Apache 2.0&quot;
}
dependencies = {
  &quot;inspect == 3.1.1&quot;,
  &quot;luasec == 0.7&quot;,
  &quot;luasocket == 3.0-rc1&quot;,
  &quot;penlight == 1.5.4&quot;,
  &quot;lua-resty-http == 0.13&quot;,
  &quot;lua-resty-jit-uuid == 0.0.7&quot;,
  &quot;multipart == 0.5.5&quot;,
  &quot;version == 1.0.1&quot;,
  &quot;kong-lapis == 1.6.0.1&quot;,
  &quot;lua-cassandra == 1.3.4&quot;,
  &quot;pgmoon == 1.9.0&quot;,
  &quot;luatz == 0.3&quot;,
  &quot;http == 0.3&quot;,
  &quot;lua_system_constants == 0.1.3&quot;,
  &quot;lyaml == 6.2.3&quot;,
  &quot;lua-resty-iputils == 0.3.0&quot;,
  &quot;luaossl == 20181207&quot;,
  &quot;luasyslog == 1.0.0&quot;,
  &quot;lua_pack == 1.0.5&quot;,
  &quot;lua-resty-dns-client == 3.0.2&quot;,
  &quot;lua-resty-worker-events == 0.3.3&quot;,
  &quot;lua-resty-mediador == 0.1.2&quot;,
  &quot;lua-resty-healthcheck == 0.6.0&quot;,
  &quot;lua-resty-cookie == 0.1.0&quot;,
  &quot;lua-resty-mlcache == 2.3.0&quot;,
......
</code></pre><p>通过文件你可以看到，依赖项里面掺杂了 lua-resty 库和纯 Lua 世界的库，使用 OPM 只能部分安装这些依赖项。写好配置后，使用 luarocks 的 upload 命令把这个配置上传，用户就可以用 LuaRocks 来下载并安装 Kong 了。</p><p>另外，在 OpenResty 中，除了 Lua 代码外，我们还经常会调用 C 代码，这时候就需要编译才能使用。LuaRocks 是支持这么做的，你可以在 rockspec 文件中，指定 C 源码的路径和名称，这样LuaRocks 就会帮你本地编译。而 OPM 暂时还不支持这种特性。</p><p>不过，需要注意的是，OPM 和 LuaRocks 都不支持私有包。</p><h2><strong>AWESOME-RESTY</strong></h2><p>讲了这么多包管理的内容，其实呢，即使有了 OPM 和 LuaRocks，对于 OpenResty 的 lua-resty 包，我们还是管中窥豹的状态。到底有没有地方可以让我们一览全貌呢？</p><p>当然是有的，<a href="https://github.com/bungle/awesome-resty">awesome-resty</a> 这个项目，就维护了几乎所有 OpenResty 可用的包，并且都分门别类地整理好了。当你不确定是否存在适合的第三方包时，来这里“按图索骥”，可以说是最好的办法。</p><p>还是以HTTP库为例， 在awesome-resty 中，它自然是属于 <a href="https://github.com/bungle/awesome-resty#networking">networking</a> 分类：</p><pre><code>lua-resty-http by @pintsized — Lua HTTP client cosocket driver for OpenResty / ngx_lua
lua-resty-http by @liseen — Lua http client driver for the ngx_lua based on the cosocket API
lua-resty-http by @DorianGray — Lua HTTP client driver for ngx_lua based on the cosocket API
lua-resty-http-simple — Simple Lua HTTP client driver for ngx_lua
lua-resty-httpipe — Lua HTTP client cosocket driver for OpenResty / ngx_lua
lua-resty-httpclient — Nonblocking Lua HTTP Client library for aLiLua &amp; ngx_lua
lua-httpcli-resty — Lua HTTP client module for OpenResty
lua-resty-requests — Yet Another HTTP Library for OpenResty
</code></pre><p>我们看到，这里有 8 个 lua-resty-http 的第三方库。对比一下前面的结果，我们使用 OPM 只找到 2 个，而LuaRocks 里面更是只有 1 个。不过，如果你是选择困难症，请直接使用第一个，它和 LuaRocks 中的是同一个。</p><p>而对于愿意尝试的工程师，我更推荐你用最后一个库： <a href="https://github.com/tokers/lua-resty-requests">lua-resty-requests</a>，它是人类更友好的 HTTP访问库，接口风格与 Python 中大名鼎鼎的 <a href="http://docs.python-requests.org/en/master/">Requests</a> 一致。如果你跟我一样是一个 Python 爱好者，一定会喜欢上 lua-resty-requests。这个库的作者是 OpenResty 社区中活跃的 tokers，因此你可以放心使用。</p><p>必须要承认，OpenResty 现有的第三方库并不完善，所以，如果你在 awesome-resty 中没有找到你需要的库，那就需要你自己来实现，比如 OpenResty 一直没有访问 Oracle  或者 SQLServer 的 lua-rsety 库。</p><h2>写在最后</h2><p>一个开源项目想要健康地发展壮大，不仅需要有硬核的技术、完善的文档和完整的测试，还需要带动更多的开发者和公司一起加入进来，形成一个生态。正如 Apache 基金会的名言：社区胜于代码。</p><p>还是那句话，想把 OpenResty 代码写好，一点儿也不简单。OpenResty 还没有系统的学习资料，也没有官方的代码指南，很多的优化点的确已经写在了开源项目中，但大多数开发者却是知其然而不知其所以然。这也是我这个专栏的目的所在，希望你学习完之后，可以写出更高效的 OpenResty 代码，也可以更容易地参与到 OpenResty 相关的开源项目中来。</p><p>你是如何看待 OpenResty 生态的呢？欢迎留言我们一起聊聊，也欢迎你把这篇文章转发给你的同事、朋友，一起在交流中进步。</p><p></p>
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
  <div class="_2_QraFYR_0">OpenResty 一直没有访问 Oracle 或者 SQLServer 的 lua-rsety 库。这是有什么原因吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 因为 Oracle 和 SQLServer 是闭源的商业产品</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-06 11:37:40</div>
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
  <div class="_2_QraFYR_0">以前都是把openresty当做nginx+lua，使用实现一些最简单的功能，都不知道这些包管理器，感谢老师</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-03 09:20:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/8e/a1/c05ab3b3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>咖啡猫</span>
  </div>
  <div class="_2_QraFYR_0">luarocks安装了包后，在nginx.conf应该怎么设置lua_package_path呢，有时候设置了默认搜索路径，也是不生效，尝试将包拷贝到lualib的目录下才能找到</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: luarocks 和 OpenResty 并没有配合和联动，需要你单独在 lua_package_path 中增加 luarocks 安装的路径才行。一般来说，luarocks 会把库安装到 lua5.1 或者 lua5.3 的目录下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-03 00:45:32</div>
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
  <div class="_2_QraFYR_0">Use of LuaRocks with OpenResty is strongly discouraged since OpenResty provides its own package manager, OPM. (https:&#47;&#47;openresty.org&#47;en&#47;using-luarocks.html) 一脸懵逼</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-31 10:46:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/c0/2c/87cc08ce.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>baiyutang</span>
  </div>
  <div class="_2_QraFYR_0">感觉OpenResty比较偏向于运维开发的一部分，因为业务开发比较少关系服务器部署或者性能，性能的话可能大厂会遇到更多问题或者需要定制化的问题。<br>1 很多小厂多仅限于运用作为一个web服务器。<br>2 程序员圈子中业务开发还是相对比较多的？<br>所以，虽然东西是好东西，但是不是每个厂 或者每个人都能玩的起来的。不能像Vuejs或者Golang这些业务开发技术直接做比较。当然，我仍然觉得OpenResty是值得投入的，从职业规划或者个人对软件的理解。我都看好学习好OR.</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，OpenResty 使用广，但不热门。用 OpenResty 开发业务是没问题的，把它仅仅当做 nginx 的替代就有些大材小用了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-05 00:07:52</div>
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
  <div class="_2_QraFYR_0">包管理工具 最好用的应该是 composer 和 npm    包管理工具可以让我们站在前人的肩膀上更快的开发出应用</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: luarocks 相对好用一些</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-03 08:34:43</div>
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
  <div class="_2_QraFYR_0">opm 上的第三方库，还是很少的，功能还有待完善</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: opm 确实不够完善，还要多多加油才行</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-03 08:11:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/51/76/bebe6133.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>TomShine</span>
  </div>
  <div class="_2_QraFYR_0">OpenRestry 的生态看着确实不好，我们可以一起努力</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 众人拾柴火焰高</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-03 10:11:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/e2/d8/f0562ede.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>manatee</span>
  </div>
  <div class="_2_QraFYR_0">打卡，从opm中可以安装opresty相关的第三方包，从luarocks可以安装lua相关的第三方包。想请教一下老师，文中讲的cosocket具体是指什么呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: cosocket 后面会专门讲，你可以简单的认为它是 OpenResty 特有的，用来访问网络的协程技术</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-03 08:06:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/27/81/27c9d811.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>高燕军</span>
  </div>
  <div class="_2_QraFYR_0">我看了这么多，总结出一个道理，任何一个项目能不能成功，就看这个项目是不是在持续维护，只要有维护，有迭代，项目就不会死，就会变得更好。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-07 10:49:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/7a/4d/b0228a1a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>平风造雨</span>
  </div>
  <div class="_2_QraFYR_0">总结起来，对于使用方要主动甄别三方包的良莠，甄别的过程有一些潜在的规则。<br>比如，lua-resty-*作为前缀的和高性能相关，要重点关注。<br>配合OPM&#47;LuaRocks以及awesome-resty对三方库做进一步的背景调查，最后还是需要关注代码本身，关注测试用例，进行三方库选择的把控。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-26 14:21:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/8a/97/7ab3a22c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>蜥蜴1214</span>
  </div>
  <div class="_2_QraFYR_0">老师，求助啊。我mac电脑，安装完openresty后，通过opm安装了lua-resty-requests后，按照官网demo，运行resty -e &#39;print(require &quot;resty.requests&quot;.get(&quot;https:&#47;&#47;github.com&quot;, { stream = false}).content)&#39;。但是直接报错。...nresty&#47;1.19.3.1_1&#47;site&#47;lualib&#47;resty&#47;requests&#47;adapter.lua:4: module &#39;resty.socket&#39; not found。网上也没相关的资料，这个是少安装了东西吧</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-26 15:33:56</div>
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
  <div class="_2_QraFYR_0">一直以为 Kong 读 空</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-22 20:48:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/eb/19/0d990b03.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ZeroIce</span>
  </div>
  <div class="_2_QraFYR_0">老师，有什么书籍推荐的？英文书籍也行<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: OpenResty 现在并没有比较合适的书推荐，有的也是中文的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-03 13:15:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/20/d0/ad12060d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>蓝色海洋</span>
  </div>
  <div class="_2_QraFYR_0">请问老师，openresty支持grpc通信吗？有没有相关的组件可以将grpc转换为普通的http请求</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 现在 OpenResty 和 Nginx 一样，只支持 grpc 的转发，并没有实现协议的转换，也不支持 grpc 的客户端。这算是 OpenResty 的一个软肋。我们团队有计划对这方面做加强。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-15 07:13:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/4d/42/8fd7c2e2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>朋朋</span>
  </div>
  <div class="_2_QraFYR_0">我的&#47;usr&#47;local&#47;openresty&#47;bin&#47; 下只有这俩 我是centos7 的环境 看来我得重新安装一下啦<br>openresty  resty</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-25 10:35:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/d0/46/1a9229b3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>NEVER SETTLE</span>
  </div>
  <div class="_2_QraFYR_0">yum install openresty 安装好openresty之后，为什么找不到opm</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: sudo yum install openresty-opm 需要单独安装一下</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-17 00:32:24</div>
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
  <div class="_2_QraFYR_0">为什么 我这里会提示使用luarocks 命令说不存在？ 需要配置什么才可以？<br>localhost:geektime yuesf$ luarocks search http<br>-bash: luarocks: command not found<br>localhost:geektime yuesf$ </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: luarocks 需要单独安装，你可以用系统的包管理工具安装，比如 brew install luarocks</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-14 23:03:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/99/74/0203bf17.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>英雄</span>
  </div>
  <div class="_2_QraFYR_0">菜鸟推荐用哪个管理工具呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: luarocks</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-10 11:49:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/b6/46/b17cbaff.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>王斌</span>
  </div>
  <div class="_2_QraFYR_0">的确需要温老师这样的课程，lua虽然入门简单，但是openresty的正确打开方式不是很好找😢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，安装包就不容易</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-03 01:10:40</div>
  </div>
</div>
</div>
</li>
</ul>