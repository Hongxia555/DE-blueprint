# Data Engineering — Core Concepts

Reference knowledge for data engineering day-to-day work.

See also: [[database_types]] — DB type selection guide (relational vs columnar, SQL vs NoSQL, decision questions)

---

## Data Warehouse Design

### Star Schema vs Snowflake Schema

| | Star Schema | Snowflake Schema |
|-|-------------|-----------------|
| Dimension tables | Denormalized (flat) | Normalized (split into sub-dimensions) |
| Query speed | Faster (fewer joins) | Slower (more joins) |
| Storage | More (redundant data) | Less (no redundancy) |
| Maintenance | Simpler | More complex |
| Use case | Analytics / OLAP | Space-constrained or frequently updated dimensions |

**Bridge table:** Resolves many-to-many relationships between fact and dimension tables.
Example: a student can take many courses; a course can have many students → bridge table `student_course`.

### OLTP vs OLAP

| | OLTP | OLAP |
|-|------|------|
| Purpose | Transactional (insert/update/delete) | Analytical (read, aggregate) |
| Optimization | Row-oriented storage | Column-oriented storage |
| Schema | Normalized (3NF) | Denormalized (star/snowflake) |
| Query pattern | Single-row lookups | Aggregations across millions of rows |
| Examples | MySQL, PostgreSQL | Redshift, Snowflake, BigQuery, Databricks |

### Normalization vs Denormalization

**Normalization** — eliminate redundancy, enforce consistency
- Pros: less storage, easier updates (change one place)
- Cons: more joins → slower reads

**Denormalization** — flatten/combine tables
- Pros: fewer joins → faster reads
- Cons: data redundancy, update anomalies

**Rule of thumb:** OLTP → normalize; OLAP/DWH → denormalize for query performance.

### Staging Layer — Why?
- Isolate raw source data before transforming
- Allows re-processing if transformation logic changes
- Catch data quality issues before they reach the mart layer
- Audit trail: can always trace back to raw

### SCD (Slowly Changing Dimensions)

| Type | Behavior | Use Case |
|------|----------|----------|
| SCD Type 1 | Overwrite old value | When history doesn't matter |
| SCD Type 2 | Add new row with date ranges (start/end date + is_current flag) | Full history required |
| SCD Type 3 | Add "previous" column | Only need current + one prior value |

---

## ETL vs ELT

| | ETL | ELT |
|-|-----|-----|
| Transform step | Before loading (in pipeline) | After loading (in warehouse) |
| Compute location | External (Spark, Airflow, etc.) | In the warehouse (SQL) |
| Flexibility | Transform logic separate from storage | Leverage warehouse compute |
| Modern trend | Older pattern | ELT preferred with cloud DWH (Snowflake, BigQuery, Redshift) |

---

## Data Lake vs Data Warehouse

| | Data Lake | Data Warehouse |
|-|-----------|---------------|
| Schema | Schema-on-read (flexible) | Schema-on-write (enforced) |
| Data types | Raw, unstructured, semi-structured | Structured, cleaned |
| Storage cost | Cheap (S3, GCS) | More expensive |
| Query speed | Slower (no indexing) | Faster |
| Use case | ML, exploration, archive | BI, reporting, dashboards |

**Lakehouse** (Delta Lake, Iceberg): combines Lake storage cost with Warehouse query performance + ACID transactions.

---

## Data Pipeline Design

### Batch vs Streaming

| | Batch | Streaming |
|-|-------|-----------|
| Latency | Minutes to hours | Sub-second to seconds |
| Complexity | Lower | Higher |
| Use case | Daily reports, historical loads | Real-time dashboards, fraud detection, alerting |
| Tools | Airflow + Spark, dbt | Kafka, Kinesis, Flink, Spark Streaming |

### Real-Time Pipeline Pattern
```
Source → Kafka/Kinesis → Stream Processor (Flink/Lambda)
       → Aggregation (tumbling window) → Serving layer (DynamoDB/Redshift)
       → Dashboard
```

**Key concepts:**
- **Tumbling window**: fixed-size, non-overlapping (e.g., every 5 min)
- **Sliding window**: overlapping windows (e.g., last 5 min, updated every 1 min)
- **Watermark**: how long to wait for late-arriving events before closing a window
- **Idempotency**: processing the same event twice should produce the same result

### ETL Pipeline Best Practices
1. Staging → Transformation → Mart (medallion: Bronze → Silver → Gold)
2. Idempotent runs — rerunning should not duplicate data
3. Data quality checks at each layer
4. Logging + alerting on failure
5. Backfill capability — can reprocess historical data

