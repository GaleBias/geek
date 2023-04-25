<audio title="29 _ 最容易失准的性能测试？你需要压测工具界的“悍马”wrk" src="https://static001.geekbang.org/resource/audio/be/db/be440b24e1d2edb06c0f5361a7e0fedb.mp3" controls="controls"></audio> 
<p>你好，我是温铭。</p><p>在测试章节的最后一节课，我和你来聊聊性能测试。这部分内容并非 OpenResty 独有，对于其他的后端服务来说，都是一样适用的。</p><p>性能测试很常见，在我们交付产品的时候，都会带有性能指标的要求，比如 QPS、TPS 达到多少，延时要低于多少毫秒，可以并发支持多少用户的连接等等。对于开源项目而言，我们发版本之前也会做一次性能测试，和上一个版本对比，看是否有明显的衰退。也有一些中立的网站，会发布同类产品的性能对比数据。不得不说，性能测试离我们真的很近。</p><p>在我的十几年的工作中，针对不同的产品做过很多次性能测试，中间也踩过不少坑。后来，我逐渐地发现，性能测试做起来简单，但做对却并不容易，甚至可以说，很多性能测试的结果都是失准的。</p><p>那么，如何做一个科学严谨的性能测试呢？今天这节课，且听我娓娓道来。</p><h2>性能测试工具</h2><p>工欲善其事，必先利其器。选择一个趁手的性能测试工具，是成功的一半。</p><p><code>ab</code> 这个 Apache Benchmark 工具你应该很熟悉，可以说是最简单的性能测试工具，但可惜的是并不好用。这是因为，当前服务端基本都基于协程和异步 I/O 来开发，性能不差；而 ab 利用不到机器的多核，生成的请求压力不够大。这种情况下，ab 测试得到的结果，并不真实，反而变成了 ab 自身的性能测试。</p><!-- [[[read_end]]] --><p>所以，我们可以明确选择压测工具的一个标准，那就是：<strong>工具自身的性能非常强悍，可以生成足够大的压力，压垮服务端程序。</strong></p><p>当然，你也可以有钱任性，启动很多压测客户端，变为分布式压测系统。这自然是可行的，但不要忘记，与此同时的复杂度也跟着上去了。</p><p>回到 OpenResty 的实践，我们推荐使用的性能测试工具是 wrk。先来说说，为什么选择它呢？</p><p>首先， wrk 满足工具选型的标准。单机的 wrk 产生的压力，可以轻松让 Nginx 跑满 CPU，其他服务端程序更是不在话下。</p><p>其次， wrk 和 OpenResty 有很多类似的地方。wrk 也不是从零开始编写的一个开源项目，它站在 LuaJIT 和 Redis 这两个巨人的肩膀上，充分利用了系统的多核资源来生成请求。除此之外，wrk 还暴露了 Lua API，你可以嵌入自己的 Lua 脚本，来自定义请求的头和内容，使用非常灵活。</p><p>那么该如何使用 wrk呢？也很简单，看下面这段代码的内容：</p><pre><code>wrk -t12 -c400 -d30s http://127.0.0.1:8080/index.html
</code></pre><p>这意味着 wrk 会使用 12 个线程，保持 400 个长连接，持续 30 秒钟，来给指定的 API 接口发送 HTTP 请求。当然，如果你不指定参数的话，wrk 会默认启动 2 个线程和 10 个长连接。</p><h2>测试环境</h2><p>找好测试工具后，我们还不能直接开始压力测试，还需要把测试环境给检查一遍，测试环境需要检查的主要有四项，下面我分别来详细讲讲。</p><h3>检查项一：关闭 SELinux</h3><p>如果你是 CentOS/RedHat 系列的操作系统，建议你关闭 SELinux，不然可能会遇到不少诡异的权限问题。</p><p>我们通过下面这个命令，查看 SELinux 是否开启：</p><pre><code>$ sestatus
SELinux status: disabled
</code></pre><p>如果显示是开启的（enforcing），你可以通过<code>$ setenforce 0</code>来临时关闭；同时修改 <code>/etc/selinux/config</code> 文件来永久关闭，将 <code>SELINUX=enforcing</code> 改为 <code>SELINUX=disabled</code>。</p><h3>检查项二：最大打开文件数</h3><p>然后，你需要用下面的命令，查看下当前系统的全局最大打开文件数：</p><pre><code>$ cat /proc/sys/fs/file-nr
3984 0 3255296
</code></pre><p>这里的最后一个数字，就是最大打开文件数。如果你的机器中这个数字比较小，那就需要修改 <code>/etc/sysctl.conf</code> 文件来增大：</p><pre><code>fs.file-max = 1020000
net.ipv4.ip_conntrack_max = 1020000
net.ipv4.netfilter.ip_conntrack_max = 1020000
</code></pre><p>修改完以后，还需要重启系统服务来生效：</p><pre><code>sudo sysctl -p /etc/sysctl.conf
</code></pre><h3>检查项三：进程限制</h3><p>除了系统的全局最大打开文件数，一个进程可以打开的文件数也是有限制的，你可以通过命令 <code>ulimit</code> 来查看：</p><pre><code>$ ulimit -n
1024
</code></pre><p>你会发现，这个值默认是 1024，是一个很低的数值。因为每一个用户请求都会对应着一个文件句柄，而压力测试会产生大量的请求，所以我们需要增大这个数值，把它改为百万级别，你可以用下面的命令来临时修改：</p><pre><code>$ ulimit -n 1024000
</code></pre><p>也可以修改配置文件 <code>/etc/security/limits.conf</code> 来永久生效：</p><pre><code>* hard nofile 1024000
* soft nofile 1024000
</code></pre><h3>检查项四：Nginx 配置</h3><p>最后，你还需要对 Nginx 的配置，做一个小的修改，也就是下面这两行代码的操作：</p><pre><code>events {
    worker_connections 10240;
}
</code></pre><p>这样，我们就可以把每个 worker 的连接数增大了。因为它的默认值只有 512，这在大压力的测试下显然是不够的。</p><h2>压测前检查</h2><p>到此为止，测试环境已经准备好了。一定有人蠢蠢欲动想要上手测试了吧？且慢，在使用 wrk 发起测试之前，让我们最后再来检测一次。毕竟，人总会犯错，换个角度来做一次交叉测试，是非常重要的。</p><p>最后的这次检测，可以分为两步。</p><h3>第一步，使用自动化工具 <code>c1000k</code>。</h3><p>它来自 SSDB 的作者：<a href="https://github.com/ideawu/c1000k">https://github.com/ideawu/c1000k</a>。从名字你就能看出来，这个工具的目的，就是用来检测你的环境是否可以满足100万并发连接的要求。</p><p>这个工具的使用也很简单。我们分别启动一个 server 和 client，对应着监听 7000 端口的服务端程序，以及发起压力测试的客户端程序，目的是为了模拟真实环境下的压力测试：</p><pre><code>./server 7000
./client 127.0.0.1 7000
</code></pre><p>紧接着，client 会向 server 发送请求，检测当前的系统环境能否支持 100 万并发连接。你可以自己去运行一下，看看结果。</p><h3>第二步，检测服务端程序是否正常运行。</h3><p>如果服务端的程序不正常，那么压力测试可能就成了错误日志刷新测试，或者是 404 响应测试。</p><p>所以，测试环境检测的最后一步，也是最重要的一步，就是<strong>跑一遍服务端的单元测试集，或者手动调用几个主要的接口，来保证 wrk 测试的所有接口、返回的内容和 http 响应码都正常，并且在 <code>logs/error.log</code> 中没有出现任何错误级别的信息。</strong></p><h2>发送请求</h2><p>好了，到现在，万事俱备，只欠东风了。让我们开始用 wrk 来做压力测试吧！</p><pre><code>$ wrk -d 30 http://127.0.0.2:9080/hello
Running 30s test @ http://127.0.0.2:9080/hello
  2 threads and 10 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   595.39us  178.51us  22.24ms   90.63%
    Req/Sec     8.33k   642.91     9.46k    59.80%
  499149 requests in 30.10s, 124.22MB read
