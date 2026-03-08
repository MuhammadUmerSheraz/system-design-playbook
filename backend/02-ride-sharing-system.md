# Ride Sharing System Design

## 1. Introduction

### Purpose

A ride-sharing system (Uber-like) connects riders with drivers in real-time. It must track driver locations, match riders with nearby drivers, manage ride lifecycle, calculate fares, and process payments. Low latency and high availability are critical.

### Overview

The backend architecture includes a location service with geospatial indexing, a matching service for driver assignment, a trip service for ride lifecycle, a pricing service for fare calculation, maps integration for routing/ETA, and real-time updates via WebSocket or push.

---

## 2. Requirements

### Functional Requirements

- Driver location tracking (GPS every 4-10s)
- Rider requests ride (pickup, dropoff)
- Match rider with nearby available driver
- Real-time ETA and route
- Fare calculation (surge, distance, time)
- Ride lifecycle: request → accept → in-progress → completed
- Payment processing
- Ride history, ratings
- Push notifications

### Non-Functional Requirements

- **Latency:** Match within 1-2 seconds
- **Availability:** 99.99%
- **Accuracy:** Location within 100m; ETA within 10%
- **Scalability:** Millions of concurrent users
- **Real-time:** Sub-second location/status updates

---

## 3. High-Level Architecture

### Components

1. **Location Service** — Ingest GPS; Geohash index; Redis GEO
2. **Matching Service** — Find nearby drivers; dispatch
3. **Trip Service** — Ride lifecycle; state machine
4. **Pricing Service** — Fare calculation; surge
5. **Maps Service** — Geocoding, routing, ETA
6. **Payment Service** — Charges; idempotency
7. **Real-time Service** — WebSocket/push for updates

### Communication Flow

- Location: Driver app → Location Service → Redis GEO + Kafka
- Request: Rider → Matching → Geo query → Dispatch → Driver
- Trip: State transitions → Trip Service → Payment → Notification

---

## 4. Architecture Diagram

```
        ┌────────────┐         ┌────────────┐
        │ Rider App  │         │ Driver App │
        └─────┬──────┘         └─────┬──────┘
              │                      │
              │    GPS Stream        │
              └──────────┬───────────┘
                         │
                ┌────────▼────────┐
                │   API Gateway   │
                └────────┬────────┘
                         │
     ┌───────────────────┼───────────────────┐
     │                   │                   │
┌────▼────┐       ┌──────▼──────┐      ┌────▼────┐
│Location │       │ Matching    │      │ Trip    │
│Service  │       │ Service     │      │ Service │
└────┬────┘       └──────┬──────┘      └────┬────┘
     │                   │                   │
     │            ┌──────▼──────┐            │
     │            │ Maps API    │            │
     │            └─────────────┘            │
     │                   │                   │
     └───────────────────┼───────────────────┘
                         │
                ┌────────▼────────┐
                │ Redis (GEO)     │
                │ PostgreSQL      │
                │ Kafka           │
                └─────────────────┘
```

---

## 5. Key Components

### Location Service

- Receives GPS; validates
- Redis GEOADD for current positions
- GEORADIUS for "drivers within 5km"
- Publishes to Kafka for analytics
- TTL for stale location cleanup

### Matching Service

- On request: GEORADIUS from pickup
- Filter: available, rating, vehicle type
- Rank: distance, ETA, acceptance rate
- Dispatch to one or batch; first accept wins
- Timeout and fallback

### Trip Service

- State: REQUESTED → ACCEPTED → IN_PROGRESS → COMPLETED
- Persists events
- Triggers Pricing, Payment
- Publishes for real-time UI

### Pricing Service

- Base + (distance × rate) + (time × rate)
- Surge by zone (cached; periodic refresh)
- Idempotent for consistency

---

## 6. Database / Storage Design

### trips

```
trip_id, rider_id, driver_id, pickup_lat, pickup_lng, dropoff_lat, dropoff_lng,
status, fare_amount, created_at, completed_at
```

### driver_locations (Redis)

```
GEOADD drivers {lng} {lat} {driver_id}
GEORADIUS drivers {lng} {lat} 5 km
```

### surge_zones

```
zone_id, geohash_prefix, multiplier, updated_at
```

---

## 7. Scaling Strategy

| Strategy | Implementation |
|----------|----------------|
| **Geohashing** | Partition by region; query relevant cells only |
| **Caching** | Redis for live locations; sub-ms reads |
| **Sharding** | Trips by region; locations by geohash |
| **Queue** | Kafka for location stream; async matching |
| **Geo-distribution** | Deploy by region; route to nearest DC |

---

## 8. Technologies Used

| Component | Technologies |
|-----------|--------------|
| **Backend** | Node.js, Go, Java |
| **Database** | PostgreSQL, Cassandra, Redis |
| **Geospatial** | Redis GEO, PostGIS, H3 |
| **Queue** | Kafka |
| **Maps** | Google Maps, Mapbox, HERE |
| **Real-time** | WebSocket, Firebase |

---

## 9. Challenges & Solutions

| Challenge | Solution |
|-----------|----------|
| **Location update volume** | Batch; throttle; Redis GEO |
| **Matching latency** | Pre-indexed GEO; minimal computation |
| **Surge accuracy** | Real-time demand signals; zone-based |
| **Driver acceptance** | Batch dispatch; timeout; next driver |
| **Concurrent requests** | Idempotency keys; optimistic locking |
