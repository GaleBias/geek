<audio title="33 _  SDK 设计（上）：如何设计出一个优秀的 Go SDK？" src="https://static001.geekbang.org/resource/audio/97/f4/979d085783a9cc210156717875b933f4.mp3" controls="controls"></audio> 
<p>你好，我是孔令飞。接下来的两讲，我们来看下如何设计和实现一个优秀的Go SDK。</p><p>后端服务通过API接口对外提供应用的功能，但是用户直接调用API接口，需要编写API接口调用的逻辑，并且需要构造入参和解析返回的数据包，使用起来效率低，而且有一定的开发工作量。</p><p>在实际的项目开发中，通常会提供对开发者更友好的SDK包，供客户端调用。很多大型服务在发布时都会伴随着SDK的发布，例如腾讯云很多产品都提供了SDK：</p><p><img src="https://static001.geekbang.org/resource/image/e1/fa/e1bb8eb03c2f26f546710e95751c17fa.png?wh=1920x747" alt="图片"></p><p>既然SDK如此重要，那么如何设计一个优秀的Go SDK呢？这一讲我就来详细介绍一下。</p><h2>什么是SDK？</h2><p>首先，我们来看下什么是SDK。</p><p>对于SDK（Software Development Kit，软件开发工具包），不同场景下有不同的解释。但是对于一个Go后端服务来说，SDK通常是指<strong>封装了Go后端服务API接口的软件包</strong>，里面通常包含了跟软件相关的库、文档、使用示例、封装好的API接口和工具。</p><p>调用SDK跟调用本地函数没有太大的区别，所以可以极大地提升开发者的开发效率和体验。SDK可以由服务提供者提供，也可以由其他组织或个人提供。为了鼓励开发者使用其系统或语言，SDK通常都是免费提供的。</p><p>通常，服务提供者会提供不同语言的SDK，比如针对Python开发者会提供Python版的SDK，针对Go开发者会提供Go版的SDK。一些比较专业的团队还会有SDK自动生成工具，可以根据API接口定义，自动生成不同语言的SDK。例如，Protocol Buffers的编译工具protoc，就可以基于Protobuf文件生成C++、Python、Java、JavaScript、PHP等语言版本的SDK。阿里云、腾讯云这些一线大厂，也可以基于API定义，生成不同编程语言的SDK。</p><!-- [[[read_end]]] --><h2>SDK设计方法</h2><p>那么，我们如何才能设计一个好的SDK呢？对于SDK，不同团队会有不同的设计方式，我调研了一些优秀SDK的实现，发现这些SDK有一些共同点。根据我的调研结果，结合我在实际开发中的经验，我总结出了一套SDK设计方法，接下来就分享给你。</p><h3>如何给SDK命名？</h3><p>在讲设计方法之前，我先来介绍两个重要的知识点：SDK的命名方式和SDK的目录结构。</p><p>SDK的名字目前没有统一的规范，但比较常见的命名方式是 <code>xxx-sdk-go</code> / <code>xxx-sdk-python</code> / <code>xxx-sdk-java</code> 。其中， <code>xxx</code> 可以是项目名或者组织名，例如腾讯云在GitHub上的组织名为tencentcloud，那它的SDK命名如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/e2/1e/e269d5d0e19a73d45ccdf5f5561c611e.png?wh=1210x863" alt="图片"></p><h3>SDK的目录结构</h3><p>不同项目SDK的目录结构也不相同，但一般需要包含下面这些文件或目录。目录名可能会有所不同，但目录功能是类似的。</p><ul>
<li><strong>README.md</strong>：SDK的帮助文档，里面包含了安装、配置和使用SDK的方法。</li>
<li><strong>examples/sample/</strong>：SDK的使用示例。</li>
<li><strong>sdk/</strong>：SDK共享的包，里面封装了最基础的通信功能。如果是HTTP服务，基本都是基于 <code>net/http</code> 包进行封装。</li>
<li><strong>api</strong>：如果 <code>xxx-sdk-go</code> 只是为某一个服务提供SDK，就可以把该服务的所有API接口封装代码存放在api目录下。</li>
<li><strong>services/{iam, tms}</strong> ：如果 <code>xxx-sdk-go</code> 中， <code>xxx</code> 是一个组织，那么这个SDK很可能会集成该组织中很多服务的API，就可以把某类服务的API接口封装代码存放在 <code>services/&lt;服务名&gt;</code>下，例如AWS的<a href="https://github.com/aws/aws-sdk-go/tree/main/service">Go SDK</a>。</li>
</ul><p>一个典型的目录结构如下：</p><pre><code class="language-bash">├── examples            # 示例代码存放目录
│   └── authz.go
├── README.md           # SDK使用文档
├── sdk                 # 公共包，封装了SDK配置、API请求、认证等代码
│   ├── client.go
│   ├── config.go
│   ├── credential.go
│   └── ...
└── services            # API封装
    ├── common
    │   └── model
    ├── iam             # iam服务的API接口
    │   ├── authz.go
    │   ├── client.go
    │   └── ...
    └── tms             # tms服务的API接口
