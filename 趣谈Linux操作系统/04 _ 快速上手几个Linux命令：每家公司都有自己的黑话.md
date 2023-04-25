<audio title="04 _ 快速上手几个Linux命令：每家公司都有自己的黑话" src="https://static001.geekbang.org/resource/audio/de/18/de5b946452e8ab098fe6292e90372018.mp3" controls="controls"></audio> 
<p>如果你还没有上手用过Linux，那么接下来的课程，你可能会感受到困惑。因为没有一手的体验，你可能很难将Linux的机制和你的使用行为关联起来。所以这一节，咱们先介绍几个上手Linux的命令，通过这些命令，我们试试先把Linux用起来。</p><p>为什么我把Linux命令称为“黑话”呢？就像上一节我们介绍的，Linux操作系统有很多功能，我们有很多种方式可以使用这些功能，其中最简单和直接的方式就是<strong>命令行</strong>（Command Line）。命令行就相当于你请求服务使用的专业术语。干任何事情，第一步就是学会使用正确的术语。这样，Linux作为服务方，才能听懂。这些术语可不就是“黑话”吗？</p><p>Window系统你肯定很熟悉吧？现在，我就沿着你使用Windows的习惯，来给你介绍相应的Linux命令。</p><h2>用户与密码</h2><p>当我们打开一个新系统的时候，第一件要做的事就是登录。系统默认有一个Administrator用户，也就是系统管理员，它的权限很大，可以在这个系统上干任何事。Linux上面也有一个类似的用户，我们叫Root。同样，它也具有最高的操作权限。</p><p>接下来，你需要输入密码了。密码从哪里来呢？对于Windows来讲，在你安装操作系统的过程中，会让你设置一下Administrator的密码；对于Linux，Root的密码同样也是在安装过程中设置的。</p><!-- [[[read_end]]] --><p><img src="https://static001.geekbang.org/resource/image/ee/76/ee95d03b1390ae08ca9c752621b03476.png?wh=613*318" alt=""></p><p>对于Windows，你设好之后，可以多次修改这个密码。比如说，我们在控制面板的账户管理里面就可以完成这个操作。但是对于Linux呢？不好意思，没有这么一个统一的配置中心了。你需要使用命令来完成这件事情。这个命令很好记，<span class="orange">passwd</span>，其实就是password的简称。</p><pre><code># passwd
Changing password for user root.
New password:
</code></pre><p>按照这个命令，我们就可以输入新密码啦。</p><p>在Windows里，除了Administrator之外，我们还可以创建一个以自己名字命名的用户。那在Linux里可不可以创建其他用户呢？当然可以了，我们同样需要一个命令<span class="orange">useradd</span>。</p><pre><code> useradd cliu8
</code></pre><p>执行这个命令，一个用户就被创建了。它不会弹出什么让你输入密码之类的页面，就会直接返回了。因为接下来你需要自己调用passwd cliu8来设置密码，再进行登录。</p><p>在Windows里设置用户的时候，用户有一个“组”的概念。你可能没注意过，不过我一说名字你估计就能想起来了，比如“Adminsitrator组”“Guests组”“Power User组”等等。同样，Linux里也是分组的。前面我们创建用户的时候，没有说加入哪个组，于是默认就会创建一个同名的组。</p><p>能不能在创建用户的时候就指定属于哪个组呢？我们来试试。我们可以使用-h参数看一下，使用useradd这个命令，有没有相应的选项。</p><pre><code>[root@deployer ~]# useradd -h
Usage: useradd [options] LOGIN
       useradd -D
       useradd -D [options]


Options:
  -g, --gid GROUP               name or ID of the primary group of the new account
</code></pre><p>一看还真有这个选项。以后命令不会用的时候，就可以通过-h参数看一下，它的意思是help。</p><p>如果想看更加详细的文档，你可以通过man useradd获得，细细阅读。</p><p><img src="https://static001.geekbang.org/resource/image/17/2d/179b8fdca3d8d57e8f1d32f3aab60a2d.png?wh=760*451" alt=""></p><p>上一节我们说过，Linux里是“命令行+文件”模式。对于用户管理来说，也是一样的。咱们通过命令创建的用户，其实是放在/etc/passwd文件里的。这是一个文本文件。我们可以通过cat命令，将里面的内容输出在命令行上。组的信息我们放在/etc/group文件中。</p><pre><code># cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
......
cliu8:x:1000:1000::/home/cliu8:/bin/bash


