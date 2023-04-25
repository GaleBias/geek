<audio title="32 _ Redis主从同步与故障切换，有哪些坑？" src="https://static001.geekbang.org/resource/audio/37/de/375630900d9ef3ce58c9b7072e2256de.mp3" controls="controls"></audio> 
<p>你好，我是蒋德钧。</p><p>Redis的主从同步机制不仅可以让从库服务更多的读请求，分担主库的压力，而且还能在主库发生故障时，进行主从库切换，提供高可靠服务。</p><p>不过，在实际使用主从机制的时候，我们很容易踩到一些坑。这节课，我就向你介绍3个坑，分别是主从数据不一致、读到过期数据，以及配置项设置得不合理从而导致服务挂掉。</p><p>一旦踩到这些坑，业务应用不仅会读到错误数据，而且很可能会导致Redis无法正常使用，我们必须要全面地掌握这些坑的成因，提前准备一套规避方案。不过，即使不小心掉进了陷阱里，也不要担心，我还会给你介绍相应的解决方案。</p><p>好了，话不多说，下面我们先来看看第一个坑：主从数据不一致。</p><h2>主从数据不一致</h2><p>主从数据不一致，就是指客户端从从库中读取到的值和主库中的最新值并不一致。</p><p>举个例子，假设主从库之前保存的用户年龄值是19，但是主库接收到了修改命令，已经把这个数据更新为20了，但是，从库中的值仍然是19。那么，如果客户端从从库中读取用户年龄值，就会读到旧值。</p><p>那为啥会出现这个坑呢？其实这是因为<strong>主从库间的命令复制是异步进行的</strong>。</p><p>具体来说，在主从库命令传播阶段，主库收到新的写命令后，会发送给从库。但是，主库并不会等到从库实际执行完命令后，再把结果返回给客户端，而是主库自己在本地执行完命令后，就会向客户端返回结果了。如果从库还没有执行主库同步过来的命令，主从库间的数据就不一致了。</p><!-- [[[read_end]]] --><p>那在什么情况下，从库会滞后执行同步命令呢？其实，这里主要有两个原因。</p><p>一方面，主从库间的网络可能会有传输延迟，所以从库不能及时地收到主库发送的命令，从库上执行同步命令的时间就会被延后。</p><p>另一方面，即使从库及时收到了主库的命令，但是，也可能会因为正在处理其它复杂度高的命令（例如集合操作命令）而阻塞。此时，从库需要处理完当前的命令，才能执行主库发送的命令操作，这就会造成主从数据不一致。而在主库命令被滞后处理的这段时间内，主库本身可能又执行了新的写操作。这样一来，主从库间的数据不一致程度就会进一步加剧。</p><p>那么，我们该怎么应对呢？我给你提供两种方法。</p><p>首先，<strong>在硬件环境配置方面，我们要尽量保证主从库间的网络连接状况良好</strong>。例如，我们要避免把主从库部署在不同的机房，或者是避免把网络通信密集的应用（例如数据分析应用）和Redis主从库部署在一起。</p><p>另外，<strong>我们还可以开发一个外部程序来监控主从库间的复制进度</strong>。</p><p>因为Redis的INFO replication命令可以查看主库接收写命令的进度信息（master_repl_offset）和从库复制写命令的进度信息（slave_repl_offset），所以，我们就可以开发一个监控程序，先用INFO replication命令查到主、从库的进度，然后，我们用master_repl_offset减去slave_repl_offset，这样就能得到从库和主库间的复制进度差值了。</p><p>如果某个从库的进度差值大于我们预设的阈值，我们可以让客户端不再和这个从库连接进行数据读取，这样就可以减少读到不一致数据的情况。不过，为了避免出现客户端和所有从库都不能连接的情况，我们需要把复制进度差值的阈值设置得大一些。</p><p>我们在应用Redis时，可以周期性地运行这个流程来监测主从库间的不一致情况。为了帮助你更好地理解这个方法，我画了一张流程图，你可以看下。</p><p><img src="https://static001.geekbang.org/resource/image/3a/05/3a89935297fb5b76bfc4808128aaf905.jpg?wh=3000*2048" alt=""></p><p>当然，监控程序可以一直监控着从库的复制进度，当从库的复制进度又赶上主库时，我们就允许客户端再次跟这些从库连接。</p><p>除了主从数据不一致以外，我们有时还会在从库中读到过期的数据，这是怎么回事呢？接下来，我们就来详细分析一下。</p><h2>读取过期数据</h2><p>我们在使用Redis主从集群时，有时会读到过期数据。例如，数据X的过期时间是202010240900，但是客户端在202010240910时，仍然可以从从库中读到数据X。一个数据过期后，应该是被删除的，客户端不能再读取到该数据，但是，Redis为什么还能在从库中读到过期的数据呢？</p><p>其实，这是由Redis的过期数据删除策略引起的。我来给你具体解释下。</p><p><strong>Redis同时使用了两种策略来删除过期的数据，分别是惰性删除策略和定期删除策略</strong>。</p><p>先说惰性删除策略。当一个数据的过期时间到了以后，并不会立即删除数据，而是等到再有请求来读写这个数据时，对数据进行检查，如果发现数据已经过期了，再删除这个数据。</p><p>这个策略的好处是尽量减少删除操作对CPU资源的使用，对于用不到的数据，就不再浪费时间进行检查和删除了。但是，这个策略会导致大量已经过期的数据留存在内存中，占用较多的内存资源。所以，Redis在使用这个策略的同时，还使用了第二种策略：定期删除策略。</p><p>定期删除策略是指，Redis每隔一段时间（默认100ms），就会随机选出一定数量的数据，检查它们是否过期，并把其中过期的数据删除，这样就可以及时释放一些内存。</p><p>清楚了这两个删除策略，我们再来看看它们为什么会导致读取到过期数据。</p><p>首先，虽然定期删除策略可以释放一些内存，但是，Redis为了避免过多删除操作对性能产生影响，每次随机检查数据的数量并不多。如果过期数据很多，并且一直没有再被访问的话，这些数据就会留存在Redis实例中。业务应用之所以会读到过期数据，这些留存数据就是一个重要因素。</p><p>其次，惰性删除策略实现后，数据只有被再次访问时，才会被实际删除。如果客户端从主库上读取留存的过期数据，主库会触发删除操作，此时，客户端并不会读到过期数据。但是，从库本身不会执行删除操作，如果客户端在从库中访问留存的过期数据，从库并不会触发数据删除。那么，从库会给客户端返回过期数据吗？</p><p>这就和你使用的Redis版本有关了。如果你使用的是Redis 3.2之前的版本，那么，从库在服务读请求时，并不会判断数据是否过期，而是会返回过期数据。在3.2版本后，Redis做了改进，如果读取的数据已经过期了，从库虽然不会删除，但是会返回空值，这就避免了客户端读到过期数据。所以，<strong>在应用主从集群时，尽量使用Redis 3.2及以上版本</strong>。</p><p>你可能会问，只要使用了Redis 3.2后的版本，就不会读到过期数据了吗？其实还是会的。</p><p>为啥会这样呢？这跟Redis用于设置过期时间的命令有关系，有些命令给数据设置的过期时间在从库上可能会被延后，导致应该过期的数据又在从库上被读取到了，我来给你具体解释下。</p><p>我先给你介绍下这些命令。设置数据过期时间的命令一共有4个，我们可以把它们分成两类：</p><ul>
<li>EXPIRE和PEXPIRE：它们给数据设置的是<strong>从命令执行时开始计算的存活时间</strong>；</li>
<li>EXPIREAT和PEXPIREAT：<strong>它们会直接把数据的过期时间设置为具体的一个时间点</strong>。</li>
</ul><p>这4个命令的参数和含义如下表所示：</p><p><img src="https://static001.geekbang.org/resource/image/06/e1/06e8cb2f1af320d450a29326a876f4e1.jpg?wh=2967*920" alt=""></p><p>为了方便你理解，我给你举两个例子。</p><p>第一个例子是使用EXPIRE命令，当执行下面的命令时，我们就把testkey的过期时间设置为60s后。</p><pre><code>EXPIRE testkey 60
</code></pre><p>第二个例子是使用EXPIREAT命令，例如，我们执行下面的命令，就可以让testkey在2020年10月24日上午9点过期，命令中的1603501200就是以秒数时间戳表示的10月24日上午9点。</p><pre><code>EXPIREAT testkey 1603501200
</code></pre><p>好了，知道了这些命令，下面我们来看看这些命令如何导致读到过期数据。</p><p>当主从库全量同步时，如果主库接收到了一条EXPIRE命令，那么，主库会直接执行这条命令。这条命令会在全量同步完成后，发给从库执行。而从库在执行时，就会在当前时间的基础上加上数据的存活时间，这样一来，从库上数据的过期时间就会比主库上延后了。</p><p>这么说可能不太好理解，我再给你举个例子。</p><p>假设当前时间是2020年10月24日上午9点，主从库正在同步，主库收到了一条命令：EXPIRE testkey 60，这就表示，testkey的过期时间就是24日上午9点1分，主库直接执行了这条命令。</p><p>但是，主从库全量同步花费了2分钟才完成。等从库开始执行这条命令时，时间已经是9点2分了。而EXPIRE命令是把testkey的过期时间设置为当前时间的60s后，也就是9点3分。如果客户端在9点2分30秒时在从库上读取testkey，仍然可以读到testkey的值。但是，testkey实际上已经过期了。</p><p>为了避免这种情况，我给你的建议是，<strong>在业务应用中使用EXPIREAT/PEXPIREAT命令，把数据的过期时间设置为具体的时间点，避免读到过期数据。</strong></p><p>好了，我们先简单地总结下刚刚学过的这两个典型的坑。</p><ul>
<li>主从数据不一致。Redis采用的是异步复制，所以无法实现强一致性保证（主从数据时时刻刻保持一致），数据不一致是难以避免的。我给你提供了应对方法：保证良好网络环境，以及使用程序监控从库复制进度，一旦从库复制进度超过阈值，不让客户端连接从库。</li>
<li>对于读到过期数据，这是可以提前规避的，一个方法是，使用Redis 3.2及以上版本；另外，你也可以使用EXPIREAT/PEXPIREAT命令设置过期时间，避免从库上的数据过期时间滞后。不过，这里有个地方需要注意下，<strong>因为EXPIREAT/PEXPIREAT设置的是时间点，所以，主从节点上的时钟要保持一致，具体的做法是，让主从节点和相同的NTP服务器（时间服务器）进行时钟同步</strong>。</li>
</ul><p>除了同步过程中有坑以外，主从故障切换时，也会因为配置不合理而踩坑。接下来，我向你介绍两个服务挂掉的情况，都是由不合理配置项引起的。</p><h2>不合理配置项导致的服务挂掉</h2><p>这里涉及到的配置项有两个，分别是<strong>protected-mode和cluster-node-timeout。</strong></p><p><strong>1.protected-mode 配置项</strong></p><p>这个配置项的作用是限定哨兵实例能否被其他服务器访问。当这个配置项设置为yes时，哨兵实例只能在部署的服务器本地进行访问。当设置为no时，其他服务器也可以访问这个哨兵实例。</p><p>正因为这样，如果protected-mode被设置为yes，而其余哨兵实例部署在其它服务器，那么，这些哨兵实例间就无法通信。当主库故障时，哨兵无法判断主库下线，也无法进行主从切换，最终Redis服务不可用。</p><p>所以，我们在应用主从集群时，要注意将protected-mode 配置项设置为no，并且将bind配置项设置为其它哨兵实例的IP地址。这样一来，只有在bind中设置了IP地址的哨兵，才可以访问当前实例，既保证了实例间能够通信进行主从切换，也保证了哨兵的安全性。</p><p>我们来看一个简单的小例子。如果设置了下面的配置项，那么，部署在192.168.10.3/4/5这三台服务器上的哨兵实例就可以相互通信，执行主从切换。</p><pre><code>protected-mode no
bind 192.168.10.3 192.168.10.4 192.168.10.5
</code></pre><p><strong>2.cluster-node-timeout配置项</strong></p><p><strong>这个配置项设置了Redis Cluster中实例响应心跳消息的超时时间</strong>。</p><p>当我们在Redis Cluster集群中为每个实例配置了“一主一从”模式时，如果主实例发生故障，从实例会切换为主实例，受网络延迟和切换操作执行的影响，切换时间可能较长，就会导致实例的心跳超时（超出cluster-node-timeout）。实例超时后，就会被Redis Cluster判断为异常。而Redis Cluster正常运行的条件就是，有半数以上的实例都能正常运行。</p><p>所以，如果执行主从切换的实例超过半数，而主从切换时间又过长的话，就可能有半数以上的实例心跳超时，从而可能导致整个集群挂掉。所以，<strong>我建议你将cluster-node-timeout调大些（例如10到20秒）</strong>。</p><h2>小结</h2><p>这节课，我们学习了Redis主从库同步时可能出现的3个坑，分别是主从数据不一致、读取到过期数据和不合理配置项导致服务挂掉。</p><p>为了方便你掌握，我把这些坑的成因和解决方法汇总在下面的这张表中，你可以再回顾下。</p><p><img src="https://static001.geekbang.org/resource/image/9f/93/9fb7a033987c7b5edc661f4de58ef093.jpg?wh=2788*1155" alt=""></p><p>最后，关于主从库数据不一致的问题，我还想再给你提一个小建议：Redis中的slave-serve-stale-data配置项设置了从库能否处理数据读写命令，你可以把它设置为no。这样一来，从库只能服务INFO、SLAVEOF命令，这就可以避免在从库中读到不一致的数据了。</p><p>不过，你要注意下这个配置项和slave-read-only的区别，slave-read-only是设置从库能否处理写命令，slave-read-only设置为yes时，从库只能处理读请求，无法处理写请求，你可不要搞混了。</p><h2>每课一问</h2><p>按照惯例，我给你提个小问题，我们把slave-read-only设置为no，让从库也能直接删除数据，以此来避免读到过期数据，你觉得，这是一个好方法吗？</p><p>欢迎在留言区写下你的思考和答案，我们一起交流讨论。如果你觉得今天的内容对你有所帮助，也欢迎你分享给你的朋友或同事。我们下节课见。</p>
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
  <div class="_2_QraFYR_0">把 slave-read-only 设置为 no，让从库也能直接删除数据，以此来避免读到过期数据，这种方案是否可行？<br><br>我个人觉得这个问题有些歧义，因为尽管把 slave-read-only 设置为 no，其实 slave 也不会主动过期删除从 master 同步过来的数据的。<br><br>我猜老师想问的应该是：假设让 slave 也可以自动删除过期数据，是否可以保证主从库的一致性？<br><br>其实这样也无法保证，例如以下场景：<br><br>1、主从同步存在网络延迟。例如 master 先执行 SET key 1 10，这个 key 同步到了 slave，此时 key 在主从库都是 10s 后过期，之后这个 key 还剩 1s 过期时，master 又执行了 expire key 60，重设这个 key 的过期时间。但 expire 命令向 slave 同步时，发生了网络延迟并且超过了 1s，如果 slave 可以自动删除过期 key，那么这个 key 正好达到过期时间，就会被 slave 删除了，之后 slave 再收到 expire 命令时，执行会失败。最后的结果是这个 key 在 slave 上丢失了，主从库发生了不一致。<br><br>2、主从机器时钟不一致。同样 master 执行 SET key 1 10，然后把这个 key 同步到 slave，但是此时 slave 机器时钟如果发生跳跃，优先把这个 key 过期删除了，也会发生上面说的不一致问题。<br><br>所以 Redis 为了保证主从同步的一致性，不会让 slave 自动删除过期 key，而只在 master 删除过期 key，之后 master 会向 slave 发送一个 DEL，slave 再把这个 key 删除掉，这种方式可以解决主从网络延迟和机器时钟不一致带来的影响。<br><br>再解释一下 slave-read-only 的作用，它主要用来控制 slave 是否可写，但是否主动删除过期 key，根据 Redis 版本不同，执行逻辑也不同。<br><br>1、如果版本低于 Redis 4.0，slave-read-only 设置为 no，此时 slave 允许写入数据，但如果 key 设置了过期时间，那么这个 key 过期后，虽然在 slave 上查询不到了，但并不会在内存中删除，这些过期 key 会一直占着 Redis 内存无法释放。<br><br>2、Redis 4.0 版本解决了上述问题，在 slave 写入带过期时间的 key，slave 会记下这些 key，并且在后台定时检测这些 key 是否已过期，过期后从内存中删除。<br><br>但是请注意，这 2 种情况，slave 都不会主动删除由 *master 同步过来带有过期时间的 key*。也就是 master 带有过期时间的 key，什么时候删除由 master 自己维护，slave 不会介入。如果 slave 设置了 slave-read-only = no，而且是 4.0+ 版本，slave 也只维护直接向自己写入 的带有过期的 key，过期时只删除这些 key。<br><br>另外，我还能想到的主从同步的 2 个问题:<br><br>1、主从库设置的 maxmemory 不同，如果 slave 比 master 小，那么 slave 内存就会优先达到 maxmemroy，然后开始淘汰数据，此时主从库也会产生不一致。<br><br>2、如果主从同步的 client-output-buffer-limit 设置过小，并且 master 数据量很大，主从全量同步时可能会导致 buffer 溢出，溢出后主从全量同步就会失败。如果主从集群配置了哨兵，那么哨兵会让 slave 继续向 master 发起全量同步请求，然后 buffer 又溢出同步失败，如此反复，会形成复制风暴，这会浪费 master 大量的 CPU、内存、带宽资源，也会让 master 产生阻塞的风险。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢Kaito同学的回复和详细分析！很赞！<br><br>我也解释下，到时出这道题的一个考虑出发点。<br><br>这道题我其实是想问大家，假设从库也能直接删除过期数据的话，是不是一个好方法。其实，是想提醒下同学们，主从复制中的增删改都需要在主库执行，即使从库能做删除，也不要在从库删除。否则会造成数据不一致。例如，假设主从库上都能做写操作的话，主从库上有a:stock的键，客户端A给主库发送一个SET命令，修改a:stock的值，客户端B给从库发送了一个SET命令，也修改a:stock的值，此时，相同键的值就不一样了。所以，让从库可以做写操作会造成主从数据不一致。<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-02 00:13:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJRFRX8kNzNet7FibNvtavbVpAwK09AhIhrib9k762qWtH6mre8ickP7hM5mgZC4ytr8NnmIfmAhxMSQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>老大不小</span>
  </div>
  <div class="_2_QraFYR_0">老师，slave-serve-stale-data这个命令说的不清不楚的，对于初学者来说不明就里。<br><br>slave-serve-stale-data<br><br>解释：当一个slave与master失去联系时，或者复制正在进行的时候，slave应对请求的行为：1) 如果为 yes（默认值） ，slave 仍然会应答客户端请求，但返回的数据可能是过时，或者数据可能是空的在第一次同步的时候；2) 如果为 no ，在你执行除了 info 和 salveof 之外的其他命令时，slave 都将返回一个 &quot;SYNC with master in progress&quot; 的错误。<br><br><br>29、slave-read-only<br><br>解释：设置slave是否是只读的。从2.6版起，slave默认是只读的。<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-28 16:56:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/21/a6/7b/1b581f07.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>思变</span>
  </div>
  <div class="_2_QraFYR_0">老师您好,关于bind参数,不是设置redis能接受哪个本机网卡接入的连接吗?为什么要配置多个哨兵的IP呢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-04 13:53:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/cf/81/96f656ef.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>杨逸林</span>
  </div>
  <div class="_2_QraFYR_0">不是个好方法，如果不同客户端，去非当前从库读取数据时，就会出现缓存不一致的情况。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-02 07:45:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/d2/65/79d89c77.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小白白不白</span>
  </div>
  <div class="_2_QraFYR_0">EXPIRE、PEXPIRE和EXPIREAT三个命令都会转换成PEXPIREAT命令来执行,难道redis对于aof日志文件没有转为PEXPIREAT吗?</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-05 22:58:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>赵茭茭</span>
  </div>
  <div class="_2_QraFYR_0">主从复制 不是复制的是rdb吗 不是aof啊 这个和指令 带AT的还有效吗 还是我理解的有问题</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-07 19:47:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/10/5b/f700f9ef.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>刘浩</span>
  </div>
  <div class="_2_QraFYR_0">slave-serve-stale-data配置了主从中断后，从库的逻辑<br>--no ：从库只能应答INFO和SLAVEOF<br>--yes默认 ：正常应答<br>这样不知道对不对</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-13 17:54:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/87/dc/85515477.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>yu</span>
  </div>
  <div class="_2_QraFYR_0">redis为何不把底层的expire实现成为expireAt，再发给从库进行同步，避免出现过期数据问题？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-28 18:39:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/f7/72/e7bc6ff1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>零点999</span>
  </div>
  <div class="_2_QraFYR_0">只有主库提供服务，从库只保证高可用</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-02 14:17:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek1254</span>
  </div>
  <div class="_2_QraFYR_0">为什么不在主库删除过期key时给从库发送删除命令</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-20 18:52:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/7f/91/962eba1a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>唐朝首都</span>
  </div>
  <div class="_2_QraFYR_0">不是一个好方法，这样从库也能处理写命令，这样更容易造成主从不一致。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-06 08:08:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/21/a6/7b/1b581f07.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>思变</span>
  </div>
  <div class="_2_QraFYR_0">protected-mode我看参数文件的解释是,如果protected-mode设置为yes,如果实例未设置密码且未设置bind参数,只能通过127.0.0.1进行本地连接,是我理解的不对吗?</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-04 13:54:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/22/28/e8/7734b8d3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>P</span>
  </div>
  <div class="_2_QraFYR_0">仔细看了下，逻辑有问题，从库读到过期数据跟过期策略没有关系吧</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-03 17:09:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/24/cb/8b/fa482708.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Zhqqqq</span>
  </div>
  <div class="_2_QraFYR_0">老师问一下关于外部程序监控 Redis INFO replication 主从延迟信息来为客户端是否能访问改从节点。这个方案似乎就是把 AP 模型改为 CP。而且增加了架构复杂度，毕竟又增加了外部程序，在客户端进行监控不行吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-12 13:53:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/46/a0/aa6d4ecd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张潇赟</span>
  </div>
  <div class="_2_QraFYR_0">老师说的slave-serve-stale-data 这个参数在后面的版本应该是修改成了replica-serve-stale-data了吧。官方的配置文件中并没有找到slave-serve-stale-data这个配置参数<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-30 20:00:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>受超凡</span>
  </div>
  <div class="_2_QraFYR_0">老师好，有个问题请教下。主从同步不是通过RDB的方式进行的吗，从库读RDB的话不用执行命令吧，是直接读的数据吧，那从库为什么还会设置过期时间呢？不是直接读的吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-23 20:48:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKib3vNM6TPT1umvR3TictnLurJPKuQq4iblH5upgBB3kHL9hoN3Pgh3MaR2rjz6fWgMiaDpicd8R5wsAQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>陈阳</span>
  </div>
  <div class="_2_QraFYR_0">主从库在一个机房不会降低容灾性吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-07 20:11:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/d9/ff/b23018a6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Heaven</span>
  </div>
  <div class="_2_QraFYR_0">这个问题有些没看懂,是直接从从库上删除数据吗,这应该是不被允许的把,现在在云上用K8S管理,也是设置了多个service,一个用来写一个用来读,从库只用来读取数据,不应该负责删除数据<br>而且,即使设置了过期时间,从库在达到之后也不会删除数据,只是返回空,只有接到主库删除操作才会进行删除</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-27 11:42:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/28/ea/65/ab8748c5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>玄墨</span>
  </div>
  <div class="_2_QraFYR_0">过期键脏读问题,有个疑惑:老师建议使用expireat 和pexpireat 因为后面跟的是时间戳,而不是当前服务时间+过期时间.但是实际上redis内部在配置过期时间之后,不是都转换成了pexpireat并存储到过期字典中吗?所以这几个命令最后在redis过期字典中理论上表现是一致的,不应该出现老师说的因为命令不一样而出现脏读现象吧.麻烦老师解读下🙏</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-29 11:52:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/19/bf/415023b5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>璩雷</span>
  </div>
  <div class="_2_QraFYR_0">从库不能执行写操作，过期的数据是如何被删除的呢？具体过程是怎样的？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-27 20:20:43</div>
  </div>
</div>
</div>
</li>
</ul>