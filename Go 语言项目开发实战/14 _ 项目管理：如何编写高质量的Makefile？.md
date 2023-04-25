<audio title="14 _ 项目管理：如何编写高质量的Makefile？" src="https://static001.geekbang.org/resource/audio/b6/16/b676dda72c61a595148bb9c2fda7fa16.mp3" controls="controls"></audio> 
<p>你好，我是孔令飞。今天我们来聊聊如何编写高质量的Makefile。</p><p>我们在 <a href="https://time.geekbang.org/column/article/384648">第10讲</a> 学习过，要写出一个优雅的Go项目，不仅仅是要开发一个优秀的Go应用，而且还要能够高效地管理项目。有效手段之一，就是通过Makefile来管理我们的项目，这就要求我们要为项目编写Makefile文件。</p><p>在和其他开发同学交流时，我发现大家都认可Makefile强大的项目管理能力，也会自己编写Makefile。但是其中的一些人项目管理做得并不好，我和他们进一步交流后发现，这些同学在用Makefile简单的语法重复编写一些低质量Makefile文件，根本没有把Makefile的功能充分发挥出来。</p><p>下面给你举个例子，你就会理解低质量的Makefile文件是什么样的了。</p><pre><code>build: clean vet
	@mkdir -p ./Role
	@export GOOS=linux &amp;&amp; go build -v .

vet:
	go vet ./...

fmt:
	go fmt ./...

clean:
	rm -rf dashboard
</code></pre><p>上面这个Makefile存在不少问题。例如：功能简单，只能完成最基本的编译、格式化等操作，像构建镜像、自动生成代码等一些高阶的功能都没有；扩展性差，没法编译出可在Mac下运行的二进制文件；没有Help功能，使用难度高；单Makefile文件，结构单一，不适合添加一些复杂的管理功能。</p><p>所以，我们不光要编写Makefile，还要编写高质量的Makefile。那么如何编写一个高质量的Makefile呢？我觉得，可以通过以下4个方法来实现：</p><!-- [[[read_end]]] --><ol>
<li>打好基础，也就是熟练掌握Makefile的语法。</li>
<li>做好准备工作，也就是提前规划Makefile要实现的功能。</li>
<li>进行规划，设计一个合理的Makefile结构。</li>
<li>掌握方法，用好Makefile的编写技巧。</li>
</ol><p>那么接下来，我们就详细看看这些方法。</p><h2>熟练掌握Makefile语法</h2><p>工欲善其事，必先利其器。编写高质量Makefile的第一步，便是熟练掌握Makefile的核心语法。</p><p>因为Makefile的语法比较多，我把一些建议你重点掌握的语法放在了近期会更新的特别放送中，包括Makefile规则语法、伪目标、变量赋值、条件语句和Makefile常用函数等等。</p><p>如果你想更深入、全面地学习Makefile的语法，我推荐你学习陈皓老师编写的<a href="https://github.com/seisman/how-to-write-makefile">《跟我一起写 Makefile》 (PDF 重制版)</a>。</p><h2>规划Makefile要实现的功能</h2><p>接着，我们需要规划Makefile要实现的功能。提前规划好功能，有利于你设计Makefile的整体结构和实现方法。</p><p>不同项目拥有不同的Makefile功能，这些功能中一小部分是通过目标文件来实现的，但更多的功能是通过伪目标来实现的。对于Go项目来说，虽然不同项目集成的功能不一样，但绝大部分项目都需要实现一些通用的功能。接下来，我们就来看看，在一个大型Go项目中Makefile通常可以实现的功能。</p><p>下面是IAM项目的Makefile所集成的功能，希望会对你日后设计Makefile有一些帮助。</p><pre><code>$ make help

Usage: make &lt;TARGETS&gt; &lt;OPTIONS&gt; ...

Targets:
  # 代码生成类命令
  gen                Generate all necessary files, such as error code files.

  # 格式化类命令
  format             Gofmt (reformat) package sources (exclude vendor dir if existed).

  # 静态代码检查
  lint               Check syntax and styling of go sources.

  # 测试类命令
  test               Run unit test.
  cover              Run unit test and get test coverage.

  # 构建类命令
  build              Build source code for host platform.
  build.multiarch    Build source code for multiple platforms. See option PLATFORMS.

  # Docker镜像打包类命令
  image              Build docker images for host arch.
  image.multiarch    Build docker images for multiple platforms. See option PLATFORMS.
  push               Build docker images for host arch and push images to registry.
  push.multiarch     Build docker images for multiple platforms and push images to registry.

  # 部署类命令
  deploy             Deploy updated components to development env.

  # 清理类命令
  clean              Remove all files that are created by building.

  # 其他命令，不同项目会有区别
  release            Release iam
  verify-copyright   Verify the boilerplate headers for all files.
  ca                 Generate CA files for all iam components.
  install            Install iam system with all its components.
  swagger            Generate swagger document.
  tools              install dependent tools.

  # 帮助命令
  help               Show this help info.

