<audio title="特别放送 _ Go Modules依赖包管理全讲" src="https://static001.geekbang.org/resource/audio/2e/c1/2eb38d6f8bdf1173391c92ee83dab7c1.mp3" controls="controls"></audio> 
<p>你好，我是孔令飞。今天我们更新一期特别放送作为加餐。</p><p>在Go项目开发中，依赖包管理是一个非常重要的内容，依赖包处理不好，就会导致编译失败。而且Go的依赖包管理有一定的复杂度，所以，我们有必要系统学习下Go的依赖包管理工具。</p><p>这一讲，我会首先介绍下Go依赖包管理工具的历史，并详细介绍下目前官方推荐的依赖包管理方案Go Modules。Go Modules主要包括了 <code>go mod</code> 命令行工具、模块下载机制，以及两个核心文件go.mod和go.sum。另外，Go Modules也提供了一些环境变量，用来控制Go Modules的行为。这一讲，我会分别介绍下这些内容。</p><p>在正式开始讲解这些内容之前，我们先来对Go Modules有个基本的了解。</p><h2>Go Modules简介</h2><p>Go Modules是Go官方推出的一个Go包管理方案，基于vgo演进而来，具有下面这几个特性：</p><ul>
<li>可以使包的管理更加简单。</li>
<li>支持版本管理。</li>
<li>允许同一个模块多个版本共存。</li>
<li>可以校验依赖包的哈希值，确保包的一致性，增加安全性。</li>
<li>内置在几乎所有的go命令中，包括<code>go get</code>、<code>go build</code>、<code>go install</code>、<code>go run</code>、<code>go test</code>、<code>go list</code>等命令。</li>
<li>具有Global Caching特性，不同项目的相同模块版本，只会在服务器上缓存一份。</li>
</ul><!-- [[[read_end]]] --><p>在Go1.14版本以及之后的版本，Go官方建议在生产环境中使用Go Modules。因此，以后的Go包管理方案会逐渐统一到Go Modules。与Go Modules相关的概念很多，我在这里把它们总结为“6-2-2-1-1”，这一讲后面还会详细介绍每个概念。</p><ul>
<li>六个环境变量：<code>GO111MODULE</code>、<code>GOPROXY</code>、<code>GONOPROXY</code>、<code>GOSUMDB</code>、<code>GONOSUMDB</code>、<code>GOPRIVATE</code>。</li>
<li>两个概念：Go module proxy和Go checksum database。</li>
<li>两个主要文件：go.mod和go.sum。</li>
<li>一个主要管理命令：go mod。</li>
<li>一个build flag。</li>
</ul><h2>Go包管理的历史</h2><p>在具体讲解Go Modules之前，我们先看一下Go包管理的历史。从Go推出之后，因为没有一个统一的官方方案，所以出现了很多种Go包管理方案，比较混乱，也没有彻底解决Go包管理的一些问题。Go包管理的历史如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/34/e5/348d772b26940f721c6fb907f6833be5.jpg?wh=2248x739" alt=""></p><p>这张图展示了Go依赖包管理工具经历的几个发展阶段，接下来我会按时间顺序重点介绍下其中的五个阶段。</p><h3><strong>Go1.5版本前：GOPATH</strong></h3><p>在Go1.5版本之前，没有版本控制，所有的依赖包都放在GOPATH下。采用这种方式，无法实现包的多版本管理，并且包的位置只能局限在GOPATH目录下。如果A项目和B项目用到了同一个Go包的不同版本，这时候只能给每个项目设置一个GOPATH，将对应版本的包放在各自的GOPATH目录下，切换项目目录时也需要切换GOPATH。这些都增加了开发和实现的复杂度。</p><h3><strong>Go1.5版本：Vendoring</strong></h3><p>Go1.5推出了vendor机制，并在Go1.6中默认启用。在这个机制中，每个项目的根目录都可以有一个vendor目录，里面存放了该项目的Go依赖包。在编译Go源码时，Go优先从项目根目录的vendor目录查找依赖；如果没有找到，再去GOPATH下的vendor目录下找；如果还没有找到，就去GOPATH下找。</p><p>这种方式解决了多GOPATH的问题，但是随着项目依赖的增多，vendor目录会越来越大，造成整个项目仓库越来越大。在vendor机制下，一个中型项目的vendor目录有几百M的大小一点也不奇怪。</p><h3>“百花齐放”：多种Go依赖包管理工具出现</h3><p>这个阶段，社区也出现了很多Go依赖包管理的工具，这里我介绍三个比较有名的。</p><ul>
<li>Godep：解决包依赖的管理工具，Docker、Kubernetes、CoreOS等Go项目都曾用过godep来管理其依赖。</li>
<li>Govendor：它的功能比Godep多一些，通过vendor目录下的<code>vendor.json</code>文件来记录依赖包的版本。</li>
<li>Glide：相对完善的包管理工具，通过<code>glide.yaml</code>记录依赖信息，通过<code>glide.lock</code>追踪每个包的具体修改。</li>
</ul><p>Govendor、Glide都是在Go支持vendor之后推出的工具，Godep在Go支持vendor之前也可以使用。Go支持vendor之后，Godep也改用了vendor模式。</p><h3>Go1.9版本：Dep</h3><p>对于从0构建项目的新用户来说，Glide功能足够，是个不错的选择。不过，Golang 依赖管理工具混乱的局面最终由官方来终结了：Golang官方接纳了由社区组织合作开发的Dep，作为official experiment。在相当长的一段时间里，Dep作为标准，成为了事实上的官方包管理工具。</p><p>因为Dep已经成为了official experiment的过去时，现在我们就不必再去深究了，让我们直接去了解谁才是未来的official experiment吧。</p><h3>Go1.11版本之后：Go Modules</h3><p>Go1.11版本推出了Go Modules机制，Go Modules基于vgo演变而来，是Golang官方的包管理工具。在Go1.13版本，Go语言将Go Modules设置为默认的Go管理工具；在Go1.14版本，Go语言官方正式推荐在生产环境使用Go Modules，并且鼓励所有用户从其他的依赖管理工具迁移过来。至此，Go终于有了一个稳定的、官方的Go包管理工具。</p><p>到这里，我介绍了Go依赖包管理工具的历史，下面再来介绍下Go Modules的使用方法。</p><h2>包（package）和模块（module）</h2><p>Go程序被组织到Go包中，Go包是同一目录中一起编译的Go源文件的集合。在一个源文件中定义的函数、类型、变量和常量，对于同一包中的所有其他源文件可见。</p><p>模块是存储在文件树中的Go包的集合，并且文件树根目录有go.mod文件。go.mod文件定义了模块的名称及其依赖包，每个依赖包都需要指定导入路径和语义化版本（Semantic Versioning），通过导入路径和语义化版本准确地描述一个依赖。</p><p>这里要注意，<code>"module" != "package"</code>，模块和包的关系更像是集合和元素的关系，包属于模块，一个模块是零个或者多个包的集合。下面的代码段，引用了一些包：</p><pre><code class="language-go">import (
    // Go 标准包
    "fmt"

    // 第三方包
    "github.com/spf13/pflag"

    // 匿名包
     _ "github.com/jinzhu/gorm/dialects/mysql"

     // 内部包
    "github.com/marmotedu/iam/internal/apiserver"
)
</code></pre><p>这里的<code>fmt</code>、<code>github.com/spf13/pflag</code>和<code>github.com/marmotedu/iam/internal/apiserver</code>都是Go包。Go中有4种类型的包，下面我来分别介绍下。</p><ul>
<li>Go标准包：在Go源码目录下，随Go一起发布的包。</li>
<li>第三方包：第三方提供的包，比如来自于github.com的包。</li>
<li>匿名包：只导入而不使用的包。通常情况下，我们只是想使用导入包产生的副作用，即引用包级别的变量、常量、结构体、接口等，以及执行导入包的<code>init()</code>函数。</li>
<li>内部包：项目内部的包，位于项目目录下。</li>
</ul><p>下面的目录定义了一个模块：</p><pre><code class="language-bash">$ ls hello/
go.mod  go.sum  hello.go  hello_test.go  world
</code></pre><p>hello目录下有一个go.mod文件，说明了这是一个模块，该模块包含了hello包和一个子包world。该目录中也包含了一个go.sum文件，该文件供Go命令在构建时判断依赖包是否合法。这里你先简单了解下，我会在下面讲go.sum文件的时候详细介绍。</p><h2>Go Modules 命令</h2><p>Go Modules的管理命令为<code>go mod</code>，<code>go mod</code>有很多子命令，你可以通过<code>go help mod</code>来获取所有的命令。下面我来具体介绍下这些命令。</p><ul>
<li>download：下载go.mod文件中记录的所有依赖包。</li>
<li>edit：编辑go.mod文件。</li>
<li>graph：查看现有的依赖结构。</li>
<li>init：把当前目录初始化为一个新模块。</li>
<li>tidy：添加丢失的模块，并移除无用的模块。默认情况下，Go不会移除go.mod文件中的无用依赖。当依赖包不再使用了，可以使用<code>go mod tidy</code>命令来清除它。</li>
<li>vendor：将所有依赖包存到当前目录下的vendor目录下。</li>
<li>verify：检查当前模块的依赖是否已经存储在本地下载的源代码缓存中，以及检查下载后是否有修改。</li>
<li>why：查看为什么需要依赖某模块。</li>
</ul><h2>Go Modules开关</h2><p>如果要使用Go Modules，在Go1.14中仍然需要确保Go Modules特性处在打开状态。你可以通过环境变量GO111MODULE来打开或者关闭。GO111MODULE有3个值，我来分别介绍下。</p><ul>
<li>auto：在Go1.14版本中是默认值，在<code>$GOPATH/src</code>下，且没有包含go.mod时则关闭Go Modules，其他情况下都开启Go Modules。</li>
<li>on：启用Go Modules，Go1.14版本推荐打开，未来版本会设为默认值。</li>
<li>off：关闭Go Modules，不推荐。</li>
</ul><p>所以，如果要打开Go Modules，可以设置环境变量<code>export GO111MODULE=on</code>或者<code>export GO111MODULE=auto</code>，建议直接设置<code>export GO111MODULE=on</code>。</p><p>Go Modules使用语义化的版本号，我们开发的模块在发布版本打tag的时候，要注意遵循语义化的版本要求，不遵循语义化版本规范的版本号都是无法拉取的。</p><h2>模块下载</h2><p>在执行 <code>go get</code> 等命令时，会自动下载模块。接下来，我会介绍下go命令是如何下载模块的。主要有三种下载方式：</p><ul>
<li>通过代理下载；</li>
<li>指定版本号下载；</li>
<li>按最小版本下载。</li>
</ul><h3>通过代理来下载模块</h3><p>默认情况下，Go命令从VCS（Version Control System，版本控制系统）直接下载模块，例如 GitHub、Bitbucket、Bazaar、Mercurial或者SVN。</p><p>在Go 1.13版本，引入了一个新的环境变量GOPROXY，用于设置Go模块代理（Go module proxy）。模块代理可以使Go命令直接从代理服务器下载模块。GOPROXY默认值为<code>https://proxy.golang.org,direct</code>，代理服务器可以指定多个，中间用逗号隔开，例如<code>GOPROXY=https://proxy.golang.org,https://goproxy.cn,direct</code>。当下载模块时，会优先从指定的代理服务器上下载。如果下载失败，比如代理服务器不可访问，或者HTTP返回码为<code>404</code>或<code>410</code>，Go命令会尝试从下一个代理服务器下载。</p><p>direct是一个特殊指示符，用来指示Go回源到模块的源地址(比如GitHub等)去抓取 ，当值列表中上一个Go module proxy返回404或410，Go会自动尝试列表中的下一个，遇见direct时回源，遇见EOF时终止，并抛出类似<code>invalid version: unknown revision...</code>的错误。 如果<code>GOPROXY=off</code>，则Go命令不会尝试从代理服务器下载模块。</p><p>引入Go module proxy会带来很多好处，比如：</p><ul>
<li>国内开发者无法访问像golang.org、gopkg.in、go.uber.org这类域名，可以设置GOPROXY为国内可以访问的代理服务器，解决依赖包下载失败的问题。</li>
<li>Go模块代理会永久缓存和存储所有的依赖，并且这些依赖一经缓存，不可更改，这也意味着我们不需要再维护一个vendor目录，也可以避免因为维护vendor目录所带来的存储空间占用。</li>
<li>因为依赖永久存在于代理服务器，这样即使模块从互联网上被删除，也仍然可以通过代理服务器获取到。</li>
<li>一旦将Go模块存储在Go代理服务器中，就无法覆盖或删除它，这可以保护开发者免受可能注入相同版本恶意代码所带来的攻击。</li>
<li>我们不再需要VCS工具来下载依赖，因为所有的依赖都是通过HTTP的方式从代理服务器下载。</li>
<li>因为Go代理通过HTTP独立提供了源代码（.zip存档）和go.mod，所以下载和构建Go模块的速度更快。因为可以独立获取go.mod（而之前必须获取整个仓库），所以解决依赖也更快。</li>
<li>当然，开发者也可以设置自己的Go模块代理，这样开发者可以对依赖包有更多的控制，并可以预防VCS停机所带来的下载失败。</li>
</ul><p>在实际开发中，我们的很多模块可能需要从私有仓库拉取，通过代理服务器访问会报错，这时候我们需要将这些模块添加到环境变量GONOPROXY中，这些私有模块的哈希值也不会在checksum database中存在，需要将这些模块添加到GONOSUMDB中。一般来说，我建议直接设置GOPRIVATE环境变量，它的值将作为GONOPROXY和GONOSUMDB的默认值。</p><p>GONOPROXY、GONOSUMDB和GOPRIVATE都支持通配符，多个域名用逗号隔开，例如<code>*.example.com,github.com</code>。</p><p>对于国内的Go开发者来说，目前有3个常用的GOPROXY可供选择，分别是官方、七牛和阿里云。</p><p>官方的GOPROXY，国内用户可能访问不到，所以我更推荐使用七牛的<code>goproxy.cn</code>，<code>goproxy.cn</code>是七牛云推出的非营利性项目，它的目标是为中国和世界上其他地方的Go开发者提供一个免费、可靠、持续在线，且经过 CDN 加速的模块代理。</p><h3>指定版本号下载</h3><p>通常，我们通过<code>go get</code>来下载模块，下载命令格式为<code>go get &lt;package[@version]&gt;</code>，如下表所示：</p><p><img src="https://static001.geekbang.org/resource/image/63/92/63fbbf7cf8b67af85aa4e7ce76a99392.jpg?wh=2248x2048" alt=""></p><p>你可以使用<code>go get -u</code>更新package到latest版本，也可以使用<code>go get -u=patch</code>只更新小版本，例如从<code>v1.2.4</code>到<code>v1.2.5</code>。</p><h3>按最小版本下载</h3><p>一个模块往往会依赖许多其他模块，并且不同的模块也可能会依赖同一个模块的不同版本，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/00/46/00794e3487e63d9d3302bfe977af6d46.jpg?wh=2248x1212" alt=""></p><p>在上述依赖中，模块A依赖了模块B和模块C，模块B依赖了模块D，模块C依赖了模块D和模块F，模块D又依赖了模块E。并且，同模块的不同版本还依赖了对应模块的不同版本。</p><p>那么Go Modules是如何选择版本的呢？Go Modules 会把每个模块的依赖版本清单都整理出来，最终得到一个构建清单，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/4f/f2/4f83ffe8125d75764c5f8069a73966f2.jpg?wh=2248x1148" alt=""></p><p>上图中，rough list和final list的区别在于重复引用的模块 D（<code>v1.3</code>、<code>v1.4</code>），最终清单选用了D的<code>v1.4</code>版本。</p><p>这样做的主要原因有两个。第一个是语义化版本的控制。因为模块D的<code>v1.3</code>和<code>v1.4</code>版本变更都属于次版本号的变更，而在语义化版本的约束下，<code>v1.4</code>必须要向下兼容<code>v1.3</code>，因此我们要选择高版本的<code>v1.4</code>。</p><p>第二个是模块导入路径的规范。主版本号不同，模块的导入路径就不一样。所以，如果出现不兼容的情况，主版本号会改变，例如从v1变为v2，模块的导入路径也就改变了，因此不会影响v1版本。</p><h2>go.mod和go.sum介绍</h2><p>在Go Modules中，go.mod和go.sum是两个非常重要的文件，下面我就来详细介绍这两个文件。</p><h3>go.mod文件介绍</h3><p>go.mod文件是Go Modules的核心文件。下面是一个go.mod文件示例：</p><pre><code class="language-bash">module github.com/marmotedu/iam

