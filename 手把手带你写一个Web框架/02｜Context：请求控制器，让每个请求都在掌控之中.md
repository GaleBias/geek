<audio title="02｜Context：请求控制器，让每个请求都在掌控之中" src="https://static001.geekbang.org/resource/audio/75/4f/755292d97f341be62c0258d71efe4d4f.mp3" controls="controls"></audio> 
<p>你好，我是轩脉刃。</p><p>上一讲我们使用 net/http 搭建了一个最简单的 HTTP 服务，为了帮你理解服务启动逻辑，我用思维导图梳理了主流程，如果你还有不熟悉，可以再回顾一下。</p><p>今天我将带你进一步丰富我们的框架，添加上下文 Context 为请求设置超时时间。</p><p>从主流程中我们知道（第三层关键结论），HTTP 服务会为每个请求创建一个 Goroutine 进行服务处理。在服务处理的过程中，有可能就在本地执行业务逻辑，也有可能再去下游服务获取数据。如下图，本地处理逻辑 A，下游服务 a/b/c/d， 会形成一个标准的树形逻辑链条。</p><p><img src="https://static001.geekbang.org/resource/image/33/71/33361f90298865f98e3038f11f02e671.jpg?wh=1920x1080" alt=""></p><p>在这个逻辑链条中，每个本地处理逻辑，或者下游服务请求节点，都有可能存在超时问题。<strong>而对于 HTTP 服务而言，超时往往是造成服务不可用、甚至系统瘫痪的罪魁祸首</strong>。</p><p>系统瘫痪也就是我们俗称的雪崩，某个服务的不可用引发了其他服务的不可用。比如上图中，如果服务 d 超时，导致请求处理缓慢甚至不可用，加剧了 Goroutine 堆积，同时也造成了服务 a/b/c 的请求堆积，Goroutine 堆积，瞬时请求数加大，导致 a/b/c 的服务都不可用，整个系统瘫痪，怎么办？</p><p>最有效的方法就是从源头上控制一个请求的“最大处理时长”，所以，对于一个 Web 框架而言，“超时控制”能力是必备的。今天我们就用 Context 为框架增加这个能力。</p><!-- [[[read_end]]] --><h2>context 标准库设计思路</h2><p>如何控制超时，官方是有提供 context 标准库作为解决方案的，但是由于标准库的功能并不够完善，一会我们会基于标准库，来根据需求自定义框架的 Context。所以理解其背后的设计思路就可以了。</p><p>为了防止雪崩，context 标准库的解决思路是：<strong>在整个树形逻辑链条中，用上下文控制器 Context，实现每个节点的信息传递和共享</strong>。</p><p>具体操作是：用 Context 定时器为整个链条设置超时时间，时间一到，结束事件被触发，链条中正在处理的服务逻辑会监听到，从而结束整个逻辑链条，让后续操作不再进行。</p><p>明白操作思路之后，我们深入 context 标准库看看要对应具备哪些功能。</p><p>按照上一讲介绍的了解标准库的方法，我们先通过  <code>go doc context | grep "^func"</code> 看提供了哪些库函数（function）：</p><pre><code class="language-go">// 创建退出 Context
func WithCancel(parent Context) (ctx Context, cancel CancelFunc){}
// 创建有超时时间的 Context
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc){}
// 创建有截止时间的 Context
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc){}
</code></pre><p>其中，WithCancel 直接创建可以操作退出的子节点，WithTimeout 为子节点设置了超时时间（还有多少时间结束），WithDeadline 为子节点设置了结束时间线（在什么时间结束）。</p><p>但是这只是表层功能的不同，其实这三个库函数的<strong>本质是一致的</strong>。怎么理解呢？</p><p>我们先通过  <code>go doc context | grep "^type"</code> ，搞清楚 Context 的结构定义和函数句柄，再来解答这个问题。</p><pre><code class="language-go">type Context interface {
    // 当 Context 被取消或者到了 deadline，返回一个被关闭的 channel
    Done() &lt;-chan struct{}
    ...
}

//函数句柄
type CancelFunc func() 
</code></pre><p>这个库虽然不大，但是设计感强，比较抽象，并不是很好理解。所以这里，我把 Context 的其他字段省略了。现在，我们只理解核心的 Done() 方法和 CancelFunc 这两个函数就可以了。</p><p>在树形逻辑链条上，<strong>一个节点其实有两个角色：一是下游树的管理者；二是上游树的被管理者</strong>，那么就对应需要有两个能力：</p><ul>
<li>一个是能让整个下游树结束的能力，也就是函数句柄 CancelFunc；</li>
<li>另外一个是在上游树结束的时候被通知的能力，也就是 Done()方法。同时因为通知是需要不断监听的，所以 Done() 方法需要通过 channel 作为返回值让使用方进行监听。</li>
</ul><p>看<a href="https://pkg.go.dev/context@go1.15.5">官方代码</a>示例：</p><pre><code class="language-go">package main

