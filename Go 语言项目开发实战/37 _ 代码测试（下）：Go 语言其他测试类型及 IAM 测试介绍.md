<audio title="37 _ 代码测试（下）：Go 语言其他测试类型及 IAM 测试介绍" src="https://static001.geekbang.org/resource/audio/dc/1f/dc614f7b1c58999dff7935eeeb1eea1f.mp3" controls="controls"></audio> 
<p>你好，我是孔令飞。</p><p><a href="https://time.geekbang.org/column/article/408529">上一讲</a>，我介绍了Go中的两类测试：单元测试和性能测试。在Go中，还有一些其他的测试类型和测试方法，值得我们去了解和掌握。此外，IAM项目也编写了大量测试用例，这些测试用例使用了不同的编写方法，你可以通过学习IAM的测试用例来验证你学到的测试知识。</p><p>今天，我就来介绍下Go 语言中的其他测试类型：示例测试、TestMain函数、Mock测试、Fake测试等，并且介绍下IAM项目是如何编写和运行测试用例的。</p><h2>示例测试</h2><p>示例测试以<code>Example</code>开头，没有输入和返回参数，通常保存在<code>example_test.go</code>文件中。示例测试可能包含以<code>Output:</code>或者<code>Unordered output:</code>开头的注释，这些注释放在函数的结尾部分。<code>Unordered output:</code>开头的注释会忽略输出行的顺序。</p><p>执行<code>go test</code>命令时，会执行这些示例测试，并且go test会将示例测试输出到标准输出的内容，跟注释作对比（比较时将忽略行前后的空格）。如果相等，则示例测试通过测试；如果不相等，则示例测试不通过测试。下面是一个示例测试（位于example_test.go文件中）：</p><pre><code class="language-go">func ExampleMax() {
    fmt.Println(Max(1, 2))
    // Output:
    // 2
}
</code></pre><!-- [[[read_end]]] --><p>执行go test命令，测试<code>ExampleMax</code>示例测试：</p><pre><code class="language-bash">$ go test -v -run='Example.*'
=== RUN   ExampleMax
--- PASS: ExampleMax (0.00s)
PASS
ok      github.com/marmotedu/gopractise-demo/31/test    0.004s
</code></pre><p>可以看到<code>ExampleMax</code>测试通过。这里测试通过是因为<code>fmt.Println(Max(1, 2))</code>向标准输出输出了<code>2</code>，跟<code>// Output:</code>后面的<code>2</code>一致。</p><p>当示例测试不包含<code>Output:</code>或者<code>Unordered output:</code>注释时，执行<code>go test</code>只会编译这些函数，但不会执行这些函数。</p><h3>示例测试命名规范</h3><p>示例测试需要遵循一些命名规范，因为只有这样，Godoc才能将示例测试和包级别的标识符进行关联。例如，有以下示例测试（位于example_test.go文件中）：</p><pre><code class="language-go">package stringutil_test

import (
    "fmt"

    "github.com/golang/example/stringutil"
)

func ExampleReverse() {
    fmt.Println(stringutil.Reverse("hello"))
    // Output: olleh
}
</code></pre><p>Godoc将在<code>Reverse</code>函数的文档旁边提供此示例，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/d8/93/d8ae5e99fe1d159e9b3ba1f815b24693.png?wh=540x374" alt="图片"></p><p>示例测试名以<code>Example</code>开头，后面可以不跟任何字符串，也可以跟函数名、类型名或者<code>类型_方法名</code>，中间用下划线<code>_</code>连接，例如：</p><pre><code class="language-go">func Example() { ... } // 代表了整个包的示例
func ExampleF() { ... } // 函数F的示例
func ExampleT() { ... } // 类型T的示例
func ExampleT_M() { ... } // 方法T_M的示例
</code></pre><p>当某个函数/类型/方法有多个示例测试时，可以通过后缀来区分，后缀必须以小写字母开头，例如：</p><pre><code class="language-go">func ExampleReverse()
func ExampleReverse_second()
func ExampleReverse_third()
</code></pre><h3>大型示例</h3><p>有时候，我们需要编写一个大型的示例测试，这时候我们可以编写一个整文件的示例（whole file example），它有这几个特点：文件名以<code>_test.go</code>结尾；只包含一个示例测试，文件中没有单元测试函数和性能测试函数；至少包含一个包级别的声明；当展示这类示例测试时，godoc会直接展示整个文件。例如：</p><pre><code class="language-go">package sort_test

import (
    "fmt"
    "sort"
)

type Person struct {
    Name string
    Age  int
}

func (p Person) String() string {
    return fmt.Sprintf("%s: %d", p.Name, p.Age)
}

// ByAge implements sort.Interface for []Person based on
// the Age field.
type ByAge []Person

func (a ByAge) Len() int           { return len(a) }
func (a ByAge) Swap(i, j int)      { a[i], a[j] = a[j], a[i] }
func (a ByAge) Less(i, j int) bool { return a[i].Age &lt; a[j].Age }

