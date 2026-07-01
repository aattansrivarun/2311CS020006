# Notification System — Design Document

This document covers Stages 1–4 of the assignment (API design, data modeling, query optimization, and performance/scaling). Stages 5–7 are implemented in code — see `notification-app-be/`, `notification-app-fe/`, and `logging-middleware/`.

---

## Stage 1 — REST API & WebSocket Design

### REST API design

Base URL: `/api`

| Resource | Method | Path | Auth |
|---|---|---|---|
| Users | POST | `/users/register` | — |
| Users | POST | `/users/login` | — |
| Users | GET | `/users/me` | JWT |
| Users | GET | `/users` | JWT |
| Notifications | POST | `/notifications` | JWT |
| Notifications | GET | `/notifications` | JWT |
| Notifications | GET | `/notifications/priority` | JWT |
| Notifications | GET | `/notifications/unread-count` | JWT |
| Notifications | GET | `/notifications/:id` | JWT |
| Notifications | PATCH | `/notifications/:id/read` | JWT |
| Notifications | PATCH | `/notifications/read-all` | JWT |
| Notifications | DELETE | `/notifications/:id` | JWT |

### Example request/response

**Create notification**

```http
POST /api/notifications
Authorization: Bearer <jwt>
Content-Type: application/json

{
  "title": "Assignment 3 Deadline Extended",
  "message": "The deadline has been moved to Friday 6 PM.",
  "recipient": "665f1c2e9a1b2c3d4e5f6789",
  "type": "ASSIGNMENT",
  "priority": "HIGH",
  "deliveryChannel": ["IN_APP", "EMAIL"]
}
```

```http
201 Created
Content-Type: application/json

{
  "notification": {
    "_id": "665f1e...",
    "title": "Assignment 3 Deadline Extended",
    "message": "The deadline has been moved to Friday 6 PM.",
    "type": "ASSIGNMENT",
    "priority": "HIGH",
    "priorityScore": 103,
    "recipient": "665f1c2e9a1b2c3d4e5f6789",
    "isRead": false,
    "deliveryChannel": ["IN_APP", "EMAIL"],
    "createdAt": "2026-06-30T10:15:00.000Z",
    "updatedAt": "2026-06-30T10:15:00.000Z"
  }
}
```

### Headers

- `Authorization: Bearer <jwt>` — required on all protected routes.
- `X-Request-Id` — set by `logging-middleware` on every response, correlates request/response/error log lines.
- `Content-Type: application/json` — all request/response bodies.

### WebSocket design

Socket.IO, authenticated via JWT passed in the handshake `auth` payload (not a query string, to avoid leaking tokens in logs). On connect, the server joins the socket to a room named `user:<userId>` so events are targeted per-user rather than broadcast.

| Event | Direction | Payload | Purpose |
|---|---|---|---|
| `notification:new` | server → client | full notification object | Push a newly created notification in real time |
| `notification:unreadCount` | server → client | `{ unreadCount }` | Keep the badge counter in sync |
| `notification:read` | server → client | `{ notificationId }` | Cross-tab/device read-state sync |
| `notification:markRead` | client → server | `notificationId` | Optional client-initiated hint (persistence still goes through REST) |

---

## Stage 2 — Data Modeling

### Entities

- **Student** (`notification-app-be/src/models/Student.js`) — account/auth record (name, email, hashed password, role, department, rollNumber).
- **Notification** (`.../models/Notification.js`) — the core entity: title, message, type, priority, computed `priorityScore`, recipient, sender, read state, delivery channels.
- **NotificationLog** (`.../models/NotificationLog.js`) — per-channel delivery audit trail (status, attempts, last error) used by the email queue/worker.

### ER diagram (textual)

```
Student (1) ──< sends ─────┐
                            │
Student (1) ──< receives ─>│ Notification (N)
                            │
                            └──< has delivery logs >── NotificationLog (N)
```

- `Notification.recipient` → `Student._id` (required, indexed)
- `Notification.sender` → `Student._id` (optional — system-generated notifications have no sender)
- `NotificationLog.notification` → `Notification._id` (indexed)

### Why MongoDB / NoSQL here

- Notification "metadata" is heterogeneous per type (assignment vs. grade vs. system message) — a flexible `metadata: Mixed` field avoids constant schema migrations that a rigid relational schema would need.
- Read pattern is dominated by "fetch this user's notifications sorted by priority/recency" — a single denormalized collection with a compound index serves that without joins.
- Write pattern is high-volume, append-mostly (new notifications, occasional read-state flips) — well suited to Mongo's document writes.

### NoSQL vs. SQL comparison (for this domain)

| Concern | MongoDB (chosen) | Relational (Postgres) |
|---|---|---|
| Schema flexibility for `metadata` | Native (`Mixed`/embedded doc) | Needs JSONB column or EAV pattern |
| Read pattern (per-user, sorted, filtered) | Single compound index, no joins | Would need a join to users + indexes on FKs |
| Write throughput (notification bursts) | Horizontally shardable by recipient | Vertical scaling / read replicas, sharding is harder |
| Strong relational integrity (FKs enforced) | Application-level only | Enforced by the DB |
| Complex multi-entity transactions | Possible (multi-doc transactions) but not the primary use case here | Native strength |

