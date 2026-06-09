# dbt FAQ

**Related:** [[dbt-and-testing]] · [[dbt-example-ecommerce]] · [[data-modeling]] · [[sql]]

---

## 1. What is dbt and what problem does it solve?

dbt (data build tool) is an open-source transformation framework. You write SELECT statements; dbt handles the DDL/DML to materialize them in your warehouse. It treats SQL transformations as software — version control, CI/CD, dependency graphs, docs, and testing built in.

**Before dbt:** transformation logic scattered across stored procedures, scripts, and ad hoc SQL with no lineage or testing.  
**After dbt:** transformation = a `.sql` file in Git, with dependency resolution, test assertions, and auto-generated docs.

---

## 2. What is a model?

A model is a single `.sql` file containing one SELECT statement in your `models/` directory. When you run `dbt run`, dbt wraps that SELECT in DDL based on the configured materialization and executes it against your warehouse.

Models are the atomic unit of transformation in dbt.

---

## 3. What are the four materializations? When do you use each?

| Materialization | What it does | When to use |
|---|---|---|
| `view` | Creates a DB view — re-executes SQL on every query | Lightweight transformations, no storage cost |
| `table` | Drops and recreates a full table each run | Downstream needs fast reads; data fits in a full rebuild |
| `incremental` | Appends/merges only new/changed rows | Large fact tables — full rebuild is too slow or expensive |
| `ephemeral` | Inlines as a CTE into the consuming model | Reusable logic you never want as a standalone object |

> **Follow-up trap:** "What's the downside of `table`?" — Full rebuild every run. On a billion-row fact table, this is too expensive → use `incremental`.

---

## 4. How does `ref()` work?

`{{ ref('model_name') }}` resolves to the fully qualified table/view name of another dbt model **and** registers a dependency edge in the DAG. dbt parses all `ref()` calls to infer model order automatically.

```sql
-- staging/stg_orders.sql
SELECT order_id, user_id, amount FROM {{ source('raw', 'orders') }}

-- marts/fct_orders.sql
SELECT * FROM {{ ref('stg_orders') }}   -- depends on stg_orders
```

Never hardcode schema/table names — always use `ref()` so dbt can manage the dependency graph and environment promotion (dev → prod schemas).

---

## 5. What is `source()` and why use it over a raw table reference?

`{{ source('schema_name', 'table_name') }}` references a raw ingested table (not a dbt model). Using `source()`:
- Registers the raw table in the lineage graph
- Enables **source freshness checks** (`dbt source freshness`)
- Provides a single place to document and test raw data quality

Without `source()`, raw tables are invisible in the DAG and you can't assert freshness SLAs.

---

## 6. What tests does dbt provide out of the box?

**Generic (schema) tests** — `data_tests:` blocks in any YAML file. Convention: co-locate with the layer being tested.

| File | Tests on |
|---|---|
| `_sources.yml` | Raw source tables |
| `_staging.yml` | Staging models |
| `_mart.yml` | Mart models |

| Test | What it checks |
|---|---|
| `unique` | No duplicate values in a column |
| `not_null` | No NULL values |
| `accepted_values` | Column only contains values from a defined list |
| `relationships` | FK integrity — value exists in another model's column |

```yaml
# models/staging/_sources.yml — generic tests on raw source tables
sources:
  - name: raw
    tables:
      - name: orders
        columns:
          - name: order_id
            data_tests:
              - unique
              - not_null
```

```yaml
# models/mart/_mart.yml — generic tests on mart models
models:
  - name: fct_orders
    columns:
      - name: order_id
        data_tests:
          - unique
          - not_null
      - name: customer_id
        data_tests:
          - relationships:
              to: ref('dim_customers')
              field: customer_id
```

**Source freshness checks** — assert that raw source tables are being updated on time. Also lives in `_sources.yml`, run with `dbt source freshness`.

```yaml
# models/staging/_sources.yml — freshness check
sources:
  - name: raw
    tables:
      - name: orders
        loaded_at_field: created_at
        freshness:
          warn_after: {count: 6, period: hour}
          error_after: {count: 24, period: hour}
```

