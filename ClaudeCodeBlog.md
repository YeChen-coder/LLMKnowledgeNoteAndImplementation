few-shot 不是 recommendation system 那边的 cold start (Cold start 是说，当一个新用户没有很多历史记录可以去做 recommendation 的时候，就去找到一群相似用户，然后用 collaborative filtering 来做)，

few-shot 就是说在 prompt 上，举个例子、按什么格式输出，就这么点事儿。- 这不是本来就是这么做的吗。

# https://code.claude.com/docs/en/best-practices

Most best practices are based on one constraint: Claude’s context window fills up fast, and performance degrades as it fills.

## verify

CC stop when the work looks done. ive Claude something that produces a pass or fail, and the loop closes on its own. 

### In one prompt:

<img width="1057" height="326" alt="image" src="https://github.com/user-attachments/assets/2006d78c-0ae1-4e47-bf39-1841ddec8f6d" />

Run /verify yourself after Claude’s check passes to confirm the change against the running app.

Across a session: set the check as a /goal condition. A separate evaluator re-checks it after every turn and Claude keeps working until the goal resolves. If Claude stalls, Claude Code eventually stops the run with the goal still set — see how /goal evaluation works.

### As a deterministic gate
a Stop hook runs your check as a script and blocks the turn from ending until it passes. Stop input covers the cap on consecutive blocks.

### By a second opinion
a verification subagent or a dynamic workflow that checks its own findings has a fresh model try to refute the result, so the agent doing the work isn’t the one grading it.

## plan mode in Claude is a real brain plan mode; not the to-do list in Codex

Planning is most useful when you’re uncertain about the approach, when the change modifies multiple files, or when you’re unfamiliar with the code being modified. If you could describe the diff in one sentence, skip the plan.

### step1: explore

Enter plan mode by pressing Shift+Tab until the status bar shows ⏸ plan mode on, or start the session with claude --permission-mode plan. Claude reads files and answers questions without making changes.

### step2: plan

<img width="1042" height="351" alt="image" src="https://github.com/user-attachments/assets/33cd8988-6a29-492d-a4d3-34154b94be8a" />

### step3: Implement

<img width="1027" height="317" alt="image" src="https://github.com/user-attachments/assets/74855d4c-d1d1-466f-9422-d6aec2ce81f8" />

### step4: commit

<img width="1020" height="247" alt="image" src="https://github.com/user-attachments/assets/299139b2-cb4b-476a-bf5a-f730373afc46" />

## Provide specific context in your prompts

<img width="1136" height="518" alt="image" src="https://github.com/user-attachments/assets/407ceb8a-ed0d-492f-9054-e28c58642b16" />

### methods to provide data

You can provide rich data to Claude in several ways:
Reference files with @ instead of describing where code lives. Claude reads the file before responding.
Paste images directly. Copy/paste or drag and drop images into the prompt.
Give URLs for documentation and API references. Use /permissions to allowlist frequently used domains.
Pipe in data by running cat error.log | claude -p "explain this error" to send file contents directly.
Let Claude fetch what it needs. Tell Claude to pull context itself using Bash commands, MCP tools, or by reading files.

## CLAUDE.md is loaded in every every session, not for specific project usage

Run /init to generate a starter CLAUDE.md file based on your current project structure, then refine over time.

<img width="1085" height="388" alt="image" src="https://github.com/user-attachments/assets/77d870ed-4505-4508-a41c-2a2e9d47f2d4" />

For a checked-in CLAUDE.md, run /doctor and Claude proposes cuts for content it can derive from the codebase. 

CLAUDE.md files can import additional files using @path/to/import syntax

Run /context to confirm Claude loaded the file. CLAUDE.md is loaded every session, so only include things that apply broadly. For domain knowledge or workflows that are only relevant sometimes, use skills instead. Claude loads them on demand without bloating every conversation.

CLAUDE.md is a special file that Claude reads at the start of every conversation. Include Bash commands, code style, and workflow rules. This gives Claude persistent context it can’t infer from code alone.

With Claude Code v2.1.283 or later, auto mode is the built-in starting permission mode for interactive terminal and VS Code sessions: a separate classifier model reviews most actions instead of you and blocks only what looks risky, such as scope escalation, unknown infrastructure, or hostile-content-driven actions.

### other tools
#### Cli: install CLI; tell Claude to use 'this-cli --help' to learn
#### MCP servers
Run claude mcp add with a server name and URL or command to connect external tools like Notion, Figma, or your database. For example: claude mcp add --transport http notion https://mcp.notion.com/mcp.
#### Hooks = hard restriction
Hooks run scripts automatically at specific points in Claude’s workflow. Unlike CLAUDE.md instructions which are advisory, hooks are deterministic and guarantee the action happens.
Claude can write hooks for you. Try prompts like “Write a hook that runs eslint after every file edit” or “Write a hook that blocks writes to the migrations folder.” Edit .claude/settings.json directly to configure hooks by hand, and run /hooks to browse what’s configured.
#### skills
Create SKILL.md files in .claude/skills/ to give Claude domain knowledge and reusable workflows.

Skills extend Claude’s knowledge with information specific to your project, team, or domain. Claude applies them automatically when relevant, or you can invoke them directly with /skill-name.

Create a skill by adding a directory with a SKILL.md to .claude/skills/:

<img width="1125" height="362" alt="image" src="https://github.com/user-attachments/assets/fbdf1924-5903-46fe-8c9d-ff45fd619c3b" />

<img width="1125" height="596" alt="image" src="https://github.com/user-attachments/assets/3d3170e3-1eb3-44c6-824d-fa6d502f877c" />

