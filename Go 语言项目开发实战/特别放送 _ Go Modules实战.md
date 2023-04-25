<audio title="特别放送 _ Go Modules实战" src="https://static001.geekbang.org/resource/audio/1d/b2/1d6ea9b5c08894349d423ce7ef3402b2.mp3" controls="controls"></audio> 
<p>你好，我是孔令飞。</p><p>今天我们更新一期特别放送作为加餐。在 <a href="https://time.geekbang.org/column/article/416397">特别放送 | Go Modules依赖包管理全讲</a>中，我介绍了Go Modules的知识，里面内容比较多，你可能还不知道具体怎么使用Go Modules来为你的项目管理Go依赖包。</p><p>这一讲，我就通过一个具体的案例，带你一步步学习Go Modules的常见用法以及操作方法，具体包含以下内容：</p><ol>
<li>准备一个演示项目。</li>
<li>配置Go Modules。</li>
<li>初始化Go包为Go模块。</li>
<li>Go包依赖管理。</li>
</ol><h2>准备一个演示项目</h2><p>为了演示Go Modules的用法，我们首先需要一个Demo项目。假设我们有一个hello的项目，里面有两个文件，分别是hello.go和hello_test.go，所在目录为<code>/home/lk/workspace/golang/src/github.com/marmotedu/gopractise-demo/modules/hello</code>。</p><p>hello.go文件内容为：</p><pre><code class="language-go">package hello

func Hello() string {
	return "Hello, world."
}
</code></pre><p>hello_test.go文件内容为：</p><pre><code class="language-go">package hello

import "testing"

func TestHello(t *testing.T) {
	want := "Hello, world."
	if got := Hello(); got != want {
		t.Errorf("Hello() = %q, want %q", got, want)
	}
}
</code></pre><!-- [[[read_end]]] --><p>这时候，该目录包含了一个Go包，但还不是Go模块，因为没有go.mod件。接下来，我就给你演示下，如何将这个包变成一个Go模块，并执行Go依赖包的管理操作。这些操作共有10个步骤，下面我们来一步步看下。</p><h2>配置Go Modules</h2><ol>
<li>打开Go Modules</li>
</ol><p>确保Go版本<code>&gt;=go1.11</code>，并开启Go Modules，可以通过设置环境变量<code>export GO111MODULE=on</code>开启。如果你觉得每次都设置比较繁琐，可以将<code>export GO111MODULE=on</code>追加到文件<code>$HOME/.bashrc</code>中，并执行 <code>bash</code> 命令加载到当前shell环境中。</p><ol start="2">
<li>设置环境变量</li>
</ol><p>对于国内的开发者来说，需要设置<code>export GOPROXY=https://goproxy.cn,direct</code>，这样一些被墙的包可以通过国内的镜像源安装。如果我们有一些模块存放在私有仓库中，也需要设置GOPRIVATE环境变量。</p><p>因为Go Modules会请求Go Checksum Database，Checksum Database国内也可能会访问失败，可以设置<code>export GOSUMDB=off</code>来关闭Checksum校验。对于一些模块，如果你希望不通过代理服务器，或者不校验<code>checksum</code>，也可以根据需要设置GONOPROXY和GONOSUMDB。</p><h2>初始化Go包为Go模块</h2><ol start="3">
<li>创建一个新模块</li>
</ol><p>你可以通过<code>go mod init</code>命令，初始化项目为Go Modules。 <code>init</code> 命令会在当前目录初始化并创建一个新的go.mod文件，也代表着创建了一个以项目根目录为根的Go Modules。如果当前目录已经存在go.mod文件，则会初始化失败。</p><p>在初始化Go Modules时，需要告知<code>go mod init</code>要初始化的模块名，可以指定模块名，例如<code>go mod init github.com/marmotedu/gopractise-demo/modules/hello</code>。也可以不指定模块名，让<code>init</code>自己推导。下面我来介绍下推导规则。</p><ul>
<li>如果有导入路径注释，则使用注释作为模块名，比如：</li>
</ul><pre><code class="language-bash">package hello // import "github.com/marmotedu/gopractise-demo/modules/hello"
</code></pre><p>则模块名为<code>github.com/marmotedu/gopractise-demo/modules/hello</code>。</p><ul>
<li>如果没有导入路径注释，并且项目位于GOPATH路径下，则模块名为绝对路径去掉<code>$GOPATH/src</code>后的路径名，例如<code>GOPATH=/home/lk/workspace/golang</code>，项目绝对路径为<code>/home/colin/workspace/golang/src/github.com/marmotedu/gopractise-demo/modules/hello</code>，则模块名为<code>github.com/marmotedu/gopractise-demo/modules/hello</code>。</li>
</ul><p>初始化完成之后，会在当前目录生成一个go.mod文件：</p><pre><code class="language-bash">$ cat go.mod
module github.com/marmotedu/gopractise-demo/modules/hello

