<audio title="25 _ RocketMQ与Kafka中如何实现事务？" src="https://static001.geekbang.org/resource/audio/c0/d1/c065d58825289cfdf89ee294cee0d3d1.mp3" controls="controls"></audio> 
<p>你好，我是李玥。</p><p>在之前《<a href="https://time.geekbang.org/column/article/111269">04 | 如何利用事务消息实现分布式事务？</a>》这节课中，我通过一个小例子来和大家讲解了如何来使用事务消息。在这节课的评论区，很多同学都提出来，非常想了解一下事务消息到底是怎么实现的。不仅要会使用，还要掌握实现原理，这种学习态度，一直是我们非常提倡的，这节课，我们就一起来学习一下，在RocketMQ和Kafka中，事务消息分别是如何来实现的？</p><h2>RocketMQ的事务是如何实现的？</h2><p>首先我们来看RocketMQ的事务。我在之前的课程中，已经给大家讲解过RocketMQ事务的大致流程，这里我们再一起通过代码，重温一下这个流程。</p><pre><code>public class CreateOrderService {

  @Inject
  private OrderDao orderDao; // 注入订单表的DAO
  @Inject
  private ExecutorService executorService; //注入一个ExecutorService

  private TransactionMQProducer producer;

  // 初始化transactionListener 和 producer
  @Init
  public void init() throws MQClientException {
    TransactionListener transactionListener = createTransactionListener();
    producer = new TransactionMQProducer(&quot;myGroup&quot;);
    producer.setExecutorService(executorService);
    producer.setTransactionListener(transactionListener);
    producer.start();
  }

  // 创建订单服务的请求入口
  @PUT
  @RequestMapping(...)
  public boolean createOrder(@RequestBody CreateOrderRequest request) {
    // 根据创建订单请求创建一条消息
    Message msg = createMessage(request);
    // 发送事务消息
    SendResult sendResult = producer.sendMessageInTransaction(msg, request);
    // 返回：事务是否成功
    return sendResult.getSendStatus() == SendStatus.SEND_OK;
  }

  private TransactionListener createTransactionListener() {
    return new TransactionListener() {
      @Override
      public LocalTransactionState executeLocalTransaction(Message msg, Object arg) {
        CreateOrderRequest request = (CreateOrderRequest ) arg;
        try {
          // 执行本地事务创建订单
          orderDao.createOrderInDB(request);
          // 如果没抛异常说明执行成功，提交事务消息
          return LocalTransactionState.COMMIT_MESSAGE;
        } catch (Throwable t) {
          // 失败则直接回滚事务消息
          return LocalTransactionState.ROLLBACK_MESSAGE;
        }
      }
      // 反查本地事务
      @Override
      public LocalTransactionState checkLocalTransaction(MessageExt msg) {、
        // 从消息中获得订单ID
        String orderId = msg.getUserProperty(&quot;orderId&quot;);

        // 去数据库中查询订单号是否存在，如果存在则提交事务；
        // 如果不存在，可能是本地事务失败了，也可能是本地事务还在执行，所以返回UNKNOW
        //（PS：这里RocketMQ有个拼写错误：UNKNOW）
        return orderDao.isOrderIdExistsInDB(orderId)?
                LocalTransactionState.COMMIT_MESSAGE: LocalTransactionState.UNKNOW;
      }
    };
  }

    //....
}
</code></pre><p>在这个流程中，我们提供一个创建订单的服务，功能就是在数据库中插入一条订单记录，并发送一条创建订单的消息，要求写数据库和发消息这两个操作在一个事务内执行，要么都成功，要么都失败。在这段代码中，我们首先在init()方法中初始化了transactionListener和发生RocketMQ事务消息的变量producer。真正提供创建订单服务的方法是createOrder()，在这个方法里面，我们根据请求的参数创建一条消息，然后调用RocketMQ producer发送事务消息，并返回事务执行结果。</p><!-- [[[read_end]]] --><p>之后的createTransactionListener()方法是在init()方法中调用的，这里面直接构造一个匿名类，来实现RocketMQ的TransactionListener接口，这个接口需要实现两个方法：</p><ul>
<li>executeLocalTransaction：执行本地事务，在这里我们直接把订单数据插入到数据库中，并返回本地事务的执行结果。</li>
<li>checkLocalTransaction：反查本地事务，在这里我们的处理是，在数据库中查询订单号是否存在，如果存在则提交事务，如果不存在，可能是本地事务失败了，也可能是本地事务还在执行，所以返回UNKNOW。</li>
</ul><p>这样，就使用RocketMQ的事务消息功能实现了一个创建订单的分布式事务。接下来我们一起通过RocketMQ的源代码来看一下，它的事务消息是如何实现的。</p><p>首先看一下在producer中，是如何来发送事务消息的：</p><pre><code>public TransactionSendResult sendMessageInTransaction(final Message msg,
                                                      final LocalTransactionExecuter localTransactionExecuter, final Object arg)
    throws MQClientException {
    TransactionListener transactionListener = getCheckListener();
    if (null == localTransactionExecuter &amp;&amp; null == transactionListener) {
        throw new MQClientException(&quot;tranExecutor is null&quot;, null);
    }
    Validators.checkMessage(msg, this.defaultMQProducer);

    SendResult sendResult = null;

    // 这里给消息添加了属性，标明这是一个事务消息，也就是半消息
    MessageAccessor.putProperty(msg, MessageConst.PROPERTY_TRANSACTION_PREPARED, &quot;true&quot;);
    MessageAccessor.putProperty(msg, MessageConst.PROPERTY_PRODUCER_GROUP, this.defaultMQProducer.getProducerGroup());

    // 调用发送普通消息的方法，发送这条半消息
    try {
        sendResult = this.send(msg);
    } catch (Exception e) {
        throw new MQClientException(&quot;send message Exception&quot;, e);
    }

    LocalTransactionState localTransactionState = LocalTransactionState.UNKNOW;
    Throwable localException = null;
    switch (sendResult.getSendStatus()) {
        case SEND_OK: {
            try {
                if (sendResult.getTransactionId() != null) {
                    msg.putUserProperty(&quot;__transactionId__&quot;, sendResult.getTransactionId());
                }
                String transactionId = msg.getProperty(MessageConst.PROPERTY_UNIQ_CLIENT_MESSAGE_ID_KEYIDX);
                if (null != transactionId &amp;&amp; !&quot;&quot;.equals(transactionId)) {
                    msg.setTransactionId(transactionId);
                }

                // 执行本地事务
                if (null != localTransactionExecuter) {
                    localTransactionState = localTransactionExecuter.executeLocalTransactionBranch(msg, arg);
                } else if (transactionListener != null) {
                    log.debug(&quot;Used new transaction API&quot;);
                    localTransactionState = transactionListener.executeLocalTransaction(msg, arg);
                }
                if (null == localTransactionState) {
                    localTransactionState = LocalTransactionState.UNKNOW;
                }

                if (localTransactionState != LocalTransactionState.COMMIT_MESSAGE) {
                    log.info(&quot;executeLocalTransactionBranch return {}&quot;, localTransactionState);
                    log.info(msg.toString());
                }
            } catch (Throwable e) {
                log.info(&quot;executeLocalTransactionBranch exception&quot;, e);
                log.info(msg.toString());
                localException = e;
            }
        }
        break;
        case FLUSH_DISK_TIMEOUT:
        case FLUSH_SLAVE_TIMEOUT:
        case SLAVE_NOT_AVAILABLE:
            localTransactionState = LocalTransactionState.ROLLBACK_MESSAGE;
            break;
        default:
            break;
    }

    // 根据事务消息和本地事务的执行结果localTransactionState，决定提交或回滚事务消息
    // 这里给Broker发送提交或回滚事务的RPC请求。
    try {
        this.endTransaction(sendResult, localTransactionState, localException);
    } catch (Exception e) {
        log.warn(&quot;local transaction execute &quot; + localTransactionState + &quot;, but end broker transaction failed&quot;, e);
    }

    TransactionSendResult transactionSendResult = new TransactionSendResult();
    transactionSendResult.setSendStatus(sendResult.getSendStatus());
    transactionSendResult.setMessageQueue(sendResult.getMessageQueue());
    transactionSendResult.setMsgId(sendResult.getMsgId());
    transactionSendResult.setQueueOffset(sendResult.getQueueOffset());
    transactionSendResult.setTransactionId(sendResult.getTransactionId());
    transactionSendResult.setLocalTransactionState(localTransactionState);
    return transactionSendResult;
}
</code></pre><p>这段代码的实现逻辑是这样的：首先给待发送消息添加了一个属性PROPERTY_TRANSACTION_PREPARED，标明这是一个事务消息，也就是半消息，然后会像发送普通消息一样去把这条消息发送到Broker上。如果发送成功了，就开始调用我们之前提供的接口TransactionListener的实现类中，执行本地事务的方法executeLocalTransaction()来执行本地事务，在我们的例子中就是在数据库中插入一条订单记录。</p><p>最后，根据半消息发送的结果和本地事务执行的结果，来决定提交或者回滚事务。在实现方法endTransaction()中，producer就是给Broker发送了一个单向的RPC请求，告知Broker完成事务的提交或者回滚。由于有事务反查的机制来兜底，这个RPC请求即使失败或者丢失，也都不会影响事务最终的结果。最后构建事务消息的发送结果，并返回。</p><p>以上，就是RocketMQ在Producer这一端事务消息的实现，然后我们再看一下Broker这一端，它是怎么来处理事务消息和进行事务反查的。</p><p>Broker在处理Producer发送消息的请求时，会根据消息中的属性判断一下，这条消息是普通消息还是半消息：</p><pre><code>// ...
if (traFlag != null &amp;&amp; Boolean.parseBoolean(traFlag)) {
    // ...
    putMessageResult = this.brokerController.getTransactionalMessageService().prepareMessage(msgInner);
} else {
    putMessageResult = this.brokerController.getMessageStore().putMessage(msgInner);
}
// ...
</code></pre><p>这段代码在org.apache.rocketmq.broker.processor.SendMessageProcessor#sendMessage方法中，然后我们跟进去看看真正处理半消息的业务逻辑，这段处理逻辑在类org.apache.rocketmq.broker.transaction.queue.TransactionalMessageBridge中：</p><pre><code>public PutMessageResult putHalfMessage(MessageExtBrokerInner messageInner) {
    return store.putMessage(parseHalfMessageInner(messageInner));
}

