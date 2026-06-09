# PySpark FAQ

**Related:** [[pyspark-overview]] · [[pyspark-example-headspace]] · [[DE_concepts]]

> Curated Q&A for data engineer interviews. Organized by topic — concepts first, then code patterns, then performance/production.

---

## 1. Core Concepts

**Q: What is lazy evaluation? Why does it matter?**
Transformations (`.filter`, `.withColumn`, `.groupBy`) don't run immediately — Spark builds a DAG of operations and executes only when an action is called (`.write`, `.count`, `.collect`). This lets the Catalyst optimizer reorder and prune operations before any data moves.

> **Plain English:** You're writing a recipe (transformations) — you're not actually cooking yet. Only when a guest arrives and says "serve the dish" (action) does anything happen. Before cooking, Spark reads the full recipe and says "steps 3 and 5 can be combined, and step 7 is unnecessary for this guest's order — I'll skip it." That's the optimization you get for free.

```python
df = df.filter(F.col("event_date") == "2026-06-01")   # nothing runs yet
df = df.withColumn("duration_mins", F.col("duration_secs") / 60)  # still nothing
df.write.format("delta").save("s3://silver/events/")  # execution happens here
```

---

**Q: What is a DAG in Spark?**
Directed Acyclic Graph — Spark's execution plan. Each node is a transformation, edges are data dependencies. "Acyclic" means no loops. Spark submits the DAG to the scheduler, which breaks it into stages separated by shuffle boundaries.

> **Plain English:** A recipe with steps that can't go backwards. "Chop onions → fry onions → add to soup" — you can't un-chop an onion. Spark maps out all the steps before cooking anything, so it can figure out the most efficient order.

---

**Q: Narrow vs. wide transformations — what's the difference?**

| | Narrow | Wide |
|---|---|---|
| **Definition** | Each input partition → one output partition | Input partitions → multiple output partitions |
| **Shuffle?** | No | Yes — data moves across the network |
| **Examples** | `map`, `filter`, `withColumn`, `union` | `groupBy`, `join`, `sortBy`, `distinct` |
| **Performance** | Fast | Expensive — minimize these |

Wide transformations are the main source of slowness. Every `groupBy` or `join` triggers a shuffle.

> **Plain English:** Narrow = each worker does their own job independently (10 cashiers each serving their own line). Wide = everyone has to compare notes first (all cashiers stop, count the total sales, then resume). The second one requires coordination — that's the expensive part.

---

**Q: RDD vs DataFrame vs Dataset — which to use?**

| | RDD | DataFrame | Dataset |
|---|---|---|---|
| **Level** | Low-level | High-level | High-level |
| **Schema** | No | Yes | Yes |
| **Type safety** | No | No | Yes (Scala/Java only) |
| **Optimization** | Manual | Catalyst optimizer | Catalyst optimizer |
| **Use when** | Fine-grained control, unstructured data | Default choice for DE work | N/A in Python |

**In practice: always use DataFrames in Python.** RDDs are rarely needed unless you're doing something DataFrames can't express.

---

**Q: What is an RDD?**
**Resilient Distributed Dataset** — Spark's original low-level data structure, predating DataFrames.

- **Resilient** — if a partition is lost (a machine dies), Spark can rebuild it by replaying the lineage (the chain of transformations that created it)
- **Distributed** — the data is split into partitions spread across many machines
- **Dataset** — a collection of any Python object: strings, dicts, tuples, custom classes — no schema required

```python
rdd = spark.sparkContext.parallelize([1, 2, 3, 4, 5])   # create from a list
rdd = spark.sparkContext.textFile("s3://raw/logs/*.txt") # create from files

rdd.map(lambda x: x * 2).filter(lambda x: x > 4).collect()
# → [6, 8, 10]
```

RDDs have no schema, no SQL, no Catalyst optimization — you're writing manual map/filter/reduce operations. DataFrames are built on top of RDDs but add all the structure and optimization back.

> **Plain English:** RDD = a big distributed Python list. You can do anything with it, but you're responsible for everything — no automatic optimization, no column names, no types. DataFrame = a distributed spreadsheet. Spark knows what's in each column and can make smart decisions about how to process it efficiently.

