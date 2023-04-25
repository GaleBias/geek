<audio title="23｜管理接口：如何集成swagger自动生成文件？" src="https://static001.geekbang.org/resource/audio/ab/a8/ab932f310208bb2d7de2ac0694442ba8.mp3" controls="controls"></audio> 
<p>你好，我是轩脉刃。</p><p>不管你是前端页面开发，还是后端服务开发，你一定经历过前后端联调的场景，前后端联调最痛苦的事情，莫过于没有完善的接口文档、没有可以调用调试的接口返回值了，所以一般都会采用形如Postman这样的第三方工具，来进行接口的调用和联调。</p><p>但是这一节课，我们要做的事情，就是为自己的Web应用集成swagger，使用swagger自动生成一个可以查看接口、可以调用执行的页面。</p><h3>swagger</h3><p>说到swagger，可能有的同学还比较陌生，我来简要介绍一下。swagger框架在2009年启动，之前是Reverb公司内部开发的一个项目，他们的工程师在与第三方调试REST接口的过程中，为了解决大量的接口与文档问题，就设计了swagger这个项目。</p><p>项目最终成型的方案是，先设计一个JSON规则，开发工程师把所有服务接口按照这种规则来写成一个JSON文件，<strong>这个JSON文件可以直接生成一个交互式UI，可以提供调用者查看、调用调试</strong>。</p><p>swagger的应用是非常广泛的。非常多的开源项目在提供对外接口的时候都使用swagger来进行描述。比如目前最火的Kubernetes项目，每次在发布版本的时候，都会在项目根目录上，带上符合swagger规则的<a href="https://github.com/kubernetes/kubernetes/blob/master/api/openapi-spec/swagger.json">JSON文件</a>，用来向使用者提供内部接口。</p><!-- [[[read_end]]] --><p>swagger的产品有两类。</p><p>一个是前面说的JSON规则，就是OpenAPI的文档，它说明了我们要写一个接口说明文档的每个字段含义和要求。</p><p>OpenAPI的规则也是有版本的，目前最新版本是3.0，但是3.0版本目前市场上相应的配套支持还不成熟，比如Golang版本的SDK库<a href="https://github.com/go-openapi/spec">spec</a>还不支持。目前市面上对OpenAPI2.0的支持还是最全的。所以我们的hade框架就使用swagger2.0版本。</p><p>swagger的另外一类产品是工具，包括swagger-ui、swagger-editor和swagger-codegen。</p><p><a href="https://editor.swagger.io/">swagger-editor</a>提供一个开源网站，在线编辑swagger文件。<a href="https://swagger.io/tools/swagger-codegen/download/">swagger-codegen</a>提供一个Java命令行工具，通过swagger文件生成client端代码。而<a href="https://petstore.swagger.io/">swagger-ui</a>，通过提供一个开源网站，将swagger接口在线展示出来，并且可以让调用者查看、调试。我们的目标是生成一个可以查看接口，进行调用调试的页面，所以要将swagger-ui集成进hade框架。</p><h3>命令设计</h3><p>了解了swagger，结合框架，我们照例先思考下希望如何使用它。</p><p>按照swagger的定义，我们应该在业务项目中维护一个JSON文件，这个文件描述了这个业务的所有接口。但是你想过没有，<strong>随着项目的接口数越来越大，维护swagger的JSON描述文档本身，就是一个很大很繁杂的工作量</strong>。</p><p>由于每个接口在代码开发的时候，我们都会有注释，而更新代码的时候，我们是会去更新注释的。所以能不能有一个方法，通过代码的注释，自动生成这个JSON文件呢？</p><p>好，这个就是我们希望定义的一个swagger命令，<code>./hade swagger gen</code> ，能通过注释生成swagger.json文件。</p><p>但是考虑具体的实现设计，怎么用Golang的代码，注释生成swagger.json呢？既然swagger.json是有一定的规则的，那么注释的写法也是有一定规则的吧？是的。目前有一个最流行的将Golang注释转化为swagger.json 的开源项目<a href="https://github.com/swaggo/swag">swag</a>。</p><h3>swag项目</h3><p>这个swag项目是MIT 协议，目前已经有4.9k 个star了。它的用法和我们想要的一样，生成swagger.json分三步：</p><ul>
<li>在API接口中编写注释。注释的详细写法需要参考<a href="https://github.com/swaggo/swag#declarative-comments-format">说明文档</a>。</li>
<li>下载swag工具或者安装swag库</li>
<li>使用工具或者库将指定代码生成swagger.json</li>
</ul><p>步骤很简单，不过第一步怎么写swag的注释说明文档，是使用这个技术必须要学习的一个知识，这个的学习确实是有些门槛的，需要熟读对应的说明文档才能写出比较好的注释。这里我们用一个例子来讲解我在编写代码的时候常用的一些字段，供你参考。</p><pre><code class="language-go">// Demo2  for godoc
// @Summary 获取所有学生
// @Description 获取所有学生，不进行分页
// @Produce  json
// @Tags demo
// @Success 200 {array} []UserDTO
// @Router /demo/demo2 [get]
func (api *DemoApi) Demo2(c *gin.Context) {
   demoProvider := c.MustMake(demoService.DemoKey).(demoService.IService)
   students := demoProvider.GetAllStudent()
   usersDTO := StudentsToUserDTOs(students)
   c.JSON(200, usersDTO)
}

