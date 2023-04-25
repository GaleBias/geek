<audio title="31 _ 数据流：通过iam-authz-server设计，看数据流服务的设计" src="https://static001.geekbang.org/resource/audio/15/54/1527955e262a2f78eb7f7a4629f92c54.mp3" controls="controls"></audio> 
<p>你好，我是孔令飞。</p><p>在 <a href="https://time.geekbang.org/column/article/401190">28讲</a> 和 <a href="https://time.geekbang.org/column/article/402206">29讲</a> ，我介绍了IAM的控制流服务iam-apiserver的设计和实现。这一讲，我们再来看下IAM数据流服务iam-authz-server的设计和实现。</p><p>因为iam-authz-server是数据流服务，对性能要求较高，所以采用了一些机制来最大化API接口的性能。另外，为了提高开发效率，避免重复造轮子，iam-authz-server和iam-apiserver共享了大部分的功能代码。接下来，我们就来看下，iam-authz-server是如何跟iam-apiserver共享代码的，以及iam-authz-server是如何保证API接口性能的。</p><h2>iam-authz-server的功能介绍</h2><p>iam-authz-server目前的唯一功能，是通过提供 <code>/v1/authz</code> RESTful API接口完成资源授权。 <code>/v1/authz</code> 接口是通过<a href="https://github.com/ory/ladon">github.com/ory/ladon</a>来完成资源授权的。</p><p>因为iam-authz-server承载了数据流的请求，需要确保API接口具有较高的性能。为了保证API接口的性能，iam-authz-server在设计上使用了大量的缓存技术。</p><!-- [[[read_end]]] --><h3>github.com/ory/ladon包介绍</h3><p>因为iam-authz-server资源授权是通过 <code>github.com/ory/ladon</code> 来完成的，为了让你更好地理解iam-authz-server的授权策略，在这里我先介绍下 <code>github.com/ory/ladon</code> 包。</p><p>Ladon是用Go语言编写的用于实现访问控制策略的库，类似于RBAC（基于角色的访问控制系统，Role Based Access Control）和ACL（访问控制列表，Access Control Lists）。但是与RBAC和ACL相比，Ladon可以实现更细粒度的访问控制，并且能够在更为复杂的环境中（例如多租户、分布式应用程序和大型组织）工作。</p><p>Ladon解决了这个问题：在特定的条件下，谁能够/不能够对哪些资源做哪些操作。为了解决这个问题，Ladon引入了授权策略。授权策略是一个有语法规范的文档，这个文档描述了谁在什么条件下能够对哪些资源做哪些操作。Ladon可以用请求的上下文，去匹配设置的授权策略，最终判断出当前授权请求是否通过。下面是一个Ladon的授权策略样例：</p><pre><code class="language-json">{
&nbsp; "description": "One policy to rule them all.",
&nbsp; "subjects": ["users:&lt;peter|ken&gt;", "users:maria", "groups:admins"],
&nbsp; "actions" : ["delete", "&lt;create|update&gt;"],
&nbsp; "effect": "allow",
&nbsp; "resources": [
&nbsp; &nbsp; "resources:articles:&lt;.*&gt;",
&nbsp; &nbsp; "resources:printer"
&nbsp; ],
&nbsp; "conditions": {
&nbsp; &nbsp; "remoteIP": {
&nbsp; &nbsp; &nbsp; &nbsp; "type": "CIDRCondition",
&nbsp; &nbsp; &nbsp; &nbsp; "options": {
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; "cidr": "192.168.0.1/16"
&nbsp; &nbsp; &nbsp; &nbsp; }
&nbsp; &nbsp; }
&nbsp; }
}
</code></pre><p>策略（Policy）由若干元素构成，用来描述授权的具体信息，你可以把它们看成一组规则。核心元素包括主题（Subject）、操作（Action）、效力（Effect）、资源（Resource）以及生效条件（Condition）。元素保留字仅支持小写，它们在描述上没有顺序要求。对于没有特定约束条件的策略，Condition元素是可选项。一条策略包含下面6个元素：</p><ul>
<li>主题（Subject），主题名是唯一的，代表一个授权主题。例如，“ken” or “printer-service.mydomain.com”。</li>
<li>操作（Action），描述允许或拒绝的操作。</li>
<li>效力（Effect），描述策略产生的结果是“允许”还是“拒绝”，包括 allow（允许）和 deny（拒绝）。</li>
<li>资源（Resource），描述授权的具体数据。</li>
<li>生效条件（Condition），描述策略生效的约束条件。</li>
<li>描述（Description），策略的描述。</li>
</ul><p>有了授权策略，我们就可以传入请求上下文，由Ladon来决定请求是否能通过授权。下面是一个请求示例：</p><pre><code class="language-json">{
&nbsp; "subject": "users:peter",
&nbsp; "action" : "delete",
&nbsp; "resource": "resources:articles:ladon-introduction",
&nbsp; "context": {
&nbsp; &nbsp; "remoteIP": "192.168.0.5"
&nbsp; }
}
</code></pre><p>可以看到，在 <code>remoteIP="192.168.0.5"</code> 生效条件（Condition）下，针对主题（Subject） <code>users:peter</code> 对资源（Resource） <code>resources:articles:ladon-introduction</code> 的 <code>delete</code> 操作（Action），授权策略的效力（Effect）是 <code>allow</code> 的。所以Ladon会返回如下结果：</p><pre><code class="language-json">{
&nbsp; &nbsp; "allowed": true
}
</code></pre><p>Ladon支持很多Condition，具体见下表：</p><p><img src="https://static001.geekbang.org/resource/image/b8/dd/b84d2a1dc0e9ac07605a867594d734dd.jpg?wh=1920x1521" alt="图片"></p><p>至于如何使用这些Condition，你可以参考 <a href="https://github.com/marmotedu/geekbang-go/blob/master/LadonCondition%E4%BD%BF%E7%94%A8%E7%A4%BA%E4%BE%8B.md">Ladon Condition使用示例</a>。此外，Ladon还支持自定义Condition。</p><p>另外，Ladon还支持授权审计，用来记录授权历史。我们可以通过在ladon.Ladon中附加一个ladon.AuditLogger来实现：</p><pre><code class="language-go">import "github.com/ory/ladon"
import manager "github.com/ory/ladon/manager/memory"

