<audio title="31丨性能篇答疑：epoll源码深度剖析" src="https://static001.geekbang.org/resource/audio/18/00/18e13085c3d778edecb408b321e3a800.mp3" controls="controls"></audio> 
<p>你好，我是盛延敏，今天是网络编程实战性能篇的答疑模块，欢迎回来。</p><p>在性能篇中，我主要围绕C10K问题进行了深入剖析，最后引出了事件分发机制和多线程。可以说，基于epoll的事件分发能力，是Linux下高性能网络编程的不二之选。如果你觉得还不过瘾，期望有更深刻的认识和理解，那么在性能篇的答疑中，我就带你一起梳理一下epoll的源代码，从中我们一定可以有更多的发现和领悟。</p><p>今天的代码有些多，建议你配合文稿收听音频。</p><h2>基本数据结构</h2><p>在开始研究源代码之前，我们先看一下epoll中使用的数据结构，分别是eventpoll、epitem和eppoll_entry。</p><p>我们先看一下eventpoll这个数据结构，这个数据结构是我们在调用epoll_create之后内核侧创建的一个句柄，表示了一个epoll实例。后续如果我们再调用epoll_ctl和epoll_wait等，都是对这个eventpoll数据进行操作，这部分数据会被保存在epoll_create创建的匿名文件file的private_data字段中。</p><pre><code>/*
 * This structure is stored inside the &quot;private_data&quot; member of the file
 * structure and represents the main data structure for the eventpoll
 * interface.
 */
struct eventpoll {
    /* Protect the access to this structure */
    spinlock_t lock;

    /*
     * This mutex is used to ensure that files are not removed
     * while epoll is using them. This is held during the event
     * collection loop, the file cleanup path, the epoll file exit
     * code and the ctl operations.
     */
    struct mutex mtx;

    /* Wait queue used by sys_epoll_wait() */
    //这个队列里存放的是执行epoll_wait从而等待的进程队列
    wait_queue_head_t wq;

    /* Wait queue used by file-&gt;poll() */
    //这个队列里存放的是该eventloop作为poll对象的一个实例，加入到等待的队列
    //这是因为eventpoll本身也是一个file, 所以也会有poll操作
    wait_queue_head_t poll_wait;

    /* List of ready file descriptors */
    //这里存放的是事件就绪的fd列表，链表的每个元素是下面的epitem
    struct list_head rdllist;

    /* RB tree root used to store monitored fd structs */
    //这是用来快速查找fd的红黑树
    struct rb_root_cached rbr;

    /*
     * This is a single linked list that chains all the &quot;struct epitem&quot; that
     * happened while transferring ready events to userspace w/out
     * holding -&gt;lock.
     */
    struct epitem *ovflist;

    /* wakeup_source used when ep_scan_ready_list is running */
    struct wakeup_source *ws;

    /* The user that created the eventpoll descriptor */
    struct user_struct *user;

    //这是eventloop对应的匿名文件，充分体现了Linux下一切皆文件的思想
    struct file *file;

    /* used to optimize loop detection check */
    int visited;
    struct list_head visited_list_link;

#ifdef CONFIG_NET_RX_BUSY_POLL
    /* used to track busy poll napi_id */
    unsigned int napi_id;
#endif
};
</code></pre><p>你能看到在代码中我提到了epitem，这个epitem结构是干什么用的呢？</p><p>每当我们调用epoll_ctl增加一个fd时，内核就会为我们创建出一个epitem实例，并且把这个实例作为红黑树的一个子节点，增加到eventpoll结构体中的红黑树中，对应的字段是rbr。这之后，查找每一个fd上是否有事件发生都是通过红黑树上的epitem来操作。</p><!-- [[[read_end]]] --><pre><code>/*
 * Each file descriptor added to the eventpoll interface will
 * have an entry of this type linked to the &quot;rbr&quot; RB tree.
 * Avoid increasing the size of this struct, there can be many thousands
 * of these on a server and we do not want this to take another cache line.
 */
struct epitem {
    union {
        /* RB tree node links this structure to the eventpoll RB tree */
        struct rb_node rbn;
        /* Used to free the struct epitem */
        struct rcu_head rcu;
    };

    /* List header used to link this structure to the eventpoll ready list */
    //将这个epitem连接到eventpoll 里面的rdllist的list指针
    struct list_head rdllink;

    /*
     * Works together &quot;struct eventpoll&quot;-&gt;ovflist in keeping the
     * single linked chain of items.
     */
    struct epitem *next;

    /* The file descriptor information this item refers to */
    //epoll监听的fd
    struct epoll_filefd ffd;

    /* Number of active wait queue attached to poll operations */
    //一个文件可以被多个epoll实例所监听，这里就记录了当前文件被监听的次数
    int nwait;

    /* List containing poll wait queues */
    struct list_head pwqlist;

    /* The &quot;container&quot; of this item */
    //当前epollitem所属的eventpoll
    struct eventpoll *ep;

    /* List header used to link this item to the &quot;struct file&quot; items list */
    struct list_head fllink;

    /* wakeup_source used when EPOLLWAKEUP is set */
    struct wakeup_source __rcu *ws;

    /* The structure that describe the interested events and the source fd */
    struct epoll_event event;
};
</code></pre><p>每次当一个fd关联到一个epoll实例，就会有一个eppoll_entry产生。eppoll_entry的结构如下：</p><pre><code>/* Wait structure used by the poll hooks */
struct eppoll_entry {
    /* List header used to link this structure to the &quot;struct epitem&quot; */
    struct list_head llink;

