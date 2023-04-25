<audio title="34 _ SDK 设计（下）：IAM项目Go SDK设计和实现" src="https://static001.geekbang.org/resource/audio/7b/01/7b72394f7175da9db183106b51076101.mp3" controls="controls"></audio> 
<p>你好，我是孔令飞。</p><p>上一讲，我介绍了公有云厂商普遍采用的SDK设计方式。其实，还有一些比较优秀的SDK设计方式，比如 Kubernetes的 <a href="https://github.com/kubernetes/client-go">client-go</a> SDK设计方式。IAM项目参考client-go，也实现了client-go风格的SDK：<a href="https://github.com/marmotedu/marmotedu-sdk-go">marmotedu-sdk-go</a>。</p><p>和 <a href="https://time.geekbang.org/column/article/406389">33讲</a> 介绍的SDK设计方式相比，client-go风格的SDK具有以下优点：</p><ul>
<li>大量使用了Go interface特性，将接口的定义和实现解耦，可以支持多种实现方式。</li>
<li>接口调用层级跟资源的层级相匹配，调用方式更加友好。</li>
<li>多版本共存。</li>
</ul><p>所以，我更推荐你使用marmotedu-sdk-go。接下来，我们就来看下marmotedu-sdk-go是如何设计和实现的。</p><h2>marmotedu-sdk-go设计</h2><p>和medu-sdk-go相比，marmotedu-sdk-go的设计和实现要复杂一些，但功能更强大，使用体验也更好。</p><p>这里，我们先来看一个使用SDK调用iam-authz-server  <code>/v1/authz</code> 接口的示例，代码保存在<a href="https://github.com/marmotedu/marmotedu-sdk-go/blob/v1.0.3/examples/authz_clientset/main.go"> marmotedu-sdk-go/examples/authz_clientset/main.go</a>文件中：</p><pre><code class="language-go">package main

import (
	"context"
	"flag"
	"fmt"
	"path/filepath"

	"github.com/ory/ladon"

	metav1 "github.com/marmotedu/component-base/pkg/meta/v1"
	"github.com/marmotedu/component-base/pkg/util/homedir"

	"github.com/marmotedu/marmotedu-sdk-go/marmotedu"
	"github.com/marmotedu/marmotedu-sdk-go/tools/clientcmd"
)

func main() {
	var iamconfig *string
	if home := homedir.HomeDir(); home != "" {
		iamconfig = flag.String(
			"iamconfig",
			filepath.Join(home, ".iam", "config"),
			"(optional) absolute path to the iamconfig file",
		)
	} else {
		iamconfig = flag.String("iamconfig", "", "absolute path to the iamconfig file")
	}
	flag.Parse()

	// use the current context in iamconfig
	config, err := clientcmd.BuildConfigFromFlags("", *iamconfig)
	if err != nil {
		panic(err.Error())
	}

	// create the clientset
	clientset, err := marmotedu.NewForConfig(config)
	if err != nil {
		panic(err.Error())
	}

	request := &amp;ladon.Request{
		Resource: "resources:articles:ladon-introduction",
		Action:&nbsp; &nbsp;"delete",
		Subject:&nbsp; "users:peter",
		Context: ladon.Context{
			"remoteIP": "192.168.0.5",
		},
	}

	// Authorize the request
	fmt.Println("Authorize request...")
	ret, err := clientset.Iam().AuthzV1().Authz().Authorize(context.TODO(), request, metav1.AuthorizeOptions{})
	if err != nil {
		panic(err.Error())
	}

	fmt.Printf("Authorize response: %s.\n", ret.ToString())
}
</code></pre><!-- [[[read_end]]] --><p>在上面的代码示例中，包含了下面的操作。</p><ul>
<li>首先，调用 <code>BuildConfigFromFlags</code> 函数，创建出SDK的配置实例config；</li>
<li>接着，调用 <code>marmotedu.NewForConfig(config)</code> 创建了IAM项目的客户端 <code>clientset</code> ;</li>
<li>最后，调用以下代码请求 <code>/v1/authz</code> 接口执行资源授权请求：</li>
</ul><pre><code class="language-go">ret, err := clientset.Iam().AuthzV1().Authz().Authorize(context.TODO(), request, metav1.AuthorizeOptions{})&nbsp; &nbsp;&nbsp;
if err != nil {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
    panic(err.Error())&nbsp; &nbsp;&nbsp;
}&nbsp; &nbsp;&nbsp;