func main() {

&nbsp; &nbsp; warden := ladon.Ladon{
&nbsp; &nbsp; &nbsp; &nbsp; Manager: manager.NewMemoryManager(),
&nbsp; &nbsp; &nbsp; &nbsp; AuditLogger: &amp;ladon.AuditLoggerInfo{}
&nbsp; &nbsp; }

&nbsp; &nbsp; // ...
}
</code></pre><p>在上面的示例中，我们提供了ladon.AuditLoggerInfo，该AuditLogger会在授权时打印调用的策略到标准错误。AuditLogger是一个interface：</p><pre><code class="language-go">// AuditLogger tracks denied and granted authorizations.
type AuditLogger interface {
&nbsp; &nbsp; LogRejectedAccessRequest(request *Request, pool Policies, deciders Policies)
&nbsp; &nbsp; LogGrantedAccessRequest(request *Request, pool Policies, deciders Policies)
}
</code></pre><p>要实现一个新的AuditLogger，你只需要实现AuditLogger接口就可以了。比如，我们可以实现一个AuditLogger，将授权日志保存到Redis或者MySQL中。</p><p>Ladon支持跟踪一些授权指标，比如 deny、allow、not match、error。你可以通过实现ladon.Metric接口，来对这些指标进行处理。ladon.Metric接口定义如下：</p><pre><code class="language-go">// Metric is used to expose metrics about authz
type Metric interface {
&nbsp; &nbsp; // RequestDeniedBy is called when we get explicit deny by policy
&nbsp; &nbsp; RequestDeniedBy(Request, Policy)
&nbsp; &nbsp; // RequestAllowedBy is called when a matching policy has been found.
&nbsp; &nbsp; RequestAllowedBy(Request, Policies)
&nbsp; &nbsp; // RequestNoMatch is called when no policy has matched our request
&nbsp; &nbsp; RequestNoMatch(Request)
&nbsp; &nbsp; // RequestProcessingError is called when unexpected error occured
&nbsp; &nbsp; RequestProcessingError(Request, Policy, error)
}
</code></pre><p>例如，你可以通过下面的示例，将这些指标暴露给prometheus：</p><pre><code class="language-go">type prometheusMetrics struct{}

func (mtr *prometheusMetrics) RequestDeniedBy(r ladon.Request, p ladon.Policy) {}
func (mtr *prometheusMetrics) RequestAllowedBy(r ladon.Request, policies ladon.Policies) {}
func (mtr *prometheusMetrics) RequestNoMatch(r ladon.Request) {}
func (mtr *prometheusMetrics) RequestProcessingError(r ladon.Request, err error) {}

