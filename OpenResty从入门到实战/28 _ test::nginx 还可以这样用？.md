<audio title="28 _ test::nginx 还可以这样用？" src="https://static001.geekbang.org/resource/audio/3f/21/3f20a1b788027e4950de581b9bce6621.mp3" controls="controls"></audio> 
<p>你好，我是温铭。</p><p>在前面两个章节中，你已经掌握了 <code>test::nginx</code> 的大部分使用方法，我相信你已经能够看明白 OpenResty 项目中大部分的测试案例集了。这对于学习 OpenResty 和它的周边库而言，已经足够了。</p><p>但如果你有志于成为 OpenResty 的代码贡献者，或者你正在自己的项目中使用 <code>test::nginx</code> 来编写测试案例，那么你还需要来学习一些更高级、更复杂的用法。</p><p>今天的内容，可能会是这个专栏中最“高冷”的部分，因为这都是从来没有人分享过的内容。 以 lua-nginx-module 这个 OpenResty 中最核心的模块为例，全球一共有 70 多个贡献者，但并非每个贡献者都写过测试案例。所以，如果学完今天的课程，你在 <code>test::nginx</code> 上的理解，绝对可以进入全球 Top 100。</p><h2>测试中的调试</h2><p>首先，我们来看几个最简单、也是开发者最常用到的原语，它们在平时的调试中会被使用到。下面，我们就来依次介绍下，这几个调试相关的原语的使用场景。</p><h3>ONLY</h3><p>很多时候，我们都是在原有的测试案例集基础上，新增了一个测试案例。如果这个测试文件包含了很多的测试案例，那么从头到尾跑一遍显然是比较耗时的，这在你需要反复修改测试案例的时候尤为明显。</p><!-- [[[read_end]]] --><p>那么，有没有什么方法只运行你指定的某一个测试案例呢？ <code>ONLY</code> 这个标记可以轻松实现这一点：</p><pre><code>=== TEST 1: sanity
=== TEST 2: get
--- ONLY
</code></pre><p>上面这段伪码就展示了如何使用这个原语。把 <code>--- ONLY</code> 放在需要单独运行的测试案例的最后一行，那么使用 prove 来运行这个测试案例文件的时候，就会忽略其他所有的测试案例，只运行这一个测试了。</p><p>不过，这只适合在你做调试的时候使用。所以， prove 命令发现 ONLY 标记的时候，也会给出提示，告诉你不要忘记在提交代码时把它去掉。</p><h3>SKIP</h3><p>与只执行一个测试案例对应的需求，就是忽略掉某一个测试案例。<code>SKIP</code> 这个标记，一般用于测试尚未实现的功能：</p><pre><code>=== TEST 1: sanity
=== TEST 2: get
--- SKIP
</code></pre><p>从这段伪码你可以看到，它的用法和ONLY类似。因为我们是测试驱动开发，需要先编写测试案例；而在集体编码实现时，可能由于实现难度或者优先级的关系，导致某个功能需要延后实现。那么这时候，你就可以先跳过对应的测试案例集，等实现完成后，再把 SKIP 标记去掉即可。</p><h3>LAST</h3><p>还有一个常用的标记是 <code>LAST</code>，它的用法也很简单，在它之前的测试案例集都会被执行，后面的就会被忽略掉：</p><pre><code>=== TEST 1: sanity
=== TEST 2: get
--- LAST
=== TEST 3: set
</code></pre><p>你可能疑惑，ONLY和SKIP我能理解，但LAST这个功能有什么用呢？实际上，有时候你的测试案例是有依赖关系的，需要你执行完前面几个测试案例后，之后的测试才有意义。那么，在这种情况下去调试的话，LAST 就非常有用了。</p><h2>测试计划 plan</h2><p>在 <code>test::nginx</code> 所有的原语中，<code>plan</code> 是最容易让人抓狂、也是最难理解的一个。它源自于 perl 的 <code>Test::Plan</code> 模块，所以文档并不在 <code>test::nginx</code>中，找到它的解释并不容易，所以我把它放在靠前的位置来介绍。我见过好几个 OpenResty 的代码贡献者，都在这个坑里面跌倒，甚至爬不出来。</p><p>下面是一个示例，在 OpenResty 官方测试集的每一个文件的开始部分，你都能看到类似的配置：</p><pre><code>plan tests =&gt; repeat_each() * (3 * blocks());
</code></pre><p>这里 plan 的含义是，在整个测试文件中，按照计划应该会做多少次检测项。如果最终运行的结果和计划不符，整个测试就会失败。</p><p>拿这个示例来说，如果 <code>repeat_each</code> 的值是 2，一共有 10 个测试案例，那么 plan 的值就应该是2 x 3 x 10 = 60。这里估计你唯一搞不清楚的，就是数字 3 的含义吧，看上去完全是一个 magic number！</p><p>别着急，我们继续看示例，一会儿你就能搞懂了。先来说说，你能算清楚下面这个测试案例中，plan 的正确值是多少吗？</p><pre><code>=== TEST 1: sanity
--- config
    location /t {
        content_by_lua_block {
            ngx.say(&quot;hello&quot;)
        }
    }
