<audio title="13 _ API 风格（下）：RPC API介绍" src="https://static001.geekbang.org/resource/audio/2f/c0/2f388c7254df251057a18275aa4716c0.mp3" controls="controls"></audio> 
<p>你好，我是孔令飞。这一讲，我们继续来看下如何设计应用的API风格。</p><p>上一讲，我介绍了REST API风格，这一讲我来介绍下另外一种常用的API风格，RPC。在Go项目开发中，如果业务对性能要求比较高，并且需要提供给多种编程语言调用，这时候就可以考虑使用RPC API接口。RPC在Go项目开发中用得也非常多，需要我们认真掌握。</p><h2>RPC介绍</h2><p>根据维基百科的定义，RPC（Remote Procedure Call），即远程过程调用，是一个计算机通信协议。该协议允许运行于一台计算机的程序调用另一台计算机的子程序，而程序员不用额外地为这个交互作用编程。</p><p>通俗来讲，就是服务端实现了一个函数，客户端使用RPC框架提供的接口，像调用本地函数一样调用这个函数，并获取返回值。RPC屏蔽了底层的网络通信细节，使得开发人员无需关注网络编程的细节，可以将更多的时间和精力放在业务逻辑本身的实现上，从而提高开发效率。</p><p>RPC的调用过程如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/98/1d/984yy094616b9b24193b22a1f2f2271d.png?wh=2521x1671" alt=""></p><p>RPC调用具体流程如下：</p><ol>
<li>Client通过本地调用，调用Client Stub。</li>
<li>Client Stub将参数打包（也叫Marshalling）成一个消息，然后发送这个消息。</li>
<li>Client所在的OS将消息发送给Server。</li>
<li>Server端接收到消息后，将消息传递给Server Stub。</li>
<li>Server Stub将消息解包（也叫 Unmarshalling）得到参数。</li>
<li>Server Stub调用服务端的子程序（函数），处理完后，将最终结果按照相反的步骤返回给 Client。</li>
</ol><!-- [[[read_end]]] --><p>这里需要注意，Stub负责调用参数和返回值的流化（serialization）、参数的打包和解包，以及网络层的通信。Client端一般叫Stub，Server端一般叫Skeleton。</p><p>目前，业界有很多优秀的RPC协议，例如腾讯的Tars、阿里的Dubbo、微博的Motan、Facebook的Thrift、RPCX，等等。但使用最多的还是<a href="https://github.com/grpc/grpc-go">gRPC</a>，这也是本专栏所采用的RPC框架，所以接下来我会重点介绍gRPC框架。</p><h2>gRPC介绍</h2><p>gRPC是由Google开发的高性能、开源、跨多种编程语言的通用RPC框架，基于HTTP 2.0协议开发，默认采用Protocol Buffers数据序列化协议。gRPC具有如下特性：</p><ul>
<li>支持多种语言，例如 Go、Java、C、C++、C#、Node.js、PHP、Python、Ruby等。</li>
<li>基于IDL（Interface Definition Language）文件定义服务，通过proto3工具生成指定语言的数据结构、服务端接口以及客户端Stub。通过这种方式，也可以将服务端和客户端解耦，使客户端和服务端可以并行开发。</li>
<li>通信协议基于标准的HTTP/2设计，支持双向流、消息头压缩、单TCP的多路复用、服务端推送等特性。</li>
<li>支持Protobuf和JSON序列化数据格式。Protobuf是一种语言无关的高性能序列化框架，可以减少网络传输流量，提高通信效率。</li>
</ul><p>这里要注意的是，gRPC的全称不是golang Remote Procedure Call，而是google Remote Procedure Call。</p><p>gRPC的调用如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/01/09/01ac424c7c1d64f678e1218827bc0109.png?wh=2079x1025" alt=""></p><p>在gRPC中，客户端可以直接调用部署在不同机器上的gRPC服务所提供的方法，调用远端的gRPC方法就像调用本地的方法一样，非常简单方便，通过gRPC调用<strong>，我们可以非常容易地构建出一个分布式应用。</strong></p><p>像很多其他的RPC服务一样，gRPC也是通过IDL语言，预先定义好接口（接口的名字、传入参数和返回参数等）。在服务端，gRPC服务实现我们所定义的接口。在客户端，gRPC存根提供了跟服务端相同的方法。</p><p>gRPC支持多种语言，比如我们可以用Go语言实现gRPC服务，并通过Java语言客户端调用gRPC服务所提供的方法。通过多语言支持，我们编写的gRPC服务能满足客户端多语言的需求。</p><p>gRPC API接口通常使用的数据传输格式是Protocol Buffers。接下来，我们就一起了解下Protocol Buffers。</p><h2>Protocol Buffers介绍</h2><p>Protocol Buffers（ProtocolBuffer/ protobuf）是Google开发的一套对数据结构进行序列化的方法，可用作（数据）通信协议、数据存储格式等，也是一种更加灵活、高效的数据格式，与XML、JSON类似。它的传输性能非常好，所以常被用在一些对数据传输性能要求比较高的系统中，作为数据传输格式。Protocol Buffers的主要特性有下面这几个。</p><ul>
<li>更快的数据传输速度：protobuf在传输时，会将数据序列化为二进制数据，和XML、JSON的文本传输格式相比，这可以节省大量的IO操作，从而提高数据传输速度。</li>
<li>跨平台多语言：protobuf自带的编译工具 protoc 可以基于protobuf定义文件，编译出不同语言的客户端或者服务端，供程序直接调用，因此可以满足多语言需求的场景。</li>
<li>具有非常好的扩展性和兼容性，可以更新已有的数据结构，而不破坏和影响原有的程序。</li>
<li>基于IDL文件定义服务，通过proto3工具生成指定语言的数据结构、服务端和客户端接口。</li>
</ul><p>在gRPC的框架中，Protocol Buffers主要有三个作用。</p><p><strong>第一，可以用来定义数据结构。</strong>举个例子，下面的代码定义了一个SecretInfo数据结构：</p><pre><code>// SecretInfo contains secret details.
message SecretInfo {
    string name = 1;
    string secret_id  = 2;
    string username   = 3;
    string secret_key = 4;
    int64 expires = 5;
    string description = 6;
    string created_at = 7;
    string updated_at = 8;
}
</code></pre><p><strong>第二，可以用来定义服务接口。</strong>下面的代码定义了一个Cache服务，服务包含了ListSecrets和ListPolicies 两个API接口。</p><pre><code>// Cache implements a cache rpc service.
service Cache{
  rpc ListSecrets(ListSecretsRequest) returns (ListSecretsResponse) {}
  rpc ListPolicies(ListPoliciesRequest) returns (ListPoliciesResponse) {}
}
</code></pre><p><strong>第三，可以通过protobuf序列化和反序列化，提升传输效率。</strong></p><h2>gRPC示例</h2><p>我们已经对gRPC这一通用RPC框架有了一定的了解，但是你可能还不清楚怎么使用gRPC编写API接口。接下来，我就通过gRPC官方的一个示例来快速给大家展示下。运行本示例需要在Linux服务器上安装Go编译器、Protocol buffer编译器（protoc，v3）和 protoc 的Go语言插件，在 <a href="https://time.geekbang.org/column/article/378076"><strong>02讲</strong></a> 中我们已经安装过，这里不再讲具体的安装方法。</p><p>这个示例分为下面几个步骤：</p><ol>
<li>定义gRPC服务。</li>
<li>生成客户端和服务器代码。</li>
<li>实现gRPC服务。</li>
<li>实现gRPC客户端。</li>
</ol><p>示例代码存放在<a href="https://github.com/marmotedu/gopractise-demo/tree/main/apistyle/greeter">gopractise-demo/apistyle/greeter</a>目录下。代码结构如下：</p><pre><code>$ tree
├── client
│   └── main.go
├── helloworld
│   ├── helloworld.pb.go
│   └── helloworld.proto
└── server
    └── main.go
