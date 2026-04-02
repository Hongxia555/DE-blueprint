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