go 1.14
</code></pre><p>文件内容表明，当前模块的导入路径为<code>github.com/marmotedu/gopractise-demo/modules/hello</code>，使用的Go版本是<code>go 1.14</code>。</p><p>如果要新增子目录创建新的package，则package的导入路径自动为 <code>模块名/子目录名</code> ：<code>github.com/marmotedu/gopractise-demo/modules/hello/&lt;sub-package-name&gt;</code>，不需要在子目录中再次执行<code>go mod init</code>。</p><p>比如，我们在hello目录下又创建了一个world包<code>world/world.go</code>，则world包的导入路径为<code>github.com/marmotedu/gopractise-demo/modules/hello/world</code>。</p><h2>Go包依赖管理</h2><ol start="4">
<li>增加一个依赖</li>
</ol><p>Go Modules主要是用来对包依赖进行管理的，所以这里我们来给hello包增加一个依赖<code>rsc.io/quote</code>：</p><pre><code class="language-go">package hello

import "rsc.io/quote"

func Hello() string {
	return quote.Hello()
}
</code></pre><p>运行<code>go test</code>：</p><pre><code class="language-bash">$ go test
go: finding module for package rsc.io/quote
go: downloading rsc.io/quote v1.5.2
go: found rsc.io/quote in rsc.io/quote v1.5.2
go: downloading rsc.io/sampler v1.3.0
PASS
ok  	github.com/google/addlicense/golang/src/github.com/marmotedu/gopractise-demo/modules/hello	0.003s
</code></pre><p>当go命令在解析源码时，遇到需要导入一个模块的情况，就会去go.mod文件中查询该模块的版本，如果有指定版本，就导入指定的版本。</p><p>如果没有查询到该模块，go命令会自动根据模块的导入路径安装模块，并将模块和其最新的版本写入go.mod文件中。在我们的示例中，<code>go test</code>将模块<code>rsc.io/quote</code>解析为<code>rsc.io/quote v1.5.2</code>，并且同时还下载了<code>rsc.io/quote</code>模块的两个依赖模块：<code>rsc.io/quote</code>和<code>rsc.io/sampler</code>。只有直接依赖才会被记录到go.mod文件中。</p><p>查看go.mod文件：</p><pre><code class="language-bash">module github.com/marmotedu/gopractise-demo/modules/hello

go 1.14

