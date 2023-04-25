<audio title="45｜Master高可用：怎样借助etcd实现服务选主？" src="https://static001.geekbang.org/resource/audio/b2/2d/b2dcac12c91ede3df4f5520d545b8a2d.mp3" controls="controls"></audio> 
<p>你好，我是郑建勋。</p><p>上一节课，我们搭建起了Master的基本框架。这一节课，让我们接着实现分布式Master的核心功能：选主。</p><h2>etcd选主API</h2><p>我们在讲解架构设计时提到过，可以开启多个Master来实现分布式服务的故障容错。其中，只有一个Master能够成为Leader，只有Leader能够完成任务的分配，只有Leader能够处理外部访问。当Leader崩溃时，其他的Master将竞争上岗成为Leader。</p><p>实现分布式的选主并没有想象中那样复杂，在我们的项目中，只需要借助分布式协调服务etcd就能实现。<a href="https://pkg.go.dev/go.etcd.io/etcd/clientv3/concurrency">etcd clientv3</a> 已经为我们封装了对分布式选主的实现，核心的API如下。</p><pre><code class="language-plain">// client/v3/concurrency
func NewSession(client *v3.Client, opts ...SessionOption) (*Session, error)
func NewElection(s *Session, pfx string) *Election
func (e *Election) Campaign(ctx context.Context, val string) error
func (e *Election) Leader(ctx context.Context) (*v3.GetResponse, error)
func (e *Election) Observe(ctx context.Context) &lt;-chan v3.GetResponse
func (e *Election) Resign(ctx context.Context) (err error)
</code></pre><!-- [[[read_end]]] --><p>我来解释一下这些API的含义。</p><ul>
<li>NewSession函数：创建一个与etcd服务端带租约的会话。</li>
<li>NewElection函数：创建一个选举对象Election，Election有许多方法。</li>
<li>Election.Leader方法可以查询当前集群中的Leader信息。</li>
<li>Election.Observe 可以接收到当前Leader的变化</li>
<li>Election.Campaign方法：开启选举，该方法会阻塞住协程，直到调用者成为Leader。</li>
</ul><h2>实现Master选主与故障容错</h2><p>现在让我们在项目中实现分布式选主算法，核心逻辑位于Master.Campaign方法中，完整的代码位于<a href="https://github.com/dreamerjackson/crawler">v0.3.5分支</a>。</p><pre><code class="language-plain">func (m *Master) Campaign() {
	endpoints := []string{m.registryURL}
	cli, err := clientv3.New(clientv3.Config{Endpoints: endpoints})
	if err != nil {
		panic(err)
	}

	s, err := concurrency.NewSession(cli, concurrency.WithTTL(5))
	if err != nil {
		fmt.Println("NewSession", "error", "err", err)
	}
	defer s.Close()

	// 创建一个新的etcd选举election
	e := concurrency.NewElection(s, "/resources/election")
	leaderCh := make(chan error)
	go m.elect(e, leaderCh)
	leaderChange := e.Observe(context.Background())
	select {
	case resp := &lt;-leaderChange:
		m.logger.Info("watch leader change", zap.String("leader:", string(resp.Kvs[0].Value)))
	}

	for {
		select {
		case err := &lt;-leaderCh:
			if err != nil {
				m.logger.Error("leader elect failed", zap.Error(err))
				go m.elect(e, leaderCh)
			} else {
				m.logger.Info("master change to leader")
				m.leaderID = m.ID
				if !m.IsLeader() {
					m.BecomeLeader()
				}
			}
		case resp := &lt;-leaderChange:
			if len(resp.Kvs) &gt; 0 {
				m.logger.Info("watch leader change", zap.String("leader:", string(resp.Kvs[0].Value)))
			}
		}
	}
}
</code></pre><p>我们一步步来解析这段分布式选主的代码。</p><ol>
<li>第3行调用clientv3.New函数创建一个etcd clientv3的客户端。</li>
<li>第15行，<code>concurrency.NewElection(s, "/resources/election")</code> 意为创建一个新的etcd选举对象。其中的第二个参数就是所有Master都在抢占的Key，抢占到该Key的Master将变为Leader。</li>
</ol><p>在etcd中，一般都会选择这种目录形式的结构作为Key，这种方式可以方便我们进行前缀查找。例如，Kubernetes 资源在 etcd 中的存储格式为  <code>prefix/资源类型/namespace/资源名称</code> 。</p><pre><code class="language-plain">/registry/clusterrolebindings/system:coredns
/registry/clusterroles/system:coredns
/registry/configmaps/kube-system/coredns
/registry/deployments/kube-system/coredns
/registry/replicasets/kube-system/coredns-7fdd6d65dc
/registry/secrets/kube-system/coredns-token-hpqbt
/registry/serviceaccounts/kube-system/coredns
</code></pre><ol start="3">
<li>第17行 <code>go m.elect(e, leaderCh)</code> 代表开启一个新的协程，让当前的Master进行Leader的选举。如果集群中已经有了其他的Leader，当前协程将陷入到堵塞状态。如果当前Master选举成功，成为了Leader，e.Campaign方法会被唤醒，我们将其返回的消息传递到ch通道中。</li>
</ol><pre><code class="language-plain">func (m *Master) elect(e *concurrency.Election, ch chan error) {
	// 堵塞直到选取成功
	err := e.Campaign(context.Background(), m.ID)
	ch &lt;- err
}
</code></pre><p>e.Campaign方法的第二个参数为Master成为Leader后，设置到Key中的Value值。在这里，我们将Master ID作为Value值。 Master的ID是初始化时设置的，它当前包含了Master的序号、Master的IP地址和监听的GRPC地址。</p><p>其实，Master的ID就足够标识唯一的Master了，但这里还存储了 Master IP，是为了方便后续其他的Master拿到Leader的IP地址，从而对Leader进行访问。</p><pre><code class="language-plain">// master/master.go
func New(id string, opts ...Option) (*Master, error) {
	m := &amp;Master{}

	options := defaultOptions
	for _, opt := range opts {
		opt(&amp;options)
	}
	m.options = options

	ipv4, err := getLocalIP()
	if err != nil {
		return nil, err
	}
	m.ID = genMasterID(id, ipv4, m.GRPCAddress)
	m.logger.Sugar().Debugln("master_id:", m.ID)
	go m.Campaign()

	return &amp;Master{}, nil
}

