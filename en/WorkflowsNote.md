> English version of [WorkflowsNote.md](../WorkflowsNote.md). Original notes are written in Chinese/English mix.

https://learn.microsoft.com/en-us/agent-framework/get-started/workflows?pivots=programming-language-python

# Workflow, executors and edges
Define workflow steps (executors) and connect them with edges:

Python
 Step 1: A class-based executor that converts text to uppercase
 
class UpperCase(Executor): //I'm creating something called UpperCase, and it belongs to the Executor category of workflow steps. Note: the parentheses mean class inheritance, so UpperCase is also an executor
    def __init__(self, id: str): //runs automatically when this object is created
        super().__init__(id=id) //Basically: when creating it you have to give it a string as its ID. Oh, and the super is because, as just said, it inherits, so it has to go to its "parent," the Executor class, and use Executor's initializer, i.e. call Executor's original init and pass it this id.

    @handler //@ is a decorator; @handler = tells the Workflow: the function below handles incoming messages
    async def to_upper_case(self, text: str, **ctx**: WorkflowContext[str]) -> None: //to_upper_case is the function UpperCase runs after receiving data; ctx = context, which has nothing to do with agent context. It's a more basic usage here: the interface between the current executor and the workflow system, i.e. after UpperCase finishes, it tells the workflow it's done and it's time for the next step. Also, the return value is None, i.e. nothing is returned. That's because the code below already uses ctx.send_message to pass the result to the workflow, so no return value is needed.
        """Convert input to uppercase and forward to the next node."""
        await **ctx**.send_message**(text.upper()) //OK, bit by bit. text.upper() is built into Python: it turns lowercase into uppercase, that's all; ctx.send_message(...) means: send this result to the workflow's next executor
//In short, this whole UpperCase class receives text, converts it to uppercase, and sends it to the next step. Although it's a class, because it uses the handler decorator, to_upper_case naturally runs when it's invoked. It then tells the workflow it's done and hands off to the next executor.

 Step 2: A function-based executor that reverses the string and yields output
 
@executor(id="reverse_text") //Each step is called an executor. @executor turns the plain Python function below directly into a Workflow executor. Both styles work: the previous one uses a class as an executor, this one uses a function directly. Write it either way, doesn't matter. And by the way, this executor's ID is reverse_text
async def reverse_text(text: str, ctx: WorkflowContext[Never, str]) -> None:
// ctx: WorkflowContext[Never, str], Never = won't send on to another executor, because this is the last step and there are no further steps, str = the final output is a string
    """Reverse the string and yield the final workflow output."""
    await **ctx.yield_output**(text[::-1]) // text[::-1] = reverse the string ABC->CBA, built into Python.
//send_message = to the next executor ;  yield_output = the final result of the whole Workflow.


def create_workflow():
    """Build the workflow: UpperCase → reverse_text."""
    upper = UpperCase(id="upper_case") //This is because the capitalized UpperCase is a class, and the ID is passed in, so there's an extra layer. The only differences from the function-based one: 1. Since it's a class, the ID has to be passed in parentheses. 2. With the function-based one you can write it directly in @executor(id). It's just a syntax difference. This creates an instance of the class; the ID passed in is upper_case, but when you use this object you still use the name upper.
    return WorkflowBuilder(start_executor=upper).add_edge(upper, reverse_text).build()
//OK, this long return line does three things:

First, it uses WorkflowBuilder with start_executor = upper in the parentheses, meaning I'm building a workflow and the first step starts from the upper object.
Second, it adds an edge between upper and reverse_text (an edge is the bridge connecting two executors). So think of it as connecting the upper executor and the reverse_text executor. reverse_text is the function we just wrote; it's a function itself, so it's directly an executor. No need to explain further.
Third, the build. As I said, if you have other executors, keep chaining them in front, one after another. But adding build() means it ends here: go build it.


---
Build and run the workflow:

workflow = create_workflow() //This might get lost in there and be easy to miss. But create_workflow is itself a function, so it has to be called here, and only then is this workflow a real workflow

events = await workflow.run("hello world") //Oh, and this is where it actually runs; the string "hello world" has been passed in as input; await workflow.run(...) doesn't "magically await every async inside automatically." Rather, workflow.run() itself is responsible for calling, awaiting, and chaining those async executors one by one. Outside, you only need to wait for the whole workflow to finish. So there are actually two asyncs inside, but I use only one await and just wait for each of them to finish.

events is used because the Workflow produces many "events" while running. Conceptually, for example:

workflow started
upper started
upper produced message
reverse_text started
reverse_text produced output
workflow finished

The framework collects everything that happens during the run.
print(f"Output: {events.get_outputs()}")
print(f"Final state: {events.get_final_state()}")

---

# harness in Microsoft Agent Framework

from agent_framework import create_harness_agent
from agent_framework.openai import OpenAIChatClient //agent_framework is Microsoft's Agent Framework, not OpenAI's SDK. OpenAIChatClient here is just a "client adapter for connecting to OpenAI models" provided by Microsoft Agent Framework.

**agent = create_harness_agent(
    OpenAIChatClient(model="gpt-4o"),
)
** //Here the agent platform's OpenAI provider (especially OpenAI Chat Client) handles everything; all the harness is inside, meaning you don't have to write it yourself
// A session carries the harness state (plan, todos, history) across turns.
session = agent.create_session()

print("Harness agent ready. Type 'exit' to quit.")
while True:
    user_input = input("> ") //input() is a built-in Python function: it waits for the user to type text in the terminal. > is the prompt text.
    if user_input.strip().lower() in {"exit", "quit"}:
        break

    # Stream this turn's output as the harness plans and works through the request.
    async for chunk in agent.run(user_input, session=session, stream=True): //agent.run(...) means: have this Agent handle one user input. session here is the one created by session = agent.create_session() just now, which carries continuity. stream=True means: don't wait until the Agent finishes generating everything and give it to me all at once; give it to me as it's generated. This is an async data stream; whenever a new piece of data comes out, grab a chunk
        if chunk.text: //Some chunks don't necessarily contain text. They might represent other events, like state changes, tool calls, metadata, etc. if chunk.text: means: only print if this piece actually has text.
            print(chunk.text, end="", flush=True) //end="" means: don't add a newline after printing; flush=True: "show the piece I just printed on screen right away, don't hold onto it." Sometimes, for efficiency, a program puts output into a buffer and shows it once more has accumulated. Streaming hates that.
    print() // This last one runs after the async for loop. Since nothing before it added a newline (end="" is empty), this line is here to add a newline