private MessageExtBrokerInner parseHalfMessageInner(MessageExtBrokerInner msgInner) {

    // 记录消息的主题和队列，到新的属性中
    MessageAccessor.putProperty(msgInner, MessageConst.PROPERTY_REAL_TOPIC, msgInner.getTopic());
    MessageAccessor.putProperty(msgInner, MessageConst.PROPERTY_REAL_QUEUE_ID,
        String.valueOf(msgInner.getQueueId()));
    msgInner.setSysFlag(
        MessageSysFlag.resetTransactionValue(msgInner.getSysFlag(), MessageSysFlag.TRANSACTION_NOT_TYPE));
    // 替换消息的主题和队列为：RMQ_SYS_TRANS_HALF_TOPIC，0
    msgInner.setTopic(TransactionalMessageUtil.buildHalfTopic());
    msgInner.setQueueId(0);
    msgInner.setPropertiesString(MessageDecoder.messageProperties2String(msgInner.getProperties()));
    return msgInner;
}
</code></pre><p>我们可以看到，在这段代码中，RocketMQ并没有把半消息保存到消息中客户端指定的那个队列中，而是记录了原始的主题队列后，把这个半消息保存在了一个特殊的内部主题RMQ_SYS_TRANS_HALF_TOPIC中，使用的队列号固定为0。这个主题和队列对消费者是不可见的，所以里面的消息永远不会被消费。这样，就保证了在事务提交成功之前，这个半消息对消费者来说是消费不到的。</p><p>然后我们再看一下，RocketMQ是如何进行事务反查的：在Broker的TransactionalMessageCheckService服务中启动了一个定时器，定时从半消息队列中读出所有待反查的半消息，针对每个需要反查的半消息，Broker会给对应的Producer发一个要求执行事务状态反查的RPC请求，这部分的逻辑在方法org.apache.rocketmq.broker.transaction.AbstractTransactionalMessageCheckListener#sendCheckMessage中，根据RPC返回响应中的反查结果，来决定这个半消息是需要提交还是回滚，或者后续继续来反查。</p><p>最后，提交或者回滚事务实现的逻辑是差不多的，首先把半消息标记为已处理，如果是提交事务，那就把半消息从半消息队列中复制到这个消息真正的主题和队列中去，如果要回滚事务，这一步什么都不需要做，最后结束这个事务。这部分逻辑的实现在org.apache.rocketmq.broker.processor.EndTransactionProcessor这个类中。</p><h2>Kafka的事务和Exactly Once可以解决什么问题？</h2><p>接下来我们再说一下Kafka的事务。之前我们讲事务的时候说过，Kafka的事务解决的问题和RocketMQ是不太一样的。RocketMQ中的事务，它解决的问题是，确保执行本地事务和发消息这两个操作，要么都成功，要么都失败。并且，RocketMQ增加了一个事务反查的机制，来尽量提高事务执行的成功率和数据一致性。</p><p>而Kafka中的事务，它解决的问题是，确保在一个事务中发送的多条消息，要么都成功，要么都失败。注意，这里面的多条消息不一定要在同一个主题和分区中，可以是发往多个主题和分区的消息。当然，你可以在Kafka的事务执行过程中，加入本地事务，来实现和RocketMQ中事务类似的效果，但是Kafka是没有事务反查机制的。</p><p>Kafka的这种事务机制，单独来使用的场景不多。更多的情况下被用来配合Kafka的幂等机制来实现Kafka 的Exactly Once语义。我在之前的课程中也强调过，这里面的Exactly Once，和我们通常理解的消息队列的服务水平中的Exactly Once是不一样的。</p><p>我们通常理解消息队列的服务水平中的Exactly Once，它指的是，消息从生产者发送到Broker，然后消费者再从Broker拉取消息，然后进行消费。这个过程中，确保每一条消息恰好传输一次，不重不丢。我们之前说过，包括Kafka在内的几个常见的开源消息队列，都只能做到At Least Once，也就是至少一次，保证消息不丢，但有可能会重复。做不到Exactly Once。</p><p><img src="https://static001.geekbang.org/resource/image/ac/40/ac330e3e22b0114f5642491889510940.png?wh=354*91" alt=""></p><p>那Kafka中的Exactly Once又是解决的什么问题呢？它解决的是，在流计算中，用Kafka作为数据源，并且将计算结果保存到Kafka这种场景下，数据从Kafka的某个主题中消费，在计算集群中计算，再把计算结果保存在Kafka的其他主题中。这样的过程中，保证每条消息都被恰好计算一次，确保计算结果正确。</p><p><img src="https://static001.geekbang.org/resource/image/15/ff/15f8776de71b79cc232d8b60c3c24bff.png?wh=435*82" alt=""></p><p>举个例子，比如，我们把所有订单消息保存在一个Kafka的主题Order中，在Flink集群中运行一个计算任务，统计每分钟的订单收入，然后把结果保存在另一个Kafka的主题Income里面。要保证计算结果准确，就要确保，无论是Kafka集群还是Flink集群中任何节点发生故障，每条消息都只能被计算一次，不能重复计算，否则计算结果就错了。这里面有一个很重要的限制条件，就是数据必须来自Kafka并且计算结果都必须保存到Kafka中，才可以享受到Kafka的Excactly Once机制。</p><p>可以看到，Kafka的Exactly Once机制，是为了解决在“读数据-计算-保存结果”这样的计算过程中数据不重不丢，而不是我们通常理解的使用消息队列进行消息生产消费过程中的Exactly Once。</p><h2>Kafka的事务是如何实现的？</h2><p>那Kafka的事务又是怎么实现的呢？它的实现原理和RocketMQ的事务是差不多的，都是基于两阶段提交来实现的，但是实现的过程更加复杂。</p><p>首先说一下，参与Kafka事务的几个角色，或者说是模块。为了解决分布式事务问题，Kafka引入了事务协调者这个角色，负责在服务端协调整个事务。这个协调者并不是一个独立的进程，而是Broker进程的一部分，协调者和分区一样通过选举来保证自身的可用性。</p><p>和RocketMQ类似，Kafka集群中也有一个特殊的用于记录事务日志的主题，这个事务日志主题的实现和普通的主题是一样的，里面记录的数据就是类似于“开启事务”“提交事务”这样的事务日志。日志主题同样也包含了很多的分区。在Kafka集群中，可以存在多个协调者，每个协调者负责管理和使用事务日志中的几个分区。这样设计，其实就是为了能并行执行多个事务，提升性能。</p><p><img src="https://static001.geekbang.org/resource/image/c8/f4/c8c286a5e32ee324c8a32ba967d1f2f4.png?wh=1024*693" alt=""><br>
（图片来源：<a href="https://www.confluent.io/blog/transactions-apache-kafka/">Kafka官方</a>）</p><p>下面说一下Kafka事务的实现流程。</p><p>首先，当我们开启事务的时候，生产者会给协调者发一个请求来开启事务，协调者在事务日志中记录下事务ID。</p><p>然后，生产者在发送消息之前，还要给协调者发送请求，告知发送的消息属于哪个主题和分区，这个信息也会被协调者记录在事务日志中。接下来，生产者就可以像发送普通消息一样来发送事务消息，这里和RocketMQ不同的是，RocketMQ选择把未提交的事务消息保存在特殊的队列中，而Kafka在处理未提交的事务消息时，和普通消息是一样的，直接发给Broker，保存在这些消息对应的分区中，Kafka会在客户端的消费者中，暂时过滤未提交的事务消息。</p><p>消息发送完成后，生产者给协调者发送提交或回滚事务的请求，由协调者来开始两阶段提交，完成事务。第一阶段，协调者把事务的状态设置为“预提交”，并写入事务日志。到这里，实际上事务已经成功了，无论接下来发生什么情况，事务最终都会被提交。</p><p>之后便开始第二阶段，协调者在事务相关的所有分区中，都会写一条“事务结束”的特殊消息，当Kafka的消费者，也就是客户端，读到这个事务结束的特殊消息之后，它就可以把之前暂时过滤的那些未提交的事务消息，放行给业务代码进行消费了。最后，协调者记录最后一条事务日志，标识这个事务已经结束了。</p><p>我把整个事务的实现流程，绘制成一个简单的时序图放在这里，便于你理解。</p><p><img src="https://static001.geekbang.org/resource/image/27/fe/27742f00f70ead8c7194ef1aab503efe.png?wh=654*570" alt=""></p><p>总结一下Kafka这个两阶段的流程，准备阶段，生产者发消息给协调者开启事务，然后消息发送到每个分区上。提交阶段，生产者发消息给协调者提交事务，协调者给每个分区发一条“事务结束”的消息，完成分布式事务提交。</p><h2>小结</h2><p>这节课我分别讲解了Kafka和RocketMQ是如何来实现事务的。你可以看到，它们在实现事务过程中的一些共同的地方，它们都是基于两阶段提交来实现的事务，都利用了特殊的主题中的队列和分区来记录事务日志。</p><p>不同之处在于对处于事务中的消息的处理方式，RocketMQ是把这些消息暂存在一个特殊的队列中，待事务提交后再移动到业务队列中；而Kafka直接把消息放到对应的业务分区中，配合客户端过滤来暂时屏蔽进行中的事务消息。</p><p>同时你需要了解，RocketMQ和Kafka的事务，它们的适用场景是不一样的，RocketMQ的事务适用于解决本地事务和发消息的数据一致性问题，而Kafka的事务则是用于实现它的Exactly Once机制，应用于实时计算的场景中。</p><h2>思考题</h2><p>课后，请你根据我们课程中讲到的Kafka事务的实现流程，去Kafka的源代码中把这个事务的实现流程分析出来，将我们上面这个时序图进一步细化，绘制一个粒度到类和方法调用的时序图。然后请你想一下，如果事务进行过程中，协调者宕机了，那事务又是如何恢复的呢？欢迎你在评论区留言，写下你的想法。</p><p>感谢阅读，如果你觉得这篇文章对你有一些启发，也欢迎把它分享给你的朋友。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/6d/46/e16291f8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>丁小明</span>
  </div>
  <div class="_2_QraFYR_0">也就是说其实kafka的Exactly Once模式，是kafka的consumer通过PID去实现了一个幂等操作，原理上来说是和at last once我们业务自己通过其他唯一ID实现幂等是一样的效果,并不是正真的只传输到客户端一次，而是重复传输实现了幂等。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是这样的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-08 09:40:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/f9/31/b75cc6d5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ξ！</span>
  </div>
  <div class="_2_QraFYR_0">老师，如果本地事务是有返回值的话可不可以先执行本地事务如果有异常就抛出,再去执行发送消息,因为现在这么写获取不到执行完本地事务的结果呀</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 考虑这种情况：<br><br>1. 客户端提交了本地事务；<br>2. 本地事务在数据库执行成功了。<br>3. 这个时候客户端宕机了；<br><br>这种情况下，就没法保证一致性了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-13 14:04:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/48/71/bee9bca5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>二明儿</span>
  </div>
  <div class="_2_QraFYR_0">老师好，请教个问题，kafka consumer将事务未提交的消息 在客户端过滤后不放行给业务代码消费，如果这样如果有大量未提交的消息对于客户端端内存会不会有影响？如果这个时候客户端重启或者发生reblance，offset已经提交会不会导致消息丢失？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 大量未提交消息对客户端内存影响不大，因为Kafka客户端有一个固定大小的buffer用来保存拉取的消息。<br><br>只要你遵循：先执行消费业务逻辑，再提交，这样的原则。<br><br>即使客户端重启或者Rebalance，也不会丢消息。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-10 19:12:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/4e/f9/b12388bd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>weilai</span>
  </div>
  <div class="_2_QraFYR_0">查好像很多博客都说阿里已经把RocketMQ的这个反查接口给干掉了？老师，是这样吗？遇到这种问题，您是怎么找到答案的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 以官网文档和代码为准吧，至少目前的版本是没有变化的。<br><br>https:&#47;&#47;rocketmq.apache.org&#47;docs&#47;transaction-example&#47;</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-24 22:45:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/FheCgo4Ovibo0L1vAGgMdZkzQMm1GUMHMMqQ8aglufXaD2hW9z96DjQicAam723jOCZwXVmiaNiaaq4PLsf4COibZ5A/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>miniluo</span>
  </div>
  <div class="_2_QraFYR_0">老师，有个疑问：文中说到rocketmq#checkLocalTransaction这个方法反查到可能本地事务还在提交中就返回了unknow，那后续呢？还会通过定时轮询检查？求解，谢谢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 会一直定时轮询，直到有结果或者超时。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-21 08:19:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/6b/27/8c964e52.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>不惑ing</span>
  </div>
  <div class="_2_QraFYR_0">&#39; Kafka 的事务则是用于实现它的 Exactly Once 机制，应用于实时计算的场景中。&#39;这句话的意思理解为kafka的事务针对本地事务和发消息一致性没有rocketmq好，但是也可以用，这样理解对吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以这么理解，Kafka没有RocketMQ的事务反查补偿机制。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-05 13:23:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/9b/46/ad3194bd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jack</span>
  </div>
  <div class="_2_QraFYR_0">老师，如果仅仅把kafka作为数据源，流计算的结果保存到了其他数据库中，是不是就用不到kafka的事务了呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是这样的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-02 17:33:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/12/1b/f62722ca.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>A9</span>
  </div>
  <div class="_2_QraFYR_0">请问老师，失败的半消息也是在commit log中存储着吧。如果失败的事务消息存储过多，会不会导致在读取commit log时频繁触发缺页？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 一般来说不会，因为如果是已经关闭的事务，就不会再去读它对应的半消息了。<br><br>由于事务的超时机制存在，一般来说，活动的事务的日志大多都在commit log的尾部。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-27 17:31:46</div>
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
  <div class="_2_QraFYR_0">kafka的第二阶段，事务协调者发送给每个分区的事务结束的消息，每个分区是怎么处理这个事务结束的消息的？这个事务结束的消息保存到哪儿了？是不是消费者挂机重启之后，事务结束的消息就没了？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 事务结束消息就是一条特殊的消息，和普通消息一样保存在分区中。同普通消息一样，事务结束消息只要不被删除，就会一直存在。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-23 10:29:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/15/4e/4636a81d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jian</span>
  </div>
  <div class="_2_QraFYR_0">请问老师，这里说“消息发送完成后，生产者给协调者发送提交或回滚事务的请求，由协调者来开始两阶段提交，完成事务。第一阶段，协调者把事务的状态设置为“预提交”，并写入事务日志。到这里，实际上事务已经成功了，无论接下来发生什么情况，事务最终都会被提交。”假如协调者执行完第一阶段之后还没有执行第二阶段，这时候机器宕机或者进程被KILL掉了，是不是重启之后会继续执行第二阶段呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是这样的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-07 20:57:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/b2/08/92f42622.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>尔冬橙</span>
  </div>
  <div class="_2_QraFYR_0">orderId 属性是不是要在createMessageRequest的时候给设置进去</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-18 15:20:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/b2/08/92f42622.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>尔冬橙</span>
  </div>
  <div class="_2_QraFYR_0">createMessage(request)内容是啥</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-18 15:10:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/8e/62/435148c1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>SinKitwah</span>
  </div>
  <div class="_2_QraFYR_0">我可以理解为这个kafka的事务主要作用点是在计算集群到kafka b的那一块吗？是不是把kafka a换成rocket mq也能达到一样的事务效果呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-28 19:20:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/3a/6d/80c885ed.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Wheat Liu</span>
  </div>
  <div class="_2_QraFYR_0">最喜欢这节课，老师指明源码位置之后，rocketmq的逻辑简单易懂</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-03 17:59:46</div>
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
  <div class="_2_QraFYR_0">Kafka是将事务的控制全权交给了生产者,没有RMQ的反查稳定<br>至于,宕机后如何恢复事务,Kafka在每一步的操作之后都进行录入到了日志中,大可不必担心从日志中恢复事务</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-19 10:42:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/16/bd/e14ba493.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>翠羽香凝</span>
  </div>
  <div class="_2_QraFYR_0">二阶段提交，最大的问题是，第二阶段协调者发送了commit消息后，部分参与者成功，其他参与者失败，而同时协调者又宕机的情况。在这种情况下，失败的参与者会长期锁定资源无法释放，不知道kafka是如何解决这个问题的？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-22 18:53:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/78/31/c7f8d1db.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Laputa</span>
  </div>
  <div class="_2_QraFYR_0">文中提到的“Kafka 会在客户端的消费者中，暂时过滤未提交的事务消息。”，这里说的过滤具体是什么意思呢，一个分区里可以同时存在事务消息和非事务消息吗？如果可以同时存在，那这里的过滤会不会导致前面的事务消息没提交但后面的非事务消息被消费了？所以说文中提到的过滤是指客户端发现当前消费的消息是没提交的就一直卡在那直到提交或回滚吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-27 09:36:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83erbUHD8KH3mYz39QuHj5BXB06VqCus5WR4oHfNuSYdPVLno3Tq61yhMiaugxpMicKibDFQ4KwX7icoDWQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ipromiseu</span>
  </div>
  <div class="_2_QraFYR_0">请问老师，两种事务消息都只是保证【生产者发送消息，生产者完成业务逻辑，消费者收到消息】的原子性，如果想保证【生产者发送消息，生产者完成业务逻辑，消费者收到消息，消费者完成业务逻辑】的院子性呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-14 23:55:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/24/5d/65e61dcb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>学无涯</span>
  </div>
  <div class="_2_QraFYR_0">为什么用Kafka作为数据源，计算结果保存到其他数据库，就用不到事务了呢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-13 17:56:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/dc/08/64f5ab52.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>陈斌</span>
  </div>
  <div class="_2_QraFYR_0">Kafka事务的怎么使用呢，我这边刚好有一个topic A到 topic B的场景，我怎么利用Kafka的事务和幂等保证消息的 Exactly Once ?</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-03 17:25:30</div>
  </div>
</div>
</div>
</li>
</ul>