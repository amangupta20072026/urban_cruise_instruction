# Customer Caller Identification on Incoming Calls
## Android + React Native Technical Architecture & Implementation Proposal

**Document Type:** Technical Architecture / Feasibility & Implementation Proposal  
**Platform:** Android  
**Application:** Existing React Native Application  
**Primary Use Case:** Identify existing customers when they call the company's Sales Team  
**Audience:** Engineering Manager, Team Manager, Product/Technical Stakeholders  
**Status:** Proposed Architecture  
**Research Basis:** Official Android Developers, AndroidX, React Native, and Google Play documentation

---

# 1. Executive Summary

The company currently maintains a centralized customer database containing customer information such as:

- Customer name
- Mobile phone number
- Email
- Customer ID
- Booking/service information
- Other business-related customer information

The company also has a React Native mobile application with multiple user roles, including:

- Vendor
- Driver
- Customer
- Company Admin

A new requirement has been identified for the Sales Team:

> When an existing customer calls a Sales Team member's Android phone using a normal cellular phone call, the Sales Team member should be able to identify the customer immediately from the incoming-call experience, without manually searching for the phone number in the application.

This requirement must work even when the React Native application is:

- Not open
- In the background
- Removed from the recent-apps screen
- Not actively running as a React Native JavaScript process

The recommended Android architecture is based on the official Android Telecom framework and specifically:

**`CallScreeningService` + `ROLE_CALL_SCREENING` + local caller-ID data.**

Android officially documents `CallScreeningService` for both call screening and **call identification/caller ID**. Android Telecom binds the user-selected call-screening application when a new incoming call is received. The service receives the call details and can provide a caller-identification UI.

The architecture should not depend on keeping the React Native JavaScript application process permanently alive.

---

# 2. Business Requirement

## 2.1 Current Situation

The company has a central customer database.

Example:

| Customer ID | Name | Mobile Number |
|---|---|---|
| CUST-1001 | Rahul Sharma | +91XXXXXXXXXX |
| CUST-1002 | Priya Verma | +91XXXXXXXXXX |

A customer calls the company's Sales Team.

The Sales Team member currently receives a normal phone call.

The desired experience is:

```text
Customer calls Sales Team
        ↓
Android receives incoming cellular call
        ↓
System identifies caller number
        ↓
Application matches number against customer data
        ↓
Customer identity is displayed
        ↓
Sales representative answers the call
        ↓
Sales representative immediately knows who is calling
```

---

# 3. Exact Functional Requirement

The system should:

1. Detect incoming cellular calls.
2. Obtain the incoming caller's phone number when Android makes it available to the call-screening service.
3. Normalize the phone number.
4. Search for the number in the locally available customer index.
5. Determine whether the caller is an existing customer.
6. Display the customer's identifying information through the supported caller-ID experience.
7. Allow the normal phone call to continue.
8. Avoid blocking or rejecting the call unless a future business requirement explicitly asks for call screening.
9. Work independently of the React Native JavaScript UI being open.
10. Synchronize customer changes from the central backend to the Sales Team devices.

---

# 4. Important Technical Clarification

There are two concepts that must not be confused.

## 4.1 Call Detection

Android provides an official API for a third-party application to participate in incoming-call screening and caller identification:

`android.telecom.CallScreeningService`

Android explicitly states that `CallScreeningService` can provide:

- Call blocking/screening
- Call identification

For caller identification, Android says that the service can display a user interface of its choosing containing identifying information about the call.

## 4.2 Native Phone Application UI

Android device manufacturers may customize the actual Phone/Dialer interface.

Therefore, the implementation must **not promise that the company's application can arbitrarily modify every manufacturer's native incoming-call screen**.

Instead, the technically defensible requirement is:

> The application will use Android's official caller-ID mechanism to identify the caller and present customer-identification information through the supported caller-ID experience.

The exact visual placement can vary by Android version, device manufacturer, and Phone application.

This distinction is important for production expectations.

---

# 5. Official Android API

## 5.1 CallScreeningService

The core Android component is:

```text
android.telecom.CallScreeningService
```

Android documents this service as an API that can be implemented by the default dialer or a third-party application for call screening and caller identification.

The service is bound by Android Telecom when a new incoming or outgoing call is available.

The service receives:

```text
onScreenCall(Call.Details)
```

For an incoming call, the application can obtain the call handle/phone number and perform the caller identification process.

---

# 6. Call Screening Role

The application must qualify for and obtain the Android role:

```text
ROLE_CALL_SCREENING
```

Android defines this role as:

> The call screening and caller ID role.

The role is user-consented and controlled by Android's RoleManager.

The application should check whether the role is available and then request the role from the user.

Conceptually:

```text
Sales Team installs application
        ↓
Application checks ROLE_CALL_SCREENING
        ↓
Android displays system role-consent UI
        ↓
Sales Team member approves
        ↓
Application becomes the selected CallScreeningService
        ↓
Android Telecom can bind the service for calls
```

Android explicitly states that Telecom binds to a **single app chosen by the user** for the call-screening role.

---

# 7. Required Native Android Component

The React Native application will require native Android code.

The core native component will be something similar to:

```text
CustomerCallScreeningService.kt
```

It will extend:

```kotlin
CallScreeningService
```

The Android manifest will declare the service with:

```xml
<service
    android:name=".telecom.CustomerCallScreeningService"
    android:permission="android.permission.BIND_SCREENING_SERVICE"
    android:exported="true">

    <intent-filter>
        <action android:name="android.telecom.CallScreeningService" />
    </intent-filter>

</service>
```

Android officially documents this service registration pattern.

`BIND_SCREENING_SERVICE` is a system/signature-level binding permission; it is used so that the Android system can bind to the call-screening service.

---

# 8. React Native's Role

The existing application does not need to be rewritten.

React Native remains responsible for:

- Existing application UI
- Authentication
- Customer management
- Admin functionality
- Sales UI
- Backend communication
- Configuration
- Synchronization controls
- User settings

Android native code will be responsible for:

