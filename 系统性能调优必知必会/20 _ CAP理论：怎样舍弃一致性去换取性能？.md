<audio title="20 _ CAP理论：怎样舍弃一致性去换取性能？" src="https://static001.geekbang.org/resource/audio/00/d2/00e03249adac4f0c4b3dec4df91c99d2.mp3" controls="controls"></audio> 
<p>你好，我是陶辉。</p><p>上一讲我们介绍了如何通过监控找到性能瓶颈，从这一讲开始，我们将具体讨论如何通过分布式系统来提升性能。</p><p>在第一部分课程中，我介绍了多种提升单机处理性能的途径，然而，进程的性能必然受制于一台服务器上各硬件的处理能力上限。如果需要进一步地提升服务性能，那只有整合多台主机组成分布式系统才能办到。</p><p>然而，当多台主机通过网络协同处理用户请求时，如果主机上的进程含有数据状态，那么必然需要在多台主机之间跨网络同步数据，由于网络存在时延，也并不稳定，因此不可靠的数据同步操作将会增加请求的处理时延。</p><p>CAP理论指出，当数据同时存放在多个主机上时，可用性与一致性是不可兼得的。根据CAP的指导性思想，我们可以通过牺牲一致性，来提升可用性中的核心因素：性能。当然，在实践中对一致性与性能并不是非黑即白的选择，而是从概率上进行不同程度的取舍。</p><p>这一讲，我们将基于分布式系统中的经典理论，从总体上看看如何设计一致性模型，通过牺牲部分数据的一致性来提升性能。</p><h2>如何权衡性能与一致性？</h2><p>首先，这节课针对的是<strong>有状态服务</strong>的性能优化。所谓有状态服务，是指进程会在处理完请求后，仍然保存着影响下次请求结果的用户数据，而无状态服务则只从每个请求的输入参数中获取数据，在请求处理完成后并不保存任何会话信息。因此，无状态服务拥有最好的可伸缩性，但涉及数据持久化时，则必须由有状态服务处理，这是CAP理论所要解决的问题。</p><!-- [[[read_end]]] --><p>什么是CAP理论呢？这是2000年<a href="https://en.wikipedia.org/wiki/University_of_California,_Berkeley">University of California, Berkeley</a> 的计算机教授<a href="https://en.wikipedia.org/wiki/Eric_Brewer_(scientist)">Eric Brewer</a>（也是谷歌基础设施VP）提出的理论。所谓CAP，是以下3个单词的首字母缩写，它们都是分布式系统最核心的特性：</p><ul>
<li><strong>C</strong>onsistency一致性</li>
<li><strong>A</strong>vailability可用性</li>
<li><strong>P</strong>artition tolerance分区容错性</li>
</ul><p>我们通过以下3张示意图，快速理解下这3个词的意义。下图中N1、N2两台主机上运行着A进程和B进程，它们操作着同一个用户的数据（数据的初始值是V0），这里N1和N2主机就处于不同的Partition分区中，如下所示：</p><p><img src="https://static001.geekbang.org/resource/image/74/f4/744834f9a5dd04244e5f83719fb3f6f4.jpg?wh=1840*682" alt=""></p><p>正常情况下，当用户请求到达N1主机上的A进程，并将数据V0修改为V1后，A进程将会把这一修改行为同步到N2主机上的B进程，最终N1、N2上的数据都是V1，这就保持了系统的Consistency一致性。</p><p><img src="https://static001.geekbang.org/resource/image/23/25/23ab7127cbc89bcbe83c2a3669d66125.jpg?wh=1718*632" alt=""></p><p>然而，一旦N1和N2之间网络异常，数据同步行为就会失败。这时，N1和N2之间数据不一致，如果我们希望在分区间网络不通的情况下，N2能够继续为用户提供服务，就必须容忍数据的不一致，此时系统的Availability可用性更高，系统的并发处理能力更强，比如Cassandra数据库。</p><p><img src="https://static001.geekbang.org/resource/image/a7/fb/a7302dc2491229a019aa27f043ba08fb.jpg?wh=1718*622" alt=""></p><p>反之，如果A、B进程一旦发现数据同步失败，那么B进程自动拒绝新请求，仅由A进程独立提供服务，那么虽然降低了系统的可用性，但保证了更强的一致性，比如MySQL的主备同步模式。</p><p>这就是CAP中三者只能取其二的简要示意，对于这一理论，2002年MIT的<a href="http://lpd.epfl.ch/sgilbert/">Seth Gilbert</a> 、 <a href="http://people.csail.mit.edu/lynch/">Nancy Lynch</a> 在<a href="https://dl.acm.org/doi/pdf/10.1145/564585.564601">这篇论文</a>中，证明了这一理论。当然，<a href="https://en.wikipedia.org/wiki/High_availability">可用性</a>是一个很大的概念，它描述了分布式系统的持续服务能力，如下表所示：</p><p><img src="https://static001.geekbang.org/resource/image/0c/df/0c11b949c35e4cce86233843ccb152df.jpg?wh=1392*602" alt=""></p><p>当用户、流量不断增长时，系统的性能将变成衡量可用性的关键因素。当我们希望拥有更强的性能时，就不得不牺牲数据的一致性。当然，一致性并不是只有是和否这两种属性，我们既可以从时间维度上设计短暂不一致的同步模型，也可以从空间维度上为不含有因果、时序关系的用户数据设计并发模型。</p><p>在学术上，通常会按照2个维度对<a href="https://en.wikipedia.org/wiki/Consistency_model#Client-centric_consistency_models%5B19%5D">一致性模型</a>分类。首先是从数据出发设计出的一致性模型，比如顺序一致性必须遵照读写操作的次序来保持一致性，而弱一些的因果一致性则允许不具备因果关系的读写操作并发执行。其次是从用户出发设计出的一致性模型，比如单调读一致性保证客户端不会读取到旧值，而单调写一致性则保证写操作是串行的。</p><p>实际工程中一致性与可用性的边界会模糊很多，因此又有了<a href="https://en.wikipedia.org/wiki/Eventual_consistency">最终一致性</a>这样一个概念，这个“最终”究竟是多久，将由业务特性、网络故障率等因素综合决定。伴随最终一致性的是BASE理论：</p><ul>
<li><strong>B</strong>asically <strong>A</strong>vailable基本可用性</li>
<li><strong>S</strong>oft state软状态</li>
<li><strong>E</strong>ventually consistent最终一致性</li>
</ul><p>BASE与CAP一样并没有很精确的定义，它们最主要的用途是从大方向上给你指导性的思想。接下来，我们看看如何通过最终一致性来提升性能。</p><h2>怎样舍弃一致性提升性能？</h2><p>服务的性能，主要体现在请求的时延和系统的并发性这两个方面，而它们都与上面提到的最终一致性有关。我通常会把分布式系统分为纵向、横向两个维度，其中<strong>纵向是请求的处理路径，横向则是同类服务之间的数据同步路径。</strong>这样，在纵向上在离客户端更近的位置增加数据的副本，并把它存放在处理速度更快的物理介质上，就可以作为缓存降低请求的时延；而在横向上对数据增加副本，并在这些主机间同步数据，这样工作在数据副本上的进程也可以同时对客户端提供服务，这就增加了系统的并发性，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/5a/06/5a6dbac922c500eea108d374dccc6406.png?wh=1086*733" alt=""></p><p>我们先来看纵向上的缓存，是如何在降低请求处理时延时，保持最终一致性的。</p><p>缓存可以由读、写两个操作触发更新，<a href="https://time.geekbang.org/column/article/242667">[第15课]</a> 介绍过的HTTP私有缓存，就是工作在浏览器上基于读操作触发的缓存。</p><p><img src="https://static001.geekbang.org/resource/image/9d/ab/9dea133d832d8b7ab642bb74b48502ab.png?wh=1007*676" alt=""></p><p>工作在代理服务器上的HTTP共享缓存，也是由读操作触发的。这些缓存与源服务器上的数据最终是一致的，但在CacheControl等HTTP头部指定的时间内，缓存完全有可能与源数据不同，这就是牺牲一致性来提升性能的典型例子。</p><p>事实上，也有很多以写操作触发缓存更新的设计，它们通常又分为write back和write through两种模式。其中，<strong>write back牺牲了更多的一致性，但带来了更低的请求时延。<strong>比如<a href="https://time.geekbang.org/column/article/232676">[第4课]</a> 介绍过的Linux磁盘高速缓存就采用了write back这种设计，它虽然是单机内的一种缓存设计，但在分布式系统中缓存的设计方式也是一样的。而</strong>write through会在更新数据成功后再更新缓存，虽然带来了很好的一致性，但写操作的时延会更久</strong>，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/de/ac/de0ed171a392b63a87af28b9aa6ec7ac.png?wh=1087*751" alt=""></p><p>write through的一致性非常好，它常常是我们的首选设计。然而，一旦缓存到源数据的路径很长、延时很高的时候，就值得考虑write back模式，此时一致性模型虽然复杂了许多，但可以带来显著的性能提升。比如机械磁盘相对内存延时高了很多，因此磁盘高速缓存会采用write back模式。</p><p>虽然缓存也可以在一定程度上增加系统的并发处理连接，但这更多是缘于缓存采用了更快的算法以及存储介质带来的收益。从水平方向上，在更多的主机上添加数据副本，并在其上用新的进程提供服务，这才是提升系统并发处理能力最有效的手段。此时，进程间同步数据主要包含两种方式：同步方式以及异步方式，前者会在每个更新请求完成前，将数据成功同步到每个副本后，请求才会处理完成，而后者则会在处理完请求后，才开始异步地同步数据到副本中，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/d7/f6/d770cf24f61d671ab1f0c1a6170627f6.png?wh=733*728" alt=""></p><p>同步方式下系统的一致性最好，但由于请求的处理时延包含了副本间的通讯时长，所以性能并不好。而异步方式下系统的一致性要差，特别是在主进程宕机后，副本上的进程很容易出现数据丢失，但异步方式的性能要好得多，这不只由于更新请求的返回更快，而且异步化后主进程可以基于合并、批量操作等技巧，进行效率更高的数据同步。比如MySQL的主备模式下，默认就使用异步方式同步数据，当然你也可以改为同步方式，包括Full-synchronous Replication（同步所有数据副本后请求才能返回）以及Semi-synchronous Replication（仅成功同步1个副本后请求就可以返回）两种方式。</p><p>当然，在扩容、宕机恢复等场景下，副本之间的数据严重不一致，如果仍然基于单个操作同步数据，时间会很久，性能很差。此时应结合定期更新的Snapshot快照，以及实时的Oplog操作日志，协作同步数据，这样效率会高很多。其中，快照是截止时间T0时所有数据的状态，而操作日志则是T0时间到当下T1之间的所有更新操作，这样，副本载入快照恢复至T0时刻的数据后，再通过有限的操作日志与主进程保持一致。</p><h2>小结</h2><p>这一讲我们介绍了分布式系统中的CAP理论，以及根据CAP如何通过一致性来换取性能。</p><p>CAP理论指出，可用性、分区容错性、一致性三者只能取其二，因此当分布式系统需要服务更多的用户时，只能舍弃一致性，换取可用性中的性能因子。当然，性能与一致性并不是简单的二选一，而是需要你根据网络时延、故障概率设计出一致性模型，在提供高性能的同时，保持时间、空间上可接受的最终一致性。</p><p>具体的设计方法，可以分为纵向上添加缓存，横向上添加副本进程两种做法。对于缓存的更新，write through模式保持一致性更容易，但写请求的时延偏高，而一致性模型更复杂的write back模式时延则更低，适用于性能要求很高的场景。</p><p>提升系统并发性可以通过添加数据副本，并让工作在副本上的进程同时对用户提供服务。副本间的数据同步是由写请求触发的，其中包括同步、异步两种同步方式。异步方式的最终一致性要差一些，但写请求的处理时延更快。在宕机恢复、系统扩容时，采用快照加操作日志的方式，系统的性能会好很多。</p><h2>思考题</h2><p>最后留给你一道讨论题，当数据副本都在同一个IDC内部时，网络时延很低，带宽大而且近乎免费使用。然而，一旦数据副本跨IDC后，特别是IDC之间物理距离很遥远，那么网络同步就变得很昂贵。这两种不同场景下，你是如何考虑一致性模型的？它与性能之间有多大的影响？期待你的分享。</p><p>感谢阅读，如果你觉得这节课对你有一些启发，也欢迎把它分享给你的朋友。</p>
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
  <div class="_2_QraFYR_0">Idc内部因为时延小，网络稳定，可采用同步模式；不同Idc之间，可采用异步模式，以保证各Idc的性能，实现最终的一致性。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-22 08:53:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/19/f9/62ae32d7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Ken</span>
  </div>
  <div class="_2_QraFYR_0">老师，对于多机房往往存储也会做双活，在这个基础上，是否应用&#47;中间件的数据同步也能依赖于存储双活实现呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-30 08:35:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/34/df/64e3d533.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>leslie</span>
  </div>
  <div class="_2_QraFYR_0">老师说的物理模型其实是主从与灾备：2地3中心。<br>其实同城主从速率还好，我觉得这块其实云服务在这块还是做的不错的。<br>异地网络问题确实没有过多的好的选择：可能我们能做的选择只是把某些关键设备放在这块相对好的城市，曾经某金融交易所就是因此把核心机房搬到某一线城市。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-29 08:50:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/bc/25/1c92a90c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>tt</span>
  </div>
  <div class="_2_QraFYR_0">如果把缓存和对应的源看做异构的副本，那write backq就能对应于异步复制，write through就能对应于同步复制了。这样忽略底层细节，只从数据一致性的角度出发，就可以统一记忆这两种处理数据的方法了。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-22 17:07:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/6f/a6/8a9cbc4f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>新@青春</span>
  </div>
  <div class="_2_QraFYR_0">对于某些一致性要求特别高的请求，采用同步比较合适，性能相对而言重要性轻，对于一致性要求不高的请求，采用异步，比较比较合适，提高用户的体验。需要对功能进行拆分（按一致性的重要性）。不知这样理解，老师认为会有些偏吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-25 23:21:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83eo2SjCeylLv0P3Glle5277kA4b8cAuxr1NrC0njPKEqzSpB8IEicHB29GicFFwG1qiaxs4hxRiaBmoibVw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>阳仔</span>
  </div>
  <div class="_2_QraFYR_0">相同idc内可以考虑强一致性cp，不同idc间选择ap，达到最终一致性就可以</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-25 11:55:11</div>
  </div>
</div>
</div>
</li>
</ul>