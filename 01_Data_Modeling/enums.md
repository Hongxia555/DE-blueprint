# Enums in Data Modeling
**Source:** DataExpert.io Boot Camp — Video 7 (Lecture)
**Topic:** Enum types, cardinality constraints, data quality enforcement

---

## What is an Enum?

An **enum** (enumeration) is a column type where the allowed values are defined upfront as a fixed list. The schema — not a test, not a constraint written by hand — enforces that only those values can exist.

```sql
-- Define the enum type
CREATE TYPE scoring_class AS ENUM ('star', 'good', 'average', 'bad');

-- Use it as a column type
CREATE TABLE players (
  player_id    INT,
  season       INT,
  score_class  scoring_class   -- can ONLY be one of the 4 values
);
```

If a pipeline tries to insert a value not in the list:

```sql
INSERT INTO players VALUES (1, 2024, 'legend');
-- ERROR: invalid input value for enum scoring_class: "legend"
```

Caught at write time — not discovered 3 days later when someone queries a dashboard.

---

## Why Use Enums

### 1. Free data quality enforcement
The type itself is the guard. No `dbt` test, no `CHECK` constraint needed. Invalid values break the pipeline immediately and loudly.

### 2. Static, discoverable schema
Analysts can inspect the enum definition to see all valid states — no need to `SELECT DISTINCT score_class` and wonder if the result is complete.

### 3. Storage efficiency
Most engines store enums as integers internally (`'star'` → `0`). Cheaper than `VARCHAR` at scale.

### 4. Encourages explicit domain modeling
Defining an enum forces a conversation: *"What are the valid states for this field?"* That question catches ambiguity before it becomes a data quality incident.

---

## When to Use vs. Avoid

| Use enum | Avoid enum |
|---|---|
| Cardinality < ~50 values | High cardinality (e.g. `country` ~200 values) |
| Values are stable / rarely change | Values change frequently or are discovered from data |
| Invalid values should break the pipeline | You want silent nulls instead of hard failures |
| Domain is well-understood upfront | Values are user-generated / unbounded |

### Good enum candidates
- `scoring_class` → `star`, `good`, `average`, `bad`
- `event_type` → `click`, `impression`, `purchase`, `share`
- `status` → `active`, `inactive`, `suspended`
- `device_type` → `mobile`, `desktop`, `tablet`
- `environment` → `prod`, `staging`, `dev`

### Bad enum candidates
- `country` — ~200 values, new ones appear over time
- `product_category` — grows organically, business-defined
- `error_message` — unbounded free text
- `currency_code` — can expand with new markets

---

## The Philosophy: Loud Failures vs. Silent Nulls

There are two schools of thought for handling bad data:

| Approach | Behavior | Risk |
|---|---|---|
| **Enum (loud failure)** | Pipeline errors on invalid value | Bad data never enters the warehouse |
| **Nullable string** | Invalid value silently becomes NULL | Corrupted metrics, discovered weeks later |

**The enum mindset**: bad data should break things upstream so it gets fixed at the source.

At scale (Meta, Google), a silent NULL in `event_type` means click metrics are wrong for weeks before anyone notices. An enum failure pages the on-call engineer the moment bad data is produced.

---

## Enums in Different Engines

| Engine | Syntax |
|---|---|
| PostgreSQL | `CREATE TYPE status AS ENUM ('active', 'inactive')` |
| MySQL | `status ENUM('active', 'inactive')` as column definition |
| Hive / Spark | No native enum — enforce via `CHECK` or dbt tests on string columns |
| dbt | Use `accepted_values` test as a soft enum equivalent |
| Avro / Protobuf | `enum` type with named symbols — enforced at serialization |

For Hive/Spark where native enums don't exist, the closest equivalent is:

```sql
-- dbt schema.yml
- name: event_type
  tests:
    - accepted_values:
        values: ['click', 'impression', 'purchase']
```

This gives you the discoverability and the test — but it's a soft check at query time, not a hard block at write time.

---

## Enum vs. Lookup Table vs. String Column

| Approach | Type safety | Flexibility | Query simplicity |
|---|---|---|---|
| Enum | Hard (fails on invalid) | Low (schema change to add value) | High (no join needed) |
| Lookup/dim table | Soft (FK constraint optional) | High (insert new row) | Low (requires join) |
| String column | None | Unlimited | High (but `SELECT DISTINCT` to explore) |

**Rule of thumb:**
- < 50 values, stable, business-critical → **enum**
- < 50 values, but may grow or need metadata → **lookup table**
- Unbounded or user-generated → **string column**

---

## Interview Angles

- When would you use an enum vs. a lookup table? → Enum for fixed, stable, low-cardinality domains where invalid values should hard-fail; lookup table when you need metadata on each value or expect growth
- Why is `country` a bad enum? → ~200 values, new countries/territories appear, ISO codes change — schema migrations become a maintenance burden
- What's the risk of not using enums for low-cardinality dimensions? → Silent NULLs or misspellings corrupt metrics without alerting anyone; forces downstream consumers to defensively handle unexpected values
- How do you approximate enums in Spark/Hive? → `accepted_values` dbt test, or a pipeline-level validation step that rejects rows with invalid values before writing to the table
