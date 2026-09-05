# FlowWatch — Vision Document

## 1. Project Overview

FlowWatch is an Operational Intelligence Data Platform simulating the analytics
infrastructure behind a food-delivery company (Swiggy/Zomato-style). It ingests
simulated app events, processes them through an incremental ETL pipeline with
historical dimension tracking (SCD Type 2), and serves operational KPIs through a
dashboard — orchestrated automatically end to end.

## 2. Simulated Company Assumptions

> Keep these numbers modest and plausible — a believable small-scale simulation is more
> credible than an inflated one.

- Orders per hour: **[fill in — e.g. 300–500]**
- Active riders: **[fill in — e.g. 100–200]**
- Restaurants onboarded: **[fill in]**
- Operating cities: **[fill in — e.g. 2–3]**
- Average events generated per order: **[fill in — e.g. 6–10]**

## 3. Business Questions

### Core (must answer with v1)
1.
2.
3.
4.
5.

### Extended (v2 / financial — future scope, not built yet)
1.
2.

## 4. KPIs to Track

### Core Operational KPIs (Phase 1 — must build)
-
-
-

### Extended KPIs (future)
-
-

## 5. High-Level Architecture (Text Only — Diagram Comes Later)

```
Event Generator (simulated app)
  → Data Lake (Bronze/Silver)
  → Incremental ETL + SCD Type 2
  → Postgres Warehouse (Star Schema)
  → Airflow (daily orchestration)
  → Dashboard (Metabase)
```

## 6. Project Goals

- Build an event-driven ingestion pipeline
- Implement SCD Type 2 for dimension history
- Automate the full pipeline with Airflow
- Serve operational KPIs through a live dashboard
- (Generation 2, later) Evolve into a streaming lakehouse: Kafka, Spark Structured
  Streaming, Delta Lake, dbt, lineage tracking

## 7. Explicit Non-Goals for v1

- No real OLTP application — an event generator simulates OLTP output
- No Kafka/streaming in v1 (scheduled for Generation 2)
- No cloud deployment in v1 (scheduled for Generation 2)
