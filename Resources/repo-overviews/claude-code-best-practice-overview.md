# claude-code-best-practice — What It Is and How It Works

## What Is It?

A community-curated reference repo for getting the most out of Claude Code — covering architecture patterns, best practices, tips from the Claude Code creator (Boris Cherny), and real-world development workflows. Originally by [@shanraisshan](https://github.com/shanraisshan), trended #1 on GitHub.

**Repo:** `Repos/claude-code-best-practice/`

---

## The Core Architecture: Commands → Agents → Skills

The repo's central insight is a three-layer orchestration pattern:

```
Command (user triggers /weather-orchestrator)
    │
    ▼
Agent (breaks task into subtasks, runs in fresh context)
    │
    ▼
Skills (domain knowledge auto-injected as needed)
```

| Layer | Lives in | What it is |
|---|---|---|
| **Skills** | `.claude/skills/<name>/SKILL.md` | Knowledge injected into context — auto-discoverable, preloadable, supports context forking |
| **Commands** | `.claude/commands/<name>.md` | User-invoked prompt templates for workflow orchestration (`/commit`, `/review`) |
| **Subagents** | `.claude/agents/<name>.md` | Autonomous actors in a fresh isolated context — own tools, permissions, model, memory |
| **Plugins** | distributable packages | Bundles of the above — skills + agents + hooks + MCP servers, installable across teams |

---

## What's in the Repo

| Folder | Contents |
|---|---|
| `best-practice/` | Deep dives: subagents, commands, skills, MCP, settings, memory, CLI flags |
| `implementation/` | Working implementations of each concept |
| `orchestration-workflow/` | The Command → Agent → Skill pattern with diagrams and demo |
| `development-workflows/` | Cross-model workflow, RPI workflow, parallel agent setups |
| `tips/` | Boris Cherny's tip collections (13 tips, 10 tips, 12 tips) |
| `reports/` | Deep-dives: skills in monorepos, AI terminology, comparisons |
| `agent-teams/` | Multi-agent parallel development patterns |
| `videos/` | Video references and demos |

---

## Key Concepts Explained

### Subagents vs Commands vs Skills

**Commands** — simple, user-invoked. Inject a prompt template into the existing context. Best for inner-loop workflows you do many times a day.
> Boris: "use commands for every workflow you do more than once a day"

**Subagents** — run in a **fresh, isolated context**. They can have their own model, tools, and permissions. Main context only sees the final result.
> Boris: "say 'use subagents' to throw more compute at a problem"

**Skills** — like commands but smarter: auto-discoverable (Claude fires them when relevant), support `context: fork` to run in isolation, and can include `references/`, `scripts/`, `examples/` subdirectories.

### Hot Features (as of early 2026)

| Feature | What it does |
|---|---|
| **Agent Teams** | Multiple agents working in parallel on the same codebase with shared task coordination |
| **Git Worktrees** | Isolated git branches per agent — parallel development without conflicts |
| **Channels** | Push events from Telegram/Discord/webhooks into a running Claude session |
| **Code Review** | Multi-agent PR analysis via GitHub App |
| **Scheduled Tasks** | Run prompts on a recurring schedule (`/loop`) |
| **Remote Control** | Continue local sessions from phone/tablet/browser |
| **Ralph Wiggum Loop** | Autonomous dev loop that iterates until task is complete |

---

## Top Tips (selected from 84)

**Prompting**
- Challenge Claude: *"grill me on these changes and don't make a PR until I pass your test"*
- After a mediocre fix: *"knowing everything you know now, scrap this and implement the elegant solution"*

**Planning**
- Always start with plan mode before coding
- Spin up a second Claude to review your plan as a staff engineer
- Prototype 20-30 versions instead of writing long specs — cost of building is low

**CLAUDE.md**
- Keep under 200 lines per file
- Wrap critical rules in `<important if="...">` tags so Claude doesn't ignore them as files grow
- Any developer should be able to type "run the tests" and it works on the first try

**Skills**
- The `description` field is a trigger, not a summary — write it as "when should I fire?"
- Build a **Gotchas** section in every skill — highest-signal content
- Don't be prescriptive — give goals and constraints, not step-by-step instructions
- Use `context: fork` to run a skill in isolated subagent; main context only sees the final result

**Agents**
- One agent causes bugs, another (same model, fresh context) finds them — use test-time compute
- Separate contexts make results better; subagents keep your main context clean

---

## Development Workflows Catalogued

The repo maps 8 major community workflows by star count and distinctive features:

| Workflow | Stars | What's unique |
|---|---|---|
| Superpowers | 103k | TDD-first, Iron Laws, whole-plan review |
| Everything Claude Code | 93k | Instinct scoring, AgentShield |
| Spec Kit (GitHub) | 79k | Spec-driven, 22+ tools |
| BMAD-METHOD | 42k | Full SDLC, agent personas |
| Get Shit Done | 38k | Fresh 200K contexts, wave execution |
| gstack | 34k | Role personas, parallel sprints |
| OpenSpec | 33k | Delta specs, brownfield, artifact DAG |
| HumanLayer | 10k | Context engineering, 300k+ LOC |

---

## How It Differs from Other Repos

| | `claude-code-best-practice` | `claude-plugins-official` | `knowledge-work-plugins` |
|---|---|---|---|
| **Focus** | How to use Claude Code well | Dev tools + SaaS integrations | Role-based plugins (data, legal, etc.) |
| **Format** | Tips, patterns, best practices | Installable plugins | Installable plugins |
| **Who it's for** | Any Claude Code user | Developers | Knowledge workers |
| **Value** | Mindset + architecture patterns | Live tool integrations | Domain expertise |
