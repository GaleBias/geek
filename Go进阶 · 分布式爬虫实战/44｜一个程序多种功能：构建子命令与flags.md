<audio title="44｜一个程序多种功能：构建子命令与flags" src="https://static001.geekbang.org/resource/audio/8e/86/8e52206f5ce6yy7e6a78ba818d0c2786.mp3" controls="controls"></audio> 
<p>你好，我是郑建勋。</p><p>之前，我们介绍了Worker的开发以及代码的测试，但之前的程序其实还是单机执行的。接下来让我们打开分布式开发的大门，一起看看如何开发Master服务，实现任务的调度与故障容错。</p><p>考虑到Worker和Master有许多可以共用的代码，并且关系紧密，所以我们可以将Worker与Master放到同一个代码仓库里。</p><h2>Cobra实现命令行工具</h2><p>代码放置在同一个仓库后，我们遇到了一个新的问题。代码中只有一个main函数，该如何构建两个程序呢？其实，我们可以参考Linux中的一些命令行工具，或者Go这个二进制文件的处理方式。例如，执行go fmt代表执行代码格式化程序，执行go doc代表执行文档注释程序。</p><p>在本项目中，我们使用 <a href="http://xn--github-hz8ig3bo82im51b.com/spf13/cobra">github.com/spf13/cobra</a> 库提供的能力构建命令行应用程序。命令行应用程序通常接受各种输入作为参数，这些参数也被称为子命令，例如go fmt中的fmt和go doc中的doc。同时，命令行应用程序也提供了一些选项或运行参数来控制程序的不同行为，这些选项通常被称为flags。</p><h3>Cobra实例代码</h3><p>怎么用Cobra来实现命令行工具呢？我们先来看一个简单的例子。在下面这个例子中，cmdPrint、cmdEcho、cmdTimes 表示我们将向程序加入的3个子命令。</p><!-- [[[read_end]]] --><pre><code class="language-plain">package main

import (
	"fmt"
	"strings"

	"github.com/spf13/cobra"
)

func main() {
	var echoTimes int

	var cmdPrint = &amp;cobra.Command{
		Use:   "c [string to print]",
		Short: "Print anything to the screen",
		Long: `print is for printing anything back to the screen.
For many years people have printed back to the screen.`,
		Args: cobra.MinimumNArgs(1),
		Run: func(cmd *cobra.Command, args []string) {
			fmt.Println("Print: " + strings.Join(args, " "))
		},
	}

	var cmdEcho = &amp;cobra.Command{
		Use:   "echo [string to echo]",
		Short: "Echo anything to the screen",
		Long: `echo is for echoing anything back.
Echo works a lot like print, except it has a child command.`,
		Args: cobra.MinimumNArgs(1),
		Run: func(cmd *cobra.Command, args []string) {
			fmt.Println("Echo: " + strings.Join(args, " "))
		},
	}

	var cmdTimes = &amp;cobra.Command{
		Use:   "times [string to echo]",
		Short: "Echo anything to the screen more times",
		Long: `echo things multiple times back to the user by providing
a count and a string.`,
		Args: cobra.MinimumNArgs(1),
		Run: func(cmd *cobra.Command, args []string) {
			for i := 0; i &lt; echoTimes; i++ {
				fmt.Println("Echo: " + strings.Join(args, " "))
			}
		},
	}

	cmdTimes.Flags().IntVarP(&amp;echoTimes, "times", "t", 1, "times to echo the input")

	var rootCmd = &amp;cobra.Command{Use: "app"}
	rootCmd.AddCommand(cmdPrint, cmdEcho)
	cmdEcho.AddCommand(cmdTimes)
	rootCmd.Execute()
}
</code></pre><p>以cmdPrint变量为例，它定义了一个子命令。cobra.Command中的第一个字段Use定义了子命令的名字为print；Short和Long描述了子命令的使用方法；Args为子命令需要传入的参数，在这里 <code>cobra.MinimumNArgs(1)</code> 表示至少需要传入一个参数；Run为该子命令要执行的入口函数。</p><p>rootCmd 为程序的根命令，在这里命名为app。AddCommand方法会为命令添加子命令。 例如，rootCmd.AddCommand(cmdPrint, cmdEcho)表示为根命令添加了两个子命令cmdPrint与cmdEcho。而cmdTimes命令为cmdEcho的子命令。</p><p>接下来，我们执行上面的程序，会发现出现了一连串的文字。这是Cobra自动为我们生成的帮助文档，非常清晰。帮助文档中显示了我们当前程序有3个子命令echo、help与print。</p><pre><code class="language-plain">» go build app.go
» ./app -h
Usage:
  app [command]