Requests/sec:  16582.76
Transfer/sec:      4.13MB
</code></pre><p>这里，我并没有指定参数，所以wrk会默认启动 2 个线程和 10 个长连接。其实，你也并不需要把wrk 的线程数和连接数调整得很大，只要能够让目标程序跑满 CPU 就达到要求了。</p><p>但压测的时间一定不能太短，几秒钟的压测是没有意义的，不然很有可能服务端的程序还没加载完热数据，压测就已经结束了。同时，在压测期间，你需要使用 top 或者 htop 这样的监控工具，来确认服务端目标程序是否跑满 CPU。</p><p>从现象上来看，如果 CPU 满载，而且压测停止后，CPU 和内存占用迅速降低，那么恭喜你，这次压测顺利完成。但如果有下面这样的异常，作为服务端开发的你就得特别留意了。</p><ul>
<li>CPU 不能满载。这不会是 wrk 的问题，可能是网络的限制，更可能是你的代码中有阻塞的操作。你可以通过 review 代码来确定，也可以使用 off CPU 火焰图来确定。</li>
<li>CPU 一直满载，即使压测停止仍然如此。这说明在代码中存在热循环，可能是正则表达式引起的，也可能是 LuaJIT 的 bug 引起的，这两点都是我在真实的环境中遇到过的问题。这时，你就需要用 on CPU 火焰图来确定了。</li>
</ul><p>最后再来一起看下 wrk 的统计结果。关于这个结果，我们一般会关注两个值：</p><p>第一个是 QPS，也就是 <code>Requests/sec: 16582.76</code>，这个数据很直接，表示服务端每秒钟处理了多少请求。</p><p>第二个是延时 <code>Latency 595.39us 178.51us 22.24ms 90.63%</code>，这个数据和 QPS 一样重要，它体现了系统的响应速度。比如对于网关的应用来讲，我们就希望能够把延时控制在 1 毫秒以内。</p><p>另外， wrk 还提供了 latency 参数，可以把延时的分布百分比详细地打印出来，比如：</p><pre><code>Latency Distribution
     50%  134.00us
     75%  180.00us
     90%  247.00us
     99%  552.00us
