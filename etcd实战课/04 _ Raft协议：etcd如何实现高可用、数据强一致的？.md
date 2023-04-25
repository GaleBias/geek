<audio title="04 _ Raft协议：etcd如何实现高可用、数据强一致的？" src="https://static001.geekbang.org/resource/audio/b0/a1/b093f933e3244f08d0c5a9f0e30dbda1.mp3" controls="controls"></audio> 
<p>你好，我是唐聪。</p><p>在前面的etcd读写流程学习中，我和你多次提到了etcd是基于Raft协议实现高可用、数据强一致性的。</p><p>那么etcd是如何基于Raft来实现高可用、数据强一致性的呢？</p><p>这节课我们就以上一节中的hello写请求为案例，深入分析etcd在遇到Leader节点crash等异常后，Follower节点如何快速感知到异常，并高效选举出新的Leader，对外提供高可用服务的。</p><p>同时，我将通过一个日志复制整体流程图，为你介绍etcd如何保障各节点数据一致性，并介绍Raft算法为了确保数据一致性、完整性，对Leader选举和日志复制所增加的一系列安全规则。希望通过这节课，让你了解etcd在节点故障、网络分区等异常场景下是如何基于Raft算法实现高可用、数据强一致的。</p><h2>如何避免单点故障</h2><p>在介绍Raft算法之前，我们首先了解下它的诞生背景，Raft解决了分布式系统什么痛点呢？</p><p>首先我们回想下，早期我们使用的数据存储服务，它们往往是部署在单节点上的。但是单节点存在单点故障，一宕机就整个服务不可用，对业务影响非常大。</p><p>随后，为了解决单点问题，软件系统工程师引入了数据复制技术，实现多副本。通过数据复制方案，一方面我们可以提高服务可用性，避免单点故障。另一方面，多副本可以提升读吞吐量、甚至就近部署在业务所在的地理位置，降低访问延迟。</p><!-- [[[read_end]]] --><p><strong>多副本复制是如何实现的呢？</strong></p><p>多副本常用的技术方案主要有主从复制和去中心化复制。主从复制，又分为全同步复制、异步复制、半同步复制，比如MySQL/Redis单机主备版就基于主从复制实现的。</p><p><strong>全同步复制</strong>是指主收到一个写请求后，必须等待全部从节点确认返回后，才能返回给客户端成功。因此如果一个从节点故障，整个系统就会不可用。这种方案为了保证多副本的一致性，而牺牲了可用性，一般使用不多。</p><p><strong>异步复制</strong>是指主收到一个写请求后，可及时返回给client，异步将请求转发给各个副本，若还未将请求转发到副本前就故障了，则可能导致数据丢失，但是可用性是最高的。</p><p><strong>半同步复制</strong>介于全同步复制、异步复制之间，它是指主收到一个写请求后，至少有一个副本接收数据后，就可以返回给客户端成功，在数据一致性、可用性上实现了平衡和取舍。</p><p>跟主从复制相反的就是<strong>去中心化复制</strong>，它是指在一个n副本节点集群中，任意节点都可接受写请求，但一个成功的写入需要w个节点确认，读取也必须查询至少r个节点。</p><p>你可以根据实际业务场景对数据一致性的敏感度，设置合适w/r参数。比如你希望每次写入后，任意client都能读取到新值，如果n是3个副本，你可以将w和r设置为2，这样当你读两个节点时候，必有一个节点含有最近写入的新值，这种读我们称之为法定票数读（quorum read）。</p><p>AWS的Dynamo系统就是基于去中心化的复制算法实现的。它的优点是节点角色都是平等的，降低运维复杂度，可用性更高。但是缺陷是去中心化复制，势必会导致各种写入冲突，业务需要关注冲突处理。</p><p>从以上分析中，为了解决单点故障，从而引入了多副本。但基于复制算法实现的数据库，为了保证服务可用性，大多数提供的是最终一致性，总而言之，不管是主从复制还是异步复制，都存在一定的缺陷。</p><p><strong>如何解决以上复制算法的困境呢？</strong></p><p>答案就是共识算法，它最早是基于复制状态机背景下提出来的。 下图是复制状态机的结构（引用自Raft paper）， 它由共识模块、日志模块、状态机组成。通过共识模块保证各个节点日志的一致性，然后各个节点基于同样的日志、顺序执行指令，最终各个复制状态机的结果实现一致。</p><p><img src="https://static001.geekbang.org/resource/image/3y/eb/3yy3fbc1ab564e3af9ac9223db1435eb.png?wh=605*319" alt=""></p><p>共识算法的祖师爷是Paxos， 但是由于它过于复杂，难于理解，工程实践上也较难落地，导致在工程界落地较慢。standford大学的Diego提出的Raft算法正是为了可理解性、易实现而诞生的，它通过问题分解，将复杂的共识问题拆分成三个子问题，分别是：</p><ul>
<li>Leader选举，Leader故障后集群能快速选出新Leader；</li>
<li>日志复制， 集群只有Leader能写入日志， Leader负责复制日志到Follower节点，并强制Follower节点与自己保持相同；</li>
<li>安全性，一个任期内集群只能产生一个Leader、已提交的日志条目在发生Leader选举时，一定会存在更高任期的新Leader日志中、各个节点的状态机应用的任意位置的日志条目内容应一样等。</li>
</ul><p>下面我以实际场景为案例，分别和你深入讨论这三个子问题，看看Raft是如何解决这三个问题，以及在etcd中的应用实现。</p><h2>Leader选举</h2><p>当etcd server收到client发起的put hello写请求后，KV模块会向Raft模块提交一个put提案，我们知道只有集群Leader才能处理写提案，如果此时集群中无Leader， 整个请求就会超时。</p><p>那么Leader是怎么诞生的呢？Leader crash之后其他节点如何竞选呢？</p><p>首先在Raft协议中它定义了集群中的如下节点状态，任何时刻，每个节点肯定处于其中一个状态：</p><ul>
<li>Follower，跟随者， 同步从Leader收到的日志，etcd启动的时候默认为此状态；</li>
<li>Candidate，竞选者，可以发起Leader选举；</li>
<li>Leader，集群领导者， 唯一性，拥有同步日志的特权，需定时广播心跳给Follower节点，以维持领导者身份。</li>
</ul><p><img src="https://static001.geekbang.org/resource/image/a5/09/a5a210eec289d8e4e363255906391009.png?wh=1808*978" alt=""></p><p>上图是节点状态变化关系图，当Follower节点接收Leader节点心跳消息超时后，它会转变成Candidate节点，并可发起竞选Leader投票，若获得集群多数节点的支持后，它就可转变成Leader节点。</p><p>下面我以Leader crash场景为案例，给你详细介绍一下etcd Leader选举原理。</p><p>假设集群总共3个节点，A节点为Leader，B、C节点为Follower。</p><p><img src="https://static001.geekbang.org/resource/image/a2/59/a20ba5b17de79d6ce8c78a712a364359.png?wh=1920*942" alt=""></p><p>如上Leader选举图左边部分所示， 正常情况下，Leader节点会按照心跳间隔时间，定时广播心跳消息（MsgHeartbeat消息）给Follower节点，以维持Leader身份。 Follower收到后回复心跳应答包消息（MsgHeartbeatResp消息）给Leader。</p><p>细心的你可能注意到上图中的Leader节点下方有一个任期号（term）， 它具有什么样的作用呢？</p><p>这是因为Raft将时间划分成一个个任期，任期用连续的整数表示，每个任期从一次选举开始，赢得选举的节点在该任期内充当Leader的职责，随着时间的消逝，集群可能会发生新的选举，任期号也会单调递增。</p><p>通过任期号，可以比较各个节点的数据新旧、识别过期的Leader等，它在Raft算法中充当逻辑时钟，发挥着重要作用。</p><p>了解完正常情况下Leader维持身份的原理后，我们再看异常情况下，也就Leader crash后，etcd是如何自愈的呢？</p><p>如上Leader选举图右边部分所示，当Leader节点异常后，Follower节点会接收Leader的心跳消息超时，当超时时间大于竞选超时时间后，它们会进入Candidate状态。</p><p>这里要提醒下你，etcd默认心跳间隔时间（heartbeat-interval）是100ms， 默认竞选超时时间（election timeout）是1000ms， 你需要根据实际部署环境、业务场景适当调优，否则就很可能会频繁发生Leader选举切换，导致服务稳定性下降，后面我们实践篇会再详细介绍。</p><p>进入Candidate状态的节点，会立即发起选举流程，自增任期号，投票给自己，并向其他节点发送竞选Leader投票消息（MsgVote）。</p><p>C节点收到Follower B节点竞选Leader消息后，这时候可能会出现如下两种情况：</p><ul>
<li>第一种情况是C节点判断B节点的数据至少和自己一样新、B节点任期号大于C当前任期号、并且C未投票给其他候选者，就可投票给B。这时B节点获得了集群多数节点支持，于是成为了新的Leader。</li>
<li>第二种情况是，恰好C也心跳超时超过竞选时间了，它也发起了选举，并投票给了自己，那么它将拒绝投票给B，这时谁也无法获取集群多数派支持，只能等待竞选超时，开启新一轮选举。Raft为了优化选票被瓜分导致选举失败的问题，引入了随机数，每个节点等待发起选举的时间点不一致，优雅的解决了潜在的竞选活锁，同时易于理解。</li>
</ul><p>Leader选出来后，它什么时候又会变成Follower状态呢？ 从上面的状态转换关系图中你可以看到，如果现有Leader发现了新的Leader任期号，那么它就需要转换到Follower节点。A节点crash后，再次启动成为Follower，假设因为网络问题无法连通B、C节点，这时候根据状态图，我们知道它将不停自增任期号，发起选举。等A节点网络异常恢复后，那么现有Leader收到了新的任期号，就会触发新一轮Leader选举，影响服务的可用性。</p><p>然而A节点的数据是远远落后B、C的，是无法获得集群Leader地位的，发起的选举无效且对集群稳定性有伤害。</p><p>那如何避免以上场景中的无效的选举呢？</p><p>在etcd 3.4中，etcd引入了一个PreVote参数（默认false），可以用来启用PreCandidate状态解决此问题，如下图所示。Follower在转换成Candidate状态前，先进入PreCandidate状态，不自增任期号， 发起预投票。若获得集群多数节点认可，确定有概率成为Leader才能进入Candidate状态，发起选举流程。</p><p><img src="https://static001.geekbang.org/resource/image/16/06/169ae84055byya38b616d2e71cfb9706.png?wh=1920*971" alt=""></p><p>因A节点数据落后较多，预投票请求无法获得多数节点认可，因此它就不会进入Candidate状态，导致集群重新选举。</p><p>这就是Raft Leader选举核心原理，使用心跳机制维持Leader身份、触发Leader选举，etcd基于它实现了高可用，只要集群一半以上节点存活、可相互通信，Leader宕机后，就能快速选举出新的Leader，继续对外提供服务。</p><h2>日志复制</h2><p>假设在上面的Leader选举流程中，B成为了新的Leader，它收到put提案后，它是如何将日志同步给Follower节点的呢？ 什么时候它可以确定一个日志条目为已提交，通知etcdserver模块应用日志条目指令到状态机呢？</p><p>这就涉及到Raft日志复制原理，为了帮助你理解日志复制的原理，下面我给你画了一幅Leader收到put请求后，向Follower节点复制日志的整体流程图，简称流程图，在图中我用序号给你标识了核心流程。</p><p>我将结合流程图、后面的Raft的日志图和你简要分析Leader B收到put hello为world的请求后，是如何将此请求同步给其他Follower节点的。</p><p><img src="https://static001.geekbang.org/resource/image/a5/83/a57a990cff7ca0254368d6351ae5b983.png?wh=1920*1327" alt=""></p><p>首先Leader收到client的请求后，etcdserver的KV模块会向Raft模块提交一个put hello为world提案消息（流程图中的序号2流程），  它的消息类型是MsgProp。</p><p>Leader的Raft模块获取到MsgProp提案消息后，为此提案生成一个日志条目，追加到未持久化、不稳定的Raft日志中，随后会遍历集群Follower列表和进度信息，为每个Follower生成追加（MsgApp）类型的RPC消息，此消息中包含待复制给Follower的日志条目。</p><p>这里就出现两个疑问了。第一，Leader是如何知道从哪个索引位置发送日志条目给Follower，以及Follower已复制的日志最大索引是多少呢？第二，日志条目什么时候才会追加到稳定的Raft日志中呢？Raft模块负责持久化吗？</p><p>首先我来给你介绍下什么是Raft日志。下图是Raft日志复制过程中的日志细节图，简称日志图1。</p><p>在日志图中，最上方的是日志条目序号/索引，日志由有序号标识的一个个条目组成，每个日志条目内容保存了Leader任期号和提案内容。最开始的时候，A节点是Leader，任期号为1，A节点crash后，B节点通过选举成为新的Leader， 任期号为2。</p><p>日志图1描述的是hello日志条目未提交前的各节点Raft日志状态。</p><p><img src="https://static001.geekbang.org/resource/image/3d/87/3dd2b6042e6e0cc86f96f24764b7f587.png?wh=1920*1003" alt=""></p><p>我们现在就可以来回答第一个疑问了。Leader会维护两个核心字段来追踪各个Follower的进度信息，一个字段是NextIndex， 它表示Leader发送给Follower节点的下一个日志条目索引。一个字段是MatchIndex， 它表示Follower节点已复制的最大日志条目的索引，比如上面的日志图1中C节点的已复制最大日志条目索引为5，A节点为4。</p><p>我们再看第二个疑问。etcd Raft模块设计实现上抽象了网络、存储、日志等模块，它本身并不会进行网络、存储相关的操作，上层应用需结合自己业务场景选择内置的模块或自定义实现网络、存储、日志等模块。</p><p>上层应用通过Raft模块的输出接口（如Ready结构），获取到待持久化的日志条目和待发送给Peer节点的消息后（如上面的MsgApp日志消息），需持久化日志条目到自定义的WAL模块，通过自定义的网络模块将消息发送给Peer节点。</p><p>日志条目持久化到稳定存储中后，这时候你就可以将日志条目追加到稳定的Raft日志中。即便这个日志是内存存储，节点重启时也不会丢失任何日志条目，因为WAL模块已持久化此日志条目，可通过它重建Raft日志。</p><p>etcd Raft模块提供了一个内置的内存存储（MemoryStorage）模块实现，etcd使用的就是它，Raft日志条目保存在内存中。网络模块并未提供内置的实现，etcd基于HTTP协议实现了peer节点间的网络通信，并根据消息类型，支持选择pipeline、stream等模式发送，显著提高了网络吞吐量、降低了延时。</p><p>解答完以上两个疑问后，我们继续分析etcd是如何与Raft模块交互，获取待持久化的日志条目和发送给peer节点的消息。</p><p>正如刚刚讲到的，Raft模块输入是Msg消息，输出是一个Ready结构，它包含待持久化的日志条目、发送给peer节点的消息、已提交的日志条目内容、线性查询结果等Raft输出核心信息。</p><p>etcdserver模块通过channel从Raft模块获取到Ready结构后（流程图中的序号3流程），因B节点是Leader，它首先会通过基于HTTP协议的网络模块将追加日志条目消息（MsgApp）广播给Follower，并同时将待持久化的日志条目持久化到WAL文件中（流程图中的序号4流程），最后将日志条目追加到稳定的Raft日志存储中（流程图中的序号5流程）。</p><p>各个Follower收到追加日志条目（MsgApp）消息，并通过安全检查后，它会持久化消息到WAL日志中，并将消息追加到Raft日志存储，随后会向Leader回复一个应答追加日志条目（MsgAppResp）的消息，告知Leader当前已复制的日志最大索引（流程图中的序号6流程）。</p><p>Leader收到应答追加日志条目（MsgAppResp）消息后，会将Follower回复的已复制日志最大索引更新到跟踪Follower进展的Match Index字段，如下面的日志图2中的Follower C MatchIndex为6，Follower A为5，日志图2描述的是hello日志条目提交后的各节点Raft日志状态。</p><p><img src="https://static001.geekbang.org/resource/image/eb/63/ebbf739a94f9300a85f21da7e55f1e63.png?wh=1920*994" alt=""></p><p>最后Leader根据Follower的MatchIndex信息，计算出一个位置，如果这个位置已经被一半以上节点持久化，那么这个位置之前的日志条目都可以被标记为已提交。</p><p>在我们这个案例中日志图2里6号索引位置之前的日志条目已被多数节点复制，那么他们状态都可被设置为已提交。Leader可通过在发送心跳消息（MsgHeartbeat）给Follower节点时，告知它已经提交的日志索引位置。</p><p>最后各个节点的etcdserver模块，可通过channel从Raft模块获取到已提交的日志条目（流程图中的序号7流程），应用日志条目内容到存储状态机（流程图中的序号8流程），返回结果给client。</p><p>通过以上流程，Leader就完成了同步日志条目给Follower的任务，一个日志条目被确定为已提交的前提是，它需要被Leader同步到一半以上节点上。以上就是etcd Raft日志复制的核心原理。</p><h2>安全性</h2><p>介绍完Leader选举和日志复制后，最后我们再来看看Raft是如何保证安全性的。</p><p>如果在上面的日志图2中，Leader B在应用日志指令put hello为world到状态机，并返回给client成功后，突然crash了，那么Follower A和C是否都有资格选举成为Leader呢？</p><p>从日志图2中我们可以看到，如果A成为了Leader那么就会导致数据丢失，因为它并未含有刚刚client已经写入成功的put hello为world指令。</p><p>Raft算法如何确保面对这类问题时不丢数据和各节点数据一致性呢？</p><p>这就是Raft的第三个子问题需要解决的。Raft通过给选举和日志复制增加一系列规则，来实现Raft算法的安全性。</p><h3>选举规则</h3><p>当节点收到选举投票的时候，需检查候选者的最后一条日志中的任期号，若小于自己则拒绝投票。如果任期号相同，日志却比自己短，也拒绝为其投票。</p><p>比如在日志图2中，Folllower A和C任期号相同，但是Follower C的数据比Follower A要长，那么在选举的时候，Follower C将拒绝投票给A， 因为它的数据不是最新的。</p><p>同时，对于一个给定的任期号，最多只会有一个leader被选举出来，leader的诞生需获得集群一半以上的节点支持。每个节点在同一个任期内只能为一个节点投票，节点需要将投票信息持久化，防止异常重启后再投票给其他节点。</p><p>通过以上规则就可防止日志图2中的Follower A节点成为Leader。</p><h3>日志复制规则</h3><p>在日志图2中，Leader B返回给client成功后若突然crash了，此时可能还并未将6号日志条目已提交的消息通知到Follower A和C，那么如何确保6号日志条目不被新Leader删除呢？ 同时在etcd集群运行过程中，Leader节点若频繁发生crash后，可能会导致Follower节点与Leader节点日志条目冲突，如何保证各个节点的同Raft日志位置含有同样的日志条目？</p><p>以上各类异常场景的安全性是通过Raft算法中的Leader完全特性和只附加原则、日志匹配等安全机制来保证的。</p><p><strong>Leader完全特性</strong>是指如果某个日志条目在某个任期号中已经被提交，那么这个条目必然出现在更大任期号的所有Leader中。</p><p>Leader只能追加日志条目，不能删除已持久化的日志条目（<strong>只附加原则</strong>），因此Follower C成为新Leader后，会将前任的6号日志条目复制到A节点。</p><p>为了保证各个节点日志一致性，Raft算法在追加日志的时候，引入了一致性检查。Leader在发送追加日志RPC消息时，会把新的日志条目紧接着之前的条目的索引位置和任期号包含在里面。Follower节点会检查相同索引位置的任期号是否与Leader一致，一致才能追加，这就是<strong>日志匹配特性</strong>。它本质上是一种归纳法，一开始日志空满足匹配特性，随后每增加一个日志条目时，都要求上一个日志条目信息与Leader一致，那么最终整个日志集肯定是一致的。</p><p>通过以上的Leader选举限制、Leader完全特性、只附加原则、日志匹配等安全特性，Raft就实现了一个可严格通过数学反证法、归纳法证明的高可用、一致性算法，为etcd的安全性保驾护航。</p><h2>小结</h2><p>最后我们来小结下今天的内容。我从如何避免单点故障说起，给你介绍了分布式系统中实现多副本技术的一系列方案，从主从复制到去中心化复制、再到状态机、共识算法，让你了解了各个方案的优缺点，以及主流存储产品的选择。</p><p>Raft虽然诞生晚，但它却是共识算法里面在工程界应用最广泛的。它将一个复杂问题拆分成三个子问题，分别是Leader选举、日志复制和安全性。</p><p>Raft通过心跳机制、随机化等实现了Leader选举，只要集群半数以上节点存活可相互通信，etcd就可对外提供高可用服务。</p><p>Raft日志复制确保了etcd多节点间的数据一致性，我通过一个etcd日志复制整体流程图为你详细介绍了etcd写请求从提交到Raft模块，到被应用到状态机执行的各个流程，剖析了日志复制的核心原理，即一个日志条目只有被Leader同步到一半以上节点上，此日志条目才能称之为成功复制、已提交。Raft的安全性，通过对Leader选举和日志复制增加一系列规则，保证了整个集群的一致性、完整性。</p><h2>思考题</h2><p>好了，这节课到这里也就结束了，最后我给你留了一个思考题。</p><p>哪些场景会出现Follower日志与Leader冲突，我们知道etcd WAL模块只能持续追加日志条目，那冲突后Follower是如何删除无效的日志条目呢？</p><p>感谢你阅读，也欢迎你把这篇文章分享给更多的朋友一起阅读。</p><h2>03思考题答案</h2><p>在上一节课中，我给大家留了一个思考题：expensive request是否影响写请求性能。要搞懂这个问题，我们得回顾下etcd读写性能优化历史。</p><p>在etcd 3.0中，线性读请求需要走一遍Raft协议持久化到WAL日志中，因此读性能非常差，写请求肯定也会被影响。</p><p>在etcd 3.1中，引入了ReadIndex机制提升读性能，读请求无需再持久化到WAL中。</p><p>在etcd 3.2中,  优化思路转移到了MVCC/boltdb模块，boltdb的事务锁由粗粒度的互斥锁，优化成读写锁，实现“N reads or 1 write”的并行，同时引入了buffer来提升吞吐量。问题就出在这个buffer，读事务会加读锁，写事务结束时要升级锁更新buffer，但是expensive request导致读事务长时间持有锁，最终导致写请求超时。</p><p>在etcd 3.4中，实现了全并发读，创建读事务的时候会全量拷贝buffer, 读写事务不再因为buffer阻塞，大大缓解了expensive request对etcd性能的影响。尤其是Kubernetes List Pod等资源场景来说，etcd稳定性显著提升。在后面的实践篇中，我会和你再次深入讨论以上问题。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/97/d3/239aa4d5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>科精</span>
  </div>
  <div class="_2_QraFYR_0">Raft分布式算法，打开连接玩1分钟就能学会<br>http:&#47;&#47;kailing.pub&#47;raft&#47;index.html</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-20 10:41:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/3e/e7/261711a5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>blentle</span>
  </div>
  <div class="_2_QraFYR_0">老师，如果一个日志完整度相对较高的节点因为自己随机时间比其他节点的长，没能最先发起竞选，其他节点当上leader后同步自己的日志岂不是冲突了？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你所说的这个日志完整度相对较高的节点，投票时有竞选规则安全限制，如果它的节点比较新会拒绝投票，至于最终先发起选举的节点能否赢得选举，要看其他节点数据情况，如果多数节点的数据比它新，那么先发起选举的节点就无法获得多数选票，如果5个节点中，只有一个节点数据比较长，那的确会被覆盖，但是这是安全的，说明这个数据并未被集群节点多数确认</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-27 08:49:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/46/0f/f6cfc659.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>mckee</span>
  </div>
  <div class="_2_QraFYR_0">思考题：<br>1.哪些场景会出现 Follower 日志与 Leader 冲突？<br>leader崩溃的情况下可能(如老的leader可能还没有完全复制所有的日志条目)，如果leader和follower出现持续崩溃会加剧这个现象。follower可能会丢失一些在新的leader中有的日志条目，他也可能拥有一些leader没有的日志条目，或者两者都发生。<br>2.follower如何删除无效日志？<br>leader处理不一致是通过强制follower直接复制自己的日志来解决了。因此在follower中的冲突的日志条目会被leader的日志覆盖。leader会记录follower的日志复制进度nextIndex，如果follower在追加日志时一致性检查失败，就会拒绝请求，此时leader就会减小 nextIndex 值并进行重试，最终在某个位置让follower跟leader一致。<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 怒赞，优秀，还需要补充下为什么WAL模块只能通过追加日志，那它是如何删除已持久化的废弃日志条目呢？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-31 02:19:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/9d/84/171b2221.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jeffery</span>
  </div>
  <div class="_2_QraFYR_0">讲的太好了、图文并貌、形象生动、raft这章就够本了！老师问下.es 也用raft！为啥会出现数据不一致性？部署etcd高可用有运维有最佳实践建议吗？谢谢老师</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢你的认可，有收获就好，数据不一致原因和高可用运维最佳实践，实践篇都有，稍等，一起学下去，加油，保证让你收获满满</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-28 18:26:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/41/38/4f89095b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>写点啥呢</span>
  </div>
  <div class="_2_QraFYR_0">关于选举过程和节点崩溃后恢复有几个问题请教老师：<br>1. 如果多数follower崩溃后重启恢复（比如极端情况只剩下Leader其它follower同时重启），根据选举规则是不是会出现重启follower占多数投票给了一个数据不是最新的节点而导致数据丢失。我理解这种情况不满足法定票算法前提，所以是无法保障数据一致的。<br>2. 对于少数节点崩溃恢复后，它是如何追上leader的最新数据的呢？比如对于日志复制过程，leader完成自身的WAL及raft日志然后发送MsgApp，但是其它follower没有来得及发送MsgResp就崩溃了，那么这条raft日志其实没有得到法定票数的提交信息，raft模块应该通过什么方式来让follower恢复这份数据，让它能够最终在恢复的节点得到法定票数提交，然后应用到上层状态机？<br>3.  如果在写数据过程中leader崩溃了。比如在leader完成自身WAL并发送MsgApp后崩溃了，本次写提案没有完成，重新选leader之后新leader有这次写记录的log，它会怎么恢复数据？<br><br>谢谢老师</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 好问题，第一个问题不能说是数据丢失，正如你所说的，可能存在一个新日志条目，这些异常重启后的follower中没有，因为异常的follower数达到了法定数，那么这个新日志条目肯定没同步到多数节点上，因此是合理的。<br>第二个问题，看落后数据程度，通过内存中的raft log和快照文件追赶，因篇幅关系，未介绍日志压缩和快照，后面我根据情况是否单独加餐篇。如果提交过程中，不能达到法定数，这时会超时，依赖客户端重试<br>第三个问题，描述可以更详细点吗，没太懂，按我初步理解回答下，其他节点收到此日志条目后会持久化，选出leader也有此条记录，新老leader如果数据有冲突，以新leader为准<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-27 18:16:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/41/38/4f89095b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>写点啥呢</span>
  </div>
  <div class="_2_QraFYR_0">谢谢老师的回答，关于第三个问题，“3. 如果在写数据过程中leader崩溃了。比如在leader完成自身WAL并发送MsgApp后崩溃了，本次写提案没有完成，重新选leader之后新leader有这次写记录的log，它会怎么恢复数据？” 我详细描述下，请老师帮忙解答下：<br>1. Leader收到用户put请求后，完成自己的WAL日志及raft日志持久化后发送MsgApp到其它follower节点。<br>2. 其它follower节点接收到MsgApp后也完成自己的WAL及raft日志操作。<br>3. 此时Leader节点崩溃了，follower节点无法发送自己日志MsgAppResp给已经崩溃的Leader。此时集群开始新一轮选举并选出一个Leader<br>4. 新的Leader接收过之前的put提案并在日志中找到它，新的Leader如何恢复这条提案到提交状态？新的leader是不是在日志中看到这条没有完成提案后重新在做一轮提案？<br><br>谢谢老师<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 首先在raft中并没有什么数据结构来保存提案状态，leader只维护了一个committed index, 它表示这个index之前的日志条目已被成功同步到了大多数follower节点。当原leader crash后，其他follower节点选举出新leader, 按照raft安全性原则，它是不能删除前任leader的任何日志条目，因leader crash前这条日志条目已经被持久化到了多数follower节点上，那么follower节点选举出新leader后，它含有这条日志条目，并且多数节点已经同步了，那么对新leader而言，它的状态就是已提交，可以直接提交给状态机模块执行。对client而言，虽然写请求超时了，但最终它的提案是成功执行的，client需要自己确保幂等性，也就是写超时后，你的提交可能是成功的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-29 09:33:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/76/7d/04c95885.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Index</span>
  </div>
  <div class="_2_QraFYR_0">关于etcd的raft实现源码有个问题<br>func ExampleNode() {<br>	c := &amp;Config{}<br>	n := StartNode(c, nil)<br>	defer n.Stop()<br><br>	&#47;&#47; stuff to n happens in other goroutines<br><br>	&#47;&#47; the last known state<br>	var prev pb.HardState<br>	for {<br>		&#47;&#47; Ready blocks until there is new state ready.<br>		rd := &lt;-n.Ready()<br>		if !isHardStateEqual(prev, rd.HardState) {<br>			saveStateToDisk(rd.HardState)<br>			prev = rd.HardState<br>		}<br><br>		saveToDisk(rd.Entries)<br>		go applyToStore(rd.CommittedEntries)<br>		sendMessages(rd.Messages)<br>	}<br>}<br><br>网络，存储都是用户自己实现的，如果这里在处理存储和发送消息很慢，这样不会影响到心跳吗？比如心跳默认是1s，那如果处理存储和网络的时间经常超过1s，岂不是心跳就时常超时，集群经常处于选举的状态吗？求解答</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，etcd特别依赖磁盘I&#47;O性能，日志条目需要同步持久化到磁盘，当心跳间隔是100ms,选举超时是1s, 如果fsync 日志条目WAL耗时不稳定、波动很大、超过1秒，那集群就会频繁选举。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-29 11:17:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/d8/63/e4c28138.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>春风</span>
  </div>
  <div class="_2_QraFYR_0">假如老的 Leader A 因为网络问题无法连通 B、C 节点，这时候根据状态图，我们知道它将不停自增任期号，发起选举。等 A 节点网络异常恢复后，那么现有 Leader 收到了新的任期号，就会触发新一轮 Leader 选举，影响服务的可用性。<br><br>老师，这里没太懂，A不是本身是leader吗，为什么还要发起选举？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢反馈，是我描述不严谨，让你误会了，在上文中老的Leader A crash后，重启会变成Follower节点，已完善描述，谢谢你</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-27 09:04:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/22/3a/de/e5c30589.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>云原生工程师</span>
  </div>
  <div class="_2_QraFYR_0">日志复制流程图很赞，清晰易懂</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-27 00:56:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/IIkdC2gohpcibib0AJvSdnJQefAuQYGlLySQOticThpF7Ck9WuDUQLJlgZ7ic13LIFnGBXXbMsSP3nZsbibBN98ZjGA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>batman</span>
  </div>
  <div class="_2_QraFYR_0">raft log 和 WAL是同一东西吗</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-09 23:14:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLDUJyeq54fiaXAgF62tNeocO3lHsKT4mygEcNoZLnibg6ONKicMgCgUHSfgW8hrMUXlwpNSzR8MHZwg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>types</span>
  </div>
  <div class="_2_QraFYR_0">选举规则当节点收到选举投票的时候，需检查候选者的最后一条日志中的任期号，若小于自己则拒绝投票。如果任期号相同，日志却比自己短，也拒绝为其投票。<br><br>请问“日志比自己短” 是指什么状态下的日志 ，是commited还是？<br>如果不是commited， B（leader）AppMsg 给A和C后crash， A收到了， C没收到。 这个时候C会不会成为leader，导致数据丢失</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-01 20:20:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/96/47/93838ff7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>青鸟飞鱼</span>
  </div>
  <div class="_2_QraFYR_0">主从复制缺点是因为主节点崩溃后，没有选主机制（是否可以考虑redis的哨兵选主）呢？还是因为数据一致性呢（raft保持强一致性的话，也是通过某些机制保证强一致性（主节点读或者ReadIndex））</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，主从复制我个人认为最大的缺点还是数据一致性，Raft的保证一致性背后的核心数学原理其实是抽屉原理，你可以详细了解下，不管是集群选举还是数据复制，都要求一半以上节点确认</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-27 08:25:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/28/c6/6e/49aff2bd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>erge</span>
  </div>
  <div class="_2_QraFYR_0">老师好，peer节点是什么呢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-23 09:54:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/11/3e/925aa996.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>HelloBug</span>
  </div>
  <div class="_2_QraFYR_0">老师在讲解流程的时候，会说raft模块，kvserver模块，这些都好理解。但又会说etcd发送什么消息，或者etcdserver怎么怎么样，具体是什么模块呢？我之前以为这两者不指哪个模块，就是etcd的框架代码，但是这篇文章里又看到etcdserver模块，着实不知道怎么理解了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，有些术语的确可以更友好点，推荐先简单看看etcd源码（3.3&#47;3.4版本），跟着02和03大致走读下，这样你会更加清晰</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-02 07:37:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/e1/da/d7f591a7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>励研冰</span>
  </div>
  <div class="_2_QraFYR_0">等 A 节点网络异常恢复后，那么现有 Leader 收到了新的任期号，就会触发新一轮 Leader 选举，影响服务的可用性。<br>上文说到leader选举收到选举票之后会跟自己的现在的任期号对比，且要比自己的数据新，那发起选票之后B跟C收到投票请求任期号可能比自己的要大，但是数据应该没有比B跟C的大吧，所以这时候B告诉A现在自己是leader让A直接成为follower 就行了，为什么会重新选举呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-04 20:52:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_aaa517</span>
  </div>
  <div class="_2_QraFYR_0">raft日志跟wal日志没太搞明白，分别是什么作用呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: WAL日志是用来持久化raft日志条目和相关集群元数据信息的，可防止节点发生重启、crash后，对应的已提交日志条目丢失等异常情况，一般情况下，它是将数据持久化到磁盘中。<br>Raft日志记录了节点过去一段时间内收到的写请求，如put&#47;del&#47;txn操作等，一般是通过一个内存数组来存储最近一系列日志条目。当Follower节点落后Leader较小时，就可以通过Leader内存中维护的日志条目信息, 将落后的日志条目发送给它，最终各个节点应用一样的日志条目内容，来确保各个节点数据一致性。<br>etcd节点重启后，可通过WAL日志来重建部分raft日志条目。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-20 19:34:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/98/b1/f89a84d0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Tokamak</span>
  </div>
  <div class="_2_QraFYR_0">老师，您好请教一写问题：<br>1. 如果一个集群有3个节点, 则Leader会维护3组 (NextIndex, MatchIndex)吗?<br>2. 由于每个节点都可以变为Leader, 是否也意味着Follower也会同步更新这两个变量的值呢?<br>3. 假设当前任期号是1，此时Leader crash了，进行选举理论上这次任期号是2，但此次选举失败了，再下一次选举成功时会使用任期号3还是任期号2？<br><br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-22 10:42:02</div>
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
  <div class="_2_QraFYR_0">厉害了，不过之前没接触过相关知识，看的有点吃力，找了一篇通俗易懂的补了下基础，为后面的课程做准备，分享一下：https:&#47;&#47;blog.csdn.net&#47;wangxuelei036&#47;article&#47;details&#47;108894053</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-16 00:05:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/cb/38/4c9cfdf4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>谢小路</span>
  </div>
  <div class="_2_QraFYR_0">行吧。raft 动画版。http:&#47;&#47;thesecretlivesofdata.com&#47;raft&#47;</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-04 23:47:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/KjXNw3FNfsJhktyGDNEpQdicnaR0MZCiaQwGg3icQXUGVzqsRHL1FfxO87FX0VNzcl858AUjWPaABmogsWNFa8OGw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>阿基米德</span>
  </div>
  <div class="_2_QraFYR_0">如果etcd集群有5个节点，分别是a,b,c,d,e节点，a是leader，a将一条新数据x=5刷盘到wal之后，只给b节点发去了追加日志的消息，然后a宕机了。重新选举，b有可能成为新一任的leader吗？b成为leader之后，它日志文件中的x=5这条数据会被删除还是会同步到其他follower节点，因为这条x=5的数据在上一个leader的任期内是还没有写入到大多数节点的？如果不删除x=5，算不算是脏数据，因为写入这条数据的客户端肯定超时报错了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-05 17:24:37</div>
  </div>
</div>
</div>
</li>
</ul>