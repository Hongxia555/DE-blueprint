# paperclip — What It Is and How It Works

## What Is It?

An open-source **AI company orchestration platform** — a Node.js server + React UI that acts as the company layer above individual AI agents (Claude Code, Codex, Cursor, OpenClaw, etc.). You define a business goal, hire AI agents into roles (CEO, CTO, engineer, marketer), set budgets, and let them run autonomously.

**Repo:** `Repos/Repo Bank/paperclip/`
**GitHub:** `https://github.com/paperclipai/paperclip`
**License:** MIT

---

## Core Concept

> "If OpenClaw is an _employee_, Paperclip is the _company_."

Paperclip is not a chatbot, workflow builder, or prompt manager. It models a real organization — org charts, goal hierarchies, budgets, governance, and a ticket system — where every employee is an AI agent.

**The three-step loop:**

| Step | What you do |
|---|---|
| **01 Define the goal** | "Build the #1 AI note-taking app to $1M MRR." |
| **02 Hire the team** | CEO, CTO, engineers, marketers — any bot, any provider |
| **03 Approve and run** | Review strategy, set budgets, hit go, monitor from dashboard |

---

## Key Concepts

| Concept | What it means |
|---|---|
| **Company** | A fully isolated org — its own agents, goals, data, and audit trail |
| **Heartbeat** | Scheduled pulse that wakes an agent to check and act on work |
| **Goal alignment** | Every task carries full goal ancestry — agents always know the *why* |
| **Ticket system** | Every conversation traced, every decision explained, immutable audit log |
| **Budget enforcement** | Per-agent monthly token budgets; agents stop when they hit the limit |
| **Clipmart** *(coming)* | Marketplace of pre-built company templates importable in one click |

---

## Problems It Solves

| Without Paperclip | With Paperclip |
|---|---|
| 20 Claude Code tabs open, lose track on reboot | Ticket-based tasks, threaded conversations, sessions persist |
| Manually gather context to remind each agent what to do | Context flows from task → project → company goal automatically |
| Runaway loops burn hundreds of dollars unnoticed | Per-agent budget cap throttles agents when quota is hit |
| Recurring jobs (reports, support) require manual kicks | Heartbeats handle recurring work on a schedule |

---

## What It's Not

- Not a single-agent tool — designed for teams of 10–20+ agents
- Not an agent framework — doesn't tell you how to build agents
- Not a code review tool — orchestrates work, not pull requests
- Not a chatbot — agents have jobs, not chat windows

---

## Tech Stack

- **Backend:** Node.js (TypeScript), embedded PostgreSQL (no setup required for local), REST API on port 3100
- **Frontend:** React UI, mobile-ready
- **Agent support:** HTTP adapter (any HTTP agent), process adapter (subprocess), OpenClaw, Claude Code, Codex, Cursor, Bash

**Quickstart:**
```bash
npx paperclipai onboard --yes
# or
git clone https://github.com/paperclipai/paperclip.git && cd paperclip && pnpm install && pnpm dev
```

---

## Key Files in Repo

| Path | What it is |
|---|---|
| `cli/` | The `paperclipai` CLI — onboarding, config, company management |
| `cli/src/adapters/` | Agent adapters: HTTP (any agent via webhook) and Process (subprocess) |
| `cli/src/checks/` | Doctor checks: DB, LLM, auth, config, JWT |
| `.agents/skills/` | Paperclip-specific agent skills (company-creator, pr-report, release, doc-maintenance) |
| `.claude/skills/design-guide/` | UI design system guide for the React frontend |
| `AGENTS.md` | Agent configuration guide for Paperclip-aware agents |

---

## How It Differs from Other Repos in Repo Bank

| | `paperclip` | `everything-claude-code` | `claude-code-best-practice` |
|---|---|---|---|
| **Focus** | Orchestrate a company of agents | Optimize a single Claude Code harness | Best practices and patterns for Claude Code |
| **Scope** | Multi-agent, multi-company, org-level | Single harness, session-level | Tips, architecture patterns |
| **Who it's for** | Running autonomous AI businesses | Power users squeezing performance | Any Claude Code user |
| **Value** | Org charts, budgets, governance, ticketing | Token savings, memory, instincts, security | Mindset and workflow patterns |