# cat /etc/group
root:x:0:
......
cliu8:x:1000:
</code></pre><p>在/etc/passwd文件里，我们可以看到root用户和咱们刚创建的cliu8用户。x的地方应该是密码，密码当然不能放在这里，不然谁都知道了。接下来是用户ID和组ID，这和/etc/group里面就对应上了。</p><p>/root和/home/cliu8是什么呢？它们分别是root用户和cliu8用户的主目录。主目录是用户登录进去后默认的路径。其实Windows里面也是这样的。当我们打开文件夹浏览器的时候，左面会有“文档”“图片”“下载”等文件夹，路径在C:\Users\cliu8下面。要注意，同一台电脑，不同的用户情况会不一样。</p><p><img src="https://static001.geekbang.org/resource/image/d2/a7/d21ce3cd2ade7b71300df6a805b45aa7.png?wh=452*383" alt=""></p><p>/bin/bash的位置是用于配置登录后的默认交互命令行的，不像Windows，登录进去是界面，其实就是explorer.exe。而Linux登录后的交互命令行是一个解析脚本的程序，这里配置的是/bin/bash。</p><h2>浏览文件</h2><p>终于登录进来啦，接下来你可以在文件系统里面随便逛一逛、看一看了。</p><p>可以看到，Linux的文件系统和Windows是一样的，都是用文件夹把文件组织起来，形成一个树形的结构。这一点没有什么差别。只不过在Linux下面，大多数情况，我们需要通过命令行来查看Linux的文件。</p><p>其实在Windows下也有命令行，例如<span class="orange">cd</span>就是change directory，就是切换目录；cd .表示切换到当前目录；cd ..表示切换到上一级目录；使用dir，可以列出当前目录下的文件。Linux基本也是这样，只不过列出当前目录下的文件我们用的是<span class="orange">ls</span>，意思是list。</p><p><img src="https://static001.geekbang.org/resource/image/27/2e/27cc0efe8d33b730eba8aee7d51cda2e.png?wh=927*439" alt=""></p><p>我们常用的是ls -l，也就是用列表的方式列出文件。</p><pre><code># ls -l
drwxr-xr-x 6 root root    4096 Oct 20  2017 apt
-rw-r--r-- 1 root root     211 Oct 20  2017 hosts
</code></pre><p>其中第一个字段的第一个字符是<strong>文件类型</strong>。如果是“-”，表示普通文件；如果是d，就表示目录。当然还有很多种文件类型，咱们后面遇到的时候再说，你现在先记住我说的这两个就行了。</p><p>第一个字段剩下的9个字符是<strong>模式</strong>，其实就是<strong>权限位</strong>（access permission bits）。3个一组，每一组rwx表示“读（read）”“写（write）”“执行（execute）”。如果是字母，就说明有这个权限；如果是横线，就是没有这个权限。</p><p>这三组分别表示文件所属的用户权限、文件所属的组权限以及其他用户的权限。例如，上面的例子中，-rw-r–r--就可以翻译为，这是一个普通文件，对于所属用户，可读可写不能执行；对于所属的组，仅仅可读；对于其他用户，也是仅仅可读。如果想改变权限，可以使用命令chmod 711 hosts。</p><p>第二个字段是<strong>硬链接</strong>（hard link）<strong>数目</strong>，这个比较复杂，讲文件的时候我会详细说。</p><p>第三个字段是<strong>所属用户</strong>，第四个字段是<strong>所属组</strong>。第五个字段是文件的大小，第六个字段是<strong>文件被修改的日期</strong>，最后是<strong>文件名</strong>。你可以通过命令<span class="orange">chown</span>改变所属用户，<span class="orange">chgrp</span>改变所属组。</p><h2>安装软件</h2><p>好了，你现在应该会浏览文件夹了，接下来应该做什么呢？当然是开始安装那些“装机必备”的软件啦！</p><p>在Windows下面，在没有类似软件管家的软件之前，我们其实都是在网上下载installer，然后再进行安装的。</p><p>就以我们经常要安装的JDK为例子。应该去哪里下载呢？为了安全起见，一般去官网比较好。如果你去JDK的官网，它会给你一个这样的列表。</p><p><img src="https://static001.geekbang.org/resource/image/5e/02/5e54fe2dba0e86e14a7a92d9ea46c202.jpg?wh=1849*877" alt=""></p><p>对于Windows系统，最方便的方式就是下载exe，也就是安装文件。下载后我们直接双击安装即可。</p><p>对于Linux来讲，也是类似的方法，你可以下载rpm或者deb。这个就是Linux下面的安装包。为什么有两种呢？因为Linux现在常用的有两大体系，一个是CentOS体系，一个是Ubuntu体系，前者使用rpm，后者使用deb。</p><p>在Linux上面，没有双击安装这一说，因此想要安装，我们还得需要命令。CentOS下面使用<span class="orange">rpm -i jdk-XXX_linux-x64_bin.rpm</span>进行安装，Ubuntu下面使用<span class="orange">dpkg -i jdk-XXX_linux-x64_bin.deb</span>。其中-i就是install的意思。</p><p>在Windows下面，控制面板里面有程序管理，我们可以查看目前安装了哪些软件，可以删除这些软件。</p><p><img src="https://static001.geekbang.org/resource/image/4c/9b/4c0cddd6f5ea77bc4aeabc135e6e8a9b.png?wh=575*453?wh=575*453" alt=""></p><p>在Linux下面，凭借<span class="orange">rpm -qa</span>和<span class="orange">dpkg -l</span>就可以查看安装的软件列表，-q就是query，a就是all，-l的意思就是list。</p><p>如果真的去运行的话，你会发现这个列表很长很长，很难找到你安装的软件。如果你知道要安装的软件包含某个关键词，可以用一个很好用的搜索工具grep。</p><p><span class="orange">rpm -qa | grep jdk</span>，这个命令是将列出来的所有软件形成一个输出。| 是管道，用于连接两个程序，前面rpm -qa的输出就放进管道里面，然后作为grep的输入，grep将在里面进行搜索带关键词jdk的行，并且输出出来。grep支持正则表达式，因此搜索的时候很灵活，再加上管道，这是一个很常用的模式。同理<span class="orange">dpkg -l | grep jdk</span>也是能够找到的。</p><p>如果你不知道关键词，可以使用<span class="orange">rpm -qa | more</span>和<span class="orange">rpm -qa | less</span>这两个命令，它们可以将很长的结果分页展示出来。这样你就可以一个个来找了。</p><p>我们还是利用管道的机制。more是分页后只能往后翻页，翻到最后一页自动结束返回命令行，less是往前往后都能翻页，需要输入q返回命令行，q就是quit。</p><p>如果要删除，可以用<span class="orange">rpm -e</span>和<span class="orange">dpkg -r</span>。-e就是erase，-r就是remove。</p><p>我们刚才说的都是没有软件管家的情况，后来Windows上有了软件管家，就方便多了。我们直接搜索一下，然后点击安装就行了。</p><p><img src="https://static001.geekbang.org/resource/image/4c/9b/4c0cddd6f5ea77bc4aeabc135e6e8a9b.png?wh=575*453?wh=575*453" alt=""></p><p>Linux也有自己的软件管家，CentOS下面是yum，Ubuntu下面是apt-get。</p><p>你可以根据关键词搜索，例如搜索<span class="orange">jdk</span>、<span class="orange">yum search jdk</span>和<span class="orange">apt-cache search jdk</span>，可以搜索出很多很多可以安装的jdk版本。如果数目太多，你可以通过管道grep、more、less来进行过滤。</p><p>选中一个之后，我们就可以进行安装了。你可以用<span class="orange">yum install java-11-openjdk.x86_64</span>和<span class="orange">apt-get install openjdk-9-jdk</span>来进行安装。</p><p>安装以后，如何卸载呢？我们可以使用<span class="orange">yum erase java-11-openjdk.x86_64</span>和<span class="orange">apt-get purge openjdk-9-jdk</span>。</p><p>Windows上的软件管家会有一个统一的服务端，来保存这些软件，但是我们不知道服务端在哪里。而Linux允许我们配置从哪里下载这些软件的，地点就在配置文件里面。</p><p>对于CentOS来讲，配置文件在<span class="orange">/etc/yum.repos.d/CentOS-Base.repo</span>里。</p><pre><code>[base]
name=CentOS-$releasever - Base - 163.com
baseurl=http://mirrors.163.com/centos/$releasever/os/$basearch/
gpgcheck=1
gpgkey=http://mirrors.163.com/centos/RPM-GPG-KEY-CentOS-7

