<audio title="14 _ 答疑（二）：如何用perf工具分析Java程序？" src="https://static001.geekbang.org/resource/audio/7d/ec/7d0510752b0453d81492527dd11a87ec.mp3" controls="controls"></audio> 
<p>你好，我是倪朋飞。</p><p>今天是我们第二期答疑，这期答疑的主题是我们多次用到的perf工具，内容主要包括前面案例中， perf 使用方法的各种疑问。</p><p>perf 在性能分析中非常有效，是我们每个人都需要掌握的核心工具。perf 的使用方法也很丰富，不过不用担心，目前你只要会用 perf record 和 perf report 就够了。而对于 perf 显示的调用栈中的某些内核符号，如果你不理解也没有关系，可以暂时跳过，并不影响我们的分析。</p><p>同样的，为了便于你学习理解，它们并不是严格按照文章顺序排列的，如果你需要回顾内容原文，可以扫描每个问题右下方的二维码查看。</p><h2>问题 1： 使用 perf 工具时，看到的是16进制地址而不是函数名</h2><p><img src="https://static001.geekbang.org/resource/image/94/69/94f67501d5be157a25a26d852b8c2869.png?wh=750*1782" alt=""></p><p>这也是留言比较多的一个问题，在 CentOS 系统中，使用 perf 工具看不到函数名，只能看到一些16进制格式的函数地址。</p><p>其实，只要你观察一下perf界面最下面的那一行，就会发现一个警告信息：</p><pre><code>Failed to open /opt/bitnami/php/lib/php/extensions/opcache.so, continuing without symbols
</code></pre><p>这说明，perf 找不到待分析进程依赖的库。当然，实际上这个案例中有很多依赖库都找不到，只不过，perf工具本身只在最后一行显示警告信息，所以你只能看到这一条警告。</p><p>这个问题，其实也是在分析Docker容器应用时，我们经常碰到的一个问题，因为容器应用依赖的库都在镜像里面。</p><!-- [[[read_end]]] --><p>针对这种情况，我总结了下面<strong>四个解决方法</strong>。</p><p><strong>第一个方法，在容器外面构建相同路径的依赖库</strong>。这种方法从原理上可行，但是我并不推荐，一方面是因为找出这些依赖库比较麻烦，更重要的是，构建这些路径，会污染容器主机的环境。</p><p><strong>第二个方法，在容器内部运行 perf</strong>。不过，这需要容器运行在特权模式下，但实际的应用程序往往只以普通容器的方式运行。所以，容器内部一般没有权限执行 perf 分析。</p><p>比方说，如果你在普通容器内部运行 perf record ，你将会看到下面这个错误提示：</p><pre><code>$ perf_4.9 record -a -g
perf_event_open(..., PERF_FLAG_FD_CLOEXEC) failed with unexpected error 1 (Operation not permitted)
perf_event_open(..., 0) failed unexpectedly with error 1 (Operation not permitted)
</code></pre><p>当然，其实你还可以通过配置 /proc/sys/kernel/perf_event_paranoid （比如改成-1），来允许非特权用户执行 perf 事件分析。</p><p>不过还是那句话，为了安全起见，这种方法我也不推荐。</p><p><strong>第三个方法，指定符号路径为容器文件系统的路径</strong>。比如对于第05讲的应用，你可以执行下面这个命令：</p><pre><code>$ mkdir /tmp/foo
$ PID=$(docker inspect --format {{.State.Pid}} phpfpm)
$ bindfs /proc/$PID/root /tmp/foo
$ perf report --symfs /tmp/foo

