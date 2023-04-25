<audio title="41 _ IPC（中）：不同项目组之间抢资源，如何协调？" src="https://static001.geekbang.org/resource/audio/5e/d6/5e5caec562b544ec17f47848e115cdd6.mp3" controls="controls"></audio> 
<p>了解了如何使用共享内存和信号量集合之后，今天我们来解析一下，内核里面都做了什么。</p><p>不知道你有没有注意到，咱们讲消息队列、共享内存、信号量的机制的时候，我们其实能够从中看到一些统一的规律：<strong>它们在使用之前都要生成key，然后通过key得到唯一的id，并且都是通过xxxget函数。</strong></p><p>在内核里面，这三种进程间通信机制是使用统一的机制管理起来的，都叫ipcxxx。</p><p>为了维护这三种进程间通信进制，在内核里面，我们声明了一个有三项的数组。</p><p>我们通过这段代码，来具体看一看。</p><pre><code>struct ipc_namespace {
......
	struct ipc_ids	ids[3];
......
}

#define IPC_SEM_IDS	0
#define IPC_MSG_IDS	1
#define IPC_SHM_IDS	2

#define sem_ids(ns)	((ns)-&gt;ids[IPC_SEM_IDS])
#define msg_ids(ns)	((ns)-&gt;ids[IPC_MSG_IDS])
#define shm_ids(ns)	((ns)-&gt;ids[IPC_SHM_IDS])
</code></pre><p>根据代码中的定义，第0项用于信号量，第1项用于消息队列，第2项用于共享内存，分别可以通过sem_ids、msg_ids、shm_ids来访问。</p><p>这段代码里面有ns，全称叫namespace。可能不容易理解，你现在可以将它认为是将一台Linux服务器逻辑的隔离为多台Linux服务器的机制，它背后的原理是一个相当大的话题，我们需要在容器那一章详细讲述。现在，你就可以简单的认为没有namespace，整个Linux在一个namespace下面，那这些ids也是整个Linux只有一份。</p><p>接下来，我们再来看struct ipc_ids里面保存了什么。</p><!-- [[[read_end]]] --><p>首先，in_use表示当前有多少个ipc；其次，seq和next_id用于一起生成ipc唯一的id，因为信号量，共享内存，消息队列，它们三个的id也不能重复；ipcs_idr是一棵基数树，我们又碰到它了，一旦涉及从一个整数查找一个对象，它都是最好的选择。</p><pre><code>struct ipc_ids {
	int in_use;
	unsigned short seq;
	struct rw_semaphore rwsem;
	struct idr ipcs_idr;
	int next_id;
};

struct idr {
	struct radix_tree_root	idr_rt;
	unsigned int		idr_next;
};
</code></pre><p>也就是说，对于sem_ids、msg_ids、shm_ids各有一棵基数树。那这棵树里面究竟存放了什么，能够统一管理这三类ipc对象呢？</p><p>通过下面这个函数ipc_obtain_object_idr，我们可以看出端倪。这个函数根据id，在基数树里面找出来的是struct kern_ipc_perm。</p><pre><code>struct kern_ipc_perm *ipc_obtain_object_idr(struct ipc_ids *ids, int id)
{
	struct kern_ipc_perm *out;
	int lid = ipcid_to_idx(id);
	out = idr_find(&amp;ids-&gt;ipcs_idr, lid);
	return out;
}
</code></pre><p>如果我们看用于表示信号量、消息队列、共享内存的结构，就会发现，这三个结构的第一项都是struct kern_ipc_perm。</p><pre><code>struct sem_array {
	struct kern_ipc_perm	sem_perm;	/* permissions .. see ipc.h */
	time_t			sem_ctime;	/* create/last semctl() time */
	struct list_head	pending_alter;	/* pending operations */
						                /* that alter the array */
	struct list_head	pending_const;	/* pending complex operations */
						/* that do not alter semvals */
	struct list_head	list_id;	/* undo requests on this array */
	int			sem_nsems;	/* no. of semaphores in array */
	int			complex_count;	/* pending complex operations */
	unsigned int		use_global_lock;/* &gt;0: global lock required */

	struct sem		sems[];
} __randomize_layout;

