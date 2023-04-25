<audio title="29 _ 案例篇：Redis响应严重延迟，如何解决？" src="https://static001.geekbang.org/resource/audio/d8/cc/d81f154d511dcc2c4927472ed3d589cc.mp3" controls="controls"></audio> 
<p>你好，我是倪朋飞。</p><p>上一节，我们一起分析了一个基于 MySQL 的商品搜索案例，先来回顾一下。</p><p>在访问商品搜索接口时，我们发现接口的响应特别慢。通过对系统 CPU、内存和磁盘 I/O 等资源使用情况的分析，我们发现这时出现了磁盘的 I/O 瓶颈，并且正是案例应用导致的。</p><p>接着，我们借助 pidstat，发现罪魁祸首是 mysqld 进程。我们又通过 strace、lsof，找出了 mysqld 正在读的文件。根据文件的名字和路径，我们找出了 mysqld 正在操作的数据库和数据表。综合这些信息，我们猜测这是一个没利用索引导致的慢查询问题。</p><p>为了验证猜测，我们到 MySQL 命令行终端，使用数据库分析工具发现，案例应用访问的字段果然没有索引。既然猜测是正确的，那增加索引后，问题就自然解决了。</p><p>从这个案例你会发现，MySQL 的 MyISAM 引擎，主要依赖系统缓存加速磁盘 I/O 的访问。可如果系统中还有其他应用同时运行， MyISAM 引擎很难充分利用系统缓存。缓存可能会被其他应用程序占用，甚至被清理掉。</p><p>所以，一般我并不建议，把应用程序的性能优化完全建立在系统缓存上。最好能在应用程序的内部分配内存，构建完全自主控制的缓存；或者使用第三方的缓存应用，比如 Memcached、Redis 等。</p><!-- [[[read_end]]] --><p>Redis 是最常用的键值存储系统之一，常用作数据库、高速缓存和消息队列代理等。Redis 基于内存来存储数据，不过，为了保证在服务器异常时数据不丢失，很多情况下，我们要为它配置持久化，而这就可能会引发磁盘 I/O 的性能问题。</p><p>今天，我就带你一起来分析一个利用 Redis 作为缓存的案例。这同样是一个基于 Python Flask 的应用程序，它提供了一个 查询缓存的接口，但接口的响应时间比较长，并不能满足线上系统的要求。</p><p>非常感谢携程系统研发部资深后端工程师董国星，帮助提供了今天的案例。</p><h2>案例准备</h2><p>本次案例还是基于 Ubuntu 18.04，同样适用于其他的 Linux 系统。我使用的案例环境如下所示：</p><ul>
<li>
<p>机器配置：2 CPU，8GB 内存</p>
</li>
<li>
<p>预先安装 docker、sysstat 、git、make 等工具，如 apt install <a href="http://docker.io">docker.io</a> sysstat</p>
</li>
</ul><p>今天的案例由 Python应用+Redis 两部分组成。其中，Python 应用是一个基于 Flask 的应用，它会利用 Redis ，来管理应用程序的缓存，并对外提供三个 HTTP 接口：</p><ul>
<li>
<p>/：返回 hello redis；</p>
</li>
<li>
<p>/init/<num>：插入指定数量的缓存数据，如果不指定数量，默认的是 5000 条；</num></p>
</li>
<li>
<p>缓存的键格式为 uuid:<uuid></uuid></p>
</li>
<li>
<p>缓存的值为 good、bad 或 normal 三者之一</p>
</li>
<li>
<p>/get_cache/&lt;type_name&gt;：查询指定值的缓存数据，并返回处理时间。其中，type_name 参数只支持 good, bad 和 normal（也就是找出具有相同 value 的 key 列表）。</p>
</li>
</ul><p>由于应用比较多，为了方便你运行，我把它们打包成了两个 Docker 镜像，并推送到了 <a href="https://github.com/feiskyer/linux-perf-examples/tree/master/redis-slow">Github</a> 上。这样你就只需要运行几条命令，就可以启动了。</p><p>今天的案例需要两台虚拟机，其中一台用作案例分析的目标机器，运行 Flask 应用，它的 IP 地址是 192.168.0.10；而另一台作为客户端，请求缓存查询接口。我画了一张图来表示它们的关系。</p><p><img src="https://static001.geekbang.org/resource/image/c8/87/c8e0ca06d70a1c7f1520d103a3edfc87.png?wh=770*519" alt=""></p><p>接下来，打开两个终端，分别 SSH 登录到这两台虚拟机中，并在第一台虚拟机中安装上述工具。</p><p>跟以前一样，案例中所有命令都默认以 root 用户运行，如果你是用普通用户身份登陆系统，请运行 sudo su root 命令切换到 root 用户。</p><p>到这里，准备工作就完成了。接下来，我们正式进入操作环节。</p><h2>案例分析</h2><p>首先，我们在第一个终端中，执行下面的命令，运行本次案例要分析的目标应用。正常情况下，你应该可以看到下面的输出：</p><pre><code># 注意下面的随机字符串是容器ID，每次运行均会不同，并且你不需要关注它
$ docker run --name=redis -itd -p 10000:80 feisky/redis-server
ec41cb9e4dd5cb7079e1d9f72b7cee7de67278dbd3bd0956b4c0846bff211803
$ docker run --name=app --network=container:redis -itd feisky/redis-app
2c54eb252d0552448320d9155a2618b799a1e71d7289ec7277a61e72a9de5fd0
</code></pre><p>然后，再运行 docker ps 命令，确认两个容器都处于运行（Up）状态：</p><pre><code>$ docker ps
CONTAINER ID        IMAGE                 COMMAND                  CREATED             STATUS              PORTS                             NAMES
2c54eb252d05        feisky/redis-app      &quot;python /app.py&quot;         48 seconds ago      Up 47 seconds                                         app
ec41cb9e4dd5        feisky/redis-server   &quot;docker-entrypoint.s…&quot;   49 seconds ago      Up 48 seconds       6379/tcp, 0.0.0.0:10000-&gt;80/tcp   redis

