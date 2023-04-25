<audio title="28 _ 控制流（上）：通过iam-apiserver设计，看Web服务的构建" src="https://static001.geekbang.org/resource/audio/8d/26/8d0c5e17a5a7a007842d1fc8cae28c26.mp3" controls="controls"></audio> 
<p>你好，我是孔令飞。</p><p>前面我们讲了很多关于应用构建的内容，你一定迫不及待地想看下IAM项目的应用是如何构建的。那么接下来，我就讲解下IAM应用的源码。</p><p>在讲解过程中，我不会去讲解具体如何Code，但会讲解一些构建过程中的重点、难点，以及Code背后的设计思路、想法。我相信这是对你更有帮助的。</p><p>IAM项目有很多组件，这一讲，我先来介绍下IAM项目的门面服务：iam-apiserver（管理流服务）。我会先给你介绍下iam-apiserver的功能和使用方法，再介绍下iam-apiserver的代码实现。</p><h2>iam-apiserver服务介绍</h2><p>iam-apiserver是一个Web服务，通过一个名为iam-apiserver的进程，对外提供RESTful API接口，完成用户、密钥、策略三种REST资源的增删改查。接下来，我从功能和使用方法两个方面来具体介绍下。</p><h3>iam-apiserver功能介绍</h3><p>这里，我们可以通过iam-apiserver提供的RESTful API接口，来看下iam-apiserver具体提供的功能。iam-apiserver提供的RESTful API接口可以分为四类，具体如下：</p><p><strong>认证相关接口</strong></p><!-- [[[read_end]]] --><p><img src="https://static001.geekbang.org/resource/image/43/6d/43ec9261ccdb165c56e9c25b45e6af6d.jpg?wh=1920x1062" alt="图片"></p><p><strong>用户相关接口</strong></p><p><img src="https://static001.geekbang.org/resource/image/60/24/60f8a05f4cb43cbac84c0fb12c40c824.jpg?wh=1920x1314" alt="图片"></p><p><strong>密钥相关接口</strong></p><p><img src="https://static001.geekbang.org/resource/image/e8/95/e8f6aee66a29ff2a5aefeb00c5045c95.jpg?wh=1920x1326" alt="图片"></p><p><strong>策略相关接口</strong></p><p><img src="https://static001.geekbang.org/resource/image/0f/9e/0f3fcaa80020c3f72229fbab2f014a9e.jpg?wh=1920x1275" alt="图片"></p><h3>iam-apiserver使用方法介绍</h3><p>上面我介绍了iam-apiserver的功能，接下来就介绍下如何使用这些功能。</p><p>我们可以通过不同的客户端来访问iam-apiserver，例如前端、API调用、SDK、iamctl等。这些客户端最终都会执行HTTP请求，调用iam-apiserver提供的RESTful API接口。所以，我们首先需要有一个顺手的REST API客户端工具来执行HTTP请求，完成开发测试。</p><p>因为不同的开发者执行HTTP请求的方式、习惯不同，为了方便讲解，这里我统一通过cURL工具来执行HTTP请求。接下来先介绍下cURL工具。</p><p>标准的Linux发行版都安装了cURL工具。cURL可以很方便地完成RESTful API的调用场景，比如设置Header、指定HTTP请求方法、指定HTTP消息体、指定权限认证信息等。通过<code>-v</code>选项，也能输出REST请求的所有返回信息。cURL功能很强大，有很多参数，这里列出cURL工具常用的参数：</p><pre><code>-X/--request [GET|POST|PUT|DELETE|…]  指定请求的 HTTP 方法
-H/--header                           指定请求的 HTTP Header
-d/--data                             指定请求的 HTTP 消息体（Body）
-v/--verbose                          输出详细的返回信息
-u/--user                             指定账号、密码
-b/--cookie                           读取 cookie
</code></pre><p>此外，如果你想使用带UI界面的工具，这里我推荐你使用 Insomnia 。</p><p>Insomnia是一个跨平台的REST API客户端，与Postman、Apifox是一类工具，用于接口管理、测试。Insomnia功能强大，支持以下功能：</p><ul>
<li>发送HTTP请求；</li>
<li>创建工作区或文件夹；</li>
<li>导入和导出数据；</li>
<li>导出cURL格式的HTTP请求命令；</li>
<li>支持编写swagger文档；</li>
<li>快速切换请求；</li>
<li>URL编码和解码。</li>
<li>…</li>
</ul><p>Insomnia界面如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/63/e0/635aa6f3374af05ec2bff7e193314ae0.png?wh=1920x749" alt="图片"></p><p>当然了，也有很多其他优秀的带UI界面的REST API客户端，例如 Postman、Apifox等，你可以根据需要自行选择。</p><p>接下来，我用对secret资源的CURD操作，来给你演示下<strong>如何使用iam-apiserver的功能</strong>。你需要执行6步操作。</p><ol>
<li>登录iam-apiserver，获取token。</li>
<li>创建一个名为secret0的secret。</li>
<li>获取secret0的详细信息。</li>
<li>更新secret0的描述。</li>
<li>获取secret列表。</li>
<li>删除secret0。</li>
</ol><p>具体操作如下：</p><ol>
<li>登录iam-apiserver，获取token：</li>
</ol><pre><code>$ curl -s -XPOST -H&quot;Authorization: Basic `echo -n 'admin:Admin@2021'|base64`&quot; http://127.0.0.1:8080/login | jq -r .token
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXBpLm1hcm1vdGVkdS5jb20iLCJleHAiOjE2MzUwNTk4NDIsImlkZW50aXR5IjoiYWRtaW4iLCJpc3MiOiJpYW0tYXBpc2VydmVyIiwib3JpZ19pYXQiOjE2MjcyODM4NDIsInN1YiI6ImFkbWluIn0.gTS0n-7njLtpCJ7mvSnct2p3TxNTUQaduNXxqqLwGfI
</code></pre><p>这里，为了便于使用，我们将token设置为环境变量：</p><pre><code>TOKEN=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXBpLm1hcm1vdGVkdS5jb20iLCJleHAiOjE2MzUwNTk4NDIsImlkZW50aXR5IjoiYWRtaW4iLCJpc3MiOiJpYW0tYXBpc2VydmVyIiwib3JpZ19pYXQiOjE2MjcyODM4NDIsInN1YiI6ImFkbWluIn0.gTS0n-7njLtpCJ7mvSnct2p3TxNTUQaduNXxqqLwGfI
</code></pre><ol start="2">
<li>创建一个名为secret0的secret：</li>
</ol><pre><code>$ curl -v -XPOST -H &quot;Content-Type: application/json&quot; -H&quot;Authorization: Bearer ${TOKEN}&quot; -d'{&quot;metadata&quot;:{&quot;name&quot;:&quot;secret0&quot;},&quot;expires&quot;:0,&quot;description&quot;:&quot;admin secret&quot;}' http://iam.api.marmotedu.com:8080/v1/secrets
* About to connect() to iam.api.marmotedu.com port 8080 (#0)
*   Trying 127.0.0.1...
* Connected to iam.api.marmotedu.com (127.0.0.1) port 8080 (#0)
&gt; POST /v1/secrets HTTP/1.1
&gt; User-Agent: curl/7.29.0
&gt; Host: iam.api.marmotedu.com:8080
&gt; Accept: */*
&gt; Content-Type: application/json
&gt; Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOiJpYW0uYXBpLm1hcm1vdGVkdS5jb20iLCJleHAiOjE2MzUwNTk4NDIsImlkZW50aXR5IjoiYWRtaW4iLCJpc3MiOiJpYW0tYXBpc2VydmVyIiwib3JpZ19pYXQiOjE2MjcyODM4NDIsInN1YiI6ImFkbWluIn0.gTS0n-7njLtpCJ7mvSnct2p3TxNTUQaduNXxqqLwGfI
&gt; Content-Length: 72
&gt; 
* upload completely sent off: 72 out of 72 bytes
&lt; HTTP/1.1 200 OK
&lt; Content-Type: application/json; charset=utf-8
&lt; X-Request-Id: ff825bea-53de-4020-8e68-4e87574bd1ba
&lt; Date: Mon, 26 Jul 2021 07:20:26 GMT
&lt; Content-Length: 313
&lt; 
* Connection #0 to host iam.api.marmotedu.com left intact
{&quot;metadata&quot;:{&quot;id&quot;:60,&quot;instanceID&quot;:&quot;secret-jedr3e&quot;,&quot;name&quot;:&quot;secret0&quot;,&quot;createdAt&quot;:&quot;2021-07-26T15:20:26.885+08:00&quot;,&quot;updatedAt&quot;:&quot;2021-07-26T15:20:26.907+08:00&quot;},&quot;username&quot;:&quot;admin&quot;,&quot;secretID&quot;:&quot;U6CxKs0YVWyOp5GrluychYIRxDmMDFd1mOOD&quot;,&quot;secretKey&quot;:&quot;fubNIn8jLA55ktuuTpXM8Iw5ogdR2mlf&quot;,&quot;expires&quot;:0,&quot;description&quot;:&quot;admin secret&quot;}
</code></pre><p>可以看到，请求返回头中返回了<code>X-Request-Id</code> Header，<code>X-Request-Id</code>唯一标识这次请求。如果这次请求失败，就可以将<code>X-Request-Id</code>提供给运维或者开发，通过<code>X-Request-Id</code>定位出失败的请求，进行排障。另外<code>X-Request-Id</code>在微服务场景中，也可以透传给其他服务，从而实现请求调用链。</p><ol start="3">
<li>获取secret0的详细信息：</li>
</ol><pre><code>$ curl -XGET -H&quot;Authorization: Bearer ${TOKEN}&quot; http://iam.api.marmotedu.com:8080/v1/secrets/secret0
{&quot;metadata&quot;:{&quot;id&quot;:60,&quot;instanceID&quot;:&quot;secret-jedr3e&quot;,&quot;name&quot;:&quot;secret0&quot;,&quot;createdAt&quot;:&quot;2021-07-26T15:20:26+08:00&quot;,&quot;updatedAt&quot;:&quot;2021-07-26T15:20:26+08:00&quot;},&quot;username&quot;:&quot;admin&quot;,&quot;secretID&quot;:&quot;U6CxKs0YVWyOp5GrluychYIRxDmMDFd1mOOD&quot;,&quot;secretKey&quot;:&quot;fubNIn8jLA55ktuuTpXM8Iw5ogdR2mlf&quot;,&quot;expires&quot;:0,&quot;description&quot;:&quot;admin secret&quot;}
</code></pre><ol start="4">
<li>更新secret0的描述：</li>
</ol><pre><code>$ curl -XPUT -H&quot;Authorization: Bearer ${TOKEN}&quot; -d'{&quot;metadata&quot;:{&quot;name&quot;:&quot;secret&quot;},&quot;expires&quot;:0,&quot;description&quot;:&quot;admin secret(modify)&quot;}' http://iam.api.marmotedu.com:8080/v1/secrets/secret0
{&quot;metadata&quot;:{&quot;id&quot;:60,&quot;instanceID&quot;:&quot;secret-jedr3e&quot;,&quot;name&quot;:&quot;secret0&quot;,&quot;createdAt&quot;:&quot;2021-07-26T15:20:26+08:00&quot;,&quot;updatedAt&quot;:&quot;2021-07-26T15:23:35.878+08:00&quot;},&quot;username&quot;:&quot;admin&quot;,&quot;secretID&quot;:&quot;U6CxKs0YVWyOp5GrluychYIRxDmMDFd1mOOD&quot;,&quot;secretKey&quot;:&quot;fubNIn8jLA55ktuuTpXM8Iw5ogdR2mlf&quot;,&quot;expires&quot;:0,&quot;description&quot;:&quot;admin secret(modify)&quot;}
</code></pre><ol start="5">
<li>获取secret列表：</li>
</ol><pre><code>$ curl -XGET -H&quot;Authorization: Bearer ${TOKEN}&quot; http://iam.api.marmotedu.com:8080/v1/secrets
{&quot;totalCount&quot;:1,&quot;items&quot;:[{&quot;metadata&quot;:{&quot;id&quot;:60,&quot;instanceID&quot;:&quot;secret-jedr3e&quot;,&quot;name&quot;:&quot;secret0&quot;,&quot;createdAt&quot;:&quot;2021-07-26T15:20:26+08:00&quot;,&quot;updatedAt&quot;:&quot;2021-07-26T15:23:35+08:00&quot;},&quot;username&quot;:&quot;admin&quot;,&quot;secretID&quot;:&quot;U6CxKs0YVWyOp5GrluychYIRxDmMDFd1mOOD&quot;,&quot;secretKey&quot;:&quot;fubNIn8jLA55ktuuTpXM8Iw5ogdR2mlf&quot;,&quot;expires&quot;:0,&quot;description&quot;:&quot;admin secret(modify)&quot;}]}
</code></pre><ol start="6">
<li>删除secret0：</li>
</ol><pre><code>$ curl -XDELETE -H&quot;Authorization: Bearer ${TOKEN}&quot; http://iam.api.marmotedu.com:8080/v1/secrets/secret0
null
</code></pre><p>上面，我给你演示了密钥的使用方法。用户和策略资源类型的使用方法跟密钥类似。详细的使用方法你可以参考 <a href="https://github.com/marmotedu/iam/blob/v1.0.6/scripts/install/test.sh">test.sh</a> 脚本，该脚本是用来测试IAM应用的，里面包含了各个接口的请求方法。</p><p>这里，我还想顺便介绍下<strong>如何测试IAM应用中的各个部分</strong>。确保iam-apiserver、iam-authz-server、iam-pump等服务正常运行后，进入到IAM项目的根目录，执行以下命令：</p><pre><code>$ ./scripts/install/test.sh iam::test::test # 测试整个IAM应用是否正常运行
$ ./scripts/install/test.sh iam::test::login # 测试登陆接口是否可以正常访问
$ ./scripts/install/test.sh iam::test::user # 测试用户接口是否可以正常访问
$ ./scripts/install/test.sh iam::test::secret # 测试密钥接口是否可以正常访问
$ ./scripts/install/test.sh iam::test::policy # 测试策略接口是否可以正常访问
$ ./scripts/install/test.sh iam::test::apiserver # 测试iam-apiserver服务是否正常运行
$ ./scripts/install/test.sh iam::test::authz # 测试authz接口是否可以正常访问
$ ./scripts/install/test.sh iam::test::authzserver # 测试iam-authz-server服务是否正常运行
$ ./scripts/install/test.sh iam::test::pump # 测试iam-pump是否正常运行
$ ./scripts/install/test.sh iam::test::iamctl # 测试iamctl工具是否可以正常使用
$ ./scripts/install/test.sh iam::test::man # 测试man文件是否正确安装
</code></pre><p>所以，每次发布完iam-apiserver后，你可以执行以下命令来完成iam-apiserver的冒烟测试：</p><pre><code>$ export IAM_APISERVER_HOST=127.0.0.1 # iam-apiserver部署服务器的IP地址
$ export IAM_APISERVER_INSECURE_BIND_PORT=8080 # iam-apiserver HTTP服务的监听端口
$ ./scripts/install/test.sh iam::test::apiserver
</code></pre><h2>iam-apiserver代码实现</h2><p>上面，我介绍了iam-apiserver的功能和使用方法，这里我们再来看下iam-apiserver具体的代码实现。我会从配置处理、启动流程、请求处理流程、代码架构4个方面来讲解。</p><h3>iam-apiserver配置处理</h3><p>iam-apiserver服务的main函数位于<a href="https://github.com/marmotedu/iam/blob/v1.0.4/cmd/iam-apiserver/apiserver.go#L18">apiserver.go</a>文件中，你可以跟读代码，了解iam-apiserver的代码实现。这里，我来介绍下iam-apiserver服务的一些设计思想。</p><p>首先，来看下iam-apiserver中的3种配置：Options配置、应用配置和 HTTP/GRPC服务配置。</p><ul>
<li><strong>Options配置：</strong>用来构建命令行参数，它的值来自于命令行选项或者配置文件（也可能是二者Merge后的配置）。Options可以用来构建应用框架，Options配置也是应用配置的输入。</li>
<li><strong>应用</strong><strong>配置：</strong>iam-apiserver组件中需要的一切配置。有很多地方需要配置，例如，启动HTTP/GRPC需要配置监听地址和端口，初始化数据库需要配置数据库地址、用户名、密码等。</li>
<li><strong>HTTP/GRPC服务配置：</strong>启动HTTP服务或者GRPC服务需要的配置。</li>
</ul><p>这三种配置的关系如下图：</p><p><img src="https://static001.geekbang.org/resource/image/8c/b5/8ca8d9fa1efaab21e012471874e89cb5.jpg?wh=1346x727" alt=""></p><p>Options配置接管命令行选项，应用配置接管整个应用的配置，HTTP/GRPC服务配置接管跟HTTP/GRPC服务相关的配置。这3种配置独立开来，可以解耦命令行选项、应用和应用内的服务，使得这3个部分可以独立扩展，又不相互影响。</p><p>iam-apiserver根据Options配置来构建命令行参数和应用配置。</p><p>我们通过<code>github.com/marmotedu/iam/pkg/app</code>包的<a href="https://github.com/marmotedu/iam/blob/v1.0.4/pkg/app/app.go#L199">buildCommand</a>方法来构建命令行参数。这里的核心是，通过<a href="https://github.com/marmotedu/iam/blob/v1.0.4/pkg/app/app.go#L157">NewApp</a>函数构建Application实例时，传入的<a href="https://github.com/marmotedu/iam/blob/v1.0.4/internal/apiserver/options/options.go#L19">Options</a>实现了<code>Flags() (fss cliflag.NamedFlagSets)</code>方法，通过buildCommand方法中的以下代码，将option的Flag添加到cobra实例的FlagSet中：</p><pre><code>	if a.options != nil {
			namedFlagSets = a.options.Flags()
			fs := cmd.Flags()
			for _, f := range namedFlagSets.FlagSets {
				fs.AddFlagSet(f)
			}
	
            ...
		}
</code></pre><p>通过<a href="https://github.com/marmotedu/iam/blob/v1.0.4/internal/apiserver/config/config.go#L16">CreateConfigFromOptions</a>函数来构建应用配置：</p><pre><code>        cfg, err := config.CreateConfigFromOptions(opts)                      
        if err != nil {                                               
            return err                                                
        }  
</code></pre><p>根据应用配置来构建HTTP/GRPC服务配置。例如，以下代码根据应用配置，构建了HTTP服务器的Address参数：</p><pre><code>func (s *InsecureServingOptions) ApplyTo(c *server.Config) error {
    c.InsecureServing = &amp;server.InsecureServingInfo{
        Address: net.JoinHostPort(s.BindAddress, strconv.Itoa(s.BindPort)),
    }

    return nil
}
</code></pre><p>其中，<code>c *server.Config</code>是HTTP服务器的配置，<code>s *InsecureServingOptions</code>是应用配置。</p><h3>iam-apiserver启动流程设计</h3><p>接下来，我们来详细看下iam-apiserver的启动流程设计。启动流程如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/8a/c7/8a94938bc087ed96d0ec87261db292c7.jpg?wh=4770x1487" alt=""></p><p><strong>首先，</strong>通过<code>opts := options.NewOptions()</code>创建带有默认值的Options类型变量opts。opts变量作为<code>github.com/marmotedu/iam/pkg/app</code>包的<code>NewApp</code>函数的输入参数，最终在App框架中，被来自于命令行参数或配置文件的配置（也可能是二者Merge后的配置）所填充，opts变量中各个字段的值会用来创建应用配置。</p><p><strong>接着，</strong>会注册<a href="https://github.com/marmotedu/iam/blob/v1.0.4/internal/apiserver/apiserver.go#L36">run</a>函数到App框架中。run函数是iam-apiserver的启动函数，里面封装了我们自定义的启动逻辑。run函数中，首先会初始化日志包，这样我们就可以根据需要，在后面的代码中随时记录日志了。</p><p><strong>然后，</strong>会创建应用配置。应用配置和Options配置其实是完全独立的，二者可能完全不同，但在iam-apiserver中，二者配置项是相同的。</p><p><strong>之后，</strong>根据应用配置，创建HTTP/GRPC服务器所使用的配置。在创建配置后，会先分别进行配置补全，再使用补全后的配置创建Web服务实例，例如：</p><pre><code>genericServer, err := genericConfig.Complete().New()
if err != nil {
    return nil, err
}
extraServer, err := extraConfig.complete().New()
if err != nil {
    return nil, err
}
...
func (c *ExtraConfig) complete() *completedExtraConfig {
    if c.Addr == &quot;&quot; {
        c.Addr = &quot;127.0.0.1:8081&quot;
    }

    return &amp;completedExtraConfig{c}
}
</code></pre><p>上面的代码中，首先调用<code>Complete</code>/<code>complete</code>函数补全配置，再基于补全后的配置，New一个HTTP/GRPC服务实例。</p><p>这里有个设计技巧：<code>complete</code>函数返回的是一个<code>*completedExtraConfig</code>类型的实例，在创建GRPC实例时，是调用<code>completedExtraConfig</code>结构体提供的<code>New</code>方法，这种设计方法可以确保我们创建的GRPC实例一定是基于complete之后的配置（completed）。</p><p>在实际的Go项目开发中，我们需要提供一种机制来处理或补全配置，这在Go项目开发中是一个非常有用的步骤。</p><p><strong>最后，</strong>调用<code>PrepareRun</code>方法，进行HTTP/GRPC服务器启动前的准备。在准备函数中，我们可以做各种初始化操作，例如初始化数据库，安装业务相关的Gin中间件、RESTful API路由等。</p><p>完成HTTP/GRPC服务器启动前的准备之后，调用<code>Run</code>方法启动HTTP/GRPC服务。在<code>Run</code>方法中，分别启动了GRPC和HTTP服务。</p><p>可以看到，整个iam-apiserver的软件框架是比较清晰的。</p><p>服务启动后，就可以处理请求了。所以接下来，我们再来看下iam-apiserver的RESTAPI请求处理流程。</p><h3>iam-apiserver 的REST API请求处理流程</h3><p>iam-apiserver的请求处理流程也是清晰、规范的，具体流程如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/94/76/9400e9855b10yyac47871a7af87e9776.jpg?wh=5771x1691" alt=""></p><p>结合上面这张图，我们来看下iam-apiserver 的REST API请求处理流程，来帮你更好地理解iam-apiserver是如何处理HTTP请求的。</p><p><strong>首先，</strong>我们通过API调用（<code>&lt;HTTP Method&gt; + &lt;HTTP Request Path&gt;</code>）请求iam-apiserver提供的RESTful API接口。</p><p><strong>接着，</strong>Gin Web框架接收到HTTP请求之后，会通过认证中间件完成请求的认证，iam-apiserver提供了Basic认证和Bearer认证两种认证方式。</p><p><strong>认证</strong><strong>通过后，</strong>请求会被我们加载的一系列中间件所处理，例如跨域、RequestID、Dump等中间件。</p><p><strong>最后，</strong>根据<code>&lt;HTTP Method&gt; + &lt;HTTP Request Path&gt;</code>进行路由匹配。</p><p>举个例子，假设我们请求的RESTful API是<code>POST + /v1/secrets</code>，Gin Web框架会根据HTTP Method和HTTP Request Path，查找注册的Controllers，最终匹配到<a href="https://github.com/marmotedu/iam/blob/v1.0.4/internal/apiserver/controller/v1/secret/create.go">secretController.Create</a>Controller。在Create Controller中，我们会依次执行请求参数解析、请求参数校验、调用业务层的方法创建Secret、处理业务层的返回结果，最后返回最终的HTTP请求结果。</p><h3>iam-apiserver代码架构</h3><p>iam-apiserver代码设计遵循简洁架构设计，一个简洁架构具有以下5个特性：</p><ul>
<li><strong>独立于框架：</strong>该架构不会依赖于某些功能强大的软件库存在。这可以让你使用这样的框架作为工具，而不是让你的系统陷入到框架的约束中。</li>
<li><strong>可测试性：</strong>业务规则可以在没有UI、数据库、Web服务或其他外部元素的情况下进行测试，在实际的开发中，我们通过Mock来解耦这些依赖。</li>
<li><strong>独立于UI ：</strong>在无需改变系统其他部分的情况下，UI可以轻松地改变。例如，在没有改变业务规则的情况下，Web UI可以替换为控制台UI。</li>
<li><strong>独立于数据库：</strong>你可以用Mongo、Oracle、Etcd或者其他数据库来替换MariaDB，你的业务规则不要绑定到数据库。</li>
<li><strong>独立于外部媒介：</strong>实际上，你的业务规则可以简单到根本不去了解外部世界。</li>
</ul><p>所以，基于这些约束，每一层都必须是独立的和可测试的。iam-apiserver代码架构分为4层：模型层（Models）、控制层（Controller）、业务层 （Service）、仓库层（Repository）。从控制层、业务层到仓库层，从左到右层级依次加深。模型层独立于其他层，可供其他层引用。如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/f2/f0/f2fffd84dfbc1a6643887db3d5d541f0.jpg?wh=2498x747" alt=""></p><p>层与层之间导入包时，都有严格的导入关系，这可以防止包的循环导入问题。导入关系如下：</p><ul>
<li>模型层的包可以被仓库层、业务层和控制层导入；</li>
<li>控制层能够导入业务层和仓库层的包。这里需要注意，如果没有特殊需求，控制层要避免导入仓库层的包，控制层需要完成的业务功能都通过业务层来完成。这样可以使代码逻辑更加清晰、规范。</li>
<li>业务层能够导入仓库层的包。</li>
</ul><p>接下来，我们就来详细看下每一层所完成的功能，以及其中的一些注意点。</p><ol>
<li>模型层（Models）</li>
</ol><p>模型层在有些软件架构中也叫做实体层（Entities），模型会在每一层中使用，在这一层中存储对象的结构和它的方法。IAM项目模型层中的模型存放在<a href="https://github.com/marmotedu/api/tree/master/apiserver/v1">github.com/marmotedu/api/apiserver/v1</a>目录下，定义了<code>User</code>、<code>UserList</code>、<code>Secret</code>、<code>SecretList</code>、<code>Policy</code>、<code>PolicyList</code>、<code>AuthzPolicy</code>模型及其方法。例如：</p><pre><code>type Secret struct {
	// May add TypeMeta in the future.
	// metav1.TypeMeta `json:&quot;,inline&quot;`

	// Standard object's metadata.
	metav1.ObjectMeta `       json:&quot;metadata,omitempty&quot;`
	Username          string `json:&quot;username&quot;           gorm:&quot;column:username&quot;  validate:&quot;omitempty&quot;`
	SecretID          string `json:&quot;secretID&quot;           gorm:&quot;column:secretID&quot;  validate:&quot;omitempty&quot;`
	SecretKey         string `json:&quot;secretKey&quot;          gorm:&quot;column:secretKey&quot; validate:&quot;omitempty&quot;`

	// Required: true
	Expires     int64  `json:&quot;expires&quot;     gorm:&quot;column:expires&quot;     validate:&quot;omitempty&quot;`
	Description string `json:&quot;description&quot; gorm:&quot;column:description&quot; validate:&quot;description&quot;`
}
</code></pre><p>之所以将模型层的模型存放在<code>github.com/marmotedu/api</code>项目中，而不是<code>github.com/marmotedu/iam</code>项目中，是为了让这些模型能够被其他项目使用。例如，iam的模型可以被<code>github.com/marmotedu/shippy</code>应用导入。同样，shippy应用的模型也可以被iam项目导入，导入关系如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/13/c9/1307e374f4193ecc3d5b73a987cdd0c9.jpg?wh=3896x1433" alt=""></p><p>上面的依赖关系都是单向的，依赖关系清晰，不存在循环依赖的情况。</p><p>要增加shippy的模型定义，只需要在api目录下创建新的目录即可。例如，shippy应用中有一个vessel服务，其模型所在的包可以为<code>github.com/marmotedu/api/vessel</code>。</p><p>另外，这里的模型既可以作为数据库模型，又可以作为API接口的请求模型（入参、出参）。如果我们能够确保<strong>创建资源时的属性</strong>、<strong>资源保存在数据库中的属性</strong>、<strong>返回资源的属性</strong>三者一致，就可以使用同一个模型。通过使用同一个模型，可以使我们的代码更加简洁、易维护，并能提高开发效率。如果这三个属性有差异，你可以另外新建模型来适配。</p><ol start="2">
<li>仓库层（Repository)</li>
</ol><p>仓库层用来跟数据库/第三方服务进行CURD交互，作为应用程序的数据引擎进行应用数据的输入和输出。这里需要注意，仓库层仅对数据库/第三方服务执行CRUD操作，不封装任何业务逻辑。</p><p>仓库层也负责选择应用中将要使用什么样的数据库，可以是MySQL、MongoDB、MariaDB、Etcd等。无论使用哪种数据库，都要在这层决定。仓库层依赖于连接数据库或其他第三方服务（如果存在的话）。</p><p>这一层也会起到数据转换的作用：将从数据库/微服务中获取的数据转换为控制层、业务层能识别的数据结构，将控制层、业务层的数据格式转换为数据库或微服务能识别的数据格式。</p><p>iam-apiserver的仓库层位于<a href="https://github.com/marmotedu/iam/tree/v1.0.3/internal/apiserver/store/mysql">internal/apiserver/store/mysql</a>目录下，里面的方法用来跟MariaDB进行交互，完成CURD操作，例如，从数据库中获取密钥：</p><pre><code>func (s *secrets) Get(ctx context.Context, username, name string, opts metav1.GetOptions) (*v1.Secret, error) {
    secret := &amp;v1.Secret{}
    err := s.db.Where(&quot;username = ? and name= ?&quot;, username, name).First(&amp;secret).Error
    if err != nil {
        if errors.Is(err, gorm.ErrRecordNotFound) {
            return nil, errors.WithCode(code.ErrSecretNotFound, err.Error())
        }

        return nil, errors.WithCode(code.ErrDatabase, err.Error())
    }

    return secret, nil
}
</code></pre><ol start="3">
<li>业务层 (Service)</li>
</ol><p>业务层主要用来完成业务逻辑处理，我们可以把所有的业务逻辑处理代码放在业务层。业务层会处理来自控制层的请求，并根据需要请求仓库层完成数据的CURD操作。业务层功能如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/61/b6/6103c58d837fd81769977bc3c947ffb6.jpg?wh=1796x1236" alt=""></p><p>iam-apiserver的业务层位于<a href="https://github.com/marmotedu/iam/tree/v1.0.3/internal/apiserver/service">internal/apiserver/service</a>目录下。下面是iam-apiserver业务层中，用来创建密钥的函数：</p><pre><code>func (s *secretService) Create(ctx context.Context, secret *v1.Secret, opts metav1.CreateOptions) error {
    if err := s.store.Secrets().Create(ctx, secret, opts); err != nil {
        return errors.WithCode(code.ErrDatabase, err.Error())
    }

    return nil
}
</code></pre><p>可以看到，业务层最终请求仓库层的<code>s.store</code>的<code>Create</code>方法，将密钥信息保存在MariaDB数据库中。</p><ol start="4">
<li>控制层（Controller）</li>
</ol><p>控制层接收HTTP请求，并进行参数解析、参数校验、逻辑分发处理、请求返回这些操作。控制层会将逻辑分发给业务层，业务层处理后返回，返回数据在控制层中被整合再加工，最终返回给请求方。控制层相当于实现了业务路由的功能。具体流程如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/12/08/120137fc2749aa12a013099ec11e1b08.jpg?wh=960x1029" alt=""></p><p>这里我有个建议，不要在控制层写复杂的代码，如果需要，请将这些代码分发到业务层或其他包中。</p><p>iam-apiserver的控制层位于<a href="https://github.com/marmotedu/iam/tree/v1.0.3/internal/apiserver/controller">internal/apiserver/controller</a>目录下。下面是iam-apiserver控制层中创建密钥的代码：</p><pre><code>func (s *SecretHandler) Create(c *gin.Context) {
	log.L(c).Info(&quot;create secret function called.&quot;)

	var r v1.Secret

	if err := c.ShouldBindJSON(&amp;r); err != nil {
		core.WriteResponse(c, errors.WithCode(code.ErrBind, err.Error()), nil)

		return
	}

	if errs := r.Validate(); len(errs) != 0 {
		core.WriteResponse(c, errors.WithCode(code.ErrValidation, errs.ToAggregate().Error()), nil)

		return
	}

	username := c.GetString(middleware.UsernameKey)

	secrets, err := s.srv.Secrets().List(c, username, metav1.ListOptions{
		Offset: pointer.ToInt64(0),
		Limit:  pointer.ToInt64(-1),
	})
	if err != nil {
		core.WriteResponse(c, errors.WithCode(code.ErrDatabase, err.Error()), nil)

		return
	}

	if secrets.TotalCount &gt;= maxSecretCount {
		core.WriteResponse(c, errors.WithCode(code.ErrReachMaxCount, &quot;secret count: %d&quot;, secrets.TotalCount), nil)

		return
	}

	// must reassign username
	r.Username = username

	if err := s.srv.Secrets().Create(c, &amp;r, metav1.CreateOptions{}); err != nil {
		core.WriteResponse(c, err, nil)

		return
	}

	core.WriteResponse(c, nil, r)
}
</code></pre><p>上面的代码完成了以下操作：</p><ol>
<li>解析HTTP请求参数。</li>
<li>进行参数验证，这里可以添加一些业务性质的参数校验，例如：<code>secrets.TotalCount &gt;= maxSecretCount</code>。</li>
<li>调用业务层<code>s.srv</code>的<code>Create</code>方法，完成密钥的创建。</li>
<li>返回HTTP请求参数。</li>
</ol><p>上面，我们介绍了iam-apiserver采用的4层结构，接下来我们再看看<strong>每一层之间是如何通信的</strong>。</p><p>除了模型层，控制层、业务层、仓库层之间都是通过接口进行通信的。通过接口通信，一方面可以使相同的功能支持不同的实现（也就是说具有插件化能力），另一方面也使得每一层的代码变得可测试。</p><p>这里，我用创建密钥API请求的例子，来给你讲解下层与层之间是如何进行通信的。</p><p><strong>首先，来看下控制层如何跟业务层进行通信。</strong></p><p>对密钥的请求处理都是通过SecretController提供的方法来处理的，创建密钥调用的是它的<code>Create</code>方法：</p><pre><code>func (s *SecretController) Create(c *gin.Context) {
    ...
	if err := s.srv.Secrets().Create(c, &amp;r, metav1.CreateOptions{}); err != nil {
		core.WriteResponse(c, err, nil)

		return
	}
	...
}
</code></pre><p>在<code>Create</code>方法中，调用了<code>s.srv.Secrets().Create()</code>来创建密钥，<code>s.srv</code>是一个接口类型，定义如下：</p><pre><code>type Service interface {
    Users() UserSrv
    Secrets() SecretSrv
    Policies() PolicySrv
}

type SecretSrv interface {                                                             
    Create(ctx context.Context, secret *v1.Secret, opts metav1.CreateOptions) error    
    Update(ctx context.Context, secret *v1.Secret, opts metav1.UpdateOptions) error            
    Delete(ctx context.Context, username, secretID string, opts metav1.DeleteOptions) error                        
    DeleteCollection(ctx context.Context, username string, secretIDs []string, opts metav1.DeleteOptions) error    
    Get(ctx context.Context, username, secretID string, opts metav1.GetOptions) (*v1.Secret, error)    
    List(ctx context.Context, username string, opts metav1.ListOptions) (*v1.SecretList, error)    
} 
</code></pre><p>可以看到，控制层通过业务层提供的<code>Service</code>接口类型，剥离了业务层的具体实现。业务层的Service接口类型提供了<code>Secrets()</code>方法，该方法返回了一个实现了<code>SecretSrv</code>接口的实例。在控制层中，通过调用该实例的<code>Create(ctx context.Context, secret *v1.Secret, opts metav1.CreateOptions) error</code>方法来完成密钥的创建。至于业务层是如何创建密钥的，控制层不需要知道，也就是说创建密钥可以有多种实现。</p><p>这里使用到了设计模式中的<strong>工厂方法模式</strong>。<code>Service</code>是工厂接口，里面包含了一系列创建具体业务层对象的工厂函数：<code>Users()</code>、<code>Secrets()</code>、<code>Policies()</code>。通过工厂方法模式，不仅隐藏了业务层对象的创建细节，而且还可以很方便地在<code>Service</code>工厂接口实现方法中添加新的业务层对象。</p><p>例如，我们想新增一个<code>Template</code>业务层对象，用来在iam-apiserver中预置一些策略模板，可以这么来加：</p><pre><code>type Service interface {
    Users() UserSrv
    Secrets() SecretSrv
    Policies() PolicySrv
    Templates() TemplateSrv
}

func (s *service) Templates() TemplateSrv {
    return newTemplates(s)
}
</code></pre><p>接下来，新建一个<code>template.go</code>文件：</p><pre><code>type TemplateSrv interface {
    Create(ctx context.Context, template *v1.Template, opts metav1.CreateOptions) error
    // Other methods
}

type templateService struct {
    store store.Factory
}

var _ TemplateSrv = (*templateService)(nil)

func newTemplates(srv *service) *TemplateService {
    // more create logic
    return &amp;templateService{store: srv.store}
}

func (u *templateService) Create(ctx context.Context, template *v1.Template, opts metav1.CreateOptions) error {
    // normal code

    return nil
}
</code></pre><p>可以看到，我们通过以下三步新增了一个业务层对象：</p><ol>
<li>在<code>Service</code>接口定义中，新增了一个入口：<code>Templates() TemplateSrv</code>。</li>
<li>在<code>service.go</code>文件中，新增了一个函数：<code>Templates()</code>。</li>
<li>新建了<code>template.go</code>文件，在<code>template.go</code>中定义了templateService结构体，并为它实现了<code>TemplateSrv</code>接口。</li>
</ol><p>可以看到，我们新增的Template业务对象的代码几乎都闭环在<code>template.go</code>文件中。对已有的<code>Service</code>工厂接口的创建方法，除了新增一个工厂方法<code>Templates() TemplateSrv</code>外，没有其他任何入侵。这样做可以避免影响已有业务。</p><p>在实际项目开发中，你也有可能会想到下面这种错误的创建方式：</p><pre><code>// 错误方法一
type Service interface {
    UserSrv
    SecretSrv
    PolicySrv
    TemplateSrv
}
</code></pre><p>上面的创建方式中，我们如果想创建User和Secret，那只能定义两个不同的方法：CreateUser和 CreateSecret，远没有在User和Secret各自的域中提供同名的Create方法来得优雅。</p><p>IAM项目中还有其他地方也使用了工厂方法模式，例如<a href="https://github.com/marmotedu/iam/blob/v1.0.4/internal/apiserver/store/store.go#L12">Factory</a>工厂接口。</p><p><strong>再来看下业务层和仓库层是如何通信的。</strong></p><p>业务层和仓库层也是通过接口来通信的。例如，在业务层中创建密钥的代码如下：</p><pre><code>func (s *secretService) Create(ctx context.Context, secret *v1.Secret, opts metav1.CreateOptions) error {
    if err := s.store.Secrets().Create(ctx, secret, opts); err != nil {
        return errors.WithCode(code.ErrDatabase, err.Error())
    }

    return nil
}
</code></pre><p><code>Create</code>方法中调用了<code>s.store.Secrets().Create()</code>方法来将密钥保存到数据库中。<code>s.store</code>是一个接口类型，定义如下：</p><pre><code>type Factory interface {
    Users() UserStore
    Secrets() SecretStore
    Policies() PolicyStore
    Close() error
}
</code></pre><p>业务层与仓库层的通信实现，和控制层与业务层的通信实现类似，所以这里不再详细介绍。</p><p>到这里我们知道了，控制层、业务层和仓库层之间是通过接口来通信的。通过接口通信有一个好处，就是可以让各层变得可测。那接下来，我们就来看下<strong>如何测试各层的代码</strong>。因为<strong>第38讲</strong>和<strong>第39讲</strong>会详细介绍如何测试Go代码，所以这里只介绍下测试思路。</p><ol>
<li>模型层</li>
</ol><p>因为模型层不依赖其他任何层，我们只需要测试其中定义的结构及其函数和方法即可。</p><ol start="2">
<li>控制层</li>
</ol><p>控制层依赖于业务层，意味着该层需要业务层来支持测试。你可以通过<a href="https://github.com/golang/mock">golang/mock</a>来mock业务层，测试用例可参考<a href="https://github.com/marmotedu/iam/blob/v1.0.4/internal/apiserver/controller/v1/user/create_test.go#L19">TestUserController_Create</a>。</p><ol start="3">
<li>业务层</li>
</ol><p>因为该层依赖于仓库层，意味着该层需要仓库层来支持测试。我们有两种方法来模拟仓库层：</p><ul>
<li>通过<code>golang/mock</code>来mock仓库层。</li>
<li>自己开发一个fake仓库层。</li>
</ul><p>使用<code>golang/mock</code>的测试用例，你可以参考<a href="https://github.com/marmotedu/iam/blob/v1.0.4/internal/apiserver/service/v1/secret_test.go#L19">Test_secretService_Create</a>。</p><p>fake的仓库层可以参考<a href="https://github.com/marmotedu/iam/tree/v1.0.4/internal/apiserver/store/fake">fake</a>，使用该fake仓库层进行测试的测试用例为<a href="https://github.com/marmotedu/iam/blob/v1.0.4/internal/apiserver/service/v1/user_test.go#L76"> Test_userService_List</a>。</p><ol start="4">
<li>仓库层</li>
</ol><p>仓库层依赖于数据库，如果调用了其他微服务，那还会依赖第三方服务。我们可以通过<a href="https://github.com/DATA-DOG/go-sqlmock">sqlmock</a>来模拟数据库连接，通过<a href="https://github.com/jarcoal/httpmock">httpmock</a>来模拟HTTP请求。</p><h2>总结</h2><p>这一讲，我主要介绍了iam-apiserver的功能和使用方法，以及它的代码实现。iam-apiserver是一个Web服务，提供了REST API来完成用户、密钥、策略三种REST资源的增删改查。我们可以通过cURL、Insomnia等工具，来完成REST API请求。</p><p>iam-apiserver包含了3种配置：Options配置、应用配置、HTTP/GRPC服务配置。这三种配置分别用来构建命令行参数、应用和HTTP/GRPC服务。</p><p>iam-apiserver在启动时，会先构建应用框架，接着会设置应用选项，然后对应用进行初始化，最后创建HTTP/GRPC服务的配置和实例，最终启动HTTP/GRPC服务。</p><p>服务启动之后，就可以接收HTTP请求了。一个HTTP请求会先进行认证，接着会被注册的中间件处理，然后，会根据<code>(HTTP Method, HTTP Request Path)</code>匹配到处理函数。在处理函数中，会解析请求参数、校验参数、调用业务逻辑处理函数，最终返回请求结果。</p><p>iam-apiserver采用了简洁架构，整个应用分为4层：模型层、控制层、业务层和仓库层。模型层存储对象的结构和它的方法；仓库层用来跟数据库/第三方服务进行CURD交互；业务层主要用来完成业务逻辑处理；控制层接收HTTP请求，并进行参数解析、参数校验、逻辑分发处理、请求返回操作。控制层、业务层、仓库层之间通过接口通信，通过接口通信可以使相同的功能支持不同的实现，并使每一层的代码变得可测试。</p><h2>课后练习</h2><ol>
<li>
<p>iam-apiserver和iam-authz-server都提供了REST API服务，阅读它们的源码，看看iam-apiserver和iam-authz-server是如何共享REST API相关代码的。</p>
</li>
<li>
<p>思考一下，iam-apiserver的服务构建方式，能够再次抽象成一个模板（Go包）吗？如果能，该如何抽象？</p>
</li>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/87/64/3882d90d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>yandongxiao</span>
  </div>
  <div class="_2_QraFYR_0">总结：<br>介绍了 internal&#47;pkg&#47;apiserver 内部的组织结构<br>1. 最外层的Go文件主要完成：<br>   1. 构建应用框架（app.go）<br>   2. 提供应用框架执行时，需要的Run函数（run.go）<br>   3. 创建 apiServer 服务实例（server.go 和 grpc.go）<br>   4. 创建路由规则，安装中间件等（auth.go, router.go）<br>2. config 对应**应用配置**<br>3. options 对应命令行参数和配置文件。<br>4. controller、service、store 三个层次，controller 负责参数解析、校验；service 层负责业务逻辑；store负责数据库操作。<br><br>4层模型，模型层、控制层、业务层、存储层。层级之间如何解耦？<br>1. 通过 interface 实现解耦；<br>2. service层和store层的实例，是在请求执行过程中动态创建，它们都依赖一个工厂对象，作为实例化的输入。<br>3. 工厂对象 store store.Factory 不与具体的表、具体的操作绑定，通过store对象，你可以执行数据库的任何操作。类似的，srv srvv1.Service 工厂也不与具体的业务绑定，通过srv对象，你可以执行任何业务逻辑。<br><br>服务启动流程分为三个阶段：配置阶段、PreRun阶段、Run阶段。配置相关的对象有：Options、Config、HTTP&#47;GRPC 相关的配置对象。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 强！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-02 12:28:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/14/b9/47377590.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Jasper</span>
  </div>
  <div class="_2_QraFYR_0">var _ UserSrv = (*userService)(nil)  我是go初学者，这种语法表示没看明白</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这段代码的意思是：强制确保userService实现了UserSrv接口。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-23 12:59:31</div>
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
  <div class="_2_QraFYR_0">结合前面章节的铺垫，介绍iam-apiserver的使用方法、架构和代码实现，前后连贯，逻辑清晰，简洁易懂。<br>顺便一提，跟郑晔老师的《软件设计之美》中讲到的：快速了解一个项目时，要了解项目的模型、接口、实现，核心思想如出一辙。软件设计的路上，殊途同归。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，项目源码结合背后的设计思路，学习效果会翻倍的。学习一次，职业生涯中会一直受益</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-15 23:13:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/75/00/618b20da.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>徐海浪</span>
  </div>
  <div class="_2_QraFYR_0">练习1：通过component-base共享 REST API 相关代码</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-27 17:55:54</div>
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
  <div class="_2_QraFYR_0">似懂非懂，还得多看几遍😭</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-20 17:10:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/6b/9a/786b1ed8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>果粒橙</span>
  </div>
  <div class="_2_QraFYR_0">学习学习MVC模式，第一次接触</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-31 16:00:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/37/92/961ba560.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>授人以🐟，不如授人以渔</span>
  </div>
  <div class="_2_QraFYR_0">iam-apiserver 代码架构中「控制层能够导入业务层和仓库层的包」，是不是应该修改为：控制层能够导入业务层和模型层的包。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这里是：控制层能够导入业务层和仓库层的包<br><br>上面有提到：模型层的包可以被仓库层、业务层和控制层导入；</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-18 10:05:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/47/e0/1ff26e99.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>gecko</span>
  </div>
  <div class="_2_QraFYR_0">app包中的 如下类型 <br>type Command struct {<br>	usage    string<br>	desc     string<br>	options  CliOptions<br>	commands []*Command<br>	runFunc  RunCommandFunc<br>}<br><br><br>如果 commands 字段 有很多层的话  那么如下实现中<br><br>func (c *Command) cobraCommand() *cobra.Command {<br>	cmd := &amp;cobra.Command{<br>		Use:   c.usage,<br>		Short: c.desc,<br>	}<br>	cmd.SetOutput(os.Stdout)<br>	cmd.Flags().SortFlags = false<br>	if len(c.commands) &gt; 0 { &#47;&#47; c 的类型为 *Command，为c的commands 字段赋值  通过 ddCommand 方法装填<br>		for _, command := range c.commands {<br>			cmd.AddCommand(command.cobraCommand()) &#47;&#47; 迭代 直到 commands 切片的 长度为0 <br>		}<br>	}<br>	if c.runFunc != nil {<br>		cmd.Run = c.runCommand<br>	}<br>	if c.options != nil {<br>		for _, f := range c.options.Flags().FlagSets {<br>			cmd.Flags().AddFlagSet(f)<br>		}<br>		&#47;&#47; c.options.AddFlags(cmd.Flags())<br>	}<br>	addHelpCommandFlag(c.usage, cmd.Flags())<br><br>	return cmd<br>}<br><br>func (c *Command) AddCommand(cmd *Command) {<br>	c.commands = append(c.commands, cmd)<br>}<br><br><br>每层的 *cobra.Command  在调用方那里  cobra.Command.AddCommand 是被绑定到同一个层级的 cobra.Command 吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-30 11:25:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/5a/d0/7e58f993.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>梓荣</span>
  </div>
  <div class="_2_QraFYR_0">老师你好。如果出现业务逻辑相互调用，这种情况如何处理较妥？是否有一些业务逻辑的上层不是controller？这部分代码应该放在哪个目录。谢谢。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以试试拆包，将依赖的代码独立一个包，另外2个包import这个包。<br>这种情况很可能说明包设计不合理，需要根据实际情况重新设计包。<br><br>会有的。这部分代码仍然可以放在service目录下<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-18 09:53:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/67/3a/0dd9ea02.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Summer  空城</span>
  </div>
  <div class="_2_QraFYR_0">老师，您好，在app&#47;app.go 中 func NewApp(name string, basename string, opts ...Option) *App内部执行Option的方法，其实就是给App设置参数。为什么不来个set方法直接设置，现在这种设置方法感觉绕了一圈。。。。。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这里使用的是选项模式，好处是，新加参数选项时，不用修改NewApp代码</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-02 10:17:22</div>
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
  <div class="_2_QraFYR_0">“独立于 UI ：在无需改变系统其他部分的情况下，UI 可以轻松地改变。例如，在没有改变业务规则的情况下，Web UI 可以替换为控制台 UI。”, 这里的 Web UI和控制台 UI有什么区别啊, 这俩不是一回事吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: UI内容不一样，可能表述有点问题</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-10 18:32:17</div>
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
  <div class="_2_QraFYR_0">大佬，router.go 里面：<br>storeIns, _ := mysql.GetMySQLFactoryOr(nil)<br>userController := user.NewUserController(storeIns)<br>这个写法感觉好怪呀。感觉没必要，在这边初始化一个nil 的store.Factory。是否可以优化一下<br><br>但，我也还没想好如何修改😂</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 老哥可以改下，等着你的PR，哈哈</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-03 23:58:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_cede76</span>
  </div>
  <div class="_2_QraFYR_0">这里使用到了设计模式中的工厂方法模式。Service是工厂接口，里面包含了一系列创建具体业务层对象的工厂函数：Users()、Secrets()、Policies()。通过工厂方法模式，不仅隐藏了业务层对象的创建细节，而且还可以很方便地在Service工厂接口实现方法中添加新的业务层对象<br><br>老师，我感觉这里更像是抽象工厂方法模式</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这里用到了  工厂方法模式模式。可以参考 https:&#47;&#47;zhuanlan.zhihu.com&#47;p&#47;514427527</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-18 16:59:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_cede76</span>
  </div>
  <div class="_2_QraFYR_0">这里使用到了设计模式中的工厂方法模式。Service是工厂接口，里面包含了一系列创建具体业务层对象的工厂函数：Users()、Secrets()、Policies()。通过工厂方法模式，不仅隐藏了业务层对象的创建细节，而且还可以很方便地在Service工厂接口实现方法中添加新的业务层对象</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 666666</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-18 16:59:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>3608375821</span>
  </div>
  <div class="_2_QraFYR_0">api接口是在哪里注册到router上面的啊，没找到，比如&#47;v1&#47;secrets&#47;</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 在 internal&#47;apiserver&#47;router.go 文件中</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-06 22:42:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2c/82/ec/99b480e8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>hiDaLao</span>
  </div>
  <div class="_2_QraFYR_0">老师，请问下context作为参数一直传下去的目的是什么呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 2个目的：<br>1. 可以通过context.WithValue进行参数传递，这样就不用通过给函数添加入参的方式进行参数传递了，简单、灵活，扩展性高；<br>2. 可以通过context.WithTimeout的方式，传递关停信号、过期时间等</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-30 09:30:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/34/09/898d084e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>呆呆</span>
  </div>
  <div class="_2_QraFYR_0">func initRouter(g *gin.Engine) {<br>	installMiddleware(g)<br>	installController(g)<br>}<br>如上apiserver&#47;router.go的函数参数都为gin.Engine，是不是不太对？apiserver应该只与genericapiserver有关系，genericapiserver中集成了gin.Engine， genericapiserver中提供路由注册接口会不会更好些。<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以的，可能genericapiserver包要再封装一下</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-29 11:52:20</div>
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
  <div class="_2_QraFYR_0">请教：在仓库层如果不在想使用 mysql ，而是换成 etcd ，应该如何修改代码呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: internal&#47;apiserver&#47;server.go:	storeIns, _ := mysql.GetMySQLFactoryOr(c.mysqlOptions)<br>internal&#47;apiserver&#47;router.go:	storeIns, _ := mysql.GetMySQLFactoryOr(nil)<br><br>将 GetMySQLFactoryOr 改成 GetEtcdFactoryOr</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-10 12:10:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1c/63/72/a85661ee.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>船长</span>
  </div>
  <div class="_2_QraFYR_0">大佬，在模型层中依赖gorm，这种设计好吗？个人感觉模型层中是尽量不要依赖第三方跟某种实现强相关的东西，不知道大佬这么设计是出于什么样的考虑。<br>func (u *User) AfterCreate(tx *gorm.DB) error {<br>	u.InstanceID = idutil.GetInstanceID(u.ID, &quot;user-&quot;)<br><br>	return tx.Save(u).Error<br>}</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对，这里可以优化。<br><br>当时这么设计，是为了便捷吧，不用再另外搞一套model定义</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-05 23:36:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/f7/76/16c52796.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>逆行*乱世</span>
  </div>
  <div class="_2_QraFYR_0">➜  iam git:(f9fd52f) go mod vendor<br>go: github.com&#47;marmotedu&#47;errors@v1.0.2: stream error: stream ID 1; INTERNAL_ERROR 请问一下这个啥情况？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可能是网络不稳定吧</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-19 10:17:13</div>
  </div>
</div>
</div>
</li>
</ul>