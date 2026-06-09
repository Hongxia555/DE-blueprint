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
| — | [Resources](./Resources/) | Learning materials, repo overviews, reference links |

---

## Topics to Explore

### 03 — AI Augmentation
- [ ] **Context Engineering** — how to design, structure, and manage context fed to LLMs for reliable, high-quality outputs
- [ ] **Agentic Validation** — techniques for verifying agent outputs, preventing hallucinations, and building trust layers in agentic pipelines
- [x] **Agentic Tooling** — building and exposing tools (MCP servers, function calling, skills) that agents can use to take real actions
- [ ] **Agentic Codebase** — structuring a codebase so AI agents can navigate, understand, and modify it safely (CLAUDE.md, subagents, knowledge base)
- [x] **Compound Engineering** — orchestrating multiple agents, models, and tools in coordinated workflows to solve complex, multi-step tasks
- [ ] **Agentic Memory — Unified Cross-Harness** — decoupling memory from any single AI tool (Claude Code, Cursor, Codex) using lifecycle hooks; three-layer architecture: hooks (deterministic logging) → dream phase (offline distillation into semantic markdown) → injection (context retrieval at session start)
- [ ] **Agentic Memory — Five Memory Types** — context-resident compression, retrieval-augmented stores, reflective self-improvement, hierarchical virtual context, policy-learned management (store/retrieve/update/summarize/discard as agent-callable tools)
- [ ] **Memory as Production Engineering** — benchmarking agent memory, measuring accuracy gains from accumulated context (2.5% → 50%+ on enterprise data tasks), and operational patterns for memory lifecycle management

### 00 — Fundamentals
- [x] SQL window functions deep dive
- [x] Python for DE: generators, async, type hints
- [x] Distributed systems: CAP theorem, consistency models

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
- [x] dbt project structure best practices

### 05 — Pipeline Design
- [x] Idempotency and incremental load patterns
- [ ] CDC (Change Data Capture) strategies
- [ ] Schema evolution handling
- [x] PySpark end-to-end pipeline (Medallion, Delta Lake, Pydantic validation)
- [x] dbt testing and materialization patterns

### 06 — Data Quality & Ops
- [ ] dbt test coverage strategy
- [ ] Data observability: Monte Carlo, Anomalo, Elementary
- [ ] Incident response runbook for data outages

---

## How to Use This Repo

- Each folder contains topic-specific notes, diagrams, and code snippets
- Files are prefixed with numbers for ordering (e.g., `01_star_schema.md`)
- Use the `Resources/` folder for learning materials, repo overviews, and reference links
- Check off items in **Topics to Explore** as you write notes on them

---

## Structure

```
de-blueprint/
├── 00_Fundamentals/
│   ├── DE_concepts.md          ← core DE concepts: schema design, ETL/ELT, distributed systems, data quality
│   ├── database_types.md       ← relational vs columnar, SQL vs NoSQL, CAP theorem, decision guide
│   ├── ab-testing.md
│   └── kpis-experiments.md
├── 01_Data_Modeling/
│   ├── 00_data_modeling_overview.md
│   ├── cumulative-dimensions.md
│   ├── graph-modeling.md
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
│   ├── dbt/
│   │   ├── dbt-and-testing.md          ← dbt test types, materialization, source freshness
│   │   ├── dbt-example-ecommerce.md    ← full e-commerce pipeline (staging → mart)
│   │   └── dbt-faq.md                  ← dbt Q&A reference
│   ├── pyspark/
│   │   ├── pyspark-overview.md         ← PySpark concepts, Delta Lake, transformations vs actions
│   │   ├── pyspark-example-pipeline.md ← end-to-end Medallion pipeline with Pydantic + Delta
│   │   └── pyspark-faq.md              ← PySpark Q&A reference
│   └── write-audit-publish.md
├── 06_Data_Quality_Ops/
│   ├── data-contracts.md
│   ├── pipeline-maintenance.md
│   └── spark-testing.md
├── Resources/
│   └── repo-overviews/
└── CLAUDE.md
```
