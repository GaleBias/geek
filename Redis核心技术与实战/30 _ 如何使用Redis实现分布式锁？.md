<audio title="30 _ 如何使用Redis实现分布式锁？" src="https://static001.geekbang.org/resource/audio/0e/69/0e517f6ef22d893534yyee6dc72da269.mp3" controls="controls"></audio> 
<p>你好，我是蒋德钧。</p><p>上节课，我提到，在应对并发问题时，除了原子操作，Redis客户端还可以通过加锁的方式，来控制并发写操作对共享数据的修改，从而保证数据的正确性。</p><p>但是，Redis属于分布式系统，当有多个客户端需要争抢锁时，我们必须要保证，<strong>这把锁不能是某个客户端本地的锁</strong>。否则的话，其它客户端是无法访问这把锁的，当然也就不能获取这把锁了。</p><p>所以，在分布式系统中，当有多个客户端需要获取锁时，我们需要分布式锁。此时，锁是保存在一个共享存储系统中的，可以被多个客户端共享访问和获取。</p><p>Redis本身可以被多个客户端共享访问，正好就是一个共享存储系统，可以用来保存分布式锁。而且Redis的读写性能高，可以应对高并发的锁操作场景。所以，这节课，我就来和你聊聊如何基于Redis实现分布式锁。</p><p>我们日常在写程序的时候，经常会用到单机上的锁，你应该也比较熟悉了。而分布式锁和单机上的锁既有相似性，但也因为分布式锁是用在分布式场景中，所以又具有一些特殊的要求。</p><p>所以，接下来，我就先带你对比下分布式锁和单机上的锁，找出它们的联系与区别，这样就可以加深你对分布式锁的概念和实现要求的理解。</p><h2>单机上的锁和分布式锁的联系与区别</h2><p>我们先来看下单机上的锁。</p><!-- [[[read_end]]] --><p>对于在单机上运行的多线程程序来说，锁本身可以用一个变量表示。</p><ul>
<li>变量值为0时，表示没有线程获取锁；</li>
<li>变量值为1时，表示已经有线程获取到锁了。</li>
</ul><p>我们通常说的线程调用加锁和释放锁的操作，到底是啥意思呢？我来解释一下。实际上，一个线程调用加锁操作，其实就是检查锁变量值是否为0。如果是0，就把锁的变量值设置为1，表示获取到锁，如果不是0，就返回错误信息，表示加锁失败，已经有别的线程获取到锁了。而一个线程调用释放锁操作，其实就是将锁变量的值置为0，以便其它线程可以来获取锁。</p><p>我用一段代码来展示下加锁和释放锁的操作，其中，lock为锁变量。</p><pre><code>acquire_lock(){
  if lock == 0
     lock = 1
     return 1
  else
     return 0
} 

