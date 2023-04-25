<audio title="05 _ 规范设计（下）：commit 信息风格迥异、难以阅读，如何规范？" src="https://static001.geekbang.org/resource/audio/d0/76/d027ed5729f302482bd654b860b48676.mp3" controls="controls"></audio> 
<p>你好，我是孔令飞。今天，我们继续学习非编码类规范中的 Commit 规范。</p><p>我们在做代码开发时，经常需要提交代码，提交代码时需要填写 Commit Message（提交说明），否则就不允许提交。</p><p>而在实际开发中，我发现每个研发人员提交 Commit Message 的格式可以说是五花八门，有用中文的、有用英文的，甚至有的直接填写“11111”。这样的 Commit Message，时间久了可能连提交者自己都看不懂所表述的修改内容，更别说给别人看了。</p><p>所以在 Go 项目开发时，一个好的 Commit Message 至关重要：</p><ul>
<li>可以使自己或者其他开发人员能够<strong>清晰地知道每个 commit 的变更内容</strong>，方便快速浏览变更历史，比如可以直接略过文档类型或者格式化类型的代码变更。</li>
<li>可以基于这些 Commit Message <strong>进行过滤查找</strong>，比如只查找某个版本新增的功能：<code>git log --oneline --grep "^feat|^fix|^perf"</code>。</li>
<li>可以基于规范化的 Commit Message <strong>生成 Change Log</strong>。</li>
<li>可以依据某些类型的 Commit Message <strong>触发构建或者发布流程</strong>，比如当 type 类型为 feat、fix 时我们才触发 CI 流程。</li>
<li><strong>确定语义化版本的版本号</strong>。比如 <code>fix</code> 类型可以映射为 PATCH 版本，<code>feat</code> 类型可以映射为 MINOR 版本。带有 <code>BREAKING CHANGE</code> 的 commit，可以映射为 MAJOR 版本。在这门课里，我就是通过这种方式来自动生成版本号。</li>
</ul><!-- [[[read_end]]] --><p>总结来说，一个好的 Commit Message 规范可以使 Commit Message 的可读性更好，并且可以实现自动化。那究竟如何写一个易读的 Commit Message 呢？</p><p>接下来，我们来看下如何规范 Commit Message。另外，除了 Commit Message 之外，我还会介绍跟 Commit 相关的 3 个重点，以及如何通过自动化流程来保证 Commit Message 的规范化。</p><h2>Commit Message 的规范有哪些？</h2><p>毫无疑问，我们可以根据需要自己来制定 Commit Message 规范，但是我更建议你采用开源社区中比较成熟的规范。一方面，可以避免重复造轮子，提高工作效率。另一方面，这些规范是经过大量开发者验证的，是科学、合理的。</p><p>目前，社区有多种 Commit Message 的规范，例如 jQuery、Angular 等。我将这些规范及其格式绘制成下面一张图片，供你参考：</p><p><img src="https://static001.geekbang.org/resource/image/16/48/1699f5c1933cfe72803dfb038152fc48.png?wh=973*172" alt=""></p><p>在这些规范中，Angular 规范在功能上能够满足开发者 commit 需求，在格式上清晰易读，目前也是用得最多的。</p><p>Angular 规范其实是一种语义化的提交规范（Semantic Commit Messages），所谓语义化的提交规范包含以下内容：</p><ul>
<li>Commit Message 是语义化的：Commit Message 都会被归为一个有意义的类型，用来说明本次 commit 的类型。</li>
<li>Commit Message 是规范化的：Commit Message 遵循预先定义好的规范，比如 Commit Message 格式固定、都属于某个类型，这些规范不仅可被开发者识别也可以被工具识别。</li>
</ul><p>为了方便你理解 Angular 规范，我们直接看一个遵循 Angular 规范的 commit 历史记录，见下图：</p><p><img src="https://static001.geekbang.org/resource/image/e2/fe/e227e4976406daaa039438feb5affefe.png?wh=825*420" alt=""></p><p>再来看一个完整的符合 Angular 规范的 Commit Message，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/da/cb/da69572c5605556b8144eb4ee281c4cb.png?wh=1877*406" alt=""></p><p>通过上面 2 张图，我们可以看到符合 Angular Commit Message 规范的 commit 都是有一定格式，有一定语义的。</p><p>那我们该<strong>怎么写出符合 Angular 规范的 Commit Message 呢</strong>？</p><p>在 Angular 规范中，Commit Message 包含三个部分，分别是 <strong>Header</strong>、<strong>Body</strong> 和 <strong>Footer</strong>，格式如下：</p><pre><code>&lt;type&gt;[optional scope]: &lt;description&gt;
// 空行
[optional body]
// 空行
[optional footer(s)]
</code></pre><p>其中，Header是必需的，Body和Footer可以省略。在以上规范中，<code>&lt;scope&gt;</code>必须用括号 <code>()</code> 括起来， <code>&lt;type&gt;[&lt;scope&gt;]</code> 后必须紧跟冒号 ，冒号后必须紧跟空格，2 个空行也是必需的。</p><p>在实际开发中，为了使 Commit Message 在 GitHub 或者其他 Git 工具上更加易读，我们往往会限制每行 message 的长度。根据需要，可以限制为 50/72/100 个字符，这里我将长度限制在 72 个字符以内（也有一些开发者会将长度限制为 100，你可根据需要自行选择）。</p><p>以下是一个符合 Angular 规范的 Commit Message：</p><pre><code>fix($compile): couple of unit tests for IE9
# Please enter the Commit Message for your changes. Lines starting
# with '#' will be ignored, and an empty message aborts the commit.
# On branch master
# Changes to be committed:
# ...

