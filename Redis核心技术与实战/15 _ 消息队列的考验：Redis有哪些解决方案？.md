<audio title="15 _ 消息队列的考验：Redis有哪些解决方案？" src="https://static001.geekbang.org/resource/audio/ce/8c/ce703b9yy58ff12b214e59624070c68c.mp3" controls="controls"></audio> 
<p>你好，我是蒋德钧。</p><p>现在的互联网应用基本上都是采用分布式系统架构进行设计的，而很多分布式系统必备的一个基础软件就是消息队列。</p><p>消息队列要能支持组件通信消息的快速读写，而Redis本身支持数据的高速访问，正好可以满足消息队列的读写性能需求。不过，除了性能，消息队列还有其他的要求，所以，很多人都很关心一个问题：“Redis适合做消息队列吗？”</p><p>其实，这个问题的背后，隐含着两方面的核心问题：</p><ul>
<li>消息队列的消息存取需求是什么？</li>
<li>Redis如何实现消息队列的需求？</li>
</ul><p>这节课，我们就来聊一聊消息队列的特征和Redis提供的消息队列方案。只有把这两方面的知识和实践经验串连起来，才能彻底理解基于Redis实现消息队列的技术实践。以后当你需要为分布式系统组件做消息队列选型时，就可以根据组件通信量和消息通信速度的要求，选择出适合的Redis消息队列方案了。</p><p>我们先来看下第一个问题：消息队列的消息读取有什么样的需求？</p><h2>消息队列的消息存取需求</h2><p>我先介绍一下消息队列存取消息的过程。在分布式系统中，当两个组件要基于消息队列进行通信时，一个组件会把要处理的数据以消息的形式传递给消息队列，然后，这个组件就可以继续执行其他操作了；远端的另一个组件从消息队列中把消息读取出来，再在本地进行处理。</p><!-- [[[read_end]]] --><p>为了方便你理解，我还是借助一个例子来解释一下。</p><p>假设组件1需要对采集到的数据进行求和计算，并写入数据库，但是，消息到达的速度很快，组件1没有办法及时地既做采集，又做计算，并且写入数据库。所以，我们可以使用基于消息队列的通信，让组件1把数据x和y保存为JSON格式的消息，再发到消息队列，这样它就可以继续接收新的数据了。组件2则异步地从消息队列中把数据读取出来，在服务器2上进行求和计算后，再写入数据库。这个过程如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/d7/bc/d79d46ec4aa22bf46fde3ae1a99fc2bc.jpg?wh=3000*1459" alt=""></p><p>我们一般把消息队列中发送消息的组件称为生产者（例子中的组件1），把接收消息的组件称为消费者（例子中的组件2），下图展示了一个通用的消息队列的架构模型：</p><p><img src="https://static001.geekbang.org/resource/image/f4/62/f470bb957c1faff674c08b1fa65a3a62.jpg?wh=3000*857" alt=""></p><p>在使用消息队列时，消费者可以异步读取生产者消息，然后再进行处理。这样一来，即使生产者发送消息的速度远远超过了消费者处理消息的速度，生产者已经发送的消息也可以缓存在消息队列中，避免阻塞生产者，这是消息队列作为分布式组件通信的一大优势。</p><p><strong>不过，消息队列在存取消息时，必须要满足三个需求，分别是消息保序、处理重复的消息和保证消息可靠性。</strong></p><h3>需求一：消息保序</h3><p>虽然消费者是异步处理消息，但是，消费者仍然需要按照生产者发送消息的顺序来处理消息，避免后发送的消息被先处理了。对于要求消息保序的场景来说，一旦出现这种消息被乱序处理的情况，就可能会导致业务逻辑被错误执行，从而给业务方造成损失。</p><p>我们来看一个更新商品库存的场景。</p><p>假设生产者负责接收库存更新请求，消费者负责实际更新库存，现有库存量是10。生产者先后发送了消息1和消息2，消息1要把商品X的库存记录更新为5，消息2是把商品X库存更新为3。如果消息1和2在消息队列中无法保序，出现消息2早于消息1被处理的情况，那么，很显然，库存更新就出错了。这是业务应用无法接受的。</p><p>面对这种情况，你可能会想到一种解决方案：不要把更新后的库存量作为生产者发送的消息，而是<strong>把库存扣除值作为消息的内容</strong>。这样一来，消息1是扣减库存量5，消息2是扣减库存量2。如果消息1和消息2之间没有库存查询请求的话，即使消费者先处理消息2，再处理消息1，这个方案也能够保证最终的库存量是正确的，也就是库存量为3。</p><p>但是，我们还需要考虑这样一种情况：假如消费者收到了这样三条消息：消息1是扣减库存量5，消息2是读取库存量，消息3是扣减库存量2，此时，如果消费者先处理了消息3（把库存量扣减2），那么库存量就变成了8。然后，消费者处理了消息2，读取当前的库存量是8，这就会出现库存量查询不正确的情况。从业务应用层面看，消息1、2、3应该是顺序执行的，所以，消息2查询到的应该是扣减了5以后的库存量，而不是扣减了2以后的库存量。所以，用库存扣除值作为消息的方案，在消息中同时包含读写操作的场景下，会带来数据读取错误的问题。而且，这个方案还会面临一个问题，那就是重复消息处理。</p><h3>需求二：重复消息处理</h3><p>消费者从消息队列读取消息时，有时会因为网络堵塞而出现消息重传的情况。此时，消费者可能会收到多条重复的消息。对于重复的消息，消费者如果多次处理的话，就可能造成一个业务逻辑被多次执行，如果业务逻辑正好是要修改数据，那就会出现数据被多次修改的问题了。</p><p>还是以库存更新为例，假设消费者收到了一次消息1，要扣减库存量5，然后又收到了一次消息1，那么，如果消费者无法识别这两条消息实际是一条相同消息的话，就会执行两次扣减库存量5的操作，此时，库存量就不对了。这当然也是无法接受的。</p><h3>需求三：消息可靠性保证</h3><p>另外，消费者在处理消息的时候，还可能出现因为故障或宕机导致消息没有处理完成的情况。此时，消息队列需要能提供消息可靠性的保证，也就是说，当消费者重启后，可以重新读取消息再次进行处理，否则，就会出现消息漏处理的问题了。</p><p>Redis的List和Streams两种数据类型，就可以满足消息队列的这三个需求。我们先来了解下基于List的消息队列实现方法。</p><h2>基于List的消息队列解决方案</h2><p>List本身就是按先进先出的顺序对数据进行存取的，所以，如果使用List作为消息队列保存消息的话，就已经能满足消息保序的需求了。</p><p>具体来说，生产者可以使用LPUSH命令把要发送的消息依次写入List，而消费者则可以使用RPOP命令，从List的另一端按照消息的写入顺序，依次读取消息并进行处理。</p><p>如下图所示，生产者先用LPUSH写入了两条库存消息，分别是5和3，表示要把库存更新为5和3；消费者则用RPOP把两条消息依次读出，然后进行相应的处理。</p><p><img src="https://static001.geekbang.org/resource/image/b0/7c/b0959216cbce7ac383ce206b8884777c.jpg?wh=3000*2250" alt=""></p><p>不过，在消费者读取数据时，有一个潜在的性能风险点。</p><p>在生产者往List中写入数据时，List并不会主动地通知消费者有新消息写入，如果消费者想要及时处理消息，就需要在程序中不停地调用RPOP命令（比如使用一个while(1)循环）。如果有新消息写入，RPOP命令就会返回结果，否则，RPOP命令返回空值，再继续循环。</p><p>所以，即使没有新消息写入List，消费者也要不停地调用RPOP命令，这就会导致消费者程序的CPU一直消耗在执行RPOP命令上，带来不必要的性能损失。</p><p>为了解决这个问题，Redis提供了BRPOP命令。<strong>BRPOP命令也称为阻塞式读取，客户端在没有读到队列数据时，自动阻塞，直到有新的数据写入队列，再开始读取新数据</strong>。和消费者程序自己不停地调用RPOP命令相比，这种方式能节省CPU开销。</p><p>消息保序的问题解决了，接下来，我们还需要考虑解决重复消息处理的问题，这里其实有一个要求：<strong>消费者程序本身能对重复消息进行判断。</strong></p><p>一方面，消息队列要能给每一个消息提供全局唯一的ID号；另一方面，消费者程序要把已经处理过的消息的ID号记录下来。</p><p>当收到一条消息后，消费者程序就可以对比收到的消息ID和记录的已处理过的消息ID，来判断当前收到的消息有没有经过处理。如果已经处理过，那么，消费者程序就不再进行处理了。这种处理特性也称为幂等性，幂等性就是指，对于同一条消息，消费者收到一次的处理结果和收到多次的处理结果是一致的。</p><p>不过，List本身是不会为每个消息生成ID号的，所以，消息的全局唯一ID号就需要生产者程序在发送消息前自行生成。生成之后，我们在用LPUSH命令把消息插入List时，需要在消息中包含这个全局唯一ID。</p><p>例如，我们执行以下命令，就把一条全局ID为101030001、库存量为5的消息插入了消息队列：</p><pre><code>LPUSH mq &quot;101030001:stock:5&quot;
(integer) 1
</code></pre><p>最后，我们再来看下，List类型是如何保证消息可靠性的。</p><p>当消费者程序从List中读取一条消息后，List就不会再留存这条消息了。所以，如果消费者程序在处理消息的过程出现了故障或宕机，就会导致消息没有处理完成，那么，消费者程序再次启动后，就没法再次从List中读取消息了。</p><p>为了留存消息，List类型提供了BRPOPLPUSH命令，这个命令的作用是让消费者程序从一个List中读取消息，同时，Redis会把这个消息再插入到另一个List（可以叫作备份List）留存。这样一来，如果消费者程序读了消息但没能正常处理，等它重启后，就可以从备份List中重新读取消息并进行处理了。</p><p>我画了一张示意图，展示了使用BRPOPLPUSH命令留存消息，以及消费者再次读取消息的过程，你可以看下。</p><p><img src="https://static001.geekbang.org/resource/image/50/3d/5045395da08317b546aab7eb698d013d.jpg?wh=3000*2135" alt=""></p><p>生产者先用LPUSH把消息“5”“3”插入到消息队列mq中。消费者程序使用BRPOPLPUSH命令读取消息“5”，同时，消息“5”还会被Redis插入到mqback队列中。如果消费者程序处理消息“5”时宕机了，等它重启后，可以从mqback中再次读取消息“5”，继续处理。</p><p>好了，到这里，你可以看到，基于List类型，我们可以满足分布式组件对消息队列的三大需求。但是，在用List做消息队列时，我们还可能遇到过一个问题：<strong>生产者消息发送很快，而消费者处理消息的速度比较慢，这就导致List中的消息越积越多，给Redis的内存带来很大压力</strong>。</p><p>这个时候，我们希望启动多个消费者程序组成一个消费组，一起分担处理List中的消息。但是，List类型并不支持消费组的实现。那么，还有没有更合适的解决方案呢？这就要说到Redis从5.0版本开始提供的Streams数据类型了。</p><p>和List相比，Streams同样能够满足消息队列的三大需求。而且，它还支持消费组形式的消息读取。接下来，我们就来了解下Streams的使用方法。</p><h2>基于Streams的消息队列解决方案</h2><p>Streams是Redis专门为消息队列设计的数据类型，它提供了丰富的消息队列操作命令。</p><ul>
<li>XADD：插入消息，保证有序，可以自动生成全局唯一ID；</li>
<li>XREAD：用于读取消息，可以按ID读取数据；</li>
<li>XREADGROUP：按消费组形式读取消息；</li>
<li>XPENDING和XACK：XPENDING命令可以用来查询每个消费组内所有消费者已读取但尚未确认的消息，而XACK命令用于向消息队列确认消息处理已完成。</li>
</ul><p>首先，我们来学习下Streams类型存取消息的操作XADD。</p><p>XADD命令可以往消息队列中插入新消息，消息的格式是键-值对形式。对于插入的每一条消息，Streams可以自动为其生成一个全局唯一的ID。</p><p>比如说，我们执行下面的命令，就可以往名称为mqstream的消息队列中插入一条消息，消息的键是repo，值是5。其中，消息队列名称后面的<code>*</code>，表示让Redis为插入的数据自动生成一个全局唯一的ID，例如“1599203861727-0”。当然，我们也可以不用<code>*</code>，直接在消息队列名称后自行设定一个ID号，只要保证这个ID号是全局唯一的就行。不过，相比自行设定ID号，使用<code>*</code>会更加方便高效。</p><pre><code>XADD mqstream * repo 5
&quot;1599203861727-0&quot;
</code></pre><p>可以看到，消息的全局唯一ID由两部分组成，第一部分“1599203861727”是数据插入时，以毫秒为单位计算的当前服务器时间，第二部分表示插入消息在当前毫秒内的消息序号，这是从0开始编号的。例如，“1599203861727-0”就表示在“1599203861727”毫秒内的第1条消息。</p><p>当消费者需要读取消息时，可以直接使用XREAD命令从消息队列中读取。</p><p>XREAD在读取消息时，可以指定一个消息ID，并从这个消息ID的下一条消息开始进行读取。</p><p>例如，我们可以执行下面的命令，从ID号为1599203861727-0的消息开始，读取后续的所有消息（示例中一共3条）。</p><pre><code>XREAD BLOCK 100 STREAMS  mqstream 1599203861727-0
1) 1) &quot;mqstream&quot;
   2) 1) 1) &quot;1599274912765-0&quot;
         2) 1) &quot;repo&quot;
            2) &quot;3&quot;
      2) 1) &quot;1599274925823-0&quot;
         2) 1) &quot;repo&quot;
            2) &quot;2&quot;
      3) 1) &quot;1599274927910-0&quot;
         2) 1) &quot;repo&quot;
            2) &quot;1&quot;
