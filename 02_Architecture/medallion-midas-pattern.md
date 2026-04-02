# Medallion Architecture & Airbnb MIDAS Pattern
**Source:** DataExpert.io Boot Camp — Video 13 (Week 3, Analytics Track)
**Topic:** Gold layer pipeline design, Medallion architecture, Write-Audit-Publish

---

## Key Concepts

### Medallion Architecture
| Layer | Name | Description |
|-------|------|-------------|
| Bronze | Raw | Exact copy of source data, append-only, no transformations |
| Silver | Cleaned | Deduplicated, typed, validated — conforms to a schema |
| Gold | Curated | Business-level aggregations, ready for BI tools and dashboards |

### Airbnb MIDAS Process (Gold Layer Pattern)
- **M**etric — define the business metric precisely
- **I**nput — identify upstream Silver tables
- **D**ata contract — specify schema, SLAs, and owner
- **A**udit — validate output before publishing
- **S**tore — write to Gold layer only after audit passes

### Write-Audit-Publish (WAP) Pattern
```
Write → staging table (not visible to consumers)
  ↓
Audit → run quality checks (row counts, nulls, referential integrity)
  ↓
Pass? → Publish (swap/rename staging → production table)
Fail? → Alert + rollback (staging table is dropped, no impact to consumers)
```

**Key benefit:** Consumers never see bad data because writes are atomic.

---

## Implementation Pattern

```sql
-- Step 1: Write to staging
INSERT OVERWRITE gold_metrics_staging
SELECT ... FROM silver_events WHERE ds = '{{ ds }}';

-- Step 2: Audit
SELECT COUNT(*) FROM gold_metrics_staging WHERE metric_value IS NULL;
-- If count > 0 → fail the pipeline, alert, do NOT proceed

-- Step 3: Publish (only if audit passes)
ALTER TABLE gold_metrics EXCHANGE PARTITION (ds='{{ ds }}')
WITH TABLE gold_metrics_staging;
```

---

## Interview Angles
- What prevents bad data from reaching dashboards? → WAP pattern — staging layer absorbs failures
- How does Medallion differ from Lambda architecture? → Medallion is batch-focused layer separation; Lambda adds a speed layer for real-time alongside batch
- Why not just write directly to Gold? → No rollback capability; consumers see partial/bad data mid-write

---

## Notes (from transcript)

### MIDAS = How Airbnb builds Gold pipelines
- MIDAS stands for: the process of ensuring pipelines have **long-term value**, are debuggable, and are improvable
- "Good pipelines start with good documentation" — the spec comes before the SQL
- Zach: "if you actually go through the entire spec process you will create way better data that answers way better questions than if you just immediately start writing SQL"

### The spec-first approach (Airbnb's MIDAS process)
A Gold pipeline spec must define:
1. **Business question** being answered
2. **Input tables** and their owners/SLAs
3. **Grain** — one row per what?
4. **Metrics** being computed
5. **Quality checks** — what would make this wrong?
6. **Output schema** — column names, types, nullable
7. **Downstream consumers** — who reads this and how

### Why data quality matters beyond the data
- Zillow lost ~$100M because they over-trusted their ML model's housing price predictions
- "Without trust — without that acknowledgement — are you even being data driven if decisions aren't being made off the datasets you're creating?"
- Trust in data = **discoverability + correctness + completeness**

### Discoverability (underrated component of data quality)
- If someone wants to make a decision and doesn't know whether data exists → they can't use it
- Cataloging problem: data catalog/metadata layer is as important as the data itself
- Zach: "that's not as much a data engineering problem as more of a data infrastructure problem but they are related"

### Common causes of bad data (from Zach's list)
- Logging errors (software engineer changed schema → nulls appear)
- Double-click logging bug (button not disabled → 2 events per action)
- Snapshotting errors (rare — Zach sees this <1/year)
- Production data quality issues (bad data in the app itself)
- Schema evolution — source schema changed, pipeline schema didn't
- Non-idempotent pipeline + backfill = corrupted data
- **Not enough validation before release** (most common cause per Zach)
