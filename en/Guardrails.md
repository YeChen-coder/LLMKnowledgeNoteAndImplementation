> English version of [Guardrails.md](../Guardrails.md). Original notes are written in Chinese/English mix.

# There is no single unified "Guardrail AI"

## Input Guardrail
user's input -> Guard decides whether it's a safety issue / contains sensitive information / is a forbidden topic, etc.

## LLM's, in the system prompt

## Output Guardrail
After the model finishes generating, the output rail can decide to allow, modify, or block outright.
There is no single unified "Guardrail AI." The simpler and more direct something is, the faster it often is. For example, you can just build a word filter.
One step up from that is a safety model, a fairly small model, since you can't turn every word in the world into a keyword. This model only considers whether a sentence is problematic, so it's definitely faster than the main Language Model and doesn't need as much intelligence.

So the output of the safety classifier model gets sent into this safety model, and it outputs a probability, e.g. probability values for categories like violence, sexual, and hate.

<img width="815" height="170" alt="image" src="https://github.com/user-attachments/assets/a189a345-9154-446f-ae3c-3fa1464ab3f1" />

##  LLM-as-a-Judge OR LLM self-check / policy check.

Add another layer after the whole model output: send it as input to a separate session. That other session then judges whether your entire output is correct and good. Likewise, if it's not OK, rewrite/block.

## Execution / Tool Guardrail.
Also, since today's systems all have Agent features (called Agent, but dig to the bottom and it's just Tool Calling), this Guardrail also involves deciding "whether this operation is allowed to happen." This part of the guardrail doesn't need to be an LLM at all; hard-coded logic is better in both stability and feasibility.

The main model says: "I want to call transfer_money(amount=5000, to=Bob)"

The Guardrail doesn't re-plan the whole thing. It only asks a few very mechanical questions: Can this tool be called? → Does this user have permission? → Are the parameters valid? → Does the amount exceed the limit? → Is human confirmation required? -Put that way it's easy to understand: permissions/parameters are all things you can hard-code.

Then it gives a result: pass → actually call the tool
or: fail → reject / ask the user to confirm


## Retrieval Guardrail for Indirect Prompt Injection
If the system uses RAG and your RAG content contains jailbreak text, like prompt injection or data leakage, that's indirect prompt injection.
RAG Search -> Retrieval Guard -> LLM
