<audio title="35 _ 并发安全字典sync.Map (下)" src="https://static001.geekbang.org/resource/audio/cc/b6/ccdbcb5010588e22947e628c382b47b6.mp3" controls="controls"></audio> 
<p>你好，我是郝林，今天我们继续来分享并发安全字典sync.Map的内容。</p><p>我们在上一篇文章中谈到了，由于并发安全字典提供的方法涉及的键和值的类型都是<code>interface{}</code>，所以我们在调用这些方法的时候，往往还需要对键和值的实际类型进行检查。</p><p>这里大致有两个方案。我们上一篇文章中提到了第一种方案，在编码时就完全确定键和值的类型，然后利用Go语言的编译器帮我们做检查。</p><p>这样做很方便，不是吗？不过，虽然方便，但是却让这样的字典类型缺少了一些灵活性。</p><p>如果我们还需要一个键类型为<code>uint32</code>并发安全字典的话，那就不得不再如法炮制地写一遍代码了。因此，在需求多样化之后，工作量反而更大，甚至会产生很多雷同的代码。</p><h2>知识扩展</h2><h2>问题1：怎样保证并发安全字典中的键和值的类型正确性？（方案二）</h2><p>那么，如果我们既想保持<code>sync.Map</code>类型原有的灵活性，又想约束键和值的类型，那么应该怎样做呢？这就涉及了第二个方案。</p><p><strong>在第二种方案中，我们封装的结构体类型的所有方法，都可以与<code>sync.Map</code>类型的方法完全一致（包括方法名称和方法签名）。</strong></p><p>不过，在这些方法中，我们就需要添加一些做类型检查的代码了。另外，这样并发安全字典的键类型和值类型，必须在初始化的时候就完全确定。并且，这种情况下，我们必须先要保证键的类型是可比较的。</p><!-- [[[read_end]]] --><p>所以在设计这样的结构体类型的时候，只包含<code>sync.Map</code>类型的字段就不够了。</p><p>比如：</p><pre><code>type ConcurrentMap struct {
 m         sync.Map
 keyType   reflect.Type
 valueType reflect.Type
}
</code></pre><p>这里<code>ConcurrentMap</code>类型代表的是：可自定义键类型和值类型的并发安全字典。这个类型同样有一个<code>sync.Map</code>类型的字段<code>m</code>，代表着其内部使用的并发安全字典。</p><p>另外，它的字段<code>keyType</code>和<code>valueType</code>，分别用于保存键类型和值类型。这两个字段的类型都是<code>reflect.Type</code>，我们可称之为反射类型。</p><p>这个类型可以代表Go语言的任何数据类型。并且，这个类型的值也非常容易获得：通过调用<code>reflect.TypeOf</code>函数并把某个样本值传入即可。</p><p>调用表达式<code>reflect.TypeOf(int(123))</code>的结果值，就代表了<code>int</code>类型的反射类型值。</p><p><strong>我们现在来看一看<code>ConcurrentMap</code>类型方法应该怎么写。</strong></p><p><strong>先说<code>Load</code>方法</strong>，这个方法接受一个<code>interface{}</code>类型的参数<code>key</code>，参数<code>key</code>代表了某个键的值。</p><p>因此，当我们根据ConcurrentMap在<code>m</code>字段的值中查找键值对的时候，就必须保证ConcurrentMap的类型是正确的。由于反射类型值之间可以直接使用操作符<code>==</code>或<code>!=</code>进行判等，所以这里的类型检查代码非常简单。</p><pre><code>func (cMap *ConcurrentMap) Load(key interface{}) (value interface{}, ok bool) {
 if reflect.TypeOf(key) != cMap.keyType {
  return
 }
 return cMap.m.Load(key)
}
</code></pre><p>我们把一个接口类型值传入<code>reflect.TypeOf</code>函数，就可以得到与这个值的实际类型对应的反射类型值。</p><p>因此，如果参数值的反射类型与<code>keyType</code>字段代表的反射类型不相等，那么我们就忽略后续操作，并直接返回。</p><p>这时，<code>Load</code>方法的第一个结果<code>value</code>的值为<code>nil</code>，而第二个结果<code>ok</code>的值为<code>false</code>。这完全符合<code>Load</code>方法原本的含义。</p><p><strong>再来说<code>Store</code>方法。</strong><code>Store</code>方法接受两个参数<code>key</code>和<code>value</code>，它们的类型也都是<code>interface{}</code>。因此，我们的类型检查应该针对它们来做。</p><pre><code>func (cMap *ConcurrentMap) Store(key, value interface{}) {
 if reflect.TypeOf(key) != cMap.keyType {
  panic(fmt.Errorf(&quot;wrong key type: %v&quot;, reflect.TypeOf(key)))
 }
 if reflect.TypeOf(value) != cMap.valueType {
  panic(fmt.Errorf(&quot;wrong value type: %v&quot;, reflect.TypeOf(value)))
 }
 cMap.m.Store(key, value)
}
</code></pre><p>这里的类型检查代码与<code>Load</code>方法中的代码很类似，不同的是对检查结果的处理措施。当参数<code>key</code>或<code>value</code>的实际类型不符合要求时，<code>Store</code>方法会立即引发panic。</p><p>这主要是由于<code>Store</code>方法没有结果声明，所以在参数值有问题的时候，它无法通过比较平和的方式告知调用方。不过，这也是符合<code>Store</code>方法的原本含义的。</p><p>如果你不想这么做，也是可以的，那么就需要为<code>Store</code>方法添加一个<code>error</code>类型的结果。</p><p>并且，在发现参数值类型不正确的时候，让它直接返回相应的<code>error</code>类型值，而不是引发panic。要知道，这里展示的只一个参考实现，你可以根据实际的应用场景去做优化和改进。</p><p>至于与<code>ConcurrentMap</code>类型相关的其他方法和函数，我在这里就不展示了。它们在类型检查方式和处理流程上并没有特别之处。你可以在demo72.go文件中看到这些代码。</p><p>稍微总结一下。第一种方案适用于我们可以完全确定键和值具体类型的情况。在这种情况下，我们可以利用Go语言编译器去做类型检查，并用类型断言表达式作为辅助，就像<code>IntStrMap</code>那样。</p><p>在第二种方案中，我们无需在程序运行之前就明确键和值的类型，只要在初始化并发安全字典的时候，动态地给定它们就可以了。这里主要需要用到<code>reflect</code>包中的函数和数据类型，外加一些简单的判等操作。</p><p>第一种方案存在一个很明显的缺陷，那就是无法灵活地改变字典的键和值的类型。一旦需求出现多样化，编码的工作量就会随之而来。</p><p>第二种方案很好地弥补了这一缺陷，但是，那些反射操作或多或少都会降低程序的性能。我们往往需要根据实际的应用场景，通过严谨且一致的测试，来获得和比较程序的各项指标，并以此作为方案选择的重要依据之一。</p><h2>问题2：并发安全字典如何做到尽量避免使用锁？</h2><p><code>sync.Map</code>类型在内部使用了大量的原子操作来存取键和值，并使用了两个原生的<code>map</code>作为存储介质。</p><p><strong>其中一个原生<code>map</code>被存在了<code>sync.Map</code>的<code>read</code>字段中，该字段是<code>sync/atomic.Value</code>类型的。</strong> 这个原生字典可以被看作一个快照，它总会在条件满足时，去重新保存所属的<code>sync.Map</code>值中包含的所有键值对。</p><p>为了描述方便，我们在后面简称它为只读字典。不过，只读字典虽然不会增减其中的键，但却允许变更其中的键所对应的值。所以，它并不是传统意义上的快照，它的只读特性只是对于其中键的集合而言的。</p><p>由<code>read</code>字段的类型可知，<code>sync.Map</code>在替换只读字典的时候根本用不着锁。另外，这个只读字典在存储键值对的时候，还在值之上封装了一层。</p><p>它先把值转换为了<code>unsafe.Pointer</code>类型的值，然后再把后者封装，并储存在其中的原生字典中。如此一来，在变更某个键所对应的值的时候，就也可以使用原子操作了。</p><p><strong><code>sync.Map</code>中的另一个原生字典由它的<code>dirty</code>字段代表。</strong> 它存储键值对的方式与<code>read</code>字段中的原生字典一致，它的键类型也是<code>interface{}</code>，并且同样是把值先做转换和封装后再进行储存的。我们暂且把它称为脏字典。</p><p><strong>注意，脏字典和只读字典如果都存有同一个键值对，那么这里的两个键指的肯定是同一个基本值，对于两个值来说也是如此。</strong></p><p>正如前文所述，这两个字典在存储键和值的时候都只会存入它们的某个指针，而不是基本值。</p><p><code>sync.Map</code>在查找指定的键所对应的值的时候，总会先去只读字典中寻找，并不需要锁定互斥锁。只有当确定“只读字典中没有，但脏字典中可能会有这个键”的时候，它才会在锁的保护下去访问脏字典。</p><p>相对应的，<code>sync.Map</code>在存储键值对的时候，只要只读字典中已存有这个键，并且该键值对未被标记为“已删除”，就会把新值存到里面并直接返回，这种情况下也不需要用到锁。</p><p>否则，它才会在锁的保护下把键值对存储到脏字典中。这个时候，该键值对的“已删除”标记会被抹去。</p><p><img src="https://static001.geekbang.org/resource/image/41/51/418e648a9c370f67dffa70e84c96f451.png?wh=1164*727" alt=""></p><p><strong>sync.Map中的read与dirty</strong></p><p>顺便说一句，只有当一个键值对应该被删除，但却仍然存在于只读字典中的时候，才会被用标记为“已删除”的方式进行逻辑删除，而不会直接被物理删除。</p><p>这种情况会在重建脏字典以后的一段时间内出现。不过，过不了多久，它们就会被真正删除掉。在查找和遍历键值对的时候，已被逻辑删除的键值对永远会被无视。</p><p>对于删除键值对，<code>sync.Map</code>会先去检查只读字典中是否有对应的键。如果没有，脏字典中可能有，那么它就会在锁的保护下，试图从脏字典中删掉该键值对。</p><p>最后，<code>sync.Map</code>会把该键值对中指向值的那个指针置为<code>nil</code>，这是另一种逻辑删除的方式。</p><p>除此之外，还有一个细节需要注意，只读字典和脏字典之间是会互相转换的。在脏字典中查找键值对次数足够多的时候，<code>sync.Map</code>会把脏字典直接作为只读字典，保存在它的<code>read</code>字段中，然后把代表脏字典的<code>dirty</code>字段的值置为<code>nil</code>。</p><p>在这之后，一旦再有新的键值对存入，它就会依据只读字典去重建脏字典。这个时候，它会把只读字典中已被逻辑删除的键值对过滤掉。理所当然，这些转换操作肯定都需要在锁的保护下进行。</p><p><img src="https://static001.geekbang.org/resource/image/c5/f2/c5a9857311175ac94451fcefe52c30f2.png?wh=1045*802" alt=""><br>
<strong>sync.Map中read与dirty的互换</strong></p><p>综上所述，<code>sync.Map</code>的只读字典和脏字典中的键值对集合，并不是实时同步的，它们在某些时间段内可能会有不同。</p><p>由于只读字典中键的集合不能被改变，所以其中的键值对有时候可能是不全的。相反，脏字典中的键值对集合总是完全的，并且其中不会包含已被逻辑删除的键值对。</p><p>因此，可以看出，在读操作有很多但写操作却很少的情况下，并发安全字典的性能往往会更好。在几个写操作当中，新增键值对的操作对并发安全字典的性能影响是最大的，其次是删除操作，最后才是修改操作。</p><p>如果被操作的键值对已经存在于<code>sync.Map</code>的只读字典中，并且没有被逻辑删除，那么修改它并不会使用到锁，对其性能的影响就会很小。</p><h2>总结</h2><p>这两篇文章中，我们讨论了<code>sync.Map</code>类型，并谈到了怎样保证并发安全字典中的键和值的类型正确性。</p><p>为了进一步明确并发安全字典中键值的实际类型，这里大致有两种方案可选。</p><ul>
<li>
<p>其中一种方案是，在编码时就完全确定键和值的类型，然后利用Go语言的编译器帮我们做检查。</p>
</li>
<li>
<p>另一种方案是，接受动态的类型设置，并在程序运行的时候通过反射操作进行检查。</p>
</li>
</ul><p>这两种方案各有利弊，前一种方案在扩展性方面有所欠缺，而后一种方案通常会影响到程序的性能。在实际使用的时候，我们一般都需要通过客观的测试来帮助决策。</p><p>另外，在有些时候，与单纯使用原生字典和互斥锁的方案相比，使用<code>sync.Map</code>可以显著地减少锁的争用。<code>sync.Map</code>本身确实也用到了锁，但是，它会尽可能地避免使用锁。</p><p>这就要说到<code>sync.Map</code>对其持有两个原生字典的巧妙使用了。这两个原生字典一个被称为只读字典，另一个被称为脏字典。通过对它们的分析，我们知道了并发安全字典的适用场景，以及每种操作对其性能的影响程度。</p><h2>思考题</h2><p>今天的思考题是：关于保证并发安全字典中的键和值的类型正确性，你还能想到其他的方案吗？</p><p><a href="https://github.com/hyper0x/Golang_Puzzlers">戳此查看Go语言专栏文章配套详细代码。</a></p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/ac/95/9b3e3859.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Timo</span>
  </div>
  <div class="_2_QraFYR_0">做个sync.Map优化点的小总结：<br>1. 空间换时间。 通过冗余的两个数据结构(read、dirty),实现加锁对性能的影响<br>2. 使用只读数据(read)，避免读写冲突。<br>3. 动态调整，miss次数多了之后，将dirty数据提升为read<br>4. 延迟删除。 删除一个键值只是打标记，只有在提升dirty的时候才清理删除的数据<br>5. 优先从read读取、更新、删除，因为对read的读取不需要锁</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-30 11:29:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/83/4b/0e96fcae.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>sky</span>
  </div>
  <div class="_2_QraFYR_0">郝大 go方面能推荐下比较成熟的微服务框架吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 在我发布的Github优秀Go语言项目的思维导图里有。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-01 09:57:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/64/05/6989dce6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我来也</span>
  </div>
  <div class="_2_QraFYR_0">最近几天,golang升级到了1.15后,sync.Map又`火`了一把.<br>我又来温习了下本篇文章,也看了下对应的源码,<br>对sync.Map的印象又深刻了点.<br><br>具体的链接如下:<br>[踩了 Golang sync.Map 的一个坑](https:&#47;&#47;gocn.vip&#47;topics&#47;10860)<br>[sync: sync.Map keys will never be garbage collected](https:&#47;&#47;github.com&#47;golang&#47;go&#47;issues&#47;40999)<br><br>有兴趣的小伙伴可以去看看.</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-26 11:32:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/91/17/89c3d249.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>下雨天</span>
  </div>
  <div class="_2_QraFYR_0">老师好，关于：sync.Map在存储键值对的时候，只要只读字典中已存有这个键，并且该键值对未被标记为“已删除”，就会把新值存到里面并直接返回，这种情况下也不需要用到锁。这句话，只读map里面的值可以被替换的话，为什么不需要加锁？不会 有读写冲突吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 【第一个层面】<br>因为这里面的只读字典 read 是通过 sync&#47;atomic.Value 存储的，又正因为 read 是只读的，不存在增删键值对的情况，所以从 read 整体的层面可以安全地操作其中的某个键值对。<br><br>【第一个层面：更具体的细节】<br>在这种保护下，只要这里的键值对在数量上没有增减，就不会出现新数据丢失或者弄脏数据的情况。反例的话，可以看我最近写的这篇文章：https:&#47;&#47;mp.weixin.qq.com&#47;s&#47;ru161EtyQMrQVtWji0CqzQ<br><br>【第二个层面】<br>由于其中的每一个键值对（entry 结构）都可以保证自身的并发性（ entry 内部只会存指针，因此用原子操作就可以保证并发读写的安全性），所以从 read 中单个键值对的层面也就有了并发安全性方面的保证。<br><br>【结论】<br>综合以上两种措施，这里才无需使用锁来保护。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-20 11:31:18</div>
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
  <div class="_2_QraFYR_0">这个设计很巧妙，在自己的开发中可以借鉴这种思想。有个问题请问老师：<br>“脏字典中的键值对集合总是完全的”，而“read 和 dirty 互换之后 dirty 会置空”，那么重建的意思是不是这样的：在下一次访问 read 的时候，将 read 中的键值对全部复制到 dirty 中?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: read 和 dirty 互换是分两步走的。Load 的时候如果发现“不得不去 dirty 中查找”的情况已经有很多了，就会把 dirty 作为新的 read，然后把 dirty 置为 nil。之后，在 Store 的时候，如果发现健是新的，而且是对于新 read 的第一个新健（此时 dirty 必定为 nil），那么就重新初始化 dirty，然后把新 read 中的有效键一个一个地存入 dirty。<br><br>另外你可以尝试阅读一下 sync.Map 的源码，写得还是挺清晰的。可以配合着这里的文章去看。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-05 09:36:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/a7/8b/208d9e2c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Rainman</span>
  </div>
  <div class="_2_QraFYR_0">这个脏字典让我想起了mysql的刷脏页。<br>给老师点赞。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-09 22:53:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/4b/69/c02eac91.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>大漠胡萝卜</span>
  </div>
  <div class="_2_QraFYR_0">sync.Map适用于读多写少的情况，如果写数据比较频繁可以参考：https:&#47;&#47;github.com&#47;orcaman&#47;concurrent-map</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-12 17:47:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/83/f1/d6676d6b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>墨水里的鱼</span>
  </div>
  <div class="_2_QraFYR_0">如何初始化reflect.Type？reflect.TypeOf(1) reflect.TypeOf(&quot;a&quot;) 只能这样吗?</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-30 17:10:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/ce/07/15a91f75.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>渺小de尘埃</span>
  </div>
  <div class="_2_QraFYR_0">当一个结构体里的字段是sync.map类型的，怎么json序列化呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 既然要序列化了就用不着同步了吧，用个普通map倒腾一下呗。或者你再包装一下，自定义序列化过程。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-01 17:27:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/18/ac/4d68ba46.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>金时</span>
  </div>
  <div class="_2_QraFYR_0">&#47;&#47; The read field itself is always safe to load, but must only be stored with mu held.<br>老师，源代码里对read变量注释时说read 在store时，需要加锁，没懂这是为什么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你应该知道，atomic.Value 类型的值（以下简称“value”吧）只能保证（完整地）存&#47;取操作的原子性。比如，你在里面存一个 map，它只能保证存这个 map 或取这个 map 的时候是原子操作，但你如果要存、取、改、删这个 map 里的键值对，那 value 就管不到了（这时就会是非原子的操作）。<br><br>另外，这里还有一个问题，如果有多个 goroutine 同时向同一个 value 里存值，那么，里面到底存成了哪一个 goroutine 提供的值就不好说了。如果没有并发保护，多个并发写非常容易造成问题。<br><br>最后，你看源码肯定也知道，read 里存的是私有类型 readOnly 的实例，这个类型本身并不是并发安全的。所以无论对它做什么操作，都需要并发安全保护。 这也跟“并发多写”的问题有关。<br><br>所以，无论是替换 read 里的 readOnly 值（如果是单 goroutine 写就不用，可惜这里不可能做出这种保证），还是修改该值的内部，都需要有锁的保护。<br><br>你看，原子操作虽然能保证单值存取的原子性，但还是太简单了，在很多场景下是不足以完全保证并发安全的。不过，作为“可升级的并发安全保障”中的第一级防护还是挺好用的，就像 sync.Map 里做的那样。<br><br>sync.Map 的 Store 方法和 Load 方法里都是有一个升级保障的过程的，“升级”的标志就是用了 m.mu 。 </p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-10 23:32:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/8c/e2/48f4e4fa.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>mkii</span>
  </div>
  <div class="_2_QraFYR_0">老师，看到源码中Store的时候有个疑惑。如果read中存在此key对应的vlaue，则tryStore替换read的value。这里如果在dirty给read并将dirty置为nil的时候不会丢失新数据吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不会啊，这里操作（包括存取和转移）的都是指针啊。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-04 23:45:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/07/ba/c80fac39.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>夜来寒雨晓来风</span>
  </div>
  <div class="_2_QraFYR_0">关于文中提到的“键值对应该被删除，但却仍然存在于只读字典”，什么时候会出现这种情形呢？对于sync.Map的删除机制看的不是很明白，希望能解答一下，谢谢~</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 虽然在只读字典里，但是你肯定是获取不到的。这样做是为了操作效率，是一种“用空间换时间”的做法。之后内部做整理的时候，这些都会被一并删除的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-07 23:09:34</div>
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
  <div class="_2_QraFYR_0">看了一下源码有地方不理解，有劳解答一下 <br>第一：Store、Load等方法都会执行两次m.read.Load().(readOnly)，去判断两次  这样做的目的是什么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 主要是怕第一次的操作没有体现真实情况，所以在锁的保护下再来一次。这就相当于快路径和慢路径。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-21 16:30:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/0e/ef/030e6d27.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>xl000</span>
  </div>
  <div class="_2_QraFYR_0">老师，read、dirty交换和访问read这两个操作，难道不需要保护read变量吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 交换时会有互斥锁的保护，而 read 实际上存在了一个 sync&#47;atomic.Value 里面。所以相关操作都是并发安全的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-21 09:26:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1c/96/dd/1620a744.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>simple_孙</span>
  </div>
  <div class="_2_QraFYR_0">更新时只修改只读map，不会造成数据不一致吗，后面应该会定期同步到脏map里吧？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不会造成数据不一致，因为读的时候会先读 read 再读 dirty。而且不是“只修改 read”，如果 read 里面有这个键，那就修改 read，如果没有还会到 dirty 里面找。<br><br>sync.Map 会定期让 dirty 变成 read，然后重建 dirty（以下简称“read 转换”吧）。“定期”的时机是，dirty 中有太多 read 里没有的键值对（以下简称“脏键值对”吧）。<br><br>具体来说是，发现一个“脏键值对”就增加一次计数。每当已发现的“脏键值对”的数量等于或大于 dirty 的长度时，就触发一次“read 转换”。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-27 10:44:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/54/d6/124e2e93.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Calios</span>
  </div>
  <div class="_2_QraFYR_0">个人以为是读过的这个专栏里最精彩的一篇~ 只读字典和脏字典的实现好精彩，忍不住再去细细读一下sync.Map的实现。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-14 11:23:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/05/3c/6f2a4724.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>唐大少在路上。。。</span>
  </div>
  <div class="_2_QraFYR_0">班门弄斧，其实有个细节帮老师丰富一下：<br>两个原生map的定义为 map[interface{}]*entry<br>其中的entry为一个只包含一个unsafe.pointer的结构体<br>这里之所以value设置为指针类型，个人感觉就是为了在dirtry重建的时候直接把read里面这个entry的地址copy到dirty里面，这样当read中对entry里面的pointer执行原子替换的时候，dirty里面的值也会跟随着改变。<br>这样，当每次对read已有key进行更新的时候就不用单独去操作一次dirty了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-11 15:57:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/20/27/a6932fbe.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>虢國技醬</span>
  </div>
  <div class="_2_QraFYR_0">打卡</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-03 00:34:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/ca/e3/7e860739.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>一介农夫</span>
  </div>
  <div class="_2_QraFYR_0">1.18后可以用泛型实现了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯嗯，你可以利用泛型改进一下这里的代码。试试看。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-06 23:30:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/26/20/bf/7c053186.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Ronin</span>
  </div>
  <div class="_2_QraFYR_0">老师，我读了好几遍，还是不明白为什么要检查键值对的类型，这不是强制整个字典的key都为一个类型，整个字典的value都为一个类型了，而实际上原先的字典的键值对类型是可以多样化的存储，像这样：<br>var m sync.Map<br>m.Store(&quot;test&quot;, 18)<br>m.Store(18, &quot;test&quot;)<br>而不是只能：<br>m.Store(1, &quot;a&quot;)<br>m.Store(2, &quot;b&quot;)<br>就算要检查，应该是检查键的实际类型不能是函数类型、字典类型和切片类型<br>还望老师解惑下~谢谢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: sync.Map 依然是可以多类型存储的啊，你觉得这是优势还是劣势呢？<br><br>我觉得这是并不是什么好事。因为这相当于把检查类型的工作抛给了程序员。实际上，这在当初也是一种在缺少自定义泛型支持情况下的 plan B，并不是最佳实践。<br><br>现在有了泛型，sync.Map其实可以被改造的更好，只不过为了向后兼容性，折中改造就被搁置了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-10-28 22:16:26</div>
  </div>
</div>
</div>
</li>
</ul>