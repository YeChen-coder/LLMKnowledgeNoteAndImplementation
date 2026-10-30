> English version of [JesusMyNote.md](../JesusMyNote.md). Original notes are written in Chinese/English mix.

The README Claude Code wrote has way too much in it, and since I don't fully understand it yet, I don't want to delete things now only to hit the same problem later. No way around it: this kind of Technical Debt just shows up during exploration.

So that I at least have a somewhat clearer grasp, all the notes below are basically dictated by me after understanding things, and they're more focused on the key points than the original README.

I'll explain why it's configured this way in a bit. First let's pin down a few facts so it doesn't get too messy. There are quite a few knobs here, and it really does a lot of engineering optimization:

1. OpenAI's built-in interrupt response is set to false, meaning OpenAI's native barge-in isn't used here. This is related to auto-response; I'll explain why shortly.
2. create response (i.e. OpenAI's own auto-response) is also set to false.
3. Barge-in is actually implemented by us here: locally, using acoustic echo cancellation plus detecting whether the ambient volume suddenly rises, the local side decides whether the user is currently interrupting.

OK. To make what follows clearer, let's first review the general input structure of OpenAI's Large Language Models. The point is to explain the session.instructions vs. response.instructions issue.

Generally, we can think of a language model's input as having three parts.

The first part is the instructions in effect for this call. This is usually what everyone online talks about: how to write prompts, various conventions, structured schemas, and so on. It comes in two kinds:

One is what OpenAI calls session.instructions, the default. It takes effect when this response isn't given response.instructions.

But if this response does provide response.instructions, it replaces session.instructions.

session.instructions and response.instructions are either/or: there has to be one, and only one. One replaces the other; they don't overlap.

The second part is the tool definitions available for this call, including tool names, tool descriptions, parameter structures, etc., all provided by the tools field.

The third part is the conversation history used for this call, broken down into many fields:

• system: supplementary instructions or material added earlier
• user: the user's input
• assistant: the model's previous answers
• system: RAG results added later also go here
• function call and function call output: included if a function was called
• the user's latest input, etc.

The whole thing keeps going in conversation order. Basically it's always the added material plus the user's input; once it gets an answer, it runs a new round.

If an instruction comes in via item.create, it goes into the system part of the third section. That's not the same thing as the replaceable instruction in the first part.

---

OK, let's continue.

In this project, we need to guarantee both response time and that it doesn't go OOC, so we have to alternate between two prompts. (A note on why it's done this way:

Reading on, you'll see only prompt 2 does the real work. So why must prompt 1 exist? There are two reasons, and the main one is that running the RAG pipeline in the background takes time. The prompter has to get the RAG result first and then push it into the conversation. Before the result is fetched and pushed, there's a gap of roughly two to three seconds, which is unforgivable for a realtime conversation. Someone might say OpenAI's Realtime API can already talk instantly, and the docs describe backend support for this: start the response first, let the model say a filler line, and run RAG and return the result while it's saying the filler. Why doesn't that work? Because if you do that, the model's filler will be something like "Let me check what the scriptures say" or "Let me look at my experiences." For a Persona that's very out of character and wrecks the experience. So that approach doesn't work. The only way is to get the memory retrieved from RAG locally first, assemble all the prompts and push them to the model, so what the model says is right from the start, rather than stalling with filler.)

The specific response.instruction structure is below. It actually uses an alternating 1, 2, 1, 2 mechanism.

Prompt 1 = shared persona + user's transcript for this turn + A's instructions

Prompt 2 = shared persona + user's transcript for this turn + B's instructions + conditional instructions appended as needed + [memory from RAG]

Source of the shared persona text: INSTRUCTIONS_MEMORY in persona.py.

Note: to keep the context window from exploding, and my wallet from exploding, the design here is actually quite clever.

The shared persona, the transcribed text of the user's turn, A's instructions, B's instructions, and the conditional instructions we're discussing all apply only to the current response.

That's because they're passed in via response.instructions. Written this way, they're instructions for this one response only, and they don't keep piling up as prompt after prompt in the historical context window.

So what does stay? Only three things keep getting pushed in:

1. The user's speech (audio): sent in continuously as item.content via conversation.item.create
2. The memories fetched round after round are also pushed in
3. Its own responses are pushed in too

That's why the shared persona (base persona) has to go into both Prompt 1 and Prompt 2.

