好吧，来谈吧。

因为它确实不是一个有代码的东西，而是一个 workflow，也不太好放进 repository 里，就直接搁这儿吐槽吧，破事可真是一堆一堆的。

先说一下当前整体的框架：目前是用到了 Grokbot 和 Claude Code 去做前期的 data collection。
• 为什么一定要用 Grokbot？因为它要盯 Indeed 和 LinkedIn 上新出来的岗位。
• Claude Code 则是从一些其他小网站抓取岗位。

所有收集到的 raw data，都要先进到 Claude Code 这边的一个 scout 队列里先筛一遍。只有筛完合格、格式正确的，才会送到 Codex 那边，走一个每 45 分钟跑一次的定时任务。

现在的简历投递，每次投完都会往邮箱发一封投递成功的邮件，所以是拿这个邮件来作为 evidence 的。当然也得把这个邮箱连到 Codex 这边，由 Codex 这个投递者去确认的 task 真的完成了。

这里就涉及谁来监控邮箱的问题(其实只要是找过工作的人都能理解：当投递的量上去了，无用的邮件就开始轰炸。

还有各种简历系统的流氓行为，有的非得让你订阅东西才许你提交。另外 Indeed 也很喜欢发那种“We found a few positions that are a perfect fit for you”的营销短信，看着就很烦，全是一堆无用的东西。

再加上一些拒信，很影响心境。所以这种损耗让人类的心智来承担，还是有点不太合适)。

我没有让 Codex 或者 Claude Code 来接管/监控邮箱，原因很多：

1. Claude Code 这边：
我其实挺想让 Claude Code 帮我监控邮箱的，但一来我 Duet 的 waitlist 还没排到；二来可能是我没找对地方，我找不到登录其他账号邮箱的入口，一登默认就是我 Claude Code 的账号邮箱，而我投递用的并不是这个，点 connect 根本不让我登。
2. Codex 这边：
它确实能连我的其他邮箱，但问题是它的 computer use 太耗 token 了，我的 token 额度真的很吃紧。
3. 为什么不让 Grok 干？
因为 Grok 我家人也在用，我不太想让他们看到这些消息，尽量隔离一下比较好。

所以这事最终交给了免费的 Muse 来干。不过 Muse 也是一堆事：
• 定时任务：如果把定时任务设得频繁一点，不知道为什么，它不会每次都执行，而且不执行也不会通知你。
• 内容读取：它不是全量读取，而是按关键词匹配。最开始如果邮件标题写了 "congratulations" 或者 "next step" 之类的，它读一下就会通知我；后来我让它全量读，它依然不读正文内容，只读整个标题，目前全靠邮件标题来判断。

这个事情也许还能再优化改进一下，但我实在太累了，不想折腾了，之后有点什么事我主动问它吧

出于对上下文的考虑，也为了让我不用每天去 archive 一大堆被 scheduled task 自动创建出来的会话，这边把每开一次 session 的投递上限设成了 3。

这也是被逼出来的：Codex 的 token 太不经用了，最开始官方还暗改 token。其实最开始没这么复杂，非常单纯，全都是 Codex 自己干，自己搜岗位、自己投。但后来因为 usage 限制，就不得不把前期 data collection 的活儿交给其他有富余 token 容量的 AI bot 去干。

其实它这个 filter 也是经过很多次修改的：最开始范围定得太小，结果根本找不到几个 job；后来改了之后范围又不准，搞得太大了；再后来又反复调，才修到一个合理的尺度。

至于为什么一定要走一遍 Claude Code 那边的 SQS 队列，主要是为了帮 Codex 省 token，避免 Codex 再因为网址不准、访问不到或者缺失而去重新找一遍。

只要是经过 Claude Code 进到队列里的，格式就都合格了，可以直接放心地 hand off 给 Codex

其实 Codex 那边还加了一层 layer。

之所以加这层，是因为我注意到grok找岗位的时候，有时没办法把整个 job description 都抓下来，只能看到卡片展示的那一部分；只有在 Claude Code 那边真正点开页面，才能看到完整的 job description。之前的情况是，它把一些要求法语是必需语言、或者需要 8 年工作经历这种纯 senior 的岗位都给我填了，我觉得这没什么意义。

所以我就在 Codex 那边又加了一层过滤：看一下完整的 job description，如果里面提到了我完全做不了的（比如法语、8 年工作经验），就都不要投；但像 3 到 5 年经验的其实都可以投。AI 有时候太较真了，就因为这个较真的问题，才需要把筛选范围稍微放宽一点，适度扩大

