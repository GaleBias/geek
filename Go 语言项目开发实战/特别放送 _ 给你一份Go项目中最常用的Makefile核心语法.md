<audio title="特别放送 _ 给你一份Go项目中最常用的Makefile核心语法" src="https://static001.geekbang.org/resource/audio/c3/8d/c3e5d99d3a5575f52148ed0104ecd48d.mp3" controls="controls"></audio> 
<p>你好，我是孔令飞。今天，我们更新一期特别放送作为“加餐”，希望日常催更的朋友们食用愉快。</p><p>在第 <a href="https://time.geekbang.org/column/article/388920"><strong>14讲</strong></a>  里<strong>，</strong>我强调了熟练掌握Makefile语法的重要性，还推荐你去学习陈皓老师编写的<a href="https://github.com/seisman/how-to-write-makefile">《跟我一起写 Makefile》 (PDF 重制版)</a>。也许你已经点开了链接，看到那么多Makefile语法，是不是有点被“劝退”的感觉？</p><p>其实在我看来，虽然Makefile有很多语法，但不是所有的语法都需要你熟练掌握，有些语法在Go项目中是很少用到的。要编写一个高质量的Makefile，首先应该掌握一些核心的、最常用的语法知识。这一讲我就来具体介绍下Go项目中常用的Makefile语法和规则，帮助你快速打好最重要的基础。</p><p>Makefile文件由三个部分组成，分别是Makefile规则、Makefile语法和Makefile命令（这些命令可以是Linux命令，也可以是可执行的脚本文件）。在这一讲里，我会介绍下Makefile规则和Makefile语法里的一些核心语法知识。在介绍这些语法知识之前，我们先来看下如何使用Makefile脚本。</p><h2>Makefile的使用方法</h2><p>在实际使用过程中，我们一般是先编写一个Makefile文件，指定整个项目的编译规则，然后通过Linux make命令来解析该Makefile文件，实现项目编译、管理的自动化。</p><!-- [[[read_end]]] --><p>默认情况下，make命令会在当前目录下，按照GNUmakefile、makefile、Makefile文件的顺序查找Makefile文件，一旦找到，就开始读取这个文件并执行。</p><p>大多数的make都支持“makefile”和“Makefile”这两种文件名，但<strong>我建议使用“Makefile”</strong>。因为这个文件名第一个字符大写，会很明显，容易辨别。make也支持 <code>-f</code> 和 <code>--file</code> 参数来指定其他文件名，比如 <code>make -f golang.mk</code> 或者 <code>make --file golang.mk</code> 。</p><h2>Makefile规则介绍</h2><p>学习Makefile，最核心的就是学习Makefile的规则。规则是Makefile中的重要概念，它一般由目标、依赖和命令组成，用来指定源文件编译的先后顺序。Makefile之所以受欢迎，核心原因就是Makefile规则，因为Makefile规则可以自动判断是否需要重新编译某个目标，从而确保目标仅在需要时编译。</p><p>这一讲我们主要来看Makefile规则里的规则语法、伪目标和order-only依赖。</p><h3>规则语法</h3><p>Makefile的规则语法，主要包括target、prerequisites和command，示例如下：</p><pre><code>target ...: prerequisites ...
    command
	  ...
	  ...