require rsc.io/quote v1.5.2
</code></pre><p>再次执行<code>go test</code>：</p><pre><code class="language-bash">$ go test
PASS
ok  	github.com/marmotedu/gopractise-demo/modules/hello	0.003s
</code></pre><p>当我们再次执行<code>go test</code>时，不会再下载并记录需要的模块，因为go.mod目前是最新的，并且需要的模块已经缓存到了本地的<code>$GOPATH/pkg/mod</code>目录下。可以看到，在当前目录还新生成了一个go.sum文件：</p><pre><code class="language-bash">$ cat go.sum
golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c h1:qgOY6WgZOaTkIIMiVjBQcw93ERBE4m30iBm00nkL0i8=
golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c/go.mod h1:NqM8EUOU14njkJ3fqMW+pc6Ldnwhi/IjpwHt7yyuwOQ=
rsc.io/quote v1.5.2 h1:w5fcysjrx7yqtD/aO+QwRjYZOKnaM9Uh2b40tElTs3Y=
rsc.io/quote v1.5.2/go.mod h1:LzX7hefJvL54yjefDEDHNONDjII0t9xZLPXsUe+TKr0=
rsc.io/sampler v1.3.0 h1:7uVkIFmeBqHfdjD+gZwtXXI+RODJ2Wc4O7MPEh/QiW4=
rsc.io/sampler v1.3.0/go.mod h1:T1hPZKmBbMNahiBKFy5HrXp6adAjACjK9JXDnKaTXpA=
</code></pre><p><code>go test</code>在执行时，还可以添加<code>-mod</code>选项，比如<code>go test -mod=vendor</code>。<code>-mod</code>有3个值，我来分别介绍下。</p><ul>
<li>readonly：不更新go.mod，任何可能会导致go.mod变更的操作都会失败。通常用来检查go.mod文件是否需要更新，例如用在CI或者测试场景。</li>
<li>vendor：从项目顶层目录下的vendor中导入包，而不是从模块缓存中导入包，需要确保vendor包完整准确。</li>
<li>mod：从模块缓存中导入包，即使项目根目录下有vendor目录。</li>
</ul><p>如果<code>go test</code>执行时没有<code>-mod</code>选项，并且项目根目录存在vendor目录，go.mod中记录的go版本大于等于<code>1.14</code>，此时<code>go test</code>执行效果等效于<code>go test -mod=vendor</code>。<code>-mod</code>标志同样适用于go build、go install、go run、go test、go list、go vet命令。</p><ol start="5">
<li>查看所有依赖模块</li>
</ol><p>我们可以通过<code>go list -m all</code>命令查看所有依赖模块：</p><pre><code class="language-bash">$ go list -m all
github.com/marmotedu/gopractise-demo/modules/hello
golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c
rsc.io/quote v1.5.2
rsc.io/sampler v1.3.0
</code></pre><p>可以看出，除了<code>rsc.io/quote v1.5.2</code>外，还间接依赖了其他模块。</p><ol start="6">
<li>更新依赖</li>
</ol><p>通过<code>go list -m all</code>，我们可以看到模块依赖的<code>golang.org/x/text</code>模块版本是<code>v0.0.0</code>，我们可以通过<code>go get</code>命令，将其更新到最新版本，并观察测试是否通过：</p><pre><code class="language-bash">$ go get golang.org/x/text
go: golang.org/x/text upgrade =&gt; v0.3.3
$ go test
PASS
ok  	github.com/marmotedu/gopractise-demo/modules/hello	0.003s
</code></pre><p><code>go test</code>命令执行后输出PASS说明升级成功，再次看下<code>go list -m all</code>和go.mod文件：</p><pre><code class="language-bash">$ go list -m all
github.com/marmotedu/gopractise-demo/modules/hello
golang.org/x/text v0.3.3
golang.org/x/tools v0.0.0-20180917221912-90fa682c2a6e
rsc.io/quote v1.5.2
rsc.io/sampler v1.3.0
$ cat go.mod
module github.com/marmotedu/gopractise-demo/modules/hello

go 1.14

require (
	golang.org/x/text v0.3.3 // indirect
	rsc.io/quote v1.5.2
)
</code></pre><p>可以看到，<code>golang.org/x/text</code>包被更新到最新的tag版本(<code>v0.3.3</code>)，并且同时更新了go.mod文件。<code>// indirect</code>说明<code>golang.org/x/text</code>是间接依赖。现在我们再尝试更新<code>rsc.io/sampler</code>并测试：</p><pre><code class="language-bash">$ go get rsc.io/sampler
go: rsc.io/sampler upgrade =&gt; v1.99.99
go: downloading rsc.io/sampler v1.99.99
$ go test
--- FAIL: TestHello (0.00s)
    hello_test.go:8: Hello() = "99 bottles of beer on the wall, 99 bottles of beer, ...", want "Hello, world."
FAIL
exit status 1
FAIL	github.com/marmotedu/gopractise-demo/modules/hello	0.004s
</code></pre><p>测试失败，说明最新的版本<code>v1.99.99</code>与我们当前的模块不兼容，我们可以列出<code>rsc.io/sampler</code>所有可用的版本，并尝试更新到其他版本：</p><pre><code class="language-bash">$ go list -m -versions rsc.io/sampler
rsc.io/sampler v1.0.0 v1.2.0 v1.2.1 v1.3.0 v1.3.1 v1.99.99