</code></pre><p>client目录存放Client端的代码，helloworld目录用来存放服务的IDL定义，server目录用来存放Server端的代码。</p><p>下面我具体介绍下这个示例的四个步骤。</p><ol>
<li>定义gRPC服务。</li>
</ol><p>首先，需要定义我们的服务。进入helloworld目录，新建文件helloworld.proto：</p><pre><code>$ cd helloworld
$ vi helloworld.proto
</code></pre><p>内容如下：</p><pre><code>syntax = &quot;proto3&quot;;

option go_package = &quot;github.com/marmotedu/gopractise-demo/apistyle/greeter/helloworld&quot;;

package helloworld;

// The greeting service definition.
service Greeter {
  // Sends a greeting
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// The request message containing the user's name.
message HelloRequest {
  string name = 1;
}

// The response message containing the greetings
message HelloReply {
  string message = 1;
}
</code></pre><p>在helloworld.proto定义文件中，option关键字用来对.proto文件进行一些设置，其中go_package是必需的设置，而且go_package的值必须是包导入的路径。package关键字指定生成的.pb.go文件所在的包名。我们通过service关键字定义服务，然后再指定该服务拥有的RPC方法，并定义方法的请求和返回的结构体类型：</p><pre><code>service Greeter {
  // Sends a greeting
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}
</code></pre><p>gRPC支持定义4种类型的服务方法，分别是简单模式、服务端数据流模式、客户端数据流模式和双向数据流模式。</p><ul>
<li>
<p>简单模式（Simple RPC）：是最简单的gRPC模式。客户端发起一次请求，服务端响应一个数据。定义格式为rpc SayHello (HelloRequest) returns (HelloReply) {}。</p>
</li>
<li>
<p>服务端数据流模式（Server-side streaming RPC）：客户端发送一个请求，服务器返回数据流响应，客户端从流中读取数据直到为空。定义格式为rpc SayHello (HelloRequest) returns (stream HelloReply) {}。</p>
</li>
<li>
<p>客户端数据流模式（Client-side streaming RPC）：客户端将消息以流的方式发送给服务器，服务器全部处理完成之后返回一次响应。定义格式为rpc SayHello (stream HelloRequest) returns (HelloReply) {}。</p>
</li>
<li>
<p>双向数据流模式（Bidirectional streaming RPC）：客户端和服务端都可以向对方发送数据流，这个时候双方的数据可以同时互相发送，也就是可以实现实时交互RPC框架原理。定义格式为rpc SayHello (stream HelloRequest) returns (stream HelloReply) {}。</p>
</li>
</ul><p>本示例使用了简单模式。.proto文件也包含了Protocol Buffers 消息的定义，包括请求消息和返回消息。例如请求消息：</p><pre><code>// The request message containing the user's name.
message HelloRequest {
  string name = 1;
}
</code></pre><ol start="2">
<li>生成客户端和服务器代码。</li>
</ol><p>接下来，我们需要根据.proto服务定义生成gRPC客户端和服务器接口。我们可以使用protoc编译工具，并指定使用其Go语言插件来生成：</p><pre><code>$ protoc -I. --go_out=plugins=grpc:$GOPATH/src helloworld.proto
$ ls
helloworld.pb.go  helloworld.proto
</code></pre><p>你可以看到，新增了一个helloworld.pb.go文件。</p><ol start="3">
<li>实现gRPC服务。</li>
</ol><p>接着，我们就可以实现gRPC服务了。进入server目录，新建main.go文件：</p><pre><code>$ cd ../server
$ vi main.go
</code></pre><p>main.go内容如下：</p><pre><code>// Package main implements a server for Greeter service.
package main

import (
	&quot;context&quot;
	&quot;log&quot;
	&quot;net&quot;

	pb &quot;github.com/marmotedu/gopractise-demo/apistyle/greeter/helloworld&quot;
	&quot;google.golang.org/grpc&quot;
)

const (
	port = &quot;:50051&quot;
)

// server is used to implement helloworld.GreeterServer.
type server struct {
	pb.UnimplementedGreeterServer
}

// SayHello implements helloworld.GreeterServer
func (s *server) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
	log.Printf(&quot;Received: %v&quot;, in.GetName())
	return &amp;pb.HelloReply{Message: &quot;Hello &quot; + in.GetName()}, nil
}