| | RDD | DataFrame |
|---|---|---|
| **Schema** | No — just Python objects | Yes — named, typed columns |
| **Optimization** | None — you write every step | Catalyst optimizer rewrites your plan |
| **API style** | `map`, `filter`, `reduce` | `select`, `groupBy`, `join`, SQL |
| **Speed** | Slower (no optimization) | Faster (Catalyst + Tungsten engine) |
| **Use when** | Unstructured data, custom objects, fine-grained control | Everything else — the default choice |

---

**Q: In PySpark, do we use RDD or DataFrame?**
**DataFrames — almost always.** RDDs are the old API (Spark 1.x era). DataFrames replaced them as the standard in Spark 2.0+. In real DE work today, you'd only see RDD code in legacy pipelines or very niche cases.

| Situation | Use |
|---|---|
| Structured/semi-structured data (Parquet, JSON, CSV, Delta) | DataFrame |
| SQL-style transformations | DataFrame |
| ML pipelines (MLlib) | DataFrame |
| Streaming (Structured Streaming) | DataFrame |
| Unstructured data (raw text, binary, custom objects) | RDD — rare |
| Need to process each row as an arbitrary Python object | RDD — rare |

**Why DataFrames won:**
- Catalyst optimizer rewrites your plan for free — RDDs get no optimization
- Tungsten execution engine (memory + CPU efficiency) — DataFrames only
- Readable — `df.groupBy("member_id").agg(F.sum("duration"))` vs chained `map/reduce` lambdas
- Interop — DataFrames plug directly into Delta Lake, dbt, MLlib, SQL, Pandas

**The only reason to drop to RDD today:**
```python
# calling a custom NLP library row-by-row with no DataFrame equivalent
df.rdd.map(lambda row: my_nlp_model.process(row["text"]))
# even then, most people use df.mapInPandas() or a UDF instead
```

**One-liner for interviews:** *"We use DataFrames. RDDs are the low-level foundation DataFrames are built on, but you rarely touch them directly — DataFrames give you the same power with automatic optimization on top."*

---

**Q: What is the Catalyst optimizer?**
Spark SQL's query optimizer. Automatically applies rules before execution:

- **Predicate pushdown** — move filters as early as possible to reduce data scanned
- **Projection trimming** — drop columns not needed downstream
- **Constant folding** — instead of recomputing the same fixed value on every row, Spark calculates it once at plan time and injects the result
  ```python
  # You write:
  df.filter(F.col("duration_secs") > 60 * 60 * 24)
  # Catalyst computes 86400 once at plan time — not once per row
  ```
  > **Plain English:** You're baking 1000 cookies and the recipe says "preheat to 100 + 80 + 20 degrees". A smart baker calculates 200°C once before starting, not before each cookie.

- **Join reordering** — pick the most efficient join strategy based on table sizes and statistics
  ```python
  # You write (naively — large joined with large first):
  orders.join(order_items, "order_id").join(countries, "country_code")
  # Catalyst sees countries is tiny (50 rows) → broadcasts it first, then joins the two large tables
  ```

  | Strategy | When | How |
  |---|---|---|
  | **Broadcast Hash Join** | One table is small | Copy small table to every executor — no shuffle |
  | **Sort Merge Join** | Both tables are large | Sort both on join key, merge — requires shuffle |

  > **Plain English:** You need to match students to their teacher AND their school district. Naive: cross-reference all students with all teachers first (huge), then add district. Smart: look up district first (50 rows, tiny) — narrows the student list before the expensive teacher matching.

Both happen invisibly — you describe *what* you want, Catalyst figures out *how*.

You get all of this for free when using DataFrames; you lose it with RDDs.

> **Plain English:** A smart GPS that doesn't follow your directions literally — it looks at your destination and re-routes around traffic you didn't know about. You say "go left, then right, then left" and it says "actually, there's a highway that gets you there in half the time."

---

## 2. SparkSession & Reading Data

**Q: How do you create a SparkSession?**
```python
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("my_pipeline") \
    .getOrCreate()              # reuses existing session if one exists
```

---

**Q: What are the ways to read data into PySpark?**
```python
df = spark.read.csv("s3://path/file.csv", header=True, inferSchema=True)
df = spark.read.parquet("s3://path/file.parquet")
df = spark.read.json("s3://path/file.json")
df = spark.read.format("delta").load("s3://silver/events/")
```
Prefer explicit schema over `inferSchema=True` in production — inference scans the full file and can be wrong.

---