struct msg_queue {
	struct kern_ipc_perm q_perm;
	time_t q_stime;			/* last msgsnd time */
	time_t q_rtime;			/* last msgrcv time */
	time_t q_ctime;			/* last change time */
	unsigned long q_cbytes;		/* current number of bytes on queue */
	unsigned long q_qnum;		/* number of messages in queue */
	unsigned long q_qbytes;		/* max number of bytes on queue */
	pid_t q_lspid;			/* pid of last msgsnd */
	pid_t q_lrpid;			/* last receive pid */

	struct list_head q_messages;
	struct list_head q_receivers;
	struct list_head q_senders;
} __randomize_layout;

struct shmid_kernel /* private to the kernel */
{	
	struct kern_ipc_perm	shm_perm;
	struct file		*shm_file;
	unsigned long		shm_nattch;
	unsigned long		shm_segsz;
	time_t			shm_atim;
	time_t			shm_dtim;
	time_t			shm_ctim;
	pid_t			shm_cprid;
	pid_t			shm_lprid;
	struct user_struct	*mlock_user;

	/* The task created the shm object.  NULL if the task is dead. */
	struct task_struct	*shm_creator;
	struct list_head	shm_clist;	/* list by creator */
} __randomize_layout;
</code></pre><p>也就是说，我们完全可以通过struct kern_ipc_perm的指针，通过进行强制类型转换后，得到整个结构。做这件事情的函数如下：</p><pre><code>static inline struct sem_array *sem_obtain_object(struct ipc_namespace *ns, int id)
{
	struct kern_ipc_perm *ipcp = ipc_obtain_object_idr(&amp;sem_ids(ns), id);
	return container_of(ipcp, struct sem_array, sem_perm);
}

static inline struct msg_queue *msq_obtain_object(struct ipc_namespace *ns, int id)
{
	struct kern_ipc_perm *ipcp = ipc_obtain_object_idr(&amp;msg_ids(ns), id);
	return container_of(ipcp, struct msg_queue, q_perm);
}

static inline struct shmid_kernel *shm_obtain_object(struct ipc_namespace *ns, int id)
{
	struct kern_ipc_perm *ipcp = ipc_obtain_object_idr(&amp;shm_ids(ns), id);
	return container_of(ipcp, struct shmid_kernel, shm_perm);
}
</code></pre><p>通过这种机制，我们就可以将信号量、消息队列、共享内存抽象为ipc类型进行统一处理。你有没有觉得，这有点儿面向对象编程中抽象类和实现类的意思？没错，如果你试图去了解C++中类的实现机制，其实也是这么干的。</p><p><img src="https://static001.geekbang.org/resource/image/08/af/082b742753d862cfeae520fb02aa41af.png?wh=1813*1771" alt=""></p><p>有了抽象类，接下来我们来看共享内存和信号量的具体实现。</p><h2>如何创建共享内存？</h2><p>首先，我们来看创建共享内存的的系统调用。</p><pre><code>SYSCALL_DEFINE3(shmget, key_t, key, size_t, size, int, shmflg)
{
	struct ipc_namespace *ns;
	static const struct ipc_ops shm_ops = {
		.getnew = newseg,
		.associate = shm_security,
		.more_checks = shm_more_checks,
	};
	struct ipc_params shm_params;
	ns = current-&gt;nsproxy-&gt;ipc_ns;
	shm_params.key = key;
	shm_params.flg = shmflg;
	shm_params.u.size = size;
	return ipcget(ns, &amp;shm_ids(ns), &amp;shm_ops, &amp;shm_params);
}
</code></pre><p>这里面调用了抽象的ipcget、参数分别为共享内存对应的shm_ids、对应的操作shm_ops以及对应的参数shm_params。</p><p>如果key设置为IPC_PRIVATE则永远创建新的，如果不是的话，就会调用ipcget_public。ipcget的具体代码如下：</p><pre><code>int ipcget(struct ipc_namespace *ns, struct ipc_ids *ids,
			const struct ipc_ops *ops, struct ipc_params *params)
{
	if (params-&gt;key == IPC_PRIVATE)
		return ipcget_new(ns, ids, ops, params);
	else
		return ipcget_public(ns, ids, ops, params);
}