Older IEs serialize html uppercased, but IE9 does not...
Would be better to expect case insensitive, unfortunately jasmine does
not allow to user regexps for throw expectations.

Closes #392
Breaks foo.bar api, foo.baz should be used instead
</code></pre><p>接下来，我们详细看看 Angular 规范中 Commit Message 的三个部分。</p><h3>Header</h3><p>Header 部分只有一行，包括三个字段：type（必选）、scope（可选）和 subject（必选）。</p><p>我们先来说 <strong>type</strong>，它用来说明 commit 的类型。为了方便记忆，我把这些类型做了归纳，它们主要可以归为 Development 和 Production 共两类。它们的含义是：</p><ul>
<li>Development：这类修改一般是项目管理类的变更，不会影响最终用户和生产环境的代码，比如 CI 流程、构建方式等的修改。遇到这类修改，通常也意味着可以免测发布。</li>
<li>Production：这类修改会影响最终的用户和生产环境的代码。所以对于这种改动，我们一定要慎重，并在提交前做好充分的测试。</li>
</ul><p>我在这里列出了 Angular 规范中的常见 type 和它们所属的类别，你在提交 Commit Message 的时候，一定要注意区分它的类别。举个例子，我们在做 Code Review 时，如果遇到 Production 类型的代码，一定要认真 Review，因为这种类型，会影响到现网用户的使用和现网应用的功能。</p><p><img src="https://static001.geekbang.org/resource/image/89/27/89c618a7415c0c38b09d86d7f882a427.png?wh=726*511" alt=""></p><p>有这么多 type，我们该如何确定一个 commit 所属的 type 呢？这里我们可以通过下面这张图来确定。</p><p><img src="https://static001.geekbang.org/resource/image/35/a7/3509bd169ce285f59fbcfa6ebea75aa7.png?wh=2513*1078" alt=""></p><p>如果我们变更了应用代码，比如某个 Go 函数代码，那这次修改属于代码类。在代码类中，有 4 种具有明确变更意图的类型：feat、fix、perf 和 style；如果我们的代码变更不属于这 4 类，那就全都归为 refactor 类，也就是优化代码。</p><p>如果我们变更了非应用代码，例如更改了文档，那它属于非代码类。在非代码类中，有 3 种具有明确变更意图的类型：test、ci、docs；如果我们的非代码变更不属于这 3 类，那就全部归入到 chore 类。</p><p>Angular 的 Commit Message 规范提供了大部分的 type，在实际开发中，我们可以使用部分 type，或者扩展添加我们自己的 type。但无论选择哪种方式，<strong>我们一定要保证一个项目中的 type 类型一致。</strong></p><p>接下来，我们说说 Header 的第二个字段 <strong>scope</strong>。</p><p>scope 是用来说明 commit 的影响范围的，它必须是名词。显然，不同项目会有不同的 scope。在项目初期，我们可以设置一些粒度比较大的 scope，比如可以按组件名或者功能来设置 scope；后续，如果项目有变动或者有新功能，我们可以再用追加的方式添加新的 scope。</p><p>我们这门课采用的 scope，主要是根据组件名和功能来设置的。例如，支持 apiserver、authzserver、user 这些 scope。</p><p><strong>这里想强调的是，scope 不适合设置太具体的值</strong>。太具体的话，一方面会导致项目有太多的 scope，难以维护。另一方面，开发者也难以确定 commit 属于哪个具体的 scope，导致错放 scope，反而会使 scope 失去了分类的意义。</p><p>当然了，在指定 scope 时，也需要遵循我们预先规划的 scope，所以我们要将 scope 文档化，放在类似 devel 这类文档中。这一点你可以参考下 IAM 项目的 scope 文档： <a href="https://github.com/marmotedu/iam/blob/master/docs/devel/zh-CN/scope.md">IAM commit message scope</a> 。</p><p>最后，我们再说说 <strong>subject。</strong></p><p>subject 是 commit 的简短描述，必须以动词开头、使用现在时。比如，我们可以用 change，却不能用 changed 或 changes，而且这个动词的第一个字母必须是小写。通过这个动词，我们可以明确地知道 commit 所执行的操作。此外我们还要注意，subject 的结尾不能加英文句号。</p><h3>Body</h3><p>Header 对 commit 做了高度概括，可以方便我们查看 Commit Message。那我们如何知道具体做了哪些变更呢？答案就是，可以通过 Body 部分，它是对本次 commit 的更详细描述，是可选的。</p><p>Body 部分可以分成多行，而且格式也比较自由。不过，和 Header 里的<subject>一样，它也要以动词开头，使用现在时。此外，它还必须<strong>要包括修改的动机</strong>，以及<strong>和跟上一版本相比的改动点</strong>。</subject></p><p>我在下面给出了一个范例，你可以看看：</p><pre><code>The body is mandatory for all commits except for those of scope &quot;docs&quot;. When the body is required it must be at least 20 characters long.
</code></pre><h3>Footer</h3><p>Footer 部分不是必选的，可以根据需要来选择，主要用来说明本次 commit 导致的后果。在实际应用中，Footer 通常用来说明不兼容的改动和关闭的 Issue 列表，格式如下：</p><pre><code>BREAKING CHANGE: &lt;breaking change summary&gt;
// 空行
&lt;breaking change description + migration instructions&gt;
// 空行
// 空行
Fixes #&lt;issue number&gt;
</code></pre><p>接下来，我给你详细说明下这两种情况：</p><ul>
<li>不兼容的改动：如果当前代码跟上一个版本不兼容，需要在 Footer 部分，以 <code>BREAKING CHANG:</code> 开头，后面跟上不兼容改动的摘要。Footer 的其他部分需要说明变动的描述、变动的理由和迁移方法，例如：</li>
</ul><pre><code>BREAKING CHANGE: isolate scope bindings definition has changed and
    the inject option for the directive controller injection was removed.

    To migrate the code follow the example below:

    Before:

    scope: {
      myAttr: 'attribute',
    }

    After:

    scope: {
      myAttr: '@',
    }
    The removed `inject` wasn't generaly useful for directives so there should be no code using it.
