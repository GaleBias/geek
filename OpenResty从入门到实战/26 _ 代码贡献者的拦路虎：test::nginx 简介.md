<audio title="26 _ 代码贡献者的拦路虎：test::nginx 简介" src="https://static001.geekbang.org/resource/audio/b3/64/b3fb7ed797152551cbee231d8a946864.mp3" controls="controls"></audio> 
<p>你好，我是温铭。</p><p>测试，是软件开发中必不可少的一个重要环节。测试驱动开发（TDD）的理念已经深入人心，几乎每家软件公司都有 QA 团队来负责测试的工作。</p><p>测试也是 OpenResty 质量稳定和好口碑的基石，不过同时，它也是 OpenResty 众多开源项目中最被人忽视的部分。很多开发者每天都在使用 lua-nginx-module，偶尔跑一跑火焰图，但有几个人会去运行测试案例呢？甚至很多基于 OpenResty 的开源项目，都是没有测试案例的。但没有测试案例和持续集成的开源项目，显然是不值得信赖的。</p><p>不过，和商业公司不同的是，大部分的开源项目都没有专职的测试工程师，那么它们是如何来保证代码质量的呢？答案很简单，就是“自动化测试”和“持续集成”，关键点在于自动和持续，而OpenResty 在这两个方面都做到了极致。</p><p>OpenResty 有 70 个开源项目，它们的单元测试、集成测试、性能测试、mock 测试、fuzz 测试等工作量，是无法靠社区的人力解决的。所以，OpenResty 一开始在自动化测试上的投入就比较大。这样做短期看起来会拖慢项目进度，但可以说是一劳永逸，长期来看在这方面的投入是非常划算的。因此，每当我和其他工程师聊起 OpenResty 在测试方面的思路和工具集时，他们都会惊叹不已。</p><!-- [[[read_end]]] --><p>下面，我们就先来说说OpenResty的测试理念。</p><h2>理念</h2><p><code>test::nginx</code> 是 OpenResty 测试体系中的核心，OpenResty 本身和周边的 lua-rety 库，都是使用它来组织和编写测试集的。虽然它一个是测试框架，但它的<strong>门槛非常高</strong>。这是因为， <code>test::nginx</code> 和一般的测试框架不同，并非基于断言，也不使用 Lua 语言，这就要求开发者从零开始学习和使用 <code>test::nginx</code>，并得扭转自身对测试框架固有的认知。</p><p>我认识几个 OpenResty 的贡献者，他们可以流畅地给 OpenResty 提交 C 和 Lua 代码，但在使用 <code>test::nginx</code> 编写测试用例时都卡壳了，要么不知道怎么写，要么遇到测试跑不过时不知道如何解决。所以，我把 <code>test::nginx</code> 称为代码贡献者的拦路虎。</p><p><code>test::nginx</code> <strong>糅合了Perl、数据驱动以及 DSL（领域小语言）</strong>。对于同一份测试案例集，通过对参数和环境变量的控制，可以实现乱序执行、多次重复、内存泄漏检测、压力测试等不同的效果。</p><h2>安装和示例</h2><p>说了这么多概念，让我们来对 <code>test::nginx</code> 有一个直观的认识吧。在使用前，我们先来看下如何安装。</p><p>关于 OpenResty 体系内软件的安装，只有官方 CI 中的安装方法才是最及时和有效的，其他方式的安装总是会遇到各种各样的问题。所以，我总是推荐你去参考它在 travis 中的<a href="https://github.com/openresty/lua-resty-core/blob/master/.travis.yml">方法</a>。</p><p><code>test::nginx</code> 的安装和使用也不例外，在 travis 中，它可以分为 4 步。</p><p><strong>1. </strong>先安装 Perl 的包管理器 cpanminus。<br>
<strong>2. </strong>然后，通过 cpanm 来安装 <code>test::nginx</code>：</p><pre><code>sudo cpanm --notest Test::Nginx IPC::Run &gt; build.log 2&gt;&amp;1 || (cat build.log &amp;&amp; exit 1)
</code></pre><p><strong>3. </strong>再接着， clone 最新的源码：</p><pre><code>git clone https://github.com/openresty/test-nginx.git
</code></pre><p><strong>4. </strong>最后，通过 Perl 的 <code>prove</code> 命令来加载 test-nginx 的库，并运行 <code>/t</code> 目录下的测试案例集：</p><pre><code>prove -Itest-nginx/lib -r t
</code></pre><p>安装完以后，让我们看下 <code>test::nginx</code> 中最简单的测试案例。下面这段代码改编自<a href="https://metacpan.org/pod/Test::Nginx::Socket">官方文档</a>，我已经把个性化的控制参数都去掉了：</p><pre><code>use Test::Nginx::Socket 'no_plan';