dbt compares `MAX(loaded_at_field)` against the current time. If the gap exceeds `warn_after` → warning; exceeds `error_after` → error. Not a data quality test — it checks whether ingestion is running at all.

How dbt evaluates it internally:

```sql
SELECT MAX(created_at) FROM raw.orders
-- if NOW() - MAX(created_at) > 6h  → warn
-- if NOW() - MAX(created_at) > 24h → error
```

| Gap since last row | Status | Meaning |
|---|---|---|
| < 6 hours | ✅ Fresh | All good |
| 6–24 hours | ⚠️ Warn | Ingestion may be delayed |
| > 24 hours | ❌ Error | Ingestion is broken |

`period` accepts `minute`, `hour`, or `day`. A high-frequency event stream might use `warn_after: {count: 30, period: minute}`.

**Singular tests** — custom `.sql` files in `tests/` that return rows on failure (zero rows = pass).

```sql
-- tests/assert_order_amount_positive.sql
SELECT order_id, amount
FROM {{ ref('fct_orders') }}
WHERE amount <= 0
```

```sql
-- tests/assert_orders_have_valid_users.sql
SELECT o.order_id
FROM {{ ref('fct_orders') }} o
LEFT JOIN {{ ref('dim_users') }} u ON o.user_id = u.user_id
WHERE u.user_id IS NULL
```

The pattern is always the same: write a query that selects the bad rows. Zero rows = pass.

---

## 7. What is the difference between `schema.yml` and `sources.yml`?

- `schema.yml` — documents and tests **dbt models** (your transformed tables)
- `sources.yml` — documents and tests **raw source tables** (ingested by external tools)

In practice, teams split these files by layer rather than keeping one big `schema.yml`:

```
models/
  staging/
    _sources.yml    ← source declarations, freshness checks, tests on raw tables
    _staging.yml    ← tests + descriptions for staging models
  mart/
    _mart.yml       ← tests + descriptions for mart models
```

The file names don't matter to dbt — it picks up any `.yml` file in the `models/` directory. The convention of `_sources.yml` / `_staging.yml` / `_mart.yml` just keeps each layer self-contained and easier to navigate.

---

## 8. How does incremental materialization work?

On first run: full table build.  
On subsequent runs: dbt filters the incoming SQL to only new/changed rows (using a `unique_key` + `is_incremental()` macro) and appends or merges.

```sql
-- models/fct_events.sql
{{ config(materialized='incremental', unique_key='event_id') }}

SELECT event_id, user_id, event_type, created_at
FROM {{ source('raw', 'events') }}
{% if is_incremental() %}
  WHERE created_at > (SELECT MAX(created_at) FROM {{ this }})
{% endif %}
```

> **Follow-up:** "What is `{{ this }}`?" — It resolves to the current model's own table, allowing you to filter against its current max timestamp.

Everything in that file belongs together — there are no separate config files per model:

- `{{ config() }}` — materialization settings, top of the same file
- `SELECT` — your transformation logic
- `{% if is_incremental() %}` — Jinja conditional, part of the same SQL file

The only things that live in separate files are:

| What | Where |
|---|---|
| Tests + descriptions | `_staging.yml` / `_mart.yml` |
| Singular tests | `tests/assert_*.sql` |
| Snapshots | `snapshots/snap_*.sql` |

Everything else (config, logic, incremental filter) = one `.sql` file per model.

---

## 9. What is the DAG and why does it matter?

The **DAG (Directed Acyclic Graph)** is the dependency graph dbt builds by parsing all `ref()` calls. It determines execution order and enables:
- `dbt run --select +model_name` — run a model and all its upstream dependencies
- `dbt run --select model_name+` — run a model and all downstream dependents
- Visualizing data lineage in dbt docs

---

## 10. What are seeds?

CSV files in the `seeds/` directory that dbt loads as tables. Use for small static lookup data (country codes, status maps, category labels).

**Do not use for:** anything over a few thousand rows, or data that changes frequently → use a source table instead.

---

## 11. What are snapshots?

Snapshots implement **SCD Type 2** — they track row-level history for a source table that mutates over time.

```sql
-- snapshots/snap_customers.sql
{% snapshot snap_customers %}
  {{ config(target_schema='snapshots', unique_key='id', strategy='timestamp', updated_at='updated_at') }}
  SELECT * FROM {{ source('raw', 'customers') }}
{% endsnapshot %}
```

