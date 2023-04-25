<audio title="06 _ 案例篇：系统的 CPU 使用率很高，但为啥却找不到高 CPU 的应用？" src="https://static001.geekbang.org/resource/audio/0b/70/0be8a102696d191c057158c7c9818f70.mp3" controls="controls"></audio> 
<p>你好，我是倪朋飞。</p><p>上一节我讲了 CPU 使用率是什么，并通过一个案例教你使用 top、vmstat、pidstat 等工具，排查高 CPU 使用率的进程，然后再使用 perf top 工具，定位应用内部函数的问题。不过就有人留言了，说似乎感觉高 CPU 使用率的问题，还是挺容易排查的。</p><p>那是不是所有 CPU 使用率高的问题，都可以这么分析呢？我想，你的答案应该是否定的。</p><p>回顾前面的内容，我们知道，系统的 CPU 使用率，不仅包括进程用户态和内核态的运行，还包括中断处理、等待 I/O 以及内核线程等。所以，<strong>当你发现系统的 CPU 使用率很高的时候，不一定能找到相对应的高 CPU 使用率的进程</strong>。</p><p>今天，我就用一个 Nginx + PHP 的 Web 服务的案例，带你来分析这种情况。</p><h2>案例分析</h2><h3>你的准备</h3><p>今天依旧探究系统CPU使用率高的情况，所以这次实验的准备工作，与上节课的准备工作基本相同，差别在于案例所用的 Docker 镜像不同。</p><p>本次案例还是基于 Ubuntu 18.04，同样适用于其他的 Linux 系统。我使用的案例环境如下所示：</p><ul>
<li>
<p>机器配置：2 CPU，8GB 内存</p>
</li>
<li>
<p>预先安装 docker、sysstat、perf、ab 等工具，如 apt install <a href="http://docker.io">docker.io</a> sysstat linux-tools-common apache2-utils</p>
</li>
</ul><!-- [[[read_end]]] --><p>前面我们讲到过，ab（apache bench）是一个常用的 HTTP 服务性能测试工具，这里同样用来模拟 Nginx 的客户端。由于 Nginx 和 PHP 的配置比较麻烦，我把它们打包成了两个 <a href="https://github.com/feiskyer/linux-perf-examples/tree/master/nginx-short-process">Docker 镜像</a>，这样只需要运行两个容器，就可以得到模拟环境。</p><p>注意，这个案例要用到两台虚拟机，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/90/3d/90c30b4f555218f77241bfe2ac27723d.png?wh=408*258" alt=""></p><p>你可以看到，其中一台用作 Web 服务器，来模拟性能问题；另一台用作 Web 服务器的客户端，来给 Web 服务增加压力请求。使用两台虚拟机是为了相互隔离，避免“交叉感染”。</p><p>接下来，我们打开两个终端，分别 SSH 登录到两台机器上，并安装上述工具。</p><p>同样注意，下面所有命令都默认以 root 用户运行，如果你是用普通用户身份登陆系统，请运行 sudo su root 命令切换到 root 用户。</p><p>走到这一步，准备工作就完成了。接下来，我们正式进入操作环节。</p><blockquote>
<p>温馨提示：案例中 PHP 应用的核心逻辑比较简单，你可能一眼就能看出问题，但实际生产环境中的源码就复杂多了。所以，我依旧建议，<strong>操作之前别看源码</strong>，避免先入为主，而要把它当成一个黑盒来分析。这样，你可以更好把握，怎么从系统的资源使用问题出发，分析出瓶颈所在的应用，以及瓶颈在应用中大概的位置。</p>
</blockquote><h3>操作和分析</h3><p>首先，我们在第一个终端，执行下面的命令运行 Nginx 和 PHP 应用：</p><pre><code>$ docker run --name nginx -p 10000:80 -itd feisky/nginx:sp
$ docker run --name phpfpm -itd --network container:nginx feisky/php-fpm:sp
</code></pre><p>然后，在第二个终端，使用 curl 访问 http://[VM1的IP]:10000，确认 Nginx 已正常启动。你应该可以看到 It works! 的响应。</p><pre><code># 192.168.0.10是第一台虚拟机的IP地址
$ curl http://192.168.0.10:10000/
It works!
</code></pre><p>接着，我们来测试一下这个 Nginx 服务的性能。在第二个终端运行下面的 ab 命令。要注意，与上次操作不同的是，这次我们需要并发100个请求测试Nginx性能，总共测试1000个请求。</p><pre><code># 并发100个请求测试Nginx性能，总共测试1000个请求
$ ab -c 100 -n 1000 http://192.168.0.10:10000/
This is ApacheBench, Version 2.3 &lt;$Revision: 1706008 $&gt;
Copyright 1996 Adam Twiss, Zeus Technology Ltd, 
...
Requests per second:    87.86 [#/sec] (mean)
Time per request:       1138.229 [ms] (mean)
...
</code></pre><p>从ab的输出结果我们可以看到，Nginx能承受的每秒平均请求数，只有 87 多一点，是不是感觉它的性能有点差呀。那么，到底是哪里出了问题呢？我们再用 top 和 pidstat 来观察一下。</p><p>这次，我们在第二个终端，将测试的并发请求数改成5，同时把请求时长设置为10分钟（-t 600）。这样，当你在第一个终端使用性能分析工具时， Nginx 的压力还是继续的。</p><p>继续在第二个终端运行 ab 命令：</p><pre><code>$ ab -c 5 -t 600 http://192.168.0.10:10000/
</code></pre><p>然后，我们在第一个终端运行 top 命令，观察系统的 CPU 使用情况：</p><pre><code>$ top
...
%Cpu(s): 80.8 us, 15.1 sy,  0.0 ni,  2.8 id,  0.0 wa,  0.0 hi,  1.3 si,  0.0 st
...

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 6882 root      20   0    8456   5052   3884 S   2.7  0.1   0:04.78 docker-containe
 6947 systemd+  20   0   33104   3716   2340 S   2.7  0.0   0:04.92 nginx
 7494 daemon    20   0  336696  15012   7332 S   2.0  0.2   0:03.55 php-fpm
 7495 daemon    20   0  336696  15160   7480 S   2.0  0.2   0:03.55 php-fpm
10547 daemon    20   0  336696  16200   8520 S   2.0  0.2   0:03.13 php-fpm
10155 daemon    20   0  336696  16200   8520 S   1.7  0.2   0:03.12 php-fpm
10552 daemon    20   0  336696  16200   8520 S   1.7  0.2   0:03.12 php-fpm
15006 root      20   0 1168608  66264  37536 S   1.0  0.8   9:39.51 dockerd
 4323 root      20   0       0      0      0 I   0.3  0.0   0:00.87 kworker/u4:1
...
</code></pre><p>观察 top 输出的进程列表可以发现，CPU 使用率最高的进程也只不过才 2.7%，看起来并不高。</p><p>然而，再看系统 CPU 使用率（ %Cpu ）这一行，你会发现，系统的整体 CPU 使用率是比较高的：用户 CPU 使用率（us）已经到了 80%，系统 CPU 为 15.1%，而空闲 CPU （id）则只有 2.8%。</p><p>为什么用户 CPU 使用率这么高呢？我们再重新分析一下进程列表，看看有没有可疑进程：</p><ul>
<li>
<p>docker-containerd 进程是用来运行容器的，2.7% 的 CPU 使用率看起来正常；</p>
</li>
<li>
<p>Nginx 和 php-fpm 是运行 Web 服务的，它们会占用一些 CPU 也不意外，并且 2% 的 CPU 使用率也不算高；</p>
</li>
<li>
<p>再往下看，后面的进程呢，只有 0.3% 的 CPU 使用率，看起来不太像会导致用户 CPU 使用率达到 80%。</p>
</li>
</ul><p>那就奇怪了，明明用户 CPU 使用率都80%了，可我们挨个分析了一遍进程列表，还是找不到高 CPU 使用率的进程。看来top是不管用了，那还有其他工具可以查看进程 CPU 使用情况吗？不知道你记不记得我们的老朋友  pidstat，它可以用来分析进程的 CPU 使用情况。</p><p>接下来，我们还是在第一个终端，运行 pidstat 命令：</p><pre><code># 间隔1秒输出一组数据（按Ctrl+C结束）
$ pidstat 1
...
04:36:24      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
04:36:25        0      6882    1.00    3.00    0.00    0.00    4.00     0  docker-containe
04:36:25      101      6947    1.00    2.00    0.00    1.00    3.00     1  nginx
04:36:25        1     14834    1.00    1.00    0.00    1.00    2.00     0  php-fpm
04:36:25        1     14835    1.00    1.00    0.00    1.00    2.00     0  php-fpm
04:36:25        1     14845    0.00    2.00    0.00    2.00    2.00     1  php-fpm
04:36:25        1     14855    0.00    1.00    0.00    1.00    1.00     1  php-fpm
04:36:25        1     14857    1.00    2.00    0.00    1.00    3.00     0  php-fpm
04:36:25        0     15006    0.00    1.00    0.00    0.00    1.00     0  dockerd
04:36:25        0     15801    0.00    1.00    0.00    0.00    1.00     1  pidstat
04:36:25        1     17084    1.00    0.00    0.00    2.00    1.00     0  stress
04:36:25        0     31116    0.00    1.00    0.00    0.00    1.00     0  atopacctd
...
</code></pre><p>观察一会儿，你是不是发现，所有进程的 CPU 使用率也都不高啊，最高的 Docker 和 Nginx 也只有 4% 和 3%，即使所有进程的 CPU 使用率都加起来，也不过是 21%，离 80% 还差得远呢！</p><p>最早的时候，我碰到这种问题就完全懵了：明明用户 CPU 使用率已经高达 80%，但我却怎么都找不到是哪个进程的问题。到这里，你也可以想想，你是不是也遇到过这种情况？还能不能再做进一步的分析呢？</p><p>后来我发现，会出现这种情况，很可能是因为前面的分析漏了一些关键信息。你可以先暂停一下，自己往上翻，重新操作检查一遍。或者，我们一起返回去分析 top 的输出，看看能不能有新发现。</p><p>现在，我们回到第一个终端，重新运行 top 命令，并观察一会儿：</p><pre><code>$ top
top - 04:58:24 up 14 days, 15:47,  1 user,  load average: 3.39, 3.82, 2.74
Tasks: 149 total,   6 running,  93 sleeping,   0 stopped,   0 zombie
%Cpu(s): 77.7 us, 19.3 sy,  0.0 ni,  2.0 id,  0.0 wa,  0.0 hi,  1.0 si,  0.0 st
KiB Mem :  8169348 total,  2543916 free,   457976 used,  5167456 buff/cache
KiB Swap:        0 total,        0 free,        0 used.  7363908 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 6947 systemd+  20   0   33104   3764   2340 S   4.0  0.0   0:32.69 nginx
 6882 root      20   0   12108   8360   3884 S   2.0  0.1   0:31.40 docker-containe
15465 daemon    20   0  336696  15256   7576 S   2.0  0.2   0:00.62 php-fpm
15466 daemon    20   0  336696  15196   7516 S   2.0  0.2   0:00.62 php-fpm
15489 daemon    20   0  336696  16200   8520 S   2.0  0.2   0:00.62 php-fpm
 6948 systemd+  20   0   33104   3764   2340 S   1.0  0.0   0:00.95 nginx
15006 root      20   0 1168608  65632  37536 S   1.0  0.8   9:51.09 dockerd
15476 daemon    20   0  336696  16200   8520 S   1.0  0.2   0:00.61 php-fpm
15477 daemon    20   0  336696  16200   8520 S   1.0  0.2   0:00.61 php-fpm
24340 daemon    20   0    8184   1616    536 R   1.0  0.0   0:00.01 stress
24342 daemon    20   0    8196   1580    492 R   1.0  0.0   0:00.01 stress
24344 daemon    20   0    8188   1056    492 R   1.0  0.0   0:00.01 stress
24347 daemon    20   0    8184   1356    540 R   1.0  0.0   0:00.01 stress
...
</code></pre><p>这次从头开始看 top 的每行输出，咦？Tasks 这一行看起来有点奇怪，就绪队列中居然有 6 个 Running 状态的进程（6 running），是不是有点多呢？</p><p>回想一下 ab 测试的参数，并发请求数是 5。再看进程列表里， php-fpm 的数量也是 5，再加上 Nginx，好像同时有 6 个进程也并不奇怪。但真的是这样吗？</p><p>再仔细看进程列表，这次主要看 Running（R） 状态的进程。你有没有发现， Nginx 和所有的 php-fpm 都处于Sleep（S）状态，而真正处于 Running（R）状态的，却是几个 stress 进程。这几个 stress 进程就比较奇怪了，需要我们做进一步的分析。</p><p>我们还是使用 pidstat 来分析这几个进程，并且使用 -p 选项指定进程的 PID。首先，从上面 top 的结果中，找到这几个进程的 PID。比如，先随便找一个 24344，然后用 pidstat 命令看一下它的 CPU 使用情况：</p><pre><code>$ pidstat -p 24344

16:14:55      UID       PID    %usr %system  %guest   %wait    %CPU   CPU  Command
</code></pre><p>奇怪，居然没有任何输出。难道是pidstat 命令出问题了吗？之前我说过，<strong>在怀疑性能工具出问题前，最好还是先用其他工具交叉确认一下</strong>。那用什么工具呢？ ps 应该是最简单易用的。我们在终端里运行下面的命令，看看 24344 进程的状态：</p><pre><code># 从所有进程中查找PID是24344的进程
$ ps aux | grep 24344
root      9628  0.0  0.0  14856  1096 pts/0    S+   16:15   0:00 grep --color=auto 24344
</code></pre><p>还是没有输出。现在终于发现问题，原来这个进程已经不存在了，所以 pidstat 就没有任何输出。既然进程都没了，那性能问题应该也跟着没了吧。我们再用 top 命令确认一下：</p><pre><code>$ top
...
%Cpu(s): 80.9 us, 14.9 sy,  0.0 ni,  2.8 id,  0.0 wa,  0.0 hi,  1.3 si,  0.0 st
...

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 6882 root      20   0   12108   8360   3884 S   2.7  0.1   0:45.63 docker-containe
 6947 systemd+  20   0   33104   3764   2340 R   2.7  0.0   0:47.79 nginx
 3865 daemon    20   0  336696  15056   7376 S   2.0  0.2   0:00.15 php-fpm
  6779 daemon    20   0    8184   1112    556 R   0.3  0.0   0:00.01 stress
...
</code></pre><p>好像又错了。结果还跟原来一样，用户 CPU 使用率还是高达 80.9%，系统 CPU 接近 15%，而空闲 CPU 只有 2.8%，Running 状态的进程有 Nginx、stress等。</p><p>可是，刚刚我们看到stress 进程不存在了，怎么现在还在运行呢？再细看一下 top 的输出，原来，这次 stress 进程的 PID 跟前面不一样了，原来的 PID 24344 不见了，现在的是 6779。</p><p>进程的 PID 在变，这说明什么呢？在我看来，要么是这些进程在不停地重启，要么就是全新的进程，这无非也就两个原因：</p><ul>
<li>
<p>第一个原因，进程在不停地崩溃重启，比如因为段错误、配置错误等等，这时，进程在退出后可能又被监控系统自动重启了。</p>
</li>
<li>
<p>第二个原因，这些进程都是短时进程，也就是在其他应用内部通过 exec 调用的外面命令。这些命令一般都只运行很短的时间就会结束，你很难用 top 这种间隔时间比较长的工具发现（上面的案例，我们碰巧发现了）。</p>
</li>
</ul><p>至于 stress，我们前面提到过，它是一个常用的压力测试工具。它的 PID 在不断变化中，看起来像是被其他进程调用的短时进程。要想继续分析下去，还得找到它们的父进程。</p><p>要怎么查找一个进程的父进程呢？没错，用 pstree  就可以用树状形式显示所有进程之间的关系：</p><pre><code>$ pstree | grep stress
        |-docker-containe-+-php-fpm-+-php-fpm---sh---stress
        |         |-3*[php-fpm---sh---stress---stress]
</code></pre><p>从这里可以看到，stress 是被 php-fpm 调用的子进程，并且进程数量不止一个（这里是3个）。找到父进程后，我们能进入 app 的内部分析了。</p><p>首先，当然应该去看看它的源码。运行下面的命令，把案例应用的源码拷贝到 app 目录，然后再执行 grep 查找是不是有代码再调用 stress 命令：</p><pre><code># 拷贝源码到本地
$ docker cp phpfpm:/app .

# grep 查找看看是不是有代码在调用stress命令
$ grep stress -r app
app/index.php:// fake I/O with stress (via write()/unlink()).
app/index.php:$result = exec(&quot;/usr/local/bin/stress -t 1 -d 1 2&gt;&amp;1&quot;, $output, $status);
</code></pre><p>找到了，果然是 app/index.php 文件中直接调用了 stress 命令。</p><p>再来看看<a href="https://github.com/feiskyer/linux-perf-examples/blob/master/nginx-short-process/app/index.php"> app/index.php </a>的源代码：</p><pre><code>$ cat app/index.php
&lt;?php
// fake I/O with stress (via write()/unlink()).
$result = exec(&quot;/usr/local/bin/stress -t 1 -d 1 2&gt;&amp;1&quot;, $output, $status);
if (isset($_GET[&quot;verbose&quot;]) &amp;&amp; $_GET[&quot;verbose&quot;]==1 &amp;&amp; $status != 0) {
  echo &quot;Server internal error: &quot;;
  print_r($output);
} else {
  echo &quot;It works!&quot;;
}
?&gt;
</code></pre><p>可以看到，源码里对每个请求都会调用一个 stress 命令，模拟 I/O 压力。从注释上看，stress 会通过 write() 和 unlink() 对 I/O 进程进行压测，看来，这应该就是系统 CPU 使用率升高的根源了。</p><p>不过，stress 模拟的是 I/O 压力，而之前在 top 的输出中看到的，却一直是用户 CPU 和系统 CPU 升高，并没见到 iowait 升高。这又是怎么回事呢？stress 到底是不是 CPU 使用率升高的原因呢？</p><p>我们还得继续往下走。从代码中可以看到，给请求加入 verbose=1 参数后，就可以查看 stress 的输出。你先试试看，在第二个终端运行：</p><pre><code>$ curl http://192.168.0.10:10000?verbose=1
Server internal error: Array
(
    [0] =&gt; stress: info: [19607] dispatching hogs: 0 cpu, 0 io, 0 vm, 1 hdd
    [1] =&gt; stress: FAIL: [19608] (563) mkstemp failed: Permission denied
    [2] =&gt; stress: FAIL: [19607] (394) &lt;-- worker 19608 returned error 1
    [3] =&gt; stress: WARN: [19607] (396) now reaping child worker processes
    [4] =&gt; stress: FAIL: [19607] (400) kill error: No such process
    [5] =&gt; stress: FAIL: [19607] (451) failed run completed in 0s
)
</code></pre><p>看错误消息 <span class="orange">mkstemp failed: Permission denied </span>，以及 <span class="orange">failed run completed in 0s</span>。原来 stress 命令并没有成功，它因为权限问题失败退出了。看来，我们发现了一个 PHP 调用外部 stress 命令的 bug：没有权限创建临时文件。</p><p>从这里我们可以猜测，正是由于权限错误，大量的 stress 进程在启动时初始化失败，进而导致用户 CPU 使用率的升高。</p><p>分析出问题来源，下一步是不是就要开始优化了呢？当然不是！既然只是猜测，那就需要再确认一下，这个猜测到底对不对，是不是真的有大量的 stress 进程。该用什么工具或指标呢？</p><p>我们前面已经用了 top、pidstat、pstree 等工具，没有发现大量的 stress 进程。那么，还有什么其他的工具可以用吗？</p><p>还记得上一期提到的 perf 吗？它可以用来分析 CPU 性能事件，用在这里就很合适。依旧在第一个终端中运行 perf record -g 命令 ，并等待一会儿（比如15秒）后按 Ctrl+C 退出。然后再运行 perf report 查看报告：</p><pre><code># 记录性能事件，等待大约15秒后按 Ctrl+C 退出
$ perf record -g

# 查看报告
$ perf report
</code></pre><p>这样，你就可以看到下图这个性能报告：</p><p><img src="https://static001.geekbang.org/resource/image/c9/33/c99445b401301147fa41cb2b5739e833.png?wh=720*527" alt=""></p><p>你看，stress 占了所有CPU时钟事件的 77%，而 stress  调用调用栈中比例最高的，是随机数生成函数 random()，看来它的确就是 CPU 使用率升高的元凶了。随后的优化就很简单了，只要修复权限问题，并减少或删除 stress 的调用，就可以减轻系统的 CPU 压力。</p><p>当然，实际生产环境中的问题一般都要比这个案例复杂，在你找到触发瓶颈的命令行后，却可能发现，这个外部命令的调用过程是应用核心逻辑的一部分，并不能轻易减少或者删除。</p><p>这时，你就得继续排查，为什么被调用的命令，会导致 CPU 使用率升高或 I/O 升高等问题。这些复杂场景的案例，我会在后面的综合实战里详细分析。</p><p>最后，在案例结束时，不要忘了清理环境，执行下面的 Docker 命令，停止案例中用到的 Nginx 进程：</p><pre><code>$ docker rm -f nginx phpfpm
</code></pre><h2>execsnoop</h2><p>在这个案例中，我们使用了 top、pidstat、pstree 等工具分析了系统 CPU 使用率高的问题，并发现 CPU 升高是短时进程 stress 导致的，但是整个分析过程还是比较复杂的。对于这类问题，有没有更好的方法监控呢？</p><p><a href="https://github.com/brendangregg/perf-tools/blob/master/execsnoop">execsnoop</a> 就是一个专为短时进程设计的工具。它通过 ftrace 实时监控进程的 exec() 行为，并输出短时进程的基本信息，包括进程 PID、父进程 PID、命令行参数以及执行的结果。</p><p>比如，用 execsnoop 监控上述案例，就可以直接得到 stress 进程的父进程 PID 以及它的命令行参数，并可以发现大量的 stress 进程在不停启动：</p><pre><code># 按 Ctrl+C 结束
$ execsnoop
PCOMM            PID    PPID   RET ARGS
sh               30394  30393    0
stress           30396  30394    0 /usr/local/bin/stress -t 1 -d 1
sh               30398  30393    0
stress           30399  30398    0 /usr/local/bin/stress -t 1 -d 1
sh               30402  30400    0
stress           30403  30402    0 /usr/local/bin/stress -t 1 -d 1
sh               30405  30393    0
stress           30407  30405    0 /usr/local/bin/stress -t 1 -d 1
...
</code></pre><p>execsnoop 所用的 ftrace 是一种常用的动态追踪技术，一般用于分析 Linux 内核的运行时行为，后面课程我也会详细介绍并带你使用。</p><h2>小结</h2><p>碰到常规问题无法解释的 CPU 使用率情况时，首先要想到有可能是短时应用导致的问题，比如有可能是下面这两种情况。</p><ul>
<li>
<p>第一，<strong>应用里直接调用了其他二进制程序，这些程序通常运行时间比较短，通过 top 等工具也不容易发现</strong>。</p>
</li>
<li>
<p>第二，<strong>应用本身在不停地崩溃重启，而启动过程的资源初始化，很可能会占用相当多的 CPU</strong>。</p>
</li>
</ul><p>对于这类进程，我们可以用 pstree 或者 execsnoop 找到它们的父进程，再从父进程所在的应用入手，排查问题的根源。</p><h2>思考</h2><p>最后，我想邀请你一起来聊聊，你所碰到的 CPU 性能问题。有没有哪个印象深刻的经历可以跟我分享呢？或者，在今天的案例操作中，你遇到了什么问题，又解决了哪些呢？你可以结合我的讲述，总结自己的思路。</p><p>欢迎在留言区和我讨论，也欢迎把这篇文章分享给你的同事、朋友。我们一起在实战中演练，在交流中进步。</p><p><img src="https://static001.geekbang.org/resource/image/56/52/565d66d658ad23b2f4997551db153852.jpg?wh=1110*549" alt=""></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/a4/39/6a5cd1d8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>sotey</span>
  </div>
  <div class="_2_QraFYR_0">对老师膜拜！今天一早生产tomcat夯住了，16颗cpu全部98%以上，使用老师的方法加上java的工具成功定位到了问题线程和问题函数。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 举一反三动手实践的楷模😊 希望看到更多的人把这些思路用起来</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-03 11:29:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/35/25/bab760a1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>好好学习</span>
  </div>
  <div class="_2_QraFYR_0">perf record -ag -- sleep 2;perf report<br>一部到位</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-10 21:12:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/a0/39/ddcf26ac.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>bruceding</span>
  </div>
  <div class="_2_QraFYR_0">sar -w 或者 sar -w 1  也能直观的看到每秒生成线程或者进程的数量。Brendan Gregg 确实是这个领域的大师，贡献了很多的技术理念和实践经验。他的《性能之巅》 可以和本课对比着看，会有更多的理解。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-12 07:50:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/a0/39/ddcf26ac.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>bruceding</span>
  </div>
  <div class="_2_QraFYR_0">http:&#47;&#47;blog.bruceding.com&#47;420.html  这个是之前的优化经历，通过 perf + 火焰图，定位热点代码，结合业务和网络分析，最终确定问题原因</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞，再加上APM定位就更简单了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-11 15:38:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/f3/ea/2b2adda5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>EncodedStar</span>
  </div>
  <div class="_2_QraFYR_0">这种课程，多学多益，一旦遇到被你解决了，你就是公司最靓的仔~~~</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-07 11:30:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/54/9a/76c0af70.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>每天晒白牙</span>
  </div>
  <div class="_2_QraFYR_0">【D6补卡】<br>如果碰到不好解释的CPU问题时，比如现象：<br>通过top观察CPU使用率很高，但是看下面的进程的CPU使用率好像很正常，通过pidstat命令查看cpu也很正常。但通过top查看task数量不正常，处于R状态的进程是可疑点。<br><br>首先要想到可能是短时间的应用导致的问题，如下面的两个：<br>（1）应用里直接调用了其他二进制程序，这些程序通常运行时间比较短，通过top等工具发现不了<br>（2）应用本身在不停地崩溃重启，而启动过程的资源初始化，很可能会占用很多CPU资源<br><br>对于这类进程就需要 pstree 或execsnoop命令查找父进程，再从父进程的应用入手，找原因。<br>我实战的问题：<br>（1）centos的系统，发现通过 yum install pstress命令或其他命令，总是说没有匹配的软件包，这个很头疼<br>①pstress装不上可以尝试 yum install psmisc<br>②execsnoop这个命令我没有装上，想请教下老师如何解决这种yum install命令报没有匹配包的情况<br>（2）然后就是通过perf命令只能看到十六进制符号，看不到具体函数名的问题，这个可以使用上篇文章中我的留言的方法，我在copy一份，供大家参考<br>分析：当没有看到函数名，只看到十六进制时，说明perf无法找到待分析进程所依赖的库。<br>解决办法：<br>在容器外面把分析记录保存，到容器里面查看结果<br>操作：<br>（1）在centos系统上运行 perf record -g ，执行一会儿按ctrl+c停止<br>（2）把生成的perf.data（通常文件生成在命令执行的当前目录下，当然可以通过find | grep perf.data或 find &#47; -name perf.data查看路径）文件拷贝到容器里面分析：<br>docker cp perf.data phpfpm:&#47;tpm<br>docker exec -i -t phpfpm bash<br>cd &#47;tmp&#47;<br>apt-get update &amp;&amp; apt-get install -y linux-perf linux-tools procps<br>perf_4.9 report<br>这样就可以看到函数名了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-05 07:18:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/20/1d/0c1a184c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>罗辑思维</span>
  </div>
  <div class="_2_QraFYR_0">#下载execsnoop#<br>1 cd &#47;usr&#47;bin<br>2 wget https:&#47;&#47;raw.githubusercontent.com&#47;brendangregg&#47;perf-tools&#47;master&#47;execsnoop<br>3 chmod 755 execsnoop </div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-21 14:37:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/8c/3e/e767c15f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>western</span>
  </div>
  <div class="_2_QraFYR_0">感觉看侦探小说，遍看遍等老师说那句话——”真相只有一个”</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-06 10:19:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/2o1Izf2YyJSnnI0ErZ51pYRlnrmibqUTaia3tCU1PjMxuwyXSKOLUYiac2TQ5pd5gNGvS81fVqKWGvDsZLTM8zhWg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>划时代</span>
  </div>
  <div class="_2_QraFYR_0">今天早上收到线上一台机器的CPU使用率告警邮件，马上登陆机器查看告警进程号的&#47;proc情况，使用uptime、pidstat和top命令查当前运行情况，后面定位到是crontab定时任务中的程序引起负载过高。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-04 11:03:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/f9/3c/f75ab868.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>风</span>
  </div>
  <div class="_2_QraFYR_0">老师好，我在实验的过程中，在最后使用 perf record -ag 的时候，发现记录下来的值，其中 stress 并不是消耗 CPU 最猛的进程，而是swapper，不知道什么原因？碰到这种情况时，该如何继续排查下去？以下是我的 perf report<br>Samples: 223K of event &#39;cpu-clock&#39;, Event count (approx.): 55956000000<br>  Children      Self  Command          Shared Object            Symbol<br>+   11.54%     0.00%  swapper          [kernel.kallsyms]        [k] cpu_startup_entry<br>+   11.42%     0.00%  swapper          [kernel.kallsyms]        [k] default_idle_call<br>+   11.42%     0.00%  swapper          [kernel.kallsyms]        [k] arch_cpu_idle<br>+   11.42%     0.00%  swapper          [kernel.kallsyms]        [k] default_idle<br>+   11.05%    11.05%  swapper          [kernel.kallsyms]        [k] native_safe_halt<br>+    8.69%     0.00%  swapper          [kernel.kallsyms]        [k] start_secondary<br>+    4.36%     4.36%  stress           libc-2.24.so             [.] 0x0000000000036387<br>+    3.44%     0.00%  php-fpm          libc-2.24.so             [.] 0xffff808406d432e1<br>+    3.44%     0.00%  php-fpm          [unknown]                [k] 0x6cb6258d4c544155<br>+    3.43%     3.43%  stress           stress                   [.] 0x0000000000002eff<br>+    3.20%     0.00%  stress           [kernel.kallsyms]        [k] page_fault<br>+    3.20%     0.00%  stress           [kernel.kallsyms]        [k] do_page_fault<br>+    3.15%     0.76%  stress           [kernel.kallsyms]        [k] __do_page_fault</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯嗯 很好的问题。简单的说，swapper跟我们要分析的对象无关。这也是为什么我们不上来就用perf，而是先用其他方法缩小范围。我还会在答疑篇里解释swapper的作用</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-16 23:27:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/18/e4/46d002ed.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>looperX</span>
  </div>
  <div class="_2_QraFYR_0">Day03留言。<br>Github上找到一个链接，里面有各种工具，包括execsnoop。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-04 10:57:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/26/d4/add17f35.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>聰</span>
  </div>
  <div class="_2_QraFYR_0">分析步骤：<br>1，在top中，可见用户态cpu使用率很高，但未见相关有问题的进程；<br>2，尝试pidstat 1 找出问题进程，无果；<br>3，再次top,从load average 可见负载高，shift + f ，然后选择以s (进程状态）排序，按R倒序，可见running的进程主要就是几个stress进程，观察发现，running的stress进程的pid一直在变化；<br>4，watch -d &#39;ps aux | grep stress | grep -v grep&#39; 发现多个stress进程在不断生成，由R变S再变Z（由于watch的时间间隔是1秒，所以只是看到有多个stress进程，有的状态是R有的是S有的是Z，所以我推断其生命周期是如此）<br>5，进程Pid再不断变化，有以下两个原因：<br>5.1，进程在不断崩溃重启，如因为段错误，配置错误等，这时进程在退出后又被监控系统自动重启<br>5.2，这些都是短进程，即在应用内部通过exec调用外部命令。这些命令一般只运行很短的时间就会结束，很难用时间间隔长的工具，如top去发现<br>6，用pstress去找其父进程<br><br>7，发现是php容器，故查看php源码<br># 拷贝源码到本地<br>$ docker cp phpfpm:&#47;app .<br># grep 查找看看是不是有代码在调用 stress 命令<br>$ grep stress -r app<br><br>8，从源代码中找相关字段<br><br>9，stress -t 1 -d 1 是模拟IO压力的，但在之前的top中，未见%iowait异常；<br>10，通过代码中的判断字段，手动赋值访问<br>curl http:&#47;&#47;192.168.0.10:10000?verbose=1<br>Server internal error: Array<br>(<br>    [0] =&gt; stress: info: [19607] dispatching hogs: 0 cpu, 0 io, 0 vm, 1 hdd<br>    [1] =&gt; stress: FAIL: [19608] (563) mkstemp failed: Permission denied<br>    [2] =&gt; stress: FAIL: [19607] (394) &lt;-- worker 19608 returned error 1<br>    [3] =&gt; stress: WARN: [19607] (396) now reaping child worker processes<br>    [4] =&gt; stress: FAIL: [19607] (400) kill error: No such process<br>    [5] =&gt; stress: FAIL: [19607] (451) failed run completed in 0s<br>)<br>11，从这里可以猜测，由于权限错误，大量的stress进程在启动时初始化失败，结合第4点，我认为CPU消耗在进程上下文切换，从vmstat可见cs由问题出现前的100多骤升并稳定维持在3000多<br>[root@master ~]# vmstat 1<br>procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----<br> r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st<br> 9  0      0 6932288   2160 790400    0    0    20    24   44   71  8  5 87  0  0<br> 8  0      0 6932784   2160 790560    0    0     0     0 4027 3512 71 27  3  0  0<br>11  0      0 6934920   2160 790488    0    0     0     0 4170 3646 72 28  1  0  0<br><br>但pidstat -w |grep stress 却未见异常，因为同名进程一直在更换，所以用pidstat -w在这里是不太适用的<br>[root@master ~]# pidstat -w 2 | grep stress<br>09:55:29 AM     1     18583      0.50      0.50  stress<br>09:55:29 AM     1     18584      0.50      0.50  stress<br>09:55:29 AM     1     18586      0.50      0.00  stress<br>。。。<br>请问以上对吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-01 22:05:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKZicicibGibtR5zAia782Ajc5I5BN3F3tjAdlibATIknHv67gbxeH21N7B6vbgwLjYb1miaLKhqicptB5ibYw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>阿飞</span>
  </div>
  <div class="_2_QraFYR_0">关于使用perf工具分析docker容器共用3种可行的办法，详见：https:&#47;&#47;www.cnblogs.com&#47;luoahong&#47;p&#47;10833689.html</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-22 14:51:13</div>
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
  <div class="_2_QraFYR_0">实际生产环境中的进程更多，stress藏在ps中根本不容易发现，pstree的结果也非常大。老师有空讲讲如何找到这些异常进程的方法和灵感。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯嗯，这是一个模块，侧重于CPU的分析，后面还会讲磁盘、网络等等</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-09 13:53:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/61/bc/88a905a5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>亮点</span>
  </div>
  <div class="_2_QraFYR_0">倪老师您好，我在网上看到一个关于iowait指标的解释，非常形象，但不确定是否准确，帮忙鉴别一下，谢谢，链接 http:&#47;&#47;linuxperf.com&#47;?p=33</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-03 19:30:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/54/9a/76c0af70.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>每天晒白牙</span>
  </div>
  <div class="_2_QraFYR_0">【D6打卡】这几天回丈母娘家，没带电脑，没能实战，只能看文章了，回来后要补上</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-03 09:02:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/56/2d/dd3b12b8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>汤🐠🥣昱</span>
  </div>
  <div class="_2_QraFYR_0">1： 对于pidstat,vmstat,top无法定位到问题的时候。<br>2： 可以选择perf record -g 记录。<br>3： 用perf report查看是否可以定位到问题。<br>4： 用pstree | grep [xx],这样定位到具体的调用方法里。<br>5： 用grep [xx] -r [项目文件],找到具体代码位置。<br>6： 查找源码，定位到具体位置，修改。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-14 18:45:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/65/ae/9727318e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Boy-struggle</span>
  </div>
  <div class="_2_QraFYR_0">打卡，对于大数据的系统分析起来还是比较困难的，进程已经达到上千，只能用top去分析</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-04 10:48:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/24/7c/b76ad65d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>每一段路都是一种领悟</span>
  </div>
  <div class="_2_QraFYR_0">今天一个程序负载飙到140，最高点240，我们的服务器没有挂掉，真的是牛逼，另外使用这几天的方法，基本确认了程序的问题，质问开发后，他不好意思的告诉我，io高是因为自己程序偷了懒，好在这次找到证据了，作为以后的分析案例</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 😊</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-24 21:18:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/34/bb/0b971fca.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>walker</span>
  </div>
  <div class="_2_QraFYR_0">execsnoop这个工具在centos里找不到，有类似的代替品吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 点击链接到github上。<br><br>完全一样的工具应该没有，但可以基于内核追踪技术（比如BPF）来自己写一个。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-03 10:43:57</div>
  </div>
</div>
</div>
</li>
</ul>