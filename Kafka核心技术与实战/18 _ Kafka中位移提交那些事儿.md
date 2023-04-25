<audio title="18 _ Kafka中位移提交那些事儿" src="https://static001.geekbang.org/resource/audio/e9/1d/e906d8f6d04720fd021b92663becf61d.mp3" controls="controls"></audio> 
<p>你好，我是胡夕。今天我们来聊聊Kafka中位移提交的那些事儿。</p><p>之前我们说过，Consumer端有个位移的概念，它和消息在分区中的位移不是一回事儿，虽然它们的英文都是Offset。今天我们要聊的位移是Consumer的消费位移，它记录了Consumer要消费的下一条消息的位移。这可能和你以前了解的有些出入，不过切记是下一条消息的位移，而不是目前最新消费消息的位移。</p><p>我来举个例子说明一下。假设一个分区中有10条消息，位移分别是0到9。某个Consumer应用已消费了5条消息，这就说明该Consumer消费了位移为0到4的5条消息，此时Consumer的位移是5，指向了下一条消息的位移。</p><p><strong>Consumer需要向Kafka汇报自己的位移数据，这个汇报过程被称为提交位移</strong>（Committing Offsets）。因为Consumer能够同时消费多个分区的数据，所以位移的提交实际上是在分区粒度上进行的，即<strong>Consumer需要为分配给它的每个分区提交各自的位移数据</strong>。</p><p>提交位移主要是为了表征Consumer的消费进度，这样当Consumer发生故障重启之后，就能够从Kafka中读取之前提交的位移值，然后从相应的位移处继续消费，从而避免整个消费过程重来一遍。换句话说，位移提交是Kafka提供给你的一个工具或语义保障，你负责维持这个语义保障，即如果你提交了位移X，那么Kafka会认为所有位移值小于X的消息你都已经成功消费了。</p><!-- [[[read_end]]] --><p>这一点特别关键。因为位移提交非常灵活，你完全可以提交任何位移值，但由此产生的后果你也要一并承担。假设你的Consumer消费了10条消息，你提交的位移值却是20，那么从理论上讲，位移介于11～19之间的消息是有可能丢失的；相反地，如果你提交的位移值是5，那么位移介于5～9之间的消息就有可能被重复消费。所以，我想再强调一下，<strong>位移提交的语义保障是由你来负责的，Kafka只会“无脑”地接受你提交的位移</strong>。你对位移提交的管理直接影响了你的Consumer所能提供的消息语义保障。</p><p>鉴于位移提交甚至是位移管理对Consumer端的巨大影响，Kafka，特别是KafkaConsumer API，提供了多种提交位移的方法。<strong>从用户的角度来说，位移提交分为自动提交和手动提交；从Consumer端的角度来说，位移提交分为同步提交和异步提交</strong>。</p><p>我们先来说说自动提交和手动提交。所谓自动提交，就是指Kafka Consumer在后台默默地为你提交位移，作为用户的你完全不必操心这些事；而手动提交，则是指你要自己提交位移，Kafka Consumer压根不管。</p><p>开启自动提交位移的方法很简单。Consumer端有个参数enable.auto.commit，把它设置为true或者压根不设置它就可以了。因为它的默认值就是true，即Java Consumer默认就是自动提交位移的。如果启用了自动提交，Consumer端还有个参数就派上用场了：auto.commit.interval.ms。它的默认值是5秒，表明Kafka每5秒会为你自动提交一次位移。</p><p>为了把这个问题说清楚，我给出了完整的Java代码。这段代码展示了设置自动提交位移的方法。有了这段代码做基础，今天后面的讲解我就不再展示完整的代码了。</p><pre><code>Properties props = new Properties();
     props.put(&quot;bootstrap.servers&quot;, &quot;localhost:9092&quot;);
     props.put(&quot;group.id&quot;, &quot;test&quot;);
     props.put(&quot;enable.auto.commit&quot;, &quot;true&quot;);
     props.put(&quot;auto.commit.interval.ms&quot;, &quot;2000&quot;);
     props.put(&quot;key.deserializer&quot;, &quot;org.apache.kafka.common.serialization.StringDeserializer&quot;);
     props.put(&quot;value.deserializer&quot;, &quot;org.apache.kafka.common.serialization.StringDeserializer&quot;);
     KafkaConsumer&lt;String, String&gt; consumer = new KafkaConsumer&lt;&gt;(props);
     consumer.subscribe(Arrays.asList(&quot;foo&quot;, &quot;bar&quot;));
     while (true) {
         ConsumerRecords&lt;String, String&gt; records = consumer.poll(100);
         for (ConsumerRecord&lt;String, String&gt; record : records)
             System.out.printf(&quot;offset = %d, key = %s, value = %s%n&quot;, record.offset(), record.key(), record.value());
     }
