# Cumulative Dimensions & Complex Types
**Source:** DataExpert.io Boot Camp — Videos 3 (Lecture) & 4 (Lab)
**Topic:** Complex data types, cumulation patterns, struct/array schemas

---

## Key Concepts

### Cumulative Table Design
- Tracks entity state across time by accumulating history into a single row
- Avoids expensive full scans — query only the latest snapshot row
- Pattern: `INSERT INTO cumulative SELECT ... FROM today JOIN yesterday`

### Complex Types: Struct & Array
- `Struct` — embed nested attributes inline (avoid extra joins)
- `Array<Struct>` — store a time-series of states per entity in one row
- Enables "history in a single row" — all past activity for a user in one record

### Idempotency Requirement
- Cumulative pipelines MUST be idempotent — re-running for the same date produces the same result
- Use `INSERT OVERWRITE` (not `INSERT INTO`) when writing daily partitions

---

## Design Pattern

```sql
-- Cumulative user activity table
CREATE TABLE users_cumulated (
  user_id BIGINT,
  dates_active ARRAY<DATE>,  -- all dates user was active
  date DATE                   -- current snapshot date (partition key)
);

-- Daily accumulation
INSERT OVERWRITE users_cumulated
SELECT
  COALESCE(t.user_id, y.user_id) AS user_id,
  CASE
    WHEN y.dates_active IS NULL THEN ARRAY[t.event_date]
    WHEN t.user_id IS NOT NULL THEN ARRAY_UNION(y.dates_active, ARRAY[t.event_date])
    ELSE y.dates_active
  END AS dates_active,
  CURRENT_DATE AS date
FROM today_events t
FULL OUTER JOIN yesterday_snapshot y ON t.user_id = y.user_id;
```

---

## Interview Angles
- Why use arrays instead of one row per day? → Reduces scan cost, enables bitmap operations
- What makes a cumulative pipeline idempotent? → Overwrite partition, no side effects
- When would you NOT use cumulative tables? → Very high cardinality user base + frequent schema changes

---

## Notes (from transcript)

### Who is your data customer? (Zach's #1 principle)
Before modeling, always ask who will query this data:
- **Analysts/Data Scientists** → flat primitive types, easy to sum/avg — no nested types
- **Other Data Engineers** → master data layer, structs/arrays OK, they can handle it
- **ML Models** → identifier + flat decimal feature columns
- **Customers** → charts only, never raw tables

### OLTP vs OLAP vs Master Data (the continuum)
- OLTP: normalized, many tables, fast single-row lookups (app databases)
- Master Data: **middle ground** — complete entity definitions, still normalized, denormalized enough to be useful
- OLAP: flattened, duplicated columns, optimized for population-level scans
- Zach built a master data table at Airbnb that **merged 40 transactional tables into 1** for pricing/availability

### Temporal Cardinality Explosion
- When a slowly-changing dimension gets a date range attached, cardinality explodes
- Example: one user × 365 days × 10 slowly-changing attributes = thousands of rows
- Solution: use **run-length encoding** (collapse consecutive identical values into one row with a date range)

### Run-Length Encoding
```sql
-- Instead of: user_id | date       | favorite_food
--             1       | 2020-01-01 | lasagna
--             1       | 2020-01-02 | lasagna   (duplicated!)
--             1       | 2021-06-01 | curry

-- Use: user_id | start_date | end_date   | favorite_food
--      1       | 2020-01-01 | 2021-05-31 | lasagna
--      1       | 2021-06-01 | NULL       | curry
```

### Airbnb Example (Array of Struct)
- Modeled listing availability across all Airbnbs using `ARRAY<STRUCT<date, availability_status>>`
- **Shrunk dataset by over 95%** compared to one-row-per-day approach
- Trade-off: harder to query for analysts, but perfect for other data engineers or ML pipelines