    /* The &quot;base&quot; pointer is set to the container &quot;struct epitem&quot; */
    struct epitem *base;

    /*
     * Wait queue item that will be linked to the target file wait
     * queue head.
     */
    wait_queue_entry_t wait;

    /* The wait queue head that linked the &quot;wait&quot; wait queue item */
    wait_queue_head_t *whead;
};
</code></pre><h2>epoll_create</h2><p>我们在使用epoll的时候，首先会调用epoll_create来创建一个epoll实例。这个函数是如何工作的呢?</p><p>首先，epoll_create会对传入的flags参数做简单的验证。</p><pre><code>/* Check the EPOLL_* constant for consistency.  */
BUILD_BUG_ON(EPOLL_CLOEXEC != O_CLOEXEC);

if (flags &amp; ~EPOLL_CLOEXEC)
    return -EINVAL;
/*
</code></pre><p>接下来，内核申请分配eventpoll需要的内存空间。</p><pre><code>/* Create the internal data structure (&quot;struct eventpoll&quot;).
*/
error = ep_alloc(&amp;ep);
if (error &lt; 0)
  return error;
</code></pre><p>在接下来，epoll_create为epoll实例分配了匿名文件和文件描述字，其中fd是文件描述字，file是一个匿名文件。这里充分体现了UNIX下一切都是文件的思想。注意，eventpoll的实例会保存一份匿名文件的引用，通过调用fd_install函数将匿名文件和文件描述字完成了绑定。</p><p>这里还有一个特别需要注意的地方，在调用anon_inode_get_file的时候，epoll_create将eventpoll作为匿名文件file的private_data保存了起来，这样，在之后通过epoll实例的文件描述字来查找时，就可以快速地定位到eventpoll对象了。</p><p>最后，这个文件描述字作为epoll的文件句柄，被返回给epoll_create的调用者。</p><pre><code>/*
 * Creates all the items needed to setup an eventpoll file. That is,
 * a file structure and a free file descriptor.
 */
fd = get_unused_fd_flags(O_RDWR | (flags &amp; O_CLOEXEC));
if (fd &lt; 0) {
    error = fd;
    goto out_free_ep;
}
file = anon_inode_getfile(&quot;[eventpoll]&quot;, &amp;eventpoll_fops, ep,
             O_RDWR | (flags &amp; O_CLOEXEC));
if (IS_ERR(file)) {
    error = PTR_ERR(file);
    goto out_free_fd;
}
ep-&gt;file = file;
fd_install(fd, file);
return fd;
</code></pre><h2>epoll_ctl</h2><p>接下来，我们看一下一个套接字是如何被添加到epoll实例中的。这就要解析一下epoll_ctl函数实现了。</p><h3>查找epoll实例</h3><p>首先，epoll_ctl函数通过epoll实例句柄来获得对应的匿名文件，这一点很好理解，UNIX下一切都是文件，epoll的实例也是一个匿名文件。</p><pre><code>//获得epoll实例对应的匿名文件
f = fdget(epfd);
if (!f.file)
    goto error_return;
</code></pre><p>接下来，获得添加的套接字对应的文件，这里tf表示的是target file，即待处理的目标文件。</p><pre><code>/* Get the &quot;struct file *&quot; for the target file */
//获得真正的文件，如监听套接字、读写套接字
tf = fdget(fd);
if (!tf.file)
    goto error_fput;
</code></pre><p>再接下来，进行了一系列的数据验证，以保证用户传入的参数是合法的，比如epfd真的是一个epoll实例句柄，而不是一个普通文件描述符。</p><pre><code>/* The target file descriptor must support poll */
//如果不支持poll，那么该文件描述字是无效的
error = -EPERM;
if (!tf.file-&gt;f_op-&gt;poll)
    goto error_tgt_fput;
...
</code></pre><p>如果获得了一个真正的epoll实例句柄，就可以通过private_data获取之前创建的eventpoll实例了。</p><pre><code>/*
 * At this point it is safe to assume that the &quot;private_data&quot; contains
 * our own data structure.
 */
ep = f.file-&gt;private_data;
</code></pre><h3>红黑树查找</h3><p>接下来epoll_ctl通过目标文件和对应描述字，在红黑树中查找是否存在该套接字，这也是epoll为什么高效的地方。红黑树（RB-tree）是一种常见的数据结构，这里eventpoll通过红黑树跟踪了当前监听的所有文件描述字，而这棵树的根就保存在eventpoll数据结构中。</p><pre><code>/* RB tree root used to store monitored fd structs */
struct rb_root_cached rbr;
</code></pre><p>对于每个被监听的文件描述字，都有一个对应的epitem与之对应，epitem作为红黑树中的节点就保存在红黑树中。</p><pre><code>/*
 * Try to lookup the file inside our RB tree, Since we grabbed &quot;mtx&quot;
 * above, we can be sure to be able to use the item looked up by
 * ep_find() till we release the mutex.
 */
epi = ep_find(ep, tf.file, fd);
</code></pre><p>红黑树是一棵二叉树，作为二叉树上的节点，epitem必须提供比较能力，以便可以按大小顺序构建出一棵有序的二叉树。其排序能力是依靠epoll_filefd结构体来完成的，epoll_filefd可以简单理解为需要监听的文件描述字，它对应到二叉树上的节点。</p><p>可以看到这个还是比较好理解的，按照文件的地址大小排序。如果两个相同，就按照文件文件描述字来排序。</p><pre><code>struct epoll_filefd {
	struct file *file; // pointer to the target file struct corresponding to the fd
	int fd; // target file descriptor number
} __packed;

