# Claude Code Built-In Commands

A ranked reference of the most useful built-in slash commands that ship with Claude Code.

> **Before adding a custom command:** check this file first. Many useful commands are already built in and don't need to be imported from external sources.

---

## WOW Custom Commands (installed, not built-in)

Commands added to `WOW/.claude/commands/` from [everything-claude-code](https://github.com/Hongxia555/everything-claude-code):

| Command | What it does |
|---|---|
| `/learn` / `/learn-eval` | Extract reusable patterns from a session into instincts |
| `/evolve` | Review and improve accumulated instincts |
| `/instinct-status` / `/instinct-export` / `/instinct-import` | Manage the instinct learning system |
| `/skill-create` | Auto-generate skills from git history |
| `/skill-health` | Portfolio dashboard — see skill gaps |
| `/rules-distill` | Distill topic principles into a rules file |
| `/update-docs` | Keep documentation current after study sessions |
| `/checkpoint` | Mark named progress point mid-task |
| `/code-review` | Security + quality review of uncommitted changes |
| `/python-review` | PEP 8, type hints, security, Pythonic idioms |
| `/prompt-optimize` | Analyze and improve a draft prompt |
| `/tdd` | Test-driven development workflow |
| `/harness-audit` | Audit Claude Code config for reliability and cost |
| `/orchestrate` | Multi-agent orchestration patterns |

---

## Core Commands (Daily Drivers)

| Command | What it does |
|---|---|
| `/model` | Switch between Claude models (Sonnet, Opus, Haiku) mid-session |
| `/resume` | Continue a previous conversation by ID or name |
| `/config` | Open settings UI — theme, model, output style, preferences |
| `/rewind` | Restore code/conversation to a previous state — press `Esc` twice to access the undo menu |
| `/cost` | Show token usage and spend for the session |
| `/compact [focus]` | Compress conversation history to free up context window; optionally focus the summary |
| `/clear` | Reset conversation and free context entirely (aliases: `/reset`, `/new`) |

---

## High-Value Commands

| Command | What it does |
|---|---|
| `/plan` | Enter read-only plan mode — safely explore and analyze code without accidental edits (`Shift+Tab` toggles) |
| `/init` | Create a `CLAUDE.md` project documentation file that Claude reads at the start of every session |
| `/chrome` | Connect to your Chrome browser for live web testing and automation (requires Chrome extension v1.0.36+) |
| `/agents` | Create, configure, and manage specialized subagents with specific tools and models |
| `/memory` | Edit CLAUDE.md, enable/disable auto-memory, view stored memory entries |
| `/diff` | Interactive diff viewer for turn-by-turn code changes in the session |
| `/permissions` | View and update tool permissions mid-session |
| `/btw <question>` | Ask a quick side question without cluttering conversation history — reads full context, no tools |
| `/add-dir <path>` | Expand Claude's working context to include additional directories (session only) |
| `/fork` | Branch the current conversation to experiment safely without losing your main thread |

---

## Utilities

| Command | What it does |
|---|---|
| `/context` | Visualize what's consuming your context budget (files, history, skills, MCP tools) |
| `/hooks` | View and manage hook configurations for tool events |
| `/fast` | Toggle fast mode — cheaper, faster responses for the session |
| `/effort <level>` | Set model effort (low / medium / high / max); max only available on Opus |
| `/rename <name>` | Name the current session — shows in prompt bar, useful before `/clear` so you can `/resume` later |
| `/export` | Export the conversation as a plain text file |
| `/doctor` | Diagnose and verify Claude Code installation and settings |
| `/mcp` | Manage MCP server connections and OAuth |
| `/ide` | Manage IDE integrations (VSCode, Windsurf) and check status |
| `/debug [description]` | Read the session debug log; optionally describe the issue to focus the analysis |
| `/status` | Show version, model, account, and connectivity status |
| `/release-notes` | View changelog with most recent version first |

---

## Built-In Skills (Bundled)

These ship with Claude Code and behave like skills (not fixed commands) — they can spawn agents and adapt to your codebase.

| Skill | What it does |
|---|---|
| `/batch <instruction>` | Decomposes a large change into 5–30 independent units, spawns background agents in isolated git worktrees, each opens a PR |
| `/simplify [focus]` | Reviews recently changed files for quality/efficiency issues; spawns parallel review agents and applies fixes |
| `/loop [interval] <prompt>` | Runs a prompt repeatedly on an interval — useful for polling deploys, babysitting PRs (max 3 days) |
| `/claude-api` | Loads Claude API / Anthropic SDK reference for your language; auto-triggers when code imports `anthropic` or `@anthropic-ai/sdk` |
| `/commit` | Generates a conventional commit message from the current staged diff |
| `/standup` | Drafts a standup update (Yesterday / Today / Blockers) from recent git activity |

---

## Underrated Commands Worth Learning

| Command | Why it's useful |
|---|---|
| `/plan` | Prevents accidental edits while exploring an unfamiliar codebase |
| `/fork` | Lets you try a risky approach without losing your working conversation |
| `/context` | Shows exactly what's burning your token budget — useful for cost control |
| `/diff` | Better than running `git diff` manually; shows changes per turn |
| `/btw` | Lets you ask a quick clarifying question mid-task without polluting the thread |
| `/batch` | Parallelizes large refactors across many worktrees — massive time saver |

---

## Commands NOT in the CLI

| Feature | Where it actually lives |
|---|---|
| `/voice` | Claude.ai web and mobile app only — not available in the CLI |
| `/branch` | Not a built-in command; use `/fork` to branch conversations or `--worktree` CLI flag for git worktrees |

---

## Multi-Surface Commands

| Command | What it does |
|---|---|
| `/desktop` | Continue your session in the Claude Desktop app (macOS/Windows) |
| `/remote-control` / `/rc` | Enable remote control of the CLI session from claude.ai |
| `/mobile` / `/ios` / `/android` | Show QR code to download the Claude mobile app |

---

## MCP Prompt Commands

Commands in the format `/mcp__<server>__<prompt>` are dynamically discovered from connected MCP servers. Examples:

```
/mcp__github__search_repositories
/mcp__slack__post_message
/mcp__sentry__find_recent_errors
```

---

## Practical Workflow Tips

From community usage patterns — high-impact habits:

| Tip | Why |
|---|---|
| Run `/context` at session start | Most users are 15–20% through budget before typing a word (CLAUDE.md, skills, MCP tools all cost tokens) |
| Run `/compact` at ~50% context | Don't wait until 95% — compacting late loses more useful history |
| Run `/clear` between unrelated tasks | Old context from task A degrades output quality for task B |
| Use `/plan` before touching unfamiliar code | "5 minutes of planning saves an hour" — maps every step before making changes |
| Say "ultra think" for hard single tasks | Forces maximum reasoning depth (equivalent to high effort on a single prompt) |

---

## Key Shortcuts

| Shortcut | What it does |
|---|---|
| `Shift + Tab` | Cycles between **normal → auto-accept → plan** modes |
| `Esc` twice | Opens the rewind/undo menu mid-session |

---

## CLI Flags Worth Knowing

| Flag | What it does |
|---|---|
| `--dangerously-skip-permissions` | Removes all guardrails — Claude executes end-to-end without approval prompts. Use only in sandboxed/CI environments. |
| `--model` | Set model at launch (e.g., `--model claude-opus-4-6`) |
| `--add-dir <path>` | Add extra directory to context at launch |
| `--allowedTools` | Pre-approve specific tools at launch |
| `--worktree` | Run Claude in an isolated git worktree |

---

## Quick Reference: Command vs Setting vs CLI Flag

| Goal | In-session command | CLI flag | Settings |
|---|---|---|---|
| Switch model | `/model` | `--model` | `config` → model |
| Set effort | `/effort` | `--effort` | — |
| Add directory | `/add-dir` | `--add-dir` | — |
| Permissions | `/permissions` | `--allowedTools` | `settings.json` |
| Enable Chrome | `/chrome` | `--chrome` | — |