</code></pre><ul>
<li>关闭的 Issue 列表：关闭的 Bug 需要在 Footer 部分新建一行，并以 Closes 开头列出，例如：<code>Closes #123</code>。如果关闭了多个 Issue，可以这样列出：<code>Closes #123, #432, #886</code>。例如:</li>
</ul><pre><code> Change pause version value to a constant for image
    
    Closes #1137
</code></pre><h3>Revert Commit</h3><p>除了 <strong>Header</strong>、<strong>Body</strong> 和 <strong>Footer 这 3 个部分，</strong>Commit Message 还有一种特殊情况：如果当前 commit 还原了先前的 commit，则应以 <code>revert:</code> 开头，后跟还原的 commit 的 Header。而且，在 Body 中必须写成 <code>This reverts commit &lt;hash&gt;</code> ，其中 hash 是要还原的 commit 的 SHA 标识。例如：</p><pre><code>revert: feat(iam-apiserver): add 'Host' option

This reverts commit 079360c7cfc830ea8a6e13f4c8b8114febc9b48a.
</code></pre><p>为了更好地遵循 Angular 规范，建议你在提交代码时养成不用 <code>git commit -m</code>，即不用-m 选项的习惯，而是直接用 <code>git commit</code> 或者 <code>git commit -a</code> 进入交互界面编辑 Commit Message。这样可以更好地格式化 Commit Message。</p><p>但是除了 Commit Message 规范之外，在代码提交时，我们还需要关注 3 个重点内容：提交频率、合并提交和 Commit Message 修改。</p><h2>Commit 相关的 3 个重要内容</h2><p>我们先来看下提交频率。</p><h3>提交频率</h3><p>在实际项目开发中，如果是个人项目，随意 commit 可能影响不大，但如果是多人开发的项目，随意 commit 不仅会让 Commit Message 变得难以理解，还会让其他研发同事觉得你不专业。因此，我们要规定 commit 的提交频率。</p><p>那到底什么时候进行 commit 最好呢？</p><p>我认为主要可以分成两种情况。一种情况是，只要我对项目进行了修改，一通过测试就立即 commit。比如修复完一个 bug、开发完一个小功能，或者开发完一个完整的功能，测试通过后就提交。另一种情况是，我们规定一个时间，定期提交。这里我建议代码下班前固定提交一次，并且要确保本地未提交的代码，延期不超过 1 天。这样，如果本地代码丢失，可以尽可能减少丢失的代码量。</p><p>按照上面 2 种方式提交代码，你可能会觉得代码 commit 比较多，看起来比较随意。或者说，我们想等开发完一个完整的功能之后，放在一个 commit 中一起提交。这时候，我们可以在最后合并代码或者提交 Pull Request 前，执行 <code>git rebase -i</code> 合并之前的所有 commit。</p><p>那么如何合并 commit 呢？接下来，我来详细说说。</p><h3>合并提交</h3><p>合并提交，就是将多个 commit 合并为一个 commit 提交。这里，我建议你把新的 commit 合并到主干时，只保留 2~3 个 commit 记录。那具体怎么做呢？</p><p>在 Git 中，我们主要使用 git rebase 命令来合并。git rebase 也是我们日后开发需要经常使用的一个命令，所以我们一定要掌握好它的使用方法。</p><p><strong>git rebase 命令介绍</strong></p><p>git rebase 的最大作用是它可以重写历史。</p><p>我们通常会通过 <code>git rebase -i &lt;commit ID&gt;</code>使用 git rebase 命令，<code>-i</code> 参数表示交互（interactive），该命令会进入到一个交互界面中，其实就是 Vim 编辑器。在该界面中，我们可以对里面的 commit 做一些操作，交互界面如图所示：</p><p><img src="https://static001.geekbang.org/resource/image/c6/ac/c63a8682c03862802e5eacf1641b86ac.png?wh=1345*866" alt=""></p><p>这个交互界面会首先列出给定<code>&lt;commit ID&gt;</code>之前（不包括<commit id="">，越下面越新）的所有 commit，每个 commit 前面有一个操作命令，默认是 pick。我们可以选择不同的 commit，并修改 commit 前面的命令，来对该 commit 执行不同的变更操作。</commit></p><p>git rebase 支持的变更操作如下：</p><p><img src="https://static001.geekbang.org/resource/image/5f/f2/5f5a79a5d2bde029d4de9d98026ef3f2.png?wh=629*393" alt=""></p><p>在上面的 7 个命令中，squash 和 fixup 可以用来合并 commit。例如用 squash 来合并，我们只需要把要合并的 commit 前面的动词，改成 squash（或者 s）即可。你可以看看下面的示例：</p><pre><code>pick 07c5abd Introduce OpenPGP and teach basic usage
s de9b1eb Fix PostChecker::Post#urls
s 3e7ee36 Hey kids, stop all the highlighting
pick fa20af3 git interactive rebase, squash, amend
</code></pre><p>rebase 后，第 2 行和第 3 行的 commit 都会合并到第 1 行的 commit。这个时候，我们提交的信息会同时包含这三个 commit 的提交信息：</p><pre><code># This is a combination of 3 commits.
# The first commit's message is:
Introduce OpenPGP and teach basic usage

