<audio title="58 _ cgroup技术：内部创业公司应该独立核算成本" src="https://static001.geekbang.org/resource/audio/71/d9/7187ee713149abd66837faa9173a30d9.mp3" controls="controls"></audio> 
<p>我们前面说了，容器实现封闭的环境主要靠两种技术，一种是“看起来是隔离”的技术Namespace，另一种是用起来是隔离的技术cgroup。</p><p>上一节我们讲了“看起来隔离“的技术Namespace，这一节我们就来看一下“用起来隔离“的技术cgroup。</p><p>cgroup全称是control group，顾名思义，它是用来做“控制”的。控制什么东西呢？当然是资源的使用了。那它都能控制哪些资源的使用呢？我们一起来看一看。</p><p>首先，cgroup定义了下面的一系列子系统，每个子系统用于控制某一类资源。</p><ul>
<li>CPU子系统，主要限制进程的CPU使用率。</li>
<li>cpuacct 子系统，可以统计 cgroup 中的进程的 CPU 使用报告。</li>
<li>cpuset 子系统，可以为 cgroup 中的进程分配单独的 CPU 节点或者内存节点。</li>
<li>memory 子系统，可以限制进程的 Memory 使用量。</li>
<li>blkio 子系统，可以限制进程的块设备 IO。</li>
<li>devices 子系统，可以控制进程能够访问某些设备。</li>
<li>net_cls 子系统，可以标记 cgroups 中进程的网络数据包，然后可以使用 tc 模块（traffic control）对数据包进行控制。</li>
<li>freezer 子系统，可以挂起或者恢复 cgroup 中的进程。</li>
</ul><!-- [[[read_end]]] --><p>这么多子系统，你可能要说了，那我们不用都掌握吧？没错，这里面最常用的是对于CPU和内存的控制，所以下面我们详细来说它。</p><p>在容器这一章的第一节，我们讲了，Docker有一些参数能够限制CPU和内存的使用，如果把它落地到cgroup里面会如何限制呢？</p><p>为了验证Docker的参数与cgroup的映射关系，我们运行一个命令特殊的docker run命令，这个命令比较长，里面的参数都会映射为cgroup的某项配置，然后我们运行docker ps，可以看到，这个容器的id为3dc0601189dd。</p><pre><code>docker run -d --cpu-shares 513 --cpus 2 --cpuset-cpus 1,3 --memory 1024M --memory-swap 1234M --memory-swappiness 7 -p 8081:80 testnginx:1

# docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED              STATUS              PORTS                  NAMES
3dc0601189dd        testnginx:1         &quot;/bin/sh -c 'nginx -…&quot;   About a minute ago   Up About a minute   0.0.0.0:8081-&gt;80/tcp   boring_cohen
</code></pre><p>在Linux上，为了操作cgroup，有一个专门的cgroup文件系统，我们运行mount命令可以查看。</p><pre><code># mount -t cgroup
cgroup on /sys/fs/cgroup/systemd type cgroup (rw,nosuid,nodev,noexec,relatime,xattr,release_agent=/usr/lib/systemd/systemd-cgroups-agent,name=systemd)
cgroup on /sys/fs/cgroup/net_cls,net_prio type cgroup (rw,nosuid,nodev,noexec,relatime,net_prio,net_cls)
cgroup on /sys/fs/cgroup/perf_event type cgroup (rw,nosuid,nodev,noexec,relatime,perf_event)
cgroup on /sys/fs/cgroup/devices type cgroup (rw,nosuid,nodev,noexec,relatime,devices)
cgroup on /sys/fs/cgroup/blkio type cgroup (rw,nosuid,nodev,noexec,relatime,blkio)
cgroup on /sys/fs/cgroup/cpu,cpuacct type cgroup (rw,nosuid,nodev,noexec,relatime,cpuacct,cpu)
cgroup on /sys/fs/cgroup/memory type cgroup (rw,nosuid,nodev,noexec,relatime,memory)
cgroup on /sys/fs/cgroup/cpuset type cgroup (rw,nosuid,nodev,noexec,relatime,cpuset)
cgroup on /sys/fs/cgroup/hugetlb type cgroup (rw,nosuid,nodev,noexec,relatime,hugetlb)
cgroup on /sys/fs/cgroup/freezer type cgroup (rw,nosuid,nodev,noexec,relatime,freezer)
cgroup on /sys/fs/cgroup/pids type cgroup (rw,nosuid,nodev,noexec,relatime,pids)
</code></pre><p>cgroup文件系统多挂载到/sys/fs/cgroup下，通过上面的命令行，我们可以看到我们可以用cgroup控制哪些资源。</p><p>对于CPU的控制，我在这一章的第一节讲过，Docker可以控制cpu-shares、cpus和cpuset。</p><p>我们在/sys/fs/cgroup/下面能看到下面的目录结构。</p><pre><code>drwxr-xr-x 5 root root  0 May 30 17:00 blkio
lrwxrwxrwx 1 root root 11 May 30 17:00 cpu -&gt; cpu,cpuacct
lrwxrwxrwx 1 root root 11 May 30 17:00 cpuacct -&gt; cpu,cpuacct
drwxr-xr-x 5 root root  0 May 30 17:00 cpu,cpuacct
drwxr-xr-x 3 root root  0 May 30 17:00 cpuset
drwxr-xr-x 5 root root  0 May 30 17:00 devices
drwxr-xr-x 3 root root  0 May 30 17:00 freezer
drwxr-xr-x 3 root root  0 May 30 17:00 hugetlb
drwxr-xr-x 5 root root  0 May 30 17:00 memory
lrwxrwxrwx 1 root root 16 May 30 17:00 net_cls -&gt; net_cls,net_prio
drwxr-xr-x 3 root root  0 May 30 17:00 net_cls,net_prio
lrwxrwxrwx 1 root root 16 May 30 17:00 net_prio -&gt; net_cls,net_prio
drwxr-xr-x 3 root root  0 May 30 17:00 perf_event
drwxr-xr-x 5 root root  0 May 30 17:00 pids
drwxr-xr-x 5 root root  0 May 30 17:00 systemd
</code></pre><p>我们可以想象，CPU的资源控制的配置文件，应该在cpu,cpuacct这个文件夹下面。</p><pre><code># ls
cgroup.clone_children  cpu.cfs_period_us  notify_on_release
cgroup.event_control   cpu.cfs_quota_us   release_agent
cgroup.procs           cpu.rt_period_us   system.slice
cgroup.sane_behavior   cpu.rt_runtime_us  tasks
cpuacct.stat           cpu.shares         user.slice
cpuacct.usage          cpu.stat
cpuacct.usage_percpu   docker
</code></pre><p>果真，这下面是对CPU的相关控制，里面还有一个路径叫docker。我们进入这个路径。</p><pre><code>]# ls
cgroup.clone_children
cgroup.event_control
cgroup.procs
cpuacct.stat
cpuacct.usage
cpuacct.usage_percpu
cpu.cfs_period_us
cpu.cfs_quota_us
cpu.rt_period_us
cpu.rt_runtime_us
cpu.shares
cpu.stat
3dc0601189dd218898f31f9526a6cfae83913763a4da59f95ec789c6e030ecfd
notify_on_release
tasks
</code></pre><p>这里面有个很长的id，是我们创建的docker的id。</p><pre><code>[3dc0601189dd218898f31f9526a6cfae83913763a4da59f95ec789c6e030ecfd]# ls
cgroup.clone_children  cpuacct.usage_percpu  cpu.shares
cgroup.event_control   cpu.cfs_period_us     cpu.stat
cgroup.procs           cpu.cfs_quota_us      notify_on_release
cpuacct.stat           cpu.rt_period_us      tasks
cpuacct.usage          cpu.rt_runtime_us
</code></pre><p>在这里，我们能看到cpu.shares，还有一个重要的文件tasks。这里面是这个容器里所有进程的进程号，也即所有这些进程都被这些CPU策略控制。</p><pre><code>[3dc0601189dd218898f31f9526a6cfae83913763a4da59f95ec789c6e030ecfd]# cat tasks 
39487
39520
39526
39527
39528
39529
</code></pre><p>如果我们查看cpu.shares，里面就是我们设置的513。</p><pre><code>[3dc0601189dd218898f31f9526a6cfae83913763a4da59f95ec789c6e030ecfd]# cat cpu.shares
513
</code></pre><p>另外，我们还配置了cpus，这个值其实是由cpu.cfs_period_us和cpu.cfs_quota_us共同决定的。cpu.cfs_period_us是运行周期，cpu.cfs_quota_us是在周期内这些进程占用多少时间。我们设置了cpus为2，代表的意思是，在周期100000微秒的运行周期内，这些进程要占用200000微秒的时间，也即需要两个CPU同时运行一个整的周期。</p><pre><code>[3dc0601189dd218898f31f9526a6cfae83913763a4da59f95ec789c6e030ecfd]# cat cpu.cfs_period_us
100000
[3dc0601189dd218898f31f9526a6cfae83913763a4da59f95ec789c6e030ecfd]# cat cpu.cfs_quota_us
200000
</code></pre><p>对于cpuset，也即CPU绑核的参数，在另外一个文件夹里面/sys/fs/cgroup/cpuset，这里面同样有一个docker文件夹，下面同样有docker id 也即3dc0601189dd218898f31f9526a6cfae83913763a4da59f95ec789c6e030ecfd文件夹，这里面的cpuset.cpus就是配置的绑定到1、3两个核。</p><pre><code>[3dc0601189dd218898f31f9526a6cfae83913763a4da59f95ec789c6e030ecfd]# cat cpuset.cpus
1,3
</code></pre><p>这一章的第一节我们还讲了Docker可以限制内存的使用量，例如memory、memory-swap、memory-swappiness。这些在哪里控制呢？</p><p>/sys/fs/cgroup/下面还有一个memory路径，控制策略就是在这里面定义的。</p><pre><code>[root@deployer memory]# ls
cgroup.clone_children               memory.memsw.failcnt
cgroup.event_control                memory.memsw.limit_in_bytes
cgroup.procs                        memory.memsw.max_usage_in_bytes
cgroup.sane_behavior                memory.memsw.usage_in_bytes
docker                              memory.move_charge_at_immigrate
memory.failcnt                      memory.numa_stat
memory.force_empty                  memory.oom_control
memory.kmem.failcnt                 memory.pressure_level
memory.kmem.limit_in_bytes          memory.soft_limit_in_bytes
memory.kmem.max_usage_in_bytes      memory.stat
memory.kmem.slabinfo                memory.swappiness
memory.kmem.tcp.failcnt             memory.usage_in_bytes
memory.kmem.tcp.limit_in_bytes      memory.use_hierarchy
memory.kmem.tcp.max_usage_in_bytes  notify_on_release
memory.kmem.tcp.usage_in_bytes      release_agent
memory.kmem.usage_in_bytes          system.slice
memory.limit_in_bytes               tasks
memory.max_usage_in_bytes           user.slice
</code></pre><p>这里面全是对于memory的控制参数，在这里面我们可看到了docker，里面还有容器的id作为文件夹。</p><pre><code>[docker]# ls
3dc0601189dd218898f31f9526a6cfae83913763a4da59f95ec789c6e030ecfd
cgroup.clone_children
cgroup.event_control
cgroup.procs
memory.failcnt
memory.force_empty
memory.kmem.failcnt
memory.kmem.limit_in_bytes
memory.kmem.max_usage_in_bytes
memory.kmem.slabinfo
memory.kmem.tcp.failcnt
memory.kmem.tcp.limit_in_bytes
memory.kmem.tcp.max_usage_in_bytes
memory.kmem.tcp.usage_in_bytes
memory.kmem.usage_in_bytes
memory.limit_in_bytes
memory.max_usage_in_bytes
memory.memsw.failcnt
memory.memsw.limit_in_bytes
memory.memsw.max_usage_in_bytes
memory.memsw.usage_in_bytes
memory.move_charge_at_immigrate
memory.numa_stat
memory.oom_control
memory.pressure_level
memory.soft_limit_in_bytes
memory.stat
memory.swappiness
memory.usage_in_bytes
memory.use_hierarchy
notify_on_release
tasks

