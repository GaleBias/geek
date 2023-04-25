<audio title="12 _ 容器文件Quota：容器为什么把宿主机的磁盘写满了？" src="https://static001.geekbang.org/resource/audio/fc/c4/fc13b1771af810738b26c6005dc214c4.mp3" controls="controls"></audio> 
<p>你好，我是程远。今天我们聊一聊容器文件Quota。</p><p>上一讲，我们学习了容器文件系统OverlayFS，这个OverlayFS有两层，分别是lowerdir和upperdir。lowerdir里是容器镜像中的文件，对于容器来说是只读的；upperdir存放的是容器对文件系统里的所有改动，它是可读写的。</p><p>从宿主机的角度看，upperdir就是一个目录，如果容器不断往容器文件系统中写入数据，实际上就是往宿主机的磁盘上写数据，这些数据也就存在于宿主机的磁盘目录中。</p><p>当然对于容器来说，如果有大量的写操作是不建议写入容器文件系统的，一般是需要给容器挂载一个volume，用来满足大量的文件读写。</p><p>但是不能避免的是，用户在容器中运行的程序有错误，或者进行了错误的配置。</p><p>比如说，我们把log写在了容器文件系统上，并且没有做log rotation，那么时间一久，就会导致宿主机上的磁盘被写满。这样影响的就不止是容器本身了，而是整个宿主机了。</p><p>那对于这样的问题，我们该怎么解决呢？</p><h2>问题再现</h2><p>我们可以自己先启动一个容器，一起试试不断地往容器文件系统中写入数据，看看是一个什么样的情况。</p><p>用Docker启动一个容器后，我们看到容器的根目录(/)也就是容器文件系统OverlayFS，它的大小是160G，已经使用了100G。其实这个大小也是宿主机上的磁盘空间和使用情况。</p><!-- [[[read_end]]] --><p><img src="https://static001.geekbang.org/resource/image/d3/3e/d32c83e404a81b301fbf8bdfd7a9c23e.png?wh=1264*586" alt=""></p><p>这时候，我们可以回到宿主机上验证一下，就会发现宿主机的根目录(/)的大小也是160G，同样是使用了100G。</p><p><img src="https://static001.geekbang.org/resource/image/b5/0b/b54deb08e46ff1155303581e9d1c8f0b.png?wh=1242*366" alt=""></p><p>好，那现在我们再往容器的根目录里写入10GB的数据。</p><p>这里我们可以看到容器的根目录使用的大小增加了，从刚才的100G变成现在的110G。而多写入的10G大小的数据，对应的是test.log这个文件。</p><p><img src="https://static001.geekbang.org/resource/image/c0/46/c04dcff3aa4773302495113dd2a8d546.png?wh=1704*472" alt=""></p><p>接下来，我们再回到宿主机上，可以看到宿主机上的根目录(/)里使用的大小也是110G了。</p><p><img src="https://static001.geekbang.org/resource/image/15/9d/155d30bc20b72c0678d1948f25cbe29d.png?wh=1220*366" alt=""></p><p>我们还是继续看宿主机，看看OverlayFS里upperdir目录中有什么文件？</p><p>这里我们仍然可以通过/proc/mounts这个路径，找到容器OverlayFS对应的lowerdir和upperdir。因为写入的数据都在upperdir里，我们就只要看upperdir对应的那个目录就行了。果然，里面存放着容器写入的文件test.log，它的大小是10GB。</p><p><img src="https://static001.geekbang.org/resource/image/4d/7d/4d32334dc0f1ba69881c686037f6577d.png?wh=1920*461" alt=""></p><p>通过这个例子，我们已经验证了在容器中对于OverlayFS中写入数据，<strong>其实就是往宿主机的一个目录（upperdir）里写数据。</strong>我们现在已经写了10GB的数据，如果继续在容器中写入数据，结果估计你也知道了，就是会写满宿主机的磁盘。</p><p>那遇到这种情况，我们该怎么办呢？</p><h2>知识详解</h2><p>容器写自己的OverlayFS根目录，结果把宿主机的磁盘写满了。发生这个问题，我们首先就会想到需要对容器做限制，限制它写入自己OverlayFS的数据量，比如只允许一个容器写100MB的数据。</p><p>不过我们实际查看OverlayFS文件系统的特性，就会发现没有直接限制文件写入量的特性。别担心，在没有现成工具的情况下，我们只要搞懂了原理，就能想出解决办法。</p><p>所以我们再来分析一下OverlayFS，它是通过lowerdir和upperdir两层目录联合挂载来实现的，lowerdir是只读的，数据只会写在upperdir中。</p><p>那我们是不是可以通过限制upperdir目录容量的方式，来限制一个容器OverlayFS根目录的写入数据量呢？</p><p>沿着这个思路继续往下想，因为upperdir在宿主机上也是一个普通的目录，这样就要看<strong>宿主机上的文件系统是否可以支持对一个目录限制容量了。</strong></p><p>对于Linux上最常用的两个文件系统XFS和ext4，它们有一个特性Quota，那我们就以XFS文件系统为例，学习一下这个Quota概念，然后看看这个特性能不能限制一个目录的使用量。</p><h3>XFS  Quota</h3><p>在Linux系统里的XFS文件系统缺省都有Quota的特性，这个特性可以为Linux系统里的一个用户（user），一个用户组（group）或者一个项目（project）来限制它们使用文件系统的额度（quota），也就是限制它们可以写入文件系统的文件总量。</p><p>因为我们的目标是要限制一个目录中总体的写入文件数据量，那么显然给用户和用户组限制文件系统的写入数据量的模式，并不适合我们的这个需求。</p><p>因为同一个用户或者用户组可以操作多个目录，多个用户或者用户组也可以操作同一个目录，这样对一个用户或者用户组的限制，就很难用来限制一个目录。</p><p>那排除了限制用户或用户组的模式，我们再来看看Project模式。Project模式是怎么工作的呢？</p><p>我举一个例子你会更好理解，对Linux熟悉的同学可以一边操作，一边体会一下它的工作方式。不熟悉的同学也没关系，可以重点关注我后面的讲解思路。</p><p>首先我们要使用XFS Quota特性，必须在文件系统挂载的时候加上对应的Quota选项，比如我们目前需要配置Project Quota，那么这个挂载参数就是"pquota"。</p><p>对于根目录来说，<strong>这个参数必须作为一个内核启动的参数"rootflags=pquota"，这样设置就可以保证根目录在启动挂载的时候，带上XFS Quota的特性并且支持Project模式。</strong></p><p>我们可以从/proc/mounts信息里，看看根目录是不是带"prjquota"字段。如果里面有这个字段，就可以确保文件系统已经带上了支持project模式的XFS quota特性。</p><p><img src="https://static001.geekbang.org/resource/image/72/3d/72d653f67717fe047c98fce37156da3d.png?wh=1230*112" alt=""></p><p>下一步，我们还需要给一个指定的目录打上一个Project ID。这个步骤我们可以使用XFS文件系统自带的工具 <a href="https://linux.die.net/man/8/xfs_quota">xfs_quota</a> 来完成，然后执行下面的这个命令就可以了。</p><p>执行命令之前，我先对下面的命令和输出做两点解释，让你理解这个命令的含义。</p><p>第一点，新建的目录/tmp/xfs_prjquota，我们想对它做Quota限制。所以在这里要对它打上一个Project ID。</p><p>第二点，通过xfs_quota这条命令，我们给/tmp/xfs_prjquota打上Project ID值101，这个101是我随便选的一个数字，就是个ID标识，你先有个印象。在后面针对Project进行Quota限制的时候，我们还会用到这个ID。</p><pre><code class="language-shell"># mkdir -p  /tmp/xfs_prjquota
# xfs_quota -x -c 'project -s -p /tmp/xfs_prjquota 101' /
Setting up project 101 (path /tmp/xfs_prjquota)...
Processed 1 (/etc/projects and cmdline) paths for project 101 with recursion depth infinite (-1).
</code></pre><p>最后，我们还是使用xfs_quota命令，对101（我们刚才建立的这个Project ID）做Quota限制。</p><p>你可以执行下面这条命令，里面的"-p bhard=10m 101"就代表限制101这个project ID，限制它的数据块写入量不能超过10MB。</p><pre><code># xfs_quota -x -c 'limit -p bhard=10m 101' /
</code></pre><p>做好限制之后，我们可以尝试往/tmp/xfs_prjquota写数据，看看是否可以超过10MB。比如说，我们尝试写入20MB的数据到/tmp/xfs_prjquota里。</p><p>我们可以看到，执行dd写入命令，就会有个出错返回信息"No space left on device"。这表示已经不能再往这个目录下写入数据了，而最后写入数据的文件test.file大小也停留在了10MB。</p><pre><code class="language-shell"># dd if=/dev/zero of=/tmp/xfs_prjquota/test.file bs=1024 count=20000
dd: error writing '/tmp/xfs_prjquota/test.file': No space left on device
10241+0 records in
10240+0 records out
10485760 bytes (10 MB, 10 MiB) copied, 0.0357122 s, 294 MB/s