</code></pre><p><strong>target，</strong>可以是一个object file（目标文件），也可以是一个执行文件，还可以是一个标签（label）。target可使用通配符，当有多个目标时，目标之间用空格分隔。</p><p><strong>prerequisites，</strong>代表生成该target所需要的依赖项。当有多个依赖项时，依赖项之间用空格分隔。</p><p><strong>command</strong>，代表该target要执行的命令（可以是任意的shell命令）。</p><ul>
<li>在执行command之前，默认会先打印出该命令，然后再输出命令的结果；如果不想打印出命令，可在各个command前加上<code>@</code>。</li>
<li>command可以为多条，也可以分行写，但每行都要以tab键开始。另外，如果后一条命令依赖前一条命令，则这两条命令需要写在同一行，并用分号进行分隔。</li>
<li>如果要忽略命令的出错，需要在各个command之前加上减号<code>-</code>。</li>
</ul><p><strong>只要targets不存在，或prerequisites中有一个以上的文件比targets文件新，那么command所定义的命令就会被执行，从而产生我们需要的文件，或执行我们期望的操作。</strong></p><p>我们直接通过一个例子来理解下Makefile的规则吧。</p><p>第一步，先编写一个hello.c文件。</p><pre><code>#include &lt;stdio.h&gt;
int main()
{
  printf(&quot;Hello World!\n&quot;);
  return 0;
}
</code></pre><p>第二步，在当前目录下，编写Makefile文件。</p><pre><code>hello: hello.o
	gcc -o hello hello.o

hello.o: hello.c
	gcc -c hello.c

clean:
	rm hello.o
</code></pre><p>第三步，执行make，产生可执行文件。</p><pre><code>$ make
gcc -c hello.c
gcc -o hello hello.o
$ ls
hello  hello.c  hello.o  Makefile
</code></pre><p>上面的示例Makefile文件有两个target，分别是hello和hello.o，每个target都指定了构建command。当执行make命令时，发现hello、hello.o文件不存在，就会执行command命令生成target。</p><p>第四步，不更新任何文件，再次执行make。</p><pre><code>$ make
make: 'hello' is up to date.
</code></pre><p>当target存在，并且prerequisites都不比target新时，不会执行对应的command。</p><p>第五步，更新hello.c，并再次执行make。</p><pre><code>$ touch hello.c
$ make
gcc -c hello.c
gcc -o hello hello.o
</code></pre><p>当target存在，但 prerequisites 比 target 新时，会重新执行对应的command。</p><p>第六步，清理编译中间文件。</p><p>Makefile一般都会有一个clean伪目标，用来清理编译中间产物，或者对源码目录做一些定制化的清理：</p><pre><code>$ make clean
rm hello.o
</code></pre><p>我们可以在规则中使用通配符，make 支持三个通配符：*，?和~，例如：</p><pre><code>objects = *.o
print: *.c
    rm *.c
</code></pre><h3>伪目标</h3><p>接下来我们介绍下Makefile中的伪目标。Makefile的管理能力基本上都是通过伪目标来实现的。</p><p>在上面的Makefile示例中，我们定义了一个clean目标，这其实是一个伪目标，也就是说我们不会为该目标生成任何文件。因为伪目标不是文件，make 无法生成它的依赖关系，也无法决定是否要执行它。</p><p>通常情况下，我们需要显式地标识这个目标为伪目标。在Makefile中可以使用<code>.PHONY</code>来标识一个目标为伪目标：</p><pre><code>.PHONY: clean
clean:
    rm hello.o
</code></pre><p>伪目标可以有依赖文件，也可以作为“默认目标”，例如：</p><pre><code>.PHONY: all
all: lint test build
</code></pre><p>因为伪目标总是会被执行，所以其依赖总是会被决议。通过这种方式，可以达到<strong>同时执行所有依赖项</strong>的目的。</p><h3>order-only依赖</h3><p>在上面介绍的规则中，只要prerequisites中有任何文件发生改变，就会重新构造target。但是有时候，我们希望<strong>只有当prerequisites中的部分文件改变时，才重新构造target。</strong>这时，你可以通过order-only prerequisites实现。</p><p>order-only prerequisites的形式如下：</p><pre><code>targets : normal-prerequisites | order-only-prerequisites
    command
    ...
    ...
