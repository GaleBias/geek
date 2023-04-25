<audio title="09｜即学即练：构建一个Web服务就是这么简单" src="https://static001.geekbang.org/resource/audio/56/fd/5632ced763644c1e8553649c30a1c8fd.mp3" controls="controls"></audio> 
<p>你好，我是Tony Bai。</p><p>在入门篇前面的几节课中，我们已经从Go开发环境的安装，一路讲到了Go包的初始化次序与Go入口函数。讲解这些，不仅仅是因为它们是你学习Go语言的基础，同时我也想为你建立“手勤”的意识打好基础。</p><p>作为Go语言学习的“过来人”，学到这个阶段，我深知你心里都在跃跃欲试，想将前面学到的知识综合运用起来，实现一个属于自己的Go程序。但到目前为止，我们还没有开始Go基础语法的系统学习，你肯定会有一种“无米下炊”的感觉。</p><p>不用担心，我在这节课安排了一个实战小项目。在这个小项目里，我希望你不要困在各种语法里，而是先跟着我““照猫画虎”地写一遍、跑一次，感受Go项目的结构，体会Go语言的魅力。</p><h2>预热：最简单的HTTP服务</h2><p>在想选择以什么类型的项目的时候，我还颇费了一番脑筋。我查阅了<a href="https://go.dev/blog/survey2020-results">Go官方用户2020调查报告</a>，找到Go应用最广泛的领域调查结果图，如下所示：</p><p><img src="https://static001.geekbang.org/resource/image/9a/91/9ab73568ef659d75a313f3394a811491.png?wh=1194x1158" alt="图片"></p><p>我们看到，Go应用的前4个领域中，有两个都是Web服务相关的。一个是排在第一位的API/RPC服务，另一个是排在第四位的Web服务（返回html页面）。考虑到后续你把Go应用于Web服务领域的机会比较大，所以，在这节课我们就选择一个Web服务项目作为实战小项目。</p><!-- [[[read_end]]] --><p>不过在真正开始我们的实战小项目前，我们先来预热一下，做一下技术铺垫。我先来给你演示一下<strong>在Go中创建一个基于HTTP协议的Web服务是多么的简单</strong>。</p><p>这种简单又要归功于Go“面向工程”特性。在02讲介绍Go的设计哲学时，我们也说过，Go“面向工程”的特性，不仅体现在语言设计方面时刻考虑开发人员的体验，而且它还提供了完善的工具链和“自带电池”的标准库，这就使得Go程序大大减少了对外部第三方包的依赖。以开发Web服务为例，我们可以基于Go标准库提供的net/http包，轻松构建一个承载Web内容传输的HTTP服务。</p><p>下面，我们就来构建一个最简单的HTTP服务，这个服务的功能很简单，就是当收到一个HTTP请求后，给请求方返回包含“hello, world”数据的响应。</p><p>我们首先按下面步骤建立一个simple-http-server目录，并创建一个名为simple-http-server的Go Module：</p><pre><code class="language-plain">$mkdir simple-http-server
$cd simple-http-server
$go mod init simple-http-server
</code></pre><p>由于这个HTTP服务比较简单，我们采用最简项目布局，也就是在simple-http-server目录下创建一个main.go源文件：</p><pre><code class="language-go">  package main
 
  import "net/http"
  
  func main() {
      http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request){
          w.Write([]byte("hello, world"))
      })
      http.ListenAndServe(":8080", nil)
 }
