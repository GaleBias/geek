<audio title="结束语｜聊聊我的SRE落地心路历程" src="https://static001.geekbang.org/resource/audio/32/e0/3215ba424ab84611039ca716f0bedde0.mp3" controls="controls"></audio> 
<p>你好，我是赵成，不知不觉我们已经来到了结束语，非常感谢你的一路陪伴。</p><p>学完咱们的专栏，我想对于SRE到底是怎么一回事儿这个问题，你应该有一个大致的了解了。就像我们在开篇词中提到的，<strong>SRE真的没有那么神秘</strong>，你平时在做的很多事情本身就属于SRE的范畴，学到这里，你应该对此深有体会了。</p><p>其实这个感受我也是在不断实践的过程中总结出来的。刚接触这个概念的时候立马被它吸引，但同时也觉得这东西有点儿高大上，自己有种心有余而力不足的感觉。幸好和团队一起，就是一点一点死磕，解决一个又一个具体的问题，然后因为一直有这样一个大的框架和目标在那里，最后慢慢发现，这个框架居然已经落地得差不多了。如果总结下我自己实践SRE的心路历程，我觉得王阳明《传习录》里的“<strong>知者行之始，行者知之成</strong>”就特别恰当、准确。</p><p>你是不是在想，这不就是知行合一嘛，也没啥特殊啊！嗯，确实是，听起来、说起来都挺简单的，但是很多时候我们想要做到还真不容易。</p><p>其实，在学习这个课程的过程里，我们也需要知行合一，从知出发，到行完成一个闭环，然后积累新的知，把这个知行的循环一直继续下去。</p><p>这么说，有点抽象，这里我特别举咱专栏里一位同学的例子。这位同学名字叫胡凯，他一边学习课程，一边和我探讨一些SRE问题。每次提问，他总是可以带着具体场景和具体问题，非常有针对性，而且针对不同的场景，他又会有自己的一些见解和解决方案，然后在与我讨论的过程中，不断迭代优化他的思路和方案，特别是在SLO设定这一块，因为很多监控指标都是现成的，他马上就根据我们课程里给出的VALET方法，整理出了一个新的表格，这种从更多SLO维度分析稳定性的方法，一下子就解答了他之前一直以单一维度判断稳定性的很多疑惑和问题。</p><!-- [[[read_end]]] --><p>像胡凯这样的同学，我们专栏里还有很多，大家都提出了非常好的问题，也分享了自己的思考和总结。这个我们一起交流探讨的过程，对于我来讲也是一次难得的学习机会，我想这就是“教学相长”的意义吧。</p><p>那么，接着这个话题，我再唠叨几句我的期待吧。这个课程基础篇的几讲是我花费心思最大的内容，因为我想从基础上就讲明白SRE的一些概念和理论。说实话，这部分内容也是需要你花费很大的精力和实践去消化的。如果你之前有过一些实践，再结合我们的课程去看的时候，你会发现理解起来就会轻松很多，也会有更多的收获；如果你现在还没有那么多的实践，这些内容你理解起来还没那么直观，那接下来就要抓住工作中的具体场景和问题，先去实践下，再回过头来看这几讲，到时候你肯定会有不一样的理解，我也会在这里，继续等你提出更好的问题来。</p><p>所以你看，对于我们从书本、课程中学习到的知识，要想把它们真正地转化为自己的能力，唯一的方法就是实践、思考、优化实践，并且不断重复这个过程。</p><p>对于我们要学习的SRE来说，也是这样。我认为很多人之所以没能好好落地SRE，一个最大的障碍不是技术难度、甚至不是组织架构和文化等问题，而是大家先把自己局限在了概念上，很多人深深地沉浸在SRE到底是什么，它跟现在非常流行的DevOps、AIOps、混沌工程以及各类中台的概念到底是怎样的一个关系？我们该怎么选？……<strong>纠结在这样那样的问题中，结果就是在问题漩涡中停滞不前，迈不出第一步，那就永远都走不前去</strong>。</p><p>这时候应该怎么做呢？我的建议就是，从你遇到的实际问题出发，从你所在的实际场景出发，解决问题，满足场景需求，先做起来再说，然后参考优秀的实践案例和分享，再做优化和调整。</p><p>其实，在蘑菇街实践SRE的时候，我们也不是天天把SRE挂在嘴边，也不是动不动就提DevOps、AIOps这些名词的，相反，我们提到的更多是面对某个场景，我们的容量评估应该怎么做？细化到每个应用、每个接口上限流阈值是多少，降级和熔断的具体判断策略是怎么样的？发生故障时，我们Step by Step的响应过程应该是怎么样的？需要哪些人参与？大家应该怎么协作？对于监控，怎么才能更准确？需要用到什么具体算法，参数应该怎么设定？……</p><p>你看，这些问题基本都是针对具体问题和具体场景的，而且针对这些问题和场景业界都已经有非常多的经验和案例供我们参考了，也就是我们大有可为的地方太多了。你可以设想一下，如果这些问题都能够解决得很好，我们是不是就已经达到了SRE的标准了呢？我们是不是就已经是SRE了呢？</p><p>我想答案是肯定的。</p><p>好了，到这里，我们专栏的内容就全部结束了。Google给我们呈现的SRE是理论性的、指导性的，业内在这方面的实践还是相对稀缺。想要更好地落地SRE，那就需要我们每一个团队和每一个热爱SRE的同行一起实践、一起总结、一起分享。</p><p>那还等什么，SRE并不神秘，让我们一起探索出一条适合我们自己的SRE实践之路。</p><p><a href="https://jinshuju.net/f/LpoFKG"><img src="https://static001.geekbang.org/resource/image/0f/77/0ff24b3805b9494193071ab274498777.jpg?wh=1142*801" alt=""></a></p>
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
<div class="_2sjJGcOH_0"><img src="http://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83eoYIAOJ64s9BIHsCkjia1eGyfTEBvIibfOPbwzoib7JiaP0zWdlssybicBPj0VYuT5rufa5TMjxanvOSibw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>艾比利夫</span>
  </div>
  <div class="_2_QraFYR_0">谢谢老师一个月的分析，一章不差的看完了，收获颇深。<br><br>我和大家不太一样，我在一个小公司就职。所以在学习各种大厂体系的过程中，总有一个困惑，就是体系很牛，但我没法用，因为小公司无论人力资源、技术能力、硬件能力等都太小了，即使理论上学了，但根本无法耗时耗力搭建这么一套东西。<br><br>但这次我学习咱们的SRE体会就不太一样，我先了解了MTBF、MTTR(更细的说是MTTR里的四个阶段)，然后对照我们公司的自身的情况对照着表格看，看看是哪个环节是目前的薄弱环节。这样即使我无法向您一样搭建整个体系，我也能针对性的解决最薄弱的环节。<br><br>但老师您在课程中也有说：SRE是一套体系，多部门合作出来的，并不是某一个点或某一个技术，那请问老师，对于我们这些中小型公司，资源有限，那怎么做才能让系统全方位的稳定起来呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 可以针对现在的问题做个排序，从最消耗你精力，最让你难受的的问题入手。<br><br>大处着眼，小处入手。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-10 14:48:13</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/c3/48/3a739da6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>天草二十六</span>
  </div>
  <div class="_2_QraFYR_0">大清早看到更新了，第一时间转发了这段到朋友圈：其实，在蘑菇街实践 SRE 的时候，我们也不是天天把 SRE 挂在嘴边，也不是动不动就提 DevOps、AIOps 这些名词的，相反，我们提到的更多是面对某个场景，我们的容量评估应该怎么做？细化到每个应用、每个接口上限流阈值是多少，降级和熔断的具体判断策略是怎么样的？发生故障时，我们 Step by Step 的响应过程应该是怎么样的？需要哪些人参与？大家应该怎么协作？对于监控，怎么才能更准确？需要用到什么具体算法，参数应该怎么设定？……<br><br>我想，这才是我要去实践的，不是跟领导或同事灌输思想</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 对，不要被Buzzword给迷惑了。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-10 05:27:19</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/ac/bb/69281ec2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>李杨</span>
  </div>
  <div class="_2_QraFYR_0">谢谢赵老师分享！感觉 DevOps 和 SRE 相辅相成，没有 DevOps 的CI、CD、监控就没有SRE的SLI, SLO。返过来，没有SRE的指标，DevOps也不知道往哪个方向发展。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 很精辟的理解。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-25 09:41:35</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/34/df/64e3d533.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>leslie</span>
  </div>
  <div class="_2_QraFYR_0">SER&#47;DevOps与另外一个现在提出很多的概念“中台”类似，落地的过程其实就是循序渐进中梳理出自己的东西；然后不断反复。<br>概念是浮在面上的东西：如何合理去体现在实践中去摸索相关实践修正这其实是大家需要探索的一条路。概念无处不在如何合理组合然后落地这个是一条漫长的路。<br>谢谢老师一路的分享，希望将来还有机会交流学习；愿老师未来的路越来越好。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 实践的过程中，有问题可以继续给我留言提问。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-10 08:37:21</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/9e/d3/abb7bfe3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>wholly</span>
  </div>
  <div class="_2_QraFYR_0">跟着老师把课程学完了，谢谢老师，老师辛苦了！就像老师说的，学习课程还只是一个理论的开始，后面更关键的是结合理论不断实践不断思考，把实际遇到的场景和问题一个个解决闭环，才能真正成为一个优秀的SRE。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 一起努力，也希望看到大家更多关于SRE实践方面的分享。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-10 08:13:47</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/87/9a/40d73657.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Mander</span>
  </div>
  <div class="_2_QraFYR_0">感谢老师分享</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 也感谢你的聆听和阅读，一起进步。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-10 10:44:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1a/05/fc/bceb3f2b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>开心哥</span>
  </div>
  <div class="_2_QraFYR_0">知易行难啊。一点点来吧</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-02 20:39:25</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/3a/39/72d81605.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>大尾巴老猫</span>
  </div>
  <div class="_2_QraFYR_0">这么快就结束语了？还意犹未尽...</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 还想听什么可以留言给我哈</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-10 11:46:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/dd/3c/a595eb2a.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>台风骆骆</span>
  </div>
  <div class="_2_QraFYR_0">知行合一，从具体场景，业务出发。把学到的知识真正融入到业务中，然后反哺知识，形成闭环</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 一起努力。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-10 11:03:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/46/be/d3040f9e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>小广</span>
  </div>
  <div class="_2_QraFYR_0">感谢赵老师提供了那么优质的课程，特别是课程前面部分的内容，基础性知识讲解得非常细致，这部分内容可移植性和可适配性非常好，对指导改变现实中的问题帮助非常大😄后面我会尝试使用这些指导思想定义我司的稳定性指标，希望能有个好的开始(^_^)v祝老师🐯年顺利</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 加油，共同进步</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-24 09:08:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/be/16/9ddeb937.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Tyrone.Zhao</span>
  </div>
  <div class="_2_QraFYR_0">限流阈（yu）值</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-04-21 09:50:52</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/34/df/64e3d533.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>leslie</span>
  </div>
  <div class="_2_QraFYR_0">用2天的时间过了一遍笔记做了一遍，结合过去的1年的经历；又看到不一样的东西。PM的过程中其实就包含了PE的东西，只不过多了一块实践而已-一切皆无界。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-11-22 09:46:34</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/b0/8a/3ecf6853.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Browser</span>
  </div>
  <div class="_2_QraFYR_0">很有收获，目前公司内部也老是遇到各种问题，每次都充当救火队员，根据老师的这份资料，决定实践一下</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 加油，也可以多给我们分享一下你的实践</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-10-24 00:10:43</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>三毛</span>
  </div>
  <div class="_2_QraFYR_0">成哥的课反复看了好几遍，每次都有新的收获，谢谢成哥！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢，希望对你有所帮助和提升。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-09-06 22:05:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/83/14/bcc58354.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>li3huo</span>
  </div>
  <div class="_2_QraFYR_0">之前做2C业务时的线上故障大致能分成两类：1种是由软件质量缺陷导致；另外1种就是上章这种大促场景由于扩容不足、预案不充分时错误应对忙中出错。虽然也有事后复盘，但之前的总结没有这么系统，赵老师讲得真的很好！</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 谢谢你的认同，多交流。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2020-04-22 23:29:21</div>
  </div>
</div>
</div>
</li>
</ul>