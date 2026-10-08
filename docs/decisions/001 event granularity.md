# 001 — Event Granularity

**Status:** Accepted
**Phase:** 1 — Event & Data Modeling

## Context

FlowWatch records the lifecycle of a food order as events. Before writing the event
generator, three questions about granularity had to be settled: how many event types to
use, how to record where in the lifecycle a cancellation happened, and how to handle an
order that needs more than one payment attempt.

Full field definitions are in [`docs/event_schema.md`](../event_schema.md).

---

## Decision 1 — One event type per lifecycle step

Each step of the order lifecycle is its own event type, rather than one generic
`order_status_change` event with a status field. There are 8 types: `order_created`,
`order_confirmed`, `rider_assigned`, `rider_reached_restaurant`, `picked_up`,
`delivered`, `payment_event`, and `order_cancelled`.

**Why:** each step carries different facts in its payload, so one generic shape would
either be mostly empty or hide the real structure. Every event type exists because a KPI
needs it:

- **`rider_reached_restaurant` earned its place** because restaurant-side wait time
  (`picked_up − rider_reached_restaurant`) is a Core KPI. It separates delay caused by the
  restaurant from delay caused by the rider.
- **`payment_event` is separate from `order_created`** because the amount quoted at
  checkout and the amount actually settled are different facts — a coupon can fail or a
  discount can change.

Durations (waiting time, travel time) are **not** events. They are calculated from the
`event_time` of two events.

---

## Decision 2 — Record the cancellation stage in the payload

`order_cancelled` records `cancelled_by`, `cancelled_at_stage`, and `reason` as three
separate fields, each with a fixed vocabulary.

**Why:** the cancellation stage is a vital fact — it decides how much a cancellation cost
the business and affects payment and system handling. At the moment of cancellation, the
app knows the true stage for certain, so it is recorded right then.

The alternative — working out the stage later from which events exist for the order — is
unreliable. Events can arrive late or never (for example, a rider's phone with no signal).
If `rider_assigned` has not arrived yet, the pipeline would wrongly decide that no rider
was assigned, and it would not know it was wrong.

> Record a fact at the moment it is known for certain. Don't rebuild it later from data
> that might be incomplete.

Cancellation is only allowed **before pickup**. The four valid stages:

| Stage | Example |
|---|---|
| `before_confirmation` | The customer cancels right after ordering, or the restaurant rejects it |
| `after_confirmation` | The restaurant confirmed, but a mistake or technical issue forces a cancel |
| `after_rider_assigned` | No rider is available, or the item turns out to be out of stock |
| `after_rider_reached` | The customer changes their mind while the rider is at the restaurant |

After pickup, problems are not cancellations: a failed payment is a `payment_event` with
`status: failed`, and a customer unavailable at the door is still an open question (see
Parked).

---

## Decision 3 — One `payment_event` per payment attempt

One order can produce several `payment_event`s, one for every attempt, numbered with
`attempt_number`. Attempts are recorded until a payment succeeds.

Example: the card machine fails, then a UPI payment fails because of a bank server
problem, then the customer pays cash. That is **three** `payment_event`s for one order.

**Grain of payment data: one row = one payment attempt** — not one order.

**Why keep every attempt:** the Core question "what is the daily payment success/failure
rate?" needs the failed attempts. If only successful payments were stored, the failures
would be gone and the rate could not be calculated.

**How to avoid doubling revenue:** joining orders (one row = one order) to payments (one
row = one attempt) multiplies rows — an order with two attempts appears twice, and its
amount is counted twice. So the payments table is filtered or collapsed **only for the
query that needs it**, never in storage:

| Question | Use |
|---|---|
| Payment failure rate | All attempts — the fine grain |
| Revenue | Successful payments only — one row per order |
| Joining payments to orders | Filter or collapse to one row per order first |

The failure rate KPI must state which meaning it uses: *% of attempts that failed*, or
*% of orders with at least one failed attempt*. Both are valid; they answer different
questions.

---

## Parked

- **Cash change.** When a customer pays cash and the rider covers the bill or gives change
  (e.g. ₹10 extra), the amount handed over differs from `total_amount`. Cash-handling
  detail — not modelled in v1.
- **Customer unavailable at the door.** This happens after pickup, so it cannot be a
  cancellation under Decision 2. Undecided whether it needs its own outcome event or is
  out of scope for v1.

## Consequences

- The event generator must produce one event per lifecycle step, multiple payment attempts
  for some orders, and cancellations only at the four valid stages.
- Any fact table built on payments has a grain of one attempt, and must be filtered or
  collapsed before joining to order-level tables.
  
