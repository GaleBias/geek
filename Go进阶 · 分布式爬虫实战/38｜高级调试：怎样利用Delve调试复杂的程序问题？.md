<audio title="38｜高级调试：怎样利用Delve调试复杂的程序问题？" src="https://static001.geekbang.org/resource/audio/63/b2/63f389d04a8a8449216826c13f6169b2.mp3" controls="controls"></audio> 
<p>你好，我是郑建勋。</p><p>工欲善其事，必先利其器。这节课，我们来看看怎么合理地使用调试器让开发事半功倍。调试器能够控制应用程序的执行，它可以让程序在特定的位置暂停并观察当前的状态，还能够控制单步执行代码和指令，以便观察程序的执行分支。</p><p>当我们谈到调试器，一些有经验的开发可能会想到GDB，不过在Go语言中，我们一般会选择使用Delve（dlv）。这不仅因为Delve比 GDB 更了解 Go 运行时、数据结构和表达式，还因为Go中栈扩容等特性会让<a href="https://go.dev/doc/gdb">GDB得到错误的结果</a>。所以这节课，我们就主要来看看如何利用Delve完成Go程序的调试。</p><h2>Delve的内部架构</h2><p>我们先来看看<a href="https://github.com/go-delve/delve">Delve</a>的内部架构。Delve本身也是用Go语言实现的，它的内部可以分为3层。</p><ul>
<li>UI Layer</li>
</ul><p>UI layer 为用户交互层，用于接收用户的输入，解析用户输入的指令。例如打印变量信息时用户需要在交互层输入print a。</p><ul>
<li>Symbolic Layer</li>
</ul><p>Symbolic Layer用于解析用户的输入。例如对于print a这个打印指令，变量a可能是结构体、int等多种类型，Symbolic Layer负责将变量a转化为实际的内存地址和它对应的字节大小，最后通过Target Layer层读取内存数据。同时，Symbolic Layer也会把从Target Layer中读取到的数据解析为对应的结构、行号等信息。</p><!-- [[[read_end]]] --><ul>
<li>Target Layer</li>
</ul><p>Target Layer用于控制程序，它主要是通过调用操作系统的API来实现的。例如在Linux中，Delve会使用ptrace、waitpid、tgkill等操作系统API来读取、修改、追踪内存地址的内容，但是它并不知道具体内容的含义。</p><h2>用 Delve 进行实战</h2><p>简单地了解了Delve的内部架构，下面让我们来使用常见的Delve指令实战一下。首先我们需要安装好Delve。</p><pre><code class="language-plain">$ go install github.com/go-delve/delve/cmd/dlv@latest
</code></pre><p>如果要安装指定的版本，可以用下面的指令。</p><pre><code class="language-plain">$ go install github.com/go-delve/delve/cmd/dlv@v1.7.3
</code></pre><p>以代码v0.3.9为例，程序构建时，指定编译器选项 -gcflags=all=“-N -l”，禁止内联，禁止编译器优化。这有助于我们在使用Delve进行调试时得到更精准的行号等信息。</p><pre><code class="language-plain">debug:
	go build -gcflags=all="-N -l" -ldflags '$(LDFLAGS)' $(BUILD_FLAGS) main.go
