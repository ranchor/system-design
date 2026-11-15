## Problem Statement
Design a **local delivery service** (like DoorDash, Uber Eats, Instacart, Gopuff): customers browse nearby merchants, place orders, and the system dispatches **couriers** to pick up and deliver within
~30-60 minutes, with live tracking and accurate ETAs.

## Clarification Questions to Interviewer
- Marketplace of many merchants (restaurants) or a single operator with warehouses (dark stores)?
- Order volume (e.g. 1M orders/day, 10k/min peaks at dinner), couriers online, cities/regions?
- Dispatch: nearest courier or optimized batching (multiple orders per courier)? Real-time ETA accuracy target?
- Payments, promotions, ratings in scope? Scheduled orders?

## Requirements
### Functional Requirements
- **Discovery**: search/browse merchants and menus near a location, with availability and delivery-fee/ETA estimates.
- **Order placement**: cart, checkout, payment authorization, merchant acceptance.
- **Dispatch/matching**: assign a courier; reassign on rejection/timeout.
- **Live tracking**: courier location + order status updates to customer, merchant, courier.
- **ETA** estimation; notifications; ratings.
#### Below the line (out of scope)
- Courier payouts/payroll, fraud models, advertising, route-planning internals, support tooling.

### Non-Functional Requirements
- **Low-latency discovery** (<200 ms) with fresh availability; **high availability** at peak (dinner, weather spikes).
- **Consistency for orders and payments** (no double charge/assign); eventual consistency for browse/location.
- **Scalability** for bursty geo-concentrated load; **real-time updates** with seconds of lag.
#### Below the line (out of scope)
- Global single-instance deployment (shard by city/region).