</code></pre><p>今天的应用在 10000 端口监听，所以你可以通过 <a href="http://192.168.0.10:10000">http://192.168.0.10:10000</a>  ，来访问前面提到的三个接口。</p><p>比如，我们切换到第二个终端，使用 curl 工具，访问应用首页。如果你看到 <code>hello redis</code> 的输出，说明应用正常启动：</p><pre><code>$ curl http://192.168.0.10:10000/
hello redis
</code></pre><p>接下来，继续在终端二中，执行下面的 curl 命令，来调用应用的 /init 接口，初始化 Redis 缓存，并且插入 5000 条缓存信息。这个过程比较慢，比如我的机器就花了十几分钟时间。耐心等一会儿后，你会看到下面这行输出：</p><pre><code># 案例插入5000条数据，在实践时可以根据磁盘的类型适当调整，比如使用SSD时可以调大，而HDD可以适当调小
$ curl http://192.168.0.10:10000/init/5000
{&quot;elapsed_seconds&quot;:30.26814079284668,&quot;keys_initialized&quot;:5000}
</code></pre><p>继续执行下一个命令，访问应用的缓存查询接口。如果一切正常，你会看到如下输出：</p><pre><code>$ curl http://192.168.0.10:10000/get_cache
{&quot;count&quot;:1677,&quot;data&quot;:[&quot;d97662fa-06ac-11e9-92c7-0242ac110002&quot;,...],&quot;elapsed_seconds&quot;:10.545469760894775,&quot;type&quot;:&quot;good&quot;}
</code></pre><p>我们看到，这个接口调用居然要花 10 秒！这么长的响应时间，显然不能满足实际的应用需求。</p><p>到底出了什么问题呢？我们还是要用前面学过的性能工具和原理，来找到这个瓶颈。</p><p>不过别急，同样为了避免分析过程中客户端的请求结束，在进行性能分析前，我们先要把 curl 命令放到一个循环里来执行。你可以在终端二中，继续执行下面的命令：</p><pre><code>$ while true; do curl http://192.168.0.10:10000/get_cache; done
</code></pre><p>接下来，再重新回到终端一，查找接口响应慢的“病因”。</p><p>最近几个案例的现象都是响应很慢，这种情况下，我们自然先会怀疑，是不是系统资源出现了瓶颈。所以，先观察 CPU、内存和磁盘 I/O 等的使用情况肯定不会错。</p><p>我们先在终端一中执行 top 命令，分析系统的 CPU 使用情况：</p><pre><code>$ top
top - 12:46:18 up 11 days,  8:49,  1 user,  load average: 1.36, 1.36, 1.04
Tasks: 137 total,   1 running,  79 sleeping,   0 stopped,   0 zombie
%Cpu0  :  6.0 us,  2.7 sy,  0.0 ni,  5.7 id, 84.7 wa,  0.0 hi,  1.0 si,  0.0 st
%Cpu1  :  1.0 us,  3.0 sy,  0.0 ni, 94.7 id,  0.0 wa,  0.0 hi,  1.3 si,  0.0 st
KiB Mem :  8169300 total,  7342244 free,   432912 used,   394144 buff/cache
KiB Swap:        0 total,        0 free,        0 used.  7478748 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
 9181 root      20   0  193004  27304   8716 S   8.6  0.3   0:07.15 python
 9085 systemd+  20   0   28352   9760   1860 D   5.0  0.1   0:04.34 redis-server
  368 root      20   0       0      0      0 D   1.0  0.0   0:33.88 jbd2/sda1-8
  149 root       0 -20       0      0      0 I   0.3  0.0   0:10.63 kworker/0:1H
 1549 root      20   0  236716  24576   9864 S   0.3  0.3  91:37.30 python3
