# Incremental ETL Design
**Source:** Session notes — incremental load design discussion
**Topic:** Choosing an incremental strategy from source capabilities; watermarks, idempotency, late-arriving data

---

## The Core Idea

You don't *choose* an incremental pattern. The source system's capabilities determine what is
possible, and the design falls out of that. Profile the source first, then pick.

Related: [SCD & Idempotency](../01_Data_Modeling/scd-idempotency.md) · [Write-Audit-Publish](./write-audit-publish.md)

---

## Part 1 — Five Questions About the Source

### 1. Is there a reliable change timestamp?
| Source has | You can do |
|---|---|
| `updated_at`, bumped on every write | Watermark-based incremental pull |
| Only `created_at` | Append-only capture; updates are invisible |
| Neither | Full compare, or CDC |

### 2. Are rows append-only or mutable?
- **Append-only** (events, clickstream, transaction ledger) — easiest case.
- **Mutable** (order status, user profile, inventory) — needs upsert or SCD 2.
- The real question is **how long the mutation window is**. If an order can still change 30 days
  after creation, your lookback window must be 30 days.

### 3. Are there hard deletes?
The most commonly missed requirement. A watermark pull can **never** see a `DELETE` — the row is
gone, and there is no timestamp left to tell you it left.

| Delete style | Detection |
|---|---|
| Soft delete (`is_deleted` flag) | Incremental pull sees it as a normal update |
| Hard delete | CDC only |
| Compromise | Periodic full key reconciliation: keys present in target but absent in source → mark deleted |

### 4. How bad is late-arriving data?
Mobile clients buffer offline and report days later. An event with `event_time = 2026-09-01` may
land on 2026-09-05. This decides:
- whether you partition by **event time** or **ingestion time**, and
- how wide the reprocessing window has to be.

### 5. Can you trust the source clock?
- Clock skew across distributed source nodes.
- Commit order ≠ timestamp order. **Long transactions are the classic data-loss bug**: `updated_at`
  is stamped at transaction *start*, but the row commits minutes later. Your pull at T doesn't see
  the uncommitted row; your pull at T+1 filters it out because its timestamp is already below the
  watermark. The row is lost permanently.
- Defense: pull from `watermark - safety_buffer` (e.g. 15 min) and make writes idempotent so the
  overlap is harmless.

---

## Part 2 — Three Questions About the Target

### 6. Does the target need history?
| Requirement | Approach |
|---|---|
| Current state only | Upsert / `MERGE` |
| Full change history | SCD Type 2, or append-only change log + "latest" view |

**Preference:** store an append-only change log at the base layer and derive current state in a
view or materialization on top. Reruns are safe, history is free, and debugging can answer
"what did this row look like at the time?".

### 7. Is a rerun safe? (idempotency)
This is the dividing line. You *will* rerun — upstream corrections, bug fixes, backfills.

| Write pattern | Rerun-safe | Use when |
|---|---|---|
| `INSERT INTO` | No — duplicates | Only with dedup downstream or true exactly-once delivery |
| `INSERT OVERWRITE PARTITION` | Yes, partition-level | Batch default, when the source can be recomputed per partition |
| `MERGE INTO` | Yes, key-level | Mutable tables with a stable primary key |

**A non-idempotent production pipeline is a bug, not a trade-off.**

### 8. What is the volume and cost profile?
`MERGE` scans the target to find matches, which gets expensive on large tables. If daily change
volume is ~1% of the table but the `MERGE` still scans everything, partition pruning plus overwrite
is usually cheaper.

---

## Part 3 — Four Implementation Patterns

### Pattern A — Append-only + partition overwrite
```sql
INSERT OVERWRITE TABLE fct_events PARTITION (dt = '2026-09-05')
SELECT * FROM staging_events WHERE dt = '2026-09-05';
```
Simplest and fully idempotent. Partition on a key that can be **recomputed** (usually event date).
Backfill = loop the same statement over N days.

