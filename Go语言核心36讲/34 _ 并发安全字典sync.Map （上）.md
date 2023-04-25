<audio title="34 _ 并发安全字典sync.Map （上）" src="https://static001.geekbang.org/resource/audio/90/ab/907d94311773ebb569b1b49ed34b07ab.mp3" controls="controls"></audio> 
<p>在前面，我几乎已经把Go语言自带的同步工具全盘托出了。你是否已经听懂了会用了呢？</p><p>无论怎样，我都希望你能够多多练习、多多使用。它们和Go语言独有的并发编程方式并不冲突，相反，配合起来使用，绝对能达到“一加一大于二”的效果。</p><p>当然了，至于怎样配合就是一门学问了。我在前面已经讲了不少的方法和技巧，不过，更多的东西可能就需要你在实践中逐渐领悟和总结了。</p><hr></hr><p>我们今天再来讲一个并发安全的高级数据结构：<code>sync.Map</code>。众所周知，Go语言自带的字典类型<code>map</code>并不是并发安全的。</p><h2>前导知识：并发安全字典诞生史</h2><p>换句话说，在同一时间段内，让不同goroutine中的代码，对同一个字典进行读写操作是不安全的。字典值本身可能会因这些操作而产生混乱，相关的程序也可能会因此发生不可预知的问题。</p><p>在<code>sync.Map</code>出现之前，我们如果要实现并发安全的字典，就只能自行构建。不过，这其实也不是什么麻烦事，使用 <code>sync.Mutex</code>或<code>sync.RWMutex</code>，再加上原生的<code>map</code>就可以轻松地做到。</p><p>GitHub网站上已经有很多库提供了类似的数据结构。我在《Go并发编程实战》的第2版中也提供了一个比较完整的并发安全字典的实现。它的性能比同类的数据结构还要好一些，因为它在很大程度上有效地避免了对锁的依赖。</p><!-- [[[read_end]]] --><p>尽管已经有了不少的参考实现，Go语言爱好者们还是希望Go语言官方能够发布一个标准的并发安全字典。</p><p>经过大家多年的建议和吐槽，Go语言官方终于在2017年发布的Go 1.9中，正式加入了并发安全的字典类型<code>sync.Map</code>。</p><p>这个字典类型提供了一些常用的键值存取操作方法，并保证了这些操作的并发安全。同时，它的存、取、删等操作都可以基本保证在常数时间内执行完毕。换句话说，它们的算法复杂度与<code>map</code>类型一样都是<code>O(1)</code>的。</p><p>在有些时候，与单纯使用原生<code>map</code>和互斥锁的方案相比，使用<code>sync.Map</code>可以显著地减少锁的争用。<code>sync.Map</code>本身虽然也用到了锁，但是，它其实在尽可能地避免使用锁。</p><p>我们都知道，使用锁就意味着要把一些并发的操作强制串行化。这往往会降低程序的性能，尤其是在计算机拥有多个CPU核心的情况下。</p><p>因此，我们常说，能用原子操作就不要用锁，不过这很有局限性，毕竟原子只能对一些基本的数据类型提供支持。</p><p>无论在何种场景下使用<code>sync.Map</code>，我们都需要注意，与原生<code>map</code>明显不同，它只是Go语言标准库中的一员，而不是语言层面的东西。也正因为这一点，Go语言的编译器并不会对它的键和值，进行特殊的类型检查。</p><p>如果你看过<code>sync.Map</code>的文档或者实际使用过它，那么就一定会知道，它所有的方法涉及的键和值的类型都是<code>interface{}</code>，也就是空接口，这意味着可以包罗万象。所以，我们必须在程序中自行保证它的键类型和值类型的正确性。</p><p>好了，现在第一个问题来了。<strong>今天的问题是：并发安全字典对键的类型有要求吗？</strong></p><p>这道题的典型回答是：有要求。键的实际类型不能是函数类型、字典类型和切片类型。</p><p><strong>解析一下这个问题。</strong> 我们都知道，Go语言的原生字典的键类型不能是函数类型、字典类型和切片类型。</p><p>由于并发安全字典内部使用的存储介质正是原生字典，又因为它使用的原生字典键类型也是可以包罗万象的<code>interface{}</code>；所以，我们绝对不能带着任何实际类型为函数类型、字典类型或切片类型的键值去操作并发安全字典。</p><p>由于这些键值的实际类型只有在程序运行期间才能够确定，所以Go语言编译器是无法在编译期对它们进行检查的，不正确的键值实际类型肯定会引发panic。</p><p>因此，我们在这里首先要做的一件事就是：一定不要违反上述规则。我们应该在每次操作并发安全字典的时候，都去显式地检查键值的实际类型。无论是存、取还是删，都应该如此。</p><p>当然，更好的做法是，把针对同一个并发安全字典的这几种操作都集中起来，然后统一地编写检查代码。除此之外，把并发安全字典封装在一个结构体类型中，往往是一个很好的选择。</p><p>总之，我们必须保证键的类型是可比较的（或者说可判等的）。如果你实在拿不准，那么可以先通过调用<code>reflect.TypeOf</code>函数得到一个键值对应的反射类型值（即：<code>reflect.Type</code>类型的值），然后再调用这个值的<code>Comparable</code>方法，得到确切的判断结果。</p><h2>知识扩展</h2><h2>问题1：怎样保证并发安全字典中的键和值的类型正确性？（方案一）</h2><p>简单地说，可以使用类型断言表达式或者反射操作来保证它们的类型正确性。</p><p>为了进一步明确并发安全字典中键值的实际类型，这里大致有两种方案可选。</p><p><strong>第一种方案是，让并发安全字典只能存储某个特定类型的键。</strong></p><p>比如，指定这里的键只能是<code>int</code>类型的，或者只能是字符串，又或是某类结构体。一旦完全确定了键的类型，你就可以在进行存、取、删操作的时候，使用类型断言表达式去对键的类型做检查了。</p><p>一般情况下，这种检查并不繁琐。而且，你要是把并发安全字典封装在一个结构体类型里面，那就更加方便了。你这时完全可以让Go语言编译器帮助你做类型检查。请看下面的代码：</p><pre><code>type IntStrMap struct {
 m sync.Map
}

