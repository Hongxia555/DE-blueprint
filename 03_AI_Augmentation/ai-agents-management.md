# AI Agents: Management & Orchestration

> Continues from [ai-agents-fundamentals.md](ai-agents-fundamentals.md). Covers how to structure, coordinate, and operate systems with more than one agent.

---

## 1. Why Multi-Agent?

A single agent with tools works for most tasks — but it breaks down in three scenarios:

| Problem | What happens | Multi-agent solution |
|---|---|---|
| **Context limits** | Long tasks exceed the LLM's context window | Split into agents, each with a focused context |
| **Specialization** | One agent tries to do everything poorly | Dedicated agents per function (research, writing, QA) |
| **Parallelism** | Sequential processing is too slow | Fan out to multiple agents working simultaneously |

> **Default:** Always start with a single agent. Add agents only when you hit one of these walls.

---

## 2. Four Orchestration Patterns

### 2.1 Sequential (Chain)

Each agent's output is the next agent's input. Simple and predictable.

```
[Trigger] → [Agent A: Research] → [Agent B: Draft] → [Agent C: Edit] → [Output]
```

- Best for: linear pipelines, one dependency per step
- Risk: one failure breaks the whole chain

---

### 2.2 Parallel (Fan-Out / Fan-In)

A coordinator sends work to multiple agents simultaneously and collects results.

```
                   ┌→ [Agent B: Market research] ─┐
[Agent A: Coordinator] →|                           |→ [Agent E: Aggregator] → Output
                   └→ [Agent C: Competitor scan]  ─┘
                   └→ [Agent D: Financial data]   ─┘
```

- Best for: independent sub-tasks, when speed matters
- Risk: aggregator logic can be complex; partial failures need handling

---

### 2.3 Supervisor / Manager

One manager agent receives the goal, breaks it into tasks, routes them to specialists, and compiles results.

```
                         ┌→ [Research Agent]
[User Goal] → [Manager] ─|→ [Writing Agent]
                         └→ [QA Agent]
```

- Best for: complex goals with variable sub-tasks, business workflows
- This is the most common pattern in no-code tools (n8n, Zapier)
- Risk: manager prompt quality determines everything

---

### 2.4 Hierarchical

Nested supervisors — a top-level manager delegates to department managers, who delegate to workers.

```
[CEO Agent] → [Marketing Manager Agent] → [Copywriter Agent]
                                        → [Social Media Agent]
           → [Sales Manager Agent]      → [Lead Research Agent]
                                        → [Outreach Agent]
```

- Best for: large, complex systems modeling org structures
- Risk: high complexity, expensive, hard to debug — only for advanced use cases

---

## 3. Agent Handoffs

A handoff is when one agent passes control (and context) to another.

### What gets passed
| Data | Passed? | Notes |
|---|---|---|
| **Final output** | Always | The result of the current agent's work |
| **Intermediate scratchpad** | Sometimes | Internal reasoning steps — often stripped to save tokens |
| **System prompt** | No | Each agent has its own |
| **Tool call history** | Sometimes | Useful for the next agent to avoid re-doing work |
| **User context** | Yes (if relevant) | Original goal, user identity, constraints |

### Handoff best practices
- Pass only what the next agent needs — every extra token costs money
- Summarize intermediate outputs rather than passing full transcripts
- Include the original goal at every handoff so sub-agents don't lose sight of the objective

---

## 4. Shared State & Memory

In multi-agent systems, agents need a way to share information without passing it all in every prompt.

| Memory type | Where it lives | Access | Example |
|---|---|---|---|
| **In-prompt context** | LLM context window | Current agent only | Previous message in this session |
| **Shared scratchpad** | External variable / node | All agents read/write | n8n variable storing intermediate JSON |
| **Long-term store** | Vector DB or database | Any agent can query | Company research saved to Pinecone |
| **Structured state** | Key-value store | Any agent | Current task status, flags, counters |

> **Key principle:** Don't rely on one agent's memory being visible to another. Explicitly pass or store shared state.

---

## 5. Tool Scoping (Principle of Least Privilege)

Each agent should only have access to the tools it needs — no more.

| Agent | Tools it should have | Tools it should NOT have |
|---|---|---|
| Research Agent | Web search, read Google Sheets | Send email, write to database |
| Writing Agent | Read research output | Web search, send email |
| Outreach Agent | Send email, update CRM | Web search, read financials |
| QA Agent | Read all outputs | Write to any system |

**Why it matters:**
- Limits blast radius if an agent misbehaves or gets prompt-injected
- Makes agents easier to reason about
- Reduces cost (fewer tool calls attempted)

---

## 6. Error Handling

Multi-agent failures are more complex — an error in one agent can cascade.

### Failure modes
| Type | Description | Example |
|---|---|---|
| **Tool failure** | API call returns error or timeout | Weather API is down |
| **Hallucination** | Agent produces wrong output | Wrong company name in research |
| **Loop** | Agent calls the same tool repeatedly | Tries to fetch data, gets empty result, retries forever |
| **Cascade** | Upstream failure propagates downstream | Bad research → bad draft → bad email |

