# System Design Playbook

A comprehensive guide to system design across **Mobile**, **Backend**, **Web**, and **DevOps**. Production-oriented patterns, scaling strategies, and architecture decisions for building large-scale systems.

## Author

**Muhammad Umer Sheraz** — Senior Software Engineer / Technical Lead

- **Portfolio:** [muhammadumersheraz.com](https://muhammadumersheraz.com/)
- **Email:** muhamadumersheraz@gmail.com
- **Phone:** +923076450994
- **LinkedIn:** [linkedin.com/in/muhammad-umer-sheraz](https://www.linkedin.com/in/muhammad-umer-sheraz/)

---

## Purpose

This repository helps engineers understand large-scale system architecture. Whether preparing for technical interviews, designing production systems, or leveling up as a tech lead, these documents provide real-world patterns used at companies like WhatsApp, Instagram, Uber, Netflix, and Stripe.

---

## Table of Contents

### Mobile

| # | Document | Description |
|---|----------|-------------|
| 1 | [WhatsApp Messaging System](mobile/01-whatsapp-system-design.md) | Real-time chat, WebSockets, message queues |
| 2 | [Instagram Mobile Architecture](mobile/02-instagram-mobile-architecture.md) | Feed generation, media, CDN |
| 3 | [Offline-First Architecture](mobile/03-offline-first-architecture.md) | Local DB, sync queue, conflict resolution |
| 4 | [Push Notification System](mobile/04-push-notification-system.md) | FCM, APNs, delivery pipeline |
| 5 | [Deep Linking Architecture](mobile/05-deep-linking-architecture.md) | Universal links, deferred linking, attribution |

### Backend

| # | Document | Description |
|---|----------|-------------|
| 1 | [Real-time Chat System](backend/01-chat-system-design.md) | WebSockets, presence, message storage |
| 2 | [Ride Sharing System](backend/02-ride-sharing-system.md) | Uber-like; location, matching, trips |
| 3 | [Video Streaming System](backend/03-video-streaming-system.md) | TikTok/YouTube; upload, transcoding, CDN |
| 4 | [Payment System Design](backend/04-payment-system-design.md) | Stripe-like; idempotency, webhooks |
| 5 | [Notification Service Design](backend/05-notification-service-design.md) | Multi-channel, queues, retry |

### Web

| # | Document | Description |
|---|----------|-------------|
| 1 | [Large Scale Dashboard](web/01-large-scale-dashboard.md) | Admin dashboards at scale |
| 2 | [Real-time Web App](web/02-real-time-web-app.md) | WebSockets, SSE, live updates |
| 3 | [CDN Architecture](web/03-cdn-architecture.md) | Edge caching, global delivery |
| 4 | [Analytics Dashboard](web/04-analytics-dashboard.md) | Data pipelines, visualization |

### DevOps

| # | Document | Description |
|---|----------|-------------|
| 1 | [CI/CD Pipeline Design](devops/01-ci-cd-pipeline-design.md) | Build, test, deploy automation |
| 2 | [Microservices Deployment](devops/02-microservices-deployment.md) | Service mesh, discovery |
| 3 | [Kubernetes Architecture](devops/03-kubernetes-architecture.md) | Cluster design, ingress |
| 4 | [Auto Scaling System](devops/04-auto-scaling-system.md) | HPA, VPA, cluster autoscaler |
| 5 | [Logging & Monitoring](devops/05-logging-monitoring-system.md) | ELK, Prometheus, Grafana |

### Diagrams

| Document | Description |
|----------|-------------|
| [Architecture Diagrams](diagrams/architecture-diagrams.md) | Central reference for key system diagrams |

---

## License

MIT — Use for learning and reference.
