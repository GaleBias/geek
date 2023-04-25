<audio title="加餐（一）_ 经典的Redis学习资料有哪些？" src="https://static001.geekbang.org/resource/audio/58/a0/588ef87ce0f83d7da8e48d2da3715fa0.mp3" controls="controls"></audio> 
<p>你好，我是蒋德钧。</p><p>咱们课程的“基础篇”已经结束了。在这个模块，我们学习了Redis的系统架构、数据结构、线程模型、持久化、主从复制和切片集群这些核心知识点，相信你已经初步构建了自己的一套基础知识框架。</p><p>不过，如果想要持续提升自己的技术能力，还需要不断丰富自己的知识体系，那么，阅读就是一个很好的方式。所以，这节课，我就给你推荐几本优秀的书籍，以及一些拓展知识面的其他资料，希望能够帮助你全面掌握Redis。</p><h2>经典书籍</h2><p>在学习Redis时，最常见的需求有三个方面。</p><ul>
<li>日常使用操作：比如常见命令和配置，集群搭建等；</li>
<li>关键技术原理：比如我们介绍过的IO模型、AOF和RDB机制等；</li>
<li>在实际使用时的经验教训，比如，Redis响应变慢了怎么办？Redis主从库数据不一致怎么办？等等。</li>
</ul><p>接下来，我就根据这些需求，把参考资料分成工具类、原理类、实战类三种。我们先来看工具类参考资料。</p><h3>工具书：《Redis使用手册》</h3><p>一本好的工具书，可以帮助我们快速地了解或查询Redis的日常使用命令和操作方法。我要推荐的《Redis使用手册》，就是一本非常好用的工具书。</p><p>在这本书中，作者把Redis的内容分成了三大部分，分别是“数据结构与应用”“附加功能”和“多机功能”。其中，我认为最有用的就是“数据结构与应用”的内容，因为它提供了丰富的操作命令介绍，不仅涵盖了Redis的5大基本数据类型的主要操作命令，还介绍了4种扩展数据类型的命令操作，包括位图、地址坐标、HyperLogLog和流。只要这本书在手边，我们就能很轻松地了解和正确使用Redis的大部分操作命令了。</p><!-- [[[read_end]]] --><p>不过，如果你想要了解最全、最新的Redis命令操作，我建议你把Redis的命令参考网站收录到你的浏览器书签中，随用随查。目前，Redis官方提供的所有命令操作参考肯定是最全、最新的，建议你优先使用这个<a href="https://redis.io/commands/">官方网站</a>。在这个网页上查找命令操作非常方便，我们既可以通过命令操作的名称直接查找，也可以根据Redis的功能，分类查找对应功能下的操作，例如和集群相关的操作，和发布订阅相关的操作。考虑到有些同学可能想看中文版，我再给你提供一个<a href="http://redisdoc.com/">翻译版的命令参考</a>。</p><p>除了提供Redis的命令操作介绍外，《Redis使用手册》还提供了“附加功能”部分，介绍了Redis数据库的管理操作和过期key的操作，这对我们进行Redis数据库运维（例如迁移数据、清空数据库、淘汰数据等）提供了操作上的指导。</p><p>有了工具手册，我们就能很轻松地掌握不同命令操作的输入参数、返回结果和复杂度了。接下来，就是进一步了解各种机制背后的原理了，我再跟你分享一本原理书。</p><h3>原理书：《Redis设计与实现》</h3><p>虽然《Redis设计与实现》和《Redis使用手册》是同一个作者写的，但是它们的侧重点不一样，这本书更加关注Redis关键机制的实现原理。</p><p>介绍Redis原理的资料有很多，但我认为，这本书讲解得非常透彻，尤其是在Redis底层数据结构、RDB和AOF持久化机制，以及哨兵机制和切片集群的介绍上，非常容易理解，我建议你重点学习下这些部分的内容。</p><p>除了文字讲解，这本书还针对一些难点问题，例如数据结构的组成、哨兵实例间的交互过程、切片集群实例的交互过程等，都使用了非常清晰的插图来表示，可以最大程度地降低学习难度。</p><p>其实，这本书也是我自己读的第一本Redis参考书，可以说，是它把我领进了Redis原理的大门。当时在学习时，正是因为有了这些插图的帮助，我才能快速地搞懂核心原理。直到今天，我都还记得这本书中的一些插图，真是受益匪浅。</p><p>虽然这本书的出版日期比较早（它针对的是Redis 3.0），但是里面讲的很多原理现在依然是适用的，它可以帮助你在从入门Redis到精通的道路上，迈进一大步。</p><h3>实战书：《Redis开发与运维》</h3><p>在实战方面，《Redis开发与运维》是一本不错的参考书。</p><p>首先，它介绍了Redis的Java和Python客户端，以及Redis用于缓存设计的关键技术和注意事项，这些内容在其他参考书中不太常见，你可以重点学习下。</p><p>其次，它围绕客户端、持久化、主从复制、哨兵、切片集群等几个方面，着重介绍了在日常的开发运维过程中遇到的问题和“坑”，都是经验之谈，可以帮助你提前做规避。</p><p>另外，这本书还针对Redis阻塞、优化内存使用、处理bigkey这几个经典问题，提供了解决方案，非常值得一读。在阅读的时候，你可以把目录里的问题整理一下，做成列表，这样，在遇到问题的时候，就可以对照着这个列表，快速地找出原因，并且利用书中的方案去解决问题了。</p><p>当然，要想真正提升实战能力，光读书是远远不够的，毕竟，“纸上得来终觉浅”。所以，我还想再给你分享两条建议。</p><p>第一个建议是阅读源码。读源码其实也是一种实战锻炼，可以帮助你从代码逻辑中彻底理解Redis系统的实际运行机制，<strong>当遇到问题时，可以直接从代码层面进行定位、分析和解决问题</strong>。阅读Redis源码，最直接的材料就是Redis在GitHub上的<a href="https://github.com/redis/redis">源码库</a>。另外，有一个<a href="https://github.com/huangz1990/redis-3.0-annotated">网站</a>提供了Redis 3.0源码的部分中文注释，你也可以参考一下。</p><p>另外，我们还需要亲自动手实践。在课程的留言中，我看到有同学说“没有服务器无法实践”，其实，Redis运行后本身就是一个进程，我们是可以直接使用自己的电脑进行部署的。只要不是性能测试，在功能测试或者场景模拟上，自己电脑的环境一般都是可以胜任的。比如说，要想部署主从集群或者切片集群，模拟主库故障，我们完全可以在自己电脑上起多个Redis实例来完成，只要保证它们的端口号不同，就可以了。</p><p>好了，关于Redis本身的书籍的推荐，就先告一段落了，接下来，我想再给你分享一些扩展内容。</p><h2>扩展阅读方向</h2><p>通过前面几节课的学习，我相信你一定已经发现了，Redis的很多关键功能，其实和操作系统底层的实现机制是相关的，比如说，非阻塞的网络框架、RDB生成和AOF重写时涉及到的fork和写时复制机制，等等。另外，Redis主从集群中的哨兵机制，以及切片集群的数据分布还涉及到一些分布式系统的内容。</p><p>我用一张图片，展示一下Redis的关键机制和操作系统、分布式系统的对应知识点。</p><p><img src="https://static001.geekbang.org/resource/image/a0/2c/a0f558fbf9105817744ee2c44230c62c.jpg?wh=2381*1140" alt=""></p><p>AOF日志的刷盘时机和操作系统的fsync机制、高速页缓存的刷回有关，而网络框架跟epoll有关，RDB生成和AOF重写与fork、写时复制有关（我在前面第3、4、5讲上讲过它们的关联）。</p><p>此外，我在<a href="https://time.geekbang.org/column/article/275337">第8讲</a>介绍的哨兵选主过程，其实是分布式系统中的经典的Raft协议的执行过程，如果你比较了解Raft协议，就能很轻松地掌握哨兵选主的运行机制了。在<a href="https://time.geekbang.org/column/article/276545">第9讲</a>，我们学习了实现切片集群的Redis Cluster方案，其实，业界还有一种实现方案，就是ShardedJedis，而它就用到了分布式系统中经典的一致性哈希机制。</p><p>所以，如果说你希望自己的实战能力能够更强，我建议你<strong>读一读操作系统和分布式系统方面的经典教材，</strong>比如《操作系统导论》。尤其是这本书里对进程、线程的定义，对进程API、线程API以及对文件系统fsync操作、缓存和缓冲的介绍，都是和Redis直接相关的；再比如，《大规模分布式存储系统：原理解析与架构实战》中的分布式系统章节，可以让你掌握Redis主从集群、切片集群涉及到的设计规范。了解下操作系统和分布式系统的基础知识，既能帮你厘清容易混淆的概念（例如Redis主线程、子进程），也可以帮助你将一些通用的设计方法（例如一致性哈希）应用到日常的实践中，做到融会贯通，举一反三。</p><h2>小结</h2><p>这节课，我给你推荐了三本参考书，分别对应了Redis的命令操作使用、关键机制的实现原理，以及实战经验，还介绍了Redis操作命令快速查询的两个网站，这可是我们日常使用Redis的必备工具，可以提升你使用操作Redis的效率。另外，对于Redis关键机制涉及到的扩展知识点，我从操作系统和分布式系统两个方面进行了补充。</p><p>Redis的源码阅读是成为Redis专家的必经之路，你可以阅读一下Redis在GitHub上的源码库，如果觉得有难度，也可以从带有中文注释的源码阅读网站入手。</p><p>最后，我也想请你聊一聊，你的Redis学习资料都有哪些呢？欢迎在留言区分享一下，我们一起进步。另外，如果你觉得今天的内容对你有所帮助，也欢迎你分享给你的朋友或同事。</p>
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
  <div class="_2_QraFYR_0">老师推荐的书籍都非常经典，这几本是学习Redis的必读书籍。<br><br>如果你觉得这些书读起来困难，我推荐一本之前同事写的《Redis 深度历险：核心原理与应用实践》，这本书很薄，而且最大的特点是讲解接地气，它可以让你对Redis的基础使用、业务场景、原理分析有一个基本的认识和了解，作为入门和进阶非常合适，起码可以让你重新树立起深入学习Redis的信心。<br><br>另外，真心建议大家试着去读一下Redis源码，没有想象的那么难，而且Redis的代码质量非常高，由于是单线程的内存数据库，没有多线程运行时的复杂逻辑，读起来非常顺畅！其实很多我们纠结的小问题，不要只靠猜和网上查资料，读一下源码就能快速找到答案。而且现在源码分析的文章非常多，讲解的也很细，结合起来读代码并不难。<br><br>只有自己试着去读源码，当遇到问题时，再查资料，学习到的东西才是最深刻的。而且在查资料时，还会发现更大的世界，例如老师文章提到的操作系统知识、分布式系统问题、架构设计的取舍等等，这样我们所学到的知识不再是一个面，而是慢慢形成一个知识网，这样才能够达到融会贯通，举一反三。<br><br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 同意Kaito同学的源码阅读建议 ：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-28 00:40:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/43/79/18073134.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>test</span>
  </div>
  <div class="_2_QraFYR_0">三本书读了两，源码也过了一遍，操作系统导论也看过，推荐《redis5设计与源码分析》讲源码的，很不错。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 爱读书的好同学！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-28 00:11:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/42/44/d3d67640.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Hills录</span>
  </div>
  <div class="_2_QraFYR_0">推荐一本书《数据密集型应用系统设计》，一个专栏《分布式数据库30讲》，可以从更高视角看待 redis 的设计</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-01 22:05:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/fe/83/df562574.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>慎独明强</span>
  </div>
  <div class="_2_QraFYR_0">之前组长走的时候留了一本《Redis开发和运维》给我，面试问到redis伸缩容的时候去看了下。后面面试又被问到Redis的数据结构.bitmap，自己就去网上买了《Redis设计与实现》 ，目前也在看。看了老师的建议去阅读源码，没有学过C，阅读起来会有难度吗？上面是自己的学习资料</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: redis是用C写的，所以学习时还是要有C语言的基础，否则看起来会有些困难。可以先把C的基础打下。<br><br>不过可以按照工具使用，了解原理，掌握实战三阶段来渐进学习，源码阅读基本在第二阶段后期，或第三阶段了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-28 08:17:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/5f/e5/54325854.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>范闲</span>
  </div>
  <div class="_2_QraFYR_0">推荐两本书:一本老师已经提到过了:redis设计与实现，另外一本redis深度历险。<br><br>建议阅读Redis源码，从基础数据结构看，再到db，再到网络部分，整体内容都很清晰明了。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-28 08:03:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/5f/e5/54325854.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>范闲</span>
  </div>
  <div class="_2_QraFYR_0">另外再补充下setinel选主的过程是用的Gossip协议吧。redis的选主过程没有raft里面那种明显的角色划分</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-28 08:06:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/5c/44/d07c0865.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_d960af</span>
  </div>
  <div class="_2_QraFYR_0">巧了 都下载了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-12 22:50:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/d4/f3/129d6dfe.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>李二木</span>
  </div>
  <div class="_2_QraFYR_0">之前就觉得哨兵选主机制像raft</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-28 08:35:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/c2/fe/038a076e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>阿卧</span>
  </div>
  <div class="_2_QraFYR_0">看了《redis设计与实现》和《redis深度历险：核心原理与应用实践》，源码内容还没有接触过，需要再看看源码。缓存的设计基本可以串起来形成知识网，但是有些细节知识还需要打磨学习</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-28 11:44:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/92/6d/becd841a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>escray</span>
  </div>
  <div class="_2_QraFYR_0">三本书《Redis 使用手册》、《Redis设计与实现》、《Redis开发与运维》<br><br>官方网站<br><br>阅读源码，动手实践<br><br>拓展阅读《操作系统导论》、《大规模分布式存储系统：原理解析与架构实战》<br><br>还有 Kaito 大神推荐的《Redis 深度历险：核心原理与应用实践》<br><br>有一点好奇，为什么推荐的 Redis 的书大多是中文的？<br><br>另外一点，最近也在学习 Elastic 相关的内容，Elastic 有自己的宇宙——全文检索、日志审计、安全分析，而 Redis 似乎要“单纯”一些。<br><br>目前手里并没有和 Redis 直接相关的项目，所以估计暂时也只能是把专栏先过一遍，如果后续有需要，再按图索骥，深入学习。<br><br>另外，蒋德钧老师在极客时间有一个两天的 Redis 集训班，应该也很值得推荐。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-18 20:51:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/41/13/ab14ad25.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>树心</span>
  </div>
  <div class="_2_QraFYR_0">看到评论区有人推荐《数据密集型应用系统设计》，之前也看到过这本书，单看书名觉得可能会比较生涩，但也许会像评论里面说的那样可以从更高视角看待redis。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-06 08:23:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTITcwicqBDYzXtLibUtian172tPs7rJpqG1Vab4oGjnguA9ziaYjDCILSGaS6qRiakvRdUEhdmSG0BGPKw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>大饶Raysir</span>
  </div>
  <div class="_2_QraFYR_0">都是好书！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-21 23:34:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/66/34/0508d9e4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>u</span>
  </div>
  <div class="_2_QraFYR_0">www.redis.cn也不错</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-09 10:05:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/16/5b/83a35681.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Monday</span>
  </div>
  <div class="_2_QraFYR_0">没看过一本的举个爪。<br>以前只是在某网站下载了一套redis视频，学习了几遍，敲了些命令。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-28 16:09:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/5e/b3/e35e5d3c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>..e</span>
  </div>
  <div class="_2_QraFYR_0">redis单机，aof中有散列表写入记录没有删除记录，但是散列表丢失，访问key不存在。请问有什么排查思路？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-28 10:40:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/d4/f3/129d6dfe.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>李二木</span>
  </div>
  <div class="_2_QraFYR_0">视频资料话可以去B站搜搜看，适合入门</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-28 08:36:53</div>
  </div>
</div>
</div>
</li>
</ul>