fmt.Printf("Authorize response: %s.\n", ret.ToString())
</code></pre><p>调用格式为<code>项目客户端.应用客户端.服务客户端.资源名.接口</code> 。</p><p>所以，上面的代码通过创建项目级别的客户端、应用级别的客户端和服务级别的客户端，来调用资源的API接口。接下来，我们来看下如何创建这些客户端。</p><h3>marmotedu-sdk-go客户端设计</h3><p>在讲客户端创建之前，我们先来看下客户端的设计思路。</p><p>Go项目的组织方式是有层级的：<strong>Project -&gt; Application -&gt; Service</strong>。marmotedu-sdk-go很好地体现了这种层级关系，使得SDK的调用更加易懂、易用。marmotedu-sdk-go的层级关系如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/3a/21/3a4721afa7fe365c0954019087d82021.jpg?wh=2248x1043" alt=""></p><p>marmotedu-sdk-go定义了3类接口，分别代表了项目、应用和服务级别的API接口：</p><pre><code class="language-go">// 项目级别的接口
type Interface interface {
    Iam() iam.IamInterface
    Tms() tms.TmsInterface
}

// 应用级别的接口
type IamInterface interface {
    APIV1() apiv1.APIV1Interface
    AuthzV1() authzv1.AuthzV1Interface
}

// 服务级别的接口
type APIV1Interface interface {
    RESTClient() rest.Interface
    SecretsGetter
    UsersGetter
    PoliciesGetter
}

// 资源级别的客户端
type SecretsGetter interface {
    Secrets() SecretInterface
}

// 资源的接口定义
type SecretInterface interface {
    Create(ctx context.Context, secret *v1.Secret, opts metav1.CreateOptions) (*v1.Secret, error)
    Update(ctx context.Context, secret *v1.Secret, opts metav1.UpdateOptions) (*v1.Secret, error)
    Delete(ctx context.Context, name string, opts metav1.DeleteOptions) error
    DeleteCollection(ctx context.Context, opts metav1.DeleteOptions, listOpts metav1.ListOptions) error
    Get(ctx context.Context, name string, opts metav1.GetOptions) (*v1.Secret, error)
    List(ctx context.Context, opts metav1.ListOptions) (*v1.SecretList, error)
    SecretExpansion
}
</code></pre><p><code>Interface</code> 代表了项目级别的接口，里面包含了 <code>Iam</code> 和 <code>Tms</code>  两个应用； <code>IamInterface</code> 代表了应用级别的接口，里面包含了api（iam-apiserver）和authz（iam-authz-server）两个服务级别的接口。api和authz服务中，又包含了各自服务中REST资源的CURD接口。</p><p>marmotedu-sdk-go通过 <code>XxxV1</code> 这种命名方式来支持不同版本的API接口，好处是可以在程序中同时调用同一个API接口的不同版本，例如：</p><p><code>clientset.Iam().AuthzV1().Authz().Authorize()</code>  、<code>clientset.Iam().AuthzV2().Authz().Authorize()</code> 分别调用了 <code>/v1/authz</code> 和 <code>/v2/authz</code>  两个版本的API接口。</p><p>上述关系也可以从目录结构中反映出来，marmotedu-sdk-go目录设计如下（只列出了一些重要的文件）：</p><pre><code class="language-bash">├── examples                        # 存放SDK的使用示例
├── Makefile                        # 管理SDK源码，静态代码检查、代码格式化、测试、添加版权信息等
├── marmotedu
│   ├── clientset.go                # clientset实现，clientset中包含多个应用，多个服务的API接口
│   ├── fake                        # clientset的fake实现，主要用于单元测试
│   └── service                     # 按应用进行分类，存放应用中各服务API接口的具体实现
│       ├── iam                     # iam应用的API接口实现，包含多个服务
│       │   ├── apiserver           # iam应用中，apiserver服务的API接口，包含多个版本
│       │   │   └── v1              # apiserver v1版本API接口
│       │   ├── authz               # iam应用中，authz服务的API接口
│       │   │   └── v1              # authz服务v1版本接口
│       │   └── iam_client.go       # iam应用的客户端，包含了apiserver和authz 2个服务的客户端
│       └── tms                     # tms应用的API接口实现
├── pkg                             # 存放一些共享包，可对外暴露
├── rest                            # HTTP请求的底层实现
├── third_party                     # 存放修改过的第三方包，例如：gorequest
└── tools
    └── clientcmd                   # 一些函数用来帮助创建rest.Config配置