import (
	"context"
	"fmt"
	"time"
)

const shortDuration = 1 * time.Millisecond

func main() {
    // 创建截止时间
	d := time.Now().Add(shortDuration)
    // 创建有截止时间的 Context
	ctx, cancel := context.WithDeadline(context.Background(), d)
	defer cancel()

    // 使用 select 监听 1s 和有截止时间的 Context 哪个先结束
	select {
	case &lt;-time.After(1 * time.Second):
		fmt.Println("overslept")
	case &lt;-ctx.Done():
		fmt.Println(ctx.Err())
	}

}
</code></pre><p>主线程创建了一个 1 毫秒结束的定时器 Context，在定时器结束的时候，主线程会通过 Done()函数收到事件结束通知，然后主动调用函数句柄 cancelFunc 来通知所有子 Context 结束（这个例子比较简单没有子 Context）。</p><p>我打个更形象的比喻，CancelFunc 和 Done 方法就像是电话的话筒和听筒，话筒 CancelFunc，用来告诉管辖范围内的所有 Context 要进行自我终结，而通过监听听筒 Done 方法，我们就能听到上游父级管理者的终结命令。</p><p>总之，<strong>CancelFunc 是主动让下游结束，而 Done 是被上游通知结束</strong>。</p><p>搞懂了具体实现方法，我们回过头来看这三个库函数 WithCancel / WithDeadline / WithTimeout 就很好理解了。</p><p>它们的本质就是“通过定时器来自动触发终结通知”，WithTimeout 设置若干秒后通知触发终结，WithDeadline 设置未来某个时间点触发终结。</p><p>对应到 Context 代码中，它们的功能就是：<strong>为一个父节点生成一个带有 Done 方法的子节点，并且返回子节点的 CancelFunc 函数句柄</strong>。</p><p><img src="https://static001.geekbang.org/resource/image/90/c5/900361486571bb703261d2cfd56e87c5.jpg?wh=1920x1080" alt=""><br>
我们用一张图来辅助解释一下，Context的使用会形成一个树形结构，下游指的是树形结构中的子节点及所有子节点的子树，而上游指的是当前节点的父节点。比如图中圈起来的部分，当WithTimeout调用CancelFunc的时候，所有下游的With系列产生的Context都会从Done中收到消息。</p><h2>Context 是怎么产生的</h2><p>现在我们已经了解标准库 context 的设计思路了，在开始写代码之前，我们还要把 Context 放到 net/http 的主流程逻辑中，其中有两个问题要搞清楚：<strong>Context 在哪里产生？它的上下游逻辑是什么？</strong></p><p>要回答这两个问题，可以用我们在上一讲介绍的思维导图方法，因为主流程已经拎清楚了，现在你只需要把其中 Context 有关的代码再详细过一遍，然后在思维导图上标记出来就可以了。</p><p>这里，我已经把 Context 的关键代码都用蓝色背景做了标记，你可以检查一下自己有没有标漏。</p><p><img src="https://static001.geekbang.org/resource/image/79/a4/79a3c7eafc3ccfbe1b162e646902c5a4.png?wh=1413x906" alt=""></p><p>照旧看图梳理代码流程，来看蓝色部分，从前到后的层级梳理就不再重复讲了，我们看关键位置。</p><p>从图中最后一层的代码 req.ctx = ctx 中看到，每个连接的 Context 最终是放在 request 结构体中的。</p><p>而且这个时候， Context 已经有多层父节点。因为，在代码中，每执行一次 WithCancel、WithValue，就封装了一层 Context，我们通过这一张流程图能清晰看到最终 Context 的生成层次。</p><p><img src="https://static001.geekbang.org/resource/image/dd/22/ddbca0e4d1c66ed417b9de97c338ae22.jpg?wh=1920x1080" alt=""></p><p>你发现了吗，<strong>其实每个连接的 Context 都是基于 baseContext 复制来的</strong>。对应到代码中就是，在为某个连接开启 Goroutine 的时候，为当前连接创建了一个 connContext，这个 connContext 是基于 server 中的 Context 而来，而 server 中 Context 的基础就是 baseContext。</p><p>所以，Context 从哪里产生这个问题，我们就解决了，但是如果我们想要对 Context 进行必要的修改，还要从上下游逻辑中，找到它的修改点在哪里。</p><p>生成最终的 Context 的流程中，net/http 设计了<strong>两处可以注入修改</strong>的地方，都在 Server 结构里面，一处是 BaseContext，另一处是 ConnContext。</p><ul>
<li>BaseContext 是整个 Context 生成的源头，如果我们不希望使用默认的 context.Backgroud()，可以替换这个源头。</li>
<li>而在每个连接生成自己要使用的 Context 时，会调用 ConnContext ，它的第二个参数是 net.Conn，能让我们对某些特定连接进行设置，比如要针对性设置某个调用 IP。</li>
</ul><p>这两个函数的定义我写在下面的代码里了，展示一下，你可以看看。</p><pre><code class="language-go">type Server struct {
	...

    // BaseContext 用来为整个链条创建初始化 Context
    // 如果没有设置的话，默认使用 context.Background()
	BaseContext func(net.Listener) context.Context{}

    // ConnContext 用来为每个连接封装 Context
    // 参数中的 context.Context 是从 BaseContext 继承来的
	ConnContext func(ctx context.Context, c net.Conn) context.Context{}
    ...
}
</code></pre><p>最后，我们回看一下 req.ctx 是否能感知连接异常。<img src="https://static001.geekbang.org/resource/image/a1/fc/a1d0ece41997ac873a5292301ee988fc.jpg?wh=1920x1080" alt=""><br>
是可以的，因为链条中一个父节点为 CancelContext，其 cancelFunc 存储在代表连接的 conn 结构中，连接异常的时候，会触发这个函数句柄。</p><p>好，讲完 context 库的核心设计思想，以及在 net/http 的主流程逻辑中嵌入 context 库的关键实现，我们现在心中有图了，就可以撸起袖子开始写框架代码了。</p><p>你是不是有点疑惑，为啥要自己先理解一遍 context 标准库的生成流程，咱们直接动手干不是更快？有句老话说得好，磨刀不误砍柴功。</p><p>我们确实是要自定义，不是想直接使用标准库的 Context，因为它完全是标准库 Context 接口的实现，只能控制链条结束，封装性并不够。但是只有先搞清楚了 context 标准库的设计思路，才能精准确定自己能怎么改、改到什么程度合适，下手的时候才不容易懵。</p><p>下面我们就基于刚才讲的设计思路，从封装自己的 Context 开始，写今天的核心逻辑，也就是为单个请求设置超时，最后考虑一些边界场景，并且进行优化。</p><p>我们还是再拉一个分支 <a href="https://github.com/gohade/coredemo/tree/geekbang/02">geekbang/02</a>，接着上一节课的代码结构，在框架文件夹中封装一个自己的Context。</p><h2>封装一个自己的 Context</h2><p>在框架里，我们需要有更强大的 Context，除了可以控制超时之外，常用的功能比如获取请求、返回结果、实现标准库的 Context 接口，也都要有。</p><p><strong>我们首先来设计提供获取请求、返回结果功能</strong>。</p><p>先看一段未封装自定义 Context 的控制器代码：</p><pre><code class="language-go">// 控制器
func Foo1(request *http.Request, response http.ResponseWriter) {
	obj := map[string]interface{}{
		"data":&nbsp; &nbsp;nil,
	}
    // 设置控制器 response 的 header 部分
	response.Header().Set("Content-Type", "application/json")

    // 从请求体中获取参数
	foo := request.PostFormValue("foo")
	if foo == "" {
		foo = "10"
	}
	fooInt, err := strconv.Atoi(foo)
	if err != nil {
		response.WriteHeader(500)
		return
	}
    // 构建返回结构
	obj["data"] = fooInt&nbsp;
	byt, err := json.Marshal(obj)
	if err != nil {
		response.WriteHeader(500)
		return
	}
    // 构建返回状态，输出返回结构
	response.WriteHeader(200)
	response.Write(byt)
	return
}
</code></pre><p>这段代码重点是操作调用了 http.Request 和 http.ResponseWriter ，实现 WebService 接收和处理协议文本的功能。但这两个结构提供的接口粒度太细了，需要使用者非常熟悉这两个结构的内部字段，比如response里设置Header和设置Body的函数，用起来肯定体验不好。</p><p>如果我们能<strong>将这些内部实现封装起来，对外暴露语义化高的接口函数</strong>，那么我们这个框架的易用性肯定会明显提升。什么是好的封装呢？再看这段有封装的代码：</p><pre><code class="language-go">// 控制器
func Foo2(ctx *framework.Context) error {
	obj := map[string]interface{}{
		"data":&nbsp; &nbsp;nil,
	}
    // 从请求体中获取参数
 	fooInt := ctx.FormInt("foo", 10)
    // 构建返回结构  
	obj["data"] = fooInt
    // 输出返回结构
	return ctx.Json(http.StatusOK, obj)
}
</code></pre><p>你可以明显感受到封装性高的 Foo2 函数，更优雅更易读了。首先它的代码量更少，而且语义性也更好，近似对业务的描述：从请求体中获取 foo 参数，并且封装为 Map，最后 JSON 输出。</p><p>思路清晰了，所以这里可以将 request 和 response 封装到我们自定义的 Context 中，对外提供请求和结果的方法，我们把这个Context结构写在框架文件夹的context.go文件中：</p><pre><code class="language-go">// 自定义 Context
type Context struct {
	request&nbsp; &nbsp; &nbsp; &nbsp; *http.Request
	responseWriter http.ResponseWriter
	...
}
</code></pre><p>对request和response封装的具体实现，我们到第五节课封装的时候再仔细说。</p><p><strong>然后是第二个功能，标准库的 Context 接口</strong>。</p><p>标准库的 Context 通用性非常高，基本现在所有第三方库函数，都会根据官方的建议，将第一个参数设置为标准 Context 接口。所以我们封装的结构只有实现了标准库的 Context，才能方便直接地调用。</p><p>到底有多方便，我们看使用示例：</p><pre><code class="language-go">func Foo3(ctx *framework.Context) error {
	rdb := redis.NewClient(&amp;redis.Options{
		Addr:&nbsp; &nbsp; &nbsp;"localhost:6379",
		Password: "", // no password set
		DB:&nbsp; &nbsp; &nbsp; &nbsp;0,&nbsp; // use default DB
	})

	return rdb.Set(ctx, "key", "value", 0).Err()
}