Available Commands:
  echo        Echo anything to the screen
  help        Help about any command
  print       Print anything to the screen

Flags:
  -h, --help   help for app

Use "app [command] --help" for more information about a command.

</code></pre><p>接下来，我们输入子命令echo，发现依然无法正确地执行并打印出新的帮助文档。帮助文档中提示，我们echo必须要传递一个启动参数。</p><pre><code class="language-plain">» ./app echo 
Error: requires at least 1 arg(s), only received 0
Usage:
  app echo [string to echo] [flags]
  app echo [command]

Available Commands:
  times       Echo anything to the screen more times

Flags:
  -h, --help   help for echo

Use "app echo [command] --help" for more information about a command.
</code></pre><p>正确的执行方式如下。在这里，我们的echo子命令模拟了Linux中的echo指令，打印出了我们输入的信息。</p><pre><code class="language-plain">» ./app echo hello world
Echo: hello world
</code></pre><p>由于我们还为echo添加了一个子命令times，因此我们可以方便地使用它。另外我们会看到子命令times绑定了一个flags，名字是times，缩写为t。</p><pre><code class="language-plain">	cmdTimes.Flags().IntVarP(&amp;echoTimes, "times", "t", 1, "times to echo the input")
</code></pre><p>因此，我们可以用下面的方式执行times子命令，-t 这个flag则可以控制打印文本的次数。</p><pre><code class="language-plain">» ./app echo times hello-world  -t=3 
Echo: hello-world
Echo: hello-world
Echo: hello-world
</code></pre><p>接下来，让我们在项目中使用Cobra。</p><p>在这里，我们遵循Cobra给出的组织代码的推荐目录结构。在最外层main.go的main函数中，只包含一个简单清晰的cmd.Execute()函数调用。实际的Worker与Master子命令则放置到了cmd包中。</p><pre><code class="language-plain">package main

import (
	"github.com/dreamerjackson/crawler/cmd"
)

func main() {
	cmd.Execute()
}
</code></pre><h3>Worker子命令</h3><p>在cmd.go中，Execute函数添加了Worker、Master、Version 这三个子命令，他们都不需要添加运行参数。Worker子命令最终会调用worker.Run(), 和之前一样运行GRPC与HTTP服务。我们只是将之前main.go中的Worker代码迁移到了cmd/worker下。</p><pre><code class="language-plain">// cmd.go
package cmd

import (
	"github.com/dreamerjackson/crawler/cmd/master"
	"github.com/dreamerjackson/crawler/cmd/worker"
	"github.com/dreamerjackson/crawler/version"
	"github.com/spf13/cobra"
)

var workerCmd = &amp;cobra.Command{
	Use:   "worker",
	Short: "run worker service.",
	Long:  "run worker service.",
	Args:  cobra.NoArgs,
	Run: func(cmd *cobra.Command, args []string) {
		worker.Run()
	},
}

var masterCmd = &amp;cobra.Command{
	Use:   "master",
	Short: "run master service.",
	Long:  "run master service.",
	Args:  cobra.NoArgs,
	Run: func(cmd *cobra.Command, args []string) {
		master.Run()
	},
}