func main() {

&nbsp; &nbsp; warden := ladon.Ladon{
&nbsp; &nbsp; &nbsp; &nbsp; Manager: manager.NewMemoryManager(),
&nbsp; &nbsp; &nbsp; &nbsp; Metric:&nbsp; &amp;prometheusMetrics{},
&nbsp; &nbsp; }

&nbsp; &nbsp; // ...
}
</code></pre><p>在使用Ladon的过程中，有两个地方需要你注意：</p><ul>
<li>所有检查都区分大小写，因为主题值可能是区分大小写的ID。</li>
<li>如果ladon.Ladon无法将策略与请求匹配，会默认授权结果为拒绝，并返回错误。</li>
</ul><h3>iam-authz-server使用方法介绍</h3><p>上面，我介绍了iam-authz-server的资源授权功能，这里介绍下如何使用iam-authz-server，也就是如何调用 <code>/v1/authz</code> 接口完成资源授权。你可以通过下面的3大步骤，来完成资源授权请求。</p><p><strong>第一步，登陆iam-<strong><strong>a</strong></strong>p<strong><strong>i</strong></strong>s<strong><strong>e</strong></strong>r<strong><strong>v</strong></strong>er，创建授权策略和密钥。</strong></p><p>这一步又分为3个小步骤。</p><ol>
<li>登陆iam-apiserver系统，获取访问令牌：</li>
</ol><pre><code class="language-shell">$ token=`curl -s -XPOST -H'Content-Type: application/json' -d'{"username":"admin","password":"Admin@2021"}' http://127.0.0.1:8080/login | jq -r .token`
</code></pre><ol start="2">
<li>创建授权策略：</li>
</ol><pre><code class="language-shell">$ curl -s -XPOST -H"Content-Type: application/json" -H"Authorization: Bearer $token" -d'{"metadata":{"name":"authztest"},"policy":{"description":"One policy to rule them all.","subjects":["users:&lt;peter|ken&gt;","users:maria","groups:admins"],"actions":["delete","&lt;create|update&gt;"],"effect":"allow","resources":["resources:articles:&lt;.*&gt;","resources:printer"],"conditions":{"remoteIP":{"type":"CIDRCondition","options":{"cidr":"192.168.0.1/16"}}}}}' http://127.0.0.1:8080/v1/policies
</code></pre><ol start="3">
<li>创建密钥，并从请求结果中提取secretID 和 secretKey：</li>
</ol><pre><code class="language-shell">$ curl -s -XPOST -H"Content-Type: application/json" -H"Authorization: Bearer $token" -d'{"metadata":{"name":"authztest"},"expires":0,"description":"admin secret"}' http://127.0.0.1:8080/v1/secrets
{"metadata":{"id":23,"name":"authztest","createdAt":"2021-04-08T07:24:50.071671422+08:00","updatedAt":"2021-04-08T07:24:50.071671422+08:00"},"username":"admin","secretID":"ZuxvXNfG08BdEMqkTaP41L2DLArlE6Jpqoox","secretKey":"7Sfa5EfAPIwcTLGCfSvqLf0zZGCjF3l8","expires":0,"description":"admin secret"}
</code></pre><p><strong>第二步，生成访问 iam-authz-server的 token。</strong></p><p>iamctl 提供了 <code>jwt sigin</code> 子命令，可以根据 secretID 和 secretKey 签发 Token，方便使用。</p><pre><code class="language-shell">$ iamctl jwt sign ZuxvXNfG08BdEMqkTaP41L2DLArlE6Jpqoox 7Sfa5EfAPIwcTLGCfSvqLf0zZGCjF3l8 # iamctl jwt sign $secretID $secretKey
eyJhbGciOiJIUzI1NiIsImtpZCI6Ilp1eHZYTmZHMDhCZEVNcWtUYVA0MUwyRExBcmxFNkpwcW9veCIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXV0aHoubWFybW90ZWR1LmNvbSIsImV4cCI6MTYxNzg0NTE5NSwiaWF0IjoxNjE3ODM3OTk1LCJpc3MiOiJpYW1jdGwiLCJuYmYiOjE2MTc4Mzc5OTV9.za9yLM7lHVabPAlVQLCqXEaf8sTU6sodAsMXnmpXjMQ
</code></pre><p>你可以通过 <code>iamctl jwt show &lt;token&gt;</code> 来查看Token的内容：</p><pre><code class="language-shell">$ iamctl jwt show eyJhbGciOiJIUzI1NiIsImtpZCI6Ilp1eHZYTmZHMDhCZEVNcWtUYVA0MUwyRExBcmxFNkpwcW9veCIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXV0aHoubWFybW90ZWR1LmNvbSIsImV4cCI6MTYxNzg0NTE5NSwiaWF0IjoxNjE3ODM3OTk1LCJpc3MiOiJpYW1jdGwiLCJuYmYiOjE2MTc4Mzc5OTV9.za9yLM7lHVabPAlVQLCqXEaf8sTU6sodAsMXnmpXjMQ
Header:
{
&nbsp; &nbsp; "alg": "HS256",
&nbsp; &nbsp; "kid": "ZuxvXNfG08BdEMqkTaP41L2DLArlE6Jpqoox",
&nbsp; &nbsp; "typ": "JWT"
}
Claims:
{
&nbsp; &nbsp; "aud": "iam.authz.marmotedu.com",
&nbsp; &nbsp; "exp": 1617845195,
&nbsp; &nbsp; "iat": 1617837995,
&nbsp; &nbsp; "iss": "iamctl",
&nbsp; &nbsp; "nbf": 1617837995
}
</code></pre><p>我们生成的Token包含了下面这些信息。</p><p><strong>Header</strong></p><ul>
<li>alg：生成签名的算法。</li>
<li>kid：密钥ID。</li>
<li>typ：Token的类型，这里是JWT。</li>
</ul><p><strong>Claims</strong></p><ul>
<li>aud：JWT Token的接受者。</li>
<li>exp：JWT Token的过期时间（UNIX时间格式）。</li>
<li>iat：JWT Token的签发时间（UNIX时间格式）。</li>
<li>iss：签发者，因为我们是用 iamctl 工具签发的，所以这里的签发者是 iamctl。</li>
<li>nbf：JWT Token的生效时间（UNIX时间格式），默认是签发时间。</li>
</ul><p><strong>第三步，调用</strong><code>/v1/authz</code><strong>接口</strong><strong>，</strong><strong>完成资源授权请求。</strong></p><p>请求方法如下：</p><pre><code class="language-shell">$ curl -s -XPOST -H'Content-Type: application/json' -H'Authorization: Bearer eyJhbGciOiJIUzI1NiIsImtpZCI6Ilp1eHZYTmZHMDhCZEVNcWtUYVA0MUwyRExBcmxFNkpwcW9veCIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXV0aHoubWFybW90ZWR1LmNvbSIsImV4cCI6MTYxNzg0NTE5NSwiaWF0IjoxNjE3ODM3OTk1LCJpc3MiOiJpYW1jdGwiLCJuYmYiOjE2MTc4Mzc5OTV9.za9yLM7lHVabPAlVQLCqXEaf8sTU6sodAsMXnmpXjMQ' -d'{"subject":"users:maria","action":"delete","resource":"resources:articles:ladon-introduction","context":{"remoteIP":"192.168.0.5"}}' http://127.0.0.1:9090/v1/authz
{"allowed":true}
</code></pre><p>如果授权通过，会返回：<code>{"allowed":true}</code> 。 如果授权失败，则返回：</p><pre><code class="language-shell">{"allowed":false,"denied":true,"reason":"Request was denied by default"}
</code></pre><h2>iam-authz-server的代码实现</h2><p>接下来，我们来看下iam-authz-server的具体实现，我会从配置处理、启动流程、请求处理流程和代码架构4个方面来讲解。</p><h3>iam-authz-server的配置处理</h3><p>iam-authz-server服务的main函数位于<a href="https://github.com/marmotedu/iam/blob/v1.0.4/cmd/iam-authz-server/authzserver.go">authzserver.go</a>文件中，你可以跟读代码，了解iam-authz-server的代码实现。iam-authz-server的服务框架设计跟iam-apiserver的服务框架设计保持一致，也是有3种配置：Options配置、组件配置和HTTP服务配置。</p><p>Options配置见<a href="https://github.com/marmotedu/iam/blob/v1.0.4/internal/authzserver/options/options.go">options.go</a>文件：</p><pre><code class="language-go">type Options struct {
&nbsp; &nbsp; RPCServer&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;string
&nbsp; &nbsp; ClientCA&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; string
&nbsp; &nbsp; GenericServerRunOptions *genericoptions.ServerRunOptions
&nbsp; &nbsp; InsecureServing&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;*genericoptions.InsecureServingOptions
&nbsp; &nbsp; SecureServing&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;*genericoptions.SecureServingOptions
&nbsp; &nbsp; RedisOptions&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; *genericoptions.RedisOptions
&nbsp; &nbsp; FeatureOptions&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; *genericoptions.FeatureOptions
&nbsp; &nbsp; Log&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;*log.Options
&nbsp; &nbsp; AnalyticsOptions&nbsp; &nbsp; &nbsp; &nbsp; *analytics.AnalyticsOptions
}
</code></pre><p>和iam-apiserver相比，iam-authz-server多了 <code>AnalyticsOptions</code>，用来配置iam-authz-server内的Analytics服务，Analytics服务会将授权日志异步写入到Redis中。</p><p>iam-apiserver和iam-authz-server共用了GenericServerRunOptions、InsecureServing、SecureServing、FeatureOptions、RedisOptions、Log这些配置。所以，我们只需要用简单的几行代码，就可以将很多配置项都引入到iam-authz-server的命令行参数中，这也是命令行参数分组带来的好处：批量共享。</p><h3>iam-authz-server启动流程设计</h3><p>接下来，我们来详细看下iam-authz-server的启动流程。</p><p>iam-authz-server的启动流程也和iam-apiserver基本保持一致。二者比较大的不同在于Options参数配置和应用初始化内容。另外，和iam-apiserver相比，iam-authz-server只提供了REST API服务。启动流程如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/19/35/195178d37854bac7d5243d80e42a4c35.jpg?wh=2248x799" alt=""></p><h3>iam-authz-server 的 RESTful API请求处理流程</h3><p>iam-authz-server的请求处理流程也是清晰、规范的，具体流程如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/5a/89/5a83384f5762c41831190628bfa60989.jpg?wh=2248x780" alt=""></p><p><strong>首先，</strong>我们通过API调用（<code>&lt;HTTP Method&gt; + &lt;HTTP Request Path&gt;</code>）请求iam-authz-server提供的RESTful API接口 <code>POST /v1/authz</code> 。</p><p><strong>接着，</strong>Gin Web框架接收到HTTP请求之后，会通过认证中间件完成请求的认证，iam-authz-server采用了Bearer认证方式。</p><p><strong>然后，</strong>请求会被我们加载的一系列中间件所处理，例如跨域、RequestID、Dump等中间件。</p><p><strong>最后，</strong>根据<code>&lt;HTTP Method&gt; + &lt;HTTP Request Path&gt;</code>进行路由匹配。</p><p>比如，我们请求的RESTful API是<code>POST /v1/authz</code>，Gin Web框架会根据 HTTP Method 和 HTTP Request Path，查找注册的Controllers，最终匹配到 <a href="https://github.com/marmotedu/iam/blob/v1.0.4/internal/authzserver/controller/v1/authorize/authorize.go#L33">authzController.Authorize</a> Controller。在 Authorize Controller中，会先解析请求参数，接着校验请求参数、调用业务层的方法进行资源授权，最后处理业务层的返回结果，返回最终的 HTTP 请求结果。</p><h3>iam-authz-server的代码架构</h3><p>iam-authz-server的代码设计和iam-apiserver一样，遵循简洁架构设计。</p><p>iam-authz-server的代码架构也分为4层，分别是模型层（Models）、控制层（Controller）、业务层 （Service）和仓库层（Repository）。从控制层、业务层到仓库层，从左到右层级依次加深。模型层独立于其他层，可供其他层引用。如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/a5/dd/a57832495c9e031a94282f0a8a3a61dd.jpg?wh=2248x702" alt=""></p><p>iam-authz-server 和 iam-apiserver 的代码架构有这三点不同：</p><ul>
<li>iam-authz-server客户端不支持前端和命令行。</li>
<li>iam-authz-server仓库层对接的是iam-apiserver微服务，而非数据库。</li>
<li>iam-authz-server业务层的代码存放在目录<a href="https://github.com/marmotedu/iam/tree/v1.0.4/internal/authzserver/authorization">authorization</a>中。</li>
</ul><h2>iam-authz-server关键代码分析</h2><p>和 iam-apiserver 一样，iam-authz-server也包含了一些优秀的设计思路和关键代码，这里我来一一介绍下。</p><h3>资源授权</h3><p>先来看下，iam-authz-server是如何实现资源授权的。</p><p>我们可以调用iam-authz-server的 <code>/v1/authz</code>  API接口，实现资源的访问授权。 <code>/v1/authz</code> 对应的controller方法是<a href="https://github.com/marmotedu/iam/blob/v1.0.4/internal/authzserver/controller/v1/authorize/authorize.go#L33">Authorize</a>：</p><pre><code class="language-go">func (a *AuthzController) Authorize(c *gin.Context) {
	var r ladon.Request
	if err := c.ShouldBind(&amp;r); err != nil {
		core.WriteResponse(c, errors.WithCode(code.ErrBind, err.Error()), nil)

		return
	}

	auth := authorization.NewAuthorizer(authorizer.NewAuthorization(a.store))
	if r.Context == nil {
		r.Context = ladon.Context{}
	}

	r.Context["username"] = c.GetString("username")
	rsp := auth.Authorize(&amp;r)

	core.WriteResponse(c, nil, rsp)
}
</code></pre><p>该函数使用 <code>github.com/ory/ladon</code> 包进行资源访问授权，授权流程如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/7c/a6/7c251c61cb535714edd390eac18df8a6.jpg?wh=2248x920" alt=""></p><p>具体分为以下几个步骤：</p><p>第一步，在Authorize方法中调用 <code>c.ShouldBind(&amp;r)</code> ，将API请求参数解析到 <code>ladon.Request</code> 类型的结构体变量中。</p><p>第二步，调用<a href="https://github.com/marmotedu/iam/blob/v1.0.4/internal/authzserver/authorization/authorizer.go#L21">authorization.NewAuthorizer</a>函数，该函数会创建并返回包含Manager和AuditLogger字段的<a href="https://github.com/marmotedu/iam/blob/v1.0.4/internal/authzserver/authorization/authorizer.go#L16">Authorizer</a>类型的变量。</p><p>Manager包含一些函数，比如 Create、Update和FindRequestCandidates等，用来对授权策略进行增删改查。AuditLogger包含 LogRejectedAccessRequest 和 LogGrantedAccessRequest 函数，分别用来记录被拒绝的授权请求和被允许的授权请求，将其作为审计数据使用。</p><p>第三步，调用<a href="https://github.com/marmotedu/iam/blob/v1.0.4/internal/authzserver/authorization/authorizer.go#L31">auth.Authorize</a>函数，对请求进行访问授权。auth.Authorize函数内容如下：</p><pre><code class="language-go">func (a *Authorizer) Authorize(request *ladon.Request) *authzv1.Response {
	log.Debug("authorize request", log.Any("request", request))

	if err := a.warden.IsAllowed(request); err != nil {
		return &amp;authzv1.Response{
			Denied: true,
			Reason: err.Error(),
		}
	}

	return &amp;authzv1.Response{
		Allowed: true,
	}
}
</code></pre><p>该函数会调用 <code>a.warden.IsAllowed(request)</code> 完成资源访问授权。IsAllowed函数会调用 <code>FindRequestCandidates(r)</code> 查询所有的策略列表，这里要注意，我们只需要查询请求用户的policy列表。在Authorize函数中，我们将username存入ladon Request的context中：</p><pre><code class="language-go">r.Context["username"] = c.GetHeader("username")
</code></pre><p>在<a href="https://github.com/marmotedu/iam/blob/v1.0.4/internal/authzserver/authorization/manager.go#L54">FindRequestCandidates</a>函数中，我们可以从Request中取出username，并根据username查询缓存中的policy列表，FindRequestCandidates实现如下：</p><pre><code class="language-go">func (m *PolicyManager) FindRequestCandidates(r *ladon.Request) (ladon.Policies, error) {
		username := ""
	
		if user, ok := r.Context["username"].(string); ok {
			username = user
		}
	
		policies, err := m.client.List(username)
		if err != nil {
			return nil, errors.Wrap(err, "list policies failed")
		}
	
		ret := make([]ladon.Policy, 0, len(policies))
		for _, policy := range policies {
			ret = append(ret, policy)
		}
	
		return ret, nil
	}
