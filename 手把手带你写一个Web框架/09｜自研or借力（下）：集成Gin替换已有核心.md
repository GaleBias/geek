<audio title="09｜自研or借力（下）：集成Gin替换已有核心" src="https://static001.geekbang.org/resource/audio/b4/88/b430cf56422893836985f9f2d01bff88.mp3" controls="controls"></audio> 
<p>你好，我是轩脉刃。</p><p>在上一节课，比对了 Gin 框架和我们之前写的框架，明确框架设计的目标是，要设计出真正具有实用性的一个工业级框架，所以我们接下来会基于现有的比较成熟的 Gin 框架，并且好好利用其中间件的生态，站在巨人的肩膀，继续搭建自己的框架。</p><h2>如何借力，讨论开源项目的许可协议</h2><p>有的人可能就有点困惑了，这样借鉴其他框架或者其他库不是侵权行为吗？</p><p>要解答这个问题，我们得先搞清楚站在巨人肩膀是要做什么操作。借鉴和使用 Gin 框架作为我们的底层框架基本思路是，以复制的形式将 Gin 框架引入到我们的自研框架中，替换和修改之前实现的几个核心模块。</p><p>我们后续会在这个以 Gin 为核心的新框架上，进行其余核心或者非核心框架模块的设计和开发，同时我们也需要找到比较好的方式，能将 Gin 生态中丰富的开源中间件进行复制和使用。</p><p>现在我们再来回答是否侵权的问题，首先得了解开源许可证，并且知道可以对 Gin 框架做些什么操作？</p><p>开源社区有非常多的开源项目，每个项目都需要有许可说明，包含：是否可以引用、是否允许修改、是否允许商用等。目前的开源许可证有非常多种，每个许可证都是一份使用这个开源项目需要遵守的协议，而主流的开源许可证在 OSI 组织（开放源代码促进会）都有 <a href="https://opensource.org/licenses">登记</a> 。<strong>最主流的开源许可证有 6 种：Apache、BSD、GPL、LGPL、MIT、Mozillia</strong>。</p><!-- [[[read_end]]] --><p>BSD 许可证、MIT 许可证和 Apache 许可证属于三个比较宽松的许可，都允许对源代码进行修改，且可以在闭源软件中使用，区别在于对新的修改，是否必须使用原先的许可证格式，以及修改后的软件是否能以原软件的名义进行销售等。</p><p>我们这里重点讲一下 Gin 框架使用的 <a href="https://github.com/gin-gonic/gin/blob/master/LICENSE">MIT开源许可证</a> ，这个许可证内容非常简单，对使用者的要求也最低。</p><ul>
<li>允许被许可人使用、复制、修改、合并、出版发行、散布、再许可、售卖软件及软件副本。</li>
<li>唯一条件是在软件和软件副本中必须包含著作权声明和本许可声明。</li>
</ul><p>所以如果软件用的是MIT许可证，不管你开发的项目是开源的，还是商业闭源的，都可以放心使用这个软件，<strong>只需要在软件中包含著作权声明和许可协议声明就行，且不要求新的文件必须使用 MIT 协议</strong>。</p><p>以使用了 MIT 协议的 Gin 框架为例，你可以在新项目中以引用或者复制的形式，使用 Gin 框架，也允许你在复制的 Gin 框架中进行修改，修改可以不用注明 Gin 的版权，但是非修改部分是需要包含版权声明的。</p><p>所以，如果我们使用复制形式使用 Gin 框架的话，需要在 Gin 框架每个文件的头部，增加上著作权和许可声明：</p><pre><code class="language-go">// Copyright 2014 Manu Martinez-Almeida.&nbsp; All rights reserved.
// Use of this source code is governed by a MIT style
// license that can be found in the LICENSE file.
</code></pre><h2>如何将 Gin 迁移进入我们的框架</h2><p>明确了 Gin 是允许被复制、修改和合并进入我们的框架，我们就可以开始规划迁移的具体方案了。</p><p>首先确定 Gin 的迁移版本，截止到目前（2021 年 8 月 14 日）最新的 Gin 版本为 1.7.3，所以我们将 Gin 的版本确定为 <a href="https://github.com/gin-gonic/gin/releases/tag/v1.7.3">1.7.3</a> 。</p><p>在 Golang 中，要在一个项目中引入另外一个项目，一般有两种做法，一种做法是<strong>把要引用的项目通过 go mod 引入到目标库中</strong>，而另外一种做法则费劲的多，<strong>使用复制源码的方式引入要引用的项目</strong>。</p><p>go mod 是 Go 官方提供的第三方库管理工具。go mod 的使用方式是在代码方面，也就是在你要使用第三方库功能的时候，使用 import 关键字进行引用。它的好处是简单方便，我们直接使用官方提供的 go get 方法，就能将某个框架进行使用和升级。</p><p>但是如果希望对第三方库进行源码级别的修改，go mod 的方式就会受到限制了。比如我们使用了 Gin 项目，想扩展项目中的 Context 这个结构，为其增加一个函数方法 A，这个需求用 go mod 的方式是做不到的。</p><p>go mod 的方式提倡的是“使用第三方库”而不是“定制第三方库”。所以<strong>对于很强的定制第三方库的需求，我们只能选择复制源码的方式</strong>。</p><p>有的同学可能会觉得这种源码复制的方式有点奇怪，但是这种复制出来，再进行定制化的方式，其实在一些项目中是常常可以见到的。</p><p>比如 Gin 框架在使用 httprouter 的时候，由于要对其进行深度定制化和优化，所以直接将 httprouter 的代码以复制的方式引入到自己的项目中；上一讲简单提过，比较成功的开源项目 <a href="https://github.com/go-macaron/macaron">macaron</a> ，在项目的最初期也是参考另外一个项目 <a href="https://github.com/go-martini/martini">martini</a> ，将 martini 中的一些源码拷贝后进行优化。我们还能在这两个项目的某些代码文件中看到对第三方库的版权申明。</p><p>由于我们对Gin框架有深度定制改造的需求，所以接下来也采用源码复制的方式引入Gin框架。首先将Gin1.7.3的源码完整复制进入项目，存放在framework目录中创建一个gin目录。<br>
<img src="https://static001.geekbang.org/resource/image/e0/3a/e0ae3bba9e6775d407893969eb26253a.png?wh=1084x962" alt=""></p><p>复制之后需要做两步操作：</p><ul>
<li>将Gin目录下的go.mod的内容放在项目目录下</li>
</ul><p>既然我们以复制源码的方式引入了Gin，那么Gin的地址就成为我们项目中的一部分，它就不是一个有go.mod的第三方库了，而Gin原本引入的第三方库，成为了我们项目所需要的第三方库。所以第一步需要将Gin目录下的go.mod和go.sum删除，并且将go.mod的内容复制到项目的根目录中。</p><ul>
<li>将Gin中原有Gin库的引用地址，统一替换为当前项目的地址</li>
</ul><p>go.mod中的module代表了当前项目的模块名称，而在项目的go.mod中，我们将我们这个框架的模块名称从“coredemo”修改为“github.com/gohade/hade”。在项目的go.mod文件：</p><pre><code class="language-go">module github.com/gohade/hade