</code></pre><p>在上面的规则中，只有第一次构造targets时，才会使用order-only-prerequisites。后面即使order-only-prerequisites发生改变，也不会重新构造targets。</p><p>只有normal-prerequisites中的文件发生改变时，才会重新构造targets。这里，符号“ | ”后面的prerequisites就是order-only-prerequisites。</p><p>到这里，我们就介绍了Makefile的规则。接下来，我们再来看下Makefile中的一些核心语法知识。</p><h2>Makefile语法概览</h2><p>因为Makefile的语法比较多，这一讲只介绍Makefile的核心语法，以及 IAM项目的Makefile用到的语法，包括命令、变量、条件语句和函数。因为Makefile没有太多复杂的语法，你掌握了这些知识点之后，再在实践中多加运用，融会贯通，就可以写出非常复杂、功能强大的Makefile文件了。</p><h3>命令</h3><p>Makefile支持Linux命令，调用方式跟在Linux系统下调用命令的方式基本一致。默认情况下，make会把正在执行的命令输出到当前屏幕上。但我们可以通过在命令前加<code>@</code>符号的方式，禁止make输出当前正在执行的命令。</p><p>我们看一个例子。现在有这么一个Makefile：</p><pre><code>.PHONY: test
test:
    echo &quot;hello world&quot;
</code></pre><p>执行make命令：</p><pre><code>$ make test
echo &quot;hello world&quot;
hello world
</code></pre><p>可以看到，make输出了执行的命令。很多时候，我们不需要这样的提示，因为我们更想看的是命令产生的日志，而不是执行的命令。这时就可以在命令行前加<code>@</code>，禁止make输出所执行的命令：</p><pre><code>.PHONY: test
test:
    @echo &quot;hello world&quot;
</code></pre><p>再次执行make命令：</p><pre><code>$ make test
hello world
</code></pre><p>可以看到，make只是执行了命令，而没有打印命令本身。这样make输出就清晰了很多。</p><p>这里，<strong>我建议在命令前都加</strong><code>@</code>符号，禁止打印命令本身，以保证你的Makefile输出易于阅读的、有用的信息。</p><p>默认情况下，每条命令执行完make就会检查其返回码。如果返回成功（返回码为0），make就执行下一条指令；如果返回失败（返回码非0），make就会终止当前命令。很多时候，命令出错（比如删除了一个不存在的文件）时，我们并不想终止，这时就可以在命令行前加 <code>-</code> 符号，来让make忽略命令的出错，以继续执行下一条命令，比如：</p><pre><code>clean:
    -rm hello.o
</code></pre><h3>变量</h3><p>变量，可能是Makefile中使用最频繁的语法了，Makefile支持变量赋值、多行变量和环境变量。另外，Makefile还内置了一些特殊变量和自动化变量。</p><p>我们先来看下最基本的<strong>变量赋值</strong>功能。</p><p>Makefile也可以像其他语言一样支持变量。在使用变量时，会像shell变量一样原地展开，然后再执行替换后的内容。</p><p>Makefile可以通过变量声明来声明一个变量，变量在声明时需要赋予一个初值，比如<code>ROOT_PACKAGE=github.com/marmotedu/iam</code>。</p><p>引用变量时可以通过<code>$()</code>或者<code>${}</code>方式引用。我的建议是，用<code>$()</code>方式引用变量，例如<code>$(ROOT_PACKAGE)</code>，也建议整个makefile的变量引用方式保持一致。</p><p>变量会像bash变量一样，在使用它的地方展开。比如：</p><pre><code>GO=go
build:
    $(GO) build -v .
</code></pre><p>展开后为：</p><pre><code>GO=go
build:
    go build -v .
