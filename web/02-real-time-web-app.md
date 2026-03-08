# Real-time Web App Architecture

## 1. Introduction

### Purpose

Real-time web apps (collaborative editors, live dashboards, chat) require low-latency bidirectional communication. The architecture must support WebSockets at scale, handle reconnection, manage presence, and scale horizontally. This document covers patterns for real-time web applications.

### Overview

The architecture centers on WebSocket servers (or managed services like Ably, PubNub), a pub/sub layer (Redis) for cross-server messaging, presence tracking, and fallbacks (SSE, long-polling) for restrictive environments. Key patterns: connection scaling, sticky sessions, and event-driven updates.

---

## 2. Requirements

### Functional Requirements

- Bidirectional real-time communication
- Presence (who's online)
- Typing indicators
- Room/channel-based messaging
- Reconnection with state sync
- Fallback for restrictive proxies (SSE, long-poll)
- Message history (optional)

### Non-Functional Requirements

- **Latency:** < 100ms for message delivery
- **Availability:** 99.9%
- **Scalability:** Hundreds of thousands of connections
- **Reconnection:** Seamless; resume from last event
- **Ordering:** Per-channel message order

---

## 3. High-Level Architecture

### Components

1. **WebSocket Server** — Persistent connections; event handler
2. **Redis Pub/Sub** — Cross-server message distribution
3. **Presence Service** — Online users; heartbeats
4. **API Gateway** — REST for initial load; WebSocket upgrade
5. **Message Store** — Optional history (Redis, DB)
6. **Load Balancer** — Sticky sessions for WebSocket

### Communication Flow

- Connect: Client → LB (sticky) → WebSocket Server → Subscribe to Redis channel
- Send: Client → Server → Publish to Redis → All servers with subscribers → Clients
- Presence: Heartbeat → Redis; Subscribe to presence channel → Broadcast joins/leaves

---

## 4. Architecture Diagram

```
                    ┌─────────────────┐
                    │   Web Clients   │
                    └────────┬────────┘
                             │ WebSocket
                    ┌────────▼────────┐
                    │ Load Balancer   │
                    │ (sticky)        │
                    └────────┬────────┘
                             │
           ┌─────────────────┼─────────────────┐
           │                 │                 │
    ┌──────▼──────┐   ┌──────▼──────┐   ┌──────▼──────┐
    │ WS Server 1 │   │ WS Server 2 │   │ WS Server N │
    └──────┬──────┘   └──────┬──────┘   └──────┬──────┘
           │                 │                 │
           └─────────────────┼─────────────────┘
                             │
                    ┌────────▼────────┐
                    │ Redis Pub/Sub   │
                    │ (channels)      │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ Presence        │
                    │ (Redis)         │
                    └─────────────────┘
```

---

## 5. Key Components

### WebSocket Server

- Accept connections; map connection → user, channels
- On message: Validate → Publish to Redis channel
- On Redis message: Forward to local subscribers
- Heartbeat/ping-pong for keepalive
- Graceful shutdown: Drain connections

### Redis Pub/Sub

- Channel per room: room:{id}
- Server subscribes to channels for connected clients
- Publish from any server; all subscribers receive
- No persistence; use Redis Streams if history needed

### Presence

- Redis: user:{id} = {status, last_seen} TTL 60s
- Heartbeat every 30s from client
- On join/leave: Publish presence event to channel
- Clients subscribe to presence channel for room

### Reconnection

- Client stores last_event_id
- On reconnect: Request events since last_event_id (server maintains buffer or store)
- Or: Full state sync on reconnect
- Exponential backoff for retry

### Fallback

- SSE: Server → Client only; use when WebSocket blocked
- Long-poll: Client polls; server holds until event or timeout
- Both increase latency vs WebSocket

---

## 6. Database / Storage Design

### Optional Message History (Redis Streams)

```
room:{id} = Stream of messages
XADD room:123 * user user1 msg "hello"
XRANGE room:123 - + COUNT 100
```

### Presence (Redis)

```
presence:{room_id} = SET of user_ids  (or hash with metadata)
TTL per user entry for stale cleanup
```

---

## 7. Scaling Strategy

| Strategy | Implementation |
|----------|----------------|
| **Sticky sessions** | Hash on connection ID or user; same server |
| **Redis Pub/Sub** | Enables cross-server fan-out |
| **Horizontal scaling** | Add WebSocket servers; Redis scales subscribers |
| **Connection limit** | Max per server; scale out when approaching |
| **Geographic** | Regional servers; Redis Cluster for global |
| **Managed services** | Ably, PubNub, Firebase for fully managed |

---

## 8. Technologies Used

| Component | Technologies |
|-----------|--------------|
| **WebSocket** | ws, Socket.io, uWebSockets |
| **Backend** | Node.js, Go, Elixir (Phoenix) |
| **Pub/Sub** | Redis, Kafka |
| **Load Balancer** | NGINX, HAProxy, AWS ALB (sticky) |
| **Managed** | Ably, PubNub, Firebase Realtime DB |

---

## 9. Challenges & Solutions

| Challenge | Solution |
|-----------|----------|
| **Cross-server delivery** | Redis Pub/Sub; subscribe per channel |
| **Connection limits** | Multiple servers; load balance |
| **Reconnection storm** | Backoff; rate limit reconnects |
| **Presence accuracy** | TTL; accept eventual consistency |
| **Message ordering** | Single Redis channel; FIFO per channel |
| **History** | Redis Streams or DB; fetch on reconnect |
| **Proxy/WebSocket block** | Fallback SSE or long-poll |
