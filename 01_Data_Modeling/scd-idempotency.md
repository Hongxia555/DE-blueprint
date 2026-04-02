# Slowly Changing Dimensions & Idempotency
**Source:** DataExpert.io Boot Camp — Videos 5 (Lecture) & 6 (Lab)
**Topic:** SCD types, idempotent pipeline design

---

## Key Concepts

### SCD Types
| Type | Behavior | Use Case |
|------|----------|----------|
| SCD 0 | Never changes | Reference data (country codes) |
| SCD 1 | Overwrite in place | Correcting errors, no history needed |
| SCD 2 | Add new row with effective dates | Full history required (user profile changes) |
| SCD 3 | Add column for previous value | Only need current + previous |

### SCD 2 Pattern (Most Common in Production)
- Add `start_date`, `end_date`, `is_current` columns
- On change: close old row (`end_date = today`, `is_current = false`), insert new row
- `end_date = NULL` or `9999-12-31` for current record

### Idempotency
- Definition: running a pipeline multiple times for the same input produces the same output
- Required for: backfills, retries, disaster recovery
- Anti-pattern: `INSERT INTO` without deduplication — creates duplicate rows on retry
- Fix: use `MERGE` (upsert) or `INSERT OVERWRITE partition`

---

## Design Pattern

```sql
-- SCD 2 merge pattern
MERGE INTO dim_users AS target
USING (SELECT * FROM staging_users) AS source
ON target.user_id = source.user_id AND target.is_current = true
WHEN MATCHED AND source.email != target.email THEN
  UPDATE SET end_date = CURRENT_DATE, is_current = false
WHEN NOT MATCHED THEN
  INSERT (user_id, email, start_date, end_date, is_current)
  VALUES (source.user_id, source.email, CURRENT_DATE, NULL, true);

-- Insert new version after closing old
INSERT INTO dim_users (user_id, email, start_date, end_date, is_current)
SELECT user_id, email, CURRENT_DATE, NULL, true
FROM staging_users s
WHERE EXISTS (
  SELECT 1 FROM dim_users d
  WHERE d.user_id = s.user_id AND d.is_current = false AND d.end_date = CURRENT_DATE
);
```

---

## Interview Angles
- How do you handle late-arriving data with SCD 2? → Backfill by replaying from event log
- What's the difference between SCD 2 and event sourcing? → SCD 2 is snapshot-based; event sourcing stores every event
- How do you make a pipeline idempotent? → Deterministic keys, overwrite semantics, no side effects outside the transaction

---

## Notes (from transcript)

### Idempotency — Zach's most emphasized concept
- "You are going to hear me say the word idempotent a million times"
- Definition: pipeline produces **same results whether running in production OR in backfill**
- Why it matters: backfills are necessary for disaster recovery, bug fixes, schema changes
- Non-idempotent pipelines → **corrupt data when re-run** (duplicates, wrong values)
- "If you can build idempotent pipelines you will be a much much better data engineer and I promise that is going to get you a lot more money"

### What breaks idempotency

| Anti-pattern | Why it breaks | Fix |
|---|---|---|
| `INSERT INTO` without dedup | Duplicates on re-run | Use `INSERT OVERWRITE` or `MERGE` |
| `NOW()` / `CURRENT_TIMESTAMP` in logic | Different value each run | Pass run date as explicit parameter |
| Reading "today's" data without a fixed date | Non-deterministic — re-run on different day = different result | Parameterize: `WHERE ds = {{ run_date }}` |

### SCD and idempotency connection
- SCDs track attribute changes over time → if your SCD pipeline isn't idempotent, backfilling corrupts the history
- SCD Type 2 is the hardest to make idempotent because inserting new rows must be conditional

### Key pattern: Slowly Changing + Cumulative together
- SCDs tell you the current and past values
- Cumulative table design tells you what happened over time
- These two work together: you join the SCD snapshot to the cumulative activity table to get full user history

### Real example from lecture
- "My favorite food": lasagna (2000–2008) → curry (2008–present)
- This is a SCD — it drifts over time and must be modeled with date ranges
- Birthday = fixed dimension → SCD Type 0, just store once, never changes