type Master struct {
	ID        string
	ready     int32
	leaderID  string
	workNodes map[string]*registry.Node
	options
}

func genMasterID(id string, ipv4 string, GRPCAddress string) string {
	return "master" + id + "-" + ipv4 + GRPCAddress
}
</code></pre><p>获取本机的IP地址有一个很简单的方式，那就是遍历所有网卡，找到第一个IPv4地址，代码如下所示。</p><pre><code class="language-plain">func getLocalIP() (string, error) {
	var (
		addrs []net.Addr
		err   error
	)
	// 获取所有网卡
	if addrs, err = net.InterfaceAddrs(); err != nil {
		return "", err
	}
	// 取第一个非lo的网卡IP
	for _, addr := range addrs {
		if ipNet, isIpNet := addr.(*net.IPNet); isIpNet &amp;&amp; !ipNet.IP.IsLoopback() {
			if ipNet.IP.To4() != nil {
				return ipNet.IP.String(), nil
			}
		}
	}

	return "", errors.New("no local ip")
}
</code></pre><ol start="4">
<li>
<p>当Master并行进行选举的同时（第18行），调用e.Observe监听Leader的变化。e.Observe函数会返回一个通道，当Leader状态发生变化时，会将当前Leader的信息发送到通道中。在这里我们初始化时首先堵塞读取了一次e.Observe返回的通道信息。因为只有成功收到e.Observe返回的消息，才意味着集群中已经存在Leader，表示集群完成了选举。</p>
</li>
<li>
<p>第24行，我们在for循环中使用select监听了多个通道的变化，其中通道leaderCh负责监听当前Master是否当上了Leader，而leaderChange负责监听当前集群中Leader是否发生了变化。</p>
</li>
</ol><p>书写好Master的选主逻辑之后，接下来让我们执行Master程序，完整的代码位于<a href="https://github.com/dreamerjackson/crawler">v0.3.5分支</a>。</p><pre><code class="language-plain">» go run main.go master --id=2 --http=:8081  --grpc=:9091
</code></pre><p>由于当前只有一个Master，因此当前Master一定会成为Leader。我们可以看到打印出的当前Leader的信息：master2-192.168.0.107:9091。</p><pre><code class="language-plain">{"level":"INFO","ts":"2022-12-07T18:23:28.494+0800","logger":"master","caller":"master/master.go:65","msg":"watch leader change","leader:":"master2-192.168.0.107:9091"}
{"level":"INFO","ts":"2022-12-07T18:23:28.494+0800","logger":"master","caller":"master/master.go:65","msg":"watch leader change","leader:":"master2-192.168.0.107:9091"}
{"level":"INFO","ts":"2022-12-07T18:23:28.494+0800","logger":"master","caller":"master/master.go:75","msg":"master change to leader"}
{"level":"DEBUG","ts":"2022-12-07T18:23:38.500+0800","logger":"master","caller":"master/master.go:87","msg":"get Leader","Value":"master2-192.168.0.107:9091"}
</code></pre><p>如果这时我们查看etcd的信息，会看到自动生成了 <code>/resources/election/xxx</code> 的Key，并且它的Value是我们设置的 <code>master2-192.168.0.107:9091</code>。</p><pre><code class="language-plain">» docker exec etcd-gcr-v3.5.6 /bin/sh -c "/usr/local/bin/etcdctl get --prefix /"                                                                           jackson@bogon

