<audio title="难点解析｜eBPF多内核版本兼容详解" src="https://static001.geekbang.org/resource/audio/e6/00/e6f6d67b4e99699078de9b9aa67a2100.mp3" controls="controls"></audio> 
<p>你好，我是倪朋飞。</p><p>在上一讲中，我带你详细梳理了 eBPF 开发环境的配置方法，特别是 eBPF 相关开发软件包的安装和升级方法，以及内核的配置和编译方法。熟悉了 eBPF 的开发环境和内核编译之后，在留言区和微信群中我还看到很多同学依然在使用较旧版本的内核。而为了学习 eBPF，很多同学已经配置了一个非常新的内核版本作为开发环境，但却发现在新内核中开发的 eBPF 程序有时却没法直接在旧版本的内核中运行。</p><p>今天，我就带你一起来看看如何让 eBPF 程序兼容新旧版本的内核，以便在新版本内核中使用诸如 CO-RE 等新特性的同时，还可以在旧版本内核中正常运行。</p><h2>为什么需要考虑 eBPF 程序的内核兼容性？</h2><p>在开始正式的内容之前，我想你肯定有一个问题，那就是什么时候需要考虑新旧内核版本的兼容性，以及为什么会出现新旧内核版本兼容性的问题。</p><p>在理想情况下，开发测试环境和生产环境应该都是一致的，包括使用相同的内核版本。如果你已经满足了这个条件，那么自然也就不需要考虑内核兼容的问题。但注意这只是理想情况，实际情况下内核版本不一致的问题是不可避免的，比如：</p><ul>
<li>为了获取更好的稳定性和社区支持，内核版本（甚至是 Linux&nbsp;发行版版本）需要持续跟随上游社区进行升级；</li>
<li>为了采纳新技术，新的产品架构可能一开始就会采纳较新的内核，而使用旧内核的遗留系统还需要很长时间的迭代过程；</li>
<li>为了获得更广的用户，很多商业或开源项目不仅要支持最新的内核版本，还需要兼容各种各样的用户环境，而这些用户所使用的内核版本也是千差万别的。</li>
</ul><!-- [[[read_end]]] --><p>那么，在新旧版本的内核中运行 eBPF 程序时到底会碰到哪些问题呢？最典型的有以下这三个。</p><p><strong>第一，新内核引入的 eBPF 新特性无法在旧内核中运行。</strong>与绝大部分的技术产品类似，Linux 内核新版本中开发的特性绝大部分都不会迁移到旧版本的内核中，并且新特性的默认开启通常也需要一个比较长的过程（你可以在<a href="https://github.com/iovisor/bcc/blob/master/docs/kernel-versions.md">这里</a>找到不同内核版本支持的 eBPF 特性）。</p><p><strong>第二，新内核中的数据结构、函数签名以及跟踪点等有可能跟旧版本内核不同。</strong>比如，在我们课程<a href="https://time.geekbang.org/column/article/481090">第03讲</a>的跟踪案例中提到的 <code>openat2()</code> 系统调用是在内核 5.6 中才新增的，而旧版本内核需要换成 <code>openat()</code> 系统调用。</p><p><strong>第三，即便是相同的内核版本，不同的编译选项也可能会导致内核数据结构的不同。</strong>比如，<code>CONFIG_THREAD_INFO_IN_TASK</code> 的开关会直接影响内核 <a href="https://elixir.bootlin.com/linux/v5.13/source/include/linux/sched.h#L657">task_struct</a> 数据结构所有成员变量的偏移地址。</p><pre><code class="language-c++">  struct task_struct {
  #ifdef CONFIG_THREAD_INFO_IN_TASK
  	/*
  	 * For reasons of header soup (see current_thread_info()), this
  	 * must be the first element of task_struct.
  	 */
  	struct thread_info		thread_info;
  #endif
  	/* -1 unrunnable, 0 runnable, &gt;0 stopped: */
  	volatile long			state;
    ...
  }