# 选项
Options:
  DEBUG        Whether to generate debug symbols. Default is 0.
  BINS         The binaries to build. Default is all of cmd.
               This option is available when using: make build/build.multiarch
               Example: make build BINS=&quot;iam-apiserver iam-authz-server&quot;
  ...
</code></pre><p>更详细的命令，你可以在IAM项目仓库根目录下执行<code>make help</code>查看。</p><p>通常而言，Go项目的Makefile应该实现以下功能：格式化代码、静态代码检查、单元测试、代码构建、文件清理、帮助等等。如果通过docker部署，还需要有docker镜像打包功能。因为Go是跨平台的语言，所以构建和docker打包命令，还要能够支持不同的CPU架构和平台。为了能够更好地控制Makefile命令的行为，还需要支持Options。</p><p>为了方便查看Makefile集成了哪些功能，我们需要支持help命令。help命令最好通过解析Makefile文件来输出集成的功能，例如：</p><pre><code>## help: Show this help info.
.PHONY: help
help: Makefile
  @echo -e &quot;\nUsage: make &lt;TARGETS&gt; &lt;OPTIONS&gt; ...\n\nTargets:&quot;
  @sed -n 's/^##//p' $&lt; | column -t -s ':' | sed -e 's/^/ /'
  @echo &quot;$$USAGE_OPTIONS&quot;
</code></pre><p>上面的help命令，通过解析Makefile文件中的<code>##</code>注释，获取支持的命令。通过这种方式，我们以后新加命令时，就不用再对help命令进行修改了。</p><p>你可以参考上面的Makefile管理功能，结合自己项目的需求，整理出一个Makefile要实现的功能列表，并初步确定实现思路和方法。做完这些，你的编写前准备工作就基本完成了。</p><h2>设计合理的Makefile结构</h2><p>设计完Makefile需要实现的功能，接下来我们就进入Makefile编写阶段。编写阶段的第一步，就是设计一个合理的Makefile结构。</p><p>对于大型项目来说，需要管理的内容很多，所有管理功能都集成在一个Makefile中，可能会导致Makefile很大，难以阅读和维护，所以<strong>建议采用分层的设计方法，根目录下的Makefile聚合所有的Makefile命令，具体实现则按功能分类，放在另外的Makefile中</strong>。</p><p>我们经常会在Makefile命令中集成shell脚本，但如果shell脚本过于复杂，也会导致Makefile内容过多，难以阅读和维护。并且在Makefile中集成复杂的shell脚本，编写体验也很差。对于这种情况，<strong>可以将复杂的shell命令封装在shell脚本中，供Makefile直接调用，而一些简单的命令则可以直接集成在Makefile中</strong>。</p><p>所以，最终我推荐的Makefile结构如下：</p><p><img src="https://static001.geekbang.org/resource/image/5c/f7/5c524e0297b6d6e4e151643d2e1bbbf7.png?wh=2575x1017" alt=""></p><p>在上面的Makefile组织方式中，根目录下的Makefile聚合了项目所有的管理功能，这些管理功能通过Makefile伪目标的方式实现。同时，还将这些伪目标进行分类，把相同类别的伪目标放在同一个Makefile中，这样可以使得Makefile更容易维护。对于复杂的命令，则编写成独立的shell脚本，并在Makefile命令中调用这些shell脚本。</p><p>举个例子，下面是IAM项目的Makefile组织结构：</p><pre><code>├── Makefile
├── scripts
│   ├── gendoc.sh
│   ├── make-rules
│   │   ├── gen.mk
│   │   ├── golang.mk
│   │   ├── image.mk
│   │   └── ...
    └── ...
</code></pre><p>我们将相同类别的操作统一放在scripts/make-rules目录下的Makefile文件中。Makefile的文件名参考分类命名，例如 golang.mk。最后，在/Makefile 中 include 这些 Makefile。</p><p>为了跟Makefile的层级相匹配，golang.mk中的所有目标都按<code>go.xxx</code>这种方式命名。通过这种命名方式，我们可以很容易分辨出某个目标完成什么功能，放在什么文件里，这在复杂的Makefile中尤其有用。以下是IAM项目根目录下，Makefile的内容摘录，你可以看一看，作为参考：</p><pre><code>include scripts/make-rules/golang.mk
include scripts/make-rules/image.mk
include scripts/make-rules/gen.mk
include scripts/make-rules/...

## build: Build source code for host platform.
.PHONY: build
build:
	@$(MAKE) go.build