# ls -l /tmp/xfs_prjquota/test.file
-rw-r--r-- 1 root root 10485760 Oct 31 10:00 /tmp/xfs_prjquota/test.file
</code></pre><p>好了，做到这里，我们发现使用XFS Quota的Project模式，确实可以限制一个目录里的写入数据量，它实现的方式其实也不难，就是下面这两步。</p><p>第一步，给目标目录打上一个Project ID，这个ID最终是写到目录对应的inode上。</p><p>这里我解释一下，inode是文件系统中用来描述一个文件或者一个目录的元数据，里面包含文件大小，数据块的位置，文件所属用户/组，文件读写属性以及其他一些属性。</p><p>那么一旦目录打上这个ID之后，在这个目录下的新建的文件和目录也都会继承这个ID。</p><p>第二步，在XFS文件系统中，我们需要给这个project ID设置一个写入数据块的限制。</p><p>有了ID和限制值之后，文件系统就可以统计所有带这个ID文件的数据块大小总和，并且与限制值进行比较。一旦所有文件大小的总和达到限制值，文件系统就不再允许更多的数据写入了。</p><p>用一句话概括，XFS Quota就是通过前面这两步限制了一个目录里写入的数据量。</p><h2>解决问题</h2><p>我们理解了XFS Quota对目录限流的机制之后，再回到我们最开始的问题，如何确保容器不会写满宿主机上的磁盘。</p><p>你应该已经想到了，方法就是<strong>对OverlayFS的upperdir目录做XFS Quota的限流</strong>，没错，就是这个解决办法！</p><p>其实Docker也已经实现了限流功能，也就是用XFS Quota来限制容器的OverlayFS大小。</p><p>我们在用 <code>docker run</code> 启动容器的时候，加上一个参数 <code>--storage-opt size= &lt;SIZE&gt;</code> ，就能限制住容器OverlayFS文件系统可写入的最大数据量了。</p><p>我们可以一起试一下，这里我们限制的size是10MB。</p><p>进入容器之后，先运行 <code>df -h</code> 命令，这时候你可以看到根目录(/)overlayfs文件系统的大小就10MB，而不是我们之前看到的160GB的大小了。这样容器在它的根目录下，最多只能写10MB数据，就不会把宿主机的磁盘给写满了。</p><p><img src="https://static001.geekbang.org/resource/image/a7/8a/a7906f56d9d107f0a290e610b8cd6f8a.png?wh=1194*210" alt=""></p><p>完成了上面这个小试验之后，我们可以再看一下Docker的代码，看看它的实现是不是和我们想的一样。</p><p>Docker里<a href="https://github.com/moby/moby/blob/19.03/daemon/graphdriver/quota/projectquota.go#L155">SetQuota()</a>函数就是用来实现XFS Quota 限制的，我们可以看到它里面最重要的两步，分别是 <code>setProjectID</code> 和 <code>setProjectQuota</code> 。</p><p>其实，这两步做的就是我们在基本概念中提到的那两步：</p><p>第一步，给目标目录打上一个Project ID；第二步，为这个Project ID在XFS文件系统中，设置一个写入数据块的限制。</p><pre><code class="language-shell">// SetQuota - assign a unique project id to directory and set the quota limits
// for that project id

