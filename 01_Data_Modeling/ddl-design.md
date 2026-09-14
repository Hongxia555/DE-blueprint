# DDL Design
**Source:** Session notes — DDL as physical design, from MPP columnar warehouses to lakehouse tables
**Topic:** What a `CREATE TABLE` statement actually commits you to: grain, types, storage layout, update mechanics, and evolution

---

## The Core Idea

DDL is the bridge between the logical model and everything downstream: query latency, compute cost,
update mechanics, lineage granularity, and how painful the next schema change will be. Almost every
line in a `CREATE TABLE` is a bet about **how the table will be read and how it will be written**.

A useful test for any DDL decision: *what breaks, and how expensively, if I get this wrong and only
find out in six months?* Column order in a sort key is cheap to fix (rewrite the data). Grain is not.

See [SCD & Idempotency](./scd-idempotency.md) · [Incremental ETL Design](../05_Pipeline_Design/incremental-etl-design.md) · [Backfill Design](../05_Pipeline_Design/backfill-design.md) · [Data Contracts](../06_Data_Quality_Ops/data-contracts.md)

---

## Part 1 — Grain First

Before any type or partition decision: **one row means what, exactly?**

Write it as a sentence, then formalize it as the primary key:

| Grain sentence | Primary key |
|---|---|
| One row per order line item | `(order_id, line_item_id)` |
| One row per user per day | `(user_id, ds)` |
| One row per device per signal per event timestamp | `(device_id, signal_name, event_ts)` |
| One row per customer per attribute-version | `(customer_id, valid_from)` |

Consequences of getting this wrong:

- **Grain too fine** → the table is huge, every consumer aggregates before use, and the aggregation
  logic is duplicated (and diverges) across teams.
- **Grain too coarse** → the question that needs the finer detail cannot be answered without a
  backfill from raw, and raw may already be past retention.
- **Grain undeclared** → duplicates accumulate silently. Nobody notices until a revenue metric is
  1.7× the truth.

Put the grain in a table comment. It is the single most valuable piece of documentation and it lives
in the DDL, where a lineage tool can read it:

```sql
COMMENT ON TABLE fct_order_items IS
  'Grain: one row per order line item. PK (order_id, line_item_id). Partitioned by order_date (event time, UTC).';
```

**Nuance:** in most lakehouse engines (Iceberg, Delta, Hive) a declared `PRIMARY KEY` is
*informational only* — it is not enforced. Declaring it is still worth doing (planners and lineage
tools use it, humans read it), but the actual enforcement has to be a data quality test.

---

## Part 2 — Type Decisions That Cost Money Later

| Decision | Rule of thumb | Why |
|---|---|---|
| Money / quantities | `DECIMAL(p, s)`, never `FLOAT`/`DOUBLE` | Binary floats cannot represent 0.1; sums drift and reconciliation against finance fails |
| Timestamps | Store UTC; be explicit about `TIMESTAMP` vs `TIMESTAMPTZ`; keep event time and processing time as separate columns | Mixing local times across regions silently misassigns rows to partitions |
| Dates vs timestamps | `_at` = timestamp, `_date`/`ds` = date | Naming carries the type so readers don't guess |
| Low-cardinality categories | Short `VARCHAR` with a documented allowed set, or an int code + dimension table | Dictionary encoding makes strings cheap in columnar formats; the value is readability vs join-free lookups |
| Free text / blobs | Bounded `VARCHAR(n)` where the engine cares; never a default `VARCHAR(MAX)` habit | Some engines allocate by declared width for buffers and sorts; unbounded types poison memory estimation |
| IDs | Fixed-width type, consistent across all tables that join on it | Cross-type joins (`BIGINT` vs `STRING` user_id) force casts and defeat partition/file pruning |
| Booleans | `BOOLEAN`, named `is_` / `has_` | Three-valued nullable "flags" stored as `INT` become a semantics guessing game |
| Semi-structured | `STRUCT`/`ARRAY` when the shape is known; `MAP<string, x>` / raw JSON only when it genuinely is not | See below |

### The semi-structured trap

`MAP<STRING, FLOAT>` or a JSON blob is attractive for sparse, fast-changing attribute sets (ML
features, event properties, device signals). The cost is real and usually underpriced:

- Column-level lineage stops at the map. Every consumer looks like it depends on "the whole column."
- Schema validation cannot fire — a key that disappears is not a schema change, it is just absent.
- Columnar pruning and statistics stop working; you scan and decode the whole structure.
- Type errors move from write time to read time.