func Example() {
    people := []Person{
        {"Bob", 31},
        {"John", 42},
        {"Michael", 17},
        {"Jenny", 26},
    }

    fmt.Println(people)
    sort.Sort(ByAge(people))
    fmt.Println(people)

    // Output:
    // [Bob: 31 John: 42 Michael: 17 Jenny: 26]
    // [Michael: 17 Jenny: 26 Bob: 31 John: 42]
}
</code></pre><p>一个包可以包含多个whole file example，一个示例一个文件，例如<code>example_interface_test.go</code>、<code>example_keys_test.go</code>、<code>example_search_test.go</code>等。</p><h2>TestMain函数</h2><p>有时候，我们在做测试的时候，可能会在测试之前做些准备工作，例如创建数据库连接等；在测试之后做些清理工作，例如关闭数据库连接、清理测试文件等。这时，我们可以在<code>_test.go</code>文件中添加<code>TestMain</code>函数，其入参为<code>*testing.M</code>。</p><p><code>TestMain</code>是一个特殊的函数（相当于main函数），测试用例在执行时，会先执行<code>TestMain</code>函数，然后可以在<code>TestMain</code>中调用<code>m.Run()</code>函数执行普通的测试函数。在<code>m.Run()</code>函数前面我们可以编写准备逻辑，在<code>m.Run()</code>后面我们可以编写清理逻辑。</p><p>我们在示例测试文件<a href="https://github.com/marmotedu/gopractise-demo/blob/master/test/math_test.go">math_test.go</a>中添加如下TestMain函数：</p><pre><code class="language-go">func TestMain(m *testing.M) {
    fmt.Println("do some setup")
    m.Run()
    fmt.Println("do some cleanup")
}
</code></pre><p>执行go test，输出如下：</p><pre><code class="language-bash">$ go test -v
do some setup
=== RUN   TestAbs
--- PASS: TestAbs (0.00s)
...
=== RUN   ExampleMax
--- PASS: ExampleMax (0.00s)
PASS
do some cleanup
ok  	github.com/marmotedu/gopractise-demo/31/test	0.006s
</code></pre><p>在执行测试用例之前，打印了<code>do some setup</code>，在测试用例运行完成之后，打印了<code>do some cleanup</code>。</p><p>IAM项目的测试用例中，使用TestMain函数在执行测试用例前连接了一个fake数据库，代码如下（位于<a href="https://github.com/marmotedu/iam/blob/v1.0.8/internal/apiserver/service/v1/user_test.go">internal/apiserver/service/v1/user_test.go</a>文件中）：</p><pre><code class="language-go">func TestMain(m *testing.M) {
    fakeStore, _ := fake.NewFakeStore()
    store.SetClient(fakeStore)
    os.Exit(m.Run())
}
</code></pre><p>单元测试、性能测试、示例测试、TestMain函数是go test支持的测试类型。此外，为了测试在函数内使用了Go Interface的函数，我们还延伸出了Mock测试和Fake测试两种测试类型。</p><h2>Mock测试</h2><p>一般来说，单元测试中是不允许有外部依赖的，那么也就是说，这些外部依赖都需要被模拟。在Go中，一般会借助各类Mock工具来模拟一些依赖。</p><p>GoMock是由Golang官方开发维护的测试框架，实现了较为完整的基于interface的Mock功能，能够与Golang内置的testing包良好集成，也能用于其他的测试环境中。GoMock测试框架包含了GoMock包和mockgen工具两部分，其中GoMock包用来完成对象生命周期的管理，mockgen工具用来生成interface对应的Mock类源文件。下面，我来分别详细介绍下GoMock包和mockgen工具，以及它们的使用方法。</p><h3>安装GoMock</h3><p>要使用GoMock，首先需要安装GoMock包和mockgen工具，安装方法如下:</p><pre><code class="language-bash">$ go get github.com/golang/mock/gomock
$ go install github.com/golang/mock/mockgen
</code></pre><p>下面，我通过一个<strong>获取当前Golang最新版本的例子</strong>，来给你演示下如何使用GoMock。示例代码目录结构如下（目录下的代码见<a href="https://github.com/marmotedu/gopractise-demo/tree/master/gomock">gomock</a>）：</p><pre><code class="language-bash">tree .
.
├── go_version.go
├── main.go
└── spider
    └── spider.go
</code></pre><p><code>spider.go</code>文件中定义了一个<code>Spider</code>接口，<code>spider.go</code>代码如下：</p><pre><code class="language-go">package spider

type Spider interface {
    GetBody() string
}
</code></pre><p><code>Spider</code>接口中的GetBody方法可以抓取<code>https://golang.org</code>首页的<code>Build version</code>字段，来获取Golang的最新版本。</p><p>我们在<code>go_version.go</code>文件中，调用<code>Spider</code>接口的<code>GetBody</code>方法，<code>go_version.go</code>代码如下：</p><pre><code class="language-go">package gomock

import (
    "github.com/marmotedu/gopractise-demo/gomock/spider"
)

func GetGoVersion(s spider.Spider) string {
    body := s.GetBody()
    return body
}
</code></pre><p><code>GetGoVersion</code>函数直接返回表示版本的字符串。正常情况下，我们会写出如下的单元测试代码：</p><pre><code class="language-go">func TestGetGoVersion(t *testing.T) {
    v := GetGoVersion(spider.CreateGoVersionSpider())
    if v != "go1.8.3" {
        t.Error("Get wrong version %s", v)
    }
}
</code></pre><p>上面的测试代码，依赖<code>spider.CreateGoVersionSpider()</code>返回一个实现了<code>Spider</code>接口的实例（爬虫）。但很多时候，<code>spider.CreateGoVersionSpider()</code>爬虫可能还没有实现，或者在单元测试环境下不能运行（比如，在单元测试环境中连接数据库），这时候<code>TestGetGoVersion</code>测试用例就无法执行。</p><p>那么，如何才能在这种情况下运行<code>TestGetGoVersion</code>测试用例呢？这时候，我们就可以通过Mock工具，Mock一个爬虫实例。接下来我讲讲具体操作。</p><p>首先，用 GoMock 提供的mockgen工具，生成要 Mock 的接口的实现，我们在gomock目录下执行以下命令：</p><pre><code class="language-bash">$ mockgen -destination spider/mock/mock_spider.go -package spider github.com/marmotedu/gopractise-demo/gomock/spider Spider
</code></pre><p>上面的命令会在<code>spider/mock</code>目录下生成<code>mock_spider.go</code>文件：</p><pre><code class="language-bash">$ tree .
.
├── go_version.go
├── go_version_test.go
├── go_version_test_traditional_method.go~
└── spider
    ├── mock
    │&nbsp;&nbsp; └── mock_spider.go
    └── spider.go