</code></pre><p>这些代码就是一个最简单的HTTP服务的实现了。在这个实现中，我们只使用了Go标准库的http包。可能你现在对http包还不熟悉，但没有关系，你现在只需要大致了解上面代码的结构与原理就可以了。</p><p>这段代码里，你要注意两个重要的函数，一个是ListenAndServe，另一个是HandleFunc。</p><p>你会看到，代码的第9行，我们通过http包提供的ListenAndServe函数，建立起一个HTTP服务，这个服务监听本地的8080端口。客户端通过这个端口与服务建立连接，发送HTTP请求就可以得到相应的响应结果。</p><p>那么服务端是如何处理客户端发送的请求的呢？我们看上面代码中的第6行。在这一行中，我们为这个服务设置了一个处理函数。这个函数的函数原型是这样的：</p><pre><code class="language-go">func(w http.ResponseWriter, r *http.Request)
</code></pre><p>这个函数里有两个参数，w和r。第二个参数r代表来自客户端的HTTP请求，第一个参数w则是用来操作返回给客户端的应答的，基于http包实现的HTTP服务的处理函数都要符合这一原型。</p><p>你也发现了，在这个例子中，所有来自客户端的请求，无论请求的URI路径（RequestURI）是什么，请求都会被我们设置的处理函数处理。为什么会这样呢？</p><p>这是因为，我们通过http.HandleFunc设置这个处理函数时，传入的模式字符串为“/”。HTTP服务器在收到请求后，会将请求中的URI路径与设置的模式字符串进行<strong>最长前缀匹配</strong>，并执行匹配到的模式字符串所对应的处理函数。在这个例子中，我们仅设置了“/”这一个模式字符串，并且所有请求的URI都能与之匹配，自然所有请求都会被我们设置的处理函数处理。</p><p>接着，我们再来编译运行一下这个程序，直观感受一下HTTP服务处理请求的过程。我们首先按下面步骤来编译并运行这个程序：</p><pre><code class="language-go">$cd simple-http-server
$go build
$./simple-http-server
</code></pre><p>接下来，我们用curl命令行工具模拟客户端，向上述服务建立连接并发送http请求：</p><pre><code class="language-go">$curl localhost:8080/
hello, world
</code></pre><p>我们看到，curl成功得到了http服务返回的“hello, world”响应数据。到此，我们的HTTP服务就构建成功了。</p><p>当然了，真实世界的Web服务不可能像上述例子这么简单，这仅仅是一个“预热”。我想让你知道，使用Go构建Web服务是非常容易的。并且，这样的预热也能让你初步了解实现代码的结构，先有一个技术铺垫。</p><p>下面我们就进入这节课的实战小项目，一个更接近于真实世界情况的<strong>图书管理API服务</strong>。</p><h2>图书管理API服务</h2><p>首先，我们先来明确一下我们的业务逻辑。</p><p>在这个实战小项目中，我们模拟的是真实世界的一个书店的图书管理后端服务。这个服务为平台前端以及其他客户端，提供针对图书的CRUD（创建、检索、更新与删除）的基于HTTP协议的API。API采用典型的RESTful风格设计，这个服务提供的API集合如下：</p><p><img src="https://static001.geekbang.org/resource/image/99/51/99717b62f7553e1a5139edcf2ac03b51.jpg?wh=1980x788" alt=""></p><p>这个API服务的逻辑并不复杂。简单来说，我们通过id来唯一标识一本书，对于图书来说，这个id通常是ISBN号。至于客户端和服务端中请求与响应的数据，我们采用放在HTTP协议包体（Body）中的Json格式数据来承载。</p><p>业务逻辑是不是很简单啊？下面我们就直接开始创建这个项目。</p><h3>项目建立与布局设计</h3><p>我们按照下面步骤创建一个名为bookstore的Go项目并创建对应的Go Module：</p><pre><code class="language-go">$mkdir bookstore
$cd bookstore
$go mod init bookstore
go: creating new go.mod: module bookstore
</code></pre><p>通过上面的业务逻辑说明，我们可以把这个服务大体拆分为两大部分，一部分是HTTP服务器，用来对外提供API服务；另一部分是图书数据的存储模块，所有的图书数据均存储在这里。</p><p>同时，这是一个以构建可执行程序为目的的Go项目，我们参考Go项目布局标准一讲中的项目布局，把这个项目的结构布局设计成这样：</p><pre><code class="language-go">├── cmd/
│&nbsp;&nbsp; └── bookstore/         // 放置bookstore main包源码
│&nbsp;&nbsp;     └── main.go
├── go.mod                 // module bookstore的go.mod
├── go.sum
├── internal/              // 存放项目内部包的目录
│&nbsp;&nbsp; └── store/
│&nbsp;&nbsp;     └── memstore.go     
├── server/                // HTTP服务器模块
│&nbsp;&nbsp; ├── middleware/
│&nbsp;&nbsp; │&nbsp;&nbsp; └── middleware.go
│&nbsp;&nbsp; └── server.go          
└── store/                 // 图书数据存储模块
    ├── factory/
    │&nbsp;&nbsp; └── factory.go
    └── store.go
</code></pre><p>现在，我们既给出了这个项目的结构布局，也给出了这个项目最终实现的源码文件分布情况。下面我们就从main包开始，自上而下逐一看看这个项目的模块设计与实现。</p><h3>项目main包</h3><p>main包是主要包，为了搞清楚各个模块之间的关系，我在这里给出了main包的实现逻辑图：</p><p><img src="https://static001.geekbang.org/resource/image/5e/19/5e8ee50b67a4229210b12afb94f55a19.jpg?wh=1980x1080" alt=""></p><p>同时，我也列出了main包（main.go）的所有代码，你可以先花几分钟看一下：</p><pre><code class="language-go">  package main
 
  import (
      _ "bookstore/internal/store"
      "bookstore/server"
      "bookstore/store/factory"
      "context"
      "log"
      "os"
      "os/signal"
      "syscall"
      "time"
 )
 
 func main() {
     s, err := factory.New("mem") // 创建图书数据存储模块实例
     if err != nil {
         panic(err)
     }
 
     srv := server.NewBookStoreServer(":8080", s) // 创建http服务实例
 
     errChan, err := srv.ListenAndServe() // 运行http服务
     if err != nil {
         log.Println("web server start failed:", err)
         return
     }
     log.Println("web server start ok")
 
     c := make(chan os.Signal, 1)
     signal.Notify(c, syscall.SIGINT, syscall.SIGTERM)
 
     select { // 监视来自errChan以及c的事件
     case err = &lt;-errChan:
         log.Println("web server run failed:", err)
         return
     case &lt;-c:
         log.Println("bookstore program is exiting...")
         ctx, cf := context.WithTimeout(context.Background(), time.Second)
         defer cf()
         err = srv.Shutdown(ctx) // 优雅关闭http服务实例
     }
 
     if err != nil {
         log.Println("bookstore program exit error:", err)
         return
     }
     log.Println("bookstore program exit ok")
 }
