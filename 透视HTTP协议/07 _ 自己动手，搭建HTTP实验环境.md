<audio title="07 _ 自己动手，搭建HTTP实验环境" src="https://static001.geekbang.org/resource/audio/5a/80/5a81d350ff30475e95ccc4261b90b580.mp3" controls="controls"></audio> 
<p>这一讲是“破冰篇”的最后一讲，我会先简单地回顾一下之前的内容，然后在Windows系统上实际操作，用几个应用软件搭建出一个“最小化”的HTTP实验环境，方便后续的“基础篇”“进阶篇”“安全篇”的学习。</p><h2>“破冰篇”回顾</h2><p>HTTP协议诞生于30年前，设计之初的目的是用来传输纯文本数据。但由于形式灵活，搭配URI、HTML等技术能够把互联网上的资源都联系起来，构成一个复杂的超文本系统，让人们自由地获取信息，所以得到了迅猛发展。</p><p>HTTP有多个版本，目前应用的最广泛的是HTTP/1.1，它几乎可以说是整个互联网的基石。但HTTP/1.1的性能难以满足如今的高流量网站，于是又出现了HTTP/2和HTTP/3。不过这两个新版本的协议还没有完全推广开。在可预见的将来，HTTP/1.1还会继续存在下去。</p><p>HTTP翻译成中文是“超文本传输协议”，是一个应用层的协议，通常基于TCP/IP，能够在网络的任意两点之间传输文字、图片、音频、视频等数据。</p><p>HTTP协议中的两个端点称为<strong>请求方</strong>和<strong>应答方</strong>。请求方通常就是Web浏览器，也叫user agent，应答方是Web服务器，存储着网络上的大部分静态或动态的资源。</p><p>在浏览器和服务器之间还有一些“中间人”的角色，如CDN、网关、代理等，它们也同样遵守HTTP协议，可以帮助用户更快速、更安全地获取资源。</p><!-- [[[read_end]]] --><p>HTTP协议不是一个孤立的协议，需要下层很多其他协议的配合。最基本的是TCP/IP，实现寻址、路由和可靠的数据传输，还有DNS协议实现对互联网上主机的定位查找。</p><p>对HTTP更准确的称呼是“<strong>HTTP over TCP/IP</strong>”，而另一个“<strong>HTTP over SSL/TLS</strong>”就是增加了安全功能的HTTPS。</p><h2>软件介绍</h2><p>常言道“实践出真知”，又有俗语“光说不练是假把式”。要研究HTTP协议，最好有一个实际可操作、可验证的环境，通过实际的数据、现象来学习，肯定要比单纯的“动嘴皮子”效果要好的多。</p><p>现成的环境当然有，只要能用浏览器上网，就会有HTTP协议，就可以进行实验。但现实的网络环境又太复杂了，有很多无关的干扰因素，这些“噪音”会“淹没”真正有用的信息。</p><p>所以，我给你的建议是：搭建一个“<strong>最小化</strong>”的环境，在这个环境里仅有HTTP协议的两个端点：请求方和应答方，去除一切多余的环节，从而可以抓住重点，快速掌握HTTP的本质。</p><p><img src="https://static001.geekbang.org/resource/image/85/0b/85cadf90dc96cf413afaf8668689ef0b.png?wh=3000*1681" alt=""></p><p>简单说一下这个“最小化”环境用到的应用软件：</p><ul>
<li>Wireshark</li>
<li>Chrome/Firefox</li>
<li>Telnet</li>
<li>OpenResty</li>
</ul><p><strong>Wireshark</strong>是著名的网络抓包工具，能够截获在TCP/IP协议栈中传输的所有流量，并按协议类型、地址、端口等任意过滤，功能非常强大，是学习网络协议的必备工具。</p><p>它就像是网络世界里的一台“高速摄像机”，把只在一瞬间发生的网络传输过程如实地“拍摄”下来，事后再“慢速回放”，让我们能够静下心来仔细地分析那一瞬到底发生了什么。</p><p><strong>Chrome</strong>是Google开发的浏览器，是目前的主流浏览器之一。它不仅上网方便，也是一个很好的调试器，对HTTP/1.1、HTTPS、HTTP/2、QUIC等的协议都支持得非常好，用F12打开“开发者工具”还可以非常详细地观测HTTP传输全过程的各种数据。</p><p>如果你更习惯使用<strong>Firefox</strong>，那也没问题，其实它和Chrome功能上都差不太多，选择自己喜欢的就好。</p><p>与Wireshark不同，Chrome和Firefox属于“事后诸葛亮”，不能观测HTTP传输的过程，只能看到结果。</p><p><strong>Telnet</strong>是一个经典的虚拟终端，基于TCP协议远程登录主机，我们可以使用它来模拟浏览器的行为，连接服务器后手动发送HTTP请求，把浏览器的干扰也彻底排除，能够从最原始的层面去研究HTTP协议。</p><p><strong>OpenResty</strong>你可能比较陌生，它是基于Nginx的一个“强化包”，里面除了Nginx还有一大堆有用的功能模块，不仅支持HTTP/HTTPS，还特别集成了脚本语言Lua简化Nginx二次开发，方便快速地搭建动态网关，更能够当成应用容器来编写业务逻辑。</p><p>选择OpenResty而不直接用Nginx的原因是它相当于Nginx的“超集”，功能更丰富，安装部署更方便。我也会用Lua编写一些服务端脚本，实现简单的Web服务器响应逻辑，方便实验。</p><h2>安装过程</h2><p>这个“最小化”环境的安装过程也比较简单，大约只需要你半个小时不到的时间就能搭建完成。</p><p>我在GitHub上为本专栏开了一个项目：<a href="https://github.com/chronolaw/http_study.git">http_study</a>，可以直接用“git clone”下载，或者去Release页面，下载打好的<a href="https://github.com/chronolaw/http_study/releases">压缩包</a>。</p><p>我使用的操作环境是Windows 10，如果你用的是Mac或者Linux，可以用VirtualBox等虚拟机软件安装一个Windows虚拟机，再在里面操作（或者可以到“答疑篇”的<a href="https://time.geekbang.org/column/article/146833">Linux/Mac实验环境搭建</a>中查看搭建方法）。</p><p>首先你要获取<strong>最新</strong>的http_study项目源码，假设clone或解压的目录是“D:\http_study”，操作完成后大概是下图这个样子。</p><p><img src="https://static001.geekbang.org/resource/image/86/ee/862511b8ef87f78218631d832927bcee.png?wh=2000*1121" alt=""></p><p>Chrome和WireShark的安装比较简单，一路按“下一步”就可以了。版本方面使用最新的就好，我的版本可能不是最新的，Chrome是73，WireShark是3.0.0。</p><p>Windows 10自带Telnet，不需要安装，但默认是不启用的，需要你稍微设置一下。</p><p>打开Windows的设置窗口，搜索“Telnet”，就会找到“启用或关闭Windows功能”，在这个窗口里找到“Telnet客户端”，打上对钩就可以了，可以参考截图。</p><p><img src="https://static001.geekbang.org/resource/image/1a/47/1af035861c4fd33cb42005eaa1f5f247.png?wh=2000*1121" alt=""></p><p>接下来我们要安装OpenResty，去它的<a href="http://openresty.org">官网</a>，点击左边栏的“Download”，进入下载页面，下载适合你系统的版本（这里我下载的是64位的1.15.8.1，包的名字是“openresty-1.15.8.1-win64.zip”）。</p><p><img src="https://static001.geekbang.org/resource/image/ee/0a/ee7016fecd79919de550677af32f740a.png?wh=2000*760" alt=""></p><p>然后要注意，你必须把OpenResty的压缩包解压到刚才的“D:\http_study”目录里，并改名为“openresty”。</p><p><img src="https://static001.geekbang.org/resource/image/5a/b5/5acb89c96041f91bbc747b7e909fd4b5.png?wh=2000*1121" alt=""></p><p>安装工作马上就要完成了，为了能够让浏览器能够使用DNS域名访问我们的实验环境，还要改一下本机的hosts文件，位置在“C:\WINDOWS\system32\drivers\etc”，在里面添加三行本机IP地址到测试域名的映射，你也可以参考GitHub项目里的hosts文件，这就相当于在一台物理实机上“托管”了三个虚拟主机。</p><pre><code>127.0.0.1       www.chrono.com
127.0.0.1       www.metroid.net
127.0.0.1       origin.io
</code></pre><p>注意修改hosts文件需要管理员权限，直接用记事本编辑是不行的，可以切换管理员身份，或者改用其他高级编辑器，比如Notepad++，而且改之前最好做个备份。</p><p>到这里，我们的安装工作就完成了！之后你就可以用Wireshark、Chrome、Telnet在这个环境里随意“折腾”，弄坏了也不要紧，只要把目录删除，再来一遍操作就能复原。</p><h2>测试验证</h2><p>实验环境搭建完了，但还需要把它运行起来，做一个简单的测试验证，看是否运转正常。</p><p>首先我们要启动Web服务器，也就是OpenResty。</p><p>在http_study的“www”目录下有四个批处理文件，分别是：</p><p><img src="https://static001.geekbang.org/resource/image/e5/da/e5d35bb94c46bfaaf8ce5c143b2bb2da.png?wh=2000*1121" alt=""></p><ul>
<li>start：启动OpenResty服务器；</li>
<li>stop：停止OpenResty服务器；</li>
<li>reload：重启OpenResty服务器；</li>
<li>list：列出已经启动的OpenResty服务器进程。</li>
</ul><p>使用鼠标双击“start”批处理文件，就会启动OpenResty服务器在后台运行，这个过程可能会有Windows防火墙的警告，选择“允许”即可。</p><p>运行后，鼠标双击“list”可以查看OpenResty是否已经正常启动，应该会有两个nginx.exe的后台进程，大概是下图的样子。</p><p><img src="https://static001.geekbang.org/resource/image/db/1d/dba34b8a38e98bef92289315db29ee1d.png?wh=2000*661" alt=""></p><p>有了Web服务器后，接下来我们要运行Wireshark，开始抓包。</p><p>因为我们的实验环境运行在本机的127.0.0.1上，也就是loopback“环回”地址。所以，在Wireshark里要选择“Npcap loopback Adapter”，过滤器选择“HTTP TCP port(80)”，即只抓取HTTP相关的数据包。鼠标双击开始界面里的“Npcap loopback Adapter”即可开始抓取本机上的网络数据。</p><p><img src="https://static001.geekbang.org/resource/image/12/c4/128d8a5ed9cdd666dbfa4e17fd39afc4.png?wh=2000*728" alt=""></p><p>然后我们打开Chrome，在地址栏输入“<code>http://localhost</code>”，访问刚才启动的OpenResty服务器，就会看到一个简单的欢迎界面，如下图所示。</p><p><img src="https://static001.geekbang.org/resource/image/d7/88/d7f12d4d480d7100cd9804d2b16b8a88.png?wh=2000*672" alt=""></p><p>这时再回头去看Wireshark，应该会显示已经抓到了一些数据，就可以用鼠标点击工具栏里的“停止捕获”按钮告诉Wireshark“到此为止”，不再继续抓包。</p><p><img src="https://static001.geekbang.org/resource/image/f7/79/f7d05a3939d81742f18d2da7a1883179.png?wh=2000*587" alt=""></p><p>至于这些数据是什么，表示什么含义，我会在下一讲再详细介绍。</p><p>如果你能够在自己的电脑上走到这一步，就说明“最小化”的实验环境已经搭建成功了，不要忘了实验结束后运行批处理“stop”停止OpenResty服务器。</p><h2>小结</h2><p>这次我们学习了如何在自己的电脑上搭建HTTP实验环境，在这里简单小结一下今天的内容。</p><ol>
<li><span class="orange">现实的网络环境太复杂，有很多干扰因素，搭建“最小化”的环境可以快速抓住重点，掌握HTTP的本质；</span></li>
<li><span class="orange">我们选择Wireshark作为抓包工具，捕获在TCP/IP协议栈中传输的所有流量；</span></li>
<li><span class="orange">我们选择Chrome或Firefox浏览器作为HTTP协议中的user agent；</span></li>
<li><span class="orange">我们选择OpenResty作为Web服务器，它是一个Nginx的“强化包”，功能非常丰富；</span></li>
<li><span class="orange">Telnet是一个命令行工具，可用来登录主机模拟浏览器操作；</span></li>
<li><span class="orange">在GitHub上可以下载到本专栏的专用项目源码，只要把OpenResty解压到里面即可完成实验环境的搭建。</span></li>
</ol><h2>课下作业</h2><p>1.按照今天所学的，在你自己的电脑上搭建出这个HTTP实验环境并测试验证。</p><p>2.由于篇幅所限，我无法详细介绍Wireshark，你有时间可以再上网搜索Wireshark相关的资料，了解更多的用法。</p><p>欢迎你把自己的学习体会写在留言区，与我和其他同学一起讨论。如果你觉得有所收获，也欢迎把文章分享给你的朋友。</p><p><img src="https://static001.geekbang.org/resource/image/03/dd/03727c2a64cbc628ec18cf39a6a526dd.png?wh=1769*3004" alt="unpreview"></p><p><img src="https://static001.geekbang.org/resource/image/56/63/56d766fc04654a31536f554b8bde7b63.jpg?wh=1110*659" alt="unpreview"></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/56/1a/abb7bfe3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>cylim</span>
  </div>
  <div class="_2_QraFYR_0">在Mac上，<br><br>拷贝项目（需要Git）<br>1. git clone https:&#47;&#47;github.com&#47;chronolaw&#47;http_study<br><br>安装OpenResty （推荐使用Homebrew）<br>1. brew tap openresty&#47;brew<br>2. brew install openresty <br><br>运行项目<br>1. cd http_study&#47;www&#47;<br>2. openresty -p `pwd` -c conf&#47;nginx.conf<br><br>停止项目<br>1. openresty -s quit -p `pwd` -c conf&#47;nginx.conf</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 好同学！！赞！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-12 09:02:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/0d/40/f70e5653.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>前端西瓜哥</span>
  </div>
  <div class="_2_QraFYR_0">对 cylim 的 mac 上运行 openresty 的教程进行补充：<br>按照 cylim 的做法，我遇到了访问 localhost 时，网页报 403 错误的情况，原因是没有 html&#47;index.html 文件的访问权限。我研究并找到了解决方案：<br>先 ls -la html，查看文件的权限，得到 user 和 group，我这里是 fstar 和 staff。<br><br>然后在 conf&#47;nginx.conf 文件的顶部添加<br><br>user fstar staff;<br><br>然后再启动 openresty 就可以正常访问了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 感谢同学的热心补充。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-28 01:19:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/78/ac/e5e6e7f3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>古夜</span>
  </div>
  <div class="_2_QraFYR_0">我打赌很多人抓不到包，找不到本地回环地址，不知道最新版的wireshark是否修复了这个问题，如果出现以上问题，记得卸载重装wireshark，不要勾选它自带的ncap应该是这个名字，然后自己去单独下一个这个软件</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 有问题欢迎提出来，我机器上的Wireshark装的比较早，具体的步骤记不太清了，应该是很简单的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-12 09:07:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/4e/31/3a7e74c1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>名曰蓝兮</span>
  </div>
  <div class="_2_QraFYR_0">centos上的安装步骤，有错误请指出<br>wireshark：<br>1. yum install wireshark<br>    yum install wireshark-gnome<br>2. 如果不是root用户，启动后没有权限，做如下操作<br>    2.1 添加当前用户到wireshark组，我的用户叫&#39;zp&#39;：<br>          usermod -a -G wireshark zp<br>    2.2 然后给dumpcap读网卡的权限：<br>          setcap cap_net_raw,cap_net_admin+eip &#47;usr&#47;sbin&#47;dumpcap<br>完成后重启机器。<br><br>telnet：<br>yum install telnet<br><br>OpenResty:<br>官网有说明，按照说明一步步来<br>1. 添加OpenResty仓库：<br>    sudo yum install yum-utils<br>    sudo yum-config-manager --add-repo https:&#47;&#47;openresty.org&#47;package&#47;centos&#47;openresty.repo<br>2. 安装OpenResty:<br>    sudo yum install openresty<br>    sudo yum install openresty-resty<br>3. 在～目录下创建conf和logs文件夹：<br>    mkdir ~&#47;work<br>    cd ~&#47;work<br>    mkdir logs&#47; conf&#47;<br>4. 在conf文件夹下创建nginx.conf文件，内容如下：<br>worker_processes  1;<br>error_log logs&#47;error.log;<br>events {<br>    worker_connections 1024;<br>}<br>http {<br>    server {<br>        listen 8080;<br>        location &#47; {<br>            default_type text&#47;html;<br>            content_by_lua_block {<br>                ngx.say(&quot;&lt;p&gt;hello, world&lt;&#47;p&gt;&quot;)<br>            }<br>        }<br>    }<br>}<br>5. 添加OpenResty环境变量，注意冒号，别丢了:<br>    PATH=&#47;usr&#47;local&#47;openresty&#47;nginx&#47;sbin:$PATH<br>    export PATH<br>6. 在&#39;~&#47;work&#39;目录下启动OpenResty:<br>    nginx -p `pwd`&#47; -c conf&#47;nginx.conf<br>7. 验证安装：<br>    curl http:&#47;&#47;localhost:8080<br>    输出：<br>    &lt;p&gt;hello, world&lt;&#47;p&gt;</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 写的很详细，赞！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-19 11:01:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/25/8e/30/71b84ce7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>geek桃</span>
  </div>
  <div class="_2_QraFYR_0">送给后来的同学：<br>如果你按照步骤操作之后出现：start启动完成后，cmd窗口一闪而过，点击list启动时显示“没有运行的任务匹配制定标准”，请按任意键继续，当随便输入数据时，cmd窗口又没了；去查找www&#47;logs&#47;error.log，如果日志报错为“10013: An attempt was made to access a socket in a way forbidden by its access permissions”，说明你的80端口被占用了，按照下面步骤操作。<br>1.按键盘win+r 打开运行界面，输入cmd，确定，打开管理员界面<br>2.输入 netstat -aon | findstr :80 （有一条0.0.0.0的数据，记住这条数据最后的数字；我的是5884）<br>3.输入  tasklist|findstr &quot;5884&quot; （根据上一步查到的数字，找到5884端口对应的服务名称，我的是snv）<br>4.在控制台关闭服务<br>5.重新启动start.bat，成功！<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 非常好的经验分享，鼓励。<br><br>Windows环境比较复杂，容易出各种错误，如果不好解决可以尝试用虚拟机或者docker。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-03 11:05:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/42/81/788cd471.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>珈蓝白塔</span>
  </div>
  <div class="_2_QraFYR_0">Mac 开发环境的搭建参考《答疑篇41》，项目中已经有前辈写好的 shell 脚本，终端里直接运行就可以，不需要自己输入 openresty 命令啦；服务器启动以后访问 localhost 环境遇到了 403 问题，显示不出来 HTML，可参照留言区中提出的，在 conf&#47;nginx.conf 文件的顶部添加 user xxxx staff; 来解决，这个 xxxx 是自己的 mac 账户名；Wireshark(v3.2.3) 中选择环回地址时，选择 lockback：lo0 就可以啦，过滤器是和文中一样的，已成功搭建环境（2020年4月13日）</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 欢迎经验分享，让同学都少走弯路。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-13 14:46:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/39/b8/e4b6a677.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>YUANWOW</span>
  </div>
  <div class="_2_QraFYR_0">我一开始nginx一直起不来<br>后面看了error.log<br>发现本机443端口被占用了<br>netstat -ano | findstr &quot;443&quot;<br>看到一个 0.0.0.0:443  最后一列是进程的PID<br>查找到是vmware-hostd这个进程<br>后面谷歌搜索了下 <br>vmware的虚拟机共享会默认占用443端口<br>所以安装了vmware的把虚拟机共享关闭就好了  </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 欢迎经验分享。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-02 22:47:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/ibZVAmmdAibBeVpUjzwId8ibgRzNk7fkuR5pgVicB5mFSjjmt2eNadlykVLKCyGA0GxGffbhqLsHnhDRgyzxcKUhjg/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>pyhhou</span>
  </div>
  <div class="_2_QraFYR_0">想请问下在 MacOs 或者是 Linux 上怎么搭建？（不是太想弄 Windows 虚拟机）</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 需要用brew或者yum安装OpenResty，然后看一下nginx.conf，里面的注释有说明。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-12 05:46:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/3f/23/8ff389d2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>郁方林</span>
  </div>
  <div class="_2_QraFYR_0">start启动完成后，cmd窗口一闪而过，当我点击list启动时显示“没有运行的任务匹配制定标准”，请按任意键继续，当我随便输入数据时，cmd窗口又没了</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 看一下www&#47;logs&#47;error.log，是否有端口被占用了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-12 10:01:59</div>
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
  <div class="_2_QraFYR_0">破冰篇最后一篇，是马上开展破冰行动，抓捕林耀东了吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 写这个的时候电视剧还没出呢，完全的碰巧，笑。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-12 09:59:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTLw9fVyja3eQLGQenLf5EqVaxGQoibo7rq8A7IRjlXED9FhicKukcn0ibCCtiaBqpEib4ZEIWfFOkiaGMSQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_d4dee7</span>
  </div>
  <div class="_2_QraFYR_0">老师 最近我维护的一个网站打开速度非常慢  服务器CPU 负载0.5到0.8之间 有十多台web 服务器  redis  db  负载都正常 只是nginx 的链接数在出问题的时间点有上升  我目前不知道从哪下手排查这个问题 是用php symfony 开发的  能否给点思路 万分感谢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 在日志里加上$upstream_connect_time、$upstream_header_time、$upstream_response_time这几个变量，看看反向代理耗时在哪里。<br><br>另外也可以用systemtap，抓火焰图看看。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-12 08:28:08</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/f5/0b/73628618.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>兔嘟嘟</span>
  </div>
  <div class="_2_QraFYR_0">windows上装的最新的3.4.7版，没有npcap，但是有一个Adapter for loopback traffic，用起来效果一样，就是抓到的Source是::1，猜测是这台电脑比较新，用上了IPv6的localhost</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这篇文章已经是两年前了，随着软件的升级，可能有的选项已经过时了，欢迎同学们随时更新。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-16 19:08:01</div>
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
  <div class="_2_QraFYR_0">老师，上面过程怎么没有用到telnet</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 后面会用，Telnet需要手动输入http请求，比较麻烦，只有在比较特殊的时候才会用。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-12 10:50:37</div>
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
  <div class="_2_QraFYR_0">老师可以把环境打包成容器，我们进容器直接嗨，隔离更彻底</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 考虑大多数同学都用的是Windows，所以暂时只能这样，手动操作也能加深一下印象吧。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-12 10:03:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/7b/f0/269139d5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Cris</span>
  </div>
  <div class="_2_QraFYR_0">在浏览器和服务器之间还存在“中间人”，这些中间人也都遵循http协议，我想问下，这些中间人是不是都工作在应用层？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，都是用http协议，当然就是在应用层。<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-11 07:38:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/7b/57/a9b04544.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>QQ怪</span>
  </div>
  <div class="_2_QraFYR_0">为啥有时候批处理stop不掉openresty？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可能是多次start，stop就失效了，只能手动在任务管理器里关闭。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-12 23:20:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c7/47/30d4b61e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>不是云不飘</span>
  </div>
  <div class="_2_QraFYR_0">建议还是能有win和Mac，逼近做开发的Mac不再少数。这些东西之前只有客户对接问题才会看到运维大哥在哪捣腾那时候看的一脸们逼，难得如此细致的了解。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 有同学已经写的很详细了，看看后续是否再专门详细写一下Linux和mac的搭建吧。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-17 22:28:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/27/35/ba972e11.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>因缺思厅</span>
  </div>
  <div class="_2_QraFYR_0">这次环境搭建很顺利呀<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: nice</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-12 20:35:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/9e/e7/2050698e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>6欢</span>
  </div>
  <div class="_2_QraFYR_0">建议环境搭建都在linux操作，哈哈</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我也是这么想，可惜用Windows的同学还是不少。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-12 17:58:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/7e/53/c29c2fc9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>sdjdd</span>
  </div>
  <div class="_2_QraFYR_0">正在学温铭老师的《OpenResty从入门到实战》，正好用上。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 搭配起来学习，效果更好。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-06-12 13:43:49</div>
  </div>
</div>
</div>
</li>
</ul>