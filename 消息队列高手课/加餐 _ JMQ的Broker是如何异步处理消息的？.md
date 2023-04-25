<audio title="加餐 _ JMQ的Broker是如何异步处理消息的？" src="https://static001.geekbang.org/resource/audio/38/c3/381b2f7171557ed827a742c5fad481c3.mp3" controls="controls"></audio> 
<p>你好，我是李玥。</p><p>我们的课程更新到进阶篇之后，通过评论区的留言，我看到有一些同学不太理解，为什么在进阶篇中要讲这些“看起来和消息队列关系不大的”内容呢？</p><p>在这里，我跟你分享一下这门课程的设计思路。我们这门课程的名称是“消息队列高手课”，我希望你在学习完这门课程之后，不仅仅只是成为一个使用消息队列的高手，而是<strong>设计和实现</strong>消息队列的高手。所以我们在设计课程的时候，分了基础篇、进阶篇和案例篇三部分。</p><p>基础篇中我们给大家讲解消息队列的原理和一些使用方法，重点是让大家学会使用消息队列。</p><p>你在进阶篇中，我们课程设计的重点是讲解实现消息队列必备的技术知识，通过分析源码讲解消息队列的实现原理。<strong>希望你通过进阶篇的学习能够掌握到设计、实现消息队列所必备的知识和技术，这些知识和技术也是设计所有高性能、高可靠的分布式系统都需要具备的。</strong></p><p>进阶篇的上半部分，我们每一节课一个专题，来讲解设计实现一个高性能消息队列，必备的技术和知识。这里面每节课中讲解的技术点，不仅可以用来设计消息队列，同学们在设计日常的应用系统中也一定会用得到。</p><p>前几天我在极客时间直播的时候也跟大家透露过，由我所在的京东基础架构团队开发的消息队列JMQ，它的综合性能要显著优于目前公认性能非常好的Kafka。虽然在开发JMQ的过程中有很多的创新，但是对于性能的优化这块，并没有什么全新的划时代的新技术，JMQ之所以能做到这样的极致性能，靠的就是合理地设计和正确地使用已有的这些通用的底层技术和优化技巧。我把这些技术和知识点加以提炼和总结，放在进阶篇的上半部分中。</p><!-- [[[read_end]]] --><p>进阶篇的下半部分，我们主要通过分析源码的方式，来学习优秀开源消息队列产品中的一些实现原理和它们的设计思想。</p><p>在最后的案例篇，我会和大家一起，利用进阶篇中学习的知识，一起来开发一个简单的RPC框架。为什么我们要开发一个RPC框架，而不是一个消息队列？这里面就是希望大家不只是机械的去学习，仅仅是我告诉这个问题怎么解决，你就只是学会了这个问题怎么解决，而是能做到真正理解原理，掌握知识和技术，并且能融会贯通，灵活地去使用。只有这样，你才是真的“学会了”。</p><p>有的同学在看了进阶篇中已更新的这几节课程之后，觉得只讲技术原理不过瘾，希望能看到这些技术是如何在消息队列中应用并落地的，看到具体的实现和代码，所以我以京东JMQ为例，将这些基础技术点在消息队列实现中的应用讲解一下。</p><h2>JMQ的Broker是如何异步处理消息的？</h2><p>对于消息队列的Broker，它最核心的两个流程就是接收生产者发来的消息，以及给消费者发送消息。后者的业务逻辑相对比较简单，影响消息队列性能的关键，就是消息生产的这个业务流程。在JMQ中，经过优化后的消息生产流程，实测它每秒钟可以处理超过100万次请求。</p><p>我们在之前的课程中首先讲了异步的设计思想，这里给你分享一下我在设计这个流程时，是如何来将异步的设计落地的。</p><p>消息生产的流程需要完成的功能是这样的：</p><p><img src="https://static001.geekbang.org/resource/image/a7/ba/a7589a7b4525e107f9b82de133bc43ba.jpg?wh=4124*2895" alt=""></p><ul>
<li>首先，生产者发送一批消息给Broker的主节点；</li>
<li>Broker收到消息之后，会对消息做一系列的解析、检查等处理；</li>
<li>然后，把消息复制给所有的Broker从节点，并且需要把消息写入到磁盘中；</li>
<li>主节点收到大多数从节点的复制成功确认后，给生产者回响应告知消息发送成功。</li>
</ul><p>由于使用各种异步框架或多或少都会有一些性能损失，所以我在设计这个流程的时候，没有使用任何的异步框架，而是自行设计一组互相配合的处理线程来实现，但使用的异步设计思想和我们之前课程中所讲的是一样的。</p><p>对于这个流程，我们设计的线程模型是这样的：</p><p><img src="https://static001.geekbang.org/resource/image/c9/fc/c9bf75cafc246f4ace9d36831e95e1fc.png?wh=1157*580" alt=""></p><p>图中白色的细箭头是数据流，蓝色的箭头是控制流，白色的粗箭头代表远程调用。蓝白相间的方框代表的是处理的步骤，我在蓝色方框中标注了这个步骤是在什么线程中执行的。圆角矩形代表的是流程中需要使用的一些关键的数据结构。</p><p>这里我们设计了6组线程，将一个大的流程拆成了6个小流程。并且整个过程完全是异步化的。</p><p>流程的入口在图中的左上角，Broker在收到来自生产者的发消息请求后，会在一个Handler中处理这些请求，这和我们在普通的业务系统中，用Handler接收HTTP请求是一样的，执行Handler中业务逻辑使用的是Netty的IO线程。</p><p>收到请求后，我们在Handler中不做过多的处理，执行必要的检查后，将请求放到一个内存队列中，也就是图中的Requests Queue。请求被放入队列后，Handler的方法就结束了。可以看到，在Handler中只是把请求放到了队列中，没有太多的业务逻辑，这个执行过程是非常快的，所以即使是处理海量的请求，也不会过多的占用IO线程。</p><p>由于要保证消息的有序性，整个流程的大部分过程是不能并发的，只能单线程执行。所以，接下来我们使用一个线程WriteThread从请求队列中按照顺序来获取请求，依次进行解析请求等其他的处理逻辑，最后将消息序列化并写入存储。序列化后的消息会被写入到一个内存缓存中，就是图中的JournalCache，等待后续的处理。</p><p>执行到这里，一条一条的消息已经被转换成一个连续的字节流，每一条消息都在这个字节流中有一个全局唯一起止位置，也就是这条消息的Offset。后续的处理就不用关心字节流中的内容了，只要确保这个字节流能快速正确的被保存和复制就可以了。</p><p>这里面还有一个工作需要完成，就是给生产者回响应，但在这一步，消息既没有落盘，也没有完成复制，还不能给客户端返回响应，所以我们把待返回的响应按照顺序放到一个内存的链表Pending Callbacks中，并记录每个请求中的消息对应的Offset。</p><p>然后，我们有2个线程，FlushThread和ReplicationThread，这两个线程是并行执行的，分别负责批量异步进行刷盘和复制，刷盘和复制又分别是2个比较复杂的流程，我们暂时不展开讲。刷盘线程不停地将新写入Journal Cache的字节流写到磁盘上，完成一批数据的刷盘，它就会更新一个刷盘位置的内存变量，确保这个刷盘位置之前数据都已经安全的写入磁盘中。复制线程的逻辑也是类似的，同样维护了一个复制位置的内存变量。</p><p>最后，我们设计了一组专门用于发送响应的线程ReponseThreads，在刷盘位置或者复制位置更新后，去检查待返回的响应链表Pending Callbacks，根据QOS级别的设置（因为不同QOS基本对发送成功的定义不一样，有的设置需要消息写入磁盘才算成功，有的需要复制完成才算成功），将刷盘位置或者复制位置之前所有响应，以及已经超时的响应，利用这组线程ReponseThreads异步并行的发送给各个客户端。</p><p>这样就完成了消息生产这个流程。整个流程中，除了JournalCache的加载和卸载需要对文件加锁以外，没有用到其他的锁。每个小流程都不会等待其他流程的共享资源，也就不用互相等待资源（没有数据需要处理时等待上游流程提供数据的情况除外），并且只要有数据就能第一时间处理。</p><p>这个流程中，最核心的部分在于WriteThread执行处理的这个步骤，对每条消息进行处理的这些业务逻辑，都只能在WriteThread中单线程执行，虽然这里面干了很多的事儿，但是我们确保这些逻辑中，没有缓慢的磁盘和网络IO，也没有使用任何的锁来等待资源，全部都是内存操作，这样即使单线程可以非常快速地执行所有的业务逻辑。</p><p><strong>这个里面有很重要的几点优化：</strong></p><ul>
<li>一是我们使用异步设计，把刷盘和复制这两部分比较慢的操作从这个流程中分离出去异步执行；</li>
<li>第二是，我们使用了一个写缓存Journal Cache将一个写磁盘的操作，转换成了一个写内存的操作，来提升数据写入的性能，关于如何使用缓存，后面我会专门用一节课来讲；</li>
<li>第三是，这个处理的全流程是近乎无锁的设计，避免了线程因为等待锁导致的阻塞；</li>
<li>第四是，我们把回复响应这个需要等待资源的操作，也异步放到其他的线程中去执行。</li>
</ul><p>你看，一个看起来很简单的接收请求写入数据并回响应的流程，需要涉及的技术包括：<strong>异步的设计、缓存设计、锁的正确使用、线程协调、序列化和内存管理</strong>，等等。你需要对这些技术都有深入的理解，并合理地使用，才能在确保逻辑正确、数据准确的前提下，做到极致的性能。这也是为什么我们在课程的进阶篇中，用这么多节课来逐一讲解这些“看起来和消息队列没什么关系”的知识点和技术。</p><p>我也希望同学们在学习这些知识点的时候，不仅仅只是记住了，能说出来，用于回答面试问题，还要能真正理解这些知识点和技术背后深刻的思想，并使用在日常的设计和开发过程中。</p><p>比如说，在面试的时候，很多同学都可以很轻松地讲JVM内存结构，也知道怎么用jstat、jmap、jstack这些工具来查看虚拟机的状态。但是，当我给出一个有内存溢出的问题程序和源代码，让他来分析原因并改正的时候，却很少有人能给出正确的答案。在我看来，对于JVM这些基础知识，这样的同学他以为自己已经掌握了，但是，无法领会技术背后的思想，做不到学以致用，那还只是别人知识，不是你的。</p><p>再比如，我下面要说的这个俩大爷的作业，你是真的花时间把代码写出来了，还只是在脑子想了想怎么做，就算完成了呢？</p><h2>俩大爷的思考题</h2><p>我们在进阶篇的开始，花了4节课的内容，来讲解如何实现高性能的异步网络通信，在《<a href="http://time.geekbang.org/column/article/119988">13 | 传输协议：应用程序之间对话的语言</a>》中，我给大家留了一个思考题：写一个程序，让俩大爷在胡同口遇见10万次并记录下耗时。</p><p>有几个同学在留言区分享了自己的代码，每一个同学分享的代码我都仔细读过，有的作业实现了异步的网络通信，有的作业序列化和协议设计实现得很好，但很遗憾的是，没有一份作业能在序列化、协议设计和异步网络传输这几方面都做到我期望的水平。</p><p>在这个作业中，应用到了我们进阶篇中前四节课讲到的几个知识点：</p><ul>
<li>使用异步设计的方法；</li>
<li>异步网络IO；</li>
<li>专用序列化、反序列化方法；</li>
<li>设计良好的传输协议；</li>
<li>双工通信。</li>
</ul><p>这里面特别是双工通信的方法，大家都没能正确的实现。所以，这些作业的实际执行性能也没能达到一个应有的水平。</p><p>这里，我也给出一个作业的参考实现，我们用Go语言来实现这个作业：</p><p>我们用Go语言来实现这个作业：</p><pre><code>package main

