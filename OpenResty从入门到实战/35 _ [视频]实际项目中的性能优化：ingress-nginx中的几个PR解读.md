<p><video poster="https://static001.geekbang.org/resource/image/53/80/536c067253bc7d68cfbb54f762484980.jpg" preload="none" controls=""><source src="https://media001.geekbang.org/customerTrans/fe4a99b62946f2c31c2095c167b26f9c/1a97c39b-16ce823e7a2-0000-0000-01d-dbacd.mp4" type="video/mp4"><source src="https://media001.geekbang.org/402947d480674f578b44c42e194ef714/8e5079a430654fe4a9901fad3e5a9a3a-72367b8ed74bf04d395686d3b65b78d1-sd.m3u8" type="application/x-mpegURL"><source src="https://media001.geekbang.org/402947d480674f578b44c42e194ef714/8e5079a430654fe4a9901fad3e5a9a3a-71ddb15e8eaa40b4179db24e849427d2-hd.m3u8" type="application/x-mpegURL"></video></p><p>你好，我是温铭。</p><p>今天的内容，我同样会以视频的形式来讲解。老规矩，在你进行视频学习之前，先问你这么几个问题：</p><ul>
<li>如何在开源项目中找到可能存在的性能问题？</li>
<li>在 Github 上，如何与其他开发者正确地交流？</li>
</ul><p>这几个问题，也是今天视频课要解决的核心内容，希望你可以先自己思考一下，并带着问题来学习今天的视频内容。</p><p>同时，我会给出相应的文字介绍，方便你在听完视频内容后，及时总结与复习。下面是今天这节课的文字介绍部分。</p><h2>今日核心</h2><p><a href="https://github.com/kubernetes/ingress-nginx">ingress-nginx</a> 是 k8s 官方的一个项目，主要使用Go、 Nginx 和 lua-nginx-module 来处理入口流量。</p><p>在今天的视频中，我会为你清楚介绍，如何运用我们刚刚学习的性能优化方面的知识，来发现开源项目的性能问题。要知道，在我们给开源项目贡献 PR 时，跑通测试案例集以及与项目维护者积极沟通，都是非常重要的。</p><p>下面是 ingress-nginx 中，和 OpenResty 性能相关的两个 PR：</p><ul>
<li><a href="https://github.com/kubernetes/ingress-nginx/pull/3673">https://github.com/kubernetes/ingress-nginx/pull/3673</a></li>
<li><a href="https://github.com/kubernetes/ingress-nginx/pull/3674">https://github.com/kubernetes/ingress-nginx/pull/3674</a></li>
</ul><!-- [[[read_end]]] --><p>从中你也可以发现，即使是资深的开发者，对 LuaJIT 相关的优化，可能也并不是很熟悉。一方面是因为，这两个 PR 涉及到的代码，并不会对整体系统造成严重的性能下降；另一个方面，这方面的优化知识，没有人系统地总结过，开发者即使想优化也找不到方向。</p><p>事实上，很多时候，我们站在代码可读性和可维护性的角度来看，可有可无的优化是不必要的，你只要去优化那些被频繁执行的代码片段就可以了，过度优化是万恶之源。</p><p>那么，学完今天这节课后，你是否可以在其他的开源项目中，找到类似的性能优化点呢？</p><h2>课件参考</h2><p>今天的课件已经上传到了我的GitHub上，你可以自己下载学习。</p><p>链接如下：<a href="https://github.com/iresty/geektime-slides">https://github.com/iresty/geektime-slides</a></p><p>如果有不清楚的地方，你可以在留言区提问，另也可以在留言区分享你的学习心得。期待与你的对话，也欢迎你把这篇文章分享给你的同事、朋友，我们一起交流、一起进步。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4d/fd/0aa0e39f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>许童童</span>
  </div>
  <div class="_2_QraFYR_0">感觉ingress-nginx贡献者和我们一样，一脸蒙蔽啊，哈哈。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-08-14 15:42:57</div>
  </div>
</div>
</div>
</li>
</ul>