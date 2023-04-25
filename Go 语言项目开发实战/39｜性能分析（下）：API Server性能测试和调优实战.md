<audio title="39｜性能分析（下）：API Server性能测试和调优实战" src="https://static001.geekbang.org/resource/audio/f0/cc/f02b0b4bd4b9115476019a84f359fccc.mp3" controls="controls"></audio> 
<p>你好，我是孔令飞。</p><p>上一讲，我们学习了如何分析Go代码的性能。掌握了性能分析的基本知识之后，这一讲，我们再来看下如何分析API接口的性能。</p><p>在API上线之前，我们需要知道API的性能，以便知道API服务器所能承载的最大请求量、性能瓶颈，再根据业务对性能的要求，来对API进行性能调优或者扩缩容。通过这些，可以使API稳定地对外提供服务，并且让请求在合理的时间内返回。这一讲，我就介绍如何用wrk工具来测试API Server接口的性能，并给出分析方法和结果。</p><h2>API性能测试指标</h2><p>API性能测试，往大了说其实包括API框架的性能和指定API的性能。不过，因为指定API的性能跟该API具体的实现（比如有无数据库连接，有无复杂的逻辑处理等）有关，我认为脱离了具体实现来探讨单个API的性能是毫无意义的，所以这一讲只探讨API框架的性能。</p><p>用来衡量API性能的指标主要有3个：</p><ul>
<li>并发数（Concurrent）：并发数是指某个时间范围内，同时在使用系统的用户个数。广义上的并发数是指同时使用系统的用户个数，这些用户可能调用不同的API；严格意义上的并发数是指同时请求同一个API的用户个数。这一讲我们讨论的并发数是严格意义上的并发数。</li>
<li>每秒查询数（QPS）：每秒查询数QPS是对一个特定的查询服务器在规定时间内所处理流量多少的衡量标准。QPS = 并发数 / 平均请求响应时间。</li>
<li>请求响应时间（TTLB）：请求响应时间指的是从客户端发出请求到得到响应的整个时间。这个过程从客户端发起的一个请求开始，到客户端收到服务器端的响应结束。在一些工具中，请求响应时间通常会被称为TTLB（Time to last byte，意思是从发送一个请求开始，到客户端收到最后一个字节的响应为止所消费的时间）。请求响应时间的单位一般为“秒”或“毫秒”。</li>
</ul><!-- [[[read_end]]] --><p>这三个指标中，衡量API性能的最主要指标是QPS，但是在说明QPS时，需要指明是多少并发数下的QPS，否则毫无意义，因为不同并发数下的QPS是不同的。举个例子，单用户100 QPS和100用户100 QPS是两个不同的概念，前者说明API可以在一秒内串行执行100个请求，而后者说明在并发数为100的情况下，API可以在一秒内处理100个请求。当QPS相同时，并发数越大，说明API性能越好，并发处理能力越强。</p><p>在并发数设置过大时，API同时要处理很多请求，会频繁切换上下文，而真正用于处理请求的时间变少，反而使得QPS会降低。并发数设置过大时，请求响应时间也会变长。API会有一个合适的并发数，在该并发数下，API的QPS可以达到最大，但该并发数不一定是最佳并发数，还要参考该并发数下的平均请求响应时间。</p><p>此外，在有些API接口中，也会测试API接口的TPS（Transactions Per Second，每秒事务数）。一个事务是指客户端向服务器发送请求，然后服务器做出反应的过程。客户端在发送请求时开始计时，收到服务器响应后结束计时，以此来计算使用的时间和完成的事务个数。</p><p>那么，TPS和QPS有什么区别呢？如果是对一个查询接口（单场景）压测，且这个接口内部不会再去请求其他接口，那么TPS=QPS，否则，TPS≠QPS。如果是对多个接口（混合场景）压测，假设N个接口都是查询接口，且这个接口内部不会再去请求其他接口，QPS=N*TPS。</p><h2>API性能测试方法</h2><p>Linux下有很多Web性能测试工具，常用的有Jmeter、AB、Webbench和wrk。每个工具都有自己的特点，IAM项目使用wrk来对API进行性能测试。wrk非常简单，安装方便，测试结果也相对专业，并且可以支持Lua脚本来创建更复杂的测试场景。下面，我来介绍下wrk的安装方法和使用方法。</p><h3>wrk安装方法</h3><p>wrk的安装很简单，一共可分为两步。</p><p>第一步，Clone wrk repo：</p><pre><code class="language-bash">$ git clone https://github.com/wg/wrk
</code></pre><p>第二步，编译并安装：</p><pre><code class="language-bash">$ cd wrk
$ make
$ sudo cp ./wrk /usr/bin
</code></pre><h3>wrk使用简介</h3><p>这里我们来看下wrk的使用方法。wrk使用起来不复杂，执行<code>wrk --help</code>可以看到wrk的所有运行参数：</p><pre><code class="language-bash">$ wrk --help
Usage: wrk &lt;options&gt; &lt;url&gt;
  Options:
    -c, --connections &lt;N&gt;  Connections to keep open
    -d, --duration    &lt;T&gt;  Duration of test
    -t, --threads     &lt;N&gt;  Number of threads to use

    -s, --script      &lt;S&gt;  Load Lua script file
    -H, --header      &lt;H&gt;  Add header to request
        --latency          Print latency statistics
        --timeout     &lt;T&gt;  Socket/request timeout
    -v, --version          Print version details

  Numeric arguments may include a SI unit (1k, 1M, 1G)
  Time arguments may include a time unit (2s, 2m, 2h)
