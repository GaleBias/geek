<audio title="27 _ 条件变量sync.Cond （上）" src="https://static001.geekbang.org/resource/audio/ec/48/ece047da1672d0fe5ea940d15ffd5048.mp3" controls="controls"></audio> 
<p>在上篇文章中，我们主要说的是互斥锁，今天我和你来聊一聊条件变量（conditional variable）。</p><h2>前导内容：条件变量与互斥锁</h2><p>我们常常会把条件变量这个同步工具拿来与互斥锁一起讨论。实际上，条件变量是基于互斥锁的，它必须有互斥锁的支撑才能发挥作用。</p><p>条件变量并不是被用来保护临界区和共享资源的，它是用于协调想要访问共享资源的那些线程的。当共享资源的状态发生变化时，它可以被用来通知被互斥锁阻塞的线程。</p><p>比如说，我们两个人在共同执行一项秘密任务，这需要在不直接联系和见面的前提下进行。我需要向一个信箱里放置情报，你需要从这个信箱中获取情报。这个信箱就相当于一个共享资源，而我们就分别是进行写操作的线程和进行读操作的线程。</p><p>如果我在放置的时候发现信箱里还有未被取走的情报，那就不再放置，而先返回。另一方面，如果你在获取的时候发现信箱里没有情报，那也只能先回去了。这就相当于写的线程或读的线程阻塞的情况。</p><p>虽然我们俩都有信箱的钥匙，但是同一时刻只能有一个人插入钥匙并打开信箱，这就是锁的作用了。更何况咱们俩是不能直接见面的，所以这个信箱本身就可以被视为一个临界区。</p><p>尽管没有协调好，咱们俩仍然要想方设法的完成任务啊。所以，如果信箱里有情报，而你却迟迟未取走，那我就需要每过一段时间带着新情报去检查一次，若发现信箱空了，我就需要及时地把新情报放到里面。</p><!-- [[[read_end]]] --><p>另一方面，如果信箱里一直没有情报，那你也要每过一段时间去打开看看，一旦有了情报就及时地取走。这么做是可以的，但就是太危险了，很容易被敌人发现。</p><p>后来，我们又想了一个计策，各自雇佣了一个不起眼的小孩儿。如果早上七点有一个戴红色帽子的小孩儿从你家楼下路过，那么就意味着信箱里有了新情报。另一边，如果上午九点有一个戴蓝色帽子的小孩儿从我家楼下路过，那就说明你已经从信箱中取走了情报。</p><p>这样一来，咱们执行任务的隐蔽性高多了，并且效率的提升非常显著。这两个戴不同颜色帽子的小孩儿就相当于条件变量，在共享资源的状态产生变化的时候，起到了通知的作用。</p><p>当然了，我们是在用Go语言编写程序，而不是在执行什么秘密任务。因此，条件变量在这里的最大优势就是在效率方面的提升。当共享资源的状态不满足条件的时候，想操作它的线程再也不用循环往复地做检查了，只要等待通知就好了。</p><p>说到这里，想考考你知道怎么使用条件变量吗？所以，<strong>我们今天的问题就是：条件变量怎样与互斥锁配合使用？</strong></p><p><strong>这道题的典型回答是：条件变量的初始化离不开互斥锁，并且它的方法有的也是基于互斥锁的。</strong></p><p>条件变量提供的方法有三个：等待通知（wait）、单发通知（signal）和广播通知（broadcast）。</p><p>我们在利用条件变量等待通知的时候，需要在它基于的那个互斥锁保护下进行。而在进行单发通知或广播通知的时候，却是恰恰相反的，也就是说，需要在对应的互斥锁解锁之后再做这两种操作。</p><h2>问题解析</h2><p>这个问题看起来很简单，但其实可以基于它,延伸出很多其他的问题。比如，每个方法的使用时机是什么？又比如，每个方法执行的内部流程是怎样的？</p><p>下面，我们一边用代码实现前面那个例子，一边讨论条件变量的使用。</p><p>首先，我们先来创建如下几个变量。</p><pre><code>var mailbox uint8
var lock sync.RWMutex
sendCond := sync.NewCond(&amp;lock)
recvCond := sync.NewCond(lock.RLocker())
</code></pre><p><strong>变量<code>mailbox</code>代表信箱，是<code>uint8</code>类型的。</strong> 若它的值为<code>0</code>则表示信箱中没有情报，而当它的值为<code>1</code>时则说明信箱中有情报。<code>lock</code>是一个类型为<code>sync.RWMutex</code>的变量，是一个读写锁，也可以被视为信箱上的那把锁。</p><p>另外，基于这把锁，我还创建了两个代表条件变量的变量，<strong>名字分别叫<code>sendCond</code>和<code>recvCond</code>。</strong> 它们都是<code>*sync.Cond</code>类型的，同时也都是由<code>sync.NewCond</code>函数来初始化的。</p><p>与<code>sync.Mutex</code>类型和<code>sync.RWMutex</code>类型不同，<code>sync.Cond</code>类型并不是开箱即用的。我们只能利用<code>sync.NewCond</code>函数创建它的指针值。这个函数需要一个<code>sync.Locker</code>类型的参数值。</p><p>还记得吗？我在前面说过，条件变量是基于互斥锁的，它必须有互斥锁的支撑才能够起作用。因此，这里的参数值是不可或缺的，它会参与到条件变量的方法实现当中。</p><p><code>sync.Locker</code>其实是一个接口，在它的声明中只包含了两个方法定义，即：<code>Lock()</code>和<code>Unlock()</code>。<code>sync.Mutex</code>类型和<code>sync.RWMutex</code>类型都拥有<code>Lock</code>方法和<code>Unlock</code>方法，只不过它们都是指针方法。因此，这两个类型的指针类型才是<code>sync.Locker</code>接口的实现类型。</p><p>我在为<code>sendCond</code>变量做初始化的时候，把基于<code>lock</code>变量的指针值传给了<code>sync.NewCond</code>函数。</p><p>原因是，<strong><code>lock</code>变量的<code>Lock</code>方法和<code>Unlock</code>方法分别用于对其中写锁的锁定和解锁，它们与<code>sendCond</code>变量的含义是对应的。</strong><code>sendCond</code>是专门为放置情报而准备的条件变量，向信箱里放置情报，可以被视为对共享资源的写操作。</p><p>相应的，<strong><code>recvCond</code>变量代表的是专门为获取情报而准备的条件变量。</strong> 虽然获取情报也会涉及对信箱状态的改变，但是好在做这件事的人只会有你一个，而且我们也需要借此了解一下，条件变量与读写锁中的读锁的联用方式。所以，在这里，我们暂且把获取情报看做是对共享资源的读操作。</p><p>因此，为了初始化<code>recvCond</code>这个条件变量，我们需要的是<code>lock</code>变量中的读锁，并且还需要是<code>sync.Locker</code>类型的。</p><p>可是，<code>lock</code>变量中用于对读锁进行锁定和解锁的方法却是<code>RLock</code>和<code>RUnlock</code>，它们与<code>sync.Locker</code>接口中定义的方法并不匹配。</p><p>好在<code>sync.RWMutex</code>类型的<code>RLocker</code>方法可以实现这一需求。我们只要在调用<code>sync.NewCond</code>函数时，传入调用表达式<code>lock.RLocker()</code>的结果值，就可以使该函数返回符合要求的条件变量了。</p><p>为什么说通过<code>lock.RLocker()</code>得来的值就是<code>lock</code>变量中的读锁呢？实际上，这个值所拥有的<code>Lock</code>方法和<code>Unlock</code>方法，在其内部会分别调用<code>lock</code>变量的<code>RLock</code>方法和<code>RUnlock</code>方法。也就是说，前两个方法仅仅是后两个方法的代理而已。</p><p>好了，我们现在有四个变量。一个是代表信箱的<code>mailbox</code>，一个是代表信箱上的锁的<code>lock</code>。还有两个是，代表了蓝帽子小孩儿的<code>sendCond</code>，以及代表了红帽子小孩儿的<code>recvCond</code>。</p><p><img src="https://static001.geekbang.org/resource/image/36/5d/3619456ade9d45a4d9c0fbd22bb6fd5d.png?wh=1627*829" alt=""></p><p>（互斥锁与条件变量）</p><p>我，现在是一个goroutine（携带的<code>go</code>函数），想要适时地向信箱里放置情报并通知你，应该怎么做呢？</p><pre><code>lock.Lock()
for mailbox == 1 {
 sendCond.Wait()
}
mailbox = 1
lock.Unlock()
recvCond.Signal()
</code></pre><p>我肯定需要先调用<code>lock</code>变量的<code>Lock</code>方法。注意，这个<code>Lock</code>方法在这里意味的是：持有信箱上的锁，并且有打开信箱的权利，而不是锁上这个锁。</p><p>然后，我要检查<code>mailbox</code>变量的值是否等于<code>1</code>，也就是说，要看看信箱里是不是还存有情报。如果还有情报，那么我就回家去等蓝帽子小孩儿了。</p><p>这就是那条<code>for</code>语句以及其中的调用表达式<code>sendCond.Wait()</code>所表示的含义了。你可能会问，为什么这里是<code>for</code>语句而不是<code>if</code>语句呢？我在后面会对此进行解释的。</p><p>我们再往后看，如果信箱里没有情报，那么我就把新情报放进去，关上信箱、锁上锁，然后离开。用代码表达出来就是<code>mailbox = 1</code>和<code>lock.Unlock()</code>。</p><p>离开之后我还要做一件事，那就是让红帽子小孩儿准时去你家楼下路过。也就是说，我会及时地通知你“信箱里已经有新情报了”，我们调用<code>recvCond</code>的<code>Signal</code>方法就可以实现这一步骤。</p><p>另一方面，你现在是另一个goroutine，想要适时地从信箱中获取情报，然后通知我。</p><pre><code>lock.RLock()
for mailbox == 0 {
 recvCond.Wait()
}
mailbox = 0
lock.RUnlock()
sendCond.Signal()
</code></pre><p>你跟我做的事情在流程上其实基本一致，只不过每一步操作的对象是不同的。你需要调用的是<code>lock</code>变量的<code>RLock</code>方法。因为你要进行的是读操作，并且会使用<code>recvCond</code>变量作为辅助。<code>recvCond</code>与<code>lock</code>变量的读锁是对应的。</p><p>在打开信箱后，你要关注的是信箱里是不是没有情报，也就是检查<code>mailbox</code>变量的值是否等于<code>0</code>。如果它确实等于<code>0</code>，那么你就需要回家去等红帽子小孩儿，也就是调用<code>recvCond</code>的<code>Wait</code>方法。这里使用的依然是<code>for</code>语句。</p><p>如果信箱里有情报，那么你就应该取走情报，关上信箱、锁上锁，然后离开。对应的代码是<code>mailbox = 0</code>和<code>lock.RUnlock()</code>。之后，你还需要让蓝帽子小孩儿准时去我家楼下路过。这样我就知道信箱中的情报已经被你获取了。</p><p>以上这些，就是对咱们俩要执行秘密任务的代码实现。其中的条件变量的用法需要你特别注意。</p><p>再强调一下，只要条件不满足，我就会通过调用<code>sendCond</code>变量的<code>Wait</code>方法，去等待你的通知，只有在收到通知之后我才会再次检查信箱。</p><p>另外，当我需要通知你的时候，我会调用<code>recvCond</code>变量的<code>Signal</code>方法。你使用这两个条件变量的方式正好与我相反。你可能也看出来了，利用条件变量可以实现单向的通知，而双向的通知则需要两个条件变量。这也是条件变量的基本使用规则。</p><p>你可以打开demo61.go文件，看到上述例子的全部实现代码。</p><h2>总结</h2><p>我们这两期的文章会围绕条件变量的内容展开，条件变量是基于互斥锁的一种同步工具，它必须有互斥锁的支撑才能发挥作用。  条件变量可以协调那些想要访问共享资源的线程。当共享资源的状态发生变化时，它可以被用来通知被互斥锁阻塞的线程。我在文章举了一个两人访问信箱的例子，并用代码实现了这个过程。</p><h2>思考题</h2><p><code>*sync.Cond</code>类型的值可以被传递吗？那<code>sync.Cond</code>类型的值呢？</p><p>感谢你的收听，我们下期再见。</p><p><a href="https://github.com/hyper0x/Golang_Puzzlers">戳此查看Go语言专栏文章配套详细代码。</a></p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/54/ba/8721e403.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>hello peter</span>
  </div>
  <div class="_2_QraFYR_0">老师, 感觉这个送信的例子似乎用chanel实现更简单.在网上也查了一些例子, 发现都可以用chanel替代. 那使用sync.Cond 的优势是什么呢, 或者有哪些独特的使用场景?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 优势是并发流程上的协同，chan的主要任务是传递数据。另外cond是更低层次的工具，效率更高一些，但是肯定没有chan方便。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-26 12:01:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/59/c0/6f02c096.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>属雨</span>
  </div>
  <div class="_2_QraFYR_0">个人理解，不确定对不对，请老师评判一下：<br>因为Go语言传递对象时，使用的是浅拷贝的值传递，所以，当传递一个Cond对象时复制了这个Cond对象，但是低层保存的L(Locker类型)，noCopy(noCopy类型)，notify(notifyList类型)，checker(copyChecker)对象的指针没变，因此，*sync.Cond和sync.Cond都可以传递。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 基本正确。Locker是接口，是引用类型，nocopy是结构体，所以直接拷贝值的话，底层锁还是用的同一个，使用上容易出问题。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-12 21:15:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/df/1e/cea897e8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>传说中的成大大</span>
  </div>
  <div class="_2_QraFYR_0">这几天一直对条件变量的理解比较模糊，但是我想既然要学就学好 于是又去翻了Unix环境高级编程 总算把它跟互斥锁区分开了<br>互斥锁 是对一个共享区域进行加锁 所有线程都是一种竞争的状态去访问<br>而条件变量 主要是通过条件状态来判断，实际上他还是会阻塞 只不过不会像互斥锁一样去参与竞争，而是在哪里等待条件变量的状态发生改变过后的通知 再被唤醒</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-09 17:18:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKdiaUiaCYQe9tibemaNU5ya7RrU3MYcSGEIG7zF27u0ZDnZs5lYxPb7KPrAsj3bibM79QIOnPXAatfIw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_a8be59</span>
  </div>
  <div class="_2_QraFYR_0">var mailbox uint8<br>var lock sync.RWMutex<br>sendCond := sync.NewCond(&amp;lock)<br>recvCond := sync.NewCond(&amp;lock)<br>为什么不能向上面那样都用同一个互斥量，非要两个不同呢？老师，能讲一下区别么<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果都用同一个互斥量的话，操作双方就无法独立行事，这就是完全串行的操作了，效率上会大打折扣。<br><br>进一步说，本来就是一个发一个收，理应一个用写锁一个用读锁，这样效率高，之后扩展起来也方便。因为读之间不用互斥。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-12 18:04:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c6/73/abb7bfe3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>疯琴</span>
  </div>
  <div class="_2_QraFYR_0">秘密接头的类比形象生动，学习过程轻松有趣又不失深度。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-10 09:10:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/d4/e0/513d185e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>文@雨路</span>
  </div>
  <div class="_2_QraFYR_0">指针可以传递，值不可以，传递值会拷贝一份，导致出现两份条件变量，彼此之间没有联系</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-12 09:10:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/ea/80/8759e4c1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>🐻</span>
  </div>
  <div class="_2_QraFYR_0">需传递 *sync.Cond<br><br>因为 Cond 结构体中的 notify 变量和 checker 变量都是值类型，传递sync.Cond 会复制值，这样两个锁保留的被阻塞的 Goroutine 就不同了。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-09 22:14:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://wx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLmBgic9UlGySyG377pCVzNnbgsGttrKTCFztunJlBTDS32oTyHsJjAFJJsYJyhk9cNE5OZeGKWJ6Q/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>beiliu</span>
  </div>
  <div class="_2_QraFYR_0">您好，老师，官方文档是建议，singal在锁住的情况下使用的“Signal唤醒等待c的一个线程（如果存在）。调用者在调用本方法时，建议（但并非必须）保持c.L的锁定“</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-31 17:04:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/e8/58/ecb493dc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ming</span>
  </div>
  <div class="_2_QraFYR_0">多routine从信箱中获取情报, 都在等mailbox变量的值不为0的时候再把它的值变为0，<br>这个 RLock 限制不了写操作，可能会有多个routine同时将 mailbox 变为0的，跟文中的场景有些不合。 <br>不知道我理解的有没有问题<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-17 18:14:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/d9/d2/4ae9b17f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>q</span>
  </div>
  <div class="_2_QraFYR_0">个人感觉，从底层原理解释条件变量可能更容易理解</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-03 08:20:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/0a/0b/985d3800.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>郭星</span>
  </div>
  <div class="_2_QraFYR_0">在for 循环中使用 wait,在我的测试中,当条件变量处于wait状态时,如果没有唤醒,当前协程会一直阻塞等待在wait这行代码,因此使用for 和 使用if 实际最终结果是相同的,为什么要使用for呢?<br><br>package lesson27<br><br>import (<br>	&quot;sync&quot;<br>	&quot;testing&quot;<br>	&quot;time&quot;<br>)<br><br>&#47;&#47; 利用条件变量实现协调多协程发取信件操作<br>func TestCond(t *testing.T) {<br>	var wg sync.WaitGroup<br>	var mu sync.RWMutex<br>	&#47;&#47; 信箱<br>	mail := false<br>	&#47;&#47; 两个条件变量<br>	&#47;&#47; 发送信条件变量<br>	sendCond := sync.NewCond(&amp;mu)<br>	&#47;&#47; 接收信条件变量, 对于接收实际是只读操作,因此只需要使用读锁就可以<br>	receiveCond := sync.NewCond(mu.RLocker())<br>	&#47;&#47; 最大发送接收次数<br>	max := 5<br>	wg.Add(2)<br>	&#47;&#47; 发送人协程<br>	go func(i int) {<br>		for ; i &gt; 0; i-- {<br>			time.Sleep(time.Second * 3)<br>			mu.Lock()<br>			&#47;&#47; 如果信箱不为空,则需要等待<br>			&#47;&#47;for mail {<br>			if mail {<br>				&#47;&#47; 发送者等待<br>				t.Log(&quot;sendCond准备进入等待队列&quot;)<br>				sendCond.Wait()<br>				t.Log(&quot;sendCond进入等待队列&quot;)<br>			}<br>			mail = true<br>			t.Log(&quot;发送信件成功&quot;)<br>			mu.Unlock()<br>			&#47;&#47; 通知发送者<br>			receiveCond.Signal()<br>			t.Log(&quot;唤醒receiveCond&quot;)<br>		}<br>		wg.Done()<br>	}(max)<br><br>	go func(i int) {<br>		for ; i &gt; 0; i-- {<br>			mu.RLock()<br>			&#47;&#47;for !mail {<br>			if !mail {<br>				&#47;&#47;接收者等待<br>				t.Log(&quot;receiveCond准备进入等待队列&quot;)<br>				receiveCond.Wait() &#47;&#47; 如果没有被唤醒会一直阻塞在此<br>				t.Log(&quot;receiveCond进入等待队列&quot;)<br>			}<br>			mail = false<br>			t.Log(&quot;获取信件成功&quot;)<br>			mu.RUnlock()<br>			&#47;&#47; 通知接收者<br>			sendCond.Signal()<br>			t.Log(&quot;唤醒sendCond&quot;)<br>		}<br>		wg.Done()<br>	}(max)<br>	wg.Wait()<br>}<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我在文章里说了啊，有可能会碰到“假唤醒”的情况。而且，如果存在“有多个wait但只需唤醒一个”的情况，也需要用for语句。在for语句里，唤醒后可以再次检查状态，如果状态符合就开始后续工作，如果不符合就再次wait。用if语句就办不到。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-02 11:02:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/38/e2/28aa8e6c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>会玩code</span>
  </div>
  <div class="_2_QraFYR_0">老师，不懂这里的recvCond为什么可以用读锁呢？这里也是有对资源做操作的呀（将mailbox置为0），用读锁不会有问题吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 在只有一个“取信者”的情况下，这里使用读锁是没问题的。但如果有多个“取信者”就需要用写锁啦。你可以看一看下一篇文章对应的demo文件（demo62.go），其中就有多个“取信者”的演示。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-15 10:35:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2e/16/ed/acd6df5e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>啦啦啦</span>
  </div>
  <div class="_2_QraFYR_0">想请问下老师，两个goroutine都使用了同一把锁，26讲（Mutex）里不是说明，尽量使用：是让每一个互斥锁都只保护一个临界区或一组相关临界区。有点搞不明白，望老师指点<br><br><br><br>go func(max int) { &#47;&#47; 用于发信。<br>		defer func() {<br>			sign &lt;- struct{}{}<br>		}()<br>		for i := 1; i &lt;= max; i++ {<br>			time.Sleep(time.Millisecond * 500)<br>			lock.Lock()<br>			for mailbox == 1 {<br>				sendCond.Wait()<br>			}<br>			log.Printf(&quot;sender [%d]: the mailbox is empty.&quot;, i)<br>			mailbox = 1<br>			log.Printf(&quot;sender [%d]: the letter has been sent.&quot;, i)<br>			lock.Unlock()<br>			recvCond.Signal()<br>		}<br>	}(max)<br>	go func(max int) { &#47;&#47; 用于收信。<br>		defer func() {<br>			sign &lt;- struct{}{}<br>		}()<br>		for j := 1; j &lt;= max; j++ {<br>			time.Sleep(time.Millisecond * 500)<br>			lock.RLock()<br>			for mailbox == 0 {<br>				recvCond.Wait()<br>			}<br>			log.Printf(&quot;receiver [%d]: the mailbox is full.&quot;, j)<br>			mailbox = 0<br>			log.Printf(&quot;receiver [%d]: the letter has been received.&quot;, j)<br>			lock.RUnlock()<br>			sendCond.Signal()<br>		}<br>	}(max)</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: “一组相关临界区”可以是操作同一个共享资源的多段代码，也可以是在逻辑上有强相关关系的多段代码。在这里，这两段代码都是针对 mailbox 变量的，属于前者。只不过其中还夹杂了 不同条件变量 的 Wait 操作。<br><br>我总体再说一下吧，我们可以从两个维度来理解互斥锁的保护作用：<br><br>1. 多个goroutine并发执行同一段代码（如一个函数或方法），且这段代码访问或修改了共享的状态或资源。这时候，我们需要使用互斥锁保护这段代码。这相当于保护了那个共享状态或资源。这里的“这段代码”就是一个临界区。<br><br>2. （如本例）多个goroutine中的代码（确切地说，是go函数）直接访问或修改了共享的状态或资源。这时候，我们往往就需要使用同一个互斥锁同时去保护那几段相关的代码。这里的几段代码各自成为独立的临界区，但它们是相关的。这就是我说的“一组相关的临界区”。<br><br>以上这两个维度说的就是互斥锁的常用场景。一个是“单段代码的并发执行”，一个是“多段代码的各自执行（也是并发执行）”。关键是，不管是单段代码还是多段代码，它们都分别访问了同一个共享的东西。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-07 10:48:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/9d/a4/e481ae48.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lesserror</span>
  </div>
  <div class="_2_QraFYR_0">郝林老师，demo61.go 中的  两个go function（收信 和 发信），是怎么保证先 发信 后收信的呢？<br><br>不是说 go function 函数 的执行 是 随机的么？ 我打印了很多遍，发现 都是执行的 发信 操作，然后是 收信 操作。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 正因为有了条件变量才会有这样的同步状态啊。条件变量就相当于信号弹的发射器。<br><br>双方各有一个信号弹发射器（就像示例程序中的 sendCond 和 recvCond）并互相发送信号，这就是 demo61 所做的。<br><br><br>这个程序你也可以这样理解：<br><br>A 和 B 在下象棋。步骤如下：<br><br>A 走了一步棋，并发送信号示意让 B 走一步；<br>B 收到信号走了一步棋，然后反过来向 A 发送信号示意“我走了一步，该你了”；<br>A 再走一步棋，并再次发射信号示意让 B 再走一步；<br>B 再走一步后，再发射信号告诉 A；<br>如此循环往复。<br><br><br>再做一个类比：<br><br>单个条件变量用于“教官”指挥“士兵”，“教官”只能有一个，而“士兵”可以有多个（demo61 中只有一个士兵）。在 demo61 中，我们又多加了一个用于“反馈”的信号弹发射器。这样一来，“士兵”按照指挥行进一步之后，就可以及时告诉指挥官并请求下一步的行动了。<br><br><br><br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-15 21:38:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1c/f7/62/947004d0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>www</span>
  </div>
  <div class="_2_QraFYR_0">这篇讲解太棒了，看了lock.RLocker()的源码，又返回去看了前面第14篇讲解接口的文章。多看多写，逐步提升</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-28 11:08:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/2b/39/19041d78.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>😳</span>
  </div>
  <div class="_2_QraFYR_0">这个类比生动形象，很有趣，简单易懂</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-22 17:21:22</div>
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
  <div class="_2_QraFYR_0">recvCond.Signal()表示接收方可以接受了，发送这个信号的是发送方。<br>sendCond.Signal()表示发送方可以再次发送了，发送这个信号的是接收方。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-27 16:37:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/aa/21/6c3ba9af.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lfn</span>
  </div>
  <div class="_2_QraFYR_0">打卡，2020-01-16.</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-16 18:21:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/5b/66/ad35bc68.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>党</span>
  </div>
  <div class="_2_QraFYR_0">demo60太经典了 不仅体会到了锁的用法 还体会到了 如何利用chan 阻塞主线程 以使goroutine完全跑完 算是chan经典用法吧</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-29 11:24:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/6d/52/404912c3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>...</span>
  </div>
  <div class="_2_QraFYR_0">老师 wait会释放锁吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 每次执行结束前都会释放，要不其他goroutine没法进入锁保护的临界区。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-20 17:53:30</div>
  </div>
</div>
</div>
</li>
</ul>