# This is the 2ndCommit Message:
Fix PostChecker::Post#urls

# This is the 3rdCommit Message:
Hey kids, stop all the highlighting
</code></pre><p>如果我们将第 3 行的 squash 命令改成 fixup 命令：</p><pre><code>pick 07c5abd Introduce OpenPGP and teach basic usage
s de9b1eb Fix PostChecker::Post#urls
f 3e7ee36 Hey kids, stop all the highlighting
pick fa20af3 git interactive rebase, squash, amend
</code></pre><p>rebase 后，还是会生成两个 commit，第 2 行和第 3 行的 commit，都合并到第 1 行的 commit。但是，新的提交信息里面，第 3 行 commit 的提交信息会被注释掉：</p><pre><code># This is a combination of 3 commits.
# The first commit's message is:
Introduce OpenPGP and teach basic usage

# This is the 2ndCommit Message:
Fix PostChecker::Post#urls

# This is the 3rdCommit Message:
# Hey kids, stop all the highlighting
</code></pre><p>除此之外，我们在使用 git rebase 进行操作的时候，还需要注意以下几点：</p><ul>
<li>删除某个 commit 行，则该 commit 会丢失掉。</li>
<li>删除所有的 commit 行，则 rebase 会被终止掉。</li>
<li>可以对 commits 进行排序，git 会从上到下进行合并。</li>
</ul><p>为了加深你的理解，我给你完整演示一遍合并提交。</p><p><strong>合并提交操作示例</strong></p><p>假设我们需要研发一个新的模块：user，用来在平台里进行用户的注册、登录、注销等操作，当模块完成开发和测试后，需要合并到主干分支，具体步骤如下。</p><p><strong>首先，我们新建一个分支</strong>。我们需要先基于 master 分支新建并切换到 feature 分支：</p><pre><code>$ git checkout -b feature/user
Switched to a new branch 'feature/user'
</code></pre><p>这是我们的所有 commit 历史：</p><pre><code>$ git log --oneline
7157e9e docs(docs): append test line 'update3' to README.md
5a26aa2 docs(docs): append test line 'update2' to README.md
55892fa docs(docs): append test line 'update1' to README.md
89651d4 docs(doc): add README.md
</code></pre><p><strong>接着，我们在</strong> <code>feature/user</code>分支进行功能的开发和测试，并遵循规范提交 commit，功能开发并测试完成后，Git 仓库的 commit 记录如下：</p><pre><code>$ git log --oneline
4ee51d6 docs(user): update user/README.md
176ba5d docs(user): update user/README.md
5e829f8 docs(user): add README.md for user
f40929f feat(user): add delete user function
fc70a21 feat(user): add create user function
7157e9e docs(docs): append test line 'update3' to README.md
5a26aa2 docs(docs): append test line 'update2' to README.md
55892fa docs(docs): append test line 'update1' to README.md
89651d4 docs(doc): add README.md
</code></pre><p>可以看到我们提交了 5 个 commit。接下来，我们需要将 <code>feature/user</code>分支的改动合并到 master 分支，但是 5 个 commit 太多了，我们想将这些 commit 合并后再提交到 master 分支。</p><p><strong>接着，我们合并所有 commit</strong>。在上一步中，我们知道 <code>fc70a21</code>是 <code>feature/user</code>分支的第一个 commit ID，其父 commit ID 是 <code>7157e9e</code>，我们需要将<code>7157e9e</code>之前的所有分支 进行合并，这时我们可以执行：</p><pre><code>$ git rebase -i 7157e9e
</code></pre><p>执行命令后，我们会进入到一个交互界面，在该界面中，我们可以将需要合并的 4 个 commit，都执行 squash 操作，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/6e/4e/6e41e61c27ca2a46e55e8801c47cd04e.png?wh=1097*320" alt=""></p><p>修改完成后执行<code>:wq</code> 保存，会跳转到一个新的交互页面，在该页面，我们可以编辑 Commit Message，编辑后的内容如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/73/87/73a884bac481236969ba2a219a2e9187.png?wh=1184*399" alt=""></p><p><code>#</code>开头的行是 git 的注释，我们可以忽略掉，在 rebase 后，这些行将会消失掉。修改完成后执行<code>:wq</code> 保存，就完成了合并提交操作。</p><p>除此之外，这里有 2 个点需要我们注意：</p><ul>
<li><code>git rebase -i &lt;commid ID&gt;</code>这里的<commid id="">一定要是需要合并 commit 中最旧 commit 的父 commit ID。</commid></li>
<li>我们希望将 feature/user 分支的 5 个 commit 合并到一个 commit，在 git rebase 时，需要保证其中最新的一个 commit 是 pick 状态，这样我们才可以将其他 4 个 commit 合并进去。</li>
</ul><p><strong>然后，我们用如下命令来检查 commits 是否成功合并</strong>。可以看到，我们成功将 5 个 commit 合并成为了一个 commit：<code>d6b17e0</code>。</p><pre><code>$ git log --oneline
d6b17e0 feat(user): add user module with all function implements
7157e9e docs(docs): append test line 'update3' to README.md
5a26aa2 docs(docs): append test line 'update2' to README.md
55892fa docs(docs): append test line 'update1' to README.md
89651d4 docs(doc): add README.md
</code></pre><p><strong>最后，我们就可以将 feature 分支</strong> <code>feature/user</code> 的改动合并到主干分支，从而完成新功能的开发<strong>。</strong></p><pre><code>$ git checkout master
$ git merge feature/user
$ git log --oneline
d6b17e0 feat(user): add user module with all function implements
7157e9e docs(docs): append test line 'update3' to README.md
5a26aa2 docs(docs): append test line 'update2' to README.md
55892fa docs(docs): append test line 'update1' to README.md
89651d4 docs(doc): add README.md
</code></pre><p>这里给你一个小提示，如果你有太多的 commit 需要合并，那么可以试试这种方式：先撤销过去的 commit，然后再建一个新的。</p><pre><code>$ git reset HEAD~3
$ git add .
$ git commit -am &quot;feat(user): add user resource&quot;
</code></pre><p>需要说明一点：除了 commit 实在太多的时候，一般情况下我不建议用这种方法，有点粗暴，而且之前提交的 Commit Message 都要重新整理一遍。</p><h3>修改 Commit Message</h3><p>即使我们有了 Commit Message 规范，但仍然可能会遇到提交的 Commit Message 不符合规范的情况，这个时候就需要我们能够修改之前某次 commit 的 Commit Message。</p><p>具体来说，我们有两种修改方法，分别对应两种不同情况：</p><ol>
<li>git commit --amend：修改最近一次 commit 的 message；</li>
<li>git rebase -i：修改某次 commit 的 message。</li>
</ol><p>接下来，我们分别来说这两种方法。</p><p><strong>git commit --amend：修改最近一次 commit 的 message</strong></p><p>有时候，我们刚提交完一个 commit，但是发现 commit 的描述不符合规范或者需要纠正，这时候，我们可以通过 <code>git commit --amend</code> 命令来修改刚刚提交 commit 的 Commit Message。具体修改步骤如下：</p><ol>
<li>查看当前分支的日志记录。</li>
</ol><pre><code>$ git log –oneline
418bd4 docs(docs): append test line 'update$i' to README.md
89651d4 docs(doc): add README.md
</code></pre><p>可以看到，最近一次的 Commit Message 是 <code>docs(docs): append test line 'update$i' to README.md</code>，其中 <code>update$i</code> 正常应该是 <code>update1</code>。</p><ol start="2">
<li>更新最近一次提交的 Commit Message</li>
</ol><p>在当前 Git 仓库下执行命令：<code>git commit --amend</code>，后会进入一个交互界面，在交互界面中，修改最近一次的 Commit Message，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/9e/a3/9ea39c153fccdbfd951a990e131ddea3.png?wh=1362*466" alt=""></p><p>修改完成后执行:wq 保存，退出编辑器之后，会在命令行显示，该 commit 的 message 的更新结果如下：</p><pre><code>[master 55892fa] docs(docs): append test line 'update1' to README.md
 Date: Fri Sep 18 13:40:42 2020 +0800
 1 file changed, 1 insertion(+)
