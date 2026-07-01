## Notification App Backend
# notification-app-be

Express + MongoDB (Mongoose) + Socket.IO + BullMQ backend for the notification system.

## Stack

- **Runtime:** Node.js / Express
- **DB:** MongoDB via Mongoose
- **Realtime:** Socket.IO (per-user rooms, JWT-authenticated handshake)
- **Queue:** BullMQ + Redis (email delivery, retries, dead-letter tracking)
- **Auth:** JWT (bcrypt-hashed passwords)
- **Logging:** `logging-middleware` (local workspace package) — request/response/error logs

## Setup

```bash
cd notification-app-be
npm install
cp .env.example .env   # fill in Mongo URI, JWT secret, SMTP, Redis
npm run dev             # starts API server with nodemon
npm run worker          # starts the BullMQ email worker (separate process)
```

Requires a running MongoDB instance and a running Redis instance (for the email queue).

## Folder structure

```
src/
├── config/       # db.js (Mongoose connect), socket.js (Socket.IO init)
├── controllers/  # request handlers
├── routes/       # Express routers
├── services/     # business logic (notification, email, priority, websocket)
├── middleware/   # auth, validation, error handling
├── models/       # Mongoose schemas
├── repository/   # data-access layer (keeps Mongoose queries out of services)
├── jobs/         # BullMQ queue producer + worker
└── socket/       # Socket.IO event handlers
app.js            # Express app (middleware + routes)
server.js         # HTTP + Socket.IO server bootstrap
```

## API overview

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| POST | `/api/users/register` | — | Create an account |
| POST | `/api/users/login` | — | Log in, returns JWT |
| GET | `/api/users/me` | ✅ | Current user profile |
| GET | `/api/users` | ✅ | List active users (for picking recipients) |
| POST | `/api/notifications` | ✅ | Create a notification (optionally queues an email) |
| GET | `/api/notifications` | ✅ | Paginated, filterable list (`type`, `priority`, `isRead`, `search`, `page`, `limit`) |
| GET | `/api/notifications/priority?limit=10` | ✅ | Top-N notifications by priority score (Stage 6) |
| GET | `/api/notifications/unread-count` | ✅ | Unread badge count |
| GET | `/api/notifications/:id` | ✅ | Single notification |
| PATCH | `/api/notifications/:id/read` | ✅ | Mark one as read |
| PATCH | `/api/notifications/read-all` | ✅ | Mark all as read |
| DELETE | `/api/notifications/:id` | ✅ | Delete a notification |
| GET | `/health` | — | Health check |

All authenticated routes expect `Authorization: Bearer <token>`.

## Realtime events (Socket.IO)

Client connects with `{ auth: { token: '<JWT>' } }`. Server events:

- `notification:new` — a new notification for this user
- `notification:unreadCount` — updated unread badge count
- `notification:read` — a notification was marked read (cross-tab sync)

## Stage 5 — Email queue & retries

`src/jobs/emailQueue.js` adds jobs with 5 attempts and exponential backoff. `src/jobs/notificationWorker.js` processes them, logs each attempt to `NotificationLog`, and marks jobs `DEAD_LETTER` after all retries are exhausted.

## Stage 6 — Priority inbox

`src/services/priority.service.js` computes a numeric `priorityScore` per notification (priority weight + type weight + recency boost), which is stored and indexed for fast top-N queries (`GET /api/notifications/priority`).