Practical compromise: **promote the stable subset to real columns, keep the long tail in the map.**
Promote a key out of the map as soon as more than one consumer references it.

### Nullability

`NOT NULL` is the cheapest data quality check that exists — it runs at write time and costs nothing.
Use it wherever the semantics are truly "must exist" (keys, event time, partition column).

Caveat: several lakehouse engines do not enforce nullability the way an OLTP database does. Verify
your engine's behavior; where it is not enforced, the constraint is documentation and the real check
belongs in the pipeline's audit step.

---

## Part 3 — Physical Layout: Partitioning and Sorting

This is where DDL directly buys or burns money. Two separate mechanisms, often confused:

- **Partitioning** — coarse, directory/metadata-level elimination. Skips *whole groups of files*.
- **Clustering / sort order** — fine, file- and block-level elimination via min/max statistics.
  Skips *files or row groups within a partition*.

### Choosing a partition key

Rules, in priority order:

1. **The partition must be the unit of reprocessing.** If you rerun a day, the day must be one
   partition, so `INSERT OVERWRITE PARTITION` is atomic and idempotent. This ties DDL directly to
   [Incremental ETL](../05_Pipeline_Design/incremental-etl-design.md) and
   [Backfill](../05_Pipeline_Design/backfill-design.md) — partition granularity *is* your blast radius.
2. **It must be in the predicate of most queries.** Usually event date (`ds`).
3. **It must produce sane file sizes.** Target roughly 128 MB–1 GB per file. Too many partitions is
   the classic self-inflicted wound: metadata explosion, small files, and a planner that spends
   longer listing than scanning.
4. **It must not be high-cardinality.** `user_id` is not a partition key. It may be a sort key.

Common shapes:

| Volume per day | Partitioning |
|---|---|
| < ~1 GB | Daily, or monthly if very small |
| GBs–TBs | Daily, optionally hourly for latency-sensitive pipelines |
| Very large + one dominant filter | Daily + a low-cardinality second level (e.g. `region`, `event_type`) |

Second-level partitioning multiplies partition count — only add it when the secondary predicate is
present in most queries *and* the resulting files are still large enough.

### Sort / cluster keys

Sort order determines how effective min/max statistics are. The general rule: **filter columns
first, and within those, low cardinality before high cardinality** — because sorting is
lexicographic, only the leading columns get tight ranges.

Engine-specific names for the same idea:

| Engine | Mechanism |
|---|---|
| Iceberg | Table `SORT ORDER` (write order), plus `rewrite_data_files` with sort/z-order |
| Delta Lake | `OPTIMIZE ... ZORDER BY`, or liquid clustering |
| Snowflake | Clustering keys with automatic background reclustering |
| Redshift | `SORTKEY` (compound vs interleaved) + `DISTKEY` |
| ClickHouse | `ORDER BY` in the `MergeTree` engine definition (also the sparse index) |

Sorting is not free — it is a rewrite. It pays off when the same predicate appears in many queries
over the same data repeatedly; it does not pay off on write-once-read-once staging tables.

---

## Part 4 — MPP Columnar: Logical Table vs Physical Storage

Classic MPP columnar warehouses (Vertica is the clearest example) make a distinction worth
internalizing even if you never use one: **the table is only the logical contract; a separate
physical object decides performance.**

In Vertica that object is the **projection**:

| Layer | What it defines |
|---|---|
| Table | Column names, types, constraints — the logical schema |
| Projection | Sort order, segmentation (sharding), encoding/compression — the physical storage |

Two decisions inside a projection:

- **Sort order** — same principle as above: frequent-filter, low-cardinality columns first. Sort
  order also enables merge joins and streaming `GROUP BY` without a hash build.
- **Segmentation** — how rows are distributed across nodes:
  - `SEGMENTED BY HASH(join_key)` for large fact tables, so that tables segmented on the same key
    join **locally**, with no network shuffle.
  - `UNSEGMENTED ALL NODES` (replicated) for small dimensions, so any node can join locally.

The generalizable lesson: **co-locate what you join, replicate what is small.** The same principle
appears as `DISTKEY`/`DISTSTYLE ALL` in Redshift, bucketing in Spark/Hive, and partition keys in
distributed OLTP stores. Multiple projections on one table also mean you can serve two different
access patterns from the same logical table at the cost of write amplification — an early version of
what "materialized views" and "clustering variants" do elsewhere.

---

## Part 5 — Lakehouse Tables: What Iceberg Changes About DDL

The move to open table formats (Iceberg, Delta, Hudi) changed which DDL decisions are permanent.

**What it buys:**

