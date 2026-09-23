https://learn.microsoft.com/en-us/agent-framework/get-started/workflows?pivots=programming-language-python

# Workflow, executors and edges
Define workflow steps (executors) and connect them with edges:

Python
 Step 1: A class-based executor that converts text to uppercase
 
class UpperCase(Executor): //我要创建一个叫 UpperCase 的东西，而且它属于 Executor 这一类工作步骤。注意，这个括号是类的继承，所以这里UpperCase也是一个executor
    def __init__(self, id: str): //创建这个对象的时候，自动运行这里
        super().__init__(id=id) //反正意思就是，创建的时候需要给它一个 string 类型的数据作为它的 ID; 哦，以及这个 super 是因为刚才说的，它是继承的，所以得去找它的“爸爸”，也就是 Executor 这个类，然后用 Executor 里面的这个 initiate 给它拿过来用，即调用 Executor 原本的初始化功能，并且把这个 id 给它。

    @handler //@是decorator装饰器，@handler = 告诉 Workflow：下面这个函数是用来处理输入消息的
    async def to_upper_case(self, text: str, **ctx**: WorkflowContext[str]) -> None: //这个to_upper_case函数就是 UpperCase 收到数据以后要执行的函数。；ctx = context, 这里跟那个 agent 那边 context 的那个没有任何关系，它这里是一个更原初的用法，就是当前的 executor 和 workflow 系统沟通的接口， 就是说这个 UpperCase 做完事情之后，去告诉 workflow 说它处理完了，然后该走下一步了。哦，此外，这里的返回值是 None，也就是没有返回值。那是因为它下边的代码里边已经用了 ctx.send_message，去把结果传给 workflow 了，所以这边用不上返回值了。
        """Convert input to uppercase and forward to the next node."""
        await **ctx**.send_message**(text.upper()) //好，一点点说。那个 text.upper()，这个是 Python 自带的，就是把小写都改成大写，就这么一个意思； ctx.send_message(...) 这个意思是：把这个结果发送给 workflow 的下一步 executor
//总之就是这么一整个 uppercase 的类，它所做的事情就是收到文字，然后转大写，再发送给下一步。虽然说它是个类，但因为这边用了 handler 装饰器，所以当它创建的时候，自然就会运行 to_upper_case 这么一个函数。然后它会顺手告诉这个 workflow 自己已经干完了，再交给下一步的 executor。

 Step 2: A function-based executor that reverses the string and yields output
 
@executor(id="reverse_text") //每一个步骤都叫一个 executor， @executor指的是把下面这个普通 Python 函数直接变成一个 Workflow executor。两种写法都行。上一个写法是用一个 class 当一个 executor，然后这边就直接拿一个 function 当 executor。怎么写都行，无所谓的。以及顺手说，这个 executor 的 ID 就是 reverse_text
async def reverse_text(text: str, ctx: WorkflowContext[Never, str]) -> None:
// ctx: WorkflowContext[Never, str], Never = 不会继续发送给下一个 executor,因为这里就是最后一步了，它没有别的步骤了, str   = 最终输出是字符串
    """Reverse the string and yield the final workflow output."""
    await **ctx.yield_output**(text[::-1]) // text[::-1] = 把字符串倒过来ABC->CBA，是python自带的。
//send_message = 给下一个 executor ;  yield_output = 给整个 Workflow 最终结果。


def create_workflow():
    """Build the workflow: UpperCase → reverse_text."""
    upper = UpperCase(id="upper_case") //这个是因为大写的 UpperCase 它是一个类，然后它这个 ID 传过去，反正多了一层。跟 function-based 的就这么点差别：1. 因为它是个类，所以它这个 ID 得带括号传进去。2. function-based 那边可以直接去写，在 @executor(id) 里面写就行。只是因为这个语法的区别而已。 这里创建了一个类，类传进去的 ID 叫 UpperCase，但是你用这个 object 的时候，还是得用 upper 这个名字。
    return WorkflowBuilder(start_executor=upper).add_edge(upper, reverse_text).build()
