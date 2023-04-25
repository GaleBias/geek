<audio title="18 _ 如何用硬件同步原语（CAS）替代锁？" src="https://static001.geekbang.org/resource/audio/75/49/7535b08b3e3c2ae76998afc5b43a6349.mp3" controls="controls"></audio> 
<p>你好，我是李玥。上节课，我们一起学习了如何使用锁来保护共享资源，你也了解到，使用锁是有一定性能损失的，并且，如果发生了过多的锁等待，将会非常影响程序的性能。</p><p>在一些特定的情况下，我们可以使用硬件同步原语来替代锁，可以保证和锁一样的数据安全性，同时具有更好的性能。</p><p>在今年的NSDI（NSDI是USENIX组织开办的关于网络系统设计的著名学术会议）上，伯克利大学发表了一篇论文《<a href="http://www.usenix.org/conference/nsdi19/presentation/khandelwal">Confluo: Distributed Monitoring and Diagnosis Stack for High-speed Networks</a>》，这个论文中提到的Confluo，也是一个类似于消息队列的流数据存储，它的吞吐量号称是Kafka的4～10倍。对于这个实验结论我个人不是很认同，因为它设计的实验条件对Kafka来说不太公平。但不可否认的是，Confluo它的这个设计思路是一个创新，并且实际上它的性能也非常好。</p><p>Confluo是如何做到这么高的吞吐量的呢？这里面非常重要的一个创新的设计就是，它使用硬件同步原语来代替锁，在一个日志上（你可以理解为消息队列中的一个队列或者分区），保证严格顺序的前提下，实现了多线程并发写入。</p><p>今天，我们就来学习一下，如何用硬件同步原语（CAS）替代锁？</p><!-- [[[read_end]]] --><h2>什么是硬件同步原语？</h2><p>为什么硬件同步原语可以替代锁呢？要理解这个问题，你要首先知道硬件同步原语是什么。</p><p>硬件同步原语（Atomic Hardware Primitives）是由计算机硬件提供的一组原子操作，我们比较常用的原语主要是CAS和FAA这两种。</p><p>CAS（Compare and Swap），它的字面意思是：先比较，再交换。我们看一下CAS实现的伪代码：</p><pre><code>&lt;&lt; atomic &gt;&gt;
function cas(p : pointer to int, old : int, new : int) returns bool {
    if *p ≠ old {
        return false
    }
    *p ← new
    return true
}
</code></pre><p>它的输入参数一共有三个，分别是：</p><ul>
<li>p: 要修改的变量的指针。</li>
<li>old: 旧值。</li>
<li>new: 新值。</li>
</ul><p>返回的是一个布尔值，标识是否赋值成功。</p><p>通过这个伪代码，你就可以看出CAS原语的逻辑，非常简单，就是先比较一下变量p当前的值是不是等于old，如果等于，那就把变量p赋值为new，并返回true，否则就不改变变量p，并返回false。</p><p>这是CAS这个原语的语义，接下来我们看一下FAA原语（Fetch and Add）：</p><pre><code>&lt;&lt; atomic &gt;&gt;
function faa(p : pointer to int, inc : int) returns int {
    int value &lt;- *location
    *p &lt;- value + inc
    return value
}
</code></pre><p>FAA原语的语义是，先获取变量p当前的值value，然后给变量p增加inc，最后返回变量p之前的值value。</p><p>讲到这儿估计你会问，这两个原语到底有什么特殊的呢？</p><p>上面的这两段伪代码，如果我们用编程语言来实现，肯定是无法保证原子性的。而原语的特殊之处就是，它们都是由计算机硬件，具体说就是CPU提供的实现，可以保证操作的原子性。</p><p>我们知道，<strong>原子操作具有不可分割性，也就不存在并发的问题</strong>。所以在某些情况下，原语可以用来替代锁，实现一些即安全又高效的并发操作。</p><p>CAS和FAA在各种编程语言中，都有相应的实现，可以来直接使用，无论你是使用哪种编程语言，它们底层的实现是一样的，效果也是一样的。</p><p>接下来，还是拿我们熟悉的账户服务来举例说明一下，看看如何使用CAS原语来替代锁，实现同样的安全性。</p><h2>CAS版本的账户服务</h2><p>假设我们有一个共享变量balance，它保存的是当前账户余额，然后我们模拟多个线程并发转账的情况，看一下如何使用CAS原语来保证数据的安全性。</p><p>这次我们使用Go语言来实现这个转账服务。先看一下使用锁实现的版本：</p><pre><code>package main

