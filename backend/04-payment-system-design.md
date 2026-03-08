# Payment System Design

## 1. Introduction

### Purpose

A payment system (Stripe-like) processes charges, refunds, subscriptions, and payouts. It must be highly reliable, support idempotency, handle webhooks for async events, and comply with PCI-DSS. This document covers backend architecture for a production payment system.

### Overview

The architecture includes an API layer with idempotency, a payment service that orchestrates with processors (Stripe, Adyen), a webhook handler for async events, a ledger for financial records, and reconciliation jobs. Key principles: idempotency, idempotency, idempotency.

---

## 2. Requirements

### Functional Requirements

- Process one-time charges (card, wallet)
- Refunds (full, partial)
- Subscriptions (recurring)
- Payouts to sellers
- Multiple payment methods
- Webhooks for async status
- Payment history and receipts
- Dispute handling

### Non-Functional Requirements

- **Reliability:** Zero double charges; at-most-once processing
- **Idempotency:** Same request = same result
- **Auditability:** Full ledger; immutable
- **Compliance:** PCI-DSS; no raw card storage
- **Availability:** 99.99%
- **Latency:** < 3s for charge confirmation

---

## 3. High-Level Architecture

### Components

1. **Payment API** — Idempotency; validation; routing
2. **Payment Service** — Orchestrates charges; calls processor
3. **Processor Adapter** — Stripe, Adyen, etc.
4. **Webhook Handler** — Processes async events
5. **Ledger** — Immutable financial records
6. **Reconciliation** — Match ledger vs processor
7. **Subscription Engine** — Recurring billing

### Communication Flow

- Charge: API (idempotency check) → Payment Service → Processor → Ledger → Response
- Webhook: Processor → Webhook Handler → Verify signature → Update state → Ledger
- Refund: API → Payment Service → Processor → Ledger

---

## 4. Architecture Diagram

```
                    ┌─────────────────┐
                    │   Client        │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ Payment API     │
                    │ (idempotency)   │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ Payment Service │
                    └────────┬────────┘
                             │
           ┌─────────────────┼─────────────────┐
           │                 │                 │
    ┌──────▼──────┐   ┌──────▼──────┐   ┌──────▼──────┐
    │ Processor   │   │ Ledger      │   │ Webhook     │
    │ (Stripe)    │   │ (immutable) │   │ Handler     │
    └──────┬──────┘   └─────────────┘   └──────┬──────┘
           │                                    │
           └──────────── Events ────────────────┘
```

---

## 5. Key Components

### Payment API

- Idempotency-Key header
- Store (idempotency_key → response) for 24h
- Same key → return cached response
- Validate amount, currency, customer

### Payment Service

- Create charge at processor
- On success: Record in ledger; return
- On failure: No ledger entry; return error
- Never duplicate; idempotency at API ensures

### Webhook Handler

- Verify signature (HMAC)
- Idempotent by event_id
- Update payment state
- Append to ledger (credits, refunds)
- Retry on transient failure
- Alert on unknown events

### Ledger

- Double-entry: debit/credit per account
- Immutable; append-only
- Each row: id, account_id, amount, type, reference_id, created_at
- Balance = SUM(amount) per account

### Reconciliation

- Daily: Fetch processor settlements
- Match with ledger
- Flag discrepancies
- Alert on mismatch

---

## 6. Database / Storage Design

### payments

```
payment_id, idempotency_key, customer_id, amount, currency, status,
processor_charge_id, created_at
UNIQUE (idempotency_key)
```

### ledger_entries

```
id, account_id, amount, type (debit/credit), reference_type, reference_id,
created_at
```

### webhook_events

```
event_id, processor, payload_hash, processed_at
UNIQUE (event_id) -- idempotency
```

---

## 7. Scaling Strategy

| Strategy | Implementation |
|----------|----------------|
| **Idempotency** | Redis or DB; 24h TTL |
| **Queue** | Async webhook processing |
| **Sharding** | Ledger by account_id |
| **Read replicas** | History queries from replica |
| **Circuit breaker** | Stop calling processor on high error rate |

---

## 8. Technologies Used

| Component | Technologies |
|-----------|--------------|
| **Processors** | Stripe, Adyen, Braintree |
| **Backend** | Node.js, Go, Java |
| **Database** | PostgreSQL (ACID) |
| **Cache** | Redis (idempotency) |
| **Queue** | Kafka, SQS (webhooks) |

---

## 9. Challenges & Solutions

| Challenge | Solution |
|-----------|----------|
| **Double charge** | Idempotency key; never retry without same key |
| **Webhook duplicate** | Store event_id; deduplicate |
| **Eventual consistency** | Webhook updates state; client polls or subscribes |
| **PCI compliance** | No card storage; tokenize via processor |
| **Reconciliation** | Daily batch; automate; alert on drift |
| **Refund race** | Optimistic locking; idempotent refund by refund_id |
