# dbt and dbt Tests

**Related:** [[topics]] · [[semantic-layer]] · [[applications]]

---

## What dbt Is

dbt (data build tool) handles the **T in ELT** — it transforms data that's already in your warehouse using plain SQL.

```
Raw data (already in warehouse)
        ↓
   dbt models (SELECT statements)
        ↓
   Transformed tables/views
        ↓
   BI tools / semantic layer / AI agents
```

You write `SELECT` statements. dbt compiles them into `CREATE TABLE AS` or `CREATE VIEW AS` and runs them in the right order.

---

## dbt Models

Each model is a `.sql` file containing a single `SELECT`:

```sql
-- models/mart/fct_orders.sql
select
    order_id,
    customer_id,
    sum(amount) as revenue
from {{ ref('stg_orders') }}   -- ref() = dbt resolves the dependency
group by 1, 2
```

`ref()` is how dbt knows the DAG — it builds models in dependency order automatically.

### Materializations

| Type | What it creates | When to use |
|---|---|---|
| `view` | A SQL view | Lightweight, always fresh |
| `table` | A full table rebuild | Query performance matters |
| `incremental` | Appends/merges new rows only | Large tables, expensive to rebuild |
| `ephemeral` | CTE, never persisted | Intermediate logic, not queried directly |

---

## `source()` vs `ref()`

Simple rule:
- **`source()`** — for raw tables you **don't own** (landed by ingestion, outside dbt)
- **`ref()`** — for models you **do own** (built by dbt itself)

```sql
-- first touch of raw data → source()
from {{ source('raw', 'orders') }}

-- referencing another dbt model → ref()
from {{ ref('stg_orders') }}
```

**Why the distinction matters:**

`source()` tells dbt: *"this table comes from outside — track its freshness, document it separately, don't try to build it."*

`ref()` tells dbt: *"this table was built by dbt — wire it into the DAG so you know the dependency order."*

If you used `ref()` on a raw table, dbt would look for a model file called `orders.sql` and fail. If you used `source()` on a dbt model, dbt wouldn't know to build it first — the DAG dependency would be missing.

**The boundary in the pipeline:**

```
raw.orders (source)           ← source() — dbt reads but doesn't own
      ↓
stg_orders.sql (dbt model)    ← source() used inside this file
      ↓
int_orders_enriched.sql       ← ref('stg_orders') — dbt owns both
      ↓
fct_orders.sql                ← ref('int_orders_enriched')
```

`source()` is only ever used in staging models — the first layer that touches raw data. Everything downstream uses `ref()`.

---

## dbt Tests

Tests are assertions that run after models build. They return 0 rows if passing, >0 rows if failing.

### Generic Tests (built-in)

Defined in `schema.yml` alongside the model:

```yaml
models:
  - name: fct_orders
    description: One record per order. Grain is order_id.
    columns:
      - name: order_id
        description: Primary key.
        data_tests:
          - unique
          - not_null
      - name: status
        description: Current order status.
        data_tests:
          - accepted_values:
              values: ['placed', 'shipped', 'delivered', 'cancelled']
      - name: customer_id
        description: Foreign key to dim_customers.
        data_tests:
          - relationships:
              to: ref('dim_customers')
              field: customer_id
```

Four built-in generics: `unique`, `not_null`, `accepted_values`, `relationships`.

### Singular Tests (custom SQL)

Any `.sql` file in `tests/` folder — returns rows on failure:

```sql
-- tests/assert_revenue_not_negative.sql
select order_id
from {{ ref('fct_orders') }}
where revenue < 0
```

If this returns any rows, the test fails.

### Test Severity

```yaml
- name: order_id
  data_tests:
    - unique:
        severity: warn    # won't fail the run, just warns
    - not_null:
        severity: error   # default — fails the run
```

### Extended Tests (packages)

`dbt_expectations` adds ~40 more tests modeled after Great Expectations:
- `expect_column_values_to_be_between`
- `expect_table_row_count_to_be_between`
- `expect_column_mean_to_be_between`

---

## End-to-End Example

→ [[dbt-example-ecommerce]] — E-commerce orders pipeline (staging → intermediate → mart, full YAML, DAG)

