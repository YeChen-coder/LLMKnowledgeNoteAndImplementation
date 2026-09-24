Note of https://openai.github.io/openai-agents-python/quickstart/?utm_source=chatgpt.com

# async 异步 + await 

用 asyncio 启动一个异步的 main() 函数，在里面等待 Runner.run(...) 执行完成，然后打印最终结果。(async/await 的一个核心意义是：等待 I/O 的时候，可以把执行权让给其他异步任务。)
import asyncio
from agents import Agent, Runner

agent = Agent(
    name="History Tutor",
    instructions="You answer history questions clearly and concisely.",
)
async def main(): //async def 表示定义一个“异步函数”，也叫 coroutine function（协程函数）。用了async并不会像普通函数一样顺序的立即把里面所有代码执行完，而是得到一个 coroutine /ˌkəuru:'ti:n/对象。通常需要通过：**await main()** 或者：**asyncio.run(main())** 来真正运行它。async 通常就是 asynchronous /ei'siŋkrənəs/（异步）的缩写
    result = **await** Runner.run(agent, "When did the Roman Empire fall?")  //await 可以理解成：等这个异步操作完成，然后拿到它返回的结果。启动 Runner.run(...)-> 这个操作可能需要一段时间-> 等待结果-> 结果回来以后-> 赋值给 result; 假设 Runner.run() 是在调用 AI Agent，那么它可能需要：发送请求→ 等服务器响应→ Agent 处理→ 返回结果,网络请求可能需要几百毫秒甚至几秒。如果完全使用同步程序，那么“等待网络”的这段时间 Python 线程通常只能干等。这里面其实是异步套异步。
    
    print(result.final_output)

if __name__ == "__main__":
    **asyncio.run(main())**

<img width="760" height="307" alt="image" src="https://github.com/user-attachments/assets/0b533756-0c9b-4d17-ba0f-c2eef229a7f7" />

第一个好说，就是最原始的那种，什么都没有，纯手工拼。

第二个是 SDK 里的 session，把之前的记录放到 session 里。其实它仍然是让开发者自己去控制会话的一套机制，可以做的事情很多：里面可以接数据库，可以放各种自定义内容，甚至用的模型都不一定是 OpenAI。Agent SDK 之所以这么设计，核心就是两个目的：

1. 本着不应该把 conversation memory 和 OpenAI Response API 那边的 server state 绑死，让开发者可以在 history 里接数据库，自己灵活修改内容
2. 开发者用的根本就不是 OpenAI 的 API，自然就不可能用第三种方式的 previous_response_id 或者 conversation_id

第三条：
• previous_response_id 是最无脑的一种，跟着上面走就行
• conversation_id 则是一个很固定的东西，可以跨窗口去用

不过这块我实在还没太搞懂，先继续往下看吧，后面看懂了我再回来补。

加tools:


@tool
def history_fun_fact() -> str:
    """Return a short history fact."""
    return "Sharks are older than trees."

## Agent orchestration 
hands off 它和 agents as tools 这两个是同等价位的关系。当然也可以组合使用：首先由一个 triage agent 接收，把所有的 task 都 hand off 给一个 specialist，然后这个 specialist 再去 call 一群小 agent，用 agents as tools 来处理。

hands off 它是全给了，就是由下一任 take full ownership。而 agents as tools 仍然是有一个大领班从头到尾去负责，agent 只是在中间处理某些小事，交由它去办。

此外，还有更容易想到的方案：

如果非不乐意用 handoff，或者代码、应用程序确实有特殊情况，那就直接用代码写：

• 一种是走 structured output：上一个返回这个的 large model，只要返回了 structured output，就能从 JSON 里面解析出 parameter 或者其他东西，你别管具体是什么，反正肯定能搞出来。

• 另一种对于很简单的情况：流程就是 1234567 必须顺着走的，那就一个一个来顺着走，Python 代码从上往下执行就行了。

既然都谈到这儿了，那普通的 while loop 什么的都能干。

然后有的时候可能要搞并发，很多 Agent 都要同时跑， 用asyncio.gather 收结果。你别管是因为什么，反正等到具体的应用场景里自然就会用得上，都可以，就是单纯的一个代码逻辑设计，其实跟 Agent SDK 没有什么太大关系，把它当成一个通用的函数去理解就行了。
 
### handoff

先谈 handoff。因为它的 sub-agent 都是一个并行的关系，但在你分给 sub-agent 之前，总得有人来分发，所以这边引入了一个 triage agent。

在 triage agent 的 handoffs[] 里面，得把所有 agent 的名字都列出来，这样它才能知道把这个东西分给谁。
triage_agent = Agent(
    name="Triage Agent",
    instructions="Route each homework question to the right specialist.",
    **handoffs=[history_tutor_agent, math_tutor_agent],**
)

from agents import Agent

history_tutor_agent = Agent(
    name="History Tutor",
    **handoff_description="Specialist agent for historical questions",**
    instructions="You answer history questions clearly and concisely.",
)

math_tutor_agent = Agent(
    name="Math Tutor",
    **handoff_description="Specialist agent for math questions",** //handoff_description gives the routing agent extra context about when to delegate.这个 handoff description 是可以被 triage_agent 给看到的，只要它在那个 handoffs 的那个数组里。
    instructions="You explain math step by step and include worked examples.",
)

---
具体用的时候，因为之前这些 triage_agent、history_tutor_agent、math_tutor_agent 都已经定义过了，所以在 main 文件里真正调用的时候，就不用再把它们重写一遍，跟之前一样直接 Runner.run，传 triage_agent 和 message 就行。

