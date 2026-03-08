# Offline-First Mobile Architecture

## 1. Introduction

### Purpose

Offline-first architecture ensures apps work without connectivity. Users can read, create, and update data locally; changes sync when the network returns. This improves perceived performance and reliability, especially in low-connectivity regions.

### Overview

The architecture centers on a local database as the source of truth for the UI, a sync queue for pending mutations, a sync engine for pull/push, and conflict resolution when the same data changes in multiple places.

---

## 2. Requirements

### Functional Requirements

- Read and write locally without network
- Queue mutations when offline
- Sync when connection restored
- Resolve conflicts (multi-device edits)
- Partial sync (incremental updates)
- Bi-directional sync
- Handle large datasets (pagination)

### Non-Functional Requirements

- **Durability:** No data loss on app kill
- **Performance:** Non-blocking UI; background sync
- **Battery:** Efficient; batch sync
- **Consistency:** Eventual consistency; clear conflict rules

---

## 3. High-Level Architecture

### Components

1. **Local Database** — SQLite, Realm, WatermelonDB
2. **Sync Queue** — Pending create/update/delete
3. **Sync Engine** — Pull (fetch changes) + Push (send mutations)
4. **Conflict Resolver** — Last-write-wins, merge, or manual
5. **Sync API** — Server endpoints for delta sync
6. **Change Tracker** — Vector clocks, version numbers

### Communication Flow

- **Pull:** Client sends last_sync_token → Server returns changes → Client merges
- **Push:** Client sends mutation batch → Server applies → Returns conflicts if any
- **Conflict:** Resolver picks winning version; optionally update server

---

## 4. Architecture Diagram

```
                    ┌─────────────────┐
                    │   Mobile App    │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │  Data Layer     │
                    │  (Repository)   │
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
 ┌──────▼──────┐     ┌──────▼──────┐     ┌──────▼──────┐
 │ Local DB    │     │ Sync Queue  │     │ Conflict    │
 │ (SQLite)    │     │ (pending)   │     │ Resolver    │
 └──────┬──────┘     └──────┬──────┘     └──────┬──────┘
        │                    │                    │
        └────────────────────┼────────────────────┘
                             │
                    ┌────────▼────────┐
                    │  Sync Engine    │
                    │  (pull + push)  │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │  Sync API       │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │  Server DB      │
                    └─────────────────┘
```

---

## 5. Key Components

### Local Database

- Schema mirrors server or subset
- Indexes for common queries
- Encrypted for sensitive data

### Sync Queue

- Table of pending operations (op, entity_type, entity_id, payload, timestamp)
- Processed in order per entity
- Persisted across restarts

### Sync Engine

- **Pull:** GET /sync?since=token
- **Push:** POST /sync with batch
- Triggers: Foreground, network change, periodic
- Exponential backoff on failure

### Conflict Resolution

| Strategy | Use Case |
|----------|----------|
| Last-write-wins | Simple; timestamp comparison |
| Server-wins | Central authority |
| Client-wins | User-edited content |
| Merge | Field-level; domain-specific |
| Manual | Present to user |
| CRDT | Automatic; no conflicts |

---

## 6. Database / Storage Design

### Local Schema

```
entities (id, data, updated_at, sync_status)
sync_queue (id, operation, entity_type, entity_id, payload, created_at)
sync_metadata (key, value)  -- last_sync_token
```

### Server Sync API

- **GET /sync?since=X** — Changes since token
- **POST /sync** — Mutations batch; returns applied + conflicts

---

## 7. Scaling Strategy

| Strategy | Implementation |
|----------|----------------|
| **Delta sync** | Only changed records; reduce payload |
| **Pagination** | Large result sets paginated |
| **Batching** | Push 50-100 mutations per request |
| **Compression** | Gzip sync payloads |
| **Background** | WorkManager/BackgroundTasks |

---

## 8. Technologies Used

| Component | Technologies |
|-----------|--------------|
| **Local** | SQLite, Room, Realm, WatermelonDB |
| **Sync** | Custom, Couchbase Lite, Realm Sync |
| **Backend** | REST or GraphQL |
| **Conflict** | Custom, CRDT (Yjs, Automerge) |

---

## 9. Challenges & Solutions

| Challenge | Solution |
|-----------|----------|
| **Conflict detection** | Version/vector clock; compare on sync |
| **Large sync** | Delta + pagination; cursor-based |
| **Ordering** | Timestamp or sequence per entity |
| **Deleted records** | Soft delete; tombstone propagation |
| **Schema migration** | Versioned schema; migration scripts |