</code></pre><p>接下来，我给你介绍下Makefile中的4种变量赋值方法。</p><ol>
<li><code>=</code> 最基本的赋值方法。</li>
</ol><p>例如：</p><pre><code>BASE_IMAGE = alpine:3.10
</code></pre><p>使用 <code>=</code> 进行赋值时，要注意下面这样的情况：</p><pre><code>A = a
B = $(A) b
A = c
</code></pre><p>B最后的值为 c b，而不是a b。也就是说，在用变量给变量赋值时，右边变量的取值，取的是最终的变量值。</p><ol start="2">
<li><code>:=</code>直接赋值，赋予当前位置的值。</li>
</ol><p>例如：</p><pre><code>A = a
B := $(A) b
A = c
</code></pre><p>B最后的值为 a b。通过 <code>:=</code> 的赋值方式，可以避免 <code>=</code> 赋值带来的潜在的不一致。</p><ol start="3">
<li><code>?=</code> 表示如果该变量没有被赋值，则赋予等号后的值。</li>
</ol><p>例如：</p><pre><code>PLATFORMS ?= linux_amd64 linux_arm64
</code></pre><ol start="4">
<li><code>+=</code>表示将等号后面的值添加到前面的变量上。</li>
</ol><p>例如：</p><pre><code>MAKEFLAGS += --no-print-directory
</code></pre><p>Makefile还支持<strong>多行变量</strong>。可以通过define关键字设置多行变量，变量中允许换行。定义方式为：</p><pre><code>define 变量名
变量内容
...
endef
</code></pre><p>变量的内容可以包含函数、命令、文字或是其他变量。例如，我们可以定义一个USAGE_OPTIONS变量：</p><pre><code>define USAGE_OPTIONS

Options:
  DEBUG        Whether to generate debug symbols. Default is 0.
  BINS         The binaries to build. Default is all of cmd.
  ...
  V            Set to 1 enable verbose build. Default is 0.
