# de-blueprint

> A personal Second Brain for Data Engineering — structured notes, patterns, and reference designs across the full DE stack.

---

## Table of Contents

| # | Folder | Description |
|---|---|---|
| 00 | [Fundamentals](./00_Fundamentals/) | Core concepts: SQL, Python, distributed systems, CS foundations |
| 01 | [Data Modeling](./01_Data_Modeling/) | Dimensional modeling, Data Vault, entity relationships, schema design |
| 02 | [Architecture](./02_Architecture/) | System design patterns, Lambda/Kappa, Medallion, data mesh, lakehouse |
| 03 | [AI Augmentation](./03_AI_Augmentation/) | LLM integration, AI-assisted pipelines, context engineering, agentic tooling, compound engineering |
| 04 | [Infrastructure](./04_Infrastructure/) | Cloud platforms, Spark, Kafka, Airflow, containerization, IaC |
| 05 | [Pipeline Design](./05_Pipeline_Design/) | Ingestion, transformation, orchestration patterns, ELT/ETL best practices |
| 06 | [Data Quality & Ops](./06_Data_Quality_Ops/) | Testing, observability, SLAs, incident response, dbt tests, Great Expectations |
| — | [Resources](./resources/) | Books, courses, cheat sheets, interview prep |

---

## Topics to Explore

### 03 — AI Augmentation
- [ ] **Context Engineering** (上下文工程) — how to design, structure, and manage context fed to LLMs for reliable, high-quality outputs
- [ ] **Agentic Validation** (Agentic验证) — techniques for verifying agent outputs, preventing hallucinations, and building trust layers in agentic pipelines
- [x] **Agentic Tooling** (Agentic工具化) — building and exposing tools (MCP servers, function calling, skills) that agents can use to take real actions
- [ ] **Agentic Codebase** (Agentic代码库) — structuring a codebase so AI agents can navigate, understand, and modify it safely (CLAUDE.md, subagents, knowledge base)
- [x] **Compound Engineering** (复合式工程) — orchestrating multiple agents, models, and tools in coordinated workflows to solve complex, multi-step tasks

### 00 — Fundamentals
- [ ] SQL window functions deep dive
- [ ] Python for DE: generators, async, type hints
- [ ] Distributed systems: CAP theorem, consistency models

### 01 — Data Modeling
- [ ] Star schema vs Data Vault trade-offs
- [x] SCD Type 1 / 2 / 3 implementation patterns
- [ ] One Big Table (OBT) pattern and when to use it

### 02 — Architecture
- [x] Medallion architecture (Bronze / Silver / Gold)
- [ ] Data mesh: domain ownership and data contracts
- [ ] Lakehouse vs traditional data warehouse

### 04 — Infrastructure
- [ ] Airflow vs Dagster vs Prefect comparison
- [x] Spark optimization: partitioning, caching, skew handling
- [ ] dbt project structure best practices

### 05 — Pipeline Design
- [x] Idempotency and incremental load patterns
- [ ] CDC (Change Data Capture) strategies
- [ ] Schema evolution handling

### 06 — Data Quality & Ops
- [ ] dbt test coverage strategy
- [ ] Data observability: Monte Carlo, Anomalo, Elementary
- [ ] Incident response runbook for data outages

---

## How to Use This Repo

- Each folder contains topic-specific notes, diagrams, and code snippets
- Files are prefixed with numbers for ordering (e.g., `01_star_schema.md`)
- Use the `resources/` folder for reference materials and external links
- Check off items in **Topics to Explore** as you write notes on them

---

## Structure

```
de-blueprint/
├── 00_Fundamentals/
│   └── kpis-experiments.md
├── 01_Data_Modeling/
│   ├── 00_data_modeling_overview.md
│   ├── cumulative-dimensions.md
│   ├── graph-modeling.md
│   ├── meta-event-data-modeling.md
│   ├── scd-idempotency.md
│   └── xfn-partner-needs.md
├── 02_Architecture/
│   └── medallion-midas-pattern.md
├── 03_AI_Augmentation/
│   ├── ai-agents-fundamentals.md
│   ├── ai-agents-management.md
│   ├── llm-how-it-works.md
│   ├── llm-model-comparison.md
│   └── claude-tips/
│       ├── built-in-commands.md
│       ├── claude-code-memory.md
│       └── claude-code-tips.md
├── 04_Infrastructure/
│   ├── kafka-flink-streaming.md
│   └── spark-iceberg.md
├── 05_Pipeline_Design/
│   ├── meta-funnel-analysis.md
│   ├── meta-growth-accounting.md
│   └── write-audit-publish.md
├── 06_Data_Quality_Ops/
│   ├── data-contracts.md
│   ├── pipeline-maintenance.md
│   └── spark-testing.md
├── Repo Bank/
│   └── repo-overviews/
├── dataexpert-bootcamp.md
└── CLAUDE.md
```