# 我们尝试选择一个次新的版本v1.3.1
$ go get rsc.io/sampler@v1.3.1
go: downloading rsc.io/sampler v1.3.1
$ go test
PASS
ok  	github.com/marmotedu/gopractise-demo/modules/hello	0.004s
</code></pre><p>可以看到，更新到<code>v1.3.1</code>版本，测试是通过的。<code>go get</code>还支持多种参数，如下表所示：</p><p><img src="https://static001.geekbang.org/resource/image/2b/f6/2b80e94c1e91bb18dea9c20695b25bf6.jpg?wh=2248x1496" alt=""></p><ol start="7">
<li>添加一个新的major版本依赖</li>
</ol><p>我们尝试添加一个新的函数<code>func Proverb</code>，该函数通过调用<code>rsc.io/quote/v3</code>的<code>quote.Concurrency</code>函数实现。</p><p>首先，我们在hello.go文件中添加新函数：</p><pre><code class="language-go">package hello

import (
	"rsc.io/quote"
	quoteV3 "rsc.io/quote/v3"
)

func Hello() string {
	return quote.Hello()
}

func Proverb() string {
	return quoteV3.Concurrency()
}
</code></pre><p>在hello_test.go中添加该函数的测试用例：</p><pre><code class="language-go">func TestProverb(t *testing.T) {
    want := "Concurrency is not parallelism."
    if got := Proverb(); got != want {
        t.Errorf("Proverb() = %q, want %q", got, want)
    }
}
</code></pre><p>然后执行测试：</p><pre><code class="language-bash">$ go test
go: finding module for package rsc.io/quote/v3
go: found rsc.io/quote/v3 in rsc.io/quote/v3 v3.1.0
PASS
ok  	github.com/marmotedu/gopractise-demo/modules/hello	0.003s
</code></pre><p>测试通过，可以看到当前模块同时依赖了同一个模块的不同版本<code>rsc.io/quote</code>和<code>rsc.io/quote/v3</code>：</p><pre><code class="language-bash">$ go list -m rsc.io/q...
rsc.io/quote v1.5.2
rsc.io/quote/v3 v3.1.0
</code></pre><ol start="8">
<li>升级到不兼容的版本</li>
</ol><p>在上一步中，我们使用<code>rsc.io/quote v1</code>版本的<code>Hello()</code>函数。按照语义化版本规则，如果我们想升级<code>major</code>版本，可能面临接口不兼容的问题，需要我们变更代码。我们来看下<code>rsc.io/quote/v3</code>的函数：</p><pre><code class="language-bash">$ go doc rsc.io/quote/v3
package quote // import "github.com/google/addlicense/golang/pkg/mod/rsc.io/quote/v3@v3.1.0"

Package quote collects pithy sayings.

func Concurrency() string
func GlassV3() string
func GoV3() string
func HelloV3() string
func OptV3() string
</code></pre><p>可以看到，<code>Hello()</code>函数变成了<code>HelloV3()</code>，这就需要我们变更代码做适配。因为我们都统一模块到一个版本了，这时候就不需要再为了避免重名而重命名模块，所以此时hello.go内容为：</p><pre><code class="language-go">package hello

import (
	"rsc.io/quote/v3"
)

func Hello() string {
	return quote.HelloV3()
}

func Proverb() string {
	return quote.Concurrency()
}
</code></pre><p>执行<code>go test</code>：</p><pre><code class="language-bash">$ go test
PASS
ok  	github.com/marmotedu/gopractise-demo/modules/hello	0.003s
</code></pre><p>可以看到测试成功。</p><ol start="9">
<li>删除不使用的依赖</li>
</ol><p>在上一步中，我们移除了<code>rsc.io/quote</code>包，但是它仍然存在于<code>go list -m all</code>和go.mod中，这时候我们要执行<code>go mod tidy</code>清理不再使用的依赖：</p><pre><code class="language-bash">$ go mod tidy
[colin@dev hello]$ cat go.mod
module github.com/marmotedu/gopractise-demo/modules/hello

