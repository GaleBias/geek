<audio title="07｜构建模式：Go Module的6类常规操作" src="https://static001.geekbang.org/resource/audio/2e/bd/2eyy47cb3c38740aaf7524263c25a7bd.mp3" controls="controls"></audio> 
<p>你好，我是Tony Bai。</p><p>通过上一节课的讲解，我们掌握了Go Module构建模式的基本概念和工作原理，也初步学会了如何通过go mod命令，将一个Go项目转变为一个Go Module，并通过Go Module构建模式进行构建。</p><p>但是，围绕一个Go Module，Go开发人员每天要执行很多Go命令对其进行维护。这些维护又是怎么进行的呢？</p><p>具体来说，维护Go Module 无非就是对Go Module 依赖包的管理。但在具体工作中还有很多情况，我们接下来会拆分成六个场景，层层深入给你分析。可以说，学好这些是每个Go开发人员成长的必经之路。</p><p>我们首先来看一下日常进行Go应用开发时遇到的最为频繁的一个场景：<strong>为当前项目添加一个依赖包</strong>。</p><h2>为当前module添加一个依赖</h2><p>在一个项目的初始阶段，我们会经常为项目引入第三方包，并借助这些包完成特定功能。即便是项目进入了稳定阶段，随着项目的演进，我们偶尔还需要在代码中引入新的第三方包。</p><p>那么我们如何为一个Go Module添加一个新的依赖包呢？</p><p>我们还是以上一节课中讲过的module-mode项目为例。如果我们要为这个项目增加一个新依赖：github.com/google/uuid，那需要怎么做呢？</p><!-- [[[read_end]]] --><p>我们首先会更新源码，就像下面代码中这样：</p><pre><code class="language-plain">package main

import (
	"github.com/google/uuid"
	"github.com/sirupsen/logrus"
)

func main() {
	logrus.Println("hello, go module mode")
	logrus.Println(uuid.NewString())
}
</code></pre><p>新源码中，我们通过import语句导入了github.com/google/uuid，并在main函数中调用了uuid包的函数NewString。此时，如果我们直接构建这个module，我们会得到一个错误提示：</p><pre><code class="language-plain">$go build
main.go:4:2: no required module provides package github.com/google/uuid; to add it:
	go get github.com/google/uuid