run_tests();

__DATA__

=== TEST 1: set Server
--- config
    location /foo {
        echo hi;
        more_set_headers 'Server: Foo';
    }
--- request
    GET /foo
--- response_headers
Server: Foo
--- response_body
hi
</code></pre><p>虽然 <code>test::nginx</code> 是用 Perl 编写的，并且是其中的一个模块，但从上面的测试中，你是不是完全看不到，Perl 或者其他任何其他语言的影子呀？有这个感觉这就对了。因为，<code>test::nginx</code> 本身就是作者自己用 Perl 实现的 DSL（小语言），是专门针对 Nginx 和 OpenResty 的测试而抽象出来的。</p><p>所以，当你第一次看到这种测试的时候，大概率是看不懂的。不过不用着急，让我来为“你庖丁解牛”，分析以下上面的测试案例吧。</p><p>首先是 <code>use Test::Nginx::Socket;</code>，这是 Perl 里面引用库的方式，就像 Lua 里面 require 一样。这也在提醒我们，<code>test::nginx</code>  是一个 Perl 程序。</p><p>第二行的<code>run_tests();</code> ，是 <code>test::nginx</code>  中的一个 Perl 函数，它是测试框架的入口函数。如果你还想调用 <code>test::nginx</code>  中其他的 Perl 函数，都要放在 <code>run_tests</code> 之前才有效。</p><p>第三行的 <code>__DATA__</code> 是一个标记，表示它下面的都是测试数据。Perl 函数都应该在这个标记之前完成。</p><p>接下来的 <code>=== TEST 1: set Server</code>，是测试案例的标题，是为了注明这个测试的目的，它里面的数字编号有工具可以自动排列。</p><p><code>--- config</code> 是 Nginx 配置段。在上面的案例中，我们用的都是 Nginx 的指令，没有涉及到 Lua。如果你要添加 Lua 代码，也是在这里用类似 content_by_lua 的指令完成的。</p><p><code>--- request</code> 用于模拟终端来发送一个请求，下面紧跟的 <code>GET /foo</code> ，则指明了请求的方法和 URI。</p><p><code>--- response_headers</code>，是用来检测响应头的。下面的 <code>Server: Foo</code> 表示在响应头中必须出现的 header 和 value，如果没有出现，测试就会失败。</p><p>最后的<code>--- response_body</code>，是用来检测相应体的。下面的 <code>hi</code> 则是响应体中必须出现的字符串，如果没有出现，测试就会失败；</p><p>好了，到这里，最简单的测试案例就分析完了，你看明白了吗？如果哪里还不清楚，一定要及时留言提问暴露出来，毕竟，能够看懂测试案例，是完成 OpenResty 相关开发工作的前提。</p><h2>编写自己的测试案例</h2><p>光说不练假把式，接下来，我们就该进入动手试验环节了。还记得上节课中，我们是如何测试 memcached server 的吗？没错，我们是用 <code>resty</code> 来手动发送请求的，也就是用下面这段代码表示：</p><pre><code>$ resty -e 'local memcached = require &quot;resty.memcached&quot;
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
</code></pre><p>不过，是不是觉得手动发送还不够智能呢？没关系，在学习完 <code>test::nginx</code>  之后，我们就可以尝试把手动的测试变为自动化的了，比如下面这段代码：</p><pre><code>use Test::Nginx::Socket::Lua::Stream;

run_tests();

__DATA__
  
=== TEST 1: basic get and set
--- config
        location /test {
            content_by_lua_block {
                local memcached = require &quot;resty.memcached&quot;
                local memc, err = memcached:new()
                if not memc then
                    ngx.say(&quot;failed to instantiate memc: &quot;, err)
                    return
                end

                memc:set_timeout(1000) -- 1 sec
                local ok, err = memc:connect(&quot;127.0.0.1&quot;, 11212)

                local ok, err = memc:set(&quot;dog&quot;, 32)
                if not ok then
                    ngx.say(&quot;failed to set dog: &quot;, err)
                    return
                end

                local res, flags, err = memc:get(&quot;dog&quot;)
                ngx.say(&quot;dog: &quot;, res)
            }
        }

--- stream_config
    lua_shared_dict memcached 100m;

