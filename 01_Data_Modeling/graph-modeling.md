# Graph Modeling & Additive Dimensions
**Source:** DataExpert.io Boot Camp — Videos 7 (Lecture) & 8 (Lab: NBA Player Network)
**Topic:** Graph databases, vertices/edges schema, additive vs. non-additive dimensions

---

## Key Concepts

### Graph Data Model
- **Vertices** — entities (users, pages, products)
- **Edges** — relationships between entities (follows, clicks, purchases)
- Used at Meta for social graph, friend recommendations, fraud detection

### Vertex/Edge Schema Pattern
```sql
-- Generic vertex table
CREATE TABLE vertices (
  identifier STRING,       -- entity ID
  type STRING,             -- e.g. 'user', 'page', 'video'
  properties MAP<STRING, STRING>  -- flexible attributes
);

-- Generic edge table
CREATE TABLE edges (
  subject_identifier STRING,
  subject_type STRING,
  object_identifier STRING,
  object_type STRING,
  edge_type STRING,        -- e.g. 'FOLLOWS', 'WATCHED', 'PURCHASED'
  properties MAP<STRING, STRING>
);
```

### Additive vs. Non-Additive Dimensions
| Type | Definition | Example |
|------|-----------|---------|
| Additive | Can SUM across all dimensions | Revenue, clicks |
| Semi-additive | Can SUM across some dims, not time | Account balance (snapshot) |
| Non-additive | Cannot meaningfully SUM | Ratios, percentages, distinct counts |

- Non-additive metrics require pre-aggregation or approximate algorithms (HyperLogLog for distinct counts)

---

## When to Pick Graph Modeling

The core question: **"Is the relationship itself what I care about — not just the entities?"**

If yes → graph model. If you care about aggregating attributes on entities → star schema.

### Question Types That Signal Graph Modeling

1. **"Who knows whom / how far apart are they?"**
   - *"Is this user 2 hops away from a known fraudster?"*
   - *"What's the shortest connection between User A and User B?"*
   - Star schema can't answer this — it has no concept of traversal.

2. **"What clusters/communities exist?"**
   - *"Which groups of accounts are behaving similarly?"* (fraud rings)
   - *"Which pages does this user's social circle engage with?"*

3. **"What influenced what?"** (lineage / causality chains)
   - *"Which upstream table failures caused this downstream dashboard to break?"* (data lineage)
   - *"How did this content go viral — who shared it to whom?"*

4. **"What are users similar to each other based on shared connections?"**
   - *"Recommend friends because User A and User C both follow User B"* (collaborative filtering via graph)
   - Instagram Explore's retrieval stage does this — shared-follow graph to find candidate content.

5. **"What paths exist?"** (recommendation / traversal)
   - *"Users who bought X also tend to buy Y — find the purchase chain"*

### Quick Decision Rule

| Situation | Model |
|---|---|
| "How many users did X per day?" | Star schema (fact table) |
| "Which users are connected to fraudsters within 3 hops?" | Graph |
| "Total revenue by region?" | Star schema |
| "Did this account cluster form recently?" | Graph |
| "Who influenced whom in this viral post?" | Graph |

---

## Meta Case Study: Graph Modeling in Practice

Meta's entire product is a graph. Every meaningful question they ask is a relationship question.

### The Core Graph Structure

```
Users ──FRIENDS_WITH──► Users
Users ──LIKES──────────► Posts
Users ──MEMBER_OF──────► Groups
Posts ──TAGGED_IN──────► Users
Users ──FOLLOWS────────► Pages
```

Vertices: `users`, `posts`, `groups`, `pages`. Edges: the actions between them.

### Example Schema (Generic Vertex/Edge)

```sql
-- Vertices
INSERT INTO vertices VALUES
  ('user_123', 'USER', MAP('name', 'Alice', 'city', 'NYC')),
  ('user_456', 'USER', MAP('name', 'Bob',   'city', 'LA')),
  ('page_789', 'PAGE', MAP('name', 'NBA',   'category', 'Sports'));

-- Edges
INSERT INTO edges VALUES
  ('user_123', 'USER', 'user_456', 'USER', 'FRIENDS_WITH', MAP('since', '2022-01-01')),
  ('user_123', 'USER', 'page_789', 'PAGE', 'FOLLOWS',      MAP('notif', 'true')),
  ('user_456', 'USER', 'page_789', 'PAGE', 'FOLLOWS',      MAP('notif', 'false'));
```

### Use Case 1: "People You May Know" (PYMK)

**Question:** *Which users should we suggest as friends to User A?*

Logic: find 2nd-degree connections (friends of friends) and rank by mutual friend count.

```
User A → friends → [User B, User C, User D]
User B → friends → [User E, User F]
User C → friends → [User E, User G]
User D → friends → [User E, User I]

Mutual count: User E = 3 → top recommendation
```

