<audio title="24 _ 实战：处理四层流量，实现Memcached Server" src="https://static001.geekbang.org/resource/audio/3f/f5/3f538d45054560e043ee575907fb5df5.mp3" controls="controls"></audio> 
<p>你好，我是温铭。</p><p>在前面几节课中，我们介绍了不少处理请求的 Lua API ，不过它们都是和七层相关的。除此之外，OpenResty 其实还提供了 <code>stream-lua-nginx-module</code> 模块来处理四层的流量。它提供的指令和 API ，与 <code>lua-nginx-module</code> 基本一致。</p><p>今天，我就带你一起用 OpenResty 来实现一个 memcached server，而且大概只需要 100 多行代码就可以完成。在这个小的实战中，我们会用到不少前面学过的内容，也会带入一些后面测试和性能优化章节的内容。</p><p>所以，我希望你能够明确一点，我们这节课的重点，不在于你必须读懂每一行代码的具体作用，而是你要从需求、测试、开发等角度，把 OpenResty 如何从零开发一个项目的全貌了然于心。</p><h2>原始需求和技术方案</h2><p>在开发之前，我们都需要明白需求是什么，到底是用来解决什么问题的，否则就会在迷失在技术选择中。比如看到我们今天的主题，你就应该先反问一下自己，为什么要实现一个 memcached server 呢？直接安装一个原版的 memcached 或者 redis 不就行了吗？</p><p>我们知道，HTTPS 流量逐渐成为主流，但一些比较老的浏览器并不支持 session ticket，那么我们就需要在服务端把 session ID 存下来。如果本地存储空间不够，就需要一个集群进行存放，而这个数据又是可以丢弃的，所以选用 memcached 就比较合适。</p><!-- [[[read_end]]] --><p>这时候，直接引入 memcached ，应该是最简单直接的方案。但出于以下几个方面的考虑，我还是选择使用 OpenResty 来造一个轮子：</p><ul>
<li>第一，直接引入会多引入一个进程，增加部署和维护成本；</li>
<li>第二，这个需求足够简单，只需要 get 和 set 操作，并且支持过期即可；</li>
<li>第三，OpenResty 有 stream 模块，可以很快地实现这个需求。</li>
</ul><p>既然要实现 memcached server，我们就需要先弄明白它的协议。memcached 的协议可以支持 TCP 和 UDP，这里我选择 TCP，下面是 get 和 set 命令的具体协议：</p><pre><code>Get
根据 key 获取 value
Telnet command: get &lt;key&gt;*\r\n

示例：
get key
VALUE key 0 4 data END

</code></pre><pre><code>Set
存储键值对到 memcached 中
Telnet command：set &lt;key&gt; &lt;flags&gt; &lt;exptime&gt; &lt;bytes&gt; [noreply]\r\n&lt;value&gt;\r\n

示例：
set key 0 900 4 data
STORED
</code></pre><p>除了 get 和 set 外，我们还需要知道 memcached 的协议的“错误处理”是怎么样做的。“错误处理”对于服务端的程序是非常重要的，我们在编写程序时，除了要处理正常的请求，也要考虑到各种异常。比如下面这样的场景：</p><ul>
<li>memcached 发送了一个get、set 之外的请求，我要怎么处理呢？</li>
<li>服务端出错，我要给 memcached 的客户端一个什么样的反馈呢？</li>
</ul><p>同时，我们希望写出能够兼容 memcached 的客户端程序。这样，使用者就不用区分这是 memcached 官方的版本，还是 OpenResty 实现的版本了。</p><p>下面这张图出自memcached 的文档，描述了出错的时候，应该返回什么内容和具体的格式，你可以用做参考：</p><p><img src="https://static001.geekbang.org/resource/image/37/b0/3767ed0047e34aabaa7bf7d568438ab0.png?wh=1974*1446" alt=""></p><p>现在，再来确定下技术方案。我们知道，OpenResty 的 shared dict 可以跨各个 worker 来使用，把数据放在 shared dict 里面，和放在 memcached 里面非常类似——它们都支持 get 和 set 操作，并且在进程重启后数据就丢失了。所以，使用 shared dict 来模拟 memcached 是非常合适的，它们的原理和行为都是一致的。</p><h2>测试驱动开发</h2><p>接下来就要开始动工了。不过，基于测试驱动开发的思想，在写具体的代码之前，让我们先来构造一个最简单的测试案例。这里我们不用 <code>test::nginx</code> 框架，毕竟它的上手难度也不低，我们不妨先用熟悉的 <code>resty</code> 来手动测试下：</p><pre><code>$ resty -e 'local memcached = require &quot;resty.memcached&quot;
    local memc, err = memcached:new()

    memc:set_timeout(1000) -- 1 sec
    local ok, err = memc:connect(&quot;127.0.0.1&quot;, 11212)
    local ok, err = memc:set(&quot;dog&quot;, 32)
    if not ok then
        ngx.say(&quot;failed to set dog: &quot;, err)
        return
    end

    local res, flags, err = memc:get(&quot;dog&quot;)
    ngx.say(&quot;dog: &quot;, res)'
