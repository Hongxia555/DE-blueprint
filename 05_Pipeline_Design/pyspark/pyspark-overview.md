# PySpark Overview

**Related:** [[pyspark-example-pipeline]] · [[dbt-and-testing]] · [[DE_concepts]] · [[industry-ai-foundations]]

---

## What PySpark Is

PySpark is the Python API for Apache Spark — a distributed computing framework that processes large datasets across a cluster of machines in parallel.

```
Raw data (files on S3, Kafka stream, database)
        ↓
   PySpark (read, transform, validate)
        ↓
   Delta Lake tables (bronze → silver → gold)
        ↓
   dbt (SQL transformations, metrics, semantic layer)
        ↓
   BI tools / AI agents
```

PySpark and dbt are complementary, not alternatives:

| | PySpark | dbt |
|---|---|---|
| **Language** | Python | SQL |
| **Where it runs** | Spark cluster (distributed) | Inside the warehouse (Snowflake, Redshift, Databricks) |
| **What it handles** | Ingestion, complex transformations, ML feature prep, streaming | SQL transformations, metrics, documentation, lineage |
| **When to use** | Data too large for SQL, non-SQL sources, ML pipelines | Business logic, reporting layers, semantic layer |

---

## Key Concepts

### Transformations vs Actions

PySpark is **lazy** — calling `.withColumn()`, `.filter()`, or `.groupBy()` does not run anything. Spark builds a logical plan. Execution only happens when you call an **action**.

```python
# These are transformations — nothing runs yet
df = spark.read.parquet("s3://bronze/events/")
df = df.filter(F.col("event_date") == "2026-06-01")
df = df.withColumn("duration_mins", F.col("duration_secs") / 60)

# This is an action — Spark now executes the full plan
df.write.format("delta").save("s3://silver/events/")
```

Common actions: `.write()`, `.count()`, `.show()`, `.collect()`, `.first()`

This is the distributed equivalent of a Python generator — defer work until you actually need the result, then execute optimally.

---

### DataFrames

The primary data structure. Think of it as a distributed table — rows and typed columns, spread across many machines.

```python
from pyspark.sql import SparkSession
from pyspark.sql import functions as F

spark = SparkSession.builder.appName("my_pipeline").getOrCreate()

df = spark.read.parquet("s3://bronze/member_events/")
df.printSchema()   # inspect column names and types
df.show(5)         # preview first 5 rows (triggers execution)
```

---

### Partitions

Spark splits a DataFrame into **partitions** — chunks of data processed in parallel across executors. More partitions = more parallelism, up to a point.

```python
df.rdd.getNumPartitions()          # check current partition count
df.repartition(200, "member_id")   # redistribute by member_id — 200 partitions
df.coalesce(10)                    # reduce partitions without full shuffle (write optimization)
```

Rule of thumb: target ~128–200MB per partition.

---

### Delta Lake

The storage format used with Databricks. Adds ACID transactions, time travel, and schema enforcement on top of Parquet files.

```python
# Write as Delta
df.write.format("delta").mode("append").save("s3://silver/member_sessions/")

# Read as Delta
df = spark.read.format("delta").load("s3://silver/member_sessions/")

# Time travel — read a previous version
df = spark.read.format("delta").option("versionAsOf", 10).load("s3://silver/member_sessions/")

# Restore after a bad pipeline run
spark.sql("RESTORE TABLE delta.`s3://silver/member_sessions/` TO VERSION AS OF 10")
```

**What's physically on S3:**

```
s3://silver/member_sessions/
├── _delta_log/
│   ├── 00000000000000000000.json   ← commit 0: initial write (add files)
│   ├── 00000000000000000001.json   ← commit 1: next operation
│   └── 00000000000000000010.checkpoint.parquet  ← collapsed snapshot every 10 commits
├── part-00000-abc123.snappy.parquet   ← actual data (plain Parquet)
├── part-00001-def456.snappy.parquet
└── part-00002-ghi789.snappy.parquet
```

The data files are plain Parquet. The `_delta_log/` is what makes it Delta — each JSON entry records which files were added or removed. Delta knows what's "live" by reading the log, not by scanning the folder.

**Plain S3 vs Delta Lake:**

| | Plain S3 (Parquet) | Delta Lake on S3 |
|---|---|---|
| **ACID transactions** | No — concurrent writers can corrupt | Yes — atomic commits via log |
| **Time travel** | No | Yes — `VERSION AS OF N` |
| **Schema enforcement** | No — bad schema silently lands | Yes — rejects mismatches |
| **Deletes / Updates** | Rewrite the whole partition | Log marks old files removed, adds new ones |
| **Read consistency** | Dirty reads possible mid-write | Snapshot isolation |
| **Overhead** | Zero | Small: `_delta_log/` JSON files |

**Delta Lake vs Data Governance vs Data Observability** — these are distinct layers:

```
Data Governance        ← who can touch it, lineage, compliance
        ↑                 (Unity Catalog, AWS Lake Formation, Apache Ranger)
Data Observability     ← is the data healthy? anomaly detection
        ↑                 (Great Expectations, Monte Carlo, dbt tests)
Delta Lake             ← is the write/read operation safe and consistent?
        ↑                 (ACID, time travel, schema enforcement)
S3 / object storage    ← raw bytes
```

- **Delta Lake** = storage reliability ("can I trust what's in this table right now?")
- **Observability** = data health monitoring ("did row count drop 50% overnight?")
- **Governance** = access + compliance ("who is allowed to query this? what PII can they see?")

Delta is a *prerequisite* for the layers above — you can't reliably monitor or govern data that can be corrupted mid-write. But it doesn't do those jobs itself.

---

### Window Functions

Apply a function across a group of rows ordered by a column — without collapsing them into one row (unlike `groupBy`).

```python
from pyspark.sql.window import Window

w = Window.partitionBy("member_id").orderBy("event_ts")

df = df.withColumn("prev_ts", F.lag("event_ts").over(w))       # previous row
df = df.withColumn("row_num", F.row_number().over(w))          # rank within member
df = df.withColumn("running_total", F.sum("amount").over(w))   # cumulative sum
```

---

## How PySpark Differs from dbt

```
dbt:    you write SELECT statements → dbt runs them in the warehouse
PySpark: you write Python → Spark executes distributed jobs across a cluster
```

- dbt can't read raw files from S3 or Kafka — PySpark can
- dbt can't run ML training or feature engineering — PySpark can
- dbt is much simpler for pure SQL transformations and metrics — use it there
- PySpark is necessary when data volume or complexity exceeds what SQL can handle cleanly

In a Medallion architecture: **PySpark owns Bronze → Silver** (ingestion, cleaning, complex transformations). **dbt owns Silver → Gold** (business logic, metrics, semantic layer).

---

*See also: [[pyspark-example-pipeline]] for a full end-to-end pipeline · [[dbt-and-testing]] for the dbt side*
