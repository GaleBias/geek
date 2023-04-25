<audio title="17 _ API 文档：如何生成 Swagger API 文档 ？" src="https://static001.geekbang.org/resource/audio/8d/3f/8d4b8ef3cac87bd828bc7a25d082bb3f.mp3" controls="controls"></audio> 
<p>你好，我是孔令飞。</p><p>作为一名开发者，我们通常讨厌编写文档，因为这是一件重复和缺乏乐趣的事情。但是在开发过程中，又有一些文档是我们必须要编写的，比如API文档。</p><p>一个企业级的Go后端项目，通常也会有个配套的前端。为了加快研发进度，通常是后端和前端并行开发，这就需要后端开发者在开发后端代码之前，先设计好API接口，提供给前端。所以在设计阶段，我们就需要生成API接口文档。</p><p>一个好的API文档，可以减少用户上手的复杂度，也意味着更容易留住用户。好的API文档也可以减少沟通成本，帮助开发者更好地理解API的调用方式，从而节省时间，提高开发效率。这时候，我们一定希望有一个工具能够帮我们自动生成API文档，解放我们的双手。Swagger就是这么一个工具，可以帮助我们<strong>生成易于共享且具有足够描述性的API文档</strong>。</p><p>接下来，我们就来看下，如何使用Swagger生成API文档。</p><h2>Swagger介绍</h2><p>Swagger是一套围绕OpenAPI规范构建的开源工具，可以设计、构建、编写和使用REST API。Swagger包含很多工具，其中主要的Swagger工具包括：</p><ul>
<li><strong>Swagger编辑器：</strong>基于浏览器的编辑器，可以在其中编写OpenAPI规范，并实时预览API文档。<a href="https://editor.swagger.io/">https://editor.swagger.io</a> 就是一个Swagger编辑器，你可以尝试在其中编辑和预览API文档。</li>
<li><strong>Swagger UI：</strong>将OpenAPI 规范呈现为交互式API文档，并可以在浏览器中尝试API调用。</li>
<li><strong>Swagger Codegen：</strong>根据OpenAPI规范，生成服务器存根和客户端代码库，目前已涵盖了40多种语言。</li>
</ul><!-- [[[read_end]]] --><h3>Swagger和OpenAPI的区别</h3><p>我们在谈到Swagger时，也经常会谈到OpenAPI。那么二者有什么区别呢？</p><p>OpenAPI是一个API规范，它的前身叫Swagger规范，通过定义一种用来描述API格式或API定义的语言，来规范RESTful服务开发过程，目前最新的OpenAPI规范是<a href="https://swagger.io/docs/specification">OpenAPI 3.0</a>（也就是Swagger 2.0规范）。</p><p>OpenAPI规范规定了一个API必须包含的基本信息，这些信息包括：</p><ul>
<li>对API的描述，介绍API可以实现的功能。</li>
<li>每个API上可用的路径（/users）和操作（GET /users，POST /users）。</li>
<li>每个API的输入/返回的参数。</li>
<li>验证方法。</li>
<li>联系信息、许可证、使用条款和其他信息。</li>
</ul><p>所以，你可以简单地这么理解：OpenAPI是一个API规范，Swagger则是实现规范的工具。</p><p>另外，要编写Swagger文档，首先要会使用Swagger文档编写语法，因为语法比较多，这里就不多介绍了，你可以参考Swagger官方提供的<a href="https://swagger.io/specification/">OpenAPI Specification</a>来学习。</p><h2>用go-swagger来生成Swagger API文档</h2><p>在Go项目开发中，我们可以通过下面两种方法来生成Swagger API文档：</p><p>第一，如果你熟悉Swagger语法的话，可以直接编写JSON/YAML格式的Swagger文档。建议选择YAML格式，因为它比JSON格式更简洁直观。</p><p>第二，通过工具生成Swagger文档，目前可以通过<a href="https://github.com/swaggo/swag">swag</a>和<a href="https://github.com/go-swagger/go-swagger">go-swagger</a>两个工具来生成。</p><p>对比这两种方法，直接编写Swagger文档，不比编写Markdown格式的API文档工作量小，我觉得不符合程序员“偷懒”的习惯。所以，本专栏我们就使用go-swagger工具，基于代码注释来自动生成Swagger文档。为什么选go-swagger呢？有这么几个原因：</p><ul>
<li>go-swagger比swag功能更强大：go-swagger提供了更灵活、更多的功能来描述我们的API。</li>
<li>使我们的代码更易读：如果使用swag，我们每一个API都需要有一个冗长的注释，有时候代码注释比代码还要长，但是通过go-swagger我们可以将代码和注释分开编写，一方面可以使我们的代码保持简洁，清晰易读，另一方面我们可以在另外一个包中，统一管理这些Swagger API文档定义。</li>
<li>更好的社区支持：go-swagger目前有非常多的Github star数，出现Bug的概率很小，并且处在一个频繁更新的活跃状态。</li>
</ul><p>你已经知道了，go-swagger是一个功能强大的、高性能的、可以根据代码注释生成Swagger API文档的工具。除此之外，go-swagger还有很多其他特性：</p><ul>
<li>根据Swagger定义文件生成服务端代码。</li>
<li>根据Swagger定义文件生成客户端代码。</li>
<li>校验Swagger定义文件是否正确。</li>
<li>启动一个HTTP服务器，使我们可以通过浏览器访问API文档。</li>
<li>根据Swagger文档定义的参数生成Go model结构体定义。</li>
</ul><p>可以看到，使用go-swagger生成Swagger文档，可以帮助我们减少编写文档的时间，提高开发效率，并能保证文档的及时性和准确性。</p><p>这里需要注意，如果我们要对外提供API的Go SDK，可以考虑使用go-swagger来生成客户端代码。但是我觉得go-swagger生成的服务端代码不够优雅，所以建议你自行编写服务端代码。</p><p>目前，有很多知名公司和组织的项目都使用了go-swagger，例如 Moby、CoreOS、Kubernetes、Cilium等。</p><h3>安装Swagger工具</h3><p>go-swagger通过swagger命令行工具来完成其功能，swagger安装方法如下：</p><pre><code>$ go get -u github.com/go-swagger/go-swagger/cmd/swagger
$ swagger version
dev
</code></pre><h3>swagger命令行工具介绍</h3><p>swagger命令格式为<code>swagger [OPTIONS] &lt;command&gt;</code>。可以通过<code>swagger -h</code>查看swagger使用帮助。swagger提供的子命令及功能见下表：</p><p><img src="https://static001.geekbang.org/resource/image/yy/78/yy3428aa968c7029cb4f6b11f2596678.png?wh=1176x1180" alt=""></p><h2>如何使用swagger命令生成Swagger文档？</h2><p>go-swagger通过解析源码中的注释来生成Swagger文档，go-swagger的详细注释语法可参考<a href="https://goswagger.io">官方文档</a>。常用的有如下几类注释语法：</p><p><img src="https://static001.geekbang.org/resource/image/94/b3/947262c5175f6f518ff677063af293b3.png?wh=1087x1134" alt=""></p><h3>解析注释生成Swagger文档</h3><p>swagger generate命令会找到main函数，然后遍历所有源码文件，解析源码中与Swagger相关的注释，然后自动生成swagger.json/swagger.yaml文件。</p><p>这一过程的示例代码为<a href="https://github.com/marmotedu/gopractise-demo/tree/main/swagger">gopractise-demo/swagger</a>。目录下有一个main.go文件，定义了如下API接口：</p><pre><code>package main