dbt adds `dbt_valid_from`, `dbt_valid_to`, `dbt_scd_id` columns on top of all source columns. The logic on each run:

- **Row unchanged** — untouched
- **Row changed** — old row gets `dbt_valid_to = now()` (closed); new row inserted with updated values and `dbt_valid_to = NULL` (current)
- **Initial load** — all rows inserted with `dbt_valid_from = now()`, `dbt_valid_to = NULL`

Example — customer changes email:

| customer_id | email | dbt_valid_from | dbt_valid_to |
|---|---|---|---|
| 42 | old@gmail.com | 2024-01-01 | 2024-02-01 |
| 42 | new@gmail.com | 2024-02-01 | NULL |

```sql
-- current state only
SELECT * FROM snap_customers WHERE dbt_valid_to IS NULL

-- state at a point in time
SELECT * FROM snap_customers
WHERE dbt_valid_from <= '2024-01-15'
  AND (dbt_valid_to > '2024-01-15' OR dbt_valid_to IS NULL)
```

> **Relation to [[ads-ranking]]:** `dim_ad_sets` uses SCD Type 2 for the same reason — bid and targeting changes must be reconstructable for historical audit.

---

## 12. What is `generate_schema_name` and why would you override it?

By default, dbt puts all models in the target schema (e.g., `dbt_alice` in dev, `analytics` in prod). `generate_schema_name` is a macro you override to control how custom schema configs like `{{ config(schema='marketing') }}` resolve across environments.

**Step 1 — `profiles.yml` defines the target schema per environment:**

```yaml
# ~/.dbt/profiles.yml
my_project:
  target: dev
  outputs:
    dev:
      schema: dbt_alice      # target.schema in dev
    ci:
      schema: ci             # target.schema in CI (automated PR checks)
    prod:
      schema: analytics      # target.schema in prod
```

**Step 2 — override the macro in `macros/generate_schema_name.sql`:**

```sql
{% macro generate_schema_name(custom_schema_name, node) -%}
    {%- set default_schema = target.schema -%}
    {%- if custom_schema_name is none -%}
        {{ default_schema }}
    {%- else -%}
        {{ default_schema }}_{{ custom_schema_name | trim }}
    {%- endif -%}
{%- endmacro %}
```

**How it resolves:**

| Environment | `target.schema` | Model config | Output schema |
|---|---|---|---|
| dev | `dbt_alice` | none | `dbt_alice` |
| dev | `dbt_alice` | `schema='marketing'` | `dbt_alice_marketing` |
| prod | `analytics` | none | `analytics` |
| prod | `analytics` | `schema='marketing'` | `analytics_marketing` |

**Why the default behavior is problematic:** without this override, `schema='marketing'` resolves to just `marketing` in both dev and prod — they'd write to the same schema. The override always prefixes `target.schema`, keeping environments fully isolated.

---

## 13. How do you promote models from dev to prod?

dbt uses **target profiles** — `dev`, `ci`, and `prod` are different targets in `profiles.yml` with different schemas. The same model code runs against all three; dbt resolves `ref()` and `source()` to the correct target.

| Environment | Who uses it | Schema |
|---|---|---|
| dev | Individual developer | `dbt_alice` |
| ci | Automated PR checks | `ci` (or `ci_pr_123` per PR) |
| prod | Live data, BI tools | `analytics` |

The `ci` schema is a dedicated temporary schema used only for automated PR checks — keeps test runs isolated from both dev and prod. Multiple PRs can run in parallel without stepping on each other. Gets cleaned up after the PR merges.

CI/CD pattern:
1. PR opened → CI pipeline runs `dbt run` + `dbt test` with `--target ci` → models land in `ci` schema
2. Tests pass, PR merges → CD deploys to `prod` schema via dbt Cloud or Airflow/Prefect

---

## 14. Where is dbt transformation logic NOT appropriate?

- **Data ingestion** — dbt doesn't move data into the warehouse (use Fivetran, Airbyte, Kafka)
- **Orchestration** — dbt doesn't schedule itself (use Airflow, Prefect, dbt Cloud)
- **Very large row-level Python transforms** — dbt SQL can't do ML feature engineering; use dbt-python models sparingly or push to Spark/Ray

