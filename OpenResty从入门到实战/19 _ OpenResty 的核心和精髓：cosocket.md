<audio title="19 _ OpenResty 的核心和精髓：cosocket" src="https://static001.geekbang.org/resource/audio/b3/39/b33be3f616df6012d81f7605d30a5e39.mp3" controls="controls"></audio> 
<p>你好，我是温铭，今天我们来学习下 OpenResty 中的核心技术：cosocket。</p><p>其实在前面的课程中，我们就已经多次提到过它了，cosocket 是各种 <code>lua-resty-*</code> 非阻塞库的基础，没有 cosocket，开发者就无法用 Lua 来快速连接各种外部的网络服务。</p><p>在早期的 OpenResty 版本中，如果你想要去与 Redis、memcached 这些服务交互的话，需要使用 <code>redis2-nginx-module</code>、<code>redis-nginx-module</code> 和 <code>memc-nginx-module</code>这些 C 模块.这些模块至今仍然在 OpenResty 的发行包中。</p><p>不过，cosocket 功能加入以后，它们都已经被 <code>lua-resty-redis</code> 和 <code>lua-resty-memcached</code> 替代，基本上没人再去使用 C 模块连接外部服务了。</p><h2>什么是 cosocket？</h2><p>那究竟什么是cosocket 呢？事实上，cosocket是 OpenResty 中的专有名词，是把协程和网络套接字的英文拼在一起形成的，即 cosocket = coroutine + socket。所以，你可以把 cosocket 翻译为“协程套接字”。</p><p>cosocket 不仅需要 Lua 协程特性的支持，也需要 Nginx 中非常重要的事件机制的支持，这两者结合在一起，最终实现了非阻塞网络 I/O。另外，cosocket 支持 TCP、UDP 和 Unix Domain Socket。</p><!-- [[[read_end]]] --><p>如果我们在 OpenResty 中调用一个 cosocket 相关函数，内部实现便是下面这张图的样子：</p><p><img src="https://static001.geekbang.org/resource/image/80/06/80d16e11d2750d6e4127445c126c9f06.png?wh=1692*756" alt=""></p><p>记性比较好的同学应该发现了，在前面 OpenResty 原理和基本概念的那节课里，我也用过这张图。从图中你可以看到，用户的 Lua 脚本每触发一个网络操作，都会有协程的 yield 以及 resume。</p><p>遇到网络 I/O 时，它会交出控制权（yield），把网络事件注册到 Nginx 监听列表中，并把权限交给 Nginx；当有 Nginx 事件达到触发条件时，便唤醒对应的协程继续处理（resume）。</p><p>OpenResty 正是以此为蓝图，封装实现 connect、send、receive 等操作，形成了我们如今见到的 cosocket API。下面，我就以处理 TCP 的 API 为例来介绍一下。处理 UDP 和 Unix Domain Socket ，与TCP 的接口基本是一样的。</p><h2>cosocket API 和指令简介</h2><p>TCP 相关的 cosocket API 可以分为下面这几类。</p><ul>
<li>创建对象：ngx.socket.tcp。</li>
<li>设置超时：tcpsock:settimeout 和 tcpsock:settimeouts。</li>
<li>建立连接：tcpsock:connect。</li>
<li>发送数据：tcpsock:send。</li>
<li>接受数据：tcpsock:receive、tcpsock:receiveany 和 tcpsock:receiveuntil。</li>
<li>连接池：tcpsock:setkeepalive。</li>
<li>关闭连接：tcpsock:close。</li>
</ul><p>我们还要特别注意下，这些 API 可以使用的上下文：</p><pre><code>rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.*, ssl_certificate_by_lua*, ssl_session_fetch_by_lua*_
</code></pre><p>这里我还要强调一点，归咎于 Nginx 内核的各种限制，cosocket API 在 <code>set_by_lua*</code>， <code>log_by_lua*</code>， <code>header_filter_by_lua*</code> 和 <code>body_filter_by_lua*</code> 中是无法使用的。而在 <code>init_by_lua*</code> 和 <code>init_worker_by_lua*</code> 中暂时也不能用，不过 Nginx 内核对这两个阶段并没有限制，后面可以增加对这它们的支持。</p><p>此外，与这些 API 相关的，还有 8 个 <code>lua_socket_</code> 开头的 Nginx 指令，我们简单来看一下。</p><ul>
<li><code>lua_socket_connect_timeout</code>：连接超时，默认 60 秒。</li>
<li><code>lua_socket_send_timeout</code>：发送超时，默认 60 秒。</li>
<li><code>lua_socket_send_lowat</code>：发送阈值（low water），默认为 0。</li>
<li><code>lua_socket_read_timeout</code>： 读取超时，默认 60 秒。</li>
<li><code>lua_socket_buffer_size</code>：读取数据的缓存区大小，默认 4k/8k。</li>
<li><code>lua_socket_pool_size</code>：连接池大小，默认 30。</li>
<li><code>lua_socket_keepalive_timeout</code>：连接池 cosocket 对象的空闲时间，默认 60 秒。</li>
<li><code>lua_socket_log_errors</code>：cosocket 发生错误时，是否记录日志，默认为 on。</li>
</ul><p>这里你也可以看到，有些指令和 API 的功能一样的，比如设置超时时间和连接池大小等。不过，如果两者有冲突的话，API 的优先级高于指令，会覆盖指令设置的值。所以，一般来说，我们都推荐使用 API 来做设置，这样也会更加灵活。</p><p>接下来，我们一起来看一个具体的例子，弄明白到底如何使用这些 cosocket API。下面这段代码的功能很简单，是发送 TCP 请求到一个网站，并把返回的内容打印出来：</p><pre><code>$ resty -e 'local sock = ngx.socket.tcp()
        sock:settimeout(1000)  -- one second timeout
        local ok, err = sock:connect(&quot;www.baidu.com&quot;, 80)
        if not ok then
            ngx.say(&quot;failed to connect: &quot;, err)
            return
        end

        local req_data = &quot;GET / HTTP/1.1\r\nHost: www.baidu.com\r\n\r\n&quot;
        local bytes, err = sock:send(req_data)
        if err then
            ngx.say(&quot;failed to send: &quot;, err)
            return
        end

        local data, err, partial = sock:receive()
        if err then
            ngx.say(&quot;failed to receive: &quot;, err)
            return
        end

        sock:close()
        ngx.say(&quot;response is: &quot;, data)'