--- stream_server_config
    listen 11212;
    content_by_lua_block {
        local m = require(&quot;memcached-server&quot;)
        m.go()
    }

--- request
GET /test
--- response_body
dog: 32
--- no_error_log
[error]
</code></pre><p>在这个测试案例中，我新增了 <code>--- stream_config</code>、<code>--- stream_server_config</code>、<code>--- no_error_log</code> 这些配置项，但它们的本质上都是一样的，即：</p><p><strong>通过抽象好的原语（也可以看做配置），把测试的数据和检测进行剥离，让可读性和扩展性变得更好。</strong></p><p>这就是 <code>test::nginx</code> 和其他测试框架的根本不同之处。这种 DSL 是一把双刃剑，它可以让测试逻辑变得清晰和方便扩展，但同时也提高了学习的门槛，你需要重新学习新的语法和配置才能开始编写测试案例。</p><h2>写在最后</h2><p>不得不说，<code>test::nginx</code>  虽然强大，但很多时候，它可能不一定适合你的场景。杀鸡焉用宰牛刀？在 OpenResty 中，你也选择使用断言风格的测试框架 <code>busted</code>。<code>busted</code>结合 <code>resty</code> 这个命令行工具，也可以满足不少测试的需求。</p><p>最后，给你留一个作业题，你可以在本地把 memcached 的这个测试跑起来吗？如果你能新增一个测试案例，那就更棒了。</p><p>欢迎在留言区记录你的操作和心得，也可以写下你今天学习的疑惑地方。同时，欢迎你把这篇文章分享给更多对OpenResty感兴趣的人，我们一起交流和探讨。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/51/61/e4b1b737.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>K1</span>
  </div>
  <div class="_2_QraFYR_0">最近开始学习Perl了，因为在复杂一点的测试案例中，需要在DATA前做一些事情，而又不得不使用perl；虽然已经抽象成数据驱动了，但是还是有很多细节涉及到perl语法，对于从未接触过perl的人来说，因为一些细节问题，会耽误很长时间。<br><br>test::nginx确实是门槛，是代码贡献者和二次开发者们的门槛。<br><br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，perl 的语法真不是一般人能够适应的，需要一个明显的学习曲线</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-24 11:49:07</div>
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
  <div class="_2_QraFYR_0">请问，怎么修改一些非nginx的标准配置呢？也是用config吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-18 10:58:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/6b/43/b6bcab56.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>三叶虫tlb</span>
  </div>
  <div class="_2_QraFYR_0">测试报错<br>Can&#39;t determine section names. Need two sections in first block at t&#47;01-memd.t line 0.<br>END failed--call queue aborted.<br># Looks like your test exited with 3 just after 3.<br><br>看了下报错提示和文档<br>改为：use Test::Nginx::Socket::Lua::Stream &#39;no_plan&#39;;<br>或：use Test::Nginx::Socket::Lua::Stream tests =&gt; 3;<br>测试通过<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-08 22:27:20</div>
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
  <div class="_2_QraFYR_0">老师 我一直有如下错误：<br>prove -Itest-nginx&#47;lib -r t&#47;mem_server.t <br>t&#47;mem_server.t .. nginx: [alert] could not open error log file: open() &quot;&#47;usr&#47;local&#47;var&#47;log&#47;nginx&#47;error.log&quot; failed (13: Permission denied)<br>t&#47;mem_server.t .. 1&#47;? Can&#39;t determine section names. Need two sections in first block at t&#47;mem_server.t line 0.<br>END failed--call queue aborted.<br># Looks like your test exited with 3 just after 3.<br>t&#47;mem_server.t .. Dubious, test returned 3 (wstat 768, 0x300)<br>All 3 subtests passed <br><br>Test Summary Report<br>-------------------<br>t&#47;mem_server.t (Wstat: 768 Tests: 3 Failed: 0)<br>  Non-zero exit status: 3<br>Files=1, Tests=3,  0 wallclock secs ( 0.02 usr  0.01 sys +  0.12 cusr  0.04 csys =  0.19 CPU)<br>Result: FAIL<br><br><br>请问如何解决？实在没办法了。。。。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-20 14:48:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/83/5e/7f211555.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>undefined　　　</span>
  </div>
  <div class="_2_QraFYR_0">#!&#47;usr&#47;bin&#47;env bash<br><br>export PATH=&#47;usr&#47;local&#47;openresty&#47;nginx&#47;sbin&#47;nginx:$PATH<br><br># 部分系统perl环境会提示(Can&#39;t locate t&#47;.pm in @INC)<br>export PERL5LIB=$(pwd):$PERL5LIB<br><br>exec prove &quot;$@&quot;</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-07 19:50:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/83/5e/7f211555.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>undefined　　　</span>
  </div>
  <div class="_2_QraFYR_0">prove -Itest-nginx&#47;lib -r t<br>t&#47;hello.t .. Bailout called.  Further testing stopped:  Failed to get the version of the Nginx in PATH<br>Can&#39;t determine section names. Need two sections in first block at &#47;usr&#47;local&#47;share&#47;perl5&#47;Test2&#47;Hub.pm line 395.<br>END failed--call queue aborted at t&#47;hello.t line 395, &lt;DATA&gt; line 1.<br>FAILED--Further testing stopped: Failed to get the version of the Nginx in PATH</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-07 19:45:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/64/9b/d1ab239e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>J.Smile</span>
  </div>
  <div class="_2_QraFYR_0">.&#47;resty -e &#39;local memcached = require &quot;resty.memcached&quot;<br>&gt;     local memc, err = memcached:new()<br>&gt; <br>&gt;     memc:set_timeout(1000)<br>&gt;     local ok, err = memc:connect(&quot;127.0.0.1&quot;, 11212)<br>&gt;     local ok, err = memc:set(&quot;dog&quot;, 32)<br>&gt;     if not ok then<br>&gt;         ngx.say(&quot;failed to set dog: &quot;, err)<br>&gt;         return<br>&gt;     end<br>&gt; <br>&gt;     local res, flags, err = memc:get(&quot;dog&quot;)<br>&gt;     ngx.say(&quot;dog: &quot;, res)&#39;<br><br>-----执行结果是：<br>failed to set dog: closed<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你可以在这一行的后面判断下 err:<br>local ok, err = memc:connect(&quot;127.0.0.1&quot;, 11212)<br><br>可能是 memcached 的服务没有启动。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-09 17:43:44</div>
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
  <div class="_2_QraFYR_0">老师，DSL的翻译应该是领域专用语言？搜领域小语言没有搜到，搜到了这个DSL（Domain Specific Language）https:&#47;&#47;en.wikipedia.org&#47;wiki&#47;Domain-specific_language</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，就是领域专用语言。之所以叫它小语言，是因为它不是通用的编程语言，只在特定领域内使用。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-27 15:22:26</div>
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
  <div class="_2_QraFYR_0">问个可能很白痴的问题，如果环境里安装了好几个不同版本或者不同编译参数的nginx，怎么配置test::nginx使用指定的的nginx呢？在执行的时候，发现默认使用的是一个不支持stream的nginx，然后看prove 提供的参数也没有，感觉是在.t文件里指定，请老师指点一下~</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是从系统的 PATH 里面查找 nginx 的路径的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-27 14:47:51</div>
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
  <div class="_2_QraFYR_0">上面的案例，还有github文档中mysql 的例子中set_timeout通常都是设置成1秒，但是并发大的情况下经常会出现超时的情况，一般怎么评估自己工程中timeout应该设置成多少</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个是看具体的场景了，比如统计 MySQL 的日志，看看请求时间的分布情况。如果 MySQL 的慢查询比较多，很多超过 1 秒的，那这个时候更好的方案是去优化数据库。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-24 16:10:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/82/42/8b04d489.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>刘丹</span>
  </div>
  <div class="_2_QraFYR_0">请问执行完git clone后，是否要执行以下命令来安装test::nginx?<br>cd test-nginx<br>perl Makefile.PL<br>make<br>sudo make install</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 并非如何，你可以参考一些开源项目的 travis 的做法：<br>1. 先通过包管理器安装：<br>sudo cpanm --notest Test::Nginx &gt;build.log 2&gt;&amp;1 || (cat build.log &amp;&amp; exit 1)<br><br>https:&#47;&#47;github.com&#47;iresty&#47;apisix&#47;blob&#47;master&#47;.travis&#47;linux_runner.sh#L20<br><br>2. git clone 最新的 test::nginx:<br>https:&#47;&#47;github.com&#47;iresty&#47;apisix&#47;blob&#47;master&#47;.travis&#47;linux_runner.sh#L35<br><br>3. 用 prove 命令的时候，把 test nginx 的目录包含进去：<br>prove -Itest-nginx&#47;lib -r t</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-24 12:42:38</div>
  </div>
</div>
</div>
</li>
</ul>