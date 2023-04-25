<p><video poster="https://static001.geekbang.org/resource/image/2b/04/2b19372f8c88bb89c799382bb4767504.jpg" preload="none" controls=""><source src="https://media001.geekbang.org/customerTrans/fe4a99b62946f2c31c2095c167b26f9c/55ac88a8-16ce81d8277-0000-0000-01d-dbacd.mp4" type="video/mp4"><source src="https://media001.geekbang.org/e63152456c494c38a25ef45b274c9610/f2a99770955f4dff8f68622573c395ca-37963b86cd6511e18c03ec2809815ddb-sd.m3u8" type="application/x-mpegURL"><source src="https://media001.geekbang.org/e63152456c494c38a25ef45b274c9610/f2a99770955f4dff8f68622573c395ca-fbd7fdcd1c8e947dad21fe0f44731b04-hd.m3u8" type="application/x-mpegURL"></video></p><p>你好，我是温铭。</p><p>今天的内容，我同样会以视频的形式来讲解。老规矩，在你进行视频学习之前，我想先问你这么几个问题：</p><ul>
<li>你在使用 OpenResty 的时候，是否注意到有 API 存在安全隐患呢？</li>
<li>在安全和性能之间，如何去平衡它们的关系呢？</li>
</ul><p>这几个问题，也是今天视频课要解决的核心内容，希望你可以先自己思考一下，并带着问题来学习今天的视频内容。</p><p>同时，我会给出相应的文字介绍，方便你在听完视频内容后，及时总结与复习。下面是今天这节课的文字介绍部分。</p><h2>今日核心</h2><p>安全，是一个永恒的话题，不管你是写开发业务代码，还是做底层的架构，都离不开安全方面的考虑。</p><p>CVE-2018-9230 是与 OpenResty 相关的一个安全漏洞，但它并非 OpenResty 自身的安全漏洞。这听起来是不是有些拗口呢？没关系，接下来让我们具体看下，攻击者是如何构造请求的。</p><p>OpenResty 中的 <code>ngx.req.get_uri_args</code>、<code>ngx.req.get_post_args</code> 和 <code>ngx.req.get_headers</code>接口，默认只返回前 100 个参数。如果 WAF 的开发者没有注意到这个细节，就会被参数溢出的方式攻击。攻击者可以填入 100 个无用参数，把 payload 放在第 101 个参数中，借此绕过 WAF 的检测。</p><!-- [[[read_end]]] --><p>那么，应该如何处理这个 CVE 呢？</p><p>显然，OpenResty 的维护者需要考虑到向下兼容、不引入更多安全风险和不影响性能这么几个因素，并要在其中做出一个平衡的选择。</p><p>最终，OpenResty 维护者选择新增一个 err 的返回值来解决这个问题。如果输入参数超过 100 个，err 的提示信息就是 truncated。这样一来，这些 API 的调用者就必须要处理错误信息，自行判断拒绝请求还是放行。</p><p>其实，归根到底，安全是一种平衡。究竟是选择基于规则的黑名单方式，还是选择基于身份的白名单方式，抑或是两种方式兼用，都取决于你的实际业务场景。</p><h2>课件参考</h2><p>今天的课件已经上传到了我的GitHub上，你可以自己下载学习。</p><p>链接如下：<a href="https://github.com/iresty/geektime-slides">https://github.com/iresty/geektime-slides</a></p><p>如果有不清楚的地方，你可以在留言区提问，另也可以在留言区分享你的学习心得。期待与你的对话，也欢迎你把这篇文章分享给你的同事、朋友，我们一起交流、一起进步。</p>
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
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/MlmSR4YXUfrNlZdMv7bv1ic64HaxxVKcVtaxjzhXCvNC4XByICCmYUTprhOESzIV8p59N6DnSJ7HywfvGr5nicgA/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>mz</span>
  </div>
  <div class="_2_QraFYR_0">老师可以推荐几个可用的开源 WAF 防火墙吗？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 推荐 https:&#47;&#47;github.com&#47;starjun&#47;openstar，因为我见过作者，其他的不太熟悉</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-16 08:58:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/da/36/ac0ff6a7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>wusiration</span>
  </div>
  <div class="_2_QraFYR_0">老师，能推荐几个有代表性的CVE问题吗？想研究一下</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这方面我就不是专家了，可以看看freebuf</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-15 21:45:48</div>
  </div>
</div>
</div>
</li>
</ul>