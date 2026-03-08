# Notification Service Design

## 1. Introduction

### Purpose

A notification service delivers multi-channel notifications (push, email, SMS, in-app) from a unified API. It must support routing, templates, batching, retries, and user preferences. This document covers backend architecture for a production notification service.

### Overview

The architecture includes an API for sending, a router for channel selection, channel-specific adapters (FCM, SendGrid, Twilio), a queue for async delivery, template engine, preference service, and analytics. Key patterns: async, idempotency, and graceful degradation.

---

## 2. Requirements

### Functional Requirements

- Send via push, email, SMS, in-app
- Template-based content
- User preference (opt-in/out per channel)
- Batching (digest emails)
- Scheduled delivery
- Delivery status and analytics
- Rate limiting per user/channel

### Non-Functional Requirements

- **Reliability:** Retry with backoff; DLQ for permanent failures
- **Latency:** < 5s for real-time; batch can be delayed
- **Throughput:** Millions per day
- **Scalability:** Horizontal; queue-based
- **Observability:** Delivery rate, latency, errors

---

## 3. High-Level Architecture

### Components

1. **Notification API** — Accept send requests; validate; enqueue
2. **Router** — Select channel(s) based on preference, content type
3. **Template Engine** — Resolve template + variables
4. **Queue** — Kafka/SQS; partition by channel or priority
5. **Channel Workers** — FCM, SendGrid, Twilio adapters
6. **Preference Service** — User opt-in/out; quiet hours
7. **Analytics** — Delivery, open, click tracking

### Communication Flow

- Send: API → Router → Template → Queue
- Worker: Consume → Adapter → FCM/SendGrid/Twilio
- On success: Log; update analytics
- On failure: Retry; DLQ if max retries exceeded

---

## 4. Architecture Diagram

```
                    ┌─────────────────┐
                    │   Event / API   │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ Notification    │
                    │ API             │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ Router +        │
                    │ Template Engine │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ Message Queue   │
                    └────────┬────────┘
                             │
           ┌─────────────────┼─────────────────┐
           │                 │                 │
    ┌──────▼──────┐   ┌──────▼──────┐   ┌──────▼──────┐
    │ Push        │   │ Email       │   │ SMS         │
    │ Worker      │   │ Worker      │   │ Worker      │
    │ (FCM/APNs)  │   │ (SendGrid)  │   │ (Twilio)    │
    └─────────────┘   └─────────────┘   └─────────────┘
```

---

## 5. Key Components

### Notification API

- Input: user_id, template_id, variables, channel_override
- Validates; checks preferences
- Resolves template
- Publishes to queue
- Returns notification_id for tracking

### Router

- Default: Push for mobile; Email for transactional
- Preference: User's opted-in channels
- Fallback: If push fails, email
- Batching: Digest for non-urgent email

### Template Engine

- Template: "Hi {{name}}, your order {{order_id}} shipped"
- Variables: name, order_id
- Localization: Template per locale
- Store in DB or file

### Channel Adapters

- FCM: Batch API; handle token refresh
- SendGrid: SMTP or API; track bounces
- Twilio: SMS API; handle opt-out
- In-app: Write to DB; poll or WebSocket

### Retry & DLQ

- Retry: 3-5 attempts; exponential backoff
- DLQ: Permanent failures (invalid email, unsubscribed)
- Alert on DLQ depth

---

## 6. Database / Storage Design

### notifications

```
notification_id, user_id, channel, template_id, status, sent_at, 
delivered_at, error_message
```

### preferences

```
user_id, channel, enabled, quiet_hours_start, quiet_hours_end
```

### templates

```
template_id, channel, subject, body, variables (json), locale
```

---

## 7. Scaling Strategy

| Strategy | Implementation |
|----------|----------------|
| **Queue** | Kafka; partition by channel; consumer groups |
| **Workers** | Horizontal scaling per channel |
| **Batching** | Email: batch 100; Push: FCM batch 500 |
| **Rate limiting** | Per-user; token bucket; Redis |
| **Circuit breaker** | Stop sending to channel on high error rate |

---

## 8. Technologies Used

| Component | Technologies |
|-----------|--------------|
| **Push** | FCM, APNs |
| **Email** | SendGrid, SES, Postmark |
| **SMS** | Twilio, SNS |
| **Queue** | Kafka, SQS |
| **Database** | PostgreSQL, Redis |
| **Managed** | OneSignal, Customer.io, Braze |

---

## 9. Challenges & Solutions

| Challenge | Solution |
|-----------|----------|
| **Channel failure** | Retry; fallback to alternate channel |
| **User fatigue** | Frequency caps; smart batching; preference center |
| **Deliverability** | Email: SPF/DKIM; warm-up; bounce handling |
| **Latency** | Queue for non-urgent; direct for urgent |
| **Unsubscribe sync** | Update preference; propagate to all channels |
| **Template versioning** | Version field; A/B test templates |