### Mitigation strategies
| Strategy | How |
|---|---|
| **Retry with backoff** | Retry failed tool calls 2-3 times with delay before failing |
| **Fallback agent** | If primary agent fails, route to a simpler backup |
| **Human-in-the-loop checkpoint** | Pause at high-risk steps and require human approval before proceeding |
| **Output validation** | Add a QA agent or structured output check before downstream agents consume results |
| **Max iterations** | Set a hard limit on how many tool calls an agent can make per run |

---

## 7. Observability & Token/Cost Monitoring

### Why costs compound in multi-agent systems

Each agent handoff = a new LLM API call. A 5-agent chain that processes 100 items per day = 500 API calls/day. At $0.01/call, that's $5/day or ~$150/month — before tools.

```
Single agent:  1 call per task
3-agent chain: 3 calls per task  (3x cost)
5-agent chain: 5 calls per task  (5x cost)
+ parallel:    even more simultaneous calls
```

### Token budgets

Set limits at the agent level to prevent runaway spend:

| Control | Where to set it | Example |
|---|---|---|
| **max_tokens** | LLM node config | Cap output at 500 tokens for a summarizer agent |
| **Max tool calls** | Agent node settings (n8n) | Max 5 tool calls per run |
| **Timeout** | Workflow trigger settings | Kill any run exceeding 30 seconds |
| **Run frequency** | Schedule trigger | Don't run every minute if hourly is sufficient |

### Monitoring tools

| Tool | What it tracks | Best for |
|---|---|---|
| **n8n execution logs** | Per-run logs, node-level duration, error messages | n8n workflows |
| **OpenAI usage dashboard** | Token usage and cost by API key, per model | Any OpenAI-based agent |
| **Langfuse** | Traces, per-agent token counts, latency, costs | Code-based agents (LangChain, etc.) |
| **LangSmith** | Full trace visualization, token/cost per step | LangChain/LangGraph specifically |

### Key metrics to track

| Metric | What it tells you |
|---|---|
| **Tokens / run** | How much context each run consumes |
| **Cost / run** | Dollar cost per task execution |
| **Cost / task type** | Compare agent cost to human baseline |
| **Error rate per agent** | Which agent fails most — target for improvement |
| **Latency / agent** | Which agent is the bottleneck |
| **Tool call count** | Are agents using tools efficiently or thrashing? |

---

## 8. When to Use Each Pattern

| Situation | Recommended pattern |
|---|---|
| Simple, linear workflow (research → write → send) | Sequential chain |
| Multiple independent tasks that can run at the same time | Parallel fan-out |
| Complex goal with variable sub-tasks | Supervisor / Manager |
| Large organization simulation or very complex system | Hierarchical |
| Everything fits in context, task isn't complex | Single agent (no multi-agent) |
| Near-perfect accuracy required | Single agent + human review checkpoint |

---

## 9. Real-World Example: Lead Research & Outreach System

A supervisor-pattern multi-agent system for B2B sales prospecting.

```
[New lead in CRM]
       ↓
[Manager Agent]
  - Reads lead name + company
  - Decides which sub-agents to call
       ↓
┌──────────────────────────────────┐
│  [Research Agent]                │  Tools: web search, LinkedIn scraper
│  Output: company summary JSON    │
│                                  │
│  [ICP Scoring Agent]             │  Tools: read ICP criteria from Sheets
│  Output: fit score (1-10)        │
└──────────────────────────────────┘
       ↓
[Manager Agent] — if score ≥ 7:
       ↓
[Outreach Agent]
  - Tools: Gmail
  - Writes personalized email from research summary
  - Sends email
       ↓
[CRM Update Agent]
  - Tools: update HubSpot record with score + email sent timestamp
```

**Cost controls applied:**
- Research Agent: max_tokens = 800 (summary only, not full page)
- ICP Scoring Agent: max_tokens = 100 (just the score + reason)
- Manager Agent: max tool calls = 3
- Workflow only triggers on new leads (not on a schedule)

**Observability:**
- n8n execution log captures each node's output and duration
- OpenAI dashboard tracks daily token usage by API key
- Google Sheet logs each lead's score + outcome for quality review

---

## Glossary Additions

| Term | Definition |
|---|---|
| **Orchestration** | The coordination of multiple agents toward a shared goal |
| **Handoff** | Transfer of control and context from one agent to another |
| **Supervisor agent** | An agent whose job is to delegate to and coordinate sub-agents |
| **Fan-out** | Parallel dispatch of tasks to multiple agents simultaneously |
| **Scratchpad** | Temporary shared memory agents use to store intermediate results |
| **Max iterations** | A hard limit on tool calls or reasoning steps per agent run |
| **LangSmith** | LangChain's observability and tracing platform |
| **Langfuse** | Open-source LLM observability tool for token/cost/trace monitoring |
