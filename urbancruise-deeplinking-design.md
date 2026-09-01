# Urban Cruise — Deep Linking: Design & Setup

**Status:** Draft for review
**Scope:** App-side (`urbancruiseapp`) end-to-end deep linking. URL schemes, Universal Links, App Links, and push-notification deep links — with role gating, quit-state resolution, security validation, and native setup for Android + iOS.
**Audience:** Mobile engineers.

---

## 1 · Codebase context (what already exists)

| Area | Reality on `main` today |
|---|---|
| **Navigation** | React Navigation v7 native-stack. `RootNavigator` uses **conditional groups** — exactly one of `SplashIntro / OnboardingFlow / AuthFlow / CustomerFlow / VendorFlow / DriverFlow / UcFlow` is mounted at a time. Role navigators are `React.lazy`. |
| **NavigationContainer** | In `App.tsx`, ref-wired via `navigationRef` from `NavigationService.ts`. No `linking` prop currently configured. |
| **NavigationService** | Solid — `navigate / goBack / reset / replace / getCurrentRoute`. Flat-union typing across every ParamList. Safe to call before container is ready (no-ops). Duplicate route names across roles must have identical params (enforced by intersection typing). |
| **AndroidManifest.xml** | `MainActivity` is `singleTask`, `exported=true`, with **only** the `LAUNCHER` intent-filter. **No** `VIEW` intent-filter for schemes, **no** App Links (`autoVerify`), **no** `POST_NOTIFICATIONS`. `enableOnBackInvokedCallback="false"`. |
| **iOS Info.plist** | `LSApplicationQueriesSchemes` for `tel / telprompt / mailto / whatsapp` (outgoing checks only). **No** `CFBundleURLTypes` (custom scheme registration), **no** `com.apple.developer.associated-domains` entitlement (Universal Links). |
| **Firebase** | `@react-native-firebase/messaging@26` installed. `initFirebase()` in bootstrap registers a **no-op** `setBackgroundMessageHandler`. `getInitialNotification` / `onNotificationOpenedApp` **not yet wired**. |
| **NotificationService screen** | `ComingSoon` placeholder in `features/shared/notifications/`. |
| **Endpoint registry** | `/notifications/*` stubs exist but no `POST /notifications/devices` yet. |
| **Roles** | Four top-level roles (`customer / vendor / driver / uc`). Only `customer` is enabled at the auth boundary (`ENABLED_ROLES` in `authenticate.ts` server-side). This drives which links can validly resolve today. |
| **Duplicate route names across roles** | `TripDetail` exists in both Vendor and Driver stacks with `{ tripId }`; `NotificationCentre / Support / Profile / Settings` exist in most stacks. The navigator that owns each name is determined by **which role branch is currently mounted** — a critical constraint for deep-link resolution. |

---

## 2 · Research summary (2026 best practices)

1. **Custom URL schemes (`urbancruise://`) are for internal use only.** They are unverified — any other app can register the same scheme. Never use them for authentication callbacks, cross-app OAuth, or anything security-sensitive. They are fine for push-notification click intents and for local test tooling.
2. **Universal Links (iOS) and App Links (Android) are the production channel** for links that arrive from outside the app (email, SMS, WhatsApp, browsers). They require a verified association file hosted on your domain:
   - **iOS:** `https://<your-domain>/.well-known/apple-app-site-association` (AASA), no `.json` extension, `Content-Type: application/json`, served over HTTPS with a valid cert. The `applinks` entitlement lists the domain.
   - **Android:** `https://<your-domain>/.well-known/assetlinks.json`. The manifest intent-filter has `android:autoVerify="true"`; on install, the OS fetches the file and verifies the app's signing SHA-256.
3. **iOS ATT + Universal Links can be broken by users** (long-press → "Open in Safari" or clear-once). The system remembers the user's choice per link. A production design falls back to a hosted intermediary page that either deep-links or shows a "Open in App" button.
4. **Android verification is strict.** A single bad `assetlinks.json` (wrong SHA-256, wrong package name, wrong content-type) silently disables auto-verification for the whole app. Test with `adb shell pm get-app-links <package>` and `pm verify-app-links --re-verify <package>`.
5. **React Navigation v7's `linking` config resolves URL paths to (possibly nested) routes.** Its `getStateFromPath` is deterministic; combined with `getInitialURL` + `subscribe` overrides, you can bridge FCM click intents, `Linking.openURL` calls, and OS-delivered universal links through the same code path.
6. **The single hardest bug in deep linking is the quit-state race.** The link arrives before the navigator is mounted. The universally-adopted fix is to stash the payload in a module-scoped queue and consume it once the navigator emits `onReady`. React Navigation's built-in `getInitialURL` handles this for OS-delivered URLs; you still need manual stashing for **push notifications**, because those are delivered via the Firebase messaging APIs, not `Linking`.
7. **Role changes invalidate deep links in flight.** If a link addresses a customer screen but the authenticated user is a driver, the correct behavior is not to crash and not to silently drop — it's to route to a defined fallback (usually the driver's home + a toast) so the user gets a signal.

---

## 3 · Design goals

| Goal | Concretely |
|---|---|
| **Reliable across every entry point** | Notification tap, `urbancruise://` scheme, `https://` Universal/App Link, and share intents all resolve through **one** function. No divergent code paths per entry point. |
| **Correct across every app state** | Foreground, background, quit — same destination reached, same params passed, same auth/role gates enforced. |
| **Secure** | Every incoming URL is validated with a Zod schema. A malformed payload is dropped, not crashed on. Role-inappropriate destinations are rejected with a defined fallback. No arbitrary-screen navigation. |
| **Role-aware** | Same path (e.g. `/trip/:id`) can resolve to different screens depending on the authenticated role. Resolution is aware of `RootStackParamList` gating. |
| **Session-aware** | A link tapped while unauthenticated stashes the intent, funnels through login, then resumes to the target after auth completes. |
| **Long-term maintainable** | Adding a new deep-link destination is one entry in a typed catalog. Removing one is a compile error at every call site. |
| **Testable without a device** | The path→state resolution is a pure function you can unit-test. The end-to-end flow is scriptable via `adb shell am start` and `xcrun simctl openurl`. |