## Back of Envelope Estimations/Capacity Estimation & Constraints
- 1M orders/day ≈ 12/s average, peak ×10-20 ≈ 150-250 orders/s (dinner, per country).
- 200k couriers online pinging location every 4 s → **50k writes/s** location updates (keep in-memory/Redis, don't hit the DB each time) ([Nearby friends](nearby_friends.md)).
- 50M customers browsing: search/discovery is read-heavy, mostly cacheable per geocell ([Scaling reads](../basics/scaling_reads.md)).
- Tracking fan-out: ~300k active orders × 1 update/4 s to customers ≈ 75k msgs/s ([Real-time updates](../basics/realtime_updates.md)).

## High-level API design
```
GET  /v1/merchants?lat=&lng=&cuisine=&q=          → ranked list with ETA/fee/availability
GET  /v1/merchants/{id}/menu
POST /v1/orders {merchantId, items[], address, paymentMethod, idempotencyKey} → {orderId, status}
GET  /v1/orders/{id}   ·  WS/SSE /v1/orders/{id}/track   (courier location, status)
POST /v1/merchant/orders/{id}/accept|ready|reject        (merchant app)
POST /v1/courier/location {lat,lng,ts}  ·  POST /v1/courier/offers/{id}/accept|decline
```

## Data Model
- `merchants(id, geo, hours, status, prep_time_stats)`; `menu_items(merchant_id, item_id, price, available)`.
- `orders(id, customer_id, merchant_id, items, total, status, courier_id, created_at, version)`; **state machine**: `CREATED → PAID → MERCHANT_ACCEPTED → PREPARING → READY → PICKED_UP → DELIVERED / CANCELLED`.
- `couriers(id, status, vehicle)` and **latest location** in an in-memory geo index (Redis/H3 cells) ([S2/H3](../basics/s2_h3.md)).
- `offers(order_id, courier_id, expires_at, status)`; `payments(order_id, auth_id, status, idempotency_key)`.
- Search index of merchants/items (Elasticsearch) with geo filters ([Elasticsearch](../basics/elasticsearch.md)). Order data in a relational/sharded store partitioned by customer or region ([Data modeling](../basics/data_modeling.md)).

## High Level System Design
```
Customer app ─▶ API GW ─▶ Discovery svc ─▶ Search index (geo) + cache      (read path)
              ─▶ Order svc ─▶ DB (state machine) ─▶ events (Kafka) ─┬▶ Payment svc (auth/capture)
                                                                    ├▶ Merchant notifier (tablet/app)
                                                                    ├▶ Dispatch svc ─▶ geo index of couriers ─▶ offer to courier
                                                                    └▶ Notification / tracking svc ─▶ WS/SSE to apps
Courier app ─location─▶ Location svc ─▶ Redis geo/H3 + Kafka ─▶ ETA svc / tracking
```
- **Order flow as a saga**: create order → authorize payment → merchant accepts → dispatch → pickup → deliver → capture payment; compensations: cancel/refund, re-dispatch ([Sagas](../basics/multi_step_processes.md), [Reliable messaging](../basics/reliable_messaging.md)).
- **Discovery**: geo-index (geohash/H3 cell + filters) → rank by distance, ETA, rating, merchant load; cache popular cells; availability updates via CDC/events ([Geohash](../basics/geohash_quadtree.md)).
- **Dispatch**: when the order nears ready (merchant prep time + courier travel time), find candidate couriers in nearby cells → score (distance to pickup, current load, rating, ETA, fairness) → send **offer** with a short TTL (10-30 s) → on decline/timeout go to the next; **at most one active offer per order**, one active order per courier (batching optional) ([Uber](uber.md), [Contention](../basics/contention.md)).
- **Tracking**: couriers publish locations (throttled, batched); tracking service fans out to the order's subscribers via WebSocket/SSE; also store breadcrumbs for audit/ETAs ([Real-time updates](../basics/realtime_updates.md)).
- **ETA**: prep time (merchant history + current queue) + courier-to-merchant + merchant-to-customer; ML model plus routing engine (traffic aware) ([Google Maps](google_maps.md)).

## Deep Dive
### Matching quality vs latency
Greedy nearest-courier is simple; **batched matching** (every few seconds, solve an assignment problem via Hungarian/min-cost flow or heuristics) improves global efficiency and wait times. Trade-off: slight added latency vs fewer total delivery minutes. Support **stacked orders** (second pickup en route) with detour limits.

### Preventing double assignment & duplicate orders
`orders.version` optimistic locking and conditional transition `READY → ASSIGNED(courier)`; **idempotency key** at checkout; courier availability CAS (`status=AVAILABLE → OFFERED`) with offer TTL ([Idempotency](../basics/idempotency.md)).

### Hot regions and spikes
Shard by **city/region/geocell** (orders, dispatch, location indexes are local); cache menus per merchant; queue-based load leveling for order intake; surge controls (pricing, merchant throttling, delivery-radius shrinking) ([Backpressure](../basics/backpressure_load_shedding.md), [Hot keys](../basics/hot_keys.md)).

### Merchant integration
Order push to tablet/POS via WebSocket with **ack + retry + fallback to SMS/phone**; handle merchant rejects/timeouts by auto-cancel/reassign; inventory/availability toggles propagate to discovery quickly.

### Payments
Authorize at checkout, capture at delivery (or merchant accept), adjust for changes; idempotent calls and reconciliation ([Payment system](payment_gateway.md)).

### Reliability
Dispatch and tracking are best-effort real-time but orders are durable: persist state transitions before notifying; replay from Kafka; multi-AZ; degrade gracefully (stale ETAs, coarser tracking) rather than failing checkout ([Resilience](../basics/resilience_patterns.md)).

### Fraud/ops (brief)
Rate limits, address validation, courier GPS spoofing detection, proof of delivery photos.

### Pitfalls
- Writing every GPS ping to the main DB; dispatching too early/late relative to food prep; offering one order to many couriers simultaneously; no timeout on offers; single global dispatcher; ETA ignoring merchant prep time.

## References
* [DoorDash Engineering: Dispatch / ETA / Geo-sharding](https://careersatdoordash.com/engineering-blog/), [Uber Engineering: H3 and dispatch](https://www.uber.com/blog/h3/)
* [Hello Interview: Design Uber / DoorDash](https://www.hellointerview.com/learn/system-design/problem-breakdowns/uber) · [Uber](uber.md) · [Proximity service](proximity_service.md)