[3dc0601189dd218898f31f9526a6cfae83913763a4da59f95ec789c6e030ecfd]# ls
cgroup.clone_children               memory.memsw.failcnt
cgroup.event_control                memory.memsw.limit_in_bytes
cgroup.procs                        memory.memsw.max_usage_in_bytes
memory.failcnt                      memory.memsw.usage_in_bytes
memory.force_empty                  memory.move_charge_at_immigrate
memory.kmem.failcnt                 memory.numa_stat
memory.kmem.limit_in_bytes          memory.oom_control
memory.kmem.max_usage_in_bytes      memory.pressure_level
memory.kmem.slabinfo                memory.soft_limit_in_bytes
memory.kmem.tcp.failcnt             memory.stat
memory.kmem.tcp.limit_in_bytes      memory.swappiness
memory.kmem.tcp.max_usage_in_bytes  memory.usage_in_bytes
memory.kmem.tcp.usage_in_bytes      memory.use_hierarchy
memory.kmem.usage_in_bytes          notify_on_release
memory.limit_in_bytes               tasks
memory.max_usage_in_bytes
</code></pre><p>在docker id的文件夹下面，有一个memory.limit_in_bytes，里面配置的就是memory。</p><pre><code>[3dc0601189dd218898f31f9526a6cfae83913763a4da59f95ec789c6e030ecfd]# cat memory.limit_in_bytes
1073741824
</code></pre><p>还有memory.swappiness，里面配置的就是memory-swappiness。</p><pre><code>[3dc0601189dd218898f31f9526a6cfae83913763a4da59f95ec789c6e030ecfd]# cat memory.swappiness
7
</code></pre><p>还有就是memory.memsw.limit_in_bytes，里面配置的是memory-swap。</p><pre><code>[3dc0601189dd218898f31f9526a6cfae83913763a4da59f95ec789c6e030ecfd]# cat memory.memsw.limit_in_bytes
1293942784
</code></pre><p>我们还可以看一下tasks文件的内容，tasks里面是容器里面所有进程的进程号。</p><pre><code>[3dc0601189dd218898f31f9526a6cfae83913763a4da59f95ec789c6e030ecfd]# cat tasks 
39487
39520
39526
39527
39528
39529
</code></pre><p>至此，我们看到了cgroup对于Docker资源的控制，在用户态是如何表现的。我画了一张图总结一下。</p><p><img src="https://static001.geekbang.org/resource/image/1c/0f/1c762a6283429ff3587a7fc370fc090f.png?wh=3106*1366" alt=""></p><p>在内核中，cgroup是如何实现的呢？</p><p>首先，在系统初始化的时候，cgroup也会进行初始化，在start_kernel中，cgroup_init_early和cgroup_init都会进行初始化。</p><pre><code>asmlinkage __visible void __init start_kernel(void)
{
......
  cgroup_init_early();
......
  cgroup_init();
......
}
</code></pre><p>在cgroup_init_early和cgroup_init中，会有下面的循环。</p><pre><code>for_each_subsys(ss, i) {
	ss-&gt;id = i;
	ss-&gt;name = cgroup_subsys_name[i];
......
	cgroup_init_subsys(ss, true);
}

#define for_each_subsys(ss, ssid)					\
	for ((ssid) = 0; (ssid) &lt; CGROUP_SUBSYS_COUNT &amp;&amp;		\
	     (((ss) = cgroup_subsys[ssid]) || true); (ssid)++)
</code></pre><p>for_each_subsys会在cgroup_subsys数组中进行循环。这个cgroup_subsys数组是如何形成的呢？</p><pre><code>#define SUBSYS(_x) [_x ## _cgrp_id] = &amp;_x ## _cgrp_subsys,
struct cgroup_subsys *cgroup_subsys[] = {
#include &lt;linux/cgroup_subsys.h&gt;
};
#undef SUBSYS
</code></pre><p>SUBSYS这个宏定义了这个cgroup_subsys数组，数组中的项定义在cgroup_subsys.h头文件中。例如，对于CPU和内存有下面的定义。</p><pre><code>//cgroup_subsys.h

#if IS_ENABLED(CONFIG_CPUSETS)
SUBSYS(cpuset)
#endif