</code></pre><p>不过，wrk 的延时分布数据并不准确，因为它人为地加入了网络和工具的扰动，放大了延时，这一点需要你特别注意。关于wrk Latency Distribution，你可以通过我以前写的<a href="https://mp.weixin.qq.com/s/n8a4wzmf6I8kUc-T47PylA">这篇文章</a>来了解详细内容。</p><h2>写在最后</h2><p>性能测试是个技术活儿，能做对、做好的人不多。希望今天这节课，能让你对性能测试有一个更全面的认识。</p><p>最后给你留一个作业题：wrk 支持自定义 Lua 脚本来做压力测试，那么，你可以根据它的文档，写一段简单的 Lua 脚本吗？这可能会有一些难度，但完成的同时，你一定能更深刻地理解 wrk 暴露接口的用意。</p><p>欢迎留言写下你的答案和思考，也欢迎你把这篇文章分享给更多的人，我们共同进步。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/69/bc/defd41a9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>老王</span>
  </div>
  <div class="_2_QraFYR_0">我怎么记得春哥在 google group 里多次提到 ab 是当前最佳测试工具呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你应该记错了吧？或者是他遇到 wrk 之前的看法</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-31 15:50:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/f0/17/796a3d20.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>言十年</span>
  </div>
  <div class="_2_QraFYR_0">wrk.method=&quot;PUT&quot;<br>function request()<br>   local  num = math.random(1,99999999)<br>   local num1 = math.random(1,99999999)<br>   local num2 = math.random(1,99999999)<br>   local path = &quot;&#47;v3&#47;333333&#47;&quot; .. num .. &quot;dfadnl&quot; ..num1..&quot;sssssdf&quot; .. num2 .. &quot;.jpg&quot;;<br>   headers = {authorization= &quot;xxxxxx&quot;}<br>   local content = load_file(&quot;&#47;home&#47;www&#47;1M.csv&quot;)<br>   return wrk.format(&quot;PUT&quot;, path, headers, content)<br>end<br>local open = io.open<br>local insert = table.insert<br>local concat = table.concat<br><br>function load_file(filename)<br>        local file = open(filename, &#39;rb&#39;)<br>        if not file then<br>                return<br>        end<br>        local f = {}<br>        for line in file:lines(&quot;l&quot;) do<br>                insert(f, line)<br>        end<br>        file:close()<br>        return concat(f, &#39;\n&#39;)<br>end</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-01 13:34:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/aa/41/b52af10b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>土老鳖</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，如果我想调试ngx源码，比如测试内存池，或者ngx自己实现的list，甚至是配置文件解析流程。有什么好的办法能让我做加桩测试吗？或者有其他好的建议吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: NGINX 官方提供了调试的方法：https:&#47;&#47;docs.nginx.com&#47;nginx&#47;admin-guide&#47;monitoring&#47;debugging&#47;</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-06 20:19:08</div>
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
  <div class="_2_QraFYR_0">老师，从redis读取数据和向redis写数据，是阻塞操作吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 使用 lua-resty-redis 库来操作的话，就不是阻塞操作，因为底层使用的是 cosocket</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-31 23:33:19</div>
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
  <div class="_2_QraFYR_0">老师你好，局域网内带宽有限，测不到QPS最大值怎么办？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以用交换机自己组建一个网络，里面只有测试服务器</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-31 16:03:31</div>
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
  <div class="_2_QraFYR_0">老师你好，<br>net.ipv4.ip_conntrack_max = 1020000<br>net.ipv4.netfilter.ip_conntrack_max = 1020000<br>这两个在centos下配置会报错，是什么原因呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 具体什么报错呢？centos 什么版本？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-31 15:58:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/47/c2/e9fa4cf6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Charles</span>
  </div>
  <div class="_2_QraFYR_0">请教温老师，跑wrk的客户端是放在外网上的机器，还是和服务端同一局域网内的机器，哪个更有性能测试意义？谢谢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 最好在局域网，排除网络的干扰因素。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-31 09:12:06</div>
  </div>
</div>
</div>
</li>
</ul>