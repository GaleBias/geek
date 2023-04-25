<audio title="37 _ systemtap-toolkit和stapxx：如何用数据搞定“疑难杂症”？" src="https://static001.geekbang.org/resource/audio/5e/2e/5e990a9aaf32d686769080c79fa07a2e.mp3" controls="controls"></audio> 
<p>你好，我是温铭。</p><p>正如上节课介绍过的，作为服务端开发工程师，我们并不会对动态调试的工具集做深入的学习，大都是停留在使用的这个层面上，最多去编写一些简单的 stap 脚本。更底层的，比如 CPU 缓存、体系结构、编译器等，那就是性能工程师的领域了。</p><p>在 OpenResty 中有两个开源项目：<code>openresty-systemtap-toolkit</code> 和 <code>stapxx</code> 。它们是基于 Systemtap 封装好的工具集，用于 Nginx 和 OpenResty 的实时分析和诊断。它们可以覆盖 on CPU、off CPU、共享字典、垃圾回收、请求延迟、内存池、连接池、文件访问等常用的功能和调试场景。</p><p>在今天这节课中，我会带你浏览下这些工具和对应的使用方法，目的是帮你在遇到 Nginx 和 OpenResty 的疑难杂症时，可以快速找到定位问题的工具。在 OpenResty 的世界中，学会使用这些工具是你进阶的必经之路，也是和其他开发者沟通的非常有效的方式——毕竟，工具产生的数据，会比你用文字描述更加准确和详尽。</p><p>不过，需要特别注意的是，OpenResty 的最新版本 1.15.8 默认开启了 LuaJIT GC64 模式，但是 <code>openresty-systemtap-toolkit</code> 和 <code>stapxx</code> 并没有跟着做对应的修改，这就会导致里面的工具都无法正常使用。所以，你最好在 OpenResty 旧的 1.13 版本中来使用这些工具。</p><!-- [[[read_end]]] --><p>开源项目的贡献者大都是兼职身份，他们并没有义务来保证这些工具可以一直正常使用，这也是你在使用开源项目时候需要意识到的一点。</p><h2>以共享字典为例</h2><p>按照惯例，我先用一个你最熟悉的、也是上手最简单的工具 <code>ngx-lua-shdict</code>，来作为今天开篇的示例。</p><p><code>ngx-lua-shdict</code> 这个工具，可以分析 Nginx 的共享内存字典，并且追踪字典的操作。你可以用 <code>-f</code> 选项指定 dict 和 key，来获取共享内存字典里面的数据。 <code>--raw</code> 选项可以导出指定 key 的原始值。</p><p>下面是一个从共享内存字典中获取数据的命令行示例：</p><pre><code># 假设 nginx worker pid 是 5050
$ ./ngx-lua-shdict -p 5050 -f --dict dogs --key Jim --luajit20
Tracing 5050 (/opt/nginx/sbin/nginx)...

type: LUA_TBOOLEAN
value: true
expires: 1372719243270
flags: 0xa
</code></pre><p>类似的，你可以用 <code>-w</code>选项，来追踪指定 key 的字典写操作：</p><pre><code>$./ngx-lua-shdict -p 5050 -w --key Jim --luajit20
Tracing 5050 (/opt/nginx/sbin/nginx)...

Hit Ctrl-C to end

set Jim exptime=4626322717216342016
replace Jim exptime=4626322717216342016
^C
</code></pre><p>让我们看看这个工具是怎么实现的吧。<code>ngx-lua-shdict</code> 是一个 perl 的脚本，但具体的实现和 perl 并没有关系，perl 只是被用来生成了 stap 脚本并运行起来：</p><pre><code>open my $in, &quot;|stap $stap_args -x $pid -&quot; or die &quot;Cannot run stap: $!\n&quot;;
</code></pre><p>你完全可以用 Python、PHP、Go 或者你喜欢的任何语言来编写。stap 脚本中，比较关键的地方是下面这行代码：</p><pre><code>probe process(&quot;$nginx_path&quot;).function(&quot;ngx_http_lua_shdict_set_helper&quot;)
</code></pre><p>这就是我们在上节课中提到的探针<code>probe</code>，探测的是 <code>ngx_http_lua_shdict_set_helper</code> 这个函数。而这个函数的调用，都是在 <code>lua-nginx-module</code> 模块的 <code>lua-nginx-module/src/ngx_http_lua_shdict.c</code> 文件中：</p><pre><code>static int
ngx_http_lua_shdict_add(lua_State *L)
{
return ngx_http_lua_shdict_set_helper(L, NGX_HTTP_LUA_SHDICT_ADD);
}

static int
ngx_http_lua_shdict_safe_add(lua_State *L)
{
return ngx_http_lua_shdict_set_helper(L, NGX_HTTP_LUA_SHDICT_ADD
|NGX_HTTP_LUA_SHDICT_SAFE_STORE);
}

