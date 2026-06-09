# dbt Example — E-commerce Orders Pipeline

**Related:** [[dbt-and-testing]] · [[topics]]

---

## Scenario

Raw source tables land in the warehouse:
- `raw.orders` — one row per order
- `raw.customers` — customer info
- `raw.order_items` — line items per order

Goal: build a clean `fct_orders` fact table and `dim_customers` dimension for BI consumption.

---

## Project Structure

```
models/
  staging/
    stg_orders.sql
    stg_customers.sql
    stg_order_items.sql
    _sources.yml       ← declares raw sources + freshness checks
    _staging.yml       ← column-level tests + descriptions
  intermediate/
    int_orders_enriched.sql
  mart/
    fct_orders.sql
    dim_customers.sql
    _mart.yml          ← tests on final tables
snapshots/
    snap_customers.sql ← SCD Type 2 history for customers
macros/
    generate_schema_name.sql ← overrides schema resolution across environments
tests/
    assert_revenue_non_negative.sql       ← singular test
    assert_orders_have_valid_customers.sql ← singular test
    assert_items_match_orders.sql          ← singular test
```

---

## Layer 1 — Staging (clean & rename raw sources)

```sql
-- models/staging/stg_orders.sql
-- materialization: view (cheap, always fresh)
select
    order_id,
    customer_id,
    created_at                          as ordered_at,
    status,
    coalesce(amount_usd, 0)             as amount_usd
from {{ source('raw', 'orders') }}      -- source() for raw tables
where order_id is not null
```

```sql
-- models/staging/stg_order_items.sql
select
    item_id,
    order_id,
    product_id,
    quantity,
    unit_price_usd
from {{ source('raw', 'order_items') }}
```

**Rule:** staging does nothing but clean — rename columns, cast types, filter nulls. No joins, no aggregations.

---

## Layer 2 — Intermediate (business logic)

```sql
-- models/intermediate/int_orders_enriched.sql
-- materialization: ephemeral (CTE, not stored)
select
    o.order_id,
    o.customer_id,
    o.ordered_at,
    o.status,
    sum(i.quantity * i.unit_price_usd)  as calculated_revenue
from {{ ref('stg_orders') }} o
left join {{ ref('stg_order_items') }} i
    on o.order_id = i.order_id
group by 1, 2, 3, 4
```

**Rule:** intermediate is where joins and transformations live. Not exposed to BI directly.

---

## Layer 3 — Mart (final tables for consumption)

```sql
-- models/mart/fct_orders.sql
-- materialization: incremental (large table, append new orders daily)
{{
  config(
    materialized='incremental',
    unique_key='order_id',
    incremental_strategy='merge'
  )
}}

select
    order_id,
    customer_id,
    ordered_at,
    status,
    calculated_revenue
from {{ ref('int_orders_enriched') }}

{% if is_incremental() %}
  where ordered_at >= (select max(ordered_at) from {{ this }})
{% endif %}
```

```sql
-- models/mart/dim_customers.sql
-- materialization: table (small, full rebuild is fine)
select
    customer_id,
    first_name,
    last_name,
    email,
    created_at                          as customer_since
from {{ ref('stg_customers') }}
```

---

## Materialization Config

All four materializations can be set in the config block, `schema.yml`, or `dbt_project.yml`.

**Config block (per model):**
```sql
{{ config(materialized='view') }}          -- stg_orders, stg_customers
{{ config(materialized='ephemeral') }}     -- int_orders_enriched
{{ config(materialized='table') }}         -- dim_customers
{{ config(materialized='incremental', unique_key='order_id') }}  -- fct_orders
```

**`dbt_project.yml` (folder-wide defaults):**
```yaml
models:
  ecommerce:
    staging:
      +materialized: view
    intermediate:
      +materialized: ephemeral
    mart:
      +materialized: table      # fct_orders overrides this to incremental via config block
```

**`schema.yml` (per model override):**
```yaml
models:
  - name: fct_orders
    config:
      materialized: incremental
      unique_key: order_id
```