**Q: How do you define a schema explicitly?**
```python
from pyspark.sql.types import StructType, StructField, StringType, TimestampType, IntegerType

schema = StructType([
    StructField("member_id", StringType(), nullable=False),
    StructField("event_type", StringType(), nullable=False),
    StructField("event_ts", TimestampType(), nullable=False),
    StructField("duration_secs", IntegerType(), nullable=True),
])

df = spark.read.csv("s3://raw/events/", schema=schema, header=True)
```
Explicit schema = faster reads, catches upstream changes, no silent type coercion.

---

## 3. Transformations

**Q: What does `withColumn` do?**
Adds a new column or replaces an existing one. Returns a new DataFrame (DataFrames are immutable).

```python
df = df.withColumn("duration_mins", F.col("duration_secs") / 60)  # new column
df = df.withColumn("event_type", F.lower(F.col("event_type")))    # overwrite existing
```

---

**Q: `map()` vs `flatMap()` on RDDs?**
```python
rdd = spark.sparkContext.parallelize([1, 2, 3])

rdd.map(lambda x: [x, x*2]).collect()
# → [[1, 2], [2, 4], [3, 6]]   one list per element

rdd.flatMap(lambda x: [x, x*2]).collect()
# → [1, 2, 2, 4, 3, 6]         flattened — one element per item in the returned list
```
`flatMap` is useful for tokenizing text or expanding one row into many.

---

**Q: How do you filter rows?**
```python
df.filter(F.col("event_type") == "session_start")       # keep matching rows
df.filter(F.col("duration_secs") > 60)
df.where(F.col("member_id").isNotNull())                 # .where() is an alias
```

---

**Q: How do you run SQL on a DataFrame?**
```python
df.createOrReplaceTempView("events")                    # register as a temp table
result = spark.sql("SELECT member_id, COUNT(*) FROM events GROUP BY member_id")
```
The view only lives for the duration of the SparkSession.

---

## 4. Aggregations & Window Functions

**Q: How do you aggregate data?**
```python
from pyspark.sql import functions as F

df.groupBy("member_id").agg(
    F.count("session_id").alias("session_count"),
    F.sum("duration_secs").alias("total_secs"),
    F.max("event_ts").alias("last_active_ts"),
    F.countDistinct("content_id").alias("unique_content"),
)
```

---

**Q: When do you use window functions instead of groupBy?**
`groupBy` collapses rows into one per group. Window functions compute over a group **without collapsing** — each row keeps its own output.

> **Plain English:** `groupBy` is like grading a class and returning only the class average — individual scores are gone. Window functions return the class average *alongside each student's own score* — every row survives, enriched with the group-level calculation.

```python
from pyspark.sql.window import Window

w = Window.partitionBy("member_id").orderBy("event_ts")

df = df.withColumn("prev_ts", F.lag("event_ts").over(w))        # previous row's value
df = df.withColumn("row_num", F.row_number().over(w))           # rank within member
df = df.withColumn("running_total", F.sum("duration_secs").over(  # cumulative sum
    w.rowsBetween(Window.unboundedPreceding, Window.currentRow)
))
```

Common window functions: `lag`, `lead`, `row_number`, `rank`, `dense_rank`, `sum`, `avg`.

---

## 5. Joins

**Q: How do you join DataFrames?**
```python
df1.join(df2, on="member_id", how="inner")   # inner: only matching rows
df1.join(df2, on="member_id", how="left")    # left: all rows from df1
df1.join(df2, on="member_id", how="outer")   # outer: all rows from both
df1.join(df2, on="member_id", how="anti")    # anti: rows in df1 NOT in df2
```

---

**Q: What is a broadcast join and when do you use it?**
When one DataFrame is small enough to fit in memory, broadcast it to all executors — avoids a shuffle entirely.

> **Plain English:** You need to match 100M customer orders with a list of 50 country codes. Instead of moving all 100M rows around to find their country, you just hand every worker the tiny 50-row country list. Each worker already has everything they need — no network traffic.

```python
from pyspark.sql.functions import broadcast

df_large.join(broadcast(df_small), on="product_id")
```
Rule of thumb: broadcast tables under ~200MB. Configured via `spark.sql.autoBroadcastJoinThreshold`.

---

## 6. Partitioning & Performance

**Q: What is a partition in Spark?**
The basic unit of parallelism — one partition = one task on one executor. More partitions = more parallelism, up to the number of cores available. Too many small partitions = overhead from task scheduling.

Target: ~128–200MB per partition.

