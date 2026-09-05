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

- Orders per hour: **400**
- Active riders: **150**
- Restaurants onboarded: **230**
- Operating cities (v1): **Bangalore, Kolkata** — chosen to contrast a high-volume metro
  (Bangalore) against a smaller market (Kolkata), so city-level KPIs show meaningful
  variation rather than two samey numbers
- Full target market (future / Generation 2 scale-up): Bangalore, Delhi, Mumbai, Chennai,
  Kolkata
- Average events generated per order: **8** (to be confirmed against `event_schema.md`
  once finalized in Session 2 — update this line to match reality, not the other way
  around)

## 3. Business Questions

### Core (must answer with v1)
1. What is order volume by hour-of-day and day-of-week, per city — is peak demand a daily pattern or specific to certain days?
2. What is the on-time delivery percentage, measured daily, by city (Bangalore vs Kolkata)?
3. What is the delivery time breakdown — average restaurant prep time, average rider-assignment wait time, and average travel time — by city? (Lets us see *why* delivery time differs between cities, not just that it does.)
4. Which restaurants generate the highest order volume, per week?
5. What is the rider utilization rate (orders delivered per rider per shift), by city?
6. What percentage of orders are cancelled, broken down by the lifecycle stage they were cancelled at — before restaurant confirmation, after confirmation but before rider assignment, after rider assignment but before pickup, or after pickup?
7. What is the daily payment success/failure rate?

### Extended (v2 / financial — future scope, not built yet)
1. What is the total revenue and estimated profit per city, per month, including estimated losses from payment failures? (requires expense/commission event modeling not yet in scope)
2. What is customer repeat-order rate over a 30-day window? (requires longer historical retention than v1's initial dataset)
3. Does cancellation rate increase when predicted delivery time exceeds the SLA threshold? (a correlation question — meaningful, but a step up in analytical complexity from the Core list; revisit once Core KPIs are solid)

### Parked — Real Hypotheses, No Data to Test Them Yet
These came up naturally while thinking through the order lifecycle, but answering them
honestly would require event types or fields FlowWatch doesn't currently model. Naming
them here (instead of quietly dropping them) keeps the scope decision visible and
intentional:
- **Is rider supply distributed proportionally to demand across zones/areas within a
  city?** Would require adding a zone/area concept to orders and rider locations —
  currently out of scope, but a strong candidate for a v1.5 schema extension since it's
  answerable without new event *types*, just a new field.
- **Root cause of delivery delays: traffic vs. distance vs. vehicle breakdown vs. GPS/location
  error.** Would require a `delivery_incident` event type and/or GPS accuracy data —
  genuinely out of scope for v1's 5 event families.
- **Does dine-in demand compete with delivery capacity at busy restaurants?** Would
  require dine-in order data, which FlowWatch never models — this is a "not ever in
  scope" item, not a near-term extension.

## 4. KPIs to Track

### Core Operational KPIs (Phase 1 — must build)
- Order volume, by hour-of-day and day-of-week, by city
- Delivery SLA % (orders delivered within target time / total delivered orders), by city
- Average restaurant prep time (order_confirmed → picked_up-ready), by city
- Average rider-assignment wait time (order_confirmed → rider_assigned), by city
- Average travel time (picked_up → delivered), by city
- Top restaurants by order volume, weekly
- Rider utilization rate (orders per rider per day), by city
- Order cancellation rate, broken down by lifecycle stage
- Payment success/failure rate, daily

### Extended KPIs (future)
- Revenue and estimated profit per city, per month
- Estimated financial loss from payment failures
- Customer repeat-order rate (30-day window)

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

- No real OLTP application — an event generator simulates OLTP output
- No Kafka/streaming in v1 (scheduled for Generation 2)
- No cloud deployment in v1 (scheduled for Generation 2)