func (iMap *IntStrMap) Delete(key int) {
 iMap.m.Delete(key)
}

func (iMap *IntStrMap) Load(key int) (value string, ok bool) {
 v, ok := iMap.m.Load(key)
 if v != nil {
  value = v.(string)
 }
 return
}

func (iMap *IntStrMap) LoadOrStore(key int, value string) (actual string, loaded bool) {
 a, loaded := iMap.m.LoadOrStore(key, value)
 actual = a.(string)
 return
}

func (iMap *IntStrMap) Range(f func(key int, value string) bool) {
 f1 := func(key, value interface{}) bool {
  return f(key.(int), value.(string))
 }
 iMap.m.Range(f1)
}

func (iMap *IntStrMap) Store(key int, value string) {
 iMap.m.Store(key, value)
}
</code></pre><p>如上所示，我编写了一个名为<code>IntStrMap</code>的结构体类型，它代表了键类型为<code>int</code>、值类型为<code>string</code>的并发安全字典。在这个结构体类型中，只有一个<code>sync.Map</code>类型的字段<code>m</code>。并且，这个类型拥有的所有方法，都与<code>sync.Map</code>类型的方法非常类似。</p><p>两者对应的方法名称完全一致，方法签名也非常相似，只不过，与键和值相关的那些参数和结果的类型不同而已。在<code>IntStrMap</code>类型的方法签名中，明确了键的类型为<code>int</code>，且值的类型为<code>string</code>。</p><p>显然，这些方法在接受键和值的时候，就不用再做类型检查了。另外，这些方法在从<code>m</code>中取出键和值的时候，完全不用担心它们的类型会不正确，因为它的正确性在当初存入的时候，就已经由Go语言编译器保证了。</p><p>稍微总结一下。第一种方案适用于我们可以完全确定键和值的具体类型的情况。在这种情况下，我们可以利用Go语言编译器去做类型检查，并用类型断言表达式作为辅助，就像<code>IntStrMap</code>那样。</p><h2>总结</h2><p>我们今天讨论的是<code>sync.Map</code>类型，它是一种并发安全的字典。它提供了一些常用的键、值存取操作方法，并保证了这些操作的并发安全。同时，它还保证了存、取、删等操作的常数级执行时间。</p><p>与原生的字典相同，并发安全字典对键的类型也是有要求的。它们同样不能是函数类型、字典类型和切片类型。</p><p>另外，由于并发安全字典提供的方法涉及的键和值的类型都是<code>interface{}</code>，所以我们在调用这些方法的时候，往往还需要对键和值的实际类型进行检查。</p><p>这里大致有两个方案。我们今天主要提到了第一种方案，这是在编码时就完全确定键和值的类型，然后利用Go语言的编译器帮我们做检查。</p><p>在下一次的文章中，我们会提到另外一种方案，并对比这两种方案的优劣。除此之外，我会继续探讨并发安全字典的相关问题。</p><p>感谢你的收听，我们下期再见。</p><p><a href="https://github.com/hyper0x/Golang_Puzzlers">戳此查看Go语言专栏文章配套详细代码。</a></p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/20/27/a6932fbe.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>虢國技醬</span>
  </div>
  <div class="_2_QraFYR_0">打卡，后面的留言越来越少，看来有点难度了，加油啃下它！后面的兄弟</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-01 22:21:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/75/aa/21275b9d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>闫飞</span>
  </div>
  <div class="_2_QraFYR_0">我觉得这里的主要麻烦之一是golang不支持泛型编程这一重大的范式，用户不得不在上层代码里面做繁琐的检查。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-17 08:25:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/fe/2d/2c9177ca.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>给力</span>
  </div>
  <div class="_2_QraFYR_0">并发安全的字典里少了两个方法，比如已经有多少key，我们有什么解决办法没，只能自己每次插入或者删除key记录元素个数变化吗？<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，这种并发安全字典是在一些特定场景下用的，比如配置信息同步、批量计数等等。在这类场景下，键值对数量并不是关键问题。如果你想找通用的，可以参看一下这个：https:&#47;&#47;github.com&#47;gopcp&#47;example.v2&#47;tree&#47;master&#47;src&#47;gopcp.v2&#47;chapter5&#47;cmap</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-18 09:00:41</div>
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
  <div class="_2_QraFYR_0">这下可以把项目里丑陋的mutex+map去掉了👍👍👍</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-28 15:42:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/e8/55/92f82281.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>MClink</span>
  </div>
  <div class="_2_QraFYR_0">感觉并发编程和日常的业务CRUD还是有很多区别的，一般业务很多东西都用不上。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 程序例有共享程序实体或者资源的时候就肯定会用上并发编程的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-10 22:38:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/93/38/71615300.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>DayDayUp</span>
  </div>
  <div class="_2_QraFYR_0">老师，看到有人用github.com&#47;orcaman&#47;concurrent-map，不知道二者有何区别呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我没研究过那个concurrent-map。你可以功能测试比较一下两者的功能差异和API便捷度，并用基准测试比较一下两者的性能。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-08 22:01:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/29/0c/8d/90bee755.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>寄居与蟹</span>
  </div>
  <div class="_2_QraFYR_0">打卡，冲</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-25 17:41:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKELX1Rd1vmLRWibHib8P95NA87F4zcj8GrHKYQL2RcLDVnxNy1ia2geTWgW6L2pWn2kazrPNZMRVrIg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jxs1211</span>
  </div>
  <div class="_2_QraFYR_0">能用原子操作就不要用锁，不过这很有局限性，毕竟原子只能对一些基本的数据类型提供支持。问题：<br>atomic.Value不是可以支持除了二进制和int类型以外的数据类型的原子操作吗，还是其仍有一定局限性，另外atomic.Value使用是否有禁忌或者需要注意的地方</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: atomic.Value 原则上可以存任何值，但对于引用类型的值，要避免外界直接去修改这个值，否则 Value 的保护就形同虚设了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-30 11:42:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKELX1Rd1vmLRWibHib8P95NA87F4zcj8GrHKYQL2RcLDVnxNy1ia2geTWgW6L2pWn2kazrPNZMRVrIg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jxs1211</span>
  </div>
  <div class="_2_QraFYR_0">我们都知道，使用锁就意味着要把一些并发的操作强制串行化。这往往会降低程序的性能，尤其是在计算机拥有多个 CPU 核心的情况下。问题：<br>1 一个程序的每个进程只能跑在一个cpu的核心上，对吗？<br>2 每个进程中的锁只会控制该进程内部的goroutine串行执行，应该是不能控制其他进程里的goroutine的吧，那这里多核CPU下使用互斥锁更加影响性能的意思是指什么</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 1. 不对，进程与CPU核心之间一般不会有特定的对应关系，除非程序里手动锁定一对一的关系。大多数程序一般都会充分利用所有的CPU核心。<br><br>2. 每个进程的锁都只能控制当前进程中的代码执行流程。至于你的疑问，请参考第一个问题的答案：程序一般会同时使用多个CPU核心。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-30 11:41:46</div>
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
  <div class="_2_QraFYR_0">郝林老师，在 IntStrMap 的方法中这种 值点上 括号 string int。  key.(int) 、value.(string) 、a.(string) 代表的是将对应的数据转换为 string和 int 类型吧？ 我这样理解对吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: v.(T) 是在做类型转换断言，把 v 的类型转换成 T，如果转换失败就抛出 panic，如果不想让它抛 panic，可以：<br><br>v2, ok := v.(T)<br><br>如果 ok 为 false，就说明失败了；如果 ok 为 true 就说明成功了，此时 v2 就是转换后的值。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-21 22:34:27</div>
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
  <div class="_2_QraFYR_0">我最近遇到一个问题，sync.Map之间赋值操作之后，会变成线程不安全的map（原始map），赋值之后可能会导致底的原始map操作出现读写，导致问题。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-19 16:11:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/a7/b2/274a4192.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>漂泊的小飘</span>
  </div>
  <div class="_2_QraFYR_0">什么字典的并发写会造成fatal error呢，简单的原理是什么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你没说上下文，我没法解释啊。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-24 11:15:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/77/59/8bb1f879.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>涛声依旧</span>
  </div>
  <div class="_2_QraFYR_0">这个章节我又学习一招</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-28 22:52:53</div>
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
  <div class="_2_QraFYR_0">我觉得老师的示例代码要是对 sync.map 做一些可能引发冲突的并发操作就更好了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个不容易复现，尤其对于简单的示例来说。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-04 20:55:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/4b/35/28fa7039.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>猫九</span>
  </div>
  <div class="_2_QraFYR_0">打卡</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-08 21:13:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/67/6e/d59a413f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>羽仔</span>
  </div>
  <div class="_2_QraFYR_0">Fine.</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-22 22:29:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/0e/12/0398de50.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>么乞儿</span>
  </div>
  <div class="_2_QraFYR_0">打卡!</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-26 18:04:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/ee/69/39d0a2b2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>show</span>
  </div>
  <div class="_2_QraFYR_0">打卡，读完了，还需要实践下</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-15 08:22:22</div>
  </div>
</div>
</div>
</li>
</ul>