</code></pre><p>这段测试代码，使用 <code>lua-rety-memcached</code> 客户端库发起 connect 和 set 操作，并假设 memcached 的服务端监听本机的 11212 端口。</p><p>看起来应该没有问题了吧。你可以在自己的机器上执行一下这段代码，不出意外的话，会返回 <code>failed to set dog: closed</code> 这样的错误提示，因为此时服务并没有启动。</p><p>到现在为止，你的技术方案就已经明确了，那就是使用 stream 模块来接收和发送数据，同时使用 shared dict 来存储数据。</p><p>衡量需求是否完成的指标也很明确，那就是跑通上面这段代码，并把 dog 的实际值给打印出来。</p><h2>搭建框架</h2><p>那还等什么，开始动手写代码吧！</p><p>我个人的习惯，是先搭建一个最小的可以运行的代码框架，然后再逐步地去填充代码。这样的好处是，在编码过程中，你可以给自己设置很多小目标；而且在完成一个小目标后，测试案例也会给你正反馈。</p><p>让我们先来设置好 Nginx 的配置文件，因为stream 和 shared dict 要在其中预设。下面是我设置的配置文件：</p><pre><code>stream {
    lua_shared_dict memcached 100m;
    lua_package_path 'lib/?.lua;;';
    server {
        listen 11212;
        content_by_lua_block {
            local m = require(&quot;resty.memcached.server&quot;)
            m.run()
        }
    }
}
</code></pre><p>你可以看到，这段配置文件中有几个关键的信息：</p><ul>
<li>首先，代码运行在 Nginx 的 stream 上下文中，而非 HTTP 上下文中，并且监听了 11212 端口；</li>
<li>其次，shared dict 的名字为 memcached，大小是 100M，这些在运行期是不可以修改的；</li>
<li>另外，代码所在目录为 <code>lib/resty/memcached</code>, 文件名为 <code>server.lua</code>, 入口函数为 <code>run()</code>，这些信息你都可以从<code>lua_package_path</code> 和 <code>content_by_lua_block</code> 中找到。</li>
</ul><p>接着，就该搭建代码框架了。你可以自己先动手试试，然后我们一起来看下我的框架代码：</p><pre><code>local new_tab = require &quot;table.new&quot;
local str_sub = string.sub
local re_find = ngx.re.find
local mc_shdict = ngx.shared.memcached

local _M = { _VERSION = '0.01' }

local function parse_args(s, start)
end

function _M.get(tcpsock, keys)
end

function _M.set(tcpsock, res)
end

function _M.run()
    local tcpsock = assert(ngx.req.socket(true))

    while true do
        tcpsock:settimeout(60000) -- 60 seconds
        local data, err = tcpsock:receive(&quot;*l&quot;)

        local command, args
        if data then
            local from, to, err = re_find(data, [[(\S+)]], &quot;jo&quot;)
            if from then
                command = str_sub(data, from, to)
                args = parse_args(data, to + 1)
            end
        end

        if args then
            local args_len = #args
            if command == 'get' and args_len &gt; 0 then
                _M.get(tcpsock, args)
            elseif command == &quot;set&quot; and args_len == 4 then
                _M.set(tcpsock, args)
            end
        end
    end
end