另外我还很想吐槽的一点是：现在基本上 95% 的岗位，都要在里面框框填一大堆东西。比如 gender、gender identity、select the race，还有什么 "Are you identified as having a disability?"、"Do you identify as a member of Two-Spirit / Lesbian / LGBT?" 等等。

光上传简历还不行，它解析完八成有问题，你还得再过一遍看它写没写对。接着又是一大堆问题：willing to relocate、what's your expected salary……各种稀里八糟、重复又重复的问题，纯粹白白消耗投递人的精力。

以后咱们能不能搞成：起码每个求职者自己有一个 AI 分身，招聘方自己也有一个 AI 分身，有什么事让两个 AI 自己去互相连线，别再空耗一个人的精力了

反正谁也不要互相磨损了，有啥事请先找我的 AI。我真诚地觉得，这也许是未来社会的一个形态吧。

回到正题，为什么之前一直没写这个，今天突然开始写？因为刚才实在被 Codex 给气到了，it somehow got on my nerves。

刚才在他跑 task 的时候，他内置的浏览器出了几分钟问题。那没问题，我这边还有 Chrome 呢，而且 Codex 自己也有 computer use，我浏览器开着他总能自己登。可是他跟我说他去找了 Edge，Edge 返回说工具不可用，然后他就直接停了。- 吐槽一下，我觉得这周 Codex 那边有些问题。其实这套东西在我这边起码跑满一个月了，之前它基本没有 call 过整台电脑的 computer use，甚至连 Edge、Chrome 都没有调用过，直接用内置的 browser 就搞定了。但这周它就开始疯狂动不动要整台电脑的 computer use

问题是，我的 prompt 里写得清清楚楚：要从队列里拿 3 份进行投递，而且队列里起码还有 10 到 12 份。他在第 1 份碰壁报错之后就停了，我就去查他凭什么停：哪怕第 1 份报错，it always happens, but for the other two, why did you suddenly stop? 

So I asked Codex, and Codex returned me the error response from Edge. And that is where the problem is lying: that error message is sent by Edge, but my prompt is sent as a user. Of course I have more priority, but Codex is taking the message from Edge as a cut signal.

后来我就把这件事写进 Run.md 里了。这个 Run.md 是 Claude Code 和 Codex 共同维护的文件，里面写明了我最新 resume 的位置、个人情况，以及需要动态补充的内容。

另外还有其他问题，真的一堆一堆的。

有些网站会要求自评能力，我刚才亲眼看着 Codex 在某一个投递网站上，把我的 Large Language Model 能力、MLOps、Time Series 这些，统统给我填成了 Practitioner。他们招的是 AI 工程师啊，哪家 HR 看到这样的简历或表单会觉得可以走下一步？说句心里话，我问心无愧，我觉得我调教 AI 的能力还挺可以的

我很生气，完全不知道 Codex 为什么会这么搞。我也不知道之前放手让他在后台定时跑的时候，到底给我整出过多少次这种事，真的很悲伤。

不过说归说，我其实觉得我这一套设计还真的挺精巧的。

因为我现在简历还在一个快速成长期，但我直接在 README.md 里规定好了简历放在哪儿。我现在简历一般都是让 Claude Code 去帮我写，倒不是图它写起来方便，主要是跟它会话的过程中，有些东西它能直接帮我记下来。我不用再专门在那儿哐哐发一大堆，然后再去翻看自己刚才学了点什么。

这点其实还蛮好的，不用每次简历更新之后，我都得在 Codex 那个页面上再重新上传一次。不然如果不把这个东西解耦的话，做起来会很烦

上述是今天晚上刚遇到的 Codex 这边的问题。

再回忆一下，Grok-a-bot 那边问题也少不了。我这里谈的就是验证码的问题：这个 Grok-a-bot 遇到验证码没问题，但它倒是喊我一下、给我发个消息啊，它不发。

它其实是用 subversion 去跑的。它遇到问题之后，就只把当前定时跑的这一组硬干完，然后屁颠屁颠回来跟我说：“今天的第 1 组已经完成了，在 Indeed 上拿到 0 个工作，因为被 Cloudflare 给挡了。”

每次看到这个我都想骂，脑子都疼。你被 Cloudflare 挡了没问题，告诉我一声让我去登一下啊。Grok bot 既有网页端也有手机端，我在哪不能顺手过去登一下？我跟它说过无数次“有事喊我一下”，可它是在用 subagent，没有办法喊我。subagent 和 sub-bot 还是不太一样的：sub-bot 能喊我，subagent不行