var versionCmd = &amp;cobra.Command{
	Use:   "version",
	Short: "print version.",
	Long:  "print version.",
	Args:  cobra.NoArgs,
	Run: func(cmd *cobra.Command, args []string) {
		version.Printer()
	},
}

func Execute() {
	var rootCmd = &amp;cobra.Command{Use: "crawler"}
	rootCmd.AddCommand(masterCmd, workerCmd, versionCmd)
	rootCmd.Execute()
}
</code></pre><p>接着运行go run main.go worker，可以看到Worker程序已经正常地运行了。</p><pre><code class="language-plain">» go run main.go worker                                                                                                      jackson@bogon
{"level":"INFO","ts":"2022-12-10T18:07:20.615+0800","caller":"worker/worker.go:63","msg":"log init end"}
{"level":"INFO","ts":"2022-12-10T18:07:20.615+0800","caller":"worker/worker.go:71","msg":"proxy list: [&lt;http://127.0.0.1:8888&gt; &lt;http://127.0.0.1:8888&gt;] timeout: 3000"}
{"level":"ERROR","ts":"2022-12-10T18:07:21.050+0800","caller":"engine/schedule.go:258","msg":"can not find preset tasks","task name":"xxx"}
{"level":"DEBUG","ts":"2022-12-10T18:07:21.050+0800","caller":"worker/worker.go:114","msg":"grpc server config,{GRPCListenAddress::9090 HTTPListenAddress::8080 ID:1 RegistryAddress::2379 RegisterTTL:60 RegisterInterval:15 Name:go.micro.server.worker ClientTimeOut:10}"}
{"level":"DEBUG","ts":"2022-12-10T18:07:21.052+0800","caller":"worker/worker.go:188","msg":"start http server listening on :8080 proxy to grpc server;:9090"}
2022-12-10 18:07:21  file=worker/worker.go:161 level=info Starting [service] go.micro.server.worker
2022-12-10 18:07:21  file=v4@v4.9.0/service.go:96 level=info Server [grpc] Listening on [::]:9090
2022-12-10 18:07:21  file=grpc@v1.2.0/grpc.go:913 level=info Registry [etcd] Registering node: go.micro.server.worker-1
</code></pre><h3>Master子命令</h3><p>我们再来看看怎么书写Master程序。cmd/master包用于启动Master程序。和Worker代码非常类似，Master也需要启动GRPC服务和HTTP服务，但是和Worker不同的是，Master服务的配置文件参数需要做相应的修改。如下，我们增加了Master的服务配置。</p><pre><code class="language-plain">// config.toml
[MasterServer]
HTTPListenAddress = ":8081"
GRPCListenAddress = ":9091"
ID = "1"
RegistryAddress = ":2379"
RegisterTTL = 60
RegisterInterval = 15
ClientTimeOut   = 10
Name = "go.micro.server.master"
</code></pre><p>接着执行go run main.go master，可以看到Master服务已经正常地运行了。</p><pre><code class="language-plain">» go run main.go master                                                                                                      jackson@bogon
{"level":"INFO","ts":"2022-12-10T18:03:21.986+0800","caller":"master/master.go:55","msg":"log init end"}
hello master
{"level":"DEBUG","ts":"2022-12-10T18:03:21.986+0800","caller":"master/master.go:67","msg":"grpc server config,{GRPCListenAddress::9091 HTTPListenAddress::8081 ID:1 RegistryAddress::2379 RegisterTTL:60 RegisterInterval:15 Name:go.micro.server.master ClientTimeOut:10}"}
{"level":"DEBUG","ts":"2022-12-10T18:03:21.988+0800","caller":"master/master.go:141","msg":"start master http server listening on :8081 proxy to grpc server;:9091"}
2022-12-10 18:03:21  file=master/master.go:114 level=info Starting [service] go.micro.server.master
2022-12-10 18:03:21  file=v4@v4.9.0/service.go:96 level=info Server [grpc] Listening on [::]:9091
2022-12-10 18:03:21  file=grpc@v1.2.0/grpc.go:913 level=info Registry [etcd] Registering node: go.micro.server.master-1
</code></pre><h3>Version子命令</h3><p>接下来我们来看看Version子命令，该命令主要用于打印程序的版本号。我们将打印版本的功能从main.go迁移到version/version.go中。同时，我们在Makefile中构建程序时的编译时选项 <code>ldflags</code> 也需要进行一些调整。如下所示，我们将版本信息注入到了version包的全局变量中。</p><pre><code class="language-plain">// Makefile
LDFLAGS = -X "github.com/dreamerjackson/crawler/version.BuildTS=$(shell date -u '+%Y-%m-%d %I:%M:%S')"
LDFLAGS += -X "github.com/dreamerjackson/crawler/version.GitHash=$(shell git rev-parse HEAD)"
LDFLAGS += -X "github.com/dreamerjackson/crawler/version.GitBranch=$(shell git rev-parse --abbrev-ref HEAD)"
LDFLAGS += -X "github.com/dreamerjackson/crawler/version.Version=${VERSION}"

