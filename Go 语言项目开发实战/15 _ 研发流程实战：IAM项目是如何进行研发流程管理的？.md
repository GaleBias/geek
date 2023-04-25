<audio title="15 _ 研发流程实战：IAM项目是如何进行研发流程管理的？" src="https://static001.geekbang.org/resource/audio/55/9e/557e9380b88bfc481876d576254yyd9e.mp3" controls="controls"></audio> 
<p>你好，我是孔令飞。</p><p>在 <a href="https://time.geekbang.org/column/article/383390"><strong>08讲</strong></a>  和 <a href="https://time.geekbang.org/column/article/388920"><strong>14讲</strong></a>  ，我分别介绍了如何设计研发流程，和如何基于 Makefile 高效地管理项目。那么今天，我们就以研发流程为主线，来看下IAM项目是如何通过Makefile来高效管理项目的。学完这一讲，你不仅能更加深刻地理解 <strong>08讲</strong> 和 <strong>14讲</strong> 所介绍的内容，还能得到很多可以直接用在实际操作中的经验、技巧。</p><p>研发流程有很多阶段，其中的开发阶段和测试阶段是需要开发者深度参与的。所以在这一讲中，我会重点介绍这两个阶段中的Makefile项目管理功能，并且穿插一些我的Makefile的设计思路。</p><p>为了向你演示流程，这里先假设一个场景。我们有一个需求：给IAM客户端工具iamctl增加一个helloworld命令，该命令向终端打印hello world。</p><p>接下来，我们就来看下如何具体去执行研发流程中的每一步。首先，我们进入开发阶段。</p><h2>开发阶段</h2><p>开发阶段是开发者的主战场，完全由开发者来主导，它又可分为代码开发和代码提交两个子阶段。我们先来看下代码开发阶段。</p><h3>代码开发</h3><p>拿到需求之后，首先需要开发代码。这时，我们就需要选择一个适合团队和项目的Git工作流。因为Git  Flow工作流比较适合大型的非开源项目，所以这里我们选择<strong>Git</strong>  <strong>Flow工作流</strong>。代码开发的具体步骤如下：</p><!-- [[[read_end]]] --><p>第一步，基于develop分支，新建一个功能分支  feature/helloworld。</p><pre><code>$ git checkout -b feature/helloworld develop
</code></pre><p><strong>这里需要注意</strong>：新建的branch名要符合Git  Flow工作流中的分支命名规则。否则，在git commit阶段，会因为branch不规范导致commit失败。IAM项目的分支命令规则具体如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/15/79/15bb43219269273baf70a27ea94e1279.png?wh=2775x1250" alt=""></p><p>IAM项目通过pre-commit githooks来确保分支名是符合规范的。在IAM项目根目录下执行git commit 命令，git会自动执行<a href="https://github.com/marmotedu/iam/blob/master/githooks/pre-commit">pre-commit</a>脚本，该脚本会检查当前branch的名字是否符合规范。</p><p>这里还有一个地方需要你注意：git不会提交 <code>.git/hooks</code> 目录下的githooks脚本，所以我们需要通过以下手段，确保开发者clone仓库之后，仍然能安装我们指定的githooks脚本到 <code>.git/hooks</code> 目录：</p><pre><code># Copy githook scripts when execute makefile    
COPY_GITHOOK:=$(shell cp -f githooks/* .git/hooks/) 
</code></pre><p>上述代码放在<a href="https://github.com/marmotedu/iam/blob/master/scripts/make-rules/common.mk#L74">scripts/make-rules/common.mk</a>文件中，每次执行make命令时都会执行，可以确保githooks都安装到 <code>.git/hooks</code> 目录下。</p><p>第二步，在feature/helloworld分支中，完成helloworld命令的添加。</p><p>首先，通过 <code>iamctl new helloworld</code> 命令创建helloworld命令模板：</p><pre><code>$ iamctl new helloworld -d internal/iamctl/cmd/helloworld
Command file generated: internal/iamctl/cmd/helloworld/helloworld.go
</code></pre><p>接着，编辑<code>internal/iamctl/cmd/cmd.go</code>文件，在源码文件中添加<code>helloworld.NewCmdHelloworld(f, ioStreams),</code>，加载helloworld命令。这里将helloworld命令设置为<code>Troubleshooting and Debugging Commands</code>命令分组：</p><pre><code>import (
    &quot;github.com/marmotedu/iam/internal/iamctl/cmd/helloworld&quot;
)
        ...
        {
            Message: &quot;Troubleshooting and Debugging Commands:&quot;,
            Commands: []*cobra.Command{
                validate.NewCmdValidate(f, ioStreams),
                helloworld.NewCmdHelloworld(f, ioStreams),
            },
        },
</code></pre><p>这些操作中包含了low code的思想。在第 <a href="https://time.geekbang.org/column/article/384648"><strong>10讲</strong></a> 中我就强调过，要尽可能使用代码自动生成这一技术。这样做有两个好处：一方面能够提高我们的代码开发效率；另一方面也能够保证规范，减少手动操作可能带来的错误。所以这里，我将iamctl的命令也模板化，并通过 <code>iamctl new</code> 自动生成。</p><p>第三步，生成代码。</p><pre><code>$ make gen
</code></pre><p>如果改动不涉及代码生成，可以不执行<code>make gen</code>操作。 <code>make gen</code> 执行的其实是gen.run伪目标：</p><pre><code>gen.run: gen.clean gen.errcode gen.docgo.doc
</code></pre><p>可以看到，当执行 <code>make gen.run</code> 时，其实会先清理之前生成的文件，再分别自动生成error code和doc.go文件。</p><p>这里需要注意，通过<code>make gen</code> 生成的存量代码要具有幂等性。只有这样，才能确保每次生成的代码是一样的，避免不一致带来的问题。</p><p>我们可以将更多的与自动生成代码相关的功能放在 gen.mk Makefile 中。例如：</p><ul>
<li>gen.docgo.doc，代表自动生成doc.go文件。</li>
<li>gen.ca.%，代表自动生成iamctl、iam-apiserver、iam-authz-server证书文件。</li>
</ul><p>第四步，版权检查。</p><p>如果有新文件添加，我们还需要执行 <code>make verify-copyright</code>  ，来检查新文件有没有添加版权头信息。</p><pre><code>$ make verify-copyright
</code></pre><p>如果版权检查失败，可以执行<code>make add-copyright</code>自动添加版权头。添加版权信息只针对开源软件，如果你的软件不需要添加，就可以略过这一步。</p><p>这里还有个Makefile编写技巧：如果Makefile的command需要某个命令，就可以使该目标依赖类似tools.verify.addlicense这种目标，tools.verify.addlicense会检查该工具是否已安装，如果没有就先安装。</p><pre><code>.PHONY: copyright.verify    
copyright.verify: tools.verify.addlicense 
  ...
tools.verify.%:          
  @if ! which $* &amp;&gt;/dev/null; then $(MAKE) tools.install.$*; fi
.PHONY: install.addlicense                              
install.addlicense:        
  @$(GO) get -u github.com/marmotedu/addlicense
</code></pre><p>通过这种方式，可以使 <code>make copyright.verify</code> 尽可能自动化，减少手动介入的概率。</p><p>第五步，代码格式化。</p><pre><code>$ make format
</code></pre><p>执行<code>make format</code>会依次执行以下格式化操作：</p><ol>
<li>调用gofmt格式化你的代码。</li>
<li>调用goimports工具，自动增删依赖的包，并将依赖包按字母序排序并分类。</li>
<li>调用golines工具，把超过120行的代码按golines规则，格式化成&lt;120行的代码。</li>
<li>调用 <code>go mod edit -fmt</code> 格式化go.mod文件。</li>
</ol><p>第六步，静态代码检查。</p><pre><code>$ make lint
</code></pre><p>关于静态代码检查，在这里你可以先了解代码开发阶段有这个步骤，至于如何操作，我会在下一讲给你详细介绍。</p><p>第七步，单元测试。</p><pre><code>$ make test
</code></pre><p>这里要注意，并不是所有的包都需要执行单元测试。你可以通过如下命令，排除掉不需要单元测试的包：</p><pre><code>go test `go list ./...|egrep -v $(subst $(SPACE),'|',$(sort $(EXCLUDE_TESTS)))`
</code></pre><p>在go.test的command中，我们还运行了以下命令：</p><pre><code>sed -i '/mock_.*.go/d' $(OUTPUT_DIR)/coverage.out
</code></pre><p>运行该命令的目的，是把mock_.* .go文件中的函数单元测试信息从coverage.out中删除。mock_.*.go文件中的函数是不需要单元测试的，如果不删除，就会影响后面的单元测试覆盖率的计算。</p><p>如果想检查单元测试覆盖率，请执行：</p><pre><code>$ make cover
</code></pre><p>默认测试覆盖率至少为60%，也可以在命令行指定覆盖率阈值为其他值，例如：</p><pre><code>$ make cover COVERAGE=90
</code></pre><p>如果测试覆盖率不满足要求，就会返回以下错误信息：</p><pre><code>test coverage is 62.1%
test coverage does not meet expectations: 90%, please add test cases!
make[1]: *** [go.test.cover] Error 1
make: *** [cover] Error 2
</code></pre><p>这里make命令的退出码为<code>1</code>。</p><p>如果单元测试覆盖率达不到设置的阈值，就需要补充测试用例，否则禁止合并到develop和master分支。IAM项目配置了GitHub Actions CI自动化流水线，CI流水线会自动运行，检查单元测试覆盖率是否达到要求。</p><p>第八步，构建。</p><p>最后，我们执行<code>make build</code>命令，构建出<code>cmd/</code>目录下所有的二进制安装文件。</p><pre><code>$ make build
</code></pre><p><code>make build</code> 会自动构建 <code>cmd/</code> 目录下的所有组件，如果只想构建其中的一个或多个组件，可以传入 <code>BINS</code>选项，组件之间用空格隔开，并用双引号引起来：</p><pre><code>$ make build BINS=&quot;iam-apiserver iamctl&quot;
</code></pre><p>到这里，我们就完成了代码开发阶段的全部操作。</p><p>如果你觉得手动执行的make命令比较多，可以直接执行make命令：</p><pre><code>$ make
===========&gt; Generating iam error code go source files
===========&gt; Generating error code markdown documentation
===========&gt; Generating missing doc.go for go packages
===========&gt; Verifying the boilerplate headers for all files
===========&gt; Formating codes
===========&gt; Run golangci to lint source codes
===========&gt; Run unit test
...
===========&gt; Building binary iam-pump v0.7.2-24-g5814e7b for linux amd64
===========&gt; Building binary iamctl v0.7.2-24-g5814e7b for linux amd64
...
</code></pre><p>直接执行<code>make</code>会执行伪目标<code>all</code>所依赖的伪目标 <code>all: tidy gen add-copyright format lint cover build</code>，也即执行以下操作：依赖包添加/删除、生成代码、自动添加版权头、代码格式化、静态代码检查、覆盖率测试、构建。</p><p>这里你需要注意一点：all中依赖cover，cover实际执行的是 <code>go.test.cover</code> ，而 <code>go.test.cover</code> 又依赖 <code>go.test</code> ，所以cover实际上是先执行单元测试，再检查单元测试覆盖率是否满足预设的阈值。</p><p>最后补充一点，在开发阶段我们可以根据需要随时执行 <code>make gen</code> 、 <code>make format</code> 、 <code>make lint</code> 、 <code>make cover</code> 等操作，为的是能够提前发现问题并改正。</p><h3>代码提交</h3><p>代码开发完成之后，我们就需要将代码提交到远程仓库，整个流程分为以下几个步骤。</p><p>第一步，开发完后，将代码提交到feature/helloworld分支，并push到远端仓库。</p><pre><code>$ git add internal/iamctl/cmd/helloworld internal/iamctl/cmd/cmd.go
$ git commit -m &quot;feat: add new iamctl command 'helloworld'&quot;
$ git push origin feature/helloworld
</code></pre><p>这里我建议你只添加跟<code>feature/helloworld</code>相关的改动，这样就知道一个commit做了哪些变更，方便以后追溯。所以，我不建议直接执行<code>git add .</code>这类方式提交改动。</p><p>在提交commit时，commit-msg githooks会检查commit message是否符合Angular Commit Message规范，如果不符合会报错。commit-msage调用了<a href="https://github.com/llorllale/go-gitlint">go-gitlint</a>来检查commit message。go-gitlint会读取 <code>.gitlint</code> 中配置的commit message格式：</p><pre><code>--subject-regex=^((Merge branch.*of.*)|((revert: )?(feat|fix|perf|style|refactor|test|ci|docs|chore)(\(.+\))?: [^A-Z].*[^.]$))
--subject-maxlen=72
--body-regex=^([^\r\n]{0,72}(\r?\n|$))*$
</code></pre><p>IAM项目配置了GitHub Actions，当有代码被push后，会触发CI流水线，流水线会执行<code>make all</code>目标。GitHub Actions CI流程执行记录如下图：</p><p><img src="https://static001.geekbang.org/resource/image/68/22/6819f96bda8dcb214c3b7eeba2f37022.png?wh=2061x435" alt=""></p><p>如果CI不通过，就需要修改代码，直到CI流水线通过为止。</p><p>这里，我们来看下GitHub Actions的配置：</p><pre><code>name: IamCI

on: 
  push:
    branchs:
    - '*'
  pull_request:
    types: [opened, reopened]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2

    - name: Set up Go
      uses: actions/setup-go@v2
      with:
        go-version: 1.16

    - name: all
      run: make
</code></pre><p>可以看到，GitHub Actions实际上执行了3步：拉取代码、设置Go编译环境、执行make命令（也就是执行 <code>make all</code> 目标）。</p><p>GitHub Actions也执行了 <code>make all</code> 目标，和手动操作执行的 <code>make all</code> 目标保持一致，这样做是为了让线上的CI流程和本地的CI流程完全保持一致。这样，当我们在本地执行make命令通过后，在线上也会通过。保持一个一致的执行流程和执行结果很重要。否则，本地执行make通过，但是线上却不通过，岂不很让人头疼？</p><p>第二步，提交pull request。</p><p>登陆GitHub，基于feature/helloworld创建pull request，并指定Reviewers进行code review。具体操作如下图：</p><p><img src="https://static001.geekbang.org/resource/image/53/ab/53f4103f5c8cabb76ef2fddaec3a54ab.png?wh=1694x733" alt=""></p><p>当有新的pull request被创建后，也会触发CI流水线。</p><p>第三步，创建完pull request后，就可以通知reviewers 来 review代码，GitHub也会发站内信。</p><p>第四步，Reviewers 对代码进行review。</p><p>Reviewer通过review github diff后的内容，并结合CI流程是否通过添加评论，并选择Comment（仅评论）、Approve（通过）、Request Changes（不通过，需要修改），如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/39/ce/39d992c7bdb35848706bce792877e8ce.png?wh=2473x1001" alt=""></p><p>如果review不通过，feature开发者可以直接在feature/helloworld分支修正代码，并push到远端的feature/helloworld分支，然后通知reviewers再次review。因为有push事件发生，所以会触发GitHub Actions CI流水线。</p><p>第五步，code review通过后，maintainer就可以将新的代码合并到develop分支。</p><p>使用<strong>Create a merge commit</strong>的方式，将pull request合并到develop分支，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/30/7d/30de6bb6c8ff431ec56debbc0f5b667d.png?wh=1247x366" alt=""></p><p><strong>Create a merge commit</strong>的实际操作是 <code>git merge --no-ff</code>，feature/helloworld分支上所有的 commit 都会加到 develop 分支上，并且会生成一个 merge commit。使用这种方式，可以清晰地知道是谁做了哪些提交，回溯历史的时候也会更加方便。</p><p>第六步，合并到develop分支后，触发CI流程。</p><p>到这里，开发阶段的操作就全部完成了，整体流程如下：</p><p><img src="https://static001.geekbang.org/resource/image/44/73/444b0701f8866b50a49bd0138488c873.png?wh=1697x1028" alt=""></p><p>合并到develop分支之后，我们就可以进入开发阶段的下一阶段，也就是测试阶段了。</p><h2>测试阶段</h2><p>在测试阶段，开发人员主要负责提供测试包和修复测试期间发现的bug，这个过程中也可能会发现一些新的需求或变动点，所以需要合理评估这些新的需求或变动点是否要放在当前迭代修改。</p><p>测试阶段的操作流程如下。</p><p>第一步，基于develop分支，创建release分支，测试代码。</p><pre><code>$ git checkout -b release/1.0.0 develop
$ make
</code></pre><p>第二步，提交测试。</p><p>将release/1.0.0分支的代码提交给测试同学进行测试。这里假设一个测试失败的场景：我们要求打印“hello world”，但打印的是“Hello World”，需要修复。那具体应该怎么操作呢？</p><p>你可以直接在release/1.0.0分支修改代码，修改完成后，本地构建并提交代码：</p><pre><code>$ make
$ git add internal/iamctl/cmd/helloworld/
$ git commit -m &quot;fix: fix helloworld print bug&quot;
$ git push origin release/1.0.0
</code></pre><p>push到release/1.0.0后，GitHub Actions会执行CI流水线。如果流水线执行成功，就将代码提供给测试；如果测试不成功，再重新修改，直到流水线执行成功。</p><p>测试同学会对release/1.0.0分支的代码进行充分的测试，例如功能测试、性能测试、集成测试、系统测试等。</p><p>第三步，测试通过后，将功能分支合并到master分支和develop分支。</p><pre><code>$ git checkout develop
$ git merge --no-ff release/1.0.0
$ git checkout master
$ git merge --no-ff release/1.0.0
$ git tag -a v1.0.0 -m &quot;add print hello world&quot; # master分支打tag
</code></pre><p>到这里，测试阶段的操作就基本完成了。测试阶段的产物是master/develop分支的代码。</p><p>第四步，删除feature/helloworld分支，也可以选择性删除release/1.0.0分支。</p><p>我们的代码都合并入master/develop分支后，feature开发者可以选择是否要保留feature。不过，如果没有特别的原因，我建议删掉，因为feature分支太多的话，不仅看起来很乱，还会影响性能，删除操作如下：</p><pre><code>$ git branch -d feature/helloworld
</code></pre><h2>IAM项目的Makefile项目管理技巧</h2><p>在上面的内容中，我们以研发流程为主线，亲身体验了IAM项目的Makefile项目管理功能。这些是你最应该掌握的核心功能，但IAM项目的Makefile还有很多功能和设计技巧。接下来，我会给你分享一些很有价值的Makefile项目管理技巧。</p><h3>help自动解析</h3><p>因为随着项目的扩展，Makefile大概率会不断加入新的管理功能，这些管理功能也需要加入到 <code>make help</code> 输出中。但如果每添加一个目标，都要修改 <code>make help</code> 命令，就比较麻烦，还容易出错。所以这里，我通过自动解析的方式，来生成<code>make help</code>输出：</p><pre><code>## help: Show this help info.    
.PHONY: help           
help: Makefile               
  @echo -e &quot;\nUsage: make &lt;TARGETS&gt; &lt;OPTIONS&gt; ...\n\nTargets:&quot;                         
  @sed -n 's/^##//p' $&lt; | column -t -s ':' | sed -e 's/^/ /'    
  @echo &quot;$$USAGE_OPTIONS&quot;    
</code></pre><p>目标help的命令中，通过 <code>sed -n 's/^##//p' $&lt; | column -t -s ':' | sed -e 's/^/ /'</code> 命令，自动解析Makefile中 <code>##</code> 开头的注释行，从而自动生成 <code>make help</code> 输出。</p><h3>Options中指定变量值</h3><p>通过以下赋值方式，变量可以在Makefile options中被指定：</p><pre><code>ifeq ($(origin COVERAGE),undefined)    
COVERAGE := 60    
endif   
</code></pre><p>例如，如果我们执行<code>make</code>  ，则COVERAGE设置为默认值60；如果我们执行<code>make COVERAGE=90</code>  ，则COVERAGE值为90。通过这种方式，我们可以更灵活地控制Makefile的行为。</p><h3>自动生成CHANGELOG</h3><p>一个项目最好有CHANGELOG用来展示每个版本之间的变更内容，作为Release Note的一部分。但是，如果每次都要手动编写CHANGELOG，会很麻烦，也不容易坚持，所以这里我们可以借助<a href="https://github.com/git-chglog/git-chglog">git-chglog</a>工具来自动生成。</p><p>IAM项目的git-chglog工具的配置文件放在<a href="https://github.com/marmotedu/iam/tree/master/.chglog">.chglog</a>目录下，在学习git-chglog工具时，你可以参考下。</p><h3>自动生成版本号</h3><p>一个项目也需要有一个版本号，当前用得比较多的是语义化版本号规范。但如果靠开发者手动打版本号，工作效率低不说，经常还会出现漏打、打的版本号不规范等问题。所以最好的办法是，版本号也通过工具自动生成。在IAM项目中，采用了<a href="https://github.com/arnaud-deprez/gsemver">gsemver</a>工具来自动生成版本号。</p><p>整个IAM项目的版本号，都是通过<a href="https://github.com/marmotedu/iam/blob/master/scripts/ensure_tag.sh">scripts/ensure_tag.sh</a>脚本来生成的：</p><pre><code>version=v`gsemver bump`
if [ -z &quot;`git tag -l $version`&quot; ];then
  git tag -a -m &quot;release version $version&quot; $version
fi
</code></pre><p>在scripts/ensure_tag.sh脚本中，通过 <code>gsemver bump</code> 命令来自动化生成语义化的版本号，并执行 <code>git tag -a</code> 给仓库打上版本号标签，<code>gsemver</code> 命令会根据Commit Message自动生成版本号。</p><p>之后，Makefile和Shell脚本用到的所有版本号均统一使用<a href="https://github.com/marmotedu/iam/blob/v1.0.0/scripts/make-rules/common.mk#L28">scripts/make-rules/common.mk</a>文件中的VERSION变量：</p><pre><code>VERSION := $(shell git describe --tags --always --match='v*')
</code></pre><p>上述的Shell命令通过 <code>git describe</code> 来获取离当前提交最近的tag（版本号）。</p><p>在执行 <code>git describe</code> 时，如果符合条件的tag指向最新提交，则只显示tag的名字，否则会有相关的后缀，来描述该tag之后有多少次提交，以及最新的提交commit id。例如：</p><pre><code>$ git describe --tags --always --match='v*'
v1.0.0-3-g1909e47
</code></pre><p>这里解释下版本号中各字符的含义：</p><ul>
<li>3：表示自打tag v1.0.0以来有3次提交。</li>
<li>g1909e47：g 为git的缩写，在多种管理工具并存的环境中很有用处。</li>
<li>1909e47：7位字符表示为最新提交的commit id 前7位。</li>
</ul><p>最后解释下参数：</p><ul>
<li>–tags，使用所有的标签，而不是只使用带注释的标签（annotated tag）。<code>git tag &lt;tagname&gt; </code>生成一个 unannotated tag，<code>git tag -a &lt;tagname&gt; -m '&lt;message&gt;' </code>生成一个 annotated tag。</li>
<li>–always，如果仓库没有可用的标签，那么使用commit缩写来替代标签。</li>
<li>–match <pattern>，只考虑与给定模式相匹配的标签。</pattern></li>
</ul><h3>保持行为一致</h3><p>上面我们介绍了一些管理功能，例如检查Commit Message是否符合规范、自动生成CHANGELOG、自动生成版本号。这些可以通过Makefile来操作，我们也可以手动执行。例如，通过以下命令，检查IAM的所有Commit是否符合Angular Commit Message规范：</p><pre><code>$ go-gitlint
b62db1f: subject does not match regex [^(revert: )?(feat|fix|perf|style|refactor|test|ci|docs|chore)(\(.+\))?: [^A-Z].*[^.]$]
</code></pre><p>也可以通过以下命令，手动来生成CHANGELOG：</p><pre><code>$ git-chglog v1.0.0 CHANGELOG/CHANGELOG-1.0.0.md
</code></pre><p>还可以执行gsemver来生成版本号：</p><pre><code>$ gsemver bump
1.0.1
</code></pre><p>这里要强调的是，我们要保证<strong>不管使用手动操作，还是通过Makefile操作</strong>，都要确保git commit message规范检查结果、生成的CHANGELOG、生成的版本号是一致的。这需要我们<strong>采用同一种操作方式</strong>。</p><h2>总结</h2><p>在整个研发流程中，需要开发人员深度参与的阶段有两个，分别是开发阶段和测试阶段。在开发阶段，开发者完成代码开发之后，通常需要执行生成代码、版权检查、代码格式化、静态代码检查、单元测试、构建等操作。我们可以将这些操作集成在Makefile中，来提高效率，并借此统一操作。</p><p>另外，IAM项目在编写Makefile时也采用了一些技巧，例如<code>make help</code> 命令中，help信息是通过解析Makefile文件的注释来完成的；可以通过git-chglog自动生成CHANGELOG；通过gsemver自动生成语义化的版本号等。</p><h2>课后练习</h2><ol>
<li>看下IAM项目的 <code>make dependencies</code> 是如何实现的，这样实现有什么好处？</li>
<li>IAM项目中使用 了<code>gofmt</code> 、<code>goimports</code> 、<code>golines</code> 3种格式化工具，思考下，还有没有其他格式化工具值得集成在 <code>make format</code> 目标的命令中？</li>
</ol><p>欢迎你在留言区分享你的见解，和我一起交流讨论，我们下一讲见！</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/7a/d2/4ba67c0c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Sch0ng</span>
  </div>
  <div class="_2_QraFYR_0">这一套流程下来确实能节省很多时间，重复性的操作交给自动化比人手工写更加可靠。<br>高效，省时省力还不易出错。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-10 00:29:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/a5/cd/3aff5d57.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Alery</span>
  </div>
  <div class="_2_QraFYR_0">“如果 review 不通过，feature 开发者可以直接在 feature&#47;helloworld 分支修正代码，并 push 到远端的 feature&#47;helloworld 分支，然后通知 reviewers 再次 review。因为有 push 事件发生，所以会触发 GitHub Actions CI 流水线。”<br><br>请问修复后的代码是直接执行gith commit --amend还是重新创建一个commit</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 都可以的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-13 20:50:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/64/53/c93b8110.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>daz2yy</span>
  </div>
  <div class="_2_QraFYR_0">老师好，想问下，测试阶段过了之后，这个特性就能直接上线吗？还是说等大家一起开发完这个迭代内容再上线？<br>另外，后端开发这里经常会设计 SQL 的变动，一种是数据变动，一种是结构变动，老师这块怎么去管理的呢？还有如何集成到特性研发流程里的呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 测试完之后就会上线<br><br>数据库变动这种运维开发，发布的时候变动，可以写在发布计划里</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-29 08:09:59</div>
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
  <div class="_2_QraFYR_0">release分支是从develop分支来的，如果测试直接通过，没有做进一步修改，就不用再合并到develop分支了吧，直接合并到master分支就可以了吧，这样理解对不对呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果代码都一样没必要再合，不过再合下没什么大问题，小心总不为过。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-29 00:46:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/ea/11/4f464693.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>徐蕴</span>
  </div>
  <div class="_2_QraFYR_0">测试完成的tag应该打在master上，还是release分支对应的commit上呢？master上应该还有其它没有测试的功能吧？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 好问题。打在release分支上。<br><br>master分支上的代码都是充分测试过的代码。不然不让合master的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-05 16:37:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/xfclWEPQ7szTZnKqnX9icSbgDWV0VAib3Cyo8Vg0OG3Usby88ic7ZgO2ho5lj0icOWI4JeJ70zUBiaTW1xh1UCFRPqA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_6bdb4e</span>
  </div>
  <div class="_2_QraFYR_0">我来回答一下第一个问题：make dependencies会执行dependencies.run，进而执行1. dependencies.packages，这个是对代码本身依赖的packages进行管理，2. dependencies.tools，这个是对项目本身依赖的一些工具进行检查和管理，其中又根据重要等级区分为blocker和critical以及trivial，缺失blocker可能导致CI流程失败，缺失critical可能导致make的一些环节失败，tiivial是可选的工具，缺失没有任何影响。我有一个小问题，这个make dependencies的调用时机是什么时候，发现只有通过make dependencies时候才会调用，为什么不把这条放到make all的依赖项里面呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 调用时机，项目刚clone下来的时候：<br>git clone https:&#47;&#47;github.com&#47;marmotedu&#47;iam<br>make dependencies<br><br>不放在all的原因时，make dependencies 只需要执行一次就可以了。<br>放在all中，每次都要执行dependencies，会导致make命令很慢<br><br><br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-02 09:50:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/21/5f/f8/1d16434b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>陈麒文</span>
  </div>
  <div class="_2_QraFYR_0">这么多的知识点，从哪开始入手比较能跟上大佬的节奏？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 跟着老师的思路学习就可以了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-17 14:43:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_4c902b</span>
  </div>
  <div class="_2_QraFYR_0">老师，您好：<br>iamctl new helloworld  这个iamctl 命令哪来的呀</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: make build BINS=iamctl会构建iamctl工具。<br><br>iamctl在IAM项目安装那一讲会介绍如何安装。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-29 23:12:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/23/1b/75/0d18b0bc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Lee</span>
  </div>
  <div class="_2_QraFYR_0">这里有个疑惑的店：如果基于develop分支checkout的话,假如开发已经完成并且通过测试了功能A，但是develop环境此时还有通过测试但是未打算上线的其他功能B，如果此时把dev分支合并到release分支的话，是不是功能B也一并被上线了？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-17 14:10:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/4d/26/44095eba.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>SuperSu</span>
  </div>
  <div class="_2_QraFYR_0">慢慢的干活</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-23 21:21:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/67/17/cbd5c23f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>H.xu</span>
  </div>
  <div class="_2_QraFYR_0">[going@192 iam]$ make test<br>===========&gt; Run unit test<br>found packages example (doc.go) and main (example.go) in &#47;home&#47;going&#47;workspace&#47;golang&#47;src&#47;github.com&#47;marmotedu&#47;iam&#47;pkg&#47;rollinglog&#47;example<br>found packages example (doc.go) and main (example.go) in &#47;home&#47;going&#47;workspace&#47;golang&#47;src&#47;github.com&#47;marmotedu&#47;iam&#47;pkg&#47;validator&#47;example<br>no Go files in &#47;home&#47;going&#47;workspace&#47;golang&#47;src&#47;github.com&#47;marmotedu&#47;iam<br>make[1]: *** [scripts&#47;make-rules&#47;golang.mk:81: go.test] Error 1<br>make: *** [Makefile:108: test] Error 2<br>这个错误是什么原因的呀</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是不是没有执行go work use iam呀</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-02 18:11:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/28/b4/35/3a96e893.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>🌿小毒草</span>
  </div>
  <div class="_2_QraFYR_0">如何保证“通过make gen 生成的存量代码要具有幂等性”， 如何测试？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个要具体情况具体分析。具体，可以添加代码生成功能项的时候，评估测试下</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-27 10:30:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小达</span>
  </div>
  <div class="_2_QraFYR_0">iamctl new helloworld -d internal&#47;iamctl&#47;cmd&#47;helloworld这个在哪个目录下执行呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 在iam项目根目录下执行</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-25 21:43:56</div>
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
  <div class="_2_QraFYR_0">Mark一个知识点：Maintainer 处理 pull request 冲突合并。https:&#47;&#47;blog.csdn.net&#47;danchenziDCZ&#47;article&#47;details&#47;81061989</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢分享</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-04 22:54:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/f0/eb/24a8be29.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>RunDouble</span>
  </div>
  <div class="_2_QraFYR_0">好像没有用 git rebase origin&#47;develop</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-24 16:58:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/a2/94/ae0a60d8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>江山未</span>
  </div>
  <div class="_2_QraFYR_0">根据符合规范的commit message 生成版本号，这里有个疑问<br>是一次commit会滚动一次数字吗？因为一个版本可能包含多个feature，feature a有一次commit，feature b有一次commit，但它们都属于这次的版本更新。那会造成 minor version +2 吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 版本号可按以下规则递增：<br>1. 主版本号（MAJOR）：当做了不兼容的 API 修改。例如：带有 BREAKING CHANGE 的 commit。<br>2. 次版本号（MINOR）：当做了向下兼容的功能性新增及修改。这里有个不成文的约定需要你注意，偶数为稳定版本，奇数为开发版本。（feat: xxxx）<br>3. 修订号（PATCH）：当做了向下兼容的问题修正。(fix:  xxx)</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-22 09:57:53</div>
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
  <div class="_2_QraFYR_0">总结：这些步骤应该是开发者经常遇到的。<br>开发阶段：执行 make all，本地走一遍CI流程。包括，自动生成部分代码或文档、静态代码检查、代码格式化、构建等。<br>提交阶段：检查 commit message 是否符合 angular 规范；在 push commit &#47; pull request &#47; merge to develop 点，触发 CI 流程。所以，开发者要有在本地跑CI的方法。<br>测试阶段：按照 Git Flow 的原则，创建出 release 分支，对代码进行测试。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 66666</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-27 00:16:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c6/73/abb7bfe3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>疯琴</span>
  </div>
  <div class="_2_QraFYR_0">私房干活👍</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-21 18:43:20</div>
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
  <div class="_2_QraFYR_0">孔老师，--tags  和 --always 还是没能理解。能再详细说说看吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: tag有2种，有带注释的(git tag -a &lt;tagname&gt; -m &#39;&lt;message&gt;&#39;, 生成一个 annotated tag)，有不带注释的(git tag &lt;tagname&gt;, 生成一个 unannotated tag). git describe默认只列出annotated tag，--tags指定git describe也列出unannotated tag，也就是所有tag。<br><br>--always: 意思是，如果当前git仓库没有tag，那么使用commit来替代，例如：<br>$ git describe<br>fatal: No names found, cannot describe anything.<br>[colin@dev 222]$ git describe --always<br>18fedf3</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-02 21:28:11</div>
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
  <div class="_2_QraFYR_0">孔老板，这里文中创建的release分支的版本1.0.0：git checkout -b release&#47;1.0.0 develop。<br><br>是最佳实践吗？ 还是根据需要任意创建一个版本即可。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 根据需要创建一个版本即可</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-02 17:56:38</div>
  </div>
</div>
</div>
</li>
</ul>