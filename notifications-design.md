# Urban Cruise — Notification System Design

Status: Draft v1 · Scope: `urbancruisebackend` + `urbancruiseapp` · Transport: FCM (HTTP v1) primary, in-app centre secondary, SMS/email deferred.

This document is written against the code in both repos as of the current head. Everything below is grounded in what your app already declares: the role model in `src/rbac/roles.ts`, the deep-link catalog in `src/services/deeplinks/catalog.ts`, the endpoint registry in `src/api/endpoints.ts`, the RBAC matrix in `src/shared/rbac/policies.ts`, the visibility rules in `src/rbac/visibility.ts`, and the branded ID + Money types in `src/types/`. Where a decision could reasonably go two ways, both options are named and one is chosen with a rationale.

---

## 1. Goals and non-goals

**Goals.**
- Deliver time-sensitive, role-scoped push notifications for every state transition that a customer, vendor, driver or UC staff needs to act on.
- Guarantee at-least-once delivery of transactional notifications (a booking assignment, a driver-arrival alert, a payment success) even when the recipient's device is offline, the FCM call fails, or the backend restarts mid-fanout.
- Reuse the existing typed deep-link surface so a notification tap lands on the exact screen with validated params — no new navigation vocabulary.
- Give the notification centre a durable server-side record so users can reopen alerts from history, not just their OS shade.
- Keep the compile-time contract between backend and app tight: one `NotificationKind` enum drives payload validation on both sides.

**Non-goals for v1.**
- SMS and email fallback. The channel abstraction is designed so these plug in later; only FCM is implemented.
- In-app real-time streams (websocket). The Notification Centre pulls; TanStack Query already handles that pattern in your app.
- Marketing / campaign broadcasts. This system carries **transactional** notifications only. Marketing gets its own module later so opt-outs stay clean under DPDP.
- Rich media attachments in the push itself (image URLs are fine; downloading large blobs from the notification is out of scope).

---

## 2. Business events — the authoritative catalogue

Extracted from `src/api/endpoints.ts`, the deep-link catalogue, `src/rbac/visibility.ts`, and the module folder structure under `urbancruisebackend/src/modules/`. This is the ground-truth list of what triggers a notification and to whom. Every row here becomes a `NotificationKind` in section 8.

### 2.1 Customer-facing

| Event | Trigger | Deep-link kind | Priority | Rationale |
|---|---|---|---|---|
| Booking created & awaiting acknowledgement | Customer submits booking; server writes `pendingAck` | `customer.bookingDetail` | high | Kicks off the ack SLA the visibility rules assume. |
| Quotation ready | UC creates quotation against enquiry (`POST /uc/enquiries/:id/quotations`) | `customer.quotationDetail` | high | Purchase intent — response window matters. |
| Quotation revised | UC updates quotation | `customer.quotationDetail` | high | Cheaper than losing the deal. |
| Booking assignment approved (driver + vehicle visible) | UC sets `ucApprovedAssignment=true` — the exact flag `isCustomerDriverRevealed`/`isCustomerVehicleRevealed` gate on | `customer.bookingDetail` | high | This is the moment the customer sees who's picking them up. |
| Vehicle change approved | `POST /uc/trips/:id/vehicle-change/approve` | `customer.bookingDetail` | normal | Trust: customer must know the ride changed. |
| Driver en route / arrived | Driver leg-start event (`POST /driver/trips/:id/legs/:leg/start` for leg 1) | `customer.tripLive` | high | Time-critical; drives live-track screen. |
| Trip started (OTP verified) | `POST /driver/trips/:id/otp` | `customer.tripLive` | normal | Confirmation. |
| Trip completed | `POST /driver/trips/:id/legs/:leg/end` for final leg | `customer.feedback` | normal | Feedback capture window opens. |
| Balance payment due | Reconciliation cron finds outstanding balance ≤ N hours from trip end | `customer.payBalance` | high | Money. |
| Payment received | `POST /customer/bookings/:id/payments/pay` success | `customer.bookingDetail` | normal | Receipt. |
| GST invoice ready | Invoice job finishes | `customer.bookingDetail` | low | Non-blocking. |
| Feedback reminder | Trip ended > 24h, no feedback | `customer.feedback` | low | Once only, respects quiet hours. |
| Cancellation confirmed | Booking status → `cancelled` | `customer.bookingDetail` | high | Money implications — refund status included. |

**Corporate sub-role fan-out.** From `roles.ts`: a corporate customer has multiple logins (`admin`, `bookingPerson`) sharing one `CustomerId`. Section 9.3 defines who receives what.

### 2.2 Vendor-facing

| Event | Trigger | Deep-link kind | Priority |
|---|---|---|---|
| New assignment (booking offered) | `POST /uc/bookings/:id/assign` targeting this vendor | `vendor.assignmentDetail` | high |
| Assignment reminder | 30 min elapsed, still not accepted | `vendor.assignmentDetail` | high |
| Assignment cancelled by UC | UC retracts an offer | `vendor.assignmentDetail` | normal |
| Driver approval requested | `POST /vendor/drivers` (UC-side must approve) | (link into own drivers list; deep-link kind TBD if needed) | normal |
| Vehicle-change decision | `POST /uc/trips/:id/vehicle-change/approve` or reject | `vendor.tripDetail` | high |
| Payout initiated | `POST /vendor/payments/payouts` | (open vendor payments; deep-link kind TBD) | normal |
| Driver marked customer-collected cash | `POST /vendor/payments/customer-collected` | (payments listing) | normal |

Vendor has 4 sub-roles per `VendorSubRole` (`owner`, `bookingManager`, `opsManager`, `accountsManager`). Assignment notifications route to `bookingManager` + `opsManager`; payout notifications route to `accountsManager` + `owner`. See section 9.3.

### 2.3 Driver-facing

| Event | Trigger | Deep-link kind | Priority |
|---|---|---|---|
| New trip assigned | Vendor assigns driver to a trip leg | `driver.tripDetail` | high |
| Trip starts in T-6h / T-1h | Scheduler | `driver.tripDetail` | high |
| Customer contact unlocked (visibility RULE 1) | The moment `isDriverCustomerContactUnlocked` flips true (T-24h and assigned/ongoing) | `driver.tripDetail` | normal |
| Vehicle change approved / rejected | UC decision | `driver.tripDetail` | normal |
| Collect balance from customer | Trip ending with cash-collect flag | `driver.collectPayment` | high |
| Cancelled by UC | Booking cancelled after driver assigned | `driver.tripDetail` | high |

