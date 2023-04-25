<audio title="39 _ Redis 6.0的新特性：多线程、客户端缓存与安全" src="https://static001.geekbang.org/resource/audio/a9/7b/a9f6891ca274e7155b4826beceebc57b.mp3" controls="controls"></audio> 
<p>你好，我是蒋德钧。</p><p>Redis官方在今年5月份正式推出了6.0版本，这个版本中有很多的新特性。所以，6.0刚刚推出，就受到了业界的广泛关注。</p><p>所以，在课程的最后，我特意安排了这节课，想来和你聊聊Redis 6.0中的几个关键新特性，分别是面向网络处理的多IO线程、客户端缓存、细粒度的权限控制，以及RESP 3协议的使用。</p><p>其中，面向网络处理的多IO线程可以提高网络请求处理的速度，而客户端缓存可以让应用直接在客户端本地读取数据，这两个特性可以提升Redis的性能。除此之外，细粒度权限控制让Redis可以按照命令粒度控制不同用户的访问权限，加强了Redis的安全保护。RESP 3协议则增强客户端的功能，可以让应用更加方便地使用Redis的不同数据类型。</p><p>只有详细掌握了这些特性的原理，你才能更好地判断是否使用6.0版本。如果你已经在使用6.0了，也可以看看怎么才能用得更好，少踩坑。</p><p>首先，我们来了解下6.0版本中新出的多线程特性。</p><h2>从单线程处理网络请求到多线程处理</h2><p><strong>在Redis 6.0中，非常受关注的第一个新特性就是多线程</strong>。这是因为，Redis一直被大家熟知的就是它的单线程架构，虽然有些命令操作可以用后台线程或子进程执行（比如数据删除、快照生成、AOF重写），但是，从网络IO处理到实际的读写命令处理，都是由单个线程完成的。</p><!-- [[[read_end]]] --><p>随着网络硬件的性能提升，Redis的性能瓶颈有时会出现在网络IO的处理上，也就是说，<strong>单个主线程处理网络请求的速度跟不上底层网络硬件的速度</strong>。</p><p>为了应对这个问题，一般有两种方法。</p><p>第一种方法是，用用户态网络协议栈（例如DPDK）取代内核网络协议栈，让网络请求的处理不用在内核里执行，直接在用户态完成处理就行。</p><p>对于高性能的Redis来说，避免频繁让内核进行网络请求处理，可以很好地提升请求处理效率。但是，这个方法要求在Redis的整体架构中，添加对用户态网络协议栈的支持，需要修改Redis源码中和网络相关的部分（例如修改所有的网络收发请求函数），这会带来很多开发工作量。而且新增代码还可能引入新Bug，导致系统不稳定。所以，Redis 6.0中并没有采用这个方法。</p><p>第二种方法就是采用多个IO线程来处理网络请求，提高网络请求处理的并行度。Redis 6.0就是采用的这种方法。</p><p>但是，Redis的多IO线程只是用来处理网络请求的，对于读写命令，Redis仍然使用单线程来处理。这是因为，Redis处理请求时，网络处理经常是瓶颈，通过多个IO线程并行处理网络操作，可以提升实例的整体处理性能。而继续使用单线程执行命令操作，就不用为了保证Lua脚本、事务的原子性，额外开发多线程互斥机制了。这样一来，Redis线程模型实现就简单了。</p><p>我们来看下，在Redis 6.0中，主线程和IO线程具体是怎么协作完成请求处理的。掌握了具体原理，你才能真正地会用多线程。为了方便你理解，我们可以把主线程和多IO线程的协作分成四个阶段。</p><p><strong>阶段一：服务端和客户端建立Socket连接，并分配处理线程</strong></p><p>首先，主线程负责接收建立连接请求。当有客户端请求和实例建立Socket连接时，主线程会创建和客户端的连接，并把 Socket 放入全局等待队列中。紧接着，主线程通过轮询方法把Socket连接分配给IO线程。</p><p><strong>阶段二：IO线程读取并解析请求</strong></p><p>主线程一旦把Socket分配给IO线程，就会进入阻塞状态，等待IO线程完成客户端请求读取和解析。因为有多个IO线程在并行处理，所以，这个过程很快就可以完成。</p><p><strong>阶段三：主线程执行请求操作</strong></p><p>等到IO线程解析完请求，主线程还是会以单线程的方式执行这些命令操作。下面这张图显示了刚才介绍的这三个阶段，你可以看下，加深理解。</p><p><img src="https://static001.geekbang.org/resource/image/58/cd/5817b7e2085e7c00e63534a07c4182cd.jpg?wh=3000*2000" alt=""></p><p><strong>阶段四：IO线程回写Socket和主线程清空全局队列</strong></p><p>当主线程执行完请求操作后，会把需要返回的结果写入缓冲区，然后，主线程会阻塞等待IO线程把这些结果回写到Socket中，并返回给客户端。</p><p>和IO线程读取和解析请求一样，IO线程回写Socket时，也是有多个线程在并发执行，所以回写Socket的速度也很快。等到IO线程回写Socket完毕，主线程会清空全局队列，等待客户端的后续请求。</p><p>我也画了一张图，展示了这个阶段主线程和IO线程的操作，你可以看下。</p><p><img src="https://static001.geekbang.org/resource/image/2e/1b/2e1f3a5bafc43880e935aaa4796d131b.jpg?wh=3000*1693" alt=""></p><p>了解了Redis主线程和多线程的协作方式，我们该怎么启用多线程呢？在Redis 6.0中，多线程机制默认是关闭的，如果需要使用多线程功能，需要在redis.conf中完成两个设置。</p><p><strong>1.设置io-thread-do-reads配置项为yes，表示启用多线程。</strong></p><pre><code>io-threads-do-reads yes
</code></pre><p>2.设置线程个数。一般来说，<strong>线程个数要小于Redis实例所在机器的CPU核个数</strong>，例如，对于一个8核的机器来说，Redis官方建议配置6个IO线程。</p><pre><code>io-threads  6
</code></pre><p>如果你在实际应用中，发现Redis实例的CPU开销不大，吞吐量却没有提升，可以考虑使用Redis 6.0的多线程机制，加速网络处理，进而提升实例的吞吐量。</p><h2>实现服务端协助的客户端缓存</h2><p>和之前的版本相比，Redis 6.0新增了一个重要的特性，就是实现了服务端协助的客户端缓存功能，也称为跟踪（Tracking）功能。有了这个功能，业务应用中的Redis客户端就可以把读取的数据缓存在业务应用本地了，应用就可以直接在本地快速读取数据了。</p><p>不过，当把数据缓存在客户端本地时，我们会面临一个问题：<strong>如果数据被修改了或是失效了，如何通知客户端对缓存的数据做失效处理？</strong></p><p>6.0实现的Tracking功能实现了两种模式，来解决这个问题。</p><p><strong>第一种模式是普通模式</strong>。在这个模式下，实例会在服务端记录客户端读取过的key，并监测key是否有修改。一旦key的值发生变化，服务端会给客户端发送invalidate消息，通知客户端缓存失效了。</p><p>在使用普通模式时，有一点你需要注意一下，服务端对于记录的key只会报告一次invalidate消息，也就是说，服务端在给客户端发送过一次invalidate消息后，如果key再被修改，此时，服务端就不会再次给客户端发送invalidate消息。</p><p>只有当客户端再次执行读命令时，服务端才会再次监测被读取的key，并在key修改时发送invalidate消息。这样设计的考虑是节省有限的内存空间。毕竟，如果客户端不再访问这个key了，而服务端仍然记录key的修改情况，就会浪费内存资源。</p><p>我们可以通过执行下面的命令，打开或关闭普通模式下的Tracking功能。</p><pre><code>CLIENT TRACKING ON|OFF
</code></pre><p><strong>第二种模式是广播模式</strong>。在这个模式下，服务端会给客户端广播所有key的失效情况，不过，这样做了之后，如果key 被频繁修改，服务端会发送大量的失效广播消息，这就会消耗大量的网络带宽资源。</p><p>所以，在实际应用时，我们会让客户端注册希望跟踪的key的前缀，当带有注册前缀的key被修改时，服务端会把失效消息广播给所有注册的客户端。<strong>和普通模式不同，在广播模式下，即使客户端还没有读取过key，但只要它注册了要跟踪的key，服务端都会把key失效消息通知给这个客户端</strong>。</p><p>我给你举个例子，带你看一下客户端如何使用广播模式接收key失效消息。当我们在客户端执行下面的命令后，如果服务端更新了user:id:1003这个key，那么，客户端就会收到invalidate消息。</p><pre><code>CLIENT TRACKING ON BCAST PREFIX user
</code></pre><p>这种监测带有前缀的key的广播模式，和我们对key的命名规范非常匹配。我们在实际应用时，会给同一业务下的key设置相同的业务名前缀，所以，我们就可以非常方便地使用广播模式。</p><p>不过，刚才介绍的普通模式和广播模式，需要客户端使用RESP 3协议，RESP 3协议是6.0新启用的通信协议，一会儿我会给你具体介绍。</p><p>对于使用RESP 2协议的客户端来说，就需要使用另一种模式，也就是重定向模式（redirect）。在重定向模式下，想要获得失效消息通知的客户端，就需要执行订阅命令SUBSCRIBE，专门订阅用于发送失效消息的频道_redis_:invalidate。同时，再使用另外一个客户端，执行CLIENT TRACKING命令，设置服务端将失效消息转发给使用RESP 2协议的客户端。</p><p>我再给你举个例子，带你了解下如何让使用RESP 2协议的客户端也能接受失效消息。假设客户端B想要获取失效消息，但是客户端B只支持RESP 2协议，客户端A支持RESP 3协议。我们可以分别在客户端B和A上执行SUBSCRIBE和CLIENT TRACKING，如下所示：</p><pre><code>//客户端B执行，客户端B的ID号是303
SUBSCRIBE _redis_:invalidate