Given notifications are largely independent, per-user documents with eventual consistency across services (email delivery, sockets) fits Mongo well. A relational store would be a reasonable alternative if the system needed strict referential integrity across many more entities (courses, enrollments, grades) — that's a valid extension point.

### Representative SQL queries (if this were relational)

For reference / comparison, the equivalent core query in SQL:

```sql
-- Paginated, filtered notifications for a user, sorted by priority score
SELECT n.*, s.name AS sender_name
FROM notifications n
LEFT JOIN students s ON s.id = n.sender_id
WHERE n.recipient_id = :userId
  AND (:type IS NULL OR n.type = :type)
  AND (:priority IS NULL OR n.priority = :priority)
  AND (:isRead IS NULL OR n.is_read = :isRead)
ORDER BY n.priority_score DESC, n.created_at DESC
LIMIT :limit OFFSET :offset;
```

```sql
-- Top 10 unread by priority (Priority Inbox)
SELECT * FROM notifications
WHERE recipient_id = :userId AND is_read = false
ORDER BY priority_score DESC, created_at DESC
LIMIT 10;
```

---

## Stage 3 — Query Optimization

### Composite index

The Notification model defines:

```js
notificationSchema.index({ recipient: 1, isRead: 1, priorityScore: -1, createdAt: -1 });
```

This single compound index serves the two hottest queries directly (no in-memory sort, no collection scan):

1. `findByRecipient` (All Notifications list, filtered by `isRead`)
2. `findTopPriority` (Priority Inbox — `recipient` + `isRead: false`, sorted by `priorityScore`)

Field order matters: `recipient` first (equality, highest selectivity), `isRead` second (equality), then the two sort keys (`priorityScore`, `createdAt`) last, matching MongoDB's ESR (Equality → Sort → Range) index rule.

Secondary single-field indexes (`type`, `priority`, `priorityScore`) support ad-hoc filtering combinations that fall outside the compound index's prefix.

### Cost analysis

Without the compound index, `GET /api/notifications?isRead=false` on a large collection (e.g. 500k documents, 5k for the given user) would require either:
- A collection scan (`COLLSCAN`) — O(n) over all notifications, or
- An index scan on `recipient` alone followed by an in-memory sort — bounded by the 32MB sort limit and still O(k log k) for k = that user's documents.

With the compound index, the query plan is a pure `IXSCAN` that returns already-sorted results directly off the index — no separate sort stage, and the scanned key range is limited to exactly `{recipient, isRead}` matches.

### Optimized query pattern in code

`notification.repository.js` always queries with `.lean()` (skips Mongoose document hydration overhead for read-only responses) and pushes pagination (`skip`/`limit`) and sorting down to MongoDB rather than fetching everything and slicing in application code.

For very deep pagination (large `skip` values), a future optimization would switch to cursor-based pagination (`_id`-based `$lt`/`$gt` seeks) to avoid the linear cost of `skip()` — noted here as a known scaling limit of the current offset-based approach.

---

## Stage 4 — Performance & Scaling

### Caching (Redis)

- **Unread count** is requested frequently (every page load, every socket reconnect). It's a good candidate for a short-TTL Redis cache (`unread:<userId>`, TTL ~15s, invalidated on notification create/read) to avoid hitting Mongo on every badge render. Not yet wired in code — the current implementation queries Mongo directly, since Redis is already a dependency (via BullMQ) and can be reused for this.
- **Priority Inbox** results could similarly be cached per-user for a few seconds, since they change less often than raw creation events would suggest (users don't refresh constantly).

### Pagination

`GET /api/notifications` already implements offset pagination (`page`, `limit`, capped at 100/page) backed by the compound index (Stage 3). For infinite-scroll UX specifically, cursor-based pagination (keyed on `_id` or `createdAt`) is the recommended evolution to avoid `skip()` cost at high offsets.

### Lazy loading / infinite scroll (frontend)

The frontend currently uses classic prev/next pagination (`AllNotifications.jsx`) for simplicity and testability. To convert to infinite scroll: replace the page-number state with a cursor, use an `IntersectionObserver` on a sentinel element at the bottom of `NotificationList`, and append new pages to the existing array instead of replacing it.

### API-level optimizations already in place

- `.lean()` reads throughout the repository layer (no Mongoose hydration for read paths).
- Compound index eliminates in-memory sorts for the two hottest queries.
- Email delivery is fully decoupled from the request/response cycle via BullMQ — notification creation returns immediately; email sending happens asynchronously in the worker process (Stage 5), so a slow SMTP provider never blocks the API.
- Socket.IO events are targeted per-user room rather than broadcast to all connected clients, keeping event fan-out O(1) per notification instead of O(all connected sockets).

### Scaling considerations

- **Horizontal scaling of the API**: stateless Express processes behind a load balancer; Socket.IO would need a Redis adapter (`@socket.io/redis-adapter`) so that rooms/events work correctly across multiple server instances — noted as a required addition before scaling `notification-app-be` past a single instance.
- **Horizontal scaling of the worker**: BullMQ workers are already safe to run as multiple processes/instances since Redis coordinates job locking — `npm run worker` can be scaled out independently of the API.
- **Database scaling**: MongoDB can be sharded on `recipient` (or a hashed variant) once a single replica set's write throughput becomes the bottleneck, since almost all queries are already scoped to a single recipient.
