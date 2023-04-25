<audio title="40 _ IPC（上）：不同项目组之间抢资源，如何协调？" src="https://static001.geekbang.org/resource/audio/52/52/5292ca2ad2cdc2f2d54796e5c3d55b52.mp3" controls="controls"></audio> 
<p>我们前面讲了，如果项目组之间需要紧密合作，那就需要共享内存，这样就像把两个项目组放在一个会议室一起沟通，会非常高效。这一节，我们就来详细讲讲这个进程之间共享内存的机制。</p><p>有了这个机制，两个进程可以像访问自己内存中的变量一样，访问共享内存的变量。但是同时问题也来了，当两个进程共享内存了，就会存在同时读写的问题，就需要对于共享的内存进行保护，就需要信号量这样的同步协调机制。这些也都是我们这节需要探讨的问题。下面我们就一一来看。</p><p>共享内存和信号量也是System V系列的进程间通信机制，所以很多地方和我们讲过的消息队列有点儿像。为了将共享内存和信号量结合起来使用，我这里定义了一个share.h头文件，里面放了一些共享内存和信号量在每个进程都需要的函数。</p><pre><code>#include &lt;stdio.h&gt;
#include &lt;stdlib.h&gt;
#include &lt;sys/ipc.h&gt;
#include &lt;sys/shm.h&gt;
#include &lt;sys/types.h&gt;
#include &lt;sys/sem.h&gt;
#include &lt;string.h&gt;

#define MAX_NUM 128

struct shm_data {
  int data[MAX_NUM];
  int datalength;
};

union semun {
  int val; 
  struct semid_ds *buf; 
  unsigned short int *array; 
  struct seminfo *__buf; 
}; 

int get_shmid(){
  int shmid;
  key_t key;
  
  if((key = ftok(&quot;/root/sharememory/sharememorykey&quot;, 1024)) &lt; 0){
      perror(&quot;ftok error&quot;);
          return -1;
  }
  
  shmid = shmget(key, sizeof(struct shm_data), IPC_CREAT|0777);
  return shmid;
}

int get_semaphoreid(){
  int semid;
  key_t key;
  
  if((key = ftok(&quot;/root/sharememory/semaphorekey&quot;, 1024)) &lt; 0){
      perror(&quot;ftok error&quot;);
          return -1;
  }
  
  semid = semget(key, 1, IPC_CREAT|0777);
  return semid;
}

int semaphore_init (int semid) {
  union semun argument; 
  unsigned short values[1]; 
  values[0] = 1; 
  argument.array = values; 
  return semctl (semid, 0, SETALL, argument); 
}

int semaphore_p (int semid) {
  struct sembuf operations[1]; 
  operations[0].sem_num = 0; 
  operations[0].sem_op = -1; 
  operations[0].sem_flg = SEM_UNDO; 
  return semop (semid, operations, 1); 
}