</code></pre><p>另外，消费者也可以在调用XRAED时设定block配置项，实现类似于BRPOP的阻塞读取操作。当消息队列中没有消息时，一旦设置了block配置项，XREAD就会阻塞，阻塞的时长可以在block配置项进行设置。</p><p>举个例子，我们来看一下下面的命令，其中，命令最后的“$”符号表示读取最新的消息，同时，我们设置了block 10000的配置项，10000的单位是毫秒，表明XREAD在读取最新消息时，如果没有消息到来，XREAD将阻塞10000毫秒（即10秒），然后再返回。下面命令中的XREAD执行后，消息队列mqstream中一直没有消息，所以，XREAD在10秒后返回空值（nil）。</p><pre><code>XREAD block 10000 streams mqstream $
(nil)
(10.00s)
</code></pre><p>刚刚讲到的这些操作是List也支持的，接下来，我们再来学习下Streams特有的功能。</p><p>Streams本身可以使用XGROUP创建消费组，创建消费组之后，Streams可以使用XREADGROUP命令让消费组内的消费者读取消息，</p><p>例如，我们执行下面的命令，创建一个名为group1的消费组，这个消费组消费的消息队列是mqstream。</p><pre><code>XGROUP create mqstream group1 0
OK
</code></pre><p>然后，我们再执行一段命令，让group1消费组里的消费者consumer1从mqstream中读取所有消息，其中，命令最后的参数“&gt;”，表示从第一条尚未被消费的消息开始读取。因为在consumer1读取消息前，group1中没有其他消费者读取过消息，所以，consumer1就得到mqstream消息队列中的所有消息了（一共4条）。</p><pre><code>XREADGROUP group group1 consumer1 streams mqstream &gt;
1) 1) &quot;mqstream&quot;
   2) 1) 1) &quot;1599203861727-0&quot;
         2) 1) &quot;repo&quot;
            2) &quot;5&quot;
      2) 1) &quot;1599274912765-0&quot;
         2) 1) &quot;repo&quot;
            2) &quot;3&quot;
      3) 1) &quot;1599274925823-0&quot;
         2) 1) &quot;repo&quot;
            2) &quot;2&quot;
      4) 1) &quot;1599274927910-0&quot;
         2) 1) &quot;repo&quot;
            2) &quot;1&quot;
