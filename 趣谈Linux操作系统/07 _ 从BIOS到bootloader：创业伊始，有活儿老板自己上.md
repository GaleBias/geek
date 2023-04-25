<audio title="07 _ 从BIOS到bootloader：创业伊始，有活儿老板自己上" src="https://static001.geekbang.org/resource/audio/75/1c/759af1740f3fa587eab1cf0211182c1c.mp3" controls="controls"></audio> 
<p>有了开放的营商环境，咱们外包公司的创业之旅就要开始了。</p><p>上一节我们说，x86作为一个开放的营商环境，有两种模式，一种模式是实模式，只能寻址1M，每个段最多64K。这个太小了，相当于咱们创业的个体户模式。有了项目只能老板自己上，本小利微，万事开头难。另一种是保护模式，对于32位系统，能够寻址4G。这就是大买卖了，老板要雇佣很多人接项目。</p><p>几乎所有成功的公司，都是从个体户模式发展壮大的，因此，这一节咱们就从系统刚刚启动的个体户模式开始说起。</p><h2>BIOS时期</h2><p>当你轻轻按下计算机的启动按钮时，你的主板就加上电了。</p><p>按照我们之前说的，这时候你的CPU应该开始执行指令了。你作为老板，同时也作为员工，要开始干活了。可是你发现，这个时候还没有项目执行计划书，所以你没啥可干的。</p><p>也就是说，这个时候没有操作系统，内存也是空的，一穷二白。CPU该怎么办呢？</p><p>你作为这个创业公司的老板，由于原来没开过公司，对于公司的运营当然是一脸懵的。但是我们有一个良好的营商环境，其中的创业指导中心早就考虑到这种情况了。于是，创业指导中心就给了你一套创业公司启动指导手册。你只要按着指导手册来干就行了。</p><p><img src="https://static001.geekbang.org/resource/image/a4/6a/a4009d3de2dbae10340256af2737c26a.jpeg?wh=2113*1162" alt=""></p><p>计算机系统也早有计划。在主板上，有一个东西叫<strong>ROM</strong>（Read Only Memory，只读存储器）。这和咱们平常说的内存<strong>RAM</strong>（Random Access Memory，随机存取存储器）不同。</p><!-- [[[read_end]]] --><p>咱们平时买的内存条是可读可写的，这样才能保存计算结果。而ROM是只读的，上面早就固化了一些初始化的程序，也就是<strong>BIOS</strong>（Basic Input and Output System，基本输入输出系统）。</p><p>如果你自己安装过操作系统，刚启动的时候，按某个组合键，显示器会弹出一个蓝色的界面。能够调整启动顺序的系统，就是我说的BIOS，然后我们就可以先执行它。</p><p><img src="https://static001.geekbang.org/resource/image/13/b7/13187b1ffe878bc406da53967e8cddb7.png?wh=640*453" alt=""></p><p>创业初期，你的办公室肯定很小。假如现在你有1M的内存地址空间。这个空间非常有限，你需要好好利用才行。</p><p><img src="https://static001.geekbang.org/resource/image/5f/fc/5f364ef5c9d1a3b1d9bb7153bd166bfc.jpeg?wh=1796*1148" alt=""></p><p>在x86系统中，将1M空间最上面的0xF0000到0xFFFFF这64K映射给ROM，也就是说，到这部分地址访问的时候，会访问ROM。</p><p>当电脑刚加电的时候，会做一些重置的工作，将CS设置为0xFFFF，将IP设置为0x0000，所以第一条指令就会指向0xFFFF0，正是在ROM的范围内。在这里，有一个JMP命令会跳到ROM中做初始化工作的代码，于是，BIOS开始进行初始化的工作。</p><p>创业指导手册第一条，BIOS要检查一下系统的硬件是不是都好着呢。</p><p>创业指导手册第二条，要有个办事大厅，只不过自己就是办事员。这个时期你能提供的服务很简单，但也会有零星的客户来提要求。</p><p>这个时候，要建立一个中断向量表和中断服务程序，因为现在你还要用键盘和鼠标，这些都要通过中断进行的。</p><p>这个时期也要给客户输出一些结果，因为需要你自己来，所以你还要充当客户对接人。你做了什么工作，做到了什么程度，都要主动显示给客户，也就是在内存空间映射显存的空间，在显示器上显示一些字符。</p><p><img src="https://static001.geekbang.org/resource/image/29/63/2900bed28c7345e6c90437da8a5cd563.jpeg?wh=1949*1316" alt=""></p><p>最后，政府领进门，创业靠个人。接下来就是你发挥聪明才智的时候了。</p><h2>bootloader时期</h2><p>政府给的创业指导手册只能保证你把公司成立起来，但是公司如何做大做强，需要你自己有一套经营方法。你可以试着从档案库里面翻翻，看哪里能够找到《企业经营宝典》。通过这个宝典，可以帮你建立一套完整的档案库管理体系，使得任何项目的档案查询都十分方便。</p><p>现在，什么线索都没有的BIOS，做完自己的事情，只能从档案库门卫开始，慢慢打听操作系统的下落。</p><p>操作系统在哪儿呢？一般都会在安装在硬盘上，在BIOS的界面上。你会看到一个启动盘的选项。启动盘有什么特点呢？它一般在第一个扇区，占512字节，而且以0xAA55结束。这是一个约定，当满足这个条件的时候，就说明这是一个启动盘，在512字节以内会启动相关的代码。</p><p>这些代码是谁放在这里的呢？在Linux里面有一个工具，叫<strong>Grub2</strong>，全称Grand Unified Bootloader Version 2。顾名思义，就是搞系统启动的。</p><p>你可以通过grub2-mkconfig -o /boot/grub2/grub.cfg来配置系统启动的选项。你可以看到里面有类似这样的配置。</p><pre><code>menuentry 'CentOS Linux (3.10.0-862.el7.x86_64) 7 (Core)' --class centos --class gnu-linux --class gnu --class os --unrestricted $menuentry_id_option 'gnulinux-3.10.0-862.el7.x86_64-advanced-b1aceb95-6b9e-464a-a589-bed66220ebee' {
	load_video
	set gfxpayload=keep
	insmod gzio
	insmod part_msdos
	insmod ext2
	set root='hd0,msdos1'
	if [ x$feature_platform_search_hint = xy ]; then
	  search --no-floppy --fs-uuid --set=root --hint='hd0,msdos1'  b1aceb95-6b9e-464a-a589-bed66220ebee
	else
	  search --no-floppy --fs-uuid --set=root b1aceb95-6b9e-464a-a589-bed66220ebee
	fi
	linux16 /boot/vmlinuz-3.10.0-862.el7.x86_64 root=UUID=b1aceb95-6b9e-464a-a589-bed66220ebee ro console=tty0 console=ttyS0,115200 crashkernel=auto net.ifnames=0 biosdevname=0 rhgb quiet 
	initrd16 /boot/initramfs-3.10.0-862.el7.x86_64.img
}
</code></pre><p>这里面的选项会在系统启动的时候，成为一个列表，让你选择从哪个系统启动。最终显示出来的结果就是下面这张图。至于上面选项的具体意思，我们后面再说。</p><p><img src="https://static001.geekbang.org/resource/image/88/97/883f3f5d4227a593228e1bcb93f67297.png?wh=716*245" alt=""></p><p>使用grub2-install /dev/sda，可以将启动程序安装到相应的位置。</p><p>grub2第一个要安装的就是boot.img。它由boot.S编译而成，一共512字节，正式安装到启动盘的第一个扇区。这个扇区通常称为<strong>MBR</strong>（Master Boot Record，主引导记录/扇区）。</p><p>BIOS完成任务后，会将boot.img从硬盘加载到内存中的0x7c00来运行。</p><p>由于512个字节实在有限，boot.img做不了太多的事情。它能做的最重要的一个事情就是加载grub2的另一个镜像core.img。</p><p>引导扇区就是你找到的门卫，虽然他看着档案库的大门，但是知道的事情很少。他不知道你的宝典在哪里，但是，他知道应该问谁。门卫说，档案库入口处有个管理处，然后把你领到门口。</p><p>core.img就是管理处，它们知道的和能做的事情就多了一些。core.img由lzma_decompress.img、diskboot.img、kernel.img和一系列的模块组成，功能比较丰富，能做很多事情。</p><p><img src="https://static001.geekbang.org/resource/image/2b/6a/2b8573bbbf31fc0cb0420e32d07b196a.jpeg?wh=2489*1520" alt=""></p><p>boot.img先加载的是core.img的第一个扇区。如果从硬盘启动的话，这个扇区里面是diskboot.img，对应的代码是diskboot.S。</p><p>boot.img将控制权交给diskboot.img后，diskboot.img的任务就是将core.img的其他部分加载进来，先是解压缩程序lzma_decompress.img，再往下是kernel.img，最后是各个模块module对应的映像。这里需要注意，它不是Linux的内核，而是grub的内核。</p><p>lzma_decompress.img对应的代码是startup_raw.S，本来kernel.img是压缩过的，现在执行的时候，需要解压缩。</p><p>在这之前，我们所有遇到过的程序都非常非常小，完全可以在实模式下运行，但是随着我们加载的东西越来越大，实模式这1M的地址空间实在放不下了，所以在真正的解压缩之前，lzma_decompress.img做了一个重要的决定，就是调用real_to_prot，切换到保护模式，这样就能在更大的寻址空间里面，加载更多的东西。</p><h2>从实模式切换到保护模式</h2><p>好了，管理处听说你要找宝典，知道你将来是要做老板的人。既然是老板，早晚都要雇人干活的。这不是个体户小打小闹，所以，你需要切换到老板角色，进入保护模式了，把哪些是你的权限，哪些是你可以授权给别人的，都分得清清楚楚。</p><p>切换到保护模式要干很多工作，大部分工作都与内存的访问方式有关。</p><p>第一项是<strong>启用分段</strong>，就是在内存里面建立段描述符表，将寄存器里面的段寄存器变成段选择子，指向某个段描述符，这样就能实现不同进程的切换了。第二项是<strong>启动分页</strong>。能够管理的内存变大了，就需要将内存分成相等大小的块，这些我们放到内存那一节详细再讲。</p><p>切换到了老板角色，也是为了招聘很多人，同时接多个项目，这时候就需要划清界限，懂得集权与授权。</p><p>当了老板，眼界要宽多了，同理保护模式需要做一项工作，那就是打开Gate A20，也就是第21根地址线的控制线。在实模式8086下面，一共就20个地址线，可访问1M的地址空间。如果超过了这个限度怎么办呢？当然是绕回来了。在保护模式下，第21根要起作用了，于是我们就需要打开Gate A20。</p><p>切换保护模式的函数DATA32 call real_to_prot会打开Gate A20，也就是第21根地址线的控制线。</p><p>现在好了，有的是空间了。接下来我们要对压缩过的kernel.img进行解压缩，然后跳转到kernel.img开始运行。</p><p>切换到了老板角色，你可以正大光明地进入档案馆，寻找你的那本宝典。</p><p>kernel.img对应的代码是startup.S以及一堆c文件，在startup.S中会调用grub_main，这是grub kernel的主函数。</p><p>在这个函数里面，grub_load_config()开始解析，我们上面写的那个grub.conf文件里的配置信息。</p><p>如果是正常启动，grub_main最后会调用grub_command_execute (“normal”, 0, 0)，最终会调用grub_normal_execute()函数。在这个函数里面，grub_show_menu()会显示出让你选择的那个操作系统的列表。</p><p>同理，作为老板，你发现这类的宝典不止一本，经营企业的方式也有很多种，到底是人性化的，还是强纪律的，这个时候你要做一个选择。</p><p>一旦，你选定了某个宝典，启动某个操作系统，就要开始调用 grub_menu_execute_entry() ，开始解析并执行你选择的那一项。接下来你的经营企业之路就此打开了。</p><p>例如里面的linux16命令，表示装载指定的内核文件，并传递内核启动参数。于是grub_cmd_linux()函数会被调用，它会首先读取Linux内核镜像头部的一些数据结构，放到内存中的数据结构来，进行检查。如果检查通过，则会读取整个Linux内核镜像到内存。</p><p>如果配置文件里面还有initrd命令，用于为即将启动的内核传递init ramdisk路径。于是grub_cmd_initrd()函数会被调用，将initramfs加载到内存中来。</p><p>当这些事情做完之后，grub_command_execute (“boot”, 0, 0)才开始真正地启动内核。</p><h2>总结时刻</h2><p>启动的过程比较复杂，我这里画一个图，让你比较形象地理解这个过程。你可以根据我讲的，自己来梳理一遍这个过程，做到不管是从流程还是细节上，都能心中有数。</p><p><img src="https://static001.geekbang.org/resource/image/0a/6b/0a29c1d3e1a53b2523d2dcab3a59886b.jpeg?wh=1819*4309" alt=""></p><h2>课堂练习</h2><p>grub2是一个非常牛的Linux启动管理器，请你研究一下grub2的命令和配置，并试试通过它启动Ubuntu和centOS两个操作系统。</p><p>欢迎留言和我分享你的疑惑和见解，也欢迎你收藏本节内容，反复研读。你也可以把今天的内容分享给你的朋友，和他一起学习、进步。</p><p><img src="https://static001.geekbang.org/resource/image/8c/37/8c0a95fa07a8b9a1abfd394479bdd637.jpg?wh=1110*659" alt=""></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/74/c9/d3439ca4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>why</span>
  </div>
  <div class="_2_QraFYR_0">- 实模式只有 1MB 内存寻址空间(X86)<br>- 加电, 重置 CS 为 0xFFFF , IP 为 0x0000, 对应 BIOS 程序<br>- 0xF0000-0xFFFFF 映射到 BIOS 程序(存储在ROM中), BIOS 做以下三件事:<br>    - 检查硬件<br>    - 提供基本输入(中断)输出(显存映射)服务<br>    - 加载 MBR 到内存(0x7c00)<br>- MRB: 启动盘第一个扇区(512B, 由 Grub2 写入 boot.img 镜像)<br>- boot.img 加载 Grub2 的 core.img 镜像<br>- core.img 包括 diskroot.img, lzma_decompress.img, kernel.img 以及其他模块<br>- boot.img 先加载运行 diskroot.img, 再由 diskroot.img 加载 core.img 的其他内容<br>- diskroot.img 解压运行 lzma_compress.img, 由lzma_compress.img 切换到保护模式<br><br>-----------<br><br>- 切换到保护模式需要做以下三件事:<br>    - 启用分段, 辅助进程管理<br>    - 启动分页, 辅助内存管理<br>    - 打开其他地址线<br>- lzma_compress.img 解压运行 grub 内核 kernel.img, kernel.img 做以下四件事:<br>    - 解析 grub.conf 文件<br>    - 选择操作系统<br>    - 例如选择 linux16, 会先读取内核头部数据进行检查, 检查通过后加载完整系统内核<br>    - 启动系统内核</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-10 11:13:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/67/0e/c77ad9b1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>eason2017</span>
  </div>
  <div class="_2_QraFYR_0">而且我就觉得您的这个企业的例子就是个干扰项，实在是干扰学习知识，来回的切换思路！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-02 23:52:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/8e/10/10092bb1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Luke</span>
  </div>
  <div class="_2_QraFYR_0">这部分的实验，大家可以去github看我的工程哈，icecoobe&#47;oslab，已经进入保护模式了，还有很远的路，一起加油！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 牛</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-11 00:00:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/8e/10/10092bb1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Luke</span>
  </div>
  <div class="_2_QraFYR_0">看到很多人留言需要资料，我来推荐一本新书《一个64位操作系统的设计与实现》，如果你有汇编基础，很感兴趣底层的细节，可以看李忠的那本《从实模式到保护模式》</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞，看来我得收集一下书名，统一推荐给大家</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-11 00:11:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/54/3f/8e3b39f7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>zhangbing</span>
  </div>
  <div class="_2_QraFYR_0">看来从这篇开始我要看三遍四遍五遍的节奏了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 三遍就够，加油</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-10 09:50:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/5f/b7/c474f406.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Socrakit</span>
  </div>
  <div class="_2_QraFYR_0">查了一些资料，关于 Gate A20 我的理解是：<br><br>- 8086 地址线20根 -&gt; 可用内存 0 ~ FFFFF<br>  寄存器却是16位，寻址模式为 segment(16位):offset(16位)， 最大范围变成 0FFFF0(左移了4位) + 0FFFF = 10FFEF<br>  后果是多出来了 100000 ~ 10FFEF （访问这些地址时会回绕到 0 ~ FFEF）<br><br>- 80286 开始地址线变多，寻址范围大大增大，但是又必须兼容旧程序，8086在访问 100000 ~ 10FFEF时会回绕，但是 80286 不会 ，因为有第21根线的存在，会访问到实际的 100000 ~ 10FFEF 地址的内存。<br>于是 Gate A20 开关就诞生了，它的作用是：<br><br>- 实模式下 （存在的唯一理由是为了兼容8086）：<br>  - 打开 -&gt;  寻址100000 ~ 10FFEF会真正访问<br>  - 关闭-&gt; 回绕到 0 ~ FFEF<br><br>- 保护模式下：<br>  - 打开 -&gt; 可连续访问内存<br>  - 关闭 -&gt; 只能访问到奇数的1M段，即 00000-FFFFF, 200000-2FFFFF,300000-3FFFFF…  <br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-16 13:20:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/a8/e2/f8e51df2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Li Shunduo</span>
  </div>
  <div class="_2_QraFYR_0">老板选择了《狼性文化》😂😂</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-10 08:41:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/9d/f4/d8260b2b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Memoria</span>
  </div>
  <div class="_2_QraFYR_0">大家有兴趣实践的话可以参考清华大学的操作系统实验课，里面第一个实验讲的就是启动的过程，可以让人理解的更加透彻。https:&#47;&#47;github.com&#47;chyyuu&#47;ucore_os_lab</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-20 20:40:50</div>
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
  <div class="_2_QraFYR_0">总结:ROM只读存储器，ROm固化了一些程序就是BIOS，用来初始化系统，一开始的内存空间比较小，只有1M，最上面的64k映射为BIOS，指针指向这64k，开始进行初始化，有2个事情，一个是检查硬件环境，另一个是建立中断程序和中断向量表，同时把结果显示在显示器上，BIOS只是做初始化工作，真正安装系统了，首先要找系统，grub2是搞系统启动的，他把系统代码放在硬盘上，一般在第一个扇区，以0xAA55结束，512个字节，满足这个条件，就是系统启动的代码，grub2要首先安装的是第一个扇区MBR主引导扇区，他在BIOS初始化完成之后进行，会讲boot.img加载到内存，他能做的另一个事是加载core.img镜像，boot.img先加载core.img 的第一个扇区，diskboot.img，将core.img的其他程序加载进来，然后diskboot.img解压lzma_decompress.img， 再解压kernel.img，再然后是各个模块对应的映像。lzma_decompress在解压之前，调用real_to_prot，切换到保护模式。切换到保护模式，做的事情，启用分段，在内存里建立段描述表，将段寄存器里的段寄存器变成段选择子，指向某个段描述符，就能完成进程的切换，启动分页，管理的内存大了，将内存分成大小相等的块，打开Gate20，第21根地址线的控制线，有空间了，对kernel.img解压缩，开始运行，是一堆.c文件，里面有主函数，显示出操作系统的列表，选择了一个操作系统，开始调用grub_menu_execute_entry()，开始执行选择的那一项，里面的linux16命令，表示装载指定的内核文件，并传递内核启动参数，于是grub_cmd_linux()函数被调用，首先会读取linux内核头部的数据结构，加载到内存中来，检查通过，会加载整个linux内核镜像到内存，当都做完，调用grub_command_execute(&quot;boot&quot;,0,0)，开始真正的启动内核。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-11 23:09:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/aa/81/97e36f32.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>影影影</span>
  </div>
  <div class="_2_QraFYR_0">补充阅读<br>https:&#47;&#47;opensource.com&#47;article&#47;17&#47;2&#47;linux-boot-and-startup<br>https:&#47;&#47;opensource.com&#47;article&#47;17&#47;3&#47;introduction-grub2-configuration-linux</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-10 14:28:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/0b/0a/fa152399.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>wahaha</span>
  </div>
  <div class="_2_QraFYR_0">grub2 是一个非常牛的 Linux 启动管理器<br>这句应该去掉Linux，因为GRUB2也能启动其它操作系统</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-12 14:05:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/79/3d/29746a4b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>赵又新</span>
  </div>
  <div class="_2_QraFYR_0">之前课上说的，如果没有理解错的话：<br>32位，分为16位寻址空间和16位偏移量。但通过左移4位的方式，将寻址空间扩充为20位。所以，0xFFFF的位置实际指的是0xFFFF0。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-11 08:09:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/88/fe/c18a85fe.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>随风</span>
  </div>
  <div class="_2_QraFYR_0">当电脑刚加电的时候，会做一些重置的工作，将 CS 设置为 0xFFFF,将 IP 设置为 0x0000,所以第一条指令就会指向 0xFFFF0。这个所以怎么得到的结果？为什么上面都是五位0xFFFFF, cs&#47;ip都是四位0xFFFF? 小白越看越不明白了。<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 左移4位+偏移量</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-10 18:32:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/3a/d7/454a1b90.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>leon</span>
  </div>
  <div class="_2_QraFYR_0">32位处理器不是有32根地址线嘛？为啥只打开第21根地址线的控制线？这里可以再稍微解释一下吗？控制线是另外一种线嘛？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 实模式下，只开20以下的，32才开20以上的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-14 15:50:01</div>
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
  <div class="_2_QraFYR_0">1.BIOS时期约定了启动扇区的位置与大小<br><br>2.BootLoader时期主要是通过grub2来启动系统<br><br>3.启动过程中有实模式到保护模式的切换。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-10 08:44:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/b2/08/92f42622.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>尔冬橙</span>
  </div>
  <div class="_2_QraFYR_0">说实话，现在计算机硕士没有几个懂这些的。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-31 01:10:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/35/64/1ad5530d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>马上想好</span>
  </div>
  <div class="_2_QraFYR_0">当电脑刚加电的时候，会做一些重置的工作，将 CS 设置为 0xFFFF，将 IP 设置为 0x0000，所以第一条指令就会指向 0xFFFF0，正是在 ROM 的范围内。 为什么第一条指令会指向0xFFFF0呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 左移四位</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-10 23:27:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/f2/2b/7d9751bb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>许山山</span>
  </div>
  <div class="_2_QraFYR_0">老师可不可以建个 GitHub 仓库，这样大家就可以在 issue 讨论交流了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-12 09:04:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/df/ca/7c223fce.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>天使也有爱</span>
  </div>
  <div class="_2_QraFYR_0">老师 我现在看这些内容有点晕 太细了 我是要用那本书做配套看 还是直接用内核源码结合着看呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我推荐了书籍，对着源码看挺好的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-10 16:21:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJAl7V1ibk8gX9qWZKLlCmKAl6nicoTZ03PWksrUbItVraTGk5zpne1BEtUam8w8VID4EzcyyhC1LAw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>aingwm</span>
  </div>
  <div class="_2_QraFYR_0">“将 CS 设置为 0xFFFF，将 IP 设置为 0x0000”是以往的做法，Intel没有再延续，新的做法是“将 CS 设置为 0xF000，将 IP 设置为 0xFFF0”，当然，CS:IP的指向仍然是 0xFFFF0 ，这一点倒是没有变。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-08 17:04:36</div>
  </div>
</div>
</div>
</li>
</ul>