triage_agent 自己拿到这个 message 之后，会自动根据它自己的 instruction，再加上收到的 handoff description 去做分发。所以这里面的东西它会自动跑、自动包圆。最后拿到的结果就跟之前一样，就拿到了一个 result.final_output。然后这个就是被 route 到的那个 agent 的返回结果。

import asyncio
from agents import Runner


async def main():
    result = await Runner.run(
        triage_agent,
        "Who was the first president of the United States?",
    )
    print(result.final_output)
    print(f"Answered by: {result.last_agent.name}")


if __name__ == "__main__":
    asyncio.run(main())

# a deterministic flow
https://github.com/openai/openai-agents-python/blob/main/examples/agent_patterns/deterministic.py?utm_source=chatgpt.com

这里面先定义了三个 agent：

1. 第一个是写 outline
2. 第二个是检查 outline
3. 第三个是根据 outline 写这个 story

定义 agent 这件事非常简单，非常的直截了当

class OutlineCheckerOutput(BaseModel): //BaseModel来自 Pydantic。你目前可以非常粗暴地先记成：BaseModel = 一个专门用来描述和检查“结构化数据长什么样”的 Python 基类。这里意思是我现在定义一个结构化数据类型。
    good_quality: bool
    is_scifi: bool


outline_checker_agent = Agent(
    name="outline_checker_agent",
    instructions="Read the given story outline, and judge the quality. Also, determine if it is a scifi story.",
    output_type=OutlineCheckerOutput, //output_type=OutlineCheckerOutput 也是 SDK 正式支持的写法。官方文档明确说明：output_type 可以直接传一个 Pydantic BaseModel 类；传进去之后，Agent 会使用 structured outputs，而不是普通字符串输出。
)

这里：output_type=OutlineCheckerOutput

不是说：“模型，给你一个 Python class，你自己研究研究这个 class 是什么意思。” 不是。

Agents SDK 会读取这个 class。因为这是一个 Pydantic model，所以 SDK 可以把它转换成类似这样的 JSON Schema：

{
  "type": "object",
  "properties": {
    "good_quality": {
      "type": "boolean"
    },
    "is_scifi": {
      "type": "boolean"
    }
  },
  "required": [
    "good_quality",
    "is_scifi"
  ]
}

意思就是最终答案必须是一个 object，里面必须有：good_quality → boolean + is_scifi → boolean。
Agents SDK 的源码里确实就是这么干的：它用 Pydantic 的 TypeAdapter 从 output_type 生成 JSON Schema，并负责验证/解析模型返回的 JSON。

主函数是：

async def main():
    input_prompt = input_with_fallback( // input_with_fallback 它只是这个 example 自己写的辅助函数。询问用户输入 → 如果用户真的输入了内容，就用用户输入 → 如果用户没输入，就用默认内容
        "What kind of story do you want? ",
        "Write a short sci-fi story.",
    )

    // Ensure the entire workflow is a single trace
    with trace("Deterministic story flow"): // with 在某个特殊的“环境”里面运行下面这一整块代码。eg 进入 something 管理的环境 → 执行 A → 执行 B → 执行 C → 离开这个环境; 这里意思就是：从这里开始，把下面这整个流程记录成一个叫 "Deterministic story flow" 的 trace。你可以把 trace 暂时理解成：一次 Agent workflow 的运行记录。        # 1. Generate an outline
        **outline_result** = await Runner.run(
            story_outline_agent,
            input_prompt,
        ) 
        print("Outline generated")

        // 2. Check the outline
        outline_checker_result = await Runner.run(
            outline_checker_agent,
            **outline_result.final_output,** //因为上面Runner.run() 返回的不只是那段文字。它返回一个 result object。把第一个 Agent 生成的 outline，交给第二个 Agent 检查。
        )

        //outline_result = 整个 run 的结果对象； outline_result.final_output = Agent 最终输出

        // 3. Add a gate to stop if the outline is not good quality or not a scifi story； 就是单纯的判断两个条件满不满足，不满足直接退，根本不会往story agent推
        assert isinstance(outline_checker_result.final_output, OutlineCheckerOutput)
        //is instance 意思就是：某个东西是不是某个 class 的实例？， 比如“hello” 是 str的instance ， 所以isinstance(x, str) 是true; 这里再问 outline_checker_result.final_output 是不是一个 OutlineCheckerOutput 对象，即做类型检查。 
        //assert something 你可以先记成：我断言 something 必须是真的。
        //总之 我现在确认一下：checker agent 的最终输出必须真的是 OutlineCheckerOutput 类型。如果不是，就说明出现了我们没预期到的情况，直接报错。
        if not outline_checker_result.final_output.good_quality:
            print("Outline is not good quality, so we stop here.")
            exit(0)

        if not outline_checker_result.final_output.is_scifi:
            print("Outline is not a scifi story, so we stop here.")
            exit(0)

        print("Outline is good quality and a scifi story, so we continue to write the story.")

        // 4. Write the story
        story_result = await Runner.run(
            story_agent,
            outline_result.final_output,
        )
        print(f"Story: {story_result.final_output}")

if __name__ == "__main__":
    asyncio.run(main())



# parallelization

https://github.com/openai/openai-agents-python/blob/main/examples/agent_patterns/parallelization.py?utm_source=chatgpt.com

多个 Agent 不互相依赖 → 同时运行 → 等全部完成 → 聚合结果

也就是 Parallel / Fan-out / Fan-in。

OpenAI 官方的 agent_patterns README 也把它列为一个独立常见模式：多个 agent 可以并行运行，既可以降低 latency，也可以同时生成多个候选结果再挑一个。官方 orchestration 文档明确用 asyncio.gather 作为 code-driven parallel orchestration 的例子。