</code></pre><p>这里使用了 go-redis 库，它每个方法的参数中都有一个标准 Context 接口，这让我们能将自定义的 Context 直接传递给 rdb.Set。</p><p>所以在我们的框架上实现这一步，只需要调用刚才封装的 request 中的 Context 的标准接口就行了，很简单，我们继续在context.go中进行补充：</p><pre><code class="language-go">func (ctx *Context) BaseContext() context.Context {
	return ctx.request.Context()
}

func (ctx *Context) Done() &lt;-chan struct{} {
	return ctx.BaseContext().Done()
}
</code></pre><p>这里举例了两个method的实现，其他的都大同小异就不在文稿里展示，你可以先自己写，然后对照我放在<a href="https://github.com/gohade/coredemo/blob/geekbang/02/framework/context.go">GitHub</a>上的完整代码检查一下。</p><p>自己封装的 Context 最终需要提供四类功能函数：</p><ul>
<li>base 封装基本的函数功能，比如获取 http.Request 结构</li>
<li>context 实现标准 Context 接口</li>
<li>request 封装了 http.Request 的对外接口</li>
<li>response 封装了 http.ResponseWriter 对外接口</li>
</ul><p>完成之后，使用我们的IDE里面的结构查看器（每个IDE显示都不同），就能查看到如下的函数列表：<img src="https://static001.geekbang.org/resource/image/3c/cc/3c0c98741275beb7bdf5d6333b0c91cc.jpg?wh=1248x1456" alt=""></p><p>有了我们自己封装的 Context 之后，控制器就非常简化了。把框架定义的 ControllerHandler 放在框架目录下的controller.go文件中：</p><pre><code class="language-go">type ControllerHandler func(c *Context) error
</code></pre><p>把处理业务的控制器放在业务目录下的controller.go文件中：</p><pre><code class="language-go">func FooControllerHandler(ctx *framework.Context) error {
	return ctx.Json(200, map[string]interface{}{
		"code": 0,
	})
}
</code></pre><p>参数只有一个 framework.Context，是不是清爽很多，这都归功于刚完成的自定义 Context。</p><h2>为单个请求设置超时</h2><p>上面我们封装了自定义的 Context，从设计层面实现了标准库的Context。下面回到我们这节课核心要解决的问题，为单个请求设置超时。</p><p>如何使用自定义 Context 设置超时呢？结合前面分析的标准库思路，我们三步走完成：</p><ol>
<li>继承 request 的 Context，创建出一个设置超时时间的 Context；</li>
<li>创建一个新的 Goroutine 来处理具体的业务逻辑；</li>
<li>设计事件处理顺序，当前 Goroutine 监听超时时间 Contex 的 Done()事件，和具体的业务处理结束事件，哪个先到就先处理哪个。</li>
</ol><p>理清步骤，我们就可以在业务的controller.go文件中完成业务逻辑了。<strong>第一步生成一个超时的 Context</strong>：</p><pre><code class="language-go">durationCtx, cancel := context.WithTimeout(c.BaseContext(), time.Duration(1*time.Second))
// 这里记得当所有事情处理结束后调用 cancel，告知 durationCtx 的后续 Context 结束
defer cancel()
</code></pre><p>这里为了最终在浏览器做验证，我设置超时事件为1s，这样最终验证的时候，最长等待1s 就可以知道超时是否生效。</p><p><strong>第二步创建一个新的 Goroutine 来处理业务逻辑</strong>：</p><pre><code class="language-go">finish := make(chan struct{}, 1)