#if IS_ENABLED(CONFIG_CGROUP_SCHED)
SUBSYS(cpu)
#endif

#if IS_ENABLED(CONFIG_CGROUP_CPUACCT)
SUBSYS(cpuacct)
#endif

#if IS_ENABLED(CONFIG_MEMCG)
SUBSYS(memory)
#endif
</code></pre><p>根据SUBSYS的定义，SUBSYS(cpu)其实是[cpu_cgrp_id] = &amp;cpu_cgrp_subsys，而SUBSYS(memory)其实是[memory_cgrp_id] = &amp;memory_cgrp_subsys。</p><p>我们能够找到cpu_cgrp_subsys和memory_cgrp_subsys的定义。</p><pre><code>cpuset_cgrp_subsys
struct cgroup_subsys cpuset_cgrp_subsys = {
	.css_alloc	= cpuset_css_alloc,
	.css_online	= cpuset_css_online,
	.css_offline	= cpuset_css_offline,
	.css_free	= cpuset_css_free,
	.can_attach	= cpuset_can_attach,
	.cancel_attach	= cpuset_cancel_attach,
	.attach		= cpuset_attach,
	.post_attach	= cpuset_post_attach,
	.bind		= cpuset_bind,
	.fork		= cpuset_fork,
	.legacy_cftypes	= files,
	.early_init	= true,
};

cpu_cgrp_subsys
struct cgroup_subsys cpu_cgrp_subsys = {
	.css_alloc	= cpu_cgroup_css_alloc,
	.css_online	= cpu_cgroup_css_online,
	.css_released	= cpu_cgroup_css_released,
	.css_free	= cpu_cgroup_css_free,
	.fork		= cpu_cgroup_fork,
	.can_attach	= cpu_cgroup_can_attach,
	.attach		= cpu_cgroup_attach,
	.legacy_cftypes	= cpu_files,
	.early_init	= true,
};

memory_cgrp_subsys
struct cgroup_subsys memory_cgrp_subsys = {
	.css_alloc = mem_cgroup_css_alloc,
	.css_online = mem_cgroup_css_online,
	.css_offline = mem_cgroup_css_offline,
	.css_released = mem_cgroup_css_released,
	.css_free = mem_cgroup_css_free,
	.css_reset = mem_cgroup_css_reset,
	.can_attach = mem_cgroup_can_attach,
	.cancel_attach = mem_cgroup_cancel_attach,
	.post_attach = mem_cgroup_move_task,
	.bind = mem_cgroup_bind,
	.dfl_cftypes = memory_files,
	.legacy_cftypes = mem_cgroup_legacy_files,
	.early_init = 0,
};
</code></pre><p>在for_each_subsys的循环里面，cgroup_subsys[]数组中的每一个cgroup_subsys，都会调用cgroup_init_subsys，对于cgroup_subsys对于初始化。</p><pre><code>static void __init cgroup_init_subsys(struct cgroup_subsys *ss, bool early)
{
	struct cgroup_subsys_state *css;
......
	idr_init(&amp;ss-&gt;css_idr);
	INIT_LIST_HEAD(&amp;ss-&gt;cfts);

	/* Create the root cgroup state for this subsystem */
	ss-&gt;root = &amp;cgrp_dfl_root;
	css = ss-&gt;css_alloc(cgroup_css(&amp;cgrp_dfl_root.cgrp, ss));
......
	init_and_link_css(css, ss, &amp;cgrp_dfl_root.cgrp);
......
	css-&gt;id = cgroup_idr_alloc(&amp;ss-&gt;css_idr, css, 1, 2, GFP_KERNEL);
	init_css_set.subsys[ss-&gt;id] = css;
......
	BUG_ON(online_css(css));
......
}
</code></pre><p>cgroup_init_subsys里面会做两件事情，一个是调用cgroup_subsys的css_alloc函数创建一个cgroup_subsys_state；另外就是调用online_css，也即调用cgroup_subsys的css_online函数，激活这个cgroup。</p><p>对于CPU来讲，css_alloc函数就是cpu_cgroup_css_alloc。这里面会调用 sched_create_group创建一个struct task_group。在这个结构中，第一项就是cgroup_subsys_state，也就是说，task_group是cgroup_subsys_state的一个扩展，最终返回的是指向cgroup_subsys_state结构的指针，可以通过强制类型转换变为task_group。</p><pre><code>struct task_group {
	struct cgroup_subsys_state css;

#ifdef CONFIG_FAIR_GROUP_SCHED
	/* schedulable entities of this group on each cpu */
	struct sched_entity **se;
	/* runqueue &quot;owned&quot; by this group on each cpu */
	struct cfs_rq **cfs_rq;
	unsigned long shares;

#ifdef	CONFIG_SMP
	atomic_long_t load_avg ____cacheline_aligned;
#endif
#endif

	struct rcu_head rcu;
	struct list_head list;

	struct task_group *parent;
	struct list_head siblings;
	struct list_head children;

	struct cfs_bandwidth cfs_bandwidth;
};
</code></pre><p>在task_group结构中，有一个成员是sched_entity，前面我们讲进程调度的时候，遇到过它。它是调度的实体，也即这一个task_group也是一个调度实体。</p><p>接下来，online_css会被调用。对于CPU来讲，online_css调用的是cpu_cgroup_css_online。它会调用sched_online_group-&gt;online_fair_sched_group。</p><pre><code>void online_fair_sched_group(struct task_group *tg)
{
	struct sched_entity *se;
	struct rq *rq;
	int i;

	for_each_possible_cpu(i) {
		rq = cpu_rq(i);
		se = tg-&gt;se[i];
		update_rq_clock(rq);
		attach_entity_cfs_rq(se);
		sync_throttle(tg, i);
	}
}
</code></pre><p>在这里面，对于每一个CPU，取出每个CPU的运行队列rq，也取出task_group的sched_entity，然后通过attach_entity_cfs_rq将sched_entity添加到运行队列中。</p><p>对于内存来讲，css_alloc函数就是mem_cgroup_css_alloc。这里面会调用 mem_cgroup_alloc，创建一个struct mem_cgroup。在这个结构中，第一项就是cgroup_subsys_state，也就是说，mem_cgroup是cgroup_subsys_state的一个扩展，最终返回的是指向cgroup_subsys_state结构的指针，我们可以通过强制类型转换变为mem_cgroup。</p><pre><code>struct mem_cgroup {
	struct cgroup_subsys_state css;

	/* Private memcg ID. Used to ID objects that outlive the cgroup */
	struct mem_cgroup_id id;

	/* Accounted resources */
	struct page_counter memory;
	struct page_counter swap;

	/* Legacy consumer-oriented counters */
	struct page_counter memsw;
	struct page_counter kmem;
	struct page_counter tcpmem;

	/* Normal memory consumption range */
	unsigned long low;
	unsigned long high;

	/* Range enforcement for interrupt charges */
	struct work_struct high_work;

	unsigned long soft_limit;

......
	int	swappiness;
......
	/*
	 * percpu counter.
	 */
	struct mem_cgroup_stat_cpu __percpu *stat;

	int last_scanned_node;

	/* List of events which userspace want to receive */
	struct list_head event_list;
	spinlock_t event_list_lock;

	struct mem_cgroup_per_node *nodeinfo[0];
	/* WARNING: nodeinfo must be the last member here */
};
</code></pre><p>在cgroup_init函数中，cgroup的初始化还做了一件很重要的事情，它会调用cgroup_init_cftypes(NULL, cgroup1_base_files)，来初始化对于cgroup文件类型cftype的操作函数，也就是将struct kernfs_ops *kf_ops设置为cgroup_kf_ops。</p><pre><code>struct cftype cgroup1_base_files[] = {
......
    {   
        .name = &quot;tasks&quot;,
        .seq_start = cgroup_pidlist_start,
        .seq_next = cgroup_pidlist_next,
        .seq_stop = cgroup_pidlist_stop,
        .seq_show = cgroup_pidlist_show,
        .private = CGROUP_FILE_TASKS,
        .write = cgroup_tasks_write,
    },  
}

