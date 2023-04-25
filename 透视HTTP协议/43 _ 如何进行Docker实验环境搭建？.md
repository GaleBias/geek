<audio title="43 _ 如何进行Docker实验环境搭建？" src="https://static001.geekbang.org/resource/audio/38/c7/3855a80ef278a46a10d97d775e2e18c7.mp3" controls="controls"></audio> 
<p>你好，我是Chrono。</p><p>《透视HTTP协议》这个专栏正式完结已经一年多了，感谢你的支持与鼓励。</p><p>这一年的时间下来，我发现专栏“实验环境的搭建”确实是一个比较严重的问题：虽然我已经尽量把Windows、macOS、Linux里的搭建步骤写清楚了，但因为具体的系统环境千差万别，总会有各式各样奇怪的问题出现，比如端口冲突、目录权限等等。</p><p>所以，为了彻底解决这个麻烦，我特意制作了一个Docker镜像，里面是完整可用的HTTP实验环境，下面我就来详细说一下该怎么用。</p><h2>安装Docker环境</h2><p>因为我不知道你对Docker是否了解，所以第一步我还是先来简单介绍一下它。</p><p>Docker是一种虚拟化技术，基于Linux的容器机制（Linux Containers，简称LXC），你可以把它近似地理解成是一个“轻量级的虚拟机”，只消耗较少的资源就能实现对进程的隔离保护。</p><p>使用Docker可以把应用程序和它相关的各种依赖（如底层库、组件等）“打包”在一起，这就是Docker镜像（Docker image）。Docker镜像可以让应用程序不再顾虑环境的差异，在任意的系统中以容器的形式运行（当然必须要基于Docker环境），极大地增强了应用部署的灵活性和适应性。</p><!-- [[[read_end]]] --><p>Docker是跨平台的，支持Windows、macOS、Linux等操作系统，在Windows、macOS上只要下载一个安装包，然后简单点几下鼠标就可以完成安装。</p><p><img src="https://static001.geekbang.org/resource/image/24/b2/2490f6a2a514710e0b683d6cdb4614b2.png?wh=1142*558" alt=""></p><p>下面我以Ubuntu为例，说一下在Linux上的安装方法。</p><p>你可以在Linux上用apt-get或者yum安装Docker，不过更好的方式是使用Docker官方提供的脚本，自动完成全套的安装步骤。</p><pre><code>curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun
</code></pre><p>因为Docker是国外网站，直接从官网安装速度可能比较慢。所以你还可以选择国内的镜像网站来加快速度，像这里我就使用“–mirror”选项指定了“某某云”。</p><p>Docker是C/S架构，安装之后还需要再执行一条命令启动它的服务。</p><pre><code>sudo service docker start    #Ubuntu启动docker服务
</code></pre><p>此外，操作Docker必须要有sudo权限，你可以用“usermod”命令把当前的用户加入“Docker”组里。如果图省事，也可以用sudo命令直接切换成root用户来操作。</p><pre><code>sudo usermod -aG docker ${USER}  #当前用户加入Docker组
sudo su -                        #或者直接用root用户
</code></pre><p>这些做完后，你需要执行命令“<strong>docker version</strong>”“<strong>docker info</strong>”来检查是否安装成功。比如下面这张图，显示的就是我使用的Docker环境，版本是“18.06.3-ce”。</p><p><img src="https://static001.geekbang.org/resource/image/d3/91/d3ba248bf564yy289a197dab8635c991.png?wh=1142*848" alt=""></p><h2>获取Docker镜像</h2><p>如果你已经安装好了Docker运行环境，现在就可以从Docker Hub上获取课程相应的Docker镜像文件了，用的是“<strong>docker pull</strong>”命令。</p><pre><code>docker pull chronolaw/http_study
</code></pre><p>这个镜像里打包了操作系统Ubuntu 18.04和最新的Openresty 1.17.8.2，还有项目的全部示例代码。为了方便你学习，我还在里面加入了Vim、Git、Telnet、Curl、Tcpdump等实用工具。</p><p>由于镜像里的东西多，所以体积比较大，下载需要一些时间，你要有点耐心。镜像下载完成之后，你可以用“Docker images”来查看结果，列出目前本地的所有镜像文件。</p><p><img src="https://static001.geekbang.org/resource/image/07/22/0779b1016002f4bdafc6a785c991ba22.png?wh=1142*75" alt=""></p><p>从图中你可以看到，这个镜像的名字是“chronolaw/http_study”，大小是645MB。</p><h2>启动Docker容器</h2><p>有了镜像文件，你就可以用“<strong>docker run</strong>”命令，从镜像启动一个容器了。</p><p>这里面就是我们完整的HTTP实验环境，不需要再操心这样、那样的问题了，做到了真正的“开箱即用”。</p><pre><code>docker run -it --rm chronolaw/http_study
</code></pre><p>对于上面这条命令，我还要稍微解释一下：“-it”参数表示开启一个交互式的Shell，默认使用的是bash；“–rm”参数表示容器是“用完即扔”，不保存容器实例，一旦退出Shell就会自动删除容器（但不会删除镜像），免去了管理容器的麻烦。</p><p>“docker run”之后，你就会像虚拟机一样进入容器的运行环境，这里就是Ubuntu 18.04，身份也自动变成了root用户，大概是下面这样的。</p><pre><code>docker run -it --rm chronolaw/http_study

