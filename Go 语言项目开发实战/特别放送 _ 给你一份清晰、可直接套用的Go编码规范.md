<audio title="特别放送 _ 给你一份清晰、可直接套用的Go编码规范" src="https://static001.geekbang.org/resource/audio/d9/7e/d986bdf482b71296yybbd457934aeb7e.mp3" controls="controls"></audio> 
<p>你好，我是孔令飞。</p><p>我们在上一讲学习了“写出优雅Go项目的方法论”，那一讲内容很丰富，是我多年Go项目开发的经验沉淀，需要你多花一些时间好好消化吸收。吃完大餐之后，咱们今天来一期特别放送，就是上一讲我提到过的编码规范。这一讲里，为了帮你节省时间和精力，我会给你一份清晰、可直接套用的 Go 编码规范，帮助你编写一个高质量的 Go 应用。</p><p>这份规范，是我参考了Go官方提供的编码规范，以及Go社区沉淀的一些比较合理的规范之后，加入自己的理解总结出的，它比很多公司内部的规范更全面，你掌握了，以后在面试大厂的时候，或者在大厂里写代码的时候，都会让人高看你一眼，觉得你code很专业。</p><p>这份编码规范中包含代码风格、命名规范、注释规范、类型、控制结构、函数、GOPATH 设置规范、依赖管理和最佳实践九类规范。如果你觉得这些规范内容太多了，看完一遍也记不住，这完全没关系。你可以多看几遍，也可以在用到时把它翻出来，在实际应用中掌握。这篇特别放送的内容，更多是作为写代码时候的一个参考手册。</p><h2>1. 代码风格</h2><h3>1.1 代码格式</h3><ul>
<li>
<p>代码都必须用 <code>gofmt</code> 进行格式化。</p>
</li>
<li>
<p>运算符和操作数之间要留空格。</p>
</li>
<li>
<p>建议一行代码不超过120个字符，超过部分，请采用合适的换行方式换行。但也有些例外场景，例如import行、工具自动生成的代码、带tag的struct字段。</p>
</li>
<li>
<p>文件长度不能超过800行。</p>
</li>
<li>
<p>函数长度不能超过80行。</p>
</li>
<li>
<p>import规范</p>
<ul>
<li>代码都必须用 goimports进行格式化（建议将代码Go代码编辑器设置为：保存时运行 goimports）。<br>
- 不要使用相对路径引入包，例如 import …/util/net 。<br>
- 包名称与导入路径的最后一个目录名不匹配时，或者多个相同包名冲突时，则必须使用导入别名。</li>
</ul>
</li>
</ul><!-- [[[read_end]]] --><pre><code>// bad
	&quot;github.com/dgrijalva/jwt-go/v4&quot;

	//good
	jwt &quot;github.com/dgrijalva/jwt-go/v4&quot;
</code></pre><ul style="list-style: none;">
   <li>
      <ul>
<li style="margin-left: 23px; padding-left: 17px">导入的包建议进行分组，匿名包的引用使用一个新的分组，并对匿名包引用进行说明。</li>
      </ul>
   </li>
</ul><pre><code>	import (
		// go 标准包
		&quot;fmt&quot;

		// 第三方包
	    &quot;github.com/jinzhu/gorm&quot;
	    &quot;github.com/spf13/cobra&quot;
	    &quot;github.com/spf13/viper&quot;

		// 匿名包单独分组，并对匿名包引用进行说明
	    // import mysql driver
	    _ &quot;github.com/jinzhu/gorm/dialects/mysql&quot;

		// 内部包
	    v1 &quot;github.com/marmotedu/api/apiserver/v1&quot;
	    metav1 &quot;github.com/marmotedu/apimachinery/pkg/meta/v1&quot;
	    &quot;github.com/marmotedu/iam/pkg/cli/genericclioptions&quot;
	)
</code></pre><h3>1.2 声明、初始化和定义</h3><ul>
<li>当函数中需要使用到多个变量时，可以在函数开始处使用var声明。在函数外部声明必须使用 <code>var</code> ，不要采用 <code>:=</code> ，容易踩到变量的作用域的问题。</li>
</ul><pre><code>var (
	Width  int
	Height int
)
</code></pre><ul>
<li>在初始化结构引用时，请使用&amp;T{}代替new(T)，以使其与结构体初始化一致。</li>
</ul><pre><code>// bad
sptr := new(T)
sptr.Name = &quot;bar&quot;

// good
sptr := &amp;T{Name: &quot;bar&quot;}
</code></pre><ul>
<li>struct 声明和初始化格式采用多行，定义如下。</li>
</ul><pre><code>type User struct{
    Username  string
    Email     string
}

user := User{
	Username: &quot;colin&quot;,
	Email: &quot;colin404@foxmail.com&quot;,
}
</code></pre><ul>
<li>相似的声明放在一组，同样适用于常量、变量和类型声明。</li>
</ul><pre><code>// bad
import &quot;a&quot;
import &quot;b&quot;