</code></pre><p>在Go中，main包不仅包含了整个程序的入口，它还是整个程序中主要模块初始化与组装的场所。那对应在我们这个程序中，主要模块就是第16行的创建图书存储模块实例，以及第21行创建HTTP服务模块实例。而且，你还要注意的是，第21行创建HTTP服务模块实例的时候，我们把图书数据存储实例s作为参数，传递给了NewBookStoreServer函数。这两个实例的创建原理，我们等会再来细细探讨。</p><p>这里，我们重点来看main函数的后半部分（第30行~第42行），这里表示的是，我们通过监视系统信号实现了http服务实例的优雅退出。</p><p>所谓优雅退出，指的就是程序有机会等待其他的事情处理完再退出。比如尚未完成的事务处理、清理资源（比如关闭文件描述符、关闭socket）、保存必要中间状态、内存数据持久化落盘，等等。如果你经常用Go来编写http服务，那么http服务如何优雅退出，就是你经常要考虑的问题。</p><p>在这个问题的具体实现上，我们通过signal包的Notify捕获了SIGINT、SIGTERM这两个系统信号。这样，当这两个信号中的任何一个触发时，我们的http服务实例都有机会在退出前做一些清理工作。</p><p>然后，我们再使用http服务实例（srv）自身提供的Shutdown方法，来实现http服务实例内部的退出清理工作，包括：立即关闭所有listener、关闭所有空闲的连接、等待处于活动状态的连接处理完毕，等等。当http服务实例的清理工作完成后，我们整个程序就可以正常退出了。</p><p>接下来，我们再重点看看构成bookstore程序的两个主要模块：图书数据存储模块与HTTP服务模块的实现。我们按照main函数中的初始化顺序，先来看看图书数据存储模块。</p><h3>图书数据存储模块（store)</h3><p>图书数据存储模块的职责很清晰，就是用来<strong>存储整个bookstore的图书数据</strong>的。图书数据存储有很多种实现方式，最简单的方式莫过于在内存中创建一个map，以图书id作为key，来保存图书信息，我们在这一讲中也会采用这种方式。但如果我们要考虑上生产环境，数据要进行持久化，那么最实际的方式就是通过Nosql数据库甚至是关系型数据库，实现对图书数据的存储与管理。</p><p>考虑到对多种存储实现方式的支持，我们将针对图书的有限种存储操作，放置在一个接口类型Store中，如下源码所示：</p><pre><code class="language-go">// store/store.go

 type Book struct {
     Id      string   `json:"id"`      // 图书ISBN ID
     Name    string   `json:"name"`    // 图书名称
     Authors []string `json:"authors"` // 图书作者
     Press   string   `json:"press"`   // 出版社
 }
 
 type Store interface {
     Create(*Book) error        // 创建一个新图书条目
     Update(*Book) error        // 更新某图书条目
     Get(string) (Book, error)  // 获取某图书信息
     GetAll() ([]Book, error)   // 获取所有图书信息
     Delete(string) error       // 删除某图书条目
 }
</code></pre><p>这里，我们建立了一个对应图书条目的抽象数据类型Book，以及针对Book存取的接口类型Store。这样，对于想要进行图书数据操作的一方来说，他只需要得到一个满足Store接口的实例，就可以实现对图书数据的存储操作了，不用再关心图书数据究竟采用了何种存储方式。这就实现了图书存储操作与底层图书数据存储方式的解耦。而且，这种面向接口编程也是Go组合设计哲学的一个重要体现。</p><p>那我们具体如何创建一个满足Store接口的实例呢？我们可以参考《设计模式》提供的多种创建型模式，选择一种Go风格的工厂模式（创建型模式的一种）来实现满足Store接口实例的创建。我们看一下store/factory包的源码：</p><pre><code class="language-go">// store/factory/factory.go

 var (
     providersMu sync.RWMutex
     providers   = make(map[string]store.Store)
 )
 
 func Register(name string, p store.Store) {
     providersMu.Lock()
    defer providersMu.Unlock()
     if p == nil {
         panic("store: Register provider is nil")
     }
 
     if _, dup := providers[name]; dup {
         panic("store: Register called twice for provider " + name)
     }
     providers[name] = p
 }
 
 func New(providerName string) (store.Store, error) {
     providersMu.RLock()
     p, ok := providers[providerName]
     providersMu.RUnlock()
     if !ok {
         return nil, fmt.Errorf("store: unknown provider %s", providerName)
     }
 
     return p, nil
 }
</code></pre><p>这段代码实际上是效仿了Go标准库的database/sql包采用的方式，factory包采用了一个map类型数据，对工厂可以“生产”的、满足Store接口的实例类型进行管理。factory包还提供了Register函数，让各个实现Store接口的类型可以把自己“注册”到工厂中来。</p><p>一旦注册成功，factory包就可以“生产”出这种满足Store接口的类型实例。而依赖Store接口的使用方，只需要调用factory包的New函数，再传入期望使用的图书存储实现的名称，就可以得到对应的类型实例了。</p><p>在项目的internal/store目录下，我们还提供了一个基于内存map的Store接口的实现，我们具体看一下这个实现是怎么自注册到factory包中的：</p><pre><code class="language-go">// internal/store/memstore.go

 package store
  
 import (
     mystore "bookstore/store"
     factory "bookstore/store/factory"
     "sync"
 )
  
 func init() {
     factory.Register("mem", &amp;MemStore{
         books: make(map[string]*mystore.Book),
     })
 }
 
 type MemStore struct {
     sync.RWMutex
     books map[string]*mystore.Book
 }