int semaphore_v (int semid) {
  struct sembuf operations[1]; 
  operations[0].sem_num = 0; 
  operations[0].sem_op = 1; 
  operations[0].sem_flg = SEM_UNDO; 
  return semop (semid, operations, 1); 
} 
</code></pre><h2>共享内存</h2><p>我们先来看里面对于共享内存的操作。</p><p>首先，创建之前，我们要有一个key来唯一标识这个共享内存。这个key可以根据文件系统上的一个文件的inode随机生成。</p><p>然后，我们需要创建一个共享内存，就像创建一个消息队列差不多，都是使用xxxget来创建。其中，创建共享内存使用的是下面这个函数：</p><pre><code>int shmget(key_t key, size_t size, int shmflag);
</code></pre><p>其中，key就是前面生成的那个key，shmflag如果为IPC_CREAT，就表示新创建，还可以指定读写权限0777。</p><!-- [[[read_end]]] --><p>对于共享内存，需要指定一个大小size，这个一般要申请多大呢？一个最佳实践是，我们将多个进程需要共享的数据放在一个struct里面，然后这里的size就应该是这个struct的大小。这样每一个进程得到这块内存后，只要强制将类型转换为这个struct类型，就能够访问里面的共享数据了。</p><p>在这里，我们定义了一个struct shm_data结构。这里面有两个成员，一个是一个整型的数组，一个是数组中元素的个数。</p><p>生成了共享内存以后，接下来就是将这个共享内存映射到进程的虚拟地址空间中。我们使用下面这个函数来进行操作。</p><pre><code>void *shmat(int  shm_id, const  void *addr, int shmflg);
</code></pre><p>这里面的shm_id，就是上面创建的共享内存的id，addr就是指定映射在某个地方。如果不指定，则内核会自动选择一个地址，作为返回值返回。得到了返回地址以后，我们需要将指针强制类型转换为struct shm_data结构，就可以使用这个指针设置data和datalength了。</p><p>当共享内存使用完毕，我们可以通过shmdt解除它到虚拟内存的映射。</p><pre><code>int shmdt(const  void *shmaddr)；
</code></pre><h2>信号量</h2><p>看完了共享内存，接下来我们再来看信号量。信号量以集合的形式存在的。</p><p>首先，创建之前，我们同样需要有一个key，来唯一标识这个信号量集合。这个key同样可以根据文件系统上的一个文件的inode随机生成。</p><p>然后，我们需要创建一个信号量集合，同样也是使用xxxget来创建，其中创建信号量集合使用的是下面这个函数。</p><pre><code>int semget(key_t key, int nsems, int semflg);
</code></pre><p>这里面的key，就是前面生成的那个key，shmflag如果为IPC_CREAT，就表示新创建，还可以指定读写权限0777。</p><p>这里，nsems表示这个信号量集合里面有几个信号量，最简单的情况下，我们设置为1。</p><p>信号量往往代表某种资源的数量，如果用信号量做互斥，那往往将信号量设置为1。这就是上面代码中semaphore_init函数的作用，这里面调用semctl函数，将这个信号量集合的中的第0个信号量，也即唯一的这个信号量设置为1。</p><p>对于信号量，往往要定义两种操作，P操作和V操作。对应上面代码中semaphore_p函数和semaphore_v函数，semaphore_p会调用semop函数将信号量的值减一，表示申请占用一个资源，当发现当前没有资源的时候，进入等待。semaphore_v会调用semop函数将信号量的值加一，表示释放一个资源，释放之后，就允许等待中的其他进程占用这个资源。</p><p>我们可以用这个信号量，来保护共享内存中的struct shm_data，使得同时只有一个进程可以操作这个结构。</p><p>你是否记得咱们讲线程同步机制的时候，构建了一个老板分配活的场景。这里我们同样构建一个场景，分为producer.c和consumer.c，其中producer也即生产者，负责往struct shm_data塞入数据，而consumer.c负责处理struct shm_data中的数据。</p><p>下面我们来看producer.c的代码。</p><pre><code>#include &quot;share.h&quot;

int main() {
  void *shm = NULL;
  struct shm_data *shared = NULL;
  int shmid = get_shmid();
  int semid = get_semaphoreid();
  int i;
  
  shm = shmat(shmid, (void*)0, 0);
  if(shm == (void*)-1){
    exit(0);
  }
  shared = (struct shm_data*)shm;
  memset(shared, 0, sizeof(struct shm_data));
  semaphore_init(semid);
  while(1){
    semaphore_p(semid);
    if(shared-&gt;datalength &gt; 0){
      semaphore_v(semid);
      sleep(1);
    } else {
      printf(&quot;how many integers to caculate : &quot;);
      scanf(&quot;%d&quot;,&amp;shared-&gt;datalength);
      if(shared-&gt;datalength &gt; MAX_NUM){
        perror(&quot;too many integers.&quot;);
        shared-&gt;datalength = 0;
        semaphore_v(semid);
        exit(1);
      }
      for(i=0;i&lt;shared-&gt;datalength;i++){
        printf(&quot;Input the %d integer : &quot;, i);
        scanf(&quot;%d&quot;,&amp;shared-&gt;data[i]);
      }
      semaphore_v(semid);
    }
  }
}
</code></pre><p>在这里面，get_shmid创建了共享内存，get_semaphoreid创建了信号量集合，然后shmat将共享内存映射到了虚拟地址空间的shm指针指向的位置，然后通过强制类型转换，shared的指针指向放在共享内存里面的struct shm_data结构，然后初始化为0。semaphore_init将信号量进行了初始化。</p><p>接着，producer进入了一个无限循环。在这个循环里面，我们先通过semaphore_p申请访问共享内存的权利，如果发现datalength大于零，说明共享内存里面的数据没有被处理过，于是semaphore_v释放权利，先睡一会儿，睡醒了再看。如果发现datalength等于0，说明共享内存里面的数据被处理完了，于是开始往里面放数据。让用户输入多少个数，然后每个数是什么，都放在struct shm_data结构中，然后semaphore_v释放权利，等待其他的进程将这些数拿去处理。</p><p>我们再来看consumer的代码。</p><pre><code>#include &quot;share.h&quot;