/* Compare RB tree keys */
static inline int ep_cmp_ffd(struct epoll_filefd *p1,
                            struct epoll_filefd *p2)
{
	return (p1-&gt;file &gt; p2-&gt;file ? +1:
		   (p1-&gt;file &lt; p2-&gt;file ? -1 : p1-&gt;fd - p2-&gt;fd));
}
</code></pre><p>在进行完红黑树查找之后，如果发现是一个ADD操作，并且在树中没有找到对应的二叉树节点，就会调用ep_insert进行二叉树节点的增加。</p><pre><code>case EPOLL_CTL_ADD:
    if (!epi) {
        epds.events |= POLLERR | POLLHUP;
        error = ep_insert(ep, &amp;epds, tf.file, fd, full_check);
    } else
        error = -EEXIST;
    if (full_check)
        clear_tfile_check_list();
    break;
</code></pre><h3>ep_insert</h3><p>ep_insert首先判断当前监控的文件值是否超过了/proc/sys/fs/epoll/max_user_watches的预设最大值，如果超过了则直接返回错误。</p><pre><code>user_watches = atomic_long_read(&amp;ep-&gt;user-&gt;epoll_watches);
if (unlikely(user_watches &gt;= max_user_watches))
    return -ENOSPC;
</code></pre><p>接下来是分配资源和初始化动作。</p><pre><code>if (!(epi = kmem_cache_alloc(epi_cache, GFP_KERNEL)))
        return -ENOMEM;

    /* Item initialization follow here ... */
    INIT_LIST_HEAD(&amp;epi-&gt;rdllink);
    INIT_LIST_HEAD(&amp;epi-&gt;fllink);
    INIT_LIST_HEAD(&amp;epi-&gt;pwqlist);
    epi-&gt;ep = ep;
    ep_set_ffd(&amp;epi-&gt;ffd, tfile, fd);
    epi-&gt;event = *event;
    epi-&gt;nwait = 0;
    epi-&gt;next = EP_UNACTIVE_PTR;
</code></pre><p>再接下来的事情非常重要，ep_insert会为加入的每个文件描述字设置回调函数。这个回调函数是通过函数ep_ptable_queue_proc来进行设置的。这个回调函数是干什么的呢？其实，对应的文件描述字上如果有事件发生，就会调用这个函数，比如套接字缓冲区有数据了，就会回调这个函数。这个函数就是ep_poll_callback。这里你会发现，原来内核设计也是充满了事件回调的原理。</p><pre><code>/*
 * This is the callback that is used to add our wait queue to the
 * target file wakeup lists.
 */
static void ep_ptable_queue_proc(struct file *file, wait_queue_head_t *whead,poll_table *pt)
{
    struct epitem *epi = ep_item_from_epqueue(pt);
    struct eppoll_entry *pwq;

    if (epi&gt;nwait &gt;= 0 &amp;&amp; (pwq = kmem_cache_alloc(pwq_cache, GFP_KERNEL))) {
        init_waitqueue_func_entry(&amp;pwq-&gt;wait, ep_poll_callback);
        pwq-&gt;whead = whead;
        pwq-&gt;base = epi;
        if (epi-&gt;event.events &amp; EPOLLEXCLUSIVE)
            add_wait_queue_exclusive(whead, &amp;pwq-&gt;wait);
        else
            add_wait_queue(whead, &amp;pwq-&gt;wait);
        list_add_tail(&amp;pwq-&gt;llink, &amp;epi-&gt;pwqlist);
        epi-&gt;nwait++;
    } else {
        /* We have to signal that an error occurred */
        epi-&gt;nwait = -1;
    }
}
</code></pre><h3>ep_poll_callback</h3><p>ep_poll_callback函数的作用非常重要，它将内核事件真正地和epoll对象联系了起来。它又是怎么实现的呢？</p><p>首先，通过这个文件的wait_queue_entry_t对象找到对应的epitem对象，因为eppoll_entry对象里保存了wait_queue_entry_t，根据wait_queue_entry_t这个对象的地址就可以简单计算出eppoll_entry对象的地址，从而可以获得epitem对象的地址。这部分工作在ep_item_from_wait函数中完成。一旦获得epitem对象，就可以寻迹找到eventpoll实例。</p><pre><code>/*
 * This is the callback that is passed to the wait queue wakeup
 * mechanism. It is called by the stored file descriptors when they
 * have events to report.
 */
