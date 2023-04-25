<audio title="21 _ panic函数、recover函数以及defer语句 （上）" src="https://static001.geekbang.org/resource/audio/fb/86/fb1424dc84fb9487b84c18b5d3049586.mp3" controls="controls"></audio> 
<p>我在上两篇文章中，详细地讲述了Go语言中的错误处理，并从两个视角为你总结了错误类型、错误值的处理技巧和设计方式。</p><p>在本篇，我要给你展示Go语言的另外一种错误处理方式。不过，严格来说，它处理的不是错误，而是异常，并且是一种在我们意料之外的程序异常。</p><h2>前导知识：运行时恐慌panic</h2><p>这种程序异常被叫做panic，我把它翻译为运行时恐慌。其中的“恐慌”二字是由panic直译过来的，而之所以前面又加上了“运行时”三个字，是因为这种异常只会在程序运行的时候被抛出来。</p><p>我们举个具体的例子来看看。</p><p>比如说，一个Go程序里有一个切片，它的长度是5，也就是说该切片中的元素值的索引分别为<code>0</code>、<code>1</code>、<code>2</code>、<code>3</code>、<code>4</code>，但是，我在程序里却想通过索引<code>5</code>访问其中的元素值，显而易见，这样的访问是不正确的。</p><p>Go程序，确切地说是程序内嵌的Go语言运行时系统，会在执行到这行代码的时候抛出一个“index out of range”的panic，用以提示你索引越界了。</p><p>当然了，这不仅仅是个提示。当panic被抛出之后，如果我们没有在程序里添加任何保护措施的话，程序（或者说代表它的那个进程）就会在打印出panic的详细情况（以下简称panic详情）之后，终止运行。</p><!-- [[[read_end]]] --><p>现在，就让我们来看一下这样的panic详情中都有什么。</p><pre><code>panic: runtime error: index out of range

goroutine 1 [running]:
main.main()
 /Users/haolin/GeekTime/Golang_Puzzlers/src/puzzlers/article19/q0/demo47.go:5 +0x3d
exit status 2
</code></pre><p>这份详情的第一行是“panic: runtime error: index out of range”。其中的“runtime error”的含义是，这是一个<code>runtime</code>代码包中抛出的panic。在这个panic中，包含了一个<code>runtime.Error</code>接口类型的值。<code>runtime.Error</code>接口内嵌了<code>error</code>接口，并做了一点点扩展，<code>runtime</code>包中有不少它的实现类型。</p><p>实际上，此详情中的“panic：”右边的内容，正是这个panic包含的<code>runtime.Error</code>类型值的字符串表示形式。</p><p>此外，panic详情中，一般还会包含与它的引发原因有关的goroutine的代码执行信息。正如前述详情中的“goroutine 1 [running]”，它表示有一个ID为<code>1</code>的goroutine在此panic被引发的时候正在运行。</p><p>注意，这里的ID其实并不重要，因为它只是Go语言运行时系统内部给予的一个goroutine编号，我们在程序中是无法获取和更改的。</p><p>我们再看下一行，“main.main()”表明了这个goroutine包装的<code>go</code>函数就是命令源码文件中的那个<code>main</code>函数，也就是说这里的goroutine正是主goroutine。再下面的一行，指出的就是这个goroutine中的哪一行代码在此panic被引发时正在执行。</p><p>这包含了此行代码在其所属的源码文件中的行数，以及这个源码文件的绝对路径。这一行最后的<code>+0x3d</code>代表的是：此行代码相对于其所属函数的入口程序计数偏移量。不过，一般情况下它的用处并不大。</p><p>最后，“exit status 2”表明我的这个程序是以退出状态码<code>2</code>结束运行的。在大多数操作系统中，只要退出状态码不是<code>0</code>，都意味着程序运行的非正常结束。在Go语言中，因panic导致程序结束运行的退出状态码一般都会是<code>2</code>。</p><p>综上所述，我们从上边的这个panic详情可以看出，作为此panic的引发根源的代码处于demo47.go文件中的第5行，同时被包含在<code>main</code>包（也就是命令源码文件所在的代码包）的<code>main</code>函数中。</p><p>那么，我的第一个问题也随之而来了。我今天的问题是：<strong>从panic被引发到程序终止运行的大致过程是什么？</strong></p><p><strong>这道题的典型回答是这样的。</strong></p><p>我们先说一个大致的过程：某个函数中的某行代码有意或无意地引发了一个panic。这时，初始的panic详情会被建立起来，并且该程序的控制权会立即从此行代码转移至调用其所属函数的那行代码上，也就是调用栈中的上一级。</p><p>这也意味着，此行代码所属函数的执行随即终止。紧接着，控制权并不会在此有片刻的停留，它又会立即转移至再上一级的调用代码处。控制权如此一级一级地沿着调用栈的反方向传播至顶端，也就是我们编写的最外层函数那里。</p><p>这里的最外层函数指的是<code>go</code>函数，对于主goroutine来说就是<code>main</code>函数。但是控制权也不会停留在那里，而是被Go语言运行时系统收回。</p><p>随后，程序崩溃并终止运行，承载程序这次运行的进程也会随之死亡并消失。与此同时，在这个控制权传播的过程中，panic详情会被逐渐地积累和完善，并会在程序终止之前被打印出来。</p><h2>问题解析</h2><p>panic可能是我们在无意间（或者说一不小心）引发的，如前文所述的索引越界。这类panic是真正的、在我们意料之外的程序异常。不过，除此之外，我们还是可以有意地引发panic。</p><p>Go语言的内建函数<code>panic</code>是专门用于引发panic的。<code>panic</code>函数使程序开发者可以在程序运行期间报告异常。</p><p>注意，这与从函数返回错误值的意义是完全不同的。当我们的函数返回一个非<code>nil</code>的错误值时，函数的调用方有权选择不处理，并且不处理的后果往往是不致命的。</p><p>这里的“不致命”的意思是，不至于使程序无法提供任何功能（也可以说僵死）或者直接崩溃并终止运行（也就是真死）。</p><p>但是，当一个panic发生时，如果我们不施加任何保护措施，那么导致的直接后果就是程序崩溃，就像前面描述的那样，这显然是致命的。</p><p>为了更清楚地展示答案中描述的过程，我编写了demo48.go文件。你可以先查看一下其中的代码，再试着运行它，并体会它打印的内容所代表的含义。</p><p>我在这里再提示一点。panic详情会在控制权传播的过程中，被逐渐地积累和完善，并且，控制权会一级一级地沿着调用栈的反方向传播至顶端。</p><p>因此，在针对某个goroutine的代码执行信息中，调用栈底端的信息会先出现，然后是上一级调用的信息，以此类推，最后才是此调用栈顶端的信息。</p><p>比如，<code>main</code>函数调用了<code>caller1</code>函数，而<code>caller1</code>函数又调用了<code>caller2</code>函数，那么<code>caller2</code>函数中代码的执行信息会先出现，然后是<code>caller1</code>函数中代码的执行信息，最后才是<code>main</code>函数的信息。</p><pre><code>goroutine 1 [running]:
main.caller2()
 /Users/haolin/GeekTime/Golang_Puzzlers/src/puzzlers/article19/q1/demo48.go:22 +0x91
