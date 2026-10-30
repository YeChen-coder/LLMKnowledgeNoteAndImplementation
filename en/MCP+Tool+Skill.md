> English version of [MCP+Tool+Skill.md](../MCP+Tool+Skill.md). Original notes are written in Chinese/English mix.

If all three of these exist, there must be a reason. I think a lot of things have merged by now, MCP and Tool especially, but to make it easy to remember, you can think of it like this:

Tool calling conceptually covers both local direct tools (the function is already written inside the app; whichever one the LLM response names, you just call it) and MCP (MCP can also be written locally, and local MCP isn't much different from tool calling in what it does).

MCP can be local or remote. The real point of MCP is remote: once remote is supported, you can directly use MCPs offered by other providers. For closed-source tools like Gmail, Notion, and Google Drive, asking them to open-source is a pipe dream, and it's equally absurd to reimplement all those functions yourself locally. So in that case you just use their MCP.

This kind of MCP is essentially another layer wrapped around the API the tool already offers, so the Large Language Model knows how to assemble a request / fill variables into a predefined, half-built request and POST it. Or it just hands the model a mature workflow: to update, POST this request; to read something, do this; to delete something, do that. It's really just giving it options, plus whatever assembly is needed; at minimum, all the user-side headers, authentication, and so on still have to be put together.

As for Skill, think of it as a fixed prompt that works well and that you're too lazy to copy-paste every time. If you had to paste these steps to the model every time, wouldn't that be exhausting? You can't remember them either, so you'd need a doc to save them and paste from. Not elegant at all, so it became a Skill.

Skills now work much like MCP: neither exposes the full original text up front, or the context would blow up.

Everyone does it the same way now: give only the table of contents first. The model looks at the TOC, says what it needs, and then the full text of that Skill is fed to the model.

-This is really about the context window being limited, which runs through the whole story of Large Language Models. Whether it's context engineering and prompt engineering early on, or MCP and tool calling later, or not stuffing everything into the model at once and giving it only a TOC, or compacting, the core is the same: it's all because of the limited context window.

Dig one level deeper: why can't the model handle something this long? When we say it can't handle it, we don't mean it won't fit. We mean that in such a huge input, the few core goals the user actually cares about gradually get forgotten by the model as more and more content is added.

A Language Model isn't a person after all; it tends to treat all input pretty "equally." Technically you can apply some weighting or emphasis in the computation, but when input keeps pouring in, the model still forgets some points. And most of the time, those points are exactly what the user actually wants to achieve, and the model forgets them.

Dig one more layer and you get into some underlying issues with the attention mechanism in the Transformer architecture itself. In short, the best that the smartest people in the world have managed, by far, is what we have now.

So accept the limits, and then get creative within them.

Continuing: because of the context window limits, we have to trade time for space. Huh, how did this connect back to algorithms? Anyway, there are just two choices: trade space for time, or trade time for space.

Here we choose to trade time for space. The context window is the space, and it's limited, so you can't stuff everything in. You can only add more turns, more rounds of iteration, and put into the context window only what the large language model decides it needs. Leave out what it doesn't. That's the logic.

And in all this back-and-forth, whether it's tool calling, skills, or local/remote MCP, there are many rounds of turns in between: trading number of requests for space.
