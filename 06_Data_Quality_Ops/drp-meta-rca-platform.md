# DrP: Meta's Root Cause Analysis Platform at Scale

**Source:** [Meta Engineering Blog, Dec 19 2025](https://engineering.fb.com/2025/12/19/data-infrastructure/drp-metas-root-cause-analysis-platform-at-scale/)

---

## What is DrP?

An automated **Root Cause Analysis (RCA) platform** that codifies incident investigation workflows into reusable "analyzers" — programmatic units that run automatically when alerts fire. Instead of on-call engineers manually triaging every incident, DrP runs pre-written investigation logic and surfaces findings immediately.

---

## Scale

| Metric | Value |
|---|---|
| Teams using it | 300+ |
| Daily analyses | 50,000 |
| Analyzers deployed | 2,000+ |
| MTTR reduction | 20–80% |
| In production | 5+ years |

---

## Architecture: 4 Core Components

### 1. Expressive SDK
- Engineers write **analyzers** — codified investigation decision trees
- Built-in ML libraries: anomaly detection, time series correlation, dimension analysis, event isolation
- Bootstrap templates to reduce authoring friction
- Automated **backtesting** integrated into code review to validate analyzer quality

### 2. Scalable Backend
- Multi-tenant execution environments with isolated worker pools
- Queue management + async result processing
- Handles 50,000 analyses/day reliably

### 3. Workflow Integration
- Auto-triggers on alert activation
- Integrates with incident management tools, UI, and CLI
- Delivers results directly to on-call engineers at time of incident

### 4. Post-Processing System
- Annotates alerts with findings automatically
- Creates tasks and PRs from analysis results
- Powers **DrP Insights** — periodic ranking of top alert causes across the org

---

## Analyzer Authoring Workflow

1. **Enumerate** — list investigation steps, inputs, and problem isolation paths
2. **Bootstrap** — SDK generates template code with boilerplate
3. **Code** — build investigation decision tree using SDK libraries
4. **Chain** — link dependent service analyzers via API for multi-service traces
5. **Test** — automated backtesting in code review ensures correctness before deployment

---

## Why It Matters

### For On-Call Engineers
- Replaces repetitive manual triage with automated, consistent workflows
- Reduces on-call fatigue — findings arrive before you've even started digging
- Consistent execution: no steps skipped, no knowledge gaps under pressure

### For the Organization
- Institutional knowledge gets **codified and reused** across 300+ teams rather than living in people's heads
- Compound value: each new analyzer benefits everyone who shares similar alert patterns
- Reusable analyzers across teams lower the cost of future incident investigations

### MTTR Reduction Mechanisms
- **Efficiency**: Automated triage reduces time from alert → hypothesis
- **Consistency**: Codified workflows prevent investigation drift
- **Scalability**: Handles thousands of daily analyses across complex distributed systems

---

## Future Direction

DrP is evolving into an **AI-native platform** as part of Meta's broader **AI4Ops** vision — integrating streamlined ML algorithms and smarter integrations to improve analysis accuracy and simplify the user experience further.

---

## Key Takeaways for DE Practice

- RCA automation is most valuable when investigation steps are **repeatable and data-driven** — good candidates to codify first
- Backtesting analyzers against historical incidents is critical for trust and adoption
- The "DrP Insights" pattern (periodic ranking of top causes) is a useful mental model for any observability system — surface the signal, not just the noise
- Codifying runbooks as code (analyzers) > keeping them in Confluence/Notion — they stay current and are actually executed