</code></pre><h3>SDK设计方法</h3><p>SDK的设计方法如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/9f/ca/9fb7aa8d3da4210223e9b0c87943e8ca.jpg?wh=1920x841" alt="图片"></p><p>我们可以通过Config配置创建客户端Client，例如 <code>func NewClient(config sdk.Config) (Client, error)</code>，配置中可以指定下面的信息。</p><ul>
<li>服务的后端地址：服务的后端地址可以通过配置文件来配置，也可以直接固化在SDK中，推荐后端服务地址可通过配置文件配置。</li>
<li>认证信息：最常用的认证方式是通过密钥认证，也有一些是通过用户名和密码认证。</li>
<li>其他配置：例如超时时间、重试次数、缓存时间等。</li>
</ul><p>创建的Client是一个结构体或者Go interface。这里我建议你使用interface类型，这样可以将定义和具体实现解耦。Client具有一些方法，例如 CreateUser、DeleteUser等，每一个方法对应一个API接口，下面是一个Client定义：</p><pre><code class="language-go">type Client struct {
    client *sdk.Request
}

func (c *Client) CreateUser(req *CreateUserRequest) (*CreateUserResponse, error) {
    // normal code
    resp := &amp;CreateUserResponse{}
    err := c.client.Send(req, resp)
    return resp, err
}
</code></pre><p>调用 <code>client.CreateUser(req)</code> 会执行HTTP请求，在 <code>req</code> 中可以指定HTTP请求的方法Method、路径Path和请求Body。 <code>CreateUser</code> 函数中，会调用 <code>c.client.Send(req)</code> 执行具体的HTTP请求。</p><p><code>c.client</code> 是 <code>*Request</code> 类型的变量，  <code>*Request</code> 类型的变量具有一些方法，可以根据传入的请求参数 <code>req</code> 和 <code>config</code> 配置构造出请求路径、认证头和请求Body，并调用 <code>net/http</code> 包完成最终的HTTP请求，最后将返回结果Unmarshal到传入的 <code>resp</code> 结构体中。</p><p>根据我的调研，目前有两种SDK设计方式可供参考，一种是各大公有云厂商采用的SDK设计方式，一种是Kubernetes client-go的设计方式。IAM项目分别实现了这两种SDK设计方式，但我还是更倾向于对外提供client-go方式的SDK，我会在下一讲详细介绍它。这两种设计方式的设计思路跟上面介绍的是一致的。</p><h2>公有云厂商采用的SDK设计方式</h2><p>这里，我先来简单介绍下公有云厂商采用的SDK设计模式。SDK架构如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/82/ce/82ebe90b0490b9a2a76e2f302dd896ce.jpg?wh=1920x866" alt="图片"></p><p>SDK框架分为两层，分别是API层和基础层。API层主要用来构建客户端实例，并调用客户端实例提供的方法来完成API请求，每一个方法对应一个API接口。API层最终会调用基础层提供的能力，来完成REST API请求。基础层通过依次执行构建请求参数（Builder）、签发并添加认证头（Signer）、执行HTTP请求（Request）三大步骤，来完成具体的REST API请求。</p><p>为了让你更好地理解公有云SDK的设计方式，接下来我会结合一些真实的代码，给你讲解API层和基础层的具体设计，SDK代码见<a href="https://github.com/marmotedu/medu-sdk-go">medu-sdk-go</a>。</p><h3>API层：创建客户端实例</h3><p>客户端在使用服务A的SDK时，首先需要根据Config配置创建一个服务A的客户端Client，Client实际上是一个struct，定义如下：</p><pre><code class="language-go">type Client struct {
    sdk.Client
}
</code></pre><p>在创建客户端时，需要传入认证（例如密钥、用户名/密码）、后端服务地址等配置信息。例如，可以通过<a href="https://github.com/marmotedu/medu-sdk-go/blob/v1.0.0/services/iam/authz/client.go#L24-L29">NewClientWithSecret</a>方法来构建一个带密钥对的客户端：</p><pre><code class="language-go">func NewClientWithSecret(secretID, secretKey string) (client *Client, err error) {
    client = &amp;Client{}
    config := sdk.NewConfig().WithEndpoint(defaultEndpoint)
    client.Init(serviceName).WithSecret(secretID, secretKey).WithConfig(config)
    return
}
</code></pre><p>这里要注意，上面创建客户端时，传入的密钥对最终会在基础层中被使用，用来签发JWT Token。</p><p>Client有多个方法（Sender），例如 Authz等，每个方法代表一个API接口。Sender方法会接收AuthzRequest等结构体类型的指针作为输入参数。我们可以调用 <code>client.Authz(req)</code> 来执行REST API调用。可以在 <code>client.Authz</code> 方法中添加一些业务逻辑处理。<code>client.Authz</code> 代码如下：</p><pre><code class="language-go">type AuthzRequest struct {
&nbsp; &nbsp; *request.BaseRequest
&nbsp; &nbsp; Resource *string `json:"resource"`
&nbsp; &nbsp; Action *string `json:"action"`
&nbsp; &nbsp; Subject *string `json:"subject"`
&nbsp; &nbsp; Context *ladon.Context
}

