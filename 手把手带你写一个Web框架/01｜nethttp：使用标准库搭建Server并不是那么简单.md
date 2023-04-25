<audio title="01｜nethttp：使用标准库搭建Server并不是那么简单" src="https://static001.geekbang.org/resource/audio/05/47/05a5f2dfda387cfbded0fffaa7ff1a47.mp3" controls="controls"></audio> 
<p>你好，我是轩脉刃。欢迎加入我的课程，和我一起从0开始构建Web框架。</p><p>之前我简单介绍了整个课程的设计思路，也是搭建Web框架的学习路径，我们会先基于标准库搭建起Server，然后一步一步增加控制器、路由、中间件，最后完善封装和重启，在整章学完后，你就能建立起一套自己的Web框架了。</p><p>其实你熟悉Golang的话就会知道，用官方提供的 net/http 标准库搭建一个Web Server，是一件非常简单的事。我在面试的时候也发现，不少同学，在怎么搭怎么用的问题上，回答的非常溜，但是再追问一句为什么这个 Server 这么设计，涉及的 net/http 实现原理是什么? 一概不知。</p><p>这其实是非常危险的。<strong>实际工作中，我们会因为不了解底层原理，想当然的认为它的使用方式</strong>，直接导致在代码编写、应用调优的时候出现各种问题。</p><p>所以今天，我想带着你从最底层的 HTTP 协议开始，搞清楚Web Server本质，通过 net/http 代码库梳理 HTTP 服务的主流程脉络，先知其所以然，再搭建框架的Server结构。</p><p>之后，我们会基于今天分析的整个 HTTP 服务主流程原理继续开发。所以这节课你掌握的程度，对于后续内容的理解至关重要。</p><h2>Web Server 的本质</h2><!-- [[[read_end]]] --><p>既然要搭 Web Server，那我也先简单介绍一下，维基百科上是这么解释的，Web Server 是一个通过 HTTP 协议处理Web请求的计算机系统。这句话乍听有点绕口，我给你解释下。</p><p>HTTP 协议，在 OSI 网络体系结构中，是基于TCP/IP之上第七层应用层的协议，全称叫做超文本传输协议。啥意思？就是说HTTP协议传输的都是文本字符，只是这些字符是有规则排列的。这些字符的排列规则，就是一种约定，也就是协议。这个协议还有一个专门的描述文档，就是<a href="https://datatracker.ietf.org/doc/html/rfc2616">RFC 2616</a>。</p><p>对于 HTTP 协议，无论是请求还是响应，传输的消息体都可以分为两个部分：HTTP头部和 HTTP Body体。头部描述的一般是和业务无关但与传输相关的信息，比如请求地址、编码格式、缓存时长等；Body 里面主要描述的是与业务相关的信息。<img src="https://static001.geekbang.org/resource/image/cb/e0/cbbafb7ac6128b6e6f8bde0c983c7ae0.jpg?wh=1920x1080" alt=""></p><p>Web Server 的本质，实际上就是<strong>接收、解析</strong> HTTP 请求传输的文本字符，理解这些文本字符的指令，然后进行<strong>计算</strong>，再将返回值<strong>组织成</strong> HTTP 响应的文本字符，通过 TCP 网络<strong>传输回去</strong>。</p><p>理解了Web Server 干的事情，我们接下来继续看看在语言层面怎么实现。</p><h2>一定要用标准库吗</h2><p>对 Web Server 来说，Golang 提供了 net 库和 net/http 库，分别对应OSI的 TCP 层和 HTTP 层，它们两个负责的就是 HTTP 的接收和解析。</p><p>一般我们会使用 net/http 库解析 HTTP 消息体。但是可能会有人问，如果我想实现 Web 服务，可不可以不用 net/http 库呢？比如我直接用 net 库，逐字读取消息体，然后自己解析获取的传输字符。</p><p>答案是可以的，如果你有兼容其它协议、追求极致性能的需求，而且你有把握能按照 HTTP 的RFC 标准进行解析，那完全可以自己封装一个HTTP库。</p><p>其实在一些大厂中确实是这么做的，每当有一些通用的协议需求，比如一个服务既要支持 HTTP，又要支持 Protocol Buffers，又或者想要支持自定义的协议，那么他们就可能抛弃 HTTP 库，甚至抛弃 net 库，直接自己进行网络事件驱动，解析 HTTP 协议。</p><p>有个开源库，叫 <a href="https://github.com/valyala/fasthttp">FastHTTP</a>，它就是抛弃标准库 net/http 来实现的。作者为了追求极高的HTTP性能，自己封装了网络事件驱动，解析了HTTP协议。你感兴趣的话，可以去看看。</p><p>但是现在绝大部分的 Web 框架，都是基于 net/http 标准库的。我认为原因主要有两点：</p><ul>
<li>第一是<strong>相信官方开源的力量</strong>。自己实现HTTP协议的解析，不一定会比标准库实现得更好，即使当前标准库有一些不足之处，我们也都相信，随着开源贡献者越来越多，标准库也会最终达到完美。</li>
<li>第二是<strong>Web 服务架构的变化</strong>。随着容器化、Kubernetes 等技术的兴起，业界逐渐达成共识，单机并发性能并不是评判 Web 服务优劣的唯一标准了，易用性、扩展性也是底层库需要考量的。</li>
</ul><p>所以总体来说，net/http 标准库，作为官方开源库，其易用性和扩展性都经过开源社区和Golang官方的认证，是我们目前构建Web Server首选的HTTP协议库。</p><p>用net/http来创建一个 HTTP 服务，其实很简单，下面是<a href="https://pkg.go.dev/net/http@go1.15.5">官方文档</a>里的例子。我做了些注释，帮你理解。</p><pre><code class="language-go">// 创建一个Foo路由和处理函数
http.Handle("/foo", fooHandler)