### 2.4 UC (internal ops)

| Event | Trigger | Deep-link kind | Priority |
|---|---|---|---|
| New enquiry submitted | `POST /customer/enquiries` | `uc.enquiryDetail` | high |
| Quotation confirmed by customer | `POST /customer/quotations/:id/confirm` | `uc.enquiryDetail` (or booking detail once split) | high |
| Assignment SLA breach | Vendor didn't accept in T minutes | `uc.tripMonitor` (or bookings monitor) | high |
| Vehicle-change request from vendor | `POST /vendor/trips/:id/vehicle/change-request` | `uc.tripMonitor` | high |
| Driver / vendor sign-up awaiting approval | Registration flows | (deep-link kind TBD) | normal |
| Payment reconciliation exception | Finance job | (deep-link kind TBD) | high |
| Trip issue raised | Support flow | `uc.tripMonitor` | high |

### 2.5 Cross-role

| Event | Kinds |
|---|---|
| Support ticket update | `common.support` |
| Generic system announcement (per-role) | `common.notificationCentre` |

Note two deep-link kinds are marked "TBD". These will be added to `src/services/deeplinks/schema.ts` and `catalog.ts` alongside their notifications — the compile-time exhaustiveness check in `toNavigate.ts` will refuse to build until the case is handled, which is exactly the safety net we want.

---

## 3. Delivery channels and where each concern lives

Three channels are recognised by the design, only the first two are implemented in v1.

**Channel 1 — FCM push.** The primary channel. Data-only payloads (see section 8) so the app renders through Notifee for consistent styling and channel behaviour on Android, and consistent action handling on both platforms. The FCM `notification` block is not used, because a `notification`-block message delivered while the app is in the background bypasses the JS handler on Android and is displayed by the OS with our default channel — killing our category / channel routing.

**Channel 2 — In-app centre.** Every notification enqueued for push is also persisted to `notifications` and served by the existing `/notifications` endpoints. This is the audit trail and the recovery mechanism when a device is offline or the user has push disabled.

**Channel 3 (deferred) — SMS / email.** The `Channel` enum in the schema accounts for these values so column semantics don't change when they're added.

---

## 4. High-level architecture

```
                          ┌────────────────────────────────────────────┐
  Business modules ──────▶│  NotificationService.enqueue({...})        │
  (auth / customer /      │                                            │
   vendor / driver / uc)  │  1. Resolve recipients (identity fan-out)  │
                          │  2. Apply preferences + quiet hours        │
                          │  3. Persist Notification rows              │
                          │  4. Persist NotificationOutbox rows        │
                          │     (in the SAME transaction as step 3)    │
                          └───────────────┬────────────────────────────┘
                                          │
                                    withTransaction commit
                                          │
                                          ▼
                           ┌──────────────────────────────┐
                           │ NotificationWorker (poll)    │
                           │  claim → dispatch → mark     │
                           └──────────┬───────────────────┘
                                      │
                        ┌─────────────┼──────────────┐
                        ▼             ▼              ▼
                    FCM adapter   (SMS adapter)  (Email adapter)
                        │             (later)       (later)
                        ▼
                    FCM HTTP v1 (google-auth-library)
                        │
                        ▼
                 Device tokens (per user × device)
                        │
                        ▼
                 App onMessage / background handler
                 → Notifee → user tap → deep-link resolver
                 → typed target → navigation
```

Key architectural choices:

**Transactional outbox.** A business service writing "assignment approved → notify" cannot risk the notification succeeding while the DB commit fails, or vice versa. Row inserts into `notifications` and `notification_outbox` share the business `withTransaction` boundary (the utility already exists at `src/shared/db/transaction.ts`). The worker picks up committed outbox rows asynchronously. This is the only way to hit at-least-once without distributed transactions.

**Poller, not queue, for v1.** The infrastructure is MySQL + PM2 + Nginx today (no Redis, no SQS). A short-interval `SELECT ... FOR UPDATE SKIP LOCKED` poll is boring and correct at Urban Cruise's scale. If throughput ever demands it, the outbox becomes a shim in front of a real broker without changing service-layer callers.

**One module.** `src/modules/notifications/` follows the same layout as every other module in the backend (controller / service / repository / routes / schemas / policy / visibility / types). New submodules inside it: `devices/`, `dispatch/`, `preferences/`.

---

## 5. Data model