</code></pre><p>观察 top 的输出可以发现，CPU0 的 iowait 比较高，已经达到了 84%；而各个进程的 CPU 使用率都不太高，最高的 python 和 redis-server ，也分别只有 8% 和 5%。再看内存，总内存 8GB，剩余内存还有 7GB多，显然内存也没啥问题。</p><p>综合top的信息，最有嫌疑的就是 iowait。所以，接下来还是要继续分析，是不是 I/O 问题。</p><p>还在第一个终端中，先按下 Ctrl+C，停止 top 命令；然后，执行下面的 iostat 命令，查看有没有 I/O 性能问题：</p><pre><code>$ iostat -d -x 1
Device            r/s     w/s     rkB/s     wkB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz  svctm  %util
...
sda              0.00  492.00      0.00   2672.00     0.00   176.00   0.00  26.35    0.00    1.76   0.00     0.00     5.43   0.00   0.00
</code></pre><p>观察 iostat 的输出，我们发现，磁盘 sda 每秒的写数据（wkB/s）为 2.5MB，I/O 使用率（%util）是 0。看来，虽然有些 I/O操作，但并没导致磁盘的 I/O 瓶颈。</p><p>排查一圈儿下来，CPU和内存使用没问题，I/O 也没有瓶颈，接下来好像就没啥分析方向了？</p><p>碰到这种情况，还是那句话，反思一下，是不是又漏掉什么有用线索了。你可以先自己思考一下，从分析对象（案例应用）、系统原理和性能工具这三个方向下功夫，回忆它们的特性，查找现象的异常，再继续往下走。</p><p>回想一下，今天的案例问题是从 Redis 缓存中查询数据慢。对查询来说，对应的 I/O 应该是磁盘的读操作，但刚才我们用 iostat 看到的却是写操作。虽说 I/O 本身并没有性能瓶颈，但这里的磁盘写也是比较奇怪的。为什么会有磁盘写呢？那我们就得知道，到底是哪个进程在写磁盘。</p><p>要知道 I/O请求来自哪些进程，还是要靠我们的老朋友 pidstat。在终端一中运行下面的 pidstat 命令，观察进程的 I/O 情况：</p><pre><code>$ pidstat -d 1
12:49:35      UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command
12:49:36        0       368      0.00     16.00      0.00      86  jbd2/sda1-8
12:49:36      100      9085      0.00    636.00      0.00       1  redis-server
</code></pre><p>从 pidstat 的输出，我们看到，I/O 最多的进程是 PID 为 9085 的 redis-server，并且它也刚好是在写磁盘。这说明，确实是 redis-server 在进行磁盘写。</p><p>当然，光找到读写磁盘的进程还不够，我们还要再用 strace+lsof 组合，看看 redis-server 到底在写什么。</p><p>接下来，还是在终端一中，执行 strace 命令，并且指定 redis-server 的进程号 9085：</p><pre><code># -f表示跟踪子进程和子线程，-T表示显示系统调用的时长，-tt表示显示跟踪时间
$ strace -f -T -tt -p 9085
[pid  9085] 14:20:16.826131 epoll_pwait(5, [{EPOLLIN, {u32=8, u64=8}}], 10128, 65, NULL, 8) = 1 &lt;0.000055&gt;
[pid  9085] 14:20:16.826301 read(8, &quot;*2\r\n$3\r\nGET\r\n$41\r\nuuid:5b2e76cc-&quot;..., 16384) = 61 &lt;0.000071&gt;
[pid  9085] 14:20:16.826477 read(3, 0x7fff366a5747, 1) = -1 EAGAIN (Resource temporarily unavailable) &lt;0.000063&gt;
[pid  9085] 14:20:16.826645 write(8, &quot;$3\r\nbad\r\n&quot;, 9) = 9 &lt;0.000173&gt;
[pid  9085] 14:20:16.826907 epoll_pwait(5, [{EPOLLIN, {u32=8, u64=8}}], 10128, 65, NULL, 8) = 1 &lt;0.000032&gt;
[pid  9085] 14:20:16.827030 read(8, &quot;*2\r\n$3\r\nGET\r\n$41\r\nuuid:55862ada-&quot;..., 16384) = 61 &lt;0.000044&gt;
[pid  9085] 14:20:16.827149 read(3, 0x7fff366a5747, 1) = -1 EAGAIN (Resource temporarily unavailable) &lt;0.000043&gt;
[pid  9085] 14:20:16.827285 write(8, &quot;$3\r\nbad\r\n&quot;, 9) = 9 &lt;0.000141&gt;
[pid  9085] 14:20:16.827514 epoll_pwait(5, [{EPOLLIN, {u32=8, u64=8}}], 10128, 64, NULL, 8) = 1 &lt;0.000049&gt;
[pid  9085] 14:20:16.827641 read(8, &quot;*2\r\n$3\r\nGET\r\n$41\r\nuuid:53522908-&quot;..., 16384) = 61 &lt;0.000043&gt;
[pid  9085] 14:20:16.827784 read(3, 0x7fff366a5747, 1) = -1 EAGAIN (Resource temporarily unavailable) &lt;0.000034&gt;
[pid  9085] 14:20:16.827945 write(8, &quot;$4\r\ngood\r\n&quot;, 10) = 10 &lt;0.000288&gt;
[pid  9085] 14:20:16.828339 epoll_pwait(5, [{EPOLLIN, {u32=8, u64=8}}], 10128, 63, NULL, 8) = 1 &lt;0.000057&gt;
[pid  9085] 14:20:16.828486 read(8, &quot;*3\r\n$4\r\nSADD\r\n$4\r\ngood\r\n$36\r\n535&quot;..., 16384) = 67 &lt;0.000040&gt;
[pid  9085] 14:20:16.828623 read(3, 0x7fff366a5747, 1) = -1 EAGAIN (Resource temporarily unavailable) &lt;0.000052&gt;
[pid  9085] 14:20:16.828760 write(7, &quot;*3\r\n$4\r\nSADD\r\n$4\r\ngood\r\n$36\r\n535&quot;..., 67) = 67 &lt;0.000060&gt;
[pid  9085] 14:20:16.828970 fdatasync(7) = 0 &lt;0.005415&gt;
[pid  9085] 14:20:16.834493 write(8, &quot;:1\r\n&quot;, 4) = 4 &lt;0.000250&gt;
</code></pre><p>观察一会儿，有没有发现什么有趣的现象呢？</p><p>事实上，从系统调用来看， epoll_pwait、read、write、fdatasync 这些系统调用都比较频繁。那么，刚才观察到的写磁盘，应该就是 write 或者 fdatasync 导致的了。</p><p>接着再来运行 lsof 命令，找出这些系统调用的操作对象：</p><pre><code>$ lsof -p 9085
redis-ser 9085 systemd-network    3r     FIFO   0,12      0t0 15447970 pipe
redis-ser 9085 systemd-network    4w     FIFO   0,12      0t0 15447970 pipe
redis-ser 9085 systemd-network    5u  a_inode   0,13        0    10179 [eventpoll]
redis-ser 9085 systemd-network    6u     sock    0,9      0t0 15447972 protocol: TCP
redis-ser 9085 systemd-network    7w      REG    8,1  8830146  2838532 /data/appendonly.aof
redis-ser 9085 systemd-network    8u     sock    0,9      0t0 15448709 protocol: TCP
</code></pre><p>现在你会发现，描述符编号为 3 的是一个 pipe 管道，5 号是 eventpoll，7 号是一个普通文件，而 8 号是一个 TCP socket。</p><p>结合磁盘写的现象，我们知道，只有 7 号普通文件才会产生磁盘写，而它操作的文件路径是 /data/appendonly.aof，相应的系统调用包括 write 和 fdatasync。</p><p>如果你对 Redis 的持久化配置比较熟，看到这个文件路径以及 fdatasync 的系统调用，你应该能想到，这对应着正是 Redis 持久化配置中的 appendonly 和 appendfsync 选项。很可能是因为它们的配置不合理，导致磁盘写比较多。</p><p>接下来就验证一下这个猜测，我们可以通过 Redis 的命令行工具，查询这两个选项的配置。</p><p>继续在终端一中，运行下面的命令，查询 appendonly 和 appendfsync 的配置：</p><pre><code>$ docker exec -it redis redis-cli config get 'append*'
1) &quot;appendfsync&quot;
2) &quot;always&quot;
3) &quot;appendonly&quot;
4) &quot;yes&quot;
</code></pre><p>从这个结果你可以发现，appendfsync 配置的是 always，而 appendonly 配置的是 yes。这两个选项的详细含义，你可以从 <a href="https://redis.io/topics/persistence">Redis Persistence</a> 的文档中查到，这里我做一下简单介绍。</p><p>Redis 提供了两种数据持久化的方式，分别是快照和追加文件。</p><p><strong>快照方式</strong>，会按照指定的时间间隔，生成数据的快照，并且保存到磁盘文件中。为了避免阻塞主进程，Redis 还会 fork 出一个子进程，来负责快照的保存。这种方式的性能好，无论是备份还是恢复，都比追加文件好很多。</p><p>不过，它的缺点也很明显。在数据量大时，fork子进程需要用到比较大的内存，保存数据也很耗时。所以，你需要设置一个比较长的时间间隔来应对，比如至少5分钟。这样，如果发生故障，你丢失的就是几分钟的数据。</p><p><strong>追加文件</strong>，则是用在文件末尾追加记录的方式，对 Redis 写入的数据，依次进行持久化，所以它的持久化也更安全。</p><p>此外，它还提供了一个用 appendfsync 选项设置 fsync 的策略，确保写入的数据都落到磁盘中，具体选项包括 always、everysec、no 等。</p><ul>
<li>
<p>always表示，每个操作都会执行一次 fsync，是最为安全的方式；</p>
</li>
<li>
<p>everysec表示，每秒钟调用一次 fsync ，这样可以保证即使是最坏情况下，也只丢失1秒的数据；</p>
</li>
<li>
<p>而 no 表示交给操作系统来处理。</p>
</li>
</ul><p>回忆一下我们刚刚看到的配置，appendfsync 配置的是 always，意味着每次写数据时，都会调用一次 fsync，从而造成比较大的磁盘 I/O 压力。</p><p>当然，你还可以用 strace ，观察这个系统调用的执行情况。比如通过 -e 选项指定 fdatasync 后，你就会得到下面的结果：</p><pre><code>$ strace -f -p 9085 -T -tt -e fdatasync
strace: Process 9085 attached with 4 threads
[pid  9085] 14:22:52.013547 fdatasync(7) = 0 &lt;0.007112&gt;
[pid  9085] 14:22:52.022467 fdatasync(7) = 0 &lt;0.008572&gt;
[pid  9085] 14:22:52.032223 fdatasync(7) = 0 &lt;0.006769&gt;
...
[pid  9085] 14:22:52.139629 fdatasync(7) = 0 &lt;0.008183&gt;
</code></pre><p>从这里你可以看到，每隔 10ms 左右，就会有一次 fdatasync 调用，并且每次调用本身也要消耗 7~8ms。</p><p>不管哪种方式，都可以验证我们的猜想，配置确实不合理。这样，我们就找出了 Redis 正在进行写入的文件，也知道了产生大量 I/O 的原因。</p><p>不过，回到最初的疑问，为什么查询时会有磁盘写呢？按理来说不应该只有数据的读取吗？这就需要我们再来审查一下 strace -f -T -tt -p 9085 的结果。</p><pre><code>read(8, &quot;*2\r\n$3\r\nGET\r\n$41\r\nuuid:53522908-&quot;..., 16384)
write(8, &quot;$4\r\ngood\r\n&quot;, 10)
read(8, &quot;*3\r\n$4\r\nSADD\r\n$4\r\ngood\r\n$36\r\n535&quot;..., 16384)
write(7, &quot;*3\r\n$4\r\nSADD\r\n$4\r\ngood\r\n$36\r\n535&quot;..., 67)
write(8, &quot;:1\r\n&quot;, 4)
</code></pre><p>细心的你应该记得，根据 lsof 的分析，文件描述符编号为 7 的是一个普通文件 /data/appendonly.aof，而编号为 8 的是 TCP socket。而观察上面的内容，8 号对应的 TCP 读写，是一个标准的“请求-响应”格式，即：</p><ul>
<li>
<p>从 socket 读取 GET uuid:53522908-… 后，响应 good；</p>
</li>
<li>
<p>再从 socket 读取 SADD good 535… 后，响应 1。</p>
</li>
</ul><p>对 Redis 来说，SADD是一个写操作，所以 Redis 还会把它保存到用于持久化的 appendonly.aof 文件中。</p><p>观察更多的 strace 结果，你会发现，每当 GET 返回 good 时，随后都会有一个 SADD 操作，这也就导致了，明明是查询接口，Redis 却有大量的磁盘写。</p><p>到这里，我们就找出了 Redis 写磁盘的原因。不过，在下最终结论前，我们还是要确认一下，8 号 TCP socket 对应的 Redis 客户端，到底是不是我们的案例应用。</p><p>我们可以给 lsof 命令加上 -i 选项，找出 TCP socket 对应的 TCP 连接信息。不过，由于 Redis 和 Python 应用都在容器中运行，我们需要进入容器的网络命名空间内部，才能看到完整的 TCP 连接。</p><blockquote>
<p>注意：下面的命令用到的 <a href="http://man7.org/linux/man-pages/man1/nsenter.1.html">nsenter</a>  工具，可以进入容器命名空间。如果你的系统没有安装，请运行下面命令安装 nsenter：<br>
docker run --rm -v /usr/local/bin:/target jpetazzo/nsenter</p>
</blockquote><p>还是在终端一中，运行下面的命令：</p><pre><code># 由于这两个容器共享同一个网络命名空间，所以我们只需要进入app的网络命名空间即可
$ PID=$(docker inspect --format {{.State.Pid}} app)
# -i表示显示网络套接字信息
$ nsenter --target $PID --net -- lsof -i
COMMAND    PID            USER   FD   TYPE   DEVICE SIZE/OFF NODE NAME
redis-ser 9085 systemd-network    6u  IPv4 15447972      0t0  TCP localhost:6379 (LISTEN)
redis-ser 9085 systemd-network    8u  IPv4 15448709      0t0  TCP localhost:6379-&gt;localhost:32996 (ESTABLISHED)
python    9181            root    3u  IPv4 15448677      0t0  TCP *:http (LISTEN)
python    9181            root    5u  IPv4 15449632      0t0  TCP localhost:32996-&gt;localhost:6379 (ESTABLISHED)