---

## Query Optimization

### Index Types

| Type | Description |
|------|-------------|
| Clustered index | Determines physical row order; one per table; best for range scans on the index key |
| Non-clustered index | Separate structure pointing to rows; multiple allowed; good for lookups on specific columns |
| Column store index | Stores data column-by-column; best for analytical aggregations (scan many rows, few columns) |
| Covering index | Includes all columns needed by a query — avoids going back to base table |

### Partitioning
- Split large tables by a key (usually date or region)
- Queries with partition filter only scan relevant partitions → major speedup
- **Partition skew**: uneven sizes (e.g., 80% of rows in one partition) → degrade performance; choose partition key carefully

### General Optimization Checklist
1. Check execution plan — identify full scans, missing indexes
2. Filter early — push WHERE conditions before JOINs
3. Avoid `SELECT *` — only pull needed columns
4. Use appropriate JOIN type — avoid Cartesian products
5. Keep statistics up to date (optimizer relies on them)
6. Materialize expensive CTEs if referenced multiple times
7. Use partitioned tables for large datasets
8. Column store for read-heavy analytical tables

---

## Distributed Systems Concepts

### CAP Theorem
In a distributed system, you can only guarantee 2 of 3:
- **C**onsistency — every read gets the most recent write
- **A**vailability — every request gets a response
- **P**artition tolerance — system works despite network partition

Most distributed DBs choose CP (strong consistency) or AP (high availability).

### ACID (Relational DB Transactions)
- **A**tomicity — all or nothing
- **C**onsistency — data stays valid per rules
- **I**solation — concurrent transactions don't interfere
- **D**urability — committed data persists

### Data Sharding
- Horizontal partitioning across multiple databases/nodes
- Shard key choice is critical — avoid hot shards
- Complicates cross-shard queries and transactions

---

## Data Quality

**How to check data quality:**
1. **Completeness** — are required fields populated? (`COUNT(*) vs COUNT(col)`)
2. **Uniqueness** — are PKs actually unique? (`COUNT(*) vs COUNT(DISTINCT pk)`)
3. **Validity** — do values fall within expected ranges/formats?
4. **Timeliness** — is data arriving on schedule? (lag monitoring)
5. **Consistency** — do values match across tables/sources?
6. **Referential integrity** — do FK values exist in the referenced table?

**Tools:** dbt tests (not_null, unique, accepted_values, relationships), Great Expectations

---

## AWS Data Services (Redshift-specific)

### Redshift-Specific Functions
- `DATEDIFF(part, start, end)` — date difference
- `DATEADD(part, n, date)` — add to date
- `DATE_TRUNC('month', col)` — truncate to period
- `GETDATE()` — current timestamp
- `NVL(col, default)` — same as COALESCE for two args
- `LISTAGG(col, ',') WITHIN GROUP (ORDER BY col)` — string aggregation

### Redshift Architecture
- Columnar storage (good for analytics)
- Massively parallel processing (MPP)
- Distribution styles: `EVEN`, `KEY`, `ALL`
  - `KEY`: distribute by a join key to co-locate joined rows
  - `ALL`: replicate small dimension tables to all nodes
  - `EVEN`: round-robin for tables without a natural join key
- Sort keys: similar to indexes — ORDER BY and range filter optimization

---

## Python for Data Engineering

### Key Libraries
- `pandas` — in-memory DataFrame manipulation, CSV/Excel I/O
- `boto3` — AWS SDK (S3, Glue, Lambda, etc.)
- `psycopg2` / `sqlalchemy` — PostgreSQL / Redshift connections
- `pyspark` — distributed data processing

### Common Patterns
```python
# Read CSV → DataFrame → ingest to DB
import pandas as pd
from sqlalchemy import create_engine

df = pd.read_csv('data.csv')
engine = create_engine('postgresql://user:pw@host/db')
df.to_sql('table_name', engine, if_exists='append', index=False)

# Max depth of nested JSON
def max_depth(d):
    if not isinstance(d, dict) or not d:
        return 0
    return 1 + max(max_depth(v) for v in d.values())
```

### String Manipulation
```python
# First non-repeating character
from collections import Counter
def first_unique(s):
    count = Counter(s)
    for i, c in enumerate(s):
        if count[c] == 1:
            return i + 1
    return -1

# Anagram check
def is_anagram(s1, s2):
    return sorted(s1) == sorted(s2)

# Reverse words in sentence
def reverse_words(s):
    return ' '.join(s.split()[::-1])
```