// good
import (
  &quot;a&quot;
  &quot;b&quot;
)
</code></pre><ul>
<li>尽可能指定容器容量，以便为容器预先分配内存，例如：</li>
</ul><pre><code>v := make(map[int]string, 4)
v := make([]string, 0, 4)
</code></pre><ul>
<li>在顶层，使用标准var关键字。请勿指定类型，除非它与表达式的类型不同。</li>
</ul><pre><code>// bad
var _s string = F()

func F() string { return &quot;A&quot; }

// good
var _s = F()
// 由于 F 已经明确了返回一个字符串类型，因此我们没有必要显式指定_s 的类型
// 还是那种类型

func F() string { return &quot;A&quot; }
</code></pre><ul>
<li>对于未导出的顶层常量和变量，使用_作为前缀。</li>
</ul><pre><code>// bad
const (
  defaultHost = &quot;127.0.0.1&quot;
  defaultPort = 8080
)

// good
const (
  _defaultHost = &quot;127.0.0.1&quot;
  _defaultPort = 8080
)
</code></pre><ul>
<li>嵌入式类型（例如 mutex）应位于结构体内的字段列表的顶部，并且必须有一个空行将嵌入式字段与常规字段分隔开。</li>
</ul><pre><code>// bad
type Client struct {
  version int
  http.Client
}

// good
type Client struct {
  http.Client

  version int
}
</code></pre><h3>1.3 错误处理</h3><ul>
<li><code>error</code>作为函数的值返回，必须对<code>error</code>进行处理，或将返回值赋值给明确忽略。对于<code>defer xx.Close()</code>可以不用显式处理。</li>
</ul><pre><code>func load() error {
	// normal code
}

// bad
load()

// good
 _ = load()
</code></pre><ul>
<li><code>error</code>作为函数的值返回且有多个返回值的时候，<code>error</code>必须是最后一个参数。</li>
</ul><pre><code>// bad
func load() (error, int) {
	// normal code
}

// good
func load() (int, error) {
	// normal code
}
</code></pre><ul>
<li>尽早进行错误处理，并尽早返回，减少嵌套。</li>
</ul><pre><code>// bad
if err != nil {
	// error code
} else {
	// normal code
}

// good
if err != nil {
	// error handling
	return err
}
// normal code
</code></pre><ul>
<li>如果需要在 if 之外使用函数调用的结果，则应采用下面的方式。</li>
</ul><pre><code>// bad
if v, err := foo(); err != nil {
	// error handling
}

// good
v, err := foo()
if err != nil {
	// error handling
}
</code></pre><ul>
<li>错误要单独判断，不与其他逻辑组合判断。</li>
</ul><pre><code>// bad
v, err := foo()
if err != nil || v  == nil {
	// error handling
	return err
}

// good
v, err := foo()
if err != nil {
	// error handling
	return err
}

if v == nil {
	// error handling
	return errors.New(&quot;invalid value v&quot;)
}
</code></pre><ul>
<li>如果返回值需要初始化，则采用下面的方式。</li>
</ul><pre><code>v, err := f()
if err != nil {
    // error handling
    return // or continue.
}
// use v
</code></pre><ul>
<li>
<p>错误描述建议</p>
<ul>
<li>告诉用户他们可以做什么，而不是告诉他们不能做什么。</li>
<li>当声明一个需求时，用must 而不是should。例如，must be greater than 0、must match regex ‘[a-z]+’。</li>
<li>当声明一个格式不对时，用must not。例如，must not contain。</li>
<li>当声明一个动作时用may not。例如，may not be specified when otherField is empty、only name may be specified。</li>
<li>引用文字字符串值时，请在单引号中指示文字。例如，ust not contain ‘…’。</li>
<li>当引用另一个字段名称时，请在反引号中指定该名称。例如，must be greater than <code>request</code>。</li>
<li>指定不等时，请使用单词而不是符号。例如，must be less than 256、must be greater than or equal to 0 (不要用 larger than、bigger than、more than、higher than)。</li>
<li>指定数字范围时，请尽可能使用包含范围。</li>
<li>建议 Go 1.13 以上，error 生成方式为 <code>fmt.Errorf("module xxx: %w", err)</code>。</li>
<li>错误描述用小写字母开头，结尾不要加标点符号，例如：</li>
</ul>
</li>
</ul><pre><code>	// bad
	errors.New(&quot;Redis connection failed&quot;)
	errors.New(&quot;redis connection failed.&quot;)

	// good
	errors.New(&quot;redis connection failed&quot;)
