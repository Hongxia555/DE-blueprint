# everything-claude-code — What It Is and How It Works

## What Is It?

A production-ready **agent harness performance system** for Claude Code and other AI agents — built from 10+ months of intensive daily use, winner of an Anthropic hackathon. Provides a complete system: skills, instincts, memory persistence, continuous learning, security scanning, and research-first development workflows.

**Repo:** `Repos/Repo Bank/everything-claude-code/`
**GitHub:** `https://github.com/affaan-m/everything-claude-code`
**Stars:** 50K+ | **Forks:** 6K+ | **License:** MIT
**npm:** `ecc-universal` (installable as a package)

---

## Core Purpose

> "Not just configs. A complete system."

ECC frames itself as a **harness performance system** — it doesn't change how Claude works, it changes how well the harness (Claude Code, Codex, Cowork, etc.) performs on real tasks. The primary levers are: token optimization, memory persistence, and continuous learning from sessions.

---

## What's in the Repo

| Folder | Contents |
|---|---|
| `agents/` | 30+ specialized subagents (code reviewer, python-reviewer, typescript-reviewer, tdd-guide, harness-optimizer, etc.) |
| `skills/` | 135+ workflow skills (coding standards, testing patterns, CI workflows, MCP server patterns, language-specific rules) |
| `commands/` | 60+ slash commands (`/tdd`, `/plan`, `/learn`, `/skill-create`, `/code-review`, `/build-fix`, etc.) |
| `hooks/` | Trigger-based automations (session persistence, pre/post-tool hooks, memory explosion prevention) |
| `rules/` | Always-follow guidelines (security, coding style, testing, language-specific rules for 12 ecosystems) |
| `mcp-configs/` | MCP server configurations for external integrations |
| `scripts/` | Cross-platform Node.js utilities for hooks and setup |
| `tests/` | Test suite for scripts and utilities |

---

## Key Systems

### Continuous Learning + Instincts
- After each session, `/learn` extracts reusable patterns as **atomic instincts** with confidence scoring
- Instincts accumulate over time; `/evolve` promotes high-confidence ones into permanent rules
- `/instinct-status` shows all learned instincts with scores
- Memory persistence hooks save/load context automatically across sessions

### Token Optimization
- Model selection guidance (when to use Haiku vs Sonnet vs Opus)
- System prompt slimming — keep CLAUDE.md under 200 lines
- Background process patterns — offload expensive work to subagents
- `context: fork` skills run in isolated subagent; main context stays clean

### Security (AgentShield)
- `ecc-agentshield` npm package: injection detection, sandboxing, sanitization
- Security guide covers attack vectors, CVEs, sandbox setup
- Pre-tool hooks scan for dangerous patterns before execution

### Verification Loops
- Checkpoint evals vs continuous evals
- Grader types and pass@k metrics
- Second-Claude pattern: one agent writes, another (fresh context, same model) reviews

### Parallelization
- Git worktrees for isolated parallel development
- Cascade method for multi-agent coordination
- When to scale instances vs when a single session is better

---

## Key Commands

| Command | What it does |
|---|---|
| `/learn` | Extract reusable patterns from the current session into instincts |
| `/learn-eval` | Same as `/learn` but self-evaluates session quality first |
| `/evolve` | Review accumulated instincts, suggest promotions |
| `/skill-create` | Auto-generate SKILL.md files from git history |
| `/tdd` | Test-driven development: scaffold → tests → implement → verify |
| `/plan` | Implementation planning before coding |
| `/code-review` | Security + quality review of uncommitted changes |
| `/harness-audit` | Audit Claude Code config for reliability and cost |
| `/orchestrate` | Multi-agent orchestration patterns |

---

## Language Ecosystem Support

Rules and patterns for 12 languages: TypeScript, Python, Go, Java, Kotlin, PHP, Perl, C++, Rust, Node.js, common rules, and more.

---

## How It Differs from Other Repos in Repo Bank

| | `everything-claude-code` | `claude-code-best-practice` | `paperclip` |
|---|---|---|---|
| **Focus** | Optimize a single agent harness end-to-end | Tips and architecture patterns for Claude Code | Orchestrate a company of agents |
| **Format** | Installable system (npm, selective install) | Reference/learning material | Running server + UI |
| **Key value** | Instinct learning, token savings, security, memory | Mindset, workflow patterns, community workflows | Org charts, budgets, governance, ticketing |
| **Install** | `npx ecc-universal install` | Read and adapt manually | `npx paperclipai onboard` |
| **Who it's for** | Power users wanting systematic harness improvement | Any Claude Code user wanting better results | Anyone running multi-agent autonomous businesses |

---

## Notes on This Workspace

The WOW workspace already imports many ECC components — agents (`chief-of-staff`, `python-reviewer`, `doc-updater`, etc.), skills (`continuous-learning-v2`, `mcp-server-patterns`), and commands (`/learn`, `/evolve`, `/skill-create`). The `Repo Bank/everything-claude-code/` copy is the source for those imports.