MySQL, matching the conventions used in `refresh_tokens` (see `src/shared/auth/tokens.ts` header comment) — CHAR(36) UUID primary keys, DATETIME columns as UTC (`timezone: 'Z'` is set in `src/shared/db/pool.ts`), no ENUMs in DDL (use VARCHAR + application-level Zod enums so kind additions don't require ALTERs).

### 5.1 `notification_devices`

Registered push tokens. One row per (user × device × platform). The critical uniqueness is on the token itself — FCM guarantees a given token maps to exactly one device+app+installation, and reassigning a token to a different user is a common cause of "user A sees user B's notifications" bugs.

```sql
CREATE TABLE notification_devices (
  id            CHAR(36)     PRIMARY KEY,
  user_id       CHAR(36)     NOT NULL,
  role          VARCHAR(16)  NOT NULL,          -- 'customer' | 'vendor' | 'driver' | 'uc'
  sub_role      VARCHAR(32)  NULL,              -- CorporateSubRole | VendorSubRole | NULL
  entity_id     CHAR(36)     NOT NULL,          -- customerId / vendorId / driverId / ucUserId
  platform      VARCHAR(8)   NOT NULL,          -- 'ios' | 'android'
  token         VARCHAR(255) NOT NULL,          -- FCM registration token
  app_version   VARCHAR(24)  NOT NULL,
  device_model  VARCHAR(64)  NULL,
  os_version    VARCHAR(24)  NULL,
  locale        VARCHAR(16)  NULL,              -- 'en-IN', 'hi-IN' — for future SMS/email
  last_seen_at  DATETIME     NOT NULL,          -- refreshed on every /devices heartbeat
  created_at    DATETIME     NOT NULL,
  disabled_at   DATETIME     NULL,              -- set on logout OR on FCM 404/NotRegistered
  UNIQUE KEY uk_token (token),
  KEY idx_user_active (user_id, disabled_at),
  KEY idx_last_seen (last_seen_at)
);
```

`disabled_at` is a soft flag, not a delete, because the same token can come back tomorrow after a `getToken` refresh on the client and we want to know whether it was "user logged out" or "FCM invalidated it".

### 5.2 `notifications` — the durable record

```sql
CREATE TABLE notifications (
  id              CHAR(36)     PRIMARY KEY,
  user_id         CHAR(36)     NOT NULL,        -- recipient
  role            VARCHAR(16)  NOT NULL,        -- snapshotted for audit
  kind            VARCHAR(64)  NOT NULL,        -- e.g. 'booking.assigned'
  category        VARCHAR(24)  NOT NULL,        -- 'transactional' | 'reminder' | 'system'
  priority        VARCHAR(8)   NOT NULL,        -- 'high' | 'normal' | 'low'
  title           VARCHAR(120) NOT NULL,
  body            VARCHAR(240) NOT NULL,
  click_target    JSON         NOT NULL,        -- DeepLinkTarget — matches the Zod union
  data            JSON         NULL,            -- extra payload (money, counters, etc.)
  correlation_id  CHAR(36)     NOT NULL,        -- one notify() call may fan out to N rows;
                                                --   they all share correlation_id
  request_id      CHAR(36)     NULL,            -- pino/pino-http request that produced it
  read_at         DATETIME     NULL,
  seen_at         DATETIME     NULL,            -- shown in centre list (weaker than read)
  created_at      DATETIME     NOT NULL,
  KEY idx_user_created (user_id, created_at),
  KEY idx_user_unread (user_id, read_at),
  KEY idx_correlation (correlation_id)
);
```

The `click_target` column stores exactly the shape the app's `DeepLinkTarget` Zod union expects. Section 8 covers how it's serialised into the FCM `data.click` field the app already looks for in `src/services/notifications/deeplink.ts`.

### 5.3 `notification_outbox`

```sql
CREATE TABLE notification_outbox (
  id                 CHAR(36)     PRIMARY KEY,
  notification_id    CHAR(36)     NOT NULL,     -- FK to notifications.id
  channel            VARCHAR(16)  NOT NULL,     -- 'push' (v1) | 'sms' | 'email' (later)
  device_id          CHAR(36)     NULL,         -- for 'push' — the specific device targeted
  status             VARCHAR(16)  NOT NULL,     -- 'pending' | 'inflight' | 'sent' | 'failed' | 'dropped'
  attempt_count      INT UNSIGNED NOT NULL DEFAULT 0,
  next_attempt_at    DATETIME     NOT NULL,     -- exponential backoff cursor
  last_error_code    VARCHAR(64)  NULL,         -- e.g. 'UNREGISTERED', 'QUOTA_EXCEEDED'
  last_error_at      DATETIME     NULL,
  locked_by          CHAR(36)     NULL,         -- worker instance id claiming this row
  locked_until       DATETIME     NULL,         -- lease expiry
  provider_message_id VARCHAR(128) NULL,        -- FCM 'name' from HTTP v1 response
  sent_at            DATETIME     NULL,
  created_at         DATETIME     NOT NULL,
  KEY idx_ready (status, next_attempt_at),
  KEY idx_notification (notification_id)
);
```

One outbox row per (notification × device). A user with three devices produces three push outbox rows sharing a `notification_id`. Deduping is a device-level concern (section 8.4), not an outbox concern.

### 5.4 `notification_preferences`

Category-level opt-outs, per user. Transactional categories can be listed but the service always ignores opt-out for `category='transactional'` and `priority='high'` — an operator cannot silently opt out of the "your driver has arrived" push. The row exists so the UI can show it as "always on, cannot be disabled" honestly.

```sql
CREATE TABLE notification_preferences (
  user_id      CHAR(36)     NOT NULL,
  category     VARCHAR(24)  NOT NULL,
  channel      VARCHAR(16)  NOT NULL,
  enabled      TINYINT(1)   NOT NULL DEFAULT 1,
  quiet_start  TIME         NULL,               -- user-local, per section 10
  quiet_end    TIME         NULL,
  updated_at   DATETIME     NOT NULL,
  PRIMARY KEY (user_id, category, channel)
);
```

### 5.5 Migration ordering

Migrations run in this order, because `notification_outbox.notification_id` and `notifications.user_id` foreign-key against the `users` table (created by the auth module's own migrations, currently stubbed):

1. `users` (owned by auth module)
2. `notification_devices`
3. `notifications`
4. `notification_outbox`
5. `notification_preferences`

No FK cascade on delete for users. Notifications are business records; the retention policy (section 15) governs deletion, not cascading from user deletion.

---

## 6. Backend module layout

```
src/modules/notifications/
├── index.ts                 # mount() — same convention as health module
├── routes.ts                # HTTP surface
├── controller.ts            # HTTP layer only (parses validated req; returns via ok/paginated)
├── service.ts               # NotificationService — enqueue / list / mark read
├── repository.ts            # SQL only (mysql2 pool.execute with ?)
├── schemas.ts               # Zod schemas: request bodies + NotificationKind + payload shapes
├── policy.ts                # canOperate (only the owner reads their notifications)
├── visibility.ts            # response DTO projection (never leak internal columns)
├── types.ts                 # row shapes + DTOs
│
├── devices/                 # sub-feature: device registration
│   ├── routes.ts            # POST /notifications/devices, DELETE /notifications/devices/:id
│   ├── controller.ts
│   ├── service.ts
│   ├── repository.ts
│   ├── schemas.ts
│   └── types.ts
│
├── preferences/
│   ├── routes.ts            # GET / PATCH /notifications/preferences
│   ├── controller.ts
│   ├── service.ts
│   ├── repository.ts
│   └── schemas.ts
│
└── dispatch/                # server-internal, no HTTP surface
    ├── kinds.ts             # NotificationKind enum + payload Zod schemas per kind
    ├── renderer.ts          # kind + payload → { title, body, clickTarget }
    ├── recipients.ts        # kind + payload → Identity[]  (fan-out logic)
    ├── enqueue.ts           # the transactional writer used by every business service
    ├── worker.ts            # poller: claim outbox rows, dispatch, retry, mark
    └── providers/
        ├── fcm.ts           # HTTP v1 adapter using google-auth-library
        ├── fcmClient.ts     # low-level: token bearer, POST /messages:send
        └── errors.ts        # error-code → outbox retry/kill decision
```

The `enqueue.ts` module is the sole way business code creates notifications. Every service in `customer/`, `vendor/`, `driver/`, `uc/` imports it and calls `notify(...)` from inside its `withTransaction` block. This preserves atomicity and gives one code path to audit.

New env vars added to `src/config/env.ts`:

```
FCM_PROJECT_ID              string, required in prod
FCM_SERVICE_ACCOUNT_JSON    string, required in prod — the JSON key, single-line
                            (or FCM_SERVICE_ACCOUNT_PATH for file-based deploys)
NOTIF_WORKER_ENABLED        bool,  default true
NOTIF_WORKER_POLL_MS        int,   default 2000
NOTIF_WORKER_BATCH_SIZE     int,   default 50
NOTIF_WORKER_MAX_ATTEMPTS   int,   default 6      -- ~ 30 min total with backoff below
NOTIF_WORKER_LEASE_MS       int,   default 30000
```

Poller backoff schedule for `attempt_count`: 2s, 10s, 60s, 5m, 15m, 30m (then `status='failed'`, `dropped_reason` logged). High-priority kinds get the same schedule; a shorter one buys nothing when FCM is genuinely down.

---

## 7. Device token lifecycle

FCM tokens are the moving part most likely to cause "why didn't my notification arrive" tickets. The rules below are non-negotiable.

**Register after authentication, never before.** Registration is done from the `runBootstrap` flow in `src/app/bootstrap/index.ts`, but only inside a new `steps/deviceRegistration.ts` step that runs after `authResult.status === 'authenticated'`. If auth is `provisional`, we defer — the interceptor at `src/api/interceptors/refresh` will resolve the session shortly and a resume-time hook re-attempts registration.

**Refresh on token rotation.** FCM rotates tokens (app reinstall, data cleared, refresh triggered by GCM). Register a listener with `onTokenRefresh` inside the firebase bootstrap step and POST the new token to the same endpoint. Server treats `POST /notifications/devices` as an upsert keyed on `token`; if the same token appears for a different `user_id` the old row is `disabled_at`-flagged before the new one inserts.

**Deregister on logout.** `DELETE /notifications/devices/{id}` on interactive logout, before the tokens leave Keychain. On failure (offline logout), enqueue a local pending action in MMKV that retries on next app-start — a stale token still on the server is a leak of ex-user data.

**Deregister on 404 / NotRegistered.** When the FCM adapter (`providers/fcm.ts`) receives HTTP 404 or the `UNREGISTERED` / `INVALID_ARGUMENT` error code for a token, it flips the device row's `disabled_at` immediately. The outbox row is marked `dropped` (not `failed`) — retrying an unregistered token is a permanent waste.

**Heartbeat via authenticated calls.** On every successful `/auth/me` or explicit `/notifications/devices/heartbeat`, `last_seen_at` is bumped. A weekly job disables rows with `last_seen_at < now - 90 days` — an inactive install cannot receive pushes anyway.

Endpoints added to `src/api/endpoints.ts`:

```ts
notifications: {
  list:        () => '/notifications',
  markRead:    (id) => `/notifications/${id}/read`,
  markAllRead: () => '/notifications/read-all',
  devices:     {
    register:   () => '/notifications/devices',
    deregister: (id) => `/notifications/devices/${id}`,
    heartbeat:  () => '/notifications/devices/heartbeat',
  },
  preferences: {
    get:    () => '/notifications/preferences',
    update: () => '/notifications/preferences',
  },
},
```

---

## 8. FCM payload contract

The app's existing bridge, `src/services/notifications/deeplink.ts`, already looks for `data.click` as a JSON-encoded `DeepLinkTarget` and calls `handleFcmClick` which runs it through the exact same Zod union that URL-borne deep links run through. This design keeps that contract and formalises everything else around it.

### 8.1 Message shape (data-only)

```jsonc
{
  "message": {
    "token": "<device token>",
    "data": {
      "kind":         "booking.assigned",              // NotificationKind — closed enum
      "notifId":      "b48d5b9c-...",                  // notifications.id
      "correlation":  "cb812fc7-...",                  // notifications.correlation_id
      "title":        "Booking assigned",              // pre-rendered, PII-safe
      "body":         "Vendor confirmed. Vehicle KA-05-1234, driver Rajesh.",
      "click":        "{\"kind\":\"customer.bookingDetail\",\"bookingId\":\"BKG20260905A\"}",
      "priority":     "high",
      "category":     "transactional",
      "channelId":    "trip_alerts"                    // Android — matches the app's Notifee channel
    },
    "android": {
      "priority": "HIGH",                              // OR "NORMAL" for low-prio
      "ttl":      "3600s",                             // per section 8.5
      "collapse_key": "booking-b48d5b9c"               // optional; see section 8.4
    },
    "apns": {
      "headers": {
        "apns-priority":  "10",                        // '5' for low-prio
        "apns-push-type": "alert",
        "apns-expiration": "<unix-epoch of ttl>",
        "apns-collapse-id": "booking-b48d5b9c"
      },
      "payload": {
        "aps": {
          "alert": { "title": "...", "body": "..." },
          "sound": "default",
          "content-available": 1,
          "mutable-content": 1
        }
      }
    }
  }
}
```

**Why data-only, no `notification` block?** As covered in section 3: on Android, when a message includes a `notification` block and the app is in the background/killed, JS never sees it — the OS auto-displays it via the default channel with default styling. That breaks our channel-per-category strategy and the deep-link resolver never runs. Notifee renders every notification consistently on foreground and background.

### 8.2 Android channels (Notifee)

Registered once inside the firebase bootstrap step, keyed by category+priority so importance survives OEM tweaks:

| channelId | Importance | Sound | Categories that use it |
|---|---|---|---|
| `trip_alerts` | HIGH | default | driver-assigned, arrival, OTP, cancel |
| `booking_updates` | HIGH | default | assignment approved, quotation ready, payment due |
| `payments` | DEFAULT | default | payment success, invoice ready, payout initiated |
| `system` | LOW | none | announcements, feedback reminder |

Categories not shown to the user (they see the higher-level toggles in preferences UI). No `channelGroups` split by role — a device serves exactly one authenticated user at a time.

### 8.3 iOS specifics

- APNs entitlement + push cert already assumed configured (firebase.json shows `messaging_auto_init_enabled: true`).
- `content-available: 1` allows a limited JS wakeup on iOS 15+ for silent data sync. Do not rely on it for state updates — the OS throttles it.
- Category actions (e.g. "Accept" / "Reject" on a driver-assignment notification) can be layered in a future revision using `apns.payload.aps.category` + a UNNotificationCategory registered in the app's `AppDelegate`. Out of scope for v1 to keep the surface small.
- Notification Service Extension is not needed for v1 (no rich media, no per-message decryption).

### 8.4 Collapsing and deduplication

`collapse_key` (Android) and `apns-collapse-id` (iOS) suppress duplicates when multiple pushes about the same subject arrive while the device is offline. Rules:

- Booking-level pushes: `collapse_key = "booking-{bookingId}"`. If a user misses "assignment created" and then "assignment approved" while offline, only the second one shows.
- Trip-level pushes: `collapse_key = "trip-{tripId}"`. Same reasoning.
- Payment pushes: no collapse — a paid + then refunded notification are distinct events.
- System pushes: no collapse.

Deduplication in-app: on receive, look up `data.notifId` in the local MMKV `seen_notifications` set (size-bounded LRU, ~200 entries) before calling Notifee. FCM occasionally redelivers the same message across app reinstalls or after a network partition.

### 8.5 TTL

Set per notification, from the outbox row rather than a global default:

- High-priority transactional (assigned / arrival / OTP / collect): 1 hour. If the device isn't online in an hour the event is stale — the customer app will refetch the current state on next foreground, driven by TanStack Query. A late push here misleads.
- Normal-priority: 24 hours.
- Low-priority (feedback reminder, announcement): 72 hours.

FCM's default is 4 weeks — never use it for transactional traffic.

---

## 9. Enqueue, fan-out, and quiet-hours logic

`notify()` is the single entrypoint. Signature (types elided):

```ts
export interface NotifyInput<K extends NotificationKind> {
  kind:        K;
  payload:     NotificationPayload<K>;   // Zod-validated against dispatch/kinds.ts
  correlation: string;                    // caller-owned; usually the domain entity's id
  requestId?:  string;                    // pulled from getRequestContext() if omitted
  conn:        PoolConnection;            // MUST be inside a transaction the caller owns
}
```

### 9.1 Ordered pipeline

1. **Validate payload** against the Zod schema for that kind. Throws `ValidationError` — same error taxonomy every other module uses.
2. **Resolve recipients** — `recipients.ts` maps `(kind, payload)` to `Identity[]`. This is where corporate + vendor sub-role fan-out lives (9.3).
3. **Render** — `renderer.ts` produces `{ title, body, clickTarget }` from `(kind, payload, recipient locale)`. Templates live per kind, plain strings with variables — no MJML, no HTML, kept intentionally boring.
4. **Filter by preferences and quiet hours** — 9.4.
5. **Insert `notifications` rows** — one per accepted recipient — sharing `correlation_id`.
6. **Insert `notification_outbox` rows** — one per `(notification × active device)` for channel `push`. Devices where `disabled_at IS NULL` and `role/entity_id` match the recipient identity.
7. **Return** — the caller's transaction commits both business changes and notification rows atomically. Any failure rolls back everything.

### 9.2 Idempotency

`notify()` is idempotent per `(kind, correlation)` scoped to a short window (e.g. 30 seconds). Implemented by:

```sql
INSERT IGNORE INTO notification_idempotency (kind, correlation_id, first_seen_at)
VALUES (?, ?, NOW());
```

If the `INSERT IGNORE` affects 0 rows, `notify()` returns early with a `duplicate` outcome. The idempotency window matters because business services can end up calling `notify()` twice on retry (network hiccup between service and DB, or client-retry hitting a not-yet-committed request). Rare, but corrosive to trust when it does happen — a user seeing "your driver has arrived" twice within a second is the memorable class of bug.

### 9.3 Recipient fan-out — the sub-role matrix

| Kind | Recipients |
|---|---|
| `enquiry.*`, `quotation.*` on customer side | Every user under the CustomerId. For personal + agent customers that's one user. For corporate: `admin` + `bookingPerson` both receive. |
| `payment.due`, `payment.receiptReady`, `invoice.ready` | Corporate: `admin` only. Personal/agent: the owner. |
| `booking.assignmentApproved`, `trip.driverArrived`, `trip.otpVerified`, `trip.completed` | Corporate: the `bookingPerson` who created the booking + all `admin`s. Personal/agent: the owner. |
| `booking.cancelled` | Everyone with visibility to that booking (same set as approved event). |
| Vendor `assignment.*` | All `bookingManager` + `opsManager` users under that VendorId. |
| Vendor `payment.*` | `owner` + all `accountsManager`. |
| Driver kinds | The single driver user tied to `DriverId`. |
| UC kinds | Route via a per-kind UC subscription table (`uc_notification_subscriptions`, keyed by kind + ucUserId). Ops teams grow; hardcoding is wrong. Default subscription list bootstrapped from `role='uc'` for kinds that must never be missed (`assignment.slaBreach`, `payment.reconciliationException`). |

The mapping is expressed in `dispatch/recipients.ts` as a `switch(kind)` returning an `Identity[]`. TypeScript exhaustiveness on the `NotificationKind` union means adding a kind without wiring recipients is a compile error.

### 9.4 Preferences and quiet hours

- `category='transactional'` AND `priority='high'`: bypasses opt-outs and quiet hours. Written to `notifications`, dispatched immediately.
- `category='transactional'` AND `priority='normal'`: bypasses opt-outs but observes quiet hours (deferred to `quiet_end`).
- `category='reminder'` or `'system'`: observes opt-outs and quiet hours.

Quiet hours are stored per user in `notification_preferences` in the user's local time, along with the IANA timezone from the user profile (add a `timezone` column to `users` if not already there — default `Asia/Kolkata` given the target market). Deferral: the outbox row is inserted with `next_attempt_at` set to the next tick after `quiet_end` in that user's timezone. The worker naturally picks it up then.

---

## 10. Worker semantics

`worker.ts` is a per-process background loop started from `src/server.ts` (or a dedicated PM2 process — see 10.4). Every `NOTIF_WORKER_POLL_MS` (default 2s) it does one batch pass.

### 10.1 Claim pattern

```sql
START TRANSACTION;

SELECT id
FROM notification_outbox
WHERE status = 'pending'
  AND next_attempt_at <= NOW()
ORDER BY next_attempt_at
LIMIT :batch
FOR UPDATE SKIP LOCKED;   -- MySQL 8+, essential for multi-worker safety

UPDATE notification_outbox
SET status = 'inflight',
    locked_by = :workerId,
    locked_until = DATE_ADD(NOW(), INTERVAL :leaseMs/1000 SECOND),
    attempt_count = attempt_count + 1
WHERE id IN (...);

COMMIT;
```

Rows are then dispatched outside the transaction (network call). On success or terminal failure, a final `UPDATE` marks them `sent` / `failed` / `dropped`.

`SKIP LOCKED` is the correct primitive — two workers racing for the same batch never block each other, never process the same row twice.

### 10.2 Retry and terminal decisions

The FCM error → outbox status mapping (implemented in `providers/errors.ts`):

| FCM error | Meaning | Outbox action |
|---|---|---|
| 200 OK | Delivered to FCM | `sent` |
| `UNREGISTERED` / 404 | Token invalid | `dropped`; device row `disabled_at = NOW()` |
| `INVALID_ARGUMENT` on `token` | Malformed token | `dropped`; device disabled |
| `INVALID_ARGUMENT` on payload | Our bug | `failed` immediately; log at `error` level; alert |
| `SENDER_ID_MISMATCH` | Wrong FCM project | `dropped`; device disabled; alert |
| `QUOTA_EXCEEDED` | Rate limited | `pending`; backoff × 4 for this attempt |
| `UNAVAILABLE` / 5xx | FCM transient | `pending`; normal backoff |
| Network timeout | Transient | `pending`; normal backoff |
| `attempt_count >= NOTIF_WORKER_MAX_ATTEMPTS` | Exhausted | `failed`; log; the in-app centre still shows the notification |

The centre remains the safety net: even if push fails permanently, the user opening the app sees the alert in their list.

### 10.3 Lease reclamation

A worker that crashes mid-dispatch leaves a row in `inflight` with `locked_until` in the past. A separate "reaper" query (run once per poll cycle) resets those to `pending`:

```sql
UPDATE notification_outbox
SET status = 'pending', locked_by = NULL, locked_until = NULL
WHERE status = 'inflight' AND locked_until < NOW();
```

### 10.4 Process placement — one worker per instance, not per PM2 worker

Under PM2 cluster mode, running the poller in every Node worker is wasteful (they'll compete for the same rows; `SKIP LOCKED` makes it correct but not free). Two options:

- **Simple (v1):** run the poller only when `process.env.NODE_APP_INSTANCE === '0'` (PM2 sets this per cluster worker). Simplest possible thing that works.
- **Cleaner:** a dedicated `pm2` process defined in `ecosystem.config.cjs`, `instances: 1`, `script: dist/workers/notifications.js`, sharing the same DB pool config. Recommended once traffic warrants a second instance.

---

## 11. Frontend integration

Everything below hooks into structures already in the app. No new architectural primitives.

### 11.1 New bootstrap step — `steps/deviceRegistration.ts`

Called from `runBootstrap` after `authResult.status === 'authenticated'`:

```ts
export async function registerDeviceForPush(identity: Identity) {
  // 1. Ensure notifications capability — RBAC-aware, already implemented
  const status = await ensureCapability('notifications');
  if (status !== 'granted') return;

  // 2. Get the FCM token (modular API — messaging is already initialised)
  const messaging = getMessaging(getApp());
  const token = await getToken(messaging);
  if (!token) return;

  // 3. Register with server (upsert)
  await apiClient.post(endpoints.notifications.devices.register(), {
    token,
    platform: Platform.OS,               // 'ios' | 'android'
    appVersion: DeviceInfo.getVersion(),
    deviceModel: DeviceInfo.getModel(),
    osVersion: DeviceInfo.getSystemVersion(),
    locale: I18nManager.getConstants().localeIdentifier,
  });

  // 4. Register token-refresh listener (idempotent)
  onTokenRefresh(messaging, async (fresh) => {
    await apiClient.post(endpoints.notifications.devices.register(), {
      token: fresh, /* ...device info same as above */
    });
  });
}
```

Failures are logged via `logError` and swallowed. Push registration must never block boot.

### 11.2 Foreground handler

Wired inside the same bootstrap step:

```ts
onMessage(messaging, async (remoteMessage) => {
  const data = remoteMessage.data ?? {};
  const alreadySeen = seenNotifications.has(data.notifId);
  if (alreadySeen) return;
  seenNotifications.add(data.notifId);

  // Bump the notifications query so the centre re-fetches
  queryClient.invalidateQueries({ queryKey: queryKeys.notifications.all });

  // Show a Notifee heads-up on the correct channel
  await notifee.displayNotification({
    id: data.notifId,
    title: data.title,
    body: data.body,
    android: { channelId: data.channelId ?? 'system', pressAction: { id: 'default' } },
    ios:     { sound: data.priority === 'high' ? 'default' : undefined },
    data,   // preserved for onForegroundEvent PRESS
  });
});
```

The centre uses `queryKeys.notifications.list()` — already declared in `src/constants/queryKeys.ts` — so a single invalidation refreshes any open list.

### 11.3 Background handler

Already registered in `src/app/bootstrap/steps/firebase.ts` as `setBackgroundMessageHandler`. Extend it to invalidate the persisted query cache so opening the app after a background push shows fresh data:

```ts
setBackgroundMessageHandler(messaging, async (remoteMessage) => {
  // Notifee will display on receipt via the OS if data.notification is present,
  // but since we send data-only, we display here:
  const data = remoteMessage.data ?? {};
  await notifee.displayNotification({
    id: data.notifId,
    title: data.title,
    body: data.body,
    android: { channelId: data.channelId ?? 'system', pressAction: { id: 'default' } },
    data,
  });
  // Optional: write into a "pending centre refresh" MMKV flag so onForeground
  // knows to invalidate immediately without waiting for the poll.
});
```

### 11.4 Tap handling

Already wired end-to-end via `services/notifications/deeplink.ts` → `handleFcmClick` → `resolveFcmClick` → `DeepLinkTarget` Zod parse → `handleResolved` → drain. Three call sites to add (each is a one-liner that hands `data` to `onFcmNotificationTapped`):

1. `onNotificationOpenedApp` (background → foreground via tap) — inside the bootstrap step.
2. `getInitialNotification` (cold-start via tap) — awaited during bootstrap; the result stashes a deep link that drains after nav is ready. The `drainPendingDeepLink` listener at `src/store/listeners/deeplinkDrainListener.ts` already handles this.
3. Notifee's `onForegroundEvent` with `type === EventType.PRESS` — inside the bootstrap step.

### 11.5 Notification Centre — no longer a stub

The screen at `src/features/shared/notifications/screens/NotificationCentreScreen.tsx` swaps its `ComingSoon` for a real list. Data comes from the endpoints already declared in `endpoints.notifications.*`, wrapped in a `useNotifications()` hook using TanStack Query. Item tap uses the same deep-link resolver as FCM taps — `handleResolved({ ok: true, target: notification.click_target })`. Bell/badge count comes from an `unreadCount` on the list response.

### 11.6 Preferences UI

A new screen under `features/shared/settings/NotificationPreferencesScreen.tsx`. Categories rendered from a static list matched to the server's `Category` union; the "transactional / high" rows render as disabled with a "always on" label — mirroring the server rule.

### 11.7 Permission flow — already correct

`ensureCapability('notifications')` in the capabilities registry (`src/rbac/capabilities.ts`) already covers Android 13 `POST_NOTIFICATIONS` and iOS provisional prompts through `PermissionService`. No new work here — the bootstrap step above uses it as-is.

---

## 12. Compile-time contract between app and server

The kinds and payload shapes are defined once, in server-side Zod, and mirrored to the app via a generated `.d.ts` (or copy-paste with a CI check that diffs them — the pragmatic v1 choice given no shared-package tooling exists yet).

`urbancruisebackend/src/modules/notifications/dispatch/kinds.ts`:

```ts
export const NotificationKind = z.enum([
  'enquiry.submitted',
  'enquiry.assignedInternally',
  'quotation.ready',
  'quotation.revised',
  'quotation.confirmed',
  'booking.created',
  'booking.assignmentApproved',
  'booking.vehicleChangeApproved',
  'booking.cancelled',
  'trip.driverEnRoute',
  'trip.driverArrived',
  'trip.otpVerified',
  'trip.completed',
  'trip.customerContactUnlocked',
  'payment.due',
  'payment.received',
  'payment.invoiceReady',
  'vendor.assignmentOffered',
  'vendor.assignmentReminder',
  'vendor.assignmentCancelled',
  'vendor.vehicleChangeDecision',
  'vendor.payoutInitiated',
  'driver.tripAssigned',
  'driver.tripStartingSoon',
  'driver.collectPayment',
  'uc.enquirySubmitted',
  'uc.quotationConfirmed',
  'uc.assignmentSlaBreach',
  'uc.vehicleChangeRequested',
  'uc.paymentReconciliationException',
  'system.announcement',
  'support.ticketUpdate',
  'feedback.reminder',
]);

export type NotificationKind = z.infer<typeof NotificationKind>;

// Payload schema per kind — matches what recipients() and renderer() expect
export const NotificationPayloads = {
  'booking.assignmentApproved': z.object({
    bookingId: z.string(),
    vehicleReg: z.string().optional(),
    driverName: z.string().optional(),
    tripStartAt: z.string(), // ISO
  }),
  // ...one per kind
} as const;
```

The app receives `kind` and treats it as an opaque routing string except for two uses: (a) mapping to Notifee `channelId` (Android — already carried on `data.channelId`), (b) analytics. The typed navigation target lives in `data.click`, which parses through the existing `DeepLinkTarget` union — that's the app's real type check.

---

## 13. Security

- **Token binding.** `POST /notifications/devices` requires an authenticated request. The row's `user_id` is taken from `req.identity`, never the request body. Attempting to register a token seen under another user disables the previous row and starts fresh — a shared device between two users must not fan out to both.
- **PII in `body`.** Rendered strings avoid full customer name, exact address, or phone. `"Vehicle KA-05-1234, driver Rajesh."` is fine; `"Rajesh will collect Anjali at 12/A MG Road"` is not — lock-screen previews are visible to bystanders. The rendered `body` is written to the DB exactly as sent; no separate re-render for display.
- **Deep-link parameters.** Every `click_target` goes through `DeepLinkTarget.safeParse` on the app side before navigation. The security invariants documented in `src/services/deeplinks/resolve.ts` (scheme allow-list, host allow-list, exact segment matching) also apply because the app takes the same path for FCM as for URLs.
- **Cross-role targeting.** `dispatch/recipients.ts` MUST NOT put a `role='driver'` identity in the recipient set for a customer-kind. Enforced by a runtime assertion at the end of each recipient function: `assert(all(r => r.role === expectedRole(kind)))`. Belt-and-braces against future editing mistakes.
- **Rate limit per user.** In the `notify()` path, at most one push per user per kind per 60 seconds (checked via the idempotency table's `first_seen_at`). Prevents a runaway loop in a business service from carpet-bombing a device.
- **FCM service account.** Stored in an env var per section 6; loaded once into `google-auth-library`. The scope is `https://www.googleapis.com/auth/firebase.messaging` only.
- **Logging.** `redactPaths.ts` (see `src/shared/logger/redactPaths.ts`) is extended to redact `token`, `data.click` (may contain IDs), and any `Authorization` echoed in FCM error responses.

---

## 14. Observability

**Structured logs.** Every stage logs a single `pino` line with the shared `requestId` (populated by `requestContext` middleware — the store follows the notification into the worker if the enqueue happens inline; the worker-side line has its own `workerId` in addition):

```
{ scope: 'notify.enqueue', kind, correlation, recipients: n, dropped: m, requestId }
{ scope: 'notify.worker.claim', batchSize: n, workerId }
{ scope: 'notify.worker.dispatch', notifId, deviceId, channel: 'push', durationMs, outcome }
```

**Metrics (Prometheus-style, or via pino counters until Prom lands).**

- `notif_enqueue_total{kind}`
- `notif_dispatch_total{kind, channel, outcome}`
- `notif_dispatch_duration_ms_bucket{channel}`
- `notif_outbox_backlog{status}` (gauge, sampled every 30s)
- `notif_device_disabled_total{reason}`

**Alerts.**

- `notif_outbox_backlog{status='pending'} > 1000` for 5m → paging alert (worker down or FCM outage).
- `notif_dispatch_total{outcome='failed'}` rate > 5/min → warn (bad payload deploy).
- `notif_device_disabled_total{reason='SENDER_ID_MISMATCH'}` > 0 → paging (config regression).

**Per-notification tracing.** Given a user complaint "I didn't get my arrival notification for booking BKG20260905A", the operator query is one SELECT:

```sql
SELECT n.*, o.status, o.attempt_count, o.last_error_code, o.sent_at
FROM notifications n
LEFT JOIN notification_outbox o ON o.notification_id = n.id
WHERE JSON_EXTRACT(n.click_target, '$.bookingId') = 'BKG20260905A'
  AND n.kind = 'trip.driverArrived'
ORDER BY n.created_at DESC;
```

---

## 15. Retention

- `notifications`: 180 days, then hard-delete via nightly job. In-app centre lists the last 90 days by default; older rows are queryable by `id` but hidden from the list.
- `notification_outbox`: 30 days for `sent`, 90 days for `failed` / `dropped`. Debugging window matters more than storage saving.
- `notification_devices`: `disabled_at` rows purged after 30 days.
- `notification_idempotency`: 24 hours (a sliding window; the 30-second dedupe rule doesn't need long history).

---

## 16. Testing strategy

**Unit — dispatch/renderer/kinds.** Every kind gets: a payload validation test (accept + reject fixtures), a rendering test (title/body/click snapshot), a recipient test (given synthesized DB fixtures, does the fan-out produce the expected identities).

**Unit — worker retry logic.** Given an outbox row and a mock FCM adapter returning each documented error, assert the resulting `(status, attempt_count, next_attempt_at, device.disabled_at)`.

**Integration — enqueue in a transaction.** In-process MySQL (test container). Call `notify()` inside a transaction and roll back; assert no rows persist. Call and commit; assert `notifications` + `notification_outbox` are consistent.

**Integration — end-to-end with fake FCM.** A local HTTP server that mimics FCM HTTP v1 responses. Assert the worker's HTTP request shape matches section 8.

**Contract — kind sync.** A CI job that fails if the app's copy of `NotificationKind` diverges from the server's `dispatch/kinds.ts` file. Zero-cost alternative to a shared package.

**Manual test matrix.**

- Foreground receive: notifee heads-up on the right channel + centre refresh.
- Background receive: OS shade shows correctly.
- Cold-start tap: app opens on the deep-linked screen (not the tab home).
- Airplane mode → back online: catch-up delivery via TTL (only fresh ones); centre reflects backlog.
- Permission denied on Android 13: centre still populates; no push crash.
- Corporate customer with admin + bookingPerson: both devices ring for approved-assignment; only admin rings for invoice-ready.

---

## 17. Rollout plan

**Phase 0 — schema and endpoints.** Create the four tables. Ship `POST/DELETE /notifications/devices` + `GET /notifications` + `POST /notifications/:id/read` + `POST /notifications/read-all` + preferences endpoints. Ship the app changes for device registration, foreground/background handlers, tap wiring, and a real Centre screen. No `notify()` calls from business services yet — the pipeline is dark.

**Phase 1 — dispatch pipeline (dark launch).** Ship `notify()`, worker, FCM adapter. Behind a feature flag on a single kind — `system.announcement` — that UC can trigger by hand. Exercise every observability surface on real devices before opening the floodgates.

**Phase 2 — high-value transactional.** Wire the top-5 kinds first: `booking.assignmentApproved`, `trip.driverArrived`, `trip.otpVerified`, `trip.completed`, `payment.due`. These are the ones a customer will complain about not getting.

**Phase 3 — remaining kinds.** Roll out per module: vendor, then driver, then UC. Each module ships its kinds together so their triggers are reviewed as a unit.

**Phase 4 — preferences UI and reminder kinds.** Once transactional traffic is stable, expose preferences and enable `feedback.reminder` + `system.announcement` under user control.

---

## 18. Open questions / follow-ups

- **UC subscription model.** The current design uses a `uc_notification_subscriptions` table. If UC turns out to have many roles internally (accounts vs ops vs support), this may benefit from being merged with the emerging vendor `sub_role` pattern rather than being ops-team-scoped freeform.
- **Localisation.** `renderer.ts` currently returns English. `notification_devices.locale` is captured so translations can slot in with per-locale template files. Not blocking v1.
- **Silent trip-live sync.** Right now the customer's live-trip screen relies on TanStack Query polling. A silent data push (`data.kind === 'trip.locationTick'`) could reduce polling load on the backend. Deferred until measurements show polling is actually a problem.
- **Batch send (`sendAll` vs `send`).** FCM HTTP v1 has no true batch — the SDK's `sendAll` is client-side parallelism. Worth wrapping in the adapter later; v1 sends serially with concurrency limit 10 to keep the code obvious.

---

## Appendix A — Where each change lands (checklist for the person implementing this)

Backend (`urbancruisebackend`):
- `src/config/env.ts` — add `FCM_*`, `NOTIF_WORKER_*` variables and validation.
- `src/config/constants.ts` — add default channelIds, TTL constants.
- `src/shared/logger/redactPaths.ts` — add `token`, `data.click`.
- Migrations under `src/shared/db/migrations/` (add the folder if you don't have one yet) — the four tables.
- `src/modules/notifications/**` — all files listed in section 6.
- `src/app.ts` — mount `notificationsModule.mount()` at `/notifications`.
- `src/server.ts` OR `ecosystem.config.cjs` — start the worker (section 10.4).
- Wire `notify()` into `customer/*/service.ts`, `vendor/*/service.ts`, `driver/*/service.ts`, `uc/*/service.ts` as each module is implemented — one row from section 2 per call.

Frontend (`urbancruiseapp`):
- `src/api/endpoints.ts` — extend the `notifications` block per section 7.
- `src/app/bootstrap/steps/deviceRegistration.ts` — new file, per section 11.1.
- `src/app/bootstrap/steps/firebase.ts` — extend background handler per 11.3; register Notifee channels per 8.2.
- `src/app/bootstrap/index.ts` — call `registerDeviceForPush` inside the authenticated branch.
- `src/services/notifications/handlers.ts` — new file wiring `onMessage`, `onNotificationOpenedApp`, `getInitialNotification`, `notifee.onForegroundEvent`.
- `src/features/shared/notifications/screens/NotificationCentreScreen.tsx` — real implementation.
- `src/features/shared/settings/NotificationPreferencesScreen.tsx` — new screen, wire into MoreSheet.
- `src/services/notifications/kinds.ts` — mirrored copy of the server's kind list; CI check for drift.
- Update `src/services/notifications/deeplink.ts` header comment (currently references "§8.3 of the notifications design") to point at section 8 of this doc.