</code></pre><p>每种类型的客户端，都可以通过以下相似的方式来创建：</p><pre><code class="language-go">config, err := clientcmd.BuildConfigFromFlags("", "/root/.iam/config")
clientset, err := xxxx.NewForConfig(config)
</code></pre><p><code>/root/.iam/config</code> 为配置文件，里面包含了服务的地址和认证信息。<code>BuildConfigFromFlags</code> 函数加载配置文件，创建并返回 <code>rest.Config</code> 类型的配置变量，并通过 <code>xxxx.NewForConfig</code> 函数创建需要的客户端。<code>xxxx</code> 是所在层级的client包，例如 iam、tms。</p><p>marmotedu-sdk-go客户端定义了3类接口，这可以带来两个好处。</p><p>第一，API接口调用格式规范，层次清晰，可以使API接口调用更加清晰易记。</p><p>第二，可以根据需要，自行选择客户端类型，调用灵活。举个例子，在A服务中需要同时用到iam-apiserver 和 iam-authz-server提供的接口，就可以创建应用级别的客户端IamClient，然后通过 <code>iamclient.APIV1()</code> 和 <code>iamclient.AuthzV1()</code> ，来切换调用不同服务的API接口。</p><p>接下来，我们来看看如何创建三个不同级别的客户端。</p><h3>项目级别客户端创建</h3><p><code>Interface</code> 对应的客户端实现为<a href="https://github.com/marmotedu/marmotedu-sdk-go/blob/v1.0.2/marmotedu/clientset.go#L20-L23">Clientset</a>，所在的包为 <a href="https://github.com/marmotedu/marmotedu-sdk-go/tree/v1.0.2/marmotedu">marmotedu-sdk-go/marmotedu</a>，Clientset客户端的创建方式为：</p><pre><code class="language-go">config, err := clientcmd.BuildConfigFromFlags("", "/root/.iam/config")
clientset, err := marmotedu.NewForConfig(config)
</code></pre><p>调用方式为 <code>clientset.应用.服务.资源名.接口</code> ，例如：</p><pre><code class="language-go">rsp, err := clientset.Iam().AuthzV1().Authz().Authorize()
</code></pre><p>参考示例为 <a href="https://github.com/marmotedu/marmotedu-sdk-go/blob/v1.0.3/examples/authz_clientset/main.go">marmotedu-sdk-go/examples/authz_clientset/main.go</a>。</p><h3>应用级别客户端创建</h3><p><code>IamInterface</code> 对应的客户端实现为<a href="https://github.com/marmotedu/marmotedu-sdk-go/blob/v1.0.2/marmotedu/service/iam/iam_client.go#L22-L25">IamClient</a>，所在的包为 <a href="https://github.com/marmotedu/marmotedu-sdk-go/tree/v1.0.2/marmotedu/service/iam">marmotedu-sdk-go/marmotedu/service/iam</a>，IamClient客户端的创建方式为：</p><pre><code class="language-go">config, err := clientcmd.BuildConfigFromFlags("", "/root/.iam/config")
iamclient,, err := iam.NewForConfig(config)
</code></pre><p>调用方式为 <code>iamclient.服务.资源名.接口</code> ，例如：</p><pre><code class="language-go">rsp, err := iamclient.AuthzV1().Authz().Authorize()
</code></pre><p>参考示例为 <a href="https://github.com/marmotedu/marmotedu-sdk-go/blob/v1.0.2/examples/authz_iam/main.go">marmotedu-sdk-go/examples/authz_iam/main.go</a>。</p><h3>服务级别客户端创建</h3><p><code>AuthzV1Interface</code> 对应的客户端实现为<a href="https://github.com/marmotedu/marmotedu-sdk-go/blob/v1.0.2/marmotedu/service/iam/authz/v1/authz_client.go#L21-L23">AuthzV1Client</a>，所在的包为 <a href="https://github.com/marmotedu/marmotedu-sdk-go/tree/v1.0.2/marmotedu/service/iam/authz/v1">marmotedu-sdk-go/marmotedu/service/iam/authz/v1</a>，AuthzV1Client客户端的创建方式为：</p><pre><code class="language-go">config, err := clientcmd.BuildConfigFromFlags("", "/root/.iam/config")
client, err := v1.NewForConfig(config)
</code></pre><p>调用方式为 <code>client.资源名.接口</code> ，例如：</p><pre><code class="language-go">rsp, err := client.Authz().Authorize()
</code></pre><p>参考示例为 <a href="https://github.com/marmotedu/marmotedu-sdk-go/blob/v1.0.3/examples/authz/main.go">marmotedu-sdk-go/examples/authz/main.go</a>。</p><p>上面我介绍了marmotedu-sdk-go的客户端创建方法，接下来我们再来看下，这些客户端具体是如何执行REST API请求的。</p><h2>marmotedu-sdk-go的实现</h2><p>marmotedu-sdk-go的实现和medu-sdk-go一样，也是采用分层结构，分为API层和基础层。如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/c4/b2/c40439c97998a01758923394116c33b2.jpg?wh=2248x2097" alt=""></p><p><a href="https://github.com/marmotedu/marmotedu-sdk-go/blob/v1.0.2/rest/client.go#L95-L105">RESTClient</a>是整个SDK的核心，RESTClient向下通过调用<a href="https://github.com/marmotedu/marmotedu-sdk-go/blob/v1.0.2/rest/request.go#L28-L50">Request</a>模块，来完成HTTP请求方法、请求路径、请求体、认证信息的构建。Request模块最终通过调用<a href="https://github.com/parnurzeal/gorequest">gorequest</a>包提供的方法，完成HTTP的POST、PUT、GET、DELETE等请求，获取HTTP返回结果，并解析到指定的结构体中。RESTClient向上提供 <code>Post()</code> 、 <code>Put()</code> 、 <code>Get()</code> 、 <code>Delete()</code> 等方法来供客户端完成HTTP请求。</p><p>marmotedu-sdk-go提供了两类客户端，分别是RESTClient客户端和基于RESTClient封装的客户端。</p><ul>
<li>RESTClient：Raw类型的客户端，可以通过指定HTTP的请求方法、请求路径、请求参数等信息，直接发送HTTP请求，例如 <code>client.Get().AbsPath("/version").Do().Into()</code> 。</li>
<li>基于RESTClient封装的客户端：例如AuthzV1Client、APIV1Client等，执行特定REST资源、特定API接口的请求，方便开发者调用。</li>
</ul><p>接下来，我们具体看下如何创建RESTClient客户端，以及Request模块的实现。</p><h3>RESTClient客户端实现</h3><p>我通过下面两个步骤，实现了RESTClient客户端。</p><p><strong>第一步，创建</strong><a href="https://github.com/marmotedu/marmotedu-sdk-go/blob/v1.0.2/rest/config.go#L29-L60">rest.Config</a><strong>类型的变量。</strong></p><p><a href="https://github.com/marmotedu/marmotedu-sdk-go/blob/v1.0.2/tools/clientcmd/client_config.go#L190-L203">BuildConfigFromFlags</a>函数通过加载yaml格式的配置文件，来创建 <code>rest.Config</code> 类型的变量，加载的yaml格式配置文件内容为：</p><pre><code class="language-yaml">apiVersion: v1
user:
  #token: # JWT Token
  username: admin # iam 用户名
  password: Admin@2020 # iam 密码
  #secret-id: # 密钥 ID
  #secret-key: # 密钥 Key
  client-certificate: /home/colin/.iam/cert/admin.pem # 用于 TLS 的客户端证书文件路径
  client-key: /home/colin/.iam/cert/admin-key.pem # 用于 TLS 的客户端 key 文件路径
  #client-certificate-data:
  #client-key-data:

server:
  address: https://127.0.0.1:8443 # iam api-server 地址
  timeout: 10s # 请求 api-server 超时时间
  #max-retries: # 最大重试次数，默认为 0
  #retry-interval: # 重试间隔，默认为 1s
  #tls-server-name: # TLS 服务器名称
  #insecure-skip-tls-verify: # 设置为 true 表示跳过 TLS 安全验证模式，将使得 HTTPS 连接不安全
  certificate-authority: /home/colin/.iam/cert/ca.pem # 用于 CA 授权的 cert 文件路径
  #certificate-authority-data:
</code></pre><p>在配置文件中，我们可以指定服务的地址、用户名/密码、密钥、TLS证书、超时时间、重试次数等信息。</p><p>创建方法如下：</p><pre><code class="language-go">config, err := clientcmd.BuildConfigFromFlags("", *iamconfig)&nbsp; &nbsp;&nbsp;
if err != nil {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; panic(err.Error())&nbsp; &nbsp;&nbsp;
}&nbsp;&nbsp;
</code></pre><p>这里的代码中，<code>*iamconfig</code> 是yaml格式的配置文件路径。<code>BuildConfigFromFlags</code> 函数中，调用<a href="https://github.com/marmotedu/marmotedu-sdk-go/blob/v1.0.3/tools/clientcmd/loader.go#L32-L56">LoadFromFile</a>函数来解析yaml配置文件。LoadFromFile最终是通过 <code>yaml.Unmarshal</code> 的方式来解析yaml格式的配置文件的。</p><p><strong>第二步，根据rest.Config类型的变量，创建RESTClient客户端。</strong></p><p>通过<a href="https://github.com/marmotedu/marmotedu-sdk-go/blob/v1.0.2/rest/config.go#L191-L237">RESTClientFor</a>函数来创建RESTClient客户端：</p><pre><code class="language-go">func RESTClientFor(config *Config) (*RESTClient, error) {
    ...
    baseURL, versionedAPIPath, err := defaultServerURLFor(config)
    if err != nil {
        return nil, err
    }

    // Get the TLS options for this client config
    tlsConfig, err := TLSConfigFor(config)
    if err != nil {
        return nil, err
    }

    // Only retry when get a server side error.
    client := gorequest.New().TLSClientConfig(tlsConfig).Timeout(config.Timeout).
        Retry(config.MaxRetries, config.RetryInterval, http.StatusInternalServerError)
    // NOTICE: must set DoNotClearSuperAgent to true, or the client will clean header befor http.Do
    client.DoNotClearSuperAgent = true

    ...

    clientContent := ClientContentConfig{
        Username:           config.Username,
        Password:           config.Password,
        SecretID:           config.SecretID,
        SecretKey:          config.SecretKey,
        ...
    }

    return NewRESTClient(baseURL, versionedAPIPath, clientContent, client)
}
</code></pre><p>RESTClientFor函数调用<a href="https://github.com/marmotedu/marmotedu-sdk-go/blob/v1.0.2/rest/url_utils.go#L69-L81">defaultServerURLFor(config)</a>生成基本的HTTP请求路径：baseURL=http://127.0.0.1:8080，versionedAPIPath=/v1。然后，通过<a href="https://github.com/marmotedu/marmotedu-sdk-go/blob/v1.0.2/rest/config.go#L241-L298">TLSConfigFor</a>函数生成TLS配置，并调用 <code>gorequest.New()</code> 创建gorequest客户端，将客户端配置信息保存在变量中。最后，调用<a href="https://github.com/marmotedu/marmotedu-sdk-go/blob/v1.0.2/rest/client.go#L109-L130">NewRESTClient</a>函数创建RESTClient客户端。</p><p>RESTClient客户端提供了以下方法，来供调用者完成HTTP请求：</p><pre><code class="language-go">func (c *RESTClient) APIVersion() scheme.GroupVersion
func (c *RESTClient) Delete() *Request
func (c *RESTClient) Get() *Request
func (c *RESTClient) Post() *Request
func (c *RESTClient) Put() *Request
func (c *RESTClient) Verb(verb string) *Request
</code></pre><p>可以看到，RESTClient提供了 <code>Delete</code> 、 <code>Get</code> 、 <code>Post</code> 、 <code>Put</code> 方法，分别用来执行HTTP的DELETE、GET、POST、PUT方法，提供的 <code>Verb</code> 方法可以灵活地指定HTTP方法。这些方法都返回了 <code>Request</code> 类型的变量。<code>Request</code> 类型的变量提供了一些方法，用来完成具体的HTTP请求，例如：</p><pre><code class="language-go">  type Response struct {
&nbsp; &nbsp; Allowed bool&nbsp; &nbsp;`json:"allowed"`
&nbsp; &nbsp; Denied&nbsp; bool&nbsp; &nbsp;`json:"denied,omitempty"`
&nbsp; &nbsp; Reason&nbsp; string `json:"reason,omitempty"`
&nbsp; &nbsp; Error&nbsp; &nbsp;string `json:"error,omitempty"`
}

