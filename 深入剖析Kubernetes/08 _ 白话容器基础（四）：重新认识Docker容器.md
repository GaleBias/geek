<audio title="08 _ 白话容器基础（四）：重新认识Docker容器" src="https://static001.geekbang.org/resource/audio/79/31/79e8421b8d2f88dc0b4733aa635def31.mp3" controls="controls"></audio> 
<p>你好，我是张磊。今天我和你分享的主题是：白话容器基础之重新认识Docker容器。</p><p>在前面的三次分享中，我分别从Linux Namespace的隔离能力、Linux Cgroups的限制能力，以及基于rootfs的文件系统三个角度，为你剖析了一个Linux容器的核心实现原理。</p><blockquote>
<p>备注：之所以要强调Linux容器，是因为比如Docker on Mac，以及Windows Docker（Hyper-V实现），实际上是基于虚拟化技术实现的，跟我们这个专栏着重介绍的Linux容器完全不同。</p>
</blockquote><p>而在今天的分享中，我会通过一个实际案例，对“白话容器基础”系列的所有内容做一次深入的总结和扩展。希望通过这次的讲解，能够让你更透彻地理解Docker容器的本质。</p><p>在开始实践之前，你需要准备一台Linux机器，并安装Docker。这个流程我就不再赘述了。</p><p>这一次，<span class="orange">我要用Docker部署一个用Python编写的Web应用</span>。这个应用的代码部分（<code>app.py</code>）非常简单：</p><pre><code>from flask import Flask
import socket
import os

app = Flask(__name__)

@app.route('/')
def hello():
    html = &quot;&lt;h3&gt;Hello {name}!&lt;/h3&gt;&quot; \
           &quot;&lt;b&gt;Hostname:&lt;/b&gt; {hostname}&lt;br/&gt;&quot;           
    return html.format(name=os.getenv(&quot;NAME&quot;, &quot;world&quot;), hostname=socket.gethostname())
    
if __name__ == &quot;__main__&quot;:
    app.run(host='0.0.0.0', port=80)
</code></pre><p>在这段代码中，我使用Flask框架启动了一个Web服务器，而它唯一的功能是：如果当前环境中有“NAME”这个环境变量，就把它打印在“Hello”之后，否则就打印“Hello world”，最后再打印出当前环境的hostname。</p><!-- [[[read_end]]] --><p>这个应用的依赖，则被定义在了同目录下的requirements.txt文件里，内容如下所示：</p><pre><code>$ cat requirements.txt
Flask
</code></pre><p><strong>而将这样一个应用容器化的第一步，是制作容器镜像。</strong></p><p>不过，相较于我之前介绍的制作rootfs的过程，Docker为你提供了一种更便捷的方式，叫作Dockerfile，如下所示。</p><pre><code># 使用官方提供的Python开发镜像作为基础镜像
FROM python:2.7-slim

# 将工作目录切换为/app
WORKDIR /app

# 将当前目录下的所有内容复制到/app下
ADD . /app

# 使用pip命令安装这个应用所需要的依赖
RUN pip install --trusted-host pypi.python.org -r requirements.txt

# 允许外界访问容器的80端口
EXPOSE 80

# 设置环境变量
ENV NAME World