import (
	&quot;encoding/binary&quot;
	&quot;fmt&quot;
	&quot;io&quot;
	&quot;net&quot;
	&quot;sync&quot;
	&quot;time&quot;
)

var zRecvCount = uint32(0) // 张大爷听到了多少句话
var lRecvCount = uint32(0) // 李大爷听到了多少句话
var total = uint32(100000) // 总共需要遇见多少次

var z0 = &quot;吃了没，您吶?&quot;
var z3 = &quot;嗨！吃饱了溜溜弯儿。&quot;
var z5 = &quot;回头去给老太太请安！&quot;
var l1 = &quot;刚吃。&quot;
var l2 = &quot;您这，嘛去？&quot;
var l4 = &quot;有空家里坐坐啊。&quot;

var liWriteLock sync.Mutex    // 李大爷的写锁
var zhangWriteLock sync.Mutex // 张大爷的写锁

type RequestResponse struct {
	Serial  uint32 // 序号
	Payload string // 内容
}

// 序列化RequestResponse，并发送
// 序列化后的结构如下：
// 	长度	4字节
// 	Serial 4字节
// 	PayLoad 变长
func writeTo(r *RequestResponse, conn *net.TCPConn, lock *sync.Mutex) {
	lock.Lock()
	defer lock.Unlock()
	payloadBytes := []byte(r.Payload)
	serialBytes := make([]byte, 4)
	binary.BigEndian.PutUint32(serialBytes, r.Serial)
	length := uint32(len(payloadBytes) + len(serialBytes))
	lengthByte := make([]byte, 4)
	binary.BigEndian.PutUint32(lengthByte, length)

	conn.Write(lengthByte)
	conn.Write(serialBytes)
	conn.Write(payloadBytes)
	// fmt.Println(&quot;发送: &quot; + r.Payload)
}

