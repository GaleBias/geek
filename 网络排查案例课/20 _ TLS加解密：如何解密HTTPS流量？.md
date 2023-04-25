<audio title="20 _ TLS加解密：如何解密HTTPS流量？" src="https://static001.geekbang.org/resource/audio/a4/42/a4697d44cb29fd97a286acf1d29bda42.mp3" controls="controls"></audio> 
<p>你好，我是胜辉。</p><p>在上节课里，我们对TLS的整体的知识体系做了总览性的介绍，然后回顾了两个实际的案例，从中领略了TLS握手的奥妙。我们也知道了，TLS握手的信息量还是很大的，稍有差池就可能引发问题。我们只有对这些知识有深刻的理解，才能更准确地展开排查。</p><p>不过，也正因为这种种严苛的条件，TLS才足够安全，因为满足了这些前提条件后，真正的数据传送就令人十分放心了。除非你能调动超级计算机或者拥有三体人的智慧，要不然一个TLS连接里面的加密数据，你是真的没有办法破解的。</p><p>可话说回来，<strong>如果排查工作确实需要我们解开密文，查看应用层信息，那我们又该如何做到呢？</strong></p><p>所以在这节课里，我会带你学习TLS解密的技术要点，以及背后的技术原理，最后进行实战演练，让加密不再神秘。好了，让我们开始吧。</p><h2>TLS加密原理</h2><p>在上节课里我们已经了解到，TLS是结合了对称加密和非对称加密这两大类算法的优点，而密码套件是四种主要加密算法的组合。那么这些概念，跟我们的日常工作又有着什么样的交集呢？</p><h3>解读TLS证书</h3><p>下面这个证书，是我在访问站点<a href="https://sharkfesteurope.wireshark.org">https://sharkfesteurope.wireshark.org</a>的时候获取到的，我们来仔细读一下这里面的内容，看看哪些是跟我们学过的TLS知识相关的。</p><!-- [[[read_end]]] --><p>我把图中的很多关键信息做了标记，希望可以帮助你更好地理解。</p><p><img src="https://static001.geekbang.org/resource/image/cb/cf/cbb41d50695d62f090ec5804b6728bcf.jpg?wh=476x762" alt=""></p><p>从上到下，我们了解了这张证书所在的证书链，然后是证书名称、身份验证和签名算法、有效期。不过，看完这个证书，你可能也发现了一个小问题：站点名称跟证书名称不一致？这两个不匹配，浏览器为啥不报错呢？</p><p><img src="https://static001.geekbang.org/resource/image/e4/84/e4d8b4c388b7599a1f765b8b951abe84.jpg?wh=1684x734" alt="图片"></p><p>其实，这里的站点名称跟证书实际上是匹配的，但它匹配的不是Common Name，而是另外一个概念：SAN。</p><p>TLS证书为了支持更多的域名，设计了一个扩展选项Subject Alternative Name，简称 <strong>SAN</strong>，它就包含有多个域名。比如还是这张证书，它的SAN里的域名里就有wireshark.org、sni.cloudflaressl.com，还有跟这次访问的站点名直接相关的*.wireshark.org。这个是通配符域名，就意味着sharkfesteurope.wireshark.org也被支持了。SAN列表如下：</p><p><img src="https://static001.geekbang.org/resource/image/c8/dd/c84e5165254173fccb30e56c2dddb7dd.jpg?wh=964x1510" alt="图片"></p><p>这里也有一个小的注意点：通配符证书只能支持一级域名，比如*.wireshark.org证书可以支持以下域名：</p><ul>
<li>a.wireshark.org</li>
<li>b.wireshark.org</li>
</ul><p>但不支持这样的域名：</p><ul>
<li>a.b.wireshark.org</li>
<li>a.b.c.wireshark.org</li>
</ul><p>然后我们再来温习一下<strong>密码套件</strong>。在这张证书里，我们能看出它用到的密码套件是什么了吗？下面我们来解读一下。</p><p><strong>密钥交换算法是什么呢？</strong>这在证书里看不出来，需要根据握手协商的结果来判定。不过，我们也可以有个初步的判断。如果这次通信用的TLS版本是1.3，那么就是DHE或者ECDHE这样的“前向加密”的密钥交换算法了。结尾的E是Ephemeral，意思是“短时间的”，也就是密钥是每次会话临时生成的。</p><blockquote>
<p>补充：稍后我会介绍什么是前向加密。</p>
</blockquote><p><strong>身份验证和签名算法呢？</strong>就是证书里明确写着的ECDSA，其中EC就是Elliptic Curve的缩写，也就是椭圆曲线算法，它可以用更短的密钥达到跟RSA同样的密码强度。后面跟着的SHA-256是哈希摘要算法，证书内容用这个SHA-256算法做了哈希摘要，然后用ECDSA算法对摘要值做了签名，这样的话，客户端就可以验证这张证书的内容有没有被篡改了。</p><p><img src="https://static001.geekbang.org/resource/image/8b/e6/8b0df16302e77710e097abe770978fe6.jpg?wh=432x254" alt="图片"></p><p><strong>对称加密算法又是什么呢？</strong>在证书这里看不出来，因为它也是通过握手协商出来的。当然，用OpenSSL或者curl命令就可以观察到，我们稍后演示。</p><p><strong>最后是完整性校验算法了。</strong>其实在2里面已经提过了，是SHA-256。</p><p>我们用OpenSSL命令，可以直接观测到这次TLS里协商出来的密码套件：</p><pre><code class="language-bash">$ openssl s_client -connect sharkfesteurope.wireshark.org:443
......
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
......
</code></pre><p>原来，这里用的就是最新的TLS1.3版本，密码套件是TLS_AES_256_GCM_SHA384。你有没有发现它跟TLS1.2的那些密码套件相比还有一个区别呢？比如跟下面这个TLS1.2的密码套件比较一下：</p><center>
<p><strong>TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA</strong></p>
</center><p>相比之下，TLS1.3的套件<strong>TLS_AES_256_GCM_SHA384</strong>是不是少了两个算法：身份验证和签名算法，还有密钥交换算法。不过，身份验证和签名算法倒是可以从证书里看到，它是ECDSA。那密钥交换算法又到哪里去了呢？</p><p><img src="https://static001.geekbang.org/resource/image/d3/b7/d39bcc2b12f573c5866ed5198491aab7.jpg?wh=2000x844" alt=""></p><p>其实，这是因为TLS1.3只允许前向加密（PFS）的密钥交换算法了，所以使用静态密钥的RSA已经被排除了，它默认使用的是DHE和ECDHE，所以就不写在密码套件名称里了。</p><p>那么在这里，就又涉及了一个新的概念：前向加密。</p><h3>前向加密（PFS）</h3><p>前向加密又称为“完美前向加密”，它的英文就是Forward Secrecy和Perfect Forward Secrecy。</p><p>虽然我前面刚提到TLS1.3去掉了RSA，不过你不要误会了，TLS1.3只是把RSA从密钥交换算法中排除了，但证书签名算法还是可以用RSA的。</p><p><strong>为什么TLS1.3强制要求前向加密呢？</strong>这是因为，如果在密钥交换的时候用非前向加密的算法（比如RSA），那么一旦黑客取得了服务端私钥，并且抓取了历史上的TLS密文，他就可以用这个私钥和抓包文件，把这些TLS会话的对称密钥给还原出来，从而破解所有这些密文。因为可以把之前的密文都破解，RSA就不属于“前向”加密。</p><p>人们发现，要解决这个问题的关键，就要做到：<strong>每次参与协商对称密钥时的非对称密钥都不一样</strong>。这样的话，即使黑客破解了其中一次会话的密钥，也无法用这个密钥破解其他会话。</p><p>我们可以用一个例子来帮助理解“前向加密”。假设我们不断地生成一个个的保险箱，相当于一个个的TLS加密报文，如果每个箱子用同样的锁，那么一旦其中一把锁被破解，所有的保险箱都可以被打开了。用上“前向加密”锁之后，每次新的保险箱都用不同的锁，那么即使一把锁被破解，损失的只是一个保险箱，其他的箱子依旧安全。</p><h2>TLS的软件实现</h2><p>TLS只是一套协议，主要是“动动嘴皮子”，具体的活当然还是代码来干。目前应用最为广泛的SSL/TLS实现可能就是OpenSSL了，它既是一个开发库，也是一个命令行工具的名称。另外，NSS和GnuTLS也是开源的TLS实现。应用程序会基于这些TLS库来实现TLS加解密功能。</p><p><img src="https://static001.geekbang.org/resource/image/04/ec/0457ed7f7d9a730b03393ee834526aec.jpg?wh=1548x681" alt=""></p><p>有没有觉得这个很像OSI的分层模型？业务代码工作在应用层，TLS库工作在表示层和会话层，两层之间有交互也有解耦，起到了很好的协同的效果。</p><p><img src="https://static001.geekbang.org/resource/image/5d/e6/5d42c523686ffa6f5edfba03384f3fe6.jpg?wh=2000x980" alt=""></p><p>学习完TLS加密原理，我们就要进入动手环节了，也就是期待已久的TLS抓包解密，让秘密不再是秘密。</p><h2>客户端如何做TLS解密？</h2><p>这里说的客户端，包括了Chrome、Firefox等浏览器，也包括curl这样的命令行工具。我在上节课里提过，为了把TLS解密，我们需要完成几个前提条件。其实这些前提条件就是下面这三件事：</p><ul>
<li>创建一个用来存放key信息的日志文件，然后在系统里<strong>配置一个环境变量SSLKEYLOGFILE</strong>，它的值就是这个文件的路径。</li>
<li><strong>重启浏览器</strong>，启动抓包程序，然后访问HTTPS站点，此时TLS密钥信息将会导出到这个日志文件，而加密报文也会随着抓包，被保存到抓包文件中。</li>
</ul><blockquote>
<p>补充：如果是Mac又不想改动全局配置，那么你可以在terminal中的 <code>export SSLKEYLOGFILE=</code>路径，然后执行 <code>open "/Applications/Google\ Chrome.app"</code>，这时Chrome就继承了这个shell父进程的环境变量，而terminal退出后，这个环境变量就自动卸除了。</p>
</blockquote><ul>
<li>在Wireshark里，打开Preferences菜单，在Protocol列表里找到TLS，然后把<strong>(Pre)-Master-Secret log filename配置为那个文件的路径</strong>。</li>
</ul><p><img src="https://static001.geekbang.org/resource/image/0b/b8/0be2c0e7c0e6a9b2949634a24a6758b8.jpg?wh=1830x1004" alt="图片"></p><p>在做完这三件事之后，我们用Wireshark打开抓包文件，就能看到解密后的报文了，比如HTTP请求和响应，还有TLS的控制信息，都会展示为明文。</p><p>比如，在默认情况下，我们看到的会是密文：</p><p><img src="https://static001.geekbang.org/resource/image/ed/87/ed692a27ba9f3471821663abe0498187.jpg?wh=1119x299" alt="图片"></p><blockquote>
<p>补充：抓包示例文件已经上传至<a href="https://gitee.com/steelvictor/network-analysis/blob/master/20/lesson20.pcap">Gitee</a>，建议结合抓包文件和文稿一起学习。</p>
</blockquote><p>而配置解密步骤之后，看到的就是明文了：</p><p><img src="https://static001.geekbang.org/resource/image/fb/3c/fb4a1972d1786034c5735404950f2a3c.jpg?wh=1117x299" alt="图片"></p><p>好了，你可以看到明文了，很多应用层的信息都可以辅助你做排查了，是不是有点小小的激动？</p><p>那么这背后的原理是什么呢？</p><p>其实是这样的：浏览器在启动过程中会尝试读取SSLKEYLOGFILE这个环境变量。如果存在这个变量，而它指向的又是一个有效的文件，那么浏览器就会做最为关键的事情了：它去调用TLS库，让TLS库把访问HTTPS站点的过程中的key信息导出到SSLKEYLOGFILE文件中。我画了一张示意图供你参考：</p><p><img src="https://static001.geekbang.org/resource/image/64/6d/649e9001c4391ddec54e93fb9784bb6d.jpg?wh=2000x671" alt=""></p><p>整个过程倒是不难理解，不过你可能会好奇：为什么这个日志文件有这么强大的能力，能解密TLS？然后又不免担心，<strong>如果这个文件“被坏人利用了”，该怎么办？</strong></p><p>所以我们还需要近距离地认识一下SSLKEYLOGFILE。</p><h3>SSLKEYLOGFILE</h3><p>这个文件之所以能够解密TLS，最关键的是，TLS库把密钥交换阶段的核心信息Master secret导出到了这个文件中。基于这个信息，Wireshark就可以还原出当时的对称密钥，从而破解密文。</p><p>我们先来认识一下SSLKEYLOGFILE的格式。它是由很多条记录组成的，对于TLS1.2来说，每一行是一条记录，每一条记录由3个部分组成，中间用空格分隔，也就是下面这样：</p><ul>
<li><code>&lt;Label1&gt; &lt;ClientRandom1&gt; &lt;Secret1&gt;</code></li>
<li><code>&lt;Label2&gt; &lt;ClientRandom2&gt; &lt;Secret2&gt;</code></li>
<li>……</li>
</ul><p>这三个部分的具体含义是这样的。</p><p><strong>Label：是对这条记录的描述，对于TLS1.2来说，这个值一般是CLIENT_RANDOM</strong>。另外，RSA也是一个可能的值，但是前面说过，因为RSA算法在密钥交换方面不是前向加密的，所以已经不推荐使用了。所以如果你在日志文件里看到RSA，可能要小心一点，说明你的TLS不是前向加密的，所以并不是很安全。</p><p><strong>ClientRandom：这个部分就是客户端生成的随机数</strong>，随机数原始长度为32字节，但在这里是用16进制表示的，每个字节用16进制表示就会成为2个字符，所以就变成了64个字符的字符串。我们在抓包文件里也能看到它，因为在密钥交换算法的设计中，ClientRandom就是要在网络上公开传输的。</p><p><strong>Secret：这就是Master secret，也就是通过它可以生成对称密钥</strong>。Master secret固定是48字节，也是十六进制表示的关系，成为96个字节的字符串。你应该明白了，这个Master secret就是最为关键的信息了，也正是黑客苦苦寻求的东西。它是万万不能在网络上传输的，自然也不可能在抓包文件里看到它，只有TLS库才能导出它。</p><blockquote>
<p>补充：TLS1.3的格式会很不一样，具体细节你可以参考这里的<a href="https://firefox-source-docs.mozilla.org/security/nss/legacy/key_log_format/index.html">链接</a>。</p>
</blockquote><p>我们来看一个TLS1.2的KEYLOGFILE的具体的例子：</p><pre><code class="language-bash">CLIENT_RANDOM 770c2c73ef1ab58dda9360a94587e5f8b0a80c0b1abf628ddd7b55a118ec18ec bea2c01c5b6f9c577e8ba251c8f262adf33c5aa31a238d464a9c56dbd1bf30cf55cbf14e6175102fa1db9b8a0183a721
</code></pre><blockquote>
<p>补充：这个日志文件也已经上传至<a href="https://gitee.com/steelvictor/network-analysis/blob/master/20/lesson20sslkey.log">Gitee</a>，你可以按照前面介绍的3个步骤，结合上面的抓包文件和这个日志文件，自己来观察解密前后的区别。</p>
</blockquote><p>在输出这个key信息的时候，我们也做了对应的抓包，现在看一下抓包文件。我们选中Client Hello报文，点开TLS详情部分，继续点开TLSv1.2 Record Layer -&gt; Handshake Protocol -&gt; Random，你看看它是不是就是前面日志文件里，第二列的值（开头是770c）？</p><p><img src="https://static001.geekbang.org/resource/image/86/a8/86979e6ba6087a4cf6a13e46fdb675a8.jpg?wh=863x359" alt="图片"></p><p>SSLKEYLOGFILE日志文件格式我们了解了，接下来了解Wireshark是怎么跟它协同工作，解开密文的。</p><h2>Wireshark是怎么解开密文的？</h2><p>在TLS1.2的SSLKEYLOGFILE中，每条记录的第一列是CLIENT_RANDOM这个字符串，第二列是这个client random的值，Wireshark就是通过它，找到对应的TLS会话（你可以理解为是TCP流）。就像上图所示，通过这个随机数，就找到了这条KEYLOG记录对应的TLS会话了。</p><p><img src="https://static001.geekbang.org/resource/image/a8/57/a89a94c72aca54686fb495d861b4c257.jpg?wh=1234x454" alt="图片"></p><p>那么接下来，Wireshark就知道真正的Master secret在哪里了：它就是前面匹配了这个客户端随机数的记录的第三列，也就是那96个字节的字符串。</p><p><img src="https://static001.geekbang.org/resource/image/44/df/443c085877yy55b4891760eeba5b3fdf.jpg?wh=1242x52" alt="图片"></p><p>由于在抓包文件里就有ECDHE密钥交换算法所需要的各种参数，结合这里的Master secret，Wireshark就可以解析出对称密钥，从而把密文解密了！</p><p><img src="https://static001.geekbang.org/resource/image/a6/9d/a6d8303e04ae39e8aa465c40e4852b9d.jpg?wh=1241x434" alt="图片"></p><h3>SYSKEYLOGFILE的安全性</h3><p>现在回答一下前面的问题：如果这个文件“被坏人利用了”，怎么办？</p><p>通过前面的学习，我们知道了：要想破解密文，既要有抓包文件，也要有SSLKEYLOGFILE日志文件，两者结合才能解密。而且不要忘了，即使你有抓包文件和日志文件，要是没抓到TLS握手阶段的报文，也还是不能解密，因为缺少了客户端随机数、加密算法参数等信息，Master secret对你也是无法下嘴的美食。</p><h2>服务端如何做TLS解密？</h2><p>其实，上面的客户端做解密的过程，网络上已经有很多资料了，但是接下来要介绍的“服务端做TLS解密”这个话题，却鲜有人讨论。要知道，TLS是双方的加密任务，但是我们一边倒地关心客户端如何解密，却对服务端的解密不闻不问，这又是什么道理呢？</p><p>我觉得可能有这么几个原因：</p><ul>
<li>多数人接触的还是以客户端为主，能在客户端解密已经满足了大多数的需求。而服务端只有一部分专职运维或者开发工程师在维护，关注度少了很多。</li>
<li>服务端一般有详细的日志，比如Nginx可以向日志里输出HTTP头部和性能数据，还可以输出TLS选择的密码套件等信息，这些在一般场景下也够用了。</li>
<li>很多服务端程序并没有提供TLS解密的功能，也就是想做抓包解密也做不了。而要自己实现这个特性，难度跟简单的参数配置，不在一个等级。难度高，可能是更加关键的原因。</li>
</ul><p>其实，在软件架构上，服务端和客户端也是类似的，也是基于TLS库来构建TLS加解密的能力的。</p><p><img src="https://static001.geekbang.org/resource/image/c9/5f/c94f7c2d6a59455ec03792288793a55f.jpg?wh=1591x673" alt=""></p><p>就我们eBay的情况来说，在我们的软件负载均衡方案中，第七层的部分是基于<a href="https://www.envoyproxy.io/docs/envoy/latest/intro/what_is_envoy">Envoy</a>实现的。我们在把一套新系统搬上生产环境之前有一系列的考量，就像体检的检查表一样逐一校验。我们发现Envoy的体检就有一项“不合格”：Envoy不提供对TLS流量进行抓包解密的功能。</p><blockquote>
<p>补充：Envoy是硅谷的共享出行公司Lyft于2016年发起的开源项目，可以认为是云原生时代的Nginx。</p>
</blockquote><p>在不少情况下，这个抓包解密特性对排查很有用。像一些商业产品比如Netscaler就是可以一边抓包，一边导出TLS key。就跟我们在客户端做解密类似，我们结合这两个文件，就可以在Wireshark中既观察到TCP行为，也读到应用层信息了。</p><p>你可能会问：“Envoy可以输出Web日志呀，把各种HTTP性能指标、访问头部，甚至包括用了什么Cipher，都输出到文件，这样还不香吗？”</p><p>其实，我在开篇词和<a href="https://time.geekbang.org/column/article/480068">第4讲</a>中都提到过网络排查的两大鸿沟。</p><ul>
<li><strong>应用现象跟网络现象之间的鸿沟</strong>：你可能看得懂应用层的日志，但是不知道网络上具体发生了什么。</li>
<li><strong>工具提示跟协议理解之间的鸿沟</strong>：你看得懂 Wireshark、tcpdump 这类工具的输出信息的含义，但就是无法真正地把它们跟你对协议的理解对应起来。</li>
</ul><p>而包括Envoy在内的反向代理和LB软件，虽然也都提供了应用层日志，但跟实际的网络行为还有距离，这就是“应用现象跟网络现象之间的鸿沟”：日志是日志，网络是网络。如果日志里说某个HTTP请求耗时很长，你是无法知道网络上到底什么问题导致了这么长的耗时，是丢包引起了重传？还是没有丢包，纯粹是传输速度慢呢？</p><p>为了跨越第一个鸿沟，我们选择做tcpdump抓包。但是，如果抓取到的TLS密文无法被解密，就无法知道这些究竟是应用层的什么信息，这个鸿沟依然没有被跨越。</p><p><img src="https://static001.geekbang.org/resource/image/2d/9f/2d86df6591f65bba7425d10b7426819f.jpg?wh=1828x1044" alt=""></p><p>当然，我们可以选择“妥协”，采用一些灵活的策略来开展抓包分析。比如，可以把抓包分析的重心转移到TCP和TLS本身的层面，而不再关心其承载的应用层信息。但是“隔靴搔痒”总让排查工作不是特别“痛快”，我们犹豫许久之后，还是决定把这个解密的特性实现！</p><p>这并不是一件容易的事情。不过通过调研，我们发现，其实在服务端启用跟客户端类似的TLS解密功能，技术上是可行的，其中最为关键的信息就在2017年一位Wireshark开发工程师的<a href="https://sharkfesteurope.wireshark.org/assets/presentations17eu/15.pdf">演讲</a>中：</p><blockquote>
<p>Applications using OpenSSL 1.1.1 or BoringSSL d28f59c27bac (2015-11-19) can be configured to dump keys: void SSLCTXsetkeylogcallback(SSL CTX ∗ctx, void (∗cb)(const SSL ∗ssl, const char ∗line));</p>
</blockquote><p>也就是说，只要我们使用这个BoringSSL（是谷歌Fork自OpenSSL的项目）的 <strong>SSLCTXsetkeylogcallback()</strong> 回调函数，就可以把TLS信息导出来，于是我们信心大增。核心突破就是要把这个BoringSSL的回调函数给用起来。</p><p>具体来说，我们需要做这样的几件事：</p><ul>
<li>在Envoy代码中增加调用SSLCTXsetkeylogcallback()函数的逻辑。</li>
<li>增加了对外的接口，使得用户可以通过某种方式让Envoy知道，它需要去使用这个调用逻辑。</li>
</ul><p>第二点，其实就是一种接口方式，比如SSLKEYLOGFILE环境变量就是一种，我们也可以选择API接口，或者某种别的接口。总之，只要让程序（这里是Envoy）知道：它需要去叫BoringSSL这个小弟去办点事情，整个功能就可以运作起来了。你可以参考这张示意图：</p><p><img src="https://static001.geekbang.org/resource/image/95/82/950d019fayye428c046c3d4504aa9082.jpg?wh=1707x785" alt=""></p><p>“体检通过”！我们在服务端Envoy上也可以做到方便的TLS解密了。其实，不仅实现了这个具体的需求本身，也实现了我们作为技术工作者，对“自我实现”的需求。</p><blockquote>
<p>这个特性的主要开发者张博已经提交了PR，相信不久之后我们就能在正式版的Envoy里用上这个特性了。</p>
</blockquote><p>实现Envoy的TLS抓包解密的具体的做法跟客户端解密的步骤差不多：</p><ul>
<li>调用Envoy接口，启用SSL KEY导出功能。</li>
<li>做tcpdump抓包，然后把抓包文件和KEY文件复制出来。</li>
<li>在Wireshark里同样配置好<strong>TLS协议的(Pre)-Master-Secret log filename</strong>，打开抓包文件后，就可以跟在客户端类似，直接看到明文了。</li>
</ul><h2>几个问题</h2><p>然后到这里，这里我们还需要搞清楚几个问题。</p><h4>问题1：我想实时查看解密信息行不行？</h4><p>一个字的答案：行。只要设置好前面提及的3件事：</p><ul>
<li>SSLKEYLOGFILE环境变量；</li>
<li>之后再启动浏览器，然后直接在Wireshark里开始抓包；</li>
<li>设置Wireshark的TLS协议，配置(Pre-)Master secret logfile。</li>
</ul><p>这时候你访问HTTPS站点时，在Wireshark里看到的就直接是解密好的信息了！因为Wireshark已经能从SSLKEYLOGFILE里读取到密钥信息，同时又在实时地抓取到TLS密文，这种解密工作是可以实时进行的。</p><p>当然，这里还有一个小的注意点，我们在第二个问题里展开。</p><h4>问题2：为什么停止抓包后再启动抓包，抓包文件又变成密文了？</h4><p>有同学就遇到这个问题：重启浏览器后，在Wireshark里马上就能看到HTTP数据包，确实能解密。但是停止抓包之后，再启动抓包，看到的又变成了TLS密文了。必须得重启浏览器才行。这是为何呢？</p><p>表面上看，这似乎又是一个“重启大法”的问题，但本质上呢？</p><p>我们知道，密文是用对称密钥加密的。而对称密钥的生成，是在TLS握手阶段完成的。我们前面提到过，Wireshark（也包括其他需要读取SSLKEYLOGFILE的程序）正是根据第二列的客户端随机数，来找到抓包文件中的TLS session，然后运用第三列的Master secret来获取到对称密钥的。</p><p>抓包停止后，新的HTTPS请求所触发的TLS握手就不会被抓取到。这也就意味着，Wireshark没有抓取到客户端随机数这个关键信息，尽管SSLKEYLOGFILE里依然在输出着一行行的key信息，但是Wireshark已经不知道用哪个Master secret了。自然，解密就无从做起。</p><p><img src="https://static001.geekbang.org/resource/image/50/50/50dbcec8fe9a381d06b6b50a8c72fd50.jpg?wh=2000x478" alt=""></p><p>而在浏览器重启后，事实上造成了TLS的重新握手，此时就又可以抓取到客户端随机数了，这样，解密工作就可以恢复。你看，这其实跟<a href="https://time.geekbang.org/column/article/479163">第3讲</a>中，没抓到TCP握手报文就无法知道Window Scale参数这个问题差不多，也是关于握手的，只不过这次是TLS握手。“技术是相通的”，这句话真不是随便说说。</p><h2>小结</h2><p>这节课，我们通过对一张真实的TLS证书的解读，复习了各个加密算法在现实场景中的实现。你也需要重点掌握以下知识点：</p><ul>
<li>证书中的SAN列表包括了它所支持的站点域名，所以只要被访问的站点名称在这个列表里，名称匹配就不是问题了。</li>
<li>证书中的域名通配符只支持一级域名，而不支持二级或者更多级的域名。</li>
<li>在TLS1.3中，密钥交换算法被强制要求是前向加密算法，所以默认采用DHE和ECDHE，而RSA已经弃用。</li>
<li>RSA依然可以作为可靠的身份验证和签名算法来使用。另外一种验证和签名算法是ECDSA，它可以用更短的密钥实现跟RSA同样的密码强度。</li>
<li>前向加密可以防止黑客破解发生在过去的加密流量，提供了更好的安全性。</li>
</ul><p>之后就是这节课的核心了：<strong>如何做到对抓包文件进行解密</strong>。这里又分客户端和服务端两个不同场景，你也需要重点关注。</p><p>首先，在客户端做抓包解密，需要做三件事：</p><ul>
<li>创建一个文件，并设置为SSLKEYLOGFILE这个环境变量的值；</li>
<li>重启浏览器，开始做抓包，此时key信息被浏览器自动导入到日志文件；</li>
<li>在Wireshark里把该日志文件配置为TLS的(Pre)-Mater-Secret log filename。</li>
</ul><p>这样，我们就能在Wireshark里直接读取到应用层信息了。</p><p>而在服务端抓包解密，就要依托于软件实现了，但是有些软件并没有提供这种功能，比如Envoy。借助底层BoringSSL库的接口，eBay流量管理团队实现了对这个接口的调用，我们也可以在Envoy上完成抓包解密了。</p><p>另外，你还要知道<strong>Wireshark能解读出密文的原理</strong>：</p><ul>
<li>从抓包文件中定位到client random；</li>
<li>从日志文件中找到同样这个client random，然后找到紧跟着的Master secret；</li>
<li>用这个Master secret导出对称密钥，最后把密文解密。</li>
</ul><h2>思考题</h2><p>最后，给你留两道思考题：</p><ul>
<li>DH、DHE、ECDHE，这三者的联系和区别是什么呢？</li>
<li>浏览器会根据SSLKEYLOGFILE这个环境变量，把key信息导出到相应的文件，那么curl也会读取这个变量并导出key信息吗？</li>
</ul><p>欢迎你把答案分享到留言区，我们一起交流、进步。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/7f/d3/b5896293.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Realm</span>
  </div>
  <div class="_2_QraFYR_0">问题一：<br>1 DH算法是为了解决密钥协商的算法,Bob和Alice分别用对方的公钥和自己的私钥，一通骚操作后，得到相同的会话密钥k,这就解决了密钥不直接传输而通过协商出来；<br>2 DH有static DH和DHE两种实现，static的方式，私钥是不变的，有被破解的可能性，进而搞出来DHE,每次双方的私钥都变化，安全性提高了不少；<br>3 DHE算法性能不行，然后出现基于椭圆曲线的ECDHE;<br>参考:https:&#47;&#47;www.likecs.com&#47;default&#47;index&#47;show?id=124371<br><br>问题二：<br>经过测试curl也会读取SSLKEYLOGFILE，并把随机数和secret写入到这个文件中<br>SSLKEYLOGFILE=&#47;my&#47;path&#47;to&#47;file.log curl https:&#47;&#47;example.com<br>参考:https:&#47;&#47;davidhamann.de&#47;2019&#47;08&#47;06&#47;sniffing-ssl-traffic-with-curl-wireshark&#47;</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 很好：）我提第二个问题也是想鼓励大家去实践一下，这个过程会把知识掌握的更好，你做的很赞~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-07 13:43:13</div>
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
  <div class="_2_QraFYR_0">Secret：这就是 Master secret，也就是通过它可以生成对称密钥。----------&gt;老师，这个应该是premaster吧，fun(client 随机数+server 随机数+premaster） 才算出master 吧</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以参考这个文档：https:&#47;&#47;firefox-source-docs.mozilla.org&#47;security&#47;nss&#47;legacy&#47;key_log_format&#47;index.html<br>RSA: 48 bytes for the premaster secret, encoded as 96 hexadecimal characters (removed in NSS 3.34)<br><br>CLIENT_RANDOM: 48 bytes for the master secret, encoded as 96 hexadecimal characters (for SSL 3.0, TLS 1.0, 1.1 and 1.2)<br><br>如果keylog文件中，记录行的开头是CLIENT_RANDOM，那么第三列的数字是master secret。如果开头是RSA，那么第三列就是premaster secret。<br>供你参考：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-11 21:40:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/bf/35/0e3a92a7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>晴天了</span>
  </div>
  <div class="_2_QraFYR_0">对如何在linux抓取https明文包有点困惑 老师有什么工具推荐吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你好，课程里有介绍，对于一个客户端来说，就是下面三个步骤：<br>1. 创建一个用来存放 key 信息的日志文件，然后在系统里配置一个环境变量 SSLKEYLOGFILE，它的值就是这个文件的路径。<br>2. 重启浏览器，启动抓包程序，然后访问 HTTPS 站点，此时 TLS 密钥信息将会导出到这个日志文件，而加密报文也会随着抓包，被保存到抓包文件中。补充：如果是 Mac 又不想改动全局配置，那么你可以在 terminal 中的 export SSLKEYLOGFILE=路径，然后执行 open &quot;&#47;Applications&#47;Google\ Chrome.app&quot;，这时 Chrome 就继承了这个 shell 父进程的环境变量，而 terminal 退出后，这个环境变量就自动卸除了。<br>3. 在 Wireshark 里，打开 Preferences 菜单，在 Protocol 列表里找到 TLS，然后把 (Pre)-Master-Secret log filename 配置为那个文件的路径。<br><br>如果你说的linux是指服务端，那要看具体的应用本身。比如，如果这台linux是nginx，那么可以参考我对“宝仔”铜须的提问的答复，即：默认不行，所以有人写了一个程序模块在做这件事，参考https:&#47;&#47;security.stackexchange.com&#47;questions&#47;216065&#47;extracting-openssl-pre-master-secret-from-nginx <br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-04-08 17:01:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/52/5c/d4a9accb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>仄言</span>
  </div>
  <div class="_2_QraFYR_0">linux  客户端 使用tcpdump  怎么抓取https 解密?</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: tcpdump只是抓取网络报文（TLS报文），具体的解密是需要密钥文件的，而密钥文件是通过“openssl库能通过SSLKEYLOGFILE这个系统变量来导出”这个特性来实现的。<br>有了tcpdump的抓包文件，结合密钥文件，wireshark就可以解出密文。<br><br>如果你问的是tcpdump抓取https流量的具体命令，那就很简单。如果https是在443端口上，那就是tcpdump port 443<br>如果要导出到文件，就假设-w file.pcap，比如变成tcpdump -w file.pcap port 443<br>如果还是抓取不到报文，有可能要指定抓取所有网卡接口，加上-i any，就变成tcpdump -i any -w file.pcap port 443</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-13 10:31:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/34/bb/0b971fca.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>walker</span>
  </div>
  <div class="_2_QraFYR_0">另外，在抓取凤凰资讯的时候，发现返回两条<br>第一条： 5850	138.461067	140.143.221.194	192.168.2.104	HTTP2	2838		HEADERS[27]: 200 OK, DATA[27] [TCP segment of a reassembled PDU]<br>第二条：5851	138.461067	140.143.221.194	192.168.2.104	HTTP2	124		DATA[27]<br>第二条中data数据并没有解密，这是怎么回事</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-26 16:46:41</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/34/bb/0b971fca.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>walker</span>
  </div>
  <div class="_2_QraFYR_0">我在抓极客的页面是发现了protocol类型多了一个HTTP&#47;JSON,为什么协议会有 http&#47;json 类型的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你是指wireshark里看到的吗？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-24 16:32:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/47/00/3202bdf0.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>piboye</span>
  </div>
  <div class="_2_QraFYR_0">Just TrustMe 解决 ssl pin 的方法， 老师可以也一块介绍吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: ssl pin是指，做了pin（别针绑定）的证书才会被这个客户端（比如app）信任，其他的证书哪怕是合规CA签发的也不信任。这样可以达到更高的安全级别（当然也会带来别的问题）。<br>你说的Just TrustMe能破解ssl pin，推测是从系统内部hook了跟ssl通信相关的函数。比如，本来走向ssl verification的函数被替换了，而这个替换函数就不会去校验是否有pin了，那就相当于解除了pin。<br>andriod方面我是外行，不过从原理上讲估计就是这样了~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-06 18:05:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/50/33/9dcd30c4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>斯蒂芬.赵</span>
  </div>
  <div class="_2_QraFYR_0">为什么curl的时候，有的可以解析，有的不可以，比如https:&#47;&#47;www.baidu.com可以解密出https，但是curl  https:&#47;&#47;openapi-fxg.jinritemai.com 就解密不出来？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 解密不出来，具体是指什么？keylog文件里没有对应的key信息生成？还是有keylog信息，但是导入到wireshark后还是不能解密？</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-28 16:36:59</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/50/33/9dcd30c4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>斯蒂芬.赵</span>
  </div>
  <div class="_2_QraFYR_0">如果是命令行curl请求，不是通过浏览器访问，是否能解密呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我看你已经自己搞定啦：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-28 14:28:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/f4/d5/1c9c59b4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>青梅煮酒</span>
  </div>
  <div class="_2_QraFYR_0">请问Windows上配置了SSLKEYLOGFILE环境变量后，重启浏览器抓包，文件里面sslkey.log不会写入东西，老师有遇到吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 在重启浏览器之前sslkey.log里有写入东西吗？还是说从来没有没有写入过？如果是后者，那要看看具体步骤是否有遗漏了，你可以再试一下再回复我~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-26 15:59:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>woJA1wCgAASVwFBCYVuFLQY8_9xjIc3w</span>
  </div>
  <div class="_2_QraFYR_0">可以看下这https:&#47;&#47;github.com&#47;ehids&#47;ecapture ，不需要CA证书，即可捕获HTTPS&#47;TLS通信数据的明文</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 好的，也是一个有意思的新项目。根据作者的描述：<br>“本项目hook了&#47;lib&#47;x86_64-linux-gnu&#47;libssl.so.1.1的SSL_write、SSL_read函数的返回值，拿到明文信息，通过ebpf map传递给用户进程。”<br>它的工作机制是“截胡”用户空间程序传给SSL_write和SSL_read函数的值（应该也包括传入的参数），然后读取相应变量的数据出来，也就是下面这样：<br>用户空间程序 -&gt; SSL_write&#47;SSL_read (hook点，数据还未加密) -&gt; syscall write&#47;read（数据已经加密） -&gt; NIC -&gt; network<br><br>虽然上面这个方式也同样能读取到密文，但是它的逻辑不是抓包，所以应该是缺乏下面这些信息的：<br>1. 内核TCP栈对这些加密报文的发送和接收细节，比如时间戳、重传、超时等，因为这些信息是SSL库无法感知到的，所以hook以后也依然无法获取到<br>2. SSL库本身的行为，比如握手信息细节、TLS alert消息等，因为这些不是SSL_write&#47;SSL_read的行为，而是SSL库自身的行为<br><br>而用key导出的方式，就结合了解密功能还有网络行为抓取的功能，信息量更多一些。主要看你的场景：<br>1. 如果是仅仅想解密，可以用eCapture<br>2 如果还要知道网络行为细节，建议用key导出的方式</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-29 17:04:42</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/57/4f/6fb51ff1.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>一步</span>
  </div>
  <div class="_2_QraFYR_0">只要 不乱抓信任其他不明来源的 证书，就没有人可以破解加密的信息。 聊天记录什么的都拿不到</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，这就是PKI的信任基础:)</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-19 21:45:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/4f/ca/3ac60658.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>orange</span>
  </div>
  <div class="_2_QraFYR_0">通过 fiddler抓https的包，在fiddler控制台看到的是明文，是fiddler做了实时解密？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 通过fiddler解密https，首先要让系统信任fiddler的证书，所以这个过程并没有对TLS原理的破解~<br>过程是：客户端请求到fiddler（所以能看到明文请求），fiddler转发给最终站点，后者回复响应给fiddler（所以能看到明文响应），最后转回给客户端~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-11 07:05:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/4f/ca/3ac60658.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>orange</span>
  </div>
  <div class="_2_QraFYR_0">老师我们的公众号进入直接变成了 第三方的充值页面，  这个https可以搞定么？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你好，你是指微信公众号吗？这个第三方充值页面并不是你们设置的对吗？<br>普适的来说，如果https站点是你自己的，别人（ISP等）是无法植入广告甚至劫持的。多年前国内这种现象十分严重，最大的原因是没有启用http，所以可以被随意劫持和植入~<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-10 09:15:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/58/c7/a65f5080.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Lucky  Guy</span>
  </div>
  <div class="_2_QraFYR_0">老师最近刚好遇到一个和TLS相关的问题.问题如下<br>浏览器----https---&gt; nginx(四层正向代理,开启了TLS SNI ) ----https---&gt;源站<br>nginx使用https正向代理网站的时候,个别网站刚开始是可以正常访问的,如果一直执行页面操作(如:点击页面连接,刷新页面)是没问题的,但是只要浏览器闲置一段时间(3分钟左右)后,再次刷新页面就会出现500报错. 目前感觉导致问题的原因是nginx代理到源站这段有设备校验了SNI导致的. 但是抓包也没有找到证据,请问这种情况改如何分析.. </div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 你好，我觉得应该不是SNI的问题。<br>你试试配置nginx去源站的keepalive参数，比如配置为15秒这种比较短的时间，看看问题是否会消失。<br>nginx做正向代理用，那么浏览器跟nginx之间第一个请求是CONNECT G站，然后nginx去尝试TCP连接G站，能通则回复HTTP 200，浏览器会发送真正的请求给nginx，后者跟G站之间建立TCP连接，然后这个请求的TCP报文会原样发给G站，浏览器是直接跟G站做TLS通信的。<br>一般浏览器默认就有做TCP保活，比如15秒一次，但是这些保活报文发给了nginx后，并没有转发给G站（这是我的推测），然后G站在过了一段时间（比如3分钟）后就撤销了这条TCP连接，此时nginx并不知情。<br>你3分钟后再次访问G站， nginx还以为它跟G站的TCP连接依然有效，于是直接发送请求过去，G站应该是回复了RST。nginx把这个信息转化为HTTP 500给浏览器。<br>你按这个思路查一下，然后来这里更新吧~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-08 20:49:20</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/58/c7/a65f5080.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Lucky  Guy</span>
  </div>
  <div class="_2_QraFYR_0">老师你好,最近刚好遇到了一个和证书相关的问题<br>用户(浏览器)--https--&gt; nginx(四层正向代理代理N多个域名) --https--&gt; 源站(G网站) , 用户刚开始访问G网站的时候是可以正常访问的,如果一直频繁打开子页面也是没问题的.只要页面打开后空闲2-3分钟左右(没有数据交互)再次刷新当前页面就会出现500错误,在浏览器后台清理所有sockets后即可恢复正常. 目前只发现正向代理的G网站有这个问题,其他网站都正常. 当前怀疑是G网站校验了SNI.   目前我了解到的是 只有在tls握手的时候回会</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯我看你发了两遍，我已经在另一个地方回啦</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-08 20:14:12</div>
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
  <div class="_2_QraFYR_0">老师你好，服务端用的是nginx，可以做TLS解密码？我指在服务端</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 您好，默认不行，所以有人写了一个程序模块在做这件事，参考https:&#47;&#47;security.stackexchange.com&#47;questions&#47;216065&#47;extracting-openssl-pre-master-secret-from-nginx <br>我实际操作过，是可行的：）<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-08 08:21:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/eb/09/ba5f0135.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Chao</span>
  </div>
  <div class="_2_QraFYR_0">DH 不具备前向安全性， 升级成DHE（E代表临时的），又由于DHE的性能不好。 然后再DHE上增加了ECC（椭圆曲线特性）升级为ECDHE</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 没错：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-07 13:21:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/8f/cf/890f82d6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>那时刻</span>
  </div>
  <div class="_2_QraFYR_0">DH 算法是非对称加密算法， 因此它可以用于密钥交换，该算法的核心数学思想是离散对数。<br><br>根据私钥生成的方式，DH 算法分为两种实现：<br>1. static DH 算法，这个是已经被废弃了， static DH 算法不具备前向安全性。<br>2. DHE 算法，现在常用的；让双方的私钥在每次密钥交换通信时，都是随机生成的、临时的，这个方式也就是 DHE 算法<br><br>DHE 算法由于计算性能不佳，因为需要做大量的乘法，为了提升 DHE 算法的性能，所以就出现了现在广泛用于密钥交换算法 — ECDHE 算法是在 DHE 算法的基础上利用了 ECC 椭圆曲线特性，可以用更少的计算量计算出公钥，以及最终的会话密钥。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 正确而且完整，可以说是标准答案：）</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-07 11:02:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/8f/cf/890f82d6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>那时刻</span>
  </div>
  <div class="_2_QraFYR_0">实际测试了下，curl也会使用 SSLKEYLOGFILE 这个环境变量，并导出key信息</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，curl也同样支持这种方式来导出key信息，其实我们也可以不用浏览器，直接用curl这种命令行工具来做这种解密测试~</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-03-07 10:52:44</div>
  </div>
</div>
</div>
</li>
</ul>