# 设置容器进程为：python app.py，即：这个Python应用的启动命令
CMD [&quot;python&quot;, &quot;app.py&quot;]
</code></pre><p>通过这个文件的内容，你可以看到<strong>Dockerfile的设计思想，是使用一些标准的原语（即大写高亮的词语），描述我们所要构建的Docker镜像。并且这些原语，都是按顺序处理的。</strong></p><p>比如FROM原语，指定了“python:2.7-slim”这个官方维护的基础镜像，从而免去了安装Python等语言环境的操作。否则，这一段我们就得这么写了：</p><pre><code>FROM ubuntu:latest
RUN apt-get update -yRUN apt-get install -y python-pip python-dev build-essential
...
</code></pre><p>其中，RUN原语就是在容器里执行shell命令的意思。</p><p>而WORKDIR，意思是在这一句之后，Dockerfile后面的操作都以这一句指定的/app目录作为当前目录。</p><p>所以，到了最后的CMD，意思是Dockerfile指定python app.py为这个容器的进程。这里，app.py的实际路径是/app/app.py。所以，<code>CMD ["python", "app.py"]</code>等价于"<code>docker run &lt;image&gt; python app.py</code>"。</p><p>另外，在使用Dockerfile时，你可能还会看到一个叫作ENTRYPOINT的原语。实际上，它和CMD都是Docker容器进程启动所必需的参数，完整执行格式是：“ENTRYPOINT CMD”。</p><p>但是，默认情况下，Docker会为你提供一个隐含的ENTRYPOINT，即：/bin/sh -c。所以，在不指定ENTRYPOINT时，比如在我们这个例子里，实际上运行在容器里的完整进程是：<code>/bin/sh -c "python app.py"</code>，即CMD的内容就是ENTRYPOINT的参数。</p><blockquote>
<p>备注：基于以上原因，<strong>我们后面会统一称Docker容器的启动进程为ENTRYPOINT，而不是CMD。</strong></p>
</blockquote><p>需要注意的是，Dockerfile里的原语并不都是指对容器内部的操作。就比如ADD，它指的是把当前目录（即Dockerfile所在的目录）里的文件，复制到指定容器内的目录当中。</p><p>读懂这个Dockerfile之后，我再把上述内容，保存到当前目录里一个名叫“Dockerfile”的文件中：</p><pre><code>$ ls
Dockerfile  app.py   requirements.txt
</code></pre><p>接下来，我就可以让Docker制作这个镜像了，在当前目录执行：</p><pre><code>$ docker build -t helloworld .
</code></pre><p>其中，-t的作用是给这个镜像加一个Tag，即：起一个好听的名字。docker build会自动加载当前目录下的Dockerfile文件，然后按照顺序，执行文件中的原语。而这个过程，实际上可以等同于Docker使用基础镜像启动了一个容器，然后在容器中依次执行Dockerfile中的原语。</p><p><strong>需要注意的是，Dockerfile中的每个原语执行后，都会生成一个对应的镜像层</strong>。即使原语本身并没有明显地修改文件的操作（比如，ENV原语），它对应的层也会存在。只不过在外界看来，这个层是空的。</p><p>docker build操作完成后，我可以通过docker images命令查看结果：</p><pre><code>$ docker image ls

REPOSITORY            TAG                 IMAGE ID
helloworld         latest              653287cdf998
</code></pre><p>通过这个镜像ID，你就可以使用在<a href="https://time.geekbang.org/column/article/17921">《白话容器基础（三）：深入理解容器镜像》</a>中讲过的方法，查看这些新增的层在AuFS路径下对应的文件和目录了。</p><p><strong>接下来，我使用这个镜像，通过docker run命令启动容器：</strong></p><pre><code>$ docker run -p 4000:80 helloworld
</code></pre><p>在这一句命令中，镜像名helloworld后面，我什么都不用写，因为在Dockerfile中已经指定了CMD。否则，我就得把进程的启动命令加在后面：</p><pre><code>$ docker run -p 4000:80 helloworld python app.py
</code></pre><p>容器启动之后，我可以使用docker ps命令看到：</p><pre><code>$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED
4ddf4638572d        helloworld       &quot;python app.py&quot;     10 seconds ago
</code></pre><p>同时，我已经通过-p 4000:80告诉了Docker，请把容器内的80端口映射在宿主机的4000端口上。</p><p>这样做的目的是，只要访问宿主机的4000端口，我就可以看到容器里应用返回的结果：</p><pre><code>$ curl http://localhost:4000
&lt;h3&gt;Hello World!&lt;/h3&gt;&lt;b&gt;Hostname:&lt;/b&gt; 4ddf4638572d&lt;br/&gt;
</code></pre><p>否则，我就得先用docker inspect命令查看容器的IP地址，然后访问“http://&lt;容器IP地址&gt;:80”才可以看到容器内应用的返回。</p><p>至此，我已经使用容器完成了一个应用的开发与测试，如果现在想要把这个容器的镜像上传到DockerHub上分享给更多的人，我要怎么做呢？</p><p>为了能够上传镜像，<strong>我首先需要注册一个Docker Hub账号，然后使用docker login命令登录</strong>。</p><p>接下来，我要<strong>用docker tag命令给容器镜像起一个完整的名字</strong>：</p><pre><code>$ docker tag helloworld geektime/helloworld:v1
</code></pre><blockquote>
<p>注意：你自己做实验时，请将"geektime"替换成你自己的Docker Hub账户名称，比如zhangsan/helloworld:v1</p>
</blockquote><p>其中，geektime是我在Docker Hub上的用户名，它的“学名”叫镜像仓库（Repository）；“/”后面的helloworld是这个镜像的名字，而“v1”则是我给这个镜像分配的版本号。</p><p><strong>然后，我执行docker push：</strong></p><pre><code>$ docker push geektime/helloworld:v1
</code></pre><p>这样，我就可以把这个镜像上传到Docker Hub上了。</p><p>此外，我还可以使用docker commit指令，把一个正在运行的容器，直接提交为一个镜像。一般来说，需要这么操作原因是：这个容器运行起来后，我又在里面做了一些操作，并且要把操作结果保存到镜像里，比如：</p><pre><code>$ docker exec -it 4ddf4638572d /bin/sh
# 在容器内部新建了一个文件
root@4ddf4638572d:/app# touch test.txt
root@4ddf4638572d:/app# exit

