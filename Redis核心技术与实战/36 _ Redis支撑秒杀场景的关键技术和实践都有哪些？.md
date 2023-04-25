<audio title="36 _ Redis支撑秒杀场景的关键技术和实践都有哪些？" src="https://static001.geekbang.org/resource/audio/10/b3/107116ef311346a103cbf1c45dc6a4b3.mp3" controls="controls"></audio> 
<p>你好，我是蒋德钧。</p><p>秒杀是一个非常典型的活动场景，比如，在双11、618等电商促销活动中，都会有秒杀场景。秒杀场景的业务特点是<strong>限时限量</strong>，业务系统要处理瞬时的大量高并发请求，而Redis就经常被用来支撑秒杀活动。</p><p>不过，秒杀场景包含了多个环节，可以分成秒杀前、秒杀中和秒杀后三个阶段，每个阶段的请求处理需求并不相同，Redis并不能支撑秒杀场景的每一个环节。</p><p>那么，Redis具体是在秒杀场景的哪个环节起到支撑作用的呢？又是如何支持的呢？清楚了这个问题，我们才能知道在秒杀场景中，如何使用Redis来支撑高并发压力，并且做好秒杀场景的应对方案。</p><p>接下来，我们先来了解下秒杀场景的负载特征。</p><h2>秒杀场景的负载特征对支撑系统的要求</h2><p>秒杀活动售卖的商品通常价格非常优惠，会吸引大量用户进行抢购。但是，商品库存量却远远小于购买该商品的用户数，而且会限定用户只能在一定的时间段内购买。这就给秒杀系统带来两个明显的负载特征，相应的，也对支撑系统提出了要求，我们来分析下。</p><p><strong>第一个特征是瞬时并发访问量非常高</strong>。</p><p>一般数据库每秒只能支撑千级别的并发请求，而Redis的并发处理能力（每秒处理请求数）能达到万级别，甚至更高。所以，<strong>当有大量并发请求涌入秒杀系统时，我们就需要使用Redis先拦截大部分请求，避免大量请求直接发送给数据库，把数据库压垮</strong>。</p><!-- [[[read_end]]] --><p><strong>第二个特征是读多写少，而且读操作是简单的查询操作</strong>。</p><p>在秒杀场景下，用户需要先查验商品是否还有库存（也就是根据商品ID查询该商品的库存还有多少），只有库存有余量时，秒杀系统才能进行库存扣减和下单操作。</p><p>库存查验操作是典型的键值对查询，而Redis对键值对查询的高效支持，正好和这个操作的要求相匹配。</p><p>不过，秒杀活动中只有少部分用户能成功下单，所以，商品库存查询操作（读操作）要远多于库存扣减和下单操作（写操作）。</p><p>当然，实际秒杀场景通常有多个环节，刚才介绍的用户查验库存只是其中的一个环节。那么，Redis具体可以在整个秒杀场景中哪些环节发挥作用呢？这就要说到秒杀活动的整体流程了，我们来分析下。</p><h2>Redis可以在秒杀场景的哪些环节发挥作用？</h2><p>我们一般可以把秒杀活动分成三个阶段。在每一个阶段，Redis所发挥的作用也不一样。</p><p>第一阶段是秒杀活动前。</p><p>在这个阶段，用户会不断刷新商品详情页，这会导致详情页的瞬时请求量剧增。这个阶段的应对方案，一般是尽量<strong>把商品详情页的页面元素静态化，然后使用CDN或是浏览器把这些静态化的元素缓存起来</strong>。这样一来，秒杀前的大量请求可以直接由CDN或是浏览器缓存服务，不会到达服务器端了，这就减轻了服务器端的压力。</p><p>在这个阶段，有CDN和浏览器缓存服务请求就足够了，我们还不需要使用Redis。</p><p>第二阶段是秒杀活动开始。</p><p>此时，大量用户点击商品详情页上的秒杀按钮，会产生大量的并发请求查询库存。一旦某个请求查询到有库存，紧接着系统就会进行库存扣减。然后，系统会生成实际订单，并进行后续处理，例如订单支付和物流服务。如果请求查不到库存，就会返回。用户通常会继续点击秒杀按钮，继续查询库存。</p><p>简单来说，这个阶段的操作就是三个：库存查验、库存扣减和订单处理。因为每个秒杀请求都会查询库存，而请求只有查到有库存余量后，后续的库存扣减和订单处理才会被执行。所以，这个阶段中最大的并发压力都在库存查验操作上。</p><p>为了支撑大量高并发的库存查验请求，我们需要在这个环节使用Redis保存库存量，这样一来，请求可以直接从Redis中读取库存并进行查验。</p><p>那么，库存扣减和订单处理是否都可以交给后端的数据库来执行呢?</p><p>其实，订单处理可以在数据库中执行，但库存扣减操作，不能交给后端数据库处理。</p><p>在数据库中处理订单的原因比较简单，我先说下。</p><p>订单处理会涉及支付、商品出库、物流等多个关联操作，这些操作本身涉及数据库中的多张数据表，要保证处理的事务性，需要在数据库中完成。而且，订单处理时的请求压力已经不大了，数据库可以支撑这些订单处理请求。</p><p>那为啥库存扣减操作不能在数据库执行呢？这是因为，一旦请求查到有库存，就意味着发送该请求的用户获得了商品的购买资格，用户就会下单了。同时，商品的库存余量也需要减少一个。如果我们把库存扣减的操作放到数据库执行，会带来两个问题。</p><ol>
<li><strong>额外的开销</strong>。Redis中保存了库存量，而库存量的最新值又是数据库在维护，所以数据库更新后，还需要和Redis进行同步，这个过程增加了额外的操作逻辑，也带来了额外的开销。</li>
<li><strong>下单量超过实际库存量，出现超售</strong>。由于数据库的处理速度较慢，不能及时更新库存余量，这就会导致大量库存查验的请求读取到旧的库存值，并进行下单。此时，就会出现下单数量大于实际的库存量，导致出现超售，这就不符合业务层的要求了。</li>
</ol><p>所以，我们就需要直接在Redis中进行库存扣减。具体的操作是，当库存查验完成后，一旦库存有余量，我们就立即在Redis中扣减库存。而且，为了避免请求查询到旧的库存值，库存查验和库存扣减这两个操作需要保证原子性。</p><p>第三阶段就是秒杀活动结束后。</p><p>在这个阶段，可能还会有部分用户刷新商品详情页，尝试等待有其他用户退单。而已经成功下单的用户会刷新订单详情，跟踪订单的进展。不过，这个阶段中的用户请求量已经下降很多了，服务器端一般都能支撑，我们就不重点讨论了。</p><p>好了，我们先来总结下秒杀场景对Redis的需求。</p><p>秒杀场景分成秒杀前、秒杀中和秒杀后三个阶段。秒杀开始前后，高并发压力没有那么大，我们不需要使用Redis，但在秒杀进行中，需要查验和扣减商品库存，库存查验面临大量的高并发请求，而库存扣减又需要和库存查验一起执行，以保证原子性。这就是秒杀对Redis的需求。</p><p>下图显示了在秒杀场景中需要Redis参与的两个环节：</p><p><img src="https://static001.geekbang.org/resource/image/7c/1b/7c3e5def912d7c8c45bca00f955d751b.jpg?wh=2176*1582" alt=""></p><p>了解需求后，我们使用Redis来支撑秒杀场景的方法就比较清晰了。接下来，我向你介绍两种方法。</p><h2>Redis的哪些方法可以支撑秒杀场景？</h2><p>秒杀场景对Redis操作的根本要求有两个。</p><ol>
<li><strong>支持高并发</strong><strong>。</strong>这个很简单，Redis本身高速处理请求的特性就可以支持高并发。而且，如果有多个秒杀商品，我们也可以使用切片集群，用不同的实例保存不同商品的库存，这样就避免，使用单个实例导致所有的秒杀请求都集中在一个实例上的问题了。不过，需要注意的是，当使用切片集群时，我们要先用CRC算法计算不同秒杀商品key对应的Slot，然后，我们在分配Slot和实例对应关系时，才能把不同秒杀商品对应的Slot分配到不同实例上保存。</li>
<li><strong>保证库存查验和库存扣减原子性执行</strong>。针对这条要求，我们就可以使用Redis的原子操作或是分布式锁这两个功能特性来支撑了。</li>
</ol><p>我们先来看下Redis是如何基于原子操作来支撑秒杀场景的。</p><h3>基于原子操作支撑秒杀场景</h3><p>在秒杀场景中，一个商品的库存对应了两个信息，分别是总库存量和已秒杀量。这种数据模型正好是一个key（商品ID）对应了两个属性（总库存量和已秒杀量），所以，我们可以使用一个Hash类型的键值对来保存库存的这两个信息，如下所示：</p><pre><code>key: itemID
value: {total: N, ordered: M}
</code></pre><p>其中，itemID是商品的编号，total是总库存量，ordered是已秒杀量。</p><p>因为库存查验和库存扣减这两个操作要保证一起执行，<strong>一个直接的方法就是使用Redis的原子操作</strong>。</p><p>我们在<a href="https://time.geekbang.org/column/article/299806">第29讲</a>中学习过，原子操作可以是Redis自身提供的原子命令，也可以是Lua脚本。因为库存查验和库存扣减是两个操作，无法用一条命令来完成，所以，我们就需要使用Lua脚本原子性地执行这两个操作。</p><p>那怎么在Lua脚本中实现这两个操作呢？我给你提供一段Lua脚本写的伪代码，它显示了这两个操作的实现。</p><pre><code>#获取商品库存信息            
local counts = redis.call(&quot;HMGET&quot;, KEYS[1], &quot;total&quot;, &quot;ordered&quot;);
#将总库存转换为数值
local total = tonumber(counts[1])
#将已被秒杀的库存转换为数值
local ordered = tonumber(counts[2])  
#如果当前请求的库存量加上已被秒杀的库存量仍然小于总库存量，就可以更新库存         
if ordered + k &lt;= total then
    #更新已秒杀的库存量
    redis.call(&quot;HINCRBY&quot;,KEYS[1],&quot;ordered&quot;,k)                              return k;  