func main() {
	lis, err := net.Listen(&quot;tcp&quot;, port)
	if err != nil {
		log.Fatalf(&quot;failed to listen: %v&quot;, err)
	}
	s := grpc.NewServer()
	pb.RegisterGreeterServer(s, &amp;server{})
	if err := s.Serve(lis); err != nil {
		log.Fatalf(&quot;failed to serve: %v&quot;, err)
	}
}
</code></pre><p>上面的代码实现了我们上一步根据服务定义生成的Go接口。</p><p>我们先定义了一个Go结构体server，并为server结构体添加<code>SayHello(context.Context, pb.HelloRequest) (pb.HelloReply, error)</code>方法，也就是说server是GreeterServer接口（位于helloworld.pb.go文件中）的一个实现。</p><p>在我们实现了gRPC服务所定义的方法之后，就可以通过 <code>net.Listen(...)</code> 指定监听客户端请求的端口；接着，通过 <code>grpc.NewServer()</code> 创建一个gRPC Server实例，并通过 <code>pb.RegisterGreeterServer(s, &amp;server{})</code> 将该服务注册到gRPC框架中；最后，通过 <code>s.Serve(lis)</code> 启动gRPC服务。</p><p>创建完main.go文件后，在当前目录下执行 <code>go run main.go</code> ，启动gRPC服务。</p><ol start="4">
<li>实现gRPC客户端。</li>
</ol><p>打开一个新的Linux终端，进入client目录，新建main.go文件：</p><pre><code>$ cd ../client
$ vi main.go
</code></pre><p>main.go内容如下：</p><pre><code>// Package main implements a client for Greeter service.
package main

