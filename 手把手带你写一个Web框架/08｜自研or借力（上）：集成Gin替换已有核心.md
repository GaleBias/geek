<audio title="08｜自研or借力（上）：集成Gin替换已有核心" src="https://static001.geekbang.org/resource/audio/5a/e8/5a9635da064e2d41149b90yy2d67f4e8.mp3" controls="controls"></audio> 
<p>你好，我是轩脉刃。</p><p>之前我们从零开始研发了一个有控制器、路由、中间件等组件的 Web 服务器框架，这个服务器框架可以说是麻雀虽小，但五脏俱全。上节课我们还介绍了目前最主流的一些框架 Gin、Echo 和Beego。</p><p>这里不知道你有没有这些疑问，我们的框架和这些顶级的框架相比，差了什么呢？如何才能快速地把我们的框架可用性，和这些框架提升到同一个级别？我们做这个框架除了演示每个实现细节，它的优势是什么呢？</p><p>不妨带着这些问题，把我们新出炉的框架和GitHub上star数最高的<a href="https://github.com/gin-gonic/gin">Gin框架</a>比对一下，思考下之间的差距到底是什么。</p><h2>和Gin对比</h2><p>Gin 框架无疑是现在最火的框架，你能想出很多它的好处，但是在我看来，它之所以那么成功，<strong>最主要的原因在于两点：细节和生态</strong>。</p><p>其实框架之间的实现原理都差不多，但是生产级别的框架和我们写的示例级别的框架相比，差别就在于细节，这个细节是需要很多人、很多时间去不断打磨的。</p><p>如果你的 Golang 经验积累到一定时间，那肯定能很轻松实现一个示例级别的框架，但是往往是没有开源市场的，因为你的框架，在细节上的设计没有经过很多人验证，也没有经过在生产环境中的实战。这些都需要一个较为庞大的使用群体和较长时间才能慢慢打磨出来。</p><!-- [[[read_end]]] --><p>Gin 里面的每一行代码，基本都是在很长时间里，经过各个贡献者的打磨，才演变至今的。我们看 Gin 框架的代码优化和提交频率，从这一讲落笔算起（2021 年 8 月 9 日），最近的一次提交改进是 6 天前（2021 年 8 月 3 日），Gin 框架升级到了 v1.7.3 版本。</p><p>我们也可以统计一下Gin 的 master 分支，从 2014 开始至今已经有 1475 次优化提交。这里的每一次优化提交，都是对 Gin 框架细节的再一次打磨。</p><h3>Recovery 的错误捕获</h3><p>光放数字说服力不明显，我们直接比较代码，看看之前实现的各个逻辑在 Gin 框架中是如何实现的，你就可以感受到在细节上的差距了。</p><p>还记得在第四课的时候，我们实现了一个 Recovery 中间件么？放在框架middleware文件夹的recovery.go中：</p><pre><code class="language-go">// recovery 机制，将协程中的函数异常进行捕获
func Recovery() framework.ControllerHandler {
	// 使用函数回调
	return func(c *framework.Context) error {
		// 核心在增加这个 recover 机制，捕获 c.Next()出现的 panic
		defer func() {
			if err := recover(); err != nil {
				c.Json(500, err)
			}
		}()
		// 使用 next 执行具体的业务逻辑
		c.Next()

		return nil
	}
}
</code></pre><p>这个中间件的作用是捕获协程中的函数异常。我们使用 defer、recover 函数，捕获了 c.Next 中抛出的异常，并且在 HTTP 请求中返回 500 内部错误的状态码。</p><p>乍看这段代码，是没有什么问题的。但是我们再仔细思考下是否有需要完善的细节？</p><p>首先是异常类型，我们原先认为，所有异常都可以通过状态码，直接将异常状态返回给调用方，但是这里是有问题的。<strong>这里的异常，除了业务逻辑的异常，是不是也有可能是底层连接的异常</strong>？</p><p>以底层连接中断异常为例，对于这种连接中断，我们是没有办法通过设置 HTTP 状态码来让浏览器感知的，并且一旦中断，后续的所有业务逻辑都没有什么作用了。同时，如果我们持续给已经中断的连接发送请求，会在底层持续显示网络连接错误（broken pipe）。</p><p>所以在遇到这种底层连接异常的时候，应该直接中断后续逻辑。来看看 Gin 对于连接中断的捕获是怎么处理的。（你可以查看Gin的Recovery <a href="https://github.com/gin-gonic/gin/blob/master/recovery.go">GitHub地址</a>）</p><pre><code class="language-go">return func(c *Context) {
		defer func() {
			if err := recover(); err != nil {
				// 判断是否是底层连接异常，如果是的话，则标记 brokenPipe
				var brokenPipe bool
				if ne, ok := err.(*net.OpError); ok {
					if se, ok := ne.Err.(*os.SyscallError); ok {
						if strings.Contains(strings.ToLower(se.Error()), "broken pipe") || strings.Contains(strings.ToLower(se.Error()), "connection reset by peer") {
							brokenPipe = true
						}
					}
				}
				...

                
				if brokenPipe {
					// 如果有标记位，我们不能写任何的状态码
					c.Error(err.(error)) // nolint: errcheck
					c.Abort()
				} else {
					handle(c, err)
				}
			}
		}()
		c.Next()
	}