### Pattern B — Watermark pull + MERGE
```sql
-- extract: read last_watermark from a control table
SELECT *
FROM   source_orders
WHERE  updated_at >  :last_watermark - INTERVAL '15 minutes'
  AND  updated_at <= :run_start_time;
```
```sql
-- load: dedup to one row per key FIRST, then merge
MERGE INTO dim_orders AS t
USING (
  SELECT * FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY order_id ORDER BY updated_at DESC) AS rn
    FROM   staging_orders
  ) WHERE rn = 1
) AS s
ON t.order_id = s.order_id
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *;
```
Rules:
- Use a **half-open interval** so a boundary row belongs to exactly one run.
- Advance the watermark to `run_start_time`, **not** `max(updated_at)` — the latter stalls forever
  if the source has an empty run.
- Always dedup by primary key before `MERGE`; multiple updates to the same key in one batch
  otherwise make the merge error out or behave non-deterministically.
- Persist the watermark in a transactional control table, committed with the data write.

### Pattern C — CDC (Debezium, binlog, Snowflake streams)
Gives the full `INSERT / UPDATE / DELETE` stream. The only correct way to handle hard deletes.
Costs: operational complexity, schema-change handling, and stitching the initial snapshot to the
ongoing stream without gaps or duplicates.

### Pattern D — Rolling reprocess window
```sql
INSERT OVERWRITE TABLE fct_orders PARTITION (dt)
SELECT ... FROM source WHERE dt >= CURRENT_DATE - 7;
```
The cheapest cure for late-arriving data: trade a little compute for an entire class of bugs.
Teams often over-optimize toward "compute only the delta" and end up with fragile catch-up logic
to save five minutes of cluster time.

---

## Decision Table

| Source situation | Pattern |
|---|---|
| Event stream, immutable, no deletes | A |
| `updated_at` present, soft deletes, short mutation window | B |
| `updated_at` present, long mutation window / heavy lateness | D, or B with a wide lookback |
| Hard deletes that must be accurate | C (or B + scheduled reconciliation) |
| Full change history required | C, or append-only change log + view |
| Small source (< ~10M rows) | Don't go incremental — full refresh |

**Most important line in the table is the last one.** Many incremental pipelines should not be
incremental. If a full refresh fits the time and cost budget, take it: no watermark bugs, no
late-arrival handling, no idempotency concerns, no reconciliation job. Incremental load is a forced
optimization, not a default design.

---

## Production Failure Modes

1. **Watermark not durably stored** — kept in memory or config, so a crash loses the position.
   Store it in a transactional control table alongside the data commit, or rely on overwrite
   semantics to make the position recoverable.
2. **Timezone mismatch** — source stamps UTC, partition key is local time; the boundary hour is
   silently dropped or duplicated every day.
3. **Schema evolution** — the source adds a column, `SELECT *` quietly drops it, and nobody notices
   for six months. Pin the column list or explicitly handle drift (see medallion schema enforcement).
4. **Backfill code diverges from daily code** — two code paths always drift apart. Use one
   parameterized job with a date range argument.
5. **Monitoring job success instead of data volume** — upstream stops publishing, the incremental
   job successfully loads 0 rows every day, and the dashboard is green for a quarter. At minimum,
   alert on a row-count drop versus the trailing average.

---

## Checklist for a New Incremental Pipeline

- [ ] Change timestamp identified, and confirmed to be updated on every write
- [ ] Delete semantics known (soft / hard / none) and handled
- [ ] Mutation window measured → lookback window set from it
- [ ] Late-arrival profile measured → partition key chosen (event vs ingestion time)
- [ ] Safety buffer applied for long transactions / clock skew
- [ ] Write is idempotent (overwrite or merge), verified by running the job twice
- [ ] Watermark durably persisted and monotonically advancing
- [ ] Same code path used for backfill and scheduled runs
- [ ] Row-count and freshness alerts wired up
