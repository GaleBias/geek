<audio title="50｜思考实例（上）：探险家Amundson是凭什么胜出的？" src="https://static001.geekbang.org/resource/audio/10/88/10f9a5008ce0185fc785b8e8684af988.mp3" controls="controls"></audio> 
<p>你好，我是郭东白。</p><p>我们在之前的课程中曾多次提到<a href="https://en.wikipedia.org/wiki/Roald_Amundsen">Amundson</a>和<a href="https://en.wikipedia.org/wiki/Robert_Falcon_Scott">Scott</a>在南极探险的经历，那么这节课，我们就以此为例，讲讲如何通过软件架构之外的案例来提升你的思考力。</p><p>我先对这个案例的背景与结果做个简单的说明。有两个不同的团队，采用了不同的策略，最终Amundson先于Scott到达南极并安全返回。而Scott团队中向极地冲刺的五人，全部葬身在回程之中。在这个过程中，两个领导者的决策质量，也就是他们的思考力有着巨大的差异，因此也对最终的结果产生了巨大的影响。</p><p>我想你肯定会有疑问，为什么要选择南极探险的案例呢？这似乎跟软件架构没什么太大的关系啊。先别着急，这其中有很多个原因，听我慢慢道来。</p><h2>为什么选择南极探险的案例？</h2><p>首先，南极探险案例的复杂程度不亚于一个架构活动，因此对互联网企业有着非常大的启发性价值。</p><p>南极探险，哪怕在高科技发达的今天，也是一个高风险项目。更何况，在当年缺乏资源、设备、气象、救援等能力的时候，探险无疑就是拿着生命在赌博。但南极探险又是人类开拓边界的一个壮举，所以吸了不少顶级人才和资本参与其中。</p><p>这种高风险高回报的场景，是不是跟我们当下所处的互联网时代的很多商业活动十分类似？答案是肯定的。所以研究南极探险，对我们日常的商业、风险和组织决策有着非常高的指导意义。</p><!-- [[[read_end]]] --><p>其实不光是互联网行业。南极探险的案例在近二十年来吸引了越来越多的人的关注，在国内一些领导力和组织培训的课程中也经常被引用。</p><p>其次，这是一个典型的众说纷纭的案例，因此是研究他人思考逻辑和观点的一个绝佳案例。</p><p>对于同样一件事情，为什么会有完全不同的看法呢？因为评论者的前提假设、价值观和思考路径都有不小的差异，那么研究他们的观点，可以帮助我们提升思考力，尤其是独立判断的能力。</p><p>值得一提的是，南极探险距离现在已经有一百多年了，人们思考事件的角度也发生了不小的变化。在这个视角迁移的过程中，我们也能看到整个西方世界在价值观上的改变，从中获取更深度的思考。</p><p>第三个原因，这个案例背后的资料非常完整，比如两个团队的成员都写有传记，而且还有大量一手资料可以对照。这在人类历史上也是非常罕见的。就像有两家公司针对同一个事件同时做A/B测试，结果是一成一败。难能可贵的是，这两家公司把他们的内部资料做了全盘分享。</p><p>如果从架构文档的角度看，传记本身也有着非常高的学习价值。他们对整个探险过程从多个维度进行了非常详细的描述，包括详细的清单、事件记录、决策过程和当事人的日记摘录等。整个著作几乎没有任何主观上的渲染，对时间、事件、人物、资源、决策过程的描述，几乎到了可以完全还原的地步。</p><p>如果把这两部著作当成复盘文档来读的话，我做了这么多年的互联网，可以说，平时见的复盘文档连它们的十分之一都不到。</p><p>其中<a href="http://www.gutenberg.org/ebooks/14363">Apsley写的</a><a href="http://www.gutenberg.org/ebooks/14363">传记</a>，被《国家地理》评为有史以来<a href="https://www.smithsonianmag.com/smart-news/inside-worst-journey-world-180961589/">最好的100本探险作品</a>。另外一本由Amundson亲笔所著，是非常优秀的文学著作，文本流畅，故事惊心动魄，引人入胜。</p><p>最后一个原因与我的个人爱好有关。我们这个模块讲的是独立思考，所以我需要找一个我非常熟悉，并且从中获益匪浅的实例。我是个户外活动的爱好者，很喜欢这类型的读物，也十分崇拜Amundson。对于他和Scott在南极探险的案例，我曾花了大量时间来研究，无论是各自的传记，还是研究文献和相关资料，学习过好多遍。</p><p>相比很多没有看过第一手资料的人，我想我能够分析得更全面一些，也能比较好地引导你来学习。</p><h2>案例背景</h2><p>我先来简单描述一下Amundson和Scott在1910～1912年南极探险的背景。需要预先说明的是，这些描述还不足以让你感受当时恶劣的环境、匮乏的资源、惊险的过程、频繁的冲突和决策的困难。因此我强烈建议你去读一读原文，而我的描述只能作为一个引子。</p><p>当时已经是人类大航海时代的末尾，在此前的近400年里，探险家们征服了一个又一个目标。但是南极作为地球上最后一个人类尚未到达的目的地，是皇冠上的明珠。而争夺这个目标的最有力竞争者，就是英国的南极探险队队长Scott。他是英国海军上尉，曾率领团队<a href="https://en.wikipedia.org/wiki/Terra_Nova_Expedition">到达南纬82度</a>，发现了南极高原。之后英国人Stockton还做过一次尝试，到达了南纬88度。</p><p>而这一次，是英国在南极探险上的第三次尝试。作为英国官方赞助的活动，资金和设备都十分充足，技术也非常先进，甚至动用了机动雪橇。当然，由于此次活动也兼具科学考察的性质，团队成员人数众多，有65名。经验也非常丰富，其中有７名曾是Stockton团队的成员。</p><p>Amundson是挪威的职业探险家。他的人生经历可以说是<a href="http://www.dioi.org/smp.htm">一个传奇</a>。顺便说一句，他的每一次探险经历都十分惊心动魄，很值得研究。而对于这次探险，他本来不是去南极的。但是很不幸，已经有人宣称先他一步到达了北极。于是Amundson临时决定改道南极。</p><p>而挪威官方担心此举会惹恼英国人，所以拒绝资助他。没办法，Amundson只好通过抵押自己的房子来<a href="https://en.wikipedia.org/wiki/Amundsen%27s_South_Pole_expedition#CITEREFAmundsen">筹措资金</a>。而且，Amundson的团队只有19名成员。甚至为了保密，都没有把自己的行程公开给队员。直到船出发，并且无法折返之后，Amundson 才告诉船员他们此行的真实目的地是南极。</p><p>Amundson之前到过北极圈，他的设备选择是基于<a href="https://archive.org/details/roaldamundsenmyl00amun_0">爱斯基摩人的方法</a>。不先进，但是已经被爱斯基摩人的长期生存证实是可靠的。</p><p>南极的气候条件恶劣。Amundson团队中没有任何南极陆地探险的经验，完全靠阅读英国团队的传记来获取知识。</p><p>Scott因为有南极探险的经验，就选择了与前两次英国探险队相同的路径，而Amundson则选择了一个完全不同的路径。Amundson在出发后就向全世界公开了自己的意图，而Scott的团队甚至在当年冬天造访过Amundson的营地。所以两个团队都知道对方的存在，也铆足了劲儿想赢得这场竞赛。</p><p>在探险的过程中，两个团队或多或少都发生了一些意外，Amundson团队的选择最终被证实更明智一些。在完成这次壮举之后，Amundson发表了<a href="http://www.gutenberg.org/ebooks/14363">传记</a>。Scott团队里的一个核心成员，Apsley Cherry-Garrard，发表了题为<a href="http://www.gutenberg.org/ebooks/3414"><em>The worst journey in the world</em></a>的团队传记。他们的传记在社会引起了广泛传播和讨论，也被历史等领域的专家反复研究分析。</p><p>这里我们先汇总一下后人的评论与看法。</p><h2>观点汇总</h2><p>我们这节课的目的是帮助你提升思考力。就像所有的技能一样，我们并不是从头开始重新推导先哲们的理论，而是先试图通过各种途径学习研究，然后在他们的认知的基础之上开始自己的思考，这样才有可能超越前人。</p><p>而作为一个思考的案例，我们首先要做的是理解不同的人是如何得出不同的观点。他们的论点是什么？论据是什么？为什么同样一个案例，却派生出这么多不同的观点？</p><h2>目标决定成败</h2><p>Amundson在出发之前，就把全部身家压在了此次探险上。他的目标单一且明确，就是到达南极并安全回来，因为他要靠出售传记版权拿回投资。</p><p>对于Scott来说，到达南极是多个目标中的一个。举个例子，其中一个很重要的目标就是科学考察。所以Scott在得知Amundson的营地位置之后，意识到Amundson可能会先到达南极。虽然表达了失望，但并没有对自己的计划做出更改。</p><p>事实上，Scott团队也的确忠于科学考察这个子目标。他们最后一程的突击队员里就有一名科学家，团队也从南极拉回来重达14公斤的矿石标本，其中植物化石证实了南极大陆曾经是有生命繁衍的地方，而不是一片冰原。</p><p>因而有观点认为，Scott自始至终都在做多目标优化。到达南极并回来这个目标，就不是最优的。还有一个细节可以证明这一点。当Scott选择参与冲刺最后一程的成员时，并没有把队员的体力当作选择标准。</p><p>而Amundson虽然在出发时欺骗了团队，但因为到达南极是个伟大的目标，就感染了所有船员。正是在这个单一目标的驱动下，很多选择变得简单又正确。而诸多正确的选择，带领团队最后走向了成功。</p><h3>细节决定成败</h3><p>Amundson 的自传里有一段话：</p><blockquote>
<p>Victory awaits him who has everything in order—luck, people call it. Defeat is certain for him who has neglected to take the necessary precautions in time; this is called bad luck.</p>
</blockquote><p>这段话的核心意思是“胜利只属于那些有准备的人”，也被很多人看作Amundson团队成功的根因。</p><p>Amundson<strong>在准备细节<strong><strong>上做</strong></strong>到了极致</strong>。举个例子，Amundson为了防止队员迷路，便把罐头盒子漆成黑色，每半英里放一个盒子。并在与路线垂直的方向上立了20根杆子，覆盖了10英里的路程。他这么做，就是为了标记储藏点。</p><p>有趣的是，他使用的标记方式是数据结构里的链表结构。简单来说，只要找到一根杆子，就能找到储藏点。哪怕是在大风雪天气偏离了路线，也能确保队员找到食物。</p><p>而Scott团队立的杆子要低很多，而且仅仅立在了储藏点。事实证明，他们确实在寻找储藏点上花费了不少时间。</p><p>还有一个细节。Amundson团队准备了4个指南针，而Scott团队只有一个，后来还用不了了。</p><p>这样的细节还有很多。综合下来，这种观点认为错误积少成多，压垮Scott团队的最后一根稻草不一定是哪一个，但肯定是在细节上。</p><h3>领导力决定成败</h3><p>Scott是一名军官，当时英国军队中的指挥权比较绝对，军官的命令都需要严格执行。相比之下，Amundson的团队是雇佣关系，没有绝对的领导权。只有他的弟弟自始自终都是团队中的核心成员，并且陪他一起到了南极。</p><p>这种观点认为，Amundson没有绝对的领导权，但却有绝对的领导力，而他的领导力就源自一个又一个优秀的决策。</p><p>Amundson有很多决策后来都被证实是更加合理的。举个例子，Amundson选择用狗来拉雪橇。最终这些狗不但胜任，甚至超出了团队的期望。对比之下，Scott选择用机动雪橇和西伯利亚矮马。然而雪橇技术太新了，很快就坏了。矮马呢，在去往南极的路上就被冻死了，没有帮上实质性的忙，最终还要靠人来拉雪橇。</p><p>除此之外，Amundson也显示了非常强的<strong>决策应变能力</strong>。他的团队在第一年的冬天里做了三次运送食物的尝试，气温最低时达到了零下40摄氏度。在这个过程中，他发现狗在气温太低时没有足够的力气拉雪橇，所以就临时做出调整，减少向南极冲刺的人数，增加狗的数量。</p><p>这样一来，每条狗的相对负载就减少了。等到了南极高原可以把狗杀掉，作为队员和其他狗的食物，在最大程度上降低挨饿的风险。对比之下，Scott在得知Amundson的营地更有优势的情况下，也没有对自己的计划做出任何变更。</p><p>这种观点的持有者认为，一个人的领导力，尤其是决策应变能力，会提升整个团队的决策质量和执行质量，最终带来结果上的显著差异。</p><h3>人才是成败的关键</h3><p>前面提到Amundson团队是多国雇佣军。在这些人里，除了他的弟弟外，还有很多人都是之前跟Amundson一起探险过的朋友，因此也建立了非常深厚的信任关系。</p><p>Amundson的传奇经历里就有营救朋友的故事，他最后也是在营救朋友的过程中遇难的。所以Amundson团队的人其实更团结，更适合极地探险。最后的结果也证明了这一点，参与极地冲刺的五位成员都是同去同回，没有抛弃或放弃。</p><p>举个例子，在大雾封锁了南极冰川，到处都是致命的冰川裂缝，队员和狗也都经历过掉入冰缝的极端情况下，当Amundson询问队员是否还要冒着生命危险向南极继续挺进时，所有人的回答竟然都是一致的：“我们冲吧！”</p><p>相比之下，Scott在人才的选择上就有一些疏漏。由于他们选择使用机动雪橇，技术上本来就不够成熟，人类更是没有在南极使用这类机械的经验。但就是因为副官和机械师之间的个人间隙，导致副官强烈反对机械师参与南极探险。而Scott竟然也同意了这一建议。结果机动雪橇很快就出了故障，没有帮上任何忙。</p><p>此外，Scott要求团队在南纬82.5度的地方来迎接自己。但传递这个消息的人却没把消息传到位，最后只派了团队中最年轻、最缺乏判断力的人，Apsley，到南纬80度去迎接Scott。于是Apsley到达南纬80度后就按照指令停了下来，没有灵活应变，前往Scott期望的82.5度。</p><p>结果Apsley在食物储藏点整整等了7天，饿死在帐篷里。事实上，Apsley的雪橇每天至少可以跑15英里，完全有可能找到Scott，把队友救回来。但他没有见机行事。很遗憾，Scott就是在这7天里饿死的。而他饿死的地方，距离Apsley只有11英里！</p><p>如果Scott有一个称职的机师，或者一个有应变能力的队友，或者在选择冲刺团队时选择４个体力最优的人，再或者传达命令的人更靠谱一些，那么Scott完全有可能像他计划地那样安全回到大本营。可以说，他错误的用人选择，导致了最终的失败。</p><h3>资源决定成败</h3><p>Scott团队的死因一般被归为饥饿。虽然他们的船队携带了大量资源，但真正投入到南极探险上的资源其实严重不足。参与最后冲刺的5名队员，全部由于饥饿无法抵抗严寒而导致死亡。坚持到最后的三个人，也倒在了距离食物储藏点只有11英里的地方。而这个距离，是一个饱腹状态的人一天就能完成的行走距离。</p><p>Amundson的资源虽然在成本上低于Scott，但是更适合极地条件。除了前面提到的狗和雪橇外，滑雪板、爱斯基摩人的防雪盲的眼罩、狼皮大衣等，都与探险环境更匹配。</p><p>最后冲刺的同样是5个人，Amundson的团队准备了15吨食物，分别在南纬80度、80.3度和82度设置了食物储藏点。而Scott团队仅仅准备了不到1吨的食物，只在南纬79.3度设置了食物储藏点。如果他们也能像Amundson团队一样，哪怕把储藏点放在距离南极最远的南纬80度，也能安全回到大本营。</p><p>持有这种观点的认为，所谓工欲善其事，必先利其器。Amundson使用的是与环境完全匹配的，并且是经过时间验证的可靠的装备。而Scott团队使用的装备和资源配置，都不适合南极恶劣的环境。事实证明，Scott团队被冻死就与资源强相关，尤其是食物不足。</p><p>BBC在2006做了一次重现（Re-Enactment），完全按照当时的食物、装备试图重走两人的探险路线。英国团队又一次失败了，队员们给出的理由就是食物严重不足。所以这次重现从某种程度上也证实了这个观点的价值。</p><h3>复杂性决定成败</h3><p>也有观点认为系统复杂性决定了探险的成败。Scott团队的工种非常多，有科学家、工程师、不同类型的军种代表。依赖的设备也很复杂，冲击南极的计划也很复杂，甚至接应的计划都很复杂。这种复杂性之下一环套一环，任何一个强依赖失败都会导致最终的失败。</p><p>比如Scott的南极冲刺采用了像火箭发射一样的多级策略，先将16人送到第一个补给站，而后再选择其中12人到达第二级补给站，再然后是8人，最后选择５人冲刺。</p><p>在整个过程中，谁来接应、每个人应该吃多少食物、给其他人留下多少食物、在哪里接应冲刺队员，接应是为了早点发布到达南极的消息，还是把人安全带回来，这些目标和决策原则都因为计划和沟通过于复杂，导致计划与沟通的记录缺失，以至于很多年后历史学家还在研究<a href="https://en.wikipedia.org/wiki/Comparison_of_the_Amundsen_and_Scott_Expeditions">到底是谁掉了链子</a>。</p><p>另外一个直接的后果是，这种接力模式导致他们存储在补给站的煤油被开罐使用后，因为没有密封好而挥发了，食物不能得到充分的加热。Scott也在自己的日记里提到他们的食物和煤油不够用了。</p><p>对比之下，Amundson团队的计划就简单很多。一行五人出发，自行回来。路上风险点的数量少，Amundson真正关注的点就比较充分。所以这种观点认为，Amundson选择了更加简单的计划，那么最终失败的概率就会小一点。</p><h2>思考题</h2><p>在你看来，哪个原因才是真正具有决定性的呢？欢迎把你的思考和想法分享出来。</p><p>最后也有一个小提示。这节课我分享了Scott和Amundson的南极探险故事，以及后人的观点。故事本身非常长，我只能介绍一些背景，还是很期望你能抓紧时间阅读一些资料，最大化自己的学习效果。</p><p>另外，关于南极探险的故事背景，我的抖音号“郭东白”里有一些补充内容。期望你在进入下节课之前学习一下。如果还有其他的问题，也欢迎你提出来，我可以及时解答。这样才能帮助形成更完整的思考，也让我们形成有效的思维碰撞。</p>
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
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/09/fb/52a662b2.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>spark</span>
  </div>
  <div class="_2_QraFYR_0">郭老师，take away~~~首先这个目标是可达的，是因为人的思考逻辑和决策，导致人为简单失败。要用思考的方式去记忆，不是记忆的方式去思考~~~<br>其次，思维方式的重要性~~~<br>递归思维是以终为始，逆向思维，自顶向下，做减法，从10000减到1。<br>递推思维是从小到大，做加法，从1加到10000，坚持一个月放弃了，因为至少需要3年~~~<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: “要用思考的方式去记忆，不是记忆的方式去思考”   这句话超赞！<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-05 00:37:00</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1d/55/4a/8a841200.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ACK</span>
  </div>
  <div class="_2_QraFYR_0">关于第一个南极探险故事的一些思考，我查阅了一些资料，也有自己的浅见，请老师指导<br><br><br>关于 1911 年阿蒙森团队第一个到达南极点演讲的思考。有计划容易做成事，但是光有计划是不够的，不应过度崇拜计划，不应认为任何事有计划就能办成，敬畏市场，敬畏自然也是同样重要，因为运气也是成功的因素之一。<br><br>1、周全的计划和对计划的执行能力是成功的可靠保障。<br><br>2、但一件事情能做成，除了可控因素外，不可控因素也会对结果造成很重要的影响。运气也是成功的一部分。因为如果阿蒙森团队在计划中遇到了暴风雪，如果外部的风险到达了生理极限，他们也会像斯科特团队一样会停留一段时间，只是阿蒙森团队这次探险时的天气好，在计划之内。1928年，阿蒙森在一次飞跃北极的探险中，失去了联系，最终没有找到下落，这次探险阿蒙森应该更有经验、计划更周全了，但是还是失败。<br><br>3、斯科特团队的失败，计划不周是一个因素。恶劣的天气和当时对存放燃油物资的铝在低温下金色属性变化造成燃油泄漏也是一个因素。毕竟能去南极探险说明已经是有非常专业的能力了，并且面临死亡的威胁的情况下，强大的求生欲也会激发他们想尽办法来解决问题。所以，斯科特团队全军覆没除了计划不周外，外部因素也是悲剧发生的重要原因之一。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的，关于计划， Amundson自己的确是这么说的， 他后来还有一本传记， 也是这么写的。   运气好这个说法Amundson自己是不太认可的。 倒是也可以理解， 哈哈哈。  <br>不过“如果阿蒙森团队在计划中遇到了暴风雪”这件事情上有明确答案： 他们的确遇到过，而且也不止一次， 在第一次False start 和后来的Butcher Point 碰到的都非常大， 甚至比Scott碰到的更恶劣。 <br>Scott不是运气更差， 是他把自己带到了一个运气更长的时间段， 南极的冬天来了， 能没有暴风雪吗？ 而且两个团队在同一个冬天出发， 虽然有一定的距离， 但是并不是相隔千里， 所以整体气候条件对两个团队是公平的。 <br>“1928年，阿蒙森在一次飞跃北极的探险中，失去了联系，最终没有找到下落，这次探险阿蒙森应该更有经验、计划更周全了，但是还是失败。” 这个过程存世的描述几乎没有。  这次不是他自己去探险， 而是临时出发去营救其他人的，计划不太可能很完美。<br><br><br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-05 08:53:30</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/22/ba/32/cf75ea4b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>大道至简</span>
  </div>
  <div class="_2_QraFYR_0">目标最重要，目标决定了南极探险的成败。<br>目标决定要找什么样的人<br>资源是人决策出来的，细节是人执行出来的<br>领导力不行，如果有足够优秀的下属，可以补足一些能力不足<br>复杂性是一系列决策影响的</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-07 17:20:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/17/2e/f0/0bbb0df5.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>乐天</span>
  </div>
  <div class="_2_QraFYR_0">之前这个南极探险故事听过多次，但每次听到总能有点不一样的想法，特别是东白老师把这个故事和架构活动关联到一起。<br>我自己总结一下，两个团队有这几点差别：<br>1、团队的差异。阿蒙森的团队更有凝聚力，人数不多，但每人都很强力。斯科特的团队更庞大，但没有很多的组织管理，各人分工不明确。<br>2、计划及保障措施不同。阿蒙森的计划就是到达南极并顺利返回，同时在多处预埋食物，食物也更充足。斯科特的计划则包含多重目标，但最关键的目标我认为应该是带领队伍到达南极并安全返回，这点不够突出。同时，食物准备不足，在冰天雪地，没有食物保障，怎么能保证目标达成呢？<br>3、切实可行的方法和工具。南极探险是一项难度很高的项目，中间肯定会有很多难点，需要有针对性的办法。阿蒙森的团队使用狗拉雪橇，比起斯科特使用机动雪橇，在当时的场景肯定更适用。<br>斯科特的探险最终以返程路上牺牲而结束，软件开发史上，我相信也有很多的项目，也因各种原因烂尾或者严重延期。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-23 14:28:16</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>明兴</span>
  </div>
  <div class="_2_QraFYR_0">个人觉得目标是核心原因，目标会牵引领导者去对团队配置，资源配置，路线等进行优化。目标分散不利于这些调整。scott以科学考察等为目标，导致了团队，资源等配置不合理，已经在有限的精力下忽视了生存这个目标所需要的准备工作</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-03-05 11:41:04</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src=""
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Geek_42abae</span>
  </div>
  <div class="_2_QraFYR_0">很喜欢这个事故，我写了一篇读后感：https:&#47;&#47;mp.weixin.qq.com&#47;s?__biz=MzU5MzEzOTY3Ng==&amp;mid=2247484305&amp;idx=1&amp;sn=91a601f6ac26b841cce13d0f493e59ff&amp;chksm=fe144122c963c834ed1d28e9b69f26ee3c554a2440105f4e53de8206ec751dba6020375aa7f7&amp;token=105859240&amp;lang=zh_CN#rd</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2023-01-21 15:39:45</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/42/bf/8d366dd4.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>波斯码</span>
  </div>
  <div class="_2_QraFYR_0">没有去看相关书籍，从文中获取的信息，我认为是对风险的识别和准备不足，比如雪橇坏掉的问题、煤油不够的问题，这其中是否应该有一些最佳实践。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的， 不过这里面挑战在于最佳实践也是靠经验和判断而来的。  尤其是探险家， 等到别人都总结出最佳实践了，你也没啥出名的机会了</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-21 20:36:38</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/10/99/85/12a7cc69.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Linsto</span>
  </div>
  <div class="_2_QraFYR_0">看到这一节，我又切到抖音看您对 Amundson &amp; Scott 的视频分享，不小心又点到了“程序员怎么处理大 BUG”，哈哈哈哈，太好玩了，东白老师做学问非常较真，行文非常准确、严肃，原来生活中也可以如此活泼有趣的啊，有趣，太有趣了。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 哈哈哈， 谢谢谢谢！</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-09-06 17:24:28</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/9b/91/5e105a9e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Cfunx</span>
  </div>
  <div class="_2_QraFYR_0">Scott的团队对方案的可行性评估不到位，出现这种情况，是因为其成功的经验、装备上的优势、傲慢自大、还是意识不到位？ 或是以上因素的种种叠加。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 方案也并非完全不可行。 后世有学者认为接应者的确是掉了链子的。  </p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-08-12 08:16:24</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/21/14/423a821f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Steven</span>
  </div>
  <div class="_2_QraFYR_0">我认为【目标】真正决定了成败：<br>做事情需要目标明确，尽量简单。目标越多，想的越多，越不容易达成。而且如果多个目标，是否有优先级呢。必要的时候保留必保的目标，甚至终止项目。<br><br>Scott团队人力，资金各方面都充足，装备精良，到南极只是多个目标中的一个。也缺少了 Amundson 团队那种必胜的、破釜沉舟的决心。<br></div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 多谢回复</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-26 15:15:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/8c/e1/63adf36f.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>欧阳绍聪</span>
  </div>
  <div class="_2_QraFYR_0">我认为是做出决策的依据，是基于预测。预测未来应该是挺不靠谱的一件事。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 我没有准确理解你想表达的意思， 感觉这里似乎少了个词：<br>我认为【？？】是做出决策的依据，是基于预测。<br></p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-06 16:59:33</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/2b/a0/50/390187f3.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>罗均</span>
  </div>
  <div class="_2_QraFYR_0">东白老师的课程实在太精彩了，一幕幕英雄探险的镜头在脑海里显现，又同时与老师以往授课的主题一一关联！<br><br>思考此课的作业，学生认为两位英雄成败的关键，在于“目标的唯一性与正确性”，Scott或许就败在了“唯一性”！<br><br>古代圣王尧帝禅让给舜帝传授了八字经文：“惟精惟一，允阙执中。”“惟精”就是在整个活动中，客观地如大浪淘沙般地分析一切有限资源与客服种种困难——“允阙”，不偏不倚地“执中”“惟一”的目标。<br><br>敬请老师批评指正。</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这的确是个关键因素。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-07-05 14:00:21</div>
  </div>
</div>
</div>
</li>
</ul>