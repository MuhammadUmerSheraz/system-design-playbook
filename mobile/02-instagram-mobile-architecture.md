# Instagram Mobile Architecture

## 1. Introduction

### Purpose

Instagram is a photo/video sharing social network. The mobile architecture must support media upload, personalized feed generation, and global content delivery. This document covers the mobile-specific architecture: feed consumption, media handling, and offline support.

### Overview

The architecture emphasizes CDN-first media delivery, fan-out or fan-in feed generation, and efficient mobile data usage. Key patterns include pre-fetching, infinite scroll optimization, and progressive image loading.

---

## 2. Requirements

### Functional Requirements

- Upload photos and short videos
- View personalized home feed
- Like, comment, share
- Stories (24h ephemeral)
- Search users, hashtags
- Offline viewing of cached content
- Push notifications

### Non-Functional Requirements

- **Feed load:** < 2 seconds
- **Media:** Progressive loading; multiple resolutions
- **Battery:** Efficient background sync
- **Data:** Minimize cellular data usage
- **Scalability:** Millions of daily active users

---

## 3. High-Level Architecture

### Components

1. **Mobile App** — Swift/Kotlin; ViewModel + Repository pattern
2. **API Gateway** — Auth, rate limit
3. **Feed Service** — Pre-computed or on-demand feed
4. **Media Service** — Upload, processing, URLs
5. **CDN** — Global media delivery
6. **Social Graph** — Follow relationships
7. **Local Cache** — SQLite/Realm for offline

### Communication Flow

- Feed: App → API → Feed Service → Redis/Cassandra → Response
- Media: App → CDN (cached) or Origin (cache miss)
- Upload: App → API → Media Service → S3 → CDN

---

## 4. Architecture Diagram

```
                    ┌─────────────────┐
                    │   Mobile App    │
                    │   (local cache) │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │   API Gateway   │
                    └────────┬────────┘
                             │
           ┌─────────────────┼─────────────────┐
           │                 │                 │
    ┌──────▼──────┐   ┌──────▼──────┐   ┌──────▼──────┐
    │ Feed        │   │ Media       │   │ Social      │
    │ Service     │   │ Service     │   │ Graph       │
    └──────┬──────┘   └──────┬──────┘   └─────────────┘
           │                 │
           │          ┌──────▼──────┐
           │          │ CDN         │
           │          │ (media)     │
           │          └─────────────┘
           │
    ┌──────▼──────┐
    │ Redis       │
    │ (feed cache)│
    └─────────────┘
```

---

## 5. Key Components

### Feed Service

- Fan-out on write: Push new post to followers' feed caches
- Fan-in on read: For long-tail users, compute on demand
- Ranking: Recency, engagement, relationship strength
- Pagination: Cursor-based

### Media Service

- Accepts uploads; generates thumbnails
- Stores in S3; serves via CDN
- Multiple resolutions: thumbnail, low, high

### Local Cache (Mobile)

- Cached feed and media URLs
- Offline-first for recently viewed content
- Sync on app foreground

---

## 6. Database / Storage Design

### Posts (Server)

- **posts:** post_id, user_id, media_urls, caption, created_at
- **user_feed:** user_id, post_id, score, created_at (Redis or Cassandra)

### Local (Mobile)

- **cached_posts:** post_id, user_id, media_url, caption, cached_at
- **cached_media:** URL, local_path, expiry

---

## 7. Scaling Strategy

| Strategy | Implementation |
|----------|----------------|
| **CDN** | All media via CDN; minimize origin |
| **Feed cache** | Redis; fan-out for hot users |
| **Sharding** | Posts by user_id; feeds by user_id |
| **Pre-fetch** | Load next page before user scrolls |
| **Progressive images** | Blur hash → low res → full res |

---

## 8. Technologies Used

| Component | Technologies |
|-----------|--------------|
| **Mobile** | Swift, Kotlin, SwiftUI, Jetpack Compose |
| **Backend** | Python, Go, Node.js |
| **Database** | Cassandra, Redis, PostgreSQL |
| **Storage** | S3, CloudFront |
| **Local** | SQLite, Realm, Kingfisher/SDWebImage |

---

## 9. Challenges & Solutions

| Challenge | Solution |
|-----------|----------|
| **Feed freshness** | Fan-out on write; TTL cache; background refresh |
| **Media load time** | CDN; progressive JPEG; blur placeholder |
| **Offline** | Local cache; sync queue; conflict resolution |
| **Data usage** | Compressed responses; image quality based on network |
| **Battery** | Batch requests; background fetch limits |