static struct kernfs_ops cgroup_kf_ops = {
	.atomic_write_len	= PAGE_SIZE,
	.open			= cgroup_file_open,
	.release		= cgroup_file_release,
	.write			= cgroup_file_write,
	.seq_start		= cgroup_seqfile_start,
	.seq_next		= cgroup_seqfile_next,
	.seq_stop		= cgroup_seqfile_stop,
	.seq_show		= cgroup_seqfile_show,
};
</code></pre><p>在cgroup初始化完毕之后，接下来就是创建一个cgroup的文件系统，用于配置和操作cgroup。</p><p>cgroup是一种特殊的文件系统。它的定义如下：</p><pre><code>struct file_system_type cgroup_fs_type = {
	.name = &quot;cgroup&quot;,
	.mount = cgroup_mount,
	.kill_sb = cgroup_kill_sb,
	.fs_flags = FS_USERNS_MOUNT,
};
</code></pre><p>当我们mount这个cgroup文件系统的时候，会调用cgroup_mount-&gt;cgroup1_mount。</p><pre><code>struct dentry *cgroup1_mount(struct file_system_type *fs_type, int flags,
			     void *data, unsigned long magic,
			     struct cgroup_namespace *ns)
{
	struct super_block *pinned_sb = NULL;
	struct cgroup_sb_opts opts;
	struct cgroup_root *root;
	struct cgroup_subsys *ss;
	struct dentry *dentry;
	int i, ret;
	bool new_root = false;
......
	root = kzalloc(sizeof(*root), GFP_KERNEL);
	new_root = true;

	init_cgroup_root(root, &amp;opts);

	ret = cgroup_setup_root(root, opts.subsys_mask, PERCPU_REF_INIT_DEAD);
......
	dentry = cgroup_do_mount(&amp;cgroup_fs_type, flags, root,
				 CGROUP_SUPER_MAGIC, ns);
......
	return dentry;
}
</code></pre><p>cgroup被组织成为树形结构，因而有cgroup_root。init_cgroup_root会初始化这个cgroup_root。cgroup_root是cgroup的根，它有一个成员kf_root，是cgroup文件系统的根struct kernfs_root。kernfs_create_root就是用来创建这个kernfs_root结构的。</p><pre><code>int cgroup_setup_root(struct cgroup_root *root, u16 ss_mask, int ref_flags)
{
	LIST_HEAD(tmp_links);
	struct cgroup *root_cgrp = &amp;root-&gt;cgrp;
	struct kernfs_syscall_ops *kf_sops;
	struct css_set *cset;
	int i, ret;

	root-&gt;kf_root = kernfs_create_root(kf_sops,
					   KERNFS_ROOT_CREATE_DEACTIVATED,
					   root_cgrp);
	root_cgrp-&gt;kn = root-&gt;kf_root-&gt;kn;

	ret = css_populate_dir(&amp;root_cgrp-&gt;self);
	ret = rebind_subsystems(root, ss_mask);
......
	list_add(&amp;root-&gt;root_list, &amp;cgroup_roots);
	cgroup_root_count++;
......
	kernfs_activate(root_cgrp-&gt;kn);
......
}
</code></pre><p>就像在普通文件系统上，每一个文件都对应一个inode，在cgroup文件系统上，每个文件都对应一个struct kernfs_node结构，当然kernfs_root作为文件系的根也对应一个kernfs_node结构。</p><p>接下来，css_populate_dir会调用cgroup_addrm_files-&gt;cgroup_add_file-&gt;cgroup_add_file，来创建整棵文件树，并且为树中的每个文件创建对应的kernfs_node结构，并将这个文件的操作函数设置为kf_ops，也即指向cgroup_kf_ops 。</p><pre><code>static int cgroup_add_file(struct cgroup_subsys_state *css, struct cgroup *cgrp,
			   struct cftype *cft)
{
	char name[CGROUP_FILE_NAME_MAX];
	struct kernfs_node *kn;
......
	kn = __kernfs_create_file(cgrp-&gt;kn, cgroup_file_name(cgrp, cft, name),
				  cgroup_file_mode(cft), 0, cft-&gt;kf_ops, cft,
				  NULL, key);
......
}