#将这个新建的文件提交到镜像中保存
$ docker commit 4ddf4638572d geektime/helloworld:v2
</code></pre><p>这里，我使用了docker exec命令进入到了容器当中。在了解了Linux Namespace的隔离机制后，你应该会很自然地想到一个问题：<span class="orange">docker exec是怎么做到进入容器里的呢？</span></p><p>实际上，Linux Namespace创建的隔离空间虽然看不见摸不着，但一个进程的Namespace信息在宿主机上是确确实实存在的，并且是以一个文件的方式存在。</p><p>比如，通过如下指令，你可以看到当前正在运行的Docker容器的进程号（PID）是25686：</p><pre><code>$ docker inspect --format '{{ .State.Pid }}'  4ddf4638572d
25686
</code></pre><p>这时，你可以通过查看宿主机的proc文件，看到这个25686进程的所有Namespace对应的文件：</p><pre><code>$ ls -l  /proc/25686/ns
total 0
lrwxrwxrwx 1 root root 0 Aug 13 14:05 cgroup -&gt; cgroup:[4026531835]
lrwxrwxrwx 1 root root 0 Aug 13 14:05 ipc -&gt; ipc:[4026532278]
lrwxrwxrwx 1 root root 0 Aug 13 14:05 mnt -&gt; mnt:[4026532276]
lrwxrwxrwx 1 root root 0 Aug 13 14:05 net -&gt; net:[4026532281]
lrwxrwxrwx 1 root root 0 Aug 13 14:05 pid -&gt; pid:[4026532279]
lrwxrwxrwx 1 root root 0 Aug 13 14:05 pid_for_children -&gt; pid:[4026532279]
lrwxrwxrwx 1 root root 0 Aug 13 14:05 user -&gt; user:[4026531837]
lrwxrwxrwx 1 root root 0 Aug 13 14:05 uts -&gt; uts:[4026532277]
</code></pre><p>可以看到，一个进程的每种Linux Namespace，都在它对应的/proc/[进程号]/ns下有一个对应的虚拟文件，并且链接到一个真实的Namespace文件上。</p><p>有了这样一个可以“hold住”所有Linux Namespace的文件，我们就可以对Namespace做一些很有意义事情了，比如：加入到一个已经存在的Namespace当中。</p><p><strong>这也就意味着：一个进程，可以选择加入到某个进程已有的Namespace当中，从而达到“进入”这个进程所在容器的目的，这正是docker exec的实现原理。</strong></p><p>而这个操作所依赖的，乃是一个名叫setns()的Linux系统调用。它的调用方法，我可以用如下一段小程序为你说明：</p><pre><code>#define _GNU_SOURCE
#include &lt;fcntl.h&gt;
#include &lt;sched.h&gt;
#include &lt;unistd.h&gt;
#include &lt;stdlib.h&gt;
#include &lt;stdio.h&gt;

#define errExit(msg) do { perror(msg); exit(EXIT_FAILURE);} while (0)

int main(int argc, char *argv[]) {
    int fd;
    
    fd = open(argv[1], O_RDONLY);
    if (setns(fd, 0) == -1) {
        errExit(&quot;setns&quot;);
    }
    execvp(argv[2], &amp;argv[2]); 
    errExit(&quot;execvp&quot;);
}
</code></pre><p>这段代码功能非常简单：它一共接收两个参数，第一个参数是argv[1]，即当前进程要加入的Namespace文件的路径，比如/proc/25686/ns/net；而第二个参数，则是你要在这个Namespace里运行的进程，比如/bin/bash。</p><p>这段代码的核心操作，则是通过open()系统调用打开了指定的Namespace文件，并把这个文件的描述符fd交给setns()使用。在setns()执行后，当前进程就加入了这个文件对应的Linux Namespace当中了。</p><p>现在，你可以编译执行一下这个程序，加入到容器进程（PID=25686）的Network Namespace中：</p><pre><code>$ gcc -o set_ns set_ns.c 
$ ./set_ns /proc/25686/ns/net /bin/bash 
$ ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:ac:11:00:02  
          inet addr:172.17.0.2  Bcast:0.0.0.0  Mask:255.255.0.0
          inet6 addr: fe80::42:acff:fe11:2/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:12 errors:0 dropped:0 overruns:0 frame:0
          TX packets:10 errors:0 dropped:0 overruns:0 carrier:0
	   collisions:0 txqueuelen:0 
          RX bytes:976 (976.0 B)  TX bytes:796 (796.0 B)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
	  collisions:0 txqueuelen:1000 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
