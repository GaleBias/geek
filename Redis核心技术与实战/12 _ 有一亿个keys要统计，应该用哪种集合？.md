<audio title="12 _ 有一亿个keys要统计，应该用哪种集合？" src="https://static001.geekbang.org/resource/audio/69/92/697b9d7ce3152b5636450e5a571e9c92.mp3" controls="controls"></audio> 
<p>你好，我是蒋德钧。</p><p>在Web和移动应用的业务场景中，我们经常需要保存这样一种信息：一个key对应了一个数据集合。我举几个例子。</p><ul>
<li>手机App中的每天的用户登录信息：一天对应一系列用户ID或移动设备ID；</li>
<li>电商网站上商品的用户评论列表：一个商品对应了一系列的评论；</li>
<li>用户在手机App上的签到打卡信息：一天对应一系列用户的签到记录；</li>
<li>应用网站上的网页访问信息：一个网页对应一系列的访问点击。</li>
</ul><p>我们知道，Redis集合类型的特点就是一个键对应一系列的数据，所以非常适合用来存取这些数据。但是，在这些场景中，除了记录信息，我们往往还需要对集合中的数据进行统计，例如：</p><ul>
<li>在移动应用中，需要统计每天的新增用户数和第二天的留存用户数；</li>
<li>在电商网站的商品评论中，需要统计评论列表中的最新评论；</li>
<li>在签到打卡中，需要统计一个月内连续打卡的用户数；</li>
<li>在网页访问记录中，需要统计独立访客（Unique Visitor，UV）量。</li>
</ul><p>通常情况下，我们面临的用户数量以及访问量都是巨大的，比如百万、千万级别的用户数量，或者千万级别、甚至亿级别的访问信息。所以，我们必须要选择能够非常高效地统计大量数据（例如亿级）的集合类型。</p><p><strong>要想选择合适的集合，我们就得了解常用的集合统计模式。</strong>这节课，我就给你介绍集合类型常见的四种统计模式，包括聚合统计、排序统计、二值状态统计和基数统计。我会以刚刚提到的这四个场景为例，和你聊聊在这些统计模式下，什么集合类型能够更快速地完成统计，而且还节省内存空间。掌握了今天的内容，之后再遇到集合元素统计问题时，你就能很快地选出合适的集合类型了。</p><!-- [[[read_end]]] --><h2>聚合统计</h2><p>我们先来看集合元素统计的第一个场景：聚合统计。</p><p>所谓的聚合统计，就是指统计多个集合元素的聚合结果，包括：统计多个集合的共有元素（交集统计）；把两个集合相比，统计其中一个集合独有的元素（差集统计）；统计多个集合的所有元素（并集统计）。</p><p>在刚才提到的场景中，统计手机App每天的新增用户数和第二天的留存用户数，正好对应了聚合统计。</p><p>要完成这个统计任务，我们可以用一个集合记录所有登录过App的用户ID，同时，用另一个集合记录每一天登录过App的用户ID。然后，再对这两个集合做聚合统计。我们来看下具体的操作。</p><p>记录所有登录过App的用户ID还是比较简单的，我们可以直接使用Set类型，把key设置为user:id，表示记录的是用户ID，value就是一个Set集合，里面是所有登录过App的用户ID，我们可以把这个Set叫作累计用户Set，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/99/ca/990e56babf199d9a7fa4c7343167ecca.jpg?wh=1774*1286" alt=""></p><p>需要注意的是，累计用户Set中没有日期信息，我们是不能直接统计每天的新增用户的。所以，我们还需要把每一天登录的用户ID，记录到一个新集合中，我们把这个集合叫作每日用户Set，它有两个特点：</p><ol>
<li>key是 user:id 以及当天日期，例如 user:id:20200803；</li>
<li>value是Set集合，记录当天登录的用户ID。</li>
</ol><p><img src="https://static001.geekbang.org/resource/image/a6/9e/a63dd95d5e44bf538fe960e67761b59e.jpg?wh=1931*1327" alt=""></p><p>在统计每天的新增用户时，我们只用计算每日用户Set和累计用户Set的差集就行。</p><p>我借助一个具体的例子来解释一下。</p><p>假设我们的手机App在2020年8月3日上线，那么，8月3日前是没有用户的。此时，累计用户Set是空集，当天登录的用户ID会被记录到 key为user:id:20200803的Set中。所以，user:id:20200803这个Set中的用户就是当天的新增用户。</p><p>然后，我们计算累计用户Set和user:id:20200803  Set的并集结果，结果保存在user:id这个累计用户Set中，如下所示：</p><pre><code>SUNIONSTORE  user:id  user:id  user:id:20200803 
</code></pre><p>此时，user:id这个累计用户Set中就有了8月3日的用户ID。等到8月4日再统计时，我们把8月4日登录的用户ID记录到user:id:20200804 的Set中。接下来，我们执行SDIFFSTORE命令计算累计用户Set和user:id:20200804 Set的差集，结果保存在key为user:new的Set中，如下所示：</p><pre><code>SDIFFSTORE  user:new  user:id:20200804 user:id  
</code></pre><p>可以看到，这个差集中的用户ID在user:id:20200804 的Set中存在，但是不在累计用户Set中。所以，user:new这个Set中记录的就是8月4日的新增用户。</p><p>当要计算8月4日的留存用户时，我们只需要再计算user:id:20200803 和 user:id:20200804两个Set的交集，就可以得到同时在这两个集合中的用户ID了，这些就是在8月3日登录，并且在8月4日留存的用户。执行的命令如下：</p><pre><code>SINTERSTORE user:id:rem user:id:20200803 user:id:20200804
</code></pre><p>当你需要对多个集合进行聚合计算时，Set类型会是一个非常不错的选择。不过，我要提醒你一下，这里有一个潜在的风险。</p><p>Set的差集、并集和交集的计算复杂度较高，在数据量较大的情况下，如果直接执行这些计算，会导致Redis实例阻塞。所以，我给你分享一个小建议：<strong>你可以从主从集群中选择一个从库，让它专门负责聚合计算，或者是把数据读取到客户端，在客户端来完成聚合统计</strong>，这样就可以规避阻塞主库实例和其他从库实例的风险了。</p><h2>排序统计</h2><p>接下来，我们再来聊一聊应对集合元素排序需求的方法。我以在电商网站上提供最新评论列表的场景为例，进行讲解。</p><p>最新评论列表包含了所有评论中的最新留言，<strong>这就要求集合类型能对元素保序</strong>，也就是说，集合中的元素可以按序排列，这种对元素保序的集合类型叫作有序集合。</p><p>在Redis常用的4个集合类型中（List、Hash、Set、Sorted Set），List和Sorted Set就属于有序集合。</p><p><strong>List是按照元素进入List的顺序进行排序的，而Sorted Set可以根据元素的权重来排序</strong>，我们可以自己来决定每个元素的权重值。比如说，我们可以根据元素插入Sorted Set的时间确定权重值，先插入的元素权重小，后插入的元素权重大。</p><p>看起来好像都可以满足需求，我们该怎么选择呢？</p><p>我先说说用List的情况。每个商品对应一个List，这个List包含了对这个商品的所有评论，而且会按照评论时间保存这些评论，每来一个新评论，就用LPUSH命令把它插入List的队头。</p><p>在只有一页评论的时候，我们可以很清晰地看到最新的评论，但是，在实际应用中，网站一般会分页显示最新的评论列表，一旦涉及到分页操作，List就可能会出现问题了。</p><p>假设当前的评论List是{A, B, C, D, E, F}（其中，A是最新的评论，以此类推，F是最早的评论），在展示第一页的3个评论时，我们可以用下面的命令，得到最新的三条评论A、B、C：</p><pre><code>LRANGE product1 0 2
1) &quot;A&quot;
2) &quot;B&quot;
3) &quot;C&quot;
</code></pre><p>然后，再用下面的命令获取第二页的3个评论，也就是D、E、F。</p><pre><code>LRANGE product1 3 5
1) &quot;D&quot;
2) &quot;E&quot;
3) &quot;F&quot;
</code></pre><p>但是，如果在展示第二页前，又产生了一个新评论G，评论G就会被LPUSH命令插入到评论List的队头，评论List就变成了{G, A, B, C, D, E, F}。此时，再用刚才的命令获取第二页评论时，就会发现，评论C又被展示出来了，也就是C、D、E。</p><pre><code>LRANGE product1 3 5
1) &quot;C&quot;
2) &quot;D&quot;
3) &quot;E&quot;
</code></pre><p>之所以会这样，关键原因就在于，List是通过元素在List中的位置来排序的，当有一个新元素插入时，原先的元素在List中的位置都后移了一位，比如说原来在第1位的元素现在排在了第2位。所以，对比新元素插入前后，List相同位置上的元素就会发生变化，用LRANGE读取时，就会读到旧元素。</p><p>和List相比，Sorted Set就不存在这个问题，因为它是根据元素的实际权重来排序和获取数据的。</p><p>我们可以按评论时间的先后给每条评论设置一个权重值，然后再把评论保存到Sorted Set中。Sorted Set的ZRANGEBYSCORE命令就可以按权重排序后返回元素。这样的话，即使集合中的元素频繁更新，Sorted Set也能通过ZRANGEBYSCORE命令准确地获取到按序排列的数据。</p><p>假设越新的评论权重越大，目前最新评论的权重是N，我们执行下面的命令时，就可以获得最新的10条评论：</p><pre><code>ZRANGEBYSCORE comments N-9 N
</code></pre><p>所以，在面对需要展示最新列表、排行榜等场景时，如果数据更新频繁或者需要分页显示，建议你优先考虑使用Sorted Set。</p><h2>二值状态统计</h2><p>现在，我们再来分析下第三个场景：二值状态统计。这里的二值状态就是指集合元素的取值就只有0和1两种。在签到打卡的场景中，我们只用记录签到（1）或未签到（0），所以它就是非常典型的二值状态，</p><p>在签到统计时，每个用户一天的签到用1个bit位就能表示，一个月（假设是31天）的签到情况用31个bit位就可以，而一年的签到也只需要用365个bit位，根本不用太复杂的集合类型。这个时候，我们就可以选择Bitmap。这是Redis提供的扩展数据类型。我来给你解释一下它的实现原理。</p><p>Bitmap本身是用String类型作为底层数据结构实现的一种统计二值状态的数据类型。String类型是会保存为二进制的字节数组，所以，Redis就把字节数组的每个bit位利用起来，用来表示一个元素的二值状态。你可以把Bitmap看作是一个bit数组。</p><p>Bitmap提供了GETBIT/SETBIT操作，使用一个偏移值offset对bit数组的某一个bit位进行读和写。不过，需要注意的是，Bitmap的偏移量是从0开始算的，也就是说offset的最小值是0。当使用SETBIT对一个bit位进行写操作时，这个bit位会被设置为1。Bitmap还提供了BITCOUNT操作，用来统计这个bit数组中所有“1”的个数。</p><p>那么，具体该怎么用Bitmap进行签到统计呢？我还是借助一个具体的例子来说明。</p><p>假设我们要统计ID 3000的用户在2020年8月份的签到情况，就可以按照下面的步骤进行操作。</p><p>第一步，执行下面的命令，记录该用户8月3号已签到。</p><pre><code>SETBIT uid:sign:3000:202008 2 1 
</code></pre><p>第二步，检查该用户8月3日是否签到。</p><pre><code>GETBIT uid:sign:3000:202008 2 
</code></pre><p>第三步，统计该用户在8月份的签到次数。</p><pre><code>BITCOUNT uid:sign:3000:202008
</code></pre><p>这样，我们就知道该用户在8月份的签到情况了，是不是很简单呢？接下来，你可以再思考一个问题：如果记录了1亿个用户10天的签到情况，你有办法统计出这10天连续签到的用户总数吗？</p><p>在介绍具体的方法之前，我们要先知道，Bitmap支持用BITOP命令对多个Bitmap按位做“与”“或”“异或”的操作，操作的结果会保存到一个新的Bitmap中。</p><p>我以按位“与”操作为例来具体解释一下。从下图中，可以看到，三个Bitmap bm1、bm2和bm3，对应bit位做“与”操作，结果保存到了一个新的Bitmap中（示例中，这个结果Bitmap的key被设为“resmap”）。</p><p><img src="https://static001.geekbang.org/resource/image/41/7a/4151af42513cf5f7996fe86c6064f97a.jpg?wh=2923*1016" alt=""></p><p>回到刚刚的问题，在统计1亿个用户连续10天的签到情况时，你可以把每天的日期作为key，每个key对应一个1亿位的Bitmap，每一个bit对应一个用户当天的签到情况。</p><p>接下来，我们对10个Bitmap做“与”操作，得到的结果也是一个Bitmap。在这个Bitmap中，只有10天都签到的用户对应的bit位上的值才会是1。最后，我们可以用BITCOUNT统计下Bitmap中的1的个数，这就是连续签到10天的用户总数了。</p><p>现在，我们可以计算一下记录了10天签到情况后的内存开销。每天使用1个1亿位的Bitmap，大约占12MB的内存（10^8/8/1024/1024），10天的Bitmap的内存开销约为120MB，内存压力不算太大。不过，在实际应用时，最好对Bitmap设置过期时间，让Redis自动删除不再需要的签到记录，以节省内存开销。</p><p>所以，如果只需要统计数据的二值状态，例如商品有没有、用户在不在等，就可以使用Bitmap，因为它只用一个bit位就能表示0或1。在记录海量数据时，Bitmap能够有效地节省内存空间。</p><h2>基数统计</h2><p>最后，我们再来看一个统计场景：基数统计。基数统计就是指统计一个集合中不重复的元素个数。对应到我们刚才介绍的场景中，就是统计网页的UV。</p><p>网页UV的统计有个独特的地方，就是需要去重，一个用户一天内的多次访问只能算作一次。在Redis的集合类型中，Set类型默认支持去重，所以看到有去重需求时，我们可能第一时间就会想到用Set类型。</p><p>我们来结合一个例子看一看用Set的情况。</p><p>有一个用户user1访问page1时，你把这个信息加到Set中：</p><pre><code>SADD page1:uv user1
</code></pre><p>用户1再来访问时，Set的去重功能就保证了不会重复记录用户1的访问次数，这样，用户1就算是一个独立访客。当你需要统计UV时，可以直接用SCARD命令，这个命令会返回一个集合中的元素个数。</p><p>但是，如果page1非常火爆，UV达到了千万，这个时候，一个Set就要记录千万个用户ID。对于一个搞大促的电商网站而言，这样的页面可能有成千上万个，如果每个页面都用这样的一个Set，就会消耗很大的内存空间。</p><p>当然，你也可以用Hash类型记录UV。</p><p>例如，你可以把用户ID作为Hash集合的key，当用户访问页面时，就用HSET命令（用于设置Hash集合元素的值），对这个用户ID记录一个值“1”，表示一个独立访客，用户1访问page1后，我们就记录为1个独立访客，如下所示：</p><pre><code>HSET page1:uv user1 1
</code></pre><p>即使用户1多次访问页面，重复执行这个HSET命令，也只会把user1的值设置为1，仍然只记为1个独立访客。当要统计UV时，我们可以用HLEN命令统计Hash集合中的所有元素个数。</p><p>但是，和Set类型相似，当页面很多时，Hash类型也会消耗很大的内存空间。那么，有什么办法既能完成统计，还能节省内存吗？</p><p>这时候，就要用到Redis提供的HyperLogLog了。</p><p>HyperLogLog是一种用于统计基数的数据集合类型，它的最大优势就在于，当集合元素数量非常多时，它计算基数所需的空间总是固定的，而且还很小。</p><p>在Redis中，每个 HyperLogLog只需要花费 12 KB 内存，就可以计算接近 2^64 个元素的基数。你看，和元素越多就越耗费内存的Set和Hash类型相比，HyperLogLog就非常节省空间。</p><p>在统计UV时，你可以用PFADD命令（用于向HyperLogLog中添加新元素）把访问页面的每个用户都添加到HyperLogLog中。</p><pre><code>PFADD page1:uv user1 user2 user3 user4 user5
</code></pre><p>接下来，就可以用PFCOUNT命令直接获得page1的UV值了，这个命令的作用就是返回HyperLogLog的统计结果。</p><pre><code>PFCOUNT page1:uv
</code></pre><p>关于HyperLogLog的具体实现原理，你不需要重点掌握，不会影响到你的日常使用，我就不多讲了。如果你想了解一下，课下可以看看<a href="http://en.wikipedia.org/wiki/HyperLogLog">这条链接</a>。</p><p>不过，有一点需要你注意一下，HyperLogLog的统计规则是基于概率完成的，所以它给出的统计结果是有一定误差的，标准误算率是0.81%。这也就意味着，你使用HyperLogLog统计的UV是100万，但实际的UV可能是101万。虽然误差率不算大，但是，如果你需要精确统计结果的话，最好还是继续用Set或Hash类型。</p><h2>小结</h2><p>这节课，我们结合统计新增用户数和留存用户数、最新评论列表、用户签到数以及网页独立访客量这4种典型场景，学习了集合类型的4种统计模式，分别是聚合统计、排序统计、二值状态统计和基数统计。为了方便你掌握，我把Set、Sorted Set、Hash、List、Bitmap、HyperLogLog的支持情况和优缺点汇总在了下面的表格里，希望你把这张表格保存下来，时不时地复习一下。</p><p><img src="https://static001.geekbang.org/resource/image/c0/6e/c0bb35d0d91a62ef4ca1bd939a9b136e.jpg?wh=2866*1739" alt=""></p><p>可以看到，Set和Sorted Set都支持多种聚合统计，不过，对于差集计算来说，只有Set支持。Bitmap也能做多个Bitmap间的聚合计算，包括与、或和异或操作。</p><p>当需要进行排序统计时，List中的元素虽然有序，但是一旦有新元素插入，原来的元素在List中的位置就会移动，那么，按位置读取的排序结果可能就不准确了。而Sorted Set本身是按照集合元素的权重排序，可以准确地按序获取结果，所以建议你优先使用它。</p><p>如果我们记录的数据只有0和1两个值的状态，Bitmap会是一个很好的选择，这主要归功于Bitmap对于一个数据只用1个bit记录，可以节省内存。</p><p>对于基数统计来说，如果集合元素量达到亿级别而且不需要精确统计时，我建议你使用HyperLogLog。</p><p>当然，Redis的应用场景非常多，这张表中的总结不一定能覆盖到所有场景。我建议你也试着自己画一张表，把你遇到的其他场景添加进去。长久积累下来，你一定能够更加灵活地把集合类型应用到合适的实践项目中。</p><h2>每课一问</h2><p>依照惯例，我给你留个小问题。这节课，我们学习了4种典型的统计模式，以及各种集合类型的支持情况和优缺点，我想请你聊一聊，你还遇到过其他的统计场景吗？用的是怎样的集合类型呢？</p><p>欢迎你在留言区写下你的思考和答案，和我交流讨论。如果你身边还有需要解决这些统计问题的朋友或同事，也欢迎你把今天的内容分享给他/她，我们下节课见。</p>
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
  <div class="_2_QraFYR_0">使用Sorted Set可以实现统计一段时间内的在线用户数：用户上线时使用zadd online_users $timestamp $user_id把用户添加到Sorted Set中，使用zcount online_users $start_timestamp $end_timestamp就可以得出指定时间段内的在线用户数。<br><br>如果key是以天划分的，还可以执行zinterstore online_users_tmp 2 online_users_{date1} online_users_{date2} aggregate max，把结果存储到online_users_tmp中，然后通过zrange online_users_tmp 0 -1 withscores就可以得到这2天都在线过的用户，并且score就是这些用户最近一次的上线时间。<br><br>还有一个有意思的方式，使用Set记录数据，再使用zunionstore命令求并集。例如sadd user1 apple orange banana、sadd user2 apple banana peach记录2个用户喜欢的水果，使用zunionstore fruits_union 2 user1 user2把结果存储到fruits_union这个key中，zrange fruits_union 0 -1 withscores可以得出每种水果被喜欢的次数。<br><br>使用HyperLogLog计算UV时，补充一点，还可以使用pfcount page1:uv page2:uv page3:uv或pfmerge page_union:uv page1:uv page2:uv page3:uv得出3个页面的UV总和。<br><br>另外，需要指出老师文章描述不严谨的地方：“Set数据类型，使用SUNIONSTORE、SDIFFSTORE、SINTERSTORE做并集、差集、交集时，选择一个从库进行聚合计算”。这3个命令都会在Redis中生成一个新key，而从库默认是readonly不可写的，所以这些命令只能在主库使用。想在从库上操作，可以使用SUNION、SDIFF、SINTER，这些命令可以计算出结果，但不会生成新key。<br><br>最后需要提醒一下：<br><br>1、如果是在集群模式使用多个key聚合计算的命令，一定要注意，因为这些key可能分布在不同的实例上，多个实例之间是无法做聚合运算的，这样操作可能会直接报错或者得到的结果是错误的！<br><br>2、当数据量非常大时，使用这些统计命令，因为复杂度较高，可能会有阻塞Redis的风险，建议把这些统计数据与在线业务数据拆分开，实例单独部署，防止在做统计操作时影响到在线业务。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-02 02:51:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/fd/fd/326be9bb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>注定非凡</span>
  </div>
  <div class="_2_QraFYR_0">1，作者讲了什么？<br>    1，Redis有那些数据结构适合做统计<br><br>2，作者是怎么把这事给讲明白的？<br>    1，列举了常见的数据统计需求。从实际需求出发，推荐适合的数据类型，讲解了怎么用，并解答这种数据结构为什么可以<br>    2，将数据统计需求，分了四类，分类分别讲解<br><br>3，为了讲明白，作者讲了哪些要点，有哪些亮点？<br>    1，亮点1：BITMAP的特性和使用场景，方式<br>    2，亮点2：HyperLogLog的特性和使用场景，方式<br>    3，要点1：日常的统计需求可以分为四类：聚合，排序，二值状态，基数，选用适合的数据类型可以实现即快速又节省内存<br>    4，要点2：聚合统计，可以选用Set类型完成，但Set的差，并，交集操作复杂度高，在数据量大的时候会阻塞主进程<br>    5，要点3：排序统计，可以选用List和Sorted Set<br>    6，要点4：二值状态统计：Bitmap本身是用String类型作为底层数据结构实现，String类型会保存为二进制字节数组，所以可以看作是一个bit数组<br>    7，要点5：基数统计：HyperLogLog ,计算基数所需空间总是固定的，而且很小。但要注意，HyperLogLog是统计规则是基于概率完成的，不是非常准确<br><br>4，对于作者所讲，我有那些发散性思考？<br>    1，对于统计用户的打卡情况，我们项目组也做了这个需求，但遗憾的是我们没有采用bitmap这种方案，而是使用了 sortSet<br>    2，HyperLogLog可以考虑使用到，我们项目中的统计视频播放次数，现在这块，我们的方案是，每天产生一个key，单调递增。在通过定时任务，将缓存中的结果，每天一条数据记录，存入数据库<br><br>5，在将来的那些场景中，我能够使用它？<br><br>6，留言区的收获<br><br>1，主从库模式使用Set数据类型聚合命令(来自 @kaito 大神)<br>    ①：使用SUNIONSTORE，SDIFFSTORE，SINTERSTOR做并集，差集，交集时，这三个命令都会在Redis中生成一个新key,而从库默认是readOnly。所以这些命令只能在主库上使用<br>    ②：SUNION，SDIFF,SINTER，这些命令可以计算出结果，不产生新的key可以在从库使用<br>   </div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-14 09:07:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/fc/d4/743d3f02.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Anthony</span>
  </div>
  <div class="_2_QraFYR_0">感觉第一个聚合统计这种场景一般频率不会太高，一般都是用在运营统计上，可以直接在mysql的从库上去统计，而不需要在redis上维护复杂的数据结构</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-02 09:34:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/20/47/d5/24e60497.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>波哥威武</span>
  </div>
  <div class="_2_QraFYR_0">现在大数据情况下都是通过实时流方式统计pvuv，不太会基于redis，基于存在即合理，老师能分析下相关优劣吗，我个人的想法，一个是在大量pvuv对redis的后端读写压力，还有复杂的统计结果redis也需要复杂的数据结构设计去实现，最后是业务和分析任务解耦。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-02 23:17:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/ea/51/9132e9cc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>土豆哪里挖</span>
  </div>
  <div class="_2_QraFYR_0">在集群的情况下，聚合统计就没法用了吧，毕竟不是同一个实例了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-02 17:09:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_960d5b</span>
  </div>
  <div class="_2_QraFYR_0">老师只是提供了一种使用思路，<br>但做统计业界主流还是上数仓用hive等做报表</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-02 14:18:17</div>
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
  <div class="_2_QraFYR_0">1.redis里不建议用聚合统计。原因有几点:<br>单实例会阻塞。cluster的时候key可能分布在不同的节点，需要调用方做聚合。<br>2.带排序的统计可以使用sorted set。cluster的时候可能一样需要做聚合<br>3.hyperlog是带误差的统计，可以用来统计总量。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-20 18:14:35</div>
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
  <div class="_2_QraFYR_0">对于这个问题：假设越新的评论权重越大，目前最新评论的权重是 N，我们执行下面的命令时，就可以获得最新的 10 条评论。<br><br>理解如下：<br><br>假设当前的评论 List 是{A, B, C, D, E, F}（其中，A 是最新的评论，以此类推，F 是最早的评论，权重分别为 10，9，8，7，6，5）。<br><br>在展示第一页的 3 个评论时，按照权重排序，查出 ABC。<br><br>展示第二页的 3 个评论时，按照权重排序，查出 DEF。<br><br>如果在展示第二页前，又产生了一个新评论 G，权重为 11，排序为 {G, A, B, C, D, E, F}。<br><br>再次查询第二页数据时，权重还是会以 10 为准，逻辑上，第一页的权重还是 10，9，8。<br><br>查询第二页数据时，可以查询出权重等于 7，6，5 的数据，返回评论 DEF。<br><br>当想查询出最新评论时，需要以权重 11 为准，第一页数据的权重就是 11，10，9，返回评论 GAB。<br><br>再次查询第二页数据时，以权重 11 为准，查询出评论 CDE。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-18 14:41:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/e8/e4/c9dd6058.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>阿基米德</span>
  </div>
  <div class="_2_QraFYR_0">这里一亿个数据返回给客户端处理，这个场景是不是就会有大key问题</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-14 01:00:54</div>
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
  <div class="_2_QraFYR_0"><br>老师说的大部分场景都没用到过。。。。。<br>我们有这么一种场景：<br>	在多实例下，定时任务就不能使用@Schedule使用，必须使用分布式定时调度，我们自研的分布式调度系统支持MQ和Http两种模式，同时支持一次性的调用和Cron表达是式形式的多次调用。<br>	在MQ模式下（暂时不支持Cron的调用），分布式调度系统作为MQ的消费者消费需要调度的任务，同时消息中会有所使用的资源，调度系统有对应的资源上线，也可以做资源限制，没有可用资源时，消息不调度（不投递）等待之前任务资源的释放，不投递时消息就在Zset中保存着，当然不同的类型在不同的Zset中，当有对用的资源类型释放后，会有专门的MQ确认消息，告诉任务调度系统，某种类型的资源已经释放，然后从对应type的Zset中获取排队中优先级最高的消息，进行资源匹配，如果可以匹配，则进行消息发送。<br>	当然http也是类似的，只是http不做资源管理，业务方自己掌控资源及调用频次，http请求的调用时调度系统自己发起的，引入quartz，在时间到达后，通过Http发送调用。<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-02 10:44:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/1b/93/e3b44969.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>sgl</span>
  </div>
  <div class="_2_QraFYR_0">12MB的bitmap是大key了，生产环境会有问题等我</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-02 09:10:19</div>
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
  <div class="_2_QraFYR_0">没懂， 文中说到的list由于插入的新的评论，第二页可能会读到原第一页的值， 我觉得这个本来不应该就是这样的吗？ 应该因为最新评论已经刷新了啊， 难道还要回去读原来的老数据吗 这块没太看懂</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-07 17:59:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/fe/83/df562574.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>慎独明强</span>
  </div>
  <div class="_2_QraFYR_0">有个疑问，统计亿级用户连续10天登录的场景，每天用一个bitmap的key，来存储每个用户的登录情况，将10个bitmap的key进行与运算来统计连续10天登录的用户，这个是怎么保证10个bitmap相同位是同一个用户的登录情况呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-02 08:07:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/5a/34/4cbadca6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>吃饭睡觉打酱油</span>
  </div>
  <div class="_2_QraFYR_0">老师，我对bitmap统计1亿用户的有个疑问，缓存中的bitmap是怎么初始化或者怎么来的呢，怎么保证用户的顺序呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-20 15:06:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/ef/21/69c181b8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Rain</span>
  </div>
  <div class="_2_QraFYR_0">redis真是应用开发利器啊</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-16 20:02:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/00/69/3b1375ca.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>海拉鲁</span>
  </div>
  <div class="_2_QraFYR_0">之前做过利用redis一个统计最近200个客户触达率的方案，借助list+lua<br>具体是用0代表触达，1代表未触达，不断丢入队列中。需要统计是lrang key 0 -1 取出全部元素，计算0的比例就是触达率了。<br>这样不需要每次都计算一次触达率，而是按需提供，也能保证最新。应该不是很有共性的需求，是我们对用户特定需求的一个尝试</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-02 07:06:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/0e/6c/1f3b1372.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>哈哈哈</span>
  </div>
  <div class="_2_QraFYR_0">个人感觉redis不太适合做聚合统计，这种统计是通过埋点+大数据平台</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-31 14:38:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/cc/de/e28c01e1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>剑八</span>
  </div>
  <div class="_2_QraFYR_0">redis还是适合缓存提速场景<br>像评论这样的，要实际看业务，是有一定业务逻辑的。比如评论还有几星，图片什么的，这种用redis就比较被动了。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-20 23:51:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/6c/a4/7f7c1955.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>死磕郎一世</span>
  </div>
  <div class="_2_QraFYR_0">list新增元素如果插入到尾部，这样，前面的元素位置就不用改变了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-21 00:07:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/3e/8a/891b0e58.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>wnz27</span>
  </div>
  <div class="_2_QraFYR_0">有个疑问，每个用户签到是横向情况，也就是一亿个十位bit，怎么转变为竖向的10个一亿位bit</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-07 13:09:07</div>
  </div>
</div>
</div>
</li>
</ul>