root@8932f62c972:/#
</code></pre><p>项目的源码我放在了root用户目录下，你可以直接进入“<strong>http_study/www</strong>”目录，然后执行“<strong>run.sh</strong>”启动OpenResty服务（可参考<a href="https://time.geekbang.org/column/article/146833">第41讲</a>）。</p><pre><code>cd ~/http_study/www
./run.sh start
</code></pre><p>不过因为Docker自身的限制，镜像里的hosts文件不能直接添加“<a href="http://www.chrono.com">www.chrono.com</a>”等实验域名的解析。如果你想要在URI里使用域名，就必须在容器启动后手动修改hosts文件，用Vim或者cat都可以。</p><pre><code>vim /etc/hosts                        #手动编辑hosts文件
cat ~/http_study/hosts &gt;&gt; /etc/hosts  #cat追加到hosts末尾
</code></pre><p>另一种方式是在“docker run”的时候用“<strong>–add-host</strong>”参数，手动指定域名/IP的映射关系。</p><pre><code>docker run -it --rm --add-host=www.chrono.com:127.0.0.1 chronolaw/http_study 
</code></pre><p>保险起见，我建议你还是用第一种方式比较好。也就是启动容器后，用“cat”命令，把实验域名的解析追加到hosts文件里，然后再启动OpenResty服务。</p><pre><code>docker run -it --rm chronolaw/http_study

