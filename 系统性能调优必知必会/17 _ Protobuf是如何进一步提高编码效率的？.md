<audio title="17 _ Protobuf是如何进一步提高编码效率的？" src="https://static001.geekbang.org/resource/audio/8e/20/8e66aeee4819a94f8edbf01c09c26320.mp3" controls="controls"></audio> 
<p>你好，我是陶辉。</p><p>上一讲介绍的HTTP/2协议在编码上拥有非常高的空间利用率，这一讲我们看看，相比其中的HPACK编码技术，Protobuf又是通过哪些新招式进一步提升编码效率的。</p><p>Google在2008年推出的Protobuf，是一个针对具体编程语言的编解码工具。它面向Windows、Linux等多种平台，也支持Java、Python、Golang、C++、Javascript等多种面向对象编程语言。使用Protobuf编码消息速度很快，消耗的CPU计算力也不多，而且编码后的字符流体积远远小于JSON等格式，能够大量节约昂贵的带宽，因此gRPC也把Protobuf作为底层的编解码协议。</p><p>然而，很多同学并不清楚Protobuf到底是怎样做到这一点的。这样，当你希望通过更换通讯协议这个高成本手段，提升整个分布式系统的性能时，面对可供选择的众多通讯协议，仅凭第三方的性能测试报告，你仍将难以作出抉择。</p><p>而且，面对分布式系统中的疑难杂症，往往需要通过分析抓取到的网络报文，确定到底是哪个组件出现了问题。可是由于Protobuf编码太过紧凑，即使对照着Proto消息格式文件，在不清楚编码逻辑时，你也很难解析出消息内容。</p><!-- [[[read_end]]] --><p>下面，我们将基于上一讲介绍过的HPACK编码技术，看看Protobuf是怎样进一步缩减编码体积的。</p><h2>怎样用最少的空间编码字段名？</h2><p>消息由多个名、值对组成，比如HTTP请求中，头部Host: www.taohui.pub就是一个名值对，其中，Host是字段名称，而www.taohui.pub是字段值。我们先来看Protobuf如何编码字段名。</p><p>对于多达几十字节的HTTP头部，HTTP/2静态表仅用一个数字来表示，其中，映射数字与字符串对应关系的表格，被写死在HTTP/2实现框架中。这样的编码效率非常高，<strong>但通用的HTTP/2框架只能将61个最常用的HTTP头部映射为数字，它能发挥出的作用很有限。</strong></p><p>动态表可以让更多的HTTP头部编码为数字，在上一讲的例子中，动态表将Host头部减少了96%的体积，效果惊人。但动态表生效得有一个前提：必须在一个会话连接上反复传输完全相同的HTTP头部。<strong>如果消息字段在1个连接上只发送了1次，或者反复传输时字段总是略有变动，动态表就无能为力了。</strong></p><p>有没有办法既使用静态表的预定义映射关系，又享受到动态表的灵活多变呢？<strong>其实只要把由HTTP/2框架实现的字段名映射关系，交由应用程序自行完成即可。</strong>而Protobuf就是这么做的。比如下面这段39字节的JSON消息，虽然一目了然，但字段名name、id、sex其实都是多余的，因为客户端与服务器的处理代码都清楚字段的含义。</p><pre><code>{&quot;name&quot;:&quot;John&quot;,&quot;id&quot;:1234,&quot;sex&quot;:&quot;MALE&quot;}
</code></pre><p>Protobuf将这3个字段名预分配了3个数字，定义在proto文件中：</p><pre><code>message Person {
  string name = 1;
  uint32 id = 2;  

  enum SexType {
    MALE = 0;
    FEMALE = 1;
  }
  SexType sex = 3;
}
</code></pre><p>接着，通过protoc程序便可以针对不同平台、编程语言，将它生成编解码类，最后通过类中自动生成的SerializeToString方法将消息序列化，编码后的信息仅有11个字节。其中，报文与字段的对应关系我放在下面这张图中。</p><p><img src="https://static001.geekbang.org/resource/image/12/b4/12907732b38fd0c0f41330985bb02ab4.png?wh=1256*760" alt=""></p><p>从图中可以看出，Protobuf是按照字段名、值类型、字段值的顺序来编码的，由于编码极为紧凑，所以分析时必须基于二进制比特位进行。比如红色的00001、00010、00011等前5个比特位，就分别代表着name、id、sex字段。</p><p>图中字段值的编码方式我们后面再解释，这里想必大家会有疑问，如果只有5个比特位表示字段名的值，那不是限制消息最多只有31个（2<sup>5</sup> - 1）字段吗？当然不是，字段名的序号可以从1到536870911（即2<sup>29</sup> - 1），可是，多数消息不过只有几个字段，这意味着可以用很小的序号表示它们。因此，对于小于16的序号，Protobuf仅有5个比特位表示，这样加上3位值类型，只需要1个字节表示字段名。对于大于16小于2027的序号，也只需要2个字节表示。</p><p>Protobuf可以用1到5个字节来表示一个字段名，因此，每个字节的第1个比特位保留，它为0时表示这是字段名的最后一个字节。下表列出了几种典型序号的编码值（请把黑色的二进制位，从右至左排列，比如2049应为000100000000001，即2048+1）。</p><p><img src="https://static001.geekbang.org/resource/image/43/33/43983f7fcba1d26eeea952dc0934d833.jpg?wh=1668*656" alt=""></p><p>说完字段名，我们再来看字段值是如何编码的。</p><h2>怎样高效地编码字段值？</h2><p>Protobuf对不同类型的值，采用6种不同的编码方式，如下表所示：</p><p><img src="https://static001.geekbang.org/resource/image/b2/67/b20120a8bac33d985275b5a2768ad067.jpg?wh=2148*800" alt=""></p><p>字符串用Length-delimited方式编码，顾名思义，在值长度后顺序添加ASCII字节码即可。比如上文例子中的John，对应的ASCII码如下表所示：</p><p><img src="https://static001.geekbang.org/resource/image/9f/cb/9f472ea914f98a81c03a7ad309f687cb.jpg?wh=1518*606" alt=""></p><p>这样，"John"需要5个字节进行编码，如下图所示（绿色表示长度，紫色表示ASCII码）：</p><p><img src="https://static001.geekbang.org/resource/image/6e/ae/6e45b5c7bb5e8766f6baef8c0e8b7bae.png?wh=1257*692" alt=""></p><p>这里需要注意，字符串长度的编码逻辑与字段名相同，当长度小于128（2<sup>7</sup>）时，1个字节就可以表示长度。若长度从128到16384（2<sup>14</sup>），则需要2个字节，以此类推。</p><p>由于字符串编码时未做压缩，所以并不会节约空间，但胜在速度快。<strong>如果你的消息中含有大量字符串，那么使用Huffman等算法压缩后再编码效果更好。</strong></p><p>我们再来看id：1234这个数字是如何编码的。其实Protobuf中所有数字的编码规则是一致的，字节中第1个比特位仅用于指示由哪些字节编码1个数字。例如图中的1234，将由14个比特位00010011010010表示（1024+128+64+16+2，正好是1234）。</p><p><strong>由于消息中的大量数字都很小，这种编码方式可以带来很高的空间利用率！</strong>当然，如果你确定数字很大，这种编码方式不但不能节约空间，而且会导致原先4个字节的大整数需要用5个字节来表示时，你也可以使用fixed32、fixed64等类型定义数字。</p><p>Protobuf还可以通过enum枚举类型压缩空间。回到第1幅图，sex: FEMALE仅用2个字节就编码完成，正是枚举值FEMALE使用数字1表示所达到的效果。</p><p><img src="https://static001.geekbang.org/resource/image/c9/c7/c9b6c10399a34d7a0e577a0397cd5ac7.png?wh=1256*760" alt=""></p><p>而且，由于Protobuf定义了每个字段的默认值，因此，当消息使用字段的默认值时，Protobuf编码时会略过该字段。以sex: MALE为例，由于MALE=0是sex的默认值，因此在第2幅示例图中，这2个字节都省去了。</p><p>另外，当使用repeated语法将多个数字组成列表时，还可以通过打包功能提升编码效率。比如下图中，对numbers字段添加101、102、103、104这4个值后，如果不使用打包功能，共需要8个字节编码，其中每个数字前都需要添加字段名。而使用打包功能后，仅用6个字节就能完成编码，显然列表越庞大，节约的空间越多。</p><p><img src="https://static001.geekbang.org/resource/image/ce/47/ce7ed2695b1e3dd869b59c438ee66147.png?wh=1524*564" alt=""></p><p>在Protobuf2版本中，需要显式设置 [packed=True] 才能使用打包功能，而在Protobuf3版本中这是默认功能。</p><p>最后，从<a href="https://github.com/protocolbuffers/protobuf/blob/master/docs/performance.md">这里</a>可以查看Protobuf的编解码性能测试报告，你能看到，在保持高空间利用率的前提下，Protobuf仍然拥有飞快的速度！</p><h2>小结</h2><p>这一讲我们介绍了Protobuf的编码原理。</p><p>通过在proto文件中为每个字段预分配1个数字，编码时就省去了完整字段名占用的空间。而且，数字越小编码时用掉的空间也越小，实际网络中大量传输的是小数字，这带来了很高的空间利用率。Protobuf的枚举类型也通过类似的原理，用数字代替字符串，可以节约许多空间。</p><p>对于字符串Protobuf没有做压缩，因此如果消息中的字符串比重很大时，建议你先压缩后再使用Protobuf编码。对于拥有默认值的字段，Protobuf编码时会略过它。对于repeated列表，使用打包功能可以仅用1个字段前缀描述所有数值，它在列表较大时能带来可观的空间收益。</p><h2>思考题</h2><p>下一讲我将介绍gRPC协议，它结合了HTTP/2与Protobuf的优点，在应用层提供方便而高效的RPC远程调用协议。你也可以提前思考下，既然Protobuf的空间效率远甚过HPACK技术，为什么gRPC还要使用HTTP/2协议呢？</p><p>在Protobuf的性能测试报告中，C++语言还拥有arenas功能，你可以通过option cc_enable_arenas = true语句打开它。请结合<a href="https://time.geekbang.org/column/article/230221">[第2讲]</a> 的内容，谈谈arenas为什么能提升消息的解码性能？欢迎你在留言区与我一起探讨。</p><p>感谢阅读，如果你觉得这节课对你有一些启发，也欢迎把它分享给你的朋友。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/19/f9/62ae32d7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Ken</span>
  </div>
  <div class="_2_QraFYR_0">gRPC基于Http2可以复用http2带来的新特性，比如双向流，单连接多路复用，头部压缩（hpack）。protobuf解决的是body的序列化空间效率，hpack解决的是header的空间效率，两者不冲突。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 完全正确！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-12 07:49:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/39/fa/a7edbc72.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>安排</span>
  </div>
  <div class="_2_QraFYR_0">protobuf需要通信双方提前约定好proto文件，这是一个限制，限制了它的使用场景。而http2没有这个要求，是一种更通用的设计，只要符合规范，就可以通信。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-12 07:29:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/47/b0/a9b77a1e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>冬风向左吹</span>
  </div>
  <div class="_2_QraFYR_0">wireshark支持protobuf协议插件：https:&#47;&#47;code.google.com&#47;archive&#47;p&#47;protobuf-wireshark&#47;downloads</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞！谢谢东郭的分享！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-15 09:17:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/11/78/4f0cd172.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>妥协</span>
  </div>
  <div class="_2_QraFYR_0">protobuf是按照字段名，字段类型和字段值编码。如果传输的是表格数据，就是第一行是多个字段，后面的多行都是字段值，这种情况下，字段名和字段类型只在第一行时传输，后面的多行字段值记录就不用传输了，这种效率是不是更高些?我们公司的自定义协议应该就是这种</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，表格其实就是二维数组，你可以通过protobuf中数组的packed功能，实现同样的效果。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-29 04:35:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/8f/cf/890f82d6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>那时刻</span>
  </div>
  <div class="_2_QraFYR_0">参考这里https:&#47;&#47;developers.google.com&#47;protocol-buffers&#47;docs&#47;reference&#47;arenas，学习了下protobuf对于arenas的介绍。<br><br>arena相当于内存池的概念，预先分配一块大内存，当protobuf操作消息对象需要分配内存的时候，去arenas来取，使用完之后放回到arena里。<br><br>这种做法的优势在于，1，加速内存分配和释放。2，有效利用cache line。 3，减少CPU时间</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-12 12:42:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/43/79/18073134.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>test</span>
  </div>
  <div class="_2_QraFYR_0">protobuf对body进行压缩，http2对header进行压缩。<br>http2还可以使用stream方式传输，这些都是protobuf没有的。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-12 11:22:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83erbY9UsqHZhhVoI69yXNibBBg0TRdUVsKLMg2UZ1R3NJxXdMicqceI5yhdKZ5Ad6CJYO0XpFHlJzIYQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>饭团</span>
  </div>
  <div class="_2_QraFYR_0">老师，请问红色和蓝色位为保留位，请问蓝色是出于什么目的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你好饭团，你是说整数编码吗？由于1个整数数值越小，可以用更少的字节表示，而越大则使用更多的字节表示，所以红色的首bit位用于表示这是否为最后1个字节。<br>蓝色保留位没明白什么意思。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-12 08:40:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/28/ef/9c/275dfed4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>诸葛子房</span>
  </div>
  <div class="_2_QraFYR_0">protobuf怎么使用缓存的</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-08 18:53:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/rRCSdTPyqWcW6U8DO9xL55ictNPlbQ38VAcaBNgibqaAhcH7mn1W9ddxIJLlMiaA5sngBicMX02w2HP5pAWpBAJsag/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>butterfly</span>
  </div>
  <div class="_2_QraFYR_0">一直有个疑问:<br>服务器端和客户端的两端都定义了.proto文件, 两端应该都是可以知道某个字段名字和值类型的。<br>如果只传输 字段的 顺序 和 值(字段名字和类型都不传输)，数据传到对端的时候， 再解码出来. 为什么不能这样做呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 实际就是像你说的这样做的，传递的只是字段的序号（为了提升灵活性，序号未必只能从1递增，可以任意指定）！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-22 16:56:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/d8/ee/6e7c2264.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Only now</span>
  </div>
  <div class="_2_QraFYR_0">proto和h2不是一个层级上的技术。h2工作在更底层。 grgp2 可以利用h2实现灵活的双工请求，一定程度上增加交互效率。proto主要用来编码消息体，降低body体积。<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-20 10:43:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_78d3bb</span>
  </div>
  <div class="_2_QraFYR_0">json简化了xml，protobuffer又优化了json 的key部分，双方都在proto中定义了key，所以只传序号查proto就知道是什么key了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的。proto对整数、数组的编码，以及编解码的速度都非常好。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-22 09:18:02</div>
  </div>
</div>
</div>
</li>
</ul>