build:
	go build -ldflags '$(LDFLAGS)' $(BUILD_FLAGS) main.go
</code></pre><p>执行 make build构建程序，然后运行./main version 即可打印出程序的详细版本信息。</p><pre><code class="language-plain">» make build                                                                                                                 jackson@bogon
go build -ldflags '-X "github.com/dreamerjackson/crawler/version.BuildTS=2022-12-10 10:25:17" -X "github.com/dreamerjackson/crawler/version.GitHash=c841af5deb497745d1ae39d3f565579344950777" -X "github.com/dreamerjackson/crawler/version.GitBranch=HEAD" -X "github.com/dreamerjackson/crawler/version.Version=v1.0.0"'  main.go
» ./main version                                                                                                             jackson@bogon
Version:           v1.0.0-c841af5
Git Branch:        HEAD
Git Commit:        c841af5deb497745d1ae39d3f565579344950777
Build Time (UTC):  2022-12-10 10:25:17
</code></pre><p>此外，运行./main -h 还可以看到Cobra自动生成的帮助文档。</p><pre><code class="language-plain">» ./main -h                                                                                                                  jackson@bogon
Usage:
  crawler [command]

Available Commands:
  completion  Generate the autocompletion script for the specified shell
  help        Help about any command
  master      run master service.
  version     print version.
  worker      run worker service.

Flags:
  -h, --help   help for crawler

Use "crawler [command] --help" for more information about a command.
</code></pre><p>这节课我们先把框架搭建起来，后续我们还会具体实现Master的功能。这节课的代码我放在了<a href="https://github.com/dreamerjackson/crawler">v0.3.4</a>分支，你可以打开链接查看。</p><h2>flags控制程序行为</h2><p>刚才，我们都是将一些通用的配置写到配置文件中的。不过很快我们会发现一个问题，如果我们想在同一台机器上运行多个Worker或Master程序，就会发生端口冲突，导致程序异常退出。</p><pre><code class="language-plain">» go run main.go master                                                                                                      jackson@bogon
{"level":"INFO","ts":"2022-12-10T18:37:26.318+0800","caller":"master/master.go:55","msg":"log init end"}
{"level":"DEBUG","ts":"2022-12-10T18:37:26.318+0800","caller":"master/master.go:67","msg":"grpc server config,{GRPCListenAddress::9091 HTTPListenAddress::8081 ID:1 RegistryAddress::2379 RegisterTTL:60 RegisterInterval:15 Name:go.micro.server.master ClientTimeOut:10}"}
{"level":"DEBUG","ts":"2022-12-10T18:37:26.320+0800","caller":"master/master.go:141","msg":"start master http server listening on :8081 proxy to grpc server;:9091"}
{"level":"FATAL","ts":"2022-12-10T18:37:26.320+0800","caller":"master/master.go:143","msg":"http listenAndServe failed","error":"listen tcp :8081: bind: address already in use","stacktrace":"github.com/dreamerjackson/crawler/cmd/master.RunHTTPServer\\n\\t/Users/jackson/career/crawler/cmd/master/master.go:143"}
</code></pre><p>要解决这一问题，我们可以为不同的程序指定不同的配置文件，或者我们也可以先修改我们的配置文件再运行，但这些做法都非常繁琐。这时我们就可以借助flags来解决这类问题了。</p><p>如下所示，我们将Master的ID、监听的HTTP地址与GRPC地址作为flags，并将flags与子命令master绑定在一起。这时，我们可以手动传递运行程序时的flags，并将flags的值设置到全局变量masterID、HTTPListenAddress与GRPCListenAddress中。这样，我们就能够比较方便地为不同的程序设置不同的运行参数了。</p><pre><code class="language-plain">var MasterCmd = &amp;cobra.Command{
	Use:   "master",
	Short: "run master service.",
	Long:  "run master service.",
	Args:  cobra.NoArgs,
	Run: func(cmd *cobra.Command, args []string) {
		Run()
	},
}

