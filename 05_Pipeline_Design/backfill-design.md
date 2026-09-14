# Backfill Design
**Source:** Session notes — backfill strategy discussion
**Topic:** Deciding how to recompute historical partitions safely: triggers, ordering, isolation, verification

---

## The Core Idea

Backfill is not "run the job for old dates." It is a controlled bulk rewrite of history, and the
design is driven by two things: **why** you are backfilling, and **whether the transformation carries
state across time**. Everything else follows.

Prerequisite: the pipeline must already be idempotent. If a rerun is not safe, you do not have a
backfill problem — you have an idempotency problem.
See [Incremental ETL Design](./incremental-etl-design.md) · [SCD & Idempotency](../01_Data_Modeling/scd-idempotency.md)

---

## Part 1 — Why Are You Backfilling?

The trigger determines whether the *inputs* or the *logic* changed, and that changes everything.

| Trigger | Inputs changed? | Logic changed? | Implication |
|---|---|---|---|
| New pipeline, needs history | — | — | Source retention is the binding constraint |
| Bug fix in transformation | No | Yes | Output will differ from previously published numbers |
| Upstream corrected its data | Yes | No | Must also recompute every downstream table |
| New column / new dimension added | No | Yes (additive) | Often can backfill the column alone, not the whole row |
| Data loss / incident recovery | Yes | No | Time-critical; verify against a known-good snapshot |

**The uncomfortable one is the bug fix.** Recomputing 3 years with today's logic produces
"what the numbers would have been if we'd always been right", which will not match what was
reported at the time. Whether that is acceptable is a **business and compliance decision**, not an
engineering one — surface it before running, don't decide it silently.

---

## Part 2 — Six Questions Before Running

### 1. Is the transformation stateful across time?
This is the single biggest fork in the road.

| Type | Example | Backfill order |
|---|---|---|
| **Stateless** — partition depends only on its own input | Daily event aggregation, dimension snapshot from a full extract | Any order, **parallel** |
| **Stateful / cumulative** — partition N reads partition N-1 | SCD 2 dimensions, cumulative tables, running totals, sessionization spanning days, retention arrays | **Strictly sequential, oldest first** |

Running a cumulative backfill in parallel silently produces garbage: every partition reads a
prior-day state that hasn't been written yet. There is no error, just wrong numbers.

### 2. Does the source still have the history?
- Kafka retention is typically 7 days; a topic cannot be replayed from 2 years ago.
- Operational databases purge; a `DELETE` two years ago left no trace.
- Object-storage raw/bronze layers are usually the *only* replayable history — which is the main
  argument for keeping an immutable raw landing zone.
- If history is gone, the honest options are: rebuild from a downstream snapshot or backup, or
  accept a truncated backfill window and document the cutoff.

### 3. Was the source schema the same back then?
Old partitions may have different columns, different enum values, different units, or a different
timezone convention. Backfill code often needs **era-aware branching**, and pretending otherwise
produces nulls that look like real zeros.

### 4. What is the blast radius downstream?
- Which downstream tables read this one, and do they also need recomputing?
- Will rewriting partitions emit change events / trigger CDC consumers again?
- Will dashboards read a half-finished backfill and show a dip?
- Will freshness or volume monitors fire a storm of alerts?

Answer these *before* the run, and mute or coordinate deliberately rather than reactively.

### 5. What does it cost, and does it fit?
Measure one partition, then multiply. If a day costs $8 and you need 900 days, that is a $7,200
decision that deserves an explicit sign-off — and probably a narrower window.

### 6. How will you know it worked?
Decide the verification query *before* running, not after. See Part 5.

---

## Part 3 — Four Execution Patterns

### Pattern A — Parameterized rerun of the production job
```bash
for dt in $(seq_dates 2026-01-01 2026-03-31); do
  run_job --start "$dt" --end "$dt"
done
```
Same code, same target, date range as a parameter. **Default choice** for stateless pipelines with a
modest window. The rule it enforces: never write a separate "backfill script" — two code paths
guarantee divergence.

### Pattern B — Shadow table, verify, then swap
```sql
-- 1. build into a parallel table
INSERT OVERWRITE TABLE fct_orders_backfill PARTITION (dt) SELECT ...;
-- 2. verify (see Part 5)
-- 3. atomic swap or partition-level exchange
ALTER TABLE fct_orders EXCHANGE PARTITION (dt = '...') WITH TABLE fct_orders_backfill;
```
Use when the target is production-critical or the change is large. Consumers never see partial
state, and rollback is "don't swap." This is [Write-Audit-Publish](./write-audit-publish.md)
applied at backfill scale.