> `incremental` needs `unique_key` as an extra config. `ephemeral` models are never stored in the warehouse — dbt inlines them as CTEs into the consuming model's SQL.

---

## YAML Files

### `_sources.yml` — Source Declarations + Freshness

```yaml
# models/staging/_sources.yml
sources:
  - name: raw
    tables:
      - name: orders
        freshness:
          warn_after: {count: 6, period: hour}
          error_after: {count: 24, period: hour}
        loaded_at_field: created_at
      - name: order_items
      - name: customers
```

`dbt source freshness` checks if raw data is arriving on time — the UPM equivalent in dbt.

### `_staging.yml` — Column-level Tests + Descriptions

```yaml
# models/staging/_staging.yml
models:
  - name: stg_orders
    description: One record per order, cleaned from raw.orders.
    columns:
      - name: order_id
        description: Primary key. Natural key from the source system.
        data_tests:
          - unique
          - not_null
      - name: status
        description: Current order status. Always one of the accepted values below.
        data_tests:
          - accepted_values:
              values: ['placed', 'shipped', 'delivered', 'cancelled']
      - name: amount_usd
        description: Order total in USD. Coalesced to 0 if null in source.
        data_tests:
          - not_null

  - name: stg_order_items
    description: One record per line item per order.
    columns:
      - name: item_id
        description: Primary key.
        data_tests:
          - unique
          - not_null
      - name: order_id
        description: Foreign key to stg_orders.
        data_tests:
          - not_null
      - name: quantity
        description: Number of units ordered. Always positive.
        data_tests:
          - not_null

  - name: stg_customers
    description: One record per customer, cleaned from raw.customers.
    columns:
      - name: customer_id
        description: Primary key.
        data_tests:
          - unique
          - not_null
      - name: email
        description: Customer email. NULL if not provided at signup.
        data_tests:
          - unique
          - not_null
```

### `_mart.yml` — Tests on Final Tables

```yaml
# models/mart/_mart.yml
models:
  - name: fct_orders
    description: One record per order. Grain is order_id. Source of truth for revenue reporting.
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
      - name: calculated_revenue
        description: Sum of quantity × unit_price_usd across all line items. Never negative.
        data_tests:
          - not_null
          - dbt_expectations.expect_column_values_to_be_between:
              min_value: 0
```

---

## Testing

### Generic Tests (schema.yml)

Already defined in the YAML files above — here's a summary of what's covered:

| Model | Column | Tests |
|---|---|---|
| `stg_orders` | `order_id` | `unique`, `not_null` |
| `stg_orders` | `status` | `accepted_values` |
| `stg_order_items` | `item_id` | `unique`, `not_null` |
| `stg_order_items` | `order_id` | `not_null` |
| `stg_customers` | `customer_id` | `unique`, `not_null` |
| `fct_orders` | `order_id` | `unique`, `not_null` |
| `fct_orders` | `customer_id` | `relationships` → `dim_customers` |
| `fct_orders` | `calculated_revenue` | `not_null` |

### Singular Tests (tests/)

Custom SQL — **returns bad rows on failure, zero rows = pass.**

```sql
-- tests/assert_revenue_non_negative.sql
-- Every order must have revenue >= 0
SELECT order_id, calculated_revenue
FROM {{ ref('fct_orders') }}
WHERE calculated_revenue < 0
```

```sql
-- tests/assert_orders_have_valid_customers.sql
-- Every order must link to a customer that exists in dim_customers
SELECT o.order_id
FROM {{ ref('fct_orders') }} o
LEFT JOIN {{ ref('dim_customers') }} c ON o.customer_id = c.customer_id
WHERE c.customer_id IS NULL
```

```sql
-- tests/assert_items_match_orders.sql
-- Every line item must belong to an order that exists in stg_orders
SELECT i.item_id, i.order_id
FROM {{ ref('stg_order_items') }} i
LEFT JOIN {{ ref('stg_orders') }} o ON i.order_id = o.order_id
WHERE o.order_id IS NULL
```