static int ipcget_public(struct ipc_namespace *ns, struct ipc_ids *ids, const struct ipc_ops *ops, struct ipc_params *params)
{
	struct kern_ipc_perm *ipcp;
	int flg = params-&gt;flg;
	int err;
	ipcp = ipc_findkey(ids, params-&gt;key);
	if (ipcp == NULL) {
		if (!(flg &amp; IPC_CREAT))
			err = -ENOENT;
		else
			err = ops-&gt;getnew(ns, params);
	} else {
		if (flg &amp; IPC_CREAT &amp;&amp; flg &amp; IPC_EXCL)
			err = -EEXIST;
		else {
			err = 0;
			if (ops-&gt;more_checks)
				err = ops-&gt;more_checks(ipcp, params);
......
		}
	}
	return err;
}
</code></pre><p>在ipcget_public中，我们会按照key，去查找struct kern_ipc_perm。如果没有找到，那就看是否设置了IPC_CREAT；如果设置了，就创建一个新的。如果找到了，就将对应的id返回。</p><p>我们这里重点看，如何按照参数shm_ops，创建新的共享内存，会调用newseg。</p><pre><code>static int newseg(struct ipc_namespace *ns, struct ipc_params *params)
{
	key_t key = params-&gt;key;
	int shmflg = params-&gt;flg;
	size_t size = params-&gt;u.size;
	int error;
	struct shmid_kernel *shp;
	size_t numpages = (size + PAGE_SIZE - 1) &gt;&gt; PAGE_SHIFT;
	struct file *file;
	char name[13];
	vm_flags_t acctflag = 0;
......
	shp = kvmalloc(sizeof(*shp), GFP_KERNEL);
......
	shp-&gt;shm_perm.key = key;
	shp-&gt;shm_perm.mode = (shmflg &amp; S_IRWXUGO);
	shp-&gt;mlock_user = NULL;

	shp-&gt;shm_perm.security = NULL;
......
	file = shmem_kernel_file_setup(name, size, acctflag);
......
	shp-&gt;shm_cprid = task_tgid_vnr(current);
	shp-&gt;shm_lprid = 0;
	shp-&gt;shm_atim = shp-&gt;shm_dtim = 0;
	shp-&gt;shm_ctim = get_seconds();
	shp-&gt;shm_segsz = size;
	shp-&gt;shm_nattch = 0;
	shp-&gt;shm_file = file;
	shp-&gt;shm_creator = current;

	error = ipc_addid(&amp;shm_ids(ns), &amp;shp-&gt;shm_perm, ns-&gt;shm_ctlmni);
......
	list_add(&amp;shp-&gt;shm_clist, &amp;current-&gt;sysvshm.shm_clist);
......
	file_inode(file)-&gt;i_ino = shp-&gt;shm_perm.id;

	ns-&gt;shm_tot += numpages;
	error = shp-&gt;shm_perm.id;
......
	return error;
}
</code></pre><p><strong>newseg函数的第一步，通过kvmalloc在直接映射区分配一个struct shmid_kernel结构。</strong>这个结构就是用来描述共享内存的。这个结构最开始就是上面说的struct kern_ipc_perm结构。接下来就是填充这个struct shmid_kernel结构，例如key、权限等。</p><p><strong>newseg函数的第二步，共享内存需要和文件进行关联</strong>。**为什么要做这个呢？我们在讲内存映射的时候讲过，虚拟地址空间可以和物理内存关联，但是物理内存是某个进程独享的。虚拟地址空间也可以映射到一个文件，文件是可以跨进程共享的。</p><p>咱们这里的共享内存需要跨进程共享，也应该借鉴文件映射的思路。只不过不应该映射一个硬盘上的文件，而是映射到一个内存文件系统上的文件。mm/shmem.c里面就定义了这样一个基于内存的文件系统。这里你一定要注意区分shmem和shm的区别，前者是一个文件系统，后者是进程通信机制。</p><p>在系统初始化的时候，shmem_init注册了shmem文件系统shmem_fs_type，并且挂在到了shm_mnt下面。</p><pre><code>int __init shmem_init(void)
{
	int error;
	error = shmem_init_inodecache();
	error = register_filesystem(&amp;shmem_fs_type);
	shm_mnt = kern_mount(&amp;shmem_fs_type);
......
	return 0;
}