int main() {
  void *shm = NULL;
  struct shm_data *shared = NULL;
  int shmid = get_shmid();
  int semid = get_semaphoreid();
  int i;
  
  shm = shmat(shmid, (void*)0, 0);
  if(shm == (void*)-1){
    exit(0);
  }
  shared = (struct shm_data*)shm;
  while(1){
    semaphore_p(semid);
    if(shared-&gt;datalength &gt; 0){
      int sum = 0;
      for(i=0;i&lt;shared-&gt;datalength-1;i++){
        printf(&quot;%d+&quot;,shared-&gt;data[i]);
        sum += shared-&gt;data[i];
      }
      printf(&quot;%d&quot;,shared-&gt;data[shared-&gt;datalength-1]);
      sum += shared-&gt;data[shared-&gt;datalength-1];
      printf(&quot;=%d\n&quot;,sum);
      memset(shared, 0, sizeof(struct shm_data));
      semaphore_v(semid);
    } else {
      semaphore_v(semid);
      printf(&quot;no tasks, waiting.\n&quot;);
      sleep(1);
    }
  }
}
</code></pre><p>在这里面，get_shmid获得producer创建的共享内存，get_semaphoreid获得producer创建的信号量集合，然后shmat将共享内存映射到了虚拟地址空间的shm指针指向的位置，然后通过强制类型转换，shared的指针指向放在共享内存里面的struct shm_data结构。</p><p>接着，consumer进入了一个无限循环，在这个循环里面，我们先通过semaphore_p申请访问共享内存的权利，如果发现datalength等于0，就说明没什么活干，需要等待。如果发现datalength大于0，就说明有活干，于是将datalength个整型数字从data数组中取出来求和。最后将struct shm_data清空为0，表示任务处理完毕，通过semaphore_v释放权利。</p><p>通过程序创建的共享内存和信号量集合，我们可以通过命令ipcs查看。当然，我们也可以通过ipcrm进行删除。</p><pre><code># ipcs
------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages    
------ Shared Memory Segments --------
key        shmid      owner      perms      bytes      nattch     status      
0x00016988 32768      root       777        516        0             
------ Semaphore Arrays --------
key        semid      owner      perms      nsems     
0x00016989 32768      root       777        1 
</code></pre><p>下面我们来运行一下producer和consumer，可以得到下面的结果：</p><pre><code># ./producer 
how many integers to caculate : 2
Input the 0 integer : 3
Input the 1 integer : 4
how many integers to caculate : 4
Input the 0 integer : 3
Input the 1 integer : 4
Input the 2 integer : 5
Input the 3 integer : 6
how many integers to caculate : 7
Input the 0 integer : 9
Input the 1 integer : 8
Input the 2 integer : 7
Input the 3 integer : 6
Input the 4 integer : 5
Input the 5 integer : 4
Input the 6 integer : 3