</code></pre><ol start="3">
<li>查看最近一次的 Commit Message 是否被更新</li>
</ol><pre><code>$ git log --oneline
55892fa docs(docs): append test line 'update1' to README.md
89651d4 docs(doc): add README.md
</code></pre><p>可以看到最近一次 commit 的 message 成功被修改为期望的内容。</p><p><strong>git rebase -i：修改某次 commit 的 message</strong></p><p>如果我们想修改的 Commit Message 不是最近一次的 Commit Message，可以通过 <code>git rebase -i &lt;父 commit ID&gt;</code>命令来修改。这个命令在实际开发中使用频率比较高，我们一定要掌握。具体来说，使用它主要分为 4 步。</p><ol>
<li>查看当前分支的日志记录。</li>
</ol><pre><code>$ git log --oneline
1d6289f docs(docs): append test line 'update3' to README.md
a38f808 docs(docs): append test line 'update$i' to README.md
55892fa docs(docs): append test line 'update1' to README.md
89651d4 docs(doc): add README.md
</code></pre><p>可以看到倒数第 3 次提交的 Commit Message 是：<code>docs(docs): append test line 'update$i' to README.md</code>，其中 update$i 正常应该是 update2。</p><ol start="2">
<li>修改倒数第 3 次提交 commit 的 message。</li>
</ol><p>在 Git 仓库下直接执行命令 <code>git rebase -i 55892fa</code>，然后会进入一个交互界面。在交互界面中，修改最近一次的 Commit Message。这里我们使用 reword 或者 r，保留倒数第3次的变更信息，但是修改其 message，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/9e/b9/9e8344df0dd8f66f63a307b5a6487fb9.png?wh=1319*273" alt=""></p><p>修改完成后执行<code>:wq</code> 保存，还会跳转到一个新的交互页面，如下图所示：</p><p><img src="https://static001.geekbang.org/resource/image/0e/24/0ef693d6fc195ebd179b7992e0776f24.png?wh=1392*359" alt=""></p><p>修改完成后执行<code>:wq</code> 保存，退出编辑器之后，会在命令行显示该 commit 的 message 的更新结果：</p><pre><code>[detached HEAD 5a26aa2] docs(docs): append test line 'update2' to README.md
 Date: Fri Sep 18 13:45:54 2020 +0800
 1 file changed, 1 insertion(+)