main.caller1()
 /Users/haolin/GeekTime/Golang_Puzzlers/src/puzzlers/article19/q1/demo48.go:15 +0x66
main.main()
 /Users/haolin/GeekTime/Golang_Puzzlers/src/puzzlers/article19/q1/demo48.go:9 +0x66
exit status 2
</code></pre><p><img src="https://static001.geekbang.org/resource/image/60/d7/606ff433a6b58510f215e57792822bd7.png?wh=887*1060" alt=""></p><p>（从panic到程序崩溃）</p><p>好了，到这里，我相信你已经对panic被引发后的程序终止过程有一定的了解了。深入地了解此过程，以及正确地解读panic详情应该是我们的必备技能，这在调试Go程序或者为Go程序排查错误的时候非常重要。</p><h2>总结</h2><p>最近的两篇文章，我们是围绕着panic函数、recover函数以及defer语句进行的。今天我主要讲了panic函数。这个函数是专门被用来引发panic的。panic也可以被称为运行时恐慌，它是一种只能在程序运行期间抛出的程序异常。</p><p>Go语言的运行时系统可能会在程序出现严重错误时自动地抛出panic，我们在需要时也可以通过调用<code>panic</code>函数引发panic。但不论怎样，如果不加以处理，panic就会导致程序崩溃并终止运行。</p><h2>思考题</h2><p>一个函数怎样才能把panic转化为<code>error</code>类型值，并将其作为函数的结果值返回给调用方？</p><p><a href="https://github.com/hyper0x/Golang_Puzzlers">戳此查看Go语言专栏文章配套详细代码。</a></p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/21/b8/aca814dd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>江山如画</span>
  </div>
  <div class="_2_QraFYR_0">一个函数如果要把 panic 转化为error类型值，并将其结果返回给调用方，可以考虑把 defer 语句封装到一个匿名函数之中，下面是实验的一个例子，所用函数是一个除法函数，当除数为0的时候会抛出 panic并捕获。<br><br>func divide(a, b int) (res int, err error) {<br>	func() {<br>		defer func() {<br>			if rec := recover(); rec != nil {<br>				err = fmt.Errorf(&quot;%s&quot;, rec)<br>			}<br>		}()<br>		res = a &#47; b<br>	}()<br>	return<br>}<br><br>func main() {<br>	res, err := divide(1, 0)<br>	fmt.Println(res, err) &#47;&#47; 0 runtime error: integer divide by zero<br><br>	res, err = divide(2, 1)<br>	fmt.Println(res, err) &#47;&#47; 2 &lt;nil&gt;<br>}<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-08 18:59:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/95/dc/07195a63.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>锋</span>
  </div>
  <div class="_2_QraFYR_0">老师，你好，我有一个疑问，请教一下，谢谢~<br>Go在设计的时候没有设计try...catch...finally这样的方式来捕获异常。<br>我在网上查很多人用panic、defer和recover组合来实现异常的捕获，甚至很多都将这个二次封装之后作为一个库来进行使用。<br>我的疑问是，从Go的设计角度为什么要这么做？是出于什么样的目的，还是他俩之间有什么优劣？<br>非常感谢~，烦请解答。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Go的错误处理机制是由两个部分组成的，panic代表着特殊的（或者说意外的）错误，error代表着普通的错误。与try-catch不同，error并不是打断正常的控制流程的执行。单单这一点来讲，就已经是非常好的进步了。相比之下，panic会打断正常的控制流程。从这一点上看，panic与try-catch很像。<br><br>说到这里，你可能也意识到了，try-catch是一套行为单一的错误处理机制，而Go语言的（error+panic）把错误处理机制在代码级别分为了两个部分。<br><br>这样的好处是，倒逼开发者去思考，什么时候应该返回普通的错误，什么时候应该抛出意外的错误。这种思考在设计一个程序的错误体系的时候是非常重要的，关系到程序运行的稳定性。<br><br>至于缺点，error容易被滥用，导致程序中到处是 if err != nil 的代码。但是我们要清楚的是，这往往是程序设计上的问题，而不是语言层面的问题。如果不当心，try-catch照样会被弄的满屏都是。而且try-catch还有一个颗粒度和数量的问题（与临界区的颗粒度和数量问题类似）。<br><br>总之，我个人认为Go语言的错误处理机制是一种创新和进步。不过，由于容易被滥用，Go语言团队不是还在近几年一直在考虑更好的解决方案吗。我也很期待他们新的设计。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-06 10:09:19</div>
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
  <div class="_2_QraFYR_0">先使用go中的类似try catch这样的语句，将异常捕获的异常转为相应的错误error就可以了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-28 00:15:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/60/c2/6d50bfdf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>唐丹</span>
  </div>
  <div class="_2_QraFYR_0">郝大，你好，我在golang 8中通过recover处理panic时发现，必须在引发panic的当前协程就处理掉，否则待其传递到父协程直至main方法中，都不能通过recover成功处理掉了，程序会因此结束。请问这样设计的原因是什么？那么协程是通过panic中记录的协程id来区分是不是在当前协程引发的panic的吗？另外，这样的话，我们应用程序中每一个通过go新起的协程都应该在开始的地方recover，否则即使父协程有recover也不能阻止程序因为一个意外的panic而挂掉？盼望解答，谢谢🙏</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 只要在调用栈路径上就都可以处理，如果你用了defer语句和recover函数等正确处理方式还是不行的话，就要看看这个panic是不是不了恢复的。一些runtime抛出来的panic是不可恢复的，因为问题很严重必须整改代码才行。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-28 07:55:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLia2EwyyEVs3tWRnMlqaAG7R7HvlW4vGvxthKsicgsCEeXO1qL7mMy6GAzgdkSKcH3c70Qa2hY3JLw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>沐夜星光</span>
  </div>
  <div class="_2_QraFYR_0">“控制权如此一级一级地沿着调用栈的反方向传播至顶端，也就是我们编写的最外层函数那里”。最外层函数是go函数，也就说当panic触发，通过其所在的goroutine，将控制权转移给运行时系统时，是不一定经过main函数的吗？另外老师能不能讲讲，go是怎么回收一个进程的，怎么处理运行中的goroutine以及涉及的资源。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这会经过main函数，异步调用也是调用。main函数就是主goroutine的go函数。<br><br>go回收进程？这没什么稀奇的啊，就是调用操作系统的底层API，你可以查看 runtime.main 函数了解相关过程，或者查看 runtime.exit 函数了解进程退出时调用的API。<br><br>在一个goroutine中的go函数执行完之后，这个goroutine会转为_Gidle状态，其中的所有关键字段都会被重置，它的栈内存也会被释放。最后，它会被放入自由G列表。你可以通过查看 runtime.goexit0 了解到。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-21 19:51:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/99/c9/a7c77746.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>冰激凌的眼泪</span>
  </div>
  <div class="_2_QraFYR_0">panic时，会捕获异常及异常上下文（函数名+文件行）<br>类似看作有一个异常上下文列表，始于异常触发处，沿着函数调用逆向展开，每一级append自己的异常上下文，直至goroutine入口函数，最终被runtime捕获<br>最终异常信息被打印，异常上下文列表被顺序打印，程序退出</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-02 07:42:09</div>
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
  <div class="_2_QraFYR_0">https:&#47;&#47;gist.github.com&#47;bwangelme&#47;9ce1c606ba9f69c72f82722adf1402e1</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-03 12:34:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/45/31/53910b61.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>A 凡</span>
  </div>
  <div class="_2_QraFYR_0">之前自己学习时候的一些模糊点更加清晰了，支持！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-24 18:31:17</div>
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
  <div class="_2_QraFYR_0">初学go都会吐槽说没有 try catch , 应该不止我一个</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 每种语言的风格都不同啊，习惯习惯就好了（注意力可以更多的放在“怎样高效构建优秀软件上” ，语法的一些特点不属于关键问题 :-) ），而且后续Go在这方面会有改善。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-25 21:19:15</div>
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
  <div class="_2_QraFYR_0">这个看起来比较简单<br>func main() {<br>	res, err := divide(1, 0)<br>	if err != nil {<br>		fmt.Println(err)<br>	}<br>	fmt.Println(res)<br>}<br><br>func divide(a, b int) (res int, err error) {<br>	defer func() {<br>		if r := recover(); r != nil {<br>			err = fmt.Errorf(&quot;omg, panic ! err:%v&quot;, r)<br>			return<br>		}<br>	}()<br>	res = a &#47; b<br>	return<br>}</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-07 22:34:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83erpYZalYvFGcBs7zZvYwaQAZwTLiaw0mycJ4PdYpP3VxAYkAtyIRHhjSOrOK0yESaPpgEbVQUwf6LA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Harlan</span>
  </div>
  <div class="_2_QraFYR_0">go这种满屏幕都是 判断 err!=nil 这种没有意义的代码  代码侵入性极强  也不优雅</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-12 11:37:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/PeibZXsEcwic1zvrAQpDlnNlPxZvmAtIZ6XCenC8NaPbVVfCk7PXgAYzb8icqrYlb9cJd82hia9FYTicxqSdgyCEP4w/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_37a441</span>
  </div>
  <div class="_2_QraFYR_0">老师，你好，我想问下，panic触发的时机，比如指令执行过程中，在什么时候会调用到相关的panic，比如数组越界是什么时候调用runtime.panicIndex，是有个额外的线程不断检测有异常了吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 当然不是了，是执行程序的程序碰到严重错误就抛出来，属于Go运行时系统的职责范围。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-15 10:55:21</div>
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
  <div class="_2_QraFYR_0">今日总结<br>今天主要是讲了panic运行时恐慌，一般发生在运行时<br>同样也可以自己调用内置的panic来主动引起恐慌 一般用来自己调试程序异常等等<br>主要掌握panic的执行过程 <br>首先从某一行引起了panic 然后返回到其调用函数 这样一级一级的返回 直到最顶层也即main函数中 最后把控制权交给了go运行时状态，最后程序奔溃 进程退出<br>panic的信息一般也在这个返回过程中不断的完善<br>panic: runtime error: index out of range &#47;&#47;错误信息<br>&#47;&#47;以下是调用堆栈<br>goroutine 1 [running]:<br>main.caller2()<br>        &#47;home&#47;ubuntu&#47;geek_go&#47;Golang_Puzzlers&#47;src&#47;puzzlers&#47;article19&#47;q1&#47;demo48.go:22 +0x91<br>main.caller1()<br>        &#47;home&#47;ubuntu&#47;geek_go&#47;Golang_Puzzlers&#47;src&#47;puzzlers&#47;article19&#47;q1&#47;demo48.go:15 +0x66<br>main.main()<br>        &#47;home&#47;ubuntu&#47;geek_go&#47;Golang_Puzzlers&#47;src&#47;puzzlers&#47;article19&#47;q1&#47;demo48.go:9 +0x66<br>exit status 2<br>关于思考题 我也先想了一下 我们捕获异常并将错误信息封装到一个error当中最后返回这个err<br>后来又去翻了解答 好像就是这个 样子的</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-25 19:51:04</div>
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
    <div class="_3Hkula0k_0">2019-01-21 19:08:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/5d/f8/62a8b90d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>melody_future</span>
  </div>
  <div class="_2_QraFYR_0">panic、recover 有点像try、、catch。这样应该会好理解很多</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-25 11:27:58</div>
  </div>
</div>
</div>
</li>
</ul>