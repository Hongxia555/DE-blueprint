# Write-Audit-Publish (WAP) Pattern
**Source:** DataExpert.io Boot Camp — Video 13 (Week 3 Analytics Track) + Video 14
**Topic:** Pipeline specs, data quality gates, safe production writes

---

## Key Concepts

### The Problem WAP Solves
- Without WAP: bad data written directly to production → dashboards/models break → rollback is manual and slow
- With WAP: staging absorbs all failures; consumers see only validated data

### WAP Stages

```
[Source] → WRITE → [Staging Table]
                        ↓
                     AUDIT (quality checks)
                    /         \
                 PASS          FAIL
                  ↓              ↓
              PUBLISH         ALERT + STOP
           (atomic swap)    (staging dropped)
```

### Audit Checks (Standard Suite)
```sql
-- 1. Row count vs. yesterday (± 20% threshold)
SELECT ABS(today_count - yesterday_count) / yesterday_count > 0.2 AS anomaly

-- 2. Null check on required fields
SELECT COUNT(*) FROM staging WHERE primary_key IS NULL

-- 3. Referential integrity
SELECT COUNT(*) FROM staging s
LEFT JOIN dim_users d ON s.user_id = d.user_id
WHERE d.user_id IS NULL

-- 4. Duplicate check
SELECT COUNT(*) - COUNT(DISTINCT primary_key) AS duplicates FROM staging

-- 5. Date range sanity
SELECT MIN(ds), MAX(ds) FROM staging
```

### Pipeline Specification Template
Every Gold pipeline should have a spec:
```
Pipeline: <name>
Owner: <team>
Input tables: <silver tables + SLA>
Output table: <gold table + schema>
Grain: <one row per X per Y>
Refresh cadence: <daily/hourly>
SLA: <available by HH:MM>
WAP checks: <list of audit queries>
Alerting: <PagerDuty/Slack channel>
```

---

## Interview Angles
- How do you prevent bad data from reaching a dashboard? → WAP pattern — atomic publish only after audit passes
- What audit checks do you always run? → Row count anomaly, null check on keys, duplicate check, referential integrity
- How do you rollback a bad pipeline run? → With WAP: staging was never published, so nothing to rollback. Without WAP: restore from backup partition or replay from source.

---

## Notes (from transcript)

### WAP = Write Audit Publish (not the Cardi B song)
- "Cardi B ruined this data quality pattern" — Zach's running joke
- WAP prevents **80–90% of data quality headaches** from publishing bad data
- Used at **Netflix and Airbnb**; Facebook used a different pattern called Signal Table

### Signal Table Pattern (Facebook's alternative to WAP)
- Instead of a staging table, Facebook writes data and then sets a "signal" flag when it's ready
- Downstream pipelines wait for the signal before reading
- Pros: faster for large-scale pipelines; Cons: more complex, harder to roll back

### Blocking vs Non-blocking quality checks
- **Blocking check**: serious issue → stop the pipeline, alert, manual investigation required
- **Non-blocking check**: data weirdness → fire alert, but still publish (don't block SLA)
- Example blocking check: Ethiopia internet shutdown caused near-zero activity → blocking check failed → investigated → determined it was real world event → manually published

### Why you MUST have someone else validate your data
- "It's impossible — if you build the pipeline it's totally impossible to validate it yourself"
- Zach tried at Airbnb when the analytics engineer was out sick: "I totally didn't have it"
- Always have a data analyst or analytics engineer check your assumptions independently

### Airbnb MIDAS pre-validation process
1. **Backfill 1 month** of the new dataset first (not full history)
2. Check: null rates, duplicates, enumerations, time series patterns, row counts
3. Document findings in a "data look" doc with charts
4. Only if it looks good → backfill the rest

### Writing to production is a contract (3 components)
1. **Schema** — column names and data types
2. **Quality checks** — row counts, duplicates, nulls, referential integrity
3. **How data shows up in production** — WAP or signal table, atomic vs incremental