import (
    &quot;fmt&quot;
    &quot;log&quot;
    &quot;net/http&quot;

    &quot;github.com/gin-gonic/gin&quot;

    &quot;github.com/marmotedu/gopractise-demo/swagger/api&quot;
    // This line is necessary for go-swagger to find your docs!
    _ &quot;github.com/marmotedu/gopractise-demo/swagger/docs&quot;
)

var users []*api.User

func main() {
    r := gin.Default()
    r.POST(&quot;/users&quot;, Create)
    r.GET(&quot;/users/:name&quot;, Get)

    log.Fatal(r.Run(&quot;:5555&quot;))
}

// Create create a user in memory.
func Create(c *gin.Context) {
    var user api.User
    if err := c.ShouldBindJSON(&amp;user); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{&quot;message&quot;: err.Error(), &quot;code&quot;: 10001})
        return
    }

    for _, u := range users {
        if u.Name == user.Name {
            c.JSON(http.StatusBadRequest, gin.H{&quot;message&quot;: fmt.Sprintf(&quot;user %s already exist&quot;, user.Name), &quot;code&quot;: 10001})
            return
        }
    }

    users = append(users, &amp;user)
    c.JSON(http.StatusOK, user)
}

// Get return the detail information for a user.
func Get(c *gin.Context) {
    username := c.Param(&quot;name&quot;)
    for _, u := range users {
        if u.Name == username {
            c.JSON(http.StatusOK, u)
            return
        }
    }

    c.JSON(http.StatusBadRequest, gin.H{&quot;message&quot;: fmt.Sprintf(&quot;user %s not exist&quot;, username), &quot;code&quot;: 10002})
}
</code></pre><p>main包中引入的<strong>User struct</strong>位于gopractise-demo/swagger/api目录下的<a href="https://github.com/marmotedu/gopractise-demo/blob/main/swagger/api/user.go">user.go</a>文件：</p><pre><code>// Package api defines the user model.
package api

