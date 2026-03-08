# Video Streaming System Design

## 1. Introduction

### Purpose

A video streaming system (TikTok/YouTube-like) handles uploads, transcoding, storage, and global delivery. It must support multiple resolutions, adaptive bitrate (ABR), low-latency start, and recommendation. This document covers the backend architecture for video ingest and delivery.

### Overview

The architecture includes an upload service, a transcoding pipeline (queue + workers), origin storage (S3), CDN for delivery, metadata database, recommendation service, and analytics. Key patterns: async processing, event-driven, and CDN-first delivery.

---

## 2. Requirements

### Functional Requirements

- Upload videos (chunked, resumable)
- Transcode to multiple resolutions (360p-1080p)
- Stream with ABR (HLS/DASH)
- Metadata: title, description, tags
- Thumbnail generation
- Recommendation engine
- Search and discovery
- Analytics: views, watch time

### Non-Functional Requirements

- **Start latency:** Video plays in < 2 seconds
- **Scalability:** Millions of uploads; billions of plays
- **Quality:** Smooth playback; ABR based on network
- **Cost:** Optimize transcoding and CDN
- **Availability:** 99.9%+

---

## 3. High-Level Architecture

### Components

1. **Upload Service** — Accept chunks; validate; store raw; enqueue job
2. **Transcode Pipeline** — Queue + workers (FFmpeg/MediaConvert)
3. **Origin Storage** — S3 for segments
4. **CDN** — Global segment delivery
5. **Metadata Service** — Video metadata; segment URLs
6. **Recommendation Service** — Ranking; personalization
7. **Analytics** — Views; watch events; reporting

### Communication Flow

- Upload: Client → Upload Service → S3 (raw) → Queue
- Transcode: Worker pulls job → FFmpeg → S3 (segments) → Update metadata
- Play: Client → CDN (segments) → Adaptive quality
- Recommendation: User history + signals → Model → Ranked list

---

## 4. Architecture Diagram

```
                    ┌─────────────────┐
                    │   Upload        │
                    │   Client        │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ Upload Service  │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ Transcode Queue │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ FFmpeg Workers  │
                    │ (multi-res)     │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ Origin (S3)     │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ CDN             │
                    │ (segments)      │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ Playback Client │
                    └─────────────────┘
```

---

## 5. Key Components

### Upload Service

- Multipart or resumable uploads
- Validates format, size, duration
- Stores raw in S3
- Publishes transcode job
- Returns video_id for status polling

### Transcode Pipeline

- Workers pull from SQS/Kafka
- FFmpeg: H.264/H.265; 360p, 480p, 720p, 1080p
- Output: HLS or DASH segments
- Thumbnail extraction
- Upload segments to S3
- Update metadata with segment URLs

### Metadata Service

- Video: id, user_id, title, duration, segment_urls (by quality)
- Serves manifest (m3u8) for HLS
- Cassandra or PostgreSQL

### Recommendation Service

- Collaborative filtering; content-based; sequence models
- Input: watch history, likes, follows
- Output: Ranked video list
- Batch + real-time; A/B test models

---

## 6. Database / Storage Design

### videos

```
video_id, user_id, title, description, duration_seconds, thumbnail_url,
segment_base_url, qualities (map), created_at, view_count, like_count
```

### watch_events

```
user_id, video_id, watch_seconds, completed, timestamp
```

### segments (S3)

```
/videos/{video_id}/{quality}/segment_0.ts, segment_1.ts, ...
```

---

## 7. Scaling Strategy

| Strategy | Implementation |
|----------|----------------|
| **Async transcoding** | Queue; scale workers horizontally |
| **CDN** | All delivery via CDN; minimize origin |
| **Sharding** | Videos by user_id; metadata by video_id |
| **Caching** | Metadata in Redis; CDN for segments |
| **Pre-warming** | Preload next N videos in feed |
| **ABR** | Client adapts; reduce buffering |

---

## 8. Technologies Used

| Component | Technologies |
|-----------|--------------|
| **Transcoding** | FFmpeg, AWS MediaConvert, GCP Transcoder |
| **Storage** | S3, GCS |
| **CDN** | CloudFront, Fastly, Akamai |
| **Queue** | Kafka, SQS |
| **Database** | Cassandra, PostgreSQL, Redis |
| **ABR** | HLS, DASH |
| **ML** | TensorFlow, PyTorch (recommendation) |

---

## 9. Challenges & Solutions

| Challenge | Solution |
|-----------|----------|
| **Transcode time** | Parallel workers; priority queue; partial availability |
| **CDN cost** | Multi-CDN; tiered caching; compress |
| **Start latency** | CDN edge; small initial segment; preload |
| **Thumbnail at scale** | Async; cache; multiple sizes |
| **Recommendation freshness** | Hybrid batch + real-time; frequent model updates |