# ./consumer 
3+4=7
3+4+5+6=18
9+8+7+6+5+4+3=42
</code></pre><h2>总结时刻</h2><p>这一节的内容差不多了，我们来总结一下。共享内存和信号量的配合机制，如下图所示：</p><ul>
<li>无论是共享内存还是信号量，创建与初始化都遵循同样流程，通过ftok得到key，通过xxxget创建对象并生成id；</li>
<li>生产者和消费者都通过shmat将共享内存映射到各自的内存空间，在不同的进程里面映射的位置不同；</li>
<li>为了访问共享内存，需要信号量进行保护，信号量需要通过semctl初始化为某个值；</li>
<li>接下来生产者和消费者要通过semop(-1)来竞争信号量，如果生产者抢到信号量则写入，然后通过semop(+1)释放信号量，如果消费者抢到信号量则读出，然后通过semop(+1)释放信号量；</li>
<li>共享内存使用完毕，可以通过shmdt来解除映射。</li>
</ul><p><img src="https://static001.geekbang.org/resource/image/46/0b/469552bffe601d594c432d4fad97490b.png?wh=2383*2206" alt=""></p><h2>课堂练习</h2><p>信号量大于1的情况下，应该如何使用？你可以试着构建一个场景。</p><p>欢迎留言和我分享你的疑惑和见解 ，也欢迎可以收藏本节内容，反复研读。你也可以把今天的内容分享给你的朋友，和他一起学习和进步。</p><p><img src="https://static001.geekbang.org/resource/image/8c/37/8c0a95fa07a8b9a1abfd394479bdd637.jpg?wh=1110*659" alt=""></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/1c/2e/93812642.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Amark</span>
  </div>
  <div class="_2_QraFYR_0">请教一个问题，CPU调度是以进程为单位的吗，还是以线程?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 以task，在内核里面，进程和线程都是task</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-29 11:29:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/5e/96/a03175bc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>莫名</span>
  </div>
  <div class="_2_QraFYR_0">System V IPC具有很好的移植性，但缺点也比较明显，不能接口自成一套，难以使用现有的fd操作函数。建议对比讲一下比较流行的POSIX IPC。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 赞</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-28 23:43:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/1e/3a/5b21c01c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>nightmare</span>
  </div>
  <div class="_2_QraFYR_0">信号量大于1的情况，可以让进程不操作共享变量，比如操作不同的变量，比如对一批数据做操作，然后做完之后给消费端读取</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-30 22:37:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/0b/d4/39763233.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Tianz</span>
  </div>
  <div class="_2_QraFYR_0">超哥，现在是不是推荐使用 POSIX 系列的 IPC 呢？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-28 09:58:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/4f/6a/0a6b437e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>当你的世界里有风吹过</span>
  </div>
  <div class="_2_QraFYR_0">信号量大于1，可以用于限流。如线程或进程的个数，访问请求的个数等。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-16 21:26:38</div>
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
  <div class="_2_QraFYR_0">信号量和锁的区别就是 信号量可以控制资源数量（&gt;1）, 而锁是 互斥排他的？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-12-06 21:52:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/12/ce/a8c8b5e8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Jason</span>
  </div>
  <div class="_2_QraFYR_0">这篇看的很明白，嘿嘿。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-27 13:44:12</div>
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
  <div class="_2_QraFYR_0">信号量大于 1 的情况下，应该如何使用？<br>可以让多个进程同时访问一个共享内存。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个不行，大于1的时候，不能排他，但是可以控制资源</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-28 16:06:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTI4Gr57Aia5McdvyQco8hKpaibeeYUhQcMtaFhNtHESSF7MPq5OdQBQpCBYicl7Libt6MjWKNJvmGwODA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_93a721</span>
  </div>
  <div class="_2_QraFYR_0">如果大于1时，应该使用三个信号量，一个表示任务这种资源，一个表示空间这种资源，第三个将其置为1用于互斥访问。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-29 15:26:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKJrOl63enWXCRxN0SoucliclBme0qrRb19ATrWIOIvibKIz8UAuVgicBMibIVUznerHnjotI4dm6ibODA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Helios</span>
  </div>
  <div class="_2_QraFYR_0">信号量大于1的时候应该就不能控制写操作了。应该是控制读操作的进程数量。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-15 10:46:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/12/ce/a8c8b5e8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Jason</span>
  </div>
  <div class="_2_QraFYR_0">老师好，ftok提示我的机器里没有“&#47;root&#47;sharememory&#47;semaphorekey”这个文件，我随便新建一个文件可以吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，创建一个就行</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-28 15:10:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/50/39/14adb0f0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>trllllllll</span>
  </div>
  <div class="_2_QraFYR_0">老师，share.h 里面 include 了两次 ipc.h。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢指正</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-07 19:44:37</div>
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
  <div class="_2_QraFYR_0">如果线程是掉用的到基本单位，那么进程的共享资源呢?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 内存，变量，文件，都是共享的呀</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-29 11:37:56</div>
  </div>
</div>
</div>
</li>
</ul>