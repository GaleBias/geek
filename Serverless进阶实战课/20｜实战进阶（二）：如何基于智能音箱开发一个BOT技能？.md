<audio title="20｜实战进阶（二）：如何基于智能音箱开发一个BOT技能？" src="https://static001.geekbang.org/resource/audio/8f/f5/8f8d216449b1e340bb29b81996a8f9f5.mp3" controls="controls"></audio> 
<p>你好，我是静远。</p><p>上节课，我们一起感受了FaaS 作为Serverless “连接器”的强大，但也仅限于云厂商单个云平台上的体验。</p><p>今天，我将带你体验跨平台的开发，通过智能音箱、IOT 开发平台、云函数计算平台三者之间的联动，一起完成一个不同年龄段宝宝食谱推荐BOT技能的开发。</p><p>你也可以通过这样的方式，完成<strong>智能客服、智能问答等对话式应用</strong>的开发。通过这个实战，相信你能进一步感受到Serverless的无限可能。</p><h2>如何实现</h2><p>我们本次的目标是通过语音对话的方式来完成宝宝食谱的推荐。这个功能符合Serverless的特性：事件触发、轻量应用、按需调用、流量不固定。同时，该功能需要有语音转文字的处理能力以及一个物理载体。</p><p>因此，我们以<a href="https://cloud.baidu.com/product/cfc.html">百度智能云函数计算平台</a>、<a href="https://dueros.baidu.com/dbp/main/console">百度DuerOS开放平台</a>来进行本次的案例开发，并选择一个小度X8作为载体。当然，你也可以直接用DuerOS平台上的<a href="https://dueros.baidu.com/didp/doc/dueros-bot-platform/dbp-quickstart/skills-test-sample_markdown">模拟测试工具</a>达到一样的效果。</p><p>正所谓磨刀不误砍柴工，在开始实战之前，我们先了解一下今天涉及到的各类概念以及整体的实现思路。</p><h3><strong>什么是技能</strong><strong>（BOT）</strong><strong>？</strong></h3><p>我们通常将<strong>在DuerOS平台上开发的对话式服务</strong>称为“技能”，它还有一个英文名，“BOT”。一般来说，平台有内置技能和开发者提交发布的两种技能。前者可以直接拿来用，后者分为免费和收费两种，收费的技能和你在小度音箱上看到的一样，是需要支付购买才能使用的。</p><!-- [[[read_end]]] --><p>一个技能通常由4个重要部分组成，我们的动手实战中都会涉及到。</p><ol>
<li><strong>意图</strong>：指技能要满足的用户请求或目的。比如这次的意图就是给不同年龄段的宝宝推荐食谱，由“常用表达语”“槽位信息”等组成；</li>
<li><strong>词典</strong>：是某领域词汇的集合，也是用户与技能交互过程中的重要信息。比如我们本次实验中需要识别的关键词“年龄”就是一个词典中的词条，对应英文识别为number。为了方面开发者，DuerOS在词典中内置了不少词条，比如这里的number就是sys.number；</li>
<li><strong>用户表达</strong>：是用户表达意图时具体的样例，它是组成意图的关键。用户表达样例越多，意图识别能力越强。槽位信息就是从用户表达语句中提取出的关键信息，并能够将这些关键信息和词典一一匹配。比如“3岁宝宝吃什么”就是一句用户表达，提取的槽位标识是number，数值是3，对应的词典是sys.number；</li>
<li><strong>配置服务</strong>：技能创建成功后，需要部署到云服务上，我们配置函数计算的入口函数标识BRN就可以了。BRN可以在函数计算平台上的函数基本信息展示里获取到。如“brn:bce:cfc:bj:*******:function:babaycook:$LATEST”。</li>
</ol><h3>实现思路</h3><p>接下来，我们看联动示意图，感知一下“宝宝食谱”技能通过设备和平台实现交互的过程。</p><p><img src="https://static001.geekbang.org/resource/image/93/1c/93c4b2becec85f8faae0eebae33b241c.jpg?wh=1920x689" alt="图片"></p><p>如图所示，小度音箱在收到语音输入后，如“小度小度，宝宝能吃什么？”，会通过既定协议请求DuerOS平台，通过DuerOS触发器触发函数计算CFC的执行。之后，你编写好的处理逻辑就会按照指令输出返回给DuerOS平台，并通过语音的方式在小度音箱播放。</p><p>这里使用的DuerOS触发器是百度CFC平台特有的一种触发器类型，用于和DuerOS联动，当然，你也可以自定义任何的<a href="https://time.geekbang.org/column/article/560737">触发器</a>。</p><p>如果涉及到存储，你可以选择云上的存储资源来使用，如Redis、RDS等等，这些也都在之前的课程中讲到过，相信你已经驾轻就熟了。</p><p>接下来，我带你来梳理一下开发的实现思路，我把整个过程划分为3大步骤。</p><p><strong>步骤一，云函数创建与自定义</strong>：你需要在CFC管理端选择一个合适的模板，快速创建一个DuerOS的初始函数，完成函数的开发和配置，最后设置触发器和配置日志存储就可以了。</p><p><strong>步骤二，技能创建与绑定</strong>：在DuerOS平台选择一个技能模板，并设置相应的元信息，根据你的诉求，设置意图和词典、话术场景配置等等。最后绑定CFC函数的时候，将函数入口的BRN拷贝到DuerOS平台即可。</p><p><strong>步骤三</strong><strong>，</strong><strong>运行时请求路由</strong>：其实这一步，不需要你做什么，直接体验就行。但如果你想发布到智能音箱上，还需要在控制台发布。发布之后，你就可以通过音箱来体验了。</p><p>整个流程你也可以对着下面的架构示意图梳理一遍。</p><p><img src="https://static001.geekbang.org/resource/image/e6/03/e666e8c7889ec7953143c0bf7af6d503.jpg?wh=1920x887" alt="图片"></p><h2>动手实战</h2><p>下面，我们就可以根据上面的思路开始动手实现不同年龄段宝宝食谱推荐BOT技能的开发了。</p><h3>云函数创建和定义</h3><p>首先，我们在函数计算平台CFC上创建一个函数，就本案例来说，为了便于你熟悉DuerOS的处理方式，可以选择一个模板来生成，比如我这里选择“dueros-bot-tax”来做案例演示。</p><p><img src="https://static001.geekbang.org/resource/image/5e/f3/5ecfed3754d963cdeaba0ed3fff5d0f3.jpg?wh=992x812" alt="图片"></p><p>然后，输入相关的函数元信息，这里函数名为“baby_recipe”。</p><p><img src="https://static001.geekbang.org/resource/image/7a/e0/7a573a069ae274617253a75f11350de0.jpg?wh=991x795" alt="图片"></p><p>我们先不用关心模板代码是什么，先继续创建触发器。这里需要选择DuerOS触发器来对接小度音箱的请求。</p><p><img src="https://static001.geekbang.org/resource/image/55/b6/55eec3714571f471f86700215ab305b6.jpg?wh=739x348" alt="图片"></p><p>好了，到这里，一个小度处理的函数就搞定了。接下来，我们就基于模板编写自己的业务逻辑代码。</p><p>这里需要注意一下，生成的模板代码中，在文件的开头，有一段PUBLIC KEY需要复制到DuerOS平台。先不要急，我们继续关注代码。</p><pre><code class="language-javascript">*/
/**
&nbsp;* @file&nbsp;&nbsp; index.js 此文件是函数入口文件，用于接收函数请求调用
&nbsp;* @author dueros
&nbsp;*/
const Bot = require('bot-sdk');
const privateKey = require('./rsaKeys.js').privateKey;
&nbsp;
class InquiryBot extends Bot {
&nbsp;&nbsp;&nbsp; constructor(postData) {
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; super(postData);
&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; this.addLaunchHandler(() =&gt; {
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; this.waitAnswer();
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; return {
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; outputSpeech: '欢迎使用小宝宝食谱推荐!'
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; };
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; });
&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; this.addSessionEndedHandler(() =&gt; {
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; this.endSession();
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; return {
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; outputSpeech: '多谢使用小宝宝食谱推荐!'
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; };
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; });
&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; this.addIntentHandler('babyCookBooks', () =&gt; {
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; let ageVar = this.getSlot('number');
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; if (!ageVar) {
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; this.nlu.ask('number');
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; let card = new Bot.Card.TextCard('宝宝几岁了呢');
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; // 可以返回异步 Promise
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; return Promise.resolve({
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; card: card,
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; outputSpeech: '宝宝几岁了呢'
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; });
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; }
&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; if (this.request.isDialogStateCompleted()) {
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; let cook_1 = '母乳为主，可以混合牛奶，建议辅以虾仁碎沫或者菜沫等';
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; let cook_3 = '可以吃富含蛋白质的食物如：鸡蛋，瘦肉。也可以吃富含维生素的食物:如芹菜、菠菜。';
&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; var speechOutput = cook_1;
&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; if(ageVar &gt; 1 ) {
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; speechOutput = cook_3;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; }
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; let card = new Bot.Card.TextCard(speechOutput);
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; return {
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; card: card,
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; outputSpeech: speechOutput
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; };
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; }
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; });
&nbsp;&nbsp;&nbsp; }
}
&nbsp;
exports.handler = function (event, context, callback) {
&nbsp;&nbsp;&nbsp; try {
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; let b = new InquiryBot(event);
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; // 0: debug&nbsp; 1: online
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; b.botMonitor.setEnvironmentInfo(privateKey, 0);
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; b.botMonitor.setMonitorEnabled(true);
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; b.run().then(function (result) {
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; callback(null, result);
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; }).catch(callback);
&nbsp;&nbsp;&nbsp; }
&nbsp;&nbsp;&nbsp; catch (e) {
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; callback(e);
&nbsp;&nbsp;&nbsp; }
};
&nbsp;
</code></pre><p>代码中，我们主要关注的是意图的处理入口this.addIntentHandler(‘babyCookBooks’)，其中“babyCookBooks”是在DuerOS技能平台创建技能的时候，需要填写的意图名称，你也可以按照后续自己定义的技能名称来修改。</p><p>接下来的代码部分，为了理解方便，我引入了一个宝宝菜谱推荐的简单判断，你可以在这之上添加更丰富的功能。将复杂的功能抽离出来，然后引入调用，就能实现比较完整的技能了。</p><h3>技能创建与绑定</h3><p>接下来就是创建技能了。进入<a href="https://dueros.baidu.com/dbp/main/console">DuerOS技能开放平台</a>，点击授权，点击创建新技能。</p><p><img src="https://static001.geekbang.org/resource/image/15/fd/159259a7131f627c05139610ea967ffd.jpg?wh=1213x690" alt="图片"></p><p>选择“自定义技能”，“从头开始”。</p><p><img src="https://static001.geekbang.org/resource/image/8f/4d/8ffa88fe70c4bde7cb5f073f53c4214d.jpg?wh=1192x745" alt="图片"></p><p>下拉页面，在“技能名称”和“调用名称”部分输入“宝宝食谱推荐”，应用场景选择有屏和无屏场景，点击确定：</p><p><img src="https://static001.geekbang.org/resource/image/46/0c/46a41db09787d7f1cd51a25775a8310c.jpg?wh=810x756" alt="图片"></p><p>接下来，继续创建意图：</p><p><img src="https://static001.geekbang.org/resource/image/1b/e7/1bb21c064379049yycf608486bcea5e7.jpg?wh=1007x689" alt="图片"></p><p>这里，我们需要注意两点。</p><p>第一，意图识别名的填写，需要跟你在函数计算CFC中addIntentHandler设置的名称保持一致。</p><p>第二，<a href="https://dueros.baidu.com/didp/doc/dueros-bot-platform/dbp-nlu/intents_markdown">表达式和槽位</a>比较关键。不同用户的表达形式不同，我们需要确保常用表达覆盖尽可能多的用户口语表达，这样意图识别就会更加准确。DuerOS平台可以通过表达式提取出槽位信息，你也可以手动核准和纠正。</p><p>槽位是意图的参数信息，是非常关键的一环，我们在代码里面获取到的参数就是槽位标识。更多关于技能的使用方法，如果你感兴趣的话，也可以在DuerOS开发平台深入了解。</p><p><img src="https://static001.geekbang.org/resource/image/05/65/05f4c684b877d1d42eba77371d65c265.jpg?wh=1504x817" alt="图片"></p><p>接着，我们来关联技能与后端的服务。我们选用百度CFC作为服务部署，<strong>依次拷贝函数计算代码中的PUBLIC KEY<strong><strong>以及</strong></strong>函数的唯一标识BRN到“配置服务”页面的“Public Key”和“BRN”框中。</strong></p><p><img src="https://static001.geekbang.org/resource/image/70/96/70ff5cfff43e9ec538b63b6413ccc196.jpg?wh=1126x726" alt="图片"></p><h3>测试与发布</h3><p>到这里，我们就完成了云函数的创建和定义，技能创建与绑定两个关键的步骤。最后，我们来测试一下程序是否能够正常运行。</p><p>我们可以选择模拟测试或者真机测试两种方法。假如你身边暂时没有小度音箱，可以选择模拟测试来验证一下。我们打开“模拟测试”，选择“小度在家”，输入“打开宝宝食谱推荐”这个技能。</p><p><img src="https://static001.geekbang.org/resource/image/44/cc/44484606fd4da689454f22f929f35fcc.jpg?wh=829x664" alt="图片"></p><p>你会看到，小度的屏幕上出现了我们刚才代码中的欢迎语“欢迎使用小宝宝食谱推荐”。</p><p><img src="https://static001.geekbang.org/resource/image/96/d1/96b26f6f95a6c8542c5fc54985c75ed1.jpg?wh=829x672" alt="图片"></p><p>接着，我们输入“1岁宝宝吃什么”，会发现技能服务给我们返回的正是代码中“1岁宝宝的食谱推荐”用语：“母乳为主，可以混合牛奶，建议辅以虾仁碎沫或者菜沫等”。</p><p><img src="https://static001.geekbang.org/resource/image/07/d3/0725a53ba31063ccd8813e97860033d3.jpg?wh=826x650" alt="图片"></p><p>假如你的身边有小度音箱，也可以选择“真机测试”，打开“技能调试模式”。这里要注意的是，你登录的DuerOS平台的账号一定要和登录小度智能音箱的账号一致。</p><p><img src="https://static001.geekbang.org/resource/image/dd/54/dd38b4b080675ece8dfc0855a5133b54.jpg?wh=1870x455" alt="图片"></p><p>我们来看一段视频，感受一下通过函数计算实现的技能：</p><p><video poster="https://media001.geekbang.org/1e5ab74c22364c4bacd27a2e6b9eb906/snapshots/fe2e8b6aaf8547c48da5b23088d07e55-00004.jpg" preload="none" controls=""><source src="https://media001.geekbang.org/customerTrans/7e27d07d27d407ebcc195a0e78395f55/2f67c8eb-183c694136d-0000-0000-01d-dbacd.mp4" type="video/mp4"><source src=" https://media001.geekbang.org/3a9b1cf100ab4bfda5d1460024398f14/293f3807b98647f0b282df99621df81a-0ada43293ac128109685958b406d87b9-sd.m3u8" type="application/x-mpegURL"></video></p><p>最后，经过精修和加工后的程序，也可以正式发布出来分享给其他用户。</p><h2>小结</h2><p>最后我们来小结一下，今天这节课我们通过联动智能音箱、DuerOS技能开放平台、云函数计算CFC一起完成了一个宝宝食谱BOT技能的开发体验。</p><p>通过这次的实验，我们可以发现，一个基于Serverless和语音交互的对话式应用开发，主要包括如下三个步骤：</p><p>步骤一，构建一个符合业务场景的Serverless应用，你可以使用函数打包或者镜像的方式完成开发和上线，如本案例中的宝宝食谱函数服务；</p><p>步骤二，构建一个交互技能的意图，设置好语音交互规则，如本案例中的宝宝食谱意图；</p><p>步骤三，构建技能和函数的触发绑定。如本案例中创建DuerOS触发器的过程。</p><p>基于这三个流程和延伸，由于DuerOS的开源开放，可以广泛支持手机、电视、音箱、汽车、机器人等多种硬件设备，我们还可以用来开发智能客服、居家电器控制、智能助理等多种对话式应用，通过DuerOS与Serverless函数计算的结合，可以进一步降低健康、金融、酒旅、电信等各行各业使用人工智能对话系统的门槛。</p><p>其实，各大云厂商在基于Serverless的生态集成上都有非常丰富的经验和成熟的产品，如果你的业务已经在云上，那么可以挖掘一下哪些场景还可以用起来，进一步减少运维和处理带来的工作量；如果你的业务暂时还没有上云，这种开放集成和Serverless本身所体现出的这种思想，希望你也能在现有的工作中灵活运用起来。</p><p>最后，这节课虽然是实验课，但我最想跟你交流的，是这次实验背后的两个感悟。一个是发散思维很重要，我们可以有意识地将身边的需求和技术联系起来，活学活用Serverless；另一方面，在工作生活压力比较大的现在，<strong>作为码农的我们，<strong><strong>也可以</strong></strong>运用自己的技术，给孩子带来成长的快乐</strong>，还可能会带来一份收入，在这么“卷”的当下，是不是也能感受到不一样的幸福呢？</p><h2>思考题</h2><p>好了，这节课到这里也就结束了，最后我给你留了一个思考题。</p><p>少儿编程的后端服务是否可以采用Serverless的技术？哪些地方可以用到函数计算，哪些可以用弹性应用托管服务更合适？</p><p>欢迎在留言区写下你的思考和答案，我们一起交流讨论。</p><p>感谢你的阅读，也欢迎你把这节课分享给更多的朋友一起阅读。</p>