It's not duplication. It's because what was put in earlier isn't in its context window at all. So when it's time to respond this round, it has to be pushed in again.

Prompt 1: after the shared persona, append this

Source: _phase_instructions("A", ...) in voice.py.

'''
// What they just said
Transcribed: "<USER_TRANSCRIPT>" //We've seen it mishear words and drop negations in this project, so this text was added as a correction hint
Trust these words over what you think you heard, including every "not" and "don't".

// Right now
Say only ONE short sentence, at most about twelve words, that receives what they said and how they feel, in your own voice. No advice, no question, no scripture, and do not say what you are about to do (thinking, looking, searching, closing). Then stop; you will go on in a moment. If it is a plain question for information (the weather, the news, a price), say only a few natural words such as "Let me see." instead.

'''

Prompt 2: after the shared persona, append this: _phase_instructions("B", ...) in voice.py.

'''
// What they just said
Transcribed: "<USER_TRANSCRIPT>"
Trust these words over what you think you heard, including every "not" and "don't".

// Right now
You have just said to them: "<PHASE_A_SPOKEN_TEXT>"
Go on from there as one flowing reply: do not repeat or rephrase that sentence, do not greet them again, and do not open with a filler such as "let me sit with you". Say one or two more short sentences that add something new, following everything above (including any MEMORY note just given). If that sentence already answered a simple question, such as your name, add only one short warm line. Never describe what you are about to say or do: no "I'll share a memory", no "then I'll stay with you". If you tell a memory, simply begin it.
'''

Prompt 2 also appends the following text depending on the situation:

If it's a factual query:
They asked for information. If you have not looked it up yet, call research_web now; then tell them the answer plainly in a sentence or two, with nothing spiritual attached.

If it's a question that needs professional judgment:
This needs a professional's judgment. Do not answer it, give steps or look it up: say kindly who would know (the doctor who has seen them, a lawyer, someone who works in that trade), then turn to how they are. (It's written this way because I sincerely think someone trying to solve a technical problem, whether medical, programming, or anything else, really shouldn't be asking here.)

If it's ordinary conversation and the provided memory is a close fit:
A moment from your life speaks closely to this (the MEMORY note): these sentences are that memory. Begin it at once ("I remember..."), with its names, then one short line back to them. No advice or image of your own this time.

If no memory is provided this turn:
No particular moment of your life comes to mind this time: do not tell one, and do not begin "I remember". Speak as yourself.
These conditions aren't all added at once. For example, for ordinary conversation with no memory found, the last one is added; for a factual query with no memory, both the "factual query" and "no memory" ones are added.


With all of the above in mind, let's walk through the overall flow. To avoid too many nested layers, the RAG details are set aside for later; for now, just the overall flow.

<img width="2244" height="1280" alt="JesusFlow" src="https://github.com/user-attachments/assets/3b6214b7-f7f6-4fb3-b489-b4bcd4c98856" />



Everything starts with the user:

1. The user says something, which goes into the real-time model (note: the session is already created when the program starts, before the user speaks, so no need to worry about that).
2. After the user's speech enters the real-time model, it goes two places:
(a) The audio is appended directly to the real-time model as a user message.
(b) The real-time model must return a transcription to text. Later steps need text to match against RAG; you can't use audio directly, so it has to return text no matter what.
3. Once the program has the transcribed text, it does two things in parallel:
(a) create response with prompt 1
(b) use the just-transcribed text to match RAG and fetch memory
4. At this point the user hears response 1, generated by the real-time model from prompt 1. While the user is listening to response 1, the backend retrieves the memory.
5. Once it has the memory, the backend makes another request, create response with memory + prompt 2, sent to the real-time model.
6. The real-time model generates response 2 from the memory and prompt 2 and plays it to the user in the current conversation.

OK, that's from the start to the end of the first full turn. Multi-turn after that is just this loop repeating.

As for exactly what's kept, what's currently in the context window, and how it's done, that's all covered above, so I won't repeat it. Once the first turn is clear, the rest is easy.

Note: it's the same session from start to finish. Preventing the two audio segments from clashing is just engineering (very simple, handled on the playback side.

Either way, both have to be output on the same audio track and queue in the same playback queue. The only thing done here is waiting for Response 1's audio to finish playing before playing Response 2's audio, since it's a queue. That's it.), but it must never be split into two sessions. This ensures the model has all previous responses during multi-turn; split it into two and the context becomes incoherent, nothing connects.