/micro/registry/go.micro.server.master/go.micro.server.master-2
{"name":"go.micro.server.master","version":"latest","metadata":null,"endpoints":[{"name":"Greeter.Hello","request":{"name":"Request","type":"Request","values":[{"name":"name","type":"string","values":null}]},"response":{"name":"Response","type":"Response","values":[{"name":"greeting","type":"string","values":null}]},"metadata":{"endpoint":"Greeter.Hello","handler":"rpc","method":"POST","path":"/greeter/hello"}}],"nodes":[{"id":"go.micro.server.master-2","address":"192.168.0.107:9091","metadata":{"broker":"http","protocol":"grpc","registry":"etcd","server":"grpc","transport":"grpc"}}]}

/resources/election/3f3584fc571ae898
master2-192.168.0.107:9091
</code></pre><p>如果我们再启动一个新的Master程序，会发现当前获取到的Leader仍然是 <code>master2-192.168.0.107:9091</code>。</p><pre><code class="language-plain">» go run main.go master --id=3 --http=:8082  --grpc=:9092
{"level":"DEBUG","ts":"2022-12-07T18:23:52.371+0800","logger":"master","caller":"master/master.go:33","msg":"master_id: master3-192.168.0.107:9092"}
{"level":"INFO","ts":"2022-12-07T18:23:52.387+0800","logger":"master","caller":"master/master.go:65","msg":"watch leader change","leader:":"master2-192.168.0.107:9091"}
{"level":"DEBUG","ts":"2022-12-07T18:24:02.393+0800","logger":"master","caller":"master/master.go:87","msg":"get Leader","value":"master2-192.168.0.107:9091"}
</code></pre><p>再次查看etcd的信息，会发现 <code>go.micro.server.master-3</code> 也成功注册到etcd中了，并且c在/resources/election 下方注册了自己的Key，但是该Key比master-2要大。</p><pre><code class="language-plain">/micro/registry/go.micro.server.master/go.micro.server.master-2
{"name":"go.micro.server.master","version":"latest","metadata":null,"endpoints":[{"name":"Greeter.Hello","request":{"name":"Request","type":"Request","values":[{"name":"name","type":"string","values":null}]},"response":{"name":"Response","type":"Response","values":[{"name":"greeting","type":"string","values":null}]},"metadata":{"endpoint":"Greeter.Hello","handler":"rpc","method":"POST","path":"/greeter/hello"}}],"nodes":[{"id":"go.micro.server.master-2","address":"192.168.0.107:9091","metadata":{"broker":"http","protocol":"grpc","registry":"etcd","server":"grpc","transport":"grpc"}}]}
/micro/registry/go.micro.server.master/go.micro.server.master-3
{"name":"go.micro.server.master","version":"latest","metadata":null,"endpoints":[{"name":"Greeter.Hello","request":{"name":"Request","type":"Request","values":[{"name":"name","type":"string","values":null}]},"response":{"name":"Response","type":"Response","values":[{"name":"greeting","type":"string","values":null}]},"metadata":{"endpoint":"Greeter.Hello","handler":"rpc","method":"POST","path":"/greeter/hello"}}],"nodes":[{"id":"go.micro.server.master-3","address":"192.168.0.107:9092","metadata":{"broker":"http","protocol":"grpc","registry":"etcd","server":"grpc","transport":"grpc"}}]}
/resources/election/3f3584fc571ae898
master2-192.168.0.107:9091
/resources/election/3f3584fc571ae8a9
master3-192.168.0.107:9092
</code></pre><p>到这里，我们就实现了Master的选主操作，所有的Master都只认定一个Leader。当我们终止master-2程序，在master-3程序中会立即看到如下日志，说明当前的Leader已经顺利完成了切换，master-3当选为了新的Leader。</p><pre><code class="language-plain">{"level":"INFO","ts":"2022-12-12T00:46:58.288+0800","logger":"master","caller":"master/master.go:93","msg":"watch leader change","leader:":"master3-192.168.0.107:9092"}
{"level":"INFO","ts":"2022-12-12T00:46:58.289+0800","logger":"master","caller":"master/master.go:85","msg":"master change to leader"}
{"level":"DEBUG","ts":"2022-12-12T00:47:18.296+0800","logger":"master","caller":"master/master.go:107","msg":"get Leader","value":"master3-192.168.0.107:9092"}
</code></pre><p>当我们再次查看etcd，发现/resources/election/路径下只剩下master-3程序的注册信息了，证明Master的选举成功。</p><pre><code class="language-plain">» docker exec etcd-gcr-v3.5.6 /bin/sh -c "/usr/local/bin/etcdctl get --prefix /"                                                                           jackson@bogon
/micro/registry/go.micro.server.master/go.micro.server.master-3
{"name":"go.micro.server.master","version":"latest","metadata":null,"endpoints":[{"name":"Greeter.Hello","request":{"name":"Request","type":"Request","values":[{"name":"name","type":"string","values":null}]},"response":{"name":"Response","type":"Response","values":[{"name":"greeting","type":"string","values":null}]},"metadata":{"endpoint":"Greeter.Hello","handler":"rpc","method":"POST","path":"/greeter/hello"}}],"nodes":[{"id":"go.micro.server.master-3","address":"192.168.0.107:9092","metadata":{"broker":"http","protocol":"grpc","registry":"etcd","server":"grpc","transport":"grpc"}}]}
/resources/election/3f3584fc571ae8a9
master3-192.168.0.107:9092
</code></pre><h2>etcd选主原理</h2><p>经过上面的实践，我们可以看到，借助etcd，分布式选主变得非常容易了，现在我们来看一看etcd实现分布式选主的原理。它的核心代码位于Election.Campaign方法中，如下所示，下面代码做了简化，省略了对异常情况的处理。</p><pre><code class="language-plain">func (e *Election) Campaign(ctx context.Context, val string) error {
	s := e.session
	client := e.session.Client()

	k := fmt.Sprintf("%s%x", e.keyPrefix, s.Lease())
	txn := client.Txn(ctx).If(v3.Compare(v3.CreateRevision(k), "=", 0))
	txn = txn.Then(v3.OpPut(k, val, v3.WithLease(s.Lease())))
	txn = txn.Else(v3.OpGet(k))
	resp, err := txn.Commit()
	if err != nil {
		return err
	}
	e.leaderKey, e.leaderRev, e.leaderSession = k, resp.Header.Revision, s
	_, err = waitDeletes(ctx, client, e.keyPrefix, e.leaderRev-1)
	if err != nil {
		// clean up in case of context cancel
		select {
		case &lt;-ctx.Done():
			e.Resign(client.Ctx())
		default:
			e.leaderSession = nil
		}
		return err
	}
	e.hdr = resp.Header

	return nil
}
</code></pre><p>Campaign首先用了一个事务操作在要抢占的e.keyPrefix路径下维护一个Key。其中，e.keyPrefix是Master要抢占的etcd路径，在我们的项目中为/resources/election/。这段事务操作会首选判断当前生成的Key（例如/resources/election/3f3584fc571ae8a9）是否已经在etcd中了。如果不存在，才会创建该Key。这样，每一个Master都会在/resources/election/下维护一个Key，并且当前的Key是带租约的。</p><p>Campaign第二步会调用waitDeletes函数堵塞等待，直到自己成为Leader为止。那什么时候当前Master会成为Leader呢？</p><p>waitDeletes函数会调用client.Get获取到当前争抢的/resources/election/路径下具有最大版本号的Key，并调用waitDelete函数等待该Key被删除。而waitDelete会调用client.Watch来完成对特定版本Key的监听。</p><p>当前Master需要监听这个最大版本号Key的删除事件。当这个特定的Key被删除，就意味着已经没有比当前Master创建的Key更早的Key了，因此当前的Master理所当然就排队成为了Leader。</p><pre><code class="language-plain">func waitDeletes(ctx context.Context, client *v3.Client, pfx string, maxCreateRev int64) (*pb.ResponseHeader, error) {
	getOpts := append(v3.WithLastCreate(), v3.WithMaxCreateRev(maxCreateRev))
	for {
		resp, err := client.Get(ctx, pfx, getOpts...)
		if err != nil {
			return nil, err
		}
		if len(resp.Kvs) == 0 {
			return resp.Header, nil
		}
		lastKey := string(resp.Kvs[0].Key)
		if err = waitDelete(ctx, client, lastKey, resp.Header.Revision); err != nil {
			return nil, err
		}
	}
}