</code></pre><p>这段代码先判断了底层抛出的异常是否是网络异常（net.OpError），如果是的话，再根据异常内容是否包含“broken pipe”或者“connection reset by peer”，来判断这个异常是否是连接中断的异常。如果是，就设置标记位，并且直接使用 c.Abort() 来中断后续的处理逻辑。</p><p>这个处理异常的逻辑可以说是非常细节了，<strong>区分了网络连接错误的异常和普通的逻辑异常，并且进行了不同的处理逻辑</strong>。这一点，可能是绝大多数的开发者都没有考虑到的。</p><h3>Recovery 的日志打印</h3><p>我们再看下 Recovery 的日志打印部分，也非常能体现出 Gin 框架对细节的考量。</p><p>当然在第四讲我们只是完成了最基础的部分，没有考虑到打印 Recovery 捕获到的异常，不妨在这里先试想下，如果是自己实现，你会如何打印这个异常呢？如果你有想法了，我们再来对比看看 Gin 是如何实现这个异常信息打印的(<a href="https://github.com/gin-gonic/gin/blob/master/recovery.go">GitHub地址</a>)。</p><p>首先，打印异常内容是一定得有的。这里直接使用 logger.Printf 就可以打印出来了。</p><pre><code class="language-go">logger.Printf("%s\n%s%s", err, ...)
</code></pre><p>其次，异常是由某个请求触发的，所以触发这个异常的请求内容，也是必要的调试信息，需要打印。</p><pre><code class="language-go">httpRequest, _ := httputil.DumpRequest(c.Request, false)
headers := strings.Split(string(httpRequest), "\r\n")
// 如果 header 头中有认证信息，隐藏，不打印。
for idx, header := range headers {
current := strings.Split(header, ":")
  if current[0] == "Authorization" {
  	  headers[idx] = current[0] + ": *"
  }
}
headersToStr := strings.Join(headers, "\r\n")
</code></pre><p>分析一下这段代码，Gin 使用了一个 DumpRequest 来输出请求中的数据，包括了 HTTP header 头和 HTTP Body。</p><p>这里值得注意的是，为了安全考虑 Gin 还注意到了，<strong>如果请求头中包含 Authorization 字段，即包含 HTTP 请求认证信息，在输出的地方会进行隐藏处理</strong>，不会由于 panic 就把请求的认证信息输出在日志中。这一个细节，可能大多数开发者也都没有考虑到。</p><p>最后看堆栈信息打印，Gin 也有其特别的实现。我们打印堆栈一般是使用 <a href="https://pkg.go.dev/runtime@go1.17.1#Caller">runtime 库的 Caller</a> 来打印：</p><pre><code class="language-go">// 打印堆栈信息，是否有这个堆栈
func Caller(skip int) (pc uintptr, file string, line int, ok bool)
</code></pre><p>caller 方法返回的是堆栈函数所在的函数指针、文件名、函数所在行号信息。但是在使用过程中你就会发现，使用 Caller 是打印不出真实代码的。</p><p>比如下面这个例子，我们使用 Caller 方法，将文件名、行号、堆栈函数打印出来：</p><pre><code class="language-go">// 在 prog.go 文件，main 库中调用 call 方法
func call(skip int) bool {   //24 行
	pc,file,line,ok := runtime.Caller(skip) //25 行
	pcName := runtime.FuncForPC(pc).Name()&nbsp; //26 行
	fmt.Println(fmt.Sprintf("%v %s %d %s",pc,file,line,pcName)) //27 行
	return ok //28 行
} //29 行
</code></pre><p>打印出的第一层堆栈函数的信息：</p><pre><code class="language-go">4821380 /tmp/sandbox064396492/prog.go 25 main.call
</code></pre><p>这个堆栈信息并不友好，它告诉我们，第一层信息所在地址为 prog.go 文件的第 25 行，在 main 库的 call 函数里面。所以如果想了解下第 25 行有什么内容，用这个堆栈信息去源码中进行文本查找，是做不到的。这个时候就非常希望信息能打印出具体的真实代码。</p><p>在 Gin 中，打印堆栈的时候就有这么一个逻辑：<strong>先去本地查找是否有这个源代码文件，如果有的话，获取堆栈所在的代码行数，将这个代码行数直接打印到控制台中</strong>。</p><pre><code class="language-go">// 打印具体的堆栈信息
func stack(skip int) []byte {
	...
    // 循环从第 skip 层堆栈到最后一层
	for i := skip; ; i++ { 
		pc, file, line, ok := runtime.Caller(i)
		// 获取堆栈函数所在源文件
		if file != lastFile {
			data, err := ioutil.ReadFile(file)
			if err != nil {
				continue
			}
			lines = bytes.Split(data, []byte{'\n'})
			lastFile = file
		}
        // 打印源代码的内容
		fmt.Fprintf(buf, "\t%s: %s\n", function(pc), source(lines, line))
	}
	return buf.Bytes()
}