func (c *authz) Authorize(ctx context.Context, request *ladon.Request, opts metav1.AuthorizeOptions) (result *Response, err error) {
&nbsp; &nbsp; result = &amp;Response{}&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
&nbsp; &nbsp; err = c.client.Post().
&nbsp; &nbsp; &nbsp; &nbsp; Resource("authz").
&nbsp; &nbsp; &nbsp; &nbsp; VersionedParams(opts).
&nbsp; &nbsp; &nbsp; &nbsp; Body(request).
&nbsp; &nbsp; &nbsp; &nbsp; Do(ctx).
&nbsp; &nbsp; &nbsp; &nbsp; Into(result)

&nbsp; &nbsp; return
}
</code></pre><p>上面的代码中， <code>c.client</code> 是RESTClient客户端，通过调用RESTClient客户端的 <code>Post</code> 方法，返回了 <code>*Request</code> 类型的变量。</p><p><code>*Request</code> 类型的变量提供了 <code>Resource</code> 和 <code>VersionedParams</code> 方法，来构建请求HTTP URL中的路径 <code>/v1/authz</code> ；通过 <code>Body</code> 方法，指定了HTTP请求的Body。</p><p>到这里，我们分别构建了HTTP请求需要的参数：HTTP Method、请求URL、请求Body。所以，之后就可以调用 <code>Do</code> 方法来执行HTTP请求，并将返回结果通过 <code>Into</code> 方法保存在传入的result变量中。</p><h3>Request模块实现</h3><p>RESTClient客户端的方法会返回Request类型的变量，Request类型的变量提供了一系列的方法用来构建HTTP请求参数，并执行HTTP请求。</p><p>所以，Request模块可以理解为最底层的通信层，我们来看下Request模块具体是如何完成HTTP请求的。</p><p>我们先来看下<a href="https://github.com/marmotedu/marmotedu-sdk-go/blob/v1.0.3/rest/request.go#L28-L50">Request结构体</a>的定义：</p><pre><code class="language-go">type RESTClient struct {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
&nbsp; &nbsp; // base is the root URL for all invocations of the client&nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; base *url.URL&nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; // group stand for the client group, eg: iam.api, iam.authz&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
&nbsp; &nbsp; group string&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; // versionedAPIPath is a path segment connecting the base URL to the resource root&nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; versionedAPIPath string&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; // content describes how a RESTClient encodes and decodes responses.&nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; content ClientContentConfig&nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; Client&nbsp; *gorequest.SuperAgent&nbsp; &nbsp;&nbsp;
}