</code></pre><p>正如上所示，当我们执行ifconfig命令查看网络设备时，我会发现能看到的网卡“变少”了：只有两个。而我的宿主机则至少有四个网卡。这是怎么回事呢？</p><p>实际上，在setns()之后我看到的这两个网卡，正是我在前面启动的Docker容器里的网卡。也就是说，我新创建的这个/bin/bash进程，由于加入了该容器进程（PID=25686）的Network Namepace，它看到的网络设备与这个容器里是一样的，即：/bin/bash进程的网络设备视图，也被修改了。</p><p>而一旦一个进程加入到了另一个Namespace当中，在宿主机的Namespace文件上，也会有所体现。</p><p>在宿主机上，你可以用ps指令找到这个set_ns程序执行的/bin/bash进程，其真实的PID是28499：</p><pre><code># 在宿主机上
ps aux | grep /bin/bash
root     28499  0.0  0.0 19944  3612 pts/0    S    14:15   0:00 /bin/bash
</code></pre><p>这时，如果按照前面介绍过的方法，查看一下这个PID=28499的进程的Namespace，你就会发现这样一个事实：</p><pre><code>$ ls -l /proc/28499/ns/net
lrwxrwxrwx 1 root root 0 Aug 13 14:18 /proc/28499/ns/net -&gt; net:[4026532281]

