# Spark + Iceberg: Tuning, Joins, Performance
**Source:** DataExpert.io Boot Camp — Videos 10 & 11 (Week 3, Infrastructure Track)
**Topic:** Iceberg table format, Spark memory tuning, join strategies, UDFs, caching

---

## Key Concepts

### Apache Iceberg
- Open table format for huge analytic datasets
- Supports ACID transactions, schema evolution, time travel, and partition evolution
- Preferred over Hive tables for: concurrent writes, safe schema changes, reliable deletes

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

### Partitioning
- **Too few partitions** → large tasks, OOM, slow
- **Too many partitions** → scheduler overhead, small file problem
- Rule of thumb: target 100–200MB per partition
- Use `repartition(n)` before wide transforms, `coalesce(n)` before writes

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
