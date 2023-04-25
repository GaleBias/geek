<audio title="42 _ DHEECDHE算法的原理" src="https://static001.geekbang.org/resource/audio/66/7b/669bffe5b009bca02f827d434fec157b.mp3" controls="controls"></audio> 
<p>你好，我是Chrono。</p><p>在<a href="https://time.geekbang.org/column/article/110354">第26讲</a>里，我介绍了TLS 1.2的握手过程，在Client Hello和Server Hello里用到了ECDHE算法做密钥交换，参数完全公开，但却能够防止黑客攻击，算出只有通信双方才能知道的秘密Pre-Master。</p><p>这是TLS握手的关键步骤，也让很多同学不太理解，“为什么数据都是不保密的，但中间人却无法破解呢？”</p><p>解答这个问题必须要涉及密码学，我原本觉得有点太深了，不想展开细讲，但后来发现大家都对这个很关心，有点“打破砂锅问到底”的精神。所以，这次我就试着从底层来解释一下。不过你要有点心理准备，这不是那么好懂的。</p><p>先从ECDHE算法的名字说起。ECDHE就是“短暂-椭圆曲线-迪菲-赫尔曼”算法（ephemeral Elliptic Curve Diffie–Hellman），里面的关键字是“短暂”“椭圆曲线”和“迪菲-赫尔曼”，我先来讲“迪菲-赫尔曼”，也就是DH算法。</p><h2>离散对数</h2><p>DH算法是一种非对称加密算法，只能用于密钥交换，它的数学基础是“<strong>离散对数</strong>”（Discrete logarithm）。</p><p>那么，什么是离散对数呢？</p><p>上中学的时候我们都学过初等代数，知道指数和对数，指数就是幂运算，对数是指数的逆运算，是已知底数和真数（幂结果），反推出指数。</p><!-- [[[read_end]]] --><p>例如，如果以10作为底数，那么指数运算是y=10^x，对数运算是y=logx，100的对数是2（10^2=100，log100=2），2的对数是0.301（log2≈0.301）。</p><p>对数运算的域是实数，取值是连续的，而“离散对数”顾名思义，取值是不连续的，数值都是整数，但运算具有与实数对数相似的性质。</p><p>离散对数里的一个核心操作是模运算，也就是取余数（mod，在C、Java、Lua等语言里的操作符是“%”）。</p><p>假设有模数17，底数5，那么“5的3次方再对17取余数得6”（5 ^ 3 % 17 = 6）就是在离散整数域上的一次指数运算（5 ^ 3 (mod 17) = 6）。反过来，以5为底，17为模数，6的离散对数就是3（Ind(5, 6) = 3 ( mod 17)）。</p><p>这里的（17，5）是离散对数的公共参数，6是真数，3是对数。知道了对数，就可以用幂运算很容易地得到真数，但反过来，知道真数却很难推断出对数，于是就形成了一个“<strong>单向函数</strong>”。</p><p>在这个例子里，选择的模数17很小，使用穷举法从1到17暴力破解也能够计算得到6的离散对数是3。</p><p>但如果我们选择的是一个非常非常大的数，比如说是有1024位的超大素数，那么暴力破解的成本就非常高了，几乎没有什么有效的方法能够快速计算出离散对数，这就是DH算法的数学基础。</p><h2>DH算法</h2><p>知道了离散对数，我们来看DH算法，假设Alice和Bob约定使用DH算法来交换密钥。</p><p>基于离散对数，Alice和Bob需要首先确定模数和底数作为算法的参数，这两个参数是公开的，用P和G来代称，简单起见我们还是用17和5（P=17，G=5）。</p><p>然后Alice和Bob各自选择一个随机整数作为<strong>私钥</strong>（必须在1和P-2之间），严格保密。比如Alice选择a=10，Bob选择b=5。</p><p>有了DH的私钥，Alice和Bob再计算幂作为<strong>公钥</strong>，也就是A = (G ^ a % P) = 9，B = (G ^ b % P) = 14，这里的A和B完全可以公开，因为根据离散对数的原理，从真数反向计算对数a和b是非常困难的。</p><p>交换DH公钥之后，Alice手里有五个数：P=17，G=5，a=10，A=9，B=14，然后执行一个运算：(B ^ a % P)= 8。</p><p>因为离散对数的幂运算有交换律，B ^ a = (G ^ b ) ^ a = (G ^ a) ^ b = A ^ b，所以Bob计算A ^ b % P也会得到同样的结果8，这个就是Alice和Bob之间的共享秘密，可以作为会话密钥使用，也就是TLS里的Pre-Master。</p><p><img src="https://static001.geekbang.org/resource/image/4f/ef/4fd1b613d46334827b53a1f31fa4b3ef.png?wh=3875*2230" alt=""></p><p>那么黑客在这个密钥交换的通信过程中能否实现攻击呢？</p><p>整个通信过程中，Alice和Bob公开了4个信息：P、G、A、B，其中P、G是算法的参数，A和B是公钥，而a、b是各自秘密保管的私钥，无法获取，所以黑客只能从已知的P、G、A、B下手，计算9或14的离散对数。</p><p>由离散对数的性质就可以知道，如果P非常大，那么他很难在短时间里破解出私钥a、b，所以Alice和Bob的通信是安全的（但在本例中数字小，计算难度很低）。</p><p>实验环境的URI“/42-1”演示了这个简单DH密钥交换过程，可以用浏览器直接访问，命令行下也可以用“resty www/lua/42-1.lua”直接运行。</p><h2>DHE算法</h2><p>DH算法有两种实现形式，一种是已经被废弃的DH算法，也叫static DH算法，另一种是现在常用的DHE算法（有时候也叫EDH）。</p><p>static DH算法里有一方的私钥是静态的，通常是服务器方固定，即a不变。而另一方（也就是客户端）随机选择私钥，即b采用随机数。</p><p>于是DH交换密钥时就只有客户端的公钥会变，而服务器公钥不变，在长期通信时就增加了被破解的风险，使得拥有海量计算资源的攻击者获得了足够的时间，最终能够暴力破解出服务器私钥，然后计算得到所有的共享秘密Pre-Master，不具有“前向安全”。</p><p>而DHE算法的关键在于“E”表示的临时性上（ephemeral），每次交换密钥时双方的私钥都是随机选择、临时生成的，用完就扔掉，下次通信不会再使用，相当于“一次一密”。</p><p>所以，即使攻击者破解了某一次的私钥，其他通信过程的私钥仍然是安全的，不会被解密，实现了“前向安全”。</p><h2>ECDHE算法</h2><p>现在如果你理解了DHE，那么理解ECDHE也就不那么困难了。</p><p>ECDHE算法，就是把DHE算法里整数域的离散对数，替换成了椭圆曲线上的离散对数。</p><p><img src="https://static001.geekbang.org/resource/image/b4/ba/b452ceb3cbfc5c644a3053f2054b1aba.jpg?wh=1643*1493" alt=""></p><p>原来DHE算法里的是任意整数，而ECDHE则是把连续的椭圆曲线给“离散化”成整数，用椭圆曲线上的“<strong>倍运算</strong>”替换了DHE里的幂运算。</p><p>在ECDHE里，算法的公开参数是椭圆曲线C、基点G和模数P，私钥是倍数x，公钥是倍点xG，已知倍点xG要想计算出离散对数x是非常困难的。</p><p>在通信时Alice和Bob各自随机选择两个数字a和b作为私钥，计算A=aG、B=bG作为公钥，然后互相交换，用与DHE相同的算法，计算得到aB=abG=Ab，就是共享秘密Pre-Master。</p><p>因为椭圆曲线离散对数的计算难度比普通的离散对数更大，所以ECDHE的安全性比DHE还要高，更能够抵御黑客的攻击。</p><h2>思考题</h2><p>最后留一个思考题吧：为什么DH算法只能用于密钥交换，不能用于数字签名，如果你理解了DH算法的原理应该不难回答出来。</p><p><img src="https://static001.geekbang.org/resource/image/07/af/0773f7b9a098a64cdbe1bf2a666f87af.png?wh=1769*3085" alt=""></p><p></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/a5/98/a65ff31a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>djfhchdh</span>
  </div>
  <div class="_2_QraFYR_0">DH算法只能用于密钥的交换，没有原文、摘要这些参数，无法生成数字签名。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: good。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-25 11:24:13</div>
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
  <div class="_2_QraFYR_0">回答一下思考题：我觉得原因是,根据DH算法的原理，只能算出一个新的值出来用于交换密钥，而数字签名是需要解密数字证书得到数字签名，从而判断数字证书是否真实有效。DH是基于现有数据算出一个新值，公钥私钥算出的结果并不相同，RSA是对数据进行加解密。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 有部分不太准确。<br><br>数字签名与证书没有直接关系。数字签名是对原文摘要的私钥加密，用对应的公钥解密后可以比对摘要，验证确实是私钥持有者做的加密，也就是签名。<br><br>证书是为了保证公钥不被伪造和有效性。<br><br>DH算法里没有原文、摘要这些参与者，所以无法生成签名。也就是说，给出一份文件或者摘要，dh算法无法对它进行任何操作。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-13 14:11:46</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/8f/8b/9080f1fa.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>猫头鹰波波</span>
  </div>
  <div class="_2_QraFYR_0">老师，为什么ECDHE更难破解么，是因为离散的点选取更具备随机性吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这是由椭圆曲线的特性决定的，具体的数学理论我也不是很了解，无法解释的更细。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-08 22:21:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/8b/fa/90126961.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>fxs007</span>
  </div>
  <div class="_2_QraFYR_0">刚才看了下RSA验证签名的过程(https:&#47;&#47;crypto.stackexchange.com&#47;questions&#47;12768&#47;why-hash-the-message-before-signing-it-with-rsa)，我觉得DH算法本身是可以用来验证数字签名。比如双方已经完成了DH秘钥交换过程，<br>签名方发送 text + DH-enc(sha256(text))，其中DH-enc(sha256(text))是对text进行hash算法然后DH加密<br>验证方 用DH-dec解密签名，然后和sha256(text)比较，相等就说明验证通过<br>只是DH一般用在双方确定身份以前，验证没有身份的签名并没有什么意义。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 嗯，如果用变通的方法也是可以做到的，但意义不大，属于“曲线救国”。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-08 17:44:36</div>
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
  <div class="_2_QraFYR_0">老师你好，有个疑问。握手过程中的第三个参数，pre-master为何要用用那么复杂的算法去避免破解呢？<br><br>我的理解是，第三个随机数是通过服务端的公钥加密后传输的，传递到服务端后，用服务端的私钥才能解密出来这个随机数。<br><br>黑客没有服务端的私钥，完全不可能破解pre-master的啊，为何要那么复杂的加密算法去生成这个pre-master？？？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 如果用rsa算法，公钥加密pre-master，当然也是可以的，但这个不具有前向安全。国家级别的计算能力是有可能算出私钥的，这就会导致公钥加密的所有pre-master被解密，从而所有历史消息都被破解。可以参考安全篇对前向安全的解释。<br><br>而dhe和ecdhe不仅难以破解，而且密钥都是随机生成的，所以即使破解了也不影响其他消息的安全。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-23 17:13:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2d/92/7a/0c2317ab.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>江湖骗子</span>
  </div>
  <div class="_2_QraFYR_0">ECDHE握手中有4个数，ClientRandom，ServerRandom，ClientParam，ServerParam，分别对应DH算法中的P，G，A，B，请问老师我的理解对吗？但是DH中P和G不是要求为质数吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: ECDHE和DHE虽然原理相似，但底下的数学基础是不同的，是椭圆曲线而不是离散对数，所以不能和DH算法里的PGAB对应。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-09 18:43:50</div>
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
  <div class="_2_QraFYR_0">学习了，数学就是计算机的力量源泉。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: nice</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-09 16:30:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/e7/20/70a95f94.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>潮汐</span>
  </div>
  <div class="_2_QraFYR_0">老师好，二刷完成了，对课程内容有更深的学习！<br>这里的思考题，我的看法是，从非对称加密算法来说DH是可以对消息做签名的，只是在连接密钥交换阶段没有这个必要，另外DH算法的公私钥是一次一密，也是不符合数字签名中解签的公钥从证书获取保持不变的场景？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: DH算法的过程决定了它只能做密钥交换，签名要用DSA算法。<br><br>EDH算法才是一次一密，标准的DH算法公私钥是静态的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-02-06 09:30:01</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83errj9iaMOHaUFf2XkdkMFU7kNnOiarW6bU8yggOzkJj4ncoqHXiaFwc8nosCdeSvfdeMfBpoVibO724ow/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_91cf3b</span>
  </div>
  <div class="_2_QraFYR_0">老师，PFS报文能解密码？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: PFS是什么？任何加密算法，理论上没有密钥都是不能解密的，但随着算力的提升，有的算法已经可以被破解了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-12 19:15:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>GeekCoder</span>
  </div>
  <div class="_2_QraFYR_0">也就是说DH算法中，也是有私钥的(私密部分)，而外部并不知道私钥。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: DH是非对称算法，当然会有公钥和私钥的区别。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-05-29 20:15:50</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTIuTjCibv0afd7SSdLicfNk0f7KO5ga9VMleD1hc2DtQfianK20ht06SekClKV7M8UXLRHqQLm9hJ3ow/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Jasmine</span>
  </div>
  <div class="_2_QraFYR_0">老师，等式B ^ a = (G ^ b ) ^ a = (G ^ a) ^ b = A ^ b左右两边就算忽略mod17值也不相等啊，为什么说经过运算都等于8呢？实际应用计算的底数超级大，给定了算法，数字运算的结果还是固定的呀，哪怕差0.00001那pre-master也不相同啊。困惑ing</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个是离散指数、对数在整数域的运算，不是普通的实数运算，可以照着正文里的例子再算一下。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-09 17:58:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/25/5e/40/dee7906c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>张欣</span>
  </div>
  <div class="_2_QraFYR_0">老师，不知道我理解的对不对：<br>根据文中以及之前tls文章的讲解，DH算法的主要目的就生成不可逆操作的公钥和私钥，然后再次执行算法生成两端相同的pre-master。这里面生成的公钥和私钥是可以拿来再次对文件摘要进行签名和验证，但是DH算法本身并没有这个作用。假设参数可以是文件摘要，就算算法能够算出，之后拿到证书的人也破解不开，根本没有意义。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 理解有点偏差。<br><br>DH算法主要是用于密钥交换，生成的pre-master用来加密会话，一般不做签名验签。<br><br>签名算法常用的是RSA和ECDSA。<br><br>建议再回顾一下之前的课程，有不明白的或者我没说清楚的这问。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-13 22:41:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/37/2b/b32f1d66.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Ball</span>
  </div>
  <div class="_2_QraFYR_0">原理完全没看懂，得花点时间消化一下了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 先把dh理解了，然后dhe和ecdhe就好懂了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-06 21:35:57</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/32/75/abbc6810.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>久念</span>
  </div>
  <div class="_2_QraFYR_0">老师好，”国家级别的计算能力是有可能算出私钥的“ -- 如果私钥是可以被算出来的，那 Root CA 对应的私钥也有可能被破解，这样的话 黑客是不是就可以随意的颁发证书</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，但这个的难度非常大。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-15 11:42:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/a5/98/a65ff31a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>djfhchdh</span>
  </div>
  <div class="_2_QraFYR_0">因为DH算法，由公钥反向计算私钥是非常难的。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 回答的不是太对，可参考其他同学的答案。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-03-24 19:49:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/df/43/1aa8708a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>子杨</span>
  </div>
  <div class="_2_QraFYR_0">对证书这块还是有点晕。证书应该只是用来证明站点的身份的吧？使用证书颁发机构的公钥对证书中的 hash 值进行解密，同时对 data 做 hash，如果相等就说明身份可信。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 基本正确。<br><br>首先要理解公钥体系，理解签名。证书是为了安全可信地发布公钥。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-23 17:56:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/42/44/d3d67640.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Hills录</span>
  </div>
  <div class="_2_QraFYR_0">DHE不能用于数字签名，是因为无法使用公钥验证私钥有效性</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 注意DHE和DH还是有区别的，DHE里的公私钥对都是临时产生的，显然无法签名，因为私钥用完就丢弃了，不会被任何人持有。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-02-20 17:35:06</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/e5/ac/6c87d5ee.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>mark</span>
  </div>
  <div class="_2_QraFYR_0">DH算法是选择一个数字作为私钥，hash好像不可逆，如果类似hash这样的算法，生成一个整数，是不是就可以用DH算法加密文本了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: DH算法只能用于密钥交换，这种思路有点接近DSA算法。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-21 17:19:22</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83erbY9UsqHZhhVoI69yXNibBBg0TRdUVsKLMg2UZ1R3NJxXdMicqceI5yhdKZ5Ad6CJYO0XpFHlJzIYQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>饭团</span>
  </div>
  <div class="_2_QraFYR_0">老师问题是不是是因为DH算法是动态的！即2个参数是相辅相成的，而数字签名中，公钥是不变的！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不是，可以再看一下dh算法的过程，它不能对一段文本的摘要做加密，无法生成签名，只能双方交换得到共享秘密。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-11 21:54:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83erbY9UsqHZhhVoI69yXNibBBg0TRdUVsKLMg2UZ1R3NJxXdMicqceI5yhdKZ5Ad6CJYO0XpFHlJzIYQ/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>饭团</span>
  </div>
  <div class="_2_QraFYR_0">真棒老师，估计好多同学都不知道您更新了！谢谢您了！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是啊，好像缺少这个更新通知的功能。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-11 21:43:47</div>
  </div>
</div>
</div>
</li>
</ul>