type UserDTO struct {
   ID   int    `json:"id"`
   Name string `json:"name"`
}
</code></pre><p>观察注释。第一行  <code>Demo2 for godoc</code> 这个在swagger中并没有实际作用，它是用来给godoc工具生成说明文档的。从第二行开始，就是我们swaggo的注释语法了，使用@符号加上关键字的方式来进行说明。<br>
例子的关键字有这些：</p><ul>
<li>Summary，为接口增加简要说明</li>
<li>Description，为接口增加详细说明</li>
<li>Produce，说明接口返回格式</li>
<li>Tags，为接口打标签，可以为多个，便于查看者查找</li>
<li>Success，接口返回成功时候的说明</li>
<li>Router，接口的路由调用</li>
</ul><p>具体对应的swagger-ui界面是这样的：<br>
<img src="https://static001.geekbang.org/resource/image/94/ec/94c4a5c83dc771be6f8b60527e9951ec.png?wh=2872x1644" alt=""></p><p>我们对照注释和界面，很容易就看出每个注释的最终显示效果。不过这里再啰嗦解释下比较复杂的Success注释。</p><p>在这个例子中，是这样使用Success注释的：</p><pre><code class="language-go">// @Success 200 {array} UserDTO
</code></pre><p>在成功的时候，返回UserDTO结构的数组，这里，swaggo会自动去项目中寻找UserDTO结构，来生成swagger-ui中的返回结构说明。</p><p>不过这里能这么写，是因为恰好UserDTO是和API放在同一个namespace下，如果你的返回结构放在不同的namespace下，需要在注释中注明返回结构的命名空间。比如：</p><pre><code class="language-go">// @Success 200 {array} model.Account
</code></pre><p>同时，这个返回结构还支持返回对象嵌套，比如下面这个例子：</p><pre><code class="language-go">// 返回了一个JsonResult对象，其中这个对象的data字段是Order结构
@success 200 {object} jsonresult.JSONResult{data=proto.Order} "desc"

type JSONResult struct {
    Code    int          `json:"code" `
    Message string       `json:"message"`
    Data    interface{}  `json:"data"`
}

