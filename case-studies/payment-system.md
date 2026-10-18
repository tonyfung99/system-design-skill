# Design a Payment System (Stripe-like)

Source: Hello Interview outline (premium); answers are independent · Patterns: multi-step processes · Difficulty: hard

## Scope
- **FR:** merchants initiate payment requests; customers pay with credit/debit cards; payment status available (and pushed via webhooks — bonus). Below the line: refunds/disputes detail, payouts, FX, subscriptions.
- **NFR:** highly secure (PCI); durable and auditable — no transaction data ever lost; financial integrity despite asynchronous external networks (no double charge, no lost charge); 10k+ TPS.

## Crux
Correctness across an unreliable external boundary (card networks/processors): idempotency, a durable state machine, and reconciliation — not throughput.

## Skeleton
Entities: Merchant, PaymentIntent (amount, currency, status, idempotency key), Transaction/Attempt (each call to the processor), LedgerEntry (double-entry), Webhook/Event. API: `POST /payment_intents {amount, currency, idempotencyKey}`, `POST /payment_intents/{id}/confirm {paymentMethodToken}`, `GET /payment_intents/{id}`.
Client-side tokenization (card data never touches merchant servers; vault isolated in a PCI zone) → Payment service → state machine in Postgres → processor/network gateway → events to Kafka → ledger, webhooks, reconciliation.

## State machine
`created → processing → (requires_action) → succeeded | failed`, with `unknown`/`pending_verification` for timeouts. Transitions are conditional updates; every transition is an append-only event (audit trail).

## Deep dives (typical)

**Security:** tokenization/vault, network segmentation for PCI scope, encryption at rest with KMS/HSM, least privilege, signed webhooks, fraud scoring inline.

**Durability & auditability:** write intent + outbox in one DB transaction before calling the processor; append-only event log in Kafka (replicated, long retention) feeding an immutable **double-entry ledger**; never update balances in place without ledger entries.

**Integrity with async networks:**
- Idempotency keys on our API (merchant retries) and on calls to processors.
- **Timeout ≠ failure:** a processor timeout leaves the charge in an unknown state → mark `pending_verification`, query the processor's status endpoint / wait for webhook, never blindly retry a charge without the same idempotency key.
- **Reconciliation:** daily (and intraday) match internal records against processor settlement files; discrepancies flagged and resolved.
- Saga for multi-step flows (authorize → capture → ledger → notify), compensations (void/refund).

**Scale (10k TPS):** shard by merchant_id or payment_id; Kafka for event fan-out; idempotency table with TTL; it's the correctness that's hard, not the throughput.

**Bonus — webhooks:** at-least-once delivery with retries and exponential backoff, signed payloads, event ids for merchant dedupe, per-merchant queues so one slow endpoint can't block others, DLQ + replay UI.

## Staff+ signals
- Name the timeout/unknown-state problem before being asked; it's the heart of payments.
- Double-entry ledger as source of financial truth; balances derived.
- Exactly-once *effects* = idempotency + reconciliation, not a magic protocol.
- Regulatory/audit: immutability, retention, access logging.

## Traps
Retrying charges on timeout without idempotency; storing raw PANs; mutable balance columns as truth; synchronous webhook delivery.
