<audio title="12 _ API 风格（上）：如何设计RESTful API？" src="https://static001.geekbang.org/resource/audio/d8/a3/d8082362ba254eeffc5b5928c2d384a3.mp3" controls="controls"></audio> 
<p>你好，我是孔令飞。从今天开始，我们就要进入实战第二站，开始学习如何设计和开发Go项目开发中的基础功能了。接下来两讲，我们一起来看下如何设计应用的API风格。</p><p>绝大部分的Go后端服务需要编写API接口，对外提供服务。所以在开发之前，我们需要确定一种API风格。API风格也可以理解为API类型，目前业界常用的API风格有三种：REST、RPC和GraphQL。我们需要根据项目需求，并结合API风格的特点，确定使用哪种API风格，这对以后的编码实现、通信方式和通信效率都有很大的影响。</p><p>在Go项目开发中，用得最多的是REST和RPC，我们在IAM实战项目中也使用了REST和RPC来构建示例项目。接下来的两讲，我会详细介绍下REST和RPC这两种风格，如果你对GraphQL感兴趣，<a href="https://graphql.cn/">GraphQL中文官网</a>有很多文档和代码示例，你可以自行学习。</p><p>这一讲，我们先来看下RESTful API风格设计，下一讲再介绍下RPC API风格。</p><h2>RESTful API介绍</h2><p>在回答“RESTful API是什么”之前，我们先来看下REST是什么意思：REST代表的是表现层状态转移（REpresentational State Transfer），由Roy Fielding在他的论文<a href="https://www.ics.uci.edu/~fielding/pubs/dissertation/top.htm">《Architectural Styles and the Design of Network-based Software Architectures》</a>里提出。REST本身并没有创造新的技术、组件或服务，它只是一种软件架构风格，是一组架构约束条件和原则，而不是技术框架。</p><!-- [[[read_end]]] --><p><strong>REST有一系列规范，满足这些规范的API均可称为RESTful API</strong>。REST规范把所有内容都视为资源，也就是说网络上一切皆资源。REST架构对资源的操作包括获取、创建、修改和删除，这些操作正好对应HTTP协议提供的GET、POST、PUT和DELETE方法。HTTP动词与 REST风格CRUD的对应关系见下表：</p><p><img src="https://static001.geekbang.org/resource/image/40/92/409164157ce4cde3131f0236d660e092.png?wh=1754x582" alt=""></p><p>REST风格虽然适用于很多传输协议，但在实际开发中，由于REST天生和HTTP协议相辅相成，因此HTTP协议已经成了实现RESTful API事实上的标准。所以，REST具有以下核心特点：</p><ul>
<li>
<p>以资源(resource)为中心，所有的东西都抽象成资源，所有的行为都应该是在资源上的CRUD操作。</p>
<ul>
<li>资源对应着面向对象范式里的对象，面向对象范式以对象为中心。</li>
<li>资源使用URI标识，每个资源实例都有一个唯一的URI标识。例如，如果我们有一个用户，用户名是admin，那么它的URI标识就可以是/users/admin。</li>
</ul>
</li>
<li>
<p>资源是有状态的，使用JSON/XML等在HTTP Body里表征资源的状态。</p>
</li>
<li>
<p>客户端通过四个HTTP动词，对服务器端资源进行操作，实现“表现层状态转化”。</p>
</li>
<li>
<p>无状态，这里的无状态是指每个RESTful API请求都包含了所有足够完成本次操作的信息，服务器端无须保持session。无状态对于服务端的弹性扩容是很重要的。</p>
</li>
</ul><p>因为怕你弄混概念，这里强调下REST和RESTful API的区别：<strong>REST是一种规范，而RESTful API则是满足这种规范的API接口。</strong></p><h2>RESTful API设计原则</h2><p>上面我们说了，RESTful API就是满足REST规范的API，由此看来，RESTful API的核心是规范，那么具体有哪些规范呢？</p><p>接下来，我就从URI设计、API版本管理等七个方面，给你详细介绍下RESTful API的设计原则，然后再通过一个示例来帮助你快速启动一个RESTful API服务。希望你学完这一讲之后，对如何设计RESTful API有一个清楚的认知。</p><h3>URI设计</h3><p>资源都是使用URI标识的，我们应该按照一定的规范来设计URI，通过规范化可以使我们的API接口更加易读、易用。以下是URI设计时，应该遵循的一些规范：</p><ul>
<li>
<p>资源名使用名词而不是动词，并且用名词复数表示。资源分为Collection和Member两种。</p>
<ul>
<li>Collection：一堆资源的集合。例如我们系统里有很多用户（User）,这些用户的集合就是Collection。Collection的URI标识应该是 <code>域名/资源名复数</code>, 例如<code>https:// iam.api.marmotedu.com/users</code>。</li>
<li>Member：单个特定资源。例如系统中特定名字的用户，就是Collection里的一个Member。Member的URI标识应该是 <code>域名/资源名复数/资源名称</code>, 例如<code>https:// iam.api.marmotedu/users/admin</code>。</li>
</ul>
</li>
<li>
<p>URI结尾不应包含<code>/</code>。</p>
</li>
<li>
<p>URI中不能出现下划线 <code>_</code>，必须用中杠线 <code>-</code>代替（有些人推荐用 <code>_</code>，有些人推荐用 <code>-</code>，统一使用一种格式即可，我比较推荐用 <code>-</code>）。</p>
</li>
<li>
<p>URI路径用小写，不要用大写。</p>
</li>
<li>
<p>避免层级过深的URI。超过2层的资源嵌套会很乱，建议将其他资源转化为<code>?</code>参数，比如：</p>
</li>
</ul><pre><code>/schools/tsinghua/classes/rooma/students/zhang # 不推荐
/students?school=qinghua&amp;class=rooma # 推荐
</code></pre><p>这里有个地方需要注意：在实际的API开发中，可能你会发现有些操作不能很好地映射为一个REST资源，这时候，你可以参考下面的做法。</p><ul>
<li>将一个操作变成资源的一个属性，比如想在系统中暂时禁用某个用户，可以这么设计URI：<code>/users/zhangsan?active=false</code>。</li>
<li>将操作当作是一个资源的嵌套资源，比如一个GitHub的加星操作：</li>
</ul><pre><code>PUT /gists/:id/star # github star action
DELETE /gists/:id/star # github unstar action
</code></pre><ul>
<li>如果以上都不能解决问题，有时可以打破这类规范。比如登录操作，登录不属于任何一个资源，URI可以设计为：/login。</li>
</ul><p>在设计URI时，如果你遇到一些不确定的地方，推荐你参考  <a href="https://developer.github.com/v3/">GitHub标准RESTful API</a>。</p><h3>REST资源操作映射为HTTP方法</h3><p>基本上RESTful API都是使用HTTP协议原生的GET、PUT、POST、DELETE来标识对资源的CRUD操作的，形成的规范如下表所示：</p><p><img src="https://static001.geekbang.org/resource/image/d9/2d/d970bcd53d2827b7f2096e639d5fa82d.png?wh=1524x698" alt=""></p><p>对资源的操作应该满足安全性和幂等性：</p><ul>
<li>安全性：不会改变资源状态，可以理解为只读的。</li>
<li>幂等性：执行1次和执行N次，对资源状态改变的效果是等价的。</li>
</ul><p>使用不同HTTP方法时，资源操作的安全性和幂等性对照见下表：</p><p><img src="https://static001.geekbang.org/resource/image/b7/e1/b746421291654e4d2e51509b885c4ee1.png?wh=1434x448" alt=""></p><p>在使用HTTP方法的时候，有以下两点需要你注意：</p><ul>
<li>GET返回的结果，要尽量可用于PUT、POST操作中。例如，用GET方法获得了一个user的信息，调用者修改user的邮件，然后将此结果再用PUT方法更新。这要求GET、PUT、POST操作的资源属性是一致的。</li>
<li>如果对资源进行状态/属性变更，要用PUT方法，POST方法仅用来创建或者批量删除这两种场景。</li>
</ul><p>在设计API时，经常会有批量删除的需求，需要在请求中携带多个需要删除的资源名，但是HTTP的DELETE方法不能携带多个资源名，这时候可以通过下面三种方式来解决：</p><ul>
<li>发起多个DELETE请求。</li>
<li>操作路径中带多个id，id之间用分隔符分隔, 例如：<code>DELETE /users?ids=1,2,3</code>  。</li>
<li>直接使用POST方式来批量删除，body中传入需要删除的资源列表。</li>
</ul><p>其中，第二种是我最推荐的方式，因为使用了匹配的DELETE动词，并且不需要发送多次DELETE请求。</p><p>你需要注意的是，这三种方式都有各自的使用场景，你可以根据需要自行选择。如果选择了某一种方式，那么整个项目都需要统一用这种方式。</p><h3>统一的返回格式</h3><p>一般来说，一个系统的RESTful API会向外界开放多个资源的接口，每个接口的返回格式要保持一致。另外，每个接口都会返回成功和失败两种消息，这两种消息的格式也要保持一致。不然，客户端代码要适配不同接口的返回格式，每个返回格式又要适配成功和失败两种消息格式，会大大增加用户的学习和使用成本。</p><p>返回的格式没有强制的标准，你可以根据实际的业务需要返回不同的格式。本专栏 <strong>第19讲</strong> 中会推荐一种返回格式，它也是业界最常用和推荐的返回格式。</p><h3>API 版本管理</h3><p>随着时间的推移、需求的变更，一个API往往满足不了现有的需求，这时候就需要对API进行修改。对API进行修改时，不能影响其他调用系统的正常使用，这就要求API变更做到向下兼容，也就是新老版本共存。</p><p>但在实际场景中，很可能会出现同一个API无法向下兼容的情况。这时候最好的解决办法是从一开始就引入API版本机制，当不能向下兼容时，就引入一个新的版本，老的版本则保留原样。这样既能保证服务的可用性和安全性，同时也能满足新需求。</p><p>API版本有不同的标识方法，在RESTful API开发中，通常将版本标识放在如下3个位置：</p><ul>
<li>URL中，比如<code>/v1/users</code>。</li>
<li>HTTP Header中，比如<code>Accept: vnd.example-com.foo+json; version=1.0</code>。</li>
<li>Form参数中，比如<code>/users?version=v1</code>。</li>
</ul><p>我们这门课中的版本标识是放在URL中的，比如<code>/v1/users</code>，这样做的好处是很直观，GitHub、Kubernetes、Etcd等很多优秀的API均采用这种方式。</p><p>这里要注意，有些开发人员不建议将版本放在URL中，因为他们觉得不同的版本可以理解成同一种资源的不同表现形式，所以应该采用同一个URI。对于这一点，没有严格的标准，根据项目实际需要选择一种方式即可。</p><h3>API命名</h3><p>API通常的命名方式有三种，分别是驼峰命名法(serverAddress)、蛇形命名法(server_address)和脊柱命名法(server-address)。</p><p>驼峰命名法和蛇形命名法都需要切换输入法，会增加操作的复杂性，也容易出错，所以这里建议用脊柱命名法。GitHub API用的就是脊柱命名法，例如  <a href="https://docs.github.com/en/rest/reference/actions#get-allowed-actions-for-an-organization">selected-actions</a>。</p><h3>统一分页/过滤/排序/搜索功能</h3><p>REST资源的查询接口，通常情况下都需要实现分页、过滤、排序、搜索功能，因为这些功能是每个REST资源都能用到的，所以可以实现为一个公共的API组件。下面来介绍下这些功能。</p><ul>
<li>分页：在列出一个Collection下所有的Member时，应该提供分页功能，例如<code>/users?offset=0&amp;limit=20</code>（limit，指定返回记录的数量；offset，指定返回记录的开始位置）。引入分页功能可以减少API响应的延时，同时可以避免返回太多条目，导致服务器/客户端响应特别慢，甚至导致服务器/客户端crash的情况。</li>
<li>过滤：如果用户不需要一个资源的全部状态属性，可以在URI参数里指定返回哪些属性，例如<code>/users?fields=email,username,address</code>。</li>
<li>排序：用户很多时候会根据创建时间或者其他因素，列出一个Collection中前100个Member，这时可以在URI参数中指明排序参数，例如<code>/users?sort=age,desc</code>。</li>
<li>搜索：当一个资源的Member太多时，用户可能想通过搜索，快速找到所需要的Member，或着想搜下有没有名字为xxx的某类资源，这时候就需要提供搜索功能。搜索建议按模糊匹配来搜索。</li>
</ul><h3>域名</h3><p>API的域名设置主要有两种方式：</p><ul>
<li><code>https://marmotedu.com/api</code>，这种方式适合API将来不会有进一步扩展的情况，比如刚开始marmotedu.com域名下只有一套API系统，未来也只有这一套API系统。</li>
<li><code>https://iam.api.marmotedu.com</code>，如果marmotedu.com域名下未来会新增另一个系统API，这时候最好的方式是每个系统的API拥有专有的API域名，比如：<code>storage.api.marmotedu.com</code>，<code>network.api.marmotedu.com</code>。腾讯云的域名就是采用这种方式。</li>
</ul><p>到这里，我们就将REST设计原则中的核心原则讲完了，这里有个需要注意的点：不同公司、不同团队、不同项目可能采取不同的REST设计原则，以上所列的基本上都是大家公认的原则。</p><p>REST设计原则中，还有一些原则因为内容比较多，并且可以独立成模块，所以放在后面来讲。比如  RESTful API安全性、状态返回码和认证等。</p><h2>REST示例</h2><p>上面介绍了一些概念和原则，这里我们通过一个“Hello World”程序，来教你用Go快速启动一个RESTful API服务，示例代码存放在<a href="https://github.com/marmotedu/gopractise-demo/blob/main/apistyle/ping/main.go">gopractise-demo/apistyle/ping/main.go</a>。</p><pre><code>package main

