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

  res_1, res_2, res_3 = await asyncio.gather( //gather 会等 A、B、C 全部完成 → 然后一次性返回所有结果
            Runner.run(
                spanish_agent,
                msg,
            ),
            Runner.run(
                spanish_agent,
                msg,
            ),
            Runner.run(
                spanish_agent,
                msg,
            ),
        )

        
//这边就是这三种。它们都是同一个 agent，但是开了三个 session，输出了三个 output。最后再用一个 translation picker 去挑一个好的，也就是跑另外一个 agent 的一个 session 再筛选一次。就这么简单，就这样。
多个 Agent 不互相依赖 → 同时运行 → 等全部完成 → 聚合结果

也就是 Parallel / Fan-out / Fan-in。

Input → A + B + C = fan-out

A + B + C → Results = fan-in
OpenAI 官方的 agent_patterns README 也把它列为一个独立常见模式：多个 agent 可以并行运行，既可以降低 latency，也可以同时生成多个候选结果再挑一个。官方 orchestration 文档明确用 asyncio.gather 作为 code-driven parallel orchestration 的例子。

# LLM as a judge

https://github.com/openai/openai-agents-python/blob/main/examples/agent_patterns/llm_as_a_judge.py

排除掉 Python 语法的用法，其实都是非常简单的逻辑。

核心就是让每一轮的 Evaluation Agent 输出一个 structural response，里面分两块：

1. score
2. 一个比较具体的评价

只有当 score 是 pass 的时候，才会认定通过并继续往下走。这里其实没有下一个 story，只有 outline 和 evaluator。

逻辑非常简单：限定一下最大轮数（max turn），while 循环不能无限进行下去，不然 token 受不了。限定之后，用代码做个简单的判断，来判定这个 while 循环是要继续还是 break 掉。

除了里面用到的各种 Python 语法（对不起，是我菜），整个逻辑非常简单， 是个人就能想到的限定。

from __future__ import annotations //这个是 Python 自己的东西，跟 Agent 没关系。你现在先记成：让 Python 在处理 type hint 的时候更灵活。

例如后面这些：
list[TResponseInputItem]
str | None


都属于 type hint。
这行目前不影响你理解 Agent orchestration，知道它是 Python 类型标注相关设置就够了。

import asyncio
from dataclasses import dataclass //dataclass 可以先理解成：Python 帮你很方便地定义一个“主要负责装数据”的 class。
from typing import Literal //Literal 是 Python type hint。后面会出现：Literal["pass", "needs_improvement", "fail"] 意思是：这个值只能是这三个字符串之一。不是随便什么 str 都可以。

from agents import Agent, ItemHelpers, Runner, TResponseInputItem, trace
from examples.auto_mode import input_with_fallback, is_auto_mode //input_with_fallback就是：用户有输入 → 用用户输入，用户没输入 → 用默认值；is_auto_mode() 你可以先理解成：这个 example 现在是不是运行在 OpenAI examples 自己的自动测试模式里。这不是 Agent orchestration 的核心概念。

"""
This example shows the LLM as a judge pattern. The first agent generates an outline for a story.
The second agent judges the outline and provides feedback. We loop until the judge is satisfied
with the outline.
"""

story_outline_generator = Agent(
    name="story_outline_generator",
    instructions=(
        "You generate a very short story outline based on the user's input. "
        "If there is any feedback provided, use it to improve the outline."
    ),
)


@dataclass //decorator, 可以先理解成：把 EvaluationFeedback 这个 class 做成一个专门装数据的 class。 
class EvaluationFeedback: // 
    feedback: str // feedback 必须得是一个 string 的格式
    score: Literal["pass", "needs_improvement", "fail"] //这刚才说过了，就是用了 literal。就是说，它这个 score，你不仅得是个 string，你还必须得是这三个值之一。

// Evaluator 最后必须返回两个东西：文字 feedback + 一个固定范围里的 score。

evaluator = Agent[None]( // 现在不用深究这个 [None]。这是 Python generic type 相关的东西。这里大致是在说：这个 Agent 没有额外的 dependency/context 类型需要传进来。

    name="evaluator",
    instructions=(
        "You evaluate a story outline and decide if it's good enough. "
        "If it's not good enough, you provide feedback on what needs to be improved. "
        "Never give it a pass on the first try. After 5 attempts, you can give it a pass if the story outline is good enough - do not go for perfection"
    ),
    output_type=EvaluationFeedback, //Evaluator 不要随便返回一坨自然语言，要按照 EvaluationFeedback 的结构输出。
)


