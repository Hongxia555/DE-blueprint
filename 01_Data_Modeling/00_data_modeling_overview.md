# 01 — Data Modeling

> Notes on designing data models that are correct, queryable, and maintainable — from dimensional modeling to modern lakehouse patterns.

---

## Topics

### Schemas & Patterns
- [ ] **Star Schema** — facts, dimensions, grain definition; when to use
- [ ] **Snowflake Schema** — normalized dimensions; trade-offs vs star
- [ ] **Data Vault 2.0** — Hubs, Links, Satellites; auditability and flexibility
- [ ] **One Big Table (OBT)** — denormalized wide tables; use cases and pitfalls
- [ ] **Wide table vs normalized** — query performance, storage, and maintainability trade-offs

### Slowly Changing Dimensions (SCD)
- [ ] **SCD Type 1** — overwrite; no history retained
- [ ] **SCD Type 2** — new row per change; full history with `valid_from` / `valid_to`
- [ ] **SCD Type 3** — previous + current column; limited history
- [ ] **SCD Type 6** — hybrid of Type 1 + 2 + 3
- [ ] **Late-arriving data** — handling out-of-order events in SCD pipelines

### Grain & Fact Table Types
- [ ] **Defining grain** — the most important modeling decision
- [ ] **Transactional fact table** — one row per event (e.g. one row per ad impression)
- [ ] **Periodic snapshot** — state at regular intervals (e.g. daily account balance)
- [ ] **Accumulating snapshot** — tracks lifecycle of a process (e.g. order fulfillment)
- [ ] **Factless fact table** — records events with no numeric measure

### Modern Patterns
- [ ] **Medallion architecture** — Bronze / Silver / Gold applied to modeling layers
- [ ] **Activity schema / event modeling** — entity + activity grain for flexible analytics
- [ ] **Semantic layer** — dbt metrics, Cube.dev; separating business logic from queries
- [ ] **Data contracts** — schema agreements between producers and consumers

### dbt Modeling Patterns
- [ ] **dbt model layers** — staging → intermediate → marts
- [ ] **ref() and sources** — dependency management and lineage
- [ ] **dbt snapshots** — implementing SCD Type 2 with `strategy: timestamp`
- [ ] **Incremental models** — `is_incremental()`, unique keys, merge strategies

### Interview-Focused
- [ ] **Explaining grain to non-technical stakeholders**
- [ ] **Star schema vs Data Vault** — choosing for a given use case
- [ ] **Design from scratch** — e.g. ad impression model, e-commerce orders, SaaS subscriptions

---

## Key Concepts Cheat Sheet

| Concept | One-liner |
|---|---|
| Grain | The level of detail one row represents — define this first |
| Fact table | Stores measurable events (impressions, transactions, clicks) |
| Dimension table | Describes the who/what/where/when of facts (user, product, date) |
| SCD Type 2 | Tracks history by adding a new row each time a dimension attribute changes |
| Data Vault | Modular, audit-friendly design using Hubs (entities), Links (relationships), Satellites (attributes) |
| OBT | One wide denormalized table — fast to query, expensive to maintain |
| Semantic layer | A translation layer that maps raw columns to business metrics |