</code></pre><p>这次我们可以看到，redis-server 的 8 号文件描述符，对应 TCP 连接 localhost:6379-&gt;localhost:32996。其中， localhost:6379 是 redis-server 自己的监听端口，自然 localhost:32996 就是 redis 的客户端。再观察最后一行，localhost:32996 对应的，正是我们的 Python 应用程序（进程号为 9181）。</p><p>历经各种波折，我们总算找出了 Redis 响应延迟的潜在原因。总结一下，我们找到两个问题。</p><p>第一个问题，Redis 配置的 appendfsync 是 always，这就导致 Redis 每次的写操作，都会触发 fdatasync 系统调用。今天的案例，没必要用这么高频的同步写，使用默认的 1s 时间间隔，就足够了。</p><p>第二个问题，Python 应用在查询接口中会调用 Redis 的 SADD 命令，这很可能是不合理使用缓存导致的。</p><p>对于第一个配置问题，我们可以执行下面的命令，把 appendfsync 改成 everysec：</p><pre><code>$ docker exec -it redis redis-cli config set appendfsync everysec
OK
</code></pre><p>改完后，切换到终端二中查看，你会发现，现在的请求时间，已经缩短到了 0.9s：</p><pre><code>{..., &quot;elapsed_seconds&quot;:0.9368953704833984,&quot;type&quot;:&quot;good&quot;}
</code></pre><p>而第二个问题，就要查看应用的源码了。点击 <a href="https://github.com/feiskyer/linux-perf-examples/blob/master/redis-slow/app.py">Github</a>  ，你就可以查看案例应用的源代码：</p><pre><code>def get_cache(type_name):
    '''handler for /get_cache'''
    for key in redis_client.scan_iter(&quot;uuid:*&quot;):
        value = redis_client.get(key)
        if value == type_name:
            redis_client.sadd(type_name, key[5:])
    data = list(redis_client.smembers(type_name))
    redis_client.delete(type_name)
    return jsonify({&quot;type&quot;: type_name, 'count': len(data), 'data': data})