// 接收数据，反序列化成RequestResponse
func readFrom(conn *net.TCPConn) (*RequestResponse, error) {
	ret := &amp;RequestResponse{}
	buf := make([]byte, 4)
	if _, err := io.ReadFull(conn, buf); err != nil {
		return nil, fmt.Errorf(&quot;读长度故障：%s&quot;, err.Error())
	}
	length := binary.BigEndian.Uint32(buf)
	if _, err := io.ReadFull(conn, buf); err != nil {
		return nil, fmt.Errorf(&quot;读Serial故障：%s&quot;, err.Error())
	}
	ret.Serial = binary.BigEndian.Uint32(buf)
	payloadBytes := make([]byte, length-4)
	if _, err := io.ReadFull(conn, payloadBytes); err != nil {
		return nil, fmt.Errorf(&quot;读Payload故障：%s&quot;, err.Error())
	}
	ret.Payload = string(payloadBytes)
	return ret, nil
}

// 张大爷的耳朵
func zhangDaYeListen(conn *net.TCPConn, wg *sync.WaitGroup) {
	defer wg.Done()
	for zRecvCount &lt; total*3 {
		r, err := readFrom(conn)
		if err != nil {
			fmt.Println(err.Error())
			break
		}
		// fmt.Println(&quot;张大爷收到：&quot; + r.Payload)
		if r.Payload == l2 { // 如果收到：您这，嘛去？
			go writeTo(&amp;RequestResponse{r.Serial, z3}, conn, &amp;zhangWriteLock) // 回复：嗨！吃饱了溜溜弯儿。
		} else if r.Payload == l4 { // 如果收到：有空家里坐坐啊。
			go writeTo(&amp;RequestResponse{r.Serial, z5}, conn, &amp;zhangWriteLock) // 回复：回头去给老太太请安！
		} else if r.Payload == l1 { // 如果收到：刚吃。
			// 不用回复
		} else {
			fmt.Println(&quot;张大爷听不懂：&quot; + r.Payload)
			break
		}
		zRecvCount++
	}
}