static int
ngx_http_lua_shdict_replace(lua_State *L)
{
return ngx_http_lua_shdict_set_helper(L, NGX_HTTP_LUA_SHDICT_REPLACE);
}
</code></pre><p>这样，我们只要探测这个函数，就可以追踪到共享字典的所有操作了。</p><h2>on CPU 和 off CPU</h2><p>在使用 OpenResty 的过程中，你最常遇到的应该就是性能问题了把。性能比较差，也就是 QPS 很低的表现主要有两类，CPU 占用过高和 CPU 占用过低。前者的瓶颈，可能是没有使用我们之前介绍过的性能优化的方法；而后者可能是因为使用了阻塞函数。相对应的，on CPU 和 off CPU 火焰图，可以帮助我们确认最终的根源所在。</p><p>要生成 C 级别的 on CPU 火焰图，你需要使用 systemtap-toolkit 中的<code>sample-bt</code>；而 Lua 级别的 on CPU 火焰图，则是由 stapxx 中的 <code>lj-lua-stacks</code> 来生成的。</p><p>我们以 <code>sample-bt</code> 为例来介绍下如何使用。<code>sample-bt</code> 这个脚本，可以对你指定的任意用户进程（不仅限于 Nginx 和 OpenResty 进程），来进行调用栈的采样。</p><p>例如，我们可以用下列代码，对一个正在运行的 Nginx worker 进程（PID 是 8736）采样 5 秒钟：</p><pre><code>$ ./sample-bt -p 8736 -t 5 -u &gt; a.bt
WARNING: Tracing 8736 (/opt/nginx/sbin/nginx) in user-space only...
WARNING: Missing unwind data for module, rerun with 'stap -d stap_df60590ce8827444bfebaf5ea938b5a_11577'
WARNING: Time's up. Quitting now...(it may take a while)
WARNING: Number of errors: 0, skipped probes: 24
</code></pre><p>它输出的结果文件 a.bt， 可以使用 FlameGraph 工具集来生成火焰图:</p><pre><code>stackcollapse-stap.pl a.bt &gt; a.cbt
flamegraph.pl a.cbt &gt; a.svg
</code></pre><p>这里的<code>a.svg</code> ，就是生成的火焰图，你可以用浏览器打开查看。不过要注意，在采样期间，我们需要保持一定的请求压力，否则采样数为 0 的话，就没办法生成火焰图了。</p><p>接着我们再来看下如何采样 off CPU，你需要使用的脚本是 systemtap-toolkit 中的 <code>sample-bt-off-cpu</code>。它的使用方法和 <code>sample-bt</code> 类似，我也写在了下面的代码中：</p><pre><code>$ ./sample-bt-off-cpu -p 10901 -t 5 &gt; a.bt
WARNING: Tracing 10901 (/opt/nginx/sbin/nginx)...
WARNING: _stp_read_address failed to access memory location
WARNING: Time's up. Quitting now...(it may take a while)
WARNING: Number of errors: 0, skipped probes: 23
</code></pre><p>在stapxx 中，分析延迟的工具是<code>epoll-loop-blocking-distr</code>，它会对指定的用户进程进行采样，并输出连续的 <code>epoll_wait</code> 系统调用之间的延迟分布：</p><pre><code>$ ./samples/epoll-loop-blocking-distr.sxx -x 19647 --arg time=60
Start tracing 19647...
Please wait for 60 seconds.
Distribution of epoll loop blocking latencies (in milliseconds)
max/avg/min: 1097/0/0
value |-------------------------------------------------- count
    0 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@  18471
    1 |@@@@@@@@                                            3273
    2 |@                                                    473
    4 |                                                     119
    8 |                                                      67
   16 |                                                      51
   32 |                                                      35
   64 |                                                      20
  128 |                                                      23
  256 |                                                       9
  512 |                                                       2
 1024 |                                                       2
 2048 |                                                       0
 4096 |                                                       0
