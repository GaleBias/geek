<audio title="20 _ 容器安全（2）：在容器中，我不以root用户来运行程序可以吗？" src="https://static001.geekbang.org/resource/audio/3e/0c/3eae9ccddca78e7a95e3b1fc7ba1140c.mp3" controls="controls"></audio> 
<p>你好，我是程远。</p><p>在<a href="https://time.geekbang.org/column/article/326253">上一讲</a>里，我们学习了Linux capabilities的概念，也知道了对于非privileged的容器，容器中root用户的capabilities是有限制的，因此容器中的root用户无法像宿主机上的root用户一样，拿到完全掌控系统的特权。</p><p>那么是不是让非privileged的容器以root用户来运行程序，这样就能保证安全了呢？这一讲，我们就来聊一聊容器中的root用户与安全相关的问题。</p><h2>问题再现</h2><p>说到容器中的用户（user），你可能会想到，在Linux Namespace中有一项隔离技术，也就是User Namespace。</p><p>不过在容器云平台Kubernetes上目前还不支持User Namespace，所以我们先来看看在没有User Namespace的情况下，容器中用root用户运行，会发生什么情况。</p><p>首先，我们可以用下面的命令启动一个容器，在这里，我们把宿主机上/etc目录以volume的形式挂载到了容器中的/mnt目录下面。</p><pre><code class="language-shell"># docker run -d --name root_example -v /etc:/mnt  centos sleep 3600
</code></pre><p>然后，我们可以看一下容器中的进程"sleep 3600"，它在容器中和宿主机上的用户都是root，也就是说，容器中用户的uid/gid和宿主机上的完全一样。</p><!-- [[[read_end]]] --><pre><code class="language-shell"># docker exec -it root_example bash -c "ps -ef | grep sleep"
root         1     0  0 01:14 ?        00:00:00 /usr/bin/coreutils --coreutils-prog-shebang=sleep /usr/bin/sleep 3600
 
# ps -ef | grep sleep
root      5473  5443  0 18:14 ?        00:00:00 /usr/bin/coreutils --coreutils-prog-shebang=sleep /usr/bin/sleep 3600
</code></pre><p>虽然容器里root用户的capabilities被限制了一些，但是在容器中，对于被挂载上来的/etc目录下的文件，比如说shadow文件，以这个root用户的权限还是可以做修改的。</p><pre><code class="language-shell"># docker exec -it root_example bash
[root@9c7b76232c19 /]# ls /mnt/shadow -l
---------- 1 root root 586 Nov 26 13:47 /mnt/shadow
[root@9c7b76232c19 /]# echo "hello" &gt;&gt; /mnt/shadow
</code></pre><p>接着我们看看后面这段命令输出，可以确认在宿主机上文件被修改了。</p><pre><code class="language-shell"># tail -n 3 /etc/shadow
grafana:!!:18437::::::
tcpdump:!!:18592::::::
hello
</code></pre><p>这个例子说明容器中的root用户也有权限修改宿主机上的关键文件。</p><p>当然在云平台上，比如说在Kubernetes里，我们是可以限制容器去挂载宿主机的目录的。</p><p>不过，由于容器和宿主机是共享Linux内核的，一旦软件有漏洞，那么容器中以root用户运行的进程就有机会去修改宿主机上的文件了。比如2019年发现的一个RunC的漏洞 <a href="https://nvd.nist.gov/vuln/detail/CVE-2019-5736">CVE-2019-5736</a>， 这导致容器中root用户有机会修改宿主机上的RunC程序，并且容器中的root用户还会得到宿主机上的运行权限。</p><h2>问题分析</h2><p>对于前面的问题，接下来我们就来讨论一下<strong>解决办法</strong>，在讨论问题的过程中，也会涉及一些新的概念，主要有三个。</p><h3>方法一：Run as non-root user（给容器指定一个普通用户）</h3><p>我们如果不想让容器以root用户运行，最直接的办法就是给容器指定一个普通用户uid。这个方法很简单，比如可以在docker启动容器的时候加上"-u"参数，在参数中指定uid/gid。</p><p>具体的操作代码如下：</p><pre><code class="language-shell"># docker run -ti --name root_example -u 6667:6667 -v /etc:/mnt  centos bash
bash-4.4$ id
uid=6667 gid=6667 groups=6667
bash-4.4$ ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
6667         1     0  1 01:27 pts/0    00:00:00 bash
6667         8     1  0 01:27 pts/0    00:00:00 ps -ef
</code></pre><p>还有另外一个办法，就是我们在创建容器镜像的时候，用Dockerfile为容器镜像里建立一个用户。</p><p>为了方便你理解，我还是举例说明。就像下面例子中的nonroot，它是一个用户名，我们用USER关键字来指定这个nonroot用户，这样操作以后，容器里缺省的进程都会以这个用户启动。</p><p>这样在运行Docker命令的时候就不用加"-u"参数来指定用户了。</p><pre><code class="language-shell"># cat Dockerfile
FROM centos
 