</code></pre><p>执行 make debug 完成代码的编译。</p><pre><code class="language-plain">» make debug                                                                                                                  jackson@bogon
go build -gcflags=all="-N -l" -ldflags '-X "github.com/dreamerjackson/crawler/version.BuildTS=2022-12-25 03:33:21" -X "github.com/dreamerjackson/crawler/version.GitHash=6a4e939d8e68f5f29ee9f46bb3dc898157a8ca8e" -X "github.com/dreamerjackson/crawler/version.GitBranch=master" -X "github.com/dreamerjackson/crawler/version.Version=v1.0.0"'  main.go
</code></pre><p>执行 dlv exec 指令启动程序并开始调试执行，执行完毕后会出现如下的(dlv)提示符。</p><pre><code class="language-plain">» sudo dlv exec ./main worker                                                                                                 jackson@bogon
Password:
Type 'help' for list of commands.
(dlv)
</code></pre><p>下面我们来看看在Delve调试中一些常见的命令。</p><ul>
<li><strong>查看帮助信息：help。</strong>当我们记不清楚具体指令的含义的时候，可以执行该指令。</li>
</ul><pre><code class="language-plain">(dlv) help
The following commands are available:
    args ------------------------ Print function arguments.
    break (alias: b) ------------ Sets a breakpoint.
    breakpoints (alias: bp) ----- Print out info for active breakpoints.
    call ------------------------ Resumes process, injecting a function call (EXPERIMENTAL!!!)
    clear ----------------------- Deletes breakpoint.
    clearall -------------------- Deletes multiple breakpoints.
    condition (alias: cond) ----- Set breakpoint condition.
    config ---------------------- Changes configuration parameters.
    continue (alias: c) --------- Run until breakpoint or program termination.
    deferred -------------------- Executes command in the context of a deferred call.
    disassemble (alias: disass) - Disassembler.
    ....
</code></pre><ul>
<li><strong>打断点：break 或者b。</strong>执行该指令会在main函数处打印一个断点。</li>
</ul><pre><code class="language-plain">(dlv) b main.main
Breakpoint 1 set at 0x2089e86 for main.main() ./main.go:8
</code></pre><ul>
<li><strong>继续运行程序：continue 或者c。</strong>程序将一直运行，直到在我们断点处停下来。</li>
</ul><pre><code class="language-plain">(dlv) c
&gt; main.main() ./main.go:8 (hits goroutine(1):1 total:1) (PC: 0x2089e86)
     3: import (
     4:         "github.com/dreamerjackson/crawler/cmd"
     5:         _ "net/http/pprof"
     6: )
     7: 
=&gt;   8: func main() {
     9:         cmd.Execute()
    10: }
</code></pre><ul>
<li><strong>单步执行： n 或 next。</strong> 程序在单步一行代码后将会暂停下来，同时我们还能看到程序当前暂停的位置。</li>
</ul><pre><code class="language-plain">(dlv) n
&gt; main.main() ./main.go:9 (PC: 0x2089e92)
     4:         "github.com/dreamerjackson/crawler/cmd"
     5:         _ "net/http/pprof"
     6: )
     7: 
     8: func main() {
=&gt;   9:         cmd.Execute()
    10: }
</code></pre><ul>
<li><strong>跳进函数中： s 或 step。</strong>这时将进入到调用函数的堆栈中执行。</li>
</ul><pre><code class="language-plain">(dlv) s
&gt; github.com/dreamerjackson/crawler/cmd.Execute() ./cmd/cmd.go:20 (PC: 0x2089d4a)
    15:         Run: func(cmd *cobra.Command, args []string) {
    16:                 version.Printer()
    17:         },
    18: }
    19: 
=&gt;  20: func Execute() {
    21:         var rootCmd = &amp;cobra.Command{Use: "crawler"}
    22:         rootCmd.AddCommand(master.MasterCmd, worker.WorkerCmd, versionCmd)
    23:         rootCmd.Execute()
    24: }
</code></pre><p>接下来我们用b worker.go:135在worker.go文件的135行打上断点。</p><pre><code class="language-plain">(dlv) b worker.go:135
Breakpoint 2 set at 0x2071659 for github.com/dreamerjackson/crawler/cmd/worker.Run() ./cmd/worker/worker.go:135
(dlv) c
{"level":"INFO","ts":"2022-12-26T00:02:57.026+0800","caller":"worker/worker.go:101","msg":"log init end"}
{"level":"INFO","ts":"2022-12-26T00:02:57.029+0800","caller":"worker/worker.go:109","msg":"proxy list: [&lt;http://192.168.0.105:8888&gt; &lt;http://192.168.0.105:8888&gt;] timeout: 3000"}
&gt; github.com/dreamerjackson/crawler/cmd/worker.Run() ./cmd/worker/worker.go:135 (hits goroutine(1):1 total:1) (PC: 0x2071659)
   130:         // init tasks
   131:         var tcfg []spider.TaskConfig
   132:         if err := cfg.Get("Tasks").Scan(&amp;tcfg); err != nil {
   133:                 logger.Error("init seed tasks", zap.Error(err))
   134:         }