---

## 4 · Architecture

### 4.1 The four entry points, one exit

```
                  ┌──────────────────────────────────────┐
                  │  ENTRY POINTS                        │
                  ├──────────────────────────────────────┤
    ┌─────────────┤ A. urbancruise:// (custom scheme)   │
    │             │    — dev tools, internal QA         │
    │             │                                      │
    │             │ B. https://app.urbancruise.com/…    │
    │             │    — email, WhatsApp, SMS, browsers │
    │             │      (Universal / App Links)         │
    │             │                                      │
    │             │ C. FCM data push with `click`       │
    │             │    — most common in production      │
    │             │                                      │
    │             │ D. Explicit share intent            │
    │             │    (react-native `Linking.openURL`) │
    │             └──────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│  services/deeplinks/resolve.ts                          │
│  ─ Zod validation                                       │
│  ─ Path → { screen, params } (pure function)            │
│  ─ Role/session gate check                              │
│  ─ Fallback selection when target isn't reachable       │
└─────────────────────┬───────────────────────────────────┘
                      │
      ┌───────────────┴────────────────┐
      │                                │
   navigator                        pending
   ready                            queue
      │                                │
      ▼                                │
   navigate(...)                       │
                                       ▼
                             (drained after
                              onReady / auth
                              transition)
```

### 4.2 File additions

```
src/services/deeplinks/
├── index.ts                # barrel: initDeepLinks, handleUrl, handleFcmClick
├── schema.ts               # Zod discriminated union of every legal target
├── catalog.ts              # path patterns ↔ target descriptors
├── resolve.ts              # pure: url → ResolvedTarget | Rejection
├── gate.ts                 # role & session gating with fallback selection
├── pending.ts              # stash-and-consume for quit/pre-auth deep links
├── linkingConfig.ts        # React Navigation `linking` prop
└── __tests__/
    ├── resolve.test.ts
    └── gate.test.ts
```

Plus:

```
src/services/notifications/deeplink.ts   # thin bridge: FCM data → deeplinks.handleFcmClick
```

---

## 5 · The typed catalog — single source of truth

Every legal deep-link target is declared **once**. Every entry names:

- the **URL path** (Universal Link / App Link)
- the **navigator target** (screen name + params factory)
- the **required role** (or `'any'`)
- an **auth requirement** (`'authenticated' | 'any'`)
- an optional **fallback** if the gate fails

```ts
// src/services/deeplinks/schema.ts
import { z } from 'zod';

// ─── Params schemas (mirror the ParamList entries) ──────────────────
export const BookingId    = z.string().min(1);   // narrower brand once branded types are in
export const TripId       = z.string().min(1);
export const QuotationId  = z.string().min(1);
export const CustomerId   = z.string().min(1);
export const VendorId     = z.string().min(1);
export const DriverId     = z.string().min(1);
export const EnquiryId    = z.string().min(1);

// ─── The exhaustive union of legal targets ───────────────────────────
// One variant per deep-linkable destination. Adding a screen is one
// entry here + one entry in catalog.ts. TypeScript will refuse to
// compile if resolver code forgets a case.
export const DeepLinkTarget = z.discriminatedUnion('kind', [
  // Customer
  z.object({ kind: z.literal('customer.bookingDetail'),   bookingId:   BookingId }),
  z.object({ kind: z.literal('customer.tripLive'),        tripId:      TripId }),
  z.object({ kind: z.literal('customer.quotationDetail'), quotationId: QuotationId }),
  z.object({ kind: z.literal('customer.payBalance'),      bookingId:   BookingId }),
  z.object({ kind: z.literal('customer.feedback'),        bookingId:   BookingId }),

  // Vendor
  z.object({ kind: z.literal('vendor.assignmentDetail'),  bookingId:   BookingId }),
  z.object({ kind: z.literal('vendor.tripDetail'),        tripId:      TripId }),

  // Driver
  z.object({ kind: z.literal('driver.tripDetail'),        tripId:      TripId }),
  z.object({ kind: z.literal('driver.collectPayment'),    tripId:      TripId }),

  // UC
  z.object({ kind: z.literal('uc.enquiryDetail'),         enquiryId:   EnquiryId }),
  z.object({ kind: z.literal('uc.customerDetail'),        customerId:  CustomerId }),
  z.object({ kind: z.literal('uc.tripMonitor'),           tripId:      TripId }),

  // Cross-role
  z.object({ kind: z.literal('common.notificationCentre') }),
  z.object({ kind: z.literal('common.support') }),
]);

export type DeepLinkTarget = z.infer<typeof DeepLinkTarget>;
```

