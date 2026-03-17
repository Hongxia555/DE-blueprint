# Claude Code Tips

## Setup

- Prefer **terminal** (`claude` command) or **IDE extension** (Windsurf/VSCode) over the browser chat — both give automatic file context with no copy-paste
- Use configuration files for persistent rules — three levels of scope:
  - `CLAUDE.md` — main project file, generated with `/init`, committed to source control, shared with the whole team
  - `CLAUDE.local.md` — personal project-level file, not shared, for your own preferences and customizations
  - `~/.claude/CLAUDE.md` — global file, applies to all projects on your machine, best for universal rules (e.g. "never commit without asking")
- There are **two** `.claude/` folders — global and per-project:

**`~/.claude/`** — global, applies to all projects on your machine
```
~/.claude/
├── CLAUDE.md                          # universal rules (coding style, commit habits)
└── skills/
    └── meeting-notes/                 # personal skills available in every project
        └── SKILL.md
```

**`<project>/.claude/`** — per-project knowledge base and project-specific skills
```
project/
├── CLAUDE.md                          # references knowledge base files
├── CLAUDE.local.md
└── .claude/
    ├── business-info.md               # company/product/data context
    ├── writing-styles/
    │   ├── technical.md               # tone for code docs, PR descriptions
    │   └── executive-summary.md       # tone for stakeholder comms
    ├── examples/
    │   ├── sql-patterns.md            # preferred query patterns (CTEs, window functions, etc.)
    │   ├── pipeline-template.py       # boilerplate pipeline structure
    │   └── dbt-model-template.sql     # standard dbt model layout
    ├── meeting-transcripts/
    │   └── 2024-01-15-kickoff.md
    └── skills/
        └── prd-review/                # project-specific skills
            └── SKILL.md
```

**Rule of thumb:**

| What | Where |
|---|---|
| Universal rules (coding style, commit habits) | `~/.claude/CLAUDE.md` |
| Skills used across all projects | `~/.claude/skills/` |
| Project context (data model, business info) | `project/.claude/` |
| Project-specific skills | `project/.claude/skills/` |

**Reference knowledge base files in `CLAUDE.md`:**
```markdown
Read @.claude/business-info.md for company and data context.
When writing SQL, follow patterns in @.claude/examples/sql-patterns.md.
Match the tone in @.claude/writing-styles/technical.md for all code docs.
```

### Environment Comparison

| | Browser Chat | Terminal | IDE Extension (Windsurf/VSCode) |
|---|---|---|---|
| Shell / run commands | No | Yes | Limited |
| Sees open file/selection | No | No | Yes |
| Parallel agents | No | Best here | Possible |
| Inline diff review | No | No | Yes |
| Automation / scripts | No | Best here | Not ideal |

- **Terminal** — best for automation, parallel agents, shell-heavy tasks
- **IDE Extension** — best for file editing; sees your current open file and selection
- **Browser Chat** — avoid for engineering work; no file context, requires manual copy-paste

---

## Workflow

- Save reusable prompts as **skills** (formerly "slash commands") — invoke with `/skill-name` (e.g. `/meeting-notes`, `/prd-review`)
  - **Personal skill** (all projects): `~/.claude/skills/<name>/SKILL.md`
  - **Project skill** (this project only): `.claude/skills/<name>/SKILL.md`
  - Claude can also auto-invoke skills when it detects they're relevant
- Use **Plan Mode** (`Shift+Tab`) before complex tasks to preview before executing
- Run parallel agents for simultaneous work (e.g. 3 interviews → 3 agents at once) — **terminal only**; spawn multiple `claude` processes in separate tabs/panes
- Build specialized **subagents** for focused review tasks — **terminal only**; each runs in its own isolated context with restricted tools

**How to build one** — create a `.md` file with YAML frontmatter + system prompt:
```markdown
---
name: engineering-reviewer
description: Reviews code for quality, security, and performance. Use after code changes.
tools: Read, Grep, Glob, Bash
model: sonnet
---

You are a senior engineering reviewer. When invoked:
1. Run git diff to see recent changes
2. Check for security issues, performance problems, test coverage
3. Return feedback categorized by severity
```

**Where they live:**

| Location | Scope |
|---|---|
| `~/.claude/agents/` | All your projects (personal) |
| `.claude/agents/` | This project only (shareable with team) |

**How to invoke:**
- Explicit: *"Use the engineering-reviewer to check this PR"*
- Implicit: Claude auto-delegates based on the `description` field
- Parallel: *"Review this from all angles"* → runs multiple subagents at once

**Key difference from skills:** Subagents run in their own isolated context window — they can't accidentally affect your main session

---

## Recommended Skills

### Personal skills — `~/.claude/skills/` (all projects)

| Skill | What it does |
|---|---|
| `/sql-review` | Review SQL for style, performance, and correctness |
| `/commit` | Generate a commit message from staged diff |
| `/explain` | Explain a piece of code or concept in plain English |
| `/standup` | Draft a standup update from recent git activity |

### DE project skills — `.claude/skills/` (per project)

| Skill | What it does |
|---|---|
| `/dbt-model` | Scaffold a new dbt model with standard structure + tests |
| `/pipeline-design` | Design a pipeline given source/target requirements |
| `/data-quality` | Generate dbt tests or Great Expectations checks for a model |
| `/incident-report` | Structure a data incident/outage post-mortem |
| `/meeting-notes` | Format a raw transcript into structured notes + action items |
| `/prd-review` | Review a data requirements doc for completeness and edge cases |

### Interview prep skills — `Job_Search/.claude/skills/`

| Skill | What it does |
|---|---|
| `/metric-breakdown` | Given a metric drop, outline the investigation framework |

---

## Cost & Scaling

- Budget stack: Claude Pro ($17/mo) + Cursor ($20/mo) — no need for the $200 plan
- Monitor token usage with built-in commands:

| Command | What it does |
|---|---|
| `/cost` | Shows total cost, API duration, lines changed for the session |
| `/stats` | Usage patterns (Pro/Max subscribers) |
| `/context` | Shows what's consuming your token budget (files, history, skills, MCP tools) per session|
| `/statusline` | Persistent status bar with context %, cost, model in real-time |

- Manage context to control costs:
  - `/clear` — wipes context, start fresh (use between unrelated tasks)
  - `/rename <name>` before clearing — saves session so you can `/resume` it later
  - `/compact <focus>` — compresses context but keeps a summary (better than clearing mid-task)
- Keep `CLAUDE.md` under 500 lines — move specialized instructions to skills so they load on-demand
- Use subagents for verbose operations (test runs, log processing) — keeps their output out of your main context
- Ramp up week by week — don't skip steps, each builds on the last:

| Week | Focus | What to do |
|---|---|---|
| 1 | **File reading** | Use Claude to read and explain existing code. No writing yet. Goal: get comfortable with how Claude navigates your project |
| 2 | **Knowledge base** | Set up `.claude/` with `business-info.md`, examples, writing styles. Add `@` references in `CLAUDE.md`. Goal: Claude has project context automatically every session |
| 3 | **Skills** | Create 1-2 skills for repetitive tasks (`/commit`, `/sql-review`). Goal: stop copy-pasting prompts |
| 4 | **Parallel agents** | Open multiple terminal tabs, run separate `claude` processes for independent tasks. Goal: compress multi-hour tasks into parallel workstreams |
| 5 | **Subagents** | Build specialized subagents with restricted tools and focused system prompts. Goal: delegate review tasks to isolated agents that run automatically |