```sql
SELECT
  e2.object_identifier AS suggested_friend,
  COUNT(*) AS mutual_friends
FROM edges e1
JOIN edges e2
  ON e1.object_identifier = e2.subject_identifier
  AND e2.edge_type = 'FRIENDS_WITH'
WHERE e1.subject_identifier = 'user_123'
  AND e1.edge_type = 'FRIENDS_WITH'
  AND e2.object_identifier != 'user_123'
  AND e2.object_identifier NOT IN (
    SELECT object_identifier FROM edges
    WHERE subject_identifier = 'user_123'
      AND edge_type = 'FRIENDS_WITH'
  )
GROUP BY suggested_friend
ORDER BY mutual_friends DESC;
```

This is a 2-hop traversal — impossible to express cleanly in a star schema without expensive self-joins.

### Use Case 2: Fake Account Detection (Integrity)

**Question:** *Is this new account part of a coordinated fake network?*

Graph red flags that a star schema can't surface:
- 500 accounts all created the same week
- All LIKE the same 20 Pages
- All FRIENDS_WITH each other but nobody outside the cluster
- Zero mutual friends with legitimate users

A star schema shows "500 accounts liked Page X" — looks normal. Only the **graph topology** (cluster isolated from real social graph) reveals the fraud ring.

### Use Case 3: Feed Content Recommendations

**Question:** *What should we show User A in their Feed?*

2-hop logic: *"What are User A's friends engaging with?"*

```
User A → FRIENDS_WITH → [User B, User C]
User B → LIKED → [Post 101, Post 202]
User C → LIKED → [Post 202, Post 303]

Post 202 has 2 signals → boost for User A
```

### Why Not Star Schema for These?

| Question | Star Schema Problem |
|---|---|
| PYMK | Multi-level self-joins explode at billions-of-edges scale |
| Fraud ring detection | Can detect time clusters, but misses topological isolation |
| Viral content path | Recursive CTEs possible but slow; no native traversal |

Graph databases (Neo4j, Neptune) or graph processing engines (GraphX, GraphFrames on Spark) use **index-free adjacency** — each node directly points to its neighbors in memory, making hops O(1) instead of O(log n) per join.

---

## Interview Angles
- When would you model data as a graph vs. star schema? → Graph when relationships are the query target; star schema when aggregations on attributes dominate
- How does Meta use graph modeling for integrity? → Detect fake accounts via network clustering
- Why is `distinct_users` non-additive? → Counting distinct users across segments double-counts users in multiple segments

---

## Notes (from transcript)

### Graph data modeling vs dimensional modeling
- Graph = **relationship-focused**, not entity-focused
- When to use: "if you're looking at how things are connected, that is where graph data modeling truly shines"
- Zach at Netflix: built graphs with **70–80 different types of entities** to understand how everything is connected
- Trade-off: schema is very flexible (vertex + properties, edge + properties) → less type safety

### Flexible data types: struct vs array vs map
- `STRUCT` — fixed schema, all keys and types must be defined upfront. Not flexible.
- `ARRAY` — flexible count but must be homogeneous type (array of string, array of struct)
- `MAP<STRING, STRING>` — most flexible, any number of key-value pairs. Good for heterogeneous properties.
- Graph vertex/edge properties are typically stored as `MAP<STRING, STRING>` due to their variability

| | Struct | Array | Map |
|---|---|---|---|
| Keys/fields | Fixed, named upfront | No keys, just positions | Dynamic, any string |
| Length | Fixed (always same fields) | Variable | Variable |
| Type safety | Strong (each field has its own type) | Medium (all elements same type) | Weak (values all same type) |
| Best for | Embedding a related object (address, event) | Time-series per entity (active dates, sessions) | Heterogeneous properties (graph vertices/edges) |
| Query style | `col.field_name` | `ARRAY_CONTAINS`, `UNNEST` | `col['key']` |

### Additive vs non-additive dimensions (Video 7's most important concept)
- **Additive dimension**: sub-totals can be summed to grand total
  - Example: age groups — sum of all age groups = total population ✓
- **Non-additive dimension**: sub-totals **cannot** be summed (due to overlap/double counting)
  - Example: Honda drivers ≠ Civic drivers + Corolla drivers (someone can own both)
  - Example: app users ≠ iPhone users + Android users (some people use both)

### The rule for additivity
- "A dimension is additive over a time window if and only if the grain of data over that window can only ever be one value at a time"
- On a small enough time scale (1 second), most dimensions become additive
- On a day/week/month scale, "tool-type" dimensions (cars, devices) become non-additive

### Why additivity matters for count metrics
- Non-additive dimension → you **cannot use pre-aggregated subtotals**
- Must go back to row-level data and use `COUNT DISTINCT`
- Applies to: count metrics, ratio metrics (miles per driver is also non-additive)
- Additive dimension → `SUM(subtotals)` = grand total, much cheaper queries

### Milky Way — Facebook's framework for non-additive aggregations
- Facebook built internal frameworks to handle non-additive dimensions automatically
- Engineers could request non-additive aggregations without manually writing COUNT DISTINCT logic
- Shows the scale at which this problem matters at big tech

### Enums — when and why
- Use when cardinality < ~50 distinct values
- Benefits: automatic data quality (pipeline fails if invalid value), static field definitions
- Don't use for `country` (~200 values) — too many for an enum
- Good use case: scoring_class (star/good/average/bad), event_type (click/impression/purchase)
