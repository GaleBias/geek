<audio title="29 _ 无锁的原子操作：Redis如何应对并发访问？" src="https://static001.geekbang.org/resource/audio/84/e3/846079205efc98381146183fa72df4e3.mp3" controls="controls"></audio> 
<p>你好，我是蒋德钧。</p><p>我们在使用Redis时，不可避免地会遇到并发访问的问题，比如说如果多个用户同时下单，就会对缓存在Redis中的商品库存并发更新。一旦有了并发写操作，数据就会被修改，如果我们没有对并发写请求做好控制，就可能导致数据被改错，影响到业务的正常使用（例如库存数据错误，导致下单异常）。</p><p>为了保证并发访问的正确性，Redis提供了两种方法，分别是加锁和原子操作。</p><p>加锁是一种常用的方法，在读取数据前，客户端需要先获得锁，否则就无法进行操作。当一个客户端获得锁后，就会一直持有这把锁，直到客户端完成数据更新，才释放这把锁。</p><p>看上去好像是一种很好的方案，但是，其实这里会有两个问题：一个是，如果加锁操作多，会降低系统的并发访问性能；第二个是，Redis客户端要加锁时，需要用到分布式锁，而分布式锁实现复杂，需要用额外的存储系统来提供加解锁操作，我会在下节课向你介绍。</p><p><strong>原子操作是另一种提供并发访问控制的方法</strong>。原子操作是指执行过程保持原子性的操作，而且原子操作执行时并不需要再加锁，实现了无锁操作。这样一来，既能保证并发控制，还能减少对系统并发性能的影响。</p><p>这节课，我就来和你聊聊Redis中的原子操作。原子操作的目标是实现并发访问控制，那么当有并发访问请求时，我们具体需要控制什么呢？接下来，我就先向你介绍下并发控制的内容。</p><!-- [[[read_end]]] --><h2>并发访问中需要对什么进行控制？</h2><p>我们说的并发访问控制，是指对多个客户端访问操作同一份数据的过程进行控制，以保证任何一个客户端发送的操作在Redis实例上执行时具有互斥性。例如，客户端A的访问操作在执行时，客户端B的操作不能执行，需要等到A的操作结束后，才能执行。</p><p>并发访问控制对应的操作主要是数据修改操作。当客户端需要修改数据时，基本流程分成两步：</p><ol>
<li>客户端先把数据读取到本地，在本地进行修改；</li>
<li>客户端修改完数据后，再写回Redis。</li>
</ol><p>我们把这个流程叫做“读取-修改-写回”操作（Read-Modify-Write，简称为RMW操作）。当有多个客户端对同一份数据执行RMW操作的话，我们就需要让RMW操作涉及的代码以原子性方式执行。访问同一份数据的RMW操作代码，就叫做临界区代码。</p><p>不过，当有多个客户端并发执行临界区代码时，就会存在一些潜在问题，接下来，我用一个多客户端更新商品库存的例子来解释一下。</p><p>我们先看下临界区代码。假设客户端要对商品库存执行扣减1的操作，伪代码如下所示：</p><pre><code>current = GET(id)
current--
SET(id, current)
</code></pre><p>可以看到，客户端首先会根据商品id，从Redis中读取商品当前的库存值current（对应Read)，然后，客户端对库存值减1（对应Modify），再把库存值写回Redis（对应Write）。当有多个客户端执行这段代码时，这就是一份临界区代码。</p><p>如果我们对临界区代码的执行没有控制机制，就会出现数据更新错误。在刚才的例子中，假设现在有两个客户端A和B，同时执行刚才的临界区代码，就会出现错误，你可以看下下面这张图。</p><p><img src="https://static001.geekbang.org/resource/image/dc/5c/dce821cd00c1937b4aab1f130424335c.jpg?wh=2925*1801" alt=""></p><p>可以看到，客户端A在t1时读取库存值10并扣减1，在t2时，客户端A还没有把扣减后的库存值9写回Redis，而在此时，客户端B读到库存值10，也扣减了1，B记录的库存值也为9了。等到t3时，A往Redis写回了库存值9，而到t4时，B也写回了库存值9。</p><p>如果按正确的逻辑处理，客户端A和B对库存值各做了一次扣减，库存值应该为8。所以，这里的库存值明显更新错了。</p><p>出现这个现象的原因是，临界区代码中的客户端读取数据、更新数据、再写回数据涉及了三个操作，而这三个操作在执行时并不具有互斥性，多个客户端基于相同的初始值进行修改，而不是基于前一个客户端修改后的值再修改。</p><p>为了保证数据并发修改的正确性，我们可以用锁把并行操作变成串行操作，串行操作就具有互斥性。一个客户端持有锁后，其他客户端只能等到锁释放，才能拿锁再进行修改。</p><p>下面的伪代码显示了使用锁来控制临界区代码的执行情况，你可以看下。</p><pre><code>LOCK()
current = GET(id)
current--
SET(id, current)
UNLOCK()
</code></pre><p>虽然加锁保证了互斥性，但是<strong>加锁也会导致系统并发性能降低</strong>。</p><p>如下图所示，当客户端A加锁执行操作时，客户端B、C就需要等待。A释放锁后，假设B拿到锁，那么C还需要继续等待，所以，t1时段内只有A能访问共享数据，t2时段内只有B能访问共享数据，系统的并发性能当然就下降了。</p><p><img src="https://static001.geekbang.org/resource/image/84/25/845b4694700264482d64a3dbb7a36525.jpg?wh=2620*1414" alt=""></p><p>和加锁类似，原子操作也能实现并发控制，但是原子操作对系统并发性能的影响较小，接下来，我们就来了解下Redis中的原子操作。</p><h2>Redis的两种原子操作方法</h2><p>为了实现并发控制要求的临界区代码互斥执行，Redis的原子操作采用了两种方法：</p><ol>
<li>把多个操作在Redis中实现成一个操作，也就是单命令操作；</li>
<li>把多个操作写到一个Lua脚本中，以原子性方式执行单个Lua脚本。</li>
</ol><p>我们先来看下Redis本身的单命令操作。</p><p>Redis是使用单线程来串行处理客户端的请求操作命令的，所以，当Redis执行某个命令操作时，其他命令是无法执行的，这相当于命令操作是互斥执行的。当然，Redis的快照生成、AOF重写这些操作，可以使用后台线程或者是子进程执行，也就是和主线程的操作并行执行。不过，这些操作只是读取数据，不会修改数据，所以，我们并不需要对它们做并发控制。</p><p>你可能也注意到了，虽然Redis的单个命令操作可以原子性地执行，但是在实际应用中，数据修改时可能包含多个操作，至少包括读数据、数据增减、写回数据三个操作，这显然就不是单个命令操作了，那该怎么办呢？</p><p>别担心，Redis提供了INCR/DECR命令，把这三个操作转变为一个原子操作了。INCR/DECR命令可以对数据进行<strong>增值/减值</strong>操作，而且它们本身就是单个命令操作，Redis在执行它们时，本身就具有互斥性。</p><p>比如说，在刚才的库存扣减例子中，客户端可以使用下面的代码，直接完成对商品id的库存值减1操作。即使有多个客户端执行下面的代码，也不用担心出现库存值扣减错误的问题。</p><pre><code>DECR id 
</code></pre><p>所以，如果我们执行的RMW操作是对数据进行增减值的话，Redis提供的原子操作INCR和DECR可以直接帮助我们进行并发控制。</p><p>但是，如果我们要执行的操作不是简单地增减数据，而是有更加复杂的判断逻辑或者是其他操作，那么，Redis的单命令操作已经无法保证多个操作的互斥执行了。所以，这个时候，我们需要使用第二个方法，也就是Lua脚本。</p><p>Redis会把整个Lua脚本作为一个整体执行，在执行的过程中不会被其他命令打断，从而保证了Lua脚本中操作的原子性。如果我们有多个操作要执行，但是又无法用INCR/DECR这种命令操作来实现，就可以把这些要执行的操作编写到一个Lua脚本中。然后，我们可以使用Redis的EVAL命令来执行脚本。这样一来，这些操作在执行时就具有了互斥性。</p><p>我再给你举个例子，来具体解释下Lua的使用。</p><p>当一个业务应用的访问用户增加时，我们有时需要限制某个客户端在一定时间范围内的访问次数，比如爆款商品的购买限流、社交网络中的每分钟点赞次数限制等。</p><p>那该怎么限制呢？我们可以把客户端IP作为key，把客户端的访问次数作为value，保存到Redis中。客户端每访问一次后，我们就用INCR增加访问次数。</p><p>不过，在这种场景下，客户端限流其实同时包含了对访问次数和时间范围的限制，例如每分钟的访问次数不能超过20。所以，我们可以在客户端第一次访问时，给对应键值对设置过期时间，例如设置为60s后过期。同时，在客户端每次访问时，我们读取客户端当前的访问次数，如果次数超过阈值，就报错，限制客户端再次访问。你可以看下下面的这段代码，它实现了对客户端每分钟访问次数不超过20次的限制。</p><pre><code>//获取ip对应的访问次数
current = GET(ip)
//如果超过访问次数超过20次，则报错
IF current != NULL AND current &gt; 20 THEN
    ERROR &quot;exceed 20 accesses per second&quot;
