# Database Types — Selection Guide

Reference guide for database type selection. See also [[DE_concepts]].

---

## Relational vs Columnar

| | Relational (Row-store) | Columnar (Column-store) |
|---|---|---|
| **Storage** | Row by row on disk | Column by column on disk |
| **Read pattern** | Fetch full rows | Fetch only queried columns |
| **Write pattern** | Fast single-row inserts | Slower (must update each column file) |
| **Best for** | OLTP — transactional, point lookups | OLAP — analytics, aggregations over wide tables |
| **Compression** | Moderate | Excellent — same-type values compress well |
| **Examples** | MySQL, PostgreSQL, Oracle | BigQuery, Redshift, Snowflake, ClickHouse, Parquet |

**The intuition:** A 1B-row events table with 50 columns.
`SELECT country, SUM(revenue) FROM events GROUP BY country`
- Row-store: reads all 50 columns × 1B rows, discards 48
- Column-store: reads only 2 columns — 2/50th of the I/O

**Key points:**
- Parquet/ORC are columnar — why Spark prefers them for analytics
- `SELECT *` is expensive in BigQuery/Redshift for this reason
- Snowflake uses micro-partitions (columnar within partitions); Delta Lake adds row-level ops on top of Parquet

---

## SQL vs NoSQL

| | SQL | NoSQL |
|---|---|---|
| **Schema** | Fixed, predefined | Flexible / schema-on-read |
| **Scaling** | Vertical (scale up) | Horizontal (scale out) |
| **Consistency** | ACID | Usually BASE (eventually consistent) |
| **Joins** | Native | Avoided — denormalize instead |
| **Query language** | SQL (standardized) | API/driver specific |
| **Data model** | Tables + rows | Document, KV, wide-column, graph |

### NoSQL Subtypes

| Type | Model | Sweet spot | Examples |
|---|---|---|---|
| **Key-Value** | `key → blob` | Caching, sessions, leaderboards | Redis, DynamoDB |
| **Document** | JSON/BSON docs | User profiles, catalogs, CMS | MongoDB, Firestore |
| **Wide-Column** | Rows with dynamic columns | Time-series, write-heavy, IoT | Cassandra, HBase, Bigtable |
| **Graph** | Nodes + edges | Social graphs, fraud detection, recommendations | Neo4j, Neptune |
| **Search** | Inverted index | Full-text search, log analytics | Elasticsearch, OpenSearch |
| **Time-Series** | Timestamped metrics | Monitoring, IoT, financial ticks | InfluxDB, TimescaleDB |

### CAP Theorem

NoSQL systems sacrifice **Consistency** or **Availability** to gain **Partition tolerance** at scale.
- CP systems: HBase, Zookeeper
- AP systems: Cassandra, DynamoDB

**Key points:**
- "Why Cassandra for writes?" → AP, LSM-tree writes are sequential = fast
- "Why DynamoDB at scale?" → KV, single-digit ms latency, infinite horizontal scale
- OLTP → SQL; if write throughput explodes → Cassandra/DynamoDB

---

## Decision Flow

```
Structured + ACID + joins?       → SQL (Postgres, MySQL)
Massive writes, known access?    → Wide-column (Cassandra, Bigtable)
Flexible schema, nested docs?    → Document (MongoDB, Firestore)
Sub-ms cache / session?          → KV (Redis, DynamoDB)
Analytics / aggregations at PB?  → Columnar (BigQuery, Snowflake)
Graph traversals?                → Graph (Neo4j, Neptune)
Full-text search?                → Search (Elasticsearch)
Metrics / monitoring?            → Time-series (InfluxDB, Timescale)
```

---

## Questions to Ask When Choosing a Database

### 1. Data Model
- What is the shape of the data — structured, semi-structured, unstructured?
- Are there clear relationships between entities? Do you need joins?
- Is the schema stable or will it evolve frequently?
- Is it hierarchical / graph-like / time-series in nature?

### 2. Access Patterns
- Read-heavy or write-heavy?
- Point lookups (`WHERE id = X`) or range scans (`WHERE ts BETWEEN ...`)?
- Full-table aggregations or single-row fetches?
- Do you need full-text search?
- What does a typical query look like — known key, ad-hoc SQL, graph traversal?

### 3. Scale
- How much data — GB, TB, PB?
- How many reads/writes per second (QPS/TPS)?
- Does traffic spike? How bursty is it?
- Do you need to scale horizontally or is vertical enough?

### 4. Consistency & Correctness
- Do you need ACID transactions? Multi-row? Cross-table?
- Can you tolerate eventual consistency?
- What happens on a write conflict — last-write-wins acceptable?
- Are there strict ordering guarantees required?

### 5. Latency & Throughput
- What is the SLA? Sub-millisecond, single-digit ms, seconds?
- Is this user-facing (latency-sensitive) or batch (throughput-sensitive)?
- Can reads be served from cache, or must they be fresh?

### 6. Availability & Durability
- What is the acceptable downtime (99.9% vs 99.999%)?
- What is RPO (how much data loss is tolerable)?
- What is RTO (how fast must you recover)?
- Do you need multi-region replication?

### 7. Operational & Cost
- Managed service or self-hosted?
- What is the team's existing expertise?
- Storage cost vs compute cost — which dominates?
- Licensing cost (Oracle vs Postgres)?
- How complex is it to operate at scale?

### 8. Ecosystem & Integration
- Does it integrate with your existing stack (Spark, Kafka, dbt)?
- Does it support your language/driver?
- Is there good monitoring / observability tooling?
- Community size and maturity?