</code></pre><p>需要注意的是，消息队列中的消息一旦被消费组里的一个消费者读取了，就不能再被该消费组内的其他消费者读取了。比如说，我们执行完刚才的XREADGROUP命令后，再执行下面的命令，让group1内的consumer2读取消息时，consumer2读到的就是空值，因为消息已经被consumer1读取完了，如下所示：</p><pre><code>XREADGROUP group group1 consumer2  streams mqstream 0
1) 1) &quot;mqstream&quot;
   2) (empty list or set)
</code></pre><p>使用消费组的目的是让组内的多个消费者共同分担读取消息，所以，我们通常会让每个消费者读取部分消息，从而实现消息读取负载在多个消费者间是均衡分布的。例如，我们执行下列命令，让group2中的consumer1、2、3各自读取一条消息。</p><pre><code>XREADGROUP group group2 consumer1 count 1 streams mqstream &gt;
1) 1) &quot;mqstream&quot;
   2) 1) 1) &quot;1599203861727-0&quot;
         2) 1) &quot;repo&quot;
            2) &quot;5&quot;

XREADGROUP group group2 consumer2 count 1 streams mqstream &gt;
1) 1) &quot;mqstream&quot;
   2) 1) 1) &quot;1599274912765-0&quot;
         2) 1) &quot;repo&quot;
            2) &quot;3&quot;