ELSE
    //如果访问次数不足20次，增加一次访问计数
    value = INCR(ip)
    //如果是第一次访问，将键值对的过期时间设置为60s后
    IF value == 1 THEN
        EXPIRE(ip,60)
    END
    //执行其他操作
    DO THINGS
END
</code></pre><p>可以看到，在这个例子中，我们已经使用了INCR来原子性地增加计数。但是，客户端限流的逻辑不只有计数，还包括<strong>访问次数判断和过期时间设置</strong>。</p><p>对于这些操作，我们同样需要保证它们的原子性。否则，如果客户端使用多线程访问，访问次数初始值为0，第一个线程执行了INCR(ip)操作后，第二个线程紧接着也执行了INCR(ip)，此时，ip对应的访问次数就被增加到了2，我们就无法再对这个ip设置过期时间了。这样就会导致，这个ip对应的客户端访问次数达到20次之后，就无法再进行访问了。即使过了60s，也不能再继续访问，显然不符合业务要求。</p><p>所以，这个例子中的操作无法用Redis单个命令来实现，此时，我们就可以使用Lua脚本来保证并发控制。我们可以把访问次数加1、判断访问次数是否为1，以及设置过期时间这三个操作写入一个Lua脚本，如下所示：</p><pre><code>local current
current = redis.call(&quot;incr&quot;,KEYS[1])
if tonumber(current) == 1 then
    redis.call(&quot;expire&quot;,KEYS[1],60)