end               
return 0
</code></pre><p>有了Lua脚本后，我们就可以在Redis客户端，使用EVAL命令来执行这个脚本了。</p><p>最后，客户端会根据脚本的返回值，来确定秒杀是成功还是失败了。如果返回值是k，就是成功了；如果是0，就是失败。</p><p>到这里，我们学习了如何使用原子性的Lua脚本来实现库存查验和库存扣减。其实，要想保证库存查验和扣减这两个操作的原子性，我们还有另一种方法，就是<strong>使用分布式锁来保证多个客户端能互斥执行这两个操作</strong>。接下来，我们就来看下如何使用分布式锁来支撑秒杀场景。</p><h3>基于分布式锁来支撑秒杀场景</h3><p><strong>使用分布式锁来支撑秒杀场景的具体做法是，先让客户端向Redis申请分布式锁，只有拿到锁的客户端才能执行库存查验和库存扣减</strong>。这样一来，大量的秒杀请求就会在争夺分布式锁时被过滤掉。而且，库存查验和扣减也不用使用原子操作了，因为多个并发客户端只有一个客户端能够拿到锁，已经保证了客户端并发访问的互斥性。</p><p>你可以看下下面的伪代码，它显示了使用分布式锁来执行库存查验和扣减的过程。</p><pre><code>//使用商品ID作为key
key = itemID
//使用客户端唯一标识作为value
val = clientUniqueID
//申请分布式锁，Timeout是超时时间
lock =acquireLock(key, val, Timeout)
//当拿到锁后，才能进行库存查验和扣减
if(lock == True) {
   //库存查验和扣减
   availStock = DECR(key, k)
   //库存已经扣减完了，释放锁，返回秒杀失败
   if (availStock &lt; 0) {
      releaseLock(key, val)
      return error
   }
   //库存扣减成功，释放锁
   else{
     releaseLock(key, val)
     //订单处理
   }
}
//没有拿到锁，直接返回
else
   return