</code></pre><p>由于这些兼容性问题都是由内核版本不同而导致的，所以我们很容易想到的一个笨方法就是给所有不兼容的内核版本分别开发不同的 eBPF 程序。但这种方法的缺点太明显了，维护大量功能重复的 eBPF 程序成本太高，所以我并不推荐你使用这种方法。</p><p>那么还有哪些更好的方法呢？接下来，我就带你一起来看看如何更好地解决内核版本不同带来的兼容性问题。</p><h2>BCC 是如何兼容多内核版本的？</h2><p>BCC 和 bpftrace 作为使用最广泛的 eBPF 项目，自然也最容易碰到内核兼容性的问题。那么，它们是怎么解决这些兼容性问题的呢？其实也很简单，主要就是下面两个方法：</p><ul>
<li>第一，在运行 eBPF 程序的时候使用当前系统安装的内核头文件进行就地编译，这样就可以确保 eBPF 程序中所引用的内核数据结构和函数签名等，跟运行中的内核是完全匹配的。</li>
<li>第二，在 eBPF 程序编译前事先探测内核支持的函数签名和数据结构，进而为 eBPF 程序生成适配当前内核的版本。比如，在块设备 I/O 延迟跟踪程序 <a href="https://github.com/iovisor/bcc/blob/master/tools/biolatency.py#L209">biolantecy</a> 中，BCC 借助库函数 <code>BPF.get_kprobe_functions()</code> 来判断内核是不是支持特定的探针函数，进而再根据结果去选择挂载点：</li>
</ul><pre><code class="language-python">  if BPF.get_kprobe_functions(b'__blk_account_io_start'):
    b.attach_kprobe(event="__blk_account_io_start", fn_name="trace_req_start")
  else:
    b.attach_kprobe(event="blk_account_io_start", fn_name="trace_req_start")
  
  if BPF.get_kprobe_functions(b'__blk_account_io_done'):
      b.attach_kprobe(event="__blk_account_io_done", fn_name="trace_req_done")
  else:
      b.attach_kprobe(event="blk_account_io_done", fn_name="trace_req_done")
</code></pre><p>当然了，BCC 采用的这些方法虽然解决了内核版本兼容的问题，但同时也存在很多的缺点，包括需要在所有目标机器安装开发工具和内核头文件、编译消耗额外资源、eBPF 程序启动慢以及编译时错误难以排查等。</p><p>那么，有没有更好的方法呢？答案是肯定的。Linux 内核维护者提供了一个更好的方案，那就是一次编译到处执行（Compile Once Run Everywhere，简称 CO-RE）。接下来，我们来看看 CO-RE 是如何解决这些问题的。</p><h2>一次编译到处执行（CO-RE）</h2><p>eBPF 的一次编译到处执行（简称 CO-RE）项目借助了 BPF 类型格式（BPF Type Format, 简称 BTF）提供的调试信息，再通过下面的四个步骤，使得 eBPF 程序可以适配不同版本的内核：</p><ul>
<li>第一，在 bpftool 工具中提供了从 BTF 生成头文件的工具，从而摆脱了对内核头文件的依赖。</li>
<li>第二，通过对 BPF 代码中的访问偏移量进行重写，解决了不同内核版本中数据结构偏移量不同的问题。</li>
<li>第三，在 libbpf 中预定义不同内核版本中数据结构的修改，解决了不同内核中数据结构不兼容的问题。</li>
<li>第四，在 libbpf 中提供一系列的内核特性探测库函数，解决了 eBPF 程序在不同内核内版本中需要执行不同行为的问题。比如，你可以用 <code>bpf_core_type_exists()</code> 和<code>bpf_core_field_exists()</code> 分别检查内核数据类型和成员变量是否存在，也可以用类似 <code>extern int LINUX_KERNEL_VERSION __kconfig</code> 的方式查询内核的配置选项。</li>
</ul><p>采用这些方法之后，CO-RE 就使得 eBPF 程序可以在开发环境编译完成之后，分发到不同版本内核的机器中运行，并且也不再需要目标机器安装各种开发工具和内核头文件。所以，<strong>Linux 内核社区更推荐所有开发者使用 CO-RE 和 libbpf 来构建 eBPF 程序。</strong>实际上，如果你看过 BCC 的源代码，你会发现 BCC 已经把很多工具都<a href="https://github.com/iovisor/bcc/tree/master/libbpf-tools">迁移</a>到了 CO-RE。</p><p>需要注意的是，CO-RE 需要比较新的内核版本（大于等于5.2）并且需要打开 <code>CONFIG_DEBUG_INFO_BTF</code> 配置选项。所以，实际上采用 CO-RE 技术的 eBPF 程序还是只能运行在满足这两个条件的内核版本中。那么，不支持 BTF 的内核怎么办呢？根据开源社区的实践经验，有两种不同的解决办法。</p><p>第一种，采用条件编译的方式，根据是否支持 CO-RE，生成两个不同的 eBPF 字节码文件。而到程序运行时，再根据内核是否支持 CO-RE 选择对应的字节码文件加载运行。</p><p>第二种，采用 Aqua Security 开源的 <a href="https://github.com/aquasecurity/btfhub-archive">btfhub</a> ，为目标机器匹配的内核版本下载独立的 BTF 信息库，最后再通过如下的方法借助 libbpf 进行加载：</p><pre><code class="language-c++">	struct bpf_object_open_opts openopts = {
		.sz = sizeof(struct bpf_object_open_opts),
         // 从BPF_CUSTOM_BTF环境变量读取BTF文件路径
		.btf_custom_path = getenv("BPF_CUSTOM_BTF"),
	};

	obj = hello_btf_bpf__open_opts(&amp;openopts);
	if (!obj) {
		fprintf(stderr, "failed to open and/or load BPF object\n");
		return 1;
	}