async def main() -> None: // -> None 这是 return type hint。意思：main() 本身不打算 return 一个值。
    msg = input_with_fallback(
        "What kind of story would you like to hear? ",
        "A detective story in space.",
    )
    
    input_items: list[TResponseInputItem] = [{"content": msg, "role": "user"}] //后面里面的{}是一个 Python dictionary；[]是个list;

    // : list[TResponseInputItem]是 Type hint。意思大概就是：input_items 是一个 list，而且里面每一项应该是 Responses API 可以接受的 input item。
    // TResponseInputItem约定于一条合法的模型输入 item 的类型。
    //这一整句就是：创建一个“模型输入消息列表”。

    latest_outline: str | None = None //latest_outline 的格式可以是 string，也可以是 None。然后为什么要加上等于呢？因为一开始它就是什么都没有，所以一开始就是 None
    auto_mode = is_auto_mode()
    max_rounds = 3 if auto_mode else None // conditional expression / ternary expression。常见的 Python 语法就是：如果 auto_mode 是 True 的话，那么就是让这个 max_rows 等于 3；如果 auto_mode 是 False 的话，那么就是让 max_rows 等于 None。 意思就是在普通模式下，让 max runs 等于 null；但如果是在自动测试模式下，就给 max runs 赋一个 3 的值
    rounds = 0

    # We'll run the entire workflow in a single trace
    
    with trace("LLM as a judge"): //然后 With Trace 之前说过，要把整个 flow 记录成一次 Trace
        while True: // 外层循环无限跑，直到碰到里面的 break
            story_outline_result = await Runner.run(
                story_outline_generator,
                input_items, //就是放进去的时候是 input items 这个消息列表，然后第一次的时候就只有user: A detective story in space.
            )

            input_items = story_outline_result.to_input_list() //把刚才这一轮运行的内容变成下一轮还能继续使用的 input list。旧 input + 新生成内容 → 新 input_items
            
            latest_outline = ItemHelpers.text_message_outputs(story_outline_result.new_items)//从 Generator 这一轮新产生的 items 里面，把文本内容拿出来。ItemHelpers.text_message_outputs(...)是SDK提供的拿信息用的。
            print("Story outline generated")

            evaluator_result = await Runner.run(evaluator, input_items) // 把刚才拼接好的input_itmes给evaluator。
            result: EvaluationFeedback = evaluator_result.final_output //这里 result: EvaluationFeedback 只是再给 Python/type checker 一个提示：result 应该是 EvaluationFeedback 类型。Final_Output 就是之前说过了，这是它最后真的出来的那个结果。

            print(f"Evaluator score: {result.score}")

            if result.score == "pass":
                print("Story outline is good enough, exiting.")
                break

            if auto_mode:
                rounds += 1
                if max_rounds is not None and rounds >= max_rounds:
                    print("Auto mode: stopping after limited rounds.")
                    break

            print("Re-running with feedback")

            input_items.append({"content": f"Feedback: {result.feedback}", "role": "user"}) //append 就是在 list 最后再加一个 item， 现在input_items是用户要求 + Generator 第一版 outline + Feedback

    print(f"Final story outline: {latest_outline}")


if __name__ == "__main__":
    asyncio.run(main())

# Checkpoint in LongGraph

这个东西之所以存在，是因为分情况来看，它确实有做 checkpoint 的必要性。checkpoint 本质上就是一个 snapshot。

主要是为了防止一个巨大的 workflow 或者 loop 中途在某一个节点断掉。但与此同时，其他一些 function calling 已经正常跑完了，比如扣费这种对金钱敏感的操作，它已经扣掉了。在发现中间断掉出问题之后，怎么让它接着往下跑？在一些关键服务上，必须在没有重复执行的情况下，再去决定接下来怎么搞。

这其实是必然的一条路。不管是从 token 的成本考虑，还是某些特定的服务机制：比如付钱，它本身就是一次性的操作。哪怕中途断了，哪怕机房烧了，它也不能再去扣第二次费。

这里我指的是像网上买东西这种单次扣费服务，不是指订阅类的扣费。抱歉我这边说得有点太含糊了，大概就是这样