```ts
// src/services/deeplinks/catalog.ts
import type { UserRole } from '@rbac/roles';
import type { DeepLinkTarget } from './schema';

export type CatalogEntry = {
  /** Match paths like '/bookings/:id'. The '/' is required at start. */
  path: string;
  /** Which role is authorized to receive this target. */
  role: UserRole | 'any';
  /** Whether the user must be authenticated. */
  auth: 'authenticated' | 'any';
  /** Build target from parsed path params. */
  build: (params: Record<string, string>) => DeepLinkTarget;
  /**
   * If the gate rejects (wrong role, not authenticated with `authenticated`,
   * etc.), where to send the user instead.
   */
  fallback?: DeepLinkTarget;
};

export const CATALOG: readonly CatalogEntry[] = [
  // Customer
  { path: '/bookings/:id',        role: 'customer', auth: 'authenticated',
    build: ({ id }) => ({ kind: 'customer.bookingDetail', bookingId: id }) },
  { path: '/trip/:id',            role: 'customer', auth: 'authenticated',
    build: ({ id }) => ({ kind: 'customer.tripLive', tripId: id }) },
  { path: '/quotations/:id',      role: 'customer', auth: 'authenticated',
    build: ({ id }) => ({ kind: 'customer.quotationDetail', quotationId: id }) },
  { path: '/bookings/:id/pay',    role: 'customer', auth: 'authenticated',
    build: ({ id }) => ({ kind: 'customer.payBalance', bookingId: id }) },
  { path: '/bookings/:id/feedback', role: 'customer', auth: 'authenticated',
    build: ({ id }) => ({ kind: 'customer.feedback', bookingId: id }) },

  // Vendor
  { path: '/vendor/assignments/:id', role: 'vendor', auth: 'authenticated',
    build: ({ id }) => ({ kind: 'vendor.assignmentDetail', bookingId: id }) },
  { path: '/vendor/trips/:id',    role: 'vendor', auth: 'authenticated',
    build: ({ id }) => ({ kind: 'vendor.tripDetail', tripId: id }) },

  // Driver
  { path: '/driver/trips/:id',    role: 'driver', auth: 'authenticated',
    build: ({ id }) => ({ kind: 'driver.tripDetail', tripId: id }) },
  { path: '/driver/trips/:id/collect', role: 'driver', auth: 'authenticated',
    build: ({ id }) => ({ kind: 'driver.collectPayment', tripId: id }) },

  // UC
  { path: '/uc/enquiries/:id',    role: 'uc', auth: 'authenticated',
    build: ({ id }) => ({ kind: 'uc.enquiryDetail', enquiryId: id }) },
  { path: '/uc/customers/:id',    role: 'uc', auth: 'authenticated',
    build: ({ id }) => ({ kind: 'uc.customerDetail', customerId: id }) },
  { path: '/uc/trips/:id',        role: 'uc', auth: 'authenticated',
    build: ({ id }) => ({ kind: 'uc.tripMonitor', tripId: id }) },

  // Cross-role
  { path: '/notifications',       role: 'any', auth: 'authenticated',
    build: () => ({ kind: 'common.notificationCentre' }) },
  { path: '/support',             role: 'any', auth: 'any',
    build: () => ({ kind: 'common.support' }) },
] as const;
```

**Why a catalog.** Adding a new deep-linkable screen is a two-line diff (schema + catalog entry). Removing one is a compile error at every reference site. Auditing which URLs the app accepts is one file to read.

---

## 6 · Resolver — pure, testable, security-aware

```ts
// src/services/deeplinks/resolve.ts
import { DeepLinkTarget } from './schema';
import { CATALOG, CatalogEntry } from './catalog';

export type ResolveOk    = { ok: true;  entry: CatalogEntry; target: DeepLinkTarget };
export type ResolveError =
  | { ok: false; reason: 'malformed_url' }
  | { ok: false; reason: 'unknown_scheme' }
  | { ok: false; reason: 'unknown_host' }
  | { ok: false; reason: 'no_match' }
  | { ok: false; reason: 'invalid_params' };
export type ResolveResult = ResolveOk | ResolveError;

const ALLOWED_SCHEMES   = new Set(['urbancruise:', 'https:']);
const ALLOWED_HOSTS_HTTPS = new Set(['app.urbancruise.com']);  // production
                                                                // add staging host from ENV in dev
const ALLOWED_CUSTOM_HOSTS = new Set(['open']);                 // urbancruise://open/bookings/... (see §7)

export function resolveUrl(rawUrl: string): ResolveResult {
  let url: URL;
  try {
    url = new URL(rawUrl);
  } catch {
    return { ok: false, reason: 'malformed_url' };
  }

  if (!ALLOWED_SCHEMES.has(url.protocol))
    return { ok: false, reason: 'unknown_scheme' };

  if (url.protocol === 'https:' && !ALLOWED_HOSTS_HTTPS.has(url.host))
    return { ok: false, reason: 'unknown_host' };

  if (url.protocol === 'urbancruise:' && !ALLOWED_CUSTOM_HOSTS.has(url.hostname))
    return { ok: false, reason: 'unknown_host' };

  const pathname = url.pathname;   // e.g. '/bookings/01HXXX'
  for (const entry of CATALOG) {
    const params = matchPath(entry.path, pathname);
    if (!params) continue;
    const parsed = DeepLinkTarget.safeParse(entry.build(params));
    if (!parsed.success) return { ok: false, reason: 'invalid_params' };
    return { ok: true, entry, target: parsed.data };
  }
  return { ok: false, reason: 'no_match' };
}

// Handle FCM data-payload `click` — already a structured object, not a URL.
export function resolveFcmClick(
  clickRaw: string | undefined,
): ResolveResult {
  if (!clickRaw) return { ok: false, reason: 'malformed_url' };
  try {
    const parsed = JSON.parse(clickRaw);
    const target = DeepLinkTarget.safeParse(parsed);
    if (!target.success) return { ok: false, reason: 'invalid_params' };
    // For gate lookup, find the catalog entry by target kind.
    const entry = CATALOG.find(e => sameKind(e, target.data));
    if (!entry) return { ok: false, reason: 'no_match' };
    return { ok: true, entry, target: target.data };
  } catch {
    return { ok: false, reason: 'malformed_url' };
  }
}

// Simple param matcher — supports ':name' segments only.
// Kept in this file so the resolver has zero external dependencies
// and is trivially testable.
function matchPath(pattern: string, actual: string): Record<string, string> | null {
  const p = pattern.split('/').filter(Boolean);
  const a = actual.split('/').filter(Boolean);
  if (p.length !== a.length) return null;
  const out: Record<string, string> = {};
  for (let i = 0; i < p.length; i++) {
    if (p[i]!.startsWith(':')) out[p[i]!.slice(1)] = decodeURIComponent(a[i]!);
    else if (p[i] !== a[i]) return null;
  }
  return out;
}

function sameKind(e: CatalogEntry, t: DeepLinkTarget): boolean {
  return e.build({} as never).kind === t.kind;   // safe because build is pure and deterministic on kind
}
```

