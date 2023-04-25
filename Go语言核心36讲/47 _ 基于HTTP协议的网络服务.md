<audio title="47 _ 基于HTTP协议的网络服务" src="https://static001.geekbang.org/resource/audio/e5/8f/e5086c42c16596609483acb1e4bfb88f.mp3" controls="controls"></audio> 
<p>我们在上一篇文章中简单地讨论了网络编程和socket，并由此提及了Go语言标准库中的<code>syscall</code>代码包和<code>net</code>代码包。</p><p>我还重点讲述了<code>net.Dial</code>函数和<code>syscall.Socket</code>函数的参数含义。前者间接地调用了后者，所以正确理解后者，会对用好前者有很大裨益。</p><p>之后，我们把视线转移到了<code>net.DialTimeout</code>函数以及它对操作超时的处理上，这又涉及了<code>net.Dialer</code>类型。实际上，这个类型正是<code>net</code>包中这两个“拨号”函数的底层实现。</p><p>我们像上一篇文章的示例代码那样用<code>net.Dial</code>或<code>net.DialTimeout</code>函数来访问基于HTTP协议的网络服务是完全没有问题的。HTTP协议是基于TCP/IP协议栈的，并且它也是一个面向普通文本的协议。</p><p>原则上，我们使用任何一个文本编辑器，都可以轻易地写出一个完整的HTTP请求报文。只要你搞清楚了请求报文的头部（header）和主体（body）应该包含的内容，这样做就会很容易。所以，在这种情况下，即便直接使用<code>net.Dial</code>函数，你应该也不会感觉到困难。</p><p>不过，不困难并不意味着很方便。如果我们只是访问基于HTTP协议的网络服务的话，那么使用<code>net/http</code>代码包中的程序实体来做，显然会更加便捷。</p><!-- [[[read_end]]] --><p>其中，最便捷的是使用<code>http.Get</code>函数。我们在调用它的时候只需要传给它一个URL就可以了，比如像下面这样：</p><pre><code>url1 := &quot;http://google.cn&quot;
fmt.Printf(&quot;Send request to %q with method GET ...\n&quot;, url1)
resp1, err := http.Get(url1)
if err != nil {
	fmt.Printf(&quot;request sending error: %v\n&quot;, err)
}
defer resp1.Body.Close()
line1 := resp1.Proto + &quot; &quot; + resp1.Status
fmt.Printf(&quot;The first line of response:\n%s\n&quot;, line1)
</code></pre><p><code>http.Get</code>函数会返回两个结果值。第一个结果值的类型是<code>*http.Response</code>，它是网络服务给我们传回来的响应内容的结构化表示。</p><p>第二个结果值是<code>error</code>类型的，它代表了在创建和发送HTTP请求，以及接收和解析HTTP响应的过程中可能发生的错误。</p><p><code>http.Get</code>函数会在内部使用缺省的HTTP客户端，并且调用它的<code>Get</code>方法以完成功能。这个缺省的HTTP客户端是由<code>net/http</code>包中的公开变量<code>DefaultClient</code>代表的，其类型是<code>*http.Client</code>。它的基本类型也是可以被拿来使用的，甚至它还是开箱即用的。下面的这两行代码：</p><pre><code>var httpClient1 http.Client
resp2, err := httpClient1.Get(url1)
</code></pre><p>与前面的这一行代码</p><pre><code>resp1, err := http.Get(url1)
</code></pre><p>是等价的。</p><p><code>http.Client</code>是一个结构体类型，并且它包含的字段都是公开的。之所以该类型的零值仍然可用，是因为它的这些字段要么存在着相应的缺省值，要么其零值直接就可以使用，且代表着特定的含义。</p><p>现在，我问你一个问题，是关于这个类型中的最重要的一个字段的。</p><p><strong>今天的问题是：<code>http.Client</code>类型中的<code>Transport</code>字段代表着什么？</strong></p><p>这道题的<strong>典型回答</strong>是这样的。</p><p><code>http.Client</code>类型中的<code>Transport</code>字段代表着：向网络服务发送HTTP请求，并从网络服务接收HTTP响应的操作过程。也就是说，该字段的方法<code>RoundTrip</code>应该实现单次HTTP事务（或者说基于HTTP协议的单次交互）需要的所有步骤。</p><p>这个字段是<code>http.RoundTripper</code>接口类型的，它有一个由<code>http.DefaultTransport</code>变量代表的缺省值（以下简称<code>DefaultTransport</code>）。当我们在初始化一个<code>http.Client</code>类型的值（以下简称<code>Client</code>值）的时候，如果没有显式地为该字段赋值，那么这个<code>Client</code>值就会直接使用<code>DefaultTransport</code>。</p><p>顺便说一下，<code>http.Client</code>类型的<code>Timeout</code>字段，代表的正是前面所说的单次HTTP事务的超时时间，它是<code>time.Duration</code>类型的。它的零值是可用的，用于表示没有设置超时时间。</p><h2>问题解析</h2><p>下面，我们再通过该字段的缺省值<code>DefaultTransport</code>，来深入地了解一下这个<code>Transport</code>字段。</p><p><code>DefaultTransport</code>的实际类型是<code>*http.Transport</code>，后者即为<code>http.RoundTripper</code>接口的默认实现。这个类型是可以被复用的，也推荐被复用，同时，它也是并发安全的。正因为如此，<code>http.Client</code>类型也拥有着同样的特质。</p><p><code>http.Transport</code>类型，会在内部使用一个<code>net.Dialer</code>类型的值（以下简称<code>Dialer</code>值），并且，它会把该值的<code>Timeout</code>字段的值，设定为<code>30</code>秒。</p><p>也就是说，这个<code>Dialer</code>值如果在30秒内还没有建立好网络连接，那么就会被判定为操作超时。在<code>DefaultTransport</code>的值被初始化的时候，这样的<code>Dialer</code>值的<code>DialContext</code>方法会被赋给前者的<code>DialContext</code>字段。</p><p><code>http.Transport</code>类型还包含了很多其他的字段，其中有一些字段是关于操作超时的。</p><ul>
<li><code>IdleConnTimeout</code>：含义是空闲的连接在多久之后就应该被关闭。</li>
<li><code>DefaultTransport</code>会把该字段的值设定为<code>90</code>秒。如果该值为<code>0</code>，那么就表示不关闭空闲的连接。注意，这样很可能会造成资源的泄露。</li>
<li><code>ResponseHeaderTimeout</code>：含义是，从客户端把请求完全递交给操作系统到从操作系统那里接收到响应报文头的最大时长。<code>DefaultTransport</code>并没有设定该字段的值。</li>
<li><code>ExpectContinueTimeout</code>：含义是，在客户端递交了请求报文头之后，等待接收第一个响应报文头的最长时间。在客户端想要使用HTTP的“POST”方法把一个很大的报文体发送给服务端的时候，它可以先通过发送一个包含了“Expect: 100-continue”的请求报文头，来询问服务端是否愿意接收这个大报文体。这个字段就是用于设定在这种情况下的超时时间的。注意，如果该字段的值不大于<code>0</code>，那么无论多大的请求报文体都将会被立即发送出去。这样可能会造成网络资源的浪费。<code>DefaultTransport</code>把该字段的值设定为了<code>1</code>秒。</li>
<li><code>TLSHandshakeTimeout</code>：TLS是Transport Layer Security的缩写，可以被翻译为传输层安全。这个字段代表了基于TLS协议的连接在被建立时的握手阶段的超时时间。若该值为<code>0</code>，则表示对这个时间不设限。<code>DefaultTransport</code>把该字段的值设定为了<code>10</code>秒。</li>
</ul><p>此外，还有一些与<code>IdleConnTimeout</code>相关的字段值得我们关注，即：<code>MaxIdleConns</code>、<code>MaxIdleConnsPerHost</code>以及<code>MaxConnsPerHost</code>。</p><p>无论当前的<code>http.Transport</code>类型的值（以下简称<code>Transport</code>值）访问了多少个网络服务，<code>MaxIdleConns</code>字段都只会对空闲连接的总数做出限定。而<code>MaxIdleConnsPerHost</code>字段限定的则是，该<code>Transport</code>值访问的每一个网络服务的最大空闲连接数。</p><p>每一个网络服务都会有自己的网络地址，可能会使用不同的网络协议，对于一些HTTP请求也可能会用到代理。<code>Transport</code>值正是通过这三个方面的具体情况，来鉴别不同的网络服务的。</p><p><code>MaxIdleConnsPerHost</code>字段的缺省值，由<code>http.DefaultMaxIdleConnsPerHost</code>变量代表，值为<code>2</code>。也就是说，在默认情况下，对于某一个<code>Transport</code>值访问的每一个网络服务，它的空闲连接数都最多只能有两个。</p><p>与<code>MaxIdleConnsPerHost</code>字段的含义相似的，是<code>MaxConnsPerHost</code>字段。不过，后者限制的是，针对某一个<code>Transport</code>值访问的每一个网络服务的最大连接数，不论这些连接是否是空闲的。并且，该字段没有相应的缺省值，它的零值表示不对此设限。</p><p><code>DefaultTransport</code>并没有显式地为<code>MaxIdleConnsPerHost</code>和<code>MaxConnsPerHost</code>这两个字段赋值，但是它却把<code>MaxIdleConns</code>字段的值设定为了<code>100</code>。</p><p>换句话说，在默认情况下，空闲连接的总数最大为<code>100</code>，而针对每个网络服务的最大空闲连接数为<code>2</code>。注意，上述两个与空闲连接数有关的字段的值应该是联动的，所以，你有时候需要根据实际情况来定制它们。</p><p>当然了，这首先需要我们在初始化<code>Client</code>值的时候，定制它的<code>Transport</code>字段的值。定制这个值的方式，可以参看<code>DefaultTransport</code>变量的声明。</p><p>最后，我简单说一下为什么会出现空闲的连接。我们都知道，HTTP协议有一个请求报文头叫做“Connection”。在HTTP协议的1.1版本中，这个报文头的值默认是“keep-alive”。</p><p>在这种情况下的网络连接都是持久连接，它们会在当前的HTTP事务完成后仍然保持着连通性，因此是可以被复用的。</p><p>既然连接可以被复用，那么就会有两种可能。一种可能是，针对于同一个网络服务，有新的HTTP请求被递交，该连接被再次使用。另一种可能是，不再有对该网络服务的HTTP请求，该连接被闲置。</p><p>显然，后一种可能就产生了空闲的连接。另外，如果分配给某一个网络服务的连接过多的话，也可能会导致空闲连接的产生，因为每一个新递交的HTTP请求，都只会征用一个空闲的连接。所以，为空闲连接设定限制，在大多数情况下都是很有必要的，也是需要斟酌的。</p><p>如果我们想彻底地杜绝空闲连接的产生，那么可以在初始化<code>Transport</code>值的时候把它的<code>DisableKeepAlives</code>字段的值设定为<code>true</code>。这时，HTTP请求的“Connection”报文头的值就会被设置为“close”。这会告诉网络服务，这个网络连接不必保持，当前的HTTP事务完成后就可以断开它了。</p><p>如此一来，每当一个HTTP请求被递交时，就都会产生一个新的网络连接。这样做会明显地加重网络服务以及客户端的负载，并会让每个HTTP事务都耗费更多的时间。所以，在一般情况下，我们都不要去设置这个<code>DisableKeepAlives</code>字段。</p><p>顺便说一句，在<code>net.Dialer</code>类型中，也有一个看起来很相似的字段<code>KeepAlive</code>。不过，它与前面所说的HTTP持久连接并不是一个概念，<code>KeepAlive</code>是直接作用在底层的socket上的。</p><p>它的背后是一种针对网络连接（更确切地说，是TCP连接）的存活探测机制。它的值用于表示每间隔多长时间发送一次探测包。当该值不大于<code>0</code>时，则表示不开启这种机制。<code>DefaultTransport</code>会把这个字段的值设定为<code>30</code>秒。</p><p>好了，以上这些内容阐述的就是，<code>http.Client</code>类型中的<code>Transport</code>字段的含义，以及它的值的定制方式。这涉及了<code>http.RoundTripper</code>接口、<code>http.DefaultTransport</code>变量、<code>http.Transport</code>类型，以及<code>net.Dialer</code>类型。</p><h2>知识扩展</h2><h3>问题：<code>http.Server</code>类型的<code>ListenAndServe</code>方法都做了哪些事情？</h3><p><code>http.Server</code>类型与<code>http.Client</code>是相对应的。<code>http.Server</code>代表的是基于HTTP协议的服务端，或者说网络服务。</p><p><code>http.Server</code>类型的<code>ListenAndServe</code>方法的功能是：监听一个基于TCP协议的网络地址，并对接收到的HTTP请求进行处理。这个方法会默认开启针对网络连接的存活探测机制，以保证连接是持久的。同时，该方法会一直执行，直到有严重的错误发生或者被外界关掉。当被外界关掉时，它会返回一个由<code>http.ErrServerClosed</code>变量代表的错误值。</p><p>对于本问题，典型回答可以像下面这样。</p><p>这个<code>ListenAndServe</code>方法主要会做下面这几件事情。</p><ol>
<li>检查当前的<code>http.Server</code>类型的值（以下简称当前值）的<code>Addr</code>字段。该字段的值代表了当前的网络服务需要使用的网络地址，即：IP地址和端口号. 如果这个字段的值为空字符串，那么就用<code>":http"</code>代替。也就是说，使用任何可以代表本机的域名和IP地址，并且端口号为<code>80</code>。</li>
<li>通过调用<code>net.Listen</code>函数在已确定的网络地址上启动基于TCP协议的监听。</li>
<li>检查<code>net.Listen</code>函数返回的错误值。如果该错误值不为<code>nil</code>，那么就直接返回该值。否则，通过调用当前值的<code>Serve</code>方法准备接受和处理将要到来的HTTP请求。</li>
</ol><p>可以从当前问题直接衍生出的问题一般有两个，一个是“<code>net.Listen</code>函数都做了哪些事情”，另一个是“<code>http.Server</code>类型的<code>Serve</code>方法是怎样接受和处理HTTP请求的”。</p><p><strong>对于第一个直接的衍生问题，如果概括地说，回答可以是：</strong></p><ol>
<li>解析参数值中包含的网络地址隐含的IP地址和端口号；</li>
<li>根据给定的网络协议，确定监听的方法，并开始进行监听。</li>
</ol><p>从这里的第二个步骤出发，我们还可以继续提出一些间接的衍生问题。这往往会涉及<code>net.socket</code>函数以及相关的socket知识。</p><p><strong>对于第二个直接的衍生问题，我们可以这样回答：</strong></p><p>在一个<code>for</code>循环中，网络监听器的<code>Accept</code>方法会被不断地调用，该方法会返回两个结果值；第一个结果值是<code>net.Conn</code>类型的，它会代表包含了新到来的HTTP请求的网络连接；第二个结果值是代表了可能发生的错误的<code>error</code>类型值。</p><p>如果这个错误值不为<code>nil</code>，除非它代表了一个暂时性的错误，否则循环都会被终止。如果是暂时性的错误，那么循环的下一次迭代将会在一段时间之后开始执行。</p><p>如果这里的<code>Accept</code>方法没有返回非<code>nil</code>的错误值，那么这里的程序将会先把它的第一个结果值包装成一个<code>*http.conn</code>类型的值（以下简称<code>conn</code>值），然后通过在新的goroutine中调用这个<code>conn</code>值的<code>serve</code>方法，来对当前的HTTP请求进行处理。</p><p>这个处理的细节还是很多的，所以我们依然可以找出不少的间接的衍生问题。比如，这个<code>conn</code>值的状态有几种，分别代表着处理的哪个阶段？又比如，处理过程中会用到哪些读取器和写入器，它们的作用分别是什么？再比如，这里的程序是怎样调用我们自定义的处理函数的，等等。</p><p>诸如此类的问题很多，我就不在这里一一列举和说明了。你只需要记住一句话：“源码之前了无秘密”。上面这些问题的答案都可以在Go语言标准库的源码中找到。如果你想对本问题进行深入的探索，那么一定要去看<code>net/http</code>代码包的源码。</p><h2>总结</h2><p>今天，我们主要讲的是基于HTTP协议的网络服务，侧重点仍然在客户端。</p><p>我们在讨论了<code>http.Get</code>函数和<code>http.Client</code>类型的简单使用方式之后，把目光聚焦在了后者的<code>Transport</code>字段。</p><p>这个字段代表着单次HTTP事务的操作过程。它是<code>http.RoundTripper</code>接口类型的。它的缺省值由<code>http.DefaultTransport</code>变量代表，其实际类型是<code>*http.Transport</code>。</p><p><code>http.Transport</code>包含的字段非常多。我们先讲了<code>DefaultTransport</code>中的<code>DialContext</code>字段会被赋予什么样的值，又详细说明了一些关于操作超时的字段。</p><p>比如<code>IdleConnTimeout</code>和<code>ExpectContinueTimeout</code>，以及相关的<code>MaxIdleConns</code>和<code>MaxIdleConnsPerHost</code>等等。之后，我又简单地解释了出现空闲连接的原因，以及相关的定制方式。</p><p>最后，作为扩展，我还为你简要地梳理了<code>http.Server</code>类型的<code>ListenAndServe</code>方法，执行的主要流程。不过，由于篇幅原因，我没有做深入讲述。但是，这并不意味着没有必要深入下去。相反，这个方法很重要，值得我们认真地去探索一番。</p><p>在你需要或者有兴趣的时候，我希望你能去好好地看一看<code>net/http</code>包中的相关源码。一切秘密都在其中。</p><h2>思考题</h2><p>我今天留给你的思考题比较简单，即：怎样优雅地停止基于HTTP协议的网络服务程序？</p><p><a href="https://github.com/hyper0x/Golang_Puzzlers">戳此查看Go语言专栏文章配套详细代码。</a></p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/74/7f/114062a3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>晨曦</span>
  </div>
  <div class="_2_QraFYR_0">“人生的道路都是由心来描绘的，所以，无论自己处于多么严酷的境遇之中，心头都不应为悲观的思想所萦绕。”<br>被老师的精神打动，真心祝愿早日康复！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-28 08:08:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/b1/0f/e81a93ed.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>嘎嘎</span>
  </div>
  <div class="_2_QraFYR_0">看测试用例中是用 srv.Shutdown(context.Background()) 的方式停止服务，通过RegisterOnShutdown可添加服务停止时的调用</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-04 18:29:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/13/00/a4a2065f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Michael</span>
  </div>
  <div class="_2_QraFYR_0">看了下源码之后感觉应该这样做：<br><br>quit := make(chan os.Signal, 1)<br>signal.Notify(quit, os.Interrupt, syscall.SIGTERM)<br><br>server := http.Server{..}<br><br>go func(){<br>     server. ListenAndServe()<br>}()<br><br>&lt;-quit<br><br>server.Shutdown()<br><br>Shutdown 并不会立即退出，他会首先停止监听，并且启动一个定时器，避免新的请求进来，然后关闭空闲链接，等待处理中的请求完成或者如果定时器到了，再退出，和 NGINX 的 平滑退出很像。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-30 12:18:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/71/e5/bcdc382a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>My dream</span>
  </div>
  <div class="_2_QraFYR_0">老师可以讲一下这个不：gomonkey，我看半天都没明白这个打桩是什么意思</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-28 21:30:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/46/26/fd4eda95.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>aebn</span>
  </div>
  <div class="_2_QraFYR_0">谢谢老师分享，努力学习中^_^</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-28 22:26:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/5c/67/4e776ee6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>袁树立</span>
  </div>
  <div class="_2_QraFYR_0">如此一来，每当一个 HTTP 请求被递交时，就都会产生一个新的网络连接。这样做会明显地加重网络服务以及客户端的负载，并会让每个 HTTP 事务都耗费更多的时间。所以，在一般情况下，我们都不要去设置这个DisableKeepAlives字段。<br><br>老师，针对这句话，有个问题。<br>因为我们的服务调用其他内网接口，会通过公司的负载均衡。七层负载均衡是关闭了keep-alive的。所以我们的http就每次都是短链接。   那每次http事务耗费的时间大概是什么量级？  我这里看到，设置了500ms超时的情况下。在频繁请求的场景，每过几分钟就会有一批超时。报net&#47;http  timeout 。<br>用http trace看，是在getConn前就耗费了500ms<br><br>请问，这种情况正常吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个问题太复杂了。你们的网络拓扑、中间件版本和配置以及 Go 程序本身等等都可能会对此影响。你们需要有 request id，然后串起来进行分析。<br><br>定位问题需要定位到某一个或某几行代码。只知道在 getConn 前耗时的话，粒度太粗了。有必要的话，需要跟进 net 包的源码。<br><br>另外，怎样设置需要按照你们的实际情况来。我不知道你们具体因为什么关闭 ka，也许是合理的，也许不是。不论怎样，这相当于放弃了对操作系统底层优化机制的利用。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-23 10:30:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/ywV5EjGPovkbcj9zRmibTKBQjAvCFrKVYMmGfDwGfcz6dmq6Sia1AlHvSX8ydibu2xueLuSen1YVDZSKNib1UTJBsQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>路人</span>
  </div>
  <div class="_2_QraFYR_0">这节老师讲得特别好，特别是问题的衍生能思考到很多go的其他重要模块，比如net、io等。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-14 18:26:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/78/c7/083a3a0b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>新世界</span>
  </div>
  <div class="_2_QraFYR_0">讲的很不错，把http常用的参数注意事项都讲了一遍，这一块是通用技能，无论哪种语言，发送http请求都会有这些参数，很不错</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-25 09:28:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/3b/ba/3b30dcde.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>窝窝头</span>
  </div>
  <div class="_2_QraFYR_0">我一般通过context的cancel函数，同时通过系统信号量来触发cancel事件</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-18 19:15:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/9d/a4/e481ae48.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lesserror</span>
  </div>
  <div class="_2_QraFYR_0">郝林老师，在demo94.go中这个字段的值：<br><br>DualStack: true,<br><br>其中“DualStack”会被编辑器用横线从中穿过，并提示：&#39;DualStack&#39; is deprecated 。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个字段现在已经被废弃了。DualStack 主要的作用是在对方服务器不支持 IPv6 协议或者配置错误的情况下快速回退到 IPv4。现在这个行为已经是默认的了。<br><br>如果想关掉，你可以通过把 FallbackDelay 字段设置为负值。我稍后会更新一下源码。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-02 22:34:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/7c/e5/3dca2495.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>上山的o牛</span>
  </div>
  <div class="_2_QraFYR_0">学习中，衍生的内容可以看一周</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-12 22:31:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>qiushye</span>
  </div>
  <div class="_2_QraFYR_0">http.Transport里没有MaxConnsPerHost字段了，article36&#47;q1的程序运行报错</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Go 1.13 里依然有啊。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-11 13:09:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/ac/95/9b3e3859.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Timo</span>
  </div>
  <div class="_2_QraFYR_0">demo91.go  例子中<br>reqStrTpl := `HEAD &#47; HTTP&#47;1.1<br>Accept: *&#47;*<br>Accept-Encoding: gzip, deflate<br>Connection: keep-alive<br>Host: %s<br>User-Agent: Dialer&#47;%s<br>`<br>协议和头信息之间要空两行，才能正常发出信息</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-13 16:30:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/20/27/a6932fbe.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>虢國技醬</span>
  </div>
  <div class="_2_QraFYR_0">打卡</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-15 09:39:07</div>
  </div>
</div>
</div>
</li>
</ul>