type Request struct {
	c *RESTClient

	timeout time.Duration

	// generic components accessible via method setters
	verb&nbsp; &nbsp; &nbsp; &nbsp;string
	pathPrefix string
	subpath&nbsp; &nbsp; string
	params&nbsp; &nbsp; &nbsp;url.Values
	headers&nbsp; &nbsp; http.Header

	// structural elements of the request that are part of the IAM API conventions
	// namespace&nbsp; &nbsp; string
	// namespaceSet bool
	resource&nbsp; &nbsp; &nbsp;string
	resourceName string
	subresource&nbsp; string

	// output
	err&nbsp; error
	body interface{}
}&nbsp;&nbsp;
</code></pre><p>再来看下Request结构体提供的方法：</p><pre><code class="language-go">func (r *Request) AbsPath(segments ...string) *Request
func (r *Request) Body(obj interface{}) *Request
func (r *Request) Do(ctx context.Context) Result
func (r *Request) Name(resourceName string) *Request
func (r *Request) Param(paramName, s string) *Request
func (r *Request) Prefix(segments ...string) *Request
func (r *Request) RequestURI(uri string) *Request
func (r *Request) Resource(resource string) *Request
func (r *Request) SetHeader(key string, values ...string) *Request
func (r *Request) SubResource(subresources ...string) *Request
func (r *Request) Suffix(segments ...string) *Request
func (r *Request) Timeout(d time.Duration) *Request
func (r *Request) URL() *url.URL
func (r *Request) Verb(verb string) *Request
func (r *Request) VersionedParams(v interface{}) *Request
</code></pre><p>通过Request结构体的定义和使用方法，我们不难猜测出：Request模块通过 <code>Name</code> 、 <code>Resource</code> 、 <code>Body</code> 、 <code>SetHeader</code> 等方法来设置Request结构体中的各个字段。这些字段最终用来构建出一个HTTP请求，并通过 <code>Do</code> 方法来执行HTTP请求。</p><p>那么，如何构建并执行一个HTTP请求呢？我们可以通过以下5步，来构建并执行HTTP请求：</p><ol>
<li>构建HTTP URL；</li>
<li>构建HTTP Method；</li>
<li>构建HTTP Body；</li>
<li>执行HTTP请求；</li>
<li>保存HTTP返回结果。</li>
</ol><p>接下来，我们就来具体看下Request模块是如何构建这些请求参数，并发送HTTP请求的。</p><p><strong>第一步，构建HTTP URL。</strong></p><p>首先，通过<a href="https://github.com/marmotedu/marmotedu-sdk-go/blob/v1.0.3/rest/url_utils.go#L69-L81">defaultServerURLFor</a>函数返回了<code>http://iam.api.marmotedu.com:8080</code> 和 <code>/v1</code> ，并将二者分别保存在了Request类型结构体变量中 <code>c</code> 字段的 <code>base</code> 字段和 <code>versionedAPIPath</code> 字段中。</p><p>通过 <a href="https://github.com/marmotedu/marmotedu-sdk-go/blob/v1.0.3/rest/request.go#L379-L416">Do</a> 方法执行HTTP时，会调用<a href="https://github.com/marmotedu/marmotedu-sdk-go/blob/v1.0.3/rest/request.go#L392">r.URL()</a>方法来构建请求URL。 <code>r.URL</code> 方法中，通过以下代码段构建了HTTP请求URL：</p><pre><code class="language-go">func (r *Request) URL() *url.URL {
&nbsp; &nbsp; p := r.pathPrefix
&nbsp; &nbsp; if len(r.resource) != 0 {
&nbsp; &nbsp; &nbsp; &nbsp; p = path.Join(p, strings.ToLower(r.resource))
&nbsp; &nbsp; }

&nbsp; &nbsp; if len(r.resourceName) != 0 || len(r.subpath) != 0 || len(r.subresource) != 0 {
&nbsp; &nbsp; &nbsp; &nbsp; p = path.Join(p, r.resourceName, r.subresource, r.subpath)
&nbsp; &nbsp; }
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
&nbsp; &nbsp; finalURL := &amp;url.URL{}
&nbsp; &nbsp; if r.c.base != nil {
&nbsp; &nbsp; &nbsp; &nbsp; *finalURL = *r.c.bas
&nbsp; &nbsp; }
&nbsp;
&nbsp; &nbsp; finalURL.Path = p
    ...&nbsp; &nbsp;&nbsp;
}
</code></pre><p><code>p := r.pathPrefix</code> 和 <code>r.c.base</code> ，是通过 <code>defaultServerURLFor</code> 调用返回的 <code>v1</code> 和 <code>http://iam.api.marmotedu.com:8080</code> 来构建的。</p><p><code>resourceName</code> 通过 <code>func (r *Request) Resource(resource string) *Request</code> 来指定，例如 <code>authz</code> 。</p><p>所以，最终我们构建的请求URL为 <code>http://iam.api.marmotedu.com:8080/v1/authz</code> 。</p><p><strong>第二步，构建HTTP Method。</strong></p><p>HTTP Method通过RESTClient提供的 <code>Post</code> 、<code>Delete</code> 、<code>Get</code> 等方法来设置，例如：</p><pre><code class="language-go">func (c *RESTClient) Post() *Request {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
&nbsp; &nbsp; return c.Verb("POST")&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;
}