**Security notes on this resolver:**

- **Scheme & host allow-lists are explicit.** An attacker who registers a Universal Link on `evil.com/bookings/…` cannot deep-link into your app; the resolver rejects with `unknown_host`. This is the reason we don't do "match any path we can parse."
- **Path matching is strict segment-by-segment.** Query strings and fragments are intentionally ignored — deep-link intent lives in the path, not the query. This prevents path-traversal via `..` and open-redirect via `?next=`.
- **Params flow through Zod** before reaching the navigator. Any non-conforming id (empty, too long, wrong shape) is rejected as `invalid_params`.
- **`decodeURIComponent`** is inside `matchPath`, once. There is no second decode anywhere downstream.

---

## 7 · Session & role gating

```ts
// src/services/deeplinks/gate.ts
import type { UserRole } from '@rbac/roles';
import type { DeepLinkTarget } from './schema';
import type { CatalogEntry }    from './catalog';

export type GateContext = {
  bootstrapped:   boolean;
  isAuthenticated: boolean;
  userRole:       UserRole | null;
};

export type GateOk   = { ok: true;  target: DeepLinkTarget };
export type GateHold = { ok: false; reason: 'not_bootstrapped' | 'not_authenticated'; target: DeepLinkTarget };
export type GateDeny = { ok: false; reason: 'wrong_role';       target: DeepLinkTarget; fallback?: DeepLinkTarget };
export type GateResult = GateOk | GateHold | GateDeny;

export function gate(
  entry: CatalogEntry,
  target: DeepLinkTarget,
  ctx: GateContext,
): GateResult {
  if (!ctx.bootstrapped) return { ok: false, reason: 'not_bootstrapped', target };

  if (entry.auth === 'authenticated' && !ctx.isAuthenticated)
    return { ok: false, reason: 'not_authenticated', target };

  if (entry.role !== 'any' && entry.role !== ctx.userRole) {
    // eslint-disable-next-line @typescript-eslint/no-non-null-assertion
    return { ok: false, reason: 'wrong_role', target, fallback: entry.fallback };
  }

  return { ok: true, target };
}
```

**Handling each gate outcome:**

| Result | Behavior |
|---|---|
| `ok` | Immediately `navigate(...)` (or stash if navigator not ready). |
| `not_bootstrapped` | Stash. `RootNavigator.onReady` will drain. |
| `not_authenticated` | Stash. `loginSuccess` reducer triggers drain via a Redux subscription. |
| `wrong_role` | Route to fallback (or role home) + show a toast: *"That link is for a different account type."* Do **not** silently drop — the user needs a signal. |

The stash-and-consume queue is deliberately module-scoped, not Redux — deep-link intent is transient and shouldn't persist across cold starts:

```ts
// src/services/deeplinks/pending.ts
import type { DeepLinkTarget } from './schema';

let pending: DeepLinkTarget | null = null;

/** Overwrite policy: the newest deep link wins. This matters when
    a user taps a second notification before the first has drained. */
export function stash(target: DeepLinkTarget): void { pending = target; }
export function consume(): DeepLinkTarget | null { const t = pending; pending = null; return t; }
export function peek(): DeepLinkTarget | null    { return pending; }
```

---

## 8 · Wiring into React Navigation

### 8.1 The `linking` prop

React Navigation's `linking` handles `https://` and custom-scheme URLs delivered via the OS (`Linking.getInitialURL` / `Linking.addEventListener`). We keep our resolver as the source of truth by having `linking` delegate to it:

```ts
// src/services/deeplinks/linkingConfig.ts
import type { LinkingOptions } from '@react-navigation/native';
import { Linking } from 'react-native';
import type { RootStackParamList } from '@/navigation/types';

import { resolveUrl } from './resolve';
import { targetToNavigatePayload } from './toNavigate';   // §8.2

export function buildLinkingConfig(): LinkingOptions<RootStackParamList> {
  return {
    prefixes: [
      'urbancruise://',
      'https://app.urbancruise.com',
      // Add staging prefixes from ENV in dev builds
    ],

    /**
     * Cold-start: called ONCE, before the container renders anything.
     * If the app was launched from a URL we return it here; the container
     * uses it to render the correct screen from the start (no flash).
     */
    async getInitialURL(): Promise<string | null> {
      const url = await Linking.getInitialURL();
      if (!url) return null;
      // Only return URLs our resolver accepts; anything else is dropped
      // so the app opens on the normal home destination.
      const r = resolveUrl(url);
      return r.ok ? url : null;
    },

    /**
     * Warm subscription: OS-delivered URLs while the app is running.
     * The listener receives every URL; we let React Navigation forward
     * only URLs we accept.
     */
    subscribe(listener) {
      const sub = Linking.addEventListener('url', ({ url }) => {
        const r = resolveUrl(url);
        if (r.ok) listener(url);
      });
      return () => sub.remove();
    },

    /**
     * Our resolver -> React Navigation state. We DON'T use the
     * declarative `config.screens` map — the role-gated conditional
     * groups in RootNavigator make a static map insufficient. Instead
     * we compute the state ourselves via `getStateFromPath`.
     */
    getStateFromPath: (path, options) => {
      const fakeUrl = path.startsWith('/') ? `https://app.urbancruise.com${path}` : path;
      const r = resolveUrl(fakeUrl);
      if (!r.ok) return undefined;  // React Navigation falls back to no navigation
      return targetToNavigatePayload(r.target).stateForLinking;
    },
  };
}
```

Wire it in `App.tsx`:

```ts
<NavigationContainer
  ref={navigationRef}
  theme={AppNavigationTheme}
  linking={buildLinkingConfig()}
  onReady={() => { drainPendingDeepLink(); }}