$ ls -l  /proc/25686/ns/net
lrwxrwxrwx 1 root root 0 Aug 13 14:05 /proc/25686/ns/net -&gt; net:[4026532281]
</code></pre><p>在/proc/[PID]/ns/net目录下，这个PID=28499进程，与我们前面的Docker容器进程（PID=25686）指向的Network Namespace文件完全一样。这说明这两个进程，共享了这个名叫net:[4026532281]的Network Namespace。</p><p>此外，Docker还专门提供了一个参数，可以让你启动一个容器并“加入”到另一个容器的Network Namespace里，这个参数就是-net，比如:</p><pre><code>$ docker run -it --net container:4ddf4638572d busybox ifconfig
</code></pre><p>这样，我们新启动的这个容器，就会直接加入到ID=4ddf4638572d的容器，也就是我们前面的创建的Python应用容器（PID=25686）的Network Namespace中。所以，这里ifconfig返回的网卡信息，跟我前面那个小程序返回的结果一模一样，你也可以尝试一下。</p><p>而如果我指定–net=host，就意味着这个容器不会为进程启用Network Namespace。这就意味着，这个容器拆除了Network Namespace的“隔离墙”，所以，它会和宿主机上的其他普通进程一样，直接共享宿主机的网络栈。这就为容器直接操作和使用宿主机网络提供了一个渠道。</p><p><strong>转了一个大圈子，我其实是为你详细解读了docker exec这个操作背后，Linux Namespace更具体的工作原理。</strong></p><p><strong>这种通过操作系统进程相关的知识，逐步剖析Docker容器的方法，是理解容器的一个关键思路，希望你一定要掌握。</strong></p><p>现在，我们再一起回到前面提交镜像的操作docker commit上来吧。</p><p>docker commit，实际上就是在容器运行起来后，把最上层的“可读写层”，加上原先容器镜像的只读层，打包组成了一个新的镜像。当然，下面这些只读层在宿主机上是共享的，不会占用额外的空间。</p><p>而由于使用了联合文件系统，你在容器里对镜像rootfs所做的任何修改，都会被操作系统先复制到这个可读写层，然后再修改。这就是所谓的：Copy-on-Write。</p><p>而正如前所说，Init层的存在，就是为了避免你执行docker commit时，把Docker自己对/etc/hosts等文件做的修改，也一起提交掉。</p><p>有了新的镜像，我们就可以把它推送到Docker Hub上了：</p><pre><code>$ docker push geektime/helloworld:v2
</code></pre><p>你可能还会有这样的问题：我在企业内部，能不能也搭建一个跟Docker Hub类似的镜像上传系统呢？</p><p>当然可以，这个统一存放镜像的系统，就叫作Docker Registry。感兴趣的话，你可以查看<a href="https://docs.docker.com/registry/">Docker的官方文档</a>，以及<a href="https://github.com/goharbor/harbor">VMware的Harbor项目</a>。</p><p><span class="orange">最后，我再来讲解一下Docker项目另一个重要的内容：Volume（数据卷）。</span></p><p>前面我已经介绍过，容器技术使用了rootfs机制和Mount Namespace，构建出了一个同宿主机完全隔离开的文件系统环境。这时候，我们就需要考虑这样两个问题：</p><ol>
<li>
<p>容器里进程新建的文件，怎么才能让宿主机获取到？</p>
</li>
<li>
<p>宿主机上的文件和目录，怎么才能让容器里的进程访问到？</p>
</li>
</ol><p>这正是Docker Volume要解决的问题：<strong>Volume机制，允许你将宿主机上指定的目录或者文件，挂载到容器里面进行读取和修改操作。</strong></p><p>在Docker项目里，它支持两种Volume声明方式，可以把宿主机目录挂载进容器的/test目录当中：</p><pre><code>$ docker run -v /test ...
$ docker run -v /home:/test ...
</code></pre><p>而这两种声明方式的本质，实际上是相同的：都是把一个宿主机的目录挂载进了容器的/test目录。</p><p>只不过，在第一种情况下，由于你并没有显示声明宿主机目录，那么Docker就会默认在宿主机上创建一个临时目录 /var/lib/docker/volumes/[VOLUME_ID]/_data，然后把它挂载到容器的/test目录上。而在第二种情况下，Docker就直接把宿主机的/home目录挂载到容器的/test目录上。</p><p>那么，Docker又是如何做到把一个宿主机上的目录或者文件，挂载到容器里面去呢？难道又是Mount Namespace的黑科技吗？</p><p>实际上，并不需要这么麻烦。</p><p>在《白话容器基础（三）：深入理解容器镜像》的分享中，我已经介绍过，当容器进程被创建之后，尽管开启了Mount Namespace，但是在它执行chroot（或者pivot_root）之前，容器进程一直可以看到宿主机上的整个文件系统。</p><p>而宿主机上的文件系统，也自然包括了我们要使用的容器镜像。这个镜像的各个层，保存在/var/lib/docker/aufs/diff目录下，在容器进程启动后，它们会被联合挂载在/var/lib/docker/aufs/mnt/目录中，这样容器所需的rootfs就准备好了。</p><p>所以，我们只需要在rootfs准备好之后，在执行chroot之前，把Volume指定的宿主机目录（比如/home目录），挂载到指定的容器目录（比如/test目录）在宿主机上对应的目录（即/var/lib/docker/aufs/mnt/[可读写层ID]/test）上，这个Volume的挂载工作就完成了。</p><p>更重要的是，由于执行这个挂载操作时，“容器进程”已经创建了，也就意味着此时Mount Namespace已经开启了。所以，这个挂载事件只在这个容器里可见。你在宿主机上，是看不见容器内部的这个挂载点的。这就<strong>保证了容器的隔离性不会被Volume打破</strong>。</p><blockquote>
<p>注意：这里提到的"容器进程"，是Docker创建的一个容器初始化进程(dockerinit)，而不是应用进程(ENTRYPOINT + CMD)。dockerinit会负责完成根目录的准备、挂载设备和目录、配置hostname等一系列需要在容器内进行的初始化操作。最后，它通过execv()系统调用，让应用进程取代自己，成为容器里的PID=1的进程。</p>
</blockquote><p>而这里要使用到的挂载技术，就是Linux的<strong>绑定挂载（bind mount）机制</strong>。它的主要作用就是，允许你将一个目录或者文件，而不是整个设备，挂载到一个指定的目录上。并且，这时你在该挂载点上进行的任何操作，只是发生在被挂载的目录或者文件上，而原挂载点的内容则会被隐藏起来且不受影响。</p><p>其实，如果你了解Linux 内核的话，就会明白，绑定挂载实际上是一个inode替换的过程。在Linux操作系统中，inode可以理解为存放文件内容的“对象”，而dentry，也叫目录项，就是访问这个inode所使用的“指针”。</p><p><img src="https://static001.geekbang.org/resource/image/95/c6/95c957b3c2813bb70eb784b8d1daedc6.png?wh=1497*342" alt=""><br>
正如上图所示，mount --bind /home /test，会将/home挂载到/test上。其实相当于将/test的dentry，重定向到了/home的inode。这样当我们修改/test目录时，实际修改的是/home目录的inode。这也就是为何，一旦执行umount命令，/test目录原先的内容就会恢复：因为修改真正发生在的，是/home目录里。</p><p><strong>所以，在一个正确的时机，进行一次绑定挂载，Docker就可以成功地将一个宿主机上的目录或文件，不动声色地挂载到容器中。</strong></p><p>这样，进程在容器里对这个/test目录进行的所有操作，都实际发生在宿主机的对应目录（比如，/home，或者/var/lib/docker/volumes/[VOLUME_ID]/_data）里，而不会影响容器镜像的内容。</p><p>那么，这个/test目录里的内容，既然挂载在容器rootfs的可读写层，它会不会被docker commit提交掉呢？</p><p>也不会。</p><p>这个原因其实我们前面已经提到过。容器的镜像操作，比如docker commit，都是发生在宿主机空间的。而由于Mount Namespace的隔离作用，宿主机并不知道这个绑定挂载的存在。所以，在宿主机看来，容器中可读写层的/test目录（/var/lib/docker/aufs/mnt/[可读写层ID]/test），<strong>始终是空的。</strong></p><p>不过，由于Docker一开始还是要创建/test这个目录作为挂载点，所以执行了docker commit之后，你会发现新产生的镜像里，会多出来一个空的/test目录。毕竟，新建目录操作，又不是挂载操作，Mount Namespace对它可起不到“障眼法”的作用。</p><p>结合以上的讲解，我们现在来亲自验证一下：</p><p>首先，启动一个helloworld容器，给它声明一个Volume，挂载在容器里的/test目录上：</p><pre><code>$ docker run -d -v /test helloworld
cf53b766fa6f
</code></pre><p>容器启动之后，我们来查看一下这个Volume的ID：</p><pre><code>$ docker volume ls
DRIVER              VOLUME NAME
local               cb1c2f7221fa9b0971cc35f68aa1034824755ac44a034c0c0a1dd318838d3a6d
</code></pre><p>然后，使用这个ID，可以找到它在Docker工作目录下的volumes路径：</p><pre><code>$ ls /var/lib/docker/volumes/cb1c2f7221fa/_data/
</code></pre><p>这个_data文件夹，就是这个容器的Volume在宿主机上对应的临时目录了。</p><p>接下来，我们在容器的Volume里，添加一个文件text.txt：</p><pre><code>$ docker exec -it cf53b766fa6f /bin/sh
cd test/
touch text.txt
</code></pre><p>这时，我们再回到宿主机，就会发现text.txt已经出现在了宿主机上对应的临时目录里：</p><pre><code>$ ls /var/lib/docker/volumes/cb1c2f7221fa/_data/
text.txt
</code></pre><p>可是，如果你在宿主机上查看该容器的可读写层，虽然可以看到这个/test目录，但其内容是空的（关于如何找到这个AuFS文件系统的路径，请参考我上一次分享的内容）：</p><pre><code>$ ls /var/lib/docker/aufs/mnt/6780d0778b8a/test
</code></pre><p>可以确认，容器Volume里的信息，并不会被docker commit提交掉；但这个挂载点目录/test本身，则会出现在新的镜像当中。</p><p>以上内容，就是Docker Volume的核心原理了。</p><h2>总结</h2><p>在今天的这次分享中，我用了一个非常经典的Python应用作为案例，讲解了Docke容器使用的主要场景。熟悉了这些操作，你也就基本上摸清了Docker容器的核心功能。</p><p>更重要的是，我着重介绍了如何使用Linux Namespace、Cgroups，以及rootfs的知识，对容器进行了一次庖丁解牛似的解读。</p><p>借助这种思考问题的方法，最后的Docker容器，我们实际上就可以用下面这个“全景图”描述出来：</p><p><img src="https://static001.geekbang.org/resource/image/31/e5/3116751445d182687ce496f2825117e5.jpg?wh=2398*1419" alt=""></p><p>这个容器进程<code>“python app.py”</code>，运行在由Linux Namespace和Cgroups构成的隔离环境里；而它运行所需要的各种文件，比如python，<code>app.py</code>，以及整个操作系统文件，则由多个联合挂载在一起的rootfs层提供。</p><p>这些rootfs层的最下层，是来自Docker镜像的只读层。</p><p>在只读层之上，是Docker自己添加的Init层，用来存放被临时修改过的/etc/hosts等文件。</p><p>而rootfs的最上层是一个可读写层，它以Copy-on-Write的方式存放任何对只读层的修改，容器声明的Volume的挂载点，也出现在这一层。</p><p>通过这样的剖析，对于曾经“神秘莫测”的容器技术，你是不是感觉清晰了很多呢？</p><h2>思考题</h2><ol>
<li>
<p>你在查看Docker容器的Namespace时，是否注意到有一个叫cgroup的Namespace？它是Linux 4.6之后新增加的一个Namespace，你知道它的作用吗？</p>
</li>
<li>
<p>如果你执行docker run -v /home:/test的时候，容器镜像里的/test目录下本来就有内容的话，你会发现，在宿主机的/home目录下，也会出现这些内容。这是怎么回事？为什么它们没有被绑定挂载隐藏起来呢？（提示：Docker的“copyData”功能）</p>
</li>
<li>
<p>请尝试给这个Python应用加上CPU和Memory限制，然后启动它。根据我们前面介绍的Cgroups的知识，请你查看一下这个容器的Cgroups文件系统的设置，是不是跟我前面的讲解一致。</p>
</li>
</ol><p>感谢你的收听，欢迎你给我留言，也欢迎分享给更多的朋友一起阅读。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/57/4f/6fb51ff1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>一步</span>
  </div>
  <div class="_2_QraFYR_0">这样把原理刨根究底的讲解出来，很好，理解的很透彻</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-10 08:59:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/09/da/2fbd0760.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>兽医</span>
  </div>
  <div class="_2_QraFYR_0">讲得非常不错，曾经翻遍了几乎所有Docker官方文档，都没教程中来得深刻，谢谢。。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-13 13:29:51</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ch_ort</span>
  </div>
  <div class="_2_QraFYR_0">容器的本质是一个进程，只不过这个进程加上了视图上的隔离和资源上的限制。 就容器本质而言，它并没有在宿主机上启动一个“容器进程”，它启动的还是用户原来要启动的应用程序，只不过这个应用程序上加了视图隔离和资源限制。虚拟机也能实现视图隔离和资源限制，但它底层的技术实现与容器不同，在宿主机上你能看到这样一个“虚拟机进程”。因此，与容器相比，虚拟机会带来更多的性能损耗，主要在（1）虚拟机本身的进程需要占用一部分资源 （2）与底层硬件交互的时候（网络、磁盘I&#47;O）都需要经过虚拟化软件拦截，会有损耗。但是它比容器有更好的隔离性和安全性。<br><br>容器使用的底层技术：（1）视图隔离：Namespace （2）资源限制：cgropus<br><br>对于Docker项目来说，它最核心的原理实际就是为待创建的用户进程：<br>1、启用Linux Namespace配置：视图隔离<br>2、设置指定的Cgroups参数：资源限制<br>3、切换进程的根目录（Change Root）：容器镜像生效，以实现环境一致性。所谓容器镜像，本质就是容器的根文件系统(rootfs)。<br><br>Docker容器的增量rootfs：即下层已经生成的永远不会改变，所有的修改都通过在上层叠加。比如，删除A文件，就是在上层添加一个“白障”，让系统无法读取到下层A文件。修改则是先copy一个备份到新的层（新老的文件可能都在可读写层），然后读取的时候直接读取新的层。<br><br>Docker通过Volume来实现将宿主机的文件挂载到容器中<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-17 11:27:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/2a/07/74b3c14a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>黄文刚 </span>
  </div>
  <div class="_2_QraFYR_0">收货很大，感谢张磊！请教一个问题，请问在容器内部如何获取宿主机的IP? 谢谢。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 单靠容器，在隔离开的情况下是拿不到的。但是有了kubernetes之后这些系统信息都可以从环境变量里拿到。这个功能叫downwardapi</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-10 14:44:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJicBibF1UEOAhicickjCRXzVBDDyAMTw5C7EP3IPSKyTqWjvtmicB4AFu0B72BVyxFAVnNHrbx672p2Ow/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>多肉</span>
  </div>
  <div class="_2_QraFYR_0">一切从问题出发，根据问题理解答案，总结问题如下:<br>一、docker镜像如何制作的两种方式是什么？<br>二、容器既然是一个封闭的进程，那么外接程序是如何进入容器这个进程的呢？<br>三、docker commit对挂载点volume内容修改的影响是什么？<br>四、容器与宿主机如何进行文件读写？或volume是为了解决什么题？<br>五、Docker的copyData功能是什么？解决了什么问题？<br>六、bind mount机制是什么？<br>七、cgroup Namespace的作用是什么？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-17 20:14:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/60/85/f72f1d94.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>与路同飞</span>
  </div>
  <div class="_2_QraFYR_0">有预感专栏会破2万</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-10 18:43:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/5e/aa/b29ee77d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>robin</span>
  </div>
  <div class="_2_QraFYR_0">只会最后这个。<br>docker run -p 8000:80 --cpuset-cpus=&quot;1&quot; -m 500M helloworld   限制只能在cpu1上运行，内存限制为500M</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-21 11:55:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/1b/b4/a6db1c1e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>silver</span>
  </div>
  <div class="_2_QraFYR_0">所以说 docker exec 每次都会创建一个和容器共享namespace的新进程？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以这么理解，i.e. spawn a new process in existing namespace</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-10 10:51:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/9b/a8/6a391c66.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Leon📷</span>
  </div>
  <div class="_2_QraFYR_0">cgroupns作用<br>(1)可以限制容器的cgroup filesytem视图，使得在容器中也可以安全的使用cgroup；<br>(2)此外，会使容器迁移更加容易；在迁移时，&#47;proc&#47;self&#47;cgroup需要复制到目标机器，这要求容器的cgroup路径是唯一的，否则可能会与目标机器冲突。有了cgroupns，每个容器都有自己的cgroup filesystem视图，不用担心这种冲突。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-06 18:00:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/42/61/78d5d09a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>long904</span>
  </div>
  <div class="_2_QraFYR_0">图中不太明白为什么&#39;CMD&#39;属于只读层，那如果 dockfile 里面 yum install 并且 commit 的话，这些 CMD 执行的 yum 命令修改的内容还属于只读层？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 任何镜像里的内容，都属于只读层。commit之后的东西当然也属于只读层。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-13 11:30:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_9ca34e</span>
  </div>
  <div class="_2_QraFYR_0">；老师，有个地方还是不太明白,以下是你些的原文：<br>注意：这里提到的 &quot; 容器进程 &quot;，是 Docker 创建的一个容器初始化进程(dockerinit)，而不是应用进程 (ENTRYPOINT + CMD) dockerinit 会负责完成根目录的准备、挂载设备和目录、配置 hostname 等一系列需要在容器内进行的初始化操作。 最后，它通过 execv() 系统调用，让应用进程取代自己，成为容器里的 PID=1的进程。<br>我有以下疑问：<br> 1、dockerinit 是一个进程，完成初始化之后 ，让应用进程取代自己，那么dockerinit 进程会自动销毁么？<br>2、这里的dockerinit和应用进程在宿主机中是两个不同的进程id么？<br>3、执行 docker exec -it container bin&#47;sh   命令后，bin&#47;sh 命令创建的进程只是加入了container的namespace，但是从宿主机的角度它是一个独立的进程，只是共享了namespace信息么？<br>4、一个容器内部是否只有一个进程？，或者说容器只是一个房间（由namespace和cgroup组成），而其他的进程都是走进了这个房间，让大家以为这个房间就是一个系统，里面包含了这么多进程，其实在宿主机的角度他们都是一个个进程，只是共享了namespace和cgroup 而已，这样理解对么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是。替换当然就是为了保证pid不变。注意，容器里的其他进程都是1号进程的子进程，不存在独立的说法。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-21 10:49:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/43/32/0b81fc39.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>陶希阳</span>
  </div>
  <div class="_2_QraFYR_0">想知道云服务器等技术是不是也是通过namespace + cgroup实习的？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 当然不是，那可是正儿八经的虚拟化技术</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-10 10:43:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/8c/f7/a4de6f64.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>大卫</span>
  </div>
  <div class="_2_QraFYR_0">张老师，当Dockerfile中使用sh脚本启动，而不是exec启动java应用时，若通过docker stop不能优雅的停掉Java进程。查资料说可用trap接受信号处理，这个有什么其他好的解决办法没？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: trap sig term kill很标准的做法</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-01-04 18:45:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/22/72/bebffb62.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Han</span>
  </div>
  <div class="_2_QraFYR_0">有两个问题不是很清楚：<br>1）推荐一个docker运行一个进程，多出来的进程会变成“野”进程。那么如何理解这个野？是说不再受docker设置的namespace以及Cgroup约束了么？<br>2）docker exec是在相同的namespace下创建的新进程。那么这个进程是不是也是“野”的？相比与docker attach来讲。attach是不是没有新建进程，只是attach到docker中的主进程去？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-06 20:53:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/4e/e3/1a2e379f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>蔡鹏飞</span>
  </div>
  <div class="_2_QraFYR_0">docker run 时指定-v挂载宿主机目录到容器目录，即使容器原有目录内有数据，也会被我宿主机目录数据替代的呀。难道是和我使用的storage-driver有关？我用的是overlay存储。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个行为其实是可配置的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-12 23:00:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/6c/47/57b6c4e8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>帅</span>
  </div>
  <div class="_2_QraFYR_0">老师好，又来问您问题了。生产中有没有什么好的工具管理本地的docker registry，比如磁盘满了想清理等等的操作。期待老师的回复，谢谢！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: harbor啊</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-10-18 17:06:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/5a/56/115c6433.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jssfy</span>
  </div>
  <div class="_2_QraFYR_0">请问docker挂载有何限制没，是否随便一个目录都可以挂载？在容器里应该是root用户，岂不是可以对目录无节制地操作，哪怕原本主机目录中有些文件并不允许当前用户访问？是否可以相应限制</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 无限制。至于用户权限，是有user namespace可以做一定的限制。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-11 01:11:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/98/b1/f89a84d0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Tokamak</span>
  </div>
  <div class="_2_QraFYR_0">对于linux大部分容器做不到在运行容器中动态添加宿主机目录，那在什么特定场合下可以做到呢？给个大致思路即可，谢谢。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果你用的是katacontainers这种基于虚拟化的容器，才可以实现。但原生其实也不提供这个功能。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-10 17:28:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/b3/c5/7fc124e2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Liam</span>
  </div>
  <div class="_2_QraFYR_0">这节干货满满啊</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2018-09-10 08:43:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/AYCIZYjDO4UAMhqNYNElcNiczfuD2qU6VkcEmSugImsq5bjicRNTQE1dJhPIuUDunzxSLexDoYazK7QadT3AekJw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_84025d</span>
  </div>
  <div class="_2_QraFYR_0">思考题的第二题是不是讲的有误啊，实验了一下并不会复制容器文件。<br>上网查到的：docker容器持久化数据有两种方式，一种是bind volume，一种是volume。bind volume就是如文章讲到的，会发生inode替换，无论宿主机目录是否为空，此方法都会隐藏容器中的挂载点的内容；volume的方式，容器中已有目录，则会将目录中的文件copy到volume中。<br><br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-02 10:12:17</div>
  </div>
</div>
</div>
</li>
</ul>