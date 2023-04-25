<audio title="22 _ panic函数、recover函数以及defer语句（下）" src="https://static001.geekbang.org/resource/audio/3f/98/3f98658635e74f6aba9f05ce55e42298.mp3" controls="controls"></audio> 
<p>你好，我是郝林，今天我们继续来聊聊panic函数、recover函数以及defer语句的内容。</p><p>我在前一篇文章提到过这样一个说法，panic之中可以包含一个值，用于简要解释引发此panic的原因。</p><p>如果一个panic是我们在无意间引发的，那么其中的值只能由Go语言运行时系统给定。但是，当我们使用<code>panic</code>函数有意地引发一个panic的时候，却可以自行指定其包含的值。我们今天的第一个问题就是针对后一种情况提出的。</p><h2>知识扩展</h2><h3>问题 1：怎样让panic包含一个值，以及应该让它包含什么样的值？</h3><p>这其实很简单，在调用<code>panic</code>函数时，把某个值作为参数传给该函数就可以了。由于<code>panic</code>函数的唯一一个参数是空接口（也就是<code>interface{}</code>）类型的，所以从语法上讲，它可以接受任何类型的值。</p><p>但是，我们最好传入<code>error</code>类型的错误值，或者其他的可以被有效序列化的值。这里的“有效序列化”指的是，可以更易读地去表示形式转换。</p><p>还记得吗？对于<code>fmt</code>包下的各种打印函数来说，<code>error</code>类型值的<code>Error</code>方法与其他类型值的<code>String</code>方法是等价的，它们的唯一结果都是<code>string</code>类型的。</p><p>我们在通过占位符<code>%s</code>打印这些值的时候，它们的字符串表示形式分别都是这两种方法产出的。</p><!-- [[[read_end]]] --><p>一旦程序异常了，我们就一定要把异常的相关信息记录下来，这通常都是记到程序日志里。</p><p>我们在为程序排查错误的时候，首先要做的就是查看和解读程序日志；而最常用也是最方便的日志记录方式，就是记下相关值的字符串表示形式。</p><p>所以，如果你觉得某个值有可能会被记到日志里，那么就应该为它关联<code>String</code>方法。如果这个值是<code>error</code>类型的，那么让它的<code>Error</code>方法返回你为它定制的字符串表示形式就可以了。</p><p>对于此，你可能会想到<code>fmt.Sprintf</code>，以及<code>fmt.Fprintf</code>这类可以格式化并输出参数的函数。</p><p>是的，它们本身就可以被用来输出值的某种表示形式。不过，它们在功能上，肯定远不如我们自己定义的<code>Error</code>方法或者<code>String</code>方法。因此，为不同的数据类型分别编写这两种方法总是首选。</p><p>可是，这与传给<code>panic</code>函数的参数值又有什么关系呢？其实道理是相同的。至少在程序崩溃的时候，panic包含的那个值字符串表示形式会被打印出来。</p><p>另外，我们还可以施加某种保护措施，避免程序的崩溃。这个时候，panic包含的值会被取出，而在取出之后，它一般都会被打印出来或者记录到日志里。</p><p>既然说到了应对panic的保护措施，我们再来看下面一个问题。</p><h3>问题 2：怎样施加应对panic的保护措施，从而避免程序崩溃？</h3><p>Go语言的内建函数<code>recover</code>专用于恢复panic，或者说平息运行时恐慌。<code>recover</code>函数无需任何参数，并且会返回一个空接口类型的值。</p><p>如果用法正确，这个值实际上就是即将恢复的panic包含的值。并且，如果这个panic是因我们调用<code>panic</code>函数而引发的，那么该值同时也会是我们此次调用<code>panic</code>函数时，传入的参数值副本。请注意，这里强调用法的正确。我们先来看看什么是不正确的用法。</p><pre><code>package main

import (
 &quot;fmt&quot;
 &quot;errors&quot;
)