</code></pre><p>从memstore的代码来看，它是在包的init函数中调用factory包提供的Register函数，把自己的实例以“mem”的名称注册到factory中的。这样做有一个好处，依赖Store接口进行图书数据管理的一方，只要导入internal/store这个包，就可以自动完成注册动作了。</p><p>理解了这个之后，我们再看下面main包中，创建图书数据存储模块实例时采用的代码，是不是就豁然开朗了？</p><pre><code class="language-go">import (
    ... ...
    _ "bookstore/internal/store" // internal/store将自身注册到factory中
)

func main() {
    s, err := factory.New("mem") // 创建名为"mem"的图书数据存储模块实例
    if err != nil {
        panic(err)
    }
    ... ...
}
</code></pre><p>至于memstore.go中图书数据存储的具体逻辑，就比较简单了，我这里就不详细分析了，你课后自己阅读一下吧。</p><p>接着，我们再来看看bookstore程序的另外一个重要模块：HTTP服务模块。</p><h3>HTTP服务模块（server）</h3><p>HTTP服务模块的职责是<strong>对外提供HTTP API服务，处理来自客户端的各种请求，并通过Store接口实例执行针对图书数据的相关操作</strong>。这里，我们抽象处理一个server包，这个包中定义了一个BookStoreServer类型如下：</p><pre><code class="language-go">// server/server.go

 type BookStoreServer struct {
     s   store.Store
     srv *http.Server
 }
</code></pre><p>我们看到，这个类型实质上就是一个标准库的http.Server，并且组合了来自store.Store接口的能力。server包提供了NewBookStoreServer函数，用来创建一个BookStoreServer类型实例：</p><pre><code class="language-go">// server/server.go

 func NewBookStoreServer(addr string, s store.Store) *BookStoreServer {
     srv := &amp;BookStoreServer{
         s: s,
         srv: &amp;http.Server{
             Addr: addr,
         },
     }
 
     router := mux.NewRouter()
     router.HandleFunc("/book", srv.createBookHandler).Methods("POST")
     router.HandleFunc("/book/{id}", srv.updateBookHandler).Methods("POST")
     router.HandleFunc("/book/{id}", srv.getBookHandler).Methods("GET")
     router.HandleFunc("/book", srv.getAllBooksHandler).Methods("GET")
     router.HandleFunc("/book/{id}", srv.delBookHandler).Methods("DELETE")
 
     srv.srv.Handler = middleware.Logging(middleware.Validating(router))
     return srv
 }
</code></pre><p>我们看到函数NewBookStoreServer接受两个参数，一个是HTTP服务监听的服务地址，另外一个是实现了store.Store接口的类型实例。这种函数原型的设计是Go语言的一种惯用设计方法，也就是接受一个接口类型参数，返回一个具体类型。返回的具体类型组合了传入的接口类型的能力。</p><p>这个时候，和前面预热时实现的简单http服务一样，我们还需为HTTP服务器设置请求的处理函数。</p><p>由于这个服务请求URI的模式字符串比较复杂，标准库http包内置的URI路径模式匹配器（ServeMux，也称为路由管理器）不能满足我们的需求，因此在这里，我们需要借助一个第三方包github.com/gorilla/mux来实现我们的需求。</p><p>在上面代码的第11行到第16行，我们针对不同URI路径模式设置了不同的处理函数。我们以createBookHandler和getBookHandler为例来看看这些处理函数的实现：</p><pre><code class="language-go">// server/server.go

  func (bs *BookStoreServer) createBookHandler(w http.ResponseWriter, req *http.Request) {
      dec := json.NewDecoder(req.Body)
      var book store.Book
      if err := dec.Decode(&amp;book); err != nil {
          http.Error(w, err.Error(), http.StatusBadRequest)
          return
      }
  
      if err := bs.s.Create(&amp;book); err != nil {
          http.Error(w, err.Error(), http.StatusBadRequest)
          return
      }
  }

  func (bs *BookStoreServer) getBookHandler(w http.ResponseWriter, req *http.Request) {
      id, ok := mux.Vars(req)["id"]
      if !ok {
          http.Error(w, "no id found in request", http.StatusBadRequest)
          return
      }
  
     book, err := bs.s.Get(id)
     if err != nil {
         http.Error(w, err.Error(), http.StatusBadRequest)
         return
     }
 
     response(w, book)
 }

 func response(w http.ResponseWriter, v interface{}) {
     data, err := json.Marshal(v)
     if err != nil {
         http.Error(w, err.Error(), http.StatusInternalServerError)
         return
     }
     w.Header().Set("Content-Type", "application/json")
     w.Write(data)
 }