static struct file_system_type shmem_fs_type = {
	.owner		= THIS_MODULE,
	.name		= &quot;tmpfs&quot;,
	.mount		= shmem_mount,
	.kill_sb	= kill_litter_super,
	.fs_flags	= FS_USERNS_MOUNT,
};
</code></pre><p>接下来，newseg函数会调用shmem_kernel_file_setup，其实就是在shmem文件系统里面创建一个文件。</p><pre><code>/**
 * shmem_kernel_file_setup - get an unlinked file living in tmpfs which must be kernel internal.  
 * @name: name for dentry (to be seen in /proc/&lt;pid&gt;/maps
 * @size: size to be set for the file
 * @flags: VM_NORESERVE suppresses pre-accounting of the entire object size */
struct file *shmem_kernel_file_setup(const char *name, loff_t size, unsigned long flags)
{
	return __shmem_file_setup(name, size, flags, S_PRIVATE);
}

static struct file *__shmem_file_setup(const char *name, loff_t size,
				       unsigned long flags, unsigned int i_flags)
{
	struct file *res;
	struct inode *inode;
	struct path path;
	struct super_block *sb;
	struct qstr this;
......
	this.name = name;
	this.len = strlen(name);
	this.hash = 0; /* will go */
	sb = shm_mnt-&gt;mnt_sb;
	path.mnt = mntget(shm_mnt);
	path.dentry = d_alloc_pseudo(sb, &amp;this);
	d_set_d_op(path.dentry, &amp;anon_ops);
......
	inode = shmem_get_inode(sb, NULL, S_IFREG | S_IRWXUGO, 0, flags);
	inode-&gt;i_flags |= i_flags;
	d_instantiate(path.dentry, inode);
	inode-&gt;i_size = size;
......
	res = alloc_file(&amp;path, FMODE_WRITE | FMODE_READ,
		  &amp;shmem_file_operations);
	return res;
}
</code></pre><p>__shmem_file_setup会创建新的shmem文件对应的dentry和inode，并将它们两个关联起来，然后分配一个struct file结构，来表示新的shmem文件，并且指向独特的shmem_file_operations。</p><pre><code>static const struct file_operations shmem_file_operations = {
	.mmap		= shmem_mmap,
	.get_unmapped_area = shmem_get_unmapped_area,
#ifdef CONFIG_TMPFS
	.llseek		= shmem_file_llseek,
	.read_iter	= shmem_file_read_iter,
	.write_iter	= generic_file_write_iter,
	.fsync		= noop_fsync,
	.splice_read	= generic_file_splice_read,
	.splice_write	= iter_file_splice_write,
	.fallocate	= shmem_fallocate,
#endif
};
</code></pre><p><strong>newseg函数的第三步，通过ipc_addid将新创建的struct shmid_kernel结构挂到shm_ids里面的基数树上，并返回相应的id，并且将struct shmid_kernel挂到当前进程的sysvshm队列中。</strong></p><p>至此，共享内存的创建就完成了。</p><h2>如何将共享内存映射到虚拟地址空间？</h2><p>从上面的代码解析中，我们知道，共享内存的数据结构struct shmid_kernel，是通过它的成员struct file *shm_file，来管理内存文件系统shmem上的内存文件的。无论这个共享内存是否被映射，shm_file都是存在的。</p><p>接下来，我们要将共享内存映射到虚拟地址空间中。调用的是shmat，对应的系统调用如下：</p><pre><code>SYSCALL_DEFINE3(shmat, int, shmid, char __user *, shmaddr, int, shmflg)
{
    unsigned long ret;
    long err;
    err = do_shmat(shmid, shmaddr, shmflg, &amp;ret, SHMLBA);
    force_successful_syscall_return();
    return (long)ret;
}

