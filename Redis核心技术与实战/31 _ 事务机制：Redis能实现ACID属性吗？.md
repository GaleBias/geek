<audio title="31 _ 事务机制：Redis能实现ACID属性吗？" src="https://static001.geekbang.org/resource/audio/30/eb/3053a049b7df4e99db44167310569eeb.mp3" controls="controls"></audio> 
<p>你好，我是蒋德钧。</p><p>事务是数据库的一个重要功能。所谓的事务，就是指对数据进行读写的一系列操作。事务在执行时，会提供专门的属性保证，包括原子性（Atomicity）、一致性（Consistency）、隔离性（Isolation）和持久性（Durability），也就是ACID属性。这些属性既包括了对事务执行结果的要求，也有对数据库在事务执行前后的数据状态变化的要求。</p><p>那么，Redis可以完全保证ACID属性吗？毕竟，如果有些属性在一些场景下不能保证的话，很可能会导致数据出错，所以，我们必须要掌握Redis对这些属性的支持情况，并且提前准备应对策略。</p><p>接下来，我们就先了解ACID属性对事务执行的具体要求，有了这个知识基础后，我们才能准确地判断Redis的事务机制能否保证ACID属性。</p><h2>事务ACID属性的要求</h2><p>首先来看原子性。原子性的要求很明确，就是一个事务中的多个操作必须都完成，或者都不完成。业务应用使用事务时，原子性也是最被看重的一个属性。</p><p>我给你举个例子。假如用户在一个订单中购买了两个商品A和B，那么，数据库就需要把这两个商品的库存都进行扣减。如果只扣减了一个商品的库存，那么，这个订单完成后，另一个商品的库存肯定就错了。</p><!-- [[[read_end]]] --><p>第二个属性是一致性。这个很容易理解，就是指数据库中的数据在事务执行前后是一致的。</p><p>第三个属性是隔离性。它要求数据库在执行一个事务时，其它操作无法存取到正在执行事务访问的数据。</p><p>我还是借助用户下单的例子给你解释下。假设商品A和B的现有库存分别是5和10，用户X对A、B下单的数量分别是3、6。如果事务不具备隔离性，在用户X下单事务执行的过程中，用户Y一下子也购买了5件B，这和X购买的6件B累加后，就超过B的总库存值了，这就不符合业务要求了。</p><p>最后一个属性是持久性。数据库执行事务后，数据的修改要被持久化保存下来。当数据库重启后，数据的值需要是被修改后的值。</p><p>了解了ACID属性的具体要求后，我们再来看下Redis是如何实现事务机制的。</p><h2>Redis如何实现事务？</h2><p>事务的执行过程包含三个步骤，Redis提供了MULTI、EXEC两个命令来完成这三个步骤。下面我们来分析下。</p><p>第一步，客户端要使用一个命令显式地表示一个事务的开启。在Redis中，这个命令就是MULTI。</p><p>第二步，客户端把事务中本身要执行的具体操作（例如增删改数据）发送给服务器端。这些操作就是Redis本身提供的数据读写命令，例如GET、SET等。不过，这些命令虽然被客户端发送到了服务器端，但Redis实例只是把这些命令暂存到一个命令队列中，并不会立即执行。</p><p>第三步，客户端向服务器端发送提交事务的命令，让数据库实际执行第二步中发送的具体操作。Redis提供的<strong>EXEC命令</strong>就是执行事务提交的。当服务器端收到EXEC命令后，才会实际执行命令队列中的所有命令。</p><p>下面的代码就显示了使用MULTI和EXEC执行一个事务的过程，你可以看下。</p><pre><code>#开启事务
127.0.0.1:6379&gt; MULTI
OK
#将a:stock减1，
127.0.0.1:6379&gt; DECR a:stock
QUEUED
#将b:stock减1
127.0.0.1:6379&gt; DECR b:stock
QUEUED
#实际执行事务
127.0.0.1:6379&gt; EXEC
1) (integer) 4
2) (integer) 9
</code></pre><p>我们假设a:stock、b:stock两个键的初始值是5和10。在MULTI命令后执行的两个DECR命令，是把a:stock、b:stock两个键的值分别减1，它们执行后的返回结果都是QUEUED，这就表示，这些操作都被暂存到了命令队列，还没有实际执行。等到执行了EXEC命令后，可以看到返回了4、9，这就表明，两个DECR命令已经成功地执行了。</p><p>好了，通过使用MULTI和EXEC命令，我们可以实现多个操作的共同执行，但是这符合事务要求的ACID属性吗？接下来，我们就来具体分析下。</p><h2>Redis的事务机制能保证哪些属性？</h2><p>原子性是事务操作最重要的一个属性，所以，我们先来分析下Redis事务机制能否保证原子性。</p><h3>原子性</h3><p>如果事务正常执行，没有发生任何错误，那么，MULTI和EXEC配合使用，就可以保证多个操作都完成。但是，如果事务执行发生错误了，原子性还能保证吗？我们需要分三种情况来看。</p><p>第一种情况是，<strong>在执行EXEC命令前，客户端发送的操作命令本身就有错误</strong>（比如语法错误，使用了不存在的命令），在命令入队时就被Redis实例判断出来了。</p><p>对于这种情况，在命令入队时，Redis就会报错并且记录下这个错误。此时，我们还能继续提交命令操作。等到执行了EXEC命令之后，Redis就会拒绝执行所有提交的命令操作，返回事务失败的结果。这样一来，事务中的所有命令都不会再被执行了，保证了原子性。</p><p>我们来看一个因为事务操作入队时发生错误，而导致事务失败的小例子。</p><pre><code>#开启事务
127.0.0.1:6379&gt; MULTI
OK
#发送事务中的第一个操作，但是Redis不支持该命令，返回报错信息
127.0.0.1:6379&gt; PUT a:stock 5
(error) ERR unknown command `PUT`, with args beginning with: `a:stock`, `5`, 
#发送事务中的第二个操作，这个操作是正确的命令，Redis把该命令入队
127.0.0.1:6379&gt; DECR b:stock
QUEUED
#实际执行事务，但是之前命令有错误，所以Redis拒绝执行
127.0.0.1:6379&gt; EXEC
(error) EXECABORT Transaction discarded because of previous errors.
</code></pre><p>在这个例子中，事务里包含了一个Redis本身就不支持的PUT命令，所以，在PUT命令入队时，Redis就报错了。虽然，事务里还有一个正确的DECR命令，但是，在最后执行EXEC命令后，整个事务被放弃执行了。</p><p>我们再来看第二种情况。</p><p>和第一种情况不同的是，<strong>事务操作入队时，命令和操作的数据类型不匹配，但Redis实例没有检查出错误</strong>。但是，在执行完EXEC命令以后，Redis实际执行这些事务操作时，就会报错。不过，需要注意的是，虽然Redis会对错误命令报错，但还是会把正确的命令执行完。在这种情况下，事务的原子性就无法得到保证了。</p><p>举个小例子。事务中的LPOP命令对String类型数据进行操作，入队时没有报错，但是，在EXEC执行时报错了。LPOP命令本身没有执行成功，但是事务中的DECR命令却成功执行了。</p><pre><code>#开启事务
127.0.0.1:6379&gt; MULTI
OK
#发送事务中的第一个操作，LPOP命令操作的数据类型不匹配，此时并不报错
127.0.0.1:6379&gt; LPOP a:stock
QUEUED
#发送事务中的第二个操作
127.0.0.1:6379&gt; DECR b:stock
QUEUED
#实际执行事务，事务第一个操作执行报错
127.0.0.1:6379&gt; EXEC
1) (error) WRONGTYPE Operation against a key holding the wrong kind of value
2) (integer) 8
</code></pre><p>看到这里，你可能有个疑问，传统数据库（例如MySQL）在执行事务时，会提供回滚机制，当事务执行发生错误时，事务中的所有操作都会撤销，已经修改的数据也会被恢复到事务执行前的状态，那么，在刚才的例子中，如果命令实际执行时报错了，是不是可以用回滚机制恢复原来的数据呢？</p><p>其实，Redis中并没有提供回滚机制。虽然Redis提供了DISCARD命令，但是，这个命令只能用来主动放弃事务执行，把暂存的命令队列清空，起不到回滚的效果。</p><p>DISCARD命令具体怎么用呢？我们来看下下面的代码。</p><pre><code>#读取a:stock的值4
127.0.0.1:6379&gt; GET a:stock
&quot;4&quot;
#开启事务
127.0.0.1:6379&gt; MULTI 
OK
#发送事务的第一个操作，对a:stock减1
127.0.0.1:6379&gt; DECR a:stock
QUEUED
#执行DISCARD命令，主动放弃事务
127.0.0.1:6379&gt; DISCARD
OK
#再次读取a:stock的值，值没有被修改
127.0.0.1:6379&gt; GET a:stock
&quot;4&quot;
</code></pre><p>这个例子中，a:stock键的值一开始为4，然后，我们执行一个事务，想对a:stock的值减1。但是，在事务的最后，我们执行的是DISCARD命令，所以事务就被放弃了。我们再次查看a:stock的值，会发现仍然为4。</p><p>最后，我们再来看下第三种情况：<strong>在执行事务的EXEC命令时，Redis实例发生了故障，导致事务执行失败</strong>。</p><p>在这种情况下，如果Redis开启了AOF日志，那么，只会有部分的事务操作被记录到AOF日志中。我们需要使用redis-check-aof工具检查AOF日志文件，这个工具可以把未完成的事务操作从AOF文件中去除。这样一来，我们使用AOF恢复实例后，事务操作不会再被执行，从而保证了原子性。</p><p>当然，如果AOF日志并没有开启，那么实例重启后，数据也都没法恢复了，此时，也就谈不上原子性了。</p><p>好了，到这里，你了解了Redis对事务原子性属性的保证情况，我们来简单小结下：</p><ul>
<li>命令入队时就报错，会放弃事务执行，保证原子性；</li>
<li>命令入队时没报错，实际执行时报错，不保证原子性；</li>
<li>EXEC命令执行时实例故障，如果开启了AOF日志，可以保证原子性。</li>
</ul><p>接下来，我们再来学习下一致性属性的保证情况。</p><h3>一致性</h3><p>事务的一致性保证会受到错误命令、实例故障的影响。所以，我们按照命令出错和实例故障的发生时机，分成三种情况来看。</p><p><strong>情况一：命令入队时就报错</strong></p><p>在这种情况下，事务本身就会被放弃执行，所以可以保证数据库的一致性。</p><p><strong>情况二：命令入队时没报错，实际执行时报错</strong></p><p>在这种情况下，有错误的命令不会被执行，正确的命令可以正常执行，也不会改变数据库的一致性。</p><p><strong>情况三：EXEC命令执行时实例发生故障</strong></p><p>在这种情况下，实例故障后会进行重启，这就和数据恢复的方式有关了，我们要根据实例是否开启了RDB或AOF来分情况讨论下。</p><p>如果我们没有开启RDB或AOF，那么，实例故障重启后，数据都没有了，数据库是一致的。</p><p>如果我们使用了RDB快照，因为RDB快照不会在事务执行时执行，所以，事务命令操作的结果不会被保存到RDB快照中，使用RDB快照进行恢复时，数据库里的数据也是一致的。</p><p>如果我们使用了AOF日志，而事务操作还没有被记录到AOF日志时，实例就发生了故障，那么，使用AOF日志恢复的数据库数据是一致的。如果只有部分操作被记录到了AOF日志，我们可以使用redis-check-aof清除事务中已经完成的操作，数据库恢复后也是一致的。</p><p>所以，总结来说，在命令执行错误或Redis发生故障的情况下，Redis事务机制对一致性属性是有保证的。接下来，我们再继续分析下隔离性。</p><h3>隔离性</h3><p>事务的隔离性保证，会受到和事务一起执行的并发操作的影响。而事务执行又可以分成命令入队（EXEC命令执行前）和命令实际执行（EXEC命令执行后）两个阶段，所以，我们就针对这两个阶段，分成两种情况来分析：</p><ol>
<li>并发操作在EXEC命令前执行，此时，隔离性的保证要使用WATCH机制来实现，否则隔离性无法保证；</li>
<li>并发操作在EXEC命令后执行，此时，隔离性可以保证。</li>
</ol><p>我们先来看第一种情况。一个事务的EXEC命令还没有执行时，事务的命令操作是暂存在命令队列中的。此时，如果有其它的并发操作，我们就需要看事务是否使用了WATCH机制。</p><p>WATCH机制的作用是，在事务执行前，监控一个或多个键的值变化情况，当事务调用EXEC命令执行时，WATCH机制会先检查监控的键是否被其它客户端修改了。如果修改了，就放弃事务执行，避免事务的隔离性被破坏。然后，客户端可以再次执行事务，此时，如果没有并发修改事务数据的操作了，事务就能正常执行，隔离性也得到了保证。</p><p>WATCH机制的具体实现是由WATCH命令实现的，我给你举个例子，你可以看下下面的图，进一步理解下WATCH命令的使用。</p><p><img src="https://static001.geekbang.org/resource/image/4f/73/4f8589410f77df16311dd29131676373.jpg?wh=3000*1921" alt=""></p><p>我来给你具体解释下图中的内容。</p><p>在t1时，客户端X向实例发送了WATCH命令。实例收到WATCH命令后，开始监测a:stock的值的变化情况。</p><p>紧接着，在t2时，客户端X把MULTI命令和DECR命令发送给实例，实例把DECR命令暂存入命令队列。</p><p>在t3时，客户端Y也给实例发送了一个DECR命令，要修改a:stock的值，实例收到命令后就直接执行了。</p><p>等到t4时，实例收到客户端X发送的EXEC命令，但是，实例的WATCH机制发现a:stock已经被修改了，就会放弃事务执行。这样一来，事务的隔离性就可以得到保证了。</p><p>当然，如果没有使用WATCH机制，在EXEC命令前执行的并发操作是会对数据进行读写的。而且，在执行EXEC命令的时候，事务要操作的数据已经改变了，在这种情况下，Redis并没有做到让事务对其它操作隔离，隔离性也就没有得到保障。下面这张图显示了没有WATCH机制时的情况，你可以看下。</p><p><img src="https://static001.geekbang.org/resource/image/8c/57/8ca37debfff91282b9c62a25fd7e9a57.jpg?wh=3000*1562" alt=""></p><p>在t2时刻，客户端X发送的EXEC命令还没有执行，但是客户端Y的DECR命令就执行了，此时，a:stock的值会被修改，这就无法保证X发起的事务的隔离性了。</p><p>刚刚说的是并发操作在EXEC命令前执行的情况，下面我再来说一说第二种情况：<strong>并发操作在EXEC命令之后被服务器端接收并执行</strong>。</p><p>因为Redis是用单线程执行命令，而且，EXEC命令执行后，Redis会保证先把命令队列中的所有命令执行完。所以，在这种情况下，并发操作不会破坏事务的隔离性，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/11/ae/11a1eff930920a0b423a6e46c23f44ae.jpg?wh=2958*1799" alt=""></p><p>最后，我们来分析一下Redis事务的持久性属性保证情况。</p><h3>持久性</h3><p>因为Redis是内存数据库，所以，数据是否持久化保存完全取决于Redis的持久化配置模式。</p><p>如果Redis没有使用RDB或AOF，那么事务的持久化属性肯定得不到保证。如果Redis使用了RDB模式，那么，在一个事务执行后，而下一次的RDB快照还未执行前，如果发生了实例宕机，这种情况下，事务修改的数据也是不能保证持久化的。</p><p>如果Redis采用了AOF模式，因为AOF模式的三种配置选项no、everysec和always都会存在数据丢失的情况，所以，事务的持久性属性也还是得不到保证。</p><p>所以，不管Redis采用什么持久化模式，事务的持久性属性是得不到保证的。</p><h2>小结</h2><p>在这节课上，我们学习了Redis中的事务实现。Redis通过MULTI、EXEC、DISCARD和WATCH四个命令来支持事务机制，这4个命令的作用，我总结在下面的表中，你可以再看下。</p><p><img src="https://static001.geekbang.org/resource/image/95/50/9571308df0620214d7ccb2f2cc73a250.jpg?wh=2505*734" alt=""></p><p>事务的ACID属性是我们使用事务进行正确操作的基本要求。通过这节课的分析，我们了解到了，Redis的事务机制可以保证一致性和隔离性，但是无法保证持久性。不过，因为Redis本身是内存数据库，持久性并不是一个必须的属性，我们更加关注的还是原子性、一致性和隔离性这三个属性。</p><p>原子性的情况比较复杂，只有当事务中使用的命令语法有误时，原子性得不到保证，在其它情况下，事务都可以原子性执行。</p><p>所以，我给你一个小建议：<strong>严格按照Redis的命令规范进行程序开发，并且通过code review确保命令的正确性</strong>。这样一来，Redis的事务机制就能被应用在实践中，保证多操作的正确执行。</p><h2>每课一问</h2><p>按照惯例，我给你提个小问题，在执行事务时，如果Redis实例发生故障，而Redis使用了RDB机制，那么，事务的原子性还能得到保证吗？</p><p>欢迎在留言区写下你的思考和答案，我们一起交流讨论。如果你觉得今天的内容对你有所帮助，也欢迎你分享给你的朋友或同事。我们下节课见。</p>
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
  <div class="_2_QraFYR_0">在执行事务时，如果 Redis 实例发生故障，而 Redis 使用的 RDB 机制，事务的原子性还能否得到保证？<br><br>我觉得是可以保证原子性的。<br><br>如果一个事务只执行了一半，然后 Redis 实例故障宕机了，由于 RDB 不会在事务执行时执行，所以 RDB 文件中不会记录只执行了一部分的结果数据。之后用 RDB 恢复实例数据，恢复的还是事务之前的数据。但 RDB 本身是快照持久化，所以会存在数据丢失，丢失的是距离上一次 RDB 之间的所有更改操作。<br><br>关于 Redis 事务的使用，有几个细节我觉得有必要补充下，关于 Pipeline 和 WATCH 命令的使用。<br><br>1、在使用事务时，建议配合 Pipeline 使用。<br><br>a) 如果不使用 Pipeline，客户端是先发一个 MULTI 命令到服务端，客户端收到 OK，然后客户端再发送一个个操作命令，客户端依次收到 QUEUED，最后客户端发送 EXEC 执行整个事务（文章例子就是这样演示的），这样消息每次都是一来一回，效率比较低，而且在这多次操作之间，别的客户端可能就把原本准备修改的值给修改了，所以无法保证隔离性。<br><br>b) 而使用 Pipeline 是一次性把所有命令打包好全部发送到服务端，服务端全部处理完成后返回。这么做好的好处，一是减少了来回网络 IO 次数，提高操作性能。二是一次性发送所有命令到服务端，服务端在处理过程中，是不会被别的请求打断的（Redis单线程特性，此时别的请求进不来），这本身就保证了隔离性。我们平时使用的 Redis SDK 在使用开启事务时，一般都会默认开启 Pipeline 的，可以留意观察一下。<br><br>2、关于 WATCH 命令的使用场景。<br><br>a) 在上面 1-a 场景中，也就是使用了事务命令，但没有配合 Pipeline 使用，如果想要保证隔离性，需要使用 WATCH 命令保证，也就是文章中讲 WATCH 的例子。但如果是 1-b 场景，使用了 Pipeline 一次发送所有命令到服务端，那么就不需要使用 WATCH 了，因为服务端本身就保证了隔离性。<br><br>b) 如果事务 + Pipeline 就可以保证隔离性，那 WATCH 还有没有使用的必要？答案是有的。对于一个资源操作为读取、修改、写回这种场景，如果需要保证事物的原子性，此时就需要用到 WATCH 了。例如想要修改某个资源，但需要事先读取它的值，再基于这个值进行计算后写回，如果在这期间担心这个资源被其他客户端修改了，那么可以先 WATCH 这个资源，再读取、修改、写回，如果写回成功，说明其他客户端在这期间没有修改这个资源。如果其他客户端修改了这个资源，那么这个事务操作会返回失败，不会执行，从而保证了原子性。<br><br>细节比较多，如果不太好理解，最好亲自动手试一下。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-30 00:12:21</div>
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
  <div class="_2_QraFYR_0">	• 原子性就是一批操作，要不全部完成，要不一个也不执行。<br>	• 原子性的结果就是中间结果对外不可见，如果中间结果对外可见，则一致性就不会得到满足（比如操作）。<br>	• 而隔离性，指一个事务内部的操作及使用的数据对正在进行的其他事务是隔离的，并发执行的各个事务之间不能互相干扰，正是它保证了原子操作的过程中，中间结果对其它事务不可见。<br><br>本文在讨论一致性的时候，说到“ 命令入队时没报错，实际执行时报错在这种情况下，有错误的命令不会被执行，正确的命令可以正常执行，也不会改变数据库的一致性。”，我觉得这一点是存疑的，不保证原子性就保证不了一致性。比如转账操作，扣减转出账户的操作成功，增加转入账户的操作失败，则原子性和一致性都被破坏。<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-02 16:50:06</div>
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
  <div class="_2_QraFYR_0">redis开启RDB，因为RDB不会在事务执行的时候执行，所以是可以保证原子性的</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-30 09:01:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/0a/a4/828a431f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张申傲</span>
  </div>
  <div class="_2_QraFYR_0">Redis 为什么不支持事务的回滚？可以参考下官网的解释：https:&#47;&#47;redis.io&#47;topics&#47;transactions </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 官网是个非常好的学习地方 :)</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-18 14:00:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/49/a5/e4c1c2d4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小文同学</span>
  </div>
  <div class="_2_QraFYR_0">1、作者讲了什么？<br>作者通过本文讨论，Redis 是否可以保证 ACID 的事务功能。事务是对数据进行一系列操作。<br>2、作者是怎么把事情说明白的？<br>作者先讨论 事务ACID 属性的要求：然后作者说明了 Redis 的 API ：MULTI 和 EXEC 是如何完成事务的；完成说明后，作者开始针对事务的每个特性，讨论 Redis 是否已经完成达成。<br>2.1 原子性。原子性的保证分三种情况<br>2.1.1 队列中有命令存在错误，队列清空；（可保证原子）<br>2.1.2 队列中命令到执行的时候才被发现有错误，不会滚，执行多少算多少；（不保证原子）<br>2.1.3 EXEC 时， Redis 实例发生故障。这个涉及到日志，AOF 的 redis-check-aof 可以发现没执行完成的操作，进而清除；（可以保证原子）<br>2.2 一致性。作者分三种情况说明，并且确认都可以提供一致性。<br>2.3 隔离性。WATCH 机制提供事务隔离性。<br>2.4 持久性。Redis 任何时候都无法提供持久性保障。<br><br>3、为了讲明白，作者讲了哪些要点？哪些是亮点？<br>在 Redis 的事务上，作者通过 三种情况 ，分别说明了 Redis 是否满足 ACID 特性，这个划分方法是一个亮点；<br><br>4、对于作者所讲，我有哪些发散性思考？<br>Redis 始终坚持是一个高性能的内存数据库，并没有因为事务的重要性而放弃这一个宗旨，故在内存中实现了隔离性，一致性，有条件原子性，不实现持久性。这个也可以放映出 Redis 的定位和一般数据库 MySQL 是不一样的；<br><br>5、在未来哪些场景，我可以使用它？<br>在高并发，竞争环境下，需要保证数据正确时，可以考虑 Redis 的事务性实现。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-04 13:45:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/29/a1/41607383.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>hello</span>
  </div>
  <div class="_2_QraFYR_0">“错误的命令不会被执行，正确的命令可以正常执行，也不会改变数据库的一致性”这个怎么就没有改变数据库的一致性了呢？我是菜鸟一枚，有大神指点一二吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-12 00:03:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>与君共勉</span>
  </div>
  <div class="_2_QraFYR_0">AOF如果开启always模式不是可以保证数据不丢失吗？为啥也保证不了持久性呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-30 10:42:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/20/99/5e/33481a74.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Lemon</span>
  </div>
  <div class="_2_QraFYR_0">老师，我对Redis能保证一致性这点表示困惑：在命令入队时没有报错，实际执行时报错的情况下，如果A给B转账，A的账户被扣钱了，此时命令出错，B账户并没有增加转账金额，这不就导致了事务前后的数据不一致了吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-02 15:34:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/c9/5e/b79e6d5d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>꧁子华宝宝萌萌哒꧂</span>
  </div>
  <div class="_2_QraFYR_0">老师，在上一节中，分布式锁重要的就是保证操作的原子性，既然事物能保证原子性，为啥上一节中没有提到用事物来做呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-30 09:06:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>JohnReese</span>
  </div>
  <div class="_2_QraFYR_0">那请问老师，Multi 命令 和 Lua脚本 的功能上有什么区别嘛？（似乎都是保证‘原子性’地执行多命令？）</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-30 09:12:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/60/a1/8f003697.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>静心</span>
  </div>
  <div class="_2_QraFYR_0">老师，在集群模式下，ACID是个什么情况？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-30 07:10:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/9c/53/ade0afb0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ub8</span>
  </div>
  <div class="_2_QraFYR_0">老师redis的pipline 可以保证执行命令结果集按顺序返回么：具体操作如下：<br>Pipeline pipeline = jedis.pipelined();<br>            Map&lt;Integer,PrizeConfig&gt; day2PrizeConfigMap = new HashMap&lt;&gt;();<br>            Map&lt;Integer,redis.clients.jedis.Response&lt;Set&lt;Tuple&gt;&gt;&gt; day2PrizeCache = new HashMap&lt;&gt;();<br>            for (Integer dayIndex : days) {<br>                String cacheKey = String.format(SignInConstants.PRIZE_VERSION_KEY,<br>                        dayIndex);<br>                redis.clients.jedis.Response&lt;Set&lt;Tuple&gt;&gt; prizeConfigsResponse  =<br>                        pipeline.zrevrangeByScoreWithScores(cacheKey, version, 0);<br>                day2PrizeCache.put(dayIndex, prizeConfigsResponse);<br>            }<br>            pipeline.sync();<br>&#47;&#47; 我的问题是 day2PrizeCache 这个map中的 dayIndex 和 返回值prizeConfigsResponse 可以对应关系有可能发生错位吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-26 21:04:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJF1c2h1dCWAkdvs049lb3y7vzIicvv2kZOZwEFpUyhxmmehdpVicWGBaSsGv2TPkuastTW0MgxoLxg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>吕宁博</span>
  </div>
  <div class="_2_QraFYR_0">显然大家对关系型数据库ACID的C没理解透彻。C是通过用户自定义的一系列“数据类型、约束、触发器”等保证的。就拿银行取钱来说，如果用户没设置check(balance&gt;=0)，即使最终因为各种原因导致balance&lt;0，那也没违反C。数据库是在一定的系统+用户规则下运行的，只要没违反规则，就是保证了C。<br><br>数据库只是个软件，理解C的时候不要把现实世界的规则强加给它，除非你明确告诉它规则（数据类型、约束、触发器），否则它就是一个满足前后一致性状态规则的C。<br><br>这样理解C比较简单：我（数据库）管你balance是不是小于0，你又没告诉我（设置check），那我小于0违法了吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-22 18:25:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Z8t0JKFjnmdx4s4wuRePZXRL2L9awEpicp0rjT9rfXmZKOBIleZuOC86OzZE0tSdkfy3LWWa7YU67MicWeiaFd3jA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小白</span>
  </div>
  <div class="_2_QraFYR_0">文中的例子 DISCARD ，这个起到的效果不是和回滚的效果是一样的嘛，为什么不能算回滚？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-20 11:28:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/MOuCWWOnoQjOr8KjicQ84R7xu6DRcfDv3VAuHseGJ1gxXicKJboA24vOcrcJickTJPwFAU38VuwCGGkGq7f8WkTIg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_b14c55</span>
  </div>
  <div class="_2_QraFYR_0">如果使用了rdb机制，事务的原子性没办法保证，因为已经有部分数据落盘了，所以没办法保证原子性</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-07 23:38:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/22/bd/9b/366bb87b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>飞龙</span>
  </div>
  <div class="_2_QraFYR_0">我测试的结果貌似和有些例子不太一样，开启multi lpop a:stock;decr b:stock  exec   因为a:stock类型对不上报错，b:stock并没有执行。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-09 16:05:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>风之翼</span>
  </div>
  <div class="_2_QraFYR_0">在使用 watch 机制后，若存在并发写多个变量的情况，在 watch 到变量发生变化后，停止事务执行前，已经做的修改会回滚吗</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-05 17:12:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKq0oQVibKcmYJqmpqaNNQibVgia7EsEgW65LZJIpDZBMc7FyMcs7J1JmFCtp06pY8ibbcpW4ibRtG7Frg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>zhoufeng</span>
  </div>
  <div class="_2_QraFYR_0">既然redis能实现事务的隔离性，为什么在应对多客户端并发访问时，还需要争抢分布式锁呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-25 20:56:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/9c/53/ade0afb0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ub8</span>
  </div>
  <div class="_2_QraFYR_0">越看评论越蒙</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-26 19:29:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/24/27/77/8493aa4a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>伟</span>
  </div>
  <div class="_2_QraFYR_0">可以</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-17 19:49:01</div>
  </div>
</div>
</div>
</li>
</ul>