go 1.15

require (
   ...
)

</code></pre><p>其实这个module并不一定是GitHub上的一个项目地址，也可以自己定义的，我们之前用的coredemo就是一个自定义的字符串。但是开源项目的普遍做法是，将模块名定义为你的开源地址名称，这样能让开源项目使用者在导入包的时候，直接根据模块名查找到对应文件。</p><p>所有第三方引用在查询时，go.mod中的当前模块名称为github.com/gohade/hade，那么复制过来之后，对应的Gin框架的引用地址就是github.com/gohade/hade/framework/gin，而Gin框架之前go.mod中的模块名称为github.com/gin-gonic/gin。</p><p>所以我们这里<strong>要做一次统一替换</strong>，将之前Gin框架中的所有引用github.com/gin-gonic/gin的地方替换为github.com/gohade/hade/framework/gin。</p><p>比如：</p><pre><code class="language-plain">"github.com/gin-gonic/gin/binding"  替换为 "github.com/gohade/hade/framework/gin/binding"
"github.com/gin-gonic/gin/render"  替换为 "github.com/gohade/hade/framework/gin/render"
</code></pre><p>这里就要借用IDE的强大替换工具了，进行一次批量替换。</p><p>做完上述两步的操作之后，我们的项目github.com/gohade/hade就包含了Gin 1.7.3了。下面就是重头戏了，需要思考如何将之前研发的定制化功能迁移到Gin上。</p><h2>如何迁移</h2><p>首先，梳理下目前已经实现的模块：</p><ul>
<li>Context：请求控制器，控制每个请求的超时等逻辑；</li>
<li>路由：让请求更快寻找目标函数，并且支持通配符、分组等方式制定路由规则；</li>
<li>中间件：能将通用逻辑转化为中间件，并串联中间件实现业务逻辑；</li>
<li>封装：提供易用的逻辑，把 request 和 response 封装在 Context 结构中；</li>
<li>重启：实现优雅关闭机制，让服务可以重启。</li>
</ul><p>在 Gin 的框架中，Context、路由、中间件，都已经有了 Gin 自己的实现，而且我们从源码上分析了细节。</p><p>Context方面，Gin的实现基本和我们之前的实现是一致的。之前实现的Core数据结构对应Gin中的Engine，Group数据结构对应Gin的Group结构，Context数据结构对应Gin的Context数据结构。</p><p>路由方面，Gin的路由实现得比我们要好，这一点上一节课详细分析了Gin路由就不再赘述。</p><p>中间件方面，Gin的中间件实现和我们之前的没有什么太大的差别，只有一点，我们定义的Handler为：</p><pre><code class="language-plain">type ControllerHandler func(c *Context) error
</code></pre><p>而Gin定义的Handler为：</p><pre><code class="language-plain">type HandlerFunc func(*Context)
</code></pre><p>可以看到相比Gin，我们多定义了一个error返回值。因为Gin的作者认为，中断一个请求返回一个error并没有什么用，<strong>他希望中断一个请求的时候直接操作Response</strong>，比如设置返回状态码、设置返回错误信息，而不希望用error来进行返回，所以框架也不会用这个error来操作任何的返回信息。这一点我认为Gin框架的考量是有道理的，所以我们也沿用这种方式。<br>
而对于 Request 和 Response 的封装， Gin 的实现比较简陋。Gin对Request并没有以接口的方式，将Request支持哪些接口展示出来；并且在参数查询的时候，返回类型并不多，比如从Form中获取参数的系列方法，Gin只实现了几个方法：</p><pre><code class="language-go">PostForm
DefaultPostForm
GetPostForm
PostFormArray
GetPostFormArray
PostFormMap
GetPostFormMap
</code></pre><p>但是在我们定义的Request中，我们实现了按照不同类型获取参数的方法。</p><pre><code class="language-go">// form表单中带的参数
DefaultFormInt(key string, def int) (int, bool)
DefaultFormInt64(key string, def int64) (int64, bool)
DefaultFormFloat64(key string, def float64) (float64, bool)
DefaultFormFloat32(key string, def float32) (float32, bool)
DefaultFormBool(key string, def bool) (bool, bool)
DefaultFormString(key string, def string) (string, bool)
DefaultFormStringSlice(key string, def []string) ([]string, bool)
DefaultFormFile(key string) (*multipart.FileHeader, error)
DefaultForm(key string) interface{}
</code></pre><p>而且在Response中，我们的设计是带有链式调用方法的，而Gin中没有。</p><pre><code class="language-plain">c.SetOkStatus().Json("ok, UserLoginController: " + foo)
</code></pre><p>这两点在使用者使用框架开发具体业务的时候会非常便利，所以我们将这些功能集成到Gin中，迁移这部分Request和Response的封装。</p><p>最后一个优雅关闭的逻辑，我们和Gin都是直接使用 HTTP 库的 server.Shutdown 实现的，不受 Gin 代码迁移的影响。</p><p>所以再明确下，context、路由、中间件、重启机制，我们都不需要迁移，唯一需要迁移的是对Request和Response的封装。</p><h3>使用加法最小化和定制化我们的需求</h3><p>对于 request 和 response 的封装，涉及易用性，我们希望能保留自己的定制化需求，同时又不希望影响 Gin 原有的代码逻辑。所以，可以用<strong>加法最小化的方式</strong>迁移这个封装。</p><p>为了尽量不侵入原有的文件，我们创建两个新的文件 hade_request.go、hade_response.go 来存放之前对request和response的封装。</p><p>回顾下第五节课封装的request，定义的IRequest接口包含：通过地址URL获取参数的QueryXXX系列接口、通过路由匹配获取参数的ParamXXX系列接口、通过Form表单获取参数的FormXXX系列接口，以及一些基础接口。</p><p><strong>所以现在的目标是，要让Gin框架的Context也实现这些接口</strong>。对比之前写的和Gin框架原有的实现方法，可以发现，接口存在下列三种情况：</p><ol>
<li>Gin框架中已经存在相同参数、相同返回值、相同接口名的接口</li>
<li>Gin框架中不存在相同接口名的接口</li>
<li>Gin框架中存在相同接口名，但是不同返回值的接口</li>
</ol><p>第一种情况，由于Gin框架原先就已经有了相同的接口，所以不需要做任何迁移动作，Gin的Context就已经具备了我们设计的封装。对第二种情况来说，由于Gin框架没有对应接口，我们把之前实现的接口原封不动迁移过来即可。</p><p>对于第三种情况则棘手一些。以Gin中已经有的QueryXXX系列接口为例，它的QueryXXX系列接口和我们想要的有一定差别，比如它的Query不支持多种数据类型的直接获取。怎么办？</p><p><strong>可以选择将QueryXXX系列接口重新命名</strong>，又因为Query接口都带有一个默认值，所以我们将其重新命名为DefaultQueryXXX。</p><p>经过上述三种情况的修改，IRequest的定义修改为(在框架目录的framework/gin/hade_request.go中)：</p><pre><code class="language-go">// 代表请求包含的方法
type IRequest interface {

	// 请求地址url中带的参数
	// 形如: foo.com?a=1&amp;b=bar&amp;c[]=bar
	DefaultQueryInt(key string, def int) (int, bool)
	DefaultQueryInt64(key string, def int64) (int64, bool)
	DefaultQueryFloat64(key string, def float64) (float64, bool)
	DefaultQueryFloat32(key string, def float32) (float32, bool)
	DefaultQueryBool(key string, def bool) (bool, bool)
	DefaultQueryString(key string, def string) (string, bool)
	DefaultQueryStringSlice(key string, def []string) ([]string, bool)

	// 路由匹配中带的参数
	// 形如 /book/:id
	DefaultParamInt(key string, def int) (int, bool)
	DefaultParamInt64(key string, def int64) (int64, bool)
	DefaultParamFloat64(key string, def float64) (float64, bool)
	DefaultParamFloat32(key string, def float32) (float32, bool)
	DefaultParamBool(key string, def bool) (bool, bool)
	DefaultParamString(key string, def string) (string, bool)
	DefaultParam(key string) interface{}

	// form表单中带的参数
	DefaultFormInt(key string, def int) (int, bool)
	DefaultFormInt64(key string, def int64) (int64, bool)
	DefaultFormFloat64(key string, def float64) (float64, bool)
	DefaultFormFloat32(key string, def float32) (float32, bool)
	DefaultFormBool(key string, def bool) (bool, bool)
	DefaultFormString(key string, def string) (string, bool)
	DefaultFormStringSlice(key string, def []string) ([]string, bool)
	DefaultFormFile(key string) (*multipart.FileHeader, error)
	DefaultForm(key string) interface{}

	// json body
	BindJson(obj interface{}) error

	// xml body
	BindXml(obj interface{}) error

	// 其他格式
	GetRawData() ([]byte, error)

	// 基础信息
	Uri() string
	Method() string
	Host() string
	ClientIp() string

	// header
	Headers() map[string]string
	Header(key string) (string, bool)

	// cookie
	Cookies() map[string]string
	Cookie(key string) (string, bool)
}
</code></pre><p>IRequest的封装就迁移完成了，对于我们封装的IResponse结构，也是同样的思路。</p><p>和Gin的response实现对比之后，我们发现由于设计了一个链式调用，很多方法的返回值使用 IResponse 接口本身，所以大部分定义的IResponse的接口，在Gin中都有同样接口名，但是返回值不同。所以我们可以用同样的方式来修改接口名。</p><p>因为大都返回IResponse接口，那么可以在所有接口名前面，加一个I字母作为区分。在 framework/gin/hade_response.go中：</p><pre><code class="language-go">// IResponse代表返回方法
type IResponse interface {
	// Json输出
	IJson(obj interface{}) IResponse

	// Jsonp输出
	IJsonp(obj interface{}) IResponse

	//xml输出
	IXml(obj interface{}) IResponse

	// html输出
	IHtml(template string, obj interface{}) IResponse

	// string
	IText(format string, values ...interface{}) IResponse

	// 重定向
	IRedirect(path string) IResponse

	// header
	ISetHeader(key string, val string) IResponse

	// Cookie
	ISetCookie(key string, val string, maxAge int, path, domain string, secure, httpOnly bool) IResponse

	// 设置状态码
	ISetStatus(code int) IResponse

	// 设置200状态
	ISetOkStatus() IResponse
}
</code></pre><p>现在 IRequest 和 IResponse 接口的修改已经完成了。</p><p>下面我们就应该迁移每个接口的具体实现。这里接口的实现比较多，就不一一赘述，Request和Response我们分别举其中一个接口例子进行说明，其他的接口迁移可以具体参考GitHub仓库的<a href="https://github.com/gohade/coredemo/tree/geekbang/09">geekbang/09分支</a>。</p><p>在Request中我们定义了一个DefaultQueryInt方法，是Gin框架中没有的，怎么迁移这个接口呢？首先将之前定义的QueryInt迁移过来，并重新命名为 DefaultQueryInt。</p><pre><code class="language-go">// 获取请求地址中所有参数
func (ctx *Context) QueryAll() map[string][]string {
   if ctx.request != nil {
      return map[string][]string(ctx.request.URL.Query())
   }
   return map[string][]string{}
}