</code></pre><p><code>mock_spider.go</code>文件中，定义了一些函数/方法，可以支持我们编写<code>TestGetGoVersion</code>测试函数。这时候，我们的单元测试代码如下（见<a href="https://github.com/marmotedu/gopractise-demo/blob/master/gomock/go_version_test.go">go_version_test.go</a>文件）：</p><pre><code class="language-go">package gomock

import (
	"testing"

	"github.com/golang/mock/gomock"

	spider "github.com/marmotedu/gopractise-demo/gomock/spider/mock"
)

func TestGetGoVersion(t *testing.T) {
	ctrl := gomock.NewController(t)
	defer ctrl.Finish()

	mockSpider := spider.NewMockSpider(ctrl)
	mockSpider.EXPECT().GetBody().Return("go1.8.3")
	goVer := GetGoVersion(mockSpider)

	if goVer != "go1.8.3" {
		t.Errorf("Get wrong version %s", goVer)
	}
}
</code></pre><p>这一版本的<code>TestGetGoVersion</code>通过GoMock， Mock了一个<code>Spider</code>接口，而不用去实现一个<code>Spider</code>接口。这就大大降低了单元测试用例编写的复杂度。通过Mock，很多不能测试的函数也变得可测试了。</p><p>通过上面的测试用例，我们可以看到，GoMock 和<a href="https://time.geekbang.org/column/article/408529">上一讲</a>介绍的testing单元测试框架可以紧密地结合起来工作。</p><h3>mockgen工具介绍</h3><p>上面，我介绍了如何使用 GoMock 编写单元测试用例。其中，我们使用到了<code>mockgen</code>工具来生成 Mock代码，<code>mockgen</code>工具提供了很多有用的功能，这里我来详细介绍下。</p><p><code>mockgen</code>工具是 GoMock 提供的，用来Mock一个Go接口。它可以根据给定的接口，来自动生成Mock代码。这里，有两种模式可以生成Mock代码，分别是源码模式和反射模式。</p><ol>
<li>源码模式</li>
</ol><p>如果有接口文件，则可以通过以下命令来生成Mock代码：</p><pre><code class="language-bash">$ mockgen -destination spider/mock/mock_spider.go -package spider -source spider/spider.go
</code></pre><p>上面的命令，Mock了<code>spider/spider.go</code>文件中定义的<code>Spider</code>接口，并将Mock代码保存在<code>spider/mock/mock_spider.go</code>文件中，文件的包名为<code>spider</code>。</p><p>mockgen工具的参数说明见下表：</p><p><img src="https://static001.geekbang.org/resource/image/e7/9c/e72102362e2ae3225e868f125654689c.jpg?wh=1920x1210" alt="图片"></p><ol start="2">
<li>反射模式</li>
</ol><p>此外，mockgen工具还支持通过使用反射程序来生成 Mock 代码。它通过传递两个非标志参数，即导入路径和逗号分隔的接口列表来启用，其他参数和源码模式共用，例如：</p><pre><code class="language-bash">$ mockgen -destination spider/mock/mock_spider.go -package spider github.com/marmotedu/gopractise-demo/gomock/spider Spider
</code></pre><h3>通过注释使用mockgen</h3><p>如果有多个文件，并且分散在不同的位置，那么我们要生成Mock文件的时候，需要对每个文件执行多次mockgen命令（这里假设包名不相同）。这种操作还是比较繁琐的，mockgen还提供了一种通过注释生成Mock文件的方式，此时需要借助<code>go generate</code>工具。</p><p>在接口文件的代码中，添加以下注释（具体代码见<a href="https://github.com/marmotedu/gopractise-demo/blob/master/gomock/spider/spider.go#L3">spider.go</a>文件）：</p><pre><code class="language-go">//go:generate mockgen -destination mock_spider.go -package spider github.com/cz-it/blog/blog/Go/testing/gomock/example/spider Spider
</code></pre><p>这时候，我们只需要在<code>gomock</code>目录下，执行以下命令，就可以自动生成Mock代码：</p><pre><code class="language-bash">$ go generate ./...
</code></pre><h3>使用Mock代码编写单元测试用例</h3><p>生成了Mock代码之后，我们就可以使用它们了。这里我们结合<code>testing</code>来编写一个使用了Mock代码的单元测试用例。</p><p><strong>首先，</strong>需要在单元测试代码里创建一个Mock控制器：</p><pre><code class="language-go">ctrl := gomock.NewController(t)
</code></pre><p>将<code>*testing.T</code>传递给GoMock ，生成一个<code>Controller</code>对象，该对象控制了整个Mock的过程。在操作完后，还需要进行回收，所以一般会在<code>NewController</code>后面defer一个Finish，代码如下：</p><pre><code class="language-go">defer ctrl.Finish()
</code></pre><p><strong>然后，</strong>就可以调用Mock的对象了：</p><pre><code class="language-go">mockSpider := spider.NewMockSpider(ctrl)
</code></pre><p>这里的<code>spider</code>是mockgen命令里面传递的包名，后面是<code>NewMockXxxx</code>格式的对象创建函数，<code>Xxx</code>是接口名。这里，我们需要传递控制器对象进去，返回一个Mock实例。</p><p><strong>接着，</strong>有了Mock实例，我们就可以调用其断言方法<code>EXPECT()</code>了。</p><p>gomock采用了链式调用法，通过<code>.</code>连接函数调用，可以像链条一样连接下去。例如：</p><pre><code class="language-go">mockSpider.EXPECT().GetBody().Return("go1.8.3")
</code></pre><p>Mock一个接口的方法，我们需要Mock该方法的入参和返回值。我们可以通过参数匹配来Mock入参，通过Mock实例的 <code>Return</code> 方法来Mock返回值。下面，我们来分别看下如何指定入参和返回值。</p><p>先来看如何指定入参。如果函数有参数，我们可以使用参数匹配来指代函数的参数，例如：</p><pre><code class="language-go">mockSpider.EXPECT().GetBody(gomock.Any(), gomock.Eq("admin")).Return("go1.8.3")
</code></pre><p>gomock支持以下参数匹配：</p><ul>
<li>gomock.Any()，可以用来表示任意的入参。</li>
<li>gomock.Eq(value)，用来表示与 value 等价的值。</li>
<li>gomock.Not(value)，用来表示非 value 以外的值。</li>
<li>gomock.Nil()，用来表示 None 值。</li>
</ul><p>接下来，我们看如何指定返回值。</p><p><code>EXPECT()</code>得到Mock的实例，然后调用Mock实例的方法，该方法返回第一个<code>Call</code>对象，然后可以对其进行条件约束，比如使用Mock实例的 <code>Return</code> 方法约束其返回值。<code>Call</code>对象还提供了以下方法来约束Mock实例：</p><pre><code class="language-go">func (c *Call) After(preReq *Call) *Call // After声明调用在preReq完成后执行
func (c *Call) AnyTimes() *Call // 允许调用次数为 0 次或更多次
func (c *Call) Do(f interface{}) *Call // 声明在匹配时要运行的操作
func (c *Call) MaxTimes(n int) *Call // 设置最大的调用次数为 n 次
func (c *Call) MinTimes(n int) *Call // 设置最小的调用次数为 n 次
func (c *Call) Return(rets ...interface{}) *Call //  // 声明模拟函数调用返回的值
func (c *Call) SetArg(n int, value interface{}) *Call // 声明使用指针设置第 n 个参数的值
func (c *Call) Times(n int) *Call // 设置调用次数为 n 次
</code></pre><p>上面列出了多个 <code>Call</code> 对象提供的约束方法，接下来我会介绍3个常用的约束方法：指定返回值、指定执行次数和指定执行顺序。</p><ol>
<li>指定返回值</li>
</ol><p>我们可以提供调用<code>Call</code>的<code>Return</code>函数，来指定接口的返回值，例如：</p><pre><code class="language-go">mockSpider.EXPECT().GetBody().Return("go1.8.3")
</code></pre><ol start="2">
<li>指定执行次数</li>
</ol><p>有时候，我们需要指定函数执行多少次，例如：对于接受网络请求的函数，计算其执行了多少次。我们可以通过<code>Call</code>的<code>Times</code>函数来指定执行次数：</p><pre><code class="language-go">mockSpider.EXPECT().Recv().Return(nil).Times(3)
</code></pre><p>上述代码，执行了三次Recv函数，这里gomock还支持其他的执行次数限制：</p><ul>
<li>AnyTimes()，表示执行0到多次。</li>
<li>MaxTimes(n int)，表示如果没有设置，最多执行n次。</li>
<li>MinTimes(n int)，表示如果没有设置，最少执行n次。</li>
</ul><ol start="3">
<li>指定执行顺序</li>
</ol><p>有时候，我们还要指定执行顺序，比如要先执行 Init 操作，然后才能执行Recv操作：</p><pre><code class="language-go">initCall := mockSpider.EXPECT().Init()
mockSpider.EXPECT().Recv().After(initCall)
</code></pre><p>最后，我们可以使用<code>go test</code>来测试使用了Mock代码的单元测试代码：</p><pre><code class="language-bash">$ go test -v
=== RUN   TestGetGoVersion
--- PASS: TestGetGoVersion (0.00s)
PASS
ok  	github.com/marmotedu/gopractise-demo/gomock	0.002s
</code></pre><h2>Fake测试</h2><p>在Go项目开发中，对于比较复杂的接口，我们还可以Fake一个接口实现，来进行测试。所谓Fake测试，其实就是针对接口实现一个假（fake）的实例。至于如何实现Fake实例，需要你根据业务自行实现。例如：IAM项目中iam-apiserver组件就实现了一个fake store，代码见<a href="https://github.com/marmotedu/iam/tree/v1.0.8/internal/apiserver/store/fake">fake</a>目录。因为这一讲后面的IAM项目测试实战部分有介绍，所以这里不再展开讲解。</p><h2>何时编写和执行单元测试用例？</h2><p>上面，我介绍了Go代码测试的基础知识，这里我再来分享下在做测试时一个比较重要的知识点：何时编写和执行单元测试用例。</p><h3>编码前：TDD</h3><p><img src="https://static001.geekbang.org/resource/image/48/b2/4830b21b55d194eccf1ec74637ee3eb2.png?wh=538x516" alt="图片"></p><p>Test-Driven Development，也就是测试驱动开发，是敏捷开发的⼀项核心实践和技术，也是⼀种设计方法论。简单来说，TDD原理就是：开发功能代码之前，先编写测试用例代码，然后针对测试用例编写功能代码，使其能够通过。这样做的好处在于，通过测试的执行代码肯定满足需求，而且有助于面向接口编程，降低代码耦合，也极大降低了bug的出现几率。</p><p>然而，TDD的坏处也显而易见：由于测试用例是在进行代码设计之前写的，很有可能限制开发者对代码的整体设计；并且，由于TDD对开发⼈员要求非常高，体现的思想跟传统开发思维也不⼀样，因此实施起来比较困难；此外，因为要先编写测试用例，TDD也可能会影响项目的研发进度。所以，在客观情况不满足的情况下，不应该盲目追求对业务代码使用TDD的开发模式。</p><h3>与编码同步进行：增量</h3><p>及时为增量代码写单测是一种良好的习惯。一方面是因为，此时我们对需求有一定的理解，能够更好地写出单元测试来验证正确性。并且，在单测阶段就发现问题，而不是等到联调测试中才发现，修复的成本也是最小的。</p><p>另一方面，在写单测的过程中，我们也能够反思业务代码的正确性、合理性，推动我们在实现的过程中更好地反思代码的设计，并及时调整。</p><h3>编码后：存量</h3><p>在完成业务需求后，我们可能会遇到这种情况：因为上线时间比较紧张、没有单测相关规划，开发阶段只手动测试了代码是否符合功能。</p><p>如果这部分存量代码出现较大的新需求，或者维护已经成为问题，需要大规模重构，这正是推动补全单测的好时机。为存量代码补充上单测，一方面能够推进重构者进一步理解原先的逻辑，另一方面也能够增强重构者重构代码后的信心，降低风险。</p><p>但是，补充存量单测可能需要再次回忆理解需求和逻辑设计等细节，而有时写单测的人并不是原编码的设计者，所以编码后编写和执行单元测试用例也有一定的不足。</p><h2>测试覆盖率</h2><p>我们写单元测试的时候应该想得很全面，能够覆盖到所有的测试用例，但有时也会漏过一些 case，Go提供了cover工具来统计测试覆盖率。具体可以分为两大步骤。</p><p>第一步，生成测试覆盖率数据：</p><pre><code class="language-bash">$ go test -coverprofile=coverage.out
do some setup
PASS
coverage: 40.0% of statements
do some cleanup
ok  	github.com/marmotedu/gopractise-demo/test	0.003s
</code></pre><p>上面的命令在当前目录下生成了<code>coverage.out</code>覆盖率数据文件。</p><p><img src="https://static001.geekbang.org/resource/image/3c/01/3c11a0d41d6ed736f364c1693a2eff01.png?wh=1920x366" alt="图片"></p><p>第二步，分析覆盖率文件：</p><pre><code class="language-bash">$ go tool cover -func=coverage.out
do some setup
PASS
coverage: 40.0% of statements
do some cleanup
ok  	github.com/marmotedu/gopractise-demo/test	0.003s
[colin@dev test]$ go tool cover -func=coverage.out
github.com/marmotedu/gopractise-demo/test/math.go:9:	Abs		100.0%
github.com/marmotedu/gopractise-demo/test/math.go:14:	Max		100.0%
github.com/marmotedu/gopractise-demo/test/math.go:19:	Min		0.0%
github.com/marmotedu/gopractise-demo/test/math.go:24:	RandInt		0.0%
github.com/marmotedu/gopractise-demo/test/math.go:29:	Floor		0.0%
total:							(statements)	40.0%
</code></pre><p>在上述命令的输出中，我们可以查看到哪些函数没有测试，哪些函数内部的分支没有测试完全。cover工具会根据被执行代码的行数与总行数的比例计算出覆盖率。可以看到，Abs和Max函数的测试覆盖率为100%，Min和RandInt的测试覆盖率为0。</p><p>我们还可以使用<code>go tool cover -html</code>生成<code>HTML</code>格式的分析文件，可以更加清晰地展示代码的测试情况：</p><pre><code class="language-bash">$ go tool cover -html=coverage.out -o coverage.html
</code></pre><p>上述命令会在当前目录下生成一个<code>coverage.html</code>文件，用浏览器打开<code>coverage.html</code>文件，可以更加清晰地看到代码的测试情况，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/f0/5e/f089f5d44ba06f052c1c46c858c2b75e.png?wh=1524x1075" alt=""></p><p>通过上图，我们可以知道红色部分的代码没有被测试到，可以让我们接下来有针对性地添加测试用例，而不是一头雾水，不知道需要为哪些代码编写测试用例。</p><p>在Go项目开发中，我们往往会把测试覆盖率作为代码合并的一个强制要求，所以需要在进行代码测试时，同时生成代码覆盖率数据文件。在进行代码测试时，可以通过分析该文件，来判断我们的代码测试覆盖率是否满足要求，如果不满足则代码测试失败。</p><h2>IAM项目测试实战</h2><p>接下来，我来介绍下IAM项目是如何编写和运行测试用例的，你可以通过IAM项目的测试用例，加深对上面内容的理解。</p><h3>IAM项目是如何运行测试用例的？</h3><p>首先，我们来看下IAM项目是如何执行测试用例的。</p><p>在IAM项目的源码根目录下，可以通过运行<code>make test</code>执行测试用例，<code>make test</code>会执行<code>iam/scripts/make-rules/golang.mk</code>文件中的<code>go.test</code>伪目标，规则如下：</p><pre><code class="language-makefile">.PHONY: go.test
go.test: tools.verify.go-junit-report
  @echo "===========&gt; Run unit test"
  @set -o pipefail;$(GO) test -race -cover -coverprofile=$(OUTPUT_DIR)/coverage.out \\
    -timeout=10m -short -v `go list ./...|\
    egrep -v $(subst $(SPACE),'|',$(sort $(EXCLUDE_TESTS)))` 2&gt;&amp;1 | \\
    tee &gt;(go-junit-report --set-exit-code &gt;$(OUTPUT_DIR)/report.xml)
  @sed -i '/mock_.*.go/d' $(OUTPUT_DIR)/coverage.out # remove mock_.*.go files from test coverage
  @$(GO) tool cover -html=$(OUTPUT_DIR)/coverage.out -o $(OUTPUT_DIR)/coverage.html
