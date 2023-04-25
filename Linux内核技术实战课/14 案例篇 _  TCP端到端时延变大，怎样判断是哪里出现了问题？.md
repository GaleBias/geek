<audio title="14 案例篇 _  TCP端到端时延变大，怎样判断是哪里出现了问题？" src="https://static001.geekbang.org/resource/audio/aa/23/aa43be1fbc9bc24ef4d430db2544d923.mp3" controls="controls"></audio> 
<p>你好，我是邵亚方。</p><p>如果你是一名互联网从业者，那你对下面这个场景应该不会陌生：客户端发送请求给服务端，服务端将请求处理完后，再把响应数据发送回客户端，这就是典型的C/S（Client/Server）架构。对于这种请求-响应式的服务，我们不得不面对的问题是：</p><ul>
<li>如果客户端收到的响应时间变大了，那么这是客户端自身的问题呢，还是因为服务端处理得慢呢，又或者是因为网络有抖动呢？</li>
<li>即使我们已经明确了是服务端或者客户端的问题，那么究竟是应用程序自身引起的问题呢，还是内核导致的问题呢？</li>
<li>而且很多时候，这种问题往往是一天可能最多抖动一两次，我们很难去抓现场信息。</li>
</ul><p>为了更好地处理这类折磨人的问题，我也摸索了一些手段来进行实时追踪，既不会给应用程序和系统带来明显的开销，又可以在出现这些故障时能够把信息给抓取出来，从而帮助我们快速定位出问题所在。</p><p>因此，这节课我来分享下我在这方面的一些实践，以及解决过的一些具体案例。</p><p>当然，这些实践并不仅仅适用于这种C/S架构，对于其他应用程序，特别是对延迟比较敏感的应用程序，同样具备参考意义。比如说：</p><ul>
<li>如果我的业务运行在虚拟机里面，那怎么追踪呢？</li>
<li>如果Client和Server之间还有一个Proxy，那怎么判断是不是Proxy引起的问题呢？</li>
</ul><!-- [[[read_end]]] --><p>那么，我们就先从生产环境中C/S架构的网络抖动案例说起。</p><h2>如何分析C/S架构中的网络抖动问题？</h2><p><img src="https://static001.geekbang.org/resource/image/bf/37/bf1cd8873fff3a7528f194658753e837.jpg?wh=2172x948" alt="" title="典型的C/S架构"></p><p>上图就是一个典型的C/S架构，Client和Server之间可能经过了很复杂的网络，但是对于服务器开发者或者运维人员而言，这些中间网络可以理解为是一个黑盒，很难去获取这些网络的详细信息，更不用说到这些网络设备上去做debug了。所以，我在这里把它们都简化为了一个Router（路由器），然后Client和Server通过这个路由器来相互通信。比如互联网场景中的数据库服务（像MySQL）、http服务等都是这种架构。而当时给我们提需求来诊断网络抖动问题的也是MySQL业务。因此，接下来我们就以MySQL为例来进行具体讲解。</p><p>MySQL的业务方反馈说他们的请求偶尔会超时很长，但不清楚是什么原因引起了超时，对应到上图中，就是D点收到响应的时刻和A点发出请求的时刻，这个时间差会偶然性地有毛刺。在发生网络问题时，使用tcpdump来抓包是常用的分析手段，当你不清楚该如何来分析网络问题时，那就使用tcpdump先把事故现场保存下来吧。</p><p>如果你使用过tcpdump来分析问题，那你应该也吐槽过用tcpdump分析问题会很麻烦。如果你熟悉wireshark的话，就会相对容易一些了。但是对于大部分人而言呢，学习wirkshark的成本也是很高的，而且wireshark的命令那么多，把每一条命令都记清楚也是件很麻烦的事。</p><p>回到我们这个案例中，在MySQL发生抖动的时候，我们的业务人员也使用了tcpdump来抓包保存了现场，但是他们并不清楚该如何分析这些tcpdump信息。在我帮他们分析这些tcpdump信息的时候，也很难把tcpdump信息和业务抖动的时刻关联起来。这是因为虽然我们知道业务抖动发生的时刻，比如说21:00:00.000这个时刻，但是在这一时刻的附近可能会有非常多的TCP数据包，很难简单地依赖时间戳把二者关联起来。而且，更重要的原因是，我们知道TCP是数据流，上层业务的一个请求可能会被分为多个TCP包（TCP Segment），同样地，多个请求也可能被合并为一个TCP包。也就是说，TCP流是很难和应用数据关联起来的。这就是用tcpdump分析业务请求和响应的难点。</p><p>针对tcpdump难以分析应用协议的问题，有一个思路是，在tcpdump的时候把数据也保存下来，然后使用tcpdump再进一步去解析这些应用协议。但是你会发现，用这种方式来处理生产环境中的抖动问题是很不现实的，因为在生产环境中的 TCP 连接动辄数百上千条，我们往往并不清楚抖动发生在哪个TCP连接上。如果把这些TCP流都dump出来，data部分即使只dump与应用协议有关的数据，这对磁盘I/O也是个负担。那有什么好办法吗？</p><p>好办法还是需要和应用协议关联起来，不过我们可以把这些应用协议做一层抽象，从而可以更简单地来解析它们，甚至无需解析。对于MySQL而言呢，工具<a href="https://www.percona.com/blog/2010/08/31/introducing-tcprstat-a-tcp-response-time-tool/">tcprstat</a>就是来做这件事的。</p><p>tcprstat的大致原理是利用MySQL的request-response特征来简化对协议内容的处理。request-response是指一个请求到达MySQL后，MySQL处理完该请求，然后回response，Client侧收到response后再去发下一个request，然后MySQL收到下一个request并处理。也就是说这种模型是典型的串行方式，处理完了一个再去处理下一个。所以tcprstat就可以以数据包到达MySQL Server侧作为起始时间点，以MySQL将最后一个数据包发出去作为结束时间点，然后这二者的时间差就是RT（Response Time），这个过程大致如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/62/d6/62666eab402a7d4ed9eb41ae8a2581d6.jpg?wh=2969*1685" alt="" title="tcprstat追踪RT抖动"></p><p>tcprstat会记录request的到达时间点，以及request发出去的时间点，然后计算出RT并记录到日志中。当时我们把tcprstat部署到MySQL server侧后，发现每一个RT值都很小，并没有延迟很大的情况，所以看起来服务端并没有问题。那么问题是否发生在Client这里呢？</p><p>在我们想要把tcprstat也部署在Client侧抓取信息时，发现它只支持在Server侧部署，所以我们对tcprstat做了一些改造，让它也可以部署在Client侧。</p><p>这种改造并不麻烦，因为在Client侧解析的也是MySQL协议，只是TCP流的方向跟Server侧相反，Client侧是发请求收响应，而Server侧是收请求发响应。</p><p>在改造完成后，我们就开始部署tcprstat来抓取抖动现场了。在业务发生抖动时，通过我们抓取到的信息显示，Client在收到响应包的时候就已经发生延迟了，也就是说问题同样也不是发生在Client侧。这就有些奇怪了，既然Client和Server都没有问题，难道是网络链路出现了问题？</p><p>为了明确这一点，我们就在业务低峰期使用ping包来检查网络是否存在问题。ping了大概数小时后，我们发现ping响应时间忽然变得很大，从不到1ms的时间增大到了几十甚至上百ms，然后很快又恢复正常。</p><p>根据这个信息，我们推断某个交换机可能存在拥塞，于是就联系交换机管理人员来分析交换机。在交换机管理人员对这个链路上的交换机逐一排查后，最终定位到一台接入交换机确实有问题，它会偶然地出现排队很长的情况。而之所以MySQL反馈有抖动，其他业务没有反馈，只是因为这个接入交换机上的其他业务并不关心抖动。在交换机厂商帮忙修复了这个问题后，就再也没有出现过这种偶发性的抖动了。</p><p>这个结果看似很简单，但是分析过程还是很复杂的。因为一开始我们并不清楚问题发生在哪里，只能一步步去排查，所以这个分析过程也花费了几天的时间。</p><p>交换机引起的网络抖动问题，只是我们分析过的众多抖动案例之一。除了这类问题外，我们还分析过很多抖动是由于Client侧存在问题或者Server侧存在问题。在分析了这么多抖动问题之后，我们就开始思考，能否针对这类问题来做一个自动化分析系统呢？而且我们部署运行tcprstat后，也发现它存在一些不足之处，主要是它的性能开销略大，特别是在TCP连接数较多的情况下，它的CPU利用率甚至能够超过10%，这难以满足我们生产环境中长时间运行的需要。</p><p>tcprstat会有这么高的CPU开销，原因其实与tcpdump是类似的，它在旁路采集数据后会拷贝到用户空间来处理，这个拷贝以及处理时间就比较消耗CPU。</p><p>为了满足生产环境的需求，我们在tcprstat的基础上，做了一个更加轻量级的分析系统。</p><h2>如何轻量级地判断抖动发生在哪里？</h2><p>我们的目标是在10Gb网卡的高并发场景下尽量地降低监控开销，最好可以控制在1%以内，而且不能给业务带来明显延迟。要想降低CPU开销，很多工作就需要在内核里面来完成，就跟现在很流行的eBPF这个追踪框架类似：内核处理完所有的数据，然后将结果返回给用户空间。</p><p>能够达到这个目标的方案大致有两种：一种是使用内核模块，另一种是使用轻量级的内核追踪框架。</p><p>使用内核模块的缺点是它的安装部署会很不方便，特别是在线上内核版本非常多的情况下，比如说，我们的线上既有CentOS-6的操作系统，也有CentOS-7的操作系统，每个操作系统又各自有很多小版本，以及我们自己发布的版本。统一线上的内核版本是件很麻烦的事，这会涉及到很多变更，不太现实。所以这种现状也就决定了，使用内核模块的方式需要付出比较高的维护成本。而且内核模块的易用性也很差，这会导致业务人员和运维人员排斥使用它，从而增加它的推广难度。基于这些考虑，我们最终选择了基于systemtap这个追踪框架来开发。没有选择eBPF的原因是，它对内核版本要求较高，而我们线上很多都是CentOS-7的内核。</p><p>基于systemtap实现的追踪框架大致如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/0e/ff/0e6bf3f396e9c89f8a35376075d8c5ff.jpg?wh=2899*2027" alt="" title="TCP流的追踪"></p><p>它会追踪每一个TCP流，TCP流对应到内核里的实现就是一个struct sock实例，然后记录TCP流经过A/B/C/D这四个点的时刻，依据这几个时间点，我们就可以得到下面的结论：</p><ul>
<li>如果C-B的时间差较大，那就说明Server侧有抖动，否则是Client或网络的问题；</li>
<li>如果D-A的时间差较小，那就说明是Client侧问题，否则是Server或者网络的问题。</li>
</ul><p>这样在发生RT抖动时，我们就能够区分出抖动是发生在Client，Server，还是网络中了，这会大大提升分析定位问题的效率。在定位到问题出在哪里后，你就可以使用我们在“<a href="https://time.geekbang.org/column/article/284912">11讲</a>”、“<a href="https://time.geekbang.org/column/article/285816">12讲</a>”和“<a href="https://time.geekbang.org/column/article/286494">13讲</a>”里讲到的知识点，再去进一步分析具体的原因是什么了。</p><p>在使用systemtap的过程中我们也踩了不少坑，在这里也分享给你，希望你可以避免：</p><ul>
<li>systemtap的加载过程是一个开销很大的过程，主要是CPU的开销。因为systemtap的加载会编译systemtap脚本，这会比较耗时。你可以提前将你的systemtap脚本编译为内核模块，然后直接加载该模块来避免CPU开销；</li>
<li>systemtap有很多开销控制选项，你可以设置开销阈值来作为兜底方案，以防止异常情况下它占用太多CPU；</li>
<li>systemtap进程异常退出后可能不会卸载systemtap模块，在你发现systemtap进程退出后，你需要检查它是否也把对应的内核模块给卸载了。如果没有，那你需要手动卸载一下，以免产生不必要的问题。</li>
</ul><p>C/S架构是互联网服务中比较典型的场景，那针对其他场景我们该如何来分析问题呢？接下来我们以虚拟机这种场景为例来看一下。</p><h2>虚拟机场景下该如何判断抖动是发生在宿主机上还是虚拟机里？</h2><p>随着云计算的发展，越来越多的业务开始部署在云上，很多企业或者使用自己定制的私有云，或者使用公有云。我们也有很多业务部署在自己的私有云中，既有基于KVM的虚拟机，也有基于Kubernetes和Docker的容器。以虚拟机为例，在Server侧发生抖动的时候，业务人员还想进一步知道，抖动是发生在Server侧的虚拟机内部，还是发生在Server侧的宿主机上。要想实现这个需求，我们只需要进一步扩展，再增加新的hook点，去记录TCP流经过虚拟机的时间点就好了，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/27/c1/27e5240900409a0848842f73704d3bc1.jpg?wh=3121*2066" alt="" title="虚拟机场景下TCP流的追踪"></p><p>这样我们就可以根据F和E的时间差来判断抖动是否发生在虚拟机内部。针对这个需求，我们对tcprstat也做了类似的改造，让它可以识别出抖动是否发生在虚拟机内部。这个改造也不复杂，tcprstat默认只处理目标地址为本机的数据包，不会处理转发包，所以我们让它支持混杂模式，然后就可以处理转发包了。当然，虚拟机的具体网络配置是千差万别的，你需要根据你的实际虚拟网络配置来做调整。</p><p>总之，希望你可以举一反三，根据你的实际业务场景来做合理的数据分析，而不要局限于我们这节课所列举的这几个具体场景。</p><h2>课堂总结</h2><p>我们这堂课以典型的C/S架构为例，分析了RT发生抖动时，该如何高效地识别出问题发生在哪里。我来总结一下这节课的重点：</p><ul>
<li>tcpdump是分析网络问题必须要掌握的工具，但是用它分析问题并不容易。在你不清楚该如何分析网络问题时，你可以先使用tcpdump把现场信息保存下来；</li>
<li>TCP是数据流，如何把TCP流和具体的业务请求/响应数据包关联起来，是分析具体应用问题的前提。你需要结合你的业务模型来做合理的关联；</li>
<li>RT抖动问题是很棘手的，你需要结合你的业务模型来开发一些高效的问题分析工具。如果你使用的是Redhat或者CentOS，那么你可以考虑使用systemtap；如果是Ubuntu，你可以考虑使用lttng。</li>
</ul><h2>课后作业</h2><p>结合这堂课的第一张图，在这张图中，请问是否可以用TCP流到达B点的时刻（到达Server这台主机的时间）减去TCP流经过A点的时刻（到达Client这台主机的时间）来做为网络耗时？为什么？</p><p>结合我们在“<a href="https://time.geekbang.org/column/article/286494">13讲</a>“里讲到的RTT这个往返时延，你还可以进一步思考：是否可以使用RTT来作为这次网络耗时？为什么？欢迎你在留言区与我讨论。</p><p>感谢你的阅读，如果你认为这节课的内容有收获，也欢迎把它分享给你的朋友，我们下一讲见。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/3b/d7/9d942870.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>邵亚方</span>
  </div>
  <div class="_2_QraFYR_0">课后作业答案：<br>- 结合这堂课的第一张图，在这张图中，请问是否可以用 TCP 流到达 B 点的时刻（到达 Server 这台主机的时间）减去 TCP 流经过 A 点的时刻（到达 Client 这台主机的时间）来做为网络耗时？为什么？<br><br>很多同学在课后评论里都回答的很好了。比如：<br>“主要原因是两边的时间不统一。<br>有可能客户端的时间比服务端的快。<br>只有选取相同的时间基点，算出的差值才有意义。”<br><br><br>- 结合我们在“13 讲“里讲到的 RTT 这个往返时延，你还可以进一步思考：是否可以使用 RTT 来作为这次网络耗时？为什么？欢迎你在留言区与我讨论。<br>也不可以。<br>rtt时间不仅仅包含网络时间 还有内核软中断处理时间 另外还存在delayed ack，所以它不仅仅是网络传输耗时。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-11 14:00:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83epUBQibdMCca340MFZOe5I1GwZ0PosPIzA0TPCNzibgH00w45Zmv4jmL0mFRHMUM9FuKiclKOCBjSmsw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_circle</span>
  </div>
  <div class="_2_QraFYR_0">老师好<br>有个现象帮忙看下如何分析定位下<br>centos7.2 长连接redis ，应用日志会显示无规律的出现hmget 间歇性超过200ms，最长达6s返回的情况，持续几分钟后恢复。后续要求定位，<br>主服务器不可中断， 不准部署抓包，安装bcc 感觉工具又可能消耗资源。排除了中间网络监控的问题 ，redis服务端也未发现超时的记录，怀疑系统层面的，是否有什么思路或者工具手段监控定位呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果redis服务端未发生超时的情况，大概率是数据包阻塞在内核缓冲区太久导致的，也就是说数据包到达了内核缓冲区，但是redis没有及时读取它。没有及时读取的原因可能为：<br>1. 软中断打断redis<br>2. redis调度延迟 <br>也可能会有其他原因。<br><br>排查思路也有的。这类问题我会有ftrace的kprobe_events机制来排查。比如追踪下面几个内核函数：<br>- tcp_rcv_established ： 数据包到达内核缓冲区<br>- tcp_recvmsg： 数据包被应用读取<br>这两个时间戳相减就是数据包在内核缓冲区停留的时间。<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-29 18:15:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/aa/9a/92d2df36.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>tianfeiyu</span>
  </div>
  <div class="_2_QraFYR_0">老师，看评论里面多次提到 delayed ack，能详细给说明下这个参数的意义吗 </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: delayed ack是用来延迟发送ack包的 这样的目的是为了提高网络传输效率 因为一个ack的数据是很小的 主要的数据都是tcp header，如果发送一个ack的话，有效信息就很小。所以一般都会等待有真正数据发送时 和数据一起传输，以减少header的传输开销。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-24 14:56:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/69/62/b874af21.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>颛顼</span>
  </div>
  <div class="_2_QraFYR_0">这个文中几个时间点，是怎么联系起来，来验证是同一个请求的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 根据tcp的sequence number可以确定。就像tcpdump一样。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-15 23:22:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_6909db</span>
  </div>
  <div class="_2_QraFYR_0">l老师在思考一个问题，<br>1.整个链路的超时时间如何设置合理性，如何做实验，比如一个请求，到nginx 服务器 TCP超时时间，nginx超时时间，到Tomcat服务器TCP超时时间，Tomcat超时时间，到DB，redis,mq超时时间，</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-18 17:24:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/64/05/6989dce6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我来也</span>
  </div>
  <div class="_2_QraFYR_0">思考题2<br>RTT 是否可以作为这次网络耗时？<br>好像不行吧。<br><br>看图好像是可以，毕竟就是tcp的一去一回。<br>但是这个还取决于对方返回ACK时是否及时。<br>有时ack是会合并的。有些内核参数好像还会延迟ack的返回时机吧。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对的 rtt时间不仅仅包含网络时间 还有内核软中断处理时间 另外还存在delayed ack</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-19 14:34:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/fc/fc/1e235814.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>耿长学</span>
  </div>
  <div class="_2_QraFYR_0">唉，一个操作案例都没有，跟《Linux性能优化实战》几万的订阅量完全没法比，因为一个用心了，一个感觉很随意</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-25 18:36:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4a/15/106eaaa8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>stackWarn</span>
  </div>
  <div class="_2_QraFYR_0">1.请问作者公司线上机器都装了systemtap吗？<br>2.这个systemtap脚本和模块以后会开源吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. 部分线上机器装了systemtap，特别是centos7的系统上；在centos8的系统上，我们一般是使用ebpf，比如bcc或者boftrace。<br>2. 目前还没有开源计划，但是该模块的一些思想以及一些关键的tracepont我在这个系列课程里都讲到了 如果你看到仔细的话你会找到，利用这些tracepoint你也可以实现类似的模块。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-19 09:36:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/37/a0/032d0828.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>上杉夏香</span>
  </div>
  <div class="_2_QraFYR_0">老师，你好，捉个虫。「tcprstat 会记录 request 的到达时间点，以及 request 发出去的时间点，然后计算出 RT 并记录到日志中。」应该是「以及 response 发出去的时间点」吧？<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-04-03 23:18:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/18/fb/c1334976.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>王崧霁</span>
  </div>
  <div class="_2_QraFYR_0">不可以，tcp有重传</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-24 09:05:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/64/05/6989dce6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我来也</span>
  </div>
  <div class="_2_QraFYR_0">老师公司也在用CentOS7啊。<br>不知道有没有遇到过7.6版本的3.10.0.957内核bug，导致tcp的Send-Q居高不下的问题，哈哈。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 主要是centos7，也有centos6和8，还有一些是Ubuntu。<br>这个问题倒没有遇到过。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-19 14:36:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/64/05/6989dce6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我来也</span>
  </div>
  <div class="_2_QraFYR_0">思考题1<br>我觉得tcp流到B点的时间减出A点的时间做网络延迟 不合适。<br><br>主要原因是两边的时间不统一。<br>有可能客户端的时间比服务端的快。<br>只有选取相同的时间基点，算出的差值才有意义。<br><br>另外，因为TCP流会有很多Segment，可能一次请求就包含多个Segment，取哪一个或平均值，都不好实现。<br>而取同一个节点的时间，可以取最后一个发出去的报文和第一个发出去的报文来计算真实的时间。<br>毕竟数据只接收了一半，也是无法开始执行逻辑的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的！回答的很好。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-19 14:27:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4a/15/106eaaa8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>stackWarn</span>
  </div>
  <div class="_2_QraFYR_0">rtt统计的是tcp协议栈发出到收到的时间，包括客户端自身协议栈到发出网卡，网络传输，对方网卡接收并发到服务端协议栈，再经过返回流程到客户端协议栈的时间，rtt延时不能具体区分是哪个部分的问题。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的 另外还存在delayed ack这种情况。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-19 09:41:36</div>
  </div>
</div>
</div>
</li>
</ul>