go func() {
		...
		// 这里做具体的业务
		time.Sleep(10 * time.Second)
        c.Json(200, "ok")
        ...
        // 新的 goroutine 结束的时候通过一个 finish 通道告知父 goroutine
		finish &lt;- struct{}{}
}()
</code></pre><p>为了最终的验证效果，我们使用time.Sleep将新 Goroutine 的业务逻辑事件人为往后延迟了10s，再输出“ok”，这样最终验证的时候，效果比较明显，因为前面的超时设置会在1s生效了，浏览器就有表现了。</p><p><strong>到这里我们这里先不急着进入第三步，还有错误处理情况没有考虑到位</strong>。这个新创建的Goroutine如果出现未知异常怎么办？需要我们额外捕获吗？</p><p>其实在 Golang 的设计中，每个 Goroutine 都是独立存在的，父 Goroutine 一旦使用 Go 关键字开启了一个子 Goroutine，父子 Goroutine 就是平等存在的，他们互相不能干扰。而在异常面前，所有 Goroutine 的异常都需要自己管理，不会存在父 Goroutine 捕获子 Goroutine 异常的操作。</p><p>所以切记：在 Golang 中，每个 Goroutine 创建的时候，我们要使用 defer 和 recover 关键字为当前 Goroutine 捕获 panic 异常，并进行处理，否则，任意一处 panic 就会导致整个进程崩溃！</p><p>这里你可以标个重点，面试会经常被问到。</p><p>搞清楚这一点，<strong>我们回看第二步，做完具体业务逻辑就结束是不行的，还需要处理 panic</strong>。所以这个 Goroutine 应该要有两个 channel 对外传递事件：</p><pre><code class="language-go">// 这个 channal 负责通知结束
finish := make(chan struct{}, 1)
// 这个 channel 负责通知 panic 异常
panicChan := make(chan interface{}, 1)