// User represents body of User request and response.
type User struct {
    // User's name.
    // Required: true
    Name string `json:&quot;name&quot;`

    // User's nickname.
    // Required: true
    Nickname string `json:&quot;nickname&quot;`

    // User's address.
    Address string `json:&quot;address&quot;`

    // User's email.
    Email string `json:&quot;email&quot;`
}
</code></pre><p><code>// Required: true</code>说明字段是必须的，生成Swagger文档时，也会在文档中声明该字段是必须字段。</p><p>为了使代码保持简洁，我们在另外一个Go包中编写带go-swagger注释的API文档。假设该Go包名字为docs，在开始编写Go API注释之前，需要在main.go文件中导入docs包：</p><pre><code>_ &quot;github.com/marmotedu/gopractise-demo/swagger/docs&quot;
</code></pre><p>通过导入docs包，可以使go-swagger在递归解析main包的依赖包时，找到docs包，并解析包中的注释。</p><p>在gopractise-demo/swagger目录下，创建docs文件夹：</p><pre><code>$ mkdir docs
$ cd docs
</code></pre><p>在docs目录下，创建一个doc.go文件，在该文件中提供API接口的基本信息：</p><pre><code>// Package docs awesome.
//
// Documentation of our awesome API.
//
//     Schemes: http, https
//     BasePath: /
//     Version: 0.1.0
//     Host: some-url.com
//
//     Consumes:
//     - application/json
//
//     Produces:
//     - application/json
//
//     Security:
//     - basic
//
//    SecurityDefinitions:
//    basic:
//      type: basic
//
// swagger:meta
package docs
</code></pre><p><strong>Package docs</strong>后面的字符串 <code>awesome</code> 代表我们的HTTP服务名。<code>Documentation of our awesome API</code>是我们API的描述。其他都是go-swagger可识别的注释，代表一定的意义。最后以<code>swagger:meta</code>注释结束。</p><p>编写完doc.go文件后，进入gopractise-demo/swagger目录，执行如下命令，生成Swagger API文档，并启动HTTP服务，在浏览器查看Swagger：</p><pre><code>$ swagger generate spec -o swagger.yaml
$ swagger serve --no-open -F=swagger --port 36666 swagger.yaml