func (c *Client) Authz(req *AuthzRequest) (resp *AuthzResponse, err error) {
&nbsp; &nbsp; if req == nil {
&nbsp; &nbsp; &nbsp; &nbsp; req = NewAuthzRequest()
&nbsp; &nbsp; }

&nbsp; &nbsp; resp = NewAuthzResponse()
&nbsp; &nbsp; err = c.Send(req, resp)
&nbsp; &nbsp; return
}
</code></pre><p>请求结构体中的字段都是指针类型的，使用指针的好处是可以判断入参是否有被指定，如果<code>req.Subject == nil</code> 就说明传参中没有Subject参数，如果<code>req.Subject != nil</code>就说明参数中有传Subject参数。根据某个参数是否被传入，执行不同的业务逻辑，这在Go API接口开发中非常常见。</p><p>另外，因为Client通过匿名的方式继承了基础层中的<a href="https://github.com/marmotedu/medu-sdk-go/blob/v1.0.0/sdk/client.go#L18-L24">Client</a>：</p><pre><code class="language-go">type Client struct {
	sdk.Client
}
</code></pre><p>所以，API层创建的Client最终可以直接调用基础层中的Client提供的<code>Send(req, resp)</code> 方法，来执行RESTful API调用，并将结果保存在 <code>resp</code> 中。</p><p>为了方便和API层的Client进行区分，我下面统一将基础层中的Client称为<strong>sdk.Client</strong>。</p><p>最后，一个完整的客户端调用示例代码如下：</p><pre><code class="language-go">package main

