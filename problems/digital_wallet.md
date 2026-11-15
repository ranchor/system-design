## Problem Statement
Design a **digital wallet** (like PayPal balance, Venmo, Cash App, Paytm): users hold a stored balance, **top up** from bank/cards, **send money** to other users, **pay merchants**, and **withdraw**,
with exact balances, no double-spend, full auditability, and compliance. Related: [Payment system](payment_gateway.md), [Ledger](distributed_ledger.md), [Payment webhook](payment_webhook.md).

## Clarification Questions to Interviewer
- Scope: peer-to-peer transfers, merchant payments, top-up/withdraw via external rails (cards/ACH), multi-currency?
- Scale: users (100M), transfers/day (100M), peak TPS (thousands)? Hot accounts (merchants, platform fee account)?
- Consistency: strict correctness for money (yes). Latency target (<500 ms for transfers)?
- Compliance: KYC/AML, limits, fraud checks, regional regulation, audit retention?
- Reversals/refunds and disputes?

## Requirements
### Functional Requirements
- **Account/wallet** per user and currency; view **balance** and **transaction history**.
- **Transfer** between wallets (P2P/merchant) atomically; **top-up** (funds in) and **withdraw** (funds out) through external providers.
- **Idempotent** requests; **refund/reversal**; **limits** and holds (pending, authorized).
- Statements, notifications, receipts.
#### Below the line (out of scope)
- FX trading, lending, loyalty/rewards, card issuing, tax reporting.

### Non-Functional Requirements
- **Correctness above all**: no lost or duplicated money; balances never negative (unless overdraft allowed); **ACID** for transfers.
- **Auditability**: immutable ledger, reconstructible balances, reconciliation with external providers.
- **High availability** and durability (sync replication, no data loss, RPO 0 for ledger).
- **Security & compliance**: encryption, fraud/AML checks, access control.
- **Performance**: low latency, scalable to peak spikes; hot accounts handled.
#### Below the line (out of scope)
- Offline payments; cross-border settlement specifics.

## Back of Envelope Estimations/Capacity Estimation & Constraints
- 100M transfers/day ≈ 1,160 TPS avg; peak ×5-10 ≈ 6-12k TPS. Each transfer = 2 ledger entries + balance updates + events ≈ 5-10 writes → ~60-100k writes/s peak across shards.
- Ledger growth: 100M tx × 2 entries × ~200 B = 40 GB/day ≈ 15 TB/year → partition/archive by time ([Data partitioning](../basics/data_partitioning.md)).
- Reads (balance/history) ≫ writes: cache balances carefully (only as display, not for authorization) ([Scaling reads](../basics/scaling_reads.md)).

## High-level API design
```
POST /v1/wallets/{id}/transfers {toWalletId, amount, currency, idempotencyKey, memo}   → {transferId, status}
POST /v1/wallets/{id}/topups {source, amount, idempotencyKey}     POST /v1/wallets/{id}/withdrawals {destination, amount, idempotencyKey}
GET  /v1/wallets/{id}/balance          GET /v1/wallets/{id}/transactions?cursor=
POST /v1/transfers/{id}/reverse        Webhooks: /v1/provider/events (top-up/withdraw settlement)
```
Amounts as **integers in minor units** (cents) with currency; never floats ([API design](../basics/api_design.md)).

## Data Model
- `accounts(id, owner, currency, status, balance_cached, version, limits)`.
- **Double-entry ledger** (append-only): `ledger_entries(entry_id, transfer_id, account_id, direction DEBIT|CREDIT, amount, currency, created_at)`; every transfer's entries sum to zero; balance = Σ credits − Σ debits.
- `transfers(id, idempotency_key UNIQUE, from, to, amount, status[PENDING|COMPLETED|FAILED|REVERSED], created_at)`.
- `external_transactions(id, provider, provider_ref, status, wallet_id)` for top-ups/withdrawals; `outbox` for events ([Reliable messaging](../basics/reliable_messaging.md)).
- Relational/NewSQL with strong consistency (Postgres/Aurora, Spanner/Cockroach), sharded by account id ([Postgres & MySQL](../basics/postgres_mysql.md), [Data modeling](../basics/data_modeling.md)).

