# DataExpert.io Free DE Boot Camp — Master Index
**Instructor:** Zach Wilson (Data with Zach)
**Source:** https://www.youtube.com/playlist?list=PLwUdL9DpGWU0lhwp3WCxRsb1385KFTLYE
**Platform:** https://learn.dataexpert.io (required for certification + homework)
**GitHub:** https://github.com/DataTalksClub/data-engineering-zoomcamp

---

## Curriculum Map

### Phase 1: Foundational Data Modeling (Weeks 1–2)
| Video | Topic | Notes File |
|-------|-------|-----------|
| 3–4 | Complex Types & Cumulative Dimensions | `01_Data_Modeling/cumulative-dimensions.md` |
| 5–6 | SCDs & Idempotency | `01_Data_Modeling/scd-idempotency.md` |
| 7–8 | Graph Modeling & Additive Dimensions | `01_Data_Modeling/graph-modeling.md` |
| 9 | How Meta Models Big Volume Event Data (4hr) | `01_Data_Modeling/meta-event-data-modeling.md` |

### Phase 2: Track A — Analytics Engineering (Weeks 3–6)
| Video | Topic | Notes File |
|-------|-------|-----------|
| 13 | Gold Pipeline / Airbnb MIDAS + WAP Pattern | `05_Pipeline_Design/write-audit-publish.md` |
| 14 | Data Contracts | `06_Data_Quality_Ops/data-contracts.md` |
| 16 | Meta Design Pattern: Growth Accounting | `05_Pipeline_Design/meta-growth-accounting.md` |
| 17 | Meta Design Pattern: Funnel Analysis | `05_Pipeline_Design/meta-funnel-analysis.md` |
| 19 | DE like a PM: KPIs & Experiments | `00_Fundamentals/kpis-experiments.md` |
| 20 | Maintaining Pipelines (Netflix/Airbnb) | `06_Data_Quality_Ops/pipeline-maintenance.md` |

### Phase 2: Track B — Data Infrastructure (Weeks 3–6)
| Video | Topic | Notes File |
|-------|-------|-----------|
| 10–11 | Spark + Iceberg: Tuning, Joins, UDFs | `04_Infrastructure/spark-iceberg.md` |
| 12 | Testing Spark Jobs in CI/CD | `06_Data_Quality_Ops/spark-testing.md` |
| 15 | Kafka + Flink Real-time Pipelines (3hr) | `04_Infrastructure/kafka-flink-streaming.md` |
| 13 | Medallion / Gold Layer Architecture | `02_Architecture/medallion-midas-pattern.md` |

---

## Study Sequence (Dependency Order)
1. Videos 3–4 → Cumulative dimensions (foundation for everything)
2. Videos 5–6 → SCDs (required for pipeline design)
3. Videos 7–8 → Graph modeling (optional, advanced)
4. Video 9 → Meta event data (high value for interview prep)
5. Videos 10–11 → Spark internals (required for Track B)
6. Video 15 → Kafka + Flink (build on Spark knowledge)
7. Video 13 → MIDAS / WAP pattern (connects modeling → pipeline)
8. Videos 16–17 → Meta design patterns (interview gold)
9. Video 14 → Data contracts (quality foundation)
10. Video 12 → Spark testing (quality + infra)
11. Video 20 → Pipeline maintenance (ops maturity)
12. Video 19 → KPIs & experiments (product thinking)

---

## Key Principles from Zach
- **Idempotency first** — pipelines must produce the same result when re-run
- **Write-Audit-Publish (WAP)** — never write bad data directly to production
- **Struggle principle** — troubleshoot environment issues yourself; learning happens in the pain
- **Docker required** — for Spark and Flink modules; local setups are insufficient

## Certification
- **Attendance cert** — watch all content
- **Expert cert** — complete all AI-graded homework (more valuable for recruiting)

---

## Topic Notes Index

| Folder | Files |
|--------|-------|
| `00_Fundamentals/` | `kpis-experiments.md` |
| `01_Data_Modeling/` | `cumulative-dimensions.md`, `scd-idempotency.md`, `graph-modeling.md`, `meta-event-data-modeling.md` |
| `02_Architecture/` | `medallion-midas-pattern.md` |
| `04_Infrastructure/` | `spark-iceberg.md`, `kafka-flink-streaming.md` |
| `05_Pipeline_Design/` | `write-audit-publish.md`, `meta-growth-accounting.md`, `meta-funnel-analysis.md` |
| `06_Data_Quality_Ops/` | `spark-testing.md`, `data-contracts.md`, `pipeline-maintenance.md` |