</code></pre><p>需要提醒你的是，在使用分布式锁时，客户端需要先向Redis请求锁，只有请求到了锁，才能进行库存查验等操作，这样一来，客户端在争抢分布式锁时，大部分秒杀请求本身就会因为抢不到锁而被拦截。</p><p>所以，我给你一个小建议，<strong>我们可以使用切片集群中的不同实例来分别保存分布式锁和商品库存信息</strong>。使用这种保存方式后，秒杀请求会首先访问保存分布式锁的实例。如果客户端没有拿到锁，这些客户端就不会查询商品库存，这就可以减轻保存库存信息的实例的压力了。</p><h2>小结</h2><p>这节课，我们学习了Redis在秒杀场景中的具体应用。秒杀场景有2个负载特征，分别是瞬时高并发请求和读多写少。Redis良好的高并发处理能力，以及高效的键值对读写特性，正好可以满足秒杀场景的需求。</p><p>在秒杀场景中，我们可以通过前端CDN和浏览器缓存拦截大量秒杀前的请求。在实际秒杀活动进行时，库存查验和库存扣减是承受巨大并发请求压力的两个操作，同时，这两个操作的执行需要保证原子性。Redis的原子操作、分布式锁这两个功能特性可以有效地来支撑秒杀场景的需求。</p><p>当然，对于秒杀场景来说，只用Redis是不够的。秒杀系统是一个系统性工程，Redis实现了对库存查验和扣减这个环节的支撑，除此之外，还有4个环节需要我们处理好。</p><ol>
<li><strong>前端静态页面的设计</strong>。秒杀页面上能静态化处理的页面元素，我们都要尽量静态化，这样可以充分利用CDN或浏览器缓存服务秒杀开始前的请求。</li>
<li><strong>请求拦截和流控</strong>。在秒杀系统的接入层，对恶意请求进行拦截，避免对系统的恶意攻击，例如使用黑名单禁止恶意IP进行访问。如果Redis实例的访问压力过大，为了避免实例崩溃，我们也需要在接入层进行限流，控制进入秒杀系统的请求数量。</li>
<li><strong>库存信息过期时间处理</strong>。Redis中保存的库存信息其实是数据库的缓存，为了避免缓存击穿问题，我们不要给库存信息设置过期时间。</li>
<li><strong>数据库订单异常处理</strong>。如果数据库没能成功处理订单，可以增加订单重试功能，保证订单最终能被成功处理。</li>
</ol><p>最后，我也再给你一个小建议：秒杀活动带来的请求流量巨大，我们需要把秒杀商品的库存信息用单独的实例保存，而不要和日常业务系统的数据保存在同一个实例上，这样可以避免干扰业务系统的正常运行。</p><h2>每课一问</h2><p>按照惯例，我给你提个小问题，假设一个商品的库存量是800，我们使用一个包含了4个实例的切片集群来服务秒杀请求。我们让每个实例各自维护库存量200，然后，客户端的秒杀请求可以分发到不同的实例上进行处理，你觉得这是一个好方法吗？</p><p>欢迎在留言区写下你的思考和答案，我们一起交流讨论。如果你觉得今天的内容对你有所帮助，也欢迎你分享给你的朋友或同事。我们下节课见。</p>
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
  <div class="_2_QraFYR_0">使用多个实例的切片集群来分担秒杀请求，是否是一个好方法？<br><br>使用切片集群分担秒杀请求，可以降低每个实例的请求压力，前提是秒杀请求可以平均打到每个实例上，否则会出现秒杀请求倾斜的情况，反而会增加某个实例的压力，而且会导致商品没有全部卖出的情况。<br><br>但用切片集群分别存储库存信息，缺点是如果需要向用户展示剩余库存，要分别查询多个切片，最后聚合结果后返回给客户端。这种情况下，建议不展示剩余库存信息，直接针对秒杀请求返回是否秒杀成功即可。<br><br>秒杀系统最重要的是，把大部分请求拦截在最前面，只让很少请求能够真正进入到后端系统，降低后端服务的压力，常见的方案包括：页面静态化（推送到CDN）、网关恶意请求拦截、请求分段放行、缓存校验和扣减库存、消息队列处理订单。<br><br>另外，为了不影响其他业务系统，秒杀系统最好和业务系统隔离，主要包括应用隔离、部署隔离、数据存储隔离。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-13 00:33:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/20/08/bc06bc69.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Dovelol</span>
  </div>
  <div class="_2_QraFYR_0">老师好，想问下，用redis扣库存直接用HINCRBY已秒杀库存量，判断返回值&gt;=总库存就是没库存，小于总库存就是扣减库存成功可以购买，这样判断有什么问题么？为什么要引入lua脚本来增加复杂度呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-14 13:02:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/78/c7/083a3a0b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>新世界</span>
  </div>
  <div class="_2_QraFYR_0">一般秒杀场景 用redis 扣减库存，不去保证原子性，扣减后有可能超卖，再用数据库去保证最终不超卖，因为超卖的不会多，能够打到mysql的操作就是超卖+实际库存，所以mysql压力也会比较少<br><br>课后习题：  redis的 800个库存，分布到4个server上，打到哪个server上，取决于key，如果key分布不均匀，会导致 一定的不公平，就像高考一样，有的地方考生多，有的地方考生少，虽然在每个省中录取名额一样，但是也不公平</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 高考的比喻很形象！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-07 16:10:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_f6a7c7</span>
  </div>
  <div class="_2_QraFYR_0">不是一个好方案，一个商品库存数切片不同实例存储的确可以减少单个实例的压力，但是在业务上可行性有待商榷，如果当前切片实例没有库存，是不是要再请求别的切片实例？特别是库存数都被抢购完成后，后面的请求是不是都要请求4个切片节点做库存数的聚合才知道又没人退货，做库存数的聚合？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 分析的不错。使用切片后，处理逻辑要考虑分布式的数据情况了，变得复杂了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-04 12:10:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_0879b1</span>
  </div>
  <div class="_2_QraFYR_0">老师，秒杀过程中，如果不论Redis采用RDB还是AOF的方式来持久化，都有可能导致库存数据的丢失。而如果不采用持久化的方式，数据丢失的风险更大。此时如果redis中的库存数据丢失，并且库存数据没有同步到关系型数据库，这种丢失的风险该如何应对？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-14 10:09:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/f0/9f/6689d26e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>花轮君</span>
  </div>
  <div class="_2_QraFYR_0">设计上是可行的，将秒杀请求也进行分片，库存的校验可以按照分库顺序扣。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 请求分片的均衡度会比较难控制，另外，如果按照分库顺序扣的话，那么设计四个分片期望达到的支持并发扣减库存的目标就达不到了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-13 09:43:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/d2/74/7861f504.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>马听</span>
  </div>
  <div class="_2_QraFYR_0">为啥库存扣减不能再数据库执行呢？<br>除了文中提到的额外开销和可能导致超售，另外如果数据库是 MySQL，则可能导致 MySQL 的死锁检测，导致消耗大量的 CPU 资源。具体可以参考《MySQL实战45 讲》07 节 ：行锁功过</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-02 18:06:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/25/7f/473d5a77.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>曾轼麟</span>
  </div>
  <div class="_2_QraFYR_0">我个人认为这个方式不是很好，因为单个sku分开后，需要对用户的请求做路由判断，假如一个用户请求本身就发出了两条，按照随机路由的方式，他有可能在两个切片中抢到商品。假如路由规则是固定的，那么会出现sku还有库存，但是用户就是抢不到的情况</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这里的关键技术挑战就是：请求分发规则和库存分布式查询。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-20 08:54:56</div>
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
  <div class="_2_QraFYR_0">老师问下，最后的订单处理是如何处理的，库存扣完成功操作，接着进行下单操作，下单操作是直接对数据库进行操作么？还是将行为打入kafka之类的消息队列，然后慢慢的去消费处理订单</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-15 21:31:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83eoSMRiaMtAcqQz7oNzcNg0M2vEic2sByibGib0l9z2g5Niafo7caLhqJtACRfCCT1sdAKQQM40BHEWsdOg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>王云星</span>
  </div>
  <div class="_2_QraFYR_0">使用多个实例的切片集群来分担秒杀请求，是否是一个好方法？<br>不太行，难度很大，第一个是流量需要均衡分配到每个实例，第二个是如果说一个订单需要扣减多个库存，而单个实例库存剩余不够，导致扣减库存失败，但其实总库存是够的。比如每个实例还剩1件库存，一个订单同时秒杀4件，这下就会扣减失败，实际上库存是够的</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-08 14:58:46</div>
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
  <div class="_2_QraFYR_0">如果秒杀请求能够比较均匀的分别打到4个实例上是没有问题的。<br>这个时候不返回剩余库存，不做聚合，能大幅度提高速度。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 关键是这个假设不一定能实际做到。。。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-03 14:22:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/KFgDEHIEpnTUibfcckj33D1LVj9VapfrK3Yq2Gj00wnLt4nkWS7HvYy5NxvmnQcQpaysuBHVrB9MILWZ9hibUNasicPNtueYoNM/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>JAVA初级开发工程师</span>
  </div>
  <div class="_2_QraFYR_0">关于秒杀场景有几个问题<br>1、redis并不保证数据的一致性 发生主备切换有可能会造成超卖这种情况怎么处理<br>2、秒杀如何处理大库存场景 如果一个商品库存对应redis一个key  这个key可能同时承受数十万的qps 这种情况redis也是扛不住的</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-20 13:56:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_f00f74</span>
  </div>
  <div class="_2_QraFYR_0">实际生产中能用Redis做库存校验和扣减吗？如果Redis宕机如何保证Redis库存和数据库库存一致？如果发生主从切换导致库存更新丢失呢？<br>使用分布式锁的话就是串行化了，这个性能太差了吧。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-13 17:17:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2d/35/d3/8de43dd5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Breeze</span>
  </div>
  <div class="_2_QraFYR_0">使用多个实例的切片集群来分担秒杀请求，是否是一个好方法?<br>如果并发量不大，或者提前做了限流策略，那么使用多个实例切片来分担压力就没有太大必要性了，主从集群就足够应对了。<br>如果并发量非常大，比如10w+请求，那么此时通过多实例来分担秒杀请求是合理的，因为并发量足够大，800库存理论上在极短的时间内会出现库存不均匀的现象，但是很快整个库存就都被消化掉，所以这个影响是可以忽略的。<br>所以整个方案取决秒杀的并发量和秒杀整个过程时长，如果并发量很大，秒杀过程很短（库存不大），那么这个方案是可行的。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-19 11:43:19</div>
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
  <div class="_2_QraFYR_0">客户端的秒杀请求可以分发到不同的实例上进行处理，你觉得这是一个好方法吗？<br><br>使用切片集群会将问题复杂化：<br>1.如果分片的处理能力不是均衡的，可能出现某个分片的商品在秒杀活动结束后未卖完。<br>2.前端查询总库存时，后端需要查多个分片的库存，然后进行合并，增加了网络交互。<br>3.当出现一个订单购买多件商品时，商品库存被分摊到多个分片，需要用锁占用分片资源，多个分片扣减库存成功后，才会释放分片资源，影响性能。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-20 22:19:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/77/a5/c5ae871d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>zenk</span>
  </div>
  <div class="_2_QraFYR_0">多个实例，需要客户段做负载均衡，这是额外的复杂度<br><br>查询<br><br>查询的时候需要查询每个结点，因此集群并没减少每个节点的请求量<br><br>扣库存<br><br>扣库存的时候，考虑到高并发的时候很容易给每个实例分配到200个并发请求，因此只有这些请求会分散在各个节点<br><br>如果不命中，需要请求其他每个节点，而这些请求量更大，因此整体而言扣库存也没有降低节点的压力<br><br>单机情况下，假设一次扣库存操作是10us（量级不确定），800次就是8ms，200次是2ms<br><br>可靠性<br>集群高，一个挂了其他节点还能处理<br><br>结论<br>集群在命中第一个节点的时候可缓解节点压力，但是增加客户端复杂度<br><br>可靠性，秒杀集中在很短时间，减少故障概率<br><br>扣库存差6ms可以接受<br><br>所以，集群好处不明显</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-03 09:59:08</div>
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
  <div class="_2_QraFYR_0">有秒杀场景，那老师，在 Redis 中，当一个 Key 的 QPS 达到 100万时，应该如何处理呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-13 21:14:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/cf/c2/fe1fb863.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ss柚子</span>
  </div>
  <div class="_2_QraFYR_0">这边采用redis支撑秒杀场景业务，那数据库和redis如何保证数据的一致性？这里没有涉及到什么时候去更新数据库中的数据</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-20 13:42:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/ca/07/22dd76bf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>kobe</span>
  </div>
  <div class="_2_QraFYR_0">那怎么在 Lua 脚本中实现这两个操作呢？我给你提供一段 Lua 脚本写的伪代码，它显示了这两个操作的实现。#获取商品库存信息            local counts = redis.call(&quot;HMGET&quot;, KEYS[1], &quot;total&quot;, &quot;ordered&quot;);#将总库存转换为数值local total = tonumber(counts[1])#将已被秒杀的库存转换为数值local ordered = tonumber(counts[2])  #如果当前请求的库存量加上已被秒杀的库存量仍然小于总库存量，就可以更新库存         if ordered + k &lt;= total then    #更新已秒杀的库存量    redis.call(&quot;HINCRBY&quot;,KEYS[1],&quot;ordered&quot;,k)                              return k;  end               return 0有了 Lua 脚本后，我们就可以在 Redis 客户端，使用 EVAL 命令来执行这个脚本了。最后，客户端会根据脚本的返回值，来确定秒杀是成功还是失败了。如果返回值是 k，就是成功了；如果是 0，就是失败。<br>这里hincrby命令的返回值应该是ordered的新值，而不是这个增量k吧</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-29 11:32:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/25/86/2b/7f9b94d8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Seajunnn</span>
  </div>
  <div class="_2_QraFYR_0">基于Redis原子操作和 基于分布式锁 这两种方案 各有什么优劣呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-05 13:59:35</div>
  </div>
</div>
</div>
</li>
</ul>