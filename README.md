# FlowWatch

**An operational intelligence data platform for a simulated food-delivery business.**

FlowWatch simulates the data infrastructure behind a delivery company like Swiggy or
Zomato — the part a data engineer actually owns. It generates realistic order-lifecycle
events, moves them through a layered data lake, applies incremental ETL with historical
dimension tracking, and serves operational KPIs through an orchestrated daily pipeline.

> **Status: in active development.** Foundations and modeling phase. See
> [Progress](#progress) for exactly what exists today.

---

## The Idea

A delivery company's app emits a constant stream of events: an order is placed, a
restaurant confirms it, a rider is assigned, food is picked up and delivered, payment
clears. Individually these are just rows. Together they answer questions the business
can't operate without — *is our delivery SLA slipping, and is it the restaurants or the
riders causing it?*

FlowWatch builds everything between those two points.

**What FlowWatch is not:** a food delivery app. There is no customer-facing product here.
The event generator is a stand-in for an operational system, so the project can focus
entirely on the data platform.

---

## Why This Project

Most portfolio data projects load a static CSV into a database and chart it. That
demonstrates almost nothing about data engineering, because the hard parts never appear:
data doesn't arrive twice, nothing changes over time, and nothing needs to run tomorrow
without you.

FlowWatch is deliberately built around those hard parts:

| Real-world problem | How FlowWatch forces you to solve it |
|---|---|
| Data arrives continuously, not once | Event generator producing an ongoing lifecycle stream |
| The same file gets processed twice | Idempotent ingestion, deduplication by `event_id` |
| Records are malformed | Schema validation with quarantine for invalid events |
| Business entities change over time | SCD Type 2 dimensions preserving full history |
| Nobody runs pipelines by hand | Airflow orchestration with retries and backfill |
| Data can be silently wrong | Automated quality checks and tests in CI |

---

## Architecture

```
┌─────────────────────┐
│  Event Generator    │  Simulated app traffic (Python + Faker)
│  8 event types      │  Bangalore & Kolkata · ~400 orders/hr
└──────────┬──────────┘
           │  JSON, partitioned by event_type / event_date
           ▼
┌─────────────────────┐
│  BRONZE             │  Raw immutable events, exactly as received
└──────────┬──────────┘
           │  validate → flatten → convert
           ▼
┌─────────────────────┐
│  SILVER             │  Validated, flattened, columnar (Parquet)
└──────────┬──────────┘
           │  incremental load → change detection
           ▼
┌─────────────────────┐
│  STAGING            │  Typed, hashed, pre-merge (PostgreSQL)
└──────────┬──────────┘
           │  SCD2 merge → idempotent fact upsert
           ▼
┌─────────────────────┐
│  GOLD               │  Star schema warehouse + aggregates
│  facts + dimensions │
└──────────┬──────────┘
           ▼
     ┌───────────┐
     │ Metabase  │  Operational KPI dashboard
     └───────────┘

  Orchestrated end-to-end by Apache Airflow
```

**Design choices worth noting:**

- **Medallion layers (bronze/silver/gold)** — raw data is never mutated, so any downstream
  bug can be fixed by reprocessing rather than re-collecting.
- **Event-driven, not table-driven** — events are immutable facts with an `event_time`
  distinct from `ingestion_time`, so analytics reflect when things *happened*, not when
  they were recorded.
- **Every event earns its place** — an event type exists only because a specific KPI needs
  it. Design decisions are recorded in [`docs/decisions/`](docs/decisions).

---

## What It Does

**Event model** — eight event types covering the full order lifecycle:

`order_created` → `order_confirmed` → `rider_assigned` → `rider_reached_restaurant` →
`picked_up` → `delivered` → `payment_event`, plus `order_cancelled` for the alternate path.

**Questions it answers** — seven core business questions, including:

- Is peak demand a daily pattern, or specific to certain days?
- What is the on-time delivery percentage, by city?
- **Where** does delivery time actually go — restaurant prep, rider assignment wait,
  restaurant-side wait, or travel? *(Separates restaurant-caused delay from rider-caused
  delay.)*
- At which lifecycle stage are orders cancelled?

Full list, with scope reasoning and explicitly parked hypotheses:
[`docs/vision.md`](docs/vision.md)

**Simulated scale:** ~400 orders/hour · 150 riders · 230 restaurants · 2 cities
(Bangalore, Kolkata) · ~8 events per order.

---

## Tech Stack

### Generation 1 — Batch Platform

| Layer | Tool |
|---|---|
| Language | Python 3.11+ |
| Event simulation | Faker + custom lifecycle logic |
| Data lake | Local filesystem → Parquet |
| Processing | pandas + DuckDB |
| Warehouse | PostgreSQL (star schema, SCD Type 2) |
| Orchestration | Apache Airflow |
| Validation | Pydantic |
| Dashboard | Metabase |
| Testing / CI | pytest + GitHub Actions |
| Environment | Docker Compose |

### Generation 2 — Streaming Lakehouse

| Layer | Tool |
|---|---|
| Batch processing | Apache Spark (PySpark) |
| Ingestion | Apache Kafka (KRaft mode) |
| Stream processing | Spark Structured Streaming — watermarking, dedup |
| Table format | Delta Lake |
| Transformation | dbt |
| Lineage | OpenLineage + Marquez |
| Cloud | Azure / AWS fundamentals |

---

## Build Path: v1 → v2

FlowWatch is built in two deliberate generations rather than all at once.

**Generation 1** delivers a complete, tested batch platform. **Generation 2** then
replaces one component at a time — Kafka for file ingestion, Spark for pandas, Delta for
Parquet — so the pipeline stays runnable at every step, never mid-rewrite.

Why this order: Kafka, Spark Structured Streaming, Delta Lake, and dbt are each an
independent skill. Attempting all of them before anything works is how ambitious projects
end up abandoned at 60%. Building the batch version first also means the event model and
SCD2 logic get validated under a simple execution model before streaming complexity is
added on top.

| | Generation 1 | Generation 2 |
|---|---|---|
| **Goal** | Correct, tested batch platform | Streaming lakehouse |
| **Ingestion** | File-based | Kafka |
| **Processing** | pandas / DuckDB | Spark (batch → streaming) |
| **Storage** | Parquet | Delta Lake |
| **Transformation** | SQL + Python | dbt |
| **Observability** | Logs + audit tables | OpenLineage + Marquez |
| **Release** | `v1.0` | `v2.0` |

---

## Progress

**Currently:** Phase 1 — Event & Data Modeling

| Phase | Focus | Status |
|---|---|---|
| 0 | Setup, scope, vision | ✅ Complete |
| 1 | Event modeling, star schema design | 🔄 In progress |
| 2 | Data lake & ingestion | ⬜ Not started |
| 3 | Incremental ETL + SCD Type 2 | ⬜ Not started |
| 4 | Airflow orchestration | ⬜ Not started |
| 5 | Warehouse, KPIs, dashboard | ⬜ Not started |
| 6 | Testing, data quality, CI | ⬜ Not started |
| 7 | Documentation & `v1.0` release | ⬜ Not started |
| 8–14 | Generation 2 | ⬜ Not started |

**Done so far:** project scope and simulated business assumptions defined · seven core
business questions and associated KPIs · event model settled at eight event types with a
standard envelope · explicit scope boundaries documented, including hypotheses parked for
lack of supporting data.

---

## Repository Structure

```
flowwatch-platform/
├── docs/                  # Vision, event schema, architecture, decision records
├── event_generator/       # Simulated app event producer
├── data_lake/             # bronze / silver / gold
├── etl/                   # Ingestion, validation, transforms, SCD2 merges
├── warehouse/             # DDL and KPI queries
├── airflow/dags/          # Orchestration
├── tests/                 # Unit and integration tests
└── scripts/               # Local run helpers, replay/backfill
```

---

## Getting Started

> Setup instructions will be completed as the pipeline is built. Currently only the
> warehouse container is defined.

```bash
git clone https://github.com/<username>/flowwatch-platform.git
cd flowwatch-platform

python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

docker compose up -d          # PostgreSQL
```

---

## Documentation

| Document | Contents |
|---|---|
| [`docs/vision.md`](docs/vision.md) | Scope, business questions, KPIs, parked hypotheses |
| `docs/event_schema.md` | Event envelope and payload definitions |
| `docs/decisions/` | Design decision records |
| `docs/architecture/` | Architecture and data model diagrams |
| `docs/runbook.md` | Operating and troubleshooting the pipeline |

---

License

**MIT**
## License

MIT