</code></pre><p>这些处理函数的实现都大同小异，都是先获取http请求包体数据，然后通过标准库json包将这些数据，解码（decode）为我们需要的store.Book结构体实例，再通过Store接口对图书数据进行存储操作。如果我们是获取图书数据的请求，那么处理函数将通过response函数，把取出的图书数据编码到http响应的包体中，并返回给客户端。</p><p>然后，在NewBookStoreServer函数实现的尾部，我们还看到了这样一行代码：</p><pre><code class="language-go">     srv.srv.Handler = middleware.Logging(middleware.Validating(router))
</code></pre><p>这行代码的意思是说，我们在router的外围包裹了两层middleware。什么是middleware呢？对于我们的上下文来说，这些middleware就是一些通用的http处理函数。我们看一下这里的两个middleware，也就是Logging与Validating函数的实现：</p><pre><code class="language-go">// server/middleware/middleware.go

  func Logging(next http.Handler) http.Handler {
     return http.HandlerFunc(func(w http.ResponseWriter, req *http.Request) {
         log.Printf("recv a %s request from %s", req.Method, req.RemoteAddr)
         next.ServeHTTP(w, req)
     })
 }
 
 func Validating(next http.Handler) http.Handler {
     return http.HandlerFunc(func(w http.ResponseWriter, req *http.Request) {
         contentType := req.Header.Get("Content-Type")
         mediatype, _, err := mime.ParseMediaType(contentType)
         if err != nil {
             http.Error(w, err.Error(), http.StatusBadRequest)
             return
         }
         if mediatype != "application/json" {
             http.Error(w, "invalid Content-Type", http.StatusUnsupportedMediaType)
             return
         }
         next.ServeHTTP(w, req)
     })
 }
</code></pre><p>我们看到，Logging函数主要用来输出每个到达的HTTP请求的一些概要信息，而Validating则会对每个http请求的头部进行检查，检查Content-Type头字段所表示的媒体类型是否为application/json。这些通用的middleware函数，会被串联到每个真正的处理函数之前，避免我们在每个处理函数中重复实现这些逻辑。</p><p>创建完BookStoreServer实例后，我们就可以调用其ListenAndServe方法运行这个http服务了，显然这个方法的名字是仿效http.Server类型的同名方法，我们的实现是这样的：</p><pre><code class="language-go">// server/server.go

 func (bs *BookStoreServer) ListenAndServe() (&lt;-chan error, error) {
     var err error
     errChan := make(chan error)
     go func() {
         err = bs.srv.ListenAndServe()
         errChan &lt;- err
     }()
 
     select {
     case err = &lt;-errChan:
         return nil, err
     case &lt;-time.After(time.Second):
         return errChan, nil
     }
 }
