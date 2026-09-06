# React Native → iOS App Store Migration Guide

> **Enterprise-grade reference** for migrating a bare React Native CLI project (developed on Windows for Android only) to iOS and publishing to the Apple App Store.
>
> All links in this document point to **official** primary sources (vendor, project maintainer, or standards body). No third-party mirrors, no guesses.

---

## Table of Contents

1. [Problem Statement & Context](#1-problem-statement--context)
2. [Current Situation Assessment](#2-current-situation-assessment)
3. [Prerequisites Checklist](#3-prerequisites-checklist)
4. [Phase 1 — Mac Environment Setup](#4-phase-1--mac-environment-setup)
5. [Phase 2 — Project Transfer & Initial Build](#5-phase-2--project-transfer--initial-build)
6. [Phase 3 — Dependencies Audit](#6-phase-3--dependencies-audit)
7. [Phase 4 — iOS-Specific Configuration](#7-phase-4--ios-specific-configuration)
8. [Phase 5 — Assets & Icons Setup](#8-phase-5--assets--icons-setup)
9. [Phase 6 — Apple Developer Account & Code Signing](#9-phase-6--apple-developer-account--code-signing)
10. [Phase 7 — Testing Strategy](#10-phase-7--testing-strategy)
11. [Phase 8 — App Store Submission](#11-phase-8--app-store-submission)
12. [Common Issues & Troubleshooting](#12-common-issues--troubleshooting)
13. [Realistic Timeline & Effort](#13-realistic-timeline--effort)
14. [Post-Launch Considerations](#14-post-launch-considerations)
15. [Appendix — Commands & Official Resources](#15-appendix--commands--official-resources)

---

## 1. Problem Statement & Context

### 1.1 The Situation

A mobile application has been built using **React Native CLI (bare workflow, not Expo)**. The entire lifecycle — scaffolding, feature work, third-party package selection, testing, and release build — was carried out on **Windows** targeting **Android only**. The Android build has been signed and shipped through the [Google Play Console](https://play.google.com/console/).

Management now requires the same application to be published on the **Apple App Store**.

### 1.2 Why This Is Not a Copy-Paste Task

- Apple requires **macOS + Xcode** for iOS builds — a hard technical constraint. See Apple's [Xcode requirements](https://developer.apple.com/xcode/).
- The `ios/` folder has never been opened, configured, or tested with real dependencies.
- Third-party npm packages often require iOS-specific native setup (CocoaPods, `Info.plist` entries, usage descriptions) that has never been performed.
- Apple's [App Store Review Guidelines](https://developer.apple.com/app-store/review/guidelines/) are stricter and more prescriptive than Google Play policies.
- iOS-specific UI behaviors (safe areas, notches, Dynamic Island, keyboard handling, fonts, shadows) frequently differ from Android.
- A separate [Apple Developer Program](https://developer.apple.com/programs/) membership, signing certificates, and provisioning profiles are mandatory.

---

## 2. Current Situation Assessment

| Aspect | Current State | Impact on Migration |
|---|---|---|
| Framework | React Native CLI (bare workflow) | Full control over `ios/`; no Expo abstraction layer |
| Development OS | Windows | Cannot build iOS locally; Mac is mandatory |
| Target Platform | Android only | iOS side has never been built, tested, or configured |
| Native Android Code | None written by developer | No custom Java/Kotlin bridges to port to Swift/Objective-C |
| Distribution Status | Published on Google Play Console | Android version is live; iOS is greenfield |
| New Hardware | macOS purchased | iOS development environment is technically possible |

> ✅ **Positive indicator** — the absence of custom native Android code means application logic is entirely JavaScript/TypeScript, which is inherently portable. The migration focuses on **configuration, packaging, and platform polish**, not rewriting business logic.

---

## 3. Prerequisites Checklist

### 3.1 Hardware

- Mac running macOS (Ventura 13 or newer recommended). See [Apple system requirements](https://support.apple.com/en-us/HT201260).
- **≥ 50 GB** free disk space (Xcode alone consumes 15–20 GB plus simulators and DerivedData).
- **≥ 16 GB** RAM recommended (8 GB works but slower).
- A physical iPhone for real-device testing (**strongly recommended**, not optional for production apps).
- Lightning or USB-C cable.

### 3.2 Accounts & Access

- Apple ID (personal or company-managed) — create at [appleid.apple.com](https://appleid.apple.com/).
- [Apple Developer Program](https://developer.apple.com/programs/) enrollment (USD **99/year** individual/organization; USD **299/year** enterprise).
- Access to project source code (Git repository preferred).
- Access to all app assets: original icon files, splash screens, marketing materials.
- Backend API access if the app depends on internal services.

### 3.3 Information to Gather

- Complete list of npm dependencies (`package.json`).
- Device features the app uses: camera, microphone, location, push notifications, Bluetooth, Face ID/Touch ID, photo library, contacts, calendar, etc.
- **Privacy policy URL** — mandatory for App Store submission. See [App Store Connect Help — Privacy](https://developer.apple.com/app-store/app-privacy-details/).
- App description, keywords, support URL for App Store Connect metadata.
- Bundle identifier (e.g., `com.yourcompany.yourapp`).

---

## 4. Phase 1 — Mac Environment Setup

Complete every step **in order**. Skipping ahead typically causes cryptic build errors later.

### 4.1 Install Xcode

Install from the [Mac App Store — Xcode](https://apps.apple.com/us/app/xcode/id497799835). Download is ~15 GB and can take 1–3 hours.

Launch it once, accept the license, allow additional components to install, then verify:

```bash
xcodebuild -version
```

Official documentation: [Apple Developer — Xcode](https://developer.apple.com/xcode/).

### 4.2 Install Xcode Command Line Tools

```bash
xcode-select --install
```

Required by many build scripts and package managers.

### 4.3 Install Homebrew

Homebrew is the de-facto macOS package manager. Official site: **[brew.sh](https://brew.sh/)**.

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

Follow on-screen instructions to add Homebrew to your `PATH` (usually one or two lines in `~/.zshrc`).

### 4.4 Install Node.js (via nvm)

Use the same major Node version you used on Windows to avoid dependency mismatches.

- Node.js official: **[nodejs.org](https://nodejs.org/)**
- nvm official: **[github.com/nvm-sh/nvm](https://github.com/nvm-sh/nvm)**

```bash
brew install nvm
mkdir ~/.nvm
# Add the nvm init lines from `brew info nvm` to ~/.zshrc
nvm install 20   # or whichever LTS your project uses
nvm use 20
```

### 4.5 Install Watchman

Used by React Native (and Metro) to watch file changes. Official site: **[facebook.github.io/watchman](https://facebook.github.io/watchman/)**.

```bash
brew install watchman
```

### 4.6 Install Ruby via rbenv (required for CocoaPods)

Do **not** use the system Ruby that ships with macOS — it requires `sudo` for gems and causes permission errors.

- rbenv: **[github.com/rbenv/rbenv](https://github.com/rbenv/rbenv)**
- Ruby: **[ruby-lang.org](https://www.ruby-lang.org/en/)**

```bash
brew install rbenv ruby-build
rbenv install 3.2.2
rbenv global 3.2.2
# Add `eval "$(rbenv init - zsh)"` to ~/.zshrc
```

### 4.7 Install CocoaPods

CocoaPods is the Ruby-based dependency manager for iOS native libraries. Official site: **[cocoapods.org](https://cocoapods.org/)**.

```bash
gem install cocoapods
pod --version
```

Every time a React Native package includes iOS native code, CocoaPods installs it into the `ios/` folder.

### 4.8 Verify React Native CLI

The React Native community recommends `npx` (no global install needed). Official docs: **[reactnative.dev](https://reactnative.dev/)** — see [Environment Setup](https://reactnative.dev/docs/set-up-your-environment).

```bash
node --version
npm --version
npx react-native --version
```

---

## 5. Phase 2 — Project Transfer & Initial Build

### 5.1 Transfer the Project to Mac

Preferred: push to a Git remote ([GitHub](https://github.com/), [GitLab](https://about.gitlab.com/), or [Bitbucket](https://bitbucket.org/)) and clone on the Mac.

```bash
cd ~/Projects
git clone <repo-url>
```

If Git is not an option, transfer via external drive or cloud storage but **exclude `node_modules` and `android/build`** — they are OS-specific and must be regenerated.

### 5.2 Install JavaScript Dependencies

```bash
cd YourProject
rm -rf node_modules package-lock.json   # remove Windows leftovers
npm install
```

Some packages ship optional native binaries that differ between Windows and macOS; a fresh install ensures the correct ones are downloaded.

### 5.3 Install iOS Native Dependencies via CocoaPods

```bash
cd ios
pod install
cd ..
```

This reads the auto-generated `Podfile`, downloads iOS dependencies for every RN package that requires them, and generates the `.xcworkspace` file.

> ⚠️ **From this point onward, always open the `.xcworkspace` file in Xcode — never the `.xcodeproj` file.**

> **Common failure at `pod install`** — errors about Ruby versions, Bundler, or missing modular headers are almost always a Ruby/CocoaPods version mismatch. Confirm `rbenv` is active in your shell and the correct Ruby version is set.

### 5.4 First Build Attempt on iOS Simulator

```bash
npx react-native run-ios
```

This starts Metro bundler, builds the iOS project, launches the default Simulator, and installs the app. **Expect errors on the first attempt** — read each one carefully and resolve them one at a time.

---

## 6. Phase 3 — Dependencies Audit

This is often the most time-consuming phase. Every third-party package must be reviewed for iOS compatibility.

### 6.1 Audit Methodology

1. Open `package.json` and list every entry under `"dependencies"` (skip `devDependencies` for now).
2. For each package, open its official npm page (**[npmjs.com](https://www.npmjs.com/)**) or its GitHub README.
3. Look for an **iOS** subsection in "Installation" or "Getting Started". Follow it exactly.
4. Verify the package supports your minimum iOS deployment target (typically iOS 15+ in 2026).
5. If a package is unmaintained or Android-only, find a cross-platform alternative.

### 6.2 Packages That Typically Require Extra iOS Setup

| Category | Common Package | Official Source | iOS Setup Required |
|---|---|---|---|
| Permissions | `react-native-permissions` | [github.com/zoontek/react-native-permissions](https://github.com/zoontek/react-native-permissions) | Add each permission pod to `Podfile`; add usage descriptions to `Info.plist` |
| Push Notifications | `@react-native-firebase/messaging` | [rnfirebase.io](https://rnfirebase.io/) | Enable Push Notifications capability; upload APNs key to Firebase |
| Push Notifications | `notifee` | [notifee.app](https://notifee.app/) | Configure notification categories, sounds, and background modes |
| Firebase | `@react-native-firebase/app` + modules | [rnfirebase.io](https://rnfirebase.io/) | Download `GoogleService-Info.plist` from [Firebase Console](https://console.firebase.google.com/); add to Xcode; modify `AppDelegate` |
| Camera / Media | `react-native-vision-camera` | [react-native-vision-camera.com](https://react-native-vision-camera.com/) · [github.com/mrousavy/react-native-vision-camera](https://github.com/mrousavy/react-native-vision-camera) | Add `NSCameraUsageDescription`, `NSMicrophoneUsageDescription` |
| Media picker | `react-native-image-picker` | [github.com/react-native-image-picker/react-native-image-picker](https://github.com/react-native-image-picker/react-native-image-picker) | Add `NSPhotoLibraryUsageDescription`, `NSCameraUsageDescription` |
| Location | `react-native-geolocation-service` | [github.com/Agontuk/react-native-geolocation-service](https://github.com/Agontuk/react-native-geolocation-service) | Add `NSLocationWhenInUseUsageDescription`, optionally `NSLocationAlwaysAndWhenInUseUsageDescription` |
| Location | `@react-native-community/geolocation` | [github.com/michalchudziak/react-native-geolocation](https://github.com/michalchudziak/react-native-geolocation) | Same as above |
| Maps | `react-native-maps` | [github.com/react-native-maps/react-native-maps](https://github.com/react-native-maps/react-native-maps) | Configure [Apple Maps](https://developer.apple.com/maps/) (free) or [Google Maps SDK for iOS](https://developers.google.com/maps/documentation/ios-sdk) (requires API key + Podfile changes) |
| Deep Linking | `@react-navigation/native` | [reactnavigation.org](https://reactnavigation.org/) | Configure Associated Domains capability; add URL schemes to `Info.plist` |
| In-App Purchase | `react-native-iap` | [github.com/hyochan/react-native-iap](https://github.com/hyochan/react-native-iap) | Enable [In-App Purchase capability](https://developer.apple.com/in-app-purchase/); configure products in [App Store Connect](https://appstoreconnect.apple.com/) |
| Bluetooth | `react-native-ble-plx` | [github.com/dotintent/react-native-ble-plx](https://github.com/dotintent/react-native-ble-plx) | Add `NSBluetoothAlwaysUsageDescription` and `NSBluetoothPeripheralUsageDescription` |
| Contacts | `react-native-contacts` | [github.com/morenoh149/react-native-contacts](https://github.com/morenoh149/react-native-contacts) | Add `NSContactsUsageDescription` |
| Calendar | `react-native-calendar-events` | [github.com/wmcmahan/react-native-calendar-events](https://github.com/wmcmahan/react-native-calendar-events) | Add `NSCalendarsUsageDescription` |
| Biometrics | `react-native-biometrics` | [github.com/SelfLender/react-native-biometrics](https://github.com/SelfLender/react-native-biometrics) | Add `NSFaceIDUsageDescription` |
| Keychain | `react-native-keychain` | [github.com/oblador/react-native-keychain](https://github.com/oblador/react-native-keychain) | Configure Keychain Sharing entitlement |

### 6.3 Handling Packages Without iOS Support

Three options:

1. **Find a cross-platform replacement** (best option).
2. **Guard with `Platform.OS === "android"`** and gracefully disable on iOS with a fallback UI. See [Platform-specific code](https://reactnative.dev/docs/platform-specific-code).
3. **Write a custom native iOS module** in Swift/Objective-C. See [Native modules — iOS](https://reactnative.dev/docs/legacy/native-modules-ios).

### 6.4 After Every Dependency Change

```bash
cd ios && pod install && cd ..
```

---

## 7. Phase 4 — iOS-Specific Configuration

### 7.1 Configure the Bundle Identifier

Must follow reverse-domain notation (e.g., `com.yourcompany.yourapp`) and be registered in your Apple Developer account. In Xcode: open the `.xcworkspace`, select the project, choose the target, and set **Bundle Identifier** under the **General** tab.

Reference: [Apple — Configuring your target's build settings](https://developer.apple.com/documentation/xcode/build-settings-reference).

### 7.2 Set Display Name, Version, and Build Number

- **Display Name** — shown under the app icon (keep to 12–15 characters).
- **Version** — user-facing (e.g., `1.0.0`). Increment for every App Store release.
- **Build Number** — internal counter. **Must be incremented for every upload** to App Store Connect, even for the same version.

Reference: [Apple — CFBundleShortVersionString](https://developer.apple.com/documentation/bundleresources/information_property_list/cfbundleshortversionstring) and [CFBundleVersion](https://developer.apple.com/documentation/bundleresources/information_property_list/cfbundleversion).

### 7.3 Configure `Info.plist` Permission Descriptions

Apple requires a human-readable explanation for every permission. Without a usage description, the app **crashes on the permission request and is rejected by review**.

Full list of keys: [Apple — Protected Resources / Information Property List](https://developer.apple.com/documentation/bundleresources/information_property_list/protected_resources).

| `Info.plist` Key | Purpose | Example Description |
|---|---|---|
| `NSCameraUsageDescription` | Camera access | This app uses the camera to scan QR codes and capture profile photos. |
| `NSPhotoLibraryUsageDescription` | Read photos | This app accesses your photo library to let you upload profile pictures. |
| `NSPhotoLibraryAddUsageDescription` | Save to photos | This app saves generated images to your photo library. |
| `NSMicrophoneUsageDescription` | Microphone access | This app uses the microphone for voice messages. |
| `NSLocationWhenInUseUsageDescription` | Location while using app | This app uses your location to show nearby items. |
| `NSLocationAlwaysAndWhenInUseUsageDescription` | Background location | This app tracks your route during activities. |
| `NSContactsUsageDescription` | Read contacts | This app accesses your contacts to help you invite friends. |
| `NSFaceIDUsageDescription` | Face ID authentication | This app uses Face ID to secure your account. |
| `NSBluetoothAlwaysUsageDescription` | Bluetooth access | This app uses Bluetooth to connect with nearby devices. |
| `NSCalendarsUsageDescription` | Calendar access | This app adds events to your calendar. |
| `NSUserTrackingUsageDescription` | App Tracking Transparency | This identifier will be used to deliver personalized ads. |

### 7.4 Set Minimum iOS Deployment Target

In Xcode → target settings → **Deployment Info** → **iOS Deployment Target**. **iOS 15+** is a reasonable minimum in 2026 (covers well over 95% of active devices). See [Apple — Supported configurations](https://developer.apple.com/support/app-store/) for current adoption data.

### 7.5 Configure App Transport Security (ATS)

Apple enforces HTTPS by default. If your app must talk to an HTTP endpoint, add an exception in `Info.plist` — but review may reject it without a valid reason.

Reference: [Apple — NSAppTransportSecurity](https://developer.apple.com/documentation/bundleresources/information_property_list/nsapptransportsecurity).

**Ideal**: ensure all backend APIs support HTTPS with a valid certificate.

### 7.6 Configure URL Schemes and Deep Links

Replicate Android deep-linking in `Info.plist` under `CFBundleURLTypes`. For **universal links** (recommended over custom URL schemes), configure the **Associated Domains** capability in Xcode and host an `apple-app-site-association` file on your web domain.

Reference: [Apple — Supporting associated domains](https://developer.apple.com/documentation/xcode/supporting-associated-domains).

### 7.7 Configure Background Modes (If Applicable)

For remote notifications, background audio, background location updates, etc., enable the corresponding **Background Modes** under **Signing & Capabilities** in Xcode.

Reference: [Apple — UIBackgroundModes](https://developer.apple.com/documentation/bundleresources/information_property_list/uibackgroundmodes).

---

## 8. Phase 5 — Assets & Icons Setup

### 8.1 App Icon Requirements

iOS 14+ accepts a **single 1024×1024 PNG** — Xcode generates the smaller sizes automatically. The icon must have:

- **No transparency**
- **No rounded corners** (iOS rounds them automatically)
- **PNG** format, **sRGB** color space

Reference: [Apple — App icons (Human Interface Guidelines)](https://developer.apple.com/design/human-interface-guidelines/app-icons).

Workflow:

1. Take your existing 1024×1024 master icon.
2. Generate the full `AppIcon.appiconset` (any Mac tool that follows Apple's spec works — the primary source of truth is Apple's own [Xcode asset catalog documentation](https://developer.apple.com/documentation/xcode/managing-assets-with-asset-catalogs)).
3. In Xcode, open `Images.xcassets`, delete the placeholder `AppIcon`, drag the generated set into place.

### 8.2 Launch Screen (Splash Screen)

iOS uses a **Launch Screen storyboard** — a static file displayed briefly while the app initializes. It **cannot contain dynamic content, JavaScript images, or animations**.

Reference: [Apple — Launch screen (HIG)](https://developer.apple.com/design/human-interface-guidelines/launching).

Options:

- Edit `LaunchScreen.storyboard` directly in Xcode's Interface Builder — recommended for simple splash screens.
- Use [react-native-bootsplash](https://github.com/zoontek/react-native-bootsplash) to generate a consistent splash across both platforms and control dismissal from JS.

### 8.3 Screenshots for App Store Listing

The App Store listing requires screenshots for specific iPhone display sizes. As of 2026, at minimum provide screenshots for the largest iPhone (6.9″ or 6.7″); the 5.5″ legacy size may still be requested for older listings.

Full specifications: [App Store Connect — Screenshot specifications](https://developer.apple.com/help/app-store-connect/reference/screenshot-specifications/).

Recommended: **3–10 screenshots per device size**, showcasing key features.

### 8.4 Marketing Assets

- **App Store icon**: 1024×1024 (uploaded separately in App Store Connect).
- **Optional app preview video**: 15–30 seconds, up to 3 per language. See [App preview specifications](https://developer.apple.com/help/app-store-connect/reference/app-preview-specifications/).
- **Promotional text**: up to 170 characters, editable without a new submission.

---

## 9. Phase 6 — Apple Developer Account & Code Signing

### 9.1 Enroll in the Apple Developer Program

1. Visit **[developer.apple.com/programs/enroll](https://developer.apple.com/programs/enroll/)** and sign in.
2. Choose **Individual** (personal Apple ID) or **Organization** (requires a [D-U-N-S number](https://developer.apple.com/support/D-U-N-S/) and legal entity verification).
3. Pay the annual fee (USD 99 individual/organization; USD 299 enterprise).
4. Wait for approval — individual is typically 24–48h; organization can take 1–2 weeks.

### 9.2 Register the App in App Store Connect

1. Log in to **[appstoreconnect.apple.com](https://appstoreconnect.apple.com/)**.
2. Go to **My Apps** → click **+** → **New App**.
3. Fill in: platform (iOS), name, primary language, **bundle identifier** (must match Xcode), and SKU (any unique internal identifier).

Reference: [App Store Connect Help](https://developer.apple.com/help/app-store-connect/).

### 9.3 Code Signing Setup

Xcode automates most of this now:

1. In Xcode → project → target → **Signing & Capabilities**.
2. Check **Automatically manage signing**.
3. Select your development **Team** from the dropdown.
4. Xcode creates certificates and provisioning profiles automatically.

Reference: [Apple — Code signing overview](https://developer.apple.com/support/code-signing/).

> **Two types of signing** — **Development** lets you run the app on your registered test devices. **Distribution** is required for TestFlight and the App Store. Xcode manages both automatically when the toggle is enabled; for CI/CD pipelines you will need to export and manage them manually.

---

## 10. Phase 7 — Testing Strategy

### 10.1 Simulator Testing

Fast and convenient for UI/logic verification, but has real limitations:

- No camera (falls back to a synthetic image)
- No real GPS (simulated location)
- No push notifications from real APNs (local notifications work)
- No Bluetooth
- No Face ID / Touch ID (can be simulated only)
- No real StoreKit purchases (requires the [StoreKit testing framework](https://developer.apple.com/documentation/xcode/setting-up-storekit-testing-in-xcode))

Reference: [Apple — Simulator documentation](https://developer.apple.com/documentation/xcode/running-your-app-in-simulator-or-on-a-device).

### 10.2 Physical Device Testing

Essential before App Store submission. Connect the iPhone via cable, trust the computer when prompted on the phone, and select the device in Xcode. First build to a new device registers it in your developer account automatically.

### 10.3 TestFlight Beta Distribution

**[TestFlight](https://developer.apple.com/testflight/)** is Apple's official beta platform, integrated with App Store Connect.

1. In Xcode: **Product → Archive**.
2. In the Organizer: **Distribute App → App Store Connect**.
3. Wait for processing (15–60 minutes).
4. In App Store Connect → **TestFlight** tab → add internal testers (up to 100, no review) or external testers (up to 10,000, lightweight review).

### 10.4 What to Specifically Test for iOS

- **Safe area insets** on notched devices and Dynamic Island — see [Apple HIG — Layout](https://developer.apple.com/design/human-interface-guidelines/layout).
- **Keyboard behavior** — inputs being covered by the keyboard is a very common iOS issue.
- **Font rendering** — some Android fonts aren't available on iOS; system fonts differ.
- **Shadow rendering** — RN shadow properties behave differently across platforms.
- **Scroll bounce** — iOS has elastic bounce that can affect UI logic.
- **StatusBar** styling — light vs dark content, translucent vs opaque.
- **Deep links** from Safari, Mail, other apps.
- **Permissions flow** — every first-time permission request.
- **Backgrounding / foregrounding** — state restoration.
- **Performance** on older devices (iPhone SE, iPhone 8 if targeted).

---

## 11. Phase 8 — App Store Submission

### 11.1 Prepare App Store Connect Metadata

Complete every required field before submitting the build. Reference: [App Store Connect — Managing metadata](https://developer.apple.com/help/app-store-connect/manage-app-information/edit-app-information/).

- **App Name** — up to 30 characters
- **Subtitle** — up to 30 characters (appears below name in search)
- **Description** — up to 4,000 characters; focus on features/value; no pricing or "download now" language
- **Keywords** — up to 100 characters, comma-separated, no spaces
- **Support URL** — required, must be live
- **Marketing URL** — optional
- **Privacy Policy URL** — required
- **App Category** — primary + optional secondary
- **Age Rating** — fill the questionnaire honestly
- **Screenshots** — for every required device size
- **App icon** — 1024×1024

### 11.2 App Privacy Details

Since 2020, Apple requires a detailed disclosure of what data the app collects, how it is used, whether it is linked to the user, and whether it is used for tracking. Displayed as a "privacy nutrition label" on the listing.

Reference: **[App Privacy details on the App Store](https://developer.apple.com/app-store/app-privacy-details/)**.

Since 2024, Apple also requires **[privacy manifests](https://developer.apple.com/documentation/bundleresources/privacy_manifest_files)** for many third-party SDKs.

### 11.3 Upload the Final Build

1. In Xcode, ensure scheme is your app (not a test scheme) and destination is **Any iOS Device (arm64)**.
2. **Product → Archive**.
3. In Organizer, click **Distribute App**.
4. Choose **App Store Connect → Upload**.
5. Xcode validates, signs, and uploads.
6. Wait 15–30 minutes for App Store Connect processing.

### 11.4 Submit for Review

1. In App Store Connect, open the app → version tab.
2. Under **Build**, click **+** and select the uploaded build.
3. Add **What's New in This Version** release notes.
4. Choose release option: manual (release after approval) or automatic.
5. Click **Submit for Review**.

### 11.5 Review Timeline

As of 2026, Apple typically reviews apps within **24–48 hours**. Some reviews complete within hours; others take up to a week if flagged. Live status: [App Store Review times (community-reported)](https://developer.apple.com/system-status/) is not published officially — set expectations conservatively.

If rejected, you receive a detailed reason in the [Resolution Center](https://developer.apple.com/help/app-store-connect/manage-submissions-to-app-review/respond-to-review-issues/). Fix, respond, resubmit — subsequent reviews are often faster.

---

## 12. Common Issues & Troubleshooting

### 12.1 Build Errors

| Error Pattern | Likely Cause | Resolution |
|---|---|---|
| `Command PhaseScriptExecution failed` | Node not found in Xcode's PATH | Add a `.xcode.env.local` file in `ios/` pointing to your node binary |
| `Multiple commands produce (duplicate output)` | Old build artifacts | **Product → Clean Build Folder** (⇧⌘K) |
| `Undefined symbols for architecture arm64` | Missing pod or misconfigured library | Re-run `pod install`; check library docs |
| `Signing for "AppName" requires a development team` | No signing team selected | Set Team in **Signing & Capabilities** |
| `Bundle identifier already in use` | Someone else registered the bundle ID | Change to a truly unique identifier |
| `ITMS-90000` series errors on upload | App Store Connect validation failure | Read the specific error; usually missing icon, permission, or metadata |
| App crashes on launch on device but works on simulator | Missing usage description in `Info.plist` | Add the required `NS...UsageDescription` key |
| White/black screen after splash | JavaScript bundle failed to load | Check Metro is running; or the release bundle is properly embedded |

### 12.2 UI Differences From Android

| Concern | iOS Behavior | Android Behavior |
|---|---|---|
| Shadows | `shadowColor`, `shadowOffset`, `shadowOpacity`, `shadowRadius` | `elevation` |
| Fonts | System fonts (San Francisco); Roboto not available | Roboto default |
| Touch feedback | `TouchableOpacity` / `Pressable` | `TouchableNativeFeedback` (ripple) |
| StatusBar | Does not respect `backgroundColor`; use `SafeAreaView` | Full `backgroundColor` support |
| KeyboardAvoidingView | `behavior="padding"` | `behavior="height"` |

Reference: [React Native — Platform-specific code](https://reactnative.dev/docs/platform-specific-code).

### 12.3 Common App Store Rejection Reasons

Full guidelines: **[App Store Review Guidelines](https://developer.apple.com/app-store/review/guidelines/)**.

- **Guideline 2.1** — App crashes during review. Test on the exact iOS version reviewers will use.
- **Guideline 5.1.1** — Vague permission descriptions. Be specific about *why* each permission is needed.
- **Guideline 4.0** — Poor design or incomplete features. Broken buttons or blank screens will be rejected.
- **Guideline 3.1.1** — External payment systems for digital goods (must use Apple's [In-App Purchase](https://developer.apple.com/in-app-purchase/)).
- **Guideline 5.1.2** — Data collection without clear consent or a privacy policy.
- Missing [Sign in with Apple](https://developer.apple.com/sign-in-with-apple/) when other social sign-ins are offered.
- Placeholder text or Lorem Ipsum visible in the app.

---

## 13. Realistic Timeline & Effort

Assumes one full-time developer familiar with RN but new to iOS-specific work.

| Phase | Duration | Notes |
|---|---|---|
| Mac environment setup | 1–2 days | Downloads take time; verify each tool works |
| Project transfer + first build | 1–2 days | Expect errors on first `pod install` and first build |
| Dependencies audit + fixes | 3–7 days | Highly variable; each library needs individual review |
| iOS configuration | 1–2 days | Depends on permission/feature count |
| Assets preparation | 1–2 days | Design team may need iOS-specific assets |
| Apple Developer enrollment | 1–3 days (waiting) | Blocks App Store Connect access |
| Dev + simulator testing | 2–4 days | Fixing iOS-specific UI/behavior |
| Physical device testing | 2–3 days | Real-world issues surface here |
| TestFlight beta | 3–5 days | Depends on team feedback speed |
| Submission + review | 2–5 days | Includes waiting + one likely revision cycle |
| **Total (realistic)** | **3–5 weeks** | Assuming no major architectural issues |
| Total (best case) | 2 weeks | Small app, few deps, experienced dev |
| Total (worst case) | 8–10 weeks | Many problematic deps, custom native code |

> **Recommendation to management** — communicate an initial estimate of **4–6 weeks** and update weekly. Setting expectations upfront prevents pressure to skip testing steps, which is the primary cause of App Store rejections and post-launch crashes.

---

## 14. Post-Launch Considerations

### 14.1 Crash Monitoring

Integrate a crash reporter. Official options:

- **[Firebase Crashlytics](https://firebase.google.com/products/crashlytics)** — free, integrates with existing Firebase setup
- **[Sentry](https://sentry.io/)** — see [Sentry React Native docs](https://docs.sentry.io/platforms/react-native/)
- **[Bugsnag](https://www.bugsnag.com/)** — see [Bugsnag React Native docs](https://docs.bugsnag.com/platforms/react-native/)

iOS crashes require **dSYM symbolication** — upload dSYM files after each release for readable crash reports. See [Apple — Adding identifiable symbol names to a crash report](https://developer.apple.com/documentation/xcode/adding-identifiable-symbol-names-to-a-crash-report).

### 14.2 Update Cycle

Every subsequent update: increment build number → archive → upload → release notes → submit for review. Keep Android and iOS in sync feature-wise but expect slight timing differences due to review delays.

### 14.3 Continuous Integration

For automated builds, tests, and TestFlight uploads:

- **[Xcode Cloud](https://developer.apple.com/xcode-cloud/)** — Apple's first-party CI
- **[Fastlane](https://fastlane.tools/)** — the de-facto RN release automation tool
- **[Bitrise](https://www.bitrise.io/)** — mobile-focused CI
- **[GitHub Actions with macOS runners](https://docs.github.com/en/actions/using-github-hosted-runners/about-github-hosted-runners#supported-runners-and-hardware-resources)**

### 14.4 Annual Renewal

Apple Developer Program renews annually. **If it lapses, the app is removed from the App Store.** Set a calendar reminder **30 days before renewal**.

### 14.5 Compliance and Policy Updates

Apple updates the [App Store Review Guidelines](https://developer.apple.com/app-store/review/guidelines/) regularly (privacy nutrition labels, App Tracking Transparency, [privacy manifests](https://developer.apple.com/documentation/bundleresources/privacy_manifest_files)). Subscribe to [Apple Developer News](https://developer.apple.com/news/) and review the guidelines at least quarterly.

---

## 15. Appendix — Commands & Official Resources

### 15.1 Frequently Used Commands

```bash
# Install JS dependencies
npm install

# Install iOS native dependencies
cd ios && pod install && cd ..

# Run on iOS simulator
npx react-native run-ios

# Run on a specific simulator
npx react-native run-ios --simulator="iPhone 15 Pro"

# Run on a physical device
npx react-native run-ios --device "My iPhone"

# Reset Metro bundler cache
npx react-native start --reset-cache

# Clean iOS build
cd ios && xcodebuild clean && cd ..

# Remove Pods and reinstall from scratch
cd ios && rm -rf Pods Podfile.lock && pod install && cd ..

# List connected iOS devices / simulators
xcrun xctrace list devices
```

### 15.2 Official Reference Documentation

| Topic | Official Source |
|---|---|
| React Native | [reactnative.dev](https://reactnative.dev/) |
| React Native — Environment Setup | [reactnative.dev/docs/set-up-your-environment](https://reactnative.dev/docs/set-up-your-environment) |
| React Native — Publishing to Apple App Store | [reactnative.dev/docs/publishing-to-app-store](https://reactnative.dev/docs/publishing-to-app-store) |
| Apple Developer | [developer.apple.com](https://developer.apple.com/) |
| Apple Developer Documentation | [developer.apple.com/documentation](https://developer.apple.com/documentation/) |
| App Store Connect | [appstoreconnect.apple.com](https://appstoreconnect.apple.com/) |
| App Store Connect Help | [developer.apple.com/help/app-store-connect](https://developer.apple.com/help/app-store-connect/) |
| App Store Review Guidelines | [developer.apple.com/app-store/review/guidelines](https://developer.apple.com/app-store/review/guidelines/) |
| Human Interface Guidelines | [developer.apple.com/design/human-interface-guidelines](https://developer.apple.com/design/human-interface-guidelines/) |
| Xcode | [developer.apple.com/xcode](https://developer.apple.com/xcode/) |
| Xcode Release Notes | [developer.apple.com/documentation/xcode-release-notes](https://developer.apple.com/documentation/xcode-release-notes) |
| TestFlight | [developer.apple.com/testflight](https://developer.apple.com/testflight/) |
| CocoaPods | [cocoapods.org](https://cocoapods.org/) |
| Homebrew | [brew.sh](https://brew.sh/) |
| Node.js | [nodejs.org](https://nodejs.org/) |
| nvm | [github.com/nvm-sh/nvm](https://github.com/nvm-sh/nvm) |
| Watchman | [facebook.github.io/watchman](https://facebook.github.io/watchman/) |
| rbenv | [github.com/rbenv/rbenv](https://github.com/rbenv/rbenv) |
| Ruby | [ruby-lang.org](https://www.ruby-lang.org/) |
| Firebase | [firebase.google.com](https://firebase.google.com/) |
| React Native Firebase | [rnfirebase.io](https://rnfirebase.io/) |
| React Navigation | [reactnavigation.org](https://reactnavigation.org/) |
| npm registry | [npmjs.com](https://www.npmjs.com/) |

### 15.3 Recommended Tools (Official Sites)

| Tool | Purpose | Official Site |
|---|---|---|
| **Xcode** | Apple's IDE, mandatory for iOS builds | [developer.apple.com/xcode](https://developer.apple.com/xcode/) |
| **Simulator** | Bundled with Xcode | (same as above) |
| **Fastlane** | Signing, building, uploading automation | [fastlane.tools](https://fastlane.tools/) |
| **Reactotron** | Inspection tool for React Native apps | [github.com/infinitered/reactotron](https://github.com/infinitered/reactotron) |
| **Flipper** | Debugging platform (future uncertain, still widely used) | [fbflipper.com](https://fbflipper.com/) |
| **Charles Proxy** | Network debugging | [charlesproxy.com](https://www.charlesproxy.com/) |
| **Proxyman** | Network debugging (macOS-native) | [proxyman.io](https://proxyman.io/) |

### 15.4 Final Checklist Before Submission

- [ ] App icon (1024×1024) uploaded to App Store Connect
- [ ] All required screenshots uploaded for every required device size
- [ ] Every permission has a matching `NS...UsageDescription` in `Info.plist`
- [ ] Privacy policy URL is live and accessible
- [ ] App Privacy nutrition label filled accurately
- [ ] Privacy manifests present for all third-party SDKs that require them
- [ ] App tested on **at least one physical iPhone**
- [ ] App tested via TestFlight by at least one non-developer
- [ ] No placeholder text, Lorem Ipsum, or debug UI in production build
- [ ] Bundle identifier matches between Xcode and App Store Connect
- [ ] Version and build number set correctly (build number incremented from last upload)
- [ ] Release notes written
- [ ] Backend infrastructure ready for iOS traffic
- [ ] Sign in with Apple implemented (if other social sign-ins are offered)

---

*End of guide. Bookmark this file and reuse it for every future RN → iOS project.*
