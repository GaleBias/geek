<audio title="18 _ if语句、for语句和switch语句" src="https://static001.geekbang.org/resource/audio/b4/f4/b4aac5abbe96b64f4e3f62a81a0abff4.mp3" controls="controls"></audio> 
<p>在上两篇文章中，我主要为你讲解了与<code>go</code>语句、goroutine和Go语言调度器有关的知识和技法。</p><p>内容很多，你不用急于完全消化，可以在编程实践过程中逐步理解和感悟，争取夯实它们。</p><hr></hr><p>现在，让我们暂时走下神坛，回归民间。我今天要讲的<code>if</code>语句、<code>for</code>语句和<code>switch</code>语句都属于Go语言的基本流程控制语句。它们的语法看起来很朴素，但实际上也会有一些使用技巧和注意事项。我在本篇文章中会以一系列面试题为线索，为你讲述它们的用法。</p><p>那么，<strong>今天的问题是：使用携带<code>range</code>子句的<code>for</code>语句时需要注意哪些细节？</strong> 这是一个比较笼统的问题。我还是通过编程题来讲解吧。</p><blockquote>
<p><span class="reference">本问题中的代码都被放在了命令源码文件demo41.go的<code>main</code>函数中的。为了专注问题本身，本篇文章中展示的编程题会省略掉一部分代码包声明语句、代码包导入语句和<code>main</code>函数本身的声明部分。</span></p>
</blockquote><pre><code>numbers1 := []int{1, 2, 3, 4, 5, 6}
for i := range numbers1 {
	if i == 3 {
		numbers1[i] |= i
	}
}
fmt.Println(numbers1)
</code></pre><p>我先声明了一个元素类型为<code>int</code>的切片类型的变量<code>numbers1</code>，在该切片中有6个元素值，分别是从<code>1</code>到<code>6</code>的整数。我用一条携带<code>range</code>子句的<code>for</code>语句去迭代<code>numbers1</code>变量中的所有元素值。</p><p>在这条<code>for</code>语句中，只有一个迭代变量<code>i</code>。我在每次迭代时，都会先去判断<code>i</code>的值是否等于<code>3</code>，如果结果为<code>true</code>，那么就让<code>numbers1</code>的第<code>i</code>个元素值与<code>i</code>本身做按位或的操作，再把操作结果作为<code>numbers1</code>的新的第<code>i</code>个元素值。最后我会打印出<code>numbers1</code>的值。</p><!-- [[[read_end]]] --><p>所以具体的问题就是，这段代码执行后会打印出什么内容？</p><p>这里的<strong>典型回答</strong>是：打印的内容会是<code>[1 2 3 7 5 6]</code>。</p><h2>问题解析</h2><p>你心算得到的答案是这样吗？让我们一起来复现一下这个计算过程。</p><p>当<code>for</code>语句被执行的时候，在<code>range</code>关键字右边的<code>numbers1</code>会先被求值。</p><p>这个位置上的代码被称为<code>range</code>表达式。<code>range</code>表达式的结果值可以是数组、数组的指针、切片、字符串、字典或者允许接收操作的通道中的某一个，并且结果值只能有一个。</p><p>对于不同种类的<code>range</code>表达式结果值，<code>for</code>语句的迭代变量的数量可以有所不同。</p><p>就拿我们这里的<code>numbers1</code>来说，它是一个切片，那么迭代变量就可以有两个，右边的迭代变量代表当次迭代对应的某一个元素值，而左边的迭代变量则代表该元素值在切片中的索引值。</p><p>那么，如果像本题代码中的<code>for</code>语句那样，只有一个迭代变量的情况意味着什么呢？这意味着，该迭代变量只会代表当次迭代对应的元素值的索引值。</p><p>更宽泛地讲，当只有一个迭代变量的时候，数组、数组的指针、切片和字符串的元素值都是无处安放的，我们只能拿到按照从小到大顺序给出的一个个索引值。</p><p>因此，这里的迭代变量<code>i</code>的值会依次是从<code>0</code>到<code>5</code>的整数。当<code>i</code>的值等于<code>3</code>的时候，与之对应的是切片中的第4个元素值<code>4</code>。对<code>4</code>和<code>3</code>进行按位或操作得到的结果是<code>7</code>。这就是答案中的第4个整数是<code>7</code>的原因了。</p><p><strong>现在，我稍稍修改一下上面的代码。我们再来估算一下打印内容。</strong></p><pre><code>numbers2 := [...]int{1, 2, 3, 4, 5, 6}
maxIndex2 := len(numbers2) - 1
for i, e := range numbers2 {
	if i == maxIndex2 {
		numbers2[0] += e
	} else {
		numbers2[i+1] += e
	}
}
fmt.Println(numbers2)
</code></pre><p>注意，我把迭代的对象换成了<code>numbers2</code>。<code>numbers2</code>中的元素值同样是从<code>1</code>到<code>6</code>的6个整数，并且元素类型同样是<code>int</code>，但它是一个数组而不是一个切片。</p><p>在<code>for</code>语句中，我总是会对紧挨在当次迭代对应的元素后边的那个元素，进行重新赋值，新的值会是这两个元素的值之和。当迭代到最后一个元素时，我会把此<code>range</code>表达式结果值中的第一个元素值，替换为它的原值与最后一个元素值的和，最后，我会打印出<code>numbers2</code>的值。</p><p><strong>对于这段代码，我的问题依旧是：打印的内容会是什么？你可以先思考一下。</strong></p><p>好了，我要公布答案了。打印的内容会是<code>[7 3 5 7 9 11]</code>。我先来重现一下计算过程。当<code>for</code>语句被执行的时候，在<code>range</code>关键字右边的<code>numbers2</code>会先被求值。</p><p>这里需要注意两点：</p><ol>
<li><code>range</code>表达式只会在<code>for</code>语句开始执行时被求值一次，无论后边会有多少次迭代；</li>
<li><code>range</code>表达式的求值结果会被复制，也就是说，被迭代的对象是<code>range</code>表达式结果值的副本而不是原值。</li>
</ol><p>基于这两个规则，我们接着往下看。在第一次迭代时，我改变的是<code>numbers2</code>的第二个元素的值，新值为<code>3</code>，也就是<code>1</code>和<code>2</code>之和。</p><p>但是，被迭代的对象的第二个元素却没有任何改变，毕竟它与<code>numbers2</code>已经是毫不相关的两个数组了。因此，在第二次迭代时，我会把<code>numbers2</code>的第三个元素的值修改为<code>5</code>，即被迭代对象的第二个元素值<code>2</code>和第三个元素值<code>3</code>的和。</p><p>以此类推，之后的<code>numbers2</code>的元素值依次会是<code>7</code>、<code>9</code>和<code>11</code>。当迭代到最后一个元素时，我会把<code>numbers2</code>的第一个元素的值修改为<code>1</code>和<code>6</code>之和。</p><p>好了，现在该你操刀了。你需要把<code>numbers2</code>的值由一个数组改成一个切片，其中的元素值都不要变。为了避免混淆，你还要把这个切片值赋给变量<code>numbers3</code>，并且把后边代码中所有的<code>numbers2</code>都改为<code>numbers3</code>。</p><p>问题是不变的，执行这段修改版的代码后打印的内容会是什么呢？如果你实在估算不出来，可以先实际执行一下，然后再尝试解释看到的答案。提示一下，切片与数组是不同的，前者是引用类型的，而后者是值类型的。</p><p>我们可以先接着讨论后边的内容，但是我强烈建议你一定要回来，再看看我留给你的这个问题，认真地思考和计算一下。</p><p><strong>知识扩展</strong></p><p><strong>问题1：<code>switch</code>语句中的<code>switch</code>表达式和<code>case</code>表达式之间有着怎样的联系？</strong></p><p>先来看一段代码。</p><pre><code>value1 := [...]int8{0, 1, 2, 3, 4, 5, 6}
switch 1 + 3 {
case value1[0], value1[1]:
	fmt.Println(&quot;0 or 1&quot;)
case value1[2], value1[3]:
	fmt.Println(&quot;2 or 3&quot;)
case value1[4], value1[5], value1[6]:
	fmt.Println(&quot;4 or 5 or 6&quot;)
}
</code></pre><p>我先声明了一个数组类型的变量<code>value1</code>，该变量的元素类型是<code>int8</code>。在后边的<code>switch</code>语句中，被夹在<code>switch</code>关键字和左花括号<code>{</code>之间的是<code>1 + 3</code>，这个位置上的代码被称为<code>switch</code>表达式。这个<code>switch</code>语句还包含了三个<code>case</code>子句，而每个<code>case</code>子句又各包含了一个<code>case</code>表达式和一条打印语句。</p><p>所谓的<code>case</code>表达式一般由<code>case</code>关键字和一个表达式列表组成，表达式列表中的多个表达式之间需要有英文逗号<code>,</code>分割，比如，上面代码中的<code>case value1[0], value1[1]</code>就是一个<code>case</code>表达式，其中的两个子表达式都是由索引表达式表示的。</p><p>另外的两个<code>case</code>表达式分别是<code>case value1[2], value1[3]</code>和<code>case value1[4], value1[5], value1[6]</code>。</p><p>此外，在这里的每个<code>case</code>子句中的那些打印语句，会分别打印出不同的内容，这些内容用于表示<code>case</code>子句被选中的原因，比如，打印内容<code>0 or 1</code>表示当前<code>case</code>子句被选中是因为<code>switch</code>表达式的结果值等于<code>0</code>或<code>1</code>中的某一个。另外两条打印语句会分别打印出<code>2 or 3</code>和<code>4 or 5 or 6</code>。</p><p>现在问题来了，拥有这样三个<code>case</code>表达式的<code>switch</code>语句可以成功通过编译吗？如果不可以，原因是什么？如果可以，那么该<code>switch</code>语句被执行后会打印出什么内容。</p><p>我刚才说过，只要<code>switch</code>表达式的结果值与某个<code>case</code>表达式中的任意一个子表达式的结果值相等，该<code>case</code>表达式所属的<code>case</code>子句就会被选中。</p><p>并且，一旦某个<code>case</code>子句被选中，其中的附带在<code>case</code>表达式后边的那些语句就会被执行。与此同时，其他的所有<code>case</code>子句都会被忽略。</p><p>当然了，如果被选中的<code>case</code>子句附带的语句列表中包含了<code>fallthrough</code>语句，那么紧挨在它下边的那个<code>case</code>子句附带的语句也会被执行。</p><p>正因为存在上述判断相等的操作（以下简称判等操作），<code>switch</code>语句对<code>switch</code>表达式的结果类型，以及各个<code>case</code>表达式中子表达式的结果类型都是有要求的。毕竟，在Go语言中，只有类型相同的值之间才有可能被允许进行判等操作。</p><p>如果<code>switch</code>表达式的结果值是无类型的常量，比如<code>1 + 3</code>的求值结果就是无类型的常量<code>4</code>，那么这个常量会被自动地转换为此种常量的默认类型的值，比如整数<code>4</code>的默认类型是<code>int</code>，又比如浮点数<code>3.14</code>的默认类型是<code>float64</code>。</p><p>因此，由于上述代码中的<code>switch</code>表达式的结果类型是<code>int</code>，而那些<code>case</code>表达式中子表达式的结果类型却是<code>int8</code>，它们的类型并不相同，所以这条<code>switch</code>语句是无法通过编译的。</p><p>再来看一段很类似的代码：</p><pre><code>value2 := [...]int8{0, 1, 2, 3, 4, 5, 6}
switch value2[4] {
case 0, 1:
	fmt.Println(&quot;0 or 1&quot;)
case 2, 3:
	fmt.Println(&quot;2 or 3&quot;)
case 4, 5, 6:
	fmt.Println(&quot;4 or 5 or 6&quot;)
}
</code></pre><p>其中的变量<code>value2</code>与<code>value1</code>的值是完全相同的。但不同的是，我把<code>switch</code>表达式换成了<code>value2[4]</code>，并把下边那三个<code>case</code>表达式分别换为了<code>case 0, 1</code>、<code>case 2, 3</code>和<code>case 4, 5, 6</code>。</p><p>如此一来，<code>switch</code>表达式的结果值是<code>int8</code>类型的，而那些<code>case</code>表达式中子表达式的结果值却是无类型的常量了。这与之前的情况恰恰相反。那么，这样的<code>switch</code>语句可以通过编译吗？</p><p>答案是肯定的。因为，如果<code>case</code>表达式中子表达式的结果值是无类型的常量，那么它的类型会被自动地转换为<code>switch</code>表达式的结果类型，又由于上述那几个整数都可以被转换为<code>int8</code>类型的值，所以对这些表达式的结果值进行判等操作是没有问题的。</p><p>当然了，如果这里说的自动转换没能成功，那么<code>switch</code>语句照样通不过编译。</p><p><img src="https://static001.geekbang.org/resource/image/91/1c/91add0a66b9956f81086285aabc20c1c.png?wh=1402*631" alt=""></p><p>（switch语句中的自动类型转换）</p><p>通过上面这两道题，你应该可以搞清楚<code>switch</code>表达式和<code>case</code>表达式之间的联系了。由于需要进行判等操作，所以前者和后者中的子表达式的结果类型需要相同。</p><p><code>switch</code>语句会进行有限的类型转换，但肯定不能保证这种转换可以统一它们的类型。还要注意，如果这些表达式的结果类型有某个接口类型，那么一定要小心检查它们的动态值是否都具有可比性（或者说是否允许判等操作）。</p><p>因为，如果答案是否定的，虽然不会造成编译错误，但是后果会更加严重：引发panic（也就是运行时恐慌）。</p><p><strong>问题2：<code>switch</code>语句对它的<code>case</code>表达式有哪些约束？</strong></p><p>我在上一个问题的阐述中还重点表达了一点，不知你注意到了没有，那就是：<code>switch</code>语句在<code>case</code>子句的选择上是具有唯一性的。</p><p>正因为如此，<code>switch</code>语句不允许<code>case</code>表达式中的子表达式结果值存在相等的情况，不论这些结果值相等的子表达式，是否存在于不同的<code>case</code>表达式中，都会是这样的结果。具体请看这段代码：</p><pre><code>value3 := [...]int8{0, 1, 2, 3, 4, 5, 6}
switch value3[4] {
case 0, 1, 2:
	fmt.Println(&quot;0 or 1 or 2&quot;)
case 2, 3, 4:
	fmt.Println(&quot;2 or 3 or 4&quot;)
case 4, 5, 6:
	fmt.Println(&quot;4 or 5 or 6&quot;)
}
</code></pre><p>变量<code>value3</code>的值同<code>value1</code>，依然是由从<code>0</code>到<code>6</code>的7个整数组成的数组，元素类型是<code>int8</code>。<code>switch</code>表达式是<code>value3[4]</code>，三个<code>case</code>表达式分别是<code>case 0, 1, 2</code>、<code>case 2, 3, 4</code>和<code>case 4, 5, 6</code>。</p><p>由于在这三个<code>case</code>表达式中存在结果值相等的子表达式，所以这个<code>switch</code>语句无法通过编译。不过，好在这个约束本身还有个约束，那就是只针对结果值为常量的子表达式。</p><p>比如，子表达式<code>1+1</code>和<code>2</code>不能同时出现，<code>1+3</code>和<code>4</code>也不能同时出现。有了这个约束的约束，我们就可以想办法绕过这个对子表达式的限制了。再看一段代码：</p><pre><code>value5 := [...]int8{0, 1, 2, 3, 4, 5, 6}
switch value5[4] {
case value5[0], value5[1], value5[2]:
	fmt.Println(&quot;0 or 1 or 2&quot;)
case value5[2], value5[3], value5[4]:
	fmt.Println(&quot;2 or 3 or 4&quot;)
case value5[4], value5[5], value5[6]:
	fmt.Println(&quot;4 or 5 or 6&quot;)
}
</code></pre><p>变量名换成了<code>value5</code>，但这不是重点。重点是，我把<code>case</code>表达式中的常量都换成了诸如<code>value5[0]</code>这样的索引表达式。</p><p>虽然第一个<code>case</code>表达式和第二个<code>case</code>表达式都包含了<code>value5[2]</code>，并且第二个<code>case</code>表达式和第三个<code>case</code>表达式都包含了<code>value5[4]</code>，但这已经不是问题了。这条<code>switch</code>语句可以成功通过编译。</p><p>不过，这种绕过方式对用于类型判断的<code>switch</code>语句（以下简称为类型<code>switch</code>语句）就无效了。因为类型<code>switch</code>语句中的<code>case</code>表达式的子表达式，都必须直接由类型字面量表示，而无法通过间接的方式表示。代码如下：</p><pre><code>value6 := interface{}(byte(127))
switch t := value6.(type) {
case uint8, uint16:
	fmt.Println(&quot;uint8 or uint16&quot;)
case byte:
	fmt.Printf(&quot;byte&quot;)
default:
	fmt.Printf(&quot;unsupported type: %T&quot;, t)
}
</code></pre><p>变量<code>value6</code>的值是空接口类型的。该值包装了一个<code>byte</code>类型的值<code>127</code>。我在后面使用类型<code>switch</code>语句来判断<code>value6</code>的实际类型，并打印相应的内容。</p><p>这里有两个普通的<code>case</code>子句，还有一个<code>default case</code>子句。前者的<code>case</code>表达式分别是<code>case uint8, uint16</code>和<code>case byte</code>。你还记得吗？<code>byte</code>类型是<code>uint8</code>类型的别名类型。</p><p>因此，它们两个本质上是同一个类型，只是类型名称不同罢了。在这种情况下，这个类型<code>switch</code>语句是无法通过编译的，因为子表达式<code>byte</code>和<code>uint8</code>重复了。好了，以上说的就是<code>case</code>表达式的约束以及绕过方式，你学会了吗。</p><p><strong>总结</strong></p><p>我们今天主要讨论了<code>for</code>语句和<code>switch</code>语句，不过我并没有说明那些语法规则，因为它们太简单了。我们需要多加注意的往往是那些隐藏在Go语言规范和最佳实践里的细节。</p><p>这些细节其实就是我们很多技术初学者所谓的“坑”。比如，我在讲<code>for</code>语句的时候交代了携带<code>range</code>子句时只有一个迭代变量意味着什么。你必须知道在迭代数组或切片时只有一个迭代变量的话是无法迭代出其中的元素值的，否则你的程序可能就不会像你预期的那样运行了。</p><p>还有，<code>range</code>表达式的结果值是会被复制的，实际迭代时并不会使用原值。至于会影响到什么，那就要看这个结果值的类型是值类型还是引用类型了。</p><p>说到<code>switch</code>语句，你要明白其中的<code>case</code>表达式的所有子表达式的结果值都是要与<code>switch</code>表达式的结果值判等的，因此它们的类型必须相同或者能够都统一到<code>switch</code>表达式的结果类型。如果无法做到，那么这条<code>switch</code>语句就不能通过编译。</p><p>最后，同一条<code>switch</code>语句中的所有<code>case</code>表达式的子表达式的结果值不能重复，不过好在这只是对于由字面量直接表示的子表达式而言的。</p><p>请记住，普通<code>case</code>子句的编写顺序很重要，最上边的<code>case</code>子句中的子表达式总是会被最先求值，在判等的时候顺序也是这样。因此，如果某些子表达式的结果值有重复并且它们与<code>switch</code>表达式的结果值相等，那么位置靠上的<code>case</code>子句总会被选中。</p><p><strong>思考题</strong></p><ol>
<li>在类型<code>switch</code>语句中，我们怎样对被判断类型的那个值做相应的类型转换？</li>
<li>在<code>if</code>语句中，初始化子句声明的变量的作用域是什么？</li>
</ol><p><a href="https://github.com/hyper0x/Golang_Puzzlers">戳此查看Go语言专栏文章配套详细代码。</a></p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/fb/44/8b2600fd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>咖啡色的羊驼</span>
  </div>
  <div class="_2_QraFYR_0">好久没留言了，<br><br>1.断言判断value.(type)<br>2.if的判断的域和后面跟着的花括号里头的域。和函数雷同，参数和花括号里头的域同一个</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 好，继续加油吧。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-21 00:21:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTIu1n1DhUGGKTjelrQaLYOSVK2rsFeia0G8ASTIftib5PTOx4pTqdnfwb0NiaEFGRgS661nINyZx9sUg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Zzz</span>
  </div>
  <div class="_2_QraFYR_0">个人理解： for .. range ..  实际上可以认为是方法调用的语法糖，range后面的变量就是方法参数，对于数组类型的变量，传入的参数是数组的副本，更新的是原数组的元素，取的是副本数组的元素；对于切片类型的变量，传入的参数是切片的副本，但是它指向的底层数组与原切片相同，所以取的元素和更新的元素都是同一个数组的元素。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-15 16:29:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c6/f9/caf27bd3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>大王叫我来巡山</span>
  </div>
  <div class="_2_QraFYR_0">了解这些只能证明您对这个语言足够的了解，但是实际中谁会写这么蛋疼的代码呢，这一篇通篇其实说明的还是go语言中关于类型转换的内容</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这些语法细节你要是不注意的话，说不定什么时候就会“踩坑”。文章中的代码是为了演示原理而设计的，因此不一定适用于生产。<br><br>你可以去看看哪些热门开源项目的代码，里面仍然会有体现这些知识点的代码。所以不充分理解这些，可能看复杂些的项目源码都费劲。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-09 17:07:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/84/6c/effc3a5a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>澎湃哥</span>
  </div>
  <div class="_2_QraFYR_0">好像还没有人回答数组变切片的问题，贴一下运行结果吧：<br>i:0, e:1<br>i:1, e:3<br>i:2, e:6<br>i:3, e:10<br>i:4, e:15<br>i:5, e:21<br>[22 3 6 10 15 21]<br><br>每次循环打印了一个索引和值，看起来 range 切片的话，是会每次取 slice[i] 的值，但是应该还是发生了拷贝，不能通过 e 直接修改原值。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-11-29 12:57:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJAWUhO0xSjD6wbGScY5WOujAE94vNYWlWmsVdibb0IWbXzSSNXJHp0lqfWVq8ZicKBsEY1EuAWArew/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Felix</span>
  </div>
  <div class="_2_QraFYR_0">关于数组变切片那个地方。我理解如下：切片自己不拥有任何数据，它只是底层数组的一种表示，对切片的任何操作都会被反映到底层数组中去。 <br>package main<br><br>import &quot;fmt&quot;<br><br>func main() {<br><br>	numbers3 := []int{1, 2, 3, 4, 5, 6}<br>	maxIndex3 := len(numbers2) - 1 &#47;&#47;6-1= 5<br>	for i, e := range numbers3 {  &#47;&#47; 0:1 1:2 2:3 3:4 4:5 5:6<br>		if i == maxIndex3 {   &#47;&#47; 5<br>			numbers3[0] += e  &#47;&#47; 0,7<br>		} else {<br>			numbers3[i+1] += e  &#47;&#47; 1:3<br>		}<br>		&#47;&#47; 0:1 1:(1+2)3 2:3 3:4 4:5 5:6<br>		&#47;&#47; 0:1 1:3 2:(3+3)6 3:4 4:5 5:6<br>		&#47;&#47; 0:1 1:3 2:6 3:(6+4)10 4:5 5:6<br>		&#47;&#47; 0:1 1:3 2:6 3:10 4:(10+5)15 5:6<br>		&#47;&#47; 0:1 1:3 2:6 3:10 4:15 5:(15+6)21<br>		&#47;&#47; 0:(21+1)22 1:3 2:6 3:10 4:15 5:21<br>		&#47;&#47; 22 3 6 10 15 21<br>	}<br>	fmt.Println(numbers3)<br>}</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对，所以我们才称切片为引用类型。它本身只是底层数组及其存储状态的一种描述。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-09 16:16:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/9b/a8/6a391c66.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Leon📷</span>
  </div>
  <div class="_2_QraFYR_0">val.(type)需要提前将类型转换成interface{},一楼的留言有点问题</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-22 16:41:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/a6/87/e802a33e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Dr.Li</span>
  </div>
  <div class="_2_QraFYR_0">感觉go的语法有点变态啊</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 要包容：）工具而已。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-21 00:19:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/GBU53SA3W8GNRAwZIicc3gTEc0nSvfPJw7iboAMicjicmP6egDcibib28DkUfTYOjMd31DIznmofdRZrpIXvmXvjV1PQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>博博</span>
  </div>
  <div class="_2_QraFYR_0">老师遇到一个问题，希望能帮忙解答下！<br>您在文章中说range表达式的结果值是会被复制的，那么是所有的都会被复制么？ 我看了资料，发现字典和通道类型好像没有发生复制！<br><br>&#47;&#47; Lower a for range over a map.<br>&#47;&#47; The loop we generate:<br>&#47;&#47;   var hiter map_iteration_struct<br>&#47;&#47;   for mapiterinit(type, range, &amp;hiter); hiter.key != nil; mapiternext(&amp;hiter) {<br>&#47;&#47;           index_temp = *hiter.key<br>&#47;&#47;           value_temp = *hiter.val<br>&#47;&#47;           index = index_temp<br>&#47;&#47;           value = value_temp<br>&#47;&#47;           original body<br>&#47;&#47;   }<br>很是疑惑，希望能得到指点！谢谢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你看的是 Go 的源码吗？我没找到这段代码。不过我看了下 mapiterinit 函数的代码。其中还是有复制的。只不过，对于这些引用类型的值来说，即使有复制也只会复制一些指针而已，底层数据结构是不会被赋值的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-23 11:49:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/a4/76/585dc6b3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>hiyanxu</span>
  </div>
  <div class="_2_QraFYR_0">老师，我想问一下，range的副本，是说k、v是副本，还是被迭代的数组是副本？<br>我自己测试在for的里面和外面数组地址是一样的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 迭代变量是副本。另外在Go程序里的变量地址是不能完全说明问题的，因为goroutine的栈空间有可能会被优化。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-12-23 20:07:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/21/b8/aca814dd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>江山如画</span>
  </div>
  <div class="_2_QraFYR_0">第一个问题，在类型switch语句中，如何对被判断类型的那个值做类型转换，尝试在 switch 语句中重新定义了一个 uint8 类型的变量和被判断类型的值做加法操作，一共尝试了三种方法，发现需要使用 type assertion 才可以，强转或者直接相加都会出错。<br><br>转换语句是：val.(uint8)<br><br>完整验证代码：<br><br>val := interface{}(byte(1))<br>switch t := val.(type) {<br>case uint8:<br>	var c uint8 = 2<br><br>	&#47;&#47;use type assertion<br>	fmt.Println(c + val.(uint8))<br><br>	&#47;&#47;invalid operation: c + val (mismatched types uint8 and interface {})<br>	&#47;&#47;fmt.Println(c + val)<br><br>	&#47;&#47;cannot convert val (type interface {}) to type uint8: need type assertion<br>	&#47;&#47;fmt.Println(c + uint8(val))<br><br>default:<br>	fmt.Printf(&quot;unsupported type: %T&quot;, t)<br>}<br><br>第二个问题，在if语句中，初始化子句声明的变量的作用域是在该if语句之内，if语句之外使用该变量会提示 “undefined”。<br><br>验证代码：<br><br>m := make(map[int]bool)<br>if _, ok := m[1]; ok {<br>	fmt.Printf(&quot;exist: %v\n&quot;, ok)<br>} else {<br>	fmt.Printf(&quot;not exist: %v\n&quot;, ok)<br>}<br><br>&#47;&#47;fmt.Println(ok)  &#47;&#47;报错，提示 undefined: ok</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-08 12:11:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/87/64/3882d90d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>yandongxiao</span>
  </div>
  <div class="_2_QraFYR_0">咖啡色的羊驼的答案貌似是错误的吧？<br>1. 在类型switch语句中，t := value6.(type)；匹配到哪个case表达式，t就会是哪种具体的数据类型；<br>2. 在if语句中，子句声明的变量的作用域  比 随后的花括号内的变量的的作用域  更大<br>	if a := 10; a &gt; 0 {<br>		a := 20<br>		println(a)<br>	}<br>反证法：如果咖啡色的羊驼说的对，上面的语句不应该编译通过才对。<br><br>关于2这个细微的差异，也适用于for i:=0; i&lt;10; i++ 语句。<br>在for语句中的使用匿名函数，很可能出现“loop variable capture”问题。根本原因也是i的作用域与{}中的变量的作用域是不同的。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-27 19:36:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/24/3e/692a93f7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>茶底</span>
  </div>
  <div class="_2_QraFYR_0">老师什么时候讲逃逸分析啊</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-23 08:12:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/34/67/06a7f9be.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>while (1)等;</span>
  </div>
  <div class="_2_QraFYR_0">文中说“被迭代的对象是range表达式结果值的副本而不是原值。”，那被迭代的对象是切片时，可以理解为指针的副本吗？也就是指针和指针的副本指向同一地址？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不是指针的副本，是切片结构体的副本，切片结构体中有指向底层数组的指针。这个副本与原本会指向同一个底层数组。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-09 18:03:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/22/c0/177d6750.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Rico</span>
  </div>
  <div class="_2_QraFYR_0">在类型switch语句中，我们怎样对被判断类型的那个值做相应的类型转换？<br>------val.(type)<br><br>在if语句中，初始化子句声明的变量的作用域是什么？<br>-------变量作用域为if语句{}内部的范围<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 第二个回答不太严谨，应该是：if语句块内，因为 else 的 {} 也是包含其中的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-08 12:28:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/ed/04/a63ed1e3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>hua</span>
  </div>
  <div class="_2_QraFYR_0">把总结结论放在最前面再看主体内容会容易理解得多。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-21 19:03:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/2d/18/918eaecf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>后端进阶</span>
  </div>
  <div class="_2_QraFYR_0">真的很喜欢go的语法与简洁的哲学</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-21 13:04:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/71/e5/bcdc382a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>My dream</span>
  </div>
  <div class="_2_QraFYR_0">Go1.11已经正式发布，最大的一个亮点是增加了对WebAssembly的实验性支持。老师要讲一下不？我们都不懂这个有什么意义</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 主要是写Web端的时候有些用，不过我不觉得用处很大，因为现在大型网站都是前后端分离的。最后我视情况而定吧。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-21 10:12:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/24/a2/d61e4e28.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jack</span>
  </div>
  <div class="_2_QraFYR_0">        index := 0<br>	var mu sync.Mutex<br>	fp := func(i int, fn func()) {<br>		for {<br>			mu.Lock()<br>			if index == i {<br>				fn()<br>				index++<br>				mu.Unlock()<br>				break<br>			}<br>			mu.Unlock()<br>			time.Sleep(time.Nanosecond)<br>		}<br>	}<br>	for i := 0; i &lt; 10; i++ {<br>		go func(i int) {<br>			fn := func() {<br>				fmt.Println(i)<br>			}<br>			fp(i, fn)<br>		}(i)<br>	}<br>	&#47;&#47; 这里就简单点了，不用 sync.WaitGroup 等那些结束了<br>	time.Sleep(time.Second)</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 功能上可以，性能上还有待进一步推敲。比如：不用锁。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-20 22:02:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/94/12/15558f28.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Jason</span>
  </div>
  <div class="_2_QraFYR_0">老师，我在Go编译器源码的statements.cc里找到了range的原型，其实是对数组本身做了整体拷贝，后面的循环都是基于这个副本。我想问问老师，为什么range表达式要去做拷贝呢，明明代价更高了。还是说go设计range的初衷就是读取，真要修改就使用传统的for循环，期待老师的回答<br>&#47;&#47; Arrange to do a loop appropriate for the type.  We will produce<br>&#47;&#47;   for INIT ; COND ; POST {<br>&#47;&#47;           ITER_INIT<br>&#47;&#47;           INDEX = INDEX_TEMP<br>&#47;&#47;           VALUE = VALUE_TEMP &#47;&#47; If there is a value<br>&#47;&#47;           original statements<br>&#47;&#47;   }<br>针对数组<br>&#47;&#47; The loop we generate:<br>&#47;&#47;   len_temp := len(range)<br>&#47;&#47;   range_temp := range<br>&#47;&#47;   for index_temp = 0; index_temp &lt; len_temp; index_temp++ {<br>&#47;&#47;           value_temp = range_temp[index_temp]<br>&#47;&#47;           index = index_temp<br>&#47;&#47;           value = value_temp<br>&#47;&#47;           original body<br>&#47;&#47;   }</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你的最后一句话其实已经带出了答案。这也是为了安全考虑。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-24 12:02:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/27/ff/e4/927547a9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>无名无姓</span>
  </div>
  <div class="_2_QraFYR_0">package main<br>import(<br>	&quot;fmt&quot;<br>)<br>func main() {<br>	arr:=[]int{1,2,3,4,5,6}<br>	arrlen:=len(arr)-1<br>	for i,e:= range arr {<br>		if i==arrlen{<br>			arr[0]+=e<br>		}else{<br>			arr[i+1]+=e<br>		}<br>	}<br>	fmt.Println(arr)<br>}<br>========================</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-13 16:51:43</div>
  </div>
</div>
</div>
</li>
</ul>