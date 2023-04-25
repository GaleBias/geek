<audio title="第32讲 _ RPC协议综述：远在天边，近在眼前" src="https://static001.geekbang.org/resource/audio/97/2d/97718e118ab62bfaa6283e753ee9392d.mp3" controls="controls"></audio> 
<p>前面我们讲了容器网络如何实现跨主机互通，以及微服务之间的相互调用。</p><p><img src="https://static001.geekbang.org/resource/image/a0/0d/a0763d50fc4e8dcec37ae25a2f6cc60d.jpeg?wh=1582*1080" alt=""></p><p>网络是打通了，那服务之间的互相调用，该怎么实现呢？你可能说，咱不是学过Socket吗。服务之间分调用方和被调用方，我们就建立一个TCP或者UDP的连接，不就可以通信了？</p><p><img src="https://static001.geekbang.org/resource/image/87/ea/87c8ae36ae1b42653565008fc47aceea.jpg?wh=1626*2172" alt=""></p><p>你仔细想一下，这事儿没这么简单。我们就拿最简单的场景，客户端调用一个加法函数，将两个整数加起来，返回它们的和。</p><p>如果放在本地调用，那是简单的不能再简单了，只要稍微学过一种编程语言，三下五除二就搞定了。但是一旦变成了远程调用，门槛一下子就上去了。</p><p>首先你要会Socket编程，至少先要把咱们这门网络协议课学一下，然后再看N本砖头厚的Socket程序设计的书，学会咱们学过的几种Socket程序设计的模型。这就使得本来大学毕业就能干的一项工作，变成了一件五年工作经验都不一定干好的工作，而且，搞定了Socket程序设计，才是万里长征的第一步。后面还有很多问题呢！</p><h2>如何解决这五个问题？</h2><h3>问题一：如何规定远程调用的语法？</h3><p>客户端如何告诉服务端，我是一个加法，而另一个是乘法。我是用字符串“add”传给你，还是传给你一个整数，比如1表示加法，2表示乘法？服务端该如何告诉客户端，我的这个加法，目前只能加整数，不能加小数，不能加字符串；而另一个加法“add1”，它能实现小数和整数的混合加法。那返回值是什么？正确的时候返回什么，错误的时候又返回什么？</p><!-- [[[read_end]]] --><h3>问题二：如果传递参数？</h3><p>我是先传两个整数，后传一个操作符“add”，还是先传操作符，再传两个整数？是不是像咱们数据结构里一样，如果都是UDP，想要实现一个逆波兰表达式，放在一个报文里面还好，如果是TCP，是一个流，在这个流里面，如何将两次调用进行分界？什么时候是头，什么时候是尾？把这次的参数和上次的参数混了起来，TCP一端发送出去的数据，另外一端不一定能一下子全部读取出来。所以，怎么才算读完呢？</p><h3>问题三：如何表示数据？</h3><p>在这个简单的例子中，传递的就是一个固定长度的int值，这种情况还好，如果是变长的类型，是一个结构体，甚至是一个类，应该怎么办呢？如果是int，不同的平台上长度也不同，该怎么办呢？</p><p>在网络上传输超过一个Byte的类型，还有大端Big Endian和小端Little Endian的问题。</p><p>假设我们要在32位四个Byte的一个空间存放整数1，很显然只要一个Byte放1，其他三个Byte放0就可以了。那问题是，最后一个Byte放1呢，还是第一个Byte放1呢？或者说1作为最低位，应该是放在32位的最后一个位置呢，还是放在第一个位置呢？</p><p>最低位放在最后一个位置，叫作Little Endian，最低位放在第一个位置，叫作Big Endian。TCP/IP协议栈是按照Big Endian来设计的，而X86机器多按照Little Endian来设计的，因而发出去的时候需要做一个转换。</p><h3>问题四：如何知道一个服务端都实现了哪些远程调用？从哪个端口可以访问这个远程调用？</h3><p>假设服务端实现了多个远程调用，每个可能实现在不同的进程中，监听的端口也不一样，而且由于服务端都是自己实现的，不可能使用一个大家都公认的端口，而且有可能多个进程部署在一台机器上，大家需要抢占端口，为了防止冲突，往往使用随机端口，那客户端如何找到这些监听的端口呢？</p><h3>问题五：发生了错误、重传、丢包、性能等问题怎么办？</h3><p>本地调用没有这个问题，但是一旦到网络上，这些问题都需要处理，因为网络是不可靠的，虽然在同一个连接中，我们还可通过TCP协议保证丢包、重传的问题，但是如果服务器崩溃了又重启，当前连接断开了，TCP就保证不了了，需要应用自己进行重新调用，重新传输会不会同样的操作做两遍，远程调用性能会不会受影响呢？</p><h2>协议约定问题</h2><p>看到这么多问题，你是不是想起了我<a href="https://time.geekbang.org/column/article/7581">第一节</a>讲过的这张图。</p><p><img src="https://static001.geekbang.org/resource/image/7b/33/7be56272a7e738b6cfe5bcbf658c3933.jpg?wh=2643*380" alt=""></p><p>本地调用函数里有很多问题，比如词法分析、语法分析、语义分析等等，这些编译器本来都能帮你做了。但是在远程调用中，这些问题你都需要重新操心。</p><p>很多公司的解决方法是，弄一个核心通信组，里面都是Socket编程的大牛，实现一个统一的库，让其他业务组的人来调用，业务的人不需要知道中间传输的细节。通信双方的语法、语义、格式、端口、错误处理等，都需要调用方和被调用方开会协商，双方达成一致。一旦有一方改变，要及时通知对方，否则通信就会有问题。</p><p>可是不是每一个公司都有这种大牛团队，往往只有大公司才配得起，那有没有已经实现好的框架可以使用呢？</p><p>当然有。一个大牛Bruce Jay Nelson写了一篇论文<a href="http://www.cs.cmu.edu/~dga/15-712/F07/papers/birrell842.pdf">Implementing Remote Procedure Calls</a>，定义了RPC的调用标准。后面所有RPC框架，都是按照这个标准模式来的。</p><p><img src="https://static001.geekbang.org/resource/image/29/db/2933bbd1ee6471b6de3824bb86f6d0db.jpg?wh=1999*707" alt=""></p><p>当客户端的应用想发起一个远程调用时，它实际是通过本地调用本地调用方的Stub。它负责将调用的接口、方法和参数，通过约定的协议规范进行编码，并通过本地的RPCRuntime进行传输，将调用网络包发送到服务器。</p><p>服务器端的RPCRuntime收到请求后，交给提供方Stub进行解码，然后调用服务端的方法，服务端执行方法，返回结果，提供方Stub将返回结果编码后，发送给客户端，客户端的RPCRuntime收到结果，发给调用方Stub解码得到结果，返回给客户端。</p><p>这里面分了三个层次，对于用户层和服务端，都像是本地调用一样，专注于业务逻辑的处理就可以了。对于Stub层，处理双方约定好的语法、语义、封装、解封装。对于RPCRuntime，主要处理高性能的传输，以及网络的错误和异常。</p><p>最早的RPC的一种实现方式称为Sun RPC或ONC RPC。Sun公司是第一个提供商业化RPC库和 RPC编译器的公司。这个RPC框架是在NFS协议中使用的。</p><p>NFS（Network File System）就是网络文件系统。要使NFS成功运行，要启动两个服务端，一个是mountd，用来挂载文件路径；一个是nfsd，用来读写文件。NFS可以在本地mount一个远程的目录到本地的一个目录，从而本地的用户在这个目录里面写入、读出任何文件的时候，其实操作的是远程另一台机器上的文件。</p><p>操作远程和远程调用的思路是一样的，就像操作本地一样。所以NFS协议就是基于RPC实现的。当然无论是什么RPC，底层都是Socket编程。</p><p><img src="https://static001.geekbang.org/resource/image/2a/eb/2a0fd84c2d3dced623511e2a5226d0eb.jpg?wh=2366*1704" alt=""></p><p>XDR（External Data Representation，外部数据表示法）是一个标准的数据压缩格式，可以表示基本的数据类型，也可以表示结构体。</p><p>这里是几种基本的数据类型。</p><p><img src="https://static001.geekbang.org/resource/image/4a/af/4a649954fea1cee22fcfa8bdb34c03af.jpg?wh=3514*1668" alt=""></p><p>在RPC的调用过程中，所有的数据类型都要封装成类似的格式。而且RPC的调用和结果返回，也有严格的格式。</p><ul>
<li>
<p>XID唯一标识一对请求和回复。请求为0，回复为1。</p>
</li>
<li>
<p>RPC有版本号，两端要匹配RPC协议的版本号。如果不匹配，就会返回Deny，原因就是RPC_MISMATCH。</p>
</li>
<li>
<p>程序有编号。如果服务端找不到这个程序，就会返回PROG_UNAVAIL。</p>
</li>
<li>
<p>程序有版本号。如果程序的版本号不匹配，就会返回PROG_MISMATCH。</p>
</li>
<li>
<p>一个程序可以有多个方法，方法也有编号，如果找不到方法，就会返回PROC_UNAVAIL。</p>
</li>
<li>
<p>调用需要认证鉴权，如果不通过，则Deny。</p>
</li>
<li>
<p>最后是参数列表，如果参数无法解析，则返回GABAGE_ARGS。</p>
</li>
</ul><p><img src="https://static001.geekbang.org/resource/image/c7/65/c724675527afdbd43964bdf24684fa65.jpg?wh=3777*1506" alt=""></p><p>为了可以成功调用RPC，在客户端和服务端实现RPC的时候，首先要定义一个双方都认可的程序、版本、方法、参数等。</p><p><img src="https://static001.geekbang.org/resource/image/5c/58/5c3ebb31ac4415d7895247bf8758fa58.jpg?wh=3849*1878" alt=""></p><p>如果还是上面的加法，则双方约定为一个协议定义文件，同理如果是NFS、mount和读写，也会有类似的定义。</p><p>有了协议定义文件，ONC RPC会提供一个工具，根据这个文件生成客户端和服务器端的Stub程序。</p><p><img src="https://static001.geekbang.org/resource/image/27/b9/27dc1ccd0481408055c87e0e5d8b02b9.jpg?wh=4035*2151" alt=""></p><p>最下层的是XDR文件，用于编码和解码参数。这个文件是客户端和服务端共享的，因为只有双方一致才能成功通信。</p><p>在客户端，会调用clnt_create创建一个连接，然后调用add_1，这是一个Stub函数，感觉是在调用本地一样。其实是这个函数发起了一个RPC调用，通过调用clnt_call来调用ONC RPC的类库，来真正发送请求。调用的过程非常复杂，一会儿我详细说这个。</p><p>当然服务端也有一个Stub程序，监听客户端的请求，当调用到达的时候，判断如果是add，则调用真正的服务端逻辑，也即将两个数加起来。</p><p>服务端将结果返回服务端的Stub，这个Stub程序发送结果给客户端，客户端的Stub程序正在等待结果，当结果到达客户端Stub，就将结果返回给客户端的应用程序，从而完成整个调用过程。</p><p>有了这个RPC的框架，前面五个问题中的前三个“如何规定远程调用的语法？”“如何传递参数？”以及“如何表示数据？”基本解决了，这三个问题我们统称为<strong>协议约定问题</strong>。</p><h2>传输问题</h2><p>但是错误、重传、丢包、性能等问题还没有解决，这些问题我们统称为<strong>传输问题</strong>。这个就不用Stub操心了，而是由ONC RPC的类库来实现。这是大牛们实现的，我们只要调用就可以了。</p><p><img src="https://static001.geekbang.org/resource/image/33/e4/33e1afe4a79e81096e09b850424930e4.jpg?wh=3862*1882" alt=""></p><p>在这个类库中，为了解决传输问题，对于每一个客户端，都会创建一个传输管理层，而每一次RPC调用，都会是一个任务，在传输管理层，你可以看到熟悉的队列机制、拥塞窗口机制等。</p><p>由于在网络传输的时候，经常需要等待，因而同步的方式往往效率比较低，因而也就有Socket的异步模型。为了能够异步处理，对于远程调用的处理，往往是通过状态机来实现的。只有当满足某个状态的时候，才进行下一步，如果不满足状态，不是在那里等，而是将资源留出来，用来处理其他的RPC调用。</p><p><img src="https://static001.geekbang.org/resource/image/02/f5/0258775aac1126735504c9a6399745f5.jpg?wh=4045*2276" alt=""></p><p>从这个图可以看出，这个状态转换图还是很复杂的。</p><p>首先，进入起始状态，查看RPC的传输层队列中有没有空闲的位置，可以处理新的RPC任务。如果没有，说明太忙了，或直接结束或重试。如果申请成功，就可以分配内存，获取服务的端口号，然后连接服务器。</p><p>连接的过程要有一段时间，因而要等待连接的结果，会有连接失败，或直接结束或重试。如果连接成功，则开始发送RPC请求，然后等待获取RPC结果，这个过程也需要一定的时间；如果发送出错，可以重新发送；如果连接断了，可以重新连接；如果超时，可以重新传输；如果获取到结果，就可以解码，正常结束。</p><p>这里处理了连接失败、重试、发送失败、超时、重试等场景。不是大牛真写不出来，因而实现一个RPC的框架，其实很有难度。</p><h2>服务发现问题</h2><p>传输问题解决了，我们还遗留一个问题，就是问题四“如何找到RPC服务端的那个随机端口”。这个问题我们称为服务发现问题。在ONC RPC中，服务发现是通过portmapper实现的。</p><p><img src="https://static001.geekbang.org/resource/image/2a/7c/2aff190d1f878749d2a5bd73228ca37c.jpg?wh=2382*2074" alt=""></p><p>portmapper会启动在一个众所周知的端口上，RPC程序由于是用户自己写的，会监听在一个随机端口上，但是RPC程序启动的时候，会向portmapper注册。客户端要访问RPC服务端这个程序的时候，首先查询portmapper，获取RPC服务端程序的随机端口，然后向这个随机端口建立连接，开始RPC调用。从图中可以看出，mount命令的RPC调用，就是这样实现的。</p><h2>小结</h2><p>好了，这一节就到这里，我们来总结一下。</p><ul>
<li>
<p>远程调用看起来用Socket编程就可以了，其实是很复杂的，要解决协议约定问题、传输问题和服务发现问题。</p>
</li>
<li>
<p>大牛Bruce Jay Nelson的论文、早期ONC RPC框架，以及NFS的实现，给出了解决这三大问题的示范性实现，也即协议约定要公用协议描述文件，并通过这个文件生成Stub程序；RPC的传输一般需要一个状态机，需要另外一个进程专门做服务发现。</p>
</li>
</ul><p>最后，给你留两个思考题。</p><ol>
<li>
<p>在这篇文章中，mount的过程是通过系统调用，最终调用到RPC层。一旦mount完毕之后，客户端就像写入本地文件一样写入NFS了，这个过程是如何触发RPC层的呢？</p>
</li>
<li>
<p>ONC RPC是早期的RPC框架，你觉得它有哪些问题呢？</p>
</li>
</ol><p>我们的专栏更新到第32讲，不知你掌握得如何？每节课后我留的思考题，你都有没有认真思考，并在留言区写下答案呢？我会从<strong>已发布的文章中选出一批认真留言的同学</strong>，赠送<strong>学习奖励礼券</strong>和我整理的<strong>独家网络协议知识图谱</strong>。</p><p>欢迎你留言和我讨论。趣谈网络协议，我们下期见！</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/a8/1b/ced1d171.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>空档滑行</span>
  </div>
  <div class="_2_QraFYR_0">1.rpc调用是在进行读写操作时，调用的操作系统的读写接口，nfs对接口做了实现，实现的代码里封装了rpc<br>2.需要调用双方有接口描述文件，有文件就需要双方要做信息交换，所以客户端和服务端不是完全透明的</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-07-30 20:16:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/01/38/5daf2cfb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>吴军旗^_^</span>
  </div>
  <div class="_2_QraFYR_0">越往后人的留言越少， 看来成为大牛的路上会越来越孤单</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 坚持到最后，你就是大牛</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-10 21:23:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/43/62/cd7d8b3b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>叹息无门</span>
  </div>
  <div class="_2_QraFYR_0">1.应用在读写文件时，会创建文件描述符，NFS Client会将文件描述符的操作代理成RPC请求。<br>2.XDR有严格的格式限制，两端必须完全匹配，无法支持灵活数据格式的传递。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-07-30 19:16:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/7c/16/4d1e5cc1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>mgxian</span>
  </div>
  <div class="_2_QraFYR_0">1.nfs挂载的时候指定了文件系统类型 当应用对文件进行read write等操作时 会调用系统底层的vfs文件系统相关函数， nfs 实现了 vfs规定的 接口函数，调用vfs相关函数时 vfs其实会调用nfs的实现 实现访问远程文件系统<br><br>2.不支持多语言 </div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-07-30 08:18:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/a2/12/c429f550.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>何重阳</span>
  </div>
  <div class="_2_QraFYR_0">能不能收我为徒😂</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-03 16:09:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/82/3d/356fc3d6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>忆水寒</span>
  </div>
  <div class="_2_QraFYR_0">刘老师，我们目前的分布式系统采用以下方式。我们实现了一种中间件，每个进程（客户端）要与其他进程通信，就要到中间件注册（注册了自己进程的一个ID，任务名称，还有一个消息队列），然后将消息用google的protobuf封装进行传输（因为这种序列化的效率高）。在其他进程中接收到消息，会解析消息id，然后根据定义好的格式去取内容。这样也算RPC调用吧？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-07-30 13:08:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/c1/60/fc3689d0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小谢同学</span>
  </div>
  <div class="_2_QraFYR_0">作为一名不会coding的从业者，想问刘老师几个基础问题，首先是文中提到的onc rpc框架，就是包含了stub（编解码）+传输（类库）+服务发现的一套东西？那么目前主流的rpc框架，比如dubbo也是实现了这些功能的集大成者？另外一个问题就是，thrift 和protobuf 我理解只是实现了rpc编解码环节的工作，也就是所说的序列化与反序列化，对么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，rpc的几个时代，就是这样演进过来的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-07-31 18:06:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/5b/bc/1b6e3848.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>朽木自雕</span>
  </div>
  <div class="_2_QraFYR_0">刘老师，我都认真的看了就是有的看不太懂，但是我真的好期待您的那个知识图谱，我觉得这个有助于对知识的加深理解，因为我认为这种图谱被我所喜欢的原因是它属于空间的结构，我自己这么认为的。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-07-30 09:03:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/pTD8nS0SsORKiaRD3wB0NK9Bpd0wFnPWtYLPfBRBhvZ68iaJErMlM2NNSeEibwQfY7GReILSIYZXfT9o8iaicibcyw3g/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>雷刚</span>
  </div>
  <div class="_2_QraFYR_0">1. nfs 和 hdfs 一样都是分布式文件系统。当往往 nfs 目录中读写时，就会和这个目录对应的远程服务器建立 RPC 通道，我们看这是像往本地文件读写，实际是通过 RPC 往远程服务器读写。<br>2. 现代的 RPC 框架越来越像一个生态（如 Dubbo），不仅仅是一个 RPC 框架，更是一整套微服务的解决方案。如分布式配置（Nacos）、服务注册与发现（Nacos）、服务调用、负载均衡、限流与熔断、网关、分布式消息中间件等。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-10 16:27:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/f6/32/358f9411.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>梦想启动的蜗牛</span>
  </div>
  <div class="_2_QraFYR_0">最低位放在最后不是大端模式嘛？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-08-09 10:02:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/a5/98/a65ff31a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>djfhchdh</span>
  </div>
  <div class="_2_QraFYR_0">问题2：如果约定的协议内容发生变化，比如，增加参数、增加方法等，都需要重新生成stub程序，并重启rpc的客户端和服务端，比较麻烦</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-25 14:13:54</div>
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
  <div class="_2_QraFYR_0">1. mounted NFS会创建RPC的client端调用的映射，当NFS有写入时，就根据这个映射将结果发给client的stub。<br>2. 每次添加新的RPC方法，会需要重新部署（启动）client和server。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-27 17:02:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/12/ce/a8c8b5e8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Jason</span>
  </div>
  <div class="_2_QraFYR_0">这篇我看懂了哈哈。工作中一涉及到rpc，我简直是thrift的铁杆粉丝，Google的protobuf也不错，但其中的原理的我并没深究。通过这篇，我学到了rpc的架构原理，赞。至于nfs，其实工作中也有用过，但仅仅是用而已，没有深究其中的奥妙，期待超哥下篇的解答。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-07-31 08:03:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/dc/b8/31c7e110.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>LVM_23</span>
  </div>
  <div class="_2_QraFYR_0">终于来了些看得懂和日常已经使用的知识了，手动狗头</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-03 09:40:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLvdWoCic6ItzibF8ia8vrUTRuyj6AT3tg5f4QicIK0jTIFheJ6274ZkibuRLFP1NXG3jibv5TiaSKNoJpLw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_37984c</span>
  </div>
  <div class="_2_QraFYR_0">老师GABAGE_ARGS 是写错了吗<br>GARBAGE_ARGS?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-24 11:23:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/78/ac/e5e6e7f3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>古夜</span>
  </div>
  <div class="_2_QraFYR_0">看完知道一点zookeeper这个注册中心为啥要注册了，懵懵懂懂的也算是知道一点了😂</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-25 12:39:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/1a/e5/6899701e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>favorlm</span>
  </div>
  <div class="_2_QraFYR_0">rpc，现在用框架已经简单了很多</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-07-30 08:19:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/fd/c9/9453237e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>T</span>
  </div>
  <div class="_2_QraFYR_0">ONC RPC 是早期的 RPC 框架，你觉得它有哪些问题呢？<br>---- 1.现在的rpc好像不需要端口注册了，约定俗成了吧，服务端告诉你我是哪个端口，你用就好了。<br>         2. 我好像没看到线程模型，就是socket accept只需要一个线程，然后处理的可以用其他线程这种模型，更不用说epoll这些了。<br><br>不知道我说的对不对，哈哈</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-11 15:17:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/6f/70/db94799a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Wienbc</span>
  </div>
  <div class="_2_QraFYR_0">原来这就是RPC，终于get了。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-21 17:30:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/7c/59/dc9bbb21.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Join</span>
  </div>
  <div class="_2_QraFYR_0">RPC看得比较爽了，学习了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-11 09:21:08</div>
  </div>
</div>
</div>
</li>
</ul>