</code></pre><h3>1.4 panic处理</h3><ul>
<li>在业务逻辑处理中禁止使用panic。</li>
<li>在main包中，只有当程序完全不可运行时使用panic，例如无法打开文件、无法连接数据库导致程序无法正常运行。</li>
<li>在main包中，使用 <code>log.Fatal</code> 来记录错误，这样就可以由log来结束程序，或者将panic抛出的异常记录到日志文件中，方便排查问题。</li>
<li>可导出的接口一定不能有panic。</li>
<li>包内建议采用error而不是panic来传递错误。</li>
</ul><h3>1.5 单元测试</h3><ul>
<li>单元测试文件名命名规范为 <code>example_test.go</code>。</li>
<li>每个重要的可导出函数都要编写测试用例。</li>
<li>因为单元测试文件内的函数都是不对外的，所以可导出的结构体、函数等可以不带注释。</li>
<li>如果存在 <code>func (b *Bar) Foo</code> ，单测函数可以为 <code>func TestBar_Foo</code>。</li>
</ul><h3>1.6 类型断言失败处理</h3><p>type assertion 的单个返回值针对不正确的类型将产生 panic。请始终使用 “comma ok”的惯用法。</p><pre><code>// bad
t := n.(int)

// good
t, ok := n.(int)
if !ok {
	// error handling
}
// normal code
</code></pre><h2>2. 命名规范</h2><p>命名规范是代码规范中非常重要的一部分，一个统一的、短小的、精确的命名规范可以大大提高代码的可读性，也可以借此规避一些不必要的Bug。</p><h3>2.1 包命名</h3><ul>
<li>包名必须和目录名一致，尽量采取有意义、简短的包名，不要和标准库冲突。</li>
<li>包名全部小写，没有大写或下划线，使用多级目录来划分层级。</li>
<li>项目名可以通过中划线来连接多个单词。</li>
<li>包名以及包所在的目录名，不要使用复数，例如，是<code>net/url</code>，而不是<code>net/urls</code>。</li>
<li>不要用 common、util、shared 或者 lib 这类宽泛的、无意义的包名。</li>
<li>包名要简单明了，例如 net、time、log。</li>
</ul><h3>2.2 函数命名</h3><ul>
<li>函数名采用驼峰式，首字母根据访问控制决定使用大写或小写，例如：MixedCaps或者mixedCaps。</li>
<li>代码生成工具自动生成的代码(如xxxx.pb.go)和为了对相关测试用例进行分组，而采用的下划线(如TestMyFunction_WhatIsBeingTested)排除此规则。</li>
</ul><h3>2.3 文件命名</h3><ul>
<li>文件名要简短有意义。</li>
<li>文件名应小写，并使用下划线分割单词。</li>
</ul><h3>2.4 结构体命名</h3><ul>
<li>采用驼峰命名方式，首字母根据访问控制决定使用大写或小写，例如MixedCaps或者mixedCaps。</li>
<li>结构体名不应该是动词，应该是名词，比如 Node、NodeSpec。</li>
<li>避免使用Data、Info这类无意义的结构体名。</li>
<li>结构体的声明和初始化应采用多行，例如：</li>
</ul><pre><code>// User 多行声明
type User struct {
    Name  string
    Email string
}

// 多行初始化
u := User{
    UserName: &quot;colin&quot;,
    Email:    &quot;colin404@foxmail.com&quot;,
}
</code></pre><h3>2.5 接口命名</h3><ul>
<li>
<p>接口命名的规则，基本和结构体命名规则保持一致：</p>
<ul>
<li>单个函数的接口名以 “er"”作为后缀（例如Reader，Writer），有时候可能导致蹩脚的英文，但是没关系。</li>
<li>两个函数的接口名以两个函数名命名，例如ReadWriter。</li>
<li>三个以上函数的接口名，类似于结构体名。</li>
</ul>
</li>
</ul><p>例如：</p><pre><code>	// Seeking to an offset before the start of the file is an error.
	// Seeking to any positive offset is legal, but the behavior of subsequent
	// I/O operations on the underlying object is implementation-dependent.
	type Seeker interface {
	    Seek(offset int64, whence int) (int64, error)
	}

	// ReadWriter is the interface that groups the basic Read and Write methods.
	type ReadWriter interface {
	    Reader
	    Writer
	}
