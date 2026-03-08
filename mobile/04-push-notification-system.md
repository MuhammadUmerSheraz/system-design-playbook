# Push Notification System

## 1. Introduction

### Purpose

Push notifications re-engage users when the app is closed or in background. The system must reliably deliver to iOS (APNs) and Android (FCM), support high throughput, handle retries, and integrate with user preferences. This document covers the end-to-end push notification architecture.

### Overview

The architecture includes an ingestion API, a message queue for buffering, worker pools for FCM/APNs delivery, a device token registry, and analytics for delivery and engagement. Reliability is achieved through queues, retries, and dead-letter handling.

---

## 2. Requirements

### Functional Requirements

- Send to iOS (APNs) and Android (FCM)
- Text, image, action buttons
- Target by user_id, segment, topic, token
- Schedule future delivery
- Silent push (background sync)
- Deep linking from tap
- Delivery and open tracking

### Non-Functional Requirements

- **Reliability:** At-least-once; retry on failure
- **Latency:** Seconds for real-time use cases
- **Throughput:** Millions per day
- **Scalability:** Handle campaign bursts
- **Compliance:** Respect opt-out; quiet hours

---

## 3. High-Level Architecture

### Components

1. **Notification API** — Validates, resolves tokens, enqueues
2. **Message Queue** — Kafka/SQS; buffer and retry
3. **Worker Pool** — Consumes; sends to FCM/APNs
4. **FCM / APNs** — Platform gateways
5. **Token Registry** — user_id → device tokens
6. **Analytics** — Delivery, open, failure

### Communication Flow

- Event → API → Queue → Worker → FCM/APNs → Device
- On failure: Retry with backoff; invalid token → remove from registry

---

## 4. Architecture Diagram

```
                    ┌─────────────────┐
                    │  Event Source   │
                    │ (chat, order)   │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ Notification    │
                    │ API             │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ Message Queue   │
                    │ (Kafka/SQS)     │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ Worker Pool     │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
       ┌──────▼──────┐ ┌─────▼─────┐ ┌─────▼─────┐
       │ FCM         │ │ APNs      │ │ Token     │
       │ (Android)   │ │ (iOS)     │ │ Registry  │
       └──────┬──────┘ └─────┬─────┘ └───────────┘
              └──────┬───────┘
                     │
              ┌──────▼──────┐
              │ User Device │
              └─────────────┘
```

---

## 5. Key Components

### Notification API

- Validates user preferences, rate limits
- Resolves user_id → tokens
- Builds FCM/APNs payload
- Publishes to queue

### Message Queue

- Buffers; enables retry
- Dead-letter for invalid tokens
- Batching for FCM (500/request)

### FCM / APNs

- FCM: HTTP v1 or legacy
- APNs: HTTP/2; .p8 auth
- Handle: invalid token, unavailable, rate limit

### Retry Mechanism

- Exponential backoff: 1s, 2s, 4s, 8s
- Max 3-5 retries
- InvalidRegistration → remove token; no retry
- Idempotency key for deduplication

---

## 6. Database / Storage Design

### device_tokens

```
user_id, device_token, platform, app_version, created_at
```

### notification_log

```
notification_id, user_id, status, sent_at, opened_at
```

### preferences

```
user_id, channel, enabled, quiet_hours_start, quiet_hours_end
```

---

## 7. Scaling Strategy

| Strategy | Implementation |
|----------|----------------|
| **Queue** | Kafka/SQS; auto-scaling consumers |
| **Batching** | FCM batch API |
| **Connection pooling** | Reuse HTTP/2 to APNs |
| **Circuit breaker** | Stop on high error rate |
| **Rate limiting** | Per-user, global |

---

## 8. Technologies Used

| Component | Technologies |
|-----------|--------------|
| **Push** | FCM, APNs |
| **Queue** | Kafka, SQS, RabbitMQ |
| **Database** | PostgreSQL, Redis |
| **Managed** | OneSignal, Braze, Airship |

---

## 9. Challenges & Solutions

| Challenge | Solution |
|-----------|----------|
| **Token invalidation** | Remove on InvalidRegistration; refresh flow |
| **Delivery guarantee** | Queue + retry; DLQ for permanent failures |
| **Burst traffic** | Queue absorbs; scale workers |
| **APNs connection limits** | Connection pooling; multiple certs |
| **User fatigue** | Frequency caps; smart timing; A/B test |