import (
	&quot;fmt&quot;
	&quot;sync&quot;
)

func main() {
	// 账户初始值为0元
	var balance int32
	balance = int32(0)
	done := make(chan bool)
	// 执行10000次转账，每次转入1元
	count := 10000

	var lock sync.Mutex

	for i := 0; i &lt; count; i++ {
		// 这里模拟异步并发转账
		go transfer(&amp;balance, 1, done, &amp;lock)
	}
	// 等待所有转账都完成
	for i := 0; i &lt; count; i++ {
		&lt;-done
	}
	// 打印账户余额
	fmt.Printf(&quot;balance = %d \n&quot;, balance)
}
// 转账服务
func transfer(balance *int32, amount int, done chan bool, lock *sync.Mutex) {
	lock.Lock()
	*balance = *balance + int32(amount)
	lock.Unlock()
	done &lt;- true
}
</code></pre><p>这个例子中，我们让账户的初始值为0，然后启动多个协程来并发执行10000次转账，每次往账户中转入1元，全部转账执行完成后，账户中的余额应该正好是10000元。</p><p>如果你没接触过Go语言，不了解协程也没关系，你可以简单地把它理解为进程或者线程都可以，这里我们只是希望能异步并发执行转账，我们并不关心这几种“程”他们之间细微的差别。</p><p>这个使用锁的版本，反复多次执行，每次balance的结果都正好是10000，那这段代码的安全性是没问题的。接下来我们看一下，使用CAS原语的版本。</p><pre><code>func transferCas(balance *int32, amount int, done chan bool) {
	for {
		old := atomic.LoadInt32(balance)
		new := old + int32(amount)
		if atomic.CompareAndSwapInt32(balance, old, new) {
			break
		}
	}
	done &lt;- true
}
</code></pre><p>这个CAS版本的转账服务和上面使用锁的版本，程序的总体结构是一样的，主要的区别就在于，“异步给账户余额+1”这一小块儿代码的实现。</p><p>那在使用锁的版本中，需要先获取锁，然后变更账户的值，最后释放锁，完成一次转账。我们可以看一下使用CAS原语的实现：</p><p>首先，它用for来做了一个没有退出条件的循环。在这个循环的内部，反复地调用CAS原语，来尝试给账户的余额+1。先取得账户当前的余额，暂时存放在变量old中，再计算转账之后的余额，保存在变量new中，然后调用CAS原语来尝试给变量balance赋值。我们刚刚讲过，CAS原语它的赋值操作是有前置条件的，只有变量balance的值等于old时，才会将balance赋值为new。</p><p>我们在for循环中执行了3条语句，在并发的环境中执行，这里面会有两种可能情况：</p><p>一种情况是，执行到第3条CAS原语时，没有其他线程同时改变了账户余额，那我们是可以安全变更账户余额的，这个时候执行CAS的返回值一定是true，转账成功，就可以退出循环了。并且，CAS这一条语句，它是一个原子操作，赋值的安全性是可以保证的。</p><p>另外一种情况，那就是在这个过程中，有其他线程改变了账户余额，这个时候是无法保证数据安全的，不能再进行赋值。执行CAS原语时，由于无法通过比较的步骤，所以不会执行赋值操作。本次尝试转账失败，当前线程并没有对账户余额做任何变更。由于返回值为false，不会退出循环，所以会继续重试，直到转账成功退出循环。</p><p>这样，每一次转账操作，都可以通过若干次重试，在保证安全性的前提下，完成并发转账操作。</p><p>其实，对于这个例子，还有更简单、性能更好的方式：那就是，直接使用FAA原语。</p><pre><code>func transferFaa(balance *int32, amount int, done chan bool) {
	atomic.AddInt32(balance, int32(amount))
	done &lt;- true
}
</code></pre><p>FAA原语它的操作是，获取变量当前的值，然后把它做一个加法，并且保证这个操作的原子性，一行代码就可以搞定了。看到这儿，你可能会想，那CAS原语还有什么意义呢？</p><p>在这个例子里面，肯定是使用FAA原语更合适，但是我们上面介绍的，使用CAS原语的方法，它的适用范围更加广泛一些。类似于这样的逻辑：先读取数据，做计算，然后更新数据，无论这个计算是什么样的，都可以使用CAS原语来保护数据安全，但是FAA原语，这个计算的逻辑只能局限于简单的加减法。所以，我们上面讲的这种使用CAS原语的方法并不是没有意义的。</p><p>另外，你需要知道的是，这种使用CAS原语反复重试赋值的方法，它是比较耗费CPU资源的，因为在for循环中，如果赋值不成功，是会立即进入下一次循环没有等待的。如果线程之间的碰撞非常频繁，经常性的反复重试，这个重试的线程会占用大量的CPU时间，随之系统的整体性能就会下降。</p><p>缓解这个问题的一个方法是使用Yield()， 大部分编程语言都支持Yield()这个系统调用，Yield()的作用是，告诉操作系统，让出当前线程占用的CPU给其他线程使用。每次循环结束前调用一下Yield()方法，可以在一定程度上减少CPU的使用率，缓解这个问题。你也可以在每次循环结束之后，Sleep()一小段时间，但是这样做的代价是，性能会严重下降。</p><p>所以，这种方法它只适合于线程之间碰撞不太频繁，也就是说绝大部分情况下，执行CAS原语不需要重试这样的场景。</p><h2>小结</h2><p>这节课我们一起学习了CAS和FAA这两个原语。这些原语，是由CPU提供的原子操作，在并发环境中，单独使用这些原语不用担心数据安全问题。在特定的场景中，CAS原语可以替代锁，在保证安全性的同时，提供比锁更好的性能。</p><p>接下来，我们用转账服务这个例子，分别演示了CAS和FAA这两个原语是如何替代锁来使用的。对于类似：“先读取数据，做计算，然后再更新数据”这样的业务逻辑，可以使用CAS原语+反复重试的方式来保证数据安全，前提是，线程之间的碰撞不能太频繁，否则太多重试会消耗大量的CPU资源，反而得不偿失。</p><h2>思考题</h2><p>这节课的课后作业，依然需要你去动手来写代码。你需要把我们这节课中的讲到的账户服务这个例子，用你熟悉的语言，用锁、CAS和FAA这三种方法，都完整地实现一遍。每种实现方法都要求是完整的，可以执行的程序。</p><p>因为，对于并发和数据安全这块儿，你不仅要明白原理，熟悉相关的API，会正确地使用，是非常重要的。在这部分写出的Bug，都比较诡异，不好重现，而且很难调试。你会发现，你的数据一会儿是对的，一会儿又错了。或者在你开发的电脑上都正确，部署到服务器上又错了等等。所以，熟练掌握，一次性写出正确的代码，这样会帮你省出很多找Bug的时间。</p><p>验证作业是否正确的方法是，你反复多次执行你的程序，应该每次打印的结果都是：</p><pre><code>balance = 10000
</code></pre><p>欢迎你把代码上传到GitHub上，然后在评论区给出访问链接。如果你有任何问题，也可以在评论区留言与我交流。</p><p>感谢阅读，如果你觉得这篇文章对你有一些启发，也欢迎把它分享给你的朋友。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/99/7b/0a056674.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ponymm</span>
  </div>
  <div class="_2_QraFYR_0">“CAS 和 FAA 在各种编程语言中，都有相应的实现，可以来直接使用，无论你是使用哪种编程语言，它底层使用的系统调用是一样的，效果也是一样的。” 李老师这句话有点小问题：car,faa并不是通过系统调用实现的，系统调用的开销不小，cas本来就是为了提升性能，不会走系统调用。事实上是在用户态直接使用汇编指令就可以实现</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢你指出错误，我已经联系编辑在文稿中改正了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-03 13:20:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/14/17/8763dced.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>微微一笑</span>
  </div>
  <div class="_2_QraFYR_0">老师好，实现了下CAS,代码连接：https:&#47;&#47;github.com&#47;shenyachen&#47;JKSJ&#47;blob&#47;master&#47;study&#47;src&#47;main&#47;java&#47;com&#47;jksj&#47;study&#47;casAndFaa&#47;CASThread.java。<br>对于FAA，通过查找资料，jdk1.8在调用sun.misc.Unsafe#getAndAddInt方法时，会根据系统底层是否支持FAA，来决定是使用FAA还是CAS。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍👍👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-03 16:16:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/28/9c/73e76b19.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>姜戈</span>
  </div>
  <div class="_2_QraFYR_0">JAVA中的FAA和CAS: FAA就是用CAS实现的。<br><br>public final int getAndAddInt(Object var1, long var2, int var4) {<br>        int var5;<br>        do {<br>            var5 = this.getIntVolatile(var1, var2);<br>        } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));<br><br>        return var5;<br>    }</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-03 18:32:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/58/28/c86340ca.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>达文西</span>
  </div>
  <div class="_2_QraFYR_0">cas需要注意 aba 问题吧</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-10 18:00:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/57/4f/6fb51ff1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>一步</span>
  </div>
  <div class="_2_QraFYR_0">NodeJS中，没有发现有关操作CpU原语CAS或者FAA的实现的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以试试这个：https:&#47;&#47;developer.mozilla.org&#47;en-US&#47;docs&#47;Web&#47;JavaScript&#47;Reference&#47;Global_Objects&#47;Atomics</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-03 17:33:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/7b/57/a9b04544.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>QQ怪</span>
  </div>
  <div class="_2_QraFYR_0">MutxLock：https:&#47;&#47;github.com&#47;xqq1994&#47;algorithm&#47;blob&#47;master&#47;src&#47;main&#47;java&#47;com&#47;test&#47;concurrency&#47;MutxLock.java<br>CAS、FFA:<br>https:&#47;&#47;github.com&#47;xqq1994&#47;algorithm&#47;blob&#47;master&#47;src&#47;main&#47;java&#47;com&#47;test&#47;concurrency&#47;CAS.java<br>完成了老师的作业，好高兴</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍👍👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-03 21:31:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/BKusySicUz9ppf2aZCxe5jMMX2aI8o6kHRRQibFosGEZf7zNFue9OcTsmuicOHOBicZ404Z522AgrTlJa3gSW9AL8w/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_a5453a</span>
  </div>
  <div class="_2_QraFYR_0">没懂为什么消息队列的课一半都是在介绍其他一些基础概念，是我对高手这个词有误解吗</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-10 21:30:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/03/10/26f9f762.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Switch</span>
  </div>
  <div class="_2_QraFYR_0">java实现， 锁、Unsafe类、AtomicInteger类<br><br>https:&#47;&#47;github.com&#47;Switch-vov&#47;mq-learing&#47;tree&#47;master&#47;src&#47;main&#47;java&#47;com&#47;switchvov&#47;transfer</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-25 14:36:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/52/3c/d6fcb93a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张三</span>
  </div>
  <div class="_2_QraFYR_0">复习了一下Java中的原子类，对应到go里边的CAS实现中的for循环是自旋，还有就是要注意ABA问题吧。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-03 05:34:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/bb/61/2c2f5024.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>haijian.yang</span>
  </div>
  <div class="_2_QraFYR_0">是不是应该说一下这部分在消息队列的应用？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-03 13:19:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/7e/99/c4302030.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Khirye</span>
  </div>
  <div class="_2_QraFYR_0"> 用JAVA实验了下，看起来在竞争比较激烈的情况下，CAS和Lock的耗时是差不多的，想问下老师这种情况下怎么选择用Lock还CAS呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-30 17:48:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/d1/16/6347bbc0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>顶新</span>
  </div>
  <div class="_2_QraFYR_0">.net 实现代码：<br>lock 实现：<br>static void transfer(object amount)<br>        {<br>            lock (obj)<br>            {<br>                balance = balance + Convert.ToInt32(amount);<br>            }            <br>        }<br>cas 实现：<br>static void transfer_cas(object amount)<br>        {<br>            int initialValue, computedValue;<br>            do<br>            {<br>                initialValue = balance;                <br>                computedValue = initialValue + Convert.ToInt32(amount);                <br>            } while (initialValue != Interlocked.CompareExchange(<br>                ref balance, computedValue, initialValue));<br>        }<br><br>faa 实现：<br>static void transfer_faa(object amount){<br>            Interlocked.Add(ref balance,Convert.ToInt32(amount));<br>        }<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-23 13:57:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/51/8d/09f28606.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>明日</span>
  </div>
  <div class="_2_QraFYR_0">Java实现: https:&#47;&#47;gist.github.com&#47;imgaoxin&#47;a2b09715af99b993e30b44963cebc530</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: transfer2要放在循环中，否则有可能转账失败。<br>另外，transfer1中，虽然一个简单的加法不会引起任何异常，但总是把unlock放到finnally中是一个好习惯。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-03 17:40:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/34/df/64e3d533.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>leslie</span>
  </div>
  <div class="_2_QraFYR_0">   打卡：老师一步步剥离一层层拨开实质-又涨知识了，期待老师的下节课。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-03 16:30:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/52/3c/d6fcb93a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张三</span>
  </div>
  <div class="_2_QraFYR_0">Java里边有支持FAA这种CPU指令的实现吗？以前没听说</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 在java中，可以看一下java.util.concurrent.atomic.AtomicLong#getAndAdd</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-03 05:23:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/d4/44/0ec958f4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Eleven</span>
  </div>
  <div class="_2_QraFYR_0">这个FAA的原语为什么是先获取之前的值x，然后做计算x+a，最后返回x?</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-14 09:15:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/d3/01/716d45b6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>LQS  KF</span>
  </div>
  <div class="_2_QraFYR_0">转账服务涉及两个账户的原子性操作，感觉还是用锁比较好，文章中只操作了接受转账的单账号原子操作，个人觉得不妥。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-14 10:05:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/7b/36/fd46331c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Jussi Lee</span>
  </div>
  <div class="_2_QraFYR_0">Java 可以用completedFuture 实现比较方便一些</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-19 11:26:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83eou1BMETumU21ZI4yiaLenOMSibzkAgkw944npIpsJRicmdicxlVQcgibyoQ00rdGk9Htp1j0dM5CP2Fibw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>寥若晨星</span>
  </div>
  <div class="_2_QraFYR_0">yield那段有问题，使用yield会让出cpu，导致用户态和内核态的切换，产生系统开销，这样还不如用锁呢。。。违背了使用CAS的初衷</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-09 13:42:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/STqKg1kLgvRuduQfo0R2E2osYBian7XrQAjSWmOwL9nyZVhq7vyLPnlGcgvguFV4aV7ToWLFiauEMKy96KWHKBVg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>离境”</span>
  </div>
  <div class="_2_QraFYR_0">为什么我的实验结果，加锁方式比cas快许多</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-06 16:05:48</div>
  </div>
</div>
</div>
</li>
</ul>