static int ep_poll_callback(wait_queue_entry_t *wait, unsigned mode, int sync, void *key)
{
    int pwake = 0;
    unsigned long flags;
    struct epitem *epi = ep_item_from_wait(wait);
    struct eventpoll *ep = epi-&gt;ep;
</code></pre><p>接下来，进行一个加锁操作。</p><pre><code>spin_lock_irqsave(&amp;ep-&gt;lock, flags);
</code></pre><p>下面对发生的事件进行过滤，为什么需要过滤呢？为了性能考虑，ep_insert向对应监控文件注册的是所有的事件，而实际用户侧订阅的事件未必和内核事件对应。比如，用户向内核订阅了一个套接字的可读事件，在某个时刻套接字的可写事件发生时，并不需要向用户空间传递这个事件。</p><pre><code>/*
 * Check the events coming with the callback. At this stage, not
 * every device reports the events in the &quot;key&quot; parameter of the
 * callback. We need to be able to handle both cases here, hence the
 * test for &quot;key&quot; != NULL before the event match test.
 */
if (key &amp;&amp; !((unsigned long) key &amp; epi-&gt;event.events))
    goto out_unlock;
</code></pre><p>接下来，判断是否需要把该事件传递给用户空间。</p><pre><code>if (unlikely(ep-&gt;ovflist != EP_UNACTIVE_PTR)) {
  if (epi-&gt;next == EP_UNACTIVE_PTR) {
      epi-&gt;next = ep-&gt;ovflist;
      ep-&gt;ovflist = epi;
      if (epi-&gt;ws) {
          /*
           * Activate ep-&gt;ws since epi-&gt;ws may get
           * deactivated at any time.
           */
          __pm_stay_awake(ep-&gt;ws);
      }
  }
  goto out_unlock;
}
</code></pre><p>如果需要，而且该事件对应的event_item不在eventpoll对应的已完成队列中，就把它放入该队列，以便将该事件传递给用户空间。</p><pre><code>/* If this file is already in the ready list we exit soon */
if (!ep_is_linked(&amp;epi-&gt;rdllink)) {
    list_add_tail(&amp;epi-&gt;rdllink, &amp;ep-&gt;rdllist);
    ep_pm_stay_awake_rcu(epi);
}
</code></pre><p>我们知道，当我们调用epoll_wait的时候，调用进程被挂起，在内核看来调用进程陷入休眠。如果该epoll实例上对应描述字有事件发生，这个休眠进程应该被唤醒，以便及时处理事件。下面的代码就是起这个作用的，wake_up_locked函数唤醒当前eventpoll上的等待进程。</p><pre><code>/*
 * Wake up ( if active ) both the eventpoll wait list and the -&gt;poll()
 * wait list.
 */
if (waitqueue_active(&amp;ep-&gt;wq)) {
    if ((epi-&gt;event.events &amp; EPOLLEXCLUSIVE) &amp;&amp;
                !((unsigned long)key &amp; POLLFREE)) {
        switch ((unsigned long)key &amp; EPOLLINOUT_BITS) {
        case POLLIN:
            if (epi-&gt;event.events &amp; POLLIN)
                ewake = 1;
            break;
        case POLLOUT:
            if (epi-&gt;event.events &amp; POLLOUT)
                ewake = 1;
            break;
        case 0:
            ewake = 1;
            break;
        }
    }
    wake_up_locked(&amp;ep-&gt;wq);
}
</code></pre><h3>查找epoll实例</h3><p>epoll_wait函数首先进行一系列的检查，例如传入的maxevents应该大于0。</p><pre><code>/* The maximum number of event must be greater than zero */
if (maxevents &lt;= 0 || maxevents &gt; EP_MAX_EVENTS)
    return -EINVAL;

/* Verify that the area passed by the user is writeable */
if (!access_ok(VERIFY_WRITE, events, maxevents * sizeof(struct epoll_event)))
    return -EFAULT;
</code></pre><p>和前面介绍的epoll_ctl一样，通过epoll实例找到对应的匿名文件和描述字，并且进行检查和验证。</p><pre><code>/* Get the &quot;struct file *&quot; for the eventpoll file */
f = fdget(epfd);
if (!f.file)
    return -EBADF;

/*
 * We have to check that the file structure underneath the fd
 * the user passed to us _is_ an eventpoll file.
 */
error = -EINVAL;
if (!is_file_epoll(f.file))
    goto error_fput;
</code></pre><p>还是通过读取epoll实例对应匿名文件的private_data得到eventpoll实例。</p><pre><code>/*
 * At this point it is safe to assume that the &quot;private_data&quot; contains
 * our own data structure.
 */
ep = f.file-&gt;private_data;
</code></pre><p>接下来调用ep_poll来完成对应的事件收集并传递到用户空间。</p><pre><code>/* Time to fish for events ... */
error = ep_poll(ep, events, maxevents, timeout);
</code></pre><h3>ep_poll</h3><p>还记得<a href="https://time.geekbang.org/column/article/143245">第23讲</a>里介绍epoll函数的时候，对应的timeout值可以是大于0，等于0和小于0么？这里ep_poll就分别对timeout不同值的场景进行了处理。如果大于0则产生了一个超时时间，如果等于0则立即检查是否有事件发生。</p><pre><code>*/
static int ep_poll(struct eventpoll *ep, struct epoll_event __user *events,int maxevents, long timeout)
{
int res = 0, eavail, timed_out = 0;
unsigned long flags;
u64 slack = 0;
wait_queue_entry_t wait;
ktime_t expires, *to = NULL;

if (timeout &gt; 0) {
    struct timespec64 end_time = ep_set_mstimeout(timeout);
    slack = select_estimate_accuracy(&amp;end_time);
    to = &amp;expires;
    *to = timespec64_to_ktime(end_time);
} else if (timeout == 0) {
    /*
     * Avoid the unnecessary trip to the wait queue loop, if the
     * caller specified a non blocking operation.
     */
    timed_out = 1;
    spin_lock_irqsave(&amp;ep-&gt;lock, flags);
    goto check_events;
}
</code></pre><p>接下来尝试获得eventpoll上的锁：</p><pre><code>spin_lock_irqsave(&amp;ep-&gt;lock, flags);
</code></pre><p>获得这把锁之后，检查当前是否有事件发生，如果没有，就把当前进程加入到eventpoll的等待队列wq中，这样做的目的是当事件发生时，ep_poll_callback函数可以把该等待进程唤醒。</p><pre><code>if (!ep_events_available(ep)) {
    /*
     * Busy poll timed out.  Drop NAPI ID for now, we can add
     * it back in when we have moved a socket with a valid NAPI
     * ID onto the ready list.
     */
    ep_reset_busy_poll_napi_id(ep);

    /*
     * We don't have any available event to return to the caller.
     * We need to sleep here, and we will be wake up by
     * ep_poll_callback() when events will become available.
     */
    init_waitqueue_entry(&amp;wait, current);
    __add_wait_queue_exclusive(&amp;ep-&gt;wq, &amp;wait);
</code></pre><p>紧接着是一个无限循环, 这个循环中通过调用schedule_hrtimeout_range，将当前进程陷入休眠，CPU时间被调度器调度给其他进程使用，当然，当前进程可能会被唤醒，唤醒的条件包括有以下四种：</p><ol>
<li>当前进程超时；</li>
<li>当前进程收到一个signal信号；</li>
<li>某个描述字上有事件发生；</li>
<li>当前进程被CPU重新调度，进入for循环重新判断，如果没有满足前三个条件，就又重新进入休眠。</li>
</ol><p>对应的1、2、3都会通过break跳出循环，直接返回。</p><pre><code>//这个循环里，当前进程可能会被唤醒，唤醒的途径包括
//1.当前进程超时
//2.当前进行收到一个signal信号
//3.某个描述字上有事件发生
//对应的1.2.3都会通过break跳出循环
//第4个可能是当前进程被CPU重新调度，进入for循环的判断，如果没有满足1.2.3的条件，就又重新进入休眠
for (;;) {
    /*
     * We don't want to sleep if the ep_poll_callback() sends us
     * a wakeup in between. That's why we set the task state
     * to TASK_INTERRUPTIBLE before doing the checks.
     */
    set_current_state(TASK_INTERRUPTIBLE);
    /*
     * Always short-circuit for fatal signals to allow
     * threads to make a timely exit without the chance of
     * finding more events available and fetching
     * repeatedly.
     */
    if (fatal_signal_pending(current)) {
        res = -EINTR;
        break;
    }
    if (ep_events_available(ep) || timed_out)
        break;
    if (signal_pending(current)) {
        res = -EINTR;
        break;
    }

    spin_unlock_irqrestore(&amp;ep-&gt;lock, flags);

    //通过调用schedule_hrtimeout_range，当前进程进入休眠，CPU时间被调度器调度给其他进程使用
    if (!schedule_hrtimeout_range(to, slack, HRTIMER_MODE_ABS))
        timed_out = 1;

    spin_lock_irqsave(&amp;ep-&gt;lock, flags);
}
</code></pre><p>如果进程从休眠中返回，则将当前进程从eventpoll的等待队列中删除，并且设置当前进程为TASK_RUNNING状态。</p><pre><code>//从休眠中结束，将当前进程从wait队列中删除，设置状态为TASK_RUNNING，接下来进入check_events，来判断是否是有事件发生
    __remove_wait_queue(&amp;ep-&gt;wq, &amp;wait);
    __set_current_state(TASK_RUNNING);
</code></pre><p>最后，调用ep_send_events将事件拷贝到用户空间。</p><pre><code>//ep_send_events将事件拷贝到用户空间
/*
 * Try to transfer events to user space. In case we get 0 events and
 * there's still timeout left over, we go trying again in search of
 * more luck.
 */
if (!res &amp;&amp; eavail &amp;&amp;
    !(res = ep_send_events(ep, events, maxevents)) &amp;&amp; !timed_out)
    goto fetch_events;


return res;
</code></pre><h3>ep_send_events</h3><p>ep_send_events这个函数会将ep_send_events_proc作为回调函数并调用ep_scan_ready_list函数，ep_scan_ready_list函数调用ep_send_events_proc对每个已经就绪的事件循环处理。</p><p>ep_send_events_proc循环处理就绪事件时，会再次调用每个文件描述符的poll方法，以便确定确实有事件发生。为什么这样做呢？这是为了确定注册的事件在这个时刻还是有效的。</p><p>可以看到，尽管ep_send_events_proc已经尽可能的考虑周全，使得用户空间获得的事件通知都是真实有效的，但还是有一定的概率，当ep_send_events_proc再次调用文件上的poll函数之后，用户空间获得的事件通知已经不再有效，这可能是用户空间已经处理掉了，或者其他什么情形。还记得<a href="https://time.geekbang.org/column/article/141573">第22讲</a>吗，在这种情况下，如果套接字不是非阻塞的，整个进程将会被阻塞，这也是为什么将非阻塞套接字配合epoll使用作为最佳实践的原因。</p><p>在进行简单的事件掩码校验之后，ep_send_events_proc将事件结构体拷贝到用户空间需要的数据结构中。这是通过__put_user方法完成的。</p><pre><code>//这里对一个fd再次进行poll操作，以确认事件
revents = ep_item_poll(epi, &amp;pt);

/*
 * If the event mask intersect the caller-requested one,
 * deliver the event to userspace. Again, ep_scan_ready_list()
 * is holding &quot;mtx&quot;, so no operations coming from userspace
 * can change the item.
 */
if (revents) {
    if (__put_user(revents, &amp;uevent-&gt;events) ||
        __put_user(epi-&gt;event.data, &amp;uevent-&gt;data)) {
        list_add(&amp;epi-&gt;rdllink, head);
        ep_pm_stay_awake(epi);
        return eventcnt ? eventcnt : -EFAULT;
    }
    eventcnt++;
    uevent++;
</code></pre><h2>Level-triggered VS Edge-triggered</h2><p>在<a href="https://time.geekbang.org/column/article/143245">前面的</a><a href="https://time.geekbang.org/column/article/143245">文章</a>里，我们一直都在强调level-triggered和edge-triggered之间的区别。</p><p>从实现角度来看其实非常简单，在ep_send_events_proc函数的最后，针对level-triggered情况，当前的epoll_item对象被重新加到eventpoll的就绪列表中，这样在下一次epoll_wait调用时，这些epoll_item对象就会被重新处理。</p><p>在前面我们提到，在最终拷贝到用户空间有效事件列表中之前，会调用对应文件的poll方法，以确定这个事件是不是依然有效。所以，如果用户空间程序已经处理掉该事件，就不会被再次通知；如果没有处理，意味着该事件依然有效，就会被再次通知。</p><pre><code>//这里是Level-triggered的处理，可以看到，在Level-triggered的情况下，这个事件被重新加回到ready list里面
//这样，下一轮epoll_wait的时候，这个事件会被重新check
else if (!(epi-&gt;event.events &amp; EPOLLET)) {
    /*
     * If this file has been added with Level
     * Trigger mode, we need to insert back inside
     * the ready list, so that the next call to
     * epoll_wait() will check again the events
     * availability. At this point, no one can insert
     * into ep-&gt;rdllist besides us. The epoll_ctl()
     * callers are locked out by
     * ep_scan_ready_list() holding &quot;mtx&quot; and the
     * poll callback will queue them in ep-&gt;ovflist.
     */
    list_add_tail(&amp;epi-&gt;rdllink, &amp;ep-&gt;rdllist);
    ep_pm_stay_awake(epi);
}
</code></pre><h2>epoll VS poll/select</h2><p>最后，我们从实现角度来说明一下为什么epoll的效率要远远高于poll/select。</p><p>首先，poll/select先将要监听的fd从用户空间拷贝到内核空间, 然后在内核空间里面进行处理之后，再拷贝给用户空间。这里就涉及到内核空间申请内存，释放内存等等过程，这在大量fd情况下，是非常耗时的。而epoll维护了一个红黑树，通过对这棵黑红树进行操作，可以避免大量的内存申请和释放的操作，而且查找速度非常快。</p><p>下面的代码就是poll/select在内核空间申请内存的展示。可以看到select 是先尝试申请栈上资源, 如果需要监听的fd比较多, 就会去申请堆空间的资源。</p><pre><code>int core_sys_select(int n, fd_set __user *inp, fd_set __user *outp,
               fd_set __user *exp, struct timespec64 *end_time)
{
    fd_set_bits fds;
    void *bits;
    int ret, max_fds;
    size_t size, alloc_size;
    struct fdtable *fdt;
    /* Allocate small arguments on the stack to save memory and be faster */
    long stack_fds[SELECT_STACK_ALLOC/sizeof(long)];

    ret = -EINVAL;
    if (n &lt; 0)
        goto out_nofds;

    /* max_fds can increase, so grab it once to avoid race */
    rcu_read_lock();
    fdt = files_fdtable(current-&gt;files);
    max_fds = fdt-&gt;max_fds;
    rcu_read_unlock();
    if (n &gt; max_fds)
        n = max_fds;

    /*
     * We need 6 bitmaps (in/out/ex for both incoming and outgoing),
     * since we used fdset we need to allocate memory in units of
     * long-words. 
     */
    size = FDS_BYTES(n);
    bits = stack_fds;
    if (size &gt; sizeof(stack_fds) / 6) {
        /* Not enough space in on-stack array; must use kmalloc */
        ret = -ENOMEM;
        if (size &gt; (SIZE_MAX / 6))
            goto out_nofds;


        alloc_size = 6 * size;
        bits = kvmalloc(alloc_size, GFP_KERNEL);
        if (!bits)
            goto out_nofds;
    }
    fds.in      = bits;
    fds.out     = bits +   size;
    fds.ex      = bits + 2*size;
    fds.res_in  = bits + 3*size;
    fds.res_out = bits + 4*size;
    fds.res_ex  = bits + 5*size;
    ...
</code></pre><p>第二，select/poll从休眠中被唤醒时，如果监听多个fd，只要其中有一个fd有事件发生，内核就会遍历内部的list去检查到底是哪一个事件到达，并没有像epoll一样, 通过fd直接关联eventpoll对象，快速地把fd直接加入到eventpoll的就绪列表中。</p><pre><code>static int do_select(int n, fd_set_bits *fds, struct timespec64 *end_time)
{
    ...
    retval = 0;
    for (;;) {
        unsigned long *rinp, *routp, *rexp, *inp, *outp, *exp;
        bool can_busy_loop = false;

        inp = fds-&gt;in; outp = fds-&gt;out; exp = fds-&gt;ex;
        rinp = fds-&gt;res_in; routp = fds-&gt;res_out; rexp = fds-&gt;res_ex;

        for (i = 0; i &lt; n; ++rinp, ++routp, ++rexp) {
            unsigned long in, out, ex, all_bits, bit = 1, mask, j;
            unsigned long res_in = 0, res_out = 0, res_ex = 0;

            in = *inp++; out = *outp++; ex = *exp++;
            all_bits = in | out | ex;
            if (all_bits == 0) {
                i += BITS_PER_LONG;
                continue;
            }
        
        if (!poll_schedule_timeout(&amp;table, TASK_INTERRUPTIBLE,
                   to, slack))
        timed_out = 1;
...
</code></pre><h2>总结</h2><p>在这次答疑中，我希望通过深度分析epoll的源码实现，帮你理解epoll的实现原理。</p><p>epoll维护了一棵红黑树来跟踪所有待检测的文件描述字，黑红树的使用减少了内核和用户空间大量的数据拷贝和内存分配，大大提高了性能。</p><p>同时，epoll维护了一个链表来记录就绪事件，内核在每个文件有事件发生时将自己登记到这个就绪事件列表中，通过内核自身的文件file-eventpoll之间的回调和唤醒机制，减少了对内核描述字的遍历，大大加速了事件通知和检测的效率，这也为level-triggered和edge-triggered的实现带来了便利。</p><p>通过对比poll/select的实现，我们发现epoll确实克服了poll/select的种种弊端，不愧是Linux下高性能网络编程的皇冠。我们应该感谢Linux社区的大神们设计了这么强大的事件分发机制，让我们在Linux下可以享受高性能网络服务器带来的种种技术红利。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/9b/88/34c171f1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>herongwei</span>
  </div>
  <div class="_2_QraFYR_0">这篇文章写得非常棒！之前学 epoll，只会流于表面，也很少去深度剖析底层的数据结构，多读几遍!<br>另外分享一个大佬同学 epoll 内核源码详解发的帖子：https:&#47;&#47;www.nowcoder.com&#47;discuss&#47;26226</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞分享。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-26 15:57:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/IPdZZXuHVMibwfZWmm7NiawzeEFGsaRoWjhuN99iaoj5amcRkiaOePo6rH1KJ3jictmNlic4OibkF4I20vOGfwDqcBxfA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>鱼向北游</span>
  </div>
  <div class="_2_QraFYR_0">select这种是把等待队列和就绪队列混在一起，epoll根据这两种队列的特性用两种数据结构把这两个队列分开，果然在程序世界没有解决不了的事情，如果有，就加一个中间层</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你这个理解倒是比较有趣，程序是伟大的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-18 09:50:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/51/29/24739c58.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>凉人。</span>
  </div>
  <div class="_2_QraFYR_0">感觉这一章是最难以理解，如果多一些结构图，流程图，会好很多。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，好建议，记下了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-11 00:40:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/86QEF74Mhc6ECbBBMr62hVz0ezOicI2Kbv8QBA7qR7KepeoDib9W6KLxxMPuQ24JGusvjC03NNr8uj8GyK0DxKiaw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>HerofH</span>
  </div>
  <div class="_2_QraFYR_0">首先感谢老师的精彩分析。<br><br>然后说下我的个人理解：epoll之所以效率比select高，原因在于select要做的事情太多，每次调用select不仅需要将描述符集添加到监听队列中，还要负责监听事件发生后的处理，以及select最后的清理工作等等...<br>而epoll则把这么多事情分到了epoll_ctl和epoll_wait去处理，调用epoll_ctl可以开启事件的监听工作，epoll_wait则可以完成对已激活的事件的处理工作，最后close(epfd)完成epoll的清理工作。<br>而select和epoll_wait是循环中反复调用的，epoll_wait做的事情比select少多了，因此从这个角度来说epoll效率会比select效率高。<br>除此之外，个人感觉epoll还有一个妙处就是就绪链表，当监听的事件发生后，相应的epitem会自动在监听回调中将其添加到就绪链表中，而对于select来说，则需要不停对所有监听的描述符进行遍历，来检查它们的状态。<br><br>不知道理解是否正确，敬请指正。<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 👍</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-15 21:48:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/85/d3/abb7bfe3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>fackgc17</span>
  </div>
  <div class="_2_QraFYR_0">可以提供一下分析用的 kernel 版本吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: linux 4.14.4</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-26 10:43:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/e9/86/d34800a4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>heyman</span>
  </div>
  <div class="_2_QraFYR_0">请问一下，为什么要用红黑树？是因为要排序吗？排序的意义又在哪里？确保查找、插入和删除的高效还不够吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 就是要确保查找、插入和删除的高效啊，要知道，10K以上的连接时，这个对性能要求非常苛刻，一个高效的数据结构对于完成上述操作是非常关键的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-18 12:59:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/67/f4/9a1feb59.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>钱</span>
  </div>
  <div class="_2_QraFYR_0">功力不够，读起来有些费劲，好似拿到了九阴真经，不过需要黄姑娘翻译一下！<br>如果能来个图，就好了😁</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 待我找黄姑娘来:)</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-01 19:00:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/22/b1/54/6d663b95.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>瓜牛</span>
  </div>
  <div class="_2_QraFYR_0">还是没明白为啥epoll有红黑树之后就不用在用户空间和内核空间之间拷贝了，或者说poll&#47;select为啥要在用户空间和内核空间之间拷贝？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 简单的说，poll&#47;select需要每次循环判断出有事件的fd，这份信息是一个全量的拷贝；而epoll则直接拿到事件发生后的信息。一个是ALL，一个是局部变化的部分，你说少没少拷贝？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-05 15:05:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83errIIarFicghpKamvkUaJmGdIV488iaOUyUqcTwbQ6IeRS40ZFfIOfb369fgleydAT8pkucHuj2x45A/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>xupeng1644</span>
  </div>
  <div class="_2_QraFYR_0">老师 在Level-triggered VS Edge-triggered小节中给出了Level-triggered的实现机制，可以再给下Edge-triggered的吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: Edge-triggered不需要特殊处理，就是有事件通知一次结束，这里Level-triggered还需要把没有完结的事件在拷贝一次通知给应用程序，所以需要特殊处理下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-28 10:12:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/4f/a5/71358d7b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>J.M.Liu</span>
  </div>
  <div class="_2_QraFYR_0">老师，我请教两个问题：<br>1.ep_send_events_proc这个函数在把events列表拷贝到用户空间前会调用ep_item_poll函数，已确定对应的fd上的事件依旧有效。那ep_item_poll是根据什么来确定事件的有效性呢？<br>2.在ep_send_events_proc处理level-triggered的时候，有这么一段话“At this point, no one can insert into ep-&gt;rdllist besides us. The epoll_ctl()  callers are locked out by ep_scan_ready_list() holding &quot;mtx&quot; and the  poll callback will queue them in ep-&gt;ovflist.”意思是说epoll_ctl（）的调用者也被锁在外面了。这个锁是说在ep_send_events_proc还没处理完的时候，epoll_ctl()无法操纵rdllist，但是之后是可以的，以实现再次注册感兴趣的时间。是这样吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 看得好自信，汗，我试着回答哈：<br><br>1.根据套接字文件上的poll接口，套接字文件上记录了它有效的事件：<br>static inline unsigned int ep_item_poll(struct epitem *epi, poll_table *pt)<br>{<br>	pt-&gt;_key = epi-&gt;event.events;<br><br>	return epi-&gt;ffd.file-&gt;f_op-&gt;poll(epi-&gt;ffd.file, pt) &amp; epi-&gt;event.events;<br>}<br><br>2.是的。<br><br>static int ep_scan_ready_list(struct eventpoll *ep,<br>			      int (*sproc)(struct eventpoll *,<br>					   struct list_head *, void *),<br>			      void *priv, int depth, bool ep_locked)<br>{<br>	int error, pwake = 0;<br>	unsigned long flags;<br>	struct epitem *epi, *nepi;<br>	LIST_HEAD(txlist);<br><br>	&#47;*<br>	 * We need to lock this because we could be hit by<br>	 * eventpoll_release_file() and epoll_ctl().<br>	 *&#47;<br><br>	if (!ep_locked)<br>		mutex_lock_nested(&amp;ep-&gt;mtx, depth);<br>        ...<br>}</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-29 23:31:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/f6/e3/e4bcd69e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>沉淀的梦想</span>
  </div>
  <div class="_2_QraFYR_0">缺乏C语言和linux内核基础的人读起这些源码来相当吃力，虽然老师讲得很好</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 主要是理解整体的概念和设计，细节不用理解太深。可以读理解几遍，肯定会有收获的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-19 16:12:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/06/7e/735968e2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>西门吹牛</span>
  </div>
  <div class="_2_QraFYR_0">当内核监测到有就绪事件后，将对应的fd 加入就绪 队列，这里其实还会涉及到将有事件的fd拷贝到用户空间，可不可以让用户空间和内核空间采用共享内存的方式，避免拷贝呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这样内核空间的数据不就很不安全了？很容易陷入crash状态。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-01-27 08:42:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83eqf54z1ZmqQY1kmJ6t1HAnrqMM3j6WKf0oDeVLhtnA2ZUKY6AX9MK6RjvcO8SiczXy3uU0IzBQ3tpw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_68d3d2</span>
  </div>
  <div class="_2_QraFYR_0"> epi&gt;nwait &gt;= 0请问这行代码什么意思</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对应的文件描述字上有poll的动作，正在等待事件发生。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-08 09:06:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>haozhang</span>
  </div>
  <div class="_2_QraFYR_0">老师   进程阻塞的形式是什么呢   是for死循环吗     还是加入到等待队列呢，我看for循环前面不是加入到等待队列了吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 进程阻塞其实就是把CPU时间调度给了其他的进程，因为得不到CPU时间，所以当前进程的代码没有办法再执行下去，表现的行为就是当前程序执行&quot;休眠&quot;在那里了。<br><br>这里的等待队列是LInux内核把等待调度的进行按照自己内部的数据结构，形成了一个队列。按照LInux内核任务调度算法进行调度，来唤醒对应的&quot;休眠&quot;进程。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-06 13:09:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/ce/5d/5297717a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>初见</span>
  </div>
  <div class="_2_QraFYR_0">老师，我之前面试被问到过说，epoll 更适合连接很多，但活跃的连接较少的情况<br><br>那么，连接很多，活跃连接也很多的情况下，用什么方案呢？  堆机器嘛</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不知道为什么会有这样的说法，我觉得这个说法不正确，epoll适合活跃连接很多的场景。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-25 00:10:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/d7/49/c2426b51.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>影帝</span>
  </div>
  <div class="_2_QraFYR_0">我发现看留言学到的更多。🤓</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这就是social的力量 :)</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-20 19:37:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/ce/4e/17fd7195.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>TM</span>
  </div>
  <div class="_2_QraFYR_0">hi 老师您好，有个问题想咨询下。把 redis 的 backlog 设置为 1，然后在 redis 里 debug sleep 50，然后发起两个请求，一个成功连接，另一个会出 『opration timeout』 ，而不是 connect timeout，然后大概是 26 s ，反复试了几次、都是26s左右的时间。很奇怪这个报错是内核爆出来的吗？为什么是26s这个时间呢？扩展是 phpredis，php 底层 socket 超时是 60s。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你好像问过一次了吧，这个我认为不是内核报出来的，我建议你debug一下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-18 09:14:53</div>
  </div>
</div>
</div>
</li>
</ul>