go 1.14

require (
	golang.org/x/text v0.3.3 // indirect
	rsc.io/quote/v3 v3.1.0
	rsc.io/sampler v1.3.1 // indirect
)
</code></pre><ol start="10">
<li>使用vendor</li>
</ol><p>如果我们想把所有依赖都保存起来，在Go命令执行时不再下载，可以执行<code>go mod vendor</code>，该命令会把当前项目的所有依赖都保存在项目根目录的vendor目录下，也会创建<code>vendor/modules.txt</code>文件，来记录包和模块的版本信息：</p><pre><code class="language-bash">$ go mod vendor
$ ls
go.mod  go.sum  hello.go  hello_test.go  vendor  world
</code></pre><p>到这里，我就讲完了Go依赖包管理常用的10个操作。</p><h2>总结</h2><p>这一讲中，我详细介绍了如何使用Go Modules来管理依赖，它包括以下Go Modules操作：</p><ol>
<li>打开Go Modules；</li>
<li>设置环境变量；</li>
<li>创建一个新模块；</li>
<li>增加一个依赖；</li>
<li>查看所有依赖模块；</li>
<li>更新依赖；</li>
<li>添加一个新的major版本依赖；</li>
<li>升级到不兼容的版本；</li>
<li>删除不使用的依赖。</li>
<li>使用vendor。</li>
</ol><h2>课后练习</h2><ol>
<li>
<p>思考下，如何更新项目的所有依赖到最新的版本？</p>
</li>
<li>
<p>思考下，如果我们的编译机器访问不了外网，如何通过Go Modules下载Go依赖包？</p>
</li>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/25/06/13/e4f9f79b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>你赖东东不错嘛</span>
  </div>
  <div class="_2_QraFYR_0">1. 项目根目录下，执行go get -d -u .&#47;...<br>2. 在外网环境把package下载到vendor目录下，在无网环境用go vendor构建应用。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是一种解决方法。<br><br>第2个问题，还有一种思路：在内网搭建Go Modules代理服务器</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-22 16:42:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/45/be/a4182df4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>2035去台湾</span>
  </div>
  <div class="_2_QraFYR_0">客户练习2目前我们使用了nexus代理</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对，可以选择搭建代理</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-21 22:22:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/57/4f/6fb51ff1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>一步</span>
  </div>
  <div class="_2_QraFYR_0">go test 没有自动下载依赖，是需要配置什么吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: go mod tidy试试</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-14 22:27:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Ge7uhlEVxicQT73YuomDPrVKI8UmhqxKWrhtO5GMNlFjrHWfd3HAjgaSribR4Pzorw8yalYGYqJI4VPvUyPzicSKg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>陈东</span>
  </div>
  <div class="_2_QraFYR_0">go构建约束问题，Build constraints exclude all Go files in ？<br>尝试以下办法解决不了<br>1、searcheverything 搜索后删除所有包,<br>$GOPATH目录下，把对应的包删除，重新go get,还是不行.<br>2、go get -u -v github.com&#47;karalabe&#47;xgo<br>3、Right click -&gt; Mark folder as not excluded.<br>4、引用包报错，重启电脑，查看goproxy配置，还不行重装goland<br>怎么解决，寻求老师帮助，谢谢。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 试试这个？<br><br>把 CGO_ENABLED=1 GOOS=linux<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-18 01:36:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/29/b0/d3/200e82ff.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>功夫熊猫</span>
  </div>
  <div class="_2_QraFYR_0">有没有办法就是直接导入本地包。而不是设置代理</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以搭建本地代理，暂时没有其他办法</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-03 17:54:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJkOj8VUxLjDKp6jRWJrABnnsg7U1sMSkM8FO6ULPwrqNpicZvTQ7kwctmu38iaJYHybXrmbusd8trg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>yss</span>
  </div>
  <div class="_2_QraFYR_0">2. 我们内网机是与外网物理隔离的机器，使用 vendor在内网机构建是我们的解决方案。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 现在不建议使用vendor了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-28 09:53:57</div>
  </div>
</div>
</div>
</li>
</ul>