</code></pre><h3>2.6 变量命名</h3><ul>
<li>
<p>变量名必须遵循<strong>驼峰式</strong>，首字母根据访问控制决定使用大写或小写。</p>
</li>
<li>
<p>在相对简单（对象数量少、针对性强）的环境中，可以将一些名称由完整单词简写为单个字母，例如：</p>
<ul>
<li>user 可以简写为 u；</li>
<li>userID 可以简写 uid。</li>
</ul>
</li>
<li>
<p>特有名词时，需要遵循以下规则：</p>
<ul>
<li>如果变量为私有，且特有名词为首个单词，则使用小写，如 apiClient。</li>
<li>其他情况都应当使用该名词原有的写法，如 APIClient、repoID、UserID。</li>
</ul>
</li>
</ul><p>下面列举了一些常见的特有名词。</p><pre><code>// A GonicMapper that contains a list of common initialisms taken from golang/lint
var LintGonicMapper = GonicMapper{
    &quot;API&quot;:   true,
    &quot;ASCII&quot;: true,
    &quot;CPU&quot;:   true,
    &quot;CSS&quot;:   true,
    &quot;DNS&quot;:   true,
    &quot;EOF&quot;:   true,
    &quot;GUID&quot;:  true,
    &quot;HTML&quot;:  true,
    &quot;HTTP&quot;:  true,
    &quot;HTTPS&quot;: true,
    &quot;ID&quot;:    true,
    &quot;IP&quot;:    true,
    &quot;JSON&quot;:  true,
    &quot;LHS&quot;:   true,
    &quot;QPS&quot;:   true,
    &quot;RAM&quot;:   true,
    &quot;RHS&quot;:   true,
    &quot;RPC&quot;:   true,
    &quot;SLA&quot;:   true,
    &quot;SMTP&quot;:  true,
    &quot;SSH&quot;:   true,
    &quot;TLS&quot;:   true,
    &quot;TTL&quot;:   true,
    &quot;UI&quot;:    true,
    &quot;UID&quot;:   true,
    &quot;UUID&quot;:  true,
    &quot;URI&quot;:   true,
    &quot;URL&quot;:   true,
    &quot;UTF8&quot;:  true,
    &quot;VM&quot;:    true,
    &quot;XML&quot;:   true,
    &quot;XSRF&quot;:  true,
    &quot;XSS&quot;:   true,
}
</code></pre><ul>
<li>若变量类型为bool类型，则名称应以Has，Is，Can或Allow开头，例如：</li>
</ul><pre><code>var hasConflict bool
var isExist bool
var canManage bool
var allowGitHook bool
</code></pre><ul>
<li>局部变量应当尽可能短小，比如使用buf指代buffer，使用i指代index。</li>
<li>代码生成工具自动生成的代码可排除此规则(如xxx.pb.go里面的Id)</li>
</ul><h3>2.7 常量命名</h3><ul>
<li>常量名必须遵循驼峰式，首字母根据访问控制决定使用大写或小写。</li>
<li>如果是枚举类型的常量，需要先创建相应类型：</li>
</ul><pre><code>// Code defines an error code type.
type Code int

// Internal errors.
const (
    // ErrUnknown - 0: An unknown error occurred.
    ErrUnknown Code = iota
    // ErrFatal - 1: An fatal error occurred.
    ErrFatal
)
</code></pre><h3>2.8 Error的命名</h3><ul>
<li>Error类型应该写成FooError的形式。</li>
</ul><pre><code>type ExitError struct {
	// ....
}
</code></pre><ul>
<li>Error变量写成ErrFoo的形式。</li>
</ul><pre><code>var ErrFormat = errors.New(&quot;unknown format&quot;)
</code></pre><h2>3. 注释规范</h2><ul>
<li>每个可导出的名字都要有注释，该注释对导出的变量、函数、结构体、接口等进行简要介绍。</li>
<li>全部使用单行注释，禁止使用多行注释。</li>
<li>和代码的规范一样，单行注释不要过长，禁止超过 120 字符，超过的请使用换行展示，尽量保持格式优雅。</li>
<li>注释必须是完整的句子，以需要注释的内容作为开头，句点作为结尾，格式为 <code>// 名称 描述.</code> 。例如：</li>
</ul><pre><code>// bad
// logs the flags in the flagset.
func PrintFlags(flags *pflag.FlagSet) {
	// normal code
}