</code></pre><p>IsAllowed函数代码如下：</p><pre><code class="language-go">func (l *Ladon) IsAllowed(r *Request) (err error) {
&nbsp; &nbsp; policies, err := l.Manager.FindRequestCandidates(r)
&nbsp; &nbsp; if err != nil {
&nbsp; &nbsp; &nbsp; &nbsp; go l.metric().RequestProcessingError(*r, nil, err)
&nbsp; &nbsp; &nbsp; &nbsp; return err
&nbsp; &nbsp; }

&nbsp; &nbsp; return l.DoPoliciesAllow(r, policies)
}
</code></pre><p>IsAllowed会调用 <code>DoPoliciesAllow(r, policies)</code> 函数进行权限校验。如果权限校验不通过（请求在指定条件下不能够对资源做指定操作），就调用 <code>LogRejectedAccessRequest</code> 函数记录拒绝的请求，并返回值为非nil的error，error中记录了授权失败的错误信息。如果权限校验通过，则调用 <code>LogGrantedAccessRequest</code> 函数记录允许的请求，并返回值为nil的error。</p><p>为了降低请求延时，LogRejectedAccessRequest和LogGrantedAccessRequest会将授权记录存储在Redis中，之后由iam-pump进程读取Redis，并将授权记录持久化存储在MongoDB中。</p><h3>缓存设计</h3><p>iam-authz-server主要用来做资源访问授权，属于数据流的组件，对接口访问性能有比较高的要求，所以该组件采用了缓存的机制。如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/05/51/05d1c9a9acdc451f915684c18c8b9f51.jpg?wh=2248x822" alt=""></p><p>iam-authz-server组件通过<strong>缓存密钥和授权策略信息</strong>到内存中，加快密钥和授权策略的查询速度。通过<strong>缓存授权记录</strong>到内存中，提高了授权数据的写入速度，从而大大降低了授权请求接口的延时。</p><p>上面的缓存机制用到了Redis key-value存储，所以在iam-authz-server初始化阶段，需要先建立Redis连接（位于<a href="https://github.com/marmotedu/iam/blob/v1.0.5/internal/authzserver/server.go#L132">initialize</a>函数中）：</p><pre><code class="language-go">go storage.ConnectToRedis(ctx, s.buildStorageConfig())
</code></pre><p>这个代码会维护一个Redis连接，如果Redis连接断掉，会尝试重连。这种方式可以使我们在调用Redis接口进行数据读写时，不用考虑连接断开的问题。</p><p>接下来，我们就来详细看看，iam-authz-server是如何实现缓存机制的。</p><p><strong>先来看下密钥和策略缓存。</strong></p><p>iam-authz-server通过<a href="https://github.com/marmotedu/iam/tree/v1.0.5/internal/authzserver/load">load</a>包来完成密钥和策略的缓存。</p><p>在iam-authz-server进程启动时，会创建并启动一个Load服务（位于<a href="https://github.com/marmotedu/iam/blob/v1.0.5/internal/authzserver/server.go#L144">initialize</a>函数中）：</p><pre><code class="language-go">load.NewLoader(ctx, cacheIns).Start()&nbsp;
</code></pre><p><strong>先来看创建Load服务。</strong>创建Load服务时，传入了cacheIns参数，cacheIns是一个实现了<a href="https://github.com/marmotedu/iam/blob/v1.0.5/internal/authzserver/load/load.go#L16">Loader</a>接口的实例：</p><pre><code class="language-go">type Loader interface {
&nbsp; &nbsp; Reload() error
}
</code></pre><p><strong>然后看启动Load服务。</strong>通过Load实例的 <a href="https://github.com/marmotedu/iam/blob/v1.0.5/internal/authzserver/load/load.go#L37">Start</a> 方法来启动Load服务：</p><pre><code class="language-go">func (l *Load) Start() {
&nbsp; &nbsp; go startPubSubLoop()
&nbsp; &nbsp; go l.reloadQueueLoop()
&nbsp; &nbsp; go l.reloadLoop()

&nbsp; &nbsp; l.DoReload()
}
</code></pre><p>Start函数先启动了3个协程，再调用 <code>l.DoReload()</code> 完成一次密钥和策略的同步：</p><pre><code class="language-go">func (l *Load) DoReload() {
&nbsp; &nbsp; l.lock.Lock()
&nbsp; &nbsp; defer l.lock.Unlock()

&nbsp; &nbsp; if err := l.loader.Reload(); err != nil {
&nbsp; &nbsp; &nbsp; &nbsp; log.Errorf("faild to refresh target storage: %s", err.Error())
&nbsp; &nbsp; }

&nbsp; &nbsp; log.Debug("refresh target storage succ")
}
</code></pre><p>上面我们说了，创建Load服务时，传入的cacheIns实例是一个实现了Loader接口的实例，所以在<a href="https://github.com/marmotedu/iam/blob/v1.0.5/internal/authzserver/load/load.go#L119">DoReload</a>方法中，可以直接调用Reload方法。cacheIns的Reload方法会从iam-apiserver中同步密钥和策略信息到iam-authz-server缓存中。</p><p>我们再来看下，startPubSubLoop、reloadQueueLoop、reloadLoop 这3个Go协程分别完成了什么功能。</p><ol>
<li>startPubSubLoop协程</li>
</ol><p><a href="https://github.com/marmotedu/iam/blob/v1.0.5/internal/authzserver/load/redis_signals.go#L46">startPubSubLoop</a>函数通过<a href="https://github.com/marmotedu/iam/blob/v1.0.5/pkg/storage/redis_cluster.go#L897">StartPubSubHandler</a>函数，订阅Redis的 <code>iam.cluster.notifications</code> channel，并注册一个回调函数：</p><pre><code class="language-go">func(v interface{}) {
&nbsp; &nbsp; handleRedisEvent(v, nil, nil)
}
</code></pre><p><a href="https://github.com/marmotedu/iam/blob/v1.0.5/internal/authzserver/load/redis_signals.go#L65">handleRedisEvent</a>函数中，会将消息解析为<a href="https://github.com/marmotedu/iam/blob/v1.0.5/internal/authzserver/load/redis_signals.go#L32">Notification</a>类型的消息，并判断Command的值。如果是NoticePolicyChanged或NoticeSecretChanged，就会向 <code>reloadQueue</code> channel中写入一个回调函数。因为我们不需要用回调函数做任何事情，所以这里回调函数是nil。 <code>reloadQueue</code> 主要用来告诉程序，需要完成一次密钥和策略的同步。</p><ol start="2">
<li>reloadQueueLoop协程</li>
</ol><p>reloadQueueLoop函数会监听 <code>reloadQueue</code> ，当发现有新的消息（这里是回调函数）写入时，会实时将消息缓存到 <code>requeue</code> 切片中，代码如下：</p><pre><code class="language-go">func (l *Load) reloadQueueLoop(cb ...func()) {
		for {
			select {
			case &lt;-l.ctx.Done():
				return
			case fn := &lt;-reloadQueue:
				requeueLock.Lock()
				requeue = append(requeue, fn)
				requeueLock.Unlock()
				log.Info("Reload queued")
				if len(cb) != 0 {
					cb[0]()
				}
			}
		}
	}