</code></pre><p>常用的参数有下面这些：</p><ul>
<li>-t，线程数（线程数不要太多，是核数的2到4倍就行，多了反而会因为线程切换过多造成效率降低）。</li>
<li>-c，并发数。</li>
<li>-d，测试的持续时间，默认为10s。</li>
<li>-T，请求超时时间。</li>
<li>-H，指定请求的HTTP Header，有些API需要传入一些Header，可通过wrk的-H参数来传入。</li>
<li>–latency，打印响应时间分布。</li>
<li>-s，指定Lua脚本，Lua脚本可以实现更复杂的请求。</li>
</ul><p>然后，我们来看一个wrk的测试结果，并对结果进行解析。</p><p>一个简单的测试如下（确保iam-apiserver已经启动，并且开启了健康检查）：</p><pre><code class="language-bash">$ wrk -t144 -c30000 -d30s -T30s --latency http://10.0.4.57:8080/healthz
Running 30s test @ http://10.0.4.57:8080/healthz
  144 threads and 30000 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   508.77ms  604.01ms   9.27s    81.59%
    Req/Sec   772.48      0.94k   10.45k    86.82%
  Latency Distribution
     50%  413.35ms
     75%  948.99ms
     90%    1.33s
     99%    2.44s
  2276265 requests in 30.10s, 412.45MB read
  Socket errors: connect 1754, read 40, write 0, timeout 0
