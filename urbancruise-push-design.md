# Urban Cruise — Mobile Push Notifications: Design

**Status:** Draft for review
**Scope:** End-to-end design of the push-notification system spanning `urbancruisebackend` and `urbancruiseapp`. Production-grade, secure, and evolvable — designed to work on the current MySQL-only stack and to grow without rewrites.
**Audience:** Backend + mobile engineers, on-call.

---

## 1 · Codebase context (what already exists)

| Layer | Reality on `main` today |
|---|---|
| **Backend framework** | Express 5 + TypeScript, MySQL 8 (mysql2), Pino, Zod, JWT (access + refresh with separate secrets), Helmet, express-rate-limit, PM2 with `wait_ready`. No Redis, no Kafka, no queue system. |
| **Backend module shape** | `routes → authenticate → authorize(action, subject) → validate(schemas) → controller → service → repository`. Every write goes through `withTransaction()`. Every query filters by `identity.entityId` on the tenant column. Files: `routes / controller / service / repository / schemas / policy / visibility / types / index`. |
| **RBAC** | 4 top-level roles (`customer / vendor / driver / uc`) with sub-roles for corporate customers and vendor staff. `Notification` already exists as an RBAC subject; `read:Notification` is granted to `anyCustomer`. |
| **Notifications module** | Fully scaffolded but **entirely stubbed** — all files exist under `src/modules/notifications/` with `TODO(step-2)` markers. This is a greenfield build inside an existing skeleton. |
| **Client** | RN 0.86, `@react-native-firebase/app@26` + `messaging@26` + `crashlytics@26` already installed. Firebase auto-init from `google-services.json` / `GoogleService-Info.plist`. `initFirebase()` runs during `runBootstrap` and registers a **no-op background message handler**. Redux Toolkit + TanStack Query (with persister), MMKV for non-sensitive storage, Keychain for tokens. `NotificationCentreScreen` is a `ComingSoon` placeholder. Endpoints registry has `/notifications`, `/notifications/:id/read`, `/notifications/read-all` stubs. `NavigationService` exists. |
| **What is missing** | `POST_NOTIFICATIONS` permission and default notification channel/icon/color meta in `AndroidManifest.xml`; APNs entitlements + Notification Service Extension for iOS; Notifee (or equivalent presentation lib); device-registration endpoint; any producer/consumer plumbing; token lifecycle; preferences; observability. |

The design below is written to **drop into that shape cleanly** — no framework/pattern deviation.

---

## 2 · Research summary (why the choices are what they are)