</code></pre><p>这样，打印出来的堆栈信息形如：</p><pre><code class="language-go">/Users/yejianfeng/Documents/gopath/pkg/mod/github.com/gin-gonic/gin@v1.7.2/context.go:165 (0x1385b5a)
        (*Context).Next: c.handlers[c.index](c)
</code></pre><p>这个堆栈信息就友好多了，它告诉我们，这个堆栈函数发生在文件的 165 行，它的代码为 c.handlers<a href="c">c.index</a>， 这行代码所在的父级别函数为 (*Context).Next。</p><p>最终，Gin 打印出来的 panic 信息形式如下：</p><pre><code class="language-go">2021/08/15 14:18:57 [Recovery] 2021/08/15 - 14:18:57 panic recovered:
GET /first HTTP/1.1
Host: localhost:8080
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/ *;q=0.8,application/signed-exchange;v=b3;q=0.9
Accept-Encoding: gzip, deflate, br
...
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/92.0.4515.131 Safari/537.36



%!s(int=121321321)
/Users/yejianfeng/Documents/UGit/gindemo/main.go:19 (0x1394214)
&nbsp; &nbsp; &nbsp; &nbsp; main.func1: panic(121321321)
/Users/yejianfeng/Documents/gopath/pkg/mod/github.com/gin-gonic/gin@v1.7.2/context.go:165 (0x1385b5a)
&nbsp; &nbsp; &nbsp; &nbsp; (*Context).Next: c.handlers[c.index](c)
/Users/yejianfeng/Documents/gopath/pkg/mod/github.com/gin-gonic/gin@v1.7.2/recovery.go:99 (0x1392c48)
&nbsp; &nbsp; &nbsp; &nbsp; CustomRecoveryWithWriter.func1: c.Next()
/Users/yejianfeng/Documents/gopath/pkg/mod/github.com/gin-gonic/gin@v1.7.2/context.go:165 (0x1385b5a)
&nbsp; &nbsp; &nbsp; &nbsp; (*Context).Next: c.handlers[c.index](c)
...
/usr/local/Cellar/go/1.15.5/libexec/src/net/http/server.go:1925 (0x12494ac)
&nbsp; &nbsp; &nbsp; &nbsp; (*conn).serve: serverHandler{c.server}.ServeHTTP(w, w.req)
/usr/local/Cellar/go/1.15.5/libexec/src/runtime/asm_amd64.s:1374 (0x106bb00)
&nbsp; &nbsp; &nbsp; &nbsp; goexit: BYTE&nbsp; &nbsp; $0x90&nbsp; &nbsp;// NOP
</code></pre><p>可以说这个调试信息就非常丰富了，对我们在实际工作中的调试会有非常大的帮助。但是这些丰富的细节都是在开源过程中，不断补充起来的。</p><h3>路由对比</h3><p>刚才只挑Recovery 中间件的错误捕获和日志打印简单说了一下，我们可以再挑核心一些的功能，例如比较一下我们实现的路由和 Gin 的路由的区别。</p><p>在第三节课使用 trie 树实现了一个路由，它的每一个节点是一个 segment。<br>
<img src="https://static001.geekbang.org/resource/image/4b/b7/4b53584aa86dbd992263867ff1f3e7b7.jpg?wh=1920x1224" alt=""></p><p>而 Gin 框架的路由选用的是一种压缩后的基数树（radix tree），它和我们之前实现的 trie 树相比最大的区别在于，它并不是把按照分隔符切割的 segment 作为一个节点，而是<strong>把整个 URL 当作字符串，尽可能匹配相同的字符串作为公共前缀</strong>。<br>
<img src="https://static001.geekbang.org/resource/image/88/94/8869327a3ef4fb0037f364f74f3d0994.jpg?wh=1920x1080" alt=""></p><p>为什么 Gin 选了这个模型？其实 radix tree 和 trie 树相比，最大的区别就在于它节点的压缩比例最大化。直观比较上面两个图就能看得出来，对于 URL 比较长的路由规则，trie 树的节点数就比 radix tree 的节点数更多，整个数的层级也更深。</p><p>针对路由这种功能模块，创建路由树的频率远低于查找路由点频率，那么<strong>减少节点层级，无异于能提高查找路由的效率，整体路由模块的性能也能得到提高</strong>，所以 Gin 用 radix tree 是更好的选择。</p><p>另外在路由查找中，Gin 也有一些细节做得很好。首先，从父节点查找子节点，并不是像我们之前实现的那样，通过遍历所有子节点来查找是否有子节点。Gin 在每个 node 节点保留一个 indices 字符串，这个字符串的每个字符代表子节点的第一个字符：</p><p>在Gin源码的<a href="https://github.com/gin-gonic/gin/blob/master/tree.go">tree.go</a>中可以看到。</p><pre><code class="language-go">// node 节点
type node struct {
	path&nbsp; &nbsp; &nbsp; string
	indices&nbsp; &nbsp;string  // 子节点的第一个字符
	...
	children&nbsp; []*node // 子节点
    ...
}
</code></pre><p>这个设计是为了加速子节点的查询效率。使用 Hash 查找的时间复杂度为 O(1)，而我们使用遍历子节点的方式进行查找的效率为 O(n)。</p><p>在拼接 indices 字符串的时候，这里 Gin 还有一个代码方面的细节值得注意，<a href="https://github.com/gin-gonic/gin/blob/master/tree.go">在tree.go中</a>有一段这样的代码：</p><pre><code class="language-go"> 	path = path[i:]
	c := path[0]

	...
	// 插入子节点
	if c != ':' &amp;&amp; c != '*' &amp;&amp; n.nType != catchAll {
		// []byte for proper unicode char conversion, see #65
        // 将字符串拼接进入 indices
		n.indices += bytesconv.BytesToString([]byte{c})
		...
		n.addChild(child)
		...
</code></pre><p>将字符 c 拼接进入 indices 的时候，使用了一个 bytesconv.BytesToString 方法来将字符 c 转换为 string。你可以先想想，这里为什么不使用 string 关键字直接转换呢？</p><p>因为在 Golang 中，string 转化为 byte 数组，以及 byte 数组转换为 string ，都是有内存消耗的。以 byte 数组使用关键字 string 转换为字符串为例，Golang 会先在内存空间中重新开辟一段字符串空间，然后将 byte 数组复制进入这个字符串空间，这样不管是在内存使用的效率，还是在CPU资源使用的效率上，都存在一定消耗。</p><p>而在 Gin 的第 <a href="https://github.com/gin-gonic/gin/pull/2206/files">#2206</a> 号提交中，有开源贡献者就使用自研的 bytesconv 包，将 Gin 中的字符数组和 string 的转换统一做了一次修改。</p><pre><code class="language-go">package bytesconv

// 字符串转化为字符数组，不需要创建新的内存
func StringToBytes(s string) []byte {
	return *(*[]byte)(unsafe.Pointer(
		&amp;struct {
			string
			Cap int
		}{s, len(s)},
	))
}

// 字符数组转换为字符串，不需要创建新的内存
func BytesToString(b []byte) string {
	return *(*string)(unsafe.Pointer(&amp;b))
}
</code></pre><p>bytesconv 包的原理就是，直接使用 unsafe.Pointer 来强制转换字符串或者字符数组的类型。</p><p>这些细节的修改，一定程度上减少了 Gin 包内路由的内存占用空间。类似的细节点还有很多，需要每一行代码琢磨过去，而且这里的每一个细节优化点都是，开源贡献者在生产环境中长期使用 Gin 才提炼出来的。</p><p>不要小看这些看似非常细小的修改。因为细节往往是积少成多的，所有的这些细节逐渐优化、逐渐完善，才让 Gin 框架的实用度得到持久的提升，和其他框架的差距就逐渐体现出来了，这样才让Gin框架成了生产环境中首选的框架。</p><h2>生态</h2><p>聊完细节，再来看看生态。你肯定有点疑惑，为什么我会把生态这个点单独拿出来说，难道生态不是由于框架的质量好而附带的生态繁荣吗？</p><p>其实不然。一个开源项目的成功，最重要的是两个事情，<strong>第一个是质量，开源项目的代码质量是摆在第一位的，但是还有一个是不能被忽略的：生态的完善</strong>。</p><p>一个好的开源框架项目，不仅仅只有代码，还需要围绕着核心代码打造文档、框架扩展、辅助命令等。这些代码周边的组件和功能的打造，虽然从难度上看，并没有核心代码那么重要，但是它是一个长期推进和完善的过程。</p><p>Gin 的整个生态环境是非常优质的，在开源中间件、社区上都能看到其优势。</p><p>从 2014 年至今 Gin 已有多个开源共享者为其共享了开源中间件，目前<a href="https://github.com/orgs/gin-contrib/repositories">官方GitHub</a>组织收录的中间件有 23 个，非收录官方，但是在<a href="https://github.com/gin-gonic/contrib">官方README</a>记录的也有 45 个。</p><p>这些中间件包含了 Web 开发各个方面的功能，比如提供跨域请求的 cors 中间件、提供本地缓存的 cache 中间件、集成了 pprof 工具的 pprof 中间件、自动生成全链路 trace 的 opengintracing 中间件等等。<strong>如果你用一个自己的框架，就需要重建生态一一开发，这是非常烦琐的，而且工作量巨大</strong>。</p><p>Gin 的 GitHub 官网的社区活跃度比较高，对于已有的一些问题，在官网的 issue 提出来立刻就会有人回复，在 Google 或者 Baidu 上搜索 Gin，也会有很多资料。所以对于工作中必然会出现的问题，我们可以在网络上很方便找寻到这个框架的解决办法。这也是 Gin 大受欢迎的原因之一。</p><p>其实除了Gin框架，我们可以从不少其他语言的框架中看到生态的重要性。比如前端的Vue框架，它不仅仅提供了最核心的Vue的框架代码，还提供了脚手架工具Vue-CLI、Vue的Chrome插件、Vue的官方论坛和聊天室、Vue的示例文档Cookbook。这些周边生态对Vue框架的成功起到了至关重要的作用。</p><h2>站在巨人的肩膀才能做得更好</h2><p>刚才分析了一些 Gin 框架的细节以及它强大的生态，相信已经回答了开头我们提出来的问题：和这些顶级的框架相比，我们做的到底差了什么？我们的框架在可用性上，能不能迅速提升到这些顶级框架的级别？很明显短时间是不可能的。</p><p>讲了这么多，其实我想说：只有站在巨人的肩膀才能做得更好。</p><p>如果是为了学习，我们之前从零自己边造轮子边学是个好方法；<strong>但是如果你的目标是工业使用，那从零开始就非常不明智了</strong>。</p><p>因为在现在的技术氛围下，开源早已成为了共识。互联网的开源社区早就有我们需要的各个成形零件，我们要做的是，使用这些零件来继续开发。在 Golang Web 框架这个领域也是一样的，如果是想从头开始制造框架，除非你后面有很强大的团队在支撑，否则写出来的框架不仅没有市场，可能连实用性也会受到质疑。</p><p><strong>其实很多市面上的框架，也都是基于已有的轮子来再开发的</strong>。就拿 Gin 框架本身来说吧，它的路由是基于 httprouter 这个项目来定制化修改的；再比如 Macaron 框架，它是基于 Martini 框架的设计实现的。它们都是在原有的开源项目基础上，按照自己的设计思路重新改造的，也都获得了成功。</p><p>而且从我的个人经验来看，那些从头开始所有框架功能都是由自己开发的同学，往往很难坚持下来。所以你现在是不是明白了，为什么在课程最开始会说，我们先从零搭建出框架的核心部分，然后基于 Gin 来做进一步拓展和完善。</p><p>因为我们希望通过这门课花大力气搭建出来的 Golang Web 框架，不只是一个示例级别的框架，而是真正能用到具体工作环境中的，要做到这一点，它得使用开源社区的各种优秀开源库，目的就是提升你解决问题的效率。</p><h2>小结</h2><p>今天通过对比 Gin 框架和我们之前设计的框架间的细节，展示了一个成熟的生产级别的框架与一个示例级别框架在细节上的距离。</p><p><strong>现代框架的理念不在于实现，而更多在于组合</strong>。基于某些基础组件或者基础实现，不断按照自己或者公司的需求，进行二次改造和二次开发，从而打造出适合需求的形态。</p><p>比如 PHP 领域的 Laravel 框架，就是将各种底层组件、Symfony、Eloquent ORM、Monolog 等进行组装，而框架自身提供统一的组合调度方式；比如 Ruby 领域的 Rails 框架，整合了 Ruby 领域的各种优秀开源库，<strong>而框架的重点在于如何整理组件库、如何提供更便捷的机制，让程序员迅速解决问题</strong>。</p><p>所以，接下来我们设计框架的思路，就要从之前的从零开始造轮子，转换为站在巨人的肩膀上了，会借用 Gin 框架来继续实现。准备好，下一讲我们就要开始动手改造了。</p><h3>思考题</h3><p>在Gin框架中，也和我们第5讲一样提供了Query系列方法，你能看出Gin实现的Query系列方法和我们实现的有什么不同么？</p><p>欢迎在留言区分享你的思考。感谢你的收听，如果你觉得有收获，也欢迎你把今天的内容分享给你身边的朋友，邀他一起学习。我们下节课见。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/9a/0f/da7ed75a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>芒果少侠</span>
  </div>
  <div class="_2_QraFYR_0">gin的query方法，通过本地内存cache缓存了请求的query参数。后续每次读query参数时，都会从内存中直接读，减少了每次都要调用request.Query()方法的计算开销。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-08 20:33:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/5e/67/133d2da6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_5244fa</span>
  </div>
  <div class="_2_QraFYR_0">众人拾柴火焰高，这个课程这么多人学习，可以一起做一个框架。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-29 09:44:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/d0/7f/1ad28cd3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>王博</span>
  </div>
  <div class="_2_QraFYR_0">前面的读了好几遍了，真的受益匪浅，期待后续</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-29 17:21:28</div>
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
  <div class="_2_QraFYR_0">老师我准备慢下来 我要把 trie树给优化一下 之前写leetcode的时候这种压缩的情况我比较清楚 我可以尝试一下</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞，期待</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-03 16:46:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/3a/8c/fc2c3e5c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>xl666</span>
  </div>
  <div class="_2_QraFYR_0">老师请教下，错误拦截的那个底层连接异常，如果服务器异常，不应该都建立不了tcp连接吗，怎么还能在业务代码中拦截呢。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-17 09:11:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/70/07/bb4e6568.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我是熊大</span>
  </div>
  <div class="_2_QraFYR_0">看完第六讲时，看到地下的评论，为作者捏了一把汗，直到第八讲，才松了一口气，每个框架都是无数次优化得到完善的，优秀的人的贡献存在才能让社区更强大</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-28 22:45:14</div>
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
  <div class="_2_QraFYR_0">a==&gt;b==&gt;c<br>c打印出了：a==&gt;b==&gt;c<br>b打印出了：a==&gt;b</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-14 21:39:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://wx.qlogo.cn/mmopen/vi_32/1gKhS58rAibx6KHQhgJNL7k1jsiblXbwJwveqNU0zJJUKLTCDX51haJibicMd6ic3KREezXVqpWqTaGo2Gc9jfFGpdw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_d217a5</span>
  </div>
  <div class="_2_QraFYR_0">indices作用是什么？源码是遍历indices数组找到对应的字符，不能直接遍历childnode的path吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 其实也是可以便利childnode的path，但是那样的效率就没有indeces高了，毕竟indexes是一个内存数组，这个是最快的，不需要根据指针寻址啥的。所以indices顾名思义就是一个索引，加速查找的目的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-10 11:20:35</div>
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
  <div class="_2_QraFYR_0">只有站在巨人的肩膀才能做得更好。 <br>这个就是吸取前人经验，自己可以站在巨人肩膀上把事情做的更好或许会让这个巨人更高让别人站在自己肩膀上继续增高</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，什么都自己再造一遍轮子，就没有办法形成一个合力了，哈哈</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-03 16:55:13</div>
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
  <div class="_2_QraFYR_0">Eloquent ORM 是真的方便好用，laravel框架本身我就觉得是符合创业公司开发的利器。只不过我个人从刚刚学习听过很多php的言论导致 潜移默化的不喜欢php </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 语言没有好坏，只有流行与否。我也是laravel爱好者</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-03 16:45:09</div>
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
  <div class="_2_QraFYR_0">感觉老师讲解源码让我会看的很入神 </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-03 16:31:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/70/67/0c1359c2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>qinsi</span>
  </div>
  <div class="_2_QraFYR_0">打印堆栈是对本地开发友好的功能，线上不会那么搞。因为通常部署的时候不会把源码一起部署上线，而且一个个去读取源文件开销也大</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-29 16:33:32</div>
  </div>
</div>
</div>
</li>
</ul>