Requests/sec:  75613.16
Transfer/sec:     13.70MB
</code></pre><p>下面是对测试结果的解析。</p><ul>
<li>144 threads and 30000 connections：用144个线程模拟20000个连接，分别对应-t和-c参数。</li>
<li>Thread Stats是线程统计，包括Latency和Req/Sec。
<ul>
<li>Latency：响应时间，有平均值、标准偏差、最大值、正负一个标准差占比。</li>
<li>Req/Sec：每个线程每秒完成的请求数, 同样有平均值、标准偏差、最大值、正负一个标准差占比。</li>
</ul>
</li>
<li>Latency Distribution是响应时间分布。
<ul>
<li>50%：50%的响应时间为413.35ms。</li>
<li>75%：75%的响应时间为948.99ms。</li>
<li>90%：90%的响应时间为1.33s。</li>
<li>99%：99%的响应时间为2.44s。</li>
</ul>
</li>
<li>2276265 requests in 30.10s, 412.45MB read：30.10s完成的总请求数（2276265）和数据读取量（412.45MB）。</li>
<li>Socket errors: connect 1754, read 40, write 0, timeout 0：错误统计，会统计connect连接失败请求个数（1754）、读失败请求个数、写失败请求个数、超时请求个数。</li>
<li>Requests/sec：QPS。</li>
<li>Transfer/sec：平均每秒读取13.70MB数据（吞吐量）。</li>
</ul><h2>API Server性能测试实践</h2><p>接下来，我们就来测试下API Server的性能。影响API Server性能的因素有很多，除了iam-apiserver自身的原因之外，服务器的硬件和配置、测试方法、网络环境等都会影响。为了方便你对照性能测试结果，我给出了我的测试环境配置，你可以参考下。</p><ul>
<li>客户端硬件配置：1核4G。</li>
<li>客户端软件配置：干净的<code>CentOS&nbsp;Linux&nbsp;release&nbsp;8.2.2004&nbsp;(Core)</code>。</li>
<li>服务端硬件配置：2核8G。</li>
<li>服务端软件配置：干净的<code>CentOS&nbsp;Linux&nbsp;release&nbsp;8.2.2004&nbsp;(Core)</code>。</li>
<li>测试网络环境：腾讯云VPC内访问，除了性能测试程序外，没有其他资源消耗型业务程序。</li>
</ul><p>测试架构如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/7d/c4/7df51487bc7b761d79247a5d547745c4.jpg?wh=2248x575" alt=""></p><h3>性能测试脚本介绍</h3><p>在做API Server的性能测试时，需要先执行wrk，生成性能测试数据。为了能够更直观地查看性能数据，我们还需要以图表的方式展示这些性能数据。这一讲，我使用 <code>gnuplot</code> 工具来自动化地绘制这些性能图，为此我们需要确保Linux服务器已经安装了 <code>gnuplot</code> 工具。你可以通过以下方式安装：</p><pre><code class="language-bash">$ sudo yum -y install gnuplot
</code></pre><p>在这一讲的测试中，我会绘制下面这两张图，通过它们来观测和分析API Server的性能。</p><ul>
<li>QPS &amp; TTLB图：<code>X</code>轴为并发数（Concurrent），<code>Y</code>轴为每秒查询数（QPS）和请求响应时间（TTLB）。</li>
<li>成功率图：<code>X</code>轴为并发数（Concurrent），<code>Y</code>轴为请求成功率。</li>
</ul><p>为了方便你测试API接口性能，我将性能测试和绘图逻辑封装在<a href="https://github.com/marmotedu/iam/blob/v1.0.8/scripts/wrktest.sh">scripts/wrktest.sh</a>脚本中，你可以在iam源码根目录下执行如下命令，生成性能测试数据和性能图表：</p><pre><code class="language-bash">$ scripts/wrktest.sh http://10.0.4.57:8080/healthz
</code></pre><p>上面的命令会执行性能测试，记录性能测试数据，并根据这些性能测试数据绘制出QPS和成功率图。</p><p>接下来，我再来介绍下wrktest.sh性能测试脚本，并给出一个使用示例。</p><p>wrktest.sh性能测试脚本，用来测试API Server的性能，记录测试的性能数据，并根据性能数据使用gnuplot绘制性能图。</p><p>wrktest.sh也可以对比前后两次的性能测试结果，并将对比结果通过图表展示出来。wrktest.sh会根据CPU的核数自动计算出适合的wrk启动线程数（<code>-t</code>）：<code>CPU核数 * 3</code>。</p><p>wrktest.sh默认会测试多个并发下的API性能，默认测试的并发数为<code>200 500 1000 3000 5000 10000 15000 20000 25000 50000</code>。你需要根据自己的服务器配置选择测试的最大并发数，我因为服务器配置不高（主要是<code>8G</code>内存在高并发下，很容易就耗尽），最大并发数选择了<code>50000</code>。如果你的服务器配置够高，可以再依次尝试下测试 <code>100000</code> 、<code>200000</code> 、<code>500000</code> 、<code>1000000</code> 并发下的API性能。</p><p>wrktest.sh的使用方法如下：</p><pre><code class="language-bash">$ scripts/wrktest.sh -h

Usage: scripts/wrktest.sh [OPTION] [diff] URL
Performance automation test script.

  URL                    HTTP request url, like: http://10.0.4.57:8080/healthz
  diff                   Compare two performance test results

OPTIONS:
  -h                     Usage information
  -n                     Performance test task name, default: apiserver
  -d                     Directory used to store performance data and gnuplot graphic, default: _output/wrk