</code></pre><p>我们来具体分析下这段代码。</p><ul>
<li>首先，通过 <code>ngx.socket.tcp()</code> ，创建  TCP 的 cosocket 对象，名字是 sock。</li>
<li>然后，使用 <code>settimeout()</code> ，把超时时间设置为 1 秒。注意这里的超时没有区分 connect、receive，是统一的设置。</li>
<li>接着，使用 <code>connect()</code> 去连接指定网站的 80 端口，如果失败就直接退出。</li>
<li>连接成功的话，就使用 <code>send()</code> 来发送构造好的数据，如果发送失败就退出。</li>
<li>发送数据成功的话，就使用 <code>receive()</code> 来接收网站返回的数据。这里 <code>receive()</code> 的默认参数值是 <code>*l</code>，也就是只返回第一行的数据；如果参数设置为了<code>*a</code>，就是持续接收数据，直到连接关闭；</li>
<li>最后，调用 <code>close()</code> ，主动关闭 socket 连接。</li>
</ul><p>你看，短短几步就可以完成，使用 cosocket API 来做网络通信，就是这么简单。不过，不能满足于此，接下来，我们对这个示例再做一些调整。</p><p><strong>第一个动作，对 socket 连接、发送和读取这三个动作，分别设置超时时间。</strong></p><p>我们刚刚用的<code>settimeout()</code> ，作用是把超时时间统一设置为一个值。如果要想分开设置，就需要使用 <code>settimeouts()</code> 函数，比如下面这样的写法：</p><pre><code>sock:settimeouts(1000, 2000, 3000) 
</code></pre><p>这行代码表示连接超时为 1 秒，发送超时为 2 秒，读取超时为 3 秒。</p><p>在OpenResty 和 lua-resty 库中，大部分和时间相关的 API 的参数，都以毫秒为单位，但也有例外，需要你在调用的时候特别注意下。</p><p><strong>第二个动作，receive接收指定大小的内容。</strong></p><p>刚刚说了，<code>receive()</code> 接口可以接收一行数据，也可以持续接收数据。不过，如果你只想接收 10K 大小的数据，应该怎么设置呢？</p><p>这时，<code>receiveany()</code> 闪亮登场。它就是专为满足这种需求而设计的，一起来看下面这行代码：</p><pre><code>local data, err, partial = sock:receiveany(10240)
</code></pre><p>这段代码就表示，最多只接收 10K 的数据。</p><p>当然，关于receive，还有另一个很常见的用户需求，那就是一直获取数据，直到遇到指定字符串才停止。</p><p><code>receiveuntil()</code> 专门用来解决这类问题，它不会像 <code>receive()</code> 和 <code>receiveany()</code> 一样返回字符串，而会返回一个迭代器。这样，你就可以在循环中调用它来分段读取匹配到的数据，当读取完毕时，就会返回 nil。下面就是一个例子：</p><pre><code> local reader = sock:receiveuntil(&quot;\r\n&quot;)

 while true do
     local data, err, partial = reader(4)
     if not data then
         if err then
             ngx.say(&quot;failed to read the data stream: &quot;, err)
             break
         end

         ngx.say(&quot;read done&quot;)
         break
     end
     ngx.say(&quot;read chunk: [&quot;, data, &quot;]&quot;)
 end