</code></pre><p>我们看到，这个函数把BookStoreServer内部的http.Server的运行，放置到一个单独的轻量级线程Goroutine中。这是因为，http.Server.ListenAndServe会阻塞代码的继续运行，如果不把它放在单独的Goroutine中，后面的代码将无法得到执行。</p><p>为了检测到http.Server.ListenAndServe的运行状态，我们再通过一个channel（位于第5行的errChan），在新创建的Goroutine与主Goroutine之间建立的通信渠道。通过这个渠道，这样我们能及时得到http server的运行状态。</p><h3>编译、运行与验证</h3><p>到这里，bookstore项目的大部分重要代码我们都分析了一遍，是时候将程序跑起来看看了。</p><p>不过，因为我们在程序中引入了一个第三方依赖包，所以在构建项目之前，我们需要执行下面这个命令，让Go命令自动分析依赖项和版本，并更新go.mod：</p><pre><code class="language-go">$go mod tidy
go: finding module for package github.com/gorilla/mux
go: found github.com/gorilla/mux in github.com/gorilla/mux v1.8.0
</code></pre><p>完成后，我们就可以按下面的步骤来构建并执行bookstore了：</p><pre><code class="language-go">$go build bookstore/cmd/bookstore
$./bookstore
2021/10/05 16:08:36 web server start ok
</code></pre><p>如果你看到上面这个输出的日志，说明我们的程序启动成功了。</p><p>现在，我们就可以像前面一样使用curl命令行工具，模仿客户端向bookstore服务发起请求了，比如创建一个新书条目：</p><pre><code class="language-go">$curl -X POST -H "Content-Type:application/json" -d '{"id": "978-7-111-55842-2", "name": "The Go Programming Language", "authors":["Alan A.A.Donovan", "Brian W. Kergnighan"],"press": "Pearson Education"}' localhost:8080/book
</code></pre><p>此时服务端会输出如下日志，表明我们的bookstore服务收到了客户端请求。</p><pre><code class="language-go">2021/10/05 16:09:10 recv a POST request from [::1]:58021
</code></pre><p>接下来，我们再来获取一下这本书的信息：</p><pre><code class="language-go">$curl -X GET -H "Content-Type:application/json" localhost:8080/book/978-7-111-55842-2
{"id":"978-7-111-55842-2","name":"The Go Programming Language","authors":["Alan A.A.Donovan","Brian W. Kergnighan"],"press":"Pearson Education"}
</code></pre><p>我们看到curl得到的响应与我们预期的是一致的。</p><p>好了，我们不再进一步验证了，你课后还可以自行编译、执行并验证。</p><h2>小结</h2><p>到这里，我们就完成了我们第一个实战小项目，不知道你感觉如何呢？</p><p>在这一讲中，我们带你用Go语言构建了一个最简单的HTTP服务，以及一个接近真实的图书管理API服务。在整个实战小项目的实现过程中，你也能初步学习到Go编码时常用的一些惯用法，比如基于接口的组合、类似database/sql所使用的惯用创建模式，等等。</p><p>通过这节课的学习，你是否体会到了Go语言的魅力了呢？是否察觉到Go编码与其他主流语言不同的风格了呢？其实不论你的理解程度有多少，都不重要。只要你能“照猫画虎”地将上面的程序自己编写一遍，构建、运行起来并验证一遍，就算是完美达成这一讲的目标了。</p><p>你在这个过程肯定会有各种各样的问题，但没关系，这些问题会成为你继续向下学习Go的动力。毕竟，<strong>带着问题的学习，能让你的学习过程更有的放矢、更高效</strong>。</p><h2>思考题</h2><p>如果你完成了今天的代码，觉得自己学有余力，可以再挑战一下，不妨试试基于nosql数据库，我们怎么实现一个新store.Store接口的实现吧？</p><p>欢迎把这节课分享给更多对Go语言感兴趣的朋友。我是Tony Bai，我们下节课见。</p><h2>资源链接</h2><p><a href="https://github.com/bigwhite/publication/tree/master/column/timegeek/go-first-course/09/bookstore">这节课的图书管理项目的完整源码在这里！</a></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/27/52/40/db9b0eb2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>自由</span>
  </div>
  <div class="_2_QraFYR_0">Tony Bai 老师的这篇“简单”的 Web 代码构建，细节特别多，足矣看出功力深厚。例如锁的选型、工程化、为什么用锁、面向接口、设计模式、通过 Channel 通信而不是共享内存、panic 和 error 的区别。我认为这样的代码，是一个完美的开始。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-05 11:34:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2b/92/af/ad02ae4b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>扣剑书生</span>
  </div>
  <div class="_2_QraFYR_0">store.go提供了 图书 和 接口的模板<br>factory 用于生产 接口实例<br>memstore.go 用于具体实现一个接口实例，实现其方法，并把样例发送到工厂<br>server.go 用于把路由和接口的方法对接起来</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-30 19:06:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/29/8b/7c/fe7b1bf7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>尧九之阳</span>
  </div>
  <div class="_2_QraFYR_0">Go现在有流行的web服务器框架么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: go最初有很多web框架，经过多年演进，目前gin这个web框架似乎成为了go社区的首选。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-10 00:35:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/59/21/d2efde18.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>布凡</span>
  </div>
  <div class="_2_QraFYR_0">终于实验成功：<br>以下几个点需注意：<br>1、项目应放到 gopath货goroot相关的目录下，否则本地包的引用会报错，报错信息如下：&quot;could not import errors (cannot find package &quot;errors&quot; in any of c:\go\src\errors (from $GOROOT)...)&quot;<br>2、如果是复制的代码应该注意文件格式，可能报错&quot;package main: read unexpected NUL in input&quot;</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 项目都是在mac&#47;linux上做的。但理论上windows可以直接使用。你提到的项目放在gopath或goroot下是不必要的。支持go module模式后，如果启用了go module构建模式，放在哪个目录下都是可以的。<br><br>另外项目代码放在了github上，在文章末尾有链接。可直接clone后使用。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-04 15:23:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/fe/f8/3e8de9b1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>猫饼</span>
  </div>
  <div class="_2_QraFYR_0">我果然还是太菜了 我开始看不懂了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 先“照猫画虎”吧。一旦涉及此类实战小项目，必然会有没有系统学习的内容。后面学习新语法时，可以回过头来复习。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-02 17:53:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/49/4e/8798cd01.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>顷</span>
  </div>
  <div class="_2_QraFYR_0">老师的这个示例，麻雀虽小，五脏俱全。不过main.go里有一点疑问：http.Server.Shutdown(ctx)被调用后，http.Server.ListenAndServe()方法马上会返回error吧，按照实例代码里的写法，接收到中断信号后，马上调用Shutdown方法，此时errChan会返回ErrServerClosed，select逻辑走完，main方法就退出了，而go的http包里示意了我们需要确保shutdown调用后，整个代码不能马上退出，请老师解惑。。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 好问题！<br><br>首先errChan并不是等shutdown之后 listenAndServe报错的。而是等运行国产中，listenandserve一旦报错，程序可以退出。<br><br>接下来，我们再说说优雅退出。按照你的描述，我翻了一下net.http的文档：<br><br>“Shutdown gracefully shuts down the server without interrupting any active connections. Shutdown works by first closing all open listeners, then closing all idle connections, and then waiting indefinitely for connections to return to idle and then shut down. If the provided context expires before the shutdown is complete, Shutdown returns the context&#39;s error, otherwise it returns any error returned from closing the Server&#39;s underlying Listener(s).”<br><br>我的理解是调用Shutdown后，ListenAndServe()方法的确会马上返回，不过这个无所谓了,listenandserver只是将listen端口关闭，不接受新连接了。<br><br>真正等待存量连接处理完毕的是shutdown方法，shutdown方法调用后，不是马上退出的哦。shutdown方法有一个ctx，专栏中传入了一个timeout ctx，时间为1s。即等待1s。关键就在这里。在生产环境，我们可能不能无限期等待程序退出，我们会设置一个timeout。如果shutdown在1s内完成等待，成功退出，这是最理想的。如果shutdown没有在1s内完成等待，即存量连接还没有处理完，那么按照我们的约定，我们也不能再等了。<br><br>当然这里只是demo，使用的timeout=1s。在生产中，你可以根据业务的类型以及经验值来等待。甚至可以进行无限等待，这完全取决于你的系统。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-21 22:16:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2b/26/bc/18314557.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>高雪斌</span>
  </div>
  <div class="_2_QraFYR_0">C++转Go, Go语言设计的确实很简洁、精妙。Tony Bai老师从语言结构开始讲的风格非常受用，比通常的从语法开始讲的形式要好，虽然一些代码现在看不明白，但这样带着问题学习语法更容易理解。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-01 18:55:30</div>
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
  <div class="_2_QraFYR_0">老师讲的太赞了<br>市面上 讲 go 接口思想很好， 推荐组合思想。  但是都光说不练，看懂但学不会。<br>今天老师 给的这个例子 真实，简短 且 把接口和组合的 思想都 做了出来， 太牛了。<br>这一篇 需要实践而且需要反复揣摩。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-01 12:06:08</div>
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
  <div class="_2_QraFYR_0">感谢 Tony Bai 老师这样由浅入深，并且尽可能贴近实战的讲解。<br><br>有以下困惑，麻烦解忧。<br><br>1. 对于一个module中，相同的包名，go内部是根据文件位置的不同，加以区分的吧？ 例如：bookstore&#47;store 和 bookstore&#47;internal&#47;store 这两个 package 同名，但是文件的路径不同，所以并不会有什么问题。如果在同一个package中，导入另一个同名的package，最佳实践是取个别名，我的理解没错吧？<br><br>2. 另外在 memstore.go 中的 包导入语句中： factory &quot;bookstore&#47;store&#47;factory&quot;，这个 默认使用的是factory名字，不需要再另起名字为factory吧？<br><br>3. 这门课中的知识和你在另外的一个平台的《改善Go语言编程质量的50个有效实践》的内容重合度大吗？ 精力有限，如果重合度大，就专心看这个就好了。<br><br>4. 这门课会讲讲Go写RPC服务方面的知识吗？ 这个在生产中挺常用的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 问题1. 你的理解没有错。<br>问题2. 可以不用另起，记得好像有些工具在格式化时会自动填上。<br>3. 定位不同，这个是入门专栏。那个进阶专栏。有很小部分有相似的地方，但讲解方式都做了调整和优化。建议学完这个专栏后，再去看看那个专栏。<br>4. rpc封装的很深，如果仅是使用的话，没啥东西。这个专栏更多聚焦于go语法以及尽量使用标准库的东西做一些实用程序和项目。<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-02 17:57:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/6d/cf/ec335526.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jc9090kkk</span>
  </div>
  <div class="_2_QraFYR_0">这节课的项目内容理解起来不困难，但是对于实际的生产项目而言，尤其是对第三方的中间件有依赖的前提下，大都会针对的将配置单独存储为配置文件以方便维护，我想问下老师go项目针对配置文件有什么最佳实践方式吗? 我自己本地写了一个小项目，用的是以下方式来配置MySQL连接的，我总觉得不太优雅，大多项目会根据开发环境的不同选取不同的配置文件作为配置项加载，但是如果通过硬编码的方式添加进去会让项目变得很奇怪。。。<br><br>package config<br><br>&#47;&#47; GetDBConfig 数据库配置<br>func GetDBConfig() map[string]string {<br>	&#47;&#47; 初始化数据库配置map<br>	dbConfig := make(map[string]string)<br><br>	dbConfig[&quot;DB_HOST&quot;] = &quot;127.0.0.1&quot;<br>	dbConfig[&quot;DB_PORT&quot;] = &quot;3306&quot;<br>	dbConfig[&quot;DB_NAME&quot;] = &quot;test&quot;<br>	dbConfig[&quot;DB_USER&quot;] = &quot;root&quot;<br>	dbConfig[&quot;DB_PWD&quot;] = &quot;123456&quot;<br>	dbConfig[&quot;DB_CHARSET&quot;] = &quot;utf8&quot;<br><br>	&#47;&#47; 连接池最大连接数<br>	dbConfig[&quot;DB_MAX_OPEN_CONNECTS&quot;] = &quot;20&quot;<br>	&#47;&#47; 连接池最大空闲数<br>	dbConfig[&quot;DB_MAX_IDLE_CONNECTS&quot;] = &quot;10&quot;<br>	&#47;&#47; 连接池链接最长生命周期<br>	dbConfig[&quot;DB_MAX_LIFETIME_CONNECTS&quot;] = &quot;7200&quot;<br><br>	return dbConfig<br>}</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 生产项目一般不会将配置“硬编码”进去的。一般而言，配置项会存储在配置文件中，也有通过命令行参数传入程序的，也可以通过环境变量传入程序。Go标准库没有内置配置读写框架，目前go社区应用较多的第三方库是Go核心团队成员开发的viper(github.com&#47;spf13&#47;viper)。<br><br>对于一些更大的平台，常常有很多服务，这些服务的配置一般存储在专门的配置中心中，由配置中心管理与分发。<br><br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-01 10:43:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/23/52/66/3e4d4846.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>includestdio.h</span>
  </div>
  <div class="_2_QraFYR_0">作为基本0基础学go，这一节完全没看懂-，- 把后面的基础篇看完回来再试试吧</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这一讲开头我就说了：“在这个小项目里，我希望你不要困在各种语法里，而是先跟着我““照猫画虎”地写一遍、跑一次，感受 Go 项目的结构，体会 Go 语言的魅力。” 😁  不用担心，往后学一学之后再来复习。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-28 22:53:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2a/e9/dc/cc05ebc7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小明</span>
  </div>
  <div class="_2_QraFYR_0">能在gitee 上也放一份吗？github现在已经没那么友好了<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good idea。已创建https:&#47;&#47;gitee.com&#47;bigwhite&#47;publication&#47;tree&#47;master&#47;column&#47;timegeek&#47;go-first-course 供下载。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-20 02:26:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/b5/e6/c67f12bd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>左耳朵东</span>
  </div>
  <div class="_2_QraFYR_0">server&#47;server.go 文件中 select 那里，第二个 case 的意思是定时 1 秒后就会触发，从而执行后面的 return，为什么服务没有终止一直在运行呢？麻烦老师解答一下</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你说的是func (bs *BookStoreServer) ListenAndServe这个方法吧？这个函数等待1s后返回啊。但它启动的子goroutine依旧在运行的。子goroutine中bs.srv.ListenAndServe一直在工作，并提供服务。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-03 10:30:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/61/3f/2ce014e7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>snow</span>
  </div>
  <div class="_2_QraFYR_0">我看这里使用了mux包，我只用过gin包，请问这两个老师更喜欢哪个？以及这里选择mux的原因。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 原则就是如果标准库中的mux可以满足，尽量不引用第三方包。如果标准库无法满足，尽量引用规模较小的包。gin是更大的web框架。除了mux，还有很多其他功能。如果不是项目必需，使用更“专一”的包可能更好。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-05 11:58:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/22/86/3fe860c9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>邰洋一</span>
  </div>
  <div class="_2_QraFYR_0">老师，采用Restful规范，更新一条图书条目  http方法采用PUT，当然post也是可以的，put book&#47;id，是我太限定自己了吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你的建议很好。不过一个例子而已，别太计较：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-05 11:15:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/e9/7b/4e0ede2a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>莫名四下里</span>
  </div>
  <div class="_2_QraFYR_0">Tony Bai 老师<br>$ go build bookstore&#47;cmd&#47;bookstore&#47;        <br>package bookstore&#47;cmd&#47;bookstore is not in GOROOT (&#47;usr&#47;local&#47;go&#47;src&#47;bookstore&#47;cmd&#47;bookstore)<br>无法构建   <br><br>配置 GOROOT=&quot;&#47;usr&#47;local&#47;go&quot;<br>报错  &#47;usr&#47;local&#47;go&#47;src&#47;bookstore&#47;cmd&#47;bookstore</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 进入bookstore目录，go build .&#47;cmd&#47;bookstore或go build bookstore&#47;cmd&#47;bookstore</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-25 16:20:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/ce/d7/5315f6ce.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>不负青春不负己🤘</span>
  </div>
  <div class="_2_QraFYR_0">突然发现一个好玩的github 功能，即将.com 变成dev 会变成 web 版的<br>https:&#47;&#47;github.dev&#47;bigwhite&#47;publication&#47;tree&#47;master&#47;column&#47;timegeek&#47;go-first-course&#47;09&#47;bookstore</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-02 22:43:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2a/e9/dc/cc05ebc7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小明</span>
  </div>
  <div class="_2_QraFYR_0">看了两遍代码，能跑起来，但是只吃透了百分之三十，好着急啊  memstore.go   没看懂</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 很正常，这个例子并非让你完全理解，学到第09讲要是完全理解了，那后面就不用学习了:)</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-28 17:30:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/24/42/3f/cb0d8a8d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_zkg</span>
  </div>
  <div class="_2_QraFYR_0">我是mac环境，按照老师上面的步骤走下来，在go build bookstore&#47;cmd&#47;bookstore的时候<br>报错package bookstore&#47;cmd&#47;bookstore is not in GOROOT (&#47;usr&#47;local&#47;go&#47;src&#47;bookstore&#47;cmd&#47;bookstore)<br>我已经设置了GO111MODULE=&quot;on&quot;<br>用goland开发的<br>想请老师帮忙解决一下</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 问一下，你的goland版本是？是否支持go module编译？GO111MODULE=&quot;on&quot;对于goland是否生效了？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-18 17:52:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/1f/14/57cb7926.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>FECoding</span>
  </div>
  <div class="_2_QraFYR_0">providersMu.RLock()<br>我的goland里面报错啊，没有这个方法呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 方法肯定是有的啊。<br><br>$go doc sync.RWMutex.RLock<br>package sync &#47;&#47; import &quot;sync&quot;<br><br>func (rw *RWMutex) RLock()<br>    RLock locks rw for reading.<br><br>    It should not be used for recursive read locking; a blocked Lock call<br>    excludes new readers from acquiring the lock. See the documentation on the<br>    RWMutex type.<br><br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-23 11:38:25</div>
  </div>
</div>
</div>
</li>
</ul>