RUN adduser -u 6667 nonroot
USER nonroot
 
# docker build -t registry/nonroot:v1 .
…
 
# docker run -d --name root_example -v /etc:/mnt registry/nonroot:v1 sleep 3600
050809a716ab0a9481a6dfe711b332f74800eff5fea8b4c483fa370b62b4b9b3
 
# docker exec -it root_example bash
[nonroot@050809a716ab /]$ id
uid=6667(nonroot) gid=6667(nonroot) groups=6667(nonroot)
[nonroot@050809a716ab /]$ ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
nonroot      1     0  0 01:43 ?        00:00:00 /usr/bin/coreutils --coreutils-prog-shebang=sleep /usr/bin/sleep 3600
</code></pre><p>好，在容器中使用普通用户运行之后，我们再看看，现在能否修改被挂载上来的/etc目录下的文件? 显然，现在不可以修改了。</p><pre><code class="language-shell">[nonroot@050809a716ab /]$ echo "hello" &gt;&gt; /mnt/shadow
bash: /mnt/shadow: Permission denied
</code></pre><p>那么是不是只要给容器中指定了一个普通用户，这个问题就圆满解决了呢？其实在云平台上，这么做还是会带来别的问题，我们一起来看看。</p><p>由于用户uid是整个节点中共享的，那么在容器中定义的uid，也就是宿主机上的uid，这样就很容易引起uid的冲突。</p><p>比如说，多个客户在建立自己的容器镜像的时候都选择了同一个uid 6667。那么当多个客户的容器在同一个节点上运行的时候，其实就都使用了宿主机上uid 6667。</p><p>我们都知道，在一台Linux系统上，每个用户下的资源是有限制的，比如打开文件数目（open files）、最大进程数目（max user processes）等等。一旦有很多个容器共享一个uid，这些容器就很可能很快消耗掉这个uid下的资源，这样很容易导致这些容器都不能再正常工作。</p><p>要解决这个问题，必须要有一个云平台级别的uid管理和分配，但选择这个方法也要付出代价。因为这样做是可以解决问题，但是用户在定义自己容器中的uid的时候，他们就需要有额外的操作，而且平台也需要新开发对uid平台级别的管理模块，完成这些事情需要的工作量也不少。</p><h3>方法二：User Namespace（用户隔离技术的支持）</h3><p>那么在没有使用User Namespace的情况，对于容器平台上的用户管理还是存在问题。你可能会想到，我们是不是应该去尝试一下User Namespace?</p><p>好的，我们就一起来看看使用User Namespace对解决用户管理问题有没有帮助。首先，我们简单了解一下<a href="https://man7.org/linux/man-pages/man7/user_namespaces.7.html">User Namespace</a>的概念。</p><p>User Namespace隔离了一台Linux节点上的User ID（uid）和Group ID（gid），它给Namespace中的uid/gid的值与宿主机上的uid/gid值建立了一个映射关系。经过User Namespace的隔离，我们在Namespace中看到的进程的uid/gid，就和宿主机Namespace中看到的uid和gid不一样了。</p><p>你可以看下面的这张示意图，应该就能很快知道User Namespace大概是什么意思了。比如namespace_1里的uid值是0到999，但其实它在宿主机上对应的uid值是1000到1999。</p><p>还有一点你要注意的是，User Namespace是可以嵌套的，比如下面图里的namespace_2里可以再建立一个namespace_3，这个嵌套的特性是其他Namespace没有的。</p><p><img src="https://static001.geekbang.org/resource/image/64/9c/647a11a38498128e0b00a48931e2f09c.jpg?wh=1920*1080" alt=""></p><p>我们可以启动一个带User Namespace的容器来感受一下。这次启动容器，我们用一下<a href="https://podman.io/">podman</a>这个工具，而不是Docker。</p><p>跟Docker相比，podman不再有守护进程dockerd，而是直接通过fork/execve的方式来启动一个新的容器。这种方式启动容器更加简单，也更容易维护。</p><p>Podman的命令参数兼容了绝大部分的docker命令行参数，用过Docker的同学也很容易上手podman。你感兴趣的话，可以跟着这个<a href="https://podman.io/getting-started/installation">手册</a>在你自己的Linux系统上装一下podman。</p><p>那接下来，我们就用下面的命令来启动一个容器：</p><pre><code class="language-shell"># podman run -ti  -v /etc:/mnt --uidmap 0:2000:1000 centos bash
</code></pre><p>我们可以看到，其他参数和前面的Docker命令是一样的。</p><p>这里我们在命令里增加一个参数，"--uidmap 0:2000:1000"，这个是标准的User Namespace中uid的映射格式："ns_uid:host_uid:amount"。</p><p>那这个例子里的"0:2000:1000"是什么意思呢？我给你解释一下。</p><p>第一个0是指在新的Namespace里uid从0开始，中间的那个2000指的是Host Namespace里被映射的uid从2000开始，最后一个1000是指总共需要连续映射1000个uid。</p><p>所以，我们可以得出，<strong>这个容器里的uid 0是被映射到宿主机上的uid 2000的。</strong>这一点我们可以验证一下。</p><p>首先，我们先在容器中以用户uid 0运行一下 <code>sleep</code> 这个命令：</p><pre><code class="language-shell"># id
uid=0(root) gid=0(root) groups=0(root)
# sleep 3600
</code></pre><p>然后就是第二步，到宿主机上查看一下这个进程的uid。这里我们可以看到，进程uid的确是2000了。</p><pre><code class="language-shell"># ps -ef |grep sleep
2000     27021 26957  0 01:32 pts/0    00:00:00 /usr/bin/coreutils --coreutils-prog-shebang=sleep /usr/bin/sleep 3600
</code></pre><p>第三步，我们可以再回到容器中，仍然以容器中的root对被挂载上来的/etc目录下的文件做操作，这时可以看到操作是不被允许的。</p><pre><code class="language-shell"># echo "hello" &gt;&gt; /mnt/shadow
bash: /mnt/shadow: Permission denied
# id
uid=0(root) gid=0(root) groups=0(root)
</code></pre><p>好了，通过这些操作以及和前面User Namespace的概念的解释，我们可以总结出容器使用User Namespace有两个好处。</p><p><strong>第一，它把容器中root用户（uid 0）映射成宿主机上的普通用户。</strong></p><p>作为容器中的root，它还是可以有一些Linux capabilities，那么在容器中还是可以执行一些特权的操作。而在宿主机上uid是普通用户，那么即使这个用户逃逸出容器Namespace，它的执行权限还是有限的。</p><p><strong>第二，对于用户在容器中自己定义普通用户uid的情况，我们只要为每个容器在节点上分配一个uid范围，就不会出现在宿主机上uid冲突的问题了。</strong></p><p>因为在这个时候，我们只要在节点上分配容器的uid范围就可以了，所以从实现上说，相比在整个平台层面给容器分配uid，使用User Namespace这个办法要方便得多。</p><p>这里我额外补充一下，前面我们说了Kubernetes目前还不支持User Namespace，如果你想了解相关工作的进展，可以看一下社区的这个<a href="https://github.com/kubernetes/enhancements/pull/2101">PR</a>。</p><h3>方法三：rootless container（以非root用户启动和管理容器）</h3><p>前面我们已经讨论了，在容器中以非root用户运行进程可以降低容器的安全风险。除了在容器中使用非root用户，社区还有一个rootless container的概念。</p><p>这里rootless container中的"rootless"不仅仅指容器中以非root用户来运行进程，还指以非root用户来创建容器，管理容器。也就是说，启动容器的时候，Docker或者podman是以非root用户来执行的。</p><p>这样一来，就能进一步提升容器中的安全性，我们不用再担心因为containerd或者RunC里的代码漏洞，导致容器获得宿主机上的权限。</p><p>我们可以参考redhat blog里的这篇<a href="https://developers.redhat.com/blog/2020/09/25/rootless-containers-with-podman-the-basics/">文档</a>， 在宿主机上用redhat这个用户通过podman来启动一个容器。在这个容器中也使用了User Namespace，并且把容器中的uid 0映射为宿主机上的redhat用户了。</p><pre><code class="language-shell">$ id
uid=1001(redhat) gid=1001(redhat) groups=1001(redhat)
$ podman run -it  ubi7/ubi bash   ### 在宿主机上以redhat用户启动容器
[root@206f6d5cb033 /]# id     ### 容器中的用户是root
uid=0(root) gid=0(root) groups=0(root)
[root@206f6d5cb033 /]# sleep 3600   ### 在容器中启动一个sleep 进程
</code></pre><pre><code class="language-shell"># ps -ef |grep sleep   ###在宿主机上查看容器sleep进程对应的用户
redhat   29433 29410  0 05:14 pts/0    00:00:00 sleep 3600
</code></pre><p>目前Docker和podman都支持了rootless container，Kubernetes对<a href="https://github.com/kubernetes/enhancements/issues/2033">rootless container支持</a>的工作也在进行中。</p><h2>重点小结</h2><p>我们今天讨论的内容是root用户与容器安全的问题。</p><p>尽管容器中root用户的Linux capabilities已经减少了很多，但是在没有User Namespace的情况下，容器中root用户和宿主机上的root用户的uid是完全相同的，一旦有软件的漏洞，容器中的root用户就可以操控整个宿主机。</p><p><strong>为了减少安全风险，业界都是建议在容器中以非root用户来运行进程。</strong>不过在没有User Namespace的情况下，在容器中使用非root用户，对于容器云平台来说，对uid的管理会比较麻烦。</p><p>所以，我们还是要分析一下User Namespace，它带来的好处有两个。一个是把容器中root用户（uid 0）映射成宿主机上的普通用户，另外一个好处是在云平台里对于容器uid的分配要容易些。</p><p>除了在容器中以非root用户来运行进程外，Docker和podman都支持了rootless container，也就是说它们都可以以非root用户来启动和管理容器，这样就进一步降低了容器的安全风险。</p><h2>思考题</h2><p>我在这一讲里提到了rootless container，不过对于rootless container的支持，还存在着不少的难点，比如容器网络的配置、Cgroup的配置，你可以去查阅一些资料，看看podman是怎么解决这些问题的。</p><p>欢迎你在留言区提出你的思考和疑问。如果这一讲对你有帮助，也欢迎转发给你的同事、朋友，一起交流学习。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/64/05/6989dce6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我来也</span>
  </div>
  <div class="_2_QraFYR_0">最近在使用Helm部署gitlab服务的过程中,就发现了 postgresql 和 redis 组件默认是不以root用户执行的,而是一个 User ID 为1001的用户在执行.<br>这样做,就需要有个k8s的 initContainer 容器先以root用户权限去修改存储目录的权限. 否则后面服务的1001号用户可能就没有权限去写文件了.<br>------------------<br><br>最近遇到一个问题,想咨询一下老师:<br>你们有使用过 容器资源可视化隔离方案(lxcfs) 么, 有没有什么坑?<br>通俗点就是:让容器中的free, top等命令看到容器的数据，而不是物理机的数据。<br>------------------<br><br>我遇到的问题是在容器内执行类似`go build&#47;test`命令时,默认是根据当前CPU核数来调整构建的并发数.<br>这就导致了实际只给容器分配了1个核,但是它以为自己有16个核.<br>然后就开16个link进程,互相之间除了有竞争,导致CPU上下文切换频繁,更要命的是把磁盘IO给弄满了.影响了整台宿主机的性能.<br>(由于项目比较大,需要构建的文件比较多,所以很容器就让宿主机的IO达到了云服务器SSD磁盘的限制 160MB&#47;s)<br><br>我知道在我这个场景下,可以通过指定构建命令`-p`来控制构建的并发数.<br>(https:&#47;&#47;golang.org&#47;cmd&#47;go&#47;#hdr-Compile_packages_and_dependencies)<br>实际也这么尝试过,效果也不错.<br>但问题是,我的项目会很多,每个人构建命令的写法都完全不一样,如果每个地方都去指定参数,就会比较繁琐,且容易遗漏.<br><br>------------------<br>后来,我看到一篇文章: 容器资源可视化隔离的实现方法<br>(https:&#47;&#47;mp.weixin.qq.com&#47;s&#47;SCxD4OiDYsmoIyN5XMk4YA)<br><br>之前也在其他专栏中看老师提到过 lxcfs.<br>我在想,老师在迁移上k8s的过程中,肯定也遇到过类似的问题,不知道老师是如何解决的呢?<br><br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: @我来也<br>很好的问题。我们在最开始也考虑过使用lxcfs, 当时碰到的问题也是当一批java应用从虚拟机迁移到容器平台之后，发现jvm看到的是整个宿主机的资源。<br><br>不过后来，发现大部分语言和应用都是可以加参数或者做修改来适配容器化的，因此，我们的方向是让应用也必须考虑容器和云原生的设计，因为这个是大的趋势，应用这边也是愿意接受这个改变的。<br><br>还有一点，当时我们在试lxcfs的时候发现，如果容器需要的cpu不是整数，似乎lxcfs也不能支持（不知道最新的lxcfs是不是有所改变），同时在host上需要额外维护这个lxcfs的可靠性。 这样在大部分主要应用都愿意往容器化方向走的大环境下，我们就不再考虑lxcfs了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-12-31 09:08:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/23/7e/c0/1c3fd7dd.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>朱新威</span>
  </div>
  <div class="_2_QraFYR_0">老师，我发现一个很有趣的现象，有点困惑；<br><br>在宿主机上：<br>以root用户运行capsh --print<br>发现Current字段包含许多capabilities<br><br>以非root用户运行capsh --print<br>发现Current 字段包含零个capabilities，说明非root用户启动的进程缺省没有任何capabilities<br><br>docker容器内：<br>root用户运行capsh --print<br>发现Current 字段包含14个capabilities，比宿主机上少了一些，对宿主机的&#47;etc&#47;shadow有读写权限<br><br>非root用户运行capsh --print<br>发现Current字段仍然包含14个capabilities，对宿主机的&#47;etc&#47;shadow没有读写权限<br><br>这就让我感觉有点困惑了，原本预期容器内非root用户运行capsh  --print的capabilities应该为空呀，或者知道少于root用户的capabilities吧？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-09 21:56:01</div>
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
  <div class="_2_QraFYR_0"><br>user limit 是session的？每个容器及时使用相同的user id ，也不会当做累计？<br><br>User resource limits dictate the amount of resources that can be used for a particular session. The resources that can be controled are:<br><br>maximum size of core files<br>maximum size of a process&#39;s data segment<br>maximum size of files created<br>maximum size that may be locked into memory<br>maximum size of resident memory<br>maximum number of file descriptors open at one time<br>maximum size of the stack<br>maximum amount of cpu time used<br>maximum number of processes allowed<br>maximum size of virtual memory available<br>It is important to note that these settings are per-session. This means that they are only effective for the time that the user is logged in (and for any processes that they run during that period). They are not global settings. In other words, they are only active for the duration of the session and the settings are not cumulative. For example, if you set the maximum number of processes to 11, the user may only have 11 processes running per session. They are not limited to 11 total processes on the machine as they may initiate another session. Each of the settings are per process settings during the session, with the exception of the maximum number of processes.<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: @Sun 很好的问题！<br>我在这里指的是pam_limits， 在&#47;etc&#47;security&#47;limits.conf中限制某个用户资源之后，然后在pam *_auth 和 runuser中enable pam_limits 之后，那么同一个用户即使在不同的session里，资源的限制也是累计了。<br>你可以在CentOS的系统里试试。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-07 10:33:55</div>
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
  <div class="_2_QraFYR_0">老师 docker -u 参数 是不是就是 通过user namespace 进行隔离</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: -u 只是指定了在容器启动的时候缺省用的uid&#47;gid, 这里的uid&#47;gid和宿主机上的是一样的，并没有建立出新的user namespace.</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-04 11:29:36</div>
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
  <div class="_2_QraFYR_0">Kubernetes v1.25 添加了对容器 user namespaces 的支持</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-11-15 09:53:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/c4/03/f753fda7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>JianXu</span>
  </div>
  <div class="_2_QraFYR_0">install slirp4netns and Podman on your machine by entering the following command:<br><br>$ yum install slirp4netns podman -y<br>We will use slirp4netns to connect a network namespace to the internet in a completely rootless (or unprivileged) way.</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-08 22:30:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/08/bf/cd6bfc22.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>自然</span>
  </div>
  <div class="_2_QraFYR_0">有个场景：用jenkins  在 openjdk镜像里 maven 编译java项目, 一个 maven目录（在主机上，而且还有其他很多工具），一个项目源码目录  需要映射到  openjdk镜像里（普通用户启动docker），jenkins 里的pipline 是大家都可以写的。 如何防止 加载主机上目录 在docker镜像里 root用户 随意修改呢（ 比如 我不想他删除 主机上的maven）？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-22 19:32:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/0b/37/20ac0432.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>sunnoy</span>
  </div>
  <div class="_2_QraFYR_0">如果容器内的用户uid在宿主机上不存在呢，这个时候描述符的分配是怎么样的呢</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-24 11:39:46</div>
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
  <div class="_2_QraFYR_0">&quot;由于用户 uid 是整个节点中共享的，那么在容器中定义的 uid，也就是宿主机上的 uid，这样就很容易引起 uid 的冲突。&quot;老师这句话怎么理解，容器内uid与宿主机uid是怎么样的关系呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 当容器没有使用user namespace， 那么容器中进程所属的uid&#47;gid, 就是宿主机里uid&#47;gid。你可以运行一下课程中的例子，从宿主机上看看容器进程的uid。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-04 14:11:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/64/08/0287f41f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>争光 Alan</span>
  </div>
  <div class="_2_QraFYR_0">老师，感谢您的分享，学到了很多知识，也感谢解答了很多疑问，有个小小的请求：能公布个微信群之类的吗？把学员加一起相互讨论问题，交流心得</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: @争光 Alan， 我还是会定期回答这个专栏中的大家的提问的。如果你有什么问题，还是在可以在这里提问的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-21 12:31:58</div>
  </div>
</div>
</div>
</li>
</ul>