</code></pre><p>除此之外，eBPF 程序在运行时一般不需要内核的所有 BTF 信息，而只是访问其中的几个少数类型。因而，从内核 5.18 开始，bpftool 还新增了 <code>bpftool gen min_core_btf</code> 命令，帮你精简 BTF 信息，进一步减轻了 eBPF 程序（包括 BTF 信息）的分发交付。</p><p>以我们课程中的 <a href="https://github.com/feiskyer/ebpf-apps/blob/main/bpf-apps/execsnoop.bpf.c">execsnoop</a> 案例为例，精简后的 BTF 只有几百个字节：</p><pre><code class="language-bash">$ ls -lh /sys/kernel/btf/vmlinux
-r--r--r-- 1 root root 4.8M Jun 20 19:23 /sys/kernel/btf/vmlinux

$ bpftool gen min_core_btf /sys/kernel/btf/vmlinux execsnoop.btf execsnoop.bpf.o
$ ls -lh execsnoop.btf
-rw-r--r-- 1 root root 236 Jun 20 20:15 execsnoop.btf
</code></pre><h2>小结</h2><p>今天，我带你一起梳理了在开发和运行 eBPF 程序过程中可能会碰到的内核版本兼容性问题，并总结了解决这些问题的实践经验。由于不同版本内核的数据结构、函数签名、内核特性以及内核配置等都有可能不同，针对一个版本开发的 eBPF 程序有可能没法直接运行在其他不同版本的内核上。</p><p>为了解决这个问题，BCC 采用了运行时编译的方式，直接从目标机器的内核头文件中获取数据结构，但同时也导致了需要为所有目标机器安装开发工具和内核头文件、编译消耗额外资源、eBPF 程序启动慢以及编译时错误难以排查等额外的问题。</p><p>相比于 BCC，CO-RE 提供了一种更优雅的方式，不仅借助 BTF 解耦了内核头文件的依赖，避免了运行时编译，还通过 libbpf 提供了一系列的辅助函数，方便 eBPF 程序动态探测内核所支持的特性，进而有针对性地处理。对于不支持 BTF 的内核，libbpf 还支持加载自定义的 BTF 文件，所以多内核版本的分发也不再是一个问题。对于开篇中提到的，在新内核中开发的 eBPF 程序有时没法直接在旧版本内核中运行的问题，通过这个方式也就可以解决了。</p><p>今天这一讲就到这里了，下一次的动态更新预计会在8月份。如果你有对我们课程未来内容的建议，欢迎在评论区提出来，期待你与我一起完善和构建一个最贴近实践的 eBPF 知识体系。</p><h2>思考题</h2><p>最后，我想请你聊一聊这两个问题：</p><ol>
<li>通过前面一段时间课程的学习，你已经把 eBPF 应用到了哪些场景？又有哪些经验和收获？</li>
<li>在分发 eBPF 程序的过程中，你有没有碰到今天讲到的内核版本兼容性的问题？结合今天的内容，你是怎么解决这些问题的？<br>
欢迎在留言区和我讨论，也欢迎把这节课分享给你的同事、朋友。我们一起在实战中演练，在交流中进步。</li>
</ol>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/17/18/e4382a8e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>有识之士</span>
  </div>
  <div class="_2_QraFYR_0">微信群在哪里？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-24 11:42:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/b7/24/17f6c240.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>janey</span>
  </div>
  <div class="_2_QraFYR_0">这里的一次编译，跟JIT即时编译是一个意思吗？如果不是，那有什么区别？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，一个意思</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-05 08:23:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_5ada4a</span>
  </div>
  <div class="_2_QraFYR_0">第二，通过对 BPF 代码中的访问偏移量进行重写，解决了不同内核版本中数据结构偏移量不同的问题。第三，在 libbpf 中预定义不同内核版本中数据结构的修改，解决了不同内核中数据结构不兼容的问题。<br><br>老师，我不太理解第二点和第三点的区别。<br>“预定义不同内核版本中数据结构的修改”，就是为了实现对“访问偏移量进行重写”吧？这两点落到实处，应该就是一点？<br>ps vmlinux 已经提供了内核数据结构，为什么libbpf中还要“预定义不同内核版本中数据结构的修改”？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-18 15:22:14</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/0c/30/bb4bfe9d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lyonger</span>
  </div>
  <div class="_2_QraFYR_0">总结下来，好像对于4.9的内核支持，还是不是特别友好，线上要使用还是很麻烦。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-31 19:58:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/24/97/be102fb1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>James</span>
  </div>
  <div class="_2_QraFYR_0">Ubuntu 20.04，内核5.4.0，但是CONFIG_DEBUG_INFO_BTF没打开，怎么打开呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-27 16:24:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/23/13/bd/3e86e791.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>十七</span>
  </div>
  <div class="_2_QraFYR_0">兼容性让人头疼</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-27 08:36:55</div>
  </div>
</div>
</div>
</li>
</ul>