endef
</code></pre><p>Makefile还支持<strong>环境变量</strong>。在Makefile中，有两种环境变量，分别是Makefile预定义的环境变量和自定义的环境变量。</p><p>其中，自定义的环境变量可以覆盖Makefile预定义的环境变量。默认情况下，Makefile中定义的环境变量只在当前Makefile有效，如果想向下层传递（Makefile中调用另一个Makefile），需要使用export关键字来声明。</p><p>下面的例子声明了一个环境变量，并可以在下层Makefile中使用：</p><pre><code>...
export USAGE_OPTIONS
...
</code></pre><p>此外，Makefile还支持两种内置的变量：特殊变量和自动化变量。</p><p><strong>特殊变量</strong>是make提前定义好的，可以在makefile中直接引用。特殊变量列表如下：</p><p><img src="https://static001.geekbang.org/resource/image/c1/1d/c1cba21aaed2eb0117yyb0470byy641d.png?wh=1052x978" alt=""></p><p>Makefile还支持<strong>自动化变量</strong>。自动化变量可以提高我们编写Makefile的效率和质量。</p><p>在Makefile的模式规则中，目标和依赖文件都是一系列的文件，那么我们如何书写一个命令，来完成从不同的依赖文件生成相对应的目标呢？</p><p>这时就可以用到自动化变量。所谓自动化变量，就是这种变量会把模式中所定义的一系列的文件自动地挨个取出，一直到所有符合模式的文件都取完为止。这种自动化变量只应出现在规则的命令中。Makefile中支持的自动化变量见下表。</p><p><img src="https://static001.geekbang.org/resource/image/13/12/13ec33008eaff973c0dd854a795ff712.png?wh=1263x1303" alt=""></p><p>上面这些自动化变量中，<code>$*</code>是用得最多的。<code>$*</code> 对于构造有关联的文件名是比较有效的。如果目标中没有模式的定义，那么 <code>$*</code> 也就不能被推导出。但是，如果目标文件的后缀是make所识别的，那么 <code>$*</code> 就是除了后缀的那一部分。例如：如果目标是foo.c ，因为.c是make所能识别的后缀名，所以 <code>$*</code> 的值就是foo。</p><h3>条件语句</h3><p>Makefile也支持条件语句。这里先看一个示例。</p><p>下面的例子判断变量<code>ROOT_PACKAGE</code>是否为空，如果为空，则输出错误信息，不为空则打印变量值：</p><pre><code>ifeq ($(ROOT_PACKAGE),)
$(error the variable ROOT_PACKAGE must be set prior to including golang.mk)
else
$(info the value of ROOT_PACKAGE is $(ROOT_PACKAGE))
endif
</code></pre><p>条件语句的语法为：</p><pre><code># if ...
&lt;conditional-directive&gt;
&lt;text-if-true&gt;
endif
# if ... else ...
&lt;conditional-directive&gt;
&lt;text-if-true&gt;
else
&lt;text-if-false&gt;
endif
</code></pre><p>例如，判断两个值是否相等：</p><pre><code>ifeq 条件表达式
...
else
...
endif
</code></pre><ul>
<li>ifeq表示条件语句的开始，并指定一个条件表达式。表达式包含两个参数，参数之间用逗号分隔，并且表达式用圆括号括起来。</li>
<li>else表示条件表达式为假的情况。</li>
<li>endif表示一个条件语句的结束，任何一个条件表达式都应该以endif结束。</li>
<li><conditional-directive>表示条件关键字，有4个关键字：ifeq、ifneq、ifdef、ifndef。</conditional-directive></li>
</ul><p>为了加深你的理解，我们分别来看下这4个关键字的例子。</p><ol>
<li>ifeq：条件判断，判断是否相等。</li>
</ol><p>例如：</p><pre><code>ifeq (&lt;arg1&gt;, &lt;arg2&gt;)
ifeq '&lt;arg1&gt;' '&lt;arg2&gt;'
ifeq &quot;&lt;arg1&gt;&quot; &quot;&lt;arg2&gt;&quot;
ifeq &quot;&lt;arg1&gt;&quot; '&lt;arg2&gt;'
ifeq '&lt;arg1&gt;' &quot;&lt;arg2&gt;&quot;
</code></pre><p>比较arg1和arg2的值是否相同，如果相同则为真。也可以用make函数/变量替代arg1或arg2，例如 <code>ifeq ($(origin ROOT_DIR),undefined)</code> 或 <code>ifeq ($(ROOT_PACKAGE),)</code> 。origin函数会在之后专门讲函数的一讲中介绍到。</p><ol start="2">
<li>ifneq：条件判断，判断是否不相等。</li>
</ol><pre><code>ifneq (&lt;arg1&gt;, &lt;arg2&gt;)
ifneq '&lt;arg1&gt;' '&lt;arg2&gt;'
ifneq &quot;&lt;arg1&gt;&quot; &quot;&lt;arg2&gt;&quot;
ifneq &quot;&lt;arg1&gt;&quot; '&lt;arg2&gt;'
ifneq '&lt;arg1&gt;' &quot;&lt;arg2&gt;&quot;
</code></pre><p>比较arg1和arg2的值是否不同，如果不同则为真。</p><ol start="3">
<li>ifdef：条件判断，判断变量是否已定义。</li>
</ol><pre><code>ifdef &lt;variable-name&gt;
</code></pre><p>如果<variable-name>值非空，则表达式为真，否则为假。<variable-name>也可以是函数的返回值。</variable-name></variable-name></p><ol start="4">
<li>ifndef：条件判断，判断变量是否未定义。</li>
</ol><pre><code>ifndef &lt;variable-name&gt;
</code></pre><p>如果<variable-name>值为空，则表达式为真，否则为假。<variable-name>也可以是函数的返回值。</variable-name></variable-name></p><h3>函数</h3><p>Makefile同样也支持函数，函数语法包括定义语法和调用语法。</p><p><strong>我们先来看下自定义函数。</strong> make解释器提供了一系列的函数供Makefile调用，这些函数是Makefile的预定义函数。我们可以通过define关键字来自定义一个函数。自定义函数的语法为：</p><pre><code>define 函数名
函数体
endef
</code></pre><p>例如，下面这个自定义函数：</p><pre><code>define Foo
    @echo &quot;my name is $(0)&quot;
    @echo &quot;param is $(1)&quot;