struct kernfs_node *__kernfs_create_file(struct kernfs_node *parent,
					 const char *name,
					 umode_t mode, loff_t size,
					 const struct kernfs_ops *ops,
					 void *priv, const void *ns,
					 struct lock_class_key *key)
{
	struct kernfs_node *kn;
	unsigned flags;
	int rc;

	flags = KERNFS_FILE;

	kn = kernfs_new_node(parent, name, (mode &amp; S_IALLUGO) | S_IFREG, flags);

	kn-&gt;attr.ops = ops;
	kn-&gt;attr.size = size;
	kn-&gt;ns = ns;
	kn-&gt;priv = priv;
......
	rc = kernfs_add_one(kn);
......
	return kn;
}
</code></pre><p>从cgroup_setup_root返回后，接下来，在cgroup1_mount中，要做的一件事情是cgroup_do_mount，调用kernfs_mount真的去mount这个文件系统，返回一个普通的文件系统都认识的dentry。这种特殊的文件系统对应的文件操作函数为kernfs_file_fops。</p><pre><code>const struct file_operations kernfs_file_fops = {
	.read		= kernfs_fop_read,
	.write		= kernfs_fop_write,
	.llseek		= generic_file_llseek,
	.mmap		= kernfs_fop_mmap,
	.open		= kernfs_fop_open,
	.release	= kernfs_fop_release,
	.poll		= kernfs_fop_poll,
	.fsync		= noop_fsync,
};
</code></pre><p>当我们要写入一个CGroup文件来设置参数的时候，根据文件系统的操作，kernfs_fop_write会被调用，在这里面会调用kernfs_ops的write函数，根据上面的定义为cgroup_file_write，在这里会调用cftype的write函数。对于CPU和内存的write函数，有以下不同的定义。</p><pre><code>static struct cftype cpu_files[] = {
#ifdef CONFIG_FAIR_GROUP_SCHED
    {   
        .name = &quot;shares&quot;,
        .read_u64 = cpu_shares_read_u64,
        .write_u64 = cpu_shares_write_u64,
    },  
#endif
#ifdef CONFIG_CFS_BANDWIDTH
    {   
        .name = &quot;cfs_quota_us&quot;,
        .read_s64 = cpu_cfs_quota_read_s64,
        .write_s64 = cpu_cfs_quota_write_s64,
    },  
    {   
        .name = &quot;cfs_period_us&quot;,
        .read_u64 = cpu_cfs_period_read_u64,
        .write_u64 = cpu_cfs_period_write_u64,
    },  
}


