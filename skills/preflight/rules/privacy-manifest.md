# iOS 17+ Privacy Manifest (PrivacyInfo.xcprivacy) Rules

Detailed rules for the `PrivacyInfo.xcprivacy` file required by Apple starting Spring 2024 for all apps and third-party SDKs.

---

## File Structure

The `PrivacyInfo.xcprivacy` file is a property list (XML plist) with the following top-level keys:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>NSPrivacyTracking</key>
    <false/>
    <key>NSPrivacyTrackingDomains</key>
    <array/>
    <key>NSPrivacyCollectedDataTypes</key>
    <array/>
    <key>NSPrivacyAccessedAPITypes</key>
    <array/>
</dict>
</plist>
```

---

## Rule 1: NSPrivacyTracking Boolean

- **What to Check**: `NSPrivacyTracking` key in `PrivacyInfo.xcprivacy`
- **What to Look For**: A boolean value (`true` or `false`) indicating whether the app or SDK uses data for tracking as defined by the App Tracking Transparency framework.
- **Pass Condition**: Key exists with a boolean value. If the app uses ATT or IDFA, this must be `true`. If the app does not track users, this must be `false`.
- **Fail Reason**: Missing key or incorrect value. If the app tracks users but declares `false`, Apple will reject upon review.
- **Validation Steps**:
  1. Read `PrivacyInfo.xcprivacy` and check for `NSPrivacyTracking`
  2. Grep source code for `ATTrackingManager`, `advertisingIdentifier`, `ASIdentifierManager`
  3. If tracking code found but `NSPrivacyTracking` is `false`, flag as FAIL
  4. If no tracking code found and `NSPrivacyTracking` is `true`, flag as WARNING (unnecessary declaration)

---

## Rule 2: NSPrivacyTrackingDomains Array

- **What to Check**: `NSPrivacyTrackingDomains` key in `PrivacyInfo.xcprivacy`
- **What to Look For**: An array of string domains used for tracking. Only required if `NSPrivacyTracking` is `true`.
- **Pass Condition**:
  - If `NSPrivacyTracking` is `true`: array must contain at least one domain.
  - If `NSPrivacyTracking` is `false`: array should be empty.
  - Domains must be valid format (e.g., `analytics.example.com`, not full URLs).
- **Fail Reason**: If tracking is enabled but no domains listed, Apple will request correction. If tracking domains are not listed, tracking requests may be blocked by the system.
- **Validation Steps**:
  1. Check `NSPrivacyTracking` value first
  2. If `true`, verify `NSPrivacyTrackingDomains` is non-empty
  3. Grep code for analytics/tracking SDK initialization URLs to cross-reference
  4. Common tracking domains to look for: Firebase Analytics, Facebook SDK, Adjust, AppsFlyer, Amplitude endpoints

---

## Rule 3: NSPrivacyCollectedDataTypes Array

- **What to Check**: `NSPrivacyCollectedDataTypes` key in `PrivacyInfo.xcprivacy`
- **What to Look For**: An array of dictionaries, each declaring a collected data type with:
  - `NSPrivacyCollectedDataType` (string): The data type identifier
  - `NSPrivacyCollectedDataTypeLinked` (boolean): Whether data is linked to user identity
  - `NSPrivacyCollectedDataTypePurposes` (array): Why the data is collected
  - `NSPrivacyCollectedDataTypeTracking` (boolean): Whether used for tracking

### Valid Data Type Identifiers:
  - `NSPrivacyCollectedDataTypeName`
  - `NSPrivacyCollectedDataTypeEmailAddress`
  - `NSPrivacyCollectedDataTypePhoneNumber`
  - `NSPrivacyCollectedDataTypePhysicalAddress`
  - `NSPrivacyCollectedDataTypeOtherUserContactInfo`
  - `NSPrivacyCollectedDataTypeHealth`
  - `NSPrivacyCollectedDataTypeFitness`
  - `NSPrivacyCollectedDataTypePaymentInfo`
  - `NSPrivacyCollectedDataTypeCreditInfo`
  - `NSPrivacyCollectedDataTypeOtherFinancialInfo`
  - `NSPrivacyCollectedDataTypePreciseLocation`
  - `NSPrivacyCollectedDataTypeCoarseLocation`
  - `NSPrivacyCollectedDataTypeSensitiveInfo`
  - `NSPrivacyCollectedDataTypeContacts`
  - `NSPrivacyCollectedDataTypeEmails`
  - `NSPrivacyCollectedDataTypeTextMessages`
  - `NSPrivacyCollectedDataTypePhotoOrVideo`
  - `NSPrivacyCollectedDataTypeAudioData`
  - `NSPrivacyCollectedDataTypeGameplayContent`
  - `NSPrivacyCollectedDataTypeCustomerSupport`
  - `NSPrivacyCollectedDataTypeOtherUserContent`
  - `NSPrivacyCollectedDataTypeBrowsingHistory`
  - `NSPrivacyCollectedDataTypeSearchHistory`
  - `NSPrivacyCollectedDataTypeUserID`
  - `NSPrivacyCollectedDataTypeDeviceID`
  - `NSPrivacyCollectedDataTypePurchaseHistory`
  - `NSPrivacyCollectedDataTypeProductInteraction`
  - `NSPrivacyCollectedDataTypeAdvertisingData`
  - `NSPrivacyCollectedDataTypeOtherUsageData`
  - `NSPrivacyCollectedDataTypeCrashData`
  - `NSPrivacyCollectedDataTypePerformanceData`
  - `NSPrivacyCollectedDataTypeOtherDiagnosticData`
  - `NSPrivacyCollectedDataTypeEnvironmentScanning`
  - `NSPrivacyCollectedDataTypeHands`
  - `NSPrivacyCollectedDataTypeHead`
  - `NSPrivacyCollectedDataTypeOtherDataTypes`

### Valid Purpose Identifiers:
  - `NSPrivacyCollectedDataTypePurposeThirdPartyAdvertising`
  - `NSPrivacyCollectedDataTypePurposeDeveloperAdvertising`
  - `NSPrivacyCollectedDataTypePurposeAnalytics`
  - `NSPrivacyCollectedDataTypePurposeProductPersonalization`
  - `NSPrivacyCollectedDataTypePurposeAppFunctionality`
  - `NSPrivacyCollectedDataTypePurposeOther`

- **Pass Condition**: Every data type the app actually collects (from code analysis) is declared in the array with correct linked/tracking/purpose values.
- **Fail Reason**: Undeclared data collection is a policy violation. Apple cross-references the privacy manifest with actual app behavior.
- **Validation Steps**:
  1. Read `NSPrivacyCollectedDataTypes` array
  2. Grep code for location APIs -> must declare location data type
  3. Grep code for contact APIs -> must declare contacts data type
  4. Check for analytics SDKs -> must declare usage/diagnostic data types
  5. Check for auth/login -> must declare user ID, email, etc.
  6. Cross-reference all findings with declarations

---

## Rule 4: NSPrivacyAccessedAPITypes - Required Reason APIs

- **What to Check**: `NSPrivacyAccessedAPITypes` key in `PrivacyInfo.xcprivacy`
- **What to Look For**: An array of dictionaries declaring each Required Reason API used:
  - `NSPrivacyAccessedAPIType` (string): The API category
  - `NSPrivacyAccessedAPITypeReasons` (array of strings): Reason codes

### API Categories and Reason Codes:

#### File Timestamp APIs (`NSPrivacyAccessedAPICategoryFileTimestamp`)
Used when accessing: `NSFileCreationDate`, `NSFileModificationDate`, `NSFileContentModificationDate`, `stat()`, `fstat()`, `getattrlist()`

Valid reason codes:
- `DDA9.1` - Display to user
- `C617.1` - Access within app's container
- `3B52.1` - Access file timestamps within app group container
- `0A2A.1` - Third-party SDK wrapper

#### Disk Space APIs (`NSPrivacyAccessedAPICategoryDiskSpace`)
Used when accessing: `NSFileSystemFreeSize`, `NSFileSystemSize`, `statfs()`, `statvfs()`, `fstatfs()`

Valid reason codes:
- `E174.1` - Check for sufficient space before writing
- `85F4.1` - Display to user
- `AB6B.1` - Health-related research
- `7D9E.1` - Third-party SDK wrapper

#### User Defaults APIs (`NSPrivacyAccessedAPICategoryUserDefaults`)
Used when accessing: `UserDefaults`

Valid reason codes:
- `CA92.1` - Access within app itself
- `1C8F.1` - Access within app group
- `C56D.1` - Third-party SDK wrapper

#### System Boot Time APIs (`NSPrivacyAccessedAPICategorySystemBootTime`)
Used when accessing: `systemUptime`, `mach_absolute_time()`, `ProcessInfo.processInfo.systemUptime`

Valid reason codes:
- `35F9.1` - Measure elapsed time
- `8FFB.1` - Calculate absolute timestamps
- `3D61.1` - Third-party SDK wrapper

#### Active Keyboards API (`NSPrivacyAccessedAPICategoryActiveKeyboards`)
Used when accessing: list of active keyboards

Valid reason codes:
- `54BD.1` - Custom keyboard app
- `3EC4.1` - Third-party SDK wrapper

- **Pass Condition**: Every Required Reason API used in the app's code (and third-party SDKs) is declared with at least one valid reason code.
- **Fail Reason**: Using a Required Reason API without declaration causes rejection starting Spring 2024.
- **Validation Steps**:
  1. Grep all source files for Required Reason API patterns:
     - `NSFileCreationDate|NSFileModificationDate|stat\(|fstat\(|getattrlist` -> File Timestamp
     - `NSFileSystemFreeSize|NSFileSystemSize|statfs|statvfs|fstatfs` -> Disk Space
     - `UserDefaults` -> User Defaults
     - `systemUptime|mach_absolute_time|processInfo\.systemUptime` -> System Boot Time
  2. For each API found, check `NSPrivacyAccessedAPITypes` for matching category and valid reason code
  3. Also check inside `Pods/` and third-party frameworks for API usage

---

## Rule 5: Third-Party SDK Privacy Manifests

- **What to Check**: `PrivacyInfo.xcprivacy` inside each third-party SDK/framework
- **What to Look For**: Apple maintains a list of SDKs that MUST include their own privacy manifests. Check:
  - `Pods/[SDKName]/PrivacyInfo.xcprivacy`
  - `*.xcframework/[arch]/[SDKName].framework/PrivacyInfo.xcprivacy`
  - `.build/` directory for SPM packages

### SDKs Requiring Privacy Manifests (Apple's list):
  - Abseil
  - AFNetworking
  - Alamofire
  - AppAuth
  - BoringSSL / openssl_grpc
  - Capacitor
  - Charts
  - CocoaLumberjack
  - ConnectivityManager
  - Cosmos
  - CryptoSwift
  - Firebase (all modules)
  - FBAEMKit / FBSDKCoreKit (Facebook)
  - Flipper
  - fmt
  - glog
  - Google Maps / Places
  - GoogleSignIn / GoogleUtilities / GTMSessionFetcher
  - gRPC
  - Kingfisher
  - leveldb
  - Lottie
  - MBProgressHUD
  - nanopb
  - OneSignal
  - OpenCV
  - OrderedSet
  - PINCache / PINRemoteImage
  - Promises
  - Realm
  - RxSwift
  - SDWebImage
  - Sentry
  - SnapKit
  - SQLite.swift
  - SVGKit
  - SVProgressHUD
  - Toast
  - YogaKit

- **Pass Condition**: Every third-party SDK from Apple's required list that is included in the project has a `PrivacyInfo.xcprivacy` bundled within it.
- **Fail Reason**: Apps including listed SDKs without privacy manifests will receive an email warning and eventually rejection.
- **Validation Steps**:
  1. List all third-party dependencies (from `Podfile.lock`, `Package.resolved`, or Carthage)
  2. Cross-reference with Apple's required list
  3. For each matched SDK, check if `PrivacyInfo.xcprivacy` exists within the SDK bundle
  4. If SDK is outdated and lacks manifest, flag with update recommendation