// good
// PrintFlags logs the flags in the flagset.
func PrintFlags(flags *pflag.FlagSet) {
	// normal code
}
</code></pre><ul>
<li>
<p>所有注释掉的代码在提交code review前都应该被删除，否则应该说明为什么不删除，并给出后续处理建议。</p>
</li>
<li>
<p>在多段注释之间可以使用空行分隔加以区分，如下所示：</p>
</li>
</ul><pre><code>// Package superman implements methods for saving the world.
//
// Experience has shown that a small number of procedures can prove
// helpful when attempting to save the world.
package superman
</code></pre><h3>3.1 包注释</h3><ul>
<li>每个包都有且仅有一个包级别的注释。</li>
<li>包注释统一用 <code>//</code> 进行注释，格式为 <code>// Package 包名 包描述</code> ，例如：</li>
</ul><pre><code>// Package genericclioptions contains flags which can be added to you command, bound, completed, and produce
// useful helper functions.
package genericclioptions
</code></pre><h3>3.2 变量/常量注释</h3><ul>
<li>每个可导出的变量/常量都必须有注释说明，格式为<code>// 变量名 变量描述</code>，例如：</li>
</ul><pre><code>// ErrSigningMethod defines invalid signing method error.
var ErrSigningMethod = errors.New(&quot;Invalid signing method&quot;)
</code></pre><ul>
<li>出现大块常量或变量定义时，可在前面注释一个总的说明，然后在每一行常量的前一行或末尾详细注释该常量的定义，例如：</li>
</ul><pre><code>// Code must start with 1xxxxx.    
const (                         
    // ErrSuccess - 200: OK.          
    ErrSuccess int = iota + 100001    
                                                   
    // ErrUnknown - 500: Internal server error.    
    ErrUnknown    

    // ErrBind - 400: Error occurred while binding the request body to the struct.    
    ErrBind    
                                                  
    // ErrValidation - 400: Validation failed.    
    ErrValidation 
)
</code></pre><h3>3.3 结构体注释</h3><ul>
<li>每个需要导出的结构体或者接口都必须有注释说明，格式为 <code>// 结构体名 结构体描述.</code>。</li>
<li>结构体内的可导出成员变量名，如果意义不明确，必须要给出注释，放在成员变量的前一行或同一行的末尾。例如：</li>
</ul><pre><code>// User represents a user restful resource. It is also used as gorm model.
type User struct {
    // Standard object's metadata.
    metav1.ObjectMeta `json:&quot;metadata,omitempty&quot;`

    Nickname string `json:&quot;nickname&quot; gorm:&quot;column:nickname&quot;`
    Password string `json:&quot;password&quot; gorm:&quot;column:password&quot;`
    Email    string `json:&quot;email&quot; gorm:&quot;column:email&quot;`
    Phone    string `json:&quot;phone&quot; gorm:&quot;column:phone&quot;`
    IsAdmin  int    `json:&quot;isAdmin,omitempty&quot; gorm:&quot;column:isAdmin&quot;`
}
</code></pre><h3>3.4 方法注释</h3><ul>
<li>每个需要导出的函数或者方法都必须有注释，格式为<code>// 函数名 函数描述.</code>，例如：</li>
</ul><pre><code>// BeforeUpdate run before update database record.
func (p *Policy) BeforeUpdate() (err error) {
	// normal code
	return nil
}
</code></pre><h3>3.5 类型注释</h3><ul>
<li>每个需要导出的类型定义和类型别名都必须有注释说明，格式为 <code>// 类型名 类型描述.</code> ，例如：</li>
</ul><pre><code>// Code defines an error code type.
type Code int
</code></pre><h2>4. 类型</h2><h3>4.1 字符串</h3><ul>
<li>空字符串判断。</li>
</ul><pre><code>// bad
if s == &quot;&quot; {
    // normal code
}

// good
if len(s) == 0 {
    // normal code
}
</code></pre><ul>
<li>[]byte/string相等比较。</li>
</ul><pre><code>// bad
var s1 []byte
var s2 []byte
...
bytes.Equal(s1, s2) == 0
bytes.Equal(s1, s2) != 0

// good
var s1 []byte
var s2 []byte
...
bytes.Compare(s1, s2) == 0
bytes.Compare(s1, s2) != 0
</code></pre><ul>
<li>复杂字符串使用raw字符串避免字符转义。</li>
</ul><pre><code>// bad
regexp.MustCompile(&quot;\\.&quot;)

// good
regexp.MustCompile(`\.`)
</code></pre><h3>4.2 切片</h3><ul>
<li>空slice判断。</li>
</ul><pre><code>// bad
if len(slice) = 0 {
    // normal code
}

// good
if slice != nil &amp;&amp; len(slice) == 0 {
    // normal code
}
</code></pre><p>上面判断同样适用于map、channel。</p><ul>
<li>声明slice。</li>
</ul><pre><code>// bad
s := []string{}
s := make([]string, 0)

// good
var s []string
</code></pre><ul>
<li>slice复制。</li>
</ul><pre><code>// bad
var b1, b2 []byte
for i, v := range b1 {
   b2[i] = v
}
for i := range b1 {
   b2[i] = b1[i]
}

// good
copy(b2, b1)
</code></pre><ul>
<li>slice新增。</li>
</ul><pre><code>// bad
var a, b []int
for _, v := range a {
    b = append(b, v)
}

// good
var a, b []int
b = append(b, a...)
</code></pre><h3>4.3 结构体</h3><ul>
<li>struct初始化。</li>
</ul><p>struct以多行格式初始化。</p><pre><code>type user struct {
	Id   int64
	Name string
}

u1 := user{100, &quot;Colin&quot;}

u2 := user{
    Id:   200,
    Name: &quot;Lex&quot;,
}
</code></pre><h2>5. 控制结构</h2><h3>5.1 if</h3><ul>
<li>if 接受初始化语句，约定如下方式建立局部变量。</li>
</ul><pre><code>if err := loadConfig(); err != nil {
	// error handling
	return err
}
</code></pre><ul>
<li>if 对于bool类型的变量，应直接进行真假判断。</li>
</ul><pre><code>var isAllow bool
if isAllow {
	// normal code
}
</code></pre><h3>5.2 for</h3><ul>
<li>采用短声明建立局部变量。</li>
</ul><pre><code>sum := 0
for i := 0; i &lt; 10; i++ {
    sum += 1
}
</code></pre><ul>
<li>不要在 for 循环里面使用 defer，defer只有在函数退出时才会执行。</li>
</ul><pre><code>// bad
for file := range files {
	fd, err := os.Open(file)
	if err != nil {
		return err
	}
	defer fd.Close()
	// normal code
}