func (c *RESTClient) Verb(verb string) *Request {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; return NewRequest(c).Verb(verb)&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;
}
</code></pre><p><code>NewRequest(c).Verb(verb)</code> 最终设置了Request结构体的 <code>verb</code> 字段，供 <code>Do</code> 方法使用。</p><p><strong>第三步，构建HTTP Body。</strong></p><p>HTTP Body通过Request结构体提供的Body方法来指定：</p><pre><code class="language-go">func (r *Request) Body(obj interface{}) *Request {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; if v := reflect.ValueOf(obj); v.Kind() == reflect.Struct {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; &nbsp; &nbsp; r.SetHeader("Content-Type", r.c.content.ContentType)&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
&nbsp; &nbsp; r.body = obj&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
&nbsp; &nbsp; return r&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
}&nbsp;
</code></pre><p><strong>第四步，执行HTTP请求。</strong></p><p>通过Request结构体提供的Do方法来执行具体的HTTP请求，代码如下：</p><pre><code class="language-go">func (r *Request) Do(ctx context.Context) Result {
	client := r.c.Client
	client.Header = r.headers

	if r.timeout &gt; 0 {
		var cancel context.CancelFunc
		ctx, cancel = context.WithTimeout(ctx, r.timeout)

		defer cancel()
	}

	client.WithContext(ctx)

	resp, body, errs := client.CustomMethod(r.verb, r.URL().String()).Send(r.body).EndBytes()
	if err := combineErr(resp, body, errs); err != nil {
		return Result{
			response: &amp;resp,
			err:&nbsp; &nbsp; &nbsp; err,
			body:&nbsp; &nbsp; &nbsp;body,
		}
	}

	decoder, err := r.c.content.Negotiator.Decoder()
	if err != nil {
		return Result{
			response: &amp;resp,
			err:&nbsp; &nbsp; &nbsp; err,
			body:&nbsp; &nbsp; &nbsp;body,
			decoder:&nbsp; decoder,
		}
	}

	return Result{
		response: &amp;resp,
		body:&nbsp; &nbsp; &nbsp;body,
		decoder:&nbsp; decoder,
	}
}
</code></pre><p>在Do方法中，使用了Request结构体变量中各个字段的值，通过 <code>client.CustomMethod</code> 来执行HTTP请求。 <code>client</code> 是 <code>*gorequest.SuperAgent</code> 类型的客户端。</p><p><strong>第五步，保存HTTP返回结果。</strong></p><p>通过Request结构体的 <code>Into</code> 方法来保存HTTP返回结果：</p><pre><code class="language-go">func (r Result) Into(v interface{}) error {
&nbsp; &nbsp; if r.err != nil {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; &nbsp; &nbsp; return r.Error()
&nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
&nbsp; &nbsp; if r.decoder == nil {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; &nbsp; &nbsp; return fmt.Errorf("serializer doesn't exist")
&nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
&nbsp; &nbsp; if err := r.decoder.Decode(r.body, &amp;v); err != nil {
&nbsp; &nbsp; &nbsp; &nbsp; return err&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; }&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
&nbsp; &nbsp; return nil&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;
}
</code></pre><p><code>r.body</code> 是在Do方法中，执行完HTTP请求后设置的，它的值为HTTP请求返回的Body。</p><h3>请求认证</h3><p>接下来，我再来介绍下marmotedu-sdk-go另外一个比较核心的功能：请求认证。</p><p>marmotedu-sdk-go支持两种认证方式：</p><ul>
<li>Basic认证：通过给请求添加 <code>Authorization: Basic xxxx</code> 来实现。</li>
<li>Bearer认证：通过给请求添加 <code>Authorization: Bearer xxxx</code> 来实现。这种方式又支持直接指定JWT Token，或者通过指定密钥对由SDK自动生成JWT Token。</li>
</ul><p>Basic认证和Bearer认证，我在 <a href="https://time.geekbang.org/column/article/398410">25讲</a>介绍过，你可以返回查看下。</p><p>认证头是RESTClient客户端发送HTTP请求时指定的，具体实现位于<a href="https://github.com/marmotedu/marmotedu-sdk-go/blob/v1.0.2/rest/request.go#L53-L102">NewRequest</a>函数中：</p><pre><code class="language-go">switch {
    case c.content.HasTokenAuth():
        r.SetHeader("Authorization", fmt.Sprintf("Bearer %s", c.content.BearerToken))
    case c.content.HasKeyAuth():
        tokenString := auth.Sign(c.content.SecretID, c.content.SecretKey, "marmotedu-sdk-go", c.group+".marmotedu.com")
        r.SetHeader("Authorization", fmt.Sprintf("Bearer %s", tokenString))
    case c.content.HasBasicAuth():
        // TODO: get token and set header
        r.SetHeader("Authorization", "Basic "+basicAuth(c.content.Username, c.content.Password))
}
</code></pre><p>上面的代码会根据配置信息，自动判断使用哪种认证方式。</p><h2>总结</h2><p>这一讲中，我介绍了Kubernetes client-go风格的SDK实现方式。和公有云厂商的SDK设计相比，client-go风格的SDK设计有很多优点。</p><p>marmotedu-sdk-go在设计时，通过接口实现了3类客户端，分别是项目级别的客户端、应用级别的客户端和服务级别的客户端。开发人员可以根据需要，自行创建客户端类型。</p><p>marmotedu-sdk-go通过<a href="https://github.com/marmotedu/marmotedu-sdk-go/blob/v1.0.2/rest/config.go#L191-L237">RESTClientFor</a>，创建了RESTClient类型的客户端，RESTClient向下通过调用<a href="https://github.com/marmotedu/marmotedu-sdk-go/blob/v1.0.2/rest/request.go#L28-L50">Request</a>模块，来完成HTTP请求方法、请求路径、请求体、认证信息的构建。Request模块最终通过调用<a href="https://github.com/parnurzeal/gorequest">gorequest</a>包提供的方法，完成HTTP的POST、PUT、GET、DELETE等请求，获取HTTP返回结果，并解析到指定的结构体中。RESTClient向上提供 <code>Post()</code> 、 <code>Put()</code> 、 <code>Get()</code> 、 <code>Delete()</code> 等方法，来供客户端完成HTTP请求。</p><h2>课后练习</h2><ol>
<li>阅读<a href="https://github.com/marmotedu/marmotedu-sdk-go/blob/v1.0.3/rest/url_utils.go#L69-L81">defaultServerURLFor</a>源码，思考下defaultServerURLFor是如何构建请求地址 <code>http://iam.api.marmotedu.com:8080</code> 和API版本 <code>/v1</code> 的。</li>
<li>使用<a href="https://github.com/parnurzeal/gorequest">gorequest</a>包，编写一个可以执行以下HTTP请求的示例：</li>
</ol><pre><code class="language-bash">curl -XPOST http://example.com/v1/user -d '{"username":"colin","address":"shenzhen"}'
</code></pre><p>欢迎你在留言区与我交流讨论，我们下一讲见。</p>
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
  <div class="_2_QraFYR_0">K8s client-go风格的sdk实现方式比公有云普遍采用的sdk更灵活，调用和扩展都更方便，还可以多版本共存。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-17 23:44:37</div>
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
  <div class="_2_QraFYR_0">这种分层设计好妙，可以按需创建某层客户端</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-02 00:10:26</div>
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
  <div class="_2_QraFYR_0">总结：<br>client-go 风格的SDK从项目、应用、服务 都有客户端，且上一层客户端封装了下层的客户端。<br>1. NewForConfig 操作，会将下层的客户端都创建出来。<br>2. clientset.Iam().AuthzV1() 就是指定客户端的过程。<br>3. 在要创建具体的资源时，客户端作为参数，动态创建一个资源的“客户端”。这是在服务客户端做的<br>4. 资源客户端拿到了“RESTClient”对象，开始操作资源。<br><br>在服务级别的接口，不但提供了 RESTClient() rest.Interface 来操作资源，也可以调用 AuthzGetter 封装的接口，更加友好。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 666</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-04 14:55:08</div>
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
  <div class="_2_QraFYR_0">K8s client-go风格确实比公有云的sdk风格有很多优势，目前项目基本都是采用公有云的方式，get到老师的点，学习了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-08 10:45:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/75/bc/e24e181e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Calvin</span>
  </div>
  <div class="_2_QraFYR_0">请问下 http 客户端使用第三方库的好吗？比如 go-resty</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-03 19:37:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/ff/c6/3586506e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>筱泉</span>
  </div>
  <div class="_2_QraFYR_0">老师￼，这个example代码怎么跑起来<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 服务部署起来。go run</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-18 19:47:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/51/84/5b7d4d95.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>冷峰</span>
  </div>
  <div class="_2_QraFYR_0">authz 默认使用的 jwt 认证， 这个例子中的认证方式使用的是 basic , 会导致认证失败</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢反馈，这个地方我考虑下怎么兼容下，不过不影响学习。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-29 19:34:48</div>
  </div>
</div>
</div>
</li>
</ul>