go 1.14

require (
	github.com/AlekSi/pointer v1.1.0
	github.com/appleboy/gin-jwt/v2 v2.6.3
	github.com/asaskevich/govalidator v0.0.0-20200428143746-21a406dcc535
	github.com/gin-gonic/gin v1.6.3
	github.com/golangci/golangci-lint v1.30.0 // indirect
	github.com/google/uuid v1.0.0
    github.com/blang/semver v3.5.0+incompatible
    golang.org/x/text v0.3.2
)

replace (
    github.com/gin-gonic/gin =&gt; /home/colin/gin
    golang.org/x/text v0.3.2 =&gt; github.com/golang/text v0.3.2
)

exclude (
    github.com/google/uuid v1.1.0
)
</code></pre><p>接下来，我会从go.mod语句、go.mod版本号、go.mod文件修改方法三个方面来介绍go.mod。</p><ol>
<li>go.mod语句</li>
</ol><p>go.mod文件中包含了4个语句，分别是module、require、replace 和 exclude。下面我来介绍下它们的功能。</p><ul>
<li>module：用来定义当前项目的模块路径。</li>
<li>go：用来设置预期的Go版本，目前只是起标识作用。</li>
<li>require：用来设置一个特定的模块版本，格式为<code>&lt;导入包路径&gt; &lt;版本&gt; [// indirect]</code>。</li>
<li>exclude：用来从使用中排除一个特定的模块版本，如果我们知道模块的某个版本有严重的问题，就可以使用exclude将该版本排除掉。</li>
<li>replace：用来将一个模块版本替换为另外一个模块版本。格式为 <code>$module =&gt; $newmodule</code> ，<code>$newmodule</code>可以是本地磁盘的相对路径，例如<code>github.com/gin-gonic/gin =&gt; ./gin</code>。也可以是本地磁盘的绝对路径，例如<code>github.com/gin-gonic/gin =&gt; /home/lk/gin</code>。还可以是网络路径，例如<code>golang.org/x/text v0.3.2 =&gt; github.com/golang/text v0.3.2</code>。</li>
</ul><p>这里需要注意，虽然我们用<code>$newmodule</code>替换了<code>$module</code>，但是在代码中的导入路径仍然为<code>$module</code>。replace在实际开发中经常用到，下面的场景可能需要用到replace：</p><ul>
<li>在开启Go Modules后，缓存的依赖包是只读的，但在日常开发调试中，我们可能需要修改依赖包的代码来进行调试，这时可以将依赖包另存到一个新的位置，并在go.mod中替换这个包。</li>
<li>如果一些依赖包在Go命令运行时无法下载，就可以通过其他途径下载该依赖包，上传到开发构建机，并在go.mod中替换为这个包。</li>
<li>在项目开发初期，A项目依赖B项目的包，但B项目因为种种原因没有push到仓库，这时也可以在go.mod中把依赖包替换为B项目的本地磁盘路径。</li>
<li>在国内访问golang.org/x的各个包都需要翻墙，可以在go.mod中使用replace，替换成GitHub上对应的库，例如<code>golang.org/x/text v0.3.0 =&gt; github.com/golang/text v0.3.0</code>。</li>
</ul><p>有一点要注意，exclude和replace只作用于当前主模块，不影响主模块所依赖的其他模块。</p><ol start="2">
<li>go.mod版本号</li>
</ol><p>go.mod文件中有很多版本号格式，我知道在平时使用中，有很多开发者对此感到困惑。这里，我来详细说明一下。</p><ul>
<li>如果模块具有符合语义化版本格式的tag，会直接展示tag的值，例如 <code>github.com/AlekSi/pointer v1.1.0</code> 。</li>
<li>除了v0和v1外，主版本号必须显试地出现在模块路径的尾部，例如<code>github.com/appleboy/gin-jwt/v2 v2.6.3</code>。</li>
<li>对于没有tag的模块，Go命令会选择master分支上最新的commit，并根据commit时间和哈希值生成一个符合语义化版本的版本号，例如<code>github.com/asaskevich/govalidator v0.0.0-20200428143746-21a406dcc535</code>。</li>
<li>如果模块名字跟版本不符合规范，例如模块的名字为<code>github.com/blang/semver</code>，但是版本为 <code>v3.5.0</code>（正常应该是<code>github.com/blang/semver/v3</code>），go会在go.mod的版本号后加<code>+incompatible</code>表示。</li>
<li>如果go.mod中的包是间接依赖，则会添加<code>// indirect</code>注释，例如<code>github.com/golangci/golangci-lint v1.30.0 // indirect</code>。</li>
</ul><p>这里要注意，Go Modules要求模块的版本号格式为<code>v&lt;major&gt;.&lt;minor&gt;.&lt;patch&gt;</code>，如果<code>&lt;major&gt;</code>版本号大于1，它的版本号还要体现在模块名字中，例如模块<code>github.com/blang/semver</code>版本号增长到<code>v3.x.x</code>，则模块名应为<code>github.com/blang/semver/v3</code>。</p><p>这里再详细介绍下出现<code>// indirect</code>的情况。原则上go.mod中出现的都是直接依赖，但是下面的两种情况只要出现一种，就会在go.mod中添加间接依赖。</p><ul>
<li>直接依赖未启用Go Modules：如果模块A依赖模块B，模块B依赖B1和B2，但是B没有go.mod文件，则B1和B2会记录到A的go.mod文件中，并在最后加上<code>// indirect</code>。</li>
<li>直接依赖go.mod文件中缺失部分依赖：如果模块A依赖模块B，模块B依赖B1和B2，B有go.mod文件，但是只有B1被记录在B的go.mod文件中，这时候B2会被记录到A的go.mod文件中，并在最后加上<code>// indirect</code>。</li>
</ul><ol start="3">
<li>go.mod文件修改方法</li>
</ol><p>要修改go.mod文件，我们可以采用下面这三种方法：</p><ul>
<li>Go命令在运行时自动修改。</li>
<li>手动编辑go.mod文件，编辑之后可以执行<code>go mod edit -fmt</code>格式化go.mod文件。</li>
<li>执行go mod子命令修改。</li>
</ul><p>在实际使用中，我建议你采用第三种修改方法，和其他两种相比不太容易出错。使用方式如下：</p><pre><code class="language-bash">go mod edit -fmt  # go.mod 格式化
go mod edit -require=golang.org/x/text@v0.3.3  # 添加一个依赖
go mod edit -droprequire=golang.org/x/text # require的反向操作，移除一个依赖
go mod edit -replace=github.com/gin-gonic/gin=/home/colin/gin # 替换模块版本
go mod edit -dropreplace=github.com/gin-gonic/gin # replace的反向操作
go mod edit -exclude=golang.org/x/text@v0.3.1 # 排除一个特定的模块版本
go mod edit -dropexclude=golang.org/x/text@v0.3.1 # exclude的反向操作
</code></pre><h3>go.sum文件介绍</h3><p>Go会根据go.mod文件中记载的依赖包及其版本下载包源码，但是下载的包可能被篡改，缓存在本地的包也可能被篡改。单单一个go.mod文件，不能保证包的一致性。为了解决这个潜在的安全问题，Go Modules引入了go.sum文件。</p><p>go.sum文件用来记录每个依赖包的hash值，在构建时，如果本地的依赖包hash值与<code>go.sum</code>文件中记录的不一致，则会拒绝构建。go.sum中记录的依赖包是所有的依赖包，包括间接和直接的依赖包。</p><p>这里提示下，为了避免已缓存的模块被更改，<code>$GOPATH/pkg/mod</code>下缓存的包是只读的，不允许修改。</p><p>接下来我从go.sum文件内容、go.sum文件生成、校验三个方面来介绍go.sum。</p><ol>
<li>go.sum文件内容</li>
</ol><p>下面是一个go.sum文件的内容：</p><pre><code class="language-bash">golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c h1:qgOY6WgZOaTkIIMiVjBQcw93ERBE4m30iBm00nkL0i8=
golang.org/x/text v0.0.0-20170915032832-14c0d48ead0c/go.mod h1:NqM8EUOU14njkJ3fqMW+pc6Ldnwhi/IjpwHt7yyuwOQ=
rsc.io/quote v1.5.2 h1:w5fcysjrx7yqtD/aO+QwRjYZOKnaM9Uh2b40tElTs3Y=
rsc.io/quote v1.5.2/go.mod h1:LzX7hefJvL54yjefDEDHNONDjII0t9xZLPXsUe+TKr0=
rsc.io/sampler v1.3.0 h1:7uVkIFmeBqHfdjD+gZwtXXI+RODJ2Wc4O7MPEh/QiW4=
rsc.io/sampler v1.3.0/go.mod h1:T1hPZKmBbMNahiBKFy5HrXp6adAjACjK9JXDnKaTXpA=
</code></pre><p>go.sum文件中，每行记录由模块名、版本、哈希算法和哈希值组成，如<code>&lt;module&gt; &lt;version&gt;[/go.mod] &lt;algorithm&gt;:&lt;hash&gt;</code>。目前，从Go1.11到Go1.14版本，只有一个算法SHA-256，用h1表示。</p><p>正常情况下，每个依赖包会包含两条记录，分别是依赖包所有文件的哈希值和该依赖包go.mod的哈希值，例如：</p><pre><code class="language-bash">rsc.io/quote v1.5.2 h1:w5fcysjrx7yqtD/aO+QwRjYZOKnaM9Uh2b40tElTs3Y=
rsc.io/quote v1.5.2/go.mod h1:LzX7hefJvL54yjefDEDHNONDjII0t9xZLPXsUe+TKr0=
</code></pre><p>但是，如果一个依赖包没有go.mod文件，就只记录依赖包所有文件的哈希值，也就是只有第一条记录。额外记录go.mod的哈希值，主要是为了在计算依赖树时不必下载完整的依赖包版本，只根据go.mod即可计算依赖树。</p><ol start="2">
<li>go.sum文件生成</li>
</ol><p>在Go Modules开启时，如果我们的项目需要引入一个新的包，通常会执行<code>go get</code>命令，例如：</p><pre><code class="language-bash">$ go get rsc.io/quote
</code></pre><p>当执行<code>go get rsc.io/quote</code>命令后，<code>go get</code>命令会先将依赖包下载到<code>$GOPATH/pkg/mod/cache/download</code>，下载的依赖包文件名格式为<code>$version.zip</code>，例如<code>v1.5.2.zip</code>。</p><p>下载完成之后，<code>go get</code>会对该zip包做哈希运算，并将结果存在<code>$version.ziphash</code>文件中，例如<code>v1.5.2.ziphash</code>。如果在项目根目录下执行<code>go get</code>命令，则<code>go get</code>会同时更新go.mod和go.sum文件。例如，go.mod新增一行<code>require rsc.io/quote v1.5.2</code>，go.sum新增两行：</p><pre><code class="language-bash">rsc.io/quote v1.5.2 h1:w5fcysjrx7yqtD/aO+QwRjYZOKnaM9Uh2b40tElTs3Y=
rsc.io/quote v1.5.2/go.mod h1:LzX7hefJvL54yjefDEDHNONDjII0t9xZLPXsUe+TKr0=
</code></pre><ol start="3">
<li>校验</li>
</ol><p>在我们执行构建时，go命令会从本地缓存中查找所有的依赖包，并计算这些依赖包的哈希值，然后与go.sum中记录的哈希值进行对比。如果哈希值不一致，则校验失败，停止构建。</p><p>校验失败可能是因为本地指定版本的依赖包被修改过，也可能是go.sum中记录的哈希值是错误的。但是Go命令倾向于相信依赖包被修改过，因为当我们在go get依赖包时，包的哈希值会经过校验和数据库（checksum database）进行校验，校验通过才会被加入到go.sum文件中。也就是说，go.sum文件中记录的哈希值是可信的。</p><p>校验和数据库可以通过环境变量<code>GOSUMDB</code>指定，<code>GOSUMDB</code>的值是一个web服务器，默认值是<code>sum.golang.org</code>。该服务可以用来查询依赖包指定版本的哈希值，保证拉取到的模块版本数据没有经过篡改。</p><p>如果设置<code>GOSUMDB</code>为<code>off</code>，或者使用<code>go get</code>的时候启用了<code>-insecure</code>参数，Go就不会去对下载的依赖包做安全校验，这存在一定的安全隐患，所以我建议你开启校验和数据库。如果对安全性要求很高，同时又访问不了<code>sum.golang.org</code>，你也可以搭建自己的校验和数据库。</p><p>值得注意的是，Go checksum database可以被Go module proxy代理，所以当我们设置了<code>GOPROXY</code>后，通常情况下不用再设置<code>GOSUMDB</code>。还要注意的是，go.sum文件也应该提交到你的 Git 仓库中去。</p><h2>模块下载流程</h2><p>上面，我介绍了模块下载的整体流程，还介绍了go.mod和go.sum这两个文件。因为内容比较多，这里用一张图片来做个总结：</p><p><img src="https://static001.geekbang.org/resource/image/8b/8d/8b92e53cebd4373f8c41fdbe9328ba8d.jpg?wh=2248x1077" alt=""></p><p>最后还想介绍下Go&nbsp;modules的全局缓存。Go&nbsp;modules中，相同版本的模块只会缓存一份，其他所有模块公用。目前，所有模块版本数据都缓存在 <code>$GOPATH/pkg/mod</code> 和 <code>$GOPATH/pkg/sum</code> 下，未来有可能移到 <code>$GOCACHE/mod</code> 和 <code>$GOCACHE/sum</code> 下，我认为这可能发生在 <code>GOPATH</code> 被淘汰后。你可以使用 <code>go&nbsp;clean&nbsp;-modcache</code> 清除所有的缓存。</p><h2>总结</h2><p>Go依赖包管理是Go语言中一个重点的功能。在Go1.11版本之前，并没有官方的依赖包管理工具，业界虽然存在多个Go依赖包管理方案，但效果都不理想。直到Go1.11版本，Go才推出了官方的依赖包管理工具，Go Modules。这也是我建议你在进行Go项目开发时选择的依赖包管理工具。</p><p>Go Modules提供了 <code>go mod</code> 命令，来管理Go的依赖包。 <code>go mod</code> 有很多子命令，这些子命令可以完成不同的功能。例如，初始化当前目录为一个新模块，添加丢失的模块，移除无用的模块，等等。</p><p>在Go Modules中，有两个非常重要的文件：go.mod和go.sum。go.mod文件是Go Modules的核心文件，Go会根据go.mod文件中记载的依赖包及其版本下载包源码。go.sum文件用来记录每个依赖包的hash值，在构建时，如果本地的依赖包hash值与go.sum文件中记录的不一致，就会拒绝构建。</p><p>Go在下载依赖包时，可以通过代理来下载，也可以指定版本号下载。如果不指定版本号，Go Modules会根据自定义的规则，选择最小版本来下载。</p><h2>课后练习</h2><ol>
<li>思考下，如果不提交go.sum，会有什么风险？</li>
<li>找一个没有使用Go Modules管理依赖包的Go项目，把它的依赖包管理方式切换为Go Modules。</li>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/28/1d/69/c21d2644.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>josephzxy</span>
  </div>
  <div class="_2_QraFYR_0">“思考下，如果不提交 go.sum，会有什么风险？”<br>如果go get时，GOSUMDB=off，就没有办法校验下载的包是否被篡改。<br><br>推荐两篇博文可做辅助阅读<br>https:&#47;&#47;zaracooper.github.io&#47;blog&#47;posts&#47;go-module-tidbits&#47;<br>https:&#47;&#47;insujang.github.io&#47;2020-04-04&#47;go-modules&#47;</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 老哥，回答给满分~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-09 12:11:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/f5/0b/73628618.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>兔嘟嘟</span>
  </div>
  <div class="_2_QraFYR_0">解惑了，整个中文互联网上都没这样好好讲go module的，至少我没找到。导致我一开始搞项目的时候，在某个地方纠结了好久</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-19 10:00:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/ed/0a/18201290.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Juniper</span>
  </div>
  <div class="_2_QraFYR_0">问下老师，go get和go mod download的场景有什么不一样吗，不太明天go mod download这个命令的场景<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: go get是下载单个包。<br><br><br>go mod download是下载go.mod中记录的包（1个&#47;多个）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-30 14:37:39</div>
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
  <div class="_2_QraFYR_0">go get如何拉取指定分支的指定commit id</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 还不支持指定commit id get</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-24 00:49:27</div>
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
  <div class="_2_QraFYR_0">分析的真细致，厉害👍</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-10 20:56:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>biby</span>
  </div>
  <div class="_2_QraFYR_0">之前看了许多篇go module的文章，都是很碎片的知识，学习完也没有完整认知，导致用的时候都是模模糊糊。这篇讲的完整细致，能系统的学习，开发使用的时候清晰很多。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-04 16:31:24</div>
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
  <div class="_2_QraFYR_0">最后一个流程图中， GOSUMDB=off 的时候，还会和 go.sum 文件中的 hash 做对比吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不会的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-01 21:55:14</div>
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
  <div class="_2_QraFYR_0">开启 go module 的环境变量 GO1111MODULE 为什么中间有 3个1？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 应该是GO1.11的意思</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-01 08:17:18</div>
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
  <div class="_2_QraFYR_0">goland构建module并引用方包logrus，能go run代码，但是logrus.Println 中的（Println ）一直显示红色报错，尝试很多办法，解决<br>go构建约束问题，错误提示是Build constraints exclude all Go files in c：代码目录文件夹？<br>尝试以下办法解决不了<br>1、searcheverything 搜索后删除所有包,<br>$GOPATH目录下，把对应的包删除，重新go get,还是不行.<br>2、go get -u -v github.com&#47;karalabe&#47;xgo<br>3、Right click -&gt; Mark folder as not excluded.<br>4、引用包报错，重启电脑，查看goproxy配置，重装goland等等都解决不了。<br>怎么解决，希望得到老师帮助，谢谢。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 试试执行以下命令重新再构建下：<br>export CGO_ENABLED=1 <br>export GOOS=linux</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-18 23:16:57</div>
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
  <div class="_2_QraFYR_0">如果不提交 go.sum，会有什么风险？<br><br>当别人使用项目并下载依赖时，无法验证他们使用的依赖和你开发时的依赖是否一致，存在下载到被篡改代码的风险</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对的，是这个风险。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-28 09:34:00</div>
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
  <div class="_2_QraFYR_0">go build或者go run的时候，如果依赖包都已经下载到 pkg&#47;mod下了，还会进行校验吗？？<br>自己测试了下已经是不会校验的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不会校验的，缓存中的文件时只读的，所以只需要校验一次</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-24 01:21:37</div>
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
  <div class="_2_QraFYR_0">“如果不指定版本号，Go Modules 会根据自定义的规则，选择最小版本来下载。”，这里说的最小版本指的是latest版本吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不是。<br><br>语义化版本号之间是可以对比的，比如：v1.1.1 &lt; v1.1.2。<br><br>按语义化版本号对比规则对比后的最小值。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-10 21:00:13</div>
  </div>
</div>
</div>
</li>
</ul>