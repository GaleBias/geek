<audio title="07 _ NIO：手撸一个简易的主从多Reactor线程模型" src="https://static001.geekbang.org/resource/audio/81/93/815c22ded455b0e387bb13f096c4e893.mp3" controls="controls"></audio> 
<p>你好，我是丁威。</p><p>中间件是互联网发展的产物，而互联网有一个非常显著的特点：集群部署、分布式部署。当越来越多的服务节点分布在不同的机器上，高效地进行网络传输就变得更加迫切了。在这之后，一大批网络编程类库如雨后春笋般出现，经过不断的实践表明，Netty框架几乎成为了网络编程领域的不二之选。</p><p>接下来的两节课，我们会通过对NIO与Netty的详细解读，让你对网络编程有一个更直观的认识。</p><h2>NIO和BIO模型的工作机制</h2><p>NIO是什么呢？简单来说，NIO就是一种新型IO编程模式，它的特点是<strong>同步</strong>、<strong>非阻塞</strong>。</p><p>很多资料将NIO中的“N”翻译为New，即新型IO模型，既然有新型的IO模式，那当然也存在中老型的IO模型，这就是BIO，同步阻塞IO模型。</p><p>定义往往是枯燥的，我们结合实际场景看一下BIO和NIO两种IO通讯模式的工作机制，更直观地感受一下它们的差异。</p><p>MySQL的客户端(mysql-connector-java)采用的就是BIO模式，它的工作机制如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/ef/c7/effdb1302033dc76c329315cece78fc7.jpg?wh=1920x1195" alt="图片"></p><p>我们模拟场景，向MySQL服务端查询表中的数据，这时会经过四个步骤。</p><p>第一步，应用程序拼接SQL，然后mysql-connector-java会将SQL语句按照MySQL通讯协议编码成二进制，通过网络API将数据写入到网络中进行传输。底层最终是使用Socket的OutputStream的write与flush这两个方法实现的。</p><!-- [[[read_end]]] --><p>第二步，调用完write方法后，再调用Socket的InputStream的read方法，读取服务端返回数据，此时会阻塞等待。</p><p>第三步，服务端在收到请求后会解析请求，从请求中提取出对应的SQL语句，然后按照SQL抽取数据。服务端在处理这些业务逻辑时，客户端阻塞，不能做其他事情，我把这个阶段称之为<strong>等待数据阶段</strong>。</p><p>第四步，服务端执行完指定逻辑，抽取到合适的数据后，会调用Socket的OutputStream的write将响应结果通过网络传递到客户端。此时，客户端用read方法从网卡中把数据读取到应用程序的内存中，此阶段我称之为<strong>数据传输阶段</strong>。</p><p>BIO的IO模型在等待数据阶段、数据传输阶段都会阻塞。其实，“IO模型”的名称基本就是这两个阶段的特质决定的。</p><p><strong>在等待数据阶段，</strong>如果发起网络调用后，在服务端数据没有准备好的情况下客户端会阻塞，我们称为阻塞IO；如果数据没有准备好，但网络调用会立即返回，我们称之为非阻塞IO。</p><p><strong>在数据传输阶段，</strong>如果发起网络调用的线程还可以做其他事情，我们称之为异步，否则称之为同步。</p><p>这样看来，BIO的完整名称叫做“同步阻塞IO”也就不足为奇了。</p><p>从JDK1.4开始，Java又引入了另外一种IO模型：NIO。</p><p>虽然MySQL客户端主要使用的是BIO模型，但是我们可以演示一下MySQL Client采用NIO与MySQL服务端通信的样子：</p><p><img src="https://static001.geekbang.org/resource/image/0e/9f/0ebecd5267cd3757fb776eede5b8059f.jpg?wh=1920x1188" alt="图片"></p><p>NIO与BIO的不同点在于，在调用read方法时，如果服务端没有返回数据，该方法不会阻塞当前调用线程，read方法的返回值会为本次网络调用实际读取到的字节数量。也就是说，客户端调用一次read方法，如果本次没有读取到数据，线程可以继续处理其他事情，然后在需要数据的时候再次调用，但是在数据返回的过程中同样会阻塞线程。这也是NIO全名的由来：同步非阻塞IO。</p><p>NIO提供了在数据等待阶段的灵活性，但如果需要客户端反复调用读相关的API进行测试，编程体验也极不友好，为了改进NIO网络模型的缺陷，又引入了“事件就绪选择机制”。</p><p>事件就绪选择机制指的是，应用程序只需要在通道（网络连接）上注册感兴趣的事件（如网络读事件），客户端向服务端发送请求后，无须立即调用read方法去尝试读取响应结果，而是等服务端将准备好的数据通过网络传输到客户端的网卡。这时，操作系统会通知客户端“数据已到达”，此时客户端再调用读取API，从中读取响应结果。其实<strong>我们现在说NIO，说的就是“NIO + 事件就绪选择”的合体</strong>。</p><h2>NIO和BIO模型的使用场景</h2><p>那BIO与NIO相比，有什么优劣势呢？它们对应的使用场景是什么？为了直观地展示两种编程模型的优缺点，我们用网络游戏这个场景来举例。</p><p>一个简易的网络游戏分为服务端与客户端（玩家）两个端口，我们一起来思考一下，如果游戏服务端分别使用BIO技术和NIO技术进行架构设计，结果会是怎样的。</p><p>BIO领域一种经典的设计范式是<strong>每个请求对应一个线程。</strong>我们就用这种思想设计一下游戏的服务端，设计图如下：</p><p><img src="https://static001.geekbang.org/resource/image/b3/b2/b39791473269ff1065d3f9a3c6c330b2.jpg?wh=1920x950" alt="图片"></p><p>游戏服务端后端的设计思想是：采用长连接模式。每当一个客户端上线，服务端就会为请求创建一个线程，在独立的线程中和客户端进行网络读写，一旦客户端下线，就会关闭对应的线程。</p><p>但是一台服务器能创建的线程个数是有限的，所以基于BIO模式构建的优秀服务端一个非常明显的弊端：在线用户数越多，需要创建的线程数就越多，支撑的并发在线用户数量受到明显制约。更加严重的问题是，服务端与其中某些客户端并不是一直在通信，大量线程的网络连接处于阻塞状态，线程资源无法得到有效利用。</p><p>为了防止因为线程急剧膨胀、线程资源耗尽影响到服务端的设计，这时候我们通常会引入线程池。因为引入线程池就相当于是在限流，超过线程池规定的线程数量，服务器就会拒绝连接。</p><p>对于需要支持大量在线并发用户（连接）的服务器来说，BIO的网络IO模型绝对不是一个好的选择。</p><p>我们再来看下NIO模式。基于NIO模式设计的游戏服务端模型如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/3e/a3/3ee3a797a3cc288e2426345331a02ba3.jpg?wh=1920x1381" alt="图片"></p><p>基于NIO，在业界有一种标准的线程模型Reactor，在这节课的后半部分我们还会详细介绍，这里我们先说明一下NIO的优势。</p><p>首先，服务端会创建一个线程池专门处理网络读写，我们称之为IO线程池。IO线程池内会内置NIO的事件选择器。当游戏服务端监听到一个客户端时，会从IO线程池中根据负载均衡算法选择一个IO线程，将其注册到事件选择器中。</p><p>事件选择器会定时进行事件轮询，挑选出数据进行传输（读取或写入），执行事件选择，然后在IO线程中按连接分别读取数据。在将请求解码后，丢到业务线程中执行对应的业务逻辑，它的主要功能是分担IO线程的压力，做到尽量不阻塞IO线程。</p><p>使用NIO可以做到用少量线程来服务大量连接，哪怕客户端连接数增长，也不会造成服务端线程膨胀。这个优势的关键点在于，基于事件选择机制，IO线程都在进行有效的读写，而不像BIO那样，在没有数据传输时还得占用线程资源。</p><p><strong>也正是因此，NIO非常适合需要同时支持大量客户端在线的场景。在NIO模型下，单个请求的数据包建议不要太大</strong>。</p><p>值得注意的是，一个IO线程在一次事件就绪选择可能会有多个网络连接具备了读或写的准备，但此时对这些网络通道是串行执行的，所以如果每一个网络通道需要读或写的数据比较大，这就必然导致其他连接的延时。</p><p>既然NIO这么优秀，那为什么MySQL数据访问客户端还是采用BIO模式呢？为啥不改造成NIO呢？</p><p>其实在进行技术选型时，并不是越新的技术就越好，我们还是要结合具体问题具体分析。</p><p>我们再回过头来看MySQL客户端的场景。目前在应用层面，我们会为每一个应用配置一个数据库连接池。当业务线程需要进行数据库操作时，它会尝试从数据库连接池获取一个数据库连接（底层是一条TCP连接，负责与服务端进行网络的读与写），然后使用这条连接发送SQL语句并获取SQL结果。任务结束之后，业务线程会把数据库连接归还给连接池，从而实现数据库连接的复用。</p><p>与此同时，我们为了保证数据库服务端的可用性，通常需要强制限制客户端能使用的连接数量。这就注定了数据库客户端没有需要支持大量连接的诉求，在这个场景下，客户端使用阻塞型IO对保护数据库服务端更有优势。</p><p><img src="https://static001.geekbang.org/resource/image/55/29/55dc0801a13342c4600c14cfeb7d1c29.jpg?wh=1920x738" alt=""></p><p>简单说明一下。假设业务代码存在缺陷，导致需要执行一条SQL语句来获取大量数据。这时，我们要尝试从数据库连接池中获取连接，并通过这个连接向MySQL服务端发送SQL语句。由于这条SQL语句的执行性能很差，这条连接在客户端一直被阻塞，无法继续发送更多的SQL。另外如果数据库连接池中没有空闲连接，再尝试获取连接时还需要等待连接被释放，服务器缓慢的执行速度确保了客户端不能持续发送新的请求，对保护数据库服务器大有裨益。</p><p>这种情况下如果使用NIO模型，客户端会无节制地用一条连接发送大量请求，导致服务端出现完全不可用的情况。</p><p>总结一下就是，NIO模型更适合需要大量在线活跃连接的场景，常见于服务端；BIO模型则适合只需要支持少量连接的场景，常常用于客户端，这也是MySQL数据访问客户端会在网络IO模型方面使用BIO的原因。</p><h2>Reactor线程模型</h2><p>学习NIO的理论知识非常枯燥，而且很难做到透彻地理解，我们需要一个实例来深入进去。结合我的学习经验，我觉得学习Reactor经典线程模型，尝试编写一个Reactor线程模型对提升NIO的理解非常有帮助。</p><p>为什么这么说呢？因为在编写网络通信相关的功能模块时，建立一套线程模型是非常重要的一环，经过各位前辈不断的实践，Reactor线程模型已成为NIO领域的事实标准，无论是网络编程类库NIO，还是Kafka、Dubbo等主流中间件的底层网络模型都是直接或间接受到了Reactor模型的影响。</p><p>那什么是Reactor线程模型？怎么使用NIO来实现Reactor模型？这两个问题，就是我们这节课后半部分的重点。</p><h3>什么是Reactor线程模型？</h3><p>Reactor主从多Reactor模型的架构设计如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/54/bd/54782abb1c48e90807d6338f21dfb4bd.jpg?wh=1920x1209" alt="图片"></p><p>说明一下各个角色的职责。</p><ul>
<li>Acceptor：请求接收者，作用是在特定端口建立监听。</li>
<li>Main Reactor Thread Pool：主Reactor模型，主要负责处理OP_ACCEPT事件（创建连接），通常一个监听端口使用一个线程。在具体实践时，如果创建连接需要进行授权校验（Auth）等处理逻辑，也可以直接让Main Reactor中的线程负责。</li>
<li>NIO Thread Group（ IO 线程组）：在Reactor模型中也叫做从Reactor，主要负责网络的读与写。当Main Reactor Thread 线程收到一个新的客户端连接时，它会使用负载均衡算法从NIO Thread Group中选择一个线程，将OP_READ、OP_WRITE事件注册在NIO Thread的事件选择器中。接下来这个连接所有的网络读与写都会在被选择的这条线程中执行。</li>
<li>NIO Thread：IO线程。负责处理网络读写与解码。IO线程会从网络中读取到二进制流，并从二进制流中解码出一个个完整的请求。</li>
<li>业务线程池：通常IO线程解码出的请求将转发到业务线程池中运行，业务线程计算出对应结果后，再通过IO线程发送到客户端。</li>
</ul><p>我们再通过一个网络通信图进一步理解Reactor线程模型。</p><p><img src="https://static001.geekbang.org/resource/image/60/f9/6004a2b6759f46a249ce1bc95ce0e7f9.jpg?wh=1920x1081" alt="图片"></p><p>网络通信的交互过程通常包括下面六个步骤。</p><ol>
<li>启动服务端，并在特定端口上监听，例如，web 应用默认在80端口监听。</li>
<li>客户端发起TCP的三次握手，与服务端建立连接。这里以 NIO 为例，成功建立连接后会创建NioSocketChannel对象。</li>
<li>服务端通过 NioSocketChannel 从网卡中读取数据。</li>
<li>服务端根据通信协议从二进制流中解码出一个个请求。</li>
<li>根据请求执行对应的业务操作，例如，Dubbo 服务端接受了请求，并根据请求查询用户ID为1的用户信息。</li>
<li>将业务执行结果返回到客户端（通常涉及到协议编码、压缩等）。</li>
</ol><p>线程模型需要解决的问题包括：连接监听、网络读写、编码、解码、业务执行等，那如何运用多线程编程优化上面的步骤从而提升性能呢？</p><p>主从多Reactor模型是业内非常经典的，专门解决网络编程中各个环节问题的线程模型。各个线程通常的职责分工如下。</p><ol>
<li>Main Reactor 线程池，主要负责连接建立（OP_ACCEPT），即创建NioSocketChannel后，将其转发给SubReactor。</li>
<li>SubReactor 线程池，主要负责网络的读写（从网络中读字节流、将字节流发送到网络中），即监听OP_READ、OP_WRITE，并且同一个通道会绑定一个SubReactor线程。</li>
</ol><p>编码、解码和业务执行则具体情况具体分析。通常，编码、解码会放在IO线程中执行，而业务逻辑的执行会采用额外的线程池。但这不是绝对的，一个好的框架通常会使用参数来进行定制化选择，例如 ping、pong 这种心跳包，直接在 IO 线程中执行，无须再转发到业务线程池，避免线程切换开销。</p><h3>怎么用NIO实现Reactor模型？</h3><p>理解了Reactor线程模型的内涵，接下来就到了实现这一步了。</p><p>我建议你在学习这部分内容时，同步阅读一下<a href="https://www.aliyundrive.com/s/9A2iNmtLMpE">《Java NIO》</a>这本电子书的前四章。这本书详细讲解了NIO的基础知识，是我学习Netty的老师，相信也会给你一些帮助。</p><p>我们先来看一下Reactor模型的时序图，从全局把握整体脉络：</p><p><img src="https://static001.geekbang.org/resource/image/4f/c5/4f4d7df4cd4b21e6fd1b03ec44ce4ac5.jpg?wh=1920x861" alt="图片"></p><p>这里核心的流程有三个。</p><ul>
<li>服务端启动，会创建MainReactor线程池，在MainReactor中创建NIO事件选择器，并注册OP_ACCEPT事件，然后在指定端口监听客户端的连接请求。</li>
<li>客户端向服务端建立连接，服务端OP_ACCEPT对应的事件处理器被执行，创建NioSocketChannel对象，并按照负载均衡机制将其转发到SubReactor线程池中的某一个线程上，注册OP_READ事件。</li>
<li>客户端向服务端发送具体请求，服务端OP_READ对应的事件处理器被执行，它会从网络中读取数据，然后解码、转发到业务线程池执行具体的业务逻辑，最后将返回结果返回到客户端。</li>
</ul><p>我们解读下核心类的核心代码。</p><p>NioServer的代码如下：</p><pre><code class="language-plain">private static class Acceptor implements Runnable {
 &nbsp; &nbsp; &nbsp; &nbsp;// main Reactor 线程池，用于处理客户端的连接请求
 &nbsp; &nbsp; &nbsp; &nbsp;private static ExecutorService mainReactor = Executors.newSingleThreadExecutor(new ThreadFactory() {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;private AtomicInteger num = new AtomicInteger(0);
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;@Override
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;public Thread newThread(Runnable r) {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;Thread t = new Thread(r);
                // 为线程池中的名称进行命名，方便分析线程栈
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;t.setName("main-reactor-" + num.incrementAndGet());
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;return t;
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;  }
 &nbsp; &nbsp; &nbsp;  });
 &nbsp; &nbsp; &nbsp; &nbsp;public void run() {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;// NIO中服务端对应的Channel
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;ServerSocketChannel ssc = null;
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;try {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;// 通过静态方法创建一个ServerSocketChannel对象
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;ssc = ServerSocketChannel.open();
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;//设置为非阻塞模式
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;ssc.configureBlocking(false);
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;//绑定端口
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;ssc.bind(new InetSocketAddress(SERVER_PORT));
​
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;//转发到 MainReactor反应堆
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;dispatch(ssc);
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;System.out.println("服务端成功启动。。。。。。");
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;  } catch (IOException e) {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;e.printStackTrace();
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;  }
 &nbsp; &nbsp; &nbsp;  }
 &nbsp; &nbsp; &nbsp; &nbsp;private void dispatch(ServerSocketChannel ssc) {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;mainReactor.submit(new MainReactor(ssc));
 &nbsp; &nbsp; &nbsp;  }
 &nbsp;  }
