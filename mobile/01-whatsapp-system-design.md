# WhatsApp Messaging System Design

## 1. Introduction

### Purpose

WhatsApp is a real-time messaging platform serving billions of users. The system must deliver messages with minimal latency, support offline users, maintain end-to-end encryption, and scale globally. This document outlines the architecture for a WhatsApp-like messaging system.

### Overview

The architecture centers on persistent WebSocket connections, message queues for async delivery, distributed storage for message persistence, and push notifications for offline users. Key design decisions prioritize reliability, low latency, and horizontal scalability.

---

## 2. Requirements

### Functional Requirements

- Send and receive text messages in real-time
- Support media (images, videos, documents)
- One-on-one and group chats
- Read receipts and typing indicators
- Offline message queuing
- Push notifications when app is in background
- Message search and history
- End-to-end encryption (optional scope)

### Non-Functional Requirements

- **Low latency:** < 500ms for online delivery
- **High availability:** 99.99% uptime
- **Scalability:** Millions of concurrent connections
- **Durability:** At-least-once delivery; no message loss
- **Consistency:** Message ordering per conversation

---

## 3. High-Level Architecture

### Components

1. **Mobile Clients** — iOS/Android apps with persistent WebSocket
2. **Connection Manager** — WebSocket gateway; sticky sessions
3. **Chat Service** — Message routing, persistence, business logic
4. **Message Queue** — Kafka for offline delivery, fan-out
5. **Message Storage** — Cassandra for high write throughput
6. **Push Service** — FCM/APNs integration
7. **Media Service** — Upload, storage, CDN delivery

### Communication Flow

- Online: Client → Connection Manager → Chat Service → Recipient's Connection Manager → Client
- Offline: Chat Service → Message Queue → Push Service → FCM/APNs → Device

---

## 4. Architecture Diagram

```
                    ┌─────────────────┐
                    │   Mobile App    │
                    └────────┬────────┘
                             │ WebSocket
                    ┌────────▼────────┐
                    │ Connection      │
                    │ Manager         │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │  Chat Service   │
                    └────────┬────────┘
                             │
           ┌─────────────────┼─────────────────┐
           │                 │                 │
    ┌──────▼──────┐   ┌──────▼──────┐   ┌──────▼──────┐
    │ Message     │   │ Message     │   │ Push        │
    │ Queue       │   │ Storage     │   │ Service     │
    │ (Kafka)     │   │ (Cassandra) │   │             │
    └─────────────┘   └─────────────┘   └─────────────┘
```

---

## 5. Key Components

### Connection Manager

- Maintains WebSocket per device
- Maps user_id → connection(s) for multi-device
- Heartbeat to detect dead connections
- Sticky sessions via load balancer

### Chat Service

- Validates sender, recipient, message format
- Persists to storage
- Publishes to queue for offline delivery
- Retrieves chat history with pagination

### Message Queue (Kafka)

- Decouples sync send from async delivery
- Partition by conversation_id for ordering
- Consumer: Push Service, analytics, other devices

### Message Storage (Cassandra)

- Wide-column store; high write throughput
- Partition key: conversation_id
- Clustering: message_id or timestamp
- TTL for ephemeral data

### Push Notification Service

- Consumes from queue when user offline
- Integrates FCM (Android), APNs (iOS)
- Batches and throttles to avoid spam

---

## 6. Database / Storage Design

### Messages Table (Cassandra)

```
messages (
  conversation_id UUID,    -- Partition key
  message_id UUID,         -- Clustering key
  sender_id UUID,
  content TEXT,
  content_type TEXT,
  media_url TEXT,
  timestamp TIMESTAMP,
  status TEXT
)
PRIMARY KEY (conversation_id, message_id)
```

### Indexes

- User's conversations: Materialized view or secondary index
- Search: Elasticsearch for full-text if needed

---

## 7. Scaling Strategy

| Strategy | Implementation |
|----------|----------------|
| **Sharding** | Partition by conversation_id across Cassandra nodes |
| **Caching** | Redis for active conversations, presence |
| **Connection scaling** | Horizontal scaling of Connection Managers |
| **Message queue** | Kafka partitions; consumer groups |
| **Geo-distribution** | Regional deployment; read from nearest DC |
| **CDN** | Media served via CDN |

---

## 8. Technologies Used

| Component | Technologies |
|-----------|--------------|
| **Mobile** | Swift, Kotlin, React Native |
| **WebSocket** | Socket.io, native WebSocket |
| **Backend** | Node.js, Go, Erlang |
| **Queue** | Apache Kafka, RabbitMQ |
| **Database** | Cassandra, ScyllaDB |
| **Cache** | Redis |
| **Push** | FCM, APNs |
| **Storage** | S3, CloudFront |

---

## 9. Challenges & Solutions

| Challenge | Solution |
|-----------|----------|
| **Connection scaling** | Stateless Connection Managers; sticky LB; Redis for session mapping |
| **Message ordering** | Kafka partition by conversation; single consumer per partition |
| **Offline delivery** | Queue + Push; client fetches on reconnect |
| **Media scalability** | S3 + CDN; async thumbnail generation |
| **Multi-device sync** | Fan-out to all user devices; last-write-wins or CRDT for presence |
| **Encryption** | Client-side E2E; server stores ciphertext only |