</code></pre><p>这段代码中的 <code>receiveuntil</code> 会返回 <code>\r\n</code> 之前的数据，并通过迭代器每次读取其中的 4 个字节，也就实现了我们想要的功能。</p><p><strong>第三个动作，不直接关闭 socket，而是放入连接池中。</strong></p><p>我们知道，没有连接池的话，每次请求进来都要新建一个连接，就会导致 cosocket 对象被频繁地创建和销毁，造成不必要的性能损耗。</p><p>为了避免这个问题，在你使用完一个 cosocket 后，可以调用 <code>setkeepalive()</code> 放到连接池中，比如下面这样的写法：</p><pre><code>local ok, err = sock:setkeepalive(2 * 1000, 100)
if not ok then
    ngx.say(&quot;failed to set reusable: &quot;, err)
end
</code></pre><p>这段代码设置了连接的空闲时间为 2 秒，连接池的大小为 100。这样，在调用 <code>connect()</code> 函数时，就会优先从连接池中获取 cosocket 对象。</p><p>不过，关于连接池的使用，有两点需要我们注意一下。</p><ul>
<li>第一，不能把发生错误的连接放入连接池，否则下次使用时，就会导致收发数据失败。这也是为什么我们需要判断每一个 API 调用是否成功的一个原因。</li>
<li>第二，要搞清楚连接的数量。连接池是 worker 级别的，每个 worker 都有自己的连接池。所以，如果你有 10 个 worker，连接池大小设置为 30，那么对于后端的服务来讲，就等于有 300 个连接。</li>
</ul><h2>写在最后</h2><p>总结一下，今天我们学习了cosocket 的基本概念，以及相关的指令和 API，并通过一个实际的例子，熟悉了TCP 相关的 API 应该如何使用。而UDP 和 Unix Domain Socket的使用类似于TCP，弄明白今天所学，你基本上都能迎刃而解了。</p><p>从中你应该也能感受到，cosocket 用起来还是比较容易上手的，而且用好它，你就可以去连接各种外部的服务了，可以说是给 OpenResty 插上了想象的翅膀。</p><p>最后，给你留两个作业题。</p><p>第一问，在今天的例子中，<code>tcpsock:send</code> 发送的是字符串，如果我们需要发送一个由字符串构成的 table，又该怎么处理呢？</p><p>第二问，你也看到了，cosocket 在很多阶段中不能使用，那么，你能否想到一些绕过的方式呢？</p><p>欢迎留言和我分享，也欢迎你把这篇文章分享给你的同事、朋友，我们一起交流，一起进步。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/11/3e/925aa996.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>HelloBug</span>
  </div>
  <div class="_2_QraFYR_0">问题一：将表格利用cjson进行序列化之后发送<br>问题二：通过设置定时器，在不能使用cosocket的阶段使用</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 第二个答案是对的。第一个确实是可以的，但并非最好的方案，send 函数不仅支持字符串，还支持 table，这也是隐藏的优化点。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-08 09:09:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/fe/1f/6a3f2abb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小超人</span>
  </div>
  <div class="_2_QraFYR_0">有个真实需求：<br>client   --密文--&gt; Openresty解密   --明文--&gt; server<br>client &lt;--密文--   Openresty加密 &lt;--明文--   server<br>1. 这是TCP应用，send&#47;recv不一定是一对儿，比如有个大请求，client需要send 10次以后再收；<br>2. Openresty知道client发过来的密文格式，带total_len，可以正常receive(total_len)就可以了；<br>3. 【问题】Openresty不知道业务数据格式，在receive上游服务器时，遇到两个问题：<br>*3.1* 不知道什么时候receive，如1.描述，Openresty将明文转发至server时，server不一定有返回数据；<br>*3.2*如果来了数据，不知道该收多少，因为不知道业务数据的格式。<br>【我现在的做法是】<br>1. Openresty receive server那一侧，设置了一个buf_len,和一个较小的超时，比如200ms;<br>2. Openresty每转发一次数据到server，都尝试receive一下server，如果超时，不报错不返回；如果receive到数据了，就加密返回后再循环receive，直到无数据（超时）；<br>这样虽然解决了问题，但是性能太差了，在openresty---server那一侧，至少要receive超时一次，才能知道收完数据了。<br>【请问】有更好的解决办法吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我理解了你的问题，尝试回答下，不一定准确：<br>1. client 会传上来 total_len，那么 OpenResty 就能计算出来第几次 send 是最后一次。在最后一次 client send 的数据解密并转发给上游 server 后，开始 receive server。是否可以解决呢？<br>2. 这个有有些尴尬了，即使OpenResty 不关心数据格式，至少也要知道数据长度或者结束符，这两个的其中一个。不然无法知道何时结束。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-13 10:03:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Leo</span>
  </div>
  <div class="_2_QraFYR_0">真是用心良苦，我在openresty的github介绍页上，看见了英文版的介绍，真挺不容易的，为了推广，等于把英文版的介绍，做了拣选之后做了整理翻译。谢谢了，我也在努力补英语，中国体制教育的受害者，我是学俄语的，初中被分到了俄语班，没机会选择，并且家里也没去争取过。没办法靠自己吧，不过真的挺无语的，我培训的机构受双减政策倒闭了，你想往出走，就要打断你的腿，我们的民族进步还有希望么</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-08 18:17:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/24/6f/dc/dba82cbe.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>金黄十月</span>
  </div>
  <div class="_2_QraFYR_0">温老师：问一下，为什么使用ngx.socket.tcp连接本机IP (192.168.0.2)的服务，总是返回”Connection refused“错误，使用python去连接本机的服务，或者使用telnet 192.168.0.2却可以成功；使用ngx.socket.tcp连接本机IP (192.168.0.6)是成功的。感觉是openresty的问题。比如连接池返回的socket有问题还是其他原因？有没有什么好建议？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-16 19:55:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/a6/5e/95b85420.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>swimming</span>
  </div>
  <div class="_2_QraFYR_0">老师，有个实际需求，需要在header filter阶段调用http接口，后续的处理依赖这次调用的返回结果，这种情况使用timer跳过无法满足需求，因为不能保证后续处理时timer已经调用完毕返回结果了。请问这种情况有处理方案吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-14 16:49:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/a4/ab/b2177dcf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>学会传球</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，请问下客户端怎么bind()本地网卡ip，我的机器有多个网卡，我想指定一个网卡出去，如果ngx.socket.tcp()没有实现bind()这个函数，我是在ngx_http_lua_socket_tcp.c这个文件里new对象的时候加上去，还是在哪个地方加？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-09 17:08:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/7a/ec/19104a37.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>李强</span>
  </div>
  <div class="_2_QraFYR_0">老师， 我在使用时遇到一个问题：<br>openresty作为客户端与上游服务建立Tcp连接时， 不能使用bind方法指定端口， 也无法获取随机分配的端口， 是我使用的姿势不对么？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-23 19:56:01</div>
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
  <div class="_2_QraFYR_0">问题一：将table通过cjson序列化，但是看到老师有说send函数支持table，查阅api文档有说The input argument data can either be a Lua string or a (nested) Lua table holding string fragments.In case of table arguments, this method will copy all the string elements piece by piece to the underlying Nginx socket send buffers, which is usually optimal than doing string concatenation operations on the Lua land.<br>问题二：通过shared dict进行数据传递</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 接收 table 而非字符串，是 OpenResty Lua API 的一个重要的优化点。<br>通过shared dict进行数据传递，这个我不是很明白，这个是如何绕过限制的呢？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-13 22:14:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/dd/09/feca820a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>helloworld</span>
  </div>
  <div class="_2_QraFYR_0">老师：<br>[error] 16239#16239: *5 attempt to send data on a closed socket: u:0000000000000000, c:0000000000000000<br>客户端每次post数据过来，lua都可以正确的将数据写入redis，但是error.log日志中总是报这个错误。<br>老师，我没用到redis连接池，我这个应用场景是每隔一段时间post请求往redis写数据，所以没必要使用连接池。关于redis的语句只有下面这些：<br>local redis = require &quot;resty.redis&quot;<br>local red = redis:new()<br>red:set_timemout(2000) -- 2 sec<br>local ip = &quot;127.0.0.1&quot;<br>local port = 6379<br>local ok, err = red:connect(ip, port)<br>if not ok then<br>    ...<br>end<br>ok, err = red:hset(data_time, client, json_encode(data))<br>if not ok then<br>    ...<br>end</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 建议在 redis:new() 的时候判断下是否成功。<br>另外附上 OpenResty 作者给这个错误的排查意见：https:&#47;&#47;github.com&#47;openresty&#47;lua-resty-redis&#47;issues&#47;27#issuecomment-28166754，自然是更加权威的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-12 15:42:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/dd/09/feca820a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>helloworld</span>
  </div>
  <div class="_2_QraFYR_0">[error] 16239#16239: *5 attempt to send data on a closed socket: u:0000000000000000, c:0000000000000000<br>客户端每次post数据过来，lua都可以正确的将数据写入redis，但是error.log日志中总是报这个错误，求解。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是否使用了redis 的连接池？猜测连接池里面有不正常的连接，但是被复用了。可以确认下放进连接池之前，代码中是否判断了连接是否正常？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-08 16:51:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/dd/09/feca820a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>helloworld</span>
  </div>
  <div class="_2_QraFYR_0">老师，是不是不论在哪个阶段运行ngx.say(&#39;hello&#39;),都会在执行完本阶段剩余代码后直接响应给客户端，不会继续执行其他阶段了。我测试是这样的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 并不是，你可以看下它的执行阶段的顺序图：https:&#47;&#47;github.com&#47;moonbingbing&#47;openresty-best-practices&#47;blob&#47;master&#47;images&#47;openresty_phases.png，你可以做这个测试，在 content 里面 ngx.say，然后在 log 或者 body filter 阶段打印下日志试试。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-08 11:13:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/11/3e/925aa996.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>HelloBug</span>
  </div>
  <div class="_2_QraFYR_0">文中利用cosocket将TCP请求发送到www.baidu.com的例子，我运行了一下包含头域Host：www.baidu.com和不包含该头域的请求，抓包来看，也确实是一个包含Host头域，一个不包含，两个请求都能得到百度的200应答。有一个问题是，我在本地使用proxy，然后转upstream时，如果不指定请求头域为请求到的域名的话，就不能得到正确的应答。这里是我写的关于这个问题的博客链接：https:&#47;&#47;blog.csdn.net&#47;u013139008&#47;article&#47;details&#47;94732537 介绍这个问题。老师，你知道这里必须要指定头域是www.baidu.com的原因吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: <br>一般来说，一个 ip 上可能会运行多个网站，所以会通过 server name 来做区分，比如 Nginx 中的 server 这个指令。如果没有设置<br> proxy_set_header Host &quot;www.baidu.com&quot;;<br>的话，在路由的时候就可能会找不到 server。<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-08 09:20:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/dd/09/feca820a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>helloworld</span>
  </div>
  <div class="_2_QraFYR_0">👍</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-08 05:26:51</div>
  </div>
</div>
</div>
</li>
</ul>