Reprot bugs to &lt;colin404@foxmail.com&gt;.
</code></pre><p>wrktest.sh提供的命令行参数介绍如下。</p><ul>
<li>URL：需要测试的API接口。</li>
<li>diff：如果比较两次测试的结果，需要执行wrktest.sh diff <data1> <data2>。</data2></data1></li>
<li>-n：本次测试的任务名，wrktest.sh会根据任务名命名生成的文件。</li>
<li>-d：输出文件存放目录。</li>
<li>-h：打印帮助信息。</li>
</ul><p>下面，我来展示一个wrktest.sh使用示例。</p><p>wrktest.sh的主要功能有两个，分别是运行性能测试并获取结果和对比性能测试结果。下面我就分别介绍下它们的具体使用方法。</p><ol>
<li>运行性能测试并获取结果</li>
</ol><p>执行如下命令：</p><pre><code class="language-bash">$ scripts/wrktest.sh http://10.0.4.57:8080/healthz
Running wrk command: wrk -t3 -d300s -T30s --latency -c 200 http://10.0.4.57:8080/healthz
Running wrk command: wrk -t3 -d300s -T30s --latency -c 500 http://10.0.4.57:8080/healthz
Running wrk command: wrk -t3 -d300s -T30s --latency -c 1000 http://10.0.4.57:8080/healthz
Running wrk command: wrk -t3 -d300s -T30s --latency -c 3000 http://10.0.4.57:8080/healthz
Running wrk command: wrk -t3 -d300s -T30s --latency -c 5000 http://10.0.4.57:8080/healthz
Running wrk command: wrk -t3 -d300s -T30s --latency -c 10000 http://10.0.4.57:8080/healthz
Running wrk command: wrk -t3 -d300s -T30s --latency -c 15000 http://10.0.4.57:8080/healthz
Running wrk command: wrk -t3 -d300s -T30s --latency -c 20000 http://10.0.4.57:8080/healthz
Running wrk command: wrk -t3 -d300s -T30s --latency -c 25000 http://10.0.4.57:8080/healthz
Running wrk command: wrk -t3 -d300s -T30s --latency -c 50000 http://10.0.4.57:8080/healthz

Now plot according to /home/colin/_output/wrk/apiserver.dat
QPS graphic file is: /home/colin/_output/wrk/apiserver_qps_ttlb.png
Success rate graphic file is: /home/colin/_output/wrk/apiserver_successrate.pngz
</code></pre><p>上面的命令默认会在<code>_output/wrk/</code>目录下生成3个文件：</p><ul>
<li>apiserver.dat，wrk性能测试结果，每列含义分别为并发数、QPS 平均响应时间、成功率。</li>
<li>apiserver_qps_ttlb.png，QPS&amp;TTLB图。</li>
<li>apiserver_successrate.png，成功率图。</li>
</ul><p>这里要注意，请求URL中的IP地址应该是腾讯云VPC内网地址，因为通过内网访问，不仅网络延时最低，而且还最安全，所以真实的业务通常都是内网访问的。</p><ol start="2">
<li>对比性能测试结果</li>
</ol><p>假设我们还有另外一次API性能测试，测试数据保存在 <code>_output/wrk/http.dat</code> 文件中。</p><p>执行如下命令，对比两次测试结果：</p><pre><code class="language-bash">$ scripts/wrktest.sh diff _output/wrk/apiserver.dat _output/wrk/http.dat
</code></pre><p><code>apiserver.dat</code>和<code>http.dat</code>是两个需要对比的Wrk性能数据文件。上述命令默认会在<code>_output/wrk</code>目录下生成下面这两个文件：</p><ul>
<li>apiserver_http.qps.ttlb.diff.png，QPS &amp; TTLB对比图。</li>
<li>apiserver_http.success_rate.diff.png，成功率对比图。</li>
</ul><h3>关闭Debug配置选项</h3><p>在测试之前，我们需要关闭一些Debug选项，以免影响性能测试。</p><p>执行下面这两步操作，修改iam-apiserver的配置文件：</p><ul>
<li>将<code>server.mode</code>设置为release，<code>server.middlewares</code>去掉dump、logger中间件。</li>
<li>将<code>log.level</code>设置为info，<code>log.output-paths</code>去掉stdout。</li>
</ul><p>因为我们要在执行压力测试时分析程序的性能，所以需要设置<code>feature.profiling</code>为true，以开启性能分析。修改完之后，重新启动iam-apiserver。</p><h3>使用wrktest.sh测试IAM API接口性能</h3><p>关闭Debug配置选项之后，就可以执行<code>wrktest.sh</code>命令测试API性能了（默认测试的并发数为<code>200 500 1000 3000 5000 10000 15000 20000 25000 50000</code>）:</p><pre><code class="language-bash">$ scripts/wrktest.sh http://10.0.4.57:8080/healthz
</code></pre><p>生成的QPS &amp; TTLB图和成功率图分别如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/0a/0f/0aca3648e72974c5a32c2fbcfea8670f.png?wh=640x480" alt="图片" title="QPS &amp;平均响应时间图"></p><p>上图中，<code>X</code>轴为并发数（Concurrent），<code>Y</code>轴为每秒查询数（QPS）和请求响应时间（TTLB）。</p><p><img src="https://static001.geekbang.org/resource/image/a2/5d/a2d7866536ee96327a10614dd332475d.png?wh=640x480" alt="" title="成功率图"></p><p>上图中，<code>X</code>轴为并发数（Concurrent），<code>Y</code>轴为请求成功率。</p><p>通过上面两张图，你可以看到，API Server在并发数为<code>200</code>时，QPS最大；并发数为<code>500</code>，平均响应时间为<code>56.33ms</code>，成功率为 <code>100.00%</code> 。在并发数达到<code>1000</code>时，成功率开始下降。一些详细数据从图里看不到，你可以直接查看<code>apiserver.dat</code>文件，里面记录了每个并发下具体的QPS、TTLB和成功率数据。</p><p>现在我们有了API Server的性能数据，那么该API Server的QPS处于什么水平呢？一方面，你可以根据自己的业务需要来对比；另一方面，可以和性能更好的Web框架进行对比，总之需要有个参照。</p><p>这里用net/http构建最简单的HTTP服务器，使用相同的测试工具和测试服务器，测试性能并作对比。HTTP服务源码为（位于文件<a href="https://github.com/marmotedu/iam/blob/v1.0.8/tools/httptest/main.go">tools/httptest/main.go</a>中）：</p><pre><code class="language-go">package main