## build.multiarch: Build source code for multiple platforms. See option PLATFORMS.
.PHONY: build.multiarch
build.multiarch:
	@$(MAKE) go.build.multiarch

## image: Build docker images for host arch.
.PHONY: image
image:
	@$(MAKE) image.build

## push: Build docker images for host arch and push images to registry.
.PHONY: push
push:
	@$(MAKE) image.push

## ca: Generate CA files for all iam components.
.PHONY: ca
ca:
	@$(MAKE) gen.ca
</code></pre><p>另外，一个合理的Makefile结构应该具有前瞻性。也就是说，要在不改变现有结构的情况下，接纳后面的新功能。这就需要你整理好Makefile当前要实现的功能、即将要实现的功能和未来可能会实现的功能，然后基于这些功能，利用Makefile编程技巧，编写可扩展的Makefile。</p><p>这里需要你注意：上面的Makefile通过 <code>.PHONY</code> 标识定义了大量的伪目标，定义伪目标一定要加 <code>.PHONY</code> 标识，否则当有同名的文件时，伪目标可能不会被执行。</p><h2>掌握Makefile编写技巧</h2><p>最后，在编写过程中，你还需要掌握一些Makefile的编写技巧，这些技巧可以使你编写的Makefile扩展性更强，功能更强大。</p><p>接下来，我会把自己长期开发过程中积累的一些Makefile编写经验分享给你。这些技巧，你需要在实际编写中多加练习，并形成编写习惯。</p><h3>技巧1：善用通配符和自动变量</h3><p>Makefile允许对目标进行类似正则运算的匹配，主要用到的通配符是<code>%</code>。通过使用通配符，可以使不同的目标使用相同的规则，从而使Makefile扩展性更强，也更简洁。</p><p>我们的IAM实战项目中，就大量使用了通配符<code>%</code>，例如：<code>go.build.%</code>、<code>ca.gen.%</code>、<code>deploy.run.%</code>、<code>tools.verify.%</code>、<code>tools.install.%</code>等。</p><p>这里，我们来看一个具体的例子，<code>tools.verify.%</code>（位于<a href="https://github.com/marmotedu/iam/blob/master/scripts/make-rules/tools.mk#L17">scripts/make-rules/tools.mk</a>文件中）定义如下：</p><pre><code>tools.verify.%:
  @if ! which $* &amp;&gt;/dev/null; then $(MAKE) tools.install.$*; fi
</code></pre><p><code>make tools.verify.swagger</code>, <code>make tools.verify.mockgen</code>等均可以使用上面定义的规则，<code>%</code>分别代表了<code>swagger</code>和<code>mockgen</code>。</p><p>如果不使用<code>%</code>，则我们需要分别为<code>tools.verify.swagger</code>和<code>tools.verify.mockgen</code>定义规则，很麻烦，后面修改也困难。</p><p>另外，这里也能看出<code>tools.verify.%</code>这种命名方式的好处：tools说明依赖的定义位于<code>scripts/make-rules/tools.mk</code> Makefile中；<code>verify</code>说明<code>tools.verify.%</code>伪目标属于verify分类，主要用来验证工具是否安装。通过这种命名方式，你可以很容易地知道目标位于哪个Makefile文件中，以及想要完成的功能。</p><p>另外，上面的定义中还用到了自动变量<code>$*</code>，用来指代被匹配的值<code>swagger</code>、<code>mockgen</code>。</p><h3>技巧2：善用函数</h3><p>Makefile自带的函数能够帮助我们实现很多强大的功能。所以，在我们编写Makefile的过程中，如果有功能需求，可以优先使用这些函数。我把常用的函数以及它们实现的功能整理在了 <a href="https://github.com/marmotedu/geekbang-go/blob/master/makefile/Makefile%E5%B8%B8%E7%94%A8%E5%87%BD%E6%95%B0%E5%88%97%E8%A1%A8.md">Makefile常用函数列表</a> 中，你可以参考下。</p><p>IAM的Makefile文件中大量使用了上述函数，如果你想查看这些函数的具体使用方法和场景，可以参考IAM项目的Makefile文件 <a href="https://github.com/marmotedu/iam/tree/master/scripts/make-rules">make-rules</a>。</p><h3>技巧3：依赖需要用到的工具</h3><p>如果Makefile某个目标的命令中用到了某个工具，可以将该工具放在目标的依赖中。这样，当执行该目标时，就可以指定检查系统是否安装该工具，如果没有安装则自动安装，从而实现更高程度的自动化。例如，/Makefile文件中，format伪目标，定义如下：</p><pre><code>.PHONY: format
format: tools.verify.golines tools.verify.goimports
  @echo &quot;===========&gt; Formating codes&quot;
  @$(FIND) -type f -name '*.go' | $(XARGS) gofmt -s -w
  @$(FIND) -type f -name '*.go' | $(XARGS) goimports -w -local $(ROOT_PACKAGE)
  @$(FIND) -type f -name '*.go' | $(XARGS) golines -w --max-len=120 --reformat-tags --shorten-comments --ignore-generated .