long do_shmat(int shmid, char __user *shmaddr, int shmflg,
	      ulong *raddr, unsigned long shmlba)
{
	struct shmid_kernel *shp;
	unsigned long addr = (unsigned long)shmaddr;
	unsigned long size;
	struct file *file;
	int    err;
	unsigned long flags = MAP_SHARED;
	unsigned long prot;
	int acc_mode;
	struct ipc_namespace *ns;
	struct shm_file_data *sfd;
	struct path path;
	fmode_t f_mode;
	unsigned long populate = 0;
......
	prot = PROT_READ | PROT_WRITE;
	acc_mode = S_IRUGO | S_IWUGO;
	f_mode = FMODE_READ | FMODE_WRITE;
......
	ns = current-&gt;nsproxy-&gt;ipc_ns;
	shp = shm_obtain_object_check(ns, shmid);
......
	path = shp-&gt;shm_file-&gt;f_path;
	path_get(&amp;path);
	shp-&gt;shm_nattch++;
	size = i_size_read(d_inode(path.dentry));
......
	sfd = kzalloc(sizeof(*sfd), GFP_KERNEL);
......
	file = alloc_file(&amp;path, f_mode,
			  is_file_hugepages(shp-&gt;shm_file) ?
				&amp;shm_file_operations_huge :
				&amp;shm_file_operations);
......
	file-&gt;private_data = sfd;
	file-&gt;f_mapping = shp-&gt;shm_file-&gt;f_mapping;
	sfd-&gt;id = shp-&gt;shm_perm.id;
	sfd-&gt;ns = get_ipc_ns(ns);
	sfd-&gt;file = shp-&gt;shm_file;
	sfd-&gt;vm_ops = NULL;
......
	addr = do_mmap_pgoff(file, addr, size, prot, flags, 0, &amp;populate, NULL);
	*raddr = addr;
	err = 0;
......
	return err;
}
</code></pre><p>在这个函数里面，shm_obtain_object_check会通过共享内存的id，在基数树中找到对应的struct shmid_kernel结构，通过它找到shmem上的内存文件。</p><p>接下来，我们要分配一个struct shm_file_data，来表示这个内存文件。将shmem中指向内存文件的shm_file赋值给struct shm_file_data中的file成员。</p><p>然后，我们创建了一个struct file，指向的也是shmem中的内存文件。</p><p>为什么要再创建一个呢？这两个的功能不同，shmem中shm_file用于管理内存文件，是一个中立的，独立于任何一个进程的角色。而新创建的struct file是专门用于做内存映射的，就像咱们在讲内存映射那一节讲过的，一个硬盘上的文件要映射到虚拟地址空间中的时候，需要在vm_area_struct里面有一个struct file *vm_file指向硬盘上的文件，现在变成内存文件了，但是这个结构还是不能少。</p><p>新创建的struct file的private_data，指向struct shm_file_data，这样内存映射那部分的数据结构，就能够通过它来访问内存文件了。</p><p>新创建的struct file的file_operations也发生了变化，变成了shm_file_operations。</p><pre><code>static const struct file_operations shm_file_operations = {
	.mmap		= shm_mmap,
	.fsync		= shm_fsync,
	.release	= shm_release,
	.get_unmapped_area	= shm_get_unmapped_area,
	.llseek		= noop_llseek,
	.fallocate	= shm_fallocate,
};
</code></pre><p>接下来，do_mmap_pgoff函数我们遇到过，原来映射硬盘上的文件的时候，也是调用它。这里我们不再详细解析了。它会分配一个vm_area_struct指向虚拟地址空间中没有分配的区域，它的vm_file指向这个内存文件，然后它会调用shm_file_operations的mmap函数，也即shm_mmap进行映射。</p><pre><code>static int shm_mmap(struct file *file, struct vm_area_struct *vma)
{
	struct shm_file_data *sfd = shm_file_data(file);
	int ret;
	ret = __shm_open(vma);
	ret = call_mmap(sfd-&gt;file, vma);
	sfd-&gt;vm_ops = vma-&gt;vm_ops;
	vma-&gt;vm_ops = &amp;shm_vm_ops;
	return 0;
}
</code></pre><p>shm_mmap中调用了shm_file_data中的file的mmap函数，这次调用的是shmem_file_operations的mmap，也即shmem_mmap。</p><pre><code>static int shmem_mmap(struct file *file, struct vm_area_struct *vma)
{
	file_accessed(file);
	vma-&gt;vm_ops = &amp;shmem_vm_ops;
	return 0;
}
</code></pre><p>这里面，vm_area_struct的vm_ops指向shmem_vm_ops。等从call_mmap中返回之后，shm_file_data的vm_ops指向了shmem_vm_ops，而vm_area_struct的vm_ops改为指向shm_vm_ops。</p><p>我们来看一下，shm_vm_ops和shmem_vm_ops的定义。</p><pre><code>static const struct vm_operations_struct shm_vm_ops = {
	.open	= shm_open,	/* callback for a new vm-area open */
	.close	= shm_close,	/* callback for when the vm-area is released */
	.fault	= shm_fault,
};

