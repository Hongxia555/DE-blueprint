# AI Agent Solutions for Warehouse Data Access and Security

**Source:** [Meta Engineering Blog, Aug 13 2025](https://engineering.fb.com/2025/08/13/data-infrastructure/agentic-solution-for-warehouse-data-access/)

---

## Core Problem

At Meta's scale, traditional **role-based access control (RBAC)** breaks down when AI systems need cross-domain data access. The hierarchical structure becomes too rigid and slow for dynamic, AI-driven workflows — creating both a productivity bottleneck and a security surface.

---

## Multi-Agent System Design

Two agent types that collaborate with each other:

### Data-User Agent — helps employees get the access they need
- **Alternative suggestion engine** — finds unrestricted tables/queries that satisfy the need
- **Low-risk exploration facilitator** — limited data sampling without full access grants
- **Permission request crafting** — auto-generates well-formed access requests

### Data-Owner Agent — helps managers maintain security controls
- **Security operations handler** — follows documented access procedures
- **Access rule configuration** — replaces traditional manual role-mining

---

## Technical Implementation

### Warehouse Representation for LLMs
- Hierarchical warehouse structure converted to text format for LLM processing
- Organizing units represented as folders; leaf nodes (tables, dashboards, policies) as resources

### Context Management — 3 Types

| Type | How it works |
|---|---|
| Automatic | System detects blocked access scenarios and triggers agent |
| Static | User specifies scope manually |
| Dynamic | Metadata filtering + similarity search to infer relevant context |

### Intention Recognition — 2 Modes

| Mode | How it works |
|---|---|
| Explicit | User declares role-based business need |
| Implicit | Inferred from user activity patterns |

---

## Partial Data Preview Use Case

A key capability that doesn't require full access grants:
1. Context analysis of user activities and business needs
2. **Query-level granular access control** — aggregation and sampling instead of raw access
3. **Daily data-access budgets** per employee — rate-limiting exposure
4. Rule-based risk management as a backstop against agent failures

---

## Safety & Guardrails

- Analytical rule-based risk computation (not just LLM judgment)
- Decision transparency and full trace logging
- **Human-in-the-loop** oversight maintained — agents assist, don't decide autonomously
- All queries, traces, context, and outputs stored for auditing

### Evaluation
- Daily regression testing against curated historical request datasets
- Tracks accuracy, recall, and performance metrics continuously

---

## Future Direction
- Agent-to-agent collaboration (agents negotiating access between systems)
- Infrastructure built natively for agent-driven operations
- Enhanced benchmarking frameworks for access agents

---

## Key Takeaways for DE Practice

- **Query-level access control** (sampling/aggregation budgets) is a practical middle ground between no access and full access — useful pattern for any data platform
- Rule-based risk guardrails alongside LLM agents are essential — don't rely on the model alone for security decisions
- Daily evaluation against historical data is the right pattern for any agentic system where correctness matters
- Implicit intention inference from activity patterns is powerful but requires careful auditing
- Converting hierarchical metadata to text for LLM context is a reusable pattern for any catalog or permissions system
