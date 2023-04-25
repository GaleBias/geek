<audio title="21 _ AKF立方体：怎样通过可扩展性来提高性能？" src="https://static001.geekbang.org/resource/audio/41/0d/415yy34cdbfd00e6df7a974e24d5a70d.mp3" controls="controls"></audio> 
<p>你好，我是陶辉。</p><p>上一讲我们谈到，调低一致性可以提升有状态服务的性能。这一讲我们扩大范围，结合无状态服务，看看怎样提高分布式系统的整体性能。</p><p>当你接收到运维系统的短信告警，得知系统性能即将达到瓶颈，或者会议上收到老板兴奋的通知，接下来市场开缰拓土，业务访问量将要上一个大台阶时，一定会马上拿起计算器，算算要加多少台机器，系统才能扛得住新增的流量。</p><p>然而，有些服务虽然可以通过加机器提升性能，但可能你加了一倍的服务器，却发现系统的吞吐量没有翻一倍。甚至有些服务无论你如何扩容，性能都没有半点提升。这缘于我们扩展分布式系统的方向发生了错误。</p><p>当我们需要分布式系统提供更强的性能时，该怎样扩展系统呢？什么时候该加机器？什么时候该重构代码？扩容时，究竟该选择哈希算法还是最小连接数算法，才能有效提升性能？</p><p>在面对Scalability可伸缩性问题时，我们必须有一个系统的方法论，才能应对日益复杂的分布式系统。这一讲我将介绍AKF立方体理论，它定义了扩展系统的3个维度，我们可以综合使用它们来优化性能。</p><h2>如何基于AKF X轴扩展系统？</h2><p>AKF立方体也叫做<a href="https://en.wikipedia.org/wiki/Scale_cube">scale cube</a>，它在《The Art of Scalability》一书中被首次提出，旨在提供一个系统化的扩展思路。AKF把系统扩展分为以下三个维度：</p><!-- [[[read_end]]] --><ul>
<li>X轴：直接水平复制应用进程来扩展系统。</li>
<li>Y轴：将功能拆分出来扩展系统。</li>
<li>Z轴：基于用户信息扩展系统。</li>
</ul><p>如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/61/aa/61633c7e7679fd10b915494b72abb3aa.jpg?wh=1218*974" alt=""></p><p>我们日常见到的各种系统扩展方案，都可以归结到AKF立方体的这三个维度上。而且，我们可以同时组合这3个方向上的扩展动作，使得系统可以近乎无限地提升性能。为了避免对AKF的介绍过于抽象，下面我用一个实际的例子，带你看看这3个方向的扩展到底该如何应用。</p><p>假定我们开发一个博客平台，用户可以申请自己的博客帐号，并在其上发布文章。最初的系统考虑了MVC架构，将数据状态及关系模型交给数据库实现，应用进程通过SQL语言操作数据模型，经由HTTP协议对浏览器客户端提供服务，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/cd/e0/cda814dcfca8820c81024808fe96b1e0.jpg?wh=876*874" alt=""></p><p>在这个架构中，处理业务的应用进程属于无状态服务，用户数据全部放在了关系数据库中。因此，当我们在应用进程前加1个负载均衡服务后，就可以通过部署更多的应用进程，提供更大的吞吐量。而且，初期增加应用进程，RPS可以获得线性增长，很实用。</p><p><img src="https://static001.geekbang.org/resource/image/d7/62/d7f75f3f5a5e6f8a07b1c47501606962.png?wh=879*856" alt=""></p><p>这就叫做沿AKF X轴扩展系统。这种扩展方式最大的优点，就是开发成本近乎为零，而且实施起来速度快！在搭建好负载均衡后，只需要在新的物理机、虚拟机或者微服务上复制程序，就可以让新进程分担请求流量，而且不会影响事务Transaction的处理。</p><p>当然，AKF X轴扩展最大的问题是只能扩展无状态服务，当有状态的数据库出现性能瓶颈时，X轴是无能为力的。例如，当用户数据量持续增长，关系数据库中的表就会达到百万、千万行数据，SQL语句会越来越慢，这时可以沿着AKF Z轴去分库分表提升性能。又比如，当请求用户频率越来越高，那么可以把单实例数据库扩展为主备多实例，沿Y轴把读写功能分离提升性能。下面我们先来看AKF Y轴如何扩展系统。</p><h2>如何基于AKF Y轴扩展系统？</h2><p>当数据库的CPU、网络带宽、内存、磁盘IO等某个指标率先达到上限后，系统的吞吐量就达到了瓶颈，此时沿着AKF X轴扩展系统，是没有办法提升性能的。</p><p>在现代经济中，更细分、更专业的产业化、供应链分工，可以给社会带来更高的效率，而AKF Y轴与之相似，当遇到上述性能瓶颈后，拆分系统功能，使得各组件的职责、分工更细，也可以提升系统的效率。比如，当我们将应用进程对数据库的读写操作拆分后，就可以扩展单机数据库为主备分布式系统，使得主库支持读写两种SQL，而备库只支持读SQL。这样，主库可以轻松地支持事务操作，且它将数据同步到备库中也并不复杂，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/86/74/865885bb7213e62b8e1b715d85c9a974.png?wh=834*791" alt=""></p><p>当然，上图中如果读性能达到了瓶颈，我们可以继续沿着AKF X轴，用复制的方式扩展多个备库，提升读SQL的性能，可见，AKF多个轴完全可以搭配着协同使用。</p><p>拆分功能是需要重构代码的，它的实施成本比沿X轴简单复制扩展要高得多。在上图中，通常关系数据库的客户端SDK已经支持读写分离，所以实施成本由中间件承担了，这对我们理解Y轴的实施代价意义不大，所以我们再来看从业务上拆分功能的例子。</p><p>当这个博客平台访问量越来越大时，一台主库是无法扛住所有写流量的。因此，基于业务特性拆分功能，就是必须要做的工作。比如，把用户的个人信息、身份验证等功能拆分出一个子系统，再把文章、留言发布等功能拆分到另一个子系统，由无状态的业务层代码分开调用，并通过事务组合在一起，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/3b/af/3bba7bc19965bb9b01c058e67a6471af.png?wh=644*833" alt=""></p><p>这样，每个后端的子应用更加聚焦于细分的功能，它的数据库规模会变小，也更容易优化性能。比如，针对用户登录功能，你可以再次基于Y轴将身份验证功能拆分，用Redis等服务搭建一个基于LRU算法淘汰的缓存系统，快速验证用户身份。</p><p>然而，沿Y轴做功能拆分，实施成本非常高，需要重构代码并做大量测试工作，上线部署也很复杂。比如上例中要对数据模型做拆分（如同一个库中的表拆分到多个库中，或者表中的字段拆到多张表中），设计组件之间的API交互协议，重构无状态应用进程中的代码，为了完成升级还要做数据迁移，等等。</p><p>解决数据增长引发的性能下降问题，除了成本较高的AKF Y轴扩展方式外，沿Z轴扩展系统也很有效，它的实施成本更低一些，下面我们具体看一下。</p><h2>如何基于AKF Z轴扩展系统？</h2><p>不同于站在服务角度扩展系统的X轴和Y轴，AKF Z轴则从用户维度拆分系统，它不仅可以提升数据持续增长降低的性能，还能基于用户的地理位置获得额外收益。</p><p>仍然以上面虚拟的博客平台为例，当注册用户数量上亿后，无论你如何基于Y轴的功能去拆分表（即“垂直”地拆分表中的字段），都无法使得关系数据库单个表的行数在千万级以下，这样表字段的B树索引非常庞大，难以完全放在内存中，最后大量的磁盘IO操作会拖慢SQL语句的执行。</p><p>这个时候，关系数据库最常用的分库分表操作就登场了，它正是AKF沿Z轴拆分系统的实践。比如已经含有上亿行数据的User用户信息表，可以分成10个库，每个库再分成10张表，利用固定的哈希函数，就可以把每个用户的数据映射到某个库的某张表中。这样，单张表的数据量就可以降低到1百万行左右，如果每个库部署在不同的服务器上（具体的部署方式视访问吞吐量以及服务器的配置而定），它们处理的数据量减少了很多，却可以独占服务器的硬件资源，性能自然就有了提升。如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/dc/83/dc9e29827c26f89ff3459b5c99313583.png?wh=852*815" alt=""></p><p>分库分表是关系数据库中解决数据增长压力的最有效办法，但分库分表同时也导致跨表的查询语句复杂许多，而跨库的事务几乎难以实现，因此这种扩展的代价非常高。当然，如果你使用的是类似MySQL这些成熟的关系数据库，整个生态中会有厂商提供相应的中间件层，使用它们可以降低Z轴扩展的代价。</p><p>再比如，最开始我们采用X轴复制扩展的服务，它们的负载均衡策略很简单，只需要选择负载最小的上游服务器即可，比如RoundRobin或者最小连接算法都可以达到目的。但若上游服务器通过Y轴扩展，开启了缓存功能，那么考虑到缓存的命中率，就必须改用Z轴扩展的方式，基于用户信息做哈希规则下的新路由，尽量将同一个用户的请求命中相同的上游服务器，才能充分提高缓存命中率。</p><p>Z轴扩展还有一个好处，就是可以充分利用IDC与用户间的网速差，选择更快的IDC为用户提供高性能服务。网络是基于光速传播的，当IDC跨城市、国家甚至大洲时，用户访问不同IDC的网速就会有很大差异。当然，同一地域内不同的网络运营商之间，也会有很大的网速差。</p><p>例如你在全球都有IDC或者公有云服务器时，就可以通过域名为当地用户就近提供服务，这样性能会高很多。事实上，CDN技术就基于IP地址的位置信息，就近为用户提供静态资源的高速访问。</p><p>下图中，我使用了2种Z轴扩展系统的方式。首先是基于客户端的地理位置，选择不同的IDC就近提供服务。其次是将不同的用户分组，比如免费用户组与付费用户组，这样在业务上分离用户群体后，还可以有针对性地提供不同水准的服务。</p><p><img src="https://static001.geekbang.org/resource/image/35/3b/353d8515d40db25eebee23889a3ecd3b.png?wh=1061*766" alt=""></p><p>沿AKF Z轴扩展系统可以解决数据增长带来的性能瓶颈，也可以基于数据的空间位置提升系统性能，然而它的实施成本比较高，尤其是在系统宕机、扩容时，一旦路由规则发生变化，会带来很大的数据迁移成本，[第24讲] 我将要介绍的一致性哈希算法，其实就是用来解决这一问题的。</p><h2>小结</h2><p>这一讲我们介绍了如何基于AKF立方体的X、Y、Z三个轴扩展系统提升性能。</p><p>X轴扩展系统时实施成本最低，只需要将程序复制到不同的服务器上运行，再用下游的负载均衡分配流量即可。X轴只能应用在无状态进程上，故无法解决数据增长引入的性能瓶颈。</p><p>Y轴扩展系统时实施成本最高，通常涉及到部分代码的重构，但它通过拆分功能，使系统中的组件分工更细，因此可以解决数据增长带来的性能压力，也可以提升系统的总体效率。比如关系数据库的读写分离、表字段的垂直拆分，或者引入缓存，都属于沿Y轴扩展系统。</p><p>Z轴扩展系统时实施成本也比较高，但它基于用户信息拆分数据后，可以在解决数据增长问题的同时，基于地理位置就近提供服务，进而大幅度降低请求的时延，比如常见的CDN就是这么提升用户体验的。但Z轴扩展系统后，一旦发生路由规则的变动导致数据迁移时，运维成本就会比较高。</p><p>当然，X、Y、Z轴的扩展并不是孤立的，我们可以同时应用这3个维度扩展系统。分布式系统非常复杂，AKF给我们提供了一种自上而下的方法论，让我们能够针对不同场景下的性能瓶颈，以最低的成本提升性能。</p><h2>思考题</h2><p>最后给你留一道思考题，我们在谈到Z轴扩展时（比如关系数据库的分库分表），提到了基于哈希函数来设置路由规则，请结合<a href="https://time.geekbang.org/column/article/232351">[第3讲]</a> 的内容，谈谈你认为应该如何设计哈希函数，才能使它满足符合Z轴扩展的预期？期待你的总结。</p><p>感谢阅读，如果你觉得这节课对你有一些启发，也欢迎把它分享给你的朋友。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/7f/91/962eba1a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>唐朝首都</span>
  </div>
  <div class="_2_QraFYR_0">（1）以质数为基数；（2）充分利用key的个性信息</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-24 08:48:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/4d/d3/67bdcca9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>林铭铭</span>
  </div>
  <div class="_2_QraFYR_0">总结下：扩实例，拆功能，拆数据</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-21 21:03:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/fe/30/c9b568c3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>NullPointer</span>
  </div>
  <div class="_2_QraFYR_0">传统单体应用演变Y轴可以理解为逐渐拆解微服务化，Z轴可以理解为为特定服务进行分库分表进一步拆解，代价也逐渐扩大，收益理论上也是对等。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-30 08:12:17</div>
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
  <div class="_2_QraFYR_0">X轴扩展，将单个应用多起几套服务器进行扩展。Y轴扩展，类似于数据库读写分离，均分流量。Z轴扩展，根据地域进行负载均衡。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，Y轴的开发成本最高</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-23 14:49:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/64/9b/d1ab239e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>J.Smile</span>
  </div>
  <div class="_2_QraFYR_0">又来读了一遍。挺有感触的，技术是不断演进的，能用最简单方式解决的问题，就不要用复杂方式，AKF立方体随x,y,z依次增加了复杂度。其中x,y倾向于提升并发性能，而z倾向于克服容量窘境。分布式系统的两大难题也就体现了，高并发与分布式存储，进而引出CAP理论中分布式架构数据存储一致性和可用性的权衡。y轴功能拆分如读写分离需要解决一致性读的问题，x轴水平复制只能是无状态，避免了一致性问题，则重在提升可用性问题。z轴分库分表，既面临拆分之后数据的可用性问题，又面临可用性冗余数据复制带来的一致性问题，更加复杂了。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-05 00:02:19</div>
  </div>
</div>
</div>
</li>
</ul>