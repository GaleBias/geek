<audio title="20 _ 详解时钟轮在RPC中的应用" src="https://static001.geekbang.org/resource/audio/3f/10/3fdb8cab3e7a7d89da65b02945c02410.mp3" controls="controls"></audio> 
<p>你好，我是何小锋。上一讲我们学习了在分布式环境下如何快速定位问题，简单回顾下重点。在分布式环境下，RPC框架自身以及服务提供方的业务逻辑实现，都应该对异常进行合理地封装，让使用方可以根据异常快速地定位问题；而在依赖关系复杂且涉及多个部门合作的分布式系统中，我们也可以借助分布式链路跟踪系统，快速定位问题。</p><p>现在，切换到咱们今天的主题，一起看看时钟轮在RPC中的应用。</p><h2>定时任务带来了什么问题？</h2><p>在讲解时钟轮之前，我们先来聊聊定时任务。相信你在开发的过程中，很多场景都会使用到定时任务，在RPC框架中也有很多地方会使用到它。就以调用端请求超时的处理逻辑为例，下面我们看一下RPC框架是如果处理超时请求的。</p><p>回顾下<a href="https://time.geekbang.org/column/article/216803">[第 17 讲]</a>，我讲解Future的时候说过：无论是同步调用还是异步调用，调用端内部实行的都是异步，而调用端在向服务端发送消息之前会创建一个Future，并存储这个消息标识与这个Future的映射，当服务端收到消息并且处理完毕后向调用端发送响应消息，调用端在接收到消息后会根据消息的唯一标识找到这个Future，并将结果注入给这个Future。</p><p>那在这个过程中，如果服务端没有及时响应消息给调用端呢？调用端该如何处理超时的请求？</p><!-- [[[read_end]]] --><p>没错，就是可以利用定时任务。每次创建一个Future，我们都记录这个Future的创建时间与这个Future的超时时间，并且有一个定时任务进行检测，当这个Future到达超时时间并且没有被处理时，我们就对这个Future执行超时逻辑。</p><p><strong>那定时任务该如何实现呢？</strong></p><p>有种实现方式是这样的，也是最简单的一种。每创建一个Future我们都启动一个线程，之后sleep，到达超时时间就触发请求超时的处理逻辑。</p><p>这种方式吧，确实简单，在某些场景下也是可以使用的，但弊端也是显而易见的。就像刚才我讲的那个Future超时处理的例子，如果我们面临的是高并发的请求，单机每秒发送数万次请求，请求超时时间设置的是5秒，那我们要创建多少个线程用来执行超时任务呢？超过10万个线程，这个数字真的够吓人了。</p><p>别急，我们还有另一种实现方式。我们可以用一个线程来处理所有的定时任务，还以刚才那个Future超时处理的例子为例。假设我们要启动一个线程，这个线程每隔100毫秒会扫描一遍所有的处理Future超时的任务，当发现一个Future超时了，我们就执行这个任务，对这个Future执行超时逻辑。</p><p>这种方式我们用得最多，它也解决了第一种方式线程过多的问题，但其实它也有明显的弊端。</p><p>同样是高并发的请求，那么扫描任务的线程每隔100毫秒要扫描多少个定时任务呢？如果调用端刚好在1秒内发送了1万次请求，这1万次请求要在5秒后才会超时，那么那个扫描的线程在这个5秒内就会不停地对这1万个任务进行扫描遍历，要额外扫描40多次（每100毫秒扫描一次，5秒内要扫描近50次），很浪费CPU。</p><p>在我们使用定时任务时，它所带来的问题，就是让CPU做了很多额外的轮询遍历操作，浪费了CPU，这种现象在定时任务非常多的情况下，尤其明显。</p><h2>什么是时钟轮？</h2><p>这个问题也不难解决，我们只要找到一种方式，减少额外的扫描操作就行了。比如我的一批定时任务是5秒之后执行，我在4.9秒之后才开始扫描这批定时任务，这样就大大地节省了CPU。这时我们就可以利用时钟轮的机制了。</p><p>我们先来看下我们生活中用到的时钟。</p><p><img src="https://static001.geekbang.org/resource/image/d6/cf/d6366d3aec1af5f1b028b5939a7753cf.jpg?wh=3169*1390" alt="" title="时钟示意图"></p><p>很熟悉了吧，时钟有时针、分针和秒针，秒针跳动一周之后，也就是跳动60个刻度之后，分针跳动1次，分针跳动60个刻度，时针走动一步。</p><p>而时钟轮的实现原理就是参考了生活中的时钟跳动的原理。</p><p><img src="https://static001.geekbang.org/resource/image/b6/5a/b61d5a5d1ec29384ba8730b71b4beb5a.jpg?wh=3181*1364" alt="" title="时钟轮示意图"></p><p>在时钟轮机制中，有时间槽和时钟轮的概念，时间槽就相当于时钟的刻度，而时钟轮就相当于秒针与分针等跳动的一个周期，我们会将每个任务放到对应的时间槽位上。</p><p>时钟轮的运行机制和生活中的时钟也是一样的，每隔固定的单位时间，就会从一个时间槽位跳到下一个时间槽位，这就相当于我们的秒针跳动了一次；时钟轮可以分为多层，下一层时钟轮中每个槽位的单位时间是当前时间轮整个周期的时间，这就相当于1分钟等于60秒钟；当时钟轮将一个周期的所有槽位都跳动完之后，就会从下一层时钟轮中取出一个槽位的任务，重新分布到当前的时钟轮中，当前时钟轮则从第0槽位从新开始跳动，这就相当于下一分钟的第1秒。</p><p>为了方便你了解时钟轮的运行机制，我们用一个场景例子来模拟下，一起看下这个场景。</p><p>假设我们的时钟轮有10个槽位，而时钟轮一轮的周期是1秒，那么我们每个槽位的单位时间就是100毫秒，而下一层时间轮的周期就是10秒，每个槽位的单位时间也就是1秒，并且当前的时钟轮刚初始化完成，也就是第0跳，当前在第0个槽位。</p><p><img src="https://static001.geekbang.org/resource/image/a6/e9/a661c85b35508dea9cd9db6969f58de9.jpg?wh=3189*1366" alt="" title="时钟轮示意图"></p><p>好，现在我们有3个任务，分别是任务A（90毫秒之后执行）、任务B（610毫秒之后执行）与任务C（1秒610毫秒之后执行），我们将这3个任务添加到时钟轮中，任务A被放到第0槽位，任务B被放到第6槽位，任务C被放到下一层时间轮的第1槽位，如下面这张图所示。</p><p><img src="https://static001.geekbang.org/resource/image/6d/de/6da0e7a3dc3d327f7f7b7a3cf7f658de.jpg?wh=3172*1362" alt="" title="时钟轮任务分布示意图"></p><p>当任务A刚被放到时钟轮，就被即刻执行了，因为它被放到了第0槽位，而当前时间轮正好跳到第0槽位（实际上还没开始跳动，状态为第0跳）；600毫秒之后，时间轮已经进行了6跳，当前槽位是第6槽位，第6槽位所有的任务都被取出执行；1秒钟之后，当前时钟轮的第9跳已经跳完，从新开始了第0跳，这时下一层时钟轮从第0跳跳到了第1跳，将第1槽位的任务取出，分布到当前的时钟轮中，这时任务C从下一层时钟轮中取出并放到当前时钟轮的第6槽位；1秒600毫秒之后，任务C被执行。</p><p><img src="https://static001.geekbang.org/resource/image/92/11/9224564f0bad64e130061e0bb1e07411.jpg?wh=6172*1589" alt="" title="任务C槽位转换示意图"></p><p>看完了这个场景，相信你对时钟轮的机制已经有所了解了。在这个例子中，时钟轮的扫描周期仍是100毫秒，但是其中的任务并没有被过多的重复扫描，它完美地解决了CPU浪费的问题。</p><p>这个机制其实不难理解，但实现起来还是很有难度的，其中要注意的问题也很多。具体的代码实现我们这里不展示，这又是另外一个比较大的话题了。有兴趣的话你可以自行查阅下相关源码，动手实现一下。到哪里卡住了，我们可以在留言区交流。</p><h2>时钟轮在RPC中的应用</h2><p>通过刚才对时钟轮的讲解，相信你可以看出，它就是用来执行定时任务的，可以说在RPC框架中只要涉及到定时相关的操作，我们就可以使用时钟轮。</p><p>那么RPC框架在哪些功能实现中会用到它呢？</p><p>刚才我举例讲到的调用端请求超时处理，这里我们就可以应用到时钟轮，我们每发一次请求，都创建一个处理请求超时的定时任务放到时钟轮里，在高并发、高访问量的情况下，时钟轮每次只轮询一个时间槽位中的任务，这样会节省大量的CPU。</p><p>调用端与服务端启动超时也可以应用到时钟轮，以调用端为例，假设我们想要让应用可以快速地部署，例如1分钟内启动，如果超过1分钟则启动失败。我们可以在调用端启动时创建一个处理启动超时的定时任务，放到时钟轮里。</p><p>除此之外，你还能想到RPC框架在哪些地方可以应用到时钟轮吗？还有定时心跳。RPC框架调用端定时向服务端发送心跳，来维护连接状态，我们可以将心跳的逻辑封装为一个心跳任务，放到时钟轮里。</p><p>这时你可能会有一个疑问，心跳是要定时重复执行的，而时钟轮中的任务执行一遍就被移除了，对于这种需要重复执行的定时任务我们该如何处理呢？在定时任务的执行逻辑的最后，我们可以重设这个任务的执行时间，把它重新丢回到时钟轮里。</p><h2>总结</h2><p>今天我们主要讲解了时钟轮的机制，以及时钟轮在RPC框架中的应用。</p><p>这个机制很好地解决了定时任务中，因每个任务都创建一个线程，导致的创建过多线程的问题，以及一个线程扫描所有的定时任务，让CPU做了很多额外的轮询遍历操作而浪费CPU的问题。</p><p>时钟轮的实现机制就是模拟现实生活中的时钟，将每个定时任务放到对应的时间槽位上，这样可以减少扫描任务时对其它时间槽位定时任务的额外遍历操作。</p><p>在时间轮的使用中，有些问题需要你额外注意：</p><ul>
<li>时间槽位的单位时间越短，时间轮触发任务的时间就越精确。例如时间槽位的单位时间是10毫秒，那么执行定时任务的时间误差就在10毫秒内，如果是100毫秒，那么误差就在100毫秒内。</li>
<li>时间轮的槽位越多，那么一个任务被重复扫描的概率就越小，因为只有在多层时钟轮中的任务才会被重复扫描。比如一个时间轮的槽位有1000个，一个槽位的单位时间是10毫秒，那么下一层时间轮的一个槽位的单位时间就是10秒，超过10秒的定时任务会被放到下一层时间轮中，也就是只有超过10秒的定时任务会被扫描遍历两次，但如果槽位是10个，那么超过100毫秒的任务，就会被扫描遍历两次。</li>
</ul><p>结合这些特点，我们就可以视具体的业务场景而定，对时钟轮的周期和时间槽数进行设置。</p><p>在RPC框架中，只要涉及到定时任务，我们都可以应用时钟轮，比较典型的就是调用端的超时处理、调用端与服务端的启动超时以及定时心跳等等。</p><h2>课后思考</h2><p>在RPC框架中，除了我说过的那几个例子，你还知道有哪些功能的实现可以应用到时钟轮？</p><p>欢迎留言和我分享你的答案，也欢迎你把文章分享给你的朋友，邀请他加入学习。我们下节课再见！</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/0f/bf/ee93c4cf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>雨霖铃声声慢</span>
  </div>
  <div class="_2_QraFYR_0">时钟轮可以实现延时消息的功能，比如让一个任务几分钟之后发送一条消息出去。在比如可以实现订单过期功能，用户下单10分钟没付款，就取消订单，可以通过时钟轮实现。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，真棒</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-06 10:57:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/82/3d/356fc3d6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>忆水寒</span>
  </div>
  <div class="_2_QraFYR_0">在java里面和Netty框架里面有这个类，TimeWheel时钟轮模型。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，netty里面有实现</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-06 09:29:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/70/67/0c1359c2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>qinsi</span>
  </div>
  <div class="_2_QraFYR_0">时钟轮存取任务的时间复杂度是O(1)，相比之下优先队列的时间复杂度是O(logN)</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-06 15:34:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/67/f4/9a1feb59.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>钱</span>
  </div>
  <div class="_2_QraFYR_0">定时任务相关解决方案：<br>1：线程休眠，可能需要N多线程<br>2：定时轮询，可能会空耗许多CPU轮询<br>3：时间轮，和时钟原理相似，规避1&#47;2的缺陷<br>只有涉及到定时任务，就可以使用时钟轮来解决，比如：<br>调用端超时处理、调用端和服务端的启动超时处理、定时心跳检测、延迟消息队列的延迟消息发送等。<br>netty框架中就有时钟轮的实现，可以研究一下。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-16 23:11:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/01/37/12e4c9c9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>高源</span>
  </div>
  <div class="_2_QraFYR_0">时钟轮这个我头一次听到，老师如果并发线程比较多，单位时间是不是划分很细啊，但是我有个疑问例如我同时有5个线程几乎之间间隔3到5毫秒，又有3个线程10到100毫秒的，我时钟轮也得调整具体怎么划分的，这么短时间内如何保证时钟轮准确性，老师哪有参考代码学习一下</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 一般都是统一划分，时间轮主要解决是长时间没有触发的问题，不解决实时性</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-06 06:42:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/67/12/51b78d88.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>🐠</span>
  </div>
  <div class="_2_QraFYR_0">请问老师，为什么说多层时间轮中的任务会被重复扫描呢？就上面例子的任务C来说，它处于下一层的第1个槽位，那么经过1s后，它被分布到当前的时间轮中，然后经过600ms后跳动到当前时间轮的第6个槽位，此时才会触发任务C。重复扫描的意思就是说在下一层的第1个槽位被扫描过一次，然后在当前时间轮的第6个槽位又被扫描过一次吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-17 09:57:11</div>
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
  <div class="_2_QraFYR_0">调用方的异常重试也可以考虑使用时间轮。感触：软件设计的本质都是生活~</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-05 13:33:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/e1/19/c756aaed.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>鸠摩智</span>
  </div>
  <div class="_2_QraFYR_0">redis中的key的超时是不是也用时钟轮实现的？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-11 08:28:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>hillwater</span>
  </div>
  <div class="_2_QraFYR_0">时间槽里面的任务用什么数据结构维护呢，hashmap？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-13 20:05:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/5a/4a/856ea384.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Hugo</span>
  </div>
  <div class="_2_QraFYR_0">文中提到，时间轮的槽位越多，那么一个任务被重复扫描的概率就越小，因为只有在多层时钟轮中的任务才会被重复扫描。<br>这里为什么会重复扫描呢<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-24 21:51:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/25/29/cf/db5be02a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>漆黑的小白</span>
  </div>
  <div class="_2_QraFYR_0">第一次听到时间轮还是在kafka里，拿来做一些轻量级的延时处理还挺不错的</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-07 17:58:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/67/8a/babd74dc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>锦</span>
  </div>
  <div class="_2_QraFYR_0">在实际开发中，单层时间轮和多层时间轮哪个用得多呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-21 09:12:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/06/7e/735968e2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>西门吹牛</span>
  </div>
  <div class="_2_QraFYR_0">定时心跳这个，可以用定时任务线程池来实现，底层是用延时队列来保存任务，普通的线程是队列中取出任务执行就完了，定时任务 ，可以从队列 取出任务，然后在重设时间，放回任务队列。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-08 18:54:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/c5/e0/7bbb6f3a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>司空摘星</span>
  </div>
  <div class="_2_QraFYR_0">为什么要自己实现时间轮，简单的超时回调不行吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-04 13:30:31</div>
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
  <div class="_2_QraFYR_0">貌似jdk的优先级队列也可以实现到期任务的优化扫描。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-10 09:26:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/3c/74/580d5bbb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lucas</span>
  </div>
  <div class="_2_QraFYR_0">技术的思想很多都是源于生活</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-31 15:00:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/30/c1/2dde6700.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>密码123456</span>
  </div>
  <div class="_2_QraFYR_0">时间轮，是不停的走动。怎么动态新添加？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-19 08:30:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/67/f4/9a1feb59.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>钱</span>
  </div>
  <div class="_2_QraFYR_0">头次听说这个东西，主要解决定时任务相关的不断轮询过多耗CPU的问题。优秀代码，真应该好好研究一下。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-16 22:41:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83eqMevH71ChH7gIJ18A79xFWnGicsbebuREjPYxzLMzwtX08icapU3hmsGpF1zZ2iayDt2ZoMiaic0PcG3g/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>719</span>
  </div>
  <div class="_2_QraFYR_0">提高扫描定时任务效率还有一种方法是所有定时任务放入按照时间排序的优先队列，每次只扫描队首节点。从性能上看，哪种方法好一些？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 应该是不同场景吧</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-06 08:59:13</div>
  </div>
</div>
</div>
</li>
</ul>