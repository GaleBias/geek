<audio title="第12讲 _ TCP协议（下）：西行必定多妖孽，恒心智慧消磨难" src="https://static001.geekbang.org/resource/audio/16/28/1621d135d4d92756ca0d9440c0651c28.mp3" controls="controls"></audio> 
<p>我们前面说到玄奘西行，要出网关。既然出了网关，那就是在公网上传输数据，公网往往是不可靠的，因而需要很多的机制去保证传输的可靠性，这里面需要恒心，也即各种<strong>重传的策略</strong>，还需要有智慧，也就是说，这里面包含着<strong>大量的算法</strong>。</p><h2>如何做个靠谱的人？</h2><p>TCP想成为一个成熟稳重的人，成为一个靠谱的人。那一个人怎么样才算靠谱呢？咱们工作中经常就有这样的场景，比如你交代给下属一个事情以后，下属到底能不能做到，做到什么程度，什么时候能够交付，往往就会有应答，有回复。这样，处理事情的过程中，一旦有异常，你也可以尽快知道，而不是交代完之后就石沉大海，过了一个月再问，他说，啊我不记得了。</p><p>对应到网络协议上，就是客户端每发送的一个包，服务器端都应该有个回复，如果服务器端超过一定的时间没有回复，客户端就会重新发送这个包，直到有回复。</p><p>这个发送应答的过程是什么样呢？可以是<strong>上一个收到了应答，再发送下一个</strong>。这种模式有点像两个人直接打电话，你一句，我一句。但是这种方式的缺点是效率比较低。如果一方在电话那头处理的时间比较长，这一头就要干等着，双方都没办法干其他事情。咱们在日常工作中也不是这样的，不能你交代你的下属办一件事情，就一直打着电话看着他做，而是应该他按照你的安排，先将事情记录下来，办完一件回复一件。在他办事情的过程中，你还可以同时交代新的事情，这样双方就并行了。</p><!-- [[[read_end]]] --><p>如果使⽤这种模式，其实需要你和你的下属就不能靠脑⼦了，⽽是要都准备⼀个本⼦，你每交代下属⼀个事情，双方的本子都要记录⼀下。</p><p>当你的下属做完⼀件事情，就回复你，做完了，你就在你的本⼦上将这个事情划去。同时你的本⼦上每件事情都有时限，如果超过了时限下属还没有回复，你就要主动重新交代⼀下：上次那件事情，你还没回复我，咋样啦？</p><p>既然多件事情可以一起处理，那就需要给每个事情编个号，防止弄错了。例如，程序员平时看任务的时候，都会看JIRA的ID，而不是每次都要描述一下具体的事情。在大部分情况下，对于事情的处理是按照顺序来的，先来的先处理，这就给应答和汇报工作带来了方便。等开周会的时候，每个程序员都可以将JIRA ID的列表拉出来，说以上的都做完了，⽽不⽤⼀个个说。</p><h2>如何实现一个靠谱的协议？</h2><p>TCP协议使用的也是同样的模式。为了保证顺序性，每一个包都有一个ID。在建立连接的时候，会商定起始的ID是什么，然后按照ID一个个发送。为了保证不丢包，对于发送的包都要进行应答，但是这个应答也不是一个一个来的，而是会应答某个之前的ID，表示都收到了，这种模式称为<strong>累计确认</strong>或者<strong>累计应答</strong>（<strong>cumulative acknowledgment</strong>）。</p><p>为了记录所有发送的包和接收的包，TCP也需要发送端和接收端分别都有缓存来保存这些记录。发送端的缓存里是按照包的ID一个个排列，根据处理的情况分成四个部分。</p><p>第一部分：发送了并且已经确认的。这部分就是你交代下属的，并且也做完了的，应该划掉的。</p><p>第二部分：发送了并且尚未确认的。这部分是你交代下属的，但是还没做完的，需要等待做完的回复之后，才能划掉。</p><p>第三部分：没有发送，但是已经等待发送的。这部分是你还没有交代给下属，但是马上就要交代的。</p><p>第四部分：没有发送，并且暂时还不会发送的。这部分是你还没有交代给下属，而且暂时还不会交代给下属的。</p><p>这里面为什么要区分第三部分和第四部分呢？没交代的，一下子全交代了不就完了吗？</p><p>这就是我们上一节提到的十个词口诀里的“流量控制，把握分寸”。作为项目管理人员，你应该根据以往的工作情况和这个员工反馈的能力、抗压力等，先在心中估测一下，这个人一天能做多少工作。如果工作布置少了，就会不饱和；如果工作布置多了，他就会做不完；如果你使劲逼迫，人家可能就要辞职了。</p><p>到底一个员工能够同时处理多少事情呢？在TCP里，接收端会给发送端报一个窗口的大小，叫<strong>Advertised window</strong>。这个窗口的大小应该等于上面的第二部分加上第三部分，就是已经交代了没做完的加上马上要交代的。超过这个窗口的，接收端做不过来，就不能发送了。</p><p>于是，发送端需要保持下面的数据结构。</p><p><img src="https://static001.geekbang.org/resource/image/dd/44/dd67ba62279a3849c11ffc1deea25d44.jpg?wh=2136*756" alt=""></p><ul>
<li>
<p>LastByteAcked：第一部分和第二部分的分界线</p>
</li>
<li>
<p>LastByteSent：第二部分和第三部分的分界线</p>
</li>
<li>
<p>LastByteAcked + AdvertisedWindow：第三部分和第四部分的分界线</p>
</li>
</ul><p>对于接收端来讲，它的缓存里记录的内容要简单一些。</p><p>第一部分：接受并且确认过的。也就是我领导交代给我，并且我做完的。</p><p>第二部分：还没接收，但是马上就能接收的。也即是我自己的能够接受的最大工作量。</p><p>第三部分：还没接收，也没法接收的。也即超过工作量的部分，实在做不完。</p><p>对应的数据结构就像这样。<br>
﻿﻿<br>
<img src="https://static001.geekbang.org/resource/image/9d/be/9d597af268016f67caa14178627188be.jpg?wh=2220*756" alt=""></p><ul>
<li>
<p>MaxRcvBuffer：最大缓存的量；</p>
</li>
<li>
<p>LastByteRead之后是已经接收了，但是还没被应用层读取的；</p>
</li>
<li>
<p>NextByteExpected是第一部分和第二部分的分界线。</p>
</li>
</ul><p>第二部分的窗口有多大呢？</p><p>NextByteExpected和LastByteRead的差其实是还没被应用层读取的部分占用掉的MaxRcvBuffer的量，我们定义为A。</p><p>AdvertisedWindow其实是MaxRcvBuffer减去A。</p><p>也就是：AdvertisedWindow=MaxRcvBuffer-((NextByteExpected-1)-LastByteRead)。</p><p>那第二部分和第三部分的分界线在哪里呢？NextByteExpected加AdvertisedWindow就是第二部分和第三部分的分界线，其实也就是LastByteRead加上MaxRcvBuffer。</p><p>其中第二部分里面，由于受到的包可能不是顺序的，会出现空档，只有和第一部分连续的，可以马上进行回复，中间空着的部分需要等待，哪怕后面的已经来了。</p><h2>顺序问题与丢包问题</h2><p>接下来我们结合一个例子来看。</p><p>还是刚才的图，在发送端来看，1、2、3已经发送并确认；4、5、6、7、8、9都是发送了还没确认；10、11、12是还没发出的；13、14、15是接收方没有空间，不准备发的。</p><p>在接收端来看，1、2、3、4、5是已经完成ACK，但是没读取的；6、7是等待接收的；8、9是已经接收，但是没有ACK的。</p><p>发送端和接收端当前的状态如下：</p><ul>
<li>
<p>1、2、3没有问题，双方达成了一致。</p>
</li>
<li>
<p>4、5接收方说ACK了，但是发送方还没收到，有可能丢了，有可能在路上。</p>
</li>
<li>
<p>6、7、8、9肯定都发了，但是8、9已经到了，但是6、7没到，出现了乱序，缓存着但是没办法ACK。</p>
</li>
</ul><p>根据这个例子，我们可以知道，顺序问题和丢包问题都有可能发生，所以我们先来看<strong>确认与重发的机制</strong>。</p><p>假设4的确认到了，不幸的是，5的ACK丢了，6、7的数据包丢了，这该怎么办呢？</p><p>一种方法就是<strong>超时重试</strong>，也即对每一个发送了，但是没有ACK的包，都有设一个定时器，超过了一定的时间，就重新尝试。但是这个超时的时间如何评估呢？这个时间不宜过短，时间必须大于往返时间RTT，否则会引起不必要的重传。也不宜过长，这样超时时间变长，访问就变慢了。</p><p>估计往返时间，需要TCP通过采样RTT的时间，然后进行加权平均，算出一个值，而且这个值还是要不断变化的，因为网络状况不断地变化。除了采样RTT，还要采样RTT的波动范围，计算出一个估计的超时时间。由于重传时间是不断变化的，我们称为<strong>自适应重传算法</strong>（<strong>Adaptive Retransmission Algorithm</strong>）。</p><p>如果过一段时间，5、6、7都超时了，就会重新发送。接收方发现5原来接收过，于是丢弃5；6收到了，发送ACK，要求下一个是7，7不幸又丢了。当7再次超时的时候，有需要重传的时候，TCP的策略是<strong>超时间隔加倍</strong>。<strong>每当遇到一次超时重传的时候，都会将下一次超时时间间隔设为先前值的两倍</strong>。<strong>两次超时，就说明网络环境差，不宜频繁反复发送。</strong></p><p>超时触发重传存在的问题是，超时周期可能相对较长。那是不是可以有更快的方式呢？</p><p>有一个可以快速重传的机制，当接收方收到一个序号大于下一个所期望的报文段时，就会检测到数据流中的一个间隔，于是它就会发送冗余的ACK，仍然ACK的是期望接收的报文段。而当客户端收到三个冗余的ACK后，就会在定时器过期之前，重传丢失的报文段。</p><p>例如，接收方发现6收到了，8也收到了，但是7还没来，那肯定是丢了，于是发送6的ACK，要求下一个是7。接下来，收到后续的包，仍然发送6的ACK，要求下一个是7。当客户端收到3个重复ACK，就会发现7的确丢了，不等超时，马上重发。</p><p>还有一种方式称为<strong>Selective Acknowledgment</strong>  （<strong>SACK</strong>）。这种方式需要在TCP头里加一个SACK的东西，可以将缓存的地图发送给发送方。例如可以发送ACK6、SACK8、SACK9，有了地图，发送方一下子就能看出来是7丢了。</p><h2>流量控制问题</h2><p>我们再来看流量控制机制，在对于包的确认中，同时会携带一个窗口的大小。</p><p>我们先假设窗口不变的情况，窗口始终为9。4的确认来的时候，会右移一个，这个时候第13个包也可以发送了。</p><p><img src="https://static001.geekbang.org/resource/image/af/87/af16ecdfabf97f696d8133a20818fd87.jpg?wh=2136*756" alt=""></p><p>这个时候，假设发送端发送过猛，会将第三部分的10、11、12、13全部发送完毕，之后就停止发送了，未发送可发送部分为0。</p><p><img src="https://static001.geekbang.org/resource/image/e0/35/e011cb0e56f43bae942f0b7ab7407b35.jpg?wh=2418*903" alt=""></p><p>当对于包5的确认到达的时候，在客户端相当于窗口再滑动了一格，这个时候，才可以有更多的包可以发送了，例如第14个包才可以发送。</p><p><img src="https://static001.geekbang.org/resource/image/f5/c2/f5a4fcc035d1bb2d7e11c38391d768c2.jpg?wh=2418*903" alt=""></p><p>如果接收方实在处理的太慢，导致缓存中没有空间了，可以通过确认信息修改窗口的大小，甚至可以设置为0，则发送方将暂时停止发送。</p><p>我们假设一个极端情况，接收端的应用一直不读取缓存中的数据，当数据包6确认后，窗口大小就不能再是9了，就要缩小一个变为8。</p><p><img src="https://static001.geekbang.org/resource/image/95/9d/953e6706cfb5083e1f25b267505f5c9d.jpg?wh=2400*771" alt=""></p><p>这个新的窗口8通过6的确认消息到达发送端的时候，你会发现窗口没有平行右移，而是仅仅左面的边右移了，窗口的大小从9改成了8。</p><p><img src="https://static001.geekbang.org/resource/image/0a/1f/0a9265c63d5e0fb08c442ea0a7cffa1f.jpg?wh=2481*933" alt=""></p><p>如果接收端还是一直不处理数据，则随着确认的包越来越多，窗口越来越小，直到为0。</p><p><img src="https://static001.geekbang.org/resource/image/c2/4a/c24c414c31bd5deb346f98417ecdb74a.jpg?wh=2304*897" alt=""></p><p>当这个窗口通过包14的确认到达发送端的时候，发送端的窗口也调整为0，停止发送。</p><p><img src="https://static001.geekbang.org/resource/image/89/cb/89fe7b73e40363182b13e3d9c9aa2acb.jpg?wh=2724*1053" alt=""><br>
如果这样的话，发送方会定时发送窗口探测数据包，看是否有机会调整窗口的大小。当接收方比较慢的时候，要防止低能窗口综合征，别空出一个字节来就赶快告诉发送方，然后马上又填满了，可以当窗口太小的时候，不更新窗口，直到达到一定大小，或者缓冲区一半为空，才更新窗口。</p><p>这就是我们常说的流量控制。</p><h2>拥塞控制问题</h2><p>最后，我们看一下拥塞控制的问题，也是通过窗口的大小来控制的，前面的滑动窗口rwnd是怕发送方把接收方缓存塞满，而拥塞窗口cwnd，是怕把网络塞满。</p><p>这里有一个公式 LastByteSent - LastByteAcked &lt;= min {cwnd, rwnd} ，是拥塞窗口和滑动窗口共同控制发送的速度。</p><p>那发送方怎么判断网络是不是慢呢？这其实是个挺难的事情，因为对于TCP协议来讲，他压根不知道整个网络路径都会经历什么，对他来讲就是一个黑盒。TCP发送包常被比喻为往一个水管里面灌水，而TCP的拥塞控制就是在不堵塞，不丢包的情况下，尽量发挥带宽。</p><p>水管有粗细，网络有带宽，也即每秒钟能够发送多少数据；水管有长度，端到端有时延。在理想状态下，水管里面水的量=水管粗细 x 水管长度。对于到网络上，通道的容量 = 带宽 × 往返延迟。</p><p>如果我们设置发送窗口，使得发送但未确认的包为为通道的容量，就能够撑满整个管道。</p><p><img src="https://static001.geekbang.org/resource/image/c4/c6/c467d450d8000472e690ed378b8019c6.jpeg?wh=1352*1080" alt=""><br>
如图所示，假设往返时间为8s，去4s，回4s，每秒发送一个包，每个包1024byte。已经过去了8s，则8个包都发出去了，其中前4个包已经到达接收端，但是ACK还没有返回，不能算发送成功。5-8后四个包还在路上，还没被接收。这个时候，整个管道正好撑满，在发送端，已发送未确认的为8个包，正好等于带宽，也即每秒发送1个包，乘以来回时间8s。</p><p>如果我们在这个基础上再调大窗口，使得单位时间内更多的包可以发送，会出现什么现象呢？</p><p>我们来想，原来发送一个包，从一端到达另一端，假设一共经过四个设备，每个设备处理一个包时间耗费1s，所以到达另一端需要耗费4s，如果发送的更加快速，则单位时间内，会有更多的包到达这些中间设备，这些设备还是只能每秒处理一个包的话，多出来的包就会被丢弃，这是我们不想看到的。</p><p>这个时候，我们可以想其他的办法，例如这个四个设备本来每秒处理一个包，但是我们在这些设备上加缓存，处理不过来的在队列里面排着，这样包就不会丢失，但是缺点是会增加时延，这个缓存的包，4s肯定到达不了接收端了，如果时延达到一定程度，就会超时重传，也是我们不想看到的。</p><p>于是TCP的拥塞控制主要来避免两种现象，<strong>包丢失</strong>和<strong>超时重传</strong>。一旦出现了这些现象就说明，发送速度太快了，要慢一点。但是一开始我怎么知道速度多快呢，我怎么知道应该把窗口调整到多大呢？</p><p>如果我们通过漏斗往瓶子里灌水，我们就知道，不能一桶水一下子倒进去，肯定会溅出来，要一开始慢慢的倒，然后发现总能够倒进去，就可以越倒越快。这叫作慢启动。</p><p>一条TCP连接开始，cwnd设置为一个报文段，一次只能发送一个；当收到这一个确认的时候，cwnd加一，于是一次能够发送两个；当这两个的确认到来的时候，每个确认cwnd加一，两个确认cwnd加二，于是一次能够发送四个；当这四个的确认到来的时候，每个确认cwnd加一，四个确认cwnd加四，于是一次能够发送八个。可以看出这是<strong>指数性的增长</strong>。</p><p>涨到什么时候是个头呢？有一个值ssthresh为65535个字节，当超过这个值的时候，就要小心一点了，不能倒这么快了，可能快满了，再慢下来。</p><p>每收到一个确认后，cwnd增加1/cwnd，我们接着上面的过程来，一次发送八个，当八个确认到来的时候，每个确认增加1/8，八个确认一共cwnd增加1，于是一次能够发送九个，变成了线性增长。</p><p>但是线性增长还是增长，还是越来越多，直到有一天，水满则溢，出现了拥塞，这时候一般就会一下子降低倒水的速度，等待溢出的水慢慢渗下去。</p><p>拥塞的一种表现形式是丢包，需要超时重传，这个时候，将sshresh设为cwnd/2，将cwnd设为1，重新开始慢启动。这真是一旦超时重传，马上回到解放前。但是这种方式太激进了，将一个高速的传输速度一下子停了下来，会造成网络卡顿。</p><p>前面我们讲过<strong>快速重传算法</strong>。当接收端发现丢了一个中间包的时候，发送三次前一个包的ACK，于是发送端就会快速地重传，不必等待超时再重传。TCP认为这种情况不严重，因为大部分没丢，只丢了一小部分，cwnd减半为cwnd/2，然后sshthresh = cwnd，当三个包返回的时候，cwnd = sshthresh + 3，也就是没有一夜回到解放前，而是还在比较高的值，呈线性增长。</p><p><img src="https://static001.geekbang.org/resource/image/19/d2/1910bc1a0048d4de7b2128eb0f5dbcd2.jpg?wh=923*613" alt=""></p><p>就像前面说的一样，正是这种知进退，使得时延很重要的情况下，反而降低了速度。但是如果你仔细想一下，TCP的拥塞控制主要来避免的两个现象都是有问题的。</p><p><strong>第一个问题</strong>是丢包并不代表着通道满了，也可能是管子本来就漏水。例如公网上带宽不满也会丢包，这个时候就认为拥塞了，退缩了，其实是不对的。</p><p><strong>第二个问题</strong>是TCP的拥塞控制要等到将中间设备都填充满了，才发生丢包，从而降低速度，这时候已经晚了。其实TCP只要填满管道就可以了，不应该接着填，直到连缓存也填满。</p><p>为了优化这两个问题，后来有了<strong>TCP BBR拥塞算法</strong>。它企图找到一个平衡点，就是通过不断地加快发送速度，将管道填满，但是不要填满中间设备的缓存，因为这样时延会增加，在这个平衡点可以很好的达到高带宽和低时延的平衡。</p><p><img src="https://static001.geekbang.org/resource/image/a2/4c/a2b3a5df5eca52e302b75824e4bbbd4c.jpg?wh=762*508" alt=""></p><h2>小结</h2><p>好了，这一节我们就到这里，总结一下：</p><ul>
<li>
<p>顺序问题、丢包问题、流量控制都是通过滑动窗口来解决的，这其实就相当于你领导和你的工作备忘录，布置过的工作要有编号，干完了有反馈，活不能派太多，也不能太少；</p>
</li>
<li>
<p>拥塞控制是通过拥塞窗口来解决的，相当于往管道里面倒水，快了容易溢出，慢了浪费带宽，要摸着石头过河，找到最优值。</p>
</li>
</ul><p>最后留两个思考题：</p><ol>
<li>
<p>TCP的BBR听起来很牛，你知道他是如何达到这个最优点的嘛？</p>
</li>
<li>
<p>学会了UDP和TCP，你知道如何基于这两种协议写程序吗？这样的程序会有什么坑呢？</p>
</li>
</ol><p>欢迎你留言和我讨论。趣谈网络协议，我们下期见！</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/87/dd/4f53f95d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>进阶的码农</span>
  </div>
  <div class="_2_QraFYR_0">AdvertisedWindow=MaxRcvBuffer-((NextByteExpected-1)-LastByteRead)。<br>我根据图中例子计算 14-((5-1)-0) 算出来是10 ，括号里边的-1是减的什么，为啥和图例算出来的结果不一样，还是我计算的有问题，麻烦详细说一下 谢谢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这里我写的的确有问题，nextbyteexpected其实是6，就是目前接收到五，下一个期望的是六，这样就对了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-13 11:07:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/49/a5/e4c1c2d4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小文同学</span>
  </div>
  <div class="_2_QraFYR_0">1 设备缓存会导致延时？<br>	假如经过设备的包都不需要进入缓存，那么得到的速度是最快的。进入缓存且等待，等待的时间就是额外的延时。BBR就是为了避免这些问题：<br>	充分利用带宽；降低buffer占用率。<br><br>2 降低发送packet的速度，为何反而提速了？<br>	标准TCP拥塞算法是遇到丢包的数据时快速下降发送速度，因为算法假设丢包都是因为过程设备缓存满了。快速下降后重新慢启动，整个过程对于带宽来说是浪费的。通过packet速度-时间的图来看，从积分上看，BBR充分利用带宽时发送效率才是最高的。可以说BBR比标准TCP拥塞算法更正确地处理了数据丢包。对于网络上有一定丢包率的公网，BBR会更加智慧一点。<br>	回顾网络发展过程，带宽的是极大地改进的，而最小延迟会受限与介质传播速度，不会明显减少。BBR可以说是应运而生。<br><br>3 BBR如何解决延时？<br>	S1：慢启动开始时，以前期的延迟时间为延迟最小值Tmin。然后监控延迟值是否达到Tmin的n倍，达到这个阀值后，判断带宽已经消耗尽且使用了一定的缓存，进入排空阶段。<br>	S2：指数降低发送速率，直至延迟不再降低。这个过程的原理同S1<br>	S3：协议进入稳定运行状态。交替探测带宽和延迟，且大多数时间下都处于带宽探测阶段。<br><br>深夜读了BBR的论文和网上大牛的讲解得出的小结，分享给大家，过程比较匆忙，不足之处也望老师能指出指正。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-14 01:47:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/3f/c5/58787480.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>大唐江山</span>
  </div>
  <div class="_2_QraFYR_0">作者辛苦啊，这一章读起来开始有点累了😂</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-13 08:08:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/42/52/19553613.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>刘培培</span>
  </div>
  <div class="_2_QraFYR_0">BBR 论文原文：https:&#47;&#47;queue.acm.org&#47;detail.cfm?id=3022184</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 就应该这样，一言不合就读论文，是最好的方式</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-27 10:49:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>华林</span>
  </div>
  <div class="_2_QraFYR_0">这么说起来，我们经常发现下载速度是慢慢的增加到顶峰，然后就在那个值的附近徘徊，原因就是tcp的流量控制喽？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-13 08:10:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/c2/1e/ec02941d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>食用淡水鱼</span>
  </div>
  <div class="_2_QraFYR_0">快速重传那一段有问题。接收端接收6，7，8，9，10时漏掉了7，不是连续发送3个6ack，而是收到6，发送6的确认ack，收到8，9，10各发送一个6的重复ack，发送端检测到3个重复ack时（加上确认ack，总共有4个ack），进入重传机制。包括下面的拥塞控制，也是类似的逻辑</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-21 16:02:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/bf/8f/51f044dc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>谛听</span>
  </div>
  <div class="_2_QraFYR_0">不太清楚累积应答，比如接收端收到了包1、2、3、4，它的应答应该是5吗？也就是说中间的包就不用应答了吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-22 20:35:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/18/bb/9299fab1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Null</span>
  </div>
  <div class="_2_QraFYR_0">BBR 不填满缓存还是不填缓存？不填缓存那么缓存干啥用，如果填了了，即使不满，但是不是还有延迟。。。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 填的少，延迟就少了，当然做不到完全避免，毕竟缓存是路径上每一个设备自己的事情。缓存是每个设备自己的设计选择，BBR算法是两端的算法。<br><br>就像买火车票，我建议网上购买，身份证刷进去，这样速度快，但是对于火车站还是要设置人工窗口，因为不是每个人都会选择速度快的方式的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-27 12:50:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTL4mxgSAKHpHI2aNmG6EicicJekPRzBMXUC7TPxKYvEYhyA2RnpyT6ELicmwaiaTic7ibnWFAAOBaxDN0dg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>卜</span>
  </div>
  <div class="_2_QraFYR_0">好奇什么级别的程序开发需要了解怎么细，开发了好多网络程序都没用到，重传这些都是应用层在做</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-24 21:49:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/58/4b/34e7ceca.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>秋去冬来</span>
  </div>
  <div class="_2_QraFYR_0">快速重传那块6.8.9   7丢了为什么会发送3个冗余的6的3个ack</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 六的ack里面强调下一个是七</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-22 08:40:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/12/b3/9e8f4b4f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>MoFanDon</span>
  </div>
  <div class="_2_QraFYR_0">5的ACK丢包了，出发发送端重发。重发过去后，接收端发现接收过了，丢弃。……如果接收端丢弃5后，没有继续发送ACK,那这样不是发送端永远也也没法接受到5的ACK？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-07-24 08:54:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/d1/15/7d47de48.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>咖啡猫口里的咖啡猫🐱</span>
  </div>
  <div class="_2_QraFYR_0">老师，TCP协议栈，保证包一定到吗，，哪几种情况下会丢失，，，能不能总结下</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不能保证，只是尽力重试，再重试</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-13 12:15:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/6f/63/abb7bfe3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>扬～</span>
  </div>
  <div class="_2_QraFYR_0">2个问题：<br>1. TCP可靠的连接会不会影响到业务层，比如超时重传导致了服务端函数调用2次，那岂不是业务都要考虑幂等性了，我很懵逼，果然是懂得越多越白痴。<br>2. 拥塞控制的窗口跟流量控制的窗口一回事吗，还是流量控制的窗口的出来后就会进入拥塞控制窗口？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不会，重传的包不会交给应用层。是一个窗口</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-01 17:48:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/68/5d/1ccee378.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>茫农</span>
  </div>
  <div class="_2_QraFYR_0">有一个值 ssthresh 为 65535 个字节，，这个是什么意思？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: slow start threshold</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-23 10:47:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/68/ef/6264ca3d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Magic</span>
  </div>
  <div class="_2_QraFYR_0">祝刘超老师教师节快乐，专栏很棒，受益良多</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-10 09:02:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/3b/36/2d61e080.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>行者</span>
  </div>
  <div class="_2_QraFYR_0">问题1，想到BBR可以根据ACK时间来判断，比如同一时刻发送了A、B、C三个包，A、B两个包10ms收到ACK，而C包20ms后收到ACK，那么就认为网络拥堵或被中间设备缓存，降低发送速度。<br>问题2，TCP优点在于准确到达，可靠性高，但是速度慢；UDP优点在于简单，但是不确认可达；像后端接口一般使用TCP协议，因为客户端和服务器之间会有多次交互，且请求数据要确认可达；但是如果是直播的话使用UDP会更好，管你网络咋样，反正我赶紧发，让用户等急了就不好了。<br>刘老师，有一个问题，接口有时候会受到2次相同时间相同内容的请求，但是客户端仅仅调用了接口一次，会是什么原因导致这个问题呢？TCP重复发送包导致的嘛？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-06-13 09:17:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/fc/75/af67e5cc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>黑猫紧张</span>
  </div>
  <div class="_2_QraFYR_0">内核是将从网络接收的tcp数据 都接收完成再一次发给应用层呢 还是在tcp接收的过程中就已经开始发给应用层了 求回复</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 接收的过程中就发给应用层了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-29 16:22:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/pTD8nS0SsORKiaRD3wB0NK9Bpd0wFnPWtYLPfBRBhvZ68iaJErMlM2NNSeEibwQfY7GReILSIYZXfT9o8iaicibcyw3g/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>雷刚</span>
  </div>
  <div class="_2_QraFYR_0">1. rwnd：滑动窗口。上一讲提到的 TCP 报文中的 windown 字段，表示接收方将能接收的数据大小告知发送发方。<br>2. cwnd：拥塞窗口。TCP 发送方维护的一个变量，表示发送方一次要发送多少字节的数据。<br><br>总而言之，rwnd 用来控制能发送多少数据，cwnd 控制发送速度。这就和我们吃饭一样，虽然能吃两碗饭，但你不能一口就全部吃完，还是要一口口吃。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-09 08:07:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/88/cd/2c3808ce.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Yangjing</span>
  </div>
  <div class="_2_QraFYR_0">对于发送端，为什么会保存着“已发送并已确认”的数据呢？已确认的不是已经没用了吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，可以回收了，这里是分一下类</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-22 17:13:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/2b/ec/af6d0b10.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>caohuan</span>
  </div>
  <div class="_2_QraFYR_0">本篇所得：<br>1.数据在不同客户端传输 需要经过网关，在TCP传输层，用有不同任务的编号JIRA ID 标注 那些 包 已发送，哪些包 有反馈，灵活的做到 4个分类（1）发送 完成 且有反馈 （2）发送完成 没有反馈（3）将发送（4）不会发送，这可能是没有 带宽和空间，也可以知道 那些包 的ack丢了、数据丢了，这个可以使用 滑动窗口 的缓存机制 来处理，然后 通过调节 滑动窗口的大小 灵活应对 顺序问题 和丢包 问题 以及流量控制，最后还可以 灵活给TCP 指派 任务量，多少可以完成，多少可以待处理，多少不能被处理。<br>2.拥塞控制 处理 网络带宽塞满，防止 包丢失，超时重传 带来的 低延时 和 带宽利用率低的情况，所以 采用 TCP BBR拥塞算法 提高带宽利用率 和 低延时，并且不占缓存 的 高效方法。<br><br>回答老师的问题，第一个问题1.TCP的BBR很牛，如何达到最优点，老师 提到过 （1）高效率利用 带宽，让带宽 处于 满贯状态 （2）不占用缓存空间（3）低延时， 一旦空间被 占满，会流入 缓冲空间时，马上降速，降低传输速度，一直到有空余空间 再慢启动 提高 带宽利用率。<br>第二个问题：1.UDP和TCP应用在不同的应用场景下， UDP性善论，网络 一片和谐，有任务只管发送 和 接受，不太理会 网络环境 和 资源，不会主动调整速度;TCP 性恶论，世界很黑暗，得制定规则，如果 网络资源好，多传输数据，不好的环境，降低 传输速度，还得注重 顺序问题、丢包问题、控制情况、带宽流量等现实情况。<br>2.TCP的坑;TCP需要 考虑 带宽资源、阻塞、延时等各种问题，里面设计很多算法和数据结构，都没完全捋清 里面的原理，还不知道 坑在哪里，期待继续听 刘超老师的课程，都遇问题，然后解决问题 ，来填补TCP的坑。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-16 13:08:29</div>
  </div>
</div>
</div>
</li>
</ul>