</code></pre><ol start="3">
<li>reloadLoop协程</li>
</ol><p>通过<a href="https://github.com/marmotedu/iam/blob/v1.0.5/internal/authzserver/load/load.go#L81">reloadLoop</a>函数启动一个timer定时器，每隔1秒会检查 <code>requeue</code> 切片是否为空，如果不为空，则调用 <code>l.DoReload</code> 方法，从iam-apiserver中拉取密钥和策略，并缓存在内存中。</p><p>密钥和策略的缓存模型如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/a2/11/a2f5694e5d6291ca610b84ee49469211.jpg?wh=2248x890" alt=""></p><p><strong>密钥和策略缓存的具体流程如下：</strong></p><p>接收上游消息（这里是从Redis中接收），将消息缓存到切片或者带缓冲的channel中，并启动一个消费协程去消费这些消息。这里的消费协程是reloadLoop，reloadLoop会每隔1s判断 <code>requeue</code> 切片是否长度为0，如果不为0，则执行 <code>l.DoReload()</code> 缓存密钥和策略。</p><p>讲完了密钥和策略缓存，<strong>再<strong><strong>来</strong></strong>看下授权日志缓存。</strong></p><p>在启动iam-authz-server时，还会启动一个Analytics服务，代码如下（位于<a href="https://github.com/marmotedu/iam/blob/v1.0.6/internal/authzserver/server.go#L147-L156">internal/authzserver/server.go</a>文件中）：</p><pre><code class="language-go">&nbsp; &nbsp; if s.analyticsOptions.Enable {&nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; &nbsp; &nbsp; analyticsStore := storage.RedisCluster{KeyPrefix: RedisKeyPrefix}&nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; &nbsp; &nbsp; analyticsIns := analytics.NewAnalytics(s.analyticsOptions, &amp;analyticsStore)&nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; &nbsp; &nbsp; analyticsIns.Start()&nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; &nbsp; &nbsp; s.gs.AddShutdownCallback(shutdown.ShutdownFunc(func(string) error {&nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; analyticsIns.Stop()&nbsp; &nbsp;&nbsp;
&nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; return nil&nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; &nbsp; &nbsp; }))&nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; }
</code></pre><p><a href="https://github.com/marmotedu/iam/blob/v1.0.6/internal/authzserver/analytics/analytics.go#L64-L79">NewAnalytics</a>函数会根据配置，创建一个Analytics实例：</p><pre><code class="language-go">func NewAnalytics(options *AnalyticsOptions, store storage.AnalyticsHandler) *Analytics {
		ps := options.PoolSize
		recordsBufferSize := options.RecordsBufferSize
		workerBufferSize := recordsBufferSize / uint64(ps)
		log.Debug("Analytics pool worker buffer size", log.Uint64("workerBufferSize", workerBufferSize))
	
		recordsChan := make(chan *AnalyticsRecord, recordsBufferSize)
	
		return &amp;Analytics{
			store:                      store,
			poolSize:                   ps,
			recordsChan:                recordsChan,
			workerBufferSize:           workerBufferSize,
			recordsBufferFlushInterval: options.FlushInterval,
		}
	}&nbsp;