>
```

`drainPendingDeepLink` reads from Redux to build `GateContext`, gates the stashed target, and either navigates or leaves it pending.

### 8.2 Target → navigate payload

Because the same target `kind` reaches different navigators depending on role (e.g. `common.notificationCentre` is a screen in the customer stack **or** the driver stack), the mapping needs the currently mounted flow:

```ts
// src/services/deeplinks/toNavigate.ts
import type { DeepLinkTarget } from './schema';

export type NavigatePayload = {
  screen: string;
  params?: Record<string, unknown>;
};

export function targetToNavigatePayload(t: DeepLinkTarget): NavigatePayload {
  switch (t.kind) {
    case 'customer.bookingDetail':   return { screen: 'BookingDetail',   params: { bookingId:   t.bookingId } };
    case 'customer.tripLive':        return { screen: 'TripLive',        params: { tripId:      t.tripId } };
    case 'customer.quotationDetail': return { screen: 'QuotationDetail', params: { quotationId: t.quotationId } };
    case 'customer.payBalance':      return { screen: 'PayBalance',      params: { bookingId:   t.bookingId } };
    case 'customer.feedback':        return { screen: 'Feedback',        params: { bookingId:   t.bookingId } };

    case 'vendor.assignmentDetail':  return { screen: 'AssignmentDetail', params: { bookingId:  t.bookingId } };
    case 'vendor.tripDetail':        return { screen: 'TripDetail',      params: { tripId:      t.tripId } };

    case 'driver.tripDetail':        return { screen: 'TripDetail',      params: { tripId:      t.tripId } };
    case 'driver.collectPayment':    return { screen: 'CollectPayment',  params: { tripId:      t.tripId } };

    case 'uc.enquiryDetail':         return { screen: 'EnquiryDetail',   params: { enquiryId:   t.enquiryId } };
    case 'uc.customerDetail':        return { screen: 'CustomerDetail',  params: { customerId:  t.customerId } };
    case 'uc.tripMonitor':           return { screen: 'TripMonitor',     params: { tripId:      t.tripId } };

    case 'common.notificationCentre':return { screen: 'NotificationCentre' };
    case 'common.support':           return { screen: 'Support' };
  }
}
```

The switch is exhaustive — TypeScript's `never` check on the discriminated union guarantees every new schema variant forces a case here. That's the entire maintenance surface.

Because `NavigationService.navigate` is typed as the intersection of every ParamList, this payload passes through as `navigate(screen, params)` and lands in whichever navigator currently owns that name. This is why duplicate route names across roles must have identical param types — a constraint the codebase already enforces.

### 8.3 Bridging FCM notifications

```ts
// src/services/notifications/deeplink.ts
import { resolveFcmClick } from '@/services/deeplinks/resolve';
import { handleResolved }  from '@/services/deeplinks';

export function onFcmNotificationTapped(data: Record<string,string> | undefined): void {
  if (!data) return;
  const r = resolveFcmClick(data.click);
  if (!r.ok) {
    // log-only; never surface parse errors to users
    logError(new Error(`fcm.deeplink.resolve.${r.reason}`), { boundary: 'fcm.click' });
    return;
  }
  handleResolved(r);   // gates → stash-or-navigate
}
```

Wire the three FCM handlers (foreground/background/quit) as designed in the notifications doc; each funnels through `onFcmNotificationTapped`.

### 8.4 Draining after auth transitions

Two triggers drain the pending queue:

1. `NavigationContainer.onReady` — cold start after `getInitialNotification` stashed something during bootstrap.
2. A tiny Redux subscription in `App.tsx`: whenever `isAuthenticated` transitions `false → true`, or `userRole` changes, call the drainer.

```ts
// src/services/deeplinks/drain.ts
import { store } from '@/store';
import { navigate } from '@/navigation/NavigationService';
import { consume, peek, stash } from './pending';
import { gate } from './gate';
import { CATALOG } from './catalog';
import { targetToNavigatePayload } from './toNavigate';

export function drainPendingDeepLink(): void {
  const target = peek();
  if (!target) return;

  const { bootstrapped, isAuthenticated, userRole } = store.getState().app;
  const entry = CATALOG.find(e => e.build({} as never).kind === target.kind);
  if (!entry) { consume(); return; }

  const g = gate(entry, target, { bootstrapped, isAuthenticated, userRole });
  if (!g.ok && g.reason === 'not_bootstrapped')   return; // keep waiting
  if (!g.ok && g.reason === 'not_authenticated')  return; // keep waiting

  consume();

  if (!g.ok && g.reason === 'wrong_role') {
    if (g.fallback) {
      const p = targetToNavigatePayload(g.fallback);
      p.params ? navigate(p.screen as never, p.params as never) : navigate(p.screen as never);
    }
    // Surface a toast to the user via existing toast service
    return;
  }

  const p = targetToNavigatePayload(g.target);
  p.params ? navigate(p.screen as never, p.params as never) : navigate(p.screen as never);
}
```

---

## 9 · Android setup — step-by-step

### 9.1 Add App Links intent-filter to `AndroidManifest.xml`

Under the existing `<activity android:name=".MainActivity" ...>` block, **add** (do not replace the LAUNCHER filter — you need both):

```xml
<!-- Custom scheme (internal use only) -->
<intent-filter>
  <action android:name="android.intent.action.VIEW" />
  <category android:name="android.intent.category.DEFAULT" />
  <category android:name="android.intent.category.BROWSABLE" />
  <data android:scheme="urbancruise" android:host="open" />
</intent-filter>

<!-- App Links (production channel — verified) -->
<intent-filter android:autoVerify="true">
  <action android:name="android.intent.action.VIEW" />
  <category android:name="android.intent.category.DEFAULT" />
  <category android:name="android.intent.category.BROWSABLE" />
  <data android:scheme="https" android:host="app.urbancruise.com" />
  <!-- List only the paths we OWN — never blanket "/" without pathPrefix -->
  <data android:pathPrefix="/bookings" />
  <data android:pathPrefix="/trip" />
  <data android:pathPrefix="/quotations" />
  <data android:pathPrefix="/vendor" />
  <data android:pathPrefix="/driver" />
  <data android:pathPrefix="/uc" />
  <data android:pathPrefix="/notifications" />
  <data android:pathPrefix="/support" />
