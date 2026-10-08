# FlowWatch — Event Schema

Every event in FlowWatch has two parts:

- **Envelope** — the outside. Same five fields on every event, always.
- **Payload** — the inside. Only the facts that are *new* at that moment. Different for
  every event type.

The ingestion pipeline reads only the envelope. It never needs to look inside the payload.

---

## Part A — The Envelope

| Field | Type | Created by | Purpose |
|---|---|---|---|
| `event_id` | string (UUID) | Event generator | A unique identity for this one event. No two events ever share it. Used by ingestion to find and reject duplicates. |
| `event_type` | enum | Event generator | Tells the pipeline which of the 8 event types this is. Decides which partition folder the event is stored in, and what shape the payload has. |
| `event_time` | timestamp, timezone-explicit | Event generator | The exact moment the event happened in the real world. Always includes the timezone offset (e.g. `+05:30`). All analytics use this time. |
| `ingestion_time` | timestamp \| null | Ingestion pipeline | When the system received the event. Always `null` when the event is created, because the generator cannot know when it will arrive — it may be delayed by a signal problem or a technical glitch. Ingestion stamps it on arrival. |
| `order_id` | string \| null | Event generator | The same for every event that belongs to one order. Each event has its own `event_id`, but all events of one order share the same `order_id`. This is what ties an order's whole story together. |

### Rules for every payload

1. **Never repeat an envelope field.** No `order_id` and no `*_time` field for the event's
   own moment — the envelope already has them.
2. **Only what is new at this moment.** Facts recorded in an earlier event are not
   repeated. Join on `order_id` to get them.
3. **IDs, not names.** Names, addresses, and tax IDs live in the dimension tables.
4. **Numbers as numbers.** Units go in the field name (`_km`, `_minutes`), not the value.
5. **Fixed vocabularies for categories**, so they can be grouped in SQL.
6. **Durations are never stored.** They are calculated from two events' `event_time`.
7. **An empty payload is valid** when the only new fact is *that* it happened.

---

## Part B — The Events

### 1. order_created

**Captures:** A customer has placed an order for food.

| Field | Type | Why it's needed |
|---|---|---|
| `customer_id` | string | Which customer made the order. Joins to `dim_customer`. |
| `restaurant_id` | string | Which restaurant the order is for. Joins to `dim_restaurant`. |
| `city` | enum | Which city the order was made in. `Bangalore` or `Kolkata`. Needed for every city-level KPI. |
| `zone` | string | Area-level delivery location (e.g. `Salt Lake Sector V`). Not a street address. Captured now for the parked zone-density analysis. |
| `items` | array | Each item ordered, with `item_id`, `name`, `quantity`, `unit_price`. Prices are a point-in-time snapshot — they must not change if the menu price changes later. |
| `quoted_amount` | decimal | The bill amount shown to the customer at checkout. Compared later with the settled amount in `payment_event`. |
| `payment_mode_selected` | enum | `online` or `pay_on_delivery`. |
| `promised_delivery_time` | timestamp | The delivery time promised to the customer at order time. Needed for the on-time delivery SLA KPI. |

```json
{
  "event_id": "evt_1a7c3e90-2f4b-4c8a-9d61-0e5b7a2c4f11",
  "event_type": "order_created",
  "event_time": "2026-03-15T11:00:00+05:30",
  "ingestion_time": null,
  "order_id": "ORDCH001",
  "payload": {
    "customer_id": "CUST050",
    "restaurant_id": "REST010",
    "city": "Kolkata",
    "zone": "Salt Lake Sector V",
    "items": [
      { "item_id": "ITM221", "name": "Butter Chicken", "quantity": 1, "unit_price": 320.00 },
      { "item_id": "ITM104", "name": "Naan", "quantity": 2, "unit_price": 45.00 }
    ],
    "quoted_amount": 405.50,
    "payment_mode_selected": "pay_on_delivery",
    "promised_delivery_time": "2026-03-15T12:05:00+05:30"
  }
}
```

---

### 2. order_confirmed

**Captures:** The restaurant has accepted the order and given a preparation estimate.

| Field | Type | Why it's needed |
|---|---|---|
| `prep_estimate_minutes` | integer | The restaurant's estimate of how long the food will take to prepare. |