func main() {
 fmt.Println(&quot;Enter function main.&quot;)
 // 引发panic。
 panic(errors.New(&quot;something wrong&quot;))
 p := recover()
 fmt.Printf(&quot;panic: %s\n&quot;, p)
 fmt.Println(&quot;Exit function main.&quot;)
}
</code></pre><p>在上面这个<code>main</code>函数中，我先通过调用<code>panic</code>函数引发了一个panic，紧接着想通过调用<code>recover</code>函数恢复这个panic。可结果呢？你一试便知，程序依然会崩溃，这个<code>recover</code>函数调用并不会起到任何作用，甚至都没有机会执行。</p><p>还记得吗？我提到过panic一旦发生，控制权就会讯速地沿着调用栈的反方向传播。所以，在<code>panic</code>函数调用之后的代码，根本就没有执行的机会。</p><p>那如果我把调用<code>recover</code>函数的代码提前呢？也就是说，先调用<code>recover</code>函数，再调用<code>panic</code>函数会怎么样呢？</p><p>这显然也是不行的，因为，如果在我们调用<code>recover</code>函数时未发生panic，那么该函数就不会做任何事情，并且只会返回一个<code>nil</code>。</p><p>换句话说，这样做毫无意义。那么，到底什么才是正确的<code>recover</code>函数用法呢？这就不得不提到<code>defer</code>语句了。</p><p>顾名思义，<code>defer</code>语句就是被用来延迟执行代码的。延迟到什么时候呢？这要延迟到该语句所在的函数即将执行结束的那一刻，无论结束执行的原因是什么。</p><p>这与<code>go</code>语句有些类似，一个<code>defer</code>语句总是由一个<code>defer</code>关键字和一个调用表达式组成。</p><p>这里存在一些限制，有一些调用表达式是不能出现在这里的，包括：针对Go语言内建函数的调用表达式，以及针对<code>unsafe</code>包中的函数的调用表达式。</p><p>顺便说一下，对于<code>go</code>语句中的调用表达式，限制也是一样的。另外，在这里被调用的函数可以是有名称的，也可以是匿名的。我们可以把这里的函数叫做<code>defer</code>函数或者延迟函数。注意，被延迟执行的是<code>defer</code>函数，而不是<code>defer</code>语句。</p><p>我刚才说了，无论函数结束执行的原因是什么，其中的<code>defer</code>函数调用都会在它即将结束执行的那一刻执行。即使导致它执行结束的原因是一个panic也会是这样。正因为如此，我们需要联用<code>defer</code>语句和<code>recover</code>函数调用，才能够恢复一个已经发生的panic。</p><p>我们来看一下经过修正的代码。</p><pre><code>package main

import (
 &quot;fmt&quot;
 &quot;errors&quot;
)