// 创建一个bar路由和处理函数
http.HandleFunc("/bar", func(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hello, %q", html.EscapeString(r.URL.Path))
})

// 监听8080端口
log.Fatal(http.ListenAndServe(":8080", nil))
</code></pre><p>是不是代码足够简单？一共就5行，但往前继续推进之前，我想先问你几个问题，<strong>这五行代码做了什么，为什么就能启动一个 HTTP 服务，具体的逻辑是什么样的</strong>？</p><p>要回答这些问题，你就要深入理解 net/http 标准库。要不然，只会简单调用，却不知道原理，后面哪里出了问题，或者你想调优，就无从下手了。</p><p>所以，我们先来看看 net/http 标准库，从代码层面搞清楚整个 HTTP 服务的主流程原理，最后再基于原理讲实现。</p><h2>net/http 标准库怎么学</h2><p>想要在 net/http 标准库纷繁复杂的代码层级和调用中，弄清楚主流程不是一件容易事。要快速熟悉一个标准库，就得找准方法。</p><p><strong>这里我教给你一个快速掌握代码库的技巧：库函数 &gt; 结构定义 &gt; 结构函数</strong>。</p><p>简单来说，就是当你在阅读一个代码库的时候，不应该从上到下阅读整个代码文档，而应该先阅读整个代码库提供的对外库函数（function），再读这个库提供的结构（struct/class），最后再阅读每个结构函数（method）。<img src="https://static001.geekbang.org/resource/image/81/79/8150f242d1f0ee96f44793112c4dcf79.jpg?wh=1920x1080" alt=""></p><p>为什么要这么学呢？因为这种阅读思路和代码库作者的思路是一致的。</p><p>首先搞清楚这个库要提供什么功能（提供什么样的对外函数），然后为了提供这些功能，我要把整个库分为几个核心模块（结构），最后每个核心模块，我应该提供什么样的能力（具体的结构函数）来满足我的需求。</p><h3>库函数（功能）</h3><p>按照这个思路，我们来阅读 net/http 库，先看提供的对外库函数是为了实现哪些功能。这里顺带补充说明一下，我们课程对应的Golang源码的版本是1.15.5，你可以在<a href="https://github.com/gohade/coredemo/blob/geekbang/01/go.mod">01分支的coredemo/go.mod</a>里看到。</p><p>你直接通过  <code>go doc net/http | grep "^func"</code> 命令行能查询出 net/http 库所有的对外库函数：</p><pre><code>func CanonicalHeaderKey(s string) string
func DetectContentType(data []byte) string
func Error(w ResponseWriter, error string, code int)
func Get(url string) (resp *Response, err error)
func Handle(pattern string, handler Handler)
func HandleFunc(pattern string, handler func(ResponseWriter, *Request))
func Head(url string) (resp *Response, err error)
func ListenAndServe(addr string, handler Handler) error
func ListenAndServeTLS(addr, certFile, keyFile string, handler Handler) error
func MaxBytesReader(w ResponseWriter, r io.ReadCloser, n int64) io.ReadCloser
func NewRequest(method, url string, body io.Reader) (*Request, error)
func NewRequestWithContext(ctx context.Context, method, url string, body io.Reader) (*Request, error)
func NotFound(w ResponseWriter, r *Request)
func ParseHTTPVersion(vers string) (major, minor int, ok bool)
func ParseTime(text string) (t time.Time, err error)
func Post(url, contentType string, body io.Reader) (resp *Response, err error)
func PostForm(url string, data url.Values) (resp *Response, err error)
func ProxyFromEnvironment(req *Request) (*url.URL, error)
func ProxyURL(fixedURL *url.URL) func(*Request) (*url.URL, error)
func ReadRequest(b *bufio.Reader) (*Request, error)
func ReadResponse(r *bufio.Reader, req *Request) (*Response, error)
func Redirect(w ResponseWriter, r *Request, url string, code int)
func Serve(l net.Listener, handler Handler) error
func ServeContent(w ResponseWriter, req *Request, name string, modtime time.Time, ...)
func ServeFile(w ResponseWriter, r *Request, name string)
func ServeTLS(l net.Listener, handler Handler, certFile, keyFile string) error
func SetCookie(w ResponseWriter, cookie *Cookie)
func StatusText(code int) string
</code></pre><p>在这个库提供的方法中，我们去掉一些 New 和 Set 开头的函数，因为你从命名上可以看出，这些函数是对某个对象或者属性的设置。</p><p>剩下的函数大致可以分成三类：</p><ul>
<li>为服务端提供创建 HTTP 服务的函数，名字中一般包含 Serve 字样，比如 Serve、ServeFile、ListenAndServe等。</li>
<li>为客户端提供调用 HTTP 服务的类库，以 HTTP 的 method 同名，比如 Get、Post、Head等。</li>
<li>提供中转代理的一些函数，比如ProxyURL、ProxyFromEnvironment 等。</li>
</ul><p>我们现在研究的是，如何创建一个 HTTP 服务，所以关注包含 Serve 字样的函数就可以了。</p><pre><code class="language-go">// 通过监听的URL地址和控制器函数来创建HTTP服务
func ListenAndServe(addr string, handler Handler) error{}
// 通过监听的URL地址和控制器函数来创建HTTPS服务
func ListenAndServeTLS(addr, certFile, keyFile string, handler Handler) error{}
// 通过net.Listener结构和控制器函数来创建HTTP服务
func Serve(l net.Listener, handler Handler) error{}
// 通过net.Listener结构和控制器函数来创建HTTPS服务
func ServeTLS(l net.Listener, handler Handler, certFile, keyFile string) error{}
</code></pre><h3>结构定义（模块）</h3><p>然后，我们过一遍这个库提供的所有struct，看看核心模块有哪些，同样使用 go doc:</p><pre><code class="language-go">&nbsp;go doc net/http | grep "^type"|grep struct
</code></pre><p>你可以看到整个库最核心的几个结构：</p><pre><code class="language-go">type Client struct{ ... }
type Cookie struct{ ... }
type ProtocolError struct{ ... }
type PushOptions struct{ ... }
type Request struct{ ... } 
type Response struct{ ... }
type ServeMux struct{ ... }
type Server struct{ ... }
type Transport struct{ ... }
</code></pre><p>看结构的名字或者go doc查看结构说明文档，能逐渐了解它们的功能：</p><ul>
<li>Client 负责构建HTTP客户端；</li>
<li>Server 负责构建HTTP服务端；</li>
<li>ServerMux 负责HTTP服务端路由；</li>
<li>Transport、Request、Response、Cookie负责客户端和服务端传输对应的不同模块。</li>
</ul><p>现在通过库方法（function）和结构体（struct），我们对整个库的结构和功能有大致印象了。整个库承担了两部分功能，一部分是构建HTTP客户端，一部分是构建HTTP服务端。</p><p>构建的HTTP服务端除了提供真实服务之外，也能提供代理中转服务，它们分别由 Client 和 Server 两个数据结构负责。除了这两个最重要的数据结构之外，HTTP 协议的每个部分，比如请求、返回、传输设置等都有具体的数据结构负责。</p><h3>结构函数（能力）</h3><p>下面从具体的需求出发，我们来阅读具体的结构函数（method）。</p><p>我们当前的需求是创建 HTTP 服务，开头我举了一个最简单的例子：</p><pre><code class="language-go">// 创建一个Foo路由和处理函数
http.Handle("/foo", fooHandler)