import (
	"fmt"

	"github.com/ory/ladon"

	"github.com/marmotedu/medu-sdk-go/sdk"
	iam "github.com/marmotedu/medu-sdk-go/services/iam/authz"
)

func main() {
	client, _ := iam.NewClientWithSecret("XhbY3aCrfjdYcP1OFJRu9xcno8JzSbUIvGE2", "bfJRvlFwsoW9L30DlG87BBW0arJamSeK")

	req := iam.NewAuthzRequest()
	req.Resource = sdk.String("resources:articles:ladon-introduction")
	req.Action = sdk.String("delete")
	req.Subject = sdk.String("users:peter")
	ctx := ladon.Context(map[string]interface{}{"remoteIPAddress": "192.168.0.5"})
	req.Context = &amp;ctx

	resp, err := client.Authz(req)
	if err != nil {
		fmt.Println("err1", err)
		return
	}
	fmt.Printf("get response body: `%s`\n", resp.String())
	fmt.Printf("allowed: %v\n", resp.Allowed)
}
</code></pre><h3>基础层：构建并执行HTTP请求</h3><p>上面我们创建了客户端实例，并调用了它的 <a href="https://github.com/marmotedu/medu-sdk-go/blob/v1.0.0/sdk/client.go#L61-L93">Send</a> 方法来完成最终的HTTP请求。这里，我们来看下Send方法具体是如何构建HTTP请求的。</p><p>sdk.Client通过Send方法，完成最终的API调用，代码如下：</p><pre><code class="language-go">func (c *Client) Send(req request.Request, resp response.Response) error {
	method := req.GetMethod()
	builder := GetParameterBuilder(method, c.Logger)
	jsonReq, _ := json.Marshal(req)
	encodedUrl, err := builder.BuildURL(req.GetURL(), jsonReq)
	if err != nil {
		return err
	}

	endPoint := c.Config.Endpoint
	if endPoint == "" {
		endPoint = fmt.Sprintf("%s/%s", defaultEndpoint, c.ServiceName)
	}
	reqUrl := fmt.Sprintf("%s://%s/%s%s", c.Config.Scheme, endPoint, req.GetVersion(), encodedUrl)

	body, err := builder.BuildBody(jsonReq)
	if err != nil {
		return err
	}

	sign := func(r *http.Request) error {
		signer := NewSigner(c.signMethod, c.Credential, c.Logger)
		_ = signer.Sign(c.ServiceName, r, strings.NewReader(body))
		return err
	}

	rawResponse, err := c.doSend(method, reqUrl, body, req.GetHeaders(), sign)
	if err != nil {
		return err
	}

	return response.ParseFromHttpResponse(rawResponse, resp)
}
</code></pre><p>上面的代码大体上可以分为四个步骤。</p><p><strong>第一步，Builder：构建请求参数。</strong></p><p>根据传入的AuthzRequest和客户端配置Config，构造HTTP请求参数，包括请求路径和请求Body。</p><p>接下来，我们来看下如何构造HTTP请求参数。</p><ol>
<li>HTTP请求路径构建</li>
</ol><p>在创建客户端时，我们通过<a href="https://github.com/marmotedu/medu-sdk-go/blob/v1.0.0/services/iam/authz/authz.go#L32-L42">NewAuthzRequest</a>函数创建了 <code>/v1/authz</code> REST API接口请求结构体AuthzRequest，代码如下：</p><pre><code class="language-go">func NewAuthzRequest() (req *AuthzRequest) {&nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; req = &amp;AuthzRequest{&nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; &nbsp; &nbsp; BaseRequest: &amp;request.BaseRequest{&nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; URL:&nbsp; &nbsp; &nbsp;"/authz",&nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; Method:&nbsp; "POST",&nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; Header:&nbsp; nil,&nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; Version: "v1",&nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; &nbsp; &nbsp; },&nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; }&nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; return&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;
}
</code></pre><p>可以看到，我们创建的 <code>req</code> 中包含了API版本（Version）、API路径（URL）和请求方法（Method）。这样，我们就可以在Send方法中，构建出请求路径：</p><pre><code class="language-go">endPoint := c.Config.Endpoint&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
if endPoint == "" {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; endPoint = fmt.Sprintf("%s/%s", defaultEndpoint, c.ServiceName)&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
}&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;
reqUrl := fmt.Sprintf("%s://%s/%s%s", c.Config.Scheme, endPoint, req.GetVersion(), encodedUrl)&nbsp;
</code></pre><p>上述代码中，c.Config.Scheme=http/https、endPoint=iam.api.marmotedu.com:8080、req.GetVersion()=v1和encodedUrl，我们可以认为它们等于/authz。所以，最终构建出的请求路径为<code>http://iam.api.marmotedu.com:8080/v1/authz</code> 。</p><ol start="2">
<li>HTTP请求Body构建</li>
</ol><p>在<a href="https://github.com/marmotedu/medu-sdk-go/blob/v1.0.0/sdk/parameter_builder.go#L68-L86">BuildBody</a>方法中构建请求Body。BuildBody会将 <code>req</code> Marshal成JSON格式的string。HTTP请求会以该字符串作为Body参数。</p><p><strong>第二步，Signer：签发并添加认证头。</strong></p><p>访问IAM的API接口需要进行认证，所以在发送HTTP请求之前，还需要给HTTP请求添加认证Header。</p><p>medu-sdk-go 代码提供了JWT和HMAC两种认证方式，最终采用了JWT认证方式。JWT认证签发方法为<a href="https://github.com/marmotedu/medu-sdk-go/blob/v1.0.0/sdk/signer.go#L108-L113">Sign</a>，代码如下：</p><pre><code class="language-go">func (v1 SignatureV1) Sign(serviceName string, r *http.Request, body io.ReadSeeker) http.Header {
	tokenString := auth.Sign(v1.Credentials.SecretID, v1.Credentials.SecretKey, "medu-sdk-go", serviceName+".marmotedu.com")
	r.Header.Set("Authorization", fmt.Sprintf("Bearer %s", tokenString))
	return r.Header

}
</code></pre><p><code>auth.Sign</code> 方法根据SecretID和SecretKey签发JWT Token。</p><p>接下来，我们就可以调用<a href="https://github.com/marmotedu/medu-sdk-go/blob/v1.0.0/sdk/client.go#L95-L112">doSend</a>方法来执行HTTP请求了。调用代码如下：</p><pre><code class="language-go">rawResponse, err := c.doSend(method, reqUrl, body, req.GetHeaders(), sign)
if err != nil {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
&nbsp; &nbsp; return err&nbsp; &nbsp; &nbsp;
}&nbsp;
</code></pre><p>可以看到，我们传入了HTTP请求方法 <code>method</code> 、HTTP请求URL  <code>reqUrl</code> 、HTTP请求Body  <code>body</code>，以及用来签发JWT Token的 <code>sign</code> 方法。我们在调用 <code>NewAuthzRequest</code> 创建 <code>req</code> 时，指定了HTTP Method，所以这里的 <code>method := req.GetMethod()</code> 、reqUrl和请求Body都是通过Builder来构建的。</p><p><strong>第三步，Request：执行H<strong><strong>T</strong></strong>T<strong><strong>P</strong></strong>请求。</strong></p><p>调用<a href="https://github.com/marmotedu/medu-sdk-go/blob/v1.0.0/sdk/client.go#L95-L112">doSend</a>方法执行HTTP请求，doSend通过调用 <code>net/http</code> 包提供的 <code>http.NewRequest</code> 方法来发送HTTP请求，执行完HTTP请求后，会返回 <code>*http.Response</code> 类型的Response。代码如下：</p><pre><code class="language-go">func (c *Client) doSend(method, url, data string, header map[string]string, sign SignFunc) (*http.Response, error) {
&nbsp; &nbsp; client := &amp;http.Client{Timeout: c.Config.Timeout}

&nbsp; &nbsp; req, err := http.NewRequest(method, url, strings.NewReader(data))
&nbsp; &nbsp; if err != nil {
&nbsp; &nbsp; &nbsp; &nbsp; c.Logger.Errorf("%s", err.Error())
&nbsp; &nbsp; &nbsp; &nbsp; return nil, err
&nbsp; &nbsp; }

&nbsp; &nbsp; c.setHeader(req, header)

&nbsp; &nbsp; err = sign(req)
&nbsp; &nbsp; if err != nil {
&nbsp; &nbsp; &nbsp; &nbsp; return nil, err
&nbsp; &nbsp; }

&nbsp; &nbsp; return client.Do(req)
}
</code></pre><p><strong>第四步，处理H<strong><strong>TT</strong></strong>P<strong><strong>请求</strong></strong>返回结果。</strong></p><p>调用doSend方法返回 <code>*http.Response</code> 类型的Response后，Send方法会调用<a href="https://github.com/marmotedu/medu-sdk-go/blob/v1.0.0/sdk/response/response.go#L37-L52">ParseFromHttpResponse</a>函数来处理HTTP Response，ParseFromHttpResponse函数代码如下：</p><pre><code class="language-go">func ParseFromHttpResponse(rawResponse *http.Response, response Response) error {
	defer rawResponse.Body.Close()
	body, err := ioutil.ReadAll(rawResponse.Body)
	if err != nil {
		return err
	}
	if rawResponse.StatusCode != 200 {
		return fmt.Errorf("request fail with status: %s, with body: %s", rawResponse.Status, body)
	}

	if err := response.ParseErrorFromHTTPResponse(body); err != nil {
		return err
	}

	return json.Unmarshal(body, &amp;response)
}
</code></pre><p>可以看到，在ParseFromHttpResponse函数中，会先判断HTTP Response中的StatusCode是否为200，如果不是200，则会报错。如果是200，会调用传入的resp变量提供的<a href="https://github.com/marmotedu/medu-sdk-go/blob/v1.0.0/sdk/response/response.go#L26-L35">ParseErrorFromHTTPResponse</a>方法，来将HTTP Response的Body Unmarshal到resp变量中。<br>
通过以上四步，SDK调用方调用了API，并获得了API的返回结果 <code>resp</code> 。</p><p>下面这些公有云厂商的SDK采用了此设计模式：</p><ul>
<li>腾讯云SDK：<a href="https://github.com/TencentCloud/tencentcloud-sdk-go">tencentcloud-sdk-go</a>。</li>
<li>AWS SDK：<a href="https://github.com/aws/aws-sdk-go">aws-sdk-go</a>。</li>
<li>阿里云SDK：<a href="https://github.com/aliyun/alibaba-cloud-sdk-go">alibaba-cloud-sdk-go</a>。</li>
<li>京东云SDK：<a href="https://github.com/jdcloud-api/jdcloud-sdk-go">jdcloud-sdk-go</a>。</li>
<li>Ucloud SDK：<a href="https://github.com/ucloud/ucloud-sdk-go">ucloud-sdk-go</a>。</li>
</ul><p>IAM公有云方式的SDK实现为 <a href="https://github.com/marmotedu/medu-sdk-go">medu-sdk-go</a>。</p><p>此外，IAM还设计并实现了Kubernetes client-go方式的Go SDK：<a href="https://github.com/marmotedu/marmotedu-sdk-go">marmotedu-sdk-go</a>，marmotedu-sdk-go也是IAM Go SDK所采用的SDK。下一讲中，我会具体介绍marmotedu-sdk-go的设计和实现。</p><h2>总结</h2><p>这一讲，我主要介绍了如何设计一个优秀的Go SDK。通过提供SDK，可以提高API调用效率，减少API调用难度，所以大型应用通常都会提供SDK。不同团队有不同的SDK设计方法，但目前比较好的实现是公有云厂商采用的SDK设计方式。</p><p>公有云厂商的SDK设计方式中，SDK按调用顺序从上到下可以分为3个模块，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/b9/a9/b9bd3020ae56f6bb49bc3a38bcaf64a9.jpg?wh=1920x878" alt="图片"></p><p>Client构造SDK客户端，在构造客户端时，会创建请求参数 <code>req</code> ， <code>req</code> 中会指定API版本、HTTP请求方法、API请求路径等信息。</p><p>Client会请求Builder和Signer来构建HTTP请求的各项参数：HTTP请求方法、HTTP请求路径、HTTP认证头、HTTP请求Body。Builder和Signer是根据 <code>req</code> 配置来构造这些HTTP请求参数的。</p><p>构造完成之后，会请求Request模块，Request模块通过调用 <code>net/http</code> 包，来执行HTTP请求，并返回请求结果。</p><h2>课后练习</h2><ol>
<li>思考下，如何实现可以支持多个API版本的SDK包，代码如何实现？</li>
<li>这一讲介绍了一种SDK实现方式，在你的Go开发生涯中，还有没有一些更好的SDK实现方法？欢迎在留言区分享。</li>
</ol><p>期待你在留言区与我交流讨论，我们下一讲见。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/25/06/13/e4f9f79b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>你赖东东不错嘛</span>
  </div>
  <div class="_2_QraFYR_0">Q1:构建Request时将API版本作为可选参数传入</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 优秀！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-11 22:49:17</div>
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
  <div class="_2_QraFYR_0">老师讲的太好了，<br>请教老师一个问题：<br>sdk中 日志应该如何设计比较好，<br>我看 阿里云是 默认了一个实现，用的标准log。<br>这样好吗， 如果用户使用的是  zap，那是不是得分文件，用同一个文件会不会冲突。<br>感觉都不优雅</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我感觉有2个思路：<br>1. 直接返回错误，不打印日志。<br>2. 如果SDK中需要打印日志，在创建SDK Client时，添加一个可选的Logger接口，将程序中的logger传进入。<br><br>建议1，感觉SDK没必要打印日志，因为SDK功能其实比较简单，就是组合参数、发送HTTP请求、返回数据。不会封装复杂的逻辑。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-06 16:07:13</div>
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
  <div class="_2_QraFYR_0">sdk为服务使用方提供了方便的同时，也为服务提供方省去很多不必要的沟通培训成本。<br>文中介绍了go sdk的目录结构，架构和云厂商常用的设计实现方案。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-17 23:36:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/3e/89/ccc2ebd9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>陈晓涛</span>
  </div>
  <div class="_2_QraFYR_0">openapi-generator是个非常不错的项目：https:&#47;&#47;github.com&#47;openapitools&#47;openapi-generator，支持生成几十种客户端语言，安装简单，使用简单，生成的代码质量高，还有特别详细的markdown使用文档，超级推荐</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢分享，后面看下能否切换到这种方式</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-19 17:38:39</div>
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
  <div class="_2_QraFYR_0">总结：<br>SDK的命名规范、目录结构、以及各个云厂商的逻辑架构。<br>云厂商的SDK的实现分为两层：API层和基础层。<br>API 层实现一个Client对象，每个方法对应了一个API接口<br>基础层：主要负责，请求的Marshall、Unmarshal、签名等功能。<br>API 层的 Client 通过匿名的方式继承了基础层的 Client。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 总结的到位！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-04 13:40:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKotsBr2icbYNYlRSlicGUD1H7lulSTQUAiclsEz9gnG5kCW9qeDwdYtlRMXic3V6sj9UrfKLPJnQojag/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ppd0705</span>
  </div>
  <div class="_2_QraFYR_0">云厂商python版本sdk感觉代码质量ucloud最好</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢分享！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-31 18:43:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/9b/e5/bd0be5c3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>⁶₆⁶₆⁶₆</span>
  </div>
  <div class="_2_QraFYR_0">你讲义里面提供的客户端调用示例执行失败了，也就是你https:&#47;&#47;github.com&#47;marmotedu&#47;medu-sdk-go里面提供的那个示例代码，错误提示如下，提示找不到对应的包，但是明明已经拉下来了呀，没看出原因，希望大佬能解答下。<br><br>[root@dev sdk]# go mod init sdk<br>go: creating new go.mod: module sdk<br>go: to add module requirements and sums:<br>        go mod tidy<br>[root@dev sdk]# go mod tidy<br>go: finding module for package github.com&#47;marmotedu&#47;medu-sdk-go&#47;services&#47;iam<br>go: finding module for package github.com&#47;marmotedu&#47;medu-sdk-go&#47;sdk<br>go: finding module for package github.com&#47;ory&#47;ladon<br>go: found github.com&#47;marmotedu&#47;medu-sdk-go&#47;sdk in github.com&#47;marmotedu&#47;medu-sdk-go v1.0.0<br>go: found github.com&#47;ory&#47;ladon in github.com&#47;ory&#47;ladon v1.2.0<br>go: finding module for package github.com&#47;marmotedu&#47;medu-sdk-go&#47;services&#47;iam<br>sdk imports<br>        github.com&#47;marmotedu&#47;medu-sdk-go&#47;services&#47;iam: module github.com&#47;marmotedu&#47;medu-sdk-go@latest found (v1.0.0), but does not contain package github.com&#47;marmotedu&#47;medu-sdk-go&#47;services&#47;iam<br>[root@dev sdk]#<br>[root@dev sdk]# ll &#47;root&#47;workspace&#47;golang&#47;pkg&#47;mod&#47;github.com&#47;marmotedu&#47;<br>total 8<br>dr-xr-xr-x  6 root root  185 Sep 16 01:16 api@v1.0.1<br>dr-xr-xr-x  3 root root  138 Sep 25 17:48 component-base@v0.0.2<br>dr-xr-xr-x  3 root root  138 Sep 22 22:43 component-base@v1.0.0<br>dr-xr-xr-x  3 root root  138 Sep 16 01:16 component-base@v1.0.1<br>dr-xr-xr-x  2 root root 4096 Sep 22 22:43 errors@v0.0.1<br>dr-xr-xr-x  2 root root 4096 Sep 16 01:16 errors@v1.0.2<br>dr-xr-xr-x 18 root root  257 Sep 15 22:57 gopractise-demo@v0.0.1<br>dr-xr-xr-x  5 root root  261 Sep 16 01:16 log@v0.0.1<br>dr-xr-xr-x  8 root root  233 Sep 16 01:18 marmotedu-sdk-go@v1.0.2-0.20210528170801-2c91b80cb4cf<br>dr-xr-xr-x  5 root root  112 Sep 25 17:48 medu-sdk-go@v1.0.0<br>[root@dev sdk]#<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: go mod init sdk<br><br>上面的命令换成：go mod init github.com&#47;marmotedu&#47;medu-sdk-go<br><br>因为你init的报名是sdk，所以找不到当前包下的iam目录</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-25 18:17:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/iaxgvyIjNFomptQ9qBk4iaakYOS1XojYHDp48TXt1kX9DxTkKuR2UXGTyhG1liahib6E4BLF12ia6mic2pF0t4ECeZIQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Joeforfun</span>
  </div>
  <div class="_2_QraFYR_0">很赞，准备给没有go-sdk的某云厂商写个简单的demo</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-31 15:32:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_d71d64</span>
  </div>
  <div class="_2_QraFYR_0">sdk的api和前端网页的api如何区分开来呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 通过 UserAgent Header 来区分</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-18 16:53:28</div>
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
  <div class="_2_QraFYR_0">前端比如安卓的sdk也是这个设计思路吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我理解思路都是一样的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-26 22:54:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2a/0a/e4/c576d62b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>阿波罗尼斯圆</span>
  </div>
  <div class="_2_QraFYR_0">doc.go是干啥的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: doc.go中，是包级别的注释，例如可以在里面写上包是如何使用的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-17 11:26:38</div>
  </div>
</div>
</div>
</li>
</ul>