// good
for file := range files {
	func() {
		fd, err := os.Open(file)
		if err != nil {
			return err
		}
		defer fd.Close()
		// normal code
	}()
}
</code></pre><h3>5.3 range</h3><ul>
<li>如果只需要第一项（key），就丢弃第二个。</li>
</ul><pre><code>for key := range keys {
// normal code
}
</code></pre><ul>
<li>如果只需要第二项，则把第一项置为下划线。</li>
</ul><pre><code>sum := 0
for _, value := range array {
    sum += value
}
</code></pre><h3>5.4 switch</h3><ul>
<li>必须要有default。</li>
</ul><pre><code>switch os := runtime.GOOS; os {
    case &quot;linux&quot;:
        fmt.Println(&quot;Linux.&quot;)
    case &quot;darwin&quot;:
        fmt.Println(&quot;OS X.&quot;)
    default:
        fmt.Printf(&quot;%s.\n&quot;, os)
}
</code></pre><h3>5.5 goto</h3><ul>
<li>业务代码禁止使用 <code>goto</code> 。</li>
<li>框架或其他底层源码尽量不用。</li>
</ul><h2>6. 函数</h2><ul>
<li>
<p>传入变量和返回变量以小写字母开头。</p>
</li>
<li>
<p>函数参数个数不能超过5个。</p>
</li>
<li>
<p>函数分组与顺序</p>
<ul>
<li>函数应按粗略的调用顺序排序。</li>
<li>同一文件中的函数应按接收者分组。</li>
</ul>
</li>
<li>
<p>尽量采用值传递，而非指针传递。</p>
</li>
<li>
<p>传入参数是 map、slice、chan、interface ，不要传递指针。</p>
</li>
</ul><h3>6.1 函数参数</h3><ul>
<li>如果函数返回相同类型的两个或三个参数，或者如果从上下文中不清楚结果的含义，使用命名返回，其他情况不建议使用命名返回，例如：</li>
</ul><pre><code>func coordinate() (x, y float64, err error) {
	// normal code
}
</code></pre><ul>
<li>传入变量和返回变量都以小写字母开头。</li>
<li>尽量用值传递，非指针传递。</li>
<li>参数数量均不能超过5个。</li>
<li>多返回值最多返回三个，超过三个请使用 struct。</li>
</ul><h3>6.2 defer</h3><ul>
<li>当存在资源创建时，应紧跟defer释放资源(可以大胆使用defer，defer在Go1.14版本中，性能大幅提升，defer的性能损耗即使在性能敏感型的业务中，也可以忽略)。</li>
<li>先判断是否错误，再defer释放资源，例如：</li>
</ul><pre><code>rep, err := http.Get(url)
if err != nil {
    return err
}

defer resp.Body.Close()
</code></pre><h3>6.3 方法的接收器</h3><ul>
<li>推荐以类名第一个英文首字母的小写作为接收器的命名。</li>
<li>接收器的命名在函数超过20行的时候不要用单字符。</li>
<li>接收器的命名不能采用me、this、self这类易混淆名称。</li>
</ul><h3>6.4 嵌套</h3><ul>
<li>嵌套深度不能超过4层。</li>
</ul><h3>6.5 变量命名</h3><ul>
<li>变量声明尽量放在变量第一次使用的前面，遵循就近原则。</li>
<li>如果魔法数字出现超过两次，则禁止使用，改用一个常量代替，例如：</li>
</ul><pre><code>// PI ...
const Prise = 3.14

func getAppleCost(n float64) float64 {
	return Prise * n
}

