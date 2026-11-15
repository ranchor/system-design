# System Design Interview: Delivery Framework

> **TL;DR** — Use the same skeleton every time: **Requirements → Core entities → API → (Data flow) → High-level design → Deep dives**.
> Spend your time where the interviewer's signal is: clarify scope early, get a working end-to-end design on the board, then go deep on the 2-3 hardest
> parts (scale, consistency, failure). Say trade-offs out loud. A clear structure beats clever tricks. Pair with the
> [requirements checklist](requirements_checklist.md) and [back-of-envelope estimation](back_of_envelope_estimation.md).

## Time budget (45-60 min)
| Phase | Time | Output |
|---|---|---|
| 1. Requirements | ~5 min | Functional (top 3), non-functional with numbers, out-of-scope list |
| 2. Core entities | ~2 min | Nouns that become tables/collections ([Data Modeling](data_modeling.md)) |
| 3. API / system interface | ~5 min | Endpoints per functional requirement ([API Design](api_design.md)) |
| 4. Data flow (optional, infra-heavy problems) | ~3 min | Steps data takes through the system |
| 5. High-level design | ~10-15 min | Boxes and arrows satisfying the functional requirements; schema next to each store |
| 6. Deep dives | ~10-15 min | Non-functional requirements: scale, latency, consistency, failures, hot spots |
| 7. Wrap-up | ~2 min | Summary, bottlenecks, what you'd monitor or do next |

## Step by step
1. **Requirements.** Ask questions; don't assume. Functional = "users can…" (prioritize 3, state what's below the line). Non-functional = scale
   (DAU, QPS, data size), latency, availability, consistency ([CAP / PACELC](cap_theorem.md)), durability, security. Quantify.
2. **Back-of-envelope (only if it changes the design).** Storage, QPS, bandwidth: use it to justify sharding, caching or a queue, not as a ritual.
3. **Core entities.** List the nouns (User, Post, Follow, Like) and sketch relationships; don't enumerate every field yet.
4. **API.** One endpoint per functional requirement (REST by default; mention gRPC/GraphQL only with a reason). Include auth, pagination, idempotency keys for writes.
5. **High-level design.** Start from the client and the simplest thing that works (client → service → database), satisfy each functional requirement in order, and
   draw the schema beside the database. Add components only when a requirement demands them (cache, queue, CDN, search).
6. **Deep dives.** Return to the non-functional requirements: *"We said 100k reads/s, so…"* Typical dives: sharding key, hot keys, cache invalidation, ordering,
   exactly-once effects, failure of a node/region, fan-out strategy, consistency trade-offs.
7. **Close.** Recap decisions, name remaining bottlenecks, and mention monitoring ([Monitoring](monitoring.md)), security and cost.

## Level expectations
- **Mid-level**: a correct, complete design; you drive the first half; interviewer helps in deep dives.
- **Senior**: you drive end to end; breadth + 1-2 real depth areas; quantified trade-offs; alternatives considered.
- **Staff+**: you identify the *hard* part unprompted, reason about failure modes, operability, cost, org/rollout concerns.

## Habits that score
- Think out loud; state assumptions and confirm them. Write the numbers down.
- Choose boring technology by default (Postgres, Redis, Kafka, S3, CDN) and justify exceptions ([Key Technologies](../README.md#part-1-key-technologies)).
- Introduce a component by the **problem it solves**, then name its trade-off.
- Prefer one coherent design to listing five options; compare briefly when a choice is genuinely contested.
- Handle the failure path: "what if this node dies, this message is duplicated, this region is down?"

## Common failure modes
- Diving into a component before the requirements are agreed. Over-engineering (microservices, Kafka, sharding) for a tiny load.
- Never quantifying anything, so no design choice is justified. Ignoring the non-functional requirements after the header.
- Silence while thinking; arguing instead of adapting to interviewer hints; skipping the schema/API entirely.

## Reusable opening script
> "Let me confirm requirements: users can X, Y, Z; out of scope are A and B. For non-functional: ~N users, reads ≫ writes, p99 < 200 ms, availability over strict consistency
> except for payments. Core entities are … The API looks like … Let me draw the simple version first, then we'll scale the hot parts."

# References
* [Hello Interview: Delivery framework](https://www.hellointerview.com/learn/system-design/in-a-hurry/delivery)
* [Back of Envelope Estimation](back_of_envelope_estimation.md), [Requirements checklist](requirements_checklist.md), [problem template](../template.md)