</code></pre><p>上面的第3、第4行代码，就是开启自动提交位移的方法。总体来说，还是很简单的吧。</p><p>和自动提交相反的，就是手动提交了。开启手动提交位移的方法就是设置enable.auto.commit为false。但是，仅仅设置它为false还不够，因为你只是告诉Kafka Consumer不要自动提交位移而已，你还需要调用相应的API手动提交位移。</p><p>最简单的API就是<strong>KafkaConsumer#commitSync()</strong>。该方法会提交KafkaConsumer#poll()返回的最新位移。从名字上来看，它是一个同步操作，即该方法会一直等待，直到位移被成功提交才会返回。如果提交过程中出现异常，该方法会将异常信息抛出。下面这段代码展示了commitSync()的使用方法：</p><pre><code>while (true) {
            ConsumerRecords&lt;String, String&gt; records =
                        consumer.poll(Duration.ofSeconds(1));
            process(records); // 处理消息
            try {
                        consumer.commitSync();
            } catch (CommitFailedException e) {
                        handle(e); // 处理提交失败异常
            }
}
</code></pre><p>可见，调用consumer.commitSync()方法的时机，是在你处理完了poll()方法返回的所有消息之后。如果你莽撞地过早提交了位移，就可能会出现消费数据丢失的情况。那么你可能会问，自动提交位移就不会出现消费数据丢失的情况了吗？它能恰到好处地把握时机进行位移提交吗？为了搞清楚这个问题，我们必须要深入地了解一下自动提交位移的顺序。</p><p>一旦设置了enable.auto.commit为true，Kafka会保证在开始调用poll方法时，提交上次poll返回的所有消息。从顺序上来说，poll方法的逻辑是先提交上一批消息的位移，再处理下一批消息，因此它能保证不出现消费丢失的情况。但自动提交位移的一个问题在于，<strong>它可能会出现重复消费</strong>。</p><p>在默认情况下，Consumer每5秒自动提交一次位移。现在，我们假设提交位移之后的3秒发生了Rebalance操作。在Rebalance之后，所有Consumer从上一次提交的位移处继续消费，但该位移已经是3秒前的位移数据了，故在Rebalance发生前3秒消费的所有数据都要重新再消费一次。虽然你能够通过减少auto.commit.interval.ms的值来提高提交频率，但这么做只能缩小重复消费的时间窗口，不可能完全消除它。这是自动提交机制的一个缺陷。</p><p>反观手动提交位移，它的好处就在于更加灵活，你完全能够把控位移提交的时机和频率。但是，它也有一个缺陷，就是在调用commitSync()时，Consumer程序会处于阻塞状态，直到远端的Broker返回提交结果，这个状态才会结束。在任何系统中，因为程序而非资源限制而导致的阻塞都可能是系统的瓶颈，会影响整个应用程序的TPS。当然，你可以选择拉长提交间隔，但这样做的后果是Consumer的提交频率下降，在下次Consumer重启回来后，会有更多的消息被重新消费。</p><p>鉴于这个问题，Kafka社区为手动提交位移提供了另一个API方法：<strong>KafkaConsumer#commitAsync()</strong>。从名字上来看它就不是同步的，而是一个异步操作。调用commitAsync()之后，它会立即返回，不会阻塞，因此不会影响Consumer应用的TPS。由于它是异步的，Kafka提供了回调函数（callback），供你实现提交之后的逻辑，比如记录日志或处理异常等。下面这段代码展示了调用commitAsync()的方法：</p><pre><code>while (true) {
            ConsumerRecords&lt;String, String&gt; records = 
	consumer.poll(Duration.ofSeconds(1));
            process(records); // 处理消息
            consumer.commitAsync((offsets, exception) -&gt; {
	if (exception != null)
	handle(exception);
	});
}
</code></pre><p>commitAsync是否能够替代commitSync呢？答案是不能。commitAsync的问题在于，出现问题时它不会自动重试。因为它是异步操作，倘若提交失败后自动重试，那么它重试时提交的位移值可能早已经“过期”或不是最新值了。因此，异步提交的重试其实没有意义，所以commitAsync是不会重试的。</p><p>显然，如果是手动提交，我们需要将commitSync和commitAsync组合使用才能达到最理想的效果，原因有两个：</p><ol>
<li>我们可以利用commitSync的自动重试来规避那些瞬时错误，比如网络的瞬时抖动，Broker端GC等。因为这些问题都是短暂的，自动重试通常都会成功，因此，我们不想自己重试，而是希望Kafka Consumer帮我们做这件事。</li>
<li>我们不希望程序总处于阻塞状态，影响TPS。</li>
</ol><p>我们来看一下下面这段代码，它展示的是如何将两个API方法结合使用进行手动提交。</p><pre><code>   try {
           while(true) {
                        ConsumerRecords&lt;String, String&gt; records = 
                                    consumer.poll(Duration.ofSeconds(1));
                        process(records); // 处理消息
                        commitAysnc(); // 使用异步提交规避阻塞
            }
} catch(Exception e) {
            handle(e); // 处理异常
} finally {
            try {
                        consumer.commitSync(); // 最后一次提交使用同步阻塞式提交
	} finally {
	     consumer.close();
}
}
</code></pre><p>这段代码同时使用了commitSync()和commitAsync()。对于常规性、阶段性的手动提交，我们调用commitAsync()避免程序阻塞，而在Consumer要关闭前，我们调用commitSync()方法执行同步阻塞式的位移提交，以确保Consumer关闭前能够保存正确的位移数据。将两者结合后，我们既实现了异步无阻塞式的位移管理，也确保了Consumer位移的正确性，所以，如果你需要自行编写代码开发一套Kafka Consumer应用，那么我推荐你使用上面的代码范例来实现手动的位移提交。</p><p>我们说了自动提交和手动提交，也说了同步提交和异步提交，这些就是Kafka位移提交的全部了吗？其实，我们还差一部分。</p><p>实际上，Kafka Consumer API还提供了一组更为方便的方法，可以帮助你实现更精细化的位移管理功能。刚刚我们聊到的所有位移提交，都是提交poll方法返回的所有消息的位移，比如poll方法一次返回了500条消息，当你处理完这500条消息之后，前面我们提到的各种方法会一次性地将这500条消息的位移一并处理。简单来说，就是<strong>直接提交最新一条消息的位移</strong>。但如果我想更加细粒度化地提交位移，该怎么办呢？</p><p>设想这样一个场景：你的poll方法返回的不是500条消息，而是5000条。那么，你肯定不想把这5000条消息都处理完之后再提交位移，因为一旦中间出现差错，之前处理的全部都要重来一遍。这类似于我们数据库中的事务处理。很多时候，我们希望将一个大事务分割成若干个小事务分别提交，这能够有效减少错误恢复的时间。</p><p>在Kafka中也是相同的道理。对于一次要处理很多消息的Consumer而言，它会关心社区有没有方法允许它在消费的中间进行位移提交。比如前面这个5000条消息的例子，你可能希望每处理完100条消息就提交一次位移，这样能够避免大批量的消息重新消费。</p><p>庆幸的是，Kafka Consumer API为手动提交提供了这样的方法：commitSync(Map&lt;TopicPartition, OffsetAndMetadata&gt;)和commitAsync(Map&lt;TopicPartition, OffsetAndMetadata&gt;)。它们的参数是一个Map对象，键就是TopicPartition，即消费的分区，而值是一个OffsetAndMetadata对象，保存的主要是位移数据。</p><p>就拿刚刚提过的那个例子来说，如何每处理100条消息就提交一次位移呢？在这里，我以commitAsync为例，展示一段代码，实际上，commitSync的调用方法和它是一模一样的。</p><pre><code>private Map&lt;TopicPartition, OffsetAndMetadata&gt; offsets = new HashMap&lt;&gt;();
int count = 0;
……
while (true) {
            ConsumerRecords&lt;String, String&gt; records = 
	consumer.poll(Duration.ofSeconds(1));
            for (ConsumerRecord&lt;String, String&gt; record: records) {
                        process(record);  // 处理消息
                        offsets.put(new TopicPartition(record.topic(), record.partition()),
                                   new OffsetAndMetadata(record.offset() + 1)；
                       if（count % 100 == 0）
                                    consumer.commitAsync(offsets, null); // 回调处理逻辑是null
                        count++;
	}
}
</code></pre><p>简单解释一下这段代码。程序先是创建了一个Map对象，用于保存Consumer消费处理过程中要提交的分区位移，之后开始逐条处理消息，并构造要提交的位移值。还记得之前我说过要提交下一条消息的位移吗？这就是这里构造OffsetAndMetadata对象时，使用当前消息位移加1的原因。代码的最后部分是做位移的提交。我在这里设置了一个计数器，每累计100条消息就统一提交一次位移。与调用无参的commitAsync不同，这里调用了带Map对象参数的commitAsync进行细粒度的位移提交。这样，这段代码就能够实现每处理100条消息就提交一次位移，不用再受poll方法返回的消息总数的限制了。</p><h2>小结</h2><p>好了，我们来总结一下今天的内容。Kafka Consumer的位移提交，是实现Consumer端语义保障的重要手段。位移提交分为自动提交和手动提交，而手动提交又分为同步提交和异步提交。在实际使用过程中，推荐你使用手动提交机制，因为它更加可控，也更加灵活。另外，建议你同时采用同步提交和异步提交两种方式，这样既不影响TPS，又支持自动重试，改善Consumer应用的高可用性。总之，Kafka Consumer API提供了多种灵活的提交方法，方便你根据自己的业务场景定制你的提交策略。</p><p><img src="https://static001.geekbang.org/resource/image/a6/d1/a6e24c364321aaa44b8fedf3836bccd1.jpg?wh=2069*2569" alt=""></p><h2>开放讨论</h2><p>实际上，手动提交也不能避免消息重复消费。假设Consumer在处理完消息和提交位移前出现故障，下次重启后依然会出现消息重复消费的情况。请你思考一下，如何实现你的业务场景中的去重逻辑呢？</p><p>欢迎写下你的思考和答案，我们一起讨论。如果你觉得有所收获，也欢迎把文章分享给你的朋友。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/d5/db/c45b90c8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>水天一色</span>
  </div>
  <div class="_2_QraFYR_0">消费者提了异步 commit 实际还没更新完offset，消费者再不断地poll，其实会有重复消费的情况吧？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 只要consumer没有重启，不会发生重复消费。因为在运行过程中consumer会记录已获取的消息位移</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-07 09:53:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c4/92/338b5609.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Roy Liang</span>
  </div>
  <div class="_2_QraFYR_0">要彻底避免消息重复消费，这样是否可行？在consumer端进行幂等操作。这样kafka就可以设置自动提交位移了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 一直以来，在业务端实现去重或幂等都是避免消费的不二法则。单纯依赖Kafka避免重复消费很难做到~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-16 11:04:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/1e/3a/5b21c01c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>nightmare</span>
  </div>
  <div class="_2_QraFYR_0">老师手动提交的设计很优美，先用异步提交不影响程序的性能，再用consumer关闭时同步提交来确保位移一定提交成功。这里我有个疑问，比如我程序运行期间有多次异步提交没有成功，比如101的offset和201的offset没有提交成功，程序关闭的时候501的offset提交成功了，是不是就代表前面500条我还是消费成功了，只要最新的位移提交成功，就代表之前的消息都提交成功了？第二点 就是批量提交哪里，如果一个消费者晓得多个分区的消息，封装在一个Map对象里面消费者也能正确的对多个分区的位移都保证正确的提交吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-13 01:13:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/c7/dc/9408c8c2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ban</span>
  </div>
  <div class="_2_QraFYR_0">老师，你好。有个场景不太明白。我做个假设，比如说我的模式是自动提交，自动提交间隔是20秒一次，那我消费了10个消息，很快一秒内就结束。但是这时候我自动提交时间还没到（那是不是意味着不会提交offer），然后这时候我又去poll获取消息，会不会导致一直获取上一批的消息？<br><br>还是说如果consumer消费完了，自动提交时间还没到，如果你去poll，这时候会自动提交，就不会出现重复消费的情况。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不会的。consumer内部维护了一个指针，能够探测到下一条要消费的数据</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-13 23:11:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLEYbNElGIxY6Le1rfiakWJecz8JIOp06Y9JQFR2YBn3T3gx3icI5CKxZNgxgqiaKbfVOicXquO3QBw9w/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>july</span>
  </div>
  <div class="_2_QraFYR_0">老师你好，这里是否可以理解为 自动提交逻辑是在poll方法中，如果间隔大于最小提交间隔，就会运行逻辑进行offset提交，如果小于最小间隔，则忽略offset提交逻辑？也就是说上次poll 的数据即便处理结束，没有调用下一次poll，那么offset也不会提交？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 基本上是这样。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-15 11:28:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/cd/2a/bdbed6ed.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>无菇朋友</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，有一个疑问，为什么poll之前的提交和按频率自动提交是一个时机，假如频率是5s提交一次，某两次poll之间的间隔是6s，这时候是怎么处理提交的？忘老师解答下，着实没想通这个地方</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯， 严格来说。提交频率指的是最小的提交间隔。比如设置5s，Kafka保证至少等待5s才会自动提交一次。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-21 12:58:56</div>
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
  <div class="_2_QraFYR_0">1 概念区分<br>	A ：Consumer端的位移概念和消息分区的位移概念不是一回事。<br>	B ：Consumer的消费位移，记录的是Consumer要消费的下一条消息的位移。<br><br>2 提交位移<br>	A ：Consumer 要向Kafka汇报自己的位移数据，这个汇报过程被称为提交位移（Committing Offsets）。<br>	B ：Consumer需要为分配给它的每个分区提交各自的位移数据。<br><br>3提交位移的作用<br>	A ：提交位移主要是为了表征Consumer的消费进度，这样当Consumer发生故障重启后，能够从kafka中读取之前提交的位移值，从相应的位置继续消费，避免从头在消费一遍。<br><br>4 位移提交的特点<br>	A ：位移提交的语义保障是由你来负责的，Kafka只会“无脑”地接受你提交的位移。位移提交错误，就会消息消费错误。<br><br>5 位移提交方式<br>	A ：从用户的角度讲，位移提交分为自动提交和手动提交；从Consumer端的角度而言，位移提交分为同步提交和异步提交。<br><br>	B ：自动提交：由Kafka consumer在后台默默的执行提交位移，用户不用管。开启简单，使用方便，但可能会出现重复消费。<br><br>	C ：手动提交：好处在更加灵活，完全能够把控位移提交的时机和频率。<br>		（1）同步提交：在调用commitSync()时，Consumer程序会处于阻塞状态，直到远端Broker返回提交结果，这个状态才会结束。对TPS影响显著<br>		（2）异步提交：在调用commitAsync()时，会立即给响应，但是出问题了它不会自动重试。<br>		（3）手动提交最好是同步和异步结合使用，正常用异步提交，如果异步提交失败，用同步提交方式补偿提交。<br>	<br>	D ：批次提交：对于一次要处理很多消费的Consumer而言，将一个大事务分割成若干个小事务分别提交。这可以有效减少错误恢复的时间，避免大批量的消息重新消费。<br>		（1）使用commitSync（Map&lt;TopicPartition，Offset&gt;）和commitAsync(Map&lt;TopicPartition，OffsetAndMetadata&gt;)。<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-05 12:46:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/4d/5e/c5c62933.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lmtoo</span>
  </div>
  <div class="_2_QraFYR_0">对于手动同步和异步提交结合的场景，如果poll出来的消息是500条，而业务处理200条的时候，业务抛异常了，后续消息根本就没有被遍历过，finally里手动同步提交的是201还是000，还是501？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果调用没有参数的commit，那么提交的是500</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-13 10:10:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/d0/42/6fd01fb9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我已经设置了昵称</span>
  </div>
  <div class="_2_QraFYR_0">auto.commit.interval.ms为5秒，且为自动提交<br>如果业务5秒内还没处理完，这个客户端怎么处理offset</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个参数其实有点误导。它其实的意思是至少5秒。可能多于5秒</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-20 16:40:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/d1/22/706c492e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Algoric</span>
  </div>
  <div class="_2_QraFYR_0">自动提交一定不会消息丢失吗，如果每次poll的数据过多，在提交时间内没有处理完，这时达到提交时间，那么Kafka还是重复提交上次poll的最大位移吗，还是讲本次poll的消息最大位移提交？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: hmmm... 其实我一直觉得提交间隔这个参数的命名有些问题。它实际保证的是位移至少要隔一段时间才会提交，如果你是单线程处理消息，那么只有处理完消息后才会提交位移，可能远比你设置的间隔长。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-25 14:59:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/b3/c5/7fc124e2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Liam</span>
  </div>
  <div class="_2_QraFYR_0">所以自动提交有2个时机吗？<br><br>1 固定频率提及，例如5s提及一次<br>2 poll新数据之前提交前面消费的数据</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 它们实际上是一个时机</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-15 08:59:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/aa/ff/e2c331e0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>bbbi</span>
  </div>
  <div class="_2_QraFYR_0">老师您好！有一个问题时。Kafka的offset是一个数字，那么这个数值最大时多少？有没有可能存在用完的情况？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: offset是long型的，几乎不可能用完。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-14 23:35:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/37/d0/26975fba.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>aof</span>
  </div>
  <div class="_2_QraFYR_0">先说一下，课后思考，解决的办法应该就是，将消息处理和位移提交放在一个事务里面，要么都成功，要么都失败。<br><br>老师文章里面的举的一个例子没有很明白，能不能再解释一下。就是那个位移提交后Rebalance的例子。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-13 23:52:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/48/bd/6c7d4230.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Tony Du</span>
  </div>
  <div class="_2_QraFYR_0">老师，您好～ 看了今天的教程我有两个问题想请教下，希望老师能赐教。<br>1. 从文中的代码看上去，使用commitAsync提供offset，不需要等待异步执行结果再次poll就能拿到下一批消息，是那么这个offset的最新值是不是理解为其实是在consumer client的内存中管理的（因为实际commitAsync如果失败的话，offset不会写入broker中）？如果是这样的话，如果在执行到commitSync之前，consumer client进程重启了，就有可能会消费到因commitAsync失败产生的重复消息。<br>2. 教程中手动提交100条消息的代码是一个同步处理的代码，我在实际工程中遇到的问题是，为了提高消息处理效率，coumser poll到一批消息后会提交到一个thread pool中处理，这种情况下，请教下怎样做到控制offset分批提交？<br>谢谢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-13 11:26:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/46/a9/70fa676f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Luke</span>
  </div>
  <div class="_2_QraFYR_0">我的理解，不管怎样做，单靠Kafka无法保证消息不被重复消费，无论时候自动提交还是手动提交，同步提交还是异步提交，消息的下游消费都要做去重和幂等处理。除非能够保证消息的消费和位点的提交是一个原子操作。而这个原子性太难保证了，基本上又要引入分布式一致性的那一套东西了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 同意。事实上，Kafka事务对producer端的重复生产消息解决很好，但对消费者端避免重复消费保证依然是弱的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-28 17:53:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/22/c1/3ba7deca.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>猫哭</span>
  </div>
  <div class="_2_QraFYR_0">业务端来保证；业务端重复消费是否是幂等的？如果是幂等，对业务无影响，重复消费没关系；如果不是幂等，在生产者给kafka发送消息的时候，给每条消息生成一个唯一ID，消费端，视业务场景而言，分两种情况：1.如果是敏感业务，如与钱相关，在数据库建一张消费消息的流水表，每次消费前到数据库去查询一下，看是否消费过，消费过就忽略，没有消费过就消费。2.非敏感业务，可以存到redis，消费前去redis查询，该条消息是否消费过</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-17 13:51:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/05/7f/d35ab9a1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>z.l</span>
  </div>
  <div class="_2_QraFYR_0">try {<br>        while (true) {<br>            ConsumerRecords&lt;String, String&gt; records = <br>                        consumer.poll(Duration.ofSeconds(1));<br>            process(records); &#47;&#47; 处理消息<br>            commitAysnc(); &#47;&#47; 使用异步提交规避阻塞<br>        }<br>     } catch (Exception e) {<br>        handle(e); &#47;&#47; 处理异常<br>    } finally {<br>        try {<br>            consumer.commitSync(); &#47;&#47; 最后一次提交使用同步阻塞式提交<br>        } finally {<br>            consumer.close();<br>        }<br>    }<br>这段代码如果异常了，不就退出while循环了么？也就相当于消费者线程异常退出？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的。这取决于你对异常的处理态度。如果你觉得处理异常后还能继续消费，也可以将try-catch放入while内</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-16 13:20:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/80/bb/c0ed9d76.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>kursk.ye</span>
  </div>
  <div class="_2_QraFYR_0">我现在有点糊涂了，kafka的offset是以broker发消息给consumer时，broker的offset为准；还是以consumer 的commit offset为准？比如，一个partition现在的offset是99，执行poll(10)方法时，broker给consumer发送了10条记录，在broker中offset变为109；假如 enable.auto.commit 为false，为手动提交consumer offset,但是cosumer在执行consumer.commitSync()或consumer.commitAsync()时进程失败，整个consumer进程都崩溃了；于是一个新的consumer接替原consumer继续消费，那么他是从99开始消费，还是从109开始消费？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 首先，poll(10)不是获取10条消息的意思。<br>其次，consumer获取的位移是它之前最新一次提交的位移，因此是99</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-14 11:46:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/64/9b/d1ab239e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>J.Smile</span>
  </div>
  <div class="_2_QraFYR_0">老师，我真的遇到了无论是自动提交或者是手动提交，没有报错，但是消费位移就是没有增长，心塞塞！！！求助🆘</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不要通过Kafka manager查，用kafka-consumer-groups命令查</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-21 21:50:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/PiajxSqBRaEIaTvOKvUt4WnuSjkBp0tjd6O6vvVyw5fcib3UgZibE8tz2ICbTfkwbzs8MHNMJjV6W2mLjywLsvBibg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>火力全开</span>
  </div>
  <div class="_2_QraFYR_0">请问老师Kafka能配置成每次连接只消费最新的消息吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以的，使用Consumer的seekToEnd</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-01 13:16:10</div>
  </div>
</div>
</div>
</li>
</ul>