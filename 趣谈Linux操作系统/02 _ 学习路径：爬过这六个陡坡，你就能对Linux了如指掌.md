<audio title="02 _ 学习路径：爬过这六个陡坡，你就能对Linux了如指掌" src="https://static001.geekbang.org/resource/audio/ae/7e/ae339f4a53a07f03795c1c46fa43c67e.mp3" controls="controls"></audio> 
<p>很多人觉得Linux操作系统刚开始学起来很难，主要是因为我们平时办公或者学习用的都是Windows系统，而Windows和Linux的使用模式是完全不一样的。</p><p>Windows的基本使用模式是“<strong>图形化界面+菜单</strong>”。也就是说，无论我们做什么事情，首先要找一个图形化的界面。在这里面，“开始”菜单是统一的入口，无论是运行程序，还是做系统设置，你都能找到一个界面，界面上会有各种各样的输入框和菜单。我们只要挨个儿看过去，总能找到想操作的功能。实在不行，还有杀手锏，就是右键菜单，挨个儿一项一项看下去，最终也能实现想做的操作。</p><p>如果你刚刚上手Linux，就会发现，情况完全不一样。你基本是这也找不着，那也找不着，觉得Linux十分难用，从而“从入门到放弃”。</p><p>Linux上手难，学习曲线陡峭，所以它的学习过程更像一个<strong>爬坡模式</strong>。这些坡看起来都很陡，但是一旦爬上一阶，就会一马平川。你会惊叹Linux的设计之美，而Linux的灵活性也会使得你有N多种方法解决问题，从而事半功倍，你就会有一切尽在掌握的感觉。只可惜，大部分同学都停留在了山脚下。</p><p>那怎样才能掌握这项爬坡技能呢？我们首先需要明确，我们要爬哪些坡。</p><p>我总结了一下，在整个Linux的学习过程中，要爬的坡有六个，分别是：熟练使用Linux命令行、使用Linux进行程序设计、了解Linux内核机制、阅读Linux内核代码、实验定制Linux组件，以及最后落到生产实践上。以下是我为你准备的爬坡秘籍以及辅助的书单弹药。</p><!-- [[[read_end]]] --><h2>第一个坡：抛弃旧的思维习惯，熟练使用Linux命令行</h2><p>上手Linux的第一步，要先从Windows的思维习惯，切换成Linux的“<strong>命令行+文件</strong>”使用模式。</p><p>在Linux中，无论我们做什么事情，都会有相应的命令工具。虽然这些命令一般会在bin或者sbin目录下面，但是这些命令的数量太多了。如果你事先不知道该用哪个命令，很难通过枚举的方式找到。因此，在这样没有统一入口的情况下，就需要你对最基本的命令有所掌握。</p><p>一旦找到某个命令行工具，替代输入框的是各种各样的启动参数。这些参数怎么填，一般可以通过-h查看help，挨个儿看过去，就能找到相应的配置项；还可以通过man命令，查看文档。无论是什么命令行工具，最终的配置一般会落到一个文件上，只要找到了那个文件，文件中会有注释，也可以挨个儿看下去，基本就知道如何配置了。</p><p>这个过程可能非常痛苦，在没有足够熟练地掌握命令行之前，你会发现干个非常小的事情都需要搜索半天，读很多文档，即便如此还不一定能得到期望的结果。这个时候你一定不要气馁，坚持下去，继续看文档、查资料，慢慢你就会发现，大部分命令的行为模式都很像，你几乎不需要搜索就能完成大部分操作了。</p><p>恭喜你，这个时候你已经爬上第一个坡了。这个时候，你能看到一些很美丽的风景，例如一些很有技巧的命令sed和awk、很神奇的正则表达式、灵活的管道和grep、强大的bash。你可以自动化地做一些事情了，例如处理一些数据，会比你使用Excel要又快又准，关键是不用框框点点，在后台就能完成一系列操作。在处理数据的同时，你还可以干别的事情，半夜处理数据，第二天早上发个邮件报告，这都是Excel很难做到的事情。</p><p>不过，在这个专栏里，命令行并不是我们的重点，但是考虑到一些刚起步的同学，在第一部分我会简单介绍一些能够让你快速上手Linux的命令行。专栏每一模块的第一节，我都会有针对性地讲解这一模块的常用命令，足够你把Linux用起来。</p><p>如果你想全面学习Linux命令，推荐你阅读《<strong>鸟哥的Linux私房菜</strong>》。如果想再深入一点，推荐你阅读《<strong>Linux系统管理技术手册</strong>》。这本砖头厚的书，可以说是Linux运维手边必备。</p><h2>第二个坡：通过系统调用或者glibc，学会自己进行程序设计</h2><p>命令行工具也是程序，只不过是别人写的程序。从用别人写的程序，到自己能够写程序，通过程序来操作Linux，这是第二个要爬的坡。</p><p>用代码操作Linux，可以直接使用Linux系统调用，也可以使用glibc的库。</p><p>Linux的系统调用非常多，而且每个函数都非常复杂，传入的参数、返回值、调用的方式等等都有很多讲究。这里面需要掌握很多Linux操作系统的原理，否则你会无法理解为什么应该这样调用。</p><p>刚开始学Linux程序设计的时候，你会发现它比命令行复杂得多。因为你的角色再次变化，这是为啥呢？我这么说，估计你就能理解了。</p><p><strong>如果说使用命令行的人是吃馒头的，那写代码操作命令行的人就是做馒头的</strong>。看着简简单单的一个馒头，可能要经过N个工序才能蒸出来。同样，你会发现，你平时用的一个简单的命令行，却需要N个系统调用组合才能完成。其中每个系统调用都要进行深入地学习、读文档、做实验。</p><p>经过一段时间的学习，你啃下了这些东西，恭喜你，又爬上了一个坡。这时候，你已经很接近操作系统的原理了，你能看到另一番风景了。大学里学的那些理论，你再回去看，现在就会开始有感觉了。你本来不理解进程树，调用了fork，就明白了；你本来不理解进程同步机制，调用了信号量，也明白了；你本来分不清楚网络应用层和传输层的分界线，调用了socket，都明白了。</p><p>同样，专栏的第一模块，我会简单介绍一下Linux有哪些系统调用，每一模块的第一节，我还会讲解这一模块的常用系统调用，以及如何编程调用这些系统调用。这样可以使你对Linux程序设计入个门，但是这对于实战肯定是远远不够的。如果要进一步学习Linux程序设计，推荐你阅读<strong>《UNIX环境高级编程》</strong>，这本书有代码，有介绍，有原理，非常实用。</p><h2>第三个坡：了解Linux内核机制，反复研习重点突破</h2><p>当你已经会使用代码操作Linux的时候，你已经很希望揭开这层面纱，看看系统调用背后到底做了什么。</p><p>这个时候，你的角色要再次面临变化，<strong>就像你蒸馒头时间长了，发现要蒸出更好吃的馒头，就必须要对面粉有所研究</strong>。怎么研究呢？当然你可以去面粉厂看人家的加工过程，但是面粉厂的流水线也很复杂，很多和你蒸馒头没有直接关系，直接去看容易蒙圈，所以这时候你最好先研究一下，面粉制造工艺与馒头口味的关系。</p><p>对于Linux也是一样的，进一步了解内核的原理，有助于你更好地使用命令行和进行程序设计，能让你的运维和开发水平上升一个层次，但是我不建议你直接看代码，因为Linux代码量太大，很容易迷失，找不到头绪。最好的办法是，先了解一下Linux内核机制，知道基本的原理和流程就可以了。</p><p>一旦学起来的时候，你会发现，Linux内核机制也非常复杂，而且其中相互关联。比如说，进程运行要分配内存，内存映射涉及文件的关联，文件的读写需要经过块设备，从文件中加载代码才能运行起来进程。这些知识点要反复对照，才能理清。</p><p>但是一旦爬上这个坡，你会发现Linux这个复杂的系统开始透明起来。无论你是运维，还是开发，你都能大概知道背后发生的事情，并在出现异常的情况时，比较准确地定位到问题所在。</p><p>Linux内核机制是我们这个专栏重点要讲述的部分，我会基于最新4.x的内核进行讲解，当然我也意识到了内核机制的复杂性，所以我选择通过故事性和图形化的方式，帮助你了解并记住这些机制。</p><p>这块内容的辅助学习，我推荐一本《<strong>深入理解LINUX内核</strong>》。这本书言简意赅地讲述了主要的内核机制。看完这本书，你会对Linux内核有总体的了解。不过这本书的内核版本有点老，不过对于了解原理来讲，没有任何问题。</p><h2>第四个坡：阅读Linux内核代码，聚焦核心逻辑和场景</h2><p>在了解内核机制的时候，你肯定会遇到困惑的地方，因为理论的描述和提炼虽然能够让你更容易看清全貌，但是容易让你忽略细节。</p><p>我在看内核原理的书的时候也遇到过这种问题，有的地方实在是难以理解，或者不同的书说的不一样，这时候该怎么办呢？其实很好办，Linux是开源的呀，我们可以看代码呀，代码是精准的。哪里有问题，找到那段代码看一看，很多问题就有方法了。</p><p>另外，当你在工作中需要重点研究某方面技术的时候，如果涉及内核，这个时候仅仅了解原理已经不够了，你需要看这部分的代码。</p><p>但是开源软件代码纷繁复杂，一开始看肯定晕，找不着北。这里有一个诀窍，就是<strong>一开始阅读代码不要纠结一城一池的得失，不要每一行都一定要搞清楚它是干嘛的，而要聚焦于核心逻辑和使用场景</strong>。</p><p>一旦爬上这个坡，对于操作系统的原理，你应该就掌握得比较清楚了。就像蒸馒头的人已经将面粉加工流程烂熟于心。这个时候，你就可以有针对性地去做课题，把所学和你现在做的东西结合起来重点突破。例如你是研究虚拟化的，就重点看KVM的部分；如果你是研究网络的，就重点看内核协议栈的部分。</p><p>在专栏里，我在讲述Linux原理的同时，也会根据场景和主要流程来分析部分代码，例如创建进程、分配内存、打开文件、读写文件、收发网络包等等。考虑到大量代码粘贴会让你看起来比较费劲，也会占用大量篇幅，所以我采取只叙述主要流程，只放必要的代码，大部分的逻辑和相互关系，尽量通过图的方式展现出来，给你讲解。</p><p>这里也推荐一本书，《<strong>LINUX内核源代码情景分析</strong>》。这本书最大的优点是结合场景进行分析，看得见、摸得着，非常直观，唯一的缺点还是内核版本比较老。</p><h2>第五个坡：实验定制化Linux组件，已经没人能阻挡你成为内核开发工程师了</h2><p>纸上得来终觉浅，绝知此事要躬行。从只看内核代码，到上手修改内核代码，这又是一个很大的坎。这相当于蒸馒头的人为了定制口味，要开始修改面粉生产流程了。</p><p>因为Linux有源代码，很多地方可以参考现有的实现，定制化自己的模块。例如，你可以自己实现一个设备驱动程序，实现一个自己的系统调用，或者实现一个自己的文件系统等等。</p><p><img src="https://static001.geekbang.org/resource/image/9e/85/9e970ed142da439f6fbe6d7c06f11785.jpeg?wh=1729*1702" alt=""></p><p>这个难度比较大，涉及的细节比较多，上一个阶段，我的建议是不计较一城一地的得失，不需要每个细节都搞清楚，这一个阶段要求就更高了。一旦代码有一个细微的bug，都有可能导致实验失败。</p><p>专栏最后一个部分，我专门设计了两个实验，帮你度过这个坎。只要跟着我的步伐进行学习，接下来，就没人能够阻挡你成为一名内核开发工程师了。</p><h2>最后一个坡：面向真实场景的开发，实践没有终点</h2><p>说了这么多，我们都只是走出了万里长征第一步。我始终坚信，真正的高手都是在实战中摸爬滚打练出来的。</p><p>如果你是运维，仅仅熟悉上面基本的操作是不够的，生产环境会有大量的不可控因素，尤其是集群规模大的更是如此，大量的运维经验是实战来的，不能光靠读书。如果你是开发，对内核进行少量修改容易，但是一旦面临真实的场景，需要考虑各种因素，并发与并行，锁与保护，扩展性和兼容性，都需要真实项目才能练出来。</p><h2>总结时刻</h2><p>今天，我把爬坡的过程，分解成了六个阶段，并给你分享了我的私家爬坡宝典。你都记住了吗？我把今天的内容总结成了下面这张图。建议你牢牢记住这张图，在接下来的四个月中，按照这个路径稳步前进，攻克Linux操作系统。</p><p><img src="https://static001.geekbang.org/resource/image/bc/5b/bcf70b988e59522de732bc1b01b45a5b.jpeg?wh=2695*1492" alt=""></p><center><span class="reference">Linux操作系统爬坡路线图</span></center><h2>课堂练习</h2><p>你可以结合第一节的测试结果，并根据我今天讲的爬坡方法，思考一下，在接下来的四个月里，你准备怎么学习这个专栏。</p><p>欢迎在留言区写下你的<strong>爬坡计划</strong>，也欢迎你把今天的文章分享给你的朋友，和他一起学习、进步。</p><p><span class="orange">编辑乱入：超哥推荐的图书，部分已上架极客时间商城，点击下方图片，即可购买。和专栏一起配合使用，学习效果会更好哦！</span></p><p><a href="https://h5.youzan.com/v2/feature/2VpBYpR2As"><img src="https://static001.geekbang.org/resource/image/6b/68/6bae103a79601bddd51b87f4e838e868.jpg?wh=900*720" alt=""></a></p><p><a href="time://mall?url=https%3A%2F%2Fj.youzan.com%2FG69gDi"><img src="https://static001.geekbang.org/resource/image/00/f2/00f868b7654dcb50ae2c91fd7688d2f2.jpg?wh=1242*526" alt="unpreview"></a><br>
限量发售中，仅限<span class="orange">5000份</span>，3大体系，22个模块，定位工作中80%的高频问题。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/75/69/791d0f5e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>rocedu</span>
  </div>
  <div class="_2_QraFYR_0">别出心裁的Linux命令学习法https:&#47;&#47;www.cnblogs.com&#47;rocedu&#47;p&#47;4902411.html, 我这篇博客可以让你尽快过第一关。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-29 03:40:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/a4/39/6a5cd1d8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>sotey</span>
  </div>
  <div class="_2_QraFYR_0">学习计划：<br>我是做运维的<br>1、第一件事情当然是安装Linux操作系统了，我用虚拟机安装了CentOS7，第一遍升级到ml居然是5.0内核，内核还是升级到lt4.x，- -~！<br>2、买书！！！Unix环境编程、深入理解Linux内核、Linux源代码情景分析，又多了三本讲操作系统的了，- -~！<br>3、自我心理疏导，逃不掉啊，还是要面对C&#47;C++<br>4、学习计划<br>4.1 每周一三五读专栏，至少两遍，记录下不清楚的地方，并且提问最想知道的知识点<br>4.2 每周六、日带小孩- -~!抽空回顾本周三篇专栏，查漏补缺，搜索和翻阅书籍至少摘抄一边周一三五不清楚的地方<br>4.3 每周日晚上总结一周学习效果，计划下一周冲刺<br>4.4 每周计划包括预习，根据老师的课程目录提前查阅资料，例如将要讲进程调度的时候，查阅一下讲操作系统概念的书。如老师要讲代码了，提前复习一下C&#47;C++语言。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-29 01:38:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/1d/13/31ea1b0b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>峰</span>
  </div>
  <div class="_2_QraFYR_0">感觉4个月后，在阶段3就ok了。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-29 06:37:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/75/69/791d0f5e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>rocedu</span>
  </div>
  <div class="_2_QraFYR_0">第二个坡，读了《UNIX 环境高级编程》当然可以一览众山小，我的学习和教学经验更推荐《Unix&#47;Linux编程实践教程》，当年我是先读《UNIX 环境高级编程》，后读的《Unix&#47;Linux编程实践教程》，要是反过来，学习起来会更好。我的博客[别出心裁的Linux系统调用学习法](http:&#47;&#47;www.cnblogs.com&#47;rocedu&#47;p&#47;6016880.html)可以让你快速入门Linux系统调用，掌握学习方法。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-29 03:53:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/75/69/791d0f5e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>rocedu</span>
  </div>
  <div class="_2_QraFYR_0">爬第三个坡，我推荐看一下《庖丁解牛Linux内核分析》（https:&#47;&#47;book.douban.com&#47;subject&#47;30350365&#47;），跟着MOOC一起自己动手搭建Linux内核，相当于自己动手做面粉厂。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-29 03:57:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/0f/ab/9748f40b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>微秒</span>
  </div>
  <div class="_2_QraFYR_0">我觉得只要让我了解操作系统的原理，能解决面试和用linux做上层的程序设计就够了。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-29 08:53:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/f1/15/8fcf8038.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>William</span>
  </div>
  <div class="_2_QraFYR_0"># 学习路径<br><br>## Step1: 熟悉Linux命令行<br><br>基础：--help、man<br>进阶：sed、awk、正则、管道、grep、find、shell脚本、vim、git<br><br>## Step2: 系统调用和glibc =&gt; 编程<br><br>+ 进程树 fork<br>+ 进程同步 信号量<br>+ 应用层与传输层的分界线 socket编程<br><br>&gt; [《UNIX环境高级编程》]()<br><br>## Step3: Linux内核机制<br><br>&gt; [《深入理解Linux内核》]()<br>&gt; 这本书内核版本比较老~<br><br>## Step4: 阅读Linux内核源码，聚焦核心逻辑和场景<br><br>+ 虚拟化 kvm<br>+ 网络 内核协议栈<br><br>&gt; [《Linux内核源码情景分析》]()<br><br>## Step5：实验定制化Linux组件<br><br>&gt; 专栏最后两个实验<br><br>## Step6: 面向真实场景开发，实践~<br><br>+ 并发与并行<br>+ 锁与保护<br>+ 扩展性和兼容性<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-29 15:12:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83erYNBIwAj3KdIXaXbeMBUjTMz31zAToHIJSdo7oQk8bfsibwViaLobVQ8miatwBlC5spLS9kVCzHMjUA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>苏籍</span>
  </div>
  <div class="_2_QraFYR_0">作为一个Linux 小白，同时作为一名java开发，定专栏时给自己定的目标就是能够到达第三阶段<br>首先能够熟练使用Linux命令行，能做正常的系统运维操作。结合专栏和鸟哥的私房菜（已经买了3，4年没看&#47;捂脸）<br>第二、能够进行基本的linux程序设计，实现代码调用linux操作，主要跟专栏进行实践<br>第三、了解linux内核机制和相关原理<br>不要求多，先上坡，上坡就一定看到不一样的风景<br>最后实践实践实践，坚持坚持坚持，前进前进前进</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-29 16:51:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/PvD4ZpoY41ltqyxZq6Xxl0zmpb9jEPYibBRE5TSe87ec788cBHYWEYl2Rsy7BVZNjNFcWP95TCB4Iopv5HKWrjw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>乄恰似一种蜕变</span>
  </div>
  <div class="_2_QraFYR_0">做为一名Java开发，认为还是要靠实践出真知，主要是跟着刘老师的专栏进行学习<br>第一个坡：阅读《鸟哥的Linux私房菜》，实践Linux常用命令<br>第二个坡：阅读《《UNIX环境高级编程》，学习如何进行Linux程序设计<br>其它的还没有想到这么远，一个个坡爬吧，不要一直停留在山脚就可以，希望自己每爬一个坡，都能体验到不同的风景。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-06 11:10:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/41/27/3ff1a1d6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>hua168</span>
  </div>
  <div class="_2_QraFYR_0">老师，这些书能不能加一个链接呢，比如豆瓣的链接地址，有些名有重名的，作者却不同。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 选好评最高的，哈哈</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-29 01:55:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/ca/e3/447aff89.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>记事本</span>
  </div>
  <div class="_2_QraFYR_0">说实话  这个专栏68元真的是捡到宝了。谢谢老师。好好学习，天天向上！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-31 20:08:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/59/0b/fb876077.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>michael</span>
  </div>
  <div class="_2_QraFYR_0">鸟哥的私房菜看了一小半了，通俗易懂，强烈推荐啊*^_^*</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-29 08:12:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/8c/e4/ad3e7c39.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>L.</span>
  </div>
  <div class="_2_QraFYR_0">鸟哥的文档一开始读起来对部分名词不适应，后来越读越觉得台湾腔很可爱</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 但是命令讲解很丰富，重点看命令</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-21 19:47:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c7/40/66a203cd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>江南皮革厂研发中心保安队长</span>
  </div>
  <div class="_2_QraFYR_0">老师，我工作是负责搞自动化运维的，目前只会python编程和shell编程，针对自动化运维研究内核的话我应该从哪一块提高，是不是要把C再滚一遍呀。-_-</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以复习一下c</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-29 08:25:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/ecruTrMgzvqIs5iaWVibZw4Rxic42ZXGTflvOFHiaZEkf32Su01gDCWT8tdIcEoybg0ibAYU2Q8f9bleL7Q37fKguxQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>做一个积极的跳蚤</span>
  </div>
  <div class="_2_QraFYR_0">我的学习计划：<br>1.跳过第一步，因为本人从事Linux下c&#47;c++开发四五年了，命令行基本熟练；<br>2.第二个坡：linux程序设计，《unix环境高级编程》也看过了，但是基本没有引用，大部分系统函数还是知道的，这里选择学习rocedu极友推荐的《Unix&#47;Linux编程实践教程》，加深哈理解。<br>3.第三个坡：《深入理解Linux内核》书买了一年了还没开始学，这里计划辅助《庖丁解牛Linux内核分析》跟着刘老师课程三个方面一起学习。<br>4.第四个坡：《Linux内核源码情景分析》跟着刘老师学习，同时看看《linux 驱动开发》一书<br>5第五个坡：跟着刘老师学。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-06-15 11:40:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/a5/49/5bf449f2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>安幕风尘笑为茜</span>
  </div>
  <div class="_2_QraFYR_0">我是做测试的，前几天刚啃完 【鸟哥Linux私房菜】，爬第二个坡，目标第三个。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞，鸟哥Linux私房菜很赞</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-04 10:34:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/0b/35/2c56c29c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>arthur</span>
  </div>
  <div class="_2_QraFYR_0">我能爬完第二个坡我就很开心了，鸟叔私房菜和Unix高级编程我都有，都看了一点就看不下去了😂 </div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-29 22:34:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/48/e4/974c38d0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小牛人</span>
  </div>
  <div class="_2_QraFYR_0">这节课让我回想起了嵌入式linux的学习历程。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-29 01:30:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://wx.qlogo.cn/mmopen/vi_32/y43slZByeeYvm9Nkfry5aWqc1axTYjkzVJTHQEVtryANXlCpn4vpWezYQWrT1uuQ5ic1b8HiaUX6DFdpdTJtK5Kw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_f026c5</span>
  </div>
  <div class="_2_QraFYR_0">向第5个坡爬<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 牛</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-11 10:50:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJiaZlM5Jfa8nSkAYTXRfib3ytDLFsWNlHndBu9JbDyA8cERkmOFdqia4wfgjPzR5natDCwqicMenYBhQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>liiiiiii</span>
  </div>
  <div class="_2_QraFYR_0">老师 能不能教一下安装Linux虚拟机</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个网上教程很多的。可以用公有云，就不用安装了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-31 23:04:30</div>
  </div>
</div>
</div>
</li>
</ul>