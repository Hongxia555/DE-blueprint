# Pipeline Maintenance & Operations
**Source:** DataExpert.io Boot Camp — Video 20 (Week 6, Analytics Track)
**Topic:** Runbooks, on-call rotations, incident response, pipeline observability (Netflix/Airbnb patterns)

---

## Key Concepts

### The "Non-Sexy" Side of DE
- Most DE job time is maintenance, not greenfield builds
- Reliable pipelines require: monitoring, alerting, runbooks, and on-call rotations
- Netflix and Airbnb treat data pipelines with the same operational rigor as production services

### Pipeline Observability Stack
| Layer | What to Monitor | Tool Examples |
|-------|----------------|---------------|
| Freshness | Is data available by SLA? | Airflow SLAs, custom alerts |
| Completeness | Are all expected rows present? | Row count checks, anomaly detection |
| Accuracy | Are values within expected ranges? | dbt tests, Great Expectations |
| Consistency | Does this table agree with upstream? | Cross-table reconciliation |
| Lineage | What does this table depend on? | OpenLineage, Datahub, dbt |

### Runbook Template
Every production pipeline should have a runbook:
```markdown
## Pipeline: <name>

### Normal operation
- Schedule: daily at 06:00 UTC
- SLA: available by 08:00 UTC
- Success indicators: row count ~X, no nulls on key fields

### Common failures and fixes

#### Issue: Pipeline missed SLA
1. Check Airflow logs for task failure
2. Check upstream table freshness: `SELECT MAX(ds) FROM upstream_table`
3. If upstream late: wait or trigger manual backfill
4. If own failure: check logs, fix, re-trigger from failed task

#### Issue: Row count anomaly (>20% deviation)
1. Check if source system had an outage
2. Compare to same day last week
3. Check for schema changes in upstream: `DESCRIBE TABLE upstream`
4. Escalate to source team if unexplained

### Escalation path
- On-call DE → #data-incidents Slack → Source team lead
```

### Runbook vs Data Contract

| | Runbook | Data Contract |
|---|---|---|
| **Audience** | DE team (internal) | Producer + consumer (external) |
| **Purpose** | How to recover from failures | What downstream can depend on |
| **Triggered by** | Incidents | Onboarding a new consumer |
| **Owner** | Pipeline owner | Agreed between teams |
| **Changes when** | Failure patterns change | Schema or semantics change |

- **Runbook** — operational guide for when things go wrong; answers *"the pipeline is broken at 3am, what do I do?"*
- **Data Contract** — agreement on what the data will look like; answers *"what can downstream teams rely on?"* — covers schema, SLA, field semantics, and how breaking changes are communicated

A runbook helps you fix the pipeline. A data contract defines what you're responsible for not breaking in the first place.

### On-Call Best Practices
- Rotate on-call weekly (no permanent on-call)
- Track every incident in a postmortem doc
- Fix root cause, not just symptoms — add monitoring to catch it earlier next time
- **Blameless postmortems** — focus on system failures, not human errors

### Backfill Patterns
```python
# Safe backfill: run for each date in range
for date in date_range(start, end):
    run_pipeline(date)
    validate_output(date)
    # Only advance if validation passes
```

- Always backfill in forward-chronological order (cumulative tables depend on previous days)
- Test idempotency before backfilling production — run same date twice and verify identical output

---

## Interview Angles
- What do you do when a pipeline misses its SLA? → Check runbook: upstream freshness first, then own logs, then alert stakeholders with ETA
- How do you handle a data incident at 3am? → Follow runbook, mitigate first (rollback/hide bad data), root cause second, postmortem after
- How do you reduce on-call burden over time? → Better monitoring catches issues earlier, runbooks reduce MTTR, root-cause fixes reduce recurrence

---

## Notes (from transcript)

### The maintenance cost problem (Zach's fear)
- "Every pipeline you build slowly crushes you with maintenance costs"
- Math: if each pipeline has 10% daily failure probability:
  - 1 pipeline → on-call acts 3 days/month
  - 10 pipelines → on-call acts every single day
  - 100 pipelines → on-call does 10 things every day
- "Doing data engineering is inherently unsustainable because every pipeline you write has an added cost"

### Data engineer burnout is real
- "Google 'data engineer burnout' — 90%+ of data engineers are experiencing burnout right now"
- Causes: invisible work (no recognition), high on-call burden, pressure to build fast without maintaining
- Lesson: **protect your peace** — maintenance strategy is as important as building strategy

### Why data engineers get paid well (Zach's theory)
- Ramp time is only 5–6 months — not that hard to learn
- But you get little recognition — analyst/data scientist presents the work, DE built it
- High burn rate means many people leave → supply stays low → salaries stay high

### 4 topics Zach covers in video 20
1. **Difficult parts of the DE job** — the real stuff nobody warns you about
2. **Ownership models** — who owns what and how to not let it all fall on one person
3. **Team models for data engineering** — centralized vs embedded vs hybrid
4. **Common maintenance problems** — the things that will wake you up at 3am

### What Zach covered from Netflix, Airbnb, Facebook
- **Netflix**: strong runbook culture, every pipeline has documented failure modes
- **Airbnb**: MIDAS process — pipelines must be self-documented before they're built
- **Facebook**: Signal table pattern for safe production writes at massive scale

### Ownership model principles
- Never have **one person own everything** — bus factor risk, burnout, knowledge silos
- Rotate on-call weekly — no permanent on-call
- **Blameless postmortems** — root cause the system failure, not the human
- Add monitoring after every incident: "don't fix the same bug twice"
