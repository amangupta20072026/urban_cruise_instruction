# Urban Cruise — Notification System Design v2

Status: Draft v2 · Supersedes: v1 (`notifications-design.md`) · Scope: infra upgrade — Redis (ioredis), RabbitMQ, BullMQ, WebSocket · Same repos as v1.

This document layers on top of v1. Anything not mentioned here is unchanged: the business event catalogue (v1 §2), the deep-link contract (v1 §8), the sub-role fan-out matrix (v1 §9.3), preferences and quiet hours (v1 §9.4), security posture (v1 §13), retention (v1 §15). What changes is the **transport plumbing** — how business events get from a business service to a user's device, and what new kinds of real-time interactions become possible along the way.

---

## 1. What v2 changes, and why

**Three tools enter the picture:**

- **Redis (ioredis)** — a shared in-memory data plane. Rate limiting across the PM2 cluster (v1 explicitly punted on this — see the comment in `src/shared/http/middleware/rateLimit.ts`), idempotency keys, unread-count caches, distributed locks, WebSocket pub/sub fanout, presence.
- **RabbitMQ** — the event bus. Business services publish state-change events; the notification module (and future subscribers — analytics, audit trail, ops alerting) consume them. Decouples "something changed" from "who cares about it".
- **BullMQ (Redis-backed)** — the notification delivery job queue. Replaces v1's `SELECT ... FOR UPDATE SKIP LOCKED` outbox poller. Native support for retries with backoff, delayed jobs (quiet hours), repeatable jobs (feedback reminders), rate limiting per queue, per-job priority.

**One tool joins them:**

- **WebSocket (Socket.IO + Redis adapter)** — a persistent duplex channel for real-time UI updates when the app is foreground. FCM stays as the fallback and background-delivery guarantee; WebSocket is the "app is open, react instantly" layer.

### 1.1 Why not just BullMQ?

The simplest v2 would drop RabbitMQ and do everything through BullMQ. It would work. Reasons to bring in RabbitMQ anyway:

1. **Multiple consumers per event.** Today the notification module is the only consumer of `booking.assignmentApproved`. Next quarter, analytics wants it. Quarter after, audit compliance wants it. With BullMQ, each new consumer means the business service pushes N jobs to N queues — the business service has to know about every consumer. With RabbitMQ topic exchange, business services publish once with a routing key; new consumers bind their own queue and start receiving. Publisher stays ignorant of consumers.
2. **Cross-language / cross-service future.** BullMQ is Node-specific. RabbitMQ is neutral. If a Python analytics pipeline lands later, RabbitMQ is already the door.
3. **Different failure semantics.** BullMQ job "failed after N retries" is a data-plane concern (this specific delivery couldn't complete). RabbitMQ message "unroutable / dead-lettered" is a topology concern. Keeping them separate keeps triage clean.

### 1.2 Why not just RabbitMQ?

RabbitMQ can do delayed messages (via the delayed-message plugin) and retries (via DLX + TTL loops). It works but is fiddly. BullMQ was designed for exactly the shape of work the FCM adapter does — small jobs, per-job state, retries with backoff, per-worker rate limits (FCM enforces ~600 msg/min per project). Use each tool for what it's good at.

---

## 2. Updated architecture

```
  ┌─────────────────────────────────────────────────────────────────┐
  │  Business service (customer/vendor/driver/uc)                   │
  │                                                                  │
  │   withTransaction(conn => {                                     │
  │     await bookingsRepo.updateStatus(conn, ...);                 │
  │     await eventOutbox.enqueue(conn, {                           │
  │       eventType: 'booking.assignmentApproved',                  │
  │       aggregate: { type: 'booking', id: bookingId },            │
  │       payload:  { bookingId, tripStartAt, vehicleReg, ... },    │
  │     });                                                          │
  │   });                                                            │
  └────────────────────────┬────────────────────────────────────────┘
                           │  DB commit (business change + event row atomic)
                           ▼
              ┌───────────────────────────────┐
              │  Outbox Relay (single worker) │  polls outbox_events with
              │                               │  SKIP LOCKED, publishes to
              └──────────────┬────────────────┘  RabbitMQ with publisher-confirm
                             │
                             ▼
     ┌─────────────────────────────────────────────────────────┐
     │  RabbitMQ  topic exchange: urbancruise.events           │
     │                                                          │
     │   Routing key format:  <domain>.<verb>.<audience>       │
     │     'booking.assignmentApproved.customer'               │
     │     'trip.driverArrived.customer'                       │
     │     'trip.locationTick.public'                          │
     │     'vendor.assignmentOffered.vendor'                   │
     └───────────┬──────────────────────────┬──────────────────┘
                 │                          │
   binds: '*.*.customer'         binds: '*.locationTick.*'
   '*.*.vendor', etc.            'trip.status.*'
                 │                          │
                 ▼                          ▼
   ┌──────────────────────────┐   ┌────────────────────────────┐
   │  q.notifications.        │   │  q.websocket.gateway       │
   │  consumer                │   │                            │
   │                          │   │  emits over Socket.IO      │
   │  1. Resolve recipients   │   │  rooms (user:*, trip:*,    │
   │  2. Preferences filter   │   │  booking:*, vendor:*,      │
   │  3. Persist notifications│   │  uc:enquiries, etc.)       │
   │  4. Enqueue BullMQ jobs  │   └────────────────────────────┘
   └──────────────┬───────────┘
                  │
                  ▼
     ┌─────────────────────────────────────────────────────────┐
     │  BullMQ queue: notifications.push                        │
     │                                                          │
     │  Job payload: { notificationId, deviceId, token, ... }  │
     │  Options: priority, delay (quiet-hours), attempts,       │
     │  backoff (exponential), removeOnComplete: 100,           │
     │  removeOnFail: 500, jobId (idempotency),                 │
     │  queue-level rateLimiter { max: 500, duration: 60000 }   │
     └──────────────┬──────────────────────────────────────────┘
                    │
                    ▼
        Worker pool (N processes, one per PM2 instance)
        → FCM HTTP v1 dispatch
        → Success:  update notifications.delivered_at
        → Failure:  BullMQ handles retry; on final failure,
                    write outbox_events 'notification.deliveryFailed'
                    (so ops sees it via a subscriber)

  ┌──────────────────── Redis (ioredis) — shared data plane ───────────────────┐
  │                                                                             │
  │  · BullMQ storage (queues, delayed set, priority ZSETs)                    │
  │  · Socket.IO Redis adapter (multi-instance pub/sub for room broadcast)     │
  │  · Rate limit buckets (rate-limit-redis backing express-rate-limit)        │
  │  · Idempotency keys  (SET NX EX  notify:idem:{kind}:{correlation})         │
  │  · Unread counts     (HINCRBY    notif:unread:{userId})                    │
  │  · Device token cache (SMEMBERS  devices:user:{userId})                    │
  │  · Presence          (SET EX     ws:presence:{userId}:{socketId})          │
  │  · Distributed locks (SET NX PX  lock:{resource}) for cron singletons      │
  └─────────────────────────────────────────────────────────────────────────────┘
```

The important structural change from v1: **business services no longer call `notify()` directly**. They publish a business event via the outbox. The notification consumer subscribes to events and does the notify work. That inversion is what buys the "add analytics later without touching business code" property.

---

## 3. Redis (ioredis) — every place it earns its keep

One Redis cluster (or, at v1 scale, one Redis instance with a replica) serves multiple concerns. Each concern uses its own key namespace so an ops person reading `redis-cli MONITOR` can tell what's happening at a glance.

### 3.1 Rate limiting — closes the v1 gap

The v1 comment in `src/shared/http/middleware/rateLimit.ts` explicitly notes that under PM2 cluster mode a client gets N buckets, one per worker. Fix: swap the in-memory store for `rate-limit-redis` + ioredis.

```ts
// src/shared/http/middleware/rateLimit.ts (updated)
import RedisStore from 'rate-limit-redis';
import { redis } from '../../redis/client.js';

export const globalRateLimit = rateLimit({
  windowMs: ENV.RATE_LIMIT_WINDOW_MS,
  limit: ENV.RATE_LIMIT_MAX,
  standardHeaders: 'draft-8',
  legacyHeaders: false,
  handler: rateLimitHandler,
  store: new RedisStore({
    sendCommand: (...args) => redis.call(...args),
    prefix: 'rl:global:',
  }),
});
```

Same treatment for `authRateLimit` and the future `notifyPerUserRateLimit` (v1 §13). Removes the v1 caveat entirely.

### 3.2 Idempotency — replaces the DB table for the hot path

v1 §9.2 used a `notification_idempotency` MySQL table. Reads and writes on every notify. Redis is a natural fit:

```
SET notify:idem:{kind}:{correlation} 1 NX EX 30
```

If the return is `nil`, this is a duplicate — return early. If it's `OK`, proceed. The DB table is retained for audit only (a nightly job copies keys that fired into a much smaller `notification_dedupe_audit` table for the same 24h window). Hot path stays at O(1) Redis.

### 3.3 Unread count — kills the "count on every list" query

Notification centre badges rely on unread count. Doing `SELECT COUNT(*) WHERE user_id=? AND read_at IS NULL` per open is a scan per user across a growing table.

```
HINCRBY notif:unread:{userId} total 1        # on new notification
HINCRBY notif:unread:{userId} category:trip 1
HINCRBY notif:unread:{userId} total -1       # on mark-read
```

Cache miss → hydrate from DB (`SELECT category, COUNT(*)`), `HSET`, done. TTL isn't required (write-through), but a 7-day TTL is a defensive rebuild trigger in case of drift.

### 3.4 Device token cache — avoids per-notification lookup

For every notification enqueued for a user we need `SELECT ... FROM notification_devices WHERE user_id=? AND disabled_at IS NULL`. Cache the token IDs in a Redis set:

```
SADD  devices:user:{userId}  {deviceId1} {deviceId2}
SREM  devices:user:{userId}  {deviceId}   # on disable
DEL   devices:user:{userId}                # on logout-all
```

Hit → skip DB. Miss → hydrate. Full device row (token, platform, etc.) is only needed at BullMQ job creation, not at fanout time — the job carries just `deviceId` and the worker reads the row once from DB (or from a short-lived per-device cache with 60s TTL, keyed by `device:{deviceId}`).

### 3.5 Distributed locks — for cron and singleton work

A "feedback reminder scheduler" that runs hourly must not double-fire when the same cron lands on two PM2 workers. Simple lock via Redis SET NX PX:

```ts
// src/shared/redis/locks.ts
export async function withLock<T>(
  key: string,
  ttlMs: number,
  fn: () => Promise<T>,
): Promise<T | 'skipped'> {
  const token = crypto.randomUUID();
  const acquired = await redis.set(`lock:${key}`, token, 'PX', ttlMs, 'NX');
  if (acquired !== 'OK') return 'skipped';
  try {
    return await fn();
  } finally {
    // release only if we still own the lock (Lua CAS)
    await redis.eval(
      "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end",
      1, `lock:${key}`, token,
    );
  }
}
```

For anything requiring stronger guarantees (multi-master Redis, extended lock times), a proper Redlock library slots in without changing callers. v1 scale doesn't warrant it.

### 3.6 Presence — knows if the user's app is open

`ws:presence:{userId}` is a Redis SET containing socket IDs currently connected for that user. Updated on Socket.IO connect/disconnect. Two uses:

- The notification handler MAY skip FCM for `category='reminder'` notifications when presence is non-empty (user is looking at the app right now; WebSocket does the delivery). It MUST still send FCM for `category='transactional'` — WS could drop the instant we send.
- The WebSocket gateway can early-exit broadcasts to rooms with no members (Socket.IO handles this natively, but presence lets us report "not delivered because offline" in analytics without inference).

### 3.7 One Redis, many concerns — key namespace map

| Namespace | Purpose | TTL | Notes |
|---|---|---|---|
| `bull:*` | BullMQ internal (managed by lib) | per-job | Don't touch by hand |
| `rl:global:*`, `rl:auth:*`, `rl:notify:*` | Rate limit windows | window size | `rate-limit-redis` prefix |
| `notify:idem:{kind}:{correlation}` | Idempotency | 30s | SET NX EX |
| `notif:unread:{userId}` | Unread hash | 7d | HINCRBY |
| `devices:user:{userId}` | User's active device IDs | 24h | SADD/SREM |
| `device:{deviceId}` | Full device row (JSON) | 60s | GETSET |
| `ws:presence:{userId}` | Connected socket IDs | 90s | SADD + heartbeat refresh |
| `lock:{resource}` | Distributed lock | ≤ 5min | SET NX PX |
| `so-ns#/#/*` | Socket.IO Redis adapter internals | managed | Don't touch |

Sharding, if throughput demands it later: move BullMQ to a dedicated Redis instance (heaviest write load), keep everything else on the primary. Zero code change beyond a second connection.

---

## 4. RabbitMQ — event bus

### 4.1 Topology

**One topic exchange:** `urbancruise.events` (durable).

**Routing key format:** `<domain>.<verb>.<audience>`

- `domain` — the business domain: `booking`, `trip`, `quotation`, `payment`, `vendor`, `driver`, `enquiry`, `support`, `system`.
- `verb` — what happened, past tense: `created`, `assignmentApproved`, `driverArrived`, `otpVerified`, `completed`, `cancelled`, `paymentReceived`, `locationTick`, `slaBreached`, etc.
- `audience` — who broadly cares: `customer`, `vendor`, `driver`, `uc`, `public` (all connected users watching the entity), `internal` (no user-facing side, e.g. reconciliation).

Examples:

```
booking.assignmentApproved.customer
booking.cancelled.customer
trip.driverArrived.customer
trip.driverArrived.uc
trip.locationTick.public
vendor.assignmentOffered.vendor
uc.enquirySubmitted.uc
system.reconciliationException.internal
```

The audience segment is what lets a subscriber bind precisely. WebSocket gateway binds `#` on the `.public` and `.customer` audiences it cares about; the notification consumer binds everything user-facing; an analytics consumer binds `#` and doesn't care.

### 4.2 Queues and bindings

| Queue | Binding | Consumer count | Prefetch | Purpose |
|---|---|---|---|---|
| `q.notifications.consumer` | `#` on `urbancruise.events` | 2–4 | 32 | Runs recipient resolution + enqueues BullMQ jobs |
| `q.websocket.gateway` | `#` on `urbancruise.events` | 1 per WS gateway process | 64 | Bridges events to Socket.IO rooms |
| `q.notifications.consumer.dlq` | DLX target | manual drain | — | Dead-letter for poison events |
| `q.audit` (future) | `#.customer`, `#.vendor`, `#.driver`, `#.uc` | 1 | 16 | Long-term event log |
| `q.analytics` (future) | `#` | 1 | 64 | Warehouse loader |

All queues durable, all messages persistent. Publisher-confirms enabled on the outbox relay so a NACK causes the outbox row to stay `pending` for retry.

### 4.3 Message envelope

```jsonc
{
  "eventId":     "9fbb2a...",              // uuid, generated at outbox insert; the idempotency key
  "eventType":   "booking.assignmentApproved",
  "occurredAt":  "2026-09-05T12:34:56.789Z",
  "requestId":   "req-3b2c...",             // from AsyncLocalStorage context
  "actor": {
    "userId":  "usr-abc...",                // who caused the event; null for system
    "role":    "uc",
    "subRole": null
  },
  "aggregate": {
    "type": "booking",
    "id":   "BKG20260905A"                  // for routing to entity-scoped WS rooms
  },
  "payload": { /* event-specific — schema per eventType */ }
}
```

All events validated against a Zod schema at publish AND at consume. A drift between producer and consumer becomes a validation error, not a silent misroute.

### 4.4 Dead-letter policy

Each work queue is declared with `x-dead-letter-exchange: urbancruise.events.dlx` and `x-dead-letter-routing-key: <original>.deadletter`. The DLQ retains messages for 14 days for postmortem. A recurring alert fires if DLQ depth grows.

### 4.5 Outbox → relay

The outbox pattern from v1 is retained but generalised. The v1 `notification_outbox` table becomes `outbox_events` — one relay, publishes to RabbitMQ for every business event, not just notifications.

```sql
CREATE TABLE outbox_events (
  id             CHAR(36)     PRIMARY KEY,          -- eventId in the envelope
  event_type     VARCHAR(80)  NOT NULL,             -- 'booking.assignmentApproved'
  aggregate_type VARCHAR(32)  NOT NULL,             -- 'booking'
  aggregate_id   VARCHAR(64)  NOT NULL,
  payload        JSON         NOT NULL,
  request_id     VARCHAR(64)  NULL,
  actor          JSON         NULL,
  occurred_at    DATETIME(3)  NOT NULL,
  published_at   DATETIME(3)  NULL,
  attempt_count  INT UNSIGNED NOT NULL DEFAULT 0,
  next_attempt_at DATETIME    NOT NULL,
  locked_by      CHAR(36)     NULL,
  locked_until   DATETIME     NULL,
  KEY idx_ready (published_at, next_attempt_at),
  KEY idx_aggregate (aggregate_type, aggregate_id, occurred_at)
);
```

Relay loop (one instance, protected by `withLock('outbox-relay', 30_000)`):

```
1. SELECT ... FROM outbox_events
   WHERE published_at IS NULL AND next_attempt_at <= NOW()
   ORDER BY occurred_at LIMIT 200 FOR UPDATE SKIP LOCKED;
2. For each row:
     ch.publish('urbancruise.events', routingKey(row), Buffer.from(envelope),
                { persistent: true, messageId: row.id });
3. Wait for publisher-confirms (channel.waitForConfirms()).
4. UPDATE outbox_events SET published_at = NOW() WHERE id IN (confirmed);
   For unconfirmed: bump attempt_count, schedule next_attempt_at.
```

The relay never runs business logic. Its only job is "get the row across the boundary". This is what makes "DB commit + event publish" atomic: business services always see a committed row; the relay guarantees at-least-once delivery to RabbitMQ; RabbitMQ guarantees at-least-once to consumers; consumers dedupe on `eventId`.

The old `notification_outbox` table is deleted. Delivery-attempt state now lives in BullMQ (§5).

### 4.6 Consumer contract

`q.notifications.consumer` handler:

```ts
async function onEvent(msg: ConsumeMessage) {
  const envelope = EventEnvelope.parse(JSON.parse(msg.content.toString()));

  // Idempotency — this consumer may see the same event twice
  const first = await redis.set(
    `notify:event-handled:${envelope.eventId}`,
    '1', 'NX', 'EX', 60 * 60 * 24, // 24h dedupe window
  );
  if (first !== 'OK') return channel.ack(msg);

  try {
    await handleEvent(envelope);          // recipient resolution + enqueue BullMQ jobs
    channel.ack(msg);
  } catch (err) {
    if (isRetryable(err)) {
      channel.nack(msg, false, true);     // requeue
    } else {
      channel.nack(msg, false, false);    // dead-letter
    }
  }
}
```

`handleEvent` does exactly what v1's `notify()` did — resolve recipients, apply preferences, insert `notifications` rows, then enqueue BullMQ jobs (§5). The Redis idempotency gate replaces the DB-based one for this hot path.

---

## 5. BullMQ — the notification delivery queue

### 5.1 Queues

Three queues, differentiated by priority so backpressure on low-priority work doesn't starve transactional pushes:

| Queue | Purpose | Rate limit | Concurrency |
|---|---|---|---|
| `notifications.push.high` | `priority='high'` transactional | 500 msg/min per worker | 20 |
| `notifications.push.normal` | `priority='normal'` | 300 msg/min per worker | 10 |
| `notifications.push.low` | `priority='low'` reminders | 100 msg/min per worker | 5 |

Rate limits protect against FCM's project-level quota (~600 msg/min soft limit — verify current values with your Google Cloud console before shipping). Total across all three queues stays under the ceiling.

Priority *within* a queue uses BullMQ's built-in `opts.priority` for finer ordering (lower number first).

### 5.2 Job payload

```ts
type PushJob = {
  notificationId: string;   // notifications.id in Postgres/MySQL
  deviceId: string;         // notification_devices.id
  fcmToken: string;         // duplicated so worker doesn't re-read the row
  platform: 'ios' | 'android';
  ttlSeconds: number;
  collapseKey: string | null;
  message: {                // pre-rendered by the consumer
    title: string;
    body:  string;
    data:  Record<string, string>;   // the section-8-of-v1 payload
    channelId: string;
    priority: 'high' | 'normal';
  };
};
```

Job options:

```ts
await pushQueue.add('deliver', jobPayload, {
  jobId: `${notificationId}:${deviceId}`,   // idempotency — retry-safe
  priority: bullPriorityFor(notification.priority),
  delay: quietHoursDelayMs,                  // 0 for transactional-high
  attempts: 6,
  backoff: { type: 'exponential', delay: 2_000 },  // 2s, 4s, 8s, 16s, 32s, 64s
  removeOnComplete: { count: 100 },
  removeOnFail: { count: 500 },
});
```

`jobId` is the (notification, device) pair — if the consumer re-processes an event and re-enqueues, BullMQ silently dedupes.

### 5.3 Worker

One `Worker` per queue per PM2 instance. Under cluster mode of N workers this gives 3×N total workers — Redis handles the coordination. No `NODE_APP_INSTANCE=='0'` guard needed; that v1 hack is retired.

```ts
new Worker<PushJob>('notifications.push.high', async job => {
  const outcome = await fcm.send(job.data);
  await recordDeliveryOutcome(job.data.notificationId, job.data.deviceId, outcome);
  if (outcome.type === 'unregistered') {
    // Disable the device and don't retry
    await devicesRepo.markDisabled(job.data.deviceId, 'UNREGISTERED');
    throw new UnrecoverableError('device unregistered');  // BullMQ won't retry
  }
  if (outcome.type === 'transient') throw outcome.err;    // BullMQ retries per backoff
  // success — nothing to throw
}, {
  connection: redis,
  concurrency: 20,
  limiter: { max: 500, duration: 60_000 },
  autorun: true,
});
```

`UnrecoverableError` (a BullMQ export) skips retries. All other throws follow the queue's retry policy.

### 5.4 Scheduled and repeatable jobs

Two categories that v1 hand-rolled:

- **Feedback reminders.** v1 had a "24h after trip end, no feedback → notify" cron. In BullMQ:
  ```ts
  await reminderQueue.add('feedback', { bookingId }, { delay: 24 * 3600 * 1000 });
  ```
  Delay is precise, survives restarts, no cron table needed.
- **Assignment SLA breach.** Vendor didn't accept in 30 min → alert UC. Same pattern: enqueue with `delay: 30 * 60 * 1000` at assignment time. If the vendor accepts first, cancel the job by `jobId`.

Repeatable jobs (BullMQ's cron-like feature) cover things v1 handled with a nightly script: retention cleanup, unread-count reconciliation, device heartbeat sweep.

### 5.5 UI

BullMQ has an official dashboard (`@bull-board/express`) — mount at `/admin/queues` behind IP allow-list + basic auth (or your existing UC session). Free ops visibility: queue depth, active workers, per-job retry history.

---

## 6. WebSocket layer

### 6.1 Choice: Socket.IO + Redis adapter

Options considered:

| Option | Pros | Cons |
|---|---|---|
| Raw `ws` + custom rooms + Redis Streams | Minimal deps | Rebuilds Socket.IO features poorly |
| `uWebSockets.js` | Fastest | Ecosystem thin; React Native client story worse |
| Socket.IO 4 + `@socket.io/redis-adapter` | Battle-tested client, auto-reconnect, rooms, ACKs, native RN client | Slightly heavier wire format |

Socket.IO. The React Native client (`socket.io-client`) handles reconnection, auth-refresh flows, and background/foreground transitions on both platforms with battle-tested defaults. Wire overhead is negligible relative to typical payloads.

Redis adapter (`@socket.io/redis-adapter`) is what lets a `broadcast to room trip:X` from any Node process reach every connected socket — necessary the moment PM2 cluster or multiple app servers is in play.

### 6.2 Where WebSocket belongs — the audit

The user asked specifically: *where in this business can WebSocket be used?* Every entry below is grounded in a feature that already exists in your code.

**Definitely WebSocket (large UX win, FCM alone won't cut it):**

| Use case | Grounded in | Room | Event names |
|---|---|---|---|
| Live driver location on customer's live-trip screen | `customer.tripLive` deep-link + `DriverLocationForegroundService.kt` referenced in `capabilities.ts` | `trip:{tripId}` | `trip.locationTick`, `trip.legStarted`, `trip.legEnded` |
| UC trip monitor (many trips at once) | `uc.tripMonitor` kind | join multiple `trip:{tripId}` rooms | same as above |
| UC live enquiry inbox | `/uc/enquiries` list | `uc:enquiries` | `enquiry.submitted`, `enquiry.assignedInternally` |
| Vendor assignment popup + SLA countdown | `/vendor/assignments` + assignment reminder from v1 §2.2 | `vendor:{vendorId}` | `vendor.assignmentOffered`, `vendor.assignmentCancelled` |
| Payment status flip during checkout | `POST /customer/bookings/:id/payments/pay` → gateway webhook | `booking:{bookingId}` | `payment.received`, `payment.failed` |
| Booking status flip on open detail screen | `customer.bookingDetail` view | `booking:{bookingId}` | `booking.assignmentApproved`, `booking.cancelled`, `booking.vehicleChangeApproved` |
| Notification-centre unread badge | Every role's More sheet has a Notifications entry | `user:{userId}` | `notification.received` |
| Vehicle-change decision (vendor UI live update) | `POST /uc/trips/:id/vehicle-change/approve` | `vendor:{vendorId}` + `booking:{bookingId}` | `vendor.vehicleChangeDecision` |
| Driver "you have a new trip" instant popup | `driver.tripAssigned` kind | `user:{driverId}` | `driver.tripAssigned` |
| OTP verification flip on customer screen | `POST /driver/trips/:id/otp` | `trip:{tripId}` | `trip.otpVerified` |
| Multi-device sync for corporate customer | Corporate `admin` + `bookingPerson` under one `CustomerId` | `customer:{customerId}` (fanout room) | any customer-audience event |

**Marginal WebSocket (nice-to-have; polling or FCM alone is acceptable):**

| Use case | Why marginal |
|---|---|
| Driver approval by UC on vendor's screen | Vendor rarely stares at the approval screen; a push + on-open refresh is enough |
| Referral status | Not time-critical |
| GST invoice ready | Low-priority notification path is fine |

**Do NOT use WebSocket:**

| Use case | Why |
|---|---|
| Content that only matters when the screen is opened (settings, historical lists, static config) | Screen mount refetch handles it |
| Anything the app doesn't need until foreground | FCM + on-foreground refetch is cheaper on battery |
| Fire-and-forget analytics | Wrong tool — HTTP POST to a collector |

### 6.3 Rooms — the naming scheme

| Room | Membership rule | Broadcasts received |
|---|---|---|
| `user:{userId}` | Every socket for that user joins on connect | Anything private to the user (notification-centre updates, personal state) |
| `customer:{customerId}` | Every socket whose identity's `role='customer'` and `entityId=customerId` | Fan-out for corporate: admin + bookingPerson receive same event |
| `vendor:{vendorId}` | Similar, all sub-roles under one VendorId | Assignment events, vehicle-change decisions, payouts |
| `trip:{tripId}` | Any authenticated socket that calls `subscribe('trip:{tripId}')` AND passes the server-side visibility check | Location ticks, status flips |
| `booking:{bookingId}` | Same subscribe-with-check pattern | Status flips, payment events |
| `uc:enquiries` | Sockets whose identity's `role='uc'` and whose UC subscription includes enquiries | New enquiry submitted |
| `uc:trips` | Similar | High-level trip monitor stream (opt-in to keep bandwidth sane) |

`subscribe` and `unsubscribe` are explicit events the client emits when a screen mounts/unmounts. The server validates the subscription against the user's visibility rules (same `canRead(Trip, tripId, identity)` policy the HTTP endpoints use) before joining the socket to the room. Never let the client dictate room membership without a check — that would be a data-leak surface.

### 6.4 Auth

JWT in the Socket.IO handshake auth field:

```ts
io.use(async (socket, next) => {
  try {
    const token = socket.handshake.auth?.token;
    if (!token) return next(new Error('AUTH_MISSING_TOKEN'));
    const claims = verifyAccessToken(token);
    if (!ENABLED_ROLES.has(claims.role)) return next(new Error('ROLE_NOT_ENABLED'));
    socket.data.identity = {
      userId: String(claims.sub),
      role: claims.role,
      subRole: claims.subRole,
      entityId: claims.entityId,
    };
    next();
  } catch (err) {
    next(new Error('AUTH_INVALID_TOKEN'));
  }
});
```

The same `verifyAccessToken` from `src/shared/auth/jwt.ts` — one JWT verifier for HTTP and WS. On expiry: client's HTTP refresh interceptor (`src/api/interceptors/refresh`) obtains a new access token; the WS client listens for that update and reconnects with the fresh token. A short helper on the client watches Redux `authStatus` and calls `socket.auth = { token: newToken }; socket.disconnect().connect();`.

### 6.5 Connection lifecycle

- **Connect:** post-bootstrap, only when `authStatus === 'authenticated'`. Never in provisional state.
- **Rooms joined on connect:** `user:{userId}`, `customer|vendor:{entityId}` (based on role), `uc:{scope}` for UC.
- **Rooms joined on demand:** `trip:*`, `booking:*` — client subscribes when the screen mounts, unsubscribes on unmount.
- **Foreground/background:** on `AppState === 'background'`, disconnect after 30s idle to save battery. On `AppState === 'active'`, reconnect. React Native's `AppState` events already flow through the app.
- **Reconnect strategy:** exponential backoff up to 30s, jitter. On reconnect, client re-fetches current state for any open screen (TanStack Query's `refetchOnReconnect: true` handles this cleanly).
- **Presence heartbeat:** server-side `pingInterval: 25000`, `pingTimeout: 60000`. Presence key TTL 90s, refreshed on each ping.

### 6.6 WebSocket ↔ FCM decision matrix

Both channels can carry the same notification, so we need a clear rule:

| Notification category | If WS presence for user | If NO WS presence |
|---|---|---|
| `transactional, high` (arrival, OTP, cancel) | **Both** — WS for instant UI flip, FCM as belt-and-braces (client dedupes by `notifId`) | FCM only |
| `transactional, normal` (assignment approved, payment received) | **Both** | FCM only |
| `reminder` (feedback, payment due) | **WS only** — user's already in the app; a push would be redundant | FCM only |
| `system` (announcements) | **WS only** if opened; **FCM only** if not | FCM only |
| Ephemeral (`trip.locationTick`, live counters) | **WS only** — never persisted, never FCM'd | Dropped (no persistence) |

The client-side dedup by `notifId` (v1 §8.4) is what makes "send both" safe. Even if the FCM push races the WS event and both arrive, only one Notifee alert renders.

### 6.7 The WebSocket gateway process

`q.websocket.gateway` consumer runs inside the same Node processes that accept HTTP + Socket.IO connections. When it consumes an event from RabbitMQ:

1. Determines target rooms from `(eventType, aggregate)` — a small map, e.g. `trip.locationTick → [trip:{aggregate.id}]`, `booking.assignmentApproved → [booking:{aggregate.id}, customer:{payload.customerId}]`.
2. `io.to(rooms).emit(eventType, envelope.payload)`. The Redis adapter fans out across processes.
3. ACKs the RabbitMQ message.

The gateway does not touch the DB. It's a pure event transformer.

---

## 7. Failure modes and degradation

The system is now three additional processes deeper. Each dependency needs an explicit degradation.

| Dependency down | What breaks | What still works | Mitigation |
|---|---|---|---|
| **Redis** | BullMQ paused, rate limiter permissive-open, WS adapter degraded to single-process broadcast, presence blind | HTTP API, DB writes, business logic | Rate limiter falls back to in-memory (existing v1 behaviour); notifications accumulate in `outbox_events` and drain when Redis returns; WS still works per-process (users on the same node still hear each other's broadcasts) |
| **RabbitMQ** | New events sit in `outbox_events`; no live WS or notifications flowing | Everything user-triggered; historical notification-centre reads | Outbox relay retries with backoff; alert on `outbox_events` backlog > 5min old |
| **BullMQ workers** | Pushes queued but not delivered | WS still delivers to online users; notifications persisted; users see them on centre open | Alert on queue depth; BullMQ jobs are durable in Redis |
| **FCM (Google)** | Pushes fail; jobs retry per backoff, then `failed` | WS delivers to online users; centre still populates | v1's retry schedule; on prolonged outage, ops can pause the `high` queue and drain manually |
| **WebSocket gateway** | No real-time UI updates | FCM push still fires; on-open refetch fills gaps | Reconnect + refetch already client-side; alert on gateway process crash-loops |

The point of the outbox is that a business change is never lost — it will eventually reach RabbitMQ, then the consumer, then Redis/BullMQ/FCM. The point of dual channels (WS + FCM) is that no single downstream failure prevents a user from finding out about a state change.

---

## 8. Environment variables added

```
# Redis (single instance or cluster URI)
REDIS_URL                       redis://localhost:6379/0   # dev
REDIS_TLS                       false | true
REDIS_KEY_PREFIX                uc:                        # global prefix for all keys

# RabbitMQ
RABBITMQ_URL                    amqp://user:pass@host:5672/vhost
RABBITMQ_EXCHANGE               urbancruise.events
RABBITMQ_PUBLISH_CONFIRM_TIMEOUT_MS   5000

# BullMQ
BULLMQ_PREFIX                   uc:bull
BULLMQ_HIGH_CONCURRENCY         20
BULLMQ_NORMAL_CONCURRENCY       10
BULLMQ_LOW_CONCURRENCY          5
BULLMQ_HIGH_RATE_PER_MIN        500
BULLMQ_NORMAL_RATE_PER_MIN      300
BULLMQ_LOW_RATE_PER_MIN         100

# WebSocket
WS_ENABLED                      true
WS_PATH                         /socket.io
WS_PING_INTERVAL_MS             25000
WS_PING_TIMEOUT_MS              60000
WS_MAX_HTTP_BUFFER_SIZE         1048576         # 1MB, matches your API body limits

# Outbox relay
OUTBOX_RELAY_ENABLED            true
OUTBOX_RELAY_POLL_MS            1000
OUTBOX_RELAY_BATCH              200
```

Existing v1 variables `NOTIF_WORKER_*` are retired (poller is gone) except `NOTIF_WORKER_MAX_ATTEMPTS`, which is renamed `NOTIF_MAX_ATTEMPTS` and passed to BullMQ's `attempts` option.

---

## 9. Backend module layout — updated

```
src/
├── shared/
│   ├── redis/
│   │   ├── client.ts               # ioredis singleton (like db/pool.ts)
│   │   ├── locks.ts                # withLock helper (§3.5)
│   │   └── keys.ts                 # centralised key builders + prefixes
│   ├── mq/
│   │   ├── connection.ts           # amqplib singleton with reconnect
│   │   ├── publisher.ts            # publishEvent(envelope): confirms wrapper
│   │   ├── consumer.ts             # helper — declare queue, bind, consume with ack/nack policy
│   │   └── envelope.ts             # Zod EventEnvelope + eventTypeRegistry
│   ├── bullmq/
│   │   ├── connection.ts           # BullMQ requires its own ioredis with maxRetriesPerRequest: null
│   │   ├── queues.ts               # push.high / push.normal / push.low + reminder queue
│   │   └── admin.ts                # bull-board mount helper
│   └── ws/
│       ├── server.ts               # Socket.IO server + Redis adapter attach
│       ├── authMiddleware.ts       # JWT handshake auth
│       ├── rooms.ts                # join helpers + visibility checks
│       └── gateway.ts              # RabbitMQ consumer that emits into rooms
│
├── modules/
│   ├── notifications/              # (same as v1 §6)
│   │   ├── ...
│   │   └── dispatch/
│   │       ├── consumer.ts         # RabbitMQ consumer (replaces old worker.ts poller)
│   │       ├── handleEvent.ts      # eventType → recipients → notifications rows → enqueue BullMQ
│   │       ├── producer.ts         # publishes 'notification.*' events (for analytics later)
│   │       └── providers/
│   │           └── fcm.ts          # unchanged from v1
│   └── outbox/                     # new
│       ├── relay.ts                # the outbox → RabbitMQ loop
│       ├── enqueue.ts              # eventOutbox.enqueue(conn, envelope) — the caller-facing API
│       └── schemas.ts              # per-eventType Zod schemas
│
└── workers/                        # PM2 entry points
    ├── api.ts                      # buildApp + createServer + Socket.IO + gateway consumer
    ├── notifications-consumer.ts   # dedicated: RabbitMQ consumer + BullMQ producer
    └── notifications-worker.ts     # dedicated: BullMQ Worker + FCM adapter
```

Splitting `workers/*` into separate PM2 processes lets each scale on its own axis. `api` fleet scales with HTTP + WS traffic. `notifications-consumer` scales with event volume (usually 1–2 is enough). `notifications-worker` scales with FCM throughput.

`ecosystem.config.cjs` adds:

```js
{ name: 'api',                     script: 'dist/workers/api.js',                    instances: 'max', exec_mode: 'cluster', wait_ready: true },
{ name: 'notifications-consumer',  script: 'dist/workers/notifications-consumer.js', instances: 1 },
{ name: 'notifications-worker',    script: 'dist/workers/notifications-worker.js',   instances: 2 },
{ name: 'outbox-relay',            script: 'dist/workers/outbox-relay.js',           instances: 1 },  // singleton, protected by Redis lock
```

---

## 10. Frontend integration additions

Beyond v1 §11, the app adds:

- **`src/services/realtime/socket.ts`** — singleton Socket.IO client. Managed connect on `authStatus === 'authenticated'`; disconnect on logout; reconnect on background→foreground with fresh token.
- **`src/services/realtime/subscriptions.ts`** — helpers `subscribeToTrip(tripId)`, `subscribeToBooking(bookingId)`. Called from `useEffect` in the screen; return-cleanup unsubscribes.
- **`src/services/realtime/bridge.ts`** — receives WS events and either:
  - Dispatches Redux actions (e.g. `notificationReceived(...)` bumps unread counter),
  - Invalidates the TanStack Query cache for the affected keys, or
  - Emits a Notifee alert if it's a notification event that FCM hasn't already handled (dedupe by `notifId` in the MMKV set).
- **`src/store/slices/notificationsSlice.ts`** — small slice holding `unreadCount` and the currently-open list of subscribed rooms (for reconnect resubscribe).
- **AppState hook** — connect on `active`, schedule disconnect 30s after `background`.

The `handleFcmClick` and deep-link resolver require zero changes; WS-delivered notification events use the same `click_target` payload and the same tap handling path.

---

## 11. Observability additions

Everything from v1 §14 plus:

**Metrics:**
- `outbox_backlog_seconds` (gauge) — age of oldest unpublished event.
- `mq_publish_total{eventType, outcome}`, `mq_consume_total{queue, outcome}`.
- `bullmq_jobs_total{queue, state}`, `bullmq_job_duration_ms_bucket{queue}`.
- `ws_connections_current` (gauge, per instance), `ws_rooms_current{prefix}` (gauge).
- `ws_broadcast_total{event, outcome}`, `ws_broadcast_recipients_total{event}` (histogram).
- `redis_command_total{command, outcome}` (via ioredis event hooks).

**Alerts:**
- `outbox_backlog_seconds > 300` for 2m → paging.
- `bullmq_jobs_total{state='failed'}` rate > 20/min → warn.
- `bullmq_jobs_total{state='waiting'}` > 5000 → warn (consumer lag).
- `mq_consume_total{outcome='deadletter'}` any → warn.
- `ws_connections_current` drop > 30% over 1m → warn (mass disconnect, likely a deploy or a gateway crash).

**Structured logs** — one line per event stage with the `eventId` and `requestId` correlating:

```
{ scope: 'outbox.relay.published',      eventId, eventType, requestId }
{ scope: 'notif.consumer.handled',      eventId, recipients: n, enqueued: m }
{ scope: 'notif.bullmq.processed',      jobId, notifId, deviceId, durationMs, outcome }
{ scope: 'ws.gateway.emitted',          eventId, rooms: [...], recipients: n }
```

Given a support ticket "I didn't see my driver arrive at 3pm", the search chain is: find `notif.consumer.handled` for `trip.driverArrived` with the relevant `tripId` in payload → follow `eventId` → `notif.bullmq.processed` for the deliver outcome → `ws.gateway.emitted` for whether the WS also fired → done in three queries.

---

## 12. Migration from v1

If v1 shipped before this upgrade:

**Phase A — introduce Redis (safe, drop-in).**
1. Deploy Redis + ioredis client.
2. Swap `rate-limit` store → `rate-limit-redis`. Behaviour becomes correct under PM2 cluster.
3. Add unread-count cache with write-through; read path falls back to DB on miss.
4. No user-visible change; can be reverted per-file.

**Phase B — introduce BullMQ, retire the DB poller.**
1. Add `notifications.push.*` queues.
2. Change `notify()` to insert `notifications` rows AND `pushQueue.add(...)` in the same transaction using `waitUntilReady` on the queue. (BullMQ writes to Redis, not the DB, so this is "eventual consistency" — see next line.)
3. Keep the v1 `notification_outbox` table populated in parallel for one week; run BOTH the old poller (dark) and new BullMQ workers (live). Compare metrics.
4. Cut over. Delete the poller and the `notification_outbox` table.

**Phase C — introduce RabbitMQ + outbox relay.**
1. Add `outbox_events` table.
2. Add `eventOutbox.enqueue()` in one business service (pick a low-traffic one, e.g. `feedback.reminder`). Business services now insert both a business row and an outbox row in the same transaction.
3. Deploy relay + `q.notifications.consumer`. Consumer calls the same `handleEvent` code the direct-call path uses.
4. For that one event type, delete the direct `notify()` call — everything now flows through the event bus.
5. Migrate remaining event types module by module.

**Phase D — introduce WebSocket.**
1. Deploy Socket.IO server behind the same Nginx (upgrade `Upgrade`/`Connection` headers already needed for WS).
2. Deploy WS gateway consumer.
3. Ship the frontend socket client behind a feature flag; default off.
4. Enable per-room, starting with `user:{userId}` (notification badge) — smallest blast radius.
5. Progressively enable `trip:*`, `booking:*`, entity rooms.

Each phase is independently reversible if problems appear.

---

## 13. Rollout plan (v2, if starting fresh)

If you haven't shipped v1 yet, do everything in this order — it's what I'd do:

1. Ship the schema and stub endpoints (v1 §17 Phase 0).
2. Ship Redis + rate-limit-redis + idempotency + unread cache. Kills the v1 rate-limit TODO on day one.
3. Ship RabbitMQ + `outbox_events` + relay + `q.notifications.consumer`.
4. Ship BullMQ + FCM worker. Now business services can emit events and users get push.
5. Ship Socket.IO + Redis adapter + gateway consumer. Now the app also gets real-time UI updates.
6. Wire the top-5 kinds (v1 §17 Phase 2).
7. Fill in the rest module by module.

---

## 14. Open questions carried over / newly raised

Carried from v1: UC subscription model, localisation, silent trip-live sync (now answered — WebSocket, not silent FCM).

New in v2:

- **Redis Sentinel vs Cluster vs single instance.** Start with a single Redis instance + a hot replica. Move to Sentinel for automated failover before the first traffic spike. Cluster only when a single instance's memory is the bottleneck (unlikely at Urban Cruise's foreseeable scale).
- **RabbitMQ HA policy.** Quorum queues (RabbitMQ 3.8+) for durability; mirrored classic queues are deprecated. Set `x-queue-type: quorum` on every work queue.
- **WebSocket sticky sessions at Nginx.** Not required with the Redis adapter, but reduces reconnect churn. `ip_hash` in the upstream block or `least_conn` is fine.
- **Fair scheduling on BullMQ.** If a small subset of high-fanout events (e.g. system announcement to all customers) crowds out normal traffic, consider a dedicated `notifications.push.broadcast` queue with its own rate limit. Not needed until you actually see the problem.
- **Client-side event replay.** If a user is offline for a day and reconnects, WebSocket alone won't backfill missed events. Solution is already in place: on reconnect, TanStack Query refetches; the notification centre paginated list reveals the backlog. No separate "event replay stream" needed.

---

## Appendix — Tool-to-concern mapping (one-glance)

| Concern | Tool | Why |
|---|---|---|
| Durable business record | MySQL | Existing, transactional |
| Business event boundary crossing (atomicity) | MySQL `outbox_events` + relay | Only way to be atomic without XA |
| Business event fan-out to multiple consumers | RabbitMQ topic exchange | Publisher-agnostic to consumers |
| Notification delivery job (retries, backoff, priorities, rate limit) | BullMQ | Purpose-built |
| Rate limit across cluster | Redis + rate-limit-redis | Closes v1 gap |
| Idempotency key (hot path) | Redis SET NX EX | Cheaper than DB |
| Unread count | Redis HINCRBY | Kills the COUNT(*) |
| Live UI updates when app foreground | Socket.IO + Redis adapter | Right shape |
| Guaranteed background delivery | FCM | Only way when app closed |
| Presence | Redis SET + Socket.IO heartbeats | Trivial with the pieces already in |
| Distributed lock for cron singletons | Redis SET NX PX | Sufficient at this scale |

Every tool in the stack has exactly one primary job. That's the property we want to protect as this grows.
