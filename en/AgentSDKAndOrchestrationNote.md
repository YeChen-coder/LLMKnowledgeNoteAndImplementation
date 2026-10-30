> English version of [AgentSDKAndOrchestrationNote.md](../AgentSDKAndOrchestrationNote.md). Original notes are written in Chinese/English mix.

Note of https://openai.github.io/openai-agents-python/quickstart/?utm_source=chatgpt.com

# async + await 

Use asyncio to start an async main() function, wait inside it for Runner.run(...) to finish, then print the final result. (A core point of async/await: while waiting on I/O, execution can be handed over to other async tasks.)
import asyncio
from agents import Agent, Runner

agent = Agent(
    name="History Tutor",
    instructions="You answer history questions clearly and concisely.",
)
async def main(): //async def defines an "async function," also called a coroutine function. With async, it doesn't run all its code immediately in order like a normal function; instead you get a coroutine /ˌkəuru:'ti:n/ object. You usually need **await main()** or **asyncio.run(main())** to actually run it. async is short for asynchronous /ei'siŋkrənəs/
    result = **await** Runner.run(agent, "When did the Roman Empire fall?")  //Think of await as: wait for this async operation to finish, then get its return value. Start Runner.run(...) -> this may take a while -> wait for the result -> once it's back -> assign it to result; if Runner.run() calls an AI Agent, it may need to: send a request → wait for the server → Agent processing → return the result. A network request can take hundreds of milliseconds or even seconds. In a fully synchronous program, the Python thread can usually only sit idle while "waiting on the network." This is actually async nested in async.
    
    print(result.final_output)

if __name__ == "__main__":
    **asyncio.run(main())**

<img width="760" height="307" alt="image" src="https://github.com/user-attachments/assets/0b533756-0c9b-4d17-ba0f-c2eef229a7f7" />

The first one is easy: the most primitive kind, nothing provided, assembled entirely by hand.

The second is the session in the SDK, which puts previous records into the session. It's still a mechanism that lets developers control the conversation themselves, and it can do a lot: you can hook up a database, put in all kinds of custom content, and the model doesn't even have to be OpenAI's. The Agent SDK is designed this way for two core reasons:

1. Conversation memory shouldn't be tied to the server state on the OpenAI Response API side, so developers can hook a database into the history and modify content flexibly themselves
2. Developers may not be using OpenAI's API at all, in which case they naturally can't use the third option's previous_response_id or conversation_id

The third:
• previous_response_id is the most no-brainer option; just follow along from the previous one
• conversation_id is a very fixed thing that can be used across windows

I honestly haven't figured this part out yet. I'll keep reading and come back to fill it in once I understand it.

Adding tools:


@tool
def history_fun_fact() -> str:
    """Return a short history fact."""
    return "Sharks are older than trees."

## Agent orchestration 
Handoffs and agents as tools are on equal footing. You can of course combine them: a triage agent receives everything first and hands off all tasks to a specialist, and then that specialist calls a group of small agents, using agents as tools.

With handoffs you give everything away: the next one takes full ownership. With agents as tools, there's still one big foreman responsible from start to finish; the agent just handles certain small things in the middle that are delegated to it.

There are also more obvious options:

If you really don't want to use handoff, or the code/application has special circumstances, just write it in code:

• One way is structured output: as long as the large model returning this gives structured output, you can parse parameters or whatever else out of the JSON. Whatever it is, you can definitely get it out.

• The other is for very simple cases: if the flow is 1234567 and must go in order, just go through them one by one; Python code executes top to bottom.

And while we're at it, an ordinary while loop can do the job too.

Then sometimes you need concurrency, with many Agents running at the same time; use asyncio.gather to collect the results. Whatever the reason, you'll naturally need it in concrete use cases. It's purely a code logic design and doesn't really have much to do with the Agent SDK; just think of it as a general-purpose function.
 
### handoff

First, handoff. Its sub-agents are all parallel, but before you can hand things to a sub-agent, someone has to dispatch them, so a triage agent is introduced here.

In the triage agent's handoffs[], you have to list all the agents' names so it knows who to hand things to.
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
    **handoff_description="Specialist agent for math questions",** //handoff_description gives the routing agent extra context about when to delegate. The triage_agent can see this handoff description, as long as the agent is in its handoffs array.
    instructions="You explain math step by step and include worked examples.",
)

---
In actual use, since triage_agent, history_tutor_agent, and math_tutor_agent are all defined already, you don't need to rewrite them when calling from the main file. Just call Runner.run as before, passing triage_agent and the message.

Once triage_agent gets the message, it automatically dispatches based on its own instructions plus the handoff descriptions it received. So everything inside runs and is handled automatically. The result you get is the same as before: a result.final_output, which is the return from the agent it was routed to.

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

This first defines three agents:

1. The first writes the outline
2. The second checks the outline
3. The third writes the story based on the outline

Defining an agent is very simple and very straightforward.

class OutlineCheckerOutput(BaseModel): //BaseModel comes from Pydantic. For now you can crudely remember it as: BaseModel = a Python base class specifically for describing and validating "what structured data looks like." Here it means I'm defining a structured data type.
    good_quality: bool
    is_scifi: bool


outline_checker_agent = Agent(
    name="outline_checker_agent",
    instructions="Read the given story outline, and judge the quality. Also, determine if it is a scifi story.",
    output_type=OutlineCheckerOutput, //output_type=OutlineCheckerOutput is also officially supported by the SDK. The official docs say clearly: output_type can take a Pydantic BaseModel class directly; once passed in, the Agent uses structured outputs instead of plain string output.
)

Here: output_type=OutlineCheckerOutput

does not mean: "Model, here's a Python class, go figure out what it means yourself." No.

The Agents SDK reads this class. Since it's a Pydantic model, the SDK can convert it into a JSON Schema like this:

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

Meaning the final answer must be an object containing: good_quality → boolean + is_scifi → boolean.
That's really what the Agents SDK source code does: it uses Pydantic's TypeAdapter to generate a JSON Schema from output_type, and handles validating/parsing the JSON the model returns.

The main function is:

