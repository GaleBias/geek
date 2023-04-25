<audio title="28 _ 条件变量sync.Cond （下）" src="https://static001.geekbang.org/resource/audio/dc/c1/dc69b4e352ee74948a122e27450197c1.mp3" controls="controls"></audio> 
<p>你好，我是郝林，今天我继续分享条件变量sync.Cond的内容。我们紧接着上一篇的内容进行知识扩展。</p><h2>问题 1：条件变量的<code>Wait</code>方法做了什么？</h2><p>在了解了条件变量的使用方式之后，你可能会有这么几个疑问。</p><ol>
<li>为什么先要锁定条件变量基于的互斥锁，才能调用它的<code>Wait</code>方法？</li>
<li>为什么要用<code>for</code>语句来包裹调用其<code>Wait</code>方法的表达式，用<code>if</code>语句不行吗？</li>
</ol><p>这些问题我在面试的时候也经常问。你需要对这个<code>Wait</code>方法的内部机制有所了解才能回答上来。</p><p>条件变量的<code>Wait</code>方法主要做了四件事。</p><ol>
<li>把调用它的goroutine（也就是当前的goroutine）加入到当前条件变量的通知队列中。</li>
<li>解锁当前的条件变量基于的那个互斥锁。</li>
<li>让当前的goroutine处于等待状态，等到通知到来时再决定是否唤醒它。此时，这个goroutine就会阻塞在调用这个<code>Wait</code>方法的那行代码上。</li>
<li>如果通知到来并且决定唤醒这个goroutine，那么就在唤醒它之后重新锁定当前条件变量基于的互斥锁。自此之后，当前的goroutine就会继续执行后面的代码了。</li>
</ol><p>你现在知道我刚刚说的第一个疑问的答案了吗？</p><p>因为条件变量的<code>Wait</code>方法在阻塞当前的goroutine之前，会解锁它基于的互斥锁，所以在调用该<code>Wait</code>方法之前，我们必须先锁定那个互斥锁，否则在调用这个<code>Wait</code>方法时，就会引发一个不可恢复的panic。</p><!-- [[[read_end]]] --><p>为什么条件变量的<code>Wait</code>方法要这么做呢？你可以想象一下，如果<code>Wait</code>方法在互斥锁已经锁定的情况下，阻塞了当前的goroutine，那么又由谁来解锁呢？别的goroutine吗？</p><p>先不说这违背了互斥锁的重要使用原则，即：成对的锁定和解锁，就算别的goroutine可以来解锁，那万一解锁重复了怎么办？由此引发的panic可是无法恢复的。</p><p>如果当前的goroutine无法解锁，别的goroutine也都不来解锁，那么又由谁来进入临界区，并改变共享资源的状态呢？只要共享资源的状态不变，即使当前的goroutine因收到通知而被唤醒，也依然会再次执行这个<code>Wait</code>方法，并再次被阻塞。</p><p>所以说，如果条件变量的<code>Wait</code>方法不先解锁互斥锁的话，那么就只会造成两种后果：不是当前的程序因panic而崩溃，就是相关的goroutine全面阻塞。</p><p>再解释第二个疑问。很显然，<code>if</code>语句只会对共享资源的状态检查一次，而<code>for</code>语句却可以做多次检查，直到这个状态改变为止。那为什么要做多次检查呢？</p><p><strong>这主要是为了保险起见。如果一个goroutine因收到通知而被唤醒，但却发现共享资源的状态，依然不符合它的要求，那么就应该再次调用条件变量的<code>Wait</code>方法，并继续等待下次通知的到来。</strong></p><p>这种情况是很有可能发生的，具体如下面所示。</p><ol>
<li>
<p>有多个goroutine在等待共享资源的同一种状态。比如，它们都在等<code>mailbox</code>变量的值不为<code>0</code>的时候再把它的值变为<code>0</code>，这就相当于有多个人在等着我向信箱里放置情报。虽然等待的goroutine有多个，但每次成功的goroutine却只可能有一个。别忘了，条件变量的<code>Wait</code>方法会在当前的goroutine醒来后先重新锁定那个互斥锁。在成功的goroutine最终解锁互斥锁之后，其他的goroutine会先后进入临界区，但它们会发现共享资源的状态依然不是它们想要的。这个时候，<code>for</code>循环就很有必要了。</p>
</li>
<li>
<p>共享资源可能有的状态不是两个，而是更多。比如，<code>mailbox</code>变量的可能值不只有<code>0</code>和<code>1</code>，还有<code>2</code>、<code>3</code>、<code>4</code>。这种情况下，由于状态在每次改变后的结果只可能有一个，所以，在设计合理的前提下，单一的结果一定不可能满足所有goroutine的条件。那些未被满足的goroutine显然还需要继续等待和检查。</p>
</li>
<li>
<p>有一种可能，共享资源的状态只有两个，并且每种状态都只有一个goroutine在关注，就像我们在主问题当中实现的那个例子那样。不过，即使是这样，使用<code>for</code>语句仍然是有必要的。原因是，在一些多CPU核心的计算机系统中，即使没有收到条件变量的通知，调用其<code>Wait</code>方法的goroutine也是有可能被唤醒的。这是由计算机硬件层面决定的，即使是操作系统（比如Linux）本身提供的条件变量也会如此。</p>
</li>
</ol><p>综上所述，在包裹条件变量的<code>Wait</code>方法的时候，我们总是应该使用<code>for</code>语句。</p><p>好了，到这里，关于条件变量的<code>Wait</code>方法，我想你知道的应该已经足够多了。</p><h2>问题 2：条件变量的<code>Signal</code>方法和<code>Broadcast</code>方法有哪些异同？</h2><p>条件变量的<code>Signal</code>方法和<code>Broadcast</code>方法都是被用来发送通知的，不同的是，前者的通知只会唤醒一个因此而等待的goroutine，而后者的通知却会唤醒所有为此等待的goroutine。</p><p>条件变量的<code>Wait</code>方法总会把当前的goroutine添加到通知队列的队尾，而它的<code>Signal</code>方法总会从通知队列的队首开始，查找可被唤醒的goroutine。所以，因<code>Signal</code>方法的通知，而被唤醒的goroutine一般都是最早等待的那一个。</p><p>这两个方法的行为决定了它们的适用场景。如果你确定只有一个goroutine在等待通知，或者只需唤醒任意一个goroutine就可以满足要求，那么使用条件变量的<code>Signal</code>方法就好了。</p><p>否则，使用<code>Broadcast</code>方法总没错，只要你设置好各个goroutine所期望的共享资源状态就可以了。</p><p>此外，再次强调一下，与<code>Wait</code>方法不同，条件变量的<code>Signal</code>方法和<code>Broadcast</code>方法并不需要在互斥锁的保护下执行。恰恰相反，我们最好在解锁条件变量基于的那个互斥锁之后，再去调用它的这两个方法。这更有利于程序的运行效率。</p><p>最后，请注意，条件变量的通知具有即时性。也就是说，如果发送通知的时候没有goroutine为此等待，那么该通知就会被直接丢弃。在这之后才开始等待的goroutine只可能被后面的通知唤醒。</p><p>你可以打开demo62.go文件，并仔细观察它与demo61.go的不同。尤其是<code>lock</code>变量的类型，以及发送通知的方式。</p><h2>总结</h2><p>我们今天主要讲了条件变量，它是基于互斥锁的一种同步工具。在Go语言中，我们需要用<code>sync.NewCond</code>函数来初始化一个<code>sync.Cond</code>类型的条件变量。</p><p><code>sync.NewCond</code>函数需要一个<code>sync.Locker</code>类型的参数值。</p><p><code>*sync.Mutex</code>类型的值以及<code>*sync.RWMutex</code>类型的值都可以满足这个要求。另外，后者的<code>RLocker</code>方法可以返回这个值中的读锁，也同样可以作为<code>sync.NewCond</code>函数的参数值，如此就可以生成与读写锁中的读锁对应的条件变量了。</p><p>条件变量的<code>Wait</code>方法需要在它基于的互斥锁保护下执行，否则就会引发不可恢复的panic。此外，我们最好使用<code>for</code>语句来检查共享资源的状态，并包裹对条件变量的<code>Wait</code>方法的调用。</p><p>不要用<code>if</code>语句，因为它不能重复地执行“检查状态-等待通知-被唤醒”的这个流程。重复执行这个流程的原因是，一个“因为等待通知，而被阻塞”的goroutine，可能会在共享资源的状态不满足其要求的情况下被唤醒。</p><p>条件变量的<code>Signal</code>方法只会唤醒一个因等待通知而被阻塞的goroutine，而它的<code>Broadcast</code>方法却可以唤醒所有为此而等待的goroutine。后者比前者的适应场景要多得多。</p><p>这两个方法并不需要受到互斥锁的保护，我们也最好不要在解锁互斥锁之前调用它们。还有，条件变量的通知具有即时性。当通知被发送的时候，如果没有任何goroutine需要被唤醒，那么该通知就会立即失效。</p><h2>思考题</h2><p><code>sync.Cond</code>类型中的公开字段<code>L</code>是做什么用的？我们可以在使用条件变量的过程中改变这个字段的值吗？</p><p><a href="https://github.com/hyper0x/Golang_Puzzlers">戳此查看Go语言专栏文章配套详细代码。</a></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/aa/53/768aec0a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>郝林</span>
  </div>
  <div class="_2_QraFYR_0">对条件变量这个工具本身还有疑问的读者可以去看我写的《Go 并发编程实战》第二版。这本书从并发程序的基本概念讲起，用一定篇幅的图文内容详细地讲解了条件变量的用法，同时还有一个贯穿了互斥锁和条件变量的示例。由于这本书的版权在出版社那里，所以我不能把其中的内容搬到这里。<br><br>我在这里只对大家共同的疑问进行简要说明：<br><br>1. 条件变量适合保护那些可执行两个对立操作的共享资源。比如，一个既可读又可写的共享文件。又比如，既有生产者又有消费者的产品池。<br><br>2. 对于有着对立操作的共享资源（比如一个共享文件），我们通常需要基于同一个锁的两个条件变量（比如 rcond 和 wcond）分别保护读操作和写操作（比如 rcond 保护读，wcond 保护写）。而且，读操作和写操作都需要同时持有这两个条件变量。因为，读操作在操作完成后还要向 wcond 发通知；写操作在操作完成后还要向 rcond 发通知。如此一来，读写操作才能在较少的锁争用的情况下交替进行。<br><br>3. 对于同一个条件变量，我们在调用它的 Signal 方法和 Broadcast 方法的时候不应该处在其包含的那个锁的保护下。也就是说，我们应该先撤掉保护屏障，再向 Wait 方法的调用方发出通知。否则，Wait 方法的调用方就有可能会错过通知。这也是我更推荐使用 Broadcast 方法的原因。所有等待方都错过通知的概率要小很多。<br><br>4. 相对应的，我们在调用条件变量的 Wait 方法的时候，应该处在其中的锁的保护之下。因为有同一个锁保护，所以不可能有多个 goroutine 同时执行到这个 Wait 方法调用，也就不可能存在针对其中锁的重复解锁。<br><br>5. 再强调一下。对于同一个锁，多个 goroutine 对它重复锁定时只会有一个成功，其余的会阻塞；多个 goroutine 对它重复解锁时也只会有一个成功，但其余的会抛 panic。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-18 21:14:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTI9X140JXPuaK4icMjjd7zJx8BObRv9MO4tuUeV9t9icm4PlZu6m8BFweOsdM8Qs5rTNT5zH0Xs240g/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_14c558</span>
  </div>
  <div class="_2_QraFYR_0">我理解是这样的。流程： 外部函数加锁 -&gt; 判断条件变量-&gt;wait内部解锁-&gt;阻塞等待信号-&gt;wait内部加锁-&gt; 修改条件变量-&gt; 外部解锁-&gt; 触发信号。 第一次加解锁是为了保证读条件变量时它不会被修改， wait解锁是为了条件变量能够被其他线程改变。wait内部再次加锁，是对条件变量的保护，因为外部要修改。 </div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-28 01:28:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/f5/b9/888fe350.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>不记年</span>
  </div>
  <div class="_2_QraFYR_0">老师，我有一个疑问，对于cond来说，每次只唤醒一个goruntine，如果这么goruntine发现消息不是自己想要的就会从新阻塞在wait函数中，那么真正需要这个消息的goruntine还会被唤醒吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不会，所以我才说应该优先用 broadcast。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-24 20:32:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4a/96/99466a06.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Laughing</span>
  </div>
  <div class="_2_QraFYR_0">L公开变量代表cond初始化时传递进来的锁，这个锁的状态是可以改变的，但会影响cond对互斥锁的控制。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 动这个L之前一定要三思，谨慎些，想想是不是会影响到程序。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-15 12:09:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/ac/a1/43d83698.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>云学</span>
  </div>
  <div class="_2_QraFYR_0">有个疑问，broadcast唤醒所有wait的goroutine，那他们被唤醒时需要去加锁(wait返回)，都能成功吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-18 19:28:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/0e/9c/cb9da823.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>猫王者</span>
  </div>
  <div class="_2_QraFYR_0">“我们最好在解锁条件变量基于的那个互斥锁之后，再去调用它的这两个方法（signal和Broadcast）。这更有利于程序的运行效率”    这个应该如何理解？<br><br>我的理解是如果先调用signal方法，然后在unlock解锁，如果在这两个操作中间该线程失去cpu，或者我人为的在siganl和unlock之间调用time.Sleep();在另一个等待线程中即使该等待线程被前者所发出的signal唤醒，但是唤醒的时候同时会去进行lock操作，但是前者的线程中由于失去了cpu，并没有调用unlock，那么这次唤醒不是应该失败了吗，即使前者有得到了cpu去执行了unlcok，但是signal操作具有及时性，等待线程不是应该继续等待下一个signal吗，感觉最后会变成死锁啊<br><br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-23 22:12:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/67/ff/d4f31b87.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>打你</span>
  </div>
  <div class="_2_QraFYR_0">在看了一遍，清楚了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-10 10:06:49</div>
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
  <div class="_2_QraFYR_0">```<br>	lock.Lock()<br>	for mailbox == 1 {<br>		sendCond.Wait()<br>	}<br>	mailbox = 1<br>	lock.Unlock()<br>	recvCond.Signal()<br>```<br>只要共享资源的状态不变，即使当前的 goroutine 因收到通知而被唤醒，也依然会再次执行这个Wait方法，并再次被阻塞。<br><br>老师我对这句话有个疑问，假设Wait不解锁，直接阻塞了当前goroutine，那么当收到通知时，mailbox的值应该已经被改成0了，此时唤醒，应该不满足for循环条件了呀，为什么会再次执行Wait方法呢？<br><br>我思考的结论是：Wait解锁是为了让其他goroutine去修改mailbox的值，不知道这么理解对吗。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 首先还是要明确这个过程：sendCond.Wait() 会先解锁再阻塞当前的 goroutine，然后等到别的地方调用 sendCond.Signal() 后，这里的 sendCond.Wait() 会加锁并唤醒当前的 goroutine。<br><br>假设有多个 goroutine 都调用了 sendCond.Wait() 方法，如果它们所在的 goroutine 都因条件变量的通知而被唤醒，那么（由于锁的缘故）只有一个 goroutine （以下称“1号G”）能够率先再次检查 mailbox。<br><br>“1号G”发现 mailbox 的值已经不是 1 了，随即退出循环，并且又把 mailbox 的值设置成了 1。最后解锁，并给 recvCond 发通知。<br><br>由于“1号G”的解锁，“2号G”、“3号G”等等（其他调用了 sendCond.Wait() 方法的G）都会竞争这个锁。得到该锁控制权的某个G会再锁住这个锁，然后发现 mailbox 的值依然是 1，随即再次调用 sendCond.Wait() 方法，并继续等待这个条件变量的下一个通知。其他的G也是类似的。<br><br>这个场景和过程要想清楚。可以自己画几条平行线，然后模拟一下这个过程。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-27 17:38:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/62/1e/8054e6db.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>樂文💤</span>
  </div>
  <div class="_2_QraFYR_0">所以如果用的signal方法只通知一个go routine的话 条件变量的判断方法改成if应该是没问题的 但如果是多个go routine 同时被唤醒就可能导致多个go routine 在不满足唤醒状态的时候被唤醒而导致处理错误，这时候就必须用for来不断地进行检测可以这么理解么</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 总是需要用 for 的，因为你没法保证收到信号之后那个状态就一定是你想要的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-03 09:15:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/93/ce/d4ac6fae.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>甦</span>
  </div>
  <div class="_2_QraFYR_0">源码问题问一下郝老师，cond wait方法里，多个协程走到c.L.Unlock()那一步不会出问题吗？ 只有一个协程可以unlock成功，其他协程重复unlock不就panic了吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-13 11:08:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/67/ff/d4f31b87.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>打你</span>
  </div>
  <div class="_2_QraFYR_0">lock.Lock()<br>for mailbox == 1 {<br> sendCond.Wait()<br>}<br>mailbox = 1<br>lock.Unlock()<br>recvCond.Signal()<br>如果wait已经解锁lock.Lock()锁住的锁，后面lock.Unlock解锁是什么意思？不会panic。<br>条件变量这2篇看起来前后理解不到</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-10 09:59:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>手指饼干</span>
  </div>
  <div class="_2_QraFYR_0">老师您好，关于发送通知是否需要在锁的保护下调用，还是有些疑问，以下想法请老师帮忙看看是否理解正确：<br>一个在等待通知的 goroutine 收到通知信号被唤醒，接下来执行的是条件变量 c.L.Lock() 操作，无论信号的发送方是否是在锁的保护下发送信号，该 goroutine 已经不是在等待通知的状态了，而是在尝试获取锁的状态，即使被阻塞，也是因为获取不到锁。区别只是，如果信号发送方在 Unlock() 之后发送信号，那么该 goroutine 被唤醒后获得锁可能会衔接得更好一点。<br>对于某些场景，比如说函数的开头两行代码就是<br>mutex.Lock() <br>defer mutex.Unlock()<br>这种情况是否可以允许通知在 Unlock() 之前调用</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 首先要明确：<br><br>条件变量的这两个方法并不需要受到互斥锁的保护，我们也最好不要在解锁互斥锁之前调用它们。<br><br>其次：<br><br>等待方法对锁的操作主要是为了让多个等待方有秩序的行事。在真正等待前释放锁，是为了让其他的等待方也具有进入等待状态的条件。在唤醒之后再次试图持有锁，是为了让多个等待方串行地检查条件是否满足以及执行后续的操作。<br><br>所以，结论是：<br><br>发送方在发送通知的时候最好不要持有等待方操作的那个锁。而等待方，应该在等待的时候持有锁加以保护。<br><br>最后：<br><br>关于你的最后一个问题，如果 mutex 是等待方操作的那个锁，那么就有可能产生发送方与等待方的互斥问题。由于等待方会在真正等待之前释放锁，所以应该不会产生完全死锁的情况。但是，这种没必要的操作肯定会影响程序的效率。<br><br>如果 mutex 与等待方没关系，那就无所谓了。你可以根据“是否想串行化通知的发送操作”来决定是否加锁。不过这通常也没什么必要。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-01 16:42:23</div>
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
  <div class="_2_QraFYR_0">老实您好，不太明白为什么一定要用条件变量呢。我看了 go并发编程第2版和你这边的做了对应，书上是已生成者消费者为例，我想的是用两个互斥量不行么？生成一个锁，消费一个锁，只是说可能会浪费在循环判断是否可生成，是否可消费中，还有可能因为某些不成功导致一直不能解锁的状况。  难不成条件变量主要就是优化上述问题的么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我记得在书里也说过类似的内容：条件变量常用于对共享资源的状态变化的监控和响应（由此可以实现状态机），同时也比较适合帮助多个线程协作操作同一个共享资源（这里的关键词是协作）。<br><br>条件变量有两种通知机制，这是互斥量没有的。互斥量只能保护共享资源（这里的关键词是保护），功能比较单一。所以条件变量在一些场景下会更高效。<br><br>你自己也说出了一些使用两个互斥锁来做的弊端。这些弊端其实就已经比较闹心了。一个是用两个锁对性能的影响，一个是两个锁如果配合不当就会造成局部的死锁。这还是多方协作操作同一个共享资源中最简单的情况。<br><br>再加上我前面说的那些，条件变量在此类场景下的优势已经非常明显了。注意它在Wait时还有原子级的操作呢。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-12 17:32:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/2f/ff/e9ebafec.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>RegExp</span>
  </div>
  <div class="_2_QraFYR_0">条件变量的Wait方法在阻塞当前的 goroutine之前，会解锁它基于的互斥锁，那是不是显示调用lock.Lock()的锁也被解锁了呢？<br><br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 原理上是这样，但是实践中最好不要这样混用同一个锁。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-30 11:55:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/a0/c5/9259d5ca.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>daydaygo</span>
  </div>
  <div class="_2_QraFYR_0">看完虽然了解了 syncCond 的用法, 但是使用场景还是不了解, 特别是从来没遇到过, 更感觉这个东西「无用」</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-13 14:02:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/50/b1/7b701518.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>黄仲辉</span>
  </div>
  <div class="_2_QraFYR_0">sync.crod 类比 java的 监视器锁</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: sync.Cond其实对应的是操作系统API中的条件变量。我记得Java也有条件变量这个东西吧。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-04 13:08:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/KhQRc8hIxHHyPV3Og2Fc5qhN3zkWUnw31wkc7mcmGyxicD9Yrvhh7N5B3icqpgWZXfuWbysn7Lv6QMPIEmYPeC4w/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_108cb5</span>
  </div>
  <div class="_2_QraFYR_0">试了一下把互斥锁改成读写锁， 中途会发生一个发送消息被两个接受者同时收到的场景， 导致最后发送者阻塞， 此时主线程的信道也阻塞了， 接受者协程执行完毕， 整体是死锁的情况。 那假如发送者和接收者都是无限循环， 而且多个接收者接收者接到同一份消息不会对业务有影响的情况下， 使用读写锁应该也是没有问题的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果你用的是读写锁的话，读写锁中的读锁定操作之间是不互斥的。所以我估计是你这块的用法错了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-01 23:56:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/d4/f3/129d6dfe.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>李二木</span>
  </div>
  <div class="_2_QraFYR_0">条件变量就是解决并发中的同步问题，原理跟Java差不多。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-22 17:11:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/96/3a/e06f8367.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>胡小涵</span>
  </div>
  <div class="_2_QraFYR_0">func testCond() {<br>	mu := sync.Mutex{}<br>	cond := sync.NewCond(&amp;mu)<br>	s := &quot;&quot;<br><br>	go func () {<br>		fmt.Println(&quot;This is consumer&quot;)<br>		for {<br>			mu.Lock()<br><br>			for len(s) == 0 {<br>				cond.Wait()<br>			}<br>			&#47;&#47;time.Sleep(time.Second * 1)<br>			fmt.Println(&quot;consumer, s:&quot;, s)<br>			s = &quot;&quot;<br>			mu.Unlock()<br>			cond.Signal()<br>		}<br>	}()<br><br>	go func () {<br>		fmt.Println(&quot;This is producer&quot;)<br>		for {<br>			mu.Lock()<br>			for len(s) &gt; 0 {<br>				cond.Wait()<br>			}<br>			fmt.Println(&quot;produce a string&quot;)<br>			s = &quot;generate s resource&quot;<br>			mu.Unlock()<br>			cond.Signal()<br>		}<br><br>	}()<br><br>	time.Sleep(time.Second * 10)<br>}<br>==========================================<br>自己测的时候发现用一个Cond就可以满足生产者消费者的简单模型，谁能告诉我为什么例子一定要使用两个Cond？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 因为核心需求不同。在你的例子只需要纯粹单向的生产-消费（这就是最简单的情况）。但在专栏的例子里，双方属于“秘密接触”，需要尽可能少的“露面”和“访问邮箱”。所以才需要互相通知。<br><br>需求决定代码。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-11 16:37:59</div>
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
  <div class="_2_QraFYR_0">看了你下面的留言 感觉条件变量主要是为了避免互斥锁或者读写锁 锁竞争条件</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 条件变量的特色是“协调”，而不是“互斥”。但是你也看到了，它是基于锁的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-07 20:29:55</div>
  </div>
</div>
</div>
</li>
</ul>