type Order struct { //in `proto` package
    Id  uint            `json:"id"`
    Data  interface{}   `json:"data"`
}
</code></pre><p>它返回了一个JSONResult对象，这个JSONResult对象中的一个字段data是Order结构。</p><h3>命令实现</h3><p>现在注释已经标记好了，我们再回到生成JSON文件的命令 <code>./hade swagger gen</code> 。</p><p>这个命令通过swaggo准备好的命令行工具swag或者类库，来生成JSON文件。由于我们的框架已经集成了命令行工具，所以不会选择额外使用swag工具，而是在我们的命令中集成swaggo类库：<a href="https://github.com/swaggo/swag/tree/master/gen">swag/gen</a>。</p><p>这个类库最核心的结构就是<a href="https://github.com/swaggo/swag/blob/master/gen/gen.go">Config结构</a>。来看这个swagger gen命令的具体实现代码，写在framework/command/swagger.go中：</p><pre><code class="language-go">// swaggerGenCommand 生成具体的swagger文档
var swaggerGenCommand = &amp;cobra.Command{
   Use:   "gen",
   Short: "生成对应的swagger文件, contain swagger.yaml, doc.go",
   Run: func(c *cobra.Command, args []string) {
      container := c.GetContainer()
      appService := container.MustMake(contract.AppKey).(contract.App)
      outputDir := filepath.Join(appService.AppFolder(), "http", "swagger")
      httpFolder := filepath.Join(appService.AppFolder(), "http")
      conf := &amp;gen.Config{
         // 遍历需要查询注释的目录
         SearchDir: httpFolder,
         // 不包含哪些文件
         Excludes: "",
         // 输出目录
         OutputDir: outputDir,
         // 整个swagger接口的说明文档注释
         MainAPIFile: "swagger.go",
         // 名字的显示策略，比如首字母大写等
         PropNamingStrategy: "",
         // 是否要解析vendor目录
         ParseVendor: false,
         // 是否要解析外部依赖库的包
         ParseDependency: false,
         // 是否要解析标准库的包
         ParseInternal: false,
         // 是否要查找markdown文件，这个markdown文件能用来为tag增加说明格式
         MarkdownFilesDir: "",
         // 是否应该在docs.go中生成时间戳
         GeneratedTime: false,
      }
      err := gen.New().Build(conf)
      if err != nil {
         fmt.Println(err)
      }
   },
}
</code></pre><p>结合这个具体实现，我们来看这个Config结构的关键字段SearchDir、OutputDir和MainAPIFile，这几个字段的含义必须完全理解才能设置正确，其他的几个字段如果不理解，直接使用默认值就行。</p><p><strong>第一个 SearchDir 表示要swaggo去哪个目录遍历代码的注释，来生成swagger的JSON文件</strong>。对于我们的hade框架，所有接口文件都存放在app/http文件夹中，所以要遍历的就是这个文件夹了。</p><p>第二个关键字段 OutputDir，表示要输出的swagger文件的存放地址。我们在app/http目录下，创建一个swagger目录，来存放要输出的swagger文件。这里再补充一点，前面说swagger最终会生成JSON文件，但是你运行一次swagger gen 会发现，这个生成目录下除了有swagger.json这个文件，还有两个文件swagger.yaml 和 docs.go。</p><p>对于额外生成的这两个文件，swagger.yaml 是YAML格式的接口说明文档，里面的内容和swagger.json其实是一样的。<strong>而docs.go 是“接口说明文档”的代码，它是为go项目直接引入接口说明文档生成swagger-ui用的</strong>。</p><p>也就是说生成的docs.go，我们的框架只需要import它，就能从这个文件的变量doc中直接获取到“接口说明文档”，不需要用读取文件的方式读取swagger.json 或者swagger.yaml。下一节课我们会使用它来让框架启动服务的时候自动启动swagger-ui。</p><p><strong>第三个关键字段 MainAPIFile，表示整个swagger接口的说明文档</strong>。这是什么意思呢？在最终生成的swagger-ui界面上的头部，你会看到对当前swagger接口的整体说明，包括作者、接口版本、接口licence等信息。<br>
<img src="https://static001.geekbang.org/resource/image/8e/a8/8ec3d0027ec7ed8e76587dcfcf7b87a8.png?wh=2918x718" alt=""></p><p>这些信息也都是使用注释来自动生成的，而这一部分注释，就是存放在这个MainApiFile所指向的Go文件中。</p><p>我们的项目就固定将这个文件命名为swagger.go，存放在app/http/swagger.go 文件中。这个文件只是增加注释，不增加任何的业务逻辑。其中的每个注释的关键字说明，也是参考swaggo的<a href="https://github.com/swaggo/swag#declarative-comments-format">说明文档</a>。</p><pre><code class="language-go">// Package http API.
// @title hade
// @version 1.1
// @description hade测试
// @termsOfService https://github.com/swaggo/swag

// @contact.name yejianfeng1
// @contact.email yejianfeng

// @license.name Apache 2.0
// @license.url http://www.apache.org/licenses/LICENSE-2.0.html

// @BasePath /
// @query.collection.format multi

// @securityDefinitions.basic BasicAuth

// @securityDefinitions.apikey ApiKeyAuth
// @in header
// @name Authorization

// @x-extension-openapi {"example": "value on a json format"}