import (
	&quot;context&quot;
	&quot;log&quot;
	&quot;os&quot;
	&quot;time&quot;

	pb &quot;github.com/marmotedu/gopractise-demo/apistyle/greeter/helloworld&quot;
	&quot;google.golang.org/grpc&quot;
)

const (
	address     = &quot;localhost:50051&quot;
	defaultName = &quot;world&quot;
)

func main() {
	// Set up a connection to the server.
	conn, err := grpc.Dial(address, grpc.WithInsecure(), grpc.WithBlock())
	if err != nil {
		log.Fatalf(&quot;did not connect: %v&quot;, err)
	}
	defer conn.Close()
	c := pb.NewGreeterClient(conn)

	// Contact the server and print out its response.
	name := defaultName
	if len(os.Args) &gt; 1 {
		name = os.Args[1]
	}
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()
	r, err := c.SayHello(ctx, &amp;pb.HelloRequest{Name: name})
	if err != nil {
		log.Fatalf(&quot;could not greet: %v&quot;, err)
	}
	log.Printf(&quot;Greeting: %s&quot;, r.Message)
}
</code></pre><p>在上面的代码中，我们通过如下代码创建了一个gRPC连接，用来跟服务端进行通信：</p><pre><code>// Set up a connection to the server.
conn, err := grpc.Dial(address, grpc.WithInsecure(), grpc.WithBlock())
if err != nil {
    log.Fatalf(&quot;did not connect: %v&quot;, err)
}
defer conn.Close()
</code></pre><p>在创建连接时，我们可以指定不同的选项，用来控制创建连接的方式，例如grpc.WithInsecure()、grpc.WithBlock()等。gRPC支持很多选项，更多的选项可以参考grpc仓库下<a href="https://github.com/grpc/grpc-go/blob/v1.37.0/dialoptions.go">dialoptions.go</a>文件中以With开头的函数。</p><p>连接建立起来之后，我们需要创建一个客户端stub，用来执行RPC请求<code>c := pb.NewGreeterClient(conn)</code>。创建完成之后，我们就可以像调用本地函数一样，调用远程的方法了。例如，下面一段代码通过 <code>c.SayHello</code> 这种本地式调用方式调用了远端的SayHello接口：</p><pre><code>r, err := c.SayHello(ctx, &amp;pb.HelloRequest{Name: name})
if err != nil {
    log.Fatalf(&quot;could not greet: %v&quot;, err)
}
log.Printf(&quot;Greeting: %s&quot;, r.Message)
</code></pre><p>从上面的调用格式中，我们可以看到RPC调用具有下面两个特点。</p><ul>
<li>调用方便：RPC屏蔽了底层的网络通信细节，使得调用RPC就像调用本地方法一样方便，调用方式跟大家所熟知的调用类的方法一致：<code>ClassName.ClassFuc(params)</code>。</li>
<li>不需要打包和解包：RPC调用的入参和返回的结果都是Go的结构体，不需要对传入参数进行打包操作，也不需要对返回参数进行解包操作，简化了调用步骤。</li>
</ul><p>最后，创建完main.go文件后，在当前目录下，执行go run main.go发起RPC调用：</p><pre><code>$ go run main.go
2020/10/17 07:55:00 Greeting: Hello world
</code></pre><p>至此，我们用四个步骤，创建并调用了一个gRPC服务。接下来我再给大家讲解一个在具体场景中的注意事项。</p><p>在做服务开发时，我们经常会遇到一种场景：定义一个接口，接口会通过判断是否传入某个参数，决定接口行为。例如，我们想提供一个GetUser接口，期望GetUser接口在传入username参数时，根据username查询用户的信息，如果没有传入username，则默认根据userId查询用户信息。</p><p>这时候，我们需要判断客户端有没有传入username参数。我们不能根据username是否为空值来判断，因为我们不能区分客户端传的是空值，还是没有传username参数。这是由Go语言的语法特性决定的：如果客户端没有传入username参数，Go会默认赋值为所在类型的零值，而字符串类型的零值就是空字符串。</p><p>那我们怎么判断客户端有没有传入username参数呢？最好的方法是通过指针来判断，如果是nil指针就说明没有传入，非nil指针就说明传入，具体实现步骤如下：</p><ol>
<li>编写protobuf定义文件。</li>
</ol><p>新建user.proto文件，内容如下:</p><pre><code>syntax = &quot;proto3&quot;;