//好的，这个 return 这么一长串，首先讲了三件事情：

第一件事情是用了 WorkflowBuilder，括号里 startExecution 等于 upper，意思就是我要搭一个 workflow，而且第一步要从 upper 这个 object 开始
第二件事就是在 upper 和 reverse text 中间连了一个 edge（edge 就是两个 executor 中间连接的桥梁）。所以现在可以这么想：就是把 upper 这个 executor 和 reverse text 这两个 executor 给连起来了。而 reverse text 是刚才我们写的那个函数，它本身就是函数，所以直接就是一个 executor 了，这就不多解释了
好，第三件事就是关于 build 的。我前面已经说完了，你要是还有其他的 executor，就在前面一直连着，挨个连下去，就是这么回事。但是如果加了 build()，就意味着到这里就结束了，把它给造出来吧。


---
Build and run the workflow:

workflow = create_workflow() //可能给混在里边了，不太容易看到。但是 create_workflow 它本身也是一个函数，所以这边得调用一下，然后这个的 workflow 才是真的 workflow

events = await workflow.run("hello world") //哦，然后这边才是真的跑起来了；而且已经把 "hello world" 这个 string 给输入进去了; await workflow.run(...) 不是“神奇地把里面所有 async 自动 await 一遍”，而是 workflow.run() 自己内部负责把那些 async executor 一个个调用、等待、串起来。你外面只需要等整个 workflow 完成，所以这边其实里面是有两个 async 的，但我只用了一次 await，就等它每一个都跑完吧。

用events因为 Workflow 在运行过程中会产生很多“事件”。例如概念上可能有：

workflow started
upper started
upper produced message
reverse_text started
reverse_text produced output
workflow finished

框架把整个运行过程中发生的东西收集起来。
print(f"Output: {events.get_outputs()}")
print(f"Final state: {events.get_final_state()}")

---

# harness in Microsoft Agent Framework

from agent_framework import create_harness_agent
from agent_framework.openai import OpenAIChatClient //agent_framework 是 Microsoft 的 Agent Framework，不是 OpenAI 的 SDK。 这里的 OpenAIChatClient 只是 Microsoft Agent Framework 提供的一个“连接 OpenAI 模型的客户端适配器”。

**agent = create_harness_agent(
    OpenAIChatClient(model="gpt-4o"),
)
** //就是这边直接在 agent platform 的 OpenAI provider（尤其是 OpenAI Chat Client）里把事情都包圆了，所有的 harness 其实都在里面，意思就是不用自己写了
// A session carries the harness state (plan, todos, history) across turns.
session = agent.create_session()

print("Harness agent ready. Type 'exit' to quit.")
while True:
    user_input = input("> ") //input() 是 Python 自带的函数，作用是：在终端里等用户输入文字。>是提示文字。
    if user_input.strip().lower() in {"exit", "quit"}:
        break

    # Stream this turn's output as the harness plans and works through the request.
    async for chunk in agent.run(user_input, session=session, stream=True): //agent.run(...) 意思就是：让这个 Agent 处理一次用户输入。这里session是刚才session = agent.create_session() 创建的这个session，自带连续性的。 stream=True它的意思是：不要等 Agent 全部生成完以后一次性给我结果，而是边生成边给我。这是一个异步的数据流；每当有新的一块数据出来，就拿一chunk
        if chunk.text: //有些 chunk 不一定包含文字。它可能代表别的事件，比如状态变化、tool call、metadata 等。if chunk.text: 意思就是：如果这一小块里面真的有文字，那我才打印。
            print(chunk.text, end="", flush=True) //end=""意思是：打印完以后不要换行; flush=True “我刚打印的这一小块，马上显示到屏幕上，不要先攒着。” 有时候程序为了效率，会把输出先放到 buffer 里，等攒多一点再显示。但 streaming 最怕这个。
    print() // 最后这个是在 item for 解释之后运行的。因为前面它一直都没有换行，end""就是里面是空的，所以放这么一句的目的是让它换个行