</code></pre><p>果然，Python 应用把 Redis 当成临时空间，用来存储查询过程中找到的数据。不过我们知道，这些数据放内存中就可以了，完全没必要再通过网络调用存储到 Redis 中。</p><p>基于这个思路，我把修改后的代码也推送到了相同的源码文件中，你可以通过 <a href="http://192.168.0.10:10000/get_cache_data">http://192.168.0.10:10000/get_cache_data</a> 这个接口来访问它。</p><p>我们切换到终端二，按 Ctrl+C 停止之前的 curl 命令；然后执行下面的 curl 命令，调用 <a href="http://192.168.0.10:10000/get_cache_data">http://192.168.0.10:10000/get_cache_data</a>  新接口：</p><pre><code>$ while true; do curl http://192.168.0.10:10000/get_cache_data; done
{...,&quot;elapsed_seconds&quot;:0.16034674644470215,&quot;type&quot;:&quot;good&quot;}
</code></pre><p>你可以发现，解决第二个问题后，新接口的性能又有了进一步的提升，从刚才的 0.9s ，再次缩短成了不到 0.2s。</p><p>当然，案例最后，不要忘记清理案例应用。你可以切换到终端一中，执行下面的命令进行清理：</p><pre><code>$ docker rm -f app redis
</code></pre><h2>小结</h2><p>今天我带你一起分析了一个 Redis 缓存的案例。</p><p>我们先用 top、iostat ，分析了系统的 CPU 、内存和磁盘使用情况，不过却发现，系统资源并没有出现瓶颈。这个时候想要进一步分析的话，该从哪个方向着手呢？</p><p>通过今天的案例你会发现，为了进一步分析，就需要你对系统和应用程序的工作原理有一定的了解。</p><p>比如，今天的案例中，虽然磁盘 I/O 并没有出现瓶颈，但从 Redis 的原理来说，查询缓存时不应该出现大量的磁盘 I/O 写操作。</p><p>顺着这个思路，我们继续借助 pidstat、strace、lsof、nsenter 等一系列的工具，找出了两个潜在问题，一个是 Redis 的不合理配置，另一个是 Python 应用对 Redis 的滥用。找到瓶颈后，相应的优化工作自然就比较轻松了。</p><h2>思考</h2><p>最后给你留一个思考题。从上一节 MySQL 到今天 Redis 的案例分析，你有没有发现 I/O 性能问题的分析规律呢？如果你有任何想法或心得，都可以记录下来。</p><p>当然，这两个案例这并不能涵盖所有的 I/O 性能问题。你在实际工作中，还碰到过哪些 I/O 性能问题吗？你又是怎么分析的呢？</p><p>欢迎在留言区和我讨论，也欢迎把这篇文章分享给你的同事、朋友。我们一起在实战中演练，在交流中进步。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://wx.qlogo.cn/mmopen/vi_32/PiajxSqBRaEKQMM4m7NHuicr55aRiblTSEWIYe0QqbpyHweaoAbG7j2v7UUElqqeP3Ihrm3UfDPDRb1Hv8LvPwXqA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ninuxer</span>
  </div>
  <div class="_2_QraFYR_0">打卡day30<br>io问题一般都是先top发展iowait比较高，然后iostat看是哪个进程比较高，然后再通过strace，lsof找出进程在读写的具体文件，然后对应的分析</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-25 08:21:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/16/af/bada0f59.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>李博</span>
  </div>
  <div class="_2_QraFYR_0">老师，有个问题咨询下，为什么top显示 iowait比较高，但是使用iostat却发现io的使用率并不高那？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: iowait不代表磁盘I&#47;O存在瓶颈，只是代表CPU上I&#47;O操作的时间占用的百分比。假如这时候没有其他进程在运行，那么很小的I&#47;O就会导致iowait升高</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-25 14:32:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/fa/b4/6892eabe.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_33409b</span>
  </div>
  <div class="_2_QraFYR_0">打卡day30 <br>IO性能问题首先可以通过top 查看机器的整体负载情况，一般会出现CPU 的iowait 较高的现象<br>然后使用 pidstat -d 1 找到读写磁盘较高的进程<br>然后通过 strace -f -TT 进行跟踪，查看系统读写调用的频率和时间<br>通过lsof 找到 strace 中的文件描述符对应的文件 opensnoop可以找到对应的问题位置<br>推测 对应的问题，mysql 案例中的大量读，可能是因为没有建立索引导致的全表查询，从而形成了慢查询的现象。redis 中则是因为 备份文件设置的不合理导致的每次查询都会写磁盘。当然不同的问题还需要结合对应的情况进行分析<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-06 10:20:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/9a/64/ad837224.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Christmas</span>
  </div>
  <div class="_2_QraFYR_0">进程iowait高，磁盘iowait不高，说明是单个进程使用了一些blocking的磁盘打开方式，比如每次都fsync</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-25 09:57:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/2f/c5/aaacb98f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>yungoo</span>
  </div>
  <div class="_2_QraFYR_0">nsenter报了loadlocale.c assertion设置<br><br>export LC_ALL=C<br><br>即可</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢分享</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-17 23:51:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/33/33/30fb3ac3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>____的我</span>
  </div>
  <div class="_2_QraFYR_0">前段时间刚找到一个由于内存数据被交换到swap文件中导致内存数据遍历效率变低的问题 问题定位过程是使用pidstat命令发现进程cpu使用率变低 mpstat命令观察到系统iowait升高 由此怀疑跟io有什么关系 perf命令观察发现内存数据遍历过程中swap相关调用时间占比有点异常 然后使用pidstat命令+r参数 也观察到进程在那段时间主缺页中断升高 由此确定问题<br><br>老师的课程非常有用 多多向您学习 希望老师能多分享一些定位网络延迟问题的方法 不仅仅局限在ping探测</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢分享性能排查的经验👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-11 18:54:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/eb/bc/6ccac4bb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>武文文武</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，一直在听您的视频，发现您用了很多的小工具来检查系统性能指标，而我们公司使用nmon工具，就能一次性将几乎所有常用的指标全部获取到了，而且还能拿到历史数据，请问我们用nmon是否就能在大部分情况下取到了您说的top pidstat等工具呢，如果不可以那您能说说原因吗？非常感谢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，实际使用中，使用类似nmon这种监控系统是更推荐的做法。不过，在监控系统的间隔时间不够小，或者指标不够全的时候，还是需要到系统上去抓取更多的细节</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-28 12:28:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/j0gBKF8EKfQ4TjoSTeRPoWia56RMCevYDauMGS9iaw4TnpC0dp2Viaict75T8TebKzAcUruWiazkyiaCQPqKZxar2M6Q/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>利俊杰</span>
  </div>
  <div class="_2_QraFYR_0">nsenter --target $PID -- lsof -i<br>执行失败，提示：loadlocale.c:130: _nl_intern_locale_data: Assertion `cnt &lt; (sizeof (_nl_value_type_LC_COLLATE) &#47; sizeof (_nl_value_type_LC_COLLATE[0]))&#39; failed<br>可以参考下https:&#47;&#47;stackoverflow.com&#47;questions&#47;37121895&#47;yocto-build-loadlocale-c-130<br>配置<br>LANG=&#47;usr&#47;lib&#47;locale&#47;en_US<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢分享</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-26 00:18:40</div>
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
  <div class="_2_QraFYR_0">分析规律就是：<br>* 根据top发现cpu0的iowait偏高<br>* iostat -d -x 1观察系统磁盘IO情况有无异常<br>* pidstat -d 1观察进程的IO其概况<br>* 用 strace+lsof 组合：strace -f -T -tt -p pid  和  lsof -p pid观察系统调用情况定位磁盘IO发生的现场</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-13 14:34:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>从远方过来</span>
  </div>
  <div class="_2_QraFYR_0">老师，你这几篇文章大量使用了strace，它的负载不是很高么？在生产上可以使用么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，strace会影响性能，所以一般不使用它来做监控。但是在已经出现严重性能问题的时候，使用它来排查问题还是可以接受的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-08 08:56:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJKkThulMFj6MNsgk6Tps4Ll1pzrricPDqAyuFkmFYMDcPgUbGuUnMYACIICRTRPWLPtib2z6V6XdBg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>开始懂了</span>
  </div>
  <div class="_2_QraFYR_0"> curl http:&#47;&#47;10.39.25.7:10000&#47;init&#47;get_cache<br>&lt;!DOCTYPE HTML PUBLIC &quot;-&#47;&#47;W3C&#47;&#47;DTD HTML 3.2 Final&#47;&#47;EN&quot;&gt;<br>&lt;title&gt;500 Internal Server Error&lt;&#47;title&gt;<br>&lt;h1&gt;Internal Server Error&lt;&#47;h1&gt;<br>&lt;p&gt;The server encountered an internal error and was unable to complete your request.  Either the server is overloaded or there is an error in the application.&lt;&#47;p&gt;</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: &#47;init&#47; 后面需要一个数字</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-25 10:43:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/da/ec/779c1a78.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>往事随风，顺其自然</span>
  </div>
  <div class="_2_QraFYR_0">git clone https:&#47;&#47;github.com&#47;feiskyer&#47;linux-perf-examples&#47;tree&#47;master&#47;redis-slow<br>Initialized empty Git repository in &#47;root&#47;redis-slow&#47;.git&#47;<br>error: The requested URL returned error: 403 Forbidden while accessing https:&#47;&#47;github.com&#47;feiskyer&#47;linux-perf-examples&#47;tree&#47;master&#47;redis-slow&#47;info&#47;refs<br><br>代码怎么克隆不下来</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: clone要指定代码仓库的路径，而不是子目录:<br><br>git clone https:&#47;&#47;github.com&#47;feiskyer&#47;linux-perf-examples</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-25 20:51:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/e6/ee/e3c4c9b3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Cranliu</span>
  </div>
  <div class="_2_QraFYR_0">top、iostat、pidstat、strace，以及对应用程序的了解，MySQL、Redis本质上也是一款应用程序。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-25 07:39:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2d/d7/0b/e92b2c13.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>trprebel</span>
  </div>
  <div class="_2_QraFYR_0">老师，过了这么长时间来提问不知道您是否还能看到，这个案例的问题我在学习过程中觉得分析过程有些蹊跷，当系统出现异常时<br>1、我们使用top命令发现cpu有一个核使用率非常高，然后有84.7%是iowait，这是我们得到的第一个线索<br>2、然后使用iostat，pidstat，lsof等工具查看，这些工具查看完，如您所说，只能说明有io操作，并不能说明是有io瓶颈，并且这些指标也没有跟第一个线索关联起来，或者相互佐证<br>3、最后我们通过猜测读请求里面不应该有大量写操作来确定写操作的异常，虽然说定位并解决了问题，但感觉解决方案与问题和线索并没有直接关系，实际生产环境中不可能只有读操作，也有不会有“读操作里面不应该有大量写操作”的推断，那如果生产环境中我们应该如何分析呢<br>感觉是不是应该继续从这两个方向继续分析<br>1、问题：RT时间过长，进程&#47;线程在干什么，导致一直无法响应<br>2、线索：是哪个进程&#47;线程导致iowait如此高，以及这个进程&#47;线程在做什么<br>最后才定位到redis的aof操作</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-17 22:17:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/dc/22/bb8e93b4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>晴天雨伞</span>
  </div>
  <div class="_2_QraFYR_0">nsenter这个命令太棒了。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-19 22:50:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/f6/24/547439f1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ghostwritten</span>
  </div>
  <div class="_2_QraFYR_0">top-------&gt;iowait<br>iostat -d -x 1-----&gt;w<br>pidstat  -d 1<br>strace -f -t -TT &lt;pid&gt;<br>lsof -p &lt;pid&gt;<br>nsenter --target $PID --net -- lsof -i<br>config set appendfsync everysec<br>python: <br>redis_client.sadd(type_name, key[5:])   把 Redis 当成临时空间，用来存储查询过程中找到的数</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-19 17:03:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/26/bc/2c/963688bb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>noisyes</span>
  </div>
  <div class="_2_QraFYR_0">为什么磁盘性能没有达到性能瓶颈，接口返回得这么慢呢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-24 10:28:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1c/a3/f5/458d277b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>伟啊伟</span>
  </div>
  <div class="_2_QraFYR_0">有个问题，生产环境中往往不可能只有一类操作。像文中发展cpu io都没问题的时候，如果不是提前知道应用应该只有读操作（生产不可能），如何能快速定位到可疑点呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-26 08:39:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/64/05/6989dce6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我来也</span>
  </div>
  <div class="_2_QraFYR_0">[D29打卡]<br>感觉每次分析的套路都差不多.<br>1.用top查看指标,发现 [系统] 有i&#47;o瓶颈 或者 cpu瓶颈.<br>2.使用iostat辅助看下磁盘i&#47;o读写速度和大小等指标.<br>3.用pidstat判断是哪个 [进程] 导致的. 既可以看进程各线程的cpu中断数,也可以看磁盘io数.<br>4.用strace追踪进程及各线程的 [系统调用].(以前经常到这里就知道了是操作的什么文件)<br>5.继续用lsof查看该进程打开的 [文件] .linux下一切皆文件,则可以查看的东西就很多很多了.连进程保持的socket等信息也一目了然.<br>6.本例因为用到了容器,所以用到了nsenter进入容器的网络命名空间,查看对应的socket信息.<br>7.根据第4.5步获取的信息,找源码或看系统配置.确定问题,做出调整.然后收工.</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果能一个套路查遍所有问题就好了😓我相信很多人都期待这样</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-26 13:31:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_b3b8da</span>
  </div>
  <div class="_2_QraFYR_0">我发现除了排查网络，排查cpu、内存、io、套路就是打开top，然后去看哪些指标，然后通过排除法慢慢缩小范围定位到对应的进程，首先使用pstree看看那个程序的pid是父进程还是子进程，然后通过strace -T -f -tt -e trace=all -p 1779 -o error.txt 万能命令 查看进程的详细问题，如果显示失败，那么就是排查是不是短时进程，如果不是那么用perf来查看最后还是优化代码去</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-07 17:09:44</div>
  </div>
</div>
</div>
</li>
</ul>