async def main():
    input_prompt = input_with_fallback( // input_with_fallback is just a helper written for this example. Ask the user for input → if the user actually typed something, use it → if not, use the default
        "What kind of story do you want? ",
        "Write a short sci-fi story.",
    )

    // Ensure the entire workflow is a single trace
    with trace("Deterministic story flow"): // with runs the whole block below inside a special "context." E.g. enter the context managed by something → run A → run B → run C → leave the context; here it means: from here on, record the whole flow below as a trace called "Deterministic story flow". For now think of a trace as: the run record of one Agent workflow.        # 1. Generate an outline
        **outline_result** = await Runner.run(
            story_outline_agent,
            input_prompt,
        ) 
        print("Outline generated")

        // 2. Check the outline
        outline_checker_result = await Runner.run(
            outline_checker_agent,
            **outline_result.final_output,** //Because Runner.run() above returns more than just the text; it returns a result object. This hands the outline generated by the first Agent to the second Agent to check.
        )

        //outline_result = the result object of the whole run; outline_result.final_output = the Agent's final output

        // 3. Add a gate to stop if the outline is not good quality or not a scifi story; this just checks whether two conditions are met. If not, exit right away; it never gets pushed to the story agent
        assert isinstance(outline_checker_result.final_output, OutlineCheckerOutput)
        //isinstance means: is something an instance of some class? E.g. "hello" is an instance of str, so isinstance(x, str) is true; here it asks whether outline_checker_result.final_output is an OutlineCheckerOutput object, i.e. a type check.
        //assert something: for now remember it as: I assert that something must be true.
        //In short: let me confirm that the checker agent's final output really is of type OutlineCheckerOutput. If not, something we didn't expect happened, so raise an error right away.
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

  res_1, res_2, res_3 = await asyncio.gather( //gather waits for A, B, and C to all finish → then returns all the results at once
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

        
//These are the three. They're all the same agent, but three sessions are opened, producing three outputs. Finally a translation picker selects a good one, i.e. runs one session of another agent to filter once more. That simple, that's it.
Multiple Agents that don't depend on each other → run at the same time → wait for all to finish → aggregate the results

i.e. Parallel / Fan-out / Fan-in.

Input → A + B + C = fan-out

A + B + C → Results = fan-in
OpenAI's official agent_patterns README also lists this as its own common pattern: multiple agents can run in parallel, which both lowers latency and lets you generate several candidates at once and pick one. The official orchestration docs explicitly use asyncio.gather as the example of code-driven parallel orchestration.

# LLM as a judge

https://github.com/openai/openai-agents-python/blob/main/examples/agent_patterns/llm_as_a_judge.py

Setting aside the Python syntax, the logic is all very simple.

The core is having the Evaluation Agent output a structured response each round, with two parts:

1. score
2. a fairly specific evaluation

Only when the score is pass is it considered passed and moves on. There's actually no next story step here, only the outline and the evaluator.

The logic is very simple: cap the maximum number of rounds (max turn), since the while loop can't run forever or the tokens won't hold up. With the cap, a simple check in code decides whether the while loop continues or breaks.

Apart from all the Python syntax used (sorry, that's on me for being weak at it), the whole logic is very simple, a limit anyone would think of.

from __future__ import annotations //This is Python's own thing, nothing to do with Agents. For now remember it as: makes Python more flexible when handling type hints.

For example, these later on:
list[TResponseInputItem]
str | None


are all type hints.
This line doesn't affect your understanding of Agent orchestration for now; it's enough to know it's a setting related to Python type annotations.

import asyncio
from dataclasses import dataclass //For now think of dataclass as: Python makes it easy for you to define a class "mainly responsible for holding data."
from typing import Literal //Literal is a Python type hint. Later you'll see: Literal["pass", "needs_improvement", "fail"], meaning the value can only be one of these three strings. Not just any str.

from agents import Agent, ItemHelpers, Runner, TResponseInputItem, trace
from examples.auto_mode import input_with_fallback, is_auto_mode //input_with_fallback means: user typed something → use the user's input; user typed nothing → use the default; for now think of is_auto_mode() as: whether this example is currently running in the OpenAI examples' own automated test mode. Not a core Agent orchestration concept.

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


@dataclass //decorator; for now think of it as: turns the EvaluationFeedback class into a class dedicated to holding data. 
class EvaluationFeedback: // 
    feedback: str // feedback must be a string
    score: Literal["pass", "needs_improvement", "fail"] //As just said, this uses Literal: score not only has to be a string, it must be one of these three values.

// The Evaluator must ultimately return two things: text feedback + a score from a fixed range.

evaluator = Agent[None]( // No need to dig into this [None] now. It's Python generic type stuff. Roughly, it says this Agent has no extra dependency/context type to pass in.

    name="evaluator",
    instructions=(
        "You evaluate a story outline and decide if it's good enough. "
        "If it's not good enough, you provide feedback on what needs to be improved. "
        "Never give it a pass on the first try. After 5 attempts, you can give it a pass if the story outline is good enough - do not go for perfection"
    ),
    output_type=EvaluationFeedback, //The Evaluator shouldn't return a blob of free-form natural language; it must output following the EvaluationFeedback structure.
)


async def main() -> None: // -> None is a return type hint, meaning main() itself doesn't intend to return a value.
    msg = input_with_fallback(
        "What kind of story would you like to hear? ",
        "A detective story in space.",
    )
    
    input_items: list[TResponseInputItem] = [{"content": msg, "role": "user"}] //The {} inside is a Python dictionary; [] is a list;

    // : list[TResponseInputItem] is a type hint. Roughly: input_items is a list, and each item in it should be an input item the Responses API accepts.
    // TResponseInputItem is the type of a valid model input item.
    //This whole line: create a "list of model input messages."

    latest_outline: str | None = None //latest_outline can be a string or None. Why the = None? Because at the start there's nothing, so it starts as None
    auto_mode = is_auto_mode()
    max_rounds = 3 if auto_mode else None // conditional expression / ternary expression. Common Python syntax: if auto_mode is True, max_rounds is 3; if auto_mode is False, max_rounds is None. I.e. in normal mode max_rounds is None, but in automated test mode it's set to 3
    rounds = 0

    # We'll run the entire workflow in a single trace
    
    with trace("LLM as a judge"): //As said before, with trace records the whole flow as one Trace
        while True: // The outer loop runs forever until it hits a break inside
            story_outline_result = await Runner.run(
                story_outline_generator,
                input_items, //What's passed in is the input_items message list; the first time it only has user: A detective story in space.
            )

            input_items = story_outline_result.to_input_list() //Turns this round's content into an input list that can be used in the next round. Old input + newly generated content → new input_items
            
            latest_outline = ItemHelpers.text_message_outputs(story_outline_result.new_items)//Pulls the text content out of the items newly produced by the Generator this round. ItemHelpers.text_message_outputs(...) is provided by the SDK for extracting information.
            print("Story outline generated")

            evaluator_result = await Runner.run(evaluator, input_items) // Gives the just-assembled input_items to the evaluator.
            result: EvaluationFeedback = evaluator_result.final_output //Here result: EvaluationFeedback is just another hint to Python/the type checker that result should be of type EvaluationFeedback. final_output, as said before, is the result that actually comes out at the end.

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

            input_items.append({"content": f"Feedback: {result.feedback}", "role": "user"}) //append adds one more item to the end of the list; now input_items is the user's request + the Generator's first outline + Feedback

    print(f"Final story outline: {latest_outline}")


if __name__ == "__main__":
    asyncio.run(main())

# Checkpoint in LongGraph

This exists because, depending on the situation, checkpoints really are necessary. A checkpoint is essentially a snapshot.

It's mainly to guard against a huge workflow or loop breaking at some node midway, while other function calls have already finished normally. For example, a money-sensitive operation like a charge has already gone through. Once you discover it broke midway, how do you get it to continue? For some critical services, you have to decide what to do next without anything being executed twice.

This is really the inevitable path, whether for token cost or for certain service mechanisms: a payment, for example, is inherently a one-time operation. Even if it breaks midway, even if the data center burns down, it must not charge a second time.

Here I mean one-time charges like buying something online, not subscription billing. Sorry, I was a bit vague there, but that's roughly it.