> **Plain English:** Imagine sorting 1 million letters. Instead of one person doing all of it, you split the pile into 200 stacks and give each to a different worker. Each worker sorts their own stack independently — that's a partition. Too few stacks = some workers are overwhelmed. Too many tiny stacks = more time managing workers than actually sorting.

---

**Q: `repartition()` vs `coalesce()` — when to use each?**

| | `repartition(n)` | `coalesce(n)` |
|---|---|---|
| **Shuffle** | Yes — full shuffle | No (or minimal) |
| **Use for** | Increasing partitions, changing partition key | Reducing partitions only |
| **When** | Before a join or groupBy on a new key | Before writing to reduce output files |

```python
df.repartition(200, "member_id")   # shuffle by member_id → even distribution
df.coalesce(10)                     # reduce to 10 without full shuffle (write optimization)
```

> **Plain English:** `repartition` = completely reshuffling a deck of cards (expensive, but you get a perfectly even spread). `coalesce` = just combining some piles without reshuffling (cheap, only works for making fewer piles, not more).

---

**Q: How do you handle data skew?**
Skew = one partition has far more data than others → one task runs forever while others finish.

Three different tools for the same problem:

| Fix | Who does it | When |
|---|---|---|
| **Salt the key** | You — manual code change | You know which key is skewed |
| **Broadcast join** | You — wrap small table in `broadcast()` | Small table exists on one side of the join |
| **AQE** | Spark — automatic | Spark 3+, just enable the config |

- **Salt the key**: add a random suffix to the skewed key, join, then aggregate
  > **Plain English:** 10 checkout lanes but everyone goes to lane 1 because of their loyalty card number. Salting = secretly redirect some lane-1 customers to other lanes under a fake name, then reconcile the totals at the end.

- **Broadcast join**: if one side of the join is small, broadcast the whole table to every executor — skew becomes irrelevant because there's no data movement to be uneven
  > **Plain English:** Instead of routing customers to lanes based on their loyalty card number (causing pile-up at lane 1), just hand every cashier a full copy of the loyalty card database. Each cashier looks it up themselves — no routing, no skew possible.

- **AQE (Adaptive Query Execution)**: Spark 3+ watches actual data sizes at runtime and automatically splits oversized partitions mid-job — enable with `spark.sql.adaptive.enabled=true`
  > **Plain English:** A store manager watching the lanes in real time. When lane 1 hits 200 people, she opens a new lane and moves 100 over — automatically, without anyone asking. You didn't have to plan for it upfront.

---

**Q: How do you optimize shuffle operations?**
Shuffle = Spark redistributing data across executors over the network. The most expensive operation in Spark — minimize it.

- **Broadcast joins for small tables** — copy the small table to every executor instead of shuffling the big table. Zero shuffle. *(see broadcast join section above)*

- **Tune `spark.sql.shuffle.partitions`** — after every shuffle (groupBy, join), Spark splits the result into this many partitions. Default 200 is arbitrary.
  - Too high for small data → 200 tasks each processing 5KB, more scheduling overhead than actual work
  - Too low for large data → each partition is too big, executor runs out of memory
  ```python
  spark.conf.set("spark.sql.shuffle.partitions", 50)    # small dataset
  spark.conf.set("spark.sql.shuffle.partitions", 2000)  # large dataset
  ```
  > **Plain English:** A restaurant that always sets exactly 200 tables regardless of reservations. 3 guests? 200 tables — wasteful. 2000 guests? 200 tables — chaos. Match the number to the actual crowd.

- **Repartition on the join key before a join** — Spark needs rows with the same key on the same executor to join them. If not already partitioned that way, Spark shuffles during the join. Pre-repartitioning moves that shuffle upfront so the join itself requires no data movement.
  ```python
  # Without: Spark shuffles both tables during the join (expensive, uncontrolled)
  orders.join(order_items, "order_id")

  # With: shuffle happens once cleanly upfront, join is local
  orders.repartition(200, "order_id")
  order_items.repartition(200, "order_id")
  orders.join(order_items, "order_id")   # matching keys already on same executor
  ```
  > **Plain English:** 10 cashiers each handling random customers. When a customer needs a price check, the cashier has to call across the store to find the right department (shuffle during join). Pre-repartition = before the shift starts, assign each cashier only customers from specific departments — when a price check comes up, the answer is already at that cashier's station.