</code></pre><p>启动服务端会创建一个Acceptor线程，它的职责就是绑定端口，创建ServerSocketChannel，然后交给MainReactor去处理接收连接的逻辑。<br>
MainReactor的具体实现如下：</p><pre><code class="language-plain">public class MainReactor implements Runnable{
 &nbsp; &nbsp;// NIO 事件选择器
 &nbsp; &nbsp;private Selector selector;
 &nbsp; &nbsp;// 子ReactorThreadGroup 即IO线程池
 &nbsp; &nbsp;private SubReactorThreadGroup subReactorThreadGroup;
 &nbsp; &nbsp;// IO线程池默认线程数量
 &nbsp; &nbsp;private static final int DEFAULT_IO_THREAD_COUNT = 4;
 &nbsp; &nbsp;// IO线程个数
 &nbsp; &nbsp;private int ioThreadCount = DEFAULT_IO_THREAD_COUNT;
 &nbsp;
 &nbsp; &nbsp;public MainReactor(ServerSocketChannel channel) {
 &nbsp; &nbsp; &nbsp; &nbsp;try {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;// 创建事件选择器
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;selector = Selector.open();
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;// 为通道注册OP_ACCEPT 事件，客户端发送数据后，服务端通过该事件进行数据的读取
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;channel.register(selector, SelectionKey.OP_ACCEPT);
 &nbsp; &nbsp; &nbsp;  } catch (IOException e) {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;e.printStackTrace();
 &nbsp; &nbsp; &nbsp;  }
        // IO线程池，里面包含负载均衡算法
 &nbsp; &nbsp; &nbsp; &nbsp;subReactorThreadGroup = new SubReactorThreadGroup(ioThreadCount);
 &nbsp;  }
 &nbsp; &nbsp;public void run() {
 &nbsp; &nbsp; &nbsp; &nbsp;System.out.println("MainReactor is running");
 &nbsp; &nbsp; &nbsp; &nbsp;while (!Thread.interrupted()) {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;Set&lt;SelectionKey&gt; ops = null;
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;try {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;// 进行事件选择
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;selector.select(1000);
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;// 经过事件选择后已经就绪的事件
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;ops = selector.selectedKeys(); 
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;  } catch (IOException e) {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;e.printStackTrace();
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;  }
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;// 处理相关事件
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;for (Iterator&lt;SelectionKey&gt; it = ops.iterator(); it.hasNext();) {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;SelectionKey key = it.next();
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;it.remove();
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;try {
                     // 如果有客户端尝试建立连接
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;if (key.isAcceptable()) {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;System.out.println("收到客户端的连接请求。。。");
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;//获取服务端的ServerSocketChannel对象， 这里其实，可以直接使用ssl这个变量
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;ServerSocketChannel serverSc = (ServerSocketChannel) key.channel();
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;// 调用ServerSocketChannel的accept方法，创建SocketChannel
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;SocketChannel clientChannel = serverSc.accept();
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;// 设置为非阻塞模式
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;clientChannel.configureBlocking(false);
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;// 转发到IO线程，由对应的IO线程去负责网络读写
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;subReactorThreadGroup.dispatch(clientChannel); // 转发该请求
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;  }
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;  } catch (Throwable e) {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;e.printStackTrace();
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;System.out.println("客户端主动断开连接。。。。。。。");
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;  }
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;  }
 &nbsp; &nbsp; &nbsp;  }
 &nbsp;  }
}
</code></pre><p>SubReactorThreadGroup内部包含一个SubReactorThread数组，并提供负载均衡机制，供MainReactor线程选择具体的SubReactorThread线程，具体代码如下：</p><pre><code class="language-plain">public class SubReactorThreadGroup {
 &nbsp; &nbsp;private static final AtomicInteger requestCounter = new AtomicInteger(); &nbsp;//请求计数器
 &nbsp; &nbsp;// 线程池IO线程的数量
    private final int ioThreadCount; &nbsp;
    // 业务线程池大小
 &nbsp; &nbsp;private final int businessTheadCout; 
 &nbsp; &nbsp;private static final int DEFAULT_NIO_THREAD_COUNT;
 &nbsp; &nbsp;// IO线程池数组
 &nbsp; &nbsp;private SubReactorThread[] ioThreads;
    //业务线程池
 &nbsp; &nbsp;private ExecutorService businessExecutePool; 
​
 &nbsp; &nbsp;static {
 &nbsp; &nbsp; &nbsp; &nbsp;DEFAULT_NIO_THREAD_COUNT = 4;
 &nbsp;  }
​
 &nbsp; &nbsp;public SubReactorThreadGroup() {
 &nbsp; &nbsp; &nbsp; &nbsp;this(DEFAULT_NIO_THREAD_COUNT);
 &nbsp;  }
 &nbsp; &nbsp;public SubReactorThreadGroup(int ioThreadCount) {
 &nbsp; &nbsp; &nbsp; &nbsp;if(ioThreadCount &lt; 1) {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;ioThreadCount = DEFAULT_NIO_THREAD_COUNT;
 &nbsp; &nbsp; &nbsp;  }
 &nbsp; &nbsp; &nbsp; &nbsp;//暂时固定为10
 &nbsp; &nbsp; &nbsp; &nbsp;businessTheadCout = 10;
        //初始化代码
 &nbsp; &nbsp; &nbsp; &nbsp;businessExecutePool = Executors.newFixedThreadPool(businessTheadCout, new ThreadFactory() {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;private AtomicInteger num = new AtomicInteger(0);
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;@Override
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;public Thread newThread(Runnable r) {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;Thread t = new Thread(r);
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;t.setName("bussiness-thread-" + num.incrementAndGet());
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;return t;
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;  }
 &nbsp; &nbsp; &nbsp;  });
 &nbsp; &nbsp; &nbsp; &nbsp;this.ioThreadCount = ioThreadCount;
 &nbsp; &nbsp; &nbsp; &nbsp;this.ioThreads = new SubReactorThread[ioThreadCount];
 &nbsp; &nbsp; &nbsp; &nbsp;for(int i = 0; i &lt; ioThreadCount; i ++ ) {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;this.ioThreads[i] = new SubReactorThread(businessExecutePool);
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;this.ioThreads[i].start(); //构造方法中启动线程，由于nioThreads不会对外暴露，故不会引起线程逃逸
 &nbsp; &nbsp; &nbsp;  }
 &nbsp; &nbsp; &nbsp; &nbsp;System.out.println("Nio 线程数量：" + ioThreadCount);
 &nbsp;  }
 &nbsp; &nbsp;public void dispatch(SocketChannel socketChannel) {
         //根据负载算法转发到具体IO线程
 &nbsp; &nbsp; &nbsp; &nbsp;if(socketChannel != null ) {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;next().register(new NioTask(socketChannel, SelectionKey.OP_READ));
 &nbsp; &nbsp; &nbsp;  }
 &nbsp;  }
 &nbsp; &nbsp;protected SubReactorThread next() {
 &nbsp; &nbsp; &nbsp; &nbsp;return this.ioThreads[ requestCounter.getAndIncrement() % &nbsp;ioThreadCount ];
 &nbsp;  }
</code></pre><p>SubReactorThread IO线程的具体实现如下：</p><pre><code class="language-plain">public class SubReactorThread extends Thread{
 &nbsp;// 事件选择器
 &nbsp; &nbsp;private Selector selector;
 &nbsp;//业务线程池
 &nbsp; &nbsp;private ExecutorService businessExecutorPool;
 &nbsp;//任务列表
 &nbsp; &nbsp;private List&lt;NioTask&gt; taskList = new ArrayList&lt;NioTask&gt;(512);
 &nbsp;// 锁
 &nbsp; &nbsp;private ReentrantLock taskMainLock = new ReentrantLock();
 &nbsp; &nbsp;/**
 &nbsp; &nbsp; * 业务线程池
 &nbsp; &nbsp; * @param businessExecutorPool
 &nbsp; &nbsp; */
 &nbsp; &nbsp;public SubReactorThread(ExecutorService businessExecutorPool) {
 &nbsp; &nbsp; &nbsp; &nbsp;try {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;this.businessExecutorPool = businessExecutorPool;
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;//创建事件选择器
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;this.selector = Selector.open();
 &nbsp; &nbsp; &nbsp;  } catch (IOException e) {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;// TODO Auto-generated catch block
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;e.printStackTrace();
 &nbsp; &nbsp; &nbsp;  }
 &nbsp;  }
 &nbsp; &nbsp;/**
 &nbsp; &nbsp; * 接受读写任务
 &nbsp; &nbsp; *
 &nbsp; &nbsp; */
 &nbsp; &nbsp;public void register(NioTask task) {
 &nbsp; &nbsp; &nbsp; &nbsp;if (task != null) {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;try {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;taskMainLock.lock();
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;taskList.add(task);
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;  } finally {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;taskMainLock.unlock();
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;  }
 &nbsp; &nbsp; &nbsp;  }
 &nbsp;  }
 &nbsp; &nbsp;/**
 &nbsp; &nbsp; * 此处的reqBuffer处于可写状态
 &nbsp; &nbsp; * @param sc
 &nbsp; &nbsp; * @param reqBuffer
 &nbsp; &nbsp; */
 &nbsp; &nbsp;private void dispatch(SocketChannel sc, ByteBuffer reqBuffer) {
 &nbsp; &nbsp; &nbsp; &nbsp;businessExecutorPool.submit( new Handler(sc, reqBuffer, this)  );
 &nbsp;  }
​
​
​
 &nbsp; &nbsp;public void run() {
 &nbsp; &nbsp; &nbsp; &nbsp;while (!Thread.interrupted()) {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;Set&lt;SelectionKey&gt; ops = null;
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;try {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;//执行事件选择
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;selector.select(1000);
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;// 获取已就绪的事件集合
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;ops = selector.selectedKeys();
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;  } catch (IOException e) {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;// TODO Auto-generated catch block
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;e.printStackTrace();
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;continue;
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;  }
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;// 处理相关事件
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;for (Iterator&lt;SelectionKey&gt; it = ops.iterator(); it.hasNext();) {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;SelectionKey key = it.next();
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;it.remove();
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;try {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; // 通道写事件就绪，说明可以继续往通道中写数据
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;if (key.isWritable()) { 
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;SocketChannel clientChannel = (SocketChannel) key.channel();
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;// 获取上次未写完的数据
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;ByteBuffer buf = (ByteBuffer) key.attachment();
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;// 将其写入到通道中。
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;// 这里实现比较粗糙，需要采用处理taskList类似的方式，因为此时通道缓冲区有可能已写满
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;clientChannel.write(buf);
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;System.out.println("服务端向客户端发送数据。。。");
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;// 重新注册读事件
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;clientChannel.register(selector, SelectionKey.OP_READ);
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;  } else if (key.isReadable()) { // 接受客户端请求
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;System.out.println("服务端接收客户端连接请求。。。");
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;SocketChannel clientChannel = (SocketChannel) key.channel();
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;ByteBuffer buf = ByteBuffer.allocate(1024);
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;System.out.println(buf.capacity());
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;/**
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; * 这里其实实现的非常不优雅，需要对读取处理办关闭，而且一次读取，并不一定能将一个请求读取
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; * 一个请求，也不要会刚好读取到一个完整对请求，
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; * 这里其实是需要编码，解码，也就是通信协议  @todo
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; * 这里如何做，大家可以思考一下，后面我们可以体验netty是否如何优雅处理的。
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; */
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;int rc = clientChannel.read(buf);//解析请求完毕
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;//转发请求到具体的业务线程；当然，这里其实可以向dubbo那样，支持转发策略，如果执行时间短，
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;//，比如没有数据库操作等，可以在io线程中执行。本实例，转发到业务线程池
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;dispatch(clientChannel, buf);
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;  }
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;  } catch (Throwable e) {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;e.printStackTrace();
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;System.out.println("客户端主动断开连接。。。。。。。");
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;  }
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;  }
​
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;// 处理完事件后，我们还需要处理其他任务，这些任务通常来自业务线程需要IO线程执行的任务
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;if (!taskList.isEmpty()) {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;try {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;taskMainLock.lock();
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;for (Iterator&lt;NioTask&gt; it = taskList
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;  .iterator(); it.hasNext();) {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;NioTask task = it.next();
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;try {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;SocketChannel sc = task.getSc();
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;if(task.getData() != null ) { // 写操作
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;ByteBuffer byteBuffer = (ByteBuffer)task.getData();
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;byteBuffer.flip();
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;// 如果调用通道的写函数，如果写入的字节数小于0，并且待写入还有剩余空间，说明缓存区已满
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;// 需要注册写事件，等缓存区空闲后继续写入
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;int wc = sc.write(byteBuffer);
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;System.out.println("服务端向客户端发送数据。。。");
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;if(wc &lt; 1 &amp;&amp; byteBuffer.hasRemaining()) { // 说明写缓存区已满，需要注册写事件
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;sc.register(selector, task.getOp(), task.getData());
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;continue;
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;  } 
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;byteBuffer = null;//释放内存
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;  } else {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;sc.register(selector, task.getOp());
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;  }
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;  } catch (Throwable e) {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;e.printStackTrace();// ignore
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;  }
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;it.remove();
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;  }
​
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;  } finally {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;taskMainLock.unlock();
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;  }
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;  }
 &nbsp; &nbsp; &nbsp;  }
 &nbsp;  }
}
</code></pre><p>IO线程负责从网络中读取二进制并将其解码成具体请求，然后转发到业务线程池执行。</p><p>接下来，业务线程池会执行业务代码并将响应结果通过IO线程写入到网络中，我们对业务进行简单的模拟：</p><pre><code class="language-plain">public class Handler implements Runnable{
​
 &nbsp;// 模拟业务处理
 &nbsp; &nbsp;private static final byte[] b = "hello,服务器收到了你的信息。".getBytes(); // 服务端给客户端的响应
 &nbsp; &nbsp;// 网络通道
 &nbsp; &nbsp;private SocketChannel sc;
 &nbsp; &nbsp;// 请求报文
 &nbsp; &nbsp;private ByteBuffer reqBuffer;
 &nbsp; &nbsp;// IO线程
 &nbsp; &nbsp;private SubReactorThread parent;
​
 &nbsp; &nbsp;public Handler(SocketChannel sc, ByteBuffer reqBuffer,
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; SubReactorThread parent) {
 &nbsp; &nbsp; &nbsp; &nbsp;super();
 &nbsp; &nbsp; &nbsp; &nbsp;this.sc = sc;
 &nbsp; &nbsp; &nbsp; &nbsp;this.reqBuffer = reqBuffer;
 &nbsp; &nbsp; &nbsp; &nbsp;this.parent = parent;
 &nbsp;  }
​
 &nbsp; &nbsp;public void run() {
 &nbsp; &nbsp; &nbsp; &nbsp;System.out.println("业务在handler中开始执行。。。");
 &nbsp; &nbsp; &nbsp; &nbsp;// TODO Auto-generated method stub
 &nbsp; &nbsp; &nbsp; &nbsp;//业务处理
 &nbsp; &nbsp; &nbsp; &nbsp;reqBuffer.put(b);
 &nbsp; &nbsp; &nbsp;// 业务处理完成后，通过向IO线程提交任务
 &nbsp; &nbsp; &nbsp; &nbsp;parent.register(new NioTask(sc, SelectionKey.OP_WRITE, reqBuffer));
 &nbsp; &nbsp; &nbsp; &nbsp;System.out.println("业务在handler中执行结束。。。");
 &nbsp;  }
}
</code></pre><p>我们再来看一下客户端创建连接的代码：</p><pre><code class="language-plain">public class NioClient {
 &nbsp; &nbsp;public static void main(String[] args) {
 &nbsp; &nbsp; &nbsp; &nbsp;// socket
 &nbsp; &nbsp; &nbsp; &nbsp;SocketChannel clientClient;
 &nbsp; &nbsp; &nbsp; &nbsp;// 事件选择器
 &nbsp; &nbsp; &nbsp; &nbsp;Selector selector = null;
 &nbsp; &nbsp; &nbsp; &nbsp;try {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;// 创建网络通道
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;clientClient = SocketChannel.open();
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;// 设置为非阻塞模型
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;clientClient.configureBlocking(false);
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;selector = Selector.open();
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;// 注册连接成功事件，在与服务端通过tcp三次握手建立连接后可以收到该事件
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;clientClient.register(selector, SelectionKey.OP_CONNECT);
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;//建立连接，该方法会立即返回
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;clientClient.connect(new InetSocketAddress("127.0.0.1",9080));
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;Set&lt;SelectionKey&gt; ops = null;
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;while(true) {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;try {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; // 执行事件选择
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;selector.select();
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;ops = selector.selectedKeys();
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;for (Iterator&lt;SelectionKey&gt; it = ops.iterator(); it.hasNext();) {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;SelectionKey key = it.next();
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;it.remove();
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;if(key.isConnectable()) &nbsp;//连接事件
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;System.out.println("client connect");
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;SocketChannel sc =  (SocketChannel) key.channel();
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;// 判断此通道上是否正在进行连接操作。
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;// 完成套接字通道的连接过程。
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;if (sc.isConnectionPending()) {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;sc.finishConnect();
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;System.out.println("完成连接!");
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;// 完成连接后，向服务端发送请求包
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;ByteBuffer buffer = ByteBuffer.allocate(1024);
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;buffer.put("Hello,Server".getBytes());
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;buffer.flip();
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;sc.write(buffer);
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;  }
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;// 注册读事件，等待服务端响应包到达
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;sc.register(selector, SelectionKey.OP_READ);
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;  } else if(key.isWritable()) {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;System.out.println("客户端写");
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;SocketChannel sc = (SocketChannel)key.channel();
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;//这里是NIO ByteBuffer的基本API
                            ByteBuffer buffer = ByteBuffer.allocate(1024);
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;buffer.put("hello server.".getBytes());
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;buffer.flip();
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;sc.write(buffer);
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;  } else if(key.isReadable()) {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;System.out.println("客户端收到服务器的响应....");
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;SocketChannel sc = (SocketChannel)key.channel();
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;ByteBuffer buffer = ByteBuffer.allocate(1024);
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;int count = sc.read(buffer);
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;if(count &gt; 0 ) {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;buffer.flip();
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;byte[] response = new byte[buffer.remaining()];
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;buffer.get(response);
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;System.out.println(new String(response));
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;  }
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;// 再次发送消息，重复输出
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;buffer = ByteBuffer.allocate(1024);
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;buffer.put("hello server.".getBytes());
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;buffer.flip();
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;sc.write(buffer);
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;  }
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;  }
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;  } catch(Throwable e) {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;e.printStackTrace();
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;  }
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;  }
 &nbsp; &nbsp; &nbsp;  } catch (IOException e) {
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;// TODO Auto-generated catch block
 &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;e.printStackTrace();
 &nbsp; &nbsp; &nbsp;  }
 &nbsp;  }
}
</code></pre><p>这样，一个Reactor模型就搭建好了。如果你想完整地学习这个Reactor模型的详细代码，可以到<a href="https://github.com/dingwpmz/netty-learning">我的GitHub</a>上查看。</p><h2><strong>总结</strong></h2><p>好了，这节课就讲到这里。</p><p>这节课，我们先结合场景介绍了BIO与NIO两种网络编程模型和它们的优缺点。</p><p>根据等待数据阶段和数据传输阶段这两个阶段的特质，我们可以得到BIO的全称同步阻塞IO，还有NIO的全称同步非阻塞IO。<strong>NIO模型更适合需要大量在线活跃连接的场景，常见于服务端；BIO模型则适合只需要支持少量连接的场景。</strong></p><p>我们还了解了一个业内非常经典的线程模型：主从多Reactor模型。它的核心设计理念是让线程分工明确，相互协作。Main Reactor 线程池主要负责连接建立，SubReactor 线程池主要负责网络的读写，而编码、解码和业务执行则需要具体情况具体分析。</p><p>最后，我还带你使用NIO技术实现了主从多Reactor模型，给你推荐了一本学习NIO必备的电子书<a href="https://www.aliyundrive.com/s/9A2iNmtLMpE">《Java NIO》</a>，这本书非常详细介绍了NIO的三大金刚：缓存、通道和选择器的各类基础知识。我建议你在阅读完本电子书后，再来反复看看这个Reactor示例，相信可以在你进修NIO的基础上助你一臂之力。</p><h2><strong>课后题</strong></h2><p>学完这节课，我也给你出两道课后题。</p><ol>
<li>为什么NIO不适合请求体很大的场景？</li>
<li>请你详细阅读《Java NIO》这本书中Reactor模型的示例子代码，尝试实现一个简易的RPC Request-Response模型。<br>
例如，模拟Dubbo服务调用需要传入基本的参数：包名、方法名，参数。客户端发送这些数据后，服务端根据接收的数据，在服务端要正确打印包名、方法名、参数，并向客户端返回 “hello,收到请求” + 包名 + 方法名。</li>
</ol><p>欢迎你在留言区留下你的思考结果，我们下节课见！</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/4b/d7/f46c6dfd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>William Ning</span>
  </div>
  <div class="_2_QraFYR_0">建议参考文档，对照着学习，思考<br>https:&#47;&#47;time.geekbang.org&#47;column&#47;article&#47;8697 -- 18 | 单服务器高性能模式：PPC与TPC<br>https:&#47;&#47;time.geekbang.org&#47;column&#47;article&#47;8805 -- 19 | 单服务器高性能模式：Reactor与Proactor<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-10 19:34:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/22/c6/36/70f2083c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>open！？</span>
  </div>
  <div class="_2_QraFYR_0">问题一 ：请求体很大 当前subreactor线程耗时会比较久 这时如果有别的channel来请求到当前subreactor线程会阻塞很久吧 不知道对不对</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 非常正确的，但Netty在这方面对这个问题做了处理。它的处理方法是就是控制一个通道在一次事件选择处理中，控制调用read方法的次数，默认为16次，也就是如果请求体太大，以至于16次读调用还无法读完，这个通道就不再处理，将时间给其他通道，等下一次读时间触发时，再继续读。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-22 08:28:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/84/55/34055533.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>孙哲</span>
  </div>
  <div class="_2_QraFYR_0">建议吧图文结合起来，比如那段文字是描述图中哪个过程</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-20 08:55:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/d2/03/aeaec5bf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>呢喃</span>
  </div>
  <div class="_2_QraFYR_0">写的非常好，对我很有帮助，感谢老师</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢认可，我建议看过后，自己后面再写一遍，然后再看netty是怎么写的，最后再来优化一遍，完成这个过程，我相信对nio的理解会更透彻。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-31 17:24:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/4b/d7/f46c6dfd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>William Ning</span>
  </div>
  <div class="_2_QraFYR_0">老师同学好，全是java代码，不过注释还是比较清晰。<br>但是关于NioClient 类中<br><br>&#47;&#47; 再次发送消息，重复输出<br>buffer = ByteBuffer.allocate(1024);<br>buffer.put(&quot;hello server.&quot;.getBytes());<br>buffer.flip();<br>sc.write(buffer);<br><br>没明白，为什么在读取数据的代码里，还要写入数据？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-11 12:34:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/4b/d7/f46c6dfd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>William Ning</span>
  </div>
  <div class="_2_QraFYR_0">FYI<br>OP_CONNECT OP是Operation。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-11 11:55:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/6b/e9/7620ae7e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>雨落～紫竹</span>
  </div>
  <div class="_2_QraFYR_0">网络传输涉及带宽 就算你是nio  你也必须过带宽吧 木桶效应？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你好，我不太明白你的疑问点是什么？网络传输必须要涉及带宽。NIO的核心优势其实就是其事件选择机制，比NIO少耗费线程资源，因为在操作系统层面进行事件就序选择后，只需要处理真正有需求的网络通道。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-03 22:32:33</div>
  </div>
</div>
</div>
</li>
</ul>