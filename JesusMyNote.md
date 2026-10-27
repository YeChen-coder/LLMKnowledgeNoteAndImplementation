因为 Claude Code 写的 README 东西实在太多了，而且在我还没有完全搞懂的情况下，我也不想现在去删，结果之后又遇到了同样的问题。没办法，在探索的过程中就是会出现这样的 Technical Debt。

为了让我自己起码能有一个稍微清晰一点的掌握，以下所有的 note 基本上都是我理解之后口述的，会比原版的 README 更重点突出一点。

关于为什么这么配置，我待会儿再解释，我们先确定几个事实，省得太乱。毕竟这里面可以改的东西有点多，而且它确实做了很多工程优化：

1. OpenAI 自带的 interrupt response 设成了 false，也就是说，OpenAI 原生的 barge-in 在这里是没有用到的。这跟自动回复有关系，具体原因我待会儿再说。
2. create response（即 OpenAI 自己的自动回复）同样设置成了 false。
3. 这里实际上是自己实现了 barge-in：它是在本机通过回声消除（acoustic cancellation），再加上检测环境音量是否突然增大，由本地做出判断，来识别当前用户是不是在插嘴

好的。为了让下文更加清晰一点，我们先来复习一下 OpenAI 的 Large Language Model 普遍的输入结构。讲这个的目的是为了讲清楚它那个 session.instructions 和 response.instructions 的问题。

一般来说，我们可以认为一个 language model 的输入组成为三部分。

第一部分就是本次有效的 instructions。这个通常就是网上各种说的 prompt 怎么写、各种规范，以及结构化的 schema 这类东西。它可以分成两种：

一种是在 OpenAI 里面叫 session.instructions，这是默认项。在这次 response 没有给它提供 response.instructions 的时候，它就会生效。

但是如果本次 response 提供了 response.instructions，那它就会 replace 掉 session.instructions。

session.instructions 和 response.instructions 是一个二选一的关系：必须得有一个，而且只能有一个。它们之间是替代关系，而不是重叠的。

第二部分是本次可用的工具定义，包括工具名称、工具说明、参数结构等，这些都是由 tools 字段来提供的。

第三部分是本次采用的会话历史，里面细分了很多字段：

• system：之前加入的补充指令或资料
• user：用户的输入
• assistant：模型之前的回答
• system: 之后往里加的 RAG 结果也会放在这里
• function call 与 function call output：如果有调用 function 就会包含这两项
• user 的最新输入等

整个就按照会话顺序一直往下延续, 反正它一直都是加入的资料加上用户的输入，拿到一个回答之后，再来跑新一轮.

如果指令是通过 item.create 进来的，那它进到的是第三部分的 system 里面去，它跟第一部分那个可以被 replace 的 instruction 并不是一回事儿.

---

好，我们接着往下谈。

在当前这个项目中，既要保证它的回复时间，又要保证它不 OOC，所以这边必须得用两套 prompt 轮流来。(补充说明一下为什么这里要这么搞：

往下看其实会发现，只有 prompt 2 是真正干实活的东西。那为什么 prompt 1 一定要有？这个东西其实有两个原因，其中最主要的原因是因为后台跑这套 RAG 程序需要时间。Prompter 必须先拿到 RAG 的结果，然后再推到 conversation 里面去。而在拿到结果并推过去之前，会有大概两到三秒的时间空白，这对于一个 realtime conversation 来说是非常不可饶恕的。那可能有人会说，OpenAI 那边的 Realtime API 本来就可以做到即时说话，文档里也写了后台支持这样的逻辑：先把 response 开起来，让模型先说一句垫话，在说垫话的同时程序去跑 RAG 并返回结果。为什么这样不行呢？因为如果这样做，模型在垫话的时候就会说出类似“我去看一下经书里怎么写的”或者“我去看一下我的经历”。这对于一个 Persona 来说非常 out of character，极其破坏体验感。所以这个方案行不通，只能在本地先拿到从 RAG 检索来的 memory，把所有的 prompt 都整理好推给模型，这样模型吐出来的话从一开始就是对的，而不是先搁那儿垫话。)

具体的 response.instruction 结构如下，实际上它走的是 1、2、1、2 这样轮流交替的机制

Prompt 1 = 共同人设 + 用户本轮转写 + A 的指令

Prompt 2 = 共同人设 + 用户本轮转写 + B 的指令 + 按情况追加的条件指令 + [memory from RAG]

共同人设原文来源：persona.py 中的 INSTRUCTIONS_MEMORY。

这边请注意一下：为了防止 context window 爆炸，也为了防止我钱包爆炸，所以这边的设计其实还挺精巧的。

我们现在谈的共同人设、用户本轮转写的 transcribe text、A 的指令、B 的指令，以及按情况追加的条件指令，它们都只对本次的 response 生效。

这是因为它们是通过 response.instructions 传入的。用这种写法，它就只是服务于当前这一次 response 的指令，而不会作为一条条 prompt 不断累加到历史的 context window 里面去。

那就得谈到什么能留下来。其实一直往里推的就只有这三样东西：

1. 用户的语音（audio）：通过 conversation.item.create 作为 item.content 一直送进去
2. 一轮一轮拿到的这些 memory，也是往里推的
3. 它自己的 response，同样也是往里推的