// 获取Int类型的请求参数
func (ctx *Context) DefaultQueryInt(key string, def int) (int, bool) {
   params := ctx.QueryAll()
   if vals, ok := params[key]; ok {
      if len(vals) &gt; 0 {
         // 使用cast库将string转换为Int
         return cast.ToInt(vals[0]), true
      }
   }
   return def, false
}
</code></pre><p>然后这里看QueryAll函数，其实是可以优化的。Gin框架在获取QueryAll的时候，使用了一个QueryCache，实现了在第一次获取参数的时候，用方法initQueryCache 将 ctx.request.URL.Query() 缓存在内存中。</p><p>所以既然已经源码引入Gin框架了，我们也是可以使用这个方法来优化QueryAll() 方法的，先调用initQueryCache，再直接从queryCache中返回参数：</p><pre><code class="language-go">// 获取请求地址中所有参数
func (ctx *Context) QueryAll() map[string][]string {
   ctx.initQueryCache()
   return map[string][]string(ctx.queryCache)
}
</code></pre><p>这样QueryAll方法和DefaultQueryInt方法就都迁移改造完成了。</p><p>在Response中，我们没有需要优化的点，只要将代码迁移就行。比如原先定义的Jsonp方法：</p><pre><code class="language-go">// Jsonp输出
func (ctx *Context) Jsonp(obj interface{}) IResponse {
   ...
}
</code></pre><p>直接修改函数名为IJsonp即可：</p><pre><code class="language-go">// Jsonp输出
func (ctx *Context) IJsonp(obj interface{}) IResponse {
   ...
}
</code></pre><p>接口和实现都修改了，最后肯定还要对应修改下业务代码中之前定义的一些控制器。</p><p>第一个是控制器的参数，从framework.Context 修改为 gin.Context， 这里的gin是引用github.com/gohade/hade/framework/gin，还有把之前定义的Handler的error返回值删除。</p><p>第二个是修改里面的调用，因为现在的Response方法都带上了一个前缀I。比如在业务目录下subject_controller.go中，把原先的SubjectListController：</p><pre><code class="language-go">func SubjectListController(c *framework.Context) error {
   c.SetOkStatus().Json("ok, SubjectListController")
   return nil
}
</code></pre><p>修改为：</p><pre><code class="language-go">func SubjectListController(c *gin.Context) {
   c.ISetOkStatus().IJson("ok, SubjectListController")
}
</code></pre><h2>验证</h2><p>所有修改完成之后，我们可以通过 test来进行验证，调用 <code>go test ./...</code> 来运行Gin程序的所有测试用例，显示成功则表示我们的迁移成功。<br>
<img src="https://static001.geekbang.org/resource/image/85/30/85832f31e6835617b804239de53cd430.png?wh=1630x378" alt=""></p><p>并且我们通过  <code>go build &amp;&amp; ./hade</code> 可以看到熟悉的gin调试模式的输出：<br>
<img src="https://static001.geekbang.org/resource/image/c3/55/c3f631feba724cc00f51033801480d55.png?wh=1378x542" alt=""></p><h2>小结</h2><p>今天我们讨论几个开源项目的许可协议，比如Gin 框架使用的 MIT 协议，在明确修改权限后，我们将Gin框架迁移进自己手写的hade框架，替换前面开发的Context、路由等模块，为后续拓展做好准备。</p><p>在迁移的过程中，我们选择使用复制源码的方式进行替换，并且用了加法最小化的方法，尽量保留了我们的定制化接口。可能有的同学会觉得这种方式比较暴力，但是后续随着我们对框架的改造需求不断增加，这种方式会越来越体现出其优势。</p><h3>思考题</h3><p>我们的hade框架也希望用MIT协议进行开源，你知道如何改造来将它开源么？</p><p>欢迎在留言区分享你的思考。感谢你的收听，如果你觉得有收获，也欢迎你把今天的内容分享给你身边的朋友，邀他一起学习。我们下节课见。</p>
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
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTI7rprdxQqcea53sw1HCz1YZjQNlSvWg7GETnWYicZLYOQR2GUMOwUnrhAIYzUKJt1zZhUv9icOCztQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>鸭补一生如梦</span>
  </div>
  <div class="_2_QraFYR_0">现在定制的是 gin 的 1.7.3 版本，那么后续 gin 升级了，尤其是频繁升级，我们如何快速及时的进行升级更新？<br>或者说隔一段时间再更新？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-06 10:22:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/d7/f2/82e1dc03.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>181</span>
  </div>
  <div class="_2_QraFYR_0">--- FAIL: TestContextFormFileFailed (0.00s)<br>panic: runtime error: invalid memory address or nil pointer dereference [recovered]<br>        panic: runtime error: invalid memory address or nil pointer dereference<br>[signal SIGSEGV: segmentation violation code=0x1 addr=0x18 pc=0x6678d2]<br><br><br>这是啥原因</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-07 15:15:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/9a/0f/da7ed75a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>芒果少侠</span>
  </div>
  <div class="_2_QraFYR_0">迁移完成后，遇到testcase不通过的问题。<br>https:&#47;&#47;github.com&#47;gin-gonic&#47;gin&#47;blob&#47;21125bbb3f550dbfa4c64151f7e01f58dd64e2b8&#47;context_test.go#L352<br>如果是这个testcase，修改正则中包路径那部分即可（与自己项目go module path保持一致即可）。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-09 12:01:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/45/62/3c6041e7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>木小柒</span>
  </div>
  <div class="_2_QraFYR_0">DefaultQueryXXX 是不是没有实现 形如: foo.com?a=1&amp;b=bar&amp;c[]=bar 中 c 的获取？<br>我看 默认就支持 c=1&amp;c=2&amp;c=3 能获取到 c 的 slice [1, 2, 3]<br>但是 c[]=1&amp;c[]=2&amp;c[]=3 是获取不到的，除了自己解析不知道有没有更正常一点的方式</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-01 13:08:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/65/b7/058276dc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>i_chase</span>
  </div>
  <div class="_2_QraFYR_0">http.request和response其实没有可扩展的，所以gin没有使用接口吧</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-14 17:25:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/87/92/9ac4c335.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>🌿</span>
  </div>
  <div class="_2_QraFYR_0">hade_request.go,hade_response.go,hade_context.go；类型、结构体、方法等命名可采用Object-C的前缀命名规则</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-29 17:31:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/26/b5/74/cd80b9f4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>友</span>
  </div>
  <div class="_2_QraFYR_0">大家不要只看文章不看github的代码 很细节的放在文章中 很占篇幅 一般都是说个思路 然后自己去迁移<br>不过 github的代码有些细节没处理好<br> IRequest 很多方法其实已经删除了 但是接口定义还在 会让人迷糊  所以会有小伙伴说没有实现所有办法 这个得自己去判断思考的。<br>不过希望老师之后可以注意下</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢提醒</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-04 00:02:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/26/b5/74/cd80b9f4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>友</span>
  </div>
  <div class="_2_QraFYR_0">作为 方法名叫做 QueryAll 以及 Form  和 Param 这种方法 改变成 DefaultXXX 方法签名语义上并没有default的属性  老师这里是否可以修改下 还是说也是合理的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 有点没有理解，FormAll ， QueryAll 这个语义上是没有Default的属性，参数也没有default的值，所以这个命名没问题吧</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-03 23:04:16</div>
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
  <div class="_2_QraFYR_0">有没有可能将gin封装一个provider，然后在封装的provider里面加container，这样就不用在gin源码上更改了呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是可以的，主要能直接使用context来make容器中的服务出来，如果在gin外面再封装一层也是可以的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-23 09:33:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/7a/55/2f4055f6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>void</span>
  </div>
  <div class="_2_QraFYR_0">这节的代码并没有把 IRequest 接口的方法都实现呀</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-12 16:39:08</div>
  </div>
</div>
</div>
</li>
</ul>