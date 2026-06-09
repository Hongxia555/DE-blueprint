# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

This is a personal second brain for Data Engineering — structured Markdown notes across 7 topic areas. There is no build, test, or lint pipeline. The ongoing goal is to fill placeholder directories with substantive notes.

## Topic Areas

| Folder | Focus | Key Files |
|---|---|---|
| `00_Fundamentals/` | SQL, Python, distributed systems, CS foundations | `DE_concepts.md`, `database_types.md` |
| `01_Data_Modeling/` | Dimensional modeling, Data Vault, SCD types, schema design | `scd-idempotency.md`, `cumulative-dimensions.md` |
| `02_Architecture/` | Lambda/Kappa, Medallion, data mesh, lakehouse | `medallion-midas-pattern.md` |
| `03_AI_Augmentation/` | LLM integration, Claude Code workflows, agentic tooling | `ai-agents-fundamentals.md`, `claude-tips/` |
| `04_Infrastructure/` | Spark, Kafka, Airflow, cloud platforms | `kafka-flink-streaming.md`, `spark-iceberg.md` |
| `05_Pipeline_Design/` | Ingestion, transformation, orchestration, ELT/ETL | `dbt/`, `pyspark/`, `write-audit-publish.md` |
| `06_Data_Quality_Ops/` | dbt tests, observability, incident response, SLAs | `data-contracts.md`, `spark-testing.md` |
| `Resources/` | Learning materials, repo overviews, reference links | `reading-list.md`, `repo-overviews/` |

## Note Conventions

- All notes are Markdown. Use clear headers, bullets, and fenced SQL/code blocks.
- Fact tables use `fct_` prefix; dimensions use `dim_` prefix.
- Files prefixed with numbers indicate ordering (e.g. `00_data_modeling_overview.md`).

## Skills (repo-scoped)

Invoke with `/skill-name`:

| Skill | When to use |
|---|---|
| `/dbt-model` | Scaffold a new dbt model (staging/intermediate/mart SQL + schema.yml tests) |
| `/pipeline-design` | Design a pipeline from source-to-target requirements |
| `/incident-report` | Structure a data incident post-mortem |

Skill definitions live in `.claude/skills/<name>/SKILL.md`.

## Working in This Repo

- Before adding notes to a topic area, read `README.md` and check what already exists in that folder.
- Read `03_AI_Augmentation/claude-tips/claude-code-tips.md` before suggesting new Claude Code workflows — it contains the canonical guidance for this workspace.
- When filling in a new topic, prefer a single well-structured file over many small stub files.
- Keep the **Topics to Explore** checklist in `README.md` updated — check off items when a note is written for them.