func getOrangeCost(n float64) float64 {
	return Prise * n
}
</code></pre><h2>7.   GOPATH 设置规范</h2><ul>
<li>Go 1.11 之后，弱化了 GOPATH 规则，已有代码（很多库肯定是在1.11之前建立的）肯定符合这个规则，建议保留 GOPATH 规则，便于维护代码。</li>
<li>建议只使用一个 GOPATH，不建议使用多个 GOPATH。如果使用多个GOPATH，编译生效的 bin 目录是在第一个 GOPATH 下。</li>
</ul><h2>8. 依赖管理</h2><ul>
<li>Go 1.11 以上必须使用 Go Modules。</li>
<li>使用Go Modules作为依赖管理的项目时，不建议提交vendor目录。</li>
<li>使用Go Modules作为依赖管理的项目时，必须提交go.sum文件。</li>
</ul><h2>9. 最佳实践</h2><ul>
<li>尽量少用全局变量，而是通过参数传递，使每个函数都是“无状态”的。这样可以减少耦合，也方便分工和单元测试。</li>
<li>在编译时验证接口的符合性，例如：</li>
</ul><pre><code>type LogHandler struct {
  h   http.Handler
  log *zap.Logger
}
var _ http.Handler = LogHandler{}
</code></pre><ul>
<li>服务器处理请求时，应该创建一个context，保存该请求的相关信息（如requestID），并在函数调用链中传递。</li>
</ul><h3>9.1 性能</h3><ul>
<li>string 表示的是不可变的字符串变量，对 string 的修改是比较重的操作，基本上都需要重新申请内存。所以，如果没有特殊需要，需要修改时多使用 []byte。</li>
<li>优先使用 strconv 而不是 fmt。</li>
</ul><h3>9.2 注意事项</h3><ul>
<li>append 要小心自动分配内存，append 返回的可能是新分配的地址。</li>
<li>如果要直接修改 map 的 value 值，则 value 只能是指针，否则要覆盖原来的值。</li>
<li>map 在并发中需要加锁。</li>
<li>编译过程无法检查 interface{} 的转换，只能在运行时检查，小心引起 panic。</li>
</ul><h2>总结</h2><p>这一讲，我向你介绍了九类常用的编码规范。但今天的最后，我要在这里提醒你一句：规范是人定的，你也可以根据需要，制定符合你项目的规范。这也是我在之前的课程里一直强调的思路。但同时我也建议你采纳这些业界沉淀下来的规范，并通过工具来确保规范的执行。</p><p>今天的内容就到这里啦，欢迎你在下面的留言区谈谈自己的看法，我们下一讲见。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/a9/66/2df6b3fe.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>叫我去学习好么</span>
  </div>
  <div class="_2_QraFYR_0">为什么函数参数尽量不用指针传递？如果是一个比较大的结构体，传指针不是更好吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果传指针存在被意外修改的风险，如果结构体很大，也可以传指针</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-17 15:09:59</div>
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
  <div class="_2_QraFYR_0">不认同的地方:<br>4.1字符串, 空字符串的判断<br>两种方式都可以<br>详见: https:&#47;&#47;stackoverflow.com&#47;questions&#47;18594330&#47;what-is-the-best-way-to-test-for-an-empty-string-in-go<br><br>4.2切片, 空 slice 判断<br>应该是&quot;如果你需要测试一个slice是否是空的，使用len(s) == 0来判断，而不应该用s == nil来判断&quot;<br>详见Go语言圣经: https:&#47;&#47;books.studygolang.com&#47;gopl-zh&#47;ch4&#47;ch4-02.html<br><br>描述不够准确定地方:<br>4.2切片, 声明 slice<br>描述不够准确<br>详见: https:&#47;&#47;github.com&#47;golang&#47;go&#47;wiki&#47;CodeReviewComments#declaring-empty-slices<br>补充: 可以根据需要指定cap, 减少内存分配次数, 降低gc压力, 如 make([]int, 0, 100)<br><br>6.函数, 尽量采用值传递，而非指针传递。<br>描述不够准确<br>详见: https:&#47;&#47;github.com&#47;golang&#47;go&#47;wiki&#47;CodeReviewComments#pass-values<br><br>6.1 函数参数, 尽量用值传递，非指针传递。<br>描述不够准确<br>详见: https:&#47;&#47;github.com&#47;golang&#47;go&#47;wiki&#47;CodeReviewComments#pass-values<br><br>6.函数, 传入参数是 map、slice、chan、interface ，不要传递指针。<br>描述不够准确, 如果是slice, 并且有append操作, 并且期望改变可以影响原函数, 应当传递指针<br>这与slice的底层结构有关, 两个value, 一个 pointer<br>这在&quot;9.2 注意事项&quot;第一条也有说明<br>详见: https:&#47;&#47;github.com&#47;golang&#47;go&#47;blob&#47;master&#47;src&#47;runtime&#47;slice.go#L22<br><br>9. 最佳实践, 在编译时验证接口的符合性，例如：<br>描述不够准确<br>详见: https:&#47;&#47;github.com&#47;xxjwxc&#47;uber_go_guide_cn#toc8</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 6666，我再check下</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-30 18:45:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>苳冬</span>
  </div>
  <div class="_2_QraFYR_0">驼峰命名 APIClient、UserID，如果遇到API+ID就是APIID这样连续两个全大写可读性很低啊</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 那是不是可以把API换成更具体的API名字呢</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-24 09:49:54</div>
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
  <div class="_2_QraFYR_0">在1.2初始化结构体引用时，给的案例里面bad 和good的sval的方式是一模一样的啊？还有在切片初始化时不都建议提前制定容量，然后后面为什么在声明slice是时候，又不建议make的方式而用var 的方式？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果不指定切片的cap，建议用var s []string</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-17 10:55:10</div>
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
  <div class="_2_QraFYR_0">终极规范：当遇到规范问题不知道如何处理时，立马查看文档！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-17 08:24:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/28/1d/69/c21d2644.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>josephzxy</span>
  </div>
  <div class="_2_QraFYR_0">想问1.2节里为什么“对于未导出的顶层常量和变量，使用 _ 作为前缀。”，有什么作用吗？首字母小写不是已经表明是未导出(unexported)了吗？谢谢！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 比如：_version，看到这种命令方式，你就知道：这是个全局的未导出的变量。可以协助你使用变量，也就是通过变量名，你能知道该变量的作用域以及是否可导出。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-05 19:47:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/4b/11/d7e08b5b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>dll</span>
  </div>
  <div class="_2_QraFYR_0">注释写中文才能说清楚的情况，该怎么规范注释呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 统一规范即可，要么全写中文，要么全写英文</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-30 17:28:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4b/24/8c9eaf7f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Seven</span>
  </div>
  <div class="_2_QraFYR_0">为方便团队都使用这份规范，需要写到开发文档里。请问这篇规范可以给个GitHub链接吗？这样方便更多人用，当然注明来自本课程也可以让更多人慕名学习。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 目前没有Github链接。我们考虑下，感谢建议。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-26 10:58:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/5GQZCecntmOibVjWkMWVnibqXEZhAYnFiaRkgfAUGdrQBWzfXjqsYteLee6afDEjvBLBVa5uvtWYTTicwO2jKia0zOw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_a4cca6</span>
  </div>
  <div class="_2_QraFYR_0">老师，请问有学习班的群吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 有的，看下iam项目&#47;README，有我微信号，加我，我拉你</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-24 13:52:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/25/37/1e/9b6554b6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Q</span>
  </div>
  <div class="_2_QraFYR_0">空slice那里为啥要 先判断slice != nil 再判断 len(slice) &gt; 0 呢？不判断不可以吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 因为nil的silce，len(slice)也是0。<br><br>空slice指的是：slice=make([]int,0)这种情况，也即是：slice不为nil，但长度为0的slice。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-19 17:44:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/22/9a/52/93416b65.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>不明真相的群众</span>
  </div>
  <div class="_2_QraFYR_0">作为一个新手，只能在实际应用的时候 再看翻看这章节</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-17 16:01:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_b797c1</span>
  </div>
  <div class="_2_QraFYR_0">催更：更新太慢了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-17 14:26:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/28/68/774b1619.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Fan</span>
  </div>
  <div class="_2_QraFYR_0">请问go web 的项目结构目录有标准吗？ 查资料查到一个社区标准（https:&#47;&#47;github.com&#47;golang-standards&#47;project-layout），但是褒贬不一。所以想问问您那的结构目录是怎么标准化的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 基本上与这个项目保持一致：https:&#47;&#47;github.com&#47;golang-standards&#47;project-layout<br><br>可以加我微信，我们详细交流下。项目根目录下的README文件中，有我微信号</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-27 10:21:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/5a/56/115c6433.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jssfy</span>
  </div>
  <div class="_2_QraFYR_0">不要使用相对路径引入包，例如 import ..&#47;util&#47;net 。<br>请问这里是不是指不要用.或者..这种符号？引用内部子目录下的其他package应该还是允许的吧？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 都不可以</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-21 01:01:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/22/9a/52/93416b65.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>不明真相的群众</span>
  </div>
  <div class="_2_QraFYR_0">日常催更</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-18 09:59:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/1d/c0/978fc470.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>dch666</span>
  </div>
  <div class="_2_QraFYR_0">判断空字符串用 len(s) == 0 比 s == &quot;&quot; 好在哪呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 用 s == &quot;&quot;</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-17 14:01:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/28/66/02/ff4638dc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>雪洛</span>
  </div>
  <div class="_2_QraFYR_0">站在巨人的肩膀，get ！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-17 09:42:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/87/2c/013afd74.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>夏夜星语</span>
  </div>
  <div class="_2_QraFYR_0">请问我们现在的iam 项目代码已经完全写完了嘛，以后就都是这种理论课嘛？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 也不算是理论课，会介绍iam背后的设计思路、开发经验、知识等</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-17 09:33:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/a7/08/a7e03564.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>冰糕🍦</span>
  </div>
  <div class="_2_QraFYR_0">if slice != nil &amp;&amp; len(slice) == 0 , 这个判断是什么原因呢，有必要多判断一下slice是否为nil吗？ </div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-17 14:59:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/e2/c7/3e1d396e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>oneWalker</span>
  </div>
  <div class="_2_QraFYR_0">这个规范可以给一份word版本的吗？或者推荐一下来源，我好做成flashcard培养编码规范习惯。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: https:&#47;&#47;github.com&#47;marmotedu&#47;geekbang-go&#47;blob&#47;master&#47;%E5%8F%AF%E7%9B%B4%E6%8E%A5%E5%A5%97%E7%94%A8%E7%9A%84Go%E7%BC%96%E7%A0%81%E8%A7%84%E8%8C%83.md</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-18 23:06:29</div>
  </div>
</div>
</div>
</li>
</ul>