2020/10/20 23:16:47 serving docs at http://localhost:36666/docs
</code></pre><ul>
<li>-o：指定要输出的文件名。swagger会根据文件名后缀.yaml或者.json，决定生成的文件格式为YAML或JSON。</li>
<li>–no-open：因为是在Linux服务器下执行命令，没有安装浏览器，所以使–no-open禁止调用浏览器打开URL。</li>
<li>-F：指定文档的风格，可选swagger和redoc。我选用了redoc，因为觉得redoc格式更加易读和清晰。</li>
<li>–port：指定启动的HTTP服务监听端口。</li>
</ul><p>打开浏览器，访问<a href="http://url">http://localhost:36666/docs</a> ，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/9a/2c/9a9fb7a31d418d8e4dc13b19cefa832c.png?wh=2524x699" alt=""></p><p>如果我们想要JSON格式的Swagger文档，可执行如下命令，将生成的swagger.yaml转换为swagger.json：</p><pre><code>$ swagger generate spec -i ./swagger.yaml -o ./swagger.json
</code></pre><p>接下来，我们就可以编写API接口的定义文件（位于<a href="https://github.com/marmotedu/gopractise-demo/blob/main/swagger/docs/user.go">gopractise-demo/swagger/docs/user.go</a>文件中）：</p><pre><code>package docs

import (
    &quot;github.com/marmotedu/gopractise-demo/swagger/api&quot;
)

// swagger:route POST /users user createUserRequest
// Create a user in memory.
// responses:
//   200: createUserResponse
//   default: errResponse

// swagger:route GET /users/{name} user getUserRequest
// Get a user from memory.
// responses:
//   200: getUserResponse
//   default: errResponse

// swagger:parameters createUserRequest
type userParamsWrapper struct {
    // This text will appear as description of your request body.
    // in:body
    Body api.User
}

// This text will appear as description of your request url path.
// swagger:parameters getUserRequest
type getUserParamsWrapper struct {
    // in:path
    Name string `json:&quot;name&quot;`
}

// This text will appear as description of your response body.
// swagger:response createUserResponse
type createUserResponseWrapper struct {
    // in:body
    Body api.User
}

// This text will appear as description of your response body.
// swagger:response getUserResponse
type getUserResponseWrapper struct {
    // in:body
    Body api.User
}