func init() {
	MasterCmd.Flags().StringVar(
		&amp;masterID, "id", "1", "set master id")
	MasterCmd.Flags().StringVar(
		&amp;HTTPListenAddress, "http", ":8081", "set HTTP listen address")

	MasterCmd.Flags().StringVar(
		&amp;GRPCListenAddress, "grpc", ":9091", "set GRPC listen address")
}

var masterID string
var HTTPListenAddress string
var GRPCListenAddress string
</code></pre><p>接下来，通过flags中，我们为不同的Master服务设置不同的HTTP监听地址与GRPC监听地址。</p><p>现在，我们就可以轻松地运行多个Master服务，不必担心端口冲突了。</p><pre><code class="language-plain">// master 2
» ./main master --id=2 --http=:8081  --grpc=:9091
//master 3
» ./main master --id=3 --http=:8082  --grpc=:9092
</code></pre><h2>总结</h2><p>总结一下。这节课，为了灵活地运行不同的程序与功能，我们使用了Cobra包构建命令行程序。</p><p>Cobra提供了推荐的项目组织结构，在main函数中有一个清晰的cmd.Execute()函数调用，并把相关子命令放置到了cmd包中。通过Cobra，我们灵活地构建了子命令和flags。子命令帮助我们将Worker与Master放置到了同一个仓库中，快速地搭建起了Master的框架。而flags帮助我们设置了程序不同的运行参数，避免了在本地的端口冲突。</p><p>下一节课，我们还将看到如何书写Master服务，完成服务的监听与选主。</p><h2>课后题</h2><p>学完这节课，给你留一道课后题。</p><p>你认为，应该在什么场景下使用子命令，什么场景下使用flags，又在什么场景下使用环境变量呢？</p><p>欢迎你在留言区与我交流讨论，我们下节课见。</p>
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
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83ep075ibtmxMf3eOYlBJ96CE9TEelLUwePaLqp8M75gWHEcM3za0voylA0oe9y3NiaboPB891rypRt7w/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>shuff1e</span>
  </div>
  <div class="_2_QraFYR_0">这是想到哪讲到哪么？课程大纲上44节不是讲微服务框架与协议的么？怎么又忽然来讲cobra？pflag？这种基础的工具放在前面讲会不会更好一些？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 之前主要是在开发Worker的功能，这一章开始讲解Master服务了。 因为有了两个服务放在同一代码中的需求，所以介绍了命令行的工具。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-19 09:41:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/9c/fb/7fe6df5b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>陈卧虫</span>
  </div>
  <div class="_2_QraFYR_0">正好在写一个命令行工具，今天就用上了，但是遇到了一个问题，我需要实现交互式的，能多次用户输入，但是cobra好像只能在启动时指定参数，无法在运行中输入向Yes 或No这样的参数，有其它的方案吗（除了直接读取标准输入，我现在就这么做的）</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-19 17:37:14</div>
  </div>
</div>
</div>
</li>
</ul>