release_lock(){
  lock = 0
  return 1
}
</code></pre><p>和单机上的锁类似，分布式锁同样可以<strong>用一个变量来实现</strong>。客户端加锁和释放锁的操作逻辑，也和单机上的加锁和释放锁操作逻辑一致：<strong>加锁时同样需要判断锁变量的值，根据锁变量值来判断能否加锁成功；释放锁时需要把锁变量值设置为0，表明客户端不再持有锁</strong>。</p><p>但是，和线程在单机上操作锁不同的是，在分布式场景下，<strong>锁变量需要由一个共享存储系统来维护</strong>，只有这样，多个客户端才可以通过访问共享存储系统来访问锁变量。相应的，<strong>加锁和释放锁的操作就变成了读取、判断和设置共享存储系统中的锁变量值</strong>。</p><p>这样一来，我们就可以得出实现分布式锁的两个要求。</p><ul>
<li>要求一：分布式锁的加锁和释放锁的过程，涉及多个操作。所以，在实现分布式锁时，我们需要保证这些锁操作的原子性；</li>
<li>要求二：共享存储系统保存了锁变量，如果共享存储系统发生故障或宕机，那么客户端也就无法进行锁操作了。在实现分布式锁时，我们需要考虑保证共享存储系统的可靠性，进而保证锁的可靠性。</li>
</ul><p>好了，知道了具体的要求，接下来，我们就来学习下Redis是怎么实现分布式锁的。</p><p>其实，我们既可以基于单个Redis节点来实现，也可以使用多个Redis节点实现。在这两种情况下，锁的可靠性是不一样的。我们先来看基于单个Redis节点的实现方法。</p><h2>基于单个Redis节点实现分布式锁</h2><p>作为分布式锁实现过程中的共享存储系统，Redis可以使用键值对来保存锁变量，再接收和处理不同客户端发送的加锁和释放锁的操作请求。那么，键值对的键和值具体是怎么定的呢？</p><p>我们要赋予锁变量一个变量名，把这个变量名作为键值对的键，而锁变量的值，则是键值对的值，这样一来，Redis就能保存锁变量了，客户端也就可以通过Redis的命令操作来实现锁操作。</p><p>为了帮助你理解，我画了一张图片，它展示Redis使用键值对保存锁变量，以及两个客户端同时请求加锁的操作过程。</p><p><img src="https://static001.geekbang.org/resource/image/1d/45/1d18742c1e5fc88835ec27f1becfc145.jpg?wh=2820*2250" alt=""></p><p>可以看到，Redis可以使用一个键值对lock_key:0来保存锁变量，其中，键是lock_key，也是锁变量的名称，锁变量的初始值是0。</p><p>我们再来分析下加锁操作。</p><p>在图中，客户端A和C同时请求加锁。因为Redis使用单线程处理请求，所以，即使客户端A和C同时把加锁请求发给了Redis，Redis也会串行处理它们的请求。</p><p>我们假设Redis先处理客户端A的请求，读取lock_key的值，发现lock_key为0，所以，Redis就把lock_key的value置为1，表示已经加锁了。紧接着，Redis处理客户端C的请求，此时，Redis会发现lock_key的值已经为1了，所以就返回加锁失败的信息。</p><p>刚刚说的是加锁的操作，那释放锁该怎么操作呢？其实，释放锁就是直接把锁变量值设置为0。</p><p>我还是借助一张图片来解释一下。这张图片展示了客户端A请求释放锁的过程。当客户端A持有锁时，锁变量lock_key的值为1。客户端A执行释放锁操作后，Redis将lock_key的值置为0，表明已经没有客户端持有锁了。</p><p><img src="https://static001.geekbang.org/resource/image/c7/82/c7c413b47d42f06f08fce92404f31e82.jpg?wh=3000*2250" alt=""></p><p>因为加锁包含了三个操作（读取锁变量、判断锁变量值以及把锁变量值设置为1），而这三个操作在执行时需要保证原子性。那怎么保证原子性呢？</p><p>上节课，我们学过，要想保证操作的原子性，有两种通用的方法，分别是使用Redis的单命令操作和使用Lua脚本。那么，在分布式加锁场景下，该怎么应用这两个方法呢？</p><p>我们先来看下，Redis可以用哪些单命令操作实现加锁操作。</p><p>首先是SETNX命令，它用于设置键值对的值。具体来说，就是这个命令在执行时会判断键值对是否存在，如果不存在，就设置键值对的值，如果存在，就不做任何设置。</p><p>举个例子，如果执行下面的命令时，key不存在，那么key会被创建，并且值会被设置为value；如果key已经存在，SETNX不做任何赋值操作。</p><pre><code>SETNX key value
</code></pre><p>对于释放锁操作来说，我们可以在执行完业务逻辑后，使用DEL命令删除锁变量。不过，你不用担心锁变量被删除后，其他客户端无法请求加锁了。因为SETNX命令在执行时，如果要设置的键值对（也就是锁变量）不存在，SETNX命令会先创建键值对，然后设置它的值。所以，释放锁之后，再有客户端请求加锁时，SETNX命令会创建保存锁变量的键值对，并设置锁变量的值，完成加锁。</p><p>总结来说，我们就可以用SETNX和DEL命令组合来实现加锁和释放锁操作。下面的伪代码示例显示了锁操作的过程，你可以看下。</p><pre><code>// 加锁
SETNX lock_key 1
// 业务逻辑
DO THINGS
// 释放锁
DEL lock_key
</code></pre><p>不过，使用SETNX和DEL命令组合实现分布锁，存在两个潜在的风险。</p><p>第一个风险是，假如某个客户端在执行了SETNX命令、加锁之后，紧接着却在操作共享数据时发生了异常，结果一直没有执行最后的DEL命令释放锁。因此，锁就一直被这个客户端持有，其它客户端无法拿到锁，也无法访问共享数据和执行后续操作，这会给业务应用带来影响。</p><p>针对这个问题，一个有效的解决方法是，<strong>给锁变量设置一个过期时间</strong>。这样一来，即使持有锁的客户端发生了异常，无法主动地释放锁，Redis也会根据锁变量的过期时间，在锁变量过期后，把它删除。其它客户端在锁变量过期后，就可以重新请求加锁，这就不会出现无法加锁的问题了。</p><p>我们再来看第二个风险。如果客户端A执行了SETNX命令加锁后，假设客户端B执行了DEL命令释放锁，此时，客户端A的锁就被误释放了。如果客户端C正好也在申请加锁，就可以成功获得锁，进而开始操作共享数据。这样一来，客户端A和C同时在对共享数据进行操作，数据就会被修改错误，这也是业务层不能接受的。</p><p>为了应对这个问题，我们需要<strong>能区分来自不同客户端的锁操作</strong>，具体咋做呢？其实，我们可以在锁变量的值上想想办法。</p><p>在使用SETNX命令进行加锁的方法中，我们通过把锁变量值设置为1或0，表示是否加锁成功。1和0只有两种状态，无法表示究竟是哪个客户端进行的锁操作。所以，我们在加锁操作时，可以让每个客户端给锁变量设置一个唯一值，这里的唯一值就可以用来标识当前操作的客户端。在释放锁操作时，客户端需要判断，当前锁变量的值是否和自己的唯一标识相等，只有在相等的情况下，才能释放锁。这样一来，就不会出现误释放锁的问题了。</p><p>知道了解决方案，那么，在Redis中，具体是怎么实现的呢？我们再来了解下。</p><p>在查看具体的代码前，我要先带你学习下Redis的SET命令。</p><p>我们刚刚在说SETNX命令的时候提到，对于不存在的键值对，它会先创建再设置值（也就是“不存在即设置”），为了能达到和SETNX命令一样的效果，Redis给SET命令提供了类似的选项NX，用来实现“不存在即设置”。如果使用了NX选项，SET命令只有在键值对不存在时，才会进行设置，否则不做赋值操作。此外，SET命令在执行时还可以带上EX或PX选项，用来设置键值对的过期时间。</p><p>举个例子，执行下面的命令时，只有key不存在时，SET才会创建key，并对key进行赋值。另外，<strong>key的存活时间由seconds或者milliseconds选项值来决定</strong>。</p><pre><code>SET key value [EX seconds | PX milliseconds]  [NX]
</code></pre><p>有了SET命令的NX和EX/PX选项后，我们就可以用下面的命令来实现加锁操作了。</p><pre><code>// 加锁, unique_value作为客户端唯一性的标识
SET lock_key unique_value NX PX 10000
</code></pre><p>其中，unique_value是客户端的唯一标识，可以用一个随机生成的字符串来表示，PX 10000则表示lock_key会在10s后过期，以免客户端在这期间发生异常而无法释放锁。</p><p>因为在加锁操作中，每个客户端都使用了一个唯一标识，所以在释放锁操作时，我们需要判断锁变量的值，是否等于执行释放锁操作的客户端的唯一标识，如下所示：</p><pre><code>//释放锁 比较unique_value是否相等，避免误释放
if redis.call(&quot;get&quot;,KEYS[1]) == ARGV[1] then
    return redis.call(&quot;del&quot;,KEYS[1])