// 创建一个bar路由和处理函数
http.HandleFunc("/bar", func(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hello, %q", html.EscapeString(r.URL.Path))
})

// 监听8080端口
log.Fatal(http.ListenAndServe(":8080", nil))
</code></pre><p>我们跟着 http.ListenAndServe 这个函数来理一下 net/http 创建服务的主流程逻辑。</p><p>阅读具体的代码逻辑用 <code>go doc</code> 命令明显就不够了，你需要两个东西：</p><p>一个是可以灵活进行代码跳转的 IDE，VS Code 和 GoLand 都是非常好的工具。以我们现在要查看的 http.ListenAndServe 这个函数为例，我们可以从上面的例子代码中，直接通过IDE跳转到这个函数的源码中阅读，有一个能灵活跳转的IDE工具是非常必要的。</p><p>另一个是可以方便记录代码流程的笔记，这里我的个人方法是使用思维导图。</p><p>具体方法是<strong>将要分析的代码从入口处一层层记录下来，每个函数，我们只记录其核心代码，然后对每个核心代码一层层解析</strong>。记得把思维导图的结构设置为右侧分布，这样更直观。</p><p>比如下面这张图，就是我解析部分HTTP库服务端画的<a href="https://github.com/gohade/geekbang/tree/main/01">代码分析图</a>。</p><p><img src="https://static001.geekbang.org/resource/image/3a/cd/3ab5c45e113ddf4cc3bdb0e09c85c7cd.png?wh=2464x1192" alt=""></p><p>这张图看上去层级复杂，不过不用担心，对照着思维导图，我带你一层一层阅读，讲解每一层的逻辑，带你看清楚代码背后的设计思路。</p><p>我们先顺着 http.ListenAndServe 的脉络读。</p><p><strong>第一层</strong>，http.ListenAndServe 本质是通过创建一个 Server 数据结构，调用 <code>server.ListenAndServe</code> 对外提供服务，这一层完全是比较简单的封装，目的是，将Server结构创建服务的方法 ListenAndServe ，直接作为库函数对外提供，增加库的易用性。</p><p><img src="https://static001.geekbang.org/resource/image/4e/01/4eeaace11e29989b3bfc2344ca8e4001.png?wh=810x582" alt=""></p><p>进入到<strong>第二层</strong>，创建服务的方法 ListenAndServe 先定义了监听信息 <code>net.Listen</code>，然后调用 Serve 函数。</p><p>而在<strong>第三层</strong> Serve 函数中，用了一个 for 循环，通过 <code>l.Accept</code>不断接收从客户端传进来的请求连接。当接收到了一个新的请求连接的时候，通过 <code>srv.NewConn</code>创建了一个连接结构（<code>http.conn</code>），并创建一个 Goroutine 为这个请求连接对应服务（<code>c.serve</code>）。</p><p>从第四层开始，后面就是单个连接的服务逻辑了。</p><p><img src="https://static001.geekbang.org/resource/image/79/72/798e88645b4d77a3da302a6c6d719472.jpeg?wh=1026x860" alt=""></p><p>在<strong>第四层</strong>，<code>c.serve</code>函数先判断本次  HTTP 请求是否需要升级为 HTTPs，接着创建读文本的reader和写文本的buffer，再进一步读取本次请求数据，然后<strong>第五层</strong>调用最关键的方法 <code>serverHandler{c.server}.ServeHTTP(w,&nbsp;w.req)</code> ，来处理这次请求。</p><p>这个关键方法是为了实现自定义的路由和业务逻辑，调用写法是比较有意思的：</p><pre><code class="language-go">serverHandler{c.server}.ServeHTTP(w,&nbsp;w.req)
</code></pre><p>serverHandler结构体，是标准库封装的，代表“请求对应的处理逻辑”，它只包含了一个指向总入口服务server的指针。</p><p>这个结构将总入口的服务结构Server和每个连接的处理逻辑巧妙联系在一起了，你可以看接着的<strong>第六层</strong>逻辑：</p><pre><code class="language-go">// serverHandler 结构代表请求对应的处理逻辑
type serverHandler struct {
	srv *Server
}