import (
	&quot;log&quot;
	&quot;net/http&quot;
)

func main() {
	http.HandleFunc(&quot;/ping&quot;, pong)
	log.Println(&quot;Starting http server ...&quot;)
	log.Fatal(http.ListenAndServe(&quot;:50052&quot;, nil))
}

func pong(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte(&quot;pong&quot;))
}
</code></pre><p>在上面的代码中，我们通过http.HandleFunc，向HTTP服务注册了一个pong handler，在pong handler中，我们编写了真实的业务代码：返回pong字符串。</p><p>创建完main.go文件后，在当前目录下执行go run main.go启动HTTP服务，在一个新的Linux终端下发送HTTP请求，进行使用curl命令测试：</p><pre><code>$ curl http://127.0.0.1:50052/ping
pong
</code></pre><h2>总结</h2><p>这一讲，我介绍了两种常用API风格中的一种，RESTful API。REST是一种API规范，而RESTful API则是满足这种规范的API接口，RESTful API的核心是规范。</p><p>在REST规范中，资源通过URI来标识，资源名使用名词而不是动词，并且用名词复数表示，资源都是分为Collection和Member两种。RESTful API中，分别使用POST、DELETE、PUT、GET来表示REST资源的增删改查，HTTP方法、Collection、Member不同组合会产生不同的操作，具体的映射你可以看下 <strong>REST资源操作映射为HTTP方法</strong> 部分的表格。</p><p>为了方便用户使用和理解，每个RESTful API的返回格式、错误和正确消息的返回格式，都应该保持一致。RESTful API需要支持API版本，并且版本应该能够向前兼容，我们可以将版本号放在URL中、HTTP Header中、Form参数中，但这里我建议将版本号放在URL中，例如 <code>/v1/users</code>，这种形式比较直观。</p><p>另外，我们可以通过脊柱命名法来命名API接口名。对于一个REST资源，其查询接口还应该支持分页/过滤/排序/搜索功能，这些功能可以用同一套机制来实现。 API的域名可以采用 <code>https://marmotedu.com/api</code> 和 <code>https://iam.api.marmotedu.com</code> 两种格式。</p><p>最后，在Go中我们可以使用net/http包来快速启动一个RESTful API服务。</p><h2>课后练习</h2><ol>
<li>使用net/http包，快速实现一个RESTful API服务，并实现/hello接口，该接口会返回“Hello World”字符串。</li>
<li>思考一下，RESTful API这种API风格是否能够满足你当前的项目需要，如果不满足，原因是什么？</li>
</ol><p>期待在留言区看到你的思考和答案，也欢迎和我一起探讨关于RESTful API相关的问题，我们下一讲见！</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/47/bd/5c34df95.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>管清麟</span>
  </div>
  <div class="_2_QraFYR_0">我觉得老师可以专门开个gin的专栏，看了一下iam源码写的真好。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 业界No.1 没有之一</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-01 16:58:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/a9/9a/5df6cc22.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>h</span>
  </div>
  <div class="_2_QraFYR_0">```避免层级过深的 URI。超过 2 层的资源嵌套会很乱，建议将其他资源转化为?参数，<br>比如：<br>&#47;schools&#47;tsinghua&#47;classes&#47;rooma&#47;students&#47;zhang # 不推荐<br>&#47;students?school=qinghua&amp;class=rooma # 推荐```<br>老师这句话，我想起来了我一个接口，是和用户组相关的。<br>我有一个组的接口，它本身有自己的增删查改等几个标准的接口<br>组表和用户表是个多对多关系，我要写三个接口，分别是：显示组用户，添加组用户，删除组用户。<br>没看这篇文章之前我写成了 get &#47;group&#47;user&#47;:uid  post &#47;group&#47;user&#47;:uid  delete &#47;group&#47;user&#47;:uid 并且还分别添加了groupid参数，看完老师说的层级过深那段话之后觉得我以前弄的确实有问题，思考之后新的想法如下：<br>显示组用户应该用 get &#47;group&#47;:gid?fields=user来显示组详情，这里可以通过你说的过滤功能只要显示用户，组其他字段不显示。<br>添加组用户和删除组用户好像正常情况来讲都是put &#47;group&#47;:gid  就是修改组的那个接口，但是仔细一想，增加用户和删除用户都用 put &#47;group&#47;:gid有点奇怪，而且没办法区分删除还是增加，或者把在后面把现在所有组列出来，比如现在组有用户1,2 增加用户3就是  put &#47;group&#47;:gid?users=1,2,3 ， 假如现在有用户1，2，3，删除3就是 put &#47;group&#47;:gid?users=1,2 <br>其实不止组，很多有外键关系的好像都存在我这个情况，比如作者表和书籍表，作者表和书籍表有自己的增删查改，那如果我要 查看某作者所有书籍，增加作者一本新书，删除作者一本书<br>其实说了这么多 就是想问下老师  显示组用户，添加组用户，删除组用户这三个例子你觉得该怎么设计接口，能指教一下如果是你会怎么设计这三个接口以及原因吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 显示组用户：GET &#47;groups&#47;:gid<br>添加组用户：POST &#47;groups&#47;:gid&#47;users&#47;:uid<br>删除组用户：DELETE &#47;groups&#47;:gid&#47;users?uids=1,2,3<br>&#47;groups&#47;:gid&#47;users&#47;:uid 这个是其实2层资源嵌套。<br><br>这样设计原因是：符合现实世界的分层关系，逻辑更合理<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-29 17:15:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/52/40/e57a736e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>pedro</span>
  </div>
  <div class="_2_QraFYR_0">在过去的经验中，RESTful API对于动词性API不能很好的work，比如说修改密码，重置密码等，很难通过URL和HTTP方法表征出来。<br>但是对于Github，豆瓣等资源性API，大量的都是资源获取与删除，就特别适合RESTful。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，可以将这些动词抽象成一个属性</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-22 09:43:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/03/30/7b658349.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>cxn</span>
  </div>
  <div class="_2_QraFYR_0">URI中使用_踩过坑,谷歌业务中URI中不能出现_</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-22 09:19:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/c2/e0/7188aa0a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>blackpiglet</span>
  </div>
  <div class="_2_QraFYR_0">感觉GraphQL API会比较适合接口变动比较频繁的开发环境，这种API设计看起来基本不用考虑版本兼容的问题，不知道在实际的使用场景中，是不是基于这个原因选择GraphQL，放弃RESTFul的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 大体不差</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-02 10:40:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/3c/bd/ea0b59f6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span></span>
  </div>
  <div class="_2_QraFYR_0">像RESTful 中 对某个资源的获取 api 中，针对某个特定资源的获取是尽量精简还是需要详细？<br>如 需求是 获取最近十条的最新数据<br>应该是 GET: &#47;data?order=createdAt,desc <br>还是使用 GET: &#47;last10data <br>像在 前后端分离的埸景下<br>是尽量希望不要暴露太多细节给前端好<br>还是尽量提供更多的参数细节可让前端调用好？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 选择GET: &#47;data?order=createdAt,desc。<br><br>前后端分离的时候，尽量设计的通用些。甚至可以不考虑前端。<br><br>前端需要的参数，是接口返回参数的一个子集。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-23 00:24:54</div>
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
  <div class="_2_QraFYR_0">老师，再请教一下, 关于动词性接口  “是的，可以将这些动词抽象成一个属性”<br>这里 抽象为属性是什么意思。<br>比如 有一个视频  &#47;video&#47;12345.mp4 , 现在要提供一个 禁播的操作（非删除），该如何操作。<br>辛苦老师<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: &#47;video&#47;12345.mp4?enabled=false<br><br>或者：<br><br>&#47;video&#47;:video&#47;disable</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-23 10:51:33</div>
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
  <div class="_2_QraFYR_0">RESTful API，符合REST风格的API。<br>对比传统API风格的好处：简单、低耦合、轻量、自解释、无状态。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 总结很到位！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-07 00:40:28</div>
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
  <div class="_2_QraFYR_0">request.Body只能读取一次。 有没有比较优雅的解决办法呢？ 网上查到的都是读取后再手动set回去</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以深度copy，具体你可以参考下gin的dump中间件是如何实现打印request.Body的：https:&#47;&#47;github.com&#47;tpkeeper&#47;gin-dump<br><br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-28 20:45:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/97/c5/84491beb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>罗峰</span>
  </div>
  <div class="_2_QraFYR_0">先做个伸手党，iam如何支持流量暴增这种情况，或者说如果提高负载能力</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 容器化部署，水平扩容</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-25 22:25:45</div>
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
  <div class="_2_QraFYR_0">如果比较复杂一点的查询，比如需要join表的情况，如何做扩展比较好呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以试试gorm的Joins方法。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-24 23:05:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/20/75/15/7b22b6c1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>穿山乙</span>
  </div>
  <div class="_2_QraFYR_0">感觉平时开发不是特别遵守rest规范，基本上通了就行</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-24 17:20:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_e4ce15</span>
  </div>
  <div class="_2_QraFYR_0">一直没有想明白  初期部署的iam到现在有什么用呢 一直是理论也米有涉及到跟iam有关的 除了目录结构</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 后面会有。<br><br>部署的iam只是打通整个编译环境，后面有需要可以自动动手魔改代码</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-22 15:18:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/bc/0d/e65ca230.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>👻</span>
  </div>
  <div class="_2_QraFYR_0">restful的写法好看是好看  但是说实话实用性不好   徒增工作量</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 其实工作量没有增加多少。但是会给后续的接口维护，调用带来很多易用性的提升</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-22 11:55:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/9d/81/d748b7eb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>千锤百炼领悟之极限</span>
  </div>
  <div class="_2_QraFYR_0">&#47;** <br> * http restful 接口 &#47;hello 打印Hello World<br> * 运行 go run hello_world.go<br> * 访问 curl http:&#47;&#47;localhost:50052&#47;hello<br>*&#47;<br>package main<br><br>&#47;&#47; 引入代码依赖<br>import (<br>    &quot;log&quot;<br>    &quot;net&#47;http&quot;<br>)<br><br>&#47;&#47; 启动http服务<br>func main(){<br>    http.HandleFunc(&quot;&#47;hello&quot;, hello)<br>    log.Println(&quot;Starting http server ...&quot;)<br>    log.Fatal(http.ListenAndServe(&quot;:50052&quot;, nil))<br>}<br><br>&#47;&#47; 打印Hello World<br>func hello(w http.ResponseWriter, r *http.Request){<br>    w.Write([]byte(&quot;Hello World&quot;))<br>}</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-18 22:12:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/59/42/9f0c7fe4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>何长杰Atom</span>
  </div>
  <div class="_2_QraFYR_0">老师，关于接口的冥等性，一直不太理解其内涵，比如：<br>POST不是冥等的，网上说会N次调用会有N个资源创建，但如果不允许重复也不会创建N个资源。<br>这里说的资源的状态改变效果该怎么理解？<br>还有谈论的冥等的目的是啥？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 幂等性是指前后两次执行的效果是一样的。<br><br>接口幂等性，能提高接口的安全和可重试性。<br><br>非幂等性的接口会带来以下问题：<br>1. 请求失败，再次请求，可能会重复创建资源，这样上一次的资源创建爱你就变成了一个脏数据<br>2. 以为前后2次执行结果不一样，就导致这个接口失败后，不能重试</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-17 17:53:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/xfclWEPQ7szTZnKqnX9icSbgDWV0VAib3Cyo8Vg0OG3Usby88ic7ZgO2ho5lj0icOWI4JeJ70zUBiaTW1xh1UCFRPqA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_6bdb4e</span>
  </div>
  <div class="_2_QraFYR_0">想问一下如何理解“直接使用 POST 方式来批量删除，body 中传入需要删除的资源列表”这句话呢，我的理解POST是用来资源注册，也就是增删改查中的增，这个是在body中加入待删除的资源列表，然后内部代码处理这个逻辑吗，也就是其实内部是删除的逻辑？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的。<br><br>因为DELETE不能传入多个资源，所以如果是批量删除，需要传入多个资源的时候，可以考虑使用POST，将资源列表放在body中。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-18 09:30:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/1b/44/82acaafc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>无为</span>
  </div>
  <div class="_2_QraFYR_0">老师, 有的时候一个请求不只是单纯某一种资源, 还需要一些关联资源, 这个时候怎么处理比较好?<br><br>返回结果通过参数有针对性的添加更多的信息? </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以这么搞</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-21 19:44:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/1b/44/82acaafc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>无为</span>
  </div>
  <div class="_2_QraFYR_0">老师, 有的时候一个请求不只是单纯某一种资源, 还需要一些关联资源, 这个时候怎么处理比较好?<br><br>返回结果通过参数有针对性的添加更多的信息? </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，可以这么搞</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-21 19:43:45</div>
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
  <div class="_2_QraFYR_0">涉及到敏感信息的Get请求，比如查询用户信息，就算敏感信息（例如手机号码已脱敏），在安全测试也是建议改用Post请求，因为请求的唯一标识一般会携带在url上。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-10 21:21:58</div>
  </div>
</div>
</div>
</li>
</ul>