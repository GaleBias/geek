<audio title="第2季回归 _ 这次我们来“趣谈Linux操作系统”" src="https://static001.geekbang.org/resource/audio/3a/69/3abcd0c589a74d612cdd5ff5c1c18169.mp3" controls="controls"></audio> 
<p>你好，我是你的老朋友刘超。在“趣谈网络协议”结课半年之后，我又给你带来了一个新的基础课程，“趣谈Linux操作系统”。</p><p>在咱们“趣谈网络协议”的留言里，我和同学们进行了很多互动，同时，我也和其他做基础知识专栏的作者有了不少交流，我发现，无论是从个人的职业发展角度，还是从公司招聘候选人的角度来看，扎实的基础知识是很多人的诉求。这让我更加坚信，我应该在“趣谈基础知识”这条道路上走下去。</p><p><strong>在设计“趣谈Linux操作系统”专栏的时候，我仍然秉承“趣谈”和“故事化”的方式，将枯燥的基础知识结合某个场景，给你生动、具象地讲述出来，帮你加深理解、巩固记忆、夯实基础。</strong></p><p>在我看来，操作系统在计算机中承担着“大管家”的角色，这个“大管家”就好比一家公司的老板，我们的目标就是把这家公司做上市，具体的过程，我用了一张图来表示：</p><p><img src="https://static001.geekbang.org/resource/image/7d/a5/7d7b2f705d4877bb331b4ea3ff3450a5.jpg?wh=2061*1303" alt=""></p><p>Linux操作系统中的概念非常多，数据结构也很多，流程也复杂，一般人在学习的过程中很容易迷路。我希望能够将这些复杂的概念、数据结构、流程通俗地讲解出来，争取每篇文章都用一张图串起这篇的知识点。</p><p>最终，整个专栏下来，你如果能把这些图都掌握了，你的知识就会形成体系和连接。在此基础上再进行深入学习，就会如鱼得水、易如反掌。</p><!-- [[[read_end]]] --><p>一段新的征途即将开始，期待与你继续同行。为了感谢老同学，我为你送上一张<span class="orange">10元专属优惠券</span>，可以与限时优惠同享，优惠券有效期仅<span class="orange">5天</span>，建议你抓紧使用。点击下方图片，可试读专栏最新文章。</p><p>我们新专栏见！</p><p><a href="https://time.geekbang.org/column/intro/164?utm_term=zeusOLMNR&amp;utm_source=app&amp;utm_medium=geektime&amp;utm_campaign=164-presell&amp;utm_content=qutanwangluoxieyi"><img src="https://static001.geekbang.org/resource/image/57/44/57f047be7ebb1f4aba7e8064e1c11544.jpg?wh=1342*638" alt=""></a></p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/53/a8/abc96f70.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>return</span>
  </div>
  <div class="_2_QraFYR_0">必须捧场， 铁杆。虽然网络协议还没学完， 但是 http原理搞懂了😆。 最爱的刘超老师和王铮老师</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢喜爱</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-25 18:15:17</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/8d/74/3bb89151.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>jason0606</span>
  </div>
  <div class="_2_QraFYR_0">超哥, 还想你出一下关于容器方面的课程,  看了你的趣谈网络, 趣谈linux, 继续出一个容器的专题, 感觉就可以这三个内容串起来了呀</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-10-18 12:03:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/c4/e4/81ee2d8f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Wisdom</span>
  </div>
  <div class="_2_QraFYR_0">期待</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-25 18:10:46</div>
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
  <div class="_2_QraFYR_0">那我必须去捧场了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-25 17:30:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/4b/2f/186918b4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>C J J</span>
  </div>
  <div class="_2_QraFYR_0">已购。请问老师你这思维导图是用哪个软件画的？看起来相当舒服。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">编辑回复: xmind呀</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-26 08:46:29</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/59/25/0efc8b38.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Dream</span>
  </div>
  <div class="_2_QraFYR_0">已购，谢谢老师分享</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-25 18:58:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/12/cc/6a08cf26.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>面朝大海春暖花开</span>
  </div>
  <div class="_2_QraFYR_0">必须支持，老学员！！</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-25 18:21:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/0f/ab/9748f40b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>微秒</span>
  </div>
  <div class="_2_QraFYR_0">肯定得听，操作系统闻名已久啊</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-25 18:13:38</div>
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
  <div class="_2_QraFYR_0">老师，网络体系很大，有什么好的书籍或者路线推荐不？</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-18 10:05:11</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1f/15/13/23a49cc4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Jose</span>
  </div>
  <div class="_2_QraFYR_0">立马入手</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-05-02 12:37:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/d6/0e/711721c3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ISA007</span>
  </div>
  <div class="_2_QraFYR_0">已经买了，读完这本教程，开始攻克Linux.</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-02-23 10:59:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/04/18/4b02510f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>明天更美好</span>
  </div>
  <div class="_2_QraFYR_0">刘老师，请教个问题，全部内网部署，还要做双中心，您这又现成的方案之类的吗</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 同城双活方案很成熟，但是由于每个客户环境差异，可能需要定制化</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-09-02 22:35:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/c0/ce/fc41ad5e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>陳先森</span>
  </div>
  <div class="_2_QraFYR_0">已购，支持一下老师。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-07-31 17:24:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/65/03/973b24ec.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>谢晋</span>
  </div>
  <div class="_2_QraFYR_0">超级棒的专题，非常细致，刘老师很用心！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-05-12 23:25:32</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/0e/91/ef0778d7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>B L O U</span>
  </div>
  <div class="_2_QraFYR_0">还想请教一下，老师是否能加个餐：<br><br>MIXIN是一个无限吞吐的链接所有现存区块链的移动端区块链网络，采用了DAG技术，链接所有现存的区块链网络，是一个跨链网络协议<br><br>能否解释一下垮链网络协议？感激不尽</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 区块链没研究过</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-14 10:02:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/d2/0e/26bf35a4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>平安喜乐</span>
  </div>
  <div class="_2_QraFYR_0">打卡</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-04-03 14:36:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/ae/a2/e4b59443.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>liao89</span>
  </div>
  <div class="_2_QraFYR_0">已经购买希望能跟上进度。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-25 23:11:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/69/4d/81c44f45.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>拉欧</span>
  </div>
  <div class="_2_QraFYR_0">第一期就感觉超值，果断支持</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-25 22:14:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/cd/0e/3b42e6d2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Happy</span>
  </div>
  <div class="_2_QraFYR_0">每节课打卡，希望这次能坚持活下去</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-25 21:51:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/d5/8a/7050236a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>东征</span>
  </div>
  <div class="_2_QraFYR_0">厉害了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2019-03-25 18:10:45</div>
  </div>
</div>
</div>
</li>
</ul>