// This text will appear as description of your error response body.
// swagger:response errResponse
type errResponseWrapper struct {
    // Error code.
    Code int `json:&quot;code&quot;`

    // Error message.
    Message string `json:&quot;message&quot;`
}
</code></pre><p>user.go文件说明：</p><ul>
<li>swagger:route：<code>swagger:route</code>代表API接口描述的开始，后面的字符串格式为<code>HTTP方法 URL Tag ID</code>。可以填写多个tag，相同tag的API接口在Swagger文档中会被分为一组。ID是一个标识符，<code>swagger:parameters</code>是具有相同ID的<code>swagger:route</code>的请求参数。<code>swagger:route</code>下面的一行是该API接口的描述，需要以英文点号为结尾。<code>responses:</code>定义了API接口的返回参数，例如当HTTP状态码是200时，返回createUserResponse，createUserResponse会跟<code>swagger:response</code>进行匹配，匹配成功的<code>swagger:response</code>就是该API接口返回200状态码时的返回。</li>
<li>swagger:response：<code>swagger:response</code>定义了API接口的返回，例如getUserResponseWrapper，关于名字，我们可以根据需要自由命名，并不会带来任何不同。getUserResponseWrapper中有一个Body字段，其注释为<code>// in:body</code>，说明该参数是在HTTP Body中返回。<code>swagger:response</code>之上的注释会被解析为返回参数的描述。api.User自动被go-swagger解析为<code>Example Value</code>和<code>Model</code>。我们不用再去编写重复的返回字段，只需要引用已有的Go结构体即可，这也是通过工具生成Swagger文档的魅力所在。</li>
<li>swagger:parameters：<code>swagger:parameters</code>定义了API接口的请求参数，例如userParamsWrapper。userParamsWrapper之上的注释会被解析为请求参数的描述，<code>// in:body</code>代表该参数是位于HTTP Body中。同样，userParamsWrapper结构体名我们也可以随意命名，不会带来任何不同。<code>swagger:parameters</code>之后的createUserRequest会跟<code>swagger:route</code>的ID进行匹配，匹配成功则说明是该ID所在API接口的请求参数。</li>
</ul><p>进入gopractise-demo/swagger目录，执行如下命令，生成Swagger API文档，并启动HTTP服务，在浏览器查看Swagger：</p><pre><code>$ swagger generate spec -o swagger.yaml
$ swagger serve --no-open -F=swagger --port 36666 swagger.yaml
2020/10/20 23:28:30 serving docs at http://localhost:36666/docs
</code></pre><p>打开浏览器，访问 <a href="http://localhost:36666/docs">http://localhost:36666/docs</a> ，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/e6/e0/e6d6d138fb890ef219d71671d146d5e0.png?wh=2450x1213" alt=""></p><p>上面我们生成了swagger风格的UI界面，我们也可以使用redoc风格的UI界面，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/dd/48/dd568a44290283861ba5c37f28307d48.png?wh=2527x1426" alt=""></p><h3>go-swagger其他常用功能介绍</h3><p>上面，我介绍了swagger最常用的generate、serve命令，关于swagger其他有用的命令，这里也简单介绍一下。</p><ol>
<li>对比Swagger文档</li>
</ol><pre><code>$ swagger diff -d change.log swagger.new.yaml swagger.old.yaml
$ cat change.log

BREAKING CHANGES:
=================
/users:post Request - Body.Body.nickname.address.email.name.Body : User - Deleted property
compatibility test FAILED: 1 breaking changes detected
</code></pre><ol start="2">
<li>生成服务端代码</li>
</ol><p>我们也可以先定义Swagger接口文档，再用swagger命令，基于Swagger接口文档生成服务端代码。假设我们的应用名为go-user，进入gopractise-demo/swagger目录，创建go-user目录，并生成服务端代码：</p><pre><code>$ mkdir go-user
$ cd go-user
$ swagger generate server -f ../swagger.yaml -A go-user
</code></pre><p>上述命令会在当前目录生成cmd、restapi、models文件夹，可执行如下命令查看server组件启动方式：</p><pre><code>$ go run cmd/go-user-server/main.go -h
</code></pre><ol start="3">
<li>生成客户端代码</li>
</ol><p>在go-user目录下执行如下命令：</p><pre><code>$ swagger generate client -f ../swagger.yaml -A go-user
</code></pre><p>上述命令会在当前目录生成client，包含了API接口的调用函数，也就是API接口的Go SDK。</p><ol start="4">
<li>验证Swagger文档是否合法</li>
</ol><pre><code>$ swagger validate swagger.yaml
2020/10/21 09:53:18
The swagger spec at &quot;swagger.yaml&quot; is valid against swagger specification 2.0
</code></pre><ol start="5">
<li>合并Swagger文档</li>
</ol><pre><code>$ swagger mixin swagger_part1.yaml swagger_part2.yaml
</code></pre><h2>IAM Swagger文档</h2><p>IAM的Swagger文档定义在<a href="https://github.com/marmotedu/iam/tree/v1.0.0/api/swagger/docs">iam/api/swagger/docs</a>目录下，遵循go-swagger规范进行定义。</p><p><a href="https://github.com/marmotedu/iam/blob/v1.0.0/api/swagger/docs/doc.go">iam/api/swagger/docs/doc.go</a>文件定义了更多Swagger文档的基本信息，比如开源协议、联系方式、安全认证等。</p><p>更详细的定义，你可以直接查看iam/api/swagger/docs目录下的Go源码文件。</p><p>为了便于生成文档和启动HTTP服务查看Swagger文档，该操作被放在Makefile中执行（位于<a href="https://github.com/marmotedu/iam/blob/v1.0.0/scripts/make-rules/swagger.mk">iam/scripts/make-rules/swagger.mk</a>文件中）：</p><pre><code>.PHONY: swagger.run    
swagger.run: tools.verify.swagger    
  @echo &quot;===========&gt; Generating swagger API docs&quot;    
  @swagger generate spec --scan-models -w $(ROOT_DIR)/cmd/genswaggertypedocs -o $(ROOT_DIR)/api/swagger/swagger.yaml    
    
