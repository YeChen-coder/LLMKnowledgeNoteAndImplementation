> English version of [CodexHarness+OpenCodex+CodexCli.md](../CodexHarness+OpenCodex+CodexCli.md). Original notes are written in Chinese/English mix.

# Codex harness = hands that designed for OpenAI model

Codex = brain + body

Brain = GPT / Claude / Gemini / Kimi / DeepSeek

Body = Codex Harness

The model just keeps telling the Codex Harness, "Here's what I want to do next," and the harness carries it out (reading, running commands, making edits, and so on).

Look at it this way: the Codex Harness and ChatGPT (whichever model) have a native relationship. ChatGPT has certainly been specifically trained on what the Codex Harness feeds it, so the two naturally fit together hand in glove (if a company's own products didn't recognize each other, that would be hugely embarrassing).

But if you swap the model from OpenAI to someone else's, that fit breaks.

Over the past two years all the big models have shown off their own strengths, a hundred schools contending. Each company's smart people built their own thing, similar in spirit but different in form; you can't expect everyone's brains to line up. So naturally each company understands tool calling in its own distinctive way. Force another company's brain into the Codex Harness body and something is bound to go wrong. Problems are completely expected; it would be strange if there weren't any.

Now down to the Codex Harness level.

The Codex Harness has some fairly special implementations, including apply_patch, which is used very frequently to edit files, and it doesn't fully follow the very standard tool calling or function calling approach. Who defines what's standard is itself a question, and there aren't many big companies out there anyway. Here I'm only talking about the compatibility problems that come up.

For example, when the Codex Harness tells an OpenAI model it has a special tool and how the tool works, the OpenAI model does understand, outputs in that format, and the Harness processes it fine.

But another company's model can't make sense of it and can't output in the format the Codex Harness wants. So naturally the corresponding function never fires, the parameters don't line up, and the Codex Harness can't parse or execute the function at all. It not running, not working, is entirely reasonable. It's like asking someone with no sense of direction to lead the way: that part of their nervous system is just weak. They can walk the wrong way holding a phone map, and the moment they look up they don't know where they are. They really can't do it. Getting from the station to their front door is pure autopilot, and the moment they actually think about it they can't tell which building is their apartment.

Ahem, I'm off topic. Back to it. When problems like this show up, some smart people are bound to find a way to smooth over the gap.

Sorry, one more aside. After all this, I think it's the same as the protocol fights in IoT: politically, it's a fight over who gets to set the terms; technically, it's purely because we're still in a contested phase. The models out there each have their own strengths and nobody can dominate everything, which leads to a flowering of interfaces and tool formats. Right now there's no need, no possibility, and no point in pushing each model to proactively fit other vendors' harnesses, and that's why these problems exist.

In this chaotic phase, smart people naturally come up with ways to smooth out the differences by whatever means: build middleware, with the Codex harness on the left and whichever large language model on the right.

Doing it this way in the middle surely costs some performance/security, or whatever other factor was the reason for the special design in the first place. There will always be some loss, but at least it works now. For now, working is a good thing. As for the future, future smart people will solve it.

# OpenCodex is bornned to diminish /dɪˈmɪnɪʃ/ the gap between harness and thrid-party LLM

Back to the present: on the Codex side, the thing that smooths over this interface-format gap is OpenCodex.

The problem can be abstracted and looked at from two angles: if it doesn't connect, either the left side is the problem or the right side is.

Let's treat the left as the Harness and the right as the third-party Large Model:

(1) If the problem is on the left:
   
The fix is to translate the custom tools in Codex on the left into a format the right side understands, which is what OpenCodex does. From the left-side-is-wrong angle, the "manual" the left Harness gives the third-party model is the problem: the model on the right can't read the manual the Harness gave it. Easy: translate the manual into something the right side can read. But note: the right side gets a manual it can read and replies according to it. But then the left side can't quite read the reply, so it has to be translated again, into the format the left side ideally wants. Think of it like two people speaking different dialects who can't understand each other, so a middleman has to relay messages both ways.

The flow is:

1. The left tells the middleman what it wants
2. The middleman translates the left's dialect into the right's dialect and passes it on
3. The right replies in its own dialect
4. The middleman translates the right's dialect back into the left's and returns it to the left

Run through that flow and it works.

(2) If you look at it from the right side:

If we decide the Codex Harness itself is fine, and the custom tool descriptions it sends the LLM are perfectly normal, then the brain on the right is the problem: it doesn't know how to take it or how to reply. Fixing it on the right means changing the model, whether by fine-tuning or whatever. But that's really not worth it. Let's be reasonable; no need to actually go tune a model of this size. If you just want to play around, stick to a few-B model, seriously. -And trust me, it's not easy. Way back last summer I spent two weeks on it: everything theoretically could run, but as soon as it was unpacked and I tried to train, it blew up with out of memory. It really doesn't work. Until you've committed to professional hardware for local deployment, don't get too fixated on this. Know when to let go.

Back to how to actually get this done: there are just too many models out there, so you need a common language in the middle. Otherwise, as models multiply, complexity grows quadratically.

So you need to build a bridge in the middle and bring the complexity down to O(n). OpenCodex doesn't build a direct mapping for every pair of models. Instead it sits in the middle as a bridge: everything coming in is first translated into OpenCodex's own language, and on the way out it's translated into whatever format the target model understands.

To add: besides the comparison above, the two sides not being aligned really is a pain, especially between the harness and different vendors' models, where nobody understands anyone and they can't communicate.

There are also some small mismatches that are just eye-roll-worthy. For example, some MCPs don't allow dots in names. Easy: OpenCodex converts on the way in and out, dots to underscores coming in, underscores back to dots going out. Whoever wants what, give it to them.

From that angle, how can you not call OpenCodex the herald of universal harmony?

# codex cli VS openai agent api

I suddenly figured out a rather odd comparison. It may sound a bit strange.

It's because while writing another diagnostic agent, I directly replicated real-world T1 and T2 troubleshooting logic in the agent program:

• At T1, first use a not-so-smart lightweight model in Codex (e.g. 5.6 Luna) to investigate.

• If it can't find the problem, escalate to T2 and switch to Codex's 5.6 Astro to gather more complete information and figure out what actually happened.

My computer is high-spec, with Codex installed locally and Docker running directly, so there's no pressure on compute or storage at all and I never feel a bottleneck. I really was out of touch, "let them eat cake," with a cognitive blind spot. I didn't quite get why the Agent API's sandbox was made so flexibly configurable. With Codex CLI already there, isn't just using it locally better? No need to set up a sandbox yourself, and no agonizing over self-hosted vs. managed.

But later I suddenly realized: Codex is well known for being very resource-hungry, both CPU and memory. The Agent API didn't appeal to me much simply because this computer has plenty of compute and I don't need to worry about performance.

Switch to a laptop with average specs and resources get tight immediately. Forget running two Codexes; even running one is a struggle. Not to mention running two containers in Docker (each running its own Codex), plus the Codex that always runs on the host: three Codexes running at once is extremely demanding on local resources.

If you use the Agent API instead and host the sandbox elsewhere (however it's hosted), the compute doesn't touch the local host at all, which greatly relieves resource pressure on your machine.

Many things are ultimately limited by real-world conditions. A powerful desktop is great, but you can't carry it around. It's exactly these real-world constraints that give rise to diverse needs and use cases.