endef
</code></pre><p>define本质上是定义一个多行变量，可以在call的作用下当作函数来使用，在其他位置使用只能作为多行变量来使用，例如：</p><pre><code>var := $(call Foo)
new := $(Foo)
</code></pre><p>自定义函数是一种过程调用，没有任何的返回值。可以使用自定义函数来定义命令的集合，并应用在规则中。</p><p><strong>再来看下预定义函数。</strong> 刚才提到，make编译器也定义了很多函数，这些函数叫作预定义函数，调用语法和变量类似，语法为：</p><pre><code>$(&lt;function&gt; &lt;arguments&gt;)
</code></pre><p>或者</p><pre><code>${&lt;function&gt; &lt;arguments&gt;}
</code></pre><p><code>&lt;function&gt;</code>是函数名，<code>&lt;arguments&gt;</code>是函数参数，参数间用逗号分割。函数的参数也可以是变量。</p><p>我们来看一个例子：</p><pre><code>PLATFORM = linux_amd64
GOOS := $(word 1, $(subst _, ,$(PLATFORM)))
</code></pre><p>上面的例子用到了两个函数：word和subst。word函数有两个参数，1和subst函数的输出。subst函数将PLATFORM变量值中的_替换成空格（替换后的PLATFORM值为linux amd64）。word函数取linux amd64字符串中的第一个单词。所以最后GOOS的值为linux。</p><p>Makefile预定义函数能够帮助我们实现很多强大的功能，在编写Makefile的过程中，如果有功能需求，可以优先使用这些函数。如果你想使用这些函数，那就需要知道有哪些函数，以及它们实现的功能。</p><p>常用的函数包括下面这些，你需要先有个印象，以后用到时再来查看。</p><p><img src="https://static001.geekbang.org/resource/image/96/5f/96da0853e8225a656d2c0489e544865f.jpg?wh=2248x3692" alt=""></p><h2>引入其他Makefile</h2><p>除了Makefile规则、Makefile语法之外，Makefile还有很多特性，比如可以引入其他Makefile、自动生成依赖关系、文件搜索等等。这里我再介绍一个IAM项目的Makefile用到的重点特性：引入其他Makefile。</p><p>在 <a href="https://time.geekbang.org/column/article/388920"><strong>14讲</strong></a> 中，我们介绍过Makefile要结构化、层次化，这一点可以通过<strong>在项目根目录下的Makefile中引入其他Makefile</strong>来实现。</p><p>在Makefile中，我们可以通过关键字include，把别的makefile包含进来，类似于C语言的<code>#include</code>，被包含的文件会插入在当前的位置。include用法为<code>include &lt;filename&gt;</code>，示例如下：</p><pre><code>include scripts/make-rules/common.mk
include scripts/make-rules/golang.mk
</code></pre><p>include也可以包含通配符<code>include scripts/make-rules/*</code>。make命令会按下面的顺序查找makefile文件：</p><ol>
<li>如果是绝对或相对路径，就直接根据路径include进来。</li>
<li>如果make执行时，有<code>-I</code>或<code>--include-dir</code>参数，那么make就会在这个参数所指定的目录下去找。</li>
<li>如果目录<code>&lt;prefix&gt;/include</code>（一般是<code>/usr/local/bin</code>或<code>/usr/include</code>）存在的话，make也会去找。</li>
</ol><p>如果有文件没有找到，make会生成一条警告信息，但不会马上出现致命错误，而是继续载入其他的文件。一旦完成makefile的读取，make会再重试这些没有找到或是不能读取的文件。如果还是不行，make才会出现一条致命错误信息。如果你想让make忽略那些无法读取的文件继续执行，可以在include前加一个减号<code>-</code>，如<code>-include &lt;filename&gt;</code>。</p><h2>总结</h2><p>在这一讲里，为了帮助你编写一个高质量的Makefile，我重点介绍了Makefile规则和Makefile语法里的一些核心语法知识。</p><p>在讲Makefile规则时，我们主要学习了规则语法、伪目标和order-only依赖。掌握了这些Makefile规则，你就掌握了Makefile中最核心的内容。</p><p>在介绍Makefile的语法时，我只介绍了Makefile的核心语法，以及 IAM项目的Makefile用到的语法，包括命令、变量、条件语句和函数。你可能会觉得这些语法学习起来比较枯燥，但还是那句话，工欲善其事，必先利其器。希望你能熟练掌握Makefile的核心语法，为编写高质量的Makefile打好基础。</p><p>今天的内容就到这里啦，欢迎你在下面的留言区谈谈自己的看法，我们下一讲见。</p>
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
  <div class="_2_QraFYR_0">我觉得变量用${}, 函数用$(),这样能很好区分，对熟悉shell的人也更友好</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 听起来没毛病，保持一致即可</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-28 00:45:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/52/40/e57a736e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>pedro</span>
  </div>
  <div class="_2_QraFYR_0">周一就更新，值😂</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">编辑回复: 这篇是额外的加餐，不占排期哦</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-28 08:08:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/6b/90/ca3bf377.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>成楼</span>
  </div>
  <div class="_2_QraFYR_0">有学员群吗？可以加入下吗？方便交流和提升，谢谢。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以看下iam仓库根目录下的README文档，最后有微信号</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-16 16:48:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/04/60/64d166b6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Fan</span>
  </div>
  <div class="_2_QraFYR_0">多行变量 的例子没明白<br><br><br>define USAGE_OPTIONS<br><br>Options:<br>  DEBUG        Whether to generate debug symbols. Default is 0.<br>  BINS         The binaries to build. Default is all of cmd.<br>  ...<br>  V            Set to 1 enable verbose build. Default is 0.<br>endef</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: USAGE_OPTIONS的值可以是多行字符串</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-11 16:07:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/01/8c/41adb537.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Tiandh</span>
  </div>
  <div class="_2_QraFYR_0">老师，这两句话不理解<br>因为伪目标不是文件，make 无法生成它的依赖关系，也无法决定是否要执行它。<br>因为伪目标总是会被执行，所以其依赖总是会被决议。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 就是伪目标总是会被执行</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-28 09:30:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/7b/c5/35f92dad.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Jone_乔泓恺</span>
  </div>
  <div class="_2_QraFYR_0">老师，有问格式问题想问下：ifeq 语句中的内容建议要用 tab，还是顶格呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 用tab好些</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-01 10:46:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/7b/c5/35f92dad.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Jone_乔泓恺</span>
  </div>
  <div class="_2_QraFYR_0">ifeq ($(origin ROOT_DIR),undefined)<br>ROOT_DIR := $(abspath $(shell cd $(COMMON_SELF_DIR)&#47;..&#47;.. &amp;&amp; pwd -P))<br>endif<br>和 <br>ROOT_DIR ?= $(abspath $(shell cd $(COMMON_SELF_DIR)&#47;..&#47;.. &amp;&amp; pwd -P))<br><br>请问：这两种方式的效果是否相同？<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是一样的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-30 16:59:23</div>
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
  <div class="_2_QraFYR_0">大家有什么不懂的，可以结合：陈皓老师编写的《跟我一起写 Makefile》 (PDF 重制版) 来看，本文受限于篇幅，有些概念可能不能花很大的篇幅去讲解。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 陈皓老师编写的这份Makefile指南很详尽</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-29 17:17:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/6b/90/ca3bf377.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>成楼</span>
  </div>
  <div class="_2_QraFYR_0">由于缺少云上开发资源，本地虚拟机的环境配置能否给些建议，可以创建一个群方便交流吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以看下IAM项目根目录下的README文档，有微信号</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-16 16:54:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/7f/d3/b5896293.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Realm</span>
  </div>
  <div class="_2_QraFYR_0">老师，函数dir ，nodir注释好像有笔误.</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 没找到你说的哪个位置，可以再明确下位置吗，thanks</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-10 08:41:44</div>
  </div>
</div>
</div>
</li>
</ul>