**dbt Python models** — dbt added Python model support for Snowflake/Databricks. You write a Python function instead of SQL:

```python
# models/fct_features.py
def model(dbt, session):
    df = dbt.ref("stg_events").to_pandas()
    df["rolling_7d"] = df.groupby("user_id")["amount"].transform(
        lambda x: x.rolling(7).mean()
    )
    return df
```

**Why "use sparingly":**
- Runs on warehouse compute (Snowflake warehouse, Databricks cluster) — expensive
- Slower than SQL for simple transforms
- Harder to test and document than SQL models
- Not supported on all warehouses (Redshift, BigQuery have limited support)

**When to push to Spark/Ray instead:**

| Scenario | Use |
|---|---|
| Simple feature: rolling avg, lag, rank | dbt Python model is fine |
| Training a model on 100M+ rows | Spark/Ray — warehouse compute isn't designed for this |
| Real-time inference at serving time | Spark/Ray or a dedicated ML platform (Feast, Tecton) |
| Distributed matrix operations | Spark MLlib / Ray Train |

**The mental model:** dbt = SQL-first warehouse transforms; Spark/Ray = heavy distributed Python compute. Many pipelines use both — dbt for warehouse transforms, Spark/Ray for ML-specific heavy lifting upstream or downstream.

---

## 15. How do you monitor dbt in production?

- `run_results.json` — emitted to `target/run_results.json` after every `dbt run`/`dbt test`; contains per-model timing and status
- `dbt source freshness` — checks when source tables were last updated against a configured `warn_after` / `error_after` threshold
- dbt Cloud: built-in run history, alerting, and observability dashboard — no manual setup needed
- dbt Core: parse `run_results.json` and push metrics to Datadog, Grafana, or a monitoring table

**What `run_results.json` looks like:**

```json
{
  "metadata": {
    "dbt_version": "1.7.0",
    "generated_at": "2024-02-01T10:30:00Z",
    "elapsed_time": 45.2
  },
  "results": [
    {
      "unique_id": "model.my_project.fct_orders",
      "status": "success",
      "execution_time": 12.4,
      "adapter_response": {"rows_affected": 150000}
    },
    {
      "unique_id": "test.my_project.not_null_fct_orders_order_id",
      "status": "pass",
      "execution_time": 0.8,
      "failures": 0
    },
    {
      "unique_id": "test.my_project.unique_fct_orders_order_id",
      "status": "fail",
      "execution_time": 0.9,
      "failures": 42        
    }
  ]
}
```

**Key fields per result:**

| Field | What it tells you |
|---|---|
| `unique_id` | Which model or test |
| `status` | `success` / `error` for models; `pass` / `fail` / `warn` for tests |
| `execution_time` | How long it took in seconds |
| `failures` | Number of failing rows (tests only) |
| `rows_affected` | Rows inserted/merged (models only) |

**dbt Cloud vs dbt Core:**

| | dbt Cloud | dbt Core |
|---|---|---|
| Run history | Built-in dashboard | Parse `run_results.json` yourself |
| Test failures | Alerts via email/Slack | You build the alerting |
| Model timing | Visual charts | Push to Datadog/Grafana |
| Scheduling | Built-in scheduler | Need Airflow/Prefect |
| `run_results.json` | Still generated, but no need to touch it | Must parse manually |

Most companies start with dbt Core (free, flexible) and move to dbt Cloud when the operational overhead of managing scheduling, alerting, and observability becomes too much.

---

## Quick-Reference Cheat Sheet

| Question | One-line answer |
|---|---|
| What is a model? | One `.sql` SELECT → one warehouse object |
| `ref()` vs `source()` | `ref()` = another dbt model; `source()` = raw ingested table |
| Best materialization for big facts? | `incremental` with `unique_key` |
| How does dbt know run order? | DAG parsed from `ref()` calls |
| What are snapshots for? | SCD Type 2 — track row history |
| What are seeds for? | Small static CSV lookup tables |
| Four generic tests? | `unique`, `not_null`, `accepted_values`, `relationships` |
| How to run only affected models? | `dbt run --select +changed_model+` |

