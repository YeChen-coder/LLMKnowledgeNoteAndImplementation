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