1. **Storage/compute separation** — data sits as open Parquet/ORC on object storage; Spark, Trino,
   Flink and others read the same table. The DDL becomes engine-neutral.
2. **ACID + row-level `UPDATE`/`DELETE`/`MERGE`** on a data lake, which Hive-style tables could not do.
3. **Hidden partitioning** — the partition transform is declared in the DDL
   (`PARTITIONED BY days(event_ts)`), and queries filter on `event_ts` directly. No more
   "the query was slow because someone forgot `WHERE ds = ...`", and no more redundant physical
   partition column that can drift from the timestamp it was derived from.
4. **Partition evolution** — change hourly → daily without rewriting history. Old data keeps its old
   layout; new data uses the new one; the planner handles both. This removes what used to be one of
   the most expensive DDL mistakes.
5. **Metadata tree** (catalog → metadata file → manifest list → manifest) — file pruning from
   statistics without listing object storage, plus optimistic-concurrency commits.

**Schema evolution guarantees:** column-level field IDs mean `ADD` / `DROP` / `RENAME` / `REORDER`
are metadata-only and safe — a rename does not orphan data, because matching is by ID, not by name
or position. This is a genuine step change from Hive-style tables where renames were dangerous and
column order mattered.

### Row-level update mechanics: CoW vs MoR

The DDL-level write mode choice that most affects cost:

| | Copy-on-Write (CoW) | Merge-on-Read (MoR) |
|---|---|---|
| Write path | Rewrite every data file touched by the change | Write small delete/position files alongside |
| Write cost | High — write amplification proportional to file size, not row count | Low |
| Read cost | Low — files are already clean | Higher — merge deltas at query time |
| Maintenance | Fewer moving parts | Requires regular compaction or read cost creeps up |
| Fits | Read-heavy tables, infrequent updates, batch pipelines | Streaming upserts, CDC sinks, frequent small changes |

Set it per table, per operation type, in table properties — e.g. MoR for the CDC landing table,
CoW for the curated table many consumers scan.

### Why "just overwrite the whole partition" is still often correct

At very large scale, on append-only event streams, full-partition overwrite frequently beats
row-level updates. That is a trade-off, not backwardness:

- **Read-heavy fan-out.** With thousands of concurrent analytical queries, paying a merge cost on
  every read (MoR) is enormously more expensive in aggregate than paying a rewrite cost once.
- **Small-file pressure.** Frequent row-level writes generate huge numbers of small files and
  metadata entries, which degrade planning and stress the metadata layer.
- **The data is immutable anyway.** Event logs are append-only; there is no meaningful row-level
  update to perform. Corrections arrive as reprocessing of a whole day.
- **Idempotency is trivially achieved.** `INSERT OVERWRITE PARTITION` is atomic and rerunnable,
  which is exactly what backfill wants.

The heuristic: **row-level update mechanics earn their cost when changes are sparse, scattered, and
frequent. Partition overwrite wins when changes are dense, bounded to a partition, or rare.**

---

## Part 6 — Write Pattern Determines DDL

Decide up front which of these the table is; the DDL differs materially:

| Table type | DDL implications |
|---|---|
| **Append-only event / fact** | Partition by event time; no PK enforcement expected; sort by the common filter (entity id); CoW or plain overwrite; long retention with partition-level expiry |
| **Upsert dimension (SCD 1)** | Business key + `updated_at`; MERGE target; MoR viable; needs a dedup-by-key step before merge |
| **Historized dimension (SCD 2)** | Key + `valid_from` / `valid_to` / `is_current`; the PK is `(business_key, valid_from)`; as-of joins depend on this DDL existing |
| **Cumulative / stateful aggregate** | Partition = snapshot date; each partition depends on the previous, so backfills are strictly serial |
| **Staging / raw landing** | Loose types, permissive schema, short retention; the place to fail *loudly* on drift rather than coerce |

See [SCD & Idempotency](./scd-idempotency.md) and
[Cumulative Dimensions](./cumulative-dimensions.md) for the modeling side of these.

---

## Part 7 — Evolution: DDL's Time Dimension

Every table's DDL will change. Design so that change is a metadata operation, not a migration project.