package proto;
option go_package = &quot;github.com/marmotedu/gopractise-demo/protobuf/user&quot;;

//go:generate protoc -I. --experimental_allow_proto3_optional --go_out=plugins=grpc:.

service User {
  rpc GetUser(GetUserRequest) returns (GetUserResponse) {}
}

message GetUserRequest {
  string class = 1;
  optional string username = 2;
  optional string user_id = 3;
}

message GetUserResponse {
  string class = 1;
  string user_id = 2;
  string username = 3;
  string address = 4;
  string sex = 5;
  string phone = 6;
}
</code></pre><p>你需要注意，这里我们在需要设置为可选字段的前面添加了<strong>optional</strong>标识。</p><ol start="2">
<li>使用protoc工具编译protobuf文件。</li>
</ol><p>在执行protoc命令时，需要传入<code>--experimental_allow_proto3_optional</code>参数以打开<strong>optional</strong>选项，编译命令如下：</p><pre><code>$ protoc --experimental_allow_proto3_optional --go_out=plugins=grpc:. user.proto
</code></pre><p>上述编译命令会生成user.pb.go文件，其中的GetUserRequest结构体定义如下：</p><pre><code>type GetUserRequest struct {
    state         protoimpl.MessageState
    sizeCache     protoimpl.SizeCache
    unknownFields protoimpl.UnknownFields

    Class    string  `protobuf:&quot;bytes,1,opt,name=class,proto3&quot; json:&quot;class,omitempty&quot;`
    Username *string `protobuf:&quot;bytes,2,opt,name=username,proto3,oneof&quot; json:&quot;username,omitempty&quot;`
    UserId   *string `protobuf:&quot;bytes,3,opt,name=user_id,json=userId,proto3,oneof&quot; json:&quot;user_id,omitempty&quot;`
}
</code></pre><p>通过 <code>optional</code> + <code>--experimental_allow_proto3_optional</code> 组合，我们可以将一个字段编译为指针类型。</p><ol start="3">
<li>编写gRPC接口实现。</li>
</ol><p>新建一个user.go文件，内容如下：</p><pre><code>package user