XREADGROUP group group2 consumer3 count 1 streams mqstream &gt;
1) 1) &quot;mqstream&quot;
   2) 1) 1) &quot;1599274925823-0&quot;
         2) 1) &quot;repo&quot;
            2) &quot;2&quot;
</code></pre><p>为了保证消费者在发生故障或宕机再次重启后，仍然可以读取未处理完的消息，Streams会自动使用内部队列（也称为PENDING List）留存消费组里每个消费者读取的消息，直到消费者使用XACK命令通知Streams“消息已经处理完成”。如果消费者没有成功处理消息，它就不会给Streams发送XACK命令，消息仍然会留存。此时，消费者可以在重启后，用XPENDING命令查看已读取、但尚未确认处理完成的消息。</p><p>例如，我们来查看一下group2中各个消费者已读取、但尚未确认的消息个数。其中，XPENDING返回结果的第二、三行分别表示group2中所有消费者读取的消息最小ID和最大ID。</p><pre><code>XPENDING mqstream group2
1) (integer) 3
2) &quot;1599203861727-0&quot;
3) &quot;1599274925823-0&quot;
4) 1) 1) &quot;consumer1&quot;
      2) &quot;1&quot;
   2) 1) &quot;consumer2&quot;
      2) &quot;1&quot;
   3) 1) &quot;consumer3&quot;
      2) &quot;1&quot;
</code></pre><p>如果我们还需要进一步查看某个消费者具体读取了哪些数据，可以执行下面的命令：</p><pre><code>XPENDING mqstream group2 - + 10 consumer2
1) 1) &quot;1599274912765-0&quot;
   2) &quot;consumer2&quot;
   3) (integer) 513336
   4) (integer) 1
