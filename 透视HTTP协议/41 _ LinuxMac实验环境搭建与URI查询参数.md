<audio title="41 _ LinuxMac实验环境搭建与URI查询参数" src="https://static001.geekbang.org/resource/audio/53/09/53387d0bb500b74eea2e2b8ca622d009.mp3" controls="controls"></audio> 
<p>你好，我是Chrono。</p><p>先要说一声“抱歉”。由于工作比较紧张、项目实施频繁出差，导致原本预定的“答疑篇”迟迟没有进展，这次趁着“十一”长假，总算赶出了两期，集中回答几个同学们问得比较多的问题：Linux/Mac实验环境搭建（<a href="https://time.geekbang.org/column/article/100124">第7讲</a>），URI查询参数（<a href="https://time.geekbang.org/column/article/102008">第11讲</a>），还有DHE/ECDHE算法的原理（<a href="https://time.geekbang.org/column/article/110354">第26讲</a>），后续有时间可能还会再陆续补充完善。</p><p>很高兴在时隔一个多月后与你再次见面，废话不多说了，让我们开始吧。</p><h2>Linux上搭建实验环境</h2><p>我们先来看一下如何在Linux上搭建课程的实验环境。</p><p>首先，需要安装OpenResty，但它在Linux上提供的不是zip压缩包，而是各种Linux发行版的预编译包，支持常见的Ubuntu、Debian、CentOS等等，而且<a href="http://openresty.org/cn/linux-packages.html">官网</a>上有非常详细安装步骤。</p><p>以Ubuntu为例，只要“按部就班”地执行下面的几条命令就可以了，非常轻松：</p><pre><code># 安装导入GPG公钥所需的依赖包：
sudo apt-get -y install --no-install-recommends wget gnupg ca-certificates


# 导入GPG密钥：
wget -O - https://openresty.org/package/pubkey.gpg | sudo apt-key add -


# 安装add-apt-repository命令
sudo apt-get -y install --no-install-recommends software-properties-common


# 添加官方仓库：
sudo add-apt-repository -y &quot;deb http://openresty.org/package/ubuntu $(lsb_release -sc) main&quot;


# 更新APT索引：
sudo apt-get update


