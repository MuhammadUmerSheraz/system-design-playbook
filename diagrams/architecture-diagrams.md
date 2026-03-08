# Architecture Diagrams Reference

Central reference for key system architecture diagrams across Mobile, Backend, Web, and DevOps.

---

## Mobile

### WhatsApp / Chat Flow

```
           Mobile App
               |
           WebSocket
               |
        Connection Manager
               |
           Chat Service
       |         |         |
    Queue     Storage    Push
       |         |         |
    Kafka   Cassandra   FCM/APNs
```

### Instagram Feed

```
           Mobile App
               |
           API Gateway
               |
        Service Layer
       |         |         |
    Feed      Media     Social
   Service   Service    Graph
       |         |
    Redis      CDN
```

### Offline Sync

```
           Mobile App
               |
        Data Layer
               |
    Local DB  |  Sync Queue  |  Conflict Resolver
               |
           Sync Engine
               |
           Sync API
               |
          Server DB
```

### Push Notifications

```
        Event Source
               |
        Notification API
               |
        Message Queue
               |
        Worker Pool
       |         |
    FCM       APNs
               |
        User Device
```

### Deep Linking

```
        User taps link
               |
        Link Server
       |         |
    App      Web/Store
    installed   |
               |
        Attribution
        (fingerprint)
               |
        App opens with
        deep link data
```

---

## Backend

### Real-time Chat

```
           Clients
               |
        WebSocket Gateway
               |
        Redis Pub/Sub
               |
           Chat Service
       |         |         |
    Queue     Storage    Presence
```

### Ride Sharing

```
        Rider  |  Driver
               |
        API Gateway
               |
    Location | Matching | Trip
     Service   Service   Service
               |
        Redis GEO  |  Maps API
               |
        PostgreSQL
```

### Video Streaming

```
        Upload Client
               |
        Upload Service
               |
        Transcode Queue
               |
        FFmpeg Workers
               |
        Origin (S3)
               |
        CDN
               |
        Playback Client
```

### Payment System

```
           Client
               |
        Payment API
        (idempotency)
               |
        Payment Service
       |         |         |
  Processor   Ledger   Webhook
               |
        Stripe/Adyen
```

### Notification Service

```
        Event / API
               |
        Notification API
               |
        Router + Template
               |
        Message Queue
               |
    Push | Email | SMS
    Workers
```

---

## Web

### Large Scale Dashboard

```
           Browser (SPA)
               |
        CDN (static)
               |
        API Gateway / BFF
               |
        Service Layer
    Billing | Analytics | Users
               |
        Cache | Database
```

### Real-time Web App

```
           Web Clients
               |
        Load Balancer
        (sticky)
               |
        WebSocket Servers
               |
        Redis Pub/Sub
               |
        Presence
```

### CDN Architecture

```
           Users (Global)
               |
        DNS (geo-routing)
               |
        Edge Caches
       US | EU | APAC
               |
        Origin (cache miss)
```

### Analytics Pipeline

```
        Event Sources
               |
        Ingestion API
               |
        Kafka / Kinesis
       |         |
    Stream    S3 (raw)
    (Flink)      |
               ETL
               |
        Data Warehouse
               |
        Query API | Viz
```

---

## DevOps

### CI/CD Pipeline

```
        Git Push/PR
               |
        CI Trigger
               |
        Lint → Test → Build
               |
        Docker Build
        → Registry
               |
        Deploy
        (K8s/ECS)
               |
        Health Check
```

### Microservices Deployment

```
           Clients
               |
        API Gateway
               |
        Service Mesh
               |
    Service A | B | C
    (pods)
               |
        DB | Cache | MQ
```

### Kubernetes Cluster

```
        Load Balancer
               |
        Control Plane
    Master 1 | 2 | 3
               |
        Worker Nodes
    Node 1 | 2 | N
    (Pods)
```

### Auto Scaling

```
        Workload
               |
        Metrics
        (CPU, Custom)
               |
        HPA | VPA | CA
               |
    Scale Pods | Nodes
```

### Logging & Monitoring

```
        Applications
               |
        Fluent Bit
               |
        Kafka / Buffer
               |
    Elasticsearch | Loki
               |
    Prometheus | Grafana | Alertmanager
```

---

## Cross-Cutting

### Generic Service Architecture

```
           Mobile App
               |
           API Gateway
               |
           Service Layer
       |         |         |
    Chat      Media      Notification
   Service   Service      Service
       |
   Database Cluster
       |
   CDN / Cache
```

---

*See individual documents in each folder for detailed diagrams and context.*