### Pattern C — Chunked with a checkpoint ledger
```sql
CREATE TABLE backfill_progress (
  backfill_id   STRING,
  partition_dt  DATE,
  status        STRING,   -- pending | running | done | failed
  rows_written  BIGINT,
  finished_at   TIMESTAMP
);
```
Drive the loop from this table and mark each partition done as it lands. A 900-day backfill *will*
be interrupted — by preemption, quota, or a human. Resuming must mean "continue", not "start over."
Also gives you an audit trail of which partitions were rewritten and when.

### Pattern D — Parallel run and compare (for logic changes)
Run new logic into a shadow table alongside the old logic for a recent overlap window, diff the two,
explain every difference, then backfill and cut over. The diff is the artifact that gets reviewed —
it converts "trust me, the fix is right" into evidence.

---

## Decision Table

| Situation | Pattern |
|---|---|
| Stateless, small window, non-critical table | A |
| Stateless, large window | A + C (chunked, parallel workers) |
| Cumulative / SCD 2 / sessionization | A + C, **serial, oldest → newest** |
| Production-critical target, or numbers change | B |
| Transformation logic changed | D, then B |
| Incident recovery under time pressure | B, narrowest window that closes the gap |

---

## Part 4 — Operational Rules

- **Isolate the resources.** Run backfills in a separate pool / cluster / warehouse with lower
  priority. A backfill that starves the daily SLA turns one problem into two.
- **Never let backfill and scheduled runs write the same partition concurrently.** Either pause the
  schedule for the overlap, or make the backfill window strictly older than the scheduled window.
- **Watch out for scheduler auto-catchup.** Airflow `catchup=True` on a DAG with an old
  `start_date` launches a mass backfill nobody asked for. Set `catchup=False` and backfill
  deliberately.
- **Backfill order for stateless jobs: newest first.** Recent partitions are what people actually
  query, so value lands early and a cancelled run still leaves the useful half done. For cumulative
  jobs this is not an option — oldest first is mandatory.
- **Record the metadata.** Tag rewritten partitions with a backfill id and timestamp. Six months
  later, "why does this month look different" needs an answer.
- **Cap the concurrency against the source.** A 50-way parallel backfill reading a production
  replica is an outage waiting to happen.

---

## Part 5 — Verification

Decide these before the run:

1. **Overlap check** — recompute a window that was already correct and diff against the existing
   output. Zero unexplained differences is the entry ticket.
2. **Row count per partition vs. the historical trend** — a partition that comes back 40% lighter
   means the source no longer has that history.
3. **Aggregate reconciliation** — sum a key measure by month, old vs. new, and explain every delta
   in business terms.
4. **Boundary partitions** — check the first and last partition explicitly; off-by-one on the range
   is the most common backfill bug.
5. **Downstream refresh** — confirm dependent tables were recomputed, not just the base table.

```sql
-- overlap diff skeleton
SELECT dt,
       COUNT(*) FILTER (WHERE o.pk IS NULL) AS only_in_new,
       COUNT(*) FILTER (WHERE n.pk IS NULL) AS only_in_old,
       COUNT(*) FILTER (WHERE o.pk = n.pk AND o.measure <> n.measure) AS changed
FROM   fct_orders          o
FULL OUTER JOIN fct_orders_backfill n USING (pk, dt)
WHERE  dt BETWEEN :overlap_start AND :overlap_end
GROUP BY dt ORDER BY dt;
```

---

## Common Failure Modes

1. **Joining historical facts to the *current* dimension table** — a user who moved from UK to US
   makes two years of history relabel themselves. Use an as-of join against SCD 2 effective dates.
2. **Assuming idempotency without testing it.** Run one partition twice on a scratch target and
   compare before committing to 900.
3. **Backfilling the base table and forgetting the aggregates** — mismatched layers are worse than
   consistently stale ones.
4. **No cancellation path.** Know how to stop it cleanly and what state that leaves behind.
5. **Silent alert fatigue.** Muting monitors for the backfill and forgetting to unmute them.
6. **Blowing the file layout.** Rewriting a year of partitions with a large parallel job often
   produces thousands of tiny files; schedule compaction / `OPTIMIZE` after the backfill.

---

## Backfill Runbook Template

```
Backfill: <name / id>
Reason: <new pipeline | bug fix | upstream correction | new column | recovery>
Range: <start_dt> .. <end_dt>   (N partitions)
Stateful: <yes → serial oldest-first | no → parallel, K workers>
Source history available: <yes | truncated at ...>
Numbers will change: <yes/no; who signed off>
Target: <direct | shadow table + swap>
Resource pool: <isolated pool/cluster>
Schedule conflict handling: <paused | non-overlapping window>
Downstream to recompute: <list>
Monitors muted: <list + unmute owner>
Estimated cost: <per-partition × N>
Verification: <overlap window, reconciliation queries>
Rollback: <don't swap | restore from snapshot X>
```