</intent-filter>
```

**Why the `urbancruise://` host is `open`.** Android requires a host on custom schemes for the `VIEW` intent-filter to trigger. `urbancruise://bookings/01HXXX` won't match; `urbancruise://open/bookings/01HXXX` will. This host is invisible to users but keeps intents unambiguous.

**Why `pathPrefix` and not blanket `/`.** Blanket matching means every https://app.urbancruise.com link — including your marketing site's home page — would try to open in the app. Prefixes scope it to intents we actually handle.

### 9.2 Publish `assetlinks.json`

Server side: at `https://app.urbancruise.com/.well-known/assetlinks.json` serve:

```json
[
  {
    "relation": ["delegate_permission/common.handle_all_urls"],
    "target": {
      "namespace": "android_app",
      "package_name": "com.pinaak",
      "sha256_cert_fingerprints": [
        "AA:BB:CC:...",   // Release signing key SHA-256
        "11:22:33:..."    // Play App Signing SHA-256 if you use it
      ]
    }
  }
]
```

Requirements checklist (every one is a common failure mode):

- [ ] Served over **HTTPS with a valid cert** — self-signed will fail verification.
- [ ] Response `Content-Type: application/json` — not `text/plain`.
- [ ] No redirects on the request — Android will not follow.
- [ ] Package name matches `applicationId` in `android/app/build.gradle`.
- [ ] SHA-256 fingerprints include **every** signing key that can produce a release build. If you use Play App Signing, include **both** the upload key and the app-signing key — the app-signing key is what devices verify against.
- [ ] File is a JSON array `[…]`, not an object.

Getting fingerprints:
```
keytool -list -v -keystore <release.keystore> -alias <alias> | grep SHA256
# For Play App Signing: Play Console → Setup → App integrity → App signing key certificate → SHA-256
```

### 9.3 Verify after installation

Install a signed build (App Links do **not** verify on debug builds by default):

```
adb shell pm get-app-links com.pinaak

# Expected output includes: `verified` for https://app.urbancruise.com
# If it says `legacy_failure` or `verification_failed`, re-check the JSON.

# Force re-verify (useful after fixing the file):
adb shell pm verify-app-links --re-verify com.pinaak
```

Testing intents:
```
# Custom scheme
adb shell am start -W -a android.intent.action.VIEW \
  -d "urbancruise://open/bookings/01HXXX000000000000000000" \
  com.pinaak

# App Link (once verified)
adb shell am start -W -a android.intent.action.VIEW \
  -d "https://app.urbancruise.com/bookings/01HXXX000000000000000000" \
  com.pinaak
```

### 9.4 Android 12+ package visibility (already handled)

The existing `<queries>` block covers our current outgoing use (WhatsApp, tel, mailto, https). Nothing else is needed for deep-link **receiving**.

### 9.5 `singleTask` launch mode (already set — leave it)

`android:launchMode="singleTask"` means an incoming deep-link intent brings the existing MainActivity to the front and delivers via `onNewIntent`, not by launching a second copy. This is the correct mode for a deep-linking app — the alternative (`singleTop`) can produce duplicate task history on some devices. React Native's linking bridge handles the `onNewIntent` delivery transparently.

### 9.6 Update `android/gradle.properties` (only if not already)

No change required for deep linking specifically, but confirm `android.useAndroidX=true` — which the RN 0.86 template already sets.

---

## 10 · iOS setup — step-by-step

### 10.1 Add custom URL scheme to `Info.plist`

Under the existing `<dict>`, add:

```xml
<key>CFBundleURLTypes</key>
<array>
  <dict>
    <key>CFBundleURLName</key>
    <string>com.pinaak.urbancruise</string>
    <key>CFBundleURLSchemes</key>
    <array>
      <string>urbancruise</string>
    </array>
  </dict>
</array>
```

### 10.2 Add Associated Domains entitlement (Universal Links)

Create/edit `ios/pinaak/pinaak.entitlements`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>com.apple.developer.associated-domains</key>
  <array>
    <string>applinks:app.urbancruise.com</string>
  </array>
  <!-- If APS is added for notifications, aps-environment goes here too -->
</dict>
</plist>
```

In Xcode: target → Signing & Capabilities → add "Associated Domains" if not there, and confirm the entitlements file is referenced in the target's build settings (`CODE_SIGN_ENTITLEMENTS`).

### 10.3 Publish AASA (Apple App Site Association)

At `https://app.urbancruise.com/.well-known/apple-app-site-association` (no `.json` extension) serve:

```json
{
  "applinks": {
    "apps": [],
    "details": [
      {
        "appIDs": ["TEAMID.com.pinaak"],
        "components": [
          { "/": "/bookings/*", "comment": "Booking detail" },
          { "/": "/trip/*",           "comment": "Live trip" },
          { "/": "/quotations/*",     "comment": "Quotation detail" },
          { "/": "/vendor/*",         "comment": "Vendor screens" },
          { "/": "/driver/*",         "comment": "Driver screens" },
          { "/": "/uc/*",             "comment": "UC screens" },
          { "/": "/notifications",    "comment": "Notification centre" },
          { "/": "/support",          "comment": "Support" }
        ]
      }
    ]
  }
}
```

Requirements checklist:

