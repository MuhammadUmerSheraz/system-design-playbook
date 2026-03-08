# CDN Architecture

## 1. Introduction

### Purpose

A Content Delivery Network (CDN) caches static and dynamic content at edge locations close to users. It reduces latency, offloads origin servers, and improves global performance. This document covers CDN architecture, caching strategies, and integration patterns for web applications.

### Overview

The architecture includes origin servers (source of truth), CDN edge nodes (cache), and cache control via headers. Key concepts: cache keys, TTL, invalidation, and when to use CDN vs origin. CDNs like CloudFront, Cloudflare, Fastly provide global edge networks.

---

## 2. Requirements

### Functional Requirements

- Cache static assets (JS, CSS, images)
- Cache dynamic content (API responses) where appropriate
- Cache media (images, video segments)
- Support cache invalidation
- HTTPS at edge
- Geographic routing (optional)
- DDoS protection (optional)

### Non-Functional Requirements

- **Latency:** < 50ms from edge to user
- **Availability:** 99.99%
- **Scalability:** Absorb traffic spikes
- **Cost:** Minimize origin egress; optimize cache hit ratio
- **Security:** SSL termination; WAF

---

## 3. High-Level Architecture

### Components

1. **Origin** — Application servers; source of truth
2. **CDN Edge** — Cache nodes; serve on hit
3. **DNS** — Route to nearest edge
4. **Origin Shield** — Optional; reduce origin load further
5. **Invalidation** — Purge cache on content update

### Communication Flow

- Miss: User → Edge → Origin → Cache at edge → Response to user
- Hit: User → Edge → Cache → Response (no origin)
- Invalidation: Trigger → Purge keys or paths → Next request = miss → Refresh

---

## 4. Architecture Diagram

```
                    ┌─────────────────┐
                    │   Users         │
                    │   (Global)      │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ DNS             │
                    │ (geo-routing)   │
                    └────────┬────────┘
                             │
           ┌─────────────────┼─────────────────┐
           │                 │                 │
    ┌──────▼──────┐   ┌──────▼──────┐   ┌──────▼──────┐
    │ Edge (US)   │   │ Edge (EU)   │   │ Edge (APAC) │
    │ Cache       │   │ Cache       │   │ Cache       │
    └──────┬──────┘   └──────┬──────┘   └──────┬──────┘
           │                 │                 │
           └─────────────────┼─────────────────┘
                             │ Cache Miss
                    ┌────────▼────────┐
                    │ Origin          │
                    │ (app servers)   │
                    └─────────────────┘
```

---

## 5. Key Components

### Cache Keys

- Default: URL (including query string or not)
- Custom: Include headers (e.g., Accept-Language) in key
- Vary: Cache different variants (e.g., by User-Agent)
- Static: Use content hash in filename (e.g., app.abc123.js) for long TTL

### TTL (Time to Live)

- Static: 1 year (immutable filenames)
- HTML: Short (60s) or no cache (dynamic)
- API: Varies; cache GET with suitable TTL
- Images: Hours to days

### Cache Headers

- **Cache-Control:** max-age=3600, s-maxage=86400, stale-while-revalidate
- **ETag:** Conditional requests; 304 Not Modified
- **Vary:** Cache per Accept-Language, etc.

### Invalidation

- **Purge:** Invalidate by path or key
- **Versioned URLs:** Change URL on deploy; no purge needed
- **Stale-while-revalidate:** Serve stale; refresh in background

### Origin Shield

- Optional tier: Edge → Shield → Origin
- Shield caches for region; reduces origin requests further
- Use for high-traffic, cacheable content

---

## 6. Database / Storage Design

CDN typically doesn't use application DB. Cache metadata:

- **Cache keys** — Defined by URL/path + headers
- **TTL** — Set per path pattern
- **Invalidation log** — Audit of purges (optional)

---

## 7. Scaling Strategy

| Strategy | Implementation |
|----------|----------------|
| **Edge distribution** | Use CDN with many POPs globally |
| **Cache hit ratio** | Long TTL for static; versioned URLs |
| **Origin protection** | CDN absorbs traffic; origin sees only misses |
| **Multi-CDN** | Failover; cost optimization |
| **Compression** | Gzip/Brotli at edge |
| **Image optimization** | Resize, WebP at edge (e.g., Cloudflare Images) |

---

## 8. Technologies Used

| Component | Technologies |
|-----------|--------------|
| **CDN** | CloudFront, Cloudflare, Fastly, Akamai |
| **Origin** | Any (S3, app servers) |
| **DNS** | Route 53, Cloudflare DNS |
| **SSL** | Let's Encrypt, CDN-managed certs |

---

## 9. Challenges & Solutions

| Challenge | Solution |
|-----------|----------|
| **Stale content** | Purge on deploy; or versioned URLs |
| **Personalized content** | Don't cache; or cache per user (careful with keys) |
| **Cache stampede** | Stale-while-revalidate; request coalescing |
| **Invalidation cost** | Prefer versioned URLs over purge |
| **Origin load** | Increase TTL; use Shield; optimize cache keys |
| **HTTPS** | Terminate at edge; origin can be HTTP or HTTPS |
| **Cookie handling** | Exclude cookies from cache key for static; include for personalized |