# 使用完成后不要忘记解除绑定
$ umount /tmp/foo/
</code></pre><p>不过这里要注意，bindfs 这个工具需要你额外安装。bindfs 的基本功能是实现目录绑定（类似于 mount --bind），这里需要你安装的是 1.13.10 版本（这也是它的最新发布版）。</p><p>如果你安装的是旧版本，你可以到 <a href="https://github.com/mpartel/bindfs">GitHub</a>上面下载源码，然后编译安装。</p><p><strong>第四个方法，在容器外面把分析纪录保存下来，再去容器里查看结果</strong>。这样，库和符号的路径也就都对了。</p><p>比如，你可以这么做。先运行 perf record -g -p &lt; pid&gt;，执行一会儿（比如15秒）后，按Ctrl+C停止。</p><p>然后，把生成的 perf.data 文件，拷贝到容器里面来分析：</p><pre><code>$ docker cp perf.data phpfpm:/tmp 
$ docker exec -i -t phpfpm bash
</code></pre><p>接下来，在容器的 bash 中继续运行下面的命令，安装 perf 并使用 perf report 查看报告：</p><pre><code>$ cd /tmp/ 
$ apt-get update &amp;&amp; apt-get install -y linux-tools linux-perf procps
$ perf_4.9 report
</code></pre><p>不过，这里也有两点需要你注意。</p><p>首先是perf工具的版本问题。在最后一步中，我们运行的工具是容器内部安装的版本 perf_4.9，而不是普通的 perf 命令。这是因为， perf 命令实际上是一个软连接，会跟内核的版本进行匹配，但镜像里安装的perf版本跟虚拟机的内核版本有可能并不一致。</p><p>另外，php-fpm 镜像是基于 Debian 系统的，所以安装 perf 工具的命令，跟 Ubuntu 也并不完全一样。比如， Ubuntu 上的安装方法是下面这样：</p><pre><code>$ apt-get install -y linux-tools-common linux-tools-generic linux-tools-$(uname -r)）
</code></pre><p>而在 php-fpm 容器里，你应该执行下面的命令来安装 perf：</p><pre><code>$ apt-get install -y linux-perf
</code></pre><p>当你按照前面这几种方法操作后，你就可以在容器内部看到 sqrt 的堆栈：</p><p><img src="https://static001.geekbang.org/resource/image/76/af/76f8d0f36210001e750b0a82026dedaf.png?wh=696*459" alt=""></p><p>事实上，抛开我们的案例来说，即使是在非容器化的应用中，你也可能会碰到这个问题。假如你的应用程序在编译时，使用 strip 删除了ELF二进制文件的符号表，那么你同样也只能看到函数的地址。</p><p>现在的磁盘空间，其实已经足够大了。保留这些符号，虽然会导致编译后的文件变大，但对整个磁盘空间来说已经不是什么大问题。所以为了调试的方便，建议你还是把它们保留着。</p><p>顺便提一下，案例中各种工具的安装方法，可以算是我们专栏学习的基本功，这一点希望你能够熟悉并掌握。还是那句话，不会安装先查文档，还是不行就上网搜索或者在文章里留言提问。</p><p>在这里也要表扬一下，很多同学已经把摸索到的方法分享到了留言中。记录并分享，是一个很好的习惯。</p><h2>问题 2：如何用perf工具分析Java程序</h2><p><img src="https://static001.geekbang.org/resource/image/1e/dc/1eb200e2a68da9a00b2ee009f3de94dc.png?wh=750*836" alt=""><img src="https://static001.geekbang.org/resource/image/97/6d/97f609aae409bd9840f606c1d9bc7e6d.jpg?wh=750*1162" alt=""></p><p>这两个问题，其实是上一个 perf 问题的延伸。 像是Java这种通过 JVM 来运行的应用程序，运行堆栈用的都是 JVM 内置的函数和堆栈管理。所以，从系统层面你只能看到JVM的函数堆栈，而不能直接得到Java应用程序的堆栈。</p><p>perf_events 实际上已经支持了 JIT，但还需要一个 /tmp/perf-PID.map文件，来进行符号翻译。当然，开源项目 <a href="https://github.com/jvm-profiling-tools/perf-map-agent">perf-map-agent </a>可以帮你生成这个符号表。</p><p>此外，为了生成全部调用栈，你还需要开启JDK的选项 -XX:+PreserveFramePointer。因为这里涉及到大量的 Java 知识，我就不再详细展开了。如果你的应用刚好基于 Java ，那么你可以参考 Netflix 的技术博客 <a href="https://medium.com/netflix-techblog/java-in-flames-e763b3d32166">Java in Flames</a> ，来查看详细的使用步骤。</p><p>说到这里，我也想强调一个问题，那就是学习性能优化时，不要一开始就把自己限定在具体的某个编程语言或者性能工具中，纠结于语言或工具的细节出不来。</p><p>掌握整体的分析思路，才是我们首先要做的。因为，性能优化的原理和思路，在任何编程语言中都是相通的。</p><h2>问题 3：为什么 perf 的报告中，很多符号都不显示调用栈</h2><p><img src="https://static001.geekbang.org/resource/image/49/05/4902af1ff4d710aa7cb150f44e3e3c05.png?wh=900*1092" alt=""></p><p>perf report 是一个可视化展示 perf.data 的工具。在第 08 讲的案例中，我直接给出了最终结果，并没有详细介绍它的参数。估计很多同学的机器在运行时，都碰到了跟路过同学一样的问题，看到的是下面这个界面。</p><p><img src="https://static001.geekbang.org/resource/image/33/e9/335d41ccf24d93aafe0e4d511218b6e9.png?wh=751*426" alt=""></p><p>这个界面可以清楚看到，perf report 的输出中，只有 swapper 显示了调用栈，其他所有符号都不能查看堆栈情况，包括我们案例中的 app 应用。</p><p>这种情况我们以前也遇到过，当你发现性能工具的输出无法理解时，应该怎么办呢？当然还是查工具的手册。比如，你可以执行 man perf-report 命令，找到 -g 参数的说明：</p><pre><code>-g, --call-graph=&lt;print_type,threshold[,print_limit],order,sort_key[,branch],value&gt; 
           Display call chains using type, min percent threshold, print limit, call order, sort key, optional branch and value. Note that 
           ordering is not fixed so any parameter can be given in an arbitrary order. One exception is the print_limit which should be 
           preceded by threshold. 

               print_type can be either: 
               - flat: single column, linear exposure of call chains. 
               - graph: use a graph tree, displaying absolute overhead rates. (default) 
               - fractal: like graph, but displays relative rates. Each branch of 
                        the tree is considered as a new profiled object. 
               - folded: call chains are displayed in a line, separated by semicolons 
               - none: disable call chain display. 

               threshold is a percentage value which specifies a minimum percent to be 
               included in the output call graph.  Default is 0.5 (%). 

               print_limit is only applied when stdio interface is used.  It's to limit 
               number of call graph entries in a single hist entry.  Note that it needs 
               to be given after threshold (but not necessarily consecutive). 
               Default is 0 (unlimited). 

               order can be either: 
               - callee: callee based call graph. 
               - caller: inverted caller based call graph. 
               Default is 'caller' when --children is used, otherwise 'callee'. 

               sort_key can be: 
               - function: compare on functions (default) 
               - address: compare on individual code addresses 
               - srcline: compare on source filename and line number 

               branch can be: 
               - branch: include last branch information in callgraph when available. 
                         Usually more convenient to use --branch-history for this. 

               value can be: 
               - percent: diplay overhead percent (default) 
               - period: display event period 
               - count: display event count