.PHONY: swagger.serve    
swagger.serve: tools.verify.swagger    
  @swagger serve -F=redoc --no-open --port 36666 $(ROOT_DIR)/api/swagger/swagger.yaml  
</code></pre><p>Makefile文件说明：</p><ul>
<li>tools.verify.swagger：检查Linux系统是否安装了go-swagger的命令行工具swagger，如果没有安装则运行go get安装。</li>
<li>swagger.run：运行 <code>swagger generate spec</code> 命令生成Swagger文档swagger.yaml，运行前会检查swagger是否安装。 <code>--scan-models</code> 指定生成的文档中包含带有swagger:model 注释的Go Models。<code>-w</code> 指定swagger命令运行的目录。</li>
<li>swagger.serve：运行 <code>swagger serve</code> 命令打开Swagger文档swagger.yaml，运行前会检查swagger是否安装。</li>
</ul><p>在iam源码根目录下执行如下命令，即可生成并启动HTTP服务查看Swagger文档：</p><pre><code>$ make swagger
$ make serve-swagger
2020/10/21 06:45:03 serving docs at http://localhost:36666/docs
</code></pre><p>打开浏览器，打开<code>http://x.x.x.x:36666/docs</code>查看Swagger文档，x.x.x.x是服务器的IP地址，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/6a/3b/6ac3529ed98aa94573862da99434683b.png?wh=2524x1440" alt=""></p><p>IAM的Swagger文档，还可以通过在iam源码根目录下执行<code>go generate ./...</code>命令生成，为此，我们需要在iam/cmd/genswaggertypedocs/swagger_type_docs.go文件中，添加<code>//go:generate</code>注释。如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/cc/d7/cc03b896e5403cc55d7e11fe2078d9d7.png?wh=1657x397" alt=""></p><h2>总结</h2><p>在做Go服务开发时，我们要向前端或用户提供API文档，手动编写API文档工作量大，也难以维护。所以，现在很多项目都是自动生成Swagger格式的API文档。提到Swagger，很多开发者不清楚其和OpenAPI的区别，所以我也给你总结了：OpenAPI是一个API规范，Swagger则是实现规范的工具。</p><p>在Go中，用得最多的是利用go-swagger来生成Swagger格式的API文档。go-swagger包含了很多语法，我们可以访问<a href="https://goswagger.io">Swagger 2.0</a>进行学习。学习完Swagger 2.0的语法之后，就可以编写swagger注释了，之后可以通过</p><pre><code>$ swagger generate spec -o swagger.yaml
</code></pre><p>来生成swagger文档 swagger.yaml。通过</p><pre><code>$ swagger serve --no-open -F=swagger --port 36666 swagger.yaml
</code></pre><p>来提供一个前端界面，供我们访问swagger文档。</p><p>为了方便管理，我们可以将 <code>swagger generate spec</code> 和 <code>swagger serve</code> 命令加入到Makefile文件中，通过Makefile来生成Swagger文档，并提供给前端界面。</p><h2>课后练习</h2><ol>
<li>尝试将你当前项目的一个API接口，用go-swagger生成swagger格式的API文档，如果中间遇到问题，欢迎在留言区与我讨论。</li>
<li>思考下，为什么IAM项目的swagger定义文档会放在iam/api/swagger/docs目录下，这样做有什么好处？</li>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/4b/46/717d5cb9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>惜心（伟祺）</span>
  </div>
  <div class="_2_QraFYR_0">老师厉害 把软件工程方方面面通过一个项目讲了<br>相信看完这个再去看大型开源项目 比如kubernates应该会容易很多 <br>聚焦在论文核心实现部分就可以<br>知道一个大项目是怎么造出来的 包括什么 再去学自然容易多了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-09 20:31:16</div>
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
  <div class="_2_QraFYR_0">一点想法，在小团队里，其实也没那么规范，能简单就简单点，目前用的是 Postman 生成的文档，考虑的点：<br>1. 一方面程序员自己调试接口必然会用到接口调用工具，postman大家也比较熟悉；<br>2. 另一方面开发完之后，它能直接生成在线文档，不用重新去定义文档使用的 API 的字段、结构体等，感觉会方便很多，而且能实时同步修改（接口有变动可以同步反映到文档上，不需要二次修改 api 文档，减少人为的错误）<br><br>请教下老师对两者的看法，主要是我觉得写 swagger 接口注释这个工作比较繁琐，另外需要特意去维护它，容易出现代码改了但是文档未改的情况；或者说老师这边有什么好的实践经验。<br><br>FYI：Makefile 说明那里有个 Models 写成了 Modles</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 文档怎么编写，这里其实没有一个统一的、标准的规范，因为不同团队对API文档的要求、对使用工具的选择都不一样。<br><br>这一讲介绍的方法，仅供参考。具体使用哪种，你们可以参考这一讲介绍的方法， 再结合自己的实际需求和情况，进行调研之后，再做选择。<br><br>如果你们觉得postman、或者一些在线工具能满足你们的需求，其实我觉得可能要不用swagger更好些，毕竟可视化、而且真的很方便、效率很高。<br><br>感谢反馈。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-05 07:54:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/28/75/30/c5b7b15d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>遇见@z</span>
  </div>
  <div class="_2_QraFYR_0">看着好麻烦，我们公司自己写的框架通过扫描ast语法树生成openapi，代码即文档</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 强，这个没有固定的方法，可以根据需要自己选择。这是个好方法，后面我看下怎么补下这种方式</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-08 21:34:40</div>
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
  <div class="_2_QraFYR_0">思考题可能的好处：1. 方便和项目根路径下的帮助文档目录docs做区分；2. 路径层次清晰辨认功能；3. 方便启api文档的http服务？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 说的很好。再补充下：放在api目录下，说明这个是api的定义文件。API文档聚合在一个目录下，后期维护，查看都很方便</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-03 18:06:44</div>
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
  <div class="_2_QraFYR_0">用go-swagger生成swagger格式的API文档。<br>API文档最大的一个痛点是更新不及时，使用自动生成工具可以降低及时更新文档的阻力，让文档实时更新变得更容易操作。使用更先进的工具，对抗人的惰性。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-10 22:21:19</div>
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
  <div class="_2_QraFYR_0">swagger文档可以手动编写，也可以通过工具解析注释生成文档。<br>说明关于swagger 文档产生了2中开发模式：<br>1. 手动编写swagger 文档，可以通过文档生成代码<br>2. 先编写代码，添加swagger规范的注释，生成文档。<br>请问一般开发中，使用哪种swagger模式？（我看kubernetes中的api doc不是通过代码生成的）</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 哪种都有，根据需要选择吧，如果需要并行开发，可以先编写swagger，否则可以根据喜好选择2.</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-31 21:46:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/91/b1/fb117c21.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>先听</span>
  </div>
  <div class="_2_QraFYR_0">swagger可以通过扫描源码生成文档。但是后端往往需要在编码之前就提供接口文档。貌似有些矛盾</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 所以这里，可以选择直接编写swagger格式的API接口文档。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-21 20:51:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_5baa01</span>
  </div>
  <div class="_2_QraFYR_0">需求确认了后,后台首先应该写swagger 提供给前端开发, 相互不影响,   按照你这个套路, 是不是得后端开发完了在让前端搞</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不是，要先写swagger文档。这里根据注解来生成swagger文档，不太建议用了。或者先把伪代码写出来。<br><br>重点结论：建议直接编写swagger文档。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-14 08:34:53</div>
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
  <div class="_2_QraFYR_0">&quot;目前最新的 OpenAPI 规范是OpenAPI 3.0（也就是 Swagger 2.0 规范）&quot;<br><br>这里错了啊, 现在最新的规范是2021年2月16日发布的 OpenAPI 3.1.0, 而且 OpenAPI 3.0 != Swagger 2.0<br><br>OpenAPI有三个版本 Swagger 2.0, OpenAPI 3.0.x, OpenAPI 3.1.0<br>详见:<br>https:&#47;&#47;swagger.io&#47;blog&#47;news&#47;whats-new-in-openapi-3-0&#47;<br>https:&#47;&#47;www.openapis.org&#47;blog&#47;2021&#47;02&#47;16&#47;migrating-from-openapi-3-0-to-3-1-0</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我联系小编更正下，感谢反馈~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-30 14:04:31</div>
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
  <div class="_2_QraFYR_0">swagger generate 命令会找到 main 函数，然后遍历所有的源代码，解析源码中与 Swagger 相关的注释，然后自动生成 swagger.json&#47;swagger.yaml 文件；<br><br>version: v0.29.0 不适用了吧？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我check下</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-06 17:49:29</div>
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
  <div class="_2_QraFYR_0">请问老师怎么看待“protoc-gen-openapiv2”这个工具，使用proto文件定义了接口出入参后可以直接生成swagger文档，不用再手动去编写。是不是比文中这种去手动编写的方便一点呢。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我比较喜欢手动编写吧。<br><br>openapi这类工具，能帮助自动生成文档、代码等。比较规范，但是编写文档的工作量跟手动编写没啥区别</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-05 17:33:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/08/a1/9166c1d8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Joker</span>
  </div>
  <div class="_2_QraFYR_0">请问老师，如何将swagger 里面的 静态资源更改为国内镜像，比如：<br>https:&#47;&#47;fonts.googleapis.com&#47;css</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-26 13:48:54</div>
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
  <div class="_2_QraFYR_0">总结：<br>1. 什么是 swagger? 它和OpenAPI之间是什么关系？如何编写swagger.yaml? <br>2. go-swagger 是一个非常流行的工具。你可以生成swagger.yaml，检查 swagger.yaml，生成客户端代码，生成服务端代码，启动HTTP服务器访问API文档<br>3. diff 命令可以检查两个yaml文件的内容，从而判断出，API文档是否引入了Break Change。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 66666</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-27 01:20:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/79/9b/66570110.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>土拨鼠</span>
  </div>
  <div class="_2_QraFYR_0">目前前后端就是通过swagger沟通，后端在开发之前先定义好swagger的 api.json，这样前后端就可以并行开发了，而且减少了很多扯皮</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-23 20:25:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/21/fb/2f/ae053a45.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>咖梵冰雨</span>
  </div>
  <div class="_2_QraFYR_0">这个demo运行会有跨域问题？这个本地咋解决，我gin加了中间件运行跨域好像不管用</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不会有跨域的问题。跨域：<br>浏览器访问的域名和浏览器发送请求的域名不一致，会触发浏览器的跨域请求。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-09 17:12:50</div>
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
  <div class="_2_QraFYR_0">通过swagger反向生成的代码后，运行swagger文档时，运行的是方向运行swagger文档，运行的是生成的项目<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-28 18:17:36</div>
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
  <div class="_2_QraFYR_0">文档版本比较多时（有的接口有3-4个版本），放在一个swagger文档下是否还需要加版本以区分呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 需要的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-06 13:09:55</div>
  </div>
</div>
</div>
</li>
</ul>