import (
    &quot;context&quot;

    pb &quot;github.com/marmotedu/api/proto/apiserver/v1&quot;

    &quot;github.com/marmotedu/iam/internal/apiserver/store&quot;
)

type User struct {
}

func (c *User) GetUser(ctx context.Context, r *pb.GetUserRequest) (*pb.GetUserResponse, error) {
    if r.Username != nil {
        return store.Client().Users().GetUserByName(r.Class, r.Username)
    }

    return store.Client().Users().GetUserByID(r.Class, r.UserId)
}
</code></pre><p>总之，在GetUser方法中，我们可以通过判断r.Username是否为nil，来判断客户端是否传入了Username参数。</p><h2>RESTful VS gRPC</h2><p>到这里，今天我们已经介绍完了gRPC API。回想一下我们昨天学习的RESTful API，你可能想问：这两种API风格分别有什么优缺点，适用于什么场景呢？我把这个问题的答案放在了下面这张表中，你可以对照着它，根据自己的需求在实际应用时进行选择。</p><p><img src="https://static001.geekbang.org/resource/image/e6/ab/e6ae61fc4b0fc821f94d257239f332ab.png?wh=1483x1026" alt=""></p><p>当然，更多的时候，RESTful API 和gRPC API是一种合作的关系，对内业务使用gRPC API，对外业务使用RESTful API，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/47/18/471ac923d2eaeca8fe13cb74731c1318.png?wh=1606x1144" alt=""></p><h2>总结</h2><p>在Go项目开发中，我们可以选择使用 RESTful API 风格和 RPC API 风格，这两种服务都用得很多。其中，RESTful API风格因为规范、易理解、易用，所以<strong>适合用在需要对外提供API接口的场景中</strong>。而RPC API因为性能比较高、调用方便，<strong>更适合用在内部业务中</strong>。</p><p>RESTful API使用的是HTTP协议，而RPC API使用的是RPC协议。目前，有很多RPC协议可供你选择，而我推荐你使用gRPC，因为它很轻量，同时性能很高、很稳定，是一个优秀的RPC框架。所以目前业界用的最多的还是gRPC协议，腾讯、阿里等大厂内部很多核心的线上服务用的就是gRPC。</p><p>除了使用gRPC协议，在进行Go项目开发前，你也可以了解业界一些其他的优秀Go RPC框架，比如腾讯的tars-go、阿里的dubbo-go、Facebook的thrift、rpcx等，你可以在项目开发之前一并调研，根据实际情况进行选择。</p><h2>课后练习</h2><ol>
<li>使用gRPC包，快速实现一个RPC API服务，并实现PrintHello接口，该接口会返回“Hello World”字符串。</li>
<li>请你思考这个场景：你有一个gRPC服务，但是却希望该服务同时也能提供RESTful API接口，这该如何实现？</li>
</ol><p>期待在留言区看到你的思考和答案，也欢迎和我一起探讨关于RPC API相关的问题，我们下一讲见！</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/52/40/e57a736e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>pedro</span>
  </div>
  <div class="_2_QraFYR_0">假定希望用RPC作为内部API的通讯，同时也想对外提供RESTful API，又不想写两套，可以使用gRPC Gateway 插件，在生成RPC的同时也生成RESTful web  server。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 正解</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-24 07:42:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_e7af5e</span>
  </div>
  <div class="_2_QraFYR_0">这里提供一些更新说明（刚踩过的坑）<br>github.com&#47;golang&#47;protobuf&#47;protoc-gen-go和google.golang.org&#47;protobuf&#47;cmd&#47;protoc-gen-go是不同的。区别在于前者是旧版本（操作类似于作者大大的），后者是google接管后的新版本，他们之间的API是不同的，也就是说用于生成的命令，以及生成的文件都是不一样的。因为目前的gRPC-go源码中的example用的是后者的生成方式，所以这里提供后者说明：<br>1. 首先需要安装两个库：<br>go install google.golang.org&#47;protobuf&#47;cmd&#47;protoc-gen-go<br>go install google.golang.org&#47;grpc&#47;cmd&#47;protoc-gen-go-grpc<br>2. 然后.proto文件保持一致，输入两个生成代码命令：<br>protoc -I. --go_out=$GOPATH&#47;src helloworld.proto<br>protoc -I. --go-grpc_out=$GOPATH&#47;src helloworld.proto<br>3. 上述两个命令会生成两个文件：<br>helloworld.pb.go      helloworld_grpc.pb.go<br>这两个文件分别生成message和service的代码，合起来就是老版本的代码<br><br>这是排查了两小时的坑，希望大家注意！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢分享！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-21 16:32:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/15/9b/9ce9f374.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>柠柠</span>
  </div>
  <div class="_2_QraFYR_0">RPC 与 RESTful 共通逻辑抽象出来 Service 层，RPC server 和 RESTful server 初始化or 启动时时都需要指定 service，真正提供服务的是 Service 层</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对的！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-25 15:55:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/68/d4/c9b5d3f9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>💎A</span>
  </div>
  <div class="_2_QraFYR_0">https:&#47;&#47;www.bookstack.cn&#47;read&#47;API-design-guide&#47;API-design-guide-04-%E6%A0%87%E5%87%86%E6%96%B9%E6%B3%95.md  我又来做贡献了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢分享，文章非常棒！已收藏！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-25 08:38:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/22/9a/52/93416b65.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>不明真相的群众</span>
  </div>
  <div class="_2_QraFYR_0">你有一个 gRPC 服务，但是却希望该服务同时也能提供 RESTful API 接口，这该如何实现？<br>---------------------------<br>在封装一层？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 用这个：https:&#47;&#47;github.com&#47;grpc-ecosystem&#47;grpc-gateway</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-24 14:24:46</div>
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
  <div class="_2_QraFYR_0">想方便调用grpc，可以使用grpcurl和grpcui，基于反射的方式使用</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢分享！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-10 21:52:36</div>
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
  <div class="_2_QraFYR_0">老师有个疑问<br>protoc --go_out=. *.proto<br>protoc --go_out=plugins=grpc:. *.proto<br>这俩有啥啥区别  😵</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 第二个指定了grpc plugin</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-12 18:28:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/3a/59/d0f326e7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张名哲</span>
  </div>
  <div class="_2_QraFYR_0">老师，文章中有四种模式，平时用的最多的是哪一种模式？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 简单模式用的比较多</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-13 21:34:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/ed/0a/18201290.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Juniper</span>
  </div>
  <div class="_2_QraFYR_0">查了下文档，optional是protoco 3.12版本加入的，如果参数设置成optional，执行时必须要带--experimental_allow_proto3_optional。但是我是3.15.8版本，执行时没有加上--experimental_allow_proto3_optional也没有报错</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我试了，后面的版本去掉了--experimental_allow_proto3_optional参数的限制。那就可以不用加这个选项了，代码我也更新了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-02 00:02:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/a9/93/2a26fe6e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>learner2021</span>
  </div>
  <div class="_2_QraFYR_0">哪里设置错误了？<br>[going@dev server]$ pwd<br>&#47;home&#47;going&#47;workspace&#47;golang&#47;src&#47;github.com&#47;marmotedu&#47;gopractise-demo&#47;apistyle&#47;greeter&#47;server<br>[going@dev server]$ go run main.go<br>main.go:8:2: no required module provides package github.com&#47;marmotedu&#47;gopractise-demo&#47;apistyle&#47;greeter&#47;helloworld: go.mod file not found in current directory or any parent directory; see &#39;go help modules&#39;<br>main.go:9:2: no required module provides package google.golang.org&#47;grpc: go.mod file not found in current directory or any parent directory; see &#39;go help modules&#39;</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 看看github.com&#47;marmotedu&#47;gopractise-demo目录下，有没有go.mod。<br><br>如果有建议执行下：go mod tidy</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-24 23:54:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2a/14/e2/f6f1627c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>顺势而为</span>
  </div>
  <div class="_2_QraFYR_0">建议作者的命令行，多点pwd，方便我们定位</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 好的，记住~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-28 17:12:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/34/84/27ecfcab.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>江湖夜雨十年灯</span>
  </div>
  <div class="_2_QraFYR_0">老师，我的proto文件是这样写的，go_package用的是我本地项目的目录：<br>option go_package = &quot;whw_scripts_stroage&#47;a_grpc_tests&#47;helloworld&quot;;<br>在helloworld目录中执行下面命令：<br>sudo protoc -I. --go_out=. .&#47;helloworld.proto <br>但是最后在helloworld目录中，与helloworld.proto文件同一个级别生成的不是helloworld.pb.go，而是：whw_scripts_stroage&#47;a_grpc_tests&#47;helloworld.pb.go<br>请问为什么生成的是目录加go文件呢？<br><br><br><br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: protoc编译器的设定哈，遵循就是了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-22 16:04:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>andox</span>
  </div>
  <div class="_2_QraFYR_0">grpc有推荐的服务治理方案吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: istio试试？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-14 14:17:12</div>
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
  <div class="_2_QraFYR_0">一文搞懂rpc，这个周末写一下demo。周日晚上再来打卡。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 强！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-07 00:52:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/a7/b2/274a4192.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>漂泊的小飘</span>
  </div>
  <div class="_2_QraFYR_0">老师，有两个问题请教下：<br>1，能否讲下四种模式的应用场景？<br>2，一般生成的文件需要放进git里面吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. 好像没有太固定的场景，具体根据需要选择就行了。<br><br>2. 生成的文件需要放在Git中。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-21 08:52:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/4c/fc/0e887697.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>kkgo</span>
  </div>
  <div class="_2_QraFYR_0">老师有对比过grpc和rpcx之间的性能，稳定性方面不? 看官方说明rpcx性能是grpc的2倍</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 没对比过，老哥可以对比下性能，分享出来</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-24 23:19:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/dd/09/feca820a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>helloworld</span>
  </div>
  <div class="_2_QraFYR_0">优秀👍</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-24 22:07:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/42/65/5bfd0a65.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>coming</span>
  </div>
  <div class="_2_QraFYR_0">飞哥, postman已支持grpc客户端了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-15 10:37:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/43/2d/af86d73f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>enjoylearning</span>
  </div>
  <div class="_2_QraFYR_0">不错，grpcgateway</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-26 13:21:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/29/97/39/1f5c6350.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>朱元彬🗿</span>
  </div>
  <div class="_2_QraFYR_0">生产环境中，使用服务提供者的接口，如果遇到接口更新的情况，要怎么跟服务提供者沟通更新协议呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 生产环境中，接口更新都是向前兼容的。如果不兼容，只能发邮件告知客户，对于VIP大客户，还要拉群人工通知，更新前确认客户业务改造完。后面就是更新时间到的时候进行接口变更。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-22 18:54:07</div>
  </div>
</div>
</div>
</li>
</ul>