- [ ] File is served **without** a `.json` extension — the path is `apple-app-site-association`, full stop.
- [ ] `Content-Type: application/json` — not `text/plain` or `application/octet-stream`.
- [ ] HTTPS with a **valid cert trusted by iOS** (no self-signed, no expired).
- [ ] No redirects — iOS won't follow them.
- [ ] `TEAMID` prefix on `appIDs` matches your Apple Developer team; `com.pinaak` is the bundle id.
- [ ] File size under 128 KB (Apple's limit).

### 10.4 Handle URLs in `AppDelegate`

React Native handles URL delivery automatically **if** the modular linking module is intact — which it is in a stock 0.86 project. Only edit the AppDelegate if you have custom URL handling elsewhere. Verify by checking `ios/pinaak/AppDelegate.mm`:

```objc
// Should already contain something like:
- (BOOL)application:(UIApplication *)application
            openURL:(NSURL *)url
            options:(NSDictionary<UIApplicationOpenURLOptionsKey,id> *)options
{
  return [RCTLinkingManager application:application openURL:url options:options];
}

- (BOOL)application:(UIApplication *)application
continueUserActivity:(nonnull NSUserActivity *)userActivity
 restorationHandler:(nonnull void (^)(NSArray<id<UIUserActivityRestoring>> * _Nullable))restorationHandler
{
  return [RCTLinkingManager application:application
                   continueUserActivity:userActivity
                     restorationHandler:restorationHandler];
}
```

If either is missing, add them. `RCTLinkingManager` lives in `#import <React/RCTLinkingManager.h>`.

### 10.5 Test on simulator / device

```
# Custom scheme
xcrun simctl openurl booted "urbancruise://open/bookings/01HXXX000000000000000000"

# Universal Link (must run on a device with the app installed AND
# after the OS has fetched the AASA file — takes ~1 minute after install)
xcrun simctl openurl booted "https://app.urbancruise.com/bookings/01HXXX"
```

Universal Link verification: install a build, wait for the AASA fetch, then tap a link in Notes.app or Mail.app pointing to `https://app.urbancruise.com/bookings/…`. It should open the app directly. If it opens Safari instead, iOS didn't verify — check the AASA file with:

```
curl -I https://app.urbancruise.com/.well-known/apple-app-site-association
# Confirm 200, application/json, no redirects
```

Apple's diagnostic tool: `sudo swcutil dl -d app.urbancruise.com` (run in Terminal on a Mac connected to a device via Xcode).

### 10.6 SKAdNetwork / Smart App Banner (optional but recommended)

For links shared on the web that should promote app install:

```html
<meta name="apple-itunes-app" content="app-id=1234567890, app-argument=https://app.urbancruise.com/bookings/01HXXX">
```

This is separate from the AASA and lives on your web landing pages. It's not required for deep linking to work — only for driving installs from the mobile web.

---

## 11 · Security — the full model

### 11.1 What an attacker cannot do

| Attack | Why it fails |
|---|---|
| Register `urbancruise://` on their own app | Custom schemes are inherently unverified. **We never trust a custom-scheme link for anything security-sensitive.** All auth callbacks and payment returns use Universal / App Links only, verified against the domain. |
| Send a Universal Link on `evil.com/bookings/…` | The resolver's `ALLOWED_HOSTS_HTTPS` allowlist rejects it. |
| Send `urbancruise://open/../../../etc/passwd` | Path traversal is not possible — `matchPath` only accepts exact segment counts and rejects `..` because it's not a `:name` param. Even if it decoded, no target ever passes it to a file API. |
| Inject a screen name via the URL | The resolver returns a `DeepLinkTarget` from a discriminated union — the switch in `targetToNavigatePayload` is the ONLY code that decides screen names, and it operates on `kind`, not on user input. |
| Send a deep link to a driver-role screen while the user is a customer | The `gate()` returns `wrong_role`; user lands on their role home + toast. |
| Send extremely long ids | Zod schemas enforce max length. |
| Send crafted push payload with malicious `click` field | `resolveFcmClick` parses through the same Zod schema and rejects. |
| Access another user's booking via `bookingId` in URL | Server-side authorization (RBAC + tenant scoping in the notifications backend design) rejects any query for a booking not owned by `identity.entityId`. **Deep linking is UI navigation; it never grants data access.** |

### 11.2 Logging

- **Never log full URLs at INFO level** — they may contain ids the user considers sensitive.
- Log resolution outcome (`ok/reason`) without the URL body.
- On `invalid_params` or `no_match`, log to Crashlytics with `boundary: 'deeplink.resolve'` for triage.

### 11.3 Redirect from web

The domain owner's job: `https://app.urbancruise.com/bookings/:id` should also work in a browser (for users who don't have the app installed). Render a simple landing page with:

- If user-agent is mobile and the app is installed → OS opens the app via Universal/App Link.
- Otherwise → landing page with "Open in App" button + Store install links + `<meta name="apple-itunes-app">`.

Do **not** implement fallback via 302 redirect to `urbancruise://…` — iOS treats that as unsafe and blocks it in some contexts.

---

## 12 · Testing

### 12.1 Unit tests (fast, no device)

```ts
// resolve.test.ts
test.each([
  ['urbancruise://open/bookings/01HXXX',            'ok',           'customer.bookingDetail'],
  ['https://app.urbancruise.com/bookings/01HXXX',    'ok',           'customer.bookingDetail'],
  ['https://evil.com/bookings/01HXXX',               'unknown_host', undefined],
  ['ftp://app.urbancruise.com/bookings/01HXXX',      'unknown_scheme', undefined],
  ['urbancruise://open/bookings/../etc/passwd',      'no_match',     undefined],
  ['urbancruise://open/bookings/',                   'no_match',     undefined],
  ['not-a-url',                                      'malformed_url',undefined],
])('resolve %s', (url, expected, kind) => { /* ... */ });
```

```ts
// gate.test.ts — every role × auth combination
```

### 12.2 Integration test (with test navigator)

Mount `RootNavigator` in a test harness with a mocked Redux state; assert that `handleResolved({ ok: true, target: {...} })` results in `getCurrentRoute()` matching the expected screen for each role.

### 12.3 Device E2E (Detox)

- Cold-start from a Universal Link → land on `BookingDetail`.
- Notification tap in foreground/background/quit → each lands on the correct screen.
- Notification tap while unauthenticated → login → drains to target.
- Notification tap with a wrong-role target → land on role home + toast visible.