---

## Appendix: DDL vs DML

**DDL — Data Definition Language**
Commands that define or modify the *structure* of database objects.

- `CREATE TABLE`, `ALTER TABLE`, `DROP TABLE`
- `CREATE VIEW`, `CREATE INDEX`

**DML — Data Manipulation Language**
Commands that work with the *data inside* those structures.

- `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `MERGE`

> In dbt: you write a plain `SELECT` (DML), and dbt generates the surrounding `CREATE TABLE AS ...` or `CREATE VIEW AS ...` (DDL) based on your materialization config.

---

## Appendix: CI/CD — Continuous Integration / Continuous Delivery

**CI — Continuous Integration**
Every code change (PR/commit) automatically triggers a build and test pipeline. Goal: catch broken code before it merges.

- Run `dbt run` + `dbt test` on every PR
- If tests fail, the merge is blocked

**CD — Continuous Delivery / Deployment**
Automatically promote validated code to the next environment (staging → prod) after CI passes.

- Merge to `main` → automatically deploys to prod schema
- No manual "go deploy it" step

> In dbt: open a PR → **CI** runs `dbt run` + `dbt test` in a `ci` schema → tests pass, PR merges → **CD** deploys models to `prod` schema via dbt Cloud or Airflow.

---

## Appendix: Auto-Generated Docs

dbt generates a **static website** you run locally or host via dbt Cloud. Two main parts:

**1. Model/Source Browser (left sidebar)**
A searchable tree of all models, sources, seeds, and snapshots. Click any model to see:
- Compiled SQL (with `ref()` resolved to real table names)
- Column descriptions from `schema.yml`
- Tests defined on each column
- `Referenced by` (downstream) and `Depends on` (upstream) models

**2. DAG Lineage Graph (bottom panel)**
An interactive node graph — each node is a model or source, edges show `ref()` dependencies.

```
[source: raw.orders] ──→ [stg_orders] ──→ [fct_orders] ──→ [rpt_revenue]
[source: raw.users]  ──→ [stg_users]  ──┘
```

You can click any node to focus, expand upstream (`+model`) or downstream (`model+`).

**How to generate:**

```bash
dbt docs generate   # builds the static site
dbt docs serve      # opens at localhost:8080
```

Content comes from `description:` fields in `schema.yml`. If you write no descriptions, the docs still generate — they just show SQL and lineage with no annotations.

---

## Appendix: Configuring Materialization

Three places to define it — highest priority wins.

**1. `{{ config() }}` block inside the model file (highest priority):**

```sql
-- models/fct_orders.sql
{{ config(
    materialized='incremental',
    unique_key='order_id'
) }}

SELECT order_id, user_id, amount, created_at
FROM {{ ref('stg_orders') }}
```

**2. `schema.yml` (per model):**

```yaml
models:
  - name: fct_orders
    config:
      materialized: incremental
      unique_key: order_id
```

**3. `dbt_project.yml` (applies to a whole folder, lowest priority):**

```yaml
models:
  my_project:
    staging:
      +materialized: view      # all models in staging/ → view
    marts:
      +materialized: table     # all models in marts/ → table
```

**Priority order:** `{{ config() }}` block > `schema.yml` > `dbt_project.yml`

Set folder-wide defaults in `dbt_project.yml`, override individual models with `{{ config() }}` when needed.

All four materializations can be set in any of the three places — same config options, just swap the value:

**Config block:**
```sql
{{ config(materialized='view') }}
{{ config(materialized='table') }}
{{ config(materialized='incremental', unique_key='order_id') }}
{{ config(materialized='ephemeral') }}
```

**`schema.yml` (per model):**
```yaml
models:
  - name: stg_orders
    config:
      materialized: view

  - name: fct_orders
    config:
      materialized: incremental
      unique_key: order_id
```

**`dbt_project.yml` (folder-wide):**
```yaml
models:
  my_project:
    staging:
      +materialized: view
    intermediate:
      +materialized: ephemeral
    mart:
      +materialized: table
```

> `incremental` needs `unique_key` as an extra config — the other three only need `materialized`. `ephemeral` is rarely set per-model; it's usually a folder-wide default in `dbt_project.yml`.