package http
</code></pre><p>现在按照前面的说明，我们设置好了Config结构，在swaggerGenCommand 命令逻辑的最后，只需调用一次Build方法，就能按照Config配置来生成 docs.go、swagger.json、swagger.yaml这三个文件了。</p><pre><code class="language-go">err := gen.New().Build(conf)
</code></pre><p>记得将这个二级命令 <code>./hade swagger gen</code> 挂载到一级命令 <code>./hade swagger</code> 下，并且挂载到framework/command/kernel.go 中。这个步骤，前面几章中都已经重复做过了，这里就不赘述了。</p><p>挂载好之后，我们尝试运行一下：<br>
<img src="https://static001.geekbang.org/resource/image/2e/b7/2e46d84c14b1b219abdd5cf378d9edb7.png?wh=1650x340" alt=""></p><p>可以看到，日志信息打印非常详细，包括去哪个目录查找、最终生成哪些文件。</p><p>查看app/http/swagger目录，确实最终生成了docs.go、swagger.json、swagger.yaml 这三个文件：<br>
<img src="https://static001.geekbang.org/resource/image/f2/3b/f2f05578d6483d2e4a4d483aee1ecb3b.png?wh=680x574" alt=""></p><h3>启动swagger-ui</h3><p>有了swagger生成文件了，应该怎么使用它呢？还是先设想，我们希望的是，能在启动服务的时候，同时启动一个swagger-ui页面，给接口使用人员来查看服务接口，并且他们可以直接在这个页面进行接口调用。</p><p>到这里相信你已经想到了，在启动服务的时候，<strong>增加一个打开swagger-ui页面路由</strong>，就可以达到我们的目的了。还是查看swaggo这个项目，<a href="https://github.com/swaggo/swag#how-to-use-it-with-gin">官方文档</a>中有一段如何将swaggo结合Gin来生成路由的方法说明，正好我们框架的路由用的Gin，所以就考虑使用这个方法来开辟一个swagger-ui路由。</p><p>swaggo开发了一个<a href="https://github.com/swaggo/gin-swagger">gin-swagger</a> 中间件，来为Gin框架增加路由设置。怎么使用它呢？看官方文档的这个<a href="https://github.com/swaggo/gin-swagger">例子</a>：</p><pre><code class="language-go">package main

import (
&nbsp; &nbsp;"github.com/gin-gonic/gin"
&nbsp; &nbsp;docs "github.com/go-project-name/docs"
&nbsp; &nbsp;swaggerfiles "github.com/swaggo/files"
&nbsp; &nbsp;ginSwagger "github.com/swaggo/gin-swagger"
&nbsp; &nbsp;"net/http"
)
// @BasePath /api/v1

// PingExample godoc
// @Summary ping example
// @Schemes
// @Description do ping
// @Tags example
// @Accept json
// @Produce json
// @Success 200 {string} Helloworld
// @Router /example/helloworld [get]
func Helloworld(g *gin.Context)&nbsp; {
&nbsp; &nbsp;g.JSON(http.StatusOK,"helloworld")
}

func main()&nbsp; {
&nbsp; &nbsp;r := gin.Default()
&nbsp; &nbsp;docs.SwaggerInfo.BasePath = "/api/v1"
&nbsp; &nbsp;v1 := r.Group("/api/v1")
&nbsp; &nbsp;{
&nbsp; &nbsp; &nbsp; eg := v1.Group("/example")
&nbsp; &nbsp; &nbsp; {
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;eg.GET("/helloworld",Helloworld)
&nbsp; &nbsp; &nbsp; }
&nbsp; &nbsp;}
&nbsp; &nbsp;r.GET("/swagger/*any", ginSwagger.WrapHandler(swaggerfiles.Handler))
&nbsp; &nbsp;r.Run(":8080")
}
</code></pre><h3>gin-swagger原理分析</h3><p>我们着重注意下这几行：</p><pre><code class="language-go">package main

import (
   ...
&nbsp; &nbsp;docs "github.com/go-project-name/docs"
&nbsp; &nbsp;swaggerfiles "github.com/swaggo/gin-swagger/swaggerFiles"
&nbsp; &nbsp;ginSwagger "github.com/swaggo/gin-swagger"
   ...
)
...