return _M
</code></pre><p>这段代码，便实现了入口函数 <code>run()</code> 的主要逻辑。虽然我还没有做异常处理，依赖的 <code>parse_args</code>、<code>get</code> 和 <code>set</code> 也都是空函数，但这个框架已经完整表达了memcached server 的逻辑。</p><h2>填充代码</h2><p>接下来，让我们按照代码的执行顺序，逐个实现这几个空函数。</p><p>首先，我们可以根据 memcached <a href="https://github.com/memcached/memcached/blob/master/doc/protocol.txt">的协议</a><a href="https://github.com/memcached/memcached/blob/master/doc/protocol.txt">文档</a>，解析 memcached 命令的参数：</p><pre><code>local function parse_args(s, start)
    local arr = {}

    while true do
        local from, to = re_find(s, [[\S+]], &quot;jo&quot;, {pos = start})
        if not from then
            break
        end

        table.insert(arr, str_sub(s, from, to))

        start = to + 1
    end

    return arr
end
</code></pre><p>这里，我的建议是，先用最直观的方式来实现一个版本，不用考虑任何性能的优化。毕竟，完成总是比完美更重要，而且，基于完成的逐步优化才可以趋近完美。</p><p>接下来，我们就来实现下 <code>get</code> 函数。它可以一次查询多个键，所以下面代码中我用了一个 for 循环：</p><pre><code>function _M.get(tcpsock, keys)
    local reply = &quot;&quot;

    for i = 1, #keys do
        local key = keys[i]
        local value, flags = mc_shdict:get(key)
        if value then
            local flags  = flags or 0
            reply = reply .. &quot;VALUE&quot; .. key .. &quot; &quot; .. flags .. &quot; &quot; .. #value .. &quot;\r\n&quot; .. value .. &quot;\r\n&quot;
        end
    end
    reply = reply ..  &quot;END\r\n&quot;

    tcpsock:settimeout(1000)  -- one second timeout
    local bytes, err = tcpsock:send(reply)
end
</code></pre><p>其实，这里最核心的代码只有一行：<code>local value, flags = mc_shdict:get(key)</code>，也就是从 shared dict 中查询到数据；至于其余的代码，都在按照 memcached 的协议拼接字符串，并最终 send 到客户端。</p><p>最后，我们再来看下 <code>set</code> 函数。它将接收到的参数转换为 shared dict API 的格式，把数据储存了起来；并在出错的时候，按照 memcached 的协议做出处理：</p><pre><code>function _M.set(tcpsock, res)
    local reply =  &quot;&quot;

    local key = res[1]
    local flags = res[2]
    local exptime = res[3]
    local bytes = res[4]

    local value, err = tcpsock:receive(tonumber(bytes) + 2)

    if str_sub(value, -2, -1) == &quot;\r\n&quot; then
        local succ, err, forcible = mc_shdict:set(key, str_sub(value, 1, bytes), exptime, flags)
        if succ then
            reply = reply .. “STORED\r\n&quot;
        else
            reply = reply .. &quot;SERVER_ERROR &quot; .. err .. “\r\n”
        end
    else
        reply = reply .. &quot;ERROR\r\n&quot;
    end

    tcpsock:settimeout(1000)  -- one second timeout
    local bytes, err = tcpsock:send(reply)