static const struct vm_operations_struct shmem_vm_ops = {
	.fault		= shmem_fault,
	.map_pages	= filemap_map_pages,
};
</code></pre><p>它们里面最关键的就是fault函数，也即访问虚拟内存的时候，访问不到应该怎么办。</p><p>当访问不到的时候，先调用vm_area_struct的vm_ops，也即shm_vm_ops的fault函数shm_fault。然后它会转而调用shm_file_data的vm_ops，也即shmem_vm_ops的fault函数shmem_fault。</p><pre><code>static int shm_fault(struct vm_fault *vmf)
{
	struct file *file = vmf-&gt;vma-&gt;vm_file;
	struct shm_file_data *sfd = shm_file_data(file);
	return sfd-&gt;vm_ops-&gt;fault(vmf);
}
</code></pre><p>虽然基于内存的文件系统，已经为这个内存文件分配了inode，但是内存也却是一点儿都没分配，只有在发生缺页异常的时候才进行分配。</p><pre><code>static int shmem_fault(struct vm_fault *vmf)
{
	struct vm_area_struct *vma = vmf-&gt;vma;
	struct inode *inode = file_inode(vma-&gt;vm_file);
	gfp_t gfp = mapping_gfp_mask(inode-&gt;i_mapping);
......
	error = shmem_getpage_gfp(inode, vmf-&gt;pgoff, &amp;vmf-&gt;page, sgp,
				  gfp, vma, vmf, &amp;ret);
......
}

/*
 * shmem_getpage_gfp - find page in cache, or get from swap, or allocate
 *
 * If we allocate a new one we do not mark it dirty. That's up to the
 * vm. If we swap it in we mark it dirty since we also free the swap
 * entry since a page cannot live in both the swap and page cache.
 *
 * fault_mm and fault_type are only supplied by shmem_fault:
 * otherwise they are NULL.
 */