func main()&nbsp; {
&nbsp; ...
&nbsp; &nbsp;r.GET("/swagger/*any", ginSwagger.WrapHandler(swaggerfiles.Handler))
&nbsp; &nbsp;...
}
</code></pre><p>首先，import的三个文件分别是做什么用的，必须要理解清楚。</p><p>swagger-ui是一个HTML + JSON接口的页面，那么里面分别有动态内容和静态内容，静态内容包括HTML、JS、CSS、png 等，动态内容包括swagger JSON化的结构。也就是说我们现在的<strong>目标是要创建一个包含HTML+JSON的服务</strong>，如何做呢？</p><p>一种方法当然是将这些静态文件直接放在项目中，在服务启动的时候，使用读取文件的方式并返回这些静态文件提供服务。但是这就要求在发布的时候，这些静态文件，必须同时被带着上线，它们成为了服务必须的一部分，特别是作为一个类库提供的时候，如果要求使用者必须带着库的静态文件，是非常不方便的。</p><p>而这里gin-swagger采用了另外一种更为极致的做法，是<strong>将这些静态文件代码化，嵌入到go代码中</strong>，比如让一个变量返回HTML的内容，我们在提供获取HTML页面的服务时，直接将变量返回就可以了。</p><p>这里代码第6行的github.com/swaggo/gin-swagger/swaggerFiles，这个库就做了这个事情，它将swagger-ui的所有HTML、JS、CSS、png 文件都变化成为了go文件，并且作为HTTP服务提供出来。其中的swaggerfiles.Handler 就是实现了net/http 的<a href="https://pkg.go.dev/net/http#HandlerFunc"> HandlerFunc 接口</a>。</p><p>而另外一部分，动态JSON接口，返回的是具体的swagger JSON化的内容。这个怎么获取呢？</p><p>是通过前面我们说的swaggo 生成的几个文件中的doc.go 文件来获取的，在例子中就是第5行引入的 github.com/go-project-name/docs 库，它的原理就是生成doc 全局变量，并且通过 ReadDoc() 方法来提供JSON的内容读取。</p><p>好，现在有了动态JSON接口和静态文件服务接口，如何集成到Gin的Engine里呢？</p><p>要一个中间件就行了，就是第7行 import中引入的 github.com/swaggo/gin-swagger 库。它通过创建一个Gin的中间件，将动态和静态的请求都承接起来，静态请求就请求到swaggo/files库，动态请求就请求到docs库中。</p><p>所以在路由中，我们*<em>创建一个路由/swagger/<em>any，就可以获取swagger-ui 并且读取swaggo 创建的doc.go 文件内容了</em></em>。</p><pre><code class="language-go">r.GET("/swagger/*any", ginSwagger.WrapHandler(swaggerfiles.Handler))
</code></pre><h3>如何集成</h3><p>好，gin-swagger的原理理解清楚了，如何集成进入我们框架呢？</p><p>gin-swagger本质就是一个Gin中间件。集成Gin的中间件，我们只需要拷贝这个中间件源码，并且将中间件的 github.com/gin-gonic/gin 替换为hade框架的gin地址：github.com/gohade/hade/framework/gin就可以了。</p><p>所以把这个中间件放在framework/middleware/gin-swagger 下。存放完之后的目录截图，你可以对比检查一下：<br>
<img src="https://static001.geekbang.org/resource/image/e3/de/e367057db0277465e92141d14cf112de.png?wh=1042x1292" alt=""></p><p>把gin-swagger中间件处理完，就剩下将路由存放到我们的app业务路由中去了。</p><p>这里再设计一个小细节。<strong>毕竟我们其实并不希望线上服务也提供这么一个swagger路由，也就是说只希望swagger-ui在测试和开发环境使用</strong>，所以可以在配置文件app.yaml 中有这么一个配置项：</p><pre><code class="language-go">swagger: true
</code></pre><p>来表示是否开启这个swagger路由。<br>
那在应用路由中如何获取到这个配置呢？之前应用路由中的参数只有一个gin.Engine。我们需要为gin.Enigne增加一个获取服务容器的接口，GetContainer()。来修改framework/gin/hade_engine.go：</p><pre><code class="language-go">// GetContainer 从Engine中获取container
func (engine *Engine) GetContainer() framework.Container {
   return engine.container
}
</code></pre><p>现在就是真正的万事俱备了，我们来改造应用路由app/http/route.go。</p><ul>
<li>首先要引入gin-swagger提示的三个import。</li>
</ul><p>这里我将最后一个docs对应的import，放在了同级目录的app/http/swagger.go文件中。</p><p>我是这么考虑的，<strong>docs.go是我们用命令行生成的，而生成的时候swagger的全局说明配置是放在swagger.go中的</strong>，所以这两个文件关系更为紧密，比较适合放在一起。</p><p>app/http/swagger.go文件信息：</p><pre><code class="language-go">// Package http API.
// @title hade
// @version 1.1
// @description hade测试
// @termsOfService https://github.com/swaggo/swag
...
package http