end
</code></pre><p>另外，在填充上面这几个函数的过程中，你可以用测试案例来做检验，并用 <code>ngx.log</code> 来做 debug。比较遗憾的是，OpenResty 中并没有断点调试的工具，所以我们都是使用 <code>ngx.say</code> 和 <code>ngx.log</code> 来调试的，在这方面可以说是还处于刀耕火种的时代。</p><h2>写在最后</h2><p>这个实战项目到现在就接近尾声了，最后，我想留一个动手作业。你可以把上面 memcached server 的实现代码，完整地运行起来，并通过测试案例吗？</p><p>今天的作业题估计要花费你不少的精力了，不过，这还是一个原始的版本，还没有错误处理、性能优化和自动化测试，这些就要放在后面继续完善了。我也希望通过后面内容的学习，你最终能够完成一个完善的版本。</p><p>如果对于今天的讲解或者自己的实践有什么疑惑，欢迎你留言和我讨论。也欢迎你把这篇文章转发给你的同事朋友，我们一起实战，一起进步。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKsz8j0bAayjSne9iakvjzUmvUdxWEbsM9iasQ74spGFayIgbSE232sH2LOWmaKtx1WqAFDiaYgVPwIQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>2xshu</span>
  </div>
  <div class="_2_QraFYR_0">老师，这篇文章我反复看了几遍。其实我还是没有领会做这个项目的真正目的和使用场景呢。因为我理解的是即是做了这样封装，和直接使用ngx.shared.DICT感觉没什么区别呢？ngx.shared.DICT操作也是原子操作的，并且ngx.shared.DICT也是一个全局共享的变量。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 最大的不同之处在于可以支持集群，而共享字典只能是单机使用。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-22 10:24:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/97/69/80945634.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>罐头瓶子</span>
  </div>
  <div class="_2_QraFYR_0">openresty lua的断点调试现在可以使用vscode插件的方式来实现。插件的名字叫luaide，我现在正在使用，挺好用的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-19 21:36:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/bc/57/e66d8571.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>katichar</span>
  </div>
  <div class="_2_QraFYR_0">在http和stream之前实现数据共享，如果对性能有要求，是不是resty.memcached是一个比较好的方案？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 走了一层外部的网络通信，性能会有一些影响，但貌似也是当前的一个方案。最好是内部打通，期望 OpenResty 实现这个功能。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-19 10:20:23</div>
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
  <div class="_2_QraFYR_0">我在执行这里的例子的时候，解决了一些语法错误之后，已经能够正常的在resty -e执行时设置和获得key value了，但是在服务器侧循环报一个错误，2019&#47;07&#47;27 13:35:43 [error] 25017#25017: *1 attempt to receive data on a closed socket: u:00007FDA656A5530, c:00007FDA655FE3F0, ft:0 eof:1, client: 127.0.0.1, server: 0.0.0.0:11212。这个错误，通过设置指令lua_socket_log_errors off;的确关闭了错误的循环打印，这里的根本原因是，因为server.lua里是一个循环，然后客户端断开socket了，服务侧还在循环读取数据，导致一直提示这个错误吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，这种错误是可以忽略的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-27 14:03:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_Wx.71</span>
  </div>
  <div class="_2_QraFYR_0">- -终于调通了...标点符号&quot;&quot; &#39;&#39; “” ， tonumber()<br>最要命的是少个空格</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-25 14:31:27</div>
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
  <div class="_2_QraFYR_0">有个挺严重的问题，get函数，在一次请求成功之后，客户端已经关闭的情况下，服务器还在不停的给客户端发送数据，不停的对一个对端已经关闭的文件描述符进行写动作，这个地方应该做个判断，如果send错误，就结束循环，不知道nginx内部处理sigpipe信号没有，我觉得应该是处理了，这是最基本的常识，写一个对端关闭的文件描述符，默认动作是结束进程的，会导致程序coredump，忽略掉这个信号，通过如果写错误，则关闭文件描述符释放资源和进行判断。<br>导致只要发送一次请求，导致服务器不停的死循环，向一个对端关了的描述符发送数据，不停的打日志，好端端的，把我的电脑弄的很卡，不过这都是测试代码，如果大家有一定常识或者钻研精神，都能解决</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-10 10:17:19</div>
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
  <div class="_2_QraFYR_0">搭是搭上了，跑也出点效果了，可惜没存进去数据，不太熟悉函数，一会nginx的函数，一会lua的函数，一会又是openresty的函数，我了个去了，人的大脑只能跟踪三件事，要不是对这三部分都差不多熟悉的花，搞这个还真的会费挺大的劲，这很多初学者找不到哪是哪，真的在所难免。lua和nginx加一起，再封装出来openresty函数，行吧，我搭成了，至于函数接口不对，程序跑飞了的问题，算是从1到优的过程了，不纠结了，我已经是合格的初级架构师了，搭完了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-09 17:30:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/24/d7/146f484b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小宇子2B</span>
  </div>
  <div class="_2_QraFYR_0">\r\n是两个字节吗？感觉应该是四个字节<br>自己调试的时候使用telnet发送\r\n，在str_sub(value, -2, -1)只能获取到\n而获取不到\r</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-19 22:05:34</div>
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
  <div class="_2_QraFYR_0">老师，我在执行 resty 脚本的时候，运行到 mc_shdict:set() 方法导致了一个报错 &#47;openresty&#47;1.15.8.2&#47;lualib&#47;resty&#47;core&#47;shdict.lua:186: attempt to compare string with number，这是什么原因，我看文中用的参数也是字符串类型的</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-15 21:10:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTL4jnwQhhibicA13DTmyW6vPHaP9FnhYVAEMUiaBRjTy7Gzx9qeuUmCSia06ibC7gMr0RiblXUQZZfBEjpQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>fangminyu</span>
  </div>
  <div class="_2_QraFYR_0">温大师，你的教程非常好，我还在入门中，基本功能已经调通了，但是发现有个严重问题，每次测试指套test一断开链接就有这样的问题：<br>2020&#47;02&#47;19 12:21:03 [error] 10681#123902: *2 attempt to receive data on a closed socket: u:00000000031192E0, c:00007FD6670081F0, ft:0 eof:1, client: 127.0.0.1, server: 0.0.0.0:11212<br><br>我网上搜了一圈也没找到办法，这个打印到底是从哪里来的？怎么解决？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-19 12:24:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/67/e4/42ea7a9a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>姚坤</span>
  </div>
  <div class="_2_QraFYR_0">我测试运行过程中需要将 function _M.get(tcpsock, keys) 中的这一行：reply = reply .. &quot;VALUE &quot; .. key .. &quot; &quot; .. flags .. &quot; &quot; .. #value .. &quot;\r\n&quot; .. value .. &quot;\r\n&quot; 的 “VALUE&quot;后加一个空格。<br>否则返回的是nil。<br>跟踪到memcached.lua中的get 函数有如下判断<br>local flags, len = match(line, &#39;^VALUE %S+ (%d+) (%d+)$&#39;)<br>    if not flags then<br>        return nil, nil, line<br>    end<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-20 22:53:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/41/38/4f89095b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>写点啥呢</span>
  </div>
  <div class="_2_QraFYR_0">请问老师，你在@2xshu同学的留言里提到了这个实现支持集群，而共享字典只能单机使用，我没有理解，请问支持集群是指可以跨物理机共享存储么？那具体是如何做到的呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你使用 memcached 的客户端，都是支持一致性 hash 这样的算法，这样就等于是实现了集群储存了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-06 08:02:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/3c/84/608f679b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>连边</span>
  </div>
  <div class="_2_QraFYR_0">我写了一个空的代码，直接写一个死循环，会直接导致cpu 100%。这是正常情况？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 要看下循环里面具体做了什么</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-20 19:59:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/3c/84/608f679b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>连边</span>
  </div>
  <div class="_2_QraFYR_0">老师见字好。我来实现文中逻辑，但是最后总说我连接close，详情如下：https:&#47;&#47;www.iffor.cn&#47;job&#47;openresty-memcached-server.html，有时间的时候帮我看看。我这边准备按照memcache去找一找原因。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你给的链接中没有具体的报错信息。另外，&#47;usr&#47;local&#47;openresty3&#47;lib&#47;resty&#47;memached 这个目录是拼写错了吗？最后的目录名应该 memcached 而不是 memached，少了一个字母 c</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-31 00:03:50</div>
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
  <div class="_2_QraFYR_0">老师，这次示例memcached server还是没有调通，您在后面答疑的时候能大约讲一下吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是哪一部分的问题呢？有具体的报错吗？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-30 16:49:43</div>
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
  <div class="_2_QraFYR_0">如果是为了实现memcached集群，那服务端口就不能监听127.0.0.1了，对吧</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，这个需要修改下</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-24 21:31:30</div>
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
  <div class="_2_QraFYR_0">和2xshu同学的感受一样，使用共享字典封装模拟成memcached服务，和直接使用共享字典，对于项目需求到低有啥本质区别，对于这个项目需求直接使用共享字典不就行了嘛，为啥还要费事的去封装一遍呢，求解惑。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 直接使用共享字典就没有办法做到集群了，只能用本机的内存资源</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-24 18:58:47</div>
  </div>
</div>
</div>
</li>
</ul>