static int shmem_getpage_gfp(struct inode *inode, pgoff_t index,
	struct page **pagep, enum sgp_type sgp, gfp_t gfp,
	struct vm_area_struct *vma, struct vm_fault *vmf, int *fault_type)
{
......
    page = shmem_alloc_and_acct_page(gfp, info, sbinfo,
					index, false);
......
}
</code></pre><p>shmem_fault会调用shmem_getpage_gfp在page cache和swap中找一个空闲页，如果找不到就通过shmem_alloc_and_acct_page分配一个新的页，他最终会调用内存管理系统的alloc_page_vma在物理内存中分配一个页。</p><p>至此，共享内存才真的映射到了虚拟地址空间中，进程可以像访问本地内存一样访问共享内存。</p><h2>总结时刻</h2><p>我们来总结一下共享内存的创建和映射过程。</p><ol>
<li>调用shmget创建共享内存。</li>
<li>先通过ipc_findkey在基数树中查找key对应的共享内存对象shmid_kernel是否已经被创建过，如果已经被创建，就会被查询出来，例如producer创建过，在consumer中就会查询出来。</li>
<li>如果共享内存没有被创建过，则调用shm_ops的newseg方法，创建一个共享内存对象shmid_kernel。例如，在producer中就会新建。</li>
<li>在shmem文件系统里面创建一个文件，共享内存对象shmid_kernel指向这个文件，这个文件用struct file表示，我们姑且称它为file1。</li>
<li>调用shmat，将共享内存映射到虚拟地址空间。</li>
<li>shm_obtain_object_check先从基数树里面找到shmid_kernel对象。</li>
<li>创建用于内存映射到文件的file和shm_file_data，这里的struct file我们姑且称为file2。</li>
<li>关联内存区域vm_area_struct和用于内存映射到文件的file，也即file2，调用file2的mmap函数。</li>
<li>file2的mmap函数shm_mmap，会调用file1的mmap函数shmem_mmap，设置shm_file_data和vm_area_struct的vm_ops。</li>
<li>内存映射完毕之后，其实并没有真的分配物理内存，当访问内存的时候，会触发缺页异常do_page_fault。</li>
<li>vm_area_struct的vm_ops的shm_fault会调用shm_file_data的vm_ops的shmem_fault。</li>
<li>在page cache中找一个空闲页，或者创建一个空闲页。</li>
</ol><p><img src="https://static001.geekbang.org/resource/image/20/51/20e8f4e69d47b7469f374bc9fbcf7251.png?wh=4903*3352" alt=""></p><h2>课堂练习</h2><p>在这里，我们只分析了shm_ids的结构，消息队列的程序我们写过了，但是msg_ids的结构没有解析，你可以试着解析一下。</p><p>欢迎留言和我分享你的疑惑和见解 ，也欢迎可以收藏本节内容，反复研读。你也可以把今天的内容分享给你的朋友，和他一起学习和进步。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/a6/43/cb6ab349.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Spring</span>
  </div>
  <div class="_2_QraFYR_0">文章一遍看不懂但底下总结的图很好，终于明白了为什么需要两个file。file1是shmem内存文件系统里的文件，file2是进程虚拟内存里映射的文件，所以file1是属于共享内存的，file2是属于某个进程的。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-27 22:51:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/fe/34/67c1ed1e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小橙子</span>
  </div>
  <div class="_2_QraFYR_0">工作了几年 ，业务代码写多了，框架与API调来调去的，遇到很多疑难杂症，还是不明所以。<br>回过头再看下 操作系统真是核心，很多人说操作系统就是功夫里面的易筋经，内功章法。学习了操作系统，再看很多其他的技术，感觉更自然，理解的更深刻了。一直想读内核代码，但是啃起来很费劲，这个专栏一直再看，越看越喜欢，很多篇章都会反复的看。相信看完专栏后，再去看一些深入理解linux内核，会清晰很多。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-06 10:31:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/11/e7/044a9a6c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>book尾汁</span>
  </div>
  <div class="_2_QraFYR_0">补充下该查了下file是VFS框架的一个基本概念，一切皆文件指的就是这个,f,然后在这个file下面会有各种各样的实现,比如设备是文件 sock是文件 pipe是文件 共享内存也是文件,file结构体里面都是一些通用的属性,而private_data里面是一些各个披着文件外衣的各种结构体的一个独特的东西,因此这里会有两个file,vm_file就是这个外壳,其private_data里就是共享内存的相关数据</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对的，</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-28 00:11:03</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/11/e7/044a9a6c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>book尾汁</span>
  </div>
  <div class="_2_QraFYR_0">共享内存:<br>创建共享内存,通过shmget系统调用来创建一个共享内存,主要是通过key来创建一个struct kern_ipc_perm,信号量 队列和共享内存的结构都是一样的,可以通过强制类型转换来转化成任意一个类型.然后接下来去填充这个结构,然后将一个文件与共享内存关联,内存映射有两种方式一种是匿名映射 一种是映射到一个文件,物理内存是不能共享的,文件可以跨进程共享,所以要映射到文件.但这里的文件是一个特殊的文件系统内存文件系统上的文件,这个内存文件系统也会在系统初始化的时候注册 挂载, 该文件也有目录项和inode以及其fs_operation,然后会将新创建的共享内存对象加入到shm_ids基数上,并将其加到当前进程的sysvshm队列中.<br><br>现在共享内存其实还只是内核中的一个结构体,我们只有共享内存的id,要想使用共享内存还要通过id来将其映射到使用者的用户态的进程空间中,通过shmat系统调用可以做到这一点,该系统调用的主要工作如下:<br>1 通过共享内存的ip在 shm_ids基数树上找到该共享内存的结构体,然后取出内存文件系统里file并将其赋值给新创建的struct shm_file_data-&gt;file,这里我们已经有了可以共享的文件&quot;file&quot;,然后在用户进程虚拟空间的mmap映射区分配一个vm_area struct来做文件映射就可以了,将vm_area_struct里的vm_file的private_data指向shm_file_data,为什么不能直接用file呢?private_data貌似只有共享内存才有用,不太理解,可能因为vm_file有其独特的file_operation的问题吧,两个file&#39;虽然可以是同一类结构体,但差异还是很大.在创建vm_file的过程中应该可以找到答案,略过,映射内存时还会将vm_area的vm_ops先指向shmem_vm_ops,然后在指向shm_vm_ops, 将shm_file_data的vm_ops指向shmem_vm_ops即内存文件系统的文件的vm_ops,到这里就完成了.后面进程读或写数据时,如果对应的页表项没有建立会触发缺页异常,跟之前的缺页异常流程差不多,最终会调用内存文件系统的缺页异常函数来分配对应的物理页,并建立页表项.</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-27 23:51:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83eov38ZkwCyNoBdr5drgX0cp2eOGCv7ibkhUIqCvcnFk8FyUIS6K4gHXIXh0fu7TB67jaictdDlic4OwQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>珠闪闪</span>
  </div>
  <div class="_2_QraFYR_0">文章两遍读下来蒙蒙的，最后对着总结图把共享内存的创建和在用户态映射的流程理顺了。关键就是因为共享内存刚开始申请的物理内存无法在进程中共享，所以先要把物理内存的shmid_kernel对应到shmem文件系统的一个文件，这样shmem中的文件可以再进程中共享；然后在shmat函数时，相当于将shm映射到shmem的file2，先映射到shmem文件系统的文件file1，然后再通过file1的mmap函数完成shm_file_data和vm_area_struct的ops设置。这样内存映射完毕后，并没有真的分配物理内存，当访问内存会触发缺页异常。然后vm_area_struct的vm_ops的shm_fault会调用shm_file_data的vm_ops的shmem_fault。最后在page cache中找一个空闲页，或者创建一个空闲页。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-08 11:08:14</div>
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
  <div class="_2_QraFYR_0">老师有没有什么通俗易懂的资料，您将的太专业了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 的确比较硬核，我实在是没办法再比喻了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-02 15:14:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2c/b8/94/d20583ef.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>NeverSeeYouAgainBUG</span>
  </div>
  <div class="_2_QraFYR_0">哎呀超哥，深入浅出啊深入浅出啊。关键在 浅出，要好好总结，</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-12 20:00:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/fd/08/c039f840.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小鳄鱼</span>
  </div>
  <div class="_2_QraFYR_0">为了统一操作入口：一切皆文件。构建了各种各样的“文件系统”。共享内存文件系统，硬盘文件系统（ext4等），设备文件系统，相信还有各种各样的文件系统！虽然感觉复杂了，但是实际的场景本来就不简单。反而统一入口之后，“上层建筑”的开发人员不再需要关系底层的具体实现，从而实现并行开发，独立维护。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-06-02 08:44:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_2b44d4</span>
  </div>
  <div class="_2_QraFYR_0">这个page cache 跟swap的选择时机是怎样的？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-09 08:45:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/8d/3b/42d9c669.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>艾瑞克小霸王</span>
  </div>
  <div class="_2_QraFYR_0">对于 sem_ids、msg_ids、shm_ids 各有一棵基数树<br>---------------------------------------------------<br>应该是共享一个树吧？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-06 22:32:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/47/fe/f2ce12cd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Leosocy</span>
  </div>
  <div class="_2_QraFYR_0">seq 和 next_id 用于一起生成 ipc 唯一的 id，因为信号量，共享内存，消息队列，它们三个的 id 也不能重复<br><br>这句话不太明白，不同的ipc_ids不是有不同的idr吗？为什么要保证他们三个id不重复？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-14 12:06:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/90/8f/9c691a5f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>奔跑的码仔</span>
  </div>
  <div class="_2_QraFYR_0">将本节所讲的共享内存实现流程与文件内存映那节所讲的流程对比着梳理一下，感觉明朗了好多</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-30 10:28:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/1c/6f/3ea2a599.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>嘉木</span>
  </div>
  <div class="_2_QraFYR_0">C的面向对象居然这么巧妙</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-13 09:58:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/7a/e3/145adba9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>不一样的烟火</span>
  </div>
  <div class="_2_QraFYR_0">听完了 快点更新😁</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 牛</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-01 12:18:59</div>
  </div>
</div>
</div>
</li>
</ul>