</code></pre><p>上面的代码创建了一个带缓冲的 <code>recordsChan</code> ：</p><pre><code class="language-go">recordsChan := make(chan *AnalyticsRecord, recordsBufferSize)
</code></pre><p><code>recordsChan</code> 存放的数据类型为<a href="https://github.com/marmotedu/iam/blob/v1.0.6/internal/authzserver/analytics/analytics.go#L26-L35">AnalyticsRecord</a>，缓冲区的大小为 <code>recordsBufferSize</code> （通过 <code>--analytics.records-buffer-size</code> 选项指定）。可以通过<a href="https://github.com/marmotedu/iam/blob/v1.0.6/internal/authzserver/analytics/analytics.go#L115-L126">RecordHit</a>函数，向<code>recordsChan</code> 中写入 AnalyticsRecord 类型的数据：</p><pre><code class="language-go">func (r *Analytics) RecordHit(record *AnalyticsRecord) error {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
&nbsp; &nbsp; // check if we should stop sending records 1st&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
&nbsp; &nbsp; if atomic.LoadUint32(&amp;r.shouldStop) &gt; 0 {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; &nbsp; &nbsp; return nil&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
&nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
&nbsp; &nbsp; // just send record to channel consumed by pool of workers&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
&nbsp; &nbsp; // leave all data crunching and Redis I/O work for pool workers&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; r.recordsChan &lt;- record&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
&nbsp; &nbsp; return nil&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
}&nbsp; &nbsp;
</code></pre><p>iam-authz-server是通过调用 LogGrantedAccessRequest 和 LogRejectedAccessRequest 函数来记录授权日志的。在记录授权日志时，会将授权日志写入 <code>recordsChan</code>  channel中。<a href="https://github.com/marmotedu/iam/blob/v1.0.6/internal/authzserver/authorization/authorizer/authorizer.go#L100-L115">LogGrantedAccessRequest</a>函数代码如下：</p><p></p><pre><code class="language-go">func (auth *Authorization) LogGrantedAccessRequest(r *ladon.Request, p ladon.Policies, d ladon.Policies) {
&nbsp; &nbsp; conclusion := fmt.Sprintf("policies %s allow access", joinPoliciesNames(d))&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
&nbsp; &nbsp; rstring, pstring, dstring := convertToString(r, p, d)&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; record := analytics.AnalyticsRecord{&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
&nbsp; &nbsp; &nbsp; &nbsp; TimeStamp:&nbsp; time.Now().Unix(),&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; &nbsp; &nbsp; Username:&nbsp; &nbsp;r.Context["username"].(string),&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
&nbsp; &nbsp; &nbsp; &nbsp; Effect:&nbsp; &nbsp; &nbsp;ladon.AllowAccess,&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
&nbsp; &nbsp; &nbsp; &nbsp; Conclusion: conclusion,&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; &nbsp; &nbsp; Request:&nbsp; &nbsp; rstring,&nbsp; &nbsp; &nbsp; &nbsp;
&nbsp; &nbsp; &nbsp; &nbsp; Policies:&nbsp; &nbsp;pstring,&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
&nbsp; &nbsp; &nbsp; &nbsp; Deciders:&nbsp; &nbsp;dstring,&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
&nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
&nbsp; &nbsp; record.SetExpiry(0)
&nbsp; &nbsp; _ = analytics.GetAnalytics().RecordHit(&amp;record)&nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
}&nbsp;
</code></pre><p>上面的代码，会创建AnalyticsRecord类型的结构体变量，并调用RecordHit将变量的值写入 <code>recordsChan</code>  channel中。将授权日志写入 <code>recordsChan</code> &nbsp; channel中，而不是直接写入Redis中，这可以大大减少写入延时，减少接口的响应延时。</p><p>还有一个worker进程从recordsChan中读取数据，并在数据达到一定阈值之后，批量写入Redis中。在<a href="https://github.com/marmotedu/iam/blob/v1.0.6/internal/authzserver/analytics/analytics.go#L87-L100">Start</a>函数中，我们创建了一批worker，worker个数可以通过 <code>--analytics.pool-size</code> 来指定 。Start函数内容如下：</p><pre><code class="language-go">func (r *Analytics) Start() {
		analytics = r
		r.store.Connect()
	
		// start worker pool
		atomic.SwapUint32(&amp;r.shouldStop, 0)
		for i := 0; i &lt; r.poolSize; i++ {
			r.poolWg.Add(1)
			go r.recordWorker()
		}
	
		// stop analytics workers
		go r.Stop()
	}
</code></pre><p>上面的代码通过 <code>go r.recordWorker()</code> 创建了 由<code>poolSize</code> 指定个数的<a href="https://github.com/marmotedu/iam/blob/v1.0.6/internal/authzserver/analytics/analytics.go#L128-L173">recordWorker</a>（worker），recordWorker函数会从 <code>recordsChan</code> 中读取授权日志并存入recordsBuffer中，recordsBuffer的大小为workerBufferSize，workerBufferSize计算公式为：</p><pre><code class="language-go">ps := options.PoolSize
recordsBufferSize := options.RecordsBufferSize
workerBufferSize := recordsBufferSize / uint64(ps)
</code></pre><p>其中，options.PoolSize由命令行参数 <code>--analytics.pool-size</code> 指定，代表worker 的个数，默认 50；options.RecordsBufferSize由命令行参数 <code>--analytics.records-buffer-size</code> 指定，代表缓存的授权日志消息数。也就是说，我们把缓存的记录平均分配给所有的worker。</p><p>当recordsBuffer存满或者达到投递最大时间后，调用 <code>r.Store.AppendToSetPipelined(analyticsKeyName, recordsBuffer)</code> 将记录批量发送给Redis，为了提高传输速率，这里将日志内容编码为msgpack格式后再传输。</p><p>上面的缓存方法可以抽象成一个缓存模型，满足实际开发中的大部分需要异步转存的场景，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/47/95/479fyy2cd16a6c1fa5f6074f7ce6fe95.jpg?wh=2248x668" alt=""></p><p>Producer将数据投递到带缓冲的channel中，后端有多个worker消费channel中的数据，并进行批量投递。你可以设置批量投递的条件，一般至少包含<strong>最大投递日志数</strong>和<strong>最大投递时间间隔</strong>这两个。</p><p>通过以上缓冲模型，你可以将日志转存的时延降到最低。</p><h3>数据一致性</h3><p>上面介绍了 iam-authz-server的 <code>/v1/authz</code> 接口，为了最大化地提高性能，采用了大量的缓存设计。因为数据会分别在持久化存储和内存中都存储一份，就可能会出现数据不一致的情况。所以，我们也要确保缓存中的数据和数据库中的数据是一致的。数据一致性架构如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/72/a4/72c2afe63d197e7335deec1ac9f550a4.jpg?wh=2248x1006" alt=""></p><p>密钥和策略同步流程如下：</p><ol>
<li>通过iam-webconsole请求iam-apiserver创建（或更新、删除）密钥（或策略）。</li>
<li>iam-apiserver收到“写”请求后，会向Redis  <code>iam.cluster.notifications</code> channel发送PolicyChanged或SecretChanged消息。</li>
<li>Loader收到消息后，会触发cache loader实例执行 <code>Reload</code> 方法，重新从iam-apiserver中同步密钥和策略信息。</li>
</ol><p>Loader不会关心 <code>Reload</code> 方法的具体实现，只会在收到指定消息时，执行 <code>Reload</code> 方法。通过这种方式，我们可以实现不同的缓存策略。</p><p>在cache实例的 <code>Reload</code> 方法中，我们其实是调用仓库层Secret和Policy的List方法来获取密钥和策略列表。仓库层又是通过执行gRPC请求，从iam-apiserver中获取密钥和策略列表。</p><p>cache的<a href="https://github.com/marmotedu/iam/blob/v1.0.6/internal/authzserver/load/cache/cache.go#L105-L132">Reload</a>方法，会将获取到的密钥和策略列表缓存在<a href="https://github.com/dgraph-io/ristretto">ristretto</a>类型的Cache中，供业务层调用。业务层代码位于<a href="https://github.com/marmotedu/iam/tree/v1.0.6/internal/authzserver/authorization">internal/authzserver/authorization</a>目录下。</p><h2>总结</h2><p>这一讲中，我介绍了IAM数据流服务iam-authz-server的设计和实现。iam-authz-server提供了 <code>/v1/authz</code> RESTful API接口，供第三方用户完成资源授权功能，具体是使用Ladon包来完成资源授权的。Ladon包解决了“在特定的条件下，谁能够/不能够对哪些资源做哪些操作”的问题。</p><p>iam-authz-server的配置处理、启动流程和请求处理流程跟iam-apiserver保持一致。此外，iam-authz-server也实现了简洁架构。</p><p>iam-authz-server通过缓存密钥和策略信息、缓存授权日志来提高 <code>/v1/authz</code> 接口的性能。</p><p>在缓存密钥和策略信息时，为了和iam-apiserver中的密钥和策略信息保持一致，使用了Redis Pub/Sub机制。当iam-apiserver有密钥/策略变更时，会往指定的Redis channel Pub一条消息。iam-authz-server订阅相同的channel，在收到新消息时，会解析消息，并重新从iam-apiserver中获取密钥和策略信息，缓存在内存中。</p><p>iam-authz-server执行完资源授权之后，会将授权日志存放在一个带缓冲的channel中。后端有多个worker消费channel中的数据，并进行批量投递。可以设置批量投递的条件，例如最大投递日志数和最大投递时间间隔。</p><h2>课后练习</h2><ol>
<li>iam-authz-server和iam-apiserver共用了应用框架（包括一些配置项）和HTTP服务框架层的代码，请阅读iam-authz-server代码，看下IAM项目是如何实现代码复用的。</li>
<li>iam-authz-server使用了<a href="https://github.com/dgraph-io/ristretto">ristretto</a>来缓存密钥和策略信息，请调研下业界还有哪些优秀的缓存包可供使用，欢迎在留言区分享。</li>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/64/53/c93b8110.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>daz2yy</span>
  </div>
  <div class="_2_QraFYR_0">老师，请问下，数据流和控制流这个怎么来理解呢，是从什么角度来定义的服务类型的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 数据流承载核心业务功能。<br><br>控制流一般用来创建一些资源，供数据流使用。这些，资源一旦建成，控制流可以一直使用。所以这时候控制流挂了，资源还是存在的，数据流仍然能够正常运转，也就是不影响业务功能。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-06 07:33:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKhuGLVRYZibOTfMumk53Wn8Q0Rkg0o6DzTicbibCq42lWQoZ8lFeQvicaXuZa7dYsr9URMrtpXMVDDww/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>hello</span>
  </div>
  <div class="_2_QraFYR_0">老师，咨询您一个问题，使用GiN做WEB服务，微服务间通过gRPC通讯，如何选择配置注册中心，老师能否推荐几款比较流行开源的配置注册中心。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 网关可以使用，tyk：https:&#47;&#47;github.com&#47;TykTechnologies&#47;tyk。<br><br>tyk后端开源，前端闭源。你可以自己开发一套前端。<br><br>配置中心可以试试：https:&#47;&#47;github.com&#47;apache&#47;servicecomb-service-center<br><br>servicecomb-service-center是华为开源的注册中心，基于etcd封装。该注册中心可以无缝对接华为开源的微服务框架go-chassis</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-05 16:54:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/60/a1/8f003697.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>静心</span>
  </div>
  <div class="_2_QraFYR_0">本地channel缓冲和redis缓存对于性能的提高效果会很明显，设计的比较好。但同时，这样的设计会导致多存储数据同步的问题。比如，如果服务突然宕机，本地缓冲中的数据就可能丢失。不知道老师有没有什么好的办法解决？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个地方最好的办法，就是做好优雅关停。<br><br>关停时获取终止信号，等待任务完成，并之后后续的清理，最后结束进程。<br><br>服务器宕机，任何高可用系统，都会存在多多少少的数据丢失。能做的是设计好时间窗口，确保服务器压力、性能都在可接受范围的情况下，及时提交、保存数据。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-29 11:20:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/96/82/8ac1e909.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Jarvis</span>
  </div>
  <div class="_2_QraFYR_0">更新缓存时每次都 List 全部密钥&#47;策略，数据量会不会太大了？ Pub 时带上变化的策略&#47;密钥 id, 只更新该 id  的内容是不是好一点？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，最好是增量拉取</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-28 09:42:50</div>
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
  <div class="_2_QraFYR_0">总结：<br>iam-authz-server 在数据流上工作，负责授权工作，对性能要求高。<br>authz也需要对请求进行认证工作，authz 的认证采用 cache 方式实现 jwt token 认证，即密钥已经缓存在内存中，通过同样的加密方案，确认token的合法性。<br>authz的认证工作主要交给了 landon 来完成。iam-apiserver 存储的授权策略符合landon的语法规范，iam-authz-server 接收的授权请求，也符合landon的语法规范。landon 通过接口的方式，暴露了manager、auditLogger、metric 等相关的接口。比如，我们需要为landon提供用户的 policy 列表，是否允许授权，由 landon 来做决策。<br>缓存设计</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 6666</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-04 12:21:25</div>
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
  <div class="_2_QraFYR_0">详细介绍了iam-authz-server的设计与实现。<br>需要结合代码和跑起来的程序反复揣摩。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-16 23:06:31</div>
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
  <div class="_2_QraFYR_0">每次从apiserver触发reload() 都是全量的拉去 s.cli.GetPolicies()，这样应该可能会产生性能问题吧 假如当Policies数量特别大的时候</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，最好的方式是增量拉取，老哥有兴趣可以实现下，提个PR</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-17 13:31:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJKObzsYVibibyibmTVKBmoGPqS0WQC16EY4p1agGDCpv5okKpjzicLtHafBVa7TCwh9HaRxTx9qQ1Qkg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>陈先生</span>
  </div>
  <div class="_2_QraFYR_0">如果iam-authz-server挂了，是不是有audit log丢失的可能性？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 服务挂了，audit log是有丢失的可能，这个无法避免。只能尽可能规避</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-07 23:33:00</div>
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
  <div class="_2_QraFYR_0">在实际应用中，请求&#47;v1&#47;authz接口的参数体是网关根据用户实际请求的某个具体业务的api的参数、请求方法、path等，并根据提前定制的规则自动构造出来的吧，这样理解对吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是请求者自己指定的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-16 23:13:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/30/49/c4/c5ddbe2b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>岑惠韬</span>
  </div>
  <div class="_2_QraFYR_0">老师请问Load的Start函数第二个协程的作用是什么呢？是主要起解耦作用吗？假如让PubSubLoop直接操作requeue切片，让reloadLoop每秒清空切片，仅考虑当前用法的话是不是也是能跑的？还是会有什么逻辑上的问题？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. 从 reloadQueue中读取回调，但是iam-authz-server中，其实没有用到回调功能；<br>2. 我觉得直接操作requeue没有什么问题</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-06 12:02:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/c9/5e/b79e6d5d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>꧁子华宝宝萌萌哒꧂</span>
  </div>
  <div class="_2_QraFYR_0">preparedAuthzServer.Run 为啥需要一个 stopChan 来阻塞不让退出？<br><br>按我的理解这个 stopChan 没有写，这个进程永远就退不了， <br><br>直接 return s.genericAPIServer.Run() 不可以吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: preparedAuthzServer.Run  没有stopChan</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-27 17:22:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/1e/4c/10174727.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>xinHAOr</span>
  </div>
  <div class="_2_QraFYR_0">func (r *Analytics) Start() 里面为什么同步执行了Stop？刚启动完就停止了吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这里是一个bug，master分支已经更新了。教程这里后边会找编辑更新。<br><br>感谢反馈</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-16 15:45:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/7orjLiard5WYicG0WaRjk01ycCDtAZadtf2sWzg0c7vXl7oqIwic0QvzlE3lr3fgMZibqXSwAibV4Qu0YSeeMlibUMSg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_226b1b</span>
  </div>
  <div class="_2_QraFYR_0">老师，请问为什么用Ristretto缓存数据，不直接用Redis缓存数据呢？在用Redis缓存的基础上，讲一下MySQL与Redis的数据一致性相关的缓存读写策略会不会更好一点？把所有数据都简单缓存到一个缓存包&#47;Redis里，没有淘汰机制是不是不太好？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: ristretto是本地缓存，没用redis原因是：<br>1. policy、secret缓存没必要持久化<br>2. 减少依赖redis<br>3. 提高性能</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-16 20:08:04</div>
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
  <div class="_2_QraFYR_0">太强调授权相关的东西，并不是很好的 demo。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这里主要是通过一个授权系统来展示如何开发Go项目。所以，里面业务部分大部分是跟授权相关的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-27 19:36:29</div>
  </div>
</div>
</div>
</li>
</ul>