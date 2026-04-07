# Spark + Iceberg: Tuning, Joins, Performance
**Source:** DataExpert.io Boot Camp — Videos 10 & 11 (Week 3, Infrastructure Track)
**Topic:** Iceberg table format, Spark memory tuning, join strategies, UDFs, caching

---

## Key Concepts

### Apache Iceberg
- Open table format for huge analytic datasets
- Supports ACID transactions, schema evolution, time travel, and partition evolution
- Preferred over Hive tables for: concurrent writes, safe schema changes, reliable deletes

#### What "open table format" means
Raw files in a data lake (Parquet, ORC) have no built-in concept of schema, transactions, or history — they're just files. A table format adds a **metadata layer** on top to manage all of that.

"Open" means engine-agnostic: Spark, Trino, Flink, Athena, DuckDB can all read and write the same Iceberg table without going through a vendor. You own the files and metadata.

#### How Iceberg works internally
```
S3 bucket/
├── metadata/
│   ├── v1.metadata.json   ← schema, partition spec, snapshot history
│   ├── snap-001.avro      ← snapshot: which files are "current"
│   └── snap-002.avro      ← next snapshot after a write
└── data/
    ├── ds=2024-01-01/file1.parquet
    └── ds=2024-01-02/file2.parquet
```
A **snapshot** is a complete picture of the table at a point in time — a list of which data files are valid. All Iceberg features are built on top of this snapshot mechanism.

#### Key features explained

| Feature | What it means | How Iceberg enables it |
|---|---|---|
| **ACID transactions** | A write is fully visible or not at all — no partial states | Write creates a new snapshot atomically; readers see the old snapshot until commit completes |
| **Time travel** | Query the table as it looked at a past point in time | Point to an old snapshot — `VERSION AS OF 12345` maps to the files valid at that moment |
| **Schema evolution** | Add, rename, or drop columns without rewriting data files | Old files missing a new column return nulls; Iceberg reconciles via metadata, not file rewrites |
| **Partition evolution** | Change partitioning strategy without rewriting all data | New partitioning applies to new files only; old files retain their original layout |

**Schema evolution caveat:** safe changes (add nullable column, rename) are allowed; unsafe changes (e.g. string → int type cast) are blocked by Iceberg to protect downstream readers.

#### Why Iceberg beats Hive tables
Hive tracks partitions via directory scans — it looks at folder structure to determine what data exists. This causes:
- Concurrent write corruption (two writers creating the same partition)
- Deletes requiring full partition rewrites
- Risky schema changes

Iceberg's metadata layer eliminates all of these by making every state change a new snapshot.

```sql
-- Create Iceberg table
CREATE TABLE catalog.db.events (
  user_id BIGINT,
  event_type STRING,
  ds DATE
) USING iceberg
PARTITIONED BY (ds);

-- Time travel
SELECT * FROM catalog.db.events VERSION AS OF 12345;
SELECT * FROM catalog.db.events TIMESTAMP AS OF '2024-01-01 00:00:00';
```

### Spark Memory Tuning
| Config | Purpose | Typical Value |
|--------|---------|---------------|
| `spark.executor.memory` | Heap per executor | 4g–16g |
| `spark.memory.fraction` | Fraction for execution + storage | 0.6 |
| `spark.memory.storageFraction` | Cache priority within memory | 0.5 |
| `spark.sql.shuffle.partitions` | Post-shuffle partition count | 200 (default), tune to ~2-4x cores |

#### How Spark divides executor memory
```
Total executor memory (spark.executor.memory = 8g)
│
├── Reserved memory (~300MB, fixed overhead)
│
└── Usable memory (remaining)
    │
    ├── Spark managed memory (spark.memory.fraction = 0.6)
    │   ├── Execution memory  ← shuffles, sorts, joins, aggregations
    │   └── Storage memory    ← cached DataFrames (.cache())
    │       (spark.memory.storageFraction = 0.5 of the 0.6)
    │
    └── User memory (remaining 0.4) ← Python/Scala objects, UDFs
```
Execution and storage share their pool and can borrow from each other — but execution can evict cached data if it needs room, not the other way around.

#### What each config actually does
- **`spark.executor.memory`** — total RAM per executor. Too low → spill to disk. Too high → fewer executors fit on the node.
- **`spark.memory.fraction`** — fraction of usable memory Spark manages. The remaining 0.4 is for user-space objects. Lower this if you have heavy UDFs or large Python objects.
- **`spark.memory.storageFraction`** — within the Spark-managed pool, how much is protected for cache. Higher = cached DataFrames less likely to be evicted by joins/shuffles.
- **`spark.sql.shuffle.partitions`** — partitions created after a shuffle (join, groupBy). Default 200 is often wrong. Target ~128–256MB per partition — if post-shuffle data is 20GB, aim for ~100–150 partitions.

#### The spill problem
When execution memory runs out mid-operation, Spark writes intermediate data to local disk and reads it back. This kills performance — it's the main thing memory tuning tries to prevent.

Signs you're spilling: Spark UI → stages tab → non-zero "Spill (disk)" column.

Fixes:
1. Increase `spark.executor.memory`
2. Repartition before wide transforms to reduce per-partition size
3. Use broadcast join if one side is small — avoids the shuffle entirely

#### Mental model
- **Execution memory** = counter space (active work — chopping, mixing)
- **Storage memory** = fridge (keeping things ready to reuse)
- **Spill to disk** = running to the grocery store mid-recipe because you ran out of counter space

### Join Strategies
| Strategy | When to Use | Config |
|----------|------------|--------|
| Broadcast Join | Small table (< ~10MB) | `spark.sql.autoBroadcastJoinThreshold` |
| Sort-Merge Join | Large + large tables, on sorted keys | Default for large joins |
| Shuffle Hash Join | Medium tables where sort is expensive | Force with hint: `/*+ SHUFFLE_HASH */` |