| Change | Compatibility | Typical safety |
|---|---|---|
| Add nullable column | Backward compatible | Safe |
| Add column with default | Backward compatible | Safe |
| Drop column | Breaking for readers selecting it | Deprecate first, drop later |
| Rename column | Safe in Iceberg/Delta (field IDs); breaking for consumer SQL | Add alias/view, migrate consumers, then rename |
| Widen type (`INT` → `BIGINT`, `DECIMAL(10,2)` → `DECIMAL(18,2)`) | Backward compatible | Safe |
| Narrow type | Breaking, may lose data | Requires backfill + validation |
| Change partition spec | Safe in Iceberg (partition evolution) | Safe; old files keep old spec |
| Tighten nullability | Breaking if existing nulls | Clean data first |
| **Change a column's meaning** | Invisible to every tool | **Most dangerous** — see below |

**Semantic drift** is the change no schema check catches: `revenue` quietly becomes net instead of
gross, `active_user` changes its window from 28 to 30 days, a status enum gains a value that
consumers' `CASE` statements do not handle. The DDL is unchanged, so CI passes, contracts pass, and
every downstream number moves. Defenses: version the column (`revenue_net_v2`) rather than
redefining it, put the definition in the column comment, and treat a definition change as a breaking
change requiring consumer sign-off. Enum churn specifically is worth handling deliberately — see
[Enums](./enums.md).

---

## Part 8 — Schema as Code

Once more than a handful of tables exist, DDL should not be typed into a console.

- **One definition, many artifacts.** Keep the table definition in the repo (YAML, dbt `schema.yml`,
  protobuf/Avro). Generate the `CREATE TABLE`, the docs, the quality tests, and typed
  producer/consumer code from it. Hand-written `ALTER` on production is how the repo definition and
  reality diverge.
- **Diff in CI.** Compare the proposed schema against the deployed one, classify each change as safe
  or breaking using the table above, and fail the build on breaking changes without an owner
  approval and a listed downstream impact set.
- **Ownership and cost tags in the DDL.** Table properties carrying `owner`, `team`, `domain`,
  `cost_center`, `sla`, `retention` are what make cost attribution and "who do I page" answerable
  automatically instead of by asking around.
- **Comments are machine-readable metadata,** not decoration. Grain, units, and definitions in
  `COMMENT` fields flow into catalogs and lineage tools.

This is the seam where DDL stops being a modeling task and becomes platform work: quality, lineage,
cost attribution, and transparency all bottom out in what the table definition declares.
See [Data Contracts](../06_Data_Quality_Ops/data-contracts.md).

---

## Naming Conventions

Consistency beats cleverness; the point is that a reader can infer type and semantics from the name.

| Pattern | Meaning |
|---|---|
| `fct_` / `dim_` | Fact / dimension table |
| `stg_` / `int_` | Staging / intermediate model |
| `_at` | Timestamp |
| `_date`, `ds` | Date |
| `_id` | Identifier (consistent type everywhere) |
| `is_`, `has_` | Boolean |
| `_amount`, `_qty`, `_cnt` | Measures, with units documented |
| `_v2` suffix | Explicitly versioned redefinition |

---

## Anti-Patterns

| Anti-pattern | Why it hurts |
|---|---|
| `CREATE TABLE AS SELECT *` | Schema is whatever the source happened to be that day; upstream drift silently changes the table |
| Everything `STRING` | Defers all type errors to read time and destroys statistics-based pruning |
| No grain declared | Duplicates go undetected until a metric is visibly wrong |
| High-cardinality partition key | Millions of tiny partitions; planning costs more than scanning |
| Partition granularity finer than the reprocessing unit | Backfill can no longer be atomic per partition |
| Manual `ALTER` on production | Repo definition drifts from reality; the next deploy fights it |
| Floats for money | Reconciliation against finance fails and nobody can explain the pennies |
| 400-column table with no owner | Nobody can safely change or delete anything |
| Redefining a column's meaning in place | Breaks every consumer with zero signal |
| Everything in one JSON/map column | No lineage, no validation, no pruning |

---

## Checklist for a New Table

- [ ] Grain written as a sentence, and encoded as a primary key
- [ ] Grain and column definitions in `COMMENT` fields
- [ ] Money is `DECIMAL`; timestamps are UTC with event time and processing time separate
- [ ] `NOT NULL` on keys, event time, and partition columns
- [ ] Partition key = the reprocessing unit, present in most query predicates, produces sane file sizes
- [ ] Sort/cluster keys chosen from actual query predicates, not guesses
- [ ] Write pattern decided (append / upsert / SCD 2 / cumulative) and DDL matches it
- [ ] Update mode chosen deliberately (CoW vs MoR vs partition overwrite)
- [ ] Retention and partition expiry policy set
- [ ] Owner, team, cost center, SLA in table properties
- [ ] Definition lives in the repo, deployed through CI, with breaking-change detection
- [ ] Downstream consumers listed, so a breaking change has a known blast radius