</code></pre><p>通过这个说明可以看到，-g 选项等同于 --call-graph，它的参数是后面那些被逗号隔开的选项，意思分别是输出类型、最小阈值、输出限制、排序方法、排序关键词、分支以及值的类型。</p><p>我们可以看到，这里默认的参数是 graph,0.5,caller,function,percent，具体含义文档中都有详细讲解，这里我就不再重复了。</p><p>现在再回过头来看我们的问题，堆栈显示不全，相关的参数当然就是最小阈值 threshold。通过手册中对threshold的说明，我们知道，当一个事件发生比例高于这个阈值时，它的调用栈才会显示出来。</p><p>threshold 的默认值为 0.5%，也就是说，事件比例超过 0.5%时，调用栈才能被显示。再观察我们案例应用 app 的事件比例，只有 0.34%，低于 0.5%，所以看不到 app 的调用栈就很正常了。</p><p>这种情况下，你只需要给 perf report 设置一个小于 0.34% 的阈值，就可以显示我们想看到的调用图了。比如执行下面的命令：</p><pre><code>$ perf report -g graph,0.3
</code></pre><p>你就可以得到下面这个新的输出界面，展开 app 后，就可以看到它的调用栈了。</p><p><img src="https://static001.geekbang.org/resource/image/b3/93/b34f95617d13088671f4d9c2b9134693.png?wh=747*536" alt=""></p><h2>问题 4：怎么理解 perf report 报告</h2><p><img src="https://static001.geekbang.org/resource/image/42/d8/42bf6b82da73656d6c3dad20074f57d8.png?wh=900*1050" alt=""><img src="https://static001.geekbang.org/resource/image/b9/8b/b90140f7d41790f74982d431f7e0238b.png?wh=750*1151" alt=""></p><p>看到这里，我估计你也曾嘀咕过，为啥不一上来就用 perf 工具解决，还要执行那么多其他工具呢？ 这个问题其实就给出了很好的解释。</p><p>在问题4的perf report 界面中，你也一定注意到了， swapper 高达 99% 的比例。直觉来说，我们应该直接观察它才对，为什么没那么做呢？</p><p>其实，当你清楚了 swapper 的原理后，就很容易理解我们为什么可以忽略它了。</p><p>看到swapper，你可能首先想到的是SWAP分区。实际上， swapper 跟 SWAP 没有任何关系，它只在系统初始化时创建 init 进程，之后，它就成了一个最低优先级的空闲任务。也就是说，当 CPU 上没有其他任务运行时，就会执行swapper 。所以，你可以称它为“空闲任务”。</p><p>回到我们的问题，在 perf report 的界面中，展开它的调用栈，你会看到， swapper 时钟事件都耗费在了 do_idle 上，也就是在执行空闲任务。</p><p><img src="https://static001.geekbang.org/resource/image/12/bd/121dcefacba1b554accd0a90ef349fbd.png?wh=708*479" alt=""></p><p>所以，分析案例时，我们直接忽略了前面这个 99% 的符号，转而分析后面只有 0.3% 的 app。其实从这里你也能理解，为什么我们一开始不先用 perf 分析。</p><p>因为在多任务系统中，次数多的事件，不一定就是性能瓶颈。所以，只观察到一个大数值，并不能说明什么问题。具体有没有瓶颈，还需要你观测多个方面的多个指标，来交叉验证。这也是我在套路篇中不断强调的一点。</p><p>另外，关于 Children 和 Self 的含义，手册里其实有详细说明，还很友好地举了一个例子，来说明它们的百分比的计算方法。简单来说，</p><ul>
<li>
<p>Self 是最后一列的符号（可以理解为函数）本身所占比例；</p>
</li>
<li>
<p>Children 是这个符号调用的其他符号（可以理解为子函数，包括直接和间接调用）占用的比例之和。</p>
</li>
</ul><p>正如同学留言问到的，很多性能工具确实会对系统性能有一定影响。就拿 perf 来说，它需要在内核中跟踪内核栈的各种事件，那么不可避免就会带来一定的性能损失。这一点，虽然对大部分应用来说，没有太大影响，但对特定的某些应用（比如那些对时钟周期特别敏感的应用），可能就是灾难了。</p><p>所以，使用性能工具时，确实应该考虑工具本身对系统性能的影响。而这种情况，就需要你了解这些工具的原理。比如，</p><ul>
<li>
<p>perf 这种动态追踪工具，会给系统带来一定的性能损失。</p>
</li>
<li>
<p>vmstat、pidstat 这些直接读取 proc 文件系统来获取指标的工具，不会带来性能损失。</p>
</li>
</ul><h2>问题 5：性能优化书籍和参考资料推荐</h2><p><img src="https://static001.geekbang.org/resource/image/a0/ba/a0be73a43e756da48bdbdd01d71598ba.png?wh=750*803" alt=""></p><p>我很高兴看到留言有这么高的学习热情，其实好多文章后面都有大量留言，希望我能推荐书籍和学习资料。这一点也是我乐意看到的。专栏学习一定不是你性能优化之旅的全部，能够带你入门、帮你解决实际问题、甚至是激发你的学习热情，已经让我非常开心。</p><p>在 <a href="https://time.geekbang.org/column/article/69346">如何学习Linux性能优化</a> 的文章中，我曾经介绍过 Brendan Gregg，他是当之无愧的性能优化大师，你在各种 Linux 性能优化的文章中，基本都能看到他的那张性能工具图谱。</p><p>所以，关于性能优化的书籍，我最喜欢的其实正是他写的那本 《Systems Performance: Enterprise and the Cloud》。这本书也出了中文版，名字是《性能之巅：洞悉系统、企业与云计算》。</p><p>从出版时间来看，这本书确实算一本老书了，英文版的是 2013 年出版的。但是经典之所以成为经典，正是因为不会过时。这本书里的性能分析思路以及很多的性能工具，到今天依然适用。</p><p>另外，我也推荐你去关注他的个人网站 <a href="http://www.brendangregg.com/">http://www.brendangregg.com/</a>，特别是 <a href="http://www.brendangregg.com/linuxperf.html">Linux Performance </a>这个页面，包含了很多 Linux 性能优化的资料，比如：</p><ul>
<li>
<p>Linux性能工具图谱 ；</p>
</li>
<li>
<p>性能分析参考资料；</p>
</li>
<li>
<p>性能优化的演讲视频 。</p>
</li>
</ul><p>不过，这里很多内容会涉及到大量的内核知识，对初学者来说并不友好。但是，如果你想成为高手，辛苦和坚持都是不可避免的。所以，希望你在查看这些资料时，不要一遇到不懂的就打退堂鼓。任何东西的第一遍学习有不懂的地方很正常，忍住恐惧别放弃，继续往后走，前面很多问题可能会一并解决掉，再看第二遍、第三遍就更轻松了。</p><p>还是那句话，抓住主线不动摇，先从最基本的原理开始，掌握性能分析的思路，然后再逐步深入，探究细节，不要试图一口吃成个大胖子。</p><p>最后，欢迎继续在留言区写下你的疑问，我会持续不断地解答。我的目的仍然不变，希望可以和你一起，把文章的知识变成你的能力，我们不仅仅在实战中演练，也要在交流中进步。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/64/05/6989dce6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我来也</span>
  </div>
  <div class="_2_QraFYR_0">[D14打卡]<br>很赞同老师的观点:<br>&quot;任何东西的第一遍学习有不懂的地方很正常，忍住恐惧别放弃，继续往后走，前面很多问题可能会一并解决掉 ，再看第二遍、第三遍就更轻松了。&quot;<br>我好像很早就这样实践了: 第一遍不管看不看得懂,先尽量细看. 再特意过一段时间回头重新学一遍,除了能掌握更多的东西,还能体会到&quot;温故而知新&quot;的感觉.<br>----------------------------<br>现在就是&quot;师傅领进门,修行看个人.&quot;<br>&quot;先从最基本的原理开始，掌握性能分析的思路，然后再逐步深入，探究细节&quot;<br>----------------------------<br>自从学了专栏,越来越觉得自己的&lt;英语&gt;该好好补补了,深感能力不足啊.<br>那么多好的资源,一手资料几乎都是英文的,看不懂真是可惜了!</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 英语也是基本要求，多读多看就熟悉了😊</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-21 11:55:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://wx.qlogo.cn/mmopen/vi_32/PiajxSqBRaEKQMM4m7NHuicr55aRiblTSEWIYe0QqbpyHweaoAbG7j2v7UUElqqeP3Ihrm3UfDPDRb1Hv8LvPwXqA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ninuxer</span>
  </div>
  <div class="_2_QraFYR_0">打卡day15<br>perf report中关于swapper的内容，后面我也去查了，才发现是自己理解有误，感谢老师指出，这里的swapper不是内存概念的swap，而是cpu空闲时执行的一个默认调用<br>要啃啃《性能之巅：洞悉系统、企业与云计算》了，作者博客http:&#47;&#47;www.brendangregg.com&#47;<br>请教老师，理解内核这块，有合适的书推荐么？我查了下，看了下目录，感觉《Linux内核设计与实现》可能比较适合，其他的如《Linux内核情景分析》，《深入理解Linux内核》怎么样？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 《深入Linux内核架构》挺不错，不过是本大块头，啃下应该要花不少时间</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-21 08:44:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/74/ad/771d2646.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>赵奇彬</span>
  </div>
  <div class="_2_QraFYR_0">perf生成的数据可以用火焰图显示更加直观</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-06 23:34:56</div>
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
  <div class="_2_QraFYR_0">任何东西的第一遍学习有不懂的地方很正常，忍住恐惧别放弃，继续往后走，前面很多问题可能会一并解决掉，再看第二遍、第三遍就更轻松了。<br>------------------------------<br>深感共鸣，以前学习总是贪大求全，总以为一遍能看懂的决不花两遍，后来发现真实能理解起码需要2遍以上，看来基本上大多数人都是这个样子。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-30 13:53:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTK0K9AM6xxDzVV6pF66jyus5NuuxZzT9icad8AQDMKibwUOy3UnoZIZdyKIKd9sA06rgFnIWwiakSeOQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>斑马Z</span>
  </div>
  <div class="_2_QraFYR_0">关于本章中遇到的工具在centos系统下的安装。<br>yum install sysstat pstree psmisc<br>以及hping3的安装参考<br>https:&#47;&#47;www.cnblogs.com&#47;shuter&#47;p&#47;11430876.html<br>如果yum的数据db损坏，可能需要clean 和 makecache一下再进行<br>docker镜像中 apt-get update &amp;&amp; apt-get install -y linux-perf linux-tools procps<br><br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-27 09:47:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/83/fb/621adceb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>linker</span>
  </div>
  <div class="_2_QraFYR_0">准备把专栏做成思维导图，方面查看。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-20 23:57:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/04/ae/0118c132.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>辉晖</span>
  </div>
  <div class="_2_QraFYR_0">对于问题1，使用方法4还是看不到函数名，只能看到一些 16 进制格式的函数地址<br><br>在载入perf.data过程不断出现提示：Failed to open ***, continuing without symbols<br>载入完成后，perf 界面最下面的那一行警告信息是：Cannot load tips.txt file, please install perf!</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Failed to open ***, continuing without symbols 说明还是无法找到相应的符号表，根据警告提示逐个安装应该就可以了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-27 11:12:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/97/61/34a0da09.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Griffin</span>
  </div>
  <div class="_2_QraFYR_0">终于赶上了，倪老师能不能讲讲网络问题应该怎么排查呀，最近老是被docker，docker swarm的container寻址困扰。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不要急，我们有网络模块的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-23 20:27:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLHS9BjwOgkqV1NSmNRFxUC6KU0DibS75f00GhMWx4s5OYLryibaNDoJ1tZAFRaHJ7jSZXA4pNumraQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Lake</span>
  </div>
  <div class="_2_QraFYR_0">第二遍了，把第一遍看没做的实验跟着做，顺便开始整理笔记了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-11 11:03:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/5a/b7/f053eda7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>康熙</span>
  </div>
  <div class="_2_QraFYR_0">《Systems Performance: Enterprise and the Cloud》这本书刚出了第二版，新鲜出炉的：http:&#47;&#47;www.brendangregg.com&#47;systems-performance-2nd-edition-book.html</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-27 20:44:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLO7rmfnq3iaicssF1Vl2lJicYl4C1Lb5q5vIJQwXQ2mbiaPpceeRibwoMAcDcLm5zFk1ZryicyRCVpNibfg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>郡主秋</span>
  </div>
  <div class="_2_QraFYR_0">老师，我的在centos7上用perf分析宿主机上的应用也是显示16进制地址，没有函数名 ，这种怎么处理呢<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: perf 界面应该有缺少符号的提示吧？按照提示安装缺少的调试信息</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-18 12:35:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/4c/c8/bed1e08a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>辣椒</span>
  </div>
  <div class="_2_QraFYR_0">老师，我把perf.data拷贝进容器，然后在容器中按照提示安装了perf, 再执行perf_4.9 report时报以下信息：<br>Kernel address maps (&#47;proc&#47;{kallsyms,modules}) were restricted.   <br>Check &#47;proc&#47;sys&#47;kernel&#47;kptr_restrict before running &#39;perf record&#39;<br>As no suitable kallsyms nor vmlinux was found, kernel samples     <br>can&#39;t be resolved.                                             <br>Samples in kernel modules can&#39;t be resolved as well.              <br>  xPress any key...  <br><br>我本身的机器是centos7.2. 请老师提示一下解决的思路，谢谢！<br><br>                                                </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: echo 0&gt; &#47;proc&#47;sys&#47;kernel&#47;kptr_restrict<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-07 16:47:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>无名老卒</span>
  </div>
  <div class="_2_QraFYR_0">全部刷完了，把各个CPU容易出现问题的情况基本上都写清楚了，而且都有详细的测试用例，除了软中断那个需要SYN攻击之外，其他的测试用例都一五一十的都做过了，受益良多。我有以下疑问，希望老师可以解答下：<br>1、软中断老师是用SYN攻击的方式来讲解这部分的实例的，那还有没有其他典型的软中断的案例呢？<br>2、硬中断的实例老师没有讲，可以补一篇吗？<br>3、老师所讲的实例，都是单一模式的，在实际的生产环境下，情况要复杂很多，老师能再讲一下印象最深刻的实际情况吗？<br><br>另外，我有一个小建议，老师的案例都是用docker来搭建环境的，在一个用例要多次下载不同的image，但其实这些image只是里面的测试用例变了，所以可以先下载file文件，使用docker -v挂载的方式进行测试。这样可以大大减少下载镜像的时间。如下，就是使用docker run -v &#47;usr&#47;local&#47;src&#47;app-fix2:&#47;app --privileged --name app-fix -d feisky&#47;app:iowait 来运行的镜像。<br>```<br>[root@linjx src]# docker ps -a<br>CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                        PORTS               NAMES<br>0ecb229f87e4        feisky&#47;app:iowait   &quot;&#47;app&quot;              3 minutes ago       Up 3 minutes                                      app-fix<br>a251996e0d60        feisky&#47;app:iowait   &quot;&#47;app&quot;              5 hours ago         Exited (137) 20 minutes ago                       app<br>```</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. 网络后面还会讲<br>2. 硬中断的原理其实差不太多，所以没有单独拎出来<br>3. 现在单个模块的案例的确不太复杂，这也是为了让大家都能更直观理解这些问题<br><br>关于image下载，其实可以使用国内的镜像加速，网络搜索找到。并且基础镜像一样的话，也只需要下载新的layer即可。<br><br>最后，你的容器推退出了，请参考github设置启动参数解决。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-23 10:26:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/9b/a8/6a391c66.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Leon📷</span>
  </div>
  <div class="_2_QraFYR_0">老师，有个问题，如果iowait过高，导致系统卡死，ssh几乎操作不动，怎么快速找出进程杀掉，iotop敲了20分钟没响应</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这种情况没法SSH操作，只能靠监控系统了，一定要把这些性能指标监控起来</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-22 10:26:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/52/5c/d4a9accb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>仄言</span>
  </div>
  <div class="_2_QraFYR_0">php容器里面运行  perf_4.9  报错<br>┌─Warning:─────────────────────────────────────────────────────────────────────────┐<br>                                   │Found 1 unknown events!                                                           │<br>                                   │                            ┌─Error:─────────────────┐                            │<br>                                   │Is this an older tool proces│failed to process sample│rated by a more recent tool?│<br>                                   │                            │                        │                            │<br>                                   │If that is not the case, con│                        │ernel@vger.kernel.org.      │<br>                                   │                            │Press any key...        │                            │<br>                                   │                            └────────────────────────┘                            │<br>                                   │                                                                                  │<br>                                   │Press any key...                                                                  │<br>                                   └──────────────────────────────────────────────────────────────────────────────────┘<br><br><br><br><br><br><br><br>0x2290 [0x50]: failed to process type: 17<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-16 19:36:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/28/91/1830fb64.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>天下第一好男生</span>
  </div>
  <div class="_2_QraFYR_0">您好，老师，请教个问题。通过proxmox装的centos系统，执行类似w ss top uptime之类的命令会夯住，其他的命令没有问题。通过strace发现一直在打开&#47;etc&#47;hosts文件，不知道是哪里的问题，该如何排查。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-27 12:25:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/53/0d/c4643e0e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>拉鲁</span>
  </div>
  <div class="_2_QraFYR_0">perf 还是二进制</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-16 17:23:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTK009KMmXKFVSEdjicpDD4ick3NZpM3JdIUibWGB03lG6yicibad0tGmQD7E3DpZ0sVRenxWNfd7iaPdp7g/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小小菜鸟</span>
  </div>
  <div class="_2_QraFYR_0">发现英文好真好。。。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-04 17:58:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/48/c7/2f051793.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Anti-Time</span>
  </div>
  <div class="_2_QraFYR_0">想请教一下：<br>关于运行状态为：S 的进程， 使用perf看不到信息是什么问题呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-09 20:52:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/8b/80/8702bd5f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>evan</span>
  </div>
  <div class="_2_QraFYR_0">感谢老师 继续加油!</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-25 08:22:53</div>
  </div>
</div>
</div>
</li>
</ul>