static struct cftype mem_cgroup_legacy_files[] = {
    {   
        .name = &quot;usage_in_bytes&quot;,
        .private = MEMFILE_PRIVATE(_MEM, RES_USAGE),
        .read_u64 = mem_cgroup_read_u64,
    },  
    {   
        .name = &quot;max_usage_in_bytes&quot;,
        .private = MEMFILE_PRIVATE(_MEM, RES_MAX_USAGE),
        .write = mem_cgroup_reset,
        .read_u64 = mem_cgroup_read_u64,
    },  
    {   
        .name = &quot;limit_in_bytes&quot;,
        .private = MEMFILE_PRIVATE(_MEM, RES_LIMIT),
        .write = mem_cgroup_write,
        .read_u64 = mem_cgroup_read_u64,
    },  
    {   
        .name = &quot;soft_limit_in_bytes&quot;,
        .private = MEMFILE_PRIVATE(_MEM, RES_SOFT_LIMIT),
        .write = mem_cgroup_write,
        .read_u64 = mem_cgroup_read_u64,
    },  
}
</code></pre><p>如果设置的是cpu.shares，则调用cpu_shares_write_u64。在这里面，task_group的shares变量更新了，并且更新了CPU队列上的调度实体。</p><pre><code>int sched_group_set_shares(struct task_group *tg, unsigned long shares)
{
	int i;

	shares = clamp(shares, scale_load(MIN_SHARES), scale_load(MAX_SHARES));

	tg-&gt;shares = shares;
	for_each_possible_cpu(i) {
		struct rq *rq = cpu_rq(i);
		struct sched_entity *se = tg-&gt;se[i];
		struct rq_flags rf;

		update_rq_clock(rq);
		for_each_sched_entity(se) {
			update_load_avg(se, UPDATE_TG);
			update_cfs_shares(se);
		}
	}
......
}
</code></pre><p>但是这个时候别忘了，我们还没有将CPU的文件夹下面的tasks文件写入进程号呢。写入一个进程号到tasks文件里面，按照cgroup1_base_files里面的定义，我们应该调用cgroup_tasks_write。</p><p>接下来的调用链为：cgroup_tasks_write-&gt;__cgroup_procs_write-&gt;cgroup_attach_task-&gt; cgroup_migrate-&gt;cgroup_migrate_execute。将这个进程和一个cgroup关联起来，也即将这个进程迁移到这个cgroup下面。</p><pre><code>static int cgroup_migrate_execute(struct cgroup_mgctx *mgctx)
{
	struct cgroup_taskset *tset = &amp;mgctx-&gt;tset;
	struct cgroup_subsys *ss;
	struct task_struct *task, *tmp_task;
	struct css_set *cset, *tmp_cset;
......
	if (tset-&gt;nr_tasks) {
		do_each_subsys_mask(ss, ssid, mgctx-&gt;ss_mask) {
			if (ss-&gt;attach) {
				tset-&gt;ssid = ssid;
				ss-&gt;attach(tset);
			}
		} while_each_subsys_mask();
	}
......
}
</code></pre><p>每一个cgroup子系统会调用相应的attach函数。而CPU调用的是cpu_cgroup_attach-&gt; sched_move_task-&gt; sched_change_group。</p><pre><code>static void sched_change_group(struct task_struct *tsk, int type)
{
	struct task_group *tg;

	tg = container_of(task_css_check(tsk, cpu_cgrp_id, true),
			  struct task_group, css);
	tg = autogroup_task_group(tsk, tg);
	tsk-&gt;sched_task_group = tg;

#ifdef CONFIG_FAIR_GROUP_SCHED
	if (tsk-&gt;sched_class-&gt;task_change_group)
		tsk-&gt;sched_class-&gt;task_change_group(tsk, type);
	else
#endif
		set_task_rq(tsk, task_cpu(tsk));
}
</code></pre><p>在sched_change_group中设置这个进程以这个task_group的方式参与调度，从而使得上面的cpu.shares起作用。</p><p>对于内存来讲，写入内存的限制使用函数mem_cgroup_write-&gt;mem_cgroup_resize_limit来设置struct mem_cgroup的memory.limit成员。</p><p>在进程执行过程中，申请内存的时候，我们会调用handle_pte_fault-&gt;do_anonymous_page()-&gt;mem_cgroup_try_charge()。</p><pre><code>int mem_cgroup_try_charge(struct page *page, struct mm_struct *mm,
			  gfp_t gfp_mask, struct mem_cgroup **memcgp,
			  bool compound)
{
	struct mem_cgroup *memcg = NULL;
......
	if (!memcg)
		memcg = get_mem_cgroup_from_mm(mm);

	ret = try_charge(memcg, gfp_mask, nr_pages);
......
}
</code></pre><p>在mem_cgroup_try_charge中，先是调用get_mem_cgroup_from_mm获得这个进程对应的mem_cgroup结构，然后在try_charge中，根据mem_cgroup的限制，看是否可以申请分配内存。</p><p>至此，cgroup对于内存的限制才真正起作用。</p><h2>总结时刻</h2><p>内核中cgroup的工作机制，我们在这里总结一下。</p><p><img src="https://static001.geekbang.org/resource/image/c9/c4/c9cc56d20e6a4bac0f9657e6380a96c4.png?wh=5836*5203" alt=""></p><p>第一步，系统初始化的时候，初始化cgroup的各个子系统的操作函数，分配各个子系统的数据结构。</p><p>第二步，mount cgroup文件系统，创建文件系统的树形结构，以及操作函数。</p><p>第三步，写入cgroup文件，设置cpu或者memory的相关参数，这个时候文件系统的操作函数会调用到cgroup子系统的操作函数，从而将参数设置到cgroup子系统的数据结构中。</p><p>第四步，写入tasks文件，将进程交给某个cgroup进行管理，因为tasks文件也是一个cgroup文件，统一会调用文件系统的操作函数进而调用cgroup子系统的操作函数，将cgroup子系统的数据结构和进程关联起来。</p><p>第五步，对于CPU来讲，会修改scheduled entity，放入相应的队列里面去，从而下次调度的时候就起作用了。对于内存的cgroup设定，只有在申请内存的时候才起作用。</p><h2>课堂练习</h2><p>这里我们用cgroup限制了CPU和内存，如何限制网络呢？给你一个提示tc，请你研究一下。</p><p>欢迎留言和我分享你的疑惑和见解，也欢迎收藏本节内容，反复研读。你也可以把今天的内容分享给你的朋友，和他一起学习和进步。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/3b/36/2d61e080.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>行者</span>
  </div>
  <div class="_2_QraFYR_0">老师，麻烦讲下华为鸿蒙系统，它和linux区别与联系是什么？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-10 15:47:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/39/fa/a7edbc72.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>安排</span>
  </div>
  <div class="_2_QraFYR_0">Cgroup文件系统是只存在内存中吗？每一次设置之后在掉电后是不是就消失了？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 掉电后，进程都挂了，pid都变了，cgroup还有啥用</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-09 07:24:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/a0/3f/06b690ba.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>刘桢</span>
  </div>
  <div class="_2_QraFYR_0">二十天闭关冲北邮</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 加油</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-09 13:16:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/21/85/609a2e51.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>fhchina</span>
  </div>
  <div class="_2_QraFYR_0">cpu.cfs_period_us的单位是us, 微秒不是毫秒</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-23 16:51:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLTFKH3aA1FVyz7VvAIlISibAPbmiaAyQ5fAK3ElyEcXuRmsmicAefXxkhbC11icjBgfbXPXkHHt5O0xw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>羊仔爸比</span>
  </div>
  <div class="_2_QraFYR_0">老师：<br>      请教一下docker 里面我设置了 内存的limit是2g，cgroup 文件中memory.usage_in_bytes这个文件是是包含memory.stat中的total_rss 和total_cache 相加的大小，oom kill的时候会根据memory.usage_in_bytes的值kill吗，如果不是根据这个文件的值kill是根据哪个值进行kill的？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-29 15:20:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/d0/4d/2116c1a4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Bravery168</span>
  </div>
  <div class="_2_QraFYR_0">容器从概念上就是通过各种数据结构的定义建构了一个模型，从执行层面，本质上是对进程资源和行为的定义和控制。牛</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-12-15 10:59:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/e0/cf/43f201f2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>幼儿编程教学</span>
  </div>
  <div class="_2_QraFYR_0">&gt;第五步，对于 CPU 来讲，会修改 scheduled entity，放入相应的队列里面去，从而下次调度的时候就起作用了。对于内存的 cgroup 设定，只有在申请内存的时候才起作用。<br><br>请教老师，2个我比较非常的问题。<br>* 设置 cgroup cpu后，极端情况下，是否会突发cpu超过限制？<br>文章中说”对于 CPU 来讲，会修改 scheduled entity，放入相应的队列里面去，从而下次调度的时候就起作用了“。如果1个进程，占用时间太长，或者短时间内，耗费了很多cpu，是否会引发cpu超量使用，导致系统卡死？k8s中，如果cpu设置的不好（配置大于宿主机实际量），是会引发宿主机卡死的情况。这时候，只能重启宿主机<br>* 设置 cgroup memory后，极端情况下，是否会突发memory超过限制？<br>内存好像会？因为上面说，”只有在申请内存的时候才起作用“。那之前申请的内存呢？申请内存的时候，申请的都是虚拟内存。假设宿主机只有1g内存。2个进程a,b，都申请内存1g。内存应该是在实际使用时，才会去分配。所以，进程a,b运行起来，可以超过宿主机实际1g内存。这时，操作系统会oom。应该是这样吧？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-02 15:14:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/d6/46/5eb5261b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Sudouble</span>
  </div>
  <div class="_2_QraFYR_0">这么多篇深度文章，很好的诠释了十几年前的一个疑问，为什么不让电脑突然断电！内存、缓存里存的大量数据，操作系统还没有触发写，这时突然断电，这部分数据全都丢了。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-19 15:33:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2a/27/65/0790efd2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>EST4What</span>
  </div>
  <div class="_2_QraFYR_0">为什么我mount -t cgroup会显示，而切换到&#47;sys&#47;fs&#47;cgroup时，文件却不见了呢<br><br>[root@jenkins cgroup]# mount -t cgroup <br>cgroup on &#47;sys&#47;fs&#47;cgroup&#47;systemd type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,xattr,release_agent=&#47;usr&#47;lib&#47;systemd&#47;systemd-cgroups-agent,name=systemd)<br>cgroup on &#47;sys&#47;fs&#47;cgroup&#47;memory type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,memory)<br>cgroup on &#47;sys&#47;fs&#47;cgroup&#47;devices type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,devices)<br>cgroup on &#47;sys&#47;fs&#47;cgroup&#47;cpu,cpuacct type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,cpuacct,cpu)<br>cgroup on &#47;sys&#47;fs&#47;cgroup&#47;hugetlb type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,hugetlb)<br>cgroup on &#47;sys&#47;fs&#47;cgroup&#47;cpuset type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,cpuset)<br>cgroup on &#47;sys&#47;fs&#47;cgroup&#47;freezer type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,freezer)<br>cgroup on &#47;sys&#47;fs&#47;cgroup&#47;perf_event type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,perf_event)<br>cgroup on &#47;sys&#47;fs&#47;cgroup&#47;blkio type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,blkio)<br>cgroup on &#47;sys&#47;fs&#47;cgroup&#47;net_cls,net_prio type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,net_prio,net_cls)<br>cgroup on &#47;sys&#47;fs&#47;cgroup&#47;pids type cgroup (rw,nosuid,nodev,noexec,relatime,seclabel,pids)<br>[root@jenkins cgroup]# ls<br>[root@jenkins cgroup]# ll<br>total 0<br><br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-02 15:26:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/5e/61/985f3eb7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>songyy</span>
  </div>
  <div class="_2_QraFYR_0">testnginx 不在docker的repo里面吧。示例里面<br><br> docker run -d --cpu-shares 513 --cpus 2 --cpuset-cpus 1,3 --memory 1024M --memory-swap 1234M --memory-swappiness 7 -p 8081:80 testnginx:1<br><br>这个代码就跑不起来了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-04 21:11:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKOnpl8fRB9r2vED2s8j7Arwbn2K6M6HUBWNjgoqV4uqe94fTGK4WGpOJLeRxXcBXk3dp23eQR0AQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>吴钩</span>
  </div>
  <div class="_2_QraFYR_0">对namespace和cgroup了解增加了很多，赞！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-26 11:37:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4d/fd/0aa0e39f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>许童童</span>
  </div>
  <div class="_2_QraFYR_0">跟着老师一起精进。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-09 15:35:03</div>
  </div>
</div>
</div>
</li>
</ul>