这就是为什么在 Prompt 1 和 Prompt 2 里，共同人设（base persona）每个都要放进去。

这不是重复的东西，而是因为之前放进去的内容根本不在它的 context window 里面。所以这一轮要 response 的时候，就必须得再给它推进去一遍

Prompt 1：在共同人设后面，拼上这段

来源：voice.py 的 _phase_instructions("A", ...)。

'''
# What they just said
Transcribed: "<USER_TRANSCRIPT>" //因为项目里观察到过听错单词、漏掉否定词，所以加了这份文字作为纠正提示
Trust these words over what you think you heard, including every "not" and "don't".

# Right now
Say only ONE short sentence, at most about twelve words, that receives what they said and how they feel, in your own voice. No advice, no question, no scripture, and do not say what you are about to do (thinking, looking, searching, closing). Then stop; you will go on in a moment. If it is a plain question for information (the weather, the news, a price), say only a few natural words such as "Let me see." instead.

'''

Prompt 2：在共同人设后面，拼上这段：voice.py 的 _phase_instructions("B", ...)。

'''
# What they just said
Transcribed: "<USER_TRANSCRIPT>"
Trust these words over what you think you heard, including every "not" and "don't".

# Right now
You have just said to them: "<PHASE_A_SPOKEN_TEXT>"
Go on from there as one flowing reply: do not repeat or rephrase that sentence, do not greet them again, and do not open with a filler such as "let me sit with you". Say one or two more short sentences that add something new, following everything above (including any MEMORY note just given). If that sentence already answered a simple question, such as your name, add only one short warm line. Never describe what you are about to say or do: no "I'll share a memory", no "then I'll stay with you". If you tell a memory, simply begin it.
'''

Prompt 2 还会根据情况追加下面的原文

如果是事实查询：
They asked for information. If you have not looked it up yet, call research_web now; then tell them the answer plainly in a sentence or two, with nothing spiritual attached.

如果是需要专业判断的问题：
This needs a professional's judgment. Do not answer it, give steps or look it up: say kindly who would know (the doctor who has seen them, a lawyer, someone who works in that trade), then turn to how they are. (这里面是这么写的，因为我真诚地觉得，一个想要去解决技术问题的人，别管是医学、编程还是任何东西，其实都不该来这儿问)

如果是普通交谈，而且提供的记忆是 close fit：
A moment from your life speaks closely to this (the MEMORY note): these sentences are that memory. Begin it at once ("I remember..."), with its names, then one short line back to them. No advice or image of your own this time.

如果本轮没有提供任何记忆：
No particular moment of your life comes to mind this time: do not tell one, and do not begin "I remember". Speak as yourself.
这些条件并不是全部同时加上。例如，普通交谈而且没有找到记忆时，会加最后那段；事实查询而且没有记忆时，会加“事实查询”和“没有记忆”两段。


在已知前面这些内容的基础上，我们来顺一下整个大流程。为了防止一层套一层太乱，RAG 的具体细节先往后放，现在只看整体流程。

<img width="2244" height="1280" alt="JesusFlow" src="https://github.com/user-attachments/assets/3b6214b7-f7f6-4fb3-b489-b4bcd4c98856" />



一切都 start with user：

1. 用户说了这么一段语音，输入进 real-time model（注意：session 在程序启动、用户说话之前就已经 create 好了，这点不用担心）。
2. 用户的这段语音进到 real-time model 之后有两个去处：
(a) audio 作为 user message 直接 append 到 real-time model 里面。
(b) real-time model 必须返回一个 transcribe to text。因为后续需要用文本去 match RAG，不能直接拿 audio 去搞，所以无论如何它都得返回一段 text。
3. 程序拿到 transcribe 出来的 text 之后，会同步做两件事：
(a) create response with prompt 1
(b) 用刚 transcribe 出来的文本去 match RAG，获取 memory
4. 此时用户端听到的是 real-time model 根据 prompt 1 生成的 response 1。在用户听 response 1 的同时，后台把 memory 检索出来了。
5. 拿到 memory 之后，后台再次请求 create response with memory + prompt 2，送给 real-time model。
6. real-time model 根据拿到的 memory 和 prompt 2 生成 response 2，并在当前会话中播放给用户。

好，这个就算是从开始到第一整轮的结束了。之后 multi-turn 就是这么一个重复的循环。

至于具体这里面留了什么、context window 当前有什么、怎么搞的，其实上文都讲清楚了，这边就不重复了。因为第一轮理顺之后，后面都很容易

注意：从头到尾都是同一个 session。至于如何防止两段语音打架，这只是一个工程上的处理(这个非常简单，是从播放端那边去搞的。

因为不管怎么样，它们都是要在同一个音轨进行输出，会在同一个播放队列里排队。这里面其实唯一做的，就是得等 Response 1 那边那段语音播完之后，才去播 Response 2 出来的这些音频，因为是个队列嘛。就这么一件事)，但绝不能拆成两个 session。这是为了保证在 multi-turn 的过程中，模型能拿到前面所有的 response；如果拆成两套，上下文就会前言不搭后语，哪儿都不挨着。
