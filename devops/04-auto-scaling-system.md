# Auto Scaling System Design

## 1. Introduction

### Purpose

Auto scaling adjusts compute resources based on demand. It reduces cost during low traffic and ensures performance during spikes. This document covers Kubernetes HPA, VPA, Cluster Autoscaler, and application-level scaling patterns.

### Overview

The architecture includes metrics collection (Prometheus), HPA for pod scaling, VPA for resource recommendations, Cluster Autoscaler for node scaling, and custom metrics for business-driven scaling. Key principles: scale proactively, avoid over-provisioning, and handle sudden spikes gracefully.

---

## 2. Requirements

### Functional Requirements

- Scale pods based on CPU, memory, custom metrics
- Scale nodes based on pending pods
- Scale to zero (optional; for batch)
- Predictive scaling (optional)
- Cooldown to avoid thrashing
- Min/max bounds
- Notifications on scale events

### Non-Functional Requirements

- **Responsiveness:** Scale within 1-2 minutes
- **Stability:** Avoid oscillation
- **Cost:** Minimize idle resources
- **Reliability:** Don't scale down too aggressively
- **Visibility:** Metrics; scale events

---

## 3. High-Level Architecture

### Components

1. **Metrics Server** — CPU, memory per pod
2. **Prometheus** — Custom metrics; query API
3. **HPA** — Horizontal Pod Autoscaler
4. **VPA** — Vertical Pod Autoscaler (recommendations)
5. **Cluster Autoscaler** — Add/remove nodes
6. **Custom Metrics Adapter** — Prometheus → HPA
7. **KEDA** — Event-driven scaling (optional)

### Communication Flow

- HPA: Query metrics every 15s → Calculate desired replicas → Update Deployment
- Cluster Autoscaler: Check pending pods → Request new node from cloud
- Scale down: Pods evicted → Node empty → CA removes node

---

## 4. Architecture Diagram

```
                    ┌─────────────────┐
                    │   Workload      │
                    │   (Traffic)     │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ Metrics         │
                    │ (CPU, Memory,   │
                    │  Custom)        │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ HPA / VPA       │
                    │ (scale decision)│
                    └────────┬────────┘
                             │
           ┌─────────────────┼─────────────────┐
           │                 │                 │
    ┌──────▼──────┐   ┌──────▼──────┐   ┌──────▼──────┐
    │ Scale Pods  │   │ Scale Nodes │   │ Recommend   │
    │ (replicas)  │   │ (CA)        │   │ Resources   │
    │             │   │             │   │ (VPA)       │
    └─────────────┘   └─────────────┘   └─────────────┘
```

---

## 5. Key Components

### Horizontal Pod Autoscaler (HPA)

- Target: CPU, memory, or custom metric
- Formula: desired = current × (target/current_avg)
- Min/max replicas
- Scale-up: Fast; scale-down: Slower (stabilization)
- Metrics: metrics-server (CPU/mem); Prometheus adapter (custom)

### Vertical Pod Autoscaler (VPA)

- Recommends CPU/memory requests and limits
- Modes: Off, Initial, Recreate, Auto
- Recreate: Evict and recreate with new resources
- Use for tuning; not for rapid scaling

### Cluster Autoscaler

- Adds nodes when pods are pending (scheduling failure)
- Removes nodes when underutilized (configurable threshold)
- Respects: PodDisruptionBudget; daemonsets; local storage
- Cloud provider integration (AWS, GCP, Azure)

### Custom Metrics

- Prometheus: Query rate, queue depth, etc.
- Prometheus Adapter: Exposes to K8s metrics API
- HPA can scale on: requests per second, queue length

### KEDA (Kubernetes Event-driven Autoscaler)

- Scale based on external events: Kafka lag, SQS depth, cron
- Scale to zero when no events
- Good for batch, event processors

---

## 6. Database / Storage Design

- **HPA/VPA:** Kubernetes resources; no separate DB
- **Metrics:** Prometheus TSDB
- **Scale events:** Kubernetes events; or custom logging

---

## 7. Scaling Strategy

| Strategy | Implementation |
|----------|----------------|
| **Scale-up fast** | HPA default; reduce scale-up delay |
| **Scale-down slow** | Stabilization window; prevent thrash |
| **Multiple metrics** | HPA v2: scale on max of CPU, custom |
| **Predictive** | KEDA + cron for known spikes; or ML |
| **Cost** | Spot instances for workers; Cluster Autoscaler |
| **PDB** | Ensure min available during scale-down |

---

## 8. Technologies Used

| Component | Technologies |
|-----------|--------------|
| **HPA** | Kubernetes built-in |
| **VPA** | Kubernetes VPA |
| **Cluster Autoscaler** | Kubernetes CA; cloud provider |
| **Metrics** | metrics-server, Prometheus |
| **Adapter** | Prometheus Adapter |
| **KEDA** | KEDA project |

---

## 9. Challenges & Solutions

| Challenge | Solution |
|-----------|----------|
| **Oscillation** | Increase stabilization window; tune sensitivity |
| **Cold start** | Pre-warm; min replicas; readiness delay |
| **Scale to zero** | KEDA; consider if latency acceptable |
| **Metrics delay** | metrics-server 15s default; adjust if needed |
| **Node scale-down** | PodDisruptionBudget; respect PDB |
| **Cost vs performance** | Right-size; spot for batch; reserved for baseline |
| **Unexpected spike** | Max replicas cap; queue; graceful degradation |