func (q *Control) SetQuota(targetPath string, quota Quota) error {
        q.RLock()
        projectID, ok := q.quotas[targetPath]
        q.RUnlock()

        if !ok {
                q.Lock()
                projectID = q.nextProjectID

                //
                // assign project id to new container directory
                //

                err := setProjectID(targetPath, projectID)
                if err != nil {
                        q.Unlock()
                        return err
                }

                q.quotas[targetPath] = projectID
                q.nextProjectID++
                q.Unlock()
        }

 

        //
        // set the quota limit for the container's project id
        //

        logrus.Debugf("SetQuota(%s, %d): projectID=%d", targetPath, quota.Size, projectID)
        return setProjectQuota(q.backingFsBlockDev, projectID, quota)
}
</code></pre><p>那 <code>setProjectID</code> 和  <code>setProjectQuota</code> 是如何实现的呢？</p><p>你可以进入到这两个函数里看一下，<strong>它们分别调用了ioctl()和quotactl()这两个系统调用来修改内核中XFS的数据结构，从而完成project ID的设置和Quota值的设置。</strong>具体的细节，我不在这里展开了，如果你有兴趣，可以继续去查看内核中对应的代码。</p><p>好了，Docker里XFS Quota操作的步骤完全和我们先前设想的一样，那么还有最后一个问题要解决，XFS Quota限制的目录是哪一个？</p><p>这个我们可以根据/proc/mounts中容器的OverlayFS Mount信息，再结合Docker的<a href="https://github.com/moby/moby/blob/19.03/daemon/graphdriver/overlay2/overlay.go#L335">代码</a>，就可以知道限制的目录是"/var/lib/docker/overlay2/&lt;docker_id&gt;"。那这个目录下有什么呢？果然upperdir目录中有对应的"diff"目录，就在里面！</p><p><img src="https://static001.geekbang.org/resource/image/4d/9f/4d4d995f052c9a8e3ff3e413c0e1199f.png?wh=1920*646" alt=""></p><p>讲到这里，我想你已经清楚了对于使用OverlayFS的容器，我们应该如何去防止它把宿主机的磁盘给写满了吧？<strong>方法就是对OverlayFS的upperdir目录做XFS Quota的限流。</strong></p><h2>重点总结</h2><p>我们这一讲的问题是，容器写了大量数据到OverlayFS文件系统的根目录，在这个情况下，就会把宿主机的磁盘写满。</p><p>由于OverlayFS自己没有专门的特性，可以限制文件数据写入量。这时我们通过实际试验找到了解决思路：依靠底层文件系统的Quota特性来限制OverlayFS的upperdir目录的大小，这样就能实现限制容器写磁盘的目的。</p><p>底层文件系统XFS Quota的Project模式，能够限制一个目录的文件写入量，这个功能具体是通过这两个步骤实现：</p><p>第一步，给目标目录打上一个Project ID。</p><p>第二步，给这个Project ID在XFS文件系统中设置一个写入数据块的限制。</p><p>Docker正是使用了这个方法，也就是<strong>用XFS Quota来限制OverlayFS的upperdir目录</strong>，通过这个方式控制容器OverlayFS的根目录大小。</p><p>当我们理解了这个方法后，对于不是用Docker启动的容器，比如直接由containerd启动起来的容器，也可以自己实现XFS Quota限制upperdir目录。这样就能有效控制容器对OverlayFS的写数据操作，避免宿主机的磁盘被写满。</p><h2>思考题</h2><p>在正文知识详解的部分，我们使用"xfs_quota"给目录打了project ID并且限制了文件写入的数据量。那在做完这样的限制之后，我们是否能用xfs_quota命令，查询到被限制目录的project ID和限制的数据量呢？</p><p>欢迎你在留言区分享你的思考或疑问。如果这篇文章让你有所收获，也欢迎转发给你的同事、朋友，一起交流和学习。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/5e/96/a03175bc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>莫名</span>
  </div>
  <div class="_2_QraFYR_0">容器 quota 的局限性：容器通常采用 overlay2 driver，仅支持在宿主文件系统 xfs 上开启 quota 功能。意味着 overlay over ext4 不支持 quota 功能，而实际生产环境上 ext4 的使用远多于 xfs。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: @莫名，<br>的确，我一直使用的是xfs, 对于ext4 quota的功能没有测试过。<br>不过我之前搜索过， ext4在2016年就应该支持project quota了。我们可以一起再确认一下。<br><br><br>```<br>commit 689c958cbe6be4f211b40747951a3ba2c73b6715<br>Author: Li Xi &lt;pkuelelixi@gmail.com&gt;<br>Date:   Fri Jan 8 16:01:22 2016 -0500<br><br>    ext4: add project quota support<br><br>    This patch adds mount options for enabling&#47;disabling project quota<br>    accounting and enforcement. A new specific inode is also used for<br>    project quota accounting.<br><br>    [ Includes fix from Dan Carpenter to crrect error checking from dqget(). ]<br><br>    Signed-off-by: Li Xi &lt;lixi@ddn.com&gt;<br>    Signed-off-by: Dmitry Monakhov &lt;dmonakhov@openvz.org&gt;<br>    Signed-off-by: Theodore Ts&#39;o &lt;tytso@mit.edu&gt;<br>    Reviewed-by: Andreas Dilger &lt;adilger@dilger.ca&gt;<br>    Reviewed-by: Jan Kara &lt;jack@suse.cz&gt;</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-11 19:04:18</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/f5/9d/104bb8ea.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek2014</span>
  </div>
  <div class="_2_QraFYR_0">上篇的评论中提到：“我们在2019年初就不用docker了。”<br>这篇中，老师提到了containerd，是说你们已经用containerd替换了docker吗？有机会对containerd和docker之间的使用对比做个介绍吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: @Geek2014<br>对的，我们已经使用containerd快两年了。<br>这是之前我们组做的分享：<br>https:&#47;&#47;www.infoq.cn&#47;article&#47;odslclsjvo8bnx*mbrbk</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-11 11:47:44</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/77/86/4ff2a872.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Sun</span>
  </div>
  <div class="_2_QraFYR_0">老师，我觉得可以补充下 现在k8s 1.14 开始，默认开启 LocalStorageCapacityIsolation， 可以通过限制resources.limits.ephemeral-storage 和resources.requests.ephemeral-storage 来保护宿主机 rootfs了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: @Sun, 谢谢，很好的补充。<br>k8s 通过du来检查ephemeral-storage相关volume&#47;目录的大小，然后如果超出limit就evict pod。<br>du的开销会比较大，evict pod比较适合stateless pod。不过最新的k8s应该在用filesystem quota来限制emptyDir的大小了。 </p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-22 09:58:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/23/81/60/71ed6ac7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>谢哈哈</span>
  </div>
  <div class="_2_QraFYR_0">可以通过xfs_quota -x -c &quot;report -pbih &quot; 目录名称查询projectid</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-11 19:41:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/7c/bb/635a2710.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>徐少文</span>
  </div>
  <div class="_2_QraFYR_0">老师，CGroup子系统中没有对硬盘使用量进行限制的功能模块吗？除了通过文件系统的quota去限制还有其他的方法吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: cgroup里没有对磁盘容量使用限制的模块。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-13 19:35:02</div>
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
  <div class="_2_QraFYR_0">老师能说下你们应用容器Quota的场景么。对于无状态服务一般就是日志会写文件了，离线任务比如机器学习的模型也不敢给人家限制呀😂<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果用户需要使用大容量的磁盘空间，需要使用volume.<br>Quota主要来限制容器的rootfs, 这个rootfs一般是在host的磁盘会和别的容器共享，所以需要对它做限制。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-13 17:16:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/e8/c2/77a413a7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Action</span>
  </div>
  <div class="_2_QraFYR_0">老师 为什么overlayFS 并不是docker_id 呢？<br>[root@localhost overlay2]# docker inspect 5440662c8db<br>[<br>    {<br>        &quot;Id&quot;: &quot;5440662c8db65d5d9ab522e0be1a3911584c492527fcde334c3fdec090cd3857&quot;,<br><br>        &quot;Image&quot;: &quot;sha256:300e315adb2f96afe5f0b2780b87f28ae95231fe3bdd1e16b9ba606307728f55&quot;,<br>  <br>        &quot;GraphDriver&quot;: {<br>            &quot;Data&quot;: {<br>                &quot;LowerDir&quot;: &quot;&#47;home&#47;data&#47;docker&#47;overlay2&#47;4146b7c314a97b46695a59ccdc323709eda7177b3e91ed30d3f4f9326a061584-init&#47;diff:&#47;home&#47;data&#47;docker&#47;overlay2&#47;dc1e8c89044058319c4e0b6917603f3ed8972cd155bfced0c7c0c2509ddaae14&#47;diff&quot;,<br>                &quot;MergedDir&quot;: &quot;&#47;home&#47;data&#47;docker&#47;overlay2&#47;4146b7c314a97b46695a59ccdc323709eda7177b3e91ed30d3f4f9326a061584&#47;merged&quot;,<br>                &quot;UpperDir&quot;: &quot;&#47;home&#47;data&#47;docker&#47;overlay2&#47;4146b7c314a97b46695a59ccdc323709eda7177b3e91ed30d3f4f9326a061584&#47;diff&quot;,<br>                &quot;WorkDir&quot;: &quot;&#47;home&#47;data&#47;docker&#47;overlay2&#47;4146b7c314a97b46695a59ccdc323709eda7177b3e91ed30d3f4f9326a061584&#47;work&quot;<br>            },<br>            &quot;Name&quot;: &quot;overlay2&quot;<br>        },<br><br>    }<br>]<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 看moby(docker)代码里，docker自己对这个目录自己做了一个随机数的生成，具体原因不清楚。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-16 11:46:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTIz9dKN1C8rKQoaVtmEGdzObhlia6zAfTsPYOm4ibz39VjTbu7Aia1LyeedHR26b6nxUtcCufpichcYgw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>上邪忘川</span>
  </div>
  <div class="_2_QraFYR_0">[root@localhost ~]# xfs_quota -x -c &#39;report -h &#47;tmp&#47;xfs_prjquota&#39;<br>Project quota on &#47; (&#47;dev&#47;mapper&#47;centos-root)<br>                        Blocks              <br>Project ID   Used   Soft   Hard Warn&#47;Grace   <br>---------- --------------------------------- <br>#0           3.2G      0      0  00 [------]<br>#2             8K   100M   100M  00 [------]<br>#3           100M   100M   100M  00 [------]<br>#101          10M      0    10M  00 [------]<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-11 23:45:09</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/76/f5/e3f5bd8d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>宝仔</span>
  </div>
  <div class="_2_QraFYR_0">这个是一定要基于xfs文件系统吗？如果是ext4文件系统呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我没有在ext4上测试过，不过通过内核信息，还有网上别人的尝试，ext4应该也是支持（project）quota的。<br><br>https:&#47;&#47;discuss.linuxcontainers.org&#47;t&#47;how-do-i-set-up-the-project-quota-needed-for-limiting-container-storage-size-for-the-dir-backend-on-ubuntu-18-04&#47;7311<br><br>```<br>commit 689c958cbe6be4f211b40747951a3ba2c73b6715<br>Author: Li Xi &lt;pkuelelixi@gmail.com&gt;<br>Date:   Fri Jan 8 16:01:22 2016 -0500<br><br>    ext4: add project quota support<br><br>    This patch adds mount options for enabling&#47;disabling project quota<br>    accounting and enforcement. A new specific inode is also used for<br>    project quota accounting.<br><br>    [ Includes fix from Dan Carpenter to crrect error checking from dqget(). ]<br><br>    Signed-off-by: Li Xi &lt;lixi@ddn.com&gt;<br>    Signed-off-by: Dmitry Monakhov &lt;dmonakhov@openvz.org&gt;<br>    Signed-off-by: Theodore Ts&#39;o &lt;tytso@mit.edu&gt;<br>    Reviewed-by: Andreas Dilger &lt;adilger@dilger.ca&gt;<br>    Reviewed-by: Jan Kara &lt;jack@suse.cz&gt;</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-11 09:38:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/93/38/71615300.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>DayDayUp</span>
  </div>
  <div class="_2_QraFYR_0">老师讲的太好了，不知道面试贵司会不会有机会呢？老师在eBay哪个team啊？应届生要吗？？？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: eBay cloud team.  我们一直在招人的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-28 13:56:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKzqiaZnBw2myRWY802u48Rw3W2zDtKoFQ6vN63m4FdyjibM21FfaOYe8MbMpemUdxXJeQH6fRdVbZA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>kissingers</span>
  </div>
  <div class="_2_QraFYR_0">k8s 中readonlyrootfs 也是同样的场景吧</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-12 09:56:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/64/05/6989dce6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我来也</span>
  </div>
  <div class="_2_QraFYR_0">之前工作中就经常因为es错误日志把宿主机的磁盘撑满。但还从未往这个方向想，还可以限制某个项目的磁盘使用量。😂</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-03 11:45:07</div>
  </div>
</div>
</div>
</li>
</ul>