=&gt; 135:         seeds := ParseTaskConfig(logger, f, storage, tcfg)
   136: 
   137:         _ = engine.NewEngine(
   138:                 engine.WithFetcher(f),
   139:                 engine.WithLogger(logger),
   140:                 engine.WithWorkCount(5),
</code></pre><ul>
<li><strong>list命令</strong>，可以为我们打印出当前断点处的源代码。</li>
</ul><pre><code class="language-plain">(dlv) list
&gt; github.com/dreamerjackson/crawler/cmd/worker.Run() ./cmd/worker/worker.go:135 (hits goroutine(1):1 total:1) (PC: 0x2071659)
   130:         // init tasks
   131:         var tcfg []spider.TaskConfig
   132:         if err := cfg.Get("Tasks").Scan(&amp;tcfg); err != nil {
   133:                 logger.Error("init seed tasks", zap.Error(err))
   134:         }
=&gt; 135:         seeds := ParseTaskConfig(logger, f, storage, tcfg)
   136: 
   137:         _ = engine.NewEngine(
   138:                 engine.WithFetcher(f),
   139:                 engine.WithLogger(logger),
   140:                 engine.WithWorkCount(5),
</code></pre><ul>
<li><strong>locals命令，</strong>为我们打印出当前所有的局部变量。</li>
</ul><pre><code class="language-plain">(dlv) locals
proxyURLs = []string len: 2, cap: 2, [...]
seeds = []*github.com/dreamerjackson/crawler/spider.Task len: 0, cap: 57, []
cfg = go-micro.dev/v4/config.Config(*go-micro.dev/v4/config.config) 0xc000221508
enc = go-micro.dev/v4/config/encoder.Encoder(github.com/go-micro/plugins/v4/config/encoder/toml.tomlEncoder) {}
err = error nil
f = github.com/dreamerjackson/crawler/spider.Fetcher(*github.com/dreamerjackson/crawler/collect.BrowserFetch) 0xc0002214b8
logText = "debug"
plugin = go.uber.org/zap/zapcore.Core(*go.uber.org/zap/zapcore.ioCore) 0xc000221498
sqlURL = "root:123456@tcp(192.168.0.105:3326)/crawler?charset=utf8"
...
</code></pre><ul>
<li><strong>print或者p命令，</strong>打印出当前变量的值。</li>
</ul><pre><code class="language-plain">(dlv) print proxyURLs
[]string len: 2, cap: 2, [
        "&lt;http://192.168.0.105:8888&gt;",
        "&lt;http://192.168.0.105:8888&gt;",
]
(dlv) p logText
"debug"
</code></pre><ul>
<li><strong>stack命令，</strong>打印出当前函数的堆栈信息，从中我们可以看出函数的调用关系。</li>
</ul><pre><code class="language-plain">(dlv) stack
0  0x0000000002071659 in github.com/dreamerjackson/crawler/cmd/worker.Run
   at ./cmd/worker/worker.go:135
1  0x00000000020702cb in github.com/dreamerjackson/crawler/cmd/worker.glob..func1
   at ./cmd/worker/worker.go:44
2  0x0000000002058734 in github.com/spf13/cobra.(*Command).execute
   at /Users/jackson/go/pkg/mod/github.com/spf13/cobra@v1.6.1/command.go:920
3  0x00000000020596c6 in github.com/spf13/cobra.(*Command).ExecuteC
   at /Users/jackson/go/pkg/mod/github.com/spf13/cobra@v1.6.1/command.go:1044
4  0x0000000002058c8f in github.com/spf13/cobra.(*Command).Execute
   at /Users/jackson/go/pkg/mod/github.com/spf13/cobra@v1.6.1/command.go:968
5  0x0000000002089e5d in github.com/dreamerjackson/crawler/cmd.Execute
   at ./cmd/cmd.go:23
6  0x0000000002089e97 in main.main
   at ./main.go:9
7  0x000000000103e478 in runtime.main
   at /usr/local/opt/go/libexec/src/runtime/proc.go:250
8  0x000000000106fee1 in runtime.goexit
   at /usr/local/opt/go/libexec/src/runtime/asm_amd64.s:1571
</code></pre><ul>
<li><strong>frame命令，</strong>可以让我们在堆栈之间做切换。在下面这个例子中，我们输入frame 1，就会切换到当前函数的调用方，再输入frame 0即可切换回去。</li>
</ul><pre><code class="language-plain">(dlv) frame 1
&gt; github.com/dreamerjackson/crawler/cmd/worker.Run() ./cmd/worker/worker.go:135 (hits goroutine(1):1 total:1) (PC: 0x2071659)
Frame 1: ./cmd/worker/worker.go:44 (PC: 20702cb)
    39:         Use:   "worker",
    40:         Short: "run worker service.",
    41:         Long:  "run worker service.",
    42:         Args:  cobra.NoArgs,
    43:         Run: func(cmd *cobra.Command, args []string) {
=&gt;  44:                 Run()
    45:         },
    46: }
    47: 
    48: func init() {
    49:         WorkerCmd.Flags().StringVar(
</code></pre><ul>
<li><strong>breakpoints命令，</strong>打印出当前的断点。</li>
</ul><pre><code class="language-plain">(dlv) breakpoints
Breakpoint 1 at 0x2089e86 for main.main() ./main.go:8 (1)
Breakpoint 2 at 0x2071659 for github.com/dreamerjackson/crawler/cmd/worker.Run() ./cmd/worker/worker.go:135 (1)
</code></pre><ul>
<li><strong>clear 命令，</strong>清除断点。下面这个例子就可以清除序号为1的断点。</li>
</ul><pre><code class="language-plain">(dlv) clear 1
Breakpoint 1 cleared at 0x2089e86 for main.main() ./main.go:8
</code></pre><ul>
<li><strong>goroutines命令，</strong>显示当前时刻所有的协程。</li>
</ul><pre><code class="language-plain">(dlv) goroutines
* Goroutine 1 - User: ./cmd/worker/worker.go:135 github.com/dreamerjackson/crawler/cmd/worker.Run (0x2071659) (thread 8118196)
  Goroutine 2 - User: /usr/local/opt/go/libexec/src/runtime/proc.go:362 runtime.gopark (0x103e892)
  Goroutine 3 - User: /usr/local/opt/go/libexec/src/runtime/proc.go:362 runtime.gopark (0x103e892)
  Goroutine 4 - User: /usr/local/opt/go/libexec/src/runtime/proc.go:362 runtime.gopark (0x103e892)
  Goroutine 5 - User: /usr/local/opt/go/libexec/src/runtime/proc.go:362 runtime.gopark (0x103e892)
  Goroutine 6 - User: /Users/jackson/go/pkg/mod/github.com/patrickmn/go-cache@v2.1.0+incompatible/cache.go:1079 github.com/patrickmn/go-cache.(*janitor).Run (0x1a88f05)
  Goroutine 7 - User: /Users/jackson/go/pkg/mod/go-micro.dev/v4@v4.9.0/config/loader/memory/memory.go:401 go-micro.dev/v4/config/loader/memory.(*watcher).Next (0x1b7af28)
</code></pre><p>goroutine还可以实现协程的切换。例如下面这个例子，我们执行goroutine 2将协程切换到了协程2，并打印出协程2的堆栈信息。接着执行goroutine 1切换回去。</p><pre><code class="language-plain">(dlv) goroutine 2
Switched from 1 to 2 (thread 8118196)
(dlv) stack
0  0x000000000103e892 in runtime.gopark
   at /usr/local/opt/go/libexec/src/runtime/proc.go:362
1  0x000000000103e92a in runtime.goparkunlock
   at /usr/local/opt/go/libexec/src/runtime/proc.go:367
2  0x000000000103e6c5 in runtime.forcegchelper
   at /usr/local/opt/go/libexec/src/runtime/proc.go:301
3  0x000000000106fee1 in runtime.goexit
   at /usr/local/opt/go/libexec/src/runtime/asm_amd64.s:1571
</code></pre><p>还有一些更高级的调试指令，例如，<strong>disassemble</strong> 可以打印出当前的汇编代码。</p><pre><code class="language-plain">(dlv) disassemble
TEXT github.com/dreamerjackson/crawler/cmd/worker.Run(SB) /Users/jackson/career/crawler/cmd/worker/worker.go
        worker.go:66    0x2070500       4c8da42408f9ffff                lea r12, ptr [rsp+0xfffff908]
        worker.go:66    0x2070508       4d3b6610                        cmp r12, qword ptr [r14+0x10]
        worker.go:66    0x207050c       0f8635180000                    jbe 0x2071d47
        worker.go:66    0x2070512       4881ec78070000                  sub rsp, 0x778
        worker.go:66    0x2070519       4889ac2470070000                mov qword ptr [rsp+0x770], rbp
        worker.go:66    0x2070521       488dac2470070000                lea rbp, ptr [rsp+0x770]
        worker.go:68    0x2070529       488d0518252f00                  lea rax, ptr [rip+0x2f2518]
</code></pre><p>另外，虽然dlv通常是在开发环境中使用的，但是有时它仍然能够用在线上环境中，例如可以在服务完全无响应时帮助我们排查问题。举个例子，假设我们的代码中有一段逻辑Bug，导致服务陷入了长时间的for循环中，这个时候要排查原因我们就可以使用dlv了。</p><pre><code class="language-plain">...
count := 0
	for {
		count++
		fmt.Println("count", count)
	}
</code></pre><p>对于一个运行中的程序，要进行调试，我们可以使用<strong>dlv attach指令</strong>，其后跟程序的进程号。而要想查找到程序的进程号，我们可以用如下指令。本例中程序的进程号为75296。</p><pre><code class="language-plain">» ps -ef | grep './main worker'                                                                                               jackson@bogon
  501 75296 91914   0 11:20PM ttys003    0:00.31 ./main worker
</code></pre><p>接着，执行dlv attach进行调试。注意，这时程序会完全暂停。</p><pre><code class="language-plain">» dlv attach 75296                                                                                                            jackson@bogon
Type 'help' for list of commands.
(dlv)
</code></pre><p>接下来，我们可以查看当前协程所处的位置，找到可能造成程序卡死的协程。</p><pre><code class="language-plain">(dlv) goroutines
  Goroutine 1 - User: /usr/local/opt/go/libexec/src/runtime/sys_darwin.go:23 syscall.syscall (0x106629f)
  Goroutine 2 - User: /usr/local/opt/go/libexec/src/runtime/proc.go:362 runtime.gopark (0x1039336)
  Goroutine 3 - User: /usr/local/opt/go/libexec/src/runtime/proc.go:362 runtime.gopark (0x1039336)
  Goroutine 4 - User: /Users/jackson/go/pkg/mod/github.com/patrickmn/go-cache@v2.1.0+incompatible/cache.go:1079 github.com/patrickmn/go-cache.(*janitor).Run (0x16c2c65)
  Goroutine 5 - User: /Users/jackson/go/pkg/mod/go-micro.dev/v4@v4.9.0/config/loader/memory/memory.go:401 go-micro.dev/v4/config/loader/memory.(*watcher).Next (0x1758bb2)
  Goroutine 6 - User: /Users/jackson/go/pkg/mod/github.com/patrickmn/go-cache@v2.1.0+incompatible/cache.go:1079 github.com/patrickmn/go-cache.(*janitor).Run (0x16c2c65)
  Goroutine 7 - User: /usr/local/opt/go/libexec/src/runtime/netpoll.go:302 internal/poll.runtime_pollWait (0x1063be9)
</code></pre><p>当我们切换到 goroutine 1 查看堆栈信息时可以发现，由于我们调用了fmt函数，所以执行了系统调用函数。继续查看调用fmt函数的位置是 ./cmd/worker/worker.go:84，结合代码就可以轻松地发现这个逻辑Bug了。</p><pre><code class="language-plain">(dlv) goroutine 1
Switched from 0 to 1 (thread 9333412)
(dlv) stack
 0  0x00000000010677e0 in runtime.systemstack_switch
    at /usr/local/opt/go/libexec/src/runtime/asm_amd64.s:436
 1  0x00000000010563e6 in runtime.libcCall
    at /usr/local/opt/go/libexec/src/runtime/sys_libc.go:48
 2  0x000000000106629f in syscall.syscall
    at /usr/local/opt/go/libexec/src/runtime/sys_darwin.go:23
 3  0x000000000107ce09 in syscall.write
    at /usr/local/opt/go/libexec/src/syscall/zsyscall_darwin_amd64.go:1653
 4  0x00000000010d188e in internal/poll.ignoringEINTRIO
    at /usr/local/opt/go/libexec/src/syscall/syscall_unix.go:216
 5  0x00000000010d188e in syscall.Write
    at /usr/local/opt/go/libexec/src/internal/poll/fd_unix.go:383
 6  0x00000000010d188e in internal/poll.(*FD).Write
    at /usr/local/opt/go/libexec/src/internal/poll/fd_unix.go:794
 7  0x00000000010d93c5 in os.(*File).write
    at /usr/local/opt/go/libexec/src/os/file_posix.go:48
 8  0x00000000010d93c5 in os.(*File).Write
    at /usr/local/opt/go/libexec/src/os/file.go:176
 9  0x00000000010e2775 in fmt.Fprintln
    at /usr/local/opt/go/libexec/src/fmt/print.go:265
10  0x0000000001a4e329 in fmt.Println
    at /usr/local/opt/go/libexec/src/fmt/print.go:274
11  0x0000000001a4e329 in github.com/dreamerjackson/crawler/cmd/worker.Run
    at ./cmd/worker/worker.go:84
12  0x0000000001a4e097 in github.com/dreamerjackson/crawler/cmd/worker.glob..func1
    at ./cmd/worker/worker.go:45
</code></pre><h2>用 Goland 进行调试</h2><p>Delve虽然强大，但是在平时的开发过程中，我们更倾向于使用Goland和VSCode来进行调试。</p><p>Goland和VSCode借助了Delve的能力，但是它提供了可视化的交互方式，可以让我们更加方便快捷地进行调试，下面我以 Goland 为例来说明一下它的用法。</p><p>使用 Goland 进行调试的第一步是设置构建的相关配置。如下图所示，我们设置了构建的目录位置和程序运行时的参数。我们启动Master程序的调试。</p><p><img src="https://static001.geekbang.org/resource/image/b5/ed/b579f2595d3b3347fc1f82d4bf15a1ed.png?wh=1920x1554" alt="图片"></p><p>第二步，在代码左边适当的位置加入断点。</p><p>第三步，点击左上方的调试按钮开始调试。这时程序会开始运行，直到遇上断点才会停下来。</p><p><img src="https://static001.geekbang.org/resource/image/2d/f6/2d8d713c977d18485dd36beaaf72c7f6.png?wh=1920x839" alt="图片"></p><p>当程序在断点处停下来之后，在Goland界面下方会显示出当前局部变量的值和当前的堆栈信息，我们还可以切换到不同的协程和不同的堆栈。还可以使用各种按钮让程序继续执行、单步执行、跳入函数、跳出函数等。点击变量的右键还可以修改变量的值。</p><p><img src="https://static001.geekbang.org/resource/image/b5/4a/b5daa65dc8bfe82cbe8e71000463e24a.png?wh=1920x622" alt="图片"></p><h2>用Goland+Delve进行远程调试</h2><p>接下来我们来看看如何让Goland与Delve配合在一起，对Go程序进行远程调试。我们需要远程调试程序的场景有很多，举几个例子。</p><ul>
<li>本地机器配置跟不上，调试起来太卡。</li>
<li>远程服务器有更加完备的上下游环境、配置文件、硬件（例如GPU）、特殊的依赖库（Linux与Windows）。</li>
<li>需要在特定环境复现问题。</li>
</ul><p>利用Goland完成远程调试的优势也有很多。</p><ul>
<li>可视化调试界面，减少心智负担。</li>
<li>本地机器负载小。</li>
<li>调试时间更快，减少繁琐的日志打印过程。</li>
</ul><p>Goland结合dlv的远程调试可以分为下面几步。</p><ol>
<li>将代码同步到远程机器，保证当前代码版本与远程机器代码版本相同。</li>
<li>在远程机器上安装最新的dlv。</li>
<li>在远程机器上构建程序，并且禁止编译器的优化与内联，如下所示。</li>
</ol><pre><code class="language-plain">go build -o crawler -gcflags=all="-N -l" main.go
</code></pre><ol start="4">
<li>执行dlv exec，这时程序不会执行，而会监听2345端口，等待远程调试客户端发过来的信号。</li>
</ol><pre><code class="language-plain">dlv --listen=:2345 --headless=true --api-version=2 --accept-multiclient --check-go-version=false exec ./crawler worker
</code></pre><ol start="5">
<li>在本地Goland中配置远程连接地址。点击Goland右上角的edit Configurations，选择Go Remote，设置远程服务器监听的IP地址与端口。</li>
</ol><p><img src="https://static001.geekbang.org/resource/image/af/09/afab744c3e5115929c07de51d3379209.png?wh=1920x1240" alt="图片"></p><p>接下来我们就可以和在本地一样进行代码调试了。</p><h2>总结</h2><p>这节课，我们介绍了如何使用Delve调试器来调试Go语言程序。Delve调试器是专门为Go语言设计的，相比于其他调试器，它更懂Go语言的运行时与数据结构。</p><p>学习Delve调试器的最好方式就是练习各个指令的含义。当然我们在平时的开发过程中，一般会选择界面化的调试方式。Goland与VSCode底层仍然是使用了Delve的能力，但是可视化的本地调试和远程调试能起到事半功倍的效果。 在一些特殊的线上环境，我们无法使用可视化界面时，可以直接使用Delve调试器attach程序。</p><h2>课后题</h2><p>学完这节课，给你留两道思考题。</p><ol>
<li>在介绍Go语言的调试时，我们说在很多场景下 Delve 相对于GDB具有优势。那么有没有什么场景是用GDB比Delve更合适的呢？</li>
<li>Delve能够用到线上的环境中吗？</li>
</ol><p>欢迎你在留言区与我交流讨论，我们下节课见。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/9c/fb/7fe6df5b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>陈卧虫</span>
  </div>
  <div class="_2_QraFYR_0">如果在远程容器中开发，如何用goland 连接远程容器中的dlv呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 首先是把容器的端口映射到宿主机上，其次调整一下docker启动时候的安全参数</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-06 08:21:40</div>
  </div>
</div>
</div>
</li>
</ul>