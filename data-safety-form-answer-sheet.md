# GOOGLE PLAY DATA SAFETY FORM — Urban Cruise Answer Sheet

**App:** Urban Cruise
**Package:** app.urbancruise (verify against your `applicationId`)
**Prepared:** [DD Month YYYY]

This sheet mirrors the Play Console Data Safety form, section-by-section. Every answer is grounded in a specific code path or manifest declaration verified in the current codebase — sources cited in the *Verified from* column. Fill this exactly and your form will be consistent with `AndroidManifest.xml`, the Privacy Policy, and the deletion page.

---

## 0. TOP-LEVEL / OVERALL QUESTIONS

| Question | Answer | Verified from |
|---|---|---|
| Does your app collect or share any of the required user data types? | **Yes** | Auth flow, endpoints, Firebase telemetry |
| Is all of the user data collected by your app encrypted in transit? | **Yes** | `axios` over HTTPS; no cleartext traffic (`usesCleartextTraffic="${usesCleartextTraffic}"` gated per build type) |
| Do you provide a way for users to request that their data is deleted? | **Yes** | `https://urbancruise.in/delete-account` + in-app entry in Settings |
| URL for account deletion | `https://urbancruise.in/delete-account` | Deletion page draft |
| Privacy policy URL | `https://urbancruise.in/privacy-policy` | Privacy Policy draft |
| Committed to follow the Play [Families Policy](https://support.google.com/googleplay/android-developer/answer/9893335)? | **No** — app is 18+, not directed at children | Terms Clause II.1; Privacy Clause 14 |
| Do you undergo independent security review (e.g., MASA)? | **No** unless you plan to | — |

---

## 1. DATA TYPES — SECTION-BY-SECTION

For every "collected" item below, answer these four sub-questions unless noted otherwise:
- **Collected / Shared** — *Collected* means sent off the device; *Shared* means transferred to a third party (Firebase, payment gateway, Vendor Partner, etc.)
- **Optional / Required** — Required means the user can't use the core function without it
- **Purposes** — pick from: App functionality, Analytics, Developer communications, Advertising or marketing, Fraud prevention/security/compliance, Personalisation, Account management
- **Ephemeral processing** — leave *unchecked* (we persist)

### 1.1 Personal info

| Data type | Collected? | Shared? | Required? | Purposes | Verified from |
|---|---|---|---|---|---|
| **Name** | Yes | Yes (with assigned Driver/Vendor Partner; with payment gateway on invoice) | Required | App functionality, Account management | `/auth/me` returns `displayName`; passengers endpoint |
| **Email address** | Yes (if the user provides one — currently returned by `/auth/me`) | Yes (invoice, communication providers) | Optional | App functionality, Account management, Developer communications | `mockCurrentUser.email`; `/customer/bookings/{id}/payments/invoice` |
| **User IDs** | Yes | Yes (with Firebase Analytics via `setUserId`) | Required | App functionality, Analytics, Fraud prevention/security/compliance, Account management | `src/services/telemetry/analytics.ts` — `fbSetUserId` |
| **Address** | Yes (billing address for GST invoice; saved trip addresses) | Yes (invoice) | Optional | App functionality | `endpoints.customer.payments.gstInvoice`; saved-address flow |
| **Phone number** | Yes (mobile number for OTP auth is required) | Yes (with assigned Driver — masked where feasible; with SMS/WhatsApp providers) | Required | App functionality, Account management, Fraud prevention/security/compliance | `LoginScreen.tsx` phoneSchema `/^[6-9]\d{9}$/`; `/auth/otp/request` |
| Race and ethnicity | **No** | — | — | — | Not present in code |
| Political or religious beliefs | **No** | — | — | — | Not present |
| Sexual orientation | **No** | — | — | — | Not present |
| **Other info** — feedback, ratings, support-chat content | Yes | Yes (with support tool / cloud) | Optional | App functionality | Feedback / support flow |

### 1.2 Financial info

| Data type | Collected? | Shared? | Required? | Purposes | Verified from |
|---|---|---|---|---|---|
| **User payment info** — masked references only (last-4 / UPI ID / transaction ID). **Full card / UPI PIN never touch the app.** | Yes (server-side handled) | Yes (payment gateway) | Required (when paying online) | App functionality, Fraud prevention/security/compliance | `/customer/bookings/{id}/payments/pay` — payment executes via gateway, not in-app card form |
| **Purchase history** — booking history | Yes | Yes (invoicing; with corporate account admin if corporate booking) | Required | App functionality, Account management | `/customer/bookings/list`, `.../invoice` |
| Credit score | **No** | — | — | — | Not present |
| **Other financial info** — Vendor Partner / Driver **bank details for payouts** | Yes (Vendor / Driver roles only) | Yes (payment processor for payouts) | Required (for the payout flow) | App functionality, Fraud prevention/security/compliance | `endpoints.vendor.payments.payout`; `endpoints.uc.finance.payouts` |

### 1.3 Location

| Data type | Collected? | Shared? | Required? | Purposes | Verified from |
|---|---|---|---|---|---|
| **Approximate location** | Yes | Yes (with assigned Driver and UC operations during an active trip) | Required for Drivers on active trip; Optional for Customers | App functionality | `AndroidManifest.xml` `ACCESS_COARSE_LOCATION`; `src/rbac/capabilities.ts` `foregroundLocation` |
| **Precise location** | Yes | Yes (with assigned Driver and UC operations during an active trip) | Required for Drivers on active trip; Optional for Customers | App functionality | `AndroidManifest.xml` `ACCESS_FINE_LOCATION`; `DriverLocationForegroundService` |

**Critical follow-up questions inside the Location section:**

| Follow-up | Answer | Verified from |
|---|---|---|
| Is any of this data collected in the background? | **No** | `platformMap.ts`: no `BACKGROUND_LOCATION_PERM`; `capabilities.ts`: "`ACCESS_BACKGROUND_LOCATION` is INTENTIONALLY NOT declared"; manifest does not declare it |
| Is `ACCESS_BACKGROUND_LOCATION` declared? | **No** | `AndroidManifest.xml` line-by-line review — permission is absent |

Because you answer *No* to background collection, the Google Play [Location Permissions declaration form](https://support.google.com/googleplay/android-developer/answer/9799150) is **not required** for you.

### 1.4 Messages

| Data type | Collected? | Verified from |
|---|---|---|
| Emails | **No** — the app does not read the device inbox | No email-reading permission declared |
| SMS or MMS | **No** — no SMS-reading permission declared | Manifest has no `READ_SMS`; sheetHandlers explicitly note |
| **Other in-app messages** — support chat / booking chat inside the app | Yes | Yes (with support tool / cloud) | Optional | App functionality, Customer support | Feedback / support flows |

### 1.5 Photos and videos

| Data type | Collected? | Shared? | Required? | Purposes | Verified from |
|---|---|---|---|---|---|
| **Photos** — profile photo, KYC docs, RC / licence / insurance / permit, KM-meter, damage / trip-receipt | Yes | Yes (cloud storage) | Optional (Customer profile); Required (Driver / Vendor KYC) | App functionality, Fraud prevention/security/compliance | `capabilities.ts` — `camera`, `photoPicker`; `AndroidManifest.xml` `CAMERA` |
| Videos | **No** | — | — | — | No video-capture capability in code |

**Follow-up:** Photos are captured / picked only via the OS Camera or OS Photo Picker (sandboxed system UI). The app does **not** request `READ_MEDIA_IMAGES` / `READ_MEDIA_VIDEO` (both explicitly removed via `tools:node="remove"` in the manifest).

### 1.6 Audio files

| Data type | Collected? | Verified from |
|---|---|---|
| Voice or sound recordings | **No** | No `RECORD_AUDIO` permission declared |
| Music files | **No** | — |
| Other audio | **No** | — |

### 1.7 Files and docs

| Data type | Collected? | Shared? | Required? | Purposes | Verified from |
|---|---|---|---|---|---|
| **Files and docs** — PDFs and KYC document uploads (e.g., RC, insurance, permit) | Yes | Yes (cloud storage) | Required for KYC (Driver/Vendor); Optional otherwise | App functionality, Fraud prevention/security/compliance | `react-native-pdf`, `react-native-file-viewer`, `react-native-blob-util` in `package.json`; Driver / Vendor registration flow |

### 1.8 Calendar

| Data type | Collected? | Verified from |
|---|---|---|
| Calendar events | **No** | No calendar permission or library |

### 1.9 Contacts

| Data type | Collected? | Verified from |
|---|---|---|
| Contacts | **No** | Full-project grep: no `READ_CONTACTS`, no contacts library, no Contact Picker use |

### 1.10 App activity

| Data type | Collected? | Shared? | Required? | Purposes | Verified from |
|---|---|---|---|---|---|
| App interactions | Yes | Yes (Firebase Analytics — Google LLC) | Required | Analytics, App functionality | `services/telemetry/analytics.ts` — `fbLogEvent`, `fbLogScreenView` |
| In-app search history | **No** — no persistent search-history storage in code | — | — | — | Not implemented |
| Installed apps | **No** | — | — | — | No `QUERY_ALL_PACKAGES`; `queries` block only lists WhatsApp + intent filters |
| **Other user-generated content** — feedback, ratings, remarks on quotations | Yes | Yes (cloud) | Optional | App functionality | `endpoints.customer.quotations.addRemark`; feedback flow |
| **Other actions** — booking events, trip acknowledgements | Yes | Yes (backend + Analytics) | Required | App functionality, Analytics | endpoints for `.acknowledge`, `.startLeg`, `.endLeg` |

### 1.11 Web browsing

| Data type | Collected? | Verified from |
|---|---|---|
| Web browsing history | **No** | Not applicable |

### 1.12 App info and performance

| Data type | Collected? | Shared? | Required? | Purposes | Verified from |
|---|---|---|---|---|---|
| **Crash logs** | Yes | Yes (Firebase Crashlytics — Google LLC) | Required | Analytics, App functionality | `services/telemetry/logError.ts` — `getCrashlytics`, `recordError` |
| **Diagnostics** — session length, performance metrics | Yes | Yes (Firebase) | Required | Analytics, App functionality | `@react-native-firebase/analytics` bundled |
| Other app performance data | **No** unless you enable Firebase Performance Monitoring | — | — | — | Not currently in `package.json` |

### 1.13 Device or other IDs

| Data type | Collected? | Shared? | Required? | Purposes | Verified from |
|---|---|---|---|---|---|
| **Device or other IDs** — Firebase Installation ID, FCM push token, device model/OS collected by Firebase | Yes | Yes (Firebase, push-notification providers) | Required | App functionality, Analytics, Fraud prevention/security/compliance | `@react-native-firebase/messaging`, `@react-native-firebase/analytics`; `react-native-device-info` in `package.json` |

**Note on Advertising ID:** the app does not have any explicit `AdvertisingId` API usage in code. However, Firebase Analytics may collect the Android Advertising ID by default unless `google_analytics_adid_collection_enabled` is set to `false`. **Recommendation:** if you do not use it for ads, add this to `AndroidManifest.xml` inside `<application>` and answer *No* for Advertising ID in the form:

```xml
<meta-data android:name="google_analytics_adid_collection_enabled" android:value="false" />
```

### 1.14 Personal identifiers (Government IDs)

Play's form doesn't have a dedicated "Government IDs" section — declare it under **Personal info → Other info** or **Files and docs** (image/scan) depending on how you store it.

| Data type | Collected? | Shared? | Required? | Purposes | Verified from |
|---|---|---|---|---|---|
| **KYC document images** (PAN / Aadhaar / driving licence / RC / insurance / permit) | Yes (Driver / Vendor roles only) | Yes (KYC / background-verification agencies) | Required (for those roles) | App functionality, Fraud prevention/security/compliance | Driver / Vendor registration flows (`endpoints.driver.registration`, `endpoints.vendor.drivers.create`, `.approve`) |

### 1.15 Health and fitness

**No** — app does not collect any health, fitness, or Health Connect data.

---

## 2. DATA SHARING SECTION

The form asks you to confirm which categories are *shared* with third parties. The following list summarises every third party your app currently sends data to, matched to a code path:

| Recipient | Data shared | Why | Verified from |
|---|---|---|---|
| **Firebase / Google LLC** — Analytics, Crashlytics, Cloud Messaging | User ID, event names, screen views, device metadata, crash stack traces, FCM tokens | Analytics, crash reporting, push delivery | `services/telemetry/analytics.ts`, `logError.ts`, `@react-native-firebase/messaging` |
| **Payment gateway(s)** (declare specific ones you integrate — e.g., Razorpay / PayU) | Name, email, mobile, amount, order/transaction reference | Payment processing | `/customer/bookings/{id}/payments/pay` |
| **SMS / WhatsApp / Email providers** | Mobile number, email, message content (OTP, booking updates) | Communication delivery | Auth OTP flow |
| **Cloud infrastructure** (AWS / GCP / whichever you use) | All server-side data | Hosting | Backend hosting arrangement |
| **KYC / background-verification agencies** | Driver / Vendor KYC documents and identifiers | Verification | Driver / Vendor onboarding |
| **Assigned Driver / Vendor Partner** | Customer name, contact number, pick-up/drop | Trip fulfilment | `endpoints.vendor.assignments`, `endpoints.driver.trips` |
| **Corporate account admin** | Employee trip and booking data | Corporate billing | UC / corporate booking flow |
| **Law-enforcement / regulators** (only when legally required) | As required | Legal compliance | Privacy Policy Clause 5(D) |

Where the form asks *whether data is shared for advertising or marketing purposes*, answer **No** unless you actually integrate an ad SDK.

---

## 3. DATA SECURITY SECTION

| Question | Answer | Verified from |
|---|---|---|
| Data is encrypted in transit | **Yes** | HTTPS enforced; `usesCleartextTraffic` gated by build type; iOS `NSAllowsArbitraryLoads = false` |
| You provide a way for users to request data deletion | **Yes** | `https://urbancruise.in/delete-account` — in-app link + web form + email |
| You follow the [Families Policy](https://support.google.com/googleplay/android-developer/answer/9893335) | **No** — 18+ app | Terms II.1 |

---

## 4. DECLARATIONS YOU DO NOT NEED

Because of what the app does and does not do, you can **skip** the following extra declaration forms in Play Console:

| Declaration | Why not needed for you |
|---|---|
| Location Permissions declaration | You do not declare `ACCESS_BACKGROUND_LOCATION` |
| Foreground Services (background access) declaration | Your foreground service is user-initiated (Driver taps "Start Trip"), of type `location`, with persistent notification — Google's standard FGS-location pattern |
| SMS/Call-Log declaration | Neither permission is declared |
| Financial features declaration | You process trip payments through a licensed gateway; you are not a financial-services / lending app. Confirm by reviewing the [in-scope categories](https://support.google.com/googleplay/android-developer/answer/13849271) — if any of those match, complete it |
| Health apps declaration | No health/fitness data |
| News and magazines declaration | Not a news app |
| Government / COVID / Elections declarations | Not applicable |
| Advertising ID declaration | Only if you disable Firebase's default AAID collection using the meta-data flag above; else declare it |

---

## 5. FINAL CROSS-CHECK CHECKLIST BEFORE HITTING "SUBMIT FOR REVIEW"

1. **Match check** — every "Yes, collected" row in this sheet appears in the Privacy Policy v2 (Clauses 2, 3, 5) and vice-versa.
2. **URL check** — Privacy Policy and Deletion URLs load from a public browser (no login, not geofenced, not a PDF).
3. **Permissions check** — every permission declared in `AndroidManifest.xml` is used somewhere in code (unused permissions = Play rejection).
4. **In-app disclosure check** — for `ACCESS_FINE_LOCATION`, `CAMERA`, `POST_NOTIFICATIONS` — the app shows a plain-language explanation *before* the OS prompt.
5. **AAID decision** — add the `google_analytics_adid_collection_enabled=false` meta-data if you do not use Advertising ID, then answer *No* to Advertising ID in the form.
6. **Payment gateway** — list the specific gateway name (Razorpay / PayU / etc.) in your Privacy Policy Clause 5(B) sub-list.
7. **Data-type consistency** — your Data Safety declaration and your Privacy Policy Clause 2 must list the **same** categories. Reviewers grep for mismatches.
8. **Screens / description** — booking screens shouldn't show categories not declared here (e.g., don't add a "share your contacts" feature without updating this form first).

---

*This sheet is prepared from a direct audit of the Urban Cruise codebase (`AndroidManifest.xml`, `Info.plist`, `src/rbac/capabilities.ts`, `src/services/permissions/*`, `src/api/endpoints.ts`, `src/services/telemetry/*`) as of the date above. If you add a new feature that touches new data (contacts, calendar, microphone, health, background location, etc.), update this sheet, the Privacy Policy, and the Data Safety form together — never in isolation.*