func waitDelete(ctx context.Context, client *v3.Client, key string, rev int64) error {
	cctx, cancel := context.WithCancel(ctx)
	defer cancel()

	var wr v3.WatchResponse
	wch := client.Watch(cctx, key, v3.WithRev(rev))
	for wr = range wch {
		for _, ev := range wr.Events {
			if ev.Type == mvccpb.DELETE {
				return nil
			}
		}
	}
	if err := wr.Err(); err != nil {
		return err
	}
	if err := ctx.Err(); err != nil {
		return err
	}
	return fmt.Errorf("lost watcher waiting for delete")
}
</code></pre><p>这种监听方式还避免了惊群效应，因为当Leader崩溃后，并不会唤醒所有在选举中的Master。只有当队列中的前一个Master创建的Key被删除，当前的Master才会被唤醒。也就是说，每一个Master都在排队等待着前一个Master退出，这样Master就以最小的代价实现了对Key的争抢。</p><h2>总结</h2><p>这节课，我们借助etcd实现了分布式Master的选主，确保了在同一时刻只能存在一个Leader。此外，我们还实现了Master的故障容错能力。</p><p>etcd clientv3为我们封装了选主的实现，它的实现方式也很简单，通过监听最近的Key的DELETE事件，我们实现了所有的节点对同一个Key的抢占，同时还避免了集群可能出现的惊群效应。在实践中，我们也可以使用其他的分布式协调组件（例如ZooKeeper、Consul）帮助我们实现选主，它们的实现原理都和etcd类似。</p><h2>课后题</h2><p>最后，给你留一道思考题吧。</p><p>其实，利用MySQL也可以实现分布式的选主，你知道如何实现吗？（参考：<a href="http://code.openark.org/blog/mysql/leader-election-using-mysql">http://code.openark.org/blog/mysql/leader-election-using-mysql</a>）</p><p>欢迎你给我留言交流讨论，我们下节课见！</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/4a/6e/5f2a8a99.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>無所畏</span>
  </div>
  <div class="_2_QraFYR_0">这段话是不是有问题？<br>&quot;waitDeletes 函数会调用 client.Get 获取到当前争抢的 &#47;resources&#47;election&#47; 路径下具有最大版本号的 Key&quot; <br>看waitDeletes 源码的注释是：<br>&quot;waitDeletes efficiently waits until all keys matching the prefix and no greater than the create revision.&quot;<br><br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: waitDeletes分为了2步，第一步是获取之前具有最大版本号的 Key，第二步是监听这个最大key的删除事件。 你截取我的文字部分，漏掉了监听这一步。 我想表达的是和注释一样的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-30 17:15:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/7f/d3/b5896293.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Realm</span>
  </div>
  <div class="_2_QraFYR_0">请问老师：<br><br>“当前 Master 需要监听这个最大版本号 Key 的删除事件。当这个特定的 Key 被删除，就意味着已经没有比当前 Master 创建的 Key 更早的 Key 了，因此当前的 Master 理所当然就排队成为了 Leader。”<br><br>1 是所有master监听的内容都相同吗？<br>2 这里如何避免惊群？<br><br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 其实每一个Master都是监听的前一个Mater创建的key，所以master监听的内容是不同的，也就没有了惊群了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-26 08:34:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_7e6c5e</span>
  </div>
  <div class="_2_QraFYR_0">太酷了，etcd让普通程序员也有了开发分布式系统的能力</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-23 23:34:07</div>
  </div>
</div>
</div>
</li>
</ul>