--- request
GET /t
--- response_body
hello
</code></pre><p>我相信所有人都会得出 plan = 1 的结论，因为测试中只对 <code>response_body</code> 进行了校验。</p><p>但，事实并非如此！正确的答案是， plan = 2。为什么呢？因为 <code>test::nginx</code> 中隐含了一个校验，也就是<code>--- error_code: 200</code>，它默认检测  HTTP 的 response code 是否为 200。</p><p>所以，上面的 magic number 3，真实含义是在每一个测试中都显式地检测了两次，比如 body 和 error log；同时，隐式地检测了 response code。</p><p>由于这个地方太容易出错，所以，我的建议是，推荐你用下面的方法，直接关闭掉 plan：</p><pre><code>use Test::Nginx::Socket 'no_plan';
</code></pre><p>如果无法关闭，比如在 OpenResty 的官方测试集中遇到 plan 不准确的情况，建议你也不要去深究原因，直接在 plan 的表达式中增加或者减少数字即可：</p><pre><code>plan tests =&gt; repeat_each() * (3 * blocks()) + 2;
</code></pre><p>这也是官方会使用到的方法。</p><h2>预处理器</h2><p>我们知道，在同一个测试文件的不同测试案例之间，可能会有一些共同的设置。如果在每一个测试案例中都重复设置，就会让代码显得冗余，后面修改起来也比较麻烦。</p><p>这时候，你就可以使用 <code>add_block_preprocessor</code> 指令，来增加一段 perl 代码，比如下面这样来写：</p><pre><code>add_block_preprocessor(sub {
    my $block = shift;

    if (!defined $block-&gt;config) {
        $block-&gt;set_value(&quot;config&quot;, &lt;&lt;'_END_');
    location = /t {
        echo $arg_a;
    }
    _END_
    }
});
</code></pre><p>这个预处理器，就会为所有的测试案例，都增加一段 config 的配置，而里面的内容就是 <code>location /t</code>。这样，在你后面的测试案例里，就都可以省略掉 config，直接访问即可：</p><pre><code>=== TEST 1:
--- request
    GET /t?a=3
--- response_body
3

=== TEST 2:
--- request
    GET /t?a=blah
--- response_body
blah
</code></pre><h2>自定义函数</h2><p>除了在预处理器中增加 perl 代码之外，你还可以在 <code>run_tests</code> 原语之前，随意地增加 perl 函数，也就是我们所说的自定义函数。</p><p>下面是一个示例，它增加了一个读取文件的函数，并结合 <code>eval</code> 指令，一起实现了 POST 文件的功能：</p><pre><code>sub read_file {
    my $infile = shift;
    open my $in, $infile
        or die &quot;cannot open $infile for reading: $!&quot;;
    my $content = do { local $/; &lt;$in&gt; };
    close $in;
    $content;
}

