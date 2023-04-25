<audio title="03 _ 你可以把Linux内核当成一家软件外包公司的老板" src="https://static001.geekbang.org/resource/audio/50/9a/50def8552f29579af4fe4dd67af1a39a.mp3" controls="controls"></audio> 
<p>在平时的生活中，我们几乎时时刻刻都在使用操作系统，只是大部分时间你都意识不到它的存在。比如你买了一部手机或者一台平板电脑，立马就能上手使用，这是因为它们里面都预先安装了操作系统。</p><p>所以啊，哪有什么岁月静好，只不过有人替你负重前行。而操作系统就扮演了这样一个负重前行的角色。那操作系统到底在背后默默地做了哪些事情，才能让我们轻松地使用这些电子设备呢？要想回答这个问题，我们需要把眼光放回到自己攒电脑的那个时代。</p><h2>电脑组装好就能直接用吗？</h2><p>那时候买电脑，经常是这样一个情景：三五个哥们儿一起来到电脑城，呼啦呼啦采购了一大堆硬件，有密密麻麻都是针脚的<strong>CPU</strong>；有铺满各种复杂电路的一块板子，也就是<strong>主板</strong>；还需要买块<strong>显卡</strong>，用来连接显示器；还需要买个<strong>网卡</strong>，里面可以插网线；还要买块<strong>硬盘</strong>，将来用来存放文件；然后还需要一大堆<strong>线</strong>，将这些设备和主板连接起来；最终再来一个<strong>鼠标</strong>，一个<strong>键盘</strong>，还有一个<strong>显示器</strong>。设备差不多啦，准备开整！</p><p><img src="https://static001.geekbang.org/resource/image/ed/45/ed03667738a92d66626914fe5dc78d45.png?wh=1504*1008" alt=""></p><p>好不容易组装完这一大堆硬件，还是不能直接用，你还需要安装一个操作系统。安装操作系统也是一件非常复杂的事，一点儿也不亚于把刚才那堆东西组装起来。这个安装过程可能会涉及十几个步骤、几十项配置。每一步骤配置完了，点击下一步，会出现个进度条。伴随着一堆难以理解的描述，最终安装步骤到达百分之百，才出现你熟悉的那个界面。</p><!-- [[[read_end]]] --><p>我这么说起来好像很容易，但是要把这事儿讲清楚估计得用一个专栏。这个复杂程度，咱们父母估计是上不了手了。所以，那个时候，能把这套东西都组装起来，是一件很拉风的事情。很多IT男甚至因为这项绝技“泡”到了妹子。</p><p>当操作系统安装完毕的时候，我妈通常会要求我一定要装一个QQ。看到妈妈在你装好的操作系统前愉快地和她的朋友聊天，这时候，经历过以上过程的你，多少应该能感受到操作系统的厉害了。</p><p><strong>操作系统究竟是如何把这么多套复杂的东西管理起来，从而弄出来一个简单到父母都会用的东西呢？</strong></p><p>很多事情就怕细想。不知道你有没有产生过这些疑问：</p><ul>
<li>
<p>桌面上的图标到底是啥？凭啥我在鼠标上一双击，就会出来一个美丽的画面？这都是从哪里跑出来的？</p>
</li>
<li>
<p>凭什么我在键盘上噼里啪啦地敲，某个位置就会显示我想要的那些字符？</p>
</li>
<li>
<p>电脑怎么知道我鼠标点击的是这个地方，又是怎么知道我要输入的是这个地方？</p>
</li>
<li>
<p>我在键盘上点“a”，是谁在显示器上画出“a”这个图像呢？</p>
</li>
<li>
<p>为什么我一回车，这些字符就发到遥远的另外一台机器上去了？</p>
</li>
</ul><p>对于普通用户来讲，其实只要会用就行了，但是咱们作为专业人士，要深入探究一下背后的答案。你别小看“双击鼠标打开聊天软件”这样一个简单的操作，它几乎涵盖了操作系统的所有功能。我们就从这个熟悉的操作，来认识陌生的操作系统。</p><p><span class="orange">操作系统其实就像一个软件外包公司，其内核就相当于这家外包公司的老板。所以接下来的整个课程中，请你将自己的角色切换成这家软件外包公司的老板，设身处地地去理解操作系统是如何协调各种资源，帮客户做成事情的。</span></p><p>想要学好咱们这门课，你要牢牢记住这段话，把这个概念牢牢扎根在心里，我之后的讲解都会基于此，帮你理解、记忆那些难搞的概念和原理。</p><p>同时，为了防止你混淆，我这里先强调一下。今后我所说的“用户”，都是指操作系统的用户，“客户”则是指外包公司的客户，这两者是对应的。</p><h2>“双击QQ”这个过程，都需要用到哪些硬件？</h2><p>好，现在用户开始对着屏幕上的QQ图标双击鼠标了。</p><p><strong>鼠标和键盘</strong>是计算机的<strong>输入设备</strong>。大部分的普通用户想要告诉计算机应该做什么，都是通过这两个设备。例如，用户移动了一下鼠标，鼠标就会通过鼠标线给电脑发消息，告知电脑，鼠标向某个方向移动了多少距离。</p><p>如果是一家外包公司，怎么才能知道客户的需求呢？你需要配备销售、售前等角色，专门负责和客户对接，把客户需求拿回来，我们把这些人统称为<strong>客户对接员</strong>。你可以跟客户说，有什么事儿都找对接员。</p><p>屏幕，也就是<strong>显示器</strong>，是计算机的<strong>输出设备</strong>，将计算机处理用户请求后的结果展现给客户，要不然用户无法知道自己的请求是不是到达并且执行了。</p><p>显示器上面显示的东西是由<strong>显卡</strong>控制的。无论是显示器还是显卡，这里都有个“坐标”的概念，也就是说，什么图像在哪个坐标，都是定义好了才画上去的。本来在某个坐标画了一个鼠标箭头，当接到鼠标移动的事件之后，你应该按相同的方向，按照一定的比例（鼠标灵敏度），在屏幕的某个坐标再画一个鼠标箭头。</p><p>作为外包公司，当客户给你提了需求，不管你做还是不做，最终做成什么样，你都需要给客户反馈，所以你要配备交付人员，将做好的需求展示给他们看。</p><p>在操作系统中，<strong>输入设备驱动</strong>其实就是<strong>客户对接员</strong>。有时候新插上一个鼠标的时候，会弹出一个通知你安装驱动，这就是操作系统这家外包公司给你配备对接人员呢。当客户告诉对接员需求的时候，对于操作系统来讲，输入设备会发送一个中断。这个概念很好理解。客户肯定希望外包公司把正在做的事情都停下来服务它。所以，这个时候客户发送的需求就被称为<strong>中断事件</strong>（Interrupt Event）。</p><p>显卡会有<strong>显卡驱动</strong>，在操作系统中称为<strong>输出设备驱动</strong>，也就是上面说的<strong>交付人员</strong>。</p><h2>从点击QQ图标，看操作系统全貌</h2><p>有了<strong>客户对接员</strong>和<strong>交付人员</strong>，外包公司就可以处理用户“在桌面上点击QQ图标”的事件了。</p><p>首先，鼠标双击会触发一个中断，这相当于客户告知客户对接员“有了新需求，需要处理一下”。你会事先把处理这种问题的方法教给客户对接员。在操作系统里面就是调用中断处理函数。操作系统发现双击的是一个图标，就明白了用户的原始诉求，准备运行QQ和别人聊天。</p><p>你会发现，运行QQ是一件大事，因为将来的一段时间，用户要一直和QQ进行交互。这就相当于你们公司接了一个大单，而不是处理零星的客户需求，这个时候应该单独立项。一旦立了项，以后与这个项目有关的事情，都由这个项目组来处理。</p><p>立项可不能随便立，一定要有一个<strong>项目执行计划书</strong>，说明这个项目打算怎么做，一步一步如何执行，遇到什么情况应该怎么办等等。换句话说，对QQ这个程序来说，它能做哪些事情，每件事情怎么做，先做啥后做啥，都已经作为程序逻辑写在程序里面，并且编译成为二进制了。这个程序就相当于项目执行计划书。</p><p>电脑上的程序有很多，什么有道云笔记的程序、Word程序等等，它们都以二进制文件的形式保存在硬盘上。硬盘是个物理设备，要按照规定格式化成为文件系统，才能存放这些程序。文件系统需要一个系统进行统一管理，称为<strong>文件管理子系统</strong>（File Management Subsystem）。</p><p>对于你们公司，项目立得多了，项目执行计划书也会很多，同样需要有个统一保存文件的档案库，而且需要有序地管理起来。</p><p>当你从资料库里面拿到这个项目执行计划书，接下来就需要开始执行这个项目了。项目执行计划书是静态的，项目的执行是动态的。</p><p>同理，当操作系统拿到QQ的二进制执行文件的时候，就可以运行这个文件了。QQ的二进制文件是静态的，称为<strong>程序</strong>（Program），而运行起来的QQ，是不断进行的，称为<strong>进程</strong>（Process）。</p><p>说了这么多，怎样才能立项呢？你会发现，一个项目要想顺畅进行，需要用到公司的各种资源，比如说盖个公章、开个证明、申请个会议室、打印个材料等等。这里有个两难的权衡，一方面，资源毕竟是有限的，甚至是涉及机密的，不能由项目组滥取滥用；另一方面，就是效率，咱是一个私营企业，保证项目申请资源的时候只跑一次，这样才能比较高效。</p><p>为了平衡这一点，一方面涉及核心权限的资源，还是应该被公司严格把控，审批了才能用；另外一方面，为了提高效率，最好有个统一的办事大厅，明文列出提供哪些服务，谁需要可以来申请，然后就会有回应。</p><p>在操作系统中，也有同样的问题，例如多个进程都要往打印机上打印文件，如果随便乱打印进程，就会出现同样一张纸，第一行是A进程输出的文字，第二行是B进程输出的文字，全乱套了。所以，打印机的直接操作是放在操作系统内核里面的，进程不能随便操作。但是操作系统也提供一个办事大厅，也就是<strong>系统调用</strong>（System Call）。</p><p>系统调用也能列出来提供哪些接口可以调用，进程有需要的时候就可以去调用。这其中，立项是办事大厅提供的关键服务之一。同样，任何一个程序要想运行起来，就需要调用系统调用，创建进程。</p><p>一旦项目正式立项，就要开始执行，就要成立项目组，将开发人员分配到这个项目组，按照项目执行计划书一步一步执行。为了管理这个项目，我们还需要一个项目经理、一套项目管理流程、一个项目管理系统，例如程序员比较熟悉的Jira。如果项目多，可能一个开发人员需要同时执行多个项目，这就要考验项目经理的调度能力了。</p><p>在操作系统中，进程的执行也需要分配CPU进行执行，也就是按照程序里面的二进制代码一行一行地执行。于是，为了管理进程，我们还需要一个<strong>进程管理子系统</strong>（Process Management Subsystem）。如果运行的进程很多，则一个CPU会并发运行多个进程，也就需要CPU的调度能力了。</p><p>每个项目都有自己的私密资料，这些资料不能被其他项目组看到。这些资料主要是项目在执行的过程中，产生的很多中间成果，例如架构图、流程图。</p><p>执行过程中，难免要在白板上或者本子上写写画画，如果不同项目的办公空间不隔离，一方面，项目的私密性不能得到保证，A项目的细节，B项目也能看到；另一方面，项目之间会相互干扰，A项目组的人刚在白板上画了一个架构图，出去上个厕所，结果B项目组的人就给擦了。</p><p>如果把不同的项目组分配到不同的会议室，就解决了这个问题。当然会议室是有限的，需要有人管理和分配，并且需要一个<strong>会议室管理系统</strong>。</p><p>在操作系统中，不同的进程有不同的内存空间，但是整个电脑内存就这么点儿，所以需要统一的管理和分配，这就需要<strong>内存管理子系统</strong>（Memory Management Subsystem）。</p><p>如果想直观地了解QQ如何使用CPU和内存，可以打开任务管理器，你就能看到QQ这个进程耗费的CPU和内存。</p><p>项目执行的时候，有了一定的成果，就要给客户演示。例如客户说要做个应用，我们做出来了要给客户看看，如果客户说哪里需要改，可以根据客户的需求再改，这就需要交付人员了。</p><p>QQ启动之后，有一部分代码会在显示器上画一个对话框，并且将键盘的焦点放在了输入框里面。CPU根据这些指令，就会告知显卡驱动程序，将这个对话框画出来。</p><p>于是使用QQ的用户就会很开心地发现，他能和别人开始聊天了。</p><p>当用户通过键盘噼里啪啦打字的时候，键盘也是输入设备，也会触发中断，通知相应的输入设备驱动程序。</p><p>我们假设用户输入了一个“a”。这就像客户提出了新的需求给客户对接员。客户对接员收到需求后，因为是对接这个项目的，所以就回来报告，客户提新需求了，项目组需要处理一下。项目执行计划书里面一般都会有当遇到何种需求应该怎么做的规定，项目组就按这个规定做了，然后让交付人员再去客户那里演示就行了。</p><p>对于QQ来讲，由于键盘闪啊闪的焦点在QQ这个对话框上，因而操作系统知道，这个事件是给这个进程的。QQ的代码里面肯定有遇到这种事件如何处理的代码，就会执行。一般是记录下客户的输入，并且告知显卡驱动程序，在那个地方画一个“a”。显卡画完了，客户看到了，就觉得自己的输入成功了。</p><p>当用户输入完毕之后，回车一下，还是会通过键盘驱动程序告诉操作系统，操作系统还是会找到QQ，QQ会将用户的输入发送到网络上。QQ进程是不能直接发送网络包的，需要调用系统调用，内核使用网卡驱动程序进行发送。</p><p>这就像客户对接员接到一个需求，但是这个需求需要和其他公司沟通，这就需要依靠公司的对外合作部，对外合作部在办事大厅有专门的窗口，非常方便。</p><p><img src="https://static001.geekbang.org/resource/image/e1/4a/e15954f1371a4c782f028202dce1f84a.jpeg?wh=3023*1709" alt=""></p><h2>总结时刻</h2><p>到这里，一个外包公司大部分的职能部门都凑齐了。你可以对应着下图的操作系统内核体系结构，回顾一下它们是如何组成一家公司的。</p><p>QQ的运行过程，只是一个简单的比喻。在后面的章节中，我会展开讲述每个部分是怎么工作的，最后我会再将这个过程串起来，这样你就能了解操作系统的全貌了。</p><p><img src="https://static001.geekbang.org/resource/image/21/f5/21a9afd64b05cf1ffc87b74515d1d4f5.jpeg?wh=2369*2216" alt=""></p><center><span class="reference">操作系统内核体系结构图</span></center><h2>课堂练习</h2><p>学习Linux，看代码是必须的。你可以找到最新版本的Linux代码，在里面找找，这几个子系统的代码都在哪里。</p><p>欢迎留言和我分享你的思考和疑问，也欢迎你把今天的内容分享给你的朋友，和他一起学习、进步。</p><p><img src="https://static001.geekbang.org/resource/image/8c/37/8c0a95fa07a8b9a1abfd394479bdd637.jpg?wh=1110*659" alt=""></p>
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
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJYzicZCK7z3OOEC3TyVn16t8iclz2jniaHEHuhnabh4qgcfeuGxS5WzolmVEwB90Co9A6lS04ukNziaA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>阿秭</span>
  </div>
  <div class="_2_QraFYR_0">对于什么办事大厅这种东西不熟悉的我，这个比喻特别凌乱。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以抛开比喻直接看干货，这次写作将比喻和干货分的比较开，可以满足愿意看比喻和不愿意看比喻的人</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-08 17:23:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/fd/4f/d14f8993.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ThisCat</span>
  </div>
  <div class="_2_QraFYR_0">我这里是在git上下载的linux源代码：<br>下载链接：https:&#47;&#47;github.com&#47;torvalds&#47;linux<br>kernel:内核管理核心代码，其中包含了进程管理子系统<br>fs（file system）:文件管理子系统<br>mm(memeroy mange):内存管理子系统，这里更多的是CPU体系结构的内存管理，与具体物理内存管理相关的代码在 arch&#47;（某种架构）&#47;mm<br>net:网络子系统<br>drivers:设备子系统，其中存放各种硬件的驱动程序，drivers&#47;block 下存放块设备的驱动程序</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-01 14:46:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/b0/ed/53c02651.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>PXQ</span>
  </div>
  <div class="_2_QraFYR_0">我本来是想通过外包公司的比喻了解操作系统，却反而通过操作系统学习了外包公司</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 哈哈</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-06 22:17:10</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/d7/23/14b98ea5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>TinnyFlames</span>
  </div>
  <div class="_2_QraFYR_0">个人的理解，操作系统本质上是构建的一层抽象层，用来屏蔽复杂的底层硬件，向上层用户提供一种“假象”。 CPU(单核情况)，实际上是只有一个的，在一个特定时刻也只可能有一个程序跑在一个CPU上(因为寄存器只有一组)，但是我们在上层观察到的却是系统上好像同时运行着那么多的程序，这实际上是操作系统用进程这个概念对CPU做的抽象。<br>内存也是相似的概念，真实的内存和我们程序员看到的内存截然不同，操作系统通过内存映射等一系列技术让上层的程序员以为自己在操作一片连续的内存空间，实际上这只是操作系统对内存的抽象，是操作系统给程序员的幻象。<br>文件系统也是如此，我们看到的所谓的abcd盘符，真实的情况可能是一块机械硬盘，要找数据必须来回的寻道，找到数据的位置，操作系统通过一系列的操作，把如此复杂的过程层层抽象，抽象出了上层看起来简单的文件系统。<br>非常期待老师的新课，希望老师在用故事精彩的讲解概念的时候，也可以适当的补充一些基础的理论知识。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-02 11:14:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/7a/88/b787338a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>devna</span>
  </div>
  <div class="_2_QraFYR_0">系统调用 kernel&#47;<br>进程管理 kernel&#47;, arch&#47;&lt;arch&gt;&#47;kernel<br>内存管理 mm&#47;, arch&#47;&lt;arch&gt;&#47;mm<br>文件 fs&#47;<br>设备 drivers&#47;char, drivers&#47;block<br>网络 net&#47;<br><br>References:<br>* https:&#47;&#47;www.kernel.org&#47;<br>* https:&#47;&#47;courses.linuxchix.org&#47;kernel-hacking-2002&#47;08-overview-kernel-source.html</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-01 15:02:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/e9/29/629d9bb0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>天王</span>
  </div>
  <div class="_2_QraFYR_0">总结：操作系统就像一个软件外包公司，内核就是公司的老板，需要了解内核是怎么协调资源工作的。点击了一个qq程序，首先是输入设备驱动，他们是公司客户的对接员，客户输了一个指令，首先会中断，调用一个中断处理函数，弄明白客户的指令是什么，然后开始立项，立项就需要项目计划书，即项目的二进制程序的逻辑，设定好了执行的步骤，操作系统拿到二进制文件，就可以运行了，运行的qq称为进程。二进制程序是保存在硬盘上的，需要用到文件存储系统。项目立项需要用到各种审批和资源调度，有一些操作放在系统内核，不能随便调用，统一在办公大厅，即系统调用，系统调用会列出哪些接口可以调用，有需要的时候调用。项目实际运行过程中，为了管理，进程的进行也需要分配cpu执行，为了管理进程，进程管理子系统，如果进程管理很多，需要cpu调度执行，需要看cpu的调度能力。项目执行过程中需要做隔离，否则会互相影响，需要用到会议室，即内存，内存有限，需要内存管理子系统，执行完毕之后，统一交给交付员显示给客户，即输出设备输出设备驱动。qq启动之后，有一部分代码会告诉启动一个对话框，并将鼠标焦点移到对话框，cpu收到指令之后，告知显卡程序，将对话框调出来，呈现给用户，用户用键盘输入，输入设备会触发中断，调用输入设备驱动程序，用户输入了一个a，焦点在这个进程上，所以操作系统知道发给哪个进程，然后交给qq程序处理，qq程序记录下客户的输入，告诉显卡程序，花一个a，画完了，客户就能看到了。输入成功之后，按enter键，通过键盘驱动程序告诉系统，系统找到qq，将用户的输入发送到网络上。qq进程需要调用系统调用，内核使用网卡驱动发送。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-01 21:26:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/3b/73/b17dc8fa.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>～～</span>
  </div>
  <div class="_2_QraFYR_0">非常同意一楼楼主的观点，感觉没有这些比喻更容易让人理解，有了比喻反而更加凌乱。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-15 16:06:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/0e/1f/d0472177.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>厉害了我的国</span>
  </div>
  <div class="_2_QraFYR_0">一会外包公司，一会操作系统，脑裂了～</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 比喻。其实跳过比喻，内容也是自洽的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-02 10:54:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/15/ef/ed/16545faf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Gavin</span>
  </div>
  <div class="_2_QraFYR_0">作为新手想问一下该怎么看Linux内核代码啊，是下一个源码包吗。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: https:&#47;&#47;elixir.bootlin.com&#47;linux&#47;v4.13.16</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-01 08:49:44</div>
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
  <div class="_2_QraFYR_0">疑问：文件管理子系统、进程管理子系统、系统调用、内存管理子系统、网络管理子系统、设备子系统都确实属于Linux内核的一部分吗？虽反复读了几遍，己确定这几个子系统属于内核的一部分，但是还是想向老师确认一下。<br>一、硬背10遍<br>      操作系统像一家软件外包公司，内核就是这家公司的老板。所以接下来整个课程中，请将自己想象成这家公司的老板，设身处地的去理解操作系统是如何协调各种资源，帮客户办成事情。<br>二、知识点<br>1、内核通过协调操作系统各种资源，实现各种功能；自补充：系统资源最终还是指时间和空间资源。<br>2、内核通过系统调用、文件管理子系统、进程管理子系统、内存管理子系统、网络管理子系统、设备子系统协调和管理着操作系统的各项源，是完成协调工作的具体实现载体。<br>3、系统调用是系统功能调用的统一入口，相当于办事大厅，项目请求公司资源的统一入口<br>4、进程管理子系统管理进程生命周期和资源，相当于项目管理系统，管理项目生命周期和资源<br>5、内存管理子系统对内存进行管理、分配、回收、隔离，相当于会议室管理系统<br>6、文件子系统管理文件，相当于档案管理系统，管理项目文档资料<br>7、设备子系统管理输入输出设备，相当于客户对接人和交付人员，管理项目的输入输出<br>8、网络子系统，网络协议栈和收发网络包，相当于对外合作部，需要和其他公司合作沟通项目<br>三、记忆理解方法随着老师的思路，映射记忆和图形记忆。<br>四、理解鼠标双击QQ，操作系统干了什么<br>1、鼠标是输入设备，鼠标驱动负责鼠标与设备子系统通信；<br>2、鼠标双击QQ图标时，鼠标发送硬中断，内核执行硬中断处理函数，快速处理硬中断信息，将鼠标双击QQ的必要信息和数据复制到内存，调起软中断处理函数，结束硬中断，软中断执行硬中断未完成后续任务；<br>这里产生疑问记录下来：第一，我理解是否正确；第二，软中断接手硬中断未完任务，是不是软中断告诉1号进程使用QQ的程序创建QQ进程。这里的疑问暂时不扩展阅读，收敛起来，否则无法完成本课时学习总结。<br>3、软中断唤起1号进程，1号进程进入CPU，根据传入的信息，知道QQ程序存放虚拟文件系统位置，实际是把位置信息从内存装载到CPU寄存器中，然后创建新进程，即QQ进程；<br>4、创建QQ进程，QQ进程进入就绪状态，被调入CPU运行处于运行状态，根据程序这个计划书，加载相应数据到内存，执行二进制的指令；<br>5、QQ启动后需要显示到显示器，QQ程序调用系统调用，QQ陷入内核态，操作硬件资源的代码都是内核态代码，也就是内核函数，产生软中断事件，告诉显示卡驱动程序，要在显示器什么位置现实QQ的登录框，显卡驱动程序收到指令和数据后，立即回应QQ进程，QQ进程让出CPU处于等待状态；显卡驱动继续协调显卡芯片、显卡内存处理显示数据，最终由显卡将QQ登录框显示在显示器上，同时产生硬中断事件，告诉QQ进程显示完成，QQ进程从等待状态恢复到就绪状态，等待用户的输入产生硬中断，再次被调入CPU执行功能。就可以愉快的聊天了。<br>还有好多不清楚的地方，不能提了，否则这节课我就没完没了了，明天Linux命令，每个公司都由几句黑话就没时间预习了。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-02 23:21:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/54/9a/76c0af70.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>每天晒白牙</span>
  </div>
  <div class="_2_QraFYR_0">四月一日【愚人节】立flag<br>这个专栏一定要认真跟下来</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-01 07:38:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/ad/45/f555c6dd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>an_time</span>
  </div>
  <div class="_2_QraFYR_0">这样的比喻很形象，把不易理解的东西形象化就很好理解了。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-01 00:57:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/dc/68/006ba72c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Untitled</span>
  </div>
  <div class="_2_QraFYR_0">Linux最新版本源代码：https:&#47;&#47;www.kernel.org&#47;<br>最新稳定内核版本时5.0.7，目录结构：<br>kernel 内核管理核心代码-进程管理子系统<br>mm 内存管理代码-内存管理子系统<br>drivers 驱动程序代码-设备管理子系统<br>fs 文件系统代码-文件管理子系统<br>net 网络协议代码-网络管理子系统<br>arch 体系结构代码，如x86、arm等<br>ipc 进程间通信子系统<br>init Linux系统启动初始化相关代码</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-07 15:44:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/63/84/f45c4af9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Vackine</span>
  </div>
  <div class="_2_QraFYR_0">刘老师好！那个内核体系结构图里面，为啥网络子系统不和系统调用直接有连线？（难道是因为在linux里面，访问网络接口该网络接口对应的是一个文件描述符，所以算作是被文件系统管理），然后在设备管理子系统里面，既和系统调用有连线又和文件系统有连线（linux里面一切皆是文件，所以设备对于的描述符也作为文件，那和系统调用里面有连线的是指特殊的设备需要通过系统调用才能访问？比如屏幕显示设备？）</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，socket先进入vfs层</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-01 22:07:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/1c/2e/93812642.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Amark</span>
  </div>
  <div class="_2_QraFYR_0">是不是也可以用医院这个系统解释操作系统，比如到医院后先去大厅，里面有各个系统，门诊，住院部，取药，....</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 理解正确，取药你不能自己取，需要通过系统调用</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-06 12:48:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/31/f4/467cf5d7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>MARK</span>
  </div>
  <div class="_2_QraFYR_0">学Linux同时学项目管理了，老师知道我早晚要转行么（：</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 哈哈</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-01 17:45:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/14/e1/ee5705a2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Zend</span>
  </div>
  <div class="_2_QraFYR_0">这个专栏一定要认真跟下来</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-01 09:42:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/6e/0f/d6773c7e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>浪子</span>
  </div>
  <div class="_2_QraFYR_0">帮老师贴上链接^_^<br><br>https:&#47;&#47;www.kernel.org&#47;</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-01 15:40:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/64/86/f5a9403a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>多襄丸</span>
  </div>
  <div class="_2_QraFYR_0">老师， 那我们安装的操作系统本身，算不算文件系统？ <br><br>意思就是，它们也是一堆有组织有纪律的文件，只是组成了可以调度物理设备的文件系统安装在了磁盘上。<br><br>我又有一个新问题了，那电脑启动的时候，就是我们按下电源键的时候，系统是怎么运行起来的呢？ <br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 后面会讲启动</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-01 10:21:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/be/d0/7f37f35f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小武</span>
  </div>
  <div class="_2_QraFYR_0">为何同学们都是如此优秀，老师的课程和评论区都能学到知识</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 很多优秀的同学</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-08 19:14:05</div>
  </div>
</div>
</div>
</li>
</ul>