# 安装 OpenResty
sudo apt-get -y install openresty
</code></pre><p>全部完成后，OpenResty会安装到“/usr/local/openresty”目录里，可以用它自带的命令行工具“resty”来验证是否安装成功：</p><pre><code>$resty -v
resty 0.23
nginx version: openresty/1.15.8.2
built with OpenSSL 1.1.0k  28 May 2019
</code></pre><p>有了OpenResty，就可以从GitHub上获取http_study项目的源码了，用“git clone”是最简单快捷的方法：</p><!-- [[[read_end]]] --><pre><code>git clone https://github.com/chronolaw/http_study
</code></pre><p>在Git仓库的“www”目录，我为Linux环境补充了一个Shell脚本“run.sh”，作用和Windows下的start.bat、stop.bat差不多，可以简单地启停实验环境，后面可以接命令行参数start/stop/reload/list：</p><pre><code>cd http_study/www/    #脚本必须在www目录下运行，才能找到nginx.conf
./run.sh start        #启动实验环境
./run.sh list         #列出实验环境的Nginx进程
./run.sh reload       #重启实验环境
./run.sh stop         #停止实验环境
</code></pre><p>启动OpenResty之后，就可以用浏览器或者curl来验证课程里的各个测试URI，但之前不要忘记修改“/etc/hosts”添加域名解析，例如：</p><pre><code>curl -v &quot;http://127.0.0.1/&quot;
curl -v &quot;http://www.chrono.com/09-1&quot;
curl -k &quot;https://www.chrono.com/24-1?key=1234&quot;
curl -v &quot;http://www.chrono.com/41-1&quot;
</code></pre><h2>Mac上搭建实验环境</h2><p>看完了Linux，我们再来看一下Mac。</p><p>这里我用的是两个环境：Mac mini 和 MacBook Air，不过都是好几年前的“老古董”了，系统是10.13 High Sierra和10.14 Mojave（更早的版本没有测试，但应该也都可以）。</p><p>首先要保证Mac里有第三方包管理工具homebrew，可以用下面的命令安装：</p><pre><code>#先安装Mac的homebrew
/usr/bin/ruby -e &quot;$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)&quot;
</code></pre><p>然后，要用homebrew安装OpenResty，但它在Mac上的安装过程和Linux不同，不是预编译包，而是要下载许多相关的源码（如OpenSSL），然后再用clang本地编译，大概要花上五六分钟的时间，整体上比较慢，要有点耐心。</p><pre><code>#使用homebrew安装OpenResty
brew install openresty/brew/openresty
</code></pre><p>安装完OpenResty，后续的操作就和Linux一样了，“git clone”项目源码：</p><pre><code>git clone https://github.com/chronolaw/http_study
</code></pre><p>然后，进“http_study/www”目录，用脚本“run.sh”启停实验环境，用Safari或者curl测试。</p><h2>Linux/Mac下的抓包</h2><p>Linux和Mac里都有图形界面版本的Wireshark，抓包的用法与Windows完全一样，简单易用。</p><p>所以，今天我主要介绍命令行形式的抓包。</p><p>命令行抓包最基本的方式就是著名的tcpdump，不过我用得不是很多，所以就尽可能地“藏拙”了。</p><p>简单的抓包使用“-i lo”指定抓取本地环回地址，“port”指定端口号，“-w”指定抓包的存放位置，抓包结束时用“Ctrl+C”中断：</p><pre><code>sudo tcpdump -i lo -w a.pcap
sudo tcpdump -i lo port 443 -w a.pcap
</code></pre><p>抓出的包也可以用tcpdump直接查看，用“-r”指定包的名字：</p><pre><code>tcpdump -r a.pcap 
tcpdump -r 08-1.pcapng -A
</code></pre><p>不过在命令行界面下可以用一个更好的工具——tshark，它是Wireshark的命令行版本，用法和tcpdump差不多，但更易读，功能也更丰富一些。</p><pre><code>tshark -r 08-1.pcapng 
tshark -r 08-1.pcapng -V
tshark -r 08-1.pcapng -O tcp|less
tshark -r 08-1.pcapng -O http|less
</code></pre><p>tshark也支持使用keylogfile解密查看HTTPS的抓包，需要用“-o”参数指定log文件，例如：</p><pre><code>tshark -r 26-1.pcapng -O http -o ssl.keylog_file:26-1.log|less
</code></pre><p>tcpdump、tshark和Linux里的许多工具一样，参数繁多、功能强大，你可以课后再找些资料仔细研究，这里就不做过多地介绍了。</p><h2>URI的查询参数和头字段</h2><p>在<a href="https://time.geekbang.org/column/article/102008">第11讲</a>里我留了一个课下作业：</p><p>“URI的查询参数和头字段很相似，都是key-value形式，都可以任意自定义，那么它们在使用时该如何区别呢？”</p><p>从课程后的留言反馈来看，有的同学没理解这个问题的本意，误以为问题问的是这两者在表现上应该如何区分，比如查询参数是跟在“？”后面，头字段是请求头里的KV对。</p><p>这主要是怪我没有说清楚。这个问题实际上想问的是：查询参数和头字段两者的形式很相近，query是key-value，头字段也是key-value，它们有什么区别，在发送请求时应该如何正确地使用它们。</p><p>换个说法就是：<span class="orange">应该在什么场景下恰当地自定义查询参数或者头字段来附加额外信息</span>。</p><p>当然了，因为HTTP协议非常灵活，这个问题也不会有唯一的、标准的答案，我只能说说我自己的理解。</p><p>因为查询参数是与URI关联在一起的，所以它针对的就是资源（URI），是长期、稳定的。而头字段是与一次HTTP请求关联的，针对的是本次请求报文，所以是短期、临时的。简单来说，就是两者的作用域和时效性是不同的。</p><p>从这一点出发，我们就可以知道在哪些场合下使用查询参数和头字段更加合适。</p><p>比如，要获取一个JS文件，而它会有多个版本，这个“版本”就是资源的一种属性，应该用查询参数来描述。而如果要压缩传输、或者控制缓存的时间，这些操作并不是资源本身固有的特性，所以用头字段来描述更好。</p><p>除了查询参数和头字段，还可以用其他的方式来向URI发送附加信息，最常用的一种方式就是POST一个JSON结构，里面能够存放比key-value复杂得多的数据，也许你早就在实际工作中这么做了。</p><p>在这种情况下，就可以完全不使用查询参数和头字段，服务器从JSON里获取所有必需的数据，让URI和请求头保持干净、整洁（^_^）。</p><p>今天的答疑就先到这里，我们下期再见，到时候再讲ECDHE算法。</p><p><img src="https://static001.geekbang.org/resource/image/c1/f9/c17f3027ba3cfb45e391107a8cf04cf9.png?wh=1769*2606" alt="unpreview"></p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/25/3f/96/ce0b9678.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>浪里淘沙的小法师</span>
  </div>
  <div class="_2_QraFYR_0">讲一下用M1芯片 mac 搭建搭建环境的遇到的问题和解决方法。<br><br>1. 运行 .&#47;run.sh start 报错 &#47;usr&#47;local&#47;bin&#47;openresty: command not found<br>这是因为 M1 芯片mac 的 homebrew 安装软件的位置与以往不同，先通过 which openresty 查询 openresty 的位置 &#47;opt&#47;homebrew&#47;bin&#47;openresty，然后打开 run.sh 脚本替换一下老师写的位置<br>if [ $os != &quot;Linux&quot; ] ; then<br>    openresty=&quot;&#47;usr&#47;local&#47;bin&#47;openresty&quot;<br>fi<br>替换成<br>if [ $os != &quot;Linux&quot; ] ; then<br>    openresty=&quot;&#47;opt&#47;homebrew&#47;bin&#47;openresty&quot;<br>fi<br><br>2. 再运行 .&#47;run.sh start 报错 nginx: [emerg] could not build server_names_hash, you should increase server_names_hash_bucket_size: 32<br>网上查寻了一下，放大 bucket_size 即可，打开 www&#47;conf&#47;nginx.conf 文件添加这一句server_names_hash_bucket_size 64; 即可<br># http conf<br>http {<br>    #include     http&#47;common.conf;<br>    #include     http&#47;cache.conf;<br>    #include     http&#47;resty.conf;<br>    #include     http&#47;mime.types;<br>    server_names_hash_bucket_size 64;<br>    <br>    include     http&#47;*.conf;<br><br>    include     http&#47;servers&#47;*.conf;<br><br>}</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 太高端了，都用上M1的Mac。<br><br>也可以参考GitHub里的Dockerfile，构建出基于arm的镜像。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-10 15:06:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/8a/e7/a6c603cf.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>GitHubGanKai</span>
  </div>
  <div class="_2_QraFYR_0">真好，又见到你了，而且我最近换个了mac，😊正愁这个。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: we meet again.</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-09 00:24:59</div>
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
  <div class="_2_QraFYR_0">老师好，<br>按文章指导搭建了MAC的环境：<br>openresty -v<br>nginx version: openresty&#47;1.11.2.2<br><br>在~&#47;git&#47;http_study&#47;www目录下执行<br> .&#47;run.sh start<br>Password:<br>nginx: [emerg] &quot;&#47;Users&#47;xiaodong&#47;git&#47;http_study&#47;www&#47;conf&#47;ssl&#47;ticket.key&quot; must be 48 bytes in &#47;Users&#47;xiaodong&#47;git&#47;http_study&#47;www&#47;conf&#47;nginx.conf:34<br>报了这个错误，在网上google没找到解决方法。<br>尝试在nginx.conf中注销相关代码，也会报其他错误。<br>老师能指点一下吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这是提示Nginx要求48字节的密钥文件，按理说附带的80字节也是可以的，你可以用命令“openssl rand 48 &gt; ticket.key”重新生成www&#47;conf&#47;ssl&#47;ticket.key。<br><br>详细可参见http:&#47;&#47;nginx.org&#47;en&#47;docs&#47;http&#47;ngx_http_ssl_module.html#ssl_session_tickets<br><br>另外，你用的openresty版本太老了，用最新的1.15.8.2可能就不会出现这样的问题。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-18 11:03:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/4a/7b/23da5db9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Luka!3055</span>
  </div>
  <div class="_2_QraFYR_0">记录下问题：<br><br>brew install openresty&#47;brew&#47;openresty 后，报错：<br>curl: (7) Failed to connect to raw.githubusercontent.com port 443: Connection refused<br>Error: An exception occurred within a child process:<br>DownloadError: Failed to download resource &quot;openresty-openssl--patch&quot;<br>Download failed: https:&#47;&#47;raw.githubusercontent.com&#47;openresty&#47;openresty&#47;master&#47;patches&#47;openssl-1.1.0d-sess_set_get_cb_yield.patch<br><br>此时把 DNS 设置为 114.114.114.114 或者 8.8.8.8 就好了，最好再挂个梯子</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: nice</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-07 22:23:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/b8/2c/0f7baf3a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Change</span>
  </div>
  <div class="_2_QraFYR_0">老师请教个问题：Mac 环境下安装以后，按照命令.&#47;run.sh start 启动后访问 localhost 显示403 Forbidden：终端返回的错误信息是下面的错误信息，这是所有端口都被占用了？我查了一下好像也没有被占用啊，不知道这是啥原因<br>nginx: [emerg] bind() to 0.0.0.0:80 failed (48: Address already in use)<br>nginx: [emerg] bind() to 0.0.0.0:8080 failed (48: Address already in use)<br>nginx: [emerg] bind() to 0.0.0.0:443 failed (48: Address already in use)<br>nginx: [emerg] bind() to 0.0.0.0:8443 failed (48: Address already in use)<br>nginx: [emerg] bind() to 0.0.0.0:440 failed (48: Address already in use)<br>nginx: [emerg] bind() to 0.0.0.0:441 failed (48: Address already in use)<br>nginx: [emerg] bind() to 0.0.0.0:442 failed (48: Address already in use)<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可能需要sudo，是否权限的问题。<br><br>可以在网上搜一下错误信息，Nginx的问题一般都有现成的解决办法。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-28 10:35:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/96/29/bcf885b9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>SmNiuhe</span>
  </div>
  <div class="_2_QraFYR_0">这个大家有遇到嘛，是不是资源的问题<br>brew install openresty&#47;brew&#47;openresty ：DownloadError: Failed to download resource &quot;openresty-openssl--patch&quot;<br>Download failed: https:&#47;&#47;raw.githubusercontent.com&#47;openresty&#47;openresty&#47;master&#47;patches&#47;openssl-1.1.0d-sess_set_get_cb_yield.patch</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果mac安装有问题，可以用virtualbox装个Linux虚拟机，暂解燃眉之急。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-07 11:40:04</div>
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
  <div class="_2_QraFYR_0">谢谢分享</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: my pleasure.</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-09 11:34:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/78/c9/6ed5ad55.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>超轶主</span>
  </div>
  <div class="_2_QraFYR_0">mac环境运行 run run.sh 返回 nginx version: openresty&#47;1.19.9.1<br>format : run.sh [start|stop|reload|list]是什么情况呢</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 后面要加参数，start|stop|reload|list。<br><br>脚本比较简单，可以用vi看看。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-13 01:39:22</div>
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
  <div class="_2_QraFYR_0">请问安装好环境后在www目录执行.&#47;run.sh start 老是command not found怎么解决</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是不是没有安装好openresty，看看是哪个命令没找到，再按照课程正文是否遗漏了哪个步骤。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-23 01:07:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/25/fa/51/5da91010.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Miroticwillbeforever</span>
  </div>
  <div class="_2_QraFYR_0">老师我有个问题。实验环境搭建好了。前两讲的实验也做成功了。<br>但是当我用浏览器 访问 www.chrono.com 时，它跳转到的 地址为 https:&#47;&#47;dp.diandongzhi.com&#47;?acct=660&amp;site=chrono.com 然后wireshark抓包并没有任何反应。我想问一下是我操作不当的原因还是怎么回事。课程大部分听完了。但是后面实验没做成挺难受的，没有去验证。等老师给个答复准备二刷！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 域名解析这个问题确实困扰了不少同学，因为我们实验环境的域名是假的，只能在本地用，所以有可能会与某些真实的域名冲突，导致输入域名跑到了外网而不是实际环境。<br><br>解决方法是改hosts文件，但因为域名会有缓存，所以有时候改了hosts也不会生效。如果遇到这种情况，最好是换个实验环境，改成虚拟机或者docker。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-22 23:15:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/55/aa/c79c292c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Erebus</span>
  </div>
  <div class="_2_QraFYR_0">老师你好、我安装好了openrestry后、启动服务说 nginx：invalid option：http，请问是怎么回事呀</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复:  这个提示信息太模糊了，不好帮你。<br><br>可以网上搜一下错误信息，一般都能找到解决方案。或者把OpenResty、GitHub项目重新安装看看。<br><br>还是不行就改用docker试试。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-20 09:27:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/e8/43/f9c0faed.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小童</span>
  </div>
  <div class="_2_QraFYR_0">不行啊，老师，我的那个openresty界面出来了，就是抓不到包！用的wireshark .搞了好久。那个telnet也安装了。是不是那步出错了 ，我就直接运行openresty，然后用抓包工具过滤信息，然后浏览器输入localhost，浏览器洁界面也出来了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以先试试Wireshark能不能抓其他网络的包，然后再试试抓本地（loopback），再加上过滤器，逐步分析来缩小范围。<br><br>如果还不行，可以试着换个环境，用虚拟机+Linux或者docker，这样的环境比较单纯隔离。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-02 17:30:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/da/17/69cca649.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>旗木卡卡</span>
  </div>
  <div class="_2_QraFYR_0">Mac电脑，耗费本人2个晚上的环境，终于搭好了，碰到了2个坑，第一个是dns查找不到，brew install openresty时，需要在本机的hosts文件，加上解析不到的url的ip地址，第二个是启动一直bind不上，nginx就自动启动了，但是很明显不是openresty，然后用root权限启动成功，也可以正常访问，发现是nginx.conf的user权限问题，修改成本机的用户user kaka(你的用户名) staff;即可。 </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 虽然都是unix，但mac的环境和Linux还是不太一样，辛苦了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-19 22:54:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/19/18/b3/848ffa10.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Jinlee</span>
  </div>
  <div class="_2_QraFYR_0">Welcome to HTTP Study Page! 还好我看得迟，成功在ubuntu下搭建起环境😊😊😊</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: nice</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-21 20:56:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/3c/8a/900ca88a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>test</span>
  </div>
  <div class="_2_QraFYR_0">最喜欢实验环境了，之前学习就是苦于没实验环境浪费了几年时间。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 搭好后一定要多做实验，实践出真知。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-01-06 01:23:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/f0/61/68462a07.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>无名</span>
  </div>
  <div class="_2_QraFYR_0">Updating Homebrew...<br>==&gt; Auto-updated Homebrew!<br>Updated 1 tap (homebrew&#47;core).<br>==&gt; Updated Formulae<br>handbrake<br><br>==&gt; Installing openresty from openresty&#47;brew<br>==&gt; Downloading https:&#47;&#47;openresty.org&#47;download&#47;openresty-1.15.8.2.tar.gz<br>Already downloaded: &#47;Users&#47;hejunbin&#47;Library&#47;Caches&#47;Homebrew&#47;downloads&#47;4395089f0fd423261d4f1124b7beb0f69e1121e59d399e89eaa6e25b641333bc--openresty-1.15.8.2.tar.gz<br>==&gt; .&#47;configure -j8 --prefix=&#47;usr&#47;local&#47;Cellar&#47;openresty&#47;1.15.8.2 --pid-path=&#47;us<br>Last 15 lines from &#47;Users&#47;hejunbin&#47;Library&#47;Logs&#47;Homebrew&#47;openresty&#47;01.configure:<br>DYNASM    host&#47;buildvm_arch.h<br>HOSTCC    host&#47;buildvm.o<br>HOSTLINK  host&#47;buildvm<br>BUILDVM   lj_vm.S<br>BUILDVM   lj_ffdef.h<br>BUILDVM   lj_bcdef.h<br>BUILDVM   lj_folddef.h<br>BUILDVM   lj_recdef.h<br>BUILDVM   lj_libdef.h<br>BUILDVM   jit&#47;vmdef.lua<br>make[1]: *** [lj_folddef.h] Segmentation fault: 11<br>make[1]: *** Deleting file `lj_folddef.h&#39;<br>make[1]: *** Waiting for unfinished jobs....<br>make: *** [default] Error 2<br>ERROR: failed to run command: gmake -j8 TARGET_STRIP=@: CCDEBUG=-g XCFLAGS=&#39;-msse4.2 -DLUAJIT_NUMMODE=2 -DLUAJIT_ENABLE_LUA52COMPAT&#39; CC=cc PREFIX=&#47;usr&#47;local&#47;Cellar&#47;openresty&#47;1.15.8.2&#47;luajit<br><br>If reporting this issue please do so at (not Homebrew&#47;brew or Homebrew&#47;core):<br>  https:&#47;&#47;github.com&#47;openresty&#47;homebrew-brew&#47;issues<br><br>These open issues may also help:<br>Can&#39;t install openresty on macOS 10.15 https:&#47;&#47;github.com&#47;openresty&#47;homebrew-brew&#47;issues&#47;10<br>Fails to install OpenResty https:&#47;&#47;github.com&#47;openresty&#47;homebrew-brew&#47;issues&#47;5<br>The openresty-debug package should use openresty-openssl-debug instead https:&#47;&#47;github.com&#47;openresty&#47;homebrew-brew&#47;issues&#47;3<br><br>macOS 10.15.1 安装失败。参考给出的链接也没有解决问题，求老师解惑。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 10.15是新出的，我没升级，看这些信息像是openresty在这上面安装有问题（luajit编译失败），可以向官方反应一下，只能期待官方更新包了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-11-15 09:44:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/1d/d6/76fe5259.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Dream.</span>
  </div>
  <div class="_2_QraFYR_0">Linux-CentOS 7下，修改了&#47;etc&#47;hosts的域名与IP的映射关系后<br><br>再使用.&#47;run.sh start启动OpenResty之后<br><br>curl localhost 或者 curl http:&#47;&#47;www.chrono.com都是返回403<br><br>按之前课程里的url访问https:&#47;&#47;www.chrono.com&#47;11-1什么的，都返回404<br><br>第一次接触OpenResty，麻烦老师回复下是哪里没配置好嘛？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我这里没有遇到你这样的现象。<br><br>先用run.sh list看看Nginx进程是否正常运行，然后用netstat等工具检查一下监听端口，是否有防火墙什么的其他应用阻碍了服务。<br><br>可以问问周围熟悉Linux运维的同事。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-17 22:28:00</div>
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
  <div class="_2_QraFYR_0">这个专栏这么好玩，留言的人这么少，真可惜。<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 答疑篇来的太晚了，没赶上当初的热度，不过总会有需要的同学看到的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-16 14:46:30</div>
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
  <div class="_2_QraFYR_0">老师又来了，很高兴再次见到老师。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: nice to meet you again.</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-13 13:35:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/a6/0b/d296c751.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>果果</span>
  </div>
  <div class="_2_QraFYR_0">当初费了好些时间，才在mac上搭建了环境</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这篇答疑来晚了，实在是抱歉。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-09 17:29:52</div>
  </div>
</div>
</div>
</li>
</ul>