</code></pre><p>你可以看到，format依赖<code>tools.verify.golines tools.verify.goimports</code>。我们再来看下<code>tools.verify.golines</code>的定义：</p><pre><code>tools.verify.%:
  @if ! which $* &amp;&gt;/dev/null; then $(MAKE) tools.install.$*; fi
</code></pre><p>再来看下<code>tools.install.$*</code>规则：</p><pre><code>.PHONY: install.golines
install.golines:
  @$(GO) get -u github.com/segmentio/golines
</code></pre><p>通过<code>tools.verify.%</code>规则定义，我们可以知道，<code>tools.verify.%</code>会先检查工具是否安装，如果没有安装，就会执行<code>tools.install.$*</code>来安装。如此一来，当我们执行<code>tools.verify.%</code>目标时，如果系统没有安装golines命令，就会自动调用<code>go get</code>安装，提高了Makefile的自动化程度。</p><h3>技巧4：把常用功能放在/Makefile中，不常用的放在分类Makefile中</h3><p>一个项目，尤其是大型项目，有很多需要管理的地方，其中大部分都可以通过Makefile实现自动化操作。不过，为了保持/Makefile文件的整洁性，我们不能把所有的命令都添加在/Makefile文件中。</p><p>一个比较好的建议是，将常用功能放在/Makefile中，不常用的放在分类Makefile中，并在/Makefile中include这些分类Makefile。</p><p>例如，IAM项目的/Makefile集成了<code>format</code>、<code>lint</code>、<code>test</code>、<code>build</code>等常用命令，而将<code>gen.errcode.code</code>、<code>gen.errcode.doc</code>这类不常用的功能放在scripts/make-rules/gen.mk文件中。当然，我们也可以直接执行 <code>make gen.errcode.code</code>来执行<code>gen.errcode.code</code>伪目标。通过这种方式，既可以保证/Makefile的简洁、易维护，又可以通过<code>make</code>命令来运行伪目标，更加灵活。</p><h3>技巧5：编写可扩展的Makefile</h3><p>什么叫可扩展的Makefile呢？在我看来，可扩展的Makefile包含两层含义：</p><ol>
<li>可以在不改变Makefile结构的情况下添加新功能。</li>
<li>扩展项目时，新功能可以自动纳入到Makefile现有逻辑中。</li>
</ol><p>其中的第一点，我们可以通过设计合理的Makefile结构来实现。要实现第二点，就需要我们在编写Makefile时采用一定的技巧，例如多用通配符、自动变量、函数等。这里我们来看一个例子，可以让你更好地理解。</p><p>在我们IAM实战项目的<a href="https://github.com/marmotedu/iam/blob/v1.0.0/scripts/make-rules/golang.mk#L34">golang.mk</a>中，执行 <code>make go.build</code> 时能够构建cmd/目录下的所有组件，也就是说，当有新组件添加时， <code>make go.build</code> 仍然能够构建新增的组件，这就实现了上面说的第二点。</p><p>具体实现方法如下：</p><pre><code>COMMANDS ?= $(filter-out %.md, $(wildcard ${ROOT_DIR}/cmd/*))
BINS ?= $(foreach cmd,${COMMANDS},$(notdir ${cmd}))

.PHONY: go.build
go.build: go.build.verify $(addprefix go.build., $(addprefix $(PLATFORM)., $(BINS)))
.PHONY: go.build.%               

go.build.%:             
  $(eval COMMAND := $(word 2,$(subst ., ,$*)))
  $(eval PLATFORM := $(word 1,$(subst ., ,$*)))
  $(eval OS := $(word 1,$(subst _, ,$(PLATFORM))))           
  $(eval ARCH := $(word 2,$(subst _, ,$(PLATFORM))))                         
  @echo &quot;===========&gt; Building binary $(COMMAND) $(VERSION) for $(OS) $(ARCH)&quot;
  @mkdir -p $(OUTPUT_DIR)/platforms/$(OS)/$(ARCH)
  @CGO_ENABLED=0 GOOS=$(OS) GOARCH=$(ARCH) $(GO) build $(GO_BUILD_FLAGS) -o $(OUTPUT_DIR)/platforms/$(OS)/$(ARCH)/$(COMMAND)$(GO_OUT_EXT) $(ROOT_PACKAGE)/cmd/$(COMMAND)
</code></pre><p>当执行<code>make go.build</code> 时，会执行go.build的依赖 <code>$(addprefix go.build., $(addprefix $(PLATFORM)., $(BINS)))</code> ,<code>addprefix</code>函数最终返回字符串 <code>go.build.linux_amd64.iamctl go.build.linux_amd64.iam-authz-server go.build.linux_amd64.iam-apiserver ...</code> ，这时候就会执行 <code>go.build.%</code> 伪目标。</p><p>在 <code>go.build.%</code> 伪目标中，通过eval、word、subst函数组合，算出了COMMAND的值 <code>iamctl/iam-apiserver/iam-authz-server/...</code>，最终通过 <code>$(ROOT_PACKAGE)/cmd/$(COMMAND)</code> 定位到需要构建的组件的main函数所在目录。</p><p>上述实现中有两个技巧，你可以注意下。首先，通过</p><pre><code>COMMANDS ?= $(filter-out %.md, $(wildcard ${ROOT_DIR}/cmd/*))
BINS ?= $(foreach cmd,${COMMANDS},$(notdir ${cmd}))
</code></pre><p>获取到了cmd/目录下的所有组件名。</p><p>接着，通过使用通配符和自动变量，自动匹配到<code>go.build.linux_amd64.iam-authz-server</code> 这类伪目标并构建。</p><p>可以看到，想要编写一个可扩展的Makefile，熟练掌握Makefile的用法是基础，更多的是需要我们动脑思考如何去编写Makefile。</p><h3>技巧6：将所有输出存放在一个目录下，方便清理和查找</h3><p>在执行Makefile的过程中，会输出各种各样的文件，例如 Go 编译后的二进制文件、测试覆盖率数据等，我建议你把这些文件统一放在一个目录下，方便后期的清理和查找。通常我们可以把它们放在<code>_output</code>这类目录下，这样清理时就很方便，只需要清理<code>_output</code>文件夹就可以，例如：</p><pre><code>.PHONY: go.clean
go.clean:
  @echo &quot;===========&gt; Cleaning all build output&quot;
  @-rm -vrf $(OUTPUT_DIR)
</code></pre><p>这里要注意，要用<code>-rm</code>，而不是<code>rm</code>，防止在没有<code>_output</code>目录时，执行<code>make go.clean</code>报错。</p><h3>技巧7：使用带层级的命名方式</h3><p>通过使用带层级的命名方式，例如<code>tools.verify.swagger</code> ，我们可以实现<strong>目标分组管理</strong>。这样做的好处有很多。首先，当Makefile有大量目标时，通过分组，我们可以更好地管理这些目标。其次，分组也能方便理解，可以通过组名一眼识别出该目标的功能类别。最后，这样做还可以大大减小目标重名的概率。</p><p>例如，IAM项目的Makefile就大量采用了下面这种命名方式。</p><pre><code>.PHONY: gen.run
gen.run: gen.clean gen.errcode gen.docgo

.PHONY: gen.errcode
gen.errcode: gen.errcode.code gen.errcode.doc

.PHONY: gen.errcode.code
gen.errcode.code: tools.verify.codegen
    ...
.PHONY: gen.errcode.doc
gen.errcode.doc: tools.verify.codegen
    ...
</code></pre><h3>技巧8：做好目标拆分</h3><p>还有一个比较实用的技巧：我们要合理地拆分目标。比如，我们可以将安装工具拆分成两个目标：验证工具是否已安装和安装工具。通过这种方式，可以给我们的Makefile带来更大的灵活性。例如：我们可以根据需要选择性地执行其中一个操作，也可以两个操作一起执行。</p><p>这里来看一个例子：</p><pre><code>gen.errcode.code: tools.verify.codegen

tools.verify.%:    
  @if ! which $* &amp;&gt;/dev/null; then $(MAKE) tools.install.$*; fi  

.PHONY: install.codegen
install.codegen:              
  @$(GO) install ${ROOT_DIR}/tools/codegen/codegen.go
</code></pre><p>上面的Makefile中，gen.errcode.code依赖了tools.verify.codegen，tools.verify.codegen会先检查codegen命令是否存在，如果不存在，再调用install.codegen来安装codegen工具。</p><p>如果我们的Makefile设计是：</p><pre><code>gen.errcode.code: install.codegen
</code></pre><p>那每次执行gen.errcode.code都要重新安装codegen命令，这种操作是不必要的，还会导致 <code>make gen.errcode.code</code> 执行很慢。</p><h3>技巧9：设置OPTIONS</h3><p>编写Makefile时，我们还需要把一些可变的功能通过OPTIONS来控制。为了帮助你理解，这里还是拿IAM项目的Makefile来举例。</p><p>假设我们需要通过一个选项 <code>V</code> ，来控制是否需要在执行Makefile时打印详细的信息。这可以通过下面的步骤来实现。</p><p><strong>首先，</strong>在/Makefile中定义 <code>USAGE_OPTIONS</code> 。定义 <code>USAGE_OPTIONS</code> 可以使开发者在执行 <code>make help</code> 后感知到此OPTION，并根据需要进行设置。</p><pre><code>define USAGE_OPTIONS    
                         
Options:
  ...
  BINS         The binaries to build. Default is all of cmd.
               ...
  ...
  V            Set to 1 enable verbose build. Default is 0.    
endef    
export USAGE_OPTIONS    
</code></pre><p><strong>接着，</strong>在<a href="https://github.com/marmotedu/iam/blob/master/scripts/make-rules/common.mk#L70">scripts/make-rules/common.mk</a>文件中，我们通过判断有没有设置V选项，来选择不同的行为：</p><pre><code>ifndef V    
MAKEFLAGS += --no-print-directory    
endif
</code></pre><p>当然，我们还可以通过下面的方法来使用 <code>V</code> ：</p><pre><code>ifeq ($(origin V), undefined)                                
MAKEFLAGS += --no-print-directory              
endif
</code></pre><p>上面，我介绍了 <code>V</code> OPTION，我们在Makefile中通过判断有没有定义 <code>V</code> ，来执行不同的操作。其实还有一种OPTION，这种OPTION的值我们在Makefile中是直接使用的，例如<code>BINS</code>。针对这种OPTION，我们可以通过以下方式来使用：</p><pre><code>BINS ?= $(foreach cmd,${COMMANDS},$(notdir ${cmd}))
...
go.build: go.build.verify $(addprefix go.build., $(addprefix $(PLATFORM)., $(BINS)))
</code></pre><p>也就是说，通过 <strong>?=</strong> 来判断 <code>BINS</code> 变量有没有被赋值，如果没有，则赋予等号后的值。接下来，就可以在Makefile规则中使用它。</p><h3>技巧10：定义环境变量</h3><p>我们可以在Makefile中定义一些环境变量，例如：</p><pre><code>GO := go                                          
GO_SUPPORTED_VERSIONS ?= 1.13|1.14|1.15|1.16|1.17    
GO_LDFLAGS += -X $(VERSION_PACKAGE).GitVersion=$(VERSION) \    
  -X $(VERSION_PACKAGE).GitCommit=$(GIT_COMMIT) \       
  -X $(VERSION_PACKAGE).GitTreeState=$(GIT_TREE_STATE) \                          
  -X $(VERSION_PACKAGE).BuildDate=$(shell date -u +'%Y-%m-%dT%H:%M:%SZ')    
ifneq ($(DLV),)                                                                                                                              
  GO_BUILD_FLAGS += -gcflags &quot;all=-N -l&quot;    
  LDFLAGS = &quot;&quot;      
endif                                                                                   
GO_BUILD_FLAGS += -tags=jsoniter -ldflags &quot;$(GO_LDFLAGS)&quot; 
...
FIND := find . ! -path './third_party/*' ! -path './vendor/*'    
XARGS := xargs --no-run-if-empty 
</code></pre><p>这些环境变量和编程中使用宏定义的作用是一样的：只要修改一处，就可以使很多地方同时生效，避免了重复的工作。</p><p>通常，我们可以将GO、GO_BUILD_FLAGS、FIND这类变量定义为环境变量。</p><h3>技巧11：自己调用自己</h3><p>在编写Makefile的过程中，你可能会遇到这样一种情况：A-Target目标命令中，需要完成操作B-Action，而操作B-Action我们已经通过伪目标B-Target实现过。为了达到最大的代码复用度，这时候最好的方式是在A-Target的命令中执行B-Target。方法如下：</p><pre><code>tools.verify.%:
  @if ! which $* &amp;&gt;/dev/null; then $(MAKE) tools.install.$*; fi
</code></pre><p>这里，我们通过 <code>$(MAKE)</code> 调用了伪目标 <code>tools.install.$*</code> 。要注意的是，默认情况下，Makefile在切换目录时会输出以下信息：</p><pre><code>$ make tools.install.codegen
===========&gt; Installing codegen
make[1]: Entering directory `/home/colin/workspace/golang/src/github.com/marmotedu/iam'
make[1]: Leaving directory `/home/colin/workspace/golang/src/github.com/marmotedu/iam'
</code></pre><p>如果觉得<strong>Entering directory</strong>这类信息很烦人，可以通过设置 <code>MAKEFLAGS += --no-print-directory</code> 来禁止Makefile打印这些信息。</p><h2>总结</h2><p>如果你想要高效管理项目，使用Makefile来管理是目前的最佳实践。我们可以通过下面的几个方法，来编写一个高质量的Makefile。</p><p>首先，你需要熟练掌握Makefile的语法。我建议你重点掌握以下语法：Makefile规则语法、伪目标、变量赋值、特殊变量、自动化变量。</p><p>接着，我们需要提前规划Makefile要实现的功能。一个大型Go项目通常需要实现以下功能：代码生成类命令、格式化类命令、静态代码检查、 测试类命令、构建类命令、Docker镜像打包类命令、部署类命令、清理类命令，等等。</p><p>然后，我们还需要通过Makefile功能分类、文件分层、复杂命令脚本化等方式，来设计一个合理的Makefile结构。</p><p>最后，我们还需要掌握一些Makefile编写技巧，例如：善用通配符、自动变量和函数；编写可扩展的Makefile；使用带层级的命名方式，等等。通过这些技巧，可以进一步保证我们编写出一个高质量的Makefile。</p><h2>课后练习</h2><ol>
<li>走读IAM项目的Makefile实现，看下IAM项目是如何通过 <code>make tools.install</code> 一键安装所有功能，通过 <code>make tools.install.xxx</code> 来指定安装 <code>xxx</code> 工具的。</li>
<li>你编写Makefile的时候，还用到过哪些编写技巧呢？欢迎和我分享你的经验，或者你踩过的坑。</li>
</ol><p>期待在留言区看到你的思考和答案，也欢迎和我一起探讨关于Makefile的问题，我们下一讲见！</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/dd/09/feca820a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>helloworld</span>
  </div>
  <div class="_2_QraFYR_0">格式化代码、静态代码检查，这种ide或vim都会配置保存时格式化和代码检查，还有必要写在makefile中吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: ide&#47;vim不会进行静态代码检查。<br><br>ide&#47;vim的格式化是可配的。而且ide&#47;vim的格式化更多的是用了gofmt -w这种格式化。<br><br>这里将格式化代码放在Makefile有个选择：<br>1. 更丰富，可配置的格式化代码功能，比如：支持golines、goimport、gofmt后面还可以根据需要增加更多<br>2. 确保每一个开发者用的都是同一种格式化方法（ide&#47;vim的格式化不一定每一个开发者都会陪，更不一定配置的格式化选项是一致的）<br><br>将格式化放在Makefile中，可以确保某个动作一定会被执行，并且执行的效果是一致的。<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-26 17:11:15</div>
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
  <div class="_2_QraFYR_0">这部分以前没接触过，先战略性放弃，后面再回来好好啃</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-29 11:57:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/63/84/f45c4af9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Vackine</span>
  </div>
  <div class="_2_QraFYR_0">写好一个功能齐全的项目的makefile，然后只对makefile 的各个功能做编排，是不是就可以做到基本的持续发布了？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 持续发布需要CI&#47;CD系统的支持。<br><br>写好Makefile只能说方便CI系统直接调用。离CI&#47;C差的还远</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-26 20:42:10</div>
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
  <div class="_2_QraFYR_0">总结：Makefile 是高效项目管理的重要工具，编写高质量的 Makefile，方便你做CI的检查，不至于自己的代码提交上以后，被提示各种不通过（静态代码检查、格式错误、单元测试等）。<br><br>要做好 Makefile 的管理，可按照下面的步骤：1. 学习 Makefile 语法；2. 规划 Makefile 要实现的功能；3. 规划 Makefile 结构；4. 善用 Makefile 技巧。<br><br>直接拷贝IAM项目的Makefile，也很香。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 直接copy可能更香</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-26 23:38:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_23bbd5</span>
  </div>
  <div class="_2_QraFYR_0">内容满满的...感觉都是作者的亲身体验</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-23 10:55:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/5d/a7/1a6b74df.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>青苔入镜</span>
  </div>
  <div class="_2_QraFYR_0">我有些困惑，iam项目将控制流和数据流放在了一个仓库中，然后使用makefile进行管理，方便我们学习部署，这个我倒是可以理解。<br>实际企业中是将所有组件也放在一个大仓库中的吗？我所在公司是将项目各个组件分为各个小仓库，开发后进行持续交付和持续部署。我感觉这样也比较合理一些啊，适合现在微服务这样，单个人就维护几个组件。而且控制面和数据面如果部署在一台机器上不会影响数据面的性能吗？<br>什么场景需要用makefile来管理项目呢？私有化部署的项目吗？我没有想的太明白</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 一个应用下面一般分多个微服务，我感觉放在一个Git仓库中好些。<br><br>放在一个Git仓库中，但编译出来的是几个二进制文件，所以部署还是独立的。<br>放在一个仓库中有以下几个优势：<br>1. 可以统一管理，比如静态代码检查，只需要维护一套就可以了，不用重复开发。<br>2. 便于阅读，不用去不同代码仓库中不断跳转。<br>3. 便于包共享，放在同一个仓库中，开发者能够更轻松的发现共享包，并且潜意识中，会去使用公用的包。<br><br>最大的优点是：编译统一维护管理，不用每个仓库都实现一套维护方法。<br><br>我觉得只要是项目都可以使用makefile管理。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-08 16:36:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/16/6b/af7c7745.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>tiny🌾</span>
  </div>
  <div class="_2_QraFYR_0">make lint执行完为什么会报这个错误<br>make[1]: *** [go.lint] Error 1<br>make: *** [lint] Error 2</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以加老师微信再帮你看下哈，这个报错看不出来什么错误</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-30 11:25:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/16/6b/af7c7745.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>tiny🌾</span>
  </div>
  <div class="_2_QraFYR_0">运行make lint报错 ，这个怎么排查<br>make[1]: *** [go.lint] Error 1<br>make: *** [lint] Error 2<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 加老师微信，现场帮你定位下哈</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-20 12:12:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/5b/66/ad35bc68.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>党</span>
  </div>
  <div class="_2_QraFYR_0">用win开发的表示 亚历山大</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-13 22:49:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/b5/e6/c67f12bd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>左耳朵东</span>
  </div>
  <div class="_2_QraFYR_0">.PHONY: format<br>format: tools.verify.golines<br>.PHONY: tools.verify.%<br>tools.verify.%:<br>	@if ! which $* &amp;&gt;&#47;dev&#47;null; then $(MAKE) tools.install.$*; fi<br>这里的 $* 是指 ‘golines’ 还是 ‘tools.verify.golines’ 呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: tools.verify.golines</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-23 23:09:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/75/bc/e24e181e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Calvin</span>
  </div>
  <div class="_2_QraFYR_0">对 Windows 系统不友好，实际 CRUD 业务中，更多人是在 Windows 下开发吧（IDE 选 VSCode 或 GoLand）。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 还有些在Mac上开发。<br><br>这个教程没必要在Mac、Windows平台下都适配一份。<br>之所以选择Linux是因为以下原因：<br><br>1. 因为Windows、Mac、Linux都有开发者，但该教程只能选择一个平台，所以选择了Linux<br>2. 将来部署服务是在Linux上部署的，在学习时，在Linux上完成开发部署，也是学习Linux的一个途径，为将来工作中， 在Linux下操作打下基础</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-18 00:43:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/e6/74/8b8097c1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>牛2特工</span>
  </div>
  <div class="_2_QraFYR_0">go为什么不设计专用的构建工具 <br>把常见通用的任务都集成了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 编译工具，不太适合集成太多这类功能。<br><br>而且这类功能具体实现也跟项目有关系，不好标准化处理</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-15 21:54:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/cf/63/23db516f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>CSB22</span>
  </div>
  <div class="_2_QraFYR_0">您有没有比较好的makefile editor推荐么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 暂时没有，直接用编辑器编辑其实也没啥问题</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-02 08:13:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2c/66/04/1919d4b4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>夏心C.J</span>
  </div>
  <div class="_2_QraFYR_0">$*是指示目标模式中 % 及其之前的部分，不是指代被匹配的值swagger、mockgen吧？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不会的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-11 14:25:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/8e/62/5cb377fd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>yangchnet</span>
  </div>
  <div class="_2_QraFYR_0">第二遍了，越读越觉得总结的到位精辟。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-09 11:32:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/5b/66/ad35bc68.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>党</span>
  </div>
  <div class="_2_QraFYR_0">win上没有make命令 老师你的开发环境是linux或者mac么 win上怎么搞make</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: win上没试过，你可以调研下怎么搞。这个课程全都是在Linux上搞的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-26 23:27:06</div>
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
  <div class="_2_QraFYR_0">请问老师如何看待Makefile已过时的言论？<br>在知乎上查找相关资料时，发现很多人认为Makefile过时了，只需要学习cmake、bazel之类的，各种说法都有，现在有点晕</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我感觉Makefile没过时，kubernetes等大型项目都是用的Makefile。<br><br>Makefile功能强大，应对超大型的项目都没问题，对于一般的项目更不是瓶颈了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-26 19:17:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/ad/e6/bf43ee14.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ggchangan</span>
  </div>
  <div class="_2_QraFYR_0">老师的课程真是超值，特别感谢！有个问题请教下，怎么自动生成基于suite的测试代码呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 调研了下，当前应该没有这类工具。如果你了解有，欢迎评论区分享。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-14 13:37:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/60/4f/db0e62b3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Daiver</span>
  </div>
  <div class="_2_QraFYR_0">老师，看到iam项目下面的license 文件中添加了其他库的说明，这个是手动加上去的吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 也是自动生成的，iam中很多都是通过工具来搞的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-23 23:17:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/7a/d2/4ba67c0c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Sch0ng</span>
  </div>
  <div class="_2_QraFYR_0">使用Makefile可以高效管理go项目。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-10 00:21:44</div>
  </div>
</div>
</div>
</li>
</ul>