//客户端A执行
CLIENT TRACKING ON BCAST REDIRECT 303
</code></pre><p>这样设置以后，如果有键值对被修改了，客户端B就可以通过_redis_:invalidate频道，获得失效消息了。</p><p>好了，了解了6.0 版本中的客户端缓存特性后，我们再来了解下第三个关键特性，也就是实例的访问权限控制列表功能（Access Control List，ACL），这个特性可以有效地提升Redis的使用安全性。</p><h2>从简单的基于密码访问到细粒度的权限控制</h2><p>在Redis 6.0 版本之前，要想实现实例的安全访问，只能通过设置密码来控制，例如，客户端连接实例前需要输入密码。</p><p>此外，对于一些高风险的命令（例如KEYS、FLUSHDB、FLUSHALL等），在Redis 6.0 之前，我们也只能通过rename-command来重新命名这些命令，避免客户端直接调用。</p><p>Redis 6.0 提供了更加细粒度的访问权限控制，这主要有两方面的体现。</p><p><strong>首先，6.0版本支持创建不同用户来使用Redis</strong>。在6.0版本前，所有客户端可以使用同一个密码进行登录使用，但是没有用户的概念，而在6.0中，我们可以使用ACL SETUSER命令创建用户。例如，我们可以执行下面的命令，创建并启用一个用户normaluser，把它的密码设置为“abc”：</p><pre><code>ACL SETUSER normaluser on &gt; abc
</code></pre><p><strong>另外，6.0版本还支持以用户为粒度设置命令操作的访问权限</strong>。我把具体操作列在了下表中，你可以看下，其中，加号（+）和减号（-）就分别表示给用户赋予或撤销命令的调用权限。</p><p><img src="https://static001.geekbang.org/resource/image/d1/c8/d1bd6891934cfa879ee080de1c5455c8.jpg?wh=2643*894" alt=""></p><p>为了便于你理解，我给你举个例子。假设我们要设置用户normaluser只能调用Hash类型的命令操作，而不能调用String类型的命令操作，我们可以执行如下命令：</p><pre><code>ACL SETUSER normaluser +@hash -@string
</code></pre><p>除了设置某个命令或某类命令的访问控制权限，6.0版本还支持以key为粒度设置访问权限。</p><p>具体的做法是使用波浪号“~”和key的前缀来表示控制访问的key。例如，我们执行下面命令，就可以设置用户normaluser只能对以“user:”为前缀的key进行命令操作：</p><pre><code>ACL SETUSER normaluser ~user:* +@all
</code></pre><p>好了，到这里，你了解了，Redis 6.0可以设置不同用户来访问实例，而且可以基于用户和key的粒度，设置某个用户对某些key允许或禁止执行的命令操作。</p><p>这样一来，我们在有多用户的Redis应用场景下，就可以非常方便和灵活地为不同用户设置不同级别的命令操作权限了，这对于提供安全的Redis访问非常有帮助。</p><h2>启用RESP 3协议</h2><p>Redis 6.0实现了RESP 3通信协议，而之前都是使用的RESP 2。在RESP 2中，客户端和服务器端的通信内容都是以字节数组形式进行编码的，客户端需要根据操作的命令或是数据类型自行对传输的数据进行解码，增加了客户端开发复杂度。</p><p>而RESP 3直接支持多种数据类型的区分编码，包括空值、浮点数、布尔值、有序的字典集合、无序的集合等。</p><p>所谓区分编码，就是指直接通过不同的开头字符，区分不同的数据类型，这样一来，客户端就可以直接通过判断传递消息的开头字符，来实现数据转换操作了，提升了客户端的效率。除此之外，RESP 3协议还可以支持客户端以普通模式和广播模式实现客户端缓存。</p><h2>小结</h2><p>这节课，我向你介绍了Redis 6.0的新特性，我把这些新特性总结在了一张表里，你可以再回顾巩固下。</p><p><img src="https://static001.geekbang.org/resource/image/21/f0/2155c01bf3129d5d58fcb98aefd402f0.jpg?wh=2922*2097" alt=""></p><p>最后，我也再给你一个小建议：因为Redis 6.0是刚刚推出的，新的功能特性还需要在实际应用中进行部署和验证，所以，如果你想试用Redis 6.0，可以尝试先在非核心业务上使用Redis 6.0，一方面可以验证新特性带来的性能或功能优势，另一方面，也可以避免因为新特性不稳定而导致核心业务受到影响。</p><h2>每课一问</h2><p>你觉得，Redis 6.0的哪个或哪些新特性会对你有帮助呢？</p><p>欢迎在留言区写下你的思考和答案，我们一起交流讨论。如果你觉得今天的内容对你有所帮助，也欢迎你分享给你的朋友或同事。我们下节课见。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/90/8a/288f9f94.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Kaito</span>
  </div>
  <div class="_2_QraFYR_0">Redis 6.0 的哪些新特性帮助最大？<br><br>我觉得 Redis 6.0 提供的多 IO 线程和客户端缓存这两大特性，对于我们使用 Redis 帮助最大。<br><br>多 IO 线程可以让 Redis 在并发量非常大时，让其性能再上一个台阶，性能提升近 1 倍，对于单机 Redis 性能要求更高的业务场景，非常有帮助。<br><br>而客户端缓存可以让 Redis 的数据缓存在客户端，相当于每个应用进程多了一个本地缓存，Redis 数据没有变化时，业务直接在应用进程内就能拿到数据，这不仅节省了网络带宽，降低了 Redis 的请求压力，还充分利用了业务应用的资源，对应用性能的提升也非常大。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-23 00:11:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83erKbqXs1LibG86q2OxHLic21eduYd9JPCf5ZaDlic3Dk7P0rS1n9jjExrvE6tmartuhPQhgSeEHZ1SaQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_04f704</span>
  </div>
  <div class="_2_QraFYR_0">老师，redis支持多线程后，怎么实现单命令操作原子性的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Redis 6.0中的多线程只是指，在客户端请求接收和解析，以及请求后的数据通过网络返回给客户端时，使用了多线程。而命令请求本身的数据读写操作还是由单线程来完成的，所以仍然可以保证单命令操作的原子性。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-16 02:41:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/8f/cf/890f82d6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>那时刻</span>
  </div>
  <div class="_2_QraFYR_0">Redis 6.0增加了IO线程来处理网络请求，如果客户端先发送了一个`set key1 val1`写命令，紧接着发送一个`get key1`读命令。请问老师，由于IO线程是多线程处理的，是否会导致`get key1`读命令 先于 `set key1 val1`写命令执行呢？结果客户端读到了key1的旧值。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-23 09:26:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1c/fb/ab/c0c29cda.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>王世艺</span>
  </div>
  <div class="_2_QraFYR_0">多线程io和epoll啥区别</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-23 08:51:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/ed/eb/88cac7a5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>东</span>
  </div>
  <div class="_2_QraFYR_0">6.0的权限细粒度控制对我们很有用，以前多个微服务共享同一个redis集群，权限没法隔离，现在可以控制不同的服务使用不同的key前缀，从而很好的隔离了服务，可以有效避免误操作，或者一个服务的bug影响到所有服务。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这是个很好的用例场景！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-23 10:50:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/19/61/119cbde2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Dolly</span>
  </div>
  <div class="_2_QraFYR_0">客户端缓存那个直接解决redis和本地缓存的一致性问题</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 除了保证一致性，客户端缓存本身也能加速访问，所以，这个特性是Redis 6.0中比较重要的一个，可以重点关注 :)</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-31 22:47:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/3a/1a/ae3c1492.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>🌾🌾🌾小麦🌾🌾🌾</span>
  </div>
  <div class="_2_QraFYR_0">请问老师客户端缓存端缓存会不会导致缓存污染及内存泄露问题？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我们说的缓存污染一般是指缓存系统里面的数据不被访问，但是由于缓存策略没有及时淘汰，又滞留在缓存中，导致缓存空间被占用，但是又不服务访问请求，造成了污染。<br><br>客户端缓存的主要问题是要和缓存系统中的数据保存一致，也就是说缓存系统中的数据被更新了，客户端需要及时做invalidation操作，避免应用在客户端缓存中读到旧的缓存数据。<br><br>内存泄露一般是内存没有正确回收导致的，和缓存污染、客户端缓存倒没有必然的联系。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-19 10:18:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/22/71/11/2d5cdb14.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>pretty.zh</span>
  </div>
  <div class="_2_QraFYR_0">老师，redis常量池是什么</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你说的是指Redis中的整数对象共享池么？例如0到9999这些整数会被频繁用到，所以，使用共享对象来表示这些数，每个数只存一份String对象，可以节省内存空间。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-23 19:39:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/dd/61/544c2838.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>kevin</span>
  </div>
  <div class="_2_QraFYR_0">请问下，redis的客户端缓存与业务实例的本地缓存有区别吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-23 08:35:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/27/22/4c/d413494f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>听秋</span>
  </div>
  <div class="_2_QraFYR_0">老师，客户端缓存的普通模式，当不在收到服务端的通知时，服务端 的 key 被修改了，应用读的是缓存到本地的数据，那不就读到旧数据了吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-26 16:03:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/23/7e/76/368394bf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>哦</span>
  </div>
  <div class="_2_QraFYR_0">请问，这里Redis6.0的多线程模式是不是可以理解为多Reactor单线程的模式</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-26 16:46:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/ba/e4/6df89add.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>芋头</span>
  </div>
  <div class="_2_QraFYR_0">Redis 6.0新特性:1.处理网络IO采用多线程模式2.服务端协助的客户端的缓存（普通模式和广播模式及重定向模式通知客户端key失效的信息）3.实现RESP3 协议，支持多种类型区分编码4.实例的细粒度访问权限控制（不同用户不同权限及用户为粒度的命令权限及支持以key为粒度用户权限）</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-09 18:22:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/03/1c/c9fe6738.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Kvicii.Y</span>
  </div>
  <div class="_2_QraFYR_0">这个多IO线程我没有看懂作用、能解释下吗</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-01 09:10:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/ec/13/49e98289.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>neohope</span>
  </div>
  <div class="_2_QraFYR_0">老师好，我翻了一下6.2的源码，多线程IO处理这个地方，主线程通知IO线程处理数据后，线程好像不算是阻塞状态吧？<br><br>int handleClientsWithPendingReadsUsingThreads(void) 函数里，主线程会一次性将多个接收请求分配给多个IO线程，然后一个死循环等待IO线程全部处理完毕，但线程状态应该是运行状态，不是阻塞状态吧？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-25 13:41:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/95/8d/e27e5c7a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>陌上花开</span>
  </div>
  <div class="_2_QraFYR_0">主线程接收完请求以后，通过轮询的方式交给IO线程处理，IO处理的速度可能会存在不一致的情况，如果保证主线程在执行命令的时候是按照轮询的顺序？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-08 10:00:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2b/d0/32/4777bf94.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_bdce73</span>
  </div>
  <div class="_2_QraFYR_0">多线程是不是把多路复用epoll模型给去掉了，各位大神怎么看</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-16 16:11:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/6f/e2/f3b05833.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>A 拽丫头</span>
  </div>
  <div class="_2_QraFYR_0">多线程io  和  io多路复用有啥区别？ 我怎么感觉这两个是一个咧</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-03 18:58:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/01/c5/b48d25da.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>cake</span>
  </div>
  <div class="_2_QraFYR_0">客户端需要根据操作的命令或是数据类型自行对传输的数据进行解码，增加了客户端开发复杂度   应该是对传输的数据进行编码叭 0.0？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-03 21:13:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/22/bf/4b/2acf59c1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我</span>
  </div>
  <div class="_2_QraFYR_0">redis6新的过期key扫描机制，相比于redis4可以有效的防止过期key堆积，可以节约成本。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-09 19:46:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/93/4f/61edeea6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Ac、</span>
  </div>
  <div class="_2_QraFYR_0">对于Redis 6.0 来说，如果开启多 IO 线程的话，就不用 epoll。也就是说，多 IO 线程是 epoll 的一个替代方案？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-06 11:26:22</div>
  </div>
</div>
</div>
</li>
</ul>