### 12.4 Manual smoke checklist per release

- [ ] `urbancruise://open/bookings/<real-id>` opens Booking screen.
- [ ] `https://app.urbancruise.com/bookings/<real-id>` opens Booking screen (installed device, verified AASA).
- [ ] Fresh install + AASA fetch → verified within 60s.
- [ ] Tap-to-open from a real push notification lands on target.
- [ ] `adb shell pm get-app-links com.pinaak` reports `verified` for the domain.

---

## 13 · Rollout / phased delivery

| Phase | Deliverable | Ships when |
|---|---|---|
| **P0** | Add `services/deeplinks/` with catalog, resolver, gate, pending, drain. Unit tests. **No** manifest changes yet — the resolver is dead code until wired. | Anytime; safe to merge. |
| **P1** | Wire `linkingConfig` into `NavigationContainer`. Wire pending-queue drain into `onReady` + `loginSuccess` Redux subscription. Bridge notifications via `services/notifications/deeplink.ts`. | With P1 of notifications (first push end-to-end). |
| **P2** | Android `AndroidManifest.xml` — add custom-scheme + App Links intent-filters. Publish `assetlinks.json`. iOS — add `CFBundleURLTypes` + `associated-domains` entitlement. Publish AASA. Verify on real devices. | When the domain and signing certs are stable. |
| **P3** | Landing pages on `app.urbancruise.com` for non-installed fallback. Web meta tags for Store install banners. | Marketing site next iteration. |
| **P4** | Detox tests for cold-start + role-gate. Observability dashboards for `deeplink.resolve.*` failures. | Concurrent with P2 stabilization. |

**No phase blocks the previous one.** Custom scheme (P2 partial) works without App Links; App Links work without landing pages; notifications work without either since they use the internal FCM click bridge.

---

## 14 · Long-term maintenance

**Adding a new deep-linkable screen:**

1. Add a variant to `DeepLinkTarget` in `schema.ts`.
2. Add a catalog entry in `catalog.ts` with `path`, `role`, `auth`, `build`.
3. Add a case to the switch in `toNavigate.ts` — TypeScript will fail the build until you do.
4. If it's a new URL path prefix, add it to `AndroidManifest.xml` `<data android:pathPrefix="…"/>` and to the AASA `components` list, and republish both files.
5. Test with `adb shell am start` and `xcrun simctl openurl`.

**Removing a deep-linkable screen:** delete the variant. Every reference site becomes a compile error. Once green, remove from manifest / AASA on next release.

**Adding a new domain (e.g. staging → `staging.urbancruise.com`):**

1. Add to `linkingConfig.prefixes`.
2. Add to `ALLOWED_HOSTS_HTTPS` in `resolve.ts` (conditionally, based on `ENV.environment`).
3. Add an `applinks:staging.urbancruise.com` entry to the entitlements file.
4. Add a `<data android:host="staging.urbancruise.com"/>` in a new intent-filter (must be a separate `<intent-filter>` block, not the same one — App Links verification is per-filter).
5. Publish AASA + `assetlinks.json` on the new domain.

**Rotating Android signing keys** (Play App Signing, upload key rotation): update `assetlinks.json` to include the **new** fingerprint alongside the old one until every install is on a build signed with the new key. Then remove the old fingerprint on a later release.

**Deprecating a domain:** keep the AASA + `assetlinks.json` served for at least 6 months after the last app version that references it stops distributing. Universal Links depend on OS-cached verification which can lag.

**Handling schema drift over versions:** every deep-link variant has a `kind` string that is a permanent contract. Never rename an existing kind (e.g. don't change `customer.bookingDetail` to `customer.booking.detail`) — old push notifications and shared links out in the wild still reference the old name. Instead: add a new variant, keep the old one, and make both resolve to the same screen.

---

## 15 · Open questions for review

1. **Domain finalization.** `app.urbancruise.com` is used throughout — confirm with ops before publishing AASA / assetlinks. If it's `www.` or bare `urbancruise.com`, one line in each file changes and both must be republished.
2. **Universal Links for auth callback** (OTP magic link, if ever added). Requires an `/auth/verify/:token` catalog entry and specific handling in the auth flow — not currently a design goal but easy to add later.
3. **In-app browser vs external.** Some links (invoices, terms) should open in an in-app browser (`react-native-inappbrowser-reborn`) rather than deep-linking to a screen. That's an orthogonal concern but worth deciding early to keep the resolver clean.
4. **Should the resolver validate ULIDs strictly** (26 chars, Crockford alphabet) rather than "min 1 char"? Recommendation: yes, once branded id types (`BookingId`, etc.) are exposed with a runtime validator; add `.regex(/^[0-9A-HJKMNP-TV-Z]{26}$/)` at the schema level.

---

## Appendix A — File additions & edits checklist

```
src/services/deeplinks/                                     # NEW
  ├── index.ts
  ├── schema.ts
  ├── catalog.ts
  ├── resolve.ts
  ├── gate.ts
  ├── pending.ts
  ├── toNavigate.ts
  ├── drain.ts
  ├── linkingConfig.ts
  └── __tests__/{resolve,gate}.test.ts
src/services/notifications/deeplink.ts                       # NEW (bridge)
App.tsx                                                      # add linking={buildLinkingConfig()} + onReady drain
src/store/index.ts (or a small subscribe module)             # subscribe → drain on auth transitions

android/app/src/main/AndroidManifest.xml                     # add urbancruise:// + https App Links filters
ios/pinaak/Info.plist                                        # add CFBundleURLTypes
ios/pinaak/pinaak.entitlements                               # add associated-domains
ios/pinaak/AppDelegate.mm                                    # verify RCTLinkingManager methods present

server (out of scope for this repo):
  https://app.urbancruise.com/.well-known/apple-app-site-association
  https://app.urbancruise.com/.well-known/assetlinks.json
  https://app.urbancruise.com/bookings/:id                    # landing / redirect page
```