- `CallScreeningService`
- Android Telecom integration
- Call-screening role
- Incoming-call processing
- Local caller lookup
- Android caller-identification UI integration

React Native officially supports integrating native platform functionality through native modules.

Therefore, the architecture is:

```text
                 React Native Application
                         │
        ┌────────────────┼────────────────┐
        │                │                │
        ▼                ▼                ▼
   Business UI       Backend API      Settings
        │
        │ Native Module / Native Android
        ▼
   Android Telecom Layer
        │
        ▼
CallScreeningService
        │
        ▼
Local Caller Database
```

---

# 9. Why React Native Alone Is Not Enough

React Native JavaScript code is not the appropriate layer to directly implement the Android Telecom callback.

The incoming call can occur while the React Native UI is not running.

The Android operating system owns the Telecom lifecycle.

Therefore:

```text
WRONG ARCHITECTURE

Incoming Call
     ↓
React Native JS listener
     ↓
Find Customer
```

This would create an unnecessary dependency on the application process.

Instead:

```text
CORRECT ARCHITECTURE

Incoming Call
     ↓
Android Telecom
     ↓
CallScreeningService
     ↓
Local Customer Index
     ↓
Caller Identification
```

The React Native application becomes the configuration and synchronization layer rather than the real-time call-processing layer.

React Native officially provides native integration mechanisms specifically for platform APIs that are not directly exposed by React Native.

---

# 10. Critical 5-Second Constraint

This is one of the most important requirements.

Android states that for incoming calls, `CallScreeningService` must call:

```text
respondToCall(...)
```

within **5 seconds**.

If the service does not respond in time, Android can stop waiting and the response will be ignored.

Android also states that the device will not begin ringing until the response is received or the timeout occurs.

Therefore:

> The incoming-call path must never depend on a slow remote database request.

---

# 11. Why We Should NOT Query the Central Database During the Call

A bad implementation would be:

```text
Incoming Call
      ↓
Get Phone Number
      ↓
HTTPS API request
      ↓
Backend
      ↓
Database
      ↓
Customer
      ↓
Response
```

Problems:

- Network latency
- Poor mobile connectivity
- Backend latency
- Database latency
- API timeout
- Server outage
- VPN/network restrictions
- Android 5-second call-screening constraint

This could delay the incoming-call flow.

Android specifically notes that local database lookups may be used for call-screening decisions and warns about repeatedly hitting the timeout.

---

# 12. Recommended Local Customer Index

The recommended architecture is:

```text
Central Customer Database
          │
          │ Secure synchronization
          ▼
Sales Android Device
          │
          ▼
Local Caller-ID Database
```

The local database should contain only the minimum information required for caller identification.

Example:

```text
customer_id
normalized_phone_number
display_name
customer_type
status
last_synced_at
```

Potentially:

```text
company_name
customer_reference
```

Only information required for the caller-ID experience should be stored locally.

---

# 13. Local Database Technology

A local Android database can be implemented using an Android-supported local persistence solution.

The exact database technology should be selected during implementation based on:

- Existing project architecture
- React Native architecture
- Android minimum SDK
- Data volume
- Encryption requirements
- Synchronization requirements
- Team expertise

The most important architectural requirement is not the database brand.

The requirement is:

> The caller number must be indexed and available locally with extremely low lookup latency.

Example index:

```text
PRIMARY/UNIQUE INDEX

normalized_phone_number
```

Lookup:

```text
+919871234567
       ↓
Rahul Sharma
```

The lookup should be local and deterministic.

---

# 14. Phone Number Normalization

Phone numbers must be normalized before storage and lookup.

For example:

```text
+91 98712 34567
091-9871234567
9871234567
+919871234567
```

may represent the same customer number depending on the country's dialing rules.

The system should define one canonical representation.

Recommended format:

```text
E.164
```

Example:

```text
+919871234567
```

The backend and mobile local index should use the same normalization rules.

This prevents:

```text
Database:
+91 98712 34567

Incoming call:
9871234567

Result:
NOT FOUND
```

when both actually represent the same customer.

---

# 15. Customer Synchronization Architecture

The central backend remains the **source of truth**.

The Sales device contains a synchronized caller-ID index.

Recommended flow:

```text
                    CENTRAL BACKEND
                          │
                          │
                 Customer Master Data
                          │
                          ▼
                  Sync API / Endpoint
                          │
                  HTTPS + Authentication
                          │
                          ▼
                 Sales Android Device
                          │
                          ▼
                  Local Customer Index
```

---

# 16. Initial Synchronization

When the Sales Team member first configures the device:

```text
Login
  ↓
Validate Sales role
  ↓
Download permitted customer caller-ID dataset
  ↓
Normalize phone numbers
  ↓
Build local index
  ↓
Mark sync complete
```

The application should not need to download unnecessary customer information.

A minimal caller-ID dataset is preferable.

---

# 17. Incremental Synchronization

Downloading the entire customer database repeatedly is not scalable.

Instead, use incremental synchronization.

Example:

```text
GET /customer-caller-id/sync?since=2026-08-28T10:00:00Z
```

Backend returns:

```text
Added customers
Updated customers
Deleted/deactivated customers
```

Example response:

```json
{
  "syncToken": "abc123",
  "changes": [
    {
      "operation": "upsert",
      "customerId": "CUST-1001",
      "phone": "+919871234567",
      "displayName": "Rahul Sharma"
    }
  ]
}
```

The exact API contract should be defined by the backend team.

---

# 18. Sync Failure Strategy

The caller-ID feature should continue working using the **last known local dataset** if synchronization temporarily fails.

Example:

```text
Internet available
      ↓
Sync succeeds
      ↓
Local DB updated
```

If the Internet later disappears:

```text
Internet unavailable
      ↓
Incoming Call
      ↓
Local database
      ↓
Existing customer identified
```

The device should not become dependent on live Internet connectivity for every incoming call.

---

# 19. Data Freshness

Because the local database may not always contain the latest customer data, the application should maintain:

```text
lastSuccessfulSync
```

and optionally:

```text
syncVersion
syncToken
recordUpdatedAt
```

The UI/settings screen can show:

```text
Caller ID
Connected

Last synchronized:
28 Aug 2026, 8:30 PM
```

If synchronization has not happened for a long period, the application should warn the Sales Team.

---

# 20. Incoming Call Workflow

The production flow should be:

```text
1. Customer calls Sales phone
        ↓
2. Android Telecom receives the call
        ↓
3. Android identifies the selected CallScreeningService
        ↓
4. Android binds the service
        ↓
5. onScreenCall(Call.Details)
        ↓
6. Determine incoming call
        ↓
7. Extract caller handle/number
        ↓
8. Normalize number
        ↓
9. Local database lookup
        ↓
10. Customer found?
       / \
     YES  NO
      │    │
      │    └── Generic/unknown caller experience
      │
      ▼
11. Customer identity available
      ↓
12. Allow call
      ↓
13. Caller-identification UI
      ↓
14. Sales representative answers
```

---

# 21. Call Must Not Be Blocked

The business requirement is identification, not blocking.

Therefore the service should normally respond with an allow/no-block response.

Conceptually:

```text
Customer found:
    Identify customer
    Allow call

Customer not found:
    No customer identity
    Allow call
```

The caller-ID feature must not accidentally become a call-blocking system.

---

# 22. Customer Lookup Performance

The call-screening path should be extremely small.

Recommended:

```text
Phone Number
     ↓
Normalization
     ↓
Indexed local lookup
     ↓
Customer result
```

Avoid:

- Network calls
- Complex joins
- Heavy computation
- Image downloads
- Large JSON parsing
- Analytics requests
- Logging full customer profiles
- Authentication refresh during call screening

The call-screening service should do the minimum required work.

---

# 23. Caller-ID UI

Android's official documentation states that a caller-ID service can display a UI of its choosing containing identifying information.

Android also states that if the app provides a caller-ID experience, it should launch an activity to show caller-ID information from `onScreenCall`.

AndroidX documentation for `ROLE_CALL_SCREENING` also states that this role enables caller identification and, on Android 11 and above, allows the application to display over other apps.

Therefore the UI can conceptually contain:

```text
--------------------------------
        Incoming Call

        Rahul Sharma
        Existing Customer

        Customer ID:
        CUST-1001

        Company:
        ABC Travels

        +91 98712 34567

        [Answer]
        [Decline]
--------------------------------
```

However:

> The exact visual placement and behavior must be validated on the target Android devices because the system Phone application and OEM software control the native call surface.

This must be treated as a device-compatibility requirement.

---

# 24. Android Version Compatibility

`CallScreeningService` was introduced in:

```text
Android API 24
Android 7.0
```

Android's official API reference lists `CallScreeningService` as added in API level 24.

The call-screening role:

```text
ROLE_CALL_SCREENING
```

was added in API level 29.

The supported production device matrix should therefore be defined explicitly.

Recommended policy:

```text
Minimum supported Android version
        ↓
Validate with target company devices
        ↓
Test Android 11+
        ↓
Test current supported Android releases
        ↓
Test major OEM devices
```

Do not assume that behavior is identical across Samsung, Xiaomi, OnePlus, Vivo, Oppo, Pixel, etc.

---

# 25. Contacts Behavior

Android's documentation contains an important detail:

Only calls whose handle uses the telephone scheme are passed for screening, and calls from numbers already in the user's contacts are not passed to the `CallScreeningService` unless the service has `READ_CONTACTS` permission.

Therefore this must be addressed during implementation.

There are two possible business strategies:

### Strategy A — Sales devices are managed company devices

The company can define a controlled device/contact policy.

### Strategy B — Application requests READ_CONTACTS

If required, the application can request the appropriate Contacts permission and explain clearly why it is needed.

The team should not automatically request permissions that are not required.

---

# 26. Call Log Permissions

The implementation should **not assume that `READ_CALL_LOG` is required simply because the feature concerns phone calls**.

The core caller-identification mechanism is `CallScreeningService`.

Google Play treats Call Log permissions as highly sensitive and restricts their use. Google Play's policy states that apps requesting Call Log permissions must meet permitted-use requirements, with default Phone/Assistant handling being a major condition, and other permitted cases are subject to policy review.

Therefore:

> Do not add `READ_CALL_LOG`, `WRITE_CALL_LOG`, or `PROCESS_OUTGOING_CALLS` unless a separately justified feature requires them and the current Play policy permits that use.

Minimize permissions.

---

# 27. Permissions / Access Requirements

The expected architecture should distinguish between:

## Android service binding

```text
BIND_SCREENING_SERVICE
```

This is used by Android to bind the system to the service and is a signature/privileged permission.

## User role consent

```text
ROLE_CALL_SCREENING
```

This is a user-controlled Android role.

## Potential Contacts permission

```text
READ_CONTACTS
```

Only if required by the selected contact-handling strategy. Android documents its effect on whether calls from contacts are provided to the screening service.

## Network permission

The application will require normal Internet access for synchronization with the backend.

---

# 28. Authentication

The local caller-ID database must belong to the authenticated Sales Team user/device.

Recommended architecture:

```text
User Login
    ↓
Authentication
    ↓
Sales role validation
    ↓
Device registration
    ↓
Caller-ID dataset authorization
```

The backend should verify:

```text
user
role
device
organization
permissions
```

before delivering customer caller-ID data.

---

# 29. Device Registration

Each company Sales phone should ideally be registered.

Example:

```text
Device ID
User ID
Organization ID
App Version
Android Version
Last Sync
Status
```

Backend example:

```text
DEVICE-001
sales.user@company.com
COMPANY-001
Android 15
App 2.4.0
Last Sync: 2026-08-28
ACTIVE
```

This makes device management and revocation possible.

---

# 30. Device Revocation

If an employee leaves the company:

```text
Admin disables user
        ↓
Authentication revoked
        ↓
Device sync token revoked
        ↓
Future sync requests rejected
```

The local database should also be cleared when the account/device is signed out or decommissioned.

This is important because customer phone numbers and names are personal information.

---

# 31. Security Architecture

Customer caller-ID information is sensitive business data.

Recommended controls:

### In transit

Use:

```text
HTTPS/TLS
```

### Backend

Use:

```text
Authentication
Authorization
Role-based access
Device registration
Audit logging
Rate limiting
```

### Local storage

Use protected/encrypted storage where appropriate.

Sensitive authentication tokens should not be stored in plain text.

### Data minimization

Store only what caller identification needs.

Example:

```text
customer_id
normalized_phone
display_name
```

rather than the entire customer profile.

---

# 32. Privacy

The application must clearly explain that it uses customer information to provide the company's caller-identification functionality.

The application should have:

- Privacy Policy
- Data usage explanation
- Appropriate consent/notice
- Data retention policy
- Data deletion process
- Access control

Google Play requires appropriate handling and disclosure for sensitive user/device data and provides app-review requirements around privacy and sensitive permissions.

---

# 33. Google Play Considerations

Google Play has specific restrictions around sensitive Call Log and SMS permissions.

The application should therefore avoid unnecessary call-log permissions.

If the final implementation requests restricted permissions, the team must review the current Google Play policy and complete any required declarations before release. Google Play explicitly states that apps requesting high-risk or sensitive permissions may need to complete a Permissions Declaration Form.

The Play Store submission should document:

```text
Why caller identification is a core feature
Why each sensitive permission is required
How user/customer data is handled
How data is protected
```

---

# 34. No Accessibility Hack

The implementation should NOT use:

```text
AccessibilityService
```

to inspect or manipulate the Phone application.

Accessibility APIs are intended for accessibility use cases and should not be used as a workaround for official Telecom APIs.

The official Android caller-ID architecture already provides:

```text
CallScreeningService
ROLE_CALL_SCREENING
```

Therefore the official Telecom mechanism should be used.

---

# 35. No Fake Overlay Dependency

The solution should not depend on arbitrary floating overlays such as:

```text
SYSTEM_ALERT_WINDOW
```

unless there is a separately justified requirement and Android policy allows it.

The preferred approach is the official caller-ID role and its supported UI mechanisms.

AndroidX explicitly documents that the call-screening role can provide caller identification and display over other apps on Android 11+.

---

# 36. Why the React Native App Can Be Closed

The architecture does not require:

```text
React Native JS process
```

to remain permanently alive.

Android Telecom owns the lifecycle of the call-screening service.

When an incoming call occurs:

```text
Android Telecom
       ↓
binds CallScreeningService
       ↓
service processes call
```

This is fundamentally different from:

```text
React Native background timer
```

or:

```text
JavaScript polling
```

Therefore the caller-ID functionality can be designed independently of whether the React Native UI is currently open.

Android officially states that Telecom binds to the user-selected `CallScreeningService` when new calls are received.

---

# 37. App Force-Stop Caveat

There is an important operational distinction between:

```text
App not open
```

and:

```text
User explicitly force-stopped the application
```

Android has system-level lifecycle behavior around force-stopped applications.

Therefore production testing must explicitly test:

- App not opened after reboot
- App in background
- App removed from recents
- Device locked
- Device unlocked
- Network disconnected
- Network connected
- Battery saver
- OEM battery optimization
- Application force-stop
- Device reboot

The final supported behavior must be verified on the exact managed-device fleet.

---

# 38. Reboot Behavior

The application should not rely on a continuously running process.

The Android Telecom role/service relationship should be validated after:

```text
Device reboot
```

The onboarding/health-check screen should verify:

```text
Caller ID enabled
Call Screening role granted
Last synchronization successful
Local database available
```

---

# 39. OEM Compatibility

Android provides the official API, but Android devices can have manufacturer-specific Phone applications and power-management behavior.

The company should maintain a supported-device matrix.

Example:

| Device | Android | Call Screening | Caller UI | Background Behavior |
|---|---:|---|---|---|
| Pixel | 15 | Test | Test | Test |
| Samsung | 15 | Test | Test | Test |
| OnePlus | 15 | Test | Test | Test |
| Xiaomi | 15 | Test | Test | Test |
| Vivo | 15 | Test | Test | Test |
| Oppo | 15 | Test | Test | Test |

The exact devices should be based on the company's Sales Team hardware.

---

# 40. Offline Behavior

Offline operation should be a first-class requirement.

If:

```text
Internet = OFF
```

then:

```text
Incoming call
     ↓
Local lookup
     ↓
Customer identified if data exists locally
```

The system should not fail just because the Internet is unavailable.

The only limitation is that newly created/updated customer records may not appear until synchronization occurs.

---

# 41. Unknown Customer

If the number does not exist locally:

```text
Incoming number
       ↓
Local lookup
       ↓
No match
```

The system should gracefully continue with:

```text
Unknown Caller
```

or the normal system caller experience.

The call must still be allowed.

---

# 42. Duplicate Phone Numbers

The backend must define the business rule for:

```text
one phone number → multiple customer records
```

Recommended policy:

```text
normalized phone number
        ↓
one canonical caller identity
```

If duplicates are possible, the backend should determine which customer record is authoritative.

Do not let the Android device arbitrarily select one record.

---

# 43. Multiple Phone Numbers Per Customer

A customer may have:

```text
Primary number
Secondary number
Office number
Alternate number
```

The caller-ID index can support:

```text
phone_number → customer_id
```

with multiple phone numbers mapping to one customer.

Example:

```text
+919871234567 → CUST-1001
+919876543210 → CUST-1001
```

---

# 44. Number Spoofing

Caller ID itself is not a cryptographic identity of the caller.

A caller can potentially use caller-ID spoofing techniques.

Android 11 introduced support for requesting STIR/SHAKEN verification information through `CallScreeningService` where available.

Therefore:

> Customer identification should be treated as a caller-ID match, not as cryptographic proof of customer identity.

For high-risk operations, the Sales representative should still verify customer identity using normal business procedures.

---

# 45. Backend API Design

A possible backend architecture:

```text
GET /caller-id/sync
```

Parameters:

```text
since
syncToken
deviceId
```

Response:

```json
{
  "syncToken": "SYNC-123",
  "changes": [
    {
      "operation": "UPSERT",
      "customerId": "CUST-1001",
      "phone": "+919871234567",
      "displayName": "Rahul Sharma"
    }
  ]
}
```

Delete:

```json
{
  "operation": "DELETE",
  "customerId": "CUST-1001",
  "phone": "+919871234567"
}
```

The final contract should be defined by the backend team.

---

# 46. Backend Database Index

The central database should have an index on normalized phone number.

Conceptually:

```sql
INDEX idx_customer_phone
ON customers(normalized_phone_number);
```

If business rules permit one-to-one mapping:

```text
UNIQUE(normalized_phone_number)
```

Otherwise:

```text
INDEX(normalized_phone_number)
```

and duplicate resolution must be explicit.

---

# 47. Local Database Index

The local database should also have:

```text
INDEX normalized_phone_number
```

The goal is:

```text
incoming phone
       ↓
O(very small lookup)
       ↓
customer
```

not:

```text
scan entire customer table
```

---

# 48. Data Synchronization Frequency

Recommended options:

### Option A — App startup + periodic sync

```text
Login
App open
Periodic background sync
```

### Option B — Push-triggered synchronization

Backend informs devices that data changed.

### Option C — Hybrid

Recommended:

```text
Push/update signal
        ↓
Device performs secure incremental sync

+
Periodic reconciliation
```

The caller-ID lookup itself remains local.

---

# 49. What Happens When a Customer Is Added?

Example:

```text
Customer created:
Rahul Sharma
+919871234567
```

Backend:

```text
Customer DB updated
        ↓
Sync system marks change
        ↓
Sales device sync
        ↓
Local caller index updated
```

Next call:

```text
+919871234567
        ↓
Rahul Sharma
```

---

# 50. What Happens When Customer Data Changes?

Example:

```text
Old:
Rahul Sharma

New:
Rahul Kumar Sharma
```

Backend change:

```text
Customer updated
        ↓
Incremental sync
        ↓
Local record replaced
```

No app reinstall is required.

---

# 51. What Happens When Customer Is Deleted?

The backend sends:

```text
DELETE / deactivation
```

The device removes the number from its caller-ID index.

This is important for privacy and data correctness.

---

# 52. React Native Application Changes

The existing React Native application should add a small configuration/administration section such as:

```text
Settings
   ↓
Caller ID
```

Example:

```text
Caller ID

Status:
Enabled

Call Screening Role:
Granted

Local Database:
Ready

Last Sync:
28 Aug 2026, 8:30 PM

Customers:
125,432

[Sync Now]

[Manage Caller ID]
```

This UI is React Native.

---

# 53. Native Module Responsibilities

A React Native native module can expose functions such as:

```text
isCallScreeningRoleAvailable()
isCallScreeningRoleGranted()
requestCallScreeningRole()
getCallerIdSyncStatus()
triggerCallerIdSync()
clearCallerIdData()
```

The React Native JavaScript layer can then display status and configuration.

The actual incoming call processing remains native Android.

React Native's official documentation supports native modules for accessing native platform APIs not directly exposed by React Native.

---

# 54. Suggested Project Structure

Example:

```text
android/
└── app/
    └── src/
        └── main/
            ├── java/
            │   └── com/company/app/
            │       ├── telecom/
            │       │   ├── CustomerCallScreeningService.kt
            │       │   ├── CallerIdManager.kt
            │       │   └── CallScreeningRoleManager.kt
            │       │
            │       ├── callerid/
            │       │   ├── CallerDatabase.kt
            │       │   ├── CustomerDao.kt
            │       │   └── CustomerEntity.kt
            │       │
            │       └── rn/
            │           └── CallerIdNativeModule.kt
            │
            └── AndroidManifest.xml
```

React Native:

```text
src/
└── features/
    └── callerId/
        ├── screens/
        ├── services/
        ├── hooks/
        ├── types/
        └── components/
```

---

# 55. Separation of Responsibilities

## Backend

Responsible for:

- Customer master data
- Authentication
- Authorization
- Customer-to-phone mapping
- Sync API
- Incremental changes
- Device registration
- Revocation
- Audit

## React Native

Responsible for:

- Login
- Configuration
- Caller-ID settings
- Sync status
- User-facing UI
- Native module bridge

## Native Android

Responsible for:

- CallScreeningService
- Telecom integration
- Call Screening role
- Incoming call processing
- Local caller lookup
- Caller-ID UI integration

---

# 56. Security Boundaries

The incoming call service should not receive unrestricted access to the entire backend.

Instead:

```text
Backend
   ↓
Minimal caller-ID dataset
   ↓
Encrypted/protected local storage
   ↓
CallScreeningService
```

This follows the principle of least privilege.

---

# 57. Logging

Do not log full customer phone numbers in production logs.

Bad:

```text
Incoming caller:
+919871234567
Customer:
Rahul Sharma
```

Better:

```text
Caller-ID lookup completed
customerMatch=true
```

If debugging requires an identifier, use a controlled redacted value.

Example:

```text
+91******4567
```

---

# 58. Analytics

Do not perform analytics network requests during the critical call-screening callback.

Bad:

```text
Incoming Call
 ↓
Lookup
 ↓
Analytics API
 ↓
Backend API
 ↓
Respond to Android
```

Instead:

```text
Incoming Call
 ↓
Local Lookup
 ↓
Respond immediately
 ↓
Optional asynchronous analytics later
```

Even then, analytics should not contain unnecessary personal information.

---

# 59. Failure Handling

## Case 1 — Local DB unavailable

```text
Incoming call
↓
No local database
↓
Allow call
↓
Show normal/unknown caller experience
```

Do not block the customer call.

## Case 2 — Number not found

```text
Allow call
```

## Case 3 — Sync failed

```text
Continue using last known data
```

## Case 4 — Role revoked

```text
Caller-ID feature disabled
↓
Notify Sales Team
↓
Provide instructions to re-enable
```

## Case 5 — Device unsupported

```text
Show unsupported/limited status
```

---

# 60. User Onboarding

When a Sales Team member logs in for the first time:

```text
1. Login
2. Verify Sales role
3. Register device
4. Check Android compatibility
5. Request Call Screening role
6. User grants role
7. Initialize local database
8. Initial customer synchronization
9. Verify local database
10. Run test caller-ID check
```

The user should see a clear status:

```text
Caller ID Setup

✓ Account authenticated
✓ Device registered
✓ Call Screening enabled
✓ Customer database synchronized

Status: READY
```

---

# 61. Admin Controls

Company Admin should be able to see:

```text
Sales User
Device
Caller ID Status
Last Sync
App Version
Android Version
```

Example:

```text
Sales Team Devices

Aman
Samsung A55
Caller ID: Enabled
Last Sync: 5 min ago

Rahul
Pixel 8
Caller ID: Enabled
Last Sync: 12 min ago

Priya
OnePlus
Caller ID: Disabled
Reason: Role revoked
```

This makes the system operationally manageable.

---

# 62. Testing Strategy

Testing must cover more than normal application testing.

## Functional tests

- Existing customer calls
- Unknown customer calls
- Customer number updated
- Customer deleted
- Duplicate numbers
- Multiple numbers
- International-format number
- Number normalization

## Application state

- App open
- App background
- App removed from recents
- App not opened
- Device locked
- Device unlocked
- Device rebooted

## Network

- Wi-Fi connected
- Mobile data connected
- No Internet
- Slow Internet
- Backend unavailable
- Sync timeout

## Android

- Android 11
- Android 12
- Android 13
- Android 14
- Android 15
- Current supported Android version

## OEM

Test the actual Sales Team device models.

---

# 63. Performance Tests

Measure:

```text
onScreenCall()
        ↓
phone extraction
        ↓
normalization
        ↓
local DB lookup
        ↓
respondToCall()
```

The implementation must consistently remain comfortably below Android's 5-second requirement.

The target should be much lower than 5 seconds.

For example:

```text
Target:
< 100 ms local lookup path

Maximum platform requirement:
5 seconds
```

The 100 ms figure is an engineering target, not an Android requirement.

---

# 64. Load Testing

The local caller-ID database must be tested with realistic company scale.

Examples:

```text
10,000 customers
100,000 customers
1,000,000 customers
```

The lookup performance should remain predictable because the phone number is indexed.

Backend synchronization should use incremental changes rather than full database downloads.

---

# 65. Scalability

The architecture scales because:

```text
Incoming call
    ↓
Local lookup
```

does not create a backend request per phone call.

If:

```text
10,000 Sales devices
```

receive:

```text
100 calls/day/device
```

that could create:

```text
1,000,000 calls/day
```

A live database lookup for every call would unnecessarily increase backend traffic.

With local caller-ID indexes:

```text
1,000,000 calls
       ↓
Local device processing
```

Backend load is primarily generated by synchronization rather than every call.

This is significantly more scalable.

---

# 66. Reliability

The architecture is designed so that the most time-sensitive component has the fewest dependencies.

```text
Incoming Call
      ↓
Android Telecom
      ↓
Native Service
      ↓
Local DB
```

No dependency on:

```text
Internet
Backend
Remote database
React Native JS
Push notification
```

during the critical lookup path.

---

# 67. Maintainability

The system should be modular:

```text
CallScreeningService
        ↓
CallerIdManager
        ↓
CallerRepository
        ↓
Local Database
```

Synchronization:

```text
Backend API
        ↓
SyncManager
        ↓
Local Database
```

React Native:

```text
Native Module
        ↓
Caller-ID Settings UI
```

This makes each layer independently testable.

---

# 68. Why This Is Better Than a Background Service

A common approach would be:

```text
Foreground/background service
        ↓
Listen for calls
        ↓
React Native
```

This is not the preferred architecture.

Android provides a dedicated Telecom API for this exact class of functionality.

The official `CallScreeningService` is bound by Telecom when calls are received.

Therefore:

```text
Dedicated Android Telecom API
```

is preferable to:

```text
Custom permanent background process
```

---

# 69. Why This Is Better Than Accessibility

Accessibility-based solutions would be:

- More fragile
- More difficult to maintain
- More dependent on OEM UI
- Potentially problematic for policy compliance
- Unnecessary when an official Telecom API exists

The recommended implementation uses:

```text
CallScreeningService
```

instead.

---

# 70. Why This Is Better Than Live API Lookup

Live API lookup:

```text
Call
 ↓
Internet
 ↓
API
 ↓
Database
 ↓
Customer
```

has too many dependencies for a time-sensitive operation.

Local index:

```text
Call
 ↓
Local phone normalization
 ↓
Indexed local lookup
 ↓
Customer
```

is faster and more reliable.

---

# 71. Data Sync Is the Key Supporting System

The caller-ID feature itself is relatively small.

The larger engineering problem is maintaining a correct local copy of:

```text
Phone Number → Customer Identity
```

Therefore the project should treat synchronization as a first-class subsystem.

Requirements:

- Initial sync
- Incremental sync
- Delete propagation
- Update propagation
- Retry
- Versioning
- Authentication
- Device registration
- Conflict handling
- Last-sync status
- Recovery after app reinstall

---

# 72. Reinstall Behavior

If the application is uninstalled:

```text
Local caller database disappears
```

After reinstall:

```text
Login
 ↓
Device registration
 ↓
Role authorization
 ↓
Full initial synchronization
 ↓
Caller-ID ready
```

---

# 73. Logout Behavior

When the Sales Team member logs out:

```text
Stop authorized sync
Clear authentication tokens
Clear local customer caller-ID data
Remove device association if required
```

Whether local data should be retained across account switching must be explicitly defined.

For shared company devices, clearing customer data on logout is recommended.

---

# 74. Multi-Tenant Consideration

If the company may support multiple organizations:

```text
organization_id
```

must be part of authorization and synchronization.

Never allow:

```text
Company A customer data
```

to appear on:

```text
Company B device
```

---

# 75. Auditability

Backend should maintain audit events such as:

```text
Device registered
Caller-ID sync started
Caller-ID sync completed
Device revoked
User disabled
```

