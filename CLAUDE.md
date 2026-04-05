# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

This is a personal second brain for Data Engineering — structured Markdown notes across 7 topic areas. There is no build, test, or lint pipeline. The ongoing goal is to fill placeholder directories with substantive notes.

## Topic Areas

| Folder | Focus |
|---|---|
| `00_Fundamentals/` | SQL, Python, distributed systems, CS foundations |
| `01_Data_Modeling/` | Dimensional modeling, Data Vault, SCD types, schema design |
| `02_Architecture/` | Lambda/Kappa, Medallion, data mesh, lakehouse |
| `03_AI_Augmentation/` | LLM integration, Claude Code workflows, agentic tooling |
| `04_Infrastructure/` | Spark, Kafka, Airflow, cloud platforms |
| `05_Pipeline_Design/` | Ingestion, transformation, orchestration, ELT/ETL |
| `06_Data_Quality_Ops/` | dbt tests, observability, incident response, SLAs |

Most directories contain only `.gitkeep` — check before assuming content exists.

## Note Conventions

- All notes are Markdown. Use clear headers, bullets, and fenced SQL/code blocks.
- Fact tables use `fct_` prefix; dimensions use `dim_` prefix.
- Interview prep structure: Business context → North Star metrics → Data model → Sample SQL → Metric investigation frameworks.
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