func main() {
 fmt.Println(&quot;Enter function main.&quot;)
 defer func(){
  fmt.Println(&quot;Enter defer function.&quot;)
  if p := recover(); p != nil {
   fmt.Printf(&quot;panic: %s\n&quot;, p)
  }
  fmt.Println(&quot;Exit defer function.&quot;)
 }()
 // 引发panic。
 panic(errors.New(&quot;something wrong&quot;))
 fmt.Println(&quot;Exit function main.&quot;)
}
</code></pre><p>在这个<code>main</code>函数中，我先编写了一条<code>defer</code>语句，并在<code>defer</code>函数中调用了<code>recover</code>函数。仅当调用的结果值不为<code>nil</code>时，也就是说只有panic确实已发生时，我才会打印一行以“panic:”为前缀的内容。</p><p>紧接着，我调用了<code>panic</code>函数，并传入了一个<code>error</code>类型值。这里一定要注意，我们要尽量把<code>defer</code>语句写在函数体的开始处，因为在引发panic的语句之后的所有语句，都不会有任何执行机会。</p><p>也只有这样，<code>defer</code>函数中的<code>recover</code>函数调用才会拦截，并恢复<code>defer</code>语句所属的函数，及其调用的代码中发生的所有panic。</p><p>至此，我向你展示了两个很典型的<code>recover</code>函数的错误用法，以及一个基本的正确用法。</p><p>我希望你能够记住错误用法背后的缘由，同时也希望你能真正地理解联用<code>defer</code>语句和<code>recover</code>函数调用的真谛。</p><p>在命令源码文件demo50.go中，我把上述三种用法合并在了一段代码中。你可以运行该文件，并体会各种用法所产生的不同效果。</p><p>下面我再来多说一点关于<code>defer</code>语句的事情。</p><h3>问题 3：如果一个函数中有多条<code>defer</code>语句，那么那几个<code>defer</code>函数调用的执行顺序是怎样的？</h3><p>如果只用一句话回答的话，那就是：在同一个函数中，<code>defer</code>函数调用的执行顺序与它们分别所属的<code>defer</code>语句的出现顺序（更严谨地说，是执行顺序）完全相反。</p><p>当一个函数即将结束执行时，其中的写在最下边的<code>defer</code>函数调用会最先执行，其次是写在它上边、与它的距离最近的那个<code>defer</code>函数调用，以此类推，最上边的<code>defer</code>函数调用会最后一个执行。</p><p>如果函数中有一条<code>for</code>语句，并且这条<code>for</code>语句中包含了一条<code>defer</code>语句，那么，显然这条<code>defer</code>语句的执行次数，就取决于<code>for</code>语句的迭代次数。</p><p>并且，同一条<code>defer</code>语句每被执行一次，其中的<code>defer</code>函数调用就会产生一次，而且，这些函数调用同样不会被立即执行。</p><p>那么问题来了，这条<code>for</code>语句中产生的多个<code>defer</code>函数调用，会以怎样的顺序执行呢？</p><p>为了彻底搞清楚，我们需要弄明白<code>defer</code>语句执行时发生的事情。</p><p>其实也并不复杂，在<code>defer</code>语句每次执行的时候，Go语言会把它携带的<code>defer</code>函数及其参数值另行存储到一个链表中。</p><p>这个链表与该<code>defer</code>语句所属的函数是对应的，并且，它是先进后出（FILO）的，相当于一个栈。</p><p>在需要执行某个函数中的<code>defer</code>函数调用的时候，Go语言会先拿到对应的链表，然后从该链表中一个一个地取出<code>defer</code>函数及其参数值，并逐个执行调用。</p><p>这正是我说“<code>defer</code>函数调用与其所属的<code>defer</code>语句的执行顺序完全相反”的原因了。</p><p>下面该你出场了，我在demo51.go文件中编写了一个与本问题有关的示例，其中的核心代码很简单，只有几行而已。</p><p>我希望你先查看代码，然后思考并写下该示例被运行时，会打印出哪些内容。</p><p>如果你实在想不出来，那么也可以先运行示例，再试着解释打印出的内容。总之，你需要完全搞明白那几行内容为什么会以那样的顺序出现的确切原因。</p><h2>总结</h2><p>我们这两期的内容主要讲了两个函数和一条语句。<code>recover</code>函数专用于恢复panic，并且调用即恢复。</p><p>它在被调用时会返回一个空接口类型的结果值。如果在调用它时并没有panic发生，那么这个结果值就会是<code>nil</code>。</p><p>而如果被恢复的panic是我们通过调用<code>panic</code>函数引发的，那么它返回的结果值就会是我们传给<code>panic</code>函数参数值的副本。</p><p>对<code>recover</code>函数的调用只有在<code>defer</code>语句中才能真正起作用。<code>defer</code>语句是被用来延迟执行代码的。</p><p>更确切地说，它会让其携带的<code>defer</code>函数的调用延迟执行，并且会延迟到该<code>defer</code>语句所属的函数即将结束执行的那一刻。</p><p>在同一个函数中，延迟执行的<code>defer</code>函数调用，会与它们分别所属的<code>defer</code>语句的执行顺序完全相反。还要注意，同一条<code>defer</code>语句每被执行一次，就会产生一个延迟执行的<code>defer</code>函数调用。</p><p>这种情况在<code>defer</code>语句与<code>for</code>语句联用时经常出现。这时更要关注<code>for</code>语句中，同一条<code>defer</code>语句产生的多个<code>defer</code>函数调用的实际执行顺序。</p><p>以上这些，就是关于Go语言中特殊的程序异常，及其处理方式的核心知识。这里边可以衍生出很多面试题目。</p><h2>思考题</h2><p>我们可以在<code>defer</code>函数中恢复panic，那么可以在其中引发panic吗？</p><p><a href="https://github.com/hyper0x/Golang_Puzzlers">戳此查看Go语言专栏文章配套详细代码。</a></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/32/09/669e21db.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>wesleydeng</span>
  </div>
  <div class="_2_QraFYR_0">从语言设计上，不使用try-catch而是用defer-recover有什么优势？c++和java作为先驱都使用try-catch，也比较清晰，为什么go作为新语言却要发明一个这样的新语法？有何设计上的考量？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这是两种完全不同的异常处理机制。Go语言的异常处理机制是两层的，defer和recover可以处理意外的的异常，而error接口及相关体系处理可预期的异常。Go语言把不同种类的异常完全区别对待，我觉得这是一个进步。<br><br>另外，defer机制能够处理的远不止异常，还有很多资源回收的任务可以用到它。defer机制和goroutine机制一样，是一种很有效果的创新。<br><br>我认为defer机制正是建立在goroutine机制之上的。因为每个函数都有可能成为go函数，所以必须要把异常处理做到函数级别。可以看到，defer机制和error机制都是以函数为边界的。前者在函数级别上阻止会导致非正常控制流的意外异常外溢，而后者在函数级别上用正常的控制流向外传递可预期异常。<br><br>不要说什么先驱，什么旧例，世界在进步，技术更是在猛进。不要把思维固化在某门或某些编程语言上。每种能够流行起来的语言都会有自己独有的、已经验证的语法、风格和哲学。<br><br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-09 08:23:49</div>
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
  <div class="_2_QraFYR_0">defer其实是预调用，产生一个函数对象，压栈保存，函数退出时依次取出执行</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-10 09:32:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/63/8c/8e803651.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>凌惜沫</span>
  </div>
  <div class="_2_QraFYR_0">如果defer中引发panic，那么在该段defer函数之前，需要另外一个defer来捕获该panic，并且代码中最后一个panic会被抛弃，由defer中的panic来成为最后的异常返回。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，是的，由于之前发生的 panic 已经被 recover 了，所以最终被抛出去的就应该是外层 defer 语句中的那个 panic。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-22 22:53:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/6a/89/3cac9f83.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小龙虾</span>
  </div>
  <div class="_2_QraFYR_0">我感觉还是go的这种设计好用，它会强迫开发者区别对待错误和异常，并做出不同的处理。相比try{}catch，我在开发中经常看到开发者把大段大段的代码或者整个处理写到try{}中，这本身就是对try{}catch的乱用</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，这是最主要好处。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-23 09:03:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/KFgDEHIEpnSjjGClCeqmKYJsSOQo40BMHRTtNYrWyQP9WypAjTToplVND944one2pkEyH5Oib4m4wUOJ9xBEIZQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>sket</span>
  </div>
  <div class="_2_QraFYR_0">感觉还是try{}catch这种异常处理好用</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-27 11:07:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/a9/90/0c5ed3d9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>颇忒妥</span>
  </div>
  <div class="_2_QraFYR_0">作者想把概念给我们讲清楚，但是我总觉着看着费劲。为啥？因为作者太啰嗦了。比如：<br>defer函数调用的执行顺序与它们分别所属的defer语句的出现顺序（更严谨地说，是执行顺序）完全相反。<br>改成这样就简单多了：defer函数的调用顺序与其defer语句执行顺序相反。<br>还有：当一个函数即将结束执行时，其中的写在最下边的defer函数调用会最先执行，其次是写在它上边、与它的距离最近的那个defer函数调用，以此类推，最上边的defer函数调用会最后一个执行。<br>改成：当一个函数即将执行结束时，最下面的defer函数先执行，然后是倒数第二个，以此类推。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-05 14:04:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/27/fc/b8d83d56.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>liangjf</span>
  </div>
  <div class="_2_QraFYR_0">Go 语言会把它携带的defer函数及其参数值另行存储到一个队列中。<br><br>这个队列与该defer语句所属的函数是对应的，并且，它是先进后出（FILO）的，相当于一个栈<br><br><br>直接表达为  创建defer时“函数对象“压栈，panic触发时出栈调用   更容易理解吧</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-02-25 19:01:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/60/ec/11cf22de.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>名:海东</span>
  </div>
  <div class="_2_QraFYR_0">&#47;&#47;测试场景1<br>func Test()  {<br>	defer func() {<br>		if errRecover := recover(); errRecover != nil {<br>			fmt.Println(&quot;recover2...&quot;)<br>		}<br>		fmt.Println(&quot;no recover2...&quot;)<br>	}()<br>	defer func() {<br>		test01()  &#47;&#47; test01()方法在defer func(){}中执行<br>	}()<br>	b := 0<br>	a := 1 &#47; b<br>	fmt.Println(a)<br>	return<br>}<br><br>func test01() {<br>	if e := recover(); e != nil {<br>		fmt.Println(&quot;recover...&quot;)<br>	} else {<br>		fmt.Println(&quot;no recover...&quot;)<br>	}<br>	fmt.Println(&quot;defer exe...&quot;)<br>}<br><br>func main() {<br>	Test()<br>}<br>&#47;&#47;输出：<br>no recover...<br>defer exe...<br>recover2...<br>no recover2...<br><br>&#47;&#47;测试场景2<br>func Test()  {<br>	defer func() {<br>		if errRecover := recover(); errRecover != nil {<br>			fmt.Println(&quot;recover2...&quot;)<br>		}<br>		fmt.Println(&quot;no recover2...&quot;)<br>	}()<br>	defer test01()  &#47;&#47;test01()直接放到defer后面 <br>	b := 0<br>	a := 1 &#47; b<br>	fmt.Println(a)<br>	return<br>}<br><br>func test01() {<br>	if e := recover(); e != nil {<br>		fmt.Println(&quot;recover...&quot;)<br>	} else {<br>		fmt.Println(&quot;no recover...&quot;)<br>	}<br>	fmt.Println(&quot;defer exe...&quot;)<br>}<br><br>func main() {<br>	Test()<br>}<br>&#47;&#47;输出：<br>recover...<br>defer exe...<br>no recover2...<br><br>我的问题是：为什么场景1中出现panic没有在defer func() {<br>		test01()<br>	}()中被recover，而在defer func() {<br>		if errRecover := recover(); errRecover != nil {<br>			fmt.Println(&quot;recover2...&quot;)<br>		}<br>		fmt.Println(&quot;no recover2...&quot;)<br>	}()中被recover。<br>场景2使用defer test01 的写法后就可以被recover。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 很简单，在场景一中，test01 函数不是一个 defer 函数（它只是被 defer 函数调用了而已）；而在场景二中，test01 函数却是一个不折不扣的 defer 函数。只有直接在 defer 函数中调用 recover() 函数才能起到恢复 panic 的作用。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-07-08 11:14:02</div>
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
  <div class="_2_QraFYR_0">试验了一下在 goroutine 里面 panic，其他的 goroutine（比如main）是 recover()不到的：<br><br>func main() {<br>	fmt.Println(&quot;start&quot;)<br>	defer func() {<br>		if p := recover(); p != nil {<br>			fmt.Println(p)<br>		}<br>	}()<br>	var wg sync.WaitGroup<br>	wg.Add(1)<br>	go func() {<br>		defer func() {<br>		    wg.Done()<br>		}()<br>		panic(errors.New(&quot;panic in goroutine&quot;))<br><br>	}()<br>	wg.Wait()<br>}</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 当然。因为它们之间不是串行的关系，所以 panic 传播不到其他的 goroutine 那里。所以，每个 goroutine 都应该有自己的异常处理代码。我们可以设计一个整体的异常处理规则或体系，并在每个 goroutine 里都遵循它。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-07 08:17:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/21/0e/b2c7469c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>翼江亭赋</span>
  </div>
  <div class="_2_QraFYR_0">iava世界里曾经try catch满天飞，现在还能看到不少这种代码，但逐渐大家认同了在去掉这种代码。<br><br>因为大部分catch住异常以后只是打个log再重新throw，这个交给框架代码在最外层catch住以后统一处理即可。非框架代码极少需要处理异常。<br><br>go世界里，err guard满天飞，但大部分的处理也是层层上传。但做不到不用，因为不像try那样去掉catch后会自动往上传递，不检查err的话就丢失了，所以这种代码去不掉。只能继续满天飞。<br><br>底层实现其实都是setjmp，主要的区别之一我认为是go设计者认为java异常的性能代价大。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Go 的 error 其实就是在用普通的控制流来处理异常。但是性能却可以有非常明显的提高。其实不管怎么弄都做不到“羊毛出在猪身上”。不管是让开发者自行处理，还是运行时系统自己控制，都会对程序的流畅度产生影响。这就是程序稳定性和程序流畅度（包括可读性、控制流和性能等）之间的trade off。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-31 10:30:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/c5/44/8ff59fc4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>風華</span>
  </div>
  <div class="_2_QraFYR_0">如果defer中引发panic，那么在该段defer函数之前，需要另外一个defer来捕获该panic，并且代码中最后一个panic会被抛弃，由defer中的panic来成为最后的异常返回。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-17 00:23:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/53/e3/39dcfb11.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>来碗绿豆汤</span>
  </div>
  <div class="_2_QraFYR_0">可以 defer 有点类似java中的final语句，里面还可以抛出异常。这样的好处是，我们捕获panic之后，可以对起内容进行查看，如果不是我们关注的panic那么可以继续抛出去</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对的。不过还是有不少不一样的地方的，可以体会一下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-01 15:00:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/9a/7f/781f89ab.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Zz~</span>
  </div>
  <div class="_2_QraFYR_0">老师，您好，我想问一下，如果在main函数里调用一个我自定义的panic方法，recover可以恢复；但是如果我将自定义的panic方法改为go mypanic这样，recover就不能恢复。这是什么原因呢？下面是我实验的代码<br><br><br>==============可以恢复的==============<br>package main<br><br>import (<br>	&quot;errors&quot;<br>	&quot;fmt&quot;<br>)<br><br>func myRecover() {<br>	if err := recover(); err != nil {<br>		fmt.Printf(&quot;panic is %s&quot;, err)<br>	}<br>}<br><br>func myPanic() {<br>	panic(errors.New(&quot;自定义异常&quot;))<br>}<br><br>func main() {<br>	defer myRecover()<br>	myPanic()<br>}<br><br><br>=================不可以恢复的==============<br>package main<br><br>import (<br>	&quot;errors&quot;<br>	&quot;fmt&quot;<br>	&quot;time&quot;<br>)<br><br>func myRecover() {<br>	if err := recover(); err != nil {<br>		fmt.Printf(&quot;panic is %s&quot;, err)<br>	}<br>}<br><br>func myPanic() {<br>	panic(errors.New(&quot;自定义异常&quot;))<br>}<br><br>func main() {<br>	defer myRecover()<br>	go myPanic()<br>	time.Sleep(time.Second * 5)<br>}<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 某个goroutine中的panic是不可能由别的goroutine中的recover恢复的。或者说，一个goroutine中的panic只能由自己例程中的recover恢复。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-02 13:38:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/aa/30/acc91f01.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>honnkyou</span>
  </div>
  <div class="_2_QraFYR_0">「延迟到什么时候呢？这要延迟到该语句所在的函数即将执行结束的那一刻，无论结束执行的原因是什么。」<br>以该节课中代码为例的话是要吃到main函数快结束时执行是吗？执行defer函数。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: defer 函数的执行时刻是在直接包含它的那个函数即将执行完毕的时候，也可以理解为下一刻就要返回结果值（如果有的话）的时候。对于 main 函数直接包含的 defer 函数来说，也是如此。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-14 00:34:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/b3/8f/4caf7f03.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Bang</span>
  </div>
  <div class="_2_QraFYR_0">请问下，按您上面说的，一个recover只能恢复所在的那个函数。那如果一个 函数中有一个goroutine函数  而这个goroutine函数触发了panic，那是只有他自己可以recover是吗，他的上级是无法recover内部的goroutine的painc是这样吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-22 21:38:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/5f/5d/63010e32.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>有匪君子</span>
  </div>
  <div class="_2_QraFYR_0">这个问题就引发了另一个问题。defer可以在同一个函数中嵌套使用吗？感觉这两个问题答案应该一致</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 只要是函数就都可以。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-01 10:56:16</div>
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
  <div class="_2_QraFYR_0">今日总结 <br>本章主要是讲了panic和recover以及defer<br>panic主要是用来抛出恐慌<br>而recover主要用来恢复恐慌，但是recover使用即恢复 如果在此时没有恐慌产生就会返回一个nil<br>也正是因为recover使用即恢复的特性 所以要把recover执行在产生恐慌之后，但是panic之后的代码不会再执行所以引入defer表达式<br>defer表达式 主要是用来代码延迟执行 延迟到函数结束，所以一般把recover跟defer搭配 <br>defer的执行 没执行一次defer语句产生一个defer函数 并且将其放入一个额外的栈中 所以是个先入后出的顺序!所以defer 函数的执行顺序与其声明顺序完全相反<br>关于思考题:<br>我觉得从理论上来说 我们没必要完全人为的引起panic 但是如果是不小心引起的panic那也无法避免,同时通过测试也是可以引发panic的</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-26 15:14:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/74/57/7b828263.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张sir</span>
  </div>
  <div class="_2_QraFYR_0">您好，请问defer函数压𣏾的时候，为什么把当时的入参也放入𣏾中呢<br>```<br>func test() {<br>	var a, b = 1, 1<br>	defer func(flag int) {<br>		fmt.Println(flag)<br>	}(a + b)<br><br>	a = 2<br>	b = 2<br><br>}<br><br>```<br><br>这个输出的２不是４的理论论据是什么呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 参数值flag是2，因为Go会在压栈前先求出给予defer函数的那些参数值。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-12 14:43:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/82/fb/9d232a7a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Pana</span>
  </div>
  <div class="_2_QraFYR_0">在defer 中 recover 了panic ，是否还能让函数返回错误呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以，函数需要有一个有名称的error类型的结果，然后在该函数内recover之后为这个结果赋值。这样就可以了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-05 10:35:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/b3/8f/4caf7f03.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Bang</span>
  </div>
  <div class="_2_QraFYR_0">请问下，按您上面说的，一个recover只能恢复所在的那个函数。那如果一个 函数中有一个goroutine函数  而这个goroutine函数触发了panic，那是只有他自己可以recover是吗，他的上级是无法recover内部的goroutine的painc是这样吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-22 21:38:23</div>
  </div>
</div>
</div>
</li>
</ul>