</code></pre><p>你可以看到，这个输出结果显示，绝大部分延迟都小于 1 毫秒，但也有少数是在 200 毫秒以上的，这些就是需要关注的。</p><h2>上游和阶段跟踪</h2><p>除了 OpenResty 的代码本身可能出现性能问题外，当 OpenResty 通过 <code>cosocket</code> 或者 <code>proxy_pass</code> 这样的上游模块，与上游服务进行通信时，如果上游服务自身的延时比较大，也会对整体的性能带来很大的影响。</p><p>这个时候，你可以使用 <code>ngx-lua-tcp-recv-time</code>、<code>ngx-lua-udp-recv-time</code> 和 <code>ngx-single-req-latency</code> 这几个工具来进行分析，这里我以 <code>ngx-single-req-latency</code> 为例解释下。</p><p>这个工具和工具集里面的大部分工具并不太一样。其他工具，多是基于大量的采样和统计分析，得出一个数学上的分布结论。而 <code>ngx-single-req-latency</code> 分析的却是单个的请求，跟踪出单个请求在 OpenResty 中各个阶段的耗时，比如 rewrite、access、content 阶段以及上游的耗时。</p><p>我们可以来看一个具体的示例代码：</p><pre><code># making the ./stap++ tool visible in PATH:
    $ export PATH=$PWD:$PATH

    # assuming an nginx worker process's pid is 27327
    $ ./samples/ngx-single-req-latency.sxx -x 27327
    Start tracing process 27327 (/opt/nginx/sbin/nginx)...

    POST /api_json
        total: 143596us, accept() ~ header-read: 43048us, rewrite: 8us, pre-access: 7us, access: 6us, content: 100507us
        upstream: connect=29us, time-to-first-byte=99157us, read=103us

    $ ./samples/ngx-single-req-latency.sxx -x 27327
    Start tracing process 27327 (/opt/nginx/sbin/nginx)...

    GET /robots.txt
        total: 61198us, accept() ~ header-read: 33410us, rewrite: 7us, pre-access: 7us, access: 5us, content: 27750us
        upstream: connect=30us, time-to-first-byte=18955us, read=96us
</code></pre><p>这个工具会跟踪它启动后遇到的第一个请求。输出的内容和 opentracing 非常类似，你甚至可以把 systemtap-toolkit 和 stapxx ，当作是 OpenResty 中 APM（应用性能管理）的非侵入版本。</p><h2>写在最后</h2><p>除了今天我讲到的这些常用工具，OpenResty 自然还提供了更多的工具，它们就交给你自己去探索和学习了。</p><p>其实，在追踪技术方面，OpenResty 和其他的开发语言、平台相比，还有一个比较大的不同之处，希望你可以慢慢体会：</p><blockquote>
<p>保持代码基的简洁和稳定，不要在其中增加探针，而是通过外部动态跟踪的技术来进行采样。</p>
</blockquote><p>最后给你留一个问题，你在使用 OpenResty 的时候，使用过哪些工具来进行跟踪和分析问题呢？欢迎留言和我探讨这个问题，也欢迎你把这篇文章分享出去，我们一起交流和进步。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/7c/f6/c7e78819.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>tenglonsen</span>
  </div>
  <div class="_2_QraFYR_0">温老师，我现在有用openresty作为mysql的中间件来用，然而我们数据库中的数据存在很多大字段，业务方会通过http接口拉取大量数据，这样nginx的worker进程跑着跑着内存就到3G+去了，通过火焰图看出大部分CPU时间是在mysql.lua文件中，目前找了很久，没有好的思路去定位这个内存问题，只能每天定时重启一下。。。请问一下这个问题有什么好的定位方法没，或者说是不是openresty不适合这种大量数据的使用场景呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-28 18:22:21</div>
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
  <div class="_2_QraFYR_0">老师想请问下如果升级了最新的1.15.8版本，如何使用火焰图呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 编译的时候，把LuaJIT 的 64 位支持去掉</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-21 08:07:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_584aa7</span>
  </div>
  <div class="_2_QraFYR_0">除了这两款工具结合使用排查性能问题，还有其他开源工具吗？如果系统不是基于openresty但是也是基于ngnix+lua实现的，也能用您说的这两款工具结合排查性能问题吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-29 09:33:23</div>
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
  <div class="_2_QraFYR_0">openresty xray 会不会比这些工具更好用？https:&#47;&#47;openresty.com&#47;en&#47;xray&#47;</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-22 12:41:28</div>
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
  <div class="_2_QraFYR_0">老师，对工作进程进行采样的时候，是否会对工作进程的性能产生可见的影响？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 性能影响大概在 5% 以内。但是使用 systemtap 可能会对系统造成其他影响，我们之前遇到过采样脚本影响网络的事情。所以在生产环境采样还是要谨慎。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-03 07:57:23</div>
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
  <div class="_2_QraFYR_0">local require = require  -- 老师， 代码文件开始需要加这条吗，我看有人这么干的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果你用了 luacheck 等代码风格的检测工具，是要这么做的，避免全局变量。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-19 16:11:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4d/fd/0aa0e39f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>许童童</span>
  </div>
  <div class="_2_QraFYR_0">老师来点图片，一图胜千言啊。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-19 15:08:34</div>
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
  <div class="_2_QraFYR_0">这个可以做一些更加深入的一些介绍么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 篇幅有限，这种优化工具需要合适的场景才更有代入感：）<br>你看下后面视频课里面的示例是否详细一些？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-19 09:24:19</div>
  </div>
</div>
</div>
</li>
</ul>