Successfully rebased and updated refs/heads/master.
</code></pre><p><code>Successfully rebased and updated refs/heads/master.</code>说明 rebase 成功，其实这里完成了两个步骤：更新 message，更新该 commit 的 HEAD 指针。</p><p>注意：这里一定要传入想要变更 Commit Message 的父 commit ID：<code>git rebase -i &lt;父 commit ID&gt;</code>。</p><ol start="3">
<li>查看倒数第 3 次 commit 的 message 是否被更新。</li>
</ol><pre><code>$ git log --oneline
7157e9e docs(docs): append test line 'update3' to README.md
5a26aa2 docs(docs): append test line 'update2' to README.md
55892fa docs(docs): append test line 'update1' to README.md
89651d4 docs(doc): add README.md
</code></pre><p>可以看到，倒数第 3 次 commit 的 message 成功被修改为期望的内容。</p><p>这里有两点需要你注意：</p><ul>
<li>Commit Message 是 commit 数据结构中的一个属性，如果 Commit Message 有变更，则 commit ID 一定会变，<code>git commit --amend</code> 只会变更最近一次的 commit ID，但是 <code>git rebase -i</code> 会变更父 commit ID 之后所有提交的 commit ID。</li>
<li>如果当前分支有未 commit 的代码，需要先执行 <code>git stash</code> 将工作状态进行暂存，当修改完成后再执行 <code>git stash pop</code> 恢复之前的工作状态。</li>
</ul><h2>Commit Message 规范自动化</h2><p>其实，到这里我们也就意识到了一点：Commit Message 规范如果靠文档去约束，就会严重依赖开发者的代码素养，并不能真正保证提交的 commit 是符合规范的。</p><p>那么，有没有一种方式可以确保我们提交的 Commit Message 一定是符合规范的呢？有的，我们可以通过一些工具，来自动化地生成和检查 Commit Message 是否符合规范。</p><p>另外，既然 Commit Message 是规范的，那么我们能不能利用这些规范来实现一些更酷的功能呢？答案是有的，我将可以围绕着 Commit Message 实现的一些自动化功能绘制成了下面一张图。</p><p><img src="https://static001.geekbang.org/resource/image/87/be/87cd05c48ac90ec93c379b568a6006be.png?wh=2141*1219" alt=""></p><p>这些自动化功能可以分为以下 2 类：</p><ul>
<li>Commit Message 生成和检查功能：生成符合 Angular 规范的 Commit Message、Commit Message 提交前检查、历史 Commit Message 检查。</li>
<li>基于 Commit Message 自动生成 CHANGELOG 和 SemVer 的工具。</li>
</ul><p>我们可以通过下面这 5 个工具自动的完成上面的功能：</p><ul>
<li><a href="https://github.com/lintingzhen/commitizen-go">commitizen-go</a>：使你进入交互模式，并根据提示生成 Commit Message，然后提交。</li>
<li>commit-msg：githooks，在 commit-msg 中，指定检查的规则，commit-msg 是个脚本，可以根据需要自己写脚本实现。这门课的 commit-msg 调用了 go-gitlint 来进行检查。</li>
<li><a href="https://github.com/llorllale/go-gitlint">go-gitlint</a>：检查历史提交的 Commit Message 是否符合 Angular 规范，可以将该工具添加在 CI 流程中，确保 Commit Message 都是符合规范的。</li>
<li><a href="https://github.com/arnaud-deprez/gsemver">gsemver</a>：语义化版本自动生成工具。</li>
<li><a href="https://github.com/git-chglog/git-chglog">git-chglog</a>：根据 Commit Message 生成 CHANGELOG。</li>
</ul><p>这些工具你先有个印象就好了，在后面的课程内容中，我会带你通过实际使用来熟悉它们的用法。</p><h2>总结</h2><p>今天我向你介绍了 Commit Message 规范，主要讲了业界使用最多的 Angular 规范。</p><p>Angular 规范中，Commit Message 包含三个部分：Header、Body 和 Footer。Header 对 commit 做了高度概括，Body 部分是对本次 commit 的更详细描述，Footer 部分主要用来说明本次 commit 导致的后果。格式如下：</p><pre><code>&lt;type&gt;[optional scope]: &lt;description&gt;
// 空行
[optional body]
// 空行
[optional footer(s)]
</code></pre><p>另外，我们也需要控制 commit 的提交频率，比如可以在开发完一个功能、修复完一个 bug、下班前提交 commit。</p><p>最后，我们也需要掌握一些常见的提交操作，例如通过 <code>git rebase -i</code> 来合并提交 commit，通过 <code>git commit --amend</code> 或 <code>git rebase -i</code> 来修改 commit message。</p><h2>课后练习</h2><ol>
<li>新建一个 git repository，提交 4 个符合 Angular 规范的 Commit Message，并合并前 2 次提交。</li>
<li>使用 git-chglog 工具来生成 CHANGEOG，使用 gsemver 工具来生成语义化版本号。</li>
</ol><p>期待在留言区看到你的思考和答案，也欢迎和我一起探讨关于规范设计的问题，我们下一讲见！</p>
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
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/DYAIOgq83eoib6BjEV4KPEaIdlLEfoVFRCxCSlL2XaIVDiaakvjhWEibibym323ZeHXAY46JMO3nSHmjiaWtAY47eww/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>dobby</span>
  </div>
  <div class="_2_QraFYR_0">这一篇写得就很实用了，在小厂的几乎都不用这些规范。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-03 09:22:05</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/28/84/b4/3ed5adc7.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>pudu168</span>
  </div>
  <div class="_2_QraFYR_0">推荐一个网站： https:&#47;&#47;www.conventionalcommits.org&#47;en&#47;v1.0.0-beta.4&#47;  有中文版，之前一直使用的规范文档</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 本讲，Commit message基于这个优化的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-10 15:40:54</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTKotsBr2icbYNYlRSlicGUD1H7lulSTQUAiclsEz9gnG5kCW9qeDwdYtlRMXic3V6sj9UrfKLPJnQojag/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>ppd0705</span>
  </div>
  <div class="_2_QraFYR_0">好细致的规范 赞</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-03 12:29:31</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/52/40/e57a736e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>pedro</span>
  </div>
  <div class="_2_QraFYR_0">每次commit和rebase的时候才发现英语和语文是多么的重要，哎~</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-03 08:10:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/11/7a/d2/4ba67c0c.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Sch0ng</span>
  </div>
  <div class="_2_QraFYR_0">这篇文章是迄今为止见过的最详细的commit规范。<br>平时只用git commit -m，实在没想到commit还有这么多讲究。<br>可见精益求精不止会体现在代码上，也可以体现在配套的生态中。<br>作者这种精益求精的态度值得钦佩。<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-04 00:44:07</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/d7/f1/ce10759d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>wei 丶</span>
  </div>
  <div class="_2_QraFYR_0">老师是不是rebase后push会出现 当前分支的最新提交落后于其对应的远程分支 <br>我每次操作都有这个想要push必须要-f才可以 是我的操作的问题么？？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 是的</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-11 15:35:40</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/4e/bf/0f0754aa.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>lianyz</span>
  </div>
  <div class="_2_QraFYR_0">孔老师，一个commit里允许同时包含一个feat和一个fix吗？如何约束开发的同事？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 最好别，可以分两次提交</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-03 13:18:42</div>
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
  <div class="_2_QraFYR_0">做个笔记<br>1 git rebase -i &lt;父commit&gt; 一定是父commit id<br>2 pick 默认不动 s 合并 r 重写</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-28 11:39:59</div>
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
  <div class="_2_QraFYR_0">实战中用起来</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-05 22:01:37</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/1e/f5/0b/73628618.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>兔嘟嘟</span>
  </div>
  <div class="_2_QraFYR_0">git-chglog只需要安装好后git-chglog -o CHANGELOG.md ，就可以一键生成change log<br>gsemver没搞懂怎么用</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: gsemver bump可以生成版本。<br><br>具体哪方面没搞懂呢</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-12-01 18:55:12</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/0f/87/64/3882d90d.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>yandongxiao</span>
  </div>
  <div class="_2_QraFYR_0">总结：<br>    1. 介绍 angular 规范，包括，type，scope，subject，body，footer。<br>    2. 开发手写 angular 规范的 commit message 几乎是不可能的，需要工具来帮助。<br>    3. commitzen-go 以交互的方式，帮助程序员生成 angular 规范的 commit message<br>    4. 通过 git hooks, 检查 commit message 规范，不符合时，git commit 执行失败<br>    5. 压缩 commit message 、修改 commit message ，都可以借助 git rebase -i 命令<br>    6. 基于 commit message, 使用 git-chglog 自动生成 CHANGELOG<br>    7. gsemver 自动生成语义化版本号，感觉和 angular message 规范，关系不大。</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-11-21 20:13:36</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/11/74/72ffa6d6.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>StayLet</span>
  </div>
  <div class="_2_QraFYR_0">孔老师，为啥 “subject 的结尾不能加英文句号”？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 这个是angular规范规定的。<br><br>我理解，加英文句号首先作用不大。再有，有些工具会根据commit发送邮件，subject是邮件标题，邮件标题一般是不带英文句号的。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-30 10:09:58</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/13/4b/11/d7e08b5b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>dll</span>
  </div>
  <div class="_2_QraFYR_0">真的干货满满的 一下子看明白rebase 我以前还学的迷迷糊糊的<br></div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-17 01:13:55</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/16/d7/39/6698b6a9.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Hector</span>
  </div>
  <div class="_2_QraFYR_0">这个好，周末分享会有东西了</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-15 08:23:02</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/14/60/5f/32d504bb.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>🌆🌇🌉🌌</span>
  </div>
  <div class="_2_QraFYR_0">老师，这个流程和git-flow哪个更好呢？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 跟gitflow一样的流程</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-06-03 22:18:23</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/18/75/bc/e24e181e.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Calvin</span>
  </div>
  <div class="_2_QraFYR_0">提交push了之后是不是不能修改了？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 还能修改。<br><br>修改后可以通过git push -f更新远端代码</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2022-02-22 14:46:48</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://thirdwx.qlogo.cn/mmopen/vi_32/Q0j4TwGTfTJyibojtJCnzAE7E8sMqgiaiaAHl3FuzcXcicQnjnT5huUFMxGUMzV5NGuqzzHHr8dBzCs3xfuhwcOnPw/132"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>好家庭</span>
  </div>
  <div class="_2_QraFYR_0">请问修改了commit message之后所有的commit id改变，如果修改的是仓库中原有的commit,再提交到仓库会不会产生冲突？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 会的，可以push -f</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-09-14 17:26:26</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/21/5f/f8/1d16434b.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>陈麒文</span>
  </div>
  <div class="_2_QraFYR_0">大厂规范就是给力，向你们看齐</div>
  <div class="_10o3OAxT_0">
    
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-08-15 11:53:27</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/1c/cd/8d552516.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Gojustforfun</span>
  </div>
  <div class="_2_QraFYR_0">commit-msg工具的github链接缺失了，不知道是不是这个https:&#47;&#47;github.com&#47;JayceChant&#47;commit-msg</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: 不是的哈，这里的commit-msg其实指的是git的Hook机制。不是工具，所以不需要连接。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-23 20:37:56</div>
  </div>
</div>
</div>
</li>
<li>
<div class="_2sjJGcOH_0"><img src="https://static001.geekbang.org/account/avatar/00/12/1c/cd/8d552516.jpg"
  class="_3FLYR4bF_0">
<div class="_36ChpWj4_0">
  <div class="_2zFoi7sd_0"><span>Gojustforfun</span>
  </div>
  <div class="_2_QraFYR_0">“在以上规范中，必须用括号 () 括起来......”<br><br>哪部份“必须用括号 () 括起来”？</div>
  <div class="_10o3OAxT_0">
    <p class="_3KxQPN3V_0">作者回复: scope，我让编辑把这个地方补充下，感谢反馈。</p>
  </div>
  <div class="_3klNVc4Z_0">
    <div class="_3Hkula0k_0">2021-07-23 18:26:37</div>
  </div>
</div>
</div>
</li>
</ul>