</code></pre><p>对于Ubuntu来讲，配置文件在<span class="orange">/etc/apt/sources.list</span>里。</p><pre><code>deb http://mirrors.163.com/ubuntu/ xenial main restricted universe multiverse
deb http://mirrors.163.com/ubuntu/ xenial-security main restricted universe multiverse
deb http://mirrors.163.com/ubuntu/ xenial-updates main restricted universe multiverse
deb http://mirrors.163.com/ubuntu/ xenial-proposed main restricted universe multiverse
deb http://mirrors.163.com/ubuntu/ xenial-backports main restricted universe multiverse
</code></pre><p>这里为什么都是163.com呢？因为Linux服务器遍布全球，不能都从一个地方下载，最好选一个就近的地方下载，例如在中国，选择163.com，就不用跨越重洋了。</p><p><strong>其实无论是先下载再安装，还是通过软件管家进行安装，都是下载一些文件，然后将这些文件放在某个路径下，然后在相应的配置文件中配置一下。</strong>例如，在Windows里面，最终会变成C:\Program Files下面的一个文件夹以及注册表里面的一些配置。对应Linux里面会放的更散一点。例如，主执行文件会放在/usr/bin或者/usr/sbin下面，其他的库文件会放在/var下面，配置文件会放在/etc下面。</p><p>所以其实还有一种简单粗暴的方法，就是将安装好的路径直接下载下来，然后解压缩成为一个整的路径。在JDK的安装目录中，Windows有jdk-XXX_Windows-x64_bin.zip，这是Windows下常用的压缩模式。Linux有jdk-XXX_linux-x64_bin.tar.gz，这是Linux下常用的压缩模式。</p><p>如何下载呢？Linux上面有一个工具wget，后面加上链接，就能从网上下载了。</p><p>下载下来后，我们就可以进行解压缩了。Windows下可以有winzip之类的解压缩程序，Linux下面默认会有tar程序。如果是解压缩zip包，就需要另行安装。</p><pre><code>yum install zip.x86_64 unzip.x86_64
apt-get install zip unzip
</code></pre><p>如果是tar.gz这种格式的，通过tar xvzf jdk-XXX_linux-x64_bin.tar.gz就可以解压缩了。</p><p>对于Windows上jdk的安装，如果采取这种下载压缩包的格式，需要在系统设置的环境变量配置里面设置<span class="orange">JAVA_HOME</span>和<span class="orange">PATH</span>。</p><p><img src="https://static001.geekbang.org/resource/image/ab/be/ab4e83ac1300658649989a2e016ac0be.png?wh=563*507" alt=""></p><p>在Linux也是一样的，通过tar解压缩之后，也需要配置环境变量，可以通过export命令来配置。</p><pre><code>export JAVA_HOME=/root/jdk-XXX_linux-x64
export PATH=$JAVA_HOME/bin:$PATH
</code></pre><p>export命令仅在当前命令行的会话中管用，一旦退出重新登录进来，就不管用了，有没有一个地方可以像Windows里面可以配置永远管用呢？</p><p>在当前用户的默认工作目录，例如/root或者/home/cliu8下面，有一个.bashrc文件，这个文件是以点开头的，这个文件默认看不到，需要ls -la才能看到，a就是all。每次登录的时候，这个文件都会运行，因而把它放在这里。这样登录进来就会自动执行。当然也可以通过source .bashrc手动执行。</p><p>要编辑.bashrc文件，可以使用文本编辑器vi，也可以使用更加友好的vim。如果默认没有安装，可以通过yum install vim及apt-get install vim进行安装。</p><p><strong>vim就像Windows里面的notepad一样，是我们第一个要学会的工具</strong>。要不然编辑、查看配置文件，这些操作你都没办法完成。vim是一个很复杂的工具，刚上手的时候，你只需要记住几个命令就行了。</p><p>vim hello，就是打开一个文件，名字叫hello。如果没有这个文件，就先创建一个。</p><p>我们其实就相当于打开了一个notepad。如果文件有内容，就会显示出来。移动光标的位置，通过上下左右键就行。如果想要编辑，就把光标移动到相应的位置，输入<span class="orange">i</span>，意思是insert。进入编辑模式，可以插入、删除字符，这些都和notepad很像。要想保存编辑的文本，我们使用<span class="orange">esc</span>键退出编辑模式，然后输入“:”，然后在“:”后面输入命令<span class="orange">w</span>，意思是write，这样就可以保存文本，冒号后面输入<span class="orange">q</span>，意思是quit，这样就会退出vim。如果编辑了，还没保存，不想要了，可以输入<span class="orange">q!</span>。</p><p>好了，掌握这些基本够用了，想了解更复杂的，你可以自己去看文档。</p><p>通过vim .bashrc，将export的两行加入后，输入:wq，写入并且退出，这样就编辑好了。</p><h2>运行程序</h2><p>好了，装好了程序，可以运行程序了。</p><p>我们都知道Windows下的程序，如果后缀名是exe，双击就可以运行了。</p><p>Linux不是根据后缀名来执行的。它的执行条件是这样的：只要文件有x执行权限，都能到文件所在的目录下，通过<span class="orange">./filename</span>运行这个程序。当然，如果放在PATH里设置的路径下面，就不用./了，直接输入文件名就可以运行了，Linux会帮你找。</p><p>这是<strong>Linux执行程序最常用的一种方式，通过shell在交互命令行里面运行</strong>。</p><p>这样执行的程序可能需要和用户进行交互，例如允许让用户输入，然后输出结果也打印到交互命令行上。这种方式比较适合运行一些简单的命令，例如通过date获取当前时间。这种模式的缺点是，一旦当前的交互命令行退出，程序就停止运行了。</p><p>这样显然不能用来运行那些需要“永远“在线的程序。比如说，运行一个博客程序，我总不能老是开着交互命令行，博客才可以提供服务。一旦我要去睡觉了，关了命令行，我的博客别人就不能访问了，这样肯定是不行的。</p><p>于是，我们就有了<strong>Linux运行程序的第二种方式，后台运行</strong>。</p><p>这个时候，我们往往使用<span class="orange">nohup</span>命令。这个命令的意思是no hang up（不挂起），也就是说，当前交互命令行退出的时候，程序还要在。</p><p>当然这个时候，程序不能霸占交互命令行，而是应该在后台运行。最后加一个&amp;，就表示后台运行。</p><p>另外一个要处理的就是输出，原来什么都打印在交互命令行里，现在在后台运行了，输出到哪里呢？输出到文件是最好的。</p><p>最终命令的一般形式为<span class="orange">nohup command &gt;out.file 2&gt;&amp;1 &amp;</span>。这里面，“1”表示文件描述符1，表示标准输出，“2”表示文件描述符2，意思是标准错误输出，“2&gt;&amp;1”表示标准输出和错误输出合并了。合并到哪里去呢？到out.file里。</p><p>那这个进程如何关闭呢？我们假设启动的程序包含某个关键字，那就可以使用下面的命令。</p><pre><code>ps -ef |grep 关键字  |awk '{print $2}'|xargs kill -9
</code></pre><p>从这个命令中，我们多少能看出shell的灵活性和精巧组合。</p><p>其中ps -ef可以单独执行，列出所有正在运行的程序，grep上面我们介绍过了，通过关键字找到咱们刚才启动的程序。</p><p>awk工具可以很灵活地对文本进行处理，这里的awk '{print $2}'是指第二列的内容，是运行的程序ID。我们可以通过xargs传递给kill -9，也就是发给这个运行的程序一个信号，让它关闭。如果你已经知道运行的程序ID，可以直接使用kill关闭运行的程序。</p><p>在Windows里面还有一种程序，称为服务。这是系统启动的时候就在的，我们可以通过控制面板的服务管理启动和关闭它。</p><p><img src="https://static001.geekbang.org/resource/image/f2/a6/f24f0f11bcb9a177861a4782ba1d82a6.png?wh=1064*512" alt=""></p><p>Linux也有相应的服务，这就是<strong>程序运行的第三种方式，以服务的方式运行</strong>。例如常用的数据库MySQL，就可以使用这种方式运行。</p><p>例如在Ubuntu中，我们可以通过apt-get install mysql-server的方式安装MySQL，然后通过命令<span class="orange">systemctl start mysql</span>启动MySQL，通过<span class="orange">systemctl enable mysql</span>设置开机启动。之所以成为服务并且能够开机启动，是因为在/lib/systemd/system目录下会创建一个XXX.service的配置文件，里面定义了如何启动、如何关闭。</p><p>在CentOS里有些特殊，MySQL被Oracle收购后，因为担心授权问题，改为使用MariaDB，它是MySQL的一个分支。通过命令<span class="orange">yum install mariadb-server mariadb</span>进行安装，命令<span class="orange">systemctl start mariadb</span>启动，命令<span class="orange">systemctl enable mariadb</span>设置开机启动。同理，会在/usr/lib/systemd/system目录下，创建一个XXX.service的配置文件，从而成为一个服务。</p><p>systemd的机制十分复杂，这里咱们不讨论。如果有兴趣，你可以自己查看相关文档。</p><p>最后咱们要学习的是如何关机和重启。这个就很简单啦。<span class="orange">shutdown -h now</span>是现在就关机，<span class="orange">reboot</span>就是重启。</p><h2>总结时刻</h2><p>好了，掌握这些基本命令足够你熟练操作Linux了。如果你是个初学者，这些命令估计看起来还是很多。我把今天这些基本的命令以及对应的操作总结了一下，方便你操作和查阅。</p><p>你不用可以去死记硬背，按照我讲的这个步骤，从设置用户和密码、浏览文件、安装软件，最后到运行程序，<strong>自己去操作几遍，再自己整理一遍</strong>，手脑并用，加深理解，巩固记忆，效果可能会更好。</p><p><img src="https://static001.geekbang.org/resource/image/88/e5/8855bb645d8ecc35c80aa89cde5d16e5.jpg?wh=3431*2125" alt=""></p><center><span class="reference">(建议保存查看清晰大图)</span></center><h2>课堂练习</h2><p>现在你应该已经学会了安装JDK和MySQL，你可以尝试搭建一个基于Java+MySQL的服务端应用，上手使用一下。</p><p>欢迎留言和我分享你的疑惑和见解，也欢迎你收藏本节内容，反复研读。你也可以把今天的内容分享给你的朋友，和他一起学习、进步。</p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/0c/69/3d2f3a58.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>李 P</span>
  </div>
  <div class="_2_QraFYR_0">留的题和本课讲的有啥关系啊？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 留的题目都是有用的，一方面是扩展阅读，一方面做了练习以后，对于当然和后面章节的理解会有帮助</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-08 09:05:17</div>
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
  <div class="_2_QraFYR_0">推荐一个工具给大家：https:&#47;&#47;github.com&#47;tldr-pages&#47;tldr<br>tl;dr 的意思是 too long; dont read，这个工具可以用来查常用的linux命令，比man更友好一些，大多数时候用这个就够了，找不到再man或者google。<br><br>➜  ~ tldr ls <br>Local data is older than two weeks, use --update to update it.<br><br><br>ls<br><br>List directory contents.<br><br>- List files one per line:<br>    ls -1<br><br>- List all files, including hidden files:<br>    ls -a<br><br>- Long format list (permissions, ownership, size and modification date) of all files:<br>    ls -la<br><br>- Long format list with size displayed using human readable units (KB, MB, GB):<br>    ls -lh<br><br>- Long format list sorted by size (descending):<br>    ls -lS<br><br>- Long format list of all files, sorted by modification date (oldest first):<br>    ls -ltr<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-03 16:49:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/75/69/791d0f5e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>rocedu</span>
  </div>
  <div class="_2_QraFYR_0">自己知道想干什么，但不知道哪个命令可以干怎么办？我们可以通过man -k key1|grep key2| grep  key...进行搜索，man -k 或apropos是我认为学习linux命令优先掌握的命令，这样你就可以自己搜索了，相当于google,baidu, 大家可以参考【别出心裁的Linux命令学习法】(https:&#47;&#47;www.cnblogs.com&#47;rocedu&#47;p&#47;4902411.html)，我更新了一下。刘老师用Windows和Linux对比的方式让初学者入门，也非常棒。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-03 05:43:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/18/f3/cd07e64c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lujg</span>
  </div>
  <div class="_2_QraFYR_0">.bash_profile是系统配置信息存储文件，写在里面的系统变量是所有用户共用的，而.bashrc是个人的配置信息存储文件，只是单用户有效。也就是说，配置了.bashrc后切换用户可能需要重新配置系统变量。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-04 09:13:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/75/69/791d0f5e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>rocedu</span>
  </div>
  <div class="_2_QraFYR_0">ubuntu 中现在都用apt了，apt = apt-get、apt-cache 和 apt-config 中最常用命令选项的集合。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-03 05:33:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/b1/58/7d4f968f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>plasmatium</span>
  </div>
  <div class="_2_QraFYR_0">老师不说下rm -rf &#47;和alias cd=&quot;rm -rf *&quot;吗，一是帮助小伙伴们避坑，二是get到黑话和笑点</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 哈哈</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-03 10:00:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/01/b9/73435279.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>学习学个屁</span>
  </div>
  <div class="_2_QraFYR_0">1 修改密码 passwd <br>2 新建用户 useradd userName  <br>   删除用户 userdel username<br><br>  添加的用户指定相应的group用户组<br>  useradd -g userName group<br><br>创建用户的时候，没有说加入哪个组，于是默认就会创建一个同名的组。<br><br><br>3 给用户设置密码 passwd password<br><br>通过命令创建的用户，其实是放在 &#47;etc&#47;passwd 文件里的<br>用户组的信息我们放在 &#47;etc&#47;group 文件<br>4   用户组<br>groupadd groupname　　添加用户组<br>groupdel groupname　　删除用户组<br><br>5 查看文件<br><br>ls -l 查看所有<br><br>如： drwxr-xr-x 6 root root 4096 Oct 20 2017 apt<br><br>文件权限 <br><br>第一个字段剩下的 9 个字符是模式<br>rwx 表示“读（read）”“写（write）”“执行（execute）”  - 表示没有权限<br><br>第二个字段是硬链接（hard link）数目<br><br>第三个字段是所属用户，第四个字段是所属组。第五个字段是文件的大小，第六个字段是文件被修改的日期，最后是文件名。<br><br>6 安装软件<br><br>CentOS 下面使用rpm -i jdk-XXX_linux-x64_bin.rpm进行安装<br><br>Ubuntu 下面使用dpkg -i jdk-XXX_linux-x64_bin.deb <br><br>其中 -i 就是 install 的意思。<br><br>或者 wget  url地址  下载 软件安装文件<br><br>7 管道技术 |  <br><br>rpm -qa | grep jdk<br><br>grep 表示 筛选搜索带关键词 jdk 的行，并且输出出来。grep 支持正则表达式。<br><br><br>不知道关键词，可以使用rpm -qa | more和rpm -qa | less这两个命令，它们可以将很长的结果分页展示出来<br><br>软件管家下载<br><br>软件管家，CentOS 下面是 yum，Ubuntu 下面是 apt-get。<br><br>例：<br>安装<br>yum install java-11-openjdk.x86_64<br>apt-get install openjdk-9-jdk<br><br>卸载<br>yum erase java-11-openjdk.x86_64<br>apt-get purge openjdk-9-jdk<br><br>软件管家<br><br>CentOS 来讲，配置文件在&#47;etc&#47;yum.repos.d&#47;CentOS-Base.repo里<br>Ubuntu 来讲，配置文件在&#47;etc&#47;apt&#47;sources.list里<br><br>8 解压<br><br>tar.gz tar vxzf   xxx.tar   解压 <br><br>9 编辑文件<br> vi或者vim   ：  vim  文件名<br><br>i 意思是 insert。进入编辑模式，可以插入、删除字符<br><br>先按 esc<br>然后：wq 保存 <br>然后 :q! 不保存<br><br><br> <br>10 启动<br>一般进入文件bin下 或者指定目录  .&#47;xxx 就可以启动 如：tomcat  启动 .&#47;start <br><br>后台运行 <br>nohup  xxx &amp; 表示当前启动的进程后台运行  如： nohup COMMAND &amp;<br><br>指定进程日志输出<br>nohup command &gt;out.file 2&gt;&amp;1 &amp;。这里面，“1”表示文件描述符 1，表示标准输出，“2”表示文件描述符 2，意思是标准错误输出，“2&gt;&amp;1”表示标准输出和错误输出合并了。合并到哪里去呢？到 out.file 里。<br>查看进程<br>启动后如何查看进程 ps -ef | grep jdk  查看 jdk的进程 <br><br>杀进程 <br>kill -9 pid  ，pid可以通过 ps -ef | grep jdk 查看 jdk的进程号<br><br>11 关机重启<br>shutdown -h now是现在就关机<br>reboot就是重启。<br><br><br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-10 21:40:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/a4/d7/5d2bfaa7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Aliliin</span>
  </div>
  <div class="_2_QraFYR_0">给大家推荐一个快速搭建各种版本 Linux 系统的开源软件，如果你电脑中安装了 docker ,可以通过浏览器访问你创建的 Linux 系统，这样子你就可以大胆的，实验各种命令了。比如 rm -rf *&#47;.<br>连接： https:&#47;&#47;github.com&#47;instantbox&#47;instantbox&#47;blob&#47;master&#47;docs&#47;README-zh_cn.md</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-03 15:40:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/82/bd/3d87ef93.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Kouei</span>
  </div>
  <div class="_2_QraFYR_0">有两个小想法：<br><br>1. 进入上级目录应该是 cd .. 而文中写成了cd ...<br>2. 查询命令的帮助用-h不一定总是能奏效，比如df dh等命令-h表示以人类可读的方式显示各类数值。用--help来打印帮助也许更普适一些吧。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 排版的原因吧。是的--help是对的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-03 03:51:17</div>
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
  <div class="_2_QraFYR_0">环境变量不是写在 .bash_profile里面吗？和.bashrc有区别吗？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-03 16:20:51</div>
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
  <div class="_2_QraFYR_0">其实没必要类比windows,  因为可能很多程序员用的是mac</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 哦。。。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-06 20:35:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/29/c2/6da531c3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>LOOK</span>
  </div>
  <div class="_2_QraFYR_0">好复杂，感觉坚持不下去了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 加油啊，后面更难</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-03 06:07:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c7/ba/4c449be2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>zhaozp</span>
  </div>
  <div class="_2_QraFYR_0">对比Windows，按照老师的梳理感觉这些命令容易记忆了，谢谢！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-03 08:27:00</div>
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
  <div class="_2_QraFYR_0">发布真准时，刚好0点0分~~！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-03 00:00:49</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/22/e1/7a/b206cded.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>人在江湖龙在江湖</span>
  </div>
  <div class="_2_QraFYR_0">systemd 手册内容太多，阮一峰总结的很好，可以看看：http:&#47;&#47;www.ruanyifeng.com&#47;blog&#47;2016&#47;03&#47;systemd-tutorial-commands.html</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-07 14:03:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/6b/8f/eba34b86.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>星光</span>
  </div>
  <div class="_2_QraFYR_0">老师我最近在Ubuntu下搭建Go的环境，看网上说的是在&#47;etc&#47;profile中添加export GOROOT和GOPATH，我很好奇Linux下这个&#47;etc&#47;profile是干嘛的，因为我用vim打开看了一下，就是一个脚本。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你知道&#47;etc&#47;profile和.bashrc的区别吗，面试经常考哦，可以系统的查一下</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-04 09:33:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/4e/38/3faa8377.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>登高</span>
  </div>
  <div class="_2_QraFYR_0">感觉自己所学的Linux在一课里就被讲了70%， 期待接下来的课程，够硬<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这门课主要讲内核，所以命令就不细讲了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-03 22:15:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/0c/ff/04e6beb9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>totoday</span>
  </div>
  <div class="_2_QraFYR_0">老师讲的相当系统和清晰了，虽然这些我基本都知道，但是当时摸索了挺久的。现在女朋友要学，由于学业压力较大，我没时间很系统的指导她，就买了老师的课，让她系统地学习，我也正好查漏补缺。期待后面的内容！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-03 13:18:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/57/6e/b6795c44.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>夜空中最亮的星</span>
  </div>
  <div class="_2_QraFYR_0">哈哈，全部掌握</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-03 21:36:13</div>
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
  <div class="_2_QraFYR_0">看完这篇后随意翻了一下 翻到了linux上raid–vheck对应的脚本 但是 看不太懂</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-03 11:36:30</div>
  </div>
</div>
</div>
</li>
</ul>