our $CONTENT = read_file(&quot;t/test.jpg&quot;);

run_tests;

__DATA__

=== TEST 1: sanity
--- request eval
&quot;POST /\n$::CONTENT&quot;
</code></pre><h2>乱序</h2><p>除了上面几点外，<code>test::nginx</code> 还有一个鲜为人知的坑：默认乱序、随机来执行测试案例，而非按照测试案例的前后顺序和编号来执行。</p><p>它的初衷是想测试出更多的问题。毕竟，每一个测试案例运行完后，都会关闭 Nginx 进程，并启动新的 Nginx 来执行，结果不应该和顺序相关才对。</p><p>对于底层的项目而言，确实如此。但是，对于应用层的项目来说，外部存在数据库等持久化存储。这时候的乱序执行，就会导致错误的结果。由于每次都是随机的，所以可能报错，也可能不报错，每次的报错还可能不同。这显然会给开发者带来困惑，就连我都在这里跌倒过好多次。</p><p>所以，我的忠告就是：请关闭掉这个特性。你可以用下面这两行代码来关闭：</p><pre><code>no_shuffle();
run_tests;
</code></pre><p>其中，<code>no_shuffle</code> 原语就是用来禁用随机，让测试严格按照测试案例的前后顺序来运行。</p><h2>reindex</h2><p>最后，让我们聊一个不烧脑的、轻松一点儿的话题。OpenResty 的测试案例集，对格式有着严格的要求。每个测试案例之间都需要有 3 个换行来分割，测试案例的编号也要严格保持自增长。</p><p>幸好，我们有对应的自动化工具 <code>reindex</code> 来做这些繁琐的事情，它隐藏在 <a href="https://github.com/openresty/openresty-devel-utils">[ openresty-devel-utils]</a>  项目中，因为没有文档来介绍，知道的人很少。</p><p>有兴趣的同学，可以尝试着把测试案例的编号打乱，或者增删分割的换行个数，然后用这个工具来整理下，看看是否可以还原。</p><h2>写在最后</h2><p>关于 <code>test::nginx</code> 的介绍就到此结束了。当然，它的功能其实还有更多，我们只讲了最核心最重要的一些。授人以鱼不如授人以渔，学习测试的基本方法和注意点我都已经教给你了，剩下的就需要你自己去官方的测试案例集中去挖掘了。</p><p>最后给你留一个问题。在你的项目开发中，是否有测试？你又是使用什么框架来测试的呢？欢迎留言和我交流这个问题，也欢迎你把这篇文章分享给更多的人，一起交流和学习。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/97/69/80945634.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>罐头瓶子</span>
  </div>
  <div class="_2_QraFYR_0">请问，我有一个接口需要调用两次来做测试，第一次返回和第二次返回的结果要保持一致。我的想法是第一次返回的body需要保存记录下来，第二次请求的body与保存的第一次body做对比。请问我如何保存第一次返回的body?</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-30 17:27:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/51/61/e4b1b737.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>K1</span>
  </div>
  <div class="_2_QraFYR_0"> ssl相关功能用test::nginx是不是测不了？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以测试的，你可以看下 https:&#47;&#47;github.com&#47;openresty&#47;lua-nginx-module&#47;blob&#47;master&#47;t 这个目录下面包含 ssl、tls 的测试文件，比如 https:&#47;&#47;github.com&#47;openresty&#47;lua-nginx-module&#47;blob&#47;master&#47;t&#47;139-ssl-cert-by.t。<br><br>OpenResty 会先生成一批测试用的证书，放在 https:&#47;&#47;github.com&#47;openresty&#47;lua-nginx-module&#47;tree&#47;master&#47;t&#47;cert 目录中，然后在测试案例中使用。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-29 11:46:38</div>
  </div>
</div>
</div>
</li>
</ul>