## High Level System Design
```
Client ─▶ API GW (authN/Z, rate limit) ─▶ Wallet service ─▶ Ledger DB (ACID) ──outbox/CDC──▶ Kafka ─▶ notifications, analytics, risk
                                  │ idempotency check (transfers.idempotency_key)
                                  ├▶ Risk/fraud + limits (sync rules, async ML)
                                  └▶ Payment provider adapters (cards/ACH) ◀── webhooks (signed, idempotent)
Reconciliation job: ledger ⇄ provider statements ⇄ bank (daily)
```
- **Intra-wallet transfer (same shard)**: single DB transaction: insert `transfers` row (unique idempotency key), insert debit+credit entries, update both balances with `UPDATE … SET balance = balance - ? WHERE id=? AND balance >= ?` (affected rows = 1 else insufficient funds); commit → emit event via outbox. Lock accounts in a **consistent order** (lower id first) to avoid deadlocks ([Contention](../basics/contention.md), [Transactions](../basics/transactions.md)).
- **Cross-shard transfer**: options: (a) **saga** with a *pending/hold* state: debit sender into a **transit/escrow account** → credit receiver → finalize; compensate on failure; (b) 2PC across shards (blocking; rarely preferred); (c) NewSQL distributed transaction ([Sagas](../basics/multi_step_processes.md), [Distributed transactions](../basics/distributed_transactions.md), [Aurora DSQL](../basics/aurora_dsql.md)).
- **Top-up**: create `PENDING` external transaction → call provider (idempotent) → on provider webhook confirm, **credit** wallet ledger atomically; handle failed/late/duplicate webhooks ([Payment webhook](payment_webhook.md)).
- **Withdraw**: **reserve** funds (debit to a pending-withdrawal account) first, then send to the bank; on failure, reverse the hold.
- **Balance reads**: derive from the account row (maintained transactionally) and periodically verify against ledger sums; cache only for display with short TTL.

## Deep Dive
### Exactly-once money movement
Client **idempotency key** (unique constraint) → retries return the original result; each step is conditional on state (`WHERE status='PENDING'`); outbox ensures an event is emitted iff the transaction commits; consumers are idempotent ([Idempotency](../basics/idempotency.md), [Delivery semantics](../basics/delivery_semantics.md)).

### Double-entry ledger and immutability
Never update/delete entries: corrections are **new reversing entries**. Gives audit trail, reconstructable history, easy reconciliation (sum of all entries = 0). Optionally an event-sourced design with the ledger as the source of truth ([Event sourcing](../basics/event_sourcing_cqrs_cdc.md)). Periodic checkpoints (snapshots) per account for fast balance recomputation.

### Hot accounts
Merchant/fee/platform accounts see enormous write contention on one row. Mitigate: **sub-accounts/sharded balances** (N shards summed on read), batch/aggregate settlements (credit merchant in micro-batches), asynchronous crediting from the ledger log, avoid updating a shared balance in the hot path ([Hot keys](../basics/hot_keys.md)).

### Consistency under failure
Synchronous replication for the ledger (RPO 0), multi-AZ; fencing on leader failover; on timeouts treat outcome as **unknown**: query by idempotency key/transfer id (state machine) before retry; reconciliation fixes mismatches with providers (unmatched/missing/duplicate).

### Fraud, limits and compliance
Pre-transaction checks: velocity limits, device/IP risk, sanctions screening, KYC tier limits; async ML scoring can place transfers in `REVIEW`; immutable audit logs; PII encryption/tokenization; retention policies.

### Reversals/refunds/disputes
Create a **linked reversal transfer** (never mutate original); disputes put funds on hold; partial refunds supported by amounts on reversal entries.

### Security
Strong auth + step-up for sensitive actions; signed requests; webhook signature verification; encryption at rest (KMS), HSMs for keys; least-privilege services; rate limiting ([Rate limiter](rate_limiter.md)).

### Multi-currency (brief)
Wallet per currency; FX as an exchange transfer between two currency accounts through a system FX account with locked rate/quote and expiry.

### Pitfalls
- Floats for money; read-modify-write without conditional updates; caching balances for authorization; non-idempotent webhook handlers; deleting/editing ledger rows; ignoring transfer ordering → deadlocks; no reconciliation job.

## References
* [Stripe: Idempotency](https://stripe.com/blog/idempotency), [Square: Books, an immutable double-entry accounting database](https://developer.squareup.com/blog/books-an-immutable-double-entry-accounting-database-service/)
* Alex Xu, System Design Interview Vol. 2: Digital Wallet · [Hello Interview: Payment system](https://www.hellointerview.com/learn/system-design/problem-breakdowns/payment-system) · [Payment system](payment_gateway.md), [Ledger](distributed_ledger.md)