```json
{
  "event_id": "evt_2b8d4f01-3a5c-4d9b-8e72-1f6c8b3d5a22",
  "event_type": "order_confirmed",
  "event_time": "2026-03-15T11:05:00+05:30",
  "ingestion_time": null,
  "order_id": "ORDCH001",
  "payload": {
    "prep_estimate_minutes": 20
  }
}
```

---

### 3. rider_assigned

**Captures:** The system has assigned a nearby available rider after the restaurant confirmed the order.

| Field | Type | Why it's needed |
|---|---|---|
| `rider_id` | string | Which rider was assigned. Joins to `dim_rider`. Needed for rider utilization. |
| `distance_to_restaurant_km` | decimal | How far the rider was from the restaurant when assigned. |
| `estimated_arrival_at_restaurant` | timestamp | When the system expects the rider to reach the restaurant. The estimate is made at assignment, so it belongs to this event. |

```json
{
  "event_id": "evt_3c9e5a12-4b6d-4e0c-9f83-2a7d9c4e6b33",
  "event_type": "rider_assigned",
  "event_time": "2026-03-15T11:15:00+05:30",
  "ingestion_time": null,
  "order_id": "ORDCH001",
  "payload": {
    "rider_id": "RIDE005",
    "distance_to_restaurant_km": 1.5,
    "estimated_arrival_at_restaurant": "2026-03-15T11:30:00+05:30"
  }
}
```

---

### 4. rider_reached_restaurant

**Captures:** The rider has arrived at the restaurant.

**Payload:** empty. The only new fact is *that* the rider arrived, and *when* — which is the
envelope's `event_time`.

This event exists for the restaurant-side wait KPI:
`picked_up.event_time − rider_reached_restaurant.event_time`

```json
{
  "event_id": "evt_4d0f6b23-5c7e-4f1d-8a94-3b8e0d5f7c44",
  "event_type": "rider_reached_restaurant",
  "event_time": "2026-03-15T11:40:00+05:30",
  "ingestion_time": null,
  "order_id": "ORDCH001",
  "payload": {}
}
```

---

### 5. picked_up

**Captures:** The rider has picked up the food from the restaurant.

**Payload:** empty. The time of pickup is the envelope's `event_time`.

```json
{
  "event_id": "evt_5e1a7c34-6d8f-4a2e-9b05-4c9f1e6a8d55",
  "event_type": "picked_up",
  "event_time": "2026-03-15T11:50:00+05:30",
  "ingestion_time": null,
  "order_id": "ORDCH001",
  "payload": {}
}
```

---

### 6. delivered

**Captures:** The rider has handed the food to the customer.

**Payload:** empty. The delivery time is the envelope's `event_time`, and the promised time
is already in `order_created`.

```json
{
  "event_id": "evt_6f2b8d45-7e9a-4b3f-8c16-5d0a2f7b9e66",
  "event_type": "delivered",
  "event_time": "2026-03-15T12:15:00+05:30",
  "ingestion_time": null,
  "order_id": "ORDCH001",
  "payload": {}
}
```

---

### 7. payment_event

**Captures:** A payment attempt was made for the order — successful or failed.

**One order can produce more than one `payment_event`.** Each attempt is its own event,
distinguished by `attempt_number`. See `docs/decisions/001-event-granularity.md`.

| Field | Type | Why it's needed |
|---|---|---|
| `payment_id` | string | The payment gateway's reference for this attempt. |
| `attempt_number` | integer | Which attempt this is for the order (1, 2, 3...). |
| `payment_method` | enum | `card`, `upi`, `wallet`, `cash`. |
| `status` | enum | `success` or `failed`. Needed for the payment success/failure rate KPI. |
| `failure_reason` | enum \| null | `card_declined`, `insufficient_balance`, `gateway_timeout`. `null` when successful. |
| `discount_code` | string \| null | The coupon applied, e.g. `FIRST50`. `null` if none. |
| `discount_amount` | decimal | Discount given. `0.00` if none. |
| `delivery_fee` | decimal | Charge for the delivery. |
| `platform_fee` | decimal | Charge for using the platform. |
| `gst_amount` | decimal | Tax charged. |
| `total_amount` | decimal | The final amount charged in this attempt. Compared with `quoted_amount` from `order_created`. |