- **`coalesce()` before writing** — after processing you might have 500 partitions → 500 tiny files on S3. Tiny files are slow to read later (each = one S3 API call, one Spark task). `coalesce()` merges partitions without a full shuffle before writing.
  ```python
  df.coalesce(10).write.format("delta").save("s3://gold/output/")
  # 10 clean files instead of 500 tiny ones
  ```
  > **Plain English:** You took 500 Post-it notes in a meeting. Before filing, you consolidate onto 10 normal pages. Takes a minute now, but anyone who reads them later thanks you — 10 pages vs. 500 scraps.

---

## 7. Caching & Persistence

**Q: When should you cache a DataFrame?**
Cache when the same DataFrame is used more than once (e.g., used in two branches of a pipeline, or used in both a count check and a write).

```python
df.cache()                                          # store in memory (MEMORY_AND_DISK by default)

from pyspark import StorageLevel
df.persist(StorageLevel.MEMORY_AND_DISK)            # spill to disk if memory fills
df.persist(StorageLevel.DISK_ONLY)                  # disk only — saves memory for large DFs

df.unpersist()                                      # free memory when done
```
Don't cache everything — caching has overhead and fills executor memory.

> **Plain English:** Like keeping a cooked meal in the fridge instead of cooking it from scratch every time you're hungry. Great if you'll eat it twice. Wasteful if you only eat it once and it takes up fridge space you need for something else.

---

## 8. UDFs

**Q: What is a UDF and what's the downside?**
User Defined Function — wrap a Python function so it can be called on a DataFrame column.

```python
from pyspark.sql.functions import udf
from pyspark.sql.types import IntegerType

@udf(returnType=IntegerType())
def cap_duration(secs):
    return min(secs, 3600) if secs else None

df = df.withColumn("duration_capped", cap_duration(F.col("duration_secs")))
```
**Downside**: UDFs break out of the JVM — Spark serializes each row to Python, calls the function, serializes back. Very slow for large datasets.

**Prefer built-in `F.*` functions** — they run in the JVM with full Catalyst optimization. Only reach for UDFs when no built-in equivalent exists.

> **Plain English:** Spark's workers speak Java natively. Built-in functions are instructions given in Java — fast. A UDF is a Python instruction — the worker has to hire a translator for every single row, translate the data to Python, wait for the answer, translate back. Multiply that by 100 million rows.

---

## 9. Missing Data

**Q: How do you handle nulls?**
```python
df.dropna(how="any")                                # drop rows with any null
df.dropna(how="all")                                # drop rows where ALL columns are null
df.dropna(subset=["member_id", "event_type"])       # drop rows where these specific cols are null

df.fillna({"duration_secs": 0, "content_id": "unknown"})  # fill per column

df.filter(F.col("member_id").isNotNull())           # filter nulls manually
df.withColumn("duration_secs",
    F.coalesce(F.col("duration_secs"), F.lit(0)))   # coalesce: use first non-null value
```

---

## 10. Broadcast Variables & Accumulators

**Q: What are broadcast variables?**
Read-only variables cached and shipped to every executor once — avoids sending the same data repeatedly with each task.

> **Plain English:** Instead of every worker walking to the central office to look up the same reference book on every task, you make photocopies and put one on every worker's desk at the start of the day. Same info, zero commute.

```python
lookup = {"US": "North America", "JP": "Asia"}
bc = spark.sparkContext.broadcast(lookup)

map_region = udf(lambda code: bc.value.get(code, "Unknown"))
df = df.withColumn("region", map_region(F.col("country_code")))
```

---

**Q: What are accumulators?**
Shared counters/sums that executors can add to but only the driver can read. Useful for counting bad records in a pipeline.

> **Plain English:** A shared tally board on the wall — every worker can add to it ("I found 1 bad record"), but only the manager (driver) can read the final total. Workers can write, but can't see each other's counts mid-job.

```python
bad_record_count = spark.sparkContext.accumulator(0)

def process(row):
    if row["event_type"] is None:
        bad_record_count.add(1)

df.foreach(process)
print(f"Bad records: {bad_record_count.value}")
```

---

## 11. ACID & Delta Lake

**Q: What is ACID?**
The four properties that guarantee a database transaction is reliable.

**A — Atomicity** ("all or nothing")
A transaction either fully completes or fully fails. No partial writes.
> **Plain English:** Bank transfer — you send $100 from Account A to Account B. Either both "debit A" and "credit B" happen, or neither does. You never end up where A lost $100 but B never received it.

