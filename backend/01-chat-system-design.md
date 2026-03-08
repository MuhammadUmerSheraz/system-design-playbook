# Real-time Chat System Design

## 1. Introduction

### Purpose

A real-time chat system enables instant messaging between users. It must support text and media, presence (online/offline), typing indicators, group chats, and scale to millions of concurrent connections. This document covers backend architecture for a Slack/Discord-like chat system.

### Overview

The backend centers on WebSocket servers for real-time delivery, a message service for persistence and routing, a message queue for async processing, presence service for online status, and storage for message history. Key patterns: publish-subscribe, fan-out, and eventual consistency for presence.

---

## 2. Requirements

### Functional Requirements

- Send/receive messages (text, media)
- One-on-one and group chats
- Presence (online/offline/last seen)
- Typing indicators
- Message history with pagination
- Search messages
- Read receipts
- Push when offline

### Non-Functional Requirements

- **Latency:** < 200ms for online delivery
- **Availability:** 99.99%
- **Scalability:** Millions of concurrent connections
- **Ordering:** Per-conversation message order
- **Durability:** No message loss

---

## 3. High-Level Architecture

### Components

1. **WebSocket Gateway** — Persistent connections; pub/sub
2. **Chat Service** — Message validation, persistence, routing
3. **Presence Service** — Online status; heartbeat
4. **Message Queue** — Kafka for offline, analytics
5. **Message Store** — Cassandra for history
6. **Media Service** — Upload, CDN
7. **Push Service** — FCM/APNs

### Communication Flow

- Send: Client → Gateway → Chat Service → Store + Queue
- Receive (online): Chat Service → Gateway (pub/sub) → Client
- Receive (offline): Queue → Push Service → Device

---

## 4. Architecture Diagram

```
                    ┌─────────────────┐
                    │   Clients       │
                    └────────┬────────┘
                             │ WebSocket
                    ┌────────▼────────┐
                    │ WebSocket       │
                    │ Gateway         │
                    │ (Redis Pub/Sub) │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │  Chat Service   │
                    └────────┬────────┘
                             │
           ┌─────────────────┼─────────────────┐
           │                 │                 │
    ┌──────▼──────┐   ┌──────▼──────┐   ┌──────▼──────┐
    │ Message     │   │ Message     │   │ Presence    │
    │ Queue       │   │ Store       │   │ Service     │
    │ (Kafka)     │   │ (Cassandra) │   │ (Redis)     │
    └─────────────┘   └─────────────┘   └─────────────┘
```

---

## 5. Key Components

### WebSocket Gateway

- Maintains connections; maps user → connection(s)
- Publishes to Redis channel per conversation
- Subscribes to channels for users in conversation
- Horizontal scaling; sticky sessions

### Chat Service

- Validates, persists, publishes
- Fetches history; pagination
- Handles group membership

### Presence Service

- Heartbeat from client every 30s
- Redis: user_id → last_seen, status
- TTL for offline detection
- Broadcast presence changes via Pub/Sub

### Message Store (Cassandra)

- Partition: conversation_id
- Clustering: message_id or timestamp
- Wide-column for high write throughput

---

## 6. Database / Storage Design

### messages

```
conversation_id, message_id, sender_id, content, content_type, 
media_url, timestamp, status
PK: (conversation_id, message_id)
```

### conversations

```
conversation_id, type (dm/group), member_ids, created_at
```

### presence (Redis)

```
user:{user_id} = {status, last_seen}  TTL: 60s
```

---

## 7. Scaling Strategy

| Strategy | Implementation |
|----------|----------------|
| **Connection** | Multiple gateway instances; LB with sticky |
| **Pub/Sub** | Redis Cluster for cross-instance messaging |
| **Sharding** | Messages by conversation_id |
| **Caching** | Recent messages in Redis |
| **Queue** | Kafka for offline, analytics |

---

## 8. Technologies Used

| Component | Technologies |
|-----------|--------------|
| **WebSocket** | Socket.io, ws, uWebSockets |
| **Backend** | Node.js, Go, Java |
| **Database** | Cassandra, Redis |
| **Queue** | Kafka |
| **Push** | FCM, APNs |

---

## 9. Challenges & Solutions

| Challenge | Solution |
|-----------|----------|
| **Cross-server delivery** | Redis Pub/Sub or Kafka; subscribe by conversation |
| **Connection churn** | Efficient reconnect; session restore |
| **Presence accuracy** | Heartbeat + TTL; accept eventual consistency |
| **Group fan-out** | Fan-out to Redis channels per member |
| **History at scale** | Cassandra; pagination; archive old to cold storage |