**Attempt 1 — failed:**

```json
{
  "event_id": "evt_7a3c9e56-8f0b-4c4a-9d27-6e1b3a8c0f77",
  "event_type": "payment_event",
  "event_time": "2026-03-15T12:16:00+05:30",
  "ingestion_time": null,
  "order_id": "ORDCH001",
  "payload": {
    "payment_id": "PAYON001",
    "attempt_number": 1,
    "payment_method": "card",
    "status": "failed",
    "failure_reason": "card_declined",
    "discount_code": "FIRST50",
    "discount_amount": 50.00,
    "delivery_fee": 20.00,
    "platform_fee": 7.50,
    "gst_amount": 18.00,
    "total_amount": 405.50
  }
}
```

**Attempt 2 — successful:**

```json
{
  "event_id": "evt_8b4d0f67-9a1c-4d5b-8e38-7f2c4b9d1a88",
  "event_type": "payment_event",
  "event_time": "2026-03-15T12:18:00+05:30",
  "ingestion_time": null,
  "order_id": "ORDCH001",
  "payload": {
    "payment_id": "PAYON002",
    "attempt_number": 2,
    "payment_method": "upi",
    "status": "success",
    "failure_reason": null,
    "discount_code": "FIRST50",
    "discount_amount": 50.00,
    "delivery_fee": 20.00,
    "platform_fee": 7.50,
    "gst_amount": 18.00,
    "total_amount": 405.50
  }
}
```

---

### 8. order_cancelled

**Captures:** The order was cancelled by the customer, the restaurant, the rider, or the
system. Cancellation is only allowed before the food is picked up.

| Field | Type | Why it's needed |
|---|---|---|
| `cancelled_by` | enum | **Who** cancelled: `customer`, `restaurant`, `rider`, `system`. |
| `cancelled_at_stage` | enum | **When in the lifecycle**: `before_confirmation`, `after_confirmation`, `after_rider_assigned`, `after_rider_reached`. Needed for Core question #6. |
| `reason` | enum | **Why**: `customer_changed_mind`, `item_out_of_stock`, `restaurant_closed`, `no_rider_available`, `rider_unavailable`, `payment_failed`. |
| `penalty_amount` | decimal \| null | Any penalty charged for the cancellation. `null` if none. |

This example uses a different order (`ORDCH002`), since `ORDCH001` was delivered.

```json
{
  "event_id": "evt_9c5e1a78-0b2d-4e6c-9f49-8a3d5c0e2b99",
  "event_type": "order_cancelled",
  "event_time": "2026-03-15T13:12:00+05:30",
  "ingestion_time": null,
  "order_id": "ORDCH002",
  "payload": {
    "cancelled_by": "restaurant",
    "cancelled_at_stage": "after_rider_assigned",
    "reason": "item_out_of_stock",
    "penalty_amount": null
  }
}
```

---

## Known Considerations

- **Incomplete order stories.** If an event never arrives (for example, the rider's phone
  dies and `rider_reached_restaurant` is lost), any KPI that subtracts its time will get
  `NULL`. Orders with missing events should be excluded from that specific KPI, and the
  number excluded should be tracked — if it is large, the KPI is not trustworthy.
- **Out-of-order arrival is normal.** Events may reach ingestion in any order. The real
  order is always told by `event_time`, never by arrival order.
- **`estimated_arrival_at_restaurant` and `distance_to_restaurant_km`** are not used by a
  Core KPI yet. Keep them only if a KPI is added for them; otherwise remove.
- **Parked — cash change.** When a customer pays cash and the rider covers the bill or
  gives change (e.g. the customer pays ₹10 extra), the amount handed over differs from
  `total_amount`. This is cash-handling detail and is not modelled in v1.
- **Open question — customer unavailable at the door.** This happens after pickup, but cancellation
  is only allowed before pickup. This case is not yet modelled. Decide whether it needs its own
  outcome (e.g. a failed-delivery event) or is out of scope for v1.
  
- **Open question — customer unavailable at the door.** This happens *after* pickup, but
  cancellation is only allowed *before* pickup. This case is not yet modelled. Decide
  whether it needs its own outcome (e.g. a failed-delivery event) or is out of scope for v1.