**C — Consistency**
A transaction brings the database from one valid state to another. It can never violate defined rules (constraints, schemas).
> **Plain English:** A library rule says every book must have an author. Consistency means no transaction can ever leave a book without an author — the rule is enforced before and after every change.

**I — Isolation**
Concurrent transactions don't interfere with each other. Each sees a consistent snapshot as if it ran alone.
> **Plain English:** Two cashiers scanning the last item in inventory at the same time. Isolation ensures one gets it and the other gets "out of stock" — not both selling the same item simultaneously.

**D — Durability**
Once a transaction is committed, it stays committed — even if the system crashes immediately after.
> **Plain English:** You submitted an online order and got a confirmation. Even if the server crashes one second later, your order is still there when it comes back up — written to permanent storage, not just held in memory.

---

**How Delta Lake implements ACID:**

| Property | How Delta Lake implements it |
|---|---|
| **Atomicity** | Writes committed as a single entry in `_delta_log/` — either the log entry exists or it doesn't |
| **Consistency** | Schema enforcement rejects writes that violate the table's schema |
| **Isolation** | Readers see a consistent snapshot; a writer mid-commit doesn't affect concurrent readers |
| **Durability** | Once the log entry is written to S3, it's permanent — crash after commit = data survives |

Plain S3 without Delta has none of these — a crashed write leaves partial files with no way to know what's valid.

---

## 12. Production & Advanced

**Q: How do you ensure fault tolerance?**
- Spark automatically retries failed tasks (configurable via `spark.task.maxFailures`)
- Use **checkpointing** to save intermediate RDDs to disk — if a stage fails, replay from checkpoint rather than from the beginning
- Delta Lake: ACID transactions mean a failed write doesn't leave partial data

> **Plain English:** Like saving your progress in a video game. If a worker crashes mid-job, Spark doesn't restart the whole game from level 1 — it reloads from the last save point (checkpoint). Delta Lake is the equivalent of a save file that can't be half-written — either it saved or it didn't.

---

**Q: Static vs. dynamic allocation?**

| | Static | Dynamic |
|---|---|---|
| **Executors** | Fixed for the job duration | Scales up/down based on workload |
| **Cost** | Wastes resources when idle | Efficient — releases executors when not needed |
| **Use when** | Latency-sensitive, predictable workload | Variable workloads, cost optimization |

Enable dynamic allocation: `spark.dynamicAllocation.enabled=true`

> **Plain English:** Static = hiring 20 staff for a restaurant every day, even Mondays when only 3 tables are booked. Dynamic = hiring based on the reservation list — 5 on Monday, 20 on Saturday. Same food gets served, much lower payroll.

---

**Q: How do you implement idempotent daily writes?**
```python
# replaceWhere: overwrite only today's partition — safe to rerun without duplicating
df.write.format("delta") \
    .mode("overwrite") \
    .option("replaceWhere", f"event_date = '{run_date}'") \
    .save("s3://gold/mart_member_engagement_daily/")
```
Without `replaceWhere`, `mode("overwrite")` would wipe the entire table.

---

**Q: How do you convert between Pandas and PySpark?**
```python
# Pandas → PySpark (distribute to cluster)
import pandas as pd
pdf = pd.DataFrame({"id": [1, 2, 3]})
df = spark.createDataFrame(pdf)

# PySpark → Pandas (collect to driver — only for small results)
pdf = df.toPandas()
```
Never call `.toPandas()` on a large DataFrame — it pulls all data to the driver and will OOM.

---

## 12. Quick-Reference Heuristics

| Problem | Solution |
|---|---|
| Job is slow | Check shuffle partitions, skew, missing broadcast joins, missing cache |
| Out of memory on driver | You called `.toPandas()` or `.collect()` on too much data |
| Out of memory on executor | Partition count too low; increase `spark.sql.shuffle.partitions` |
| Small files on S3 | `coalesce()` before write; run `OPTIMIZE` on Delta tables |
| Skewed join | Broadcast the small side; or salt the key |
| Repeated computation | Cache the DataFrame after the expensive step |
| UDF is slow | Replace with built-in `F.*` functions |
| Schema changed upstream | `mergeSchema=false` at Bronze catches it; `mergeSchema=true` at Silver absorbs additive changes |

---

*Sources: DataCamp PySpark Interview Questions (2026) · Analytics Vidhya PySpark Interview Q&A (2026)*