else
    return 0
end
</code></pre><p>这是使用Lua脚本（unlock.script）实现的释放锁操作的伪代码，其中，KEYS[1]表示lock_key，ARGV[1]是当前客户端的唯一标识，这两个值都是我们在执行Lua脚本时作为参数传入的。</p><p>最后，我们执行下面的命令，就可以完成锁释放操作了。</p><pre><code>redis-cli  --eval  unlock.script lock_key , unique_value 
</code></pre><p>你可能也注意到了，在释放锁操作中，我们使用了Lua脚本，这是因为，释放锁操作的逻辑也包含了读取锁变量、判断值、删除锁变量的多个操作，而Redis在执行Lua脚本时，可以以原子性的方式执行，从而保证了锁释放操作的原子性。</p><p>好了，到这里，你了解了如何使用SET命令和Lua脚本在Redis单节点上实现分布式锁。但是，我们现在只用了一个Redis实例来保存锁变量，如果这个Redis实例发生故障宕机了，那么锁变量就没有了。此时，客户端也无法进行锁操作了，这就会影响到业务的正常执行。所以，我们在实现分布式锁时，还需要保证锁的可靠性。那怎么提高呢？这就要提到基于多个Redis节点实现分布式锁的方式了。</p><h2>基于多个Redis节点实现高可靠的分布式锁</h2><p>当我们要实现高可靠的分布式锁时，就不能只依赖单个的命令操作了，我们需要按照一定的步骤和规则进行加解锁操作，否则，就可能会出现锁无法工作的情况。“一定的步骤和规则”是指啥呢？其实就是分布式锁的算法。</p><p>为了避免Redis实例故障而导致的锁无法工作的问题，Redis的开发者Antirez提出了分布式锁算法Redlock。</p><p>Redlock算法的基本思路，是让客户端和多个独立的Redis实例依次请求加锁，如果客户端能够和半数以上的实例成功地完成加锁操作，那么我们就认为，客户端成功地获得分布式锁了，否则加锁失败。这样一来，即使有单个Redis实例发生故障，因为锁变量在其它实例上也有保存，所以，客户端仍然可以正常地进行锁操作，锁变量并不会丢失。</p><p>我们来具体看下Redlock算法的执行步骤。Redlock算法的实现需要有N个独立的Redis实例。接下来，我们可以分成3步来完成加锁操作。</p><p><strong>第一步是，客户端获取当前时间。</strong></p><p><strong>第二步是，客户端按顺序依次向N个Redis实例执行加锁操作。</strong></p><p>这里的加锁操作和在单实例上执行的加锁操作一样，使用SET命令，带上NX，EX/PX选项，以及带上客户端的唯一标识。当然，如果某个Redis实例发生故障了，为了保证在这种情况下，Redlock算法能够继续运行，我们需要给加锁操作设置一个超时时间。</p><p>如果客户端在和一个Redis实例请求加锁时，一直到超时都没有成功，那么此时，客户端会和下一个Redis实例继续请求加锁。加锁操作的超时时间需要远远地小于锁的有效时间，一般也就是设置为几十毫秒。</p><p><strong>第三步是，一旦客户端完成了和所有Redis实例的加锁操作，客户端就要计算整个加锁过程的总耗时。</strong></p><p>客户端只有在满足下面的这两个条件时，才能认为是加锁成功。</p><ul>
<li>条件一：客户端从超过半数（大于等于 N/2+1）的Redis实例上成功获取到了锁；</li>
<li>条件二：客户端获取锁的总耗时没有超过锁的有效时间。</li>
</ul><p>在满足了这两个条件后，我们需要重新计算这把锁的有效时间，计算的结果是锁的最初有效时间减去客户端为获取锁的总耗时。如果锁的有效时间已经来不及完成共享数据的操作了，我们可以释放锁，以免出现还没完成数据操作，锁就过期了的情况。</p><p>当然，如果客户端在和所有实例执行完加锁操作后，没能同时满足这两个条件，那么，客户端向所有Redis节点发起释放锁的操作。</p><p>在Redlock算法中，释放锁的操作和在单实例上释放锁的操作一样，只要执行释放锁的Lua脚本就可以了。这样一来，只要N个Redis实例中的半数以上实例能正常工作，就能保证分布式锁的正常工作了。</p><p>所以，在实际的业务应用中，如果你想要提升分布式锁的可靠性，就可以通过Redlock算法来实现。</p><h2>小结</h2><p>分布式锁是由共享存储系统维护的变量，多个客户端可以向共享存储系统发送命令进行加锁或释放锁操作。Redis作为一个共享存储系统，可以用来实现分布式锁。</p><p>在基于单个Redis实例实现分布式锁时，对于加锁操作，我们需要满足三个条件。</p><ol>
<li>加锁包括了读取锁变量、检查锁变量值和设置锁变量值三个操作，但需要以原子操作的方式完成，所以，我们使用SET命令带上NX选项来实现加锁；</li>
<li>锁变量需要设置过期时间，以免客户端拿到锁后发生异常，导致锁一直无法释放，所以，我们在SET命令执行时加上EX/PX选项，设置其过期时间；</li>
<li>锁变量的值需要能区分来自不同客户端的加锁操作，以免在释放锁时，出现误释放操作，所以，我们使用SET命令设置锁变量值时，每个客户端设置的值是一个唯一值，用于标识客户端。</li>
</ol><p>和加锁类似，释放锁也包含了读取锁变量值、判断锁变量值和删除锁变量三个操作，不过，我们无法使用单个命令来实现，所以，我们可以采用Lua脚本执行释放锁操作，通过Redis原子性地执行Lua脚本，来保证释放锁操作的原子性。</p><p>不过，基于单个Redis实例实现分布式锁时，会面临实例异常或崩溃的情况，这会导致实例无法提供锁操作，正因为此，Redis也提供了Redlock算法，用来实现基于多个实例的分布式锁。这样一来，锁变量由多个实例维护，即使有实例发生了故障，锁变量仍然是存在的，客户端还是可以完成锁操作。Redlock算法是实现高可靠分布式锁的一种有效解决方案，你可以在实际应用中把它用起来。</p><h2>每课一问</h2><p>按照惯例，我给你提个小问题。这节课，我提到，我们可以使用SET命令带上NX和EX/PX选项进行加锁操作，那么，我想请你再思考一下，我们是否可以用下面的方式来实现加锁操作呢？</p><pre><code>// 加锁
SETNX lock_key unique_value
EXPIRE lock_key 10S
// 业务逻辑
DO THINGS
</code></pre><p>欢迎在留言区写下你的思考和答案，我们一起交流讨论。如果你觉得今天的内容对你有所帮助，也欢迎你分享给你的朋友或同事。我们下节课见。</p>
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
  <div class="_2_QraFYR_0">是否可以使用 SETNX + EXPIRE 来完成加锁操作？<br><br>不可以这么使用。使用 2 个命令无法保证操作的原子性，在异常情况下，加锁结果会不符合预期。异常情况主要分为以下几种情况：<br><br>1、SETNX 执行成功，执行 EXPIRE 时由于网络问题设置过期失败<br><br>2、SETNX 执行成功，此时 Redis 实例宕机，EXPIRE 没有机会执行<br><br>3、SETNX 执行成功，客户端异常崩溃，EXPIRE 没有机会执行<br><br>如果发生以上情况，并且客户端在释放锁时发生异常，没有正常释放锁，那么这把锁就会一直无法释放，其他线程都无法再获得锁。<br><br>下面说一下关于 Redis 分布式锁可靠性的问题。<br><br>使用单个 Redis 节点（只有一个master）使用分布锁，如果实例宕机，那么无法进行锁操作了。那么采用主从集群模式部署是否可以保证锁的可靠性？<br><br>答案是也很难保证。如果在 master 上加锁成功，此时 master 宕机，由于主从复制是异步的，加锁操作的命令还未同步到 slave，此时主从切换，新 master 节点依旧会丢失该锁，对业务来说相当于锁失效了。<br><br>所以 Redis 作者才提出基于多个 Redis 节点（master节点）的 Redlock 算法，但这个算法涉及的细节很多，作者在提出这个算法时，业界的分布式系统专家还与 Redis 作者发生过一场争论，来评估这个算法的可靠性，争论的细节都是关于异常情况可能导致 Redlock 失效的场景，例如加锁过程中客户端发生了阻塞、机器时钟发生跳跃等等。<br><br>感兴趣的可以看下这篇文章，详细介绍了争论的细节，以及 Redis 分布式锁在各种异常情况是否安全的分析，收益会非常大：http:&#47;&#47;zhangtielei.com&#47;posts&#47;blog-redlock-reasoning.html。<br><br>简单总结，基于 Redis 使用分布锁的注意点：<br><br>1、使用 SET $lock_key $unique_val EX $second NX 命令保证加锁原子性，并为锁设置过期时间<br><br>2、锁的过期时间要提前评估好，要大于操作共享资源的时间<br><br>3、每个线程加锁时设置随机值，释放锁时判断是否和加锁设置的值一致，防止自己的锁被别人释放<br><br>4、释放锁时使用 Lua 脚本，保证操作的原子性<br><br>5、基于多个节点的 Redlock，加锁时超过半数节点操作成功，并且获取锁的耗时没有超过锁的有效时间才算加锁成功<br><br>6、Redlock 释放锁时，要对所有节点释放（即使某个节点加锁失败了），因为加锁时可能发生服务端加锁成功，由于网络问题，给客户端回复网络包失败的情况，所以需要把所有节点可能存的锁都释放掉<br><br>7、使用 Redlock 时要避免机器时钟发生跳跃，需要运维来保证，对运维有一定要求，否则可能会导致 Redlock 失效。例如共 3 个节点，线程 A 操作 2 个节点加锁成功，但其中 1 个节点机器时钟发生跳跃，锁提前过期，线程 B 正好在另外 2 个节点也加锁成功，此时 Redlock 相当于失效了（Redis 作者和分布式系统专家争论的重要点就在这）<br><br>8、如果为了效率，使用基于单个 Redis 节点的分布式锁即可，此方案缺点是允许锁偶尔失效，优点是简单效率高<br><br>9、如果是为了正确性，业务对于结果要求非常严格，建议使用 Redlock，但缺点是使用比较重，部署成本高</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-28 00:09:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/26/38/ef063dc2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Darren</span>
  </div>
  <div class="_2_QraFYR_0">其实分布式锁选择Etcd的话，可能会更好。<br><br>Etcd 支持以下功能，正是依赖这些功能来实现分布式锁的：<br><br>1、Lease 即租约机制（TTL，Time To Live）机制：<br><br>Etcd 可以为存储的 KV 对设置租约，当租约到期，KV 将失效删除；同时也支持续约，即 KeepAlive；<br><br>Redis在这方面很难实现，一般假设通过SETNX设置的时间10S，如果发生网络抖动，万一业务执行超过10S，此时别的线程就能回去到锁；<br><br><br>2、Revision 机制：<br><br>Etcd 每个 key 带有一个 Revision 属性值，每进行一次事务对应的全局 Revision 值都会加一，因此每个 key 对应的 Revision 属性值都是全局唯一的。通过比较 Revision 的大小就可以知道进行写操作的顺序。 在实现分布式锁时，多个程序同时抢锁，根据 Revision 值大小依次获得锁，可以避免 “惊群效应”，实现公平锁；<br><br>Redis很难实现公平锁，而且在某些情况下，也会产生 “惊群效应”；<br> <br><br>3、Prefix 即前缀机制，也称目录机制：<br><br>Etcd 可以根据前缀（目录）获取该目录下所有的 key 及对应的属性（包括 key, value 以及 revision 等）；<br><br>Redis也可以的，使用keys命令或者scan，生产环境一定要使用scan；<br><br><br>4、Watch 机制：<br><br>Etcd Watch 机制支持 Watch 某个固定的 key，也支持 Watch 一个目录（前缀机制），当被 Watch 的 key 或目录发生变化，客户端将收到通知；<br><br>Redis只能通过客户端定时轮训的形式去判断key是否存在；</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-28 17:40:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/54/9a/76c0af70.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>每天晒白牙</span>
  </div>
  <div class="_2_QraFYR_0">配合这篇文章看，效果更佳 https:&#47;&#47;mp.weixin.qq.com&#47;s&#47;2P2-ujcdet9cUycokfyjHQ<br><br>这篇是早晨通勤坐地铁开了一个番茄钟看完的，番茄钟开启学霸模式，将这些学习和读书的app加入白名单，在一个番茄钟周期内就不能打开那些未加入白名单的app，可以防止刷耗精力的app，工作时也经常用，效果不错<br><br>如果自己能总结出来并给别人讲明白，就像课代表一样，每篇文章的留言都非常优秀，每次必看，这个知识点就掌握了，能用到工作实战上，那就锦上添花了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-30 10:10:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/37/29/b3af57a7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>凯文小猪</span>
  </div>
  <div class="_2_QraFYR_0">这里老师漏了一点 就是session timeout处理 在分布式锁的场景中就是：<br>一个key过期了 但是代码还没处理完 此时就发生了重复加锁的问题。<br><br>通常我们有两种方式处理：<br>1. 设置看门狗 也就是redision的处理方式<br>2. 设置状态机 由最后的业务层来做代码回溯</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-10 20:13:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/36/2c/8bd4be3a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小喵喵</span>
  </div>
  <div class="_2_QraFYR_0">请教下老师文章中说的客户端，这个客户端指的是什么？比如浏览器下单，app下单，这个浏览器，app就是客户端吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 客户端是指向Redis发送读写请求的应用程序。你举的例子中，app的下单请求先是被发送到后端的服务器上，一般来说，服务器上还有业务程序先解析了app的请求，然后再向Redis发送读写请求，在这种情况下，服务器上的业务程序就是Redis的客户端。而app、浏览器算是服务器上业务程序的客户端。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-13 14:04:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/28/94/ec/8db3f04a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>喰</span>
  </div>
  <div class="_2_QraFYR_0">对于给锁设置过期时间，即使再怎么预估也很难保证线程在锁有效时间内完成操作。而且预估时间设置的过大也会影响系统性能，所以可以使用一个守护线程进行续租。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-02 20:29:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/0c/d3/0be6ae81.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>COLDLY</span>
  </div>
  <div class="_2_QraFYR_0">老师，请问锁的过期时间怎么设置，会不会因为客户端执行业务操作时耗时太久超过过期时间，导致锁过期被释放了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-05 23:37:29</div>
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
  <div class="_2_QraFYR_0">不可以。这两个命令不是原子的。发生异常的时候，可能会有正常的加锁结果。<br><br>分布式锁:<br>1.唯一性<br>2.原子性<br>3.可重入<br><br>Redlock Redisson zookeeper etcd都是业界常用的分布式锁方案。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-03 12:00:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/16/eb/30864e40.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>漂泊者及其影子</span>
  </div>
  <div class="_2_QraFYR_0"><br>&#47;&#47;释放锁 比较unique_value是否相等，避免误释放<br>if redis.call(&quot;get&quot;,KEYS[1]) == ARGV[1] then<br>    return redis.call(&quot;del&quot;,KEYS[1])<br>else<br>    return 0<br>end<br><br>释放锁为什么不能使用delete操作？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-30 17:48:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/54/9a/76c0af70.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>每天晒白牙</span>
  </div>
  <div class="_2_QraFYR_0">是不能把setnx 和 expire 命令分开的，因为无法保证两个操作执行的原子性，可能遇到各种异常，无法满足预期</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-30 09:23:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLXDG2ux03bmIyEu9cR1SGX9QnDLJmSpYBg6CcfL6uUiaL90ypGIdmjXaj9uYLaxkQVoKSDeuAiaobQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_lijia</span>
  </div>
  <div class="_2_QraFYR_0">关于分布式锁，两个大神的争论，我站Martin这边。<br>https:&#47;&#47;martin.kleppmann.com&#47;2016&#47;02&#47;08&#47;how-to-do-distributed-locking.html <br>http:&#47;&#47;antirez.com&#47;news&#47;101 </div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-13 16:54:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/28/6c/bc/f751786b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Ming</span>
  </div>
  <div class="_2_QraFYR_0">课后问题回答：<br>不可以，枷锁和给定过期时间是两个命令非原子性操作，若在加完锁后redis奔溃那么该锁永远也不会有过期时间；<br>所以建议使用指令：set key value ex|px nx；<br><br>另外：redis官方已不推荐使用redlock了！！！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-11 16:30:03</div>
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
  <div class="_2_QraFYR_0">首先回答下这个问题,如果不保证原子性,那么实例宕机了,岂不是没有别人能拿到锁了,这是大问题<br>第二个问题,按照老师说的Redlock,达到半数以上,那么由一个问题,就是出现集群实例宕机一部分,导致其他客户端也达到了获取锁的数量(即在集群半数上加锁成功),那时候怎么办</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-25 16:59:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/ce/ce/53392e44.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>BingoJ</span>
  </div>
  <div class="_2_QraFYR_0">目前这把锁还有一个问题，就是不能保证可重入</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-31 15:10:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/23/5b/983408b9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>悟空聊架构</span>
  </div>
  <div class="_2_QraFYR_0">这节课，我提到，我们可以使用 SET 命令带上 NX 和 EX&#47;PX 选项进行加锁操作，那么，我想请你再思考一下，我们是否可以用下面的方式来实现加锁操作呢？<br><br><br><br>&#47;&#47; 加锁<br>SETNX lock_key unique_value<br>EXPIRE lock_key 10S<br>&#47;&#47; 业务逻辑<br>DO THINGS<br><br>答：非原子操作，所以不能用这种方式。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-08 20:18:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/78/99/6060eb2d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>平凡之路</span>
  </div>
  <div class="_2_QraFYR_0">老师，您好，使用redis集群，使用setnx能保障加锁成功吗？即使一个实例挂了，还有其他实例能读取并加锁吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-03 08:32:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_9a0c9f</span>
  </div>
  <div class="_2_QraFYR_0">python的简单实现:<br>import threading<br>import time<br><br>import redis<br><br>redis_con = redis.StrictRedis(host=&#39;127.0.0.1&#39;, port=6379, db=0)<br><br># redis_con.ttl(&#39;name&#39;)<br>redis_con.set(&quot;global&quot;, 10)<br><br>#守护进程，用来检测时间是否到期，快到期了，就续命<br>def add_time(redis_con, key_name):<br>    while True:<br>        time = redis_con.ttl(key_name)<br>        if time == 1:<br>            print(&quot;=====&quot;)<br>            redis_con.expire(key_name, 2)<br>            print(redis_con.ttl(key_name))<br><br>def test_distribute_lock():<br>    thead_id = str(time.time())<br>    if_has_key = redis_con.setnx(&quot;lock&quot;, thead_id)<br>    try:<br>        if not if_has_key:<br>            print(&quot;等待请稍后重试&quot;)<br>        else:<br>            redis_con.expire(&#39;lock&#39;, 2)<br>            # 开启守护进程<br>            t1 = threading.Thread(target=add_time, args=(redis_con, &#39;lock&#39;))<br>            t1.setDaemon(True)<br>            t1.start()<br>            redis_con.decr(&#39;global&#39;, 1)<br>            time.sleep(2)<br>    except Exception as e:<br>        print(e)<br>    finally:<br>                 # 执行完毕，释放锁，保证每个线程只能删除自己的锁<br>        if str(redis_con.get(&quot;lock&quot;), encoding=&quot;utf-8&quot;) == thead_id:<br>            redis_con.delete(&#39;lock&#39;)<br>        redis_con.close()<br><br>if __name__ == &#39;__main__&#39;:<br>    for i in range(5):<br>        t = threading.Thread(target=test_distribute_lock)<br>        t.start()<br>        t.join()<br>    print(&quot;===&gt;&quot;)</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-29 13:38:19</div>
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
  <div class="_2_QraFYR_0">不能这样做，因为两个命令就不是原子操作了。<br><br>set nx px的时候如果拿到锁的客户端在使用过程中超出了其设置的超时时间，那么就有这把锁同时被两个客户端持有的风险，所以需要在使用过程中不断去更新其过期时间。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-28 08:53:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/db/26/54f2c164.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>靠人品去赢</span>
  </div>
  <div class="_2_QraFYR_0">这个分布式锁，实际上靠一个共享的锁变量标识去锁资源还是释放资源，其他的实例依靠这个共享的变量去锁与释放资源。而不是像单机那样直接在单机本地去锁。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-02 21:11:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/56/ea/32608c44.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>giteebravo</span>
  </div>
  <div class="_2_QraFYR_0">回答每课一问：不能将 SETNX 和 EXPIRE 命令分开来执行加锁。因为这样无法保证加锁操作的原子性。<br>再问下老师：<br>1 如何使用 Redis 客户端执行释放锁的 Lua 脚本？<br>2 Redlock 算法具体要怎么来实现？新版本的 Redis 是否已经实现？<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-08 16:12:05</div>
  </div>
</div>
</div>
</li>
</ul>