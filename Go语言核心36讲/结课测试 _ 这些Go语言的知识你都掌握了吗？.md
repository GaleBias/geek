<p>你好，我是郝林。</p><p>到这里，《Go语言核心36讲》这门课程已经全部结束了。我给你准备了一个结课小测试，来帮助你检验自己的学习效果。</p><p>这套测试题共有 20 道题目，包括7道单选题和13道多选题，满分 100 分，系统自动评分。</p><p>还等什么，点击下面按钮开始测试吧！</p><p><a href="http://time.geekbang.org/quiz/intro?act_id=125&exam_id=268"><img src="https://static001.geekbang.org/resource/image/28/a4/28d1be62669b4f3cc01c36466bf811a4.png?wh=1142*201" alt=""></a></p><!-- [[[read_end]]] -->
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/64/05/6989dce6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>我来也</span>
  </div>
  <div class="_2_QraFYR_0">看来得再把专栏好好过一遍了😭</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: ;-)</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-05-12 14:19:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/dd/09/feca820a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>helloworld</span>
  </div>
  <div class="_2_QraFYR_0">第11题:<br>针对通道的操作在什么时候会有阻塞的情况发生？选项D: 针对值为 nil 的通道的发送操作和接收操作<br><br>针对值为 nil 的通道的发送操作和接收操作应该是会发生死锁, 感觉这种情况说成是阻塞不太妥当吧</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 所谓“死锁”是指。程序中的所有并发例程全部阻塞。在这种情况下，Go是会报错的。程序中只有一部分例程阻塞，那不就死锁。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-01-27 14:53:53</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_36b02c</span>
  </div>
  <div class="_2_QraFYR_0">课程比较基地 许多工作中用到的却沒讲 比如 工作中用module ，插件 cgo 这些</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 哈哈，module那时候还没稳定呢。我觉得现在都没完全稳定，讲完就过时。插件、cgo这些，面试的时候有人问你吗？这种哪里都搜得到的知识，我肯定不会讲。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-16 06:29:39</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/bb/27/689f6974.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>mgw</span>
  </div>
  <div class="_2_QraFYR_0">go 如何避免协程泄露</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 原则很简单：及时断开无用对象与有用对象之间的联系。<br><br>比如，把无用的共享变量设置为nil、不要共用大切片、及时删除大字典中的无用键值对，等等。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-03-26 17:35:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/df/1e/cea897e8.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>传说中的成大大</span>
  </div>
  <div class="_2_QraFYR_0">第九题A答案说append总会返回一个新切片？ 如果没有发生扩容 应该不会返回新切片吧？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 总是会的，切片的结构体只是一个外壳而已。上周刚有人问，你可以翻一下留言。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-08-28 14:01:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/5b/66/ad35bc68.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>党</span>
  </div>
  <div class="_2_QraFYR_0">老师 缺少点反射的课程 缺少点泛型内容 不知老师后期会不会添加这部分内容 虽然不常用 但绝对是拔高的课程 另外老师讲东西总是给人豁然开朗的感觉 别的教程看了半天都晕乎乎的</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对你有帮助就好。我后面看看是不是有机会再更新。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-04 09:01:31</div>
  </div>
</div>
</div>
</li>
</ul>