1. **FCM legacy HTTP API was shut down July 2024.** Everything must use FCM **HTTP v1** with a service-account JSON and OAuth2 access tokens (1-hour rotating). Firebase Admin SDK does this transparently. ([Google's own migration doc](https://firebase.google.com/docs/cloud-messaging/migrate-v1))
2. **iOS uses APNs token-based auth (`.p8`)** — never certificate-based in 2026. FCM can proxy APNs (default) or the backend can call APNs directly. We use FCM-as-single-provider, because it also handles topic fanout, batching, and quota back-off for us.
3. **Token lifecycle** (Firebase's own guidance): Android FCM tokens are garbage-collected after **270 days of inactivity**; iOS tokens have no such expiry but rotate on reinstall / restore / permission change / data clear. Any HTTP v1 send that returns `UNREGISTERED` (404) or `INVALID_ARGUMENT` (400 for token) means the token is permanently dead — delete it. Consider tokens stale at 30–60 days of no client ping and drop them proactively; sending to stale tokens burns quota and triggers provider throttling.
4. **Universal shape of a production notification system** (WhatsApp, Uber, Slack write-ups, plus every 2026 system-design reference): thin API returns `202 Accepted` after writing to a **transactional outbox**; workers read the outbox, apply user preferences + quiet hours, dedupe on an idempotency key, then fan out per-channel to isolated dispatchers with per-provider back-off and a dead-letter queue.
5. **Provider isolation matters.** An APNs incident must not stall FCM; a marketing burst must not sit behind an OTP. Priority lanes are non-negotiable at scale.
6. **App-state matrix on RN.** `messaging()` covers three states differently: `onMessage` (foreground), `setBackgroundMessageHandler` (background/quit for data messages), and `getInitialNotification` + `onNotificationOpenedApp` (cold launch / warm launch from a tap). **Notifee** is the widely adopted presentation layer for foreground display, Android channels/importance, action buttons, grouping, and reliable event tracking — @react-native-firebase alone doesn't render foreground notifications on Android.
7. **Notification payloads must be data-only** for reliable behavior across states. Any payload with a top-level `notification` block bypasses `setBackgroundMessageHandler` on Android and gets auto-rendered — losing us click-tracking and preventing us from applying local rules (mute, group, redact). We render everything ourselves via Notifee, from a data payload.

---

## 3 · Design goals

| Goal | Concretely |
|---|---|
| **Reliable** | At-least-once delivery to the provider; zero silent loss between DB write and provider hand-off; automatic retry on transient failure with exponential back-off + jitter; DLQ for un-retryable failures. |
| **Scalable** | The API path never blocks on provider I/O. Producer and consumer scale independently. Fanout for campaigns (all customers, all vendors) is batched, not per-user API calls. |
| **Secure** | Device tokens are treated as PII: encrypted-at-rest, never logged, revoked on logout, one-token-per-device (not per-user), never returned via API. Payload contents are never exposed cross-tenant. All routes go through the standard `authenticate + authorize + tenant-scoped repo` triple. |
| **Maintainable** | Fits the existing module scaffold exactly; every dispatcher, template, and channel is a small isolated piece; adding a new notification type is one migration + one template + one enum entry. |
| **Robust long-term** | Explicit upgrade path from **MVP (MySQL-only, single process)** → **v2 (Redis for dedupe + rate-limit)** → **v3 (Kafka/RabbitMQ for durable queue + separate worker service)**. Each step is a scoped change, not a rewrite. |
| **Observable** | Every notification has a `notification_id` traced from ingest → outbox → dispatch → provider response → device open. Structured Pino logs on every hop with the request-id AsyncLocalStorage value we already have. |

---

## 4 · High-level architecture

```
                       ┌──────────────────────────────────────────────┐
   Business event      │  Producing module (bookings, trips, quotes)  │
   (e.g. Booking       │       service.createNotification({ ... })    │
   Confirmed)          └──────────────────┬───────────────────────────┘
                                          │  (same DB tx as the business write)
                                          ▼
                       ┌───────────────────────────────────────────┐
                       │  notifications module — outbox writer     │
                       │  INSERT INTO notification (dedupe key)    │  ← business tx boundary
                       │  INSERT INTO notification_outbox          │
                       └──────────────────┬────────────────────────┘
                                          │  (async, after commit)
                       ┌──────────────────▼────────────────────────┐
                       │  Dispatcher worker (in-process job or     │
                       │  separate PM2 process — see §7)           │
                       │  ─ leases outbox rows                     │
                       │  ─ applies user prefs + quiet hours       │
                       │  ─ per-device fanout                      │
                       │  ─ template render (Zod-typed data blob)  │
                       └──────────────────┬────────────────────────┘
              ┌──────────────────┬────────┴────────┬──────────────────┐
              ▼                  ▼                 ▼                  ▼
     ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
     │ FCM channel  │   │ In-app only  │   │ (email — v2) │   │ (sms — v2)   │
     │  (HTTP v1)   │   │ (WebSocket / │   │              │   │              │
     │              │   │  next fetch) │   │              │   │              │
     └──────┬───────┘   └──────────────┘   └──────────────┘   └──────────────┘
            │  provider response
            ▼
     ┌────────────────────────────────────────────┐
     │  Token feedback: UNREGISTERED / INVALID    │
     │  → mark device_token disabled, purge       │
     └────────────────────────────────────────────┘


   Device                                                            Client app
   ┌──────────────────────────────────────────────────────────────────────────┐
   │  FCM (Android)                                                            │
   │  APNs via FCM (iOS)                                                       │
   │      │                                                                    │
   │      ▼                                                                    │
   │  RN app:                                                                  │
   │     initFirebase()  ─ registers BG handler + fetches token                 │
   │     onMessage       ─ foreground → Notifee display                         │
   │     BG handler      ─ Notifee display + optional silent cache sync         │
   │     open handler    ─ getInitialNotification + onNotificationOpenedApp     │
   │                       → NavigationService.navigateFromNotification         │
   │     token watcher   ─ onTokenRefresh → POST /notifications/devices        │
   └──────────────────────────────────────────────────────────────────────────┘
```

---

## 5 · Data model (MySQL)

Six tables. All named columns match the repo's snake_case convention. All FKs are `RESTRICT` on delete unless noted; hard-delete of a user goes through an explicit purge job so token history is preserved for audit.

### 5.1 `device_token`

Records **one row per (user, device installation)**. A device is identified by a stable `device_install_id` generated by the app (see §8.1) so a user with an iPhone and an iPad has two rows and each device replaces its own row on token refresh.

```sql
CREATE TABLE device_token (
  device_token_id       CHAR(26) NOT NULL PRIMARY KEY,             -- ULID
  user_id               CHAR(26) NOT NULL,
  entity_id             CHAR(26) NOT NULL,                          -- tenant column, mirrors JWT
  role                  ENUM('customer','vendor','driver','uc') NOT NULL,
  device_install_id     CHAR(36) NOT NULL,                          -- stable UUID minted by app
  platform              ENUM('ios','android') NOT NULL,
  provider              ENUM('fcm') NOT NULL DEFAULT 'fcm',         -- future-proofed
  token_ciphertext      VARBINARY(1024) NOT NULL,                   -- AES-256-GCM
  token_iv              BINARY(12) NOT NULL,
  token_tag             BINARY(16) NOT NULL,
  token_fingerprint     CHAR(64) NOT NULL,                          -- SHA-256 of plaintext; for equality lookups
  app_version           VARCHAR(32)  NOT NULL,
  os_version            VARCHAR(32)  NOT NULL,
  locale                VARCHAR(16)  NOT NULL,
  timezone              VARCHAR(64)  NOT NULL,
  status                ENUM('active','disabled','revoked') NOT NULL DEFAULT 'active',
  disabled_reason       VARCHAR(64)  NULL,                          -- UNREGISTERED, INVALID_ARGUMENT, LOGOUT, STALE
  last_seen_at          DATETIME     NOT NULL,                      -- app checks in on foreground
  created_at            DATETIME     NOT NULL,
  updated_at            DATETIME     NOT NULL,

  UNIQUE KEY uq_device_install (user_id, device_install_id),        -- one row per install per user
  UNIQUE KEY uq_fingerprint (token_fingerprint),                    -- one row per unique token globally
  KEY idx_user_active (user_id, status),
  KEY idx_last_seen (status, last_seen_at)
);
```

**Why encrypted at rest.** A leaked DB dump shouldn't leak a working push channel to every user. Encryption key is stored in the same secrets vault as `JWT_ACCESS_SECRET`; a `PUSH_TOKEN_ENC_KEY` env var joins the Zod-parsed `ENV`.
**Why fingerprint.** The `UNIQUE KEY uq_fingerprint` lets a token stolen by another install (device restored to a new user account, or a user re-installing) be **stolen back** cleanly: the incoming registration wins and the losing row is disabled. Without this, one physical device could have multiple active rows and receive duplicate pushes.

### 5.2 `notification_preference`

```sql
CREATE TABLE notification_preference (
  user_id               CHAR(26) NOT NULL PRIMARY KEY,
  push_enabled          TINYINT(1) NOT NULL DEFAULT 1,
  categories_json       JSON      NOT NULL,   -- { "booking": {push: true}, "marketing": {push: false}, ... }
  quiet_hours_start     TIME      NULL,        -- user-local, e.g. 22:00
  quiet_hours_end       TIME      NULL,        -- e.g. 07:00
  timezone              VARCHAR(64) NOT NULL,  -- from device
  updated_at            DATETIME  NOT NULL
);
```

Categories align with `NotificationCategory` (see §6.2). Quiet hours apply only to `priority IN ('normal','low')` — OTPs and trip-critical alerts always break through.

### 5.3 `notification`

The user-visible in-app notification (drives the Notification Centre screen) — **not** the delivery job.

```sql
CREATE TABLE notification (
  notification_id       CHAR(26) NOT NULL PRIMARY KEY,              -- ULID
  user_id               CHAR(26) NOT NULL,
  entity_id             CHAR(26) NOT NULL,
  role                  ENUM('customer','vendor','driver','uc') NOT NULL,
  category              VARCHAR(32) NOT NULL,                       -- 'booking' | 'trip' | 'payment' | ...
  event_type            VARCHAR(64) NOT NULL,                       -- 'booking.confirmed' etc.
  priority              ENUM('critical','high','normal','low') NOT NULL,
  title                 VARCHAR(120) NOT NULL,
  body                  VARCHAR(500) NOT NULL,
  data_json             JSON NOT NULL,                              -- {bookingId, tripId, ...} for deep-link
  dedupe_key            VARCHAR(128) NOT NULL,                      -- see §7.3
  read_at               DATETIME NULL,
  created_at            DATETIME NOT NULL,

  UNIQUE KEY uq_dedupe (user_id, dedupe_key),
  KEY idx_user_created (user_id, created_at DESC),
  KEY idx_user_unread (user_id, read_at, created_at DESC)
);
```

`uq_dedupe` guarantees that if a booking service accidentally raises `booking.confirmed` twice for the same booking, the user sees exactly one notification and one push — enforced at DB level, not application code.

### 5.4 `notification_outbox`

The delivery queue. Uses **row-level lease + status transitions** — a poor-man's queue that works fine on MySQL for MVP and is the standard pattern until traffic pushes us to Redis or Kafka.

```sql
CREATE TABLE notification_outbox (
  outbox_id             CHAR(26) NOT NULL PRIMARY KEY,              -- ULID (time-sorted for FIFO index scan)
  notification_id       CHAR(26) NOT NULL,
  user_id               CHAR(26) NOT NULL,
  channel               ENUM('push','inapp') NOT NULL,
  priority              ENUM('critical','high','normal','low') NOT NULL,
  status                ENUM('pending','processing','sent','failed','dead') NOT NULL DEFAULT 'pending',
  attempts              TINYINT UNSIGNED NOT NULL DEFAULT 0,
  next_attempt_at       DATETIME NOT NULL,                          -- back-off scheduled time
  leased_by             VARCHAR(64) NULL,                           -- worker id
  leased_until          DATETIME NULL,                              -- lease expiry (prevents stuck rows)
  last_error_code       VARCHAR(64) NULL,
  last_error_message    VARCHAR(500) NULL,
  created_at            DATETIME NOT NULL,
  updated_at            DATETIME NOT NULL,

  KEY idx_dispatch (status, priority, next_attempt_at),             -- worker fetch index
  KEY idx_notif (notification_id),
  CONSTRAINT fk_outbox_notif FOREIGN KEY (notification_id) REFERENCES notification(notification_id)
);
```

### 5.5 `notification_delivery`

Per-device delivery attempt log. Feeds observability + token-feedback purge.

```sql
CREATE TABLE notification_delivery (
  delivery_id           CHAR(26) NOT NULL PRIMARY KEY,
  outbox_id             CHAR(26) NOT NULL,
  device_token_id       CHAR(26) NOT NULL,
  provider              ENUM('fcm') NOT NULL,
  provider_message_id   VARCHAR(128) NULL,
  status                ENUM('accepted','delivered','failed','expired') NOT NULL,
  error_code            VARCHAR(64) NULL,                           -- UNREGISTERED, INVALID_ARGUMENT, QUOTA_EXCEEDED, INTERNAL
  latency_ms            INT UNSIGNED NULL,
  attempted_at          DATETIME NOT NULL,

  KEY idx_outbox (outbox_id),
  KEY idx_device (device_token_id, attempted_at),
  KEY idx_error (error_code, attempted_at)                          -- for on-call dashboards
);
```

### 5.6 `notification_open_event`

Client-reported open events (see §8.5). Feeds engagement analytics; independent from `notification.read_at` (which is the in-app viewed state).

```sql
CREATE TABLE notification_open_event (
  event_id              CHAR(26) NOT NULL PRIMARY KEY,
  notification_id       CHAR(26) NOT NULL,
  user_id               CHAR(26) NOT NULL,
  device_install_id     CHAR(36) NOT NULL,
  opened_at             DATETIME NOT NULL,
  app_state             ENUM('foreground','background','quit') NOT NULL,
  KEY idx_notif (notification_id)
);
```

---

## 6 · Backend module — file-by-file

Everything lives under `src/modules/notifications/` following the scaffold that already exists.

### 6.1 `types.ts`

```ts
export type NotificationCategory =
  | 'booking' | 'trip' | 'payment' | 'quotation'
  | 'account' | 'support' | 'marketing' | 'system';

export type NotificationPriority = 'critical' | 'high' | 'normal' | 'low';

export type NotificationEventType =
  | 'booking.confirmed' | 'booking.modified' | 'booking.cancelled'
  | 'trip.assigned'     | 'trip.starting'    | 'trip.completed'
  | 'quotation.received' | 'quotation.expiring'
  | 'payment.received'  | 'payment.due'
  | 'account.otp'       | 'account.password_changed'
  // ... exhaustive; every new event added here (compile-time safety)
  ;

// Row shapes (mirror DB)
export type DeviceTokenRow = { /* ... */ };
export type NotificationRow = { /* ... */ };
export type OutboxRow = { /* ... */ };

// DTOs returned to the client (never leak *_ciphertext, provider_message_id, entity_id)
export type NotificationDTO = { id; category; eventType; priority; title; body; data; readAt; createdAt };
```

### 6.2 Notification catalog (`catalog.ts` — new file)

Every notification type is declared **once** in a typed catalog:

```ts
export const CATALOG: Record<NotificationEventType, NotificationSpec> = {
  'booking.confirmed': {
    category: 'booking',
    priority: 'high',
    audience: { role: 'customer' },
    template: (input: BookingConfirmedInput) => ({
      title: 'Booking confirmed',
      body:  `Your trip to ${input.destination} on ${input.date} is confirmed.`,
      data:  { screen: 'BookingDetail', bookingId: input.bookingId },
    }),
    // Zod schema validating template input
    inputSchema: BookingConfirmedInput,
    // Dedupe strategy — see §7.3
    dedupe: (input) => `booking.confirmed:${input.bookingId}`,
    // Which roles are allowed to receive it (guards accidental mis-targeting)
    roleGuard: ['customer'],
  },
  // ... one entry per event
};
```

**Why a catalog.** Adding a new notification is a 5-line diff to one file, plus a Zod schema. No new endpoint, no new controller, no new dispatcher. TypeScript refuses to compile if a producer tries to emit an event that isn't declared.

### 6.3 `schemas.ts`

Zod for the **HTTP surface only** — internal producers call `service.createNotification()` directly.

```ts
// POST /notifications/devices  — register or refresh device token
export const registerDeviceSchema = z.object({
  deviceInstallId: z.string().uuid(),
  platform: z.enum(['ios','android']),
  token: z.string().min(100).max(4000),           // FCM tokens
  appVersion: z.string().max(32),
  osVersion:  z.string().max(32),
  locale:     z.string().max(16),
  timezone:   z.string().max(64),
});

// PATCH /notifications/preferences
export const updatePreferencesSchema = z.object({
  pushEnabled: z.boolean().optional(),
  categories:  z.record(z.enum([...]), z.object({ push: z.boolean() })).optional(),
  quietHours:  z.object({ start: z.string().regex(/^\d{2}:\d{2}$/), end: z.string().regex(/^\d{2}:\d{2}$/) }).nullable().optional(),
  timezone:    z.string().max(64).optional(),
});

// GET /notifications  — cursor pagination (created_at DESC + id tie-break)
export const listQuerySchema = z.object({
  cursor: z.string().optional(),
  limit:  z.coerce.number().int().min(1).max(50).default(20),
  unreadOnly: z.coerce.boolean().default(false),
});

// POST /notifications/:id/open — client-side open event
export const openEventSchema = z.object({
  deviceInstallId: z.string().uuid(),
  appState: z.enum(['foreground','background','quit']),
  openedAt: z.string().datetime(),
});
```

### 6.4 `routes.ts`

```ts
import { Router } from 'express';
import { authenticate } from '@shared/http/middleware/authenticate.js';
import { authorize }    from '@shared/http/middleware/authorize.js';
import { validate }     from '@shared/http/middleware/validate.js';
import { rateLimiter }  from '@shared/http/middleware/rateLimit.js';   // named per-route limiter
import * as c from './controller.js';
import * as s from './schemas.js';

const r = Router();
r.use(authenticate);

// Read the user's own notifications
r.get('/',                    authorize('read','Notification'),
      validate({ query: s.listQuerySchema }),                                     c.list);
r.post('/:id/read',           authorize('update','Notification'),                  c.markRead);
r.post('/read-all',           authorize('update','Notification'),                  c.markAllRead);

// Device registration
r.post('/devices',            authorize('update','Notification'),
      rateLimiter('device-register', { windowMs: 60_000, max: 5 }),
      validate({ body: s.registerDeviceSchema }),                                 c.registerDevice);
r.delete('/devices/:installId', authorize('update','Notification'),
      validate({ params: s.deleteDeviceParams }),                                 c.deleteDevice);

// Preferences
r.get('/preferences',         authorize('read','Notification'),                    c.getPrefs);
r.patch('/preferences',       authorize('update','Notification'),
      validate({ body: s.updatePreferencesSchema }),                              c.updatePrefs);

// Client open-event beacon
r.post('/:id/open',           authorize('acknowledge','Notification'),
      validate({ body: s.openEventSchema }),                                      c.recordOpen);

export default r;
```

Then in `policies.ts` (shared RBAC): add `'update:Notification'` and `'acknowledge:Notification'` to `anyCustomer` — extend later when vendor/driver/uc land.

### 6.5 `service.ts`

The **internal** API is what other modules call:

```ts
export async function createNotification<E extends NotificationEventType>(
  conn: PoolConnection | null,         // pass conn to enlist in an outer tx; null → own tx
  event: E,
  input: EventInput<E>,
  target: { userId: string; entityId: string; role: UserRole },
): Promise<NotificationId> {
  const spec = CATALOG[event];

  // 1. Role guard (catches producer bugs at runtime)
  if (!spec.roleGuard.includes(target.role))
    throw new ForbiddenError(`Event ${event} is not deliverable to role ${target.role}`, 'NOTIF_ROLE_MISMATCH');

  // 2. Validate template input
  const parsed = spec.inputSchema.parse(input);

  // 3. Render
  const rendered = spec.template(parsed);
  const dedupeKey = spec.dedupe(parsed);

  // 4. Persist (idempotent INSERT via uq_dedupe)
  const notifId = await repo.insertNotification(conn, {
    userId: target.userId, entityId: target.entityId, role: target.role,
    category: spec.category, eventType: event, priority: spec.priority,
    title: rendered.title, body: rendered.body, data: rendered.data,
    dedupeKey,
  });
  if (!notifId) return null;   // duplicate — dedupe hit

  // 5. Enqueue outbox rows (one per channel)
  await repo.enqueueOutbox(conn, { notificationId: notifId, channel: 'push', priority: spec.priority });
  // future: 'email', 'sms' — added by the same call as producer opts in

  return notifId;
}
```

**Contract:** if `conn` is provided, the notification INSERT and outbox INSERT participate in the caller's transaction — so the delivery job never exists without its business event, and vice-versa. This is the **transactional outbox pattern**, implemented with MySQL only.

### 6.6 `repository.ts`

Every query filters by `identity.entityId`. Example:

```ts
export async function listByUser(
  conn: PoolConnection | Pool,
  identity: Identity,
  { cursor, limit, unreadOnly }: ListParams,
): Promise<NotificationRow[]> {
  const [rows] = await conn.execute<RowDataPacket[]>(
    `SELECT notification_id, category, event_type, priority, title, body, data_json, read_at, created_at
       FROM notification
      WHERE user_id = ? AND entity_id = ?
        ${unreadOnly ? 'AND read_at IS NULL' : ''}
        ${cursor ? 'AND (created_at, notification_id) < (?, ?)' : ''}
      ORDER BY created_at DESC, notification_id DESC
      LIMIT ?`,
    cursor
      ? [identity.userId, identity.entityId, cursor.createdAt, cursor.id, limit + 1]
      : [identity.userId, identity.entityId, limit + 1],
  );
  return rows as NotificationRow[];
}
```

Two things worth calling out:

- The `entity_id` filter is a hard **defense-in-depth** — even if RBAC is misconfigured, a customer literally cannot see another tenant's notification because MySQL will return zero rows.
- The `(created_at, id)` composite cursor eliminates the "missed row when many notifications share a second" bug that plain `LIMIT/OFFSET` has.

### 6.7 `visibility.ts`

Field-level redaction. Example: `notification.data_json` may contain a `vendorCommission` for internal notifications reflected into the customer's booking notification — `redact(identity, notification)` strips it before returning.

---

## 7 · Reliability layer

### 7.1 Producer contract

Every producer of a notification uses one and only one entry point:

```ts
await notificationsService.createNotification(
  conn,                          // ← the same connection the business tx runs on
  'booking.confirmed',
  { bookingId, destination, date },
  { userId, entityId, role: 'customer' },
);
```

Because it participates in the caller's `withTransaction()`, either:

- **Both** the booking write and the notification/outbox rows commit, or
- **Neither** does — a rollback in the booking service cannot leave a phantom notification behind.

### 7.2 Dispatcher worker

**Phase 1 (MVP):** the dispatcher runs **in-process** as an interval task inside the main PM2 worker cluster — one worker per PM2 instance polls the outbox using a lease pattern that is safe under concurrency:

```sql
-- Lease claim (transaction, SELECT ... FOR UPDATE SKIP LOCKED — MySQL 8+)
START TRANSACTION;
SELECT outbox_id
  FROM notification_outbox
 WHERE status IN ('pending','failed')
   AND next_attempt_at <= NOW()
   AND (leased_until IS NULL OR leased_until < NOW())
 ORDER BY priority = 'critical' DESC, priority = 'high' DESC, next_attempt_at ASC
 LIMIT 50
 FOR UPDATE SKIP LOCKED;

UPDATE notification_outbox
   SET status = 'processing',
       leased_by = ?,
       leased_until = DATE_ADD(NOW(), INTERVAL 30 SECOND),
       attempts = attempts + 1
 WHERE outbox_id IN (...);
COMMIT;
```

`SKIP LOCKED` means N workers can poll concurrently without collision or step-on-toes. Lease expiry (`leased_until < NOW()`) automatically reclaims rows from a worker that crashed mid-dispatch, so we never have "stuck processing forever" state.

Poll cadence: **500 ms** for `critical + high`, **2 s** for `normal + low` (two schedulers, same code).

**Phase 2 (post-MVP):** move the dispatcher to a **dedicated PM2 process** with `NOTIFICATION_WORKER=1` — same code, but not sharing CPU with the API. Requires no code changes: `server.ts` already has a `NOTIFICATION_WORKER` env branch that skips `server.listen()` and just runs the dispatcher.

**Phase 3 (scale):** replace outbox polling with a Kafka/RabbitMQ topic, with an outbox-relay CDC process (e.g., Debezium against `notification_outbox`) pushing rows into the topic. The **producer API does not change** at all — this is the value of the outbox pattern.

### 7.3 Idempotency & dedupe

Three layers of protection against duplicates:

1. **User-level, semantic:** `notification.uq_dedupe (user_id, dedupe_key)` — declared in the catalog per event. E.g. `booking.confirmed:{bookingId}` guarantees one notification per (user, booking), no matter how many times `createNotification` is called.
2. **Provider-level:** each push send carries `fcmOptions.analyticsLabel = notification_id` and a stable `google.messageId = hash(outbox_id + attempt)` so retries after network timeout do not double-deliver — FCM dedupes on message-id within a short window.
3. **Client-level:** the RN app records the last-seen `notification.id` in MMKV and drops the presented notification if the id has already been rendered — defense against FCM redelivery on marginal network.

### 7.4 Retries & DLQ

| Attempt | Delay | Reason class |
|---|---|---|
| 1 | 0 | first send |
| 2 | 30 s | transient (`INTERNAL`, `UNAVAILABLE`, `QUOTA_EXCEEDED`) |
| 3 | 2 min | transient |
| 4 | 10 min | transient |
| 5 | 1 h | transient |
| — | never | permanent (`UNREGISTERED`, `INVALID_ARGUMENT`) — device disabled immediately, outbox row → `failed` |

After 5 transient failures the row is set to `status = 'dead'`. A daily job (`notifications/jobs/dlqReport.ts`) emits a Pino warning with counts by `last_error_code` for on-call.

All delays include ±20% jitter (`delay * (0.8 + Math.random() * 0.4)`) to avoid retry thundering-herd after an FCM incident.

### 7.5 Preferences + quiet hours

Applied inside the dispatcher **after leasing but before provider send** — never at ingest. Rationale: preferences may change while the row sits in the outbox; enforcement at dispatch time is always current.

Quiet-hours override is `priority IN ('critical','high')` — OTPs and trip-emergency alerts break through. Marketing (`priority='low'`) additionally checks a per-user unsubscribe flag.

Quiet-hours check uses the user's stored `timezone`, not server time — a customer in Delhi at 2 AM sees no `payment.due` push whether the server sits in `us-east-1` or `ap-south-1`.

---

## 8 · Client-side design (RN app)

### 8.1 `services/notifications/` (new)

```
src/services/notifications/
├── index.ts                    # barrel
├── deviceInstallId.ts          # stable UUID minted once per install, kept in Keychain
├── permission.ts               # requestPermission() with platform quirks
├── registration.ts             # register + refresh flow talking to /notifications/devices
├── handlers.ts                 # onMessage / onNotificationOpenedApp / getInitialNotification
├── presenter.ts                # Notifee display + channels + categories
├── deeplink.ts                 # data → NavigationService route
└── types.ts                    # shared with backend via a manual mirror (no code-gen yet)
```

**`deviceInstallId`** is generated with `uuid.v4()` on first launch and stored in Keychain (same `SERVICE_NAME = 'urbancruise.auth'`, different key). Keychain-backed means it survives app updates but rotates on reinstall — exactly the semantic the backend needs to correlate "same physical install, possibly a new user".

### 8.2 Permission flow

**Android 13+ (API 33):** `POST_NOTIFICATIONS` runtime permission is required. Ask **at the moment of value**, not on cold start — e.g. right after login on the customer role's `CustomerHomeScreen`, or right before a booking is confirmed. Never in the middle of onboarding.

**iOS:** ask via `messaging().requestPermission()`; `authorizationStatus` returned. We also record `providesAppNotificationSettings` so a future "Notification Settings" deep-link in `SettingsScreen` works.

Add to `AndroidManifest.xml`:

```xml
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
<uses-permission android:name="android.permission.WAKE_LOCK" />
<uses-permission android:name="android.permission.VIBRATE" />

<meta-data
  android:name="com.google.firebase.messaging.default_notification_icon"
  android:resource="@drawable/ic_notification"/>
<meta-data
  android:name="com.google.firebase.messaging.default_notification_color"
  android:resource="@color/notification_accent"/>
<meta-data
  android:name="com.google.firebase.messaging.default_notification_channel_id"
  android:value="default"/>
```

Ship a monochrome `ic_notification.png` in all five drawable buckets. Android silently shows a white square if you don't.

### 8.3 Payload contract (data-only)

Backend always sends **data-only messages** — no top-level `notification` object. This is critical:

- Android with a `notification` block auto-renders in the system tray, bypassing `setBackgroundMessageHandler` — we lose click-tracking and cannot apply local mute/group logic.
- Data-only means our JS runs on every state, on every OS, guaranteed.

Payload shape:

```jsonc
{
  "data": {
    "v": "1",                                // schema version
    "notificationId": "01HXXX...",
    "category": "booking",
    "eventType": "booking.confirmed",
    "priority": "high",
    "title": "Booking confirmed",
    "body":  "Your trip to Jaipur on 12 Nov is confirmed.",
    "click": "{\"screen\":\"BookingDetail\",\"bookingId\":\"01HYYY...\"}",
    "collapseKey": "booking.confirmed:01HYYY"   // FCM-level collapse for rapid updates
  },
  "android": { "priority": "HIGH", "collapseKey": "..." },
  "apns":    { "headers": { "apns-priority": "10" }, "payload": { "aps": { "content-available": 1 } } }
}
```

`content-available: 1` on iOS is what wakes the app to run `setBackgroundMessageHandler` for data-only pushes.

### 8.4 Presentation via Notifee

Install `@notifee/react-native`. Create channels at app boot:

```ts
await notifee.createChannelGroup({ id: 'trip',    name: 'Trip updates' });
await notifee.createChannelGroup({ id: 'account', name: 'Account & security' });

await notifee.createChannel({
  id: 'trip.critical', groupId: 'trip', importance: AndroidImportance.HIGH,
  name: 'Trip critical', sound: 'trip_alert', vibration: true, bypassDnd: true,
});
await notifee.createChannel({ id: 'trip.normal', groupId: 'trip', importance: AndroidImportance.DEFAULT, name: 'Trip updates' });
await notifee.createChannel({ id: 'account.otp', groupId: 'account', importance: AndroidImportance.HIGH, name: 'One-time passwords' });
await notifee.createChannel({ id: 'marketing',   importance: AndroidImportance.LOW, name: 'Offers' });
```

Channels are a one-way street on Android — once created they cannot be modified programmatically (users own them from Settings). Version any channel id (`trip.normal.v2`) if you need to change sound/importance.

The presenter maps `category + priority → channelId`:

```ts
const channelId = channelForPayload(payload);  // deterministic

await notifee.displayNotification({
  id: payload.notificationId,                  // dedupe on the client too
  title: payload.title, body: payload.body,
  data:  payload,
  android: { channelId, smallIcon: 'ic_notification', pressAction: { id: 'default' } },
  ios:    { categoryId: payload.category, sound: 'default', threadId: payload.category },
});
```

### 8.5 Handler wiring

All three handlers **funnel to the same three functions**:

```ts
// index.ts of services/notifications
export function installNotificationHandlers() {
  // Foreground
  const unsub1 = messaging().onMessage(async remote => onData(remote.data, 'foreground'));

  // Background/quit (registered inside initFirebase — see below)
  // (already registered in bootstrap/steps/firebase.ts — replace stub)

  // Tap from background
  const unsub2 = messaging().onNotificationOpenedApp(remote => onOpen(remote.data, 'background'));

  // Cold launch from tap
  messaging().getInitialNotification().then(remote => { if (remote) onOpen(remote.data, 'quit'); });

  // Notifee tap events (foreground notification tapped)
  const unsub3 = notifee.onForegroundEvent(({ type, detail }) => {
    if (type === EventType.PRESS) onOpen(detail.notification?.data, 'foreground');
  });

  // Token refresh
  const unsub4 = messaging().onTokenRefresh(token => registerDevice(token));

  return () => { unsub1(); unsub2(); unsub3(); unsub4(); };
}
```

`initFirebase()` (in `bootstrap/steps/firebase.ts`) already sets `setBackgroundMessageHandler` as a stub — replace its body with `onData(payload, 'background')`. Notifee's `onBackgroundEvent(handler)` handles tap-in-background delivery.

**Deep-link on open** goes through the existing `NavigationService`. If the app isn't yet mounted (quit state), the payload is stashed in a module-scoped variable and consumed by `RootNavigator` on its first render — pattern already exists in the codebase for provisional-auth navigation.

### 8.6 In-app state sync

The Notification Centre is powered by TanStack Query on `endpoints.notifications.list()`. Two invalidation triggers keep it fresh without polling:

1. Every foreground/background `onData` invalidates `queryKeys.notifications.list` — the centre shows the new row the next time it's opened.
2. On app foreground (`AppState` change to `active`), invalidate the same key — catches notifications the app missed while backgrounded on iOS (which suspends JS).

### 8.7 Logout hygiene

`clearTokens()` (existing) is called during logout. Extend it: also call `DELETE /notifications/devices/:installId` **before** clearing tokens, and call `messaging().deleteToken()` afterward. The order matters — if we clear tokens first, the DELETE call is unauthenticated and fails silently, leaving a receiving zombie device.

---

## 9 · Security

| Threat | Mitigation |
|---|---|
| DB dump exposes device tokens | AES-256-GCM at rest with key in secrets vault. Only decrypted inside the dispatcher, never returned via API. |
| Token in logs | Pino redaction: `{ redact: ['*.token','*.token_ciphertext','req.headers.authorization', 'body.token'] }`. Redaction test as part of the module. |
| Cross-tenant read | `entity_id` filter on every read query. Tested by a repo-level test that runs each query as identity A and asserts zero rows returned for identity B's notification id. |
| Deep-link abuse (malicious payload -> app crash / arbitrary nav) | The `click` payload is a JSON string parsed with a Zod schema that whitelists allowed screens per role. Anything not matching → drop the deep-link, still display the notification. |
| Replay of stolen access token to register a device on another user | `POST /notifications/devices` writes with `user_id = identity.userId`, but a stolen access token is a general auth problem, not push-specific. Still: the `uq_fingerprint` guarantees at most one active row per physical token — a stolen token can only reroute pushes if the attacker's install completes a full re-registration flow, at which point the rightful user's next foreground ping steals it back. |
| Notification content is sensitive (OTP, payment amounts) | Title/body carry non-sensitive summaries only. The catalog enforces this — reviewing PRs against `catalog.ts` is the single security review touchpoint. iOS notification content is decrypted end-to-end optionally via a Notification Service Extension in v2. |
| Marketing spam via internal event | Marketing category requires `role: 'uc'` producer identity checked in the service (`if (spec.category === 'marketing' && producerRole !== 'uc') throw`). |
| Rate-limit abuse on `POST /devices` | Per-route rate limiter: 5/min per user. Registration is expected to be rare. |

Add to `config/env.ts`:

```ts
FIREBASE_SA_JSON_PATH: z.string().min(1),      // path to service-account JSON
PUSH_TOKEN_ENC_KEY:    z.string().length(64),  // 32 bytes hex-encoded
FCM_SEND_TIMEOUT_MS:   z.coerce.number().int().positive().default(10_000),
```

The service-account JSON is mounted read-only into the container/PM2 environment; no secret is in-repo.

---

## 10 · Observability

- **Structured logs (Pino):** every hop logs `{ notificationId, outboxId, eventType, userId, deviceInstallId, requestId }`. The request-id already flows via `AsyncLocalStorage` — the worker synthesizes one at lease time (`worker:<workerId>:<outboxId>`) so a background-generated line is greppable end-to-end.
- **Metrics:** exposed via a `/metrics` endpoint (Prom exposition format, mount inside the health module) with counters `notifications_created_total{category,event}`, `notifications_sent_total{provider,status}`, `notifications_send_latency_ms` histogram, `outbox_backlog{status,priority}` gauge polled by the health check. This is the minimum needed for a Grafana dashboard and on-call alerts.
- **Alerts** (worth wiring even at MVP): outbox `pending` older than 5 min, DLQ growth > 100/hour, `UNREGISTERED` rate > 5% (indicates client-side push registration regression).
- **Client Crashlytics:** `logError(err, { boundary: 'push.foreground.onData' })` in every handler catch — hooks into the existing `services/telemetry/logError.ts` which is already Crashlytics-ready.

---

## 11 · Rollout / phased delivery

| Phase | Deliverable | Effort | Ships when |
|---|---|---|---|
| **P0 · Foundation** | DDL migrations + `types.ts` + `catalog.ts` scaffold + Firebase Admin SDK wired + service-account secret + client `deviceInstallId` + `POST /devices` + `GET /preferences`, `PATCH /preferences`. No delivery yet. | 3–5 days | Before any dispatcher is written — foundations first. |
| **P1 · One event, end-to-end** | Pick `account.otp` (or `booking.confirmed`). Producer calls `createNotification`. Dispatcher polls, sends, records delivery. Client handles all three states. Notifee channels created. Deep-link works. | 5–7 days | First push land in a real device. |
| **P2 · The rest of the catalog** | Add remaining event types in `catalog.ts`. Retries + DLQ. Preferences + quiet hours enforcement. `GET /notifications` + centre screen UI. | 1–2 weeks | Notification Centre is real, not "Coming Soon". |
| **P3 · Vendor + driver + uc roles** | Extend `ENABLED_ROLES` in `authenticate.ts`. Add role-specific catalog entries. Role-aware channel setup on the client (`role` from Redux drives which channel-set is created). | Concurrent with each role's feature rollout. |
| **P4 · Metrics + alerts + DLQ job** | Prom endpoint, Grafana dashboard, PagerDuty rules. DLQ nightly report. Load test: 100k pushes in 5 min from a single dispatcher. | 3–4 days |
| **P5 · Scale-out (if/when needed)** | Move dispatcher to a dedicated PM2 process. If a single MySQL still keeps up, stop here. If not: introduce Redis for lease + rate-limit; then Kafka + outbox relay (Debezium) for durable queue. | On demand, driven by metrics — see §12. |

**No phase requires refactoring the previous phase.** The catalog + outbox pattern is the fixed pivot; everything else is additive.

---

## 12 · Scale evolution — explicit triggers

The design should stay MySQL-only until the numbers say otherwise. Concrete triggers:

| Signal | Response |
|---|---|
| `outbox_backlog{status="pending"}` > 5k for > 2 min during normal traffic | Move dispatcher to a dedicated PM2 process (P5.a). |
| Dispatcher CPU saturating a single process | Increase PM2 worker count for the dispatcher role; ensure `SKIP LOCKED` (already used) prevents duplicate leases. |
| MySQL row-lock waits > 1% of dispatcher queries | Move idempotency + dedupe to Redis (`SETNX notification:dedupe:{key} 1 EX 86400`); keep the DB unique key as defense-in-depth. |
| Fanout jobs (marketing: 100k+ users in one blast) start starving OTP path | Split into two dispatcher pools with dedicated priority queues; use separate PM2 processes bound to different priority ranges. |
| DB pool saturating from outbox polling | Introduce Kafka + Debezium CDC on `notification_outbox` as the source of truth; workers consume from Kafka. Application code (producer) unchanged. |
| Multi-region deployment needed | Provider fanout moves to a regional worker fleet; database stays single-region; region-aware routing keys added to catalog specs. |

**The producer API (`createNotification`) never changes across any of these steps.** That is the whole point of designing around a catalog + outbox contract from day one.

---

## 13 · Testing strategy

- **Unit:** every `catalog.ts` template — golden-output snapshot of rendered title/body/data for a fixed input. Zod schemas — reject malformed input. `channelForPayload` — deterministic mapping.
- **Repo:** tenant-scoping assertion — for every `SELECT`, run with identity A, insert as identity B, expect zero rows.
- **Integration:** in-memory MySQL + Firebase Admin stub. Producer → outbox → dispatcher → stubbed FCM → delivery row. Verify retry back-off. Verify DLQ transition. Verify UNREGISTERED → device_token disabled.
- **Contract:** the payload contract (§8.3) is a JSON Schema checked into the repo; both the backend renderer and the client parser validate against it. Any drift fails CI on both sides.
- **E2E (device):** a Detox test on Android + iOS that runs the app, receives a mocked push in foreground/background/quit, verifies navigation lands on the right screen.
- **Load:** k6 script simulating 10k `createNotification` calls in 30s + a dispatcher against a stub FCM; measure end-to-end p50/p95 latency.

---

## 14 · Open questions for review

1. **Notifee vs Firebase-only display.** Notifee adds a dependency and native code — but without it we lose foreground Android display and rich actions. Recommendation: **adopt Notifee**. Alternative: hand-roll a native module (not worth it).
2. **Do we ever need silent data pushes** (e.g. force a client cache refresh) beyond the `content-available: 1` we use to wake iOS for our data pushes? If yes, add a `silent` category in the catalog whose template returns no title/body and the client skips presentation.
3. **In-app real-time channel** (WebSocket / SSE) for users currently in the app — some products use this instead of foreground push for lower latency and no permission prompt. Deferring; add a `websocket` channel type in v2 if we build the real-time trip screen.
4. **Localization** — `title`/`body` are currently rendered server-side in a single language. When we add locales, move the templating client-side: server sends `{eventType, params}` and the client localizes via i18n keys. The catalog stays server-side authoritative; the render swaps sides.

---

## Appendix A — File additions checklist

**Backend:**

```
src/config/env.ts                          # add FIREBASE_SA_JSON_PATH, PUSH_TOKEN_ENC_KEY, FCM_SEND_TIMEOUT_MS
src/shared/crypto/tokenCipher.ts           # AES-256-GCM helpers
src/shared/rbac/policies.ts                # add update:Notification, acknowledge:Notification
src/modules/notifications/
  ├── catalog.ts                           # NEW — event registry
  ├── types.ts                             # implement
  ├── schemas.ts                           # implement
  ├── routes.ts                            # implement
  ├── controller.ts                        # implement
  ├── service.ts                           # implement
  ├── repository.ts                        # implement
  ├── policy.ts                            # implement
  ├── visibility.ts                        # implement
  ├── providers/
  │   └── fcm.ts                           # NEW — Firebase Admin adapter
  ├── dispatcher/
  │   ├── index.ts                         # NEW — poll loop, lease, priority scheduler
  │   ├── retry.ts                         # NEW — backoff + jitter
  │   └── quietHours.ts                    # NEW — timezone-aware check
  └── jobs/
      └── dlqReport.ts                     # NEW — daily
src/app.ts                                 # mount notificationsModule at /notifications
migrations/2026_XX_notifications.sql       # DDL from §5
```

**App:**

```
android/app/src/main/AndroidManifest.xml    # POST_NOTIFICATIONS + FCM meta-data
android/app/src/main/res/drawable-*/ic_notification.png
ios/pinaak/pinaak.entitlements              # aps-environment
ios/pinaak/NotificationService/              # NEW — NSE for rich content in v2
src/services/notifications/index.ts
src/services/notifications/deviceInstallId.ts
src/services/notifications/permission.ts
src/services/notifications/registration.ts
src/services/notifications/handlers.ts
src/services/notifications/presenter.ts
src/services/notifications/deeplink.ts
src/api/endpoints.ts                        # add devices + preferences endpoints
src/app/bootstrap/steps/firebase.ts         # replace stub BG handler with real one
src/features/shared/notifications/screens/NotificationCentreScreen.tsx   # real implementation
src/features/shared/notifications/hooks/useNotifications.ts              # NEW — TanStack Query
package.json                                # + @notifee/react-native, uuid
```
