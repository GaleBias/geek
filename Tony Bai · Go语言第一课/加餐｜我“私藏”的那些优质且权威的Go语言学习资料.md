<audio title="加餐｜我“私藏”的那些优质且权威的Go语言学习资料" src="https://static001.geekbang.org/resource/audio/a8/fc/a8f12fca169322b14aae215c36ba55fc.mp3" controls="controls"></audio> 
<p>你好，我是Tony Bai。</p><p>学习编程语言并没有捷径，就像我们在开篇词中提到的那样，<strong>脑勤+手勤才是正确的学习之路</strong>。不过，留言区也一直有同学问我，除了这门课之外，还有什么推荐的Go语言学习资料。今天我们就来聊聊这个话题。</p><p>如今随着互联网的高速发展，现在很多同学学习编程语言，已经从技术书籍转向了各种屏幕，<strong>以专栏或视频实战课为主，技术书籍等参考资料为辅</strong>的学习方式已经成为主流。当然，和传统的、以编程类书籍为主的学习方式相比，谈不上哪种方式更好，只不过它更适合如今快节奏的生活工作状态，更适合碎片化学习占主流的学习形态罢了。</p><p>但在编程语言的学习过程中，技术书籍等参考资料依旧是不可或缺的，<strong>优秀的参考资料是编程语言学习过程的催化剂</strong>，拥有正确的、权威的参考资料可以让你减少反复查找资料所浪费的时间与精力，少走弯路。</p><p>这节课我会给你分享下我“私藏”的Go语言学习的参考资料，包括一些经典的技术书籍和其他电子形式的参考资料。</p><p>虽然现在编程语言学习可参考的资料形式、种类已经非常丰富了，但技术类书籍（包括电子版）在依旧占据着非常重要的地位。所以，我们就先重点看看在Go语言学习领域，有哪些优秀的书籍值得我们认真阅读。</p><h2>Go技术书籍</h2><!-- [[[read_end]]] --><p>和C（1972年）、C++（1983）、Java（1995）、Python（1991）等编程语言在市面上的书籍数量相比，Go流行于市面（尤其是中国大陆地区）上的图书要少很多。究其原因，可能有以下几个：</p><p><strong>首先，我觉得主要原因还是Go语言太年轻了。</strong>尽管<a href="https://go.dev/blog/12years">Go刚刚过完它12岁的生日</a>，但和上面这些语言中“最年轻”的Java语言之间也还有14年的“年龄差”。</p><p><strong>其次，Go以品类代名词的身份占据的“领域”还很少</strong>。提到Web，人们想到的是Java Spring；提到深度学习、机器学习、人工智能，人们想到的是Python，提到游戏，人们想到的是C++；提到前端，人们想到的是JavaScript。这些语言在这些垂直领域早早以杀手级框架入场，使得它们成为了这一领域的“品类代名词”。</p><p>但Go语言诞生晚，入场也较晚。Go虽然通过努力覆盖了一些领域并占据优势地位，比如云原生、API、微服务、区块链，等等，但还不能说已经成为了这些领域的“品类代名词”，因此被垂直领域书籍关联的机会也不像上面那几门语言那么多。</p><p><strong>最后是翻译的时间问题。</strong>相对于国内，国外关于Go语言的作品要多不少，但引进国外图书资料需要时机以及时间（毕竟要找译者翻译）。</p><p>Go在国内真正开始快速流行起来，大致是在2015年第一届GopherChina大会（2015年4月）之后，当时的Go版本是1.4。同一年下半年发布的Go 1.5版本实现了Go的自举，并让GC延迟大幅下降，让Go在国内彻底流行开来。一批又一批程序员成为Gopher，在大厂、初创公司实践着Go语言。但知识和技能的沉淀和总结需要时间。</p><p>不过，2020年开始，国内作者出版的Go语言相关书籍已经逐渐多了起来。2022年加入泛型的Go 1.18版本发布后，相信会有更多Gopher加入Go技术书籍的写作行列，在未来3年，国内Go语言技术书籍也会迎来一波高峰。</p><p>我个人接触Go语言比较早，几乎把Go语言相关的中外文书籍都通读过一遍，其中几本经典好书甚至还读过不止一遍。所以这里我也会给你推荐几本我认为系统学习Go语言必读的经典好书。说实在的，Go语言比较简单，如果单单从系统掌握这门语言的角度来看，阅读下面这几本书籍就足够了。</p><p>这几本书我按作者名气、关注度、内容实用性、经典指数这几个维度，分别打了分（每部分满分为5分，总分的满分为20分），按推荐性从低到高排了序，你可以参考下。</p><h3>第五名：《The Way To Go》- Go语言百科全书</h3><p><img src="https://static001.geekbang.org/resource/image/1a/bb/1a40d941b284a2b1357a52723046c3bb.png?wh=405x500" alt="图片"></p><p><a href="https://book.douban.com/subject/10558892/">《The Way To Go》</a>是我<strong>早期学习Go语言时最喜欢翻看的一本书</strong>。这本书成书于2012年3月，恰逢Go 1.0版本刚刚发布，当时作者承诺书中代码都可以在Go 1.0版本上编译通过并运行。这本书分为4个部分：</p><ul>
<li>为什么学习Go以及Go环境安装入门；</li>
<li>Go语言核心语法；</li>
<li>Go高级用法（I/O读写、错误处理、单元测试、并发编程、socket与web编程等)；</li>
<li>Go应用（常见陷阱、语言应用模式、从性能考量的代码编写建议、现实中的Go应用等）。</li>
</ul><p>每部分的每个章节都很精彩，而且这本书也是我目前见到的、最全面详实的、讲解Go语言的书籍了，可以说是Gopher们的第一本<strong>“Go百科全书”</strong>。</p><p>不过遗憾的是，这本书没有中文版。这可能是由于这本书出版太早了，等国内出版社意识到要引进Go语言方面的书籍的时候，这本书使用的Go版本已经太老了。不过，这本书中绝大部分例子依然可以在今天最新的Go编译器下通过编译并运行起来。好在Gopher<a href="https://github.com/Unknwon/the-way-to-go_ZH_CN">无闻</a>在GitHub上发起了这本书的<a href="https://github.com/Unknwon/the-way-to-go_ZH_CN">中译版项目</a>，如果你感兴趣的话，可以去GitHub上看或下载阅读。</p><p>这本书虽然很棒，但毕竟年头“久远”，所以我也只能委屈它一下了，将它列在推荐榜的第五位，这里我也给出了对它的各个指数的评分：</p><p><img src="https://static001.geekbang.org/resource/image/ff/0e/ff058ae7f50dc34388b80e461f2cdd0e.png?wh=1436x346" alt="图片"></p><h3>第四名：《Go 101》- Go语言参考手册</h3><p><img src="https://static001.geekbang.org/resource/image/c4/74/c4770cdb320e5f728ea4aa93d491ec74.png?wh=1400x1980" alt="图片"></p><p><a href="https://go101.org/article/101.html">《Go 101》</a><strong>是一本在国外人气和关注度比在国内高的中国人编写的英文书</strong>，当然它也是有中文版的。</p><p>如果只从书名中的<strong>101</strong>去判断，你很大可能会认为这仅仅是一本讲解Go入门基础的书，但这本书的内容可远远不止入门这么简单。这本书大致可以分为三个部分：</p><ul>
<li>Go语法基础；</li>
<li>Go类型系统与运行时实现；</li>
<li>以专题（topic）形式阐述的Go特性、技巧与实践模式。</li>
</ul><p>除了第一部分算101范畴，其余两个部分都是Go语言的高级话题，也是我们要精通Go语言必须要掌握的“知识点”。并且，作者结合Go语言规范，对每个知识点的阐述都细致入微，也结合大量示例进行辅助说明。我们知道，C和C++语言在市面上都有一些由语言作者或标准规范委员会成员编写的Annotated或Rationale书籍（语言参考手册或标准解读），而《Go 101》这本书，就可以理解为<strong>Go语言的标准解读或参考手册</strong>。</p><p>Go 101这本书是<a href="https://github.com/go101/go101">开源电子书</a>，它的作者也在国外一些支持自出版的服务商那里做了付费数字出版。这就让这本书相对于其他纸板书有着另外一个优势：<strong>与时俱进</strong>。在作者的不断努力下，这本书的知识点更新基本保持与Go的演化同步，目前书的内容已经覆盖了最新的Go 1.17版本。</p><p>这本书的作者是国内资深工程师<a href="https://gfw.tapirgames.com/">老貘</a>，他花费三年时间“呕心沥血”完成这本书并且免费奉献给Go社区，值得我们为他点一个大大的赞！近期老貘的两本<a href="https://mp.weixin.qq.com/s/-mVqgbzU3yQNmaGegU4IHg">新书《Go编程优化101》和《Go细节大全101》也将问世</a>，想必也是不可多得的优秀作品。</p><p>下面是我对这本书各个指数的评分：</p><p><img src="https://static001.geekbang.org/resource/image/19/54/198e63189cb76c37ce5d5d1yy81ac154.png?wh=1394x348" alt="图片"></p><h3>第三名：《Go语言学习笔记》- Go源码剖析与实现原理探索</h3><p><img src="https://static001.geekbang.org/resource/image/a0/d0/a08fb31ae47723a28fc51dd067053fd0.png?wh=768x1024" alt="图片"></p><p><a href="https://book.douban.com/subject/26832468/">《Go语言学习笔记》</a>是一本在国内影响力和关注度都很高的作品。一来，它的作者<a href="https://github.com/qyuhen/">雨痕老师</a>是国内资深工程师，也是2015年第一届GopherChina大会讲师；二来，这部作品的前期版本是以开源电子书的形式分享给国内Go社区的；三来，作者在Go源码剖析方面可谓之条理清晰，细致入微。</p><p>2016年《Go语言学习笔记》的纸质版出版，覆盖了当时最新的Go 1.5版本。Go 1.5版本在Go语言演化历史中的分量极高，它不仅实现了Go自举，还让Go GC的延迟下降到绝大多数应用可以将它应用到生产的程度。这本书整体上分为两大部分：</p><ul>
<li>Go语言详解：以短平快、“堆干货”的风格对Go语言语法做了说明，能用示例说明的，绝不用文字做过多修饰；</li>
<li>Go源码剖析：这是这本书的精华，也是最受Gopher们关注的部分。这部分对Go运行时神秘的内存分配、垃圾回收、并发调度、channel和defer的实现原理、sync.Pool的实现原理都做了细致的源码剖析与原理总结。</li>
</ul><p>随着Go语言的演化，它的语言和运行时实现一直在不断变化，但Go 1.5版本的实现是后续版本的基础，所以这本书对它的剖析非常值得每位Gopher阅读。从雨痕老师的<a href="https://github.com/qyuhen/book">GitHub</a>上的最新消息来看，他似乎在编写新版Go语言学习笔记。剖析源码的过程是枯燥繁琐的，期待雨痕老师新版Go学习笔记能早日与Gopher们见面。</p><p>下面是我对这本书各个指数的评分：</p><p><img src="https://static001.geekbang.org/resource/image/60/08/60caa83e4217d0700340edbffa3bca08.png?wh=1440x370" alt="图片"></p><h3>第二名：《Go语言实战》- 实战系列经典之作，紧扣Go语言的精华</h3><p><img src="https://static001.geekbang.org/resource/image/d1/83/d1d2dfa85da99579ae86589947ee1183.png?wh=500x628" alt="图片"></p><p>Manning出版社出版的“实战系列（xx in action）”一直是程序员心中高质量和经典的代名词。在出版Go语言实战系列书籍方面，这家出版社也是丝毫不敢怠慢，邀请了Go社区知名的三名明星级作者联合撰写。这三位作者分别是：</p><ul>
<li>威廉·肯尼迪 (William Kennedy) ，知名Go培训师，培训机构Ardan Labs的联合创始人，“Ultimate Go”培训的策划实施者；</li>
<li>布赖恩·克特森 (Brian Ketelsen) ，世界上最知名的Go技术大会GopherCon大会的联合发起人和组织者，<a href="https://gopheracademy.com/">GopherAcademy</a>创立者，现微软Azure工程师；</li>
<li>埃里克·圣马丁 (Erik St.Martin) ，世界上最知名的Go技术大会GopherCon大会的联合发起人和组织者。</li>
</ul><p><a href="https://book.douban.com/subject/27015617/">《Go语言实战》</a>这本书并不是大部头，而是薄薄的一本（中文版才200多页），所以你不要期望从本书得到百科全书一样的阅读感。而且，这本书的作者们显然也没有想把它写成面面俱到的作品，而是<strong>直击要点</strong>，也就是挑出Go语言和其他语言相比与众不同的特点进行着重讲解。这些特点构成了这本书的结构框架：</p><ul>
<li>入门：快速上手搭建、编写、运行一个Go程序；</li>
<li>语法：数组（作为一个类型而存在）、切片和map；</li>
<li>Go类型系统的与众不同：方法、接口、嵌入类型；</li>
<li>Go的拿手好戏：并发及并发模式；</li>
<li>标准库常用包：log、marshal/unmarshal、io（Reader和Writer）；</li>
<li>原生支持的测试。</li>
</ul><p>读完这本书，你就掌握了Go语言的精髓之处，这也迎合了多数Gopher的内心需求。而且，这本书中文版译者李兆海也是Go圈子里的资深Gopher，翻译质量上乘。</p><p>下面是我对这本书各个指数的评分：</p><p><img src="https://static001.geekbang.org/resource/image/1a/d3/1ab6463f933407d349b39dee8a491dd3.png?wh=1388x358" alt="图片"></p><h3>第一名：《Go程序设计语言》- 人手一本的Go语言“圣经”</h3><p>如果说由<a href="https://www.cs.princeton.edu/people/profile/bwk">Brian W. Kernighan</a>和<a href="http://en.wikipedia.org/wiki/Dennis_Ritchie">Dennis M. Ritchie</a>联合编写的<a href="https://book.douban.com/subject/1882483/">《The C Programming Language》</a>（也称K&R C）是C程序员（甚至是所有程序员）心目中的“圣经”的话，那么同样由Brian W. Kernighan(K)参与编写的<a href="https://book.douban.com/subject/26337545/">《The Go Programming Language》</a>（也称<a href="http://www.gopl.io/">tgpl</a>）就是Go程序员心目中的“圣经”。</p><p><img src="https://static001.geekbang.org/resource/image/78/3b/785c0543162558565ec4585fe0b5493b.png?wh=1260x586" alt="图片"></p><p>这本书模仿并致敬“The C Programming Language”的经典结构，从一个"hello, world"示例开始带领大家开启Go语言之旅。</p><p>第二章程序结构是Go语言这个“游乐园”的向导图。了解它之后，我们就会迫不及待地奔向各个“景点”细致参观。Go语言规范中的所有“景点”在这本书中都覆盖到了，并且由浅入深、循序渐进：从基础数据类型到复合数据类型，从函数、方法到接口，从创新的并发Goroutine到传统的基于共享变量的并发，从包、工具链到测试，从反射到低级编程（unsafe包）。</p><p>作者行文十分精炼，字字珠玑，这与《The C Programming Language》的风格保持了高度一致。而且，书中的示例在浅显易懂的同时，又极具实用性，还突出Go语言的特点（比如并发web爬虫、并发非阻塞缓存等）。</p><p>读完这本书后，你会有一种爱不释手，马上还要从头再读一遍的感觉，也许这就是“圣经”的魅力吧！</p><p>这本书出版于2015年10月26日，也是既当年中旬Go 1.5这个里程碑版本发布后，Go社区的又一重大历史事件！并且Brian W. Kernighan老爷子的影响力让更多程序员加入到Go阵营，这也或多或少促成了Go成为下一个年度，也就是2016年年度TIOBE最佳编程语言。能得到Brian W. Kernighan老爷子青睐的编程语言只有C和Go，这也是Go的幸运。</p><p>这本书的另一名作者Alan A. A. Donovan也并非等闲之辈，他是Go核心开发团队的成员，专注于Go工具链方面的开发。</p><p>现在唯一遗憾的就是Brian W. Kernighan老爷子年事已高，不知道Go 1.18版本加入泛型语法后，老爷子是否还有精力再更新这本圣经。</p><p>这本书的<a href="https://book.douban.com/subject/27044219/">中文版</a>由七牛云团队翻译，总体质量也是不错的。建议Gopher们人手购置一本圣经“供奉”起来！</p><p>这里，我对这本书的各个指数都给了满分：</p><p><img src="https://static001.geekbang.org/resource/image/95/9c/959b231e001f5542f7d78a773a979e9c.png?wh=1336x352" alt="图片"></p><h2>其他形式的参考资料</h2><p>除了技术书籍之外，Go语言学习资料的形式也呈现出多样化。下面是我个人经常阅读和使用的其他形式的Go参考资料，这里列出来供同学们参考。</p><h3>Go官方文档</h3><p>如果你要问什么Go语言资料是最权威的，那莫过于<a href="https://go.dev/doc/">Go官方文档</a>了。</p><p>Go语言从诞生那天起，就十分重视项目文档的建设。除了可以在<a href="https://go.dev">Go官方网站</a>上查看到最新稳定发布版的文档之外，我们还可以在<a href="https://tip.golang.org">https://tip.golang.org</a>上查看到项目主线分支（master）上最新开发版本的文档。</p><p>同时Go还将整个Go项目文档都加入到了Go发行版中，这样开发人员在本地安装Go的同时也拥有了一份完整的Go项目文档。这两年Go核心团队还招聘专人负责Go官方站点的研发，就在不久前，Go团队已经将原Go官方站点golang.org重定向到最新开发的go.dev网站上，新网站首页是这样的：</p><p><img src="https://static001.geekbang.org/resource/image/0a/4a/0aa436d05d74fd36455f323826d77d4a.png?wh=1920x1204" alt="图片"></p><p>Go官方文档中的<a href="https://go.dev/ref/spec">Go语言规范</a>、<a href="https://go.dev/ref/mod">Go module参考文档</a>、<a href="https://go.dev/doc/cmd">Go命令参考手册</a>、<a href="https://go.dev/doc/effective_go.html">Effective Go</a>、<a href="https://pkg.go.dev/std">Go标准库包参考手册</a>以及<a href="https://go.dev/doc/faq">Go常见问答</a>等都是<strong>每个Gopher必看的内容</strong>。我强烈建议你一定要抽出时间来仔细阅读这些文档。</p><h3>Go相关博客</h3><p>在编程语言学习过程中，诞生于Web 2.0时代的博客依旧是开发人员的一个重要参考资料来源。这里我也列出了我个人关注且经常阅读的一些博客，你可以参考一下：</p><ul>
<li><a href="https://go.dev/blog">Go语言官博</a>，Go核心团队关于Go语言的权威发布渠道；</li>
<li><a href="https://commandcenter.blogspot.com/">Go语言之父Rob Pike的个人博客</a>；</li>
<li><a href="https://research.swtch.com">Go核心团队技术负责人Russ Cox的个人博客</a>；</li>
<li><a href="https://commaok.xyz/">Go核心开发者Josh Bleecher Snyder的个人博客</a>；</li>
<li><a href="https://rakyll.org">Go核心团队前成员Jaana Dogan的个人博客</a>；</li>
<li><a href="https://dave.cheney.net">Go鼓吹者Dave Cheney的个人博客</a>；</li>
<li><a href="https://www.ardanlabs.com/blog">Go语言培训机构Ardan Labs的博客</a>；</li>
<li><a href="https://gocn.vip">GoCN社区</a>；</li>
<li><a href="https://golang.design/">Go语言百科全书</a>：由欧长坤维护的Go语言百科全书网站。</li>
</ul><h3>Go播客</h3><p>使用播客这种形式作编程语言类相关内容传播的资料并不多，能持续进行下去的就更少了。目前我唯一关注的就是changelog这个技术类播客平台下的<a href="https://changelog.com/gotime">Go Time频道</a>。这个频道有几个Go社区知名的Gopher主持，目前已经播出了200多期，每期的嘉宾也都是Go社区的重量级人物，其中也不乏像Go语言之父这样的大神参与。</p><h3>Go技术演讲</h3><p>Go技术演讲，也是我们学习Go语言以及基于Go语言的实践的优秀资料来源。关于Go技术演讲，我个人建议以各大洲举办的GopherCon技术大会为主，这些已经基本可以涵盖每年Go语言领域的最新发展。下面我也整理了一些优秀的Go技术演讲资源列表，你可以参考：</p><ul>
<li><a href="https://go.dev/talks/">Go官方的技术演讲归档</a>，这个文档我强烈建议你按时间顺序看一下，通过这些Go核心团队的演讲资料，我们可以清晰地了解Go的演化历程；</li>
<li><a href="https://www.youtube.com/c/GopherAcademy/playlists">GopherCon技术大会</a>，这是Go语言领域规模最大的技术盛会，也是Go官方技术大会；</li>
<li><a href="https://www.youtube.com/c/GopherConEurope/playlists">GopherCon Europe技术大会</a>；</li>
<li><a href="https://www.youtube.com/c/GopherConUK/playlists">GopherConUK技术大会</a>；</li>
<li><a href="https://www.youtube.com/channel/UCMEvzoHTIdZI7IM8LoRbLsQ/playlists">GoLab技术大会</a>；</li>
<li><a href="https://www.youtube.com/user/fosdemtalks/playlists">Go Devroom@FOSDEM</a>；</li>
<li><a href="https://space.bilibili.com/436361287">GopherChina技术大会</a>，这是中国大陆地区规模最大的Go语言技术大会，由GoCN社区主办。</li>
</ul><h3>Go日报/周刊邮件列表</h3><p>通过邮件订阅Go语言类日报或周刊，我们也可以获得关于Go语言与Go社区最新鲜的信息。对于国内的Gopher们来说，订阅下面两个邮件列表就足够了：</p><ul>
<li><a href="https://studygolang.com/go/weekly">Go语言爱好者周刊</a>，由Go语言中文网维护；</li>
<li><a href="https://github.com/bigwhite/gopherdaily">Gopher日报</a>，由我本人维护的Gopher日报项目，创立于2019年9月。</li>
</ul><h3>其他</h3><p>最后，这里还有两个可能经常被大家忽视的Go参考资料渠道，一个是<a href="https://github.com/golang/go/issues">Go语言项目的官方issue列表</a>。通过这个issue列表，我们可以实时看到Go项目演进状态，及时看到Go社区提交的各种bug。同时，我们通过挖掘该列表，还可以了解某个Go特性的来龙去脉，这对深入理解Go特性很有帮助。</p><p>另外一个就是<a href="https://go-review.googlesource.com/q/status:open+-is:wip">Go项目的代码review站点</a>。通过阅读Go核心团队review代码的过程与评审意见，我们可以看到Go核心团队是如何使用Go进行编码的，能够学习到很多Go编码原则以及地道的Go语言惯用法，对更深入地理解Go语言设计哲学，形成Go语言编程思维有很大帮助。</p><h2>写在最后</h2><p>和学习任何一种知识或技能一样，编程语言学习过程中的参考资料<strong>不在于多而在于精</strong>。在这里，我已经将这些年来我积累的精华Go参考资料都罗列出了。如果你还有什么推荐的资料，也欢迎在留言区补充。</p><p>希望你在以专栏为主的Go学习过程中，能充分利用好这些参考资料，让它更好地发挥催化作用，以帮助你更快、更深入地掌握Go语言，形成Go编程思维，写出更为地道的、优秀的Go代码。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/26/27/eba94899.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>罗杰</span>
  </div>
  <div class="_2_QraFYR_0">要学的东西太多太多了，不论从哪里开始，从一点一滴掌握做起。有时候选择太多跟没有选择一样，先把该专栏吃透，再计划读书，一本中文，一本英文，同时也努力去阅读 Go 官方博客。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-15 09:21:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/1d/de/62bfa83f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>aoe</span>
  </div>
  <div class="_2_QraFYR_0">《Go 语言实战》读过一遍，已经全忘了；<br>《Go程序设计语言》读到第9页，出现了map，直接懵了。吸取了之前只看不练，看完就忘的教训，现在边看边练。<br>正在tour.go-zh.org跟着官网学习基本语法的时候，Tony 老师的专栏就开了，在这里学到了很多基础知识，节省了很大时间，尤其是学完入门篇后，再也不用为编译代码瑟瑟发抖了。<br>今天又看到了这么多学习资源，心情很激动！<br>想想在 《Go 项目的代码 review 站点》可以看到世界级大佬点评代码就更激动了！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍边看边练是最有效的方法。手不能懒:)</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-15 00:45:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/7b/bd/ccb37425.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>进化菌</span>
  </div>
  <div class="_2_QraFYR_0">他山之石可以攻玉，好东西当然要多翻阅。对于当下还不熟悉go的自己，多用代码解决问题，再参考资料，应该很棒~</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-16 22:08:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/lfMbV8RibrhFxjILg4550cZiaay64mTh5Zibon64TiaicC8jDMEK7VaXOkllHSpS582Jl1SUHm6Jib2AticVlHibiaBvUOA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>用0和1改变自己</span>
  </div>
  <div class="_2_QraFYR_0">棒，方向对了，就不怕路远</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: “方向对了，就不怕路远”，说的太好了！对go来说，路也并不太遥远。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-15 10:15:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/28/55/5c/643c0e33.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ShiPF</span>
  </div>
  <div class="_2_QraFYR_0">机械工业出版社的那本go程序设计语言翻译简直辣眼睛🙈，还是多学英语啃原版比较香</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 机械工业那本中译版是七牛云团队翻译，估计是考虑上市时间，前后一致性就差一些。建议至少读一遍原版。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-02 18:48:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/05/e4/3e676c4d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ps Sensking</span>
  </div>
  <div class="_2_QraFYR_0">请问您的两本书和这个专栏讲的一样嘛？不一样的话我希望买，如果一样的话，我希望您能出一个实战类的go 和 postgresql 结合的实战 </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 专栏偏基础。那个书是偏进阶。讲解的逻辑是不同的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-13 01:59:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/33/27/e5a74107.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Da Vinci</span>
  </div>
  <div class="_2_QraFYR_0">欧神的go语言原本挺不错的，但是有点难复用，需要慢慢品</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 那本也很不错。不过目前还有很多内容不完整。相对完整的章节值得阅读，但更多用于进阶。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-16 11:50:47</div>
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
  <div class="_2_QraFYR_0">大白老师的这些推荐，很多都是我之前不知道的，可谓是大开眼界了。<br><br>不过，打开电商网站，搜索Go的图书，发现已经有很多国人写的著作了，其中不乏优秀之作。也有极客时间这类在线学习的网站。Go的技术布道在国内已经是遍地开花的状态了。<br><br>另外，文中说：“同时 Go 还将整个 Go 项目文档都加入到了 Go 发行版中，这样开发人员在本地安装 Go 的同时也拥有了一份完整的 Go 项目文档。”  我记得Go从某个版本开始，已经不在本地安装项目文档了吧？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你知道的很多啊:) 没错，文章描述不够准确。godoc程序很早就不与go发行版一并发布了。但go文档数据是在go 1.16版本开始才移出go发行版包的。标准库包的文档由于是随源码一起的，所以一直有。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-15 11:52:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4c/12/f0c145d4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Rayjun</span>
  </div>
  <div class="_2_QraFYR_0">赞，归根结底还是得自己去消化一手信息</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-15 08:53:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/56/4e/9291fac0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Jay</span>
  </div>
  <div class="_2_QraFYR_0">老师的书有电子版吗？想随时可以在手机上翻阅</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 微信读书上有。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-23 13:33:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2d/b7/30/c1e4f5b5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>微微超级丹💫</span>
  </div>
  <div class="_2_QraFYR_0">感觉学无止境啦哈哈</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-15 22:38:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/2e/ca/469f7266.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>菠萝吹雪—Code</span>
  </div>
  <div class="_2_QraFYR_0">师傅领进门，修行在个人</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-14 17:19:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/3f/0d/1e8dbb2c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>怀揣梦想的学渣</span>
  </div>
  <div class="_2_QraFYR_0">梳理出来的资源很丰富，有了作者梳理出来的资源，我自己省事了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-07 21:09:55</div>
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
  <div class="_2_QraFYR_0">请问老师，有没有比较好go相关框架学习的资料。例如go chassis 等之类的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 框架太多了，我学习框架一般都是直接看框架自己的doc&#47;guide之类的一手文档。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-07 17:40:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/70/67/0c1359c2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>qinsi</span>
  </div>
  <div class="_2_QraFYR_0">每看一个专栏主题就去看下Go语言规范里对应的主题</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-19 21:23:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/0a/a4/828a431f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张申傲</span>
  </div>
  <div class="_2_QraFYR_0">感谢老师的无私分享~</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不客气:)</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-16 12:00:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/53/a8/abc96f70.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>return</span>
  </div>
  <div class="_2_QraFYR_0">个人觉得 能把几本书啃完， 平时看官方文档， 就已经很牛了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍 </p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-16 09:25:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/29/bb/e0/c7cd5170.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Bynow</span>
  </div>
  <div class="_2_QraFYR_0">感谢老师分享。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-15 11:42:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/ce/d7/5315f6ce.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>不负青春不负己🤘</span>
  </div>
  <div class="_2_QraFYR_0">mark:)</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-15 00:22:29</div>
  </div>
</div>
</div>
</li>
</ul>