// 具体处理逻辑的处理函数
func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request) {
	handler := sh.srv.Handler
	if handler == nil {
		handler = DefaultServeMux
	}
	...
	handler.ServeHTTP(rw, req)
}
</code></pre><p>如果入口服务server结构已经设置了 Handler，就调用这个Handler来处理此次请求，反之则使用库自带的 DefaultServerMux。</p><p>这里的serverHandler设计，能同时保证这个库的扩展性和易用性：你可以很方便使用默认方法处理请求，但是一旦有需求，也能自己扩展出方法处理请求。</p><p>那么DefaultServeMux 是怎么寻找 Handler 的呢，这就是思维导图的最后一部分<strong>第七层</strong>。</p><p><img src="https://static001.geekbang.org/resource/image/34/fe/344fc7d6f2d1aca635ef1284185621fe.png?wh=2268x418" alt=""></p><p><code>DefaultServeMux.Handle</code> 是一个非常简单的 map 实现，key 是路径（pattern），value 是这个 pattern 对应的处理函数（handler）。它是通过 <code>mux.match</code>(path) 寻找对应 Handler，也就是从 DefaultServeMux 内部的 map 中直接根据 key 寻找到 value 的。</p><p>这种根据 map 直接查找路由的方式是不是可以满足我们的路由需求呢？我们会在第三讲路由中详细解说。</p><p>好，HTTP库 Server 的代码流程我们就梳理完成了，整个逻辑线大致是：</p><pre><code class="language-plain">创建服务 -&gt; 监听请求 -&gt; 创建连接 -&gt; 处理请求
</code></pre><p>如果你觉得层次比较多，对照着思维导图多看几遍就顺畅了。这里我也给你整理了一下逻辑线各层的关键结论：</p><ul>
<li>第一层，标准库创建HTTP服务是通过创建一个 Server 数据结构完成的；</li>
<li>第二层，Server 数据结构在for循环中不断监听每一个连接；</li>
<li>第三层，每个连接默认开启一个 Goroutine 为其服务；</li>
<li>第四、五层，serverHandler 结构代表请求对应的处理逻辑，并且通过这个结构进行具体业务逻辑处理；</li>
<li>第六层，Server 数据结构如果没有设置处理函数 Handler，默认使用 DefaultServerMux处理请求；</li>
<li>第七层，DefaultServerMux 是使用 map 结构来存储和查找路由规则。</li>
</ul><p>如果你对上面几点关键结论还有疑惑的，可以再去看一遍思维导图。阅读核心逻辑代码是会有点枯燥，但是<strong>这条逻辑线是HTTP服务启动最核心的主流程逻辑</strong>，后面我们会基于这个流程继续开发，你要掌握到能背下来的程度。千万不要觉得要背诵了，压力太大，其实对照着思维导图，顺几遍逻辑，理解了再记忆就很容易。</p><h2>创建框架的Server结构</h2><p>现在原理弄清楚了，该下手搭 HTTP 服务了。</p><p>刚刚咱也分析了主流程代码，其中第一层的关键结论就是：net/http 标准库创建服务，实质上就是通过创建 Server 数据结构来完成的。所以接下来，我们就来创建一个 Server 数据结构。</p><p>通过 <code>go doc net/http.Server</code> 我们可以看到 Server 的结构：</p><pre><code class="language-go">type Server struct {
    // 请求监听地址
	Addr string
    // 请求核心处理函数
	Handler Handler 
	...
}
</code></pre><p>其中最核心的是 Handler这个字段，从主流程中我们知道（第六层关键结论），当 Handler 这个字段设置为空的时候，它会默认使用 DefaultServerMux 这个路由器来填充这个值，但是我们一般都会使用自己定义的路由来替换这个默认路由。</p><p>所以在框架代码中，我们要创建一个自己的核心路由结构，实现 Handler。</p><p>先来理一下目录结构，我们在<a href="https://github.com/gohade/coredemo/tree/geekbang/01">GitHub</a>上创建一个项目coredemo，这个项目是这门课程所有的代码集合，包含要实现的框架和使用框架的示例业务代码。</p><p><strong>所有的框架代码都存放在framework文件夹中，而所有的示例业务代码都存放在framework文件夹之外</strong>。这里为了后面称呼方便，我们就把framework文件夹叫框架文件夹，而把外层称为业务文件夹。</p><p>当然 GitHub 上的这个coredemo是我在写课程的时候为了演示创建的，推荐你跟着一步一步写。成品在<a href="https://github.com/gohade/hade">hade项目</a>里，你可以先看看，在最后发布的时候，我们会将整个项目进行发布。在一个新的业务中，如果要使用到我们自己写好的框架，可以直接通过引用 “import 项目地址/framework” 来引入，在最后一部分做实战项目的时候我们会具体演示。</p><p>好，下面我们来一步步实现这个项目。</p><p>创建一个framework文件夹，新建core.go，在里面写入。</p><pre><code class="language-go">package framework