end
</code></pre><p>假设我们编写的脚本名称为lua.script，我们接着就可以使用Redis客户端，带上eval选项，来执行该脚本。脚本所需的参数将通过以下命令中的keys和args进行传递。</p><pre><code>redis-cli  --eval lua.script  keys , args
</code></pre><p>这样一来，访问次数加1、判断访问次数是否为1，以及设置过期时间这三个操作就可以原子性地执行了。即使客户端有多个线程同时执行这个脚本，Redis也会依次串行执行脚本代码，避免了并发操作带来的数据错误。</p><h2>小结</h2><p>在并发访问时，并发的RMW操作会导致数据错误，所以需要进行并发控制。所谓并发控制，就是要保证临界区代码的互斥执行。</p><p>Redis提供了两种原子操作的方法来实现并发控制，分别是单命令操作和Lua脚本。因为原子操作本身不会对太多的资源限制访问，可以维持较高的系统并发性能。</p><p>但是，单命令原子操作的适用范围较小，并不是所有的RMW操作都能转变成单命令的原子操作（例如INCR/DECR命令只能在读取数据后做原子增减），当我们需要对读取的数据做更多判断，或者是我们对数据的修改不是简单的增减时，单命令操作就不适用了。</p><p>而Redis的Lua脚本可以包含多个操作，这些操作都会以原子性的方式执行，绕开了单命令操作的限制。不过，如果把很多操作都放在Lua脚本中原子执行，会导致Redis执行脚本的时间增加，同样也会降低Redis的并发性能。所以，我给你一个小建议：<strong>在编写Lua脚本时，你要避免把不<strong><strong>需要</strong></strong>做并发控制的操作写入脚本中</strong>。</p><p>当然，加锁也能实现临界区代码的互斥执行，只是如果有多个客户端加锁时，就需要分布式锁的支持了。所以，下节课，我就来和你聊聊分布式锁的实现。</p><h2>每课一问</h2><p>按照惯例，我向你提个小问题，Redis在执行Lua脚本时，是可以保证原子性的，那么，在我举的Lua脚本例子（lua.script）中，你觉得是否需要把读取客户端ip的访问次数，也就是GET(ip)，以及判断访问次数是否超过20的判断逻辑，也加到Lua脚本中吗？</p><p>欢迎在留言区写下你的思考和答案，我们一起交流讨论。如果你觉得今天的内容对你有所帮助，也欢迎你分享给你的朋友或同事。我们下节课见。</p>
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
  <div class="_2_QraFYR_0">是否需要把读取客户端 ip 的访问次数 GET(ip)，以及判断访问次数是否超过 20 的判断逻辑，也加到 Lua 脚本中？<br><br>我觉得不需要，理由主要有2个。<br><br>1、这2个逻辑都是读操作，不会对资源临界区产生修改，所以不需要做并发控制。<br><br>2、减少 lua 脚本中的命令，可以降低Redis执行脚本的时间，避免阻塞 Redis。<br><br>另外使用lua脚本时，还有一些注意点：<br><br>1、lua 脚本尽量只编写通用的逻辑代码，避免直接写死变量。变量通过外部调用方传递进来，这样 lua 脚本的可复用度更高。<br><br>2、建议先使用SCRIPT LOAD命令把 lua 脚本加载到 Redis 中，然后得到一个脚本唯一摘要值，再通过EVALSHA命令 + 脚本摘要值来执行脚本，这样可以避免每次发送脚本内容到 Redis，减少网络开销。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-26 01:25:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_fb6ea6</span>
  </div>
  <div class="_2_QraFYR_0">Redis不是单线程吗？怎么还会有并发读写的问题呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这里的并发是指有多个客户端同时访问Redis，而客户端执行的业务逻辑不只一个命令操作。假设客户端A要先从Redis读数据，然后做了修改把数据再写回Redis，此时，在读数据和写回数据的期间就可能有其他客户端的并发操作执行，所以仍然存在并发读写问题。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-04 15:46:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/ff/51/9d5cfadd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>好运来</span>
  </div>
  <div class="_2_QraFYR_0">“对于这些操作，我们同样需要保证它们的原子性。否则，如果客户端使用多线程访问，访问次数初始值为 0，第一个线程执行了 INCR(ip) 操作后，第二个线程紧接着也执行了 INCR(ip)，此时，ip 对应的访问次数就被增加到了 2，我们就无法再对这个 ip 设置过期时间了。这样就会导致，这个 ip 对应的客户端访问次数达到 20 次之后，就无法再进行访问了。即使过了 60s，也不能再继续访问，显然不符合业务要求。&quot;<br><br>对于这段话我有疑惑，假如有两个线程A和线程B，初始ip计数是0，线程A和线程B并发执行，不管线程A和线程B谁先执行到 value = INCR(ip) ，获取到的value值总会有一个是1，而value作为线程的局部变量，也是可以继续执行下去，那不就是能够执行到 IF value == 1 THEN        EXPIRE(ip,60)    END 这个判断逻辑了吗，不明白为什么说不能设置先到的ip过期时间60s了?<br><br><br><br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-01 22:02:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>dfuru</span>
  </div>
  <div class="_2_QraFYR_0">获取访问次数和判断访问是否大于20，<br>若放到lua脚本中，获取到的访问次数是准确的最新值，进行判断更准确；<br>当放到lua脚本外，并发访问时某线程获取到的访问次数可能旧(偏小)，当获取到访问次数为19时(实际可能已经达到20了)，该线程仍然会对访问次数执行+1，所以应该放到lua中。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-07 10:57:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/cb/f9/75d08ccf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Mr.蜜</span>
  </div>
  <div class="_2_QraFYR_0">是否需要把读取客户端 ip 的访问次数 GET(ip)，以及判断访问次数是否超过 20 的判断逻辑，也加到 Lua 脚本中？<br><br>我觉得需要分3步来优化：<br>1.在执行lua之前，get(ip)，进行判断，如果大于20，就直接报错了，这样也就减少了执行lua的开销；<br>2.在执行lua时，判断incr（ip）的返回值，这个值是加一之后的值，直接判断这个值是否大于20，返回错误。<br>3.如果并发量特别大的时候，可以在incr前再判断一次get（ip），减少incr的开销。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-14 00:42:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/83/4b/0e96fcae.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>sky</span>
  </div>
  <div class="_2_QraFYR_0">对于这些操作，我们同样需要保证它们的原子性。否则，如果客户端使用多线程访问，访问次数初始值为 0，第一个线程执行了 INCR(ip) 操作后，第二个线程紧接着也执行了 INCR(ip)，此时，ip 对应的访问次数就被增加到了 2，我们就无法再对这个 ip 设置过期时间了。这样就会导致，这个 ip 对应的客户端访问次数达到 20 次之后，就无法再进行访问了。即使过了 60s，也不能再继续访问，显然不符合业务要求。<br><br><br>不太明白为何过了 60 秒，也不能继续访问呢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-29 20:33:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTIY8df9yV6vQjOMxv5xym070hFT2GWYELpqBhxSicQoq5IqBx6teS1xJaomkOBeuzv4vkXRJfibqcMw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>永不止步</span>
  </div>
  <div class="_2_QraFYR_0">除了技术之外，我觉得那段代码的判断逻辑应该属于业务范畴，最好不要放lua中，因为业务是经常变化的，lua脚本最好与具体业务无关</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-18 18:03:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/56/e2/2dcab30d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>郑印</span>
  </div>
  <div class="_2_QraFYR_0">感觉 “Redis 在执行 Lua 脚本时，是可以保证原子性的”  这句话有误导，当lua脚本中间有命令执行出错时，以执行的命令还是生效了的，这样不能完全称之为原子性。redis只是通过lua脚本让一组命令想一个命令一样执行在加上单线程的特性避免了并发产生的问题，而不能称之为原子性。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-08 14:03:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/21/f6/76/f7666f76.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>huiye</span>
  </div>
  <div class="_2_QraFYR_0">除了把多个操作在 Redis 中实现成一个操作，和使用lua脚本，使用redis的事务，把多个命令放入队列里一起执行，是不是也能保证原子性呢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-02 16:04:03</div>
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
  <div class="_2_QraFYR_0">思考题<br>思考题：是否需要把读取客户端 ip 的访问次数，也就是 GET(ip)，以及判断访问次数是否超过 20 的判断逻辑，也加到 Lua 脚本中吗？<br>我觉得不需要将其添加到 Lua 脚本中，理由如下：<br>1.获取客户端 ip 的访问次数和判断访问次数是否超过 20 并没有进行 R-M-W 操作即这个代码没有操作临界区代码，所以不会有并发问题<br>2.因为如果把更多操作加到 Lua 脚本中，会导致 Redis 执行 Lua 脚本的时间增加，会降低 Redis 的并发性能，建议就是在编写 Lua 脚本中，要避免添加不需要做并发控制的代码<br><br>补充<br>上面的 Lua 脚本中有一个是判断次数是否为 1，按道理这个操作也是读操作，没有操作临界区的代码，为何要把它加到 Lua 脚本中呢？<br>这个是因为客户端 ip 的访问次数 +1 和 判断是否等于 1 ，和后面的设置过期时间，是一个最小的逻辑单元，这个判断会影响到后面过期时间的设，所以这个读操作要加入到 Lua 脚本中</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-29 17:15:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/fc/0a/0c4b9170.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>泠小墨</span>
  </div>
  <div class="_2_QraFYR_0">关于最后的问题，我觉得可以不判断访问次数，前提稍微修改下lua脚本，将current的值返回给客户端，这样客户端可以根据返回值进行处理；<br>local current <br>current = redis.call(&#39;incr&#39;,KEYS[1]) <br>if tonumber(current)==1 then redis.call(&#39;expire&#39;,KEYS[1],60) <br>end  <br>return current</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-26 14:08:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/44/d8/1a1761f9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>James_Shangguan</span>
  </div>
  <div class="_2_QraFYR_0">不需要加到Lua脚本里面。并发编程思想里面，降低锁的粒度，提高并发的性能。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-09 20:02:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_89e362</span>
  </div>
  <div class="_2_QraFYR_0">我感觉需要，比如当前 count 为19 ，但是并发上来三个请求，调用get获取的都是19，小于20，那么访问的次数可能就会超过20了。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-07 17:53:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/ZGbq9QcsysWhYJvIXaZ0S9llXDVicF0Tvm2ibgGehHCPYWxHzZ2Z34SVOyIOKkH44qPFCANBbib2iawqrWU7azD3og/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_750e24</span>
  </div>
  <div class="_2_QraFYR_0">  value = INCR(ip)    &#47;&#47;如果是第一次访问，将键值对的过期时间设置为60s后  <br>  IF value == 1 <br>  THEN       <br>  EXPIRE(ip,60)    <br>END<br>执行 INCR命令返回value不是原子操作么？就算两个线程执行了INCR，第一个线程返回的不是1么？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-28 16:12:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/6c/b5/32374f93.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>可怜大灰狼</span>
  </div>
  <div class="_2_QraFYR_0">我觉得get(ip)可以不需要，因为incr已经可以返回当前访问次数，current值大于20的时候，直接返回客户端加锁失败。但是判断是否超过20次逻辑，应该要加到lua脚本中，不然没法保证对控制次数的原子性</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-26 09:40:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/f8/bb/8b2ba45d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>冯传博</span>
  </div>
  <div class="_2_QraFYR_0">对于这些操作，我们同样需要保证它们的原子性。否则，如果客户端使用多线程访问，访问次数初始值为 0，第一个线程执行了 INCR(ip) 操作后，第二个线程紧接着也执行了 INCR(ip)，此时，ip 对应的访问次数就被增加到了 2，我们就无法再对这个 ip 设置过期时间了。这样就会导致，这个 ip 对应的客户端访问次数达到 20 次之后，就无法再进行访问了。即使过了 60s，也不能再继续访问，显然不符合业务要求。<br><br><br>如果第一个线程正常执行，是能够给ip设置过期时间的，也就能够满足业务。出现没有设置过期时间的情景，是线程一在设置过期时间之前退出了。<br><br>这段代码还有个问题是，在高并发的时候20次的访问限制可能会被击穿，也就是访问次数能够超过20次。<br><br>不知理解是否争取，请老师指教</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-26 08:41:06</div>
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
  <div class="_2_QraFYR_0">之前文章里不是有讲另外一种方式——MULTI和EXEC吗？这种是不是可以实现lua脚本一样的功能？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-27 17:46:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/7e/a6/188817b6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>郭嵩阳</span>
  </div>
  <div class="_2_QraFYR_0">想问下老师,你们在开发项目中是否，经常使用lua脚本.或者是否建议经常去使用lua脚本,个人觉得lua脚本维护不是很方便，相听一下老师的意见</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-27 11:05:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/b8/41/d33a7f53.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Dai Yan</span>
  </div>
  <div class="_2_QraFYR_0">感觉每个客户的点赞次数都是独立的，所以不同客户端的点击次数不会互相干扰，在某一个客户的session里，读是否放入lua，效果是一样的。因为incr只会针对自己的点击次数加一，别人不会来改变这个值。如果不是这个场景，多个客户端共享一个后台数据，那get（IP）就要放到lua里面，因为，可能不同的客户端incr会改变次数，这个跟当时读取的时候，已经不一样了，可能是+2或+3，那这样可能点击不到20次，就要停止了。但这个也不可能，因为点击次数是分到每个客户的。这个是我的疑问。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-27 11:58:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>beifengzhishen001</span>
  </div>
  <div class="_2_QraFYR_0">multi&#47;exec执行的组合语句，假设所有语句都没有发生错误，算不算原子操作呢？考虑到redis执行客户端请求都是并行操作，看到第31节的内容后，回过头来看本节有了这个疑问。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-12 16:38:01</div>
  </div>
</div>
</div>
</li>
</ul>