Run all tests:

```bash
dbt test                              # run all tests
dbt test --select stg_orders          # run tests for one model only
dbt test --select test_type:singular  # run singular tests only
dbt test --select test_type:generic   # run generic tests only
```

---

## Snapshot — Tracking Customer History

Customers can change email, address, or name over time. A snapshot tracks that history using SCD Type 2.

```sql
-- snapshots/snap_customers.sql
{% snapshot snap_customers %}
  {{ config(
      target_schema='snapshots',
      unique_key='customer_id',
      strategy='timestamp',
      updated_at='updated_at'
  ) }}
  SELECT * FROM {{ source('raw', 'customers') }}
{% endsnapshot %}
```

`target_schema='snapshots'` — output table lands in the `snapshots` schema in the warehouse (not a folder on disk):

| | `snapshots/` folder | `target_schema='snapshots'` |
|---|---|---|
| What it is | Directory on disk | Schema in the warehouse |
| Purpose | Where you write snapshot `.sql` files | Where dbt writes the output table |

On each run dbt compares source rows against the snapshot table:

- **Row unchanged** — untouched
- **Row changed** — old row gets `dbt_valid_to = now()`; new row inserted with `dbt_valid_to = NULL`
- **New row** — inserted with `dbt_valid_from = now()`, `dbt_valid_to = NULL`

Example — customer changes email:

| customer_id | email | dbt_valid_from | dbt_valid_to |
|---|---|---|---|
| 42 | old@gmail.com | 2024-01-01 | 2024-02-01 |
| 42 | new@gmail.com | 2024-02-01 | NULL |

```sql
-- current state only
SELECT * FROM snapshots.snap_customers WHERE dbt_valid_to IS NULL

-- state at a point in time
SELECT * FROM snapshots.snap_customers
WHERE dbt_valid_from <= '2024-01-15'
  AND (dbt_valid_to > '2024-01-15' OR dbt_valid_to IS NULL)
```

```bash
dbt snapshot    # run all snapshots
```

---

## The DAG dbt Builds

```
raw.orders ──────────────┐
raw.order_items ─────────┼→ stg_orders ───┐
raw.customers ───────────┘ stg_order_items ┼→ int_orders_enriched → fct_orders
                           stg_customers ──┘                      → dim_customers
```

---

## Common Commands

```bash
dbt run                        # build all models
dbt run --select mart          # build mart layer only
dbt test                       # run all tests
dbt source freshness           # check raw data freshness
dbt run --select +fct_orders   # build fct_orders + all upstream dependencies
dbt snapshot                   # run all snapshots
```

---

## Schema Isolation Across Environments

`profiles.yml` sets `target.schema` per environment:

```yaml
# ~/.dbt/profiles.yml
my_project:
  target: dev
  outputs:
    dev:
      schema: dbt_alice      # individual developer
    ci:
      schema: ci             # automated PR checks
    prod:
      schema: analytics      # live data, BI tools
```

Override `generate_schema_name` so custom schemas are always prefixed with `target.schema`, keeping dev and prod fully isolated:

```sql
-- macros/generate_schema_name.sql
{% macro generate_schema_name(custom_schema_name, node) -%}
    {%- set default_schema = target.schema -%}
    {%- if custom_schema_name is none -%}
        {{ default_schema }}
    {%- else -%}
        {{ default_schema }}_{{ custom_schema_name | trim }}
    {%- endif -%}
{%- endmacro %}
```

| Environment | `target.schema` | Model config | Output schema |
|---|---|---|---|
| dev | `dbt_alice` | none | `dbt_alice` |
| dev | `dbt_alice` | `schema='marketing'` | `dbt_alice_marketing` |
| prod | `analytics` | none | `analytics` |
| prod | `analytics` | `schema='marketing'` | `analytics_marketing` |

Without this override, `schema='marketing'` resolves to just `marketing` in both environments — dev and prod would write to the same schema.