</code></pre><p>在上述规则中，我们执行<code>go test</code>时设置了超时时间、竞态检查，开启了代码覆盖率检查，覆盖率测试数据保存在了<code>coverage.out</code>文件中。在Go项目开发中，并不是所有的包都需要单元测试，所以上面的命令还过滤掉了一些不需要测试的包，这些包配置在<code>EXCLUDE_TESTS</code>变量中：</p><pre><code class="language-makefile">EXCLUDE_TESTS=github.com/marmotedu/iam/test github.com/marmotedu/iam/pkg/log github.com/marmotedu/iam/third_party github.com/marmotedu/iam/internal/pump/storage github.com/marmotedu/iam/internal/pump github.com/marmotedu/iam/internal/pkg/logger
</code></pre><p>同时，也调用了<code>go-junit-report</code>将go test的结果转化成了xml格式的报告文件，该报告文件会被一些CI系统，例如Jenkins拿来解析并展示结果。上述代码也同时生成了coverage.html文件，该文件可以存放在制品库中，供我们后期分析查看。</p><p>这里需要注意，Mock的代码是不需要编写测试用例的，为了避免影响项目的单元测试覆盖率，需要将Mock代码的单元测试覆盖率数据从<code>coverage.out</code>文件中删除掉，<code>go.test</code>规则通过以下命令删除这些无用的数据：</p><pre><code class="language-bash">sed -i '/mock_.*.go/d' $(OUTPUT_DIR)/coverage.out # remove mock_.*.go files from test coverage
</code></pre><p>另外，还可以通过<code>make cover</code>来进行单元测试覆盖率测试，<code>make cover</code>会执行<code>iam/scripts/make-rules/golang.mk</code>文件中的<code>go.test.cover</code>伪目标，规则如下：</p><pre><code class="language-makefile">.PHONY: go.test.cover
go.test.cover: go.test
  @$(GO) tool cover -func=$(OUTPUT_DIR)/coverage.out | \\
    awk -v target=$(COVERAGE) -f $(ROOT_DIR)/scripts/coverage.awk