import (
    _ "github.com/gohade/hade/app/http/swagger"
)
</code></pre><p>回到app/http/route.go中。我们引入了另外两个库，并且先判断app.swagger配置项是否为true，如果为true，则开启swagger路由。</p><p>app/http/route.go代码实现，你应该能很容易写出来。要点刚才都详细讲过了：</p><pre><code class="language-go">package http

import (
   ...
   ginSwagger "github.com/gohade/hade/framework/middleware/gin-swagger"
   "github.com/gohade/hade/framework/middleware/gin-swagger/swaggerFiles"
   ...
)

// Routes 绑定业务层路由
func Routes(r *gin.Engine) {
   container := r.GetContainer()
   configService := container.MustMake(contract.ConfigKey).(contract.Config)

   ...

   // 如果配置了swagger，则显示swagger的中间件
   if configService.GetBool("app.swagger") == true {
      r.GET("/swagger/*any", ginSwagger.WrapHandler(swaggerFiles.Handler))
   }

   ...
}
</code></pre><p>到这里我们就完美将swagger-ui集成进入我们服务了。</p><h3>验证</h3><p>最后做一下验证。先用  <code>./hade swagger gen</code> 命令生成我们要的docs.go文件：<br>
<img src="https://static001.geekbang.org/resource/image/6d/dc/6d19784c05ec703a26fd7df34d7dabdc.png?wh=1646x358" alt=""></p><p>再使用 <code>./hade build self</code> ，将生成的docs.go 打包编译进./hade命令，启动服务  <code>./hade app start</code> ：<br>
<img src="https://static001.geekbang.org/resource/image/12/a8/1263d0d128550ca1bf6abfd8631045a8.png?wh=1014x154" alt=""></p><p>浏览器打开地址 <a href="http://localhost:8888/swagger/index.html">http://localhost:8888/swagger/index.html</a> ，可以看到整个swagger-ui界面：<br>
<img src="https://static001.geekbang.org/resource/image/d4/9e/d483de87fa8da4b313cc8eeefb80d19e.png?wh=3584x2904" alt=""></p><p>并且点击某个接口的 execute 按钮，可以真实地调用这个接口，返回返回数据，进行调试。非常方便：<br>
<img src="https://static001.geekbang.org/resource/image/9c/29/9c54d8909aeff6f2752ce8abf4664329.png?wh=2288x1566" alt=""></p><p>验证完成！</p><p>今天所有代码都保存在GitHub上的<a href="https://github.com/gohade/coredemo/tree/geekbang/23">geekbang/23</a>分支了。附上目录结构供你对比查看。<br>
<img src="https://static001.geekbang.org/resource/image/bb/bb/bb80e3120d81cc92668874d5270f2cbb.png?wh=776x890" alt=""><br>
<img src="https://static001.geekbang.org/resource/image/d7/8b/d793238b11b89ccdfa78fbcf2a73988b.png?wh=1122x1570" alt=""></p><h3>小结</h3><p>这一节课，我们其实就做了一件事情：将swagger融合进入hade框架。</p><p>我们依赖swag项目和gin-swagger中间件，成功地将swagger放到hade框架中，之后使用一个配置，能同时启动hade后端服务和swagger前端调试工具，自动生成一个可以查看接口、可以调用执行的页面。相信在实际工作中开发过后端接口的同学就知道这个工具是有多实用。</p><p>当然熟练使用swagger，以及熟练编写swagger的代码注释，需要对swagger的规则和swag的注释定义有一定了解，这个需要你花时间去掌握。但是相信我，虽然写swagger注释有一些繁琐，但是它能节省大量你和前端同学联调的时间。</p><h3>思考题</h3><p>我之前在一个项目中使用swagger的JSON文件自动生成了项目的接口word说明文档。不知道你在实际工作中，是如何使用swagger的呢？能分享一下你/你们公司使用swagger的一些经历么？</p><p>欢迎在留言区分享你的思考。感谢你的收听，如果觉得今天的内容对你有所帮助，也欢迎分享给你身边的朋友，邀请他一起学习。我们下节课见～</p>
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
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_62f18d</span>
  </div>
  <div class="_2_QraFYR_0">您好，请问swagger的注释中的description.markdown怎么使用</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: https:&#47;&#47;github.com&#47;gohade&#47;coredemo&#47;commit&#47;26ad0cd830ad0da062a2a24cc517458e2dad704f<br><br>我写了一个例子在geekbang&#47;24分支上，你可以参考看下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-01 13:18:03</div>
  </div>
</div>
</div>
</li>
</ul>