import (
	"fmt"
	"log"
	"net/http"
)

func main() {
	http.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
		message := `{"status":"ok"}`
		fmt.Fprint(w, message)
	})

	addr := ":6667"
	fmt.Printf("Serving http service on %s\n", addr)
	log.Fatal(http.ListenAndServe(addr, nil))
}
</code></pre><p>我们将上述HTTP服务的请求路径设置为<code>/healthz</code>，并且返回<code>{"status":"ok"}</code>，跟API Server的接口返回数据完全一样。通过这种方式，你可以排除因为返回数据大小不同而造成的性能差异。</p><p>可以看到，该HTTP服务器很简单，只是利用<code>net/http</code>包最原生的功能，在Go中几乎所有的Web框架都是基于<code>net/http</code>包封装的。既然是封装，肯定比不上原生的性能，所以我们要把它跟用<code>net/http</code>直接启动的HTTP服务接口的性能进行对比，来衡量我们的API Server性能。</p><p>我们需要执行相同的wrk测试，并将结果跟API Server的测试结果进行对比，将对比结果绘制成对比图。具体对比过程可以分为3步。</p><p>第一步，启动HTTP服务。</p><p>在iam源码根目录下执行如下命令：</p><pre><code class="language-bash">$ go run tools/httptest/main.go
</code></pre><p>第二步，执行<code>wrktest.sh</code>脚本，测试该HTTP服务的性能：</p><pre><code class="language-bash">$ scripts/wrktest.sh -n http http://10.0.4.57:6667/healthz
</code></pre><p>上述命令会生成  <code>_output/wrk/http.dat</code> 文件。</p><p>第三步，对比两次性能测试数据：</p><pre><code class="language-bash">$ scripts/wrktest.sh diff _output/wrk/apiserver.dat _output/wrk/http.dat
</code></pre><p>生成的两张对比图表，如下所示：</p><p><img src="https://static001.geekbang.org/resource/image/51/f4/51eacf20d190080bf8b42e2f43yy00f4.png?wh=640x480" alt="图片" title="QPS &amp;平均响应时间对比"></p><p><img src="https://static001.geekbang.org/resource/image/e1/99/e1cc646da40036a44e5300ed2bef8999.png?wh=640x480" alt="图片" title="成功率对比"></p><p>通过上面两张对比图，我们可以看出，API Server在QPS、响应时间和成功率上都不如原生的HTTP Server，特别是QPS，最大QPS只有原生HTTP Server 最大QPS的<code>13.68%</code>，性能需要调优。</p><h2>API Server性能分析</h2><p>上面，我们测试了API接口的性能，如果性能不合预期，我们还需要分析性能数据，并优化性能。</p><p>在分析前我们需要对API Server加压，在加压的情况下，API接口的性能才更可能暴露出来，所以继续执行如下命令：</p><pre><code class="language-bash">$ scripts/wrktest.sh http://10.0.4.57:8080/healthz
</code></pre><p>在上述命令执行压力测试期间，可以打开另外一个Linux终端，使用<code>go tool pprof</code>工具分析HTTP的profile文件：</p><pre><code class="language-bash">$ go tool pprof http://10.0.4.57:8080/debug/pprof/profile
</code></pre><p>执行完<code>go tool pprof</code>后，因为需要采集性能数据，所以该命令会阻塞30s。</p><p>在pprof交互shell中，执行<code>top -cum</code>查看累积采样时间，我们执行<code>top30 -cum</code>，多观察一些函数：</p><pre><code class="language-bash">(pprof) top20 -cum
Showing nodes accounting for 32.12s, 39.62% of 81.07s total
Dropped 473 nodes (cum &lt;= 0.41s)
Showing top 20 nodes out of 167
(pprof) top30 -cum
Showing nodes accounting for 11.82s, 20.32% of 58.16s total
Dropped 632 nodes (cum &lt;= 0.29s)
Showing top 30 nodes out of 239
      flat  flat%   sum%        cum   cum%
     0.10s  0.17%  0.17%     51.59s 88.70%  net/http.(*conn).serve
     0.01s 0.017%  0.19%     42.86s 73.69%  net/http.serverHandler.ServeHTTP
     0.04s 0.069%  0.26%     42.83s 73.64%  github.com/gin-gonic/gin.(*Engine).ServeHTTP
     0.01s 0.017%  0.28%     42.67s 73.37%  github.com/gin-gonic/gin.(*Engine).handleHTTPRequest
     0.08s  0.14%  0.41%     42.59s 73.23%  github.com/gin-gonic/gin.(*Context).Next (inline)
     0.03s 0.052%  0.46%     42.58s 73.21%  .../internal/pkg/middleware.RequestID.func1
         0     0%  0.46%     41.02s 70.53%  .../internal/pkg/middleware.Context.func1
     0.01s 0.017%  0.48%     40.97s 70.44%  github.com/gin-gonic/gin.CustomRecoveryWithWriter.func1
     0.03s 0.052%  0.53%     40.95s 70.41%  .../internal/pkg/middleware.LoggerWithConfig.func1
     0.01s 0.017%  0.55%     33.46s 57.53%  .../internal/pkg/middleware.NoCache
     0.08s  0.14%  0.69%     32.58s 56.02%  github.com/tpkeeper/gin-dump.DumpWithOptions.func1
     0.03s 0.052%  0.74%     24.73s 42.52%  github.com/tpkeeper/gin-dump.FormatToBeautifulJson
     0.02s 0.034%  0.77%     22.73s 39.08%  github.com/tpkeeper/gin-dump.BeautifyJsonBytes
     0.08s  0.14%  0.91%     16.39s 28.18%  github.com/tpkeeper/gin-dump.format
     0.21s  0.36%  1.27%     16.38s 28.16%  github.com/tpkeeper/gin-dump.formatMap
     3.75s  6.45%  7.72%     13.71s 23.57%  runtime.mallocgc
     ...