However, avoid unnecessarily storing every incoming call's full personal information unless there is a legitimate business requirement.

---

# 76. Recommended MVP

For the first production version:

### Must Have

- Android-only
- CallScreeningService
- ROLE_CALL_SCREENING
- Customer phone normalization
- Local indexed caller-ID database
- Initial sync
- Incremental sync
- Customer name display
- Unknown caller handling
- Offline lookup
- Device registration
- Role/status screen
- Security
- Privacy policy
- OEM/device testing

### Should Have

- Admin device dashboard
- Sync health monitoring
- Revocation
- Audit logs
- Automatic retry
- Data encryption

### Future

- Customer profile deep-link after call
- Call history integration, if legally/policy permitted
- Sales CRM integration
- Call notes
- Follow-up workflow
- Customer booking history
- Lead classification

---

# 77. Future Post-Call Workflow

Once the basic caller identification is stable, the company can extend the workflow:

```text
Incoming Call
      ↓
Customer identified
      ↓
Sales representative answers
      ↓
Customer conversation
      ↓
Call ends
      ↓
Optional:
Open customer profile
      ↓
Create follow-up
      ↓
Add call notes
```

This should be implemented as a separate feature from the critical incoming-call identification path.

---

# 78. Recommended Technical Stack

## Mobile

```text
React Native
+
Native Android Kotlin
+
Android Telecom
+
CallScreeningService
```

## Local data

```text
Android local persistent database
+
indexed normalized phone number
```

## Backend

```text
Existing backend
+
caller-ID synchronization endpoint
+
device authorization
+
incremental sync
```

## Security

```text
HTTPS/TLS
+
authentication
+
authorization
+
secure local storage
+
data minimization
```

---

# 79. High-Level Architecture Diagram

```text
                         COMPANY BACKEND
                    ┌──────────────────────┐
                    │ Customer Database    │
                    │                      │
                    │ Customer ID          │
                    │ Name                 │
                    │ Phone                │
                    │ Status               │
                    └──────────┬───────────┘
                               │
                         Secure Sync API
                               │
                               ▼
                 ┌───────────────────────────┐
                 │ SALES ANDROID DEVICE      │
                 │                           │
                 │ React Native Application  │
                 │                           │
                 │ Caller-ID Settings        │
                 │ Sync Manager              │
                 └────────────┬──────────────┘
                              │
                       Native Android
                              │
                              ▼
                 ┌───────────────────────────┐
                 │ CallScreeningService      │
                 │                           │
                 │ Android Telecom           │
                 └────────────┬──────────────┘
                              │
                       Caller Number
                              │
                              ▼
                 ┌───────────────────────────┐
                 │ Local Caller-ID Database  │
                 │                           │
                 │ Phone → Customer          │
                 └────────────┬──────────────┘
                              │
                              ▼
                       Customer Identity
                              │
                              ▼
                  Caller Identification UI
                              │
                              ▼
                       SALES REPRESENTATIVE
```

---

# 80. Complete Runtime Workflow

```text
CUSTOMER
   │
   │ Normal cellular call
   ▼
SALES TEAM PHONE
   │
   ▼
ANDROID TELECOM
   │
   ▼
CALL SCREENING ROLE
   │
   ▼
CallScreeningService
   │
   ├── Get caller handle
   │
   ├── Normalize phone number
   │
   ├── Local DB lookup
   │
   ├── Customer found?
   │       │
   │       ├── YES → Customer identity
   │       │
   │       └── NO  → Unknown caller
   │
   ├── Respond to Android
   │
   └── Caller-ID UI
            │
            ▼
      SALES REPRESENTATIVE
            │
            ▼
       Answers the call
```

---

# 81. Implementation Phases

## Phase 1 — Proof of Concept

Goal:

```text
Android phone
    ↓
Incoming call
    ↓
CallScreeningService
    ↓
Get number
    ↓
Local hardcoded/test customer
    ↓
Show caller ID
```

No backend initially.

---

## Phase 2 — Local Database

Implement:

- Local DB
- Phone normalization
- Indexed lookup
- Customer model
- Unknown caller handling

---

## Phase 3 — Backend Sync

Implement:

- Authentication
- Device registration
- Initial sync
- Incremental sync
- Delete/update handling
- Retry

---

## Phase 4 — React Native Integration

Implement:

- Caller-ID settings
- Role request
- Status
- Sync status
- Manual sync
- Troubleshooting

---

## Phase 5 — Production Security

Implement:

- Secure tokens
- Data encryption
- Authorization
- Device revocation
- Privacy controls
- Audit events
- Logging policy

---

## Phase 6 — Device Certification

Test actual Sales Team devices.

---

## Phase 7 — Production Rollout

Recommended rollout:

```text
1–5 pilot devices
      ↓
10–20 devices
      ↓
One Sales Team
      ↓
Full deployment
```

Monitor:

- Caller-ID match rate
- Sync failures
- Role failures
- Device compatibility
- Crash rate
- User feedback

---

# 82. Acceptance Criteria

The feature should be considered successful when:

### AC-01

A registered Sales Team Android device has the Call Screening role enabled.

### AC-02

A known customer calls the device.

### AC-03

The system receives the caller number through Android's Telecom mechanism.

### AC-04

The local database finds the normalized phone number.

### AC-05

The customer's name is presented through the supported caller-ID experience.

### AC-06

The normal call continues without being blocked.

### AC-07

The React Native application does not need to be open.

### AC-08

The feature continues to identify previously synchronized customers without Internet access.

### AC-09

A new customer appears after successful synchronization.

### AC-10

Deleted/deactivated customer information is removed from the device after synchronization.

### AC-11

The feature handles unknown callers gracefully.

### AC-12

The system responds to Android within the required call-screening deadline.

Android requires the incoming-call response within 5 seconds.

---

# 83. Risks

## Risk 1 — OEM differences

Different Android manufacturers may display caller-ID experiences differently.

**Mitigation:** Device certification matrix.

## Risk 2 — Role not granted

Without the required Android role, Telecom will not use the app as the selected screening service.

**Mitigation:** Setup wizard + status monitoring.

