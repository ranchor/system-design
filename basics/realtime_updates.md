# Real-Time Updates

> **TL;DR** — Pushing fresh data to clients has two halves. **Client ↔ server transport**: **short polling** (simple, wasteful), **long polling** (near-real-time over plain HTTP), **SSE** (one-way server push
> over HTTP, auto-reconnect), **WebSockets** (full-duplex, stateful). **Server ↔ server fan-out**: how an event reaches the *specific* server holding the user's connection: **pub/sub** (Redis, Kafka, NATS),
> a **connection registry** (user → server), or consistent-hash routing. Pick the simplest transport that meets the latency and direction needs. See [Long polling, WebSockets, SSE, webhooks](lp_ws_sse_wh.md), [Chat](../problems/chat_system.md), [Notifications](../problems/notification_system.md).

## Transport choices
| | Short polling | Long polling | SSE | WebSocket |
|---|---|---|---|---|
| Direction | Client pull | Client pull (held) | Server → client | Bidirectional |
| Latency | Poll interval | Low | Low | Lowest |
| Connection | New per request | Re-established per event | One persistent HTTP | One persistent TCP upgrade |
| Overhead | High (headers, empty responses) | Medium | Low | Low per message |
| Proxy/LB friendliness | Excellent | Good | Good (HTTP/2 helps) | Needs WS-aware LB, sticky idle timeouts |
| Auto-reconnect/resume | Trivial | Manual | Built-in with `Last-Event-ID` | Manual (heartbeats, backoff) |
| Use for | Low-frequency status, simple | Legacy/restrictive networks | Feeds, dashboards, notifications, LLM token streaming | Chat, games, collaboration, trading |
Mobile push (APNs/FCM) covers the app-in-background case.

## Architecture: reaching the right connection
```
Producer ─▶ Pub/Sub or Kafka topic ─▶ Gateway/edge servers (hold WS/SSE connections) ─▶ clients
                                   └─ registry: user_id → gateway_id (Redis) for targeted delivery
```
- **Stateful connection tier**: millions of idle connections per fleet; each server ~10-100k+ connections (memory, file descriptors); scale horizontally ([Load balancing](load_balancing.md)).
- **Fan-out options**: (a) all gateways subscribe to topics/channels they have clients for (Redis pub/sub, NATS); (b) registry lookup then direct send; (c) shard users to gateways by consistent hash ([Consistent hashing](consistent_hashing.md)).
- **Offline users**: persist messages/events; on reconnect, sync from a cursor (`since=seq`) rather than relying on the push ([Chat](../problems/chat_system.md)).
- **Ordering & dedupe**: per-channel sequence numbers; clients ack and ignore duplicates ([Delivery semantics](delivery_semantics.md)).

## Operational concerns
- **Heartbeats/pings** to detect dead connections; client **exponential backoff with jitter** on reconnect to avoid reconnect storms after a deploy/outage ([Backpressure](backpressure_load_shedding.md)).
- **Graceful deploys**: drain connections gradually; tell clients to reconnect elsewhere.
- **Auth**: authenticate on connect, re-validate tokens periodically; per-message authorization for channels.
- **Backpressure**: slow clients can balloon server buffers → drop or disconnect; bounded outbound queues; coalesce updates (send latest state only).
- **Scaling LB**: L4 for WS passthrough; idle timeouts > heartbeat interval; sticky not required if state is externalized.
- **Presence**: heartbeat TTL keys in Redis; fan-out presence changes sparingly ([Nearby friends](../problems/nearby_friends.md)).

## Choosing
1. Updates every few seconds+, tolerance of seconds? → polling (with ETag/`304`).
2. Server → client only, simple? → **SSE**.
3. Chatty bidirectional or sub-100 ms? → **WebSocket**.
4. Server-to-server callbacks → **webhooks** (signed, retried, idempotent) ([LP/WS/SSE/WH](lp_ws_sse_wh.md)).
5. Mobile background → push notifications.

## Pitfalls
- WebSockets when SSE would do; per-connection in-memory state (can't scale or fail over); reconnect storms; no resume cursor → missed events; unbounded client buffers; sending full objects instead of deltas; broadcasting to everyone instead of subscribed channels.

## Interview follow-ups
- *"User A sends a message to B; how does it reach B's connection?"* Registry or pub/sub routes to B's gateway; offline → store + push notification + sync on reconnect.
- *"How many WebSocket servers?"* Concurrent users ÷ connections/server (memory-bound) + headroom.
- *"What happens on server crash?"* Clients reconnect (jittered), resume from last seq.
- *"Polling vs SSE?"* SSE avoids repeated requests and gives low latency; polling is simpler for infrequent updates.

# References
* [Hello Interview: Real-time updates](https://www.hellointerview.com/learn/system-design/patterns/realtime-updates) · [MDN: Server-sent events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events), [WebSocket RFC 6455](https://datatracker.ietf.org/doc/html/rfc6455)