go func() {
        // 这里增加异常处理
		defer func() {
			if p := recover(); p != nil {
				panicChan &lt;- p
			}
		}()
		// 这里做具体的业务
		time.Sleep(10 * time.Second)
        c.Json(200, "ok")
        ...
        // 新的 goroutine 结束的时候通过一个 finish 通道告知父 goroutine
		finish &lt;- struct{}{}
}(）
</code></pre><p>现在第二步才算完成了，我们继续写第三步监听。使用 select 关键字来监听三个事件：异常事件、结束事件、超时事件。</p><pre><code class="language-go">  select {
    // 监听 panic
	case p := &lt;-panicChan:
		...
        c.Json(500, "panic")
    // 监听结束事件
	case &lt;-finish:
		...
        fmt.Println("finish")
    // 监听超时事件
	case &lt;-durationCtx.Done():
		...
        c.Json(500, "time out")
	}
</code></pre><p>接收到结束事件，只需要打印日志，但是，在接收到异常事件和超时事件的时候，我们希望告知浏览器前端“异常或者超时了”，所以会使用 c.Json 来返回一个字符串信息。</p><p>三步走到这里就完成了对某个请求的超时设置，你可以通过 go build、go run 尝试启动下这个服务。如果你在浏览器开启一个请求之后，浏览器不会等候事件处理 10s，而在等待我们设置的超时事件 1s 后，页面显示“time out”就结束这个请求了，就说明我们为某个事件设置的超时生效了。</p><h2>边界场景</h2><p>到这里，我们的超时逻辑设置就结束且生效了。但是，这样的代码逻辑只能算是及格，为什么这么说呢？因为它并没有覆盖所有的场景。</p><p>我们的代码逻辑要再严谨一些，<strong>把边界场景也考虑进来</strong>。这里有两种可能：</p><ol>
<li>异常事件、超时事件触发时，需要往 responseWriter 中写入信息，这个时候如果有其他 Goroutine 也要操作 responseWriter，会不会导致 responseWriter 中的信息出现乱序？</li>
<li>超时事件触发结束之后，已经往 responseWriter 中写入信息了，这个时候如果有其他 Goroutine 也要操作 responseWriter， 会不会导致 responseWriter 中的信息重复写入？</li>
</ol><p>你先分析第一个问题，是很有可能出现的。方案不难想到，我们要保证在事件处理结束之前，不允许任何其他 Goroutine 操作 responseWriter，这里可以使用一个锁（sync.Mutex）对 responseWriter 进行写保护。</p><p>在框架文件夹的context.go中对Context结构进行一些设置：</p><pre><code class="language-go">type Context struct {
	// 写保护机制
	writerMux&nbsp; *sync.Mutex
}

// 对外暴露锁
func (ctx *Context) WriterMux() *sync.Mutex {
	return ctx.writerMux
}
</code></pre><p>在刚才写的业务文件夹controller.go 中也进行对应的修改：</p><pre><code>func FooControllerHandler(c *framework.Context) error {
	...
    // 请求监听的时候增加锁机制
	select {
	case p := &lt;-panicChan:
		c.WriterMux().Lock()
		defer c.WriterMux().Unlock()
		...
		c.Json(500, &quot;panic&quot;)
	case &lt;-finish:
        ...
		fmt.Println(&quot;finish&quot;)
	case &lt;-durationCtx.Done():
		c.WriterMux().Lock()
		defer c.WriterMux().Unlock()
		c.Json(500, &quot;time out&quot;)
		c.SetTimeout()
	}
	return nil
}
</code></pre><p>那第二个问题怎么处理，我提供一个方案。我们可以<strong>设计一个标记</strong>，当发生超时的时候，设置标记位为 true，在 Context 提供的 response 输出函数中，先读取标记位；当标记位为 true，表示已经有输出了，不需要再进行任何的 response 设置了。</p><p>同样在框架文件夹中修改context.go：</p><pre><code class="language-go">type Context struct {
    ...
	// 是否超时标记位
	hasTimeout bool
	...
}

func (ctx *Context) SetHasTimeout() {
	ctx.hasTimeout = true
}

func (ctx *Context) Json(status int, obj interface{}) error {
	if ctx.HasTimeout() {
		return nil
	}
	...
}
</code></pre><p>在业务文件夹中修改controller.go：</p><pre><code>func FooControllerHandler(c *framework.Context) error {
	...
	select {
	case p := &lt;-panicChan:
		...
	case &lt;-finish:
		fmt.Println(&quot;finish&quot;)
	case &lt;-durationCtx.Done():
		c.WriterMux().Lock()
		defer c.WriterMux().Unlock()
		c.Json(500, &quot;time out&quot;)
        // 这里记得设置标记为
		c.SetHasTimeout()
	}
	return nil
}
</code></pre><p>好了，到了这里，我们就完成了请求超时设置，并且考虑了边界场景。</p><p>剩下的验证部分，我们写一个简单的路由函数，将这个控制器路由在业务文件夹中创建一个route.go:</p><pre><code class="language-go">func registerRouter(core *framework.Core) {
  // 设置控制器
   core.Get("foo", FooControllerHandler)
}
</code></pre><p>并修改main.go：</p><pre><code class="language-go">func main() {
   ...
   // 设置路由
   registerRouter(core)
   ...
}
</code></pre><p>就可以运行了。完整代码照旧放在GitHub的 <a href="https://github.com/gohade/coredemo/tree/geekbang/02">geekbang/02 分支</a>上了。<br>
<img src="https://static001.geekbang.org/resource/image/72/e4/7277357c506bc95b9155d6a35cdee3e4.png?wh=714x656" alt=""></p><h2>小结</h2><p>今天，我们定义了一个属于自己框架的 Context，它有两个功能：在各个 Goroutine 间传递数据；控制各个 Goroutine，也就是是超时控制。</p><p>这个自定义 Context 结构封装了 net/http 标准库主逻辑流程产生的 Context，与主逻辑流程完美对接。它除了实现了标准库的 Context 接口，还封装了 request 和 response 的请求。你实现好了 Context 之后，就会发现它跟百宝箱一样，在处理具体的业务逻辑的时候，如果需要获取参数、设置返回值等，都可以通过 Context 获取。</p><p>封装后，我们通过三步走为请求设置超时，并且完美地考虑了各种边界场景。</p><p>你是不是觉得我们这一路要思考的点太多了，又是异常，又是边界场景。但是这里我要特别说明，其实真正要衡量框架的优劣，要看什么？就是看细节。</p><p><strong>所有框架的基本原理和基本思路都差不多，但是在细节方面，各个框架思考的程度是不一样的，才导致使用感天差地别</strong>。所以如果你想完成一个真正生产能用得上的框架，这些边界场景、异常分支，都要充分考虑清楚。</p><h2>思考题</h2><p>在context库的官方文档中有这么一句话：</p><blockquote>
<p>Do not store Contexts inside a struct type;<br>
instead, pass a Context explicitly to each function that needs it.<br>
The Context should be the first parameter.</p>
</blockquote><p>大意是说建议我们设计函数的时候，将Context作为函数的第一个参数。你能理解官方为什么如此建议，有哪些好处？可以结合你的工作经验，说说自己的看法。</p><p>欢迎在留言区分享你的思考，畅所欲言。如果你觉得今天的内容有所帮助，也欢迎你分享给你身边的朋友，邀他一起学习。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/70/67/0c1359c2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>qinsi</span>
  </div>
  <div class="_2_QraFYR_0">“context作为函数的第一个参数”大概有两层意思。一是作为函数的参数传入。这个应该是针对在一个struct的多个方法中共享一个context的情况说的。因为每个方法都有可能需要创建子context，所以不应该共享而是应该显式传递。二是作为第一个参数。这个多半是一种约定。在支持可变长参数的语言中，固定参数只能出现在可变参数的前面。而作为与业务逻辑关系不大的context，出现在第一个的位置也方便也其他参数作区分。<br><br>说到与业务逻辑关系不大，个人以为显式传递context是对业务逻辑的侵入，更别提单元测试的时候还需要适当地mock掉。目前代码里请求控制和请求实现混在一起的情况后面应该会改掉吧？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: context作为第一个参数在实际工作中是非常有用的一个实践。不管我们是设计一个函数，或者设计一个结构体的方法，或者服务的时候，我们一旦养成了将第一个参数作为context的习惯，那么这个context在相互调用的时候，就会传递下去。这里会带来几个好处：<br><br>1 链路通用内容传递。context中是可以通过WithValue方法将某些字段封装在context里面，并且传递的。最常见的字段是traceId, spanId。而在日志中带上这些ID，再将日志收集起来，我们就能进行分析了。这也是我们现在比较流行的全链路分析的原理。<br><br>2 链路统一设置超时。我们在定义一个服务的时候，将第一个参数固定设置为context，则可以通过这个context进行超时设置，而这个超时设置，是由上游调用方进行设置，这样就形成了一个统一的超时设置机制。比如A设置了5s超时，自己使用了1s，传递到下游B服务的时候，设置B的context超时时长为4s。这样全链路超时传递下去，就能保持统一设置了。<br><br>是的，请求控制和请求实现混在一起的情况，后面引入middleware会改掉的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-15 15:24:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/c3/3f/d96c1b97.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>zhao</span>
  </div>
  <div class="_2_QraFYR_0">代码里面context的封装跟gin的源代码真是像极了，对照来看，对框架的理解又加深了一些。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你好，是的context的封装和gin的代码很像，本质就是通过自己开发对开源的gin会有进一步理解。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-17 11:30:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/27/af/fd/a1708649.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ゝ骑着小车去兜风。</span>
  </div>
  <div class="_2_QraFYR_0">老师这个课程真的全是干货，感觉看了前两章都已经值回票价了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-23 15:22:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_cbab11</span>
  </div>
  <div class="_2_QraFYR_0">异常事件、超时事件触发时，需要往 responseWriter 中写入信息，这个时候如果有其他 Goroutine 也要操作 responseWriter，会不会导致 responseWriter 中的信息出现乱序？<br>超时事件触发结束之后，已经往 responseWriter 中写入信息了，这个时候如果有其他 Goroutine 也要操作 responseWriter， 会不会导致 responseWriter 中的信息重复写入？<br><br>问题1：这两个问题说的其他goroutine是哪些goroutine？<br>问题2：为什么controller中的goroutine没有进行Lock操作？<br>问题3：在Unlock之前已经进行了调用了Write方法，难道此次请求还没有结束吗？如果结束了为什么还允许继续写入呢？<br><br>还望老师解答一下。感谢！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-08 17:44:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/27/ab/29/e4f7fb3c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>高。</span>
  </div>
  <div class="_2_QraFYR_0">老师，我有个疑问，代码中<br>go fun() {<br>&#47;&#47; 这里做具体的业务 <br>time.Sleep(10 * time.Second) <br>c.Json(200, &quot;ok&quot;) ... <br>&#47;&#47; 新的 goroutine 结束的时候通过一个 finish 通道告知父 goroutine <br>finish &lt;- struct{}<br>} <br><br>如果说设置的超时时间也是10秒，那么子goroutine中c.Json(200, &quot;ok&quot;)执行到一半，正好轮到父goroutine执行写入c.Json(500, &quot;time out&quot;)，此时是父goroutine会等待子goroutine写入完成，还是子goroutine由于写锁的原因被打断写入<br><br>如果是第一种的话，response就会被写入两次，就会产生以下报错<br>http: superfluous response.WriteHeader call from coredemo&#47;framework.(*Context).Json <br><br>是否应该优化一下加锁的逻辑？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-04 23:37:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2a/39/93/bcafe00f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>自由人</span>
  </div>
  <div class="_2_QraFYR_0">真的很好，我读了四遍，才读懂老师代码的强大。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-29 08:24:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/65/df/11406608.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>轻剑切重剑</span>
  </div>
  <div class="_2_QraFYR_0">老师好，请教下，为什么在超时之后 case &lt;-durationCtx.Done()，不添加 c.Abort() ？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-08 17:36:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/21/41/37/b89f3d67.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我在睡觉</span>
  </div>
  <div class="_2_QraFYR_0">你好老师，这个代码运行之后，一次HTTP请求过来，ServeHTTP函数会被调用两次，请问是为什么？<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 有例子么？一个请求应该只会运行一次的，有更多信息没</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-26 13:16:03</div>
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
  <div class="_2_QraFYR_0">return rdb.Set(ctx, &quot;key&quot;, &quot;value&quot;, 0).Err()  老师 这个方法报错了欸：<br>源码： func (c *cmdable) Set(key string, value interface{}, expiration time.Duration) *StatusCmd {...}<br>好像多了一个参数啊</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-25 21:34:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/57/2a/a00a3ce7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Nio</span>
  </div>
  <div class="_2_QraFYR_0">不错不错 有深度</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-26 11:36:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>2345</span>
  </div>
  <div class="_2_QraFYR_0">文章写得很好，赞一个，比较有深度</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢支持，欢迎继续参与。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-22 21:03:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>0mfg</span>
  </div>
  <div class="_2_QraFYR_0">叶老师您好，把分支2 git下来尝试运行，报错如下，求指教，谢谢<br># command-line-arguments<br>.\main.go:10:2: undefined: registerRouter<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: https:&#47;&#47;github.com&#47;gohade&#47;coredemo&#47;blob&#47;geekbang&#47;02&#47;route.go 确认下你的目录下有route.go文件么</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-22 17:29:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Qn1PDx7xA7jKFZr4vHibmsvoZ7bwUCzHTg3uywiaESCgFTTMibPpKdZOfrqTXtdQXxUJqFqmLAj5NoIFMJpYibbcOQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>happy learn</span>
  </div>
  <div class="_2_QraFYR_0">到底是一个请求一个goroutine还是一个连接一个goroutine，前后两篇文章说的不一致</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 01梳理思维导图的时候说，接受一个新的请求连接时，会创建一个新结构，开启一个goroutine来服务。所以这两种说法其实是一致的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-21 12:22:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/f1/39/b0960780.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>恶魔果实</span>
  </div>
  <div class="_2_QraFYR_0">为什么会导致服务b和服务c的瞬时请求加大？这里不是很理解。<br>a请求失败，但是b，c请求是成功的呀。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个例子举的确实不好。我的本意是如果a请求失败，整个上游可能有重试机制，导致其他的请求连接增加。<br><br>我这里可能换一个方式举例更好，下游超时导致上游连接数增加。已经联系编辑进行更新了。<br><br>谢谢提醒。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-18 12:55:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/28/08/01/da970033.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>zh</span>
  </div>
  <div class="_2_QraFYR_0">不是视频嘛？请问</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-20 17:05:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2e/8c/f9/e1dab0ca.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>怎么睡才能做这种梦</span>
  </div>
  <div class="_2_QraFYR_0">看完这一章，感觉目前还没达到这个水平，难以跟进呀</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 加油，原理确实很难讲到不生涩，但是需要细看</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-14 15:57:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/61/77/e4d198a6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>MR.偉い！</span>
  </div>
  <div class="_2_QraFYR_0">c.Json(200, &quot;ok&quot;)也是对操作 responseWriter进行的操作，为什么不在c.Json的时候加锁呢？<br>在c.Json的时候，case &lt;-durationCtx.Done():也是有可能发生的呀？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-12 00:19:23</div>
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
  <div class="_2_QraFYR_0">最后两种边界情况不太懂，如果有其他 Goroutine 也要操作 responseWriter 不是新的请求吗，跟这次的没啥关联吧，能举个具体场景吗</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-29 15:54:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/d7/f1/ce10759d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>wei 丶</span>
  </div>
  <div class="_2_QraFYR_0">老师，我有个疑问<br>finish := make(chan struct{}, 1)<br>panicChan := make(chan interface{},1)<br>这两个为什么还需要给1的容量，其实这俩个不用给就可以吧</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-19 22:48:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTK6xwUp8JiaFNPNSlNxubQlTgcxl02Yc1eiaOzvj75Wob9AYVdsYwAapowkkicenhV0Y02dW2yibPicHDg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_96685a</span>
  </div>
  <div class="_2_QraFYR_0">你好，作者，这里我有两个疑问，文章中所说的边界场景，在什么情况下可以能出现呢？可以具体举个例子么，我认为一个请求对应一个context，为啥会出现并发问题呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-08 10:27:58</div>
  </div>
</div>
</div>
</li>
</ul>