## Risk 3 — Stale local data

A customer may have been created/updated after the last synchronization.

**Mitigation:** Frequent incremental synchronization.

## Risk 4 — Phone-number formatting

Different formats can cause false negatives.

**Mitigation:** E.164/canonical normalization.

## Risk 5 — Spoofed caller ID

Caller ID is not proof of identity.

**Mitigation:** Treat caller identification as an assistance mechanism, not authentication.

## Risk 6 — Play policy

Unnecessary Call Log permissions can create Play policy problems.

**Mitigation:** Avoid unnecessary sensitive permissions and review current Play policies before release.

---

# 84. What We Should NOT Build

The following approaches are not recommended:

```text
❌ React Native polling for calls
❌ Permanent JavaScript background loop
❌ AccessibilityService workaround
❌ Screen scraping of the Phone application
❌ Server request for every incoming call
❌ Full customer database stored unnecessarily on device
❌ Unnecessary READ_CALL_LOG permission
❌ UI automation of OEM Phone application
❌ Assuming every Android manufacturer's Phone UI behaves identically
```

Recommended:

```text
✅ Android Telecom
✅ CallScreeningService
✅ ROLE_CALL_SCREENING
✅ Local indexed caller database
✅ Secure incremental synchronization
✅ Native Kotlin integration
✅ React Native for application/business UI
```

---

# 85. Final Architecture Decision

### Recommended Decision

**Proceed with an Android-native `CallScreeningService` integrated into the existing React Native application.**

Use:

```text
React Native
    +
Kotlin Android native module
    +
CallScreeningService
    +
ROLE_CALL_SCREENING
    +
Local indexed caller-ID database
    +
Secure incremental backend synchronization
```

This architecture is:

### Reliable

The critical incoming-call lookup does not depend on the Internet or React Native JavaScript runtime.

### Maintainable

Android Telecom functionality is isolated in native Android code.

### Scalable

Incoming calls are resolved locally instead of creating a backend request for every call.

### Secure

Only the minimum required customer information can be synchronized to managed devices.

### Policy-aware

The architecture uses Android's official Telecom caller-ID mechanism rather than UI hacks or Accessibility workarounds.

---

# 86. Most Important Limitation to Communicate to Management

The manager should approve the following wording:

> **The system will use Android's official CallScreeningService and Call Screening role to identify known customers on incoming cellular calls and present customer-identification information through the supported caller-ID experience. The React Native application does not need to remain open for the Telecom service to operate. The exact visual appearance of the caller-identification UI may vary by Android version, device manufacturer, and Phone application.**

This is technically safer than promising:

> "We will modify the native incoming-call screen on every Android phone."

The first statement is supported by Android's official APIs and documentation.

---

# 87. Official Sources

## Android Developers — CallScreeningService

Official API reference covering:

- Call screening
- Caller identification
- Telecom binding
- `onScreenCall`
- 5-second response requirement
- Local database lookup
- Caller-ID UI

[Android CallScreeningService API Reference](https://developer.android.com/reference/android/telecom/CallScreeningService?utm_source=chatgpt.com)

## Android Developers — RoleManager

Official documentation for:

- `ROLE_CALL_SCREENING`
- User role consent
- Role management

[Android RoleManager API Reference](https://developer.android.com/reference/kotlin/android/app/role/RoleManager?utm_source=chatgpt.com)

## AndroidX — RoleManagerCompat

Official AndroidX documentation for the Call Screening and Caller ID role.

[AndroidX RoleManagerCompat](https://developer.android.com/reference/kotlin/androidx/core/role/RoleManagerCompat?utm_source=chatgpt.com)

## Android Developers — Android 11 CallScreening Updates

Official documentation covering caller-ID related updates and contact handling.

[Android 11 Features and APIs](https://developer.android.com/about/versions/11/features?utm_source=chatgpt.com)

## Android Developers — Manifest Permissions

Official documentation for `BIND_SCREENING_SERVICE` and Android permissions.

[Android Manifest Permissions](https://developer.android.com/reference/android/Manifest.permission?utm_source=chatgpt.com)

## React Native — Native Platform

Official React Native documentation for integrating native platform functionality.

[React Native Native Platform](https://reactnative.dev/docs/native-platform?utm_source=chatgpt.com)

## React Native — Native Modules

Official documentation for Android native modules and React Native integration.

[React Native Native Modules](https://reactnative.dev/docs/turbo-native-modules-introduction?utm_source=chatgpt.com)

## Google Play — SMS and Call Log Permissions

Official Google Play policy documentation for sensitive SMS and Call Log permissions.

[Google Play SMS and Call Log Permissions Policy](https://support.google.com/googleplay/android-developer/answer/10208820?utm_source=chatgpt.com)

## Google Play — Permission Declarations

Official Google Play documentation for permission declarations during app review.

[Google Play Permission Declaration Requirements](https://support.google.com/googleplay/android-developer/answer/9214102?utm_source=chatgpt.com)

---

# 88. Management Decision

### Recommendation

**Technically feasible on Android using official Android Telecom APIs.**

### Primary API

```text
CallScreeningService
```

### Required Android Role

```text
ROLE_CALL_SCREENING
```

### React Native Requirement

```text
Existing React Native app
+
Native Android Kotlin integration
```

### Backend Requirement

```text
Secure caller-ID synchronization API
```

### Local Requirement

```text
Indexed local customer phone → identity database
```

### Critical Performance Requirement

```text
respondToCall()
within Android's 5-second requirement
```

### Key Product Limitation

```text
Exact native Phone UI appearance is device/OEM dependent.
```

### Recommended Next Step

Build a **small Android Proof of Concept first**, before implementing the complete synchronization architecture.

The PoC should prove these exact conditions on the company's real Sales Team Android device:

```text
Known customer calls
       ↓
App is not open
       ↓
Android CallScreeningService receives call
       ↓
Customer number is matched locally
       ↓
Customer identity is presented
       ↓
Normal call rings
       ↓
Sales representative answers
```

Only after this PoC passes on the target devices should the team proceed with the full production synchronization and security architecture.