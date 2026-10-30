> English version of [ClaudeCodeBlog.md](../ClaudeCodeBlog.md). Original notes are written in Chinese/English mix.

few-shot is not the cold start from recommendation systems (cold start means that when a new user doesn't have much history to make recommendations from, you find a group of similar users and use collaborative filtering),

few-shot just means giving examples in the prompt, what format to output in, that's all. - Isn't that how it's always been done?

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
For more control, run /compact <instructions>, like /compact Focus on the API changes - So with compact, you can also guide it to focus on certain things.

To compact only part of the conversation, use Esc + Esc or /rewind, select a message checkpoint, and choose Summarize from here or Summarize up to here. The first condenses messages from that point forward while keeping earlier context intact; the second condenses earlier messages while keeping recent ones in full. See the rewind menu’s summarize options. - I'll look at this later; I really haven't done it. On control, Anthropic really does hand a lot of autonomy to the user. That helps me understand why their company culture keeps emphasizing control over CLAUDE.md. That's the whole company culture: very hacker-like, giving up autonomy so users can choose and exercise this kind of control, rather than doing everything for them.

Customize compaction behavior in CLAUDE.md with instructions like "When compacting, always preserve the full list of modified files and any test commands" to ensure critical context survives summarization

For questions that don’t need to stay in context, use /btw. The answer never enters conversation history, so you can check a detail without growing context. - Oh, this is good, really good! I didn't know this before. Learned something new; you can actually do that! I've asked plenty of questions like this.

#### use subagents for investigation

This is mainly because, especially when investigating a codebase, it reads so many files that the context window can't afford that kind of waste. So you let a sub-agent do it, so it doesn't affect the main conversation.

#### Rewind with checkpoints

Claude automatically snapshots files before each change so a checkpoint can restore them. Double-tap Escape or run /rewind to open the rewind menu. You can restore conversation only, restore code only, restore both, or summarize from a selected message. See Checkpointing for details.

Instead of carefully planning every move, you can tell Claude to try something risky. If it doesn’t work, rewind and try a different approach. Checkpoints are saved with the conversation, so you can close your terminal, resume the session later, and still rewind. (Checkpoints only track changes made through Claude’s file editing tools. Changes made through Bash commands or external processes are not captured. This isn’t a replacement for git.)

-This is good too; it comes with this kind of saving built in. At least that's better than Codex: when you ask Codex to roll back and there was no git commit at that point, it really can't.

### Automate and scale

With claude -p "your prompt", you can run Claude non-interactively, without an interactive prompt. The run still creates a resumable session unless you pass --no-session-persistence. Non-interactive mode is how you integrate Claude into CI pipelines, pre-commit hooks, or any automated workflow. The output formats let you parse results programmatically: plain text, JSON, or streaming JSON.

Think of this as using Claude as a local API, chained into a workflow. I use it in these cases, and quite often.

One-off queries:

claude -p "Explain what this project does"

Structured output for scripts: claude -p "List all API endpoints" --output-format json

Streaming for real-time processing: claude -p "Analyze this log file" --output-format stream-json --verbose

The first command prints plain text. The json format returns a single JSON object with a result field. The stream-json format prints one JSON object per line, starting with an init event.

### run multiple claude sessions

This basically means the same as orchestration, like general-purpose orchestration.

This kind of thing comes in just two flavors:

1. Have them all start from the same point and experiment in different directions
2. Do some orchestration, with another model acting as judge running the loop, to ensure the final result
3. Then some handoffs, or having a manager oversee the results of all submissions, and so on; still the usual orchestration stuff

Nothing really new; everyone does it this way.

#### Worktrees 
run separate CLI sessions in isolated git checkouts so edits don’t collide

This is for when you have many experimental ideas about the same thing: have them work in several directions at once without overwriting each other; each one gets its own separate copy.

#### Cross-session messaging: let the sessions you run yourself pass findings to each other - This is useful, but I'll look at it later

#### Agent teams: experimental and disabled by default. Automated coordination of multiple sessions with shared tasks, messaging, and a team lead

#### /batch xxx to distribute work across many parallel Claude invocations

This fits very large migrations, or analyses with a broad scope and lots of miscellaneous chores.