</code></pre><p>上述目标依赖<code>go.test</code>，也就是说执行单元测试覆盖率目标之前，会先进行单元测试，然后使用单元测试产生的覆盖率数据<code>coverage.out</code>计算出总的单元测试覆盖率，这里是通过<a href="https://github.com/marmotedu/iam/blob/v1.0.8/scripts/coverage.awk">coverage.awk</a>脚本来计算的。</p><p>如果单元测试覆盖率不达标，Makefile会报错并退出。可以通过Makefile的<a href="https://github.com/marmotedu/iam/blob/master/scripts/make-rules/common.mk#L39-L41">COVERAGE</a>变量来设置单元测试覆盖率阈值。</p><p>COVERAGE的默认值为60，我们也可以在命令行手动指定，例如：</p><pre><code class="language-bash">$ make cover COVERAGE=80
</code></pre><p>为了确保项目的单元测试覆盖率达标，需要设置单元测试覆盖率质量红线。一般来说，这些红线很难靠开发者的自觉性去保障，所以好的方法是将质量红线加入到CICD流程中。</p><p>所以，在<code>Makefile</code>文件中，我将<code>cover</code>放在<code>all</code>目标的依赖中，并且位于build之前，也就是<code>all: gen add-copyright format lint cover build</code>。这样每次当我们执行make时，会自动进行代码测试，并计算单元测试覆盖率，如果覆盖率不达标，则停止构建；如果达标，继续进入下一步的构建流程。</p><h3>IAM项目测试案例分享</h3><p>接下来，我会给你展示一些IAM项目的测试案例，因为这些测试案例的实现方法，我在<a href="https://time.geekbang.org/column/article/408529">36讲</a> 和这一讲的前半部分已有详细介绍，所以这里，我只列出具体的实现代码，不会再介绍这些代码的实现方法。</p><ol>
<li>单元测试案例</li>
</ol><p>我们可以手动编写单元测试代码，也可以使用gotests工具生成单元测试代码。</p><p>先来看手动编写测试代码的案例。这里单元测试代码见<a href="https://github.com/marmotedu/iam/blob/v1.0.8/pkg/log/log_test.go#L52-L62">Test_Option</a>，代码如下：</p><pre><code class="language-go">func Test_Option(t *testing.T) {
    fs := pflag.NewFlagSet("test", pflag.ExitOnError)
    opt := log.NewOptions()
    opt.AddFlags(fs)

    args := []string{"--log.level=debug"}
    err := fs.Parse(args)
    assert.Nil(t, err)

    assert.Equal(t, "debug", opt.Level)
}
</code></pre><p>上述代码中，使用了<code>github.com/stretchr/testify/assert</code>包来对比结果。</p><p>再来看使用gotests工具生成单元测试代码的案例（Table-Driven 的测试模式）。出于效率上的考虑，IAM项目的单元测试用例，基本都是使用gotests工具生成测试用例模板代码，并基于这些模板代码填充测试Case的。代码见<a href="https://github.com/marmotedu/iam/blob/v1.0.8/internal/apiserver/service/v1/service_test.go">service_test.go</a>文件。</p><ol start="2">
<li>性能测试案例</li>
</ol><p>IAM项目的性能测试用例，见<a href="https://github.com/marmotedu/iam/blob/v1.0.8/internal/apiserver/service/v1/user_test.go#L27-L41">BenchmarkListUser</a>测试函数。代码如下：</p><pre><code class="language-go">func BenchmarkListUser(b *testing.B) {
	opts := metav1.ListOptions{
		Offset: pointer.ToInt64(0),
		Limit:  pointer.ToInt64(50),
	}
	storeIns, _ := fake.GetFakeFactoryOr()
	u := &amp;userService{
		store: storeIns,
	}

	for i := 0; i &lt; b.N; i++ {
		_, _ = u.List(context.TODO(), opts)
	}
}
</code></pre><ol start="3">
<li>示例测试案例</li>
</ol><p>IAM项目的示例测试用例见<a href="https://github.com/marmotedu/errors/blob/v1.0.2/example_test.go">example_test.go</a>文件。<code>example_test.go</code>中的一个示例测试代码如下：</p><pre><code class="language-go">func ExampleNew() {
	err := New("whoops")
	fmt.Println(err)

	// Output: whoops
}
</code></pre><ol start="4">
<li>TestMain测试案例</li>
</ol><p>IAM项目的TestMain测试案例，见<a href="https://github.com/marmotedu/iam/blob/v1.0.8/internal/apiserver/service/v1/user_test.go">user_test.go</a>文件中的<code>TestMain</code>函数：</p><pre><code class="language-go">func TestMain(m *testing.M) {
    _, _ = fake.GetFakeFactoryOr()
    os.Exit(m.Run())
}
</code></pre><p><code>TestMain</code>函数初始化了fake Factory，然后调用<code>m.Run</code>执行测试用例。</p><ol start="5">
<li>Mock测试案例</li>
</ol><p>Mock代码见<a href="https://github.com/marmotedu/iam/blob/v1.0.8/internal/apiserver/service/v1/mock_service.go">internal/apiserver/service/v1/mock_service.go</a>，使用Mock的测试用例见<a href="https://github.com/marmotedu/iam/blob/v1.0.8/internal/apiserver/controller/v1/user/create_test.go">internal/apiserver/controller/v1/user/create_test.go</a>文件。因为代码比较多，这里建议你打开链接，查看测试用例的具体实现。</p><p>我们可以在IAM项目的根目录下执行以下命令，来自动生成所有的Mock文件：</p><pre><code class="language-bash">$ go generate ./...
</code></pre><ol start="6">
<li>Fake测试案例</li>
</ol><p>fake store代码实现位于<a href="https://github.com/marmotedu/iam/tree/v1.0.8/internal/apiserver/store/fake">internal/apiserver/store/fake</a>目录下。fake store的使用方式，见<a href="https://github.com/marmotedu/iam/blob/v1.0.8/internal/apiserver/service/v1/user_test.go">user_test.go</a>文件：</p><pre><code class="language-go">func TestMain(m *testing.M) {
    _, _ = fake.GetFakeFactoryOr()
    os.Exit(m.Run())
}