// 张大爷的嘴
func zhangDaYeSay(conn *net.TCPConn) {
	nextSerial := uint32(0)
	for i := uint32(0); i &lt; total; i++ {
		writeTo(&amp;RequestResponse{nextSerial, z0}, conn, &amp;zhangWriteLock)
		nextSerial++
	}
}

// 李大爷的耳朵，实现是和张大爷类似的
func liDaYeListen(conn *net.TCPConn, wg *sync.WaitGroup) {
	defer wg.Done()
	for lRecvCount &lt; total*3 {
		r, err := readFrom(conn)
		if err != nil {
			fmt.Println(err.Error())
			break
		}
		// fmt.Println(&quot;李大爷收到：&quot; + r.Payload)
		if r.Payload == z0 { // 如果收到：吃了没，您吶?
			writeTo(&amp;RequestResponse{r.Serial, l1}, conn, &amp;liWriteLock) // 回复：刚吃。
		} else if r.Payload == z3 {
			// do nothing
		} else if r.Payload == z5 {
			// do nothing
		} else {
			fmt.Println(&quot;李大爷听不懂：&quot; + r.Payload)
			break
		}
		lRecvCount++
	}
}

// 李大爷的嘴
func liDaYeSay(conn *net.TCPConn) {
	nextSerial := uint32(0)
	for i := uint32(0); i &lt; total; i++ {
		writeTo(&amp;RequestResponse{nextSerial, l2}, conn, &amp;liWriteLock)
		nextSerial++
		writeTo(&amp;RequestResponse{nextSerial, l4}, conn, &amp;liWriteLock)
		nextSerial++
	}
}

func startServer(wg *sync.WaitGroup) {
	tcpAddr, _ := net.ResolveTCPAddr(&quot;tcp&quot;, &quot;127.0.0.1:9999&quot;)
	tcpListener, _ := net.ListenTCP(&quot;tcp&quot;, tcpAddr)
	defer tcpListener.Close()
	fmt.Println(&quot;张大爷在胡同口等着 ...&quot;)
	for {
		conn, err := tcpListener.AcceptTCP()
		if err != nil {
			fmt.Println(err)
			break
		}

		fmt.Println(&quot;碰见一个李大爷:&quot; + conn.RemoteAddr().String())
		go zhangDaYeListen(conn, wg)
		go zhangDaYeSay(conn)
	}

}