import "net/http"

// 框架核心结构
type Core struct {
}

// 初始化框架核心结构
func NewCore() *Core {
	return &amp;Core{}
}

// 框架核心结构实现Handler接口
func (c *Core) ServeHTTP(response http.ResponseWriter, request *http.Request) {
	// TODO
}

</code></pre><p>而在业务文件夹中创建main.go，其中的main函数就变成这样：</p><pre><code class="language-go">func main() {
	server := &amp;http.Server{
        // 自定义的请求核心处理函数
		Handler: framework.NewCore(),
        // 请求监听地址
		Addr:&nbsp; &nbsp; ":8080",
	}
	server.ListenAndServe()
}

</code></pre><p>整理下这段代码，我们通过自己创建了 Server 数据结构，并且在数据结构中创建了自定义的Handler（Core数据结构）和监听地址，实现了一个 HTTP 服务。这个服务的具体业务逻辑都集中在我们自定义的Core结构中，后续我们要做的事情就是不断丰富这个Core数据结构的功能逻辑。</p><p>后续每节课学完之后，我都会把代码放在对应的GitHub的分支中。你跟着课程敲完代码过程中有不了解的地方，可以对比参考分支。</p><p>本节课我们完成的代码分支是：<a href="https://github.com/gohade/coredemo/tree/geekbang/01">geekbang/01</a> ，代码结构我也截了图：<img src="https://static001.geekbang.org/resource/image/2c/d2/2c481d0c93efede365a7e079c4eb49d2.png?wh=734x396" alt=""></p><h2>小结</h2><p>今天我以 net/http 标准库为例，分享了快速熟悉代码库的技巧，<strong>库函数 &gt; 结构定义 &gt; 结构函数。</strong>在阅读代码库时，从功能出发，先读对外库函数，再细读这个库提供的结构，搞清楚功能和对应结构之后，最后基于实际需求看每个结构函数。</p><p>读每个结构函数的时候，我们使用思维导图梳理了net/http 创建 HTTP 服务的主流程逻辑，基于主流程原理，创建了一个属于我们框架的 Server 结构，你可以再回顾一下这张图。</p><p><img src="https://static001.geekbang.org/resource/image/3a/cd/3ab5c45e113ddf4cc3bdb0e09c85c7cd.png?wh=2464x1192" alt=""></p><p>主流程的链条比较长，但是你先理顺逻辑，记住几个关键的节点，再结合思维导图，就能记住整个主流程逻辑了，之后所有关于 HTTP 的细节和问题，我们都会基于这个主流程逻辑来思考和回答。</p><h2>思考题</h2><p>今天我用思维导图梳理了最核心的 HTTP 服务启动的主流程逻辑，知易行难，你不妨用这个思路做做下面这道思考题，尝试绘制出属于你自己的思维导图。</p><p>HTTP 库提供 FileServer 来封装对文件读取的 HTTP 服务。实现代码也非常简单：</p><pre><code class="language-go">fs := http.FileServer(http.Dir("/home/bob/static"))
http.Handle("/static/", http.StripPrefix("/static", fs))
</code></pre><p>请问它的主流程逻辑是什么？你认为其中最关键的节点是什么？</p><p>欢迎在留言区分享你的思考。感谢你的收听，如果你觉得今天的内容对你有帮助，也欢迎你把今天的内容分享给你身边的朋友，邀请他一起学习。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/2d/48/bc2648c4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>程旭</span>
  </div>
  <div class="_2_QraFYR_0">1. http.FileServer 创建 FileHandler 数据结构<br>2. FileHandler 结构体中包含 FileSystem 接口，FileSystem 接口包含Open 方法<br>3. http.Dir 的 Open 方法 实现  FileSystem 接口 的 Open 方法<br>4. http.Dir 的 Open 方法对表示字符串的文件路径进行判断：<br>    （1）先判断 分隔符是否为 &quot;&#47;&quot;且该字符串中是否包含分隔符，若不满足 返回 nil 和 error信息 &quot;http: invalid character in file path&quot;<br>    （2）将 http.Dir 从 Dir 类型转换为 string 类型，判断该是否为空，若为空，将 dir 赋值为 &quot;.&quot;<br>    （3）使用 path.Clean ，filepath.FromSlash 和 filepath.Join 方法获得路径全名<br>    （4）使用 os.Open 方法打开文件，如果打开失败，返回错误信息，如果成功以读模式打开文件</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你好，你的逻辑是正确的，不过可能过多关注分支细节。在使用思维导图的时候，如果对于比较复杂的逻辑，我们需要分析哪些是关键节点，哪些是非关键节点。<br><br>比如FileServer, 其关键点有两个：<br><br>1  fileHandler 我们能和ListenAndServe 连接起来，它提供了ServeHTTP的方法， 这个是请求处理的入口函数<br><br>2 FileServer 最本质的函数是封装了io.CopyN，基本逻辑是：<br><br>如果是读取文件夹，则遍历文件夹内所有文件，将文件名直接输出返回值。<br>如果是读取文件，则设置文件的阅读指针（如果需要多次读取文件，创建goroutine，且为每个goroutine创建阅读指针），使用io.CopyN读取文件内容输出返回值。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-18 19:39:44</div>
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
  <div class="_2_QraFYR_0">很赞， 读源码的 思路和 思维导图 很值得学习</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 恩，我平时阅读源码就是这样阅读的。经验之谈。感谢支持。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-14 09:56:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJyibojtJCnzAE7E8sMqgiaiaAHl3FuzcXcicQnjnT5huUFMxGUMzV5NGuqzzHHr8dBzCs3xfuhwcOnPw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>好家庭</span>
  </div>
  <div class="_2_QraFYR_0">老师，请问 go c.serve(connCtx) 里面为什么还有一个循环？c值得是一个connection，我理解不是每个连接处理一次就好了吗，为啥还有一个for循环呢？<br>···<br>&#47; Serve a new connection.<br>func (c *conn) serve(ctx context.Context) {<br>	c.remoteAddr = c.rwc.RemoteAddr().String()<br>	ctx = context.WithValue(ctx, LocalAddrContextKey, c.rwc.LocalAddr())<br>	defer func() {<br>		if err := recover(); err != nil &amp;&amp; err != ErrAbortHandler {<br>			const size = 64 &lt;&lt; 10<br>			buf := make([]byte, size)<br>			buf = buf[:runtime.Stack(buf, false)]<br>			c.server.logf(&quot;http: panic serving %v: %v\n%s&quot;, c.remoteAddr, err, buf)<br>		}<br>		if !c.hijacked() {<br>			c.close()<br>			c.setState(c.rwc, StateClosed)<br>		}<br>	}()<br><br>	...<br><br>	for {<br>		w, err := c.readRequest(ctx)<br>		if c.r.remain != c.server.initialReadLimitSize() {<br>			&#47;&#47; If we read any bytes off the wire, we&#39;re active.<br>			c.setState(c.rwc, StateActive)<br>		}<br>		if err != nil {<br>			const errorHeaders = &quot;\r\nContent-Type: text&#47;plain; charset=utf-8\r\nConnection: close\r\n\r\n&quot;<br><br>...<br>		}<br>...<br>```</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: http服务在启动的时候，会默认开启keep-alive机制。keep-alive机制就是一个连接在一个请求结束之后，并不关闭当前连接，在下个请求的时候也能使用这个连接。这个for循环就是为keep-alive机制服务的。在服务一个连接的时候，处理完一个请求，并不关闭这个连接，而是循环等待下个请求。<br><br>那要关闭keep-alive怎么办呢？你也可以在for循环中的w.conn.server.doKeepAlives 看到，它判断如果服务端的 disableKeepAlives 不是0，则设置了关闭keep-alive，则就不进行for循环了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-16 08:30:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/5b/8f/4b0ab5db.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Middleware</span>
  </div>
  <div class="_2_QraFYR_0">目录有点不清晰，从零开始，那么是不是应该给出建立合适的文件目录结构，命名。我们也能跟着上手敲一遍。比如这个 framework .目录是如何命名。希望老师真的能手把手</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你好，在后面有一章会专门介绍目录的章节，叫《如何系统设计框架的整体目录》。每一个章节，我都有存放源码在github项目的对应分支，如果希望跟着上手敲一遍，可以跟着文章敲完代码，跟着对应分支对一遍。https:&#47;&#47;github.com&#47;gohade&#47;coredemo&#47;tree&#47;geekbang&#47;01<br><br>谢谢支持，另外为你的ID middleware 点个赞。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-14 09:42:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/47/29/425a2030.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Groot</span>
  </div>
  <div class="_2_QraFYR_0">一篇文章值回票价，感觉后续的文章都是在做慈善 😂<br><br>受益匪浅，感谢分享 👍</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你好，非常感谢支持。非常高兴本篇能让你有一些收获。后续文章是介绍写框架的过程的。感谢。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-18 02:08:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/f6/24/547439f1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ghostwritten</span>
  </div>
  <div class="_2_QraFYR_0">打卡第二天：<br>https:&#47;&#47;github.com&#47;gohade&#47;coredemo&#47;blob&#47;geekbang&#47;01&#47;go.mod<br>https:&#47;&#47;datatracker.ietf.org&#47;doc&#47;html&#47;rfc2616<br>https:&#47;&#47;github.com&#47;valyala&#47;fasthttp<br>https:&#47;&#47;pkg.go.dev&#47;net&#47;http@go1.15.5<br><br>Web Server 第一个go架构：net&#47;http<br>熟悉库技巧：库函数 &gt; 结构定义 &gt; 结构函数<br>查看库命令：<br>go doc net&#47;http | grep &quot;^func<br>go doc net&#47;http | grep &quot;^type&quot;|grep struct<br><br>结构函数如下：<br>&#47;&#47; 创建一个Foo路由和处理函数<br>http.Handle(&quot;&#47;foo&quot;, fooHandler)<br><br>&#47;&#47; 创建一个bar路由和处理函数<br>http.HandleFunc(&quot;&#47;bar&quot;, func(w http.ResponseWriter, r *http.Request) {<br>  fmt.Fprintf(w, &quot;Hello, %q&quot;, html.EscapeString(r.URL.Path))<br>})<br><br>&#47;&#47; 监听8080端口<br>log.Fatal(http.ListenAndServe(&quot;:8080&quot;, nil))<br><br>画源代码分析图，学会脑图构思是关键（略）<br>流程：<br> - 第一层，标准库创建 HTTP 服务是通过创建一个 Server 数据结构完成的；<br> - 第二层，Server 数据结构在 for 循环中不断监听每一个连接；<br> - 第三层，每个连接默认开启一个 Goroutine 为其服务；<br> - 第四、五层，serverHandler 结构代表请求对应的处理逻辑，并且通过这个结构进行具体业务逻辑处理；<br> - 第六层，Server 数据结构如果没有设置处理函数 Handler，默认使用 `DefaultServerMux` 处理请求；<br> - 第七层，`DefaultServerMux` 是使用 map 结构来存储和查找路由规则。<br><br><br>创建框架的 Server 结构<br>1.创建一个 coredemo&#47;framework&#47;core.go,实现具体业务逻辑<br>&#47;&#47; 框架核心结构<br>type Core struct {<br>}<br><br>&#47;&#47; 初始化框架核心结构<br>func NewCore() *Core {<br>  return &amp;Core{}<br>}<br><br>&#47;&#47; 框架核心结构实现Handler接口<br>func (c *Core) ServeHTTP(response http.ResponseWriter, request *http.Request) {<br>  &#47;&#47; TODO<br>}<br><br>2.创建一个 coredemo&#47;main.go,创建服务的方法 `ListenAndServe` 先定义了监听信息 `net.Listen`，然后调用 `Serve` 函数,实现对外提供服务<br><br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍坚持一起仗剑走天涯</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-22 14:59:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/cc/de/59a530dc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>布丁老厮</span>
  </div>
  <div class="_2_QraFYR_0">老师，HTTP 库 Server 的代码流程是不是应该为：创建服务 -&gt; 监听请求 -&gt; 创建连接 -&gt; 处理请求 要更准确一点？因为net.Listen是在srv.newConn之前进行的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，确实是顺序错误，已经联系编辑进行修改了。感谢提醒。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-17 17:52:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/PiajxSqBRaEJfTnE46bP9zFU0MJicYZmKYTPhm97YjgSEmNVKr3ic1BY3CL8ibPUFCBVTqyoHQPpBcbe9GRKEN1CyA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>逗逼章鱼</span>
  </div>
  <div class="_2_QraFYR_0">FileServer 的主要流程前面五层应该都一样，第六层开始不一样，FileServer --&gt; fileHandler --&gt; fileHandler里实现的 ServeHTTP --&gt; serveFile 。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，主要在于第六层的ServeHTTP，不同的handler实现的ServeHTTP是不一样的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-15 16:34:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/27/f1/e4fc57a3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>无隅</span>
  </div>
  <div class="_2_QraFYR_0">算法有时要写服务端程序，搜go看到这门课，意外的惊喜～</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-10 10:09:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/e6/54/86056001.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小马🐎</span>
  </div>
  <div class="_2_QraFYR_0">代码中NewCore返回是一个自定义的core结构体 如何和封装好的server里面的handler成为了同一个类型？不报错的嘛？还请指教下！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-09 16:21:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/32/45/b2/701f5ad7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Mark.Q</span>
  </div>
  <div class="_2_QraFYR_0">前两天再看这一块儿，画了好几个”蜘蛛图“，人都看蒙了。感谢感谢！！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-14 19:53:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/8f/1a/7a7e0225.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>子非鱼</span>
  </div>
  <div class="_2_QraFYR_0">老师：有一个实例化的问题<br>既然已经为 Core 增加了 Constructor：func NewCore() *Core 函数，那为什么要把 Core 做成 Public 呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-03 14:53:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/8f/1a/7a7e0225.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>子非鱼</span>
  </div>
  <div class="_2_QraFYR_0">老师，请问：结构函数 是不是就是 方法？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-03 02:16:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2b/0a/23/c26f4e50.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Sunrise</span>
  </div>
  <div class="_2_QraFYR_0">授人以鱼不如授人以渔，首先感谢老师教授的学习技巧，有几点不太明白，望抽空解答：<br>go doc net&#47;http | grep &quot;^func&quot; 这个命令不是能查询出 net&#47;http 库所有的对外库函数吗，那 http.FileServer 不也是对外的库函数吗，为啥没有列出来</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-29 09:54:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/72/85/c337e9a1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>老兵</span>
  </div>
  <div class="_2_QraFYR_0">1. 使用go doc net&#47;http.FileServer | grep &#39;^func&#39; 得到结果 func FileServer(root FileSystem) Handler<br>2. 使用fs作为handler传递给http.Handle 作为参数，这样可以更加route的map，得到fs作为静态文件的handler方法。<br>3. FileServer方法中创建一个fileHandler的结构，文件路径作为root被传入fileHandler的结构中。与此同时handler都需要实现ServeHTTP的方法。<br>4. fileHanlder.ServeHTTP 方法中在使用&quot;upath := r.URL.Path&quot;得到upath后，将这个路径作为文件路径调用serveFile方法，这个方法根据请求文件进行Open操作。<br>5. 最后将文件读取的内容作为参数调用serveContent方法，方法中对io操作进行处理，最后io.CopyN将content赋值到w（ResponseWriter）中，也就是response</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-23 20:25:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLJHTX1IwEl1Eh1CCO2ejL2gKe08Vxib61UZz9l5WGA81ObK0Nk5MCZ3ic6IWcW5kyX0DtwBNMEMl2Q/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_8ed998</span>
  </div>
  <div class="_2_QraFYR_0">老师为啥你代码里mani.go里面import 是&quot;coredemo&#47;framework&quot; ，而我这里按你一样的目录结构要写成&quot;.&#47;framework&quot;才能找到包</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个是gomod的机制，如果你的项目在gomod中命名为coredemo， 那么coredemo&#47;framework就会去你的项目下的framework目录查询，当然你这里也能写成.&#47;framework。但是别人用到你的库的时候就会出现错误</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-09 11:18:18</div>
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
  <div class="_2_QraFYR_0">找到了 dirList 打印文件目录 </div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-29 17:07:44</div>
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
  <div class="_2_QraFYR_0">1. dir 是一个自定义类型 本质是字符串 实现了 FileSystem 接口<br>2. 通过http.StripPrefix(&quot;&#47;static&quot;, fs) 返回一个处理请求，里面的逻辑大概看了下 应该是把前缀static 删除之后重新构建一下path<br>3. 最终还是通过 ServeHttp处理请求，代码中还处理了 “&#47;” 如果前缀没有&#47; 还贴心的帮忙补充一个<br>4. 调用 serveFile （还包含重定向代码到 index.HTML 这个不是重点），调用Open函数打开文件<br>最后关闭文件<br><br>如果是读取文件夹，则遍历文件夹内所有文件，将文件名直接输出返回值。 关于这段话我其实具体细节代码没看太明白  = =难受</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-29 17:06:51</div>
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
  <div class="_2_QraFYR_0">打卡第一题，跟着老师学习完然后有机会可以和老师交流。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-06 14:34:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/30/3c/0668d6ae.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>盘胧</span>
  </div>
  <div class="_2_QraFYR_0">老师  为什么这个实现的实例方法 一定要叫  func (c *Core) ServeHTTP    我不小心写错了， 系统会报错提示呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: servehttp是net&#47;http提供服务的关键字</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-22 20:53:45</div>
  </div>
</div>
</div>
</li>
</ul>