func BenchmarkListUser(b *testing.B) {
    opts := metav1.ListOptions{
        Offset: pointer.ToInt64(0),
        Limit:  pointer.ToInt64(50),
    }
    storeIns, _ := fake.GetFakeFactoryOr()
    u := &amp;userService{
        store: storeIns,
    }

    for i := 0; i &lt; b.N; i++ {
        _, _ = u.List(context.TODO(), opts)
    }
}
</code></pre><p>上述代码通过<code>TestMain</code>初始化fake实例（<a href="https://github.com/marmotedu/iam/blob/v1.0.8/internal/apiserver/store/store.go#L12-L17">store.Factory</a>接口类型）：</p><pre><code class="language-go">func GetFakeFactoryOr() (store.Factory, error) {
    once.Do(func() {
        fakeFactory = &amp;datastore{
            users:    FakeUsers(ResourceCount),
            secrets:  FakeSecrets(ResourceCount),
            policies: FakePolicies(ResourceCount),
        }
    })

    if fakeFactory == nil {
        return nil, fmt.Errorf("failed to get mysql store fatory, mysqlFactory: %+v", fakeFactory)
    }

    return fakeFactory, nil
}
</code></pre><p><code>GetFakeFactoryOr</code>函数，创建了一些fake users、secrets、policies，并保存在了<code>fakeFactory</code>变量中，供后面的测试用例使用，例如BenchmarkListUser、Test_newUsers等。</p><h2>其他测试工具/包</h2><p>最后，我再来分享下Go项目测试中常用的工具/包，因为内容较多，我就不详细介绍了，如果感兴趣你可以点进链接自行学习。我将这些测试工具/包分为了两类，分别是测试框架和Mock工具。</p><h3>测试框架</h3><ul>
<li><a href="https://github.com/stretchr/testify">Testify框架</a>：Testify是Go test的预判工具，它能让你的测试代码变得更优雅和高效，测试结果也变得更详细。</li>
<li><a href="https://github.com/smartystreets/goconvey">GoConvey框架</a>：GoConvey是一款针对Golang的测试框架，可以管理和运行测试用例，同时提供了丰富的断言函数，并支持很多 Web 界面特性。</li>
</ul><h3>Mock工具</h3><p>这一讲里，我介绍了Go官方提供的Mock框架GoMock，不过还有一些其他的优秀Mock工具可供我们使用。这些Mock工具分别用在不同的Mock场景中，我在 <a href="https://time.geekbang.org/column/article/384648">10讲</a>中已经介绍过。不过，为了使我们这一讲的测试知识体系更加完整，这里我还是再提一次，你可以复习一遍。</p><ul>
<li><a href="https://github.com/DATA-DOG/go-sqlmock">sqlmock</a>：可以用来模拟数据库连接。数据库是项目中比较常见的依赖，在遇到数据库依赖时都可以用它。</li>
<li><a href="https://github.com/jarcoal/httpmock">httpmock</a>：可以用来Mock HTTP请求。</li>
<li><a href="https://github.com/bouk/monkey">bouk/monkey</a>：猴子补丁，能够通过替换函数指针的方式来修改任意函数的实现。如果golang/mock、sqlmock和httpmock这几种方法都不能满足我们的需求，我们可以尝试用猴子补丁的方式来Mock依赖。可以这么说，猴子补丁提供了单元测试 Mock 依赖的最终解决方案。</li>
</ul><h2>总结</h2><p>这一讲，我介绍了除单元测试和性能测试之外的另一些测试方法。</p><p>除了示例测试和TestMain函数，我还详细介绍了Mock测试，也就是如何使用GoMock来测试一些在单元测试环境下不好实现的接口。绝大部分情况下，可以使用GoMock来Mock接口，但是对于一些业务逻辑比较复杂的接口，我们可以通过Fake一个接口实现，来对代码进行测试，这也称为Fake测试。</p><p>此外，我还介绍了何时编写和执行测试用例。我们可以根据需要，选择在编写代码前、编写代码中、编写代码后编写测试用例。</p><p>为了保证单元测试覆盖率，我们还应该为整个项目设置单元测试覆盖率质量红线，并将该质量红线加入到CICD流程中。我们可以通过 <code>go test -coverprofile=coverage.out</code> 命令来生成测试覆盖率数据，通过<code>go tool cover -func=coverage.out</code> 命令来分析覆盖率文件。</p><p>IAM项目中使用了大量的测试方法和技巧来测试代码，为了加深你对测试知识的理解，我也列举了一些测试案例，供你参考、学习和验证。具体的测试案例，你可以返回前面查看下。</p><p>除此之外，我们还可以使用其他一些测试框架，例如Testify框架和GoConvey框架。在Go代码测试中，我们最常使用的是Go官方提供的Mock框架GoMock，但仍然有其他优秀的Mock工具，可供我们在不同场景下使用，例如sqlmock、httpmock、bouk/monkey等。</p><h2>课后习题</h2><ol>
<li>请使用 <a href="https://github.com/DATA-DOG/go-sqlmock">sqlmock</a> 来Mock一个GORM数据库实例，并完成GORM的CURD单元测试用例编写。</li>
<li>思考下，在Go项目开发中，还有哪些优秀的测试框架、测试工具、Mock工具以及测试技巧？欢迎你在留言区分享。</li>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/fc/55/e03bb6db.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>i-neojos</span>
  </div>
  <div class="_2_QraFYR_0">这篇文章实用性特别强，对测试平台很有参考意义</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-02 15:35:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/47/bd/5c34df95.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>管清麟</span>
  </div>
  <div class="_2_QraFYR_0">静态检查吧，不是竞态检查</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是  竞态检查</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-10 16:03:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2e/d4/1b/82a32d66.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>是在下输了</span>
  </div>
  <div class="_2_QraFYR_0">在目录下执行go test -coverprofile=coverage.out 没有生成 coverage.out 文件</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是不是当前目录，没有*_test.go文件？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-07 23:38:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/2e/7e/ebc28e10.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>NULL</span>
  </div>
  <div class="_2_QraFYR_0">bouk&#47;monkey许可证好像有问题<br><br>也可以看看这个 https:&#47;&#47;github.com&#47;agiledragon&#47;gomonkey<br><br>agiledragon&#47;gomonkey被bouk&#47;monkey的作者推荐过, 见https:&#47;&#47;esoteric.codes&#47;blog&#47;bouk-monkey-satirical-code-used-by-people-who-dont-get-the-joke</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-06 10:33:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/2e/7e/ebc28e10.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>NULL</span>
  </div>
  <div class="_2_QraFYR_0">go1.18又新增了一种fuzzing测试<br>https:&#47;&#47;go.dev&#47;doc&#47;fuzz&#47;</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢分享</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-05 17:28:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/26/34/891dd45b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>宙斯</span>
  </div>
  <div class="_2_QraFYR_0">fake测试和mock测试有什么区别吗？感觉是一样的东西，没看出侧重点</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: fake测试是模拟的数据，具有业务逻辑，是真实的模拟行为。<br><br>mock只是模拟了输出。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-29 10:42:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/68/d4/c9b5d3f9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>💎A</span>
  </div>
  <div class="_2_QraFYR_0">越到后面人越少</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-17 17:14:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>~\(≧▽≦)&#47;~</span>
  </div>
  <div class="_2_QraFYR_0">老师，调用第三方rest api接口需要用到mock进行测试吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以使用httpmock进行测试。<br><br>但一般建议通过接口进行测试</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-04 18:46:16</div>
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
  <div class="_2_QraFYR_0">老师，请教一下，一般对 rest api 的测试 应该怎么做比较好，<br>我们现在是 按照写单元测试的方式， 测试函数内 调用http请求，判断response。<br>不知道有没有更好的方式。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 差不多就是这么来测试的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-26 15:35:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/b2/91/714c0f07.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>zero</span>
  </div>
  <div class="_2_QraFYR_0">收获满满</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-16 13:00:54</div>
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
  <div class="_2_QraFYR_0">总结：<br>1. 介绍了Example、Mock、Fake 测试规范，以及常见的框架。<br>2. 重点介绍了 gomock package 的使用方法。包括运行 mockgen 的两种方式：source mode &amp; reflect mode；如何对输入和输出参数的约束。<br>3. 对于复杂的 interface，你也可以采用自己实现fake测试。<br>4. iam 提供了 make test 和 make cover 两个命令。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-04 17:12:42</div>
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
  <div class="_2_QraFYR_0">其实可以把自动生成mock和 test 放makefile中管理</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-25 21:44:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/de/97/cda3f551.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>倪昊</span>
  </div>
  <div class="_2_QraFYR_0">老师请问单元测试一般对哪几部分代码做呢？控制层，service层，仓库层都要做吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 都可以做的，但业务层会比较多些</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-15 08:53:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/28/83/17/df99b53d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>随风而过</span>
  </div>
  <div class="_2_QraFYR_0">一般开发中就做了单元测试和mock测试。而且还都是手写，CV复制，没想到还有大量工具来自动生成，包括很多测试框架，gei到了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-08 23:03:01</div>
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
  <div class="_2_QraFYR_0">介绍了示例测试、TestMain 测试、Mock 测试、Fake 测试，加上上一讲中的单元测试和性能测试，共6种测试类型。<br>文中提供了每种测试的示例，可以作为测试方法大全，供后续需要的时候查看。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-19 23:32:54</div>
  </div>
</div>
</div>
</li>
</ul>