</code></pre><p>因为<code>top30</code>内容过多，这里只粘贴了耗时最多的一些关联函数。从上面的列表中，可以看到有ServeHTTP类的函数，这些函数是gin/http自带的函数，我们无需对此进行优化。</p><p>还有这样一些函数：</p><pre><code class="language-go">.../gin.(*Context).Next (inline)
.../internal/pkg/middleware.RequestID.func1
.../internal/pkg/middleware.Context.func1
github.com/gin-gonic/gin.CustomRecoveryWithWriter.func1
.../internal/pkg/middleware.LoggerWithConfig.func1
.../internal/pkg/middleware.NoCache
github.com/tpkeeper/gin-dump.DumpWithOptions.func1
</code></pre><p>可以看到，<code>middleware.RequestID.func1</code>、<code>middleware.Context.func1</code>、<code>gin.CustomRecoveryWithWriter.func1</code>、<code>middleware.LoggerWithConfig.func1</code>等，这些耗时较久的函数都是我们加载的Gin中间件。这些中间件消耗了大量的CPU时间，所以我们可以选择性加载这些中间件，删除一些不需要的中间件，来优化API Server的性能。<br>
假如我们暂时不需要这些中间件，也可以通过配置iam-apiserver的配置文件，将<code>server.middlewares</code>设置为空或者注释掉，然后重启iam-apiserver。重启后，再次执行<code>wrktest.sh</code>测试性能，并跟原生的HTTP Server性能进行对比，对比结果如下面2张图所示：</p><p><img src="https://static001.geekbang.org/resource/image/7b/c2/7bc94be0d44a5ac54cd0e199d2612ec2.png?wh=640x480" alt="图片" title="QPS &amp;平均响应时间对比"></p><p><img src="https://static001.geekbang.org/resource/image/88/a7/88e9fdfe7ba14061e979d0195b45cca7.png?wh=640x480" alt="图片" title="成功率对比"></p><p>可以看到，删除无用的Gin中间件后，API Server的性能有了很大的提升，并发数为<code>200</code>时性能最好，此时QPS为<code>47812</code>，响应时间为<code>4.33``ms</code>，成功率为<code>100.00``%</code>。在并发数为<code>50000</code>的时候，其QPS是原生HTTP Server的<code>75.02%</code>。</p><h3>API接口性能参考</h3><p>不同团队对API接口的性能要求不同，同一团队对每个API接口的性能要求也不同，所以并没有一个统一的数值标准来衡量API接口的性能，但可以肯定的是，性能越高越好。我根据自己的研发经验，在这里给出一个参考值（并发数可根据需要选择），如下表所示：</p><p><img src="https://static001.geekbang.org/resource/image/58/7c/581fc922afedaf36379c5a5d723ebd7c.jpg?wh=2248x585" alt=""></p><h2>API Server性能测试注意事项</h2><p>在进行API Server性能测试时，要考虑到API Server的性能影响因素。影响API Server性能的因素很多，大致可以分为两类，分别是Web框架的性能和API接口的性能。另外，在做性能测试时，还需要确保测试环境是一致的，最好是一个干净的测试环境。</p><h3>Web框架性能</h3><p>Web框架的性能至关重要，因为它会影响我们的每一个API接口的性能。</p><p>在设计阶段，我们会确定所使用的Web框架，这时候我们需要对Web框架有个初步的测试，确保我们选择的Web框架在性能和稳定性上都足够优秀。当整个Go后端服务开发完成之后，在上线之前，我们还需要对Web框架再次进行测试，确保按照我们最终的使用方式，Web框架仍然能够保持优秀的性能和稳定性。</p><p>我们通常会通过API接口来测试Web框架的性能，例如健康检查接口<code>/healthz</code>。我们需要保证该API接口足够简单，API接口里面不应该掺杂任何逻辑，只需要象征性地返回一个很小的返回内容即可。比如，这一讲中我们通过<code>/healthz</code>接口来测试Web框架的性能：</p><pre><code class="language-go">s.GET("/healthz", func(c *gin.Context) {
    core.WriteResponse(c, nil, map[string]string{"status": "ok"})
})
</code></pre><p>接口中只调用了<code>core.WriteResponse</code>函数，返回了<code>{"status":"ok"}</code>。这里使用<code>core.WriteResponse</code>函数返回请求数据，而不是直接返回<code>ok</code>字符串，这样做是为了保持API接口返回格式统一。</p><h3>API接口性能</h3><p>除了测试Web框架的性能，我们还可能需要测试某些重要的API接口，甚至所有API接口的性能。为了测试API接口在真实场景下的接口性能，我们会使用wrk这类HTTP压力测试工具，来模拟多个API请求，进而分析API的性能。</p><p>因为会模拟大量的请求，这时候测试写类接口，例如<code>Create</code>、<code>Update</code>、<code>Delete</code>等会存在一些问题，比如可能在数据库中插入了很多数据，导致磁盘空间被写满或者数据库被压爆。所以，针对写类接口，我们可以借助单元测试，来测试其性能。根据我的开发经验，写类接口通常不会有性能问题，反而读类接口更可能遇到性能问题。针对读类接口，我们可以使用wrk这类HTTP压力测试工具来进行测试。</p><h3>测试环境</h3><p>在做性能/压力测试时，为了不影响生产环境，要确保在测试环境进行压测，并且测试环境的网络不能影响到生产环境的网络。另外，为了更好地进行性能对比和分析，也要保证我们的测试方法和测试环境是一致的。这就要求我们最好将性能测试自动化，并且每次在同一个测试环境进行测试。</p><h2>总结</h2><p>在项目上线前，我们需要对API接口进行性能测试。通常API接口的性能延时要小于 <code>500ms</code> ，如果大于这个值，需要考虑优化性能。在进行性能测试时，需要确保每次测试都有一个一致的测试环境，这样不同测试之间的数据才具有可对比性。这一讲中，我推荐了一个比较优秀的性能测试工具 <code>wrk</code> ，我们可以编写shell脚本，将wrk的性能测试数据自动绘制成图，方便我们查看、对比性能。</p><p>如果发现API接口的性能不符合预期，我们可以借助 <code>go tool pprof</code> 工具来分析性能。在 <code>go tool pprof</code> 交互界面，执行 <code>top -cum</code> 命令查看累积采样时间，根据累积采样时间确定影响性能的代码，并优化代码。优化后，再进行测试，如果不满足，继续分析API接口的性能。如此往复，直到API接口的性能满足预期为止。</p><h2>课后练习</h2><ol>
<li>选择一个项目，并使用wrktest.sh脚本测试其API接口，分析并优化API接口的性能。</li>
<li>思考下，在你的工作中，还有没有其他好的API接口性能分析方法，欢迎在留言区分享讨论。</li>
</ol><p>欢迎你在留言区与我交流讨论，我们下一讲见。</p>
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
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLDUJyeq54fiaXAgF62tNeocO3lHsKT4mygEcNoZLnibg6ONKicMgCgUHSfgW8hrMUXlwpNSzR8MHZwg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>types</span>
  </div>
  <div class="_2_QraFYR_0">这篇文章系统的介绍了性能指标的测试<br>性能指标主要是：不同并发量下的QPS+延迟<br>此处有2个疑问：<br>1. 如何选取性能指标作为服务能力的上限（并发+QPS+延迟）<br>2. 如何保护服务的访问不会超过性能上限？（使用微服务的治理方法，限流、熔断之类的？）</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. 根据业务实际需求吧，另外也根据服务器配置。如果延时&lt;500ms，系统并发和QPS足以支撑业务高峰期，我感觉就是可以的。<br>2. 对的，通过网关给服务加上限流、熔断、降级的功能</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-27 13:48:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/87/64/3882d90d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>yandongxiao</span>
  </div>
  <div class="_2_QraFYR_0">总结：<br>性能测试一般包括框架测试和API接口测试。<br>API 性能测试指标：并发数、QPS、请求响应事件（TTLB），注意QPS与TPS的区别。TPS是针对多个接口进行测试。离开并发数谈QPS，毫无意义。<br>框架性能测试：提供一个很简单的接口，如 &#47;healthz，与 net&#47;http 框架进行对比；<br>API 性能接口：针对写类接口可通过单元测试来测试其性能，针对读类接口，可通过wrk进行测试<br>介绍 wrk 的使用方法<br>介绍基于wrk 的输出结果：并发数、QPS、平均响应时间，平均错误数，使用工具绘制图表。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: wrk是我自己写的一个性能测试脚本，挺好用的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-04 22:44:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2a/0a/e4/c576d62b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>阿波罗尼斯圆</span>
  </div>
  <div class="_2_QraFYR_0">为什么写类接口通常不会有性能问题，写接口不是一般都比读接口慢吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 写类接口也可能会有性能问题，但影响没有那么大。因为相比于读类接口，读类接口调用频度更高，用户体验影响最大。<br><br>在接口性能优化时，写类接口和读类接口性能都要优化</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-04 17:04:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKotsBr2icbYNYlRSlicGUD1H7lulSTQUAiclsEz9gnG5kCW9qeDwdYtlRMXic3V6sj9UrfKLPJnQojag/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ppd0705</span>
  </div>
  <div class="_2_QraFYR_0">太硬核了，写shell来画图</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-22 23:52:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/60/a1/8f003697.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>静心</span>
  </div>
  <div class="_2_QraFYR_0">在本机压测远程服务的时候，经常由于线程或打开文件数的限制，导致测试机客户端成为并发的瓶颈。不知道老师有没有什么解决的好办法？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 工作中没有遇到过这种问题。可以通过分布式测试，由多个客户端同时发起测试请求，最后汇总结果</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-03 16:44:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/70/4a/30cf63db.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>丁卯</span>
  </div>
  <div class="_2_QraFYR_0">在满足预期要求的情况下，服务器状态稳定，单台服务器QPS要求在1000+，这个服务器的QPS怎么测？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以用wrk这类压测工具来测。<br><br>服务器的QPS其实就是应用的QPS。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-21 16:09:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/28/83/17/df99b53d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>随风而过</span>
  </div>
  <div class="_2_QraFYR_0">一般都是使用Jmeter，wrk倒是没用过，这个工具比Jmeter优秀很多，学到了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-09 11:34:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLDUJyeq54fiaXAgF62tNeocO3lHsKT4mygEcNoZLnibg6ONKicMgCgUHSfgW8hrMUXlwpNSzR8MHZwg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>types</span>
  </div>
  <div class="_2_QraFYR_0">这篇文章系统的介绍了性能测试的方法<br>通过性能测试可以得到在一定并发下的QPS和延迟<br>得到性能数据以后，产生一个疑问：<br>1. 如何选取一个性能指标作为服务器的上限（并发 QPS 延迟）</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 有回复你</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-27 13:44:28</div>
  </div>
</div>
</div>
</li>
</ul>