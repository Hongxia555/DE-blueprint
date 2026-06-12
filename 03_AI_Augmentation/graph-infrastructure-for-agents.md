# Graph Infrastructure for AI Agents

> Source: LinkedIn post by Clair Sullivan (Neo4j) — 5 ways graphs improve agent quality in production.

**Related:** [[ai-agents-fundamentals]] · [[ai-agents-management]] · [[agentic-memory-cross-harness]]

---

## Why Graph?

Most agent infrastructure treats memory and logging as afterthoughts — a vector store for recall, a flat file for logs. The problem is that agents produce **relational information** (decisions, context, handoffs, rules) and flat structures can't express relationships.

A graph makes agent behavior **visible and queryable**:
- Nodes = decisions, turns, entities, rules
- Edges = "informed by", "overrode", "delegated to", "followed"
- When something goes wrong, you walk the graph backwards — not grep through logs

The common substrate: **graph is the thing your agents think on**, not just where you dump output.

---

## 5 Patterns

### 1. Decision Traces

Every agent decision gets logged as a node with the context that informed it.

```
[User Input] ──→ [Context Retrieved] ──→ [Rule Applied] ──→ [Decision Made]
                         ↑                      ↑
                 (vector search result)   (constraint from memory)
```

When something goes wrong, you can walk backwards through the chain:
- What did the agent know at decision time?
- Which rules were active?
- What context was retrieved?
- What got overridden and why?

**The DE analogy:** Decision traces are data lineage for agents. Just as `dbt docs` shows column-level lineage from source → mart, decision traces show reasoning lineage from input → output.

**Why it matters:** Without traces, post-incident analysis is "the AI said so." With traces, it's "here's exactly why."

---

### 2. Agent Memory (Three-Layer Architecture)

Most agent memory is:
- **Vector store** — semantic recall (finds similar past content)
- **Session buffer** — short-term in-context memory

Adding a **graph layer** gives you a third type:

| Layer | What it stores | What it enables |
|---|---|---|
| Vector store | Embeddings of past content | "Find similar questions" |
| Session buffer | Recent turns | "What did we just talk about?" |
| **Graph** | Entities, relationships, constraints | "How do these things connect?" |

With the graph layer, the agent doesn't just remember what you said — it knows:
- That "Project X" is owned by "Team Y" and has a "deadline constraint"
- That a specific user prefers Python over SQL
- That rule A overrides rule B in certain contexts

These relationships persist **across sessions** — not lost when the context window resets.

**Tool reference:** William Lyon's `agent-memory` library (Neo4j) is the standard starting point for graph-based agent memory.

---

### 3. Turn Logging

Logging conversation turns directly to the graph (instead of flat Markdown/JSON files) unlocks queries you can't do otherwise:

```cypher
-- Find turns that ended in failure
MATCH (t:Turn {outcome: "failure"})-[:USED_CONTEXT]->(c:Context)
RETURN t.query, c.retrieved_chunk

-- Find similar past failures for the current input
MATCH (t:Turn)-[:SEMANTICALLY_SIMILAR]->(current:Turn)
WHERE current.id = $this_turn_id
RETURN t.resolution ORDER BY t.similarity_score DESC
```

**What this enables:**
- Query for similar failures across all runs
- Trace execution paths across multiple runs
- Cold-start a new run from a specific point to test a different outcome
- **Cost optimization**: for production-scale agents, use vector similarity to find turns where the same question was answered before — pass the stored answer to context instead of re-running the full agent

**The DE analogy:** This is event sourcing applied to agent runs. Each turn is an immutable event; the graph is your event store with relationship structure.

---

### 4. Eval Layers

Standard evals score **outputs**. Graph-based evals score **the reasoning process**.

When reasoning steps are nodes:

```
[Input] → [Step 1: Retrieve] → [Step 2: Reason] → [Step 3: Tool Call] → [Output]
               ↓                     ↓                    ↓
          score: 0.9            score: 0.6           score: 0.4 ← where it broke
```

You can:
- Run evaluators over sub-graphs (just the retrieval sub-graph, just the reasoning sub-graph)
- Flag exactly which step degraded
- Build quality scoring **into your pipeline infrastructure** early, before you have production failures to react to

**The principle:** Build evals into the infrastructure from day one. Bolting them on after production incidents is much harder.

---

### 5. Multi-Agent Orchestration

In multi-agent systems, state management becomes the hard problem. The graph solves it by acting as **shared state** that all agents read and write.

```
         ┌─────────────┐
         │ Mayor Agent  │  ← orchestrator: decides what to delegate
         └──────┬───────┘
                │ delegates via graph edges
    ┌───────────┼────────────┐
    ↓           ↓            ↓
[Sub-Agent A] [Sub-Agent B] [Sub-Agent C]
    │           │            │
    └───────────┴────────────┘
                │
         writes results back to graph
                │
         [Mayor reads graph to synthesize]
```

Instead of passing JSON blobs between functions, agents:
- Read context from the graph
- Write results, decisions, and handoff metadata back to the graph
- The execution path is visible and queryable — not hidden inside function call stacks

**The DE analogy:** This is the DAG model applied to agent orchestration. Just as Airflow tracks task state in a metadata database, a graph tracks agent handoff state — with the added benefit of queryable relationships.

---

## Why This Matters for DEs

As a DE building or maintaining agentic pipelines, the graph infrastructure layer is your responsibility — not the ML team's.

| Agent problem | DE equivalent | Graph solution |
|---|---|---|
| Why did the agent do X? | Data lineage | Decision trace nodes |
| Did agent quality degrade? | Data quality monitoring | Eval layers on reasoning steps |
| Which agent run failed? | Pipeline run logs | Turn log graph |
| How do agents share context? | Shared state / event bus | Graph as shared state |

The pattern is the same one you already know from data pipelines: **make the behavior observable, then make it correctable.**

---

## Stack Reference

| Tool | Role |
|---|---|
| **Neo4j** | Graph database (most common in agent infra) |
| **agent-memory** (William Lyon) | Graph-based agent memory library for Neo4j |
| **LangGraph** | Agent orchestration with graph-structured state (different concept — DAG, not property graph) |
| **Great Expectations / dbt tests** | Analogous eval layer for data pipelines |

---

*See also: [[ai-agents-fundamentals]] for agent basics · [[ai-agents-management]] for orchestration patterns · [[agentic-memory-cross-harness]] for cross-harness memory architecture*