```python
# Force broadcast join
from pyspark.sql.functions import broadcast
df_result = large_df.join(broadcast(small_df), "user_id")
```

#### Why join strategy matters
When Spark joins two DataFrames, it has to physically move or compare data across executors. The strategy determines whether the join causes a massive network shuffle or avoids it entirely.

#### Broadcast Join
One table is small enough to be copied to every executor — no shuffle needed. Each executor joins its partition of the large table against its local copy of the small table.

```python
from pyspark.sql.functions import broadcast
result = large_df.join(broadcast(small_df), "user_id")
```
- Zero network shuffle for the large table — fastest option
- Risk: broadcasting a table that's actually large → OOM on executors

#### Sort-Merge Join
Both tables are shuffled so matching keys land on the same executor, then each partition is sorted and merged. Spark's default for large-large joins.

```
Table A → shuffle by key → sort → merge
Table B → shuffle by key → sort → merge
```
- Predictable and safe for any size
- Cost: two full network shuffles — expensive but reliable

#### Shuffle Hash Join
Shuffles both tables by key like sort-merge, but builds a hash table in memory on the smaller side instead of sorting. Faster than sort-merge when sorting is the bottleneck.

```sql
SELECT /*+ SHUFFLE_HASH(small_table) */ *
FROM large_table JOIN small_table ON large_table.id = small_table.id
```
- Risk: hash table must fit in memory — spill or OOM if it doesn't

#### Decision tree
```
Is one side small (< 10MB)?
  → Yes: Broadcast join
  → No: Is the smaller side medium and sort is expensive?
      → Yes: Shuffle Hash join
      → No: Sort-Merge join (default)
```

Spark's Catalyst optimizer picks automatically based on table size statistics. Override with hints when statistics are stale or tables were just written.

### Partitioning
- **Too few partitions** → large tasks, OOM, slow
- **Too many partitions** → scheduler overhead, small file problem
- Rule of thumb: target 100–200MB per partition
- Use `repartition(n)` before wide transforms, `coalesce(n)` before writes

#### repartition vs coalesce

**`repartition(n)`** — full shuffle, redistributes data evenly across N partitions across the network. Expensive, but gives balanced partition sizes. Use before wide transforms (joins, groupBy, aggregations) so no single executor gets overwhelmed with skewed data.

```python
# Redistribute evenly before a heavy groupBy
df.repartition(200, "country").groupBy("country").agg(...)
```

**`coalesce(n)`** — no shuffle, merges existing partitions on the same executor. Cheap, but can only reduce partition count. Use before writes to avoid the small files problem without paying shuffle cost.

```python
# Collapse to fewer files before writing
df.coalesce(10).write.parquet("s3://bucket/output/")
```

| | `repartition(n)` | `coalesce(n)` |
|---|---|---|
| Shuffle | Yes — full | No |
| Can increase partitions | Yes | No |
| Data balance | Even | Uneven (merges local partitions) |
| Use before | Wide transforms | Writes |
| Cost | High | Low |

### UDFs vs. Built-ins
- **Always prefer built-in Spark functions** — JVM-native, optimized by Catalyst
- Python UDFs: serialize data to Python process → 10-100x slower
- Pandas UDFs (vectorized): much faster than row UDFs, acceptable for complex logic

### Caching
```python
df.cache()        # lazy — cache on first action
df.persist()      # same, but can specify storage level
df.unpersist()    # release cache when done

# When to cache: reused multiple times in same job
# When NOT to cache: single-use, larger than available memory
```

---

## Interview Angles
- Why does Iceberg solve the "small files problem"? → Compaction + metadata layer tracks files without directory scans
- What causes an OOM in Spark? → Data skew (one partition too large), joining without broadcast, collecting large datasets to driver
- How do you debug a slow Spark job? → Spark UI → check skewed stages, shuffle read/write sizes, GC time

---

## Notes (from transcript)

### Zach's Spark history
- Learned Java MapReduce at Teradata (2014), Hive at Facebook (2015–2017)
- Was one of the **3rd or 4th engineers to use Spark at Facebook** (2017)
- Used Spark from 2017–2023: "I attribute learning this technology as one of the main reasons why I was wildly successful in big Tech"

### The basketball team analogy (how Spark works)
- **Plan** = the play (your DataFrame/SQL code — evaluated **lazily**)
- **Driver** = the coach (orchestrates, collects results)
- **Executors** = the players (do the actual work in parallel)
- Plan doesn't execute until you "take a shot" (write output or `.collect()`)

### Spilling to disk = Spark going back to being Hive
- Spark's advantage: keeps data in RAM between stages
- When memory is insufficient → **spill to disk** → performance degrades dramatically
- "Spilling to disk makes Spark essentially go back to being Hive and being kind of slow"
- Fix: tune `spark.executor.memory`, repartition before wide transforms

### When NOT to use Spark
1. You're the only one on your team who knows Spark → maintenance/bus factor risk
2. Your company has already standardized on BigQuery/Snowflake → **homogeneity of pipelines matters**
   - "It's better to have 20 BigQuery pipelines than 19 BigQuery + 1 Spark pipeline"

### Spark vs Presto (why Facebook adopted Spark over Presto)
- Presto: everything in memory — if operation doesn't fit, **it just fails**
- Zach worked in notifications at Facebook → couldn't use Presto (too much data)
- Spark: spills to disk as fallback, far more resilient for large datasets

### Key config Zach highlights
- `spark.sql.shuffle.partitions` default is 200 — often wrong for large or small jobs
- Target **128–256MB per partition** as a rule of thumb
- Use `explain()` to inspect query plan before running expensive jobs