</code></pre><p>可以看到，consumer2已读取的消息的ID是1599274912765-0。</p><p>一旦消息1599274912765-0被consumer2处理了，consumer2就可以使用XACK命令通知Streams，然后这条消息就会被删除。当我们再使用XPENDING命令查看时，就可以看到，consumer2已经没有已读取、但尚未确认处理的消息了。</p><pre><code> XACK mqstream group2 1599274912765-0
(integer) 1
XPENDING mqstream group2 - + 10 consumer2
(empty list or set)
</code></pre><p>现在，我们就知道了用Streams实现消息队列的方法，我还想再强调下，Streams是Redis 5.0专门针对消息队列场景设计的数据类型，如果你的Redis是5.0及5.0以后的版本，就可以考虑把Streams用作消息队列了。</p><h2>小结</h2><p>这节课，我们学习了分布式系统组件使用消息队列时的三大需求：消息保序、重复消息处理和消息可靠性保证，这三大需求可以进一步转换为对消息队列的三大要求：消息数据有序存取，消息数据具有全局唯一编号，以及消息数据在消费完成后被删除。</p><p>我画了一张表格，汇总了用List和Streams实现消息队列的特点和区别。当然，在实践的过程中，你也可以根据新的积累，进一步补充和完善这张表。</p><p><img src="https://static001.geekbang.org/resource/image/b2/14/b2d6581e43f573da6218e790bb8c6814.jpg?wh=2922*943" alt=""></p><p>其实，关于Redis是否适合做消息队列，业界一直是有争论的。很多人认为，要使用消息队列，就应该采用Kafka、RabbitMQ这些专门面向消息队列场景的软件，而Redis更加适合做缓存。</p><p>根据这些年做Redis研发工作的经验，我的看法是：Redis是一个非常轻量级的键值数据库，部署一个Redis实例就是启动一个进程，部署Redis集群，也就是部署多个Redis实例。而Kafka、RabbitMQ部署时，涉及额外的组件，例如Kafka的运行就需要再部署ZooKeeper。相比Redis来说，Kafka和RabbitMQ一般被认为是重量级的消息队列。</p><p>所以，关于是否用Redis做消息队列的问题，不能一概而论，我们需要考虑业务层面的数据体量，以及对性能、可靠性、可扩展性的需求。如果分布式系统中的组件消息通信量不大，那么，Redis只需要使用有限的内存空间就能满足消息存储的需求，而且，Redis的高性能特性能支持快速的消息读写，不失为消息队列的一个好的解决方案。</p><h2>每课一问</h2><p>按照惯例，我给你提个小问题。如果一个生产者发送给消息队列的消息，需要被多个消费者进行读取和处理（例如，一个消息是一条从业务系统采集的数据，既要被消费者1读取进行实时计算，也要被消费者2读取并留存到分布式文件系统HDFS中，以便后续进行历史查询），你会使用Redis的什么数据类型来解决这个问题呢？</p><p>欢迎在留言区写下你的思考和答案，如果觉得今天的内容对你有所帮助，也欢迎你帮我分享给更多人。我们下节课见。</p>
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
  <div class="_2_QraFYR_0">如果一个生产者发送给消息队列的消息，需要被多个消费者进行读取和处理，你会使用Redis的什么数据类型来解决这个问题？<br><br>这种情况下，只能使用Streams数据类型来解决。使用Streams数据类型，创建多个消费者组，就可以实现同时消费生产者的数据。每个消费者组内可以再挂多个消费者分担读取消息进行消费，消费完成后，各自向Redis发送XACK，标记自己的消费组已经消费到了哪个位置，而且消费组之间互不影响。<br><br>另外，老师在介绍使用List用作队列时，为了保证消息可靠性，使用BRPOPLPUSH命令把消息取出的同时，还把消息插入到备份队列中，从而防止消费者故障导致消息丢失。<br><br>这种情况下，还需要额外做一些工作，也就是维护这个备份队列：每次执行BRPOPLPUSH命令后，因为都会把消息插入一份到备份队列中，所以当消费者成功消费取出的消息后，最好把备份队列中的消息删除，防止备份队列存储过多无用的数据，导致内存浪费。<br><br>这篇文章主要是讲消息队列的使用，借这个机会，也顺便总结一下使用消息队列时的注意点：<br><br>在使用消息队列时，重点需要关注的是如何保证不丢消息？<br><br>那么下面就来分析一下，哪些情况下，会丢消息，以及如何解决？<br><br>1、生产者在发布消息时异常：<br><br>a) 网络故障或其他问题导致发布失败（直接返回错误，消息根本没发出去）<br>b) 网络抖动导致发布超时（可能发送数据包成功，但读取响应结果超时了，不知道结果如何）<br><br>情况a还好，消息根本没发出去，那么重新发一次就好了。但是情况b没办法知道到底有没有发布成功，所以也只能再发一次。所以这两种情况，生产者都需要重新发布消息，直到成功为止（一般设定一个最大重试次数，超过最大次数依旧失败的需要报警处理）。这就会导致消费者可能会收到重复消息的问题，所以消费者需要保证在收到重复消息时，依旧能保证业务的正确性（设计幂等逻辑），一般需要根据具体业务来做，例如使用消息的唯一ID，或者版本号配合业务逻辑来处理。<br><br>2、消费者在处理消息时异常：<br><br>也就是消费者把消息拿出来了，但是还没处理完，消费者就挂了。这种情况，需要消费者恢复时，依旧能处理之前没有消费成功的消息。使用List当作队列时，也就是利用老师文章所讲的备份队列来保证，代价是增加了维护这个备份队列的成本。而Streams则是采用ack的方式，消费成功后告知中间件，这种方式处理起来更优雅，成熟的队列中间件例如RabbitMQ、Kafka都是采用这种方式来保证消费者不丢消息的。<br><br>3、消息队列中间件丢失消息<br><br>上面2个层面都比较好处理，只要客户端和服务端配合好，就能保证生产者和消费者都不丢消息。但是，如果消息队列中间件本身就不可靠，也有可能会丢失消息，毕竟生产者和消费这都依赖它，如果它不可靠，那么生产者和消费者无论怎么做，都无法保证数据不丢失。<br><br>a) 在用Redis当作队列或存储数据时，是有可能丢失数据的：一个场景是，如果打开AOF并且是每秒写盘，因为这个写盘过程是异步的，Redis宕机时会丢失1秒的数据。而如果AOF改为同步写盘，那么写入性能会下降。另一个场景是，如果采用主从集群，如果写入量比较大，从库同步存在延迟，此时进行主从切换，也存在丢失数据的可能（从库还未同步完成主库发来的数据就被提成主库）。总的来说，Redis不保证严格的数据完整性和主从切换时的一致性。我们在使用Redis时需要注意。<br><br>b) 而采用RabbitMQ和Kafka这些专业的队列中间件时，就没有这个问题了。这些组件一般是部署一个集群，生产者在发布消息时，队列中间件一般会采用写多个节点+预写磁盘的方式保证消息的完整性，即便其中一个节点挂了，也能保证集群的数据不丢失。当然，为了做到这些，方案肯定比Redis设计的要复杂（毕竟是专们针对队列场景设计的）。<br><br>综上，Redis可以用作队列，而且性能很高，部署维护也很轻量，但缺点是无法严格保数据的完整性（个人认为这就是业界有争议要不要使用Redis当作队列的地方）。而使用专业的队列中间件，可以严格保证数据的完整性，但缺点是，部署维护成本高，用起来比较重。<br><br>所以我们需要根据具体情况进行选择，如果对于丢数据不敏感的业务，例如发短信、发通知的场景，可以采用Redis作队列。如果是金融相关的业务场景，例如交易、支付这类，建议还是使用专业的队列中间件。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-11 01:51:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/52/40/e57a736e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>pedro</span>
  </div>
  <div class="_2_QraFYR_0">每次想回答问题，看到 Kaito 的留言，顿时就有一种“眼前有景道不得，崔颢题诗在上头”的感觉。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-11 09:01:53</div>
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
  <div class="_2_QraFYR_0">1，作者讲了什么？<br>    如何使用redis实现消息队列的需求<br><br>2，作者是怎么把这事给说明白的？<br>    1，将一个问题拆解为两个具体的小问题：消息队列应具备哪些特性，Redis能否实现这些特性<br><br>3，为了讲明白，作者讲了哪些要点？，有哪些亮点？<br>    1，亮点1：作者首先将一个相对模糊的问题，拆解成为两个问题更精确的问题<br>    2，要点1：消息队列读取需求有三点：消息保存，消息唯一，消息可靠性<br>    3，要点2：消息还要确保有序，削峰平谷，消费性能有弹性<br>    4，要点3：Redis的list和5.0后的Stream数据结构可以满足<br>    5，要点4：消息队列的三大要求：消息数据有序存取，消息数据具有全局唯一编号，消息数据在消费完成后被删除<br>    <br>4，对于作者所讲，我有哪些发散性思考？<br>  	解决问题的前提是搞清楚问题是什么<br>	开篇：“Redis适合做消息队列吗”？对于这样的问题，我本能的就会直接想：“用Redis做消息队列？应该可以吧，很多资料上都这么说的”。<br>	这样的回答，证明我并没有思考，只是陈述了我所知道的一点东西。我总是渴求快速有个答案，没有在意答案是否正确，更没有思考该如何回答问题。<br>	虽然我总会说要想解决问题，首先要搞清楚问题是什么。这个句话似乎有点白痴，自己遇到的问题，这个问题不就它本身吗？怎么还会需要搞清楚呢？<br>	其实不然，我们遇到的往往未必是问题的真身，而只是问题的表面现象或衍生出的问题。其实一个好问题的本身，就是一个好答案，而一个好问题胜过无数好答案。<br>	那什么才是真正的解答问题？这需要先回答如何回答这个问题，也就是要先搞清楚要从哪些方面解释问题，而不是铺陈信息。<br>	老师开篇对问题的回答方法，就是个非常好的范例<br>	他是怎么解答的呢？<br>	他并没有立即回答行或不行（这是我们最常有的反应），而是向后退了一步，问这个问题真正的问题是啥呢？消息队列存取消息需要哪些特性？Redis如何实现这些特性？<br>	这就将一个相对模糊的问题拆解为两个较清晰的小问题，分别作答，最终得到一篇很好的文章<br><br>5，将来在哪些场景里，我能够使用它？<br><br>6，留言区收获<br>1，Redis是否可以作为消息队列？如果可以，哪些场景适合使用Redis，而不是消息中间件？<br>	答：这个问题应当进一步拆分为：消息队列读写消息有哪些需求和Redis如何实现这些需求。<br>        首先消息队列的消息读写有三大需求：消息读写的有序性，消息数据的唯一性，消息消费后数据删除。<br>        针对这三大需求，Redis的List和5.0后的Stream数据结构，可以支持。<br>        对于List，POP和PUSH命令，还有阻塞式的，备份式的。<br>        对于Stream，是Redis5.0后提供新的数据结构，专门用户消息队列，他可以生成全局唯一id，还有能够创建消费者组，多个消费者同时消费<br>        Stream类型，使用了应答机制，消费者消费完毕后会给Streams发送XACK命令，否则消息将会保存Streams的内部队列中，使用XPENDING命令可以查看<br>       就Redis的相关数据结构而言，是可以作为消息队列的。但也要注意使用它的场景，虽然它性能很高，部署维护也轻量，但缺点是无法严格保证数据的完整性。<br>    他适用于消息量并不是非常巨大，数据不是非常重要，从而不必引入其他消息组件的场景，如发短信，站内信<br><br>2，如果一个生产者发送给消息队列的消息，需要被多个消费者进行读取和处理，Redis的什么数据类型可以解决这个问题？<br>    答：如果要使用Redis实现这个需求，Redis的Streams数据类型可以实现。<br>		Streams可以使用XADD向队列中写入消息，XGOURP创建消费者组，XREADGROUP已消费者组形式消费消息，XACK向消息队列确认处理完成，XPENDING查询已消费待确认的消息<br><br>3，如果使用Redis作为消息队列，有哪些事项需要注意？<br>    答：消息的可靠性，重点需要关注的是如何保证不丢消息<br><br><br>4，在使用消息队列时，如何保证不丢消息？<br>    答：消息丢失可能会发生在三个环节：生产者发布消息，消息者消费消息，消息中间件丢失消息<br>    生产者丢失消息一般通过重试机制和全局唯一id来解决，<br>	消费者消费消息一般通过ack方式问询上报消费进度，<br>	消息中间件宕机，一般通过主从备份，分布式集群来解决</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-19 17:39:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83ersGSic8ib7OguJv6CJiaXY0s4n9C7Z51sWxTTljklFpq3ZAIWXoFTPV5oLo0GMTkqW5sYJRRnibNqOJQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>walle斌</span>
  </div>
  <div class="_2_QraFYR_0">redis作为队列，我觉得最大的问题应该是消息积压的问题。。任何一个mq 都要应对 高流量的消息积压问题。 </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的。所以，如果应用场景中压力不大，Redis应用起来也相对简单，可以作为一个方案备选。但是如果消息压力大，还是会考虑专用的一些MQ，例如Kafka。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-10 09:30:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/62/c9/7da27891.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>DKSky</span>
  </div>
  <div class="_2_QraFYR_0">老师的文中说“我们希望启动多个消费者程序组成一个消费组，一起分担处理 List 中的消息。但是，List 类型并不支持消费组的实现。那么，还有没有更合适的解决方案呢？“<br>这句话不太明白，如果多个消费者客户端使用brpop命令阻塞式的“抢”列表尾部的元素，这些消费者不属于同一个consumer group吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-11 11:52:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/3e/3a/2267d2a3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>hoppo</span>
  </div>
  <div class="_2_QraFYR_0">关于顺序消费的问题，其实要严格保证顺序的话，只有生产者-队列-消费者完全的一对一才行。就好比List这种。<br><br>如果使用Stream消费组的话，多个消费者一起操作的时候，是没法保证顺序性的。<br><br>专业的消息队列中间件一般都采用局部有序的方法：需要有序操作的部分通过路由发到一个队列中，队列-消费者采用一对一的方式。<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-15 16:12:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/ed/89/86340059.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>licong</span>
  </div>
  <div class="_2_QraFYR_0">redis的publish&#47;subscribe也可以作为简单的MQ使用，老师的专栏好像没提到这个</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-13 12:35:49</div>
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
  <div class="_2_QraFYR_0">1. 问题回答<br>每个 Stream 都可以挂多个消费组，每个消费组会有个游标 last_delivered_id 在 Stream 数组之上往前移动，表示当前消费组已经消费到哪条消息了。如果这是对的，那只需要开两个消费者组，消费者 1 在组1，消费者2在组2不就行了么。<br>这个 last_delivered_id 有点像 Kafka 的 offset。<br><br>2. 有点不同的观点<br>Kafka 其实已经考虑去掉 ZK 了，而且适合高吞吐量的业务，不过不适合流式处理。专业的事，还是教给专业的来做好点。<br><br>我之前用过 Redis 的 publish 和 subscribe 命令来做不同系统的数据同步。不过那个在用 Jedis 来订阅有坑，必须循环调用，不然会在一定时间后被 down 掉。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-11 02:13:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Yipsen</span>
  </div>
  <div class="_2_QraFYR_0">消费组消费时虽然分摊了压力，但是怎么保证顺序消费呢？如果要保证岂不是要上一条消息的消费者必须发出ack后，消费组的另一个成员消费者才能开始消费消息，那不就等于加锁来保证顺序了，这样是不是消费组分摊就形同虚设或者没有起到应有的作用了？所以负载处理+顺序消费redis stream是怎么保证的？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-03 15:08:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/f4/14/c1c81d88.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Luke 💨</span>
  </div>
  <div class="_2_QraFYR_0">不懂为啥要用这种模式实现消息队列，感觉有点牵强，而且大部分功能需要自己实现，redis内部不是有发布订阅模式吗，就是一种消息队列啊，两者有啥不一样吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-21 05:35:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/23/24/e5/a83c0ff5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Aecho1610213142</span>
  </div>
  <div class="_2_QraFYR_0">老师好！多个消费者 如何保证消息的顺序执行？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-13 17:25:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/12/70/10faf04b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Lywane</span>
  </div>
  <div class="_2_QraFYR_0">我们希望启动多个消费者程序组成一个消费组，一起分担处理 List 中的消息。但是，List 类型并不支持消费组的实现 （多个消费者执行BRPOP不行吗）</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-12 10:21:46</div>
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
  <div class="_2_QraFYR_0">请教下老师两个问题：<br>1.如果消息延迟了，如何做好监控？<br>2.如何尽可能的减少消息的延迟，我好像知道一个很牛逼的技术叫“零拷贝”，但是不太理解其原理，望老师解惑。谢谢。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-11 15:26:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>HJ_Liu</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，有个问题想更深入了解下。用Stream类型用作消息队列，用多个消费组，是不是也不能保证处理的顺序性？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-23 15:32:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/25/42/53/577009a5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Wang Junling</span>
  </div>
  <div class="_2_QraFYR_0">我对Redis的源码不是很熟，如果位大佬看到我的提问请帮忙解释一下。<br>1、LPUSH和RPUSH的运行原理是什么样的呢？时间算法复杂度是？<br>2、RPOP和LPOP的运行原理和时间复杂度是什么样的呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-28 18:49:42</div>
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
  <div class="_2_QraFYR_0">使用list，list留存消息队列消费者可指定，且消费者2读取后再删除。<br>stream 留存消息队列是内部匿名的，且消费者1消费完发送ACK后，消息就被删除，导致消费者2读取不到。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-24 17:05:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/b6/75/32c19395.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>土豆白菜</span>
  </div>
  <div class="_2_QraFYR_0">茅草房-土房-平房</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-11 19:58:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/59/8c/ba81a832.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>刀斧手何在</span>
  </div>
  <div class="_2_QraFYR_0">Redis做消息队列的优势是高并发写入性能和水平扩展能力。最大的坑 我觉得就是Kaito同学说的第3点 数据可靠性</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-11 08:34:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/41/13/ab14ad25.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>树心</span>
  </div>
  <div class="_2_QraFYR_0">评论区小收获：命令的时间复杂度可以在官网查看</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-15 19:29:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/ee/c6/bebcbcf0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>俯瞰风景.</span>
  </div>
  <div class="_2_QraFYR_0">消息队列的目的是在分布式的系统间进行通信，通信方式是通过网络传输的方式，通过网络传输就有消息丢失、重复发送消息、不同消息到达顺序混乱的可能。所以消息队列应用的三大需求是：<br>  1、消息保序<br>  2、重复消息处理<br>  3、消息可靠性保证<br><br>对应处理方案是：<br>  1、消息数据有序存取<br>  2、消息数据具有全局唯一编号<br>  3、消息数据在未消费完宕机恢复时继续消费，消费完成后被删除</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-30 20:50:25</div>
  </div>
</div>
</div>
</li>
</ul>