func startClient(wg *sync.WaitGroup) *net.TCPConn {
	var tcpAddr *net.TCPAddr
	tcpAddr, _ = net.ResolveTCPAddr(&quot;tcp&quot;, &quot;127.0.0.1:9999&quot;)
	conn, _ := net.DialTCP(&quot;tcp&quot;, nil, tcpAddr)
	go liDaYeListen(conn, wg)
	go liDaYeSay(conn)
	return conn
}

func main() {
	var wg sync.WaitGroup
	wg.Add(2)
	go startServer(&amp;wg)
	time.Sleep(time.Second)
	conn := startClient(&amp;wg)
	t1 := time.Now()
	wg.Wait()
	elapsed := time.Since(t1)
	conn.Close()
	fmt.Println(&quot;耗时: &quot;, elapsed)
}
</code></pre><p>在我的Mac执行10万次大约需要不到5秒钟：</p><pre><code>go run hutong.go
张大爷在胡同口等着 ...
碰见一个李大爷:127.0.0.1:50136
耗时:  4.962786896s
</code></pre><p>在这段程序里面，<strong>我没有对程序做任何特殊的性能优化，只是使用了我们之前四节课中讲到的，上面列出来的那些知识点，完成了一个基本的实现。</strong></p><p>在这段程序中，我们首先定义了RequestResponse这个结构体，代表请求或响应，它包括序号和内容两个字段。readFrom方法的功能是，接收数据，反序列化成RequestResponse。</p><p>协议的设计是这样的：首先用4个字节来标明这个请求的长度，然后用4个字节来保存序号，最后变长的部分就是大爷说的话。这里面用到了使用前置长度的方式来进行断句，这种断句的方式我在之前的课程中专门讲到过。</p><p>这里面我们使用了专有的序列化方法，原因我在之前的课程中重点讲过，专有的序列化方法具备最好的性能，序列化出来的字节数也更少，而我们这个作业比拼的就是性能，所以在这个作业中采用这种序列化方式是最合适的选择。</p><p>zhangDaYeListen和liDaYeListen这两个方法，它们的实现是差不多的，就是接收对方发出的请求，然后给出正确的响应。zhangDaYeSay和liDaYeSay这两个方法的实现也是差不多的，当俩大爷遇见后，就开始不停地说各自的请求，<strong>并不等待对方的响应</strong>，连续说10万次。</p><p>这4个方法，分别在4个不同的协程中并行运行，两收两发，实现了全双工的通信。在这个地方，不少同学还是摆脱不了“一问一答，再问再答”这种人类交流的自然方式对思维的影响，写出来的依然是单工通信的程序，单工通信的性能是远远不如双工通信的，所以，要想做到比较好的网络传输性能，双工通信的方式才是最佳的选择。</p><p>为了避免并发向同一个socket中写入造成数据混乱，我们给俩大爷分别定义了一个写锁，确保每个大爷同一时刻只能有一个协程在发送数据。后面的课程中，我们会专门来讲，如何正确地使用锁。</p><p>最后，我们给张大爷定义为服务端，李大爷定义为客户端，连接建立后，分别开启两个大爷的耳朵和嘴，来完成这10万次遇见。</p><h2>思考题</h2><p>在我给出这个俩大爷作业的实现中，我们可以计算一下，10万次遇见耗时约5秒，平均每秒可以遇见约2万次，<strong>考虑到数据量的大小，这里面仍然有非常大的优化空间。</strong></p><p>请你在充分理解这段代码之后，想一想，还有哪些地方是可以优化的，然后一定要动手把代码改出来，并运行，验证一下你的改进是否真的达到了效果。欢迎你在留言区提出你的改进意见。</p><p>感谢阅读，如果你觉得这篇文章对你有一些启发，也欢迎把它分享给你的朋友。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/95/1c/3cefe42b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>包明</span>
  </div>
  <div class="_2_QraFYR_0">Requests Queue 内存级的 怎么做到不丢请求？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你要明白一件事儿，客户端发来一条数据，直到服务端返回给客户端发送成功，这段时间内，数据是允许丢失的。<br><br>数据丢失后，客户端收不到发送成功响应，自然会重试。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-26 08:17:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/33/1c/59a4e803.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>青舟</span>
  </div>
  <div class="_2_QraFYR_0">https:&#47;&#47;github.com&#47;qingzhou413&#47;geektime-mq-rpc.git<br><br>使用netty作为网络库，server和client的io线程数都是1，笔记本4核标压2.3G时间2.3秒。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍👍👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-24 10:36:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/a9/18/393a841d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>付永强</span>
  </div>
  <div class="_2_QraFYR_0">这么看来JMQ基本上是把所有涉及io等待地方步骤进行异步设计。学到了！感谢分享！🤔</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-28 08:27:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/9c/53/ade0afb0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ub8</span>
  </div>
  <div class="_2_QraFYR_0">老师，文中提到的 “我们把回复响应这个需要等待资源的操作，也异步放到其他的线程中去执行。”；这个是怎么实现的呢？ ResponseThread ,和 RequestThread 是如何对应上的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 一般简单常用的做法都是在处理request的线程中执行业务逻辑，然后把response返回。<br><br>其实从网络层面来看，Request和Response只是客户端和服务端互相发送的两段数据，和服务端处理用什么线程完全没有关系。<br><br>Request和Response如何对应，这个取决于网络传输协议是怎么设计的。这个我们在课程中讲过。<br><br>在服务端实现中，可以接受到Request之后，进行异步处理，异步处理完成之后，无论在什么线程中，只要能拿到处理结果构建出Response，通过网络发给客户端就可以了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-14 12:18:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/53/19/dec74631.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>谢清</span>
  </div>
  <div class="_2_QraFYR_0">刷了下评论，青舟写的最好，其次Martin<br>青舟<br>https:&#47;&#47;github.com&#47;qingzhou413&#47;geektime-mq-rpc.git<br>Martin<br>https:&#47;&#47;github.com&#47;MartinDai&#47;mq-in-action&#47;blob&#47;master&#47;src&#47;main&#47;java&#47;com&#47;doodl6&#47;mq&#47;MeetInRpc.java</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-21 23:19:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/4e/3a/86196508.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>linqw</span>
  </div>
  <div class="_2_QraFYR_0">老师有个疑问，帮忙解答下哦，在上面的那个流程图中，WriteThread是单线程从请求队列中获取到消息然后把消息放到journal Cache，开启ReplicationThread、FlushThread进行处理，能否把WriteThread做成分配器了，mq只要保证topic下的队列有序就可以，同一个队列的消息由WriteThread分配给同一个线程进行处理，线程池的形式，线程池中的每个工作线程内部都有个集合保存消息，如果前面没有同一个队列的消息，分配给最空闲的线程进行执行，那这样的话，WriteThread只要分配消息，比如可以对发送过来的消息中要保存的队列属性值进行hash，然后根据hash值判断线程池中的所有线程的消息集合是否有相同队列的消息，有的话分配给同一个线程执行，没有的话最空闲的线程执行，mysql的binlog同步也是有几种这种策略，并发的同步sql，刚开始是基于库，同一个库的sql分配到一个线程执行同步，不同库进行并发，后来是基于redo log的组提交形式，能组提交的sql可以并发的同步，再后来是WRITESET，根据库名、表名、索引（主键和唯一索引）计算hash进行分发策略。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个流程中处理的数据已经是被分配过的，单个队列的数据。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-25 15:40:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/52/33/abf321b7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>吾皇万岁万岁万万岁</span>
  </div>
  <div class="_2_QraFYR_0">请问老师，JMQ在follower节点响应后，就给生产者发送确认消息，此时如果leader节点故障，数据还在JournalCache里面，拿是不是可以认为这部分数据丢失？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不会丢，因为数据已经复制到了从节点上，leader宕机后，会重新选举出新的leader（也就是之前的某个follower），这个新leader上是有原leader上的全部数据的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-24 15:51:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/24/82/b5808a60.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>李冲</span>
  </div>
  <div class="_2_QraFYR_0">借用老师的的go源码经过读，写，去掉锁3次演进，分别在金山云ECS（2核4G）上跑到3.1&#47;2.4&#47;1.4秒的样子。<br>最终的文件：https:&#47;&#47;github.com&#47;lichongsw&#47;algorithm&#47;blob&#47;master&#47;duplex_communication_optimization_3_no_write_lock.go</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍👍👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-26 17:11:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/6f/f2/1f77b0bd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>李心宇🦉</span>
  </div>
  <div class="_2_QraFYR_0">老师好，我对JMQ的broker接收生产者请求并写入消息的流程有个疑问。<br>在处理完数据落盘和多节点数据复制之后，要给生产者回复响应了，这时候broker如何能找到生产者呢？我理解是第一步生产者发送请求建立的TCP连接句柄没有释放，最后再通过这个连接句柄来write响应。这样的话，还是每个连接在得到响应之前不能释放需要占用一个线程啊。请问是怎么做到在第一步接收响应阶段只需要很少的线程的？是不是利用异步非阻塞，在线程里设置大量的协程来处理请求？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你的大部分理解都没问题，有一个小问题是，维持一个TCP连接并不一定需要占用一个线程。只有在这个连接上执行收发数据的时候，才需要占用线程。收到请求处理完，把请求交给其它线程处理后，当前线程就可以释放了。直到响应生成后，这段时间只维持TCP连接是不需要占用任何线程的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-23 10:30:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/58/3d/4ba98f04.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Martin</span>
  </div>
  <div class="_2_QraFYR_0">https:&#47;&#47;github.com&#47;MartinDai&#47;mq-in-action&#47;blob&#47;master&#47;src&#47;main&#47;java&#47;com&#47;doodl6&#47;mq&#47;MeetInRpc.java<br><br>基于netty4实现的，Macbook pro 2015款 13寸 测试差不多4.2秒左右，老师帮忙看看哪里还可以有优化空间吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍👍👍正确写出这个程序差不多就是这个耗时。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-20 12:09:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4d/fd/0aa0e39f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>许童童</span>
  </div>
  <div class="_2_QraFYR_0">老师说得很好，想成为一个真成的高手，理解并实践这些基本原理，是必不可少的，不要只停留在知识的表面，要领会技术背后的思想，才能活学活用，实践肯定是少不了的，加油。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-24 11:47:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/88/26/b8c53cee.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>南辕北辙</span>
  </div>
  <div class="_2_QraFYR_0">把老师的代码看了n遍，再用java的原生nio实现了一遍，理解了老师的用心良苦。前二章序列化与网络协议那块的知识，对应代码对于struct的构造，四字节的总长度，四字节的序号，以及变长内容，然后二个大爷在接受到数据以后也是根据这个协议进行取数据。也就对应了序列化以及协议都是自定义的！！二个大爷都懂该怎么从二进制的数据中，提取出对话。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 恭喜你学到了😄</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-26 17:20:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/e7/0f/fa840c1b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>刘天鹏</span>
  </div>
  <div class="_2_QraFYR_0">老师的writeTo函数没必要加锁吧，用一个局部缓冲把lengthByte serialBytes payloadBytes拼装好再一起发送应该就可以了<br>我把我的大爷改成双工的，多线程的，可以打包发送的，现在大爷还只有一张嘴一个耳朵（一个socket连接）<br>可能是没有逻辑处理的负担，多线程没啥改变<br>打包发送有一定的改进，1次3条数据 14s<br>现在100万次胡同碰面，8.66s(公司的i7)<br><br>https:&#47;&#47;gist.github.com&#47;liutianpeng&#47;d9330f85d47525a8e32dcd24f5738e55</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果是多个线程并发向同一个socket中写数据，这个锁是必须加的。<br><br>比如，A线程发送“123456”，B线程发送“abcd”，如果不加锁，对端收到的可能就成了“12abc345d6”，这种数据是对端是没法解析的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-26 11:58:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/d7/e3/e6cf6352.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>nfx</span>
  </div>
  <div class="_2_QraFYR_0">有个疑问,  request queue, journal cache都是生产者, 消费者模型的队列,  里面应该会有锁吧?  只是这种队列锁需要的资源很少. <br> 不知道怎么理解对不对</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个地方理论上是不需要锁的，因为是Append Only的数据模型，也就是说数据在尾部追加写入，写入后不可改变，那读写就不会有冲突，所以不需要加锁。<br><br>但实际上，因为内存是有限的，总是要发生换页操作，相当于把数据从内存中删除，这个地方还是需要用锁来控制的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-28 17:10:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/43/50/abb4ca1e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>凡</span>
  </div>
  <div class="_2_QraFYR_0">借鉴了 MySql 的思想，跟新数据先写入 Buffer pool，然后定期将内存的数据刷到磁盘，这样减少了磁盘 IO 的操作频率，提高了性能</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-23 22:48:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/d1/16/6347bbc0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>顶新</span>
  </div>
  <div class="_2_QraFYR_0">FlushThread 和 ReplicationThread线程对内存的链表 Pending Callbacks 中数据的更新，然后ResponseThread扫描链表Pending Callbacks，批量取出符合 QOS 规则的响应及超时的响应，然后并发返回给客户端。这块也存在对 Pending Callbacks 存在互斥锁吧？不知道理解的对不对，还请老师答疑 :)</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个地方是不需要锁的，写入和移出Callback分别是在2个固定的线程中分别执行，这个链表写入的时候直接是尾部追加，执行Callback的时候在头部移出。并且，执行Callback的时候一定链表中有数据的时候，所以也不需要加锁担心移除越界。所以这个地方是不需要锁的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-18 15:50:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/4e/29/1be3dd40.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ykkk88</span>
  </div>
  <div class="_2_QraFYR_0">如果要保证前一句话要对方确认回复后再发送下一句话 就不能用这种方式了吧？这样吞吐量就会很低了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的。<br><br>一般的协议都是这种“请求-响应”的方式，不需要同步的请求之间有序。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-25 21:21:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/e4/3d/5d51fda4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>TWT_Marq</span>
  </div>
  <div class="_2_QraFYR_0">JMQ中，接受请求的iothreads将请求丢给request queue，WriteThread从queue中取出来开始干活。这个queue是concurrentBlockingQueue吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-25 20:35:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/4e/3a/86196508.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>linqw</span>
  </div>
  <div class="_2_QraFYR_0">我用java实现了一版，老师帮忙看下哦，评论只能发2000字，其余在评论区补上<br>import org.slf4j.Logger;<br>import org.slf4j.LoggerFactory;<br><br>import javax.annotation.Nonnull;<br>import java.io.IOException;<br>import java.io.InputStream;<br>import java.io.OutputStream;<br>import java.net.ServerSocket;<br>import java.net.Socket;<br>import java.util.concurrent.*;<br>import java.util.concurrent.atomic.AtomicInteger;<br>import java.util.concurrent.atomic.AtomicLong;<br>import java.util.concurrent.locks.ReentrantLock;<br><br>&#47;**<br> * @author linqw<br> *&#47;<br>public class SocketExample {<br><br>    private static final Logger LOGGER = LoggerFactory.getLogger(SocketExample.class);<br><br>    private static final ReentrantLock LI_WRITE_REENTRANT_LOCK = new ReentrantLock();<br><br>    private static final ReentrantLock ZHANG_WRITE_REENTRANT_LOCK = new ReentrantLock();<br><br>    private static final int NCPU = Runtime.getRuntime().availableProcessors();<br><br>    private final ThreadPoolExecutor serverThreadPoolExecutor = new ThreadPoolExecutor(NCPU, NCPU, 5, TimeUnit.SECONDS, new ArrayBlockingQueue&lt;Runnable&gt;(200000), new ThreadFactoryImpl(&quot;server-&quot;), new ThreadPoolExecutor.CallerRunsPolicy());<br><br>    private final ThreadPoolExecutor clientThreadPoolExecutor = new ThreadPoolExecutor(NCPU, NCPU, 5, TimeUnit.SECONDS, new ArrayBlockingQueue&lt;Runnable&gt;(200000), new ThreadFactoryImpl(&quot;client-&quot;), new ThreadPoolExecutor.CallerRunsPolicy());<br><br>    private volatile boolean started = false;<br><br>    &#47;**<br>     * 俩大爷已经遇见了多少次<br>     *&#47;<br>    private volatile AtomicInteger count = new AtomicInteger(0);<br><br>    &#47;**<br>     * 总共需要遇见多少次<br>     *&#47;<br>    private static final int TOTAL = 1;<br><br>    private static final String Z0 = &quot; 吃了没，您吶?&quot;;<br><br>    private static final String Z3 = &quot; 嗨！吃饱了溜溜弯儿。&quot;;<br>    private static final String Z5 = &quot; 回头去给老太太请安！&quot;;<br>    private static final String L1 = &quot; 刚吃。&quot;;<br>    private static final String L2 = &quot; 您这，嘛去？&quot;;<br>    private static final String L4 = &quot; 有空家里坐坐啊。&quot;;</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 还是贴一下GitHub的链接把，也方便其他同学学习。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-25 15:19:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/ad/12/d4d54dd2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>豆沙包</span>
  </div>
  <div class="_2_QraFYR_0">老师，我在运行你的老大爷代码的时候发现了一个问题，经常报读长度故障。我调试了一下，应该是因为张大爷阻塞在readfrom的时候，李大爷count++了。然后count大于total，client释放了tcp。这导致readfrom无法读出准确数据。我在返回读数据故障前判断了一下count和total的关系，如果count大于等于total正常返回，就没有再报过异常了。您看我分析的是否有问题呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个地方是因为，完成了所有会话之后，关闭socket连接的时候，没有先等待2个大爷读写socket的4个线程都结束导致的。<br><br>虽然不影响结果，但还是可以改进一下，做到优雅的退出。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-24 23:13:07</div>
  </div>
</div>
</div>
</li>
</ul>