</code></pre><p>Go编译器提示我们，go.mod里的require段中，没有哪个module提供了github.com/google/uuid包，如果我们要增加这个依赖，可以手动执行go get命令。那我们就来按照提示手工执行一下这个命令：</p><pre><code class="language-plain">$go get github.com/google/uuid
go: downloading github.com/google/uuid v1.3.0
go get: added github.com/google/uuid v1.3.0
</code></pre><p>你会发现，go get命令将我们新增的依赖包下载到了本地module缓存里，并在go.mod文件的require段中新增了一行内容：</p><pre><code class="language-plain">require (
	github.com/google/uuid v1.3.0 //新增的依赖
	github.com/sirupsen/logrus v1.8.1
)
</code></pre><p>这新增的一行表明，我们当前项目依赖的是uuid的v1.3.0版本。我们也可以使用go mod tidy命令，在执行构建前自动分析源码中的依赖变化，识别新增依赖项并下载它们：</p><pre><code class="language-plain">$go mod tidy
go: finding module for package github.com/google/uuid
go: found github.com/google/uuid in github.com/google/uuid v1.3.0
</code></pre><p>对于我们这个例子而言，手工执行go get新增依赖项，和执行go mod tidy自动分析和下载依赖项的最终效果，是等价的。但对于复杂的项目变更而言，逐一手工添加依赖项显然很没有效率，go mod tidy是更佳的选择。</p><p>到这里，我们已经了解了怎么为当前的module添加一个新的依赖。但是在日常开发场景中，我们需要对依赖的版本进行更改。那这又要怎么做呢？下面我们就来看看下面升、降级修改依赖版本的场景。</p><h2>升级/降级依赖的版本</h2><p>我们先以对依赖的版本进行降级为例，分析一下。</p><p>在实际开发工作中，如果我们认为Go命令自动帮我们确定的某个依赖的版本存在一些问题，比如，引入了不必要复杂性导致可靠性下降、性能回退等等，我们可以手工将它降级为之前发布的某个兼容版本。</p><p>那这个操作依赖于什么原理呢？</p><p>答案就是我们上一节课讲过“语义导入版本”机制。我们再来简单复习一下，Go Module的版本号采用了语义版本规范，也就是版本号使用vX.Y.Z的格式。其中X是主版本号，Y为次版本号(minor)，Z为补丁版本号(patch)。主版本号相同的两个版本，较新的版本是兼容旧版本的。如果主版本号不同，那么两个版本是不兼容的。</p><p>有了语义版本号作为基础和前提，我们就可以从容地手工对依赖的版本进行升降级了，Go命令也可以根据版本兼容性，自动选择出合适的依赖版本了。</p><p>我们还是以上面提到过的logrus为例，logrus现在就存在着多个发布版本，我们可以通过下面命令来进行查询：</p><pre><code class="language-plain">$go list -m -versions github.com/sirupsen/logrus
github.com/sirupsen/logrus v0.1.0 v0.1.1 v0.2.0 v0.3.0 v0.4.0 v0.4.1 v0.5.0 v0.5.1 v0.6.0 v0.6.1 v0.6.2 v0.6.3 v0.6.4 v0.6.5 v0.6.6 v0.7.0 v0.7.1 v0.7.2 v0.7.3 v0.8.0 v0.8.1 v0.8.2 v0.8.3 v0.8.4 v0.8.5 v0.8.6 v0.8.7 v0.9.0 v0.10.0 v0.11.0 v0.11.1 v0.11.2 v0.11.3 v0.11.4 v0.11.5 v1.0.0 v1.0.1 v1.0.3 v1.0.4 v1.0.5 v1.0.6 v1.1.0 v1.1.1 v1.2.0 v1.3.0 v1.4.0 v1.4.1 v1.4.2 v1.5.0 v1.6.0 v1.7.0 v1.7.1 v1.8.0 v1.8.1
</code></pre><p>在这个例子中，基于初始状态执行的go mod tidy命令，帮我们选择了logrus的最新发布版本v1.8.1。如果你觉得这个版本存在某些问题，想将logrus版本降至某个之前发布的兼容版本，比如v1.7.0，<strong>那么我们可以在项目的module根目录下，执行带有版本号的go get命令：</strong></p><pre><code class="language-plain">$go get github.com/sirupsen/logrus@v1.7.0
go: downloading github.com/sirupsen/logrus v1.7.0
go get: downgraded github.com/sirupsen/logrus v1.8.1 =&gt; v1.7.0
</code></pre><p>从这个执行输出的结果，我们可以看到，go get命令下载了logrus v1.7.0版本，并将go.mod中对logrus的依赖版本从v1.8.1降至v1.7.0。</p><p>当然我们也可以使用万能命令go mod tidy来帮助我们降级，但前提是首先要用go mod edit命令，明确告知我们要依赖v1.7.0版本，而不是v1.8.1，这个执行步骤是这样的：</p><pre><code class="language-plain">$go mod edit -require=github.com/sirupsen/logrus@v1.7.0
$go mod tidy       
go: downloading github.com/sirupsen/logrus v1.7.0
</code></pre><p>降级后，我们再假设logrus v1.7.1版本是一个安全补丁升级，修复了一个严重的安全漏洞，而且我们必须使用这个安全补丁版本，这就意味着我们需要将logrus依赖从v1.7.0升级到v1.7.1。</p><p>我们可以使用与降级同样的步骤来完成升级，这里我只列出了使用go get实现依赖版本升级的命令和输出结果，你自己动手试一下。</p><pre><code class="language-plain">$go get github.com/sirupsen/logrus@v1.7.1
go: downloading github.com/sirupsen/logrus v1.7.1
go get: upgraded github.com/sirupsen/logrus v1.7.0 =&gt; v1.7.1
</code></pre><p>好了，到这里你就学会了如何对项目依赖包的版本进行升降级了。</p><p>但是你可能会发现一个问题，在前面的例子中，Go Module的依赖的主版本号都是1。根据我们上节课中学习的语义导入版本的规范，在Go Module构建模式下，当依赖的主版本号为0或1的时候，我们在Go源码中导入依赖包，不需要在包的导入路径上增加版本号，也就是：</p><pre><code class="language-plain">import github.com/user/repo/v0 等价于 import github.com/user/repo
import github.com/user/repo/v1 等价于 import github.com/user/repo
</code></pre><p>但是，如果我们要依赖的module的主版本号大于1，这又要怎么办呢？接着我们就来看看这个场景下该如何去做。</p><h2>添加一个主版本号大于1的依赖</h2><p>这里，我们还是先来回顾一下，上节课我们讲的语义版本规则中对主版本号大于1情况有没有相应的说明。</p><p>有的。语义导入版本机制有一个原则：<strong>如果新旧版本的包使用相同的导入路径，那么新包与旧包是兼容的</strong>。也就是说，如果新旧两个包不兼容，那么我们就应该采用不同的导入路径。</p><p>按照语义版本规范，如果我们要为项目引入主版本号大于1的依赖，比如v2.0.0，那么由于这个版本与v1、v0开头的包版本都不兼容，我们在导入v2.0.0包时，不能再直接使用github.com/user/repo，而要使用像下面代码中那样不同的包导入路径：</p><pre><code class="language-plain">import github.com/user/repo/v2/xxx
</code></pre><p>也就是说，如果我们要为Go项目添加主版本号大于1的依赖，我们就需要使用“语义导入版本”机制，<strong>在声明它的导入路径的基础上，加上版本号信息</strong>。我们以“向module-mode项目添加github.com/go-redis/redis依赖包的v7版本”为例，看看添加步骤。</p><p>首先，我们在源码中，以空导入的方式导入v7版本的github.com/go-redis/redis包：</p><pre><code class="language-plain">package main