cat ~/http_study/hosts &gt;&gt; /etc/hosts
cd ~/http_study/www
./run.sh start
</code></pre><h2>在Docker容器里做实验</h2><p>把上面的工作都做完之后，我们的实验环境就算是完美地运行起来了，现在你就可以在里面任意验证各节课里的示例了，我来举几个例子。</p><p>不过在开始之前，我要提醒你一点，因为这个Docker镜像是基于Linux的，没有图形界面，所以只能用命令行（比如telnet、curl）来访问HTTP服务。当然你也可以查一下资料，让容器对外暴露80等端口（比如使用参数“–net=host”），在外部用浏览器来访问，这里我就不细说了。</p><p>先来看最简单的，<a href="https://time.geekbang.org/column/article/100124">第7讲</a>里的测试实验环境，用curl来访问localhost，会输出一个文本形式的HTML文件内容。</p><pre><code>curl http://localhost      #访问本机的HTTP服务
</code></pre><p><img src="https://static001.geekbang.org/resource/image/30/1a/30671d607d6cb74076c467bab1b95b1a.png?wh=1142*697" alt=""></p><p>然后我们来看<a href="https://time.geekbang.org/column/article/100513">第9讲</a>，用telnet来访问HTTP服务，输入“<strong>telnet 127.0.0.1 80</strong>”，回车，就进入了telnet界面。</p><p>Linux下的telnet操作要比Windows的容易一些，你可以直接把HTTP请求报文复制粘贴进去，再按两下回车就行了，结束telnet可以用“Ctrl+C”。</p><pre><code>GET /09-1 HTTP/1.1
Host:   www.chrono.com
</code></pre><p><img src="https://static001.geekbang.org/resource/image/8d/fa/8d40c02c57b13835c8dd92bda97fa3fa.png?wh=1142*778" alt=""></p><p>实验环境里测试HTTPS和HTTP/2也是毫无问题的，只要你按之前说的，正确修改了hosts域名解析，就可以用curl来访问，但要加上“<strong>-k</strong>”参数来忽略证书验证。</p><pre><code>curl https://www.chrono.com/23-1 -vk
curl https://www.metroid.net:8443/30-1 -vk
</code></pre><p><img src="https://static001.geekbang.org/resource/image/yy/94/yyff754d5fbe34cd2dfdb002beb00094.png?wh=1142*714" alt=""></p><p><img src="https://static001.geekbang.org/resource/image/69/51/69163cc988f8b95cf906cd4dbcyy3151.png?wh=1142*171" alt=""></p><p>这里要注意一点，因为Docker镜像里的Openresty 1.17.8.2内置了OpenSSL1.1.1g，默认使用的是TLS1.3，所以如果你想要测试TLS1.2的话，需要使用参数“<strong>–tlsv1.2</strong>”。</p><pre><code>curl https://www.chrono.com/30-1 -k --tlsv1.2
</code></pre><p><img src="https://static001.geekbang.org/resource/image/b6/aa/b62a2278b73a4f264a14a04f0058cbaa.png?wh=1142*167" alt=""></p><h2>在Docker容器里抓包</h2><p>到这里，课程中的大部分示例都可以运行了。最后我再示范一下在Docker容器里tcpdump抓包的用法。</p><p>首先，你要指定抓取的协议、地址和端口号，再用“-w”指定存储位置，启动tcpdump后按“Ctrl+Z”让它在后台运行。比如为了测试TLS1.3，就可以用下面的命令行，抓取HTTPS的443端口，存放到“/tmp”目录。</p><pre><code>tcpdump tcp port 443 -i lo -w /tmp/a.pcap
</code></pre><p>然后，我们执行任意的telnet或者curl命令，完成HTTP请求之后，输入“fg”恢复tcpdump，再按“Ctrl+C”，这样抓包就结束了。</p><p>对于HTTPS需要导出密钥的情形，你必须在curl请求的同时指定环境变量“SSLKEYLOGFILE”，不然抓包获取的数据无法解密，你就只能看到乱码了。</p><pre><code>SSLKEYLOGFILE=/tmp/a.log curl https://www.chrono.com/11-1 -k
</code></pre><p>我把完整的抓包过程截了个图，你可以参考一下。</p><p><img src="https://static001.geekbang.org/resource/image/40/40/40f3c47814a174a4d135316b7cfdcf40.png?wh=1142*487" alt=""></p><p>抓包生成的文件在容器关闭后就会消失，所以还要用“<strong>docker cp</strong>”命令及时从容器里拷出来（指定容器的ID，看提示符，或者用“docker ps -a”查看，也可以从GitHub仓库里获取43-1.pcap/43-1.log）。</p><pre><code>docker cp xxx:/tmp/a.pcap .  #需要指定容器的ID
docker cp xxx:/tmp/a.log .   #需要指定容器的ID
</code></pre><p>现在有了pcap文件和log文件，我们就可以用Wireshark来看网络数据，细致地分析HTTP/HTTPS通信过程了（HTTPS还需要设置一下Wireshark，见<a href="https://time.geekbang.org/column/article/110354">第26讲</a>）。</p><p><img src="https://static001.geekbang.org/resource/image/1a/bf/1ab18685ca765e8050b58ee76abd3cbf.png?wh=1142*438" alt=""></p><p>在这个包里，你可以清楚地看到，通信时使用的是TLS1.3协议，服务器选择的密码套件是TLS_AES_256_GCM_SHA384。</p><p>掌握了tcpdump的用法之后，你也可以再参考<a href="https://time.geekbang.org/column/article/110718">第27讲</a>，改改Nginx配置文件，自己抓包仔细研究TLS1.3协议的“supported_versions”“key_share”“server_name”等各个扩展协议。</p><h2>小结</h2><p>今天讲了Docker实验环境的搭建，我再小结一下要点。</p><ol>
<li>Docker是一种非常流行的虚拟化技术，可以近似地把它理解成是一个“轻量级的虚拟机”；</li>
<li>可以用“docker pull”命令从Docker Hub上获取课程相应的Docker镜像文件；</li>
<li>可以用“docker run”命令从镜像启动一个容器，里面是完整的HTTP实验环境，支持TLS1.3；</li>
<li>可以在Docker容器里任意验证各节课里的示例，但需要使用命令行形式的telnet或者curl；</li>
<li>抓包需要使用tcpdump，指定抓取的协议、地址和端口号；</li>
<li>对于HTTPS，需要指定环境变量“SSLKEYLOGFILE”导出密钥，再发送curl请求。</li>
</ol><p>很高兴时隔一年后再次与你见面，今天就到这里吧，期待下次HTTP/3发布时的相会。</p><p><img src="https://static001.geekbang.org/resource/image/26/e3/26bbe56074b40fd5c259f396ddcfd6e3.png?wh=750*892" alt=""></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/57/6e/ae14e53b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Howard.Wundt</span>
  </div>
  <div class="_2_QraFYR_0">首先祝老师教师节快乐！很期待着与老师的重逢。有个问题请教老师：轻量化虚拟机技术除了Docker 外还有其他选择吗？Docker 现在的政治化让人很不舒服。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 目前看来还没有同量级的对手，docker已经算是事实标准了，而且还有围绕它的很多生态，比如k8s，想要替代短时期内看不到可能性。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-10 11:10:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_78044b</span>
  </div>
  <div class="_2_QraFYR_0">老师确实很敬业，最近刚买了课程，一周多时间快速学习了一篇，开始的破冰篇确实特别基础，我还以为这门科是只针对的初学者的，没什么难度，后面讲http1.1， 2， 3， 安全篇等都还是比较有深度，感谢老师的付出。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，这个也跟课程的总体设计有关，一般都是要由浅入深循序渐进，尽量全面覆盖知识点。如果基础好可以跳过前面的，直接看后面感兴趣的部分。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-20 16:06:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/ea/27/a3737d61.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Shanks-王冲</span>
  </div>
  <div class="_2_QraFYR_0">谢谢老师的Docker tutorial quick guide，让我对Docker有了fist touch；前几天看Android开发的技术博客时，不知怎么地就跳到Docker官网，并瞧了瞧，没敢下载来玩；但今天又机缘巧合看到这篇文章，I just pulled，感谢老师！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: docker挺好玩的，我之前也是不太熟悉，偶然一接触，操作了几下就会用了，然后就研究了下去。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-07 19:04:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1b/51/a0/5db02ac2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>點點點，点顛</span>
  </div>
  <div class="_2_QraFYR_0">老师教师节快乐😁。感谢老师还记得我们</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 等以后http&#47;3时再相聚。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-10 09:05:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/26/eb/d7/90391376.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ifelse</span>
  </div>
  <div class="_2_QraFYR_0">打卡</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: nice</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-10 12:37:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/61/90/de8c61a0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>dongge</span>
  </div>
  <div class="_2_QraFYR_0">Chrono老师，是我在极客时间见过的最负责的老师。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: many thanks.</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-10-17 19:08:15</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/eHNmzejyQW9Ag5g3EELS1d9pTgJsvxC7CxSCxIFQqeFLXUDT52HWianQWzw14kaAT4P9UhTUSNficc9W5DlWZWJQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>silence</span>
  </div>
  <div class="_2_QraFYR_0">老师请问我在测试tcpdump抓包的时候，将您的命令粘贴进去<br>tcpdump tcp port 443 -i lo -w &#47;tmp&#47;a.pcap<br>报了以下错误怎么解决呢？<br>tcpdump: lo: SIOCETHTOOL(ETHTOOL_GET_TS_INFO) ioctl failed: Function not implemented</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感觉像是网络的问题，如果可能的话换个环境再做实验，比如docker镜像。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-13 12:13:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/1d/64/52a5863b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>大土豆</span>
  </div>
  <div class="_2_QraFYR_0">HTTP&#47;3会有个正式的发布会吗。。。我看现在快手和百度，HTTP3都已经在线上用了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 现在只是草案，虽然还没有正式发布，但估计和最终版差距不会太大，所以大家都提前做。<br><br>发布会肯定是不会有的，不会那么夸张。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-10 10:22:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/61/90/de8c61a0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>dongge</span>
  </div>
  <div class="_2_QraFYR_0">好敬业。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: thanks。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-10 08:54:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/56/ea/32608c44.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>giteebravo</span>
  </div>
  <div class="_2_QraFYR_0"><br>期待再次与老师相会，<br>谢谢老师！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢支持！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-10 03:41:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/9d/a4/e481ae48.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lesserror</span>
  </div>
  <div class="_2_QraFYR_0">老师，很棒👍🏻！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: nice to meet you again.</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-10 00:15:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/63/1b/83ac7733.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>忧天小鸡</span>
  </div>
  <div class="_2_QraFYR_0">又学了一点点docker，很棒的体验，认真负责</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: docker很好玩，现在容器化是大趋势，早学有好处。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-22 19:30:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/db/20/491dd5cc.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>zha.qiang</span>
  </div>
  <div class="_2_QraFYR_0">Found a typo:<br><br>```<br>cu加l https:&#47;&#47;www.chrono.com&#47;30-1 -k --tlsv1.2<br>```</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢指正，会联系编辑尽快修复。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-19 22:35:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>candy</span>
  </div>
  <div class="_2_QraFYR_0">谢谢老师，节日快乐！还建立了镜像，学习更方便了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不客气，希望能帮到你。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-10 16:53:06</div>
  </div>
</div>
</div>
</li>
</ul>