Run /fix-issue 1234 to invoke it. Use disable-model-invocation: true for workflows with side effects that you want to trigger manually.

#### create custom subagents
Define specialized assistants in .claude/agents/ that Claude can delegate to for isolated tasks.

Subagents run in their own context with their own set of allowed tools. They’re useful for tasks that read many files or need specialized focus without cluttering your main conversation.

<img width="1138" height="497" alt="image" src="https://github.com/user-attachments/assets/c78526ba-2561-47b5-b3c6-146e8984d04e" />

Tell Claude to use subagents explicitly: “Use a subagent to review this code for security issues.”

#### plugins
Run /plugin to browse the marketplace. Plugins add skills, tools, and integrations without configuration.

Plugins bundle skills, hooks, subagents, and MCP servers into a single installable unit from the community and Anthropic.

## Other

### Write Specs
For larger features, have Claude interview you first. Start with a minimal prompt and ask Claude to interview you using the AskUserQuestion tool. Start a fresh session to execute the spec just completed.

The most useful specs are self-contained: they name the files and interfaces involved, state what is out of scope, and end with an end-to-end verification step that proves the feature works. Time spent making the spec precise pays off more than time spent watching the implementation.

### manage sesssion

#### manage context use /clear, /compact xxx , /btw 
Use /clear between related tasks to reset context.

When auto compaction triggers, Claude summarizes what matters most, including code patterns, file states, and key decisions
For more control, run /compact <instructions>, like /compact Focus on the API changes - 就是他的那个 compact 呀，也可以去指导他 focus on 某些事情。

To compact only part of the conversation, use Esc + Esc or /rewind, select a message checkpoint, and choose Summarize from here or Summarize up to here. The first condenses messages from that point forward while keeping earlier context intact; the second condenses earlier messages while keeping recent ones in full. See the rewind menu’s summarize options. - 这个之后看吧，确实没做过。Anthropic 在控制这方面，确实把很大的 Autonomy（自主权）都交给了用户。那这样的话，其实我就能明白为什么他们的公司文化一直都在强调对 CLAUDE.md 的控制。整个企业文化就是这样，非常 hacker 文化，把自主权交出来，让用户能够去选择、去做这样的控制，而不是直接包圆了。

Customize compaction behavior in CLAUDE.md with instructions like "When compacting, always preserve the full list of modified files and any test commands" to ensure critical context survives summarization

For questions that don’t need to stay in context, use /btw. The answer never enters conversation history, so you can check a detail without growing context. - 哎，这个好，这个确实好！之前确实不知道，长见识了，还能这么干呢！我问过不少这种问题

#### use subagents for investigation

这个主要是因为，尤其是对一个 code base 做调查的时候，因为它读的文件太多了，context window 禁不住这么挥霍。所以就是让 sub-agent 去干，免得影响到主 conversation 嘛

#### Rewind with checkpoints

Claude automatically snapshots files before each change so a checkpoint can restore them. Double-tap Escape or run /rewind to open the rewind menu. You can restore conversation only, restore code only, restore both, or summarize from a selected message. See Checkpointing for details.

Instead of carefully planning every move, you can tell Claude to try something risky. If it doesn’t work, rewind and try a different approach. Checkpoints are saved with the conversation, so you can close your terminal, resume the session later, and still rewind. (Checkpoints only track changes made through Claude’s file editing tools. Changes made through Bash commands or external processes are not captured. This isn’t a replacement for git.)

-这个也好，它能自带一个这样的保存。这点起码比 Codex 好，Codex 让他去回撤的时候，要是那个时候没有做git，他确实回撤不了。

### Automate and scale

With claude -p "your prompt", you can run Claude non-interactively, without an interactive prompt. The run still creates a resumable session unless you pass --no-session-persistence. Non-interactive mode is how you integrate Claude into CI pipelines, pre-commit hooks, or any automated workflow. The output formats let you parse results programmatically: plain text, JSON, or streaming JSON.

这个就是把 Claude 作为一个本地 API 去理解，把它串在工作流里面。这个时候会用，也挺常用的。

One-off queries:

claude -p "Explain what this project does"

Structured output for scripts: claude -p "List all API endpoints" --output-format json

Streaming for real-time processing: claude -p "Analyze this log file" --output-format stream-json --verbose

The first command prints plain text. The json format returns a single JSON object with a result field. The stream-json format prints one JSON object per line, starting with an init event.

### run multiple claude sessions

其实意思就跟 orchestration 差不多，跟通用的一些 orchestration 差不多。

这种东西无非就两种：

1. 让他们同时站在同一个起点，各自往不同的方向去做实验
2. 做一些编排，让另外一个 model 作为 judge 去跑这个 loop，以此来保证最终的结果
3. 然后是一些 handoff 呀，还有一些让一个 manager 来去管所有 submission 的结果呀什么的，就还是那种 orchestration 的东西啊

其实也没有什么新奇的东西，大家都这样

#### Worktrees 
run separate CLI sessions in isolated git checkouts so edits don’t collide

这个是用在对同一件事情有很多实验性 idea 的时候：让他们同时朝着多个方向去做，但不会互相乱写，每个人都是一份分开的东西

#### Cross-session messaging: let the sessions you run yourself pass findings to each other - 这个有用，但是回头看

#### Agent teams: experimental and disabled by default. Automated coordination of multiple sessions with shared tasks, messaging, and a team lead

#### /batch xxx to distribute work across many parallel Claude invocations

这个适用在那种很大的 migration，或者说范围很广、各种杂事破事很多的 analysis 上。