import (
	_ "github.com/go-redis/redis/v7" //&nbsp;“_”为空导入
	"github.com/google/uuid"
	"github.com/sirupsen/logrus"
)

func main() {
	logrus.Println("hello, go module mode")
	logrus.Println(uuid.NewString())
}
</code></pre><p>接下来的步骤就与添加兼容依赖一样，我们通过go get获取redis的v7版本：</p><pre><code class="language-plain">$go get github.com/go-redis/redis/v7
go: downloading github.com/go-redis/redis/v7 v7.4.1
go: downloading github.com/go-redis/redis v6.15.9+incompatible
go get: added github.com/go-redis/redis/v7 v7.4.1
</code></pre><p>我们可以看到，go get为我们选择了go-redis v7版本下当前的最新版本v7.4.1。</p><p>不过呢，这里说的是为项目添加一个主版本号大于1的依赖的步骤。有些时候，出于要使用依赖包最新功能特性等原因，我们可能需要将某个依赖的版本升级为其不兼容版本，也就是主版本号不同的版本，这又该怎么做呢？</p><p>我们还以go-redis/redis这个依赖为例，将这个依赖从v7版本升级到最新的v8版本看看。</p><h2>升级依赖版本到一个不兼容版本</h2><p>我们前面说了，按照语义导入版本的原则，不同主版本的包的导入路径是不同的。所以，同样地，我们这里也需要先将代码中redis包导入路径中的版本号改为v8：</p><pre><code class="language-plain">import (
	_ "github.com/go-redis/redis/v8"
	"github.com/google/uuid"
	"github.com/sirupsen/logrus"
)
</code></pre><p>接下来，我们再通过go get来获取v8版本的依赖包：</p><pre><code class="language-plain">$go get github.com/go-redis/redis/v8
go: downloading github.com/go-redis/redis/v8 v8.11.1
go: downloading github.com/dgryski/go-rendezvous v0.0.0-20200823014737-9f7001d12a5f
go: downloading github.com/cespare/xxhash/v2 v2.1.1
go get: added github.com/go-redis/redis/v8 v8.11.1
</code></pre><p>这样，我们就完成了向一个不兼容依赖版本的升级。是不是很简单啊！</p><p>但是项目继续演化到一个阶段的时候，我们可能还需要移除对之前某个包的依赖。</p><h2>移除一个依赖</h2><p>我们还是看前面go-redis/redis示例，如果我们这个时候不需要再依赖go-redis/redis了，你会怎么做呢？</p><p>你可能会删除掉代码中对redis的空导入这一行，之后再利用go build命令成功地构建这个项目。</p><p>但你会发现，与添加一个依赖时Go命令给出友好提示不同，这次go build没有给出任何关于项目已经将go-redis/redis删除的提示，并且go.mod里require段中的go-redis/redis/v8的依赖依旧存在着。</p><p>我们再通过go list命令列出当前module的所有依赖，你也会发现go-redis/redis/v8仍出现在结果中：</p><pre><code class="language-plain">$go list -m all
github.com/bigwhite/module-mode
github.com/cespare/xxhash/v2 v2.1.1
github.com/davecgh/go-spew v1.1.1
... ...
github.com/go-redis/redis/v8 v8.11.1
... ...
gopkg.in/yaml.v2 v2.3.0
</code></pre><p>这是怎么回事呢？</p><p>其实，要想彻底从项目中移除go.mod中的依赖项，仅从源码中删除对依赖项的导入语句还不够。这是因为如果源码满足成功构建的条件，go build命令是不会“多管闲事”地清理go.mod中多余的依赖项的。</p><p>那正确的做法是怎样的呢？我们还得用go mod tidy命令，将这个依赖项彻底从Go Module构建上下文中清除掉。go mod tidy会自动分析源码依赖，而且将不再使用的依赖从go.mod和go.sum中移除。</p><p>到这里，其实我们已经分析了Go Module依赖包管理的5个常见情况了，但其实还有一种特殊情况，需要我们借用vendor机制。</p><h2>特殊情况：使用vendor</h2><p>你可能会感到有点奇怪，为什么Go Module的维护，还有要用vendor的情况？</p><p>其实，vendor机制虽然诞生于GOPATH构建模式主导的年代，但在Go Module构建模式下，它依旧被保留了下来，并且成为了Go Module构建机制的一个很好的补充。特别是在一些不方便访问外部网络，并且对Go应用构建性能敏感的环境，比如在一些内部的持续集成或持续交付环境（CI/CD）中，使用vendor机制可以实现与Go Module等价的构建。</p><p>和GOPATH构建模式不同，Go Module构建模式下，我们再也无需手动维护vendor目录下的依赖包了，Go提供了可以快速建立和更新vendor的命令，我们还是以前面的module-mode项目为例，通过下面命令为该项目建立vendor：</p><pre><code class="language-plain">$go mod vendor
$tree -LF 2 vendor
vendor
├── github.com/
│&nbsp;&nbsp; ├── google/
│&nbsp;&nbsp; ├── magefile/
│&nbsp;&nbsp; └── sirupsen/
├── golang.org/
│&nbsp;&nbsp; └── x/
└── modules.txt
</code></pre><p>我们看到，go mod vendor命令在vendor目录下，创建了一份这个项目的依赖包的副本，并且通过vendor/modules.txt记录了vendor下的module以及版本。</p><p>如果我们要基于vendor构建，而不是基于本地缓存的Go Module构建，我们需要在go build后面加上-mod=vendor参数。</p><p>在Go 1.14及以后版本中，如果Go项目的顶层目录下存在vendor目录，那么go build默认也会优先基于vendor构建，除非你给go build传入-mod=mod的参数。</p><h2>小结</h2><p>好了，到这里，我们就完成了维护Go Module的全部常见场景的学习了，现在我们一起来回顾一下吧。</p><p>在通过go mod init为当前Go项目创建一个新的module后，随着项目的演进，我们在日常开发过程中，会遇到多种常见的维护Go Module的场景。</p><p>其中最常见的就是为项目添加一个依赖包，我们可以通过go get命令手工获取该依赖包的特定版本，更好的方法是通过go mod tidy命令让Go命令自动去分析新依赖并决定使用新依赖的哪个版本。</p><p>另外，还有几个场景需要你记住：</p><ul>
<li>通过go get我们可以升级或降级某依赖的版本，如果升级或降级前后的版本不兼容，这里千万注意别忘了变化包导入路径中的版本号，这是Go语义导入版本机制的要求；</li>
<li>通过go mod tidy，我们可以自动分析Go源码的依赖变更，包括依赖的新增、版本变更以及删除，并更新go.mod中的依赖信息。</li>
<li>通过go mod vendor，我们依旧可以支持vendor机制，并且可以对vendor目录下缓存的依赖包进行自动管理。</li>
</ul><p>在了解了如何应对Go Modules维护的日常工作场景后，你是不是有一种再也不担心Go源码构建问题的感觉了呢？</p><h2>思考题</h2><p>如果你是一个公共Go包的作者，在发布你的Go包时，有哪些需要注意的地方？</p><p>感谢你和我一起学习，也欢迎你把这节课分享给更多对Go构建模式感兴趣的朋友。我是Tony Bai，我们下节课见。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/9d/a4/e481ae48.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lesserror</span>
  </div>
  <div class="_2_QraFYR_0">Tony Bai 老师这一讲的内容很实用，可以说有很多Go教程都没有涉及到这块知识的归纳总结。<br>麻烦老师抽空回答一下我以下的疑问：<br><br>1. 空导入的方式的作用吗？我看很多源码中有使用这种包导入的方式。<br><br>2. 在go module构建模式下，怎么对vendor目录的有无进行取舍呢？老师有什么实战建议呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 问题1：<br><br>像下面代码这样的包导入方式被称为“空导入”：<br>                   <br>import _ &quot;foo&quot;                                                                               <br>                                                                                       <br>空导入也是导入，意味着我们将依赖foo这个路径下的包。但由于是空导入，我们并没有显式使用这个包中的任何语法元素。那么空导入的意义是什么呢？由于依赖foo包，程序初始化的时候会沿着包的依赖链初始化foo包，我们在08里会讲到包的初始化会按照常量-&gt;变量-&gt;init函数的次序进行。通常实践中空导入意味着期望依赖包的init函数得到执行，这个init函数中有我们需要的逻辑。<br><br>问题2：<br>通常我们直接使用go module(非vendor)模式即可满足大部分需求。如果是那种开发环境受限，因无法访问外部代理而无法通过go命令自动解决依赖和下载依赖的环境下，我们通过vendor来辅助解决。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-27 17:28:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/8d/ed/bb32e906.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>blur</span>
  </div>
  <div class="_2_QraFYR_0">go mod edit -require=github.com&#47;sirupsen&#47;logrus@v1.7.0这个指令在win 上的golangd好像会因github 后面的那个 . 识别不出来path,加引号变成 go mod edit -require=&quot;github.com&#47;sirupsen&#47;logrus@v1.7.0&quot;就可以了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢提供不同平台的差异。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-28 16:06:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/2d/1e/4a93ebb5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Aaron Liu</span>
  </div>
  <div class="_2_QraFYR_0">如果之前引用的包是v1，之后升级v2，go get可以替换引用的包，但源码里的import要怎么改，如果很多go文件都引用了呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 源码中必须要改，改为xxx&#47;v2。这个要么手动改，要么使用IDE&#47;编辑器提供的工具进行统一替换。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-27 08:34:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/53/a8/abc96f70.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>return</span>
  </div>
  <div class="_2_QraFYR_0">老师讲太好了， 有主线 有关键细节，<br>请教老师， 关于vendor， 存好副本后， 一般在其他地方怎么用呢，<br>手动传输过去 还是 上传到代码库再下载呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果采用vendor模式，建议与项目代码同等对待，一并上传的代码仓库中。其他地方直接下载使用。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-27 12:38:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/16/5c/c0322969.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Howe</span>
  </div>
  <div class="_2_QraFYR_0">老师这个专栏绝了，真的收获很大！<br>老师，想请教个问题，在一些无法连接外网的环境下，Go Module有没有类似maven和Nexus一样可以搭建自己的私库，然后私库去连接外部代理去下载依赖？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 目前私有代理做的比较好的有goproxy.io、goproxy.cn、athen等。我在后面的加餐中会聊到这个话题。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-22 17:49:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/f4/0a/616e3532.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>女干部</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，<br>有一个疑问困扰我很久了，这样一个例子:<br>安装 go get -u github.com&#47;cweill&#47;gotests&#47;...<br>然后就可以在命令行里执行 gotests了，<br>我想知道&#47;...这是个什么写法，<br>还有gotests.exe，是怎么构建并被放到我的%USERPROFILE%\go\bin目录下的<br>辛苦</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. ...表示gotests下面的所有的包 2. go get会下载gotests下面所有的包，如果gotests是一个可执行文件的项目（带有main包main函数）. go get会在下载包之后构建这个项目并把可执行文件放入$GOPATH&#47;bin下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-01 10:42:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/7b/bd/ccb37425.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>进化菌</span>
  </div>
  <div class="_2_QraFYR_0">感觉go mod tidy很常用，通过例子来学习的感觉也挺好的。<br>还有，vendor保留下来有它的道理吧，每个项目就应该管自己的依赖~</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-28 20:38:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/9d/a4/e481ae48.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lesserror</span>
  </div>
  <div class="_2_QraFYR_0">大白老师，如果我想升级go.mod中定义的Go版本的话，最佳实践是不是这么操作：<br><br>go mod edit -go=1.17<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个操作我觉得没啥最佳实践，我一般直接用vim打开go.mod文件，然后改就是了:)。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-17 18:14:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/c8/4a/3a322856.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ll</span>
  </div>
  <div class="_2_QraFYR_0">我是一名前端，初“卷”到go，对比 go module 对比 npm （node 的包管理）：<br>1. vendor 类似于 node 项目中的 node_modules,<br>2. 默认条件下用 go get xxx 相当于 npm i -g xxx，<br>总之，我的方法就是结合新学的内容，和我熟悉的其他语言体系做对比；这样一是方便记忆，二可以更好的理解新知识。<br>老师的课条理清晰，深入浅出，点赞</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 欢迎来到go世界。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-17 20:28:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2b/13/2f/24420ab6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jacky</span>
  </div>
  <div class="_2_QraFYR_0">讲得挺好，就是更新有点慢啊，这得更到啥时候</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 一周三篇，已经很快了:)。写稿不易，欢迎继续、持续支持！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-28 09:29:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/57/1e/8ed4a7cf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Paradise</span>
  </div>
  <div class="_2_QraFYR_0">Tony 的专栏不适合跳着看，因为细节干货太多啦哈哈，感谢老师</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍。嗯嗯，尽量按顺序看。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-09 14:48:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>woJA1wCgAApKZLcyM5n8DSoPyMkMZk5A</span>
  </div>
  <div class="_2_QraFYR_0">老师，请问下replace和go vendor应该如何选择呢？比如引用一个第三方依赖包，但需要对其部分内容进行自定义修改，那么是使用vendor机制将其下载下来后修改还是使用replace替换那个官方的依赖包呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果仅仅是个人开发，replace就够了，vendor亦可。但如果是协作开发，必然vendor啊，否则同组其他童鞋下载了你的代码后还得修改replace的本地路径。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-02 18:30:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1c/f7/5c/b476d429.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>BWM</span>
  </div>
  <div class="_2_QraFYR_0">老师讲的真好！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-04 21:49:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/29/bb/e0/c7cd5170.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Bynow</span>
  </div>
  <div class="_2_QraFYR_0">公共go包的作者应该比较关注 包的tag 和 go mod init 后面的v(版本)吧。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对的。Go包作者需要严格按照语义版本规则判断版本间的兼容性。当升级为不兼容版本时，要注意go.mod中module path的随动变化。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-27 19:48:08</div>
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
  <div class="_2_QraFYR_0">go 的命令好多，老师来讲讲 使用 Makefile 来管理的方法</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 手动为难。这个专栏focus的是Go语言入门。只有Makefile怎么用，建议找找相关专栏或在网络上寻找一些好资料。如果遇到难题，可以以具体的问题的形式在本专栏内留言，如果我能解答，自然会帮助大家的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-27 11:58:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/2a/36/c5d1a120.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>CLMOOK🐾</span>
  </div>
  <div class="_2_QraFYR_0">老师好，如果用go get给一个依赖包降为低一个次版本的，再跑go mod tidy是否会自动把这个依赖包升级成之前的新版本？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 应该不会。你降级依赖包A时，就会将A的相关依赖重新分析一遍，该降级的都会降级。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-17 19:02:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/d8/10/5173922c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>泽韦德</span>
  </div>
  <div class="_2_QraFYR_0">老师，当前Go Module自身的版本号怎么设置的，是不是没讲？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 版本号绝大多数都是通过仓库打tag的方式啊。tag格式：vx.y.z。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-01 20:14:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/15/8e/8fc00a53.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>🐎</span>
  </div>
  <div class="_2_QraFYR_0">总结就是 go mod tidy 最好使</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-21 15:13:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/72/65/68bd8177.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>雪飞鸿</span>
  </div>
  <div class="_2_QraFYR_0">go包管理这两节课学下来，切身体会到了go包管理不友好</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-02 14:33:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/3a/8d/f5e7a20d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>何以解忧</span>
  </div>
  <div class="_2_QraFYR_0">vendor 模式和 module 模式，互相有影响么，比如顶级目录下面，有go.mod 同时有vendor 目录。 vendor 下的modules.txt 和go.mod 可以理解为两种模式的类似的定位么，记录版本</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: vendor目录下的内容均是基于go.mod生成的，包括modules.txt。理论上modules.txt中依赖项与版本要与go.mod中的保持一致。<br><br>vendor可以理解为将项目依赖在项目下保存一份。<br><br>此外，文中最后也有提到：“在 Go 1.14 及以后版本中，如果 Go 项目的顶层目录下存在 vendor 目录，那么 go build 默认也会优先基于 vendor 构建，除非你给 go build 传入 -mod=mod 的参数。”。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-23 22:07:14</div>
  </div>
</div>
</div>
</li>
</ul>